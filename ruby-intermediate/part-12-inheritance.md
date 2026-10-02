# Part 12: Inheritance (การสืบทอด) - Steps 231-255

## บทนำ

Inheritance (การสืบทอด) เป็นหนึ่งในหลักการสำคัญของ OOP ที่ช่วยให้เราสามารถสร้าง class ใหม่โดยสืบทอดคุณสมบัติและพฤติกรรมจาก class ที่มีอยู่แล้ว ทำให้ประหยัดเวลาและลดการเขียนโค้ดซ้ำ

---

## Step 231: Single Inheritance ใน Ruby

Ruby รองรับ **Single Inheritance** เท่านั้น หมายความว่า class หนึ่งสามารถสืบทอดจาก parent class ได้เพียงหนึ่ง class

```ruby
# Syntax การสืบทอด
class ChildClass < ParentClass
  # เพิ่มหรือ override methods
end
```

### ตัวอย่างพื้นฐาน

```ruby
class Animal
  attr_reader :name, :age
  
  def initialize(name, age)
    @name = name
    @age = age
  end
  
  def speak
    "..."
  end
  
  def eat
    "#{@name} กำลังกินอาหาร"
  end
  
  def sleep_now
    "#{@name} กำลังนอนหลับ"
  end
  
  def to_s
    "#{self.class.name}: #{@name} (#{@age} ปี)"
  end
end

class Dog < Animal
  attr_reader :breed
  
  def initialize(name, age, breed)
    super(name, age)  # เรียก initialize ของ Animal
    @breed = breed
  end
  
  def speak
    "#{@name}: โฮ่งๆ!"
  end
  
  def fetch(item)
    "#{@name} วิ่งไปเอา #{item} มา!"
  end
end

class Cat < Animal
  def speak
    "#{@name}: เมี้ยวๆ!"
  end
  
  def purr
    "#{@name} กรุ๊งๆ..."
  end
end

dog = Dog.new("บัดดี้", 3, "Golden Retriever")
cat = Cat.new("วิสกี้", 5)

puts dog          # => Dog: บัดดี้ (3 ปี)
puts cat          # => Cat: วิสกี้ (5 ปี)
puts dog.speak    # => บัดดี้: โฮ่งๆ!
puts cat.speak    # => วิสกี้: เมี้ยวๆ!
puts dog.eat      # => บัดดี้ กำลังกินอาหาร (สืบทอดจาก Animal)
puts cat.sleep_now # => วิสกี้ กำลังนอนหลับ (สืบทอดจาก Animal)
puts dog.fetch("ลูกบอล")  # => บัดดี้ วิ่งไปเอา ลูกบอล มา!
puts cat.purr     # => วิสกี้ กรุ๊งๆ...
puts dog.breed    # => Golden Retriever
```

---

## Step 232: super Keyword

`super` เรียก method ที่มีชื่อเดียวกันจาก parent class

```ruby
class Vehicle
  attr_reader :make, :model, :year
  
  def initialize(make, model, year)
    @make = make
    @model = model
    @year = year
    @mileage = 0
  end
  
  def start
    "เครื่องยนต์ติดแล้ว"
  end
  
  def drive(km)
    @mileage += km
    "ขับรถไป #{km} กม. รวม #{@mileage} กม."
  end
  
  def info
    "#{@year} #{@make} #{@model}"
  end
  
  def to_s
    "#{info} (#{@mileage} กม.)"
  end
end

class ElectricVehicle < Vehicle
  attr_reader :battery_capacity
  
  def initialize(make, model, year, battery_capacity)
    super(make, model, year)  # เรียก Vehicle#initialize
    @battery_capacity = battery_capacity
    @battery_level = 100
  end
  
  def start
    # เรียก super แล้วเพิ่มข้อความ
    base = super
    "#{base} (โหมดไฟฟ้า, แบต: #{@battery_level}%)"
  end
  
  def drive(km)
    energy_used = km * 0.2  # 200Wh ต่อกิโลเมตร
    @battery_level = [0, @battery_level - (energy_used / @battery_capacity * 100)].max.round
    
    base_result = super  # เรียก Vehicle#drive
    "#{base_result} (แบตเหลือ #{@battery_level}%)"
  end
  
  def charge(percent)
    @battery_level = [@battery_level + percent, 100].min
    "ชาร์จแบต: #{@battery_level}%"
  end
  
  def info
    "#{super} (EV, #{@battery_capacity}kWh)"  # เรียก Vehicle#info แล้วเพิ่มข้อมูล
  end
end

ev = ElectricVehicle.new("Tesla", "Model 3", 2023, 75)
puts ev.start
puts ev.drive(100)
puts ev.drive(200)
puts ev.charge(50)
puts ev.info
puts ev
```

### super ไม่มี Arguments

```ruby
class Parent
  def greet(name, title = "คุณ")
    "สวัสดี #{title}#{name}"
  end
end

class Child < Parent
  def greet(name, title = "คุณ")
    result = super  # ส่ง arguments เดิมทั้งหมดไปให้ parent
    "#{result} ยินดีต้อนรับ!"
  end
  
  def greet_formal(name)
    super(name, "ท่าน")  # ส่ง arguments ที่กำหนดเอง
  end
end

c = Child.new
puts c.greet("สมชาย")       # => สวัสดี คุณสมชาย ยินดีต้อนรับ!
puts c.greet_formal("ผู้ว่า")  # => สวัสดี ท่านผู้ว่า ยินดีต้อนรับ!
```

---

## Step 233: Method Overriding

