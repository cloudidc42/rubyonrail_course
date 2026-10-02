# Part 75: Rails Engines

## Steps 1621-1640

---

## Step 1621: Rails Engines คืออะไร?

Rails Engine คือ mini Rails application ที่สามารถ mount ลงใน application หลักได้ ใช้สำหรับ:
- แยก functionality ออกเป็น module ที่ reusable
- สร้าง plugin หรือ gem ที่มี Rails features ครบ
- แบ่ง large application ออกเป็นส่วนๆ (modular monolith)

### ประเภทของ Engine

```
Full Engine:
- มี route ของตัวเอง
- มี controller, model, view ของตัวเอง
- Namespace แยกจาก app หลัก
- mount ด้วย mount EngineName::Engine => '/path'

Mountable Engine:
- เหมือน Full Engine แต่ isolated namespace อย่างสมบูรณ์
- ทุกอย่างอยู่ใน module ของตัวเอง
- ไม่ปนกับ app หลัก

Plugin (Rails::Railtie):
- เพิ่ม functionality โดยไม่มี routes
- เช่น initializers, generators, rake tasks
```

### สร้าง Engine ใหม่

```bash
# สร้าง Mountable Engine
rails plugin new blog_engine --mountable

# สร้าง Full Engine
rails plugin new blog_engine --full

# โครงสร้างที่ได้
blog_engine/
├── app/
│   ├── controllers/
│   │   └── blog_engine/
│   │       └── application_controller.rb
│   ├── models/
│   │   └── blog_engine/
│   ├── views/
│   │   └── layouts/
│   │       └── blog_engine/
│   │           └── application.html.erb
│   └── helpers/
│       └── blog_engine/
├── config/
│   └── routes.rb
├── db/
│   └── migrate/
├── lib/
│   ├── blog_engine/
│   │   ├── engine.rb
│   │   └── version.rb
│   └── blog_engine.rb
├── test/
├── blog_engine.gemspec
└── Gemfile
```

---

## Step 1622: โครงสร้าง Engine

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
    
    # กำหนด engine config
    config.autoload_paths << root.join('lib')
    
    # Initializer - รันเมื่อ app start
    initializer 'blog_engine.setup' do |app|
      # Setup code
    end
    
    # Hook เข้า ActiveRecord
    initializer 'blog_engine.add_concerns' do
      ActiveSupport.on_load(:active_record) do
        # extend หรือ include modules
      end
    end
    
    # Generators
    config.generators do |g|
      g.test_framework :rspec
      g.fixture_replacement :factory_bot
    end
  end
end

# lib/blog_engine.rb
require 'blog_engine/engine'

module BlogEngine
  # Module-level code
end

# config/routes.rb ใน engine
BlogEngine::Engine.routes.draw do
  resources :posts do
    resources :comments
  end
  root to: 'posts#index'
end

# app/controllers/blog_engine/application_controller.rb
module BlogEngine
  class ApplicationController < ActionController::Base
    protect_from_forgery with: :exception
    
    # Engine จะ inherit methods จาก app หลักไม่ได้โดยตรง
    # ต้องใช้ hooks หรือ config
  end
end
```

---

## Step 1623: Controllers และ Models ใน Engine

```ruby
# app/controllers/blog_engine/posts_controller.rb
module BlogEngine
  class PostsController < ApplicationController
    before_action :set_post, only: [:show, :edit, :update, :destroy]
    
    def index
      @posts = Post.published.order(created_at: :desc).page(params[:page])
    end
    
    def show
      @comments = @post.comments.approved
    end
    
    def new
      @post = Post.new
    end
    
    def create
      @post = Post.new(post_params)
      @post.author = current_author
      
      if @post.save
        redirect_to post_path(@post), notice: 'บทความถูกสร้างแล้ว'
      else
        render :new, status: :unprocessable_entity
      end
    end
    
    def edit; end
    
    def update
      if @post.update(post_params)
        redirect_to post_path(@post), notice: 'บทความถูกอัพเดทแล้ว'
      else
        render :edit, status: :unprocessable_entity
      end
    end
    
    def destroy
      @post.destroy
      redirect_to posts_path, notice: 'ลบบทความแล้ว'
    end
    
    private
    
    def set_post
      @post = Post.find(params[:id])
    end
    
    def post_params
      params.require(:post).permit(:title, :content, :status, :published_at, tag_ids: [])
    end
    
    def current_author
      # Hook ให้ app หลัก override ได้
      BlogEngine.current_author_proc&.call(self)
    end
  end
