# Part 53: WebSockets ด้วย Action Cable

## ขั้นตอนที่ 1161-1180: Real-time Communication

---

## ขั้นตอนที่ 1161: Action Cable Overview

Action Cable ผสาน WebSockets เข้ากับ Rails อย่างสมบูรณ์

```
Architecture:
Browser ↔ WebSocket Connection ↔ Action Cable Server ↔ Redis PubSub ↔ Rails App

Components:
- Channel: Ruby class ใน server (เหมือน controller)
- Subscription: JavaScript class ใน browser
- Stream: การส่งข้อมูลจาก server ไปยัง browser
- Broadcast: ส่งข้อมูลไปยัง subscribers ทั้งหมด
```

## ขั้นตอนที่ 1162: Setup

```ruby
# config/cable.yml
development:
  adapter: async  # In-memory, development only

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: myapp_production
```

```ruby
# config/application.rb
config.action_cable.url = "wss://myapp.com/cable"
config.action_cable.allowed_request_origins = [
  "https://myapp.com",
  /http:\/\/localhost.*/
]

# mount ใน routes.rb
Rails.application.routes.draw do
  mount ActionCable.server => '/cable'
end
```

```javascript
// app/javascript/application.js
import { createConsumer } from "@rails/actioncable"
export default createConsumer()
```

## ขั้นตอนที่ 1163: สร้าง Channel

```ruby
# rails generate channel Chat
# สร้าง: app/channels/chat_channel.rb
# สร้าง: app/javascript/channels/chat_channel.js
```

```ruby
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
        User.find_by(id: user_id) || reject_unauthorized_connection
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    # รับ params[:room_id] จาก JavaScript
    @room = ChatRoom.find(params[:room_id])
    
    # ตรวจสอบสิทธิ์
    reject unless current_user.member_of?(@room)
    
    # Subscribe ไปยัง stream
    stream_for @room
    # หรือ stream ด้วย string key
    # stream_from "chat_room_#{@room.id}"
    
    # Broadcast เมื่อ user เข้าร่วม
    broadcast_presence(:online)
  end
  
  def unsubscribed
    # ทำงานเมื่อ browser disconnect
    broadcast_presence(:offline)
  end
  
  # รับ action จาก client
  def speak(data)
    Message.create!(
      body: data['message'],
      user: current_user,
      chat_room: @room
    )
  end
  
  def typing(data)
    ChatChannel.broadcast_to(@room, {
      type: "typing",
      user: current_user.name,
      is_typing: data['typing']
    })
  end
  
  private
  
  def broadcast_presence(status)
    ChatChannel.broadcast_to(@room, {
      type: "presence",
      user: current_user.name,
      status: status
    })
  end
end
```

## ขั้นตอนที่ 1164: JavaScript Subscription

```javascript
// app/javascript/channels/chat_channel.js
import consumer from "./consumer"

const chatChannel = consumer.subscriptions.create(
  { channel: "ChatChannel", room_id: roomId },
  {
    // Lifecycle callbacks
    connected() {
      console.log("Connected to ChatChannel")
      this.enableSendButton()
    },
    
    disconnected() {
      console.log("Disconnected from ChatChannel")
      this.disableSendButton()
    },
    
    rejected() {
      console.log("Connection rejected")
    },
    
    // รับข้อมูลจาก server
    received(data) {
      switch(data.type) {
        case "message":
          this.appendMessage(data)
          break
        case "typing":
          this.showTypingIndicator(data)
          break
        case "presence":
          this.updatePresence(data)
          break
      }
    },
    
    // ส่งข้อมูลไป server
    speak(message) {
      this.perform("speak", { message })
    },
    
    typing(isTyping) {
      this.perform("typing", { typing: isTyping })
    },
    
    // UI helpers
    appendMessage(data) {
      const messages = document.getElementById("messages")
      messages.insertAdjacentHTML("beforeend", `
        <div class="message ${data.current_user ? 'mine' : 'theirs'}">
          <span class="name">${data.user}</span>
          <p>${data.body}</p>
          <small>${data.created_at}</small>
        </div>
      `)
      messages.scrollTop = messages.scrollHeight
    },
    
    showTypingIndicator(data) {
      const indicator = document.getElementById("typing-indicator")
      if (data.is_typing) {
        indicator.textContent = `${data.user} กำลังพิมพ์...`
      } else {
        indicator.textContent = ""
      }
    },
    
    enableSendButton() {
      document.getElementById("send-btn").disabled = false
    },
    
    disableSendButton() {
      document.getElementById("send-btn").disabled = true
    }
  }
)

// Event listeners
document.getElementById("message-form").addEventListener("submit", (e) => {
  e.preventDefault()
  const input = document.getElementById("message-input")
  const message = input.value.trim()
  
  if (message) {
    chatChannel.speak(message)
    input.value = ""
  }
})

let typingTimer
document.getElementById("message-input").addEventListener("input", () => {
  chatChannel.typing(true)
  clearTimeout(typingTimer)
  typingTimer = setTimeout(() => chatChannel.typing(false), 1000)
})

export default chatChannel
```

