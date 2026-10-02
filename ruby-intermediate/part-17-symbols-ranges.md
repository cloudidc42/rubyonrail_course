# Part 17: Symbols และ Ranges ใน Ruby

## ขั้นตอนที่ 346-365: Symbol และ Range อย่างละเอียด

---

## บทนำ

Symbol และ Range เป็น data types สำคัญใน Ruby ที่ใช้บ่อยมากในชีวิตประจำวัน Symbol นั้นเบากว่า String มาก ส่วน Range ช่วยให้เขียน code ที่อ่านง่ายและมีประสิทธิภาพ

---

## ขั้นตอนที่ 346: Symbol คืออะไร?

Symbol คือ identifier ที่ไม่เปลี่ยนแปลง (immutable) และเก็บไว้ในหน่วยความจำเพียงชิ้นเดียว

```ruby
# การสร้าง Symbol
sym1 = :hello
sym2 = :world
sym3 = :"hello world"  # Symbol ที่มีช่องว่าง
sym4 = :"user-name"    # Symbol ที่มีขีด

puts sym1.class        # => Symbol
puts sym1              # => hello
puts sym1.inspect      # => :hello

# Symbol เหมือนกันจะมี object_id เดียวกัน
a = :ruby
b = :ruby
puts a.object_id == b.object_id   # => true (same object!)

# String ต่างกัน
s1 = "ruby"
s2 = "ruby"
puts s1.object_id == s2.object_id  # => false (different objects)
```

---

## ขั้นตอนที่ 347: Symbol vs String - Performance

```ruby
require 'benchmark'
require 'objspace'

# Symbol ใช้หน่วยความจำน้อยกว่า
sym = :ruby
str = "ruby"

puts "Symbol size: #{ObjectSpace.memsize_of(sym)} bytes"
puts "String size: #{ObjectSpace.memsize_of(str)} bytes"

# Comparison performance
n = 1_000_000
Benchmark.bm(10) do |x|
  x.report("Symbol ==:") { n.times { :ruby == :ruby } }
  x.report("String ==:") { n.times { "ruby" == "ruby" } }
end

# Symbol comparison เร็วกว่า เพราะเปรียบเทียบ object_id โดยตรง
# String comparison ช้ากว่า เพราะต้องเปรียบเทียบทีละ character

# Symbol เหมาะสำหรับ:
# - Hash keys
# - Method names
# - Options/configurations
# - Named parameters

# Hash กับ Symbol keys เร็วกว่า String keys
hash_sym = { name: "Ruby", version: 3 }
hash_str = { "name" => "Ruby", "version" => 3 }

Benchmark.bm(15) do |x|
  x.report("Symbol key:   ") { n.times { hash_sym[:name] } }
  x.report("String key:   ") { n.times { hash_str["name"] } }
end
```

---

## ขั้นตอนที่ 348: Symbol Immutability

```ruby
# Symbol ไม่สามารถเปลี่ยนแปลงได้ (immutable)
sym = :hello

# ไม่มี method แก้ไข Symbol
# sym << " world"  # => NoMethodError!
# sym.upcase!       # => ไม่มี bang version

# แต่สามารถสร้าง Symbol ใหม่ได้
new_sym = sym.to_s.upcase.to_sym
puts new_sym   # => :HELLO

# เหตุใด immutability จึงสำคัญ?
# - Thread-safe: หลาย thread ใช้ Symbol เดียวกันได้อย่างปลอดภัย
# - Predictable: ค่าไม่เปลี่ยน ทำให้ debug ง่าย
# - Memory efficient: เก็บแค่ครั้งเดียว

# Symbol pool - ทุก Symbol มีอยู่ใน Symbol table
puts Symbol.all_symbols.count   # เห็นว่ามี symbols กี่ตัว
puts Symbol.all_symbols.include?(:hello)  # => true (หลังจากสร้าง :hello)
```

---

## ขั้นตอนที่ 349: Symbol Methods

```ruby
sym = :hello_world

# to_s / to_proc / to_sym
puts sym.to_s           # => "hello_world"
puts sym.inspect        # => ":hello_world"
puts "hello".to_sym     # => :hello

# length / size
puts sym.length   # => 11
puts sym.size     # => 11

# upcase / downcase / capitalize
puts :hello.upcase      # => :HELLO
puts :WORLD.downcase    # => :world
puts :hello.capitalize  # => :Hello

# id2name (เหมือน to_s)
puts :ruby.id2name   # => "ruby"

# match
puts :hello.match(/ell/)   # => #<MatchData "ell">

# encoding
puts :hello.encoding   # => UTF-8

# empty?
puts :"".empty?   # => true
puts :a.empty?    # => false

# ตัวอย่างการใช้งาน
methods = [:upcase, :downcase, :reverse, :length]
word = "Ruby"

methods.each do |method|
  puts "#{word}.#{method} => #{word.send(method)}"
end
```

---

## ขั้นตอนที่ 350: Symbol#to_proc

