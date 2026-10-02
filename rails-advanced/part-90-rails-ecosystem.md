# Part 90: Rails Ecosystem ปี 2024-2025

## บทนำ

Rails ecosystem พัฒนาอย่างต่อเนื่อง ในปี 2024-2025 มีเครื่องมือใหม่ๆ และ patterns ที่น่าสนใจมากมาย บทนี้จะครอบคลุม Hotwire ขั้นสูง, Rails 8 features, alternative frameworks และ Ruby tooling ล่าสุด

## 1. Rails Ecosystem Overview ปี 2024-2025

### 1.1 สถานะปัจจุบันของ Rails

```ruby
# Rails 8.0 features หลัก (released late 2024)
# 1. Solid Queue - Database-backed job queue (ไม่ต้องใช้ Redis)
# 2. Solid Cache - Database-backed cache store  
# 3. Solid Cable - Database-backed Action Cable adapter
# 4. Kamal 2.0 - Docker deployment tool
# 5. Propshaft - Modern asset pipeline

# Gemfile สำหรับ Rails 8
source "https://rubygems.org"

gem "rails", "~> 8.0"
gem "propshaft"        # Asset pipeline (แทน Sprockets)
gem "solid_queue"      # Background jobs
gem "solid_cache"      # Caching
gem "solid_cable"      # WebSockets
gem "kamal"            # Deployment
gem "thruster"         # HTTP asset caching

gem "turbo-rails"      # Hotwire Turbo
gem "stimulus-rails"   # Hotwire Stimulus
```

### 1.2 Rails 8 Default Stack

```bash
# สร้าง Rails 8 app ด้วย new defaults
rails new myapp --database=sqlite3

# Rails 8 ใช้ SQLite สำหรับทุกอย่างโดย default:
# - Primary database: SQLite (via Active Record)
# - Cache: Solid Cache (SQLite)
# - Jobs: Solid Queue (SQLite)
# - WebSockets: Solid Cable (SQLite)

# ไม่ต้องการ Redis, Memcached, or Redis ใดๆ สำหรับ small-to-medium apps
```

## 2. Hotwire (Turbo + Stimulus) ขั้นสูง

### 2.1 Turbo Streams ขั้นสูง

```ruby
# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  def create
    @message = current_user.messages.build(message_params)
    
    if @message.save
      # Broadcast ไปยัง multiple streams พร้อมกัน
      respond_to do |format|
        format.turbo_stream do
          render turbo_stream: [
            # เพิ่ม message ใน chat feed
            turbo_stream.append("messages", @message),
            # Reset form
            turbo_stream.replace("new_message_form", 
              partial: "messages/form",
              locals: { message: Message.new }),
            # อัพเดท message count
            turbo_stream.replace("message_count",
              partial: "shared/message_count",
              locals: { count: Message.count })
          ]
        end
        format.html { redirect_to messages_path }
      end
    else
      respond_to do |format|
        format.turbo_stream do
          render turbo_stream: turbo_stream.replace(
            "new_message_form",
            partial: "messages/form",
            locals: { message: @message }
          )
        end
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end
  
  private
  
  def message_params
    params.require(:message).permit(:body, :channel_id)
  end
end
```

### 2.2 Turbo Broadcasts (Real-time Updates)

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :user
  belongs_to :channel
  
  after_create_commit -> { broadcast_append_to "messages" }
  after_update_commit -> { broadcast_replace_to "messages" }
  after_destroy_commit -> { broadcast_remove_to "messages" }
  
  # หรือใช้ broadcast_* helpers
  # broadcasts_to :channel  # broadcast ไปยัง channel stream
  
  # Custom broadcast สำหรับ complex scenarios
  after_create_commit :broadcast_to_all_channels
  
  private
  
  def broadcast_to_all_channel
    # Broadcast ไปยัง channel ที่ message อยู่
    broadcast_append_to(
      [channel, "messages"],
      target: "channel_#{channel_id}_messages",
      partial: "messages/message",
      locals: { message: self }
    )
    
    # Broadcast ไปยัง user's DM ถ้ามี
    if channel.direct_message?
      channel.users.each do |user|
        broadcast_append_to(
          [user, "notifications"],
          target: "notification_count",
          partial: "notifications/count",
          locals: { user: user }
        )
      end
    end
  end
end

# app/views/channels/show.html.erb
<%= turbo_stream_from [@channel, "messages"] %>

<div id="<%= dom_id(@channel, :messages) %>">
  <%= render @messages %>
</div>

<%= turbo_frame_tag "new_message" do %>
  <%= render "messages/form", message: @new_message, channel: @channel %>
<% end %>
```

### 2.3 Turbo Frames สำหรับ Lazy Loading

```ruby
# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  def index
    @products = Product.published.page(params[:page]).per(20)
  end
  
  def details
    @product = Product.find(params[:id])
    
    respond_to do |format|
      format.turbo_stream
      format.html
    end
  end
end

# app/views/products/index.html.erb
<div id="products-grid">
  <% @products.each do |product| %>
    <%= turbo_frame_tag dom_id(product) do %>
      <%= render "products/card", product: product %>
    <% end %>
  <% end %>
</div>

<%# Infinite scroll with Turbo %>
<%= turbo_frame_tag "pagination", loading: :lazy, 
    src: products_path(page: @products.next_page) if @products.next_page %>

# app/views/products/details.html.erb (turbo_stream)
<%= turbo_stream.replace dom_id(@product) do %>
  <%= render "products/expanded_card", product: @product %>
<% end %>
```

### 2.4 Stimulus Controllers ขั้นสูง

```javascript
// app/javascript/controllers/search_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "results", "loading"]
  static values = {
    url: String,
    delay: { type: Number, default: 300 }
  }
  
  connect() {
    this.debouncedSearch = this.debounce(this.performSearch.bind(this), this.delayValue)
  }
  
  search() {
    this.debouncedSearch()
  }
  
  async performSearch() {
    const query = this.inputTarget.value.trim()
    
    if (query.length < 2) {
      this.resultsTarget.innerHTML = ""
      return
    }
    
    this.loadingTarget.classList.remove("hidden")
    
    try {
      const response = await fetch(`${this.urlValue}?q=${encodeURIComponent(query)}`, {
        headers: {
          "Accept": "text/vnd.turbo-stream.html",
          "X-Requested-With": "XMLHttpRequest"
        }
      })
      
      if (response.ok) {
        const html = await response.text()
        Turbo.renderStreamMessage(html)
      }
    } catch (error) {
      console.error("Search failed:", error)
    } finally {
      this.loadingTarget.classList.add("hidden")
    }
  }
  
  debounce(func, wait) {
    let timeout
    return function executedFunction(...args) {
      const later = () => {
        clearTimeout(timeout)
        func(...args)
      }
      clearTimeout(timeout)
      timeout = setTimeout(later, wait)
    }
  }
}

// app/javascript/controllers/form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["submit", "field"]
  static classes = ["loading"]
  
  connect() {
    this.element.addEventListener("turbo:submit-start", () => {
      this.submitTarget.disabled = true
      this.submitTarget.classList.add(...this.loadingClasses)
    })
    
    this.element.addEventListener("turbo:submit-end", () => {
      this.submitTarget.disabled = false
      this.submitTarget.classList.remove(...this.loadingClasses)
    })
  }
  
  validate() {
    const isValid = this.fieldTargets.every(field => field.reportValidity())
    this.submitTarget.disabled = !isValid
  }
  
  autoSave() {
    clearTimeout(this.autoSaveTimer)
    this.autoSaveTimer = setTimeout(() => {
      this.element.requestSubmit()
    }, 2000)
  }
}
```

### 2.5 Custom Turbo Stream Actions

```javascript
// app/javascript/turbo_custom_actions.js
import { StreamActions } from "@hotwired/turbo"