end

# app/models/blog_engine/post.rb
module BlogEngine
  class Post < ApplicationRecord
    # ใช้ชื่อ table ที่มี prefix
    # table_name จะเป็น blog_engine_posts โดยอัตโนมัติ
    
    belongs_to :author, class_name: BlogEngine.author_class
    has_many :comments, dependent: :destroy
    has_many :post_tags, dependent: :destroy
    has_many :tags, through: :post_tags
    
    has_rich_text :content
    
    validates :title, presence: true, length: { maximum: 255 }
    validates :content, presence: true
    validates :status, inclusion: { in: %w[draft published archived] }
    
    scope :published, -> { where(status: 'published').where('published_at <= ?', Time.current) }
    scope :recent, -> { order(created_at: :desc) }
    
    before_save :set_slug
    
    def published?
      status == 'published' && published_at.present? && published_at <= Time.current
    end
    
    def to_param
      slug
    end
    
    private
    
    def set_slug
      self.slug = title.parameterize if title_changed?
    end
  end
end
```

---

## Step 1624: Configuration และ Hooks

```ruby
# lib/blog_engine.rb
require 'blog_engine/engine'

module BlogEngine
  # Configuration
  mattr_accessor :author_class
  self.author_class = 'User'
  
  mattr_accessor :current_author_proc
  
  mattr_accessor :per_page
  self.per_page = 10
  
  mattr_accessor :allowed_tags
  self.allowed_tags = %w[b i u a p br blockquote code pre img]
  
  # Setup block
  def self.setup
    yield self
  end
  
  # Helper สำหรับ mount
  def self.mount_path
    '/blog'
  end
end

# config/initializers/blog_engine.rb ใน host app
BlogEngine.setup do |config|
  config.author_class = 'User'
  
  # กำหนด current_author
  config.current_author_proc = proc { |controller|
    controller.current_user
  }
  
  config.per_page = 15
end

# app/controllers/blog_engine/application_controller.rb
# Override เพื่อใช้ authentication จาก host app
module BlogEngine
  class ApplicationController < ActionController::Base
    # ใช้ method จาก host app
    def current_author
      BlogEngine.current_author_proc&.call(self)
    end
    
    # Allow host app's helpers
    helper_method :current_author
  end
end
```

---

## Step 1625: Routing ใน Engine

```ruby
# config/routes.rb ใน engine
BlogEngine::Engine.routes.draw do
  resources :posts, param: :slug do
    resources :comments, only: [:create, :destroy]
    
    member do
      patch :publish
      patch :unpublish
    end
    
    collection do
      get :drafts
      get :archived
    end
  end
  
  resources :tags, only: [:index, :show]
  resources :categories, only: [:index, :show]
  
  root to: 'posts#index'
end

# config/routes.rb ใน host app
Rails.application.routes.draw do
  mount BlogEngine::Engine => '/blog', as: 'blog_engine'
  
  # routes อื่นๆ
  root to: 'home#index'
end

# การใช้ URL helpers ใน engine views
# ใช้ main_app สำหรับ host app routes
# ใช้ blog_engine สำหรับ engine routes

# ใน engine view
link_to 'Posts', blog_engine.posts_path
link_to 'Home', main_app.root_path
link_to 'Sign In', main_app.new_session_path

# ใน engine controller
redirect_to blog_engine.posts_path
redirect_to main_app.root_path

