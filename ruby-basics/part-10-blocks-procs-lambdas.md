# ตอนที่ 10: Blocks, Procs, Lambdas (Steps 171-200)

> **เป้าหมาย**: เข้าใจ blocks, procs, lambdas อย่างลึกซึ้ง — หัวใจของ functional programming ใน Ruby

---

## Step 171: Block คืออะไร?

Block คือชิ้นส่วนของโค้ดที่ส่งผ่านไปยัง method ได้ เป็นส่วนสำคัญที่สุดของ Ruby และทำให้ Ruby มีพลัง

### สองรูปแบบของ Block

```ruby
# รูปแบบที่ 1: do...end (สำหรับ multi-line)
[1, 2, 3].each do |n|
  squared = n ** 2
  puts "#{n}^2 = #{squared}"
end
# 1^2 = 1
# 2^2 = 4
# 3^2 = 9

# รูปแบบที่ 2: { } (สำหรับ single-line)
[1, 2, 3].each { |n| puts n ** 2 }
# 1
# 4
# 9

# ทั้งสองทำงานเหมือนกัน แต่ {} มี operator precedence สูงกว่า do...end
```

### Block ไม่ใช่ Object (โดยตรง)

```ruby
# Block ไม่สามารถเก็บใน variable ได้โดยตรง
# block = { puts "hello" }  # SyntaxError!

# ต้องใช้ Proc หรือ Lambda แทน
block_as_proc = Proc.new { puts "hello" }
block_as_proc.call  # hello

# หรือใช้ -> (stabby lambda)
block_as_lambda = -> { puts "hello" }
block_as_lambda.call  # hello
```

### Block กับ Method

```ruby
# Block ส่งผ่านไปยัง method ได้
def run_block
  yield if block_given?
end

run_block { puts "Block ทำงาน!" }  # Block ทำงาน!
run_block                           # (ไม่ทำอะไร)

# Block กับ arguments
def run_with_value(x)
  yield x if block_given?
end

run_with_value(42) { |n| puts "ค่าคือ: #{n}" }
# ค่าคือ: 42
```

---

## Step 172: yield — เรียก Block ที่ส่งเข้ามา

`yield` เป็น keyword ที่ใช้เรียก block ที่ถูกส่งเข้ามาใน method

```ruby
# yield พื้นฐาน
def say_hello
  puts "ก่อน yield"
  yield
  puts "หลัง yield"
end

say_hello { puts "อยู่ใน block!" }
# ก่อน yield
# อยู่ใน block!
# หลัง yield

# yield กับ arguments
def transform(value)
  yield value
end

result = transform(5) { |n| n * 2 }
puts result  # 10

# yield หลายครั้ง
def repeat_three_times
  yield 1
  yield 2
  yield 3
end

repeat_three_times { |i| puts "ครั้งที่ #{i}" }
# ครั้งที่ 1
# ครั้งที่ 2
# ครั้งที่ 3
```

### สร้าง Custom Iterator ด้วย yield

```ruby
# สร้าง each_odd
def each_odd(array)
  array.each_with_index do |item, i|
    yield item if i.odd?
  end
end

each_odd([10, 20, 30, 40, 50]) { |n| puts n }
# 20
# 40

# สร้าง countdown
def countdown(from, to = 0)
  n = from
  while n >= to
    yield n
    n -= 1
  end
end

countdown(5) { |n| print "#{n} " }  # 5 4 3 2 1 0

# สร้าง retry_on_failure
def retry_on_failure(max_attempts: 3)
  attempts = 0
  begin
    attempts += 1
    yield attempts
  rescue => e
    retry if attempts < max_attempts
    raise "ล้มเหลวหลังจากลอง #{max_attempts} ครั้ง: #{e.message}"
  end
end

retry_on_failure(max_attempts: 3) do |attempt|
  puts "ลองครั้งที่ #{attempt}"
  raise "Connection failed" if attempt < 3
  puts "สำเร็จ!"
end
# ลองครั้งที่ 1
# ลองครั้งที่ 2
# ลองครั้งที่ 3
# สำเร็จ!
```

---

## Step 173: block_given? — ตรวจสอบว่ามี Block ส่งมาไหม

```ruby
# block_given? คืน true ถ้ามี block ส่งมา
def optional_block
  if block_given?
    yield
  else
    puts "ไม่มี block"
  end
end

optional_block { puts "มี block!" }  # มี block!
optional_block                        # ไม่มี block

# ใช้งานจริง: default behavior
def process_data(data)
  result = data.map { |x| x * 2 }
  
  if block_given?
    result.each { |item| yield item }
  else
    result  # คืน array ถ้าไม่มี block
  end
end

# ไม่มี block: คืน array
arr = process_data([1, 2, 3])
puts arr.inspect  # [2, 4, 6]

# มี block: yield แต่ละตัว
process_data([1, 2, 3]) { |n| print "#{n} " }
# 2 4 6
```

### block_given? กับ Default Behavior

```ruby
def log(message, &block)
  # ถ้าไม่มี block ใช้ default formatter
  formatter = block_given? ? block : ->(msg) { "[DEFAULT] #{msg}" }
  puts formatter.call(message)
end

log("สวัสดี")
# [DEFAULT] สวัสดี

log("สวัสดี") { |msg| "🟢 #{msg}" }
# 🟢 สวัสดี

# guard against missing block
def divide_safe(a, b)
  return yield(ZeroDivisionError.new("หารด้วยศูนย์")) if b == 0 && block_given?
  return nil if b == 0
  a.to_f / b
end

result = divide_safe(10, 0) { |err| "Error: #{err.message}" }
puts result  # Error: หารด้วยศูนย์
```

---

## Step 174: Passing Explicit Block (&block)

```ruby
# & รับ block เป็น explicit Proc object
def capture_block(&block)
  puts block.class  # Proc
  block.call
end

capture_block { puts "ฉันถูก capture!" }
# Proc
# ฉันถูก capture!

# เก็บ block ใน variable
def save_block(&block)
  @saved_block = block
end

def run_saved_block
  @saved_block.call if @saved_block
end

save_block { puts "Block ที่บันทึกไว้!" }
run_saved_block  # Block ที่บันทึกไว้!

# ส่ง block ต่อไปยัง method อื่น
def my_map(array, &block)
  result = []
  array.each do |item|
    result << block.call(item)
  end
  result
end

puts my_map([1, 2, 3]) { |n| n * 10 }.inspect  # [10, 20, 30]

# ส่ง block ต่อ
def my_select(array, &block)
  result = []
  array.each do |item|
    result << item if block.call(item)
  end
  result
end

def filter_and_transform(array, &block)
  filtered = my_select(array) { |n| n > 2 }  # ไม่ส่ง block ต่อ
  my_map(filtered, &block)  # ส่ง block ต่อด้วย &block
end

puts filter_and_transform([1, 2, 3, 4, 5]) { |n| n ** 2 }.inspect  # [9, 16, 25]
```

---

## Step 175: Proc.new {} — สร้าง Proc Object

Proc คือ block ที่ถูก objectify แล้ว เก็บได้ในตัวแปร ส่งต่อได้ เรียกซ้ำได้

