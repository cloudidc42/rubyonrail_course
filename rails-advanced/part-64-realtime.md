# Part 64: Real-time Features ด้วย Hotwire และ Action Cable

## ขั้นตอนที่ 1391-1410: สร้าง Real-time Applications

---

## ขั้นตอนที่ 1391: Hotwire คืออะไร?

Hotwire คือ framework สำหรับสร้าง real-time HTML over-the-wire โดยไม่ต้องใช้ JSON API

**ประกอบด้วย:**
1. **Turbo** - สำหรับ navigation และ real-time updates
   - Turbo Drive - SPA-like navigation
   - Turbo Frames - อัปเดต DOM บางส่วน
   - Turbo Streams - Server-sent real-time updates
2. **Stimulus** - JavaScript framework แบบ modest

### ติดตั้ง Hotwire

```ruby
# Gemfile (Rails 7+ มาพร้อมกัน)
gem 'turbo-rails'
gem 'stimulus-rails'
```

```bash
bundle install
rails turbo:install
rails stimulus:install
```

---

## ขั้นตอนที่ 1392: Turbo Drive

Turbo Drive ทำให้ page navigation เร็วโดยไม่ reload ทั้งหน้า

```html
<!-- ทำงานอัตโนมัติ - ไม่ต้องทำอะไรเพิ่ม -->
<a href="/posts">ไปหน้า Posts</a>

<!-- ปิด Turbo Drive สำหรับ link นี้ -->
<a href="/posts" data-turbo="false">ไม่ใช้ Turbo</a>

<!-- ปิดทั้งหมดใน section -->
<div data-turbo="false">
  <a href="/login">Login</a>
</div>
```

```javascript
// ฟัง events
document.addEventListener("turbo:before-visit", (event) => {
  console.log("About to visit:", event.detail.url);
})

document.addEventListener("turbo:load", () => {
  console.log("Page loaded");
  initializeComponents();
})
```

---

## ขั้นตอนที่ 1393: Turbo Frames

Turbo Frames อัปเดต DOM เฉพาะส่วนที่ต้องการ

```erb
<%# app/views/posts/index.html.erb %>
<h1>บทความทั้งหมด</h1>

<turbo-frame id="new-post">
  <%= link_to "เขียนบทความใหม่", new_post_path %>
</turbo-frame>

<turbo-frame id="posts-list">
  <%= render @posts %>
</turbo-frame>
```

```erb
<%# app/views/posts/new.html.erb %>
<turbo-frame id="new-post">
  <h2>บทความใหม่</h2>
  
  <%= form_with model: @post do |f| %>
    <%= f.text_field :title, placeholder: "หัวข้อบทความ" %>
    <%= f.text_area :body %>
    <%= f.submit "สร้างบทความ" %>
  <% end %>
</turbo-frame>
```

### Lazy Loading Frames

```erb
<%# Load content ตอนที่ frame ปรากฏ %>
<turbo-frame id="stats" src="/dashboard/stats" loading="lazy">
  <div class="loading-spinner">กำลังโหลด...</div>
</turbo-frame>
```

### Frame Targeting

```erb
<%# คลิก link นี้จะ update frame "sidebar" %>
<a href="/notifications" data-turbo-frame="sidebar">
  การแจ้งเตือน
</a>

<%# คลิก link แล้ว navigate ทั้งหน้า (ออกจาก frame) %>
<a href="/posts/1" data-turbo-frame="_top">
  อ่านบทความเต็ม
</a>
```

---

## ขั้นตอนที่ 1394: Turbo Streams

Turbo Streams อัปเดต DOM หลายส่วนพร้อมกัน

### Actions ของ Turbo Stream

```
append   - เพิ่มท้าย target
prepend  - เพิ่มต้น target
replace  - แทนที่ target ทั้งหมด
update   - อัปเดต content ใน target
remove   - ลบ target
before   - แทรกก่อน target
after    - แทรกหลัง target
```

