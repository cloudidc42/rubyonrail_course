# ตอนที่ 22: Design Patterns ใน Ruby (ขั้นตอนที่ 461-490)

Design Patterns คือแนวทางการแก้ปัญหาซ้ำๆ ที่พบบ่อยในการพัฒนาซอฟต์แวร์ ซึ่งได้รับการพิสูจน์แล้วว่าทำงานได้ดี เป็น "blueprint" หรือ "template" สำหรับการแก้ปัญหา ไม่ใช่โค้ดสำเร็จรูปที่นำมาใช้ได้ทันที

---

## ขั้นตอนที่ 461: ทำความเข้าใจ Design Patterns

### Design Patterns คืออะไร?

Design Patterns ถูกแนะนำโดย "Gang of Four" (GoF) ในหนังสือ "Design Patterns: Elements of Reusable Object-Oriented Software" ปี 1994 แบ่งออกเป็น 3 กลุ่มหลัก:

1. **Creational Patterns** - เกี่ยวกับการสร้าง Object
2. **Structural Patterns** - เกี่ยวกับการจัดโครงสร้าง Object
3. **Behavioral Patterns** - เกี่ยวกับพฤติกรรมและการสื่อสารระหว่าง Object

### ทำไมต้องใช้ Design Patterns?

```ruby
# ปัญหาโดยไม่ใช้ Pattern: โค้ดซ้ำซ้อน
class UserReport
  def generate
    # โค้ด 100 บรรทัด
  end
end

class OrderReport
  def generate
    # โค้ดคล้ายกัน 100 บรรทัด
  end
end

# ดีกว่า: ใช้ Template Method Pattern
class BaseReport
  def generate
    fetch_data
    format_data
    render
  end
  
  private
  
  def fetch_data
    raise NotImplementedError
  end
  
  def format_data
    raise NotImplementedError
  end
  
  def render
    raise NotImplementedError
  end
end
```

---

## ขั้นตอนที่ 462: Singleton Pattern

### คำนิยาม
Singleton Pattern รับประกันว่า Class จะมี Instance เพียงตัวเดียว และมี Global Access Point ไปยัง Instance นั้น

### เมื่อไหรควรใช้
- การเชื่อมต่อ Database
- Logger
- Configuration Manager
- Cache Manager

### การ Implement ใน Ruby

```ruby
# วิธีที่ 1: Manual Singleton
class DatabaseConnection
  @instance = nil
  
  def self.instance
    @instance ||= new
  end
  
  # ป้องกันการสร้าง Instance โดยตรง
  private_class_method :new
  
  def initialize
    @connection = connect_to_database
    puts "การเชื่อมต่อ Database สร้างขึ้นแล้ว"
  end
  
  def query(sql)
    @connection.execute(sql)
  end
  
  private
  
  def connect_to_database
    # ตัวอย่างการเชื่อมต่อ
    "connected"
  end
end

# การใช้งาน
db1 = DatabaseConnection.instance
db2 = DatabaseConnection.instance
puts db1.equal?(db2)  # true - เป็น Object เดียวกัน

# พยายามสร้างโดยตรงจะ Error
# DatabaseConnection.new  # NoMethodError!
```

```ruby
# วิธีที่ 2: ใช้ Module Singleton ของ Ruby
require 'singleton'

class AppConfig
  include Singleton
  
  attr_accessor :database_url, :redis_url, :debug_mode
  
  def initialize
    @database_url = ENV['DATABASE_URL'] || 'postgres://localhost/myapp'
    @redis_url    = ENV['REDIS_URL']    || 'redis://localhost:6379'
    @debug_mode   = ENV['DEBUG'] == 'true'
  end
  
  def production?
    !debug_mode
  end
end

# การใช้งาน
config = AppConfig.instance
puts config.database_url
puts config.debug_mode

# ทุกที่ในแอปจะได้ Instance เดียวกัน
config2 = AppConfig.instance
puts config.equal?(config2)  # true
```

```ruby
# วิธีที่ 3: Thread-safe Singleton
class ThreadSafeCache
  @instance = nil
  @mutex    = Mutex.new
  
  def self.instance
    return @instance if @instance
    
    @mutex.synchronize do
      @instance ||= new
    end
  end
  
  private_class_method :new
  
  def initialize
    @data = {}
    @lock = Mutex.new
  end
  
  def set(key, value)
    @lock.synchronize { @data[key] = value }
  end
  
  def get(key)
    @lock.synchronize { @data[key] }
  end
end
```

### ตัวอย่างใน Rails

```ruby
# config/initializers/app_config.rb
require 'singleton'

class AppSettings
  include Singleton
  
  attr_reader :stripe_key, :sendgrid_key, :aws_bucket
  
  def initialize
    load_settings
  end
  
  def reload!
    load_settings
  end
  
  private
  
  def load_settings
    @stripe_key   = Rails.application.credentials.stripe[:secret_key]
    @sendgrid_key = Rails.application.credentials.sendgrid[:api_key]
    @aws_bucket   = ENV['AWS_BUCKET_NAME']
  end
end

# ใช้งานใน Controller
class PaymentsController < ApplicationController
  def create
    stripe_key = AppSettings.instance.stripe_key
    # ใช้ stripe_key...
  end
end
```

---

## ขั้นตอนที่ 463: Factory Method Pattern

### คำนิยาม
Factory Method Pattern กำหนด Interface สำหรับการสร้าง Object แต่ให้ Subclass ตัดสินใจว่าจะสร้าง Class ไหน

### เมื่อไหรควรใช้
- เมื่อไม่รู้ล่วงหน้าว่าจะสร้าง Object ประเภทไหน
- เมื่อต้องการให้ Subclass ควบคุมการสร้าง Object
- เมื่อต้องการ Encapsulate ตรรกะการสร้าง

```ruby
# ตัวอย่าง: Payment Processor Factory

# Product Interface
class PaymentProcessor
  def process(amount)
    raise NotImplementedError, "#{self.class}#process ต้องถูก Implement"
  end
  
  def refund(transaction_id)
    raise NotImplementedError, "#{self.class}#refund ต้องถูก Implement"
  end
end

# Concrete Products
class StripeProcessor < PaymentProcessor
  def initialize(api_key)
    @api_key = api_key
  end
  
  def process(amount)
    puts "กำลังชำระเงิน #{amount} บาท ผ่าน Stripe"
    { status: 'success', transaction_id: "stripe_#{SecureRandom.hex(8)}" }
  end
  
  def refund(transaction_id)
    puts "กำลังคืนเงิน Transaction: #{transaction_id} ผ่าน Stripe"
    { status: 'refunded' }
  end
end

class PayPalProcessor < PaymentProcessor
  def initialize(client_id, client_secret)
    @client_id     = client_id
    @client_secret = client_secret
  end
  
  def process(amount)
    puts "กำลังชำระเงิน #{amount} บาท ผ่าน PayPal"
    { status: 'success', transaction_id: "paypal_#{SecureRandom.hex(8)}" }
  end
  
  def refund(transaction_id)
    puts "กำลังคืนเงิน Transaction: #{transaction_id} ผ่าน PayPal"
    { status: 'refunded' }
  end
end

class OmiseProcessor < PaymentProcessor
  def initialize(public_key, secret_key)
    @public_key = public_key
    @secret_key = secret_key
  end
  
  def process(amount)
    puts "กำลังชำระเงิน #{amount} บาท ผ่าน Omise"
    { status: 'success', transaction_id: "omise_#{SecureRandom.hex(8)}" }
  end
  
  def refund(transaction_id)
    puts "กำลังคืนเงิน Transaction: #{transaction_id} ผ่าน Omise"
    { status: 'refunded' }
  end
end

# Factory
class PaymentProcessorFactory
  PROCESSORS = {
    'stripe' => StripeProcessor,
    'paypal' => PayPalProcessor,
    'omise'  => OmiseProcessor
  }.freeze
  
  def self.create(type, **options)
    processor_class = PROCESSORS[type.to_s.downcase]
    
    raise ArgumentError, "Payment processor ไม่รองรับ: #{type}" unless processor_class
    
    case type.to_s.downcase
    when 'stripe'
      processor_class.new(options[:api_key])
    when 'paypal'
      processor_class.new(options[:client_id], options[:client_secret])
    when 'omise'
      processor_class.new(options[:public_key], options[:secret_key])
    end
  end
  
  def self.available_processors
    PROCESSORS.keys
  end
end

# การใช้งาน
stripe = PaymentProcessorFactory.create('stripe', api_key: 'sk_test_xxx')
result = stripe.process(500)
puts result

paypal = PaymentProcessorFactory.create('paypal', 
  client_id: 'client_xxx', 
  client_secret: 'secret_xxx'
)
paypal.process(1000)
```

### Abstract Factory Pattern