```ruby
# สร้าง Proc ด้วย Proc.new
greet = Proc.new { |name| puts "สวัสดี, #{name}!" }
greet.call("Alice")   # สวัสดี, Alice!
greet.call("Bob")     # สวัสดี, Bob!

# Proc เป็น object จริงๆ
puts greet.class   # Proc
puts greet.arity   # 1 (รับ 1 argument)
puts greet.lambda? # false

# เก็บใน array
formatters = [
  Proc.new { |x| x.upcase },
  Proc.new { |x| x.reverse },
  Proc.new { |x| x.gsub("a", "*") }
]

text = "banana"
formatters.each do |f|
  puts f.call(text)
end
# BANANA
# ananab
# b*n*n*
```

### Proc กับ Closures

```ruby
# Proc จำตัวแปรจาก context ที่สร้าง (closure)
def make_counter(start = 0)
  count = start
  Proc.new { count += 1; count }
end

counter1 = make_counter
counter2 = make_counter(10)

puts counter1.call  # 1
puts counter1.call  # 2
puts counter2.call  # 11
puts counter1.call  # 3  (independent จาก counter2)
```

---

## Step 176: proc {} — Shorthand สำหรับ Proc

```ruby
# proc {} เป็น shorthand ของ Proc.new {}
greeter = proc { |name| "Hello, #{name}!" }
puts greeter.call("World")  # Hello, World!

# เหมือนกัน 100%
p1 = Proc.new { |x| x * 2 }
p2 = proc { |x| x * 2 }

puts p1.call(5)  # 10
puts p2.call(5)  # 10
puts p1.class    # Proc
puts p2.class    # Proc
puts p1.lambda?  # false
puts p2.lambda?  # false

# ใช้งานจริง: factory functions
def make_adder(n)
  proc { |x| x + n }
end

add5  = make_adder(5)
add10 = make_adder(10)
add20 = make_adder(20)

puts add5.call(3)   # 8
puts add10.call(3)  # 13
puts add20.call(3)  # 23

# ใช้กับ map
[1, 2, 3, 4, 5].map(&add5).tap { |r| puts r.inspect }  # [6, 7, 8, 9, 10]
```

---

## Step 177: Lambda — lambda {} และ -> {}

Lambda คือ Proc พิเศษที่มี strict argument checking และ return behavior ที่แตกต่าง

```ruby
# สร้าง lambda ด้วย lambda {}
greet = lambda { |name| "Hello, #{name}!" }
puts greet.call("Alice")   # Hello, Alice!
puts greet.class           # Proc
puts greet.lambda?         # true  ← ต่างจาก Proc ทั่วไป

# สร้าง lambda ด้วย -> {} (stabby lambda, Ruby 1.9+)
greet2 = ->(name) { "Hi, #{name}!" }
puts greet2.call("Bob")  # Hi, Bob!
puts greet2.lambda?      # true

# Multi-line stabby lambda
process = ->(x, y) {
  sum = x + y
  product = x * y
  { sum: sum, product: product }
}

puts process.call(3, 4).inspect  # {:sum=>7, :product=>12}

# Lambda ที่ไม่มี argument
say_hello = -> { puts "สวัสดี!" }
say_hello.call  # สวัสดี!
```

### Lambda กับ arity

```ruby
# Lambda strict เรื่อง argument count
strict = lambda { |a, b| a + b }
puts strict.call(3, 4)  # 7

# strict.call(3)    # ArgumentError: wrong number of arguments
# strict.call(3, 4, 5)  # ArgumentError

# Proc ไม่ strict
loose = Proc.new { |a, b| "#{a}, #{b}" }
puts loose.call(3, 4)     # 3, 4
puts loose.call(3)        # 3,    (b เป็น nil)
puts loose.call(3, 4, 5)  # 3, 4  (ตัวเกินถูกละเว้น)
```

---

## Step 178: Proc.call, .(), [] — วิธีเรียก Proc/Lambda

```ruby
# 3 วิธีในการเรียก Proc/Lambda
double = proc { |n| n * 2 }

# 1. .call()
puts double.call(5)   # 10

# 2. .() — syntax sugar สำหรับ call
puts double.(5)       # 10

# 3. [] — bracket notation
puts double[5]        # 10

# 4. === — ใช้กับ case/when
puts double === 5     # 10 (ค่า truthy)

# ทั้งหมดเหมือนกัน
square = ->(n) { n ** 2 }
puts [square.call(3), square.(3), square[3], square === 3].inspect
# [9, 9, 9, 9]
```

### === กับ case/when

```ruby
# Proc/Lambda ใช้กับ case/when ได้!
even?   = ->(n) { n.even? }
positive? = ->(n) { n > 0 }
large?   = ->(n) { n > 100 }

numbers = [-5, 0, 4, 7, 150, -200]

numbers.each do |n|
  description = case n
                when large?    then "ใหญ่มาก"
                when positive? then "บวก"
                when even?     then "คู่และไม่บวก"
                else                "ลบและคี่"
                end
  puts "#{n}: #{description}"
end
# -5: ลบและคี่
# 0: คู่และไม่บวก
# 4: บวก
# 7: บวก
# 150: ใหญ่มาก
# -200: คู่และไม่บวก
```

---

## Step 179: Proc vs Lambda — ความแตกต่างสำคัญ

### ความแตกต่างที่ 1: Arity (จำนวน arguments)

```ruby
# Proc: ไม่ strict เรื่อง arity
loose_proc = Proc.new { |a, b, c| "#{a}, #{b}, #{c}" }
puts loose_proc.call(1, 2)         # "1, 2, "  (c = nil)
puts loose_proc.call(1, 2, 3, 4)  # "1, 2, 3"  (ตัวเกินละเว้น)

# Lambda: strict เรื่อง arity
strict_lambda = lambda { |a, b, c| "#{a}, #{b}, #{c}" }
# strict_lambda.call(1, 2)     # ArgumentError!
# strict_lambda.call(1, 2, 3, 4) # ArgumentError!
puts strict_lambda.call(1, 2, 3)  # "1, 2, 3"  ✅
```

### ความแตกต่างที่ 2: return Behavior

```ruby
# PROC: return ออกจาก method ที่ล้อม Proc อยู่
def proc_return_test
  puts "ก่อน Proc"
  p = Proc.new { return "จาก Proc" }
  p.call
  puts "หลัง Proc"  # ← ไม่ถูก execute!
  "จาก method"
end

puts proc_return_test
# ก่อน Proc
# จาก Proc   ← method จบที่นี่เลย

# LAMBDA: return ออกแค่จาก Lambda เอง
def lambda_return_test
  puts "ก่อน Lambda"
  l = lambda { return "จาก Lambda" }
  result = l.call
  puts "หลัง Lambda: #{result}"  # ← ยังทำงานอยู่!
  "จาก method"
end

puts lambda_return_test
# ก่อน Lambda
# หลัง Lambda: จาก Lambda
# จาก method
```

### สรุปความแตกต่าง

```ruby
# ตารางเปรียบเทียบ Proc vs Lambda
#
# Feature          | Proc                | Lambda
# ----------------|---------------------|------------------
# สร้าง           | Proc.new {} / proc{}| lambda {} / -> {}
# lambda?          | false               | true
# arity            | ไม่ strict          | strict
# return           | ออกจาก method      | ออกจาก lambda
# break            | ออกจาก method      | ออกจาก lambda
# next             | เหมือนกัน          | เหมือนกัน
```

