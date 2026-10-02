# Part 13: Modules and Mixins - Steps 256-280

## บทนำ

Module เป็นหนึ่งในความสามารถที่ทรงพลังที่สุดของ Ruby เราใช้ module เพื่อ:
1. **Namespacing** - จัดระเบียบโค้ดและป้องกันชื่อชนกัน
2. **Mixins** - แชร์พฤติกรรมระหว่าง classes ที่ไม่มีความสัมพันธ์กัน

---

## Step 256: Module Definition

```ruby
# สร้าง Module
module Greetable
  def greet
    "สวัสดี! ฉันชื่อ #{name}"
  end
  
  def farewell
    "ลาก่อน! จาก #{name}"
  end
end

module Trackable
  def log_access(resource)
    puts "[LOG] #{name} เข้าถึง #{resource} เมื่อ #{Time.now}"
  end
end

# Module ไม่สามารถสร้าง instance ได้โดยตรง
# Greetable.new  # => NoMethodError!

# แต่สามารถ include ใน class ได้
class User
  include Greetable
  include Trackable
  
  attr_reader :name, :email
  
  def initialize(name, email)
    @name = name
    @email = email
  end
end

class Admin
  include Greetable
  
  attr_reader :name, :role
  
  def initialize(name, role)
    @name = name
    @role = role
  end
end

user = User.new("สมชาย", "somchai@example.com")
admin = Admin.new("ผู้ดูแล", "superadmin")

puts user.greet     # => สวัสดี! ฉันชื่อ สมชาย
puts admin.greet    # => สวัสดี! ฉันชื่อ ผู้ดูแล
puts user.farewell  # => ลาก่อน! จาก สมชาย
user.log_access("dashboard")
```

### Module Constants

```ruby
module MathConstants
  PI = Math::PI
  E = Math::E
  GOLDEN_RATIO = (1 + Math.sqrt(5)) / 2.0
  
  def circle_area(r)
    PI * r ** 2
  end
  
  def fibonacci_ratio
    GOLDEN_RATIO
  end
end

puts MathConstants::PI           # => 3.141592653589793
puts MathConstants::GOLDEN_RATIO # => 1.618033988749895

class Geometry
  include MathConstants
  
  def sphere_volume(r)
    (4.0 / 3) * PI * r ** 3
  end
end

g = Geometry.new
puts g.circle_area(5).round(2)   # => 78.54
puts g.sphere_volume(3).round(2) # => 113.1
```

---

## Step 257: Namespacing กับ Modules

### ป้องกัน Name Conflicts

```ruby
module Animals
  class Dog
    def speak
      "โฮ่งๆ! (สุนัข)"
    end
  end
  
  class Cat
    def speak
      "เมี้ยวๆ! (แมว)"
    end
  end
end

module Robots
  class Dog
    def speak
      "BIP BOP BIP! (หุ่นยนต์สุนัข)"
    end
  end
end

animals_dog = Animals::Dog.new
robots_dog = Robots::Dog.new

puts animals_dog.speak  # => โฮ่งๆ! (สุนัข)
puts robots_dog.speak   # => BIP BOP BIP! (หุ่นยนต์สุนัข)
```

### Nested Modules

```ruby
module Company
  module HR
    class Employee
      attr_reader :name, :salary
      
      def initialize(name, salary)
        @name = name
        @salary = salary
      end
    end
    
    class Manager < Employee
      def initialize(name, salary, team_size)
        super(name, salary)
        @team_size = team_size
      end
      
      def to_s
        "Manager: #{@name} (ทีม #{@team_size} คน)"
      end
    end
  end
  
  module Finance
    class Invoice
      attr_reader :amount, :due_date
      
      def initialize(amount, due_date)
        @amount = amount
        @due_date = due_date
      end
      
      def overdue?
        Time.now > @due_date
      end
      
      def to_s
        "ใบแจ้งหนี้: #{@amount} บาท"
      end
    end
    
    class Budget
      def initialize(total)
        @total = total
        @expenses = []
      end
      
      def add_expense(name, amount)
        @expenses << { name: name, amount: amount }
      end
      
      def remaining
        @total - @expenses.sum { |e| e[:amount] }
      end
    end
  end
end

manager = Company::HR::Manager.new("สมชาย", 80000, 5)
puts manager

invoice = Company::Finance::Invoice.new(50000, Time.now + 86400 * 30)
puts invoice

budget = Company::Finance::Budget.new(1000000)
budget.add_expense("เงินเดือน", 500000)
budget.add_expense("ค่าเช่า", 100000)
puts "งบที่เหลือ: #{budget.remaining} บาท"
```

---

## Step 258: include vs extend vs prepend

### include - เพิ่ม methods เป็น Instance Methods

```ruby
module Greetable
  def greet
    "สวัสดี, ฉันชื่อ #{name}"
  end
end

class Person
  include Greetable
  
  attr_reader :name
  
  def initialize(name)
    @name = name
  end
end

p = Person.new("สมชาย")
puts p.greet         # => เรียกเป็น instance method
# Person.greet       # => NoMethodError
```

