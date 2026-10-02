# ตอนที่ 28: Debugging ใน Ruby (ขั้นตอนที่ 606-625)

Debugging คือกระบวนการค้นหาและแก้ไข Bug ใน Code Ruby มีเครื่องมือ Debugging ที่หลากหลาย ตั้งแต่วิธีง่ายๆ อย่าง `puts` ไปจนถึง Interactive Debugger เช่น Pry และ Byebug

---

## ขั้นตอนที่ 606: puts/p/pp Debugging

### วิธีพื้นฐาน

```ruby
# puts - แสดงผลและขึ้นบรรทัดใหม่
puts "Debug: ถึงจุดนี้แล้ว"
puts variable_name
puts array.inspect

# p - แสดงผล inspect (ดีกว่า puts สำหรับ Debug)
p variable_name        # แสดง inspect result
p "hello"              # "hello" (มี Quotes)
p [1, 2, 3]            # [1, 2, 3]
p nil                  # nil
p false                # false

# pp - Pretty Print (จัดรูปแบบสวยกว่า p)
require 'pp'

complex_hash = {
  user: { name: 'สมชาย', age: 30 },
  orders: [1, 2, 3],
  settings: { theme: 'dark', language: 'th' }
}

pp complex_hash
# ได้ Output ที่จัดรูปแบบสวย

# เทคนิค Debug ด้วย tap
result = [1, 2, 3, 4, 5]
  .tap { |a| puts "Before filter: #{a}" }
  .select { |n| n > 2 }
  .tap { |a| puts "After filter: #{a}" }
  .map { |n| n * 2 }
  .tap { |a| puts "After map: #{a}" }

# then/yield_self สำหรับ Debug
value = "  Hello World  "
  .then { |v| puts "Original: '#{v}'"; v }
  .strip
  .then { |v| puts "Stripped: '#{v}'"; v }
  .downcase
  .then { |v| puts "Downcased: '#{v}'"; v }
```

### Debug Helper

```ruby
# Debug Helper Module
module Debug
  def self.log(label, value, caller_info: caller(1).first)
    puts "[DEBUG] #{caller_info} - #{label}: #{value.inspect}"
    value  # คืนค่า value เพื่อ Method Chaining
  end
  
  def self.trace(label = nil)
    caller.first(5).each { |c| puts "  #{c}" }
  end
  
  def self.inspect_all(binding_ctx)
    binding_ctx.local_variables.each do |var|
      val = binding_ctx.local_variable_get(var)
      puts "  #{var} = #{val.inspect}"
    end
  end
end

# การใช้งาน
x = 10
y = Debug.log("x", x)
z = Debug.log("y * 2", y * 2)

def complex_method(a, b)
  result = a + b
  Debug.inspect_all(binding)  # แสดงทุก Local Variable
  result
end

complex_method(5, 3)
```

---

## ขั้นตอนที่ 607: Pry Debugger

### การติดตั้งและใช้งาน Pry

```bash
gem install pry
gem install pry-doc
gem install pry-byebug  # สำหรับ Debugging
```

```ruby
require 'pry'

# รัน Pry Session
def process_user(user)
  data = prepare_data(user)
  binding.pry  # หยุดและเปิด Pry Session ที่นี่
  result = calculate(data)
  result
end

# ใน Pry Session สามารถ:
# - ดูค่า Variable
# - รัน Ruby Code
# - ดู Method Definition
# - Navigate Stack Frame
```

### Pry Commands

```
# พื้นฐาน
help          # แสดงคำสั่งทั้งหมด
exit          # ออกจาก Pry Session
quit          # ออกจาก Pry Session
!!!           # ออกจาก Pry Session ทันที

# Navigation
ls            # แสดง Methods และ Variables ที่ available
ls -l         # แสดง Local Variables
ls User       # แสดง Methods ของ User class
cd user       # เข้าไปใน user object's context
nesting       # ดู Context Nesting
cd ..         # กลับ Context ก่อนหน้า

# Introspection  
show-method   # แสดง Source Code ของ Method
? method_name # ดู Documentation
wtf?          # แสดง Exception ล่าสุด
$             # แสดง Exception ล่าสุด (เหมือน wtf?)

# History
hist          # แสดง History
hist --replay # รัน Commands ใหม่

# Edit
edit          # เปิด Editor ปัจจุบัน
edit -n       # Edit ที่บรรทัด n
```

