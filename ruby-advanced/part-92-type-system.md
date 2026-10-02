# Part 92: Ruby Type System

## บทนำ

Ruby เป็น dynamically-typed language แต่ในช่วงไม่กี่ปีที่ผ่านมา มีเครื่องมือสำหรับ optional static typing มากมาย บทนี้จะครอบคลุม RBS, Steep, Sorbet, TypeProf และการ apply type safety ใน Rails applications

## 1. Ruby Type System Overview

### 1.1 Duck Typing ใน Ruby

```ruby
# Ruby ใช้ Duck Typing: "ถ้ามันเดินเหมือนเป็ดและร้องเหมือนเป็ด มันก็คือเป็ด"
# Ruby ไม่สนใจ class ของ object แต่สนใจว่า object มี method ที่ต้องการหรือไม่

class Duck
  def quack = puts "Quack!"
  def walk  = puts "Walking like a duck"
end

class Person
  def quack = puts "I'm quacking like a duck!"
  def walk  = puts "Walking like a person"
end

class RubberDuck
  def quack = puts "Squeak!"
  def walk  = puts "Can't walk, I'm rubber!"
end

def make_it_quack(duck)
  duck.quack  # ไม่สนใจว่า duck เป็น class อะไร
end

make_it_quack(Duck.new)       # Quack!
make_it_quack(Person.new)     # I'm quacking like a duck!
make_it_quack(RubberDuck.new) # Squeak!

# respond_to? สำหรับ defensive programming
def process(obj)
  if obj.respond_to?(:to_s)
    obj.to_s
  else
    "Cannot convert to string"
  end
end
```

### 1.2 เหตุผลที่ต้องการ Type System

```ruby
# ปัญหาของ dynamic typing:
# 1. Runtime errors ที่อาจหาได้ยาก
def calculate_discount(price, percentage)
  price * percentage / 100  # ถ้า price เป็น String จะ error ตอน runtime
end

calculate_discount("100", 10)  # NoMethodError: undefined method `/' for "100":String

# 2. IDE ไม่รู้ว่า method มีอะไรบ้าง
user = get_user(1)  # editor ไม่รู้ว่า user มี methods อะไร
user.  # ไม่มี auto-complete ที่ถูกต้อง

# 3. Refactoring ยากกว่า - ไม่รู้ว่าจะกระทบอะไรบ้าง
def update_user_name(user, name)
  user.name = name  # ถ้า rename attribute นี้ หาที่ใช้ทั้งหมดยาก
end

# Type system ช่วย:
# - Catch errors ก่อน runtime
# - Better IDE support
# - Self-documenting code
# - Safer refactoring
```

## 2. RBS (Ruby Signature)

### 2.1 RBS Basics

```rbs
# RBS คือ type definition language สำหรับ Ruby
# ไฟล์ .rbs อธิบาย types ของ Ruby code

# sig/models/user.rbs
class User
  # Instance variables
  @name: String
  @email: String
  @age: Integer
  @created_at: Time
  
  # Attr accessors
  attr_accessor name: String
  attr_accessor email: String
  attr_reader id: Integer
  attr_reader created_at: Time
  
  # Constructor
  def initialize: (name: String, email: String, ?age: Integer) -> void
  
  # Instance methods
  def full_name: () -> String
  def admin?: () -> bool
  def authenticate: (String password) -> bool
  
  # Methods with union types
  def display_name: () -> (String | nil)
  
  # Methods with block
  def posts: () -> ActiveRecord::Associations::CollectionProxy[Post]
  
  # Class methods
  def self.find: (Integer id) -> User
  def self.find_by_email: (String email) -> User?  # ? หมายถึง nilable
  def self.create: (name: String, email: String, password: String) -> User
  
  # Static methods แบบ overload
  def self.where: (Hash[Symbol, untyped] conditions) -> ActiveRecord::Relation
                | (String sql, *untyped binds) -> ActiveRecord::Relation
end
```

### 2.2 RBS Type Syntax

```rbs
# Types ที่ใช้บ่อยใน RBS

# Primitive types
class Example
  @name: String
  @count: Integer
  @ratio: Float
  @enabled: bool
  @data: nil
  @value: untyped    # ไม่รู้ type (any type)
end

# Collection types
class Collections
  @names: Array[String]
  @scores: Hash[String, Integer]
  @pairs: Array[Array[Integer]]
  @nested: Hash[Symbol, Array[String]]
end

# Union types (|)
class Unions
  def parse: (String | Integer) -> Float
  def value: () -> String | nil   # สั้นกว่า (String | nil)
  def find: (Integer) -> User?    # ? คือ alias ของ | nil
end

# Intersection types (&)
class Intersections
  # ต้องมีทั้ง Comparable และ Serializable
  def sort_and_save: (Comparable & Serializable item) -> void
end

# Tuple types
class Tuples
  def min_max: (Array[Integer]) -> [Integer, Integer]
  def coordinates: () -> [Float, Float, Float]
end

# Proc types
class Procs
  def transform: (^(Integer) -> String transformer) -> Array[String]
  def filter: (^(String) -> bool predicate) -> Array[String]
end

# Generic types
class Container[T]
  @items: Array[T]
  
  def initialize: () -> void
  def add: (T item) -> void
  def get: (Integer index) -> T
  def each: { (T) -> void } -> void
end
```

### 2.3 RBS สำหรับ Rails Models

```rbs
# sig/models/post.rbs
class Post < ApplicationRecord
  # Associations
  def user: () -> User
  def comments: () -> ActiveRecord::Associations::CollectionProxy[Comment]
  def tags: () -> ActiveRecord::Associations::CollectionProxy[Tag]
  
  # Scopes
  def self.published: () -> ActiveRecord::Relation
  def self.recent: () -> ActiveRecord::Relation
  def self.by_user: (User user) -> ActiveRecord::Relation
  
  # Validations (implicit)
  
  # Instance methods
  def publish!: () -> bool
  def unpublish!: () -> bool
  def published?: () -> bool
  
  def reading_time: () -> Integer  # minutes
  def excerpt: (?Integer length) -> String
  
  # Callbacks (defined in Rails)
  
  # Serialized columns
  def metadata: () -> Hash[String, untyped]
  def metadata=: (Hash[String, untyped]) -> Hash[String, untyped]
end

# sig/models/concerns/searchable.rbs
module Searchable
  def self.included: (Module base) -> void
  
  def search_index: () -> Hash[String, untyped]
  
  module ClassMethods
    def search: (String query, ?page: Integer, ?per_page: Integer) -> ActiveRecord::Relation
    def full_text_search: (String query) -> ActiveRecord::Relation
  end
end
```

### 2.4 RBS สำหรับ Services

```rbs
# sig/services/payment_service.rbs
class PaymentService
  type payment_result = {
    success: bool,
    transaction_id: String,
    amount: Integer,
    currency: String
  }
  
  type payment_error = {
    code: String,
    message: String,
    decline_code: String?
  }
  
  @user: User
  @stripe_client: Stripe::StripeClient
  
  def initialize: (user: User) -> void
  
  def charge!: (amount: Integer, ?currency: String, ?description: String) 
    -> payment_result
  
  def refund!: (transaction_id: String, ?amount: Integer) -> bool
  
  def subscribe!: (
    plan_id: String,
    ?trial_days: Integer
  ) -> Stripe::Subscription
  
  def cancel_subscription!: (?at_period_end: bool) -> bool
  
  private
    def ensure_customer: () -> Stripe::Customer
    def handle_stripe_error: (Stripe::StripeError) -> never
end

# sig/services/user_service.rbs
class UserService
  class UserNotFoundError < StandardError
  end
  
  class DuplicateEmailError < StandardError
  end
  
  def self.create: (
    name: String,
    email: String,
    password: String,
    ?role: :admin | :user | :moderator
  ) -> User
  
  def self.authenticate: (email: String, password: String) -> User?
  
  def self.update_profile: (
    user: User,
    name: String,
    ?bio: String,
    ?avatar: ActionDispatch::Http::UploadedFile
  ) -> User
  
  def self.deactivate: (User user) -> void
  def self.send_password_reset: (email: String) -> void
end
```

## 3. Steep Type Checker Setup

### 3.1 การติดตั้ง Steep

```ruby
# Gemfile
group :development do
  gem 'steep'
  gem 'rbs'  # bundled with Ruby 3.0+
end
```

```bash
# ติดตั้ง
bundle install

# สร้าง Steepfile
bundle exec steep init

# สร้าง sig directory structure
mkdir -p sig/models sig/controllers sig/services sig/lib
```

### 3.2 Steepfile Configuration

```ruby
# Steepfile
D = Steep::Diagnostic

