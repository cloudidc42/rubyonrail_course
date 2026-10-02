# ตอนที่ 26: Functional Programming ใน Ruby (ขั้นตอนที่ 566-585)

Functional Programming (FP) เป็นแนวทางการเขียนโปรแกรมที่เน้น Functions เป็นหลัก หลีกเลี่ยง Mutable State และ Side Effects Ruby ไม่ใช่ Pure Functional Language แต่รองรับ FP Concepts ได้หลายอย่าง

---

## ขั้นตอนที่ 566: Pure Functions

### นิยาม Pure Function

Pure Function คือ Function ที่:
1. **Deterministic** - Input เดิมให้ Output เดิมเสมอ
2. **No Side Effects** - ไม่เปลี่ยนแปลงสิ่งภายนอก

```ruby
# Impure Function (มี Side Effect)
total = 0

def add_to_total(n)  # Impure - depend on external state
  total += n
end

# Pure Function
def add(a, b)
  a + b  # ขึ้นกับ Input เท่านั้น
end

# Impure: depend on External State
def greet_user  # Impure - depend on Time
  hour = Time.now.hour
  if hour < 12
    "อรุณสวัสดิ์"
  elsif hour < 18
    "สวัสดีตอนบ่าย"
  else
    "สวัสดีตอนเย็น"
  end
end

# Pure Version: รับ Time เป็น Parameter
def greet(hour)
  if hour < 12
    "อรุณสวัสดิ์"
  elsif hour < 18
    "สวัสดีตอนบ่าย"
  else
    "สวัสดีตอนเย็น"
  end
end

# Pure Functions ทดสอบง่าย
def calculate_tax(income, rate)
  income * rate
end

def format_currency(amount, symbol = '฿')
  "#{symbol}#{format('%.2f', amount)}"
end

# Composable!
tax    = calculate_tax(50_000, 0.07)
result = format_currency(tax)
puts result  # ฿3500.00

# Pure Collection Operations
def filter_adults(people)
  people.select { |p| p[:age] >= 18 }
end

def get_names(people)
  people.map { |p| p[:name] }
end

def sort_by_name(names)
  names.sort
end

people = [
  { name: 'สมชาย',   age: 25 },
  { name: 'สมหญิง',  age: 16 },
  { name: 'มานี',    age: 30 },
  { name: 'ปิ่นโต',  age: 15 }
]

adult_names = sort_by_name(get_names(filter_adults(people)))
puts adult_names.inspect  # ["มานี", "สมชาย"]
```

---

## ขั้นตอนที่ 567: Immutability

### Immutable Objects ใน Ruby

```ruby
# freeze - ทำให้ Object Immutable
name = "สมชาย".freeze
# name << " ดีใจ"  # FrozenError!

config = {
  host: 'localhost',
  port: 5432
}.freeze

# config[:host] = 'production'  # FrozenError!

# Deep Freeze
def deep_freeze(obj)
  case obj
  when Hash
    obj.each_value { |v| deep_freeze(v) }
  when Array
    obj.each { |v| deep_freeze(v) }
  end
  obj.freeze
end

nested = deep_freeze({
  user: {
    name: 'สมชาย',
    roles: ['admin', 'user']
  }
})

# ทุกระดับ Frozen

# frozen_string_literal
# # frozen_string_literal: true

# เพิ่ม Magic Comment นี้บนสุดของไฟล์
# ทำให้ String ทั้งหมดใน File เป็น Frozen โดยอัตโนมัติ

# Immutable Value Objects
class Money
  include Comparable
  
  attr_reader :amount, :currency
  
  def initialize(amount, currency = 'THB')
    @amount   = amount.freeze
    @currency = currency.freeze
    freeze
  end
  
  def +(other)
    raise TypeError, "Currencies must match" unless same_currency?(other)
    Money.new(@amount + other.amount, @currency)
  end
  
  def -(other)
    raise TypeError, "Currencies must match" unless same_currency?(other)
    Money.new(@amount - other.amount, @currency)
  end
  
  def *(factor)
    Money.new(@amount * factor, @currency)
  end
  
  def /(divisor)
    raise ArgumentError, "Cannot divide by zero" if divisor.zero?
    Money.new(@amount / divisor.to_f, @currency)
  end
  
  def <=>(other)
    return nil unless same_currency?(other)
    @amount <=> other.amount
  end
  
  def to_s
    "#{@currency} #{format('%.2f', @amount)}"
  end
  
  def ==(other)
    other.is_a?(Money) && @amount == other.amount && @currency == other.currency
  end
  
  private
  
  def same_currency?(other)
    @currency == other.currency
  end
end

price   = Money.new(100)
tax     = Money.new(7)
total   = price + tax
puts total  # THB 107.00

# price ยังคงเดิม
puts price  # THB 100.00
```

