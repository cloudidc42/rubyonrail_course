# Part 42: Authentication with Devise

## ขั้นตอนที่ 926-950: การยืนยันตัวตนใน Rails

---

## ขั้นตอนที่ 926: Authentication คืออะไร?

Authentication (การยืนยันตัวตน) คือกระบวนการตรวจสอบว่าผู้ใช้เป็นใคร แตกต่างจาก Authorization ที่ตรวจสอบว่าผู้ใช้มีสิทธิ์ทำอะไรได้บ้าง

**กลไกหลักของ Authentication:**
1. Password-based (ชื่อผู้ใช้ + รหัสผ่าน)
2. Token-based (API key, JWT)
3. OAuth/SSO (Google, Facebook, GitHub)
4. Multi-factor Authentication (OTP, SMS)
5. Biometric (ลายนิ้วมือ, ใบหน้า)

## ขั้นตอนที่ 927: Authentication จากศูนย์ (From Scratch)

ก่อนใช้ Devise ลองสร้าง authentication เองเพื่อเข้าใจหลักการ

```ruby
# Migration
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :name, null: false
      t.string :email, null: false
      t.string :password_digest, null: false
      t.string :remember_digest
      t.boolean :admin, default: false
      t.string :activation_digest
      t.boolean :activated, default: false
      t.datetime :activated_at
      t.string :reset_digest
      t.datetime :reset_sent_at
      
      t.timestamps
    end
    
    add_index :users, :email, unique: true
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # has_secure_password เพิ่ม method authenticate
  # และ validates :password สำหรับ bcrypt
  has_secure_password
  
  # Validations
  validates :name, presence: true, length: { maximum: 50 }
  validates :email, 
    presence: true, 
    length: { maximum: 255 },
    format: { with: URI::MailTo::EMAIL_REGEXP },
    uniqueness: { case_sensitive: false }
  validates :password, 
    presence: true, 
    length: { minimum: 8 },
    allow_nil: true  # อนุญาต nil เพื่อ update อื่นๆ
  
  # Callbacks
  before_save :downcase_email
  before_create :create_activation_digest
  
  # Accessor สำหรับ token (ไม่ได้เก็บใน DB)
  attr_accessor :remember_token, :activation_token, :reset_token
  
  # Class methods
  def self.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? 
           BCrypt::Engine::MIN_COST : BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end
  
  def self.new_token
    SecureRandom.urlsafe_base64
  end
  
  # Instance methods
  def remember
    self.remember_token = User.new_token
    update_attribute(:remember_digest, User.digest(remember_token))
  end
  
  def forget
    update_attribute(:remember_digest, nil)
  end
  
  def authenticated?(attribute, token)
    digest = send("#{attribute}_digest")
    return false if digest.nil?
    BCrypt::Password.new(digest).is_password?(token)
  end
  
  def activate
    update_columns(activated: true, activated_at: Time.zone.now)
  end
  
  def send_activation_email
    UserMailer.account_activation(self).deliver_now
  end
  
  def create_reset_digest
    self.reset_token = User.new_token
    update_columns(
      reset_digest: User.digest(reset_token),
      reset_sent_at: Time.zone.now
    )
  end
  
  def send_password_reset_email
    UserMailer.password_reset(self).deliver_now
  end
  
  def password_reset_expired?
    reset_sent_at < 2.hours.ago
  end
  
  private
  
  def downcase_email
    email.downcase!
  end
  
  def create_activation_digest
    self.activation_token = User.new_token
    self.activation_digest = User.digest(activation_token)
  end
end
```

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  before_action :require_login, only: [:edit, :update, :destroy]
  before_action :correct_user, only: [:edit, :update]
  before_action :admin_user, only: :destroy
  
  def new
    @user = User.new
  end
  
  def create
    @user = User.new(user_params)
    
    if @user.save
      @user.send_activation_email
      flash[:info] = "กรุณาตรวจสอบอีเมลของคุณเพื่อยืนยันบัญชี"
      redirect_to root_path
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  def show
    @user = User.find(params[:id])
    redirect_to root_path unless @user.activated?
  end
  
  def edit
    @user = User.find(params[:id])
  end
  
  def update
    @user = User.find(params[:id])
    
    if @user.update(user_params)
      flash[:success] = "อัพเดทโปรไฟล์แล้ว"
      redirect_to @user
    else
      render :edit, status: :unprocessable_entity
    end
  end
  
  def destroy
    User.find(params[:id]).destroy
    flash[:success] = "ลบผู้ใช้แล้ว"
    redirect_to users_path
  end
  
  private
  
  def user_params
    params.require(:user).permit(:name, :email, :password, :password_confirmation)
  end
  
  def correct_user
    @user = User.find(params[:id])
    redirect_to root_path unless current_user?(@user)
  end
  
  def admin_user
    redirect_to root_path unless current_user.admin?
  end
end
```

## ขั้นตอนที่ 928: Devise Gem คืออะไร?

Devise เป็น gem ที่ครบครันสำหรับ authentication ใน Rails ประกอบด้วย modules ต่างๆ:

- **Database Authenticatable** - เข้ารหัสและตรวจสอบรหัสผ่าน
- **Registerable** - ลงทะเบียนผู้ใช้ใหม่
- **Recoverable** - รีเซ็ตรหัสผ่าน
- **Rememberable** - Remember Me cookie
- **Trackable** - บันทึก login information
- **Validatable** - Validation email และ password
- **Confirmable** - ยืนยัน email
- **Lockable** - ล็อคบัญชีหลัง login ผิดหลายครั้ง
- **Timeoutable** - หมดเวลา session
- **Omniauthable** - OAuth integration

## ขั้นตอนที่ 929: การติดตั้ง Devise

```ruby
# Gemfile
gem 'devise'
gem 'devise-i18n'  # สำหรับ i18n support
```

```bash
# ติดตั้ง gem
bundle install

# Run installer
rails generate devise:install
```

```ruby
# config/initializers/devise.rb (ที่สร้างขึ้นอัตโนมัติ)
Devise.setup do |config|
  # กำหนด mailer sender
  config.mailer_sender = "noreply@myapp.com"
  
  # กำหนด ORM
  config.orm = :active_record
  
  # กำหนด password
  config.password_length = 8..128
  config.email_regexp = /\A[^@\s]+@[^@\s]+\z/
  
  # กำหนด confirmation
  config.confirm_within = 3.days
  
  # กำหนด remember
  config.remember_for = 2.weeks
  
  # กำหนด timeout
  config.timeout_in = 30.minutes
  
  # กำหนด lock
  config.maximum_attempts = 5
  config.unlock_strategy = :email
  
  # กำหนด navigation
  config.sign_out_via = :delete
  
  # กำหนด secret key
  config.secret_key = ENV['DEVISE_SECRET_KEY']
