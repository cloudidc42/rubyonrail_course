# ตอนที่ 53: WebSockets กับ Action Cable (Steps 1161-1180)

## บทนำ

WebSockets ช่วยให้ server สามารถ push ข้อมูลไปยัง client ได้ทันทีโดยไม่ต้องรอ client ส่ง request ก่อน Action Cable คือ Rails framework สำหรับ WebSockets ที่รวม real-time features เข้ากับ Rails ได้อย่าง seamless

---

## Step 1161: WebSockets คืออะไร?

### HTTP vs WebSocket

```
HTTP (Traditional):
Client → Server: "GET /messages"
Server → Client: [list of messages]
(connection closed)

Client → Server: "GET /messages" (polling every 2 seconds)
Server → Client: [new messages if any]
...

WebSocket (Persistent):
Client → Server: "WebSocket Handshake"
Server → Client: "Handshake confirmed"
[Connection stays open...]
Server → Client: "New message!" (anytime)
Server → Client: "User joined!" (anytime)
Client → Server: "Message sent" (anytime)
```

### ใช้ WebSocket เมื่อไหร่

- Chat applications
- Real-time notifications
- Collaborative editing
- Live dashboards
- Online games
- Real-time tracking (delivery, location)
- Live sports scores

---

## Step 1162: Action Cable Architecture

### Components

```
Browser (Consumer)
    ↕ WebSocket
Action Cable Server
    ↕ pub/sub
Redis (or PostgreSQL)
    ↕ subscribe
Channel (Ruby class)
    ↕
Rails Application
```

### Key Concepts

- **Consumer** - client ที่ connect ผ่าน WebSocket
- **Connection** - WebSocket connection สำหรับแต่ละ consumer
- **Channel** - ช่องทางสำหรับ group ของ streams
- **Subscription** - consumer subscribe ไปยัง channel
- **Stream** - data stream ชื่อ string
- **Broadcast** - ส่ง data ไปยัง stream

---

## Step 1163: Setup Action Cable

### Configuration

```ruby
# config/application.rb
config.action_cable.mount_path = '/cable'

# config/cable.yml
development:
  adapter: async  # In-process (development only)

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: myapp_production
```

```ruby
# config/environments/production.rb
config.action_cable.url = "wss://myapp.com/cable"
config.action_cable.allowed_request_origins = [
  "https://myapp.com",
  /http:\/\/myapp.*/  # regex
]
```

### JavaScript Setup

```javascript
// app/javascript/application.js
import { createConsumer } from "@rails/actioncable"

const consumer = createConsumer()

export default consumer
```

---

## Step 1164: Creating a Channel

### Generate Channel

```bash
rails generate channel Chat
rails generate channel Notification
rails generate channel Room speaking
```

### Channel Class

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user
    
    def connect
      self.current_user = find_verified_user
      logger.add_tags "ActionCable", "User #{current_user.id}"
    end
    
    def disconnect
      # Called when connection is closed
      logger.info "User #{current_user&.id} disconnected"
    end
    
    private
    
    def find_verified_user
      # ตรวจสอบ session (สำหรับ browser-based)
      if (current_user = env["warden"].user)
        current_user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

```ruby
# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
    # Base class สำหรับทุก channels
  end
end
```

---

## Step 1165: Chat Application Complete

### Models

```ruby
# app/models/room.rb
class Room < ApplicationRecord
  has_many :messages, dependent: :destroy
  has_many :room_memberships, dependent: :destroy
  has_many :users, through: :room_memberships
  
  validates :name, presence: true, uniqueness: true, length: { maximum: 100 }
  
  scope :public_rooms, -> { where(private: false) }
  scope :recent, -> { order(updated_at: :desc) }
end

# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :user
  belongs_to :room
  
  validates :body, presence: true, length: { maximum: 2000 }
  
  after_create_commit :broadcast_message
  
  def as_json(options = {})
    super(only: [:id, :body, :created_at]).merge(
      user: { id: user.id, name: user.name, avatar_url: user.avatar_url },
      formatted_time: created_at.strftime("%H:%M"),
      is_own: options[:current_user] == user
    )
  end
  
  private
  
  def broadcast_message
    ActionCable.server.broadcast(
      "room_#{room_id}",
      {
        type: 'new_message',
        message: as_json
      }
    )
  end
end
```

