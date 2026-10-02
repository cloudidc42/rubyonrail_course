# ส่วนที่ 2: Variables และ Data Types - ขั้นตอนที่ 11-25

## บทนำ

ตัวแปร (Variables) และชนิดข้อมูล (Data Types) คือรากฐานของการเขียนโปรแกรม ใน Ruby ทุกสิ่งทุกอย่างเป็น Object ไม่ว่าจะเป็นตัวเลข สตริง หรือแม้แต่ค่า nil ก็ล้วนเป็น Object ทั้งสิ้น

---

## ขั้นตอนที่ 11: Variable Naming Conventions

### กฎการตั้งชื่อตัวแปรใน Ruby

Ruby มีกฎการตั้งชื่อตัวแปรที่ชัดเจนและใช้ประเภทตัวแปรแยกกันด้วยรูปแบบของชื่อ

```ruby
# ถูกต้อง - ชื่อตัวแปรที่ดี
user_name = "Alice"          # snake_case (แนะนำ)
first_name = "Bob"
age_in_years = 25
is_active = true
total_price = 99.99

# ผิด - ชื่อตัวแปรที่ไม่ดี
# userName = "Alice"       # camelCase (ไม่แนะนำใน Ruby)
# FirstName = "Bob"        # ขึ้นต้นด้วยตัวพิมพ์ใหญ่ = Constant ใน Ruby
# 1name = "test"           # ขึ้นต้นด้วยตัวเลข (Error!)
# my-name = "test"         # มี hyphen (Error!)
# my name = "test"         # มี space (Error!)

# ชื่อพิเศษที่ Ruby ใช้
# _ สามารถใช้ได้ (convention สำหรับตัวแปรที่ไม่ใช้)
_ = "ไม่สนใจค่านี้"
_unused = "ตัวแปรที่ไม่ได้ใช้"

# ตัวแปรที่ขึ้นต้นและลงท้ายด้วย __ (double underscore)
__method__  # method ปัจจุบัน (built-in)
__LINE__    # บรรทัดปัจจุบัน
__FILE__    # ไฟล์ปัจจุบัน
__dir__     # directory ปัจจุบัน
```

### Ruby Naming Conventions ที่ควรรู้

```ruby
# snake_case สำหรับ:
# - Local variables
user_email = "test@example.com"
# - Method names
def calculate_total_price(items)
  items.sum
end
# - Symbols
:user_name
:first_name

# CamelCase (PascalCase) สำหรับ:
# - Class names
class UserAccount; end
class HttpRequest; end
# - Module names
module DatabaseHelper; end

# SCREAMING_SNAKE_CASE สำหรับ:
# - Constants
MAX_SIZE = 100
API_BASE_URL = "https://api.example.com"
DEFAULT_TIMEOUT = 30

# ตัวอย่างครบถ้วน
class BankAccount  # PascalCase class
  MAX_WITHDRAWAL = 10_000  # SCREAMING_SNAKE for constants

  attr_reader :account_number, :balance  # snake_case for methods

  def initialize(account_number, initial_balance)
    @account_number = account_number  # @ = instance variable
    @balance = initial_balance
    @@total_accounts += 1             # @@ = class variable
  end

  def withdraw_money(amount)          # snake_case method
    @balance -= amount
  end
end
```

---

## ขั้นตอนที่ 12: Local Variables (ตัวแปรท้องถิ่น)

### ลักษณะของ Local Variables

```ruby
# Local variable ขึ้นต้นด้วยตัวพิมพ์เล็กหรือ underscore
name = "Alice"
_private_var = "ส่วนตัว"
counter = 0

# Scope ของ local variable
def show_name
  local_var = "ฉันอยู่ใน method นี้เท่านั้น"
  puts local_var
end

show_name
# puts local_var  # Error! ไม่สามารถเข้าถึงได้นอก method

# Local variable ใน block
[1, 2, 3].each do |number|
  block_var = number * 2
  puts block_var
end
# puts block_var  # Error! ออกจาก block แล้วไม่มีแล้ว

# ยกเว้น: ตัวแปรที่ประกาศก่อน block สามารถเข้าถึงได้ใน block
total = 0
[1, 2, 3, 4, 5].each do |n|
  total += n  # ใช้ total ที่ประกาศข้างนอกได้
end
puts total  # => 15

# การกำหนดค่าหลายตัวพร้อมกัน
a, b, c = 1, 2, 3
puts "#{a}, #{b}, #{c}"  # => 1, 2, 3

# Multiple assignment จาก Array
first, second, *rest = [1, 2, 3, 4, 5]
puts first   # => 1
puts second  # => 2
puts rest    # => [3, 4, 5]
```

### การตรวจสอบตัวแปรที่กำหนดแล้ว

```ruby
# defined? บอกว่าตัวแปรมีอยู่หรือไม่
x = 10
puts defined?(x)       # => "local-variable"
puts defined?(y)       # => nil (ยังไม่กำหนด)
puts defined?(puts)    # => "method"
puts defined?(String)  # => "constant"

# local_variables แสดง local variables ทั้งหมด
a = 1
b = 2
c = 3
puts local_variables.inspect  # => [:a, :b, :c]
```

---

## ขั้นตอนที่ 13: Instance Variables (ตัวแปร Instance)

### ตัวแปรที่เป็นของ Object แต่ละชิ้น