target :app do
  # Source files to check
  check "app/models"
  check "app/controllers"
  check "app/services"
  check "lib"
  
  # Signature files
  signature "sig"
  
  # Configure diagnostic severity
  configure_code_diagnostics do |hash|
    # Errors - ห้ามผ่าน
    hash[D::Ruby::UnexpectedPositionalArgument] = :error
    hash[D::Ruby::IncompatibleArguments] = :error
    hash[D::Ruby::UnexpectedKeywordArgument] = :error
    hash[D::Ruby::NoMethod] = :error
    hash[D::Ruby::IncompatibleMethodTypeAnnotation] = :error
    
    # Warnings - แจ้งเตือน
    hash[D::Ruby::UnresolvedOverloading] = :warning
    hash[D::Ruby::UnknownInstanceVariable] = :warning
    hash[D::Ruby::UnknownConstant] = :warning
    
    # Hints - ข้อมูล
    hash[D::Ruby::MethodDefinitionMissing] = :hint
  end
end

target :test do
  check "spec"
  signature "sig"
  
  library "rspec-core"
  
  configure_code_diagnostics do |hash|
    hash[D::Ruby::NoMethod] = :hint  # Less strict for tests
  end
end
```

### 3.3 Running Steep

```bash
# ตรวจสอบ type errors
bundle exec steep check

# Watch mode
bundle exec steep watch

# Check เฉพาะ file
bundle exec steep check app/models/user.rb

# Output:
# app/models/user.rb:15:5: [error] Cannot pass a value of type `::Integer` 
#   as an argument of type `::String`
#   Assigned type: ::Integer
#   Expected type: ::String
#   (RBS::Test::Errors::ArgumentTypeError)
```

### 3.4 Steep กับ Rails

```ruby
# Steep ต้องการ RBS สำหรับ Rails types
# รวมถึง gems ด้วย

# Gemfile
gem 'rbs-rails'  # RBS for Rails

# Rakefile (หรือ Rakefile.rb)
require "rbs_rails/rake_task"

RbsRails::RakeTask.new do |task|
  task.signature_root_dir = "sig"
end

# Generate RBS สำหรับ Rails
bundle exec rake rbs_rails:all

# สร้าง RBS จาก schema ของ database
bundle exec rake rbs_rails:generate

# Protips:
# - gen Rails model RBS อัตโนมัติจาก database schema
# - รวม AR associations, validations
```

### 3.5 Annotating Ruby Code

```ruby
# app/services/calculator.rb - ไม่มี type annotations
class Calculator
  def add(a, b)
    a + b
  end
  
  def divide(a, b)
    raise ArgumentError, "Division by zero" if b == 0
    a / b.to_f
  end
end

# sig/services/calculator.rbs - RBS annotations แยก
class Calculator
  def add: (Integer | Float, Integer | Float) -> (Integer | Float)
  def divide: (Integer | Float, Integer | Float) -> Float
end

# Steep จะตรวจสอบว่า implementation ตรงกับ signature
calc = Calculator.new
result = calc.add(1, "2")  # Error! String ไม่ใช่ Integer | Float
```

## 4. Sorbet Type Annotations

### 4.1 Sorbet Basics

```ruby
# Gemfile
gem 'sorbet'
gem 'sorbet-runtime'
gem 'tapioca'  # Tool สำหรับสร้าง RBI files

# Setup
bundle exec tapioca init
bundle exec srb init
```

### 4.2 Sorbet Type Levels

```ruby
# typed: ignore   - ไม่ check อะไรเลย
# typed: false    - check syntax เท่านั้น (default)
# typed: true     - check basic types
# typed: strict   - require type annotations ทุก method
# typed: strong   - strict + no untyped

# app/models/user.rb
# typed: strict

class User < ApplicationRecord
  extend T::Sig
  
  # Basic type annotations
  sig { params(name: String, email: String).void }
  def initialize(name:, email:)
    @name = T.let(name, String)
    @email = T.let(email, String)
  end
  
  sig { returns(String) }
  def greeting
    "Hello, #{@name}!"
  end
  
  sig { params(password: String).returns(T::Boolean) }
  def authenticate(password)
    BCrypt::Password.new(password_digest).is_password?(password)
  end
  
  # Nilable types
  sig { returns(T.nilable(String)) }
  def bio
    @bio
  end
  
  # Union types
  sig { returns(T.any(String, Integer)) }
  def identifier
    admin? ? @name : @id
  end
  
  # Array and Hash types
  sig { returns(T::Array[String]) }
  def permission_names
    permissions.pluck(:name)
  end
  
  sig { returns(T::Hash[String, T.untyped]) }
  def to_api_hash
    { id: id, name: @name, email: @email }
  end
end
```

### 4.3 Sorbet Advanced Types

```ruby
# typed: strict

# Generic types ด้วย T::Generic
class Stack
  extend T::Generic
  extend T::Sig
  
  Elem = type_member
  
  sig { void }
  def initialize
    @data = T.let([], T::Array[Elem])
  end
  
  sig { params(item: Elem).void }
  def push(item)
    @data.push(item)
  end
  
  sig { returns(Elem) }
  def pop
    raise "Stack empty" if @data.empty?
    T.must(@data.pop)
  end
  
  sig { returns(T::Boolean) }
  def empty?
    @data.empty?
  end
end

# ใช้งาน
int_stack = Stack[Integer].new
int_stack.push(1)
int_stack.push(2)
int_stack.pop  # => 2: Integer

# Type Aliases
Percentage = T.type_alias { Float }
UserId = T.type_alias { Integer }
UserOrGroup = T.type_alias { T.any(User, Group) }

# Interfaces
module Serializable
  extend T::Sig
  extend T::Helpers
  
  interface!
  
  sig { abstract.returns(String) }
  def serialize; end
  
  sig { abstract.params(data: String).returns(T.attached_class) }
  def self.deserialize(data); end
end

class User
  include Serializable
  
  sig { override.returns(String) }
  def serialize
    JSON.generate(to_hash)
  end
end
```

### 4.4 T::Struct และ T::Enum

```ruby
# typed: true

# T::Struct - type-safe struct
class Address < T::Struct
  const :street, String
  const :city, String
  const :state, String
  const :zip, String
  prop :country, String, default: "US"  # mutable with default
end

addr = Address.new(street: "123 Main St", city: "Springfield", state: "IL", zip: "62701")
puts addr.city  # "Springfield"
# addr.city = "Chicago"  # Error! const ไม่ mutable

# T::Enum
class Status < T::Enum
  enums do
    Pending = new
    Active = new
    Suspended = new
    Deleted = new
  end
end

def process_user(status)
  case status
  when Status::Active
    puts "User is active"
  when Status::Pending
    puts "User is pending"
  else
    puts "User status: #{status}"
  end
end

process_user(Status::Active)
```

### 4.5 Sorbet กับ Rails

```ruby
# Gemfile
gem 'sorbet-rails'  # Rails-specific Sorbet helpers

# Generate RBI files สำหรับ Rails + gems
bundle exec tapioca gems

# Generate RBI สำหรับ Rails DSL
bundle exec tapioca dsl

# app/models/post.rb
# typed: true

class Post < ApplicationRecord
  extend T::Sig
  
  belongs_to :user
  has_many :comments
  has_many :tags, through: :post_tags
  
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  
  validates :title, presence: true, length: { maximum: 200 }
  validates :body, presence: true
  
  sig { returns(T::Boolean) }
  def published?
    published_at.present? && published_at <= Time.current
  end
  
  sig { params(length: Integer).returns(String) }
  def excerpt(length = 150)
    body.truncate(length)
  end
  
  sig { returns(Integer) }
  def reading_time
    word_count = body.split.count
    (word_count / 200.0).ceil  # Assuming 200 WPM
  end
end
```

### 4.6 Sorbet ใน Services

```ruby
# typed: strict

