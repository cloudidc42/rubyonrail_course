# Part 66: Service Objects และ Business Logic

## Steps 1431-1450

---

## Step 1431: Service Objects คืออะไร?

Service Objects เป็น Plain Old Ruby Objects (PORO) ที่ใช้เก็บ business logic โดยเฉพาะ แนวคิดหลักคือการแยก logic ออกจาก Controller และ Model เพื่อให้โค้ดอ่านง่าย ทดสอบง่าย และบำรุงรักษาง่าย

### ปัญหาที่เกิดขึ้นเมื่อไม่ใช้ Service Objects

```ruby
# ❌ Fat Controller - โค้ดรกและยากต่อการทดสอบ
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    
    if @user.save
      # ส่ง email ยืนยัน
      UserMailer.confirmation_email(@user).deliver_later
      
      # สร้าง profile
      Profile.create!(user: @user, avatar_url: default_avatar)
      
      # บันทึก log
      AuditLog.create!(
        action: 'user_registered',
        user_id: @user.id,
        ip_address: request.remote_ip
      )
      
      # ให้ credits เริ่มต้น
      @user.credits.create!(amount: 100, reason: 'welcome_bonus')
      
      # ส่งข้อความต้อนรับ
      Notification.create!(
        user: @user,
        title: 'ยินดีต้อนรับ!',
        body: "สวัสดี #{@user.name} ยินดีต้อนรับสู่แอปของเรา"
      )
      
      redirect_to dashboard_path, notice: 'สมัครสมาชิกสำเร็จ!'
    else
      render :new
    end
  end
end
```

### วิธีแก้ปัญหาด้วย Service Objects

```ruby
# ✅ Thin Controller with Service Object
class UsersController < ApplicationController
  def create
    result = UserRegistration.call(
      params: user_params,
      ip_address: request.remote_ip
    )
    
    if result.success?
      redirect_to dashboard_path, notice: 'สมัครสมาชิกสำเร็จ!'
    else
      @user = result.user
      @errors = result.errors
      render :new
    end
  end
end
```

---

## Step 1432: Skinny Controllers, Fat Models → Service Objects

### วิวัฒนาการของ Rails Architecture

**ยุคแรก: Fat Controllers**
```
Controller → ทำทุกอย่าง (logic, DB, email, etc.)
```

**ยุคกลาง: Fat Models (ตามคำแนะนำของ Rails เดิม)**
```
Controller → บางเบา
Model → ทำทุกอย่าง (callbacks, validations, business logic)
```

**ยุคปัจจุบัน: Service Objects**
```
Controller → รับ request, ส่ง response
Model → validations, associations, scopes
Service Object → business logic
```

### ตัวอย่าง Fat Model (ปัญหา)

```ruby
# ❌ Fat Model - มี callback รก
class User < ApplicationRecord
  after_create :send_confirmation_email
  after_create :create_default_profile
  after_create :grant_welcome_credits
  after_create :log_registration
  after_create :send_welcome_notification
  after_update :send_profile_update_email, if: :email_changed?
  after_update :log_profile_change
  
  def send_confirmation_email
    UserMailer.confirmation_email(self).deliver_later
  end
  
  def create_default_profile
    Profile.create!(user: self)
  end
  
  # ... อีกเยอะ
end
```

**ปัญหาของ Fat Model:**
1. Callbacks ทำงานทุกครั้งที่ save ไม่ว่าจะต้องการหรือไม่
2. ยากต่อการทดสอบ
3. เมื่อต้องการ import users จาก CSV callbacks ก็ยังทำงาน
4. Coupling สูงระหว่าง concerns ต่างๆ

---

## Step 1433: Service Object Patterns

### Pattern 1: `.call` Method

```ruby
# app/services/user_registration.rb
class UserRegistration
  # Class method เรียกใช้ง่าย
  def self.call(**args)
    new(**args).call
  end
  
  def initialize(params:, ip_address: nil)
    @params = params
    @ip_address = ip_address
  end
  
  def call
    create_user
  end
  
  private
  
  def create_user
    User.create!(@params)
  end
end

# การใช้งาน
result = UserRegistration.call(
  params: { name: 'สมชาย', email: 'somchai@example.com' },
  ip_address: '127.0.0.1'
)
```

### Pattern 2: `.perform` Method (คล้าย ActiveJob)

```ruby
# app/services/payment_processor.rb
class PaymentProcessor
  def self.perform(order:, payment_method:)
    new(order: order, payment_method: payment_method).perform
  end
  
  def initialize(order:, payment_method:)
    @order = order
    @payment_method = payment_method
  end
  
  def perform
    charge_payment
    update_order_status
    send_receipt
  end
  
  private
  
  attr_reader :order, :payment_method
  
  def charge_payment
    # ติดต่อ payment gateway
    Stripe::Charge.create(
      amount: order.total_cents,
      currency: 'thb',
      source: payment_method.stripe_token
    )
  end
  
  def update_order_status
    order.update!(status: 'paid', paid_at: Time.current)
  end
  
  def send_receipt
    OrderMailer.receipt(order).deliver_later
  end
end
```

### Pattern 3: `.run` Method

```ruby
# app/services/report_generator.rb
class ReportGenerator
  attr_reader :report, :errors
  
  def self.run(**args)
    service = new(**args)
    service.run
    service
  end
  
  def initialize(user:, date_range:, format: :pdf)
    @user = user
    @date_range = date_range
    @format = format
    @errors = []
  end
  
  def run
    validate_inputs
    return self if errors.any?
    
    @report = generate_report
  end
  
  def success?
    errors.empty?
  end
  
  private
  
  def validate_inputs
    @errors << 'User is required' unless @user
    @errors << 'Date range is required' unless @date_range
  end
  
  def generate_report
    # สร้าง report
    Report.new(
      user: @user,
      data: fetch_data,
      format: @format
    )
  end
  
  def fetch_data
    Order.where(user: @user)
         .where(created_at: @date_range)
         .includes(:items)
  end
end
```

---

## Step 1434: Result Objects

Result Objects คือ objects ที่ return กลับมาจาก Service Objects เพื่อบอกว่า operation สำเร็จหรือไม่

### Simple Result Object

```ruby
# app/services/result.rb
class Result
  attr_reader :value, :error
  
  def initialize(success:, value: nil, error: nil)
    @success = success
    @value = value
    @error = error
  end
  
  def self.success(value = nil)
    new(success: true, value: value)
  end
  
  def self.failure(error)
    new(success: false, error: error)
  end
  
  def success?
    @success
  end
  
  def failure?
    !@success
  end
  
  # Pattern matching support (Ruby 3+)
  def deconstruct_keys(keys)
    { success: success?, value: value, error: error }
  end
end
```

### การใช้งาน Result Object