### Turbo Stream ใน Controller (HTTP Response)

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.new(post_params)
    
    if @post.save
      respond_to do |format|
        format.turbo_stream do
          render turbo_stream: [
            turbo_stream.prepend("posts-list", partial: "posts/post", locals: { post: @post }),
            turbo_stream.update("posts-count", @post.user.posts.count),
            turbo_stream.update("flash-messages", partial: "layouts/flash",
                              locals: { message: "สร้างบทความสำเร็จ!" })
          ]
        end
        format.html { redirect_to @post }
      end
    else
      respond_to do |format|
        format.turbo_stream do
          render turbo_stream: turbo_stream.update(
            "post-form",
            partial: "posts/form",
            locals: { post: @post }
          )
        end
        format.html { render :new }
      end
    end
  end
  
  def destroy
    @post = Post.find(params[:id])
    authorize @post
    @post.destroy
    
    respond_to do |format|
      format.turbo_stream { render turbo_stream: turbo_stream.remove("post_#{@post.id}") }
      format.html { redirect_to posts_path }
    end
  end
end
```

```erb
<%# app/views/posts/_post.html.erb %>
<div id="post_<%= post.id %>">
  <h2><%= post.title %></h2>
  <p><%= post.excerpt %></p>
  
  <%= button_to "ลบ", post_path(post), method: :delete,
    data: { turbo_confirm: "ยืนยันการลบบทความนี้?" } %>
</div>
```

---

## ขั้นตอนที่ 1395: Action Cable Real-time Chat

### ตั้งค่า Action Cable

```ruby
# config/cable.yml
development:
  adapter: async

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV['REDIS_URL'] %>
  channel_prefix: myapp_production
```

### Chat Channel

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    @room = Room.find(params[:room_id])
    
    if current_user.can_access?(@room)
      stream_for @room
      # หรือ: stream_from "chat:#{@room.id}"
    else
      reject
    end
  end
  
  def unsubscribed
    # clean up
    @room.users.delete(current_user) if @room
  end
  
  def speak(data)
    message = @room.messages.create!(
      user: current_user,
      body: data['message']
    )
    
    # Broadcast ไปทุกคนใน room
    ChatChannel.broadcast_to(
      @room,
      {
        type: "message",
        message: message_data(message)
      }
    )
  end
  
  private
  
  def message_data(message)
    {
      id: message.id,
      body: message.body,
      user: {
        id: message.user.id,
        name: message.user.name,
        avatar_url: message.user.avatar_url
      },
      created_at: message.created_at.strftime("%H:%M")
    }
  end
end
```

### Connection Authentication

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Server::Base::Connection
    identified_by :current_user
    
    def connect
      self.current_user = find_verified_user
    end
    
    private
    
    def find_verified_user
      # Session-based auth (สำหรับ browser)
      if (verified_user = env['warden']&.user)
        verified_user
      # Token-based auth (สำหรับ API)
      elsif (token = request.params[:token])
        User.find_by(auth_token: token)
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

### JavaScript Subscriber

```javascript
// app/javascript/channels/chat_channel.js
import consumer from "channels/consumer"

const chatChannel = consumer.subscriptions.create(
  { channel: "ChatChannel", room_id: roomId },
  {
    connected() {
      console.log("Connected to chat channel")
      this.installEventHandlers()
    },
    
    disconnected() {
      console.log("Disconnected")
    },
    
    received(data) {
      if (data.type === "message") {
        this.appendMessage(data.message)
      }
    },
    
    speak(message) {
      this.perform("speak", { message })
    },
    
    appendMessage(message) {
      const messagesContainer = document.getElementById("messages")
      const messageEl = this.buildMessageElement(message)
      messagesContainer.insertAdjacentHTML("beforeend", messageEl)
      messagesContainer.scrollTop = messagesContainer.scrollHeight
    },
    
    buildMessageElement(message) {
      return `
        <div class="message" id="message-${message.id}">
          <img src="${message.user.avatar_url}" class="avatar" />
          <div class="message-content">
            <strong>${message.user.name}</strong>
            <span class="time">${message.created_at}</span>
            <p>${message.body}</p>
          </div>
        </div>
      `
    },
    
    installEventHandlers() {
      const form = document.getElementById("chat-form")
      const input = document.getElementById("message-input")
      
      form.addEventListener("submit", (e) => {
        e.preventDefault()
        const message = input.value.trim()
        if (message) {
          this.speak(message)
          input.value = ""
        }
      })
    }
  }
)
```