# Cross-engine URL helper
# ใน host app view/controller
link_to 'Blog', blog_engine.root_path
blog_engine.post_path(@post)
```

---

## Step 1626: Migrations ใน Engine

```ruby
# db/migrate/20240101000001_create_blog_engine_posts.rb
class CreateBlogEnginePosts < ActiveRecord::Migration[7.1]
  def change
    create_table :blog_engine_posts do |t|
      t.string :title, null: false
      t.string :slug, null: false
      t.text :excerpt
      t.string :status, default: 'draft', null: false
      t.datetime :published_at
      t.bigint :author_id
      t.string :author_type
      t.integer :comments_count, default: 0
      t.jsonb :metadata, default: {}
      
      t.timestamps
    end
    
    add_index :blog_engine_posts, :slug, unique: true
    add_index :blog_engine_posts, :status
    add_index :blog_engine_posts, :published_at
    add_index :blog_engine_posts, [:author_id, :author_type]
  end
end

# การ run migrations ใน host app
# Gemfile ของ host app
gem 'blog_engine', path: '../blog_engine'

# Terminal
rails blog_engine:install:migrations
rails db:migrate

# หรือ ใน engine's engine.rb
initializer :append_migrations do |app|
  unless app.root.to_s == root.to_s
    config.paths['db/migrate'].expanded.each do |expanded_path|
      app.config.paths['db/migrate'] << expanded_path
    end
  end
end
```

---

## Step 1627: Assets ใน Engine

```ruby
# app/assets/stylesheets/blog_engine/application.css
/*
 *= require_tree .
 *= require_self
 */

.blog-post {
  max-width: 800px;
  margin: 0 auto;
}

.blog-post-title {
  font-size: 2rem;
  font-weight: bold;
}

# app/assets/javascripts/blog_engine/application.js
//= require_tree .

# Sprockets manifest
# app/assets/config/blog_engine_manifest.js
//= link blog_engine/application.css
//= link blog_engine/application.js

# engine.rb - precompile assets
initializer 'blog_engine.assets.precompile' do |app|
  app.config.assets.precompile += %w[
    blog_engine/application.css
    blog_engine/application.js
  ]
end

# สำหรับ Importmap
# config/importmap.rb ใน engine
pin 'blog_engine', to: 'blog_engine/application.js'
pin_all_from BlogEngine::Engine.root.join('app/javascript'), 
  under: 'blog_engine',
  to: 'blog_engine'

# engine.rb
initializer 'blog_engine.importmap', before: 'importmap' do |app|
  app.config.importmap.paths << root.join('config/importmap.rb')
end
```

---

## Step 1628: Testing Engine

```ruby
# test/dummy/ - mini app สำหรับ test
# test/dummy/config/routes.rb
Rails.application.routes.draw do
  mount BlogEngine::Engine => '/blog'
end

# spec/rails_helper.rb ใน engine
require 'spec_helper'
require File.expand_path('../test/dummy/config/environment', __dir__)
require 'rspec/rails'
require 'factory_bot_rails'
require 'capybara/rspec'

RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
  config.include BlogEngine::Engine.routes.url_helpers
  
  config.before(:each) do
    BlogEngine.setup do |c|
      c.current_author_proc = proc { |controller| create(:user) }
    end
  end
end

# spec/factories/blog_engine/posts.rb
FactoryBot.define do
  factory :post, class: 'BlogEngine::Post' do
    title { Faker::Lorem.sentence }
    content { Faker::Lorem.paragraphs(number: 3).join("\n\n") }
    status { 'draft' }
    association :author, factory: :user
    
    trait :published do
      status { 'published' }
      published_at { 1.day.ago }
    end
    
    trait :with_comments do
      after(:create) do |post|
        create_list(:comment, 3, post: post)
      end
    end
  end
end

# spec/requests/blog_engine/posts_spec.rb
require 'rails_helper'