```ruby
class Person
  def initialize(name, age)
    @name = name  # @ นำหน้า = instance variable
    @age = age
  end

  def introduce
    puts "สวัสดี ฉันชื่อ #{@name} อายุ #{@age} ปี"
  end

  def birthday
    @age += 1
    puts "สุขสันต์วันเกิด #{@name}! อายุ #{@age} ปีแล้ว"
  end

  # Getter method
  def name
    @name
  end

  # Setter method
  def name=(new_name)
    @name = new_name
  end
end

alice = Person.new("Alice", 25)
bob = Person.new("Bob", 30)

alice.introduce  # => สวัสดี ฉันชื่อ Alice อายุ 25 ปี
bob.introduce    # => สวัสดี ฉันชื่อ Bob อายุ 30 ปี

alice.birthday   # => สุขสันต์วันเกิด Alice! อายุ 26 ปีแล้ว
bob.introduce    # => สวัสดี ฉันชื่อ Bob อายุ 30 ปี (ไม่เปลี่ยน)

# ใช้ attr_accessor สร้าง getter/setter อัตโนมัติ
class Car
  attr_accessor :brand, :model, :year  # สร้าง getter + setter
  attr_reader :vin                      # getter only
  attr_writer :color                    # setter only

  def initialize(brand, model, year, vin)
    @brand = brand
    @model = model
    @year = year
    @vin = vin
    @color = "white"
  end

  def info
    "#{@year} #{@brand} #{@model}"
  end
end

car = Car.new("Toyota", "Camry", 2023, "VIN123456")
puts car.info           # => 2023 Toyota Camry
puts car.brand          # => Toyota

car.brand = "Honda"
puts car.brand          # => Honda

puts car.vin            # => VIN123456
# car.vin = "NEW_VIN"  # Error! ไม่มี setter สำหรับ vin

# Instance variable ที่ไม่ได้กำหนด = nil
class Example
  def check_variable
    puts @undefined_var.inspect  # => nil (ไม่ error!)
    puts @undefined_var.nil?     # => true
  end
end

Example.new.check_variable
```

---

## ขั้นตอนที่ 14: Class Variables (ตัวแปร Class)

### ตัวแปรที่แชร์กันทุก instance

```ruby
class BankAccount
  @@total_accounts = 0     # @@ นำหน้า = class variable
  @@total_deposits = 0

  def initialize(owner, balance = 0)
    @owner = owner
    @balance = balance
    @@total_accounts += 1
    puts "สร้างบัญชีสำหรับ #{@owner} แล้ว"
  end

  def deposit(amount)
    @balance += amount
    @@total_deposits += amount
    puts "ฝากเงิน #{amount} บาท (ยอดรวม: #{@balance} บาท)"
  end

  def self.total_accounts
    @@total_accounts
  end

  def self.total_deposits
    @@total_deposits
  end

  def balance
    @balance
  end
end

account1 = BankAccount.new("Alice", 1000)
account2 = BankAccount.new("Bob", 500)

account1.deposit(500)
account2.deposit(1000)

puts "จำนวนบัญชีทั้งหมด: #{BankAccount.total_accounts}"  # => 2
puts "ยอดฝากทั้งหมด: #{BankAccount.total_deposits}"      # => 1500

# ปัญหาของ Class Variable กับ Inheritance
class Animal
  @@count = 0

  def initialize
    @@count += 1
  end

  def self.count
    @@count
  end
end

class Dog < Animal; end
class Cat < Animal; end

Dog.new
Dog.new
Cat.new

puts Animal.count  # => 3 (แชร์กันทุก subclass!)
puts Dog.count     # => 3 (ไม่ใช่ count เฉพาะ Dog)
puts Cat.count     # => 3

# ใช้ instance variable ของ class แทนจะดีกว่า
class Vehicle
  @count = 0  # นี่คือ instance variable ของ class object

  class << self
    attr_accessor :count
  end

  def initialize
    self.class.count += 1
  end
end

class Truck < Vehicle
  @count = 0
end

class Bus < Vehicle
  @count = 0
end

Truck.new
Truck.new
Bus.new

puts Truck.count   # => 2
puts Bus.count     # => 1
```

---

## ขั้นตอนที่ 15: Global Variables (ตัวแปร Global)

### ตัวแปรที่เข้าถึงได้จากทุกที่

```ruby
# $ นำหน้า = global variable
$app_name = "My Ruby App"
$debug_mode = false
$version = "1.0.0"

def show_app_info
  # เข้าถึง global variable ได้จากทุกที่
  puts "App: #{$app_name} v#{$version}"
  puts "Debug mode: #{$debug_mode}"
end

show_app_info

# Global variables ที่ Ruby กำหนดมาให้
puts $0       # ชื่อไฟล์ที่กำลังรัน
puts $$       # Process ID ปัจจุบัน
puts $:.first # ตำแหน่งแรกใน LOAD_PATH
puts $PROGRAM_NAME  # เหมือน $0

# Global variables ที่เกี่ยวกับ I/O
# $stdin   - Standard Input
# $stdout  - Standard Output
# $stderr  - Standard Error

$stdout.puts "ออกทาง stdout"
$stderr.puts "ออกทาง stderr (error)"

# Global variables เกี่ยวกับ Regular Expression
"hello world" =~ /(\w+)\s(\w+)/
puts $~.inspect   # MatchData ทั้งหมด
puts $1           # => "hello" (group 1)
puts $2           # => "world" (group 2)
puts $&           # => "hello world" (match ทั้งหมด)
puts $`           # String ก่อน match
puts $'           # String หลัง match

# คำเตือน: ควรหลีกเลี่ยง Global Variables ในโปรแกรมจริง
# เพราะทำให้โค้ดยากต่อการ debug และ test
# ใช้ class variables, configuration objects, หรือ dependency injection แทน
```

---

## ขั้นตอนที่ 16: Constants (ค่าคงที่)

### การใช้งาน Constants

```ruby
# Constant ขึ้นต้นด้วยตัวพิมพ์ใหญ่
MAX_RETRY = 3
PI = 3.14159265358979
APP_VERSION = "2.0.0"
DATABASE_URL = "postgresql://localhost/mydb"

# Ruby จะเตือนเมื่อแก้ค่า Constant (แต่ยังทำได้)
MAX_RETRY = 5  # Warning: already initialized constant MAX_RETRY

# Constants ใน Class/Module (แนะนำ)
class Configuration
  MAX_CONNECTIONS = 10
  DEFAULT_TIMEOUT = 30
  SUPPORTED_FORMATS = [:json, :xml, :csv].freeze

  def self.max_connections
    MAX_CONNECTIONS
  end
end

puts Configuration::MAX_CONNECTIONS     # => 10
puts Configuration::DEFAULT_TIMEOUT    # => 30
puts Configuration::SUPPORTED_FORMATS  # => [:json, :xml, :csv]

