# ตอนที่ 11: Classes และ Objects (Steps 201-230)

## บทนำ

ในตอนนี้เราจะเรียนรู้เกี่ยวกับ Object-Oriented Programming (OOP) ซึ่งเป็นหัวใจสำคัญของภาษา Ruby ทุกอย่างใน Ruby คือ object และการเข้าใจ Classes จะช่วยให้คุณเขียนโค้ดที่มีโครงสร้างดี อ่านง่าย และบำรุงรักษาได้ง่ายขึ้น

---

## Step 201: Object-Oriented Programming คืออะไร?

Object-Oriented Programming (OOP) คือแนวคิดการเขียนโปรแกรมที่จัดโครงสร้างโค้ดเป็น "objects" ซึ่งแต่ละ object มีทั้ง data (attributes) และ behavior (methods)

### แนวคิดหลักของ OOP

**1. Encapsulation (การห่อหุ้มข้อมูล)**
- ข้อมูลและฟังก์ชันที่เกี่ยวข้องถูกรวมไว้ใน object เดียวกัน
- ซ่อนรายละเอียดภายในจากภายนอก

**2. Inheritance (การสืบทอด)**
- Class ลูกสามารถรับ properties และ methods จาก Class แม่

**3. Polymorphism (ความหลากหลายรูปแบบ)**
- Objects ต่างชนิดสามารถตอบสนองต่อ method เดียวกันในแบบที่แตกต่างกัน

**4. Abstraction (การนามธรรม)**
- ซ่อนความซับซ้อนและแสดงเฉพาะสิ่งที่จำเป็น

```ruby
# ตัวอย่างแนวคิด OOP เบื้องต้น
# ในโลกจริง "รถยนต์" มีคุณสมบัติ (ยี่ห้อ, สี, ความเร็ว)
# และมีพฤติกรรม (วิ่ง, หยุด, เลี้ยว)

# แบบ Procedural (ไม่ใช้ OOP)
brand = "Toyota"
color = "Red"
speed = 0

def accelerate(speed)
  speed + 10
end

speed = accelerate(speed)
puts "Speed: #{speed}"

# แบบ OOP
class Car
  def initialize(brand, color)
    @brand = brand
    @color = color
    @speed = 0
  end

  def accelerate
    @speed += 10
    puts "#{@brand} accelerates to #{@speed} km/h"
  end

  def brake
    @speed = [@speed - 10, 0].max
    puts "#{@brand} slows to #{@speed} km/h"
  end

  def info
    puts "#{@brand} (#{@color}) - Speed: #{@speed} km/h"
  end
end

my_car = Car.new("Toyota", "Red")
my_car.info
my_car.accelerate
my_car.accelerate
my_car.brake
my_car.info
```

**Output:**
```
Toyota (Red) - Speed: 0 km/h
Toyota accelerates to 10 km/h
Toyota accelerates to 20 km/h
Toyota slows to 10 km/h
Toyota (Red) - Speed: 10 km/h
```

### ทำไมต้องใช้ OOP?

```ruby
# โดยไม่ใช้ OOP - จัดการหลาย users ยากมาก
user1_name = "Alice"
user1_email = "alice@example.com"
user1_age = 30

user2_name = "Bob"
user2_email = "bob@example.com"
user2_age = 25

# ด้วย OOP - จัดการง่ายและเป็นระเบียบ
class User
  def initialize(name, email, age)
    @name = name
    @email = email
    @age = age
  end

  def greet
    puts "Hello, I'm #{@name} (#{@age} years old)"
  end

  def contact
    puts "Email: #{@email}"
  end
end

user1 = User.new("Alice", "alice@example.com", 30)
user2 = User.new("Bob", "bob@example.com", 25)

user1.greet
user2.greet
user1.contact
```

---

## Step 202: Class Definition (class ... end)

### โครงสร้างพื้นฐานของ Class

```ruby
# รูปแบบพื้นฐาน
class ClassName
  # code here
end

# ตัวอย่าง Class ง่ายๆ
class Dog
  def bark
    puts "Woof!"
  end

  def sit
    puts "Dog sits down"
  end
end

# สร้าง instance (object) จาก class
rex = Dog.new
rex.bark   # => Woof!
rex.sit    # => Dog sits down

# สร้างหลาย instances
buddy = Dog.new
buddy.bark # => Woof!
```

### การตั้งชื่อ Class

```ruby
# ชื่อ class ต้องขึ้นต้นด้วยตัวพิมพ์ใหญ่ (CamelCase)
class BankAccount   # ถูกต้อง
end

class HTTPRequest   # ถูกต้อง
end

class UserProfile   # ถูกต้อง
end

# ไม่ถูกต้อง (จะ error)
# class bankAccount  # ผิด
# class bank_account # ผิด

# ตรวจสอบว่าสิ่งที่เราสร้างเป็น instance ของ class อะไร
dog = Dog.new
puts dog.class        # => Dog
puts dog.is_a?(Dog)   # => true
puts Dog.superclass   # => Object
```

### Class เปิด (Open Class / Monkey Patching)

```ruby
# Ruby อนุญาตให้เปิด class ที่มีอยู่แล้วและเพิ่ม methods
class Dog
  def name
    "Generic Dog"
  end
end

# เปิด class เดิมและเพิ่ม method ใหม่
class Dog
  def fetch(item)
    puts "Dog fetches the #{item}!"
  end
end

rex = Dog.new
rex.bark    # ยังใช้ได้
rex.fetch("ball")  # method ใหม่

# แม้แต่ built-in classes ก็เปิดได้
class String
  def shout
    self.upcase + "!!!"
  end
end

puts "hello".shout  # => HELLO!!!
```

---

## Step 203: Instance Variables (@var)

Instance variables คือตัวแปรที่เก็บข้อมูลของแต่ละ object โดยเฉพาะ ขึ้นต้นด้วย `@`

```ruby
class Person
  def initialize(name, age)
    @name = name  # instance variable
    @age = age    # instance variable
  end

  def introduce
    puts "Hi, I'm #{@name} and I'm #{@age} years old."
  end

  def birthday
    @age += 1
    puts "Happy birthday #{@name}! Now #{@age} years old."
  end
end

alice = Person.new("Alice", 30)
bob = Person.new("Bob", 25)

alice.introduce  # => Hi, I'm Alice and I'm 30 years old.
bob.introduce    # => Hi, I'm Bob and I'm 25 years old.

alice.birthday   # => Happy birthday Alice! Now 31 years old.
alice.introduce  # => Hi, I'm Alice and I'm 31 years old.
bob.introduce    # => Hi, I'm Bob and I'm 25 years old. (ไม่เปลี่ยน)
```

### Instance Variables กับ nil

```ruby
class Cat
  def initialize(name)
    @name = name
    # @color ไม่ได้ถูก initialize
  end

  def info
    # @color จะเป็น nil ถ้าไม่ได้ set
    puts "#{@name} - Color: #{@color.inspect}"
  end

  def set_color(color)
    @color = color
  end
end

kitty = Cat.new("Kitty")
kitty.info         # => Kitty - Color: nil
kitty.set_color("white")
kitty.info         # => Kitty - Color: "white"
```

### ตรวจสอบ Instance Variables

```ruby
class Product
  def initialize(name, price)
    @name = name
    @price = price
  end
end

item = Product.new("Laptop", 25000)

# ดู instance variables ทั้งหมด
puts item.instance_variables.inspect
# => [:@name, :@price]

# อ่านค่า instance variable (ไม่แนะนำในการใช้งานจริง)
puts item.instance_variable_get(:@name)   # => Laptop
puts item.instance_variable_get(:@price)  # => 25000

# เซ็ตค่า instance variable จากภายนอก (ไม่แนะนำ)
item.instance_variable_set(:@price, 30000)
puts item.instance_variable_get(:@price)  # => 30000
```

---

## Step 204: initialize Method (Constructor)

`initialize` คือ method พิเศษที่ถูกเรียกอัตโนมัติเมื่อสร้าง object ด้วย `.new`

```ruby
class Rectangle
  def initialize(width, height)
    puts "Creating rectangle #{width}x#{height}"
    @width = width
    @height = height
  end

  def area
    @width * @height
  end

  def perimeter
    2 * (@width + @height)
  end
end

r = Rectangle.new(5, 3)
# => Creating rectangle 5x3
puts r.area       # => 15
puts r.perimeter  # => 16
```

### initialize กับ Default Values

```ruby
class Circle
  def initialize(radius, color = "red", filled = true)
    @radius = radius
    @color = color
    @filled = filled
  end

  def describe
    fill_status = @filled ? "filled" : "outline"
    puts "#{@color} #{fill_status} circle with radius #{@radius}"
  end

  def area
    Math::PI * @radius ** 2
  end
end

c1 = Circle.new(5)
c2 = Circle.new(3, "blue")
c3 = Circle.new(7, "green", false)

c1.describe  # => red filled circle with radius 5
c2.describe  # => blue filled circle with radius 3
c3.describe  # => green outline circle with radius 7

printf "Area of c1: %.2f\n", c1.area  # => Area of c1: 78.54
```

### initialize กับ Keyword Arguments

```ruby
class Student
  def initialize(name:, grade:, subject: "General")
    @name = name
    @grade = grade
    @subject = subject
  end

  def report
    puts "Student: #{@name}"
    puts "Grade: #{@grade}"
    puts "Subject: #{@subject}"
    puts "---"
  end
end

s1 = Student.new(name: "Alice", grade: "A")
s2 = Student.new(name: "Bob", grade: "B", subject: "Math")
s3 = Student.new(grade: "C", name: "Charlie", subject: "Science")

s1.report
s2.report
s3.report
```

### initialize กับ Hash Options

```ruby
class Config
  def initialize(options = {})
    @debug = options.fetch(:debug, false)
    @timeout = options.fetch(:timeout, 30)
    @max_retries = options.fetch(:max_retries, 3)
    @host = options.fetch(:host, "localhost")
  end

  def show
    puts "Debug: #{@debug}"
    puts "Timeout: #{@timeout}s"
    puts "Max Retries: #{@max_retries}"
    puts "Host: #{@host}"
  end
end

default_config = Config.new
custom_config = Config.new(debug: true, timeout: 60, host: "example.com")

puts "=== Default Config ==="
default_config.show
puts "\n=== Custom Config ==="
custom_config.show
```

---

## Step 205: Instance Methods

Instance methods คือ methods ที่เรียกใช้ผ่าน instance ของ class