### extend - เพิ่ม methods เป็น Class Methods (หรือ Singleton Methods)

```ruby
module Findable
  def find(id)
    puts "ค้นหา #{name} id=#{id}"
    # ใน real app จะ query database
  end
  
  def count
    puts "นับจำนวน #{name}"
  end
  
  def all
    puts "ดึงข้อมูล #{name} ทั้งหมด"
  end
end

class User
  extend Findable
  
  attr_reader :name
  
  def initialize(name)
    @name = name
  end
end

class Product
  extend Findable
end

User.find(1)      # => ค้นหา User id=1
User.count        # => นับจำนวน User
Product.all       # => ดึงข้อมูล Product ทั้งหมด

# extend กับ object เดี่ยว
module Speakable
  def speak
    "ฉันพูดได้!"
  end
end

obj = Object.new
obj.extend(Speakable)
puts obj.speak   # => ฉันพูดได้!
# Object.new.speak  # => NoMethodError (object อื่นไม่มี)
```

### prepend - เพิ่ม module ก่อน class ใน ancestors chain

```ruby
module Logging
  def save
    puts "[LOG] กำลังบันทึก #{self.class.name}..."
    result = super
    puts "[LOG] บันทึกสำเร็จ"
    result
  end
  
  def delete
    puts "[LOG] กำลังลบ #{self.class.name}..."
    result = super
    puts "[LOG] ลบสำเร็จ"
    result
  end
end

class ActiveRecord
  def save
    puts "บันทึกลง DB"
    true
  end
  
  def delete
    puts "ลบจาก DB"
    true
  end
end

class User < ActiveRecord
  prepend Logging  # module อยู่ก่อน User ใน chain
  
  attr_reader :name
  
  def initialize(name)
    @name = name
  end
end

u = User.new("สมชาย")
u.save
puts "\n"
u.delete

puts "\nAncestors:"
puts User.ancestors.inspect
# => [Logging, User, ActiveRecord, Object, ...]
```

---

## Step 259: Mixin Patterns

### Comparable Mixin

```ruby
class Temperature
  include Comparable
  
  attr_reader :value, :unit
  
  def initialize(value, unit = :celsius)
    @value = value.to_f
    @unit = unit
  end
  
  def to_celsius
    case @unit
    when :celsius    then @value
    when :fahrenheit then (@value - 32) * 5.0 / 9
    when :kelvin     then @value - 273.15
    end
  end
  
  def <=>(other)
    to_celsius <=> other.to_celsius
  end
  
  def to_s
    "#{@value}°#{@unit.to_s[0].upcase}"
  end
end

temps = [
  Temperature.new(100, :celsius),
  Temperature.new(32, :fahrenheit),
  Temperature.new(300, :kelvin),
  Temperature.new(-10, :celsius),
  Temperature.new(373.15, :kelvin)
]

puts "เรียงลำดับ:"
temps.sort.each { |t| puts "  #{t} = #{t.to_celsius.round(2)}°C" }

puts "\nร้อนที่สุด: #{temps.max}"
puts "เย็นที่สุด: #{temps.min}"
puts "100°C > 32°F? #{Temperature.new(100, :celsius) > Temperature.new(32, :fahrenheit)}"
```

### Enumerable Mixin

```ruby
class NumberList
  include Enumerable
  
  def initialize(*numbers)
    @numbers = numbers
  end
  
  def each(&block)
    @numbers.each(&block)
  end
  
  def to_s
    "NumberList(#{@numbers.join(', ')})"
  end
end

list = NumberList.new(3, 1, 4, 1, 5, 9, 2, 6, 5, 3)

# Enumerable methods ทั้งหมดใช้ได้!
puts list.sort.inspect          # => [1, 1, 2, 3, 3, 4, 5, 5, 6, 9]
puts list.min                   # => 1
puts list.max                   # => 9
puts list.sum                   # => 39
puts list.count                 # => 10
puts list.select(&:odd?).inspect    # => [3, 1, 1, 5, 9, 5, 3]
puts list.map { |n| n * 2 }.inspect # => [6, 2, 8, ...]
puts list.sort_by { |n| -n }.first(3).inspect  # => [9, 6, 5]
puts list.min_by { |n| n }.inspect  # => 1
puts list.group_by { |n| n % 2 == 0 ? :even : :odd }.keys.inspect
puts list.any? { |n| n > 8 }   # => true
puts list.all? { |n| n > 0 }   # => true
puts list.none? { |n| n < 0 }  # => true
puts list.include?(5)           # => true
puts list.find { |n| n > 5 }   # => 9
puts list.reduce(:+)            # => 39
```

---

## Step 260: module_function

`module_function` สร้าง method ที่ใช้ได้ทั้งแบบ module method และ instance method