## ขั้นตอนที่ 1165: Broadcasting

```ruby
# Broadcast จาก Model callback
class Message < ApplicationRecord
  belongs_to :user
  belongs_to :chat_room
  
  after_create_commit :broadcast_message
  
  private
  
  def broadcast_message
    ChatChannel.broadcast_to(
      chat_room,
      {
        type: "message",
        id: id,
        body: body,
        user: user.name,
        user_id: user_id,
        avatar_url: user.avatar_url,
        created_at: created_at.strftime("%H:%M"),
        html: ApplicationController.renderer.render(
          partial: "messages/message",
          locals: { message: self }
        )
      }
    )
  end
end
```

```ruby
# Broadcast จาก controller หรือ service
class MessagesController < ApplicationController
  def create
    @message = current_user.messages.build(message_params)
    @message.chat_room = @chat_room
    
    if @message.save
      # Broadcast จะเกิดขึ้นอัตโนมัติจาก after_create_commit
      render json: { status: "ok" }
    else
      render json: { errors: @message.errors }, status: :unprocessable_entity
    end
  end
end

# Broadcast จาก background job
class NotificationJob < ApplicationJob
  def perform(user_id, notification_data)
    ActionCable.server.broadcast(
      "notifications_#{user_id}",
      notification_data
    )
  end
end
```

## ขั้นตอนที่ 1166: Turbo Streams กับ Action Cable

```ruby
# app/channels/turbo_streams_channel.rb (ใช้ gem hotwire-rails)
# Turbo จัดการให้อัตโนมัติ

# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :chat_room
  belongs_to :user
  
  # Broadcast Turbo Stream
  after_create_commit -> { broadcast_append_to chat_room }
  after_update_commit -> { broadcast_replace_to chat_room }
  after_destroy_commit -> { broadcast_remove_to chat_room }
end
```

```erb
<%# app/views/chat_rooms/show.html.erb %>
<%= turbo_stream_from @chat_room %>

<div id="messages">
  <%= render @chat_room.messages.recent %>
</div>

<%= turbo_frame_tag "new_message" do %>
  <%= render "messages/form", chat_room: @chat_room, message: Message.new %>
<% end %>
```

```erb
<%# app/views/messages/_message.html.erb %>
<%= turbo_frame_tag message do %>
  <div class="message" id="message_<%= message.id %>">
    <strong><%= message.user.name %>:</strong>
    <%= message.body %>
    <small><%= message.created_at.strftime("%H:%M") %></small>
  </div>
<% end %>
```

## ขั้นตอนที่ 1167: Chat App สมบูรณ์

```ruby
# app/models/chat_room.rb
class ChatRoom < ApplicationRecord
  has_many :messages, dependent: :destroy
  has_many :memberships, dependent: :destroy
  has_many :users, through: :memberships
  
  validates :name, presence: true, uniqueness: true
  
  def self.general
    find_or_create_by(name: "General", public: true)
  end
end
```

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :chat_room
  belongs_to :user
  
  validates :body, presence: true, length: { maximum: 1000 }
  
  scope :recent, -> { order(created_at: :asc).last(50) }
  
  after_create_commit :broadcast_new_message
  
  private
  
  def broadcast_new_message
    ChatChannel.broadcast_to(
      chat_room,
      type: "message",
      id: id,
      body: ActionController::Base.helpers.sanitize(body),
      user: user.name,
      user_id: user_id,
      created_at: created_at.strftime("%H:%M น.")
    )
  end
