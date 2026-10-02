# ตอนที่ 10: Blocks, Procs, Lambdas (ขั้นตอนที่ 171-200)

## บทนำ

Blocks, Procs, และ Lambdas เป็นหัวใจของ functional programming ใน Ruby เป็น feature ที่ทำให้ Ruby มีความยืดหยุ่นสูงและมีพลังมาก เข้าใจ concept นี้จะช่วยให้เขียน Ruby ได้ดีขึ้นอย่างมาก

ในบทนี้เราจะเรียนรู้:
- Block คืออะไรและการใช้ yield
- block_given? 
- Proc.new และ proc {}
- Lambda vs Proc ความแตกต่างสำคัญ
- Closures และ scope
- Method objects
- Currying และ partial application
- Practical patterns
- Custom iterators
- Proc composition

---

## ขั้นตอนที่ 171: Block คืออะไร?

Block เป็น anonymous code snippet ที่ส่งให้กับ method ไม่ใช่ object แบบเต็ม ๆ (ต่างจาก Proc)

```ruby
# Block แบบ {} - single line
[1, 2, 3].each { |n| puts n }

# Block แบบ do...end - multi line
[1, 2, 3].each do |n|
  square = n ** 2
  puts "#{n}^2 = #{square}"
end

# Block ในชีวิตประจำวัน
# เปิดไฟล์
File.open("file.txt", "w") { |f| f.write("Hello") }

# วัดเวลา
require 'benchmark'
Benchmark.bm do |x|
  x.report("sort") { (1..1000).to_a.shuffle.sort }
end

# Transaction
# ActiveRecord::Base.transaction do
#   account1.withdraw(100)
#   account2.deposit(100)
# end

# Convention: ใช้ {} สำหรับ single line, do..end สำหรับ multi line
# (แต่ความแตกต่าง precedence ก็สำคัญ)
foo bar { |x| x }      # block ถูกส่งให้ bar
foo bar do |x| x end   # block ถูกส่งให้ foo
```

---

## ขั้นตอนที่ 172: yield Keyword

`yield` ใช้ใน method เพื่อ execute block ที่ส่งมา

```ruby
# method ที่รับ block ด้วย yield
def greet
  puts "ก่อน yield"
  yield   # execute block ที่ส่งมา
  puts "หลัง yield"
end

greet { puts "สวัสดี!" }
# => ก่อน yield
# => สวัสดี!
# => หลัง yield

# yield กับ arguments
def repeat(n)
  n.times { yield }
end

repeat(3) { puts "Hello" }
# => Hello x 3

# ส่งข้อมูลให้ block ผ่าน yield
def double_it(n)
  yield(n * 2)
end

double_it(5) { |result| puts result }   # => 10

# yield คืนค่าจาก block
def transform(n)
  result = yield(n)
  "Result: #{result}"
end

output = transform(5) { |n| n ** 2 }
puts output   # => Result: 25

# yield หลายครั้ง
def multi_yield
  yield(1)
  yield(2)
  yield(3)
end

multi_yield { |n| puts "Got #{n}" }
# => Got 1, Got 2, Got 3

# yield ใน loop
def each_pair(array)
  i = 0
  while i < array.length - 1
    yield array[i], array[i + 1]
    i += 2
  end
end

each_pair([1, 2, 3, 4, 5, 6]) do |a, b|
  puts "#{a} + #{b} = #{a + b}"
end
```

---

## ขั้นตอนที่ 173: block_given?

```ruby
# ตรวจสอบว่ามี block ส่งมาหรือไม่
def flexible
  if block_given?
    puts "มี block!"
    yield
  else
    puts "ไม่มี block"
  end
end

flexible             # => ไม่มี block
flexible { puts "block content" }  # => มี block! => block content

# ตัวอย่างจริง: method ที่ optional block
def log(message, level = :info)
  output = "[#{level.upcase}] #{message}"
  if block_given?
    output + "\n" + yield
  else
    output
  end
end

puts log("Server started")
puts log("Query executed") { "SELECT * FROM users" }

# Enumerator return pattern
def my_each(array)
  return to_enum(:my_each, array) unless block_given?
  array.each { |item| yield item }
end

# สามารถใช้แบบ block
my_each([1, 2, 3]) { |n| puts n }

# หรือแบบ Enumerator
enum = my_each([1, 2, 3])
puts enum.map { |n| n * 2 }.inspect   # => [2, 4, 6]

# Rails-style: return value ต่าง ๆ
def find_or_create(key, store = {})
  if store.key?(key)
    store[key]
  elsif block_given?
    store[key] = yield
  else
    nil
  end
end

cache = {}
value1 = find_or_create(:user, cache) { { name: "Alice" } }
value2 = find_or_create(:user, cache) { { name: "Bob" } }  # ไม่ execute block
puts value1.inspect   # => {:name=>"Alice"}
puts value2.inspect   # => {:name=>"Alice"} (cached)
```

---

## ขั้นตอนที่ 174: Explicit Block Parameter (&block)

```ruby
# &block - รับ block เป็น Proc object
def capture_block(&block)
  puts block.class     # => Proc
  puts block.call(5)   # execute the block
end

capture_block { |n| n * 2 }   # => Proc, 10

# เก็บ block ไว้ใช้ภายหลัง
class EventHandler
  def initialize
    @callbacks = []
  end

  def on_event(&callback)
    @callbacks << callback
    self
  end

  def trigger(event_data)
    @callbacks.each { |cb| cb.call(event_data) }
  end
end

handler = EventHandler.new
handler
  .on_event { |data| puts "Handler 1: #{data}" }
  .on_event { |data| puts "Handler 2: #{data.upcase}" }

handler.trigger("hello world")
# => Handler 1: hello world
# => Handler 2: HELLO WORLD

# ส่ง block ต่อ (&block)
def outer(&block)
  inner(&block)
end

def inner
  yield 42
end

outer { |n| puts n }   # => 42

# แปลง Proc เป็น block ด้วย &
multiplier = Proc.new { |n| n * 3 }
puts [1, 2, 3].map(&multiplier).inspect   # => [3, 6, 9]

# Symbol to Proc
puts ["hello", "world"].map(&:upcase).inspect   # => ["HELLO", "WORLD"]
# :upcase.to_proc เหมือน Proc.new { |x| x.upcase }
```

---

## ขั้นตอนที่ 175: Proc.new และ proc {}

Proc เป็น object ที่เก็บ block ได้

```ruby
# สร้าง Proc
p1 = Proc.new { |n| n * 2 }
p2 = proc { |n| n * 2 }   # shorthand

puts p1.call(5)   # => 10
puts p2.(5)       # => 10
puts p2[5]        # => 10

# Proc เป็น first-class object
def apply_twice(func, value)
  func.call(func.call(value))
end

double = proc { |n| n * 2 }
puts apply_twice(double, 3)   # => 12

# Proc สามารถเก็บใน variable, array, hash
operations = {
  double: proc { |n| n * 2 },
  square: proc { |n| n ** 2 },
  negate: proc { |n| -n }
}

puts operations[:double].call(5)   # => 10
puts operations[:square].call(4)   # => 16

# Proc ใน Array
transforms = [
  proc { |n| n + 1 },
  proc { |n| n * 2 },
  proc { |n| n ** 2 }
]

result = transforms.reduce(3) { |acc, t| t.call(acc) }
puts result   # => ((3+1)*2)^2 = 64

# Proc สามารถ store state (closure)
def make_counter(start = 0)
  count = start
  increment = proc { count += 1; count }
  decrement = proc { count -= 1; count }
  get = proc { count }
  [increment, decrement, get]
end

inc, dec, get = make_counter(10)
puts get.call    # => 10
puts inc.call    # => 11
puts inc.call    # => 12
puts dec.call    # => 11
puts get.call    # => 11
```

