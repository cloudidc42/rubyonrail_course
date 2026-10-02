# Ruby และ Ruby on Rails Interview Questions

คู่มือเตรียมตัวสัมภาษณ์งาน Ruby/Rails ครอบคลุมทุกระดับตั้งแต่ Junior ถึง Senior

---

## สารบัญ

1. [คำถาม Ruby ระดับ Junior](#ruby-junior)
2. [คำถาม Ruby ระดับ Mid-Level](#ruby-mid)
3. [คำถาม Ruby ระดับ Senior](#ruby-senior)
4. [คำถาม Rails ระดับ Junior](#rails-junior)
5. [คำถาม Rails ระดับ Mid-Level](#rails-mid)
6. [คำถาม Rails ระดับ Senior](#rails-senior)
7. [System Design Questions](#system-design)
8. [Coding Challenges พร้อมเฉลย](#coding-challenges)
9. [Behavioral Questions](#behavioral)

---

## คำถาม Ruby ระดับ Junior {#ruby-junior}

### 1. อธิบายความแตกต่างระหว่าง `nil`, `false`, และ `0` ใน Ruby

**เฉลย:**

```ruby
# nil คือ "ไม่มีค่า" หรือ "ว่างเปล่า" - เป็น object ของ NilClass
nil.class       # => NilClass
nil.nil?        # => true
nil.to_i        # => 0
nil.to_s        # => ""
nil.to_a        # => []

# false คือค่า boolean false - เป็น object ของ FalseClass
false.class     # => FalseClass
false.nil?      # => false

# 0 คือตัวเลขศูนย์ - เป็น object ของ Integer
0.class         # => Integer
0.nil?          # => false
0.zero?         # => true

# ใน Ruby มีเพียง nil และ false เท่านั้นที่เป็น "falsy"
# ทุกอย่างอื่น รวมถึง 0, "", [], {} ล้วนเป็น "truthy"
puts "nil เป็น falsy" unless nil    # พิมพ์ออกมา
puts "false เป็น falsy" unless false # พิมพ์ออกมา
puts "0 เป็น truthy" if 0           # พิมพ์ออกมา
puts "'' เป็น truthy" if ""         # พิมพ์ออกมา
puts "[] เป็น truthy" if []         # พิมพ์ออกมา
```

**ประเด็นสำคัญ:** ใน Ruby ตัวเลข 0 และ string ว่างเปล่า ถือเป็น truthy ซึ่งต่างจากภาษาอื่นเช่น JavaScript หรือ PHP

---

### 2. อธิบาย Symbol ใน Ruby และความแตกต่างกับ String

**เฉลย:**

```ruby
# Symbol คือ identifier ที่ immutable และเก็บใน memory pool
:hello.class    # => Symbol
:hello.object_id == :hello.object_id  # => true (เป็น object เดียวกัน)

# String แต่ละครั้งสร้าง object ใหม่
"hello".object_id == "hello".object_id  # => false (คนละ object)

# การแปลง
:hello.to_s     # => "hello"
"hello".to_sym  # => :hello

# ใช้ Symbol เป็น hash key แทน String เพราะเร็วกว่า
person_with_symbol = { name: "Alice", age: 30 }
person_with_string = { "name" => "Alice", "age" => 30 }

# Symbol ใช้ memory น้อยกว่า เพราะเก็บเพียงครั้งเดียว
symbols = Array.new(100) { :same_symbol }
strings = Array.new(100) { "same_string" }
# symbols ทั้ง 100 ชี้ไปที่ object เดียว
# strings สร้าง 100 objects ใหม่

# ตั้งแต่ Ruby 2.2 Symbols ที่สร้างจาก dynamic string จะถูก GC เก็บได้
dynamic_sym = "hello_#{rand}".to_sym
# symbol นี้จะถูกเก็บโดย GC เมื่อไม่ใช้แล้ว
```

---

### 3. อธิบาย Array methods ที่ใช้บ่อย

**เฉลย:**

```ruby
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

# map/collect - แปลงแต่ละ element
doubled = numbers.map { |n| n * 2 }
# => [6, 2, 8, 2, 10, 18, 4, 12, 10, 6]

# select/filter - กรอง element ที่ผ่านเงื่อนไข
evens = numbers.select { |n| n.even? }
# => [4, 2, 6]

# reject - กรอง element ที่ไม่ผ่านเงื่อนไข
odds = numbers.reject { |n| n.even? }
# => [3, 1, 1, 5, 9, 5, 3]

# reduce/inject - รวมทุก element เป็นค่าเดียว
sum = numbers.reduce(0) { |acc, n| acc + n }
# => 39

# each_with_object - สะสมผลลัพธ์ใน object
grouped = numbers.each_with_object({}) do |n, hash|
  hash[n] = (hash[n] || 0) + 1
end
# => {3=>2, 1=>2, 4=>1, 5=>2, 9=>1, 2=>1, 6=>1}

# flat_map - map แล้ว flatten หนึ่งระดับ
nested = [[1, 2], [3, 4], [5, 6]]
flat = nested.flat_map { |arr| arr.map { |n| n * 2 } }
# => [2, 4, 6, 8, 10, 12]

# zip - รวม arrays เข้าด้วยกัน
names = ["Alice", "Bob"]
ages = [30, 25]
pairs = names.zip(ages)
# => [["Alice", 30], ["Bob", 25]]

# each_slice - แบ่ง array เป็นกลุ่มๆ
numbers.each_slice(3) { |group| puts group.inspect }
# [3, 1, 4]
# [1, 5, 9]
# [2, 6, 5]
# [3]

# uniq - ลบค่าซ้ำ
unique = numbers.uniq
# => [3, 1, 4, 5, 9, 2, 6]

# sort_by - เรียงลำดับตาม criteria
words = ["banana", "apple", "cherry", "date"]
sorted = words.sort_by { |w| w.length }
# => ["date", "apple", "banana", "cherry"]
```

---

### 4. อธิบาย Hash methods ที่สำคัญ

**เฉลย:**

```ruby
person = { name: "Alice", age: 30, city: "Bangkok" }

# เข้าถึงค่า
person[:name]           # => "Alice"
person.fetch(:name)     # => "Alice"
person.fetch(:email, "N/A")  # => "N/A" (default value)
person.fetch(:email) { |k| "No #{k} found" }  # => "No email found"

# แก้ไขค่า
person[:age] = 31
person.merge!(email: "alice@example.com")

# ตรวจสอบ
person.key?(:name)      # => true
person.value?("Alice")  # => true
person.include?(:age)   # => true

# วนลูป
person.each { |key, value| puts "#{key}: #{value}" }
person.each_with_object([]) { |(k, v), arr| arr << "#{k}=#{v}" }

# แปลง
person.keys             # => [:name, :age, :city, :email]
person.values           # => ["Alice", 31, "Bangkok", "alice@example.com"]
person.to_a             # => [[:name, "Alice"], [:age, 31], ...]

# กรองและแปลง
adults = person.select { |k, v| v.is_a?(Integer) && v >= 18 }
# => {:age=>31}

stringified = person.transform_keys(&:to_s)
# => {"name"=>"Alice", "age"=>31, ...}

upcased = person.transform_values { |v| v.to_s.upcase }
# => {:name=>"ALICE", :age=>"31", ...}

# merge - รวม hashes
defaults = { role: "user", active: true }
merged = defaults.merge(person)
# person values overwrite defaults when keys conflict

# slice - ดึงเฉพาะบาง keys
subset = person.slice(:name, :age)
# => {:name=>"Alice", :age=>31}
```

---

### 5. อธิบาย Object-Oriented Programming ใน Ruby

**เฉลย:**

```ruby
# Class definition
class Animal
  # Class variable (shared across all instances)
  @@count = 0

  # Class method
  def self.count
    @@count
  end

  # Accessor methods
  attr_accessor :name
  attr_reader :species

  # Initialize (constructor)
  def initialize(name, species)
    @name = name      # Instance variable
    @species = species
    @@count += 1
  end

  # Instance method
  def speak
    "..."
  end

  def to_s
    "#{@name} (#{@species})"
  end

  protected

  def secret_method
    "This is protected"
  end

  private

  def internal_method
    "This is private"
  end
end

# Inheritance
class Dog < Animal
  def initialize(name)
    super(name, "Canis lupus familiaris")
    @tricks = []
  end

  # Override parent method
  def speak
    "Woof!"
  end

  def learn_trick(trick)
    @tricks << trick
  end

  def show_tricks
    @tricks.join(", ")
  end
end

class Cat < Animal
  def initialize(name)
    super(name, "Felis catus")
  end

  def speak
    "Meow!"
  end
end

# การใช้งาน
dog = Dog.new("Rex")
cat = Cat.new("Whiskers")

puts dog.speak     # => "Woof!"
puts cat.speak     # => "Meow!"
puts Animal.count  # => 2

dog.learn_trick("sit")
dog.learn_trick("shake")
puts dog.show_tricks  # => "sit, shake"

# Polymorphism
animals = [dog, cat]
animals.each { |animal| puts "#{animal.name} says: #{animal.speak}" }
```

---

### 6. อธิบาย Blocks ใน Ruby

**เฉลย:**

```ruby
# Block คือ anonymous function ที่ส่งเข้าไปใน method

# การส่ง block ด้วย {}
[1, 2, 3].each { |n| puts n }

# การส่ง block ด้วย do...end (ใช้เมื่อ block มีหลายบรรทัด)
[1, 2, 3].each do |n|
  squared = n ** 2
  puts "#{n} squared is #{squared}"
end

# yield - เรียกใช้ block จากภายใน method
def greet
  puts "Before block"
  yield if block_given?
  puts "After block"
end

greet { puts "Hello from block!" }
# Before block
# Hello from block!
# After block

greet  # ไม่มี block ก็รันได้เพราะใช้ block_given?

# ส่งค่าไปให้ block
def calculate(x, y)
  result = yield(x, y) if block_given?
  result
end

sum = calculate(3, 4) { |a, b| a + b }    # => 7
product = calculate(3, 4) { |a, b| a * b } # => 12

# Block สามารถเข้าถึง local variables
multiplier = 3
[1, 2, 3].map { |n| n * multiplier }  # => [3, 6, 9]

# Explicit block parameter ด้วย &
def save_block(&block)
  @saved_block = block
end

def execute_saved_block
  @saved_block.call if @saved_block
end

save_block { puts "Saved block executed!" }
execute_saved_block  # => "Saved block executed!"
```

---

## คำถาม Ruby ระดับ Mid-Level {#ruby-mid}

### 7. อธิบายความแตกต่างระหว่าง Proc และ Lambda

**เฉลย:**

```ruby
# Proc - ไม่ strict เรื่อง arguments, return ออกจาก enclosing method
my_proc = Proc.new { |x, y| puts "#{x} and #{y}" }
my_proc.call(1, 2)     # => "1 and 2"
my_proc.call(1)        # => "1 and " (ไม่ error แม้ส่ง argument ไม่ครบ)
my_proc.call(1, 2, 3)  # => "1 and 2" (ไม่ error แม้ส่งเกิน)

# Lambda - strict เรื่อง arguments, return ออกจากแค่ lambda เอง
my_lambda = lambda { |x, y| puts "#{x} and #{y}" }
# หรือใช้ stabby lambda syntax
my_lambda = ->(x, y) { puts "#{x} and #{y}" }

my_lambda.call(1, 2)     # => "1 and 2"
# my_lambda.call(1)      # => ArgumentError!
# my_lambda.call(1, 2, 3) # => ArgumentError!

# ความแตกต่างสำคัญ: พฤติกรรม return
def proc_return_demo
  my_proc = Proc.new { return "from proc" }
  my_proc.call
  "after proc"  # ไม่ถึงบรรทัดนี้!
end

def lambda_return_demo
  my_lambda = lambda { return "from lambda" }
  my_lambda.call
  "after lambda"  # ถึงบรรทัดนี้!
end

puts proc_return_demo    # => "from proc"
puts lambda_return_demo  # => "after lambda"

# ตรวจสอบว่าเป็น lambda หรือ proc
my_proc.lambda?    # => false
my_lambda.lambda?  # => true

# arity
my_proc.arity    # => 2
my_lambda.arity  # => 2

proc_with_optional = Proc.new { |x, y = 10| x + y }
proc_with_optional.arity  # => -2 (negative หมายความว่ามี optional args)
```

---

### 8. อธิบาย Modules และ Mixins

**เฉลย:**

```ruby
# Module ใช้ใน 3 วัตถุประสงค์หลัก:
# 1. Namespace - ป้องกัน name collision
# 2. Mixins - แชร์ behavior ระหว่าง classes
# 3. Collection ของ methods ที่ไม่เกี่ยวข้องกับ class

# 1. Namespace
module Geometry
  class Circle
    def initialize(radius)
      @radius = radius
    end

    def area
      Math::PI * @radius ** 2
    end
  end

  class Rectangle
    def initialize(width, height)
      @width = width
      @height = height
    end

    def area
      @width * @height
    end
  end
end

circle = Geometry::Circle.new(5)
circle.area  # => 78.53981633974483

# 2. Mixins ด้วย include (instance methods)
module Greetable
  def greet
    "Hello, I'm #{name}"
  end

  def farewell
    "Goodbye from #{name}"
  end
end

module Serializable
  def to_json_string
    vars = instance_variables.map do |var|
      key = var.to_s.delete('@')
      value = instance_variable_get(var)
      "\"#{key}\": \"#{value}\""
    end
    "{ #{vars.join(', ')} }"
  end
end

class Person
  include Greetable
  include Serializable

  attr_reader :name, :age

  def initialize(name, age)
    @name = name
    @age = age
  end
end

alice = Person.new("Alice", 30)
puts alice.greet          # => "Hello, I'm Alice"
puts alice.to_json_string # => '{ "name": "Alice", "age": "30" }'

# 3. extend - เพิ่ม module methods เป็น class methods
module ClassInfo
  def describe
    "This is #{self.name} class"
  end
end

class Dog
  extend ClassInfo
end

Dog.describe  # => "This is Dog class"

# Method Resolution Order (MRO)
class A
  def hello
    "Hello from A"
  end
end

module B
  def hello
    "Hello from B, " + super
  end
end

module C
  def hello
    "Hello from C, " + super
  end
end

class D < A
  include B
  include C  # C included last, so C is checked first
end

puts D.new.hello  # => "Hello from C, Hello from B, Hello from A"
puts D.ancestors  # => [D, C, B, A, Object, Kernel, BasicObject]
```

---

### 9. อธิบาย Metaprogramming ใน Ruby

**เฉลย:**

```ruby
# Metaprogramming คือการเขียน code ที่สร้างหรือแก้ไข code อื่นในเวลา runtime

# 1. define_method - สร้าง method แบบ dynamic
class Calculator
  [:add, :subtract, :multiply].each_with_index do |operation, i|
    define_method("perform_#{operation}") do |a, b|
      case operation
      when :add      then a + b
      when :subtract then a - b
      when :multiply then a * b
      end
    end
  end
end

calc = Calculator.new
calc.perform_add(3, 4)       # => 7
calc.perform_subtract(10, 3) # => 7
calc.perform_multiply(3, 4)  # => 12

# 2. method_missing - จัดการ method calls ที่ไม่มีอยู่
class DynamicProxy
  def initialize(target)
    @target = target
  end

  def method_missing(method_name, *args, &block)
    if @target.respond_to?(method_name)
      puts "Delegating #{method_name} to target"
      @target.send(method_name, *args, &block)
    else
      super  # ส่งต่อให้ parent จัดการ (ทำให้เกิด NoMethodError)
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    @target.respond_to?(method_name) || super
  end
end

proxy = DynamicProxy.new([1, 2, 3])
proxy.length  # => "Delegating length to target" แล้ว => 3
proxy.map { |n| n * 2 }  # => [2, 4, 6]

# 3. open classes - เพิ่ม method ให้ existing classes
class Integer
  def factorial
    return 1 if self <= 1
    self * (self - 1).factorial
  end

  def times_do_with_index
    each_with_object([]) do |i, arr|
      arr << yield(i)
    end
  end
end

5.factorial  # => 120

# 4. class_eval / module_eval - evaluate code ใน context ของ class
String.class_eval do
  def palindrome?
    self == self.reverse
  end
end

"racecar".palindrome?  # => true
"hello".palindrome?    # => false

# 5. instance_variable_get/set - เข้าถึง instance variables
class Config
  SETTINGS = %w[host port database]

  SETTINGS.each do |setting|
    define_method(setting) do
      instance_variable_get("@#{setting}")
    end

    define_method("#{setting}=") do |value|
      instance_variable_set("@#{setting}", value)
    end
  end
end

config = Config.new
config.host = "localhost"
config.port = 5432
puts config.host  # => "localhost"
puts config.port  # => 5432

# 6. attr_accessor implementation
class MyModule
  def self.my_attr_accessor(*names)
    names.each do |name|
      define_method(name) do
        instance_variable_get("@#{name}")
      end

      define_method("#{name}=") do |value|
        instance_variable_set("@#{name}", value)
      end
    end
  end
end

class Person
  extend MyModule
  my_attr_accessor :name, :age
end

p = Person.new
p.name = "Alice"
p.age = 30
puts p.name  # => "Alice"
```

---

### 10. อธิบาย Comparable และ Enumerable modules

**เฉลย:**

```ruby
# Comparable - ให้ class สามารถเปรียบเทียบได้
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  # ต้องกำหนด <=> operator เพียงอย่างเดียว
  def <=>(other)
    @degrees <=> other.degrees
  end

  def to_s
    "#{@degrees}°"
  end
end

temps = [
  Temperature.new(100),
  Temperature.new(0),
  Temperature.new(37),
  Temperature.new(-10)
]

puts temps.min    # => -10°
puts temps.max    # => 100°
puts temps.sort   # => [-10°, 0°, 37°, 100°]

t1 = Temperature.new(20)
t2 = Temperature.new(30)
puts t1 < t2     # => true
puts t1 > t2     # => false
puts t1.between?(Temperature.new(10), Temperature.new(25))  # => true
puts t1.clamp(Temperature.new(25), Temperature.new(35))     # => 25°

# Enumerable - ให้ class ทำงานกับ collection methods
class NumberList
  include Enumerable

  def initialize(*numbers)
    @numbers = numbers
  end

  # ต้องกำหนด each method เพียงอย่างเดียว
  def each(&block)
    @numbers.each(&block)
  end
end

list = NumberList.new(3, 1, 4, 1, 5, 9, 2, 6)

puts list.min        # => 1
puts list.max        # => 9
puts list.sum        # => 31
puts list.sort.inspect      # => [1, 1, 2, 3, 4, 5, 6, 9]
puts list.select(&:odd?).inspect    # => [3, 1, 1, 5, 9]
puts list.map { |n| n * 2 }.inspect # => [6, 2, 8, 2, 10, 18, 4, 12]
puts list.first(3).inspect   # => [3, 1, 4]
puts list.include?(5)        # => true
puts list.count              # => 8
puts list.group_by(&:odd?).inspect
# => {true=>[3, 1, 1, 5, 9], false=>[4, 2, 6]}
```

---

### 11. อธิบาย Exception Handling

**เฉลย:**

```ruby
# begin/rescue/ensure/raise
def divide(a, b)
  begin
    result = a / b
    puts "Result: #{result}"
  rescue ZeroDivisionError => e
    puts "Cannot divide by zero: #{e.message}"
    -1  # return value
  rescue TypeError => e
    puts "Type error: #{e.message}"
    nil
  ensure
    puts "This always runs"
  end
end

divide(10, 2)   # Result: 5 / This always runs
divide(10, 0)   # Cannot divide by zero: divided by 0 / This always runs
divide("a", 2)  # Type error: ... / This always runs

# Custom exceptions
class ApplicationError < StandardError; end

class DatabaseError < ApplicationError
  def initialize(msg = "Database operation failed")
    super(msg)
  end
end

class RecordNotFound < DatabaseError
  attr_reader :record_id

  def initialize(record_id)
    @record_id = record_id
    super("Record #{record_id} not found")
  end
end

begin
  raise RecordNotFound.new(42)
rescue RecordNotFound => e
  puts "#{e.message}, ID: #{e.record_id}"
  # => "Record 42 not found, ID: 42"
rescue DatabaseError => e
  puts "Database error: #{e.message}"
rescue ApplicationError => e
  puts "Application error: #{e.message}"
end

# retry
def connect_to_database(attempts: 3)
  retries = 0
  begin
    # Simulated connection
    raise "Connection failed" if retries < 2
    puts "Connected successfully!"
  rescue => e
    retries += 1
    if retries < attempts
      puts "Attempt #{retries} failed, retrying..."
      retry
    else
      raise "Could not connect after #{attempts} attempts: #{e.message}"
    end
  end
end

connect_to_database
# Attempt 1 failed, retrying...
# Attempt 2 failed, retrying...
# Connected successfully!

# raise with message vs raise with exception class
raise "Something went wrong"        # raises RuntimeError
raise RuntimeError, "Custom message"
raise ArgumentError.new("Bad arg")

# rescue ใน method body โดยไม่ต้องมี begin
def safe_divide(a, b)
  a / b
rescue ZeroDivisionError
  Float::INFINITY
end
```

---

## คำถาม Ruby ระดับ Senior {#ruby-senior}

### 12. อธิบาย Fiber และ Concurrency ใน Ruby

**เฉลย:**

```ruby
# Fiber คือ lightweight concurrency primitive ใน Ruby
# ต่างจาก Thread ตรงที่ต้องสลับ (yield) ด้วยตัวเอง

# Basic Fiber usage
fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield  # หยุดและส่งคืน control
  puts "Step 2"
  Fiber.yield
  puts "Step 3"
end

fiber.resume  # => "Step 1"
fiber.resume  # => "Step 2"
fiber.resume  # => "Step 3"

# Fiber สำหรับสร้าง infinite sequence
def fibonacci_generator
  Fiber.new do
    a, b = 0, 1
    loop do
      Fiber.yield(a)
      a, b = b, a + b
    end
  end
end

fib = fibonacci_generator
10.times { print "#{fib.resume} " }
# => 0 1 1 2 3 5 8 13 21 34

# Fiber with data passing
fiber = Fiber.new do |first_value|
  received = Fiber.yield(first_value * 2)
  Fiber.yield(received + 10)
end

puts fiber.resume(5)   # => 10 (5 * 2)
puts fiber.resume(3)   # => 13 (3 + 10)

# Threads ใน Ruby (GIL/GVL limitation)
require 'thread'

mutex = Mutex.new
counter = 0

threads = 10.times.map do
  Thread.new do
    1000.times do
      mutex.synchronize { counter += 1 }
    end
  end
end

threads.each(&:join)
puts counter  # => 10000 (ถูกต้องเพราะใช้ mutex)

# Ractor (Ruby 3.0+) - true parallelism
if defined?(Ractor)
  ractor1 = Ractor.new do
    "Hello from Ractor 1"
  end

  ractor2 = Ractor.new do
    "Hello from Ractor 2"
  end

  puts ractor1.take
  puts ractor2.take
end

# Async Ruby ด้วย async gem (ถ้ามี)
# require 'async'
# Async do
#   10.times.map { |i|
#     Async { puts "Task #{i}" }
#   }.each(&:wait)
# end
```

---

### 13. อธิบาย Ruby Object Model อย่างลึก

**เฉลย:**

```ruby
# ทุกอย่างใน Ruby เป็น Object
42.class          # => Integer
42.is_a?(Object)  # => true

# Classes ก็เป็น Objects ของ Class class
String.class      # => Class
Class.class       # => Class
Class.superclass  # => Module
Module.class      # => Class
Module.superclass # => Object
Object.class      # => Class
Object.superclass # => BasicObject

# Singleton classes (eigenclasses)
class Dog
  def bark
    "Woof!"
  end
end

dog1 = Dog.new
dog2 = Dog.new

# เพิ่ม method ให้เฉพาะ dog1
def dog1.special_trick
  "Roll over!"
end

dog1.special_trick  # => "Roll over!"
# dog2.special_trick  # => NoMethodError

# Singleton class ของ dog1 อยู่ใน lookup chain
puts dog1.singleton_class  # => #<Class:#<Dog:...>>
puts dog1.singleton_class.superclass  # => Dog

# Method lookup order (MRO)
class A
  def method_a
    "A"
  end
end

module M1
  def method_m1
    "M1"
  end
end

module M2
  def method_m2
    "M2"
  end
end

class B < A
  include M1
  include M2
end

b = B.new
puts b.class.ancestors
# => [B, M2, M1, A, Object, Kernel, BasicObject]

# Object#send vs public_send
class SecretClass
  private

  def secret_method
    "I'm secret!"
  end
end

obj = SecretClass.new
obj.send(:secret_method)         # => "I'm secret!" (bypass access control)
# obj.public_send(:secret_method) # => NoMethodError

# ObjectSpace - สำรวจ objects ใน memory
require 'objspace'

before = ObjectSpace.count_objects[:T_STRING]
arr = Array.new(1000) { "hello" }
after = ObjectSpace.count_objects[:T_STRING]
puts "New strings created: #{after - before}"

# Frozen objects
str = "hello".freeze
str << " world"  # => FrozenError: can't modify frozen String

# Ruby 3.0+ frozen string literals
# frozen_string_literal: true
```

---

### 14. อธิบาย Memory Management และ Garbage Collection ใน Ruby

**เฉลย:**

```ruby
# Ruby ใช้ mark-and-sweep GC (แบบ tri-color incremental ตั้งแต่ Ruby 2.x)

# GC configuration
puts GC::OPTS  # ดู available options

# GC tuning via environment variables
# RUBY_GC_HEAP_INIT_SLOTS=10000
# RUBY_GC_HEAP_FREE_SLOTS=4096
# RUBY_GC_HEAP_GROWTH_FACTOR=1.8
# RUBY_GC_HEAP_GROWTH_MAX_SLOTS=100000
# RUBY_GC_MALLOC_LIMIT=16MB
# RUBY_GC_OLDMALLOC_LIMIT=16MB

# Monitoring GC
GC.stat.each { |key, value| puts "#{key}: #{value}" }
# heap_allocated_pages, heap_sorted_length, heap_allocatable_pages,
# heap_available_slots, heap_live_slots, heap_free_slots, etc.

# สร้าง objects และดู GC behavior
def allocate_objects(count)
  count.times.map { "hello" * 100 }
end

before_gc = GC.stat[:count]
allocate_objects(100_000)
after_gc = GC.stat[:count]
puts "GC ran #{after_gc - before_gc} times"

# ObjectSpace::WeakMap - weak references
require 'objspace'

weak_map = ObjectSpace::WeakMap.new
key = Object.new
value = "important data"
weak_map[key] = value

puts weak_map[key]  # => "important data"
# เมื่อ key ถูก GC, entry จะหายไปจาก weak_map

# Memory profiling pattern
def measure_memory
  before = `ps -o rss= -p #{Process.pid}`.to_i
  yield
  after = `ps -o rss= -p #{Process.pid}`.to_i
  puts "Memory used: #{after - before} KB"
end

measure_memory do
  large_array = Array.new(1_000_000, "hello")
end

# Avoiding memory leaks
# Pattern 1: ใช้ local variables แทน instance variables เมื่อไม่จำเป็น

# Pattern 2: ระวัง closures ที่ capture variables ขนาดใหญ่
large_data = "x" * 1_000_000
small_proc = proc { "small result" }  # ไม่ capture large_data - ดี
large_proc = proc { large_data.length }  # capture large_data - ระวัง!

# Pattern 3: ใช้ freeze สำหรับ immutable values
CONSTANT = "hello".freeze  # frozen ดีกว่า mutable
```

---

## คำถาม Rails ระดับ Junior {#rails-junior}

### 15. อธิบาย MVC Architecture ใน Rails

**เฉลย:**

```
Request → Router → Controller → Model ↔ Database
                      ↓
                    View → Response
```

```ruby
# Model - จัดการ data และ business logic
# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :tags, through: :article_tags

  validates :title, presence: true, length: { minimum: 5, maximum: 100 }
  validates :body, presence: true

  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc).limit(10) }

  before_save :generate_slug

  def reading_time
    words = body.split.length
    (words / 200.0).ceil  # 200 words per minute
  end

  private

  def generate_slug
    self.slug = title.parameterize
  end
end

# Controller - จัดการ HTTP requests
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  before_action :authorize_article!, only: [:edit, :update, :destroy]

  def index
    @articles = Article.published.recent.includes(:user)
  end

  def show
    @comment = Comment.new
  end

  def new
    @article = Article.new
  end

  def create
    @article = current_user.articles.build(article_params)
    if @article.save
      redirect_to @article, notice: "Article created successfully"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit; end

  def update
    if @article.update(article_params)
      redirect_to @article, notice: "Article updated successfully"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @article.destroy
    redirect_to articles_path, notice: "Article deleted"
  end

  private

  def set_article
    @article = Article.find(params[:id])
  end

  def article_params
    params.require(:article).permit(:title, :body, :published, tag_ids: [])
  end

  def authorize_article!
    redirect_to root_path, alert: "Not authorized" unless @article.user == current_user
  end
end

# View - แสดงผล HTML
# app/views/articles/index.html.erb
# <%= @articles.each do |article| %>
#   <h2><%= link_to article.title, article_path(article) %></h2>
#   <p>By <%= article.user.name %> - <%= article.reading_time %> min read</p>
# <% end %>
```

---

### 16. อธิบาย ActiveRecord Associations

**เฉลย:**

```ruby
# belongs_to - "หนึ่งในหลาย" ฝั่งที่มี foreign key
class Comment < ApplicationRecord
  belongs_to :article   # comments.article_id
  belongs_to :user      # comments.user_id

  # Options:
  belongs_to :author, class_name: "User", foreign_key: :user_id
  belongs_to :parent_comment, class_name: "Comment", optional: true
end

# has_many - "หนึ่งมีหลาย"
class Article < ApplicationRecord
  has_many :comments, dependent: :destroy
  has_many :commenters, through: :comments, source: :user
  has_many :approved_comments, -> { where(approved: true) }, class_name: "Comment"

  # Polymorphic association
  has_many :attachments, as: :attachable
end

# has_one - "หนึ่งมีหนึ่ง"
class User < ApplicationRecord
  has_one :profile, dependent: :destroy
  has_one :subscription

  accepts_nested_attributes_for :profile
end

# has_many :through - ผ่าน join table
class Doctor < ApplicationRecord
  has_many :appointments
  has_many :patients, through: :appointments
end

class Patient < ApplicationRecord
  has_many :appointments
  has_many :doctors, through: :appointments
end

class Appointment < ApplicationRecord
  belongs_to :doctor
  belongs_to :patient
end

# has_and_belongs_to_many (HABTM) - ไม่มี join model
class Article < ApplicationRecord
  has_and_belongs_to_many :categories
end
# ต้องมี articles_categories table

# Polymorphic associations
class Attachment < ApplicationRecord
  belongs_to :attachable, polymorphic: true
end

class Photo < ApplicationRecord
  has_many :attachments, as: :attachable
end

class Document < ApplicationRecord
  has_many :attachments, as: :attachable
end

# attachments table: attachable_id, attachable_type

# Self-referential associations
class Employee < ApplicationRecord
  belongs_to :manager, class_name: "Employee", optional: true
  has_many :subordinates, class_name: "Employee", foreign_key: :manager_id
end

ceo = Employee.create!(name: "CEO")
manager = Employee.create!(name: "Manager", manager: ceo)
employee = Employee.create!(name: "Employee", manager: manager)

puts ceo.subordinates  # => [manager]
puts manager.manager   # => ceo
```

---

### 17. อธิบาย Rails Routing

**เฉลย:**

```ruby
# config/routes.rb

Rails.application.routes.draw do
  # RESTful resources - สร้าง 7 routes โดยอัตโนมัติ
  resources :articles

  # GET    /articles          => articles#index
  # GET    /articles/new      => articles#new
  # POST   /articles          => articles#create
  # GET    /articles/:id      => articles#show
  # GET    /articles/:id/edit => articles#edit
  # PATCH  /articles/:id      => articles#update
  # DELETE /articles/:id      => articles#destroy

  # Nested resources
  resources :articles do
    resources :comments, only: [:create, :destroy]
    member do
      post :publish
      post :unpublish
    end
    collection do
      get :search
      get :trending
    end
  end

  # Limiting routes
  resources :users, only: [:index, :show, :create]
  resources :sessions, only: [:new, :create, :destroy]

  # Singular resource (ไม่มี :id เพราะมีแค่อันเดียว)
  resource :profile  # สร้าง 6 routes (ไม่มี index)
  resource :session  # สร้าง 5 routes

  # Namespace - เพิ่ม prefix ให้ URL และ module
  namespace :admin do
    resources :users
    resources :articles
  end
  # => /admin/users, AdminController::UsersController

  # Scope - เพิ่ม prefix แค่ URL
  scope :api do
    resources :users
  end
  # => /api/users, UsersController

  # Module scope
  scope module: :api do
    resources :users
  end
  # => /users, Api::UsersController

  # Named routes
  get '/about', to: 'pages#about', as: :about
  # => about_path, about_url

  # Root
  root 'home#index'

  # Custom routes
  get '/search', to: 'search#index'
  post '/webhooks/stripe', to: 'webhooks#stripe'

  # Catch-all (ต้องอยู่ท้ายสุด)
  match '*path', to: 'errors#not_found', via: :all

  # Constraints
  resources :articles, constraints: { id: /[A-Z][A-Z][0-9]+/ }

  # Redirect
  get '/old-path', to: redirect('/new-path')
  get '/users/:id', to: redirect('/profiles/%{id}')
end
```

---

## คำถาม Rails ระดับ Mid-Level {#rails-mid}

### 18. อธิบาย ActiveRecord Query Interface

**เฉลย:**

```ruby
# Basic queries
User.all                        # SELECT * FROM users
User.first                      # SELECT * FROM users LIMIT 1
User.last                       # SELECT * FROM users ORDER BY id DESC LIMIT 1
User.find(1)                    # SELECT * FROM users WHERE id = 1
User.find_by(email: "a@b.com")  # LIMIT 1
User.where(active: true)        # WHERE active = TRUE

# Chaining queries
User.where(active: true)
    .where("age > ?", 18)
    .order(name: :asc)
    .limit(10)
    .offset(20)

# Joins
User.joins(:articles)
    .where(articles: { published: true })

# Includes (eager loading - แก้ N+1 problem)
User.includes(:articles, :profile)
    .where(active: true)

# Preload vs Eager Load vs Includes
# preload: แยก query เสมอ
# eager_load: JOIN เสมอ
# includes: อัจฉริยะ (ใช้ JOIN ถ้า filter บ่าย, แยก query ถ้าไม่)

# Group and aggregate
User.group(:city).count
# => {"Bangkok"=>10, "Chiang Mai"=>5}

User.group(:role)
    .select("role, COUNT(*) as count, AVG(age) as avg_age")

Article.group("DATE(created_at)")
       .count

# Having (ใช้กับ group)
User.group(:city)
    .having("COUNT(*) > ?", 5)
    .count

# Pluck - ดึงเฉพาะ columns (เร็วกว่า select ทั้ง model)
User.pluck(:name)          # => ["Alice", "Bob", ...]
User.pluck(:id, :name)     # => [[1, "Alice"], [2, "Bob"], ...]

# Select
User.select(:id, :name)

# Scopes
class Article < ApplicationRecord
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_author, ->(user) { where(user: user) }
  scope :from_last_week, -> { where("created_at > ?", 1.week.ago) }

  # default_scope (ใช้ด้วยความระวัง!)
  default_scope { order(created_at: :desc) }
end

# Scopes สามารถ chain ได้
Article.published.recent.by_author(current_user).limit(5)

# find_or_create_by, find_or_initialize_by
user = User.find_or_create_by(email: "new@example.com") do |u|
  u.name = "New User"
  u.role = "user"
end

# update_all, delete_all (ไม่ผ่าน callbacks!)
User.where(active: false).update_all(role: "inactive")
OldLog.where("created_at < ?", 1.year.ago).delete_all

# destroy_all (ผ่าน callbacks)
User.where(test_account: true).destroy_all

# Transactions
ActiveRecord::Base.transaction do
  account1.update!(balance: account1.balance - 100)
  account2.update!(balance: account2.balance + 100)
end

# Raw SQL
User.find_by_sql("SELECT * FROM users WHERE age > 18")
User.where("name LIKE ?", "%#{query}%")
User.where("created_at BETWEEN ? AND ?", start_date, end_date)
```

---

### 19. อธิบาย Rails Callbacks

**เฉลย:**

```ruby
class Order < ApplicationRecord
  belongs_to :user
  has_many :order_items

  # Before callbacks
  before_validation :normalize_data
  before_create :generate_order_number
  before_save :calculate_total
  before_update :check_cancellable
  before_destroy :cancel_payment

  # After callbacks
  after_create :send_confirmation_email
  after_create_commit :notify_warehouse
  after_update :update_user_stats, if: :status_changed?
  after_destroy :release_inventory

  # Around callbacks
  around_save :log_changes

  # Conditional callbacks
  after_save :notify_admin, if: :high_value_order?
  after_save :send_invoice, unless: :draft?

  # Callback ด้วย method name
  before_save :do_something

  # Callback ด้วย block
  before_save do
    self.name = name.downcase.strip
  end

  private

  def normalize_data
    self.email = email&.downcase&.strip
    self.phone = phone&.gsub(/\D/, '')
  end

  def generate_order_number
    self.order_number = "ORD-#{SecureRandom.hex(6).upcase}"
  end

  def calculate_total
    self.total = order_items.sum { |item| item.price * item.quantity }
  end

  def check_cancellable
    if status_changed? && status == "cancelled" && completed?
      throw(:abort)  # หยุด callback chain
    end
  end

  def send_confirmation_email
    OrderMailer.confirmation(self).deliver_later
  end

  def log_changes
    old_attrs = changes.dup
    yield  # ทำการ save
    Rails.logger.info "Order #{id} changed: #{old_attrs}"
  end

  def high_value_order?
    total > 10_000
  end
end

# Skipping callbacks (ใช้อย่างระวัง!)
order.save(validate: false)
Order.skip_callback(:save, :before, :calculate_total) do
  order.save!
end

# Observer pattern (แยก concern ออกจาก model)
# ใช้ ActiveSupport::Callbacks หรือ gem เช่น wisper
```

---

### 20. อธิบาย Performance: N+1 Queries และวิธีแก้

**เฉลย:**

```ruby
# N+1 Problem - รัน query 1+N ครั้ง
# ปัญหา:
users = User.all
users.each do |user|
  puts user.articles.count  # รัน query 1 ครั้งต่อ user!
end
# SQL:
# SELECT * FROM users
# SELECT COUNT(*) FROM articles WHERE user_id = 1
# SELECT COUNT(*) FROM articles WHERE user_id = 2
# ... N queries

# วิธีแก้ 1: counter_cache
class Article < ApplicationRecord
  belongs_to :user, counter_cache: true  # users.articles_count
end
# จะ auto-update users.articles_count เมื่อ article ถูก create/destroy

class User < ApplicationRecord
  has_many :articles
end
# users.articles_count  # ไม่ต้อง query!

# วิธีแก้ 2: includes (N+1 สำหรับ associations)
# ปัญหา:
posts = Post.all
posts.each { |p| puts p.author.name }  # N+1!

# แก้ด้วย includes
posts = Post.includes(:author).all
posts.each { |p| puts p.author.name }  # 2 queries เท่านั้น

# วิธีแก้ 3: joins กับ select
Post.joins(:author)
    .select("posts.*, users.name as author_name")
    .each { |p| puts p.author_name }

# วิธีแก้ 4: eager_load สำหรับ filtering
Post.eager_load(:author)
    .where(users: { active: true })

# Bullet gem - ตรวจหา N+1 อัตโนมัติ
# config/environments/development.rb
# config.after_initialize do
#   Bullet.enable = true
#   Bullet.alert = true
#   Bullet.rails_logger = true
# end

# ตรวจสอบ queries ด้วย to_sql
Post.includes(:author).where(published: true).to_sql
# => "SELECT posts.*, users.* FROM posts LEFT OUTER JOIN users ..."

# ตรวจสอบ query count ใน tests
expect {
  get :index
}.to make_database_queries(count: 2)  # ด้วย db-query-matchers gem

# Lazy loading vs Eager loading
# Lazy (default): query เมื่อถูกใช้งาน
articles = Article.all
# ยังไม่ query...
articles.each { |a| puts a.title }  # query ตอนนี้

# Eager: query ทันที
articles = Article.all.load
# query ทันที

# บ่อยครั้ง cache ช่วยได้
Rails.cache.fetch("users/#{user.id}/article_count", expires_in: 1.hour) do
  user.articles.count
end
```

---

## คำถาม Rails ระดับ Senior {#rails-senior}

### 21. อธิบาย Rails Security Best Practices

**เฉลย:**

```ruby
# 1. Mass Assignment Protection
# Strong Parameters ใน Controller
def user_params
  params.require(:user).permit(:name, :email, :password)
  # NEVER permit(:admin) โดยไม่จำเป็น
end

# 2. SQL Injection Prevention
# ผิด - vulnerable!
User.where("name = '#{params[:name]}'")

# ถูก - parameterized queries
User.where("name = ?", params[:name])
User.where(name: params[:name])

# 3. XSS Prevention
# Rails auto-escapes HTML in views
<%= user.name %>        # escaped ✓
<%= raw user.content %> # NOT escaped - อันตราย!
<%= user.content.html_safe %> # NOT escaped - อันตราย!

# ใช้ sanitize helper
<%= sanitize user.content, tags: %w[p br b i], attributes: %w[href] %>

# 4. CSRF Protection
# ApplicationController มี protect_from_forgery โดย default
# ใช้ authenticity_token ใน forms
# Rails form helpers include this automatically

# 5. Authentication
# ใช้ Devise หรือ implement เอง
class User < ApplicationRecord
  has_secure_password  # ต้องการ bcrypt gem

  # ป้องกัน timing attacks
  def self.find_and_authenticate(email, password)
    user = find_by(email: email)
    # BCrypt.secure_compare prevents timing attacks
    return nil unless user&.authenticate(password)
    user
  end
end

# 6. Authorization
# ใช้ Pundit หรือ CanCanCan
class ArticlePolicy < ApplicationPolicy
  def update?
    user.admin? || record.user == user
  end

  def destroy?
    user.admin?
  end

  class Scope < Scope
    def resolve
      if user.admin?
        scope.all
      else
        scope.where(user: user).or(scope.published)
      end
    end
  end
end

# 7. Secure Headers
# config/initializers/secure_headers.rb
# SecureHeaders::Configuration.default do |config|
#   config.x_frame_options = "DENY"
#   config.x_content_type_options = "nosniff"
#   config.x_xss_protection = "1; mode=block"
#   config.csp = {
#     default_src: %w('self'),
#     script_src: %w('self'),
#   }
# end

# 8. Sensitive Data
# ใช้ Rails credentials
secret_key = Rails.application.credentials.stripe[:secret_key]

# ห้ามเก็บ passwords/tokens ใน logs
# config/application.rb
config.filter_parameters += [:password, :token, :secret, :credit_card]

# 9. File Upload Security
def create_attachment
  file = params[:file]
  # ตรวจสอบ type
  allowed_types = %w[image/jpeg image/png application/pdf]
  raise "Invalid file type" unless allowed_types.include?(file.content_type)

  # จำกัดขนาด
  raise "File too large" if file.size > 5.megabytes

  # ใช้ safe filename
  safe_filename = File.basename(file.original_filename).gsub(/[^0-9A-Za-z.\-]/, '_')

  Attachment.create!(file: file, filename: safe_filename)
end

# 10. Rate Limiting
# ด้วย Rack::Attack
class Rack::Attack
  throttle("req/ip", limit: 300, period: 5.minutes) do |req|
    req.ip
  end

  throttle("logins/ip", limit: 5, period: 20.seconds) do |req|
    req.ip if req.path == "/login" && req.post?
  end
end
```

---

### 22. อธิบาย Caching Strategies ใน Rails

**เฉลย:**

```ruby
# Rails มี caching layers หลายชั้น

# 1. Fragment Caching (HTML fragment)
# app/views/articles/index.html.erb
# <% cache @articles do %>
#   <% @articles.each do |article| %>
#     <% cache article do %>
#       <%= render article %>
#     <% end %>
#   <% end %>
# <% end %>

# 2. Russian Doll Caching - nested caches
# Cache key โดยอัตโนมัติจาก updated_at
class Article < ApplicationRecord
  belongs_to :user, touch: true  # update user's updated_at when article changes
end

# 3. Low-Level Caching
class ProductsController < ApplicationController
  def expensive_operation
    @result = Rails.cache.fetch("expensive_result/#{params[:id]}", expires_in: 1.hour) do
      # คำนวณที่ใช้เวลานาน
      Product.complex_calculation(params[:id])
    end
  end
end

# 4. Counter Cache
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end
# users.posts_count จะ update อัตโนมัติ

# 5. HTTP Caching
class ArticlesController < ApplicationController
  def show
    @article = Article.find(params[:id])

    # Last-Modified header
    fresh_when(last_modified: @article.updated_at, etag: @article)

    # หรือ
    if stale?(@article)
      # จะ render เฉพาะเมื่อ content เปลี่ยน
      respond_to do |format|
        format.html
        format.json { render json: @article }
      end
    end
  end
end

# 6. Cache Stores
# Memory Store (development)
config.cache_store = :memory_store, { size: 64.megabytes }

# Redis Store (production)
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  expires_in: 1.day,
  namespace: "myapp_cache"
}

# Memcached
config.cache_store = :mem_cache_store, "cache-1.example.com", "cache-2.example.com"

# 7. Cache Invalidation
# Time-based
Rails.cache.write("key", value, expires_in: 1.hour)

# Manual invalidation
Rails.cache.delete("user/#{user.id}/profile")
Rails.cache.delete_matched("user/#{user.id}/*")

# Using cache_key
class User < ApplicationRecord
  def full_cache_key
    "user/#{id}/#{updated_at.to_i}"
  end
end

# 8. Sidekiq + Cache warming
class CacheWarmingJob < ApplicationJob
  def perform
    User.find_each do |user|
      Rails.cache.fetch("user/#{user.id}/stats", expires_in: 1.hour) do
        user.calculate_stats
      end
    end
  end
end
```

---

## System Design Questions {#system-design}

### 23. ออกแบบระบบ URL Shortener

**เฉลย:**

```ruby
# Requirements:
# - รับ URL ยาว -> คืน short URL
# - Redirect short URL ไป original
# - Track click stats
# - High availability, low latency

# Database Schema
# urls: id, original_url, short_code, user_id, created_at
# url_clicks: id, url_id, ip_address, user_agent, clicked_at

# Model
class Url < ApplicationRecord
  belongs_to :user, optional: true
  has_many :url_clicks

  validates :original_url, presence: true, format: URI::regexp(%w[http https])
  validates :short_code, presence: true, uniqueness: true

  before_create :generate_short_code

  def click_count
    Rails.cache.fetch("url/#{id}/click_count", expires_in: 5.minutes) do
      url_clicks.count
    end
  end

  private

  def generate_short_code
    loop do
      self.short_code = Base62.encode(SecureRandom.random_number(62**6))
      break unless Url.exists?(short_code: short_code)
    end
  end
end

# Base62 encoding
module Base62
  CHARS = ('0'..'9').to_a + ('a'..'z').to_a + ('A'..'Z').to_a

  def self.encode(num)
    result = ""
    while num > 0
      result = CHARS[num % 62] + result
      num /= 62
    end
    result.rjust(6, '0')
  end
end

# Controller
class UrlsController < ApplicationController
  def create
    @url = Url.find_or_create_by(original_url: url_params[:original_url]) do |u|
      u.user = current_user
    end

    if @url.persisted?
      render json: { short_url: short_url(@url.short_code) }
    else
      render json: { errors: @url.errors }, status: :unprocessable_entity
    end
  end
end

class RedirectsController < ApplicationController
  def show
    url = Url.find_by!(short_code: params[:code])

    # Track click asynchronously
    TrackClickJob.perform_later(url.id, request.ip, request.user_agent)

    redirect_to url.original_url, status: :moved_permanently
  rescue ActiveRecord::RecordNotFound
    render plain: "URL not found", status: :not_found
  end
end

# Background job for tracking
class TrackClickJob < ApplicationJob
  queue_as :low_priority

  def perform(url_id, ip_address, user_agent)
    UrlClick.create!(
      url_id: url_id,
      ip_address: ip_address,
      user_agent: user_agent
    )

    # Invalidate cache
    Rails.cache.delete("url/#{url_id}/click_count")
  end
end

# Caching ด้วย Redis
# config/initializers/redis.rb
REDIS = Redis.new(url: ENV["REDIS_URL"])

class RedirectsController < ApplicationController
  def show
    # ตรวจ Redis ก่อน (เร็วกว่า DB)
    original_url = REDIS.get("url:#{params[:code]}")

    unless original_url
      url = Url.find_by!(short_code: params[:code])
      original_url = url.original_url
      REDIS.setex("url:#{params[:code]}", 3600, original_url)
    end

    TrackClickJob.perform_later(params[:code], request.ip, request.user_agent)
    redirect_to original_url, status: :moved_permanently
  end
end
```

---

### 24. ออกแบบระบบ Chat Application

**เฉลย:**

```ruby
# Stack: Rails + ActionCable + Redis + Sidekiq

# Models
class Conversation < ApplicationRecord
  has_many :messages, dependent: :destroy
  has_many :conversation_participants, dependent: :destroy
  has_many :users, through: :conversation_participants

  def self.find_or_create_direct(user1, user2)
    conversation = joins(:conversation_participants)
      .where(conversation_participants: { user: user1 })
      .joins(:conversation_participants)
      .where(conversation_participants: { user: user2 })
      .where(direct: true)
      .first

    conversation || create_direct_conversation(user1, user2)
  end

  private

  def self.create_direct_conversation(user1, user2)
    transaction do
      conversation = create!(direct: true)
      conversation.conversation_participants.create!(user: user1)
      conversation.conversation_participants.create!(user: user2)
      conversation
    end
  end
end

class Message < ApplicationRecord
  belongs_to :conversation
  belongs_to :user
  has_many :message_reads, dependent: :destroy

  validates :body, presence: true, length: { maximum: 5000 }

  after_create_commit :broadcast_message
  after_create_commit :update_conversation_timestamp

  def read_by?(user)
    message_reads.exists?(user: user)
  end

  private

  def broadcast_message
    ActionCable.server.broadcast(
      "conversation_#{conversation_id}",
      {
        type: "new_message",
        message: MessageSerializer.new(self).as_json
      }
    )
  end

  def update_conversation_timestamp
    conversation.touch
  end
end

# ActionCable Channel
class ConversationChannel < ApplicationCable::Channel
  def subscribed
    conversation = find_conversation(params[:conversation_id])
    stream_from "conversation_#{conversation.id}"
    mark_messages_read(conversation)
  end

  def unsubscribed
    # Cleanup
  end

  def send_message(data)
    conversation = find_conversation(data["conversation_id"])
    message = conversation.messages.create!(
      user: current_user,
      body: data["body"]
    )

    # Broadcast typing stopped
    ActionCable.server.broadcast(
      "conversation_#{conversation.id}",
      { type: "typing_stopped", user_id: current_user.id }
    )
  end

  def typing(data)
    ActionCable.server.broadcast(
      "conversation_#{data['conversation_id']}",
      { type: "typing", user_id: current_user.id }
    )
  end

  private

  def find_conversation(id)
    current_user.conversations.find(id)
  end

  def mark_messages_read(conversation)
    unread = conversation.messages
      .where.not(user: current_user)
      .where(id: MessageRead.where(user: current_user).select(:message_id))

    MessageRead.insert_all(
      unread.map { |m| { message_id: m.id, user_id: current_user.id } }
    )
  end
end

# Connection authentication
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      if (token = request.params[:token])
        user = User.find_by(auth_token: token)
        return user if user

        reject_unauthorized_connection
      elsif (user = env["warden"]&.user)
        user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

---

## Coding Challenges พร้อมเฉลย {#coding-challenges}

### 25. Two Sum Problem

**โจทย์:** หา indices สองตัวใน array ที่บวกกันได้ target

```ruby
# วิธีที่ 1: Brute Force O(n²)
def two_sum_brute(nums, target)
  nums.each_with_index do |num1, i|
    nums.each_with_index do |num2, j|
      next if i == j
      return [i, j] if num1 + num2 == target
    end
  end
  nil
end

# วิธีที่ 2: Hash Map O(n) - ดีที่สุด
def two_sum(nums, target)
  seen = {}  # value => index

  nums.each_with_index do |num, i|
    complement = target - num
    if seen.key?(complement)
      return [seen[complement], i]
    end
    seen[num] = i
  end

  nil
end

# Test cases
puts two_sum([2, 7, 11, 15], 9).inspect   # => [0, 1]
puts two_sum([3, 2, 4], 6).inspect         # => [1, 2]
puts two_sum([3, 3], 6).inspect            # => [0, 1]
```

---

### 26. FizzBuzz ขั้นสูง

**โจทย์:** FizzBuzz แบบ extensible

```ruby
# วิธีที่ 1: Simple
def fizz_buzz(n)
  (1..n).map do |i|
    if i % 15 == 0 then "FizzBuzz"
    elsif i % 3 == 0 then "Fizz"
    elsif i % 5 == 0 then "Buzz"
    else i.to_s
    end
  end
end

# วิธีที่ 2: Extensible/Open for extension
class FizzBuzz
  def initialize
    @rules = {}
  end

  def add_rule(divisor, word)
    @rules[divisor] = word
    self
  end

  def generate(n)
    (1..n).map { |i| apply_rules(i) }
  end

  private

  def apply_rules(n)
    result = @rules
      .sort_by { |divisor, _| divisor }
      .select { |divisor, _| n % divisor == 0 }
      .map { |_, word| word }
      .join

    result.empty? ? n.to_s : result
  end
end

# การใช้งาน
fb = FizzBuzz.new
  .add_rule(3, "Fizz")
  .add_rule(5, "Buzz")
  .add_rule(7, "Bazz")

fb.generate(21)
# => ["1", "2", "Fizz", "4", "Buzz", "Fizz", "Bazz", "8", "Fizz", "Buzz",
#     "11", "Fizz", "13", "Bazz", "FizzBuzz", "16", "17", "Fizz", "19", "Buzz", "FizzBazz"]
```

---

### 27. Implement LRU Cache

**โจทย์:** สร้าง LRU (Least Recently Used) Cache

```ruby
class LRUCache
  def initialize(capacity)
    @capacity = capacity
    @cache = {}
    @order = []  # tracks usage order (oldest first)
  end

  def get(key)
    return -1 unless @cache.key?(key)

    # Update usage order
    @order.delete(key)
    @order.push(key)

    @cache[key]
  end

  def put(key, value)
    if @cache.key?(key)
      @order.delete(key)
    elsif @cache.size >= @capacity
      # Evict least recently used
      lru_key = @order.shift
      @cache.delete(lru_key)
    end

    @cache[key] = value
    @order.push(key)
  end

  def to_s
    @order.map { |k| "#{k}:#{@cache[k]}" }.join(" -> ")
  end
end

# วิธีที่ดีกว่า: ใช้ Hash (Ruby 1.9+ preserves insertion order)
class LRUCacheOptimized
  def initialize(capacity)
    @capacity = capacity
    @cache = {}
  end

  def get(key)
    return -1 unless @cache.key?(key)

    value = @cache.delete(key)
    @cache[key] = value  # re-insert ที่ท้าย (newest)
    value
  end

  def put(key, value)
    @cache.delete(key) if @cache.key?(key)
    @cache.shift if @cache.size >= @capacity  # ลบ oldest (อยู่หัว)
    @cache[key] = value
  end
end

# Test
cache = LRUCacheOptimized.new(3)
cache.put(1, 1)
cache.put(2, 2)
cache.put(3, 3)
puts cache.get(1)  # => 1 (1 กลายเป็น newest)
cache.put(4, 4)    # evict 2 (oldest)
puts cache.get(2)  # => -1 (2 ถูก evict)
puts cache.get(3)  # => 3
puts cache.get(4)  # => 4
```

---

### 28. Flatten Nested Array

**โจทย์:** Flatten nested array โดยไม่ใช้ built-in flatten

```ruby
def flatten_array(arr, depth = Float::INFINITY)
  result = []

  arr.each do |element|
    if element.is_a?(Array) && depth > 0
      result.concat(flatten_array(element, depth - 1))
    else
      result << element
    end
  end

  result
end

# Tests
puts flatten_array([1, [2, [3, [4, 5]]]]).inspect
# => [1, 2, 3, 4, 5]

puts flatten_array([1, [2, [3, [4, 5]]]], 1).inspect
# => [1, 2, [3, [4, 5]]]

puts flatten_array([1, [2, [3, [4, 5]]]], 2).inspect
# => [1, 2, 3, [4, 5]]

# Iterative version
def flatten_iterative(arr)
  result = []
  stack = arr.dup

  until stack.empty?
    item = stack.shift
    if item.is_a?(Array)
      stack.unshift(*item)
    else
      result << item
    end
  end

  result
end
```

---

### 29. Binary Search Tree

**โจทย์:** Implement BST พร้อม insert, search, traverse

```ruby
class Node
  attr_accessor :value, :left, :right

  def initialize(value)
    @value = value
    @left = nil
    @right = nil
  end
end

class BinarySearchTree
  def initialize
    @root = nil
  end

  def insert(value)
    @root = insert_node(@root, value)
    self
  end

  def search(value)
    search_node(@root, value)
  end

  def include?(value)
    !search(value).nil?
  end

  # In-order traversal (sorted order)
  def in_order
    result = []
    in_order_traverse(@root, result)
    result
  end

  # Pre-order traversal
  def pre_order
    result = []
    pre_order_traverse(@root, result)
    result
  end

  # Post-order traversal
  def post_order
    result = []
    post_order_traverse(@root, result)
    result
  end

  def height
    calculate_height(@root)
  end

  def min_value
    return nil if @root.nil?
    node = @root
    node = node.left while node.left
    node.value
  end

  def max_value
    return nil if @root.nil?
    node = @root
    node = node.right while node.right
    node.value
  end

  private

  def insert_node(node, value)
    return Node.new(value) if node.nil?

    if value < node.value
      node.left = insert_node(node.left, value)
    elsif value > node.value
      node.right = insert_node(node.right, value)
    end
    # value == node.value: ไม่ insert ซ้ำ

    node
  end

  def search_node(node, value)
    return nil if node.nil?
    return node if node.value == value

    if value < node.value
      search_node(node.left, value)
    else
      search_node(node.right, value)
    end
  end

  def in_order_traverse(node, result)
    return if node.nil?
    in_order_traverse(node.left, result)
    result << node.value
    in_order_traverse(node.right, result)
  end

  def pre_order_traverse(node, result)
    return if node.nil?
    result << node.value
    pre_order_traverse(node.left, result)
    pre_order_traverse(node.right, result)
  end

  def post_order_traverse(node, result)
    return if node.nil?
    post_order_traverse(node.left, result)
    post_order_traverse(node.right, result)
    result << node.value
  end

  def calculate_height(node)
    return -1 if node.nil?
    [calculate_height(node.left), calculate_height(node.right)].max + 1
  end
end

# Test
bst = BinarySearchTree.new
[5, 3, 7, 1, 4, 6, 8, 2].each { |n| bst.insert(n) }

puts bst.in_order.inspect   # => [1, 2, 3, 4, 5, 6, 7, 8]
puts bst.pre_order.inspect  # => [5, 3, 1, 2, 4, 7, 6, 8]
puts bst.include?(4)        # => true
puts bst.include?(9)        # => false
puts bst.min_value          # => 1
puts bst.max_value          # => 8
puts bst.height             # => 3
```

---

### 30. Anagram Detection

**โจทย์:** ตรวจสอบว่า strings สองตัวเป็น anagram กันหรือไม่

```ruby
# วิธีที่ 1: Sort and compare O(n log n)
def anagram_sort?(str1, str2)
  normalize(str1).chars.sort == normalize(str2).chars.sort
end

def normalize(str)
  str.downcase.gsub(/[^a-z]/, '')
end

# วิธีที่ 2: Character frequency O(n)
def anagram?(str1, str2)
  s1 = normalize(str1)
  s2 = normalize(str2)

  return false if s1.length != s2.length

  freq = Hash.new(0)
  s1.each_char { |c| freq[c] += 1 }
  s2.each_char { |c| freq[c] -= 1 }

  freq.values.all?(&:zero?)
end

# วิธีที่ 3: Group anagrams
def group_anagrams(words)
  words.group_by { |w| w.downcase.chars.sort.join }
       .values
end

# Tests
puts anagram?("listen", "silent")    # => true
puts anagram?("hello", "world")      # => false
puts anagram?("Astronomer", "Moon starer")  # => true (ignore spaces)

groups = group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"])
groups.each { |g| puts g.inspect }
# => ["eat", "tea", "ate"]
# => ["tan", "nat"]
# => ["bat"]
```

---

## Behavioral Questions {#behavioral}

### 31. คำถาม Behavioral สำหรับ Senior Rails Developer

**คำถาม:** "บอกถึงครั้งที่คุณต้องตัดสินใจเรื่อง trade-off ระหว่าง technical debt และ delivery speed"

**Framework การตอบ (STAR Method):**

**Situation:** โปรเจกต์ e-commerce ต้องการ launch ใน 2 สัปดาห์
**Task:** ต้องสร้าง checkout flow ที่ซับซ้อน
**Action:** ตัดสินใจ implement แบบ quick-and-dirty ก่อน แต่ document technical debt ทุกจุด
**Result:** Launch ได้ตามกำหนด และมี refactoring plan ที่ชัดเจน

**ตัวอย่างการตอบ:**

```
"ในโปรเจกต์ที่ผ่านมา เราต้องการ launch feature ใหม่ภายใน 2 สัปดาห์
เพื่อตอบสนองความต้องการของลูกค้าสำคัญ

ผมตัดสินใจทำ code review ร่วมกับทีม แล้วระบุส่วนที่:
1. ต้องทำให้ถูกต้องตั้งแต่แรก (security, data integrity)
2. สามารถ simplify ได้ชั่วคราว แล้วค่อย refactor ทีหลัง

เราใช้ TODO comments ที่มีชื่อและ issue tracker
เพื่อติดตาม technical debt ทุกชิ้น

ผลคือ launch ได้ตามกำหนด และใน sprint ถัดมา
เรา refactor ส่วนที่ค้างอยู่ได้ 80% ภายใน 3 sprints"
```

---

### 32. Code Review คำถาม

**คำถาม:** "คุณพบ code นี้ใน PR และมีความคิดเห็นอย่างไร?"

```ruby
# Code ที่พบใน PR
def get_user_data(user_id)
  user = User.find(user_id)
  articles = Article.where("user_id = #{user_id}")  # SQL injection!
  comments = Comment.where(user_id: user_id)

  {
    user: user,
    articles: articles,
    comments: comments
  }
end
```

**เฉลย - ปัญหาที่พบ:**

```ruby
# ปัญหา 1: SQL Injection vulnerability
articles = Article.where("user_id = #{user_id}")
# แก้เป็น:
articles = Article.where(user_id: user_id)

# ปัญหา 2: N+1 potential - ถ้าเรียกใช้หลายครั้ง
# แก้ด้วย eager loading

# ปัญหา 3: ไม่จัดการ exception เมื่อ user ไม่พบ
user = User.find(user_id)  # raise ActiveRecord::RecordNotFound

# Version ที่ดีกว่า:
def get_user_data(user_id)
  user = User.includes(:articles, :comments).find(user_id)

  {
    user: user.as_json(only: [:id, :name, :email]),
    articles: user.articles.as_json(only: [:id, :title, :created_at]),
    comments: user.comments.as_json(only: [:id, :body, :created_at])
  }
rescue ActiveRecord::RecordNotFound
  nil
end
```

---

### 33. Architecture Discussion Questions

**คำถาม:** "คุณจะออกแบบระบบ Background Job Processing อย่างไร?"

**เฉลย:**

```ruby
# ระบบที่ดีต้องประกอบด้วย:

# 1. Job definitions ที่ชัดเจน
class SendEmailJob < ApplicationJob
  queue_as :emails
  retry_on StandardError, attempts: 3, wait: :exponentially_longer
  discard_on ActiveRecord::RecordNotFound

  def perform(user_id, email_type)
    user = User.find(user_id)
    UserMailer.send(email_type, user).deliver_now
  end
end

# 2. Priority queues
# config/sidekiq.yml
# :queues:
#   - [critical, 10]
#   - [default, 5]
#   - [low, 1]

# 3. Idempotency - ทำซ้ำได้โดยไม่เกิดผลข้างเคียง
class ProcessPaymentJob < ApplicationJob
  def perform(payment_id)
    payment = Payment.find(payment_id)

    # ตรวจสอบว่าเคยทำแล้วหรือยัง
    return if payment.processed?

    ActiveRecord::Base.transaction do
      payment.process!
      payment.update!(processed_at: Time.current)
    end
  end
end

# 4. Dead Letter Queue - จัดการ jobs ที่ fail
class DeadLetterReporter
  def self.check_and_alert
    dead_jobs = Sidekiq::DeadSet.new
    if dead_jobs.size > 100
      AdminMailer.dead_jobs_alert(dead_jobs.size).deliver_later
    end
  end
end

# 5. Monitoring
# Sidekiq Web UI mounted ใน routes
require 'sidekiq/web'
mount Sidekiq::Web => '/admin/sidekiq'
```

---

## สรุป Tips สำหรับการสัมภาษณ์

### เทคนิคการตอบคำถาม Technical

1. **อธิบาย concept ก่อน** - อย่าเริ่ม code ทันที
2. **พูดถึง trade-offs** - แสดงว่าเข้าใจว่าไม่มีทางเลือกที่ดีที่สุดเสมอ
3. **ถามคำถาม** - เพื่อทำความเข้าใจ requirements
4. **เริ่มจาก brute force** - แล้วค่อย optimize
5. **Test cases ก่อน code** - แสดง TDD thinking

### คำถามที่ควรถามผู้สัมภาษณ์

1. "Team size เท่าไหร่ และ code review process เป็นอย่างไร?"
2. "Tech stack ปัจจุบันมีอะไรบ้าง และมีแผนจะเปลี่ยนอะไรไหม?"
3. "On-call responsibility เป็นอย่างไร?"
4. "Engineering culture ให้ความสำคัญกับอะไร?"
5. "Definition of Done สำหรับ feature คืออะไร?"

### Red Flags ที่ควรระวัง

- ไม่มี code review process
- ไม่มี tests
- Deployment manual ทั้งหมด
- Technical debt สูงมากและไม่มีแผนจัดการ
- Team turnover สูง

---

*อัพเดทล่าสุด: 2024 | Ruby 3.3 | Rails 7.1*