Symbol#to_proc เป็น feature ที่ทรงพลังมากใน Ruby

```ruby
# &:method_name แปลง symbol เป็น proc ที่เรียก method นั้น
words = ["hello", "world", "ruby"]

# แบบ verbose
puts words.map { |w| w.upcase }.inspect

# แบบ elegant ด้วย Symbol#to_proc
puts words.map(&:upcase).inspect   # => ["HELLO", "WORLD", "RUBY"]

# ทำงานได้กับ method ทุกตัว
numbers = [1, 2, 3, 4, 5]
puts numbers.map(&:to_s).inspect    # => ["1", "2", "3", "4", "5"]
puts numbers.select(&:odd?).inspect # => [1, 3, 5]

# กับ strings
puts ["  hello  ", "  world  "].map(&:strip).inspect
# => ["hello", "world"]

# กับ hashes
pairs = [[:a, 1], [:b, 2], [:c, 3]]
hash = pairs.to_h
puts hash.inspect  # => {:a=>1, :b=>2, :c=>3}

# ใช้กับ reduce
puts [1, 2, 3, 4, 5].reduce(:+)   # => 15
puts [1, 2, 3, 4, 5].reduce(:*)   # => 120
puts ["a", "b", "c"].reduce(:+)   # => "abc"

# Custom class กับ Symbol#to_proc
class Product
  attr_reader :name, :price
  
  def initialize(name, price)
    @name = name
    @price = price
  end
  
  def expensive?
    @price > 1000
  end
end

products = [
  Product.new("Book", 300),
  Product.new("Laptop", 30000),
  Product.new("Pen", 50)
]

puts products.map(&:name).inspect
# => ["Book", "Laptop", "Pen"]

puts products.select(&:expensive?).map(&:name).inspect
# => ["Laptop"]
```

---

## ขั้นตอนที่ 351: การแปลงระหว่าง Symbol และ String

```ruby
# String to Symbol
str = "hello"
sym = str.to_sym
puts sym.class   # => Symbol
puts sym         # => hello

# Symbol to String
sym = :world
str = sym.to_s
puts str.class   # => String
puts str         # => "world"

# ตัวอย่างการใช้งาน
def method_name_to_label(method_name)
  method_name.to_s.gsub('_', ' ').capitalize
end

puts method_name_to_label(:first_name)  # => "First name"
puts method_name_to_label(:date_of_birth)  # => "Date of birth"

# Dynamic method calling
class Calculator
  def add(a, b) = a + b
  def subtract(a, b) = a - b
  def multiply(a, b) = a * b
end

calc = Calculator.new
operations = [:add, :subtract, :multiply]

operations.each do |op|
  puts "#{op}: #{calc.send(op, 10, 3)}"
end
# => add: 13
# => subtract: 7
# => multiply: 30

# Symbol ใน Hash
# Ruby อนุญาตให้ใช้ shorthand syntax ตั้งแต่ Ruby 1.9
old_style = { :name => "Ruby", :version => 3 }
new_style = { name: "Ruby", version: 3 }

puts old_style == new_style   # => true (เหมือนกัน!)
```

---

## ขั้นตอนที่ 352: Symbols ใน Practical Use Cases

```ruby
# 1. ใช้เป็น Hash keys (ที่นิยมที่สุด)
user = {
  name: "สมชาย",
  age: 25,
  email: "somchai@example.com",
  role: :admin  # Symbol เป็น value ด้วยได้
}

puts user[:name]  # => "สมชาย"
puts user[:role]  # => admin

# 2. Enum-like behavior
module Status
  PENDING  = :pending
  ACTIVE   = :active
  INACTIVE = :inactive
end

user_status = Status::ACTIVE
case user_status
when :pending  then puts "รอการอนุมัติ"
when :active   then puts "ใช้งานอยู่"
when :inactive then puts "ปิดการใช้งาน"
end

# 3. Method options
def create_user(name, options = {})
  role    = options.fetch(:role, :user)
  active  = options.fetch(:active, true)
  
  puts "สร้าง #{role}: #{name} (active: #{active})"
end

create_user("สมหญิง", role: :admin, active: true)
create_user("สมศรี")

# 4. Callbacks
class EventEmitter
  def initialize
    @listeners = {}
  end

  def on(event, &block)
    @listeners[event] ||= []
    @listeners[event] << block
  end

  def emit(event, *args)
    @listeners[event]&.each { |block| block.call(*args) }
  end
end

emitter = EventEmitter.new
emitter.on(:data_loaded) { |data| puts "โหลดข้อมูล: #{data}" }
emitter.on(:error) { |msg| puts "Error: #{msg}" }

emitter.emit(:data_loaded, "100 records")
emitter.emit(:error, "Connection failed")
```

---

## ขั้นตอนที่ 353: Range พื้นฐาน