```ruby
# Abstract Factory: สร้างกลุ่มของ Object ที่เกี่ยวข้องกัน

# Interfaces
class Button
  def render = raise NotImplementedError
  def click  = raise NotImplementedError
end

class TextField
  def render = raise NotImplementedError
  def input  = raise NotImplementedError
end

# Concrete Products สำหรับ Bootstrap
class BootstrapButton < Button
  def render
    "<button class='btn btn-primary'>Click me</button>"
  end
  
  def click
    "Bootstrap button clicked!"
  end
end

class BootstrapTextField < TextField
  def render
    "<input type='text' class='form-control' />"
  end
  
  def input
    "Bootstrap input accepted"
  end
end

# Concrete Products สำหรับ Tailwind
class TailwindButton < Button
  def render
    "<button class='bg-blue-500 text-white px-4 py-2 rounded'>Click me</button>"
  end
  
  def click
    "Tailwind button clicked!"
  end
end

class TailwindTextField < TextField
  def render
    "<input type='text' class='border border-gray-300 rounded px-3 py-2' />"
  end
  
  def input
    "Tailwind input accepted"
  end
end

# Abstract Factory
class UIFactory
  def self.create(framework)
    case framework
    when :bootstrap then BootstrapUIFactory.new
    when :tailwind  then TailwindUIFactory.new
    else raise ArgumentError, "Framework ไม่รองรับ: #{framework}"
    end
  end
end

class BootstrapUIFactory
  def create_button    = BootstrapButton.new
  def create_textfield = BootstrapTextField.new
end

class TailwindUIFactory
  def create_button    = TailwindButton.new
  def create_textfield = TailwindTextField.new
end

# การใช้งาน
factory = UIFactory.create(:bootstrap)
button = factory.create_button
field  = factory.create_textfield

puts button.render
puts field.render
```

### ตัวอย่างใน Rails

```ruby
# app/services/notification_factory.rb
class NotificationFactory
  def self.create(user, type)
    case type
    when :email   then EmailNotification.new(user)
    when :sms     then SmsNotification.new(user)
    when :push    then PushNotification.new(user)
    when :in_app  then InAppNotification.new(user)
    else raise ArgumentError, "Notification type ไม่รองรับ: #{type}"
    end
  end
end

# ใช้งานใน Controller
class OrdersController < ApplicationController
  def create
    @order = Order.create!(order_params)
    
    notification = NotificationFactory.create(current_user, :email)
    notification.send("คำสั่งซื้อของคุณได้รับการยืนยันแล้ว!")
    
    redirect_to @order
  end
end
```

---

## ขั้นตอนที่ 464: Builder Pattern

### คำนิยาม
Builder Pattern แยกการสร้าง Complex Object ออกจาก Representation ทำให้กระบวนการสร้างเดียวกันสามารถสร้าง Representation ที่แตกต่างกัน

### เมื่อไหรควรใช้
- เมื่อ Object มีพารามิเตอร์จำนวนมาก
- เมื่อต้องการสร้าง Object แบบ Step-by-Step
- เมื่อต้องการสร้าง Object หลายแบบที่แตกต่างกัน

```ruby
# Builder Pattern ใน Ruby: HTTP Request Builder

class HttpRequest
  attr_reader :method, :url, :headers, :body, :timeout, :retry_count
  
  def initialize(builder)
    @method      = builder.method
    @url         = builder.url
    @headers     = builder.headers
    @body        = builder.body
    @timeout     = builder.timeout
    @retry_count = builder.retry_count
  end
  
  def to_s
    "#{@method} #{@url}\nHeaders: #{@headers}\nBody: #{@body}"
  end
end

class HttpRequestBuilder
  attr_reader :method, :url, :headers, :body, :timeout, :retry_count
  
  def initialize
    @method      = 'GET'
    @headers     = {}
    @body        = nil
    @timeout     = 30
    @retry_count = 0
  end
  
  def with_method(method)
    @method = method.upcase
    self  # สำคัญ! คืน self เพื่อทำ Method Chaining
  end
  
  def with_url(url)
    @url = url
    self
  end
  
  def with_header(key, value)
    @headers[key] = value
    self
  end
  
  def with_json_body(data)
    @headers['Content-Type'] = 'application/json'
    @body = data.to_json
    self
  end
  
  def with_timeout(seconds)
    @timeout = seconds
    self
  end
  
  def with_retry(count)
    @retry_count = count
    self
  end
  
  def with_auth_token(token)
    @headers['Authorization'] = "Bearer #{token}"
    self
  end
  
  def build
    raise ArgumentError, "URL จำเป็นต้องระบุ" unless @url
    HttpRequest.new(self)
  end
end

# การใช้งาน
request = HttpRequestBuilder.new
  .with_method('POST')
  .with_url('https://api.example.com/users')
  .with_header('Accept', 'application/json')
  .with_auth_token('my_secret_token')
  .with_json_body({ name: 'สมชาย', email: 'somchai@example.com' })
  .with_timeout(60)
  .with_retry(3)
  .build

puts request
```

```ruby
# Builder Pattern สำหรับ Email

class Email
  attr_reader :to, :from, :subject, :body, :cc, :bcc, :attachments
  
  def initialize(attrs)
    @to          = attrs[:to]
    @from        = attrs[:from]
    @subject     = attrs[:subject]
    @body        = attrs[:body]
    @cc          = attrs[:cc] || []
    @bcc         = attrs[:bcc] || []
    @attachments = attrs[:attachments] || []
  end
end

class EmailBuilder
  def initialize
    @attrs = {
      from: 'noreply@myapp.com',
      cc: [],
      bcc: [],
      attachments: []
    }
  end
  
  def to(address)
    @attrs[:to] = address
    self
  end
  
  def from(address)
    @attrs[:from] = address
    self
  end
  
  def subject(text)
    @attrs[:subject] = text
    self
  end
  
  def body(content)
    @attrs[:body] = content
    self
  end
  
  def cc(address)
    @attrs[:cc] << address
    self
  end
  
  def bcc(address)
    @attrs[:bcc] << address
    self
  end
  
  def attach(file_path)
    @attrs[:attachments] << file_path
    self
  end
  
  def build
    validate!
    Email.new(@attrs)
  end
  
  private
  
  def validate!
    raise ArgumentError, "ผู้รับจำเป็นต้องระบุ" unless @attrs[:to]
    raise ArgumentError, "หัวข้อจำเป็นต้องระบุ"  unless @attrs[:subject]
    raise ArgumentError, "เนื้อหาจำเป็นต้องระบุ"  unless @attrs[:body]
  end
end

# การใช้งาน
email = EmailBuilder.new
  .to('customer@example.com')
  .subject('ยืนยันคำสั่งซื้อ #12345')
  .body('ขอบคุณสำหรับการสั่งซื้อ...')
  .cc('manager@company.com')
  .attach('/tmp/invoice.pdf')
  .build

puts email.to
puts email.subject
```

### ตัวอย่างใน Rails (Query Builder)

```ruby
# app/services/report_query_builder.rb
class ReportQueryBuilder
  def initialize
    @model      = nil
    @conditions = {}
    @includes   = []
    @order      = nil
    @limit      = nil
    @date_range = nil
  end
  
  def for_model(model_class)
    @model = model_class
    self
  end
  
  def where(conditions)
    @conditions.merge!(conditions)
    self
  end
  
  def with_associations(*associations)
    @includes.concat(associations)
    self
  end
  
  def order_by(column, direction = :asc)
    @order = "#{column} #{direction}"
    self
  end
  
  def limit(count)
    @limit = count
    self
  end
  
  def in_date_range(start_date, end_date)
    @date_range = start_date..end_date
    self
  end
  
  def build
    raise ArgumentError, "Model จำเป็นต้องระบุ" unless @model
    
    scope = @model.all
    scope = scope.where(@conditions)       if @conditions.any?
    scope = scope.includes(*@includes)     if @includes.any?
    scope = scope.order(@order)            if @order
    scope = scope.limit(@limit)            if @limit
    scope = scope.where(created_at: @date_range) if @date_range
    scope
  end
end

# การใช้งาน
query = ReportQueryBuilder.new
  .for_model(Order)
  .where(status: :completed)
  .with_associations(:user, :items)
  .order_by(:created_at, :desc)
  .in_date_range(1.month.ago, Time.current)
  .limit(100)
  .build

orders = query.to_a
```

---

## ขั้นตอนที่ 465: Prototype Pattern

### คำนิยาม
Prototype Pattern สร้าง Object ใหม่โดยการ Copy หรือ Clone Object ที่มีอยู่แล้ว

### เมื่อไหรควรใช้
- เมื่อการสร้าง Object ตั้งต้นมีต้นทุนสูง
- เมื่อต้องการสร้าง Object ที่คล้ายกันแต่มีค่าที่ต่างกันเล็กน้อย
- เมื่อต้องการ Independent Copy ของ Object

```ruby
# Prototype Pattern ใน Ruby

class DocumentTemplate
  attr_accessor :title, :body, :styles, :metadata
  
  def initialize(title, body, styles = {}, metadata = {})
    @title    = title
    @body     = body
    @styles   = styles
    @metadata = metadata
  end
  
  # Shallow Clone
  def clone
    copy = super
    copy.styles   = @styles.dup    # ต้อง dup เพราะ Hash เป็น Mutable
    copy.metadata = @metadata.dup
    copy
  end
  
  # Deep Clone
  def deep_clone
    Marshal.load(Marshal.dump(self))
  end
  
  def to_s
    "Document: #{@title}"
  end
end

# การใช้งาน
template = DocumentTemplate.new(
  "แม่แบบใบแจ้งหนี้",
  "เนื้อหาใบแจ้งหนี้...",
  { font: 'Sarabun', size: 12 },
  { version: '1.0', author: 'ระบบ' }
)

# สร้าง Clone เพื่อปรับแต่ง
invoice1 = template.clone
invoice1.title    = "ใบแจ้งหนี้ #001"
invoice1.metadata = { customer: 'บริษัท A' }

invoice2 = template.clone
invoice2.title    = "ใบแจ้งหนี้ #002"
invoice2.metadata = { customer: 'บริษัท B' }

puts template.title   # แม่แบบใบแจ้งหนี้ (ไม่เปลี่ยน)
puts invoice1.title   # ใบแจ้งหนี้ #001
puts invoice2.title   # ใบแจ้งหนี้ #002
```