```ruby
class BankAccount
  def initialize(owner, initial_balance = 0)
    @owner = owner
    @balance = initial_balance
    @transactions = []
  end

  def deposit(amount)
    if amount > 0
      @balance += amount
      @transactions << { type: :deposit, amount: amount }
      puts "Deposited #{amount}. New balance: #{@balance}"
    else
      puts "Invalid deposit amount"
    end
  end

  def withdraw(amount)
    if amount > 0 && amount <= @balance
      @balance -= amount
      @transactions << { type: :withdrawal, amount: amount }
      puts "Withdrew #{amount}. New balance: #{@balance}"
    elsif amount > @balance
      puts "Insufficient funds. Balance: #{@balance}"
    else
      puts "Invalid withdrawal amount"
    end
  end

  def balance
    @balance
  end

  def statement
    puts "=== Account Statement for #{@owner} ==="
    @transactions.each do |t|
      type = t[:type] == :deposit ? "Deposit" : "Withdrawal"
      puts "#{type}: #{t[:amount]}"
    end
    puts "Current Balance: #{@balance}"
    puts "=================================="
  end

  def transfer_to(other_account, amount)
    if @balance >= amount
      withdraw(amount)
      other_account.deposit(amount)
      puts "Transferred #{amount} to #{other_account.instance_variable_get(:@owner)}"
    else
      puts "Transfer failed: Insufficient funds"
    end
  end
end

alice_account = BankAccount.new("Alice", 1000)
bob_account = BankAccount.new("Bob", 500)

alice_account.deposit(500)
alice_account.withdraw(200)
alice_account.transfer_to(bob_account, 300)

alice_account.statement
bob_account.statement
```

### Methods ที่ Return Self (Method Chaining)

```ruby
class StringBuilder
  def initialize
    @parts = []
  end

  def add(text)
    @parts << text
    self  # return self เพื่อให้ chain ได้
  end

  def upcase
    @parts = @parts.map(&:upcase)
    self
  end

  def with_newlines
    @parts = @parts.map { |p| p + "\n" }
    self
  end

  def build
    @parts.join
  end
end

result = StringBuilder.new
  .add("Hello")
  .add(" ")
  .add("World")
  .upcase
  .build

puts result  # => HELLO WORLD

result2 = StringBuilder.new
  .add("line 1")
  .add("line 2")
  .add("line 3")
  .with_newlines
  .build

puts result2
```

---

## Step 206: attr_reader, attr_writer, attr_accessor

แทนที่จะเขียน getter/setter methods เอง Ruby มี shortcuts ให้

### attr_reader - อ่านได้อย่างเดียว

```ruby
class Temperature
  attr_reader :celsius  # สร้าง getter method: def celsius; @celsius; end

  def initialize(celsius)
    @celsius = celsius
  end

  def fahrenheit
    @celsius * 9.0 / 5 + 32
  end

  def kelvin
    @celsius + 273.15
  end
end

temp = Temperature.new(100)
puts temp.celsius    # => 100
puts temp.fahrenheit # => 212.0
puts temp.kelvin     # => 373.15

# temp.celsius = 50  # => Error! NoMethodError (ตั้งค่าไม่ได้)
```

### attr_writer - เขียนได้อย่างเดียว

```ruby
class Password
  attr_writer :password  # สร้าง setter method: def password=(val); @password = val; end

  def initialize(password)
    @password = password
  end

  def valid?(input)
    @password == input
  end
end

pwd = Password.new("secret123")
puts pwd.valid?("wrong")      # => false
puts pwd.valid?("secret123")  # => true

pwd.password = "newpassword"
puts pwd.valid?("newpassword")  # => true

# puts pwd.password  # => Error! NoMethodError (อ่านไม่ได้)
```

### attr_accessor - อ่านและเขียนได้

```ruby
class Person
  attr_accessor :name, :age, :email

  def initialize(name, age, email)
    @name = name
    @age = age
    @email = email
  end

  def introduce
    puts "Hi, I'm #{@name}, #{@age} years old"
    puts "Contact: #{@email}"
  end

  def adult?
    @age >= 18
  end
end

person = Person.new("Alice", 25, "alice@example.com")
person.introduce

# อ่านค่า
puts person.name   # => Alice
puts person.age    # => 25

# เซ็ตค่า
person.name = "Alice Smith"
person.age = 26
person.email = "alicesmith@example.com"

person.introduce
puts person.adult?  # => true
```

### ใช้หลาย attr พร้อมกัน

```ruby
class Car
  attr_reader :make, :model, :year    # อ่านได้อย่างเดียว
  attr_accessor :color, :mileage      # อ่านและเขียนได้
  attr_writer :owner                  # เขียนได้อย่างเดียว

  def initialize(make, model, year, color)
    @make = make
    @model = model
    @year = year
    @color = color
    @mileage = 0
    @owner = nil
  end

  def info
    puts "#{@year} #{@make} #{@model} (#{@color})"
    puts "Mileage: #{@mileage} km"
  end

  def drive(km)
    @mileage += km
    puts "Drove #{km} km. Total: #{@mileage} km"
  end
end

car = Car.new("Toyota", "Camry", 2022, "Silver")
car.info
car.drive(500)
car.color = "White"  # เปลี่ยนสี
car.owner = "Bob"    # เซ็ตเจ้าของ
car.info

# car.make = "Honda"   # Error! ไม่มี setter
# puts car.owner       # Error! ไม่มี getter
```

### Custom Getter/Setter

```ruby
class Product
  attr_reader :name

  def initialize(name, price)
    @name = name
    self.price = price  # ใช้ setter ของตัวเอง
  end

  def price
    "฿#{@price}"  # format ราคา
  end

  def price=(value)
    raise ArgumentError, "Price must be positive" if value <= 0
    @price = value
  end

  def raw_price
    @price
  end
end

item = Product.new("Book", 250)
puts item.price      # => ฿250

item.price = 300
puts item.price      # => ฿300

begin
  item.price = -100  # จะ raise error
rescue ArgumentError => e
  puts "Error: #{e.message}"  # => Error: Price must be positive
end
```

---

## Step 207: Class Variables (@@var) และ Class Methods (self.method)

### Class Variables

Class variables ใช้ `@@` นำหน้า และแชร์ร่วมกันระหว่าง class และ instances ทั้งหมด

```ruby
class Counter
  @@count = 0  # class variable

  def initialize
    @@count += 1
    @id = @@count
  end

  def id
    @id
  end

  def self.count  # class method
    @@count
  end

  def self.reset
    @@count = 0
  end
end

puts Counter.count  # => 0

c1 = Counter.new
c2 = Counter.new
c3 = Counter.new

puts Counter.count  # => 3
puts c1.id          # => 1
puts c2.id          # => 2
puts c3.id          # => 3

Counter.reset
puts Counter.count  # => 0

c4 = Counter.new
puts c4.id          # => 1
```

### Class Methods

Class methods เรียกใช้ผ่าน class โดยตรง ไม่ใช่ผ่าน instance

```ruby
class MathHelper
  # class methods ด้วย self
  def self.square(n)
    n * n
  end

  def self.cube(n)
    n ** 3
  end

  def self.factorial(n)
    return 1 if n <= 1
    n * factorial(n - 1)
  end

  def self.fibonacci(n)
    return n if n <= 1
    fibonacci(n - 1) + fibonacci(n - 2)
  end
end

puts MathHelper.square(5)     # => 25
puts MathHelper.cube(3)       # => 27
puts MathHelper.factorial(5)  # => 120
puts MathHelper.fibonacci(8)  # => 21
```

### Factory Methods Pattern

```ruby
class Color
  attr_reader :r, :g, :b

  def initialize(r, g, b)
    @r = r
    @g = g
    @b = b
  end

  # Factory methods ช่วยสร้าง object ด้วยวิธีต่างๆ
  def self.from_hex(hex)
    hex = hex.delete('#')
    r = hex[0..1].to_i(16)
    g = hex[2..3].to_i(16)
    b = hex[4..5].to_i(16)
    new(r, g, b)
  end

  def self.red
    new(255, 0, 0)
  end

  def self.green
    new(0, 255, 0)
  end

  def self.blue
    new(0, 0, 255)
  end

  def self.white
    new(255, 255, 255)
  end

  def self.black
    new(0, 0, 0)
  end

  def to_hex
    format("#%02X%02X%02X", @r, @g, @b)
  end

  def to_s
    "rgb(#{@r}, #{@g}, #{@b})"
  end
end

red = Color.red
puts red               # => rgb(255, 0, 0)
puts red.to_hex        # => #FF0000

from_hex = Color.from_hex("#3498db")
puts from_hex          # => rgb(52, 152, 219)

custom = Color.new(128, 64, 32)
puts custom.to_hex     # => #804020
```

### Class Methods ด้วย class << self

```ruby
class StringUtils
  class << self
    def palindrome?(str)
      clean = str.downcase.gsub(/[^a-z0-9]/, '')
      clean == clean.reverse
    end

    def word_count(str)
      str.split.length
    end

    def truncate(str, length, omission = "...")
      return str if str.length <= length
      str[0, length - omission.length] + omission
    end

    def capitalize_words(str)
      str.split.map(&:capitalize).join(' ')
    end
  end
end

puts StringUtils.palindrome?("A man a plan a canal Panama")  # => true
puts StringUtils.word_count("Hello World Ruby")              # => 3
puts StringUtils.truncate("Hello World", 8)                  # => Hello...
puts StringUtils.capitalize_words("hello world ruby")        # => Hello World Ruby
```

---

## Step 208: Constants ใน Class

Constants ใน class ขึ้นต้นด้วยตัวพิมพ์ใหญ่

```ruby
class Circle
  PI = Math::PI  # constant ใน class

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

c = Circle.new(5)
printf "Area: %.2f\n", c.area
printf "Circumference: %.2f\n", c.circumference

# เข้าถึง constant จากภายนอก
puts Circle::PI
```

### Constants ที่ซับซ้อนกว่า

```ruby
class Config
  VERSION = "1.0.0"
  MAX_RETRIES = 3
  TIMEOUT = 30
  VALID_MODES = [:development, :production, :test].freeze
  DEFAULT_SETTINGS = {
    debug: false,
    log_level: :info,
    cache: true
  }.freeze

  def initialize(mode = :development)
    unless VALID_MODES.include?(mode)
      raise ArgumentError, "Invalid mode: #{mode}. Must be one of #{VALID_MODES}"
    end
    @mode = mode
    @settings = DEFAULT_SETTINGS.dup
  end

  def setting(key)
    @settings[key]
  end

  def update_setting(key, value)
    @settings[key] = value
  end

  def info
    puts "Config v#{VERSION}"
    puts "Mode: #{@mode}"
    puts "Settings: #{@settings}"
  end
end

config = Config.new(:development)
config.info

puts Config::VERSION   # => 1.0.0
puts Config::MAX_RETRIES  # => 3

begin
  bad_config = Config.new(:invalid)
rescue ArgumentError => e
  puts "Error: #{e.message}"
end
```

