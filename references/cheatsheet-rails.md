# Rails Cheatsheet — ฉบับสมบูรณ์

> อ้างอิงด่วนสำหรับ Ruby on Rails คำสั่งและ syntax ทั้งหมด

---

## 1. Rails Commands

```bash
# สร้าง app ใหม่
rails new myapp
rails new myapp --database=postgresql
rails new myapp --api
rails new myapp --skip-test --skip-git

# Server
rails server         # หรือ rails s
rails s -p 4000      # custom port
rails s -b 0.0.0.0   # bind to all IPs

# Console
rails console        # หรือ rails c
rails c --sandbox    # ไม่บันทึกการเปลี่ยนแปลง

# Database
rails db:create
rails db:migrate
rails db:rollback
rails db:rollback STEP=3
rails db:seed
rails db:reset       # drop + create + migrate + seed
rails db:drop
rails db:schema:load
rails db:migrate:status

# Generators
rails generate model User name:string email:string
rails generate controller Users index show new create edit update destroy
rails generate scaffold Post title:string body:text user:references
rails generate migration AddAgeToUsers age:integer
rails generate mailer UserMailer
rails generate job ProcessPayment
rails generate channel Chat

# Destroy (undo generate)
rails destroy model User
rails destroy controller Users

# Routes
rails routes
rails routes | grep user
rails routes --expanded

# Assets
rails assets:precompile
rails assets:clean

# Tests
rails test
rails test test/models/user_test.rb
rails test:system

# Stats
rails stats
rails notes
```

## 2. Routing

```ruby
# config/routes.rb

Rails.application.routes.draw do
  # Root
  root "home#index"

  # RESTful (generates 7 routes)
  resources :posts
  resources :users, only: [:index, :show, :new, :create]
  resources :comments, except: [:destroy]

  # Nested resources
  resources :posts do
    resources :comments, shallow: true
    member do
      post :publish
      get  :preview
    end
    collection do
      get :search
    end
  end

  # Namespace (URL prefix + module)
  namespace :admin do
    resources :users
    resources :posts
  end

  # Scope (URL prefix only)
  scope :admin do
    resources :dashboard
  end

  # Scope with module
  scope module: :api do
    resources :products
  end

  # Custom routes
  get  "/about",         to: "pages#about",   as: :about
  post "/contact",       to: "pages#contact", as: :contact
  get  "/login",         to: "sessions#new",  as: :login
  post "/login",         to: "sessions#create"
  delete "/logout",      to: "sessions#destroy", as: :logout

  # Constraints
  get "/users/:id", to: "users#show", constraints: { id: /[0-9]+/ }

  # Redirect
  get "/old-path", to: redirect("/new-path")

  # Concerns
  concern :commentable do
    resources :comments
  end
  resources :posts, concerns: :commentable
end
```

### Route Helpers

```ruby
# URL helpers
root_path              # => "/"
root_url               # => "http://localhost:3000/"
posts_path             # => "/posts"
post_path(post)        # => "/posts/1"
post_path(id: 1)       # => "/posts/1"
new_post_path          # => "/posts/new"
edit_post_path(post)   # => "/posts/1/edit"

# With query params
posts_path(page: 2, search: "ruby")  # => "/posts?page=2&search=ruby"
```

## 3. Controllers

```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :authorize_post, only: [:edit, :update, :destroy]

  def index
    @posts = Post.published.order(created_at: :desc).page(params[:page])
  end

  def show
    @comments = @post.comments.includes(:user)
  end

  def new
    @post = current_user.posts.build
  end

  def create
    @post = current_user.posts.build(post_params)
    if @post.save
      redirect_to @post, notice: "Post created!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @post.update(post_params)
      redirect_to @post, notice: "Post updated!"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @post.destroy
    redirect_to posts_url, notice: "Post deleted!"
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :body, :published, tag_ids: [])
  end

  def authorize_post
    redirect_to root_path, alert: "Not authorized" unless @post.user == current_user
  end
end
```

### Rendering and Redirecting