---

## Step 180: Closure คืออะไร? — Capturing Variables

Closure คือ block/proc/lambda ที่ "จดจำ" ตัวแปรจาก environment ที่มันถูกสร้างขึ้น

```ruby
# Closure จำ variable จาก scope ที่สร้าง
def make_multiplier(factor)
  # lambda นี้ "จำ" ค่า factor แม้ว่า make_multiplier จะ return แล้ว
  ->(n) { n * factor }
end

double = make_multiplier(2)
triple = make_multiplier(3)
times10 = make_multiplier(10)

puts double.call(5)   # 10
puts triple.call(5)   # 15
puts times10.call(5)  # 50

# Closure สามารถแก้ไขตัวแปรจาก scope ได้
def make_counter
  count = 0  # ตัวแปรที่ closure จะ capture
  
  increment = -> { count += 1; count }
  decrement = -> { count -= 1; count }
  reset     = -> { count = 0 }
  get       = -> { count }
  
  { increment: increment, decrement: decrement, reset: reset, get: get }
end

counter = make_counter
puts counter[:increment].call  # 1
puts counter[:increment].call  # 2
puts counter[:increment].call  # 3
puts counter[:decrement].call  # 2
puts counter[:get].call        # 2
counter[:reset].call
puts counter[:get].call        # 0
```

### Closure กับ Shared State

```ruby
# หลาย closures แชร์ state เดียวกัน
def make_bank_account(initial_balance)
  balance = initial_balance
  
  deposit  = ->(amount) { balance += amount; balance }
  withdraw = ->(amount) {
    return "ยอดไม่พอ" if amount > balance
    balance -= amount
    balance
  }
  statement = -> { "ยอดคงเหลือ: #{balance} บาท" }
  
  { deposit: deposit, withdraw: withdraw, statement: statement }
end

account = make_bank_account(1000)
puts account[:deposit].call(500)    # 1500
puts account[:withdraw].call(200)   # 1300
puts account[:statement].call       # ยอดคงเหลือ: 1300 บาท
puts account[:withdraw].call(2000)  # ยอดไม่พอ
```

### ระวัง: Variable Capture ใน Loops

```ruby
# ❌ ปัญหา: ทุก proc capture i เดียวกัน (ใน Python/JS มีปัญหานี้)
procs = []
[1, 2, 3].each do |i|
  procs << proc { i }  # ใน Ruby each สร้าง scope ใหม่ทุก iteration
end

puts procs.map(&:call).inspect  # [1, 2, 3] ✅ ใน Ruby ทำงานถูกต้อง!

# Ruby each ปลอดภัยเพราะสร้าง new binding ทุก iteration
```

---

## Step 181: Method Objects — .method(:name)

```ruby
# method(:name) คืน Method object
def square(n)
  n ** 2
end

m = method(:square)
puts m.class    # Method
puts m.call(5)  # 25
puts m.arity    # 1

# Method object ทำงานเหมือน Proc/Lambda
puts m.(5)     # 25
puts m[5]      # 25

# ใช้กับ map
puts [1, 2, 3, 4, 5].map(&method(:square)).inspect  # [1, 4, 9, 16, 25]

# Instance method
class Greeter
  def hello(name)
    "Hello, #{name}!"
  end
end

g = Greeter.new
m = g.method(:hello)
puts m.call("World")  # Hello, World!

# ส่งเป็น argument
def apply(value, func)
  func.call(value)
end

upcase_method = "hello".method(:*)  # ❌ ไม่ได้
# ใช้แบบนี้แทน:
def double_str(s)
  s * 2
end
puts apply("ha", method(:double_str))  # haha
```

### to_proc กับ Method

```ruby
# Method มี to_proc method
class Temperature
  def initialize(celsius)
    @celsius = celsius
  end

  def to_fahrenheit
    @celsius * 9.0 / 5 + 32
  end
end

temps = [0, 20, 37, 100].map { |c| Temperature.new(c) }
fahrenheit_values = temps.map(&:to_fahrenheit)
puts fahrenheit_values.inspect  # [32.0, 68.0, 98.6, 212.0]
```

---

## Step 182: Symbol to Proc (&:method_name)

`&:symbol` เป็น shortcut ที่ทรงพลังมาก Ruby จะแปลง symbol เป็น proc โดยเรียก method ชื่อนั้นบน argument

```ruby
# &:upcase เท่ากับ proc { |s| s.upcase }
["hello", "world"].map(&:upcase)    # ["HELLO", "WORLD"]

# &:to_i เท่ากับ proc { |s| s.to_i }
["1", "2", "3"].map(&:to_i)         # [1, 2, 3]

# &:even? เท่ากับ proc { |n| n.even? }
[1, 2, 3, 4, 5].select(&:even?)     # [2, 4]

# &:nil? เท่ากับ proc { |x| x.nil? }
[1, nil, 2, nil, 3].reject(&:nil?)  # [1, 2, 3]

# &:itself เท่ากับ proc { |x| x }
[1, nil, false, 2, "", 3].select(&:itself)  # [1, 2, "", 3]
# (nil และ false ถูกกรองออก)

# ตัวอย่างที่ซับซ้อนขึ้น
words = ["hello", "world", "ruby"]
puts words.map(&:length).inspect     # [5, 5, 4]
puts words.map(&:chars).inspect      # [["h","e","l","l","o"], ["w","o","r","l","d"], ["r","u","b","y"]]
puts words.flat_map(&:chars).inspect # ["h","e","l","l","o","w","o","r","l","d","r","u","b","y"]
puts words.sort_by(&:length).inspect # ["ruby", "hello", "world"]
```

### สร้าง Symbol to Proc เอง

```ruby
# Ruby ใช้ Symbol#to_proc ในการทำ &:symbol
class Symbol
  def to_proc
    ->(receiver, *args) { receiver.send(self, *args) }
  end
end

# ตัวอย่าง: method ที่รับ argument
# เพิ่ม argument ด้วย method(:name)
add_n = ->(n) { ->(x) { x + n } }

# ใช้กับ map
puts [1, 2, 3].map(&add_n.(5)).inspect   # [6, 7, 8]
```

---

## Step 183: Currying — Partial Application

Currying คือการแปลง function ที่รับหลาย argument เป็น chain ของ functions ที่รับ argument ทีละตัว

```ruby
# curry พื้นฐาน
add = ->(a, b) { a + b }
curried_add = add.curry

add5 = curried_add.call(5)  # ส่ง argument แรก
puts add5.call(3)           # 8
puts add5.call(10)          # 15

# หรือเรียกพร้อมกัน
puts curried_add.call(5).call(3)  # 8
puts curried_add.(5).(3)          # 8

# curry กับ 3 arguments
multiply = ->(a, b, c) { a * b * c }
curried = multiply.curry

double_and_half = curried.(2).(0.5)  # multiply(2, 0.5, ?)
puts double_and_half.(10)             # 10.0

# ตัวอย่างจริง: ฟิลเตอร์ด้วย curry
filter_by = ->(field, value, obj) { obj[field] == value }

users = [
  { name: "Alice", role: :admin },
  { name: "Bob", role: :user },
  { name: "Charlie", role: :admin }
]

is_admin = filter_by.curry.(:role).(:admin)
admins = users.select(&is_admin)
puts admins.map { |u| u[:name] }.inspect  # ["Alice", "Charlie"]
```

