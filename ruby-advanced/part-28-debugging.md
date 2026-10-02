# ตอนที่ 28: Debugging เชิงลึก (Steps 606-625)

## บทนำ

Debugging คือกระบวนการค้นหาและแก้ไข bugs ในโค้ด ทักษะ debugging ที่ดีเป็นสิ่งจำเป็นสำหรับ developer ทุกคน บทนี้จะสอนเครื่องมือและเทคนิคการ debug ใน Ruby อย่างละเอียด

---

## Step 606: puts / p / pp สำหรับ Debug

### puts - Print ค่าและ newline

```ruby
# puts แปลง object เป็น string ด้วย to_s
name = "สมชาย"
age = 25
data = [1, 2, 3]

puts name      # => สมชาย
puts age       # => 25
puts data      # แสดงแต่ละ element บนบรรทัดใหม่:
               # 1
               # 2
               # 3

puts nil       # แสดงบรรทัดว่าง
puts true      # => true
puts [nil, 1]  # แสดง nil เป็นบรรทัดว่าง, แล้ว 1
```

### p - Debug-friendly output

```ruby
# p ใช้ inspect method - แสดงข้อมูลตามจริง
name = "สมชาย"
p name      # => "สมชาย"  (มี quotes)

nil_val = nil
p nil_val   # => nil  (ไม่ใช่ blank line)

arr = [1, "two", :three, nil]
p arr       # => [1, "two", :three, nil]

# p returns ค่า (useful สำหรับ debug ใน chain)
result = p [1, 2, 3].map { |n| n * 2 }
# พิมพ์: [2, 4, 6]
# result = [2, 4, 6]
```

### pp - Pretty Print

```ruby
require 'pp'

# pp จัดรูปแบบ output ให้อ่านง่ายสำหรับ complex objects
user = {
  name: "สมชาย",
  age: 25,
  address: {
    street: "ถนนสุขุมวิท",
    city: "กรุงเทพ",
    zip: "10110"
  },
  hobbies: ["อ่านหนังสือ", "เล่นกีตาร์", "โปรแกรมมิ่ง"]
}

puts user.inspect  # ยาวมาก ยากอ่าน
pp user            # จัดรูปแบบสวยงาม

# ผลลัพธ์ pp:
# {:name=>"สมชาย",
#  :age=>25,
#  :address=>
#   {:street=>"ถนนสุขุมวิท", :city=>"กรุงเทพ", :zip=>"10110"},
#  :hobbies=>["อ่านหนังสือ", "เล่นกีตาร์", "โปรแกรมมิ่ง"]}
```

### Debug Output with Context

```ruby
# เพิ่ม context ให้กับ debug output
def debug_print(label, value)
  puts "DEBUG [#{caller_locations.first}] #{label}: #{value.inspect}"
end

# หรือใช้ __method__ และ __LINE__
def some_method
  x = 42
  puts "DEBUG #{__method__}:#{__LINE__} x=#{x.inspect}"
end

# เปิด/ปิด debug โดยใช้ ENV variable
DEBUG = ENV['DEBUG'] == 'true'

def debug(*args)
  return unless DEBUG
  puts "[DEBUG] #{args.map(&:inspect).join(', ')}"
end

# รัน: DEBUG=true ruby myprogram.rb
```

---

## Step 607: pp vs inspect vs to_s

### ความแตกต่าง

```ruby
class Person
  attr_accessor :name, :age

  def initialize(name, age)
    @name = name
    @age = age
  end

  def to_s
    "#{@name} (#{@age} ปี)"
  end

  def inspect
    "#<Person name=#{@name.inspect} age=#{@age.inspect}>"
  end
end

person = Person.new("สมชาย", 25)

puts person.to_s       # => สมชาย (25 ปี)
puts person.inspect    # => #<Person name="สมชาย" age=25>
puts person            # เรียก to_s => สมชาย (25 ปี)
p person               # เรียก inspect => #<Person name="สมชาย" age=25>
pp person              # เรียก inspect พร้อม pretty print
```

### เมื่อไหร่ใช้อะไร?

```ruby
# to_s: สำหรับแสดงให้ผู้ใช้เห็น
puts "ยินดีต้อนรับ #{person}"  # => ยินดีต้อนรับ สมชาย (25 ปี)

# inspect: สำหรับ debugging
puts person.inspect  # => #<Person name="สมชาย" age=25>

# p: ใช้ใน debug session
p person             # เหมือน puts person.inspect

# pp: สำหรับ complex nested objects
data = {
  users: [person, Person.new("สมหญิง", 30)],
  count: 2
}
pp data

# ผลลัพธ์:
# {:users=>[#<Person name="สมชาย" age=25>, #<Person name="สมหญิง" age=30>],
#  :count=>2}
```

