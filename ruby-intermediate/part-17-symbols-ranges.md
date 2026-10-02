# ตอนที่ 17: Symbols และ Ranges (Steps 346-365)

## บทนำ

ในตอนนี้เราจะเรียนรู้เกี่ยวกับ **Symbol** และ **Range** ซึ่งเป็น built-in types ที่สำคัญมากใน Ruby Symbol ช่วยให้โปรแกรมทำงานได้เร็วขึ้นและประหยัดหน่วยความจำ ส่วน Range ช่วยให้เราทำงานกับช่วงค่าต่างๆ ได้อย่างสะดวก

---

## Step 346: Symbol คืออะไร?

**Symbol** คือ object ที่แทนชื่อหรือ identifier ใน Ruby มีลักษณะพิเศษคือ Symbol ที่มีชื่อเหมือนกันจะเป็น object เดียวกันในหน่วยความจำเสมอ

```ruby
# สร้าง Symbol
:hello
:world
:user_name
:age

# Symbol มีค่า object_id เหมือนกันถ้าชื่อเหมือนกัน
puts :hello.object_id  # เช่น 2468708
puts :hello.object_id  # เลขเดิม!
puts :hello.object_id  # เลขเดิมเสมอ

# String ต่างกัน - object ใหม่ทุกครั้ง
puts "hello".object_id  # เช่น 70123456789
puts "hello".object_id  # ต่างกัน!
puts "hello".object_id  # ต่างกันอีก
```

### Symbol vs String

| คุณสมบัติ | Symbol | String |
|-----------|--------|--------|
| immutable | ✅ เปลี่ยนไม่ได้ | ❌ เปลี่ยนได้ |
| unique | ✅ มีแค่ชิ้นเดียว | ❌ สร้างใหม่ทุกครั้ง |
| หน่วยความจำ | ✅ ประหยัด | ❌ ใช้มาก |
| ความเร็วเปรียบเทียบ | ✅ เร็วกว่า | ❌ ช้ากว่า |
| method มากมาย | ❌ น้อยกว่า | ✅ มากกว่า |

```ruby
# ทดสอบความแตกต่าง
symbol1 = :ruby
symbol2 = :ruby
string1 = "ruby"
string2 = "ruby"

puts symbol1 == symbol2      # true
puts symbol1.equal?(symbol2) # true  - object เดียวกัน!
puts string1 == string2      # true
puts string1.equal?(string2) # false - คนละ object!

# ดู object_id
puts symbol1.object_id == symbol2.object_id  # true
puts string1.object_id == string2.object_id  # false
```

---

## Step 347: วิธีสร้าง Symbol

มีหลายวิธีในการสร้าง Symbol ใน Ruby:

```ruby
# วิธีที่ 1: ใช้ colon นำหน้า (ปกติที่สุด)
:name
:user_id
:first_name
:total_price

# วิธีที่ 2: ใช้ quote เมื่อมีช่องว่างหรืออักขระพิเศษ
:"hello world"
:"user-name"
:"my.method"
:"123start"

# วิธีที่ 3: แปลงจาก String ด้วย to_sym
"name".to_sym       # => :name
"hello world".to_sym # => :"hello world"
"user_id".to_sym    # => :user_id

# วิธีที่ 4: ใช้ intern (alias ของ to_sym)
"name".intern       # => :name

# ตัวอย่างการใช้งาน
name_sym = :name
greeting_sym = :"สวัสดี"

puts name_sym      # name
puts greeting_sym  # สวัสดี
puts name_sym.class # Symbol
```

### Symbol Array Literal

```ruby
# สร้าง array ของ Symbol ได้ง่ายๆ ด้วย %i
colors = %i[red green blue yellow]
puts colors.inspect
# => [:red, :green, :blue, :yellow]

# เทียบกับการเขียนแบบปกติ
colors_long = [:red, :green, :blue, :yellow]

# ใช้ %I สำหรับ interpolation
prefix = "dark"
shades = %I[#{prefix}_red #{prefix}_green #{prefix}_blue]
puts shades.inspect
# => [:dark_red, :dark_green, :dark_blue]
```

---

## Step 348: Symbol Methods

Symbol มี method หลายอย่างที่เป็นประโยชน์:

```ruby
symbol = :hello_world

# to_s - แปลงเป็น String
puts symbol.to_s        # "hello_world"
puts symbol.to_s.class  # String

# id2name - เหมือน to_s
puts symbol.id2name     # "hello_world"

# to_proc - แปลงเป็น Proc (ใช้กับ map, select ฯลฯ)
double = :to_s.to_proc
puts double.call(42)    # "42"

words = ["hello", "world", "ruby"]
upcase_words = words.map(&:upcase)
puts upcase_words.inspect  # ["HELLO", "WORLD", "RUBY"]

# length/size - ความยาวของชื่อ
puts :hello.length  # 5
puts :hello.size    # 5

# empty? - ว่างหรือไม่
puts :"".empty?     # true
puts :hello.empty?  # false

# upcase, downcase, capitalize
puts :hello.upcase    # HELLO
puts :HELLO.downcase  # hello
puts :hello.capitalize # Hello

# match? - ตรงกับ regex หรือไม่
puts :hello_world.match?(/world/)  # true
puts :hello_world.match?(/xyz/)    # false

# inspect - แสดงแบบ Symbol literal
puts :hello.inspect  # :hello
```

### Symbol เปรียบเทียบ