---

## ขั้นตอนที่ 176: Lambda

Lambda คือ Proc ชนิดพิเศษที่มีกฎเรื่อง arguments และ return ที่เข้มงวดกว่า

```ruby
# สร้าง Lambda
l1 = lambda { |n| n * 2 }
l2 = ->(n) { n * 2 }   # stabby lambda (Ruby 1.9+)

puts l1.call(5)   # => 10
puts l2.(5)       # => 10
puts l2[5]        # => 10

puts l1.class      # => Proc
puts l1.lambda?    # => true
puts proc {}.lambda?  # => false

# Lambda กับ multiple parameters
add = ->(a, b) { a + b }
puts add.(3, 4)   # => 7

# Lambda กับ default parameters
greet = ->(name, greeting = "สวัสดี") { "#{greeting} #{name}!" }
puts greet.("Alice")           # => สวัสดี Alice!
puts greet.("Bob", "Hello")   # => Hello Bob!

# Lambda กับ keyword arguments
create_point = ->(x:, y:, z: 0) { { x: x, y: y, z: z } }
puts create_point.(x: 1, y: 2).inspect          # => {:x=>1, :y=>2, :z=>0}
puts create_point.(x: 3, y: 4, z: 5).inspect   # => {:x=>3, :y=>4, :z=>5}

# Lambda เหมาะสำหรับ:
# 1. สร้าง callable objects
validators = {
  positive: ->(n) { n > 0 },
  even: ->(n) { n.even? },
  in_range: ->(n, min, max) { (min..max).include?(n) }
}

puts validators[:positive].call(5)              # => true
puts validators[:even].call(3)                  # => false
puts validators[:in_range].curry.call(5).call(1, 10)  # => true
```

---

## ขั้นตอนที่ 177: Lambda vs Proc - ความแตกต่างสำคัญ

```ruby
# ความแตกต่าง 1: Return behavior
# Lambda: return ออกจาก lambda เท่านั้น
# Proc: return ออกจาก method ที่ Proc อยู่

def test_lambda
  l = lambda { return "from lambda" }
  result = l.call
  "Method continues: #{result}"  # ยังทำงานต่อ
end

def test_proc
  p = proc { return "from proc" }
  p.call
  "Method continues"  # ไม่ถึงบรรทัดนี้!
end

puts test_lambda   # => Method continues: from lambda
puts test_proc     # => from proc (method หยุดที่ proc return)

# ความแตกต่าง 2: Argument checking
my_lambda = lambda { |a, b| a + b }
my_proc = proc { |a, b| [a, b] }

# Lambda strict
# my_lambda.call(1, 2, 3)   # ArgumentError: wrong number of arguments

# Proc lenient
puts my_proc.call(1, 2, 3).inspect   # => [1, 2] (extra args ignored)
puts my_proc.call(1).inspect          # => [1, nil] (missing args = nil)

# ความแตกต่าง 3: arity
puts my_lambda.arity   # => 2
puts my_proc.arity     # => 2 (แต่ behavior ต่างกัน)

# เมื่อไหรใช้ Lambda? เมื่อไหรใช้ Proc?
# Lambda: เมื่อต้องการ function ที่ independent (ส่งผ่านได้ปลอดภัย)
# Proc: เมื่อต้องการ code snippet ที่ทำงานในบริบทของ method
```

---

## ขั้นตอนที่ 178: Closures และ Scope

```ruby
# Closure - function ที่ capture environment ที่สร้างมัน
def make_adder(n)
  lambda { |x| x + n }  # capture n จาก outer scope
end

add5 = make_adder(5)
add10 = make_adder(10)

puts add5.(3)    # => 8
puts add10.(3)   # => 13

# Closure capture variables by reference (ไม่ใช่ by value)
x = 10
double_x = proc { x * 2 }

puts double_x.call   # => 20
x = 20
puts double_x.call   # => 40 (ค่า x เปลี่ยน!)

# สร้าง private state ด้วย closure
def make_bank_account(initial_balance)
  balance = initial_balance

  deposit = lambda { |amount| balance += amount; balance }
  withdraw = lambda do |amount|
    raise "Insufficient funds" if amount > balance
    balance -= amount
    balance
  end
  check = lambda { balance }

  { deposit: deposit, withdraw: withdraw, check: check }
end

account = make_bank_account(1000)
puts account[:check].call          # => 1000
puts account[:deposit].(500)       # => 1500
puts account[:withdraw].(200)      # => 1300
# puts account[:withdraw].(2000)   # => RuntimeError: Insufficient funds

# Binding - closure's scope
def get_binding(x)
  binding   # current binding
end

b = get_binding(42)
puts eval("x * 2", b)   # => 84
```

---

## ขั้นตอนที่ 179: Proc Composition ด้วย >> และ <<

```ruby
# >> (forward composition): f >> g = g(f(x))
# << (backward composition): f << g = f(g(x))

double = ->(n) { n * 2 }
increment = ->(n) { n + 1 }
square = ->(n) { n ** 2 }

# >> forward composition
double_then_inc = double >> increment
puts double_then_inc.(5)   # => 11 (5*2=10, 10+1=11)

inc_then_double = double << increment
puts inc_then_double.(5)   # => 12 (5+1=6, 6*2=12)

# Chain หลาย operations
pipeline = double >> increment >> square
puts pipeline.(3)   # => 49 (3*2=6, 6+1=7, 7^2=49)

# ตัวอย่างจริง
strip = :strip.to_proc
downcase = :downcase.to_proc
capitalize = :capitalize.to_proc

clean_name = strip >> downcase >> capitalize
puts clean_name.("  ALICE  ")   # => Alice

# ใช้กับ map
names = ["  ALICE  ", "  BOB  ", "  CAROL  "]
puts names.map(&clean_name).inspect
# => ["Alice", "Bob", "Carol"]

# Compose method objects
require 'ostruct'
to_open_struct = ->(h) { OpenStruct.new(h) }
get_name = ->(obj) { obj.name }

name_from_hash = to_open_struct >> get_name
puts name_from_hash.({ name: "Alice", age: 30 })   # => Alice
```

---

## ขั้นตอนที่ 180: Method#to_proc