---

## Step 608: Pry Debugger

### ติดตั้ง Pry

```bash
# ติดตั้ง
gem install pry pry-byebug pry-rails

# Gemfile
group :development, :test do
  gem 'pry-byebug'
  gem 'pry-rails'  # สำหรับ Rails
end
```

### การใช้งาน Pry พื้นฐาน

```ruby
# เพิ่ม binding.pry ในจุดที่ต้องการ debug
def calculate_total(cart_items)
  binding.pry  # โปรแกรมจะหยุดตรงนี้

  total = cart_items.sum { |item| item[:price] * item[:quantity] }
  tax = total * 0.07
  total + tax
end

cart = [
  { name: "Ruby Book", price: 350, quantity: 2 },
  { name: "Rails Guide", price: 420, quantity: 1 }
]

calculate_total(cart)
```

### Pry Session ตัวอย่าง

```
From: app.rb @ line 3 Object#calculate_total:

    1: def calculate_total(cart_items)
 => 3:   binding.pry
    4:
    5:   total = cart_items.sum { |item| item[:price] * item[:quantity] }
    6:   tax = total * 0.07
    7:   total + tax
    8: end

[1] pry(main)> cart_items
=> [{:name=>"Ruby Book", :price=>350, :quantity=>2},
    {:name=>"Rails Guide", :price=>420, :quantity=>1}]

[2] pry(main)> cart_items.first[:price]
=> 350

[3] pry(main)> cart_items.sum { |item| item[:price] * item[:quantity] }
=> 1120
```

---

## Step 609: binding.pry - เข้า Debugger ตรงจุด

### ใช้ binding.pry ใน Methods ต่างๆ

```ruby
class OrderProcessor
  def process(order)
    validate_order(order)
    calculate_totals(order)
    apply_discounts(order)
    finalize(order)
  end

  private

  def validate_order(order)
    binding.pry if order[:items].empty?  # conditional breakpoint
    # ...
  end

  def calculate_totals(order)
    subtotal = order[:items].sum { |i| i[:price] * i[:quantity] }
    binding.pry  # ดู subtotal ก่อนไปต่อ
    order[:subtotal] = subtotal
  end
end
```

### binding.pry ใน Blocks และ Loops

```ruby
[1, 2, 3, "four", 5].each do |item|
  binding.pry if item.is_a?(String)  # หยุดเมื่อเจอ string
  result = item * 2
end

# ใน Rails - debug ใน view
# app/views/products/index.html.erb
# <% binding.pry %>
# <% @products.each do |product| %>
#   ...
# <% end %>
```

### Pry ใน IRB

```ruby
# เริ่ม pry แบบ standalone
require 'pry'

x = 42
binding.pry  # เข้า pry session

# ในไฟล์ .pryrc (home directory)
# Pry.config.prompt = Pry::Prompt[:simple]
# Pry.config.editor = 'vim'
```

---

## Step 610: Pry Commands

### ls - แสดง methods และ variables

```
[1] pry(main)> ls String
Constants from String:
  BINARY, UTF_8

Class methods from String:
  new, try_convert

Instance methods from String:
  %, *, +, <<, <=>, ==, =~, [], []=, ascii_only?, b, bytes, bytesize,
  byteslice, capitalize, capitalize!, center, chars, chomp, chomp!, chop,
  chop!, chr, clear, codepoints, count, crypt, delete, ...

[2] pry(main)> user = { name: "สมชาย", age: 25 }
[3] pry(main)> ls user
Hash#methods:
  ... (hash methods)
locals: user
```

### cd - เปลี่ยน context

```
[1] pry(main)> class Dog
[1] pry(main)*   def initialize(name)
[1] pry(main)*     @name = name
[1] pry(main)*   end
[1] pry(main)*   def bark
[1] pry(main)*     "Woof!"
[1] pry(main)*   end
[1] pry(main)* end
[2] pry(main)> dog = Dog.new("บัดดี้")
[3] pry(main)> cd dog
[4] pry(#<Dog>)> @name
=> "บัดดี้"
[5] pry(#<Dog>)> bark
=> "Woof!"
[6] pry(#<Dog>)> cd ..  # กลับ
```

### show-source

```
[1] pry(main)> show-source Array#flatten

From: array.c @ line 6323:
Owner: Array
Visibility: public
Number of lines: 10

static VALUE
rb_ary_flatten(int argc, VALUE *argv, VALUE ary)
{
  ...
}

[2] pry(main)> show-source String#upcase

# Ruby source code จะแสดง
```