```ruby
# Render
render :new
render "posts/new"
render template: "posts/new"
render json: @post
render json: @post, status: :created
render json: { error: "Not found" }, status: :not_found
render plain: "Hello"
render html: "<h1>Hello</h1>".html_safe
render nothing: true  # deprecated, use head :ok
render status: :ok
render inline: "<%= @post.title %>"

# Redirect
redirect_to @post
redirect_to post_path(@post)
redirect_to posts_path
redirect_to root_path
redirect_to :back  # deprecated, use redirect_back
redirect_back fallback_location: root_path
redirect_to posts_path, notice: "Success!"
redirect_to posts_path, alert: "Error!"
redirect_to posts_path, status: :moved_permanently

# Flash
flash[:notice] = "Success!"
flash[:alert]  = "Error!"
flash.now[:notice] = "Temporary (for render)"

# Head
head :ok
head :created
head :no_content
head :not_found
head :unprocessable_entity
```

## 4. Models (Active Record)

```ruby
class Post < ApplicationRecord
  # Associations
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :taggings
  has_many :tags, through: :taggings
  has_one  :featured_image, class_name: "Image"
  has_many :likes, as: :likeable  # polymorphic

  # Validations
  validates :title,   presence: true, length: { minimum: 3, maximum: 200 }
  validates :body,    presence: true
  validates :status,  inclusion: { in: %w[draft published archived] }
  validates :slug,    uniqueness: true, format: { with: /\A[a-z0-9-]+\z/ }
  validate  :title_not_blacklisted

  # Callbacks
  before_validation :generate_slug
  before_save       :sanitize_content
  after_create      :notify_subscribers
  after_destroy     :cleanup_assets

  # Scopes
  scope :published,  -> { where(published: true) }
  scope :recent,     -> { order(created_at: :desc) }
  scope :by_author,  ->(user) { where(user: user) }
  scope :tagged_with, ->(tag) { joins(:tags).where(tags: { name: tag }) }
  default_scope     { order(:title) }

  # Enum
  enum status: { draft: 0, published: 1, archived: 2 }
  # Generates: published?, draft?, published!, Post.published, etc.

  # Delegations
  delegate :name, :email, to: :user, prefix: true

  # Virtual attributes
  attr_accessor :remember_token

  # Callbacks
  def generate_slug
    self.slug = title.parameterize if title_changed?
  end

  def title_not_blacklisted
    if %w[spam banned].any? { |word| title.downcase.include?(word) }
      errors.add(:title, "contains forbidden content")
    end
  end
end
```

### Querying

```ruby
# Find
Post.find(1)
Post.find(1, 2, 3)       # raises ActiveRecord::RecordNotFound
Post.find_by(slug: "hello")  # returns nil if not found
Post.find_by!(slug: "hello") # raises if not found

# Where
Post.where(published: true)
Post.where("created_at > ?", 1.week.ago)
Post.where(created_at: 1.week.ago..)
Post.where.not(published: false)
Post.where(user: current_user)
Post.where(status: [:published, :featured])

# Chaining
Post.published.recent.by_author(user).page(2)

# Order
Post.order(:title)
Post.order(created_at: :desc)
Post.order("created_at DESC, title ASC")
Post.reorder(:title)  # replaces existing order

# Limit/Offset
Post.limit(10).offset(20)

# Select
Post.select(:id, :title, :created_at)

# Includes (eager loading)
Post.includes(:user, :comments, :tags)
Post.includes(comments: :user)
Post.eager_load(:user)
Post.preload(:comments)

# Joins
Post.joins(:comments)
Post.joins(:user).where(users: { active: true })
Post.left_joins(:comments)

# Count, sum, avg
Post.count
Post.where(published: true).count
Post.sum(:view_count)
Post.average(:rating)
Post.minimum(:price)
Post.maximum(:price)

# Group
Post.group(:status).count
Post.group(:user_id).sum(:view_count)

# Exists?
Post.where(slug: "hello").exists?
Post.exists?(id: 1)

# First, last
Post.first
Post.first(3)
Post.last
Post.last(5)

# Pluck (returns array of values)
Post.pluck(:id)
Post.pluck(:id, :title)  # array of arrays
Post.where(published: true).pluck(:slug)

# Pick (like pluck but first record only)
Post.where(id: 1).pick(:title)

# IDs
Post.ids   # array of all ids

# Batch
Post.find_each(batch_size: 100) { |post| post.process }
Post.find_in_batches(batch_size: 100) { |batch| batch.each(&:process) }
Post.in_batches(of: 100).each_record { |post| post.process }

# Calculate
Post.calculate(:count, :id)
Post.calculate(:sum, :view_count)

# None (returns empty relation)
Post.none
```

