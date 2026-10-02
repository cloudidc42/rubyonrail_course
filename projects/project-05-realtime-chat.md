# Project 5: Real-time Chat Application

## ระบบ Chat แบบ Real-time ที่สมบูรณ์

---

## ภาพรวม

เราจะสร้าง **ChatApp** - ระบบ Chat แบบ Real-time ที่มี:
- WebSocket ด้วย Action Cable
- Turbo Streams สำหรับ Real-time updates
- Public และ Private Rooms
- Online/Offline Status
- Message History
- File Attachments
- Push Notifications
- Responsive Design ด้วย Tailwind CSS

---

## ขั้นตอนที่ 1: Setup Project

```bash
rails new chatapp \
  --database=postgresql \
  --css=tailwind \
  --javascript=importmap

cd chatapp

# เพิ่ม gems
bundle add devise
bundle add image_processing   # สำหรับ resize images
bundle add redis
bundle add turbo-rails
bundle add stimulus-rails

# ตั้งค่า Redis
# config/cable.yml
```

---

## ขั้นตอนที่ 2: Database Schema

```ruby
# db/migrate/001_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users do |t|
      t.string  :email,              null: false
      t.string  :encrypted_password, null: false
      t.string  :username,           null: false
      t.string  :display_name
      t.string  :status_message
      t.string  :online_status,      null: false, default: 'offline'
      t.datetime :last_seen_at
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at
      t.timestamps

      t.index :email,    unique: true
      t.index :username, unique: true
    end
  end
end

# db/migrate/002_create_rooms.rb
class CreateRooms < ActiveRecord::Migration[7.1]
  def change
    create_table :rooms do |t|
      t.string     :name,        null: false
      t.string     :room_type,   null: false, default: 'public'
      t.text       :description
      t.references :created_by,  null: false, foreign_key: { to_table: :users }
      t.boolean    :archived,    null: false, default: false
      t.timestamps

      t.index :room_type
      t.index :name
    end
  end
end

# db/migrate/003_create_room_memberships.rb
class CreateRoomMemberships < ActiveRecord::Migration[7.1]
  def change
    create_table :room_memberships do |t|
      t.references :room, null: false, foreign_key: true
      t.references :user, null: false, foreign_key: true
      t.string     :role,          null: false, default: 'member'
      t.datetime   :last_read_at
      t.boolean    :notifications, null: false, default: true
      t.timestamps

      t.index [:room_id, :user_id], unique: true
    end
  end
end

# db/migrate/004_create_messages.rb
class CreateMessages < ActiveRecord::Migration[7.1]
  def change
    create_table :messages do |t|
      t.references :room, null: false, foreign_key: true
      t.references :user, null: false, foreign_key: true
      t.text        :content
      t.string      :message_type, null: false, default: 'text'
      t.references  :reply_to,     foreign_key: { to_table: :messages }
      t.boolean     :edited,       null: false, default: false
      t.boolean     :deleted,      null: false, default: false
      t.jsonb       :metadata,     default: {}
      t.timestamps

      t.index :created_at
      t.index [:room_id, :created_at]
    end
  end
end

# db/migrate/005_create_reactions.rb
class CreateReactions < ActiveRecord::Migration[7.1]
  def change
    create_table :reactions do |t|
      t.references :message, null: false, foreign_key: true
      t.references :user,    null: false, foreign_key: true
      t.string     :emoji,   null: false
      t.timestamps

      t.index [:message_id, :user_id, :emoji], unique: true, name: 'idx_reactions_unique'
    end
  end
end

# db/migrate/006_create_direct_messages.rb
class CreateDirectMessages < ActiveRecord::Migration[7.1]
  def change
    create_table :direct_messages do |t|
      t.references :sender,   null: false, foreign_key: { to_table: :users }
      t.references :receiver, null: false, foreign_key: { to_table: :users }
      t.text        :content,  null: false
      t.boolean     :read,     null: false, default: false
      t.datetime    :read_at
      t.timestamps

      t.index [:sender_id, :receiver_id, :created_at]
    end
  end
end

# db/migrate/007_create_notifications.rb
class CreateNotifications < ActiveRecord::Migration[7.1]
  def change
    create_table :notifications do |t|
      t.references :user,       null: false, foreign_key: true
      t.string     :ntype,      null: false
      t.string     :title,      null: false
      t.text       :body
      t.jsonb      :data,       default: {}
      t.boolean    :read,       null: false, default: false
      t.datetime   :read_at
      t.timestamps

      t.index [:user_id, :read, :created_at]
    end
  end
end
```

---