```ruby
# เปรียบเทียบ Symbol ด้วย == และ <=>
puts :apple == :apple  # true
puts :apple == :banana # false
puts :apple != :banana # true

# เรียงลำดับ
symbols = [:cherry, :apple, :banana, :date]
puts symbols.sort.inspect
# => [:apple, :banana, :cherry, :date]

# <=> คืนค่า -1, 0, 1
puts (:apple <=> :banana)  # -1
puts (:banana <=> :banana) # 0
puts (:cherry <=> :apple)  # 1
```

---

## Step 349: to_proc และ การใช้ Symbol กับ Blocks

หนึ่งในคุณสมบัติที่ทรงพลังที่สุดของ Symbol คือการแปลงเป็น Proc:

```ruby
# to_proc แบบ manual
upcase_proc = :upcase.to_proc
puts upcase_proc.call("hello")  # "HELLO"

# ใช้ & เพื่อแปลง Symbol เป็น block
numbers = [1, 2, 3, 4, 5]
strings = numbers.map(&:to_s)
puts strings.inspect  # ["1", "2", "3", "4", "5"]

words = ["hello", "world", "ruby"]
upcased = words.map(&:upcase)
puts upcased.inspect  # ["HELLO", "WORLD", "RUBY"]

lengths = words.map(&:length)
puts lengths.inspect  # [5, 5, 4]

# select ด้วย Symbol
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
evens = numbers.select(&:even?)
odds = numbers.select(&:odd?)
puts evens.inspect  # [2, 4, 6, 8, 10]
puts odds.inspect   # [1, 3, 5, 7, 9]

# reduce ด้วย Symbol
sum = [1, 2, 3, 4, 5].reduce(:+)
puts sum  # 15

product = [1, 2, 3, 4, 5].reduce(:*)
puts product  # 120
```

### ตัวอย่างจริง: Data Processing

```ruby
# ข้อมูลผู้ใช้
users = [
  { name: "Alice", age: 30, active: true },
  { name: "Bob", age: 25, active: false },
  { name: "Charlie", age: 35, active: true },
  { name: "Diana", age: 28, active: true }
]

# ดึงชื่อทั้งหมด
names = users.map { |u| u[:name] }
puts names.inspect

# กรองเฉพาะ active users
active_users = users.select { |u| u[:active] }
puts active_users.length  # 3

# เรียงตามอายุ
sorted_users = users.sort_by { |u| u[:age] }
sorted_users.each { |u| puts "#{u[:name]}: #{u[:age]}" }
```

---

## Step 350: Symbol ใช้ทำอะไร? (Hash Keys)

การใช้ Symbol เป็น Hash key เป็นวิธีที่นิยมมากที่สุดใน Ruby:

```ruby
# Hash ด้วย Symbol keys (แนะนำ)
person = {
  name: "Alice",
  age: 30,
  email: "alice@example.com"
}

# เข้าถึงด้วย Symbol
puts person[:name]   # Alice
puts person[:age]    # 30
puts person[:email]  # alice@example.com

# Hash ด้วย String keys (น้อยนิยม)
person_str = {
  "name" => "Bob",
  "age"  => 25
}

# ทำไม Symbol keys ดีกว่า?
# 1. อ่านง่ายกว่า
# 2. เร็วกว่าในการค้นหา
# 3. ใช้หน่วยความจำน้อยกว่า

# ทดสอบความเร็ว
require 'benchmark'
n = 1_000_000

Benchmark.bm do |x|
  x.report("Symbol key:") do
    n.times { { name: "test" }[:name] }
  end
  x.report("String key:") do
    n.times { { "name" => "test" }["name"] }
  end
end
# Symbol key จะเร็วกว่าประมาณ 20-30%
```

### Symbol เป็น Method Names

```ruby
# เรียก method ด้วย send
class Calculator
  def add(a, b)
    a + b
  end

  def subtract(a, b)
    a - b
  end

  def multiply(a, b)
    a * b
  end
end

calc = Calculator.new

# ใช้ send กับ Symbol
operation = :add
result = calc.send(operation, 10, 5)
puts result  # 15

# เปลี่ยน operation
operation = :multiply
result = calc.send(operation, 10, 5)
puts result  # 50

# ใช้ method สำหรับ method object
add_method = calc.method(:add)
puts add_method.call(3, 4)  # 7

# respond_to? ด้วย Symbol
puts calc.respond_to?(:add)     # true
puts calc.respond_to?(:divide)  # false
```

---

## Step 351: Symbol ใน Callbacks และ Options

```ruby
# Symbol ในการกำหนด callback (พบบ่อยใน Rails)
class Order
  attr_accessor :status

  CALLBACKS = {
    before_create: [],
    after_create:  [],
    before_save:   []
  }

  def self.before_create(method_name)
    CALLBACKS[:before_create] << method_name
  end

  def self.after_create(method_name)
    CALLBACKS[:after_create] << method_name
  end

  before_create :validate_items
  before_create :check_stock
  after_create  :send_confirmation

  def create
    CALLBACKS[:before_create].each { |cb| send(cb) }
    puts "Creating order..."
    CALLBACKS[:after_create].each { |cb| send(cb) }
  end

  private

  def validate_items
    puts "Validating items..."
  end

  def check_stock
    puts "Checking stock..."
  end

  def send_confirmation
    puts "Sending confirmation email..."
  end
end

order = Order.new
order.create
# Validating items...
# Checking stock...
# Creating order...
# Sending confirmation email...
```

### Symbol เป็น Options

