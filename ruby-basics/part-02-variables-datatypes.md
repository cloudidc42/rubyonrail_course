# ตอนที่ 2: Variables และ Data Types

## Ruby Programming Course สำหรับผู้เริ่มต้นภาษาไทย

---

## บทนำ

ในตอนที่ 2 นี้ เราจะเรียนรู้เกี่ยวกับ Variables (ตัวแปร) และ Data Types (ประเภทข้อมูล) ซึ่งเป็นพื้นฐานที่สำคัญที่สุดในการเขียนโปรแกรมภาษา Ruby

**สิ่งที่จะได้เรียนรู้ในตอนนี้:**
- ประเภทของ Variables ใน Ruby
- การตั้งชื่อตัวแปรตาม Convention
- ประเภทข้อมูลพื้นฐาน (Integer, Float, String, Boolean, Nil, Symbol)
- การแปลงประเภทข้อมูล
- การตรวจสอบประเภทข้อมูล
- Frozen Objects
- การกำหนดค่าหลายตัวแปรพร้อมกัน

---

## Step 11: ประเภทของ Variables ใน Ruby

ใน Ruby มี Variables หลายประเภท แต่ละประเภทมี Scope (ขอบเขต) การใช้งานที่แตกต่างกัน

### 1.1 Local Variables (ตัวแปรท้องถิ่น)

Local Variables เป็นตัวแปรที่ใช้งานได้เฉพาะภายใน Block, Method, หรือ Scope ที่ประกาศไว้เท่านั้น

**กฎการตั้งชื่อ:**
- ต้องขึ้นต้นด้วยตัวอักษรพิมพ์เล็ก หรือ underscore (_)
- ประกอบด้วยตัวอักษร ตัวเลข และ underscore

```ruby
# ตัวอย่าง Local Variables
name = "สมชาย"
age = 25
_temp_value = 100
student_name = "นักเรียน A"

puts name        # => สมชาย
puts age         # => 25
puts _temp_value # => 100

# Local Variable ไม่สามารถเข้าถึงจากนอก Scope ได้
def my_method
  local_var = "ฉันอยู่ใน method"
  puts local_var
end

my_method  # => ฉันอยู่ใน method
# puts local_var  # => NameError: undefined local variable
```

```ruby
# ตัวอย่างการใช้ Local Variable ใน Block
[1, 2, 3].each do |number|
  square = number ** 2
  puts "#{number} ยกกำลังสอง = #{square}"
end
# => 1 ยกกำลังสอง = 1
# => 2 ยกกำลังสอง = 4
# => 3 ยกกำลังสอง = 9

# ตัวแปร square ไม่สามารถเข้าถึงจากนอก block ได้ใน Ruby 1.9+
```

### 1.2 Instance Variables (ตัวแปร Instance)

Instance Variables ใช้เก็บข้อมูลของ Object แต่ละตัว ขึ้นต้นด้วย `@`

```ruby
class Person
  def initialize(name, age)
    @name = name  # Instance Variable
    @age = age    # Instance Variable
  end
  
  def introduce
    puts "สวัสดี ฉันชื่อ #{@name} อายุ #{@age} ปี"
  end
  
  def birthday
    @age += 1
    puts "อายุใหม่: #{@age}"
  end
end

person1 = Person.new("สมชาย", 25)
person2 = Person.new("สมหญิง", 22)

person1.introduce  # => สวัสดี ฉันชื่อ สมชาย อายุ 25 ปี
person2.introduce  # => สวัสดี ฉันชื่อ สมหญิง อายุ 22 ปี

person1.birthday   # => อายุใหม่: 26
person1.introduce  # => สวัสดี ฉันชื่อ สมชาย อายุ 26 ปี
person2.introduce  # => สวัสดี ฉันชื่อ สมหญิง อายุ 22 ปี (ไม่เปลี่ยน)
```

```ruby
# Instance Variable ที่ไม่ได้กำหนดค่า จะมีค่าเป็น nil
class Dog
  def show_info
    puts @name.inspect  # => nil
    puts @breed.inspect # => nil
  end
  
  def set_name(name)
    @name = name
  end
end

dog = Dog.new
dog.show_info   # => nil, nil
dog.set_name("บัดดี้")
dog.show_info   # => "บัดดี้", nil
```

### 1.3 Class Variables (ตัวแปร Class)

Class Variables ใช้เก็บข้อมูลที่ใช้ร่วมกันในทุก Instance ของ Class ขึ้นต้นด้วย `@@`

```ruby
class BankAccount
  @@total_accounts = 0    # Class Variable
  @@total_balance = 0.0   # Class Variable
  
  def initialize(owner, balance)
    @owner = owner
    @balance = balance
    @@total_accounts += 1
    @@total_balance += balance
  end
  
  def self.total_accounts
    @@total_accounts
  end
  
  def self.total_balance
    @@total_balance
  end
  
  def deposit(amount)
    @balance += amount
    @@total_balance += amount
  end
end

acc1 = BankAccount.new("สมชาย", 1000.0)
acc2 = BankAccount.new("สมหญิง", 2000.0)
acc3 = BankAccount.new("สมศรี", 500.0)

puts "จำนวนบัญชีทั้งหมด: #{BankAccount.total_accounts}"  # => 3
puts "ยอดเงินรวมทั้งหมด: #{BankAccount.total_balance}"   # => 3500.0

acc1.deposit(500.0)
puts "ยอดเงินรวมทั้งหมด: #{BankAccount.total_balance}"   # => 4000.0
```

### 1.4 Global Variables (ตัวแปร Global)

Global Variables เข้าถึงได้จากทุกที่ในโปรแกรม ขึ้นต้นด้วย `$` ควรใช้อย่างระมัดระวัง

