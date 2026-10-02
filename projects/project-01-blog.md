# Project 1: Blog Application - สร้าง Blog ครบฟีเจอร์ด้วย Ruby on Rails

## เป้าหมายของโปรเจค

สร้าง Blog application ที่มีฟีเจอร์ครบถ้วน ได้แก่:
- Authentication (Devise)
- Authorization (Pundit)
- Rich Text Editor (ActionText/Trix)
- Image Uploads (ActiveStorage + S3)
- Full-text Search
- REST API
- Admin Panel
- Tags และ Categories
- Comments System
- Like/Bookmark

---

## ขั้นตอนที่ 1: สร้าง Project

```bash
rails new blog_app \
  --database=postgresql \
  --css=tailwind \
  --javascript=importmap

cd blog_app
```

### Gemfile

```ruby
# Gemfile
source "https://rubygems.org"

ruby "3.2.2"

gem "rails", "~> 7.1.0"
gem "pg", "~> 1.1"
gem "puma", ">= 5.0"
gem "importmap-rails"
gem "turbo-rails"
gem "stimulus-rails"
gem "tailwindcss-rails"
gem "jbuilder"

# Authentication
gem "devise", "~> 4.9"
gem "omniauth-google-oauth2"
gem "omniauth-rails_csrf_protection"

# Authorization
gem "pundit", "~> 2.3"

# File uploads
gem "image_processing", "~> 1.2"

# Search
gem "pg_search"

# Pagination
gem "pagy", "~> 6.0"

# Rich text
# (ActionText มาพร้อม Rails)

# Background jobs
gem "sidekiq", "~> 7.0"
gem "redis", "~> 5.0"

# Admin
gem "administrate", "~> 0.20"

# Security
gem "rack-attack"
gem "secure_headers"

# Monitoring
gem "sentry-ruby"
gem "sentry-rails"

# Storage
gem "aws-sdk-s3", require: false

group :development, :test do
  gem "rspec-rails", "~> 6.0"
  gem "factory_bot_rails"
  gem "faker"
  gem "byebug"
end

group :development do
  gem "web-console"
  gem "rack-mini-profiler"
  gem "letter_opener"
  gem "rubocop-rails", require: false
  gem "brakeman", require: false
  gem "annotate"
end

group :test do
  gem "capybara"
  gem "selenium-webdriver"
  gem "webmock"
  gem "vcr"
  gem "shoulda-matchers"
  gem "simplecov"
end
```

```bash
bundle install
rails db:create
```

---

## ขั้นตอนที่ 2: Database Setup

```ruby
# db/schema.rb (เราจะสร้าง migrations ทีละขั้น)
```

### Migrations

```bash
# Users
rails generate devise:install
rails generate devise User

# Profiles
rails generate migration CreateProfiles \
  user:references \
  bio:text \
  website:string \
  twitter:string \
  github:string \
  location:string

# Posts
rails generate model Post \
  title:string \
  slug:string \
  status:integer \
  user:references \
  published_at:datetime \
  views_count:integer:default:0 \
  likes_count:integer:default:0

# Categories
rails generate model Category \
  name:string \
  slug:string \
  description:text \
  parent:references

# Tags
rails generate model Tag name:string slug:string color:string

# PostTags (join table)
rails generate migration CreatePostTags post:references tag:references

# Comments
rails generate model Comment \
  body:text \
  user:references \
  post:references \
  parent:references

# Likes
rails generate model Like \
  user:references \
  likeable:references{polymorphic}

# Bookmarks
rails generate model Bookmark \
  user:references \
  post:references

# Follows
rails generate migration CreateFollows \
  follower_id:integer \
  followed_id:integer
```

### Add Indexes Migration

```ruby
# db/migrate/xxxx_add_indexes.rb
class AddIndexes < ActiveRecord::Migration[7.1]
  def change
    # Posts
    add_index :posts, :slug, unique: true
    add_index :posts, :status
    add_index :posts, :published_at
    add_index :posts, [:user_id, :status]
    add_index :posts, [:user_id, :created_at]
    
    # Users
    add_index :users, :username, unique: true
    
    # Tags
    add_index :tags, :slug, unique: true
    
    # PostTags
    add_index :post_tags, [:post_id, :tag_id], unique: true
    
    # Likes
    add_index :likes, [:user_id, :likeable_type, :likeable_id], unique: true
    
    # Bookmarks
    add_index :bookmarks, [:user_id, :post_id], unique: true
    
    # Follows
    add_index :follows, [:follower_id, :followed_id], unique: true
    
    # Comments
    add_index :comments, [:post_id, :created_at]
    add_index :comments, :parent_id
    
    # Full-text search
    execute "CREATE INDEX posts_search_idx ON posts USING GIN (to_tsvector('english', title || ' ' || COALESCE(body_plain_text, '')))"
  end
end
```

```bash
rails db:migrate
```

---

## ขั้นตอนที่ 3: Models

