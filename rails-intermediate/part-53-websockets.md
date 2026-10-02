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
