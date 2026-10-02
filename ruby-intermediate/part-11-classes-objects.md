# Part 11: Classes and Objects (OOP) - Steps 201-230

## บทนำ

Object-Oriented Programming (OOP) เป็นแนวคิดการเขียนโปรแกรมที่จัดระเบียบโค้ดโดยการสร้าง **Objects** ที่รวม **Data** (ข้อมูล) และ **Behavior** (พฤติกรรม) เข้าด้วยกัน Ruby เป็นภาษาที่สนับสนุน OOP อย่างสมบูรณ์แบบ ทุกอย่างใน Ruby เป็น Object

---

## Step 201: Object-Oriented Programming คืออะไร?

### แนวคิดหลักของ OOP

OOP มีหลักการสำคัญ 4 อย่าง:

1. **Encapsulation** - ห่อหุ้มข้อมูลและพฤติกรรมไว้ด้วยกัน
2. **Inheritance** - การสืบทอดคุณสมบัติจาก class แม่
3. **Polymorphism** - ความสามารถในการรับรูปร่างหลายแบบ
4. **Abstraction** - ซ่อนรายละเอียดที่ซับซ้อน แสดงเฉพาะสิ่งที่จำเป็น

### ทำไมต้องใช้ OOP?

```ruby
# แบบไม่ใช้ OOP (Procedural)
name = "สมชาย"
age = 25
salary = 50000

def show_employee_info(name, age, salary)
  puts "ชื่อ: #{name}, อายุ: #{age}, เงินเดือน: #{salary}"
end

def give_raise(salary, percent)
  salary * (1 + percent / 100.0)
end

show_employee_info(name, age, salary)
new_salary = give_raise(salary, 10)
puts "เงินเดือนใหม่: #{new_salary}"

# แบบใช้ OOP
class Employee
  def initialize(name, age, salary)
    @name = name
    @age = age
    @salary = salary
  end
  
  def show_info
    puts "ชื่อ: #{@name}, อายุ: #{@age}, เงินเดือน: #{@salary}"
  end
  
  def give_raise(percent)
    @salary *= (1 + percent / 100.0)
    puts "เงินเดือนใหม่ของ #{@name}: #{@salary}"
  end
end

emp = Employee.new("สมชาย", 25, 50000)
emp.show_info
emp.give_raise(10)
```

### ทุกอย่างใน Ruby เป็น Object

```ruby
# ตัวเลขเป็น Object
puts 42.class        # => Integer
puts 3.14.class      # => Float

# String เป็น Object
puts "hello".class   # => String
puts "hello".upcase  # => HELLO

# Array เป็น Object
puts [1, 2, 3].class # => Array
puts [1, 2, 3].length # => 3

# nil เป็น Object
puts nil.class       # => NilClass
puts nil.nil?        # => true

# true/false เป็น Object
puts true.class      # => TrueClass
puts false.class     # => FalseClass

# แม้แต่ Class ก็เป็น Object!
puts String.class    # => Class
puts Integer.class   # => Class
```

---

## Step 202: Class Definition

### การสร้าง Class พื้นฐาน

```ruby
# การสร้าง Class ใช้คีย์เวิร์ด class
class Dog
  # เนื้อหาของ class
end

# สร้าง object จาก class (instantiation)
my_dog = Dog.new
puts my_dog.class    # => Dog
puts my_dog.is_a?(Dog) # => true
```

### Naming Convention ของ Class

```ruby
# Class ชื่อต้องขึ้นต้นด้วยตัวพิมพ์ใหญ่ (CamelCase)
class MyClass; end
class UserAccount; end
class BankTransfer; end
class HttpRequest; end

# ไม่ถูกต้อง (จะเกิด SyntaxError)
# class myClass; end
# class user_account; end
```

### Class ที่มี Methods

```ruby
class Greeter
  def hello
    puts "สวัสดี!"
  end
  
  def goodbye
    puts "ลาก่อน!"
  end
  
  def greet(name)
    puts "สวัสดี, #{name}!"
  end
end

g = Greeter.new
g.hello
g.goodbye
g.greet("สมชาย")
```

---

## Step 203: Instance Variables (@variable)

Instance variables คือตัวแปรที่เก็บข้อมูลของแต่ละ object โดยเฉพาะ ขึ้นต้นด้วย `@`

```ruby
class Person
  def set_name(name)
    @name = name  # instance variable
  end
  
  def set_age(age)
    @age = age    # instance variable
  end
  
  def introduce
    puts "สวัสดี, ฉันชื่อ #{@name} อายุ #{@age} ปี"
  end
end

person1 = Person.new
person1.set_name("สมชาย")
person1.set_age(25)

person2 = Person.new
person2.set_name("สมหญิง")
person2.set_age(30)

person1.introduce  # => สวัสดี, ฉันชื่อ สมชาย อายุ 25 ปี
person2.introduce  # => สวัสดี, ฉันชื่อ สมหญิง อายุ 30 ปี

# instance variable แต่ละ object แยกกันอิสระ
puts person1.instance_variables  # => [:@name, :@age]
puts person2.instance_variables  # => [:@name, :@age]
```

### Instance Variable ที่ยังไม่ได้กำหนดค่า

```ruby
class Example
  def show_unset
    puts @undefined_var.inspect  # => nil (ไม่ใช่ error)
  end
  
  def show_set
    @value = 42
    puts @value
  end
end

e = Example.new
e.show_unset  # => nil
e.show_set    # => 42
```

---

## Step 204: Instance Methods

Instance methods คือ methods ที่เรียกใช้ผ่าน object

```ruby
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
  
  def divide(a, b)
    return "ไม่สามารถหารด้วย 0 ได้" if b == 0
    a.to_f / b
  end
  
  # Method ที่เรียก method อื่นใน class เดียวกัน
  def power(base, exp)
    result = 1
    exp.times { result = multiply(result, base) }
    result
  end
end

calc = Calculator.new
puts calc.add(5, 3)        # => 8
puts calc.subtract(10, 4)  # => 6
puts calc.multiply(3, 7)   # => 21
puts calc.divide(15, 4)    # => 3.75
puts calc.divide(10, 0)    # => ไม่สามารถหารด้วย 0 ได้
puts calc.power(2, 8)      # => 256
```

### Method ที่ Return Object เดิม (Method Chaining)

```ruby
class StringBuilder
  def initialize
    @content = ""
  end
  
  def append(text)
    @content += text
    self  # return self เพื่อให้ chain ได้
  end
  
  def prepend(text)
    @content = text + @content
    self
  end
  
  def upcase!
    @content.upcase!
    self
  end
  
  def result
    @content
  end
end

builder = StringBuilder.new
result = builder.append("hello").append(" ").append("world").upcase!.result
puts result  # => HELLO WORLD
```

---

## Step 205: initialize Method (Constructor)

`initialize` เป็น method พิเศษที่ถูกเรียกอัตโนมัติเมื่อสร้าง object ใหม่

```ruby
class Car
  def initialize(brand, model, year)
    @brand = brand
    @model = model
    @year = year
    @speed = 0
    puts "สร้างรถ #{brand} #{model} ปี #{year} แล้ว!"
  end
  
  def accelerate(amount)
    @speed += amount
    puts "ความเร็ว: #{@speed} km/h"
  end
  
  def brake(amount)
    @speed = [@speed - amount, 0].max
    puts "ความเร็ว: #{@speed} km/h"
  end
  
  def info
    "#{@year} #{@brand} #{@model}"
  end
end

car1 = Car.new("Toyota", "Camry", 2023)
# => สร้างรถ Toyota Camry ปี 2023 แล้ว!

car1.accelerate(60)  # => ความเร็ว: 60 km/h
car1.accelerate(40)  # => ความเร็ว: 100 km/h
car1.brake(30)       # => ความเร็ว: 70 km/h

puts car1.info       # => 2023 Toyota Camry
```

### Default Parameters ใน initialize

```ruby
class UserProfile
  def initialize(name, age = 0, email = nil, admin = false)
    @name = name
    @age = age
    @email = email
    @admin = admin
  end
  
  def to_s
    "#{@name} (#{@age}) - #{@email || 'ไม่มี email'} - Admin: #{@admin}"
  end
end

u1 = UserProfile.new("สมชาย")
u2 = UserProfile.new("สมหญิง", 25)
u3 = UserProfile.new("สมศักดิ์", 30, "somsak@example.com")
u4 = UserProfile.new("ผู้ดูแล", 35, "admin@example.com", true)

puts u1  # => สมชาย (0) - ไม่มี email - Admin: false
puts u2  # => สมหญิง (25) - ไม่มี email - Admin: false
puts u3  # => สมศักดิ์ (30) - somsak@example.com - Admin: false
puts u4  # => ผู้ดูแล (35) - admin@example.com - Admin: true
```

### initialize ด้วย Hash (Keyword Arguments)