# Module Constants
module HttpStatus
  OK          = 200
  CREATED     = 201
  NO_CONTENT  = 204
  BAD_REQUEST = 400
  NOT_FOUND   = 404
  SERVER_ERROR = 500

  ALL = {
    OK          => "OK",
    CREATED     => "Created",
    NO_CONTENT  => "No Content",
    BAD_REQUEST => "Bad Request",
    NOT_FOUND   => "Not Found",
    SERVER_ERROR => "Internal Server Error"
  }.freeze

  def self.message_for(code)
    ALL[code] || "Unknown Status"
  end
end

puts HttpStatus::NOT_FOUND            # => 404
puts HttpStatus.message_for(200)      # => "OK"
puts HttpStatus.message_for(999)      # => "Unknown Status"

# Freeze สำหรับ Immutable Constants
COLORS = ["red", "green", "blue"].freeze
# COLORS << "purple"  # FrozenError!

# ดู Constants ทั้งหมดใน Module/Class
puts Configuration.constants.inspect
# => [:MAX_CONNECTIONS, :DEFAULT_TIMEOUT, :SUPPORTED_FORMATS]
```

---

## ขั้นตอนที่ 17: Integer - จำนวนเต็ม

```ruby
# การสร้าง Integer
age = 25
population = 7_900_000_000  # underscore ช่วยอ่านง่าย
negative = -42

# Arithmetic operations
puts 10 + 3    # => 13
puts 10 - 3    # => 7
puts 10 * 3    # => 30
puts 10 / 3    # => 3 (Integer division!)
puts 10 % 3    # => 1 (modulo)
puts 10 ** 3   # => 1000 (power)

# Integer division vs Float division
puts 10 / 3      # => 3   (Integer / Integer = Integer)
puts 10.0 / 3    # => 3.3333... (Float / Integer = Float)
puts 10 / 3.0    # => 3.3333...
puts 10.to_f / 3 # => 3.3333...

# Integer methods
n = 42
puts n.even?      # => true
puts n.odd?       # => false
puts n.zero?      # => false
puts n.positive?  # => true
puts n.negative?  # => false
puts n.abs        # => 42
puts (-42).abs    # => 42

# Iteration methods
5.times { |i| print "#{i} " }
# 0 1 2 3 4

1.upto(5)   { |i| print "#{i} " }  # 1 2 3 4 5
5.downto(1) { |i| print "#{i} " }  # 5 4 3 2 1

1.step(10, 2) { |i| print "#{i} " }  # 1 3 5 7 9

# Conversion
puts 42.to_f    # => 42.0
puts 42.to_s    # => "42"
puts 42.to_r    # => 42/1 (Rational)
puts 42.to_c    # => (42+0i) (Complex)

# Digit methods
puts 42.digits      # => [2, 4] (หลักจากขวาไปซ้าย)
puts 42.digits.reverse  # => [4, 2]
puts 1234.digits    # => [4, 3, 2, 1]

# Base conversion
puts 255.to_s(2)   # => "11111111" (binary)
puts 255.to_s(8)   # => "377" (octal)
puts 255.to_s(16)  # => "ff" (hex)

# Integer limits (Ruby รองรับ BigInteger!)
puts (2 ** 100)  # => 1267650600228229401496703205376 (ใหญ่แค่ไหนก็ได้!)
```

---

## ขั้นตอนที่ 18: Float - จำนวนทศนิยม

```ruby
# การสร้าง Float
price = 99.99
pi = 3.14159
negative_float = -0.5
scientific = 1.5e3  # 1500.0
small = 1.5e-3      # 0.0015

# Float methods
puts 3.7.ceil    # => 4 (ปัดขึ้น)
puts 3.2.floor   # => 3 (ปัดลง)
puts 3.5.round   # => 4 (ปัดทั่วไป)
puts 3.7.round   # => 4
puts 3.14159.round(2)  # => 3.14

puts 3.7.truncate  # => 3 (ตัดทศนิยมทิ้ง)

# Float constants
puts Float::INFINITY    # => Infinity
puts -Float::INFINITY   # => -Infinity
puts Float::NAN         # => NaN

# ตรวจสอบค่าพิเศษ
puts (1.0 / 0).infinite?   # => 1
puts (-1.0 / 0).infinite?  # => -1
puts (0.0 / 0).nan?        # => true

# Float precision problem!
puts 0.1 + 0.2             # => 0.30000000000000004 (ไม่ใช่ 0.3!)
puts (0.1 + 0.2) == 0.3    # => false!

# วิธีแก้: ใช้ round หรือ BigDecimal
puts (0.1 + 0.2).round(1) == 0.3  # => true

# BigDecimal สำหรับการเงิน
require 'bigdecimal'

price1 = BigDecimal("0.1")
price2 = BigDecimal("0.2")
total = price1 + price2
puts total          # => 0.3e0
puts total.to_s     # => "0.3e0"
puts total.to_f     # => 0.3
```

---

## ขั้นตอนที่ 19: String - ข้อความ

```ruby
# การสร้าง String
name = "Alice"         # Double quotes
greeting = 'Hello'     # Single quotes

# ความแตกต่างระหว่าง ' และ "
puts "Line 1\nLine 2"   # \n = newline (double quote ทำได้)
puts 'Line 1\nLine 2'   # \n = literal (single quote ไม่แปล escape)

x = 10
puts "Value: #{x}"      # interpolation (double quote ทำได้)
puts 'Value: #{x}'      # ไม่ทำ interpolation (แสดงตรงๆ)

# String methods พื้นฐาน
str = "Hello, Ruby World!"

puts str.length         # => 18
puts str.size           # => 18 (เหมือนกัน)
puts str.upcase         # => HELLO, RUBY WORLD!
puts str.downcase       # => hello, ruby world!
puts str.capitalize     # => Hello, ruby world!
puts str.reverse        # => !dlroW ybuR ,olleH
puts str.strip          # ลบ whitespace หัวท้าย
puts "  hello  ".strip  # => "hello"
puts str.chomp          # ลบ newline ท้าย

# String inspection
puts str.include?("Ruby")  # => true
puts str.start_with?("Hello")  # => true
puts str.end_with?("!")    # => true
puts str.empty?            # => false
puts "".empty?             # => true