### CRUD

```ruby
# Create
Post.create(title: "Hello", body: "World")
Post.create!(title: "Hello")  # raises on failure

post = Post.new(title: "Hello")
post.save
post.save!  # raises on failure

# Read — see Querying above

# Update
post.update(title: "New Title")
post.update!(title: "New Title")  # raises on failure
post.title = "New Title"; post.save

Post.where(published: false).update_all(status: :archived)

# Delete
post.destroy         # triggers callbacks
post.delete          # skips callbacks, direct SQL
Post.destroy_all     # triggers callbacks
Post.delete_all      # skips callbacks
Post.where(id: [1,2,3]).destroy_all
```

## 5. Migrations

```ruby
class CreatePosts < ActiveRecord::Migration[7.1]
  def change
    create_table :posts do |t|
      t.string     :title,      null: false
      t.text       :body
      t.boolean    :published,  default: false, null: false
      t.integer    :view_count, default: 0
      t.decimal    :price,      precision: 8, scale: 2
      t.datetime   :published_at
      t.references :user,       null: false, foreign_key: true
      t.timestamps              # created_at, updated_at
    end
    add_index :posts, :slug, unique: true
    add_index :posts, [:user_id, :published]
  end
end

# Column types
t.string      :name               # VARCHAR(255)
t.text        :body                # TEXT
t.integer     :count               # INTEGER
t.bigint      :large_number        # BIGINT
t.float       :rating              # FLOAT
t.decimal     :price               # DECIMAL
t.boolean     :active              # BOOLEAN
t.date        :born_on             # DATE
t.time        :started_at          # TIME
t.datetime    :expires_at          # DATETIME
t.timestamp   :updated_at          # TIMESTAMP
t.binary      :data                # BLOB
t.json        :metadata            # JSON (PG)
t.jsonb       :settings            # JSONB (PG)
t.array       :tags                # ARRAY (PG)
t.hstore      :properties          # HSTORE (PG)
t.uuid        :external_id         # UUID (PG)
t.references  :user                # user_id + index

# Modifying tables
add_column    :posts, :slug, :string
remove_column :posts, :old_field
rename_column :posts, :old, :new
change_column :posts, :count, :bigint
add_index     :posts, :slug, unique: true
remove_index  :posts, :slug
add_foreign_key :posts, :users
add_reference :posts, :category, null: false, foreign_key: true
```

## 6. Associations

```ruby
# belongs_to
class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user, optional: true    # can be nil
  belongs_to :author, class_name: "User", foreign_key: :user_id
  belongs_to :category, touch: true   # updates category.updated_at
  belongs_to :parent, class_name: "Comment", optional: true
end

# has_many
class Post < ApplicationRecord
  has_many :comments
  has_many :comments, dependent: :destroy
  has_many :comments, dependent: :nullify
  has_many :published_comments, -> { where(approved: true) }, class_name: "Comment"
  has_many :readers, through: :page_views, source: :user
  has_many :tags, through: :taggings
end

# has_one
class User < ApplicationRecord
  has_one :profile
  has_one :profile, dependent: :destroy
end

# has_many :through
class Post < ApplicationRecord
  has_many :taggings
  has_many :tags, through: :taggings
end

class Tag < ApplicationRecord
  has_many :taggings
  has_many :posts, through: :taggings
end

class Tagging < ApplicationRecord
  belongs_to :post
  belongs_to :tag
end

# Polymorphic
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
end
class Post < ApplicationRecord
  has_many :comments, as: :commentable
end
class Video < ApplicationRecord
  has_many :comments, as: :commentable
end

# Association methods
post.comments              # returns all comments
post.comments.create(...)  # create associated
post.comments.build(...)   # new (unsaved)
post.comments << comment   # add
post.comments.destroy(id)  # destroy one
post.comment_ids           # array of IDs
post.comment_ids = [1,2,3] # replace associations
user.posts.exists?
```

## 7. Validations