### whereami และ commands อื่นๆ

```
[1] pry(main)> whereami

From: example.rb @ line 15 Object#my_method:

    10: def my_method
    11:   x = 5
    12:   y = 10
    13:   
    14:   # Comment
 => 15:   binding.pry
    16:   
    17:   x + y
    18: end

# Commands อื่นๆ
[2] pry(main)> help      # แสดง commands ทั้งหมด
[3] pry(main)> history   # แสดง command history
[4] pry(main)> exit      # ออกจาก pry
[5] pry(main)> !!!       # exit โดยโยน exception
[6] pry(main)> show-doc Array#each  # แสดง documentation
```

---

## Step 611: Byebug

### ติดตั้งและใช้งาน

```ruby
# gem 'byebug'

require 'byebug'

def factorial(n)
  byebug  # เริ่ม byebug session
  return 1 if n <= 1
  n * factorial(n - 1)
end

factorial(5)
```

### Byebug Commands

```
# step - เข้าไปใน method
# next - ข้ามไปบรรทัดถัดไป (ไม่เข้าใน method)
# continue/c - ทำงานต่อจนถึง breakpoint ถัดไป
# break/b - ตั้ง breakpoint
# list/l - แสดงโค้ดรอบๆ ตำแหน่งปัจจุบัน
# where/bt - แสดง backtrace
# display - แสดงค่าตัวแปรอัตโนมัติ

(byebug) step
[1, 9] in factorial.rb
    1: require 'byebug'
    2:
    3: def factorial(n)
    4:   byebug
=>  5:   return 1 if n <= 1
    6:   n * factorial(n - 1)
    7: end
    8:
    9: factorial(5)

(byebug) n  # next line
(byebug) p n  # print n
5
(byebug) b 6  # set breakpoint at line 6
Breakpoint 1 at factorial.rb:6
(byebug) c  # continue
Breakpoint 1 at factorial.rb:6
(byebug) p n
4
```

### Conditional Breakpoints

```ruby
def process_items(items)
  items.each_with_index do |item, index|
    byebug if item[:price] > 1000  # หยุดเมื่อราคาเกิน 1000
    process_single(item)
  end
end

# Breakpoint ใน byebug
# (byebug) break 15 if n > 3
```

---

## Step 612: Ruby Debugger (debug gem, Ruby 3.1+)

### การใช้งาน debug gem

```ruby
# ติดตั้ง (built-in Ruby 3.1+)
# gem 'debug'

require 'debug'

def calculate(a, b)
  debugger  # หรือ binding.break
  result = a + b
  result
end

calculate(5, 3)
```

### Debug Commands (Ruby debugger)

```
# คล้าย byebug แต่มีฟีเจอร์เพิ่มเติม
(rdbg) s    # step
(rdbg) n    # next
(rdbg) c    # continue
(rdbg) q    # quit
(rdbg) p expression  # evaluate
(rdbg) bt   # backtrace
(rdbg) l    # list
(rdbg) b 10  # break at line 10
(rdbg) b MyClass#method  # break at method
(rdbg) watch @variable  # watch variable
(rdbg) catch ExceptionName  # catch exception
```

### Remote Debugging

```ruby
# เริ่ม debug server
# RUBY_DEBUG_PORT=12345 ruby myapp.rb

# หรือในโค้ด
require 'debug/open'
DEBUGGER__::open(port: 12345)
binding.break

# Connect จาก terminal อื่น
# rdbg --attach 12345
```

---

## Step 613: อ่าน Ruby Error Messages

### Stack Trace

```
Traceback (most recent call last):
        4: from test.rb:10:in `<main>'
        3: from test.rb:7:in `a'
        2: from test.rb:4:in `b'
        1: from test.rb:1:in `c'
test.rb:1:in `c': undefined method `upcase' for nil (NoMethodError)
```

**วิธีอ่าน:**
1. อ่านจากล่างขึ้นบน (หรือจากบนลงล่าง แต่ส่วนล่างคือต้นตอ)
2. `test.rb:1:in 'c'` = ไฟล์ test.rb บรรทัด 1 ใน method c
3. Error message: `undefined method 'upcase' for nil (NoMethodError)`

```ruby
# โค้ดที่ทำให้เกิด error
def c
  nil.upcase  # บรรทัดที่ 1 ของ c
end

def b
  c           # เรียก c
end

def a
  b           # เรียก b
end