---

## ขั้นตอนที่ 568: Function Composition

### Method ที่เกี่ยวข้อง

```ruby
# Ruby 2.6+ มี >> และ << operators สำหรับ Proc/Method Composition

double   = ->(x) { x * 2 }
add_one  = ->(x) { x + 1 }
square   = ->(x) { x ** 2 }
to_string = ->(x) { x.to_s }

# >> : compose left to right
double_then_add = double >> add_one
puts double_then_add.call(5)  # 11 (5*2=10, 10+1=11)

# << : compose right to left
add_then_double = double << add_one
puts add_then_double.call(5)  # 12 (5+1=6, 6*2=12)

# Chain หลาย Function
pipeline = double >> add_one >> square >> to_string
puts pipeline.call(3)  # "49" (3*2=6, 6+1=7, 7**2=49)

# Method Composition
multiply_by_3 = method(:puts) << ->(x) { x * 3 }
multiply_by_3.call(4)  # prints 12

# Compose Class Methods
class Transform
  def self.upcase(str)
    str.upcase
  end
  
  def self.strip(str)
    str.strip
  end
  
  def self.reverse(str)
    str.reverse
  end
end

clean_and_upcase = method(:puts) <<
  Transform.method(:upcase) <<
  Transform.method(:strip)

clean_and_upcase.call("  hello world  ")  # HELLO WORLD
```

### ตัวอย่างจริงใน Rails

```ruby
# Validation Pipeline
module Validators
  def self.not_empty(value)
    raise ArgumentError, "ค่าว่าง" if value.nil? || value.empty?
    value
  end
  
  def self.minimum_length(min)
    ->(value) {
      raise ArgumentError, "ต้องมีอย่างน้อย #{min} ตัวอักษร" if value.length < min
      value
    }
  end
  
  def self.valid_email(value)
    raise ArgumentError, "Email ไม่ถูกต้อง" unless value.match?(/\A[^@\s]+@[^@\s]+\z/)
    value
  end
  
  def self.normalize(value)
    value.downcase.strip
  end
end

validate_email = 
  Validators.method(:not_empty) >>
  Validators.method(:normalize) >>
  Validators.method(:valid_email)

begin
  result = validate_email.call("  TEST@EXAMPLE.COM  ")
  puts result  # "test@example.com"
rescue ArgumentError => e
  puts "Validation Error: #{e.message}"
end
```

---

## ขั้นตอนที่ 569: Currying

### Currying ใน Ruby