end
```

```ruby
# config/environments/development.rb
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users
  root to: 'home#index'
end
```

## ขั้นตอนที่ 930: สร้าง User Model ด้วย Devise

```bash
# สร้าง User model ด้วย Devise
rails generate devise User

# รัน migration
rails db:migrate
```

```ruby
# Migration ที่สร้างขึ้น
class DeviseCreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table(:users) do |t|
      ## Database authenticatable
      t.string :email, null: false, default: ""
      t.string :encrypted_password, null: false, default: ""
      
      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at
      
      ## Rememberable
      t.datetime :remember_created_at
      
      ## Trackable
      t.integer  :sign_in_count, default: 0, null: false
      t.datetime :current_sign_in_at
      t.datetime :last_sign_in_at
      t.string   :current_sign_in_ip
      t.string   :last_sign_in_ip
      
      ## Confirmable
      t.string   :confirmation_token
      t.datetime :confirmed_at
      t.datetime :confirmation_sent_at
      t.string   :unconfirmed_email
      
      ## Lockable
      t.integer  :failed_attempts, default: 0, null: false
      t.string   :unlock_token
      t.datetime :locked_at
      
      t.timestamps null: false
    end
    
    add_index :users, :email, unique: true
    add_index :users, :reset_password_token, unique: true
    add_index :users, :confirmation_token, unique: true
    add_index :users, :unlock_token, unique: true
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # Include default devise modules
  devise :database_authenticatable, 
         :registerable,
         :recoverable, 
         :rememberable, 
         :validatable,
         :confirmable,
         :lockable,
         :timeoutable,
         :trackable
  
  # Additional columns
  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy
  
  # Custom validations
  validates :name, presence: true, length: { maximum: 100 }
  
  # Enums
  enum role: { user: 0, moderator: 1, admin: 2 }
  
  # Scopes
  scope :admins, -> { where(role: :admin) }
  scope :active, -> { where(active: true) }
  
  # Methods
  def full_name
    "#{first_name} #{last_name}".strip
  end
  
  def display_name
    name.presence || email.split("@").first
  end
end
```

## ขั้นตอนที่ 931: เพิ่ม Fields ให้ User

```bash
# เพิ่ม columns ให้ users table
rails generate migration AddFieldsToUsers \
  name:string \
  avatar:string \
  bio:text \
  role:integer \
  active:boolean
  
rails db:migrate
```

```ruby
# migration ที่แก้ไข
class AddFieldsToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :name, :string
    add_column :users, :avatar, :string
    add_column :users, :bio, :text
    add_column :users, :role, :integer, default: 0
    add_column :users, :active, :boolean, default: true
    add_column :users, :timezone, :string, default: "Bangkok"
    add_column :users, :locale, :string, default: "th"
    
    add_index :users, :role
    add_index :users, :active
  end
end
```

## ขั้นตอนที่ 932: Devise Views

```bash
# สร้าง views สำหรับ Devise
rails generate devise:views

# หรือ สร้างสำหรับ User เท่านั้น
rails generate devise:views users
```

โครงสร้าง views ที่สร้างขึ้น:
```
app/views/devise/
├── confirmations/
│   └── new.html.erb
├── mailer/
│   ├── confirmation_instructions.html.erb
│   ├── email_changed.html.erb
│   ├── password_change.html.erb
│   ├── reset_password_instructions.html.erb
│   └── unlock_instructions.html.erb
├── passwords/
│   ├── edit.html.erb
│   └── new.html.erb
├── registrations/
│   ├── edit.html.erb
│   └── new.html.erb
├── sessions/
│   └── new.html.erb
├── shared/
│   ├── _error_messages.html.erb
│   └── _links.html.erb
└── unlocks/
    └── new.html.erb
```

```erb
<%# app/views/devise/sessions/new.html.erb %>
<div class="min-h-screen flex items-center justify-center">
  <div class="w-full max-w-md">
    <div class="bg-white rounded-lg shadow-lg p-8">
      <h2 class="text-2xl font-bold text-center mb-6">เข้าสู่ระบบ</h2>
      
      <%= form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
        <div class="mb-4">
          <%= f.label :email, "อีเมล", class: "block text-sm font-medium text-gray-700 mb-1" %>
          <%= f.email_field :email, 
              autofocus: true, 
              autocomplete: "email",
              class: "w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500",
              placeholder: "กรอกอีเมลของคุณ" %>
        </div>
        
        <div class="mb-4">
          <%= f.label :password, "รหัสผ่าน", class: "block text-sm font-medium text-gray-700 mb-1" %>
          <%= f.password_field :password, 
              autocomplete: "current-password",
              class: "w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500",
              placeholder: "กรอกรหัสผ่าน" %>
        </div>
        
        <% if devise_mapping.rememberable? %>
          <div class="flex items-center mb-4">
            <%= f.check_box :remember_me, class: "mr-2" %>
            <%= f.label :remember_me, "จดจำฉันไว้", class: "text-sm text-gray-600" %>
          </div>
        <% end %>
        
        <div class="mb-4">
          <%= f.submit "เข้าสู่ระบบ", 
              class: "w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 transition-colors" %>
        </div>
      <% end %>
      
      <div class="text-center space-y-2">
        <%= render "devise/shared/links" %>
      </div>
    </div>
  </div>