### Chat Channel

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    @room = Room.find_by(id: params[:room_id])
    
    if @room && can_access_room?(@room)
      stream_for @room
      # หรือ stream_from "room_#{@room.id}"
      
      # Notify others ว่า user เข้าร่วม
      ActionCable.server.broadcast(
        "room_#{@room.id}",
        {
          type: 'user_joined',
          user: { id: current_user.id, name: current_user.name }
        }
      )
      
      # Track online users
      $redis.sadd("online_users_room_#{@room.id}", current_user.id)
      
      transmit({
        type: 'connected',
        message: "Connected to #{@room.name}",
        online_users: online_users_count
      })
    else
      reject  # Reject subscription
    end
  end
  
  def unsubscribed
    # Cleanup เมื่อ disconnect
    if @room
      $redis.srem("online_users_room_#{@room.id}", current_user.id)
      
      ActionCable.server.broadcast(
        "room_#{@room.id}",
        {
          type: 'user_left',
          user: { id: current_user.id, name: current_user.name }
        }
      )
    end
    stop_all_streams
  end
  
  def receive(data)
    # Receive data from client
    case data['action']
    when 'message'
      create_message(data['message'])
    when 'typing'
      broadcast_typing_indicator(data['is_typing'])
    when 'mark_read'
      mark_messages_read
    end
  end
  
  # Custom actions
  def load_history(data)
    page = data['page'] || 1
    messages = @room.messages.includes(:user).order(created_at: :desc).page(page).per(30)
    
    transmit({
      type: 'history',
      messages: messages.map { |m| m.as_json(current_user: current_user) },
      has_more: messages.next_page.present?
    })
  end
  
  private
  
  def create_message(body)
    return if body.blank?
    
    message = @room.messages.create!(
      user: current_user,
      body: body.strip.truncate(2000)
    )
    
    # Broadcast อยู่ใน after_create_commit callback
  end
  
  def broadcast_typing_indicator(is_typing)
    ActionCable.server.broadcast(
      "room_#{@room.id}",
      {
        type: 'typing',
        user: { id: current_user.id, name: current_user.name },
        is_typing: is_typing
      }
    )
  end
  
  def mark_messages_read
    current_user.mark_room_messages_read!(@room)
  end
  
  def can_access_room?(room)
    !room.private? || room.users.include?(current_user)
  end
  
  def online_users_count
    $redis.scard("online_users_room_#{@room.id}").to_i
  end
end
```

### JavaScript Consumer

```javascript
// app/javascript/channels/chat_channel.js
import consumer from "./consumer"

let chatChannel = null

function initChat(roomId) {
  // Unsubscribe from previous room
  if (chatChannel) {
    chatChannel.unsubscribe()
  }
  
  chatChannel = consumer.subscriptions.create(
    { channel: "ChatChannel", room_id: roomId },
    {
      connected() {
        console.log("Connected to chat")
        document.querySelector(".connection-status").textContent = "🟢 Connected"
        this.loadHistory({ page: 1 })
      },
      
      disconnected() {
        console.log("Disconnected from chat")
        document.querySelector(".connection-status").textContent = "🔴 Disconnected"
      },
      
      received(data) {
        switch (data.type) {
          case 'new_message':
            appendMessage(data.message)
            break
          case 'user_joined':
            showSystemMessage(`${data.user.name} เข้าร่วมห้องแชท`)
            updateOnlineCount()
            break
          case 'user_left':
            showSystemMessage(`${data.user.name} ออกจากห้องแชท`)
            updateOnlineCount()
            break
          case 'typing':
            showTypingIndicator(data.user, data.is_typing)
            break
          case 'history':
            prependMessages(data.messages, data.has_more)
            break
          case 'connected':
            document.querySelector(".online-count").textContent = data.online_users
            break
        }
      },
      
      sendMessage(message) {
        this.perform('receive', { action: 'message', message: message })
      },
      
      sendTyping(isTyping) {
        this.perform('receive', { action: 'typing', is_typing: isTyping })
      },
      
      loadHistory(params) {
        this.perform('load_history', params)
      }
    }
  )
  
  return chatChannel
}