---

## Step 209: Object Identity (object_id, equal?, eql?, ==)

### object_id

```ruby
# object_id คือ unique identifier ของแต่ละ object
a = "hello"
b = "hello"
c = a

puts a.object_id   # ตัวเลขบางตัว เช่น 70123456
puts b.object_id   # ตัวเลขต่างกัน (คนละ object)
puts c.object_id   # เหมือนกับ a (ชี้ไป object เดียวกัน)

puts a.object_id == b.object_id  # => false (คนละ object)
puts a.object_id == c.object_id  # => true (object เดียวกัน)

# Special objects มี object_id คงที่
puts nil.object_id    # => 8
puts true.object_id   # => 2
puts false.object_id  # => 0
puts 1.object_id      # => 3 (integer: 2n+1)
puts 2.object_id      # => 5
```

### equal?, eql?, ==

```ruby
a = "hello"
b = "hello"
c = a

# equal? - เปรียบเทียบ object identity (object เดียวกันไหม?)
puts a.equal?(b)  # => false (คนละ object)
puts a.equal?(c)  # => true (object เดียวกัน)

# eql? - เปรียบเทียบ value และ type
puts 1.eql?(1)    # => true
puts 1.eql?(1.0)  # => false (ต่าง type)
puts "hi".eql?("hi")  # => true

# == - เปรียบเทียบ value (อาจมี type coercion)
puts 1 == 1       # => true
puts 1 == 1.0     # => true (coercion)
puts "hi" == "hi" # => true

# กับ custom class
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
  end

  def ==(other)
    other.is_a?(Point) && x == other.x && y == other.y
  end

  def eql?(other)
    other.is_a?(Point) && x.eql?(other.x) && y.eql?(other.y)
  end

  def hash
    [x, y].hash
  end
end

p1 = Point.new(1, 2)
p2 = Point.new(1, 2)
p3 = p1

puts p1 == p2       # => true (ค่าเท่ากัน)
puts p1.equal?(p2)  # => false (คนละ object)
puts p1.equal?(p3)  # => true (object เดียวกัน)
puts p1.eql?(p2)    # => true
```

### ใช้ == ใน Hash และ Array

```ruby
class Product
  attr_reader :sku, :name

  def initialize(sku, name, price)
    @sku = sku
    @name = name
    @price = price
  end

  def ==(other)
    other.is_a?(Product) && sku == other.sku
  end

  def eql?(other)
    other.is_a?(Product) && sku.eql?(other.sku)
  end

  def hash
    sku.hash
  end
end

p1 = Product.new("SKU001", "Laptop", 25000)
p2 = Product.new("SKU001", "Laptop Pro", 30000)
p3 = Product.new("SKU002", "Mouse", 500)

puts p1 == p2  # => true (same SKU)
puts p1 == p3  # => false

# ใช้ใน Array uniq
products = [p1, p2, p3]
puts products.uniq.length  # => 2 (p1 และ p2 ถือว่าเหมือนกัน)

# ใช้ใน Set
require 'set'
product_set = Set.new([p1, p2, p3])
puts product_set.size  # => 2
```

---

## Step 210: to_s และ inspect

### to_s

`to_s` คือ method ที่แปลง object เป็น String เมื่อใช้ใน string interpolation

```ruby
class Book
  def initialize(title, author, pages)
    @title = title
    @author = author
    @pages = pages
  end

  def to_s
    "\"#{@title}\" by #{@author} (#{@pages} pages)"
  end
end

book = Book.new("Ruby Programming", "Matz", 450)
puts book            # => "Ruby Programming" by Matz (450 pages)
puts "Book: #{book}" # => Book: "Ruby Programming" by Matz (450 pages)
puts book.to_s       # => "Ruby Programming" by Matz (450 pages)
```

### inspect

`inspect` ใช้สำหรับ debugging แสดงรายละเอียดมากกว่า `to_s`

```ruby
class Point
  def initialize(x, y)
    @x = x
    @y = y
  end

  def to_s
    "(#{@x}, #{@y})"
  end

  def inspect
    "#<Point x=#{@x}, y=#{@y}>"
  end
end

p1 = Point.new(3, 4)
puts p1            # => (3, 4)  (ใช้ to_s)
puts p1.inspect    # => #<Point x=3, y=4>
p p1               # => #<Point x=3, y=4>  (p ใช้ inspect)

# Array ใช้ inspect ของ elements
points = [Point.new(1,2), Point.new(3,4)]
puts points.inspect
# => [#<Point x=1, y=2>, #<Point x=3, y=4>]
```

### ตัวอย่างสมบูรณ์

```ruby
class Invoice
  attr_reader :number, :items, :date

  def initialize(number, date = Date.today)
    @number = number
    @date = date
    @items = []
  end

  def add_item(description, quantity, unit_price)
    @items << {
      description: description,
      quantity: quantity,
      unit_price: unit_price,
      total: quantity * unit_price
    }
  end

  def subtotal
    @items.sum { |item| item[:total] }
  end

  def vat(rate = 0.07)
    subtotal * rate
  end

  def total
    subtotal + vat
  end

  def to_s
    "Invoice ##{@number} (#{@items.length} items) - Total: ฿#{format('%.2f', total)}"
  end

  def inspect
    "#<Invoice number=#{@number.inspect} date=#{@date.inspect} items=#{@items.length} total=#{format('%.2f', total)}>"
  end

  def print_invoice
    puts "=" * 50
    puts "Invoice ##{@number}"
    puts "Date: #{@date}"
    puts "-" * 50
    puts format("%-25s %5s %10s %10s", "Description", "Qty", "Unit", "Total")
    puts "-" * 50
    @items.each do |item|
      puts format("%-25s %5d %10.2f %10.2f",
        item[:description], item[:quantity],
        item[:unit_price], item[:total])
    end
    puts "-" * 50
    puts format("%-40s %10.2f", "Subtotal:", subtotal)
    puts format("%-40s %10.2f", "VAT (7%):", vat)
    puts format("%-40s %10.2f", "Total:", total)
    puts "=" * 50
  end
end

require 'date'
inv = Invoice.new("INV-2024-001", Date.new(2024, 1, 15))
inv.add_item("Ruby Programming Book", 2, 350)
inv.add_item("Ruby on Rails Course", 1, 1200)
inv.add_item("RSpec Testing Guide", 3, 280)

puts inv          # ใช้ to_s
p inv             # ใช้ inspect
inv.print_invoice
```

---

## Step 211: freeze / frozen?

`freeze` ป้องกันไม่ให้ object ถูกแก้ไข

```ruby
# freeze string
str = "Hello"
str.freeze

str << " World"   # => FrozenError (RuntimeError ใน Ruby เก่ากว่า)
str.upcase!       # => FrozenError

puts str.frozen?  # => true

# freeze array
arr = [1, 2, 3].freeze
arr << 4          # => FrozenError
arr[0] = 10       # => FrozenError
puts arr.frozen?  # => true

# freeze hash
hash = { a: 1, b: 2 }.freeze
hash[:c] = 3      # => FrozenError
hash.delete(:a)   # => FrozenError
```

### freeze ใน class

```ruby
class Configuration
  DEFAULTS = {
    host: "localhost",
    port: 3000,
    debug: false
  }.freeze  # prevent modification

  MAX_CONNECTIONS = 100
  VALID_ENVIRONMENTS = %w[development staging production].freeze

  attr_reader :host, :port, :debug

  def initialize(env = "development")
    unless VALID_ENVIRONMENTS.include?(env)
      raise ArgumentError, "Invalid environment: #{env}"
    end
    @env = env
    @host = DEFAULTS[:host]
    @port = DEFAULTS[:port]
    @debug = DEFAULTS[:debug]
  end

  def production?
    @env == "production"
  end
end

config = Configuration.new("development")
puts config.host
puts Configuration::MAX_CONNECTIONS

begin
  Configuration::DEFAULTS[:host] = "hacked"
rescue => e
  puts "Cannot modify: #{e.class}"  # => Cannot modify: FrozenError
end
```

### dup vs freeze

```ruby
str = "Hello"
str.freeze

# dup สร้าง copy ที่ไม่ frozen
copy = str.dup
copy << " World"  # ทำได้!
puts copy         # => Hello World
puts copy.frozen? # => false

# clone รักษา frozen state
clone = str.clone
puts clone.frozen?  # => true
```

---

## Step 212: dup vs clone

```ruby
class Config
  attr_accessor :settings, :name

  def initialize(name)
    @name = name
    @settings = { debug: false, timeout: 30 }
  end

  def to_s
    "Config(#{@name}): #{@settings}"
  end
end

original = Config.new("main")
original.settings[:debug] = true

# dup - shallow copy
duped = original.dup
puts duped.name         # => main
puts duped.settings     # => {:debug=>true, :timeout=>30}

duped.name = "copied"
duped.settings[:timeout] = 60

puts original.name       # => main (ไม่เปลี่ยน)
puts original.settings   # => {:debug=>true, :timeout=>60} (เปลี่ยน! เพราะ shallow copy)

# Deep copy ต้องทำเอง
class Config
  def deep_dup
    copy = dup
    copy.settings = settings.dup
    copy
  end
end

original2 = Config.new("main2")
original2.settings[:debug] = true

deep_copy = original2.deep_dup
deep_copy.settings[:timeout] = 999

puts original2.settings  # => {:debug=>true, :timeout=>30} (ไม่เปลี่ยน)
```

### clone vs dup

```ruby
# ความแตกต่างหลัก:
# 1. clone รักษา frozen state, dup ไม่รักษา
# 2. clone คัดลอก singleton methods, dup ไม่คัดลอก
# 3. clone คัดลอก internal state markers

frozen_str = "hello".freeze
puts frozen_str.dup.frozen?    # => false
puts frozen_str.clone.frozen?  # => true

# Singleton methods
obj = Object.new
def obj.hello
  "Hello from singleton!"
end

cloned = obj.clone
duped = obj.dup

puts cloned.hello   # => Hello from singleton!
# puts duped.hello  # => NoMethodError (ไม่มี singleton method)
```

---

## Step 213: Struct

Struct คือวิธีสร้าง class อย่างง่ายที่มี attributes กำหนดไว้

```ruby
# สร้าง Struct
Point = Struct.new(:x, :y)

p1 = Point.new(3, 4)
puts p1.x     # => 3
puts p1.y     # => 4
puts p1       # => #<struct Point x=3, y=4>

# Struct มี == ให้เลยโดย default
p2 = Point.new(3, 4)
p3 = Point.new(1, 2)
puts p1 == p2  # => true
puts p1 == p3  # => false

# แปลงเป็น Array หรือ Hash
puts p1.to_a.inspect    # => [3, 4]
puts p1.to_h.inspect    # => {:x=>3, :y=>4}
```