// Custom action: ล้าง form fields
StreamActions.clear = function() {
  this.targetElements.forEach(element => {
    element.querySelectorAll("input, textarea, select").forEach(field => {
      field.value = ""
    })
  })
}

// Custom action: แสดง toast notification
StreamActions.toast = function() {
  const message = this.getAttribute("message")
  const type = this.getAttribute("type") || "info"
  
  const toast = document.createElement("div")
  toast.className = `toast toast-${type}`
  toast.textContent = message
  
  document.body.appendChild(toast)
  
  setTimeout(() => toast.remove(), 3000)
}

// Custom action: scroll to element
StreamActions.scroll_to = function() {
  this.targetElements.forEach(element => {
    element.scrollIntoView({ behavior: "smooth" })
  })
}
```

```ruby
# ใช้งาน custom actions ใน controller
def create
  @post = Post.create!(post_params)
  
  respond_to do |format|
    format.turbo_stream do
      render turbo_stream: [
        turbo_stream.append("posts", @post),
        # ใช้ custom action 'toast'
        '<turbo-stream action="toast" message="Post created!" type="success"></turbo-stream>'.html_safe,
        # ใช้ custom action 'clear'
        '<turbo-stream action="clear" target="new_post_form"></turbo-stream>'.html_safe
      ]
    end
  end
end
```

## 3. Rails 8 Features ใหม่

### 3.1 Solid Queue (Database-backed Jobs)

```ruby
# config/application.rb
config.active_job.queue_adapter = :solid_queue

# config/solid_queue.yml
default: &default
  workers:
    - queues: [default, mailers]
      threads: 3
    - queues: [critical]
      threads: 5
  recurring_tasks:
    report_generation:
      class: GenerateReportJob
      schedule: "0 9 * * 1-5"  # Weekdays at 9am

production:
  <<: *default
  workers:
    - queues: [critical]
      threads: 10
    - queues: [default, mailers, notifications]
      threads: 5

# Start Solid Queue
# rails solid_queue:start

# Solid Queue Dashboard (via gem)
gem 'mission_control-jobs'

# config/routes.rb
mount MissionControl::Jobs::Engine, at: "/jobs"
```

### 3.2 Solid Cache

```ruby
# config/environments/production.rb
config.cache_store = :solid_cache_store

# Solid Cache ทำงานเหมือน Redis cache แต่ใช้ SQLite
Rails.cache.write("key", "value", expires_in: 1.hour)
Rails.cache.read("key")  # => "value"
Rails.cache.fetch("expensive_key") { compute_expensive_thing }

# Solid Cache มี automatic size management
# config/solid_cache.yml
default: &default
  size: 256.megabytes
  
production:
  <<: *default
  size: 1.gigabyte
  databases:
    cache: cache  # ชี้ไปที่ database config
```

### 3.3 Propshaft (Modern Asset Pipeline)

```ruby
# Gemfile
gem "propshaft"  # แทน sprockets

# Propshaft ทำงานอย่างไร:
# 1. ไม่มี compilation - files ถูก serve ตรงๆ
# 2. Digest fingerprinting สำหรับ cache busting
# 3. รองรับ importmaps และ bundling tools

# app/assets/javascripts/app.js - ไม่มี asset pipeline complexity
import { Application } from "@hotwired/stimulus"
import controllers from "./controllers/**/*_controller.js"
```

### 3.4 Kamal 2.0 Deployment

```yaml
# config/deploy.yml
service: myapp
image: yourusername/myapp

servers:
  web:
    hosts:
      - 192.168.1.1
    labels:
      traefik.http.routers.myapp.rule: Host(`myapp.com`)
  job:
    hosts:
      - 192.168.1.2
    cmd: bundle exec solid_queue

proxy:
  ssl: true
  host: myapp.com

registry:
  username: yourusername
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    DB_HOST: 192.168.1.3
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_PASSWORD

accessories:
  db:
    image: postgres:16
    host: 192.168.1.3
    port: 5432
    env:
      secret:
        - POSTGRES_PASSWORD
    volumes:
      - data:/var/lib/postgresql/data
```

```bash
# Kamal commands
kamal setup           # First-time setup
kamal deploy          # Deploy new version
kamal rollback        # Rollback to previous
kamal app logs        # View logs
kamal app exec --interactive -- bash  # SSH into container
```

### 3.5 Authentication Generator (Rails 8)

```bash
# Rails 8 มี built-in authentication generator
rails generate authentication

# สร้าง:
# - User model พร้อม password_digest
# - Session model
# - Authentication concern
# - Passwords controller
# - Sessions controller
```

```ruby
# app/models/concerns/authentication.rb (auto-generated)
module Authentication
  extend ActiveSupport::Concern
  
  included do
    before_action :require_authentication
    helper_method :authenticated?
  end
  
  private
  
  def authenticated?
    resume_session
  end
  
  def require_authentication
    resume_session || request_authentication
  end
  
  def resume_session
    Current.session ||= find_session_by_cookie
  end
  
  def find_session_by_cookie
    Session.find_by(id: cookies.signed[:session_id])&.tap do |session|
      Current.session = session
    end
  end
  
  def request_authentication
    session[:return_to_after_authenticating] = request.url
    redirect_to new_session_url
  end
  
  def after_authentication_url
    session.delete(:return_to_after_authenticating) || root_url
  end
  
  def start_new_session_for(user)
    user.sessions.create!(user_agent: request.user_agent, ip_address: request.remote_ip).tap do |session|
      Current.session = session
      cookies.signed.permanent[:session_id] = { value: session.id, httponly: true, same_site: :lax }
    end
  end
  
  def terminate_session
    Current.session.destroy
    cookies.delete(:session_id)
  end
end
```

## 4. Alternative Frameworks: Hanami 2.x vs Rails

### 4.1 Hanami 2.x Overview

```ruby
# Gemfile
gem "hanami", "~> 2.1"
gem "hanami-router"
gem "hanami-controller"
gem "hanami-view"

# Hanami มี architecture ที่แตกต่างจาก Rails:
# - Dependency injection แทน global state
# - Explicit configuration แทน convention magic
# - Slices แทน engines

# config/app.rb
require "hanami"

module MyApp
  class App < Hanami::App
    config.actions.default_response_format = :json
    config.logger.level = :info
    
    # Dependency injection
    config.providers do
      provider :database do
        prepare do
          require "sequel"
          Sequel.connect(target["settings"].database_url)
        end
        
        start do
          register "database", Sequel::Model.db
        end
      end
    end
  end
end
```

### 4.2 Hanami Actions vs Rails Controllers

```ruby
# Hanami Action
module MyApp
  module Actions
    module Users
      class Create < Hanami::Action
        include Deps[
          "repositories.users",
          "mailers.user_mailer"
        ]
        
        params do
          required(:user).hash do
            required(:email).filled(:string)
            required(:password).filled(:string, min_size?: 8)
            required(:name).filled(:string)
          end
        end
        
        def handle(request, response)
          # Validation แบบ explicit
          halt 422, { errors: request.params.errors.to_h } unless request.params.valid?
          
          user = users.create(request.params[:user])
          user_mailer.welcome(user).deliver_later
          
          response.status = 201
          response.body = { user: UserSerializer.new(user).to_h }.to_json
        end
      end
    end
  end
end

# เทียบกับ Rails Controller
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    
    if @user.save
      UserMailer.welcome(@user).deliver_later
      render json: UserSerializer.new(@user), status: :created
    else
      render json: { errors: @user.errors }, status: :unprocessable_entity
    end
  end
  
  private
  
  def user_params
    params.require(:user).permit(:email, :password, :name)
  end