```ruby
# app/services/user_registration.rb
class UserRegistration
  def self.call(**args)
    new(**args).call
  end
  
  def initialize(params:, ip_address: nil)
    @params = params
    @ip_address = ip_address
  end
  
  def call
    user = User.new(@params)
    
    if user.save
      send_confirmation_email(user)
      create_profile(user)
      log_registration(user)
      
      Result.success(user)
    else
      Result.failure(user.errors.full_messages)
    end
  rescue StandardError => e
    Rails.logger.error "UserRegistration failed: #{e.message}"
    Result.failure(['An unexpected error occurred'])
  end
  
  private
  
  def send_confirmation_email(user)
    UserMailer.confirmation_email(user).deliver_later
  end
  
  def create_profile(user)
    Profile.create!(user: user)
  end
  
  def log_registration(user)
    AuditLog.create!(
      action: 'user_registered',
      resource_type: 'User',
      resource_id: user.id,
      metadata: { ip_address: @ip_address }
    )
  end
end

# ใน Controller
class UsersController < ApplicationController
  def create
    result = UserRegistration.call(
      params: user_params,
      ip_address: request.remote_ip
    )
    
    if result.success?
      @user = result.value
      redirect_to dashboard_path, notice: 'สมัครสมาชิกสำเร็จ!'
    else
      flash.now[:alert] = result.error.join(', ')
      render :new
    end
  end
  
  private
  
  def user_params
    params.require(:user).permit(:name, :email, :password, :password_confirmation)
  end
end
```

### Advanced Result Object with Multiple Values

```ruby
# app/services/result.rb
class Result
  attr_reader :errors
  
  def initialize(success:, **attributes)
    @success = success
    @errors = []
    
    attributes.each do |key, value|
      instance_variable_set("@#{key}", value)
      self.class.attr_reader key
    end
  end
  
  def self.success(**attributes)
    new(success: true, **attributes)
  end
  
  def self.failure(*errors, **attributes)
    result = new(success: false, **attributes)
    result.instance_variable_set(:@errors, Array(errors).flatten)
    result
  end
  
  def success?
    @success
  end
  
  def failure?
    !@success
  end
end

# ใช้งาน
result = Result.success(user: user, token: 'abc123')
result.user  # => #<User...>
result.token # => 'abc123'

result = Result.failure('Email already taken', 'Password too short', user: user)
result.errors  # => ['Email already taken', 'Password too short']
result.user    # => #<User...> (ยังส่งกลับ user สำหรับ re-render form)
```

---

## Step 1435: Interactor Gem

`interactor` gem เป็น library ยอดนิยมสำหรับสร้าง Service Objects

### การติดตั้ง

```ruby
# Gemfile
gem 'interactor'
```

```bash
bundle install
```

### สร้าง Interactor แรก

```ruby
# app/interactors/authenticate_user.rb
class AuthenticateUser
  include Interactor
  
  def call
    user = User.find_by(email: context.email)
    
    if user&.authenticate(context.password)
      context.user = user
      context.token = JsonWebToken.encode(user_id: user.id)
    else
      context.fail!(message: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง')
    end
  end
end

# การใช้งาน
result = AuthenticateUser.call(
  email: 'user@example.com',
  password: 'password123'
)

if result.success?
  puts result.user.name
  puts result.token
else
  puts result.message
end
```

### Organizer - รวม Interactors หลายตัว

```ruby
# app/interactors/register_user.rb
class RegisterUser
  include Interactor::Organizer
  
  organize CreateUser,
           CreateProfile,
           SendWelcomeEmail,
           GrantWelcomeCredits
end

# app/interactors/create_user.rb
class CreateUser
  include Interactor
  
  def call
    user = User.create(
      name: context.name,
      email: context.email,
      password: context.password
    )
    
    if user.persisted?
      context.user = user
    else
      context.fail!(
        message: 'ไม่สามารถสร้าง user ได้',
        errors: user.errors.full_messages
      )
    end
  end
end

# app/interactors/create_profile.rb
class CreateProfile
  include Interactor
  
  def call
    profile = Profile.create!(
      user: context.user,
      display_name: context.user.name
    )
    
    context.profile = profile
  rescue ActiveRecord::RecordInvalid => e
    context.fail!(message: e.message)
  end
end

# app/interactors/send_welcome_email.rb
class SendWelcomeEmail
  include Interactor
  
  def call
    UserMailer.welcome(context.user).deliver_later
  end
end

# app/interactors/grant_welcome_credits.rb
class GrantWelcomeCredits
  include Interactor
  
  def call
    context.user.credits.create!(
      amount: 100,
      reason: 'welcome_bonus',
      description: 'Credit ต้อนรับสมาชิกใหม่'
    )
  end
end
```

### Rollback ใน Interactor

```ruby
# app/interactors/charge_order.rb
class ChargeOrder
  include Interactor
  
  def call
    charge = Stripe::Charge.create(
      amount: context.order.total_cents,
      currency: 'thb',
      source: context.payment_token,
      description: "Order #{context.order.number}"
    )
    
    context.stripe_charge_id = charge.id
    context.order.update!(
      status: 'paid',
      stripe_charge_id: charge.id
    )
  rescue Stripe::CardError => e
    context.fail!(message: e.message)
  end
  
  # เรียกเมื่อ Organizer rollback
  def rollback
    if context.stripe_charge_id
      Stripe::Refund.create(charge: context.stripe_charge_id)
      Rails.logger.info "Refunded charge #{context.stripe_charge_id}"
    end
  end
end
```

---

## Step 1436: Trailblazer Gem Overview

Trailblazer เป็น framework ที่ซับซ้อนกว่า Interactor แต่มีความสามารถมากกว่ามาก

### การติดตั้ง

```ruby
# Gemfile
gem 'trailblazer'
gem 'trailblazer-rails'
```

### Operation ใน Trailblazer

```ruby
# app/concepts/user/operation/create.rb
module User
  module Operation
    class Create < Trailblazer::Operation
      step :validate_params
      step :create_user
      step :send_email
      fail :handle_errors
      
      def validate_params(ctx, params:, **)
        ctx[:user] = User.new(params[:user])
        ctx[:user].valid?
      end
      
      def create_user(ctx, user:, **)
        user.save
      end
      
      def send_email(ctx, user:, **)
        UserMailer.welcome(user).deliver_later
        true
      end
      
      def handle_errors(ctx, user:, **)
        ctx[:errors] = user.errors.full_messages
      end
    end
  end
end

# การใช้งาน
result = User::Operation::Create.call(params: { user: { name: 'สมชาย', email: 'somchai@example.com' } })

if result.success?
  user = result[:user]
  puts "สร้าง user สำเร็จ: #{user.name}"
else
  puts result[:errors].inspect
end
```

---

## Step 1437: Form Objects

Form Objects ใช้สำหรับ validations ที่ซับซ้อนและ forms ที่ไม่ตรงกับ model โดยตรง

### ปัญหาที่ Form Objects แก้ไข

```ruby
# ❌ ปัญหา: form สมัครสมาชิกมีหลาย model
# User model + Profile model + Settings model
# แต่ต้องการ validate ทั้งหมดใน form เดียว
```

### สร้าง Form Object