# String encoding
puts str.encoding      # => UTF-8
puts "สวัสดี".length   # => 6 (characters)
puts "สวัสดี".bytesize # => 18 (bytes in UTF-8)

# Type conversion
puts "42".to_i    # => 42
puts "3.14".to_f  # => 3.14
puts "42abc".to_i # => 42 (หยุดที่ตัวแรกที่ไม่ใช่ตัวเลข)
puts "abc".to_i   # => 0
puts 42.to_s      # => "42"
```

---

## ขั้นตอนที่ 20: Boolean, Nil, และ Symbol

### Boolean

```ruby
# Boolean values
is_active = true
is_deleted = false

# Truthy และ Falsy ใน Ruby
# FALSY: false, nil (แค่สองค่านี้เท่านั้น!)
# TRUTHY: ทุกอย่างอื่น รวมถึง 0, "", []

puts "0 is truthy: #{0 ? 'yes' : 'no'}"     # => yes
puts "'' is truthy: #{'' ? 'yes' : 'no'}"   # => yes
puts "[] is truthy: #{[] ? 'yes' : 'no'}"   # => yes
puts "false is falsy: #{false ? 'yes' : 'no'}"  # => no
puts "nil is falsy: #{nil ? 'yes' : 'no'}"      # => no

# Boolean methods
puts true.class   # => TrueClass
puts false.class  # => FalseClass

puts true & false   # => false (AND)
puts true | false   # => true  (OR)
puts !true          # => false (NOT)
puts true ^ false   # => true  (XOR)
```

### Nil

```ruby
# nil คือ absence of value
nothing = nil
puts nothing.class    # => NilClass
puts nothing.nil?     # => true
puts nothing.inspect  # => "nil"
puts nothing.to_i     # => 0
puts nothing.to_s     # => ""
puts nothing.to_a     # => []

# nil check patterns
user = nil

# แบบที่ 1: ตรวจสอบตรงๆ
if user.nil?
  puts "ไม่มี user"
end

# แบบที่ 2: Conditional assignment
user ||= "Guest"  # กำหนดค่าถ้าเป็น nil หรือ false
puts user  # => "Guest"

# Safe navigation operator (&.)
puts user&.upcase    # => "GUEST" (ไม่ error ถ้า nil)
puts nil&.upcase     # => nil (ไม่ error!)
# nil.upcase         # => NoMethodError!
```

### Symbol

```ruby
# Symbol คือ immutable string-like objects
status = :active
puts status         # => active
puts status.class   # => Symbol
puts status.to_s    # => "active"
puts "active".to_sym == :active  # => true

# Symbol ใช้ memory น้อยกว่า String
# String ทุกตัวเป็น object ใหม่
puts "hello".object_id == "hello".object_id  # => false (ต่างกัน!)
# Symbol ใช้ object เดิม
puts :hello.object_id == :hello.object_id    # => true (เหมือนกัน!)

# Symbol ใช้บ่อยใน Hash keys
person = {
  name: "Alice",     # :name เป็น key (syntax ใหม่)
  age: 25,
  role: :admin
}

puts person[:name]   # => "Alice"
puts person[:role]   # => admin

# Symbol methods
sym = :hello_world
puts sym.upcase    # => :HELLO_WORLD
puts sym.length    # => 11
puts sym.to_s      # => "hello_world"

# Symbols เป็น immutable
# :hello << "!"  # NoMethodError!
```

---

## ขั้นตอนที่ 21: Type Conversion Methods

### การแปลงชนิดข้อมูล

```ruby
# to_i - แปลงเป็น Integer
puts "42".to_i       # => 42
puts "42.7".to_i     # => 42 (ตัดทศนิยม)
puts "42abc".to_i    # => 42
puts "abc".to_i      # => 0
puts nil.to_i        # => 0
puts true.to_i       # Error! (TrueClass ไม่มี to_i)
puts 3.7.to_i        # => 3

# Integer() - strict conversion
puts Integer("42")   # => 42
# Integer("42abc")   # ArgumentError!
# Integer(nil)       # TypeError!

# to_f - แปลงเป็น Float
puts "3.14".to_f     # => 3.14
puts "3.14abc".to_f  # => 3.14
puts "abc".to_f      # => 0.0
puts 42.to_f         # => 42.0
puts nil.to_f        # => 0.0

# to_s - แปลงเป็น String
puts 42.to_s         # => "42"
puts 3.14.to_s       # => "3.14"
puts true.to_s       # => "true"
puts false.to_s      # => "false"
puts nil.to_s        # => "" (empty string)
puts :symbol.to_s    # => "symbol"
puts [1,2,3].to_s    # => "[1, 2, 3]"

# to_a - แปลงเป็น Array
puts nil.to_a.inspect        # => []
puts (1..5).to_a.inspect     # => [1, 2, 3, 4, 5]
puts {a: 1, b: 2}.to_a.inspect  # => [[:a, 1], [:b, 2]]

# to_sym - แปลงเป็น Symbol
puts "hello".to_sym    # => :hello
puts "my method".to_sym  # => :"my method"

# to_r - แปลงเป็น Rational
puts 0.5.to_r          # => 1/2
puts "1/3".to_r        # => 1/3

# Implicit vs Explicit conversion
class Temperature
  def initialize(celsius)
    @celsius = celsius
  end

  # Explicit conversion (to_s)
  def to_s
    "#{@celsius}°C"
  end

  # Implicit conversion (to_str - Ruby ใช้อัตโนมัติ)
  def to_str
    "#{@celsius}°C"
  end

  # Explicit to number
  def to_f
    @celsius.to_f
  end

  def to_i
    @celsius.to_i
  end
end

temp = Temperature.new(25)
puts temp           # => 25°C (ใช้ to_s)
puts "Temp: " + temp  # => "Temp: 25°C" (ใช้ to_str)
puts temp.to_f      # => 25.0
```

---

## ขั้นตอนที่ 22: Checking Variable Types

### การตรวจสอบชนิดข้อมูล

```ruby
# class method
puts 42.class           # => Integer
puts 3.14.class         # => Float
puts "hello".class      # => String
puts true.class         # => TrueClass
puts false.class        # => FalseClass
puts nil.class          # => NilClass
puts :symbol.class      # => Symbol
puts [1,2,3].class      # => Array
puts({a: 1}.class)      # => Hash
puts (1..5).class       # => Range

