# ตอนที่ 26: Functional Programming in Ruby (Steps 566-585)

## บทนำ

Functional Programming (FP) เป็นรูปแบบการเขียนโปรแกรมที่เน้นการใช้ฟังก์ชันเป็นหน่วยพื้นฐาน โดยพยายามหลีกเลี่ยงการเปลี่ยนแปลงสถานะ (state mutation) และข้อมูลที่เปลี่ยนแปลงได้ (mutable data) Ruby แม้จะเป็นภาษา Object-Oriented แต่ก็รองรับ Functional Programming ได้อย่างดีเยี่ยม

---

## Step 566: Functional Programming คืออะไร?

### แนวคิดหลักของ Functional Programming

FP มีหลักการสำคัญดังนี้:

1. **Pure Functions** - ฟังก์ชันที่คืนค่าเดิมเสมอสำหรับ input เดิม และไม่มี side effects
2. **Immutability** - ข้อมูลไม่เปลี่ยนแปลงหลังจากสร้างแล้ว
3. **First-class Functions** - ฟังก์ชันเป็น "first-class citizens" สามารถส่งผ่านเป็น argument หรือ return value ได้
4. **Higher-order Functions** - ฟังก์ชันที่รับหรือคืนฟังก์ชันอื่น
5. **Function Composition** - การรวมฟังก์ชันเล็กๆ เป็นฟังก์ชันใหญ่

### ทำไม Functional Programming ถึงสำคัญ?

```ruby
# แบบ Imperative (บอก "ยังไง")
numbers = [1, 2, 3, 4, 5]
result = []
numbers.each do |n|
  if n.even?
    result << n * 2
  end
end
puts result.inspect  # => [4, 8]

# แบบ Functional (บอก "อะไร")
result = [1, 2, 3, 4, 5]
  .select(&:even?)
  .map { |n| n * 2 }
puts result.inspect  # => [4, 8]
```

### ข้อดีของ Functional Programming

1. **ทดสอบง่าย** - Pure functions ทดสอบได้ง่ายเพราะผลลัพธ์คาดเดาได้
2. **Debug ง่าย** - ไม่มี shared state ทำให้หาสาเหตุ bug ง่ายขึ้น
3. **Concurrent-friendly** - ไม่มี race conditions จาก shared mutable state
4. **อ่านง่าย** - โค้ดบอกว่า "ทำอะไร" มากกว่า "ทำยังไง"

---

## Step 567: Pure Functions ใน Ruby

### Pure Function คืออะไร?

Pure Function มีคุณสมบัติ 2 อย่าง:
1. **Referential Transparency** - ผลลัพธ์เดิมเสมอสำหรับ input เดิม
2. **No Side Effects** - ไม่เปลี่ยนแปลงสถานะภายนอก

```ruby
# Pure Function - ดี
def add(a, b)
  a + b
end

add(2, 3)  # => 5 เสมอ
add(2, 3)  # => 5 เสมอ

# Impure Function - ไม่ดี
$total = 0

def add_to_total(n)
  $total += n  # เปลี่ยนแปลง global state
  $total
end

add_to_total(5)  # => 5
add_to_total(5)  # => 10 (ผลลัพธ์ต่างกัน!)
```

### ตัวอย่าง Pure Functions ใน Ruby

```ruby
# Pure - ดี
def double(n)
  n * 2
end

def greet(name)
  "Hello, #{name}!"
end

def sum(numbers)
  numbers.reduce(0, :+)
end

def format_price(amount, currency = "THB")
  "#{currency} #{format('%.2f', amount)}"
end

# ทดสอบง่ายมาก
puts double(5)          # => 10
puts greet("สมชาย")    # => Hello, สมชาย!
puts sum([1, 2, 3])     # => 6
puts format_price(99.5) # => THB 99.50
```

### Side Effects ที่ควรหลีกเลี่ยง

```ruby
# Bad: แก้ไข array ที่ส่งมา
def double_values!(arr)
  arr.map! { |n| n * 2 }  # เปลี่ยน original array
end

numbers = [1, 2, 3]
double_values!(numbers)
puts numbers.inspect  # => [2, 4, 6] - numbers เปลี่ยนไป!

# Good: สร้าง array ใหม่
def double_values(arr)
  arr.map { |n| n * 2 }  # คืน array ใหม่
end

numbers = [1, 2, 3]
doubled = double_values(numbers)
puts numbers.inspect  # => [1, 2, 3] - ไม่เปลี่ยน
puts doubled.inspect  # => [2, 4, 6]
```

---

## Step 568: Immutability ใน Ruby

### freeze - ทำให้ Object ไม่เปลี่ยนแปลง

```ruby
# freeze string
str = "Hello".freeze
str << " World"  # => FrozenError: can't modify frozen String

# freeze array
arr = [1, 2, 3].freeze
arr << 4         # => FrozenError: can't modify frozen Array
arr[0] = 10      # => FrozenError: can't modify frozen Array

# freeze hash
hash = { name: "Ruby" }.freeze
hash[:version] = 3  # => FrozenError: can't modify frozen Hash

# ตรวจสอบว่า frozen หรือไม่
puts str.frozen?   # => true
puts "Hello".frozen?  # => false
```

### ข้อควรระวัง - Shallow Freeze

```ruby
# freeze ทำได้แค่ shallow
outer = [[1, 2], [3, 4]].freeze
outer << [5, 6]      # => FrozenError
outer[0] << 99       # => ได้! inner array ยังไม่ frozen
puts outer.inspect   # => [[1, 2, 99], [3, 4]]

# Deep freeze
def deep_freeze(obj)
  case obj
  when Array
    obj.each { |item| deep_freeze(item) }
    obj.freeze
  when Hash
    obj.each_value { |v| deep_freeze(v) }
    obj.freeze
  else
    obj.freeze
  end
end

data = [[1, 2], [3, 4]]
deep_freeze(data)
data[0] << 99  # => FrozenError
```