```ruby
class Product
  def initialize(name:, price:, category: "ทั่วไป", in_stock: true)
    @name = name
    @price = price
    @category = category
    @in_stock = in_stock
  end
  
  def details
    status = @in_stock ? "มีสินค้า" : "สินค้าหมด"
    "#{@name} - ราคา: #{@price} บาท - หมวดหมู่: #{@category} - #{status}"
  end
end

p1 = Product.new(name: "MacBook Pro", price: 59900, category: "คอมพิวเตอร์")
p2 = Product.new(name: "iPhone 15", price: 32900, in_stock: false)

puts p1.details
# => MacBook Pro - ราคา: 59900 บาท - หมวดหมู่: คอมพิวเตอร์ - มีสินค้า

puts p2.details
# => iPhone 15 - ราคา: 32900 บาท - หมวดหมู่: ทั่วไป - สินค้าหมด
```

---

## Step 206: attr_reader, attr_writer, attr_accessor

### ปัญหาโดยไม่มี Attribute Methods

```ruby
class Person
  def initialize(name, age)
    @name = name
    @age = age
  end
  
  # ต้องเขียน getter method เอง
  def name
    @name
  end
  
  # ต้องเขียน setter method เอง
  def name=(new_name)
    @name = new_name
  end
  
  def age
    @age
  end
  
  def age=(new_age)
    @age = new_age
  end
end
```

### attr_reader - อ่านค่าได้อย่างเดียว

```ruby
class Circle
  attr_reader :radius, :color
  
  def initialize(radius, color = "red")
    @radius = radius
    @color = color
  end
  
  def area
    Math::PI * @radius ** 2
  end
  
  def circumference
    2 * Math::PI * @radius
  end
end

c = Circle.new(5, "blue")
puts c.radius        # => 5
puts c.color         # => blue
puts c.area.round(2) # => 78.54

# c.radius = 10  # => NoMethodError! ไม่สามารถ set ค่าได้
```

### attr_writer - เขียนค่าได้อย่างเดียว

```ruby
class PasswordManager
  attr_writer :password
  
  def initialize(password)
    @password = password
  end
  
  def authenticate(input)
    input == @password
  end
end

pm = PasswordManager.new("secret123")
puts pm.authenticate("secret123")  # => true
puts pm.authenticate("wrong")      # => false

pm.password = "newpassword456"
puts pm.authenticate("newpassword456")  # => true

# pm.password  # => NoMethodError! ไม่สามารถ get ค่าได้
```

### attr_accessor - อ่านและเขียนได้

```ruby
class Student
  attr_accessor :name, :grade
  attr_reader :student_id  # อ่านอย่างเดียว
  
  @@count = 0
  
  def initialize(name, grade)
    @@count += 1
    @student_id = @@count
    @name = name
    @grade = grade
  end
  
  def self.count
    @@count
  end
  
  def to_s
    "นักเรียน ##{@student_id}: #{@name} - เกรด: #{@grade}"
  end
end

s1 = Student.new("สมชาย", 3.5)
s2 = Student.new("สมหญิง", 3.8)

puts s1          # => นักเรียน #1: สมชาย - เกรด: 3.5
puts s2          # => นักเรียน #2: สมหญิง - เกรด: 3.8

# แก้ไขค่าได้
s1.name = "สมชาย สุขสันต์"
s1.grade = 3.7

puts s1          # => นักเรียน #1: สมชาย สุขสันต์ - เกรด: 3.7
puts Student.count  # => 2
```

### Custom Validation ใน Setter

```ruby
class BankAccount
  attr_reader :balance, :owner
  
  def initialize(owner, initial_balance = 0)
    @owner = owner
    @balance = initial_balance
  end
  
  def balance=(amount)
    raise ArgumentError, "ยอดเงินต้องไม่ติดลบ" if amount < 0
    @balance = amount
  end
  
  def deposit(amount)
    raise ArgumentError, "จำนวนเงินต้องมากกว่า 0" if amount <= 0
    @balance += amount
    puts "ฝากเงิน #{amount} บาท, ยอดคงเหลือ: #{@balance} บาท"
  end
  
  def withdraw(amount)
    raise ArgumentError, "จำนวนเงินต้องมากกว่า 0" if amount <= 0
    raise "ยอดเงินไม่พอ" if amount > @balance
    @balance -= amount
    puts "ถอนเงิน #{amount} บาท, ยอดคงเหลือ: #{@balance} บาท"
  end
end

account = BankAccount.new("สมชาย", 1000)
account.deposit(500)   # => ฝากเงิน 500 บาท, ยอดคงเหลือ: 1500 บาท
account.withdraw(200)  # => ถอนเงิน 200 บาท, ยอดคงเหลือ: 1300 บาท
puts account.balance   # => 1300

begin
  account.balance = -100  # => ArgumentError!
rescue ArgumentError => e
  puts "Error: #{e.message}"
end
```

---

## Step 207: Class Variables (@@variable)

Class variables ถูกแชร์ระหว่าง instances ทุกตัวของ class

```ruby
class Counter
  @@count = 0
  @@instances = []
  
  def initialize(name)
    @@count += 1
    @name = name
    @@instances << self
  end
  
  def self.count
    @@count
  end
  
  def self.all_names
    @@instances.map(&:name)
  end
  
  def name
    @name
  end
  
  def to_s
    "Counter ##{@@count}: #{@name}"
  end
end

c1 = Counter.new("อัน")
c2 = Counter.new("สอง")
c3 = Counter.new("สาม")

puts Counter.count        # => 3
puts Counter.all_names.inspect  # => ["อัน", "สอง", "สาม"]
```

### ข้อควรระวังเรื่อง Class Variables กับ Inheritance

```ruby
class Animal
  @@count = 0
  
  def initialize
    @@count += 1
  end
  
  def self.count
    @@count
  end
end

class Dog < Animal
  # @@count ถูกแชร์กับ Animal!
end

class Cat < Animal
end

Dog.new
Dog.new
Cat.new

puts Animal.count  # => 3 (รวมทั้งหมด!)
puts Dog.count     # => 3 (เหมือนกัน!)

# ทางแก้: ใช้ class instance variable แทน
class AnimalV2
  @count = 0
  
  def initialize
    self.class.increment_count
  end
  
  def self.count
    @count
  end
  
  def self.increment_count
    @count ||= 0
    @count += 1
  end
end

class DogV2 < AnimalV2
  @count = 0
end

class CatV2 < AnimalV2
  @count = 0
end

DogV2.new
DogV2.new
CatV2.new

puts DogV2.count   # => 2
puts CatV2.count   # => 1
```

---

## Step 208: Class Methods (self.method)

Class methods เรียกผ่านชื่อ class โดยตรง ไม่ต้องสร้าง object

```ruby
class MathHelper
  # Class method
  def self.square(n)
    n * n
  end
  
  def self.cube(n)
    n * n * n
  end
  
  def self.factorial(n)
    return 1 if n <= 1
    n * factorial(n - 1)
  end
  
  def self.fibonacci(n)
    return n if n <= 1
    fibonacci(n - 1) + fibonacci(n - 2)
  end
  
  def self.prime?(n)
    return false if n < 2
    (2..Math.sqrt(n)).none? { |i| n % i == 0 }
  end
end

puts MathHelper.square(5)      # => 25
puts MathHelper.cube(3)        # => 27
puts MathHelper.factorial(6)   # => 720
puts MathHelper.fibonacci(10)  # => 55
puts MathHelper.prime?(17)     # => true
puts MathHelper.prime?(15)     # => false
```

### Factory Methods

```ruby
class Date
  attr_reader :year, :month, :day
  
  def initialize(year, month, day)
    @year = year
    @month = month
    @day = day
  end
  
  # Factory methods
  def self.today
    t = Time.now
    new(t.year, t.month, t.day)
  end
  
  def self.from_string(date_string)
    parts = date_string.split("-").map(&:to_i)
    new(parts[0], parts[1], parts[2])
  end
  
  def self.from_array(arr)
    new(arr[0], arr[1], arr[2])
  end
  
  def to_s
    "#{@year}-#{@month.to_s.rjust(2, '0')}-#{@day.to_s.rjust(2, '0')}"
  end
end

d1 = Date.today
d2 = Date.from_string("2024-03-15")
d3 = Date.from_array([2024, 12, 25])

puts d1  # => (วันนี้)
puts d2  # => 2024-03-15
puts d3  # => 2024-12-25
```

### Class Methods กับ Instance Methods รวมกัน

```ruby
class Temperature
  attr_reader :celsius
  
  def initialize(celsius)
    @celsius = celsius.to_f
  end
  
  # Class methods สำหรับสร้าง Temperature จากหน่วยต่างๆ
  def self.from_fahrenheit(f)
    new((f - 32) * 5.0 / 9.0)
  end
  
  def self.from_kelvin(k)
    new(k - 273.15)
  end
  
  # Instance methods สำหรับแปลงหน่วย
  def to_fahrenheit
    (@celsius * 9.0 / 5.0) + 32
  end
  
  def to_kelvin
    @celsius + 273.15
  end
  
  def to_s
    "#{@celsius.round(2)}°C"
  end
  
  def freezing?
    @celsius <= 0
  end
  
  def boiling?
    @celsius >= 100
  end
end

t1 = Temperature.new(100)
puts t1                        # => 100.0°C
puts t1.to_fahrenheit          # => 212.0
puts t1.to_kelvin              # => 373.15
puts t1.boiling?               # => true

t2 = Temperature.from_fahrenheit(32)
puts t2                        # => 0.0°C
puts t2.freezing?              # => true

t3 = Temperature.from_kelvin(300)
puts t3                        # => 26.85°C
```