### Pry Configuration

```ruby
# .pryrc (ในโฟลเดอร์ Home)
Pry.config.editor = 'vim'

# Custom Prompt
Pry.config.prompt = Pry::Prompt.new(
  'custom',
  'Custom Prompt',
  [
    proc { |obj, nest_level, _| "#{RUBY_VERSION} (#{obj})> " },
    proc { |obj, nest_level, _| "#{RUBY_VERSION} (#{obj})* " }
  ]
)

# ตั้งค่า Colors
Pry.config.color = true

# History
Pry.config.history_save = true
Pry.config.history_file = '~/.pry_history'

# เพิ่ม Commands
Pry::Commands.block_command 'hello' do
  puts "สวัสดีจาก Custom Pry Command!"
end

# Aliases
Pry.config.command_prefix = '%'
```

---

## ขั้นตอนที่ 608: binding.pry

### ใช้ binding.pry ใน Code

```ruby
class OrderProcessor
  def process(order)
    validate_order(order)
    
    binding.pry  # หยุดที่นี่เพื่อ Debug
    
    calculate_total(order)
    charge_payment(order)
    send_confirmation(order)
  end
  
  private
  
  def validate_order(order)
    raise "Invalid order" unless order.valid?
  end
  
  def calculate_total(order)
    order.items.sum { |item| item.price * item.quantity }
  end
  
  def charge_payment(order)
    PaymentGateway.charge(order.payment_method, order.total)
  end
  
  def send_confirmation(order)
    OrderMailer.confirmation(order).deliver_later
  end
end

# เมื่อรันโค้ด จะหยุดที่ binding.pry และเปิด Pry Session
# ดู Variables ที่ Available:
# [1] pry(#<OrderProcessor>)> order
# => #<Order id=123, status="pending", total=nil>
# [2] pry(#<OrderProcessor>)> order.items.count
# => 3
# [3] pry(#<OrderProcessor>)> calculate_total(order)
# => 1500
```

### Conditional Breakpoint

```ruby
def process_items(items)
  items.each_with_index do |item, index|
    binding.pry if item[:price] > 10_000  # Conditional Breakpoint!
    
    process_item(item)
  end
end

# หยุดเฉพาะเมื่อ price > 10,000

# Pry ใน Rails
class UsersController < ApplicationController
  def update
    @user = User.find(params[:id])
    
    if @user.update(user_params)
      redirect_to @user
    else
      binding.pry if Rails.env.development?  # Debug เฉพาะใน Development
      render :edit
    end
  end
end
```

---

## ขั้นตอนที่ 609: Byebug

### การใช้งาน Byebug

```bash
gem install byebug
```

```ruby
require 'byebug'

def divide(a, b)
  byebug  # Pause และเปิด Debugger
  result = a / b
  result
end

divide(10, 2)
```

### Byebug Commands

```
# Execution Control
next (n)    - ไปบรรทัดถัดไป (ไม่เข้า Method)
step (s)    - ไปบรรทัดถัดไป (เข้า Method)
continue (c) - ทำงานต่อจนถึง Breakpoint ถัดไป
finish (f)  - ออกจาก Method ปัจจุบัน
quit (q)    - ออกจาก Byebug

# Breakpoints
break n         - ตั้ง Breakpoint ที่บรรทัด n
break Class#method - Breakpoint ที่ต้น Method
info break      - แสดง Breakpoints ทั้งหมด
delete n        - ลบ Breakpoint n

# Information
list            - แสดงโค้ด
up              - ขึ้นไป Stack Frame ข้างบน
down            - ลงมา Stack Frame ข้างล่าง
where           - แสดง Stack Trace ปัจจุบัน
info locals     - แสดง Local Variables
info args       - แสดง Arguments ของ Method ปัจจุบัน

# Evaluate
eval expression - ประเมินค่า Expression
display expr    - แสดงค่า expr ทุกครั้งที่หยุด
undisplay n     - หยุด Auto Display
```