```ruby
class Shape
  def area
    raise NotImplementedError, "#{self.class} ต้อง implement method area"
  end
  
  def perimeter
    raise NotImplementedError, "#{self.class} ต้อง implement method perimeter"
  end
  
  def describe
    "รูปร่าง: #{self.class.name}\nพื้นที่: #{area.round(2)}\nเส้นรอบรูป: #{perimeter.round(2)}"
  end
end

class Rectangle < Shape
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
  
  def to_s
    "สี่เหลี่ยม #{@width}x#{@height}"
  end
end

class Circle < Shape
  def initialize(radius)
    @radius = radius.to_f
  end
  
  def area
    Math::PI * @radius ** 2
  end
  
  def perimeter
    2 * Math::PI * @radius
  end
  
  def to_s
    "วงกลมรัศมี #{@radius}"
  end
end

class Triangle < Shape
  def initialize(a, b, c)
    @a, @b, @c = a.to_f, b.to_f, c.to_f
    raise ArgumentError, "สามเหลี่ยมไม่ถูกต้อง" unless valid?
  end
  
  def area
    # Heron's formula
    s = perimeter / 2
    Math.sqrt(s * (s - @a) * (s - @b) * (s - @c))
  end
  
  def perimeter
    @a + @b + @c
  end
  
  def to_s
    "สามเหลี่ยม (#{@a}, #{@b}, #{@c})"
  end
  
  private
  
  def valid?
    @a + @b > @c && @b + @c > @a && @a + @c > @b
  end
end

shapes = [
  Rectangle.new(5, 3),
  Circle.new(4),
  Triangle.new(3, 4, 5)
]

shapes.each do |shape|
  puts shape.describe
  puts "---"
end

# Polymorphism: เรียก method เดียวกันกับ object ต่างชนิด
total_area = shapes.sum(&:area)
puts "พื้นที่รวม: #{total_area.round(2)}"
```

---

## Step 234: Protected และ Private Methods ใน Inheritance

### Protected Methods

Protected methods เรียกได้จาก class เดียวกันและ subclass ทุก instance

```ruby
class Account
  def initialize(balance)
    @balance = balance.to_f
  end
  
  def >(other)
    balance > other.balance  # เรียก protected method ของ other
  end
  
  def transfer_to(other, amount)
    withdraw_internal(amount)
    other.deposit_internal(amount)
    "โอนเงิน #{amount} บาท สำเร็จ"
  end
  
  def to_s
    "บัญชี (ยอด: #{@balance} บาท)"
  end
  
  protected
  
  def balance
    @balance
  end
  
  def deposit_internal(amount)
    @balance += amount
  end
  
  def withdraw_internal(amount)
    raise "เงินไม่พอ" if amount > @balance
    @balance -= amount
  end
end

class SavingsAccount < Account
  INTEREST_RATE = 0.02
  
  def initialize(balance)
    super
    @months = 0
  end
  
  def add_interest
    @months += 1
    interest = balance * INTEREST_RATE  # เรียก protected method ของ parent
    deposit_internal(interest)           # เรียก protected method ของ parent
    "ดอกเบี้ย #{interest.round(2)} บาท"
  end
end

acc1 = Account.new(1000)
acc2 = SavingsAccount.new(5000)

puts acc1.transfer_to(acc2, 500)
puts acc2.add_interest
puts acc2 > acc1  # => true

# acc1.balance  # => NoMethodError (ไม่สามารถเรียก protected จากภายนอก)
```

### Private Methods ใน Inheritance

```ruby
class BaseValidator
  def validate(data)
    errors = []
    errors += validate_required(data)
    errors += validate_format(data)
    errors += custom_validations(data)
    errors
  end
  
  private
  
  def validate_required(data)
    # จะถูก override ใน subclass ไม่ได้ ต้องใช้ protected
    []
  end
  
  def validate_format(data)
    []
  end
  
  protected
  
  def custom_validations(data)
    # Subclass สามารถ override ได้
    []
  end
end

class UserValidator < BaseValidator
  protected
  
  def custom_validations(data)
    errors = []
    errors << "ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร" if data[:username].to_s.length < 3
    errors << "อีเมลไม่ถูกต้อง" unless data[:email].to_s.include?("@")
    errors << "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร" if data[:password].to_s.length < 8
    errors
  end
end

validator = UserValidator.new
data = { username: "ab", email: "notanemail", password: "short" }
errors = validator.validate(data)
errors.each { |e| puts "- #{e}" }
```

---

## Step 235: Abstract Classes Pattern

Ruby ไม่มี abstract class ในตัว แต่เราจำลองได้

```ruby
module Abstract
  def self.included(base)
    base.instance_variable_set(:@abstract_methods, [])
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def abstract_method(*methods)
      @abstract_methods ||= []
      @abstract_methods.concat(methods)
      
      methods.each do |method_name|
        define_method(method_name) do |*args|
          raise NotImplementedError, 
            "#{self.class}##{method_name} ต้องถูก implement"
        end
      end
    end
    
    def abstract_methods
      @abstract_methods || []
    end
  end
end

class Shape
  include Abstract
  
  abstract_method :area, :perimeter
  
  def describe
    "#{self.class.name}: พื้นที่=#{area.round(2)}, เส้นรอบรูป=#{perimeter.round(2)}"
  end
  
  def larger_than?(other)
    area > other.area
  end
end

class Square < Shape
  def initialize(side)
    @side = side.to_f
  end
  
  def area
    @side ** 2
  end
  
  def perimeter
    4 * @side
  end
end

class Ellipse < Shape
  def initialize(a, b)
    @a, @b = a.to_f, b.to_f
  end
  
  def area
    Math::PI * @a * @b
  end
  
  def perimeter
    # Ramanujan approximation
    h = ((@a - @b) / (@a + @b)) ** 2
    Math::PI * (@a + @b) * (1 + 3*h / (10 + Math.sqrt(4 - 3*h)))
  end
end

s = Square.new(5)
e = Ellipse.new(3, 2)

puts s.describe  # => Square: พื้นที่=25.0, เส้นรอบรูป=20.0
puts e.describe

# ทดสอบ abstract
begin
  Shape.new.area
rescue NotImplementedError => err
  puts "Error: #{err.message}"
end
```