### dup - คัดลอก Object (unfrozen)

```ruby
original = "Hello".freeze
copy = original.dup
puts copy.frozen?  # => false

copy << " World"
puts copy     # => Hello World
puts original # => Hello (ไม่เปลี่ยน)

# dup vs clone
frozen_str = "Hello".freeze
duped = frozen_str.dup    # => ไม่ frozen
cloned = frozen_str.clone # => ยัง frozen!

puts duped.frozen?   # => false
puts cloned.frozen?  # => true
```

### frozen_string_literal

```ruby
# frozen_string_literal: true

# เพิ่ม magic comment นี้ที่ต้นไฟล์เพื่อ freeze strings ทั้งหมด
# ช่วย performance และป้องกัน accidental mutation

str = "Hello"
str << " World"  # => FrozenError

# ใช้ String.new หรือ +str สำหรับ mutable string
mutable = +"Hello"  # unary + operator
mutable << " World"
puts mutable  # => Hello World
```

### Value Objects pattern

```ruby
# Immutable Value Object
class Money
  attr_reader :amount, :currency

  def initialize(amount, currency = "THB")
    @amount = amount.freeze
    @currency = currency.freeze
    freeze  # freeze ตัวเอง
  end

  def +(other)
    raise ArgumentError, "ต้อง currency เดียวกัน" unless currency == other.currency
    Money.new(amount + other.amount, currency)  # สร้าง object ใหม่
  end

  def *(factor)
    Money.new(amount * factor, currency)
  end

  def to_s
    "#{currency} #{format('%.2f', amount)}"
  end
end

price = Money.new(100, "THB")
tax = Money.new(7, "THB")
total = price + tax
puts total  # => THB 107.00

# price ไม่เปลี่ยน
puts price  # => THB 100.00
```

---

## Step 569: Higher-order Functions

### Functions as First-class Citizens

```ruby
# เก็บ method ใน variable
greet = method(:puts)
greet.call("Hello!")  # => Hello!

# Lambda
double = ->(x) { x * 2 }
puts double.call(5)   # => 10
puts double.(5)       # => 10 (shorthand)
puts double[5]        # => 10 (array-style)

# Proc
triple = Proc.new { |x| x * 3 }
puts triple.call(5)   # => 15
```

### ส่งฟังก์ชันเป็น Argument

```ruby
# Map, Select, Reduce รับ block
numbers = [1, 2, 3, 4, 5]

# ส่ง block
doubled = numbers.map { |n| n * 2 }

# ส่ง lambda
double_fn = ->(n) { n * 2 }
doubled = numbers.map(&double_fn)
puts doubled.inspect  # => [2, 4, 6, 8, 10]

# ส่ง method reference
puts numbers.map(&method(:puts))

# Custom higher-order function
def apply_twice(fn, value)
  fn.call(fn.call(value))
end

add_ten = ->(n) { n + 10 }
puts apply_twice(add_ten, 5)  # => 25

# Higher-order ที่ return function
def multiplier(factor)
  ->(n) { n * factor }
end

double = multiplier(2)
triple = multiplier(3)
puts double.call(5)   # => 10
puts triple.call(5)   # => 15
```

### Enumerable Methods - Higher-order ที่ใช้บ่อย

```ruby
students = [
  { name: "สมชาย", grade: 85, subject: "คณิต" },
  { name: "สมหญิง", grade: 92, subject: "ภาษาไทย" },
  { name: "สมศรี", grade: 78, subject: "คณิต" },
  { name: "สมบัติ", grade: 95, subject: "ภาษาไทย" }
]

# select/filter
math_students = students.select { |s| s[:subject] == "คณิต" }
puts math_students.map { |s| s[:name] }.inspect
# => ["สมชาย", "สมศรี"]

# map/transform
names = students.map { |s| s[:name] }
puts names.inspect

# reduce/fold
total_grade = students.reduce(0) { |sum, s| sum + s[:grade] }
avg = total_grade.to_f / students.size
puts "เกรดเฉลี่ย: #{avg}"  # => เกรดเฉลี่ย: 87.5

# group_by
by_subject = students.group_by { |s| s[:subject] }
by_subject.each do |subject, group|
  avg = group.sum { |s| s[:grade] }.to_f / group.size
  puts "#{subject}: #{avg}"
end

# sort_by
sorted = students.sort_by { |s| -s[:grade] }
puts sorted.first[:name]  # => สมบัติ
```

---

## Step 570: Function Composition

### การรวมฟังก์ชัน

```ruby
# Compose ด้วยตนเอง
def compose(f, g)
  ->(x) { f.call(g.call(x)) }
end

double = ->(x) { x * 2 }
add_one = ->(x) { x + 1 }

double_then_add = compose(add_one, double)
puts double_then_add.call(5)  # => 11 (5*2=10, 10+1=11)

add_then_double = compose(double, add_one)
puts add_then_double.call(5)  # => 12 (5+1=6, 6*2=12)
```

### Ruby 2.6+ Proc Composition Operators

```ruby
# >> (left to right)
double = ->(x) { x * 2 }
add_one = ->(x) { x + 1 }
square = ->(x) { x ** 2 }

pipeline = double >> add_one >> square
puts pipeline.call(3)  # => 49 (3*2=6, 6+1=7, 7**2=49)

# << (right to left)  
pipeline2 = square << add_one << double
puts pipeline2.call(3)  # => 49 (เหมือนกัน แต่อ่านจากขวาไปซ้าย)

# ใช้กับ Method objects
upcase = :upcase.to_proc
strip = :strip.to_proc
# method(:puts)

process = strip >> upcase
puts process.call("  hello world  ")  # => HELLO WORLD
```

### Compose ฟังก์ชัน Data Processing