end
```

```ruby
# app/controllers/chat_rooms_controller.rb
class ChatRoomsController < ApplicationController
  before_action :authenticate_user!
  
  def show
    @chat_room = ChatRoom.find(params[:id])
    @messages = @chat_room.messages.includes(:user).recent
    @message = Message.new
    
    # Mark as read
    current_user.mark_room_as_read(@chat_room)
  end
end

# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_chat_room
  
  def create
    @message = @chat_room.messages.build(message_params)
    @message.user = current_user
    
    if @message.save
      head :ok
    else
      render json: { errors: @message.errors.full_messages }, status: :unprocessable_entity
    end
  end
  
  private
  
  def set_chat_room
    @chat_room = ChatRoom.find(params[:chat_room_id])
  end
  
  def message_params
    params.require(:message).permit(:body)
  end
end
```

## ขั้นตอนที่ 1168: Notifications Channel

```ruby
# app/channels/notifications_channel.rb
class NotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
    
    # Mark as connected
    current_user.update(online_at: Time.current)
  end
  
  def unsubscribed
    current_user.update(online_at: nil)
  end
end
```

```ruby
# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :user
  belongs_to :notifiable, polymorphic: true
  
  after_create_commit :broadcast_notification
  
  private
  
  def broadcast_notification
    NotificationsChannel.broadcast_to(
      user,
      {
        id: id,
        type: notification_type,
        message: message,
        url: notifiable_url,
        created_at: created_at.to_s
      }
    )
  end
  
  def notifiable_url
    Rails.application.routes.url_helpers.url_for(notifiable)
  rescue
    "/"
  end
end
```

## ขั้นตอนที่ 1169: Online Presence

```ruby
# app/channels/presence_channel.rb
class PresenceChannel < ApplicationCable::Channel
  def subscribed
    stream_from "presence_channel"
    
    # เพิ่ม user ใน Redis set
    Redis.current.sadd("online_users", current_user.id)
    
    # Broadcast
    broadcast_presence_update
  end
  
  def unsubscribed
    Redis.current.srem("online_users", current_user.id)
    broadcast_presence_update
  end
  
  private
  
  def broadcast_presence_update
    online_users = User.where(
      id: Redis.current.smembers("online_users").map(&:to_i)
    ).map { |u| { id: u.id, name: u.name } }
    
    ActionCable.server.broadcast("presence_channel", {
      type: "presence_update",
      online_users: online_users,
      count: online_users.length
    })
  end
end
```

## ขั้นตอนที่ 1170: Testing Action Cable

```ruby
# spec/channels/chat_channel_spec.rb
RSpec.describe ChatChannel, type: :channel do
  let(:user) { create(:user) }
  let(:room) { create(:chat_room) }
  
  before do
    stub_connection current_user: user
  end
  
  describe "#subscribed" do
    it "subscribes to room stream" do
      subscribe(room_id: room.id)
      
      expect(subscription).to be_confirmed
      expect(streams).to include("chat_room_#{room.id}")
    end
    
    it "rejects if not a member" do
      another_room = create(:chat_room, private: true)
      subscribe(room_id: another_room.id)
      
      expect(subscription).to be_rejected
    end
  end
  
  describe "#speak" do
    before { subscribe(room_id: room.id) }
    
    it "creates a message" do
      expect {
        perform(:speak, message: "Hello!")
      }.to change(Message, :count).by(1)
    end
    
    it "broadcasts the message" do
      perform(:speak, message: "Hello World!")
      
      expect(broadcasts("chat_room_#{room.id}")).not_to be_empty
    end
  end