### User Model

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable, :recoverable,
         :rememberable, :validatable, :confirmable, :lockable,
         :omniauthable, omniauth_providers: [:google_oauth2]
  
  has_one :profile, dependent: :destroy
  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy
  has_many :likes, dependent: :destroy
  has_many :bookmarks, dependent: :destroy
  
  # Following
  has_many :active_follows, class_name: "Follow", foreign_key: :follower_id, dependent: :destroy
  has_many :passive_follows, class_name: "Follow", foreign_key: :followed_id, dependent: :destroy
  has_many :following, through: :active_follows, source: :followed
  has_many :followers, through: :passive_follows, source: :follower
  
  # Active Storage
  has_one_attached :avatar
  
  # Callbacks
  after_create :create_profile
  
  # Validations
  validates :username, presence: true, uniqueness: { case_sensitive: false },
            format: { with: /\A[a-zA-Z0-9_]+\z/, message: "ใช้ได้แค่ a-z, 0-9, _" },
            length: { minimum: 3, maximum: 30 }
  validates :name, presence: true, length: { minimum: 2, maximum: 100 }
  
  # Roles
  enum role: { user: 0, moderator: 1, admin: 2 }
  
  # Scopes
  scope :active, -> { where(disabled_at: nil) }
  scope :authors, -> { joins(:posts).distinct }
  
  def to_param
    username
  end
  
  def avatar_url
    if avatar.attached?
      Rails.application.routes.url_helpers.rails_blob_url(avatar, only_path: true)
    else
      "https://ui-avatars.com/api/?name=#{URI.encode_www_form_component(name)}&background=random"
    end
  end
  
  def follow(other_user)
    following << other_user unless following?(other_user)
  end
  
  def unfollow(other_user)
    following.delete(other_user)
  end
  
  def following?(other_user)
    following.include?(other_user)
  end
  
  def liked?(likeable)
    likes.exists?(likeable: likeable)
  end
  
  def bookmarked?(post)
    bookmarks.exists?(post: post)
  end
  
  def self.from_omniauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0, 20]
      user.name = auth.info.name
      user.username = generate_username(auth.info.email)
      user.skip_confirmation!
    end
  end
  
  private
  
  def self.generate_username(email)
    base = email.split('@').first.gsub(/[^a-zA-Z0-9_]/, '_')
    username = base
    counter = 1
    while exists?(username: username)
      username = "#{base}_#{counter}"
      counter += 1
    end
    username
  end
end
```

### Post Model

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include PgSearch::Model
  
  belongs_to :user
  belongs_to :category, optional: true
  
  has_many :post_tags, dependent: :destroy
  has_many :tags, through: :post_tags
  has_many :comments, dependent: :destroy
  has_many :likes, as: :likeable, dependent: :destroy
  has_many :bookmarks, dependent: :destroy
  
  has_rich_text :body
  has_one_attached :cover_image
  has_many_attached :images
  
  # Enums
  enum status: { draft: 0, published: 1, archived: 2 }
  
  # Callbacks
  before_validation :generate_slug, if: :title_changed?
  before_save :set_published_at
  
  # Validations
  validates :title, presence: true, length: { minimum: 5, maximum: 200 }
  validates :body, presence: true
  validates :slug, presence: true, uniqueness: true, format: { with: /\A[a-z0-9-]+\z/ }
  validates :status, presence: true
  
  # Scopes
  scope :published, -> { where(status: :published).where("published_at <= ?", Time.current) }
  scope :recent, -> { order(created_at: :desc) }
  scope :popular, -> { order(likes_count: :desc, views_count: :desc) }
  scope :featured, -> { where(featured: true) }
  scope :by_category, ->(cat) { where(category: cat) }
  
  # Full-text search
  pg_search_scope :search_full_text,
    against: {
      title: "A",
      body: "B"
    },
    using: {
      tsearch: {
        prefix: true,
        dictionary: "english"
      }
    }
  
  def to_param
    slug
  end
  
  def increment_view!
    increment!(:views_count)
  end
  
  def reading_time
    words = body.to_plain_text.split.length
    minutes = (words / 200.0).ceil
    "#{minutes} นาที"
  end
  
  def excerpt(length: 200)
    body.to_plain_text.truncate(length)
  end
  
  private
  
  def generate_slug
    return if title.blank?
    
    base_slug = title.parameterize
    slug = base_slug
    counter = 1
    
    while Post.where(slug: slug).where.not(id: id).exists?
      slug = "#{base_slug}-#{counter}"
      counter += 1
    end
    
    self.slug = slug
  end
  
  def set_published_at
    if published? && published_at.nil?
      self.published_at = Time.current
    end
  end
end
```

### Comment Model

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :post, counter_cache: true
  belongs_to :parent, class_name: "Comment", optional: true
  
  has_many :replies, class_name: "Comment", foreign_key: :parent_id, dependent: :destroy
  has_many :likes, as: :likeable, dependent: :destroy
  
  # Callbacks
  after_create_commit :broadcast_to_post
  after_destroy_commit :broadcast_removal
  
  validates :body, presence: true, length: { minimum: 2, maximum: 2000 }
  
  scope :root, -> { where(parent_id: nil) }
  scope :recent, -> { order(created_at: :desc) }
  
  private
  
  def broadcast_to_post
    broadcast_prepend_to(
      post,
      target: "comments",
      partial: "comments/comment",
      locals: { comment: self }
    )
  end
  
  def broadcast_removal
    broadcast_remove_to(post)
  end
end
```

### Tag Model

```ruby
# app/models/tag.rb
class Tag < ApplicationRecord
  has_many :post_tags, dependent: :destroy
  has_many :posts, through: :post_tags
  
  before_validation :generate_slug
  
  validates :name, presence: true, uniqueness: { case_sensitive: false },
            length: { minimum: 1, maximum: 50 }
  validates :slug, presence: true, uniqueness: true
  
  scope :popular, -> { 
    joins(:posts)
      .where(posts: { status: :published })
      .group("tags.id")
      .order("COUNT(posts.id) DESC")
  }
  
  def to_param
    slug
  end
  
  private
  
  def generate_slug
    return if name.blank?
    self.slug = name.downcase.parameterize
  end