```ruby
$app_name = "Ruby Course App"
$version = "1.0.0"
$debug_mode = false

def show_app_info
  puts "Application: #{$app_name}"
  puts "Version: #{$version}"
  puts "Debug: #{$debug_mode}"
end

def enable_debug
  $debug_mode = true
end

show_app_info
# => Application: Ruby Course App
# => Version: 1.0.0
# => Debug: false

enable_debug
show_app_info
# => Application: Ruby Course App
# => Version: 1.0.0
# => Debug: true
```

```ruby
# Ruby มี Global Variables ที่กำหนดไว้ล่วงหน้า (Predefined Global Variables)
puts $0      # ชื่อไฟล์ที่กำลังรัน
puts $PROGRAM_NAME  # เหมือน $0
puts $$      # Process ID
puts $stdout.class  # => IO
puts $stderr.class  # => IO
puts $stdin.class   # => IO

# $_ เก็บค่าล่าสุดที่อ่านจาก gets
# $! เก็บ Exception ล่าสุด
# $@ เก็บ Backtrace ของ Exception ล่าสุด
```

---

## Step 12: การตั้งชื่อ Variable (Naming Conventions)

Ruby มี Convention การตั้งชื่อที่ชัดเจน การทำตาม Convention จะทำให้โค้ดอ่านได้ง่ายขึ้น

### 2.1 Snake_case สำหรับ Variables และ Methods

```ruby
# ถูกต้อง - snake_case
student_name = "สมชาย"
total_price = 1500.0
is_logged_in = true
max_retry_count = 3

# ผิด Convention (แม้จะทำงานได้)
studentName = "สมชาย"    # camelCase - ไม่ใช้ใน Ruby
StudentName = "สมชาย"    # PascalCase - ใช้สำหรับ Class/Module เท่านั้น
```

```ruby
# ชื่อ Method ก็ใช้ snake_case
def calculate_total_price(price, quantity, discount_rate)
  subtotal = price * quantity
  discount = subtotal * discount_rate
  subtotal - discount
end

result = calculate_total_price(100.0, 5, 0.1)
puts result  # => 450.0
```

### 2.2 PascalCase สำหรับ Classes และ Modules

```ruby
# Classes
class ShoppingCart
end

class UserAuthentication
end

class DatabaseConnection
end

# Modules
module PaymentProcessor
end

module UserNotification
end
```

### 2.3 SCREAMING_SNAKE_CASE สำหรับ Constants

```ruby
MAX_CONNECTIONS = 100
DEFAULT_TIMEOUT = 30
PI = 3.14159
DATABASE_URL = "postgresql://localhost/mydb"
APP_VERSION = "2.0.0"

puts MAX_CONNECTIONS   # => 100
puts PI                # => 3.14159
```

### 2.4 กฎพิเศษของ Ruby

```ruby
# ชื่อที่ลงท้ายด้วย ? หมายถึง predicate methods (คืนค่า boolean)
name = "สมชาย"
puts name.empty?    # => false
puts name.include?("สม")  # => true

number = 5
puts number.odd?    # => true
puts number.even?   # => false
puts number.zero?   # => false

# ชื่อที่ลงท้ายด้วย ! หมายถึงเปลี่ยนแปลง object ต้นฉบับ (bang methods)
words = ["banana", "apple", "cherry"]
words.sort!   # เรียงและเปลี่ยน array ต้นฉบับ
puts words.inspect  # => ["apple", "banana", "cherry"]

text = "  hello world  "
text.strip!   # ลบช่องว่างหัวท้ายและเปลี่ยนต้นฉบับ
puts text     # => "hello world"
```

---

## Step 13: Integer - ตัวเลขจำนวนเต็ม

Integer คือตัวเลขจำนวนเต็มไม่มีทศนิยม ใน Ruby ไม่มีขีดจำกัดขนาด

### 3.1 การสร้าง Integer

```ruby
# Integer ธรรมดา
age = 25
population = 1_000_000   # ใช้ underscore แทนเครื่องหมายจุลภาคเพื่อให้อ่านง่าย
negative = -42

# Integer ขนาดใหญ่มาก (Ruby รองรับ Bignum)
big_number = 9_999_999_999_999_999
factorial_100 = 93326215443944152681699238856266700490715968264381621468592963895217599993229915608941463976156518286253697920827223758251185210916864000000000000000000000000

puts age         # => 25
puts population  # => 1000000
puts big_number  # => 9999999999999999
```

```ruby
# Integer ในฐานต่างๆ
binary_num = 0b1010    # เลขฐาน 2 = 10
octal_num  = 0o17      # เลขฐาน 8 = 15
hex_num    = 0xFF      # เลขฐาน 16 = 255

puts binary_num   # => 10
puts octal_num    # => 15
puts hex_num      # => 255

# แปลงเป็นฐานต่างๆ
num = 255
puts num.to_s(2)   # => "11111111" (binary)
puts num.to_s(8)   # => "377" (octal)
puts num.to_s(16)  # => "ff" (hexadecimal)
```

### 3.2 Integer Methods

```ruby
num = -42

# คณิตศาสตร์พื้นฐาน
puts num.abs        # => 42 (ค่าสัมบูรณ์)
puts num.abs2       # => 1764 (กำลังสอง)

# การตรวจสอบ
puts 5.odd?         # => true (เลขคี่)
puts 4.even?        # => true (เลขคู่)
puts 0.zero?        # => true (เป็นศูนย์)
puts 5.positive?    # => true (เป็นบวก)
puts (-3).negative? # => true (เป็นลบ)

# การหาค่า
puts 15.gcd(10)     # => 5 (ห.ร.ม.)
puts 4.lcm(6)       # => 12 (ค.ร.น.)

# Integer Division
puts 17.divmod(5)   # => [3, 2] (몫 และ เศษ)
puts 17.div(5)      # => 3 (ผลหาร)
puts 17.modulo(5)   # => 2 (เศษ)
puts 17.remainder(5) # => 2 (เศษ)
```