---

## Step 236: Template Method Pattern

```ruby
class DataProcessor
  # Template method - กำหนดขั้นตอนหลัก
  def process(data)
    validated = validate(data)
    transformed = transform(validated)
    result = calculate(transformed)
    format_output(result)
  end
  
  private
  
  # Steps ที่ subclass ต้อง implement
  def validate(data)
    raise NotImplementedError, "ต้อง implement validate"
  end
  
  def transform(data)
    data  # default: ไม่เปลี่ยนแปลง
  end
  
  def calculate(data)
    raise NotImplementedError, "ต้อง implement calculate"
  end
  
  def format_output(result)
    result.to_s  # default: to_s
  end
end

class SalesAnalyzer < DataProcessor
  private
  
  def validate(data)
    data.select { |item| item[:amount] > 0 }
  end
  
  def transform(data)
    data.map { |item| item[:amount] }
  end
  
  def calculate(amounts)
    {
      total: amounts.sum,
      average: amounts.sum.to_f / amounts.length,
      max: amounts.max,
      min: amounts.min,
      count: amounts.length
    }
  end
  
  def format_output(result)
    <<~REPORT
      === รายงานยอดขาย ===
      จำนวนรายการ: #{result[:count]}
      ยอดรวม: #{result[:total]} บาท
      เฉลี่ย: #{result[:average].round(2)} บาท
      สูงสุด: #{result[:max]} บาท
      ต่ำสุด: #{result[:min]} บาท
    REPORT
  end
end

class GradeCalculator < DataProcessor
  private
  
  def validate(data)
    data.select { |s| s[:score].between?(0, 100) }
  end
  
  def transform(data)
    data.map { |s| { **s, grade: score_to_grade(s[:score]) } }
  end
  
  def calculate(data)
    {
      students: data,
      avg: data.sum { |s| s[:score] }.to_f / data.length,
      pass_count: data.count { |s| s[:score] >= 50 }
    }
  end
  
  def format_output(result)
    lines = ["=== ผลการเรียน ==="]
    result[:students].each do |s|
      lines << "#{s[:name]}: #{s[:score]} (#{s[:grade]})"
    end
    lines << "เฉลี่ย: #{result[:avg].round(2)}"
    lines << "ผ่าน: #{result[:pass_count]}/#{result[:students].length} คน"
    lines.join("\n")
  end
  
  def score_to_grade(score)
    case score
    when 80..100 then "A"
    when 70..79  then "B"
    when 60..69  then "C"
    when 50..59  then "D"
    else "F"
    end
  end
end

sales_data = [
  { amount: 1500 }, { amount: 2300 }, { amount: -100 },
  { amount: 890 }, { amount: 4500 }
]

analyzer = SalesAnalyzer.new
puts analyzer.process(sales_data)

students = [
  { name: "สมชาย", score: 85 },
  { name: "สมหญิง", score: 72 },
  { name: "สมศักดิ์", score: 45 },
  { name: "สมพร", score: 110 }  # จะถูก validate ออก
]

calc = GradeCalculator.new
puts calc.process(students)
```

---

## Step 237: Class Hierarchy

```ruby
# ตัวอย่าง hierarchy ที่ซับซ้อน
class LivingThing
  def breathe
    "หายใจ"
  end
  
  def grow
    "เติบโต"
  end
end

class Plant < LivingThing
  def photosynthesize
    "สังเคราะห์แสง"
  end
end

class Animal < LivingThing
  def move
    "เคลื่อนที่"
  end
  
  def eat
    "กินอาหาร"
  end
end

class Mammal < Animal
  def nurse_young
    "เลี้ยงลูกด้วยนม"
  end
end

class Dog < Mammal
  def speak
    "โฮ่งๆ!"
  end
  
  def fetch
    "วิ่งไปเอาของมา"
  end
end

class Cat < Mammal
  def speak
    "เมี้ยวๆ!"
  end
  
  def purr
    "กรุ๊งๆ"
  end
end

dog = Dog.new
puts dog.breathe         # สืบทอดจาก LivingThing
puts dog.move           # สืบทอดจาก Animal
puts dog.nurse_young    # สืบทอดจาก Mammal
puts dog.speak          # จาก Dog
puts dog.fetch          # จาก Dog

# แสดง hierarchy
puts "\nDog ancestors:"
puts Dog.ancestors.inspect
```

---

## Step 238: ancestors Chain

```ruby
class A
  def hello
    "Hello from A"
  end
end

class B < A
  def hello
    "Hello from B"
  end
end

class C < B
end

# ดู ancestors chain
puts C.ancestors.inspect
# => [C, B, A, Object, Kernel, BasicObject]

# Method lookup: Ruby ค้นหา method ตาม ancestors chain
c = C.new
puts c.hello  # => Hello from B (เจอที่ B ก่อน)

# C ไม่ override hello
# B override hello
# ดังนั้น C ใช้ hello จาก B

class D < A
  # D ไม่ override hello
end

d = D.new
puts d.hello  # => Hello from A
```

### ตัวอย่างกับ Modules