### Currying กับ Proc ธรรมดา

```ruby
# Proc ก็ curry ได้
multiply_proc = proc { |a, b| a * b }
triple = multiply_proc.curry.(3)
puts triple.(7)  # 21

# สร้าง pipeline ด้วย curry
def pipeline(*fns)
  fns.reduce { |composed, f| composed >> f }
end

add_one = ->(n) { n + 1 }
double  = ->(n) { n * 2 }
square  = ->(n) { n ** 2 }

transform = pipeline(add_one, double, square)
puts transform.(3)  # ((3+1)*2)^2 = 64
```

---

## Step 184: Proc Composition — >> และ <<

Ruby 2.6+ เพิ่ม `>>` และ `<<` สำหรับ compose procs/lambdas

```ruby
# >> (compose forward): f >> g หมายถึง g(f(x))
double  = ->(n) { n * 2 }
add_one = ->(n) { n + 1 }

# double แล้ว add_one
double_then_add = double >> add_one
puts double_then_add.(5)  # (5*2)+1 = 11

# << (compose backward): f << g หมายถึง f(g(x))
add_then_double = double << add_one
puts add_then_double.(5)  # (5+1)*2 = 12

# chain หลายตัว
upcase   = :upcase.to_proc
reverse  = :reverse.to_proc
strip    = :strip.to_proc

clean_string = strip >> downcase >> reverse
# หมายเหตุ: Symbol#to_proc ไม่รองรับ >> โดยตรง ต้องใช้ lambda

clean = ->(s) { s.strip } >> ->(s) { s.downcase } >> ->(s) { s.reverse }
puts clean.("  Hello World  ")  # dlrow olleh
```

### ตัวอย่างจริง: Text Processing Pipeline

```ruby
# สร้าง text processing pipeline
normalize_whitespace = ->(text) { text.gsub(/\s+/, " ").strip }
remove_punctuation   = ->(text) { text.gsub(/[^a-zA-Z0-9\s]/, "") }
downcase_text        = ->(text) { text.downcase }
tokenize             = ->(text) { text.split }
remove_stopwords     = ->(tokens) {
  stopwords = %w[the a an is are was were in on at to for of and but]
  tokens.reject { |t| stopwords.include?(t) }
}

text_pipeline = normalize_whitespace >>
                remove_punctuation >>
                downcase_text >>
                tokenize >>
                remove_stopwords

result = text_pipeline.("The Quick Brown Fox Jumps Over the Lazy Dog!")
puts result.inspect  # ["quick", "brown", "fox", "jumps", "over", "lazy", "dog"]
```

---

## Step 185: Practical Uses — Custom Iterators

```ruby
# สร้าง custom iterators ที่ยืดหยุ่น
class Tree
  attr_accessor :value, :children

  def initialize(value, children = [])
    @value = value
    @children = children
  end

  # depth-first traversal
  def each_dfs(&block)
    yield value
    children.each { |child| child.each_dfs(&block) }
  end

  # breadth-first traversal
  def each_bfs(&block)
    queue = [self]
    until queue.empty?
      node = queue.shift
      yield node.value
      queue.concat(node.children)
    end
  end
end

tree = Tree.new(1, [
  Tree.new(2, [
    Tree.new(4),
    Tree.new(5)
  ]),
  Tree.new(3, [
    Tree.new(6),
    Tree.new(7)
  ])
])

print "DFS: "
tree.each_dfs { |v| print "#{v} " }
# DFS: 1 2 4 5 3 6 7

print "\nBFS: "
tree.each_bfs { |v| print "#{v} " }
# BFS: 1 2 3 4 5 6 7
```

---

## Step 186: Callbacks Pattern

```ruby
# Event system ด้วย Procs
class EventEmitter
  def initialize
    @listeners = Hash.new { |h, k| h[k] = [] }
  end

  def on(event, &block)
    @listeners[event] << block
    self  # chain
  end

  def off(event)
    @listeners.delete(event)
    self
  end

  def emit(event, *args)
    @listeners[event].each { |listener| listener.call(*args) }
    self
  end
end

emitter = EventEmitter.new

emitter
  .on(:data) { |msg| puts "Handler 1: #{msg}" }
  .on(:data) { |msg| puts "Handler 2: #{msg.upcase}" }
  .on(:error) { |err| puts "Error: #{err}" }

emitter.emit(:data, "hello world")
# Handler 1: hello world
# Handler 2: HELLO WORLD

emitter.emit(:error, "Something went wrong")
# Error: Something went wrong
```

---

## Step 187: DSL (Domain Specific Language) ด้วย Blocks

```ruby
# สร้าง DSL สำหรับ configuration
class Config
  def initialize
    @settings = {}
  end

  def method_missing(name, *args)
    if name.to_s.end_with?("=")
      @settings[name.to_s.chomp("=").to_sym] = args.first
    else
      @settings[name]
    end
  end

  def to_h
    @settings
  end
end

def configure(&block)
  config = Config.new
  config.instance_eval(&block)
  config
end

app_config = configure do
  self.database_url = "postgresql://localhost/mydb"
  self.redis_url = "redis://localhost:6379"
  self.max_connections = 10
  self.debug = false
  self.log_level = "info"
end

puts app_config.database_url  # postgresql://localhost/mydb
puts app_config.debug         # false
puts app_config.to_h.inspect
```

### HTML Builder DSL

```ruby
class HtmlDsl
  def initialize
    @html = ""
    @indent = 0
  end

  def method_missing(tag, *args, **attrs, &block)
    content = args.first
    attr_str = attrs.map { |k, v| " #{k}=\"#{v}\"" }.join
    
    indent = "  " * @indent
    @html += "#{indent}<#{tag}#{attr_str}>"
    
    if block
      @html += "\n"
      @indent += 1
      block.call
      @indent -= 1
      @html += "#{indent}</#{tag}>\n"
    elsif content
      @html += "#{content}</#{tag}>\n"
    else
      @html += "</#{tag}>\n"
    end
  end

  def to_s
    @html
  end
end

def html(&block)
  builder = HtmlDsl.new
  builder.instance_eval(&block)
  builder.to_s
end

result = html do
  div(class: "container") do
    h1 "สวัสดี Ruby DSL"
    p("เรียนรู้ Blocks ใน Ruby", class: "intro")
    ul do
      li "Blocks"
      li "Procs"
      li "Lambdas"
    end
  end
end

puts result
```

---

## Step 188–190: Advanced Block Patterns

### Step 188: Fiber — Coroutines

```ruby
# Fiber คือ lightweight coroutine ที่ใช้ yield ในการ pause/resume
fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield "first"
  
  puts "Step 2"
  Fiber.yield "second"
  
  puts "Step 3"
  "third"
end

puts fiber.resume  # Step 1 → first
puts fiber.resume  # Step 2 → second
puts fiber.resume  # Step 3 → third
# fiber.resume   # FiberError: dead fiber called

# ใช้สร้าง infinite generator
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
# 0 1 1 2 3 5 8 13 21 34
```