```ruby
# Method objects สามารถแปลงเป็น Proc ได้ด้วย &
def double(n)
  n * 2
end

# แปลง method เป็น proc
double_proc = method(:double).to_proc
puts double_proc.call(5)   # => 10

# ใช้ & ใน argument
puts [1, 2, 3].map(&method(:double)).inspect   # => [2, 4, 6]

# Methods ของ Object
puts [1, -2, 3, -4].map(&method(:puts))   # => prints each

# Instance methods
class Converter
  def celsius_to_fahrenheit(c)
    c * 9.0 / 5 + 32
  end
end

converter = Converter.new
c_to_f = converter.method(:celsius_to_fahrenheit).to_proc

puts [0, 20, 37, 100].map(&c_to_f).inspect
# => [32.0, 68.0, 98.6, 212.0]

# UnboundMethod
class StringFormatter
  def shout
    upcase + "!!!"
  end
end

unbound_shout = StringFormatter.instance_method(:shout)
bound = unbound_shout.bind("hello")
puts bound.call   # => HELLO!!!

# ใช้ &:method pattern กับ custom classes
class Product
  attr_reader :name, :price
  
  def initialize(name, price)
    @name = name
    @price = price
  end

  def to_s
    "#{name}: ฿#{price}"
  end
end

products = [
  Product.new("Apple", 20),
  Product.new("Banana", 15),
  Product.new("Cherry", 50)
]

puts products.map(&:to_s).inspect
puts products.map(&:price).sum   # => 85
puts products.min_by(&:price).name  # => Banana
```

---

## ขั้นตอนที่ 181: Currying

Currying คือการแปลง function ที่รับ arguments หลายตัว ให้เป็น chain ของ functions ที่แต่ละอันรับ argument เดียว

```ruby
# Lambda currying
add = ->(a, b) { a + b }
curried_add = add.curry

add5 = curried_add.(5)   # partial application
puts add5.(3)    # => 8
puts add5.(10)   # => 15

# Proc currying
multiply = proc { |a, b| a * b }
double = multiply.curry.(2)
triple = multiply.curry.(3)

puts [1, 2, 3, 4, 5].map(&double).inspect   # => [2, 4, 6, 8, 10]
puts [1, 2, 3, 4, 5].map(&triple).inspect   # => [3, 6, 9, 12, 15]

# ตัวอย่างที่มีประโยชน์: Parameterized filters
in_range = ->(min, max, value) { (min..max).include?(value) }
valid_age = in_range.curry.(0).(120)
valid_score = in_range.curry.(0).(100)

puts [25, 150, -1, 99].select(&valid_age).inspect    # => [25, 99]
puts [85, 101, -5, 70].select(&valid_score).inspect  # => [85, 70]

# สร้าง operators ด้วย curry
greater_than = ->(threshold, value) { value > threshold }
less_than = ->(threshold, value) { value < threshold }

above_zero = greater_than.curry.(0)
below_hundred = less_than.curry.(100)

numbers = [-5, 0, 50, 100, 150]
puts numbers.select(&above_zero).inspect      # => [50, 100, 150]
puts numbers.select(&below_hundred).inspect   # => [-5, 0, 50]

# Curried method composition
format_currency = ->(symbol, amount) { "#{symbol}#{amount.round(2)}" }
thb = format_currency.curry.("฿")
usd = format_currency.curry.("$")

amounts = [100.5, 200.33, 50.0]
puts amounts.map(&thb).inspect   # => ["฿100.5", "฿200.33", "฿50.0"]
puts amounts.map(&usd).inspect   # => ["$100.5", "$200.33", "$50.0"]
```

---

## ขั้นตอนที่ 182: Custom Iterators ด้วย Block

```ruby
# สร้าง custom iterator
class NumberList
  def initialize(*numbers)
    @numbers = numbers
  end

  def each_even
    @numbers.each { |n| yield n if n.even? }
  end

  def each_odd
    @numbers.each { |n| yield n if n.odd? }
  end

  def each_with_running_sum
    sum = 0
    @numbers.each do |n|
      sum += n
      yield n, sum
    end
  end
end

list = NumberList.new(1, 2, 3, 4, 5, 6)

list.each_even { |n| print "#{n} " }
puts   # => 2 4 6

list.each_odd { |n| print "#{n} " }
puts   # => 1 3 5

list.each_with_running_sum { |n, sum| puts "#{n}: running sum = #{sum}" }

# Iterator ที่ return Enumerator
class PaginatedList
  def initialize(items, page_size)
    @items = items
    @page_size = page_size
  end

  def each_page
    return enum_for(:each_page) unless block_given?
    
    @items.each_slice(@page_size) do |page|
      yield page
    end
  end
end

pages = PaginatedList.new((1..25).to_a, 10)

# ใช้แบบ block
pages.each_page do |page|
  puts "Page: #{page.inspect}"
end

# ใช้แบบ Enumerator
first_page = pages.each_page.first
puts first_page.inspect   # => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

---

## ขั้นตอนที่ 183: Practical Block Patterns

```ruby
# Pattern 1: Transaction
def with_transaction
  puts "BEGIN TRANSACTION"
  result = yield
  puts "COMMIT"
  result
rescue => e
  puts "ROLLBACK: #{e.message}"
  raise
end

with_transaction do
  puts "กำลัง update database..."
  # ทำงานจริง ๆ ที่นี่
  "success"
end

# Pattern 2: Resource Management
def with_resource(resource)
  resource_obj = acquire_resource(resource)
  begin
    yield resource_obj
  ensure
    release_resource(resource_obj)
  end
end

def acquire_resource(name)
  puts "Acquiring #{name}"
  name
end

def release_resource(resource)
  puts "Releasing #{resource}"
end

with_resource("database connection") do |conn|
  puts "Using #{conn}"
end
# => Acquiring database connection
# => Using database connection
# => Releasing database connection

# Pattern 3: Measure Performance
def benchmark(label = "block")
  start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
  result = yield
  elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
  printf "%-20s %.6f seconds\n", label, elapsed
  result
end

benchmark("Array creation") { (1..100000).to_a }
benchmark("Hash creation") { (1..100000).each_with_object({}) { |n, h| h[n] = n } }

# Pattern 4: Builder
class HtmlBuilder
  def initialize(tag)
    @tag = tag
    @attributes = {}
    @content = []
  end

  def with_attr(key, value)
    @attributes[key] = value
    self
  end

  def with_content
    @content << yield
    self
  end

  def build
    attrs = @attributes.map { |k, v| " #{k}=\"#{v}\"" }.join
    content = @content.join
    "<#{@tag}#{attrs}>#{content}</#{@tag}>"
  end
end

html = HtmlBuilder.new("a")
  .with_attr("href", "https://ruby-lang.org")
  .with_attr("class", "link")
  .with_content { "Ruby Language" }
  .build

puts html
# => <a href="https://ruby-lang.org" class="link">Ruby Language</a>
```

---

## ขั้นตอนที่ 184: Proc ใน Data Structures

```ruby
# เก็บ behaviors ใน Hash
handlers = {
  success: proc { |data| puts "✓ Success: #{data}" },
  error: proc { |msg| puts "✗ Error: #{msg}" },
  warning: proc { |msg| puts "⚠ Warning: #{msg}" }
}

handlers[:success].call("Order created")
handlers[:error].call("Database connection failed")
handlers[:warning].call("Low memory")