```ruby
# Data transformation pipeline
parse_csv_line = ->(line) { line.split(",").map(&:strip) }
to_hash = ->(fields) { { name: fields[0], age: fields[1].to_i, city: fields[2] } }
validate = ->(person) { person[:age] > 0 ? person : nil }

process_line = parse_csv_line >> to_hash >> validate

data = [
  "สมชาย, 25, กรุงเทพ",
  "สมหญิง, -1, เชียงใหม่",  # invalid
  "สมศรี, 30, ภูเก็ต"
]

results = data.map(&process_line).compact
puts results.inspect
```

---

## Step 571: Currying ใน Ruby

### Curry คืออะไร?

Currying คือการแปลงฟังก์ชันที่รับหลาย argument เป็นฟังก์ชันที่รับ argument ทีละตัว

```ruby
# ฟังก์ชันปกติ
add = ->(a, b) { a + b }
puts add.call(2, 3)  # => 5

# Curried version
curried_add = add.curry
add_5 = curried_add.call(5)  # partial application
puts add_5.call(3)   # => 8
puts add_5.call(10)  # => 15

# เรียกทันที
puts curried_add.call(2).call(3)  # => 5
puts curried_add.(2).(3)          # => 5
```

### Partial Application

```ruby
# ตัวอย่างจริง
multiply = ->(a, b) { a * b }
double = multiply.curry.(2)   # partial application
triple = multiply.curry.(3)

[1, 2, 3, 4, 5].map(&double)  # => [2, 4, 6, 8, 10]
[1, 2, 3, 4, 5].map(&triple)  # => [3, 6, 9, 12, 15]

# Currying ด้วย method
def power(base, exp)
  base ** exp
end

square = method(:power).curry.(2)   # หมายถึง base=2, exp=?
# ไม่ถูกต้อง - ต้องระวัง argument order

# แก้โดยเปลี่ยน order
def power_of(exp, base)
  base ** exp
end

square = method(:power_of).curry.(2)
cube   = method(:power_of).curry.(3)

puts [2, 3, 4].map(&square).inspect  # => [4, 9, 16]
puts [2, 3, 4].map(&cube).inspect    # => [8, 27, 64]
```

### Currying ใน Real-world

```ruby
# Validation functions
validate_range = ->(min, max, value) {
  value >= min && value <= max
}

valid_age = validate_range.curry.(0).(150)
valid_score = validate_range.curry.(0).(100)

puts valid_age.(25)    # => true
puts valid_age.(200)   # => false
puts valid_score.(85)  # => true
puts valid_score.(105) # => false

# Filtering
people = [
  { name: "สมชาย", age: 25 },
  { name: "สมหญิง", age: 200 },
  { name: "เด็ก", age: 5 }
]

is_valid_age = ->(person) { valid_age.(person[:age]) }
valid_people = people.select(&is_valid_age)
puts valid_people.map { |p| p[:name] }.inspect
# => ["สมชาย"]
```

---

## Step 572: Memoization Pattern

### Memoization คืออะไร?

Memoization คือการ cache ผลลัพธ์ของฟังก์ชัน เพื่อไม่ต้องคำนวณซ้ำสำหรับ input เดิม

```ruby
# ไม่มี Memoization - ช้า
def fibonacci(n)
  return n if n <= 1
  fibonacci(n - 1) + fibonacci(n - 2)
end

# fibonacci(40) ช้ามาก!

# มี Memoization - เร็ว
def fibonacci_memo(n, cache = {})
  return cache[n] if cache.key?(n)
  return n if n <= 1
  cache[n] = fibonacci_memo(n - 1, cache) + fibonacci_memo(n - 2, cache)
end

require 'benchmark'
Benchmark.bm do |x|
  x.report("ไม่มี memo:") { fibonacci(35) }
  x.report("มี memo:   ") { fibonacci_memo(35) }
end
```

### Memoization ใน Instance Methods

```ruby
class ExpensiveCalculator
  def initialize(data)
    @data = data
    @cache = {}
  end

  def complex_result
    @complex_result ||= begin
      # คำนวณที่ใช้เวลานาน
      sleep(0.1)  # จำลองการคำนวณ
      @data.sum * 42
    end
  end

  # Memoize ที่รับ argument
  def calculate(n)
    @cache[n] ||= expensive_operation(n)
  end

  private

  def expensive_operation(n)
    n ** 2 + n * 3 + 1
  end
end

calc = ExpensiveCalculator.new([1, 2, 3])
puts calc.complex_result  # คำนวณครั้งแรก (ช้า)
puts calc.complex_result  # ใช้ cache (เร็ว)
```

### Memoize Module

```ruby
module Memoizable
  def memoize(method_name)
    original_method = instance_method(method_name)
    cache_var = "@_memo_#{method_name}"

    define_method(method_name) do |*args|
      cache = instance_variable_get(cache_var) || {}
      unless cache.key?(args)
        cache[args] = original_method.bind(self).call(*args)
        instance_variable_set(cache_var, cache)
      end
      cache[args]
    end
  end
end

class Fibonacci
  extend Memoizable

  def fib(n)
    return n if n <= 1
    fib(n - 1) + fib(n - 2)
  end

  memoize :fib
end

f = Fibonacci.new
puts f.fib(50)  # เร็วมาก!
```

---

## Step 573: Monads (Maybe/Option Pattern)

### ปัญหาของ nil

```ruby
# โค้ดที่เสี่ยง NoMethodError
def get_user_city(user_id)
  user = find_user(user_id)
  user.address.city.upcase  # จะ error ถ้า user, address, หรือ city เป็น nil!
end

# แก้ด้วย nil check ธรรมดา (verbose)
def get_user_city_safe(user_id)
  user = find_user(user_id)
  return nil unless user
  return nil unless user.address
  return nil unless user.address.city
  user.address.city.upcase
end

# Ruby &. operator (Safe Navigation)
def get_user_city_modern(user_id)
  find_user(user_id)&.address&.city&.upcase
end
```