class OrderService
  extend T::Sig
  
  class OrderNotFoundError < StandardError; end
  class PaymentFailedError < StandardError; end
  
  sig { params(user: User).void }
  def initialize(user)
    @user = T.let(user, User)
  end
  
  sig {
    params(
      items: T::Array[{ product_id: Integer, quantity: Integer }],
      shipping_address: Address,
      payment_method: String
    ).returns(Order)
  }
  def create_order(items:, shipping_address:, payment_method:)
    validate_items!(items)
    
    order = T.let(
      Order.new(
        user: @user,
        shipping_address: shipping_address.to_json,
        status: 'pending'
      ),
      Order
    )
    
    items.each do |item|
      product = T.must(Product.find_by(id: item[:product_id]))
      order.order_items.build(
        product: product,
        quantity: item[:quantity],
        unit_price: product.price
      )
    end
    
    process_payment!(order, payment_method)
    
    order.save!
    order
  end
  
  sig { params(order_id: Integer).returns(Order) }
  def cancel_order(order_id)
    order = T.must(Order.find_by(id: order_id))
    raise OrderNotFoundError unless order
    
    order.cancel!
    order
  end
  
  private
  
  sig {
    params(items: T::Array[{ product_id: Integer, quantity: Integer }]).void
  }
  def validate_items!(items)
    raise ArgumentError, "Order must have at least one item" if items.empty?
    
    items.each do |item|
      product = Product.find_by(id: item[:product_id])
      raise ArgumentError, "Product #{item[:product_id]} not found" unless product
      raise ArgumentError, "Insufficient stock" if product.stock < item[:quantity]
    end
  end
  
  sig { params(order: Order, payment_method: String).void }
  def process_payment!(order, payment_method)
    result = PaymentService.new(@user).charge!(
      amount: order.total_cents,
      description: "Order ##{order.id}"
    )
    
    raise PaymentFailedError, "Payment failed" unless result[:success]
    
    order.payment_transaction_id = result[:transaction_id]
  end
end
```

## 5. TypeProf: Type Inference

### 5.1 TypeProf Basics

```ruby
# TypeProf วิเคราะห์ Ruby code และ infer types โดยอัตโนมัติ
# ไม่ต้องเขียน annotations - TypeProf คิดให้

# calculator.rb
class Calculator
  def add(a, b)
    a + b
  end
  
  def multiply(a, b)
    a * b
  end
  
  def divide(a, b)
    return nil if b == 0
    a.to_f / b
  end
end

calc = Calculator.new
x = calc.add(1, 2)     # TypeProf infers: Integer
y = calc.add(1.0, 2.0) # TypeProf infers: Float
z = calc.divide(10, 0) # TypeProf infers: nil | Float
```

```bash
# รัน TypeProf
typeprof calculator.rb

# Output:
# class Calculator
#   def add: (Integer, Integer) -> Integer
#           | (Float, Float) -> Float
#           | (Integer | Float, Integer | Float) -> (Integer | Float)
#   
#   def multiply: (Integer, Integer) -> Integer
#               | (Float, Float) -> Float
#   
#   def divide: (Integer | Float, (0 | Integer | Float)) -> (nil | Float)
# end
```

### 5.2 TypeProf สำหรับ Rails

```bash
# Generate RBS จาก Rails models
typeprof app/models/user.rb -o sig/models/user.rbs

# Options:
# -o output file
# -v verbose mode
# --max-rec limit recursion depth
# --timeout timeout in seconds

# ใช้ TypeProf Diagnostic เพื่อ find errors
typeprof --diagnostic app/services/user_service.rb
```

```ruby
# ตัวอย่างผลลัพธ์จาก TypeProf
# user_service.rb
class UserService
  def create(email:, password:, name:)
    User.create!(email: email, password: password, name: name)
  end
  
  def authenticate(email:, password:)
    user = User.find_by(email: email)
    return nil unless user
    user if user.authenticate(password)
  end
end

# TypeProf output:
# class UserService
#   def create: (email: String, password: String, name: String) -> User
#   def authenticate: (email: String, password: String) -> User?
# end
```

### 5.3 TypeProf IDE Extension

```json
// .vscode/settings.json
{
  "ruby.typecheck": {
    "enable": true,
    "engine": "typeprof"
  }
}

// หรือใช้กับ ruby-lsp
{
  "rubyLsp.enabledFeatures": {
    "diagnostics": true,
    "typeInference": true
  },
  "rubyLsp.addon.typeprof": true
}
```

## 6. Type-safe Rails ด้วย Sorbet

### 6.1 Gradual Typing Strategy

```ruby
# Strategy สำหรับ add Sorbet ทีละน้อย:
# Phase 1: ตั้งค่า Sorbet ทั้งหมดเป็น typed: false
# Phase 2: เพิ่ม typed: true ใน utility classes และ services
# Phase 3: เพิ่ม typed: strict ใน critical paths
# Phase 4: เพิ่ม typed: strong ถ้าต้องการ

# Sorbet configuration file: sorbet/config
--dir=.
--ignore=vendor
--ignore=node_modules
--ignore=db/schema.rb

# การ classify ไฟล์
# lib/**/*.rb -> typed: true (start here)
# app/services/**/*.rb -> typed: true
# app/models/**/*.rb -> typed: true (after tapioca generates RBI)
# app/controllers/**/*.rb -> typed: true
# spec/**/*.rb -> typed: false (or true with rspec types)
```

### 6.2 Controllers ด้วย Sorbet

```ruby
# typed: true

class Api::V1::UsersController < ApplicationController
  extend T::Sig
  
  before_action :authenticate_user!
  before_action :set_user, only: [:show, :update, :destroy]
  
  sig { void }
  def index
    @users = User.active.page(params[:page]).per(20)
    render json: @users.map { |u| serialize_user(u) }
  end
  
  sig { void }
  def show
    render json: serialize_user(T.must(@user))
  end
  
  sig { void }
  def create
    user = User.new(user_params)
    
    if user.save
      render json: serialize_user(user), status: :created
    else
      render json: { errors: user.errors.full_messages }, status: :unprocessable_entity
    end
  end
  
  sig { void }
  def update
    user = T.must(@user)
    
    if user.update(user_params)
      render json: serialize_user(user)
    else
      render json: { errors: user.errors.full_messages }, status: :unprocessable_entity
    end
  end
  
  private
  
  sig { returns(T.nilable(User)) }
  def set_user
    @user = User.find_by(id: params[:id])
    head :not_found unless @user
  end
  
  sig { returns(ActionController::Parameters) }
  def user_params
    params.require(:user).permit(:name, :email, :role)
  end
  
  sig { params(user: User).returns(T::Hash[Symbol, T.untyped]) }
  def serialize_user(user)
    {
      id: user.id,
      name: user.name,
      email: user.email,
      created_at: user.created_at
    }
  end
end
```

### 6.3 Type-safe Form Objects

```ruby
# typed: strict

class UserRegistrationForm
  extend T::Sig
  include ActiveModel::Model
  include ActiveModel::Validations
  
  sig { returns(String) }
  attr_accessor :name
  
  sig { returns(String) }
  attr_accessor :email
  
  sig { returns(String) }
  attr_accessor :password
  
  sig { returns(String) }
  attr_accessor :password_confirmation
  
  validates :name, presence: true, length: { minimum: 2, maximum: 100 }
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validate :passwords_match
  
  sig { returns(T.nilable(User)) }
  def save
    return nil unless valid?
    
    User.create!(
      name: name,
      email: email,
      password: password
    )
  rescue ActiveRecord::RecordInvalid
    nil
  end
  
  private
  
  sig { void }
  def passwords_match
    if password != password_confirmation
      errors.add(:password_confirmation, "doesn't match password")
    end
  end
end
```

## 7. RBI Files และ Tapioca

### 7.1 Tapioca gem

```ruby
# Gemfile
group :development do
  gem 'tapioca'
  gem 'sorbet'
  gem 'sorbet-runtime'
end
```

```bash
# Setup Tapioca
bundle exec tapioca init

# Generate RBI สำหรับทุก gems
bundle exec tapioca gems

# Generate RBI สำหรับ Rails DSLs (routes, helpers, etc.)
bundle exec tapioca dsl

# Update หลัง add new gem
bundle exec tapioca gems --update

# รัน type checking
bundle exec srb tc
```

### 7.2 Custom RBI Files

```ruby
# sorbet/rbi/custom/stripe.rbi
# typed: true

module Stripe
  class Charge < APIResource
    sig { returns(String) }
    def id; end
    
    sig { returns(Integer) }
    def amount; end
    
    sig { returns(String) }
    def currency; end
    
    sig { returns(String) }
    def status; end
    
    sig {
      params(
        amount: Integer,
        currency: String,
        customer: String,
        description: T.nilable(String)
      ).returns(Charge)
    }
    def self.create(amount:, currency:, customer:, description: nil); end
  end
  
  class Customer < APIResource
    sig { returns(String) }
    def id; end
    
    sig { returns(String) }
    def email; end
    
    sig {
      params(email: String, name: T.nilable(String)).returns(Customer)
    }
    def self.create(email:, name: nil); end
    
    sig { params(customer_id: String).returns(Customer) }
    def self.retrieve(customer_id); end
  end