## ขั้นตอนที่ 3: Models

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  ONLINE_STATUSES = %w[online away busy offline].freeze

  has_many :created_rooms,    class_name: 'Room', foreign_key: :created_by_id
  has_many :room_memberships, dependent: :destroy
  has_many :rooms,            through: :room_memberships
  has_many :messages,         dependent: :nullify
  has_many :sent_dms,         class_name: 'DirectMessage', foreign_key: :sender_id
  has_many :received_dms,     class_name: 'DirectMessage', foreign_key: :receiver_id
  has_many :notifications,    dependent: :destroy
  has_many :reactions,        dependent: :destroy

  has_one_attached :avatar

  validates :username,      presence: true, uniqueness: true,
                            format: { with: /\A[a-z0-9_]+\z/i }
  validates :online_status, inclusion: { in: ONLINE_STATUSES }

  before_validation :set_display_name, on: :create

  def online?   = online_status == 'online'
  def offline?  = online_status == 'offline'
  def display   = display_name.presence || username

  def go_online!
    update_columns(online_status: 'online', last_seen_at: Time.current)
    broadcast_status_change
  end

  def go_offline!
    update_columns(online_status: 'offline', last_seen_at: Time.current)
    broadcast_status_change
  end

  def dm_conversation_with(other_user)
    DirectMessage.where(
      "(sender_id = ? AND receiver_id = ?) OR (sender_id = ? AND receiver_id = ?)",
      id, other_user.id, other_user.id, id
    ).order(created_at: :asc)
  end

  def unread_dm_count_from(other_user)
    received_dms.where(sender: other_user, read: false).count
  end

  def avatar_url
    if avatar.attached?
      Rails.application.routes.url_helpers.rails_blob_path(avatar, only_path: true)
    else
      "https://ui-avatars.com/api/?name=#{CGI.escape(display)}&background=random&size=128"
    end
  end

  private

  def set_display_name
    self.display_name ||= username
  end

  def broadcast_status_change
    # Broadcast ไปยัง users ทั้งหมด
    ActionCable.server.broadcast("presence_channel", {
      type:   'status_change',
      user:   { id: id, username: username, online_status: online_status, last_seen_at: last_seen_at }
    })
  end
end

# app/models/room.rb
class Room < ApplicationRecord
  ROOM_TYPES = %w[public private].freeze

  belongs_to :created_by, class_name: 'User'
  has_many   :room_memberships, dependent: :destroy
  has_many   :members,          through: :room_memberships, source: :user
  has_many   :messages,         dependent: :destroy

  validates :name,      presence: true, length: { minimum: 2, maximum: 50 }
  validates :room_type, inclusion: { in: ROOM_TYPES }

  scope :public_rooms,  -> { where(room_type: 'public', archived: false) }
  scope :active,        -> { where(archived: false) }
  scope :ordered,       -> { order(updated_at: :desc) }

  def public?   = room_type == 'public'
  def private?  = room_type == 'private'

  def member?(user)
    room_memberships.exists?(user: user)
  end

  def admin?(user)
    room_memberships.exists?(user: user, role: 'admin')
  end

  def unread_count_for(user)
    membership = room_memberships.find_by(user: user)
    return 0 unless membership

    if membership.last_read_at
      messages.where('created_at > ?', membership.last_read_at).count
    else
      messages.count
    end
  end

  def last_message
    messages.order(created_at: :desc).first
  end
end

# app/models/message.rb
class Message < ApplicationRecord
  MESSAGE_TYPES = %w[text image file system].freeze

  belongs_to :room
  belongs_to :user
  belongs_to :reply_to, class_name: 'Message', optional: true
  has_many   :reactions, dependent: :destroy
  has_one_attached :attachment

  validates :content,      presence: true, unless: :has_attachment?
  validates :message_type, inclusion: { in: MESSAGE_TYPES }

  scope :recent,       -> { order(created_at: :desc) }
  scope :visible,      -> { where(deleted: false) }
  scope :for_history,  ->(before: nil, limit: 50) {
    q = visible.order(created_at: :desc).limit(limit)
    q = q.where('created_at < ?', before) if before
    q.reverse
  }

  after_create_commit  :broadcast_create
  after_update_commit  :broadcast_update
  after_destroy_commit :broadcast_delete

  def soft_delete!
    update!(
      deleted:  true,
      content:  '[Message deleted]',
      metadata: metadata.merge('deleted_at' => Time.current.iso8601)
    )
  end

  def reactions_summary
    reactions.group(:emoji).count.map do |emoji, count|
      reacted = reactions.exists?(user: Current.user, emoji: emoji)
      { emoji: emoji, count: count, reacted_by_me: reacted }
    end
  end

  private

  def has_attachment?
    attachment.attached?
  end

  def broadcast_create
    # Broadcast ไปยัง room channel
    Turbo::StreamsChannel.broadcast_append_to(
      "room_#{room_id}",
      target:  "messages",
      partial: "messages/message",
      locals:  { message: self }
    )

    # Update room list สำหรับทุก members
    room.members.each do |member|
      Turbo::StreamsChannel.broadcast_replace_to(
        "room_list_#{member.id}",
        target:  "room_#{room_id}_preview",
        partial: "rooms/room_preview",
        locals:  { room: room, current_user: member }
      )
    end
  end

  def broadcast_update
    Turbo::StreamsChannel.broadcast_replace_to(
      "room_#{room_id}",
      target:  "message_#{id}",
      partial: "messages/message",
      locals:  { message: self }
    )
  end

  def broadcast_delete
    Turbo::StreamsChannel.broadcast_remove_to(
      "room_#{room_id}",
      target: "message_#{id}"
    )
  end