a             # เรียก a
```

---

## Step 614: Common Errors และวิธีแก้

### NoMethodError

```ruby
# Error
nil.upcase
# => NoMethodError: undefined method 'upcase' for nil:NilClass

# วิธีแก้ 1: ตรวจสอบ nil
str = nil
str.upcase if str  # ปลอดภัย

# วิธีแก้ 2: Safe navigation operator
str&.upcase  # => nil ถ้า str เป็น nil

# วิธีแก้ 3: Default value
str = nil
(str || "").upcase  # => ""

# วิธีแก้ 4: Raise ด้วย message ที่ชัดเจน
def process(str)
  raise ArgumentError, "str ต้องไม่เป็น nil" if str.nil?
  str.upcase
end
```

### NameError

```ruby
# Error
puts undefined_variable
# => NameError: undefined local variable or method 'undefined_variable'

# หรือ
class MyClass
  def my_method
    @instance_var = 42
  end

  def another_method
    puts instance_var  # ลืม @
    # => NameError: undefined local variable or method 'instance_var'
  end
end
```

### TypeError

```ruby
# Error
"hello" + 5
# => TypeError: no implicit conversion of Integer into String

# วิธีแก้
"hello" + 5.to_s   # => "hello5"
"hello" + "5"      # => "hello5"
"hello #{5}"       # => "hello5"  (interpolation แปลงอัตโนมัติ)

# ตัวอย่างอื่น
nil + 1
# => NoMethodError: undefined method '+' for nil:NilClass
```

### ArgumentError

```ruby
# Error - argument ผิดจำนวน
def greet(name, greeting)
  "#{greeting}, #{name}!"
end

greet("สมชาย")
# => ArgumentError: wrong number of arguments (given 1, expected 2)

# วิธีแก้
greet("สมชาย", "สวัสดี")  # ให้ arguments ครบ

# หรือใช้ default argument
def greet(name, greeting = "สวัสดี")
  "#{greeting}, #{name}!"
end

greet("สมชาย")           # => "สวัสดี, สมชาย!"
greet("สมชาย", "Hello")  # => "Hello, สมชาย!"
```

### LoadError / Require Error

```ruby
require 'nonexistent_gem'
# => LoadError: cannot load such file -- nonexistent_gem

# วิธีแก้
# ตรวจสอบ gem ใน Gemfile
# bundle install

# หรือ require จากไฟล์ที่ถูก path
require_relative '../lib/my_module'
require './config/settings'
```

### SystemStackError (Stack Overflow)

```ruby
# Error - Infinite recursion
def infinite
  infinite
end

infinite
# => SystemStackError: stack level too deep

# วิธีแก้ - ตรวจสอบ base case
def factorial(n)
  return 1 if n <= 1  # base case จำเป็น!
  n * factorial(n - 1)
end
```

### FrozenError

```ruby
# Error
str = "hello".freeze
str << " world"
# => FrozenError: can't modify frozen String: "hello"

# วิธีแก้
str = "hello"
str << " world"  # ไม่ freeze

# หรือสร้างใหม่
str = "hello".freeze
new_str = str + " world"  # สร้าง string ใหม่
```

---

## Step 615: Logger Class

### ใช้ Logger

```ruby
require 'logger'

# สร้าง logger
logger = Logger.new(STDOUT)  # หรือ STDERR

# หรือ log ไปยังไฟล์
logger = Logger.new('app.log')

# ตั้ง log level
logger.level = Logger::DEBUG  # DEBUG, INFO, WARN, ERROR, FATAL

# Log messages
logger.debug("Debug information")
logger.info("Process started")
logger.warn("Low disk space")
logger.error("File not found")
logger.fatal("System crash!")

# ผลลัพธ์:
# D, [2024-01-15T10:30:00.000000 #12345] DEBUG -- : Debug information
# I, [2024-01-15T10:30:00.000001 #12345]  INFO -- : Process started
```

### Logger ใน Rails

```ruby
# config/environments/development.rb
config.log_level = :debug

# ใช้ใน Controllers, Models
class OrdersController < ApplicationController
  def create
    Rails.logger.info("Creating order for user #{current_user.id}")
    
    @order = Order.new(order_params)
    if @order.save
      Rails.logger.info("Order #{@order.id} created successfully")
      redirect_to @order
    else
      Rails.logger.warn("Order creation failed: #{@order.errors.full_messages}")
      render :new
    end
  end
