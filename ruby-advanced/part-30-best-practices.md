# ตอนที่ 30: Best Practices ใน Ruby (ขั้นตอนที่ 646-665)

Best Practices คือแนวทางปฏิบัติที่ดีที่ได้รับการยอมรับจาก Ruby Community ช่วยให้โค้ดอ่านง่าย บำรุงรักษาง่าย และมีคุณภาพสูง

---

## ขั้นตอนที่ 646: Ruby Style Guide และ RuboCop

### RuboCop - Ruby Static Code Analyzer

```bash
gem install rubocop

# ตรวจสอบ Code Style
rubocop

# Auto-fix ปัญหาที่แก้ได้อัตโนมัติ
rubocop --autocorrect
rubocop -a

# ตรวจสอบเฉพาะไฟล์
rubocop app/models/user.rb

# ตรวจสอบ Rails Code
gem install rubocop-rails
rubocop --require rubocop-rails
```

### .rubocop.yml Configuration

```yaml
# .rubocop.yml
inherit_from: .rubocop_todo.yml

require:
  - rubocop-rails
  - rubocop-rspec

AllCops:
  NewCops: enable
  SuggestExtensions: false
  Exclude:
    - 'db/schema.rb'
    - 'db/migrate/*.rb'
    - 'vendor/**/*'
    - 'config/environments/*.rb'

# Style Rules
Style/StringLiterals:
  EnforcedStyle: single_quotes
  
Style/FrozenStringLiteralComment:
  Enabled: true
  EnforcedStyle: always

Style/Documentation:
  Enabled: false

Style/GuardClause:
  Enabled: true
  MinBodyLength: 3

# Naming
Naming/VariableNumber:
  Enabled: false

# Metrics
Metrics/MethodLength:
  Max: 20
  
Metrics/ClassLength:
  Max: 200
  
Metrics/AbcSize:
  Max: 20

Metrics/CyclomaticComplexity:
  Max: 10

# Rails specific
Rails/HasManyOrHasOneDependent:
  Enabled: true

Rails/InverseOf:
  Enabled: true

Rails/UniqueValidationWithoutIndex:
  Enabled: true
```

### Ruby Style Guidelines

```ruby
# Naming Conventions
class UserAccount          # CamelCase สำหรับ Class
  MY_CONSTANT = 42         # SCREAMING_SNAKE_CASE สำหรับ Constant
  
  def full_name            # snake_case สำหรับ Method
    "#{first_name} #{last_name}"
  end
  
  def admin?               # ? สำหรับ Predicate Methods
    role == 'admin'
  end
  
  def deactivate!          # ! สำหรับ Dangerous Methods
    update!(active: false)
  end
end

# Indentation: 2 Spaces (ไม่ใช่ Tab)
def method_with_blocks
  [1, 2, 3].each do |n|
    if n > 1
      puts n
    end
  end
end

# One Liners
# OK: สั้นพอ
users.each { |u| puts u.name }

# Better: เมื่อยาวขึ้น
users.each do |user|
  send_welcome_email(user)
  update_last_login(user)
end

# String: ใช้ Single Quotes เว้นแต่มี Interpolation
name = 'สมชาย'           # Single Quote
greeting = "สวัสดี #{name}"  # Double Quote เพราะมี Interpolation

# Spaces
x = 1 + 2               # Spaces รอบ Operator
arr = [1, 2, 3]          # Space หลัง Comma
hash = { a: 1, b: 2 }   # Spaces ข้างใน Hash

# Line Length: max 120 chars (default 80)
result = very_long_method_name(first_arg,
                               second_arg,
                               third_arg)

# หรือ
result = very_long_method_name(
  first_arg,
  second_arg,
  third_arg
)
```

---

## ขั้นตอนที่ 647: SOLID Principles

### Single Responsibility Principle (SRP)