end
```

### 4.3 Hanami Views

```ruby
# app/views/users/show.rb (Hanami View)
module MyApp
  module Views
    module Users
      class Show < Hanami::View
        include Deps["repositories.users"]
        
        expose :user do |id:|
          users.find(id)
        end
        
        expose :recent_posts do |user:|
          user.posts.recent.limit(5)
        end
      end
    end
  end
end

# app/templates/users/show.html.erb
<h1><%= user.name %></h1>
<% recent_posts.each do |post| %>
  <article>
    <h2><%= post.title %></h2>
    <p><%= post.excerpt %></p>
  </article>
<% end %>
```

### 4.4 เมื่อไหรควรเลือก Hanami vs Rails

| Feature | Rails | Hanami |
|---------|-------|--------|
| Learning curve | ต่ำ | สูงกว่า |
| Convention over config | มาก | น้อยกว่า |
| Performance | ดี | ดีมาก |
| Testing | ง่าย (Rails helpers) | Explicit, เร็วกว่า |
| Large team | Convention ช่วยได้ | DI ช่วยได้ |
| Microservices | ได้ (Rails API) | เหมาะมาก |

## 5. Roda สำหรับ Lightweight Apps

### 5.1 Roda Basics

```ruby
# Gemfile
gem "roda"
gem "sequel"
gem "sequel_pg"  # PostgreSQL adapter

# app.rb
require "roda"
require "sequel"

DB = Sequel.connect(ENV["DATABASE_URL"])

class App < Roda
  plugin :json
  plugin :json_parser
  plugin :halt
  plugin :all_verbs
  plugin :request_headers
  
  # Authentication middleware
  plugin :middleware do |app|
    app.use Rack::Auth::Basic do |username, password|
      User.authenticate(username, password)
    end
  end
  
  route do |r|
    # API routes
    r.on "api" do
      r.on "v1" do
        r.on "users" do
          r.get do
            # GET /api/v1/users
            DB[:users].all.to_json
          end
          
          r.post do
            # POST /api/v1/users
            data = r.params
            
            user_id = DB[:users].insert(
              email: data["email"],
              name: data["name"],
              created_at: Time.now
            )
            
            response.status = 201
            DB[:users].where(id: user_id).first.to_json
          end
          
          r.on Integer do |user_id|
            user = DB[:users].where(id: user_id).first
            r.halt(404, { error: "Not found" }.to_json) unless user
            
            r.get do
              # GET /api/v1/users/:id
              user.to_json
            end
            
            r.put do
              # PUT /api/v1/users/:id
              DB[:users].where(id: user_id).update(r.params)
              DB[:users].where(id: user_id).first.to_json
            end
            
            r.delete do
              # DELETE /api/v1/users/:id
              DB[:users].where(id: user_id).delete
              response.status = 204
            end
          end
        end
      end
    end
  end
end

# config.ru
require_relative "app"
run App.freeze.app
```

### 5.2 Roda Plugins

```ruby
# app.rb - Roda พร้อม plugins
class App < Roda
  # Security
  plugin :content_security_policy do |csp|
    csp.default_src :self
    csp.script_src :self, :unsafe_inline
    csp.style_src :self
    csp.img_src :self, "data:"
  end
  
  # Sessions
  plugin :sessions, secret: ENV["SESSION_SECRET"]
  
  # Rate limiting
  plugin :throttle_by do |r|
    [r.ip, r.path]  # Throttle by IP + path
  end
  
  # Caching
  plugin :caching
  
  route do |r|
    # Use caching
    r.on "products" do
      r.get do
        r.last_modified(Product.maximum(:updated_at))
        r.etag(Product.cache_key)
        
        # 304 ถ้า cache valid
        r.halt(304) if r.fresh?
        
        Product.all.to_json
      end
    end
  end
end
```

### 5.3 Performance Comparison

```ruby
# Benchmark: Rails vs Hanami vs Roda
# (requests per second, approximate)
# 
# Simple JSON endpoint:
# Rails:   ~4,000 req/s
# Hanami:  ~7,000 req/s
# Roda:    ~15,000 req/s
# Sinatra: ~9,000 req/s
#
# Roda เร็วกว่า Rails มากเพราะ:
# - No middleware stack overhead
# - Tree routing (O(log n) แทน O(n))
# - Minimal object allocation
```

## 6. Ruby Tooling ล่าสุด

### 6.1 RuboCop 1.x Configuration

```yaml
# .rubocop.yml
require:
  - rubocop-rails
  - rubocop-rspec
  - rubocop-performance

AllCops:
  NewCops: enable
  TargetRubyVersion: 3.3
  Exclude:
    - "db/**/*"
    - "vendor/**/*"
    - "bin/**/*"
    - "node_modules/**/*"

# Style cops
Style/StringLiterals:
  Enabled: true
  EnforcedStyle: double_quotes

Style/Documentation:
  Enabled: false

Style/FrozenStringLiteralComment:
  Enabled: true
  EnforcedStyle: always

# Metrics cops
Metrics/MethodLength:
  Max: 20

Metrics/ClassLength:
  Max: 200

Metrics/BlockLength:
  Exclude:
    - "spec/**/*"
    - "config/routes.rb"

# Rails-specific cops
Rails/OutputSafety:
  Enabled: true

Rails/FindEach:
  Enabled: true

Rails/HasManyOrHasOneDependent:
  Enabled: true

# Performance cops
Performance/Count:
  Enabled: true

Performance/StringReplacement:
  Enabled: true

# RSpec cops
RSpec/ExampleLength:
  Max: 20

RSpec/MultipleExpectations:
  Max: 5

RSpec/NestedGroups:
  Max: 4
```

### 6.2 RBS (Ruby Signature)

```ruby
# RBS คือ type signature language สำหรับ Ruby
# ไฟล์ .rbs อธิบาย type ของ Ruby code

# sig/user.rbs
class User
  attr_accessor name: String
  attr_accessor email: String
  attr_reader id: Integer
  attr_reader created_at: Time
  
  def initialize: (name: String, email: String) -> void
  
  def full_name: () -> String
  
  def admin?: () -> bool
  
  def posts: () -> ActiveRecord::Associations::CollectionProxy[Post]
  
  def authenticate: (String password) -> bool
end

# sig/services/user_service.rbs
class UserService
  @user: User
  @mailer: UserMailer
  
  def initialize: (user: User) -> void
  
  def send_welcome_email: () -> ActionMailer::MessageDelivery
  
  def update_profile: (name: String, ?bio: String) -> User
  
  def self.create: (email: String, password: String, name: String) -> User
end

# sig/lib/calculator.rbs
class Calculator
  def add: (Numeric, Numeric) -> Numeric
  def subtract: (Numeric, Numeric) -> Numeric
  def multiply: (Numeric, Numeric) -> Numeric
  def divide: (Numeric, Numeric) -> Float
  
  # Union types
  def parse_input: (String | Integer | Float) -> Float
  
  # Generic types
  def sum: (Array[Numeric]) -> Numeric
end
```

### 6.3 Steep Type Checker

```ruby
# Gemfile
gem "steep", group: :development

# Steepfile
D = Steep::Diagnostic

target :app do
  signature "sig"
  
  check "app/models"
  check "app/services"
  check "lib"
  
  # Configure severity
  configure_code_diagnostics do |hash|
    hash[D::Ruby::UnexpectedPositionalArgument] = :error
    hash[D::Ruby::IncompatibleArguments] = :error
    hash[D::Ruby::UnexpectedKeywordArgument] = :warning
    hash[D::Ruby::NoMethod] = :error
  end
end

# ใช้งาน
# steep check
# steep watch  # watch mode
```

```ruby
# app/services/email_service.rb
class EmailService
  def initialize(user)
    @user = user
  end
  
  # Steep จะ check ว่า user เป็น User instance
  def send_confirmation
    UserMailer.confirmation(@user).deliver_later
  end
  
  # Type error ถ้า return type ไม่ match
  def user_name: () -> String
  def user_name
    @user.name  # ต้องเป็น String
  end