# Strategy Pattern ด้วย Proc
class Sorter
  STRATEGIES = {
    by_name: ->(a, b) { a[:name] <=> b[:name] },
    by_age: ->(a, b) { a[:age] <=> b[:age] },
    by_salary_desc: ->(a, b) { b[:salary] <=> a[:salary] }
  }

  def initialize(strategy = :by_name)
    @strategy = STRATEGIES[strategy] || STRATEGIES[:by_name]
  end

  def sort(people)
    people.sort(&@strategy)
  end

  def change_strategy(new_strategy)
    @strategy = STRATEGIES[new_strategy]
    self
  end
end

people = [
  { name: "Charlie", age: 35, salary: 70000 },
  { name: "Alice", age: 25, salary: 80000 },
  { name: "Bob", age: 30, salary: 60000 }
]

sorter = Sorter.new(:by_age)
puts sorter.sort(people).map { |p| p[:name] }.inspect
# => ["Alice", "Bob", "Charlie"]

sorter.change_strategy(:by_salary_desc)
puts sorter.sort(people).map { |p| p[:name] }.inspect
# => ["Alice", "Charlie", "Bob"]
```

---

## ขั้นตอนที่ 185: Fibers - Cooperative Concurrency

```ruby
# Fiber - lightweight concurrency primitive
fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield   # หยุดชั่วคราว
  puts "Step 2"
  Fiber.yield
  puts "Step 3"
end

fiber.resume   # => Step 1
fiber.resume   # => Step 2
fiber.resume   # => Step 3

# Fiber สำหรับ generator pattern
def fibonacci_fiber
  Fiber.new do
    a, b = 0, 1
    loop do
      Fiber.yield a
      a, b = b, a + b
    end
  end
end

fib = fibonacci_fiber
10.times { print "#{fib.resume} " }
puts   # => 0 1 1 2 3 5 8 13 21 34

# Fiber สำหรับ infinite sequences
def counter_fiber(start = 0, step = 1)
  Fiber.new do
    n = start
    loop do
      Fiber.yield n
      n += step
    end
  end
end

odds = counter_fiber(1, 2)
puts 5.times.map { odds.resume }.inspect   # => [1, 3, 5, 7, 9]

evens = counter_fiber(0, 2)
puts 5.times.map { evens.resume }.inspect  # => [0, 2, 4, 6, 8]

# Fiber.yield กับ value
producer = Fiber.new do
  (1..5).each do |n|
    Fiber.yield n * n
  end
  nil
end

loop do
  val = producer.resume
  break if val.nil?
  puts val
end
```

---

## ขั้นตอนที่ 186: Enumerator::Lazy กับ Block

```ruby
# Lazy block evaluation
def infinite_series
  Enumerator.new do |yielder|
    n = 1
    loop do
      yielder << n
      n += 1
    end
  end
end

series = infinite_series.lazy

# ทำงานกับ infinite series อย่างมีประสิทธิภาพ
result = series.select { |n| n % 3 == 0 }
              .map { |n| n ** 2 }
              .first(5)
puts result.inspect   # => [9, 36, 81, 144, 225]

# Lazy with complex pipeline
primes_lazy = (2..Float::INFINITY).lazy.select do |n|
  (2..Math.sqrt(n).to_i).none? { |i| n % i == 0 }
end

puts primes_lazy.first(10).inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]

# Combining lazy enumerators
evens = (0..Float::INFINITY).lazy.select(&:even?)
first_10_even_squares = evens.map { |n| n ** 2 }.first(10)
puts first_10_even_squares.inspect
# => [0, 4, 16, 36, 64, 100, 144, 196, 256, 324]

# Lazy สำหรับ file processing
# File.open("large_file.txt").each_line.lazy
#   .select { |line| line.include?("ERROR") }
#   .map { |line| line.strip }
#   .first(100)
```

---

## ขั้นตอนที่ 187: Proc ใน OOP

```ruby
# Callback pattern
class Button
  attr_reader :label

  def initialize(label, &click_handler)
    @label = label
    @click_handler = click_handler || -> { puts "Default action" }
  end

  def click
    @click_handler.call(self)
  end
end

save_btn = Button.new("Save") do |btn|
  puts "#{btn.label} clicked! Saving..."
end

cancel_btn = Button.new("Cancel") { |btn| puts "Cancelled!" }
default_btn = Button.new("OK")

save_btn.click    # => Save clicked! Saving...
cancel_btn.click  # => Cancelled!
default_btn.click # => Default action

# Observer Pattern ด้วย Proc
class EventEmitter
  def initialize
    @listeners = Hash.new { |h, k| h[k] = [] }
  end

  def on(event, &callback)
    @listeners[event] << callback
    self
  end

  def emit(event, *args)
    @listeners[event].each { |cb| cb.call(*args) }
  end

  def off(event)
    @listeners.delete(event)
    self
  end
end

emitter = EventEmitter.new

emitter
  .on(:login) { |user| puts "Welcome #{user}!" }
  .on(:login) { |user| puts "Logged at #{Time.now}" }
  .on(:logout) { |user| puts "Goodbye #{user}!" }

emitter.emit(:login, "Alice")
emitter.emit(:logout, "Alice")
```

---

## ขั้นตอนที่ 188: Memoization ด้วย Proc/Lambda

```ruby
# Simple memoization
def memoize(callable)
  cache = {}
  ->(n) { cache[n] ||= callable.(n) }
end

slow_fibonacci = ->(n) do
  return n if n <= 1
  slow_fibonacci.(n - 1) + slow_fibonacci.(n - 2)
end

# เร็วขึ้นด้วย memoization
fast_fibonacci = memoize(slow_fibonacci)
puts fast_fibonacci.(30)   # เร็วมาก

# Memoize ด้วย hash และ closure
def create_memoized(func)
  cache = {}
  lambda do |*args|
    cache[args] ||= func.call(*args)
  end
end

expensive = ->(n) { sleep(0.01); n ** 3 }
memoized = create_memoized(expensive)

puts memoized.(5)   # => 125 (computed)
puts memoized.(5)   # => 125 (cached)
puts memoized.(3)   # => 27 (computed)
puts memoized.(3)   # => 27 (cached)

# Generic memoize module
module Memoization
  def memoize_method(method_name)
    original = instance_method(method_name)
    cache_var = "@#{method_name}_cache"

    define_method(method_name) do |*args|
      cache = instance_variable_get(cache_var) || {}
      unless cache.key?(args)
        cache[args] = original.bind(self).call(*args)
        instance_variable_set(cache_var, cache)
      end
      cache[args]
    end
  end
end

class Calculator
  extend Memoization

  def fib(n)
    n <= 1 ? n : fib(n - 1) + fib(n - 2)
  end

  memoize_method :fib
end

calc = Calculator.new
puts calc.fib(40)   # เร็วมาก
```

---

## ขั้นตอนที่ 189: Advanced Block Patterns

```ruby
# Pattern 1: Middleware/Pipeline
class Pipeline
  def initialize
    @middlewares = []
  end

  def use(middleware = nil, &block)
    @middlewares << (middleware || block)
    self
  end

  def call(input)
    @middlewares.reduce(input) do |result, middleware|
      middleware.call(result)
    end
  end
end