---

## Step 209: Object Identity

### object_id

```ruby
# object_id - ID ที่ไม่ซ้ำกันของแต่ละ object
a = "hello"
b = "hello"
c = a

puts a.object_id  # => 12345 (ตัวเลขใดก็ตาม)
puts b.object_id  # => 67890 (ต่างกัน!)
puts c.object_id  # => 12345 (เหมือน a)

puts a.object_id == b.object_id  # => false
puts a.object_id == c.object_id  # => true

# Symbol มี object_id เดียวกันเสมอ
puts :hello.object_id == :hello.object_id  # => true

# Integer เล็กๆ มี object_id คงที่
puts 1.object_id   # => 3
puts 2.object_id   # => 5
puts 100.object_id # => 201
```

### equal? - เปรียบเทียบ Identity

```ruby
a = "hello"
b = "hello"
c = a

puts a.equal?(b)  # => false (คนละ object)
puts a.equal?(c)  # => true  (object เดียวกัน)

puts a == b       # => true  (ค่าเท่ากัน)
puts a == c       # => true  (ค่าเท่ากัน)
```

### eql? - เปรียบเทียบค่าและประเภท

```ruby
puts 1.eql?(1)      # => true
puts 1.eql?(1.0)    # => false (int vs float)
puts 1 == 1.0       # => true  (== แปลงประเภทให้)

puts "hello".eql?("hello")  # => true
puts "hello".eql?("HELLO")  # => false
```

### == - เปรียบเทียบค่า

```ruby
class Point
  attr_reader :x, :y
  
  def initialize(x, y)
    @x = x
    @y = y
  end
  
  # Override ==
  def ==(other)
    other.is_a?(Point) && @x == other.x && @y == other.y
  end
  
  # Override eql? (ใช้ใน Hash)
  def eql?(other)
    self == other
  end
  
  # Override hash (ต้องทำเมื่อ override eql?)
  def hash
    [@x, @y].hash
  end
  
  def to_s
    "(#{@x}, #{@y})"
  end
end

p1 = Point.new(1, 2)
p2 = Point.new(1, 2)
p3 = Point.new(3, 4)

puts p1 == p2      # => true
puts p1 == p3      # => false
puts p1.equal?(p2) # => false (คนละ object)

# ใช้ใน Hash
points = { p1 => "จุด A", p2 => "จุด B" }
puts points.length  # => 1 (p1 และ p2 ถือว่าเป็น key เดียวกัน)
```

---

## Step 210: to_s และ inspect

### to_s - แสดงผลแบบ Human-readable

```ruby
class Person
  def initialize(name, age)
    @name = name
    @age = age
  end
  
  def to_s
    "#{@name} (อายุ #{@age} ปี)"
  end
  
  def inspect
    "#<Person name=#{@name.inspect}, age=#{@age}>"
  end
end

p = Person.new("สมชาย", 25)

puts p            # => สมชาย (อายุ 25 ปี)  (เรียก to_s)
puts p.to_s       # => สมชาย (อายุ 25 ปี)
puts p.inspect    # => #<Person name="สมชาย", age=25>
p p               # => #<Person name="สมชาย", age=25>  (p() เรียก inspect)

# String interpolation เรียก to_s อัตโนมัติ
puts "บุคคล: #{p}"  # => บุคคล: สมชาย (อายุ 25 ปี)
```

### Default to_s และ inspect

```ruby
class WithoutToS
  def initialize(value)
    @value = value
  end
end

obj = WithoutToS.new(42)
puts obj.to_s    # => #<WithoutToS:0x0000...> (ค่า default)
puts obj.inspect # => #<WithoutToS:0x0000... @value=42>
```

---

## Step 211: freeze

`freeze` ทำให้ object ไม่สามารถแก้ไขได้

```ruby
# String freeze
str = "hello"
str << " world"
puts str  # => hello world

str.freeze
begin
  str << "!"  # => FrozenError
rescue FrozenError => e
  puts "Error: #{e.message}"
end

puts str.frozen?  # => true

# Integer, Symbol, nil, true, false ถูก freeze อยู่แล้ว
puts 42.frozen?     # => true
puts :symbol.frozen? # => true
puts nil.frozen?    # => true
```

### Freeze กับ Object

```ruby
class Config
  attr_reader :host, :port, :debug
  
  def initialize(host, port, debug = false)
    @host = host
    @port = port
    @debug = debug
  end
  
  def to_s
    "#{@host}:#{@port} (debug: #{@debug})"
  end
end

config = Config.new("localhost", 3000, true)
config.freeze

puts config.frozen?  # => true

# ไม่สามารถแก้ไข instance variable ได้
begin
  config.instance_variable_set(:@host, "example.com")
rescue FrozenError => e
  puts "Error: #{e.message}"
end

puts config  # => localhost:3000 (debug: true)  ยังเหมือนเดิม
```

---

## Step 212: dup vs clone

ทั้งสองสร้าง shallow copy ของ object แต่มีความแตกต่าง

```ruby
class MyObject
  attr_accessor :value, :items
  
  def initialize(value, items = [])
    @value = value
    @items = items
  end
  
  def to_s
    "MyObject(#{@value}, #{@items})"
  end
end

original = MyObject.new(42, [1, 2, 3])
original.freeze

# dup - ไม่สืบทอดสถานะ frozen
duped = original.dup
puts duped.frozen?  # => false
duped.value = 100   # ทำได้

# clone - สืบทอดสถานะ frozen
cloned = original.clone
puts cloned.frozen? # => true
begin
  cloned.value = 100
rescue FrozenError => e
  puts "clone frozen: #{e.message}"
end
```

### Shallow Copy

```ruby
original = MyObject.new(42, [1, 2, 3])
copy = original.dup

# เปลี่ยน @value ของ copy ไม่กระทบ original
copy.value = 999
puts original.value  # => 42
puts copy.value      # => 999

# แต่ @items ชี้ไปที่ Array เดียวกัน!
copy.items << 4
puts original.items.inspect  # => [1, 2, 3, 4]  (เปลี่ยนไปด้วย!)

# Deep copy ทำโดยใช้ Marshal
deep_copy = Marshal.load(Marshal.dump(original))
deep_copy.items << 5
puts original.items.inspect   # => [1, 2, 3, 4]  (ไม่เปลี่ยน)
puts deep_copy.items.inspect  # => [1, 2, 3, 4, 5]
```

---

## Step 213: Struct

Struct เป็นวิธีสร้าง class ง่ายๆ สำหรับเก็บข้อมูล

```ruby
# สร้าง Struct
Point = Struct.new(:x, :y)

p1 = Point.new(1, 2)
puts p1.x  # => 1
puts p1.y  # => 2
puts p1    # => #<struct Point x=1, y=2>

# Struct มี == ในตัว
p2 = Point.new(1, 2)
p3 = Point.new(3, 4)
puts p1 == p2  # => true
puts p1 == p3  # => false

# Struct ใช้งานเหมือน Array และ Hash
puts p1.to_a.inspect     # => [1, 2]
puts p1.to_h.inspect     # => {:x=>1, :y=>2}
puts p1.members.inspect  # => [:x, :y]
```

### Struct พร้อม Methods

```ruby
Person = Struct.new(:name, :age) do
  def adult?
    age >= 18
  end
  
  def greeting
    "สวัสดี, ฉันชื่อ #{name} อายุ #{age} ปี"
  end
  
  def to_s
    "#{name} (#{age})"
  end
end

p = Person.new("สมชาย", 25)
puts p.adult?    # => true
puts p.greeting  # => สวัสดี, ฉันชื่อ สมชาย อายุ 25 ปี
puts p           # => สมชาย (25)
```

### Struct เป็น Value Object

```ruby
Address = Struct.new(:street, :city, :country, keyword_init: true)

home = Address.new(
  street: "123 ถนนสุขุมวิท",
  city: "กรุงเทพฯ",
  country: "ไทย"
)

puts home.city    # => กรุงเทพฯ
puts home.to_h.inspect
# => {:street=>"123 ถนนสุขุมวิท", :city=>"กรุงเทพฯ", :country=>"ไทย"}
```

---

## Step 214: OpenStruct

OpenStruct สร้าง object ที่เพิ่ม attribute ได้ตามต้องการ

```ruby
require 'ostruct'

person = OpenStruct.new(name: "สมชาย", age: 25)
puts person.name  # => สมชาย
puts person.age   # => 25

# เพิ่ม attribute ใหม่ได้ตลอดเวลา
person.email = "somchai@example.com"
person.city = "กรุงเทพฯ"

puts person.email  # => somchai@example.com
puts person.city   # => กรุงเทพฯ

# ตรวจสอบ attribute ที่ไม่มี
puts person.phone.inspect  # => nil (ไม่ใช่ error)

puts person.to_h.inspect
# => {:name=>"สมชาย", :age=>25, :email=>"somchai@example.com", :city=>"กรุงเทพฯ"}
```

### OpenStruct ใช้กับ JSON/API Response