end
```

---

## ขั้นตอนที่ 4: Action Cable Channels

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
      current_user.go_online!
      logger.add_tags 'ActionCable', current_user.username
    end

    def disconnect
      current_user&.go_offline!
    end

    private

    def find_verified_user
      if session_user = User.find_by(id: cookies.encrypted[:user_id])
        session_user
      else
        reject_unauthorized_connection
      end
    end
  end
end

# app/channels/room_channel.rb
class RoomChannel < ApplicationCable::Channel
  def subscribed
    room = Room.find_by(id: params[:room_id])
    return reject unless room
    return reject unless room.member?(current_user) || room.public?

    stream_from "room_#{room.id}"

    # Mark as read
    membership = room.room_memberships.find_by(user: current_user)
    membership&.update_column(:last_read_at, Time.current)

    # Broadcast typing indicators
    @room = room
  end

  def unsubscribed
    stop_all_streams
  end

  def speak(data)
    room = Room.find_by(id: params[:room_id])
    return unless room&.member?(current_user)

    message_content = data['content']&.strip
    return if message_content.blank?

    message = room.messages.create!(
      user:         current_user,
      content:      message_content,
      message_type: 'text',
      reply_to_id:  data['reply_to_id']
    )

    # Update room timestamp
    room.touch
  end

  def start_typing(data)
    ActionCable.server.broadcast(
      "room_#{params[:room_id]}",
      {
        type:     'typing_start',
        user_id:  current_user.id,
        username: current_user.username
      }
    )
  end

  def stop_typing(data)
    ActionCable.server.broadcast(
      "room_#{params[:room_id]}",
      {
        type:    'typing_stop',
        user_id: current_user.id
      }
    )
  end

  def react(data)
    message = Message.find_by(id: data['message_id'])
    return unless message&.room == @room

    emoji = data['emoji']
    reaction = Reaction.find_by(message: message, user: current_user, emoji: emoji)

    if reaction
      reaction.destroy
      action = 'remove'
    else
      Reaction.create!(message: message, user: current_user, emoji: emoji)
      action = 'add'
    end

    ActionCable.server.broadcast(
      "room_#{params[:room_id]}",
      {
        type:       'reaction_update',
        message_id: message.id,
        emoji:      emoji,
        action:     action,
        user_id:    current_user.id,
        summary:    message.reactions_summary
      }
    )
  end
end

# app/channels/direct_message_channel.rb
class DirectMessageChannel < ApplicationCable::Channel
  def subscribed
    @other_user = User.find_by(id: params[:user_id])
    return reject unless @other_user

    # ใช้ sorted IDs เพื่อให้ channel name เดียวกัน
    ids = [current_user.id, @other_user.id].sort
    stream_from "dm_#{ids.join('_')}"
  end

  def send_message(data)
    content = data['content']&.strip
    return if content.blank?

    dm = DirectMessage.create!(
      sender:   current_user,
      receiver: @other_user,
      content:  content
    )

    ids = [current_user.id, @other_user.id].sort

    ActionCable.server.broadcast(
      "dm_#{ids.join('_')}",
      {
        type:    'new_dm',
        message: {
          id:         dm.id,
          content:    dm.content,
          sender_id:  dm.sender_id,
          created_at: dm.created_at.iso8601
        }
      }
    )

    # Send notification to receiver
    NotificationService.new(@other_user).notify_new_dm(dm)
  end

  def mark_read
    DirectMessage.where(sender: @other_user, receiver: current_user, read: false)
                 .update_all(read: true, read_at: Time.current)
  end
end

# app/channels/presence_channel.rb
class PresenceChannel < ApplicationCable::Channel
  def subscribed
    stream_from "presence_channel"

    # ส่ง list of online users ให้ subscriber ใหม่
    online_users = User.where(online_status: 'online').map do |u|
      { id: u.id, username: u.username, online_status: u.online_status }
    end

    transmit({ type: 'online_users', users: online_users })
  end

  def unsubscribed
    stop_all_streams
  end
end

# app/channels/notifications_channel.rb
class NotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_from "notifications_#{current_user.id}"
  end

  def mark_read(data)
    notification_id = data['notification_id']
    notification    = current_user.notifications.find_by(id: notification_id)
    notification&.update!(read: true, read_at: Time.current)

    transmit({ type: 'notification_read', id: notification_id })
  end
end
```

---

## ขั้นตอนที่ 5: Controllers