```ruby
# Curry แปลง Function (a, b, c) → (a)(b)(c)
# คือการ "Partially Apply" Arguments

# Lambda แบบปกติ
add = ->(a, b) { a + b }
puts add.call(2, 3)  # 5

# Curried Version
curried_add = add.curry
puts curried_add.call(2).call(3)  # 5

# Partial Application
add5 = curried_add.call(5)  # รับ b แล้วจะทำ 5 + b
puts add5.call(3)   # 8
puts add5.call(10)  # 15
puts add5.call(7)   # 12

# ตัวอย่างจริง: URL Builder
build_url = ->(scheme, host, path) {
  "#{scheme}://#{host}/#{path}"
}.curry

https_builder = build_url.call('https')
api_builder   = https_builder.call('api.example.com')

user_url  = api_builder.call('users')
order_url = api_builder.call('orders')

puts user_url   # https://api.example.com/users
puts order_url  # https://api.example.com/orders

# Currying สำหรับ Filtering
greater_than = ->(threshold, value) { value > threshold }.curry

greater_than_10 = greater_than.call(10)
greater_than_50 = greater_than.call(50)

numbers = [5, 15, 25, 35, 45, 55, 65]

puts numbers.select(&greater_than_10).inspect  # [15, 25, 35, 45, 55, 65]
puts numbers.select(&greater_than_50).inspect  # [55, 65]

# Currying กับ Method
multiply = method(:*).to_proc.curry  # ไม่ work ตรงๆ แต่ทำได้แบบนี้

multiply = ->(a, b) { a * b }.curry
double   = multiply.call(2)
triple   = multiply.call(3)

puts [1, 2, 3, 4, 5].map(&double).inspect  # [2, 4, 6, 8, 10]
puts [1, 2, 3, 4, 5].map(&triple).inspect  # [3, 6, 9, 12, 15]
```

---

## ขั้นตอนที่ 570: Memoization

### Memoization Pattern

```ruby
# Memoization = Cache ผลการคำนวณ

# ไม่มี Memoization: คำนวณซ้ำ
def fibonacci(n)
  return n if n <= 1
  fibonacci(n - 1) + fibonacci(n - 2)
end

# ช้ามาก
start = Time.now
puts fibonacci(35)
puts "Time: #{Time.now - start:.3f}s"

# มี Memoization: Cache ผล
def fibonacci_memo(n, cache = {})
  return n if n <= 1
  cache[n] ||= fibonacci_memo(n - 1, cache) + fibonacci_memo(n - 2, cache)
end

start = Time.now
puts fibonacci_memo(35)
puts "Time: #{Time.now - start:.6f}s"  # เร็วกว่ามาก

# Memoization ด้วย ||=
class DataProcessor
  def initialize(data_source)
    @data_source = data_source
  end
  
  def expensive_analysis
    @analysis ||= begin
      puts "กำลัง Analyze... (ทำครั้งเดียว)"
      @data_source.map { |x| x ** 2 }.sum
    end
  end
  
  def more_expensive_report
    @report ||= build_report
  end
  
  private
  
  def build_report
    puts "กำลังสร้าง Report... (ทำครั้งเดียว)"
    {
      sum:     @data_source.sum,
      average: @data_source.sum.to_f / @data_source.size,
      min:     @data_source.min,
      max:     @data_source.max
    }
  end
end

processor = DataProcessor.new((1..1000).to_a)
puts processor.expensive_analysis  # คำนวณครั้งแรก
puts processor.expensive_analysis  # ใช้ Cache
puts processor.expensive_analysis  # ใช้ Cache
```

### Memoize Module

```ruby
module Memoizable
  def memoize(method_name)
    original_method = instance_method(method_name)
    cache_var       = :"@#{method_name}_cache"
    
    define_method(method_name) do |*args|
      cache     = instance_variable_get(cache_var) || {}
      cache_key = args
      
      unless cache.key?(cache_key)
        cache[cache_key] = original_method.bind(self).call(*args)
        instance_variable_set(cache_var, cache)
      end
      
      cache[cache_key]
    end
  end
end

class Calculator
  extend Memoizable
  
  def expensive_calc(n, m)
    puts "Computing #{n}, #{m}..."
    sleep(0.5)
    n * m + n + m
  end
  
  memoize :expensive_calc
end

calc = Calculator.new

puts calc.expensive_calc(5, 3)  # Computing... → 23
puts calc.expensive_calc(5, 3)  # ใช้ Cache → 23 (ไม่ Computing)
puts calc.expensive_calc(5, 4)  # Computing... → 25 (args ต่างกัน)
```

---

## ขั้นตอนที่ 571: Maybe/Option Pattern (Monads)

### Maybe Pattern

