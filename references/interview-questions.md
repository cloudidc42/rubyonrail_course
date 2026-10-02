# Ruby & Rails Interview Questions

## คู่มือเตรียมสัมภาษณ์งาน Ruby on Rails

ครอบคลุมระดับ Junior, Mid-level, Senior

---

# ส่วนที่ 1: Ruby Interview Questions (50 ข้อ)

---

## Junior Level (ข้อ 1-20)

### ข้อ 1: Ruby คืออะไร และมีจุดเด่นอะไรบ้าง?

**คำตอบ:**
Ruby เป็นภาษา scripting แบบ interpreted, dynamic, object-oriented ที่สร้างโดย Yukihiro "Matz" Matsumoto ในปี 1995

จุดเด่น:
- **Everything is an Object** - ทุกอย่างเป็น object รวมถึง primitives
- **Dynamic typing** - ไม่ต้องประกาศ type
- **Blocks, Procs, Lambdas** - First-class functions
- **Duck typing** - ดูพฤติกรรม ไม่ใช่ type
- **Open classes** - สามารถ extend class ที่มีอยู่แล้วได้
- **Convention over Configuration** - หลักการที่ Rails ยืม

```ruby
# ทุกอย่างเป็น object
1.class        # => Integer
true.class     # => TrueClass
nil.class      # => NilClass
"hello".class  # => String

# Integers มี methods
5.times { print "Hello " }
-3.abs  # => 3
```

---

### ข้อ 2: อธิบาย Symbol ใน Ruby และความแตกต่างจาก String

**คำตอบ:**
- **Symbol** (`:name`) - immutable, stored once ใน memory, ใช้เป็น identifier
- **String** (`"name"`) - mutable, สร้าง object ใหม่ทุกครั้ง

```ruby
# Symbol - stored ครั้งเดียวใน memory
:hello.object_id == :hello.object_id  # => true

# String - สร้าง object ใหม่ทุกครั้ง
"hello".object_id == "hello".object_id  # => false

# เมื่อใช้ Symbol
user = { name: "Alice", age: 30 }  # keys เป็น symbols
user[:name]  # => "Alice"

# แปลงระหว่างกัน
"hello".to_sym  # => :hello
:hello.to_s     # => "hello"

# ใช้ Symbol เมื่อ: ชื่อ method, hash keys, identifiers
# ใช้ String เมื่อ: text ที่เปลี่ยนแปลงได้, user input
```

---

### ข้อ 3: อธิบาย nil, false, และ truthy/falsy ใน Ruby

**คำตอบ:**
ใน Ruby มีค่าที่ falsy เพียง 2 ค่า: `nil` และ `false`
ทุกค่าอื่น รวมถึง `0` และ `""` เป็น truthy

```ruby
# Falsy ใน Ruby
if nil;   puts "nil is falsy"   end  # prints
if false; puts "false is falsy" end  # prints

# Truthy ทุกอย่างอื่น
if 0;    puts "0 is truthy"    end  # prints!
if "";   puts "empty is truthy" end  # prints!
if [];   puts "[] is truthy"   end  # prints!

# nil vs false
nil.nil?    # => true
false.nil?  # => false
nil == false # => false

# Safe navigation operator
user = nil
user&.name  # => nil (ไม่ raise NoMethodError)
```

---

### ข้อ 4: อธิบาย Array methods ที่สำคัญ

**คำตอบ:**

```ruby
arr = [3, 1, 4, 1, 5, 9, 2, 6]

# Transformation
arr.map { |n| n * 2 }     # => [6, 2, 8, 2, 10, 18, 4, 12]
arr.select { |n| n > 3 }  # => [4, 5, 9, 6]
arr.reject { |n| n > 3 }  # => [3, 1, 1, 2]
arr.reduce(:+)             # => 31

# Sorting
arr.sort                   # => [1, 1, 2, 3, 4, 5, 6, 9]
arr.sort_by { |n| -n }     # => [9, 6, 5, 4, 3, 2, 1, 1]

# Searching
arr.find { |n| n > 4 }     # => 5 (first match)
arr.any? { |n| n > 8 }     # => true
arr.all? { |n| n > 0 }     # => true
arr.none? { |n| n > 10 }   # => true
arr.count { |n| n > 3 }    # => 4
arr.min                    # => 1
arr.max                    # => 9

# Manipulation
arr.flatten                # สำหรับ nested arrays
arr.compact                # ลบ nil elements
arr.uniq                   # ลบ duplicates => [3, 1, 4, 5, 9, 2, 6]
arr.zip([1,2,3])           # => [[3,1],[1,2],[4,3],...]

# each_with_object
arr.each_with_object({}) do |n, hash|
  hash[n] = n ** 2
end
# => {3=>9, 1=>1, 4=>16, 5=>25, 9=>81, 2=>4, 6=>36}
```

---

### ข้อ 5: อธิบาย Hash ใน Ruby

**คำตอบ:**

```ruby
# สร้าง Hash
h = { name: "Alice", age: 30, city: "Bangkok" }

# Access
h[:name]              # => "Alice"
h.fetch(:name)        # => "Alice"
h.fetch(:email, nil)  # => nil (default ถ้าไม่เจอ)

# Manipulation
h.merge({ email: "alice@example.com" })  # สร้าง hash ใหม่
h.merge!({ email: "alice@example.com" }) # แก้ไข in-place

# Iteration
h.each { |key, value| puts "#{key}: #{value}" }
h.map { |key, value| [key, value.to_s] }.to_h
h.select { |_, v| v.is_a?(String) }
h.reject { |k, _| k == :age }

# Transform
h.transform_values { |v| v.to_s }
h.transform_keys    { |k| k.to_s }

# Useful methods
h.keys     # => [:name, :age, :city]
h.values   # => ["Alice", 30, "Bangkok"]
h.to_a     # => [[:name, "Alice"], [:age, 30], ...]
h.size     # => 3
h.empty?   # => false
h.key?(:name)   # => true
h.value?("Alice") # => true

# Default value
counts = Hash.new(0)
"hello world".chars.each { |c| counts[c] += 1 }
# => {"h"=>1, "e"=>1, "l"=>3, "o"=>2, " "=>1, "w"=>1, "r"=>1, "d"=>1}
```

---

### ข้อ 6: อธิบาย Blocks, Procs, และ Lambdas

**คำตอบ:**

```ruby
# Block - ไม่ใช่ object, ส่ง inline
[1, 2, 3].each { |n| puts n }
[1, 2, 3].each do |n|
  puts n
end

# yield - เรียก block ที่ส่งมา
def greet
  puts "Before"
  yield("Alice") if block_given?
  puts "After"
end

greet { |name| puts "Hello, #{name}!" }
# Before
# Hello, Alice!
# After

# Proc - Block ที่เก็บใน variable
doubler = Proc.new { |n| n * 2 }
# หรือ
doubler = proc { |n| n * 2 }

doubler.call(5)  # => 10
doubler.(5)      # => 10 (shorthand)
doubler[5]       # => 10 (shorthand)

[1, 2, 3].map(&doubler)  # => [2, 4, 6]

# Lambda - Proc ที่เข้มงวดกว่า
multiplier = lambda { |n, factor| n * factor }
# หรือ (stabby lambda)
multiplier = ->(n, factor) { n * factor }

multiplier.call(5, 3)  # => 15

# ความแตกต่าง Proc vs Lambda
# 1. Argument checking
p = Proc.new { |a, b| [a, b] }
l = lambda  { |a, b| [a, b] }

p.call(1)        # => [1, nil] (ไม่ error)
# l.call(1)      # => ArgumentError!

# 2. Return behavior
def test_proc
  p = Proc.new { return "from proc" }
  p.call
  "from method"  # ไม่ถึงบรรทัดนี้
end

def test_lambda
  l = lambda { return "from lambda" }
  l.call
  "from method"  # ถึงบรรทัดนี้
end

test_proc   # => "from proc"
test_lambda # => "from method"
```