```ruby
# Method ที่รับ Symbol เป็น option
def process_data(data, format: :json, sort: :asc, limit: nil)
  puts "Processing in #{format} format"
  puts "Sorting: #{sort}"
  puts "Limit: #{limit || 'none'}"

  case format
  when :json
    # process as JSON
    data.to_s
  when :csv
    # process as CSV
    data.join(",")
  when :xml
    # process as XML
    "<data>#{data}</data>"
  end
end

process_data([1, 2, 3])
process_data([1, 2, 3], format: :csv)
process_data([1, 2, 3], format: :xml, sort: :desc, limit: 10)
```

---

## Step 352: Memory Efficiency ของ Symbol

```ruby
# ทดสอบการใช้หน่วยความจำ
require 'objspace'

# สร้าง String หลายตัว
strings = 1000.times.map { "hello_world" }
string_memory = strings.sum { |s| ObjectSpace.memsize_of(s) }
puts "String memory: #{string_memory} bytes"

# Symbol ใช้หน่วยความจำเพียงครั้งเดียว
symbols = 1000.times.map { :hello_world }
symbol_memory = symbols.sum { |s| ObjectSpace.memsize_of(s) }
puts "Symbol memory: #{symbol_memory} bytes"
# Symbol จะใช้หน่วยความจำน้อยกว่ามาก

# แสดงว่า Symbol เป็น object เดียวกัน
puts symbols.map(&:object_id).uniq.length  # 1 (object เดียว!)
puts strings.map(&:object_id).uniq.length  # 1000 (object ต่างกัน!)
```

### ข้อควรระวัง: Dynamic Symbols

```ruby
# อย่าสร้าง Symbol แบบ dynamic จาก user input!
# เพราะ Symbol ไม่ถูก GC เก็บ (ใน Ruby เก่ากว่า 2.2)

# ❌ อันตราย! (Ruby < 2.2)
# user_input.to_sym  # ถ้ามี user input จำนวนมาก จะทำให้ memory leak

# ✅ ใน Ruby 2.2+ Symbol ถูก GC เก็บได้แล้ว
# แต่ยังควรระวังสำหรับ performance

# ใช้ freeze เพื่อระบุว่าเป็น immutable
ALLOWED_STATUSES = %i[pending active inactive].freeze

def update_status(status)
  unless ALLOWED_STATUSES.include?(status)
    raise ArgumentError, "Invalid status: #{status}"
  end
  puts "Updating to #{status}"
end

update_status(:active)    # OK
update_status(:pending)   # OK
update_status(:unknown)   # ArgumentError!
```

---

## Step 353: Range คืออะไร?

**Range** คือ object ที่แทนช่วงค่าระหว่างค่าเริ่มต้นและค่าสิ้นสุด มีสองประเภทหลัก:

```ruby
# Inclusive Range (..) - รวมค่าสุดท้าย
inclusive = 1..10
puts inclusive.include?(10)  # true
puts inclusive.to_a.inspect  # [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Exclusive Range (...) - ไม่รวมค่าสุดท้าย
exclusive = 1...10
puts exclusive.include?(10)  # false
puts exclusive.to_a.inspect  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Range ของตัวอักษร
letters = 'a'..'z'
puts letters.to_a.inspect
# => ["a", "b", "c", ..., "z"]

# Range ของ String
str_range = "aa".."az"
puts str_range.to_a.length  # 26

# Range แสดงแบบ inspect
puts (1..10).inspect    # 1..10
puts (1...10).inspect   # 1...10
```

---

## Step 354: สร้าง Range ประเภทต่างๆ

```ruby
# Integer Range
int_range = 1..100
puts int_range.first  # 1
puts int_range.last   # 100
puts int_range.size   # 100

# Float Range (ไม่สามารถ iterate ได้โดยตรง)
float_range = 1.0..5.0
puts float_range.include?(3.5)  # true
puts float_range.include?(5.0)  # true
puts float_range.include?(5.1)  # false

# String Range
str_range = 'a'..'e'
puts str_range.to_a.inspect  # ["a", "b", "c", "d", "e"]

# Date Range
require 'date'
today = Date.today
next_week = today + 7
date_range = today..next_week
puts date_range.count  # 8 (รวมทั้งสองวัน)

date_range.each do |date|
  puts date.strftime("%Y-%m-%d")
end

# Time Range
now = Time.now
later = now + 3600  # 1 ชั่วโมง
time_range = now..later
puts time_range.include?(now + 1800)  # true (30 นาที = ครึ่งทาง)
```

---

## Step 355: Range Methods - Basics

```ruby
range = 1..10

# first, last
puts range.first    # 1
puts range.last     # 10
puts range.first(3).inspect  # [1, 2, 3]
puts range.last(3).inspect   # [8, 9, 10]

# min, max
puts range.min  # 1
puts range.max  # 10

# minmax
puts range.minmax.inspect  # [1, 10]

# size, count, length
puts range.size    # 10
puts range.count   # 10
puts range.length  # 10

# sum
puts range.sum  # 55 (1+2+3+...+10)

# to_a - แปลงเป็น Array
arr = range.to_a
puts arr.inspect  # [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# each - วนซ้ำ
range.each { |n| print "#{n} " }
puts  # newline
# 1 2 3 4 5 6 7 8 9 10
```

---

## Step 356: Range Methods - include? และ cover?