```ruby
# BAD: Class มีหน้าที่มากเกินไป
class UserManager
  def register(params)
    user = User.new(params)
    user.save!
    
    # Email validation
    unless params[:email].match?(/\A[^@\s]+@[^@\s]+\z/)
      raise "Invalid email"
    end
    
    # Send email
    smtp = Net::SMTP.new('mail.example.com', 587)
    smtp.send_message(welcome_email_body, 'noreply@example.com', params[:email])
    
    # Update analytics
    Analytics.track('user_registered', { email: params[:email] })
    
    user
  end
end

# GOOD: แยก Responsibility
class UserRegistrationService
  def initialize(
    validator:     UserValidator.new,
    mailer:        UserMailer,
    analytics:     Analytics
  )
    @validator  = validator
    @mailer     = mailer
    @analytics  = analytics
  end
  
  def register(params)
    @validator.validate!(params)
    user = User.create!(params)
    @mailer.welcome_email(user).deliver_later
    @analytics.track('user_registered', user_id: user.id)
    user
  end
end

class UserValidator
  def validate!(params)
    validate_email!(params[:email])
    validate_password!(params[:password])
    validate_name!(params[:name])
  end
  
  private
  
  def validate_email!(email)
    raise ArgumentError, "Email ไม่ถูกต้อง" unless valid_email?(email)
  end
  
  def valid_email?(email)
    email.to_s.match?(/\A[^@\s]+@[^@\s]+\z/)
  end
  
  def validate_password!(password)
    raise ArgumentError, "Password ต้องมีอย่างน้อย 8 ตัวอักษร" if password.to_s.length < 8
  end
  
  def validate_name!(name)
    raise ArgumentError, "ชื่อจำเป็นต้องระบุ" if name.to_s.strip.empty?
  end
end
```

### Open/Closed Principle (OCP)

```ruby
# BAD: ต้องแก้ไข Method เมื่อเพิ่ม Payment Type ใหม่
class PaymentProcessor
  def process(payment)
    case payment[:type]
    when :credit_card
      process_credit_card(payment)
    when :paypal
      process_paypal(payment)
    when :bitcoin  # ต้องเพิ่มที่นี่ทุกครั้ง!
      process_bitcoin(payment)
    end
  end
end

# GOOD: Open for Extension, Closed for Modification
class PaymentProcessor
  def initialize
    @handlers = {}
  end
  
  def register(type, handler)
    @handlers[type] = handler
  end
  
  def process(payment)
    handler = @handlers[payment[:type]]
    raise "ไม่รองรับ Payment Type: #{payment[:type]}" unless handler
    handler.process(payment)
  end
end

processor = PaymentProcessor.new
processor.register(:credit_card, CreditCardHandler.new)
processor.register(:paypal, PayPalHandler.new)
# เพิ่ม Handler ใหม่โดยไม่แก้ Core Code
processor.register(:bitcoin, BitcoinHandler.new)
```

### Liskov Substitution Principle (LSP)

```ruby
# BAD: Subclass ทำให้ Invariant ของ Parent พัง
class Rectangle
  attr_accessor :width, :height
  
  def area
    width * height
  end
end

class Square < Rectangle
  def width=(value)
    super
    @height = value  # Square ต้องมีความยาวด้านเท่ากัน
  end
  
  def height=(value)
    super
    @width = value
  end
end

def resize_rectangle(rect)
  rect.width  = 10
  rect.height = 5
  # ควรได้ area = 50
  # แต่ Square ให้ area = 25!
end

# GOOD: ใช้ Composition หรือ Separate Hierarchy
module Shape
  def area
    raise NotImplementedError
  end
end

class Rectangle
  include Shape
  attr_reader :width, :height
  
  def initialize(width, height)
    @width  = width
    @height = height
  end
  
  def area
    width * height
  end
end

class Square
  include Shape
  attr_reader :side
  
  def initialize(side)
    @side = side
  end
  
  def area
    side ** 2
  end
end
```

### Interface Segregation Principle (ISP)

```ruby
# BAD: Interface ใหญ่เกินไป
module Worker
  def work; end
  def eat; end
  def sleep; end
end

class RobotWorker
  include Worker
  
  def work  = "กำลังทำงาน"
  def eat   = raise NotImplementedError, "หุ่นยนต์ไม่กิน!"
  def sleep = raise NotImplementedError, "หุ่นยนต์ไม่นอน!"
end

# GOOD: แยก Interface เล็กๆ
module Workable
  def work
    raise NotImplementedError
  end
end

module Eatable
  def eat
    raise NotImplementedError
  end
end

module Sleepable
  def sleep
    raise NotImplementedError
  end
end

class HumanWorker
  include Workable
  include Eatable
  include Sleepable
  
  def work  = "กำลังทำงาน"
  def eat   = "กำลังกินข้าว"
  def sleep = "กำลังนอน"
end

class RobotWorker
  include Workable
  
  def work = "หุ่นยนต์กำลังทำงาน"
end
```

