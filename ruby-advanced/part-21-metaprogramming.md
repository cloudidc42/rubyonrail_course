# Part 21: Metaprogramming ใน Ruby

## ขั้นตอนที่ 436-460: การเขียนโปรแกรมที่เขียนโปรแกรม

---

## บทนำ

Metaprogramming คือการเขียน code ที่สร้าง หรือแก้ไข code อื่นในระหว่าง runtime Ruby เป็นภาษาที่มี metaprogramming ที่ทรงพลังที่สุดภาษาหนึ่ง สิ่งที่ frameworks เช่น Rails ทำได้อย่างน่าอัศจรรย์ ล้วนมาจาก metaprogramming

---

## ขั้นตอนที่ 436: Metaprogramming คืออะไร?

```ruby
# ตัวอย่าง metaprogramming ในชีวิตประจำวัน

# attr_accessor คือ metaprogramming!
class Person
  attr_accessor :name, :age  # สร้าง methods อัตโนมัติ
end

# เทียบเท่ากับ:
class PersonManual
  def name = @name
  def name=(val) = @name = val
  def age = @age
  def age=(val) = @age = val
end

# Rails associations ก็คือ metaprogramming
# class User < ApplicationRecord
#   has_many :posts  # สร้าง posts, posts=, post_ids, etc.
# end

# Metaprogramming ทำให้:
# 1. เขียน code น้อยลง (DRY - Don't Repeat Yourself)
# 2. สร้าง DSL (Domain Specific Language)
# 3. ปรับ behavior ของ class ในขณะ runtime
# 4. สร้าง framework และ library

# Ruby runtime model
puts 42.class              # => Integer
puts Integer.class         # => Class
puts Class.class           # => Class
puts Class.superclass      # => Module
puts Module.class          # => Class

# ทุก class เป็น object ใน Ruby!
puts String.is_a?(Object)  # => true
puts String.is_a?(Class)   # => true
```

---

## ขั้นตอนที่ 437: method_missing

`method_missing` ถูกเรียกเมื่อ call method ที่ไม่มีอยู่

```ruby
class DynamicProxy
  def initialize(target)
    @target = target
  end
  
  def method_missing(method_name, *args, &block)
    if @target.respond_to?(method_name)
      puts "[Proxy] Calling #{method_name} on #{@target.class}"
      @target.send(method_name, *args, &block)
    else
      super  # ส่งต่อให้ parent class จัดการ
    end
  end
  
  def respond_to_missing?(method_name, include_private = false)
    @target.respond_to?(method_name, include_private) || super
  end
end

arr = [1, 2, 3, 4, 5]
proxy = DynamicProxy.new(arr)

puts proxy.length    # => [Proxy] Calling length, => 5
puts proxy.sum       # => [Proxy] Calling sum, => 15
proxy.each { |n| print "#{n} " }  # Proxy ส่งต่อ block ด้วย
puts

# ตัวอย่างจริง: FlexibleHash
class FlexibleHash
  def initialize(data = {})
    @data = data
  end

  def method_missing(name, *args)
    name_str = name.to_s
    
    if name_str.end_with?('=')
      # setter
      @data[name_str[0..-2].to_sym] = args.first
    elsif name_str.end_with?('?')
      # predicate
      !@data[name_str[0..-2].to_sym].nil?
    else
      # getter
      @data[name.to_sym]
    end
  end

  def respond_to_missing?(name, include_private = false)
    true  # ตอบรับทุก method
  end
end

config = FlexibleHash.new
config.host = "localhost"
config.port = 3000

puts config.host         # => "localhost"
puts config.port         # => 3000
puts config.host?        # => true
puts config.email?       # => false
```

---

## ขั้นตอนที่ 438: respond_to_missing?

```ruby
# respond_to? ควรจับคู่กับ method_missing เสมอ

class Ghost
  def method_missing(name, *args)
    if name.to_s.start_with?("ghost_")
      "I'm a ghost #{name}"
    else
      super
    end
  end
  
  def respond_to_missing?(name, include_private = false)
    name.to_s.start_with?("ghost_") || super
  end
end

g = Ghost.new
puts g.ghost_hello        # => "I'm a ghost ghost_hello"
puts g.ghost_world        # => "I'm a ghost ghost_world"

# respond_to? ทำงานถูกต้อง
puts g.respond_to?(:ghost_hello)   # => true
puts g.respond_to?(:real_hello)    # => false

# ตรวจสอบก่อนเรียก (safe pattern)
puts g.respond_to?(:ghost_test) ? g.ghost_test : "no such method"

# ตัวอย่าง: Dynamic Finder (คล้าย Rails)
class User
  attr_accessor :name, :email, :age, :active
  
  def initialize(attrs = {})
    attrs.each { |k, v| send(:"#{k}=", v) }
  end
  
  def self.all
    @users ||= []
  end
  
  def self.create(attrs)
    user = new(attrs)
    all << user
    user
  end
  
  def self.method_missing(name, *args)
    if name.to_s =~ /\Afind_by_(\w+)\z/
      attribute = $1.to_sym
      all.find { |u| u.send(attribute) == args.first }
    elsif name.to_s =~ /\Afind_all_by_(\w+)\z/
      attribute = $1.to_sym
      all.select { |u| u.send(attribute) == args.first }
    else
      super
    end
  end
  
  def self.respond_to_missing?(name, include_private = false)
    name.to_s =~ /\Afind_(all_)?by_\w+\z/ || super
  end
end

User.create(name: "สมชาย", email: "a@a.com", age: 25, active: true)
User.create(name: "สมหญิง", email: "b@b.com", age: 30, active: true)
User.create(name: "สมศรี", email: "c@c.com", age: 25, active: false)

puts User.find_by_name("สมชาย").email
puts User.find_all_by_age(25).map(&:name).inspect
puts User.respond_to?(:find_by_email)   # => true
```

---

## ขั้นตอนที่ 439: define_method

`define_method` สร้าง instance method ใน runtime

```ruby
# สร้าง methods หลายตัวด้วย loop
class Formatter
  [:bold, :italic, :underline].each do |style|
    define_method("format_#{style}") do |text|
      "<#{style}>#{text}</#{style}>"
    end
  end
end

f = Formatter.new
puts f.format_bold("Hello")       # => <bold>Hello</bold>
puts f.format_italic("World")     # => <italic>World</italic>
puts f.format_underline("Ruby")   # => <underline>Ruby</underline>

# ตัวอย่าง: สร้าง attr_accessor แบบ custom
module TypedAttr
  def typed_attr_reader(name, type)
    define_method(name) do
      instance_variable_get(:"@#{name}")
    end
  end
  
  def typed_attr_writer(name, type)
    define_method(:"#{name}=") do |value|
      unless value.is_a?(type) || value.nil?
        raise TypeError, "Expected #{type}, got #{value.class}"
      end
      instance_variable_set(:"@#{name}", value)
    end
  end
  
  def typed_attr_accessor(name, type)
    typed_attr_reader(name, type)
    typed_attr_writer(name, type)
  end
end

class StrictUser
  extend TypedAttr
  
  typed_attr_accessor :name, String
  typed_attr_accessor :age, Integer
  typed_attr_accessor :active, TrueClass
end

user = StrictUser.new
user.name = "สมชาย"    # ok
user.age = 25           # ok

begin
  user.age = "25"         # TypeError!
rescue TypeError => e
  puts "Error: #{e.message}"
end

# define_method กับ closure
class Counter
  def self.make_counter(start: 0, step: 1)
    count = start
    define_method(:next_value) do
      current = count
      count += step
      current
    end
    define_method(:reset) { count = start }
    define_method(:current) { count }
  end
end

class MyCounter < Counter
  make_counter(start: 10, step: 5)
end

c = MyCounter.new
puts c.next_value   # => 10
puts c.next_value   # => 15
puts c.next_value   # => 20
c.reset
puts c.next_value   # => 10
```

