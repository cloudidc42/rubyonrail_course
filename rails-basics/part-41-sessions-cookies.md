# Part 41: Sessions and Cookies ใน Rails

## ขั้นตอนที่ 906-925: การจัดการ Sessions และ Cookies

---

## ขั้นตอนที่ 906: Sessions คืออะไร?

Session คือกลไกที่ช่วยให้เว็บแอปพลิเคชันสามารถจดจำข้อมูลของผู้ใช้ระหว่าง HTTP requests ต่างๆ ได้ เนื่องจาก HTTP เป็น stateless protocol (ไม่จดจำสถานะ) การใช้ session จึงช่วยให้เราสามารถติดตามสถานะของผู้ใช้ได้

```ruby
# ตัวอย่างการใช้งาน session ใน controller
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])
    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to root_path, notice: "เข้าสู่ระบบสำเร็จ"
    else
      flash[:alert] = "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
      render :new
    end
  end

  def destroy
    session[:user_id] = nil
    redirect_to root_path, notice: "ออกจากระบบแล้ว"
  end
end
```

## ขั้นตอนที่ 907: Session Hash ใน Rails

Rails ให้ `session` hash ที่สามารถเก็บข้อมูลใดๆ ก็ได้ที่สามารถ serialize ได้

```ruby
class ApplicationController < ActionController::Base
  # อ่านค่า session
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  # ตรวจสอบว่า login หรือยัง
  def logged_in?
    current_user.present?
  end

  # กำหนดให้ต้อง login ก่อน
  def require_login
    unless logged_in?
      flash[:alert] = "กรุณาเข้าสู่ระบบก่อน"
      redirect_to login_path
    end
  end

  helper_method :current_user, :logged_in?
end
```

```ruby
# การใช้งาน session hash ต่างๆ
class ExamplesController < ApplicationController
  def store_data
    # เก็บค่า string
    session[:username] = "john_doe"
    
    # เก็บค่า integer
    session[:user_id] = 42
    
    # เก็บค่า array
    session[:cart_items] = [1, 2, 3, 4]
    
    # เก็บค่า hash
    session[:preferences] = { theme: "dark", language: "th" }
    
    # เก็บเวลา
    session[:logged_in_at] = Time.current
  end

  def read_data
    username = session[:username]
    user_id = session[:user_id]
    cart = session[:cart_items]
    prefs = session[:preferences]
    
    render json: {
      username: username,
      user_id: user_id,
      cart_count: cart&.length,
      theme: prefs&.dig(:theme)
    }
  end

  def delete_data
    # ลบ key เดียว
    session.delete(:username)
    
    # ล้าง session ทั้งหมด
    reset_session
    
    redirect_to root_path
  end
end
```

## ขั้นตอนที่ 908: Cookie Store (Default Session Store)

Rails ใช้ Cookie Store เป็น session store เริ่มต้น ซึ่งเก็บข้อมูล session ไว้ในคุกกี้ที่เข้ารหัสแล้ว

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store, 
  key: '_my_app_session',
  secure: Rails.env.production?,  # ใช้ HTTPS เท่านั้นใน production
  httponly: true,                  # ป้องกัน JavaScript เข้าถึง
  same_site: :lax,                 # ป้องกัน CSRF
  expire_after: 2.weeks            # หมดอายุใน 2 สัปดาห์
```

```ruby
# config/secrets.yml หรือ credentials
# ต้องมี secret_key_base สำหรับเข้ารหัส
development:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>

production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
```

```bash
# สร้าง secret key
rails secret

# หรือใช้ credentials
rails credentials:edit
```

```yaml
# config/credentials.yml.enc (เข้ารหัสแล้ว)
secret_key_base: your_long_secret_key_here_abc123def456...
```

**ข้อดีของ Cookie Store:**
- ไม่ต้องใช้ database
- เร็วมาก
- ไม่มี server-side storage

**ข้อเสียของ Cookie Store:**
- ขนาดจำกัด 4KB
- ข้อมูลส่งไปกับทุก request
- ไม่สามารถ revoke session ได้ทันที

## ขั้นตอนที่ 909: Database Session Store (Active Record Store)

สำหรับ application ที่ต้องการควบคุม session มากขึ้น ควรใช้ Database Session Store

```bash
# เพิ่ม gem ใน Gemfile
# gem 'activerecord-session_store'
bundle install

# สร้าง migration
rails generate active_record:session_migration
rails db:migrate
```

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :active_record_store,
  key: '_my_app_session',
  secure: Rails.env.production?,
  httponly: true,
  expire_after: 30.days
```

```ruby
# การ migrate ที่สร้างขึ้น
class CreateSessions < ActiveRecord::Migration[7.0]
  def change
    create_table :sessions do |t|
      t.string :session_id, null: false
      t.text :data
      t.timestamps null: false
    end

    add_index :sessions, :session_id, unique: true
    add_index :sessions, :updated_at
  end
end
```

```ruby
# ล้าง session เก่าๆ ออกด้วย rake task
# lib/tasks/sessions.rake
namespace :sessions do
  desc "ล้าง session ที่หมดอายุแล้ว"
  task cleanup: :environment do
    ActiveRecord::SessionStore::Session
      .where("updated_at < ?", 30.days.ago)
      .delete_all
    puts "ล้าง sessions เก่าแล้ว"
  end
end
```

## ขั้นตอนที่ 910: Redis Session Store

Redis เป็นตัวเลือกที่นิยมมากสำหรับ session store ใน production