end
```

### Structured Logging

```ruby
class AppLogger
  def initialize
    @logger = Logger.new(STDOUT)
    @logger.formatter = proc do |severity, datetime, progname, msg|
      log_entry = {
        severity: severity,
        timestamp: datetime.utc.iso8601(3),
        message: msg
      }
      log_entry[:progname] = progname if progname
      "#{log_entry.to_json}\n"
    end
  end

  def info(message, context = {})
    @logger.info(context.empty? ? message : "#{message} | #{context.inspect}")
  end

  def error(message, exception = nil, context = {})
    if exception
      @logger.error("#{message} | #{exception.class}: #{exception.message} | #{context.inspect}")
      @logger.error(exception.backtrace.join("\n"))
    else
      @logger.error("#{message} | #{context.inspect}")
    end
  end
end

logger = AppLogger.new
logger.info("User logged in", user_id: 42, ip: "192.168.1.1")
```

---

## Step 616: Remote Debugging

### Remote Debug ด้วย debug gem

```ruby
# ในโค้ด
require 'debug/open'

# Server รอ connection
DEBUGGER__::open(port: 12345, host: '0.0.0.0')

# Client connect
# rdbg --attach 12345 --host localhost
```

### Debug ใน Docker Container

```dockerfile
# Dockerfile.development
FROM ruby:3.2

ENV RUBY_DEBUG_PORT=12345
ENV RUBY_DEBUG_HOST=0.0.0.0

RUN gem install debug
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "3000:3000"
      - "12345:12345"  # debug port
    environment:
      - RUBY_DEBUG_OPEN=true
      - RUBY_DEBUG_PORT=12345
```

---

## Step 617: Debugging Rails

### pry-rails

```ruby
# Gemfile
group :development, :test do
  gem 'pry-rails'
  gem 'pry-byebug'
end

# ใช้ใน Rails console
# rails console → เป็น pry console อัตโนมัติ

# ใน model
class User < ApplicationRecord
  def full_name
    binding.pry  # debug ใน Rails context
    "#{first_name} #{last_name}"
  end
end
```

### better_errors

```ruby
# Gemfile (development only)
group :development do
  gem 'better_errors'
  gem 'binding_of_caller'  # enable REPL ใน browser
end

# เมื่อเกิด error ใน development
# browser จะแสดง:
# - error message พร้อม backtrace
# - local variables
# - REPL ที่ context ที่ error
# - Source code ที่เกิด error
```

### Debug ใน Test

```ruby
# RSpec + Pry
describe "User creation" do
  it "creates a user" do
    user = User.new(name: "สมชาย", email: "somchai@test.com")
    binding.pry  # หยุดใน test
    expect(user.save).to be true
  end
end

# Minitest + Byebug
class UserTest < ActiveSupport::TestCase
  test "creates a user" do
    user = User.new(name: "สมชาย")
    byebug  # หยุดใน test
    assert user.save
  end
end
```

---

## Step 618: Debugging Techniques Advanced

### Tracing Method Calls

```ruby
# set_trace_func
set_trace_func proc { |event, file, line, id, binding, classname|
  case event
  when 'call'
    puts "CALL #{classname}##{id} at #{file}:#{line}"
  when 'return'
    puts "RETURN #{classname}##{id}"
  end
}

def add(a, b)
  a + b
end

add(1, 2)

set_trace_func nil  # ปิด tracing
```

### Object Inspection

```ruby
obj = "Hello"

# Methods ที่ object มี
puts obj.methods.sort.inspect
puts obj.respond_to?(:upcase)   # => true
puts obj.respond_to?(:nonexistent)  # => false

# Instance variables
class Person
  def initialize(name, age)
    @name = name
    @age = age
  end
end

p = Person.new("สมชาย", 25)
puts p.instance_variables.inspect
# => [:@name, :@age]

puts p.instance_variable_get(:@name)  # => "สมชาย"
p.instance_variable_set(:@name, "ใหม่")

# Class hierarchy
puts "Hello".class              # => String
puts "Hello".class.superclass  # => Object
puts "Hello".is_a?(String)     # => true
puts "Hello".kind_of?(Object)  # => true
```

### Caller และ Backtrace

```ruby
# ดู call stack
def level_three
  puts caller.inspect  # array of strings
  puts caller(0, 5).inspect  # แสดง 5 levels
end

def level_two
  level_three
end

def level_one
  level_two
end

level_one

# caller_locations - structured version
def where_am_i
  location = caller_locations.first
  puts "#{location.path}:#{location.lineno}:in `#{location.label}'"
end
```

---

## Step 619: ตัวอย่าง Debugging Real-world Problems

### Debug N+1 Query

```ruby
# ปัญหา
class OrdersController
  def index
    @orders = Order.all  # ดึง orders ทั้งหมด
    # ใน view: @orders.each { |o| o.user.name }  # N+1!
  end