# is_a? / kind_of? - ตรวจสอบว่าเป็น class นั้นหรือ subclass
puts 42.is_a?(Integer)   # => true
puts 42.is_a?(Numeric)   # => true (Integer เป็น subclass ของ Numeric)
puts 42.is_a?(Float)     # => false
puts 42.is_a?(Object)    # => true (ทุกอย่างเป็น Object)

puts 42.kind_of?(Integer)  # => true (เหมือน is_a?)

# instance_of? - ตรวจสอบว่าเป็น class นั้นเท่านั้น (ไม่รวม subclass)
puts 42.instance_of?(Integer)  # => true
puts 42.instance_of?(Numeric)  # => false! (Integer ไม่ใช่ Numeric โดยตรง)

# respond_to? - ตรวจสอบว่า object มี method นั้นหรือไม่
puts "hello".respond_to?(:upcase)   # => true
puts "hello".respond_to?(:nonexistent)  # => false
puts 42.respond_to?(:times)         # => true
puts 42.respond_to?(:upcase)        # => false

# Duck typing - ตรวจสอบว่าทำ operation ได้ไหม
def process(obj)
  if obj.respond_to?(:each)
    obj.each { |item| puts item }
  elsif obj.respond_to?(:to_s)
    puts obj.to_s
  end
end

process([1, 2, 3])   # รัน each
process("hello")     # รัน to_s

# Comparable types
puts Integer.ancestors.inspect
# => [Integer, Numeric, Comparable, Object, Kernel, BasicObject]

# ตรวจสอบ nil
puts nil.nil?     # => true
puts 0.nil?       # => false
puts false.nil?   # => false
puts "".nil?      # => false

# ตรวจสอบ blank/present? (Rails methods)
# puts nil.blank?    # => true (เฉพาะ Rails)
# puts "".blank?     # => true (เฉพาะ Rails)
# puts [].blank?     # => true (เฉพาะ Rails)
# puts 0.blank?      # => false (เฉพาะ Rails)
```

---

## ขั้นตอนที่ 23: Frozen Objects

### Object ที่ไม่สามารถเปลี่ยนค่าได้

```ruby
# freeze ทำให้ object ไม่สามารถแก้ไขได้
str = "hello"
str.freeze

puts str.frozen?  # => true
# str << "!"      # FrozenError!
# str.upcase!     # FrozenError!

str_copy = str.dup  # dup สร้าง copy ที่ไม่ frozen
str_copy << "!"
puts str_copy  # => "hello!"

# Symbol และ Integer ถูก freeze อยู่แล้ว
puts :hello.frozen?  # => true
puts 42.frozen?      # => true

# Frozen Array
arr = [1, 2, 3].freeze
# arr << 4        # FrozenError!
# arr.push(4)     # FrozenError!
arr_copy = arr.dup  # copy ที่ไม่ frozen

# Frozen Hash
config = {
  host: "localhost",
  port: 5432
}.freeze
# config[:host] = "remote"  # FrozenError!

# String Literals - frozen_string_literal magic comment
# # frozen_string_literal: true
# ใส่ที่บรรทัดแรกของไฟล์ทำให้ string ทุกตัวถูก freeze

# ตัวอย่าง immutable value object
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
    freeze  # freeze ตัวเอง
  end

  def +(other)
    Point.new(@x + other.x, @y + other.y)
  end

  def to_s
    "(#{@x}, #{@y})"
  end
end

p1 = Point.new(1, 2)
p2 = Point.new(3, 4)
p3 = p1 + p2

puts p1  # => (1, 2)
puts p2  # => (3, 4)
puts p3  # => (4, 6)
puts p1.frozen?  # => true
```

---

## ขั้นตอนที่ 24: Multiple Assignment และ Swap

### Multiple Assignment

```ruby
# กำหนดหลายตัวแปรพร้อมกัน
a, b, c = 1, 2, 3
puts "#{a}, #{b}, #{c}"  # => 1, 2, 3

# Splat operator (*)
first, *rest = [1, 2, 3, 4, 5]
puts first        # => 1
puts rest.inspect # => [2, 3, 4, 5]

*beginning, last = [1, 2, 3, 4, 5]
puts beginning.inspect  # => [1, 2, 3, 4]
puts last               # => 5

head, *middle, tail = [1, 2, 3, 4, 5]
puts head           # => 1
puts middle.inspect # => [2, 3, 4]
puts tail           # => 5

# Nested assignment
a, (b, c) = 1, [2, 3]
puts "#{a}, #{b}, #{c}"  # => 1, 2, 3

# Swap variables
x = 10
y = 20
x, y = y, x  # Swap ด้วย Ruby idiom!
puts "x=#{x}, y=#{y}"  # => x=20, y=10

# แบบเดิม (ต้องใช้ temp variable)
# temp = x
# x = y
# y = temp

# Swap หลายค่า
a, b, c = 1, 2, 3
a, b, c = c, a, b  # rotate!
puts "#{a}, #{b}, #{c}"  # => 3, 1, 2

# จาก Method return หลายค่า
def min_max(arr)
  [arr.min, arr.max]
end

min, max = min_max([3, 1, 4, 1, 5, 9, 2, 6])
puts "min=#{min}, max=#{max}"  # => min=1, max=9

# Ignore บางค่า ด้วย _
_, second, _ = [1, 2, 3]
puts second  # => 2

first, _, _, fourth = [10, 20, 30, 40]
puts "#{first}, #{fourth}"  # => 10, 40
```

---

## ขั้นตอนที่ 25: String Interpolation และแบบฝึกหัด

### String Interpolation ขั้นสูง

```ruby
name = "Alice"
age = 25
score = 98.765

# Basic interpolation
puts "ชื่อ: #{name}"
puts "อายุ: #{age} ปี"
puts "คะแนน: #{score}"