```ruby
module Walkable
  def walk
    "#{self.class.name} กำลังเดิน"
  end
end

module Swimmable
  def swim
    "#{self.class.name} กำลังว่ายน้ำ"
  end
end

module Flyable
  def fly
    "#{self.class.name} กำลังบิน"
  end
end

class Duck < Animal
  include Walkable
  include Swimmable
  include Flyable
  
  def speak
    "Quack!"
  end
end

duck = Duck.new
puts duck.walk   # => Duck กำลังเดิน
puts duck.swim   # => Duck กำลังว่ายน้ำ
puts duck.fly    # => Duck กำลังบิน
puts duck.speak  # => Quack!
puts duck.eat    # สืบทอดจาก Animal

puts "\nDuck ancestors:"
puts Duck.ancestors.inspect
# => [Duck, Flyable, Swimmable, Walkable, Animal, LivingThing, ...]
```

---

## Step 239: is_a? / kind_of? / instance_of?

```ruby
class Animal; end
class Dog < Animal; end
class Poodle < Dog; end

poodle = Poodle.new

# is_a? และ kind_of? เหมือนกัน - ตรวจทั้ง class และ ancestors
puts poodle.is_a?(Poodle)  # => true
puts poodle.is_a?(Dog)     # => true (ancestor)
puts poodle.is_a?(Animal)  # => true (ancestor)
puts poodle.is_a?(Object)  # => true (ancestor)

puts poodle.kind_of?(Dog)  # => true (เหมือน is_a?)

# instance_of? - ตรวจเฉพาะ class โดยตรง ไม่รวม ancestors
puts poodle.instance_of?(Poodle)  # => true
puts poodle.instance_of?(Dog)     # => false!
puts poodle.instance_of?(Animal)  # => false!

# ใช้ใน type checking
def process_animal(animal)
  if animal.is_a?(Dog)
    puts "#{animal.class.name} เป็นสุนัข"
  elsif animal.is_a?(Animal)
    puts "#{animal.class.name} เป็นสัตว์"
  else
    puts "ไม่รู้จัก"
  end
end

process_animal(Poodle.new)  # => Poodle เป็นสุนัข
process_animal(Dog.new)     # => Dog เป็นสุนัข
process_animal(Animal.new)  # => Animal เป็นสัตว์
```

### respond_to?

```ruby
class Speakable
  def speak
    "พูดได้!"
  end
end

class NotSpeakable; end

obj1 = Speakable.new
obj2 = NotSpeakable.new

def try_speak(obj)
  if obj.respond_to?(:speak)
    obj.speak
  else
    "#{obj.class.name} พูดไม่ได้"
  end
end

puts try_speak(obj1)  # => พูดได้!
puts try_speak(obj2)  # => NotSpeakable พูดไม่ได้
```

---

## Step 240: Building a Shape Hierarchy ตัวอย่างสมบูรณ์