```ruby
# include? - ตรวจสอบว่าค่าอยู่ใน Range หรือไม่
range = 1..10
puts range.include?(5)   # true
puts range.include?(10)  # true
puts range.include?(11)  # false
puts range.include?(0)   # false

# กับ exclusive range
ex_range = 1...10
puts ex_range.include?(9)   # true
puts ex_range.include?(10)  # false

# cover? - คล้าย include? แต่ไม่ iterate (เร็วกว่า)
float_range = 1.0..10.0
puts float_range.cover?(5.5)   # true
puts float_range.cover?(10.0)  # true
puts float_range.cover?(10.1)  # false

# ความแตกต่างระหว่าง include? กับ cover?
# include? ใช้ iteration ดังนั้นช้ากว่าสำหรับ range ขนาดใหญ่
# cover? ใช้การเปรียบเทียบโดยตรง จึงเร็วกว่า

# String ranges
str_range = 'a'..'z'
puts str_range.include?('m')   # true
puts str_range.include?('1')   # false

# include? กับ String หลายตัว - cover? ต่างจาก include?
str_range2 = 'a'..'z'
puts str_range2.cover?('m')    # true
puts str_range2.cover?('aa')   # false (include? เป็น false ด้วย)
```

---

## Step 357: Range Methods - step, each_slice

```ruby
# step - วนซ้ำทีละ N
(1..20).step(2) { |n| print "#{n} " }
puts
# 1 3 5 7 9 11 13 15 17 19

(0.0..1.0).step(0.1) { |n| print "#{n.round(1)} " }
puts
# 0.0 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9 1.0

# วันในสัปดาห์
require 'date'
start_date = Date.new(2024, 1, 1)
end_date = Date.new(2024, 1, 31)
(start_date..end_date).step(7) { |date| puts date.strftime("%A, %b %d") }

# each_slice
(1..12).each_slice(3) { |group| puts group.inspect }
# [1, 2, 3]
# [4, 5, 6]
# [7, 8, 9]
# [10, 11, 12]

# each_cons
(1..6).each_cons(3) { |window| puts window.inspect }
# [1, 2, 3]
# [2, 3, 4]
# [3, 4, 5]
# [4, 5, 6]
```

---

## Step 358: Endless Ranges (Ruby 2.6+)

Ruby 2.6 แนะนำ **Endless Range** ซึ่งไม่มีค่าสิ้นสุด:

```ruby
# Endless Range
range = (1..)
puts range.class  # Range

# ตรวจสอบ
puts range.include?(100)   # true
puts range.include?(1000)  # true
puts range.include?(0)     # false

# ใช้กับ case/when
def classify_age(age)
  case age
  when (..12)   then "เด็ก"
  when (13..17) then "วัยรุ่น"
  when (18..64) then "ผู้ใหญ่"
  when (65..)   then "ผู้สูงอายุ"
  end
end

puts classify_age(5)   # เด็ก
puts classify_age(15)  # วัยรุ่น
puts classify_age(30)  # ผู้ใหญ่
puts classify_age(70)  # ผู้สูงอายุ

# ใช้กับ Array slicing
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
puts arr[3..].inspect   # [4, 5, 6, 7, 8, 9, 10]
puts arr[..4].inspect   # [1, 2, 3, 4, 5]
puts arr[2..7].inspect  # [3, 4, 5, 6, 7, 8]

# select กับ Endless Range
numbers = (1..20).to_a
big_numbers = numbers.select { |n| (10..) === n }
puts big_numbers.inspect  # [10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
```

---

## Step 359: Beginless Ranges (Ruby 2.7+)

Ruby 2.7 แนะนำ **Beginless Range** ซึ่งไม่มีค่าเริ่มต้น:

```ruby
# Beginless Range
range = (..10)
puts range.include?(5)   # true
puts range.include?(10)  # true
puts range.include?(11)  # false
puts range.include?(-999) # true!

# ใช้กับ Array slicing
arr = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
puts arr[..4].inspect   # [0, 1, 2, 3, 4]
puts arr[...4].inspect  # [0, 1, 2, 3]

# ใช้กับ case/when
def check_score(score)
  case score
  when (..49)  then "F - ล้มเหลว"
  when (50..59) then "D - พอใช้"
  when (60..69) then "C - ปานกลาง"
  when (70..79) then "B - ดี"
  when (80..89) then "A- - ดีมาก"
  when (90..)   then "A - ยอดเยี่ยม"
  end
end

puts check_score(45)   # F - ล้มเหลว
puts check_score(75)   # B - ดี
puts check_score(95)   # A - ยอดเยี่ยม

# grep กับ Beginless/Endless Range
data = [-5, 0, 3, 7, 12, 20, 35]
positives = data.grep((1..))
puts positives.inspect  # [3, 7, 12, 20, 35]

negatives = data.grep((..0))
puts negatives.inspect  # [-5, 0]
```

---

## Step 360: Range ใน case/when

Range ทำงานได้ดีมากใน `case/when`:

```ruby
# ตัวอย่างคะแนนเกรด
def letter_grade(score)
  case score
  when 90..100 then "A"
  when 80...90 then "B"
  when 70...80 then "C"
  when 60...70 then "D"
  when 0...60  then "F"
  else "Invalid score"
  end
end

(0..100).step(10) do |score|
  puts "#{score}: #{letter_grade(score)}"
end

# ตัวอย่าง: ราคาส่งสินค้า
def shipping_cost(weight_kg)
  case weight_kg
  when (0..0.5)  then 30
  when (0.5..1)  then 50
  when (1..5)    then 80
  when (5..10)   then 150
  when (10..)    then 300
  end
end

puts "ราคาส่ง 0.3 กก.: #{shipping_cost(0.3)} บาท"
puts "ราคาส่ง 2 กก.: #{shipping_cost(2)} บาท"
puts "ราคาส่ง 15 กก.: #{shipping_cost(15)} บาท"

# ตัวอย่าง: เวลา
def part_of_day(hour)
  case hour
  when 0..5   then "กลางคืน"
  when 6..11  then "เช้า"
  when 12..17 then "บ่าย"
  when 18..21 then "เย็น"
  when 22..23 then "ค่ำ"
  end
end

(0..23).step(3) { |h| puts "#{h}:00 - #{part_of_day(h)}" }
```