```ruby
# การวนซ้ำด้วย Integer
5.times do |i|
  print "#{i} "
end
puts  # => 0 1 2 3 4

1.upto(5) do |i|
  print "#{i} "
end
puts  # => 1 2 3 4 5

5.downto(1) do |i|
  print "#{i} "
end
puts  # => 5 4 3 2 1

1.step(10, 2) do |i|
  print "#{i} "
end
puts  # => 1 3 5 7 9
```

```ruby
# Integer Arithmetic
a = 17
b = 5

puts a + b   # => 22
puts a - b   # => 12
puts a * b   # => 85
puts a / b   # => 3 (Integer division!)
puts a % b   # => 2 (modulo)
puts a ** b  # => 1419857 (exponentiation)

# การแปลง
puts 42.to_f   # => 42.0
puts 42.to_r   # => 42/1 (Rational)
puts 42.to_c   # => 42+0i (Complex)
puts 42.to_s   # => "42"
```

---

## Step 14: Float - ตัวเลขทศนิยม

Float คือตัวเลขที่มีทศนิยม ใช้สำหรับการคำนวณที่ต้องการความแม่นยำ

### 4.1 การสร้าง Float

```ruby
price = 29.99
temperature = -5.5
pi = 3.14159265358979
scientific = 1.5e10    # 1.5 × 10^10

puts price        # => 29.99
puts temperature  # => -5.5
puts scientific   # => 15000000000.0
```

### 4.2 Float Methods

```ruby
num = 3.7

puts num.ceil      # => 4 (ปัดขึ้น)
puts num.floor     # => 3 (ปัดลง)
puts num.round     # => 4 (ปัดใกล้สุด)
puts num.truncate  # => 3 (ตัดทศนิยมทิ้ง)
puts num.abs       # => 3.7 (ค่าสัมบูรณ์)

# การปัดเศษหลายตำแหน่ง
pi = 3.14159
puts pi.round(2)  # => 3.14
puts pi.round(4)  # => 3.1416

# Float พิเศษ
infinity = Float::INFINITY
negative_inf = -Float::INFINITY
not_a_number = Float::NAN

puts infinity.infinite?      # => 1
puts negative_inf.infinite?  # => -1
puts pi.infinite?            # => nil
puts not_a_number.nan?       # => true
puts pi.nan?                 # => false
puts pi.finite?              # => true
```

```ruby
# ระวัง Floating Point Precision
puts 0.1 + 0.2          # => 0.30000000000000004
puts (0.1 + 0.2).round(1)  # => 0.3

# ใช้ BigDecimal สำหรับการคำนวณที่ต้องการความแม่นยำสูง
require 'bigdecimal'
a = BigDecimal("0.1")
b = BigDecimal("0.2")
puts (a + b).to_s   # => "0.3"
```

---

## Step 15: String - ข้อความ

String คือลำดับของตัวอักษร สร้างได้หลายแบบ

### 5.1 การสร้าง String

```ruby
# Single quotes - ไม่แปลง escape sequences (ยกเว้น \\ และ \')
single = 'Hello, World!'
single_escape = 'It\'s a beautiful day'
no_interpolation = 'ราคา: #{price}'   # #{} จะไม่ถูกแปลง

# Double quotes - แปลง escape sequences และ interpolation
double = "Hello, World!"
with_tab = "ชื่อ:\tสมชาย"
with_newline = "บรรทัดที่ 1\nบรรทัดที่ 2"
with_interpolation = "ราคา: #{29.99}"

puts single              # => Hello, World!
puts no_interpolation    # => ราคา: #{price}
puts with_interpolation  # => ราคา: 29.99
```

```ruby
# Escape Sequences
puts "Tab:\tHello"          # Tab
puts "Newline:\nWorld"      # Newline
puts "Backslash: \\"        # Backslash
puts "Quote: \""            # Double quote
puts "Null: \0End"          # Null character
puts "Bell: \a"             # Bell (ASCII 7)
```

### 5.2 String Methods พื้นฐาน

```ruby
text = "  Hello, Ruby World!  "

puts text.length      # => 22
puts text.size        # => 22 (เหมือน length)
puts text.upcase      # => "  HELLO, RUBY WORLD!  "
puts text.downcase    # => "  hello, ruby world!  "
puts text.capitalize  # => "  hello, ruby world!  " (ตัวแรกพิมพ์ใหญ่)
puts text.strip       # => "Hello, Ruby World!"
puts text.lstrip      # => "Hello, Ruby World!  "
puts text.rstrip      # => "  Hello, Ruby World!"
puts text.reverse     # => "  !dlroW ybuR ,olleH  "
```

---

## Step 16: Boolean - true/false

Boolean มีเพียงสองค่า: `true` และ `false`

### 6.1 การใช้ Boolean

```ruby
is_student = true
is_graduated = false
has_job = true

puts is_student     # => true
puts is_graduated   # => false
puts is_student.class  # => TrueClass
puts is_graduated.class # => FalseClass

# Logical Operations
puts true && true    # => true
puts true && false   # => false
puts false || true   # => true
puts false || false  # => false
puts !true           # => false
puts !false          # => true

# Truthiness ใน Ruby
# เฉพาะ false และ nil เท่านั้นที่เป็น falsy
# ทุกอย่างอื่นเป็น truthy (รวมถึง 0 และ "")
if 0
  puts "0 เป็น truthy ใน Ruby!"  # จะแสดงข้อความนี้
end

if ""
  puts "String ว่างเป็น truthy ใน Ruby!"  # จะแสดงข้อความนี้
end
```

