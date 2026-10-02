# ตอนที่ 12: Inheritance (Steps 231-255)

## บทนำ

Inheritance (การสืบทอด) เป็นหนึ่งในแนวคิดหลักของ OOP ที่ช่วยให้เราสร้าง class ใหม่โดยอิงจาก class ที่มีอยู่แล้ว ช่วยลดการเขียนโค้ดซ้ำและทำให้โปรแกรมมีโครงสร้างที่ดี

---

## Step 231: Single Inheritance (class Child < Parent)

Ruby รองรับ Single Inheritance เท่านั้น หมายความว่า class หนึ่งสืบทอดจาก parent class ได้เพียงหนึ่ง class เท่านั้น

```ruby
# Parent class (Base class / Superclass)
class Animal
  attr_reader :name, :age

  def initialize(name, age)
    @name = name
    @age = age
  end

  def breathe
    puts "#{@name} breathes air"
  end

  def eat(food)
    puts "#{@name} eats #{food}"
  end

  def sleep_time
    puts "#{@name} sleeps"
  end

  def to_s
    "#{self.class.name}(#{@name}, age: #{@age})"
  end
end

# Child class (Subclass / Derived class)
class Dog < Animal
  def initialize(name, age, breed)
    super(name, age)   # เรียก constructor ของ parent
    @breed = breed
  end

  def bark
    puts "#{@name} says: Woof! Woof!"
  end

  def fetch(item)
    puts "#{@name} fetches the #{item}!"
  end

  def to_s
    "#{super} [#{@breed}]"  # ใช้ to_s ของ parent
  end
end

class Cat < Animal
  def initialize(name, age, indoor)
    super(name, age)
    @indoor = indoor
  end

  def meow
    puts "#{@name} says: Meow~"
  end

  def purr
    puts "#{@name} purrs... rrrrr"
  end

  def indoor?
    @indoor
  end
end

# ทดสอบ
dog = Dog.new("Rex", 3, "Labrador")
cat = Cat.new("Whiskers", 5, true)

dog.breathe    # จาก Animal
dog.eat("bone")  # จาก Animal
dog.bark       # จาก Dog
dog.fetch("ball")  # จาก Dog
puts dog       # => Animal(Rex, age: 3) [Labrador]

cat.breathe    # จาก Animal
cat.meow       # จาก Cat
cat.purr       # จาก Cat
puts "Indoor cat? #{cat.indoor?}"
puts cat       # => Cat(Whiskers, age: 5)

# Inheritance chain
puts dog.is_a?(Dog)     # => true
puts dog.is_a?(Animal)  # => true
puts dog.is_a?(Cat)     # => false

puts Dog.superclass     # => Animal
puts Animal.superclass  # => Object
puts Object.superclass  # => BasicObject
```

### ทำไมต้องใช้ Inheritance?

```ruby
# ไม่ใช้ inheritance - code ซ้ำมาก
class Car
  attr_reader :make, :model, :year, :color
  def initialize(make, model, year, color)
    @make = make; @model = model; @year = year; @color = color
  end
  def start_engine; puts "#{@make} engine starts: Vroom!"; end
  def stop_engine; puts "#{@make} engine stops"; end
  def info; puts "#{@year} #{@make} #{@model} (#{@color})"; end
end

class Truck
  attr_reader :make, :model, :year, :color, :payload_capacity
  def initialize(make, model, year, color, payload)
    @make = make; @model = model; @year = year; @color = color
    @payload_capacity = payload
  end
  def start_engine; puts "#{@make} engine starts: ROAR!"; end
  def stop_engine; puts "#{@make} engine stops"; end
  def info; puts "#{@year} #{@make} #{@model} (#{@color}) - #{@payload_capacity}t"; end
end

# ใช้ inheritance - ดีกว่ามาก
class Vehicle
  attr_reader :make, :model, :year, :color

  def initialize(make, model, year, color)
    @make = make
    @model = model
    @year = year
    @color = color
    @running = false
  end

  def start_engine
    @running = true
    puts "#{@make} #{engine_sound}"
  end

  def stop_engine
    @running = false
    puts "#{@make} engine stops"
  end

  def running?
    @running
  end

  def info
    puts "#{@year} #{@make} #{@model} (#{@color})"
  end

  private

  def engine_sound
    "engine starts"
  end
end

class Car < Vehicle
  private
  def engine_sound
    "engine starts: Vroom!"
  end
end

class Truck < Vehicle
  attr_reader :payload_capacity

  def initialize(make, model, year, color, payload)
    super(make, model, year, color)
    @payload_capacity = payload
  end

  def info
    super
    puts "  Payload: #{@payload_capacity} tons"
  end

  private
  def engine_sound
    "engine starts: ROAR!"
  end
end

car = Car.new("Toyota", "Camry", 2022, "Silver")
truck = Truck.new("Isuzu", "D-Max", 2023, "White", 1.5)

car.start_engine
car.info

truck.start_engine
truck.info
```

---

## Step 232: super Keyword

`super` เรียก method ชื่อเดียวกันจาก parent class

```ruby
class Shape
  def initialize(color = "black")
    @color = color
  end

  def describe
    puts "I am a #{self.class.name} in #{@color}"
  end

  def area
    0  # default
  end
end

class Rectangle < Shape
  def initialize(width, height, color = "black")
    super(color)          # เรียก Shape#initialize
    @width = width
    @height = height
  end

  def area
    @width * @height
  end

  def describe
    super                  # เรียก Shape#describe ก่อน
    puts "  Width: #{@width}, Height: #{@height}"
    puts "  Area: #{area}"
  end
end

class Square < Rectangle
  def initialize(side, color = "black")
    super(side, side, color)  # เรียก Rectangle#initialize
  end

  def describe
    super                  # เรียก Rectangle#describe
    puts "  (This is a square)"
  end
end

r = Rectangle.new(10, 5, "blue")
r.describe
puts "---"
s = Square.new(7, "red")
s.describe
```

### super กับ Arguments

```ruby
class Logger
  def log(message)
    puts "[LOG] #{message}"
  end
end

class TimestampLogger < Logger
  def log(message)
    super("#{Time.now.strftime('%H:%M:%S')} - #{message}")
  end
end

class PrefixLogger < TimestampLogger
  def initialize(prefix)
    @prefix = prefix
  end

  def log(message)
    super("[#{@prefix}] #{message}")
  end
end

logger = Logger.new
ts_logger = TimestampLogger.new
prefix_logger = PrefixLogger.new("APP")

logger.log("Hello")
ts_logger.log("Hello")
prefix_logger.log("Hello")
```

### super() vs super vs super(args)

```ruby
class Parent
  def greet(name = "World")
    puts "Hello, #{name}!"
  end
end

class Child < Parent
  def greet(name)
    # super         # ส่ง arguments เดิม (name)
    # super()       # ส่ง ไม่มี arguments (ใช้ default ของ parent)
    # super(name)   # ส่ง name อย่างชัดเจน

    super           # เหมือน super(name)
    puts "From child class"
  end
end

c = Child.new
c.greet("Ruby")
```

---

## Step 233: Method Overriding

Method overriding คือการ redefine method ที่มีอยู่ใน parent class

```ruby
class Animal
  def sound
    "..."
  end

  def describe
    puts "#{self.class.name} goes #{sound}"
  end
end

class Dog < Animal
  def sound
    "Woof"
  end
end

class Cat < Animal
  def sound
    "Meow"
  end
end

class Duck < Animal
  def sound
    "Quack"
  end
end

class Mute < Animal
  # ไม่ override - ใช้ของ parent
end

animals = [Dog.new, Cat.new, Duck.new, Mute.new]
animals.each(&:describe)
# => Dog goes Woof
# => Cat goes Meow
# => Duck goes Quack
# => Mute goes ...
```

### Polymorphism ผ่าน Method Overriding