---

### ข้อ 7: อธิบาย Inheritance และ Modules

**คำตอบ:**

```ruby
# Inheritance - is-a relationship
class Animal
  attr_reader :name

  def initialize(name)
    @name = name
  end

  def speak
    raise NotImplementedError, "Subclass must implement speak"
  end

  def to_s
    "#{self.class.name}: #{name}"
  end
end

class Dog < Animal
  def speak
    "#{name} says: Woof!"
  end
end

class Cat < Animal
  def speak
    "#{name} says: Meow!"
  end
end

# super - เรียก method จาก parent
class GuideDog < Dog
  def initialize(name, owner)
    super(name)  # เรียก Dog#initialize
    @owner = owner
  end
end

# Modules - Mixins (has-a behavior)
module Swimmable
  def swim
    "#{name} is swimming!"
  end
end

module Flyable
  def fly
    "#{name} is flying!"
  end
end

class Duck < Animal
  include Swimmable
  include Flyable

  def speak
    "Quack!"
  end
end

donald = Duck.new("Donald")
donald.swim   # => "Donald is swimming!"
donald.fly    # => "Donald is flying!"

# Mixins vs Inheritance
# - Ruby ไม่มี multiple inheritance
# - ใช้ Modules เมื่อต้องการ share behavior
# - ใช้ Inheritance เมื่อมีความสัมพันธ์ is-a
```

---

### ข้อ 8: อธิบาย attr_accessor, attr_reader, attr_writer

**คำตอบ:**

```ruby
class Person
  # attr_reader - สร้าง getter method
  attr_reader :name

  # attr_writer - สร้าง setter method
  attr_writer :email

  # attr_accessor - สร้างทั้ง getter และ setter
  attr_accessor :age

  def initialize(name, age, email)
    @name  = name
    @age   = age
    @email = email
  end
end

person = Person.new("Alice", 30, "alice@example.com")

# getter
person.name   # => "Alice"
person.age    # => 30

# setter
person.age   = 31   # ได้
person.email = "newemail@example.com"  # ได้

# person.name = "Bob"  # NoMethodError! ไม่มี name setter

# เทียบเท่าเขียนมือ
class Person
  def name
    @name
  end

  def age
    @age
  end

  def age=(value)
    @age = value
  end
end
```

---

### ข้อ 9: อธิบาย Enumerable module

**คำตอบ:**

```ruby
# Enumerable ให้ collection methods เมื่อ implement each
class NumberCollection
  include Enumerable

  def initialize(*numbers)
    @numbers = numbers
  end

  def each(&block)
    @numbers.each(&block)
  end
end

nc = NumberCollection.new(3, 1, 4, 1, 5, 9, 2, 6)

# Enumerable methods ทำงานได้ทันที
nc.sort          # => [1, 1, 2, 3, 4, 5, 6, 9]
nc.min           # => 1
nc.max           # => 9
nc.sum           # => 31
nc.select { |n| n > 3 }  # => [4, 5, 9, 6]
nc.map { |n| n * 2 }     # => [6, 2, 8, 2, 10, 18, 4, 12]
nc.each_with_index.map { |n, i| "#{i}: #{n}" }
nc.group_by { |n| n > 3 ? :big : :small }
nc.partition { |n| n.even? }  # => [[4, 2, 6], [3, 1, 1, 5, 9]]
nc.flat_map { |n| [n, -n] }
nc.zip(nc.map { |n| n**2 })
nc.chunk { |n| n > 5 }.to_a
```

---

### ข้อ 10: อธิบาย Exception Handling

**คำตอบ:**

```ruby
# begin/rescue/ensure/else
def divide(a, b)
  begin
    result = a / b
  rescue ZeroDivisionError => e
    puts "Error: #{e.message}"
    result = nil
  rescue TypeError => e
    puts "Type Error: #{e.message}"
    result = nil
  else
    puts "Success! Result: #{result}"
  ensure
    puts "This always runs"
  end

  result
end

# raise custom exception
class InsufficientFundsError < StandardError
  def initialize(amount, balance)
    super("Cannot withdraw #{amount}. Balance: #{balance}")
    @amount  = amount
    @balance = balance
  end
end

def withdraw(amount, balance)
  raise ArgumentError, "Amount must be positive" unless amount > 0
  raise InsufficientFundsError.new(amount, balance) if amount > balance

  balance - amount
end

# retry
attempts = 0
begin
  attempts += 1
  raise "Network error" if attempts < 3
  puts "Success after #{attempts} attempts"
rescue => e
  retry if attempts < 3
  raise
end
```

---

### ข้อ 11-15: Intermediate Ruby Questions

### ข้อ 11: Method Missing และ Respond To Missing

```ruby
class DynamicClass
  def method_missing(method_name, *args, &block)
    if method_name.to_s.start_with?("find_by_")
      attribute = method_name.to_s.sub("find_by_", "")
      puts "Finding by #{attribute} with value #{args.first}"
    else
      super  # IMPORTANT: delegate to parent
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    method_name.to_s.start_with?("find_by_") || super
  end
end

d = DynamicClass.new
d.find_by_name("Alice")   # Finding by name with value Alice
d.respond_to?(:find_by_anything)  # => true
```

---

### ข้อ 12: Comparable Module

```ruby
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  # ต้อง implement <=> เท่านั้น
  def <=>(other)
    degrees <=> other.degrees
  end

  def to_s
    "#{degrees}°C"
  end
end

temps = [Temperature.new(30), Temperature.new(20), Temperature.new(25)]
temps.sort          # => [20°C, 25°C, 30°C]
temps.min           # => 20°C
temps.max           # => 30°C
Temperature.new(25).between?(Temperature.new(20), Temperature.new(30))  # => true
Temperature.new(25).clamp(Temperature.new(22), Temperature.new(28))     # => 25°C
```

---

### ข้อ 13: Frozen Objects และ Immutability

```ruby
# freeze - ทำให้ object ไม่เปลี่ยนได้
str = "hello".freeze
str << " world"  # RuntimeError: can't modify frozen String

# frozen? check
str.frozen?  # => true

# dup vs clone
str2 = str.dup    # dup สร้าง copy ที่ไม่ frozen
str3 = str.clone  # clone รักษา frozen state

str2.frozen?  # => false
str3.frozen?  # => true

# Ruby 3.x: frozen string literals
# frozen_string_literal: true

# ทำไมต้อง freeze?
# 1. ป้องกัน mutation bugs
# 2. Performance - frozen strings สามารถ share ใน memory
# 3. Thread safety

CONSTANT_HASH = { key: "value" }.freeze
# CONSTANT_HASH[:new] = "value"  # => RuntimeError

# Deep freeze (recursively)
deep_frozen = { a: [1, 2, 3], b: { c: "hello" } }
deep_frozen.each_value { |v| v.freeze if v.respond_to?(:freeze) }
deep_frozen.freeze
```

---

### ข้อ 14: Closures และ Scope

```ruby
# Closure - function ที่จำ scope ที่สร้างมา
x = 10

multiply = lambda { |n| n * x }
multiply.call(5)  # => 50

x = 20
multiply.call(5)  # => 100 (ใช้ x ค่าใหม่!)

# Local Scope
def outer
  x = "outer"

  inner = lambda do
    y = "inner"
    puts x  # สามารถ access outer x
  end

  inner.call
  # puts y  # NameError! y ไม่อยู่ใน scope นี้
end

# Instance vs Class vs Local Variables
class Counter
  @@total_count = 0  # Class variable

  def initialize
    @count = 0          # Instance variable
    @@total_count += 1
  end

  def increment
    count = 0          # Local variable (shadow!)
    count += 1
    @count += 1
  end

  def self.total = @@total_count
  def count      = @count
end
```