```ruby
# app/forms/registration_form.rb
class RegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes
  
  # User attributes
  attribute :email, :string
  attribute :password, :string
  attribute :password_confirmation, :string
  
  # Profile attributes
  attribute :first_name, :string
  attribute :last_name, :string
  attribute :phone, :string
  attribute :birth_date, :date
  
  # Terms
  attribute :accept_terms, :boolean, default: false
  
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :password_confirmation, presence: true
  validates :first_name, presence: true, length: { maximum: 50 }
  validates :last_name, presence: true, length: { maximum: 50 }
  validates :phone, format: { with: /\A\d{10}\z/, message: 'ต้องเป็นตัวเลข 10 หลัก' }, allow_blank: true
  validates :accept_terms, acceptance: { message: 'ต้องยอมรับเงื่อนไขการใช้งาน' }
  validate :passwords_match
  validate :email_uniqueness
  
  def save
    return false unless valid?
    
    ActiveRecord::Base.transaction do
      user = User.create!(
        email: email,
        password: password,
        password_confirmation: password_confirmation
      )
      
      user.create_profile!(
        first_name: first_name,
        last_name: last_name,
        phone: phone,
        birth_date: birth_date
      )
      
      @persisted_user = user
    end
    
    true
  rescue ActiveRecord::RecordInvalid => e
    errors.add(:base, e.message)
    false
  end
  
  def user
    @persisted_user
  end
  
  private
  
  def passwords_match
    if password != password_confirmation
      errors.add(:password_confirmation, 'ไม่ตรงกัน')
    end
  end
  
  def email_uniqueness
    if User.exists?(email: email)
      errors.add(:email, 'ถูกใช้งานแล้ว')
    end
  end
end

# ใน Controller
class RegistrationsController < ApplicationController
  def new
    @form = RegistrationForm.new
  end
  
  def create
    @form = RegistrationForm.new(registration_params)
    
    if @form.save
      sign_in @form.user
      redirect_to dashboard_path, notice: 'สมัครสมาชิกสำเร็จ!'
    else
      render :new
    end
  end
  
  private
  
  def registration_params
    params.require(:registration_form).permit(
      :email, :password, :password_confirmation,
      :first_name, :last_name, :phone, :birth_date,
      :accept_terms
    )
  end
end
```

### Form Object ใน View

```erb
<%# app/views/registrations/new.html.erb %>
<h1>สมัครสมาชิก</h1>

<%= form_with model: @form, url: registrations_path do |f| %>
  <% if @form.errors.any? %>
    <div class="alert alert-danger">
      <ul>
        <% @form.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
        <% end %>
      </ul>
    </div>
  <% end %>
  
  <div class="form-group">
    <%= f.label :email, 'อีเมล' %>
    <%= f.email_field :email, class: 'form-control' %>
  </div>
  
  <div class="form-group">
    <%= f.label :first_name, 'ชื่อ' %>
    <%= f.text_field :first_name, class: 'form-control' %>
  </div>
  
  <div class="form-group">
    <%= f.label :last_name, 'นามสกุล' %>
    <%= f.text_field :last_name, class: 'form-control' %>
  </div>
  
  <div class="form-group">
    <%= f.check_box :accept_terms %>
    <%= f.label :accept_terms, 'ฉันยอมรับเงื่อนไขการใช้งาน' %>
  </div>
  
  <%= f.submit 'สมัครสมาชิก', class: 'btn btn-primary' %>
<% end %>
```

---

## Step 1438: Query Objects

Query Objects ใช้สำหรับ encapsulate complex database queries

### ตัวอย่าง Query Object

```ruby
# app/queries/user_search_query.rb
class UserSearchQuery
  def initialize(relation = User.all)
    @relation = relation
  end
  
  def call(params = {})
    result = @relation
    result = filter_by_name(result, params[:name]) if params[:name].present?
    result = filter_by_email(result, params[:email]) if params[:email].present?
    result = filter_by_role(result, params[:role]) if params[:role].present?
    result = filter_by_status(result, params[:status]) if params[:status].present?
    result = filter_by_date_range(result, params[:from], params[:to])
    result = sort_results(result, params[:sort], params[:direction])
    result
  end
  
  private
  
  def filter_by_name(relation, name)
    relation.where('name ILIKE ?', "%#{name}%")
  end
  
  def filter_by_email(relation, email)
    relation.where('email ILIKE ?', "%#{email}%")
  end
  
  def filter_by_role(relation, role)
    relation.where(role: role)
  end
  
  def filter_by_status(relation, status)
    case status
    when 'active'
      relation.where(active: true)
    when 'inactive'
      relation.where(active: false)
    when 'banned'
      relation.where.not(banned_at: nil)
    else
      relation
    end
  end
  
  def filter_by_date_range(relation, from, to)
    return relation unless from.present? || to.present?
    
    relation.where(created_at: parse_date(from)..parse_date(to))
  end
  
  def sort_results(relation, sort, direction)
    allowed_sorts = %w[name email created_at last_sign_in_at]
    allowed_directions = %w[asc desc]
    
    sort = 'created_at' unless allowed_sorts.include?(sort)
    direction = 'desc' unless allowed_directions.include?(direction)
    
    relation.order("#{sort} #{direction}")
  end
  
  def parse_date(date_string)
    return nil unless date_string.present?
    Date.parse(date_string)
  rescue Date::Error
    nil
  end
end

# ใน Controller
class Admin::UsersController < ApplicationController
  def index
    @users = UserSearchQuery.new.call(search_params).page(params[:page])
  end
  
  private
  
  def search_params
    params.permit(:name, :email, :role, :status, :from, :to, :sort, :direction)
  end
end
```

### Composable Query Objects

```ruby
# app/queries/active_users_query.rb
class ActiveUsersQuery
  def self.call(relation = User.all)
    relation.where(active: true)
            .where('last_sign_in_at > ?', 30.days.ago)
  end
end

# app/queries/premium_users_query.rb
class PremiumUsersQuery
  def self.call(relation = User.all)
    relation.joins(:subscription)
            .where(subscriptions: { status: 'active', plan: 'premium' })
  end
end

# การ compose queries
active_premium_users = PremiumUsersQuery.call(
  ActiveUsersQuery.call
)
```

---

## Step 1439: Command Pattern

Command Pattern ใช้สำหรับ encapsulate การกระทำที่สามารถ undo ได้

```ruby
# app/commands/base_command.rb
class BaseCommand
  attr_reader :errors
  
  def initialize
    @errors = []
    @executed = false
  end
  
  def execute
    raise NotImplementedError, "#{self.class}#execute must be implemented"
  end
  
  def undo
    raise NotImplementedError, "#{self.class}#undo must be implemented"
  end
  
  def executed?
    @executed
  end
  
  def success?
    executed? && errors.empty?
  end
  
  protected
  
  def mark_executed
    @executed = true
  end
end

# app/commands/update_user_role_command.rb
class UpdateUserRoleCommand < BaseCommand
  def initialize(user:, new_role:, performed_by:)
    super()
    @user = user
    @new_role = new_role
    @old_role = user.role
    @performed_by = performed_by
  end
  
  def execute
    return false unless valid?
    
    ActiveRecord::Base.transaction do
      @user.update!(role: @new_role)
      log_change
      mark_executed
    end
    
    true
  rescue ActiveRecord::RecordInvalid => e
    @errors << e.message
    false
  end
  
  def undo
    return false unless executed?
    
    ActiveRecord::Base.transaction do
      @user.update!(role: @old_role)
      log_undo
      @executed = false
    end
    
    true
  end
  
  private
  
  def valid?
    unless User.valid_roles.include?(@new_role)
      @errors << "Invalid role: #{@new_role}"
    end
    
    if @user == @performed_by && @new_role != 'admin'
      @errors << 'Cannot demote yourself'
    end
    
    @errors.empty?
  end
  
  def log_change
    AuditLog.create!(
      action: 'role_changed',
      user: @performed_by,
      resource: @user,
      metadata: { from: @old_role, to: @new_role }
    )
  end
  
  def log_undo
    AuditLog.create!(
      action: 'role_change_undone',
      user: @performed_by,
      resource: @user,
      metadata: { from: @new_role, to: @old_role }
    )
  end
end

# การใช้งาน
command = UpdateUserRoleCommand.new(
  user: target_user,
  new_role: 'moderator',
  performed_by: current_user
)

if command.execute
  flash[:notice] = 'เปลี่ยน role สำเร็จ'
else
  flash[:alert] = command.errors.join(', ')
end

# ต่อมาต้องการ undo
if command.executed?
  command.undo
end
```