end
```

---

## แบบฝึกหัด: WebSockets (20 ข้อ)

### ข้อที่ 1: Generate Channel
```bash
rails generate channel Chat speak
```

### ข้อที่ 2: Connection Authentication
```
เชื่อม current_user กับ Connection class
```

### ข้อที่ 3: Chat Room
```
สร้าง chat channel ที่รับ room_id params
```

### ข้อที่ 4: Turbo Streams Broadcast
```
ใช้ broadcast_append_to เพื่อ append messages
```

### ข้อที่ 5: Typing Indicator
```
สร้าง typing indicator ที่แสดงเมื่อ user พิมพ์
```

### ข้อที่ 6-20 (แบบสรุป)

**ข้อ 6:** Notification channel
**ข้อ 7:** Online presence tracking
**ข้อ 8:** Read receipts
**ข้อ 9:** Unread message count
**ข้อ 10:** Room member list
**ข้อ 11:** File sharing ใน chat
**ข้อ 12:** Message reactions (emoji)
**ข้อ 13:** Private messaging
**ข้อ 14:** Channel testing
**ข้อ 15:** Redis Pub/Sub setup
**ข้อ 16:** Rate limiting connections
**ข้อ 17:** Message pagination (load more)
**ข้อ 18:** Push notifications (Web Push)
**ข้อ 19:** Reconnection logic
**ข้อ 20:** Deploy Action Cable with Puma

---

## สรุป: Action Cable

| Component | หน้าที่ |
|-----------|--------|
| Connection | Authenticate WebSocket connection |
| Channel | Handle subscriptions & actions |
| Stream | Send data to specific subscribers |
| Broadcast | Push data to all channel subscribers |
| Consumer | JavaScript client |
| Subscription | JavaScript channel handler |

**Key Takeaways:**
1. ใช้ Redis adapter ใน production
2. Authenticate ใน Connection class
3. Turbo Streams ทำงานร่วมกับ Action Cable
4. ทดสอบ channel ด้วย `stub_connection`
5. Monitor WebSocket connections ใน production

---

## Step 1171: WebSocket Authentication ด้วย JWT

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user
    
    def connect
      self.current_user = find_verified_user
    end
    
    private
    
    def find_verified_user
      # Option 1: Cookie-based
      if (user_id = cookies.encrypted[:user_id])
        User.find_by(id: user_id) || reject_unauthorized_connection
      # Option 2: JWT token in query string
      elsif (token = request.params[:token])
        verify_jwt_token(token)
      # Option 3: Session
      elsif (user_id = env["warden"]&.user&.id)
        User.find_by(id: user_id) || reject_unauthorized_connection
      else
        reject_unauthorized_connection
      end
    end
    
    def verify_jwt_token(token)
      payload = JWT.decode(
        token,
        Rails.application.secret_key_base,
        true,
        algorithm: 'HS256'
      ).first
      
      User.find_by(id: payload['user_id']) || reject_unauthorized_connection
    rescue JWT::DecodeError
      reject_unauthorized_connection
    end
  end
end
```

```javascript
// app/javascript/cable.js - เชื่อม WebSocket พร้อม JWT
import { createConsumer } from "@rails/actioncable"

function getAuthToken() {
  return document.querySelector('meta[name="jwt-token"]')?.content
}

// สร้าง consumer พร้อม token
export default createConsumer(() => {
  const token = getAuthToken()
  return token ? `/cable?token=${token}` : "/cable"
})
```

---

## Step 1172: Rate Limiting WebSocket Connections

```ruby
# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
    private
    
    def rate_limit!(action, limit: 10, period: 60)
      key = "rate_limit:#{current_user.id}:#{action}"
      count = Redis.current.incr(key)
      Redis.current.expire(key, period) if count == 1
      
      if count > limit
        transmit({ error: "Rate limit exceeded. Please slow down." })
        throw :abort
      end
    end
  end
end

# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def speak(data)
    rate_limit!(:speak, limit: 20, period: 60)
    
    Message.create!(
      body: data['message'],
      user: current_user,
      chat_room: @room
    )
  end
end
```

---

## Step 1173: Action Cable กับ Sidekiq