```ruby
# Prototype Registry
class TemplateRegistry
  def initialize
    @templates = {}
  end
  
  def register(name, template)
    @templates[name] = template
  end
  
  def create(name)
    template = @templates[name]
    raise KeyError, "ไม่พบ Template: #{name}" unless template
    template.deep_clone
  end
  
  def list
    @templates.keys
  end
end

# การใช้งาน
registry = TemplateRegistry.new

registry.register(:invoice, DocumentTemplate.new(
  "ใบแจ้งหนี้",
  "รายการสินค้า...",
  { primary_color: 'blue' }
))

registry.register(:receipt, DocumentTemplate.new(
  "ใบเสร็จรับเงิน",
  "ยืนยันการรับเงิน...",
  { primary_color: 'green' }
))

# สร้างเอกสารจาก Template
doc1 = registry.create(:invoice)
doc1.title = "ใบแจ้งหนี้ #101"

doc2 = registry.create(:invoice)
doc2.title = "ใบแจ้งหนี้ #102"
```

---

## ขั้นตอนที่ 466-467: Decorator Pattern

### คำนิยาม
Decorator Pattern เพิ่มพฤติกรรมใหม่ให้กับ Object โดยไม่ต้องแก้ไข Class เดิม โดยการ "ห่อ" Object ด้วย Decorator Object

### เมื่อไหรควรใช้
- เมื่อต้องการเพิ่มความสามารถให้ Object แต่ละตัวในเวลา Runtime
- เมื่อ Inheritance ไม่เหมาะสม (จะทำให้มี Class มากเกินไป)
- เมื่อต้องการ Combine หลาย Behavior

```ruby
# Decorator Pattern: Coffee Shop ตัวอย่าง

# Component Interface
module Beverage
  def cost
    raise NotImplementedError
  end
  
  def description
    raise NotImplementedError
  end
end

# Concrete Component
class Coffee
  include Beverage
  
  def cost
    40.0  # บาท
  end
  
  def description
    "กาแฟดำ"
  end
end

class Tea
  include Beverage
  
  def cost
    25.0
  end
  
  def description
    "ชา"
  end
end

# Base Decorator
class BeverageDecorator
  include Beverage
  
  def initialize(beverage)
    @beverage = beverage
  end
  
  def cost
    @beverage.cost
  end
  
  def description
    @beverage.description
  end
end

# Concrete Decorators
class MilkDecorator < BeverageDecorator
  def cost
    super + 10.0
  end
  
  def description
    "#{super}, นม"
  end
end

class SugarDecorator < BeverageDecorator
  def cost
    super + 5.0
  end
  
  def description
    "#{super}, น้ำตาล"
  end
end

class WhipDecorator < BeverageDecorator
  def cost
    super + 15.0
  end
  
  def description
    "#{super}, วิปครีม"
  end
end

class ExtraShotDecorator < BeverageDecorator
  def cost
    super + 20.0
  end
  
  def description
    "#{super}, เอสเปรสโซเพิ่ม"
  end
end

# การใช้งาน
# กาแฟดำธรรมดา
drink = Coffee.new
puts "#{drink.description} - #{drink.cost} บาท"

# กาแฟลาเต้ (กาแฟ + นม)
latte = MilkDecorator.new(Coffee.new)
puts "#{latte.description} - #{latte.cost} บาท"

# กาแฟลาเต้หวาน (กาแฟ + นม + น้ำตาล)
sweet_latte = SugarDecorator.new(MilkDecorator.new(Coffee.new))
puts "#{sweet_latte.description} - #{sweet_latte.cost} บาท"

# กาแฟ Deluxe (กาแฟ + เอสเปรสโซเพิ่ม + นม + วิปครีม)
deluxe = WhipDecorator.new(
  MilkDecorator.new(
    ExtraShotDecorator.new(
      Coffee.new
    )
  )
)
puts "#{deluxe.description} - #{deluxe.cost} บาท"
```

### Decorator ใน Ruby ด้วย Module

```ruby
# วิธีที่ 2: ใช้ Module สำหรับ Decorator

class User
  attr_reader :name, :email
  
  def initialize(name, email)
    @name  = name
    @email = email
  end
  
  def greeting
    "สวัสดี, #{@name}!"
  end
end

module AdminDecorator
  def greeting
    "[ADMIN] #{super}"
  end
  
  def admin_panel_url
    "/admin/users/#{object_id}"
  end
end

module PremiumDecorator
  def greeting
    "⭐ #{super} (Premium Member)"
  end
  
  def premium_features
    ['ฟีเจอร์พิเศษ 1', 'ฟีเจอร์พิเศษ 2', 'ฟีเจอร์พิเศษ 3']
  end
end

# การใช้งาน
regular_user = User.new("สมชาย", "somchai@example.com")
puts regular_user.greeting

admin_user = User.new("สมหญิง", "somying@example.com")
admin_user.extend(AdminDecorator)
puts admin_user.greeting

premium_user = User.new("มานี", "manee@example.com")
premium_user.extend(PremiumDecorator)
puts premium_user.greeting
puts premium_user.premium_features
```

### Draper Gem ใน Rails (Decorator Pattern)

```ruby
# Gemfile
# gem 'draper'

# app/decorators/user_decorator.rb
class UserDecorator < Draper::Decorator
  delegate_all
  
  def full_name
    "#{object.first_name} #{object.last_name}"
  end
  
  def avatar_url
    if object.avatar.attached?
      h.url_for(object.avatar)
    else
      h.asset_path('default_avatar.png')
    end
  end
  
  def formatted_created_at
    object.created_at.strftime("%d/%m/%Y %H:%M น.")
  end
  
  def role_badge
    case object.role
    when 'admin'   then h.content_tag(:span, 'Admin', class: 'badge badge-danger')
    when 'premium' then h.content_tag(:span, '⭐ Premium', class: 'badge badge-gold')
    else                h.content_tag(:span, 'Member', class: 'badge badge-secondary')
    end
  end
end

# ใช้งานใน Controller
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id]).decorate
  end
end

# ใน View
# = @user.full_name
# = @user.role_badge
# = @user.formatted_created_at
```

---

## ขั้นตอนที่ 468: Adapter Pattern

### คำนิยาม
Adapter Pattern แปลง Interface ของ Class หนึ่งให้เป็น Interface อื่น ทำให้ Class ที่มี Interface ไม่เข้ากันสามารถทำงานร่วมกันได้

### เมื่อไหรควรใช้
- เมื่อต้องการใช้ Class ที่มีอยู่แต่ Interface ไม่ตรงกับที่ต้องการ
- เมื่อต้องการรวม Third-party Library
- เมื่อ Migrate จากระบบเก่าไประบบใหม่

```ruby
# Adapter Pattern: Payment Gateway Adapter

# Target Interface ที่แอปเราต้องการ
module PaymentGateway
  def charge(amount:, currency:, card_token:)
    raise NotImplementedError
  end
  
  def refund(transaction_id:, amount:)
    raise NotImplementedError
  end
end

# ระบบเก่า (Legacy) ที่เราต้องการ Adapt
class LegacyPaymentSystem
  def make_payment(payment_data)
    # API เก่าที่ใช้ format ต่างออกไป
    puts "Legacy: กำลังชำระเงิน #{payment_data[:sum]} #{payment_data[:currency_code]}"
    "LEGACY_#{SecureRandom.hex(4).upcase}"
  end
  
  def reverse_payment(txn_ref, reversal_sum)
    puts "Legacy: กำลังคืนเงิน #{reversal_sum} สำหรับ #{txn_ref}"
    true
  end
end

# Third-party Library ที่มี Interface ต่างออกไป
class ModernPaymentAPI
  def process_transaction(card:, amount_cents:, currency_iso:)
    puts "Modern API: กำลังประมวลผล #{amount_cents/100.0} #{currency_iso}"
    { id: "mod_#{SecureRandom.hex(6)}", status: 'succeeded' }
  end
  
  def issue_refund(original_id:, amount_cents:)
    puts "Modern API: กำลังออกใบคืนเงิน #{amount_cents/100.0}"
    { id: "ref_#{SecureRandom.hex(6)}", status: 'pending' }
  end
end

# Adapter สำหรับระบบเก่า
class LegacyPaymentAdapter
  include PaymentGateway
  
  def initialize
    @legacy_system = LegacyPaymentSystem.new
  end
  
  def charge(amount:, currency:, card_token:)
    # แปลง Interface ของเราให้เป็น Legacy Interface
    result = @legacy_system.make_payment({
      sum: amount,
      currency_code: currency.upcase,
      card: card_token
    })
    
    { success: true, transaction_id: result }
  end
  
  def refund(transaction_id:, amount:)
    success = @legacy_system.reverse_payment(transaction_id, amount)
    { success: success }
  end
end

# Adapter สำหรับ Modern API
class ModernPaymentAdapter
  include PaymentGateway
  
  def initialize
    @api = ModernPaymentAPI.new
  end
  
  def charge(amount:, currency:, card_token:)
    result = @api.process_transaction(
      card: card_token,
      amount_cents: (amount * 100).to_i,  # แปลงเป็น Cents
      currency_iso: currency.upcase
    )
    
    { success: result[:status] == 'succeeded', transaction_id: result[:id] }
  end
  
  def refund(transaction_id:, amount:)
    result = @api.issue_refund(
      original_id: transaction_id,
      amount_cents: (amount * 100).to_i
    )
    
    { success: true, refund_id: result[:id] }
  end
end

# การใช้งาน - โค้ดเดียวกัน ใช้ได้กับทั้งสองระบบ
def process_payment(gateway, amount, currency, card_token)
  result = gateway.charge(
    amount: amount,
    currency: currency,
    card_token: card_token
  )
  
  if result[:success]
    puts "ชำระเงินสำเร็จ! Transaction ID: #{result[:transaction_id]}"
  else
    puts "ชำระเงินไม่สำเร็จ"
  end
end

legacy_gateway = LegacyPaymentAdapter.new
modern_gateway = ModernPaymentAdapter.new

process_payment(legacy_gateway, 500, 'THB', 'card_xxx')
process_payment(modern_gateway, 500, 'THB', 'card_xxx')
```