RSpec.describe 'BlogEngine Posts', type: :request do
  describe 'GET /blog/posts' do
    let!(:posts) { create_list(:post, 5, :published) }
    
    it 'returns published posts' do
      get blog_engine.posts_path
      
      expect(response).to have_http_status(:ok)
      posts.each do |post|
        expect(response.body).to include(post.title)
      end
    end
  end
  
  describe 'GET /blog/posts/:slug' do
    let(:post) { create(:post, :published) }
    
    it 'shows post details' do
      get blog_engine.post_path(post)
      
      expect(response).to have_http_status(:ok)
      expect(response.body).to include(post.title)
    end
  end
end
```

---

## Step 1629: Gemify Engine

```ruby
# blog_engine.gemspec
require_relative 'lib/blog_engine/version'

Gem::Specification.new do |spec|
  spec.name        = 'blog_engine'
  spec.version     = BlogEngine::VERSION
  spec.authors     = ['Your Name']
  spec.email       = ['your@email.com']
  spec.summary     = 'A full-featured blog engine for Rails'
  spec.description = 'Mountable blog engine with posts, comments, tags'
  spec.homepage    = 'https://github.com/yourname/blog_engine'
  spec.license     = 'MIT'
  
  spec.files = Dir[
    '{app,config,db,lib}/**/*',
    'MIT-LICENSE',
    'Rakefile',
    'README.md'
  ]
  
  spec.add_dependency 'rails', '>= 7.0'
  spec.add_dependency 'kaminari'
  spec.add_dependency 'ransack'
  
  spec.add_development_dependency 'rspec-rails'
  spec.add_development_dependency 'factory_bot_rails'
  spec.add_development_dependency 'faker'
  spec.add_development_dependency 'capybara'
end

# Publish ไปที่ RubyGems
gem build blog_engine.gemspec
gem push blog_engine-1.0.0.gem

# ใช้ใน Gemfile
gem 'blog_engine', '~> 1.0'

# หรือจาก git
gem 'blog_engine', git: 'https://github.com/yourname/blog_engine'

# หรือ local development
gem 'blog_engine', path: '../blog_engine'
```

---

## Step 1630: Engine กับ Concerns และ Hooks

```ruby
# lib/blog_engine/acts_as_author.rb
module BlogEngine
  module ActsAsAuthor
    extend ActiveSupport::Concern
    
    included do
      has_many :blog_posts,
        class_name: 'BlogEngine::Post',
        foreign_key: :author_id,
        foreign_type: :author_type,
        as: :author
    end
    
    def published_posts
      blog_posts.published
    end
    
    def draft_posts
      blog_posts.where(status: 'draft')
    end
  end
end

# Host app User model ใช้งาน concern จาก engine
class User < ApplicationRecord
  include BlogEngine::ActsAsAuthor
end

# หรือ Auto-include ผ่าน hook ใน engine
initializer 'blog_engine.extend_models' do
  ActiveSupport.on_load(:active_record) do
    # ไม่ auto-include เพราะไม่รู้ว่า User class ชื่ออะไร
    # ให้ host app เรียกเอง
  end
end

# Generator สำหรับสร้าง config file
# lib/generators/blog_engine/install/install_generator.rb
module BlogEngine
  module Generators
    class InstallGenerator < Rails::Generators::Base
      source_root File.expand_path('templates', __dir__)
      
      def copy_initializer
        template 'initializer.rb', 'config/initializers/blog_engine.rb'
      end
      
      def mount_engine
        route "mount BlogEngine::Engine => '/blog', as: 'blog_engine'"
      end
      
      def copy_migrations
        rake 'blog_engine:install:migrations'
      end
    end
  end
end

# lib/generators/blog_engine/install/templates/initializer.rb
BlogEngine.setup do |config|
  # Class ของ author (default: 'User')
  config.author_class = 'User'
  
  # Method ที่ใช้ดึง current user
  config.current_author_proc = proc { |controller|
    controller.current_user
  }
  
  # จำนวน posts ต่อหน้า
  config.per_page = 10