end
```

### 6.4 Sorbet Type Annotations

```ruby
# Gemfile
gem "sorbet"
gem "sorbet-runtime"
gem "tapioca"  # ช่วยสร้าง RBI files

# ใช้ Sorbet annotations
# typed: strict ที่ด้านบนของไฟล์

# app/services/payment_service.rb
# typed: strict

class PaymentService
  extend T::Sig
  
  sig { params(user: User, amount: Integer, currency: String).void }
  def initialize(user, amount, currency = "USD")
    @user = T.let(user, User)
    @amount = T.let(amount, Integer)
    @currency = T.let(currency, String)
  end
  
  sig { returns(T::Boolean) }
  def charge!
    stripe_charge = Stripe::Charge.create(
      amount: @amount,
      currency: @currency,
      customer: @user.stripe_customer_id
    )
    
    stripe_charge.status == "succeeded"
  rescue Stripe::CardError => e
    Rails.logger.error "Payment failed: #{e.message}"
    false
  end
  
  sig { params(plan: T.nilable(String)).returns(Stripe::Subscription) }
  def subscribe(plan = nil)
    plan_id = plan || "default_plan"
    
    Stripe::Subscription.create(
      customer: @user.stripe_customer_id,
      items: [{ price: plan_id }]
    )
  end
  
  # Union types
  sig { returns(T.any(String, Integer)) }
  def payment_id
    @payment_id || 0
  end
end
```

### 6.5 TypeProf: Type Inference

```bash
# TypeProf วิเคราะห์ code และ infer types โดยอัตโนมัติ
# gem install typeprof

# typeprof app/services/user_service.rb
# Output: RBS signature ที่ inferred จาก code

# ตัวอย่าง
typeprof lib/calculator.rb

# Output:
# sig/lib/calculator.rbs
# class Calculator
#   def add: (Integer | Float, Integer | Float) -> (Integer | Float)
#   def subtract: (Integer | Float, Integer | Float) -> (Integer | Float)
# end
```

### 6.6 Ruby LSP (Language Server Protocol)

```json
// .vscode/settings.json สำหรับ Ruby LSP
{
  "rubyLsp.enabledFeatures": {
    "diagnostics": true,
    "formatting": true,
    "completion": true,
    "hover": true,
    "inlayHints": true,
    "codeActions": true,
    "semanticHighlighting": true,
    "documentHighlights": true,
    "documentSymbols": true,
    "workspaceSymbols": true,
    "signatureHelp": true
  },
  "rubyLsp.rubyVersionManager": {
    "identifier": "rbenv"
  }
}
```

## 7. Rails DevOps และ Deployment Tools

### 7.1 Thruster (HTTP Asset Caching)

```yaml
# Dockerfile สำหรับ Rails 8 + Thruster
FROM ruby:3.3-slim

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev

WORKDIR /app

COPY Gemfile* .
RUN bundle install

COPY . .
RUN bundle exec rails assets:precompile

# Thruster จัดการ HTTP caching headers
EXPOSE 3000
CMD ["thrust", "bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```

### 7.2 Monitoring ด้วย Rails Instrumentation

```ruby
# config/initializers/monitoring.rb
ActiveSupport::Notifications.subscribe "process_action.action_controller" do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  
  Rails.logger.info({
    controller: event.payload[:controller],
    action: event.payload[:action],
    format: event.payload[:format],
    http_method: event.payload[:method],
    path: event.payload[:path],
    status: event.payload[:status],
    duration: event.duration.round(2),
    db_runtime: event.payload[:db_runtime]&.round(2),
    view_runtime: event.payload[:view_runtime]&.round(2)
  }.to_json)
end

# Custom metrics
ActiveSupport::Notifications.subscribe "payment.charged" do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  
  StatsD.increment("payments.charged")
  StatsD.timing("payments.processing_time", event.duration)
  StatsD.gauge("payments.amount", event.payload[:amount])
end

# Instrument ใน code
ActiveSupport::Notifications.instrument("payment.charged", 
                                        amount: charge.amount) do
  process_payment(charge)
end
```

## 8. Testing ใน Modern Rails

### 8.1 System Tests ด้วย Playwright

```ruby
# Gemfile
gem "playwright-ruby-client"
gem "capybara-playwright-driver"

# spec/spec_helper.rb
require "playwright"
require "capybara/playwright"

Capybara.javascript_driver = :playwright

RSpec.configure do |config|
  config.before(:each, js: true) do
    Capybara.current_driver = :playwright
  end
end

# spec/system/checkout_spec.rb
require "rails_helper"

RSpec.describe "Checkout process", type: :system, js: true do
  let(:user) { create(:user) }
  let(:product) { create(:product, price: 9999) }
  
  before { sign_in user }
  
  it "completes checkout successfully" do
    visit product_path(product)
    
    click_button "Add to Cart"
    
    # Turbo stream ทำงาน - ต้องรอ DOM update
    expect(page).to have_text("Cart (1)")
    
    click_link "Checkout"
    
    fill_in "Card number", with: "4242424242424242"
    fill_in "Expiration date", with: "12/25"
    fill_in "CVC", with: "123"
    
    click_button "Place Order"
    
    expect(page).to have_text("Order placed successfully!")
    expect(Order.last.user).to eq(user)
  end
end
```

### 8.2 Testing Turbo Streams

```ruby
# spec/requests/messages_spec.rb
RSpec.describe "Messages", type: :request do
  let(:user) { create(:user) }
  let(:channel) { create(:channel) }
  
  before { sign_in user }
  
  describe "POST /messages" do
    context "with valid params" do
      it "creates message and returns turbo stream" do
        post messages_path,
             params: { message: { body: "Hello!", channel_id: channel.id } },
             headers: { "Accept" => "text/vnd.turbo-stream.html" }
        
        expect(response).to have_http_status(:ok)
        expect(response.content_type).to include("text/vnd.turbo-stream.html")
        
        # ตรวจสอบว่า turbo stream มี correct actions
        expect(response.body).to include("turbo-stream")
        expect(response.body).to include('action="append"')
        expect(response.body).to include("messages")
        expect(response.body).to include("Hello!")
      end
    end
  end
end
```

### 8.3 Testing Background Jobs

```ruby
# spec/jobs/email_job_spec.rb
RSpec.describe UserWelcomeEmailJob, type: :job do
  include ActiveJob::TestHelper
  
  let(:user) { create(:user) }
  
  it "sends welcome email" do
    expect {
      described_class.perform_now(user.id)
    }.to have_enqueued_mail(UserMailer, :welcome).with(user)
  end
  
  it "retries on failure" do
    allow(UserMailer).to receive(:welcome).and_raise(Net::SMTPError)
    
    expect {
      perform_enqueued_jobs { described_class.perform_later(user.id) }
    }.to raise_error(Net::SMTPError)
    
    # ตรวจสอบ retry count
    expect(described_class).to have_been_performed.once
  end
end
```

## 9. Rails และ Modern JavaScript Integration

### 9.1 Importmaps (ไม่ใช้ Node.js)

```ruby
# config/importmap.rb
pin "application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js", preload: true
pin_all_from "app/javascript/controllers", under: "controllers"

# pin external packages
pin "lodash", to: "https://cdn.jsdelivr.net/npm/lodash@4.17.21/lodash.min.js"
pin "chart.js", to: "https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"

# app/javascript/application.js
import "@hotwired/turbo-rails"
import "controllers"

// ใช้ external library
import Chart from "chart.js"
```

### 9.2 Vite สำหรับ JavaScript Bundling

```ruby
# Gemfile
gem "vite_rails"

# vite.config.ts
import { defineConfig } from "vite"
import RubyPlugin from "vite-plugin-ruby"
import react from "@vitejs/plugin-react"

export default defineConfig({
  plugins: [
    RubyPlugin(),
    react()
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
          charts: ["chart.js", "react-chartjs-2"]
        }
      }
    }
  }
})
```

## 10. Rails Security Best Practices

### 10.1 Content Security Policy

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https, :unsafe_inline
  
  # Hotwire ต้องการ unsafe-inline สำหรับ Turbo
  # ใช้ nonce แทนเพื่อความปลอดภัย
  policy.script_src :self, -> { "'nonce-#{request.content_security_policy_nonce}'" }
  
  if Rails.env.development?
    policy.connect_src :self, :https, "http://localhost:3035", "ws://localhost:3035"
  end
  
  # Report violations
  if Rails.env.production?
    policy.report_uri "/csp-violation-reports"
  end
end

Rails.application.config.content_security_policy_nonce_generator = -> (request) {
  SecureRandom.base64(16)
}

Rails.application.config.content_security_policy_nonce_directives = %w[script-src]
```

### 10.2 Rate Limiting ด้วย Rack::Attack

```ruby
# config/initializers/rack_attack.rb
Rack::Attack.cache.store = ActiveSupport::Cache::RedisCacheStore.new(
  url: ENV["REDIS_URL"]
)

# Block suspicious IPs
Rack::Attack.blocklist("block suspicious IPs") do |request|
  BlockedIp.exists?(ip: request.ip)
end

# Rate limit login attempts
Rack::Attack.throttle("login attempts by IP", limit: 5, period: 1.minute) do |request|
  if request.path == "/sessions" && request.post?
    request.ip
  end
end

# Rate limit API
Rack::Attack.throttle("API requests", limit: 100, period: 1.minute) do |request|
  if request.path.start_with?("/api/")
    request.env["HTTP_AUTHORIZATION"]&.split(" ")&.last || request.ip
  end
end

# Custom response
Rack::Attack.throttled_responder = lambda do |env|
  [
    429,
    { "Content-Type" => "application/json" },
    [{ error: "Too many requests", retry_after: 60 }.to_json]
  ]
end

# Notification
ActiveSupport::Notifications.subscribe("rack.attack") do |_name, _start, _finish, _id, payload|
  request = payload[:request]
  Rails.logger.warn "Rate limited: #{request.ip} #{request.path}"
end
```

---

## แบบฝึกหัดบทที่ 90

### แบบฝึกหัดที่ 1: Turbo Stream Real-time Chat
**โจทย์:** สร้าง real-time chat room ด้วย Turbo Streams

**เฉลย:**
```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :user
  belongs_to :room
  
  validates :body, presence: true, length: { maximum: 1000 }
  
  after_create_commit -> { broadcast_append_to room }
  after_destroy_commit -> { broadcast_remove_to room }
  
  scope :recent, -> { order(created_at: :desc).limit(50) }
end

# app/controllers/rooms_controller.rb
class RoomsController < ApplicationController
  before_action :authenticate_user!
  
  def show
    @room = Room.find(params[:id])
    @messages = @room.messages.recent.reverse
    @new_message = Message.new
  end
end

# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  before_action :authenticate_user!
  
  def create
    @room = Room.find(params[:room_id])
    @message = @room.messages.build(
      body: params[:message][:body],
      user: current_user
    )
    
    if @message.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to @room }
      end
    else
      render turbo_stream: turbo_stream.replace(
        "new_message_form",
        partial: "messages/form",
        locals: { message: @message, room: @room }
      )
    end
  end