```ruby
class Payment
  attr_reader :amount, :currency

  def initialize(amount, currency = "THB")
    @amount = amount
    @currency = currency
  end

  def process
    raise NotImplementedError, "#{self.class} must implement #process"
  end

  def fee
    0  # default no fee
  end

  def total
    @amount + fee
  end

  def receipt
    puts "=== Payment Receipt ==="
    puts "Method: #{payment_method}"
    puts "Amount: #{@currency} #{format('%.2f', @amount)}"
    puts "Fee:    #{@currency} #{format('%.2f', fee)}"
    puts "Total:  #{@currency} #{format('%.2f', total)}"
    puts "======================"
  end

  private

  def payment_method
    self.class.name
  end
end

class CashPayment < Payment
  def process
    puts "Processing cash payment of #{@currency} #{@amount}"
    true
  end

  def fee
    0  # ไม่มี fee
  end
end

class CreditCardPayment < Payment
  FEE_RATE = 0.015  # 1.5%

  def initialize(amount, card_number, currency = "THB")
    super(amount, currency)
    @card_number = mask_card(card_number)
  end

  def process
    puts "Processing credit card payment: #{@card_number}"
    puts "Amount: #{@currency} #{@amount}"
    true
  end

  def fee
    (@amount * FEE_RATE).round(2)
  end

  private

  def mask_card(number)
    number.to_s.gsub(/\d(?=\d{4})/, '*')
  end

  def payment_method
    "Credit Card (#{@card_number})"
  end
end

class PromptPayPayment < Payment
  FEE_RATE = 0.005  # 0.5%

  def initialize(amount, phone_number, currency = "THB")
    super(amount, currency)
    @phone_number = phone_number
  end

  def process
    puts "Sending PromptPay to #{@phone_number}"
    puts "Amount: #{@currency} #{@amount}"
    true
  end

  def fee
    [(@amount * FEE_RATE).round(2), 10].min  # max 10 baht fee
  end

  def payment_method
    "PromptPay (#{@phone_number})"
  end
end

payments = [
  CashPayment.new(1000),
  CreditCardPayment.new(1000, "4111111111111111"),
  PromptPayPayment.new(1000, "081-234-5678"),
]

payments.each do |payment|
  payment.process
  payment.receipt
  puts
end
```

---

## Step 234: Calling Parent Methods with super

```ruby
class Vehicle
  attr_reader :make, :model

  def initialize(make, model, year)
    @make = make
    @model = model
    @year = year
    @features = ["Engine", "Wheels", "Steering"]
  end

  def features
    @features.dup
  end

  def add_feature(feature)
    @features << feature
  end

  def description
    "#{@year} #{@make} #{@model}"
  end

  def full_description
    "#{description}\nFeatures: #{@features.join(', ')}"
  end
end

class ElectricVehicle < Vehicle
  def initialize(make, model, year, battery_capacity)
    super(make, model, year)
    @battery_capacity = battery_capacity
    add_feature("Electric Motor")
    add_feature("Battery Pack")
  end

  def range_km
    @battery_capacity * 5  # rough estimate
  end

  def description
    "#{super} (Electric)"  # ต่อจาก parent
  end

  def charge(percent)
    puts "Charging #{@make} #{@model} to #{percent}%"
  end
end

class HybridVehicle < Vehicle
  def initialize(make, model, year, fuel_type)
    super(make, model, year)
    @fuel_type = fuel_type
    add_feature("Hybrid Engine")
    add_feature("Regenerative Braking")
  end

  def description
    "#{super} (Hybrid #{@fuel_type})"
  end
end

ev = ElectricVehicle.new("Tesla", "Model 3", 2023, 75)
hybrid = HybridVehicle.new("Toyota", "Prius", 2023, "Gasoline")

puts ev.description
puts ev.range_km
ev.charge(80)
puts ev.full_description
puts
puts hybrid.description
puts hybrid.full_description
```

---

## Step 235: Protected Methods in Inheritance

Protected methods สามารถเรียกได้ภายใน class และ subclasses เท่านั้น

```ruby
class Employee
  attr_reader :name, :title

  def initialize(name, title, salary)
    @name = name
    @title = title
    @salary = salary
  end

  def info
    puts "#{@name} - #{@title}"
  end

  def salary_range
    "฿#{min_salary} - ฿#{max_salary}"
  end

  def >(other)
    salary > other.salary  # protected method ของ other ใช้ได้ใน class เดียวกัน
  end

  def ==(other)
    other.is_a?(Employee) && salary == other.salary
  end

  protected

  attr_reader :salary

  def min_salary
    salary * 0.8
  end

  def max_salary
    salary * 1.2
  end
end

class Manager < Employee
  def initialize(name, salary, team_size)
    super(name, "Manager", salary)
    @team_size = team_size
  end

  def bonus
    salary * 0.2  # ใช้ protected method ของ parent ได้
  end

  def total_compensation
    salary + bonus
  end

  def info
    super
    puts "  Team Size: #{@team_size}"
    puts "  Bonus: ฿#{format('%.2f', bonus)}"
  end
end

alice = Employee.new("Alice", "Developer", 60000)
bob = Manager.new("Bob", 80000, 5)

alice.info
bob.info
puts bob.salary_range

puts alice > bob   # เปรียบเทียบ salary ผ่าน protected method

# alice.salary  # => NoMethodError! ไม่สามารถเรียก protected method จากภายนอกได้
```

---

## Step 236: Private Methods and Inheritance

Private methods ใน Ruby สืบทอดไปยัง subclass แต่เรียกได้เฉพาะภายใน class เท่านั้น

```ruby
class BankAccount
  def initialize(balance)
    @balance = balance
  end

  def deposit(amount)
    validate_amount(amount)  # เรียก private method ได้
    @balance += amount
    log_transaction(:deposit, amount)
    puts "Deposited #{amount}. Balance: #{@balance}"
  end

  def withdraw(amount)
    validate_amount(amount)
    check_sufficient_funds(amount)
    @balance -= amount
    log_transaction(:withdrawal, amount)
    puts "Withdrew #{amount}. Balance: #{@balance}"
  end

  def balance
    @balance
  end

  private

  def validate_amount(amount)
    raise ArgumentError, "Amount must be positive" unless amount > 0
  end

  def check_sufficient_funds(amount)
    raise "Insufficient funds" if amount > @balance
  end

  def log_transaction(type, amount)
    puts "[LOG] #{type}: #{amount} at #{Time.now.strftime('%H:%M:%S')}"
  end
end

class SavingsAccount < BankAccount
  MIN_BALANCE = 500

  def withdraw(amount)
    # เรียก private methods ของ parent ได้ใน subclass
    validate_amount(amount)

    if @balance - amount < MIN_BALANCE
      raise "Cannot withdraw: minimum balance ฿#{MIN_BALANCE} required"
    end

    @balance -= amount
    log_transaction(:withdrawal, amount)
    puts "Withdrew #{amount} from savings. Balance: #{@balance}"
  end

  def interest_accrual
    interest = (@balance * 0.025 / 12).round(2)
    @balance += interest
    log_transaction(:interest, interest)
    puts "Interest accrued: #{interest}. Balance: #{@balance}"
  end
end

savings = SavingsAccount.new(5000)
savings.deposit(2000)
savings.withdraw(1000)
savings.interest_accrual

begin
  savings.withdraw(5500)  # เกิน minimum balance
rescue => e
  puts "Error: #{e.message}"
end

# savings.validate_amount(100)  # => NoMethodError! private method
```

---

## Step 237: Abstract Class Pattern in Ruby

Ruby ไม่มี abstract class แบบ Java แต่เราสร้าง pattern ได้เอง