end
```

---

## Step 1631-1640: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: สร้าง Forum Engine

```bash
rails plugin new forum_engine --mountable
cd forum_engine
```

```ruby
# app/models/forum_engine/topic.rb
module ForumEngine
  class Topic < ApplicationRecord
    belongs_to :author, polymorphic: true
    belongs_to :category
    has_many :replies, dependent: :destroy
    
    validates :title, presence: true, length: { maximum: 255 }
    validates :body, presence: true
    
    scope :pinned, -> { where(pinned: true) }
    scope :recent, -> { order(last_reply_at: :desc) }
    
    counter_culture :category, column_name: :topics_count
  end
end

# app/models/forum_engine/reply.rb
module ForumEngine
  class Reply < ApplicationRecord
    belongs_to :topic, counter_cache: true, touch: true
    belongs_to :author, polymorphic: true
    
    validates :body, presence: true
    
    after_create :update_topic_last_reply
    
    private
    
    def update_topic_last_reply
      topic.update_columns(
        last_reply_at: created_at,
        last_reply_author_id: author_id,
        last_reply_author_type: author_type
      )
    end
  end
end
```

### แบบฝึกหัดที่ 2: E-Commerce Engine

```ruby
# สร้าง ShopEngine แบบ mountable
rails plugin new shop_engine --mountable

# app/models/shop_engine/product.rb
module ShopEngine
  class Product < ApplicationRecord
    belongs_to :category
    has_many :variants, dependent: :destroy
    has_many :cart_items
    
    validates :name, presence: true
    validates :base_price, numericality: { greater_than: 0 }
    
    scope :active, -> { where(active: true) }
    scope :in_stock, -> { joins(:variants).where('shop_engine_variants.stock > 0') }
    
    def in_stock?
      variants.where('stock > 0').exists?
    end
    
    def lowest_price
      variants.minimum(:price) || base_price
    end
  end
end

# config/routes.rb ใน engine
ShopEngine::Engine.routes.draw do
  resources :products, only: [:index, :show]
  resources :categories, only: [:index, :show]
  
  resource :cart, only: [:show] do
    post :add_item
    delete :remove_item
    patch :update_item
    post :checkout
  end
  
  resources :orders, only: [:index, :show]
  
  root to: 'products#index'
end
```

### แบบฝึกหัดที่ 3: Authentication Engine

```ruby
# สร้าง AuthEngine
rails plugin new auth_engine --mountable

# app/models/auth_engine/session.rb
module AuthEngine
  class Session < ApplicationRecord
    belongs_to :user, polymorphic: true
    
    before_create :generate_token
    
    scope :active, -> { where('expires_at > ?', Time.current) }
    
    def expired?
      expires_at < Time.current
    end
    
    def extend!
      update!(expires_at: 30.days.from_now)
    end
    
    private
    
    def generate_token
      self.token = SecureRandom.urlsafe_base64(32)
      self.expires_at = 30.days.from_now
    end
  end
end

# app/controllers/auth_engine/sessions_controller.rb
module AuthEngine
  class SessionsController < ApplicationController
    def new; end
    
    def create
      user = find_user(params[:email])
      
      if user&.authenticate(params[:password])
        session_record = Session.create!(user: user)
        cookies.signed[:session_token] = {
          value: session_record.token,
          expires: 30.days.from_now,
          httponly: true,
          secure: Rails.env.production?
        }
        redirect_to main_app.root_path, notice: 'เข้าสู่ระบบสำเร็จ'
      else
        flash.now[:alert] = 'อีเมลหรือรหัสผ่านไม่ถูกต้อง'
        render :new, status: :unprocessable_entity
      end
    end
    
    def destroy
      current_session&.destroy
      cookies.delete(:session_token)
      redirect_to auth_engine.new_session_path
    end
    
    private
    
    def find_user(email)
      user_class = AuthEngine.user_class.constantize
      user_class.find_by(email: email.downcase.strip)
    end
    
    def current_session
      @current_session ||= Session.active.find_by(
        token: cookies.signed[:session_token]
      )
    end
  end
