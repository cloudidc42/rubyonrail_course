# Part 72: Notifications System

## Steps 1561-1580

---

## Step 1561: In-App Notifications ด้วย Action Cable

Action Cable ให้ Rails รองรับ WebSockets แบบ built-in

### Setup Action Cable

```ruby
# config/cable.yml
development:
  adapter: async

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: myapp_production

# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user
    
    def connect
      self.current_user = find_verified_user
    end
    
    private
    
    def find_verified_user
      if (user_id = cookies.encrypted[:user_id])
        User.find(user_id)
      elsif (token = request.params[:token])
        payload = JsonWebToken.decode(token)
        User.find(payload[:user_id])
      else
        reject_unauthorized_connection
      end
    end
  end
end

# app/channels/notifications_channel.rb
class NotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
    
    # ส่ง unread count เมื่อ connect
    transmit({ 
      type: 'unread_count',
      count: current_user.notifications.unread.count 
    })
  end
  
  def unsubscribed
    stop_all_streams
  end
  
  # Client สามารถ call method นี้ได้
  def mark_read(data)
    notification_ids = data['notification_ids']
    current_user.notifications.where(id: notification_ids).update_all(read_at: Time.current)
    
    transmit({
      type: 'notifications_marked_read',
      notification_ids: notification_ids,
      unread_count: current_user.notifications.unread.count
    })
  end
  
  def mark_all_read
    current_user.notifications.unread.update_all(read_at: Time.current)
    
    transmit({
      type: 'all_notifications_read',
      unread_count: 0
    })
  end
end
```

### Notification Model

```ruby
# migration
class CreateNotifications < ActiveRecord::Migration[7.0]
  def change
    create_table :notifications do |t|
      t.references :recipient, null: false, foreign_key: { to_table: :users }
      t.string :type, null: false
      t.references :notifiable, polymorphic: true
      t.string :title
      t.text :body
      t.string :action_url
      t.datetime :read_at
      t.datetime :emailed_at
      t.jsonb :metadata, default: {}
      t.timestamps
    end
    
    add_index :notifications, [:recipient_id, :read_at]
    add_index :notifications, [:recipient_id, :created_at]
  end
end

# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :recipient, class_name: 'User'
  belongs_to :notifiable, polymorphic: true, optional: true
  
  scope :unread, -> { where(read_at: nil) }
  scope :read, -> { where.not(read_at: nil) }
  scope :recent, -> { order(created_at: :desc) }
  scope :for_display, -> { recent.limit(50) }
  
  def read?
    read_at.present?
  end
  
  def unread?
    !read?
  end
  
  def mark_read!
    update!(read_at: Time.current) if unread?
  end
  
  # Deliver notification via Action Cable
  def deliver!
    NotificationsChannel.broadcast_to(
      recipient,
      {
        type: 'new_notification',
        notification: NotificationSerializer.new(self).as_json,
        unread_count: recipient.notifications.unread.count
      }
    )
    
    # Email
    if recipient.email_notifications_enabled? && !emailed_at
      NotificationMailer.notification_email(self).deliver_later
      update_column(:emailed_at, Time.current)
    end
  end
end
```

### ส่ง Notifications