```ruby
# Boolean ในการเปรียบเทียบ
a = 10
b = 20

puts a == b    # => false
puts a != b    # => true
puts a < b     # => true
puts a > b     # => false
puts a <= b    # => true
puts a >= b    # => false

# Spaceship operator
puts a <=> b   # => -1 (a น้อยกว่า b)
puts b <=> a   # => 1  (b มากกว่า a)
puts a <=> a   # => 0  (เท่ากัน)
```

---

## Step 17: Nil - ค่าว่าง

Nil คือค่าว่าง ไม่มีค่า ใช้แทนการไม่มีอยู่ของข้อมูล

### 7.1 การใช้ Nil

```ruby
no_value = nil
empty_name = nil

puts no_value         # => (ไม่แสดงอะไร)
puts no_value.inspect # => nil
puts no_value.class   # => NilClass
puts no_value.nil?    # => true

# nil เป็น falsy
if no_value
  puts "มีค่า"
else
  puts "ไม่มีค่า"  # จะแสดงข้อความนี้
end

# Safe Navigation Operator (&.) - ป้องกัน NoMethodError
name = nil
puts name&.upcase   # => nil (ไม่เกิด error)
puts name&.length   # => nil

name = "สมชาย"
puts name&.upcase   # => สมชาย
```

```ruby
# nil? vs false?
puts nil.nil?    # => true
puts false.nil?  # => false
puts 0.nil?      # => false
puts "".nil?     # => false

# Nil Coalescing ด้วย ||
user_name = nil
display_name = user_name || "ผู้ใช้ไม่ระบุชื่อ"
puts display_name   # => ผู้ใช้ไม่ระบุชื่อ

user_name = "สมชาย"
display_name = user_name || "ผู้ใช้ไม่ระบุชื่อ"
puts display_name   # => สมชาย

# ||= (assign ถ้าค่าเป็น nil หรือ false)
score = nil
score ||= 0
puts score  # => 0

score ||= 100
puts score  # => 0 (ไม่เปลี่ยน เพราะ score ไม่ใช่ nil แล้ว)
```

---

## Step 18: Symbol - :symbols

Symbol คือ identifier ที่ immutable (เปลี่ยนค่าไม่ได้) มักใช้เป็น key ใน Hash

### 8.1 การสร้างและใช้ Symbol

```ruby
# การสร้าง Symbol
status = :active
role = :admin
color = :red

puts status       # => active
puts status.class # => Symbol
puts :hello == :hello   # => true (Symbol เดียวกันคือ object เดียวกัน)

# Symbol กับ String
puts :hello.to_s         # => "hello"
puts "hello".to_sym      # => :hello
puts :hello == "hello"   # => false (คนละ class)
puts :hello.equal?(:hello)  # => true (object เดียวกัน!)
puts "hello".equal?("hello")  # => false (object ต่างกัน)
```

```ruby
# Symbol ใน Hash (ใช้บ่อยมาก)
user = {
  name: "สมชาย",      # :name => "สมชาย"
  age: 25,             # :age => 25
  role: :admin         # :role => :admin
}

puts user[:name]   # => สมชาย
puts user[:age]    # => 25
puts user[:role]   # => admin

# Symbol Methods
sym = :hello_world
puts sym.to_s          # => "hello_world"
puts sym.upcase        # => :HELLO_WORLD
puts sym.length        # => 11
puts sym.to_proc.call("test")  # => "test" (แปลง method call)
```

```ruby
# Symbol Array (%i)
colors = %i[red green blue yellow]
puts colors.inspect  # => [:red, :green, :blue, :yellow]
puts colors.class    # => Array
puts colors[0]       # => red
puts colors[0].class # => Symbol

# All Symbols ใน program
# Symbol.all_symbols.first(5).each { |s| puts s }
```

---

## Step 19: Type Conversion (การแปลงประเภทข้อมูล)

Ruby มี Method สำหรับแปลงประเภทข้อมูลหลายรูปแบบ

### 9.1 Explicit Conversion Methods

```ruby
# to_i - แปลงเป็น Integer
puts "42".to_i         # => 42
puts "3.14".to_i       # => 3 (ตัดทศนิยม)
puts "42abc".to_i      # => 42 (หยุดที่อักษร)
puts "abc".to_i        # => 0 (ไม่ใช่ตัวเลข)
puts 3.7.to_i          # => 3 (ตัดทศนิยม)
puts nil.to_i          # => 0
puts true.to_i rescue puts "ไม่สามารถแปลงได้"  # => error
```

```ruby
# to_f - แปลงเป็น Float
puts "3.14".to_f     # => 3.14
puts "42".to_f       # => 42.0
puts "3.14abc".to_f  # => 3.14
puts "abc".to_f      # => 0.0
puts 42.to_f         # => 42.0
puts nil.to_f        # => 0.0
```

```ruby
# to_s - แปลงเป็น String
puts 42.to_s         # => "42"
puts 3.14.to_s       # => "3.14"
puts true.to_s       # => "true"
puts false.to_s      # => "false"
puts nil.to_s        # => ""
puts nil.to_s.empty? # => true
puts :hello.to_s     # => "hello"
puts [1,2,3].to_s    # => "[1, 2, 3]"
```

```ruby
# to_a - แปลงเป็น Array
puts nil.to_a.inspect         # => []
puts {a: 1, b: 2}.to_a.inspect  # => [[:a, 1], [:b, 2]]
puts (1..5).to_a.inspect      # => [1, 2, 3, 4, 5]
puts "hello".chars.inspect    # => ["h", "e", "l", "l", "o"]
```