```ruby
# Pattern 1: raise NotImplementedError
class AbstractShape
  def area
    raise NotImplementedError, "#{self.class}#area is not implemented"
  end

  def perimeter
    raise NotImplementedError, "#{self.class}#perimeter is not implemented"
  end

  def describe
    puts "#{self.class.name}: area=#{format('%.2f', area)}, perimeter=#{format('%.2f', perimeter)}"
  end

  def self.abstract_method(*methods)
    methods.each do |method|
      define_method(method) do |*args|
        raise NotImplementedError, "#{self.class}##{method} must be implemented"
      end
    end
  end
end

class ConcreteCircle < AbstractShape
  def initialize(radius)
    @radius = radius
  end

  def area
    Math::PI * @radius ** 2
  end

  def perimeter
    2 * Math::PI * @radius
  end
end

class ConcreteRectangle < AbstractShape
  def initialize(w, h)
    @w = w
    @h = h
  end

  def area
    @w * @h
  end

  def perimeter
    2 * (@w + @h)
  end
end

circle = ConcreteCircle.new(5)
rect = ConcreteRectangle.new(4, 6)

circle.describe
rect.describe

# ลองสร้าง class ที่ไม่ implement
class IncompleteShape < AbstractShape
end

begin
  incomplete = IncompleteShape.new
  incomplete.area
rescue NotImplementedError => e
  puts "Error: #{e.message}"
end

# Pattern 2: ใช้ Module
module AbstractInterface
  def self.included(klass)
    klass.instance_variable_set(:@abstract_methods, [])
    klass.extend(ClassMethods)
  end

  module ClassMethods
    def abstract_methods
      @abstract_methods
    end

    def abstract_method(*methods)
      @abstract_methods.concat(methods)
      methods.each do |method|
        define_method(method) do |*args|
          raise NotImplementedError, "#{self.class}##{method} is abstract and must be implemented"
        end
      end
    end
  end

  def check_implementation!
    unimplemented = self.class.abstract_methods.select do |m|
      method(m).owner == AbstractInterface::ClassMethods ||
      (respond_to?(m) && method(m).owner.ancestors.include?(AbstractInterface))
    end
    unless unimplemented.empty?
      raise "#{self.class} must implement: #{unimplemented.join(', ')}"
    end
  end
end
```

---

## Step 238: ancestors Chain

```ruby
class A; end
class B < A; end
class C < B; end

puts C.ancestors.inspect
# => [C, B, A, Object, Kernel, BasicObject]

module Greetable
  def greet; puts "Hello!"; end
end

module Serializable
  def serialize; puts "Serializing..."; end
end

class Base
  include Serializable
end

class Child < Base
  include Greetable
end

puts Child.ancestors.inspect
# => [Child, Greetable, Base, Serializable, Object, Kernel, BasicObject]

# Method lookup ตาม ancestors chain
child = Child.new
child.greet      # จาก Greetable
child.serialize  # จาก Serializable

# ตรวจสอบ ancestors
puts Child.ancestors.include?(Base)         # => true
puts Child.ancestors.include?(Greetable)    # => true
puts Child.ancestors.include?(Serializable) # => true
```

---

## Step 239: is_a? / kind_of? / instance_of?

```ruby
class Animal; end
class Dog < Animal
  def initialize(name)
    @name = name
  end
end
class Labrador < Dog; end

lab = Labrador.new("Buddy")

# is_a? / kind_of? - ตรวจสอบตาม inheritance chain
puts lab.is_a?(Labrador)   # => true  (class ตัวเอง)
puts lab.is_a?(Dog)        # => true  (parent)
puts lab.is_a?(Animal)     # => true  (grandparent)
puts lab.is_a?(Object)     # => true  (สืบทอดจาก Object เสมอ)
puts lab.is_a?(String)     # => false

puts lab.kind_of?(Dog)     # => true  (เหมือน is_a?)

# instance_of? - ตรวจสอบ exact class เท่านั้น
puts lab.instance_of?(Labrador)  # => true   (class ตัวเอง)
puts lab.instance_of?(Dog)       # => false  (parent ไม่นับ)
puts lab.instance_of?(Animal)    # => false

# respond_to? - ตรวจสอบว่ามี method หรือไม่
puts lab.respond_to?(:bark)    # => false (Dog ไม่มี bark)
puts lab.respond_to?(:is_a?)   # => true (มาจาก Object)

# class - ดู class ตัวเอง
puts lab.class            # => Labrador
puts lab.class.name       # => "Labrador"
puts lab.class.superclass # => Dog

# ใช้ใน type checking
def process_animal(animal)
  if animal.is_a?(Dog)
    puts "Processing a dog: #{animal.instance_variable_get(:@name)}"
  elsif animal.is_a?(Animal)
    puts "Processing some animal"
  else
    raise TypeError, "Expected Animal, got #{animal.class}"
  end
end

process_animal(lab)
```

---

## Step 240: Class Methods in Inheritance

```ruby
class Vehicle
  @@count = 0

  def initialize(make, model)
    @make = make
    @model = model
    @@count += 1
  end

  def self.count
    @@count
  end

  def self.create(make, model)
    new(make, model)
  end

  def self.description
    "Generic Vehicle"
  end

  def info
    puts "#{self.class}: #{@make} #{@model}"
  end
end

class Car < Vehicle
  @@car_count = 0

  def initialize(make, model, doors = 4)
    super(make, model)
    @doors = doors
    @@car_count += 1
  end

  def self.car_count
    @@car_count
  end

  def self.description
    "#{super} - specifically a Car"
  end

  def self.sedan(make, model)
    new(make, model, 4)
  end

  def self.coupe(make, model)
    new(make, model, 2)
  end
end

class Motorcycle < Vehicle
  def self.description
    "#{super} - specifically a Motorcycle"
  end
end

car1 = Car.sedan("Toyota", "Camry")
car2 = Car.coupe("Honda", "Civic")
moto = Motorcycle.create("Yamaha", "MT-07")

puts Vehicle.count     # => 3 (สะสมทุก subclass)
puts Car.car_count     # => 2
puts Car.description
puts Motorcycle.description

car1.info
car2.info
moto.info

# class method inheritance
puts Car.count         # => 3 (Car สืบทอด .count จาก Vehicle)
puts Motorcycle.count  # => 3
```

---

## Step 241: Template Method Pattern

Template Method Pattern กำหนดโครงสร้าง algorithm ใน parent และให้ subclass ปรับรายละเอียด

```ruby
class ReportGenerator
  # Template method - กำหนดลำดับขั้นตอน
  def generate(data)
    result = []
    result << header
    result << separator
    result += format_data(data)
    result << separator
    result << footer(data)
    result.join("\n")
  end

  private

  # Abstract methods ที่ subclass ต้อง implement
  def header
    raise NotImplementedError, "Implement header"
  end

  def format_data(data)
    raise NotImplementedError, "Implement format_data"
  end

  # Concrete method ที่ subclass อาจ override
  def separator
    "-" * 40
  end

  def footer(data)
    "Total: #{data.length} records"
  end
end

class TextReport < ReportGenerator
  private

  def header
    "=== TEXT REPORT ==="
  end

  def format_data(data)
    data.map.with_index(1) { |item, i| "#{i}. #{item}" }
  end
end

class CSVReport < ReportGenerator
  private

  def header
    "id,value"
  end

  def separator
    ""  # ไม่ต้องการ separator ใน CSV
  end

  def format_data(data)
    data.map.with_index(1) { |item, i| "#{i},#{item}" }
  end

  def footer(data)
    "# #{data.length} records exported"
  end
end

class HTMLReport < ReportGenerator
  private

  def header
    "<html><body><h1>Report</h1><ul>"
  end

  def separator
    ""
  end

  def format_data(data)
    data.map { |item| "  <li>#{item}</li>" }
  end

  def footer(data)
    "</ul><p>#{data.length} items</p></body></html>"
  end
end

data = ["Apple", "Banana", "Cherry", "Date"]

puts "=== Text Format ==="
puts TextReport.new.generate(data)

puts "\n=== CSV Format ==="
puts CSVReport.new.generate(data)

puts "\n=== HTML Format ==="
puts HTMLReport.new.generate(data)
```