```ruby
# app/services/send_notification.rb
class SendNotification
  def self.call(**args)
    new(**args).call
  end
  
  def initialize(recipient:, type:, notifiable: nil, title: nil, body: nil, action_url: nil, metadata: {})
    @recipient = recipient
    @type = type
    @notifiable = notifiable
    @title = title
    @body = body
    @action_url = action_url
    @metadata = metadata
  end
  
  def call
    notification = Notification.create!(
      recipient: @recipient,
      type: @type,
      notifiable: @notifiable,
      title: resolved_title,
      body: resolved_body,
      action_url: @action_url,
      metadata: @metadata
    )
    
    notification.deliver!
    notification
  end
  
  private
  
  def resolved_title
    @title || I18n.t("notifications.#{@type}.title", **@metadata.symbolize_keys)
  end
  
  def resolved_body
    @body || I18n.t("notifications.#{@type}.body", **@metadata.symbolize_keys)
  end
end

# config/locales/notifications.th.yml
th:
  notifications:
    order_placed:
      title: 'สั่งซื้อสำเร็จ'
      body: 'Order %{order_number} ของคุณได้รับการยืนยันแล้ว'
    order_shipped:
      title: 'สินค้าถูกจัดส่งแล้ว'
      body: 'Order %{order_number} ถูกจัดส่งด้วย %{carrier}'
    payment_failed:
      title: 'การชำระเงินล้มเหลว'
      body: 'การชำระเงินสำหรับ Order %{order_number} ล้มเหลว'
    new_follower:
      title: 'มีผู้ติดตามใหม่'
      body: '%{follower_name} ติดตามคุณแล้ว'
    comment_reply:
      title: 'มีการตอบกลับความเห็น'
      body: '%{commenter_name} ตอบกลับความเห็นของคุณ'
```

---

## Step 1562: Email Notifications

```ruby
# app/mailers/notification_mailer.rb
class NotificationMailer < ApplicationMailer
  def notification_email(notification)
    @notification = notification
    @user = notification.recipient
    
    mail(
      to: @user.email,
      subject: notification.title
    ) do |format|
      format.html { render template: "notification_mailer/#{notification.type}" }
      format.text { render template: "notification_mailer/#{notification.type}" }
    end
  end
  
  # Email digest (รวม notifications หลายอัน)
  def daily_digest(user)
    @user = user
    @notifications = user.notifications
      .unread
      .where('created_at >= ?', 1.day.ago)
      .order(created_at: :desc)
    
    return if @notifications.empty?
    
    mail(
      to: user.email,
      subject: "คุณมี #{@notifications.count} notifications ที่ยังไม่ได้อ่าน"
    )
  end
end

# app/views/notification_mailer/order_placed.html.erb
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <style>
    .container { max-width: 600px; margin: 0 auto; font-family: sans-serif; }
    .header { background: #4CAF50; color: white; padding: 20px; text-align: center; }
    .content { padding: 30px; }
    .btn { background: #4CAF50; color: white; padding: 12px 24px; text-decoration: none; border-radius: 4px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>สั่งซื้อสำเร็จ!</h1>
    </div>
    <div class="content">
      <p>สวัสดีคุณ <%= @user.name %>,</p>
      <p><%= @notification.body %></p>
      
      <% if @notification.action_url.present? %>
        <p>
          <%= link_to 'ดู Order ของคุณ', @notification.action_url, class: 'btn' %>
        </p>
      <% end %>
      
      <p>ขอบคุณที่ใช้บริการ</p>
    </div>
  </div>
</body>
</html>
```

---

## Step 1563: Noticed Gem

Noticed gem ทำให้การสร้าง notification system ง่ายขึ้นมาก

```ruby
# Gemfile
gem 'noticed'

# Terminal
bundle install
rails noticed:install:migrations
rails db:migrate

# สร้าง notifier
rails generate noticed:notification OrderPlacedNotification
```

### สร้าง Notifiers