---

## ขั้นตอนที่ 440: send และ public_send

```ruby
# send - เรียก method ด้วยชื่อ (รวม private)
class MyClass
  def public_method
    "public"
  end
  
  private
  
  def private_method
    "private"
  end
end

obj = MyClass.new

# send เรียกได้ทั้ง public และ private
puts obj.send(:public_method)   # => "public"
puts obj.send(:private_method)  # => "private" (ข้าม access control!)

# public_send - เรียกได้เฉพาะ public
puts obj.public_send(:public_method)  # => "public"
begin
  obj.public_send(:private_method)   # => NoMethodError!
rescue NoMethodError => e
  puts "Error: #{e.message}"
end

# Dynamic method dispatching
class Calculator
  def add(a, b)       = a + b
  def subtract(a, b)  = a - b
  def multiply(a, b)  = a * b
  def divide(a, b)    = a.to_f / b
end

calc = Calculator.new
operations = [:add, :subtract, :multiply, :divide]

operations.each do |op|
  result = calc.send(op, 10, 3)
  puts "#{op}(10, 3) = #{result.round(2)}"
end

# ตัวอย่าง: Event system
class EventBus
  def initialize
    @handlers = Hash.new { |h, k| h[k] = [] }
  end
  
  def subscribe(event, handler_object, method_name)
    @handlers[event] << { object: handler_object, method: method_name }
  end
  
  def publish(event, *args)
    @handlers[event].each do |handler|
      handler[:object].send(handler[:method], *args)
    end
  end
end

class EmailService
  def send_welcome_email(user)
    puts "Sending welcome email to #{user[:name]}"
  end
  
  def send_notification(user)
    puts "Sending notification to #{user[:name]}"
  end
end

bus = EventBus.new
email = EmailService.new

bus.subscribe(:user_created, email, :send_welcome_email)
bus.subscribe(:user_created, email, :send_notification)

bus.publish(:user_created, { name: "สมชาย", email: "a@a.com" })
```

---

## ขั้นตอนที่ 441: eval, class_eval, instance_eval

```ruby
# eval - ประเมิน string เป็น code (ระวัง security!)
eval("puts 'Hello from eval'")
result = eval("1 + 2 + 3")
puts result  # => 6

# DANGER: ไม่ควรใช้ eval กับ user input!
# user_input = gets.chomp
# eval(user_input)  # Security vulnerability!

# class_eval (alias: module_eval) - เพิ่ม methods ลงใน class
String.class_eval do
  def shout
    upcase + "!!!"
  end
  
  def whisper
    downcase + "..."
  end
end

puts "hello".shout    # => "HELLO!!!"
puts "HELLO".whisper  # => "hello..."

# class_eval กับ string
class MyModel
  [:created_at, :updated_at].each do |field|
    class_eval <<~RUBY, __FILE__, __LINE__ + 1
      def #{field}
        @#{field}
      end
      
      def #{field}=(value)
        @#{field} = value
      end
    RUBY
  end
end

m = MyModel.new
m.created_at = Time.now
puts m.created_at

# instance_eval - เพิ่ม methods ลง instance หรือ change context
class Configuration
  def initialize
    @settings = {}
  end
  
  def configure(&block)
    instance_eval(&block)
  end
  
  def set(key, value)
    @settings[key] = value
  end
  
  def get(key)
    @settings[key]
  end
end

config = Configuration.new
config.configure do
  set :host, "localhost"
  set :port, 3000
  set :debug, true
end

puts config.get(:host)   # => "localhost"
puts config.get(:port)   # => 3000

# instance_eval บน object โดยตรง
obj = Object.new
obj.instance_eval do
  @secret = "hidden value"
  
  def reveal_secret
    @secret
  end
end

puts obj.reveal_secret   # => "hidden value"
```

---

## ขั้นตอนที่ 442: const_get และ const_set

```ruby
# const_get - ดึง constant โดยใช้ชื่อ
puts Object.const_get("String")   # => String
puts Object.const_get("Math::PI") # => 3.14159...

# สร้าง class โดยใช้ชื่อ string
class_name = "Array"
klass = Object.const_get(class_name)
puts klass.new([1, 2, 3])   # => [1, 2, 3]

# const_set - กำหนด constant
Object.const_set("MY_CONSTANT", 42)
puts MY_CONSTANT   # => 42

# Dynamic class creation
def create_model_class(name, attributes)
  klass = Class.new do
    attributes.each do |attr|
      attr_accessor attr
    end
    
    define_method(:initialize) do |**kwargs|
      kwargs.each { |k, v| send(:"#{k}=", v) }
    end
    
    define_method(:to_h) do
      attributes.each_with_object({}) { |attr, h| h[attr] = send(attr) }
    end
    
    define_method(:to_s) do
      "#{self.class.name}(#{to_h.inspect})"
    end
  end
  
  Object.const_set(name, klass)
  klass
end

# สร้าง class ใหม่
create_model_class("UserModel", [:name, :email, :age])
user = UserModel.new(name: "สมชาย", email: "a@a.com", age: 25)
puts user.name      # => "สมชาย"
puts user.to_h.inspect
puts user.to_s

# ตรวจสอบว่า constant มีอยู่
puts Object.const_defined?("String")      # => true
puts Object.const_defined?("NoSuchClass") # => false

# Namespace
module MyApp
  class User; end
  class Post; end
end

puts MyApp.const_get("User")   # => MyApp::User
puts MyApp::User                # => MyApp::User
```

---

## ขั้นตอนที่ 443: instance_variable_get/set

```ruby
# การเข้าถึง instance variables จากภายนอก
class SecretKeeper
  def initialize
    @secret = "I love Ruby"
    @password = "12345"
    @data = { key: "value" }
  end
end

keeper = SecretKeeper.new

# instance_variable_get - อ่านค่า
puts keeper.instance_variable_get(:@secret)    # => "I love Ruby"
puts keeper.instance_variable_get("@password") # => "12345"
puts keeper.instance_variable_get(:@data).inspect  # => {:key=>"value"}

# instance_variable_set - กำหนดค่า
keeper.instance_variable_set(:@secret, "New secret!")
puts keeper.instance_variable_get(:@secret)    # => "New secret!"

# instance_variables - list ทั้งหมด
puts keeper.instance_variables.inspect
# => [:@secret, :@password, :@data]

# instance_variable_defined?
puts keeper.instance_variable_defined?(:@secret)      # => true
puts keeper.instance_variable_defined?(:@nonexistent) # => false

# ตัวอย่างจริง: generic copy
def deep_copy(obj)
  copy = obj.class.allocate  # สร้าง object ว่าง
  obj.instance_variables.each do |var|
    value = obj.instance_variable_get(var)
    copied_value = value.dup rescue value  # dup shallow copy
    copy.instance_variable_set(var, copied_value)
  end
  copy
end

class Config
  attr_accessor :host, :port, :options
  
  def initialize(host, port, options = {})
    @host = host
    @port = port
    @options = options
  end
end

original = Config.new("localhost", 3000, { debug: true })
copy = deep_copy(original)
copy.host = "production.example.com"

puts original.host  # => "localhost" (ไม่เปลี่ยน)
puts copy.host      # => "production.example.com"

# สร้าง memoize decorator
module Memoizable
  def memoize(method_name)
    original_method = instance_method(method_name)
    cache_var = :"@_memo_#{method_name}"
    
    define_method(method_name) do |*args|
      cache = instance_variable_get(cache_var) || {}
      unless cache.key?(args)
        cache[args] = original_method.bind(self).call(*args)
        instance_variable_set(cache_var, cache)
      end
      cache[args]
    end
  end
end

class ExpensiveCalc
  extend Memoizable
  
  def fib(n)
    return n if n <= 1
    fib(n - 1) + fib(n - 2)
  end
  
  memoize :fib
end

calc = ExpensiveCalc.new
puts calc.fib(30)   # fast because memoized
puts calc.fib(35)
```