### Template Method สำหรับ Data Processing

```ruby
class DataProcessor
  def process(filename)
    raw_data = load_data(filename)
    parsed_data = parse(raw_data)
    validated_data = validate(parsed_data)
    transformed_data = transform(validated_data)
    save_results(transformed_data)
  end

  private

  def load_data(filename)
    puts "Loading from: #{filename}"
    # Simulate loading
    "raw,data,here\n1,2,3\n4,5,6"
  end

  def parse(raw)
    puts "Parsing data..."
    raw.split("\n").map { |line| line.split(",") }
  end

  def validate(data)
    puts "Validating..."
    data.select { |row| row.all? { |cell| !cell.empty? } }
  end

  # subclass ต้อง implement
  def transform(data)
    raise NotImplementedError
  end

  def save_results(data)
    puts "Saving #{data.length} rows..."
    data
  end
end

class SumProcessor < DataProcessor
  private

  def transform(data)
    puts "Summing rows..."
    data.map { |row| [row.sum { |cell| cell.to_i }] }
  end
end

class UppercaseProcessor < DataProcessor
  private

  def transform(data)
    puts "Uppercasing..."
    data.map { |row| row.map(&:upcase) }
  end
end

puts "=== Sum Processor ==="
result = SumProcessor.new.process("data.csv")
puts result.inspect

puts "\n=== Uppercase Processor ==="
result = UppercaseProcessor.new.process("data.csv")
puts result.inspect
```

---

## Step 242-255: Building Complete Shape Hierarchy

```ruby
# Base Shape class
class Shape
  include Comparable

  attr_reader :color, :filled

  def initialize(color: "black", filled: true)
    @color = color
    @filled = filled
  end

  def area
    raise NotImplementedError, "#{self.class} must implement #area"
  end

  def perimeter
    raise NotImplementedError, "#{self.class} must implement #perimeter"
  end

  def <=>(other)
    area <=> other.area
  end

  def describe
    fill_status = @filled ? "filled" : "outline"
    puts "#{self.class.name} [#{@color}, #{fill_status}]"
    puts "  Area:      #{format('%.4f', area)}"
    puts "  Perimeter: #{format('%.4f', perimeter)}"
  end

  def scale(factor)
    raise NotImplementedError, "#{self.class} must implement #scale"
  end

  def to_s
    "#{self.class.name}(area=#{format('%.2f', area)})"
  end

  def inspect
    "#<#{self.class.name} color=#{@color.inspect} filled=#{@filled} area=#{format('%.4f', area)}>"
  end

  def similar_to?(other, tolerance = 0.01)
    (area - other.area).abs <= tolerance * [area, other.area].max
  end
end

class Circle < Shape
  attr_reader :radius

  def initialize(radius, **options)
    super(**options)
    raise ArgumentError, "Radius must be positive" unless radius > 0
    @radius = radius.to_f
  end

  def area
    Math::PI * @radius ** 2
  end

  def perimeter
    2 * Math::PI * @radius
  end

  def diameter
    @radius * 2
  end

  def scale(factor)
    Circle.new(@radius * factor, color: @color, filled: @filled)
  end

  def circumscribe_square_side
    @radius * Math.sqrt(2) * 2 / 2
  end

  def describe
    super
    puts "  Radius: #{@radius}"
    puts "  Diameter: #{diameter}"
  end
end

class Rectangle < Shape
  attr_reader :width, :height

  def initialize(width, height, **options)
    super(**options)
    raise ArgumentError, "Width must be positive" unless width > 0
    raise ArgumentError, "Height must be positive" unless height > 0
    @width = width.to_f
    @height = height.to_f
  end

  def area
    @width * @height
  end

  def perimeter
    2 * (@width + @height)
  end

  def diagonal
    Math.sqrt(@width ** 2 + @height ** 2)
  end

  def square?
    @width == @height
  end

  def aspect_ratio
    @width / @height
  end

  def scale(factor)
    Rectangle.new(@width * factor, @height * factor, color: @color, filled: @filled)
  end

  def describe
    super
    puts "  Width: #{@width}, Height: #{@height}"
    puts "  Diagonal: #{format('%.4f', diagonal)}"
    puts "  Square: #{square?}"
  end
end

class Square < Rectangle
  attr_reader :side

  def initialize(side, **options)
    super(side, side, **options)
    @side = side.to_f
  end

  def scale(factor)
    Square.new(@side * factor, color: @color, filled: @filled)
  end

  def inscribed_circle_radius
    @side / 2.0
  end

  def circumscribed_circle_radius
    @side * Math.sqrt(2) / 2.0
  end

  def describe
    super
    puts "  Side: #{@side}"
    puts "  Inscribed circle radius: #{format('%.4f', inscribed_circle_radius)}"
  end
end

class Triangle < Shape
  attr_reader :a, :b, :c

  def initialize(a, b, c, **options)
    validate_triangle!(a, b, c)
    super(**options)
    @a = a.to_f
    @b = b.to_f
    @c = c.to_f
  end

  def perimeter
    @a + @b + @c
  end

  def area
    # Heron's formula
    s = perimeter / 2.0
    Math.sqrt(s * (s - @a) * (s - @b) * (s - @c))
  end

  def scale(factor)
    Triangle.new(@a * factor, @b * factor, @c * factor, color: @color, filled: @filled)
  end

  def equilateral?
    @a == @b && @b == @c
  end

  def isosceles?
    @a == @b || @b == @c || @a == @c
  end

  def scalene?
    @a != @b && @b != @c && @a != @c
  end

  def right_triangle?
    sides = [@a, @b, @c].sort
    (sides[2] ** 2 - (sides[0] ** 2 + sides[1] ** 2)).abs < 0.0001
  end

  def type_description
    if equilateral? then "Equilateral"
    elsif isosceles? then "Isosceles"
    else "Scalene"
    end
  end

  def angles_degrees
    # Law of cosines
    alpha = Math.acos((@b**2 + @c**2 - @a**2) / (2 * @b * @c)) * 180 / Math::PI
    beta  = Math.acos((@a**2 + @c**2 - @b**2) / (2 * @a * @c)) * 180 / Math::PI
    gamma = 180 - alpha - beta
    [alpha.round(2), beta.round(2), gamma.round(2)]
  end

  def describe
    super
    puts "  Sides: #{@a}, #{@b}, #{@c}"
    puts "  Type: #{type_description}"
    puts "  Right triangle: #{right_triangle?}"
    angles = angles_degrees
    puts "  Angles: #{angles.join('°, ')}°"
  end

  private

  def validate_triangle!(a, b, c)
    unless a > 0 && b > 0 && c > 0
      raise ArgumentError, "All sides must be positive"
    end
    unless a + b > c && a + c > b && b + c > a
      raise ArgumentError, "Invalid triangle: sides #{a}, #{b}, #{c} don't satisfy triangle inequality"
    end
  end
end

class RegularPolygon < Shape
  attr_reader :sides, :side_length

  def initialize(sides, side_length, **options)
    raise ArgumentError, "Must have at least 3 sides" unless sides >= 3
    raise ArgumentError, "Side length must be positive" unless side_length > 0
    super(**options)
    @sides = sides
    @side_length = side_length.to_f
  end

  def perimeter
    @sides * @side_length
  end

  def area
    (@sides * @side_length ** 2) / (4 * Math.tan(Math::PI / @sides))
  end

  def interior_angle
    ((@sides - 2) * 180.0) / @sides
  end

  def exterior_angle
    360.0 / @sides
  end

  def scale(factor)
    RegularPolygon.new(@sides, @side_length * factor, color: @color, filled: @filled)
  end

  def describe
    super
    puts "  Sides: #{@sides}"
    puts "  Side length: #{@side_length}"
    puts "  Interior angle: #{format('%.2f', interior_angle)}°"
    puts "  Exterior angle: #{format('%.2f', exterior_angle)}°"
  end

  def name
    names = {
      3 => "Equilateral Triangle",
      4 => "Square",
      5 => "Pentagon",
      6 => "Hexagon",
      7 => "Heptagon",
      8 => "Octagon",
      9 => "Nonagon",
      10 => "Decagon"
    }
    names[@sides] || "#{@sides}-gon"
  end
end

# ShapeCollection สำหรับ manage shapes
class ShapeCollection
  include Enumerable

  def initialize(name = "Shapes")
    @name = name
    @shapes = []
  end

  def <<(shape)
    raise TypeError, "Expected Shape, got #{shape.class}" unless shape.is_a?(Shape)
    @shapes << shape
    self
  end

  def add(*shapes)
    shapes.each { |s| self << s }
    self
  end

  def each(&block)
    @shapes.each(&block)
  end

  def total_area
    @shapes.sum(&:area)
  end

  def total_perimeter
    @shapes.sum(&:perimeter)
  end

  def largest
    @shapes.max_by(&:area)
  end

  def smallest
    @shapes.min_by(&:area)
  end

  def by_type(type)
    @shapes.select { |s| s.is_a?(type) }
  end

  def sort_by_area
    @shapes.sort_by(&:area)
  end

  def statistics
    areas = @shapes.map(&:area)
    {
      count: @shapes.length,
      total_area: areas.sum,
      average_area: areas.sum / areas.length,
      min_area: areas.min,
      max_area: areas.max
    }
  end

  def summary
    puts "\n=== #{@name} Collection Summary ==="
    puts "Count: #{@shapes.length} shapes"
    puts format("Total Area:     %.4f", total_area)
    puts format("Total Perimeter: %.4f", total_perimeter)
    puts format("Largest:  #{largest}")
    puts format("Smallest: #{smallest}")
    puts "\nBy type:"
    [Circle, Rectangle, Square, Triangle, RegularPolygon].each do |type|
      shapes = by_type(type)
      puts "  #{type.name}: #{shapes.length}" unless shapes.empty?
    end
    puts
  end
end

# ทดสอบ Shape Hierarchy
puts "=== Testing Shape Hierarchy ==="

c = Circle.new(5, color: "red")
r = Rectangle.new(8, 6, color: "blue")
s = Square.new(7, color: "green")
t = Triangle.new(3, 4, 5, color: "yellow")
p5 = RegularPolygon.new(5, 6, color: "purple")
p6 = RegularPolygon.new(6, 4, color: "orange")

shapes = ShapeCollection.new("Test Shapes")
shapes.add(c, r, s, t, p5, p6)

puts "\nAll shapes described:"
shapes.each(&:describe)

shapes.summary

puts "Shapes sorted by area:"
shapes.sort_by_area.each { |s| puts "  #{s}" }

puts "\nScaling circle by 2:"
scaled = c.scale(2)
scaled.describe

puts "\nComparisons:"
puts "Circle vs Rectangle: #{c > r}"
puts "Square vs Triangle: #{s < t}"

# Polymorphism test
puts "\nPolymorphism - calling area on different shapes:"
[c, r, s, t, p5].each do |shape|
  puts "  #{shape.class.name}: area = #{format('%.4f', shape.area)}"
end

# Type checking
puts "\nType checking:"
puts "Square is_a? Rectangle: #{s.is_a?(Rectangle)}"
puts "Square is_a? Shape: #{s.is_a?(Shape)}"
puts "Circle is_a? Rectangle: #{c.is_a?(Rectangle)}"

# Triangle analysis
puts "\nTriangle analysis:"
t2 = Triangle.new(5, 5, 5)  # equilateral
t3 = Triangle.new(5, 5, 8)  # isosceles
t4 = Triangle.new(3, 4, 5)  # right triangle

puts "Equilateral triangle type: #{t2.type_description}"
puts "Isosceles triangle type: #{t3.type_description}"
puts "Right triangle: #{t4.right_triangle?}"
puts "Right triangle angles: #{t4.angles_degrees.join(', ')}°"

# RegularPolygon names
puts "\nRegular Polygon names:"
(3..10).each do |n|
  poly = RegularPolygon.new(n, 5)
  puts "  #{poly.name}: interior angle = #{format('%.2f', poly.interior_angle)}°"
end
```