end
```

---

## ขั้นตอนที่ 4: Authentication (Devise)

### Devise Configuration

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  config.mailer_sender = ENV['MAILER_FROM'] || "noreply@myblog.com"
  config.case_insensitive_keys = [:email]
  config.strip_whitespace_keys = [:email]
  config.skip_session_storage = [:http_auth]
  config.stretches = Rails.env.test? ? 1 : 12
  config.reconfirmable = true
  config.expire_all_remember_me_on_sign_out = true
  config.password_length = 6..128
  config.email_regexp = /\A[^@\s]+@[^@\s]+\z/
  config.reset_password_within = 6.hours
  config.sign_out_via = :delete
  config.responder.error_status = :unprocessable_entity
  config.responder.redirect_status = :see_other
end
```

### Devise Views

```bash
rails generate devise:views users
```

```erb
<%# app/views/devise/registrations/new.html.erb %>
<div class="min-h-screen flex items-center justify-center bg-gray-50 py-12 px-4">
  <div class="max-w-md w-full bg-white rounded-lg shadow p-8">
    <h2 class="text-3xl font-bold text-center mb-8">สมัครสมาชิก</h2>
    
    <%= form_for(resource, as: resource_name, url: registration_path(resource_name)) do |f| %>
      <%= render "devise/shared/error_messages", resource: resource %>
      
      <div class="space-y-4">
        <div>
          <%= f.label :name, "ชื่อ", class: "block text-sm font-medium text-gray-700" %>
          <%= f.text_field :name, autofocus: true, autocomplete: "name",
                          class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500" %>
        </div>
        
        <div>
          <%= f.label :username, "ชื่อผู้ใช้", class: "block text-sm font-medium text-gray-700" %>
          <%= f.text_field :username,
                          class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500" %>
          <p class="mt-1 text-xs text-gray-500">ใช้ได้แค่ a-z, 0-9, _</p>
        </div>
        
        <div>
          <%= f.label :email, "อีเมล", class: "block text-sm font-medium text-gray-700" %>
          <%= f.email_field :email, autocomplete: "email",
                          class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500" %>
        </div>
        
        <div>
          <%= f.label :password, "รหัสผ่าน", class: "block text-sm font-medium text-gray-700" %>
          <%= f.password_field :password, autocomplete: "new-password",
                              class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500" %>
          <% if @minimum_password_length %>
            <p class="mt-1 text-xs text-gray-500">อย่างน้อย <%= @minimum_password_length %> ตัวอักษร</p>
          <% end %>
        </div>
        
        <div>
          <%= f.label :password_confirmation, "ยืนยันรหัสผ่าน", class: "block text-sm font-medium text-gray-700" %>
          <%= f.password_field :password_confirmation, autocomplete: "new-password",
                              class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500" %>
        </div>
        
        <%= f.submit "สมัครสมาชิก", 
            class: "w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500" %>
      </div>
    <% end %>
    
    <p class="mt-4 text-center text-sm text-gray-600">
      มีบัญชีอยู่แล้ว? <%= link_to "เข้าสู่ระบบ", new_user_session_path, class: "text-indigo-600 hover:text-indigo-500" %>
    </p>
    
    <div class="mt-6">
      <div class="relative">
        <div class="absolute inset-0 flex items-center">
          <div class="w-full border-t border-gray-300"></div>
        </div>
        <div class="relative flex justify-center text-sm">
          <span class="px-2 bg-white text-gray-500">หรือ</span>
        </div>
      </div>
      
      <div class="mt-6">
        <%= button_to "เข้าสู่ระบบด้วย Google", user_google_oauth2_omniauth_authorize_path,
            method: :post,
            class: "w-full flex justify-center items-center py-2 px-4 border border-gray-300 rounded-md shadow-sm text-sm font-medium text-gray-700 bg-white hover:bg-gray-50" %>
      </div>
    </div>
  </div>
</div>
```

---

## ขั้นตอนที่ 5: Authorization (Pundit)

```bash
rails generate pundit:install
```

```ruby
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  class Scope < Scope
    def resolve
      if user&.admin? || user&.moderator?
        scope.all
      elsif user
        scope.published.or(scope.where(user: user))
      else
        scope.published
      end
    end
  end
  
  def index? = true
  def show? = record.published? || owner? || admin? || moderator?
  def create? = user.present?
  def update? = owner? || admin? || moderator?
  def destroy? = owner? || admin?
  def publish? = owner? || admin?
  def archive? = owner? || admin?
  def feature? = admin?
  
  private
  
  def owner? = record.user == user
  def admin? = user&.admin?
  def moderator? = user&.moderator?
end
```

```ruby
# app/policies/comment_policy.rb
class CommentPolicy < ApplicationPolicy
  def show? = true
  def create? = user.present?
  def update? = author_within_time? || admin?
  def destroy? = owner_or_post_owner? || admin? || moderator?
  
  private
  
  def author_within_time?
    record.user == user && record.created_at > 24.hours.ago
  end
  
  def owner_or_post_owner?
    record.user == user || record.post.user == user
  end
  
  def admin? = user&.admin?
  def moderator? = user&.moderator?
end
```