---

## ขั้นตอนที่ 469: Composite Pattern

### คำนิยาม
Composite Pattern จัดกลุ่ม Object เป็น Tree Structure เพื่อแทน Part-Whole Hierarchy ทำให้ Client ใช้ Object เดี่ยวและ Composite ได้อย่างสม่ำเสมอ

```ruby
# Composite Pattern: File System

class FileSystemItem
  attr_reader :name
  
  def initialize(name)
    @name = name
  end
  
  def size
    raise NotImplementedError
  end
  
  def display(indent = 0)
    raise NotImplementedError
  end
end

# Leaf Node
class File < FileSystemItem
  def initialize(name, size)
    super(name)
    @file_size = size
  end
  
  def size
    @file_size
  end
  
  def display(indent = 0)
    puts "#{'  ' * indent}📄 #{@name} (#{size} KB)"
  end
end

# Composite Node
class Directory < FileSystemItem
  def initialize(name)
    super(name)
    @children = []
  end
  
  def add(item)
    @children << item
    self
  end
  
  def remove(item)
    @children.delete(item)
  end
  
  def size
    @children.sum(&:size)
  end
  
  def display(indent = 0)
    puts "#{'  ' * indent}📁 #{@name} (#{size} KB)"
    @children.each { |child| child.display(indent + 1) }
  end
  
  def find(name)
    return self if @name == name
    
    @children.each do |child|
      result = child.is_a?(Directory) ? child.find(name) : (child.name == name ? child : nil)
      return result if result
    end
    nil
  end
end

# การใช้งาน
root = Directory.new('project')

src = Directory.new('src')
src.add(File.new('main.rb', 10))
src.add(File.new('config.rb', 5))

models = Directory.new('models')
models.add(File.new('user.rb', 8))
models.add(File.new('order.rb', 12))
src.add(models)

tests = Directory.new('tests')
tests.add(File.new('user_spec.rb', 15))
tests.add(File.new('order_spec.rb', 20))

root.add(src)
root.add(tests)
root.add(File.new('README.md', 3))
root.add(File.new('Gemfile', 2))

root.display
puts "\nขนาดรวมทั้งหมด: #{root.size} KB"
```

---

## ขั้นตอนที่ 470: Proxy Pattern

### คำนิยาม
Proxy Pattern จัดให้มี Surrogate หรือ Placeholder สำหรับ Object อื่น เพื่อควบคุม Access ไปยัง Object นั้น

### ประเภทของ Proxy
1. **Virtual Proxy** - Lazy Loading
2. **Protection Proxy** - ควบคุม Access
3. **Remote Proxy** - เชื่อมต่อ Remote Object
4. **Caching Proxy** - Cache Results

```ruby
# Protection Proxy: ควบคุม Access
class BankAccount
  attr_reader :balance, :owner
  
  def initialize(owner, balance)
    @owner   = owner
    @balance = balance
  end
  
  def deposit(amount)
    @balance += amount
    puts "ฝากเงิน #{amount} บาท สำเร็จ"
  end
  
  def withdraw(amount)
    if amount > @balance
      raise "ยอดเงินไม่เพียงพอ"
    end
    @balance -= amount
    puts "ถอนเงิน #{amount} บาท สำเร็จ"
  end
end

class BankAccountProxy
  def initialize(account, current_user)
    @account      = account
    @current_user = current_user
  end
  
  def deposit(amount)
    check_access!
    log_action("deposit", amount)
    @account.deposit(amount)
  end
  
  def withdraw(amount)
    check_access!
    check_withdrawal_limit!(amount)
    log_action("withdraw", amount)
    @account.withdraw(amount)
  end
  
  def balance
    check_access!
    @account.balance
  end
  
  private
  
  def check_access!
    unless @current_user == @account.owner || @current_user == 'admin'
      raise SecurityError, "ไม่มีสิทธิ์เข้าถึงบัญชีนี้"
    end
  end
  
  def check_withdrawal_limit!(amount)
    if amount > 50_000 && @current_user != 'admin'
      raise "จำนวนเงินถอนเกิน Limit (50,000 บาท)"
    end
  end
  
  def log_action(action, amount)
    puts "[LOG] #{Time.now}: #{@current_user} ทำรายการ #{action} #{amount} บาท"
  end
end

# Virtual Proxy: Lazy Loading
class ImageProxy
  def initialize(filename)
    @filename    = filename
    @real_image  = nil
  end
  
  def display
    load_image if @real_image.nil?
    @real_image.display
  end
  
  private
  
  def load_image
    puts "กำลังโหลดรูปภาพ #{@filename} (ใช้เวลา)..."
    @real_image = RealImage.new(@filename)
  end
end

class RealImage
  def initialize(filename)
    @filename = filename
    puts "โหลดรูปภาพ #{@filename} แล้ว"
  end
  
  def display
    puts "แสดงรูปภาพ: #{@filename}"
  end
end

# การใช้งาน
proxy1 = ImageProxy.new("photo1.jpg")
proxy2 = ImageProxy.new("photo2.jpg")

# รูปภาพยังไม่โหลด
puts "สร้าง Proxy แล้ว"

# โหลดเมื่อต้องการแสดง
proxy1.display
proxy1.display  # โหลดครั้งเดียว แสดงซ้ำได้
```

---

## ขั้นตอนที่ 471: Facade Pattern

### คำนิยาม
Facade Pattern จัดให้มี Interface ที่เรียบง่ายสำหรับ Subsystem ที่ซับซ้อน

```ruby
# Facade Pattern: E-Commerce Order Processing

# Subsystems ที่ซับซ้อน
class InventorySystem
  def check_stock(product_id, quantity)
    puts "ตรวจสอบสต็อก Product #{product_id}: #{quantity} ชิ้น"
    quantity <= 100  # สมมุติมีสต็อกพอ
  end
  
  def reserve(product_id, quantity)
    puts "จองสต็อก Product #{product_id}: #{quantity} ชิ้น"
  end
  
  def release(product_id, quantity)
    puts "คืนสต็อก Product #{product_id}: #{quantity} ชิ้น"
  end
end

class PaymentSystem
  def charge(customer_id, amount, payment_method)
    puts "เรียกเก็บเงิน #{amount} บาท จาก Customer #{customer_id}"
    "TXN_#{SecureRandom.hex(8).upcase}"
  end
  
  def refund(transaction_id, amount)
    puts "คืนเงิน #{amount} บาท สำหรับ Transaction #{transaction_id}"
  end
end

class ShippingSystem
  def calculate_cost(from, to, weight)
    puts "คำนวณค่าส่งจาก #{from} ถึง #{to} น้ำหนัก #{weight}kg"
    50.0  # บาท
  end
  
  def create_shipment(order_id, address, items)
    puts "สร้าง Shipment สำหรับ Order #{order_id}"
    "SHIP_#{SecureRandom.hex(6).upcase}"
  end
  
  def track(tracking_number)
    puts "ติดตาม #{tracking_number}"
    { status: 'in_transit', location: 'กรุงเทพฯ' }
  end
end

class NotificationSystem
  def send_email(email, template, data)
    puts "ส่ง Email ถึง #{email} Template: #{template}"
  end
  
  def send_sms(phone, message)
    puts "ส่ง SMS ถึง #{phone}: #{message}"
  end
end

# Facade
class OrderFacade
  def initialize
    @inventory    = InventorySystem.new
    @payment      = PaymentSystem.new
    @shipping     = ShippingSystem.new
    @notification = NotificationSystem.new
  end
  
  def place_order(customer:, items:, payment_method:, shipping_address:)
    puts "\n=== เริ่มกระบวนการสั่งซื้อ ===\n"
    
    # 1. ตรวจสอบสต็อก
    items.each do |item|
      unless @inventory.check_stock(item[:product_id], item[:quantity])
        raise "สินค้า #{item[:product_id]} สต็อกไม่เพียงพอ"
      end
    end
    
    # 2. คำนวณค่าส่ง
    total_weight  = items.sum { |i| i[:weight] * i[:quantity] }
    shipping_cost = @shipping.calculate_cost('warehouse', shipping_address, total_weight)
    
    # 3. คำนวณยอดรวม
    subtotal    = items.sum { |i| i[:price] * i[:quantity] }
    total_amount = subtotal + shipping_cost
    
    # 4. จองสต็อก
    items.each { |item| @inventory.reserve(item[:product_id], item[:quantity]) }
    
    # 5. ชำระเงิน
    begin
      transaction_id = @payment.charge(customer[:id], total_amount, payment_method)
    rescue => e
      # คืนสต็อกหากชำระไม่สำเร็จ
      items.each { |item| @inventory.release(item[:product_id], item[:quantity]) }
      raise e
    end
    
    # 6. สร้าง Shipment
    tracking = @shipping.create_shipment(
      "ORD_#{SecureRandom.hex(4)}",
      shipping_address,
      items
    )
    
    # 7. แจ้งเตือน
    @notification.send_email(customer[:email], 'order_confirmation', {
      tracking: tracking,
      total: total_amount
    })
    @notification.send_sms(customer[:phone], "ยืนยันคำสั่งซื้อ #{tracking}")
    
    puts "\n=== สั่งซื้อเสร็จสมบูรณ์ ===\n"
    
    {
      success: true,
      transaction_id: transaction_id,
      tracking_number: tracking,
      total: total_amount
    }
  end
end

# การใช้งาน - เรียบง่ายมาก!
order_facade = OrderFacade.new

result = order_facade.place_order(
  customer: { id: 1, email: 'customer@example.com', phone: '0812345678' },
  items: [
    { product_id: 'PROD001', quantity: 2, price: 299, weight: 0.5 },
    { product_id: 'PROD002', quantity: 1, price: 599, weight: 1.0 }
  ],
  payment_method: :credit_card,
  shipping_address: 'กรุงเทพฯ 10100'
)

puts result
```