```ruby
module StringUtils
  module_function
  
  def slugify(text)
    text.downcase.gsub(/[^a-z0-9\s]/, '').gsub(/\s+/, '-')
  end
  
  def truncate(text, length = 50, suffix = "...")
    text.length > length ? text[0, length] + suffix : text
  end
  
  def word_count(text)
    text.split.length
  end
  
  def titleize(text)
    text.split.map(&:capitalize).join(" ")
  end
  
  def camel_case(text)
    parts = text.split(/[_\s]/)
    parts.first + parts[1..].map(&:capitalize).join
  end
  
  def snake_case(text)
    text.gsub(/([A-Z])/, '_\1').downcase.sub(/^_/, '')
  end
end

# ใช้เป็น module method
puts StringUtils.slugify("Hello World! This is Ruby.")  # => hello-world-this-is-ruby
puts StringUtils.truncate("สวัสดีชาวโลก Ruby Programming is awesome!", 20)
puts StringUtils.titleize("hello world ruby")
puts StringUtils.camel_case("my_variable_name")  # => myVariableName
puts StringUtils.snake_case("MyVariableName")     # => my_variable_name

# include ใน class แล้วใช้เป็น instance method
class TextProcessor
  include StringUtils
  
  def process(text)
    {
      slug: slugify(text),
      truncated: truncate(text, 20),
      words: word_count(text),
      title: titleize(text)
    }
  end
end

tp = TextProcessor.new
result = tp.process("Hello World Ruby Programming")
result.each { |k, v| puts "#{k}: #{v}" }
```

---

## Step 261: Comparable Module รายละเอียด

```ruby
class Card
  include Comparable
  
  SUITS = { spades: 4, hearts: 3, diamonds: 2, clubs: 1 }
  VALUES = { ace: 14, king: 13, queen: 12, jack: 11 }
  (2..10).each { |n| VALUES[n] = n }
  
  attr_reader :value, :suit
  
  def initialize(value, suit)
    @value = value
    @suit = suit.to_sym
    raise ArgumentError, "ค่าไม่ถูกต้อง" unless VALUES.key?(@value)
    raise ArgumentError, "ดอกไพ่ไม่ถูกต้อง" unless SUITS.key?(@suit)
  end
  
  def numeric_value
    VALUES[@value]
  end
  
  def suit_value
    SUITS[@suit]
  end
  
  def <=>(other)
    comparison = numeric_value <=> other.numeric_value
    comparison != 0 ? comparison : suit_value <=> other.suit_value
  end
  
  def to_s
    suit_symbol = { spades: "♠", hearts: "♥", diamonds: "♦", clubs: "♣" }[@suit]
    "#{@value}#{suit_symbol}"
  end
end

cards = [
  Card.new(:king, :hearts),
  Card.new(:ace, :spades),
  Card.new(10, :diamonds),
  Card.new(:queen, :clubs),
  Card.new(:ace, :hearts)
]

puts "ไพ่ทั้งหมด: #{cards.map(&:to_s).join(', ')}"
puts "เรียงลำดับ: #{cards.sort.map(&:to_s).join(', ')}"
puts "ไพ่ใหญ่สุด: #{cards.max}"
puts "ไพ่เล็กสุด: #{cards.min}"

# between?
card = Card.new(7, :hearts)
low = Card.new(5, :clubs)
high = Card.new(10, :spades)
puts "7♥ อยู่ระหว่าง 5♣ และ 10♠? #{card.between?(low, high)}"
```

---

## Step 262: Enumerable Module รายละเอียด

```ruby
class Inventory
  include Enumerable
  
  Item = Struct.new(:name, :price, :quantity, :category) do
    def total_value
      price * quantity
    end
    
    def to_s
      "#{name} (#{category}): #{price}บาท x#{quantity} = #{total_value}บาท"
    end
  end
  
  def initialize
    @items = []
  end
  
  def add(name, price, quantity, category)
    @items << Item.new(name, price, quantity, category)
    self
  end
  
  def each(&block)
    @items.each(&block)
  end
  
  def total_value
    sum(&:total_value)
  end
  
  def by_category
    group_by(&:category)
  end
  
  def expensive(threshold = 1000)
    select { |item| item.price > threshold }
  end
  
  def low_stock(threshold = 5)
    select { |item| item.quantity < threshold }
  end
end

inv = Inventory.new
inv.add("MacBook Pro", 59900, 3, "คอมพิวเตอร์")
   .add("iPad", 25900, 8, "แท็บเล็ต")
   .add("iPhone 15", 32900, 15, "โทรศัพท์")
   .add("AirPods", 7900, 2, "อุปกรณ์เสริม")
   .add("Magic Mouse", 2900, 4, "อุปกรณ์เสริม")
   .add("USB Hub", 890, 20, "อุปกรณ์เสริม")

puts "=== Inventory Report ==="
puts "รายการทั้งหมด: #{inv.count} รายการ"
puts "มูลค่ารวม: #{inv.total_value} บาท"

puts "\nราคาแพงสุด: #{inv.max_by(&:price).name}"
puts "ราคาถูกสุด: #{inv.min_by(&:price).name}"

puts "\nสินค้าราคาเกิน 10,000:"
inv.expensive(10000).each { |item| puts "  #{item.name}: #{item.price} บาท" }

puts "\nสินค้าที่เหลือน้อย (< 5 ชิ้น):"
inv.low_stock(5).each { |item| puts "  #{item.name}: #{item.quantity} ชิ้น" }

puts "\nจัดกลุ่มตามหมวดหมู่:"
inv.by_category.each do |category, items|
  total = items.sum(&:total_value)
  puts "  #{category}: #{items.length} รายการ (มูลค่า #{total} บาท)"
end

puts "\nTop 3 มูลค่ารวมสูงสุด:"
inv.sort_by { |item| -item.total_value }
   .first(3)
   .each_with_index { |item, i| puts "  #{i+1}. #{item}" }
```