---

## ขั้นตอนที่ 1396: Real-time Notifications

```ruby
# app/channels/notifications_channel.rb
class NotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
  end
  
  def unsubscribed
    # nothing
  end
  
  def mark_as_read(data)
    notification = current_user.notifications.find(data['notification_id'])
    notification.mark_as_read!
    
    # Update badge count
    NotificationsChannel.broadcast_to(
      current_user,
      {
        type: "badge_update",
        count: current_user.unread_notifications_count
      }
    )
  end
end
```

```ruby
# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :user
  belongs_to :notifiable, polymorphic: true
  
  after_create :broadcast_notification
  
  scope :unread, -> { where(read_at: nil) }
  
  def mark_as_read!
    update!(read_at: Time.current)
  end
  
  private
  
  def broadcast_notification
    NotificationsChannel.broadcast_to(
      user,
      {
        type: "notification",
        id: id,
        title: title,
        body: body,
        url: url,
        created_at: created_at.strftime("%H:%M"),
        unread_count: user.notifications.unread.count
      }
    )
  end
end
```

```javascript
// app/javascript/channels/notifications_channel.js
import consumer from "channels/consumer"

consumer.subscriptions.create("NotificationsChannel", {
  received(data) {
    switch (data.type) {
      case "notification":
        this.showNotification(data)
        this.updateBadge(data.unread_count)
        break
      case "badge_update":
        this.updateBadge(data.count)
        break
    }
  },
  
  showNotification(notification) {
    // Desktop notification
    if (Notification.permission === "granted") {
      new Notification(notification.title, {
        body: notification.body,
        icon: "/notification-icon.png"
      })
    }
    
    // In-app notification toast
    const toast = document.createElement("div")
    toast.className = "notification-toast"
    toast.innerHTML = `
      <strong>${notification.title}</strong>
      <p>${notification.body}</p>
    `
    document.body.appendChild(toast)
    setTimeout(() => toast.remove(), 5000)
  },
  
  updateBadge(count) {
    const badge = document.getElementById("notification-badge")
    if (badge) {
      badge.textContent = count
      badge.style.display = count > 0 ? "block" : "none"
    }
  }
})
```

---

## ขั้นตอนที่ 1397: Broadcasting จาก Jobs

```ruby
# app/jobs/process_order_job.rb
class ProcessOrderJob < ApplicationJob
  queue_as :orders
  
  def perform(order_id)
    order = Order.find(order_id)
    
    # อัปเดต status ระหว่าง process
    broadcast_status_update(order, "processing", 0)
    
    order.update!(status: "processing")
    
    # Charge payment
    broadcast_status_update(order, "charging", 30)
    
    PaymentService.new.charge(order)
    
    # Update inventory
    broadcast_status_update(order, "updating_inventory", 60)
    
    InventoryService.reduce_stock(order)
    
    # Send confirmation
    broadcast_status_update(order, "sending_confirmation", 90)
    
    OrderMailer.confirmation(order).deliver_now
    
    # Done
    order.update!(status: "confirmed")
    broadcast_status_update(order, "confirmed", 100)
    
  rescue PaymentError => e
    order.update!(status: "payment_failed")
    broadcast_error(order, "การชำระเงินล้มเหลว: #{e.message}")
  end
  
  private
  
  def broadcast_status_update(order, status, progress)
    ActionCable.server.broadcast(
      "order_#{order.id}",
      {
        type: "status_update",
        status: status,
        progress: progress,
        message: status_message(status)
      }
    )
  end
  
  def broadcast_error(order, message)
    ActionCable.server.broadcast(
      "order_#{order.id}",
      {
        type: "error",
        message: message
      }
    )
  end
  
  def status_message(status)
    {
      "processing" => "กำลังดำเนินการ...",
      "charging" => "กำลังชำระเงิน...",
      "updating_inventory" => "กำลังอัปเดตสต็อก...",
      "sending_confirmation" => "กำลังส่งอีเมลยืนยัน...",
      "confirmed" => "สั่งซื้อสำเร็จ! 🎉"
    }[status]
  end
end
```