---

## ขั้นตอนที่ 444: Class Variables และ Class Methods

```ruby
# class_variable_get/set
class Vehicle
  @@count = 0
  
  def initialize
    @@count += 1
  end
end

class Car < Vehicle; end
class Truck < Vehicle; end

3.times { Car.new }
2.times { Truck.new }

puts Vehicle.class_variable_get(:@@count)   # => 5
puts Car.class_variable_get(:@@count)        # => 5 (shared!)

# ระวัง: class variables ถูก share กัน!
Vehicle.class_variable_set(:@@count, 0)
puts Car.class_variable_get(:@@count)   # => 0

# class instance variables - ดีกว่า class variables
class ImprovedVehicle
  @count = 0  # class instance variable
  
  def self.count
    @count
  end
  
  def self.count=(val)
    @count = val
  end
  
  def initialize
    self.class.count += 1
  end
end

class ImprovedCar < ImprovedVehicle
  @count = 0  # แยก count ต่างหาก!
end

3.times { ImprovedCar.new }
2.times { ImprovedVehicle.new }

puts ImprovedVehicle.count   # => 2 (เฉพาะ Vehicle)
puts ImprovedCar.count       # => 3 (เฉพาะ Car)
```

---

## ขั้นตอนที่ 445: Hooks - method_added

```ruby
# method_added - ถูกเรียกทุกครั้งที่มีการเพิ่ม method

module Logging
  def self.included(base)
    base.instance_variable_set(:@logged_methods, [])
    
    base.define_singleton_method(:method_added) do |method_name|
      return if [:method_added, :initialize].include?(method_name)
      @logged_methods << method_name
      puts "Method added: #{method_name} to #{self}"
    end
    
    base.define_singleton_method(:logged_methods) do
      @logged_methods
    end
  end
end

class MyService
  include Logging
  
  def perform
    "performing"
  end
  
  def validate
    "validating"
  end
end

puts MyService.logged_methods.inspect
# => [:perform, :validate]

# method_removed / method_undefined
module MethodTracker
  def self.extended(base)
    base.define_singleton_method(:method_removed) do |name|
      puts "Method removed: #{name}"
    end
    
    base.define_singleton_method(:method_undefined) do |name|
      puts "Method undefined: #{name}"
    end
  end
end

class Demo
  extend MethodTracker
  
  def greet; "hello"; end
  def bye; "goodbye"; end
end

Demo.send(:remove_method, :bye)
# => Method removed: bye

Demo.send(:undef_method, :greet)
# => Method undefined: greet
```

---

## ขั้นตอนที่ 446: Hooks - inherited

```ruby
# inherited - ถูกเรียกเมื่อ class ถูก subclass

class Plugin
  @plugins = []
  
  def self.inherited(subclass)
    @plugins << subclass
    puts "New plugin registered: #{subclass}"
    super
  end
  
  def self.all_plugins
    @plugins
  end
  
  def self.run_all
    @plugins.each do |plugin|
      instance = plugin.new
      instance.run if instance.respond_to?(:run)
    end
  end
end

class EmailPlugin < Plugin
  def run
    puts "Sending emails..."
  end
end

class LogPlugin < Plugin
  def run
    puts "Logging..."
  end
end

class MetricsPlugin < Plugin
  def run
    puts "Collecting metrics..."
  end
end

puts Plugin.all_plugins.inspect
Plugin.run_all

# ตัวอย่าง: Registry pattern
class Animal
  @registry = {}
  
  def self.inherited(subclass)
    @registry[subclass.name.downcase.to_sym] = subclass
    super
  end
  
  def self.registry
    @registry
  end
  
  def self.create(type, *args)
    klass = @registry[type.to_sym] or raise "Unknown animal: #{type}"
    klass.new(*args)
  end
end

class Dog < Animal
  def speak = "Woof!"
end

class Cat < Animal
  def speak = "Meow!"
end

class Bird < Animal
  def speak = "Tweet!"
end

puts Animal.registry.keys.inspect
# => [:dog, :cat, :bird]

animal = Animal.create(:dog)
puts animal.speak   # => "Woof!"
```

---

## ขั้นตอนที่ 447: Hooks - included

```ruby
# included - ถูกเรียกเมื่อ module ถูก include
# extended - ถูกเรียกเมื่อ module ถูก extend

module Serializable
  def self.included(base)
    puts "Serializable included in #{base}"
    base.extend(ClassMethods)
    base.instance_variable_set(:@serializable_attrs, [])
  end
  
  module ClassMethods
    def serialize(*attrs)
      @serializable_attrs += attrs
    end
    
    def serializable_attrs
      @serializable_attrs
    end
  end
  
  def to_json
    require 'json'
    data = self.class.serializable_attrs.each_with_object({}) do |attr, h|
      h[attr] = send(attr)
    end
    JSON.generate(data)
  end
  
  def to_hash
    self.class.serializable_attrs.each_with_object({}) do |attr, h|
      h[attr] = send(attr)
    end
  end
end

class User
  include Serializable
  
  attr_accessor :name, :email, :age, :password
  
  serialize :name, :email, :age  # password ไม่ serialize
  
  def initialize(name, email, age, password)
    @name = name
    @email = email
    @age = age
    @password = password
  end
end

user = User.new("สมชาย", "a@a.com", 25, "secret123")
puts user.to_json
# => {"name":"สมชาย","email":"a@a.com","age":25}

puts user.to_hash.inspect
# password ไม่ปรากฏ!

# prepend hook
module Validatable
  def self.prepended(base)
    puts "Validatable prepended to #{base}"
  end
end
```

---

## ขั้นตอนที่ 448: Dynamic Attribute Creation (like attr_accessor)