---

## Step 263: Forwardable Module

```ruby
require 'forwardable'

class Stack
  extend Forwardable
  
  def_delegators :@data, :push, :pop, :last, :empty?, :size, :length
  def_delegator :@data, :push, :add     # alias push ว่า add
  def_delegator :@data, :last, :peek    # alias last ว่า peek
  
  def initialize
    @data = []
  end
  
  def to_s
    "Stack#{@data.inspect}"
  end
end

stack = Stack.new
stack.push(1)
stack.add(2)   # alias สำหรับ push
stack.push(3)

puts stack.peek   # => 3
puts stack.size   # => 3
puts stack.pop    # => 3
puts stack.empty? # => false
puts stack

# ตัวอย่างที่ใช้งานจริง
class Printer
  extend Forwardable
  
  def_delegators :@queue, :size, :empty?
  
  def initialize
    @queue = []
    @printed = []
  end
  
  def add_job(document)
    @queue << document
    puts "เพิ่มงานพิมพ์: #{document}"
    self
  end
  
  def print_next
    return "ไม่มีงานพิมพ์" if @queue.empty?
    doc = @queue.shift
    @printed << doc
    "พิมพ์: #{doc}"
  end
  
  def print_all
    until @queue.empty?
      puts print_next
    end
  end
  
  def to_s
    "Printer (คิว: #{size} งาน, พิมพ์แล้ว: #{@printed.length} งาน)"
  end
end

printer = Printer.new
printer.add_job("รายงานประจำเดือน.pdf")
        .add_job("ใบเสนอราคา.docx")
        .add_job("รูปภาพ.jpg")

puts printer
printer.print_all
puts printer
```

---

## Step 264: Composition over Inheritance

```ruby
# แทนที่จะสืบทอด ให้ใช้ composition

module Persistable
  def save
    puts "บันทึก #{self.class.name} ลงฐานข้อมูล"
    @saved = true
    self
  end
  
  def delete
    puts "ลบ #{self.class.name} จากฐานข้อมูล"
    @saved = false
    self
  end
  
  def saved?
    @saved || false
  end
end

module Validatable
  def valid?
    validate.empty?
  end
  
  def validate
    []  # subclass override
  end
  
  def validate!
    errors = validate
    raise "Validation failed: #{errors.join(', ')}" unless errors.empty?
    self
  end
end

module Auditable
  def audit_log
    @audit_log ||= []
  end
  
  def log_change(field, old_value, new_value)
    audit_log << {
      field: field,
      old: old_value,
      new: new_value,
      at: Time.now
    }
  end
  
  def changes
    audit_log.map { |entry| "#{entry[:field]}: #{entry[:old]} -> #{entry[:new]}" }
  end
end

module Serializable
  def to_json
    require 'json'
    instance_variables.each_with_object({}) do |var, hash|
      hash[var.to_s.delete('@')] = instance_variable_get(var)
    end.to_json
  end
  
  def to_hash
    instance_variables.each_with_object({}) do |var, hash|
      hash[var.to_s.delete('@').to_sym] = instance_variable_get(var)
    end
  end
end

class User
  include Persistable
  include Validatable
  include Auditable
  include Serializable
  
  attr_reader :name, :email
  
  def initialize(name, email)
    @name = name
    @email = email
  end
  
  def name=(new_name)
    log_change(:name, @name, new_name)
    @name = new_name
  end
  
  def email=(new_email)
    log_change(:email, @email, new_email)
    @email = new_email
  end
  
  def validate
    errors = []
    errors << "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร" if @name.to_s.length < 2
    errors << "อีเมลไม่ถูกต้อง" unless @email.to_s.include?("@")
    errors
  end
  
  def to_s
    "User(#{@name}, #{@email})"
  end
end

user = User.new("สมชาย", "somchai@example.com")
puts user.valid?          # => true
user.save

user.name = "สมชาย ใหม่"
user.email = "new@example.com"

puts "\nการเปลี่ยนแปลง:"
user.changes.each { |c| puts "  #{c}" }

puts "\n#{user.to_json}"

bad_user = User.new("x", "not-an-email")
puts "\nValid? #{bad_user.valid?}"
puts "Errors: #{bad_user.validate.inspect}"
```