```ruby
# Inclusive range (..) - รวม endpoint ทั้งสอง
inclusive = (1..5)
puts inclusive.to_a.inspect   # => [1, 2, 3, 4, 5]

# Exclusive range (...) - ไม่รวม endpoint สุดท้าย
exclusive = (1...5)
puts exclusive.to_a.inspect   # => [1, 2, 3, 4]

# Range กับ characters
puts ('a'..'e').to_a.inspect   # => ["a", "b", "c", "d", "e"]
puts ('A'..'E').to_a.inspect   # => ["A", "B", "C", "D", "E"]

# Range กับ String
str_range = ('aa'..'ae')
puts str_range.to_a.inspect   # => ["aa", "ab", "ac", "ad", "ae"]

# Range class methods
r = (1..10)
puts r.class      # => Range
puts r.begin      # => 1 (หรือ r.first)
puts r.end        # => 10 (หรือ r.last)
puts r.exclude_end?   # => false (inclusive)

r2 = (1...10)
puts r2.exclude_end?  # => true (exclusive)
```

---

## ขั้นตอนที่ 354: Range Methods

```ruby
r = (1..10)

# include? / member? / cover?
puts r.include?(5)    # => true
puts r.include?(11)   # => false
puts r.member?(3)     # => true
puts r.cover?(5.5)    # => true (cover? ไม่ enumerate ทั้งหมด - เร็วกว่า)

# include? vs cover?
float_range = (1.0..10.0)
puts float_range.include?(5.5)   # => true
puts float_range.cover?(5.5)     # => true

# แต่สำหรับ non-integer ranges, cover? เร็วกว่ามาก
# cover? ใช้ comparison operators
# include? enumerate ทั้งหมด

# size / count
puts r.size     # => 10
puts r.count    # => 10

# first / last กับ argument
puts r.first        # => 1
puts r.first(3).inspect   # => [1, 2, 3]
puts r.last         # => 10
puts r.last(3).inspect    # => [8, 9, 10]

# min / max
puts r.min   # => 1
puts r.max   # => 10

# sum
puts r.sum   # => 55

# each
r.each { |n| print "#{n} " }
puts

# to_a
puts (1..5).to_a.inspect   # => [1, 2, 3, 4, 5]
```

---

## ขั้นตอนที่ 355: Step Ranges

```ruby
# step method - วนด้วย step ที่กำหนด
(1..10).step(2) { |n| print "#{n} " }
puts   # => 1 3 5 7 9

# Float ranges
(0.0..1.0).step(0.25) { |n| print "#{n} " }
puts   # => 0.0 0.25 0.5 0.75 1.0

# step คืนค่า Enumerator
enumerator = (1..10).step(3)
puts enumerator.to_a.inspect   # => [1, 4, 7, 10]

# การใช้งาน: สร้าง sequence
# ทุก 5 นาที
minutes = (0..60).step(5).to_a
puts minutes.inspect
# => [0, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 55, 60]

# ทุก 10% ตั้งแต่ 0 ถึง 100
percentages = (0..100).step(10).to_a
puts percentages.inspect
# => [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100]

# สร้าง progress bar
def progress_bar(percent, width = 20)
  filled = (percent * width / 100.0).round
  bar = "█" * filled + "░" * (width - filled)
  "[#{bar}] #{percent}%"
end

(0..100).step(10) do |pct|
  puts progress_bar(pct)
end
```

---

## ขั้นตอนที่ 356: Range ใน Case/When

```ruby
# Range ใน case/when เป็น feature ที่ทรงพลังมาก

def grade(score)
  case score
  when 90..100 then "A"
  when 80...90 then "B"
  when 70...80 then "C"
  when 60...70 then "D"
  else "F"
  end
end

[95, 85, 75, 65, 55].each do |score|
  puts "#{score} => #{grade(score)}"
end

# เกรดภาษาไทย
def bmi_category(bmi)
  case bmi
  when 0...18.5 then "น้ำหนักต่ำกว่าเกณฑ์"
  when 18.5...25 then "น้ำหนักปกติ"
  when 25...30 then "น้ำหนักเกิน"
  else "อ้วน"
  end
end

[17.0, 22.5, 27.0, 32.0].each do |bmi|
  puts "BMI #{bmi}: #{bmi_category(bmi)}"
end

# อายุ category
def age_group(age)
  case age
  when 0..12    then "เด็ก"
  when 13..17   then "วัยรุ่น"
  when 18..59   then "ผู้ใหญ่"
  when 60..Float::INFINITY then "ผู้สูงอายุ"
  end
end

[5, 15, 30, 65].each do |age|
  puts "อายุ #{age}: #{age_group(age)}"
end
```

---

## ขั้นตอนที่ 357: Endless Ranges (Ruby 2.6+)