end

# Debug ด้วย Bullet gem
# Gemfile
# gem 'bullet', group: :development

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
end

# แก้ไข
@orders = Order.includes(:user).all
```

### Debug Memory Leak

```ruby
# ตรวจหา memory leak
class PotentialLeak
  @@instances = []  # Class variable เก็บทุก instance - LEAK!

  def initialize(data)
    @data = data
    @@instances << self  # ป้องกัน GC
  end
end

# Debug ด้วย ObjectSpace
require 'objspace'

before = ObjectSpace.count_objects[:T_OBJECT]

100.times { PotentialLeak.new("some data") }

GC.start
after = ObjectSpace.count_objects[:T_OBJECT]

puts "Objects increased: #{after - before}"
# ถ้าเพิ่มขึ้น = memory leak

# แก้ไข
class FixedClass
  def initialize(data)
    @data = data
    # ไม่เก็บ reference ใน class variable
  end
end
```

---

## แบบฝึกหัด 20 ข้อ

### ข้อที่ 1: ใช้ p และ pp
```ruby
# โจทย์: Debug โค้ดต่อไปนี้
def process_users(users)
  users.map { |u| { name: u[:name].upcase, age: u[:age] + 1 } }
end

users = [
  { name: "สมชาย", age: 25 },
  { name: nil, age: 30 }  # จะเกิด error!
]

# เฉลย: เพิ่ม p/pp เพื่อ debug
def process_users_debug(users)
  users.map do |u|
    p u  # ดูแต่ละ user
    name = u[:name]
    p name  # ดู name ก่อน upcase
    { name: name&.upcase || "UNKNOWN", age: u[:age] + 1 }
  end
end
```

### ข้อที่ 2: อ่าน Stack Trace
```
# โจทย์: Stack trace ต่อไปนี้บอกอะไร?
# example.rb:5:in 'multiply': undefined method '*' for nil:NilClass (NoMethodError)
# from example.rb:10:in 'calculate'
# from example.rb:15:in '<main>'

# เฉลย:
# - Error เกิดที่ line 5 ใน method 'multiply'
# - nil ไม่มี method '*'
# - Call chain: main -> calculate -> multiply
# - แก้ไข: ตรวจสอบ nil ก่อน multiply
```

### ข้อที่ 3: Fix NoMethodError
```ruby
# เฉลย
def get_username(user)
  return "ไม่ระบุ" unless user
  return "ไม่ระบุ" if user[:name].nil?
  user[:name].upcase
end

# หรือแบบสั้น
def get_username_clean(user)
  user&.dig(:name)&.upcase || "ไม่ระบุ"
end
```

### ข้อที่ 4: Debug ด้วย binding.pry
```ruby
# เฉลย
def calculate_discount(price, discount_percent)
  binding.pry  # เพิ่มตรงนี้เพื่อ debug
  discount = price * (discount_percent / 100.0)
  price - discount
end

# ใน pry session:
# price, discount_percent ต้องไม่เป็น 0 หรือ nil
puts calculate_discount(1000, 20)  # => 800.0
```

### ข้อที่ 5: ใช้ Logger
```ruby
# เฉลย
require 'logger'

logger = Logger.new('debug.log')
logger.level = Logger::DEBUG

class Calculator
  def initialize(logger)
    @logger = logger
  end

  def divide(a, b)
    @logger.debug("divide called with a=#{a}, b=#{b}")

    if b.zero?
      @logger.error("Attempted to divide #{a} by zero!")
      raise ArgumentError, "Cannot divide by zero"
    end

    result = a / b.to_f
    @logger.info("#{a} / #{b} = #{result}")
    result
  end
end

calc = Calculator.new(logger)
calc.divide(10, 2)
calc.divide(10, 0) rescue nil
```

### ข้อที่ 6: Debug Infinite Loop
```ruby
# โจทย์: หาและแก้ infinite loop
def countdown(n)
  while n > 0
    puts n
    # Bug: ลืม n -= 1
  end
end

# เฉลย
def countdown_fixed(n)
  while n > 0
    puts n
    n -= 1  # จำเป็น!
  end
  puts "Blastoff!"
end