```ruby
# to_h - แปลงเป็น Hash
pairs = [[:name, "สมชาย"], [:age, 25]]
puts pairs.to_h.inspect  # => {:name=>"สมชาย", :age=>25}

# Integer() Float() String() - Strict Conversion (raise Error ถ้าแปลงไม่ได้)
puts Integer("42")     # => 42
puts Float("3.14")     # => 3.14

begin
  Integer("abc")
rescue ArgumentError => e
  puts "Error: #{e.message}"  # => Error: invalid value for Integer()
end
```

---

## Step 20: Checking Types (การตรวจสอบประเภทข้อมูล)

### 10.1 การตรวจสอบประเภท

```ruby
# .class - ดู class ของ object
puts 42.class         # => Integer
puts 3.14.class       # => Float
puts "hello".class    # => String
puts true.class       # => TrueClass
puts false.class      # => FalseClass
puts nil.class        # => NilClass
puts :symbol.class    # => Symbol
puts [1,2].class      # => Array
puts({a: 1}.class)    # => Hash
```

```ruby
# is_a? / kind_of? - ตรวจสอบว่าเป็น class หรือไม่
puts 42.is_a?(Integer)   # => true
puts 42.is_a?(Numeric)   # => true (Integer เป็น subclass ของ Numeric)
puts 42.is_a?(Float)     # => false
puts 42.is_a?(Object)    # => true (ทุกอย่างเป็น Object)

puts "hello".kind_of?(String)  # => true
puts "hello".kind_of?(Object)  # => true

# instance_of? - ตรวจสอบว่าเป็น class นั้นๆ ตรงๆ (ไม่รวม subclass)
puts 42.instance_of?(Integer)  # => true
puts 42.instance_of?(Numeric)  # => false!
```

```ruby
# respond_to? - ตรวจสอบว่า object มี method นั้นหรือไม่
puts "hello".respond_to?(:upcase)   # => true
puts "hello".respond_to?(:times)    # => false
puts 42.respond_to?(:times)         # => true
puts 42.respond_to?(:upcase)        # => false
puts nil.respond_to?(:nil?)         # => true
puts nil.respond_to?(:nonexistent)  # => false

# ใช้ respond_to? เพื่อเขียนโค้ดที่ยืดหยุ่น
def process(value)
  if value.respond_to?(:each)
    value.each { |v| puts "Item: #{v}" }
  else
    puts "Value: #{value}"
  end
end

process([1, 2, 3])     # => Item: 1, Item: 2, Item: 3
process("hello")        # => Value: hello
process(42)             # => Value: 42
```

```ruby
# Comparable Methods
puts 5.between?(1, 10)    # => true
puts 15.between?(1, 10)   # => false
puts 5.clamp(1, 10)       # => 5
puts 0.clamp(1, 10)       # => 1
puts 15.clamp(1, 10)      # => 10
```

---

## Step 21: Frozen Objects

Ruby 3+ ทำให้ String literals เป็น Frozen โดยอัตโนมัติ แต่เราสามารถ freeze object ได้เอง

### 11.1 freeze และ frozen?

```ruby
# Freeze object
str = "Hello"
str.freeze

puts str.frozen?  # => true

begin
  str << " World"  # พยายามเปลี่ยน frozen string
rescue FrozenError => e
  puts "Error: #{e.message}"  # => Error: can't modify frozen String
end

# Numbers และ Symbol เป็น frozen โดยอัตโนมัติ
puts 42.frozen?       # => true
puts :symbol.frozen?  # => true
puts true.frozen?     # => true
puts nil.frozen?      # => true
```

```ruby
# Frozen String Literal Comment
# เพิ่มที่ต้นไฟล์: # frozen_string_literal: true
# ทำให้ทุก String literal เป็น frozen

# ตัวอย่างการใช้ freeze เพื่อประหยัด memory
COLORS = ["red", "green", "blue"].freeze
STATUSES = { active: "Active", inactive: "Inactive" }.freeze

begin
  COLORS.push("yellow")
rescue FrozenError => e
  puts "ไม่สามารถเพิ่ม element ใน frozen array: #{e.message}"
end

# แต่สามารถอ่านได้
puts COLORS.inspect   # => ["red", "green", "blue"]
puts STATUSES[:active]  # => Active
```

```ruby
# dup vs clone - ความแตกต่างเมื่อ copy frozen object
frozen_str = "hello".freeze

duped  = frozen_str.dup    # copy แต่ไม่ frozen
cloned = frozen_str.clone  # copy และยัง frozen

puts duped.frozen?   # => false
puts cloned.frozen?  # => true

duped << " world"    # OK
puts duped           # => hello world

begin
  cloned << " world"
rescue FrozenError
  puts "ไม่สามารถเปลี่ยน cloned frozen string"
end
```

---

## Step 22: Multiple Assignment (การกำหนดค่าหลายตัวแปร)

### 12.1 Parallel Assignment

```ruby
# กำหนดค่าหลายตัวแปรพร้อมกัน
a, b, c = 1, 2, 3
puts "#{a}, #{b}, #{c}"  # => 1, 2, 3

# ถ้าค่าน้อยกว่าตัวแปร ที่เหลือจะเป็น nil
x, y, z = 10, 20
puts "#{x}, #{y}, #{z}"  # => 10, 20,

# ถ้าค่ามากกว่าตัวแปร ค่าที่เกินจะถูกละทิ้ง
p, q = 1, 2, 3, 4
puts "#{p}, #{q}"  # => 1, 2
```