```ruby
# Endless range (1..) - ไม่มีจุดสิ้นสุด
endless = (1..)
puts endless.class   # => Range

# ใช้ใน case/when
def classify(n)
  case n
  when (..0)  then "ลบหรือศูนย์"
  when 1..10  then "1 ถึง 10"
  when 11..   then "มากกว่า 10"
  end
end

[-1, 0, 5, 15].each { |n| puts "#{n}: #{classify(n)}" }

# ใช้กับ Array slicing
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
puts arr[3..].inspect    # => [4, 5, 6, 7, 8, 9, 10] (จาก index 3 ถึงสุด)
puts arr[..3].inspect    # => [1, 2, 3, 4] (ถึง index 3)
puts arr[2..5].inspect   # => [3, 4, 5, 6]

# select กับ endless range
numbers = (1..100).to_a
puts numbers.select { |n| (50..).include?(n) }.first(5).inspect
# => [50, 51, 52, 53, 54]

# ตัวอย่างจริง: pagination
def paginate(items, page:, per_page: 10)
  start = (page - 1) * per_page
  items[start..(start + per_page - 1)] || []
end

items = (1..50).to_a
puts "Page 1: #{paginate(items, page: 1, per_page: 5).inspect}"
puts "Page 3: #{paginate(items, page: 3, per_page: 5).inspect}"
```

---

## ขั้นตอนที่ 358: Beginless Ranges (Ruby 2.7+)

```ruby
# Beginless range (..5) - ไม่มีจุดเริ่มต้น
beginless = (..5)
puts beginless.include?(5)    # => true
puts beginless.include?(0)    # => true
puts beginless.include?(-100) # => true
puts beginless.include?(6)    # => false

# ใช้ใน case/when
def discount_tier(amount)
  case amount
  when ..999        then "0%"
  when 1000..4999   then "5%"
  when 5000..9999   then "10%"
  when 10000..       then "15%"
  end
end

[500, 2000, 7000, 15000].each do |amt|
  puts "฿#{amt}: ลด #{discount_tier(amt)}"
end

# ใช้กับ Array
arr = ['a', 'b', 'c', 'd', 'e']
puts arr[..2].inspect    # => ["a", "b", "c"]
puts arr[...2].inspect   # => ["a", "b"]

# Filter ด้วย beginless range
products = [
  { name: "Item A", price: 100 },
  { name: "Item B", price: 500 },
  { name: "Item C", price: 1000 },
  { name: "Item D", price: 2000 }
]

cheap = products.select { |p| (..499).include?(p[:price]) }
puts cheap.map { |p| p[:name] }.inspect
# => ["Item A"]

budget = products.select { |p| (500..1000).include?(p[:price]) }
puts budget.map { |p| p[:name] }.inspect
# => ["Item B", "Item C"]
```

---

## ขั้นตอนที่ 359: Range กับ Strings

```ruby
# Range กับ String characters
alpha_lower = ('a'..'z')
puts alpha_lower.to_a.inspect
# => ["a", "b", ..., "z"]

alpha_upper = ('A'..'Z')
puts alpha_upper.include?('M')   # => true

# String range ใช้ succ method
puts 'a'.succ   # => "b"
puts 'z'.succ   # => "aa"
puts '9'.succ   # => "10"

# ใช้ range ในการ validate
def valid_grade?(grade)
  ('A'..'F').include?(grade.upcase)
end

puts valid_grade?('A')   # => true
puts valid_grade?('C')   # => true
puts valid_grade?('G')   # => false

# สร้าง alphabet
puts ('a'..'z').to_a.join(' ')
# => a b c d e f g h i j k l m n o p q r s t u v w x y z

# เลขโรมัน (อย่างง่าย)
digits = ('0'..'9').to_a
letters = [('a'..'z').to_a, ('A'..'Z').to_a].flatten
alphanumeric = digits + letters
puts alphanumeric.length   # => 62

# Custom range คลาสสำหรับ version numbers
# (ต้องการ Comparable)
class Version
  include Comparable
  
  attr_reader :major, :minor, :patch
  
  def initialize(version_string)
    parts = version_string.split('.').map(&:to_i)
    @major = parts[0] || 0
    @minor = parts[1] || 0
    @patch = parts[2] || 0
  end
  
  def <=>(other)
    return @major <=> other.major unless @major == other.major
    return @minor <=> other.minor unless @minor == other.minor
    @patch <=> other.patch
  end
  
  def succ
    Version.new("#{@major}.#{@minor}.#{@patch + 1}")
  end
  
  def to_s
    "#{@major}.#{@minor}.#{@patch}"
  end
end

v1 = Version.new("1.0.0")
v2 = Version.new("2.0.0")
current = Version.new("1.5.3")

puts (v1..v2).include?(current)  # => true
```

---

## ขั้นตอนที่ 360: Range ใน Array Operations