countdown_fixed(5)
```

### ข้อที่ 7: Trace Method Calls
```ruby
# เฉลย
class OrderProcessor
  def process(order)
    log_call(__method__, order)
    validate(order)
    calculate(order)
    save(order)
  end

  private

  def log_call(method, *args)
    puts "TRACE: #{self.class}##{method} called"
  end

  def validate(order)
    log_call(__method__, order)
    raise "Invalid order" unless order[:items].any?
  end

  def calculate(order)
    log_call(__method__, order)
    order[:total] = order[:items].sum { |i| i[:price] }
  end

  def save(order)
    log_call(__method__, order)
    puts "Order saved: #{order.inspect}"
  end
end
```

### ข้อที่ 8: Debug TypeError
```ruby
# โจทย์
def concatenate(a, b)
  a + b
end

concatenate("Hello", 5)  # TypeError!

# เฉลย
def concatenate_safe(a, b)
  [a, b].map(&:to_s).join
end

# หรือ
def concatenate_typed(a, b)
  raise TypeError, "a ต้องเป็น String" unless a.is_a?(String)
  raise TypeError, "b ต้องเป็น String" unless b.is_a?(String)
  a + b
end
```

### ข้อที่ 9: Debug ArgumentError
```ruby
# เฉลย
def create_user(name:, email:, age: nil)
  # keyword arguments ช่วยให้ error message ชัดเจน
  raise ArgumentError, "name ต้องไม่ว่าง" if name.to_s.empty?
  raise ArgumentError, "email ต้องมี @" unless email.include?('@')

  {
    name: name,
    email: email,
    age: age
  }
end

begin
  create_user(name: "", email: "invalid")
rescue ArgumentError => e
  puts "Error: #{e.message}"
end
```

### ข้อที่ 10: ใช้ byebug ใน Loop
```ruby
# เฉลย
require 'byebug'

def process_batch(items)
  items.each_with_index do |item, i|
    byebug if i == 2  # หยุดที่ item ที่ 3

    puts "Processing: #{item}"
  end
end

process_batch(["a", "b", "c", "d", "e"])
# จะหยุดก่อน process "c"
```

### ข้อที่ 11: Debug ด้วย rescue และ re-raise
```ruby
# เฉลย
def risky_operation(data)
  data[:value].upcase
rescue NoMethodError => e
  puts "DEBUG: data = #{data.inspect}"
  puts "DEBUG: data[:value] class = #{data[:value].class}"
  raise  # re-raise ด้วย original backtrace
end

begin
  risky_operation({ value: nil })
rescue => e
  puts "Caught: #{e.message}"
end
```

### ข้อที่ 12-20: Additional Exercises

**ข้อที่ 12:** สร้าง Debug middleware สำหรับ Ruby app
```ruby
# เฉลย
module DebugMiddleware
  def self.wrap(klass, method_name)
    original = klass.instance_method(method_name)
    klass.define_method(method_name) do |*args, &block|
      puts "[DEBUG] #{klass}##{method_name} called with #{args.inspect}"
      start = Time.now
      result = original.bind(self).call(*args, &block)
      elapsed = Time.now - start
      puts "[DEBUG] #{klass}##{method_name} returned #{result.inspect} in #{elapsed.round(4)}s"
      result
    end
  end
end

class Calculator
  def add(a, b); a + b; end
  def multiply(a, b); a * b; end
end

DebugMiddleware.wrap(Calculator, :add)
DebugMiddleware.wrap(Calculator, :multiply)

calc = Calculator.new
calc.add(2, 3)
calc.multiply(4, 5)
```

**ข้อที่ 13:** เขียน assert helper สำหรับ debugging
```ruby
# เฉลย
def assert(condition, message = "Assertion failed")
  unless condition
    location = caller_locations.first
    raise "#{message} at #{location.path}:#{location.lineno}"
  end
  true
end

def divide(a, b)
  assert(!b.zero?, "Cannot divide by zero (b=#{b})")
  a / b
end

divide(10, 2)  # ok
divide(10, 0)  # => RuntimeError: Cannot divide by zero (b=0) at...
```

**ข้อที่ 14:** Debug with ObjectSpace
```ruby
# เฉลย - หา string ที่ใหญ่ที่สุดใน memory
require 'objspace'

ObjectSpace.each_object(String).select { |s| s.length > 100 }.first(5).each do |s|
  puts "#{s.length} chars: #{s[0..50]}..."
end

# นับ objects ตาม class
counts = Hash.new(0)
ObjectSpace.each_object { |obj| counts[obj.class] += 1 }
counts.sort_by { |_, v| -v }.first(10).each do |klass, count|
  puts "#{klass}: #{count}"