</div>
```

```erb
<%# app/views/devise/registrations/new.html.erb %>
<div class="container mx-auto max-w-md mt-8 px-4">
  <h2 class="text-2xl font-bold text-center mb-6">สร้างบัญชีใหม่</h2>
  
  <%= form_for(resource, as: resource_name, url: registration_path(resource_name)) do |f| %>
    <%= render "devise/shared/error_messages", resource: resource %>
    
    <div class="mb-4">
      <%= f.label :name, "ชื่อ-นามสกุล", class: "block text-sm font-medium mb-1" %>
      <%= f.text_field :name, 
          class: "w-full px-3 py-2 border rounded-md",
          placeholder: "ชื่อของคุณ" %>
    </div>
    
    <div class="mb-4">
      <%= f.label :email, "อีเมล", class: "block text-sm font-medium mb-1" %>
      <%= f.email_field :email, 
          autofocus: true, 
          autocomplete: "email",
          class: "w-full px-3 py-2 border rounded-md" %>
    </div>
    
    <div class="mb-4">
      <%= f.label :password, "รหัสผ่าน", class: "block text-sm font-medium mb-1" %>
      <% if @minimum_password_length %>
        <em class="text-gray-500 text-sm">
          อย่างน้อย <%= @minimum_password_length %> ตัวอักษร
        </em>
      <% end %>
      <%= f.password_field :password, 
          autocomplete: "new-password",
          class: "w-full px-3 py-2 border rounded-md" %>
    </div>
    
    <div class="mb-6">
      <%= f.label :password_confirmation, "ยืนยันรหัสผ่าน", class: "block text-sm font-medium mb-1" %>
      <%= f.password_field :password_confirmation, 
          autocomplete: "new-password",
          class: "w-full px-3 py-2 border rounded-md" %>
    </div>
    
    <div class="mb-4">
      <%= f.submit "สร้างบัญชี", 
          class: "w-full bg-green-600 text-white py-2 px-4 rounded-md hover:bg-green-700" %>
    </div>
  <% end %>
  
  <div class="text-center">
    <%= render "devise/shared/links" %>
  </div>
</div>
```

## ขั้นตอนที่ 933: Devise Controllers Customization

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, 
    controllers: {
      sessions: 'users/sessions',
      registrations: 'users/registrations',
      passwords: 'users/passwords',
      confirmations: 'users/confirmations'
    }
end
```

```bash
# สร้าง custom controllers
rails generate devise:controllers users
```

```ruby
# app/controllers/users/sessions_controller.rb
class Users::SessionsController < Devise::SessionsController
  # before_action :configure_sign_in_params, only: [:create]
  
  def new
    @user = User.new
    super
  end
  
  def create
    # เพิ่ม custom logic หลัง login
    super do |user|
      # บันทึก login attempt
      UserLoginLog.create!(
        user: user,
        ip_address: request.remote_ip,
        user_agent: request.user_agent
      )
      
      # แจ้งเตือนผ่าน email ถ้า login จาก IP ใหม่
      if user.last_sign_in_ip && user.last_sign_in_ip != request.remote_ip
        UserMailer.new_ip_login(user, request.remote_ip).deliver_later
      end
    end
  end
  
  def destroy
    # ทำ custom logout
    user = current_user
    super
    
    # ล้าง cache ของ user
    Rails.cache.delete("user_#{user.id}_data") if user
  end
  
  protected
  
  def after_sign_in_path_for(resource)
    # Redirect ไปหน้าที่ต้องการหลัง login
    stored_location_for(resource) || dashboard_path
  end
  
  def after_sign_out_path_for(resource_or_scope)
    root_path
  end
end
```

```ruby
# app/controllers/users/registrations_controller.rb
class Users::RegistrationsController < Devise::RegistrationsController
  before_action :configure_account_update_params, only: [:update]
  before_action :configure_sign_up_params, only: [:create]
  
  def create
    super do |user|
      # ส่ง welcome email
      UserMailer.welcome(user).deliver_later if user.persisted?
      
      # สร้าง default profile
      user.create_profile!(bio: "ยินดีต้อนรับสู่แอปของเรา!") if user.persisted?
    end
  end
  
  def update
    super do |user|
      # บันทึก profile update
      UserActivityLog.create!(user: user, action: "profile_updated")
    end
  end
  
  protected
  
  def configure_sign_up_params
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name, :avatar])
  end
  
  def configure_account_update_params
    devise_parameter_sanitizer.permit(:account_update, 
      keys: [:name, :avatar, :bio, :timezone])
  end
  
  def after_sign_up_path_for(resource)
    # หลังสมัครสมาชิกให้ไปหน้า dashboard
    dashboard_path
  end
  
  def after_inactive_sign_up_path_for(resource)
    # หลังสมัครแต่ยังไม่ยืนยัน email
    new_user_session_path
  end
end
```

## ขั้นตอนที่ 934: Devise Parameters Sanitizer

Devise ต้องการกำหนด parameters ที่อนุญาตให้ผ่าน (whitelist)

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?
  
  protected
  
  def configure_permitted_parameters
    # สำหรับการลงทะเบียน
    devise_parameter_sanitizer.permit(:sign_up, 
      keys: [:name, :email, :phone, :avatar, :terms_accepted])
    
    # สำหรับการอัพเดทบัญชี
    devise_parameter_sanitizer.permit(:account_update, 
      keys: [:name, :phone, :avatar, :bio, :timezone, :locale])
    
    # สำหรับการ login
    devise_parameter_sanitizer.permit(:sign_in,
      keys: [:remember_me])
  end
end
```

## ขั้นตอนที่ 935: Devise Helpers

Devise มี helper methods ที่ใช้งานได้ทั้งใน controllers และ views

```ruby
# Helpers ที่ Devise ให้มา
class ExamplesController < ApplicationController
  before_action :authenticate_user!  # บังคับ login
  
  def index
    # current_user - user ที่ login อยู่
    @user = current_user
    
    # user_signed_in? - ตรวจสอบว่า login หรือยัง
    if user_signed_in?
      @posts = current_user.posts
    end
    
    # user_session - session hash ของ user
    theme = user_session[:theme] || "light"
  end
end
```

```erb
<%# ใช้ helpers ใน views %>
<nav>
  <% if user_signed_in? %>
    <span>สวัสดี, <%= current_user.name %>!</span>
    <%= link_to "โปรไฟล์", edit_user_registration_path %>
    <%= link_to "ออกจากระบบ", destroy_user_session_path, method: :delete %>
  <% else %>
    <%= link_to "เข้าสู่ระบบ", new_user_session_path %>
    <%= link_to "สมัครสมาชิก", new_user_registration_path %>
  <% end %>
</nav>
```

```ruby
# ถ้า authentication scope ไม่ใช่ user
# devise_for :admins
# current_admin
# admin_signed_in?
# authenticate_admin!
```

## ขั้นตอนที่ 936: Email Confirmation

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, 
         :registerable,
         :confirmable,  # เพิ่ม confirmable
         :recoverable, 
         :rememberable, 
         :validatable
end
```

```ruby
# config/initializers/devise.rb
config.confirm_within = 3.days  # ยืนยันภายใน 3 วัน
config.reconfirmable = true      # ต้องยืนยันเมื่อเปลี่ยน email
```