```ruby
# สร้าง attr_accessor เองตั้งแต่ต้น
class Object
  def self.my_attr_reader(name)
    define_method(name) do
      instance_variable_get(:"@#{name}")
    end
  end
  
  def self.my_attr_writer(name)
    define_method(:"#{name}=") do |value|
      instance_variable_set(:"@#{name}", value)
    end
  end
  
  def self.my_attr_accessor(name)
    my_attr_reader(name)
    my_attr_writer(name)
  end
end

class Person
  my_attr_accessor :name
  my_attr_accessor :age
end

p = Person.new
p.name = "สมชาย"
p.age = 25
puts "#{p.name}, #{p.age}"

# attr_accessor พร้อม validation
module ValidatedAttributes
  def validated_attr(name, **validators)
    define_method(name) do
      instance_variable_get(:"@#{name}")
    end
    
    define_method(:"#{name}=") do |value|
      # ตรวจสอบ type
      if (type = validators[:type])
        unless value.is_a?(type)
          raise TypeError, "#{name} must be #{type}, got #{value.class}"
        end
      end
      
      # ตรวจสอบ range
      if (range = validators[:in])
        unless range.include?(value)
          raise ArgumentError, "#{name} must be in #{range}"
        end
      end
      
      # ตรวจสอบ minimum length
      if (min = validators[:min_length]) && value.respond_to?(:length)
        unless value.length >= min
          raise ArgumentError, "#{name} must be at least #{min} characters"
        end
      end
      
      # ตรวจสอบ regex
      if (pattern = validators[:format])
        unless value.to_s.match?(pattern)
          raise ArgumentError, "#{name} has invalid format"
        end
      end
      
      instance_variable_set(:"@#{name}", value)
    end
  end
end

class StrictUser
  extend ValidatedAttributes
  
  validated_attr :name,  type: String, min_length: 2
  validated_attr :age,   type: Integer, in: (0..150)
  validated_attr :email, format: /\A[^@]+@[^@]+\z/
end

user = StrictUser.new
user.name = "สมชาย"
user.age = 25
user.email = "test@example.com"
puts "#{user.name}, #{user.age}, #{user.email}"

begin
  user.age = -1
rescue ArgumentError => e
  puts "Error: #{e.message}"
end
```

---

## ขั้นตอนที่ 449: DSL Building

```ruby
# สร้าง DSL (Domain Specific Language) สำหรับ configuration

class ServerConfig
  attr_reader :settings
  
  def initialize
    @settings = {}
    @routes = []
    @middleware = []
  end
  
  # DSL entry point
  def self.configure(&block)
    config = new
    config.instance_eval(&block)
    config
  end
  
  def server(name, &block)
    @settings[:name] = name
    instance_eval(&block) if block_given?
  end
  
  def listen_on(host:, port:)
    @settings[:host] = host
    @settings[:port] = port
  end
  
  def enable(*features)
    @settings[:features] ||= []
    @settings[:features] += features
  end
  
  def use(middleware, **options)
    @middleware << { name: middleware, options: options }
  end
  
  def route(method, path, handler: nil, &block)
    @routes << { method: method, path: path, handler: handler || block }
  end
  
  def middleware = @middleware
  def routes = @routes
end

config = ServerConfig.configure do
  server "MyApp" do
    listen_on host: "0.0.0.0", port: 8080
    enable :logging, :compression, :caching
    
    use :auth, type: :jwt, secret: "mysecret"
    use :rate_limit, max: 100, window: 60
    
    route :get, "/",       handler: -> { "Home" }
    route :get, "/users",  handler: -> { "Users list" }
    route :post, "/users" do
      "Create user"
    end
  end
end

puts "Server: #{config.settings[:name]}"
puts "Host: #{config.settings[:host]}:#{config.settings[:port]}"
puts "Features: #{config.settings[:features].inspect}"
puts "Middleware: #{config.middleware.map { |m| m[:name] }.inspect}"
puts "Routes: #{config.routes.map { |r| "#{r[:method].upcase} #{r[:path]}" }.inspect}"
```

---

## ขั้นตอนที่ 450: Building a Simple ORM

```ruby
# Simple ORM ด้วย Metaprogramming

class Model
  @table_name = nil
  @columns = []
  
  def self.table_name
    @table_name || name.downcase.gsub('::', '_') + 's'
  end
  
  def self.set_table_name(name)
    @table_name = name
  end
  
  def self.column(name, type = :string, **options)
    @columns ||= []
    @columns << { name: name, type: type, options: options }
    
    attr_reader name
    
    define_method(:"#{name}=") do |value|
      @changed_attributes ||= []
      @changed_attributes << name unless instance_variable_get(:"@#{name}") == value
      instance_variable_set(:"@#{name}", value)
    end
  end
  
  def self.columns
    @columns ||= []
  end
  
  def self.all
    @records ||= []
  end
  
  def self.create(**attrs)
    obj = new(**attrs)
    obj.save
    obj
  end
  
  def self.find(id)
    all.find { |r| r.id == id }
  end
  
  def self.where(**conditions)
    all.select do |record|
      conditions.all? { |key, value| record.send(key) == value }
    end
  end
  
  def self.count
    all.size
  end
  
  def initialize(**attrs)
    @id = nil
    @persisted = false
    @changed_attributes = []
    attrs.each { |k, v| send(:"#{k}=", v) }
  end
  
  def save
    if @persisted
      update
    else
      insert
    end
    @changed_attributes = []
    self
  end
  
  def destroy
    self.class.all.delete(self)
    @persisted = false
    self
  end
  
  def persisted? = @persisted
  def new_record? = !@persisted
  def changed? = !@changed_attributes.empty?
  def changed_attributes = @changed_attributes.dup
  
  def attributes
    self.class.columns.each_with_object({}) do |col, h|
      h[col[:name]] = send(col[:name])
    end
  end
  
  def to_s
    "#{self.class.name}##{@id}(#{attributes.inspect})"
  end
  
  attr_accessor :id
  
  private
  
  def insert
    @id = self.class.all.size + 1
    self.class.all << self
    @persisted = true
  end
  
  def update
    # already in array, just mark as not changed
    true
  end
end

# ใช้งาน ORM
class User < Model
  set_table_name "users"
  
  column :name,   :string
  column :email,  :string
  column :age,    :integer
  column :active, :boolean, default: true
  
  def full_info
    "#{name} <#{email}> (#{age})"
  end
end

class Post < Model
  column :title,  :string
  column :body,   :text
  column :author, :string
end

# สร้าง records
u1 = User.create(name: "สมชาย", email: "a@a.com", age: 25, active: true)
u2 = User.create(name: "สมหญิง", email: "b@b.com", age: 30, active: false)
u3 = User.create(name: "สมศรี", email: "c@c.com", age: 25, active: true)

puts "Total users: #{User.count}"
puts "Find user 2: #{User.find(2)}"
puts "Active users: #{User.where(active: true).map(&:name).inspect}"
puts "Age 25 users: #{User.where(age: 25).map(&:name).inspect}"

u1.name = "สมชาย ใจดี"
puts "Changed: #{u1.changed?}"
puts "Changed attrs: #{u1.changed_attributes.inspect}"
u1.save
puts "After save changed: #{u1.changed?}"

puts "\nAll users:"
User.all.each { |u| puts "  #{u}" }
```

---

## ขั้นตอนที่ 451: Open Classes และ Monkey Patching