end
```

**ข้อที่ 15:** สร้าง Error Reporter
```ruby
# เฉลย
class ErrorReporter
  def self.report(error, context = {})
    puts "=" * 50
    puts "ERROR REPORT"
    puts "=" * 50
    puts "Time: #{Time.now}"
    puts "Error: #{error.class}: #{error.message}"
    puts "Context: #{context.inspect}" unless context.empty?
    puts "\nBacktrace:"
    error.backtrace.first(10).each_with_index do |line, i|
      puts "  #{i + 1}: #{line}"
    end
    puts "=" * 50
  end
end

begin
  nil.upcase
rescue => e
  ErrorReporter.report(e, user_id: 42, action: "process_payment")
end
```

**ข้อที่ 16:** Debug Rails Request Cycle
```ruby
# เฉลย
# config/initializers/debug_request.rb
if Rails.env.development?
  ActiveSupport::Notifications.subscribe('process_action.action_controller') do |*args|
    event = ActiveSupport::Notifications::Event.new(*args)
    puts "[REQUEST] #{event.payload[:controller]}##{event.payload[:action]}"
    puts "[REQUEST] Duration: #{event.duration.round(2)}ms"
    puts "[REQUEST] DB: #{event.payload[:db_runtime]&.round(2)}ms"
    puts "[REQUEST] View: #{event.payload[:view_runtime]&.round(2)}ms"
  end
end
```

**ข้อที่ 17:** ใช้ warn สำหรับ deprecation warnings
```ruby
# เฉลย
def old_method
  warn "[DEPRECATED] old_method จะถูกลบใน version 2.0, ใช้ new_method แทน"
  warn caller.first  # แสดงว่าถูกเรียกจากไหน
  new_method
end

def new_method
  "ผลลัพธ์ใหม่"
end

old_method  # จะแสดง deprecation warning
```

**ข้อที่ 18:** Debug Encoding Issues
```ruby
# เฉลย
str = "สวัสดี"
puts str.encoding     # => UTF-8
puts str.length       # => 6 (characters)
puts str.bytesize     # => 18 (bytes, UTF-8 Thai = 3 bytes/char)
puts str.valid_encoding?  # => true

# แก้ encoding issue
def fix_encoding(str)
  return str if str.valid_encoding?
  str.encode('UTF-8', invalid: :replace, undef: :replace, replace: '?')
end
```

**ข้อที่ 19:** สร้าง Debug Proxy
```ruby
# เฉลย
class DebugProxy
  def initialize(target, logger)
    @target = target
    @logger = logger
  end

  def method_missing(name, *args, &block)
    if @target.respond_to?(name)
      @logger.debug("Calling #{@target.class}##{name}(#{args.map(&:inspect).join(', ')})")
      result = @target.send(name, *args, &block)
      @logger.debug("  => #{result.inspect}")
      result
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private) || super
  end
end

require 'logger'
logger = Logger.new(STDOUT)
proxy = DebugProxy.new([1, 2, 3], logger)
proxy.length
proxy.map { |n| n * 2 }
```

**ข้อที่ 20:** Debug Concurrent Code
```ruby
# เฉลย
require 'thread'

class ConcurrentCounter
  def initialize
    @count = 0
    @mutex = Mutex.new
    @debug_log = []
  end

  def increment(thread_id)
    @mutex.synchronize do
      old_value = @count
      @count += 1
      @debug_log << "Thread #{thread_id}: #{old_value} -> #{@count}"
    end
  end

  def value
    @count
  end

  def debug_log
    @debug_log
  end
end

counter = ConcurrentCounter.new

threads = 5.times.map do |i|
  Thread.new do
    10.times { counter.increment(i) }
  end
end

threads.each(&:join)

puts "Final count: #{counter.value}"  # => 50
puts "\nDebug log (first 10):"
counter.debug_log.first(10).each { |entry| puts entry }
```

---

## สรุป

Debugging ใน Ruby มีเครื่องมือหลายอย่าง:

1. **puts/p/pp** - เร็วและง่าย เหมาะสำหรับ debug เบื้องต้น
2. **Pry** - powerful REPL debugger ที่ใช้กันมาก
3. **byebug** - debugger แบบ step-through
4. **debug gem** - built-in ใน Ruby 3.1+
5. **Logger** - สำหรับ structured logging
6. **better_errors** - สำหรับ Rails development

**Best Practices:**
- อ่าน error message ก่อน
- อ่าน stack trace จากล่างขึ้นบน
- ลด scope ของปัญหาด้วย binary search
- เพิ่ม logging ที่ strategic points
- ใช้ conditional breakpoints เพื่อประหยัดเวลา

---

*ต่อไป: ตอนที่ 29 - Security ใน Ruby*