---

## แบบฝึกหัด 25 ข้อ พร้อมเฉลย

### ข้อที่ 1: Animal Hierarchy

**โจทย์:** สร้าง class hierarchy สำหรับสัตว์ต่างๆ

```ruby
class Animal
  attr_reader :name, :habitat

  def initialize(name, habitat)
    @name = name
    @habitat = habitat
  end

  def breathe
    "breathes air"
  end

  def move
    "moves"
  end

  def sound
    "..."
  end

  def describe
    puts "#{@name}: #{move}, #{breathe}, says '#{sound}'"
  end
end

class Mammal < Animal
  def initialize(name, habitat, warm_blooded: true)
    super(name, habitat)
    @warm_blooded = warm_blooded
  end

  def warm_blooded?
    @warm_blooded
  end

  def breathe
    "breathes air through lungs"
  end
end

class Bird < Animal
  def initialize(name, can_fly: true)
    super(name, "sky/land")
    @can_fly = can_fly
  end

  def fly?
    @can_fly
  end

  def move
    @can_fly ? "flies and walks" : "walks"
  end

  def breathe
    "breathes through efficient avian lungs"
  end
end

class Fish < Animal
  def initialize(name, water_type = "fresh")
    super(name, "water")
    @water_type = water_type
  end

  def breathe
    "breathes through gills"
  end

  def move
    "swims"
  end
end

class Dog < Mammal
  def initialize(name, breed)
    super(name, "land")
    @breed = breed
  end

  def sound
    "Woof!"
  end

  def move
    "runs on 4 legs"
  end
end

class Eagle < Bird
  def initialize(name)
    super(name, can_fly: true)
  end

  def sound
    "Screech!"
  end
end

class Penguin < Bird
  def initialize(name)
    super(name, can_fly: false)
  end

  def move
    "waddles and swims"
  end

  def sound
    "Squawk!"
  end
end

class Salmon < Fish
  def initialize(name)
    super(name, "fresh/salt")
  end

  def migrate
    puts "#{@name} migrates upstream to spawn"
  end
end

animals = [
  Dog.new("Rex", "German Shepherd"),
  Eagle.new("Sam"),
  Penguin.new("Pingu"),
  Salmon.new("Flipper")
]

animals.each(&:describe)

puts "\nType checks:"
puts "Rex is Mammal? #{animals[0].is_a?(Mammal)}"
puts "Sam is Bird? #{animals[1].is_a?(Bird)}"
puts "Pingu can fly? #{animals[2].fly?}"
```

### ข้อที่ 2: Employee Hierarchy

```ruby
class Employee
  attr_reader :name, :id, :department

  @@employee_count = 0

  def initialize(name, department)
    @@employee_count += 1
    @id = "EMP#{format('%04d', @@employee_count)}"
    @name = name
    @department = department
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

  def payslip
    puts "=" * 40
    puts "Employee: #{@name} (#{@id})"
    puts "Department: #{@department}"
    puts "Position: #{self.class.name}"
    puts "-" * 40
    puts format("%-25s %10.2f", "Base Salary:", base_salary)
    puts format("%-25s %10.2f", "Bonus:", bonus)
    puts format("%-25s %10.2f", "Total:", total_compensation)
    puts "=" * 40
  end

  def self.employee_count
    @@employee_count
  end
end

class FullTimeEmployee < Employee
  def initialize(name, department, monthly_salary)
    super(name, department)
    @monthly_salary = monthly_salary
  end

  def base_salary
    @monthly_salary
  end

  def bonus
    @monthly_salary * 0.1
  end
end

class PartTimeEmployee < Employee
  def initialize(name, department, hourly_rate, hours_per_week)
    super(name, department)
    @hourly_rate = hourly_rate
    @hours_per_week = hours_per_week
  end

  def base_salary
    @hourly_rate * @hours_per_week * 4  # approximate monthly
  end
end

class Manager < FullTimeEmployee
  def initialize(name, department, monthly_salary, team_size)
    super(name, department, monthly_salary)
    @team_size = team_size
  end

  def bonus
    super + (@team_size * 500)  # extra bonus per team member
  end
end

class Contractor < Employee
  def initialize(name, department, daily_rate, days_this_month)
    super(name, department)
    @daily_rate = daily_rate
    @days = days_this_month
  end

  def base_salary
    @daily_rate * @days
  end
end

employees = [
  FullTimeEmployee.new("Alice", "Engineering", 60000),
  PartTimeEmployee.new("Bob", "Marketing", 200, 20),
  Manager.new("Charlie", "Engineering", 90000, 8),
  Contractor.new("Diana", "Design", 3000, 22)
]

employees.each do |e|
  e.payslip
  puts
end

total = employees.sum(&:total_compensation)
puts "Total payroll: ฿#{format('%.2f', total)}"
puts "Total employees: #{Employee.employee_count}"
```