---

## ขั้นตอนที่ 610: Ruby Debugger (debug gem, Ruby 3.1+)

### debug Gem

```bash
# Ruby 3.1+ มี debug gem built-in
gem install debug
```

```ruby
require 'debug'

def factorial(n)
  debugger  # หรือ binding.break
  return 1 if n <= 1
  n * factorial(n - 1)
end

puts factorial(5)
```

### Debug Commands

```
# REPL Commands
p expr        - แสดง inspect
pp expr       - Pretty Print
info           - แสดง Local Variables

# Flow Control
s[tep]        - Step into
n[ext]        - Next line
fin[ish]      - Finish current frame
c[ontinue]    - Continue

# Breakpoints
b[reak] line  - ตั้ง Breakpoint
b Class#method
catch ExceptionClass
del[ete] n    - ลบ Breakpoint

# Stack
bt            - Backtrace
frame n       - ไปที่ Frame n
up            - ขึ้น Frame
down          - ลง Frame
```

### Remote Debugging

```ruby
# เปิด Debug Server
require 'debug'
DEBUGGER__::CONFIG[:port] = 12345
DEBUGGER__::CONFIG[:host] = '0.0.0.0'

binding.break  # รอ Connection

# Connect จาก Terminal อื่น
rdbg -A 12345
```

---

## ขั้นตอนที่ 611: อ่าน Stack Trace

### ตัวอย่าง Stack Trace

```
Traceback (most recent call last):
        5: from /path/to/app.rb:25:in `<main>'
        4: from /path/to/app.rb:18:in `process_order'
        3: from /path/to/app.rb:12:in `calculate_discount'
        2: from /path/to/app.rb:8:in `apply_coupon'
        1: from /path/to/app.rb:4:in `validate_coupon'
/path/to/app.rb:4:in `validate_coupon': undefined method `expired?' for nil:NilClass (NoMethodError)
```

### วิธีอ่าน Stack Trace

```ruby
# อ่านจากล่างขึ้นบน:
# บรรทัดสุดท้าย: Error ที่เกิด (NoMethodError: expired?)
# บรรทัด 1: ตำแหน่งที่ Error เกิด (app.rb:4 in validate_coupon)
# บรรทัด 2-5: Call Stack (เรียกมาจากไหน)

# Debug: ตรวจสอบว่า coupon เป็น nil ทำไม
def validate_coupon(code)
  coupon = Coupon.find_by(code: code)  # อาจ return nil
  
  # ปัญหา: ไม่ check nil
  if coupon.expired?  # NoMethodError เมื่อ coupon เป็น nil!
    raise "Coupon หมดอายุแล้ว"
  end
  coupon
end

# Fix: ตรวจสอบ nil ก่อน
def validate_coupon(code)
  coupon = Coupon.find_by(code: code)
  raise "ไม่พบ Coupon" if coupon.nil?
  raise "Coupon หมดอายุแล้ว" if coupon.expired?
  coupon
end

# หรือใช้ Safe Navigation Operator
def validate_coupon(code)
  coupon = Coupon.find_by(code: code)
  raise "ไม่พบ Coupon" unless coupon&.valid_code?
  coupon
end
```

---

## ขั้นตอนที่ 612: Common Ruby Errors

### Error Types และวิธีแก้

```ruby
# 1. NoMethodError
nil.upcase
# => NoMethodError: undefined method `upcase' for nil:NilClass

# Fix:
str = nil
str&.upcase  # Safe Navigation - คืน nil แทน Error
str.to_s.upcase  # แปลงเป็น String ก่อน

# 2. NameError / Undefined Variable
puts undefined_var
# => NameError: undefined local variable or method `undefined_var'

# 3. ArgumentError - ส่ง Arguments ผิดจำนวน
def greet(name)
  "สวัสดี #{name}"
end

greet  # ArgumentError: wrong number of arguments (given 0, expected 1)
greet("ก", "ข")  # ArgumentError: wrong number of arguments (given 2, expected 1)

# Fix: ใช้ Default Arguments
def greet(name = "ทุกคน")
  "สวัสดี #{name}"