```ruby
# Maybe Pattern ป้องกัน nil errors
class Maybe
  def self.of(value)
    value.nil? ? Nothing.new : Just.new(value)
  end
  
  def nothing?
    is_a?(Nothing)
  end
  
  def just?
    is_a?(Just)
  end
end

class Just < Maybe
  attr_reader :value
  
  def initialize(value)
    @value = value
  end
  
  def map
    result = yield(@value)
    Maybe.of(result)
  end
  
  def flat_map
    yield(@value)
  end
  
  def or_else(_default)
    @value
  end
  
  def to_s
    "Just(#{@value})"
  end
end

class Nothing < Maybe
  def map
    self  # ไม่ทำอะไร
  end
  
  def flat_map
    self
  end
  
  def or_else(default)
    default
  end
  
  def to_s
    "Nothing"
  end
end

# การใช้งาน
def find_user(id)
  users = {
    1 => { name: 'สมชาย', email: 'somchai@example.com' },
    2 => { name: 'สมหญิง', email: 'somying@example.com' }
  }
  Maybe.of(users[id])
end

def get_email_domain(email)
  Maybe.of(email.split('@')[1])
end

# แบบเดิม (มี nil checks)
user = find_user(1)
if user.just?
  email = user.value[:email]
  domain = get_email_domain(email)
  if domain.just?
    puts domain.value
  end
end

# แบบ Functional (Chained)
result = find_user(1)
  .map { |user| user[:email] }
  .flat_map { |email| get_email_domain(email) }
  .or_else('unknown')

puts result  # "example.com"

# ถ้าไม่พบ User
result = find_user(999)
  .map { |user| user[:email] }
  .flat_map { |email| get_email_domain(email) }
  .or_else('unknown')

puts result  # "unknown"
```

---

## ขั้นตอนที่ 572: Pipeline Pattern

### Pipeline สำหรับ Data Transformation

```ruby
# Pipeline Pattern: ข้อมูลไหลผ่าน Series of Transformations

class Pipeline
  def initialize(*steps)
    @steps = steps
  end
  
  def self.[](*steps)
    new(*steps)
  end
  
  def call(input)
    @steps.reduce(input) do |result, step|
      step.call(result)
    end
  end
  
  def >>(other_step)
    Pipeline.new(*@steps, other_step)
  end
  
  alias_method :run, :call
end

# Data Processing Pipeline
normalize  = ->(data) { data.map { |s| s.strip.downcase } }
filter     = ->(data) { data.reject(&:empty?) }
sort       = ->(data) { data.sort }
unique     = ->(data) { data.uniq }
capitalize = ->(data) { data.map(&:capitalize) }

process = Pipeline[normalize, filter, sort, unique, capitalize]

input = ["  สมชาย ", "มานี", " สมชาย", "", "  ปิ่นโต  ", "มานี "]
result = process.call(input)
puts result.inspect

# Pipeline สำหรับ Text Processing
downcase     = ->(text) { text.downcase }
strip_html   = ->(text) { text.gsub(/<[^>]+>/, '') }
normalize_ws = ->(text) { text.gsub(/\s+/, ' ').strip }
truncate200  = ->(text) { text.length > 200 ? "#{text[0...197]}..." : text }

clean_text = Pipeline[downcase, strip_html, normalize_ws, truncate200]

html = "<p>  สวัสดี <b>โลก</b>   </p>"
puts clean_text.call(html)  # "สวัสดี โลก"
```

### Method Chaining Pipeline

```ruby
# Chainable Builder Pattern
class DataProcessor
  def initialize(data)
    @data = data.dup
  end
  
  def self.from(data)
    new(data)
  end
  
  def filter(&predicate)
    @data = @data.select(&predicate)
    self
  end
  
  def transform(&block)
    @data = @data.map(&block)
    self
  end
  
  def sort_by_key(key)
    @data = @data.sort_by { |item| item[key] }
    self
  end
  
  def take(n)
    @data = @data.first(n)
    self
  end
  
  def group_by_key(key)
    @data = @data.group_by { |item| item[key] }
    self
  end
  
  def result
    @data
  end
  
  def to_json
    require 'json'
    @data.to_json
  end
end

# การใช้งาน
products = [
  { name: 'A', price: 100, category: 'electronics', in_stock: true },
  { name: 'B', price: 50,  category: 'clothing',    in_stock: false },
  { name: 'C', price: 200, category: 'electronics', in_stock: true },
  { name: 'D', price: 75,  category: 'food',        in_stock: true },
  { name: 'E', price: 150, category: 'electronics', in_stock: true }
]

result = DataProcessor
  .from(products)
  .filter { |p| p[:in_stock] }
  .filter { |p| p[:category] == 'electronics' }
  .sort_by_key(:price)
  .transform { |p| p.merge(discounted_price: p[:price] * 0.9) }
  .take(3)
  .result

result.each do |p|
  puts "#{p[:name]}: #{p[:price]} -> #{p[:discounted_price]}"
end
```