```ruby
class User < ApplicationRecord
  validates :name,      presence: true
  validates :email,     presence: true,
                        uniqueness: { case_sensitive: false },
                        format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :age,       numericality: { greater_than_or_equal_to: 18,
                                        less_than: 130 }
  validates :username,  length: { minimum: 3, maximum: 20 },
                        format: { with: /\A[a-z0-9_]+\z/,
                                  message: "only lowercase letters, numbers, underscores" }
  validates :role,      inclusion: { in: %w[user moderator admin] }
  validates :terms,     acceptance: true
  validates :password,  confirmation: true
  validates :bio,       length: { maximum: 500 }, allow_blank: true

  # Conditional
  validates :company,   presence: true, if: :business_account?
  validates :phone,     presence: true, if: -> { role == "admin" }
  validates :terms,     acceptance: true, on: :create  # only on create

  # Custom method
  validate :email_domain_allowed
  validate :age_appropriate, on: :create

  # Custom validator class
  validates_with AgeValidator

  private

  def email_domain_allowed
    domain = email.split("@").last
    if %w[spam.com banned.net].include?(domain)
      errors.add(:email, "domain is not allowed")
    end
  end
end

# Check validity
user.valid?
user.invalid?
user.errors
user.errors.full_messages
user.errors[:email]
user.errors.add(:base, "Something went wrong")
```

## 8. Views (ERB)

```erb
<%# Comment %>
<% Ruby code, no output %>
<%= Ruby expression, outputs result %>
<%- Suppress newline (some configs) %>

<!-- Layouts: app/views/layouts/application.html.erb -->
<%= yield %>                  <!-- main content -->
<%= yield :sidebar %>         <!-- named section -->

<!-- In view, provide to named yield -->
<% content_for :sidebar do %>
  <p>Sidebar content</p>
<% end %>

<!-- Partial -->
<%= render "shared/header" %>
<%= render partial: "posts/post", locals: { post: @post } %>
<%= render @posts %>              <!-- renders _post.html.erb for each -->
<%= render @post %>               <!-- renders _post.html.erb -->

<!-- Link helpers -->
<%= link_to "Home", root_path %>
<%= link_to "Delete", post, method: :delete, data: { confirm: "Sure?" } %>
<%= link_to "Edit", edit_post_path(@post), class: "btn btn-primary" %>
<%= link_to post do %>
  <strong><%= post.title %></strong>
<% end %>

<!-- Image -->
<%= image_tag "logo.png", alt: "Logo", class: "logo" %>
<%= image_tag @user.avatar, size: "50x50" %>

<!-- Form (Rails 7) -->
<%= form_with model: @post do |f| %>
  <%= f.label :title %>
  <%= f.text_field :title, class: "form-control" %>
  <%= f.label :body %>
  <%= f.text_area :body, rows: 10 %>
  <%= f.check_box :published %>
  <%= f.label :published %>
  <%= f.select :status, Post.statuses.keys %>
  <%= f.collection_select :category_id, Category.all, :id, :name %>
  <%= f.submit "Save", class: "btn btn-primary" %>
<% end %>

<!-- Display errors -->
<% if @post.errors.any? %>
  <div class="alert alert-danger">
    <ul>
      <% @post.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<!-- Flash messages -->
<% if notice %>
  <div class="alert alert-success"><%= notice %></div>
<% end %>
<% if alert %>
  <div class="alert alert-danger"><%= alert %></div>
<% end %>

<!-- View helpers -->
<%= number_to_currency(1234.56) %>          <!-- $1,234.56 -->
<%= number_with_delimiter(1234567) %>        <!-- 1,234,567 -->
<%= number_to_percentage(75.5, precision: 1) %>  <!-- 75.5% -->
<%= time_ago_in_words(post.created_at) %>   <!-- "3 days ago" -->
<%= distance_of_time_in_words(from, to) %>
<%= truncate(post.body, length: 100) %>
<%= pluralize(5, "post") %>                 <!-- "5 posts" -->
<%= simple_format(text) %>                  <!-- preserves newlines -->
<%= word_wrap(text, line_width: 80) %>
<%= highlight("Hello World", "World") %>
<%= strip_tags("<b>Hello</b>") %>           <!-- "Hello" -->
<%= html_escape("<script>") %>              <!-- "&lt;script&gt;" -->
<%= raw("<b>bold</b>") %>                   <!-- don't escape -->
<%= "text".html_safe %>                     <!-- mark as safe -->
```