---

## Step 265: Building Greetable Mixin

```ruby
module Greetable
  def self.included(base)
    puts "Greetable included ใน #{base.name}"
    base.extend(ClassMethods)
    base.instance_variable_set(:@greeting_format, "สวัสดี")
  end
  
  module ClassMethods
    def greeting_format
      @greeting_format
    end
    
    def set_greeting(format)
      @greeting_format = format
    end
  end
  
  def greet(target = nil)
    format = self.class.greeting_format
    if target
      "#{format}, #{target}! ฉันชื่อ #{greeting_name}"
    else
      "#{format}! ฉันชื่อ #{greeting_name}"
    end
  end
  
  def formal_greet(title, target)
    "#{self.class.greeting_format}ท่าน#{title}#{target}, ข้าพเจ้าชื่อ #{greeting_name}"
  end
  
  def farewell
    "ลาก่อน! จาก #{greeting_name}"
  end
  
  private
  
  def greeting_name
    respond_to?(:name) ? name : self.class.name
  end
end

class Person
  include Greetable
  
  attr_reader :name
  
  def initialize(name)
    @name = name
  end
end

class Robot
  include Greetable
  
  set_greeting "Hello"  # ใช้ ClassMethod ที่ inject มา
  
  def name
    "Robot-#{object_id}"
  end
end

class FormalPerson
  include Greetable
  
  set_greeting "กราบเรียน"
  
  attr_reader :name
  
  def initialize(name)
    @name = name
  end
end

p = Person.new("สมชาย")
r = Robot.new
fp = FormalPerson.new("สมศักดิ์")

puts p.greet                    # => สวัสดี! ฉันชื่อ สมชาย
puts p.greet("สมหญิง")          # => สวัสดี, สมหญิง! ฉันชื่อ สมชาย
puts r.greet                    # => Hello! ฉันชื่อ Robot-xxx
puts fp.formal_greet("ท่าน", "นายก")  # => กราบเรียนท่านท่านนายก...
```

---

## Step 266: Building Serializable Mixin

```ruby
module Serializable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@serializable_fields, [])
  end
  
  module ClassMethods
    def serializable_fields
      @serializable_fields ||= []
    end
    
    def serialize(*fields)
      @serializable_fields ||= []
      @serializable_fields.concat(fields)
    end
    
    def from_hash(hash)
      obj = allocate
      hash.each do |key, value|
        setter = "#{key}="
        obj.send(setter, value) if obj.respond_to?(setter)
      end
      obj
    end
    
    def from_json(json_string)
      require 'json'
      from_hash(JSON.parse(json_string, symbolize_names: true))
    end
  end
  
  def to_hash
    fields = self.class.serializable_fields
    if fields.empty?
      instance_variables.each_with_object({}) do |var, hash|
        key = var.to_s.delete('@').to_sym
        hash[key] = instance_variable_get(var)
      end
    else
      fields.each_with_object({}) do |field, hash|
        hash[field] = send(field) if respond_to?(field)
      end
    end
  end
  
  def to_json
    require 'json'
    to_hash.to_json
  end
  
  def to_yaml_string
    require 'yaml'
    to_hash.transform_keys(&:to_s).to_yaml
  end
  
  def ==(other)
    return false unless self.class == other.class
    to_hash == other.to_hash
  end
end

class Product
  include Serializable
  
  serialize :name, :price, :category, :in_stock
  
  attr_accessor :name, :price, :category, :in_stock
  
  def initialize(name, price, category, in_stock = true)
    @name = name
    @price = price
    @category = category
    @in_stock = in_stock
  end
  
  def to_s
    "#{@name} (#{@price} บาท)"
  end
end

p1 = Product.new("MacBook Pro", 59900, "คอมพิวเตอร์")
puts p1.to_hash.inspect
puts p1.to_json

json_str = '{"name":"iPhone","price":32900,"category":"โทรศัพท์","in_stock":true}'
p2 = Product.from_json(json_str)
puts p2

puts p1 == p2   # => false
p3 = Product.new("MacBook Pro", 59900, "คอมพิวเตอร์")
puts p1 == p3   # => true
```

---

## Step 267: Module Hooks