```ruby
require 'ostruct'
require 'json'

# จำลอง API response
api_response = '{
  "user": {
    "id": 1,
    "name": "สมชาย",
    "email": "somchai@example.com",
    "profile": {
      "bio": "นักพัฒนา Ruby",
      "location": "กรุงเทพฯ"
    }
  }
}'

data = JSON.parse(api_response)
user = OpenStruct.new(data["user"])

puts user.name   # => สมชาย
puts user.email  # => somchai@example.com

# Profile เป็น Hash ต้องแปลงเอง
profile = OpenStruct.new(user.profile)
puts profile.bio       # => นักพัฒนา Ruby
puts profile.location  # => กรุงเทพฯ
```

---

## Step 215: Comparable Module กับ Classes

```ruby
class Weight
  include Comparable
  
  attr_reader :value, :unit
  
  CONVERSIONS = {
    "kg" => 1.0,
    "g"  => 0.001,
    "lb" => 0.453592,
    "oz" => 0.0283495
  }
  
  def initialize(value, unit = "kg")
    @value = value.to_f
    @unit = unit
  end
  
  def to_kg
    @value * CONVERSIONS[@unit]
  end
  
  # Comparable ต้องการแค่ <=>
  def <=>(other)
    to_kg <=> other.to_kg
  end
  
  def to_s
    "#{@value} #{@unit}"
  end
end

w1 = Weight.new(1, "kg")
w2 = Weight.new(500, "g")
w3 = Weight.new(2, "kg")
w4 = Weight.new(2.2, "lb")

puts w1 > w2   # => true  (1kg > 500g)
puts w1 == w2  # => false (1000g != 500g... wait)
puts w2 < w1   # => true

weights = [w3, w1, w4, w2]
sorted = weights.sort
sorted.each { |w| puts w }
# => 500 g
# => 2.2 lb
# => 1 kg
# => 2 kg

puts weights.min  # => 500 g
puts weights.max  # => 2 kg

# clamp
w = Weight.new(1.5, "kg")
puts w.clamp(w1, w3)  # อยู่ระหว่าง 1kg ถึง 2kg
```

---

## Step 216: Building a Full Example - Bank Account System

```ruby
# ระบบบัญชีธนาคารสมบูรณ์

class Transaction
  attr_reader :type, :amount, :description, :timestamp, :balance_after
  
  TYPES = [:deposit, :withdrawal, :transfer_in, :transfer_out]
  
  def initialize(type, amount, description, balance_after)
    raise ArgumentError, "ประเภทธุรกรรมไม่ถูกต้อง" unless TYPES.include?(type)
    @type = type
    @amount = amount.to_f
    @description = description
    @timestamp = Time.now
    @balance_after = balance_after.to_f
  end
  
  def to_s
    type_str = case @type
               when :deposit then "ฝากเงิน"
               when :withdrawal then "ถอนเงิน"
               when :transfer_in then "รับโอน"
               when :transfer_out then "โอนออก"
               end
    
    "#{@timestamp.strftime('%Y-%m-%d %H:%M')} | #{type_str} #{@amount} บาท | #{@description} | คงเหลือ: #{@balance_after} บาท"
  end
end

class BankAccount
  include Comparable
  
  attr_reader :account_number, :owner, :balance, :transactions
  
  @@total_accounts = 0
  @@all_accounts = {}
  
  def initialize(owner, initial_balance = 0)
    @@total_accounts += 1
    @account_number = generate_account_number
    @owner = owner
    @balance = initial_balance.to_f
    @transactions = []
    @frozen_status = false
    
    @@all_accounts[@account_number] = self
    
    if initial_balance > 0
      @transactions << Transaction.new(
        :deposit, initial_balance, "ยอดเปิดบัญชี", @balance
      )
    end
  end
  
  def deposit(amount, description = "ฝากเงิน")
    validate_amount(amount)
    check_frozen
    
    @balance += amount
    @transactions << Transaction.new(:deposit, amount, description, @balance)
    puts "ฝากเงินสำเร็จ: +#{amount} บาท, ยอดคงเหลือ: #{@balance} บาท"
    self
  end
  
  def withdraw(amount, description = "ถอนเงิน")
    validate_amount(amount)
    check_frozen
    raise "ยอดเงินไม่เพียงพอ (มี #{@balance} บาท ต้องการ #{amount} บาท)" if amount > @balance
    
    @balance -= amount
    @transactions << Transaction.new(:withdrawal, amount, description, @balance)
    puts "ถอนเงินสำเร็จ: -#{amount} บาท, ยอดคงเหลือ: #{@balance} บาท"
    self
  end
  
  def transfer_to(target_account, amount, description = "โอนเงิน")
    validate_amount(amount)
    check_frozen
    raise "ยอดเงินไม่เพียงพอ" if amount > @balance
    
    @balance -= amount
    @transactions << Transaction.new(
      :transfer_out, amount, 
      "#{description} -> #{target_account.account_number}", @balance
    )
    
    target_account.receive_transfer(self, amount, description)
    puts "โอนเงินสำเร็จ: #{amount} บาท -> บัญชี #{target_account.account_number}"
    self
  end
  
  def receive_transfer(from_account, amount, description)
    @balance += amount
    @transactions << Transaction.new(
      :transfer_in, amount,
      "#{description} <- #{from_account.account_number}", @balance
    )
  end
  
  def freeze_account
    @frozen_status = true
    puts "บัญชี #{@account_number} ถูกระงับแล้ว"
  end
  
  def unfreeze_account
    @frozen_status = false
    puts "บัญชี #{@account_number} ถูกยกเลิกการระงับแล้ว"
  end
  
  def frozen_account?
    @frozen_status
  end
  
  def statement(last_n = nil)
    txns = last_n ? @transactions.last(last_n) : @transactions
    
    puts "\n" + "="*60
    puts "รายการเดินบัญชี"
    puts "บัญชีเลขที่: #{@account_number}"
    puts "เจ้าของ: #{@owner}"
    puts "ยอดคงเหลือปัจจุบัน: #{@balance} บาท"
    puts "-"*60
    
    if txns.empty?
      puts "ไม่มีรายการ"
    else
      txns.each { |t| puts t }
    end
    
    puts "="*60
  end
  
  def <=>(other)
    @balance <=> other.balance
  end
  
  def to_s
    "บัญชี #{@account_number} (#{@owner}) ยอด: #{@balance} บาท"
  end
  
  def inspect
    "#<BankAccount account=#{@account_number}, owner=#{@owner}, balance=#{@balance}>"
  end
  
  def self.total_accounts
    @@total_accounts
  end
  
  def self.find(account_number)
    @@all_accounts[account_number]
  end
  
  def self.all
    @@all_accounts.values
  end
  
  def self.richest
    @@all_accounts.values.max
  end
  
  private
  
  def generate_account_number
    "ACC-#{Time.now.to_i}-#{@@total_accounts.to_s.rjust(4, '0')}"
  end
  
  def validate_amount(amount)
    raise ArgumentError, "จำนวนเงินต้องเป็นตัวเลขบวก" unless amount.is_a?(Numeric) && amount > 0
  end
  
  def check_frozen
    raise "บัญชีถูกระงับการใช้งาน" if @frozen_status
  end
end

# === ทดสอบระบบ ===

puts "=== สร้างบัญชี ==="
acc1 = BankAccount.new("สมชาย สุขสันต์", 10000)
acc2 = BankAccount.new("สมหญิง รักดี", 5000)
acc3 = BankAccount.new("สมศักดิ์ มั่งมี", 50000)

puts "\n=== ธุรกรรม ==="
acc1.deposit(5000, "รับเงินเดือน")
acc1.withdraw(2000, "ค่าเช่าบ้าน")
acc1.transfer_to(acc2, 1000, "ส่งเงินน้องสาว")

acc2.deposit(3000, "รับเงินค่าจ้าง")
acc2.withdraw(500, "ค่าอาหาร")

puts "\n=== บัญชี acc1 ==="
acc1.statement

puts "\n=== บัญชี acc2 ==="
acc2.statement

puts "\n=== สถิติ ==="
puts "จำนวนบัญชีทั้งหมด: #{BankAccount.total_accounts}"
puts "บัญชีที่รวยที่สุด: #{BankAccount.richest}"

puts "\n=== เปรียบเทียบบัญชี ==="
puts acc1 > acc2 ? "#{acc1.owner} รวยกว่า" : "#{acc2.owner} รวยกว่า"

accounts = BankAccount.all
sorted = accounts.sort_by(&:balance).reverse
puts "\nจัดอันดับตามยอดเงิน:"
sorted.each_with_index do |acc, i|
  puts "#{i+1}. #{acc}"
end

puts "\n=== ทดสอบ Freeze ==="
acc1.freeze_account
begin
  acc1.deposit(1000)
rescue RuntimeError => e
  puts "Error: #{e.message}"
end
acc1.unfreeze_account
acc1.deposit(1000, "ฝากหลังปลดระงับ")
```

---

## Step 217-230: เทคนิคเพิ่มเติม

### Protected Methods

```ruby
class Employee
  def initialize(name, salary)
    @name = name
    @salary = salary
  end
  
  def >(other)
    salary > other.salary  # เรียก protected method
  end
  
  def to_s
    "#{@name}: #{@salary} บาท"
  end
  
  protected
  
  def salary
    @salary
  end
end

e1 = Employee.new("สมชาย", 50000)
e2 = Employee.new("สมหญิง", 60000)

puts e1 > e2   # => false
puts e2 > e1   # => true

# e1.salary  # => NoMethodError (ไม่สามารถเรียกจากภายนอก class)
```