function appendMessage(message) {
  const messagesContainer = document.getElementById("messages")
  const messageEl = createMessageElement(message)
  messagesContainer.appendChild(messageEl)
  messagesContainer.scrollTop = messagesContainer.scrollHeight
}

function createMessageElement(message) {
  const div = document.createElement("div")
  div.className = `message ${message.is_own ? 'own' : 'other'}`
  div.innerHTML = `
    <div class="message-header">
      <img src="${message.user.avatar_url}" alt="${message.user.name}" class="avatar">
      <span class="user-name">${message.user.name}</span>
      <span class="time">${message.formatted_time}</span>
    </div>
    <div class="message-body">${escapeHtml(message.body)}</div>
  `
  return div
}

function showSystemMessage(text) {
  const div = document.createElement("div")
  div.className = "system-message"
  div.textContent = text
  document.getElementById("messages").appendChild(div)
}

let typingTimeout = null
function showTypingIndicator(user, isTyping) {
  const indicator = document.getElementById("typing-indicator")
  
  if (isTyping) {
    indicator.textContent = `${user.name} กำลังพิมพ์...`
    clearTimeout(typingTimeout)
    typingTimeout = setTimeout(() => {
      indicator.textContent = ""
    }, 3000)
  } else {
    indicator.textContent = ""
  }
}

function escapeHtml(text) {
  const div = document.createElement('div')
  div.appendChild(document.createTextNode(text))
  return div.innerHTML
}

// Event Listeners
document.addEventListener("DOMContentLoaded", function() {
  const roomId = document.querySelector("[data-room-id]")?.dataset.roomId
  if (!roomId) return
  
  const channel = initChat(roomId)
  const messageInput = document.getElementById("message-input")
  const sendButton = document.getElementById("send-button")
  
  // Send message
  function sendMessage() {
    const message = messageInput.value.trim()
    if (message) {
      channel.sendMessage(message)
      messageInput.value = ""
      channel.sendTyping(false)
    }
  }
  
  sendButton.addEventListener("click", sendMessage)
  
  messageInput.addEventListener("keypress", function(e) {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault()
      sendMessage()
    }
  })
  
  // Typing indicator
  let typingTimer = null
  messageInput.addEventListener("input", function() {
    channel.sendTyping(true)
    clearTimeout(typingTimer)
    typingTimer = setTimeout(() => channel.sendTyping(false), 1000)
  })
  
  // Load more history on scroll
  const messagesContainer = document.getElementById("messages")
  let currentPage = 1
  let hasMore = true
  
  messagesContainer.addEventListener("scroll", function() {
    if (messagesContainer.scrollTop === 0 && hasMore) {
      currentPage++
      channel.loadHistory({ page: currentPage })
    }
  })
})

export { initChat }
```

### Chat Views

```erb
<%# app/views/rooms/show.html.erb %>
<div class="chat-container" data-room-id="<%= @room.id %>">
  <div class="chat-header">
    <h2><%= @room.name %></h2>
    <span class="connection-status">🔄 Connecting...</span>
    <span class="online-count">-</span> คนออนไลน์
  </div>
  
  <div id="messages" class="messages-container">
    <% @messages.each do |message| %>
      <div class="message <%= message.user == current_user ? 'own' : 'other' %>">
        <div class="message-header">
          <img src="<%= message.user.avatar_url || gravatar_url(message.user.email) %>" 
               alt="<%= message.user.name %>"
               class="avatar">
          <span class="user-name"><%= message.user.name %></span>
          <span class="time"><%= message.created_at.strftime("%H:%M") %></span>
        </div>
        <div class="message-body"><%= message.body %></div>
      </div>
    <% end %>
  </div>
  
  <div id="typing-indicator" class="typing-indicator"></div>
  
  <div class="message-input-container">
    <textarea id="message-input" 
              placeholder="พิมพ์ข้อความ... (Enter เพื่อส่ง)"
              class="message-input"
              rows="1"></textarea>
    <button id="send-button" class="send-button">ส่ง</button>
  </div>
</div>

<script>
  // Auto-scroll to bottom on load
  const messages = document.getElementById("messages")
  messages.scrollTop = messages.scrollHeight