### Step 189: Enumerator.new กับ Blocks

```ruby
# สร้าง custom Enumerator
def infinite_primes
  Enumerator.new do |yielder|
    candidates = (2..Float::INFINITY)
    primes = []
    
    candidates.each do |n|
      is_prime = primes.none? { |p| n % p == 0 }
      if is_prime
        primes << n
        yielder << n
      end
    end
  end
end

puts infinite_primes.first(10).inspect
# [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]

puts infinite_primes.lazy.select { |p| p > 50 }.first(5).inspect
# [53, 59, 61, 67, 71]

# Enumerator::Chain (Ruby 2.6+)
natural_numbers = Enumerator.new { |y| n = 1; loop { y << n; n += 1 } }
squares = natural_numbers.lazy.map { |n| n ** 2 }
puts squares.first(5).inspect  # [1, 4, 9, 16, 25]
```

### Step 190: Proc Memoization

```ruby
# Memoize lambda calls
def memoize(func)
  cache = {}
  lambda do |*args|
    key = args.hash
    cache[key] ||= func.call(*args)
  end
end

# Fibonacci naive (exponential time)
fib_naive = lambda { |n| n <= 1 ? n : fib_naive.(n-1) + fib_naive.(n-2) }

# Memoized Fibonacci
fib_memo = nil  # ต้องประกาศก่อน
fib_memo = memoize(lambda { |n|
  n <= 1 ? n : fib_memo.(n-1) + fib_memo.(n-2)
})

puts fib_memo.(50)  # 12586269025 (เร็วมาก!)

# Generic memoize decorator
class Proc
  def memoized
    cache = {}
    ->(*args) { cache[args] ||= call(*args) }
  end
end

expensive = ->(n) {
  sleep(0.001)  # simulate expensive operation
  n ** 3
}.memoized

[1, 2, 3, 1, 2, 3].each { |n| puts expensive.(n) }
# 1, 8, 27, 1(cached), 8(cached), 27(cached)
```

---

## Step 191–195: Real-world Patterns

### Step 191: Strategy Pattern

```ruby
# ใช้ Lambda แทน Strategy objects
class Sorter
  STRATEGIES = {
    asc:        ->(a, b) { a <=> b },
    desc:       ->(a, b) { b <=> a },
    by_length:  ->(a, b) { a.length <=> b.length },
    random:     ->(a, b) { [-1, 0, 1].sample },
    case_insensitive: ->(a, b) { a.downcase <=> b.downcase }
  }

  def initialize(strategy = :asc)
    @strategy = STRATEGIES[strategy] || strategy
  end

  def sort(collection)
    collection.sort(&@strategy)
  end
end

words = ["Banana", "apple", "Cherry", "date"]

[:asc, :desc, :by_length, :case_insensitive].each do |s|
  puts "#{s}: #{Sorter.new(s).sort(words).inspect}"
end
# asc: ["Banana", "Cherry", "apple", "date"]
# desc: ["date", "apple", "Cherry", "Banana"]
# by_length: ["date", "apple", "Banana", "Cherry"]
# case_insensitive: ["apple", "Banana", "Cherry", "date"]
```

### Step 192: Command Pattern

```ruby
class CommandHistory
  def initialize
    @history = []
    @redo_stack = []
  end

  def execute(command)
    @history << command
    @redo_stack.clear
    command[:do].call
  end

  def undo
    return puts "ไม่มีคำสั่งที่จะ undo" if @history.empty?
    command = @history.pop
    @redo_stack << command
    command[:undo].call
  end

  def redo
    return puts "ไม่มีคำสั่งที่จะ redo" if @redo_stack.empty?
    command = @redo_stack.pop
    @history << command
    command[:do].call
  end
end

text = ""
history = CommandHistory.new

def make_type_command(text_ref, chars)
  {
    do:   -> { text_ref << chars; puts "พิมพ์: '#{chars}' → '#{text_ref}'" },
    undo: -> { text_ref.slice!(-chars.length..-1); puts "undo → '#{text_ref}'" }
  }
end

# ต้องใช้ reference wrapper เพื่อให้ lambda edit ตัวแปรได้
# (simplified version)
```

### Step 193: Observer Pattern

```ruby
module Observable
  def self.included(base)
    base.instance_variable_set(:@observers, {})
    base.extend(ClassMethods)
  end

  module ClassMethods
    def observers
      @observers ||= {}
    end
  end

  def subscribe(event, &observer)
    self.class.observers[event] ||= []
    self.class.observers[event] << observer
    observer  # คืน observer เพื่อ unsubscribe ได้
  end

  def unsubscribe(event, observer)
    self.class.observers[event]&.delete(observer)
  end

  def notify(event, *args)
    self.class.observers[event]&.each { |obs| obs.call(*args) }
  end
end

class Stock
  include Observable
  attr_reader :symbol, :price

  def initialize(symbol, price)
    @symbol = symbol
    @price = price
  end

  def price=(new_price)
    old_price = @price
    @price = new_price
    notify(:price_changed, symbol, old_price, new_price)
    notify(:price_up, symbol, new_price - old_price) if new_price > old_price
    notify(:price_down, symbol, old_price - new_price) if new_price < old_price
  end
end

aapl = Stock.new("AAPL", 150.0)

aapl.subscribe(:price_changed) do |symbol, old, new_price|
  change = ((new_price - old) / old * 100).round(2)
  puts "#{symbol}: #{old} → #{new_price} (#{change > 0 ? '+' : ''}#{change}%)"
end

aapl.subscribe(:price_up) { |symbol, gain| puts "📈 #{symbol} +#{gain}" }
aapl.subscribe(:price_down) { |symbol, loss| puts "📉 #{symbol} -#{loss}" }

aapl.price = 155.0  # AAPL: 150.0 → 155.0 (+3.33%) 📈 AAPL +5.0
aapl.price = 148.0  # AAPL: 155.0 → 148.0 (-4.52%) 📉 AAPL -7.0
```

---

## Step 194–195: Practical Examples

### Step 194: Middleware Pattern

```ruby
# Middleware chain ด้วย Proc/Lambda
class MiddlewareStack
  def initialize
    @middlewares = []
  end

  def use(&middleware)
    @middlewares << middleware
    self
  end

  def call(request, &final_handler)
    build_chain(final_handler).call(request)
  end

  private

  def build_chain(final)
    @middlewares.reverse.reduce(final) do |next_handler, middleware|
      ->(req) { middleware.call(req, next_handler) }
    end
  end
end

app = MiddlewareStack.new

# Logger middleware
app.use do |request, next_handler|
  puts "[LOG] #{request[:method]} #{request[:path]}"
  response = next_handler.call(request)
  puts "[LOG] Response: #{response[:status]}"
  response
end

# Auth middleware
app.use do |request, next_handler|
  if request[:token] == "valid_token"
    next_handler.call(request)
  else
    { status: 401, body: "Unauthorized" }
  end
end

# Rate limiter
request_counts = Hash.new(0)
app.use do |request, next_handler|
  ip = request[:ip] || "unknown"
  request_counts[ip] += 1
  if request_counts[ip] > 100
    { status: 429, body: "Too Many Requests" }
  else
    next_handler.call(request)
  end
end

# Final handler
final = ->(request) { { status: 200, body: "Hello!" } }

response = app.call({ method: "GET", path: "/api/users", token: "valid_token", ip: "1.2.3.4" }, &final)
puts response.inspect

response = app.call({ method: "GET", path: "/api/secret", token: "invalid", ip: "5.6.7.8" }, &final)
puts response.inspect
```