```ruby
# app/controllers/rooms_controller.rb
class RoomsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_room, only: [:show, :edit, :update, :destroy, :join, :leave]

  def index
    @public_rooms  = Room.public_rooms.includes(:members, :messages).ordered
    @my_rooms      = current_user.rooms.includes(:messages).ordered
    @online_users  = User.where(online_status: 'online').where.not(id: current_user.id)
  end

  def show
    unless @room.public? || @room.member?(current_user)
      redirect_to rooms_path, alert: "You don't have access to this room"
      return
    end

    # Mark messages as read
    membership = @room.room_memberships.find_by(user: current_user)
    membership&.update_column(:last_read_at, Time.current)

    @messages = @room.messages
                     .visible
                     .includes(:user, :reply_to, :reactions)
                     .for_history(limit: 50)
    @members  = @room.members.includes(:avatar_attachment)
  end

  def new
    @room = Room.new
  end

  def create
    @room = Room.new(room_params)
    @room.created_by = current_user

    if @room.save
      # Creator joins the room as admin
      @room.room_memberships.create!(
        user: current_user,
        role: 'admin'
      )

      # System message
      @room.messages.create!(
        user:         current_user,
        content:      "#{current_user.username} created this room",
        message_type: 'system'
      )

      redirect_to @room, notice: "Room '#{@room.name}' created!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def join
    if @room.public? && !@room.member?(current_user)
      @room.room_memberships.create!(user: current_user, role: 'member')
      @room.messages.create!(
        user:         current_user,
        content:      "#{current_user.username} joined the room",
        message_type: 'system'
      )
    end

    redirect_to @room
  end

  def leave
    membership = @room.room_memberships.find_by(user: current_user)

    if membership
      membership.destroy

      @room.messages.create!(
        user:         current_user,
        content:      "#{current_user.username} left the room",
        message_type: 'system'
      )
    end

    redirect_to rooms_path, notice: "You left #{@room.name}"
  end

  def destroy
    if @room.admin?(current_user) || current_user == @room.created_by
      @room.update!(archived: true)
      redirect_to rooms_path, notice: "Room '#{@room.name}' has been archived"
    else
      redirect_to @room, alert: "Only admins can archive rooms"
    end
  end

  private

  def set_room
    @room = Room.find(params[:id])
  rescue ActiveRecord::RecordNotFound
    redirect_to rooms_path, alert: "Room not found"
  end

  def room_params
    params.require(:room).permit(:name, :description, :room_type)
  end
end

# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  before_action :authenticate_user!

  def create
    @room    = Room.find(params[:room_id])
    @message = @room.messages.build(message_params)
    @message.user = current_user

    unless @room.member?(current_user) || @room.public?
      return head :forbidden
    end

    if @message.save
      # Turbo Stream response
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to @room }
      end
    else
      render json: { errors: @message.errors.full_messages }, status: :unprocessable_entity
    end
  end

  def destroy
    @message = Message.find(params[:id])
    @room    = @message.room

    unless @message.user == current_user || @room.admin?(current_user)
      return head :forbidden
    end

    @message.soft_delete!
    head :ok
  end

  def load_more
    @room     = Room.find(params[:room_id])
    before_id = params[:before_id]
    before_at = Message.find_by(id: before_id)&.created_at

    @messages = @room.messages
                     .visible
                     .includes(:user, :reply_to, :reactions)
                     .for_history(before: before_at, limit: 30)

    render json: {
      messages:    @messages.map { |m| render_message_json(m) },
      has_more:    @messages.first&.id != @room.messages.minimum(:id),
      oldest_id:   @messages.first&.id
    }
  end

  def upload_file
    @room = Room.find(params[:room_id])
    file  = params[:file]

    unless file.present?
      return render json: { error: 'No file provided' }, status: :bad_request
    end

    @message = @room.messages.create!(
      user:         current_user,
      content:      file.original_filename,
      message_type: file.content_type.start_with?('image/') ? 'image' : 'file'
    )
    @message.attachment.attach(file)

    render json: {
      message_id: @message.id,
      url:        rails_blob_path(@message.attachment, only_path: true)
    }
  end

  private

  def message_params
    params.require(:message).permit(:content, :reply_to_id)
  end

  def render_message_json(message)
    {
      id:         message.id,
      content:    message.content,
      user:       { id: message.user.id, username: message.user.username },
      created_at: message.created_at.iso8601,
      reactions:  message.reactions_summary
    }
  end
end

# app/controllers/direct_messages_controller.rb
class DirectMessagesController < ApplicationController
  before_action :authenticate_user!

  def index
    # List recent conversations
    @conversations = User.joins(
      "JOIN direct_messages ON (direct_messages.sender_id = users.id OR direct_messages.receiver_id = users.id)"
    ).where(
      "direct_messages.sender_id = ? OR direct_messages.receiver_id = ?",
      current_user.id, current_user.id
    ).where.not(id: current_user.id)
     .distinct
     .order("direct_messages.created_at DESC")
     .limit(20)
  end

  def show
    @other_user   = User.find(params[:id])
    @messages     = current_user.dm_conversation_with(@other_user).last(50)

    # Mark as read
    DirectMessage.where(sender: @other_user, receiver: current_user, read: false)
                 .update_all(read: true, read_at: Time.current)

    respond_to do |format|
      format.html
      format.json { render json: { messages: @messages } }
    end
  end

  def create
    @other_user = User.find(params[:user_id])

    dm = DirectMessage.create!(
      sender:   current_user,
      receiver: @other_user,
      content:  params[:content]
    )

    NotificationService.new(@other_user).notify_new_dm(dm)

    render json: { dm: dm, status: :created }
  end
end
```

---

## ขั้นตอนที่ 6: Turbo Streams Views