---

### ข้อ 15: Metaprogramming Basics

```ruby
# define_method - สร้าง method แบบ dynamic
class Person
  ATTRIBUTES = [:name, :age, :email].freeze

  ATTRIBUTES.each do |attr|
    define_method(attr) do
      instance_variable_get("@#{attr}")
    end

    define_method("#{attr}=") do |value|
      instance_variable_set("@#{attr}", value)
    end
  end
end

# send - เรียก method ด้วยชื่อเป็น string/symbol
obj = "hello"
obj.send(:upcase)          # => "HELLO"
obj.send(:[], 0)           # => "h"

# public_send - เรียกได้เฉพาะ public methods
class Secret
  private

  def hidden
    "secret"
  end
end

s = Secret.new
# s.send(:hidden)         # => "secret" (bypass!)
# s.public_send(:hidden)  # => NoMethodError

# class_eval / module_eval - เพิ่ม method ใน runtime
String.class_eval do
  def palindrome?
    self == self.reverse
  end
end

"racecar".palindrome?  # => true
"hello".palindrome?    # => false

# instance_eval - เปลี่ยน self ในส่วนนั้น
obj = Object.new
obj.instance_eval do
  def hello
    "Hello from instance!"
  end
end

obj.hello  # => "Hello from instance!"
```

---

## Mid-Level Questions (ข้อ 16-35)

### ข้อ 16: Memoization

```ruby
# Simple memoization ด้วย ||=
class Calculator
  def expensive_computation
    @result ||= begin
      puts "Computing..."
      sleep(1)
      42
    end
  end
end

calc = Calculator.new
calc.expensive_computation  # Computing... => 42
calc.expensive_computation  # => 42 (ไม่คำนวณใหม่)

# Memoize ด้วย parameter
class Fibonacci
  def initialize
    @cache = {}
  end

  def compute(n)
    return n if n <= 1
    @cache[n] ||= compute(n - 1) + compute(n - 2)
  end
end

# อย่าใช้ ||= กับ false/nil values
class Config
  def debug_mode
    @debug_mode = fetch_from_env  # ถูก!
    # @debug_mode ||= fetch_from_env  # ผิด! จะ refetch ถ้า false
  end

  def debug_mode
    return @debug_mode if defined?(@debug_mode)  # ถูก!
    @debug_mode = fetch_from_env
  end
end
```

---

### ข้อ 17: Struct ใน Ruby

```ruby
# Struct - lightweight class สำหรับ data
Point = Struct.new(:x, :y)
p = Point.new(3, 4)
p.x    # => 3
p.y    # => 4
p.to_a # => [3, 4]

# เพิ่ม methods
Point = Struct.new(:x, :y) do
  def distance_to(other)
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end

  def to_s
    "(#{x}, #{y})"
  end
end

p1 = Point.new(0, 0)
p2 = Point.new(3, 4)
p1.distance_to(p2)  # => 5.0

# keyword_init: true (Ruby 2.5+)
Person = Struct.new(:name, :age, keyword_init: true)
alice = Person.new(name: "Alice", age: 30)

# Data (Ruby 3.2+) - immutable struct
Point = Data.define(:x, :y)
p = Point.new(x: 1, y: 2)
# p.x = 3  # NoMethodError! frozen
```

---

### ข้อ 18: Enumerable ขั้นสูง

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# each_slice - แบ่งเป็นกลุ่ม
numbers.each_slice(3).to_a
# => [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

# each_cons - sliding window
numbers.each_cons(3).to_a
# => [[1,2,3], [2,3,4], [3,4,5], ...]

# flat_map
[[1, 2], [3, 4], [5, 6]].flat_map { |a| a.map { |n| n * 2 } }
# => [2, 4, 6, 8, 10, 12]

# group_by
numbers.group_by { |n| n % 3 }
# => {1=>[1, 4, 7, 10], 2=>[2, 5, 8], 0=>[3, 6, 9]}

# tally (Ruby 2.7+)
["a", "b", "a", "c", "b", "a"].tally
# => {"a"=>3, "b"=>2, "c"=>1}

# filter_map (Ruby 2.7+)
numbers.filter_map { |n| n * 2 if n.odd? }
# => [2, 6, 10, 14, 18]

# sum with initial value
numbers.sum(100)  # => 155

# minmax
numbers.minmax     # => [1, 10]
numbers.minmax_by { |n| n.to_s }  # lexicographic

# chunk_while
[1, 2, 3, 5, 6, 10, 11, 12].chunk_while { |i, j| j - i == 1 }.to_a
# => [[1, 2, 3], [5, 6], [10, 11, 12]]

# lazy evaluation
(1..Float::INFINITY).lazy.select { |n| n.odd? }.first(5)
# => [1, 3, 5, 7, 9]
```

---

### ข้อ 19: Refinements

```ruby
# Refinements - extend class เฉพาะใน scope
module StringExtensions
  refine String do
    def palindrome?
      self == self.reverse
    end

    def word_count
      split.size
    end
  end
end

# ใช้เฉพาะใน file/module ที่ using
module MyApp
  using StringExtensions

  def self.check(str)
    str.palindrome?
  end
end

MyApp.check("racecar")  # => true
# "racecar".palindrome?  # => NoMethodError! ใช้นอก scope ไม่ได้
```

---

### ข้อ 20: Thread Safety

```ruby
# Mutex - lock เพื่อ thread safety
require 'thread'

class BankAccount
  def initialize(balance)
    @balance = balance
    @mutex   = Mutex.new
  end

  def deposit(amount)
    @mutex.synchronize do
      @balance += amount
    end
  end

  def withdraw(amount)
    @mutex.synchronize do
      raise "Insufficient funds" if @balance < amount
      @balance -= amount
    end
  end

  def balance
    @mutex.synchronize { @balance }
  end
end

# Thread-safe operations
account = BankAccount.new(1000)

threads = 10.times.map do
  Thread.new { account.deposit(100) }
end
threads.each(&:join)

account.balance  # => 2000 (ไม่ใช่ random ค่า)

# ปัญหา race condition
counter = 0
# without mutex:
threads = 10.times.map do
  Thread.new { 1000.times { counter += 1 } }  # UNSAFE!
end
threads.each(&:join)
# counter อาจไม่ใช่ 10000
```

---

## Senior Level (ข้อ 21-35)

### ข้อ 21: Fiber ใน Ruby

```ruby
# Fiber - cooperative concurrency
fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield
  puts "Step 2"
  Fiber.yield
  puts "Step 3"
end

fiber.resume  # Step 1
fiber.resume  # Step 2
fiber.resume  # Step 3
# fiber.resume  # FiberError: dead fiber called

# Fiber ที่รับและส่งค่า
producer = Fiber.new do
  5.times do |i|
    Fiber.yield(i * 2)
  end
  nil
end

loop do
  value = producer.resume
  break if value.nil?
  puts value
end
# 0, 2, 4, 6, 8

# Enumerator ใช้ Fiber ภายใน
enum = Enumerator.new do |yielder|
  yielder << 1
  yielder << 2
  yielder << 3
end

enum.next  # => 1
enum.next  # => 2
```

---

### ข้อ 22: ObjectSpace

```ruby
require 'objspace'

# ดู objects ทั้งหมดใน memory
ObjectSpace.each_object(String).count  # จำนวน String objects

# หา object ด้วย id
obj_id = "hello".object_id
ObjectSpace._id2ref(obj_id)  # => "hello"

# GC statistics
GC.stat
# => { count: 1, heap_allocated_pages: 93, ... }

# Trace object allocation
ObjectSpace.trace_object_allocations_start
# ... code ที่ต้องการ trace ...
ObjectSpace.trace_object_allocations_stop