### Step 195: Functional Composition Library

```ruby
module Functional
  # compose: f.compose(g).call(x) == f(g(x))
  def self.compose(*fns)
    fns.reduce { |f, g| ->(x) { f.call(g.call(x)) } }
  end

  # pipe: แบบ compose แต่ลำดับกลับ
  def self.pipe(*fns)
    fns.reduce { |f, g| ->(x) { g.call(f.call(x)) } }
  end

  # partial: partial application
  def self.partial(fn, *partial_args)
    ->(*args) { fn.call(*partial_args, *args) }
  end

  # memoize
  def self.memoize(fn)
    cache = {}
    ->(*args) { cache[args] ||= fn.call(*args) }
  end

  # once: เรียกได้ครั้งเดียว
  def self.once(fn)
    called = false
    result = nil
    ->(*args) {
      unless called
        result = fn.call(*args)
        called = true
      end
      result
    }
  end

  # throttle: จำกัดการเรียก
  def self.throttle(fn, interval)
    last_called = nil
    ->(*args) {
      now = Time.now
      if last_called.nil? || now - last_called >= interval
        last_called = now
        fn.call(*args)
      end
    }
  end
end

# ทดสอบ
add = ->(a, b) { a + b }
add5 = Functional.partial(add, 5)
puts add5.(10)  # 15

double = ->(n) { n * 2 }
square = ->(n) { n ** 2 }
add_one = ->(n) { n + 1 }

# pipe: ทำจากซ้ายไปขวา
transform = Functional.pipe(add_one, double, square)
puts transform.(3)  # square(double(3+1)) = square(8) = 64

# memoize
fib = Functional.memoize(->(n) { n <= 1 ? n : fib.(n-1) + fib.(n-2) })
puts fib.(40)  # 102334155

# once
init = Functional.once(-> { puts "เริ่มต้นระบบ!"; 42 })
puts init.()  # เริ่มต้นระบบ! 42
puts init.()  # 42 (ไม่พิมพ์ซ้ำ)
puts init.()  # 42 (ไม่พิมพ์ซ้ำ)
```

---

## แบบฝึกหัดตอนที่ 10 (30 ข้อ)

### ข้อ 1-5: Block พื้นฐาน

```ruby
# ข้อ 1: สร้าง custom each_pair
def each_pair(array)
  i = 0
  while i < array.length - 1
    yield array[i], array[i + 1]
    i += 2
  end
end

each_pair([1, 2, 3, 4, 5, 6]) { |a, b| puts "#{a} + #{b} = #{a + b}" }
# 1 + 2 = 3
# 3 + 4 = 7
# 5 + 6 = 11

# ข้อ 2: timing block
def measure_time
  start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
  result = yield
  elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
  puts "ใช้เวลา: #{(elapsed * 1000).round(3)} ms"
  result
end

result = measure_time do
  sum = (1..1_000_000).sum
  sum
end
puts "ผลรวม: #{result}"

# ข้อ 3: lazy_map ด้วย Enumerator
def lazy_transform(array, &transform)
  Enumerator.new do |yielder|
    array.each do |item|
      yielder << transform.call(item)
    end
  end.lazy
end

result = lazy_transform([1, 2, 3, 4, 5]) { |n| n ** 2 }
          .select { |n| n > 5 }
          .first(3)
puts result.inspect  # [9, 16, 25]

# ข้อ 4: สร้าง with_logging
def with_logging(name, &block)
  puts "[START] #{name}"
  begin
    result = block.call
    puts "[END] #{name} → #{result.inspect}"
    result
  rescue => e
    puts "[ERROR] #{name}: #{e.message}"
    raise
  end
end

with_logging("calculate") { 2 + 2 }
# [START] calculate
# [END] calculate → 4

# ข้อ 5: สร้าง cached block
def cache_result(key, cache = {}, &block)
  cache[key] ||= block.call
end

cache = {}
3.times do |i|
  result = cache_result("expensive_#{i % 2}", cache) {
    puts "Computing #{i % 2}..."
    i % 2 * 100
  }
  puts "Result: #{result}"
end
# Computing 0...
# Result: 0
# Computing 1...
# Result: 100
# Result: 0 (cached)
```

### ข้อ 6-10: Proc และ Lambda

```ruby
# ข้อ 6: Function Composition ด้วย >>
double  = ->(x) { x * 2 }
add_ten = ->(x) { x + 10 }
to_str  = ->(x) { "Result: #{x}" }

pipeline = double >> add_ten >> to_str
puts pipeline.(5)   # Result: 20
puts pipeline.(15)  # Result: 40

# ข้อ 7: Curried Validators
validate = {
  min_length: ->(min) { ->(str) { str.length >= min } },
  max_length: ->(max) { ->(str) { str.length <= max } },
  matches:    ->(pattern) { ->(str) { str.match?(pattern) } },
  not_blank:  ->(_) { ->(str) { !str.strip.empty? } }
}

# สร้าง validator สำหรับ username
username_valid = [
  validate[:not_blank].(:any),
  validate[:min_length].(3),
  validate[:max_length].(20),
  validate[:matches].(/\A[a-z0-9_]+\z/)
]

def validate_all(value, validators)
  validators.all? { |v| v.call(value) }
end

puts validate_all("alice_123", username_valid)  # true
puts validate_all("ab", username_valid)         # false (too short)
puts validate_all("Alice 123", username_valid)  # false (has space/capital)

# ข้อ 8: Proc Memoization
def memoize(&block)
  cache = {}
  ->(*args) { cache[args] ||= block.call(*args) }
end

fib = memoize { |n| n <= 1 ? n : fib.(n-1) + fib.(n-2) }
puts fib.(35)  # 9227465

# ข้อ 9: Event Queue ด้วย Procs
class EventQueue
  def initialize
    @queue = []
    @handlers = {}
  end

  def on(event, &handler)
    @handlers[event] ||= []
    @handlers[event] << handler
  end

  def emit(event, *data)
    @queue << [event, data]
    process_queue
  end

  private

  def process_queue
    while event_data = @queue.shift
      event, data = event_data
      @handlers[event]&.each { |h| h.call(*data) }
    end
  end
end

eq = EventQueue.new
eq.on(:message) { |msg| puts "Received: #{msg}" }
eq.on(:message) { |msg| puts "Logged: #{msg}" }
eq.on(:error) { |err| puts "Error: #{err}" }

eq.emit(:message, "Hello!")
eq.emit(:error, "Something failed")

# ข้อ 10: Pipeline Builder
class PipelineBuilder
  def initialize
    @stages = []
  end

  def add_stage(name, &transform)
    @stages << { name: name, transform: transform }
    self
  end

  def build
    stages = @stages.dup
    ->(input) {
      stages.reduce(input) do |data, stage|
        begin
          result = stage[:transform].call(data)
          result
        rescue => e
          raise "Pipeline failed at '#{stage[:name]}': #{e.message}"
        end
      end
    }
  end
end

text_processor = PipelineBuilder.new
  .add_stage("strip") { |s| s.strip }
  .add_stage("downcase") { |s| s.downcase }
  .add_stage("tokenize") { |s| s.split(/\W+/) }
  .add_stage("filter") { |tokens| tokens.reject(&:empty?) }
  .add_stage("sort") { |tokens| tokens.sort.uniq }
  .build

result = text_processor.("  Hello, World! Hello, Ruby World!  ")
puts result.inspect  # ["hello", "ruby", "world"]
```