---

## ขั้นตอนที่ 472: Observer Pattern

### คำนิยาม
Observer Pattern กำหนด One-to-Many Dependency ระหว่าง Object เมื่อ Object หนึ่งเปลี่ยนสถานะ Object ที่ Depend ทั้งหมดจะได้รับการแจ้งและ Update โดยอัตโนมัติ

```ruby
# Observer Pattern

# Subject (Observable)
module Observable
  def self.included(base)
    base.instance_variable_set(:@observers, [])
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def add_observer(observer)
      @observers << observer
    end
    
    def observers
      @observers
    end
  end
  
  def notify_observers(event, data = {})
    self.class.observers.each do |observer|
      observer.update(event, data) if observer.respond_to?(:update)
    end
  end
end

# Concrete Subject
class Order
  include Observable
  
  attr_reader :id, :status, :total
  
  def initialize(id, total)
    @id     = id
    @total  = total
    @status = :pending
  end
  
  def confirm!
    @status = :confirmed
    notify_observers(:order_confirmed, { order: self })
  end
  
  def ship!
    @status = :shipped
    notify_observers(:order_shipped, { order: self })
  end
  
  def deliver!
    @status = :delivered
    notify_observers(:order_delivered, { order: self })
  end
  
  def cancel!
    @status = :cancelled
    notify_observers(:order_cancelled, { order: self })
  end
end

# Concrete Observers
class EmailNotifier
  def update(event, data)
    order = data[:order]
    case event
    when :order_confirmed
      puts "[Email] ส่ง Email ยืนยันคำสั่งซื้อ ##{order.id}"
    when :order_shipped
      puts "[Email] ส่ง Email แจ้งการจัดส่ง ##{order.id}"
    when :order_delivered
      puts "[Email] ส่ง Email ยืนยันการรับสินค้า ##{order.id}"
    when :order_cancelled
      puts "[Email] ส่ง Email แจ้งยกเลิก ##{order.id}"
    end
  end
end

class InventoryTracker
  def update(event, data)
    order = data[:order]
    case event
    when :order_confirmed
      puts "[Inventory] อัพเดทสต็อกสำหรับ Order ##{order.id}"
    when :order_cancelled
      puts "[Inventory] คืนสต็อกสำหรับ Order ##{order.id}"
    end
  end
end

class AnalyticsTracker
  def update(event, data)
    order = data[:order]
    puts "[Analytics] บันทึก Event: #{event} Order ##{order.id} ยอด #{order.total} บาท"
  end
end

class SlackNotifier
  def update(event, data)
    order = data[:order]
    if event == :order_confirmed && order.total > 5000
      puts "[Slack] แจ้ง Channel #orders: คำสั่งซื้อใหม่ #{order.total} บาท!"
    end
  end
end

# การใช้งาน
Order.add_observer(EmailNotifier.new)
Order.add_observer(InventoryTracker.new)
Order.add_observer(AnalyticsTracker.new)
Order.add_observer(SlackNotifier.new)

order = Order.new(12345, 6500)
puts "\n--- ยืนยันคำสั่งซื้อ ---"
order.confirm!

puts "\n--- จัดส่งสินค้า ---"
order.ship!

puts "\n--- ส่งมอบแล้ว ---"
order.deliver!
```

### Observer ใน Rails (ActiveRecord Callbacks)

```ruby
# ใน Rails ใช้ ActiveRecord Callbacks หรือ Active Support Notifications

# วิธีที่ 1: Observer Class (Rails Observer)
class OrderObserver < ActiveRecord::Observer
  def after_create(order)
    OrderMailer.confirmation_email(order).deliver_later
    InventoryService.reserve_items(order)
  end
  
  def after_update(order)
    if order.saved_change_to_status?
      case order.status
      when 'shipped'
        OrderMailer.shipping_email(order).deliver_later
      when 'delivered'
        order.update_column(:delivered_at, Time.current)
      end
    end
  end
end

# วิธีที่ 2: ActiveSupport::Notifications
class Order < ApplicationRecord
  after_create do |order|
    ActiveSupport::Notifications.instrument(
      'order.created',
      order: order
    )
  end
  
  after_update :notify_status_change, if: :saved_change_to_status?
  
  private
  
  def notify_status_change
    ActiveSupport::Notifications.instrument(
      "order.#{status}",
      order: self
    )
  end
end

# Subscriber
ActiveSupport::Notifications.subscribe('order.created') do |name, start, finish, id, payload|
  order = payload[:order]
  Analytics.track_event('order_created', { amount: order.total })
end

ActiveSupport::Notifications.subscribe('order.shipped') do |name, start, finish, id, payload|
  order = payload[:order]
  OrderMailer.shipping_confirmation(order).deliver_later
end
```

---

## ขั้นตอนที่ 473: Strategy Pattern

### คำนิยาม
Strategy Pattern กำหนดกลุ่มของ Algorithm แต่ละตัว Encapsulate แต่ละ Algorithm และทำให้แทนกันได้

```ruby
# Strategy Pattern: Sorting Algorithms

# Strategy Interface
module SortStrategy
  def sort(data)
    raise NotImplementedError
  end
end

# Concrete Strategies
class BubbleSort
  include SortStrategy
  
  def sort(data)
    arr = data.dup
    n = arr.length
    loop do
      swapped = false
      (n - 1).times do |i|
        if arr[i] > arr[i + 1]
          arr[i], arr[i + 1] = arr[i + 1], arr[i]
          swapped = true
        end
      end
      break unless swapped
    end
    arr
  end
  
  def name = "Bubble Sort"
end

class QuickSort
  include SortStrategy
  
  def sort(data)
    return data if data.length <= 1
    
    pivot = data[data.length / 2]
    left  = data.select { |x| x < pivot }
    mid   = data.select { |x| x == pivot }
    right = data.select { |x| x > pivot }
    
    sort(left) + mid + sort(right)
  end
  
  def name = "Quick Sort"
end

class RubyNativeSort
  include SortStrategy
  
  def sort(data)
    data.sort
  end
  
  def name = "Ruby Native Sort"
end

# Context
class Sorter
  def initialize(strategy = RubyNativeSort.new)
    @strategy = strategy
  end
  
  def strategy=(strategy)
    @strategy = strategy
  end
  
  def sort(data)
    puts "ใช้ Strategy: #{@strategy.name}"
    start = Time.now
    result = @strategy.sort(data)
    elapsed = Time.now - start
    puts "ใช้เวลา: #{elapsed * 1000:.4f}ms"
    result
  end
end

# การใช้งาน
data = (1..20).to_a.shuffle
puts "ข้อมูลต้นฉบับ: #{data}"

sorter = Sorter.new
puts "\n--- Bubble Sort ---"
sorter.strategy = BubbleSort.new
puts sorter.sort(data)

puts "\n--- Quick Sort ---"
sorter.strategy = QuickSort.new
puts sorter.sort(data)

puts "\n--- Native Sort ---"
sorter.strategy = RubyNativeSort.new
puts sorter.sort(data)
```

### Strategy Pattern ใน Rails (Discount Calculator)