```erb
<%# app/views/devise/mailer/confirmation_instructions.html.erb %>
<p>สวัสดี <%= @email %>!</p>

<p>คุณได้ลงทะเบียนใน <%= Rails.application.class.module_parent_name %> แล้ว</p>

<p>กรุณาคลิกลิงค์ด้านล่างเพื่อยืนยันอีเมลของคุณ:</p>

<p><%= link_to 'ยืนยันอีเมล', confirmation_url(@resource, confirmation_token: @token) %></p>

<p>ลิงค์นี้จะหมดอายุใน <%= Devise.confirm_within.to_i / 3600 %> ชั่วโมง</p>

<p>หากคุณไม่ได้ลงทะเบียน กรุณาละเว้นอีเมลนี้</p>
```

```ruby
# การ resend confirmation
class Users::ConfirmationsController < Devise::ConfirmationsController
  def new
    super
  end
  
  def create
    self.resource = resource_class.send_confirmation_instructions(resource_params)
    
    if successfully_sent?(resource)
      flash[:success] = "ส่งอีเมลยืนยันใหม่แล้ว"
      redirect_to new_user_session_path
    else
      respond_with resource
    end
  end
end
```

## ขั้นตอนที่ 937: Password Reset

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable,
         :recoverable,  # รีเซ็ตรหัสผ่าน
         :validatable
         
  # กำหนดเงื่อนไขรหัสผ่าน
  validates :password, 
    format: { 
      with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      message: "ต้องมีตัวพิมพ์ใหญ่ ตัวพิมพ์เล็ก และตัวเลขอย่างน้อย 1 ตัว",
      allow_blank: true 
    }
end
```

```ruby
# app/controllers/users/passwords_controller.rb
class Users::PasswordsController < Devise::PasswordsController
  def create
    # ส่ง reset email
    super do |resource|
      Rails.logger.info "Password reset requested for #{resource.email}"
    end
  end
  
  def update
    super do |resource|
      if resource.errors.empty?
        # บันทึก password change event
        UserActivityLog.create!(
          user: resource,
          action: "password_reset",
          ip_address: request.remote_ip
        )
      end
    end
  end
end
```

```erb
<%# app/views/devise/passwords/new.html.erb %>
<h2>ลืมรหัสผ่าน?</h2>

<p>กรอกอีเมลของคุณ เราจะส่งลิงค์รีเซ็ตรหัสผ่านให้</p>

<%= form_for(resource, as: resource_name, url: password_path(resource_name), html: { method: :post }) do |f| %>
  <%= render "devise/shared/error_messages", resource: resource %>
  
  <div class="mb-4">
    <%= f.label :email, "อีเมล" %>
    <%= f.email_field :email, autofocus: true, autocomplete: "email", class: "form-control" %>
  </div>
  
  <div>
    <%= f.submit "ส่งลิงค์รีเซ็ตรหัสผ่าน", class: "btn btn-primary" %>
  </div>
<% end %>

<div class="mt-4">
  <%= render "devise/shared/links" %>
</div>
```

```erb
<%# app/views/devise/passwords/edit.html.erb %>
<h2>ตั้งรหัสผ่านใหม่</h2>

<%= form_for(resource, as: resource_name, url: password_path(resource_name), html: { method: :put }) do |f| %>
  <%= render "devise/shared/error_messages", resource: resource %>
  
  <%= f.hidden_field :reset_password_token %>
  
  <div class="mb-4">
    <%= f.label :password, "รหัสผ่านใหม่" %>
    <% if @minimum_password_length %>
      <em>อย่างน้อย <%= @minimum_password_length %> ตัวอักษร</em>
    <% end %>
    <%= f.password_field :password, autofocus: true, autocomplete: "new-password", class: "form-control" %>
  </div>
  
  <div class="mb-4">
    <%= f.label :password_confirmation, "ยืนยันรหัสผ่านใหม่" %>
    <%= f.password_field :password_confirmation, autocomplete: "new-password", class: "form-control" %>
  </div>
  
  <div>
    <%= f.submit "บันทึกรหัสผ่านใหม่", class: "btn btn-primary" %>
  </div>
<% end %>
```

## ขั้นตอนที่ 938: Devise Mailer Templates

```ruby
# app/mailers/devise_mailer.rb (custom mailer)
class DeviseMailer < Devise::Mailer
  helper :application
  include Devise::Controllers::UrlHelpers
  default template_path: 'devise/mailer'
  
  def confirmation_instructions(record, token, opts = {})
    @token = token
    devise_mail(record, :confirmation_instructions, opts)
  end
  
  def password_reset_instructions(record, token, opts = {})
    @token = token
    @subject = "รีเซ็ตรหัสผ่านของคุณ"
    devise_mail(record, :reset_password_instructions, opts)
  end
end
```

```erb
<%# app/views/devise/mailer/reset_password_instructions.html.erb %>
<!DOCTYPE html>
<html>
<body style="font-family: sans-serif; max-width: 600px; margin: 0 auto;">
  <div style="background: #f4f4f4; padding: 20px;">
    <h1 style="color: #333;">รีเซ็ตรหัสผ่าน</h1>
    
    <p>สวัสดี <%= @resource.name || @resource.email %>!</p>
    
    <p>มีการขอรีเซ็ตรหัสผ่านสำหรับบัญชีของคุณ</p>
    
    <p>คลิกปุ่มด้านล่างเพื่อตั้งรหัสผ่านใหม่:</p>
    
    <div style="text-align: center; margin: 30px 0;">
      <%= link_to "ตั้งรหัสผ่านใหม่",
          edit_password_url(@resource, reset_password_token: @token),
          style: "background: #007bff; color: white; padding: 12px 24px; 
                  text-decoration: none; border-radius: 4px;" %>
    </div>
    
    <p style="color: #666; font-size: 14px;">
      ลิงค์นี้จะหมดอายุใน 6 ชั่วโมง
    </p>
    
    <p style="color: #666; font-size: 14px;">
      หากคุณไม่ได้ขอรีเซ็ตรหัสผ่าน กรุณาละเว้นอีเมลนี้
    </p>
  </div>
</body>
</html>
```

## ขั้นตอนที่ 939: OAuth with Devise (Omniauth)

```ruby
# Gemfile
gem 'omniauth'
gem 'omniauth-google-oauth2'
gem 'omniauth-facebook'
gem 'omniauth-rails_csrf_protection'
```

```ruby
# config/initializers/devise.rb
config.omniauth :google_oauth2,
  ENV['GOOGLE_CLIENT_ID'],
  ENV['GOOGLE_CLIENT_SECRET'],
  scope: 'email,profile'