```ruby
# app/notifiers/order_placed_notifier.rb
class OrderPlacedNotifier < Noticed::Base
  # ช่องทางการส่ง
  deliver_by :database
  deliver_by :email, mailer: 'OrderMailer', method: 'order_confirmation'
  deliver_by :action_cable,
    channel: 'NotificationsChannel',
    stream_name: -> { params[:recipient] }
  
  # ส่ง push notification ถ้า user มี push token
  deliver_by :web_push, if: :push_enabled?
  
  # Template สำหรับ message
  notification_methods do
    def message
      "Order #{params[:order].number} ของคุณได้รับการยืนยัน"
    end
    
    def url
      order_url(params[:order])
    end
    
    def icon
      'shopping-cart'
    end
  end
  
  private
  
  def push_enabled?
    recipient.push_notifications_enabled?
  end
end

# app/notifiers/new_follower_notifier.rb
class NewFollowerNotifier < Noticed::Base
  deliver_by :database
  deliver_by :email, mailer: 'FollowerMailer', if: :email_notifications?
  deliver_by :action_cable, channel: 'NotificationsChannel'
  
  notification_methods do
    def message
      "#{params[:follower].name} เริ่มติดตามคุณ"
    end
    
    def url
      profile_url(params[:follower])
    end
    
    def avatar_url
      params[:follower].avatar_url
    end
  end
  
  private
  
  def email_notifications?
    recipient.email_notifications_enabled?
  end
end

# การส่ง notification
order = Order.find(1)
user = order.user

# ส่งให้ผู้ใช้คนเดียว
OrderPlacedNotifier.with(order: order).deliver(user)

# ส่งให้หลายคน
users = User.where(role: 'admin')
OrderPlacedNotifier.with(order: order).deliver_later(users)

# ใน model callback
class Order < ApplicationRecord
  after_commit :notify_user_on_create, on: :create
  after_commit :notify_status_change, on: :update, if: :saved_change_to_status?
  
  private
  
  def notify_user_on_create
    OrderPlacedNotifier.with(order: self).deliver_later(user)
  end
  
  def notify_status_change
    OrderStatusChangedNotifier.with(
      order: self,
      previous_status: status_before_last_save,
      new_status: status
    ).deliver_later(user)
  end
end
```

---

## Step 1564: Push Notifications (Web Push)

```ruby
# Gemfile
gem 'webpush'
gem 'serviceworker-rails'

# config/initializers/webpush.rb
WebPush.vapid_keys  # สร้าง VAPID keys ครั้งแรก

# บันทึก keys ใน credentials
# VAPID_PUBLIC_KEY=xxx
# VAPID_PRIVATE_KEY=xxx

# migration
class AddPushTokenToUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :push_subscriptions do |t|
      t.references :user, null: false, foreign_key: true
      t.string :endpoint, null: false
      t.string :p256dh_key
      t.string :auth_key
      t.string :device_name
      t.datetime :last_used_at
      t.timestamps
      
      t.index :endpoint, unique: true
    end
  end
end

# app/models/push_subscription.rb
class PushSubscription < ApplicationRecord
  belongs_to :user
  
  def self.subscribe(user:, subscription_data:, device_name: nil)
    find_or_create_by(endpoint: subscription_data[:endpoint]) do |sub|
      sub.user = user
      sub.p256dh_key = subscription_data.dig(:keys, :p256dh)
      sub.auth_key = subscription_data.dig(:keys, :auth)
      sub.device_name = device_name
    end
  end
end

# app/services/send_push_notification.rb
class SendPushNotification
  def self.call(user:, title:, body:, url: nil, icon: nil)
    new(user: user, title: title, body: body, url: url, icon: icon).call
  end
  
  def initialize(user:, title:, body:, url: nil, icon: nil)
    @user = user
    @title = title
    @body = body
    @url = url
    @icon = icon
  end
  
  def call
    payload = {
      title: @title,
      body: @body,
      url: @url || '/',
      icon: @icon || '/icon-192x192.png',
      badge: '/badge-72x72.png',
      timestamp: Time.current.to_i
    }.to_json
    
    @user.push_subscriptions.each do |subscription|
      send_push(subscription, payload)
    end
  end
  
  private
  
  def send_push(subscription, payload)
    WebPush.payload_send(
      message: payload,
      endpoint: subscription.endpoint,
      p256dh: subscription.p256dh_key,
      auth: subscription.auth_key,
      vapid: {
        public_key: ENV['VAPID_PUBLIC_KEY'],
        private_key: ENV['VAPID_PRIVATE_KEY'],
        subject: 'mailto:admin@myapp.com'
      },
      ttl: 3600  # 1 hour
    )
    
    subscription.update_column(:last_used_at, Time.current)
  rescue WebPush::ExpiredSubscription, WebPush::InvalidSubscription
    subscription.destroy  # subscription หมดอายุ
  rescue WebPush::Error => e
    Rails.logger.error "WebPush error: #{e.message}"
  end
end

# JavaScript: Service Worker
# public/sw.js
self.addEventListener('push', function(event) {
  const data = event.data.json();
  
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: data.icon,
      badge: data.badge,
      data: { url: data.url }
    })
  );
});

self.addEventListener('notificationclick', function(event) {
  event.notification.close();
  
  event.waitUntil(
    clients.openWindow(event.notification.data.url)
  );
});
```