end

# app/views/rooms/show.html.erb
<%= turbo_stream_from @room %>

<div id="messages" class="messages-container">
  <%= render @messages %>
</div>

<%= turbo_frame_tag "new_message_form" do %>
  <%= render "messages/form", message: @new_message, room: @room %>
<% end %>

# app/views/messages/_message.html.erb
<%= turbo_frame_tag dom_id(message) do %>
  <div class="message <%= 'own' if message.user == current_user %>">
    <strong><%= message.user.name %></strong>
    <p><%= message.body %></p>
    <time><%= message.created_at.strftime("%H:%M") %></time>
  </div>
<% end %>

# app/views/messages/create.turbo_stream.erb
<%= turbo_stream.append "messages", @message %>
<%= turbo_stream.replace "new_message_form" do %>
  <%= render "messages/form", message: Message.new, room: @room %>
<% end %>
<%= turbo_stream.invoke "scrollToBottom", args: ["messages"] %>
```

### แบบฝึกหัดที่ 2: Solid Queue Background Jobs
**โจทย์:** Setup Solid Queue สำหรับ email sending พร้อม retry logic

**เฉลย:**
```ruby
# app/jobs/send_newsletter_job.rb
class SendNewsletterJob < ApplicationJob
  queue_as :newsletters
  
  retry_on Net::SMTPError, wait: :polynomially_longer, attempts: 5
  discard_on ActiveJob::DeserializationError
  
  def perform(newsletter_id, user_ids)
    newsletter = Newsletter.find(newsletter_id)
    users = User.where(id: user_ids)
    
    users.find_each do |user|
      NewsletterMailer.send_newsletter(user, newsletter).deliver_now
      
      # Track delivery
      newsletter.deliveries.create!(
        user: user,
        delivered_at: Time.current
      )
    rescue => e
      Rails.logger.error "Failed to send newsletter to user #{user.id}: #{e.message}"
      newsletter.failures.create!(user: user, error: e.message)
    end
  end
end

# config/solid_queue.yml
default: &default
  workers:
    - queues: [newsletters]
      threads: 2
      polling_interval: 2
    - queues: [default, mailers]
      threads: 5
      polling_interval: 1
  recurring_tasks:
    cleanup_old_jobs:
      class: CleanupOldJobsJob
      schedule: "0 3 * * *"  # 3am daily
```

### แบบฝึกหัดที่ 3: RBS Type Signatures
**โจทย์:** เขียน RBS signatures สำหรับ Order processing service

**เฉลย:**
```rbs
# sig/services/order_service.rbs

class OrderService
  type order_status = :pending | :confirmed | :shipped | :delivered | :cancelled
  
  @order: Order
  @user: User
  
  def initialize: (order: Order, user: User) -> void
  
  def confirm: () -> Order
  def cancel: (?reason: String) -> Order
  def ship: (tracking_number: String) -> Order
  def deliver: () -> Order
  
  def calculate_total: () -> Money
  def apply_discount: (code: String) -> Money
  
  def status: () -> order_status
  def status=: (order_status) -> order_status
  
  def self.create_from_cart: (
    user: User,
    cart: Cart,
    payment_method: String,
    shipping_address: Address
  ) -> Order
end

class Order
  attr_accessor id: Integer
  attr_accessor status: String
  attr_accessor total: Money
  attr_accessor user_id: Integer
  attr_accessor created_at: Time
  attr_accessor updated_at: Time
  
  def user: () -> User
  def items: () -> ActiveRecord::Associations::CollectionProxy[OrderItem]
  def shipping_address: () -> Address
  
  def pending?: () -> bool
  def confirmed?: () -> bool
  def shipped?: () -> bool