### Private Methods

```ruby
class User
  attr_reader :username, :email
  
  def initialize(username, email, password)
    @username = username
    @email = email
    @password_hash = hash_password(password)
  end
  
  def authenticate(password)
    hash_password(password) == @password_hash
  end
  
  def to_s
    "#{@username} <#{@email}>"
  end
  
  private
  
  def hash_password(password)
    # จำลอง hashing
    password.chars.map(&:ord).sum.to_s(16)
  end
end

u = User.new("somchai", "somchai@example.com", "secret123")
puts u.authenticate("secret123")  # => true
puts u.authenticate("wrong")      # => false
puts u  # => somchai <somchai@example.com>

begin
  u.hash_password("test")  # => NoMethodError
rescue NoMethodError => e
  puts "Error: #{e.message}"
end
```

### Method Visibility

```ruby
class Example
  def public_method
    "ใครก็เรียกได้"
  end
  
  protected
  
  def protected_method
    "เรียกได้จาก class เดียวกันและ subclass"
  end
  
  private
  
  def private_method
    "เรียกได้จาก object เดียวกันเท่านั้น"
  end
  
  public
  
  def another_public
    "กลับมา public"
  end
  
  private :another_public  # สามารถเปลี่ยน visibility ทีหลัง
end

e = Example.new
puts e.public_method  # => ใครก็เรียกได้
# e.protected_method  # => NoMethodError
# e.private_method    # => NoMethodError
```

### Singleton Methods (Methods เฉพาะ Object)

```ruby
dog = Object.new

def dog.speak
  "โฮ่ง!"
end

def dog.fetch(item)
  "วิ่งไปเอา #{item} มาแล้ว!"
end

puts dog.speak        # => โฮ่ง!
puts dog.fetch("ลูกบอล")  # => วิ่งไปเอา ลูกบอล มาแล้ว!

cat = Object.new
# cat.speak  # => NoMethodError (cat ไม่มี speak)
```

---

## แบบฝึกหัด Part 11 (30 ข้อ)

### ระดับง่าย (ข้อ 1-10)

**ข้อ 1:** สร้าง class `Rectangle` ที่มี `width` และ `height` พร้อม methods `area`, `perimeter`, `square?`

```ruby
# เฉลย
class Rectangle
  attr_accessor :width, :height
  
  def initialize(width, height)
    @width = width.to_f
    @height = height.to_f
  end
  
  def area
    @width * @height
  end
  
  def perimeter
    2 * (@width + @height)
  end
  
  def square?
    @width == @height
  end
  
  def to_s
    "สี่เหลี่ยมผืนผ้า #{@width}x#{@height}"
  end
end

r1 = Rectangle.new(5, 3)
puts r1.area         # => 15.0
puts r1.perimeter    # => 16.0
puts r1.square?      # => false

r2 = Rectangle.new(4, 4)
puts r2.square?      # => true
```

**ข้อ 2:** สร้าง class `Circle` ที่มี `radius` พร้อม methods `area`, `circumference`, `diameter`

```ruby
# เฉลย
class Circle
  attr_accessor :radius
  
  PI = Math::PI
  
  def initialize(radius)
    raise ArgumentError, "รัศมีต้องมากกว่า 0" if radius <= 0
    @radius = radius.to_f
  end
  
  def area
    PI * @radius ** 2
  end
  
  def circumference
    2 * PI * @radius
  end
  
  def diameter
    2 * @radius
  end
  
  def to_s
    "วงกลมรัศมี #{@radius}"
  end
end

c = Circle.new(7)
puts c.area.round(2)          # => 153.94
puts c.circumference.round(2) # => 43.98
puts c.diameter               # => 14.0
```

**ข้อ 3:** สร้าง class `Stack` ที่ทำงานแบบ Last-In-First-Out พร้อม `push`, `pop`, `peek`, `empty?`, `size`

```ruby
# เฉลย
class Stack
  def initialize
    @data = []
  end
  
  def push(item)
    @data.push(item)
    self
  end
  
  def pop
    raise "Stack is empty!" if empty?
    @data.pop
  end
  
  def peek
    raise "Stack is empty!" if empty?
    @data.last
  end
  
  def empty?
    @data.empty?
  end
  
  def size
    @data.size
  end
  
  def to_s
    "Stack#{@data.inspect}"
  end
end

s = Stack.new
s.push(1).push(2).push(3)
puts s.peek    # => 3
puts s.pop     # => 3
puts s.size    # => 2
puts s.empty?  # => false
```

**ข้อ 4:** สร้าง class `Queue` แบบ FIFO พร้อม `enqueue`, `dequeue`, `front`, `empty?`, `size`

```ruby
# เฉลย
class Queue
  def initialize
    @data = []
  end
  
  def enqueue(item)
    @data.push(item)
    self
  end
  
  def dequeue
    raise "Queue is empty!" if empty?
    @data.shift
  end
  
  def front
    raise "Queue is empty!" if empty?
    @data.first
  end
  
  def empty?
    @data.empty?
  end
  
  def size
    @data.size
  end
  
  def to_s
    "Queue#{@data.inspect}"
  end
end

q = Queue.new
q.enqueue("ลูกค้าที่ 1").enqueue("ลูกค้าที่ 2").enqueue("ลูกค้าที่ 3")
puts q.front    # => ลูกค้าที่ 1
puts q.dequeue  # => ลูกค้าที่ 1
puts q.size     # => 2
```

**ข้อ 5:** สร้าง class `Person` ด้วย `attr_accessor` พร้อม method `greet` ที่รับ Person object อื่น

```ruby
# เฉลย
class Person
  attr_accessor :name, :age, :city
  
  def initialize(name, age, city = "ไม่ระบุ")
    @name = name
    @age = age
    @city = city
  end
  
  def greet(other)
    "สวัสดี #{other.name}! ฉันชื่อ #{@name} มาจาก#{@city}"
  end
  
  def older_than?(other)
    @age > other.age
  end
  
  def to_s
    "#{@name} (#{@age}) จาก#{@city}"
  end
end

p1 = Person.new("สมชาย", 25, "กรุงเทพฯ")
p2 = Person.new("สมหญิง", 30, "เชียงใหม่")

puts p1.greet(p2)       # => สวัสดี สมหญิง! ฉันชื่อ สมชาย มาจากกรุงเทพฯ
puts p2.older_than?(p1) # => true
```

**ข้อ 6:** สร้าง class `Counter` ที่นับจำนวน สามารถ `increment`, `decrement`, `reset`, มี class variable นับจำนวน instance ทั้งหมด

```ruby
# เฉลย
class Counter
  @@instance_count = 0
  
  attr_reader :value, :name
  
  def initialize(name, start = 0)
    @@instance_count += 1
    @name = name
    @value = start
    @min_value = nil
    @max_value = nil
  end
  
  def increment(by = 1)
    @value += by
    self
  end
  
  def decrement(by = 1)
    @value -= by
    self
  end
  
  def reset
    @value = 0
    self
  end
  
  def self.instance_count
    @@instance_count
  end
  
  def to_s
    "#{@name}: #{@value}"
  end
end

c1 = Counter.new("ผู้เข้าชม")
c2 = Counter.new("คลิก", 100)

c1.increment.increment.increment
c2.decrement(10)

puts c1  # => ผู้เข้าชม: 3
puts c2  # => คลิก: 90
puts Counter.instance_count  # => 2
```

**ข้อ 7:** สร้าง class `Product` ด้วย Struct

```ruby
# เฉลย
Product = Struct.new(:name, :price, :category, :quantity) do
  def total_value
    price * quantity
  end
  
  def discount(percent)
    discounted_price = price * (1 - percent / 100.0)
    Product.new(name, discounted_price, category, quantity)
  end
  
  def in_stock?
    quantity > 0
  end
  
  def to_s
    "#{name} - ราคา: #{price} บาท (#{quantity} ชิ้น)"
  end
end

p1 = Product.new("MacBook", 59900, "คอมพิวเตอร์", 5)
p2 = Product.new("iPhone", 32900, "โทรศัพท์", 0)

puts p1                     # => MacBook - ราคา: 59900 บาท (5 ชิ้น)
puts p1.total_value         # => 299500
puts p1.in_stock?           # => true
puts p2.in_stock?           # => false

discounted = p1.discount(10)
puts discounted.price       # => 53910.0
```

**ข้อ 8:** สร้าง class `Library` ที่เก็บ Book objects พร้อม `add_book`, `remove_book`, `find_by_title`, `find_by_author`