---

## Step 361: Range กับ grep

`grep` ใช้ `===` ในการกรอง ซึ่ง Range ก็รองรับ `===`:

```ruby
# === กับ Range
puts (1..10) === 5    # true
puts (1..10) === 11   # false

# grep กับ Range
numbers = [3, 15, 7, 42, 28, 1, 99, 50]

# ตัวเลขในช่วง 1-20
small = numbers.grep(1..20)
puts small.inspect  # [3, 15, 7, 1]

# ตัวเลขในช่วง 30+
big = numbers.grep(30..)
puts big.inspect  # [42, 99, 50]

# grep กับ String
words = ["apple", "ant", "banana", "cat", "avocado"]
a_words = words.grep(/^a/)
puts a_words.inspect  # ["apple", "ant", "avocado"]

# grep_v (opposite of grep)
not_small = numbers.grep_v(1..20)
puts not_small.inspect  # [42, 28, 99, 50]

# ตัวอย่างจริง: filter log levels
log_entries = [
  { level: 1, message: "Debug info" },
  { level: 2, message: "Info message" },
  { level: 3, message: "Warning!" },
  { level: 4, message: "Error occurred" },
  { level: 5, message: "Critical failure" }
]

critical_logs = log_entries.select { |e| (3..5) === e[:level] }
critical_logs.each { |e| puts "[#{e[:level]}] #{e[:message]}" }
```

---

## Step 362: Range และ Numeric Operations

```ruby
# สร้างตัวเลขสุ่มในช่วง
range = 1..100
random_num = rand(range)
puts random_num

# sample จาก Range array
puts (1..10).to_a.sample  # สุ่มหนึ่งตัว
puts (1..10).to_a.sample(3).inspect  # สุ่ม 3 ตัว

# Arithmetic Progression
# สร้างลำดับเลขคณิต
first_20_odd = (1..40).step(2).first(20)
puts first_20_odd.inspect

# Geometric progression (ต้องใช้ each_with_object)
def geometric(start, ratio, count)
  count.times.each_with_object([start]) do |_, arr|
    arr << arr.last * ratio
  end
end

puts geometric(1, 2, 8).inspect  # [1, 2, 4, 8, 16, 32, 64, 128, 256]

# Range ในการ generate test data
test_scores = (50..100).to_a.sample(20).sort
puts test_scores.inspect
puts "Average: #{test_scores.sum.to_f / test_scores.size}"
puts "Min: #{test_scores.min}, Max: #{test_scores.max}"
```

---

## Step 363: Range กับ String Operations

```ruby
# String Range
alpha_lower = ('a'..'z').to_a
alpha_upper = ('A'..'Z').to_a
digits = ('0'..'9').to_a

puts "Lowercase: #{alpha_lower.join}"
puts "Uppercase: #{alpha_upper.join}"
puts "Digits: #{digits.join}"

# ตรวจสอบว่าเป็นตัวอักษรหรือไม่
def is_letter?(char)
  ('a'..'z').include?(char.downcase)
end

puts is_letter?('a')  # true
puts is_letter?('Z')  # true
puts is_letter?('1')  # false
puts is_letter?('!')  # false

# สร้าง Caesar cipher
def caesar_cipher(text, shift)
  text.chars.map do |char|
    if ('a'..'z').include?(char)
      shifted = ((char.ord - 'a'.ord + shift) % 26) + 'a'.ord
      shifted.chr
    elsif ('A'..'Z').include?(char)
      shifted = ((char.ord - 'A'.ord + shift) % 26) + 'A'.ord
      shifted.chr
    else
      char
    end
  end.join
end

message = "Hello, World!"
encrypted = caesar_cipher(message, 3)
decrypted = caesar_cipher(encrypted, -3)
puts "Original: #{message}"
puts "Encrypted: #{encrypted}"
puts "Decrypted: #{decrypted}"
```

---

## Step 364: Range กับ Custom Objects

```ruby
# สร้าง class ที่ใช้งานกับ Range ได้
class Temperature
  include Comparable

  attr_reader :value

  def initialize(value)
    @value = value
  end

  def <=>(other)
    @value <=> other.value
  end

  def to_s
    "#{@value}°C"
  end

  def succ
    Temperature.new(@value + 1)
  end
end

# สร้าง Range ของ Temperature
cold = Temperature.new(-10)
hot = Temperature.new(40)
temp_range = cold..hot

# ตรวจสอบ
normal = Temperature.new(20)
puts temp_range.include?(normal)    # true
puts temp_range.include?(Temperature.new(50))  # false

# iterate ได้ถ้ามี succ method
comfortable_range = Temperature.new(18)..Temperature.new(24)
comfortable_range.each do |temp|
  puts temp
end
# 18°C
# 19°C
# 20°C
# 21°C
# 22°C
# 23°C
# 24°C
```

---

## Step 365: ตัวอย่างการใช้งานจริง