```ruby
arr = [10, 20, 30, 40, 50, 60, 70, 80, 90, 100]

# Slicing ด้วย range
puts arr[2..5].inspect    # => [30, 40, 50, 60]
puts arr[2...5].inspect   # => [30, 40, 50]
puts arr[2..].inspect     # => [30, 40, 50, 60, 70, 80, 90, 100]
puts arr[..3].inspect     # => [10, 20, 30, 40]

# การแทนที่ด้วย range
arr2 = arr.dup
arr2[2..4] = [300, 400, 500]
puts arr2.inspect   # => [10, 20, 300, 400, 500, 60, 70, 80, 90, 100]

# Range ใน each_with_object
result = (1..5).each_with_object([]) do |n, arr|
  arr << n * n
end
puts result.inspect   # => [1, 4, 9, 16, 25]

# Nested ranges
matrix_range = (1..3).flat_map do |row|
  (1..3).map { |col| [row, col] }
end
puts matrix_range.inspect
# => [[1,1],[1,2],[1,3],[2,1],[2,2],[2,3],[3,1],[3,2],[3,3]]

# สร้าง times table
(1..9).each do |i|
  row = (1..9).map { |j| (i * j).to_s.rjust(3) }.join
  puts row
end
```

---

## ขั้นตอนที่ 361: Range กับ each_slice และ each_cons

```ruby
# เมธอด Enumerable บน Range
r = (1..10)

# each_slice - แบ่งเป็น chunks
r.each_slice(3) { |chunk| print chunk.inspect + " " }
puts
# => [1, 2, 3] [4, 5, 6] [7, 8, 9] [10]

# each_cons - sliding window
r.each_cons(3) { |window| print window.inspect + " " }
puts
# => [1, 2, 3] [2, 3, 4] [3, 4, 5] [4, 5, 6] [5, 6, 7] [6, 7, 8] [7, 8, 9] [8, 9, 10]

# ตัวอย่างจริง: ตรวจสอบ trend
prices = [100, 105, 110, 108, 112, 115, 113, 120]
trends = []

prices.each_cons(2) do |prev, curr|
  if curr > prev
    trends << :up
  elsif curr < prev
    trends << :down
  else
    trends << :flat
  end
end

puts trends.inspect
# => [:up, :up, :down, :up, :up, :down, :up]

# ตัวอย่าง: moving average
def moving_average(data, window_size)
  data.each_cons(window_size).map do |window|
    window.sum.to_f / window_size
  end
end

data = [10, 20, 30, 40, 50, 60, 70]
puts moving_average(data, 3).inspect
# => [20.0, 30.0, 40.0, 50.0, 60.0]
```

---

## ขั้นตอนที่ 362: Range เป็น Lazy Enumerator

```ruby
# Range กับ lazy - สำหรับ infinite หรือ large ranges
# ถ้าไม่ใช้ lazy จะพยายาม enumerate ทั้งหมด

# ตัวอย่าง: หา 5 จำนวนเฉพาะแรกที่มากกว่า 100
def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n).to_i).none? { |i| n % i == 0 }
end

# แบบ lazy - efficient
primes = (101..).lazy.select { |n| prime?(n) }.first(5)
puts primes.inspect   # => [101, 103, 107, 109, 113]

# แบบไม่ lazy จะ raise error สำหรับ infinite range
# (101..).select { |n| prime?(n) }.first(5)  # infinite loop!

# ตัวอย่าง: Fibonacci sequence
fib = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

# หา Fibonacci ที่น้อยกว่า 100
puts fib.lazy.select { |n| n < 100 }.to_a.inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]

# Range กับ lazy map
result = (1..Float::INFINITY).lazy
  .map { |n| n * n }
  .select { |n| n % 3 == 0 }
  .first(5)
puts result.inspect   # => [9, 36, 81, 144, 225]
```

---

## ขั้นตอนที่ 363: ตัวอย่างการใช้งานจริง

```ruby
# 1. Date range iteration
require 'date'

start_date = Date.new(2024, 1, 1)
end_date = Date.new(2024, 1, 7)

(start_date..end_date).each do |date|
  puts date.strftime("%A, %d %B %Y")
end

# 2. Business hours checker
def business_hours?(hour)
  (9..17).include?(hour)
end

(6..22).each do |hour|
  status = business_hours?(hour) ? "เปิด" : "ปิด"
  puts "#{hour}:00 - #{status}"
end

# 3. Pricing tiers
PRICING_TIERS = [
  { range: (..99),     name: "Basic",      price: 0 },
  { range: (100..499), name: "Standard",   price: 99 },
  { range: (500..999), name: "Pro",        price: 299 },
  { range: (1000..),   name: "Enterprise", price: 999 }
]

def get_plan(usage)
  tier = PRICING_TIERS.find { |t| t[:range].include?(usage) }
  tier || { name: "Unknown", price: 0 }
end

[50, 200, 750, 1500].each do |usage|
  plan = get_plan(usage)
  puts "Usage #{usage}: #{plan[:name]} (฿#{plan[:price]}/เดือน)"
end

# 4. Color gradient
def rgb_gradient(from_r, to_r, steps)
  (0..steps).map do |step|
    t = step.to_f / steps
    r = (from_r + (to_r - from_r) * t).round
    r
  end
end

reds = rgb_gradient(0, 255, 10)
puts reds.inspect
```