### Maybe Monad

```ruby
class Maybe
  attr_reader :value

  def initialize(value)
    @value = value
  end

  def self.of(value)
    value.nil? ? Nothing.new : Just.new(value)
  end

  def map
    raise NotImplementedError
  end

  def flat_map
    raise NotImplementedError
  end

  def get_or_else(default)
    raise NotImplementedError
  end
end

class Just < Maybe
  def map
    result = yield(@value)
    Maybe.of(result)
  end

  def flat_map
    yield(@value)
  end

  def get_or_else(_default)
    @value
  end

  def to_s
    "Just(#{@value})"
  end

  def some?
    true
  end

  def none?
    false
  end
end

class Nothing < Maybe
  def map
    self  # ไม่ทำอะไร
  end

  def flat_map
    self  # ไม่ทำอะไร
  end

  def get_or_else(default)
    default
  end

  def to_s
    "Nothing"
  end

  def some?
    false
  end

  def none?
    true
  end
end

# ใช้งาน
result = Maybe.of("hello")
  .map { |s| s.upcase }
  .map { |s| "#{s}!" }
  .get_or_else("ไม่มีค่า")
puts result  # => HELLO!

result = Maybe.of(nil)
  .map { |s| s.upcase }   # ไม่ execute
  .map { |s| "#{s}!" }    # ไม่ execute
  .get_or_else("ไม่มีค่า")
puts result  # => ไม่มีค่า
```

### Result Monad (Either)

```ruby
class Result
  def self.ok(value)
    Ok.new(value)
  end

  def self.err(error)
    Err.new(error)
  end
end

class Ok < Result
  attr_reader :value

  def initialize(value)
    @value = value
  end

  def map
    Result.ok(yield(@value))
  rescue => e
    Result.err(e.message)
  end

  def flat_map
    yield(@value)
  end

  def on_success
    yield(@value)
    self
  end

  def on_failure
    self
  end

  def ok?; true; end
  def err?; false; end
  def to_s; "Ok(#{@value})"; end
end

class Err < Result
  attr_reader :error

  def initialize(error)
    @error = error
  end

  def map
    self
  end

  def flat_map
    self
  end

  def on_success
    self
  end

  def on_failure
    yield(@error)
    self
  end

  def ok?; false; end
  def err?; true; end
  def to_s; "Err(#{@error})"; end
end

# ตัวอย่างใช้งาน
def parse_age(str)
  age = Integer(str)
  age > 0 ? Result.ok(age) : Result.err("อายุต้องมากกว่า 0")
rescue ArgumentError
  Result.err("ไม่ใช่ตัวเลข")
end

def validate_adult(age)
  age >= 18 ? Result.ok(age) : Result.err("ต้องอายุ 18 ปีขึ้นไป")
end

result = parse_age("25")
  .flat_map { |age| validate_adult(age) }
  .on_success { |age| puts "อายุ #{age} ปี - ผ่าน!" }
  .on_failure { |err| puts "Error: #{err}" }

result = parse_age("15")
  .flat_map { |age| validate_adult(age) }
  .on_success { |age| puts "ผ่าน!" }
  .on_failure { |err| puts "Error: #{err}" }
# => Error: ต้องอายุ 18 ปีขึ้นไป
```

---

## Step 574: Pipeline Operator Pattern

### สร้าง Pipeline ด้วย then/yield_self

```ruby
# Ruby 2.6+ มี then และ yield_self
result = "  hello world  "
  .then { |s| s.strip }
  .then { |s| s.split }
  .then { |words| words.map(&:capitalize) }
  .then { |words| words.join(" ") }

puts result  # => Hello World

# แบบสั้นกว่า
result = "  hello world  "
  .strip
  .split
  .map(&:capitalize)
  .join(" ")

puts result  # => Hello World
```

### Custom Pipeline

```ruby
# สร้าง Pipeline class
class Pipeline
  def initialize(value)
    @value = value
    @steps = []
  end

  def self.of(value)
    new(value)
  end

  def pipe(&block)
    @steps << block
    self
  end

  def execute
    @steps.reduce(@value) { |val, step| step.call(val) }
  end
end

result = Pipeline.of([1, 2, 3, 4, 5, 6])
  .pipe { |arr| arr.select(&:even?) }
  .pipe { |arr| arr.map { |n| n ** 2 } }
  .pipe { |arr| arr.reduce(:+) }
  .execute

puts result  # => 56 (4+16+36)

# Processing pipeline สำหรับ text
text_pipeline = Pipeline.of("  Ruby Programming is Fun!  ")
  .pipe { |s| s.strip }
  .pipe { |s| s.downcase }
  .pipe { |s| s.gsub(/[^a-z\s]/, '') }
  .pipe { |s| s.split }
  .pipe { |words| words.uniq }
  .execute

puts text_pipeline.inspect
# => ["ruby", "programming", "is", "fun"]
```

### Composable Pipeline Steps

```ruby
module PipelineSteps
  STRIP = ->(s) { s.strip }
  UPCASE = ->(s) { s.upcase }
  DOWNCASE = ->(s) { s.downcase }
  WORDS = ->(s) { s.split }
  JOIN_SPACE = ->(words) { words.join(" ") }
  CAPITALIZE_EACH = ->(words) { words.map(&:capitalize) }
  REMOVE_DUPLICATES = ->(arr) { arr.uniq }
  SORT = ->(arr) { arr.sort }
end

# Compose pipeline
title_case = PipelineSteps::STRIP >>
             PipelineSteps::DOWNCASE >>
             PipelineSteps::WORDS >>
             PipelineSteps::CAPITALIZE_EACH >>
             PipelineSteps::JOIN_SPACE

puts title_case.call("  hello world from ruby  ")
# => Hello World From Ruby
```