```ruby
module Observable
  def self.included(base)
    puts "[HOOK] Observable included ใน #{base.name}"
    base.extend(ClassMethods)
    base.instance_variable_set(:@observers, [])
  end
  
  def self.extended(base)
    puts "[HOOK] Observable extended ใน #{base.inspect}"
  end
  
  def self.prepended(base)
    puts "[HOOK] Observable prepended ใน #{base.name}"
  end
  
  module ClassMethods
    def observers
      @observers ||= []
    end
    
    def add_observer(observer)
      @observers ||= []
      @observers << observer
    end
  end
  
  def notify_observers(event, data = {})
    self.class.observers.each do |observer|
      observer.update(event, data) if observer.respond_to?(:update)
    end
  end
  
  def changed(field, old_value, new_value)
    notify_observers(:changed, {
      field: field,
      old: old_value,
      new: new_value,
      object: self
    })
  end
end

class AuditObserver
  def update(event, data)
    puts "[AUDIT] Event: #{event}, Field: #{data[:field]}, #{data[:old]} -> #{data[:new]}"
  end
end

class LogObserver
  def update(event, data)
    puts "[LOG] #{data[:object].class.name} #{event}: #{data[:field]} changed"
  end
end

class User
  include Observable
  
  attr_reader :name, :email
  
  def initialize(name, email)
    @name = name
    @email = email
  end
  
  def name=(new_name)
    changed(:name, @name, new_name)
    @name = new_name
  end
  
  def email=(new_email)
    changed(:email, @email, new_email)
    @email = new_email
  end
end

User.add_observer(AuditObserver.new)
User.add_observer(LogObserver.new)

u = User.new("สมชาย", "old@example.com")
u.name = "สมชาย ใหม่"
u.email = "new@example.com"
```

### inherited Hook

```ruby
class Plugin
  @@plugins = []
  
  def self.inherited(subclass)
    puts "Plugin ใหม่: #{subclass.name}"
    @@plugins << subclass
  end
  
  def self.all_plugins
    @@plugins
  end
  
  def execute
    raise NotImplementedError
  end
end

class AuthPlugin < Plugin
  def execute
    "ตรวจสอบการยืนยันตัวตน"
  end
end

class LogPlugin < Plugin
  def execute
    "บันทึก log"
  end
end

class CachePlugin < Plugin
  def execute
    "จัดการ cache"
  end
end

puts "\nPlugins ที่ลงทะเบียน:"
Plugin.all_plugins.each { |p| puts "  #{p.name}" }

puts "\nรันทุก Plugin:"
Plugin.all_plugins.each do |plugin_class|
  plugin = plugin_class.new
  puts "  #{plugin_class.name}: #{plugin.execute}"
end
```

---

## Step 268-280: เทคนิคขั้นสูง

### Mixin ด้วย ActiveSupport-style Concerns

```ruby
module Concern
  def self.extended(base)
    base.instance_variable_set(:@_dependencies, [])
  end
  
  def included(base = nil, &block)
    if base.nil?
      @_included_block = block
    else
      super
      base.extend(const_get(:ClassMethods)) if const_defined?(:ClassMethods)
      base.class_eval(&@_included_block) if @_included_block
    end
  end
end

module Timestamps
  extend Concern
  
  included do
    attr_reader :created_at, :updated_at
    
    def initialize_timestamps
      @created_at = Time.now
      @updated_at = Time.now
    end
  end
  
  module ClassMethods
    def oldest
      # implementation
    end
    
    def newest
      # implementation
    end
  end
  
  def touch
    @updated_at = Time.now
  end
  
  def age
    Time.now - (@created_at || Time.now)
  end
end

class Article
  include Timestamps
  
  attr_accessor :title, :body
  
  def initialize(title, body)
    @title = title
    @body = body
    initialize_timestamps
  end
  
  def update(new_body)
    @body = new_body
    touch
  end
  
  def to_s
    "#{@title} (สร้าง: #{@created_at&.strftime('%Y-%m-%d')})"
  end
end

article = Article.new("Ruby is Awesome", "เนื้อหาบทความ...")
puts article
sleep(0.01)
article.update("เนื้อหาที่แก้ไขแล้ว")
puts "Updated: #{article.updated_at}"
```

### Duck Typing กับ Modules

```ruby
module Quackable
  def quack
    raise NotImplementedError, "#{self.class}#quack ต้อง implement"
  end
end

class Duck
  include Quackable
  
  def quack
    "Quack!"
  end
end

class Person
  include Quackable
  
  def quack
    "ฉันทำเสียงเป็ดได้! ...Quack?"
  end
end

class RubberDuck
  include Quackable
  
  def quack
    "Squeeeeak!"
  end
end

def make_it_quack(quacker)
  puts "#{quacker.class}: #{quacker.quack}"
end

[Duck.new, Person.new, RubberDuck.new].each do |q|
  make_it_quack(q)
end
```

### Module กับ Method Wrapping

```ruby
module Memoizable
  def memoize(method_name)
    original = instance_method(method_name)
    cache_var = :"@_memo_#{method_name}"
    
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

class ExpensiveCalculator
  extend Memoizable
  
  def fibonacci(n)
    puts "  คำนวณ fibonacci(#{n})..."
    return n if n <= 1
    fibonacci(n - 1) + fibonacci(n - 2)
  end
  
  memoize :fibonacci
end

calc = ExpensiveCalculator.new
puts "fibonacci(10):"
puts calc.fibonacci(10)
puts "\nคำนวณอีกครั้ง (จาก cache):"
puts calc.fibonacci(10)
```

---

## แบบฝึกหัด Part 13 (25 ข้อ)

### ระดับง่าย (ข้อ 1-8)