</script>
```

---

## Step 1166: Broadcasting from Server

### Broadcast ใน Controller

```ruby
# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  def create
    @message = @room.messages.new(message_params)
    @message.user = current_user
    
    if @message.save
      # Broadcast อัตโนมัติผ่าน after_create_commit ใน model
      # หรือ broadcast manually:
      ActionCable.server.broadcast(
        "room_#{@room.id}",
        { type: 'new_message', message: @message.as_json(current_user: current_user) }
      )
      
      render json: { success: true, message: @message.as_json }
    else
      render json: { success: false, errors: @message.errors.full_messages }, 
             status: :unprocessable_entity
    end
  end
end
```

### Broadcast ใน Model Callbacks

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  after_create_commit :broadcast_new_message
  after_update_commit :broadcast_updated_message
  after_destroy_commit :broadcast_deleted_message
  
  private
  
  def broadcast_new_message
    ChatChannel.broadcast_to(room, {
      type: 'new_message',
      message: as_json
    })
  end
  
  def broadcast_updated_message
    ChatChannel.broadcast_to(room, {
      type: 'message_updated',
      message_id: id,
      new_body: body
    })
  end
  
  def broadcast_deleted_message
    ChatChannel.broadcast_to(room, {
      type: 'message_deleted',
      message_id: id
    })
  end
end
```

---

## Step 1167: Turbo Streams

### Real-time Updates ด้วย Turbo

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  after_create_commit lambda { 
    broadcast_prepend_to "messages",
                         target: "messages",
                         partial: "messages/message",
                         locals: { message: self }
  }
  
  after_update_commit lambda {
    broadcast_replace_to "messages",
                         target: self,
                         partial: "messages/message",
                         locals: { message: self }
  }
  
  after_destroy_commit lambda {
    broadcast_remove_to "messages", target: self
  }
end
```

```erb
<%# app/views/messages/index.html.erb %>
<%= turbo_stream_from "messages" %>

<div id="messages">
  <%= render @messages %>
</div>
```

```erb
<%# app/views/messages/_message.html.erb %>
<div id="<%= dom_id(message) %>" class="message">
  <strong><%= message.user.name %></strong>
  <span><%= message.body %></span>
  <time><%= message.created_at.strftime("%H:%M") %></time>
</div>
```

---

## Step 1168: Real-time Notifications

### Notification Channel

```ruby
# app/channels/notification_channel.rb
class NotificationChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
  end
  
  def unsubscribed
    stop_all_streams
  end
  
  # MarkNotificationAsRead
  def mark_read(data)
    notification = current_user.notifications.find_by(id: data['notification_id'])
    notification&.update!(read: true, read_at: Time.current)
    
    transmit({
      type: 'notification_read',
      unread_count: current_user.notifications.unread.count
    })
  end
  
  def mark_all_read
    current_user.notifications.unread.update_all(read: true, read_at: Time.current)
    
    transmit({
      type: 'all_read',
      unread_count: 0
    })
  end
end
```

```ruby
# app/services/notification_service.rb
class NotificationService
  def self.notify(user, type:, title:, body:, data: {})
    notification = user.notifications.create!(
      notification_type: type,
      title: title,
      body: body,
      data: data
    )
    
    # Push ผ่าน Action Cable
    NotificationChannel.broadcast_to(user, {
      type: 'new_notification',
      notification: {
        id: notification.id,
        type: notification.notification_type,
        title: notification.title,
        body: notification.body,
        created_at: notification.created_at.iso8601,
        data: notification.data
      },
      unread_count: user.notifications.unread.count
    })
    
    # Optional: Push notification ไปยัง mobile
    if user.push_token.present?
      PushNotificationService.send(user, title, body)
    end
    
    notification
  end
end
```

```javascript
// app/javascript/channels/notification_channel.js
import consumer from "./consumer"

const notificationChannel = consumer.subscriptions.create(
  { channel: "NotificationChannel" },
  {
    connected() {
      console.log("Connected to notifications")
    },
    
    received(data) {
      switch (data.type) {
        case 'new_notification':
          showToast(data.notification)
          updateNotificationBadge(data.unread_count)
          addNotificationToList(data.notification)
          break
          
        case 'notification_read':
        case 'all_read':
          updateNotificationBadge(data.unread_count)
          break
      }
    },
    
    markRead(notificationId) {
      this.perform('mark_read', { notification_id: notificationId })
    },
    
    markAllRead() {
      this.perform('mark_all_read')
    }
  }
)