---

## Step 575: dry-rb Gems Overview

### dry-types

```ruby
# Gemfile
# gem 'dry-types'

require 'dry-types'

module Types
  include Dry.Types()
end

# Basic types
integer = Types::Integer
puts integer.(42)     # => 42
puts integer.("42")   # => 42 (coercion)
# integer.("hello")  # => Dry::Types::CoercionError

# Strict types
strict_integer = Types::Strict::Integer
strict_integer.(42)     # => 42
# strict_integer.("42") # => Dry::Types::ConstraintError

# Coercible types
coercible_int = Types::Coercible::Integer
puts coercible_int.("42")  # => 42

# Optional types
maybe_string = Types::Maybe::String
puts maybe_string.(nil).inspect    # => None
puts maybe_string.("hi").inspect   # => Some("hi")

# Custom types
Age = Types::Coercible::Integer.constrained(gt: 0, lt: 150)
Age.(25)   # => 25
# Age.(-1) # => Dry::Types::ConstraintError
```

### dry-validation

```ruby
# gem 'dry-validation'

require 'dry-validation'

class UserContract < Dry::Validation::Contract
  params do
    required(:name).filled(:string)
    required(:age).filled(:integer, gt?: 0)
    required(:email).filled(:string)
    optional(:phone).maybe(:string)
  end

  rule(:email) do
    unless /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i.match?(value)
      key.failure("รูปแบบ email ไม่ถูกต้อง")
    end
  end

  rule(:age) do
    key.failure("ต้องอายุ 18 ปีขึ้นไป") if value < 18
  end
end

contract = UserContract.new

# Valid input
result = contract.call(
  name: "สมชาย",
  age: 25,
  email: "somchai@example.com"
)
puts result.success?  # => true
puts result.values.to_h.inspect

# Invalid input
result = contract.call(
  name: "",
  age: 15,
  email: "invalid-email"
)
puts result.failure?  # => true
puts result.errors.to_h.inspect
# => {:name=>["must be filled"], :age=>["ต้องอายุ 18 ปีขึ้นไป"], :email=>["รูปแบบ email ไม่ถูกต้อง"]}
```

### dry-monads

```ruby
# gem 'dry-monads'

require 'dry-monads'

class UserService
  include Dry::Monads[:result, :maybe]

  def find_user(id)
    user = User.find_by(id: id)
    if user
      Success(user)
    else
      Failure("ไม่พบผู้ใช้ id: #{id}")
    end
  end

  def update_user(id, params)
    find_user(id).bind do |user|
      if user.update(params)
        Success(user)
      else
        Failure(user.errors.full_messages)
      end
    end
  end

  def process_payment(user_id, amount)
    find_user(user_id)
      .bind { |user| validate_balance(user, amount) }
      .bind { |user| charge_user(user, amount) }
      .bind { |transaction| send_receipt(transaction) }
  end

  private

  def validate_balance(user, amount)
    if user.balance >= amount
      Success(user)
    else
      Failure("ยอดเงินไม่เพียงพอ")
    end
  end
end
```

---

## Step 576: Functional Patterns ใน Real Rails Apps

### Service Objects แบบ Functional

```ruby
# app/services/create_order_service.rb
class CreateOrderService
  include Dry::Monads[:result]

  def call(user, cart, payment_info)
    Success({ user: user, cart: cart, payment: payment_info })
      .bind { |data| validate_cart(data) }
      .bind { |data| calculate_totals(data) }
      .bind { |data| process_payment(data) }
      .bind { |data| create_order(data) }
      .bind { |data| send_confirmation(data) }
  end

  private

  def validate_cart(data)
    cart = data[:cart]
    if cart.empty?
      Failure("ตะกร้าสินค้าว่างเปล่า")
    elsif cart.items.any? { |item| item.out_of_stock? }
      Failure("สินค้าบางรายการหมดสต็อก")
    else
      Success(data)
    end
  end

  def calculate_totals(data)
    cart = data[:cart]
    subtotal = cart.items.sum { |item| item.price * item.quantity }
    tax = subtotal * 0.07
    total = subtotal + tax
    Success(data.merge(subtotal: subtotal, tax: tax, total: total))
  end

  def process_payment(data)
    result = PaymentGateway.charge(
      amount: data[:total],
      card: data[:payment]
    )
    if result.success?
      Success(data.merge(transaction_id: result.transaction_id))
    else
      Failure("การชำระเงินล้มเหลว: #{result.error}")
    end
  end

  def create_order(data)
    order = Order.create!(
      user: data[:user],
      total: data[:total],
      transaction_id: data[:transaction_id]
    )
    Success(data.merge(order: order))
  rescue ActiveRecord::RecordInvalid => e
    Failure("สร้าง order ไม่สำเร็จ: #{e.message}")
  end

  def send_confirmation(data)
    OrderMailer.confirmation(data[:order]).deliver_later
    Success(data[:order])
  end
end

# ใช้งาน
result = CreateOrderService.new.call(current_user, @cart, payment_params)

case result
in Success(order)
  redirect_to order_path(order), notice: "สั่งซื้อสำเร็จ!"
in Failure(error)
  render :new, alert: error
end
```

### Immutable Data Transfer Objects

```ruby
# Data Transfer Objects (DTOs)
OrderDTO = Data.define(:id, :user_id, :items, :total, :status)

# Ruby 3.2+ Data class (immutable)
order_dto = OrderDTO.new(
  id: 1,
  user_id: 42,
  items: [{ product: "Ruby Book", qty: 2, price: 350 }],
  total: 700,
  status: "pending"
)

# ไม่สามารถเปลี่ยนค่าได้
# order_dto.status = "paid"  # => NoMethodError

# สร้างใหม่แทน
paid_order = order_dto.with(status: "paid")
puts paid_order.status   # => paid
puts order_dto.status    # => pending (ไม่เปลี่ยน)
```