end
```

### แบบฝึกหัดที่ 4: Stimulus Controller สำหรับ Form Validation
**โจทย์:** สร้าง Stimulus controller ที่ทำ real-time form validation

**เฉลย:**
```javascript
// app/javascript/controllers/form_validation_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["field", "error", "submit"]
  static values = {
    rules: Object
  }
  
  connect() {
    this.validate()
  }
  
  validate() {
    let isFormValid = true
    
    this.fieldTargets.forEach(field => {
      const fieldName = field.name
      const rules = this.rulesValue[fieldName] || {}
      const errors = this.validateField(field, rules)
      
      this.showErrors(field, errors)
      
      if (errors.length > 0) {
        isFormValid = false
      }
    })
    
    this.submitTarget.disabled = !isFormValid
  }
  
  validateField(field, rules) {
    const errors = []
    const value = field.value
    
    if (rules.required && !value.trim()) {
      errors.push(`${field.dataset.label || field.name} is required`)
    }
    
    if (rules.minLength && value.length < rules.minLength) {
      errors.push(`Minimum ${rules.minLength} characters`)
    }
    
    if (rules.maxLength && value.length > rules.maxLength) {
      errors.push(`Maximum ${rules.maxLength} characters`)
    }
    
    if (rules.email && value && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      errors.push("Invalid email format")
    }
    
    if (rules.pattern && value && !new RegExp(rules.pattern).test(value)) {
      errors.push(rules.patternMessage || "Invalid format")
    }
    
    if (rules.match && value) {
      const matchField = this.element.querySelector(`[name="${rules.match}"]`)
      if (matchField && value !== matchField.value) {
        errors.push(`Must match ${rules.match}`)
      }
    }
    
    return errors
  }
  
  showErrors(field, errors) {
    const errorTarget = this.element.querySelector(`[data-error-for="${field.name}"]`)
    
    if (errors.length > 0) {
      field.classList.add("error")
      if (errorTarget) {
        errorTarget.textContent = errors[0]
        errorTarget.classList.remove("hidden")
      }
    } else {
      field.classList.remove("error")
      if (errorTarget) {
        errorTarget.classList.add("hidden")
      }
    }
  }
}
```

```erb
<!-- app/views/registrations/new.html.erb -->
<%= form_with url: registrations_path,
    data: {
      controller: "form-validation",
      "form-validation-rules-value": {
        email: { required: true, email: true },
        password: { required: true, minLength: 8 },
        password_confirmation: { required: true, match: "password" }
      }.to_json
    } do |f| %>
  
  <div>
    <%= f.label :email %>
    <%= f.email_field :email,
        data: { "form-validation-target": "field",
                label: "Email" },
        "data-action": "blur->form-validation#validate input->form-validation#validate" %>
    <span data-error-for="email" class="hidden error-message"></span>
  </div>
  
  <div>
    <%= f.label :password %>
    <%= f.password_field :password,
        data: { "form-validation-target": "field",
                label: "Password" },
        "data-action": "blur->form-validation#validate" %>
    <span data-error-for="password" class="hidden error-message"></span>
  </div>
  
  <%= f.submit "Register",
      data: { "form-validation-target": "submit" },
      disabled: true %>
<% end %>
```

### แบบฝึกหัดที่ 5: Roda Microservice
**โจทย์:** สร้าง Roda API microservice สำหรับ notification service

**เฉลย:**
```ruby
# notification_service/app.rb
require "roda"
require "sequel"
require "sidekiq"

DB = Sequel.connect(ENV["DATABASE_URL"])

class NotificationService < Roda
  plugin :json
  plugin :json_parser
  plugin :halt
  plugin :all_verbs
  plugin :hooks
  
  before do
    # Authentication
    token = request.env["HTTP_X_API_TOKEN"]
    r.halt(401, { error: "Unauthorized" }) unless valid_token?(token)
  end
  
  route do |r|
    r.on "notifications" do
      r.get do
        # GET /notifications?user_id=123
        user_id = r.params["user_id"]
        r.halt(422, { error: "user_id required" }) unless user_id
        
        notifications = DB[:notifications]
          .where(user_id: user_id.to_i)
          .order(Sequel.desc(:created_at))
          .limit(50)
          .all
        
        { notifications: notifications, count: notifications.count }
      end
      
      r.post do
        # POST /notifications
        data = r.params
        
        r.halt(422, { error: "Missing required fields" }) unless data["user_id"] && data["message"]
        
        notification_id = DB[:notifications].insert(
          user_id: data["user_id"].to_i,
          message: data["message"],
          type: data["type"] || "info",
          created_at: Time.now,
          read: false
        )
        
        # Dispatch real-time update via Sidekiq
        PushNotificationWorker.perform_async(
          notification_id,
          data["user_id"].to_i
        )
        
        response.status = 201
        DB[:notifications].where(id: notification_id).first
      end
      
      r.on Integer do |notification_id|
        notification = DB[:notifications].where(id: notification_id).first
        r.halt(404, { error: "Not found" }) unless notification
        
        r.put do
          DB[:notifications].where(id: notification_id).update(
            read: true,
            read_at: Time.now
          )
          
          DB[:notifications].where(id: notification_id).first
        end
        
        r.delete do
          DB[:notifications].where(id: notification_id).delete
          response.status = 204
        end
      end
    end
    
    r.on "users" do
      r.on Integer do |user_id|
        r.get "unread_count" do
          count = DB[:notifications]
            .where(user_id: user_id, read: false)
            .count
          
          { user_id: user_id, unread_count: count }
        end
        
        r.put "read_all" do
          DB[:notifications]
            .where(user_id: user_id, read: false)
            .update(read: true, read_at: Time.now)
          
          { success: true }
        end
      end
    end
  end
  
  private
  
  def valid_token?(token)
    ApiToken.exists?(token: token, active: true)
  end
end
```

### แบบฝึกหัดที่ 6: Hotwire Infinite Scroll
**โจทย์:** Implement infinite scroll ด้วย Turbo Frames และ Stimulus

**เฉลย:**
```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.published.order(created_at: :desc).page(params[:page]).per(10)
  end
end

# app/views/posts/index.html.erb
<div id="posts">
  <%= render @posts %>
</div>

<%= turbo_frame_tag "pagination",
    data: { controller: "infinite-scroll",
            "infinite-scroll-target": "sentinel",
            loading: :lazy,
            src: posts_path(page: 2) } %>
```

```javascript
// app/javascript/controllers/infinite_scroll_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["sentinel"]
  
  connect() {
    this.observer = new IntersectionObserver(
      entries => this.handleIntersection(entries),
      { threshold: 0.1 }
    )
    
    this.observer.observe(this.element)
  }
  
  disconnect() {
    this.observer.disconnect()
  }
  
  handleIntersection(entries) {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        this.element.click()  // Trigger Turbo Frame lazy load
      }
    })
  }
}
```

```ruby
# app/views/posts/index.turbo_stream.erb (สำหรับ paginated response)
<%= turbo_stream.append "posts" do %>
  <%= render @posts %>
<% end %>

<% if @posts.next_page %>
  <%= turbo_stream.replace "pagination" do %>
    <%= turbo_frame_tag "pagination",
        src: posts_path(page: @posts.next_page),
        loading: :lazy,
        data: { controller: "infinite-scroll" } %>
  <% end %>
<% else %>
  <%= turbo_stream.remove "pagination" %>
<% end %>
```

### แบบฝึกหัดที่ 7: Kamal Deployment Pipeline
**โจทย์:** Setup complete CI/CD pipeline ด้วย GitHub Actions + Kamal

**เฉลย:**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      
      - name: Setup database
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/test
        run: bundle exec rails db:create db:schema:load
      
      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/test
        run: bundle exec rspec
      
      - name: Run security checks
        run: |
          bundle exec brakeman -q --no-pager
          bundle audit check --update
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      
      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
      
      - name: Deploy with Kamal
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
          DATABASE_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
        run: kamal deploy
```

### แบบฝึกหัดที่ 8: Sorbet Type-safe Service
**โจทย์:** เขียน type-safe payment service ด้วย Sorbet