end
```

## 8. Practical Type Safety Patterns

### 8.1 Value Objects ด้วย Type Safety

```ruby
# typed: strict

class Money
  extend T::Sig
  
  include Comparable
  
  sig { returns(Integer) }
  attr_reader :cents
  
  sig { returns(String) }
  attr_reader :currency
  
  sig { params(cents: Integer, currency: String).void }
  def initialize(cents, currency = "USD")
    raise ArgumentError, "Cents must be non-negative" if cents < 0
    @cents = T.let(cents, Integer)
    @currency = T.let(currency, String)
  end
  
  sig { params(other: Money).returns(Money) }
  def +(other)
    validate_same_currency!(other)
    Money.new(@cents + other.cents, @currency)
  end
  
  sig { params(other: Money).returns(Money) }
  def -(other)
    validate_same_currency!(other)
    Money.new(@cents - other.cents, @currency)
  end
  
  sig { params(multiplier: Numeric).returns(Money) }
  def *(multiplier)
    Money.new((@cents * multiplier).round, @currency)
  end
  
  sig { returns(Float) }
  def to_f
    @cents / 100.0
  end
  
  sig { returns(String) }
  def to_s
    "#{@currency} #{'%.2f' % to_f}"
  end
  
  sig { params(other: T.untyped).returns(Integer) }
  def <=>(other)
    return nil unless other.is_a?(Money) && other.currency == @currency
    @cents <=> other.cents
  end
  
  sig { params(other: Money).void }
  private def validate_same_currency!(other)
    unless @currency == other.currency
      raise ArgumentError, "Cannot operate on different currencies: #{@currency} and #{other.currency}"
    end
  end
end

# ทดสอบ
price = Money.new(1000, "USD")
tax = Money.new(80, "USD")
total = price + tax
puts total  # USD 10.80
```

### 8.2 Result Types สำหรับ Error Handling

```ruby
# typed: strict

# Result monad สำหรับ type-safe error handling
class Result
  extend T::Sig
  extend T::Generic
  
  T_result = type_member
  T_error = type_member
  
  sig { params(value: T_result).returns(Result[T_result, T.untyped]) }
  def self.success(value)
    Success.new(value)
  end
  
  sig { params(error: T_error).returns(Result[T.untyped, T_error]) }
  def self.failure(error)
    Failure.new(error)
  end
end

class Success < Result
  extend T::Sig
  extend T::Generic
  
  T_result = type_member
  T_error = type_member { { fixed: NilClass } }
  
  sig { params(value: T_result).void }
  def initialize(value)
    @value = T.let(value, T_result)
  end
  
  sig { returns(T::Boolean) }
  def success? = true
  
  sig { returns(T_result) }
  def value = @value
  
  sig { returns(nil) }
  def error = nil
  
  sig {
    type_parameters(:U)
    .params(block: T.proc.params(value: T_result).returns(T.type_parameter(:U)))
    .returns(T.type_parameter(:U))
  }
  def map(&block)
    block.call(@value)
  end
end

class Failure < Result
  extend T::Sig
  extend T::Generic
  
  T_result = type_member { { fixed: NilClass } }
  T_error = type_member
  
  sig { params(error: T_error).void }
  def initialize(error)
    @error = T.let(error, T_error)
  end
  
  sig { returns(T::Boolean) }
  def success? = false
  
  sig { returns(nil) }
  def value = nil
  
  sig { returns(T_error) }
  def error = @error
  
  sig {
    type_parameters(:U)
    .params(block: T.proc.params(value: NilClass).returns(T.type_parameter(:U)))
    .returns(Failure[T_error])
  }
  def map(&block)
    self  # Don't transform failures
  end
end

# ใช้งาน
def create_user(email:, password:)
  return Result.failure("Email is required") if email.blank?
  return Result.failure("Password too short") if password.length < 8
  
  user = User.create!(email: email, password: password)
  Result.success(user)
rescue ActiveRecord::RecordInvalid => e
  Result.failure(e.message)
end

result = create_user(email: "user@example.com", password: "securepassword")

case result
when Success
  puts "User created: #{result.value.email}"
when Failure
  puts "Error: #{result.error}"
end
```

### 8.3 Type-safe Configuration

```ruby
# typed: strict

class AppConfig
  extend T::Sig
  
  # DSL สำหรับ type-safe config
  class << self
    extend T::Sig
    
    sig { params(key: Symbol, type: T.untyped, default: T.untyped).void }
    def config_option(key, type:, default: nil)
      define_method(key) do
        value = instance_variable_get("@#{key}")
        return value unless value.nil?
        
        env_value = ENV[key.to_s.upcase]
        return default if env_value.nil?
        
        cast_value(env_value, type)
      end
      
      define_method("#{key}=") do |val|
        instance_variable_set("@#{key}", val)
      end
    end
    
    private
    
    def cast_value(value, type)
      case type
      when :string then value
      when :integer then value.to_i
      when :float then value.to_f
      when :boolean then %w[true 1 yes].include?(value.downcase)
      else value
      end
    end
  end
  
  config_option :database_url, type: :string, default: "sqlite3:db/development.sqlite3"
  config_option :redis_url, type: :string, default: "redis://localhost:6379"
  config_option :max_threads, type: :integer, default: 5
  config_option :enable_caching, type: :boolean, default: false
  
  sig { returns(T.self_type) }
  def self.instance
    @instance ||= new
  end
end

config = AppConfig.instance
puts config.max_threads  # 5 (Integer)
puts config.enable_caching  # false (Boolean)
```

## 9. Testing ด้วย Type Safety

### 9.1 Type Checking ใน RSpec

```ruby
# spec/support/type_checking.rb
# typed: true

RSpec.shared_examples "returns the correct type" do |expected_type|
  it "returns #{expected_type}" do
    expect(subject).to be_a(expected_type)
  end
end

# spec/models/user_spec.rb
# typed: true

RSpec.describe User do
  subject { build(:user) }
  
  describe "#greeting" do
    subject { described_class.new(name: "John").greeting }
    it_behaves_like "returns the correct type", String
    
    it "returns a greeting" do
      expect(subject).to eq("Hello, John!")
    end
  end
  
  describe "#reading_time" do
    it "returns an integer" do
      post = build(:post, body: "word " * 200)
      expect(post.reading_time).to be_an(Integer)
    end
  end
end
```

### 9.2 Property-based Testing ด้วย Types

```ruby
# Gemfile
gem 'rantly'  # Property-based testing

require 'rantly/rspec_extensions'

RSpec.describe UserService do
  describe ".create" do
    it "always returns a User" do
      property_of do
        {
          name: string(:alpha),
          email: "#{string(:alpha)}@example.com",
          password: "password" + string(:alpha)
        }
      end.check do |params|
        result = UserService.create(**params)
        expect(result).to be_a(User)
      end
    end
    
    it "always sets the email to lowercase" do
      property_of do
        {
          name: string(:alpha),
          email: "#{string(:alpha).upcase}@EXAMPLE.COM",
          password: "password123"
        }
      end.check do |params|
        result = UserService.create(**params)
        expect(result.email).to eq(result.email.downcase)
      end
    end
  end
end
```

## 10. Migration Strategy

### 10.1 Adding Types ทีละน้อย

```bash
# Step 1: เพิ่ม sorbet ทั้งหมดแบบ typed: false ก่อน
# script/add_typed_false.sh
find app -name "*.rb" -exec grep -L "typed:" {} \; | \
  xargs sed -i '' '1s/^/# typed: false\n/'

# Step 2: รัน srb tc เพื่อดู errors ทั้งหมด
bundle exec srb tc

# Step 3: อัพเกรดทีละ file
# เริ่มจาก lib/ และ services/
find lib -name "*.rb" -exec sed -i '' 's/# typed: false/# typed: true/' {} \;