---

## Step 577-585: Exercises และ Advanced Topics

### Lazy Evaluation

```ruby
# Lazy enumerators
natural_numbers = (1..Float::INFINITY).lazy

# คำนวณเฉพาะที่ต้องการ
result = natural_numbers
  .select { |n| n % 2 == 0 }  # เลขคู่
  .map { |n| n ** 2 }           # ยกกำลัง 2
  .first(5)                     # เอา 5 ตัวแรก

puts result.inspect  # => [4, 16, 36, 64, 100]

# ไม่ lazy - จะวนลูปไม่สิ้นสุด!
# (1..Float::INFINITY).select { |n| n.even? }.map { |n| n**2 }.first(5)
```

### Functional Error Handling

```ruby
# ใช้ rescue ร่วมกับ functional style
def safe_divide(a, b)
  raise ArgumentError, "หารด้วย 0 ไม่ได้" if b.zero?
  a / b.to_f
end

def try_divide(a, b)
  Result.ok(safe_divide(a, b))
rescue ArgumentError => e
  Result.err(e.message)
end

puts try_divide(10, 2)   # => Ok(5.0)
puts try_divide(10, 0)   # => Err(หารด้วย 0 ไม่ได้)
```

---

## แบบฝึกหัด 20 ข้อ

### ข้อที่ 1: Pure Functions
เขียน pure function `calculate_tax(price, tax_rate)` ที่คืนราคาหลังภาษี

**เฉลย:**
```ruby
def calculate_tax(price, tax_rate)
  price * (1 + tax_rate / 100.0)
end

puts calculate_tax(100, 7)    # => 107.0
puts calculate_tax(200, 10)   # => 220.0
```

### ข้อที่ 2: Immutability
สร้าง Point class ที่ immutable พร้อม method `translate(dx, dy)` ที่คืน Point ใหม่

**เฉลย:**
```ruby
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
    freeze
  end

  def translate(dx, dy)
    Point.new(x + dx, y + dy)
  end

  def distance_to(other)
    Math.sqrt((x - other.x) ** 2 + (y - other.y) ** 2)
  end

  def to_s
    "(#{x}, #{y})"
  end
end

p1 = Point.new(0, 0)
p2 = p1.translate(3, 4)
puts p1      # => (0, 0)
puts p2      # => (3, 4)
puts p1.distance_to(p2)  # => 5.0
```

### ข้อที่ 3: Higher-order Functions
เขียนฟังก์ชัน `compose(*fns)` ที่ compose ฟังก์ชันหลายๆ ตัว

**เฉลย:**
```ruby
def compose(*fns)
  ->(x) { fns.reverse.reduce(x) { |acc, fn| fn.call(acc) } }
end

upcase = ->(s) { s.upcase }
exclaim = ->(s) { "#{s}!" }
greet = ->(s) { "Hello, #{s}" }

process = compose(exclaim, upcase, greet)
puts process.call("ruby")  # => "Hello, RUBY!"
```

### ข้อที่ 4: Currying
สร้าง curried function สำหรับตรวจสอบ string format

**เฉลย:**
```ruby
matches_pattern = ->(pattern, string) {
  pattern.match?(string)
}

is_email = matches_pattern.curry.(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
is_phone = matches_pattern.curry.(/\A0[0-9]{9}\z/)

emails = ["test@gmail.com", "invalid", "user@example.org"]
puts emails.select(&is_email).inspect
# => ["test@gmail.com", "user@example.org"]

phones = ["0812345678", "123", "0999999999"]
puts phones.select(&is_phone).inspect
# => ["0812345678", "0999999999"]
```

### ข้อที่ 5: Memoization
เขียน `memoize` decorator สำหรับ method

**เฉลย:**
```ruby
def memoize_method(klass, method_name)
  original = klass.instance_method(method_name)
  klass.send(:define_method, method_name) do |*args|
    @_cache ||= {}
    key = [method_name, args]
    @_cache[key] ||= original.bind(self).call(*args)
  end
end

class Calculator
  def heavy_computation(n)
    sleep(0.01)  # simulate slow computation
    n * n + n * 2 + 1
  end
end

memoize_method(Calculator, :heavy_computation)

calc = Calculator.new
puts calc.heavy_computation(5)   # slow first time
puts calc.heavy_computation(5)   # fast from cache
```

### ข้อที่ 6: Maybe Monad
ใช้ Maybe monad สำหรับ safe navigation ใน user profile

**เฉลย:**
```ruby
User = Struct.new(:name, :address)
Address = Struct.new(:city, :country)

def get_user_country(user_id)
  users = {
    1 => User.new("สมชาย", Address.new("กรุงเทพ", "ไทย")),
    2 => User.new("สมหญิง", nil),
    3 => nil
  }

  Maybe.of(users[user_id])
    .map { |u| u.address }
    .map { |a| a.country }
    .get_or_else("ไม่ทราบประเทศ")
end

puts get_user_country(1)  # => ไทย
puts get_user_country(2)  # => ไม่ทราบประเทศ
puts get_user_country(3)  # => ไม่ทราบประเทศ
```

### ข้อที่ 7: Function Composition ด้วย >>

**เฉลย:**
```ruby
normalize_text = method(:strip).to_proc     # ไม่ได้ใน Ruby โดยตรง

# ใช้ lambda แทน
strip_spaces = ->(s) { s.strip }
to_lowercase = ->(s) { s.downcase }
remove_special = ->(s) { s.gsub(/[^a-zA-Z0-9\s]/, '') }
split_words = ->(s) { s.split }
sort_words = ->(arr) { arr.sort }
join_comma = ->(arr) { arr.join(", ") }

normalize = strip_spaces >> to_lowercase >> remove_special >>
            split_words >> sort_words >> join_comma

puts normalize.call("  Hello, World! Ruby is Great!  ")
# => "great, hello, is, ruby, world"
```