### Struct กับ Methods เพิ่มเติม

```ruby
Person = Struct.new(:name, :age, :email) do
  def adult?
    age >= 18
  end

  def greet
    "Hello, I'm #{name}!"
  end

  def to_s
    "#{name} (#{age})"
  end
end

alice = Person.new("Alice", 25, "alice@example.com")
bob = Person.new("Bob", 16, "bob@example.com")

puts alice.greet    # => Hello, I'm Alice!
puts alice.adult?   # => true
puts bob.adult?     # => false
puts alice          # => Alice (25)

# Struct members
puts Person.members.inspect  # => [:name, :age, :email]

# Iterate ค่า
alice.each_pair do |member, value|
  puts "#{member}: #{value}"
end
```

### Struct ใน Practice

```ruby
# ใช้ Struct สำหรับ Value Objects
Coordinate = Struct.new(:latitude, :longitude) do
  def distance_to(other)
    # Haversine formula (simplified)
    lat_diff = (latitude - other.latitude).abs
    lon_diff = (longitude - other.longitude).abs
    Math.sqrt(lat_diff**2 + lon_diff**2) * 111  # rough km
  end

  def to_s
    "(#{latitude}°, #{longitude}°)"
  end
end

bangkok = Coordinate.new(13.7563, 100.5018)
chiangmai = Coordinate.new(18.7883, 98.9853)

puts bangkok        # => (13.7563°, 100.5018°)
puts chiangmai      # => (18.7883°, 98.9853°)
puts "Distance: #{bangkok.distance_to(chiangmai).round(0)} km"

# Struct เป็น immutable ได้ด้วย keyword_init
Config = Struct.new(:host, :port, :debug, keyword_init: true)
config = Config.new(host: "localhost", port: 3000, debug: false)
puts config.host   # => localhost
puts config.port   # => 3000
```

---

## Step 214: OpenStruct

OpenStruct ช่วยสร้าง object แบบ dynamic ที่เพิ่ม attributes ได้ตอน runtime

```ruby
require 'ostruct'

# สร้าง OpenStruct
person = OpenStruct.new(name: "Alice", age: 30)
puts person.name    # => Alice
puts person.age     # => 30

# เพิ่ม attribute ใหม่ได้เลย
person.email = "alice@example.com"
person.job = "Developer"
puts person.email   # => alice@example.com
puts person.job     # => Developer

# Predicate method อัตโนมัติ
puts person.name?   # => true (มีค่าและไม่ nil/false)

# แปลงเป็น Hash
puts person.to_h.inspect
```

### OpenStruct สำหรับ Config

```ruby
require 'ostruct'

def create_config(options = {})
  config = OpenStruct.new(
    host: "localhost",
    port: 3000,
    debug: false,
    max_connections: 10
  )

  options.each do |key, value|
    config.send("#{key}=", value)
  end

  config
end

dev_config = create_config(debug: true)
prod_config = create_config(host: "example.com", port: 443, debug: false)

puts "Dev: #{dev_config.host}:#{dev_config.port} debug=#{dev_config.debug}"
puts "Prod: #{prod_config.host}:#{prod_config.port} debug=#{prod_config.debug}"
```

### เปรียบเทียบ Struct vs OpenStruct

```ruby
require 'ostruct'

puts "=== Struct ==="
Point = Struct.new(:x, :y)
p1 = Point.new(1, 2)
puts p1.x
# p1.z = 3  # => NoMethodError

puts "\n=== OpenStruct ==="
p2 = OpenStruct.new(x: 1, y: 2)
p2.z = 3    # ได้!
puts p2.x
puts p2.z

# Performance: Struct เร็วกว่า OpenStruct มาก
# ใช้ Struct เมื่อรู้ attributes ล่วงหน้า
# ใช้ OpenStruct เมื่อต้องการความยืดหยุ่น
```

---

## Step 215: Comparable Module ใน Class

Comparable module ช่วยเพิ่ม comparison operators โดยเพียงแค่ implement `<=>`

```ruby
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  # Spaceship operator - ต้อง implement เพื่อใช้ Comparable
  def <=>(other)
    degrees <=> other.degrees
  end

  def to_s
    "#{degrees}°"
  end
end

temps = [Temperature.new(100), Temperature.new(37), Temperature.new(0), Temperature.new(22)]

puts temps.min         # => 0°
puts temps.max         # => 100°
puts temps.sort.map(&:to_s).inspect
# => ["0°", "22°", "37°", "100°"]

t1 = Temperature.new(37)
t2 = Temperature.new(100)

puts t1 < t2   # => true
puts t1 > t2   # => false
puts t1 <= t2  # => true
puts t1.between?(Temperature.new(0), Temperature.new(100))  # => true
puts temps.sort.first   # => 0°
```

### Comparable ใน Class ที่ซับซ้อน

```ruby
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
    return major <=> other.major unless major == other.major
    return minor <=> other.minor unless minor == other.minor
    patch <=> other.patch
  end

  def to_s
    "#{major}.#{minor}.#{patch}"
  end
end

v1 = Version.new("1.2.3")
v2 = Version.new("1.2.4")
v3 = Version.new("2.0.0")
v4 = Version.new("1.2.3")

puts v1 < v2   # => true
puts v1 > v3   # => false
puts v1 == v4  # => true

versions = [v3, v1, v2, Version.new("1.0.0")]
puts versions.sort.map(&:to_s).inspect
# => ["1.0.0", "1.2.3", "1.2.4", "2.0.0"]

puts versions.max  # => 2.0.0
```

---

## Step 216: Building Example - BankAccount Class

```ruby
class BankAccount
  attr_reader :account_number, :owner, :balance, :account_type

  @@total_accounts = 0
  @@next_account_number = 1000

  def initialize(owner, initial_deposit = 0, account_type = :checking)
    raise ArgumentError, "Initial deposit cannot be negative" if initial_deposit < 0
    raise ArgumentError, "Invalid account type" unless [:checking, :savings].include?(account_type)

    @@total_accounts += 1
    @@next_account_number += 1

    @account_number = "ACC#{@@next_account_number}"
    @owner = owner
    @balance = initial_deposit
    @account_type = account_type
    @transactions = []
    @created_at = Time.now

    record_transaction(:initial_deposit, initial_deposit, "Account opened") if initial_deposit > 0
  end

  def deposit(amount, description = "Deposit")
    validate_amount(amount)
    @balance += amount
    record_transaction(:deposit, amount, description)
    puts "✓ Deposited ฿#{format_amount(amount)}. Balance: ฿#{format_amount(@balance)}"
    self
  end

  def withdraw(amount, description = "Withdrawal")
    validate_amount(amount)
    raise "Insufficient funds. Available: ฿#{format_amount(@balance)}" if amount > @balance
    @balance -= amount
    record_transaction(:withdrawal, amount, description)
    puts "✓ Withdrew ฿#{format_amount(amount)}. Balance: ฿#{format_amount(@balance)}"
    self
  end

  def transfer_to(target_account, amount, description = nil)
    desc = description || "Transfer to #{target_account.account_number}"
    withdraw(amount, desc)
    target_account.deposit(amount, "Transfer from #{@account_number}")
    puts "✓ Transfer complete"
    self
  end

  def interest_rate
    case @account_type
    when :savings then 0.025
    when :checking then 0.001
    end
  end

  def apply_interest
    interest = (@balance * interest_rate).round(2)
    deposit(interest, "Monthly interest (#{(interest_rate * 100).round(1)}%)")
  end

  def statement(last_n = nil)
    puts "\n" + "=" * 55
    puts " Account Statement"
    puts " Account: #{@account_number}"
    puts " Owner:   #{@owner}"
    puts " Type:    #{@account_type.to_s.capitalize}"
    puts " Opened:  #{@created_at.strftime('%Y-%m-%d')}"
    puts "=" * 55
    puts format("%-12s %-10s %-20s %10s", "Date", "Type", "Description", "Amount")
    puts "-" * 55

    txns = last_n ? @transactions.last(last_n) : @transactions
    txns.each do |t|
      sign = t[:type] == :deposit || t[:type] == :initial_deposit ? "+" : "-"
      puts format("%-12s %-10s %-20s %+10.2f",
        t[:date].strftime('%Y-%m-%d'),
        t[:type].to_s.split('_').map(&:capitalize).first(2).join(' '),
        t[:description][0..19],
        t[:type] == :withdrawal ? -t[:amount] : t[:amount]
      )
    end

    puts "-" * 55
    puts format("%-42s %10.2f", "Current Balance:", @balance)
    puts "=" * 55
    puts
  end

  def self.total_accounts
    @@total_accounts
  end

  def to_s
    "BankAccount[#{@account_number}] #{@owner}: ฿#{format_amount(@balance)}"
  end

  def inspect
    "#<BankAccount account_number=#{@account_number.inspect} owner=#{@owner.inspect} " \
    "balance=#{@balance} type=#{@account_type}>"
  end

  private

  def validate_amount(amount)
    raise ArgumentError, "Amount must be a positive number" unless amount.is_a?(Numeric) && amount > 0
  end

  def record_transaction(type, amount, description)
    @transactions << {
      type: type,
      amount: amount,
      description: description,
      date: Time.now,
      balance_after: @balance
    }
  end

  def format_amount(amount)
    format("%.2f", amount)
  end
end

# ทดสอบ BankAccount
puts "=== BankAccount Demo ==="
puts "Total accounts: #{BankAccount.total_accounts}"

alice = BankAccount.new("Alice", 10000, :savings)
bob = BankAccount.new("Bob", 5000, :checking)

puts "Total accounts: #{BankAccount.total_accounts}"

alice.deposit(5000, "Salary")
alice.deposit(2000, "Freelance work")
alice.withdraw(3000, "Rent payment")

bob.deposit(3000, "Bonus")
alice.transfer_to(bob, 1500, "Loan repayment")

alice.apply_interest

alice.statement
bob.statement

puts alice
puts alice.inspect
```

---

## Step 217: Building Example - Person Class