end
```

### แบบฝึกหัดที่ 4: Notification Engine

```ruby
# app/models/notification_engine/notification.rb
module NotificationEngine
  class Notification < ApplicationRecord
    belongs_to :recipient, polymorphic: true
    belongs_to :subject, polymorphic: true, optional: true
    
    validates :message, presence: true
    validates :notification_type, presence: true
    
    scope :unread, -> { where(read_at: nil) }
    scope :recent, -> { order(created_at: :desc) }
    
    def read?
      read_at.present?
    end
    
    def mark_read!
      update!(read_at: Time.current) unless read?
    end
  end
end

# app/services/notification_engine/deliver_notification.rb
module NotificationEngine
  class DeliverNotification
    def self.call(recipient:, message:, notification_type:, subject: nil)
      notification = Notification.create!(
        recipient: recipient,
        message: message,
        notification_type: notification_type,
        subject: subject
      )
      
      # Broadcast ผ่าน Action Cable
      ActionCable.server.broadcast(
        "notifications_#{recipient.class.name.downcase}_#{recipient.id}",
        { notification: notification.as_json }
      )
      
      notification
    end
  end
end
```

### แบบฝึกหัดที่ 5: CMS Engine

```ruby
# Full CMS Engine
module CmsEngine
  class Engine < ::Rails::Engine
    isolate_namespace CmsEngine
    
    # Auto-mount routes
    initializer 'cms_engine.routes' do |app|
      app.routes.prepend do
        mount CmsEngine::Engine => CmsEngine.mount_path, as: 'cms_engine'
      end
    end
  end
end

# app/models/cms_engine/page.rb
module CmsEngine
  class Page < ApplicationRecord
    has_rich_text :content
    has_one_attached :featured_image
    
    validates :title, presence: true
    validates :slug, presence: true, uniqueness: true
    
    scope :published, -> { where(published: true) }
    
    before_validation :generate_slug, if: :title_changed?
    
    def to_param
      slug
    end
    
    private
    
    def generate_slug
      base_slug = title.parameterize
      self.slug = unique_slug(base_slug)
    end
    
    def unique_slug(base)
      count = Page.where("slug LIKE ?", "#{base}%").count
      count.zero? ? base : "#{base}-#{count + 1}"
    end
  end
end
```

### แบบฝึกหัดที่ 6-10: Engine ขั้นสูง

```ruby
# แบบฝึกหัดที่ 6: Engine กับ Multiple Databases
module AnalyticsEngine
  class Engine < ::Rails::Engine
    isolate_namespace AnalyticsEngine
    
    # ใช้ secondary database
    config.after_initialize do
      AnalyticsEngine::Event.connects_to database: {
        writing: :analytics,
        reading: :analytics
      }
    end
  end
end

# แบบฝึกหัดที่ 7: Engine กับ GraphQL
module ApiEngine
  class Engine < ::Rails::Engine
    isolate_namespace ApiEngine
  end
end

# config/routes.rb ใน engine
ApiEngine::Engine.routes.draw do
  post '/graphql', to: 'graphql#execute'
  get '/graphiql', to: 'graphiql#show' if Rails.env.development?
end

# แบบฝึกหัดที่ 8: Engine กับ Webhooks
module WebhookEngine
  class Endpoint < ApplicationRecord
    validates :url, presence: true, url: true
    validates :secret, presence: true
    
    before_create :generate_secret
    
    def deliver!(event_type, payload)
      WebhookDeliveryJob.perform_later(id, event_type, payload)
    end
    
    private
    
    def generate_secret
      self.secret = "wh_#{SecureRandom.hex(32)}"
    end
  end
end

# แบบฝึกหัดที่ 9: Engine กับ Multi-step Forms
module WizardEngine
  class Wizard
    include ActiveModel::Model
    
    STEPS = %w[basic details confirmation].freeze
    
    attr_accessor :current_step, :data
    
    def next_step
      STEPS[STEPS.index(current_step) + 1]
    end
    
    def previous_step
      idx = STEPS.index(current_step)
      return nil if idx.zero?
      STEPS[idx - 1]
    end
    
    def last_step?
      current_step == STEPS.last
    end
    
    def first_step?
      current_step == STEPS.first
    end
  end
end