# Memory profile
str = String.new
puts ObjectSpace.memsize_of(str)  # bytes ที่ใช้
```

---

### ข้อ 23: Pattern Matching (Ruby 3.x)

```ruby
# Case/in pattern matching
case { name: "Alice", age: 30, role: :admin }
in { name: String => name, role: :admin }
  puts "Admin: #{name}"
in { name: String => name }
  puts "User: #{name}"
end
# => "Admin: Alice"

# Array pattern
case [1, 2, 3, 4, 5]
in [Integer => first, Integer => second, *rest]
  puts "First: #{first}, Second: #{second}, Rest: #{rest}"
end
# => "First: 1, Second: 2, Rest: [3, 4, 5]"

# Find pattern
case [1, 2, "three", 4, 5]
in [*, String => str, *]
  puts "Found string: #{str}"
end
# => "Found string: three"

# Deconstruct keys
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
  end

  def deconstruct_keys(keys)
    { x: @x, y: @y }
  end
end

case Point.new(1, 2)
in { x: 0..5 => x, y: 0..5 => y }
  puts "In range: (#{x}, #{y})"
end
```

---

### ข้อ 24: Ractor (Ruby 3.x Experimental)

```ruby
# Ractor - truly parallel Ruby (experimental)
r1 = Ractor.new do
  Ractor.yield(1 + 1)
end

r2 = Ractor.new(r1) do |r|
  value = r.take
  value * 10
end

r2.take  # => 20

# Parallel processing
workers = 4.times.map do
  Ractor.new do
    loop do
      job = Ractor.receive
      Ractor.yield(job * 2)
    end
  end
end
```

---

### ข้อ 25: Memory Management และ GC

```ruby
# Ruby GC ใช้ tri-color mark-and-sweep algorithm

# ปรับ GC parameters
GC::Profiler.enable
# ... run code ...
GC::Profiler.report
GC::Profiler.disable

# GC tuning environment variables
# RUBY_GC_HEAP_INIT_SLOTS=10000
# RUBY_GC_HEAP_FREE_SLOTS=4096
# RUBY_GC_HEAP_GROWTH_FACTOR=1.8
# RUBY_GC_MALLOC_LIMIT=16MB

# Weak references
require 'weakref'

obj = Object.new
weak = WeakRef.new(obj)
weak.weakref_alive?  # => true

obj = nil
GC.start
weak.weakref_alive?  # => false

# ObjectSpace::WeakMap
cache = ObjectSpace::WeakMap.new
key = Object.new
cache[key] = "value"
# key ถูก GC ได้เมื่อไม่มี strong reference อื่น
```

---

### ข้อ 26-30: Ruby Performance Questions

### ข้อ 26: String Performance

```ruby
# String concatenation - ช้า
result = ""
1000.times { |i| result += i.to_s }  # สร้าง object ใหม่ทุกครั้ง!

# ใช้ << แทน (เร็วกว่า 10x)
result = ""
1000.times { |i| result << i.to_s }

# ใช้ Array join (เร็วที่สุด)
parts = []
1000.times { |i| parts << i.to_s }
result = parts.join

# frozen_string_literal: true
# ป้องกัน String allocation ซ้ำ
str1 = "hello"  # ไม่สร้าง object ใหม่ถ้า frozen
str2 = "hello"  # อ้างถึง object เดิม

# String.new vs literal
mutable = String.new("hello")  # ไม่ frozen
frozen  = "hello"              # frozen ถ้า frozen_string_literal: true
```

---

### ข้อ 27: Benchmark

```ruby
require 'benchmark'

n = 1_000_000

Benchmark.bm(20) do |x|
  x.report("Array#map:") do
    n.times { [1, 2, 3].map { |i| i * 2 } }
  end

  x.report("Array#each:") do
    n.times do
      result = []
      [1, 2, 3].each { |i| result << i * 2 }
    end
  end
end
```

---

### ข้อ 28-30: Advanced Ruby Concepts

### ข้อ 28: Eigenclass / Singleton Class

```ruby
class MyClass
  class << self
    def singleton_method
      "I'm a singleton method"
    end
  end
end

# หรือ
obj = Object.new
def obj.special_method
  "Only this object has this method"
end

# Eigenclass
obj.singleton_class          # => #<Class:#<Object:...>>
obj.singleton_class.ancestors

# เทียบกับ class method
class Dog
  def self.breed_info    # เพิ่มใน Dog's eigenclass
    "I'm a dog"
  end
end
```

---

### ข้อ 29: Method Lookup Path (MRO)

```ruby
module A
  def hello
    "A#hello " + (super rescue "")
  end
end

module B
  def hello
    "B#hello " + (super rescue "")
  end
end

class C
  include A
  include B  # B ถูก include หลัง จะค้นหาก่อน

  def hello
    "C#hello " + super
  end
end

C.ancestors
# => [C, B, A, Object, Kernel, BasicObject]

C.new.hello
# => "C#hello B#hello A#hello "

# prepend - insert ก่อน class
module Logging
  def hello
    puts "Calling hello..."
    result = super
    puts "Called!"
    result
  end
end

class D
  prepend Logging

  def hello
    "D#hello"
  end
end

D.ancestors
# => [Logging, D, Object, ...]
D.new.hello
# Calling hello...
# Called!
# => "D#hello"
```

---

### ข้อ 30: Concurrent Ruby Patterns

```ruby
require 'concurrent-ruby'

# Future
future = Concurrent::Future.execute do
  sleep(1)
  "Result"
end

future.value   # blocks until complete => "Result"
future.value!  # raises if exception occurred

# Promise
Concurrent::Promise.execute { 1 + 1 }
                   .then { |v| v * 10 }
                   .then { |v| "Result: #{v}" }
                   .value  # => "Result: 20"

# Atom - thread-safe mutable reference
atom = Concurrent::Atom.new(0)
10.times { atom.swap { |n| n + 1 } }
atom.value  # => 10

# IVar - single-assignment variable
ivar = Concurrent::IVar.new
Thread.new { sleep(0.5); ivar.set("computed value") }
ivar.value  # blocks => "computed value"
```

---

# ส่วนที่ 2: Rails Interview Questions (50 ข้อ)

---

## Junior Rails (ข้อ 31-45)

### ข้อ 31: MVC Pattern ใน Rails คืออะไร?

**คำตอบ:**

```
Model (M) - Business Logic, Database interaction
View  (V) - Presentation, HTML templates
Controller (C) - Orchestration, handles HTTP requests

Request Flow:
Browser → Router → Controller → Model → Controller → View → Browser
```

```ruby
# Router
# config/routes.rb
get '/articles/:id', to: 'articles#show'
resources :articles  # CRUD routes

# Controller
class ArticlesController < ApplicationController
  def show
    @article = Article.find(params[:id])  # delegate to model
  end
end

# Model
class Article < ApplicationRecord
  validates :title, presence: true
  belongs_to :user
  has_many :comments
end

# View (app/views/articles/show.html.erb)
# <h1><%= @article.title %></h1>
```

---

### ข้อ 32: ActiveRecord Associations

**คำตอบ:**

```ruby
class User < ApplicationRecord
  # has_many - user มีหลาย posts
  has_many :posts, dependent: :destroy

  # has_one - user มี profile เดียว
  has_one :profile, dependent: :destroy

  # has_many :through - many-to-many ผ่าน join table
  has_many :memberships
  has_many :groups, through: :memberships

  # has_and_belongs_to_many - many-to-many ไม่มี model กลาง
  has_and_belongs_to_many :tags

  # polymorphic
  has_many :comments, as: :commentable
end

class Post < ApplicationRecord
  # belongs_to
  belongs_to :user

  # belongs_to optional
  belongs_to :category, optional: true

  # polymorphic belongs_to
  belongs_to :commentable, polymorphic: true
end

class Profile < ApplicationRecord
  belongs_to :user
  # inverse_of สำหรับ bidirectional
  # belongs_to :user, inverse_of: :profile