---

## ขั้นตอนที่ 6: Controllers

### Posts Controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  include Pagy::Backend
  
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy, :publish, :archive]
  after_action :verify_authorized, except: [:index]
  after_action :verify_policy_scoped, only: :index
  
  def index
    @posts = policy_scope(Post)
              .published
              .includes(:user, :tags, :category)
              .recent
    
    # Filtering
    @posts = @posts.where(category: params[:category]) if params[:category]
    @posts = @posts.joins(:tags).where(tags: { slug: params[:tag] }) if params[:tag]
    @posts = @posts.where(user: User.find_by(username: params[:author])) if params[:author]
    
    # Search
    if params[:q].present?
      @posts = @posts.search_full_text(params[:q])
    end
    
    @pagy, @posts = pagy(@posts, items: 12)
  end
  
  def show
    authorize @post
    @post.increment_view! unless current_user == @post.user
    
    @comments = @post.comments.includes(:user, :replies).root.order(created_at: :desc)
    @new_comment = Comment.new
  end
  
  def new
    @post = Post.new
    authorize @post
  end
  
  def create
    @post = current_user.posts.new(post_params)
    authorize @post
    
    if @post.save
      if params[:post][:tag_names].present?
        tag_names = params[:post][:tag_names].split(",").map(&:strip)
        @post.tags = tag_names.map { |name| Tag.find_or_create_by!(name: name) }
      end
      
      redirect_to @post, notice: "สร้างบทความสำเร็จ!"
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  def edit
    authorize @post
  end
  
  def update
    authorize @post
    
    if @post.update(post_params)
      if params[:post][:tag_names]
        tag_names = params[:post][:tag_names].split(",").map(&:strip)
        @post.tags = tag_names.map { |name| Tag.find_or_create_by!(name: name) }
      end
      
      redirect_to @post, notice: "อัปเดตบทความสำเร็จ!"
    else
      render :edit, status: :unprocessable_entity
    end
  end
  
  def destroy
    authorize @post
    @post.destroy
    redirect_to posts_path, notice: "ลบบทความแล้ว"
  end
  
  def publish
    authorize @post, :publish?
    @post.update!(status: :published, published_at: Time.current)
    redirect_to @post, notice: "เผยแพร่บทความแล้ว!"
  end
  
  private
  
  def set_post
    @post = Post.find_by!(slug: params[:id])
  rescue ActiveRecord::RecordNotFound
    @post = Post.find(params[:id])
  end
  
  def post_params
    params.require(:post).permit(
      :title, :status, :category_id, :published_at,
      :cover_image, images: []
    ).merge(body: params.dig(:post, :body))
  end
end
```

---

## ขั้นตอนที่ 7: Views

### Layout

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= content_for?(:title) ? "#{yield :title} - MyBlog" : "MyBlog" %></title>
  <meta name="description" content="<%= content_for?(:description) ? yield(:description) : 'Blog สำหรับนักพัฒนา Ruby on Rails' %>">
  
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  
  <%= stylesheet_link_tag "application", media: "all" %>
  <%= javascript_importmap_tags %>
</head>
<body class="min-h-screen bg-gray-50" data-controller="flash">
  <%= render "layouts/navbar" %>
  
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
    <%= render "layouts/flash_messages" %>
    <%= yield %>
  </div>
  
  <%= render "layouts/footer" %>
</body>
</html>
```

```erb
<%# app/views/layouts/_navbar.html.erb %>
<nav class="bg-white shadow-sm border-b">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex justify-between h-16">
      <div class="flex items-center">
        <%= link_to "MyBlog", root_path, class: "text-2xl font-bold text-indigo-600" %>
        
        <div class="hidden sm:ml-8 sm:flex sm:space-x-8">
          <%= link_to "บทความ", posts_path, 
              class: "text-gray-700 hover:text-indigo-600 px-3 py-2 text-sm font-medium" %>
          <%= link_to "หมวดหมู่", categories_path,
              class: "text-gray-700 hover:text-indigo-600 px-3 py-2 text-sm font-medium" %>
          <%= link_to "แท็ก", tags_path,
              class: "text-gray-700 hover:text-indigo-600 px-3 py-2 text-sm font-medium" %>
        </div>
      </div>
      
      <div class="flex items-center space-x-4">
        <% if user_signed_in? %>
          <%= link_to "เขียนบทความ", new_post_path,
              class: "bg-indigo-600 text-white px-4 py-2 rounded-md text-sm font-medium hover:bg-indigo-700" %>
          
          <div class="relative" data-controller="dropdown">
            <button data-action="click->dropdown#toggle" class="flex items-center space-x-2">
              <img src="<%= current_user.avatar_url %>" class="w-8 h-8 rounded-full" alt="<%= current_user.name %>">
              <span class="text-sm text-gray-700"><%= current_user.name %></span>
            </button>
            
            <div data-dropdown-target="menu" class="hidden absolute right-0 mt-2 w-48 bg-white rounded-md shadow-lg py-1 z-10">
              <%= link_to "โปรไฟล์", user_path(current_user),
                  class: "block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" %>
              <%= link_to "บทความของฉัน", dashboard_posts_path,
                  class: "block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" %>
              <%= link_to "ตั้งค่า", edit_user_registration_path,
                  class: "block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" %>
              <%= button_to "ออกจากระบบ", destroy_user_session_path, method: :delete,
                  class: "block w-full text-left px-4 py-2 text-sm text-red-600 hover:bg-gray-100" %>
            </div>
          </div>
        <% else %>
          <%= link_to "เข้าสู่ระบบ", new_user_session_path,
              class: "text-gray-700 hover:text-indigo-600 text-sm font-medium" %>
          <%= link_to "สมัครสมาชิก", new_user_registration_path,
              class: "bg-indigo-600 text-white px-4 py-2 rounded-md text-sm font-medium hover:bg-indigo-700" %>
        <% end %>
      </div>
    </div>
  </div>
</nav>
```