---

## ขั้นตอนที่ 573-575: dry-rb Gems

### dry-types

```ruby
# Gemfile: gem 'dry-types'
require 'dry-types'

module Types
  include Dry.Types()
  
  # Basic Types
  Integer = Strict::Integer
  String  = Strict::String
  Bool    = Strict::Bool
  
  # Coercible Types (แปลง Type อัตโนมัติ)
  CoercibleInteger = Coercible::Integer
  CoercibleString  = Coercible::String
  
  # Custom Constrained Types
  Email         = String.constrained(format: /\A[^@\s]+@[^@\s]+\z/)
  PositiveInt   = Integer.constrained(gt: 0)
  Age           = Integer.constrained(gteq: 0, lteq: 150)
  Currency      = String.enum('THB', 'USD', 'EUR', 'SGD')
  
  # Optional Types
  NilableString  = String.optional
  NilableInteger = Integer.optional
  
  # Default Values
  PageNumber = Integer.default(1).constrained(gt: 0)
  
  # Array of Types
  StringArray   = Array.of(String)
  IntegerArray  = Array.of(Integer)
  EmailArray    = Array.of(Email)
end

# การใช้งาน
Types::PositiveInt.(5)       # 5
# Types::PositiveInt.(-1)    # Error!

Types::Email.('test@example.com')  # "test@example.com"
# Types::Email.('invalid')          # Error!

Types::Age.(25)    # 25
# Types::Age.(200)  # Error!

# Coercion
Types::CoercibleInteger.('42')  # 42 (String → Integer)
Types::CoercibleString.(123)    # "123"
```

### dry-struct

```ruby
# Gemfile: gem 'dry-struct'
require 'dry-struct'
require 'dry-types'

module Types
  include Dry.Types()
end

# Immutable Value Objects
class UserAddress < Dry::Struct
  attribute :street,   Types::String
  attribute :city,     Types::String
  attribute :province, Types::String
  attribute :postcode, Types::String.constrained(format: /\A\d{5}\z/)
  attribute :country,  Types::String.default('Thailand')
  
  def full_address
    "#{street}, #{city}, #{province} #{postcode}, #{country}"
  end
end

class User < Dry::Struct
  attribute :id,      Types::Integer.optional
  attribute :name,    Types::String
  attribute :email,   Types::String.constrained(format: /\A[^@\s]+@[^@\s]+\z/)
  attribute :age,     Types::Integer.constrained(gteq: 0)
  attribute :address, UserAddress.optional.default(nil)
  attribute :tags,    Types::Array.of(Types::String).default([].freeze)
  
  def adult?
    age >= 18
  end
end

# สร้าง User
user = User.new(
  id:    1,
  name:  'สมชาย ดีใจ',
  email: 'somchai@example.com',
  age:   25,
  address: UserAddress.new(
    street:   '123/4 ถ.สุขุมวิท',
    city:     'กรุงเทพฯ',
    province: 'กรุงเทพมหานคร',
    postcode: '10110'
  )
)

puts user.name
puts user.adult?
puts user.address.full_address

# Immutable - ไม่สามารถ Modify ได้
new_user = user.new(age: 26)  # สร้างใหม่พร้อมแก้ไข
puts new_user.age  # 26
puts user.age      # 25 (ไม่เปลี่ยน)
```

### dry-validation