---

## ขั้นตอนที่ 364: Symbol และ Range ร่วมกัน

```ruby
# ใช้ Symbol สำหรับกำหนด ranges
GRADE_RANGES = {
  excellent: (90..100),
  good:      (80...90),
  average:   (70...80),
  passing:   (60...70),
  failing:   (0...60)
}

def letter_grade(score)
  GRADE_RANGES.find { |grade, range| range.include?(score) }&.first
end

[95, 82, 73, 64, 45].each do |score|
  puts "#{score}: #{letter_grade(score)}"
end

# Symbol เป็น keys ของ Range configuration
TIME_RANGES = {
  morning:   (6..11),
  afternoon: (12..17),
  evening:   (18..21),
  night:     (22..23)
}

def time_of_day(hour)
  TIME_RANGES.find { |_, range| range.include?(hour) }&.first || :midnight
end

[7, 14, 19, 23, 3].each do |hour|
  puts "#{hour}:00 - #{time_of_day(hour)}"
end
```

---

## ขั้นตอนที่ 365: Tips และ Tricks

```ruby
# 1. Range ในการสร้าง random number
random = rand(1..6)
puts "ทอยลูกเต๋า: #{random}"

# 2. Sample จาก range
sample = (1..100).to_a.sample(5)
puts sample.inspect

# 3. Range เป็น infinite generator
natural_numbers = (1..).lazy
evens = natural_numbers.select(&:even?)
puts evens.first(10).inspect
# => [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# 4. ใช้ Range กับ sprintf
(1..5).each { |n| puts "Item %02d" % n }

# 5. Flip-flop operator (deprecated แต่น่ารู้)
# เก็บ state ระหว่าง start และ end condition
text = ["apple", "---start---", "banana", "cherry", "---end---", "date"]
in_section = false
text.each do |line|
  in_section = true if line == "---start---"
  puts line if in_section && !line.start_with?("---")
  in_section = false if line == "---end---"
end

# 6. Symbol ใน frozen string optimization
# Ruby 3+ frozen string literal
require 'set'

common_symbols = Set.new([:name, :age, :email, :phone, :address])
puts common_symbols.include?(:name)  # => true

# 7. Symbol comparison สำหรับ sort
[:banana, :apple, :cherry, :date].sort.each { |s| puts s }
```

---

## แบบฝึกหัด 20 ข้อ

### ข้อ 1-5: Symbols

**ข้อ 1:** สร้าง Hash ที่ใช้ Symbol keys เก็บข้อมูลนักศึกษา
```ruby
# เฉลย
student = {
  name: "สมชาย ใจดี",
  id: "63001234",
  gpa: 3.5,
  major: :computer_science,
  year: 3
}

puts student[:name]
puts student[:major]
```

**ข้อ 2:** ใช้ Symbol#to_proc กับ Array
```ruby
# เฉลย
words = ["hello", "world", "ruby", "programming"]

# แปลงเป็นตัวใหญ่ทั้งหมด
puts words.map(&:upcase).inspect

# เรียงลำดับตาม length
puts words.sort_by(&:length).inspect

# filter เฉพาะที่ length > 4
puts words.select { |w| w.length > 4 }.inspect
```

**ข้อ 3:** สร้าง method ที่รับ Symbol และ call method ที่ชื่อนั้นบน object
```ruby
# เฉลย
def apply_transformation(str, *methods)
  methods.reduce(str) { |result, method| result.send(method) }
end

puts apply_transformation("hello world", :capitalize, :reverse)
# => "dlrow olleH"
```

**ข้อ 4:** เปรียบเทียบ performance ของ Symbol กับ String ใน Hash lookup
```ruby
# เฉลย
require 'benchmark'

n = 1_000_000
hash_sym = { name: "Ruby", age: 30 }
hash_str = { "name" => "Ruby", "age" => 30 }

Benchmark.bm(15) do |x|
  x.report("Symbol lookup:") { n.times { hash_sym[:name] } }
  x.report("String lookup:") { n.times { hash_str["name"] } }
end
```

**ข้อ 5:** สร้าง enum-like module ด้วย Symbols
```ruby
# เฉลย
module Color
  RED    = :red
  GREEN  = :green
  BLUE   = :blue
  
  ALL = [RED, GREEN, BLUE].freeze
  
  def self.valid?(color)
    ALL.include?(color)
  end
end

puts Color.valid?(:red)     # => true
puts Color.valid?(:purple)  # => false
```

### ข้อ 6-10: Ranges

**ข้อ 6:** ใช้ Range ใน case/when สำหรับ BMI calculator
```ruby
# เฉลย
def bmi_status(height_cm, weight_kg)
  bmi = weight_kg / (height_cm / 100.0) ** 2
  
  status = case bmi.round(1)
  when (..18.4) then "น้ำหนักต่ำกว่าเกณฑ์"
  when 18.5..24.9 then "น้ำหนักปกติ"
  when 25.0..29.9 then "น้ำหนักเกิน"
  when 30.0.. then "อ้วน"
  end
  
  "BMI: #{bmi.round(1)} - #{status}"
end

puts bmi_status(170, 60)
puts bmi_status(170, 85)
```