### Dependency Inversion Principle (DIP)

```ruby
# BAD: Depend on Concrete Implementation
class ReportGenerator
  def generate
    fetcher = MySQLFetcher.new        # Concrete Dependency!
    formatter = ExcelFormatter.new    # Concrete Dependency!
    
    data      = fetcher.fetch
    formatted = formatter.format(data)
    formatted
  end
end

# GOOD: Depend on Abstractions
class ReportGenerator
  def initialize(fetcher:, formatter:)
    @fetcher   = fetcher
    @formatter = formatter
  end
  
  def generate
    data      = @fetcher.fetch
    formatted = @formatter.format(data)
    formatted
  end
end

# ใช้ Dependency Injection
generator = ReportGenerator.new(
  fetcher:   MySQLFetcher.new,
  formatter: ExcelFormatter.new
)

# หรือ
generator = ReportGenerator.new(
  fetcher:   PostgresFetcher.new,
  formatter: CSVFormatter.new
)

# Test ง่าย!
generator = ReportGenerator.new(
  fetcher:   double('Fetcher', fetch: []),
  formatter: double('Formatter', format: "formatted")
)
```

---

## ขั้นตอนที่ 648: DRY (Don't Repeat Yourself)

```ruby
# BAD: Code ซ้ำ
class BlogPost
  def title_with_prefix
    "Blog: #{title}"
  end
  
  def meta_description
    description.length > 160 ? "#{description[0...157]}..." : description
  end
end

class Product
  def title_with_prefix
    "Product: #{title}"   # ซ้ำกับ BlogPost!
  end
  
  def meta_description
    description.length > 160 ? "#{description[0...157]}..." : description  # ซ้ำ!
  end
end

# GOOD: Extract Shared Logic
module Titleable
  def title_with_prefix
    "#{self.class.name}: #{title}"
  end
end

module MetaDescribable
  META_MAX_LENGTH = 160
  
  def meta_description
    if description.length > META_MAX_LENGTH
      "#{description[0...(META_MAX_LENGTH - 3)]}..."
    else
      description
    end
  end
end

class BlogPost
  include Titleable
  include MetaDescribable
end

class Product
  include Titleable
  include MetaDescribable
end

# DRY ใน Rails Queries
# BAD: Query ซ้ำหลายที่
class ProductsController < ApplicationController
  def index
    @products = Product.where(active: true).where("price > 0").order(:name)
  end
  
  def featured
    @products = Product.where(active: true).where("price > 0").order(:name).limit(5)
  end
end

# GOOD: Extract to Scope
class Product < ApplicationRecord
  scope :active,    -> { where(active: true) }
  scope :with_price, -> { where("price > 0") }
  scope :ordered,   -> { order(:name) }
  scope :base,      -> { active.with_price.ordered }
  scope :featured,  -> { base.limit(5) }
end

class ProductsController < ApplicationController
  def index
    @products = Product.base
  end
  
  def featured
    @products = Product.featured
  end
end
```

---

## ขั้นตอนที่ 649: Convention over Configuration