# Expression ใน interpolation
puts "ปีเกิด: #{2024 - age}"
puts "คะแนนเต็ม: #{score.round(1)}%"

# Method call ใน interpolation
puts "ชื่อใหญ่: #{name.upcase}"
puts "ชื่อกลับ: #{name.reverse}"

# Conditional ใน interpolation
grade = score >= 90 ? "A" : "B"
puts "เกรด: #{grade}"

# Complex expression
items = [1, 2, 3, 4, 5]
puts "รวม: #{items.sum}, เฉลี่ย: #{items.sum.to_f / items.length}"

# Multi-line string
message = "
  สวัสดี #{name}!
  คุณอายุ #{age} ปี
  คะแนน: #{score.round(2)}%
  เกรด: #{grade}
".strip

puts message

# Heredoc
report = <<~HEREDOC
  === รายงานผล ===
  นักเรียน: #{name}
  อายุ: #{age}
  คะแนน: #{score.round(2)}
  เกรด: #{grade}
  วันที่: #{Time.now.strftime("%d/%m/%Y")}
HEREDOC

puts report

# Format String
printf("%-10s %5d %8.2f\n", name, age, score)
# Alice         25    98.77

formatted = format("ชื่อ: %-10s | คะแนน: %6.2f%%", name, score)
puts formatted

# % operator (sprintf shorthand)
puts "Hello, %s! You are %d years old." % [name, age]
```

---

## แบบฝึกหัด 15 ข้อ

### แบบฝึกหัดที่ 1: ตัวแปรและการแสดงผล

```ruby
# เขียนโปรแกรมเก็บข้อมูลนักเรียนและแสดงผล
student_name = "สมชาย"
student_id = "STD001"
grade_level = 10
gpa = 3.75
is_honor_roll = gpa >= 3.5

puts "=== ข้อมูลนักเรียน ==="
puts "ชื่อ: #{student_name}"
puts "รหัส: #{student_id}"
puts "ชั้น: #{grade_level}"
puts "GPA: #{gpa}"
puts "เกียรตินิยม: #{is_honor_roll ? 'ใช่' : 'ไม่ใช่'}"
```

### แบบฝึกหัดที่ 2: Type Checking

```ruby
# เขียน method ที่ตรวจสอบชนิดและแสดงข้อมูล
def describe(value)
  puts "ค่า: #{value.inspect}"
  puts "ชนิด: #{value.class}"
  puts "Frozen: #{value.frozen?}"

  case value
  when Integer then puts "ลักษณะ: จำนวนเต็ม #{value.even? ? 'คู่' : 'คี่'}"
  when Float   then puts "ลักษณะ: ทศนิยม #{value > 0 ? 'บวก' : 'ลบ'}"
  when String  then puts "ลักษณะ: ข้อความ ยาว #{value.length} ตัวอักษร"
  when Array   then puts "ลักษณะ: Array มี #{value.length} สมาชิก"
  when NilClass then puts "ลักษณะ: ไม่มีค่า"
  when TrueClass, FalseClass then puts "ลักษณะ: Boolean"
  end
  puts "-" * 30
end

describe(42)
describe(3.14)
describe("Hello")
describe([1, 2, 3])
describe(nil)
describe(true)
```

### แบบฝึกหัดที่ 3: Constants ในโปรแกรม

```ruby
module ShippingRates
  STANDARD_RATE = 50
  EXPRESS_RATE = 150
  OVERNIGHT_RATE = 300
  FREE_SHIPPING_MINIMUM = 500

  def self.calculate(total, method)
    return 0 if total >= FREE_SHIPPING_MINIMUM

    case method
    when :standard  then STANDARD_RATE
    when :express   then EXPRESS_RATE
    when :overnight then OVERNIGHT_RATE
    else raise "Unknown shipping method: #{method}"
    end
  end
end

orders = [
  { items: 200, method: :standard },
  { items: 600, method: :express },
  { items: 150, method: :overnight }
]

orders.each do |order|
  shipping = ShippingRates.calculate(order[:items], order[:method])
  total = order[:items] + shipping
  puts "สินค้า: #{order[:items]} บาท | ค่าส่ง: #{shipping} บาท | รวม: #{total} บาท"
end
```

### แบบฝึกหัดที่ 4: Multiple Assignment

```ruby
# ใช้ multiple assignment แก้ปัญหา
def stats(numbers)
  sorted = numbers.sort
  [sorted.first, sorted.last, numbers.sum, numbers.sum.to_f / numbers.size]
end

data = [23, 45, 12, 67, 34, 89, 11, 56]
min, max, total, average = stats(data)

puts "ข้อมูล: #{data.join(', ')}"
puts "ต่ำสุด: #{min}"
puts "สูงสุด: #{max}"
puts "ผลรวม: #{total}"
puts "ค่าเฉลี่ย: #{average.round(2)}"
```

### แบบฝึกหัดที่ 5: Global vs Local Scope

```ruby
$application_name = "Ruby Course App"
$request_count = 0

def handle_request(path)
  $request_count += 1
  local_response = "Response for #{path}"

  puts "Request ##{$request_count}: #{path}"
  puts "App: #{$application_name}"

  local_response  # return
end

["/" , "/about", "/contact"].each do |path|
  response = handle_request(path)
  puts "Got: #{response}"
  puts "-" * 40
end

puts "Total requests: #{$request_count}"
```

### แบบฝึกหัดที่ 6: Symbol vs String

```ruby
# เปรียบเทียบ performance และการใช้งาน
require 'benchmark'

n = 1_000_000

time_string = Benchmark.realtime do
  n.times { "hello" == "hello" }
end

time_symbol = Benchmark.realtime do
  n.times { :hello == :hello }
end

puts "String comparison: #{time_string.round(4)} seconds"
puts "Symbol comparison: #{time_symbol.round(4)} seconds"
puts "Symbol เร็วกว่า #{(time_string / time_symbol).round(2)}x"

# การใช้งาน Symbol เป็น Hash key
config = {
  database: "mydb",
  host: "localhost",
  port: 5432,
  pool: 5
}