### Posts Index View

```erb
<%# app/views/posts/index.html.erb %>
<% content_for :title, "บทความทั้งหมด" %>

<div class="grid grid-cols-1 lg:grid-cols-4 gap-8">
  <!-- Main Content -->
  <div class="lg:col-span-3">
    <!-- Search & Filters -->
    <div class="bg-white rounded-lg shadow-sm p-4 mb-6">
      <%= form_with url: posts_path, method: :get, class: "flex gap-4" do |f| %>
        <%= f.text_field :q, value: params[:q],
            placeholder: "ค้นหาบทความ...",
            class: "flex-1 rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500" %>
        <%= f.submit "ค้นหา",
            class: "bg-indigo-600 text-white px-4 py-2 rounded-md hover:bg-indigo-700" %>
      <% end %>
    </div>
    
    <!-- Posts Grid -->
    <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
      <% @posts.each do |post| %>
        <%= render "posts/card", post: post %>
      <% end %>
    </div>
    
    <!-- Pagination -->
    <div class="mt-8">
      <%== pagy_nav(@pagy) %>
    </div>
  </div>
  
  <!-- Sidebar -->
  <div class="lg:col-span-1 space-y-6">
    <%= render "posts/sidebar" %>
  </div>
</div>
```

```erb
<%# app/views/posts/_card.html.erb %>
<article class="bg-white rounded-lg shadow-sm overflow-hidden hover:shadow-md transition-shadow">
  <% if post.cover_image.attached? %>
    <%= link_to post_path(post) do %>
      <%= image_tag post.cover_image.variant(resize_to_fill: [400, 200]),
          class: "w-full h-48 object-cover" %>
    <% end %>
  <% end %>
  
  <div class="p-6">
    <!-- Tags -->
    <div class="flex flex-wrap gap-2 mb-3">
      <% post.tags.first(3).each do |tag| %>
        <%= link_to tag.name, tag_path(tag),
            class: "text-xs bg-indigo-100 text-indigo-700 px-2 py-1 rounded-full hover:bg-indigo-200" %>
      <% end %>
    </div>
    
    <!-- Title -->
    <h2 class="text-lg font-bold text-gray-900 mb-2 line-clamp-2">
      <%= link_to post.title, post_path(post), class: "hover:text-indigo-600" %>
    </h2>
    
    <!-- Excerpt -->
    <p class="text-gray-600 text-sm mb-4 line-clamp-3">
      <%= post.excerpt %>
    </p>
    
    <!-- Meta -->
    <div class="flex items-center justify-between">
      <div class="flex items-center space-x-2">
        <img src="<%= post.user.avatar_url %>" class="w-6 h-6 rounded-full" />
        <span class="text-sm text-gray-500">
          <%= link_to post.user.name, user_path(post.user), class: "hover:text-indigo-600" %>
        </span>
      </div>
      
      <div class="flex items-center space-x-3 text-xs text-gray-400">
        <span>❤️ <%= post.likes_count %></span>
        <span>💬 <%= post.comments_count %></span>
        <span>👁 <%= post.views_count %></span>
        <span><%= time_ago_in_words(post.published_at || post.created_at) %></span>
      </div>
    </div>
  </div>
</article>
```

### Post Show View