### ข้อ 11-15: Closure และ State

```ruby
# ข้อ 11: Functional Queue
def make_queue
  items = []
  {
    enqueue: ->(item) { items << item; nil },
    dequeue: -> { items.shift },
    peek:    -> { items.first },
    size:    -> { items.length },
    empty?:  -> { items.empty? },
    to_a:    -> { items.dup }
  }
end

q = make_queue
q[:enqueue].("apple")
q[:enqueue].("banana")
q[:enqueue].("cherry")
puts q[:size].()      # 3
puts q[:dequeue].()   # apple
puts q[:peek].()      # banana
puts q[:to_a].call.inspect  # ["banana", "cherry"]

# ข้อ 12: State Machine ด้วย Lambda
def make_traffic_light
  states = {
    red:    { next: :green,  timer: 30, action: -> { puts "🔴 หยุด!" } },
    green:  { next: :yellow, timer: 25, action: -> { puts "🟢 ไปได้!" } },
    yellow: { next: :red,    timer: 5,  action: -> { puts "🟡 ระวัง!" } }
  }
  
  current = :red
  
  {
    current: -> { current },
    advance: -> {
      state = states[current]
      state[:action].call
      current = state[:next]
    },
    timer: -> { states[current][:timer] }
  }
end

light = make_traffic_light
5.times do
  puts "สัญญาณ: #{light[:current].call} (#{light[:timer].call} วินาที)"
  light[:advance].call
  puts "---"
end

# ข้อ 13: Retry With Backoff
def with_retry(max_attempts: 3, base_delay: 1, &operation)
  attempts = 0
  delays = (0...max_attempts).map { |i| base_delay * (2 ** i) }
  
  loop do
    attempts += 1
    begin
      return yield(attempts)
    rescue => e
      if attempts >= max_attempts
        raise "ล้มเหลวหลังจาก #{max_attempts} ครั้ง: #{e.message}"
      end
      delay = delays[attempts - 1]
      puts "ครั้งที่ #{attempts} ล้มเหลว (#{e.message}), รอ #{delay}s..."
      sleep(0.01)  # ใช้ 0.01 แทน delay จริงสำหรับ demo
    end
  end
end

result = with_retry(max_attempts: 3) do |attempt|
  raise "Connection failed" if attempt < 3
  "สำเร็จในครั้งที่ #{attempt}!"
end
puts result

# ข้อ 14: Functional Option Type
class Option
  def self.some(value)
    new(value, true)
  end

  def self.none
    new(nil, false)
  end

  def initialize(value, present)
    @value = value
    @present = present
  end

  def present?; @present; end
  def empty?; !@present; end

  def map(&block)
    @present ? Option.some(block.call(@value)) : self
  end

  def flat_map(&block)
    @present ? block.call(@value) : self
  end

  def or_else(default = nil, &block)
    @present ? @value : (block ? block.call : default)
  end

  def on_some(&block)
    block.call(@value) if @present
    self
  end

  def on_none(&block)
    block.call unless @present
    self
  end
end

def find_user(id)
  users = { 1 => { name: "Alice" }, 2 => { name: "Bob" } }
  users[id] ? Option.some(users[id]) : Option.none
end

find_user(1)
  .map { |u| u[:name].upcase }
  .on_some { |name| puts "Found: #{name}" }
  .on_none { puts "Not found" }
# Found: ALICE

find_user(99)
  .map { |u| u[:name].upcase }
  .on_some { |name| puts "Found: #{name}" }
  .on_none { puts "Not found" }
# Not found

# ข้อ 15: Lazy Object
class LazyProxy
  def initialize(&initializer)
    @initializer = initializer
    @target = nil
    @initialized = false
  end

  def method_missing(method, *args, &block)
    unless @initialized
      @target = @initializer.call
      @initialized = true
    end
    @target.send(method, *args, &block)
  end

  def respond_to_missing?(method, include_private = false)
    true
  end
end

# สร้าง lazy database connection
db = LazyProxy.new do
  puts "เชื่อมต่อ Database..."
  { connected: true, query_count: 0 }
end

puts "ยังไม่เชื่อมต่อ"
# ไม่มีอะไรพิมพ์
puts db[:connected]  # เชื่อมต่อ Database... true
puts db[:connected]  # true (ไม่เชื่อมต่อซ้ำ)
```

### ข้อ 16-20: Advanced Patterns

```ruby
# ข้อ 16: Y Combinator (สำหรับ anonymous recursion)
Y = ->(f) { ->(x) { f.call(->(v) { x.(x).(v) }) }.call(->(x) { f.call(->(v) { x.(x).(v) }) }) }

factorial = Y.(->(f) { ->(n) { n <= 1 ? 1 : n * f.(n - 1) } })
puts factorial.(10)  # 3628800

fib = Y.(->(f) { ->(n) { n <= 1 ? n : f.(n-1) + f.(n-2) } })
puts fib.(10)  # 55

# ข้อ 17: สร้าง Observable Value
class Observable
  def initialize(value)
    @value = value
    @watchers = []
  end

  def value
    @value
  end

  def value=(new_val)
    old_val = @value
    @value = new_val
    @watchers.each { |w| w.call(new_val, old_val) } if new_val != old_val
  end

  def watch(&block)
    @watchers << block
    -> { @watchers.delete(block) }  # คืน unwatch function
  end
end

counter = Observable.new(0)

unwatch = counter.watch { |new_val, old_val|
  puts "เปลี่ยนจาก #{old_val} เป็น #{new_val}"
}

counter.value = 1  # เปลี่ยนจาก 0 เป็น 1
counter.value = 5  # เปลี่ยนจาก 1 เป็น 5
unwatch.call       # หยุด watch
counter.value = 10 # (ไม่มีการแจ้งเตือน)

# ข้อ 18: Template Method Pattern ด้วย Blocks
class ReportGenerator
  def generate(data, title:, &formatter)
    formatter ||= method(:default_format)
    
    lines = ["=" * 40, "  #{title}", "=" * 40]
    data.each { |item| lines << formatter.call(item) }
    lines << "=" * 40
    lines.join("\n")
  end

  private

  def default_format(item)
    "• #{item}"
  end
end

generator = ReportGenerator.new
report = generator.generate(
  ["Alice: 92", "Bob: 78", "Charlie: 85"],
  title: "ผลการสอบ"
) { |item| "  ✓ #{item}" }

puts report

# ข้อ 19: Promise-like Async Pattern
class SimplePromise
  def initialize(&work)
    @callbacks = []
    @error_handlers = []
    @result = nil
    @error = nil
    @resolved = false
    
    begin
      @result = work.call
      @resolved = true
      @callbacks.each { |cb| cb.call(@result) }
    rescue => e
      @error = e
      @error_handlers.each { |h| h.call(e) }
    end
  end

  def then(&callback)
    if @resolved
      callback.call(@result)
    else
      @callbacks << callback
    end
    self
  end

  def catch(&handler)
    if @error
      handler.call(@error)
    else
      @error_handlers << handler
    end
    self
  end
end

SimplePromise.new { 42 }
  .then { |v| puts "Success: #{v}" }
  .catch { |e| puts "Error: #{e}" }
# Success: 42

SimplePromise.new { raise "Something went wrong" }
  .then { |v| puts "Success: #{v}" }
  .catch { |e| puts "Caught: #{e.message}" }
# Caught: Something went wrong

# ข้อ 20: Finite State Machine
class FSM
  def initialize(initial_state)
    @state = initial_state
    @transitions = {}
    @entry_actions = {}
    @exit_actions = {}
  end

  def on(event, from:, to:, guard: nil, action: nil)
    key = [from, event]
    @transitions[key] = {
      to: to,
      guard: guard,
      action: action
    }
    self
  end

  def on_entry(state, &block)
    @entry_actions[state] = block
    self
  end

  def on_exit(state, &block)
    @exit_actions[state] = block
    self
  end

  def trigger(event, **context)
    key = [@state, event]
    transition = @transitions[key]
    
    return false unless transition
    return false if transition[:guard] && !transition[:guard].call(context)
    
    @exit_actions[@state]&.call
    transition[:action]&.call(context)
    @state = transition[:to]
    @entry_actions[@state]&.call
    
    true
  end

  def state; @state; end
end

# Traffic Light FSM
light = FSM.new(:red)
  .on(:timer, from: :red,    to: :green)
  .on(:timer, from: :green,  to: :yellow)
  .on(:timer, from: :yellow, to: :red)
  .on_entry(:red)    { puts "🔴 หยุด" }
  .on_entry(:green)  { puts "🟢 ไป" }
  .on_entry(:yellow) { puts "🟡 ระวัง" }

6.times { light.trigger(:timer) }
```