```ruby
# ตัวอย่าง 1: ระบบ Booking ห้องพัก
class Hotel
  PRICE_RANGES = {
    standard: 1000..2000,
    deluxe: 2001..4000,
    suite: 4001..8000,
    penthouse: 8001..Float::INFINITY
  }.freeze

  def categorize_room(price)
    PRICE_RANGES.each do |category, range|
      return category if range.cover?(price)
    end
    :unknown
  end

  def available_rooms_in_budget(rooms, budget_range)
    rooms.select { |room| budget_range.cover?(room[:price]) }
  end
end

hotel = Hotel.new
puts hotel.categorize_room(1500)   # standard
puts hotel.categorize_room(3000)   # deluxe
puts hotel.categorize_room(10000)  # penthouse

rooms = [
  { id: 101, price: 1200, type: "standard" },
  { id: 201, price: 3500, type: "deluxe" },
  { id: 301, price: 5000, type: "suite" },
  { id: 401, price: 9000, type: "penthouse" }
]

budget_rooms = hotel.available_rooms_in_budget(rooms, 1000..4000)
budget_rooms.each { |r| puts "Room #{r[:id]}: #{r[:price]} บาท" }

# ตัวอย่าง 2: Pagination
class Paginator
  def initialize(total_items, per_page = 10)
    @total_items = total_items
    @per_page = per_page
  end

  def page_range(page_number)
    start_index = (page_number - 1) * @per_page
    end_index = [start_index + @per_page - 1, @total_items - 1].min
    start_index..end_index
  end

  def total_pages
    (@total_items.to_f / @per_page).ceil
  end
end

paginator = Paginator.new(100, 10)
puts "Total pages: #{paginator.total_pages}"
puts "Page 1 range: #{paginator.page_range(1)}"
puts "Page 5 range: #{paginator.page_range(5)}"
puts "Last page range: #{paginator.page_range(10)}"

# ตัวอย่าง 3: Validation
module Validators
  AGE_RANGE = 0..150
  PRICE_RANGE = 0.01..1_000_000.00
  SCORE_RANGE = 0..100

  def self.valid_age?(age)
    AGE_RANGE.cover?(age)
  end

  def self.valid_price?(price)
    PRICE_RANGE.cover?(price)
  end

  def self.valid_score?(score)
    SCORE_RANGE.cover?(score)
  end
end

puts Validators.valid_age?(25)     # true
puts Validators.valid_age?(-1)     # false
puts Validators.valid_age?(200)    # false
puts Validators.valid_price?(99.99) # true
puts Validators.valid_score?(85)   # true
puts Validators.valid_score?(101)  # false
```

---

## แบบฝึกหัด: Symbols และ Ranges (20 ข้อ)

### ข้อที่ 1-5: Symbols

**ข้อ 1:** สร้าง array ของ Symbols แทนวันในสัปดาห์ (7 วัน) โดยใช้ `%i[]` syntax

```ruby
# เฉลย
days = %i[monday tuesday wednesday thursday friday saturday sunday]
puts days.inspect
puts days.class  # Array
puts days.first.class  # Symbol
```

**ข้อ 2:** เขียน method ที่รับ String และแปลงเป็น Symbol แบบ snake_case (เช่น "Hello World" -> :hello_world)

```ruby
# เฉลย
def to_snake_sym(str)
  str.downcase.gsub(/\s+/, '_').to_sym
end

puts to_snake_sym("Hello World")    # :hello_world
puts to_snake_sym("First Name")     # :first_name
puts to_snake_sym("Product ID")     # :product_id
```

**ข้อ 3:** สร้าง Hash ที่มี Symbol keys แทนข้อมูลนักศึกษา และเข้าถึงค่าต่างๆ

```ruby
# เฉลย
student = {
  name: "สมชาย ใจดี",
  student_id: "ST001",
  gpa: 3.75,
  year: 3,
  major: :computer_science
}

puts student[:name]
puts student[:gpa]
puts student[:major].to_s.gsub('_', ' ').capitalize
```

**ข้อ 4:** ใช้ Symbol#to_proc กับ map เพื่อแปลง array ของ String เป็นตัวพิมพ์ใหญ่ทั้งหมด

```ruby
# เฉลย
fruits = ["apple", "banana", "cherry", "date", "elderberry"]
upper_fruits = fruits.map(&:upcase)
puts upper_fruits.inspect
```

**ข้อ 5:** สร้าง method ที่ใช้ `send` กับ Symbol เพื่อเรียก math operations

```ruby
# เฉลย
def calculate(a, b, operation)
  valid_ops = %i[+ - * /]
  raise "Invalid operation" unless valid_ops.include?(operation)
  a.send(operation, b)
end

puts calculate(10, 5, :+)  # 15
puts calculate(10, 5, :-)  # 5
puts calculate(10, 5, :*)  # 50
puts calculate(10, 5, :/)  # 2
```

### ข้อที่ 6-10: Ranges (Basic)

**ข้อ 6:** สร้าง Range ของตัวอักษรพิมพ์ใหญ่ A-Z และนับจำนวน vowels ที่อยู่ใน range

```ruby
# เฉลย
uppercase = ('A'..'Z').to_a
vowels = uppercase.select { |char| "AEIOU".include?(char) }
puts "Total letters: #{uppercase.size}"
puts "Vowels: #{vowels.inspect}"
puts "Consonants: #{uppercase.size - vowels.size}"
```

**ข้อ 7:** สร้าง Range 1..50 และหา: ผลรวม, ค่าเฉลี่ย, ตัวเลขคี่, ตัวเลขที่หาร 3 ลงตัว