```erb
<!-- app/views/rooms/show.html.erb -->
<div class="flex h-screen bg-gray-100" data-controller="chat">

  <!-- Sidebar: Rooms List -->
  <div class="w-64 bg-gray-800 text-white flex-shrink-0">
    <%= render 'rooms/sidebar' %>
  </div>

  <!-- Main Chat Area -->
  <div class="flex-1 flex flex-col">

    <!-- Room Header -->
    <div class="bg-white border-b px-6 py-4 flex items-center justify-between shadow-sm">
      <div>
        <h2 class="text-lg font-semibold"># <%= @room.name %></h2>
        <p class="text-sm text-gray-500"><%= @room.description %></p>
      </div>
      <div class="flex items-center gap-3">
        <span class="text-sm text-gray-500"><%= @room.members.count %> members</span>
        <%= button_to "Leave", leave_room_path(@room), method: :delete,
            class: "text-sm text-red-500 hover:text-red-700",
            data: { confirm: "Leave this room?" } %>
      </div>
    </div>

    <!-- Messages Container -->
    <div id="messages"
         class="flex-1 overflow-y-auto p-6 space-y-4"
         data-chat-target="messages"
         data-room-id="<%= @room.id %>">

      <%= turbo_stream_from "room_#{@room.id}" %>

      <% if @messages.any? %>
        <div class="text-center">
          <button data-action="click->chat#loadMore"
                  data-oldest-id="<%= @messages.first.id %>"
                  class="text-sm text-blue-500 hover:text-blue-700">
            Load older messages
          </button>
        </div>
      <% end %>

      <%= render @messages %>
    </div>

    <!-- Typing Indicator -->
    <div id="typing_indicator" class="px-6 py-1 h-6 text-sm text-gray-400 italic">
    </div>

    <!-- Message Input -->
    <div class="bg-white border-t p-4">
      <%= render 'messages/form', room: @room %>
    </div>
  </div>

  <!-- Members Sidebar -->
  <div class="w-56 bg-white border-l hidden lg:block">
    <%= render 'rooms/members', members: @members %>
  </div>
</div>

<!-- app/views/messages/_message.html.erb -->
<div id="message_<%= message.id %>"
     class="flex gap-3 group <%= message.user == current_user ? 'flex-row-reverse' : '' %>">

  <!-- Avatar -->
  <div class="flex-shrink-0">
    <img src="<%= message.user.avatar_url %>"
         alt="<%= message.user.username %>"
         class="w-8 h-8 rounded-full">
  </div>

  <!-- Message Content -->
  <div class="max-w-xs lg:max-w-md">
    <div class="flex items-center gap-2 mb-1 <%= message.user == current_user ? 'flex-row-reverse' : '' %>">
      <span class="text-sm font-medium text-gray-800"><%= message.user.username %></span>
      <span class="text-xs text-gray-400">
        <%= message.created_at.strftime('%H:%M') %>
        <% if message.edited %>
          <span class="italic">(edited)</span>
        <% end %>
      </span>
    </div>

    <% if message.reply_to %>
      <div class="mb-1 pl-2 border-l-2 border-gray-300 text-xs text-gray-500">
        Replying to <%= message.reply_to.user.username %>:
        <%= truncate(message.reply_to.content, length: 50) %>
      </div>
    <% end %>

    <div class="<%= message.user == current_user ?
                    'bg-blue-500 text-white rounded-tl-2xl rounded-bl-2xl rounded-tr-sm' :
                    'bg-white text-gray-800 rounded-tr-2xl rounded-br-2xl rounded-tl-sm' %>
                p-3 shadow-sm">

      <% if message.deleted? %>
        <span class="text-gray-400 italic text-sm">This message was deleted</span>
      <% elsif message.image? %>
        <%= image_tag rails_blob_path(message.attachment), class: "max-w-full rounded" if message.attachment.attached? %>
        <p class="text-sm mt-1"><%= message.content %></p>
      <% elsif message.file? %>
        <div class="flex items-center gap-2">
          <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
            <path d="M4 4a2 2 0 012-2h4.586A2 2 0 0112 2.586L15.414 6A2 2 0 0116 7.414V16a2 2 0 01-2 2H6a2 2 0 01-2-2V4z"/>
          </svg>
          <%= link_to message.content, rails_blob_path(message.attachment), class: "hover:underline" if message.attachment.attached? %>
        </div>
      <% elsif message.system? %>
        <span class="text-gray-500 italic text-sm text-center block"><%= message.content %></span>
      <% else %>
        <p class="text-sm whitespace-pre-wrap"><%= message.content %></p>
      <% end %>
    </div>

    <!-- Reactions -->
    <% if message.reactions.any? %>
      <div id="reactions_<%= message.id %>" class="flex gap-1 mt-1 flex-wrap">
        <% message.reactions_summary.each do |reaction| %>
          <button class="flex items-center gap-1 text-xs bg-gray-100 rounded-full px-2 py-0.5
                         hover:bg-gray-200 transition <%= reaction[:reacted_by_me] ? 'bg-blue-100' : '' %>"
                  data-action="click->chat#react"
                  data-message-id="<%= message.id %>"
                  data-emoji="<%= reaction[:emoji] %>">
            <%= reaction[:emoji] %> <%= reaction[:count] %>
          </button>
        <% end %>
      </div>
    <% end %>

    <!-- Message Actions (hover) -->
    <% unless message.deleted? || message.system? %>
      <div class="hidden group-hover:flex items-center gap-1 mt-1">
        <button class="text-xs text-gray-400 hover:text-gray-600"
                data-action="click->chat#showReactionPicker"
                data-message-id="<%= message.id %>">😀</button>
        <button class="text-xs text-gray-400 hover:text-gray-600"
                data-action="click->chat#replyTo"
                data-message-id="<%= message.id %>"
                data-username="<%= message.user.username %>">Reply</button>
        <% if message.user == current_user %>
          <button class="text-xs text-red-400 hover:text-red-600"
                  data-action="click->chat#deleteMessage"
                  data-message-id="<%= message.id %>">Delete</button>
        <% end %>
      </div>
    <% end %>
  </div>
</div>

<!-- app/views/messages/_form.html.erb -->
<%= turbo_frame_tag "message_form" do %>
  <div id="reply_indicator" class="hidden mb-2 flex items-center gap-2 text-sm text-gray-500 bg-gray-50 p-2 rounded">
    <span>Replying to <strong id="reply_to_username"></strong></span>
    <button data-action="click->chat#cancelReply" class="text-gray-400 hover:text-gray-600">✕</button>
    <input type="hidden" id="reply_to_id" name="reply_to_id">
  </div>

  <%= form_with url: room_messages_path(room), data: {
    action: "submit->chat#sendMessage",
    "chat-target": "form"
  }, class: "flex gap-3" do |f| %>
    <%= f.hidden_field :reply_to_id, id: "reply_to_id_field" %>

    <!-- File upload button -->
    <label class="cursor-pointer flex items-center">
      <svg class="w-6 h-6 text-gray-400 hover:text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.172 7l-6.586 6.586a2 2 0 102.828 2.828l6.414-6.586a4 4 0 00-5.656-5.656l-6.415 6.585a6 6 0 108.486 8.486L20.5 13"/>
      </svg>
      <%= f.file_field :attachment, class: "hidden",
          data: { action: "change->chat#uploadFile" } %>
    </label>

    <%= f.text_area :content,
        rows: 1,
        placeholder: "Message ##{room.name}",
        class: "flex-1 border border-gray-200 rounded-lg px-4 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 resize-none",
        data: {
          "chat-target": "input",
          action: "keydown->chat#handleKeydown input->chat#handleTyping"
        } %>

    <%= f.submit "Send",
        class: "bg-blue-500 text-white px-4 py-2 rounded-lg hover:bg-blue-600 transition text-sm font-medium" %>
  <% end %>
<% end %>
```