end

# Query ผ่าน associations
user = User.first
user.posts             # => all posts
user.posts.published   # => scope on association
user.posts.count       # => SQL COUNT
user.posts.build(title: "New")  # build ไม่ save
user.posts.create!(title: "New")  # create และ save
```

---

### ข้อ 33: ActiveRecord Validations

```ruby
class User < ApplicationRecord
  # Presence
  validates :name,  presence: true
  validates :email, presence: true

  # Uniqueness
  validates :email, uniqueness: true
  validates :email, uniqueness: { scope: :company_id, case_sensitive: false }

  # Format
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }

  # Length
  validates :username, length: { minimum: 3, maximum: 20 }
  validates :bio,      length: { maximum: 500 }

  # Numericality
  validates :age, numericality: { greater_than: 0, less_than: 150 }
  validates :score, numericality: { only_integer: true }

  # Inclusion/Exclusion
  validates :status, inclusion: { in: %w[active inactive banned] }
  validates :username, exclusion: { in: %w[admin root superuser] }

  # Conditional
  validates :company_name, presence: true, if: :business_account?

  # Custom
  validate :email_domain_not_blocked

  private

  def email_domain_not_blocked
    blocked = ["spam.com", "temp.org"]
    domain  = email&.split('@')&.last
    errors.add(:email, "domain is blocked") if domain.in?(blocked)
  end
end

# Check validity
user = User.new(name: "Alice", email: "invalid")
user.valid?    # => false
user.errors.full_messages  # => ["Email is invalid"]
user.errors[:email]        # => ["is invalid"]

# ข้ามบาง validations
user.save(validate: false)  # อันตราย!
```

---

### ข้อ 34: ActiveRecord Callbacks

```ruby
class Order < ApplicationRecord
  # Before callbacks
  before_validation :normalize_data
  before_create     :set_order_number
  before_save       :calculate_total
  before_destroy    :check_can_destroy

  # After callbacks
  after_create  :send_confirmation_email
  after_update  :notify_status_change, if: :saved_change_to_status?
  after_destroy :cleanup_files
  after_commit  :update_search_index
  after_rollback :handle_failure

  # Around callbacks
  around_save :log_save_duration

  private

  def normalize_data
    self.email = email&.downcase&.strip
  end

  def set_order_number
    self.number = "ORD-#{Time.current.to_i}"
  end

  def check_can_destroy
    throw(:abort) if status == 'processing'
    # throw :abort หยุดการ destroy
  end

  def around_save
    start = Time.current
    yield
    duration = Time.current - start
    Rails.logger.info "Save took #{duration}s"
  end
end

# Callbacks ทำงานเมื่อ?
# save, create, update, destroy, validate
# ไม่ทำงานเมื่อ: update_column, update_all, delete, delete_all
```

---

### ข้อ 35: N+1 Query Problem

```ruby
# ปัญหา N+1
users = User.all
users.each { |u| puts u.posts.count }
# Query 1: SELECT * FROM users
# Query 2..N: SELECT COUNT(*) FROM posts WHERE user_id = ?

# แก้ด้วย includes
users = User.includes(:posts)
users.each { |u| puts u.posts.size }
# Query 1: SELECT * FROM users
# Query 2: SELECT * FROM posts WHERE user_id IN (1,2,3,...)

# includes vs eager_load vs preload
User.includes(:posts)   # Rails เลือก preload หรือ LEFT OUTER JOIN
User.preload(:posts)    # แยก query เสมอ
User.eager_load(:posts) # LEFT OUTER JOIN เสมอ (ใช้ WHERE ได้)

# ใช้กับ WHERE
User.eager_load(:posts).where(posts: { published: true })

# counter_cache - เก็บ count ใน parent
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end
# users table ต้องมี column posts_count

# Bullet gem - detect N+1
# config/environments/development.rb
# config.after_initialize do
#   Bullet.enable = true
#   Bullet.alert = true
# end
```

---

### ข้อ 36: ActiveRecord Scopes

```ruby
class Article < ApplicationRecord
  scope :published,  -> { where(published: true) }
  scope :recent,     -> { order(created_at: :desc) }
  scope :by_author,  ->(author) { where(author: author) }
  scope :popular,    -> { where('views_count > ?', 100) }

  # Scope ที่ chainable
  scope :search, ->(query) {
    where("title ILIKE ? OR body ILIKE ?", "%#{query}%", "%#{query}%")
  }

  # default_scope (ใช้ระวัง!)
  default_scope { order(created_at: :desc) }

  # unscoped - ลบ default_scope
  def self.all_unordered
    unscoped.all
  end
end

# Chain scopes
Article.published.recent.by_author("Alice").limit(10)

# นิยมใช้ scope vs class method
# Class method ยืดหยุ่นกว่าเมื่อ logic ซับซ้อน
def self.popular_in(category)
  return popular if category.nil?
  popular.where(category: category)
end
```

---

### ข้อ 37: Migrations

```ruby
# สร้าง migration
rails generate migration CreateProducts name:string price:decimal{10,2}

# Migration DSL
class CreateProducts < ActiveRecord::Migration[7.1]
  def change
    create_table :products do |t|
      t.string  :name,        null: false
      t.decimal :price,       precision: 10, scale: 2
      t.text    :description
      t.integer :stock,       default: 0
      t.boolean :published,   default: false
      t.references :category, foreign_key: true

      t.timestamps
    end

    add_index :products, :name
    add_index :products, [:category_id, :published]
  end
end

# Reversible migration
class AddColumnToUsers < ActiveRecord::Migration[7.1]
  def up
    add_column :users, :role, :string, default: 'user'
  end

  def down
    remove_column :users, :role
  end
end

# Strong migrations - ป้องกัน ลด downtime
class AddIndexConcurrently < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!

  def change
    add_index :users, :email, algorithm: :concurrently
  end
end
```

---

### ข้อ 38: Strong Parameters

```ruby
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    # ...
  end

  def update
    @user = User.find(params[:id])
    @user.update(user_params)
    # ...
  end

  private

  def user_params
    params.require(:user).permit(
      :name,
      :email,
      :password,
      :password_confirmation,
      profile_attributes: [:bio, :website, :avatar],
      roles: []
    )
  end
end

# Nested attributes
class UserController
  def user_params
    params.require(:user).permit(
      :name,
      addresses_attributes: [:id, :street, :city, :_destroy]
    )
  end
end

# Dynamic permitted params
def article_params
  permitted = [:title, :body]
  permitted << :published if current_user.admin?
  params.require(:article).permit(*permitted)
end
```

---

### ข้อ 39: Rails Routing

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # CRUD routes
  resources :articles do
    resources :comments, only: [:create, :destroy]

    member do
      post :publish
      delete :unpublish
    end

    collection do
      get :search
      get :popular
    end
  end

  # Nested routes
  resources :users do
    resource :profile, only: [:show, :edit, :update]
  end

  # Custom routes
  get  '/about',     to: 'pages#about', as: :about
  post '/login',     to: 'sessions#create'
  delete '/logout',  to: 'sessions#destroy'

  # Redirect
  get '/old-path', to: redirect('/new-path')

  # Constraints
  constraints(subdomain: 'api') do
    namespace :api, path: '' do
      resources :v1 do
        # ...
      end
    end
  end

  # Namespace
  namespace :admin do
    resources :users
    resources :reports
  end

  root to: 'home#index'
end
```

---

### ข้อ 40: Caching ใน Rails