puts "\nConfiguration:"
config.each do |key, value|
  puts "  #{key}: #{value}"
end
```

### แบบฝึกหัดที่ 7: Type Conversion Chain

```ruby
# แปลงชนิดข้อมูลในหลายขั้นตอน
user_input = "  42.5  "

# แปลงทีละขั้น
step1 = user_input.strip      # ลบ whitespace
step2 = step1.to_f            # แปลงเป็น Float
step3 = step2.round           # ปัดเป็น Integer
step4 = step3.to_s            # กลับเป็น String
step5 = step4 + " items"      # ต่อ String

puts "ต้นฉบับ: #{user_input.inspect}"
puts "หลัง strip: #{step1.inspect}"
puts "หลัง to_f: #{step2}"
puts "หลัง round: #{step3}"
puts "หลัง to_s: #{step4.inspect}"
puts "ผลลัพธ์: #{step5}"

# Method chain
result = "  42.5  ".strip.to_f.round.to_s + " items"
puts "Method chain: #{result}"
```

### แบบฝึกหัดที่ 8: Immutable Configuration

```ruby
# สร้าง configuration ที่ไม่สามารถเปลี่ยนแปลงได้
module AppConfig
  DATABASE = {
    host: "localhost",
    port: 5432,
    name: "production_db",
    pool: 5
  }.freeze

  CACHE = {
    provider: :redis,
    host: "localhost",
    port: 6379,
    ttl: 3600
  }.freeze

  ALLOWED_ORIGINS = [
    "https://example.com",
    "https://api.example.com"
  ].freeze

  def self.database_url
    "postgresql://#{DATABASE[:host]}:#{DATABASE[:port]}/#{DATABASE[:name]}"
  end
end

puts "Database URL: #{AppConfig.database_url}"
puts "Cache TTL: #{AppConfig::CACHE[:ttl]} seconds"
puts "Allowed origins: #{AppConfig::ALLOWED_ORIGINS.join(', ')}"

begin
  AppConfig::DATABASE[:host] = "remote"
rescue => e
  puts "Error: #{e.class}: #{e.message}"
end
```

### แบบฝึกหัดที่ 9: Nil Safety

```ruby
# เขียนโค้ดที่ปลอดภัยจาก nil
class UserProfile
  attr_reader :name, :email, :address

  def initialize(name, email, address = nil)
    @name = name
    @email = email
    @address = address
  end

  def city
    @address&.[](:city)  # safe navigation
  end

  def display_location
    city || "ไม่ระบุเมือง"
  end

  def full_info
    [
      "ชื่อ: #{@name}",
      "Email: #{@email}",
      "เมือง: #{display_location}"
    ].join("\n")
  end
end

user1 = UserProfile.new("Alice", "alice@example.com", { city: "Bangkok" })
user2 = UserProfile.new("Bob", "bob@example.com")

[user1, user2].each do |user|
  puts user.full_info
  puts "-" * 30
end
```

### แบบฝึกหัดที่ 10: Class Variable Counter

```ruby
class Product
  @@total_products = 0
  @@total_value = 0

  attr_reader :name, :price, :quantity

  def initialize(name, price, quantity)
    @name = name
    @price = price
    @quantity = quantity
    @@total_products += 1
    @@total_value += price * quantity
  end

  def total_value
    @price * @quantity
  end

  def self.total_products
    @@total_products
  end

  def self.total_value
    @@total_value
  end

  def self.summary
    puts "=== สรุปสินค้า ==="
    puts "จำนวนรายการ: #{@@total_products}"
    puts "มูลค่ารวม: #{@@total_value.to_s.gsub(/\B(?=(\d{3})+(?!\d))/, ',')} บาท"
  end
end

products = [
  Product.new("Laptop", 35000, 10),
  Product.new("Mouse", 500, 50),
  Product.new("Keyboard", 1500, 30),
  Product.new("Monitor", 8000, 15)
]

products.each do |p|
  puts "#{p.name}: #{p.quantity} ชิ้น × #{p.price} = #{p.total_value.to_s.gsub(/\B(?=(\d{3})+(?!\d))/, ',')} บาท"
end

Product.summary
```

### แบบฝึกหัดที่ 11: Swap Array Elements

```ruby
def sort_without_sort(arr)
  n = arr.length
  n.times do |i|
    (n - i - 1).times do |j|
      arr[j], arr[j+1] = arr[j+1], arr[j] if arr[j] > arr[j+1]
    end
  end
  arr
end

numbers = [64, 34, 25, 12, 22, 11, 90]
puts "ก่อน: #{numbers.inspect}"
sorted = sort_without_sort(numbers.dup)
puts "หลัง: #{sorted.inspect}"
```

### แบบฝึกหัดที่ 12: Instance Variable ใน Class

```ruby
class ShoppingCart
  def initialize
    @items = []
    @discount = 0
  end

  def add_item(name, price, quantity = 1)
    @items << { name: name, price: price, quantity: quantity }
    puts "เพิ่ม #{name} (#{quantity} ชิ้น) แล้ว"
  end

  def apply_discount(percent)
    @discount = percent
    puts "ใช้ส่วนลด #{percent}%"
  end

  def subtotal
    @items.sum { |item| item[:price] * item[:quantity] }
  end

  def discount_amount
    subtotal * @discount / 100.0
  end

  def total
    subtotal - discount_amount
  end

  def receipt
    puts "\n=== ใบเสร็จ ==="
    @items.each do |item|
      printf("%-20s %3d × %8.2f = %10.2f\n",
             item[:name], item[:quantity], item[:price],
             item[:price] * item[:quantity])
    end
    puts "-" * 50
    printf("%-30s %16.2f\n", "ราคารวม:", subtotal)
    printf("%-30s %16.2f\n", "ส่วนลด #{@discount}%:", -discount_amount) if @discount > 0
    printf("%-30s %16.2f\n", "ยอดสุทธิ:", total)
  end
end