function showToast(notification) {
  const toast = document.createElement("div")
  toast.className = "toast notification-toast"
  toast.innerHTML = `
    <div class="toast-header">
      <strong>${notification.title}</strong>
      <button class="btn-close" onclick="this.closest('.toast').remove()">×</button>
    </div>
    <div class="toast-body">${notification.body}</div>
  `
  
  document.getElementById("toast-container").appendChild(toast)
  
  setTimeout(() => toast.remove(), 5000)
}

function updateNotificationBadge(count) {
  const badge = document.getElementById("notification-badge")
  if (badge) {
    badge.textContent = count
    badge.hidden = count === 0
  }
}

export { notificationChannel }
```

---

## Step 1169: Authentication ใน Action Cable

### JWT Authentication

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
      # Method 1: จาก session (Devise)
      if (user = env["warden"]&.user)
        return user
      end
      
      # Method 2: จาก JWT token
      token = request.params[:token] || 
              request.headers['Authorization']&.split(' ')&.last
      
      if token.present?
        payload = JwtService.decode(token)
        if payload && (user = User.find_by(id: payload['user_id']))
          return user
        end
      end
      
      # Method 3: จาก cookies
      if (token = cookies.signed[:auth_token])
        if (user = User.find_by(auth_token: token))
          return user
        end
      end
      
      reject_unauthorized_connection
    end
  end
end
```

### Client ส่ง JWT Token

```javascript
// Connect กับ JWT token
import { createConsumer } from "@rails/actioncable"

const token = localStorage.getItem('auth_token')
const consumer = createConsumer(`/cable?token=${token}`)
```

---

## Step 1170: Testing Action Cable

### Channel Tests

```ruby
# spec/channels/chat_channel_spec.rb
require 'rails_helper'

RSpec.describe ChatChannel, type: :channel do
  let(:user) { create(:user) }
  let(:room) { create(:room) }
  
  before do
    stub_connection current_user: user
  end
  
  describe "#subscribed" do
    it "subscribes to the room" do
      subscribe(room_id: room.id)
      
      expect(subscription).to be_confirmed
      expect(streams).to include("room_#{room.id}")
    end
    
    it "rejects invalid room subscription" do
      subscribe(room_id: 99999)
      
      expect(subscription).to be_rejected
    end
    
    it "transmits connected message" do
      expect {
        subscribe(room_id: room.id)
      }.to have_broadcasted_to(room).from_channel(ChatChannel)
    end
  end
  
  describe "#receive" do
    before { subscribe(room_id: room.id) }
    
    context "with message action" do
      it "creates a message" do
        expect {
          perform(:receive, action: 'message', message: 'Hello!')
        }.to change(Message, :count).by(1)
      end
      
      it "broadcasts the message to the room" do
        expect {
          perform(:receive, action: 'message', message: 'Hello!')
        }.to have_broadcasted_to(room)
      end
      
      it "ignores blank messages" do
        expect {
          perform(:receive, action: 'message', message: '')
        }.not_to change(Message, :count)
      end
    end
  end
  
  describe "#unsubscribed" do
    before { subscribe(room_id: room.id) }
    
    it "stops all streams" do
      unsubscribe
      expect(streams).to be_empty
    end
    
    it "broadcasts user left notification" do
      expect {
        unsubscribe
      }.to have_broadcasted_to(room)
    end
  end
end
```

---

## แบบฝึกหัด (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** สร้าง ChatChannel ด้วย generator
```bash
# เฉลย
rails generate channel Chat
```

**ข้อ 2:** ตั้งค่า Action Cable ให้ใช้ Redis ใน production
```yaml
# เฉลย - config/cable.yml
production:
  adapter: redis
  url: <%= ENV['REDIS_URL'] %>
  channel_prefix: myapp_production
```

**ข้อ 3:** สร้าง Connection class ที่ authenticate ด้วย Devise session
```ruby
# เฉลย
def find_verified_user
  if (user = env["warden"].user)
    user
  else
    reject_unauthorized_connection
  end
end
```