```ruby
# Splat operator (*) - เก็บค่าที่เหลือ
first, *rest = [1, 2, 3, 4, 5]
puts "first: #{first}"        # => first: 1
puts "rest: #{rest.inspect}"  # => rest: [2, 3, 4, 5]

*beginning, last = [1, 2, 3, 4, 5]
puts "beginning: #{beginning.inspect}"  # => beginning: [1, 2, 3, 4]
puts "last: #{last}"                    # => last: 5

first, *middle, last = [1, 2, 3, 4, 5]
puts "first: #{first}"          # => first: 1
puts "middle: #{middle.inspect}" # => middle: [2, 3, 4]
puts "last: #{last}"            # => last: 5
```

### 12.2 Swap ตัวแปร

```ruby
# การสลับค่าระหว่างตัวแปร
a = 10
b = 20

puts "ก่อน: a=#{a}, b=#{b}"  # => ก่อน: a=10, b=20

# Swap แบบ Ruby
a, b = b, a

puts "หลัง: a=#{a}, b=#{b}"  # => หลัง: a=20, b=10

# Swap 3 ตัวแปร
x = 1
y = 2
z = 3

x, y, z = z, x, y
puts "x=#{x}, y=#{y}, z=#{z}"  # => x=3, y=1, z=2
```

```ruby
# Destructuring จาก Array
point = [10, 20]
x, y = point
puts "x=#{x}, y=#{y}"  # => x=10, y=20

# Destructuring จาก Method ที่คืน Array
def min_max(array)
  [array.min, array.max]
end

min, max = min_max([5, 3, 8, 1, 9])
puts "min=#{min}, max=#{max}"  # => min=1, max=9

# Nested destructuring
first, (second_a, second_b), third = [1, [2, 3], 4]
puts "#{first}, #{second_a}, #{second_b}, #{third}"  # => 1, 2, 3, 4
```

---

## Step 23: Constants (ค่าคงที่)

### 13.1 การใช้ Constants

```ruby
# Constants ต้องขึ้นต้นด้วยตัวพิมพ์ใหญ่
MAX_SIZE = 100
PI = 3.14159265358979
SITE_NAME = "Ruby Course"
SUPPORTED_FORMATS = ["jpg", "png", "gif"].freeze

puts MAX_SIZE      # => 100
puts PI            # => 3.14159265358979
puts SITE_NAME     # => Ruby Course

# Constants ใน Class
class Circle
  PI = 3.14159265358979
  
  def initialize(radius)
    @radius = radius
  end
  
  def area
    PI * @radius ** 2
  end
  
  def circumference
    2 * PI * @radius
  end
end

circle = Circle.new(5)
puts circle.area.round(2)          # => 78.54
puts circle.circumference.round(2) # => 31.42
puts Circle::PI                    # => 3.14159265358979
```

```ruby
# Ruby จะเตือนถ้าพยายามเปลี่ยนค่า Constant
MAX_SIZE = 100
# MAX_SIZE = 200  # => warning: already initialized constant MAX_SIZE

# ป้องกัน Constants Array/Hash ด้วย freeze
VALID_COLORS = %w[red green blue yellow].freeze
CONFIG = { max_connections: 100, timeout: 30 }.freeze

puts VALID_COLORS.inspect
puts CONFIG.inspect
```

---

## Step 24: การใช้งานขั้นสูง

### 14.1 Object ID และ Object Identity

```ruby
# object_id - ID ที่ unique ของ object
puts 1.object_id    # => 3 (fixed: 2*n+1 สำหรับ Integer)
puts 2.object_id    # => 5
puts true.object_id  # => 2 (fixed)
puts false.object_id # => 0 (fixed)
puts nil.object_id   # => 8 (fixed)

# Symbol ใช้ object เดียวกัน
puts :hello.object_id == :hello.object_id  # => true

# String ใช้ object ต่างกัน
str1 = "hello"
str2 = "hello"
puts str1.object_id == str2.object_id  # => false
puts str1 == str2                       # => true (ค่าเท่ากัน)

# equal? - ตรวจสอบว่าเป็น object เดียวกัน (ตรวจ object_id)
puts str1.equal?(str2)  # => false
puts :hello.equal?(:hello)  # => true
```

```ruby
# Frozen String Interning (String.intern)
str1 = "hello".freeze
str2 = "hello".freeze

# ใน Ruby บางกรณี frozen string อาจเป็น object เดียวกัน
puts str1.frozen?  # => true
puts str2.frozen?  # => true
```

### 14.2 Encoding

```ruby
# String encoding
str = "สวัสดีครับ"
puts str.encoding          # => UTF-8

# ตรวจสอบ valid encoding
puts str.valid_encoding?   # => true
puts str.length            # => 10 (จำนวนตัวอักษร)
puts str.bytesize          # => 30 (จำนวน bytes ใน UTF-8)
```

---

## Step 25: แบบฝึกหัด

### แบบฝึกหัดที่ 1
สร้าง Local Variables สำหรับข้อมูลนักเรียน แล้วแสดงผล

```ruby
# เฉลย
student_id = "ST001"
student_name = "นางสาวมาลี"
student_grade = "A"
student_gpa = 3.85
is_honor_student = true

puts "รหัส: #{student_id}"
puts "ชื่อ: #{student_name}"
puts "เกรด: #{student_grade}"
puts "GPA: #{student_gpa}"
puts "เกียรตินิยม: #{is_honor_student ? 'ใช่' : 'ไม่ใช่'}"
```

### แบบฝึกหัดที่ 2
สร้าง Class สินค้า พร้อม Class Variable นับจำนวน

```ruby
# เฉลย
class Product
  @@count = 0
  
  attr_reader :name, :price
  
  def initialize(name, price)
    @name = name
    @price = price
    @@count += 1
  end
  
  def self.count
    @@count
  end
end

Product.new("แอปเปิ้ล", 50)
Product.new("กล้วย", 20)
Product.new("มะม่วง", 80)

puts "สินค้าทั้งหมด: #{Product.count} รายการ"  # => 3
```