text_pipeline = Pipeline.new
  .use { |text| text.strip }
  .use { |text| text.downcase }
  .use { |text| text.gsub(/[^\w\s]/, '') }
  .use { |text| text.split.map(&:capitalize).join(" ") }

puts text_pipeline.call("  HELLO, WORLD!  ")   # => Hello World

# Pattern 2: Lazy Evaluation
class LazyValue
  def initialize(&block)
    @block = block
    @evaluated = false
  end

  def value
    unless @evaluated
      @value = @block.call
      @evaluated = true
    end
    @value
  end

  def to_s
    value.to_s
  end
end

lazy_config = LazyValue.new do
  puts "กำลังโหลด config..."
  { host: "localhost", port: 3000 }
end

puts "สร้าง lazy value แล้ว"
sleep(0.01)
puts "กำลังเข้าถึง value..."
puts lazy_config.value[:host]   # => "กำลังโหลด config..." แล้ว localhost

# Pattern 3: Continuation Passing Style (CPS)
def divide_cps(a, b, success_cont, error_cont)
  if b.zero?
    error_cont.call("Division by zero")
  else
    success_cont.call(a.to_f / b)
  end
end

on_success = ->(result) { puts "Result: #{result}" }
on_error = ->(msg) { puts "Error: #{msg}" }

divide_cps(10, 2, on_success, on_error)    # => Result: 5.0
divide_cps(10, 0, on_success, on_error)    # => Error: Division by zero
```

---

## ขั้นตอนที่ 190: Blocks สำหรับ DSL (Domain Specific Language)

```ruby
# DSL ด้วย instance_eval
class HtmlDSL
  def initialize
    @html = ""
  end

  def method_missing(tag, attrs = {}, &block)
    attr_str = attrs.map { |k, v| " #{k}=\"#{v}\"" }.join
    @html += "<#{tag}#{attr_str}>"
    if block
      @html += block.call.to_s
    end
    @html += "</#{tag}>"
    @html
  end

  def text(content)
    @html += content
  end

  def to_s
    @html
  end
end

class HtmlBuilder
  def self.build(&block)
    builder = HtmlDSL.new
    builder.instance_eval(&block)
    builder.to_s
  end
end

# ตัวอย่าง simple DSL
class Config
  attr_reader :settings

  def initialize
    @settings = {}
  end

  def self.configure(&block)
    config = new
    config.instance_eval(&block)
    config
  end

  def set(key, value)
    @settings[key] = value
  end

  def database(&block)
    db_config = Config.new
    db_config.instance_eval(&block) if block
    @settings[:database] = db_config.settings
  end
end

app_config = Config.configure do
  set :app_name, "My App"
  set :version, "1.0.0"
  set :debug, false

  database do
    set :host, "localhost"
    set :port, 5432
    set :name, "my_app_db"
  end
end

puts app_config.settings.inspect
```

---

## ขั้นตอนที่ 191-200: Advanced Topics

```ruby
# ขั้นตอนที่ 191: Proc arity
puts proc { }.arity            # => 0
puts proc { |x| }.arity       # => 1
puts proc { |x, y| }.arity    # => 2
puts proc { |*x| }.arity      # => -1
puts proc { |x, *y| }.arity   # => -2 (ต้องมีอย่างน้อย 1)

puts lambda { }.arity          # => 0
puts lambda { |x| }.arity     # => 1
puts ->(x, y) {}.arity        # => 2
puts ->(*x) {}.arity          # => -1

# ขั้นตอนที่ 192: Proc#>> และ Proc#<< (Ruby 2.6+)
not_nil = method(:puts).>>(proc {}) rescue nil   # example
add1 = ->(n) { n + 1 }
mul2 = ->(n) { n * 2 }

composed = add1 >> mul2   # mul2(add1(x))
puts composed.(5)   # => 12

composed2 = add1 << mul2  # add1(mul2(x))
puts composed2.(5)  # => 11

# ขั้นตอนที่ 193: Callable Duck Typing
class Adder
  def call(a, b)
    a + b
  end
  
  # ทำให้ respond to () syntax
  alias_method :[], :call
end

adder = Adder.new
puts adder.(3, 4)    # => 7
puts adder[3, 4]     # => 7
puts adder.call(3, 4) # => 7

# ขั้นตอนที่ 194: instance_exec vs instance_eval
class MyClass
  def initialize
    @value = 42
  end
end

obj = MyClass.new

# instance_eval กับ block
result = obj.instance_eval { @value }
puts result   # => 42

# instance_exec สามารถส่ง arguments ได้
result = obj.instance_exec(10) { |factor| @value * factor }
puts result   # => 420

# ขั้นตอนที่ 195: Proc กับ Exception Handling
safe_div = proc do |a, b|
  begin
    a / b
  rescue ZeroDivisionError
    puts "Cannot divide by zero!"
    nil
  end
end

puts safe_div.(10, 2).inspect   # => 5
puts safe_div.(10, 0).inspect   # => Cannot divide by zero! => nil

# ขั้นตอนที่ 196: Lazy Proc Chain
pipeline = [
  ->(n) { n * 2 },
  ->(n) { n + 1 },
  ->(n) { n.to_s }
]

result = pipeline.reduce(5) { |val, func| func.(val) }
puts result   # => "11"

# ขั้นตอนที่ 197: Partial Application ด้วย Currying
def partial(func, *partial_args)
  ->(*args) { func.call(*partial_args, *args) }
end

multiply = ->(a, b) { a * b }
double = partial(multiply, 2)
triple = partial(multiply, 3)

puts [1, 2, 3, 4, 5].map(&double).inspect   # => [2, 4, 6, 8, 10]
puts [1, 2, 3, 4, 5].map(&triple).inspect   # => [3, 6, 9, 12, 15]

# ขั้นตอนที่ 198: Blocks กับ Concurrency
require 'thread'

mutex = Mutex.new
results = []

threads = 5.times.map do |i|
  Thread.new do
    result = i * i
    mutex.synchronize { results << result }
  end
end

threads.each(&:join)
puts results.sort.inspect   # => [0, 1, 4, 9, 16]

# ขั้นตอนที่ 199: Proc สำหรับ Testing
class TestSuite
  def initialize(name)
    @name = name
    @tests = []
    @passed = 0
    @failed = 0
  end

  def test(description, &block)
    @tests << { description: description, test: block }
  end

  def run
    puts "=== #{@name} ==="
    @tests.each do |t|
      begin
        t[:test].call
        @passed += 1
        puts "  ✓ #{t[:description]}"
      rescue => e
        @failed += 1
        puts "  ✗ #{t[:description]}: #{e.message}"
      end
    end
    puts "Results: #{@passed} passed, #{@failed} failed"
  end
end

suite = TestSuite.new("Basic Math Tests")

suite.test("Addition works") do
  raise "Failed" unless 2 + 2 == 4
end

suite.test("Division works") do
  raise "Failed" unless 10 / 2 == 5
end

suite.test("This will fail") do
  raise "Intentional failure"
end

suite.run