```ruby
# เฉลย
class Book
  attr_reader :title, :author, :isbn, :year
  
  def initialize(title, author, isbn, year)
    @title = title
    @author = author
    @isbn = isbn
    @year = year
  end
  
  def to_s
    "\"#{@title}\" โดย #{@author} (#{@year})"
  end
end

class Library
  def initialize(name)
    @name = name
    @books = []
  end
  
  def add_book(book)
    @books << book
    puts "เพิ่มหนังสือ: #{book.title}"
    self
  end
  
  def remove_book(isbn)
    removed = @books.find { |b| b.isbn == isbn }
    if removed
      @books.delete(removed)
      puts "ลบหนังสือ: #{removed.title}"
    else
      puts "ไม่พบหนังสือ ISBN: #{isbn}"
    end
    self
  end
  
  def find_by_title(title)
    @books.select { |b| b.title.downcase.include?(title.downcase) }
  end
  
  def find_by_author(author)
    @books.select { |b| b.author.downcase.include?(author.downcase) }
  end
  
  def total_books
    @books.length
  end
  
  def to_s
    "ห้องสมุด #{@name} มีหนังสือ #{@books.length} เล่ม"
  end
end

lib = Library.new("ห้องสมุดกลาง")
lib.add_book(Book.new("Ruby Programming", "Matz", "978-1-2345", 2020))
    .add_book(Book.new("Rails in Action", "Ryan Bigg", "978-2-3456", 2021))
    .add_book(Book.new("Ruby Metaprogramming", "Paolo", "978-3-4567", 2022))

puts lib
results = lib.find_by_title("ruby")
results.each { |b| puts b }
```

**ข้อ 9:** สร้าง class `ShoppingCart` ด้วย `add_item`, `remove_item`, `total`, `checkout`

```ruby
# เฉลย
CartItem = Struct.new(:name, :price, :quantity) do
  def subtotal
    price * quantity
  end
  
  def to_s
    "#{name} x#{quantity} = #{subtotal} บาท"
  end
end

class ShoppingCart
  attr_reader :items
  
  def initialize
    @items = {}
  end
  
  def add_item(name, price, quantity = 1)
    if @items[name]
      @items[name] = CartItem.new(name, price, @items[name].quantity + quantity)
    else
      @items[name] = CartItem.new(name, price, quantity)
    end
    puts "เพิ่ม #{name} x#{quantity}"
    self
  end
  
  def remove_item(name)
    if @items.delete(name)
      puts "ลบ #{name} ออกแล้ว"
    else
      puts "ไม่พบสินค้า #{name}"
    end
    self
  end
  
  def total
    @items.values.sum(&:subtotal)
  end
  
  def item_count
    @items.values.sum(&:quantity)
  end
  
  def checkout
    puts "\n=== ใบเสร็จ ==="
    @items.values.each { |item| puts item }
    puts "-"*30
    puts "รวมทั้งหมด: #{total} บาท"
    puts "จำนวนสินค้า: #{item_count} ชิ้น"
    @items.clear
    puts "ขอบคุณที่ใช้บริการ!"
  end
  
  def to_s
    "ตะกร้า (#{item_count} ชิ้น, รวม #{total} บาท)"
  end
end

cart = ShoppingCart.new
cart.add_item("Mac Book", 59900)
    .add_item("Magic Mouse", 2900)
    .add_item("AirPods", 7900, 2)
cart.checkout
```

**ข้อ 10:** สร้าง class `Stopwatch` ด้วย `start`, `stop`, `pause`, `resume`, `elapsed_time`

```ruby
# เฉลย
class Stopwatch
  def initialize
    @start_time = nil
    @elapsed = 0
    @running = false
    @laps = []
  end
  
  def start
    raise "Stopwatch is already running" if @running
    @start_time = Time.now
    @running = true
    puts "เริ่มจับเวลา"
    self
  end
  
  def stop
    raise "Stopwatch is not running" unless @running
    @elapsed += Time.now - @start_time
    @running = false
    puts "หยุดจับเวลา: #{elapsed_time.round(3)} วินาที"
    self
  end
  
  def pause
    stop
  end
  
  def resume
    start
  end
  
  def reset
    @start_time = nil
    @elapsed = 0
    @running = false
    @laps = []
    puts "รีเซ็ตแล้ว"
    self
  end
  
  def lap
    raise "Stopwatch is not running" unless @running
    lap_time = elapsed_time
    @laps << lap_time
    puts "Lap #{@laps.length}: #{lap_time.round(3)} วินาที"
    self
  end
  
  def elapsed_time
    if @running
      @elapsed + (Time.now - @start_time)
    else
      @elapsed
    end
  end
  
  def running?
    @running
  end
  
  def to_s
    status = @running ? "กำลังทำงาน" : "หยุด"
    "Stopwatch (#{status}) - #{elapsed_time.round(3)} วินาที"
  end
end

sw = Stopwatch.new
sw.start
sleep(0.1)
sw.lap
sleep(0.1)
sw.stop
puts sw.elapsed_time.round(2)  # => ~0.2
```

### ระดับกลาง (ข้อ 11-20)

**ข้อ 11:** สร้าง class `Matrix` สำหรับเมตริกซ์ 2x2 ด้วย operations `+`, `-`, `*`

```ruby
# เฉลย
class Matrix2x2
  attr_reader :a, :b, :c, :d
  
  # [a b]
  # [c d]
  def initialize(a, b, c, d)
    @a, @b, @c, @d = a.to_f, b.to_f, c.to_f, d.to_f
  end
  
  def +(other)
    Matrix2x2.new(
      @a + other.a, @b + other.b,
      @c + other.c, @d + other.d
    )
  end
  
  def -(other)
    Matrix2x2.new(
      @a - other.a, @b - other.b,
      @c - other.c, @d - other.d
    )
  end
  
  def *(other)
    if other.is_a?(Numeric)
      Matrix2x2.new(@a*other, @b*other, @c*other, @d*other)
    elsif other.is_a?(Matrix2x2)
      Matrix2x2.new(
        @a*other.a + @b*other.c, @a*other.b + @b*other.d,
        @c*other.a + @d*other.c, @c*other.b + @d*other.d
      )
    end
  end
  
  def determinant
    @a * @d - @b * @c
  end
  
  def transpose
    Matrix2x2.new(@a, @c, @b, @d)
  end
  
  def ==(other)
    [@a, @b, @c, @d] == [other.a, other.b, other.c, other.d]
  end
  
  def to_s
    "[#{@a} #{@b}]\n[#{@c} #{@d}]"
  end
end

m1 = Matrix2x2.new(1, 2, 3, 4)
m2 = Matrix2x2.new(5, 6, 7, 8)

puts "m1 + m2:"
puts m1 + m2

puts "\nm1 * m2:"
puts m1 * m2

puts "\ndet(m1) = #{m1.determinant}"  # => -2.0
```

**ข้อ 12:** สร้าง class `Playlist` สำหรับเพลย์ลิสต์เพลง ด้วย `add_song`, `remove_song`, `shuffle`, `next_song`, `duration`

```ruby
# เฉลย
Song = Struct.new(:title, :artist, :duration) do
  def to_s
    "#{title} - #{artist} (#{duration}s)"
  end
end

class Playlist
  include Enumerable
  
  attr_reader :name
  
  def initialize(name)
    @name = name
    @songs = []
    @current_index = 0
  end
  
  def add_song(song)
    @songs << song
    self
  end
  
  def remove_song(title)
    @songs.reject! { |s| s.title == title }
    self
  end
  
  def shuffle!
    @songs.shuffle!
    @current_index = 0
    self
  end
  
  def current_song
    @songs[@current_index]
  end
  
  def next_song
    @current_index = (@current_index + 1) % @songs.length unless @songs.empty?
    current_song
  end
  
  def prev_song
    @current_index = (@current_index - 1) % @songs.length unless @songs.empty?
    current_song
  end
  
  def total_duration
    @songs.sum(&:duration)
  end
  
  def each(&block)
    @songs.each(&block)
  end
  
  def size
    @songs.size
  end
  
  def to_s
    "Playlist: #{@name} (#{@songs.length} เพลง, #{total_duration} วินาที)"
  end
end

playlist = Playlist.new("ฟังตอนเช้า")
playlist.add_song(Song.new("Shape of You", "Ed Sheeran", 235))
         .add_song(Song.new("Blinding Lights", "The Weeknd", 200))
         .add_song(Song.new("Dance Monkey", "Tones and I", 210))

puts playlist
puts "เพลงปัจจุบัน: #{playlist.current_song}"
puts "เพลงถัดไป: #{playlist.next_song}"
puts "เพลงทั้งหมด:"
playlist.each { |s| puts "  #{s}" }
```

**ข้อ 13:** สร้าง `class TodoList` ด้วย `add`, `complete`, `remove`, `pending_tasks`, `completed_tasks`