```ruby
require 'json'

# === Shape Hierarchy ===

class Shape
  include Comparable
  
  attr_reader :color, :name
  
  def initialize(color = "white")
    @color = color
    @name = self.class.name
  end
  
  def area
    raise NotImplementedError, "#{self.class}#area ต้อง implement"
  end
  
  def perimeter
    raise NotImplementedError, "#{self.class}#perimeter ต้อง implement"
  end
  
  def <=>(other)
    area <=> other.area
  end
  
  def scale(factor)
    raise NotImplementedError, "#{self.class}#scale ต้อง implement"
  end
  
  def contains_point?(x, y)
    raise NotImplementedError, "#{self.class}#contains_point? ต้อง implement"
  end
  
  def describe
    {
      type: @name,
      color: @color,
      area: area.round(4),
      perimeter: perimeter.round(4)
    }
  end
  
  def to_s
    "#{@name}(color: #{@color}, area: #{area.round(2)})"
  end
  
  def inspect
    "#<#{@name} #{describe.reject { |k,v| k == :type }.map { |k,v| "#{k}=#{v}" }.join(', ')}>"
  end
end

class Rectangle < Shape
  attr_reader :width, :height, :x, :y
  
  def initialize(width, height, x = 0, y = 0, color = "white")
    super(color)
    raise ArgumentError, "ขนาดต้องบวก" if width <= 0 || height <= 0
    @width = width.to_f
    @height = height.to_f
    @x = x.to_f  # ตำแหน่ง top-left
    @y = y.to_f
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
  
  def scale(factor)
    Rectangle.new(@width * factor, @height * factor, @x, @y, @color)
  end
  
  def contains_point?(px, py)
    px.between?(@x, @x + @width) && py.between?(@y, @y + @height)
  end
  
  def intersects?(other)
    return false unless other.respond_to?(:x) && other.respond_to?(:y)
    !(other.x > @x + @width ||
      other.x + other.width < @x ||
      other.y > @y + @height ||
      other.y + other.height < @y)
  end
  
  def describe
    super.merge(width: @width, height: @height, diagonal: diagonal.round(4))
  end
end

class Square < Rectangle
  def initialize(side, x = 0, y = 0, color = "white")
    super(side, side, x, y, color)
  end
  
  def scale(factor)
    Square.new(@width * factor, @x, @y, @color)
  end
  
  def side
    @width
  end
end

class Circle < Shape
  attr_reader :radius, :cx, :cy
  
  def initialize(radius, cx = 0, cy = 0, color = "white")
    super(color)
    raise ArgumentError, "รัศมีต้องบวก" if radius <= 0
    @radius = radius.to_f
    @cx = cx.to_f
    @cy = cy.to_f
  end
  
  def area
    Math::PI * @radius ** 2
  end
  
  def perimeter
    2 * Math::PI * @radius
  end
  
  def diameter
    2 * @radius
  end
  
  def scale(factor)
    Circle.new(@radius * factor, @cx, @cy, @color)
  end
  
  def contains_point?(px, py)
    distance = Math.sqrt((@cx - px)**2 + (@cy - py)**2)
    distance <= @radius
  end
  
  def intersects_circle?(other)
    distance = Math.sqrt((@cx - other.cx)**2 + (@cy - other.cy)**2)
    distance < @radius + other.radius
  end
  
  def describe
    super.merge(radius: @radius, diameter: diameter)
  end
end

class Triangle < Shape
  attr_reader :a, :b, :c
  
  def initialize(a, b, c, color = "white")
    super(color)
    raise ArgumentError, "สามเหลี่ยมไม่ถูกต้อง" unless valid_triangle?(a, b, c)
    @a, @b, @c = a.to_f, b.to_f, c.to_f
  end
  
  def area
    s = perimeter / 2
    Math.sqrt(s * (s - @a) * (s - @b) * (s - @c))
  end
  
  def perimeter
    @a + @b + @c
  end
  
  def type
    if @a == @b && @b == @c
      "ด้านเท่า"
    elsif @a == @b || @b == @c || @a == @c
      "สองด้านเท่า"
    else
      "ด้านไม่เท่า"
    end
  end
  
  def right_triangle?
    sides = [@a, @b, @c].sort
    (sides[0]**2 + sides[1]**2 - sides[2]**2).abs < 0.0001
  end
  
  def scale(factor)
    Triangle.new(@a * factor, @b * factor, @c * factor, @color)
  end
  
  def contains_point?(px, py)
    false  # simplified
  end
  
  def describe
    super.merge(sides: [@a, @b, @c], type: type, right_triangle: right_triangle?)
  end
  
  private
  
  def valid_triangle?(a, b, c)
    a > 0 && b > 0 && c > 0 &&
    a + b > c && b + c > a && a + c > b
  end
end

class RegularPolygon < Shape
  attr_reader :sides, :side_length
  
  def initialize(sides, side_length, color = "white")
    super(color)
    raise ArgumentError, "ต้องมีอย่างน้อย 3 ด้าน" if sides < 3
    raise ArgumentError, "ความยาวด้านต้องบวก" if side_length <= 0
    @sides = sides
    @side_length = side_length.to_f
  end
  
  def area
    (@sides * @side_length**2) / (4 * Math.tan(Math::PI / @sides))
  end
  
  def perimeter
    @sides * @side_length
  end
  
  def interior_angle
    ((@sides - 2) * 180.0) / @sides
  end
  
  def scale(factor)
    RegularPolygon.new(@sides, @side_length * factor, @color)
  end
  
  def contains_point?(px, py)
    false  # simplified
  end
end

# === ทดสอบ Hierarchy ===

puts "=== สร้าง Shapes ==="
rect = Rectangle.new(10, 5, 0, 0, "blue")
square = Square.new(7, 5, 5, "red")
circle = Circle.new(4, 10, 10, "green")
triangle = Triangle.new(3, 4, 5, "yellow")
hexagon = RegularPolygon.new(6, 5, "purple")

shapes = [rect, square, circle, triangle, hexagon]

puts "\n=== รายละเอียด Shapes ==="
shapes.each do |s|
  puts s
  puts "  #{s.describe}"
end

puts "\n=== เรียงตาม Area ==="
shapes.sort.each { |s| puts "  #{s.name}: #{s.area.round(2)}" }

puts "\n=== Polymorphism ==="
shapes.each do |s|
  puts "#{s.class.name} - area: #{s.area.round(2)}, perimeter: #{s.perimeter.round(2)}"
end

puts "\n=== Type Checking ==="
shapes.each do |s|
  puts "#{s.class.name} is_a?(Shape)=#{s.is_a?(Shape)} instance_of?=#{s.instance_of?(s.class)}"
end

puts "\n=== Scale ==="
scaled_rect = rect.scale(2)
puts "ต้นฉบับ: #{rect}"
puts "ขยาย 2x: #{scaled_rect}"

puts "\n=== Triangle Types ==="
puts Triangle.new(5, 5, 5).type   # => ด้านเท่า
puts Triangle.new(5, 5, 3).type   # => สองด้านเท่า
puts Triangle.new(3, 4, 5).type   # => ด้านไม่เท่า
puts "Right triangle? #{Triangle.new(3, 4, 5).right_triangle?}"

puts "\n=== Contains Point? ==="
puts rect.contains_point?(3, 2)   # => true
puts rect.contains_point?(15, 2)  # => false
puts circle.contains_point?(10, 10)  # => true (center)
puts circle.contains_point?(15, 15)  # => false (outside)

puts "\n=== Comparable ==="
puts "ใหญ่ที่สุด: #{shapes.max}"
puts "เล็กที่สุด: #{shapes.min}"
```

---

## Step 241-255: เทคนิคขั้นสูง

### Method Missing

```ruby
class DynamicProxy
  def initialize(target)
    @target = target
  end
  
  def method_missing(method_name, *args, &block)
    if @target.respond_to?(method_name)
      puts "เรียก: #{method_name}(#{args.inspect})"
      result = @target.send(method_name, *args, &block)
      puts "ผลลัพธ์: #{result.inspect}"
      result
    else
      super
    end
  end
  
  def respond_to_missing?(method_name, include_private = false)
    @target.respond_to?(method_name, include_private) || super
  end
end

proxy = DynamicProxy.new([1, 2, 3, 4, 5])
proxy.length
proxy.sum
proxy.select { |n| n > 3 }
```

### class << self (Eigenclass)