cart = ShoppingCart.new
cart.add_item("แล็ปท็อป", 35000)
cart.add_item("เมาส์", 500, 2)
cart.add_item("คีย์บอร์ด", 1500)
cart.apply_discount(10)
cart.receipt
```

### แบบฝึกหัดที่ 13: String Interpolation สร้าง Report

```ruby
class SalesReport
  def initialize(month, year)
    @month = month
    @year = year
    @sales = []
  end

  def add_sale(product, amount, units)
    @sales << { product: product, amount: amount, units: units }
  end

  def generate
    total_revenue = @sales.sum { |s| s[:amount] * s[:units] }
    total_units = @sales.sum { |s| s[:units] }
    best_seller = @sales.max_by { |s| s[:units] }

    <<~REPORT
      ╔═══════════════════════════════════════╗
      ║        รายงานยอดขาย #{@month}/#{@year}            ║
      ╚═══════════════════════════════════════╝

      รายละเอียด:
      #{@sales.map { |s| "  • #{s[:product]}: #{s[:units]} ชิ้น = #{(s[:amount] * s[:units]).to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse} บาท" }.join("\n")}

      สรุป:
        รวมสินค้า: #{total_units} ชิ้น
        รวมรายได้: #{total_revenue.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse} บาท
        ขายดีสุด: #{best_seller[:product]} (#{best_seller[:units]} ชิ้น)
    REPORT
  end
end

report = SalesReport.new("มกราคม", 2024)
report.add_sale("Ruby Book", 350, 45)
report.add_sale("Rails Course", 2500, 30)
report.add_sale("VS Code Plugin", 199, 120)

puts report.generate
```

### แบบฝึกหัดที่ 14: Frozen String และ Performance

```ruby
# frozen_string_literal: true

# เปรียบเทียบ frozen vs non-frozen strings
require 'benchmark'

n = 1_000_000

# สร้าง frozen string (efficient)
frozen_greeting = "Hello, World!".freeze

time1 = Benchmark.realtime do
  n.times { frozen_greeting.dup.upcase }
end

time2 = Benchmark.realtime do
  n.times { "Hello, World!".upcase }
end

puts "Frozen string: #{time1.round(4)}s"
puts "New string:    #{time2.round(4)}s"
puts "Memory comparison:"
puts "  Frozen object_id changes: #{5.times.map { frozen_greeting.object_id }.uniq.count} unique IDs"
```

### แบบฝึกหัดที่ 15: Data Type Validation

```ruby
class DataValidator
  def self.validate_age(value)
    return { valid: false, error: "ต้องไม่เป็น nil" } if value.nil?
    return { valid: false, error: "ต้องเป็นตัวเลข" } unless value.is_a?(Numeric)
    return { valid: false, error: "ต้องเป็นจำนวนเต็ม" } unless value.is_a?(Integer)
    return { valid: false, error: "ต้องมากกว่า 0" } unless value.positive?
    return { valid: false, error: "ต้องน้อยกว่า 150" } unless value < 150

    { valid: true, value: value }
  end

  def self.validate_email(value)
    return { valid: false, error: "ต้องไม่เป็น nil" } if value.nil?
    return { valid: false, error: "ต้องเป็น String" } unless value.is_a?(String)
    return { valid: false, error: "ต้องไม่ว่างเปล่า" } if value.strip.empty?

    email_regex = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
    return { valid: false, error: "รูปแบบ email ไม่ถูกต้อง" } unless value.match?(email_regex)

    { valid: true, value: value.strip.downcase }
  end

  def self.validate_name(value)
    return { valid: false, error: "ต้องไม่เป็น nil" } if value.nil?
    return { valid: false, error: "ต้องเป็น String" } unless value.is_a?(String)
    stripped = value.strip
    return { valid: false, error: "ต้องไม่ว่างเปล่า" } if stripped.empty?
    return { valid: false, error: "ต้องมีอย่างน้อย 2 ตัวอักษร" } if stripped.length < 2

    { valid: true, value: stripped }
  end
end

test_cases = [
  { field: "age",   value: 25 },
  { field: "age",   value: -5 },
  { field: "age",   value: "25" },
  { field: "email", value: "test@example.com" },
  { field: "email", value: "invalid-email" },
  { field: "name",  value: "Alice" },
  { field: "name",  value: "A" }
]

test_cases.each do |test|
  result = case test[:field]
           when "age"   then DataValidator.validate_age(test[:value])
           when "email" then DataValidator.validate_email(test[:value])
           when "name"  then DataValidator.validate_name(test[:value])
           end

  status = result[:valid] ? "✓ ผ่าน" : "✗ ไม่ผ่าน (#{result[:error]})"
  puts "#{test[:field]} = #{test[:value].inspect} → #{status}"
end
```

---

## สรุปส่วนที่ 2

ในส่วนนี้เราได้เรียนรู้:

1. **Naming Conventions** - snake_case, PascalCase, SCREAMING_SNAKE
2. **Local Variables** - scope, multiple assignment
3. **Instance Variables** (@) - ของแต่ละ object
4. **Class Variables** (@@) - แชร์กันทุก instance
5. **Global Variables** ($) - เข้าถึงได้ทุกที่
6. **Constants** - ค่าคงที่, freeze
7. **Integer** - จำนวนเต็ม, methods
8. **Float** - ทศนิยม, precision issues
9. **String** - ข้อความ, interpolation
10. **Boolean** - true/false, truthy/falsy
11. **Nil** - ค่าว่าง, safe navigation
12. **Symbol** - immutable identifiers
13. **Type Conversion** - to_i, to_f, to_s, to_a
14. **Type Checking** - class, is_a?, respond_to?
15. **Frozen Objects** - immutability

### สิ่งสำคัญที่ต้องจำ

- ใน Ruby **nil** และ **false** เท่านั้นที่เป็น falsy
- ทุกอย่างเป็น Object รวมถึง nil, true, false
- Symbol ใช้ memory น้อยกว่า String เมื่อเปรียบเทียบ
- Integer division ใน Ruby ให้ผลเป็น Integer เสมอ
- ใช้ `&.` (safe navigation) เพื่อหลีกเลี่ยง NoMethodError กับ nil

---

*เอกสารนี้เป็นส่วนหนึ่งของคอร์ส Ruby on Rails สำหรับผู้เริ่มต้น*