### แบบฝึกหัดที่ 3
ทดสอบการแปลงประเภทข้อมูล

```ruby
# เฉลย
values = ["42", "3.14", "hello", nil, true, false, :symbol]

values.each do |val|
  puts "#{val.inspect} => to_s: #{val.to_s.inspect}, class: #{val.class}"
end
```

### แบบฝึกหัดที่ 4
ใช้ Multiple Assignment เรียงลำดับตัวเลข

```ruby
# เฉลย
a = 30
b = 10
c = 20

# เรียงจากน้อยไปมาก
a, b, c = [a, b, c].sort
puts "#{a}, #{b}, #{c}"  # => 10, 20, 30
```

### แบบฝึกหัดที่ 5
ตรวจสอบประเภทข้อมูลและบอก type

```ruby
# เฉลย
def describe_type(value)
  case value
  when Integer then "จำนวนเต็ม"
  when Float   then "ทศนิยม"
  when String  then "ข้อความ"
  when TrueClass, FalseClass then "บูลีน"
  when NilClass  then "ค่าว่าง"
  when Symbol    then "สัญลักษณ์"
  when Array     then "อาร์เรย์"
  when Hash      then "แฮช"
  else "ไม่รู้จัก"
  end
end

puts describe_type(42)        # => จำนวนเต็ม
puts describe_type(3.14)      # => ทศนิยม
puts describe_type("hello")   # => ข้อความ
puts describe_type(true)      # => บูลีน
puts describe_type(nil)       # => ค่าว่าง
puts describe_type(:foo)      # => สัญลักษณ์
puts describe_type([])        # => อาร์เรย์
puts describe_type({})        # => แฮช
```

### แบบฝึกหัดที่ 6
สร้าง Frozen Configuration

```ruby
# เฉลย
DATABASE_CONFIG = {
  host: "localhost",
  port: 5432,
  name: "myapp_production",
  pool: 5
}.freeze

puts "Host: #{DATABASE_CONFIG[:host]}"
puts "Port: #{DATABASE_CONFIG[:port]}"

begin
  DATABASE_CONFIG[:host] = "remote.server.com"
rescue FrozenError => e
  puts "ไม่สามารถเปลี่ยนค่าได้: #{e.message}"
end
```

### แบบฝึกหัดที่ 7
ใช้ Symbol แทน String ใน Hash

```ruby
# เฉลย
# ไม่ดี (ใช้ String)
user_string = {
  "name" => "สมชาย",
  "age" => 25,
  "email" => "somchai@example.com"
}

# ดี (ใช้ Symbol)
user_symbol = {
  name: "สมชาย",
  age: 25,
  email: "somchai@example.com"
}

puts user_symbol[:name]
puts user_symbol[:age]

# Symbol ประหยัด memory มากกว่า String
str_a = "status"
str_b = "status"
sym_a = :status
sym_b = :status

puts "String same object? #{str_a.equal?(str_b)}"  # => false
puts "Symbol same object? #{sym_a.equal?(sym_b)}"  # => true
```

### แบบฝึกหัดที่ 8
ใช้ respond_to? เขียน flexible method

```ruby
# เฉลย
def smart_display(value)
  if value.respond_to?(:join)
    puts "Array: #{value.join(', ')}"
  elsif value.respond_to?(:keys)
    puts "Hash keys: #{value.keys.join(', ')}"
  elsif value.respond_to?(:to_str)
    puts "String: #{value}"
  else
    puts "Other (#{value.class}): #{value}"
  end
end

smart_display([1, 2, 3])                # => Array: 1, 2, 3
smart_display({a: 1, b: 2})            # => Hash keys: a, b
smart_display("Hello Ruby")             # => String: Hello Ruby
smart_display(42)                       # => Other (Integer): 42
```

### แบบฝึกหัดที่ 9
Destructuring Complex Data

```ruby
# เฉลย
students = [
  ["ST001", "สมชาย", [90, 85, 92]],
  ["ST002", "สมหญิง", [88, 95, 80]],
  ["ST003", "สมศรี", [75, 82, 88]]
]

students.each do |id, name, scores|
  avg = scores.sum.to_f / scores.length
  puts "#{id}: #{name} - เฉลี่ย #{avg.round(1)}"
end
# ST001: สมชาย - เฉลี่ย 89.0
# ST002: สมหญิง - เฉลี่ย 87.7
# ST003: สมศรี - เฉลี่ย 81.7
```

### แบบฝึกหัดที่ 10
Type Checking and Safe Operations

```ruby
# เฉลย
def safe_divide(a, b)
  unless a.is_a?(Numeric) && b.is_a?(Numeric)
    return "Error: ต้องการตัวเลข"
  end
  
  return "Error: ไม่สามารถหารด้วยศูนย์" if b.zero?
  
  result = a.to_f / b
  result.round(4)
end

puts safe_divide(10, 3)      # => 3.3333
puts safe_divide(10, 0)      # => Error: ไม่สามารถหารด้วยศูนย์
puts safe_divide("10", 3)    # => Error: ต้องการตัวเลข
puts safe_divide(10, "abc")  # => Error: ต้องการตัวเลข
```

### แบบฝึกหัดที่ 11
Instance Variables และ Getter/Setter