```ruby
class Person
  include Comparable

  attr_accessor :name, :email, :phone
  attr_reader :birth_date, :id

  @@count = 0

  def initialize(name, birth_date, email = nil, phone = nil)
    @@count += 1
    @id = @@count
    @name = name
    @birth_date = parse_date(birth_date)
    @email = email
    @phone = phone
    @friends = []
    @hobbies = []
  end

  def age
    now = Date.today
    years = now.year - @birth_date.year
    years -= 1 if now < Date.new(now.year, @birth_date.month, @birth_date.day)
    years
  end

  def adult?
    age >= 18
  end

  def birthday_today?
    today = Date.today
    @birth_date.month == today.month && @birth_date.day == today.day
  end

  def days_until_birthday
    today = Date.today
    this_year_birthday = Date.new(today.year, @birth_date.month, @birth_date.day)
    this_year_birthday = Date.new(today.year + 1, @birth_date.month, @birth_date.day) if this_year_birthday < today
    (this_year_birthday - today).to_i
  end

  def add_friend(person)
    unless @friends.include?(person)
      @friends << person
      person.add_friend(self) unless person.friends.include?(self)
      puts "#{@name} and #{person.name} are now friends!"
    end
  end

  def add_hobby(hobby)
    @hobbies << hobby unless @hobbies.include?(hobby)
  end

  def friends
    @friends.dup
  end

  def hobbies
    @hobbies.dup
  end

  def mutual_friends_with(person)
    @friends & person.friends
  end

  def <=>(other)
    age <=> other.age
  end

  def to_s
    "#{@name} (age #{age})"
  end

  def inspect
    "#<Person id=#{@id} name=#{@name.inspect} age=#{age}>"
  end

  def profile
    puts "=" * 40
    puts "Profile"
    puts "=" * 40
    puts "ID:      #{@id}"
    puts "Name:    #{@name}"
    puts "Age:     #{age}"
    puts "Born:    #{@birth_date.strftime('%B %d, %Y')}"
    puts "Email:   #{@email || 'N/A'}"
    puts "Phone:   #{@phone || 'N/A'}"
    puts "Adult:   #{adult? ? 'Yes' : 'No'}"
    puts "Birthday in #{days_until_birthday} days" unless birthday_today?
    puts "*** TODAY IS BIRTHDAY! ***" if birthday_today?
    puts "Friends: #{@friends.map(&:name).join(', ')}" unless @friends.empty?
    puts "Hobbies: #{@hobbies.join(', ')}" unless @hobbies.empty?
    puts "=" * 40
  end

  def self.count
    @@count
  end

  private

  def parse_date(date)
    case date
    when Date then date
    when String then Date.parse(date)
    else raise ArgumentError, "Invalid date format"
    end
  end
end

require 'date'
alice = Person.new("Alice", "1995-06-15", "alice@example.com", "081-234-5678")
bob = Person.new("Bob", "1998-03-22", "bob@example.com")
charlie = Person.new("Charlie", "2000-11-30", "charlie@example.com")

alice.add_hobby("Programming")
alice.add_hobby("Reading")
alice.add_hobby("Hiking")
bob.add_hobby("Gaming")
bob.add_hobby("Programming")

alice.add_friend(bob)
bob.add_friend(charlie)
alice.add_friend(charlie)

alice.profile
bob.profile

puts "Mutual friends between Alice and Charlie:"
mutual = alice.mutual_friends_with(charlie)
mutual.each { |f| puts "  - #{f.name}" }

people = [alice, bob, charlie]
puts "\nSorted by age:"
people.sort.each { |p| puts "  #{p}" }

puts "\nYoungest: #{people.min}"
puts "Oldest: #{people.max}"
puts "Total people created: #{Person.count}"
```

---

## Step 218: Building Example - Rectangle Class

```ruby
class Rectangle
  include Comparable

  attr_reader :width, :height, :color

  def initialize(width, height, color = "transparent")
    raise ArgumentError, "Width must be positive" unless width > 0
    raise ArgumentError, "Height must be positive" unless height > 0
    @width = width.to_f
    @height = height.to_f
    @color = color
    @x = 0.0
    @y = 0.0
  end

  def area
    @width * @height
  end

  def perimeter
    2 * (@width + @height)
  end

  def diagonal
    Math.sqrt(@width**2 + @height**2)
  end

  def square?
    @width == @height
  end

  def resize!(scale_factor)
    raise ArgumentError, "Scale factor must be positive" unless scale_factor > 0
    @width *= scale_factor
    @height *= scale_factor
    self
  end

  def scale_to_fit(max_width, max_height)
    scale_w = max_width.to_f / @width
    scale_h = max_height.to_f / @height
    scale = [scale_w, scale_h].min
    Rectangle.new(@width * scale, @height * scale, @color)
  end

  def move_to(x, y)
    @x = x.to_f
    @y = y.to_f
    self
  end

  def overlaps?(other)
    x_overlap = @x < other.instance_variable_get(:@x) + other.width &&
                @x + @width > other.instance_variable_get(:@x)
    y_overlap = @y < other.instance_variable_get(:@y) + other.height &&
                @y + @height > other.instance_variable_get(:@y)
    x_overlap && y_overlap
  end

  def contains?(x, y)
    x.between?(@x, @x + @width) && y.between?(@y, @y + @height)
  end

  def <=>(other)
    area <=> other.area
  end

  def ==(other)
    other.is_a?(Rectangle) && width == other.width && height == other.height
  end

  def +(other)
    # ขยาย bounding box
    new_width = [@width, other.width].max
    new_height = @height + other.height
    Rectangle.new(new_width, new_height)
  end

  def to_s
    square_note = square? ? " (Square)" : ""
    "Rectangle #{@width}x#{@height}#{square_note} [area=#{format('%.2f', area)}, color=#{@color}]"
  end

  def inspect
    "#<Rectangle w=#{@width} h=#{@height} area=#{format('%.2f', area)} " \
    "pos=(#{@x},#{@y}) color=#{@color.inspect}>"
  end

  def draw_ascii
    w = [@width.ceil, 40].min
    h = [@height.ceil, 20].min
    puts "+" + "-" * (w - 2) + "+"
    (h - 2).times { puts "|" + " " * (w - 2) + "|" }
    puts "+" + "-" * (w - 2) + "+"
  end
end

# ทดสอบ Rectangle
r1 = Rectangle.new(10, 5, "red")
r2 = Rectangle.new(7, 7, "blue")
r3 = Rectangle.new(3, 8, "green")

puts r1
puts r2
puts r3

puts "\nComparisons:"
puts "r1 > r2: #{r1 > r2}"
puts "r2 square?: #{r2.square?}"

rects = [r1, r2, r3]
puts "\nSorted by area:"
rects.sort.each { |r| puts "  #{r}" }

puts "\nLargest: #{rects.max}"
puts "Smallest: #{rects.min}"

puts "\nDiagonal of r1: #{format('%.2f', r1.diagonal)}"

r1.move_to(0, 0)
r4 = Rectangle.new(8, 4)
r4.move_to(5, 3)
puts "\nr1 overlaps r4: #{r1.overlaps?(r4)}"

puts "\nr1 scaled to fit 5x5:"
fitted = r1.scale_to_fit(5, 5)
puts fitted

puts "\nASCII art of 10x5 rectangle:"
Rectangle.new(10, 5).draw_ascii
```

---

## Step 219-230: แบบฝึกหัด 30 ข้อ พร้อมเฉลย

### ข้อที่ 1: สร้าง class Dog

**โจทย์:** สร้าง class `Dog` ที่มี name, breed, age และ methods bark, eat, sleep, birthday

```ruby
class Dog
  attr_accessor :name, :breed, :age

  def initialize(name, breed, age)
    @name = name
    @breed = breed
    @age = age
  end

  def bark
    puts "#{@name} says: Woof! Woof!"
  end

  def eat(food)
    puts "#{@name} eats #{food}. Yum!"
  end

  def sleep_time
    puts "#{@name} is sleeping... Zzz..."
  end

  def birthday
    @age += 1
    puts "Happy Birthday #{@name}! Now #{@age} years old!"
  end

  def to_s
    "#{@name} (#{@breed}, age #{@age})"
  end
end

dog = Dog.new("Rex", "Labrador", 3)
dog.bark
dog.eat("chicken")
dog.sleep_time
dog.birthday
puts dog
```

### ข้อที่ 2: Stack Class

**โจทย์:** สร้าง class `Stack` ที่มี push, pop, peek, empty?, size

```ruby
class Stack
  def initialize
    @data = []
  end

  def push(item)
    @data.push(item)
    self
  end

  def pop
    raise "Stack is empty" if empty?
    @data.pop
  end

  def peek
    raise "Stack is empty" if empty?
    @data.last
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end

  def to_s
    "Stack: #{@data.inspect}"
  end
end

stack = Stack.new
stack.push(1).push(2).push(3)
puts stack           # => Stack: [1, 2, 3]
puts stack.peek      # => 3
puts stack.pop       # => 3
puts stack.size      # => 2
puts stack.empty?    # => false
```

### ข้อที่ 3: Queue Class

**โจทย์:** สร้าง class `Queue` ที่มี enqueue, dequeue, front, empty?, size

```ruby
class Queue
  def initialize
    @data = []
  end

  def enqueue(item)
    @data.push(item)
    self
  end

  def dequeue
    raise "Queue is empty" if empty?
    @data.shift
  end

  def front
    raise "Queue is empty" if empty?
    @data.first
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end

  def to_s
    "Queue(front→back): #{@data.inspect}"
  end
end

q = Queue.new
q.enqueue("Alice").enqueue("Bob").enqueue("Charlie")
puts q
puts q.front     # => Alice
puts q.dequeue   # => Alice
puts q           # => Queue(front→back): ["Bob", "Charlie"]
```

### ข้อที่ 4: Matrix Class

**โจทย์:** สร้าง class `Matrix2x2` ที่มี +, *, determinant, transpose

```ruby
class Matrix2x2
  attr_reader :a, :b, :c, :d

  def initialize(a, b, c, d)
    @a, @b, @c, @d = a, b, c, d
  end

  def +(other)
    Matrix2x2.new(@a + other.a, @b + other.b, @c + other.c, @d + other.d)
  end

  def *(other)
    Matrix2x2.new(
      @a * other.a + @b * other.c,
      @a * other.b + @b * other.d,
      @c * other.a + @d * other.c,
      @c * other.b + @d * other.d
    )
  end

  def determinant
    @a * @d - @b * @c
  end

  def transpose
    Matrix2x2.new(@a, @c, @b, @d)
  end

  def to_s
    "| #{@a} #{@b} |\n| #{@c} #{@d} |"
  end
end

m1 = Matrix2x2.new(1, 2, 3, 4)
m2 = Matrix2x2.new(5, 6, 7, 8)

puts "M1:\n#{m1}"
puts "M2:\n#{m2}"
puts "M1 + M2:\n#{m1 + m2}"
puts "M1 * M2:\n#{m1 * m2}"
puts "Det(M1): #{m1.determinant}"
puts "Transpose(M1):\n#{m1.transpose}"
```

### ข้อที่ 5: LinkedList Node

**โจทย์:** สร้าง class `Node` และ `LinkedList` ที่มี append, prepend, delete, include?, to_s