end

# 4. TypeError
"hello" + 5
# => TypeError: no implicit conversion of Integer into String

# Fix:
"hello" + 5.to_s  # "hello5"
"hello #{5}"      # "hello5"

# 5. ZeroDivisionError
10 / 0
# => ZeroDivisionError: divided by 0

# Fix:
def safe_divide(a, b)
  return nil if b.zero?
  a / b
end

# 6. LoadError
require 'nonexistent_gem'
# => LoadError: cannot load such file

# Fix: ตรวจสอบ Gemfile

# 7. FrozenError
str = "hello".freeze
str << " world"
# => FrozenError: can't modify frozen String

# Fix: dup ก่อน modify
str = "hello".freeze
mutable = str.dup
mutable << " world"  # OK

# 8. StackOverflow
def infinite_recursion
  infinite_recursion
end
# => SystemStackError: stack level too deep

# Fix: เพิ่ม Base Case
def factorial(n)
  return 1 if n <= 1  # Base Case
  n * factorial(n - 1)
end

# 9. Encoding::UndefinedConversionError
# เมื่อแปลง Encoding ไม่ได้
str = "\xFF\xFE".encode('UTF-8', 'ASCII')
# => Encoding::UndefinedConversionError

# Fix:
str = "\xFF\xFE".encode('UTF-8', 'ASCII', invalid: :replace, undef: :replace)

# 10. Timeout::Error
require 'timeout'

begin
  Timeout.timeout(5) do
    # Code ที่อาจใช้เวลานาน
    sleep(10)
  end
rescue Timeout::Error
  puts "หมดเวลา!"
end
```

---

## ขั้นตอนที่ 613: Logger

### Ruby Logger

```ruby
require 'logger'

# สร้าง Logger
logger = Logger.new(STDOUT)  # Log ไปยัง Console

# Log ไปยังไฟล์
file_logger = Logger.new('/var/log/app.log')

# Log ไปยัง STDERR
error_logger = Logger.new(STDERR)

# Rotating Log File
rotating_logger = Logger.new('app.log', 'daily')    # ล้างทุกวัน
monthly_logger  = Logger.new('app.log', 'monthly')  # ล้างทุกเดือน
size_logger     = Logger.new('app.log', 5, 10.megabytes)  # Max 5 files, 10MB each

# Log Levels
logger.level = Logger::DEBUG  # แสดงทุก Level

logger.debug("Debug: ข้อมูล Debug")
logger.info("Info: การทำงานปกติ")
logger.warn("Warn: สิ่งที่น่าเป็นห่วง")
logger.error("Error: Error ที่จัดการได้")
logger.fatal("Fatal: Error ร้ายแรง")

# Format
logger.formatter = proc do |severity, datetime, progname, msg|
  "[#{datetime.strftime('%Y-%m-%d %H:%M:%S')}] #{severity}: #{msg}\n"
end

# การใช้งานใน Class
class PaymentService
  def initialize
    @logger = Logger.new(STDOUT)
    @logger.progname = 'PaymentService'
  end
  
  def process(payment)
    @logger.info("กำลังประมวลผลการชำระเงิน #{payment[:id]}")
    
    begin
      result = gateway.charge(payment)
      @logger.info("ชำระเงินสำเร็จ: #{result[:transaction_id]}")
      result
    rescue PaymentError => e
      @logger.error("ชำระเงินไม่สำเร็จ: #{e.message}")
      @logger.debug("Payment data: #{payment.inspect}")
      raise
    end
  end
end
```

### Rails Logger

```ruby
# ใน Rails ใช้ Rails.logger
Rails.logger.debug "Debug message"
Rails.logger.info  "Info message"
Rails.logger.warn  "Warning"
Rails.logger.error "Error occurred"

# Log ใน Controller
class OrdersController < ApplicationController
  def create
    Rails.logger.tagged("OrdersController", "create") do
      Rails.logger.info "สร้าง Order สำหรับ User #{current_user.id}"
      
      @order = Order.create!(order_params)
      
      Rails.logger.info "Order ##{@order.id} สร้างสำเร็จ"
      
      redirect_to @order
    end
  rescue ActiveRecord::RecordInvalid => e
    Rails.logger.warn "Order สร้างไม่สำเร็จ: #{e.message}"
    render :new
  end