# ขั้นตอนที่ 200: Summary - Best Practices
# 1. ใช้ lambda สำหรับ functions ที่ส่งผ่าน
# 2. ใช้ proc/block สำหรับ callbacks
# 3. ใช้ >> และ << สำหรับ composition
# 4. curry สำหรับ partial application
# 5. Fibers สำหรับ generators
# 6. Lazy สำหรับ infinite sequences
```

---

## แบบฝึกหัด (ขั้นตอนที่ 171-200)

### ข้อที่ 1: Custom Map

```ruby
# เฉลย
def my_map(array)
  return to_enum(:my_map, array) unless block_given?
  result = []
  array.each { |item| result << yield(item) }
  result
end

puts my_map([1, 2, 3]) { |n| n * 2 }.inspect   # => [2, 4, 6]
puts my_map(["a", "b", "c"], &:upcase).inspect  # => ["A", "B", "C"]
```

### ข้อที่ 2: Memoize Function

```ruby
# เฉลย
def memoize(func)
  cache = {}
  lambda do |*args|
    cache[args] ||= func.call(*args)
  end
end

slow_calc = ->(n) { sleep(0.001); n ** 3 }
fast_calc = memoize(slow_calc)

10.times { puts fast_calc.(5) }   # คำนวณครั้งเดียว
```

### ข้อที่ 3: Event System

```ruby
# เฉลย
class EventBus
  def initialize
    @subscribers = Hash.new { |h, k| h[k] = [] }
  end

  def subscribe(event, &handler)
    @subscribers[event] << handler
    -> { @subscribers[event].delete(handler) }  # return unsubscribe function
  end

  def publish(event, data = nil)
    @subscribers[event].each { |h| h.call(data) }
  end
end

bus = EventBus.new

unsubscribe = bus.subscribe(:user_created) { |user| puts "Welcome #{user[:name]}!" }
bus.subscribe(:user_created) { |user| puts "Sending email to #{user[:email]}" }

bus.publish(:user_created, { name: "Alice", email: "alice@example.com" })

unsubscribe.call  # ยกเลิก subscription แรก
bus.publish(:user_created, { name: "Bob", email: "bob@example.com" })
# เฉพาะ email handler ที่ทำงาน
```

### ข้อที่ 4: Function Pipeline

```ruby
# เฉลย
class FunctionPipeline
  def initialize(*functions)
    @functions = functions
  end

  def <<(function)
    @functions << function
    self
  end

  def call(input)
    @functions.reduce(input) { |result, func| func.call(result) }
  end

  def +(other_pipeline)
    self.class.new(*@functions, *other_pipeline.functions)
  end

  protected

  def functions
    @functions
  end
end

pipeline = FunctionPipeline.new(
  ->(x) { x * 2 },
  ->(x) { x + 1 }
)

pipeline << ->(x) { x ** 2 }

puts pipeline.call(3)   # => ((3*2)+1)^2 = 49
```

### ข้อที่ 5: Lazy Fibonacci

```ruby
# เฉลย
def lazy_fibonacci
  Enumerator.new do |y|
    a, b = 0, 1
    loop do
      y << a
      a, b = b, a + b
    end
  end.lazy
end

fibs = lazy_fibonacci

# First 10 fibonacci numbers
puts fibs.first(10).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Fibonacci ที่มากกว่า 100
puts fibs.select { |n| n > 100 }.first(5).inspect
# => [144, 233, 377, 610, 987]

# Fibonacci คู่
puts fibs.select(&:even?).first(5).inspect
# => [0, 2, 8, 34, 144]
```

### ข้อที่ 6: Builder Pattern

```ruby
# เฉลย
class SqlQueryBuilder
  def initialize
    @select = ["*"]
    @from = nil
    @wheres = []
    @order = nil
    @limit = nil
    @joins = []
  end

  def select(*fields)
    @select = fields
    self
  end

  def from(table)
    @from = table
    self
  end

  def join(table, condition)
    @joins << "JOIN #{table} ON #{condition}"
    self
  end

  def where(condition)
    @wheres << condition
    self
  end

  def order(field, direction = "ASC")
    @order = "#{field} #{direction}"
    self
  end

  def limit(n)
    @limit = n
    self
  end

  def build
    parts = ["SELECT #{@select.join(', ')}", "FROM #{@from}"]
    parts.concat(@joins)
    parts << "WHERE #{@wheres.join(' AND ')}" unless @wheres.empty?
    parts << "ORDER BY #{@order}" if @order
    parts << "LIMIT #{@limit}" if @limit
    parts.join("\n")
  end
end

query = SqlQueryBuilder.new
  .select("u.name", "u.email", "o.total")
  .from("users u")
  .join("orders o", "o.user_id = u.id")
  .where("u.active = true")
  .where("o.total > 1000")
  .order("o.total", "DESC")
  .limit(10)
  .build

puts query
```

### ข้อที่ 7: Retry with Exponential Backoff

```ruby
# เฉลย
def with_retry(max_retries: 3, base_delay: 1, max_delay: 60)
  attempts = 0
  begin
    yield
  rescue => e
    attempts += 1
    raise if attempts > max_retries
    
    delay = [base_delay * (2 ** (attempts - 1)), max_delay].min
    puts "Attempt #{attempts} failed: #{e.message}. Retrying in #{delay}s..."
    sleep(delay)
    retry
  end
end

# ใช้งาน:
begin
  result = with_retry(max_retries: 3, base_delay: 1) do
    raise "Network error" if rand < 0.7   # 70% chance of failure
    "Success!"
  end
  puts result
rescue => e
  puts "All retries failed: #{e.message}"
end
```

### ข้อที่ 8-30: แบบฝึกหัดเพิ่มเติม

```ruby
# ข้อ 8: Compose ด้วย Proc#>>
double = ->(n) { n * 2 }
square = ->(n) { n ** 2 }
to_string = ->(n) { "Value: #{n}" }

pipeline = double >> square >> to_string
puts pipeline.(3)   # => Value: 36

# ข้อ 9: Partial Application
def partial(lambda_fn, *args)
  ->(*more_args) { lambda_fn.(*args, *more_args) }
end

greet = ->(greeting, name) { "#{greeting}, #{name}!" }
hello = partial(greet, "Hello")
hi = partial(greet, "Hi")

puts hello.("Alice")   # => Hello, Alice!
puts hi.("Bob")        # => Hi, Bob!

# ข้อ 10: Curried Validators
require 'date'

between = ->(min, max, val) { (min..max).include?(val) }
valid_year = between.curry.(1900).(Date.today.year)
valid_month = between.curry.(1).(12)
valid_day = between.curry.(1).(31)

def valid_date?(y, m, d)
  Date.valid_date?(y, m, d)
end

puts valid_year.(2000)    # => true
puts valid_month.(13)     # => false

# ข้อ 11: Observable
module Observable
  def self.included(base)
    base.instance_variable_set(:@callbacks, Hash.new { |h, k| h[k] = [] })
    base.extend(ClassMethods)
  end

  module ClassMethods
    def on(event, &callback)
      @callbacks[event] << callback
    end

    def emit(event, *args)
      @callbacks[event].each { |cb| cb.call(*args) }
    end
  end
end