```ruby
# app/channels/order_channel.rb
class OrderChannel < ApplicationCable::Channel
  def subscribed
    order = Order.find(params[:order_id])
    
    if current_user == order.user
      stream_from "order_#{order.id}"
    else
      reject
    end
  end
end
```

---

## ขั้นตอนที่ 1398: Stimulus Controllers

```javascript
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["content", "button"]
  static classes = ["hidden"]
  
  connect() {
    this.hiddenClass = this.hasHiddenClass ? this.hiddenClasses[0] : "hidden"
  }
  
  toggle() {
    this.contentTarget.classList.toggle(this.hiddenClass)
    
    const isHidden = this.contentTarget.classList.contains(this.hiddenClass)
    this.buttonTarget.textContent = isHidden ? "แสดง" : "ซ่อน"
  }
}
```

```erb
<%# ใช้งาน %>
<div data-controller="toggle">
  <button data-toggle-target="button" data-action="click->toggle#toggle">
    ซ่อน
  </button>
  
  <div data-toggle-target="content">
    เนื้อหาที่ซ่อนได้
  </div>
</div>
```

### Stimulus สำหรับ Live Search

```javascript
// app/javascript/controllers/search_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "results"]
  
  connect() {
    this.timeout = null
  }
  
  search() {
    clearTimeout(this.timeout)
    
    this.timeout = setTimeout(() => {
      const query = this.inputTarget.value.trim()
      
      if (query.length < 2) {
        this.resultsTarget.innerHTML = ""
        return
      }
      
      this.performSearch(query)
    }, 300)  // Debounce 300ms
  }
  
  async performSearch(query) {
    this.resultsTarget.innerHTML = '<div class="loading">กำลังค้นหา...</div>'
    
    try {
      const response = await fetch(`/search?q=${encodeURIComponent(query)}`, {
        headers: {
          "Accept": "text/vnd.turbo-stream.html",
          "X-Requested-With": "XMLHttpRequest"
        }
      })
      
      if (response.ok) {
        const html = await response.text()
        this.resultsTarget.innerHTML = html
      }
    } catch (error) {
      this.resultsTarget.innerHTML = '<div class="error">เกิดข้อผิดพลาด</div>'
    }
  }
  
  clear() {
    this.inputTarget.value = ""
    this.resultsTarget.innerHTML = ""
  }
}
```

```erb
<%# app/views/layouts/_search.html.erb %>
<div data-controller="search">
  <input 
    type="text" 
    placeholder="ค้นหา..."
    data-search-target="input"
    data-action="input->search#search"
  />
  
  <button data-action="click->search#clear">ล้าง</button>
  
  <div data-search-target="results" id="search-results">
    <%# Results will appear here %>
  </div>
</div>
```

---

## ขั้นตอนที่ 1399: Turbo Streams กับ Broadcasting

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user
  
  after_create_commit :broadcast_created
  after_destroy_commit :broadcast_destroyed
  
  private
  
  def broadcast_created
    broadcast_prepend_to(
      post,
      target: "comments",
      partial: "comments/comment",
      locals: { comment: self }
    )
    
    # อัปเดต count
    broadcast_update_to(
      post,
      target: "comments-count",
      html: post.comments.count.to_s
    )
  end
  
  def broadcast_destroyed
    broadcast_remove_to(post)
    
    broadcast_update_to(
      post,
      target: "comments-count",
      html: post.comments.count.to_s
    )
  end
end
```

```erb
<%# app/views/posts/show.html.erb %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>