# Step 4: ตรวจสอบอีกครั้ง
bundle exec srb tc
```

### 10.2 Type Coverage Report

```ruby
# lib/tasks/type_coverage.rake
namespace :type do
  desc "Report type coverage"
  task coverage: :environment do
    files = Dir.glob("app/**/*.rb") + Dir.glob("lib/**/*.rb")
    
    stats = {
      ignore: 0,
      false: 0,
      true: 0,
      strict: 0,
      strong: 0,
      unannotated: 0
    }
    
    files.each do |file|
      content = File.read(file)
      
      if content =~ /# typed: (\w+)/
        level = $1.to_sym
        stats[level] = (stats[level] || 0) + 1
      else
        stats[:unannotated] += 1
      end
    end
    
    total = files.length
    typed = stats[:true] + stats[:strict] + stats[:strong]
    
    puts "Type Coverage Report"
    puts "=" * 40
    puts "Total files: #{total}"
    puts "typed: ignore  - #{stats[:ignore]} (#{percent(stats[:ignore], total)}%)"
    puts "typed: false   - #{stats[:false]} (#{percent(stats[:false], total)}%)"
    puts "typed: true    - #{stats[:true]} (#{percent(stats[:true], total)}%)"
    puts "typed: strict  - #{stats[:strict]} (#{percent(stats[:strict], total)}%)"
    puts "typed: strong  - #{stats[:strong]} (#{percent(stats[:strong], total)}%)"
    puts "unannotated    - #{stats[:unannotated]} (#{percent(stats[:unannotated], total)}%)"
    puts "-" * 40
    puts "Coverage: #{percent(typed, total)}%"
  end
  
  private
  
  def percent(n, total)
    return 0 if total == 0
    (n.to_f / total * 100).round(1)
  end
end
```

---

## แบบฝึกหัดบทที่ 92

### แบบฝึกหัดที่ 1: RBS สำหรับ Calculator
**โจทย์:** เขียน RBS signatures สำหรับ Calculator class ที่ handle errors อย่างถูกต้อง

**เฉลย:**
```rbs
# sig/lib/calculator.rbs
class Calculator
  class DivisionByZeroError < StandardError
  end
  
  class InvalidOperationError < StandardError
  end
  
  type operand = Integer | Float
  type result = Integer | Float
  
  def add: (operand, operand) -> result
  def subtract: (operand, operand) -> result
  def multiply: (operand, operand) -> result
  def divide: (operand, operand) -> Float
  
  # Raises DivisionByZeroError if divisor is 0
  def safe_divide: (operand, operand) -> (Float | nil)
  
  # Scientific operations
  def power: (operand, operand) -> Float
  def sqrt: (operand) -> Float
  def factorial: (Integer) -> Integer
  
  # Statistical operations
  def mean: (Array[operand]) -> Float
  def median: (Array[operand]) -> Float
  def variance: (Array[operand]) -> Float
  def std_dev: (Array[operand]) -> Float
  
  # Utility
  def round: (Float, ?Integer precision) -> Float
  def percentage: (operand base, operand percent) -> Float
end
```

### แบบฝึกหัดที่ 2: Sorbet สำหรับ API Client
**โจทย์:** เขียน type-safe API client ด้วย Sorbet

**เฉลย:**
```ruby
# typed: strict

require 'net/http'
require 'json'

class ApiClient
  extend T::Sig
  
  class ApiError < StandardError
    extend T::Sig
    
    sig { returns(Integer) }
    attr_reader :status_code
    
    sig { returns(T::Hash[String, T.untyped]) }
    attr_reader :response_body
    
    sig { params(message: String, status_code: Integer, response_body: T::Hash[String, T.untyped]).void }
    def initialize(message, status_code:, response_body:)
      super(message)
      @status_code = T.let(status_code, Integer)
      @response_body = T.let(response_body, T::Hash[String, T.untyped])
    end
  end
  
  sig { params(base_url: String, api_key: String).void }
  def initialize(base_url:, api_key:)
    @base_url = T.let(base_url, String)
    @api_key = T.let(api_key, String)
    @http_client = T.let(build_http_client, Net::HTTP)
  end
  
  sig {
    params(
      path: String,
      params: T::Hash[String, T.untyped]
    ).returns(T::Hash[String, T.untyped])
  }
  def get(path, params: {})
    uri = build_uri(path, params)
    request = Net::HTTP::Get.new(uri)
    add_auth_header(request)
    
    perform_request(request)
  end
  
  sig {
    params(
      path: String,
      body: T::Hash[String, T.untyped]
    ).returns(T::Hash[String, T.untyped])
  }
  def post(path, body: {})
    uri = build_uri(path)
    request = Net::HTTP::Post.new(uri)
    request.body = body.to_json
    request.content_type = "application/json"
    add_auth_header(request)
    
    perform_request(request)
  end
  
  private
  
  sig { params(path: String, params: T::Hash[String, T.untyped]).returns(URI::HTTP) }
  def build_uri(path, params = {})
    uri = URI("#{@base_url}#{path}")
    uri.query = URI.encode_www_form(params) unless params.empty?
    uri
  end
  
  sig { params(request: Net::HTTPRequest).void }
  def add_auth_header(request)
    request["Authorization"] = "Bearer #{@api_key}"
    request["Accept"] = "application/json"
  end
  
  sig { params(request: Net::HTTPRequest).returns(T::Hash[String, T.untyped]) }
  def perform_request(request)
    response = @http_client.request(request)
    body = T.let(JSON.parse(response.body), T::Hash[String, T.untyped])
    
    unless response.is_a?(Net::HTTPSuccess)
      raise ApiError.new(
        "API request failed",
        status_code: response.code.to_i,
        response_body: body
      )
    end
    
    body
  rescue JSON::ParserError => e
    raise ApiError.new(
      "Invalid JSON response: #{e.message}",
      status_code: 0,
      response_body: {}
    )
  end
  
  sig { returns(Net::HTTP) }
  def build_http_client
    uri = URI(@base_url)
    http = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl = uri.scheme == "https"
    http.read_timeout = 30
    http.open_timeout = 10
    http
  end
end
```

### แบบฝึกหัดที่ 3: RBS สำหรับ Service Object Pattern
**โจทย์:** เขียน RBS สำหรับ Service Object ที่ใช้ result type

**เฉลย:**
```rbs
# sig/services/base_service.rbs
module Services
  class Result
    attr_reader value: untyped
    attr_reader error: String?
    attr_reader metadata: Hash[Symbol, untyped]
    
    def success?: () -> bool
    def failure?: () -> bool
    
    def self.success: (untyped value, ?metadata: Hash[Symbol, untyped]) -> Result
    def self.failure: (String error, ?metadata: Hash[Symbol, untyped]) -> Result
  end
  
  class BaseService
    def self.call: (*untyped args, **untyped kwargs) -> Result
    
    private
      def success: (untyped value, ?metadata: Hash[Symbol, untyped]) -> Result
      def failure: (String error, ?metadata: Hash[Symbol, untyped]) -> Result
  end
end

# sig/services/create_post_service.rbs
class CreatePostService < Services::BaseService
  @user: User
  @params: Hash[Symbol, untyped]
  
  def initialize: (user: User, params: Hash[Symbol, untyped]) -> void
  
  def call: () -> Services::Result
  
  private
    def validate_params: () -> bool
    def build_post: () -> Post
    def notify_followers: (Post post) -> void
end
```

### แบบฝึกหัดที่ 4: Sorbet T::Struct
**โจทย์:** แปลง plain Ruby class เป็น T::Struct

**เฉลย:**
```ruby
# typed: strict

# Before: Plain Ruby class
class OldAddress
  attr_accessor :street, :city, :state, :zip, :country
  
  def initialize(street:, city:, state:, zip:, country: "US")
    @street = street
    @city = city
    @state = state
    @zip = zip
    @country = country
  end
  
  def full_address
    "#{@street}, #{@city}, #{@state} #{@zip}, #{@country}"
  end
end

# After: T::Struct
class Address < T::Struct
  extend T::Sig
  
  const :street, String
  const :city, String
  const :state, String
  const :zip, String
  prop :country, String, default: "US"
  
  sig { returns(String) }
  def full_address
    "#{street}, #{city}, #{state} #{zip}, #{country}"
  end
  
  sig { returns(T::Hash[Symbol, String]) }
  def to_h
    { street: street, city: city, state: state, zip: zip, country: country }
  end
  
  sig { params(data: T::Hash[Symbol, String]).returns(T.attached_class) }
  def self.from_h(data)
    new(
      street: T.must(data[:street]),
      city: T.must(data[:city]),
      state: T.must(data[:state]),
      zip: T.must(data[:zip]),
      country: data.fetch(:country, "US")
    )
  end
end