```ruby
# Gemfile: gem 'dry-validation'
require 'dry-validation'

class UserContract < Dry::Validation::Contract
  params do
    required(:name).filled(:string)
    required(:email).filled(:string)
    required(:age).filled(:integer)
    optional(:phone).maybe(:string)
    
    required(:address).hash do
      required(:city).filled(:string)
      required(:postcode).filled(:string)
    end
  end
  
  rule(:name) do
    key.failure('ชื่อต้องมีอย่างน้อย 2 ตัวอักษร') if value.length < 2
    key.failure('ชื่อต้องไม่เกิน 100 ตัวอักษร')   if value.length > 100
  end
  
  rule(:email) do
    key.failure('Email ไม่ถูกต้อง') unless value.match?(/\A[^@\s]+@[^@\s]+\z/)
  end
  
  rule(:age) do
    key.failure('อายุต้องมากกว่า 0')   if value < 0
    key.failure('อายุต้องน้อยกว่า 150') if value > 150
  end
  
  rule(:phone) do
    next unless value
    key.failure('เบอร์โทรศัพท์ไม่ถูกต้อง') unless value.match?(/\A0\d{9}\z/)
  end
end

# การใช้งาน
contract = UserContract.new

# Valid Data
result = contract.call(
  name:  'สมชาย ดีใจ',
  email: 'somchai@example.com',
  age:   25,
  address: { city: 'กรุงเทพฯ', postcode: '10110' }
)

puts result.success?  # true
puts result.to_h

# Invalid Data
result = contract.call(
  name:  'ส',  # Too short
  email: 'not-an-email',
  age:   200,  # Too old
  address: { city: 'กรุงเทพฯ', postcode: '10110' }
)

puts result.success?  # false
puts result.errors.to_h
# { name: ["ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"], 
#   email: ["Email ไม่ถูกต้อง"], 
#   age: ["อายุต้องน้อยกว่า 150"] }
```

---

## ขั้นตอนที่ 576-580: Functional Patterns

### Functor Pattern

```ruby
# Functor: อะไรก็ตามที่ respond_to map
class Container
  attr_reader :value
  
  def initialize(value)
    @value = value
  end
  
  def map(&block)
    Container.new(block.call(@value))
  end
  
  def apply(container_of_function)
    function = container_of_function.value
    Container.new(function.call(@value))
  end
  
  def to_s
    "Container(#{@value})"
  end
end

c = Container.new(5)
puts c.map { |x| x * 2 }   # Container(10)
puts c.map { |x| x + 1 }   # Container(6)
puts c.map { |x| x.to_s }  # Container("5")

# Chain
result = Container.new(10)
  .map { |x| x * 2 }
  .map { |x| x + 5 }
  .map { |x| "result: #{x}" }

puts result  # Container("result: 25")
```

### Lazy Evaluation

```ruby
# Lazy Enumerable
natural_numbers = (1..Float::INFINITY).lazy

first_5_evens = natural_numbers
  .select { |n| n.even? }
  .first(5)
puts first_5_evens.inspect  # [2, 4, 6, 8, 10]

# Lazy Pipeline
result = (1..Float::INFINITY).lazy
  .select { |n| n % 3 == 0 }   # Multiples of 3
  .map { |n| n ** 2 }            # Square
  .reject { |n| n % 2 == 0 }    # Odd squares
  .first(5)
puts result.inspect

# Lazy File Processing
def process_large_file(filename)
  File.each_line(filename)
    .lazy
    .map(&:chomp)
    .reject(&:empty?)
    .select { |line| line.start_with?('ERROR') }
    .map { |line| parse_error_line(line) }
    .first(100)
end

# ประหยัด Memory เพราะไม่โหลดทั้งไฟล์

# Custom Lazy Enumerator
class InfiniteSequence
  include Enumerable
  
  def initialize(start: 0, step: 1)
    @start = start
    @step  = step
  end
  
  def each
    return to_enum unless block_given?
    current = @start
    loop do
      yield current
      current += @step
    end
  end
  
  # Force lazy
  def lazy_take(n)
    lazy.first(n)
  end
end

seq = InfiniteSequence.new(start: 1, step: 2)  # Odd numbers
puts seq.lazy_take(5).inspect  # [1, 3, 5, 7, 9]
puts seq.lazy.select { |n| n % 3 == 0 }.first(5).inspect  # [3, 9, 15, 21, 27]
```