---

## Step 1440: Real Example - UserRegistration Service

```ruby
# app/services/user_registration.rb
class UserRegistration
  attr_reader :user, :errors
  
  def self.call(**args)
    service = new(**args)
    service.call
    service
  end
  
  def initialize(
    name:,
    email:,
    password:,
    password_confirmation:,
    role: 'user',
    ip_address: nil,
    referral_code: nil
  )
    @name = name
    @email = email
    @password = password
    @password_confirmation = password_confirmation
    @role = role
    @ip_address = ip_address
    @referral_code = referral_code
    @errors = []
  end
  
  def call
    validate_inputs
    return self if @errors.any?
    
    ActiveRecord::Base.transaction do
      @user = create_user
      create_profile
      handle_referral if @referral_code.present?
      send_welcome_email
      log_registration
    end
    
    self
  rescue ActiveRecord::RecordInvalid => e
    @errors << e.message
    self
  rescue StandardError => e
    Rails.logger.error "UserRegistration error: #{e.class} - #{e.message}"
    Rails.logger.error e.backtrace.first(5).join("\n")
    @errors << 'เกิดข้อผิดพลาดที่ไม่คาดคิด กรุณาลองใหม่อีกครั้ง'
    self
  end
  
  def success?
    @errors.empty? && @user&.persisted?
  end
  
  def failure?
    !success?
  end
  
  private
  
  def validate_inputs
    @errors << 'ชื่อจำเป็นต้องกรอก' if @name.blank?
    @errors << 'อีเมลจำเป็นต้องกรอก' if @email.blank?
    @errors << 'รหัสผ่านจำเป็นต้องกรอก' if @password.blank?
    
    if @password.present? && @password.length < 8
      @errors << 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร'
    end
    
    if @password != @password_confirmation
      @errors << 'รหัสผ่านไม่ตรงกัน'
    end
    
    if User.exists?(email: @email)
      @errors << 'อีเมลนี้ถูกใช้งานแล้ว'
    end
  end
  
  def create_user
    User.create!(
      name: @name,
      email: @email,
      password: @password,
      password_confirmation: @password_confirmation,
      role: @role
    )
  end
  
  def create_profile
    @user.create_profile!(
      display_name: @name,
      created_from_ip: @ip_address
    )
  end
  
  def handle_referral
    referrer = User.find_by(referral_code: @referral_code)
    return unless referrer
    
    # ให้ bonus กับคนที่แนะนำ
    referrer.wallet.credit!(
      amount: 50,
      description: "ได้รับ credit จากการแนะนำ #{@user.email}"
    )
    
    # ให้ bonus กับสมาชิกใหม่
    @user.wallet.credit!(
      amount: 20,
      description: "ได้รับ credit จากการใช้ referral code"
    )
    
    Referral.create!(
      referrer: referrer,
      referred: @user,
      referral_code: @referral_code
    )
  end
  
  def send_welcome_email
    UserMailer.welcome(@user).deliver_later
    UserMailer.confirmation(@user).deliver_later
  end
  
  def log_registration
    AuditLog.create!(
      action: 'user_registered',
      resource_type: 'User',
      resource_id: @user.id,
      metadata: {
        ip_address: @ip_address,
        referral_code: @referral_code,
        role: @role
      }
    )
  end
end
```

---

## Step 1441: Real Example - PaymentProcessor Service

```ruby
# app/services/payment_processor.rb
class PaymentProcessor
  attr_reader :payment, :errors
  
  def self.call(**args)
    service = new(**args)
    service.call
    service
  end
  
  def initialize(order:, payment_method_id:, user:)
    @order = order
    @payment_method_id = payment_method_id
    @user = user
    @errors = []
  end
  
  def call
    validate_order
    return self if @errors.any?
    
    ActiveRecord::Base.transaction do
      @payment = create_payment_record
      process_charge
      update_order_status
      send_receipt
      log_payment
    end
    
    self
  rescue Stripe::CardError => e
    handle_card_error(e)
    self
  rescue Stripe::StripeError => e
    handle_stripe_error(e)
    self
  rescue StandardError => e
    handle_generic_error(e)
    self
  end
  
  def success?
    @errors.empty? && @payment&.persisted? && @payment&.completed?
  end
  
  private
  
  def validate_order
    if @order.paid?
      @errors << 'Order นี้ถูกชำระเงินแล้ว'
    end
    
    if @order.cancelled?
      @errors << 'Order นี้ถูกยกเลิกแล้ว'
    end
    
    if @order.total <= 0
      @errors << 'ยอดรวม order ต้องมากกว่า 0'
    end
  end
  
  def create_payment_record
    Payment.create!(
      order: @order,
      user: @user,
      amount: @order.total,
      currency: 'THB',
      status: 'pending'
    )
  end
  
  def process_charge
    stripe_charge = Stripe::Charge.create(
      amount: (@order.total * 100).to_i, # Stripe ใช้สตางค์
      currency: 'thb',
      payment_method: @payment_method_id,
      customer: @user.stripe_customer_id,
      description: "Order #{@order.number}",
      metadata: {
        order_id: @order.id,
        user_id: @user.id
      },
      confirm: true
    )
    
    @payment.update!(
      status: 'completed',
      stripe_charge_id: stripe_charge.id,
      processed_at: Time.current
    )
  end
  
  def update_order_status
    @order.update!(
      status: 'paid',
      paid_at: Time.current,
      payment: @payment
    )
  end
  
  def send_receipt
    OrderMailer.receipt(@order).deliver_later
  end
  
  def log_payment
    AuditLog.create!(
      action: 'payment_processed',
      user: @user,
      resource: @payment,
      metadata: {
        order_id: @order.id,
        amount: @order.total,
        stripe_charge_id: @payment.stripe_charge_id
      }
    )
  end
  
  def handle_card_error(error)
    @payment&.update!(
      status: 'failed',
      failure_reason: error.message
    )
    @errors << error.message
    
    Rails.logger.warn "Card error for order #{@order.id}: #{error.message}"
  end
  
  def handle_stripe_error(error)
    @payment&.update!(
      status: 'failed',
      failure_reason: 'Stripe error'
    )
    @errors << 'เกิดข้อผิดพลาดจาก payment gateway กรุณาลองใหม่'
    
    Rails.logger.error "Stripe error for order #{@order.id}: #{error.message}"
  end
  
  def handle_generic_error(error)
    @payment&.update!(status: 'failed', failure_reason: error.message)
    @errors << 'เกิดข้อผิดพลาดที่ไม่คาดคิด'
    
    Rails.logger.error "Payment error: #{error.class} - #{error.message}"
    Rails.logger.error error.backtrace.first(10).join("\n")
  end
end
```