```ruby
# Gemfile
gem 'redis'
gem 'redis-session-store'
# หรือ
gem 'redis-actionpack'
```

```ruby
# config/initializers/session_store.rb
if Rails.env.production?
  Rails.application.config.session_store :redis_store,
    servers: ["redis://localhost:6379/0/session"],
    expire_after: 90.minutes,
    key: "_app_session",
    threadsafe: true,
    secure: true
else
  Rails.application.config.session_store :cookie_store,
    key: "_app_session"
end
```

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV['REDIS_URL'],
  connect_timeout: 30,
  read_timeout: 0.2,
  write_timeout: 0.2,
  reconnect_attempts: 1
}
```

## ขั้นตอนที่ 911: การตั้งค่า Session

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # กำหนด session store
    config.session_store :cookie_store, 
      key: '_myapp_session'
    
    # กำหนด middleware
    config.middleware.use ActionDispatch::Session::CookieStore,
      key: '_myapp_session',
      secret: Rails.application.credentials.secret_key_base
  end
end
```

```ruby
# การใช้ session ใน before_action
class ApplicationController < ActionController::Base
  before_action :set_locale_from_session

  private

  def set_locale_from_session
    I18n.locale = session[:locale] || I18n.default_locale
  end
end

class PreferencesController < ApplicationController
  def update_locale
    locale = params[:locale]
    if I18n.available_locales.include?(locale.to_sym)
      session[:locale] = locale
      flash[:notice] = "เปลี่ยนภาษาเรียบร้อย"
    else
      flash[:alert] = "ภาษาที่เลือกไม่รองรับ"
    end
    redirect_back fallback_location: root_path
  end
end
```

## ขั้นตอนที่ 912: Shopping Cart ด้วย Session

ตัวอย่างการสร้างตะกร้าสินค้าด้วย session

```ruby
# app/models/cart.rb
class Cart
  include ActiveModel::Model
  
  attr_reader :items
  
  def initialize(session)
    @session = session
    @items = session[:cart] || {}
  end
  
  def add_item(product_id, quantity = 1)
    product_id = product_id.to_s
    @items[product_id] ||= 0
    @items[product_id] += quantity
    save!
  end
  
  def remove_item(product_id)
    @items.delete(product_id.to_s)
    save!
  end
  
  def update_quantity(product_id, quantity)
    product_id = product_id.to_s
    if quantity <= 0
      remove_item(product_id)
    else
      @items[product_id] = quantity
      save!
    end
  end
  
  def total_items
    @items.values.sum
  end
  
  def products
    Product.where(id: @items.keys)
  end
  
  def total_price
    products.sum { |p| p.price * @items[p.id.to_s] }
  end
  
  def empty?
    @items.empty?
  end
  
  def clear
    @items = {}
    save!
  end
  
  private
  
  def save!
    @session[:cart] = @items
  end
end
```

```ruby
# app/controllers/carts_controller.rb
class CartsController < ApplicationController
  before_action :load_cart
  
  def show
    @products = @cart.products
  end
  
  def add
    product = Product.find(params[:product_id])
    @cart.add_item(product.id, params[:quantity]&.to_i || 1)
    flash[:notice] = "เพิ่ม #{product.name} ลงตะกร้าแล้ว"
    redirect_to cart_path
  end
  
  def remove
    @cart.remove_item(params[:product_id])
    redirect_to cart_path, notice: "ลบสินค้าออกแล้ว"
  end
  
  def update
    params[:quantities].each do |product_id, quantity|
      @cart.update_quantity(product_id, quantity.to_i)
    end
    redirect_to cart_path, notice: "อัพเดทตะกร้าแล้ว"
  end
  
  def clear
    @cart.clear
    redirect_to root_path, notice: "ล้างตะกร้าแล้ว"
  end
  
  private
  
  def load_cart
    @cart = Cart.new(session)
  end
end
```

```erb
<%# app/views/carts/show.html.erb %>
<h1>ตะกร้าสินค้า</h1>

<% if @cart.empty? %>
  <p>ตะกร้าว่างเปล่า</p>
  <%= link_to "ช้อปปิ้งต่อ", products_path, class: "btn btn-primary" %>
<% else %>
  <%= form_with url: cart_path, method: :patch do |f| %>
    <table class="table">
      <thead>
        <tr>
          <th>สินค้า</th>
          <th>ราคา</th>
          <th>จำนวน</th>
          <th>รวม</th>
          <th>ลบ</th>
        </tr>
      </thead>
      <tbody>
        <% @products.each do |product| %>
          <% quantity = @cart.items[product.id.to_s] %>
          <tr>
            <td><%= product.name %></td>
            <td><%= number_to_currency(product.price, unit: "฿") %></td>
            <td>
              <%= number_field_tag "quantities[#{product.id}]", quantity, 
                  min: 1, max: 99, class: "form-control" %>
            </td>
            <td><%= number_to_currency(product.price * quantity, unit: "฿") %></td>
            <td>
              <%= link_to "ลบ", remove_cart_path(product_id: product.id), 
                  method: :delete, class: "btn btn-danger btn-sm" %>
            </td>
          </tr>
        <% end %>
      </tbody>
      <tfoot>
        <tr>
          <td colspan="3"><strong>รวมทั้งหมด</strong></td>
          <td><strong><%= number_to_currency(@cart.total_price, unit: "฿") %></strong></td>
          <td></td>
        </tr>
      </tfoot>
    </table>
    <%= f.submit "อัพเดทตะกร้า", class: "btn btn-secondary" %>
  <% end %>
  
  <%= link_to "ชำระเงิน", checkout_path, class: "btn btn-success" %>
  <%= link_to "ล้างตะกร้า", clear_cart_path, method: :delete, 
      class: "btn btn-outline-danger",
      data: { confirm: "คุณแน่ใจหรือไม่?" } %>
<% end %>
```