### ข้อ 21-30: โจทย์ขั้นสูงพิเศษ

```ruby
# ข้อ 21: Functional Linked List ด้วย Lambda
cons = ->(head, tail) { ->(f) { f.(head, tail) } }
head = ->(list) { list.(->(h, _) { h }) }
tail = ->(list) { list.(->(_, t) { t }) }
empty_list = nil

# สร้าง list [1, 2, 3]
list = cons.(1, cons.(2, cons.(3, empty_list)))
puts head.(list)          # 1
puts head.(tail.(list))   # 2

# ข้อ 22: สร้าง Tee operator
def tee(&side_effect)
  ->(value) {
    side_effect.call(value)
    value  # pass through
  }
end

pipeline = 
  ->(n) { n * 2 } >>
  tee { |n| puts "หลัง double: #{n}" } >>
  ->(n) { n + 10 } >>
  tee { |n| puts "หลัง add: #{n}" }

result = pipeline.(5)
puts "Final: #{result}"
# หลัง double: 10
# หลัง add: 20
# Final: 20

# ข้อ 23: ทำ Trampoline สำหรับ tail recursion
def trampoline(f)
  ->(*args) {
    result = f.call(*args)
    result = result.call while result.is_a?(Proc)
    result
  }
end

# Tail-recursive factorial ด้วย trampoline
factorial_tc = trampoline(->(n, acc = 1) {
  n <= 1 ? acc : -> { factorial_tc.(n - 1, n * acc) }
})

puts factorial_tc.(100)  # ใหญ่มาก แต่ไม่ stack overflow

# ข้อ 24: สร้าง Either Monad
class Either
  class Right < Either
    def initialize(value); @value = value; end
    def map(&f); Right.new(f.call(@value)); end
    def flat_map(&f); f.call(@value); end
    def right?; true; end
    def value_or(_); @value; end
    def to_s; "Right(#{@value})"; end
  end

  class Left < Either
    def initialize(error); @error = error; end
    def map(&f); self; end
    def flat_map(&f); self; end
    def right?; false; end
    def value_or(default); default; end
    def to_s; "Left(#{@error})"; end
  end

  def self.right(v) = Right.new(v)
  def self.left(e)  = Left.new(e)
end

def safe_divide(a, b)
  b == 0 ? Either.left("Division by zero") : Either.right(a.to_f / b)
end

result = safe_divide(10, 2)
  .map { |v| v * 2 }
  .map { |v| "Result: #{v}" }

puts result  # Right(Result: 10.0)

result = safe_divide(10, 0)
  .map { |v| v * 2 }
  .map { |v| "Result: #{v}" }

puts result  # Left(Division by zero)

# ข้อ 25-30: ให้ผู้เรียนลองทำเอง
# ข้อ 25: สร้าง Reader Monad สำหรับ dependency injection
# ข้อ 26: สร้าง Writer Monad สำหรับ logging
# ข้อ 27: สร้าง State Monad
# ข้อ 28: สร้าง continuation-passing style transformer
# ข้อ 29: สร้าง transducer-based pipeline
# ข้อ 30: สร้าง Actor model ด้วย Fiber
```

---

## สรุป Blocks, Procs, Lambdas ใน Ruby

### ตารางเปรียบเทียบ

| Feature | Block | Proc | Lambda |
|---------|-------|------|--------|
| เป็น Object | ❌ | ✅ | ✅ |
| เก็บใน Variable | ❌ | ✅ | ✅ |
| Strict Arity | N/A | ❌ | ✅ |
| return behavior | ออกจาก method | ออกจาก method | ออกจาก lambda |
| lambda? | N/A | false | true |
| สร้างด้วย | `{}` `do..end` | `Proc.new {}` `proc {}` | `lambda {}` `-> {}` |

### เมื่อไหร่ใช้อะไร?

```
Block:
├── ใช้กับ iterators (each, map, select)
├── ใช้กับ resource management (File.open)
└── ใช้เมื่อ pass inline computation

Proc:
├── ใช้เมื่อต้องการ store block ใน variable
├── ใช้เป็น callback ที่ lenient เรื่อง arguments
└── ใช้กับ closures ที่ต้องการ modify outer variables

Lambda:
├── ใช้เมื่อต้องการ strict argument checking
├── ใช้แทน method ที่ portable
└── ใช้ใน functional composition (>>, <<, curry)
```

**หลักการสำคัญ:**
1. Block เป็นหัวใจของ Ruby — ใช้ทุกที่ที่มี iterator
2. Proc และ Lambda คือ "first-class functions" ใน Ruby
3. Closure จำตัวแปรจาก scope ที่สร้าง — ทรงพลังมาก
4. `&:symbol` (Symbol to Proc) เป็น pattern ที่นิยมมากใน Ruby
5. Lambda เหมาะกับ functional programming มากกว่า Proc
6. `curry` ทำให้ partial application ง่ายขึ้น
7. `>>` และ `<<` ทำให้ compose functions ง่ายขึ้น (Ruby 2.6+)

> ⬅️ [ตอนที่ 9: Methods](part-09-methods.md) | ➡️ ตอนที่ 11: Classes และ Objects