**ข้อ 7:** สร้าง method ที่คืน range ของ working days
```ruby
# เฉลย
require 'date'

def working_days_count(start_date, end_date)
  (start_date..end_date).count do |date|
    date.monday? || date.tuesday? || date.wednesday? ||
    date.thursday? || date.friday?
  end
end

start_d = Date.new(2024, 1, 1)
end_d   = Date.new(2024, 1, 31)
puts "วันทำงานใน Jan 2024: #{working_days_count(start_d, end_d)} วัน"
```

**ข้อ 8:** ใช้ step range สร้าง multiplication table
```ruby
# เฉลย
def multiplication_table(n)
  puts "  " + (1..n).map { |i| i.to_s.rjust(4) }.join
  (1..n).each do |i|
    row = (1..n).map { |j| (i * j).to_s.rjust(4) }.join
    puts "#{i.to_s.rjust(2)}#{row}"
  end
end

multiplication_table(5)
```

**ข้อ 9:** ใช้ endless range เพื่อ validate input
```ruby
# เฉลย
def validate_age(age)
  case age
  when (..0)   then raise ArgumentError, "อายุต้องมากกว่า 0"
  when 1..120  then age
  when (121..) then raise ArgumentError, "อายุสูงสุด 120 ปี"
  end
end

begin
  puts validate_age(25)
  puts validate_age(-1)
rescue ArgumentError => e
  puts "Error: #{e.message}"
end
```

**ข้อ 10:** สร้าง paginator ด้วย Range
```ruby
# เฉลย
class Paginator
  def initialize(total, per_page: 10)
    @total = total
    @per_page = per_page
  end

  def page_range(page)
    start = (page - 1) * @per_page
    finish = [start + @per_page - 1, @total - 1].min
    (start..finish)
  end

  def total_pages
    (@total.to_f / @per_page).ceil
  end
end

pager = Paginator.new(53, per_page: 10)
puts "Total pages: #{pager.total_pages}"
(1..pager.total_pages).each do |page|
  range = pager.page_range(page)
  puts "Page #{page}: items #{range.begin + 1} to #{range.end + 1}"
end
```

### ข้อ 11-15: Advanced

**ข้อ 11:** สร้าง countdown timer ด้วย range
```ruby
# เฉลย
def countdown(from)
  (0..from).to_a.reverse.each do |n|
    print "#{n}... "
    sleep(0.1)  # ลด delay สำหรับ demo
  end
  puts "เริ่ม!"
end

countdown(5)
```

**ข้อ 12:** ใช้ lazy range หาจำนวนเฉพาะ
```ruby
# เฉลย
def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n).to_i).none? { |i| n % i == 0 }
end

# หา prime 10 ตัวแรกที่มากกว่า 1000
primes = (1001..).lazy.select { |n| prime?(n) }.first(10)
puts primes.inspect
```

**ข้อ 13:** สร้าง range ของ dates และ format
```ruby
# เฉลย
require 'date'

def date_range_thai(start_date, days)
  months_th = %w[ม.ค. ก.พ. มี.ค. เม.ย. พ.ค. มิ.ย. ก.ค. ส.ค. ก.ย. ต.ค. พ.ย. ธ.ค.]
  
  (start_date...(start_date + days)).map do |date|
    "#{date.day} #{months_th[date.month - 1]} #{date.year + 543}"
  end
end

start = Date.new(2024, 12, 29)
puts date_range_thai(start, 5).inspect
```

**ข้อ 14:** เขียน method ที่รับ Symbol และคืนผลลัพธ์
```ruby
# เฉลย
class NumberAnalyzer
  def initialize(numbers)
    @numbers = numbers
  end

  def analyze(*operations)
    operations.each_with_object({}) do |op, results|
      results[op] = @numbers.send(op)
    end
  end
end

analyzer = NumberAnalyzer.new([3, 1, 4, 1, 5, 9, 2, 6, 5, 3])
puts analyzer.analyze(:sum, :min, :max, :count).inspect
```

**ข้อ 15:** สร้าง range validator
```ruby
# เฉลย
class RangeValidator
  def initialize(ranges)
    @ranges = ranges
  end

  def valid?(value)
    @ranges.any? { |range| range.include?(value) }
  end

  def category(value)
    @ranges.find { |range| range.include?(value) }&.then do |r|
      "Range #{r}"
    end || "ไม่อยู่ใน range ใดเลย"
  end
end

validator = RangeValidator.new([1..10, 20..30, 50..100])
puts validator.valid?(5)    # => true
puts validator.valid?(15)   # => false
puts validator.category(25) # => "Range 20..30"
```

### ข้อ 16-20: Applications