## ขั้นตอนที่ 913: Cookies ใน Rails

Cookies เป็นข้อมูลขนาดเล็กที่เก็บไว้ใน browser ของผู้ใช้

```ruby
class CookiesController < ApplicationController
  # การตั้งค่า cookie ธรรมดา
  def set_simple_cookie
    cookies[:user_name] = "สมชาย"
    cookies[:last_visit] = Time.current.to_s
    redirect_to root_path
  end

  # การตั้งค่า cookie พร้อม options
  def set_cookie_with_options
    cookies[:user_preference] = {
      value: "dark_theme",
      expires: 1.year.from_now,      # หมดอายุ
      domain: ".myapp.com",           # ใช้ได้กับทุก subdomain
      secure: true,                   # HTTPS เท่านั้น
      httponly: true,                 # ป้องกัน JavaScript
      same_site: :strict              # ป้องกัน CSRF
    }
    redirect_to root_path
  end

  # การอ่าน cookie
  def read_cookie
    user_name = cookies[:user_name]
    preference = cookies[:user_preference]
    
    render plain: "ชื่อ: #{user_name}, การตั้งค่า: #{preference}"
  end

  # การลบ cookie
  def delete_cookie
    cookies.delete(:user_name)
    cookies.delete(:user_preference)
    redirect_to root_path
  end
end
```

## ขั้นตอนที่ 914: Signed Cookies (คุกกี้ที่มีลายเซ็น)

Signed cookies ป้องกันการแก้ไขโดยผู้ใช้ แต่ยังอ่านได้

```ruby
class SignedCookiesController < ApplicationController
  def set_signed_cookie
    # Signed cookie - ผู้ใช้ไม่สามารถแก้ไขได้ แต่อ่านค่าได้
    cookies.signed[:user_id] = {
      value: current_user.id,
      expires: 30.days.from_now
    }
    
    # อ่าน signed cookie
    user_id = cookies.signed[:user_id]  # ได้ค่าจริง
    raw_value = cookies[:user_id]        # ได้ค่า signed string
    
    render json: { user_id: user_id, raw: raw_value }
  end

  def verify_signed_cookie
    user_id = cookies.signed[:user_id]
    
    if user_id
      user = User.find_by(id: user_id)
      render json: { valid: user.present?, user: user&.name }
    else
      render json: { valid: false, error: "Cookie ไม่ถูกต้องหรือหมดอายุ" }
    end
  end
end
```

## ขั้นตอนที่ 915: Encrypted Cookies (คุกกี้ที่เข้ารหัส)

Encrypted cookies ทั้งเข้ารหัสและมีลายเซ็น ผู้ใช้ไม่สามารถอ่านหรือแก้ไขได้

```ruby
class EncryptedCookiesController < ApplicationController
  def store_sensitive_data
    # Encrypted cookie - ทั้งเข้ารหัสและมีลายเซ็น
    cookies.encrypted[:sensitive_data] = {
      value: { 
        user_id: current_user.id,
        role: current_user.role,
        permissions: current_user.permissions
      }.to_json,
      expires: 7.days.from_now,
      secure: Rails.env.production?,
      httponly: true
    }
    
    redirect_to root_path
  end

  def read_sensitive_data
    # อ่านค่าที่เข้ารหัส
    data = cookies.encrypted[:sensitive_data]
    
    if data
      parsed = JSON.parse(data)
      render json: parsed
    else
      render json: { error: "ไม่พบข้อมูล" }, status: :not_found
    end
  end

  def remember_user
    if params[:remember_me] == "1"
      cookies.encrypted.permanent[:remember_token] = {
        value: generate_remember_token,
        httponly: true,
        secure: Rails.env.production?
      }
    end
  end
  
  private
  
  def generate_remember_token
    SecureRandom.urlsafe_base64(32)
  end
end
```

## ขั้นตอนที่ 916: Permanent Cookies

```ruby
class PermanentCookiesController < ApplicationController
  def set_permanent_cookie
    # Permanent cookie - หมดอายุใน 20 ปี
    cookies.permanent[:user_preference] = "dark_theme"
    
    # Permanent signed cookie
    cookies.permanent.signed[:user_id] = current_user.id
    
    # Permanent encrypted cookie
    cookies.permanent.encrypted[:session_token] = SecureRandom.hex(32)
    
    redirect_to root_path
  end

  def clear_all_cookies
    # ลบ cookies ทั้งหมด
    cookies.each do |name, _value|
      cookies.delete(name)
    end
    redirect_to root_path
  end
end
```

## ขั้นตอนที่ 917: Flash Messages

Flash messages เป็น session data พิเศษที่อยู่ได้แค่ request เดียว