```ruby
# เฉลย
class Task
  attr_reader :id, :title, :created_at
  attr_accessor :completed, :priority
  
  @@next_id = 1
  
  def initialize(title, priority = :normal)
    @id = @@next_id
    @@next_id += 1
    @title = title
    @priority = priority
    @created_at = Time.now
    @completed = false
    @completed_at = nil
  end
  
  def complete!
    @completed = true
    @completed_at = Time.now
    self
  end
  
  def completed?
    @completed
  end
  
  def to_s
    status = @completed ? "[✓]" : "[ ]"
    "#{status} ##{@id} #{@title} (#{@priority})"
  end
end

class TodoList
  def initialize(name = "My Tasks")
    @name = name
    @tasks = []
  end
  
  def add(title, priority = :normal)
    task = Task.new(title, priority)
    @tasks << task
    puts "เพิ่มงาน: #{title}"
    task
  end
  
  def complete(id)
    task = find_task(id)
    if task
      task.complete!
      puts "เสร็จงาน: #{task.title}"
    else
      puts "ไม่พบงาน ##{id}"
    end
  end
  
  def remove(id)
    task = find_task(id)
    if task
      @tasks.delete(task)
      puts "ลบงาน: #{task.title}"
    end
  end
  
  def pending_tasks
    @tasks.reject(&:completed?)
  end
  
  def completed_tasks
    @tasks.select(&:completed?)
  end
  
  def show
    puts "\n=== #{@name} ==="
    puts "งานที่ค้างอยู่:"
    pending_tasks.each { |t| puts "  #{t}" }
    puts "งานที่เสร็จแล้ว:"
    completed_tasks.each { |t| puts "  #{t}" }
    puts "รวม: #{@tasks.length} งาน (เสร็จ: #{completed_tasks.length}, ค้าง: #{pending_tasks.length})"
  end
  
  private
  
  def find_task(id)
    @tasks.find { |t| t.id == id }
  end
end

todo = TodoList.new("งานประจำวัน")
todo.add("ตรวจสอบอีเมล", :high)
t2 = todo.add("เขียน report")
todo.add("ประชุมทีม", :high)
todo.add("อ่านหนังสือ", :low)

todo.complete(1)
todo.complete(t2.id)
todo.show
```

**ข้อ 14:** สร้าง class `Fraction` สำหรับเศษส่วน ด้วย operations `+`, `-`, `*`, `/`

```ruby
# เฉลย
class Fraction
  include Comparable
  
  attr_reader :numerator, :denominator
  
  def initialize(numerator, denominator = 1)
    raise ArgumentError, "ตัวส่วนต้องไม่เป็น 0" if denominator == 0
    
    sign = denominator < 0 ? -1 : 1
    g = gcd(numerator.abs, denominator.abs)
    
    @numerator = sign * numerator / g
    @denominator = denominator.abs / g
  end
  
  def +(other)
    other = Fraction.new(other) if other.is_a?(Integer)
    Fraction.new(
      @numerator * other.denominator + other.numerator * @denominator,
      @denominator * other.denominator
    )
  end
  
  def -(other)
    other = Fraction.new(other) if other.is_a?(Integer)
    Fraction.new(
      @numerator * other.denominator - other.numerator * @denominator,
      @denominator * other.denominator
    )
  end
  
  def *(other)
    other = Fraction.new(other) if other.is_a?(Integer)
    Fraction.new(@numerator * other.numerator, @denominator * other.denominator)
  end
  
  def /(other)
    other = Fraction.new(other) if other.is_a?(Integer)
    Fraction.new(@numerator * other.denominator, @denominator * other.numerator)
  end
  
  def <=>(other)
    (@numerator * other.denominator) <=> (other.numerator * @denominator)
  end
  
  def ==(other)
    @numerator == other.numerator && @denominator == other.denominator
  end
  
  def to_f
    @numerator.to_f / @denominator
  end
  
  def to_s
    @denominator == 1 ? "#{@numerator}" : "#{@numerator}/#{@denominator}"
  end
  
  private
  
  def gcd(a, b)
    b == 0 ? a : gcd(b, a % b)
  end
end

f1 = Fraction.new(1, 2)
f2 = Fraction.new(1, 3)
f3 = Fraction.new(3, 4)

puts f1 + f2   # => 5/6
puts f1 - f2   # => 1/6
puts f1 * f2   # => 1/6
puts f1 / f2   # => 3/2
puts f1 > f2   # => true

fractions = [f3, f1, f2]
puts fractions.sort.map(&:to_s).inspect  # => ["1/3", "1/2", "3/4"]
```

**ข้อ 15:** สร้าง class `EventEmitter` (Observer pattern)

```ruby
# เฉลย
class EventEmitter
  def initialize
    @listeners = Hash.new { |h, k| h[k] = [] }
  end
  
  def on(event, &block)
    @listeners[event] << block
    self
  end
  
  def emit(event, *args)
    @listeners[event].each { |listener| listener.call(*args) }
    self
  end
  
  def off(event)
    @listeners.delete(event)
    self
  end
  
  def once(event, &block)
    wrapper = nil
    wrapper = ->(args) do
      block.call(*args)
      off_one(event, wrapper)
    end
    @listeners[event] << wrapper
    self
  end
  
  def listeners_count(event)
    @listeners[event].length
  end
  
  private
  
  def off_one(event, block)
    @listeners[event].delete(block)
  end
end

emitter = EventEmitter.new

emitter.on(:data) { |data| puts "ได้รับข้อมูล: #{data}" }
emitter.on(:data) { |data| puts "Log: #{data}" }
emitter.on(:error) { |err| puts "Error: #{err}" }

emitter.emit(:data, "Hello World")
emitter.emit(:error, "Connection failed")

puts "\nจำนวน listeners: #{emitter.listeners_count(:data)}"
emitter.off(:data)
puts "หลัง off: #{emitter.listeners_count(:data)}"
```

### ระดับยาก (ข้อ 16-25)

**ข้อ 16:** สร้าง class `Graph` สำหรับ undirected graph ด้วย BFS/DFS

```ruby
# เฉลย
class Graph
  def initialize
    @adjacency = Hash.new { |h, k| h[k] = [] }
  end
  
  def add_edge(from, to)
    @adjacency[from] << to unless @adjacency[from].include?(to)
    @adjacency[to] << from unless @adjacency[to].include?(from)
    self
  end
  
  def add_vertex(vertex)
    @adjacency[vertex] unless @adjacency.key?(vertex)
    self
  end
  
  def vertices
    @adjacency.keys
  end
  
  def neighbors(vertex)
    @adjacency[vertex]
  end
  
  def bfs(start)
    visited = []
    queue = [start]
    
    while !queue.empty?
      vertex = queue.shift
      next if visited.include?(vertex)
      
      visited << vertex
      @adjacency[vertex].each do |neighbor|
        queue << neighbor unless visited.include?(neighbor)
      end
    end
    
    visited
  end
  
  def dfs(start, visited = [])
    return visited if visited.include?(start)
    
    visited << start
    @adjacency[start].each do |neighbor|
      dfs(neighbor, visited)
    end
    
    visited
  end
  
  def connected?(v1, v2)
    bfs(v1).include?(v2)
  end
  
  def to_s
    @adjacency.map { |v, neighbors| "#{v} -> #{neighbors.join(', ')}" }.join("\n")
  end
end

g = Graph.new
g.add_edge("A", "B")
 .add_edge("A", "C")
 .add_edge("B", "D")
 .add_edge("C", "D")
 .add_edge("D", "E")

puts "BFS จาก A: #{g.bfs('A').inspect}"
puts "DFS จาก A: #{g.dfs('A').inspect}"
puts "A connected to E? #{g.connected?('A', 'E')}"
puts "\nGraph:\n#{g}"
```

**ข้อ 17:** สร้าง class `Cache` ด้วย LRU (Least Recently Used) eviction

```ruby
# เฉลย
class LRUCache
  def initialize(capacity)
    @capacity = capacity
    @cache = {}
    @order = []  # LRU order
  end
  
  def get(key)
    return nil unless @cache.key?(key)
    
    # อัปเดต access order
    @order.delete(key)
    @order.push(key)
    
    @cache[key]
  end
  
  def put(key, value)
    if @cache.key?(key)
      @order.delete(key)
    elsif @cache.size >= @capacity
      # Evict least recently used
      lru_key = @order.shift
      @cache.delete(lru_key)
      puts "Evicted: #{lru_key}"
    end
    
    @cache[key] = value
    @order.push(key)
    self
  end
  
  def size
    @cache.size
  end
  
  def to_s
    "LRUCache(#{@cache.inspect})"
  end
end

cache = LRUCache.new(3)
cache.put("a", 1)
cache.put("b", 2)
cache.put("c", 3)

puts cache.get("a")   # => 1 (a is most recently used)
cache.put("d", 4)     # => Evicted: b (b was least recently used)

puts cache.get("b").inspect  # => nil (evicted)
puts cache.get("c")          # => 3
puts cache
```

**ข้อ 18:** สร้าง class `StateMachine` สำหรับ traffic light

```ruby
# เฉลย
class StateMachine
  class InvalidTransition < StandardError; end
  
  attr_reader :current_state
  
  def initialize(initial_state)
    @current_state = initial_state
    @transitions = {}
    @callbacks = {}
  end
  
  def add_transition(from, event, to)
    @transitions[[from, event]] = to
    self
  end
  
  def on_transition(from, to, &block)
    @callbacks[[from, to]] = block
    self
  end
  
  def trigger(event)
    key = [@current_state, event]
    new_state = @transitions[key]
    
    unless new_state
      raise InvalidTransition, "ไม่สามารถเปลี่ยนจาก #{@current_state} ด้วย #{event}"
    end
    
    old_state = @current_state
    @current_state = new_state
    
    callback = @callbacks[[old_state, new_state]]
    callback.call(old_state, new_state) if callback
    
    puts "#{old_state} -> #{new_state} (#{event})"
    self
  end
  
  def can_trigger?(event)
    @transitions.key?([@current_state, event])
  end
end

# Traffic Light
light = StateMachine.new(:red)
light.add_transition(:red, :go, :green)
     .add_transition(:green, :slow, :yellow)
     .add_transition(:yellow, :stop, :red)

light.on_transition(:red, :green) { puts "เขียว! ไปได้เลย" }
light.on_transition(:green, :yellow) { puts "เหลือง! เตรียมหยุด" }
light.on_transition(:yellow, :red) { puts "แดง! หยุด" }

puts "สถานะเริ่มต้น: #{light.current_state}"
light.trigger(:go)
light.trigger(:slow)
light.trigger(:stop)
puts "สถานะสุดท้าย: #{light.current_state}"

begin
  light.trigger(:slow)  # ไม่สามารถทำได้จาก red
rescue StateMachine::InvalidTransition => e
  puts "Error: #{e.message}"
end
```