---

## ขั้นตอนที่ 7: Stimulus Controller

```javascript
// app/javascript/controllers/chat_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["messages", "input", "form"]

  connect() {
    this.setupWebSocket()
    this.scrollToBottom()
    this.typingTimer = null
    this.isTyping = false
  }

  disconnect() {
    this.cleanupWebSocket()
  }

  setupWebSocket() {
    const roomId = this.messagesTarget.dataset.roomId

    this.channel = this.application.consumer.subscriptions.create(
      { channel: "RoomChannel", room_id: roomId },
      {
        received: (data) => this.handleWebSocketData(data)
      }
    )
  }

  handleWebSocketData(data) {
    switch(data.type) {
      case 'typing_start':
        this.showTypingIndicator(data.username)
        break
      case 'typing_stop':
        this.hideTypingIndicator(data.user_id)
        break
      case 'reaction_update':
        this.updateReactions(data)
        break
      case 'status_change':
        this.updateUserStatus(data.user)
        break
    }
  }

  sendMessage(event) {
    event.preventDefault()
    const form    = event.target
    const content = this.inputTarget.value.trim()

    if (!content) return

    this.channel.speak({
      content:     content,
      reply_to_id: document.getElementById('reply_to_id_field')?.value
    })

    this.inputTarget.value = ''
    this.cancelReply()
    this.stopTyping()
  }

  handleKeydown(event) {
    if (event.key === 'Enter' && !event.shiftKey) {
      event.preventDefault()
      this.formTarget.dispatchEvent(new Event('submit', { bubbles: true }))
    }
  }

  handleTyping() {
    if (!this.isTyping) {
      this.isTyping = true
      this.channel.start_typing({})
    }

    clearTimeout(this.typingTimer)
    this.typingTimer = setTimeout(() => this.stopTyping(), 1500)
  }

  stopTyping() {
    if (this.isTyping) {
      this.isTyping = false
      this.channel.stop_typing({})
    }
  }

  showTypingIndicator(username) {
    const indicator = document.getElementById('typing_indicator')
    if (indicator) {
      this.typingUsers = this.typingUsers || new Set()
      this.typingUsers.add(username)
      this.renderTypingIndicator()

      setTimeout(() => {
        this.typingUsers?.delete(username)
        this.renderTypingIndicator()
      }, 3000)
    }
  }

  renderTypingIndicator() {
    const indicator = document.getElementById('typing_indicator')
    const users = Array.from(this.typingUsers || [])

    if (users.length === 0) {
      indicator.textContent = ''
    } else if (users.length === 1) {
      indicator.textContent = `${users[0]} is typing...`
    } else {
      indicator.textContent = `${users.slice(0, -1).join(', ')} and ${users.slice(-1)} are typing...`
    }
  }

  replyTo(event) {
    const messageId = event.currentTarget.dataset.messageId
    const username  = event.currentTarget.dataset.username

    document.getElementById('reply_indicator').classList.remove('hidden')
    document.getElementById('reply_to_username').textContent = username
    document.getElementById('reply_to_id_field').value = messageId

    this.inputTarget.focus()
  }

  cancelReply() {
    document.getElementById('reply_indicator')?.classList.add('hidden')
    const field = document.getElementById('reply_to_id_field')
    if (field) field.value = ''
  }

  react(event) {
    const messageId = event.currentTarget.dataset.messageId
    const emoji     = event.currentTarget.dataset.emoji

    this.channel.react({ message_id: messageId, emoji: emoji })
  }

  updateReactions(data) {
    const container = document.getElementById(`reactions_${data.message_id}`)
    if (container) {
      container.innerHTML = data.summary.map(r => `
        <button class="flex items-center gap-1 text-xs bg-gray-100 rounded-full px-2 py-0.5
                       hover:bg-gray-200 transition ${r.reacted_by_me ? 'bg-blue-100' : ''}"
                data-action="click->chat#react"
                data-message-id="${data.message_id}"
                data-emoji="${r.emoji}">
          ${r.emoji} ${r.count}
        </button>
      `).join('')
    }
  }

  deleteMessage(event) {
    const messageId = event.currentTarget.dataset.messageId
    if (!confirm('Delete this message?')) return

    fetch(`/messages/${messageId}`, {
      method:  'DELETE',
      headers: { 'X-CSRF-Token': document.querySelector('[name="csrf-token"]')?.content }
    })
  }

  loadMore(event) {
    const button   = event.currentTarget
    const oldestId = button.dataset.oldestId
    const roomId   = this.messagesTarget.dataset.roomId

    fetch(`/rooms/${roomId}/messages/load_more?before_id=${oldestId}`)
      .then(r => r.json())
      .then(data => {
        const messages    = this.messagesTarget
        const firstMsg    = messages.children[1]
        const scrollHeight = messages.scrollHeight

        data.messages.forEach(msg => {
          const el = this.createMessageElement(msg)
          messages.insertBefore(el, firstMsg)
        })

        // รักษา scroll position
        messages.scrollTop = messages.scrollHeight - scrollHeight

        if (!data.has_more) {
          button.parentElement.remove()
        } else {
          button.dataset.oldestId = data.oldest_id
        }
      })
  }

  scrollToBottom() {
    const container = this.messagesTarget
    if (container) {
      container.scrollTop = container.scrollHeight
    }
  }

  uploadFile(event) {
    const file   = event.target.files[0]
    const roomId = this.messagesTarget.dataset.roomId

    if (!file) return

    const formData = new FormData()
    formData.append('file', file)

    fetch(`/rooms/${roomId}/messages/upload_file`, {
      method:  'POST',
      headers: { 'X-CSRF-Token': document.querySelector('[name="csrf-token"]')?.content },
      body:    formData
    })
    .then(r => r.json())
    .then(data => {
      console.log('File uploaded:', data)
    })
  }
}
```