```ruby
class FlashController < ApplicationController
  def create
    @post = Post.new(post_params)
    
    if @post.save
      # Flash message จะหายหลังจาก redirect
      flash[:notice] = "บันทึกบทความเรียบร้อยแล้ว"
      redirect_to @post
    else
      # flash.now ใช้สำหรับ render (ไม่ใช่ redirect)
      flash.now[:alert] = "เกิดข้อผิดพลาดในการบันทึก"
      render :new, status: :unprocessable_entity
    end
  end

  def update
    @post = Post.find(params[:id])
    
    if @post.update(post_params)
      flash[:success] = "อัพเดทเรียบร้อยแล้ว"
      redirect_to @post
    else
      flash.now[:error] = "ไม่สามารถอัพเดทได้"
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @post = Post.find(params[:id])
    @post.destroy
    
    # สามารถใช้ shorthand ได้
    redirect_to posts_path, notice: "ลบบทความแล้ว"
    # เทียบเท่ากับ:
    # flash[:notice] = "ลบบทความแล้ว"
    # redirect_to posts_path
  end
  
  private
  
  def post_params
    params.require(:post).permit(:title, :content)
  end
end
```

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <title>My App</title>
</head>
<body>
  <%# แสดง flash messages %>
  <% flash.each do |type, message| %>
    <div class="alert alert-<%= flash_class(type) %> alert-dismissible fade show" role="alert">
      <%= message %>
      <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
  <% end %>
  
  <%= yield %>
</body>
</html>
```

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def flash_class(type)
    case type.to_sym
    when :notice, :success then "success"
    when :alert, :error, :danger then "danger"
    when :warning then "warning"
    when :info then "info"
    else "secondary"
    end
  end
  
  def flash_icon(type)
    case type.to_sym
    when :notice, :success then "check-circle"
    when :alert, :error, :danger then "x-circle"
    when :warning then "exclamation-triangle"
    else "info-circle"
    end
  end
end
```

## ขั้นตอนที่ 918: Flash.now

`flash.now` ใช้เมื่อต้องการแสดง flash ใน request เดียวกัน (ไม่ redirect)

```ruby
class PostsController < ApplicationController
  def new
    @post = Post.new
  end

  def create
    @post = Post.new(post_params)
    
    if @post.save
      redirect_to @post, notice: "สร้างบทความสำเร็จ"
    else
      # ใช้ flash.now เพราะเราจะ render (ไม่ redirect)
      flash.now[:alert] = "กรุณากรอกข้อมูลให้ครบถ้วน"
      flash.now[:errors] = @post.errors.full_messages.join(", ")
      render :new, status: :unprocessable_entity
    end
  end

  def search
    @query = params[:q]
    @posts = Post.search(@query)
    
    if @posts.empty?
      flash.now[:info] = "ไม่พบบทความที่ตรงกับ '#{@query}'"
    end
    
    render :index
  end
end
```

## ขั้นตอนที่ 919: Flash.keep และ Flash.discard

```ruby
class ExamplesController < ApplicationController
  def multi_step
    # ทำให้ flash อยู่ต่อไปอีก request
    flash.keep(:notice)
    
    # หรือ keep ทั้งหมด
    flash.keep
    
    redirect_to next_step_path
  end

  def cancel
    # ทิ้ง flash ที่ยังไม่ถูกใช้
    flash.discard(:notice)
    
    # หรือ discard ทั้งหมด
    flash.discard
    
    redirect_to root_path
  end
  
  def step_one
    flash[:wizard_data] = { step: 1, name: params[:name] }
    flash.keep(:wizard_data)  # เก็บไว้สำหรับ step 2
    redirect_to step_two_path
  end
  
  def step_two
    wizard_data = flash[:wizard_data]
    # ใช้ข้อมูลจาก step 1
    flash.keep(:wizard_data)  # เก็บไว้สำหรับ step 3
    redirect_to step_three_path
  end
end
```

## ขั้นตอนที่ 920: Remember Me Functionality

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  
  # สำหรับ remember me
  attr_accessor :remember_token
  
  before_create :generate_remember_token
  
  def self.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? 
           BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end
  
  def self.new_token
    SecureRandom.urlsafe_base64
  end
  
  def remember
    self.remember_token = User.new_token
    update_attribute(:remember_digest, User.digest(remember_token))
  end
  
  def authenticated?(attribute, token)
    digest = send("#{attribute}_digest")
    return false if digest.nil?
    BCrypt::Password.new(digest).is_password?(token)
  end
  
  def forget
    update_attribute(:remember_digest, nil)
  end
  
  private
  
  def generate_remember_token
    self.remember_token = User.new_token
    self.remember_digest = User.digest(remember_token)
  end
end
```

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def new
    # หน้า login form
  end

  def create
    user = User.find_by(email: params[:email].downcase)
    
    if user && user.authenticate(params[:password])
      log_in(user)
      
      # Remember me functionality
      if params[:remember_me] == "1"
        remember(user)
      else
        forget(user)
      end
      
      redirect_back_or(root_path)
    else
      flash.now[:alert] = "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
      render :new, status: :unprocessable_entity
    end
  end

  def destroy
    forget(current_user) if logged_in?
    log_out
    redirect_to root_path, notice: "ออกจากระบบแล้ว"
  end
  
  private
  
  def log_in(user)
    session[:user_id] = user.id
  end
  
  def log_out
    forget(current_user)
    session.delete(:user_id)
    @current_user = nil
  end
  
  def remember(user)
    user.remember
    cookies.permanent.encrypted[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end
  
  def forget(user)
    user.forget
    cookies.delete(:user_id)
    cookies.delete(:remember_token)
  end
  
  def redirect_back_or(default)
    redirect_to(session[:forwarding_url] || default)
    session.delete(:forwarding_url)
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_current_user
  
  private
  
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif (user_id = cookies.encrypted[:user_id])
      user = User.find_by(id: user_id)
      if user && user.authenticated?(:remember, cookies[:remember_token])
        log_in(user)
        @current_user = user
      end
    end
  end
  
  def logged_in?
    current_user.present?
  end
  
  def require_login
    unless logged_in?
      store_location
      flash[:alert] = "กรุณาเข้าสู่ระบบก่อน"
      redirect_to login_path
    end
  end
  
  def store_location
    session[:forwarding_url] = request.original_url if request.get?
  end
  
  helper_method :current_user, :logged_in?
end
```