```ruby
# Rails Convention over Configuration

# 1. Naming Conventions
# Model:      User → users table
# Controller: UsersController → /users routes
# View:       users/index.html.erb

# 2. File Locations
# Model:      app/models/user.rb
# Controller: app/controllers/users_controller.rb
# View:       app/views/users/
# Test:       spec/models/user_spec.rb

# 3. Association Conventions
class User < ApplicationRecord
  has_many :orders            # users.id → orders.user_id
  belongs_to :company         # users.company_id → companies.id
  has_many :products, through: :orders
end

# 4. Callbacks ตาม Convention
class Order < ApplicationRecord
  before_create :generate_reference_number
  after_create  :send_confirmation_email
  before_save   :calculate_totals, if: :items_changed?
  
  private
  
  def generate_reference_number
    self.reference = "ORD-#{SecureRandom.hex(4).upcase}"
  end
  
  def send_confirmation_email
    OrderMailer.confirmation(self).deliver_later
  end
  
  def calculate_totals
    self.total = items.sum { |item| item.price * item.quantity }
  end
  
  def items_changed?
    items.any?(&:changed?)
  end
end

# 5. Service Objects Convention
# app/services/user_registration_service.rb
class UserRegistrationService
  Result = Struct.new(:success, :user, :errors)
  
  def initialize(params)
    @params = params
  end
  
  def call
    user = User.new(@params)
    
    if user.save
      send_welcome_email(user)
      Result.new(true, user, [])
    else
      Result.new(false, user, user.errors.full_messages)
    end
  end
  
  private
  
  def send_welcome_email(user)
    UserMailer.welcome_email(user).deliver_later
  end
end
```

---

## ขั้นตอนที่ 650: Clean Code

### Readable Code

```ruby
# BAD: Code อ่านยาก
def p(u, t)
  u.a && u.r.include?(t) && u.s == 'active'
end

# GOOD: ชื่อที่สื่อความหมาย
def user_can_perform_action?(user, action_type)
  user.authenticated? &&
    user.roles.include?(action_type) &&
    user.status == 'active'
end

# BAD: Magic Numbers
def calculate_fee(amount)
  amount * 0.07
end

# GOOD: Named Constants
VAT_RATE = 0.07

def calculate_vat(amount)
  amount * VAT_RATE
end

# BAD: Deep Nesting
def process_order(order)
  if order
    if order.user
      if order.user.active?
        if order.items.any?
          # ทำงาน...
        end
      end
    end
  end
end

# GOOD: Guard Clauses
def process_order(order)
  return unless order
  return unless order.user
  return unless order.user.active?
  return unless order.items.any?
  
  # ทำงาน...
end

# หรือ Raise Exception
def process_order!(order)
  raise ArgumentError, "Order required"     unless order
  raise ArgumentError, "User not found"      unless order.user
  raise AuthorizationError, "User inactive" unless order.user.active?
  raise ArgumentError, "No items in order"  unless order.items.any?
  
  # ทำงาน...
end
```

### Expressive Ruby

```ruby
# ใช้ Ruby's Expressive Syntax
# ตรวจสอบว่า Array มีสมาชิก
users.any?    # มีหรือไม่
users.none?   # ไม่มีเลยหรือไม่
users.all?    # ทั้งหมดหรือเปล่า
users.one?    # มีแค่ตัวเดียวหรือไม่

# ใช้ Predicate Methods
if user.admin?          # ดีกว่า if user.role == 'admin'
if order.pending?       # ดีกว่า if order.status == 'pending'
if items.empty?         # ดีกว่า if items.length == 0
if name.present?        # Rails: ดีกว่า if !name.nil? && !name.empty?

# Method chaining ที่อ่านง่าย
User
  .where(active: true)
  .where(age: 18..)
  .includes(:orders)
  .order(created_at: :desc)
  .limit(20)

# Tap สำหรับ Debugging ใน Chain
User.create!(name: 'สมชาย').tap do |user|
  send_welcome_email(user)
  update_analytics(user)
end

# Then/Yield Self สำหรับ Transform
"  Hello World  "
  .then { |s| s.strip }
  .then { |s| s.downcase }
  .then { |s| s.split.first }
```

---

## ขั้นตอนที่ 651: Code Smells

### ระบุและแก้ไข Code Smells