```erb
<%# app/views/posts/show.html.erb %>
<% content_for :title, @post.title %>
<% content_for :description, @post.excerpt %>

<div class="max-w-4xl mx-auto">
  <article>
    <!-- Cover Image -->
    <% if @post.cover_image.attached? %>
      <%= image_tag @post.cover_image.variant(resize_to_fill: [1200, 500]),
          class: "w-full h-64 md:h-96 object-cover rounded-xl mb-8" %>
    <% end %>
    
    <!-- Header -->
    <header class="mb-8">
      <!-- Tags -->
      <div class="flex flex-wrap gap-2 mb-4">
        <% @post.tags.each do |tag| %>
          <%= link_to tag.name, tag_path(tag),
              class: "text-sm bg-indigo-100 text-indigo-700 px-3 py-1 rounded-full hover:bg-indigo-200" %>
        <% end %>
      </div>
      
      <h1 class="text-3xl md:text-4xl font-bold text-gray-900 mb-4">
        <%= @post.title %>
      </h1>
      
      <!-- Meta -->
      <div class="flex items-center justify-between flex-wrap gap-4">
        <div class="flex items-center space-x-4">
          <img src="<%= @post.user.avatar_url %>" class="w-10 h-10 rounded-full" />
          <div>
            <p class="font-medium text-gray-900">
              <%= link_to @post.user.name, user_path(@post.user), class: "hover:text-indigo-600" %>
            </p>
            <p class="text-sm text-gray-500">
              <%= @post.published_at&.strftime("%d %B %Y") %>
              · เวลาอ่าน <%= @post.reading_time %>
            </p>
          </div>
        </div>
        
        <!-- Actions -->
        <div class="flex items-center space-x-4">
          <% if user_signed_in? %>
            <%= button_to like_post_path(@post), method: :post,
                class: "flex items-center space-x-1 text-gray-500 hover:text-red-500 #{current_user.liked?(@post) ? 'text-red-500' : ''}" do %>
              <span>❤️</span>
              <span id="likes-count-<%= @post.id %>"><%= @post.likes_count %></span>
            <% end %>
            
            <%= button_to bookmark_post_path(@post), method: :post,
                class: "text-gray-500 hover:text-yellow-500 #{current_user.bookmarked?(@post) ? 'text-yellow-500' : ''}" do %>
              🔖
            <% end %>
          <% end %>
          
          <% if policy(@post).update? %>
            <%= link_to "แก้ไข", edit_post_path(@post),
                class: "text-sm text-indigo-600 hover:text-indigo-700" %>
          <% end %>
          
          <% if policy(@post).destroy? %>
            <%= button_to "ลบ", post_path(@post), method: :delete,
                data: { turbo_confirm: "ยืนยันการลบบทความ?" },
                class: "text-sm text-red-600 hover:text-red-700" %>
          <% end %>
        </div>
      </div>
    </header>
    
    <!-- Body (ActionText) -->
    <div class="prose prose-lg max-w-none mb-12">
      <%= @post.body %>
    </div>
  </article>
  
  <!-- Comments Section -->
  <section class="mt-12 border-t pt-8">
    <h2 class="text-2xl font-bold text-gray-900 mb-6">
      ความคิดเห็น (<span id="comments-count"><%= @post.comments_count %></span>)
    </h2>
    
    <%= turbo_stream_from @post %>
    
    <!-- New Comment Form -->
    <% if user_signed_in? %>
      <%= render "comments/form", post: @post, comment: @new_comment %>
    <% else %>
      <p class="text-gray-600 mb-6">
        <%= link_to "เข้าสู่ระบบ", new_user_session_path, class: "text-indigo-600" %>
        เพื่อแสดงความคิดเห็น
      </p>
    <% end %>
    
    <!-- Comments List -->
    <div id="comments" class="space-y-6">
      <%= render @comments %>
    </div>
  </section>
</div>
```

---

## ขั้นตอนที่ 8: API Endpoints

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      include ActionController::HttpAuthentication::Token::ControllerMethods
      
      before_action :authenticate_api_user!
      
      rescue_from ActiveRecord::RecordNotFound, with: :not_found
      rescue_from ActiveRecord::RecordInvalid, with: :unprocessable
      rescue_from Pundit::NotAuthorizedError, with: :forbidden
      
      private
      
      def authenticate_api_user!
        authenticate_with_http_token do |token|
          @current_user = User.find_by(api_token: token)
        end
        
        unless @current_user
          render json: { error: "Unauthorized" }, status: :unauthorized
        end
      end
      
      def current_user
        @current_user
      end
      
      def not_found(exception)
        render json: { error: "Not found: #{exception.message}" }, status: :not_found
      end
      
      def unprocessable(exception)
        render json: { errors: exception.record.errors.full_messages }, 
               status: :unprocessable_entity
      end
      
      def forbidden
        render json: { error: "Forbidden" }, status: :forbidden
      end
    end
  end
end
```

```ruby
# app/controllers/api/v1/posts_controller.rb
module Api
  module V1
    class PostsController < BaseController
      skip_before_action :authenticate_api_user!, only: [:index, :show]
      
      def index
        @posts = Post.published
                     .includes(:user, :tags)
                     .recent
        
        @posts = @posts.search_full_text(params[:q]) if params[:q].present?
        @posts = @posts.page(params[:page]).per(params[:per_page] || 20)
        
        render json: {
          posts: @posts.map { |p| post_json(p) },
          meta: {
            current_page: @posts.current_page,
            total_pages: @posts.total_pages,
            total_count: @posts.total_count
          }
        }
      end
      
      def show
        @post = Post.find_by!(slug: params[:id])
        render json: post_json(@post, full: true)
      end
      
      def create
        @post = current_user.posts.new(post_params)
        
        if @post.save
          render json: post_json(@post), status: :created
        else
          render json: { errors: @post.errors.full_messages }, 
                 status: :unprocessable_entity
        end
      end
      
      private
      
      def post_params
        params.require(:post).permit(:title, :status, tag_ids: [])
              .merge(body: params.dig(:post, :body))
      end
      
      def post_json(post, full: false)
        data = {
          id: post.id,
          slug: post.slug,
          title: post.title,
          status: post.status,
          excerpt: post.excerpt,
          likes_count: post.likes_count,
          comments_count: post.comments_count,
          views_count: post.views_count,
          published_at: post.published_at&.iso8601,
          created_at: post.created_at.iso8601,
          author: {
            id: post.user.id,
            name: post.user.name,
            username: post.user.username,
            avatar_url: post.user.avatar_url
          },
          tags: post.tags.map { |t| { id: t.id, name: t.name, slug: t.slug } }
        }
        
        if full
          data[:body] = post.body.to_plain_text
        end
        
        data
      end
    end
  end