```erb
<%# app/views/sessions/new.html.erb %>
<h1>เข้าสู่ระบบ</h1>

<%= form_with url: login_path do |f| %>
  <div class="mb-3">
    <%= f.label :email, "อีเมล", class: "form-label" %>
    <%= f.email_field :email, class: "form-control", 
        placeholder: "กรอกอีเมลของคุณ" %>
  </div>
  
  <div class="mb-3">
    <%= f.label :password, "รหัสผ่าน", class: "form-label" %>
    <%= f.password_field :password, class: "form-control" %>
  </div>
  
  <div class="mb-3 form-check">
    <%= f.check_box :remember_me, class: "form-check-input" %>
    <%= f.label :remember_me, "จดจำฉันไว้", class: "form-check-label" %>
  </div>
  
  <%= f.submit "เข้าสู่ระบบ", class: "btn btn-primary" %>
<% end %>
```

## ขั้นตอนที่ 921: Session Security

การรักษาความปลอดภัยของ session เป็นสิ่งสำคัญมาก

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: '_app_session',
  secure: !Rails.env.development?,   # บังคับ HTTPS ใน production
  httponly: true,                     # ป้องกัน XSS
  same_site: :strict,                 # ป้องกัน CSRF
  expire_after: 30.minutes            # timeout อัตโนมัติ
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # ป้องกัน Session Fixation Attack
  before_action :reset_session_on_login
  
  # ตรวจสอบ session timeout
  before_action :check_session_timeout
  
  private
  
  def reset_session_on_login
    # จะทำใน SessionsController#create
  end
  
  def check_session_timeout
    if session[:last_activity] && 
       session[:last_activity] < 30.minutes.ago
      reset_session
      flash[:alert] = "Session หมดอายุ กรุณาเข้าสู่ระบบใหม่"
      redirect_to login_path
    else
      session[:last_activity] = Time.current
    end
  end
end
```

```ruby
# ป้องกัน Session Fixation
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])
    
    if user&.authenticate(params[:password])
      # รีเซ็ต session ก่อน login เพื่อป้องกัน session fixation
      reset_session
      
      # หลังจาก reset ให้ set session ใหม่
      session[:user_id] = user.id
      session[:last_activity] = Time.current
      session[:user_agent] = request.user_agent  # บันทึก browser
      session[:ip_address] = request.remote_ip   # บันทึก IP
      
      redirect_to root_path, notice: "เข้าสู่ระบบสำเร็จ"
    else
      flash.now[:alert] = "ข้อมูลไม่ถูกต้อง"
      render :new
    end
  end
end
```

```ruby
# การตรวจสอบ session integrity
class ApplicationController < ActionController::Base
  before_action :verify_session_integrity
  
  private
  
  def verify_session_integrity
    return unless logged_in?
    
    # ตรวจสอบว่า User Agent เปลี่ยนหรือไม่
    if session[:user_agent] && 
       session[:user_agent] != request.user_agent
      reset_session
      flash[:alert] = "Session ไม่ถูกต้อง"
      redirect_to login_path
    end
    
    # ตรวจสอบ IP (optional - อาจมีปัญหากับ mobile)
    if session[:ip_address] && 
       session[:ip_address] != request.remote_ip
      # แจ้งเตือนแต่ไม่ต้อง logout ทันที
      Rails.logger.warn "IP changed: #{session[:ip_address]} -> #{request.remote_ip}"
    end
  end
end
```

## ขั้นตอนที่ 922: Cookie Security Best Practices

```ruby
# config/environments/production.rb
Rails.application.configure do
  # บังคับ HTTPS
  config.force_ssl = true
  
  # Cookie settings
  config.session_store :cookie_store,
    key: '_app_session',
    secure: true,       # HTTPS เท่านั้น
    httponly: true,     # ป้องกัน XSS
    same_site: :strict  # ป้องกัน CSRF
end
```

```ruby
# CSRF Protection (ใน Rails จะเปิดอยู่แล้ว)
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  
  # สำหรับ API endpoints
  protect_from_forgery with: :null_session
end
```

```ruby
# Content Security Policy
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https, :unsafe_inline
  
  # กำหนด report URI
  policy.report_uri "/csp-violation-report-endpoint"
end
```

## ขั้นตอนที่ 923: Session Storage ขั้นสูง

```ruby
# custom session store
class DatabaseSessionStore < ActionDispatch::Session::AbstractSecureStore
  def initialize(app, options = {})
    super
    @options[:domain] ||= :all
  end
  
  private
  
  def get_session(env, sid)
    session = Session.find_by(session_id: sid.public_id)
    data = session ? deserialize(session.data) : {}
    [sid, data]
  end
  
  def set_session(env, sid, session_data, options)
    session = Session.find_or_initialize_by(session_id: sid.public_id)
    session.data = serialize(session_data)
    session.save!
    sid
  end
  
  def delete_session(env, sid, options)
    Session.find_by(session_id: sid.public_id)&.destroy
    generate_sid
  end