**ข้อ 19:** สร้าง class `RPN_Calculator` (Reverse Polish Notation)

```ruby
# เฉลย
class RPNCalculator
  class CalculatorError < StandardError; end
  
  OPERATIONS = {
    "+" => ->(a, b) { a + b },
    "-" => ->(a, b) { a - b },
    "*" => ->(a, b) { a * b },
    "/" => ->(a, b) { raise CalculatorError, "หารด้วย 0" if b == 0; a.to_f / b },
    "**" => ->(a, b) { a ** b },
    "%" => ->(a, b) { a % b }
  }
  
  def initialize
    @stack = []
    @history = []
  end
  
  def calculate(expression)
    @stack = []
    tokens = expression.split
    
    tokens.each do |token|
      if OPERATIONS.key?(token)
        raise CalculatorError, "ต้องการตัวเลขอย่างน้อย 2 ตัว" if @stack.size < 2
        b = @stack.pop
        a = @stack.pop
        result = OPERATIONS[token].call(a, b)
        @stack.push(result)
      else
        @stack.push(token.to_f)
      end
    end
    
    raise CalculatorError, "Expression ไม่ถูกต้อง" if @stack.size != 1
    
    result = @stack.first
    @history << { expression: expression, result: result }
    result
  end
  
  def history
    @history.map { |h| "#{h[:expression]} = #{h[:result]}" }
  end
  
  def last_result
    @history.last&.[](:result)
  end
end

calc = RPNCalculator.new

puts calc.calculate("3 4 +")       # => 7.0   (3+4)
puts calc.calculate("10 2 /")      # => 5.0   (10/2)
puts calc.calculate("5 1 2 + 4 * + 3 -")  # => 14.0  (5 + (1+2)*4 - 3)
puts calc.calculate("2 3 **")      # => 8.0   (2^3)

puts "\nประวัติการคำนวณ:"
calc.history.each { |h| puts "  #{h}" }
```

**ข้อ 20:** สร้าง class `TreeNode` และ `BinaryTree`

```ruby
# เฉลย
class TreeNode
  attr_accessor :value, :left, :right
  
  def initialize(value)
    @value = value
    @left = nil
    @right = nil
  end
  
  def leaf?
    @left.nil? && @right.nil?
  end
  
  def to_s
    @value.to_s
  end
end

class BinarySearchTree
  def initialize
    @root = nil
  end
  
  def insert(value)
    @root = insert_node(@root, value)
    self
  end
  
  def include?(value)
    find_node(@root, value) != nil
  end
  
  def inorder
    result = []
    inorder_traverse(@root, result)
    result
  end
  
  def preorder
    result = []
    preorder_traverse(@root, result)
    result
  end
  
  def height
    calculate_height(@root)
  end
  
  def min
    return nil if @root.nil?
    node = @root
    node = node.left while node.left
    node.value
  end
  
  def max
    return nil if @root.nil?
    node = @root
    node = node.right while node.right
    node.value
  end
  
  private
  
  def insert_node(node, value)
    return TreeNode.new(value) if node.nil?
    
    if value < node.value
      node.left = insert_node(node.left, value)
    elsif value > node.value
      node.right = insert_node(node.right, value)
    end
    
    node
  end
  
  def find_node(node, value)
    return nil if node.nil?
    return node if node.value == value
    
    if value < node.value
      find_node(node.left, value)
    else
      find_node(node.right, value)
    end
  end
  
  def inorder_traverse(node, result)
    return if node.nil?
    inorder_traverse(node.left, result)
    result << node.value
    inorder_traverse(node.right, result)
  end
  
  def preorder_traverse(node, result)
    return if node.nil?
    result << node.value
    preorder_traverse(node.left, result)
    preorder_traverse(node.right, result)
  end
  
  def calculate_height(node)
    return 0 if node.nil?
    1 + [calculate_height(node.left), calculate_height(node.right)].max
  end
end

bst = BinarySearchTree.new
[5, 3, 7, 1, 4, 6, 8].each { |v| bst.insert(v) }

puts "Inorder: #{bst.inorder.inspect}"   # => [1, 3, 4, 5, 6, 7, 8]
puts "Min: #{bst.min}"                   # => 1
puts "Max: #{bst.max}"                   # => 8
puts "Height: #{bst.height}"             # => 3
puts "Include 4? #{bst.include?(4)}"     # => true
puts "Include 9? #{bst.include?(9)}"     # => false
```

### ระดับท้าทาย (ข้อ 21-30)

**ข้อ 21-30:** โจทย์ท้าทาย

```ruby
# ข้อ 21: สร้าง class Vector ใน 3D space
class Vector3D
  include Comparable
  
  attr_reader :x, :y, :z
  
  def initialize(x, y, z)
    @x, @y, @z = x.to_f, y.to_f, z.to_f
  end
  
  def +(other)
    Vector3D.new(@x + other.x, @y + other.y, @z + other.z)
  end
  
  def -(other)
    Vector3D.new(@x - other.x, @y - other.y, @z - other.z)
  end
  
  def *(scalar)
    Vector3D.new(@x * scalar, @y * scalar, @z * scalar)
  end
  
  def dot(other)
    @x * other.x + @y * other.y + @z * other.z
  end
  
  def cross(other)
    Vector3D.new(
      @y * other.z - @z * other.y,
      @z * other.x - @x * other.z,
      @x * other.y - @y * other.x
    )
  end
  
  def magnitude
    Math.sqrt(@x**2 + @y**2 + @z**2)
  end
  
  def normalize
    m = magnitude
    raise "Zero vector" if m == 0
    Vector3D.new(@x/m, @y/m, @z/m)
  end
  
  def angle_with(other)
    cos_angle = dot(other) / (magnitude * other.magnitude)
    Math.acos(cos_angle) * 180 / Math::PI
  end
  
  def <=>(other)
    magnitude <=> other.magnitude
  end
  
  def ==(other)
    [@x, @y, @z] == [other.x, other.y, other.z]
  end
  
  def to_s
    "(#{@x.round(2)}, #{@y.round(2)}, #{@z.round(2)})"
  end
end

v1 = Vector3D.new(1, 2, 3)
v2 = Vector3D.new(4, 5, 6)

puts v1 + v2              # => (5.0, 7.0, 9.0)
puts v1.dot(v2)           # => 32.0
puts v1.cross(v2)         # => (-3.0, 6.0, -3.0)
puts v1.magnitude.round(3) # => 3.742
puts v1.angle_with(v2).round(2)  # => 12.93 degrees


# ข้อ 22-30 เป็นโจทย์ให้ทำด้วยตัวเอง:
# ข้อ 22: สร้าง Polynomial class ที่คำนวณค่า polynomial ได้
# ข้อ 23: สร้าง LinkedList class ด้วย add, remove, reverse
# ข้อ 24: สร้าง PriorityQueue class
# ข้อ 25: สร้าง Observable pattern ด้วย Subject และ Observer
# ข้อ 26: สร้าง Money class ที่จัดการ currency
# ข้อ 27: สร้าง Calendar class
# ข้อ 28: สร้าง FileSystem simulation ด้วย tree structure
# ข้อ 29: สร้าง ChessBoard class
# ข้อ 30: สร้าง MiniDatabase class ด้วย CRUD operations
```

---

## สรุป Part 11

ใน Part นี้ เราได้เรียนรู้:

1. **OOP คืออะไร** - แนวคิดหลัก 4 อย่าง: Encapsulation, Inheritance, Polymorphism, Abstraction
2. **Class Definition** - การสร้าง class ด้วยคีย์เวิร์ด `class`
3. **Instance Variables** (`@variable`) - เก็บข้อมูลเฉพาะของแต่ละ object
4. **Instance Methods** - ฟังก์ชันที่เรียกผ่าน object
5. **initialize** - constructor ที่ถูกเรียกอัตโนมัติเมื่อสร้าง object
6. **attr_reader/writer/accessor** - ช่วยสร้าง getter/setter อัตโนมัติ
7. **Class Variables** (`@@variable`) - ตัวแปรที่แชร์ระหว่าง instances
8. **Class Methods** (`self.method`) - methods ที่เรียกผ่านชื่อ class
9. **Object Identity** - `object_id`, `equal?`, `eql?`, `==`
10. **to_s, inspect, freeze** - methods พิเศษของ Object
11. **dup vs clone** - shallow copy
12. **Struct** - วิธีสร้าง class ข้อมูลอย่างง่าย
13. **OpenStruct** - object ที่เพิ่ม attribute ได้แบบ dynamic
14. **Comparable** - module สำหรับเปรียบเทียบ objects

**ต่อไป:** Part 12 - Inheritance (การสืบทอด)