# แบบฝึกหัดที่ 10: Engine Testing กับ Shared Examples
RSpec.shared_examples 'a mountable engine' do
  it 'has an isolated namespace' do
    expect(described_class).to respond_to(:isolate_namespace)
  end
  
  it 'has routes' do
    expect(described_class.routes).to be_a(ActionDispatch::Routing::RouteSet)
  end
  
  it 'has a root path' do
    expect { described_class.routes.url_helpers.root_path }.not_to raise_error
  end
end

RSpec.describe BlogEngine::Engine do
  it_behaves_like 'a mountable engine'
end
```

### แบบฝึกหัดที่ 11-20: สรุปโปรเจกต์

```ruby
# โปรเจกต์ Final: Modular Monolith ด้วย Engines

# app หลัก + 3 engines
# 1. AuthEngine - Authentication/Authorization
# 2. BlogEngine - Content Management
# 3. ShopEngine - E-Commerce

# config/routes.rb ใน host app
Rails.application.routes.draw do
  mount AuthEngine::Engine => '/auth', as: 'auth_engine'
  mount BlogEngine::Engine => '/blog', as: 'blog_engine'
  mount ShopEngine::Engine => '/shop', as: 'shop_engine'
  
  root to: 'home#index'
  
  namespace :admin do
    resources :users
    resources :settings
  end
end

# Gemfile
gem 'auth_engine', path: 'engines/auth_engine'
gem 'blog_engine', path: 'engines/blog_engine'
gem 'shop_engine', path: 'engines/shop_engine'

# engines/ directory structure
engines/
├── auth_engine/
│   ├── app/
│   ├── config/
│   ├── db/
│   ├── lib/
│   └── auth_engine.gemspec
├── blog_engine/
│   ├── app/
│   ├── config/
│   ├── db/
│   ├── lib/
│   └── blog_engine.gemspec
└── shop_engine/
    ├── app/
    ├── config/
    ├── db/
    ├── lib/
    └── shop_engine.gemspec

# Communication ระหว่าง engines ผ่าน events
# lib/auth_engine/events/user_registered.rb
module AuthEngine
  module Events
    class UserRegistered
      attr_reader :user
      
      def initialize(user)
        @user = user
      end
    end
  end
end

# Publish event
class AuthEngine::RegistrationsController < ApplicationController
  def create
    user = User.create!(user_params)
    
    event = AuthEngine::Events::UserRegistered.new(user)
    ActiveSupport::Notifications.instrument('auth_engine.user_registered', event: event)
    
    # ...
  end
end

# Subscribe ใน BlogEngine
# config/initializers/blog_engine.rb
ActiveSupport::Notifications.subscribe('auth_engine.user_registered') do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  user = event.payload[:event].user
  
  # สร้าง blog profile สำหรับ user ใหม่
  BlogEngine::Author.create!(
    user_id: user.id,
    display_name: user.name
  )
end
```

---

## สรุป Part 75: Rails Engines

Rails Engines เป็นเครื่องมือที่ทรงพลังสำหรับการจัดการ codebase ขนาดใหญ่:

1. **Mountable vs Full Engine**: ใช้ Mountable เมื่อต้องการ isolated namespace
2. **Configuration Hooks**: ให้ host app customize behavior ผ่าน `mattr_accessor`
3. **URL Helpers**: ใช้ `main_app.` สำหรับ host routes และ `engine_name.` สำหรับ engine routes
4. **Assets**: precompile assets ใน engine initializer
5. **Migrations**: copy migrations ไปยัง host app ก่อน migrate
6. **Testing**: ใช้ dummy app ใน test directory
7. **Communication**: ใช้ ActiveSupport::Notifications สำหรับ loose coupling

---

**จบ Part 75: Rails Engines และ Rails Advanced Course**

*คุณได้เรียนรู้ Rails ในระดับขั้นสูงครบทุก topic แล้ว ตั้งแต่ Service Objects, Advanced ActiveRecord, API Authentication, Testing, Admin Panel, Payments, Notifications, Multi-tenancy, Analytics ไปจนถึง Rails Engines*