```ruby
class Config
  class << self
    attr_accessor :debug, :log_level, :database_url
    
    def setup
      yield self if block_given?
    end
    
    def production?
      !debug
    end
    
    def development?
      debug
    end
  end
  
  # Default values
  @debug = false
  @log_level = :info
  @database_url = "postgresql://localhost/myapp"
end

Config.setup do |c|
  c.debug = true
  c.log_level = :debug
  c.database_url = "postgresql://localhost/myapp_dev"
end

puts Config.debug           # => true
puts Config.log_level       # => debug
puts Config.development?    # => true
```

### Mixin กับ Inheritance

```ruby
module Timestamps
  def self.included(base)
    base.class_eval do
      attr_reader :created_at, :updated_at
    end
  end
  
  def touch
    @updated_at = Time.now
  end
end

module Serializable
  def to_json
    instance_variables.each_with_object({}) do |var, hash|
      key = var.to_s.delete('@')
      value = instance_variable_get(var)
      hash[key] = value
    end.to_json
  end
  
  def to_hash
    instance_variables.each_with_object({}) do |var, hash|
      key = var.to_s.delete('@').to_sym
      hash[key] = instance_variable_get(var)
    end
  end
end

class BaseModel
  include Timestamps
  include Serializable
  
  @@records = {}
  
  def initialize
    @created_at = Time.now
    @updated_at = Time.now
    @id = self.class.next_id
    @@records[@id] = self
  end
  
  def save
    touch
    puts "บันทึก #{self.class.name} ##{@id}"
    self
  end
  
  def self.find(id)
    @@records[id]
  end
  
  def self.all
    @@records.values.select { |r| r.is_a?(self) }
  end
  
  private
  
  def self.next_id
    @counter ||= 0
    @counter += 1
  end
end

class User < BaseModel
  attr_accessor :name, :email
  
  def initialize(name, email)
    super()
    @name = name
    @email = email
  end
  
  def to_s
    "User(#{@name}, #{@email})"
  end
end

class Post < BaseModel
  attr_accessor :title, :content
  
  def initialize(title, content)
    super()
    @title = title
    @content = content
  end
  
  def to_s
    "Post: #{@title}"
  end
end

u1 = User.new("สมชาย", "somchai@example.com")
u1.save

p1 = Post.new("Ruby สนุกมาก", "เนื้อหาโพสต์...")
p1.save

puts u1
puts u1.to_hash.inspect

puts "\nUsers: #{User.all.length}"
puts "Posts: #{Post.all.length}"
```

---

## แบบฝึกหัด Part 12 (25 ข้อ)

### ระดับง่าย (ข้อ 1-8)

**ข้อ 1:** สร้าง hierarchy สำหรับ Vehicle

```ruby
# เฉลย
class Vehicle
  attr_reader :make, :model, :year, :speed
  
  def initialize(make, model, year)
    @make = make
    @model = model
    @year = year
    @speed = 0
    @running = false
  end
  
  def start
    @running = true
    "เครื่องยนต์ #{@make} #{@model} ติดแล้ว"
  end
  
  def stop
    @running = false
    @speed = 0
    "รถหยุดแล้ว"
  end
  
  def accelerate(km_h)
    return "รถยังไม่ติดเครื่อง" unless @running
    @speed = [@speed + km_h, max_speed].min
    "ความเร็ว: #{@speed} km/h"
  end
  
  def max_speed
    200
  end
  
  def to_s
    "#{@year} #{@make} #{@model}"
  end
end

class Car < Vehicle
  attr_reader :num_doors
  
  def initialize(make, model, year, num_doors = 4)
    super(make, model, year)
    @num_doors = num_doors
  end
  
  def max_speed
    250
  end
end

class Truck < Vehicle
  attr_reader :payload_tons
  
  def initialize(make, model, year, payload_tons)
    super(make, model, year)
    @payload_tons = payload_tons
  end
  
  def max_speed
    120  # รถบรรทุกวิ่งช้ากว่า
  end
  
  def load_cargo(tons)
    return "บรรทุกเกิน! (สูงสุด #{@payload_tons} ตัน)" if tons > @payload_tons
    "บรรทุกสินค้า #{tons} ตัน สำเร็จ"
  end
end

class Motorcycle < Vehicle
  attr_reader :engine_cc
  
  def initialize(make, model, year, engine_cc)
    super(make, model, year)
    @engine_cc = engine_cc
  end
  
  def max_speed
    300
  end
  
  def wheelie
    return "ต้องวิ่งก่อน" if @speed < 50
    "ยกล้อหน้า!"
  end
end

car = Car.new("Toyota", "Camry", 2023)
truck = Truck.new("Isuzu", "D-MAX", 2022, 1.5)
moto = Motorcycle.new("Honda", "CBR", 2023, 600)

puts car.start
puts car.accelerate(100)
puts car.accelerate(200)
puts "\n#{truck.start}"
puts truck.load_cargo(1)
puts truck.load_cargo(2)
puts "\n#{moto.start}"
puts moto.accelerate(60)
puts moto.wheelie

puts "\n=== Type Check ==="
[car, truck, moto].each do |v|
  puts "#{v.class}: is_a?(Vehicle)=#{v.is_a?(Vehicle)}"
end
```

**ข้อ 2:** สร้าง Employee hierarchy