```ruby
class Node
  attr_accessor :value, :next_node

  def initialize(value, next_node = nil)
    @value = value
    @next_node = next_node
  end
end

class LinkedList
  def initialize
    @head = nil
    @size = 0
  end

  def append(value)
    if @head.nil?
      @head = Node.new(value)
    else
      current = @head
      current = current.next_node while current.next_node
      current.next_node = Node.new(value)
    end
    @size += 1
    self
  end

  def prepend(value)
    @head = Node.new(value, @head)
    @size += 1
    self
  end

  def delete(value)
    return if @head.nil?
    if @head.value == value
      @head = @head.next_node
      @size -= 1
      return
    end
    current = @head
    while current.next_node
      if current.next_node.value == value
        current.next_node = current.next_node.next_node
        @size -= 1
        return
      end
      current = current.next_node
    end
  end

  def include?(value)
    current = @head
    while current
      return true if current.value == value
      current = current.next_node
    end
    false
  end

  def size
    @size
  end

  def to_s
    result = []
    current = @head
    while current
      result << current.value.to_s
      current = current.next_node
    end
    result.join(" -> ")
  end
end

list = LinkedList.new
list.append(1).append(2).append(3)
list.prepend(0)
puts list               # => 0 -> 1 -> 2 -> 3
puts list.size          # => 4
puts list.include?(2)   # => true
list.delete(2)
puts list               # => 0 -> 1 -> 3
puts list.include?(2)   # => false
```

### ข้อที่ 6-10: เพิ่มเติม

```ruby
# ข้อที่ 6: Timer class
class Timer
  def initialize
    @elapsed = 0
    @running = false
    @start_time = nil
  end

  def start
    unless @running
      @start_time = Time.now
      @running = true
    end
    self
  end

  def stop
    if @running
      @elapsed += Time.now - @start_time
      @running = false
    end
    self
  end

  def reset
    @elapsed = 0
    @running = false
    @start_time = nil
    self
  end

  def elapsed
    if @running
      @elapsed + (Time.now - @start_time)
    else
      @elapsed
    end
  end

  def to_s
    mins = elapsed.to_i / 60
    secs = elapsed % 60
    format("%02d:%05.2f", mins, secs)
  end
end

t = Timer.new
t.start
sleep(0.1)
t.stop
puts "Elapsed: #{t}"
```

```ruby
# ข้อที่ 7: Fraction class
class Fraction
  include Comparable
  attr_reader :numerator, :denominator

  def initialize(numerator, denominator)
    raise ZeroDivisionError, "Denominator cannot be zero" if denominator == 0
    sign = denominator < 0 ? -1 : 1
    g = gcd(numerator.abs, denominator.abs)
    @numerator = sign * numerator / g
    @denominator = denominator.abs / g
  end

  def +(other)
    Fraction.new(
      numerator * other.denominator + other.numerator * denominator,
      denominator * other.denominator
    )
  end

  def -(other)
    Fraction.new(
      numerator * other.denominator - other.numerator * denominator,
      denominator * other.denominator
    )
  end

  def *(other)
    Fraction.new(numerator * other.numerator, denominator * other.denominator)
  end

  def /(other)
    Fraction.new(numerator * other.denominator, denominator * other.numerator)
  end

  def <=>(other)
    (numerator * other.denominator) <=> (other.numerator * denominator)
  end

  def to_f
    numerator.to_f / denominator
  end

  def to_s
    denominator == 1 ? numerator.to_s : "#{numerator}/#{denominator}"
  end

  private

  def gcd(a, b)
    b.zero? ? a : gcd(b, a % b)
  end
end

a = Fraction.new(1, 2)
b = Fraction.new(1, 3)
puts "#{a} + #{b} = #{a + b}"    # => 1/2 + 1/3 = 5/6
puts "#{a} * #{b} = #{a * b}"    # => 1/2 * 1/3 = 1/6
puts "#{a} / #{b} = #{a / b}"    # => 1/2 / 1/3 = 3/2
puts "#{a} > #{b}: #{a > b}"     # => 1/2 > 1/3: true
```

```ruby
# ข้อที่ 8: Calendar Event
class CalendarEvent
  include Comparable

  attr_accessor :title, :description
  attr_reader :start_time, :end_time, :location

  def initialize(title, start_time, duration_minutes, location = nil, description = nil)
    @title = title
    @start_time = start_time
    @end_time = start_time + duration_minutes * 60
    @location = location
    @description = description
  end

  def duration_minutes
    ((@end_time - @start_time) / 60).round
  end

  def overlaps?(other)
    start_time < other.end_time && end_time > other.start_time
  end

  def <=>(other)
    start_time <=> other.start_time
  end

  def to_s
    time_str = "#{start_time.strftime('%H:%M')}-#{end_time.strftime('%H:%M')}"
    "#{title} (#{time_str}#{@location ? ", #{@location}" : ""})"
  end
end

now = Time.now
events = [
  CalendarEvent.new("Team Meeting", now + 3600, 60, "Conference Room A"),
  CalendarEvent.new("Lunch", now + 7200, 60, "Cafeteria"),
  CalendarEvent.new("Code Review", now + 1800, 30),
  CalendarEvent.new("Deploy", now + 5400, 45, "Remote"),
]

puts "Events (sorted):"
events.sort.each { |e| puts "  #{e}" }

overlap = events[0].overlaps?(events[1])
puts "\nMeeting overlaps with Lunch: #{overlap}"
```

```ruby
# ข้อที่ 9: Password Generator
class PasswordGenerator
  LOWERCASE = ('a'..'z').to_a
  UPPERCASE = ('A'..'Z').to_a
  DIGITS = ('0'..'9').to_a
  SPECIAL = %w[! @ # $ % ^ & * ( ) - _ = + [ ] { } ; : , . < > ?]

  def initialize(length: 12, uppercase: true, digits: true, special: true)
    @length = length
    @chars = LOWERCASE.dup
    @chars += UPPERCASE if uppercase
    @chars += DIGITS if digits
    @chars += SPECIAL if special
  end

  def generate
    Array.new(@length) { @chars.sample }.join
  end

  def generate_multiple(count)
    Array.new(count) { generate }
  end

  def self.strength(password)
    score = 0
    score += 1 if password.length >= 8
    score += 1 if password.length >= 12
    score += 1 if password =~ /[A-Z]/
    score += 1 if password =~ /[a-z]/
    score += 1 if password =~ /[0-9]/
    score += 1 if password =~ /[^A-Za-z0-9]/

    case score
    when 0..2 then "Weak"
    when 3..4 then "Medium"
    when 5    then "Strong"
    else           "Very Strong"
    end
  end
end

gen = PasswordGenerator.new(length: 16)
password = gen.generate
puts "Password: #{password}"
puts "Strength: #{PasswordGenerator.strength(password)}"

puts "\n5 passwords:"
gen.generate_multiple(5).each_with_index do |p, i|
  puts "#{i+1}. #{p} (#{PasswordGenerator.strength(p)})"
end
```

```ruby
# ข้อที่ 10: Shopping Cart
class ShoppingCart
  class Item
    attr_reader :name, :price, :quantity

    def initialize(name, price, quantity = 1)
      @name = name
      @price = price.to_f
      @quantity = quantity
    end

    def quantity=(qty)
      raise ArgumentError, "Quantity must be positive" unless qty > 0
      @quantity = qty
    end

    def total
      @price * @quantity
    end

    def to_s
      "#{@name} x#{@quantity} @ ฿#{format('%.2f', @price)} = ฿#{format('%.2f', total)}"
    end
  end

  def initialize
    @items = {}
    @discount = 0
  end

  def add(name, price, quantity = 1)
    if @items.key?(name)
      @items[name].quantity += quantity
    else
      @items[name] = Item.new(name, price, quantity)
    end
    puts "Added #{quantity}x #{name}"
    self
  end

  def remove(name)
    if @items.delete(name)
      puts "Removed #{name}"
    else
      puts "#{name} not in cart"
    end
    self
  end

  def apply_discount(percent)
    @discount = percent
    puts "Applied #{percent}% discount"
    self
  end

  def subtotal
    @items.values.sum(&:total)
  end

  def discount_amount
    subtotal * @discount / 100.0
  end

  def total
    subtotal - discount_amount
  end

  def empty?
    @items.empty?
  end

  def item_count
    @items.values.sum(&:quantity)
  end

  def receipt
    puts "\n" + "=" * 45
    puts "       SHOPPING RECEIPT"
    puts "=" * 45
    @items.values.each { |item| puts "  #{item}" }
    puts "-" * 45
    puts format("  %-30s %8.2f", "Subtotal:", subtotal)
    if @discount > 0
      puts format("  %-30s %8.2f", "Discount (#{@discount}%):", -discount_amount)
    end
    puts format("  %-30s %8.2f", "TOTAL:", total)
    puts "=" * 45
    puts "  Items: #{item_count}"
    puts
  end
end

cart = ShoppingCart.new
cart.add("Apple", 15, 3)
    .add("Banana", 10, 5)
    .add("Orange", 20, 2)
    .add("Apple", 15, 2)  # เพิ่ม Apple อีก
    .apply_discount(10)

cart.receipt
```

### ข้อที่ 11-20: ระดับกลาง

```ruby
# ข้อที่ 11: Inventory System
class Inventory
  class Product
    attr_accessor :name, :price, :quantity
    attr_reader :sku

    def initialize(sku, name, price, quantity = 0)
      @sku = sku
      @name = name
      @price = price.to_f
      @quantity = quantity
    end

    def value
      @price * @quantity
    end

    def low_stock?(threshold = 10)
      @quantity <= threshold
    end

    def to_s
      format("%-10s %-25s %8.2f %8d %10.2f", @sku, @name, @price, @quantity, value)
    end
  end

  def initialize
    @products = {}
  end

  def add_product(sku, name, price, quantity = 0)
    @products[sku] = Product.new(sku, name, price, quantity)
    puts "Added product: #{name} (#{sku})"
    self
  end

  def restock(sku, quantity)
    product = find!(sku)
    product.quantity += quantity
    puts "Restocked #{product.name}: +#{quantity} (total: #{product.quantity})"
  end

  def sell(sku, quantity)
    product = find!(sku)
    raise "Insufficient stock for #{product.name}" if product.quantity < quantity
    product.quantity -= quantity
    puts "Sold #{quantity}x #{product.name} (remaining: #{product.quantity})"
  end

  def total_value
    @products.values.sum(&:value)
  end

  def low_stock_products(threshold = 10)
    @products.values.select { |p| p.low_stock?(threshold) }
  end

  def report
    puts "\n" + "=" * 65
    puts format("%-10s %-25s %8s %8s %10s", "SKU", "Name", "Price", "Qty", "Value")
    puts "=" * 65
    @products.values.sort_by(&:sku).each { |p| puts p }
    puts "=" * 65
    puts format("%-45s %10.2f", "Total Inventory Value:", total_value)
    puts

    unless low_stock_products.empty?
      puts "⚠ Low Stock Products:"
      low_stock_products.each { |p| puts "  - #{p.name} (#{p.quantity} remaining)" }
    end
    puts
  end

  private

  def find!(sku)
    @products[sku] || raise("Product not found: #{sku}")
  end
end

inv = Inventory.new
inv.add_product("SKU001", "Laptop", 25000, 50)
   .add_product("SKU002", "Mouse", 500, 100)
   .add_product("SKU003", "Keyboard", 1200, 8)
   .add_product("SKU004", "Monitor", 8500, 25)
   .add_product("SKU005", "USB Hub", 350, 5)

inv.sell("SKU001", 3)
inv.sell("SKU002", 20)
inv.restock("SKU003", 50)

inv.report
```