end
```

### API Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, controllers: {
    sessions: 'users/sessions',
    registrations: 'users/registrations',
    omniauth_callbacks: 'users/omniauth_callbacks'
  }
  
  # Main routes
  root "posts#index"
  
  resources :posts do
    member do
      post :publish
      post :archive
      post :like
      post :bookmark
    end
    
    resources :comments, only: [:create, :destroy]
  end
  
  resources :categories, only: [:index, :show]
  resources :tags, only: [:index, :show]
  resources :users, only: [:show], param: :username
  
  # Dashboard
  namespace :dashboard do
    root "posts#index"
    resources :posts
    resources :stats, only: [:index]
  end
  
  # Admin
  namespace :admin do
    root to: "dashboard#index"
    resources :users
    resources :posts
    resources :categories
    resources :tags
    resources :comments
  end
  
  # API
  namespace :api do
    namespace :v1 do
      resources :posts, only: [:index, :show, :create, :update, :destroy]
      resources :tags, only: [:index, :show]
      resources :users, only: [:show]
      post :auth, to: "auth#token"
    end
  end
  
  # Health
  get "/health", to: "health#show"
end
```

---

## ขั้นตอนที่ 9: Admin Panel

```bash
rails generate administrate:install
rails generate administrate:dashboard Post
rails generate administrate:dashboard User
rails generate administrate:dashboard Comment
rails generate administrate:dashboard Tag
rails generate administrate:dashboard Category
```

```ruby
# app/dashboards/post_dashboard.rb
require "administrate/base_dashboard"

class PostDashboard < Administrate::BaseDashboard
  ATTRIBUTE_TYPES = {
    id: Field::Number,
    title: Field::String,
    slug: Field::String,
    status: Field::Select.with_options(
      collection: Post.statuses.keys
    ),
    user: Field::BelongsTo,
    category: Field::BelongsTo,
    tags: Field::HasMany,
    likes_count: Field::Number,
    views_count: Field::Number,
    published_at: Field::DateTime,
    created_at: Field::DateTime,
    updated_at: Field::DateTime
  }.freeze
  
  SHOW_PAGE_ATTRIBUTES = %i[
    id title slug status user category tags
    likes_count views_count published_at
    created_at updated_at
  ].freeze
  
  INDEX_ATTRIBUTES = %i[
    id title status user likes_count views_count published_at
  ].freeze
  
  FORM_ATTRIBUTES = %i[
    title status user category tags published_at
  ].freeze
  
  SEARCH_ATTRIBUTES = [:title, :slug].freeze
end
```

```ruby
# app/controllers/admin/application_controller.rb
module Admin
  class ApplicationController < Administrate::ApplicationController
    before_action :authenticate_admin!
    
    private
    
    def authenticate_admin!
      unless current_user&.admin?
        redirect_to root_path, alert: "คุณไม่มีสิทธิ์เข้าถึงส่วนนี้"
      end
    end
  end
end
```

---

## ขั้นตอนที่ 10: Search

```ruby
# app/models/post.rb (เพิ่ม search)
pg_search_scope :search_full_text,
  against: { title: "A", body: "B" },
  using: {
    tsearch: { prefix: true, dictionary: "english" },
    trigram: { threshold: 0.3 }
  },
  ignoring: :accents
```

```ruby
# app/controllers/search_controller.rb
class SearchController < ApplicationController
  include Pagy::Backend
  
  def index
    @query = params[:q].to_s.strip
    
    if @query.length >= 2
      posts = Post.published.search_full_text(@query)
      @pagy, @posts = pagy(posts, items: 10)
      
      @tags = Tag.where("name ILIKE ?", "%#{@query}%").limit(5)
      @users = User.where("name ILIKE ? OR username ILIKE ?", "%#{@query}%", "%#{@query}%").limit(5)
    end
    
    respond_to do |format|
      format.html
      format.turbo_stream { render turbo_stream: turbo_stream.update("search-results", partial: "search/results") }
    end
  end
end
```

---

## ขั้นตอนที่ 11: Image Uploads

```ruby
# config/storage.yml
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: <%= ENV.fetch("AWS_REGION", "ap-southeast-1") %>
  bucket: <%= ENV.fetch("S3_BUCKET", "myblog-uploads") %>
```

```ruby
# config/environments/production.rb
config.active_storage.service = :amazon
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_one_attached :cover_image do |attachable|
    attachable.variant :thumb, resize_to_fill: [400, 200]
    attachable.variant :hero, resize_to_fill: [1200, 500]
    attachable.variant :og, resize_to_fill: [1200, 630]
  end
end
```

---

## ขั้นตอนที่ 12: Email Notifications

```ruby
# app/mailers/post_mailer.rb
class PostMailer < ApplicationMailer
  def new_comment(comment)
    @post = comment.post
    @comment = comment
    @author = @post.user
    
    mail(
      to: @author.email,
      subject: "มีความคิดเห็นใหม่ในบทความ '#{@post.title}'"
    )
  end
  
  def new_follower(user, follower)
    @user = user
    @follower = follower
    
    mail(
      to: @user.email,
      subject: "#{@follower.name} เริ่มติดตามคุณแล้ว"
    )
  end
  
  def post_published(post)
    followers = post.user.followers.select { |f| f.email_notifications? }
    return if followers.empty?
    
    @post = post
    @author = post.user
    
    followers.each do |follower|
      @follower = follower
      mail(
        to: follower.email,
        subject: "#{@author.name} เผยแพร่บทความใหม่: #{@post.title}"
      )
    end
  end
end
```

---

## ขั้นตอนที่ 13: Tests