```ruby
# เฉลย
class Temperature
  def initialize(celsius)
    @celsius = celsius.to_f
  end
  
  def celsius
    @celsius
  end
  
  def celsius=(value)
    @celsius = value.to_f
  end
  
  def fahrenheit
    (@celsius * 9.0 / 5.0) + 32
  end
  
  def kelvin
    @celsius + 273.15
  end
  
  def to_s
    "#{@celsius}°C = #{fahrenheit}°F = #{kelvin}K"
  end
end

temp = Temperature.new(100)
puts temp           # => 100.0°C = 212.0°F = 373.15K

temp.celsius = 0
puts temp           # => 0.0°C = 32.0°F = 273.15K

temp.celsius = -40
puts temp           # => -40.0°C = -40.0°F = 233.15K
```

### แบบฝึกหัดที่ 12
Global Variables สำหรับ Application Config

```ruby
# เฉลย
$log_level = :info
$app_env = :development

def log(message, level = :info)
  levels = { debug: 0, info: 1, warn: 2, error: 3 }
  return if levels[level] < levels[$log_level]
  
  timestamp = Time.now.strftime("%H:%M:%S")
  puts "[#{timestamp}] [#{level.to_s.upcase}] #{message}"
end

$log_level = :warn

log("ข้อความ debug", :debug)  # ไม่แสดง
log("ข้อความ info", :info)    # ไม่แสดง
log("ข้อความ warn", :warn)    # => [xx:xx:xx] [WARN] ข้อความ warn
log("ข้อความ error", :error)  # => [xx:xx:xx] [ERROR] ข้อความ error
```

### แบบฝึกหัดที่ 13
Nil Safety Pattern

```ruby
# เฉลย
class UserProfile
  def initialize(data)
    @data = data
  end
  
  def display_name
    @data[:first_name]&.capitalize.to_s +
    " " +
    @data[:last_name]&.capitalize.to_s
  end
  
  def email
    @data[:email] || "ไม่ได้ระบุ email"
  end
  
  def age
    @data[:age]&.to_i || 0
  end
end

user1 = UserProfile.new({ first_name: "john", last_name: "doe", email: "john@example.com", age: "25" })
user2 = UserProfile.new({ first_name: "Jane" })

puts user1.display_name  # => John Doe
puts user1.email         # => john@example.com
puts user1.age           # => 25

puts user2.display_name  # => Jane 
puts user2.email         # => ไม่ได้ระบุ email
puts user2.age           # => 0
```

### แบบฝึกหัดที่ 14
Constants ใน Module

```ruby
# เฉลย
module MathConstants
  PI = Math::PI
  E = Math::E
  GOLDEN_RATIO = 1.6180339887
  
  def self.circle_area(r)
    PI * r ** 2
  end
  
  def self.sphere_volume(r)
    (4.0/3.0) * PI * r ** 3
  end
end

puts MathConstants::PI.round(5)           # => 3.14159
puts MathConstants::GOLDEN_RATIO          # => 1.6180339887
puts MathConstants.circle_area(5).round(2)  # => 78.54
puts MathConstants.sphere_volume(3).round(2) # => 113.1
```

### แบบฝึกหัดที่ 15
สร้าง Complex Data Manipulation

```ruby
# เฉลย
# ระบบคลังสินค้า
class Inventory
  @@items = {}
  
  def self.add_item(name, quantity, price)
    if @@items.key?(name)
      @@items[name][:quantity] += quantity
    else
      @@items[name] = {
        quantity: quantity,
        price: price,
        added_at: Time.now
      }
    end
  end
  
  def self.remove_item(name, quantity)
    return "ไม่พบสินค้า #{name}" unless @@items.key?(name)
    
    if @@items[name][:quantity] < quantity
      return "สินค้าไม่เพียงพอ"
    end
    
    @@items[name][:quantity] -= quantity
    @@items.delete(name) if @@items[name][:quantity].zero?
    "ลบสินค้า #{name} จำนวน #{quantity} ชิ้น"
  end
  
  def self.total_value
    @@items.sum { |_, v| v[:quantity] * v[:price] }
  end
  
  def self.report
    puts "=== รายงานคลังสินค้า ==="
    @@items.each do |name, data|
      puts "#{name}: #{data[:quantity]} ชิ้น @ #{data[:price]} บาท"
    end
    puts "มูลค่ารวม: #{total_value} บาท"
  end
end

Inventory.add_item("แอปเปิ้ล", 100, 50)
Inventory.add_item("กล้วย", 200, 20)
Inventory.add_item("มะม่วง", 50, 80)
Inventory.add_item("แอปเปิ้ล", 50, 50)

Inventory.report

puts Inventory.remove_item("กล้วย", 30)  # => ลบสินค้า กล้วย จำนวน 30 ชิ้น
puts Inventory.remove_item("ทุเรียน", 5)  # => ไม่พบสินค้า ทุเรียน

Inventory.report
```

---

## สรุป

ในตอนที่ 2 นี้ เราได้เรียนรู้:

1. **ประเภทของ Variables** - Local, Instance, Class, Global, และ Constants
2. **Naming Conventions** - snake_case, PascalCase, SCREAMING_SNAKE_CASE
3. **Data Types พื้นฐาน** - Integer, Float, String, Boolean, Nil, Symbol
4. **Type Conversion** - to_i, to_f, to_s, to_a, to_h
5. **Type Checking** - class, is_a?, kind_of?, respond_to?
6. **Frozen Objects** - freeze, frozen?
7. **Multiple Assignment** - Parallel assignment, Splat operator
8. **Swap Variables** - การสลับค่าแบบ Ruby
9. **Constants** - การใช้งานและป้องกันการเปลี่ยนแปลง

ในตอนต่อไปเราจะลงลึกกับ **Strings** ซึ่งเป็นหนึ่งในประเภทข้อมูลที่ใช้งานบ่อยที่สุด

---

*ตอนที่ 2 จบแล้ว - ไปต่อตอนที่ 3: Strings*