**ข้อ 4:** Broadcast message จาก model callback
```ruby
# เฉลย
class Message < ApplicationRecord
  after_create_commit { broadcast_append_to "messages" }
end
```

**ข้อ 5:** สร้าง JavaScript subscription
```javascript
// เฉลย
const channel = consumer.subscriptions.create("ChatChannel", {
  received(data) { console.log(data) }
})
```

### ระดับกลาง

**ข้อ 6:** เพิ่ม typing indicator ใน chat
```ruby
# เฉลย - channel
def typing(data)
  ActionCable.server.broadcast("room_#{@room.id}", {
    type: 'typing',
    user: current_user.name,
    is_typing: data['is_typing']
  })
end
```

**ข้อ 7:** implement online users counter
```ruby
# เฉลย
def subscribed
  $redis.sadd("online_users", current_user.id)
  broadcast_online_count
end

def unsubscribed
  $redis.srem("online_users", current_user.id)
  broadcast_online_count
end

def broadcast_online_count
  ActionCable.server.broadcast("presence", {
    online_count: $redis.scard("online_users")
  })
end
```

**ข้อ 8:** เพิ่ม message read receipts
```javascript
// เฉลย
document.addEventListener('scroll', function() {
  const visibleMessages = getVisibleMessages()
  visibleMessages.forEach(msg => {
    channel.perform('mark_read', { message_id: msg.id })
  })
})
```

**ข้อ 9:** เขียน channel test ด้วย RSpec
```ruby
# เฉลย
RSpec.describe ChatChannel, type: :channel do
  let(:user) { create(:user) }
  before { stub_connection current_user: user }
  
  it "subscribes successfully" do
    subscribe(room_id: create(:room).id)
    expect(subscription).to be_confirmed
  end
end
```

**ข้อ 10:** implement reconnection logic ใน JavaScript
```javascript
// เฉลย
consumer.subscriptions.create("ChatChannel", {
  disconnected() {
    this.retryConnection()
  },
  retryConnection() {
    setTimeout(() => {
      if (this.consumer.connection.disconnected) {
        this.consumer.connect()
      }
    }, 3000)
  }
})
```

### ระดับสูง

**ข้อ 11-20:**

```ruby
# เฉลย ข้อ 11 - Private messaging channel
class DirectMessageChannel < ApplicationCable::Channel
  def subscribed
    @other_user = User.find(params[:user_id])
    room_name = [current_user.id, @other_user.id].sort.join("_")
    stream_from "direct_message_#{room_name}"
  end
end
```

```ruby
# เฉลย ข้อ 14 - Rate limiting
class ChatChannel < ApplicationCable::Channel
  RATE_LIMIT = 5  # messages per second
  
  def receive(data)
    key = "rate_limit_#{current_user.id}"
    count = $redis.incr(key)
    $redis.expire(key, 1) if count == 1
    
    if count > RATE_LIMIT
      transmit({ type: 'error', message: "ส่งข้อความเร็วเกินไป กรุณารอสักครู่" })
      return
    end
    
    create_message(data['message'])
  end
end
```

```javascript
// เฉลย ข้อ 17 - Message queue for offline
class OfflineMessageQueue {
  constructor() {
    this.queue = JSON.parse(localStorage.getItem('pending_messages') || '[]')
  }
  
  add(message) {
    this.queue.push(message)
    localStorage.setItem('pending_messages', JSON.stringify(this.queue))
  }
  
  flush(channel) {
    this.queue.forEach(msg => channel.sendMessage(msg))
    this.queue = []
    localStorage.removeItem('pending_messages')
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **WebSockets** - persistent connection ระหว่าง client และ server
2. **Action Cable Architecture** - Consumer, Channel, Stream
3. **Creating Channels** - channel class สำหรับ different features
4. **Broadcasting** - ส่ง data จาก server ไปยัง clients
5. **Complete Chat App** - message, typing indicator, online users
6. **Turbo Streams** - real-time DOM updates
7. **Notifications** - real-time notification system
8. **Authentication** - ตรวจสอบ identity ใน WebSocket
9. **Testing** - test channels ด้วย RSpec

Action Cable เป็นวิธีที่ elegant ในการเพิ่ม real-time features ให้กับ Rails applications โดยรวมเข้ากับ ecosystem ของ Rails ได้ดีมาก