class Order
  include Observable

  attr_reader :status

  def initialize(id)
    @id = id
    @status = :pending
  end

  def complete!
    @status = :completed
    self.class.emit(:order_completed, self)
  end
end

Order.on(:order_completed) { |order| puts "Order completed!" }
Order.on(:order_completed) { |order| puts "Sending confirmation email" }

order = Order.new(1)
order.complete!

# ข้อ 12: Decorator ด้วย Proc
def add_logging(func, name = "function")
  ->(*args) {
    puts "Calling #{name} with #{args.inspect}"
    result = func.call(*args)
    puts "#{name} returned #{result.inspect}"
    result
  }
end

add = ->(a, b) { a + b }
logged_add = add_logging(add, "add")

logged_add.(3, 4)
# => Calling add with [3, 4]
# => add returned 7

# ข้อ 13: Pipeline สำหรับ Data Transformation
class DataTransformer
  def initialize(data)
    @data = data
    @transforms = []
  end

  def filter(&predicate)
    @transforms << [:filter, predicate]
    self
  end

  def transform(&mapper)
    @transforms << [:transform, mapper]
    self
  end

  def reduce(initial, &reducer)
    result = execute
    result.reduce(initial, &reducer)
  end

  def to_a
    execute
  end

  private

  def execute
    @transforms.reduce(@data) do |data, (type, func)|
      case type
      when :filter then data.select(&func)
      when :transform then data.map(&func)
      end
    end
  end
end