## 9. Forms

```erb
<!-- form_with (Rails 5.1+) -->
<%= form_with model: @post, local: true do |f| %>
  <!-- Text inputs -->
  <%= f.text_field :title %>
  <%= f.email_field :email %>
  <%= f.password_field :password %>
  <%= f.number_field :age, min: 0, max: 120 %>
  <%= f.url_field :website %>
  <%= f.telephone_field :phone %>
  <%= f.date_field :birthday %>
  <%= f.time_field :starts_at %>
  <%= f.datetime_local_field :published_at %>
  <%= f.text_area :body, rows: 5, cols: 40 %>
  <%= f.hidden_field :token %>
  
  <!-- Checkboxes and Radios -->
  <%= f.check_box :published %>
  <%= f.label :published %>
  
  <%= f.radio_button :role, "admin" %>
  <%= f.label :role_admin, "Admin" %>
  <%= f.radio_button :role, "user" %>
  <%= f.label :role_user, "User" %>
  
  <!-- Select -->
  <%= f.select :status, ["draft", "published", "archived"] %>
  <%= f.select :status, options_for_select([["Draft", "draft"], ["Published", "published"]], @post.status) %>
  <%= f.collection_select :category_id, Category.all, :id, :name, { include_blank: "Select..." } %>
  <%= f.grouped_collection_select :city_id, Country.all, :cities, :name, :id, :name %>
  
  <!-- Multiple select -->
  <%= f.select :tag_ids, Tag.all.map { |t| [t.name, t.id] }, {}, { multiple: true } %>
  
  <!-- File upload -->
  <%= f.file_field :avatar %>
  <%= f.file_field :attachments, multiple: true %>
  
  <!-- Submit -->
  <%= f.submit "Save Post" %>
  <%= f.button "Cancel", type: "button" %>
<% end %>

<!-- Nested forms (accepts_nested_attributes_for) -->
<%= form_with model: @post do |f| %>
  <%= f.fields_for :comments do |c| %>
    <%= c.text_field :body %>
    <%= c.check_box :_destroy %> Remove
  <% end %>
  <%= link_to_add_fields "Add Comment", f, :comments %>
<% end %>
```

## 10. Active Storage

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached  :avatar
  has_many_attached :documents
end

# Upload
user.avatar.attach(params[:avatar])
user.avatar.attach(io: File.open("/path/to/file"), filename: "avatar.jpg")

# Check
user.avatar.attached?
user.documents.any?

# URL
url_for(user.avatar)
rails_blob_path(user.avatar, disposition: "attachment")

# Variants (image processing)
user.avatar.variant(resize_to_limit: [100, 100])
user.avatar.variant(resize_to_fill: [200, 200], format: :webp)

# In view
<%= image_tag user.avatar %>
<%= image_tag user.avatar.variant(resize_to_limit: [100, 100]) %>
<%= link_to "Download", rails_blob_path(user.document), data: { turbo: false } %>

# Delete
user.avatar.purge
user.avatar.purge_later  # background job
```

## 11. Active Record Callbacks

```ruby
class Post < ApplicationRecord
  # Order of callbacks on SAVE:
  # before_validation
  # on validation
  # after_validation
  # before_save
  # before_create (if new) / before_update (if existing)
  # (actual save to DB)
  # after_create / after_update
  # after_save
  # after_commit / after_rollback

  before_validation :normalize_title
  after_validation  :check_profanity
  before_save       :update_slug
  before_create     :set_defaults
  after_create      :send_notification
  before_update     :log_changes
  after_update      :clear_cache
  after_save        :update_search_index
  before_destroy    :check_if_deletable
  after_destroy     :log_deletion
  after_commit      :refresh_cache
  after_rollback    :revert_side_effects

  # Conditional callbacks
  before_save :encrypt_data, if: :sensitive?
  before_save :notify_admin, unless: -> { admin_notified }
  after_create :welcome_email, on: :create

  private

  def normalize_title
    self.title = title&.strip&.downcase&.capitalize
  end