```ruby
# app/jobs/broadcast_notification_job.rb
class BroadcastNotificationJob < ApplicationJob
  queue_as :cable
  
  def perform(user_id, payload)
    user = User.find(user_id)
    
    ActionCable.server.broadcast(
      "notifications_user_#{user_id}",
      payload
    )
    
    # Log the broadcast
    NotificationLog.create!(
      user: user,
      payload: payload,
      broadcast_at: Time.current
    )
  end
end

# เรียกใช้
BroadcastNotificationJob.perform_later(user.id, {
  type: 'alert',
  message: 'คุณมีข้อความใหม่',
  url: '/messages'
})
```

---

## Step 1174: Turbo Streams กับ Rails 7

```ruby
# config/routes.rb
resources :posts do
  resources :comments
end

# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user
  
  after_create_commit -> {
    broadcast_append_to(
      post,
      target: "comments",
      partial: "comments/comment",
      locals: { comment: self }
    )
  }
  
  after_update_commit -> {
    broadcast_replace_to(
      post,
      target: self,
      partial: "comments/comment",
      locals: { comment: self }
    )
  }
  
  after_destroy_commit -> {
    broadcast_remove_to(post, target: self)
  }
end

# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  before_action :authenticate_user!
  
  def create
    @post = Post.find(params[:post_id])
    @comment = @post.comments.build(comment_params)
    @comment.user = current_user
    
    if @comment.save
      # broadcast จะเกิดขึ้นอัตโนมัติจาก after_create_commit
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to @post }
      end
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  private
  
  def comment_params
    params.require(:comment).permit(:body)
  end
end
```

```erb
<%# app/views/posts/show.html.erb %>
<%= turbo_stream_from @post %>

<div id="post_<%= @post.id %>">
  <h1><%= @post.title %></h1>
  <p><%= @post.body %></p>
</div>

<div id="comments">
  <%= render @post.comments %>
</div>

<%= turbo_frame_tag "new_comment" do %>
  <%= render "comments/form", post: @post, comment: Comment.new %>
<% end %>
```

```erb
<%# app/views/comments/_comment.html.erb %>
<%= turbo_frame_tag comment do %>
  <div class="comment" id="comment_<%= comment.id %>">
    <strong><%= comment.user.name %></strong>
    <p><%= comment.body %></p>
    <small><%= time_ago_in_words(comment.created_at) %> ที่แล้ว</small>
    
    <% if can?(:destroy, comment) %>
      <%= button_to "ลบ", post_comment_path(@post, comment),
          method: :delete,
          data: { turbo_confirm: "แน่ใจหรือไม่?" } %>
    <% end %>
  </div>
<% end %>
```

---

## Step 1175: Real-time Dashboard

```ruby
# app/channels/dashboard_channel.rb
class DashboardChannel < ApplicationCable::Channel
  def subscribed
    return reject unless current_user.admin?
    
    stream_from "admin_dashboard"
    
    # ส่งข้อมูลเริ่มต้น
    transmit(current_stats)
  end
  
  private
  
  def current_stats
    {
      type: 'stats',
      users_count: User.count,
      orders_today: Order.today.count,
      revenue_today: Order.today.sum(:total).to_f,
      online_users: Redis.current.scard("online_users")
    }
  end
end

# app/jobs/dashboard_update_job.rb
class DashboardUpdateJob < ApplicationJob
  queue_as :default
  
  def perform
    stats = {
      type: 'stats',
      users_count: User.count,
      orders_today: Order.today.count,
      revenue_today: Order.today.sum(:total).to_f,
      online_users: Redis.current.scard("online_users"),
      updated_at: Time.current.strftime("%H:%M:%S")
    }
    
    ActionCable.server.broadcast("admin_dashboard", stats)
  end
end

# config/initializers/scheduled_jobs.rb
# รัน dashboard update ทุก 30 วินาที
# (ใช้ sidekiq-cron หรือ whenever)
```

```javascript
// app/javascript/dashboard.js
import consumer from "./cable"

const dashboardChannel = consumer.subscriptions.create("DashboardChannel", {
  received(data) {
    if (data.type === 'stats') {
      document.getElementById('users-count').textContent = data.users_count
      document.getElementById('orders-today').textContent = data.orders_today
      document.getElementById('revenue-today').textContent = 
        `฿${data.revenue_today.toLocaleString()}`
      document.getElementById('online-users').textContent = data.online_users
    }
  }
})
```