```ruby
# เฉลย
class Employee
  attr_reader :name, :id, :hire_date
  attr_accessor :department
  
  @@employee_count = 0
  
  def initialize(name, department)
    @@employee_count += 1
    @id = @@employee_count
    @name = name
    @department = department
    @hire_date = Date.today rescue Time.now
  end
  
  def base_salary
    raise NotImplementedError
  end
  
  def bonus
    0
  end
  
  def total_compensation
    base_salary + bonus
  end
  
  def self.count
    @@employee_count
  end
  
  def to_s
    "#{@name} (#{self.class.name}, #{@department})"
  end
end

class FullTimeEmployee < Employee
  attr_accessor :monthly_salary
  
  def initialize(name, department, monthly_salary)
    super(name, department)
    @monthly_salary = monthly_salary
  end
  
  def base_salary
    @monthly_salary * 12
  end
  
  def bonus
    base_salary * 0.10  # 10% bonus
  end
end

class PartTimeEmployee < Employee
  attr_accessor :hourly_rate, :hours_per_week
  
  def initialize(name, department, hourly_rate, hours_per_week)
    super(name, department)
    @hourly_rate = hourly_rate
    @hours_per_week = hours_per_week
  end
  
  def base_salary
    @hourly_rate * @hours_per_week * 52  # 52 weeks/year
  end
end

class Contractor < Employee
  attr_accessor :daily_rate, :contract_days
  
  def initialize(name, department, daily_rate, contract_days)
    super(name, department)
    @daily_rate = daily_rate
    @contract_days = contract_days
  end
  
  def base_salary
    @daily_rate * @contract_days
  end
end

employees = [
  FullTimeEmployee.new("สมชาย", "Engineering", 50000),
  PartTimeEmployee.new("สมหญิง", "Marketing", 200, 20),
  Contractor.new("สมศักดิ์", "Design", 2000, 60)
]

puts "=== รายชื่อพนักงาน ==="
employees.each do |emp|
  puts "#{emp} - เงินเดือน/ปี: #{emp.base_salary} บาท, รวม: #{emp.total_compensation} บาท"
end

puts "\nจำนวนพนักงานทั้งหมด: #{Employee.count}"
```

**ข้อ 3-8:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 3: สร้าง Food hierarchy (Fruit, Vegetable, Meat, Dairy)
# ข้อ 4: สร้าง MediaPlayer hierarchy (AudioPlayer, VideoPlayer)
# ข้อ 5: สร้าง Transport hierarchy (Land, Air, Sea transport)
# ข้อ 6: สร้าง Instrument hierarchy (String, Wind, Percussion)
# ข้อ 7: สร้าง Database hierarchy (MySQLDB, PostgreSQLDB, MongoDBDB)
# ข้อ 8: สร้าง Logger hierarchy (FileLogger, ConsoleLogger, DatabaseLogger)
```

### ระดับกลาง (ข้อ 9-17)

**ข้อ 9:** สร้าง Game Character Hierarchy

```ruby
# เฉลย
class GameCharacter
  attr_reader :name, :level, :hp, :max_hp
  attr_accessor :experience
  
  def initialize(name, hp)
    @name = name
    @max_hp = hp
    @hp = hp
    @level = 1
    @experience = 0
    @alive = true
  end
  
  def attack(target)
    damage = base_damage
    target.take_damage(damage)
    "#{@name} โจมตี #{target.name} ด้วย #{damage} damage"
  end
  
  def take_damage(damage)
    actual_damage = [damage - defense, 0].max
    @hp = [@hp - actual_damage, 0].max
    @alive = @hp > 0
    "#{@name} รับ #{actual_damage} damage (HP: #{@hp}/#{@max_hp})"
  end
  
  def heal(amount)
    @hp = [@hp + amount, @max_hp].min
    "#{@name} ฟื้นฟู HP: #{@hp}/#{@max_hp}"
  end
  
  def gain_exp(exp)
    @experience += exp
    if @experience >= exp_to_next_level
      level_up
    end
  end
  
  def alive?
    @alive
  end
  
  def to_s
    "#{self.class.name}[#{@name}] Lv.#{@level} HP:#{@hp}/#{@max_hp}"
  end
  
  protected
  
  def base_damage
    10 + @level * 2
  end
  
  def defense
    5 + @level
  end
  
  def exp_to_next_level
    @level * 100
  end
  
  def level_up
    @level += 1
    @max_hp += 10
    @hp = @max_hp
    puts "#{@name} LEVEL UP! Lv.#{@level}"
  end
end

class Warrior < GameCharacter
  def initialize(name)
    super(name, 150)
    @rage = 0
  end
  
  def attack(target)
    result = super
    @rage += 10
    "#{result} (rage: #{@rage})"
  end
  
  def rage_attack(target)
    return "rage ยังไม่พอ (ต้องการ 50)" if @rage < 50
    damage = base_damage * 2 + @rage
    target.take_damage(damage)
    @rage = 0
    "#{@name} โจมตีด้วย RAGE! damage: #{damage}"
  end
  
  protected
  
  def base_damage
    15 + @level * 3  # Warrior แรงกว่า
  end
  
  def defense
    10 + @level * 2  # Warrior แข็งกว่า
  end
end

class Mage < GameCharacter
  def initialize(name)
    super(name, 80)
    @mana = 100
  end
  
  def fireball(target)
    cost = 20
    return "mana ไม่พอ" if @mana < cost
    @mana -= cost
    damage = base_damage * 3
    target.take_damage(damage)
    "#{@name} ใช้ Fireball! damage: #{damage} (mana: #{@mana})"
  end
  
  def heal_self
    cost = 30
    return "mana ไม่พอ" if @mana < cost
    @mana -= cost
    heal(50)
  end
  
  protected
  
  def base_damage
    8 + @level * 2
  end
  
  def defense
    2  # Mage บางมาก
  end
end

warrior = Warrior.new("สมชาย นักรบ")
mage = Mage.new("สมหญิง นักเวท")