```ruby
# ข้อที่ 12: Text Statistics
class TextStats
  def initialize(text)
    @text = text
  end

  def word_count
    words.length
  end

  def char_count(include_spaces: true)
    include_spaces ? @text.length : @text.delete(' ').length
  end

  def sentence_count
    @text.split(/[.!?]+/).reject(&:empty?).length
  end

  def paragraph_count
    @text.split(/\n\n+/).reject(&:empty?).length
  end

  def average_word_length
    return 0 if words.empty?
    (words.sum(&:length).to_f / words.length).round(2)
  end

  def most_common_words(n = 10)
    word_frequency.sort_by { |_, count| -count }.first(n)
  end

  def reading_time_minutes(words_per_minute = 200)
    (word_count.to_f / words_per_minute).ceil
  end

  def unique_words
    words.map(&:downcase).uniq.length
  end

  def flesch_reading_ease
    return 0 if sentence_count == 0
    syllable_avg = words.sum { |w| count_syllables(w) }.to_f / words.length
    206.835 - 1.015 * (word_count.to_f / sentence_count) - 84.6 * syllable_avg
  end

  def report
    puts "=== Text Statistics ==="
    puts "Characters (with spaces): #{char_count}"
    puts "Characters (no spaces): #{char_count(include_spaces: false)}"
    puts "Words: #{word_count}"
    puts "Unique words: #{unique_words}"
    puts "Sentences: #{sentence_count}"
    puts "Paragraphs: #{paragraph_count}"
    puts "Avg word length: #{average_word_length} chars"
    puts "Reading time: ~#{reading_time_minutes} min"
    puts "\nTop 5 words:"
    most_common_words(5).each do |word, count|
      puts "  '#{word}': #{count} times"
    end
  end

  private

  def words
    @text.split(/\W+/).reject(&:empty?)
  end

  def word_frequency
    words.map(&:downcase).tally
  end

  def count_syllables(word)
    word = word.downcase
    count = word.scan(/[aeiou]/).length
    count -= word.scan(/[aeiou]{2}/).length
    count -= 1 if word.end_with?('e') && count > 1
    [count, 1].max
  end
end

text = """
Ruby is a dynamic, open source programming language with a focus on simplicity and productivity.
It has an elegant syntax that is natural to read and easy to write.

Ruby was created by Yukihiro Matsumoto in Japan. He blended parts of his favorite languages to create
a new language that balanced functional programming with imperative programming.

Ruby on Rails, or Rails, is a server-side web application framework written in Ruby.
It is a model-view-controller framework providing default structures for a database, web service, and web pages.
"""

stats = TextStats.new(text)
stats.report
```

### ข้อที่ 21-30: ระดับสูง

```ruby
# ข้อที่ 21: Observer Pattern
module Observable
  def self.included(base)
    base.instance_variable_set(:@observers, [])
    base.extend(ClassMethods)
  end

  module ClassMethods
    def observers
      @observers
    end

    def add_observer(observer)
      @observers << observer
    end
  end

  def notify_observers(event, data = nil)
    self.class.observers.each do |observer|
      observer.update(event, data, self) if observer.respond_to?(:update)
    end
  end
end

class EventLogger
  def update(event, data, source)
    puts "[LOG] #{source.class}: #{event} - #{data.inspect}"
  end
end

class EmailNotifier
  def update(event, data, source)
    puts "[EMAIL] Sending notification for event: #{event}"
  end
end

class StockMarket
  include Observable
  attr_reader :symbol, :price

  def initialize(symbol, initial_price)
    @symbol = symbol
    @price = initial_price
  end

  def update_price(new_price)
    old_price = @price
    @price = new_price
    change = ((new_price - old_price) / old_price * 100).round(2)
    notify_observers(:price_changed, { symbol: @symbol, old: old_price, new: new_price, change: change })
  end
end

logger = EventLogger.new
notifier = EmailNotifier.new

StockMarket.add_observer(logger)
StockMarket.add_observer(notifier)

aapl = StockMarket.new("AAPL", 150.0)
aapl.update_price(155.0)
aapl.update_price(148.5)
```

```ruby
# ข้อที่ 22: Memoization
class Fibonacci
  def initialize
    @cache = { 0 => 0, 1 => 1 }
  end

  def calculate(n)
    return @cache[n] if @cache.key?(n)
    @cache[n] = calculate(n - 1) + calculate(n - 2)
  end

  def sequence(n)
    (0..n).map { |i| calculate(i) }
  end
end

fib = Fibonacci.new
puts fib.sequence(15).inspect
puts "F(50) = #{fib.calculate(50)}"
```

```ruby
# ข้อที่ 23: State Machine
class TrafficLight
  STATES = [:red, :yellow, :green]
  TRANSITIONS = {
    red: :green,
    green: :yellow,
    yellow: :red
  }

  attr_reader :state

  def initialize
    @state = :red
    @history = []
  end

  def next_state
    old_state = @state
    @state = TRANSITIONS[@state]
    @history << { from: old_state, to: @state, time: Time.now }
    puts "Traffic light: #{old_state.upcase} → #{@state.upcase}"
    self
  end

  def red?    = @state == :red
  def yellow? = @state == :yellow
  def green?  = @state == :green

  def safe_to_go?
    @state == :green
  end

  def history
    @history.last(5)
  end

  def to_s
    "TrafficLight[#{@state.upcase}]"
  end
end

light = TrafficLight.new
puts light
puts "Safe to go? #{light.safe_to_go?}"
5.times { light.next_state }
puts light
```

```ruby
# ข้อที่ 24: Chess Piece
class ChessPiece
  SYMBOLS = {
    king:   { white: '♔', black: '♚' },
    queen:  { white: '♕', black: '♛' },
    rook:   { white: '♖', black: '♜' },
    bishop: { white: '♗', black: '♝' },
    knight: { white: '♘', black: '♞' },
    pawn:   { white: '♙', black: '♟' }
  }

  attr_reader :type, :color, :position

  def initialize(type, color, position)
    raise ArgumentError, "Invalid piece type" unless SYMBOLS.key?(type)
    raise ArgumentError, "Color must be :white or :black" unless [:white, :black].include?(color)
    @type = type
    @color = color
    @position = position
    @move_count = 0
  end

  def symbol
    SYMBOLS[@type][@color]
  end

  def move_to(new_position)
    if valid_move?(new_position)
      old_pos = @position
      @position = new_position
      @move_count += 1
      puts "#{symbol} moved from #{old_pos} to #{new_position}"
      true
    else
      puts "Invalid move!"
      false
    end
  end

  def file
    @position[0]  # letter (a-h)
  end

  def rank
    @position[1].to_i  # number (1-8)
  end

  def to_s
    "#{@color.to_s.capitalize} #{@type.to_s.capitalize} at #{@position}"
  end

  private

  def valid_move?(pos)
    pos =~ /\A[a-h][1-8]\z/
  end
end

king = ChessPiece.new(:king, :white, "e1")
queen = ChessPiece.new(:queen, :black, "d8")

puts king
puts queen
king.move_to("e2")
king.move_to("e3")
puts king
```

```ruby
# ข้อที่ 25: Polynomial
class Polynomial
  attr_reader :coefficients

  # coefficients: [c0, c1, c2, ...] สำหรับ c0 + c1*x + c2*x^2 + ...
  def initialize(*coefficients)
    @coefficients = coefficients.dup
    # ตัด trailing zeros
    @coefficients.pop while @coefficients.last == 0 && @coefficients.length > 1
  end

  def degree
    @coefficients.length - 1
  end

  def evaluate(x)
    @coefficients.each_with_index.sum { |c, i| c * (x ** i) }
  end

  def +(other)
    max_len = [coefficients.length, other.coefficients.length].max
    new_coeffs = Array.new(max_len, 0)
    coefficients.each_with_index { |c, i| new_coeffs[i] += c }
    other.coefficients.each_with_index { |c, i| new_coeffs[i] += c }
    Polynomial.new(*new_coeffs)
  end

  def *(scalar)
    Polynomial.new(*coefficients.map { |c| c * scalar })
  end

  def derivative
    return Polynomial.new(0) if degree == 0
    new_coeffs = coefficients[1..].each_with_index.map { |c, i| c * (i + 1) }
    Polynomial.new(*new_coeffs)
  end

  def to_s
    return "0" if @coefficients == [0]

    terms = @coefficients.each_with_index.reverse_each.filter_map do |c, i|
      next if c == 0
      if i == 0
        c.to_s
      elsif i == 1
        c == 1 ? "x" : c == -1 ? "-x" : "#{c}x"
      else
        c == 1 ? "x^#{i}" : c == -1 ? "-x^#{i}" : "#{c}x^#{i}"
      end
    end

    terms.join(" + ").gsub("+ -", "- ")
  end
end

p1 = Polynomial.new(1, 2, 3)  # 1 + 2x + 3x^2
p2 = Polynomial.new(4, 1)     # 4 + x
puts "p1 = #{p1}"
puts "p2 = #{p2}"
puts "p1 + p2 = #{p1 + p2}"
puts "p1(2) = #{p1.evaluate(2)}"
puts "p1' = #{p1.derivative}"
```