---

## Step 1176: WebSocket Reconnection Logic

```javascript
// app/javascript/cable_manager.js
import { createConsumer } from "@rails/actioncable"

class CableManager {
  constructor() {
    this.consumer = null
    this.subscriptions = new Map()
    this.reconnectAttempts = 0
    this.maxReconnectAttempts = 5
  }
  
  connect() {
    this.consumer = createConsumer()
    this.consumer.connection.monitor.reconnectDelay = (retries) => {
      // Exponential backoff: 1s, 2s, 4s, 8s, 16s
      return Math.min(Math.pow(2, retries) * 1000, 30000)
    }
    return this.consumer
  }
  
  subscribe(channelName, params, callbacks) {
    const subscription = this.consumer.subscriptions.create(
      { channel: channelName, ...params },
      {
        connected: () => {
          this.reconnectAttempts = 0
          callbacks.connected?.()
          this.showConnectionStatus('connected')
        },
        disconnected: () => {
          callbacks.disconnected?.()
          this.showConnectionStatus('disconnected')
          this.attemptReconnect(channelName, params, callbacks)
        },
        received: callbacks.received
      }
    )
    
    this.subscriptions.set(channelName, subscription)
    return subscription
  }
  
  attemptReconnect(channelName, params, callbacks) {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++
      const delay = Math.min(Math.pow(2, this.reconnectAttempts) * 1000, 30000)
      
      setTimeout(() => {
        if (!this.consumer.connection.isOpen()) {
          this.consumer.connect()
        }
      }, delay)
    }
  }
  
  showConnectionStatus(status) {
    const el = document.getElementById('connection-status')
    if (!el) return
    
    el.className = `connection-status ${status}`
    el.textContent = status === 'connected' ? '● เชื่อมต่อแล้ว' : '○ ขาดการเชื่อมต่อ'
  }
}

export default new CableManager()
```

---

## Step 1177: Multiple Consumers

```javascript
// app/javascript/channels/consumer.js
import { createConsumer } from "@rails/actioncable"
export const consumer = createConsumer()

// สร้าง channel แยกแต่ละ feature
// app/javascript/channels/chat_channel.js
import { consumer } from "./consumer"

export function createChatSubscription(roomId, handlers) {
  return consumer.subscriptions.create(
    { channel: "ChatChannel", room_id: roomId },
    {
      connected() { handlers.onConnect?.() },
      disconnected() { handlers.onDisconnect?.() },
      received(data) { handlers.onMessage?.(data) }
    }
  )
}

// app/javascript/channels/notifications_channel.js
import { consumer } from "./consumer"

export function createNotificationsSubscription(handlers) {
  return consumer.subscriptions.create("NotificationsChannel", {
    received(data) { handlers.onNotification?.(data) }
  })
}
```

---

## Step 1178: WebSocket Performance

```ruby
# config/cable.yml - tuning สำหรับ production
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") %>
  channel_prefix: <%= ENV.fetch("APP_NAME", "myapp") %>_production

# Puma configuration สำหรับ Action Cable
# config/puma.rb
workers Integer(ENV.fetch('WEB_CONCURRENCY', 2))
threads_count = Integer(ENV.fetch('RAILS_MAX_THREADS', 5))
threads threads_count, threads_count

# Action Cable ต้องการ thread-safe configuration
preload_app!

# app/config/initializers/action_cable.rb
ActionCable.server.config.worker_pool_size = 4

# ตรวจสอบ connections
ActionCable.server.config.log_tags = [:action_cable]
```

```ruby
# ลด memory ด้วยการ limit connections
# config/initializers/action_cable.rb
module ActionCable
  module Server
    class Configuration
      def max_connections
        ENV.fetch('ACTION_CABLE_MAX_CONNECTIONS', 1000).to_i
      end
    end
  end
end
```

---

## Step 1179: Monitoring WebSockets