---

## ขั้นตอนที่ 8: Notification Service

```ruby
# app/services/notification_service.rb
class NotificationService
  def initialize(user)
    @user = user
  end

  def notify_new_dm(dm)
    notification = @user.notifications.create!(
      ntype: 'new_dm',
      title: "New message from #{dm.sender.username}",
      body:  truncate(dm.content, length: 100),
      data:  { dm_id: dm.id, sender_id: dm.sender_id }
    )

    broadcast_notification(notification)
  end

  def notify_room_mention(message, room)
    notification = @user.notifications.create!(
      ntype: 'room_mention',
      title: "#{message.user.username} mentioned you in ##{room.name}",
      body:  truncate(message.content, length: 100),
      data:  { message_id: message.id, room_id: room.id }
    )

    broadcast_notification(notification)
  end

  def notify_room_invite(room, invited_by)
    notification = @user.notifications.create!(
      ntype: 'room_invite',
      title: "#{invited_by.username} invited you to ##{room.name}",
      data:  { room_id: room.id }
    )

    broadcast_notification(notification)
  end

  private

  def broadcast_notification(notification)
    ActionCable.server.broadcast(
      "notifications_#{@user.id}",
      {
        type: 'new_notification',
        notification: {
          id:    notification.id,
          ntype: notification.ntype,
          title: notification.title,
          body:  notification.body,
          data:  notification.data,
          created_at: notification.created_at.iso8601
        }
      }
    )
  end

  def truncate(text, length:)
    return text if text.length <= length
    "#{text[0, length]}..."
  end
end

# app/jobs/send_push_notification_job.rb
class SendPushNotificationJob < ApplicationJob
  queue_as :notifications

  def perform(user_id, title, body, data = {})
    user = User.find_by(id: user_id)
    return unless user&.push_token.present?

    # Firebase Cloud Messaging
    FcmService.send_notification(
      token:  user.push_token,
      title:  title,
      body:   body,
      data:   data
    )
  rescue => e
    Rails.logger.error "Push notification failed for user #{user_id}: #{e.message}"
  end
end
```

---

## ขั้นตอนที่ 9: Online Status Management

```ruby
# app/middleware/online_status_middleware.rb
class OnlineStatusMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    status, headers, body = @app.call(env)

    request = ActionDispatch::Request.new(env)
    if request.session[:user_id].present?
      # Background update - ไม่ block request
      user_id = request.session[:user_id]
      UpdateOnlineStatusJob.perform_later(user_id)
    end

    [status, headers, body]
  end
end

# app/jobs/update_online_status_job.rb
class UpdateOnlineStatusJob < ApplicationJob
  queue_as :default

  def perform(user_id)
    user = User.find_by(id: user_id)
    return unless user

    if user.last_seen_at.nil? || user.last_seen_at < 5.minutes.ago
      user.update_columns(
        online_status: 'online',
        last_seen_at:  Time.current
      )

      ActionCable.server.broadcast("presence_channel", {
        type:   'status_change',
        user:   { id: user.id, username: user.username, online_status: 'online' }
      })
    end
  end
end

# app/jobs/mark_users_offline_job.rb
class MarkUsersOfflineJob < ApplicationJob
  queue_as :default

  def perform
    # Users ที่ไม่ active นาน 10 นาที
    offline_users = User.where(online_status: 'online')
                        .where('last_seen_at < ?', 10.minutes.ago)

    offline_users.each do |user|
      user.update_columns(online_status: 'offline')

      ActionCable.server.broadcast("presence_channel", {
        type: 'status_change',
        user: { id: user.id, username: user.username, online_status: 'offline',
                last_seen_at: user.last_seen_at }
      })
    end
  end
end

# Run ทุก 5 นาทีด้วย Sidekiq Scheduler
# config/initializers/sidekiq_cron.rb
Sidekiq::Cron::Job.create(
  name:  'Mark Users Offline',
  cron:  '*/5 * * * *',
  class: 'MarkUsersOfflineJob'
)
```