# ใช้งาน
addr = Address.new(
  street: "123 Main St",
  city: "Springfield",
  state: "IL",
  zip: "62701"
)
puts addr.full_address
# addr.street = "456 Oak Ave"  # Error! const ไม่ mutable
```

### แบบฝึกหัดที่ 5: TypeProf สำหรับ Existing Code
**โจทย์:** รัน TypeProf บน existing code และ generate RBS

**เฉลย:**
```ruby
# existing_code.rb - code ที่มีอยู่แล้ว
class WeatherService
  def initialize(api_key)
    @api_key = api_key
  end
  
  def current_weather(city)
    response = fetch_weather(city)
    parse_response(response)
  end
  
  def forecast(city, days: 5)
    response = fetch_forecast(city, days)
    response["list"].map { |item| parse_forecast_item(item) }
  end
  
  private
  
  def fetch_weather(city)
    # HTTP request...
    { "main" => { "temp" => 25.5, "humidity" => 60 }, 
      "weather" => [{ "description" => "clear sky" }] }
  end
  
  def fetch_forecast(city, days)
    { "list" => Array.new(days) { |i| { "dt" => Time.now.to_i + i * 86400, "main" => { "temp" => 20 + i } } } }
  end
  
  def parse_response(data)
    {
      temperature: data["main"]["temp"],
      humidity: data["main"]["humidity"],
      description: data["weather"][0]["description"]
    }
  end
  
  def parse_forecast_item(item)
    {
      date: Time.at(item["dt"]),
      temperature: item["main"]["temp"]
    }
  end
end

# typeprof existing_code.rb
# Output:
class WeatherService
  # @api_key: String
  def initialize: (String api_key) -> void
  def current_weather: (String city) -> Hash[Symbol, Float | Integer | String]
  def forecast: (String city, ?days: Integer) -> Array[Hash[Symbol, Time | Float | Integer]]
  
  private
  def fetch_weather: (String city) -> Hash[String, Hash[String, Float | Integer] | Array[Hash[String, String]]]
  def fetch_forecast: (String city, Integer days) -> Hash[String, Array[Hash[String, Hash[String, Integer] | Integer]]]
  def parse_response: (Hash[String, untyped] data) -> Hash[Symbol, Float | Integer | String]
  def parse_forecast_item: (Hash[String, untyped] item) -> Hash[Symbol, Time | Float | Integer]
end
```

### แบบฝึกหัดที่ 6: Type Guards
**โจทย์:** Implement type guards ด้วย Sorbet

**เฉลย:**
```ruby
# typed: strict

module TypeGuards
  extend T::Sig
  
  sig { params(value: T.untyped).returns(T::Boolean) }
  def self.string?(value)
    value.is_a?(String)
  end
  
  sig { params(value: T.untyped).returns(T::Boolean) }
  def self.integer?(value)
    value.is_a?(Integer)
  end
  
  sig { params(value: T.untyped).returns(T::Boolean) }
  def self.array_of_strings?(value)
    value.is_a?(Array) && value.all? { |item| item.is_a?(String) }
  end
  
  sig { params(value: T.untyped).returns(T::Boolean) }
  def self.hash_with_string_keys?(value)
    value.is_a?(Hash) && value.keys.all? { |k| k.is_a?(String) }
  end
end

class SafeParser
  extend T::Sig
  
  sig { params(data: T.untyped).returns(T.nilable(String)) }
  def parse_name(data)
    return nil unless TypeGuards.string?(data)
    
    # Sorbet now knows data is String
    T.cast(data, String).strip.presence
  end
  
  sig { params(data: T.untyped).returns(T::Array[String]) }
  def parse_tags(data)
    return [] unless TypeGuards.array_of_strings?(data)
    
    T.cast(data, T::Array[String]).map(&:downcase).uniq
  end
  
  sig { params(json_string: String).returns(T.nilable(T::Hash[String, T.untyped])) }
  def parse_json(json_string)
    parsed = JSON.parse(json_string)
    return nil unless TypeGuards.hash_with_string_keys?(parsed)
    
    T.cast(parsed, T::Hash[String, T.untyped])
  rescue JSON::ParserError
    nil
  end
end
```

### แบบฝึกหัดที่ 7: Generic Collections
**โจทย์:** สร้าง type-safe generic repository

**เฉลย:**
```ruby
# typed: strict

class Repository
  extend T::Sig
  extend T::Generic
  
  Model = type_member { { upper: ActiveRecord::Base } }
  
  sig { params(model_class: T::Class[Model]).void }
  def initialize(model_class)
    @model_class = T.let(model_class, T::Class[Model])
  end
  
  sig { params(id: Integer).returns(Model) }
  def find(id)
    @model_class.find(id)
  end
  
  sig { params(id: Integer).returns(T.nilable(Model)) }
  def find_by_id(id)
    @model_class.find_by(id: id)
  end
  
  sig { params(conditions: T::Hash[Symbol, T.untyped]).returns(T::Array[Model]) }
  def where(conditions)
    @model_class.where(conditions).to_a
  end
  
  sig { params(attributes: T::Hash[Symbol, T.untyped]).returns(Model) }
  def create!(attributes)
    @model_class.create!(attributes)
  end
  
  sig { params(model: Model, attributes: T::Hash[Symbol, T.untyped]).returns(Model) }
  def update!(model, attributes)
    model.update!(attributes)
    model
  end
  
  sig { params(model: Model).void }
  def destroy!(model)
    model.destroy!
  end
  
  sig { returns(Integer) }
  def count
    @model_class.count
  end
  
  sig { returns(T::Array[Model]) }
  def all
    @model_class.all.to_a
  end
end

# ใช้งาน
user_repo = Repository[User].new(User)
post_repo = Repository[Post].new(Post)

users = user_repo.all  # T::Array[User]
user = user_repo.find(1)  # User
posts = post_repo.where(published: true)  # T::Array[Post]
```

### แบบฝึกหัดที่ 8: RBS สำหรับ Mixins
**โจทย์:** เขียน RBS สำหรับ ActiveSupport Concern

**เฉลย:**
```rbs
# sig/models/concerns/auditable.rbs
module Auditable
  def self.included: (Module base) -> void
  
  module ClassMethods
    def audited: (?only: Array[Symbol], ?except: Array[Symbol]) -> void
    def with_auditing: () { () -> void } -> void
    def without_auditing: () { () -> void } -> void
    
    def audits: () -> ActiveRecord::Associations::CollectionProxy[Audit]
  end
  
  def audits: () -> ActiveRecord::Associations::CollectionProxy[Audit]
  def latest_audit: () -> Audit?
  
  def current_version: () -> Integer
  def versions: () -> Array[Hash[String, untyped]]
  
  def restore_version!: (Integer version) -> void
  def diff_with_version: (Integer version) -> Hash[String, Array[untyped]]
end

# sig/models/audit.rbs
class Audit < ApplicationRecord
  attr_reader auditable_id: Integer
  attr_reader auditable_type: String
  attr_reader user_id: Integer?
  attr_reader action: String
  attr_reader audited_changes: Hash[String, untyped]
  attr_reader version: Integer
  attr_reader created_at: Time
  
  def auditable: () -> ActiveRecord::Base
  def user: () -> User?
  
  def self.for: (ActiveRecord::Base record) -> ActiveRecord::Relation
end
```

### แบบฝึกหัดที่ 9: Steep กับ Rails Routes
**โจทย์:** Setup Steep สำหรับ type-checking Rails routes helpers

**เฉลย:**
```ruby
# Steepfile
target :app do
  check "app/controllers"
  check "app/helpers"
  check "app/views/helpers"
  
  signature "sig"
  
  # Rails route helpers
  library "rails-routes"
  
  configure_code_diagnostics do |hash|
    # Route helpers - warning level
    hash[D::Ruby::NoMethod] = :warning
  end
end

# sig/routes.rbs (generated by rbs-rails)
# bundle exec rails rbs_rails:routes

module Routes
  def users_path: (**untyped options) -> String
  def user_path: (Integer | User id, **untyped options) -> String
  def new_user_path: (**untyped options) -> String
  def edit_user_path: (Integer | User id, **untyped options) -> String
  
  def users_url: (**untyped options) -> String
  def user_url: (Integer | User id, **untyped options) -> String
end

# app/controllers/application_controller.rb
# typed: true

class ApplicationController < ActionController::Base
  include Routes
  
  sig { returns(String) }
  def after_login_path
    users_path  # type-checked!
  end
end
```

### แบบฝึกหัดที่ 10: Complete Type-safe Service
**โจทย์:** สร้าง complete type-safe notification service

**เฉลย:**
```ruby
# typed: strict