```ruby
# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
    def subscribe_to_stream(stream_name)
      stream_from stream_name
      
      # Log subscription
      Rails.logger.info "[ActionCable] #{current_user.id} subscribed to #{stream_name}"
      
      # Monitor via StatsD
      StatsD.increment("cable.subscriptions.#{self.class.name.underscore}")
      StatsD.gauge("cable.connections.active", ActionCable.server.connections.length)
    end
  end
end
```

---

## Step 1180: แบบฝึกหัดเพิ่มเติม

### ข้อ 1: ระบบ Collaborative Editing

```ruby
# app/channels/document_channel.rb
class DocumentChannel < ApplicationCable::Channel
  def subscribed
    @document = Document.find(params[:document_id])
    reject unless @document.can_edit?(current_user)
    
    stream_for @document
    
    # แจ้งผู้ร่วมแก้ไขว่ามีคนเข้ามา
    DocumentChannel.broadcast_to(@document, {
      type: 'user_joined',
      user: { id: current_user.id, name: current_user.name }
    })
  end
  
  def update(data)
    @document.update!(content: data['content'])
    
    DocumentChannel.broadcast_to(@document, {
      type: 'content_update',
      content: data['content'],
      updated_by: current_user.name,
      cursor_position: data['cursor_position']
    })
  end
  
  def cursor_moved(data)
    DocumentChannel.broadcast_to(@document, {
      type: 'cursor_moved',
      user_id: current_user.id,
      user_name: current_user.name,
      position: data['position'],
      color: current_user.cursor_color
    })
  end
end
```

### ข้อ 2: Stock Price Updates

```ruby
# app/jobs/stock_price_update_job.rb
class StockPriceUpdateJob < ApplicationJob
  queue_as :realtime
  
  def perform
    StockSymbol.active.each do |stock|
      price = fetch_price(stock.symbol)
      stock.update!(current_price: price, updated_at: Time.current)
      
      ActionCable.server.broadcast("stock_#{stock.symbol}", {
        symbol: stock.symbol,
        price: price,
        change: price - stock.previous_close,
        change_percent: ((price - stock.previous_close) / stock.previous_close * 100).round(2)
      })
    end
  end
  
  private
  
  def fetch_price(symbol)
    # เรียก external API
    response = HTTP.get("https://api.stockdata.com/#{symbol}")
    response.parse['price']
  end
end
```

### ข้อ 3: Game Channel

```ruby
# app/channels/game_channel.rb
class GameChannel < ApplicationCable::Channel
  GAME_TIMEOUT = 30.minutes
  
  def subscribed
    @game = Game.find(params[:game_id])
    reject unless @game.can_join?(current_user)
    
    stream_for @game
    @game.add_player!(current_user)
    
    GameChannel.broadcast_to(@game, {
      type: 'player_joined',
      player: { id: current_user.id, name: current_user.name },
      players_count: @game.players.count
    })
  end
  
  def make_move(data)
    result = @game.make_move!(
      player: current_user,
      move: data['move']
    )
    
    GameChannel.broadcast_to(@game, {
      type: 'move_made',
      move: data['move'],
      player: current_user.name,
      board_state: result[:board],
      game_over: result[:game_over],
      winner: result[:winner]&.name
    })
  end
end
```

---

## สรุป Action Cable ขั้นสูง

| Pattern | Use Case |
|---------|----------|
| stream_for | Object-specific streams |
| stream_from | String key streams |
| broadcast_to | Send to object stream |
| ActionCable.server.broadcast | Send to string stream |
| after_create_commit | Auto-broadcast on model create |
| Turbo Streams | Rails 7 real-time HTML updates |

**Best Practices:**
1. ใช้ JWT สำหรับ authentication เมื่อไม่มี session cookies
2. Rate limit เพื่อป้องกัน WebSocket flooding
3. ใช้ Redis adapter ใน production เสมอ
4. Monitor จำนวน connections ด้วย StatsD/Prometheus
5. Test channels ด้วย `stub_connection` helper
6. ใช้ Sidekiq job สำหรับ broadcasting หนัก
7. Implement exponential backoff สำหรับ reconnection