**ข้อ 16:** สร้าง grade book ด้วย Range
```ruby
# เฉลย
class GradeBook
  GRADES = {
    A: (90..100),
    B: (80...90),
    C: (70...80),
    D: (60...70),
    F: (0...60)
  }

  def initialize
    @scores = {}
  end

  def add_score(student, score)
    @scores[student] = score
  end

  def grade(student)
    score = @scores[student]
    GRADES.find { |_, range| range.include?(score) }&.first
  end

  def report
    @scores.map do |name, score|
      "#{name}: #{score} (#{grade(name)})"
    end
  end
end

book = GradeBook.new
book.add_score("สมชาย", 92)
book.add_score("สมหญิง", 78)
book.add_score("สมศรี", 65)
puts book.report.join("\n")
```

**ข้อ 17:** ใช้ Symbol และ Range ใน DSL
```ruby
# เฉลย
class Rule
  attr_reader :field, :validations

  def initialize(field)
    @field = field
    @validations = {}
  end

  def in_range(range)
    @validations[:range] = range
    self
  end

  def required
    @validations[:required] = true
    self
  end

  def validate(value)
    errors = []
    errors << "#{@field} จำเป็นต้องมีค่า" if @validations[:required] && value.nil?
    if @validations[:range] && value
      errors << "#{@field} ต้องอยู่ใน #{@validations[:range]}" unless @validations[:range].include?(value)
    end
    errors
  end
end

age_rule = Rule.new(:age).required.in_range(0..120)
puts age_rule.validate(nil)
puts age_rule.validate(25)
puts age_rule.validate(150)
```

**ข้อ 18-20:** เพิ่มเติม (ฝึกเอง)

```ruby
# ข้อ 18: สร้าง TimeSlot class ด้วย Range
class TimeSlot
  def initialize(start_hour, end_hour)
    @range = (start_hour..end_hour)
  end

  def available?(hour)
    @range.include?(hour)
  end

  def overlaps?(other)
    @range.include?(other.begin) || other.include?(@range.begin)
  end

  def to_s
    "#{@range.begin}:00 - #{@range.end}:00"
  end

  protected
  def begin = @range.begin
  def include?(hour) = @range.include?(hour)
end

morning = TimeSlot.new(9, 12)
afternoon = TimeSlot.new(13, 17)

puts morning.available?(10)   # => true
puts morning.available?(14)   # => false
puts "Morning: #{morning}"
puts "Afternoon: #{afternoon}"

# ข้อ 19: Symbol dispatch table
COMMANDS = {
  greet:   -> (name) { "สวัสดี #{name}!" },
  farewell: -> (name) { "ลาก่อน #{name}!" },
  ask:      -> (name) { "#{name} เป็นอย่างไรบ้าง?" }
}

def dispatch(command_sym, name)
  COMMANDS[command_sym]&.call(name) || "ไม่รู้จักคำสั่ง #{command_sym}"
end

puts dispatch(:greet, "สมชาย")
puts dispatch(:farewell, "สมหญิง")
puts dispatch(:unknown, "ใครก็ตาม")

# ข้อ 20: Range-based configuration
class Config
  VALID_PORT_RANGE    = (1..65535)
  VALID_TIMEOUT_RANGE = (1..300)
  
  attr_reader :port, :timeout

  def initialize(port: 3000, timeout: 30)
    self.port = port
    self.timeout = timeout
  end

  def port=(value)
    unless VALID_PORT_RANGE.include?(value)
      raise ArgumentError, "Port ต้องอยู่ใน #{VALID_PORT_RANGE}"
    end
    @port = value
  end

  def timeout=(value)
    unless VALID_TIMEOUT_RANGE.include?(value)
      raise ArgumentError, "Timeout ต้องอยู่ใน #{VALID_TIMEOUT_RANGE}"
    end
    @timeout = value
  end
end

config = Config.new(port: 8080, timeout: 60)
puts "Port: #{config.port}, Timeout: #{config.timeout}"

begin
  Config.new(port: 99999)
rescue ArgumentError => e
  puts "Error: #{e.message}"
end
```

---

## สรุปบทที่ 17

ในบทนี้เราได้เรียนรู้:

**Symbols:**
- Symbol คือ immutable identifier ที่ share instance ในหน่วยความจำ
- เร็วกว่าและเบากว่า String สำหรับ comparison และ Hash keys
- `to_sym` / `to_s` สำหรับแปลงระหว่าง Symbol และ String
- `Symbol#to_proc` กับ `&:method_name` เป็น shorthand ที่ทรงพลัง
- ใช้เป็น enum-like constants, options, และ method names

**Ranges:**
- Inclusive `(..)` และ Exclusive `(...)` ranges
- Endless ranges `(1..)` ใน Ruby 2.6+
- Beginless ranges `(..5)` ใน Ruby 2.7+
- Range methods: `include?`, `cover?`, `each`, `to_a`, `step`
- ใช้ใน `case/when`, Array slicing, และ validation
- Lazy ranges สำหรับ infinite sequences