### Monadic Error Handling

```ruby
# Result Type (Railway Oriented Programming)
class Result
  attr_reader :value, :error
  
  def self.success(value)
    new(value: value, success: true)
  end
  
  def self.failure(error)
    new(error: error, success: false)
  end
  
  def initialize(value: nil, error: nil, success:)
    @value   = value
    @error   = error
    @success = success
  end
  
  def success?
    @success
  end
  
  def failure?
    !@success
  end
  
  def map
    return self if failure?
    begin
      Result.success(yield(@value))
    rescue => e
      Result.failure(e.message)
    end
  end
  
  def flat_map
    return self if failure?
    begin
      yield(@value)
    rescue => e
      Result.failure(e.message)
    end
  end
  
  def on_success(&block)
    block.call(@value) if success?
    self
  end
  
  def on_failure(&block)
    block.call(@error) if failure?
    self
  end
  
  def or_else(default)
    success? ? @value : default
  end
  
  def to_s
    success? ? "Success(#{@value})" : "Failure(#{@error})"
  end
end

# การใช้งาน
def validate_age(age)
  if age.is_a?(Integer) && age > 0 && age < 150
    Result.success(age)
  else
    Result.failure("อายุไม่ถูกต้อง: #{age}")
  end
end

def validate_email(email)
  if email.match?(/\A[^@\s]+@[^@\s]+\z/)
    Result.success(email.downcase)
  else
    Result.failure("Email ไม่ถูกต้อง: #{email}")
  end
end

def create_user(name, email, age)
  validate_email(email)
    .flat_map { |valid_email| 
      validate_age(age).map { |valid_age| 
        { name: name, email: valid_email, age: valid_age }
      }
    }
end

# Success case
result = create_user('สมชาย', 'somchai@example.com', 25)
result
  .on_success { |user| puts "สร้าง User สำเร็จ: #{user[:name]}" }
  .on_failure { |error| puts "Error: #{error}" }

# Failure case
result = create_user('สมชาย', 'invalid-email', 25)
result
  .on_success { |user| puts "สร้าง User สำเร็จ" }
  .on_failure { |error| puts "Error: #{error}" }
```

---

## แบบฝึกหัดบทที่ 26 (20 ข้อ)

**ข้อ 1:** เขียน Pure Functions สำหรับ String Transformations (normalize, slugify, truncate)

**ข้อ 2:** Implement Immutable `Point` Class (x, y) ที่ supports translate, scale, rotate operations

**ข้อ 3:** สร้าง Function Composition Pipeline สำหรับ Data Cleaning

**ข้อ 4:** ใช้ Currying สร้าง Collection of Validation Functions

**ข้อ 5:** Implement Memoization สำหรับ Expensive Algorithm (e.g., Longest Common Subsequence)

**ข้อ 6:** สร้าง Maybe Monad สำหรับ Null-safe Database Queries

**ข้อ 7:** สร้าง Pipeline Pattern สำหรับ ETL (Extract, Transform, Load) Process

**ข้อ 8:** Implement Lazy Infinite Sequence สำหรับ Prime Numbers

**ข้อ 9:** ใช้ dry-types สร้าง Type-safe Domain Objects

**ข้อ 10:** Implement Result Type สำหรับ Error Handling ใน Service Objects

**ข้อ 11:** สร้าง Functor สำหรับ Tree Data Structure

**ข้อ 12:** ใช้ Currying และ Composition สำหรับ Query Builder

**ข้อ 13:** Implement Transducer Pattern สำหรับ Data Transformation

**ข้อ 14:** สร้าง Monad Chain สำหรับ User Registration Validation

**ข้อ 15:** ใช้ Lazy Evaluation ประมวลผล Large Dataset

**ข้อ 16:** Implement Partial Application สำหรับ HTTP Request Builder

**ข้อ 17:** สร้าง Immutable State Management (คล้าย Redux) ใน Ruby