### ข้อที่ 3-10: เพิ่มเติม

```ruby
# ข้อที่ 3: Vehicle Fleet
class Vehicle
  attr_reader :make, :model, :year, :license_plate

  def initialize(make, model, year)
    @make = make
    @model = model
    @year = year
    @license_plate = nil
    @mileage = 0
  end

  def register(plate)
    @license_plate = plate
    puts "#{@make} #{@model} registered as #{plate}"
  end

  def drive(km)
    @mileage += km
  end

  def mileage
    @mileage
  end

  def fuel_cost_per_km
    raise NotImplementedError
  end

  def trip_cost(km)
    km * fuel_cost_per_km
  end

  def info
    puts "#{@year} #{@make} #{@model} [#{@license_plate || 'unregistered'}]"
    puts "  Mileage: #{@mileage} km"
  end
end

class GasolineCar < Vehicle
  def initialize(make, model, year, fuel_efficiency_l_per_100km)
    super(make, model, year)
    @efficiency = fuel_efficiency_l_per_100km
    @gasoline_price = 40  # baht per liter
  end

  def fuel_cost_per_km
    (@efficiency / 100.0) * @gasoline_price
  end
end

class ElectricCar < Vehicle
  def initialize(make, model, year, kwh_per_100km)
    super(make, model, year)
    @efficiency = kwh_per_100km
    @electricity_price = 4  # baht per kWh
  end

  def fuel_cost_per_km
    (@efficiency / 100.0) * @electricity_price
  end
end

class Truck < GasolineCar
  def initialize(make, model, year, efficiency, payload_tons)
    super(make, model, year, efficiency)
    @payload = payload_tons
  end

  def freight_capacity
    @payload
  end
end

gas_car = GasolineCar.new("Honda", "Civic", 2022, 7.5)
ev = ElectricCar.new("Tesla", "Model 3", 2023, 15)
truck = Truck.new("Isuzu", "D-Max", 2023, 12, 1.5)

gas_car.register("กข 1234 กรุงเทพ")
ev.register("ขค 5678 กรุงเทพ")

gas_car.drive(500)
ev.drive(500)

puts "Trip cost (500km):"
puts "  Gas car: ฿#{format('%.2f', gas_car.trip_cost(500))}"
puts "  Electric: ฿#{format('%.2f', ev.trip_cost(500))}"
puts "  Savings with EV: ฿#{format('%.2f', gas_car.trip_cost(500) - ev.trip_cost(500))}"
```

```ruby
# ข้อที่ 4: Database Record Pattern
class Record
  @@records = {}
  @@id_counter = 0

  def initialize(attributes = {})
    @@id_counter += 1
    @id = @@id_counter
    @created_at = Time.now
    @updated_at = Time.now
    set_attributes(attributes)
  end

  attr_reader :id, :created_at, :updated_at

  def self.create(attributes = {})
    record = new(attributes)
    (@@records[self.name] ||= {})[record.id] = record
    puts "Created #{self.name} with ID #{record.id}"
    record
  end

  def self.find(id)
    (@@records[self.name] || {})[id]
  end

  def self.all
    (@@records[self.name] || {}).values
  end

  def self.count
    (@@records[self.name] || {}).length
  end

  def update(attributes = {})
    set_attributes(attributes)
    @updated_at = Time.now
    puts "Updated #{self.class.name} ID #{@id}"
    self
  end

  def destroy
    (@@records[self.class.name] || {}).delete(@id)
    puts "Deleted #{self.class.name} ID #{@id}"
  end

  def to_h
    { id: @id, created_at: @created_at, updated_at: @updated_at }
  end

  private

  def set_attributes(attrs)
    attrs.each do |key, value|
      setter = "#{key}="
      send(setter, value) if respond_to?(setter)
    end
  end
end

class User < Record
  attr_accessor :name, :email, :role

  def initialize(attributes = {})
    @role = "user"
    super
  end

  def to_h
    super.merge(name: @name, email: @email, role: @role)
  end

  def to_s
    "User[#{@id}]: #{@name} <#{@email}> (#{@role})"
  end
end

class Post < Record
  attr_accessor :title, :body, :author_id, :published

  def initialize(attributes = {})
    @published = false
    super
  end

  def publish!
    @published = true
    @updated_at = Time.now
    puts "Published: #{@title}"
  end

  def to_s
    status = @published ? "published" : "draft"
    "Post[#{@id}]: #{@title} (#{status})"
  end
end

# ทดสอบ
alice = User.create(name: "Alice", email: "alice@example.com", role: "admin")
bob = User.create(name: "Bob", email: "bob@example.com")

post1 = Post.create(title: "Hello Ruby", body: "Ruby is great!", author_id: alice.id)
post2 = Post.create(title: "OOP Basics", body: "Classes and objects...", author_id: alice.id)
post3 = Post.create(title: "Draft Post", body: "Still writing...", author_id: bob.id)

post1.publish!

puts "\nAll Users:"
User.all.each { |u| puts "  #{u}" }

puts "\nAll Posts:"
Post.all.each { |p| puts "  #{p}" }

puts "\nUsers count: #{User.count}"
puts "Posts count: #{Post.count}"

found = User.find(alice.id)
puts "\nFound: #{found}"
```

### ข้อที่ 11-25

```ruby
# ข้อที่ 11: Notification System
class Notification
  attr_reader :title, :message, :created_at, :read

  def initialize(title, message)
    @title = title
    @message = message
    @created_at = Time.now
    @read = false
  end

  def read!
    @read = true
  end

  def unread?
    !@read
  end

  def deliver
    raise NotImplementedError, "#{self.class} must implement #deliver"
  end

  def summary
    status = @read ? "READ" : "UNREAD"
    "[#{status}] #{@title}: #{@message[0..50]}..."
  end
end

class EmailNotification < Notification
  def initialize(title, message, recipient_email)
    super(title, message)
    @recipient_email = recipient_email
  end

  def deliver
    puts "📧 EMAIL to #{@recipient_email}"
    puts "   Subject: #{@title}"
    puts "   Body: #{@message}"
    true
  end
end

class SMSNotification < Notification
  MAX_LENGTH = 160

  def initialize(title, message, phone_number)
    super(title, message)
    @phone = phone_number
  end

  def deliver
    sms_text = "#{@title}: #{@message}"[0..MAX_LENGTH]
    puts "📱 SMS to #{@phone}: #{sms_text}"
    true
  end
end

class PushNotification < Notification
  def initialize(title, message, device_token)
    super(title, message)
    @token = device_token
  end

  def deliver
    puts "🔔 PUSH to #{@token[0..8]}..."
    puts "   #{@title}: #{@message[0..100]}"
    true
  end
end

class NotificationBatch
  def initialize
    @notifications = []
  end

  def add(notification)
    @notifications << notification
    self
  end

  def deliver_all
    puts "\n=== Delivering #{@notifications.length} notifications ==="
    delivered = 0
    @notifications.each do |n|
      delivered += 1 if n.deliver
    end
    puts "Delivered: #{delivered}/#{@notifications.length}"
  end

  def unread
    @notifications.select(&:unread?)
  end
end

batch = NotificationBatch.new
batch.add(EmailNotification.new("Welcome!", "Welcome to our service!", "alice@example.com"))
     .add(SMSNotification.new("OTP", "Your OTP is: 123456", "081-234-5678"))
     .add(PushNotification.new("Deal!", "Flash sale - 50% off!", "device_token_abc123"))

batch.deliver_all
```