result = DataTransformer.new([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
  .filter { |n| n.even? }
  .transform { |n| n ** 2 }
  .filter { |n| n > 20 }
  .to_a

puts result.inspect   # => [36, 64, 100]

# ข้อ 14: Generator
def range_generator(from, to, step = 1)
  Enumerator.new do |y|
    current = from
    loop do
      break if current > to
      y << current
      current += step
    end
  end
end

gen = range_generator(0, 100, 10)
puts gen.to_a.inspect   # => [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100]

# ข้อ 15: Async Simulation ด้วย Fiber
class AsyncSimulator
  def self.run(&block)
    scheduler = Fiber.new do
      block.call
    end
    scheduler.resume
  end

  def self.sleep_async(seconds)
    fiber = Fiber.current
    puts "Sleeping #{seconds}s (simulated)..."
    # In real async, would register callback
    fiber.resume
    Fiber.yield
  end
end

# ข้อ 16: State Machine ด้วย Lambda
class LightSwitch
  TRANSITIONS = {
    off: { toggle: :on },
    on: { toggle: :off }
  }

  def initialize
    @state = :off
    @enter_callbacks = Hash.new { |h, k| h[k] = [] }
    @exit_callbacks = Hash.new { |h, k| h[k] = [] }
  end

  def on_enter(state, &callback)
    @enter_callbacks[state] << callback
    self
  end

  def toggle
    transitions = TRANSITIONS[@state]
    return unless transitions&.key?(:toggle)
    
    new_state = transitions[:toggle]
    @exit_callbacks[@state].each { |cb| cb.call(@state) }
    @state = new_state
    @enter_callbacks[@state].each { |cb| cb.call(@state) }
  end

  def state
    @state
  end
end

light = LightSwitch.new
  .on_enter(:on) { puts "Light is ON" }
  .on_enter(:off) { puts "Light is OFF" }

light.toggle   # => Light is ON
light.toggle   # => Light is OFF
light.toggle   # => Light is ON

# ข้อ 17: Functional Option Builder
def build_options(**defaults)
  lambda do |**overrides|
    result = defaults.merge(overrides)
    yield(result) if block_given?
    result
  end
end

configure_server = build_options(
  host: "localhost",
  port: 3000,
  ssl: false,
  timeout: 30
)

server1 = configure_server.()
server2 = configure_server.(port: 8080, ssl: true)
puts server1.inspect
puts server2.inspect

# ข้อ 18: Lazy Map + Filter
def lazy_map_filter(enumerable)
  Enumerator::Lazy.new(enumerable) do |yielder, *values|
    yielder << values.first
  end
end

result = (1..Float::INFINITY)
  .lazy
  .select { |n| n % 7 == 0 }  # divisible by 7
  .map { |n| n ** 2 }
  .reject { |n| n > 10000 }
  .to_a

puts result.inspect   # squares of numbers divisible by 7, up to 100^2

# ข้อ 19: Context Manager
def with_context(context = {})
  Thread.current[:context] = context
  yield
ensure
  Thread.current[:context] = nil
end

def current_user
  Thread.current[:context]&.fetch(:user, nil)
end

with_context(user: "Alice") do
  puts current_user   # => Alice
  with_context(user: "Bob") do
    puts current_user   # => Bob
  end
  puts current_user   # => nil (context ถูก reset)
end

# ข้อ 20: Middleware Chain
class MiddlewareChain
  def initialize
    @middlewares = []
  end

  def use(&middleware)
    @middlewares << middleware
    self
  end

  def call(request)
    build_chain.call(request)
  end

  private

  def build_chain
    final = ->(req) { "Response: #{req}" }
    @middlewares.reverse.reduce(final) do |next_handler, middleware|
      ->(req) { middleware.call(req, next_handler) }
    end
  end
end

chain = MiddlewareChain.new
  .use do |req, next_handler|
    puts "Middleware 1: logging #{req}"
    next_handler.call(req)
  end
  .use do |req, next_handler|
    puts "Middleware 2: auth check"
    next_handler.call("authenticated: #{req}")
  end
  .use do |req, next_handler|
    puts "Middleware 3: rate limiting"
    next_handler.call(req)
  end

puts chain.call("GET /api/users")

# ข้อ 21: Function Memoization with TTL
def memoize_with_ttl(ttl_seconds, &func)
  cache = {}
  timestamps = {}
  
  lambda do |*args|
    now = Time.now.to_f
    if cache.key?(args) && (now - timestamps[args]) < ttl_seconds
      cache[args]
    else
      cache[args] = func.call(*args)
      timestamps[args] = now
      cache[args]
    end
  end
end

slow_lookup = memoize_with_ttl(5) do |id|
  puts "Looking up #{id}..."
  "User #{id}"
end

puts slow_lookup.(1)   # => Looking up 1... User 1
puts slow_lookup.(1)   # => User 1 (cached)
sleep(6)
puts slow_lookup.(1)   # => Looking up 1... User 1 (expired)

# ข้อ 22: Composition สำหรับ Validation
validate_presence = ->(field, value) { !value.nil? && !value.to_s.empty? }
validate_length = ->(min, max, field, value) { (min..max).include?(value.to_s.length) }
validate_format = ->(regex, field, value) { value.to_s.match?(regex) }

def compose_validators(*validators)
  ->(field, value) {
    errors = validators.filter_map { |v| v.call(field, value) }
    errors.empty? ? { valid: true } : { valid: false, errors: errors }
  }
end

# ข้อ 23: Functional Map Reduce
def map_reduce(data, mapper, reducer, initial)
  data.map(&mapper).reduce(initial, &reducer)
end

words = ["hello", "world", "ruby", "programming"]
total_length = map_reduce(
  words,
  ->(w) { w.length },
  ->(sum, n) { sum + n },
  0
)
puts total_length   # => 25

# ข้อ 24: Trampoline สำหรับ Tail Recursion
def trampoline(func)
  result = func
  while result.is_a?(Proc) || result.is_a?(Method)
    result = result.call
  end
  result
end

# Tail recursive factorial ด้วย trampoline
def factorial_tr(n, acc = 1)
  if n <= 1
    acc
  else
    -> { factorial_tr(n - 1, n * acc) }
  end
end

puts trampoline(factorial_tr(100))   # ไม่ stack overflow!

# ข้อ 25: Block-based DSL สำหรับ Configuration
class AppConfig
  class DatabaseConfig
    attr_accessor :host, :port, :name, :username, :password, :pool

    def initialize
      @host = "localhost"
      @port = 5432
      @pool = 5
    end
  end

  class CacheConfig
    attr_accessor :host, :port, :ttl

    def initialize
      @host = "localhost"
      @port = 6379
      @ttl = 3600
    end
  end

  attr_reader :database, :cache, :app_name, :environment

  def initialize
    @database = DatabaseConfig.new
    @cache = CacheConfig.new
    @app_name = "App"
    @environment = :development
  end

  def self.configure
    config = new
    yield config
    config
  end

  def database
    yield @database if block_given?
    @database
  end

  def cache
    yield @cache if block_given?
    @cache
  end

  def app_name=(name)
    @app_name = name
  end

  def environment=(env)
    @environment = env.to_sym
  end
end

config = AppConfig.configure do |c|
  c.app_name = "My Awesome App"
  c.environment = "production"

  c.database do |db|
    db.host = "db.production.com"
    db.name = "myapp_prod"
    db.pool = 20
  end

  c.cache do |cache|
    cache.host = "redis.production.com"
    cache.ttl = 7200
  end
end

puts "App: #{config.app_name}"
puts "Env: #{config.environment}"
puts "DB host: #{config.database.host}"
puts "Cache TTL: #{config.cache.ttl}"

# ข้อ 26: Custom each_with_object
def my_each_with_object(enumerable, initial)
  obj = initial
  enumerable.each do |item|
    yield item, obj
  end
  obj
end

result = my_each_with_object([1, 2, 3, 4, 5], {}) do |n, hash|
  hash[n] = n ** 2
end
puts result.inspect   # => {1=>1, 2=>4, 3=>9, 4=>16, 5=>25}

# ข้อ 27: Timer ด้วย Block
class Timer
  def self.measure
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    result = yield
    elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    { result: result, time: elapsed }
  end
end

timing = Timer.measure do
  (1..1000000).sum
end

puts "Result: #{timing[:result]}"
puts "Time: #{timing[:time].round(4)}s"

# ข้อ 28: Flat Map Chain
nested_data = [
  { category: "fruits", items: ["apple", "banana"] },
  { category: "veggies", items: ["carrot", "broccoli", "spinach"] },
  { category: "grains", items: ["rice", "wheat"] }
]

all_items = nested_data.flat_map { |cat| cat[:items].map { |item| "#{cat[:category]}: #{item}" } }
puts all_items.inspect

# ข้อ 29: Recursive Block
def recursive_traverse(data, depth = 0, &block)
  case data
  when Hash
    data.each { |k, v| recursive_traverse(v, depth + 1, &block) }
  when Array
    data.each { |item| recursive_traverse(item, depth + 1, &block) }
  else
    block.call(data, depth)
  end
end

nested = { a: 1, b: [2, 3, { c: 4 }], d: "hello" }
recursive_traverse(nested) { |val, depth| puts "#{"  " * depth}#{val}" }

# ข้อ 30: Final - Complete DSL
class TestFramework
  @@tests = []
  @@hooks = { before: [], after: [] }
  @@stats = { passed: 0, failed: 0 }

  def self.describe(name, &block)
    puts "\n#{name}"
    new.instance_eval(&block)
    puts "\nTotal: #{@@stats[:passed] + @@stats[:failed]} tests, " \
         "#{@@stats[:passed]} passed, #{@@stats[:failed]} failed"
  end

  def it(description, &block)
    @@hooks[:before].each(&:call)
    begin
      instance_eval(&block)
      @@stats[:passed] += 1
      puts "  ✓ #{description}"
    rescue => e
      @@stats[:failed] += 1
      puts "  ✗ #{description}: #{e.message}"
    ensure
      @@hooks[:after].each(&:call)
    end
  end

  def before(&block)
    @@hooks[:before] << block
  end

  def after(&block)
    @@hooks[:after] << block
  end

  def expect(actual)
    Expectation.new(actual)
  end

  class Expectation
    def initialize(actual)
      @actual = actual
    end

    def to_equal(expected)
      raise "Expected #{expected.inspect}, got #{@actual.inspect}" unless @actual == expected
    end

    def to_be_truthy
      raise "Expected truthy, got #{@actual.inspect}" unless @actual
    end

    def to_include(value)
      raise "Expected #{@actual.inspect} to include #{value.inspect}" unless @actual.include?(value)
    end
  end
end

TestFramework.describe("Array Operations") do
  it("adds elements") do
    arr = [1, 2, 3]
    arr << 4
    expect(arr).to_equal([1, 2, 3, 4])
  end

  it("maps correctly") do
    result = [1, 2, 3].map { |n| n * 2 }
    expect(result).to_equal([2, 4, 6])
  end

  it("selects correctly") do
    result = [1, 2, 3, 4, 5].select(&:even?)
    expect(result).to_equal([2, 4])
  end

  it("intentionally fails") do
    expect(1 + 1).to_equal(3)   # จะ fail
  end
end
```

---

## สรุปบทที่ 10

| Concept | คำอธิบาย |
|---------|----------|
| Block | Anonymous code, ส่งให้ method, ไม่ใช่ object |
| yield | Execute block ที่ส่งมา |
| block_given? | ตรวจสอบว่ามี block หรือไม่ |
| Proc | Block ที่เก็บใน object |
| Lambda | Proc ที่เข้มงวดเรื่อง args และ return |
| Closure | Function ที่ capture surrounding scope |
| Curry | Partial application |
| >> / << | Proc composition |
| Fiber | Cooperative concurrency, generators |
| Lazy | Lazy evaluation สำหรับ infinite sequences |

**Lambda vs Proc ความแตกต่างหลัก:**

| | Lambda | Proc |
|-|--------|------|
| Argument checking | Strict | Lenient |
| return | ออกจาก lambda | ออกจาก enclosing method |
| lambda? | true | false |
| ใช้เมื่อ | ส่งผ่านเป็น function | callback, code snippet |

**Key Takeaways:**
1. Block ไม่ใช่ object แต่ Proc และ Lambda เป็น
2. Lambda ปลอดภัยกว่า Proc เมื่อส่งผ่าน methods
3. Closure capture variables by reference ไม่ใช่ by value
4. ใช้ `>>` สำหรับ function composition
5. Curry ช่วยสร้าง specialized functions
6. Lazy evaluation สำหรับ infinite sequences และ large data

---

*จบตอนที่ 10 - Blocks, Procs, Lambdas*

*ถัดไป: Ruby Intermediate - Object-Oriented Programming*