end

# Structured Logging
Rails.logger.info({
  event: 'order_created',
  order_id: order.id,
  user_id: current_user.id,
  amount: order.total,
  timestamp: Time.current.iso8601
}.to_json)

# Log Level ใน Development
# config/environments/development.rb
config.log_level = :debug

# Production
# config.log_level = :info

# Silence Logger (ปิดชั่วคราว)
Rails.logger.silence do
  # Code ที่ไม่ต้องการ Log
  bulk_import_data
end
```

---

## ขั้นตอนที่ 614-618: Advanced Debugging

### Exception Handling สำหรับ Debug

```ruby
# ดักจับ Exception แบบละเอียด
begin
  complex_operation
rescue => e
  puts "Exception Class: #{e.class}"
  puts "Message: #{e.message}"
  puts "Backtrace:"
  e.backtrace.first(10).each { |line| puts "  #{line}" }
  raise  # Re-raise เพื่อให้ Program หยุด
end

# rescue สำหรับหลาย Exception Types
begin
  risky_operation
rescue ArgumentError => e
  puts "Argument Error: #{e.message}"
rescue RuntimeError => e
  puts "Runtime Error: #{e.message}"
rescue StandardError => e
  puts "Other Error: #{e.class} - #{e.message}"
ensure
  puts "ทำงานเสมอ (cleanup)"
end

# Custom Exception สำหรับ Debug
class OrderValidationError < StandardError
  attr_reader :order, :field
  
  def initialize(message, order: nil, field: nil)
    super(message)
    @order = order
    @field = field
  end
end

def validate_order(order)
  unless order.items.any?
    raise OrderValidationError.new(
      "Order ต้องมีสินค้าอย่างน้อย 1 รายการ",
      order: order,
      field: :items
    )
  end
end

begin
  validate_order(empty_order)
rescue OrderValidationError => e
  puts e.message
  puts "Order ID: #{e.order&.id}"
  puts "Field: #{e.field}"
end
```

### Debugging Rails

```ruby
# ใช้ debug gem ใน Rails
# Gemfile
# gem 'debug', group: [:development, :test]

# ใน Code
def complex_action
  order = Order.find(params[:id])
  debugger  # หรือ binding.break
  order.process!
end

# Web Console (ใน Development)
# Gemfile: gem 'web-console', group: :development
# ใน View:
# <% console %>

# Better Errors (ใน Development)
# Gemfile: gem 'better_errors', group: :development
# ให้ Error Page ที่ดีกว่าพร้อม REPL

# Rails Console
# rails console
# ดีสำหรับ Explore Database

# reload! - Reload Code ใน Console
# reload!

# ค้นหา Object
User.find_by(email: 'test@example.com')
Order.where(status: :pending).count
Product.pluck(:name, :price)
```

---

## แบบฝึกหัดบทที่ 28 (20 ข้อ)

**ข้อ 1:** ใช้ `tap` Debug Method Chain ที่ซับซ้อน

**ข้อ 2:** สร้าง Debug Helper Module ที่มี log, trace, inspect_all

**ข้อ 3:** ใช้ Pry สำรวจ Ruby Class และ Method Documentation

**ข้อ 4:** Debug Code ที่มี N+1 Query Problem ด้วย Bullet gem

**ข้อ 5:** สร้าง Custom Logger ที่เขียน Log ไปยังทั้ง File และ STDOUT

**ข้อ 6:** Debug Race Condition ด้วย Thread.current[:name]

**ข้อ 7:** ใช้ `binding.pry` Debug Complex Recursion

**ข้อ 8:** อ่านและอธิบาย Stack Trace ของ Exception

**ข้อ 9:** สร้าง Error Tracking โดย Catch และ Log Exception

**ข้อ 10:** Debug Memory Leak ด้วย ObjectSpace

**ข้อ 11:** ใช้ Byebug Debug Algorithm ทีละ Step

**ข้อ 12:** สร้าง Conditional Breakpoint ด้วย Pry

**ข้อ 13:** Debug Rails Controller ด้วย Request Logging

**ข้อ 14:** ใช้ `pp` Debug Complex Nested Hash/Array

**ข้อ 15:** สร้าง Performance Logger ที่วัดเวลาทำงาน

**ข้อ 16:** Debug ด้วย `caller` และ `__method__`

**ข้อ 17:** ใช้ debug gem ตั้ง Breakpoint แบบ Conditional

**ข้อ 18:** สร้าง Exception Reporter ที่ส่ง Error ไปยัง Slack

**ข้อ 19:** Debug Encoding Issues ใน Thai Text Processing

**ข้อ 20:** ตั้งค่า Rails Logger สำหรับ Production ด้วย Structured Logging

---

### เฉลยตัวอย่าง ข้อ 5: Custom Logger

```ruby
require 'logger'