```ruby
# ข้อที่ 12: Logger Hierarchy
class BaseLogger
  LEVELS = { debug: 0, info: 1, warn: 2, error: 3, fatal: 4 }

  def initialize(level: :debug, prefix: "")
    @level = level
    @prefix = prefix
  end

  def debug(msg)   = log(:debug, msg)
  def info(msg)    = log(:info, msg)
  def warn(msg)    = log(:warn, msg)
  def error(msg)   = log(:error, msg)
  def fatal(msg)   = log(:fatal, msg)

  def level=(new_level)
    raise ArgumentError, "Invalid level" unless LEVELS.key?(new_level)
    @level = new_level
  end

  private

  def log(level, message)
    return if LEVELS[level] < LEVELS[@level]
    write(format_message(level, message))
  end

  def format_message(level, message)
    timestamp = Time.now.strftime("%Y-%m-%d %H:%M:%S")
    prefix = @prefix.empty? ? "" : "[#{@prefix}] "
    "#{timestamp} #{level.to_s.upcase.ljust(5)} #{prefix}#{message}"
  end

  def write(formatted)
    raise NotImplementedError
  end
end

class ConsoleLogger < BaseLogger
  COLORS = {
    debug: "\e[37m",  # white
    info:  "\e[32m",  # green
    warn:  "\e[33m",  # yellow
    error: "\e[31m",  # red
    fatal: "\e[35m",  # magenta
  }
  RESET = "\e[0m"

  def initialize(colorize: true, **opts)
    super(**opts)
    @colorize = colorize
  end

  private

  def write(formatted)
    if @colorize
      level = formatted.split[1].downcase.to_sym
      puts "#{COLORS[level]}#{formatted}#{RESET}"
    else
      puts formatted
    end
  end
end

class FileLogger < BaseLogger
  def initialize(filename, **opts)
    super(**opts)
    @filename = filename
  end

  private

  def write(formatted)
    File.open(@filename, "a") { |f| f.puts formatted }
  end
end

class MultiLogger < BaseLogger
  def initialize(*loggers, **opts)
    super(**opts)
    @loggers = loggers
  end

  private

  def write(formatted)
    @loggers.each { |logger| logger.send(:write, formatted) }
  end
end

console_log = ConsoleLogger.new(level: :debug, prefix: "APP", colorize: false)
console_log.debug("Starting application")
console_log.info("Server listening on port 3000")
console_log.warn("Low memory warning")
console_log.error("Database connection failed")
console_log.fatal("Critical system failure!")

console_log.level = :warn
console_log.debug("This won't show")
console_log.warn("This will show")
```

```ruby
# ข้อที่ 13-25: Pattern implementations
# ข้อที่ 13: Command Pattern
class Command
  def execute
    raise NotImplementedError
  end

  def undo
    raise NotImplementedError
  end

  def description
    self.class.name
  end
end

class TextEditor
  def initialize
    @text = ""
    @history = []
  end

  def execute(command)
    command.execute(self)
    @history << command
    puts "Executed: #{command.description}"
  end

  def undo
    return puts "Nothing to undo" if @history.empty?
    command = @history.pop
    command.undo(self)
    puts "Undone: #{command.description}"
  end

  def text
    @text.dup
  end

  def text=(new_text)
    @text = new_text
  end
end

class InsertCommand < Command
  def initialize(text, position)
    @text = text
    @position = position
  end

  def execute(editor)
    @old_text = editor.text
    editor.text = @old_text.insert(@position, @text)
  end

  def undo(editor)
    editor.text = @old_text
  end

  def description
    "Insert '#{@text}' at position #{@position}"
  end
end

class DeleteCommand < Command
  def initialize(position, length)
    @position = position
    @length = length
  end

  def execute(editor)
    @old_text = editor.text
    editor.text = @old_text.dup.tap { |t| t[@position, @length] = "" }
  end

  def undo(editor)
    editor.text = @old_text
  end

  def description
    "Delete #{@length} chars at #{@position}"
  end
end

editor = TextEditor.new
editor.execute(InsertCommand.new("Hello", 0))
puts "Text: '#{editor.text}'"

editor.execute(InsertCommand.new(" World", 5))
puts "Text: '#{editor.text}'"

editor.execute(DeleteCommand.new(5, 6))
puts "Text: '#{editor.text}'"

editor.undo
puts "After undo: '#{editor.text}'"

editor.undo
puts "After undo: '#{editor.text}'"
```

```ruby
# ข้อที่ 14: Strategy Pattern
class Sorter
  def initialize(strategy = :bubble)
    @strategy = case strategy
    when :bubble then BubbleSortStrategy.new
    when :quick  then QuickSortStrategy.new
    when :merge  then MergeSortStrategy.new
    else strategy  # allow passing object directly
    end
  end

  def sort(array)
    @strategy.sort(array.dup)
  end

  def strategy=(new_strategy)
    @strategy = new_strategy
  end
end

class SortStrategy
  def sort(array)
    raise NotImplementedError
  end
end

class BubbleSortStrategy < SortStrategy
  def sort(array)
    n = array.length
    (n - 1).times do |i|
      (n - 1 - i).times do |j|
        if array[j] > array[j + 1]
          array[j], array[j + 1] = array[j + 1], array[j]
        end
      end
    end
    array
  end
end

class QuickSortStrategy < SortStrategy
  def sort(array)
    return array if array.length <= 1
    pivot = array[array.length / 2]
    left = array.select { |x| x < pivot }
    middle = array.select { |x| x == pivot }
    right = array.select { |x| x > pivot }
    sort(left) + middle + sort(right)
  end
end

class MergeSortStrategy < SortStrategy
  def sort(array)
    return array if array.length <= 1
    mid = array.length / 2
    left = sort(array[0...mid])
    right = sort(array[mid..])
    merge(left, right)
  end

  private

  def merge(left, right)
    result = []
    until left.empty? || right.empty?
      if left.first <= right.first
        result << left.shift
      else
        result << right.shift
      end
    end
    result + left + right
  end
end

data = [64, 34, 25, 12, 22, 11, 90]

sorter = Sorter.new(:bubble)
puts "Bubble: #{sorter.sort(data).inspect}"

sorter = Sorter.new(:quick)
puts "Quick:  #{sorter.sort(data).inspect}"

sorter = Sorter.new(:merge)
puts "Merge:  #{sorter.sort(data).inspect}"
```

```ruby
# ข้อที่ 15: Decorator Pattern
class Coffee
  def cost
    10.0
  end

  def description
    "Coffee"
  end
end

class CoffeeDecorator < Coffee
  def initialize(coffee)
    @coffee = coffee
  end

  def cost
    @coffee.cost
  end

  def description
    @coffee.description
  end
end

class MilkDecorator < CoffeeDecorator
  def cost
    super + 5.0
  end

  def description
    "#{super}, Milk"
  end
end

class SugarDecocorator < CoffeeDecorator
  def cost
    super + 2.0
  end

  def description
    "#{super}, Sugar"
  end
end

class VanillaDecorator < CoffeeDecorator
  def cost
    super + 8.0
  end

  def description
    "#{super}, Vanilla"
  end
end

class WhipCreamDecorator < CoffeeDecorator
  def cost
    super + 10.0
  end

  def description
    "#{super}, Whip Cream"
  end
end

# สั่ง custom coffee
my_coffee = Coffee.new
my_coffee = MilkDecorator.new(my_coffee)
my_coffee = SugarDecocorator.new(my_coffee)
my_coffee = VanillaDecorator.new(my_coffee)
my_coffee = WhipCreamDecorator.new(my_coffee)

puts "Order: #{my_coffee.description}"
puts "Total: ฿#{format('%.2f', my_coffee.cost)}"
```