```ruby
# Ruby อนุญาตให้ reopen class เพื่อเพิ่ม methods

# Monkey patching Integer
class Integer
  def factorial
    return 1 if self <= 1
    self * (self - 1).factorial
  end
  
  def prime?
    return false if self < 2
    (2..Math.sqrt(self).to_i).none? { |i| self % i == 0 }
  end
  
  def times_do
    (1..self).each { |i| yield i }
  end
  
  def to_roman
    num = self
    result = ""
    [[1000,"M"], [900,"CM"], [500,"D"], [400,"CD"],
     [100,"C"], [90,"XC"], [50,"L"], [40,"XL"],
     [10,"X"], [9,"IX"], [5,"V"], [4,"IV"], [1,"I"]].each do |value, numeral|
      while num >= value
        result += numeral
        num -= value
      end
    end
    result
  end
end

puts 5.factorial     # => 120
puts 17.prime?       # => true
puts 15.to_roman     # => "XV"

5.times_do { |i| print "#{i} " }
puts

# Monkey patching String
class String
  def word_count
    split.size
  end
  
  def titleize
    split.map(&:capitalize).join(' ')
  end
  
  def truncate(length, omission: '...')
    return self if size <= length
    self[0, length - omission.size] + omission
  end
  
  def to_bool
    case downcase
    when 'true', '1', 'yes', 'on'  then true
    when 'false', '0', 'no', 'off' then false
    else nil
    end
  end
end

puts "hello world".word_count   # => 2
puts "hello world".titleize     # => "Hello World"
puts "hello world foo bar".truncate(10)  # => "hello w..."
puts "true".to_bool    # => true
puts "no".to_bool      # => false

# Refinements - safer monkey patching
module StringRefinements
  refine String do
    def shout
      upcase + "!!!"
    end
  end
end

# ใช้งานเฉพาะใน scope ที่ระบุ
using StringRefinements
puts "hello".shout  # => "HELLO!!!"
```

---

## ขั้นตอนที่ 452: Proxy Pattern

```ruby
# Proxy Pattern ด้วย method_missing
class Proxy
  def initialize(target)
    @target = target
    @call_log = []
  end
  
  def method_missing(name, *args, &block)
    start_time = Time.now
    result = @target.send(name, *args, &block)
    elapsed = Time.now - start_time
    
    @call_log << {
      method: name,
      args: args,
      elapsed: elapsed,
      result: result
    }
    
    result
  end
  
  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private) || super
  end
  
  def call_log
    @call_log
  end
  
  def method_stats
    @call_log.group_by { |entry| entry[:method] }
             .transform_values do |calls|
               {
                 count: calls.size,
                 total_time: calls.sum { |c| c[:elapsed] },
                 avg_time: calls.sum { |c| c[:elapsed] } / calls.size
               }
             end
  end
end

class DataService
  def fetch_users
    sleep(0.001)  # simulate delay
    [{ name: "ก" }, { name: "ข" }]
  end
  
  def fetch_posts
    sleep(0.002)
    [{ title: "Post 1" }, { title: "Post 2" }]
  end
end

service = Proxy.new(DataService.new)

5.times { service.fetch_users }
3.times { service.fetch_posts }
2.times { service.fetch_users }

puts "\nMethod statistics:"
service.method_stats.each do |method, stats|
  puts "  #{method}: #{stats[:count]} calls, avg #{(stats[:avg_time] * 1000).round(2)}ms"
end
```

---

## ขั้นตอนที่ 453: Builder Pattern กับ Metaprogramming

```ruby
# Builder Pattern ด้วย DSL
class QueryBuilder
  def initialize(table)
    @table = table
    @conditions = []
    @order_clause = nil
    @limit_value = nil
    @offset_value = nil
    @selected_columns = ["*"]
    @joins = []
  end
  
  def select(*columns)
    @selected_columns = columns.map(&:to_s)
    self
  end
  
  def where(condition = nil, **kwargs)
    if condition.is_a?(String)
      @conditions << condition
    elsif kwargs.any?
      kwargs.each do |key, value|
        case value
        when Array
          @conditions << "#{key} IN (#{value.map { |v| quote(v) }.join(', ')})"
        when Range
          @conditions << "#{key} BETWEEN #{quote(value.begin)} AND #{quote(value.end)}"
        when nil
          @conditions << "#{key} IS NULL"
        else
          @conditions << "#{key} = #{quote(value)}"
        end
      end
    end
    self
  end
  
  def order(column, direction = :asc)
    @order_clause = "#{column} #{direction.to_s.upcase}"
    self
  end
  
  def limit(n)
    @limit_value = n
    self
  end
  
  def offset(n)
    @offset_value = n
    self
  end
  
  def join(table, condition)
    @joins << "JOIN #{table} ON #{condition}"
    self
  end
  
  def build
    sql = "SELECT #{@selected_columns.join(', ')} FROM #{@table}"
    sql += " #{@joins.join(' ')}" if @joins.any?
    sql += " WHERE #{@conditions.join(' AND ')}" if @conditions.any?
    sql += " ORDER BY #{@order_clause}" if @order_clause
    sql += " LIMIT #{@limit_value}" if @limit_value
    sql += " OFFSET #{@offset_value}" if @offset_value
    sql
  end
  
  alias to_s build
  
  private
  
  def quote(value)
    value.is_a?(String) ? "'#{value}'" : value.to_s
  end
end

# ใช้งาน
query = QueryBuilder.new("users")
  .select(:id, :name, :email)
  .where(active: true)
  .where(age: 18..65)
  .order(:name)
  .limit(10)
  .offset(20)
  .build

puts query

# แบบ chaining
sql = QueryBuilder.new("orders")
  .join("users", "orders.user_id = users.id")
  .where("orders.total > 100")
  .where(status: [:pending, :processing])
  .order(:created_at, :desc)
  .limit(50)
  .build

puts sql
```

---

## ขั้นตอนที่ 454: Decorator Pattern

```ruby
# Decorator Pattern ใน Ruby
module Decoratable
  def decorate_method(method_name, with:)
    original = instance_method(method_name)
    decorator = with
    
    define_method(method_name) do |*args, &block|
      decorator.call(self, original.bind(self), *args, &block)
    end
  end
end

# Decorators
TIMING_DECORATOR = ->(obj, original_method, *args, &block) {
  start = Time.now
  result = original_method.call(*args, &block)
  elapsed = (Time.now - start) * 1000
  puts "#{original_method.name} took #{elapsed.round(2)}ms"
  result
}

LOGGING_DECORATOR = ->(obj, original_method, *args, &block) {
  puts "Calling #{original_method.name}(#{args.inspect})"
  result = original_method.call(*args, &block)
  puts "Result: #{result.inspect}"
  result
}

CACHING_DECORATOR = ->(obj, original_method, *args, &block) {
  obj.instance_variable_get(:@_cache) || obj.instance_variable_set(:@_cache, {})
  cache = obj.instance_variable_get(:@_cache)
  key = [original_method.name, args]
  
  unless cache.key?(key)
    cache[key] = original_method.call(*args, &block)
  end
  cache[key]
}

class DataProcessor
  extend Decoratable
  
  def process(data)
    sleep(0.001)  # simulate work
    data.map { |x| x * 2 }
  end
  
  def fetch(id)
    sleep(0.002)  # simulate DB call
    { id: id, name: "Item #{id}" }
  end
  
  decorate_method :process, with: TIMING_DECORATOR
  decorate_method :fetch,   with: CACHING_DECORATOR
end

dp = DataProcessor.new
result = dp.process([1, 2, 3, 4, 5])
puts result.inspect

# fetch จะ cache
first_call  = dp.fetch(42)
second_call = dp.fetch(42)  # ไม่ไปเรียก sleep อีก
puts first_call.equal?(second_call)  # true (same object)
```

---