**ข้อ 1:** สร้าง Printable module ด้วย `print_info`, `print_summary`, `print_full`

```ruby
# เฉลย
module Printable
  def print_info
    puts to_s
  end
  
  def print_summary
    data = to_hash rescue { info: to_s }
    puts data.map { |k, v| "#{k}: #{v}" }.join(", ")
  end
  
  def print_full
    if respond_to?(:to_hash)
      hash = to_hash
      max_key = hash.keys.map(&:to_s).map(&:length).max
      puts "-" * (max_key + 20)
      hash.each do |k, v|
        puts "#{k.to_s.ljust(max_key)}: #{v}"
      end
      puts "-" * (max_key + 20)
    else
      puts inspect
    end
  end
end

class Person
  include Printable
  
  attr_reader :name, :age, :email
  
  def initialize(name, age, email)
    @name = name
    @age = age
    @email = email
  end
  
  def to_s
    "#{@name} (#{@age})"
  end
  
  def to_hash
    { name: @name, age: @age, email: @email }
  end
end

p = Person.new("สมชาย", 25, "somchai@example.com")
p.print_info
p.print_summary
p.print_full
```

**ข้อ 2:** สร้าง Cacheable module

```ruby
# เฉลย
module Cacheable
  def self.included(base)
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def cache
      @cache ||= {}
    end
    
    def cache_get(key)
      cache[key]
    end
    
    def cache_set(key, value, ttl = nil)
      cache[key] = {
        value: value,
        expires_at: ttl ? Time.now + ttl : nil
      }
      value
    end
    
    def cache_valid?(key)
      entry = cache[key]
      return false unless entry
      return true if entry[:expires_at].nil?
      Time.now < entry[:expires_at]
    end
    
    def cache_fetch(key, ttl = nil)
      return cache[key][:value] if cache_valid?(key)
      value = yield
      cache_set(key, value, ttl)
      value
    end
    
    def cache_clear
      @cache = {}
    end
  end
end

class UserService
  include Cacheable
  
  def self.find_user(id)
    cache_fetch("user_#{id}", 300) do
      puts "ดึงข้อมูลจาก DB user ##{id}..."
      { id: id, name: "ผู้ใช้ #{id}", email: "user#{id}@example.com" }
    end
  end
end

puts UserService.find_user(1).inspect
puts UserService.find_user(1).inspect  # จาก cache
puts UserService.find_user(2).inspect
```

**ข้อ 3-8:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 3: สร้าง Taggable module สำหรับเพิ่ม tags ให้ objects
# ข้อ 4: สร้าง Timestampable module สำหรับ created_at/updated_at
# ข้อ 5: สร้าง Notifiable module สำหรับส่ง notifications
# ข้อ 6: สร้าง Exportable module ที่ export ข้อมูลเป็น CSV, JSON, YAML
# ข้อ 7: สร้าง Filterable module สำหรับกรองข้อมูล
# ข้อ 8: สร้าง Paginatable module
```

### ระดับกลาง (ข้อ 9-17)

**ข้อ 9:** สร้าง FullFeatured ORM-like Module

```ruby
# เฉลย
module ActiveModel
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@columns, {})
    base.instance_variable_set(:@records, [])
  end
  
  module ClassMethods
    def column(name, type = :string, default: nil, required: false)
      @columns ||= {}
      @columns[name] = { type: type, default: default, required: required }
      
      attr_accessor name
    end
    
    def columns
      @columns || {}
    end
    
    def records
      @records ||= []
    end
    
    def create(attrs = {})
      obj = new(attrs)
      obj.save
      obj
    end
    
    def all
      records.dup
    end
    
    def find(id)
      records.find { |r| r.id == id }
    end
    
    def where(conditions = {})
      records.select do |record|
        conditions.all? { |key, value| record.send(key) == value }
      end
    end
    
    def count
      records.length
    end
    
    def first
      records.first
    end
    
    def last
      records.last
    end
    
    def next_id
      @next_id ||= 0
      @next_id += 1
    end
  end
  
  attr_reader :id
  
  def initialize(attrs = {})
    @id = self.class.next_id
    
    # Set defaults
    self.class.columns.each do |name, options|
      instance_variable_set("@#{name}", options[:default])
    end
    
    # Set provided attributes
    attrs.each do |key, value|
      send("#{key}=", value) if respond_to?("#{key}=")
    end
  end
  
  def save
    self.class.records << self unless self.class.records.include?(self)
    self
  end
  
  def destroy
    self.class.records.delete(self)
    self
  end
  
  def valid?
    self.class.columns.all? do |name, options|
      !options[:required] || !send(name).nil?
    end
  end
  
  def to_hash
    self.class.columns.keys.each_with_object({ id: @id }) do |col, hash|
      hash[col] = send(col)
    end
  end
  
  def to_s
    attrs = to_hash.map { |k, v| "#{k}: #{v.inspect}" }.join(", ")
    "#{self.class.name}(#{attrs})"
  end