```ruby
# ข้อที่ 16-20: More patterns

# ข้อที่ 16: Prototype Pattern
class Prototype
  def clone_deep
    Marshal.load(Marshal.dump(self))
  end
end

class GameCharacter < Prototype
  attr_accessor :name, :level, :stats, :equipment

  def initialize(name)
    @name = name
    @level = 1
    @stats = { hp: 100, mp: 50, attack: 20, defense: 15 }
    @equipment = []
  end

  def level_up!
    @level += 1
    @stats[:hp] += 10
    @stats[:mp] += 5
    @stats[:attack] += 3
  end

  def to_s
    "#{@name} (Lv.#{@level}) HP:#{@stats[:hp]} ATK:#{@stats[:attack]}"
  end
end

# Base character template
warrior_template = GameCharacter.new("Warrior")
warrior_template.stats[:hp] = 150
warrior_template.stats[:attack] = 30

# Clone and customize
warrior1 = warrior_template.clone_deep
warrior1.name = "Arthur"
warrior1.level_up!
warrior1.level_up!

warrior2 = warrior_template.clone_deep
warrior2.name = "Lancelot"
warrior2.level_up!

puts warrior_template
puts warrior1
puts warrior2
```

```ruby
# ข้อที่ 17: Composite Pattern
class FileSystemItem
  attr_reader :name

  def initialize(name)
    @name = name
  end

  def size
    raise NotImplementedError
  end

  def display(indent = 0)
    raise NotImplementedError
  end
end

class File < FileSystemItem
  def initialize(name, size_bytes)
    super(name)
    @size_bytes = size_bytes
  end

  def size
    @size_bytes
  end

  def display(indent = 0)
    puts "#{' ' * indent}📄 #{@name} (#{size_formatted})"
  end

  private

  def size_formatted
    if @size_bytes < 1024
      "#{@size_bytes} B"
    elsif @size_bytes < 1024**2
      "#{(@size_bytes / 1024.0).round(1)} KB"
    else
      "#{(@size_bytes / 1024.0**2).round(1)} MB"
    end
  end
end

class Directory < FileSystemItem
  def initialize(name)
    super(name)
    @children = []
  end

  def add(item)
    @children << item
    self
  end

  def size
    @children.sum(&:size)
  end

  def file_count
    @children.sum do |child|
      child.is_a?(Directory) ? child.file_count : 1
    end
  end

  def display(indent = 0)
    puts "#{' ' * indent}📁 #{@name}/ (#{@children.length} items)"
    @children.each { |child| child.display(indent + 2) }
  end
end

root = Directory.new("root")
home = Directory.new("home")
user = Directory.new("alice")
docs = Directory.new("Documents")
photos = Directory.new("Photos")

docs.add(File.new("resume.pdf", 250_000))
    .add(File.new("report.docx", 1_500_000))

photos.add(File.new("vacation.jpg", 3_500_000))
      .add(File.new("family.jpg", 2_800_000))

user.add(docs).add(photos)
    .add(File.new(".bashrc", 2048))

home.add(user)
root.add(home)
    .add(Directory.new("etc").tap { |d| d.add(File.new("config.yaml", 512)) })

root.display
puts "\nTotal size: #{(root.size / 1024.0**2).round(2)} MB"
puts "Total files: #{root.file_count}"
```

```ruby
# ข้อที่ 18-25: Remaining exercises

# ข้อที่ 18: Iterator Pattern
class NumberRange
  include Enumerable

  def initialize(start, stop, step = 1)
    @start = start
    @stop = stop
    @step = step
  end

  def each
    current = @start
    while current <= @stop
      yield current
      current += @step
    end
  end
end

range = NumberRange.new(1, 20, 2)
puts range.to_a.inspect
puts "Sum: #{range.sum}"
puts "Evens only: #{range.select(&:even?).inspect}"

# ข้อที่ 19: Builder Pattern
class QueryBuilder
  def initialize(table)
    @table = table
    @conditions = []
    @columns = ["*"]
    @order = nil
    @limit = nil
    @offset = 0
  end

  def select(*columns)
    @columns = columns
    self
  end

  def where(condition)
    @conditions << condition
    self
  end

  def order(column, direction = :asc)
    @order = "#{column} #{direction.to_s.upcase}"
    self
  end

  def limit(n)
    @limit = n
    self
  end

  def offset(n)
    @offset = n
    self
  end

  def build
    sql = "SELECT #{@columns.join(', ')} FROM #{@table}"
    sql += " WHERE #{@conditions.join(' AND ')}" unless @conditions.empty?
    sql += " ORDER BY #{@order}" if @order
    sql += " LIMIT #{@limit}" if @limit
    sql += " OFFSET #{@offset}" if @offset > 0
    sql
  end

  def to_s
    build
  end
end

query = QueryBuilder.new("users")
  .select("id", "name", "email")
  .where("age > 18")
  .where("active = true")
  .order(:name)
  .limit(10)
  .offset(20)
  .build

puts query
```

```ruby
# ข้อที่ 20: Chain of Responsibility
class Handler
  attr_writer :next_handler

  def handle(request)
    if can_handle?(request)
      process(request)
    elsif @next_handler
      @next_handler.handle(request)
    else
      puts "No handler found for: #{request}"
    end
  end

  def then(handler)
    @next_handler = handler
    handler
  end

  private

  def can_handle?(request)
    raise NotImplementedError
  end

  def process(request)
    raise NotImplementedError
  end
end

class LowPriorityHandler < Handler
  private
  def can_handle?(request); request[:priority] == :low; end
  def process(request); puts "Low priority handler: #{request[:message]}"; end
end

class MediumPriorityHandler < Handler
  private
  def can_handle?(request); request[:priority] == :medium; end
  def process(request); puts "Medium priority handler: #{request[:message]}"; end
end

class HighPriorityHandler < Handler
  private
  def can_handle?(request); request[:priority] == :high; end
  def process(request); puts "HIGH PRIORITY HANDLER: #{request[:message]}"; end
end

low = LowPriorityHandler.new
medium = MediumPriorityHandler.new
high = HighPriorityHandler.new

low.then(medium).then(high)

requests = [
  { priority: :low, message: "Update user profile" },
  { priority: :high, message: "SYSTEM CRASH!" },
  { priority: :medium, message: "Generate monthly report" },
  { priority: :critical, message: "Unknown event" },
]

requests.each { |r| low.handle(r) }
```

---

## สรุปตอนที่ 12

ในตอนนี้เราได้เรียนรู้:

1. **Single Inheritance** - `class Child < Parent`, ทำไมต้องใช้ inheritance
2. **super keyword** - ส่ง arguments ต่างๆ, `super`, `super()`, `super(args)`
3. **Method Overriding** - redefine methods ใน subclass, polymorphism
4. **super ใน practice** - เพิ่มเติมจาก parent
5. **Protected methods** - เรียกได้ใน subclass
6. **Private methods** - สืบทอดไป subclass, เรียกได้ภายใน
7. **Abstract class pattern** - `NotImplementedError`
8. **ancestors chain** - method lookup path
9. **is_a? / kind_of? / instance_of?** - type checking
10. **Class methods** - สืบทอดและ override
11. **Template Method Pattern** - กำหนด algorithm structure
12. **Complete Shape Hierarchy** - Circle, Rectangle, Square, Triangle, RegularPolygon
13. **Design Patterns** - Observer, Command, Strategy, Decorator, Composite

---

*ตอนต่อไป: Part 13 - Modules และ Mixins*