```ruby
# Fragment caching
# app/views/articles/show.html.erb
# <% cache @article do %>
#   <h1><%= @article.title %></h1>
# <% end %>

# Russian doll caching
# <% cache @article do %>
#   <% @article.comments.each do |comment| %>
#     <% cache comment do %>
#       ...
#     <% end %>
#   <% end %>
# <% end %>

# Low-level caching
class Article < ApplicationRecord
  def expensive_stats
    Rails.cache.fetch("article_#{id}_stats", expires_in: 1.hour) do
      # expensive computation
      { word_count: body.split.length, read_time: body.split.length / 200 }
    end
  end
end

# Action caching
class ArticlesController < ApplicationController
  def index
    @articles = Rails.cache.fetch("articles_index", expires_in: 15.minutes) do
      Article.published.recent.limit(20)
    end
  end
end

# Cache store config
config.cache_store = :redis_cache_store, {
  url:        ENV['REDIS_URL'],
  expires_in: 1.hour,
  namespace:  'myapp'
}

# HTTP Caching
def show
  @article = Article.find(params[:id])
  fresh_when(etag: @article, last_modified: @article.updated_at)
end
```

---

## Mid-Level Rails (ข้อ 41-55)

### ข้อ 41: Service Objects

```ruby
# Plain Ruby Service Object
class UserRegistrationService
  Result = Struct.new(:success?, :user, :errors, keyword_init: true)

  def initialize(params)
    @params = params
  end

  def call
    user = User.new(@params)

    if user.save
      send_welcome_email(user)
      create_default_settings(user)
      track_signup(user)

      Result.new(success?: true, user: user, errors: [])
    else
      Result.new(success?: false, user: nil, errors: user.errors.full_messages)
    end
  end

  private

  def send_welcome_email(user)
    UserMailer.welcome(user).deliver_later
  end

  def create_default_settings(user)
    UserSettings.create!(user: user, theme: 'light', notifications: true)
  end

  def track_signup(user)
    Analytics.track(user_id: user.id, event: 'signup')
  end
end

# Controller ใช้งาน
class UsersController < ApplicationController
  def create
    result = UserRegistrationService.new(user_params).call

    if result.success?
      redirect_to root_path, notice: "Welcome!"
    else
      @errors = result.errors
      render :new, status: :unprocessable_entity
    end
  end
end
```

---

### ข้อ 42: Background Jobs

```ruby
# Active Job
class SendEmailJob < ApplicationJob
  queue_as :mailers

  # Retry configuration
  retry_on Net::OpenTimeout, wait: :polynomially_longer, attempts: 5
  discard_on ActiveJob::DeserializationError

  def perform(user_id, subject, body)
    user = User.find(user_id)
    UserMailer.custom(user, subject, body).deliver_now
  end
end

# Enqueue
SendEmailJob.perform_later(user.id, "Subject", "Body")
SendEmailJob.set(wait: 5.minutes).perform_later(user.id, "Subject", "Body")
SendEmailJob.set(wait_until: Date.tomorrow.noon).perform_later(...)

# Sidekiq config
# config/sidekiq.yml
# :queues:
#   - [critical, 3]
#   - [default, 2]
#   - [mailers, 1]

# Sidekiq callbacks
class ImportJob < ApplicationJob
  around_perform do |job, block|
    Rails.logger.info "Starting import #{job.job_id}"
    block.call
    Rails.logger.info "Finished import #{job.job_id}"
  end
end
```

---

### ข้อ 43: Action Mailer

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  default from: 'noreply@myapp.com'

  def welcome(user)
    @user    = user
    @sign_in_url = sign_in_url

    mail(
      to:      @user.email,
      subject: "Welcome to MyApp!"
    )
  end

  def reset_password(user, token)
    @user           = user
    @reset_url      = edit_password_reset_url(token)
    @expires_in     = "24 hours"

    mail(
      to:      @user.email,
      subject: "Reset your password"
    )
  end

  def weekly_report(user)
    @user    = user
    @report  = WeeklyReportService.new(user).generate

    attachments['report.pdf'] = generate_pdf(@report)

    mail(
      to:      @user.email,
      subject: "Your weekly report"
    )
  end

  private

  def generate_pdf(data)
    # WickedPDF or Prawn
  end
end

# Deliver
UserMailer.welcome(@user).deliver_now
UserMailer.welcome(@user).deliver_later
UserMailer.welcome(@user).deliver_later(wait: 5.minutes)
```

---

### ข้อ 44: Authentication vs Authorization

```ruby
# Authentication = ใครคุณ
# Devise gem
devise :database_authenticatable, :registerable,
       :recoverable, :rememberable, :validatable,
       :confirmable, :lockable, :trackable

# Custom JWT Auth
class ApplicationController < ActionController::API
  before_action :authenticate_user!

  private

  def authenticate_user!
    token   = request.headers['Authorization']&.split(' ')&.last
    payload = JWT.decode(token, Rails.application.credentials.secret_key_base)
    @current_user = User.find(payload.first['user_id'])
  rescue JWT::DecodeError
    render json: { error: 'Unauthorized' }, status: :unauthorized
  end
end

# Authorization = คุณทำอะไรได้
# Pundit gem
class ArticlePolicy < ApplicationPolicy
  def show?    = true
  def create?  = user.present?
  def update?  = user == record.author || user.admin?
  def destroy? = user.admin?
end

class ArticlesController < ApplicationController
  def update
    @article = Article.find(params[:id])
    authorize @article  # ใช้ ArticlePolicy#update?

    if @article.update(article_params)
      redirect_to @article
    else
      render :edit
    end
  end
end

# CanCanCan gem
class Ability
  include CanCan::Ability

  def initialize(user)
    can :read, Article, published: true
    return unless user.present?

    can :create, Article
    can :update, Article, user_id: user.id
    can :manage, :all if user.admin?
  end
end
```

---

### ข้อ 45: Testing ใน Rails

```ruby
# RSpec + FactoryBot
# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  subject(:user) { build(:user) }

  describe 'validations' do
    it { should validate_presence_of(:email) }
    it { should validate_uniqueness_of(:email).case_insensitive }
    it { should validate_presence_of(:name) }
  end

  describe 'associations' do
    it { should have_many(:posts).dependent(:destroy) }
    it { should have_one(:profile) }
  end

  describe '#full_name' do
    it 'combines first and last name' do
      user = build(:user, first_name: 'John', last_name: 'Doe')
      expect(user.full_name).to eq('John Doe')
    end
  end
end

# spec/requests/articles_spec.rb
RSpec.describe 'Articles', type: :request do
  let(:user) { create(:user) }
  let(:headers) { auth_headers(user) }

  describe 'GET /articles' do
    before { create_list(:article, 5, published: true) }

    it 'returns published articles' do
      get '/articles', headers: headers
      expect(response).to have_http_status(:ok)
      expect(json['articles'].length).to eq(5)
    end
  end
end

# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name  { Faker::Name.full_name }
    email { Faker::Internet.unique.email }
    password { 'password123' }

    trait :admin do
      role { 'admin' }
    end

    trait :with_posts do
      after(:create) { |user| create_list(:post, 3, user: user) }
    end
  end
end
```

---

## Senior Rails (ข้อ 46-60)

### ข้อ 46: Performance Optimization

```ruby
# 1. Database Query Optimization

# ใช้ select เฉพาะ columns ที่ต้องการ
User.select(:id, :name, :email).limit(100)

# ใช้ pluck สำหรับ single values
User.pluck(:email)  # => ["a@test.com", ...]
User.pluck(:id, :email)  # => [[1, "a@..."], ...]

# Batch processing - ไม่โหลด ทั้งหมดใน memory
User.find_each(batch_size: 1000) { |user| process(user) }

User.in_batches(of: 1000) do |batch|
  batch.update_all(updated_at: Time.current)
end

# 2. Caching
def expensive_query
  Rails.cache.fetch("expensive_#{cache_key}", expires_in: 5.minutes) do
    # heavy computation
  end
end

# 3. Bulk operations
# Bad
users.each { |u| u.update!(active: false) }

# Good
User.where(id: users.map(&:id)).update_all(active: false)