## ขั้นตอนที่ 455: ตัวอย่างจริง - Testing Framework

```ruby
# Mini testing framework ด้วย metaprogramming
module MiniTest
  class TestCase
    @tests = []
    
    def self.inherited(subclass)
      subclass.instance_variable_set(:@tests, [])
    end
    
    def self.method_added(name)
      if name.to_s.start_with?("test_")
        @tests << name
      end
    end
    
    def self.tests
      @tests
    end
    
    def self.run
      puts "\n#{self.name}"
      puts "=" * 40
      
      passed = failed = 0
      
      tests.each do |test|
        instance = new
        begin
          instance.setup if instance.respond_to?(:setup)
          instance.send(test)
          instance.teardown if instance.respond_to?(:teardown)
          puts "  ✓ #{test}"
          passed += 1
        rescue => e
          puts "  ✗ #{test}: #{e.message}"
          failed += 1
        end
      end
      
      puts "\nResults: #{passed} passed, #{failed} failed"
      failed == 0
    end
    
    def assert(condition, message = "Assertion failed")
      raise AssertionError, message unless condition
    end
    
    def assert_equal(expected, actual, message = nil)
      msg = message || "Expected #{expected.inspect}, got #{actual.inspect}"
      assert(expected == actual, msg)
    end
    
    def assert_raises(exception_class, &block)
      block.call
      raise AssertionError, "Expected #{exception_class} to be raised"
    rescue exception_class
      true
    end
    
    def refute(condition, message = "Expected false, got true")
      assert(!condition, message)
    end
    
    AssertionError = Class.new(StandardError)
  end
end

# ใช้งาน mini test framework
class CalculatorTest < MiniTest::TestCase
  def setup
    @calc = Calculator.new if defined?(Calculator)
  end
  
  def test_addition
    assert_equal 5, 2 + 3
  end
  
  def test_string_operations
    assert_equal "HELLO", "hello".upcase
    assert_equal 5, "hello".length
  end
  
  def test_array_methods
    arr = [1, 2, 3, 4, 5]
    assert_equal 15, arr.sum
    assert_equal 5, arr.max
    assert_equal 1, arr.min
  end
  
  def test_hash_operations
    h = { a: 1, b: 2, c: 3 }
    assert_equal 3, h.size
    assert h.key?(:a)
    refute h.key?(:d)
  end
  
  def test_raises
    assert_raises(ZeroDivisionError) { 1 / 0 }
  end
end

CalculatorTest.run
```

---

## ขั้นตอนที่ 456: Eigenclass (Singleton Class)

```ruby
# Eigenclass หรือ singleton class - class ของ object นั้นๆ โดยเฉพาะ

obj = "hello"

# เข้าถึง singleton class
singleton = class << obj; self; end
puts singleton.inspect   # => #<Class:#<String:...>>

# เพิ่ม singleton method
def obj.shout
  upcase + "!!!"
end

puts obj.shout   # => "HELLO!!!"

other = "world"
# other.shout  # => NoMethodError! (method อยู่แค่กับ obj)

# Class methods ก็คือ singleton methods บน class object
class Dog
  # นี่คือ singleton method บน Dog class object
  def self.bark
    "WOOF!"
  end
  
  # เหมือนกับ
  class << self
    def bark2
      "WOOF2!"
    end
    
    attr_accessor :default_breed
  end
end

puts Dog.bark    # => "WOOF!"
puts Dog.bark2   # => "WOOF2!"
Dog.default_breed = "Labrador"
puts Dog.default_breed

# extend เพิ่ม instance methods ของ module เป็น singleton methods
module Greetable
  def greet
    "Hello from #{self}!"
  end
end

name = "Ruby"
name.extend(Greetable)
puts name.greet  # => "Hello from Ruby!"

other_name = "Python"
# other_name.greet  # => NoMethodError
```

---

## ขั้นตอนที่ 457: Module ที่ทรงพลัง

```ruby
# Module เป็น namespace, mixin, และ holder ของ methods

# Namespace
module Payments
  module Gateways
    class Stripe
      def charge(amount, token)
        "Charging #{amount} with Stripe using #{token}"
      end
    end
    
    class PayPal
      def charge(amount, token)
        "Charging #{amount} with PayPal using #{token}"
      end
    end
  end
  
  class Transaction
    def initialize(gateway_name)
      @gateway = Gateways.const_get(gateway_name.to_s.capitalize).new
    end
    
    def process(amount, token)
      @gateway.charge(amount, token)
    end
  end
end

tx = Payments::Transaction.new(:stripe)
puts tx.process(100, "tok_123")

# Module hooks ที่ทรงพลัง
module Observable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@observers, [])
    base.include(InstanceMethods)
  end
  
  module ClassMethods
    def add_observer(observer)
      @observers << observer
    end
    
    def observers
      @observers
    end
  end
  
  module InstanceMethods
    def notify_observers(event, *data)
      self.class.observers.each do |obs|
        obs.update(event, self, *data) if obs.respond_to?(:update)
      end
    end
    
    def change(attribute, old_value, new_value)
      notify_observers(:attribute_changed, attribute, old_value, new_value)
    end
  end
end

class Logger
  def update(event, source, *data)
    puts "[LOG] #{source.class}: #{event} - #{data.inspect}"
  end
end

class Account
  include Observable
  
  attr_reader :balance
  
  def initialize(balance)
    @balance = balance
  end
  
  def deposit(amount)
    old = @balance
    @balance += amount
    change(:balance, old, @balance)
  end
  
  def withdraw(amount)
    old = @balance
    @balance -= amount
    change(:balance, old, @balance)
  end
end

Account.add_observer(Logger.new)

account = Account.new(1000)
account.deposit(500)
account.withdraw(200)
```

---

## ขั้นตอนที่ 458: Advanced - Tracing และ Profiling

```ruby
# ใช้ TracePoint สำหรับ monitoring
require 'set'

class MethodCallTracer
  def initialize
    @calls = []
    @trace = nil
  end
  
  def start
    @trace = TracePoint.new(:call, :return) do |tp|
      next if tp.defined_class == self.class || tp.path.include?('/gems/')
      
      if tp.event == :call
        @calls << {
          event: :call,
          class: tp.defined_class,
          method: tp.method_id,
          file: File.basename(tp.path),
          line: tp.lineno
        }
      end
    end
    
    @trace.enable
    self
  end
  
  def stop
    @trace&.disable
    self
  end
  
  def report
    puts "Method calls recorded: #{@calls.size}"
    @calls.group_by { |c| "#{c[:class]}##{c[:method]}" }
          .sort_by { |_, calls| -calls.size }
          .first(10)
          .each do |method, calls|
            puts "  #{method}: #{calls.size} calls"
          end
  end
  
  def trace(&block)
    start
    block.call
    stop
    report
    self
  end
end

# ตัวอย่าง tracing
tracer = MethodCallTracer.new
tracer.trace do
  [1, 2, 3, 4, 5].map { |n| n ** 2 }.select(&:even?).sum
end
```

---

## ขั้นตอนที่ 459: Practical Metaprogramming Patterns