```ruby
# Code Smell 1: Long Method
# BAD: Method ยาวเกินไป (มากกว่า 20 บรรทัด)
def process_order(order)
  # 100+ lines of code
  validate_items
  calculate_pricing
  apply_discounts
  charge_payment
  update_inventory
  send_notifications
  generate_invoice
end

# GOOD: แบ่งเป็น Methods ย่อย
def process_order(order)
  validate_order(order)
  total = calculate_order_total(order)
  transaction = charge_payment(order, total)
  fulfill_order(order, transaction)
end

# Code Smell 2: Large Class (God Object)
# BAD: Class รู้ทุกอย่าง ทำทุกอย่าง
class User
  # Authentication, Profile, Orders, Analytics, Email, etc.
end

# GOOD: แยก Concerns
class User < ApplicationRecord
  include Authenticatable
  include Profileable
end

module Authenticatable
  # Authentication logic
end

module Profileable
  # Profile management
end

# Code Smell 3: Primitive Obsession
# BAD: ใช้ Primitive Types แทน Value Objects
def create_order(user_id, street, city, province, postcode, items_json)
  items = JSON.parse(items_json)
  # ...
end

# GOOD: ใช้ Value Objects
def create_order(user:, shipping_address:, items:)
  Order.new(user: user, shipping_address: shipping_address, items: items)
end

Address = Struct.new(:street, :city, :province, :postcode, keyword_init: true)

address = Address.new(
  street:   '123 ถ.สุขุมวิท',
  city:     'กรุงเทพฯ',
  province: 'กรุงเทพมหานคร',
  postcode: '10110'
)

# Code Smell 4: Feature Envy
# BAD: Method ใช้ข้อมูลจาก Object อื่นมากกว่า Object ตัวเอง
class OrderPresenter
  def initialize(order)
    @order = order
  end
  
  def format_user_info
    "#{@order.user.first_name} #{@order.user.last_name} (#{@order.user.email})"
  end
end

# GOOD: ย้าย Method ไปอยู่ที่เหมาะสม
class User
  def display_name
    "#{first_name} #{last_name} (#{email})"
  end
end

class OrderPresenter
  def format_user_info
    @order.user.display_name
  end
end

# Code Smell 5: Duplicate Code (ดู DRY Section)

# Code Smell 6: Data Clumps
# BAD: Params ที่มักปรากฏด้วยกันเสมอ
def create_user(first_name, last_name, email, street, city, postcode)
  # ...
end

# GOOD: Extract ออกเป็น Object
PersonInfo = Struct.new(:first_name, :last_name, :email, keyword_init: true)
Address    = Struct.new(:street, :city, :postcode, keyword_init: true)

def create_user(person:, address:)
  # ...
end
```

---

## ขั้นตอนที่ 652: Documentation ด้วย YARD

### YARD Documentation

```ruby
# Gemfile
# gem 'yard'

# lib/user_service.rb

# UserService จัดการ Business Logic สำหรับ Users
#
# @example สร้าง User ใหม่
#   service = UserService.new
#   user = service.create(name: 'สมชาย', email: 'somchai@example.com')
#   # => #<User id=1, name="สมชาย">
#
# @example อัพเดท User
#   result = service.update(user, { name: 'สมชาย ใหม่' })
#   # => #<User id=1, name="สมชาย ใหม่">
class UserService
  
  # สร้าง User ใหม่ในระบบ
  #
  # @param params [Hash] ข้อมูล User ที่ต้องการสร้าง
  # @option params [String] :name ชื่อ User (จำเป็น)
  # @option params [String] :email Email ของ User (จำเป็น, ต้อง Unique)
  # @option params [String] :password Password (จำเป็น, อย่างน้อย 8 ตัวอักษร)
  # @option params [String] :role Role ของ User ('member', 'admin') Default: 'member'
  #
  # @return [User] User ที่สร้างขึ้น
  # @raise [ArgumentError] เมื่อ Email ไม่ถูกต้อง
  # @raise [ActiveRecord::RecordNotUnique] เมื่อ Email ซ้ำ
  #
  # @example
  #   user = service.create(name: 'สมชาย', email: 'test@example.com', password: 'pass1234')
  def create(params)
    validate_params!(params)
    User.create!(params.merge(role: params[:role] || 'member'))
  end
  
  # ค้นหา User ตาม Email
  #
  # @param email [String] Email ที่ต้องการค้นหา
  # @return [User, nil] User ที่พบ หรือ nil ถ้าไม่พบ
  def find_by_email(email)
    User.find_by(email: email.to_s.downcase.strip)
  end
  
  private
  
  # @param params [Hash] ข้อมูลที่จะตรวจสอบ
  # @raise [ArgumentError] เมื่อข้อมูลไม่ถูกต้อง
  def validate_params!(params)
    raise ArgumentError, "ชื่อจำเป็นต้องระบุ"  unless params[:name].present?
    raise ArgumentError, "Email จำเป็นต้องระบุ" unless params[:email].present?
    raise ArgumentError, "Email ไม่ถูกต้อง" unless valid_email?(params[:email])
  end
  
  def valid_email?(email)
    email.to_s.match?(/\A[^@\s]+@[^@\s]+\z/)
  end
end
```