**เฉลย:**
```ruby
# typed: strict
# app/services/stripe_payment_service.rb

class StripePaymentService
  extend T::Sig
  
  class PaymentError < StandardError; end
  class CardDeclinedError < PaymentError; end
  class InsufficientFundsError < PaymentError; end
  
  sig { params(user: User).void }
  def initialize(user)
    @user = T.let(user, User)
  end
  
  sig {
    params(
      amount_cents: Integer,
      currency: String,
      description: T.nilable(String)
    ).returns(Stripe::Charge)
  }
  def charge!(amount_cents, currency: "usd", description: nil)
    Stripe::Charge.create({
      amount: amount_cents,
      currency: currency,
      customer: ensure_customer.id,
      description: description
    })
  rescue Stripe::CardError => e
    case e.code
    when "insufficient_funds"
      raise InsufficientFundsError, e.message
    when "card_declined"
      raise CardDeclinedError, e.message
    else
      raise PaymentError, e.message
    end
  end
  
  sig { returns(Stripe::Customer) }
  def ensure_customer
    if @user.stripe_customer_id.present?
      Stripe::Customer.retrieve(@user.stripe_customer_id)
    else
      customer = Stripe::Customer.create(
        email: @user.email,
        name: @user.full_name
      )
      @user.update!(stripe_customer_id: customer.id)
      customer
    end
  end
  
  sig {
    params(
      price_id: String,
      trial_days: T.nilable(Integer)
    ).returns(Stripe::Subscription)
  }
  def subscribe!(price_id, trial_days: nil)
    params = T.let({
      customer: ensure_customer.id,
      items: [{ price: price_id }]
    }, T::Hash[Symbol, T.untyped])
    
    params[:trial_period_days] = trial_days if trial_days
    
    Stripe::Subscription.create(params)
  end
end
```

### แบบฝึกหัดที่ 9: Hanami 2 Slice
**โจทย์:** สร้าง Admin slice ใน Hanami 2 app

**เฉลย:**
```ruby
# slices/admin/config/routes.rb
module Admin
  class Routes < Hanami::Routes
    root to: "dashboard#index"
    
    resources :users, only: [:index, :show, :edit, :update, :destroy]
    resources :orders, only: [:index, :show] do
      member do
        put :refund
        put :ship
      end
    end
    
    get "analytics", to: "analytics#index"
  end
end

# slices/admin/actions/dashboard/index.rb
module Admin
  module Actions
    module Dashboard
      class Index < Admin::Action
        include Deps[
          "repositories.users",
          "repositories.orders",
          "services.analytics"
        ]
        
        def handle(request, response)
          response[:stats] = {
            total_users: users.count,
            new_users_today: users.created_today,
            total_revenue: orders.total_revenue,
            pending_orders: orders.pending.count
          }
          
          response[:recent_orders] = orders.recent(limit: 10)
          response[:top_products] = analytics.top_products(limit: 5)
        end
      end
    end
  end
end
```

### แบบฝึกหัดที่ 10: Custom Turbo Stream Action
**โจทย์:** สร้าง custom Turbo Stream action สำหรับ modal management

**เฉลย:**
```javascript
// app/javascript/turbo_modal_actions.js
import { StreamActions } from "@hotwired/turbo"

// เปิด modal
StreamActions.open_modal = function() {
  const modalId = this.getAttribute("modal")
  const modal = document.getElementById(modalId)
  
  if (modal) {
    // ใส่ content ถ้ามี
    if (this.templateContent) {
      const contentArea = modal.querySelector(".modal-content")
      if (contentArea) {
        contentArea.replaceChildren(this.templateContent)
      }
    }
    
    modal.classList.add("open")
    document.body.classList.add("modal-open")
    
    // Focus management
    const firstFocusable = modal.querySelector(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    )
    firstFocusable?.focus()
  }
}

// ปิด modal
StreamActions.close_modal = function() {
  const modal = document.querySelector(".modal.open")
  
  if (modal) {
    modal.classList.remove("open")
    document.body.classList.remove("modal-open")
  }
}
```

```ruby
# ใช้งานใน controller
def edit
  @product = Product.find(params[:id])
  
  respond_to do |format|
    format.turbo_stream do
      render turbo_stream: [
        # เปิด modal พร้อม content
        <<~HTML.html_safe
          <turbo-stream action="open_modal" modal="edit-product-modal">
            <template>
              #{render_to_string(partial: "products/edit_form", locals: { product: @product })}
            </template>
          </turbo-stream>
        HTML
      ]
    end
  end
end
```

### แบบฝึกหัดที่ 11: Rails API + Hotwire Hybrid
**โจทย์:** สร้าง app ที่รองรับทั้ง JSON API (สำหรับ mobile) และ Hotwire (สำหรับ web)

**เฉลย:**
```ruby
# app/controllers/api/base_controller.rb
module Api
  class BaseController < ActionController::API
    before_action :authenticate_api_token!
    
    private
    
    def authenticate_api_token!
      token = request.headers["Authorization"]&.split(" ")&.last
      @current_user = User.find_by_api_token(token)
      
      render json: { error: "Unauthorized" }, status: :unauthorized unless @current_user
    end
  end
end

# app/controllers/web/base_controller.rb
module Web
  class BaseController < ApplicationController
    before_action :authenticate_user!
    layout "web"
  end
end

# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :posts, only: [:index, :show, :create, :update, :destroy]
    end
  end
  
  namespace :web do
    resources :posts do
      member { post :like }
    end
  end
end

# app/controllers/api/v1/posts_controller.rb
module Api
  module V1
    class PostsController < Api::BaseController
      def index
        @posts = Post.published.page(params[:page]).per(20)
        render json: PostSerializer.new(@posts).as_json
      end
      
      def create
        @post = @current_user.posts.build(post_params)
        
        if @post.save
          render json: PostSerializer.new(@post), status: :created
        else
          render json: { errors: @post.errors }, status: :unprocessable_entity
        end
      end
    end
  end
end

# app/controllers/web/posts_controller.rb
module Web
  class PostsController < Web::BaseController
    def index
      @posts = Post.published.page(params[:page]).per(20)
    end
    
    def create
      @post = current_user.posts.build(post_params)
      
      if @post.save
        respond_to do |format|
          format.turbo_stream do
            render turbo_stream: [
              turbo_stream.prepend("posts-list", @post),
              turbo_stream.replace("new-post-form", partial: "posts/form", locals: { post: Post.new })
            ]
          end
          format.html { redirect_to web_posts_path }
        end
      else
        respond_to do |format|
          format.turbo_stream do
            render turbo_stream: turbo_stream.replace(
              "new-post-form",
              partial: "posts/form",
              locals: { post: @post }
            )
          end
          format.html { render :new, status: :unprocessable_entity }
        end
      end
    end
  end
end
```

### แบบฝึกหัดที่ 12: RuboCop Custom Cop
**โจทย์:** สร้าง custom RuboCop cop สำหรับ project-specific conventions

**เฉลย:**
```ruby
# lib/rubocop/cop/rails/no_raw_sql.rb
module RuboCop
  module Cop
    module Rails
      # ตรวจจับการใช้ raw SQL string ใน ActiveRecord queries
      # 
      # @example
      #   # bad
      #   User.where("email = '#{email}'")
      #   
      #   # good
      #   User.where(email: email)
      #   User.where("email = ?", email)
      #
      class NoRawSql < Base
        MSG = "Avoid raw SQL with string interpolation. Use parameterized queries instead."
        
        RESTRICT_ON_SEND = %i[where having order group select].freeze
        
        def on_send(node)
          return unless node.arguments.any? { |arg| arg.dstr_type? && arg.to_s.include?("#{") }
          
          add_offense(node)
        end
      end
    end
  end
end

# .rubocop.yml
require:
  - ./lib/rubocop/cop/rails/no_raw_sql

Rails/NoRawSql:
  Enabled: true
  Severity: error
```

### แบบฝึกหัดที่ 13: Stimulus + Turbo File Upload
**โจทย์:** สร้าง drag-and-drop file upload ด้วย Stimulus + ActiveStorage