puts "=== เริ่มการต่อสู้ ==="
puts warrior
puts mage

puts "\n--- รอบ 1 ---"
puts warrior.attack(mage)
puts mage.fireball(warrior)

puts "\n--- รอบ 2 ---"
puts warrior.attack(mage)
puts warrior.attack(mage)
puts warrior.attack(mage)
puts warrior.attack(mage)
puts warrior.attack(mage)  # rage = 50
puts warrior.rage_attack(mage)

puts "\n=== สถานะ ==="
puts warrior
puts mage
puts "Warrior alive? #{warrior.alive?}"
puts "Mage alive? #{mage.alive?}"
```

**ข้อ 10-17:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 10: สร้าง Animal hierarchy ที่ซับซ้อนขึ้น พร้อม lifecycle
# ข้อ 11: สร้าง Payment hierarchy (CreditCard, Bank, PromptPay, Crypto)
# ข้อ 12: สร้าง Notification hierarchy (Email, SMS, Push, Line)
# ข้อ 13: สร้าง Report hierarchy (PDF, Excel, CSV, HTML reports)
# ข้อ 14: สร้าง AuthProvider hierarchy (Facebook, Google, Twitter, Local)
# ข้อ 15: สร้าง CacheStore hierarchy (Memory, Redis, File, DB cache)
# ข้อ 16: สร้าง Stream hierarchy (FileStream, NetworkStream, MemoryStream)
# ข้อ 17: สร้าง Encryption hierarchy (AES, RSA, Hash algorithms)
```

### ระดับยาก (ข้อ 18-25)

**ข้อ 18:** สร้าง Expression Tree (AST)

```ruby
# เฉลย
class Expression
  def evaluate(context = {})
    raise NotImplementedError
  end
  
  def to_s
    raise NotImplementedError
  end
end

class Number < Expression
  def initialize(value)
    @value = value
  end
  
  def evaluate(context = {})
    @value
  end
  
  def to_s
    @value.to_s
  end
end

class Variable < Expression
  def initialize(name)
    @name = name
  end
  
  def evaluate(context = {})
    raise "Variable #{@name} ไม่ได้กำหนดค่า" unless context.key?(@name)
    context[@name]
  end
  
  def to_s
    @name.to_s
  end
end

class BinaryOp < Expression
  def initialize(left, op, right)
    @left = left
    @op = op
    @right = right
  end
  
  def evaluate(context = {})
    l = @left.evaluate(context)
    r = @right.evaluate(context)
    case @op
    when :+ then l + r
    when :- then l - r
    when :* then l * r
    when :/ then r == 0 ? raise("หารด้วย 0") : l.to_f / r
    when :** then l ** r
    else raise "ไม่รู้จัก operator #{@op}"
    end
  end
  
  def to_s
    "(#{@left} #{@op} #{@right})"
  end
end

class UnaryOp < Expression
  def initialize(op, operand)
    @op = op
    @operand = operand
  end
  
  def evaluate(context = {})
    val = @operand.evaluate(context)
    case @op
    when :- then -val
    when :abs then val.abs
    when :sqrt then Math.sqrt(val)
    end
  end
  
  def to_s
    "#{@op}(#{@operand})"
  end
end

# Build expression: (x + 2) * (y - 1)
x = Variable.new(:x)
y = Variable.new(:y)
two = Number.new(2)
one = Number.new(1)

expr = BinaryOp.new(
  BinaryOp.new(x, :+, two),
  :*,
  BinaryOp.new(y, :-, one)
)

puts "Expression: #{expr}"

context = { x: 3, y: 5 }
puts "เมื่อ x=3, y=5: #{expr.evaluate(context)}"  # => (3+2)*(5-1) = 20

# sqrt(x^2 + y^2)
hypo = UnaryOp.new(
  :sqrt,
  BinaryOp.new(
    BinaryOp.new(x, :**, Number.new(2)),
    :+,
    BinaryOp.new(y, :**, Number.new(2))
  )
)

puts "\nHypotenuse: #{hypo}"
puts "เมื่อ x=3, y=4: #{hypo.evaluate(x: 3, y: 4).round(2)}"  # => 5.0
```

**ข้อ 19-25:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 19: สร้าง DocumentNode hierarchy (HTML-like structure)
# ข้อ 20: สร้าง Command hierarchy สำหรับ Undo/Redo system
# ข้อ 21: สร้าง Visitor pattern กับ Shape hierarchy
# ข้อ 22: สร้าง Strategy pattern กับ Sorting algorithms
# ข้อ 23: สร้าง Decorator pattern
# ข้อ 24: สร้าง Builder pattern สำหรับ complex objects
# ข้อ 25: สร้าง Composite pattern สำหรับ organization hierarchy
```

---

## สรุป Part 12

ใน Part นี้ เราได้เรียนรู้:

1. **Single Inheritance** - `class Child < Parent`
2. **super keyword** - เรียก method ของ parent class
3. **Method Overriding** - redefine methods ใน subclass
4. **Protected Methods** - เข้าถึงได้จาก class และ subclass
5. **Private Methods** - เข้าถึงได้จาก object เดิมเท่านั้น
6. **Abstract Classes Pattern** - จำลอง abstract class ด้วย Ruby
7. **Template Method Pattern** - กำหนดโครงสร้างใน parent ให้ subclass implement
8. **Class Hierarchy** - ความสัมพันธ์แบบ parent-child
9. **ancestors chain** - ลำดับการค้นหา method
10. **is_a? / kind_of? / instance_of?** - ตรวจสอบประเภทของ object

**ต่อไป:** Part 13 - Modules and Mixins