---

## Step 1565: Notification Center Design

```ruby
# app/controllers/notifications_controller.rb
class NotificationsController < ApplicationController
  before_action :authenticate_user!
  
  def index
    @notifications = current_user.notifications
      .order(created_at: :desc)
      .page(params[:page])
      .per(20)
    
    # Mark visible ones as read after short delay (หน้า index เท่านั้น)
    head :ok if request.format.json?
  end
  
  def unread_count
    count = current_user.notifications.unread.count
    render json: { count: count }
  end
  
  def mark_read
    if params[:id]
      notification = current_user.notifications.find(params[:id])
      notification.mark_read!
    end
    
    render json: { 
      success: true,
      unread_count: current_user.notifications.unread.count
    }
  end
  
  def mark_all_read
    current_user.notifications.unread.update_all(read_at: Time.current)
    
    respond_to do |format|
      format.json { render json: { success: true, unread_count: 0 } }
      format.html { redirect_back fallback_location: notifications_path }
    end
  end
  
  def destroy
    notification = current_user.notifications.find(params[:id])
    notification.destroy
    
    render json: { success: true }
  end
  
  def destroy_all
    current_user.notifications.destroy_all
    redirect_to notifications_path, notice: 'ลบ notifications ทั้งหมดแล้ว'
  end
end
```

### Notification Bell Component

```erb
<%# app/views/shared/_notification_bell.html.erb %>
<div class="notification-bell" 
     data-controller="notifications"
     data-notifications-url-value="<%= unread_count_notifications_path %>">
  
  <button class="bell-button" data-action="click->notifications#toggleDropdown">
    <svg><!-- bell icon --></svg>
    
    <span class="badge" 
          data-notifications-target="badge"
          style="<%= current_user.notifications.unread.count.zero? ? 'display:none' : '' %>">
      <%= current_user.notifications.unread.count %>
    </span>
  </button>
  
  <div class="notification-dropdown" 
       data-notifications-target="dropdown"
       style="display: none;">
    
    <div class="dropdown-header">
      <h4>Notifications</h4>
      <% if current_user.notifications.unread.any? %>
        <%= button_to 'อ่านทั้งหมด', mark_all_read_notifications_path, 
            method: :post,
            class: 'btn-text',
            data: { action: 'notifications#markAllRead' } %>
      <% end %>
    </div>
    
    <div class="notifications-list" data-notifications-target="list">
      <%= render partial: 'notifications/notification',
          collection: current_user.notifications.for_display,
          as: :notification %>
    </div>
    
    <div class="dropdown-footer">
      <%= link_to 'ดูทั้งหมด', notifications_path %>
    </div>
  </div>
</div>
```

---

## Step 1566: Polymorphic Notifications

```ruby
# Notification types ที่เชื่อมกับ resources ต่างๆ

class OrderNotification < Notification
  def icon_class
    'shopping-bag'
  end
  
  def color
    case notifiable.status
    when 'paid'     then 'green'
    when 'shipped'  then 'blue'
    when 'refunded' then 'orange'
    else                 'gray'
    end
  end
end

class CommentNotification < Notification
  def icon_class
    'comment'
  end
  
  def mentioned_by
    metadata['commenter_name']
  end
end

class FollowerNotification < Notification
  def icon_class
    'person-add'
  end
  
  def follower
    User.find_by(id: metadata['follower_id'])
  end
end

class SystemNotification < Notification
  def icon_class
    'info-circle'
  end
end

# Polymorphic relation ในการ send
class Order < ApplicationRecord
  def notify_status_change
    case status
    when 'paid'
      OrderNotification.create!(
        recipient: user,
        notifiable: self,
        title: 'ชำระเงินสำเร็จ',
        body: "Order #{number} ได้รับการยืนยัน",
        action_url: order_path(self)
      ).deliver!
    when 'shipped'
      OrderNotification.create!(
        recipient: user,
        notifiable: self,
        title: 'สินค้าถูกจัดส่งแล้ว',
        body: "Order #{number} กำลังจัดส่งด้วย #{shipping_carrier}",
        action_url: order_path(self)
      ).deliver!
    end
  end
end
```