class AppLogger
  LEVELS = %i[debug info warn error fatal].freeze
  
  def initialize(log_file: nil, level: :info)
    @loggers = []
    
    # Console Logger
    console_logger = Logger.new(STDOUT)
    console_logger.formatter = method(:format_message)
    @loggers << console_logger
    
    # File Logger (ถ้าระบุ)
    if log_file
      FileUtils.mkdir_p(File.dirname(log_file))
      file_logger = Logger.new(log_file, 'daily')
      file_logger.formatter = method(:format_json)
      @loggers << file_logger
    end
    
    set_level(level)
  end
  
  LEVELS.each do |level|
    define_method(level) do |message, context = {}|
      log(level, message, context)
    end
  end
  
  def tagged(*tags)
    old_tags = Thread.current[:log_tags] || []
    Thread.current[:log_tags] = old_tags + tags
    yield
  ensure
    Thread.current[:log_tags] = old_tags
  end
  
  def silence
    original_level = @loggers.first.level
    @loggers.each { |l| l.level = Logger::FATAL + 1 }
    yield
  ensure
    @loggers.each { |l| l.level = original_level }
  end
  
  private
  
  def log(level, message, context = {})
    @loggers.each { |logger| logger.send(level, message) }
  end
  
  def set_level(level)
    log_level = Logger.const_get(level.upcase)
    @loggers.each { |l| l.level = log_level }
  end
  
  def format_message(severity, datetime, _progname, msg)
    tags = Thread.current[:log_tags]&.map { |t| "[#{t}]" }&.join(' ')
    prefix = tags ? "#{tags} " : ""
    
    color = case severity
    when 'DEBUG' then "\e[37m"
    when 'INFO'  then "\e[32m"
    when 'WARN'  then "\e[33m"
    when 'ERROR' then "\e[31m"
    when 'FATAL' then "\e[35m"
    else ""
    end
    
    "#{color}[#{datetime.strftime('%H:%M:%S')}] #{severity}: #{prefix}#{msg}\e[0m\n"
  end
  
  def format_json(severity, datetime, _progname, msg)
    {
      timestamp: datetime.iso8601,
      level: severity,
      message: msg.to_s,
      tags: Thread.current[:log_tags] || [],
      thread_id: Thread.current.object_id
    }.to_json + "\n"
  end
end

# การใช้งาน
logger = AppLogger.new(
  log_file: 'logs/app.log',
  level: :debug
)

logger.info "Application เริ่มทำงาน"
logger.debug "Debug mode เปิดอยู่"

logger.tagged("Request", "123") do
  logger.info "รับ Request"
  logger.warn "Response ช้า"
end

logger.error "เกิดข้อผิดพลาด"

logger.silence do
  logger.debug "ข้อความนี้จะไม่แสดง"
end

logger.info "หลัง Silence"
```

---

*สรุปบทที่ 28: Debugging เป็นทักษะสำคัญที่ต้องพัฒนา ตั้งแต่วิธีพื้นฐานอย่าง puts/p/pp ไปจนถึง Interactive Debugger อย่าง Pry และ Byebug การอ่าน Stack Trace ให้เป็น การใช้ Logger อย่างเหมาะสม และการตั้ง Conditional Breakpoints ช่วยให้หา Bug ได้เร็วและมีประสิทธิภาพมากขึ้น*