end

class User
  include ActiveModel
  
  column :name, :string, required: true
  column :email, :string, required: true
  column :age, :integer, default: 0
  column :active, :boolean, default: true
end

User.create(name: "สมชาย", email: "somchai@example.com", age: 25)
User.create(name: "สมหญิง", email: "somying@example.com", age: 30)
User.create(name: "สมศักดิ์", email: "somsak@example.com", age: 25)

puts "ผู้ใช้ทั้งหมด: #{User.count} คน"
puts "\nผู้ใช้ทั้งหมด:"
User.all.each { |u| puts "  #{u}" }

puts "\nหาผู้ใช้ age=25:"
User.where(age: 25).each { |u| puts "  #{u}" }

puts "\nหาผู้ใช้ id=2:"
puts User.find(2)
```

**ข้อ 10-17:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 10: สร้าง Hookable module สำหรับ before/after callbacks
# ข้อ 11: สร้าง Validatable module แบบ full-featured
# ข้อ 12: สร้าง Trackable module สำหรับ track user actions
# ข้อ 13: สร้าง Configurable module สำหรับ config management
# ข้อ 14: สร้าง Indexable module สำหรับ full-text search
# ข้อ 15: สร้าง Schedulable module สำหรับ scheduled tasks
# ข้อ 16: สร้าง Encryptable module สำหรับ encrypt/decrypt fields
# ข้อ 17: สร้าง Rateable module สำหรับระบบ rating
```

### ระดับยาก (ข้อ 18-25)

**ข้อ 18:** สร้าง Plugin System ด้วย Modules

```ruby
# เฉลย
module PluginSystem
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@plugins, {})
    base.instance_variable_set(:@hooks, Hash.new { |h, k| h[k] = [] })
  end
  
  module ClassMethods
    def plugins
      @plugins ||= {}
    end
    
    def hooks
      @hooks ||= Hash.new { |h, k| h[k] = [] }
    end
    
    def register_plugin(name, plugin)
      plugins[name] = plugin
      puts "ลงทะเบียน plugin: #{name}"
    end
    
    def hook(event, &block)
      hooks[event] << block
    end
  end
  
  def run_hooks(event, *args)
    self.class.hooks[event].each { |h| h.call(self, *args) }
  end
  
  def plugin(name)
    self.class.plugins[name]
  end
  
  def use_plugin(name, *args)
    p = plugin(name)
    raise "Plugin #{name} ไม่พบ" unless p
    p.call(self, *args)
  end
end

class Application
  include PluginSystem
  
  attr_reader :name, :config
  
  def initialize(name)
    @name = name
    @config = {}
  end
  
  def start
    run_hooks(:before_start)
    puts "เริ่ม Application: #{@name}"
    run_hooks(:after_start)
  end
  
  def stop
    run_hooks(:before_stop)
    puts "หยุด Application: #{@name}"
    run_hooks(:after_stop)
  end
end

# ลงทะเบียน plugins
Application.register_plugin(:logger, ->(app, msg) { puts "[LOG] #{app.name}: #{msg}" })
Application.register_plugin(:metrics, ->(app) { puts "[METRICS] #{app.name} started at #{Time.now}" })

# เพิ่ม hooks
Application.hook(:before_start) { |app| puts "กำลังเตรียม #{app.name}..." }
Application.hook(:after_start) { |app| app.use_plugin(:metrics) }
Application.hook(:before_stop) { |app| app.use_plugin(:logger, "กำลังหยุดทำงาน...") }

app = Application.new("MyApp")
app.start
puts "---"
app.stop
```

**ข้อ 19-25:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 19: สร้าง EventSourcing module
# ข้อ 20: สร้าง Repository pattern ด้วย Modules
# ข้อ 21: สร้าง Circuit Breaker pattern
# ข้อ 22: สร้าง Feature Flags module
# ข้อ 23: สร้าง Permission system ด้วย Modules
# ข้อ 24: สร้าง Retry logic module
# ข้อ 25: สร้าง Multi-tenancy module
```

---

## สรุป Part 13

ใน Part นี้ เราได้เรียนรู้:

1. **Module Definition** - สร้าง module ด้วยคีย์เวิร์ด `module`
2. **Namespacing** - ใช้ module จัดระเบียบโค้ด
3. **include** - เพิ่ม methods เป็น instance methods
4. **extend** - เพิ่ม methods เป็น class methods
5. **prepend** - เพิ่ม module ก่อน class ใน ancestors chain
6. **module_function** - method ที่ใช้ได้ทั้งสองแบบ
7. **Comparable** - module สำหรับเปรียบเทียบ objects
8. **Enumerable** - module สำหรับ iterate collections
9. **Forwardable** - delegate methods ไปยัง objects อื่น
10. **Composition over Inheritance** - แชร์พฤติกรรมด้วย mixins
11. **Module Hooks** - `included`, `extended`, `prepended`, `inherited`

**ต่อไป:** Part 14 - Error Handling