---

## Step 1567: Read/Unread Tracking

```ruby
# Optimized read tracking
class User < ApplicationRecord
  has_many :notifications, foreign_key: :recipient_id
  
  def unread_notifications_count
    Rails.cache.fetch("#{cache_key_with_version}/unread_notifications", expires_in: 5.minutes) do
      notifications.unread.count
    end
  end
  
  def mark_notifications_read!(notification_ids = nil)
    scope = notifications.unread
    scope = scope.where(id: notification_ids) if notification_ids
    
    scope.update_all(read_at: Time.current)
    
    # Clear cache
    Rails.cache.delete("#{cache_key_with_version}/unread_notifications")
    
    # Broadcast updated count
    NotificationsChannel.broadcast_to(
      self,
      { type: 'unread_count', count: 0 }
    )
  end
end

# Bulk mark read with pagination
class MarkNotificationsRead
  def self.call(user:, before: nil, types: nil)
    scope = user.notifications.unread
    scope = scope.where('created_at <= ?', before) if before
    scope = scope.where(type: types) if types.present?
    
    scope.find_in_batches(batch_size: 1000) do |batch|
      ids = batch.map(&:id)
      Notification.where(id: ids).update_all(read_at: Time.current)
    end
  end
end
```

---

## Step 1568: Bulk Notifications

```ruby
# ส่ง notification ให้ผู้ใช้จำนวนมาก
class BulkNotificationSender
  def self.send_to_all(title:, body:, action_url: nil, user_filter: {})
    users = User.active
    users = users.where(**user_filter) if user_filter.present?
    
    new(users: users, title: title, body: body, action_url: action_url).call
  end
  
  def initialize(users:, title:, body:, action_url: nil)
    @users = users
    @title = title
    @body = body
    @action_url = action_url
  end
  
  def call
    # สร้าง notifications แบบ bulk
    notifications_data = @users.map do |user|
      {
        recipient_id: user.id,
        type: 'SystemNotification',
        title: @title,
        body: @body,
        action_url: @action_url,
        created_at: Time.current,
        updated_at: Time.current
      }
    end
    
    # Insert batch
    Notification.insert_all(notifications_data)
    
    # Broadcast ผ่าน Action Cable (ทีละ batch)
    @users.find_in_batches(batch_size: 100) do |batch|
      batch.each do |user|
        NotificationsChannel.broadcast_to(
          user,
          {
            type: 'new_notification',
            title: @title,
            body: @body,
            action_url: @action_url
          }
        )
      end
    end
    
    # Send emails via background job
    BulkEmailJob.perform_later(
      user_ids: @users.pluck(:id),
      title: @title,
      body: @body
    )
  end
end

# ใช้งาน
BulkNotificationSender.send_to_all(
  title: 'ระบบปิดปรับปรุง',
  body: 'ระบบจะปิดปรับปรุงในวันที่ 1 ม.ค. 2025 เวลา 00:00-06:00 น.',
  action_url: '/maintenance'
)

# ส่งเฉพาะกลุ่ม
BulkNotificationSender.send_to_all(
  title: 'Flash Sale สำหรับสมาชิก Premium',
  body: 'ลด 50% ทุกสินค้าวันนี้เท่านั้น!',
  user_filter: { role: 'premium' }
)
```

---

## Step 1569-1580: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: Action Cable Notification