```ruby
# Pattern: Lazy Loading
module LazyLoading
  def lazy_load(attribute, &loader)
    ivar = :"@#{attribute}"
    
    define_method(attribute) do
      unless instance_variable_defined?(ivar)
        instance_variable_set(ivar, instance_exec(&loader))
      end
      instance_variable_get(ivar)
    end
  end
end

class DataRepository
  extend LazyLoading
  
  lazy_load(:users) do
    puts "Loading users from DB..."
    [{ id: 1, name: "สมชาย" }, { id: 2, name: "สมหญิง" }]
  end
  
  lazy_load(:products) do
    puts "Loading products from DB..."
    [{ id: 1, name: "Product A" }]
  end
end

repo = DataRepository.new
puts "Before first access"
puts repo.users.inspect    # Loading users from DB...
puts repo.users.inspect    # No loading (cached)
puts repo.products.inspect # Loading products from DB...

# Pattern: Delegation
module Delegator
  def delegate(*methods, to:)
    methods.each do |method|
      define_method(method) do |*args, &block|
        send(to).send(method, *args, &block)
      end
    end
  end
end

class UserPresenter
  extend Delegator
  
  delegate :name, :email, to: :@user
  delegate :format_date, to: :@formatter
  
  def initialize(user)
    @user = user
    @formatter = DateFormatter.new
  end
  
  def display
    "#{name} <#{email}>"
  end
end

# Pattern: Command Pattern
class CommandDispatcher
  def initialize
    @commands = {}
  end
  
  def register(name, &handler)
    @commands[name.to_sym] = handler
  end
  
  def dispatch(name, *args)
    handler = @commands[name.to_sym]
    raise "Unknown command: #{name}" unless handler
    handler.call(*args)
  end
  
  def method_missing(name, *args)
    if @commands.key?(name)
      dispatch(name, *args)
    else
      super
    end
  end
  
  def respond_to_missing?(name, include_private = false)
    @commands.key?(name) || super
  end
end

dispatcher = CommandDispatcher.new
dispatcher.register(:greet) { |name| "Hello, #{name}!" }
dispatcher.register(:shout) { |text| text.upcase + "!!!" }
dispatcher.register(:reverse) { |text| text.reverse }

puts dispatcher.greet("Ruby")      # => "Hello, Ruby!"
puts dispatcher.shout("hello")     # => "HELLO!!!"
puts dispatcher.reverse("racecar") # => "racecar"
```

---

## ขั้นตอนที่ 460: สรุปและ Best Practices

```ruby
# Metaprogramming Best Practices

# 1. ใช้ define_method แทน eval เมื่อทำได้
# Good:
[:up, :down, :left, :right].each do |direction|
  define_method("move_#{direction}") do
    @position.send("#{direction}!")
  end
end

# Bad (security risk):
# eval "def move_#{direction}; end"

# 2. ใช้ respond_to_missing? คู่กับ method_missing เสมอ
class Safe
  def method_missing(name, *args)
    return super unless name.to_s.start_with?("safe_")
    "safe: #{name}"
  end
  
  def respond_to_missing?(name, include_private = false)
    name.to_s.start_with?("safe_") || super
  end
end

# 3. ใช้ Refinements แทน Monkey Patching เมื่อเป็นไปได้
module SafeMonkeyPatch
  refine String do
    def truncate(n)
      self[0, n]
    end
  end
end

# 4. Document dynamic methods ด้วย comment
class DynamicClass
  # @!method read_name
  #   @return [String]
  # @!method read_age
  #   @return [Integer]
  [:name, :age].each { |attr| define_method("read_#{attr}") { send(attr) } }
end

# 5. ระวัง performance ของ method_missing
# method_missing ช้ากว่า regular methods
# ใช้ cache หรือ define methods จริงๆ เมื่อเป็นไปได้

class CachingProxy
  def initialize(target)
    @target = target
  end
  
  def method_missing(name, *args, &block)
    # Cache method เพื่อ performance
    if @target.respond_to?(name)
      self.class.define_method(name) do |*a, &b|
        @target.send(name, *a, &b)
      end
      send(name, *args, &block)
    else
      super
    end
  end
end

# Summary of metaprogramming techniques:
# 1. method_missing + respond_to_missing? - dynamic methods
# 2. define_method - create methods at runtime
# 3. send / public_send - dynamic method calling
# 4. class_eval / instance_eval - change context
# 5. const_get / const_set - dynamic constants
# 6. instance_variable_get/set - access state
# 7. hooks: method_added, inherited, included - respond to events
# 8. Module#extend - add singleton methods
# 9. Open classes - reopen and modify
# 10. Refinements - safe monkey patching
```

---

## แบบฝึกหัด 25 ข้อ

### ข้อ 1-5: method_missing และ respond_to_missing?

**ข้อ 1:** สร้าง DynamicHash class ที่เข้าถึง keys ด้วย method names
```ruby
# เฉลย
class DynamicHash
  def initialize(data = {})
    @data = data
  end
  
  def method_missing(name, *args)
    key = name.to_s
    if key.end_with?('=')
      @data[key[0..-2]] = args.first
    elsif @data.key?(key)
      @data[key]
    else
      super
    end
  end
  
  def respond_to_missing?(name, include_private = false)
    @data.key?(name.to_s.chomp('=')) || super
  end
end

dh = DynamicHash.new(name: "Ruby", version: 3)
puts dh.name
dh.language = "Ruby"
puts dh.language
```

**ข้อ 2:** สร้าง method ที่ log ทุก call
```ruby
# เฉลย
module AutoLogger
  def method_added(method_name)
    return if method_name.to_s.start_with?('logged_')
    return if @_logging
    @_logging = true
    
    original = instance_method(method_name)
    define_method(method_name) do |*args, &block|
      puts "[LOG] #{self.class}##{method_name}(#{args.inspect})"
      original.bind(self).call(*args, &block)
    end
    
    @_logging = false
  end
end

class MyService
  extend AutoLogger
  
  def greet(name)
    "Hello, #{name}!"
  end
  
  def calculate(a, b)
    a + b
  end
end

service = MyService.new
puts service.greet("Ruby")
puts service.calculate(3, 4)
```

**ข้อ 3-5:** (ฝึกต่อ)

```ruby
# ข้อ 3: Lazy evaluator
class Lazy
  def initialize(&block)
    @block = block
    @evaluated = false
    @value = nil
  end
  
  def value
    unless @evaluated
      @value = @block.call
      @evaluated = true
    end
    @value
  end
  
  def method_missing(name, *args, &block)
    value.send(name, *args, &block)
  end
  
  def respond_to_missing?(name, include_private = false)
    value.respond_to?(name, include_private) || super
  end
end

lazy_str = Lazy.new do
  puts "Computing..."
  "Hello, World!"
end

puts "Before access"
puts lazy_str.upcase    # Computing... HELLO, WORLD!
puts lazy_str.length    # (no "Computing..." again)

# ข้อ 4: Plugin system
class Application
  @plugins = {}
  
  def self.plugin(name, &block)
    plugin_module = Module.new(&block)
    @plugins[name] = plugin_module
    include(plugin_module)
  end
  
  def self.loaded_plugins
    @plugins.keys
  end
end

Application.plugin(:logging) do
  def log(message)
    puts "[LOG] #{message}"
  end
end

Application.plugin(:caching) do
  def cached(key)
    @cache ||= {}
    @cache[key] ||= yield
  end
end

app = Application.new
app.log("Hello from app")
result = app.cached(:data) { "expensive computation" }
puts result

# ข้อ 5: Method interception
module MethodInterceptor
  def intercept(method_name, before: nil, after: nil)
    original = instance_method(method_name)
    
    define_method(method_name) do |*args, &block|
      before&.call(self, method_name, args)
      result = original.bind(self).call(*args, &block)
      after&.call(self, method_name, result)
      result
    end
  end
end

class OrderService
  extend MethodInterceptor
  
  def process_order(order)
    puts "Processing order #{order[:id]}"
    { status: :success, order_id: order[:id] }
  end
  
  intercept :process_order,
    before: ->(obj, method, args) { puts "Before: #{method} with #{args.inspect}" },
    after:  ->(obj, method, result) { puts "After: #{method} returned #{result.inspect}" }
end

OrderService.new.process_order({ id: 123, total: 500 })
```