```ruby
# spec/models/post_spec.rb
require 'rails_helper'

RSpec.describe Post, type: :model do
  subject(:post) { build(:post) }
  
  describe "validations" do
    it { should validate_presence_of(:title) }
    it { should validate_presence_of(:body) }
    it { should validate_length_of(:title).is_at_least(5).is_at_most(200) }
  end
  
  describe "associations" do
    it { should belong_to(:user) }
    it { should have_many(:comments).dependent(:destroy) }
    it { should have_many(:tags).through(:post_tags) }
    it { should have_many(:likes).dependent(:destroy) }
  end
  
  describe "enums" do
    it { should define_enum_for(:status).with_values(draft: 0, published: 1, archived: 2) }
  end
  
  describe "#generate_slug" do
    it "สร้าง slug จาก title" do
      post = build(:post, title: "สวัสดีโลก Hello World")
      post.valid?
      expect(post.slug).to eq("hello-world")
    end
    
    it "สร้าง unique slug" do
      existing_post = create(:post, title: "Test Post")
      new_post = build(:post, title: "Test Post")
      new_post.valid?
      expect(new_post.slug).to eq("test-post-1")
    end
  end
  
  describe "#reading_time" do
    it "คำนวณเวลาอ่านถูกต้อง" do
      post = build(:post)
      allow(post.body).to receive(:to_plain_text).and_return("word " * 400)
      expect(post.reading_time).to eq("2 นาที")
    end
  end
  
  describe "scopes" do
    let!(:published_post) { create(:post, :published) }
    let!(:draft_post) { create(:post, :draft) }
    
    it ".published ดึงเฉพาะบทความที่เผยแพร่" do
      expect(Post.published).to include(published_post)
      expect(Post.published).not_to include(draft_post)
    end
  end
end
```

```ruby
# spec/system/posts_spec.rb
require 'rails_helper'

RSpec.describe "Posts", type: :system do
  let(:user) { create(:user) }
  let!(:post) { create(:post, :published, user: user) }
  
  describe "ดูรายการบทความ" do
    it "แสดงบทความที่เผยแพร่แล้ว" do
      visit posts_path
      expect(page).to have_content(post.title)
    end
  end
  
  describe "สร้างบทความ" do
    before { sign_in user }
    
    it "สร้างบทความใหม่ได้" do
      visit new_post_path
      
      fill_in "หัวข้อ", with: "Test Post"
      fill_in_rich_text_area "เนื้อหา", with: "Test content"
      
      click_button "บันทึก"
      
      expect(page).to have_content("สร้างบทความสำเร็จ")
      expect(page).to have_content("Test Post")
    end
  end
end
```

---

## ขั้นตอนที่ 14: Deployment

```bash
# ตั้งค่า Heroku
heroku create my-blog-app
heroku addons:create heroku-postgresql:essential-0
heroku addons:create heroku-redis:mini
heroku addons:create sendgrid:starter

heroku config:set RAILS_ENV=production
heroku config:set RAILS_MASTER_KEY=$(cat config/master.key)
heroku config:set RAILS_LOG_TO_STDOUT=true
heroku config:set RAILS_SERVE_STATIC_FILES=true
heroku config:set AWS_REGION=ap-southeast-1
heroku config:set S3_BUCKET=my-blog-uploads

git push heroku main
heroku run rails db:migrate
heroku run rails db:seed
```

### Seeds

```ruby
# db/seeds.rb
puts "Creating admin user..."
admin = User.find_or_create_by(email: "admin@example.com") do |u|
  u.name = "Admin"
  u.username = "admin"
  u.password = "password123"
  u.password_confirmation = "password123"
  u.role = :admin
  u.confirmed_at = Time.current
end

puts "Creating categories..."
categories = [
  { name: "Programming", description: "บทความเกี่ยวกับการเขียนโปรแกรม" },
  { name: "Ruby on Rails", description: "บทความเกี่ยวกับ Ruby on Rails" },
  { name: "JavaScript", description: "บทความเกี่ยวกับ JavaScript" },
  { name: "DevOps", description: "บทความเกี่ยวกับ DevOps" }
]

categories.each do |cat|
  Category.find_or_create_by(name: cat[:name]) do |c|
    c.description = cat[:description]
  end
end

puts "Creating sample posts..."
10.times do |i|
  post = admin.posts.find_or_create_by(title: "บทความตัวอย่างที่ #{i + 1}") do |p|
    p.body = "เนื้อหาบทความตัวอย่างที่ #{i + 1} " * 50
    p.status = :published
    p.published_at = i.days.ago
    p.category = Category.order("RANDOM()").first
  end
end

puts "Seeds complete!"
```

---

## สรุปโปรเจค

Blog Application ที่เราสร้างมีฟีเจอร์:

1. **Authentication** - Devise + Google OAuth
2. **Authorization** - Pundit Policies
3. **Rich Text** - ActionText/Trix Editor
4. **File Upload** - ActiveStorage + S3
5. **Search** - PgSearch full-text
6. **API** - RESTful JSON API
7. **Admin** - Administrate
8. **Real-time** - Turbo Streams สำหรับ comments
9. **Notifications** - Email via Sidekiq
10. **Tests** - RSpec + Capybara

เทคโนโลยีที่ใช้:
- Ruby 3.2 + Rails 7.1
- PostgreSQL + Redis
- Tailwind CSS + Hotwire
- Sidekiq สำหรับ background jobs
- AWS S3 สำหรับ file storage
- Heroku สำหรับ deployment