class NotificationService
  extend T::Sig
  
  # Notification types
  class NotificationType < T::Enum
    enums do
      Email = new("email")
      Push = new("push")
      SMS = new("sms")
      InApp = new("in_app")
    end
  end
  
  # Priority levels
  class Priority < T::Enum
    enums do
      Low = new
      Medium = new
      High = new
      Critical = new
    end
  end
  
  # Notification data structure
  class NotificationData < T::Struct
    const :user_id, Integer
    const :type, NotificationType
    const :title, String
    const :body, String
    const :priority, Priority, default: Priority::Medium
    prop :metadata, T::Hash[String, T.untyped], default: {}
    prop :scheduled_at, T.nilable(Time), default: nil
  end
  
  # Result type
  class DeliveryResult < T::Struct
    const :success, T::Boolean
    const :notification_id, T.nilable(String)
    const :error_message, T.nilable(String)
    const :delivered_at, T.nilable(Time)
  end
  
  sig { params(user: User).void }
  def initialize(user)
    @user = T.let(user, User)
  end
  
  sig { params(data: NotificationData).returns(DeliveryResult) }
  def deliver(data)
    return delivery_skipped("User not subscribed to #{data.type}") unless subscribed?(data.type)
    
    result = case data.type
    when NotificationType::Email
      deliver_email(data)
    when NotificationType::Push
      deliver_push(data)
    when NotificationType::SMS
      deliver_sms(data)
    when NotificationType::InApp
      deliver_in_app(data)
    else
      T.absurd(data.type)
    end
    
    log_delivery(data, result)
    result
  end
  
  sig {
    params(
      type: NotificationType,
      title: String,
      body: String,
      priority: Priority,
      metadata: T::Hash[String, T.untyped]
    ).returns(DeliveryResult)
  }
  def notify(type:, title:, body:, priority: Priority::Medium, metadata: {})
    data = NotificationData.new(
      user_id: @user.id,
      type: type,
      title: title,
      body: body,
      priority: priority,
      metadata: metadata
    )
    
    deliver(data)
  end
  
  sig {
    params(
      title: String,
      body: String,
      priority: Priority
    ).returns(T::Array[DeliveryResult])
  }
  def broadcast(title:, body:, priority: Priority::Medium)
    NotificationType.each_value.map do |type|
      notify(type: type, title: title, body: body, priority: priority)
    end
  end
  
  private
  
  sig { params(type: NotificationType).returns(T::Boolean) }
  def subscribed?(type)
    preference = @user.notification_preferences.find_by(
      notification_type: type.serialize
    )
    preference&.enabled? || false
  end
  
  sig { params(data: NotificationData).returns(DeliveryResult) }
  def deliver_email(data)
    result = UserMailer.notification(@user, data.title, data.body).deliver_now
    
    DeliveryResult.new(
      success: true,
      notification_id: result.message_id,
      error_message: nil,
      delivered_at: Time.current
    )
  rescue => e
    DeliveryResult.new(
      success: false,
      notification_id: nil,
      error_message: e.message,
      delivered_at: nil
    )
  end
  
  sig { params(data: NotificationData).returns(DeliveryResult) }
  def deliver_push(data)
    notification = Notification.create!(
      user: @user,
      title: data.title,
      body: data.body,
      notification_type: data.type.serialize,
      metadata: data.metadata
    )
    
    DeliveryResult.new(
      success: true,
      notification_id: notification.id.to_s,
      error_message: nil,
      delivered_at: Time.current
    )
  rescue => e
    DeliveryResult.new(
      success: false,
      notification_id: nil,
      error_message: e.message,
      delivered_at: nil
    )
  end
  
  sig { params(data: NotificationData).returns(DeliveryResult) }
  def deliver_sms(data)
    return delivery_skipped("Phone number not available") unless @user.phone.present?
    
    TwilioService.send_sms(
      to: T.must(@user.phone),
      body: "#{data.title}: #{data.body}"
    )
    
    DeliveryResult.new(
      success: true,
      notification_id: SecureRandom.uuid,
      error_message: nil,
      delivered_at: Time.current
    )
  rescue => e
    DeliveryResult.new(
      success: false,
      notification_id: nil,
      error_message: e.message,
      delivered_at: nil
    )
  end
  
  sig { params(data: NotificationData).returns(DeliveryResult) }
  def deliver_in_app(data)
    InAppNotification.create!(
      user: @user,
      title: data.title,
      body: data.body,
      metadata: data.metadata,
      priority: data.priority.serialize
    )
    
    DeliveryResult.new(
      success: true,
      notification_id: SecureRandom.uuid,
      error_message: nil,
      delivered_at: Time.current
    )
  rescue => e
    DeliveryResult.new(
      success: false,
      notification_id: nil,
      error_message: e.message,
      delivered_at: nil
    )
  end
  
  sig { params(reason: String).returns(DeliveryResult) }
  def delivery_skipped(reason)
    DeliveryResult.new(
      success: false,
      notification_id: nil,
      error_message: reason,
      delivered_at: nil
    )
  end
  
  sig { params(data: NotificationData, result: DeliveryResult).void }
  def log_delivery(data, result)
    Rails.logger.info(
      "Notification delivery: user=#{@user.id} type=#{data.type} " \
      "success=#{result.success} notification_id=#{result.notification_id}"
    )
  end
end
```

### แบบฝึกหัดที่ 11: T::Enum สำหรับ State Machine
**โจทย์:** Implement type-safe state machine ด้วย T::Enum

**เฉลย:**
```ruby
# typed: strict

class OrderStateMachine
  extend T::Sig
  
  class State < T::Enum
    enums do
      Pending = new("pending")
      Confirmed = new("confirmed")
      Processing = new("processing")
      Shipped = new("shipped")
      Delivered = new("delivered")
      Cancelled = new("cancelled")
      Refunded = new("refunded")
    end
  end
  
  VALID_TRANSITIONS = T.let({
    State::Pending => [State::Confirmed, State::Cancelled],
    State::Confirmed => [State::Processing, State::Cancelled],
    State::Processing => [State::Shipped, State::Cancelled],
    State::Shipped => [State::Delivered],
    State::Delivered => [State::Refunded],
    State::Cancelled => [],
    State::Refunded => []
  }, T::Hash[State, T::Array[State]])
  
  sig { params(order: Order).void }
  def initialize(order)
    @order = T.let(order, Order)
  end
  
  sig { returns(State) }
  def current_state
    State.deserialize(@order.status)
  end
  
  sig { params(new_state: State).returns(T::Boolean) }
  def can_transition_to?(new_state)
    allowed = VALID_TRANSITIONS[current_state]
    T.must(allowed).include?(new_state)
  end
  
  sig { params(new_state: State).returns(T::Boolean) }
  def transition_to!(new_state)
    raise InvalidTransitionError unless can_transition_to?(new_state)
    
    @order.update!(status: new_state.serialize)
    trigger_callbacks(new_state)
    true
  end
  
  sig { returns(T::Array[State]) }
  def available_transitions
    T.must(VALID_TRANSITIONS[current_state])
  end
  
  private
  
  sig { params(state: State).void }
  def trigger_callbacks(state)
    case state
    when State::Confirmed
      OrderMailer.confirmation(@order).deliver_later
    when State::Shipped
      OrderMailer.shipped(@order).deliver_later
    when State::Delivered
      @order.update!(delivered_at: Time.current)
    when State::Cancelled
      OrderMailer.cancellation(@order).deliver_later
      initiate_refund(@order) if @order.payment_captured?
    end
  end
  
  sig { params(order: Order).void }
  def initiate_refund(order)
    RefundService.new(order).process!
  end
end
```

### แบบฝึกหัดที่ 12: Type-safe Observer Pattern
**โจทย์:** Implement type-safe Observer/Event System

**เฉลย:**
```ruby
# typed: strict

module TypedEvents
  extend T::Generic
  extend T::Sig
  
  EventData = type_member
  
  class EventBus
    extend T::Sig
    extend T::Generic
    
    E = type_member
    
    sig { void }
    def initialize
      @handlers = T.let(
        Hash.new { |h, k| h[k] = [] },
        T::Hash[String, T::Array[T.proc.params(event: E).void]]
      )
    end
    
    sig {
      params(
        event_name: String,
        handler: T.proc.params(event: E).void
      ).void
    }
    def subscribe(event_name, &handler)
      @handlers[event_name] << handler
    end
    
    sig { params(event_name: String, event: E).void }
    def publish(event_name, event)
      @handlers[event_name].each { |handler| handler.call(event) }
    end
    
    sig { params(event_name: String).void }
    def unsubscribe_all(event_name)
      @handlers.delete(event_name)
    end
  end
  
  # Type-safe event definitions
  class UserCreatedEvent < T::Struct
    const :user_id, Integer
    const :email, String
    const :name, String
    const :created_at, Time
  end
  
  class OrderPlacedEvent < T::Struct
    const :order_id, Integer
    const :user_id, Integer
    const :total_cents, Integer
    const :items_count, Integer
    const :created_at, Time
  end
  
  # Global event bus instances
  USER_BUS = T.let(EventBus[UserCreatedEvent].new, EventBus[UserCreatedEvent])
  ORDER_BUS = T.let(EventBus[OrderPlacedEvent].new, EventBus[OrderPlacedEvent])