```ruby
# app/services/discount_strategies/

# Base Strategy
module DiscountStrategy
  def calculate(order)
    raise NotImplementedError
  end
  
  def description
    raise NotImplementedError
  end
end

class PercentageDiscount
  include DiscountStrategy
  
  def initialize(percentage)
    @percentage = percentage
  end
  
  def calculate(order)
    order.subtotal * (@percentage / 100.0)
  end
  
  def description
    "ลด #{@percentage}%"
  end
end

class FixedDiscount
  include DiscountStrategy
  
  def initialize(amount)
    @amount = amount
  end
  
  def calculate(order)
    [@amount, order.subtotal].min
  end
  
  def description
    "ลด #{@amount} บาท"
  end
end

class BuyXGetYDiscount
  include DiscountStrategy
  
  def initialize(buy_qty, get_qty, product_id)
    @buy_qty    = buy_qty
    @get_qty    = get_qty
    @product_id = product_id
  end
  
  def calculate(order)
    item = order.items.find { |i| i.product_id == @product_id }
    return 0 unless item
    
    free_count = (item.quantity / (@buy_qty + @get_qty)) * @get_qty
    free_count * item.unit_price
  end
  
  def description
    "ซื้อ #{@buy_qty} แถม #{@get_qty}"
  end
end

# Context
class PriceCalculator
  def initialize(discount_strategy = nil)
    @discount_strategy = discount_strategy
  end
  
  def calculate(order)
    discount = @discount_strategy ? @discount_strategy.calculate(order) : 0
    
    {
      subtotal: order.subtotal,
      discount: discount,
      total: order.subtotal - discount,
      discount_description: @discount_strategy&.description
    }
  end
end

# ใช้งานใน Controller
class OrdersController < ApplicationController
  def calculate_total
    order   = Order.find(params[:id])
    coupon  = Coupon.find_by(code: params[:coupon_code])
    
    strategy = if coupon
      case coupon.discount_type
      when 'percentage' then PercentageDiscount.new(coupon.value)
      when 'fixed'      then FixedDiscount.new(coupon.value)
      end
    end
    
    calculator = PriceCalculator.new(strategy)
    render json: calculator.calculate(order)
  end
end
```

---

## ขั้นตอนที่ 474: Command Pattern

### คำนิยาม
Command Pattern ห่อ Request เป็น Object ทำให้สามารถ Parameterize Client ด้วย Request ต่างๆ, รองรับ Undo/Redo และทำ Queue ได้

```ruby
# Command Pattern: Text Editor ที่มี Undo/Redo

# Command Interface
class Command
  def execute
    raise NotImplementedError
  end
  
  def undo
    raise NotImplementedError
  end
end

# Receiver
class TextDocument
  attr_reader :content
  
  def initialize
    @content = ""
  end
  
  def insert_text(position, text)
    @content = @content[0...position] + text + @content[position..]
  end
  
  def delete_text(position, length)
    @content = @content[0...position] + @content[(position + length)..]
  end
  
  def replace_text(position, length, new_text)
    old_text = @content[position, length]
    delete_text(position, length)
    insert_text(position, new_text)
    old_text
  end
end

# Concrete Commands
class InsertCommand < Command
  def initialize(document, position, text)
    @document = document
    @position = position
    @text     = text
  end
  
  def execute
    @document.insert_text(@position, @text)
    puts "Insert: '#{@text}' ที่ Position #{@position}"
  end
  
  def undo
    @document.delete_text(@position, @text.length)
    puts "Undo Insert: ลบ '#{@text}' ออก"
  end
end

class DeleteCommand < Command
  def initialize(document, position, length)
    @document = document
    @position = position
    @length   = length
    @deleted_text = nil
  end
  
  def execute
    @deleted_text = @document.content[@position, @length]
    @document.delete_text(@position, @length)
    puts "Delete: ลบ '#{@deleted_text}' ที่ Position #{@position}"
  end
  
  def undo
    @document.insert_text(@position, @deleted_text)
    puts "Undo Delete: คืน '#{@deleted_text}'"
  end
end

# Invoker (Command Manager)
class CommandManager
  def initialize
    @history = []
    @future  = []
  end
  
  def execute(command)
    command.execute
    @history << command
    @future.clear  # Clear redo stack เมื่อมีคำสั่งใหม่
  end
  
  def undo
    if @history.empty?
      puts "ไม่มีคำสั่งให้ Undo"
      return
    end
    
    command = @history.pop
    command.undo
    @future << command
  end
  
  def redo
    if @future.empty?
      puts "ไม่มีคำสั่งให้ Redo"
      return
    end
    
    command = @future.pop
    command.execute
    @history << command
  end
  
  def history_size = @history.size
  def future_size  = @future.size
end

# การใช้งาน
doc     = TextDocument.new
manager = CommandManager.new

puts "=== เริ่มแก้ไข Document ==="

manager.execute(InsertCommand.new(doc, 0, "สวัสดี"))
puts "Content: #{doc.content}"

manager.execute(InsertCommand.new(doc, 6, " โลก"))
puts "Content: #{doc.content}"

manager.execute(DeleteCommand.new(doc, 6, 4))
puts "Content: #{doc.content}"

puts "\n--- Undo ---"
manager.undo
puts "Content: #{doc.content}"

manager.undo
puts "Content: #{doc.content}"

puts "\n--- Redo ---"
manager.redo
puts "Content: #{doc.content}"
```

---

## ขั้นตอนที่ 475: Iterator Pattern

### คำนิยาม
Iterator Pattern จัดให้มีวิธีเข้าถึง Element ของ Aggregate Object แบบ Sequential โดยไม่ต้อง Expose Underlying Representation

```ruby
# Iterator Pattern ใน Ruby (Ruby มี Enumerable built-in)

# Custom Iterator
class NumberRange
  include Enumerable  # Ruby's built-in Iterator support
  
  def initialize(min, max, step = 1)
    @min  = min
    @max  = max
    @step = step
  end
  
  # จำเป็นต้อง Implement เพียง each
  def each
    current = @min
    while current <= @max
      yield current
      current += @step
    end
  end
end

range = NumberRange.new(1, 20, 2)

# ได้ Enumerable methods ฟรีทั้งหมด!
puts range.to_a.inspect
puts range.select(&:odd?).inspect
puts range.map { |n| n * 2 }.inspect
puts range.sum
puts range.min
puts range.max

# External Iterator
class FibonacciIterator
  include Enumerable
  
  def initialize(limit)
    @limit = limit
  end
  
  def each
    a, b = 0, 1
    count = 0
    while count < @limit
      yield a
      a, b = b, a + b
      count += 1
    end
  end
end

fibs = FibonacciIterator.new(10)
puts fibs.to_a.inspect
puts fibs.select { |n| n.even? }.inspect
puts fibs.sum
```

### Tree Iterator ที่ซับซ้อน

```ruby
class TreeNode
  attr_accessor :value, :children
  
  def initialize(value)
    @value    = value
    @children = []
  end
  
  def add_child(child)
    @children << child
    self
  end
end

class TreeIterator
  include Enumerable
  
  def initialize(root, traversal = :depth_first)
    @root       = root
    @traversal  = traversal
  end
  
  def each(&block)
    case @traversal
    when :depth_first  then depth_first_traversal(@root, &block)
    when :breadth_first then breadth_first_traversal(@root, &block)
    end
  end
  
  private
  
  def depth_first_traversal(node, &block)
    block.call(node.value)
    node.children.each { |child| depth_first_traversal(child, &block) }
  end
  
  def breadth_first_traversal(node, &block)
    queue = [node]
    until queue.empty?
      current = queue.shift
      block.call(current.value)
      queue.concat(current.children)
    end
  end
end

# สร้าง Tree
root = TreeNode.new(1)
n2 = TreeNode.new(2)
n3 = TreeNode.new(3)
n4 = TreeNode.new(4)
n5 = TreeNode.new(5)
n6 = TreeNode.new(6)

root.add_child(n2).add_child(n3)
n2.add_child(n4).add_child(n5)
n3.add_child(n6)

puts "Depth-First:"
TreeIterator.new(root, :depth_first).each { |v| print "#{v} " }

puts "\nBreadth-First:"
TreeIterator.new(root, :breadth_first).each { |v| print "#{v} " }
```

---

## ขั้นตอนที่ 476: Template Method Pattern

### คำนิยาม
Template Method Pattern กำหนด Skeleton ของ Algorithm ใน Method โดยเลื่อนการ Implement บางขั้นตอนไปยัง Subclass

```ruby
# Template Method Pattern: Report Generator

class ReportGenerator
  # Template Method - กำหนด Algorithm ไว้ที่นี่
  def generate(data)
    title   = generate_title
    header  = generate_header
    body    = format_data(data)
    footer  = generate_footer
    
    assemble_report(title, header, body, footer)
  end
  
  private
  
  def generate_title
    "รายงาน"  # Default implementation
  end
  
  def generate_header
    "=" * 50
  end
  
  def generate_footer
    "=" * 50 + "\nสร้างเมื่อ: #{Time.now.strftime('%d/%m/%Y %H:%M')}"
  end
  
  # Abstract methods - Subclass ต้อง Implement
  def format_data(data)
    raise NotImplementedError, "#{self.class} ต้อง Implement format_data"
  end
  
  def assemble_report(title, header, body, footer)
    [title, header, body, footer].join("\n")
  end
end

class SalesReport < ReportGenerator
  def generate_title
    "รายงานยอดขาย"
  end
  
  def format_data(data)
    lines = ["ลำดับ | สินค้า              | ปริมาณ | ยอดขาย"]
    lines << "-" * 50
    
    data.each_with_index do |item, index|
      lines << "#{index + 1.to_s.rjust(4)} | #{item[:name].ljust(20)} | #{item[:qty].to_s.rjust(6)} | #{item[:revenue].to_s.rjust(8)} บาท"
    end
    
    lines << "-" * 50
    total = data.sum { |i| i[:revenue] }
    lines << "รวมทั้งหมด: #{total} บาท"
    
    lines.join("\n")
  end
end

class UserReport < ReportGenerator
  def generate_title
    "รายงานผู้ใช้งาน"
  end
  
  def format_data(data)
    lines = ["ลำดับ | ชื่อ                | สถานะ"]
    lines << "-" * 50
    
    data.each_with_index do |user, index|
      status = user[:active] ? "✅ Active" : "❌ Inactive"
      lines << "#{(index + 1).to_s.rjust(4)} | #{user[:name].ljust(20)} | #{status}"
    end
    
    lines.join("\n")
  end
  
  # Override Footer
  def generate_footer
    active_count = "ข้อมูลล่าสุด"
    "=" * 50 + "\n#{active_count}"
  end
end

# การใช้งาน
sales_data = [
  { name: 'สินค้า A', qty: 100, revenue: 5000 },
  { name: 'สินค้า B', qty: 50,  revenue: 3500 },
  { name: 'สินค้า C', qty: 200, revenue: 8000 }
]

user_data = [
  { name: 'สมชาย',  active: true },
  { name: 'สมหญิง', active: false },
  { name: 'มานี',   active: true }
]

sales_report = SalesReport.new
puts sales_report.generate(sales_data)

puts "\n\n"

user_report = UserReport.new
puts user_report.generate(user_data)
```