end
```

```ruby
# การติดตาม session ด้วย database
class Session < ApplicationRecord
  belongs_to :user, optional: true
  
  before_create :generate_token
  
  scope :active, -> { where("expires_at > ?", Time.current) }
  scope :expired, -> { where("expires_at <= ?", Time.current) }
  
  def self.cleanup_expired
    expired.delete_all
  end
  
  def expired?
    expires_at < Time.current
  end
  
  def renew!
    update!(expires_at: 30.days.from_now)
  end
  
  private
  
  def generate_token
    self.token = SecureRandom.urlsafe_base64(32)
  end
end
```

## ขั้นตอนที่ 924: Testing Sessions และ Cookies

```ruby
# test/controllers/sessions_controller_test.rb
require "test_helper"

class SessionsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:valid_user)
  end

  test "ควร login สำเร็จด้วยข้อมูลที่ถูกต้อง" do
    post login_path, params: {
      email: @user.email,
      password: "password123"
    }
    
    assert_redirected_to root_path
    assert_equal @user.id, session[:user_id]
    assert_not_nil flash[:notice]
  end

  test "ควรล้มเหลวด้วยรหัสผ่านที่ผิด" do
    post login_path, params: {
      email: @user.email,
      password: "wrong_password"
    }
    
    assert_response :unprocessable_entity
    assert_nil session[:user_id]
    assert_not_nil flash[:alert]
  end

  test "ควร logout และล้าง session" do
    post login_path, params: {
      email: @user.email,
      password: "password123"
    }
    
    delete logout_path
    
    assert_redirected_to root_path
    assert_nil session[:user_id]
  end

  test "ควร remember user เมื่อ check remember_me" do
    post login_path, params: {
      email: @user.email,
      password: "password123",
      remember_me: "1"
    }
    
    assert_not_nil cookies[:remember_token]
    assert_not_nil cookies[:user_id]
  end
end
```

```ruby
# spec/controllers/sessions_controller_spec.rb (RSpec)
require 'rails_helper'

RSpec.describe SessionsController, type: :controller do
  let(:user) { create(:user, password: "password123") }

  describe "POST #create" do
    context "ด้วยข้อมูลที่ถูกต้อง" do
      it "ตั้งค่า session[:user_id]" do
        post :create, params: {
          email: user.email,
          password: "password123"
        }
        expect(session[:user_id]).to eq(user.id)
      end

      it "redirect ไปหน้าหลัก" do
        post :create, params: {
          email: user.email,
          password: "password123"
        }
        expect(response).to redirect_to(root_path)
      end
    end

    context "ด้วยข้อมูลที่ผิด" do
      it "ไม่ตั้งค่า session" do
        post :create, params: {
          email: user.email,
          password: "wrong"
        }
        expect(session[:user_id]).to be_nil
      end
    end
  end
end
```

## ขั้นตอนที่ 925: Multi-Session Management

```ruby
# app/models/user_session.rb
class UserSession < ApplicationRecord
  belongs_to :user
  
  validates :session_token, presence: true, uniqueness: true
  validates :user_agent, presence: true
  
  before_create :set_token_and_expiry
  
  scope :active, -> { where("expires_at > ?", Time.current) }
  
  def self.create_for_user(user, request)
    create!(
      user: user,
      session_token: SecureRandom.urlsafe_base64(32),
      ip_address: request.remote_ip,
      user_agent: request.user_agent,
      expires_at: 30.days.from_now,
      last_active_at: Time.current
    )
  end
  
  def touch_activity!
    update!(last_active_at: Time.current)
  end
  
  def expired?
    expires_at < Time.current
  end
  
  def revoke!
    update!(expires_at: Time.current)
  end
  
  private
  
  def set_token_and_expiry
    self.session_token ||= SecureRandom.urlsafe_base64(32)
    self.expires_at ||= 30.days.from_now
  end
end
```

```ruby
# app/controllers/user_sessions_controller.rb
class UserSessionsController < ApplicationController
  before_action :require_login
  
  def index
    @sessions = current_user.user_sessions.active.order(last_active_at: :desc)
  end
  
  def destroy
    @session = current_user.user_sessions.find(params[:id])
    
    if @session.revoke!
      if @session.session_token == session[:session_token]
        log_out
        redirect_to login_path, notice: "Session นี้ถูก revoke แล้ว"
      else
        redirect_to user_sessions_path, notice: "ยกเลิก session แล้ว"
      end
    end
  end
  
  def destroy_all_others
    current_user.user_sessions
                .active
                .where.not(session_token: session[:session_token])
                .each(&:revoke!)
    
    redirect_to user_sessions_path, notice: "ยกเลิก sessions อื่นๆ ทั้งหมดแล้ว"
  end
end
```

```erb
<%# app/views/user_sessions/index.html.erb %>
<h1>Sessions ที่ Active อยู่</h1>

<table class="table">
  <thead>
    <tr>
      <th>IP Address</th>
      <th>Browser</th>
      <th>เข้าใช้ล่าสุด</th>
      <th>หมดอายุ</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody>
    <% @sessions.each do |user_session| %>
      <tr class="<%= 'table-primary' if user_session.session_token == session[:session_token] %>">
        <td><%= user_session.ip_address %></td>
        <td><%= truncate(user_session.user_agent, length: 50) %></td>
        <td><%= time_ago_in_words(user_session.last_active_at) %> ที่แล้ว</td>
        <td><%= user_session.expires_at.strftime("%d/%m/%Y %H:%M") %></td>
        <td>
          <% if user_session.session_token == session[:session_token] %>
            <span class="badge bg-success">Session ปัจจุบัน</span>
          <% else %>
            <%= link_to "ยกเลิก", user_session_path(user_session), 
                method: :delete, 
                data: { confirm: "ยืนยันการยกเลิก session นี้?" },
                class: "btn btn-sm btn-danger" %>
          <% end %>
        </td>
      </tr>
    <% end %>
  </tbody>