```ruby
# เฉลย
range = 1..50
numbers = range.to_a

puts "ผลรวม: #{numbers.sum}"
puts "ค่าเฉลี่ย: #{numbers.sum.to_f / numbers.size}"
puts "ตัวเลขคี่: #{numbers.select(&:odd?).inspect}"
puts "หาร 3 ลงตัว: #{numbers.select { |n| n % 3 == 0 }.inspect}"
```

**ข้อ 8:** เขียน method ที่ตรวจสอบว่า IP address อยู่ในช่วงที่กำหนดหรือไม่ (อิงแค่ octet แรก)

```ruby
# เฉลย
def private_network?(ip)
  first_octet = ip.split('.').first.to_i
  private_ranges = [(10..10), (172..172), (192..192)]
  private_ranges.any? { |range| range.cover?(first_octet) }
end

puts private_network?("10.0.0.1")    # true
puts private_network?("192.168.1.1") # true
puts private_network?("8.8.8.8")     # false
```

**ข้อ 9:** สร้าง method ที่สร้าง multiplication table ด้วย Range

```ruby
# เฉลย
def multiplication_table(n)
  (1..10).each do |i|
    puts "#{n} × #{i} = #{n * i}"
  end
end

multiplication_table(7)
```

**ข้อ 10:** ใช้ Range กับ `step` เพื่อสร้าง array ของตัวเลขทศนิยม 0.0 ถึง 1.0 ทีละ 0.1

```ruby
# เฉลย
decimals = []
(0.0..1.0).step(0.1) { |n| decimals << n.round(1) }
puts decimals.inspect
# [0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
```

### ข้อที่ 11-15: Ranges (Intermediate)

**ข้อ 11:** เขียน class `DateRange` ที่ wraps Date Range และมี method นับ weekdays และ weekends

```ruby
# เฉลย
require 'date'

class DateRange
  def initialize(start_date, end_date)
    @range = start_date..end_date
  end

  def weekdays
    @range.select { |d| d.wday.between?(1, 5) }
  end

  def weekends
    @range.select { |d| d.wday == 0 || d.wday == 6 }
  end

  def count_weekdays
    weekdays.count
  end

  def count_weekends
    weekends.count
  end
end

start = Date.new(2024, 1, 1)
finish = Date.new(2024, 1, 31)
dr = DateRange.new(start, finish)
puts "Weekdays in January 2024: #{dr.count_weekdays}"
puts "Weekends in January 2024: #{dr.count_weekends}"
```

**ข้อ 12:** สร้าง Endless Range classifier สำหรับ BMI

```ruby
# เฉลย
def bmi_category(bmi)
  case bmi
  when (..18.4)   then "น้ำหนักน้อยกว่าเกณฑ์"
  when (18.5..24.9) then "น้ำหนักปกติ"
  when (25.0..29.9) then "น้ำหนักเกิน"
  when (30.0..34.9) then "โรคอ้วนระดับ 1"
  when (35.0..)     then "โรคอ้วนระดับ 2+"
  end
end

[15.0, 22.5, 27.0, 32.5, 40.0].each do |bmi|
  puts "BMI #{bmi}: #{bmi_category(bmi)}"
end
```

**ข้อ 13:** ใช้ grep กับหลาย Range เพื่อ categorize numbers

```ruby
# เฉลย
numbers = (1..100).to_a.sample(20).sort

small  = numbers.grep(1..25)
medium = numbers.grep(26..75)
large  = numbers.grep(76..100)

puts "Small (1-25): #{small.inspect}"
puts "Medium (26-75): #{medium.inspect}"
puts "Large (76-100): #{large.inspect}"
```

**ข้อ 14:** สร้าง Range ที่ cover? ตรวจสอบ time window

```ruby
# เฉลย
class BusinessHours
  MORNING = (9..12)
  AFTERNOON = (13..17)

  def self.open?(hour)
    MORNING.cover?(hour) || AFTERNOON.cover?(hour)
  end

  def self.period(hour)
    case hour
    when MORNING    then "เช้า"
    when AFTERNOON  then "บ่าย"
    else "นอกเวลาทำการ"
    end
  end
end

(8..18).each do |hour|
  status = BusinessHours.open?(hour) ? "เปิด" : "ปิด"
  puts "#{hour}:00 - #{status} (#{BusinessHours.period(hour)})"
end
```

**ข้อ 15:** สร้าง Beginless Range สำหรับ filter ราคาสินค้า

```ruby
# เฉลย
products = [
  { name: "ดินสอ", price: 10 },
  { name: "สมุด", price: 45 },
  { name: "กระเป๋า", price: 350 },
  { name: "นาฬิกา", price: 1200 },
  { name: "โทรศัพท์", price: 15000 }
]

def filter_by_price(products, range)
  products.select { |p| range.cover?(p[:price]) }
end

cheap     = filter_by_price(products, ..100)
mid_range = filter_by_price(products, 101..2000)
expensive = filter_by_price(products, 2001..)

puts "ราคาถูก (ต่ำกว่า 100): #{cheap.map { |p| p[:name] }.join(', ')}"
puts "ราคากลาง (101-2000): #{mid_range.map { |p| p[:name] }.join(', ')}"
puts "ราคาแพง (2001+): #{expensive.map { |p| p[:name] }.join(', ')}"
```

### ข้อที่ 16-20: รวม Symbols และ Ranges

**ข้อ 16:** สร้าง Hash ที่ map Symbol category ไปยัง Range ของ salary