**เฉลย:**
```javascript
// app/javascript/controllers/file_upload_controller.js
import { Controller } from "@hotwired/stimulus"
import { DirectUpload } from "@rails/activestorage"

export default class extends Controller {
  static targets = ["dropzone", "preview", "progress", "input"]
  static values = { url: String, multiple: Boolean }
  
  connect() {
    this.setupDragAndDrop()
  }
  
  setupDragAndDrop() {
    this.dropzoneTarget.addEventListener("dragover", e => {
      e.preventDefault()
      this.dropzoneTarget.classList.add("dragover")
    })
    
    this.dropzoneTarget.addEventListener("dragleave", () => {
      this.dropzoneTarget.classList.remove("dragover")
    })
    
    this.dropzoneTarget.addEventListener("drop", e => {
      e.preventDefault()
      this.dropzoneTarget.classList.remove("dragover")
      
      const files = Array.from(e.dataTransfer.files)
      this.uploadFiles(files)
    })
  }
  
  selectFiles() {
    this.inputTarget.click()
  }
  
  filesSelected(event) {
    const files = Array.from(event.target.files)
    this.uploadFiles(files)
  }
  
  async uploadFiles(files) {
    for (const file of files) {
      await this.uploadFile(file)
    }
  }
  
  uploadFile(file) {
    return new Promise((resolve, reject) => {
      const upload = new DirectUpload(
        file,
        this.urlValue,
        {
          directUploadWillStoreFileWithXHR: xhr => {
            xhr.upload.addEventListener("progress", event => {
              const progress = Math.round(event.loaded / event.total * 100)
              this.updateProgress(file.name, progress)
            })
          }
        }
      )
      
      // Show preview for images
      if (file.type.startsWith("image/")) {
        const reader = new FileReader()
        reader.onload = e => this.addPreview(file.name, e.target.result)
        reader.readAsDataURL(file)
      }
      
      upload.create((error, blob) => {
        if (error) {
          this.showError(file.name, error.message)
          reject(error)
        } else {
          this.addHiddenField(blob.signed_id)
          this.markComplete(file.name)
          resolve(blob)
        }
      })
    })
  }
  
  addHiddenField(signedId) {
    const input = document.createElement("input")
    input.type = "hidden"
    input.name = this.inputTarget.name
    input.value = signedId
    this.element.appendChild(input)
  }
  
  updateProgress(filename, percent) {
    let progressEl = this.progressTarget.querySelector(`[data-filename="${filename}"]`)
    
    if (!progressEl) {
      progressEl = document.createElement("div")
      progressEl.dataset.filename = filename
      progressEl.innerHTML = `
        <span>${filename}</span>
        <progress max="100" value="${percent}"></progress>
        <span class="percent">${percent}%</span>
      `
      this.progressTarget.appendChild(progressEl)
    }
    
    progressEl.querySelector("progress").value = percent
    progressEl.querySelector(".percent").textContent = `${percent}%`
  }
  
  addPreview(filename, dataUrl) {
    const img = document.createElement("img")
    img.src = dataUrl
    img.alt = filename
    img.className = "preview-image"
    this.previewTarget.appendChild(img)
  }
  
  markComplete(filename) {
    const el = this.progressTarget.querySelector(`[data-filename="${filename}"]`)
    el?.classList.add("complete")
  }
  
  showError(filename, message) {
    const el = document.createElement("div")
    el.className = "upload-error"
    el.textContent = `${filename}: ${message}`
    this.element.appendChild(el)
  }
}
```

### แบบฝึกหัดที่ 14: Hanami Repository Pattern
**โจทย์:** Implement Repository Pattern ใน Hanami 2

**เฉลย:**
```ruby
# app/repositories/user_repository.rb
module MyApp
  module Repositories
    class UserRepository
      include Deps["persistence.rom"]
      
      def find(id)
        users.by_pk(id).one!
      rescue ROM::TupleCountMismatchError
        raise UserNotFoundError, "User with id #{id} not found"
      end
      
      def find_by_email(email)
        users.where(email: email).one
      end
      
      def create(attributes)
        users.changeset(:create, attributes).commit
      end
      
      def update(id, attributes)
        users.by_pk(id).changeset(:update, attributes).commit
      end
      
      def delete(id)
        users.by_pk(id).changeset(:delete).commit
      end
      
      def search(query:, page: 1, per_page: 20)
        users
          .where { name.ilike("%#{query}%") | email.ilike("%#{query}%") }
          .page(page)
          .per_page(per_page)
          .to_a
      end
      
      def count
        users.count
      end
      
      def created_today
        users.where(created_at: Date.today.beginning_of_day..Date.today.end_of_day).count
      end
      
      private
      
      def users
        rom.relations[:users]
      end
    end
  end
end
```

### แบบฝึกหัดที่ 15: Complete Hotwire Todo App
**โจทย์:** สร้าง complete todo application ด้วย Hotwire

**เฉลย:**
```ruby
# app/models/todo.rb
class Todo < ApplicationRecord
  belongs_to :user
  
  validates :title, presence: true
  
  scope :incomplete, -> { where(completed: false) }
  scope :completed, -> { where(completed: true) }
  
  after_create_commit { broadcast_prepend_to [user, "todos"] }
  after_update_commit { broadcast_replace_to [user, "todos"] }
  after_destroy_commit { broadcast_remove_to [user, "todos"] }
end

# app/controllers/todos_controller.rb
class TodosController < ApplicationController
  before_action :authenticate_user!
  before_action :set_todo, only: [:show, :edit, :update, :destroy, :toggle]
  
  def index
    @todos = current_user.todos.order(created_at: :desc)
    @new_todo = Todo.new
    @stats = {
      total: @todos.count,
      completed: @todos.completed.count,
      incomplete: @todos.incomplete.count
    }
  end
  
  def create
    @todo = current_user.todos.build(todo_params)
    
    if @todo.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to todos_path }
      end
    else
      respond_to do |format|
        format.turbo_stream do
          render turbo_stream: turbo_stream.replace(
            "new_todo_form",
            partial: "todos/form",
            locals: { todo: @todo }
          )
        end
        format.html { render :index }
      end
    end
  end
  
  def toggle
    @todo.update!(completed: !@todo.completed)
    
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to todos_path }
    end
  end
  
  def destroy
    @todo.destroy
    
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to todos_path }
    end
  end
  
  private
  
  def set_todo
    @todo = current_user.todos.find(params[:id])
  end
  
  def todo_params
    params.require(:todo).permit(:title, :description, :due_date, :priority)
  end
end

# app/views/todos/index.html.erb
<%= turbo_stream_from [current_user, "todos"] %>

<div class="todos-container">
  <div class="stats" id="todo-stats">
    <%= render "todos/stats", stats: @stats %>
  </div>
  
  <%= turbo_frame_tag "new_todo_form" do %>
    <%= render "todos/form", todo: @new_todo %>
  <% end %>
  
  <div id="todos">
    <%= render @todos %>
  </div>
</div>
```

---

## สรุปบทที่ 90

ในบทนี้เราได้เรียนรู้:

1. **Rails 8 Features** - Solid Queue, Solid Cache, Solid Cable, Kamal 2, Propshaft
2. **Hotwire ขั้นสูง** - Custom Turbo Actions, Lazy Loading, Broadcasts
3. **Stimulus Controllers** - Form validation, infinite scroll, file upload
4. **Alternative Frameworks** - Hanami 2.x และ Roda
5. **Ruby Tooling** - RuboCop 1.x, RBS, Steep, Sorbet, TypeProf
6. **Security** - CSP, Rate limiting ด้วย Rack::Attack
7. **Testing** - System tests, Turbo stream testing, Job testing

### สิ่งที่ควรศึกษาเพิ่มเติม

- Rails 8 changelog อย่างละเอียด
- Turbo 8 features (morphing, page refresh)
- Stimulus 3 best practices
- Hanami 2 full documentation
- Sorbet gradual typing strategy