config.omniauth :facebook,
  ENV['FACEBOOK_APP_ID'],
  ENV['FACEBOOK_APP_SECRET'],
  scope: 'email'
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :omniauthable, omniauth_providers: [:google_oauth2, :facebook]
  
  def self.from_omniauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0, 20]
      user.name = auth.info.name
      user.avatar = auth.info.image
      user.skip_confirmation!  # ไม่ต้องยืนยัน email
    end
  end
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, 
    controllers: { 
      omniauth_callbacks: 'users/omniauth_callbacks'
    }
end
```

```ruby
# app/controllers/users/omniauth_callbacks_controller.rb
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def google_oauth2
    handle_omniauth(:google_oauth2, "Google")
  end
  
  def facebook
    handle_omniauth(:facebook, "Facebook")
  end
  
  def failure
    redirect_to root_path, alert: "การ login ผ่าน OAuth ล้มเหลว"
  end
  
  private
  
  def handle_omniauth(provider, provider_name)
    @user = User.from_omniauth(request.env["omniauth.auth"])
    
    if @user.persisted?
      flash[:notice] = I18n.t "devise.omniauth_callbacks.success", kind: provider_name
      sign_in_and_redirect @user, event: :authentication
    else
      session["devise.#{provider}_data"] = request.env["omniauth.auth"].except("extra")
      redirect_to new_user_registration_url, 
        alert: @user.errors.full_messages.join("\n")
    end
  end
end
```

```erb
<%# ปุ่ม Social Login %>
<div class="social-login">
  <p>หรือเข้าสู่ระบบด้วย:</p>
  
  <%= link_to "เข้าสู่ระบบด้วย Google",
      user_google_oauth2_omniauth_authorize_path,
      method: :post,
      class: "btn btn-google" %>
  
  <%= link_to "เข้าสู่ระบบด้วย Facebook",
      user_facebook_omniauth_authorize_path,
      method: :post,
      class: "btn btn-facebook" %>
</div>
```

## ขั้นตอนที่ 940: Devise Lockable

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable,
         :lockable  # ล็อคบัญชีหลังพยายาม login ผิดหลายครั้ง
end
```

```ruby
# config/initializers/devise.rb
config.maximum_attempts = 5  # ล็อคหลังจาก 5 ครั้ง
config.lock_strategy = :failed_attempts
config.unlock_strategy = :email  # ส่ง email เพื่อ unlock
config.unlock_in = 1.hour  # หรือ unlock อัตโนมัติใน 1 ชั่วโมง
```

## ขั้นตอนที่ 941: Devise Timeoutable

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :timeoutable
end
```

```ruby
# config/initializers/devise.rb
config.timeout_in = 30.minutes  # หมดเวลาใน 30 นาที
```

```ruby
# Custom timeout per user
# app/models/user.rb
class User < ApplicationRecord
  devise :timeoutable
  
  def timeout_in
    if admin?
      30.minutes
    else
      2.hours
    end
  end
end
```

## ขั้นตอนที่ 942: Two-Factor Authentication

```ruby
# Gemfile
gem 'devise-two-factor'
gem 'rqrcode'  # สำหรับ QR code
```

```bash
rails generate devise_two_factor User ENCRYPT_PASSWORD
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :two_factor_authenticatable,
         :two_factor_backupable,
         otp_secret_encryption_key: ENV['OTP_SECRET_KEY']
  
  has_many :otp_backup_codes
end
```

```ruby
# app/controllers/two_factor_settings_controller.rb
class TwoFactorSettingsController < ApplicationController
  before_action :authenticate_user!
  
  def show
    if current_user.otp_required_for_login
      @qr_code = generate_qr_code
    end
  end
  
  def create
    current_user.otp_secret = User.generate_otp_secret
    current_user.save!
    
    @qr_code = generate_qr_code
    render :show
  end
  
  def update
    if current_user.validate_and_consume_otp!(params[:otp_attempt])
      current_user.update!(otp_required_for_login: true)
      flash[:success] = "เปิดใช้งาน 2FA แล้ว"
      redirect_to root_path
    else
      flash[:error] = "รหัส OTP ไม่ถูกต้อง"
      render :show
    end
  end
  
  def destroy
    if current_user.validate_and_consume_otp!(params[:otp_attempt])
      current_user.update!(otp_required_for_login: false)
      flash[:success] = "ปิด 2FA แล้ว"
      redirect_to root_path
    else
      flash[:error] = "รหัส OTP ไม่ถูกต้อง"
      render :show
    end
  end
  
  private
  
  def generate_qr_code
    issuer = Rails.application.class.module_parent_name
    otp_uri = current_user.otp_provisioning_uri(
      current_user.email, 
      issuer: issuer
    )
    
    qr = RQRCode::QRCode.new(otp_uri)
    qr.as_svg(
      offset: 0,
      color: '000',
      shape_rendering: 'crispEdges',
      module_size: 6
    )
  end
end
```

## ขั้นตอนที่ 943: JWT Authentication Overview

สำหรับ API applications ที่ไม่ใช้ session

```ruby
# Gemfile
gem 'devise'
gem 'devise-jwt'
```

```ruby
# config/initializers/devise.rb
config.jwt do |jwt|
  jwt.secret = ENV['JWT_SECRET_KEY']
  jwt.dispatch_requests = [
    ['POST', %r{^/users/sign_in$}]
  ]
  jwt.revocation_requests = [
    ['DELETE', %r{^/users/sign_out$}]
  ]
  jwt.expiration_time = 1.day.to_i
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  include Devise::JWT::RevocationStrategies::JTIMatcher
  
  devise :database_authenticatable,
         :jwt_authenticatable,
         jwt_revocation_strategy: self
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  before_action :authenticate_user!
  
  respond_to :json
  
  def respond_with(resource, _opts = {})
    render json: resource
  end
  
  def respond_to_on_destroy
    head :no_content
  end
end
```

```ruby
# ตัวอย่าง JWT Request
# POST /users/sign_in
# Body: { user: { email: "test@example.com", password: "password" } }
# Response Header: Authorization: Bearer <jwt_token>