**ข้อ 18:** ใช้ dry-validation สร้าง Complex Validation Rules

**ข้อ 19:** Implement Fold/Reduce Pattern สำหรับ Tree Traversal

**ข้อ 20:** สร้าง Complete FP-style Service Layer สำหรับ E-commerce Order Processing

---

### เฉลยตัวอย่าง ข้อ 17: Immutable State Management

```ruby
# Redux-like State Management ใน Ruby

class Action
  attr_reader :type, :payload
  
  def initialize(type, payload = {})
    @type    = type.to_sym
    @payload = payload.freeze
    freeze
  end
  
  def to_s
    "Action(#{@type}, #{@payload})"
  end
end

class Store
  attr_reader :state
  
  def initialize(reducer, initial_state)
    @reducer    = reducer
    @state      = initial_state.freeze
    @listeners  = []
    @dispatching = false
  end
  
  def dispatch(action)
    raise "Cannot dispatch while dispatching" if @dispatching
    
    @dispatching = true
    @state = @reducer.call(@state, action).freeze
    @dispatching = false
    
    @listeners.each { |listener| listener.call(@state) }
    
    action
  end
  
  def subscribe(&listener)
    @listeners << listener
    -> { @listeners.delete(listener) }  # Return unsubscribe function
  end
  
  def get_state
    @state
  end
end

# Actions
class CartActions
  ADD_ITEM    = :ADD_ITEM
  REMOVE_ITEM = :REMOVE_ITEM
  CLEAR_CART  = :CLEAR_CART
  
  def self.add_item(item)
    Action.new(ADD_ITEM, { item: item })
  end
  
  def self.remove_item(item_id)
    Action.new(REMOVE_ITEM, { item_id: item_id })
  end
  
  def self.clear
    Action.new(CLEAR_CART)
  end
end

# Reducer (Pure Function)
cart_reducer = ->(state, action) {
  case action.type
  when CartActions::ADD_ITEM
    item  = action.payload[:item]
    items = state[:items].dup
    
    existing = items.find { |i| i[:id] == item[:id] }
    if existing
      items = items.map do |i|
        i[:id] == item[:id] ? i.merge(quantity: i[:quantity] + 1) : i
      end
    else
      items << item.merge(quantity: 1)
    end
    
    state.merge(
      items: items.freeze,
      total: items.sum { |i| i[:price] * i[:quantity] }
    )
    
  when CartActions::REMOVE_ITEM
    item_id = action.payload[:item_id]
    items   = state[:items].reject { |i| i[:id] == item_id }.freeze
    
    state.merge(
      items: items,
      total: items.sum { |i| i[:price] * i[:quantity] }
    )
    
  when CartActions::CLEAR_CART
    { items: [].freeze, total: 0 }
    
  else
    state
  end
}

# Setup Store
initial_state = { items: [].freeze, total: 0 }.freeze
store = Store.new(cart_reducer, initial_state)

# Subscribe to changes
unsubscribe = store.subscribe do |state|
  puts "Cart Updated: #{state[:items].size} items, Total: #{state[:total]}"
end

# Dispatch Actions
store.dispatch(CartActions.add_item({ id: 1, name: 'สินค้า A', price: 100 }))
store.dispatch(CartActions.add_item({ id: 2, name: 'สินค้า B', price: 200 }))
store.dispatch(CartActions.add_item({ id: 1, name: 'สินค้า A', price: 100 }))  # +1 qty

puts store.state.inspect

store.dispatch(CartActions.remove_item(1))
store.dispatch(CartActions.clear)

# Unsubscribe
unsubscribe.call
```

---

*สรุปบทที่ 26: Functional Programming ช่วยให้เขียนโค้ดที่ Predictable, Testable และ Composable มากขึ้น Pure Functions, Immutability, Function Composition และ Monads เป็น Concepts หลักที่นำมาประยุกต์ใช้ใน Ruby ได้ dry-rb ecosystem ช่วยให้ใช้ FP patterns ได้ง่ายขึ้นในโปรเจกต์จริง*