```javascript
// app/javascript/controllers/notifications_controller.js
import { Controller } from "@hotwired/stimulus"
import { createConsumer } from "@rails/actioncable"

export default class extends Controller {
  static targets = ["badge", "dropdown", "list"]
  static values = { url: String }
  
  connect() {
    this.subscription = createConsumer().subscriptions.create(
      "NotificationsChannel",
      {
        connected: () => console.log("Connected to notifications"),
        
        received: (data) => {
          this.handleData(data)
        }
      }
    )
  }
  
  disconnect() {
    this.subscription?.unsubscribe()
  }
  
  handleData(data) {
    switch(data.type) {
      case 'new_notification':
        this.addNotification(data.notification)
        this.updateBadge(data.unread_count)
        this.showToast(data.notification)
        break
      
      case 'unread_count':
        this.updateBadge(data.count)
        break
      
      case 'all_notifications_read':
        this.updateBadge(0)
        this.markAllAsRead()
        break
    }
  }
  
  addNotification(notification) {
    const html = this.renderNotification(notification)
    this.listTarget.insertAdjacentHTML('afterbegin', html)
    
    // Remove old if too many
    const items = this.listTarget.querySelectorAll('.notification-item')
    if (items.length > 50) {
      items[items.length - 1].remove()
    }
  }
  
  updateBadge(count) {
    if (count > 0) {
      this.badgeTarget.textContent = count > 99 ? '99+' : count
      this.badgeTarget.style.display = 'block'
    } else {
      this.badgeTarget.style.display = 'none'
    }
  }
  
  showToast(notification) {
    // Show toast message
    const toast = document.createElement('div')
    toast.className = 'notification-toast'
    toast.innerHTML = `
      <strong>${notification.title}</strong>
      <p>${notification.body}</p>
    `
    document.body.appendChild(toast)
    
    setTimeout(() => toast.remove(), 5000)
  }
  
  toggleDropdown() {
    const dropdown = this.dropdownTarget
    if (dropdown.style.display === 'none') {
      dropdown.style.display = 'block'
      this.loadNotifications()
    } else {
      dropdown.style.display = 'none'
    }
  }
  
  markAllRead() {
    this.subscription.perform('mark_all_read')
  }
  
  renderNotification(notification) {
    return `
      <div class="notification-item ${notification.read_at ? '' : 'unread'}" 
           data-notification-id="${notification.id}">
        <div class="notification-icon">${notification.icon}</div>
        <div class="notification-content">
          <p class="title">${notification.title}</p>
          <p class="body">${notification.body}</p>
          <small>${notification.created_at}</small>
        </div>
      </div>
    `
  }
}
```

### แบบฝึกหัดที่ 2: Notification Preferences

```ruby
# app/models/notification_preference.rb
class NotificationPreference < ApplicationRecord
  belongs_to :user
  
  NOTIFICATION_TYPES = %w[
    order_updates
    promotions
    new_followers
    comment_replies
    system_updates
    weekly_digest
  ].freeze
  
  DELIVERY_CHANNELS = %w[email push in_app].freeze
  
  validates :notification_type, inclusion: { in: NOTIFICATION_TYPES }
  
  def self.default_preferences_for(user)
    NOTIFICATION_TYPES.map do |type|
      user.notification_preferences.find_or_create_by!(
        notification_type: type
      ) do |pref|
        pref.email_enabled = true
        pref.push_enabled = false
        pref.in_app_enabled = true
      end
    end
  end
  
  def enabled_for?(channel)
    send("#{channel}_enabled")
  end
end

# Respect user preferences
class SendNotification
  def call
    return unless @recipient.wants_notification?(@type)
    
    notification = create_notification
    
    # Email
    if @recipient.notification_enabled?(@type, :email)
      NotificationMailer.notification_email(notification).deliver_later
    end
    
    # Push
    if @recipient.notification_enabled?(@type, :push)
      SendPushNotification.call(
        user: @recipient,
        title: notification.title,
        body: notification.body,
        url: notification.action_url
      )
    end
    
    # In-app (Action Cable)
    if @recipient.notification_enabled?(@type, :in_app)
      NotificationsChannel.broadcast_to(@recipient, {
        type: 'new_notification',
        notification: NotificationSerializer.new(notification).as_json,
        unread_count: @recipient.notifications.unread.count
      })
    end
    
    notification
  end
end
```