```bash
# สร้าง Documentation
yard doc

# ดู Documentation ใน Browser
yard server
# เปิด http://localhost:8808
```

---

## ขั้นตอนที่ 653-655: Git Workflow

### Git Flow สำหรับ Ruby Projects

```bash
# Feature Branch Workflow

# 1. สร้าง Feature Branch
git checkout main
git pull origin main
git checkout -b feature/add-payment-system

# 2. พัฒนา Feature
# แก้โค้ด...

# 3. Commit ตามหลัก Conventional Commits
git add .
git commit -m "feat: add Stripe payment processor

- Implement StripePaymentService
- Add payment form UI
- Handle webhook callbacks
- Add tests for payment flow

Closes #123"

# 4. Push และสร้าง Pull Request
git push origin feature/add-payment-system

# 5. Code Review และ Merge

# Commit Message Format (Conventional Commits)
feat: เพิ่ม Feature ใหม่
fix: แก้ Bug
docs: เปลี่ยน Documentation
style: เปลี่ยน Code Format (ไม่กระทบ Logic)
refactor: Refactor Code
test: เพิ่ม/แก้ Tests
chore: เปลี่ยน Build Process หรือ Tools
perf: ปรับปรุง Performance
ci: เปลี่ยน CI Configuration
```

### Pre-commit Hooks

```bash
# .git/hooks/pre-commit
#!/bin/sh

# รัน RuboCop ก่อน Commit
bundle exec rubocop --autocorrect-all
if [ $? -ne 0 ]; then
  echo "RuboCop พบปัญหา กรุณาแก้ไขก่อน Commit"
  exit 1
fi

# รัน Tests
bundle exec rspec --fail-fast
if [ $? -ne 0 ]; then
  echo "Tests ไม่ผ่าน กรุณาแก้ไขก่อน Commit"
  exit 1
fi

exit 0
```

```ruby
# Gemfile: gem 'overcommit'
# หรือ gem 'lefthook'

# .overcommit.yml
PreCommit:
  RuboCop:
    enabled: true
    on_warn: fail
    command: ['bundle', 'exec', 'rubocop']
  
  RSpec:
    enabled: true
    command: ['bundle', 'exec', 'rspec', '--fail-fast']
  
  BundleAudit:
    enabled: true
    command: ['bundle', 'audit', 'check', '--update']

CommitMsg:
  CapitalizedSubject:
    enabled: true
  
  SingleLineSubject:
    enabled: true
  
  TextWidth:
    enabled: true
    max_subject_width: 72
```

---

## ขั้นตอนที่ 656-660: Refactoring Patterns

### Extract Method

```ruby
# Before
def create_order
  # Validate
  raise "ไม่พบ User" unless user
  raise "ตะกร้าว่าง" if cart.empty?
  
  # Calculate
  subtotal = cart.items.sum { |item| item.price * item.quantity }
  discount = coupon ? coupon.amount : 0
  tax = (subtotal - discount) * 0.07
  total = subtotal - discount + tax
  
  # Create
  order = Order.create!(user: user, total: total)
  
  order
end

# After
def create_order
  validate_order_prerequisites!
  total = calculate_order_total
  Order.create!(user: user, total: total)
end

private

def validate_order_prerequisites!
  raise OrderError, "ไม่พบ User" unless user
  raise OrderError, "ตะกร้าว่าง" if cart.empty?
end

def calculate_order_total
  subtotal = cart.items.sum { |item| item.price * item.quantity }
  discount = applied_discount
  tax      = calculate_tax(subtotal, discount)
  subtotal - discount + tax
end

def applied_discount
  coupon ? coupon.amount : 0
end

def calculate_tax(subtotal, discount)
  (subtotal - discount) * TAX_RATE
end
```