---

## Step 1442: Real Example - ReportGenerator Service

```ruby
# app/services/report_generator.rb
class ReportGenerator
  attr_reader :report_data, :errors
  
  AVAILABLE_FORMATS = %w[csv xlsx pdf json].freeze
  
  def self.call(**args)
    service = new(**args)
    service.call
    service
  end
  
  def initialize(
    report_type:,
    user:,
    start_date:,
    end_date:,
    format: 'csv',
    filters: {}
  )
    @report_type = report_type
    @user = user
    @start_date = start_date.to_date
    @end_date = end_date.to_date
    @format = format
    @filters = filters
    @errors = []
  end
  
  def call
    validate_params
    return self if @errors.any?
    
    @report_data = case @report_type
    when 'sales'        then generate_sales_report
    when 'users'        then generate_users_report
    when 'inventory'    then generate_inventory_report
    when 'transactions' then generate_transactions_report
    else
      @errors << "ประเภท report ไม่ถูกต้อง: #{@report_type}"
      nil
    end
    
    log_report_generation
    self
  rescue StandardError => e
    @errors << "เกิดข้อผิดพลาดในการสร้าง report: #{e.message}"
    Rails.logger.error "ReportGenerator error: #{e.class} - #{e.message}"
    self
  end
  
  def success?
    @errors.empty? && @report_data.present?
  end
  
  def to_file
    return nil unless success?
    
    case @format
    when 'csv'  then to_csv
    when 'xlsx' then to_xlsx
    when 'pdf'  then to_pdf
    when 'json' then to_json
    end
  end
  
  private
  
  def validate_params
    @errors << 'วันที่เริ่มต้นจำเป็นต้องกรอก' if @start_date.blank?
    @errors << 'วันที่สิ้นสุดจำเป็นต้องกรอก' if @end_date.blank?
    
    if @start_date.present? && @end_date.present? && @start_date > @end_date
      @errors << 'วันที่เริ่มต้นต้องไม่เกินวันที่สิ้นสุด'
    end
    
    unless AVAILABLE_FORMATS.include?(@format)
      @errors << "รูปแบบไฟล์ไม่รองรับ: #{@format}"
    end
    
    if @end_date - @start_date > 365
      @errors << 'ช่วงเวลาต้องไม่เกิน 1 ปี'
    end
  end
  
  def generate_sales_report
    orders = Order
      .paid
      .where(created_at: @start_date.beginning_of_day..@end_date.end_of_day)
      .includes(:user, :items, :payment)
    
    orders = orders.where(user: @filters[:user_id]) if @filters[:user_id]
    orders = orders.where(status: @filters[:status]) if @filters[:status]
    
    {
      title: 'รายงานยอดขาย',
      period: "#{@start_date.strftime('%d/%m/%Y')} - #{@end_date.strftime('%d/%m/%Y')}",
      generated_at: Time.current,
      summary: {
        total_orders: orders.count,
        total_revenue: orders.sum(:total),
        average_order_value: orders.average(:total)&.round(2),
        refunds: orders.where(status: 'refunded').sum(:total)
      },
      records: orders.map do |order|
        {
          order_number: order.number,
          date: order.created_at.strftime('%d/%m/%Y %H:%M'),
          customer: order.user.name,
          items_count: order.items.count,
          total: order.total,
          status: order.status,
          payment_method: order.payment&.method_type
        }
      end
    }
  end
  
  def generate_users_report
    users = User
      .where(created_at: @start_date.beginning_of_day..@end_date.end_of_day)
      .includes(:profile, :orders)
    
    {
      title: 'รายงานสมาชิก',
      period: "#{@start_date.strftime('%d/%m/%Y')} - #{@end_date.strftime('%d/%m/%Y')}",
      generated_at: Time.current,
      summary: {
        new_users: users.count,
        active_users: users.where('last_sign_in_at > ?', 7.days.ago).count,
        users_with_orders: users.joins(:orders).distinct.count
      },
      records: users.map do |user|
        {
          id: user.id,
          name: user.name,
          email: user.email,
          registered_at: user.created_at.strftime('%d/%m/%Y'),
          orders_count: user.orders.count,
          total_spent: user.orders.paid.sum(:total)
        }
      end
    }
  end
  
  def generate_inventory_report
    products = Product
      .includes(:category, :stock_entries)
      .order(:name)
    
    products = products.where(category: @filters[:category_id]) if @filters[:category_id]
    
    {
      title: 'รายงานสินค้าคงคลัง',
      generated_at: Time.current,
      summary: {
        total_products: products.count,
        out_of_stock: products.where(stock: 0).count,
        low_stock: products.where('stock < ?', 10).count,
        total_value: products.sum('stock * price')
      },
      records: products.map do |product|
        {
          id: product.id,
          name: product.name,
          category: product.category&.name,
          sku: product.sku,
          stock: product.stock,
          price: product.price,
          total_value: product.stock * product.price,
          status: stock_status(product.stock)
        }
      end
    }
  end
  
  def generate_transactions_report
    transactions = Transaction
      .where(created_at: @start_date.beginning_of_day..@end_date.end_of_day)
      .includes(:user, :order)
      .order(created_at: :desc)
    
    {
      title: 'รายงานธุรกรรม',
      period: "#{@start_date.strftime('%d/%m/%Y')} - #{@end_date.strftime('%d/%m/%Y')}",
      generated_at: Time.current,
      records: transactions.map do |tx|
        {
          id: tx.id,
          date: tx.created_at.strftime('%d/%m/%Y %H:%M:%S'),
          user: tx.user.name,
          type: tx.transaction_type,
          amount: tx.amount,
          currency: tx.currency,
          status: tx.status,
          reference: tx.reference_number
        }
      end
    }
  end
  
  def stock_status(stock)
    case stock
    when 0       then 'out_of_stock'
    when 1..10   then 'low_stock'
    when 11..50  then 'medium_stock'
    else              'in_stock'
    end
  end
  
  def to_csv
    require 'csv'
    
    CSV.generate(headers: true) do |csv|
      records = @report_data[:records]
      next if records.empty?
      
      csv << records.first.keys.map { |k| k.to_s.humanize }
      records.each { |record| csv << record.values }
    end
  end
  
  def to_json
    @report_data.to_json
  end
  
  def log_report_generation
    return unless success?
    
    AuditLog.create!(
      action: 'report_generated',
      user: @user,
      metadata: {
        report_type: @report_type,
        format: @format,
        start_date: @start_date,
        end_date: @end_date,
        record_count: @report_data[:records]&.count
      }
    )
  end
end
```

---

## Step 1443: ตัวอย่างครบถ้วน - Service Object ใน Controller