end
```

## 12. Devise Quick Reference

```ruby
# Gemfile
gem 'devise'

# Install
rails generate devise:install
rails generate devise User
rails db:migrate
rails generate devise:views

# Model
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :lockable, :trackable, :omniauthable
end

# Controller helpers
before_action :authenticate_user!
before_action :authenticate_user!, except: [:index, :show]

current_user          # the signed-in user
user_signed_in?       # true/false
sign_in(user)         # sign in programmatically
sign_out(user)        # sign out programmatically
sign_out :user        # sign out

# Route helpers
new_user_session_path         # login
destroy_user_session_path     # logout
new_user_registration_path    # signup
edit_user_registration_path   # edit profile
user_confirmation_path        # confirm email
new_user_password_path        # forgot password

# In views
<% if user_signed_in? %>
  <%= link_to "Sign out", destroy_user_session_path, method: :delete %>
<% else %>
  <%= link_to "Sign in", new_user_session_path %>
<% end %>
```

## 13. Testing (RSpec Rails)

```ruby
# Gemfile
group :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'shoulda-matchers'
  gem 'faker'
  gem 'capybara'
end

# Model spec
RSpec.describe Post, type: :model do
  subject { build(:post) }

  it { is_expected.to be_valid }
  it { is_expected.to validate_presence_of(:title) }
  it { is_expected.to validate_length_of(:title).is_at_least(3) }
  it { is_expected.to belong_to(:user) }
  it { is_expected.to have_many(:comments).dependent(:destroy) }

  describe "#publish!" do
    let(:post) { create(:post, published: false) }
    it "publishes the post" do
      expect { post.publish! }.to change { post.published }.to(true)
    end
  end
end

# Request spec
RSpec.describe "Posts", type: :request do
  let(:user) { create(:user) }
  let(:post) { create(:post, user: user) }

  describe "GET /posts" do
    it "returns http success" do
      get posts_path
      expect(response).to have_http_status(:success)
    end
  end

  describe "POST /posts" do
    context "with valid params" do
      it "creates post and redirects" do
        sign_in user
        expect {
          post posts_path, params: { post: attributes_for(:post) }
        }.to change(Post, :count).by(1)
        expect(response).to redirect_to(Post.last)
      end
    end
  end
end

# Factory
FactoryBot.define do
  factory :user do
    name  { Faker::Name.name }
    email { Faker::Internet.unique.email }
    password { "password123" }
  end

  factory :post do
    title { Faker::Lorem.sentence }
    body  { Faker::Lorem.paragraphs(number: 3).join("\n") }
    published { true }
    association :user
  end
end

# System spec (Capybara)
RSpec.describe "Creating posts", type: :system do
  let(:user) { create(:user) }

  it "allows user to create a post" do
    sign_in user
    visit new_post_path
    fill_in "Title", with: "My Post"
    fill_in "Body",  with: "Post content"
    click_button "Create Post"
    expect(page).to have_content("Post created!")
    expect(page).to have_content("My Post")
  end
end
```

## 14. Turbo (Rails 7 Hotwire)

```erb
<!-- Turbo Frame -->
<%= turbo_frame_tag "post_#{post.id}" do %>
  <h2><%= post.title %></h2>
  <%= link_to "Edit", edit_post_path(post) %>
<% end %>

<!-- Turbo Stream -->
<%= turbo_stream_from "posts" %>

<!-- In controller -->
respond_to do |format|
  format.turbo_stream
  format.html { redirect_to posts_path }
end

<!-- posts/create.turbo_stream.erb -->
<%= turbo_stream.prepend "posts", @post %>
<%= turbo_stream.append "posts", partial: "posts/post", locals: { post: @post } %>
<%= turbo_stream.replace "post_#{@post.id}", @post %>
<%= turbo_stream.remove "post_#{@post.id}" %>
<%= turbo_stream.update "flash", partial: "shared/flash" %>

<!-- Broadcast from model -->
class Post < ApplicationRecord
  after_create_commit  -> { broadcast_prepend_to "posts" }
  after_update_commit  -> { broadcast_replace_to "posts" }
  after_destroy_commit -> { broadcast_remove_to "posts" }
end
```

---

*Rails 7 Cheatsheet — reference ใช้งานในชีวิตประจำวัน*