# 4. Database indexes
add_index :orders, [:user_id, :status, :created_at]
add_index :products, :name, using: :gin  # Full-text search

# 5. Connection Pooling
# database.yml
# pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

---

### ข้อ 47: Security Best Practices

```ruby
# 1. SQL Injection Prevention
# NEVER
User.where("email = '#{params[:email]}'")  # VULNERABLE!

# ALWAYS
User.where(email: params[:email])
User.where("email = ?", params[:email])
User.where("email = :email", email: params[:email])

# 2. XSS Prevention
# Rails auto-escapes ERB output
# <%= user_input %>     # safe - escaped
# <%= raw user_input %> # UNSAFE!
# <%= user_input.html_safe %> # UNSAFE unless you sanitize!

# Sanitize HTML
ActionController::Base.helpers.sanitize(
  user_content,
  tags: %w[p br strong em],
  attributes: %w[class]
)

# 3. CSRF Protection
# ApplicationController includes protect_from_forgery by default
protect_from_forgery with: :exception  # default for HTML
protect_from_forgery with: :null_session  # for API

# 4. Mass Assignment Protection
# ใช้ strong parameters (ดูข้อ 38)

# 5. Sensitive Data
# ใช้ credentials
Rails.application.credentials.stripe_api_key

# 6. Secure Headers
# gem 'secure_headers'
SecureHeaders::Configuration.default do |config|
  config.x_frame_options = "DENY"
  config.x_content_type_options = "nosniff"
  config.x_xss_protection = "1; mode=block"
  config.content_security_policy = {
    default_src: %w('none'),
    script_src:  %w('self'),
    style_src:   %w('self')
  }
end
```

---

### ข้อ 48: API Design

```ruby
# RESTful API with versioning
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      include Pagy::Backend

      before_action :authenticate_request!

      rescue_from ActiveRecord::RecordNotFound,    with: :not_found
      rescue_from ActiveRecord::RecordInvalid,     with: :unprocessable_entity
      rescue_from ActionController::ParameterMissing, with: :bad_request

      private

      def authenticate_request!
        token = request.headers['Authorization']&.sub(/^Bearer /, '')
        @current_user = User.find_by_token(token)
        render json: { error: 'Unauthorized' }, status: 401 unless @current_user
      end

      def not_found(exception)
        render json: { error: exception.message }, status: :not_found
      end

      def paginated_response(collection, serializer:)
        pagy, records = pagy(collection)

        render json: {
          data: records.map { |r| serializer.new(r).as_json },
          meta: pagy_metadata(pagy)
        }
      end
    end
  end
end

# Serializers
class UserSerializer
  def initialize(user)
    @user = user
  end

  def as_json
    {
      id:         @user.id,
      name:       @user.name,
      email:      @user.email,
      created_at: @user.created_at.iso8601
    }
  end
end
```

---

### ข้อ 49: Database Transactions

```ruby
# Basic transaction
ActiveRecord::Base.transaction do
  order = Order.create!(user: user, total: 100)
  PaymentRecord.create!(order: order, amount: 100)
  inventory.update!(quantity: inventory.quantity - 1)
end
# ถ้า exception เกิดขึ้น ทั้งหมด rollback

# Nested transactions (savepoints)
User.transaction do
  user.save!

  User.transaction(requires_new: true) do  # savepoint
    user.profile.save!
  end
end

# Transaction callbacks
class Order < ApplicationRecord
  after_commit :send_confirmation,  on: :create
  after_commit :notify_change,      on: :update
  after_rollback :handle_failure

  def send_confirmation
    OrderMailer.confirmation(self).deliver_later
  end
end

# lock! - pessimistic locking
def transfer_funds(from, to, amount)
  Account.transaction do
    from_account = Account.lock.find(from.id)
    to_account   = Account.lock.find(to.id)

    raise "Insufficient funds" if from_account.balance < amount

    from_account.update!(balance: from_account.balance - amount)
    to_account.update!(balance: to_account.balance + amount)
  end
end

# Optimistic locking
class Article < ApplicationRecord
  # ต้องมี column lock_version: integer
end

article = Article.find(1)
article.update!(title: "New")  # เพิ่ม WHERE lock_version = old_value
# raise ActiveRecord::StaleObjectError ถ้า version เปลี่ยน
```

---

### ข้อ 50: Rails Engines

```ruby
# สร้าง Engine
rails plugin new my_engine --mountable

# my_engine/lib/my_engine/engine.rb
module MyEngine
  class Engine < ::Rails::Engine
    isolate_namespace MyEngine

    initializer "my_engine.assets" do |app|
      app.config.assets.precompile += %w[my_engine/application.js]
    end

    config.generators do |g|
      g.test_framework :rspec
      g.fixture_replacement :factory_bot
    end
  end
end

# Mount ใน host app
# config/routes.rb
mount MyEngine::Engine, at: '/engine'

# ใช้ path helpers
my_engine.root_path
main_app.root_path
```

---

### ข้อ 51-60: Advanced Rails Topics

### ข้อ 51: ActiveRecord Query Interface

```ruby
# Complex queries
User.joins(:posts)
    .joins(:profile)
    .where(posts: { published: true })
    .where('profiles.completed = ?', true)
    .select('users.*, COUNT(posts.id) as post_count')
    .group('users.id')
    .having('COUNT(posts.id) > 5')
    .order('post_count DESC')
    .limit(10)

# Subquery
popular_user_ids = Post.group(:user_id)
                       .having('COUNT(*) > 10')
                       .pluck(:user_id)

User.where(id: popular_user_ids)

# exists?
User.where(id: Post.select(:user_id).where(published: true))

# Raw SQL (use carefully)
User.find_by_sql(<<~SQL)
  SELECT users.*, COUNT(posts.id) as post_count
  FROM users
  LEFT JOIN posts ON posts.user_id = users.id
  GROUP BY users.id
  ORDER BY post_count DESC
  LIMIT 10
SQL

# Arel for complex conditions
User.where(
  User.arel_table[:created_at].gteq(1.month.ago)
  .and(User.arel_table[:active].eq(true))
)
```

---

### ข้อ 52: Polymorphic Associations

```ruby
# Model
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
  belongs_to :user
end

class Article < ApplicationRecord
  has_many :comments, as: :commentable
end

class Photo < ApplicationRecord
  has_many :comments, as: :commentable
end

# Migration
create_table :comments do |t|
  t.text       :body
  t.references :user
  t.references :commentable, polymorphic: true
  t.timestamps
end
# สร้าง columns: commentable_id, commentable_type

# Usage
article.comments.create!(body: "Great!", user: user)
photo.comments.create!(body: "Nice photo!", user: user)

# Query
Comment.where(commentable_type: 'Article')
comment.commentable  # => Article หรือ Photo instance
```

---

### ข้อ 53: Concerns

```ruby
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  included do
    scope :search, ->(query) {
      where("name ILIKE :q OR description ILIKE :q", q: "%#{query}%")
    }
  end

  class_methods do
    def search_by_tags(*tags)
      joins(:tags).where(tags: { name: tags })
    end
  end

  def highlight_in(text, query)
    text.gsub(/#{Regexp.escape(query)}/i, "<mark>\\0</mark>")
  end
end

# ใช้ใน model
class Article < ApplicationRecord
  include Searchable
end

class Product < ApplicationRecord
  include Searchable
end

# Controller concerns
module Authenticatable
  extend ActiveSupport::Concern

  included do
    before_action :authenticate_user!
    helper_method :current_user
  end

  private

  def authenticate_user!
    redirect_to login_path unless user_signed_in?
  end
end
```

---

### ข้อ 54: ActionCable