---

## ขั้นตอนที่ 10: Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users

  root to: 'rooms#index'

  resources :rooms do
    member do
      post :join
      delete :leave
    end

    resources :messages, only: [:create, :destroy] do
      collection do
        get  :load_more
        post :upload_file
      end
    end
  end

  resources :users, only: [:show] do
    member do
      get :direct_messages
    end
  end

  resources :direct_messages, only: [:index, :show, :create], path: 'dm'

  resources :notifications, only: [:index] do
    member do
      patch :mark_read
    end
    collection do
      patch :mark_all_read
    end
  end

  get '/profile',     to: 'profile#show'
  patch '/profile',   to: 'profile#update'
  get  '/search',     to: 'search#index'

  # Action Cable mount
  mount ActionCable.server => '/cable'
end
```

---

## ขั้นตอนที่ 11: Search Functionality

```ruby
# app/controllers/search_controller.rb
class SearchController < ApplicationController
  before_action :authenticate_user!

  def index
    query = params[:q]&.strip
    return render json: { results: [] } if query.blank? || query.length < 2

    results = {
      messages: search_messages(query),
      rooms:    search_rooms(query),
      users:    search_users(query)
    }

    respond_to do |format|
      format.html
      format.json { render json: results }
    end
  end

  private

  def search_messages(query)
    Message.visible
           .joins(:room)
           .where("messages.content ILIKE ?", "%#{query}%")
           .where(rooms: { id: current_user.room_ids })
           .includes(:user, :room)
           .order(created_at: :desc)
           .limit(10)
           .map do |msg|
      {
        id:      msg.id,
        content: highlight(msg.content, query),
        user:    { username: msg.user.username },
        room:    { id: msg.room.id, name: msg.room.name },
        at:      msg.created_at.iso8601
      }
    end
  end

  def search_rooms(query)
    Room.public_rooms
        .where("name ILIKE ? OR description ILIKE ?", "%#{query}%", "%#{query}%")
        .limit(5)
        .map { |r| { id: r.id, name: r.name, member_count: r.members.count } }
  end

  def search_users(query)
    User.where("username ILIKE ? OR display_name ILIKE ?", "%#{query}%", "%#{query}%")
        .where.not(id: current_user.id)
        .limit(5)
        .map { |u| { id: u.id, username: u.username, avatar_url: u.avatar_url } }
  end

  def highlight(text, query)
    # Simple highlight
    text.gsub(/#{Regexp.escape(query)}/i, "<mark>\\0</mark>")
  end
end
```

---

## ขั้นตอนที่ 12: Testing

```ruby
# spec/channels/room_channel_spec.rb
RSpec.describe RoomChannel, type: :channel do
  let(:user) { create(:user) }
  let(:room) { create(:room, room_type: 'public') }

  before do
    stub_connection current_user: user
    create(:room_membership, room: room, user: user)
  end

  describe '#subscribed' do
    it 'subscribes successfully' do
      subscribe(room_id: room.id)
      expect(subscription).to be_confirmed
    end

    it 'streams from room channel' do
      subscribe(room_id: room.id)
      expect(streams).to include("room_#{room.id}")
    end

    it 'rejects unknown rooms' do
      subscribe(room_id: 99999)
      expect(subscription).to be_rejected
    end
  end

  describe '#speak' do
    before { subscribe(room_id: room.id) }

    it 'creates a message' do
      expect {
        perform :speak, content: 'Hello, World!'
      }.to change(Message, :count).by(1)
    end

    it 'broadcasts to room' do
      expect {
        perform :speak, content: 'Hello!'
      }.to have_broadcasted_to("room_#{room.id}")
    end

    it 'does not create empty messages' do
      expect {
        perform :speak, content: ''
      }.not_to change(Message, :count)
    end
  end

  describe '#react' do
    let!(:message) { create(:message, room: room, user: user) }

    before { subscribe(room_id: room.id) }

    it 'creates a reaction' do
      expect {
        perform :react, message_id: message.id, emoji: '👍'
      }.to change(Reaction, :count).by(1)
    end

    it 'removes existing reaction (toggle)' do
      create(:reaction, message: message, user: user, emoji: '👍')

      expect {
        perform :react, message_id: message.id, emoji: '👍'
      }.to change(Reaction, :count).by(-1)
    end
  end
end
```

---

## สรุป

ChatApp Real-time Application ครอบคลุม:
- **Action Cable**: WebSocket สำหรับ real-time messaging
- **Turbo Streams**: Auto-update UI โดยไม่ต้อง JavaScript มาก
- **Public/Private Rooms**: Room management สมบูรณ์
- **Online Status**: Real-time presence tracking
- **Message History**: Pagination และ load more
- **File Attachments**: Active Storage integration
- **Notifications**: In-app notifications
- **Stimulus**: Minimal JavaScript controller

แอปนี้ใช้ Rails stack เต็มรูปแบบ โดยไม่ต้องการ separate frontend framework