---

## ขั้นตอนที่ 477: State Pattern

### คำนิยาม
State Pattern อนุญาตให้ Object เปลี่ยนพฤติกรรมเมื่อ Internal State เปลี่ยน ดูเหมือน Object เปลี่ยน Class

```ruby
# State Pattern: Traffic Light

class TrafficLight
  attr_reader :state
  
  def initialize
    @state = RedState.new(self)
  end
  
  def change_state(state)
    @state = state
  end
  
  def next
    @state.next
  end
  
  def current_action
    @state.action
  end
  
  def duration
    @state.duration
  end
end

class TrafficLightState
  def initialize(light)
    @light = light
  end
  
  def next
    raise NotImplementedError
  end
  
  def action
    raise NotImplementedError
  end
  
  def duration
    raise NotImplementedError
  end
end

class RedState < TrafficLightState
  def next
    puts "🟡 เปลี่ยนเป็นไฟเหลือง"
    @light.change_state(YellowState.new(@light))
  end
  
  def action
    "🔴 หยุด!"
  end
  
  def duration
    30  # วินาที
  end
end

class YellowState < TrafficLightState
  def next
    puts "🟢 เปลี่ยนเป็นไฟเขียว"
    @light.change_state(GreenState.new(@light))
  end
  
  def action
    "🟡 เตรียมตัว"
  end
  
  def duration
    5
  end
end

class GreenState < TrafficLightState
  def next
    puts "🔴 เปลี่ยนเป็นไฟแดง"
    @light.change_state(RedState.new(@light))
  end
  
  def action
    "🟢 ไปได้!"
  end
  
  def duration
    25
  end
end

# การใช้งาน
light = TrafficLight.new

5.times do
  puts "\nสถานะปัจจุบัน: #{light.current_action} (#{light.duration} วินาที)"
  sleep(0.5)
  light.next
end
```

### State Pattern ใน Rails (AASM)

```ruby
# Gemfile: gem 'aasm'

class Order < ApplicationRecord
  include AASM
  
  aasm column: 'status' do
    state :pending, initial: true
    state :confirmed
    state :processing
    state :shipped
    state :delivered
    state :cancelled
    state :refunded
    
    event :confirm do
      transitions from: :pending, to: :confirmed
      after do
        OrderMailer.confirmation_email(self).deliver_later
        notify_observers(:order_confirmed)
      end
    end
    
    event :process do
      transitions from: :confirmed, to: :processing
      after do
        WarehouseService.process_order(self)
      end
    end
    
    event :ship do
      transitions from: :processing, to: :shipped
      after do |tracking_number|
        update!(tracking_number: tracking_number)
        OrderMailer.shipping_email(self).deliver_later
      end
    end
    
    event :deliver do
      transitions from: :shipped, to: :delivered
      after do
        update!(delivered_at: Time.current)
      end
    end
    
    event :cancel do
      transitions from: [:pending, :confirmed], to: :cancelled
      after do
        RefundService.process(self) if paid?
        InventoryService.restore_items(self)
      end
    end
    
    event :refund do
      transitions from: [:delivered, :cancelled], to: :refunded
      after do
        RefundService.issue_refund(self)
        OrderMailer.refund_email(self).deliver_later
      end
    end
  end
end

# การใช้งาน
order = Order.create!(user: current_user, items: cart_items)

order.confirm!
order.process!
order.ship!("TRACK123")
order.deliver!

puts order.status        # "delivered"
puts order.may_cancel?   # false (ไม่สามารถยกเลิกหลัง delivered)
puts order.may_refund?   # true
```

---

## ขั้นตอนที่ 478-480: Patterns เพิ่มเติม

### Chain of Responsibility Pattern

```ruby
# Chain of Responsibility: Request Processing

class Handler
  attr_writer :next_handler
  
  def handle(request)
    if @next_handler
      @next_handler.handle(request)
    else
      puts "ไม่มี Handler รองรับ Request: #{request}"
      nil
    end
  end
  
  def set_next(handler)
    @next_handler = handler
    handler  # Return handler เพื่อ Chain
  end
end

class AuthenticationHandler < Handler
  def handle(request)
    if request[:token].nil?
      puts "Authentication Failed: ไม่มี Token"
      return nil
    end
    puts "Authentication OK"
    super
  end
end

class AuthorizationHandler < Handler
  def handle(request)
    unless request[:permissions]&.include?(request[:required_permission])
      puts "Authorization Failed: ไม่มีสิทธิ์ #{request[:required_permission]}"
      return nil
    end
    puts "Authorization OK"
    super
  end
end

class ValidationHandler < Handler
  def handle(request)
    if request[:data].nil? || request[:data].empty?
      puts "Validation Failed: ข้อมูลว่างเปล่า"
      return nil
    end
    puts "Validation OK"
    super
  end
end

class ProcessingHandler < Handler
  def handle(request)
    puts "Processing: ประมวลผล #{request[:data]}"
    { success: true, processed: request[:data] }
  end
end

# การใช้งาน
auth        = AuthenticationHandler.new
authz       = AuthorizationHandler.new
validation  = ValidationHandler.new
processing  = ProcessingHandler.new

# สร้าง Chain
auth.set_next(authz).set_next(validation).set_next(processing)

puts "=== Request สมบูรณ์ ==="
result = auth.handle({
  token: 'valid_token',
  permissions: [:read, :write],
  required_permission: :write,
  data: "ข้อมูลที่จะประมวลผล"
})
puts result.inspect

puts "\n=== Request ไม่มี Token ==="
auth.handle({
  permissions: [:read],
  required_permission: :read,
  data: "ข้อมูล"
})

puts "\n=== Request ไม่มีสิทธิ์ ==="
auth.handle({
  token: 'valid_token',
  permissions: [:read],
  required_permission: :write,
  data: "ข้อมูล"
})
```

### Memento Pattern

```ruby
# Memento Pattern: Save/Restore State

class GameCharacter
  attr_reader :name, :level, :hp, :experience
  
  def initialize(name)
    @name       = name
    @level      = 1
    @hp         = 100
    @experience = 0
  end
  
  def gain_experience(points)
    @experience += points
    level_up! if @experience >= level_threshold
    puts "#{@name} ได้รับ #{points} EXP (รวม: #{@experience})"
  end
  
  def take_damage(amount)
    @hp = [@hp - amount, 0].max
    puts "#{@name} โดนโจมตี #{amount} HP เหลือ: #{@hp}"
  end
  
  def heal(amount)
    @hp = [@hp + amount, max_hp].min
    puts "#{@name} ฟื้นฟู #{amount} HP เหลือ: #{@hp}"
  end
  
  # สร้าง Memento (Snapshot)
  def save
    GameSave.new(
      name: @name,
      level: @level,
      hp: @hp,
      experience: @experience
    )
  end
  
  # Restore จาก Memento
  def restore(save)
    @level      = save.level
    @hp         = save.hp
    @experience = save.experience
    puts "#{@name} โหลด Save Game (Level #{@level}, HP #{@hp})"
  end
  
  def to_s
    "#{@name} - Level #{@level}, HP #{@hp}/#{max_hp}, EXP #{@experience}/#{level_threshold}"
  end
  
  private
  
  def level_up!
    @level += 1
    @hp     = max_hp
    @experience = 0
    puts "#{@name} Level UP! ตอนนี้ Level #{@level}"
  end
  
  def max_hp          = @level * 100
  def level_threshold = @level * 1000
end

class GameSave
  attr_reader :name, :level, :hp, :experience, :saved_at
  
  def initialize(name:, level:, hp:, experience:)
    @name       = name
    @level      = level
    @hp         = hp
    @experience = experience
    @saved_at   = Time.now
  end
end

class SaveManager
  def initialize
    @saves = []
  end
  
  def save(character)
    save_game = character.save
    @saves << save_game
    puts "บันทึก Slot #{@saves.size}"
    @saves.size
  end
  
  def load(slot_number)
    @saves[slot_number - 1] || raise("ไม่พบ Save Slot #{slot_number}")
  end
  
  def list_saves
    @saves.each_with_index do |save, i|
      puts "Slot #{i + 1}: Level #{save.level}, HP #{save.hp} (#{save.saved_at.strftime('%H:%M')})"
    end
  end
end

# การใช้งาน
hero    = GameCharacter.new("วีรบุรุษ")
manager = SaveManager.new

puts hero
hero.gain_experience(500)
manager.save(hero)

hero.gain_experience(300)
hero.take_damage(50)
puts hero
manager.save(hero)

hero.take_damage(80)
puts "หลังโดนโจมตี: #{hero}"

puts "\n--- โหลด Save Slot 1 ---"
hero.restore(manager.load(1))
puts hero
```