```ruby
# เฉลย
SALARY_GRADES = {
  junior:   20_000..35_000,
  mid:      35_001..60_000,
  senior:   60_001..100_000,
  lead:     100_001..150_000,
  director: 150_001..Float::INFINITY
}.freeze

def job_grade(salary)
  SALARY_GRADES.find { |grade, range| range.cover?(salary) }&.first
end

[25000, 45000, 80000, 120000, 200000].each do |salary|
  grade = job_grade(salary)
  puts "เงินเดือน #{salary}: #{grade}"
end
```

**ข้อ 17:** สร้าง method ที่ใช้ทั้ง Symbol และ Range ในการ validate

```ruby
# เฉลย
class UserValidator
  RULES = {
    username: { range: (3..20), pattern: /\A[a-z_]\w*\z/ },
    age:      { range: (13..120) },
    score:    { range: (0..100) }
  }.freeze

  def self.valid?(field, value)
    rules = RULES[field]
    return false unless rules

    if rules[:range]
      return false unless rules[:range].cover?(value.is_a?(String) ? value.length : value)
    end

    if rules[:pattern]
      return false unless value.match?(rules[:pattern])
    end

    true
  end
end

puts UserValidator.valid?(:username, "alice")      # true
puts UserValidator.valid?(:username, "ab")         # false (too short)
puts UserValidator.valid?(:username, "123bad")     # false (pattern)
puts UserValidator.valid?(:age, 25)                # true
puts UserValidator.valid?(:age, 5)                 # false
puts UserValidator.valid?(:score, 85)              # true
puts UserValidator.valid?(:score, 101)             # false
```

**ข้อ 18:** สร้าง program ที่แสดง calendar สำหรับเดือน

```ruby
# เฉลย
require 'date'

def print_month_calendar(year, month)
  first_day = Date.new(year, month, 1)
  last_day = Date.new(year, month, -1)

  puts "#{first_day.strftime("%B %Y")}"
  puts "จ อ พ พฤ ศ ส อา"
  puts "-" * 20

  # เริ่ม padding
  start_pad = first_day.wday == 0 ? 6 : first_day.wday - 1
  print "   " * start_pad

  (first_day..last_day).each do |date|
    print "#{date.day.to_s.rjust(2)} "
    puts if date.wday == 0  # Sunday = end of week
  end
  puts
end

print_month_calendar(2024, 1)
```

**ข้อ 19:** สร้าง generator ที่ใช้ Range สร้าง test data

```ruby
# เฉลย
class TestDataGenerator
  FIRST_NAMES = %w[สมชาย สมหญิง วิทยา อรุณ ภาวนา].freeze
  LAST_NAMES  = %w[ใจดี มีสุข เจริญ รุ่งเรือง สมบูรณ์].freeze
  AGE_RANGE   = (18..65).freeze
  SCORE_RANGE = (50..100).freeze

  def self.generate_students(count)
    count.times.map do |i|
      {
        id:    "ST#{(i + 1).to_s.rjust(3, '0')}",
        name:  "#{FIRST_NAMES.sample} #{LAST_NAMES.sample}",
        age:   rand(AGE_RANGE),
        score: rand(SCORE_RANGE)
      }
    end
  end
end

students = TestDataGenerator.generate_students(5)
students.each do |s|
  puts "#{s[:id]}: #{s[:name]}, อายุ #{s[:age]}, คะแนน #{s[:score]}"
end
```

**ข้อ 20:** สร้าง Rate Limiter ที่ใช้ Range ในการ check

```ruby
# เฉลย
class RateLimiter
  LIMITS = {
    free:       1..10,
    basic:      1..100,
    pro:        1..1000,
    enterprise: 1..Float::INFINITY
  }.freeze

  def initialize(plan)
    @plan = plan
    @requests = 0
  end

  def allow_request?
    @requests += 1
    limit_range = LIMITS[@plan]
    limit_range.cover?(@requests)
  end

  def remaining
    limit = LIMITS[@plan].last
    return Float::INFINITY if limit == Float::INFINITY
    [limit - @requests, 0].max
  end
end

limiter = RateLimiter.new(:basic)
15.times do |i|
  if limiter.allow_request?
    puts "Request #{i + 1}: Allowed (remaining: #{limiter.remaining})"
  else
    puts "Request #{i + 1}: Rate limited!"
  end
end
```

---

## สรุป

ในตอนนี้เราได้เรียนรู้:

### Symbol
- **คืออะไร**: Object แทนชื่อ ที่มีความเป็น unique และ immutable
- **สร้างได้หลายวิธี**: `:name`, `:"complex name"`, `"name".to_sym`, `%i[...]`
- **Methods**: `to_s`, `to_proc`, `id2name`, `upcase`, `downcase`, `length`
- **ใช้ทำอะไร**: Hash keys, method names, callbacks, options
- **ข้อดี**: ประหยัด memory, เร็วกว่า String ในการเปรียบเทียบ

### Range
- **คืออะไร**: Object แทนช่วงค่า ระหว่าง begin และ end
- **สองประเภท**: `..` (inclusive), `...` (exclusive)
- **Methods**: `include?`, `cover?`, `each`, `to_a`, `size`, `step`, `min`, `max`
- **Endless Range** (`1..`): Ruby 2.6+ สำหรับ "1 ขึ้นไป"
- **Beginless Range** (`..10`): Ruby 2.7+ สำหรับ "จนถึง 10"
- **ใน case/when**: ทำงานได้ดีมาก
- **กับ grep**: กรอง array ด้วย Range

ทั้งสองเป็น tools ที่ทรงพลังใน Ruby ที่ช่วยให้โค้ดอ่านง่ายและทำงานได้อย่างมีประสิทธิภาพ

---

*ตอนถัดไป: ตอนที่ 18 - Enumerables*