# Protected endpoint
# GET /api/v1/profile
# Header: Authorization: Bearer <jwt_token>
```

## ขั้นตอนที่ 944: Customizing Authentication Views with Bootstrap

```erb
<%# app/views/devise/sessions/new.html.erb (Bootstrap) %>
<div class="row justify-content-center mt-5">
  <div class="col-md-6">
    <div class="card shadow">
      <div class="card-header bg-primary text-white text-center">
        <h4 class="mb-0">เข้าสู่ระบบ</h4>
      </div>
      <div class="card-body p-4">
        <%= form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
          <%= render "devise/shared/error_messages", resource: resource %>
          
          <div class="mb-3">
            <%= f.label :email, "อีเมล", class: "form-label" %>
            <div class="input-group">
              <span class="input-group-text"><i class="bi bi-envelope"></i></span>
              <%= f.email_field :email, 
                  autofocus: true, 
                  autocomplete: "email",
                  class: "form-control",
                  placeholder: "example@email.com" %>
            </div>
          </div>
          
          <div class="mb-3">
            <%= f.label :password, "รหัสผ่าน", class: "form-label" %>
            <div class="input-group">
              <span class="input-group-text"><i class="bi bi-lock"></i></span>
              <%= f.password_field :password, 
                  autocomplete: "current-password",
                  class: "form-control",
                  placeholder: "กรอกรหัสผ่าน" %>
            </div>
          </div>
          
          <div class="d-flex justify-content-between mb-3">
            <div class="form-check">
              <%= f.check_box :remember_me, class: "form-check-input" %>
              <%= f.label :remember_me, "จดจำฉันไว้", class: "form-check-label" %>
            </div>
            <%= link_to "ลืมรหัสผ่าน?", new_user_password_path %>
          </div>
          
          <%= f.submit "เข้าสู่ระบบ", class: "btn btn-primary w-100 mb-3" %>
          
          <div class="text-center">
            <p class="text-muted">หรือ</p>
            <%= link_to "สมัครสมาชิกใหม่", new_user_registration_path, 
                class: "btn btn-outline-secondary w-100" %>
          </div>
        <% end %>
      </div>
    </div>
  </div>
</div>
```

## ขั้นตอนที่ 945: Devise Trackable

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :trackable  # บันทึกข้อมูล login
  
  # Trackable บันทึก:
  # sign_in_count
  # current_sign_in_at
  # last_sign_in_at
  # current_sign_in_ip
  # last_sign_in_ip
end
```

```ruby
# app/controllers/admin/users_controller.rb
class Admin::UsersController < ApplicationController
  before_action :authenticate_user!
  before_action :require_admin!
  
  def index
    @users = User.order(last_sign_in_at: :desc)
    @total_logins = User.sum(:sign_in_count)
    @active_today = User.where("current_sign_in_at > ?", 24.hours.ago).count
  end
  
  def show
    @user = User.find(params[:id])
    @login_history = LoginHistory.where(user: @user).order(created_at: :desc).limit(20)
  end
  
  private
  
  def require_admin!
    redirect_to root_path, alert: "ไม่มีสิทธิ์เข้าถึง" unless current_user.admin?
  end
end
```

## ขั้นตอนที่ 946: Custom Devise Redirect

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?
  
  protected
  
  def after_sign_in_path_for(resource)
    case resource
    when User
      if resource.admin?
        admin_dashboard_path
      elsif resource.first_login?
        complete_profile_path
      else
        stored_location_for(resource) || user_dashboard_path
      end
    else
      super
    end
  end
  
  def after_sign_out_path_for(resource_or_scope)
    new_user_session_path
  end
  
  def after_sign_up_path_for(resource)
    # หลังสมัครสมาชิก
    welcome_path
  end
  
  def after_update_path_for(resource)
    # หลังอัพเดทโปรไฟล์
    user_path(resource)
  end
end
```

## ขั้นตอนที่ 947: Devise I18n (ภาษาไทย)

```yaml
# config/locales/devise.th.yml
th:
  devise:
    confirmations:
      confirmed: "บัญชีของคุณได้รับการยืนยันแล้ว คุณเข้าสู่ระบบได้แล้ว"
      send_instructions: "คุณจะได้รับอีเมลพร้อมคำแนะนำการยืนยันภายในไม่กี่นาที"
      send_paranoid_instructions: "หากอีเมลของคุณอยู่ในฐานข้อมูล คุณจะได้รับอีเมลพร้อมคำแนะนำ"
    failure:
      already_authenticated: "คุณเข้าสู่ระบบแล้ว"
      inactive: "บัญชีของคุณยังไม่ได้เปิดใช้งาน"
      invalid: "%{authentication_keys} หรือรหัสผ่านไม่ถูกต้อง"
      locked: "บัญชีของคุณถูกล็อค"
      last_attempt: "คุณมีการพยายามอีก 1 ครั้งก่อนที่บัญชีจะถูกล็อค"
      not_found_in_database: "%{authentication_keys} หรือรหัสผ่านไม่ถูกต้อง"
      timeout: "เซสชันของคุณหมดเวลา กรุณาเข้าสู่ระบบใหม่เพื่อดำเนินการต่อ"
      unauthenticated: "คุณต้องเข้าสู่ระบบหรือสมัครสมาชิกก่อน"
      unconfirmed: "คุณต้องยืนยันอีเมลของคุณก่อนดำเนินการต่อ"
    mailer:
      confirmation_instructions:
        subject: "คำแนะนำการยืนยันบัญชี"
      email_changed:
        subject: "อีเมลของคุณเปลี่ยนแปลงแล้ว"
      password_change:
        subject: "รหัสผ่านของคุณเปลี่ยนแปลงแล้ว"
      reset_password_instructions:
        subject: "คำแนะนำการรีเซ็ตรหัสผ่าน"
      unlock_instructions:
        subject: "คำแนะนำการปลดล็อค"
    omniauth_callbacks:
      failure: "ไม่สามารถยืนยันตัวตนจาก %{kind} เพราะ \"%{reason}\""
      success: "เข้าสู่ระบบด้วย %{kind} สำเร็จ"
    passwords:
      no_token: "คุณไม่สามารถเข้าถึงหน้านี้โดยไม่มาจากอีเมลรีเซ็ตรหัสผ่าน"
      send_instructions: "คุณจะได้รับอีเมลพร้อมคำแนะนำภายในไม่กี่นาที"
      send_paranoid_instructions: "หากอีเมลของคุณอยู่ในฐานข้อมูล คุณจะได้รับลิงค์รีเซ็ตรหัสผ่าน"
      updated: "รหัสผ่านของคุณเปลี่ยนแปลงแล้ว คุณเข้าสู่ระบบแล้ว"
      updated_not_active: "รหัสผ่านของคุณเปลี่ยนแปลงแล้ว"
    registrations:
      destroyed: "ลบบัญชีของคุณแล้ว หวังว่าจะพบกันอีก"
      signed_up: "ยินดีต้อนรับ! คุณลงทะเบียนสำเร็จแล้ว"
      signed_up_but_inactive: "คุณลงทะเบียนสำเร็จแล้ว แต่บัญชียังไม่ได้เปิดใช้งาน"
      signed_up_but_locked: "คุณลงทะเบียนสำเร็จแล้ว แต่บัญชีถูกล็อค"
      signed_up_but_unconfirmed: "กรุณาตรวจสอบอีเมลของคุณและคลิกลิงค์ยืนยัน"
      update_needs_confirmation: "อัพเดทสำเร็จ แต่ต้องยืนยันอีเมลใหม่ กรุณาตรวจสอบอีเมล"
      updated: "อัพเดทบัญชีของคุณสำเร็จ"
      updated_but_not_signed_in: "บัญชีของคุณอัพเดทสำเร็จ แต่เนื่องจากการเปลี่ยนรหัสผ่าน คุณต้องเข้าสู่ระบบใหม่"
    sessions:
      signed_in: "เข้าสู่ระบบสำเร็จ"
      signed_out: "ออกจากระบบสำเร็จ"
      already_signed_out: "ออกจากระบบสำเร็จ"
    unlocks:
      send_instructions: "คุณจะได้รับอีเมลพร้อมคำแนะนำการปลดล็อคภายในไม่กี่นาที"
      send_paranoid_instructions: "หากบัญชีของคุณอยู่ในฐานข้อมูล คุณจะได้รับอีเมล"
      unlocked: "บัญชีของคุณปลดล็อคแล้ว คุณเข้าสู่ระบบได้แล้ว"