---

## แบบฝึกหัดบทที่ 22 (30 ข้อ)

### ระดับพื้นฐาน (ข้อ 1-10)

**ข้อ 1:** สร้าง Singleton Pattern สำหรับ Logger ที่เก็บ Log ไว้ใน Array และมี method `log(level, message)`, `logs`, `clear!`

**ข้อ 2:** สร้าง Factory Method สำหรับ Shape (Circle, Rectangle, Triangle) แต่ละ Shape มี method `area` และ `perimeter`

**ข้อ 3:** สร้าง Builder Pattern สำหรับ Pizza โดยมี topping, size, crust_type และ extra_cheese

**ข้อ 4:** สร้าง Prototype Pattern สำหรับ Document ที่สามารถ Clone และแก้ไขได้โดยไม่กระทบต้นฉบับ

**ข้อ 5:** สร้าง Decorator Pattern สำหรับ TextFormatter ที่มี Bold, Italic, Underline decorators

**ข้อ 6:** สร้าง Adapter สำหรับแปลง Temperature ระหว่าง Celsius, Fahrenheit และ Kelvin

**ข้อ 7:** สร้าง Composite Pattern สำหรับ Menu ที่มี MenuItem (Leaf) และ MenuGroup (Composite)

**ข้อ 8:** สร้าง Proxy Pattern สำหรับ ExpensiveCalculation ที่ Cache ผลการคำนวณ

**ข้อ 9:** สร้าง Facade สำหรับ MediaPlayer ที่มี VideoDecoder, AudioDecoder, Subtitle modules

**ข้อ 10:** สร้าง Observer Pattern สำหรับ Stock Market ที่แจ้งเตือนเมื่อราคาหุ้นเปลี่ยน

### ระดับกลาง (ข้อ 11-20)

**ข้อ 11:** สร้าง Strategy Pattern สำหรับ Tax Calculator ที่มีกลยุทธ์ต่างๆ (Personal Income, Corporate, VAT)

**ข้อ 12:** สร้าง Command Pattern สำหรับ Smart Home Controller ที่มี Undo/Redo

**ข้อ 13:** สร้าง Iterator Pattern สำหรับ Playlist ที่ supports shuffle, repeat, next, previous

**ข้อ 14:** สร้าง Template Method สำหรับ Data Importer (CSV, JSON, XML) ที่มีขั้นตอน: read, validate, transform, save

**ข้อ 15:** สร้าง State Pattern สำหรับ ATM Machine ที่มี states: idle, card_inserted, pin_entered, transaction_in_progress

**ข้อ 16:** สร้าง Chain of Responsibility สำหรับ Support Ticket System (Level 1, Level 2, Level 3 support)

**ข้อ 17:** สร้าง Memento Pattern สำหรับ Form ที่ support Auto-save และ Restore

**ข้อ 18:** สร้าง Abstract Factory สำหรับสร้าง UI Components ที่รองรับทั้ง Web และ Mobile

**ข้อ 19:** สร้าง Flyweight Pattern สำหรับ Character Rendering ใน Text Editor

**ข้อ 20:** สร้าง Mediator Pattern สำหรับ Chat Room ที่มี Users หลายคน

### ระดับสูง (ข้อ 21-30)

**ข้อ 21:** Implement Observer + Strategy Pattern สำหรับ Event-Driven Trading System

**ข้อ 22:** สร้าง Plugin System โดยใช้ Factory + Command + Observer Patterns รวมกัน

**ข้อ 23:** Implement Visitor Pattern สำหรับ Abstract Syntax Tree (AST) Parser

**ข้อ 24:** สร้าง Builder Pattern สำหรับ SQL Query Builder ที่รองรับ SELECT, WHERE, JOIN, ORDER BY, GROUP BY

**ข้อ 25:** สร้าง Decorator Pattern Pipeline สำหรับ Image Processing (resize, crop, filter, watermark)

**ข้อ 26:** Implement Repository Pattern สำหรับ Data Access Layer

**ข้อ 27:** สร้าง Service Object Pattern (Rails) สำหรับ User Registration Flow

**ข้อ 28:** Implement Circuit Breaker Pattern สำหรับ External API Calls

**ข้อ 29:** สร้าง Specification Pattern สำหรับ Business Rule Engine

**ข้อ 30:** สร้าง Complete E-Commerce System โดยใช้อย่างน้อย 5 Design Patterns ที่เรียนมา

---

### เฉลยตัวอย่าง ข้อ 1: Logger Singleton

```ruby
require 'singleton'

class Logger
  include Singleton
  
  LEVELS = %i[debug info warn error fatal].freeze
  
  def initialize
    @logs = []
  end
  
  def log(level, message)
    raise ArgumentError, "Level ไม่ถูกต้อง: #{level}" unless LEVELS.include?(level)
    
    entry = {
      timestamp: Time.now,
      level: level,
      message: message
    }
    @logs << entry
    
    level_prefix = case level
    when :debug then "[DEBUG]"
    when :info  then "[INFO] "
    when :warn  then "[WARN] "
    when :error then "[ERROR]"
    when :fatal then "[FATAL]"
    end
    
    puts "#{entry[:timestamp].strftime('%H:%M:%S')} #{level_prefix} #{message}"
  end
  
  def logs(level: nil)
    return @logs if level.nil?
    @logs.select { |log| log[:level] == level }
  end
  
  def clear!
    @logs.clear
    puts "Log cleared"
  end
  
  def summary
    puts "\n=== Log Summary ==="
    LEVELS.each do |level|
      count = @logs.count { |l| l[:level] == level }
      puts "#{level.upcase}: #{count} entries" if count > 0
    end
    puts "Total: #{@logs.size} entries"
  end
  
  # Convenience methods
  def debug(msg) = log(:debug, msg)
  def info(msg)  = log(:info, msg)
  def warn(msg)  = log(:warn, msg)
  def error(msg) = log(:error, msg)
  def fatal(msg) = log(:fatal, msg)
end

# การใช้งาน
logger = Logger.instance

logger.info "Application เริ่มต้นแล้ว"
logger.debug "กำลัง Load configuration"
logger.warn "ไม่พบ Cache, กำลังสร้างใหม่"
logger.error "ไม่สามารถเชื่อมต่อ Database ได้"

logger.summary

# ทุกที่ในแอปได้ Instance เดียวกัน
puts Logger.instance.equal?(logger)  # true
```

### เฉลยตัวอย่าง ข้อ 11: Tax Calculator Strategy

```ruby
module TaxStrategy
  def calculate(income)
    raise NotImplementedError
  end
  
  def name
    raise NotImplementedError
  end
end

class PersonalIncomeTax
  include TaxStrategy
  
  BRACKETS = [
    { up_to: 150_000,   rate: 0.00 },
    { up_to: 300_000,   rate: 0.05 },
    { up_to: 500_000,   rate: 0.10 },
    { up_to: 750_000,   rate: 0.15 },
    { up_to: 1_000_000, rate: 0.20 },
    { up_to: 2_000_000, rate: 0.25 },
    { up_to: 5_000_000, rate: 0.30 },
    { up_to: Float::INFINITY, rate: 0.35 }
  ].freeze
  
  def calculate(income)
    tax = 0
    remaining = income
    previous_threshold = 0
    
    BRACKETS.each do |bracket|
      bracket_income = [remaining, bracket[:up_to] - previous_threshold].min
      break if bracket_income <= 0
      
      tax += bracket_income * bracket[:rate]
      remaining -= bracket_income
      previous_threshold = bracket[:up_to]
    end
    
    tax
  end
  
  def name = "ภาษีเงินได้บุคคลธรรมดา"
end

class CorporateTax
  include TaxStrategy
  
  RATE = 0.20
  
  def calculate(profit)
    profit * RATE
  end
  
  def name = "ภาษีนิติบุคคล (20%)"
end

class VATCalculator
  include TaxStrategy
  
  RATE = 0.07
  
  def calculate(amount)
    amount * RATE
  end
  
  def name = "ภาษีมูลค่าเพิ่ม (VAT 7%)"
end

class TaxCalculator
  def initialize(strategy)
    @strategy = strategy
  end
  
  def calculate(amount)
    tax = @strategy.calculate(amount)
    
    puts "\n=== #{@strategy.name} ==="
    puts "รายได้/ยอดขาย: #{format_currency(amount)}"
    puts "ภาษีที่ต้องชำระ: #{format_currency(tax)}"
    puts "อัตราภาษีที่แท้จริง: #{(tax / amount * 100).round(2)}%"
    
    tax
  end
  
  private
  
  def format_currency(amount)
    "#{amount.round(2).to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse} บาท"
  end
end

# การใช้งาน
personal_calc  = TaxCalculator.new(PersonalIncomeTax.new)
corporate_calc = TaxCalculator.new(CorporateTax.new)
vat_calc       = TaxCalculator.new(VATCalculator.new)

personal_calc.calculate(600_000)
corporate_calc.calculate(1_000_000)
vat_calc.calculate(100_000)
```

---

*สรุปบทที่ 22: Design Patterns เป็นเครื่องมือสำคัญในการพัฒนาซอฟต์แวร์ที่ดี Ruby มีความยืดหยุ่นสูงทำให้สามารถ Implement Design Patterns ได้อย่างหลากหลายและ Elegant การเลือกใช้ Pattern ที่เหมาะสมกับปัญหาจะทำให้โค้ดมีความ Maintainable, Extensible และ Reusable สูง*