### แบบฝึกหัดที่ 3: Noticed Notifier

```ruby
# app/notifiers/new_message_notifier.rb
class NewMessageNotifier < Noticed::Base
  deliver_by :database
  deliver_by :action_cable, channel: 'NotificationsChannel'
  deliver_by :email, 
    mailer: 'MessageMailer', 
    method: 'new_message',
    if: :email_enabled?
  deliver_by :web_push, if: :push_enabled?
  
  notification_methods do
    def message
      "#{params[:sender].name} ส่งข้อความหาคุณ: #{params[:message].content.truncate(50)}"
    end
    
    def url
      conversation_url(params[:conversation])
    end
    
    def sender_name
      params[:sender].name
    end
    
    def sender_avatar
      params[:sender].avatar_url
    end
  end
  
  private
  
  def email_enabled?
    recipient.notification_preferences
      .find_by(notification_type: 'messages')
      &.email_enabled?
  end
  
  def push_enabled?
    recipient.notification_preferences
      .find_by(notification_type: 'messages')
      &.push_enabled? && recipient.push_subscriptions.any?
  end
end

# ใช้งาน
NewMessageNotifier.with(
  sender: current_user,
  message: message,
  conversation: conversation
).deliver_later(conversation.other_participant(current_user))
```

### แบบฝึกหัดที่ 4: Notification Serializer

```ruby
# app/serializers/notification_serializer.rb
class NotificationSerializer
  include JSONAPI::Serializer
  
  attributes :id, :type, :title, :body, :action_url, :created_at
  
  attribute :read do |notification|
    notification.read?
  end
  
  attribute :time_ago do |notification|
    ActionController::Base.helpers.time_ago_in_words(notification.created_at) + ' ที่ผ่านมา'
  end
  
  attribute :icon do |notification|
    case notification.type
    when 'OrderNotification'    then 'shopping-bag'
    when 'CommentNotification'  then 'comment'
    when 'FollowerNotification' then 'person-add'
    when 'SystemNotification'   then 'info-circle'
    else                             'bell'
    end
  end
  
  attribute :notifiable do |notification|
    if notification.notifiable
      {
        type: notification.notifiable_type,
        id: notification.notifiable_id
      }
    end
  end
end
```

### แบบฝึกหัดที่ 5: Testing Notifications

```ruby
# spec/notifiers/order_placed_notifier_spec.rb
RSpec.describe OrderPlacedNotifier do
  let(:user)  { create(:user) }
  let(:order) { create(:order, user: user) }
  
  describe 'delivery' do
    before { Noticed::Delivery.backend = :test }
    after  { Noticed.reset_deliveries! }
    
    it 'sends notification to user' do
      OrderPlacedNotifier.with(order: order).deliver(user)
      
      expect(Noticed.deliveries).to include(
        a_hash_including(
          recipient: user,
          notification_type: 'OrderPlacedNotifier'
        )
      )
    end
    
    it 'creates database record' do
      expect {
        OrderPlacedNotifier.with(order: order).deliver(user)
      }.to change(Notification, :count).by(1)
      
      notification = Notification.last
      expect(notification.recipient).to eq(user)
      expect(notification.title).to include(order.number)
    end
    
    it 'broadcasts via ActionCable' do
      expect(NotificationsChannel).to receive(:broadcast_to).with(
        user,
        hash_including(type: 'new_notification')
      )
      
      OrderPlacedNotifier.with(order: order).deliver(user)
    end
    
    it 'sends email when preferences enabled' do
      user.notification_preferences.find_or_create_by!(
        notification_type: 'order_updates'
      ).update!(email_enabled: true)
      
      expect {
        OrderPlacedNotifier.with(order: order).deliver(user)
      }.to change { ActionMailer::Base.deliveries.count }.by(1)
    end
  end
end
```

---

**จบ Part 72: Notifications System**

*ในส่วนถัดไป Part 73 เราจะเรียนรู้เกี่ยวกับ Multi-tenancy*