```ruby
# app/controllers/api/v1/orders_controller.rb
class Api::V1::OrdersController < ApplicationController
  def create
    result = PlaceOrder.call(
      user: current_user,
      cart_id: params[:cart_id],
      payment_method_id: params[:payment_method_id],
      shipping_address: params[:shipping_address],
      coupon_code: params[:coupon_code]
    )
    
    if result.success?
      render json: {
        order: OrderSerializer.new(result.order).as_json,
        message: 'สั่งซื้อสำเร็จ'
      }, status: :created
    else
      render json: {
        errors: result.errors,
        code: result.error_code
      }, status: :unprocessable_entity
    end
  end
end

# app/services/place_order.rb
class PlaceOrder
  attr_reader :order, :errors, :error_code
  
  def self.call(**args)
    service = new(**args)
    service.call
    service
  end
  
  def initialize(user:, cart_id:, payment_method_id:, shipping_address:, coupon_code: nil)
    @user = user
    @cart_id = cart_id
    @payment_method_id = payment_method_id
    @shipping_address = shipping_address
    @coupon_code = coupon_code
    @errors = []
  end
  
  def call
    load_cart
    return self if @errors.any?
    
    validate_cart
    return self if @errors.any?
    
    apply_coupon if @coupon_code.present?
    
    ActiveRecord::Base.transaction do
      @order = create_order
      process_payment
      update_inventory
      clear_cart
      send_confirmation
    end
    
    self
  rescue ActiveRecord::RecordInvalid => e
    @errors << e.message
    @error_code = 'validation_error'
    self
  rescue Stripe::CardError => e
    @errors << e.message
    @error_code = 'payment_failed'
    self
  end
  
  def success?
    @errors.empty? && @order&.persisted?
  end
  
  private
  
  def load_cart
    @cart = Cart.find_by(id: @cart_id, user: @user)
    
    unless @cart
      @errors << 'ไม่พบตะกร้าสินค้า'
      @error_code = 'cart_not_found'
    end
  end
  
  def validate_cart
    if @cart.items.empty?
      @errors << 'ตะกร้าสินค้าว่างเปล่า'
      @error_code = 'empty_cart'
      return
    end
    
    @cart.items.each do |item|
      if item.product.stock < item.quantity
        @errors << "สินค้า #{item.product.name} มีไม่เพียงพอ"
        @error_code = 'insufficient_stock'
      end
    end
  end
  
  def apply_coupon
    @coupon = Coupon.active.find_by(code: @coupon_code)
    
    if @coupon.nil?
      @errors << 'โค้ดส่วนลดไม่ถูกต้อง'
      @error_code = 'invalid_coupon'
    elsif !@coupon.valid_for?(@user)
      @errors << 'โค้ดส่วนลดไม่สามารถใช้ได้'
      @error_code = 'coupon_not_applicable'
    end
  end
  
  def create_order
    order = Order.create!(
      user: @user,
      items: @cart.items.map { |ci| build_order_item(ci) },
      shipping_address: @shipping_address,
      subtotal: @cart.subtotal,
      discount: @coupon&.calculate_discount(@cart.subtotal) || 0,
      total: calculate_total,
      coupon: @coupon,
      status: 'pending'
    )
    order
  end
  
  def build_order_item(cart_item)
    OrderItem.new(
      product: cart_item.product,
      quantity: cart_item.quantity,
      unit_price: cart_item.product.price,
      total_price: cart_item.quantity * cart_item.product.price
    )
  end
  
  def calculate_total
    subtotal = @cart.subtotal
    discount = @coupon&.calculate_discount(subtotal) || 0
    subtotal - discount
  end
  
  def process_payment
    processor = PaymentProcessor.call(
      order: @order,
      payment_method_id: @payment_method_id,
      user: @user
    )
    
    unless processor.success?
      raise Stripe::CardError.new(processor.errors.first, nil)
    end
  end
  
  def update_inventory
    @order.items.each do |item|
      item.product.decrement!(:stock, item.quantity)
    end
  end
  
  def clear_cart
    @cart.destroy
  end
  
  def send_confirmation
    OrderMailer.confirmation(@order).deliver_later
  end
end
```

---

## Step 1444-1450: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: สร้าง Basic Service Object

สร้าง service object สำหรับลบ user account พร้อม cleanup ข้อมูลที่เกี่ยวข้อง:

```ruby
# app/services/delete_user_account.rb
class DeleteUserAccount
  attr_reader :errors
  
  def self.call(**args)
    service = new(**args)
    service.call
    service
  end
  
  def initialize(user:, performed_by:, reason: nil)
    @user = user
    @performed_by = performed_by
    @reason = reason
    @errors = []
  end
  
  def call
    validate
    return self if @errors.any?
    
    ActiveRecord::Base.transaction do
      cancel_subscriptions
      process_refunds
      anonymize_user_data
      log_deletion
    end
    
    self
  end
  
  def success?
    @errors.empty?
  end
  
  private
  
  def validate
    unless @performed_by.admin? || @performed_by == @user
      @errors << 'ไม่มีสิทธิ์ลบบัญชีนี้'
    end
    
    if @user.has_pending_orders?
      @errors << 'มี orders ที่ยังไม่เสร็จสิ้น'
    end
  end
  
  def cancel_subscriptions
    @user.subscriptions.active.each(&:cancel!)
  end
  
  def process_refunds
    @user.orders.paid.recent(30.days).each do |order|
      RefundOrder.call(order: order, reason: 'account_deletion')
    end
  end
  
  def anonymize_user_data
    @user.update!(
      email: "deleted_#{@user.id}@deleted.example.com",
      name: '[Deleted User]',
      phone: nil,
      deleted_at: Time.current
    )
    @user.profile&.destroy
  end
  
  def log_deletion
    AuditLog.create!(
      action: 'user_account_deleted',
      user: @performed_by,
      metadata: {
        deleted_user_id: @user.id,
        reason: @reason
      }
    )
  end
end
```

### แบบฝึกหัดที่ 2: สร้าง Result Object ที่ complete

```ruby
# สร้าง Result class ที่รองรับ:
# - success/failure state
# - multiple values
# - error messages with codes
# - chaining operations

class ServiceResult
  attr_reader :value, :errors, :error_code, :metadata
  
  def initialize(success:, value: nil, errors: [], error_code: nil, metadata: {})
    @success = success
    @value = value
    @errors = Array(errors)
    @error_code = error_code
    @metadata = metadata
  end
  
  def self.ok(value = nil, metadata: {})
    new(success: true, value: value, metadata: metadata)
  end
  
  def self.err(errors, code: nil, metadata: {})
    new(
      success: false,
      errors: Array(errors),
      error_code: code,
      metadata: metadata
    )
  end
  
  def success? = @success
  def failure? = !@success
  
  # Monadic chaining
  def and_then
    return self if failure?
    yield(value)
  end
  
  def or_else
    return self if success?
    yield(errors)
  end
  
  def map
    return self if failure?
    ServiceResult.ok(yield(value))
  end
  
  def on_success(&block)
    block.call(value) if success?
    self
  end
  
  def on_failure(&block)
    block.call(errors, error_code) if failure?
    self
  end
end

# การใช้งาน
result = FindUser.call(id: params[:id])
  .and_then { |user| AuthorizeAccess.call(user: user, action: :edit) }
  .and_then { |user| UpdateUser.call(user: user, params: user_params) }
  .on_success { |user| redirect_to user_path(user) }
  .on_failure { |errors| render :edit, status: :unprocessable_entity }
```

### แบบฝึกหัดที่ 3: Interactor Organizer