```ruby
# Channel
class ChatChannel < ApplicationCable::Channel
  def subscribed
    @room = Room.find(params[:room_id])
    stream_for @room
  end

  def receive(data)
    message = @room.messages.create!(
      content: data['content'],
      user:    current_user
    )

    ChatChannel.broadcast_to(@room, {
      id:         message.id,
      content:    message.content,
      user:       current_user.username,
      created_at: message.created_at.iso8601
    })
  end
end

# Connection
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      User.find_by(id: cookies.encrypted[:user_id]) ||
        reject_unauthorized_connection
    end
  end
end

# Broadcast from anywhere
ActionCable.server.broadcast("room_#{room.id}", {
  type: 'message',
  content: message.content
})
```

---

### ข้อ 55: Deployment และ DevOps

```ruby
# Dockerfile
# FROM ruby:3.3-alpine
# WORKDIR /app
# COPY Gemfile* ./
# RUN bundle install
# COPY . .
# RUN bundle exec rails assets:precompile RAILS_ENV=production
# CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]

# Puma config
# config/puma.rb
workers ENV.fetch('WEB_CONCURRENCY', 2)
threads_count = ENV.fetch('RAILS_MAX_THREADS', 5)
threads threads_count, threads_count

preload_app!

on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end

# Health check
class HealthController < ApplicationController
  def show
    checks = {
      database: database_ok?,
      redis:    redis_ok?,
      sidekiq:  sidekiq_ok?
    }

    status = checks.values.all? ? :ok : :service_unavailable
    render json: { status: status, checks: checks }, status: status
  end

  private

  def database_ok?
    ActiveRecord::Base.connection.execute("SELECT 1")
    true
  rescue => e
    false
  end

  def redis_ok?
    Redis.new.ping == "PONG"
  rescue
    false
  end
end
```

---

### ข้อ 56: Rails Credentials และ Secrets

```ruby
# config/credentials.yml.enc (encrypted)
# rails credentials:edit
# production:
#   database_password: secret123
#   stripe:
#     api_key: sk_live_xxx
#     webhook_secret: whsec_xxx

# Access
Rails.application.credentials.database_password
Rails.application.credentials.stripe[:api_key]
Rails.application.credentials.dig(:stripe, :api_key)

# Multiple environments
rails credentials:edit --environment production
# config/credentials/production.yml.enc

# Master key
# RAILS_MASTER_KEY environment variable
# config/master.key (ไม่ commit ใน git!)
```

---

### ข้อ 57: ActiveStorage

```ruby
class User < ApplicationRecord
  has_one_attached  :avatar
  has_many_attached :documents
end

# Attach
user.avatar.attach(
  io:           File.open('avatar.jpg'),
  filename:     'avatar.jpg',
  content_type: 'image/jpeg'
)

# From controller
user.avatar.attach(params[:avatar])

# Check
user.avatar.attached?
user.avatar.blank?

# URL
url_for(user.avatar)
rails_blob_path(user.avatar, only_path: true)
rails_blob_url(user.avatar)

# Variants (image processing)
user.avatar.variant(resize_to_limit: [100, 100])
user.avatar.variant(resize_to_fill: [200, 200], format: :webp)

# Download
user.avatar.download
user.avatar.open { |file| process(file) }

# Direct upload (S3)
user.avatar.service_url  # Pre-signed URL

# Delete
user.avatar.purge
user.avatar.purge_later  # Background job
```

---

### ข้อ 58: Event-Driven Architecture ใน Rails

```ruby
# ActiveSupport::Notifications
# instrument event
ActiveSupport::Notifications.instrument('order.created', { order_id: order.id }) do
  order.process!
end

# subscribe
ActiveSupport::Notifications.subscribe('order.created') do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  Analytics.track(event: 'order_created', order_id: event.payload[:order_id])
end

# Custom event system
class EventBus
  def self.publish(event_name, payload = {})
    Rails.logger.info "Event: #{event_name}"
    subscribers(event_name).each { |s| s.call(payload) }
  end

  def self.subscribe(event_name, &handler)
    subscribers(event_name) << handler
  end

  def self.subscribers(event_name)
    @subscribers         ||= Hash.new { |h, k| h[k] = [] }
    @subscribers[event_name]
  end
end

EventBus.subscribe('user.signed_up') { |p| WelcomeEmailJob.perform_later(p[:user_id]) }
EventBus.subscribe('user.signed_up') { |p| Analytics.track('signup', p) }
EventBus.publish('user.signed_up', user_id: user.id)
```

---

### ข้อ 59: Advanced Caching Strategies

```ruby
# Cache key versioning
class Product < ApplicationRecord
  def cache_key_with_version
    "#{cache_key}-#{updated_at.to_i}-#{ENV['APP_VERSION']}"
  end
end

# Counter cache
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
  # users table: posts_count integer default 0
end

# Cached associations
class User < ApplicationRecord
  def cached_posts
    Rails.cache.fetch("user_#{id}_posts", expires_in: 5.minutes) do
      posts.published.recent.limit(10).to_a  # .to_a สำคัญ!
    end
  end
end

# HTTP Cache with Rack::Cache
# Gemfile: gem 'rack-cache'
# config/application.rb: config.action_dispatch.rack_cache = true

# Conditional GET
def show
  @article = Article.find(params[:id])

  if stale?(etag: @article, last_modified: @article.updated_at)
    render json: @article  # 200
  end
  # 304 Not Modified ถ้า cache ยังใช้ได้
end
```

---

### ข้อ 60: Multi-Database Setup

```ruby
# database.yml
primary:
  adapter:  postgresql
  database: myapp_primary

analytics:
  adapter:  postgresql
  database: myapp_analytics

# Model
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
end

class AnalyticsRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :analytics, reading: :analytics }
end

class Event < AnalyticsRecord
  # uses analytics database
end

# Read replicas
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: {
    writing: :primary,
    reading: :primary_replica
  }
end

# Switching databases
ActiveRecord::Base.connected_to(role: :reading) do
  User.all  # ใช้ read replica
end

# Horizontal sharding
class OrderRecord < ApplicationRecord
  connects_to shards: {
    shard_one: { writing: :shard_one },
    shard_two: { writing: :shard_two }
  }
end

ActiveRecord::Base.connected_to(shard: :shard_one) do
  Order.where(user_id: 1..5000)
end
```

---

## Coding Challenges

### Challenge 1: FizzBuzz
```ruby
(1..100).each do |n|
  if n % 15 == 0
    puts "FizzBuzz"
  elsif n % 3 == 0
    puts "Fizz"
  elsif n % 5 == 0
    puts "Buzz"
  else
    puts n
  end
end

# One-liner
(1..100).map { |n| (fb = [["Fizz", 3], ["Buzz", 5]].map { |s, m| s if n % m == 0 }.compact.join).empty? ? n : fb }
```

### Challenge 2: Palindrome
```ruby
def palindrome?(str)
  cleaned = str.downcase.gsub(/[^a-z0-9]/, '')
  cleaned == cleaned.reverse
end
```

### Challenge 3: Two Sum
```ruby
def two_sum(nums, target)
  seen = {}
  nums.each_with_index do |num, i|
    complement = target - num
    return [seen[complement], i] if seen.key?(complement)
    seen[num] = i
  end
  nil
end
```

---

## สรุป

ตารางระดับคำถาม:

| ระดับ | หัวข้อ | จำนวน |
|-------|--------|-------|
| Junior Ruby | Basics, OOP, Collections | 1-10 |
| Mid Ruby | Closures, Metaprogramming, Concurrency | 11-25 |
| Senior Ruby | GC, Fibers, Performance | 26-30 |
| Junior Rails | MVC, AR Basics, Routing | 31-40 |
| Mid Rails | Services, Jobs, Auth | 41-50 |
| Senior Rails | Performance, Security, Architecture | 51-60 |

เตรียมตัวเพิ่มเติม:
1. ฝึก System Design สำหรับ Senior level
2. ทำ LeetCode ใน Ruby
3. อ่าน Rails Guides ทั้งหมด
4. ทำ side project จริง