```

## ขั้นตอนที่ 948: Testing Devise

```ruby
# spec/support/devise.rb (RSpec)
RSpec.configure do |config|
  config.include Devise::Test::ControllerHelpers, type: :controller
  config.include Devise::Test::IntegrationHelpers, type: :request
  config.include Devise::Test::IntegrationHelpers, type: :feature
  
  # Sign in helper
  config.include Module.new {
    def sign_in_as(user)
      sign_in user
    end
  }
end
```

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { "ทดสอบ ผู้ใช้" }
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "Password123!" }
    password_confirmation { "Password123!" }
    confirmed_at { Time.current }
    
    trait :admin do
      role { :admin }
    end
    
    trait :unconfirmed do
      confirmed_at { nil }
    end
    
    trait :locked do
      locked_at { Time.current }
    end
  end
end
```

```ruby
# spec/requests/authentication_spec.rb
require 'rails_helper'

RSpec.describe "Authentication", type: :request do
  let(:user) { create(:user) }
  
  describe "POST /users/sign_in" do
    context "ด้วยข้อมูลที่ถูกต้อง" do
      it "เข้าสู่ระบบสำเร็จ" do
        post user_session_path, params: {
          user: { email: user.email, password: "Password123!" }
        }
        
        expect(response).to redirect_to(root_path)
        follow_redirect!
        expect(response.body).to include("เข้าสู่ระบบสำเร็จ")
      end
    end
    
    context "ด้วยรหัสผ่านที่ผิด" do
      it "ล้มเหลวและแสดง error" do
        post user_session_path, params: {
          user: { email: user.email, password: "wrongpassword" }
        }
        
        expect(response).to have_http_status(:unprocessable_entity)
        expect(response.body).to include("ไม่ถูกต้อง")
      end
    end
  end
  
  describe "DELETE /users/sign_out" do
    before { sign_in user }
    
    it "ออกจากระบบสำเร็จ" do
      delete destroy_user_session_path
      expect(response).to redirect_to(root_path)
    end
  end
end
```

## ขั้นตอนที่ 949: API Authentication with Devise Token Auth

```ruby
# Gemfile
gem 'devise_token_auth'
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      mount_devise_token_auth_for 'User', at: 'auth'
      resources :posts
    end
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  include DeviseTokenAuth::Concerns::User
  
  devise :database_authenticatable,
         :registerable,
         :recoverable,
         :validatable
end
```

```ruby
# app/controllers/api/v1/base_controller.rb
class Api::V1::BaseController < ActionController::API
  include DeviseTokenAuth::Concerns::SetUserByToken
  
  before_action :authenticate_user!
  
  rescue_from ActiveRecord::RecordNotFound do |e|
    render json: { error: "ไม่พบข้อมูล" }, status: :not_found
  end
end
```

## ขั้นตอนที่ 950: Advanced Devise Configuration

```ruby
# config/initializers/devise.rb - การตั้งค่าขั้นสูง
Devise.setup do |config|
  # Mailer
  config.mailer_sender = 'noreply@myapp.com'
  config.mailer = 'Devise::Mailer'
  config.parent_mailer = 'ActionMailer::Base'
  
  # ORM
  config.orm = :active_record
  
  # Encryption
  config.encryptor = :bcrypt
  config.pepper = ENV['DEVISE_PEPPER']
  config.stretches = Rails.env.test? ? 1 : 12
  
  # Confirmable
  config.allow_unconfirmed_access_for = 2.days
  config.confirm_within = 3.days
  config.confirmation_keys = [:email]
  config.reconfirmable = true
  
  # Rememberable
  config.remember_for = 4.weeks
  config.extend_remember_period = false
  config.rememberable_options = {}
  
  # Validatable
  config.password_length = 8..128
  config.email_regexp = /\A[^@\s]+@[^@\s]+\z/
  
  # Timeout
  config.timeout_in = 30.minutes
  
  # Lockable
  config.unlock_keys = [:email]
  config.unlock_strategy = :both
  config.maximum_attempts = 5
  config.unlock_in = 1.hour
  config.last_attempt_warning = true
  
  # Recoverable
  config.reset_password_keys = [:email]
  config.reset_password_within = 6.hours
  config.sign_in_after_reset_password = true
  
  # Omniauthable
  # config.omniauth_path_prefix = '/my_engine/users/auth'
  
  # Token
  config.token_authentication_key = :auth_token
  
  # Navigation
  config.sign_out_via = :delete
  config.sign_in_after_change_password = true
  
  # HTTP Auth
  config.http_authenticatable = false
  config.params_authenticatable = true
  config.http_authenticatable_on_xhr = true
  config.http_auth_salt = 'z6Dd@HFpeoFbARfT2rOI'
  
  # Paranoid mode
  config.paranoid = false  # ซ่อน email enumeration
  
  # Clean token on request
  config.clean_up_csrf_token_on_authentication = true
  config.clean_up_request_credentials_on_sign_out = true
end
```

---

## แบบฝึกหัด: Authentication with Devise (25 ข้อ)

### ข้อที่ 1: ติดตั้ง Devise
```
ติดตั้ง Devise ใน Rails 7 application ใหม่
- ติดตั้ง gem
- รัน installer
- สร้าง User model
- ตั้งค่า routes
```