```ruby
# สร้าง Organizer สำหรับ checkout process
class ProcessCheckout
  include Interactor::Organizer
  
  organize ValidateCart,
           ApplyDiscounts,
           CalculateShipping,
           CalculateTax,
           ChargePayment,
           CreateOrder,
           UpdateInventory,
           SendConfirmation,
           ClearCart
end

# สร้าง ValidateCart interactor
class ValidateCart
  include Interactor
  
  def call
    cart = context.cart
    
    if cart.items.empty?
      context.fail!(error: 'Cart is empty', code: :empty_cart)
    end
    
    cart.items.each do |item|
      unless item.product.available?
        context.fail!(
          error: "Product #{item.product.name} is not available",
          code: :product_unavailable
        )
      end
      
      if item.product.stock < item.quantity
        context.fail!(
          error: "Insufficient stock for #{item.product.name}",
          code: :insufficient_stock
        )
      end
    end
  end
end
```

### แบบฝึกหัดที่ 4: Form Object สำหรับ Multi-step Form

```ruby
# app/forms/job_application_form.rb
class JobApplicationForm
  include ActiveModel::Model
  include ActiveModel::Attributes
  
  # Personal info
  attribute :first_name, :string
  attribute :last_name, :string
  attribute :email, :string
  attribute :phone, :string
  
  # Work experience
  attribute :years_of_experience, :integer
  attribute :current_company, :string
  attribute :expected_salary, :decimal
  
  # Files (virtual attributes)
  attr_accessor :resume, :cover_letter
  
  validates :first_name, :last_name, :email, presence: true
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :years_of_experience, numericality: { greater_than_or_equal_to: 0 }
  validates :expected_salary, numericality: { greater_than: 0 }
  validates :resume, presence: true
  validate :valid_file_formats
  
  def save(job:)
    return false unless valid?
    
    applicant = Applicant.find_or_create_by!(email: email) do |a|
      a.first_name = first_name
      a.last_name = last_name
      a.phone = phone
    end
    
    application = JobApplication.create!(
      job: job,
      applicant: applicant,
      years_of_experience: years_of_experience,
      current_company: current_company,
      expected_salary: expected_salary
    )
    
    application.resume.attach(resume)
    application.cover_letter.attach(cover_letter) if cover_letter.present?
    
    JobApplicationMailer.confirmation(application).deliver_later
    JobApplicationMailer.notify_hr(application).deliver_later
    
    @application = application
    true
  end
  
  attr_reader :application
  
  private
  
  def valid_file_formats
    if resume.present? && !resume.content_type.in?(%w[application/pdf application/msword])
      errors.add(:resume, 'ต้องเป็นไฟล์ PDF หรือ Word')
    end
  end
end
```

### แบบฝึกหัดที่ 5: Query Object ซับซ้อน

```ruby
# app/queries/advanced_product_search.rb
class AdvancedProductSearch
  def initialize(relation = Product.all)
    @relation = relation.extending(Scopes)
  end
  
  def call(params = {})
    @relation
      .then { |r| filter_by_keyword(r, params[:q]) }
      .then { |r| filter_by_category(r, params[:category_ids]) }
      .then { |r| filter_by_price_range(r, params[:min_price], params[:max_price]) }
      .then { |r| filter_by_rating(r, params[:min_rating]) }
      .then { |r| filter_by_availability(r, params[:in_stock]) }
      .then { |r| sort(r, params[:sort]) }
  end
  
  module Scopes
    def filter_by_keyword(relation, keyword)
      return relation if keyword.blank?
      relation.where('name ILIKE :q OR description ILIKE :q', q: "%#{keyword}%")
    end
    
    def filter_by_category(relation, category_ids)
      return relation if category_ids.blank?
      relation.where(category_id: category_ids)
    end
    
    def filter_by_price_range(relation, min, max)
      relation = relation.where('price >= ?', min) if min.present?
      relation = relation.where('price <= ?', max) if max.present?
      relation
    end
    
    def filter_by_rating(relation, min_rating)
      return relation if min_rating.blank?
      relation.where('average_rating >= ?', min_rating)
    end
    
    def filter_by_availability(relation, in_stock)
      return relation if in_stock.blank?
      in_stock == 'true' ? relation.where('stock > 0') : relation
    end
    
    def sort(relation, sort_key)
      case sort_key
      when 'price_asc'   then relation.order(price: :asc)
      when 'price_desc'  then relation.order(price: :desc)
      when 'rating'      then relation.order(average_rating: :desc)
      when 'newest'      then relation.order(created_at: :desc)
      when 'popular'     then relation.order(sales_count: :desc)
      else                    relation.order(created_at: :desc)
      end
    end
  end
  
  private
  
  def filter_by_keyword(relation, keyword) = relation.filter_by_keyword(keyword)
  def filter_by_category(relation, ids) = relation.filter_by_category(ids)
  def filter_by_price_range(relation, min, max) = relation.filter_by_price_range(min, max)
  def filter_by_rating(relation, min) = relation.filter_by_rating(min)
  def filter_by_availability(relation, in_stock) = relation.filter_by_availability(in_stock)
  def sort(relation, key) = relation.sort(key)
end
```

### แบบฝึกหัดที่ 6-10: สร้าง Service Objects ต่างๆ