```ruby
# ข้อที่ 26-30: รวมตัวอย่างเพิ่มเติม

# ข้อที่ 26: Contact Book
class ContactBook
  Contact = Struct.new(:name, :phone, :email, :group) do
    def to_s
      "#{name} | #{phone} | #{email} | #{group}"
    end
  end

  def initialize
    @contacts = []
  end

  def add(name, phone, email, group = "General")
    @contacts << Contact.new(name, phone, email, group)
    puts "Added: #{name}"
    self
  end

  def search(query)
    query = query.downcase
    @contacts.select do |c|
      c.name.downcase.include?(query) ||
      c.phone.include?(query) ||
      c.email.downcase.include?(query)
    end
  end

  def by_group(group)
    @contacts.select { |c| c.group.downcase == group.downcase }
  end

  def delete(name)
    before = @contacts.length
    @contacts.reject! { |c| c.name == name }
    @contacts.length < before ? puts("Deleted: #{name}") : puts("Not found: #{name}")
  end

  def all
    @contacts.sort_by(&:name)
  end

  def list
    puts "\n=== Contact Book (#{@contacts.length} contacts) ==="
    all.each { |c| puts "  #{c}" }
    puts
  end
end

book = ContactBook.new
book.add("Alice Smith", "081-111-1111", "alice@example.com", "Friends")
    .add("Bob Jones", "082-222-2222", "bob@example.com", "Work")
    .add("Charlie Brown", "083-333-3333", "charlie@example.com", "Friends")
    .add("Diana Prince", "084-444-4444", "diana@example.com", "Work")

book.list

puts "Search 'alice':"
book.search("alice").each { |c| puts "  #{c}" }

puts "\nWork contacts:"
book.by_group("Work").each { |c| puts "  #{c}" }
```

```ruby
# ข้อที่ 27: Rate Limiter
class RateLimiter
  def initialize(max_requests:, per_seconds:)
    @max_requests = max_requests
    @per_seconds = per_seconds
    @requests = []
  end

  def allow?
    now = Time.now
    @requests.reject! { |t| t < now - @per_seconds }

    if @requests.length < @max_requests
      @requests << now
      true
    else
      false
    end
  end

  def wait_time
    return 0 unless @requests.length >= @max_requests
    oldest = @requests.first
    [@per_seconds - (Time.now - oldest), 0].max
  end

  def stats
    {
      current_requests: @requests.length,
      max_requests: @max_requests,
      period_seconds: @per_seconds,
      allowed: @requests.length < @max_requests
    }
  end
end

limiter = RateLimiter.new(max_requests: 3, per_seconds: 5)
10.times do |i|
  if limiter.allow?
    puts "Request #{i + 1}: ALLOWED"
  else
    puts "Request #{i + 1}: RATE LIMITED (wait #{limiter.wait_time.round(1)}s)"
  end
end
```

```ruby
# ข้อที่ 28: File-like Object (StringIO-like)
class StringBuffer
  attr_reader :pos

  def initialize(initial = "")
    @buffer = initial.dup
    @pos = 0
  end

  def write(str)
    @buffer[@pos, str.length] = str
    @pos += str.length
    str.length
  end

  def read(n = nil)
    if n.nil?
      result = @buffer[@pos..]
      @pos = @buffer.length
    else
      result = @buffer[@pos, n]
      @pos += (result&.length || 0)
    end
    result
  end

  def seek(offset, whence = :set)
    case whence
    when :set then @pos = offset
    when :cur then @pos += offset
    when :end then @pos = @buffer.length + offset
    end
    @pos = @pos.clamp(0, @buffer.length)
  end

  def rewind
    @pos = 0
  end

  def eof?
    @pos >= @buffer.length
  end

  def size
    @buffer.length
  end

  def string
    @buffer.dup
  end

  def truncate(new_size = 0)
    @buffer = @buffer[0, new_size] || ""
    @pos = @pos.clamp(0, @buffer.length)
  end
end

buf = StringBuffer.new
buf.write("Hello, World!")
buf.rewind
puts buf.read(5)   # => Hello
puts buf.read      # => , World!
puts buf.eof?      # => true
buf.rewind
puts buf.string    # => Hello, World!
```

```ruby
# ข้อที่ 29: Money Class
class Money
  include Comparable

  CURRENCIES = {
    "THB" => "฿",
    "USD" => "$",
    "EUR" => "€",
    "GBP" => "£",
    "JPY" => "¥"
  }

  EXCHANGE_RATES = {
    "USD_THB" => 35.5,
    "EUR_THB" => 38.2,
    "GBP_THB" => 44.1,
    "JPY_THB" => 0.24
  }

  attr_reader :amount, :currency

  def initialize(amount, currency = "THB")
    raise ArgumentError, "Amount cannot be negative" if amount < 0
    raise ArgumentError, "Unknown currency: #{currency}" unless CURRENCIES.key?(currency)
    @amount = amount.to_f.round(2)
    @currency = currency
  end

  def +(other)
    other_in_same = other.convert_to(currency)
    Money.new(@amount + other_in_same.amount, currency)
  end

  def -(other)
    other_in_same = other.convert_to(currency)
    raise "Result cannot be negative" if other_in_same.amount > @amount
    Money.new(@amount - other_in_same.amount, currency)
  end

  def *(factor)
    Money.new(@amount * factor, currency)
  end

  def <=>(other)
    in_thb <=> other.in_thb
  end

  def convert_to(target_currency)
    return self if currency == target_currency
    thb_amount = in_thb
    if target_currency == "THB"
      Money.new(thb_amount, "THB")
    else
      rate_key = "#{target_currency}_THB"
      rate = EXCHANGE_RATES[rate_key]
      raise "No exchange rate for #{target_currency}" unless rate
      Money.new(thb_amount / rate, target_currency)
    end
  end

  def in_thb
    return @amount if currency == "THB"
    rate_key = "#{currency}_THB"
    rate = EXCHANGE_RATES[rate_key]
    raise "No exchange rate for #{currency}" unless rate
    @amount * rate
  end

  def symbol
    CURRENCIES[currency]
  end

  def to_s
    "#{symbol}#{format('%.2f', @amount)} #{currency}"
  end
end

price = Money.new(1000, "THB")
tip = Money.new(10, "USD")
puts "Price: #{price}"
puts "Tip (USD): #{tip}"
puts "Tip in THB: #{tip.convert_to('THB')}"
puts "Total: #{price + tip}"

prices = [
  Money.new(500, "THB"),
  Money.new(20, "USD"),
  Money.new(15, "EUR"),
]
puts "\nSorted by value (THB equivalent):"
prices.sort.each { |p| puts "  #{p} = #{p.convert_to('THB')}" }
```

```ruby
# ข้อที่ 30: Game Character
class GameCharacter
  CLASSES = %i[warrior mage rogue]
  BASE_STATS = {
    warrior: { hp: 150, mp: 50, attack: 25, defense: 20, speed: 10 },
    mage:    { hp: 80, mp: 200, attack: 15, defense: 8, speed: 12 },
    rogue:   { hp: 100, mp: 80, attack: 30, defense: 12, speed: 20 }
  }

  attr_reader :name, :character_class, :level, :experience
  attr_accessor :equipment

  def initialize(name, character_class)
    raise ArgumentError, "Invalid class" unless CLASSES.include?(character_class)
    @name = name
    @character_class = character_class
    @level = 1
    @experience = 0
    @stats = BASE_STATS[character_class].dup
    @current_hp = max_hp
    @current_mp = max_mp
    @equipment = {}
    @skills = []
    @alive = true
  end

  def max_hp = @stats[:hp] + (@level - 1) * 10
  def max_mp = @stats[:mp] + (@level - 1) * 5
  def attack_power = @stats[:attack] + (@level - 1) * 3
  def defense_power = @stats[:defense] + (@level - 1) * 2

  def hp = @current_hp
  def mp = @current_mp
  def alive? = @alive

  def heal(amount)
    old_hp = @current_hp
    @current_hp = [@current_hp + amount, max_hp].min
    healed = @current_hp - old_hp
    puts "#{@name} healed #{healed} HP (#{@current_hp}/#{max_hp})"
  end

  def take_damage(amount)
    damage = [amount - defense_power, 1].max
    @current_hp -= damage
    puts "#{@name} takes #{damage} damage! (#{[@current_hp, 0].max}/#{max_hp} HP)"
    if @current_hp <= 0
      @current_hp = 0
      @alive = false
      puts "#{@name} has been defeated!"
    end
  end

  def attack(target)
    damage = attack_power + rand(-5..5)
    puts "#{@name} attacks #{target.name} for #{damage} damage!"
    target.take_damage(damage)
  end

  def gain_experience(amount)
    @experience += amount
    puts "#{@name} gained #{amount} EXP (total: #{@experience})"
    level_up while @experience >= experience_to_next_level
  end

  def experience_to_next_level
    @level * 100
  end

  def status
    hp_bar = progress_bar(@current_hp, max_hp, 20)
    mp_bar = progress_bar(@current_mp, max_mp, 20)
    puts "=" * 45
    puts "#{@name} (#{@character_class.to_s.capitalize}) - Level #{@level}"
    puts "HP: #{hp_bar} #{@current_hp}/#{max_hp}"
    puts "MP: #{mp_bar} #{@current_mp}/#{max_mp}"
    puts "ATK: #{attack_power}  DEF: #{defense_power}  SPD: #{@stats[:speed]}"
    puts "EXP: #{@experience}/#{experience_to_next_level}"
    puts "=" * 45
  end

  private

  def level_up
    if @experience >= experience_to_next_level
      @level += 1
      @current_hp = max_hp
      @current_mp = max_mp
      puts "🎉 #{@name} leveled up to Level #{@level}!"
    end
  end

  def progress_bar(current, max, width)
    filled = (current.to_f / max * width).round
    "[" + "█" * filled + "░" * (width - filled) + "]"
  end
end

hero = GameCharacter.new("Aragorn", :warrior)
enemy = GameCharacter.new("Goblin", :rogue)

hero.status
enemy.status

puts "\n=== Battle! ==="
hero.attack(enemy)
enemy.attack(hero)
hero.heal(30)
hero.attack(enemy)
enemy.attack(hero)
hero.attack(enemy)

hero.gain_experience(150)
hero.status
```

---

## สรุปตอนที่ 11

ในตอนนี้เราได้เรียนรู้:

1. **OOP Concepts** - Encapsulation, Inheritance, Polymorphism, Abstraction
2. **Class Definition** - `class...end`, naming conventions, open classes
3. **Instance Variables** - `@var`, nil default, `instance_variables`
4. **initialize** - constructor, default values, keyword arguments
5. **Instance Methods** - method chaining กับ `self`
6. **Accessors** - `attr_reader`, `attr_writer`, `attr_accessor`
7. **Class Variables** - `@@var`, shared state
8. **Class Methods** - `self.method`, factory pattern
9. **Constants** - uppercase naming, freezing constants
10. **Object Identity** - `object_id`, `equal?`, `eql?`, `==`
11. **to_s / inspect** - string representation
12. **freeze** - immutability
13. **dup vs clone** - shallow copy, singleton methods
14. **Struct** - lightweight value objects
15. **OpenStruct** - dynamic attributes
16. **Comparable** - `<=>` และ comparison operators
17. **Complete Examples** - BankAccount, Person, Rectangle

---

*ตอนต่อไป: Part 12 - Inheritance*