**เฉลย:**
```bash
# Gemfile
gem 'devise'

bundle install
rails generate devise:install
rails generate devise User
rails db:migrate
```

### ข้อที่ 2: เพิ่ม Name Field
```
เพิ่ม name field ให้ User model และอนุญาตให้ส่งผ่าน form
```

**เฉลย:**
```bash
rails generate migration AddNameToUsers name:string
rails db:migrate
```
```ruby
# application_controller.rb
before_action :configure_permitted_parameters, if: :devise_controller?

protected
def configure_permitted_parameters
  devise_parameter_sanitizer.permit(:sign_up, keys: [:name])
  devise_parameter_sanitizer.permit(:account_update, keys: [:name])
end
```

### ข้อที่ 3: Custom Redirect หลัง Login
```
หลัง login ให้ redirect ไปหน้า dashboard
ถ้าเป็น admin ให้ไปหน้า admin panel
```

**เฉลย:**
```ruby
def after_sign_in_path_for(resource)
  resource.admin? ? admin_root_path : dashboard_path
end
```

### ข้อที่ 4: สร้าง Custom Login View
```
สร้าง login form ที่มีรูปแบบ Bootstrap สวยงาม
```

**เฉลย:** (ดูตัวอย่างด้านบน Bootstrap view)

### ข้อที่ 5: Email Confirmation
```
เปิดใช้งาน :confirmable module
ตั้งค่าให้ยืนยันภายใน 24 ชั่วโมง
```

**เฉลย:**
```ruby
# user.rb
devise :confirmable

# devise.rb
config.confirm_within = 24.hours
config.allow_unconfirmed_access_for = 0
```

### ข้อที่ 6: Password Reset
```
ทดสอบ password reset flow:
1. Request reset
2. ได้รับ email
3. คลิกลิงค์
4. ตั้งรหัสผ่านใหม่
```

### ข้อที่ 7: Remember Me Cookie
```
ตั้งค่า Remember Me ให้อยู่ได้ 30 วัน
```

**เฉลย:**
```ruby
# devise.rb
config.remember_for = 30.days
config.extend_remember_period = true
```

### ข้อที่ 8: Google OAuth
```
เพิ่ม Google OAuth login ด้วย omniauth-google-oauth2
```

### ข้อที่ 9: Account Lockout
```
ล็อคบัญชีหลัง login ผิด 3 ครั้ง
ส่ง email เพื่อ unlock
```

**เฉลย:**
```ruby
# user.rb
devise :lockable

# devise.rb
config.maximum_attempts = 3
config.unlock_strategy = :email
```

### ข้อที่ 10: Session Timeout
```
ตั้งค่า session timeout 15 นาที
ผู้ใช้ admin ให้ timeout 5 นาที
```

**เฉลย:**
```ruby
# user.rb
devise :timeoutable

def timeout_in
  admin? ? 5.minutes : 15.minutes
end
```

### ข้อที่ 11: Custom Error Messages
```
แปล error messages เป็นภาษาไทย
```

### ข้อที่ 12: Testing with RSpec
```
เขียน tests:
1. ลงทะเบียนสำเร็จ
2. login สำเร็จ
3. logout สำเร็จ
4. password reset
```

### ข้อที่ 13: Profile Edit Form
```
สร้างหน้า edit profile ที่ผู้ใช้สามารถอัพเดท
ชื่อ, email, รหัสผ่าน
```

**เฉลย:**
```ruby
# Custom registrations controller
class Users::RegistrationsController < Devise::RegistrationsController
  def update
    @user = current_user
    
    if params[:user][:password].blank?
      params[:user].delete(:password)
      params[:user].delete(:password_confirmation)
      params[:user].delete(:current_password)
      
      @user.update_without_password(account_update_params)
    else
      @user.update_with_password(account_update_params)
    end
    
    if @user.errors.empty?
      bypass_sign_in(@user)
      redirect_to root_path, notice: "อัพเดทโปรไฟล์แล้ว"
    else
      render :edit
    end
  end
end
```

### ข้อที่ 14-25 (แบบสรุป)

**ข้อ 14:** สร้าง admin-only area ด้วย before_action :authenticate_user! และ require_admin!

**ข้อ 15:** เพิ่ม avatar upload ใน user registration

**ข้อ 16:** ใช้ devise_scope เพื่อกำหนด custom routes

**ข้อ 17:** สร้าง API endpoint ที่ใช้ JWT authentication

**ข้อ 18:** ทดสอบ email confirmation flow

**ข้อ 19:** เพิ่ม phone number validation

**ข้อ 20:** สร้าง impersonation feature (admin แทน user)

**ข้อ 21:** เพิ่ม login history tracking

**ข้อ 22:** สร้าง multi-tenant authentication

**ข้อ 23:** เพิ่ม password strength indicator

**ข้อ 24:** สร้าง inactive account cleanup rake task

**ข้อ 25:** เพิ่ม CAPTCHA ใน registration form

```ruby
# ข้อ 20: Impersonation
class AdminController < ApplicationController
  def impersonate
    user = User.find(params[:id])
    session[:admin_id] = current_user.id
    sign_in(:user, user)
    redirect_to root_path, notice: "กำลัง impersonate #{user.name}"
  end
  
  def stop_impersonating
    admin = User.find(session[:admin_id])
    session.delete(:admin_id)
    sign_in(:user, admin)
    redirect_to admin_users_path
  end
end
```

---

## สรุป: Authentication with Devise

| Feature | Module | คำอธิบาย |
|---------|--------|----------|
| Login/Logout | database_authenticatable | พื้นฐาน |
| สมัครสมาชิก | registerable | ลงทะเบียน |
| รีเซ็ตรหัสผ่าน | recoverable | ลืมรหัสผ่าน |
| จดจำผู้ใช้ | rememberable | Remember Me |
| ยืนยัน email | confirmable | Email confirmation |
| ล็อคบัญชี | lockable | Login protection |
| หมดเวลา | timeoutable | Session timeout |
| บันทึกการ login | trackable | Analytics |
| Social Login | omniauthable | OAuth |

**Key Takeaways:**
1. Devise เป็น solution ที่ครบครันและ proven สำหรับ Rails authentication
2. ใช้ modules ที่จำเป็นเท่านั้น ไม่ต้องเปิดทุก module
3. Customize views ให้ตรงกับ design ของ application
4. ตั้งค่า security options ให้เหมาะสม
5. เขียน tests ครอบคลุม authentication flows ทั้งหมด