### ข้อที่ 8: Result Monad สำหรับ Form Validation

**เฉลย:**
```ruby
def validate_username(name)
  return Result.err("ชื่อต้องมีอย่างน้อย 3 ตัวอักษร") if name.length < 3
  return Result.err("ชื่อต้องไม่เกิน 20 ตัวอักษร") if name.length > 20
  return Result.err("ชื่อต้องมีเฉพาะตัวอักษรและตัวเลข") unless name.match?(/\A[\w]+\z/)
  Result.ok(name)
end

def validate_password(password)
  return Result.err("รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร") if password.length < 8
  return Result.err("ต้องมีตัวเลขด้วย") unless password.match?(/\d/)
  return Result.err("ต้องมีตัวพิมพ์ใหญ่") unless password.match?(/[A-Z]/)
  Result.ok(password)
end

# Chain validations
def register(username, password)
  validate_username(username)
    .flat_map { validate_password(password) }
    .map { "ลงทะเบียน #{username} สำเร็จ!" }
end

puts register("ab", "password123A")  # Err
puts register("somchai", "weak")     # Err
puts register("somchai", "Strong1")  # Ok
```

### ข้อที่ 9: Lazy Evaluation สำหรับ Infinite Sequences

**เฉลย:**
```ruby
# Fibonacci sequence แบบ lazy
fibs = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

# เอา Fibonacci ที่ < 1000
puts fibs.lazy.select { |n| n < 1000 }.to_a.inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610, 987]

# เอา 10 ตัวแรกที่เป็นเลขคู่
puts fibs.lazy.select { |n| n.even? }.first(10).inspect
```

### ข้อที่ 10: Pipeline สำหรับ Data Processing

**เฉลย:**
```ruby
sales_data = [
  { product: "สินค้า A", amount: 1500, month: "Jan", region: "North" },
  { product: "สินค้า B", amount: 2000, month: "Jan", region: "South" },
  { product: "สินค้า A", amount: 1800, month: "Feb", region: "North" },
  { product: "สินค้า C", amount: 900,  month: "Feb", region: "East" },
  { product: "สินค้า B", amount: 2200, month: "Mar", region: "South" },
]

# Pipeline สำหรับวิเคราะห์ยอดขาย
report = sales_data
  .select { |s| s[:amount] > 1000 }            # filter
  .group_by { |s| s[:product] }               # group
  .transform_values { |sales|                 # transform
    {
      total: sales.sum { |s| s[:amount] },
      count: sales.length,
      avg: sales.sum { |s| s[:amount] } / sales.length.to_f
    }
  }
  .sort_by { |_, v| -v[:total] }              # sort by total desc
  .to_h

report.each do |product, stats|
  puts "#{product}: ยอดรวม #{stats[:total]}, ค่าเฉลี่ย #{stats[:avg].round(2)}"
end
```

### ข้อที่ 11-20: Advanced Exercises

**ข้อที่ 11:** สร้าง `Either` monad สำหรับ HTTP request handling
```ruby
# เฉลย
def fetch_user_data(url)
  response = HTTParty.get(url)
  case response.code
  when 200 then Result.ok(response.parsed_response)
  when 404 then Result.err("ไม่พบข้อมูล")
  when 500 then Result.err("Server error")
  else Result.err("Unknown error: #{response.code}")
  end
rescue => e
  Result.err("Network error: #{e.message}")
end
```

**ข้อที่ 12:** เขียน `memoize` สำหรับ pure functions ด้วย WeakRef
```ruby
# เฉลย
require 'weakref'

def weak_memoize(&block)
  cache = {}
  ->(input) {
    cache[input] ||= block.call(input)
    cache[input]
  }
end

expensive_fn = weak_memoize { |n| n * n * n }
puts expensive_fn.(5)  # => 125
puts expensive_fn.(5)  # => 125 (cached)
```

**ข้อที่ 13:** Compose validators แบบ functional
```ruby
# เฉลย
def all_valid(*validators)
  ->(value) {
    validators.reduce(Result.ok(value)) do |result, validator|
      result.flat_map { |v| validator.(v) }
    end
  }
end

not_empty = ->(s) { s.empty? ? Result.err("ต้องไม่ว่าง") : Result.ok(s) }
min_length = ->(n) { ->(s) { s.length >= n ? Result.ok(s) : Result.err("ต้องมีอย่างน้อย #{n} ตัว") } }
max_length = ->(n) { ->(s) { s.length <= n ? Result.ok(s) : Result.err("ต้องไม่เกิน #{n} ตัว") } }

validate_name = all_valid(not_empty, min_length.(2), max_length.(50))
puts validate_name.("สมชาย")  # Ok
puts validate_name.("")       # Err
```

**ข้อที่ 14:** สร้าง Functor สำหรับ transformation ของ data
```ruby
# เฉลย  
class Box
  def initialize(value)
    @value = value
  end

  def map
    Box.new(yield(@value))
  end

  def flat_map
    yield(@value)
  end

  def value
    @value
  end

  def to_s
    "Box(#{@value})"
  end
end

result = Box.new(5)
  .map { |n| n * 2 }
  .map { |n| n + 1 }
  .map { |n| "ผลลัพธ์: #{n}" }

puts result  # => Box(ผลลัพธ์: 11)
```

**ข้อที่ 15:** Trampolining สำหรับ deep recursion
```ruby
# เฉลย
def trampoline(fn)
  result = fn
  result = result.call while result.is_a?(Proc)
  result
end

# Factorial แบบ tail-recursive ด้วย trampoline
def fact_helper(n, acc)
  return acc if n <= 1
  -> { fact_helper(n - 1, n * acc) }
end

def factorial(n)
  trampoline(-> { fact_helper(n, 1) })
end

puts factorial(100)  # ไม่ stack overflow!
```