<div class="comments-section">
  <h3>ความคิดเห็น (<span id="comments-count"><%= @post.comments.count %></span>)</h3>
  
  <%= turbo_stream_from @post %>
  
  <%= form_with model: [@post, Comment.new], id: "new-comment-form" do |f| %>
    <%= f.text_area :body, placeholder: "เขียนความคิดเห็น..." %>
    <%= f.submit "ส่งความคิดเห็น" %>
  <% end %>
  
  <div id="comments">
    <%= render @post.comments.order(created_at: :desc) %>
  </div>
</div>
```

---

## ขั้นตอนที่ 1400: Turbo + Devise

### Setup Turbo กับ Devise

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?
  
  protected
  
  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name, :username])
    devise_parameter_sanitizer.permit(:account_update, keys: [:name, :avatar])
  end
  
  # Handle Turbo Stream requests
  def after_sign_in_path_for(resource)
    stored_location_for(resource) || dashboard_path
  end
end
```

```ruby
# app/controllers/users/sessions_controller.rb
class Users::SessionsController < Devise::SessionsController
  def create
    super do |resource|
      if resource.persisted?
        respond_to do |format|
          format.html
          format.turbo_stream do
            render turbo_stream: [
              turbo_stream.update("flash", partial: "layouts/flash",
                                locals: { notice: "ยินดีต้อนรับ #{resource.name}!" }),
              turbo_stream.update("nav", partial: "layouts/nav",
                                locals: { current_user: resource })
            ]
          end
        end
      end
    end
  end
end
```

---

## แบบฝึกหัดที่ 1-20

### แบบฝึกหัดที่ 1
ตั้งค่า Hotwire ใน Rails 7 app

### แบบฝึกหัดที่ 2
สร้าง CRUD ด้วย Turbo Frames (ไม่ reload หน้า)

### แบบฝึกหัดที่ 3
สร้าง Live Comment System ด้วย Turbo Streams

### แบบฝึกหัดที่ 4
สร้าง Real-time Chat ด้วย Action Cable

### แบบฝึกหัดที่ 5
สร้าง Notification System ด้วย Action Cable

### แบบฝึกหัดที่ 6
สร้าง Live Search ด้วย Stimulus + Turbo

### แบบฝึกหัดที่ 7
สร้าง Real-time Order Tracking

### แบบฝึกหัดที่ 8
สร้าง Typing Indicator ในห้อง Chat

### แบบฝึกหัดที่ 9
สร้าง Live Dashboard ที่อัปเดต stats อัตโนมัติ

### แบบฝึกหัดที่ 10
สร้าง Optimistic UI updates

### แบบฝึกหัดที่ 11
ตั้งค่า Action Cable Authentication ด้วย JWT

### แบบฝึกหัดที่ 12
สร้าง Stimulus Controller สำหรับ Form Validation

### แบบฝึกหัดที่ 13
สร้าง Infinite Scroll ด้วย Turbo Streams

### แบบฝึกหัดที่ 14
สร้าง Collaborative Editing (หลาย users แก้ไขพร้อมกัน)

### แบบฝึกหัดที่ 15
สร้าง Real-time Polling/Voting

### แบบฝึกหัดที่ 16
สร้าง Progress Bar สำหรับ Long-running Tasks

### แบบฝึกหัดที่ 17
เพิ่ม Desktop Notifications

### แบบฝึกหัดที่ 18
สร้าง Online Status Indicator

### แบบฝึกหัดที่ 19
ตั้งค่า Action Cable กับ Redis ใน Production

### แบบฝึกหัดที่ 20
สร้าง Complete Real-time App:
- Live Chat
- Notifications
- Real-time stats
- Collaborative editing
- Presence indicators

---

## สรุป Part 64

เราได้เรียนรู้:
1. Hotwire framework (Turbo + Stimulus)
2. Turbo Drive สำหรับ smooth navigation
3. Turbo Frames สำหรับ partial updates
4. Turbo Streams สำหรับ server-sent updates
5. Action Cable สำหรับ WebSocket communication
6. Real-time Chat
7. Notification System
8. Broadcasting จาก Background Jobs
9. Stimulus Controllers
10. Integration กับ Devise

Hotwire + Action Cable ช่วยให้สร้าง real-time features ได้โดยไม่ต้องเขียน JavaScript มาก