### Replace Conditional with Polymorphism

```ruby
# Before
class Animal
  def speak
    case animal_type
    when 'dog'
      "โฮ่ง!"
    when 'cat'
      "เมี้ยว!"
    when 'cow'
      "มู!"
    end
  end
end

# After
class Animal
  def speak
    raise NotImplementedError
  end
end

class Dog < Animal
  def speak = "โฮ่ง!"
end

class Cat < Animal
  def speak = "เมี้ยว!"
end

class Cow < Animal
  def speak = "มู!"
end
```

---

## แบบฝึกหัดบทที่ 30 (20 ข้อ)

**ข้อ 1:** ติดตั้ง RuboCop และแก้ไข Style Issues ในโปรเจกต์ที่มี

**ข้อ 2:** Refactor Class ที่มี Long Method โดยใช้ Extract Method

**ข้อ 3:** ใช้ SOLID Principles Refactor Legacy Code

**ข้อ 4:** สร้าง Module เพื่อ Extract Shared Behavior จาก Multiple Classes

**ข้อ 5:** แก้ไข Code Smell: Feature Envy ใน Presenter Class

**ข้อ 6:** เพิ่ม YARD Documentation สำหรับ Service Object

**ข้อ 7:** สร้าง Git Pre-commit Hook ที่รัน RuboCop และ Tests

**ข้อ 8:** Refactor Conditional Logic โดยใช้ Strategy Pattern

**ข้อ 9:** ใช้ DRY Principle ลด Code Duplication ใน Controllers

**ข้อ 10:** สร้าง .rubocop.yml ที่เหมาะกับ Project Style

**ข้อ 11:** Refactor Nested If/Else ด้วย Guard Clauses

**ข้อ 12:** Extract Value Objects จาก Primitive Types

**ข้อ 13:** ใช้ Convention over Configuration ใน Rails Service Objects

**ข้อ 14:** Implement Dependency Injection สำหรับ External Services

**ข้อ 15:** Code Review โดยใช้ Rubocop Guidelines

**ข้อ 16:** สร้าง Changelog ที่ดีสำหรับ Ruby Project

**ข้อ 17:** Setup CI/CD Pipeline พร้อม Linting, Testing, Security Checks

**ข้อ 18:** Refactor Large Class ด้วย Concerns และ Mixins

**ข้อ 19:** สร้าง README.md ที่สมบูรณ์สำหรับ Ruby Library

**ข้อ 20:** ทำ Complete Code Review สำหรับ Small Rails Feature

---

### เฉลยตัวอย่าง ข้อ 3: SOLID Refactoring