**ข้อที่ 16:** State monad สำหรับ counter
```ruby
# เฉลย
class State
  def initialize(&block)
    @run = block
  end

  def run(initial_state)
    @run.call(initial_state)
  end

  def self.get
    new { |s| [s, s] }
  end

  def self.put(new_state)
    new { |_| [nil, new_state] }
  end

  def flat_map
    State.new do |state|
      value, new_state = run(state)
      yield(value).run(new_state)
    end
  end

  def map
    flat_map { |v| State.new { |s| [yield(v), s] } }
  end
end

# ใช้งาน: counter
increment = State.get.flat_map { |n| State.put(n + 1) }
counter_program = increment.flat_map { increment }.flat_map { increment }
result, final_state = counter_program.run(0)
puts final_state  # => 3
```

**ข้อที่ 17:** Functional approach สำหรับ event sourcing
```ruby
# เฉลย
class EventStore
  def initialize
    @events = [].freeze
  end

  def append(event)
    EventStore.new.tap do |store|
      store.instance_variable_set(:@events, (@events + [event]).freeze)
    end
  end

  def replay(initial_state, reducer)
    @events.reduce(initial_state, &reducer)
  end

  def events
    @events
  end
end

# Bank account using event sourcing
REDUCER = ->(state, event) {
  case event[:type]
  when :deposit
    state.merge(balance: state[:balance] + event[:amount])
  when :withdrawal
    state.merge(balance: state[:balance] - event[:amount])
  end
}

store = EventStore.new
  .append({ type: :deposit, amount: 1000 })
  .append({ type: :deposit, amount: 500 })
  .append({ type: :withdrawal, amount: 200 })

final_state = store.replay({ balance: 0 }, REDUCER)
puts final_state  # => {:balance=>1300}
```

**ข้อที่ 18:** Partial application สำหรับ API calls
```ruby
# เฉลย
def api_call(base_url, endpoint, method, params)
  puts "#{method} #{base_url}#{endpoint} with #{params}"
end

make_api_call = method(:api_call).curry

# สร้าง specialized versions
github_api = make_api_call.("https://api.github.com")
github_get = github_api.("/users").("/repos").(:GET)  # ผิด - ต้องระวัง order

# ถูกต้อง
def http_request(method, base_url, endpoint, params = {})
  "#{method} #{base_url}#{endpoint} params=#{params}"
end

github_get = method(:http_request).curry.(:GET).("https://api.github.com")
puts github_get.("/users").({})          # => GET https://api.github.com/users params={}
puts github_get.("/repos/ruby/ruby").({}) # => GET https://api.github.com/repos/ruby/ruby params={}
```

**ข้อที่ 19:** สร้าง Observable/Stream แบบ functional
```ruby
# เฉลย
class Observable
  def initialize(&block)
    @subscribe = block
  end

  def self.from_array(arr)
    new do |observer|
      arr.each { |item| observer.call(item) }
    end
  end

  def map(&transform)
    Observable.new do |observer|
      @subscribe.call(->(item) { observer.call(transform.call(item)) })
    end
  end

  def select(&predicate)
    Observable.new do |observer|
      @subscribe.call(->(item) { observer.call(item) if predicate.call(item) })
    end
  end

  def subscribe(&observer)
    @subscribe.call(observer)
  end
end

# ใช้งาน
Observable.from_array([1, 2, 3, 4, 5])
  .select { |n| n.odd? }
  .map { |n| n * 10 }
  .subscribe { |n| puts n }
# => 10, 30, 50
```

**ข้อที่ 20:** สร้าง DSL แบบ functional สำหรับ validation rules
```ruby
# เฉลย
module Rules
  def self.required
    ->(value) {
      value.nil? || value.to_s.empty? ?
        Result.err("จำเป็น") : Result.ok(value)
    }
  end

  def self.min_length(n)
    ->(value) {
      value.to_s.length >= n ?
        Result.ok(value) : Result.err("ต้องมีอย่างน้อย #{n} ตัวอักษร")
    }
  end

  def self.matches(pattern, message)
    ->(value) {
      pattern.match?(value.to_s) ?
        Result.ok(value) : Result.err(message)
    }
  end

  def self.chain(*rules)
    ->(value) {
      rules.reduce(Result.ok(value)) do |result, rule|
        result.flat_map { |v| rule.(v) }
      end
    }
  end
end

username_rules = Rules.chain(
  Rules.required,
  Rules.min_length(3),
  Rules.matches(/\A[a-zA-Z0-9_]+\z/, "ใช้ได้เฉพาะ a-z, 0-9, _")
)

puts username_rules.("somchai")  # Ok
puts username_rules.("")         # Err: จำเป็น
puts username_rules.("ab")       # Err: ต้องมีอย่างน้อย 3 ตัวอักษร
puts username_rules.("hi there") # Err: ใช้ได้เฉพาะ a-z, 0-9, _
```

---

## สรุป

Functional Programming ใน Ruby ช่วยให้โค้ดของเรา:

1. **ทดสอบง่ายขึ้น** ด้วย pure functions ที่ไม่มี side effects
2. **อ่านง่ายขึ้น** ด้วย declarative style
3. **ปลอดภัยขึ้น** ด้วย immutability
4. **Reuse ได้มากขึ้น** ด้วย higher-order functions และ composition

แม้ Ruby จะไม่ใช่ภาษา functional แบบ pure แต่สามารถนำ functional patterns มาใช้ได้อย่างมีประสิทธิภาพ โดยเฉพาะเมื่อรวมกับ dry-rb gem family

---

*ต่อไป: ตอนที่ 27 - Performance Optimization*