</table>

<%= link_to "ยกเลิก Sessions อื่นๆ ทั้งหมด", 
    destroy_all_others_user_sessions_path,
    method: :delete,
    data: { confirm: "ยืนยันการยกเลิก sessions อื่นๆ ทั้งหมด?" },
    class: "btn btn-warning" %>
```

---

## แบบฝึกหัด: Sessions and Cookies (20 ข้อ)

### ข้อที่ 1: สร้าง Session Counter
```
สร้าง controller ที่นับจำนวนครั้งที่ผู้ใช้เยี่ยมชมหน้าเว็บ
โดยใช้ session เก็บตัวเลขและแสดงในหน้า view
```

**เฉลย:**
```ruby
class VisitCounterController < ApplicationController
  def index
    session[:visit_count] ||= 0
    session[:visit_count] += 1
    @visit_count = session[:visit_count]
  end
  
  def reset
    session[:visit_count] = 0
    redirect_to visit_counter_path, notice: "รีเซ็ตแล้ว"
  end
end
```

### ข้อที่ 2: สร้าง Flash Message ประเภทต่างๆ
```
สร้าง controller ที่แสดง flash messages ทั้ง 4 ประเภท
(success, warning, error, info) ในสถานการณ์ที่เหมาะสม
```

**เฉลย:**
```ruby
class ProductsController < ApplicationController
  def create
    @product = Product.new(product_params)
    if @product.save
      flash[:success] = "สร้างสินค้าสำเร็จ: #{@product.name}"
      redirect_to @product
    else
      flash.now[:error] = "ไม่สามารถสร้างสินค้าได้"
      render :new, status: :unprocessable_entity
    end
  end
  
  def stock_warning
    @products = Product.where("stock < ?", 10)
    flash.now[:warning] = "มีสินค้า #{@products.count} รายการที่ใกล้หมดสต็อก"
    render :low_stock
  end
end
```

### ข้อที่ 3: Cookie Preferences
```
สร้างระบบที่เก็บ preferences ของผู้ใช้ใน cookie
เช่น ภาษา, theme, จำนวนรายการต่อหน้า
```

**เฉลย:**
```ruby
class PreferencesController < ApplicationController
  def update
    preferences = {
      language: params[:language] || "th",
      theme: params[:theme] || "light",
      per_page: params[:per_page] || 20
    }
    
    cookies.encrypted.permanent[:user_preferences] = preferences.to_json
    
    flash[:success] = "บันทึก preferences แล้ว"
    redirect_back fallback_location: root_path
  end
  
  def load_preferences
    if cookies.encrypted[:user_preferences]
      JSON.parse(cookies.encrypted[:user_preferences], symbolize_names: true)
    else
      { language: "th", theme: "light", per_page: 20 }
    end
  end
  helper_method :load_preferences
end
```

### ข้อที่ 4: Shopping Cart Session
```
เขียน Shopping Cart class ที่ใช้ session เก็บข้อมูล
รองรับการเพิ่ม ลบ อัพเดทจำนวน และคำนวณราคารวม
```

**เฉลย:** (ดูตัวอย่างใน Cart class ด้านบน)

### ข้อที่ 5: Remember Me Feature
```
สร้างระบบ Remember Me ที่ใช้ encrypted cookie
ผู้ใช้สามารถ login อัตโนมัติแม้ปิด browser แล้วเปิดใหม่
```

**เฉลย:** (ดูตัวอย่างใน Remember Me section ด้านบน)

### ข้อที่ 6: Session Timeout
```
สร้าง before_action ที่ตรวจสอบ session timeout
ถ้าไม่ได้ใช้งานเกิน 30 นาทีให้ logout อัตโนมัติ
```

**เฉลย:**
```ruby
class ApplicationController < ActionController::Base
  before_action :check_session_expiry
  
  private
  
  def check_session_expiry
    return unless logged_in?
    
    if session[:expires_at].present? && Time.current > session[:expires_at]
      reset_session
      flash[:alert] = "Session หมดอายุ กรุณา login ใหม่"
      redirect_to login_path
    else
      session[:expires_at] = 30.minutes.from_now
    end
  end
end
```

### ข้อที่ 7: Multi-step Form with Session
```
สร้าง multi-step form ที่ใช้ session เก็บข้อมูลแต่ละ step
มี 3 steps: ข้อมูลส่วนตัว, ที่อยู่, การชำระเงิน
```

**เฉลย:**
```ruby
class RegistrationController < ApplicationController
  def step1
    @user_data = session[:registration] || {}
  end
  
  def save_step1
    session[:registration] ||= {}
    session[:registration].merge!(
      name: params[:name],
      email: params[:email],
      phone: params[:phone]
    )
    redirect_to registration_step2_path
  end
  
  def step2
    redirect_to registration_step1_path unless session[:registration]&.dig(:name)
    @address_data = session[:registration]
  end
  
  def save_step2
    session[:registration].merge!(
      address: params[:address],
      city: params[:city],
      zip: params[:zip]
    )
    redirect_to registration_step3_path
  end
  
  def step3
    @all_data = session[:registration]
  end
  
  def complete
    user_data = session[:registration]
    @user = User.create!(user_data)
    session.delete(:registration)
    redirect_to root_path, notice: "ลงทะเบียนสำเร็จ"
  end