```ruby
# แบบฝึกหัดที่ 6: Password Reset Service
class ResetPassword
  attr_reader :errors
  
  def self.call(**args) = new(**args).tap(&:call)
  
  def initialize(token:, password:, password_confirmation:)
    @token = token
    @password = password
    @password_confirmation = password_confirmation
    @errors = []
  end
  
  def call
    load_user
    return if @errors.any?
    
    validate_passwords
    return if @errors.any?
    
    update_password
  end
  
  def success? = @errors.empty? && @user&.persisted?
  def user = @user
  
  private
  
  def load_user
    @user = User.find_by(reset_password_token: digest_token)
    
    if @user.nil? || @user.reset_password_sent_at < 2.hours.ago
      @errors << 'Token ไม่ถูกต้องหรือหมดอายุแล้ว'
    end
  end
  
  def digest_token
    Digest::SHA256.hexdigest(@token)
  end
  
  def validate_passwords
    @errors << 'รหัสผ่านจำเป็นต้องกรอก' if @password.blank?
    @errors << 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร' if @password.present? && @password.length < 8
    @errors << 'รหัสผ่านไม่ตรงกัน' if @password != @password_confirmation
  end
  
  def update_password
    @user.update!(
      password: @password,
      password_confirmation: @password_confirmation,
      reset_password_token: nil,
      reset_password_sent_at: nil
    )
  end
end

# แบบฝึกหัดที่ 7: Email Verification Service
class VerifyEmail
  def self.call(token:)
    new(token: token).call
  end
  
  def initialize(token:)
    @token = token
    @errors = []
  end
  
  def call
    user = User.find_by(
      email_verification_token: @token,
      email_verified_at: nil
    )
    
    if user.nil?
      return ServiceResult.err('Token ไม่ถูกต้อง', code: :invalid_token)
    end
    
    user.update!(
      email_verified_at: Time.current,
      email_verification_token: nil
    )
    
    ServiceResult.ok(user)
  end
end

# แบบฝึกหัดที่ 8: Subscription Management
class ManageSubscription
  PLANS = {
    'basic'   => { price: 299,  features: ['10 projects', '5GB storage'] },
    'pro'     => { price: 799,  features: ['Unlimited projects', '50GB storage'] },
    'business'=> { price: 1999, features: ['Unlimited everything', 'Priority support'] }
  }.freeze
  
  def self.upgrade(user:, plan:)
    new(user: user, plan: plan, action: :upgrade).call
  end
  
  def self.downgrade(user:, plan:)
    new(user: user, plan: plan, action: :downgrade).call
  end
  
  def self.cancel(user:)
    new(user: user, action: :cancel).call
  end
  
  def initialize(user:, action:, plan: nil)
    @user = user
    @action = action
    @plan = plan
    @errors = []
  end
  
  def call
    send("handle_#{@action}")
    self
  end
  
  def success? = @errors.empty?
  def errors = @errors
  
  private
  
  def handle_upgrade
    unless PLANS.key?(@plan)
      @errors << "ไม่พบ plan: #{@plan}"
      return
    end
    
    subscription = @user.subscription || @user.build_subscription
    subscription.update!(
      plan: @plan,
      status: 'active',
      started_at: Time.current,
      renewed_at: Time.current,
      expires_at: 1.month.from_now
    )
    
    SubscriptionMailer.upgraded(@user, @plan).deliver_later
  end
  
  def handle_downgrade
    current_plan_price = PLANS[@user.subscription&.plan]&.dig(:price) || 0
    new_plan_price = PLANS[@plan]&.dig(:price) || 0
    
    if new_plan_price >= current_plan_price
      @errors << 'ไม่สามารถ downgrade ไปยัง plan ที่ราคาสูงกว่าได้'
      return
    end
    
    @user.subscription.update!(
      plan: @plan,
      downgrade_scheduled_at: Time.current
    )
    
    SubscriptionMailer.downgrade_scheduled(@user, @plan).deliver_later
  end
  
  def handle_cancel
    @user.subscription.update!(
      status: 'cancelled',
      cancelled_at: Time.current,
      expires_at: @user.subscription.expires_at # คงเหลือจนหมดอายุ
    )
    
    SubscriptionMailer.cancellation(@user).deliver_later
  end
end
```

### แบบฝึกหัดที่ 11-20: ชุดแบบฝึกหัดเพิ่มเติม

```ruby
# แบบฝึกหัดที่ 11: Import Users from CSV
class ImportUsersFromCsv
  attr_reader :results
  
  def self.call(file:, import_by:)
    new(file: file, import_by: import_by).call
  end
  
  def initialize(file:, import_by:)
    @file = file
    @import_by = import_by
    @results = { created: 0, updated: 0, errors: [] }
  end
  
  def call
    require 'csv'
    
    CSV.foreach(@file, headers: true, encoding: 'UTF-8') do |row|
      process_row(row)
    end
    
    self
  rescue CSV::MalformedCSVError => e
    @results[:errors] << "CSV format error: #{e.message}"
    self
  end
  
  def success?
    @results[:errors].empty?
  end
  
  def summary
    "นำเข้า #{@results[:created]} รายการใหม่, " \
    "อัพเดท #{@results[:updated]} รายการ, " \
    "ข้อผิดพลาด #{@results[:errors].count} รายการ"
  end
  
  private
  
  def process_row(row)
    user = User.find_or_initialize_by(email: row['email'])
    user.assign_attributes(
      name: row['name'],
      phone: row['phone'],
      role: row['role'] || 'user'
    )
    
    if user.save
      user.new_record? ? @results[:created] += 1 : @results[:updated] += 1
    else
      @results[:errors] << "Row #{$.}: #{user.errors.full_messages.join(', ')}"
    end
  end
end

# แบบฝึกหัดที่ 12: Bulk Operations
class BulkUpdateProducts
  def self.call(product_ids:, attributes:, user:)
    new(product_ids: product_ids, attributes: attributes, user: user).call
  end
  
  def initialize(product_ids:, attributes:, user:)
    @product_ids = product_ids
    @attributes = attributes.slice(:price, :stock, :status, :category_id)
    @user = user
    @errors = []
  end
  
  def call
    validate_permissions
    return self if @errors.any?
    
    updated_count = Product.where(id: @product_ids).update_all(
      @attributes.merge(updated_at: Time.current)
    )
    
    @updated_count = updated_count
    log_bulk_update
    self
  end
  
  def success? = @errors.empty?
  def updated_count = @updated_count || 0
  attr_reader :errors
  
  private
  
  def validate_permissions
    unless @user.admin? || @user.manager?
      @errors << 'ไม่มีสิทธิ์ดำเนินการนี้'
    end
  end
  
  def log_bulk_update
    AuditLog.create!(
      action: 'bulk_product_update',
      user: @user,
      metadata: {
        product_ids: @product_ids,
        attributes: @attributes,
        updated_count: @updated_count
      }
    )
  end
end

# แบบฝึกหัดที่ 13: Notification Service
class SendNotification
  def self.call(**args) = new(**args).call
  
  def initialize(user:, type:, data: {})
    @user = user
    @type = type
    @data = data
  end
  
  def call
    template = NotificationTemplate.find_by(type: @type)
    return unless template
    
    notification = @user.notifications.create!(
      title: interpolate(template.title, @data),
      body: interpolate(template.body, @data),
      notification_type: @type,
      metadata: @data
    )
    
    # Real-time via ActionCable
    NotificationsChannel.broadcast_to(
      @user,
      notification: NotificationSerializer.new(notification).as_json
    )
    
    # Email หากผู้ใช้เปิดใช้งาน
    if @user.email_notifications_enabled?
      NotificationMailer.send_email(@user, notification).deliver_later
    end
    
    # Push notification หากมี device token
    if @user.push_token.present?
      PushNotificationJob.perform_later(@user.push_token, notification)
    end
    
    notification
  end
  
  private
  
  def interpolate(template, data)
    template.gsub(/:(\w+)/) { data[$1.to_sym] || data[$1] || $& }
  end
end
```

### สรุป Service Objects Best Practices

```
1. Single Responsibility Principle
   - หนึ่ง service object = หนึ่งงาน
   - ถ้า service ใหญ่เกินไป ให้แบ่งเป็น sub-services

2. เลือก naming ที่ชัดเจน
   - UserRegistration, ProcessPayment, SendWelcomeEmail
   - ชื่อควรบ่งบอกว่า "ทำอะไร" อย่างชัดเจน

3. Return Result Objects เสมอ
   - ไม่ควร raise exceptions ใน normal flow
   - ใช้ Result objects แทน

4. ใส่ validations ใน Service ไม่ใช่แค่ใน Model
   - Business rules อยู่ใน service
   - Model rules (format, presence) อยู่ใน model

5. ใช้ Transactions อย่างถูกต้อง
   - ทุก operation ที่เกี่ยวข้องกันควรอยู่ใน transaction เดียว

6. Log ทุก important operations
   - ช่วยในการ debug และ audit

7. Test Service Objects แยกจาก Controller
   - Service objects ควร testable โดยไม่ต้อง request/response
```

---

**จบ Part 66: Service Objects และ Business Logic**

*ในส่วนถัดไป Part 67 เราจะเรียนรู้เกี่ยวกับ Advanced Active Record*