### ข้อ 6-15: define_method และ DSL

```ruby
# ข้อ 6: สร้าง State Machine
class StateMachine
  def self.state_machine(&block)
    @states = []
    @transitions = []
    @initial_state = nil
    
    instance_eval(&block)
    
    define_method(:current_state) { @current_state ||= self.class.instance_variable_get(:@initial_state) }
    
    @transitions.each do |from, event, to|
      define_method(event) do
        if current_state == from
          @current_state = to
          puts "Transitioned: #{from} -> #{to}"
          true
        else
          puts "Cannot #{event} from #{current_state}"
          false
        end
      end
    end
  end
  
  def self.state(name, initial: false)
    @states << name
    @initial_state = name if initial
  end
  
  def self.transition(from:, event:, to:)
    @transitions << [from, event, to]
  end
end

class TrafficLight < StateMachine
  state_machine do
    state :red, initial: true
    state :green
    state :yellow
    
    transition from: :red,    event: :go,   to: :green
    transition from: :green,  event: :slow, to: :yellow
    transition from: :yellow, event: :stop, to: :red
  end
end

light = TrafficLight.new
puts light.current_state   # red
light.go                   # red -> green
light.slow                 # green -> yellow
light.stop                 # yellow -> red

# ข้อ 7: Test expectations DSL
class Expectation
  def initialize(actual)
    @actual = actual
    @negated = false
  end
  
  def not
    @negated = !@negated
    self
  end
  
  def to(matcher)
    result = matcher.matches?(@actual)
    result = !result if @negated
    raise "Expectation failed: #{describe_failure(matcher)}" unless result
    true
  end
  
  private
  
  def describe_failure(matcher)
    "Expected #{@actual.inspect} #{@negated ? 'not ' : ''}to #{matcher}"
  end
end

def expect(value)
  Expectation.new(value)
end

module Matchers
  class Equal
    def initialize(expected)
      @expected = expected
    end
    
    def matches?(actual)
      actual == @expected
    end
    
    def to_s = "equal #{@expected.inspect}"
  end
  
  class BeA
    def initialize(type)
      @type = type
    end
    
    def matches?(actual)
      actual.is_a?(@type)
    end
    
    def to_s = "be a #{@type}"
  end
end

def eq(value) = Matchers::Equal.new(value)
def be_a(type) = Matchers::BeA.new(type)

expect(2 + 2).to(eq(4))
expect("hello").to(be_a(String))
expect(42).not.to(eq(43))
puts "All expectations passed!"
```

### ข้อ 16-25: Advanced Metaprogramming

```ruby
# ข้อ 16-20: ORM Extension
module ORM
  module ClassMethods
    def has_many(name, class_name: nil, foreign_key: nil)
      class_name ||= name.to_s.capitalize.chomp('s')
      foreign_key ||= "#{self.name.downcase}_id"
      
      define_method(name) do
        related_class = Object.const_get(class_name)
        related_class.where(foreign_key.to_sym => self.id)
      end
    end
    
    def belongs_to(name, class_name: nil, foreign_key: nil)
      class_name ||= name.to_s.capitalize
      foreign_key ||= "#{name}_id"
      
      define_method(name) do
        related_class = Object.const_get(class_name)
        related_class.find(send(foreign_key))
      end
    end
  end
end

# ข้อ 21-25: Advanced Patterns
class Configuration
  class << self
    def option(name, default: nil, type: nil)
      define_method(name) do
        ivar = instance_variable_get(:"@#{name}")
        ivar.nil? ? default : ivar
      end
      
      define_method(:"#{name}=") do |value|
        if type && !value.is_a?(type)
          raise TypeError, "#{name} must be #{type}"
        end
        instance_variable_set(:"@#{name}", value)
      end
      
      define_method(:"#{name}?") do
        !send(name).nil?
      end
    end
  end
  
  option :host,    default: "localhost", type: String
  option :port,    default: 3000,        type: Integer
  option :debug,   default: false
  option :timeout, default: 30,          type: Integer
end

config = Configuration.new
puts config.host        # => "localhost"
puts config.port        # => 3000
puts config.debug?      # => true (has default value)

config.port = 8080
puts config.port        # => 8080

begin
  config.port = "8080"  # wrong type
rescue TypeError => e
  puts "Error: #{e.message}"
end

# สร้าง full DSL
class Workflow
  class Step
    attr_reader :name, :action, :conditions
    
    def initialize(name, &action)
      @name = name
      @action = action
      @conditions = []
    end
    
    def if_condition(&block)
      @conditions << block
      self
    end
    
    def execute(context)
      return :skipped if @conditions.any? { |c| !c.call(context) }
      @action.call(context)
    end
  end
  
  def initialize(name)
    @name = name
    @steps = []
  end
  
  def step(name, &block)
    s = Step.new(name, &block)
    @steps << s
    s
  end
  
  def run(context = {})
    puts "Running workflow: #{@name}"
    results = {}
    
    @steps.each do |step|
      result = step.execute(context)
      results[step.name] = result
      puts "  #{step.name}: #{result}"
    end
    
    results
  end
end

wf = Workflow.new("Order Processing")
wf.step(:validate) { |ctx| ctx[:order] ? :ok : :failed }
wf.step(:charge)   { |ctx| puts "Charging #{ctx[:amount]}"; :charged }
   .if_condition   { |ctx| ctx[:amount] > 0 }
wf.step(:notify)   { |ctx| puts "Notifying #{ctx[:email]}"; :sent }

wf.run(order: { id: 1 }, amount: 100, email: "user@example.com")
```

---

## สรุปบทที่ 21

ในบทนี้เราได้เรียนรู้ Metaprogramming ใน Ruby:

**Core Techniques:**
- `method_missing` + `respond_to_missing?` - dynamic method handling
- `define_method` - สร้าง methods ใน runtime
- `send` / `public_send` - dynamic method dispatch
- `eval`, `class_eval`, `instance_eval` - runtime code evaluation
- `const_get` / `const_set` - dynamic constants
- `instance_variable_get/set` - access state dynamically

**Hooks:**
- `method_added` - respond to method creation
- `inherited` - respond to subclassing
- `included` / `extended` / `prepended` - respond to module inclusion

**Advanced Patterns:**
- Dynamic attr_accessor with validation
- DSL building
- Simple ORM
- Plugin/Registry systems
- Proxy pattern
- Builder pattern
- Decorator pattern

**Best Practices:**
- ใช้ Refinements แทน monkey patching
- ตรวจสอบ security เมื่อใช้ eval
- cache dynamic methods เพื่อ performance
- document dynamic methods
- ใช้ define_method แทน eval เมื่อทำได้