```ruby
# ก่อน Refactor: ผิด SRP, OCP, DIP
class InvoiceGenerator
  def generate(order_id)
    order = ActiveRecord::Base.connection.execute(
      "SELECT * FROM orders WHERE id = #{order_id}"
    ).first
    
    html = "<html><body>"
    html += "<h1>Invoice ##{order['id']}</h1>"
    html += "<p>Total: #{order['total']}</p>"
    html += "</body></html>"
    
    File.write("invoices/invoice_#{order_id}.html", html)
    
    # ส่ง Email
    Net::SMTP.start('smtp.example.com', 25) do |smtp|
      smtp.send_message(
        "Invoice generated",
        'invoices@example.com',
        order['user_email']
      )
    end
  end
end

# หลัง Refactor: ตาม SOLID

# Repository (Data Access)
class OrderRepository
  def find(id)
    Order.find(id)
  end
end

# Template Engine (Single Responsibility)
class InvoiceTemplate
  def render(order)
    <<~HTML
      <html>
      <body>
        <h1>Invoice ##{order.id}</h1>
        <table>
          #{render_items(order.items)}
        </table>
        <p>Total: #{order.total}</p>
      </body>
      </html>
    HTML
  end
  
  private
  
  def render_items(items)
    items.map do |item|
      "<tr><td>#{item.name}</td><td>#{item.price}</td></tr>"
    end.join("\n")
  end
end

# Storage Strategy (Open/Closed)
module StorageStrategy
  def store(filename, content)
    raise NotImplementedError
  end
end

class FileStorage
  include StorageStrategy
  
  def initialize(base_path: 'invoices')
    @base_path = base_path
  end
  
  def store(filename, content)
    path = File.join(@base_path, filename)
    FileUtils.mkdir_p(@base_path)
    File.write(path, content)
    path
  end
end

class S3Storage
  include StorageStrategy
  
  def initialize(bucket:, prefix: 'invoices')
    @bucket = bucket
    @prefix = prefix
    @s3     = Aws::S3::Resource.new
  end
  
  def store(filename, content)
    key = "#{@prefix}/#{filename}"
    @s3.bucket(@bucket).object(key).put(body: content)
    key
  end
end

# Notification Strategy (Open/Closed)
module NotificationStrategy
  def notify(recipient, subject, body)
    raise NotImplementedError
  end
end

class EmailNotification
  include NotificationStrategy
  
  def initialize(mailer: InvoiceMailer)
    @mailer = mailer
  end
  
  def notify(recipient, subject, body)
    @mailer.invoice_ready(recipient, subject, body).deliver_later
  end
end

# Invoice Generator (Dependency Inversion)
class InvoiceGenerator
  def initialize(
    repository:   OrderRepository.new,
    template:     InvoiceTemplate.new,
    storage:      FileStorage.new,
    notification: EmailNotification.new
  )
    @repository   = repository
    @template     = template
    @storage      = storage
    @notification = notification
  end
  
  def generate(order_id)
    order    = @repository.find(order_id)
    content  = @template.render(order)
    filename = "invoice_#{order.id}.html"
    location = @storage.store(filename, content)
    
    @notification.notify(
      order.user.email,
      "Invoice ##{order.id}",
      "Invoice ของคุณพร้อมแล้ว: #{location}"
    )
    
    { success: true, location: location }
  rescue ActiveRecord::RecordNotFound => e
    { success: false, error: "ไม่พบ Order: #{e.message}" }
  end
end

# การใช้งาน
# Development
generator = InvoiceGenerator.new

# Production ด้วย S3
generator = InvoiceGenerator.new(
  storage: S3Storage.new(bucket: 'my-invoices')
)

# Testing ด้วย Test Doubles
generator = InvoiceGenerator.new(
  repository:   instance_double(OrderRepository, find: mock_order),
  template:     instance_double(InvoiceTemplate, render: "<html>...</html>"),
  storage:      instance_double(FileStorage, store: '/tmp/invoice.html'),
  notification: instance_double(EmailNotification, notify: true)
)
```

---

*สรุปบทที่ 30: Best Practices ใน Ruby ครอบคลุมตั้งแต่ Style Guide, SOLID Principles, DRY, Clean Code ไปจนถึง Documentation และ Git Workflow การนำ Best Practices เหล่านี้ไปใช้อย่างสม่ำเสมอจะช่วยให้โค้ดมีคุณภาพสูง อ่านง่าย และบำรุงรักษาได้ในระยะยาว*

---

## สรุปรวม Ruby Advanced Course

หลังจากเรียนรู้ทั้ง 9 บทในส่วน Ruby Advanced แล้ว ผู้เรียนจะมีความเข้าใจใน:

1. **Design Patterns** - การแก้ปัญหาซ้ำๆ ด้วย Patterns ที่พิสูจน์แล้ว
2. **Testing with RSpec** - การเขียน Test ที่ดีและครอบคลุม
3. **Gems and Bundler** - การจัดการ Dependencies และสร้าง Gem
4. **Concurrency** - Thread, Fiber, Ractor สำหรับ Parallel Programming
5. **Functional Programming** - Pure Functions, Immutability, Composition
6. **Performance** - Benchmarking, Profiling, Optimization
7. **Debugging** - Tools และ Techniques สำหรับค้นหา Bug
8. **Security** - Input Validation, XSS, SQL Injection, Password Hashing
9. **Best Practices** - Style Guide, SOLID, Clean Code, Documentation

ความรู้เหล่านี้จะช่วยให้เป็น Ruby Developer ที่มีคุณภาพสูงและสามารถพัฒนา Application ที่ Production-ready ได้