end

# ใช้งาน
TypedEvents::USER_BUS.subscribe("user.created") do |event|
  UserMailer.welcome(event.user_id).deliver_later
  Analytics.track("User Signed Up", user_id: event.user_id)
end

TypedEvents::ORDER_BUS.subscribe("order.placed") do |event|
  # Type-safe access
  puts "Order #{event.order_id} placed by user #{event.user_id}"
  InventoryService.reserve_items(event.order_id)
end
```

### แบบฝึกหัดที่ 13: Gradual Type Adoption
**โจทย์:** วางแผน gradual type adoption สำหรับ existing Rails app

**เฉลย:**
```ruby
# lib/tasks/type_adoption.rake
namespace :types do
  desc "Show type adoption status"
  task status: :environment do
    files = Dir.glob("{app,lib}/**/*.rb")
    
    stats = Hash.new(0)
    untyped_files = []
    
    files.each do |file|
      content = File.read(file)
      
      if content =~ /# typed: (\w+)/
        stats[$1] += 1
      else
        stats['untyped'] += 1
        untyped_files << file
      end
    end
    
    total = files.length
    
    puts "Type Adoption Status"
    puts "=" * 50
    
    %w[strong strict true false ignore].each do |level|
      count = stats[level]
      puts "typed: #{level.ljust(10)} #{count.to_s.rjust(5)} (#{(count.to_f / total * 100).round(1)}%)"
    end
    
    puts "untyped          #{stats['untyped'].to_s.rjust(5)} (#{(stats['untyped'].to_f / total * 100).round(1)}%)"
    puts "-" * 50
    puts "Total           #{total.to_s.rjust(6)}"
    
    puts "\nPriority files to add types (most complex first):"
    complex_files = untyped_files.map do |f|
      [f, File.read(f).scan(/def /).length]
    end.sort_by { |_, count| -count }.first(10)
    
    complex_files.each { |f, count| puts "  #{f} (#{count} methods)" }
  end
  
  desc "Add typed: false to all unannotated files"
  task add_false: :environment do
    files = Dir.glob("{app,lib}/**/*.rb")
    
    modified = 0
    files.each do |file|
      content = File.read(file)
      
      unless content.start_with?("# typed:")
        File.write(file, "# typed: false\n#{content}")
        modified += 1
      end
    end
    
    puts "Added 'typed: false' to #{modified} files"
  end
  
  desc "Upgrade files from typed: false to typed: true"
  task upgrade: :environment do
    # รัน srb tc และ upgrade files ที่ไม่มี errors
    files = Dir.glob("lib/**/*.rb")
      .select { |f| File.read(f).start_with?("# typed: false") }
    
    upgraded = 0
    
    files.each do |file|
      content = File.read(file)
      test_content = content.sub("# typed: false", "# typed: true")
      
      File.write(file, test_content)
      result = system("bundle exec srb tc #{file} 2>/dev/null")
      
      if result
        upgraded += 1
        puts "✓ Upgraded: #{file}"
      else
        # Revert
        File.write(file, content)
      end
    end
    
    puts "\nUpgraded #{upgraded}/#{files.length} files"
  end
end
```

### แบบฝึกหัดที่ 14: RBS กับ Gem
**โจทย์:** เขียน RBS สำหรับ custom gem

**เฉลย:**
```rbs
# sig/gems/my_awesome_gem.rbs

# MyAwesomeGem - gem ของเรา
module MyAwesomeGem
  VERSION: String
  
  def self.configure: () { (Configuration) -> void } -> void
  def self.configuration: () -> Configuration
  
  class Configuration
    attr_accessor api_key: String
    attr_accessor timeout: Integer
    attr_accessor retry_count: Integer
    attr_accessor base_url: String
    attr_accessor debug: bool
    
    def initialize: () -> void
    def valid?: () -> bool
  end
  
  class Client
    def initialize: (?api_key: String?, ?timeout: Integer) -> void
    
    def get: (String path, ?params: Hash[String, untyped]) -> Response
    def post: (String path, body: Hash[String, untyped]) -> Response
    def put: (String path, body: Hash[String, untyped]) -> Response
    def delete: (String path) -> Response
    
    private
      def request: (String method, String path, ?Hash[String, untyped] options) -> Response
      def build_headers: () -> Hash[String, String]
  end
  
  class Response
    attr_reader status: Integer
    attr_reader body: Hash[String, untyped] | Array[untyped]
    attr_reader headers: Hash[String, String]
    
    def success?: () -> bool
    def error?: () -> bool
    def message: () -> String
    
    def []: (String key) -> untyped
    
    private
      def initialize: (Integer status, Hash[String, untyped] | Array[untyped] body, Hash[String, String] headers) -> void
  end
  
  class Error < StandardError
    attr_reader response: Response?
    
    def initialize: (String message, ?response: Response?) -> void
  end
  
  class AuthenticationError < Error
  end
  
  class RateLimitError < Error
    attr_reader retry_after: Integer
    
    def initialize: (String message, retry_after: Integer) -> void
  end
  
  class NotFoundError < Error
    attr_reader resource_type: String
    attr_reader resource_id: untyped
    
    def initialize: (String resource_type, untyped resource_id) -> void
  end
end
```

### แบบฝึกหัดที่ 15: Complete Type-safe Rails App Setup
**โจทย์:** Setup complete type checking สำหรับ Rails app

**เฉลย:**
```bash
# setup_types.sh - Script สำหรับ setup type checking

#!/bin/bash
set -e

echo "Setting up type checking for Rails app..."

# 1. Add gems
cat >> Gemfile << 'EOF'

group :development, :test do
  gem 'sorbet'
  gem 'sorbet-runtime'
  gem 'tapioca'
  gem 'steep'
  gem 'rbs'
  gem 'rbs-rails'
end
EOF

bundle install

# 2. Initialize Tapioca
bundle exec tapioca init

# 3. Generate RBI files for gems
bundle exec tapioca gems

# 4. Generate RBI for Rails DSLs
bundle exec tapioca dsl

# 5. Initialize Steep
bundle exec steep init

# 6. Initialize RBS directory
mkdir -p sig/{models,controllers,services,lib}

# 7. Generate RBS from database schema
bundle exec rake rbs_rails:generate

# 8. Add sorbet config
cat > sorbet/config << 'EOF'
--dir=.
--ignore=vendor
--ignore=node_modules
--ignore=db/schema.rb
--ignore=tmp
EOF

# 9. Add typed: false to all files
find app lib -name "*.rb" | while read file; do
  if ! head -1 "$file" | grep -q "# typed:"; then
    sed -i '' '1s/^/# typed: false\n/' "$file"
  fi
done

# 10. Run initial type check
echo "Running initial type check..."
bundle exec srb tc --no-error-count || true

echo "Type checking setup complete!"
echo ""
echo "Next steps:"
echo "1. Review errors: bundle exec srb tc"
echo "2. Add 'typed: true' to lib/**/*.rb files one by one"
echo "3. Create RBS signatures in sig/ directory"
echo "4. Run: bundle exec steep check"
```

```ruby
# Makefile สำหรับ type checking workflow
# In Makefile:
# 
# type-check: ## Run all type checks
#   bundle exec srb tc
#   bundle exec steep check
#
# type-gen: ## Generate type signatures
#   bundle exec tapioca gems
#   bundle exec tapioca dsl
#   bundle exec rake rbs_rails:generate
#
# type-status: ## Show type coverage
#   bundle exec rake types:status
```

---

## สรุปบทที่ 92

ในบทนี้เราได้เรียนรู้:

1. **Ruby Type System** - Duck typing และเหตุผลที่ต้องการ type annotations
2. **RBS** - Ruby Signature syntax สำหรับ type definitions
3. **Steep** - Type checker ที่ใช้ RBS
4. **Sorbet** - Facebook's type system สำหรับ Ruby
5. **TypeProf** - Type inference จาก MRI Ruby team
6. **Practical Patterns** - Value objects, Result types, generics
7. **Migration Strategy** - Gradual adoption ใน existing code

### คำแนะนำการ adopt

1. เริ่มจาก lib/ และ services/ (isolated code)
2. ใช้ TypeProf infer types แล้ว refine ด้วยมือ
3. คิดถึง typed: true ก่อน strict
4. เพิ่ม test coverage ควบคู่กับ types
5. ใช้ CI/CD enforce type checking