end
```

### ข้อที่ 8: Signed Cookie for Tracking
```
ใช้ signed cookie เก็บ tracking ID สำหรับ analytics
โดยไม่ต้องให้ผู้ใช้ login
```

**เฉลย:**
```ruby
class ApplicationController < ActionController::Base
  before_action :set_visitor_tracking
  
  private
  
  def set_visitor_tracking
    cookies.signed[:visitor_id] ||= {
      value: SecureRandom.uuid,
      expires: 1.year.from_now,
      httponly: true
    }
    
    @visitor_id = cookies.signed[:visitor_id]
  end
end
```

### ข้อที่ 9: Flash ใน AJAX Requests
```
สร้าง helper ที่ส่ง flash messages กลับมาผ่าน JSON response
สำหรับ AJAX requests ที่ไม่ได้ redirect
```

**เฉลย:**
```ruby
class ApplicationController < ActionController::Base
  def flash_messages_json
    flash.map { |type, message| { type: type, message: message } }
  end
  helper_method :flash_messages_json
end

class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    
    respond_to do |format|
      if @post.save
        format.json { 
          render json: { 
            post: @post,
            flash: [{ type: "success", message: "บันทึกสำเร็จ" }]
          }
        }
      else
        format.json { 
          render json: { 
            errors: @post.errors,
            flash: [{ type: "error", message: "บันทึกไม่สำเร็จ" }]
          }, status: :unprocessable_entity 
        }
      end
    end
  end
end
```

### ข้อที่ 10: Testing Session Data
```
เขียน integration test ที่ทดสอบ:
1. Login สำเร็จมี session[:user_id]
2. Logout ล้าง session
3. Session timeout redirect ไป login
```

**เฉลย:**
```ruby
class SessionIntegrationTest < ActionDispatch::IntegrationTest
  setup do
    @user = create(:user, password: "password123")
  end
  
  test "login ตั้งค่า session ถูกต้อง" do
    post sessions_path, params: { email: @user.email, password: "password123" }
    assert_equal @user.id, session[:user_id]
    assert_not_nil session[:expires_at]
  end
  
  test "logout ล้าง session ทั้งหมด" do
    post sessions_path, params: { email: @user.email, password: "password123" }
    delete session_path
    assert_nil session[:user_id]
  end
  
  test "expired session redirect ไป login" do
    post sessions_path, params: { email: @user.email, password: "password123" }
    session[:expires_at] = 31.minutes.ago
    get protected_path
    assert_redirected_to login_path
  end
end
```

### ข้อที่ 11-20 (แบบสรุป)

**ข้อ 11:** สร้าง Last Visited Pages ด้วย session เก็บประวัติ 5 หน้าล่าสุด

**ข้อ 12:** สร้างระบบ "แนะนำสำหรับคุณ" โดยใช้ session เก็บสินค้าที่ดู

**ข้อ 13:** สร้าง consent cookie สำหรับ GDPR compliance

**ข้อ 14:** ทดสอบ flash messages ด้วย RSpec

**ข้อ 15:** สร้าง session-based language preference

**ข้อ 16:** ใช้ flash.keep สำหรับ multi-redirect wizard

**ข้อ 17:** สร้าง cookie-based dark mode toggle

**ข้อ 18:** สร้าง secure session management ป้องกัน session fixation

**ข้อ 19:** สร้าง "Recently Viewed" feature ด้วย session

**ข้อ 20:** สร้าง session store custom ที่บันทึก analytics data

```ruby
# ข้อ 11: Last Visited Pages
class ApplicationController < ActionController::Base
  before_action :track_page_visit
  
  private
  
  def track_page_visit
    return unless request.get? && !request.xhr?
    
    session[:visited_pages] ||= []
    current_page = {
      url: request.original_url,
      title: @page_title || "ไม่ระบุ",
      visited_at: Time.current.iso8601
    }
    
    session[:visited_pages].unshift(current_page)
    session[:visited_pages] = session[:visited_pages].first(5)
  end
end
```

```ruby
# ข้อ 19: Recently Viewed Products
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])
    track_product_view(@product)
    @recently_viewed = recently_viewed_products
  end
  
  private
  
  def track_product_view(product)
    session[:viewed_products] ||= []
    session[:viewed_products].delete(product.id)
    session[:viewed_products].unshift(product.id)
    session[:viewed_products] = session[:viewed_products].first(10)
  end
  
  def recently_viewed_products
    ids = session[:viewed_products] || []
    Product.where(id: ids).index_by(&:id).values_at(*ids).compact
  end
  helper_method :recently_viewed_products
end
```

---

## สรุป: Sessions and Cookies

| คุณสมบัติ | Session | Cookie |
|-----------|---------|--------|
| เก็บที่ | Server (หรือ client ถ้า cookie store) | Client browser |
| ขนาด | ไม่จำกัด (ถ้า DB/Redis) | สูงสุด 4KB |
| ความปลอดภัย | สูงกว่า | ต้องเข้ารหัส |
| อายุ | จนกว่าจะ expire | กำหนดเองได้ |
| การใช้งาน | User login, cart | Preferences, tracking |

**Key Takeaways:**
1. ใช้ `session` สำหรับข้อมูลที่ sensitive (user_id, permissions)
2. ใช้ `cookies` สำหรับ preferences ที่ไม่ sensitive
3. ใช้ `cookies.encrypted` สำหรับข้อมูลที่ต้องปลอดภัย
4. ใช้ `flash` สำหรับข้อความแจ้งเตือนชั่วคราว
5. Always reset session on login เพื่อป้องกัน session fixation
6. ตั้งค่า httponly และ secure สำหรับ production
