# ตอนที่ 13: Modules และ Mixins (Steps 256-280)

## บทนำ

Modules ใน Ruby เป็นเครื่องมือที่ทรงพลังมาก ทำได้ 2 สิ่งหลัก:
1. **Namespace** - จัดกลุ่มโค้ดเพื่อหลีกเลี่ยง name collision
2. **Mixin** - แชร์ functionality ระหว่าง classes โดยไม่ต้องใช้ inheritance

---

## Step 256: Module คืออะไร? (ต่างจาก Class อย่างไร?)

```ruby
# สร้าง Module
module Greetable
  def greet(name)
    puts "Hello, #{name}! I'm #{self.class.name}"
  end

  def farewell(name)
    puts "Goodbye, #{name}!"
  end
end

# ความแตกต่าง Module vs Class
# 1. Module ไม่สามารถสร้าง instance ได้
# module_instance = Greetable.new  # => NoMethodError

# 2. Module ไม่สามารถ inherit ได้
# class Foo < Greetable  # => TypeError

# 3. Module ไม่มี state (ไม่มี initialize ปกติ)

# แต่ Module สามารถ:
# - มี methods
# - มี constants
# - รวมเข้ากับ class (include/extend/prepend)
# - ใช้เป็น namespace

# ตรวจสอบ
puts Greetable.class   # => Module
puts Greetable.is_a?(Module)  # => true

# ใช้งาน module ต้อง include เข้า class
class Person
  include Greetable
  attr_reader :name
  def initialize(name); @name = name; end
end

alice = Person.new("Alice")
alice.greet("Bob")     # => Hello, Bob! I'm Person
alice.farewell("Bob")  # => Goodbye, Bob!
```

### Module กับ Constants และ Methods

```ruby
module MathUtils
  PI = 3.14159265358979
  E  = 2.71828182845905

  def self.circle_area(r)
    PI * r ** 2
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

  def self.primes_up_to(limit)
    (2..limit).select { |n| prime?(n) }
  end
end

# ใช้ constants ผ่าน ::
puts MathUtils::PI
puts MathUtils::E

# เรียก module methods
puts MathUtils.circle_area(5)
puts MathUtils.factorial(10)
puts MathUtils.fibonacci(10)
puts MathUtils.prime?(17)
puts MathUtils.primes_up_to(30).inspect
```

---

## Step 257: Namespacing with Modules

Modules ช่วยจัดกลุ่มโค้ดและหลีกเลี่ยงชื่อที่ชนกัน

```ruby
# ปัญหาโดยไม่มี namespace
class User    # ชนกับ User อื่นๆ ได้
  def initialize(name)
    @name = name
  end
end

# แก้ด้วย namespace
module Authentication
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
      "Auth::User(#{@username})"
    end

    private

    def hash_password(pwd)
      # simplified - real code would use BCrypt
      pwd.chars.sum(&:ord).to_s(16)
    end
  end

  class Session
    attr_reader :user, :token, :created_at, :expires_at

    def initialize(user)
      @user = user
      @token = generate_token
      @created_at = Time.now
      @expires_at = Time.now + 3600  # 1 hour
    end

    def valid?
      Time.now < @expires_at
    end

    def expire!
      @expires_at = Time.now - 1
    end

    private

    def generate_token
      require 'securerandom'
      SecureRandom.hex(32)
    end
  end
end

module CMS
  class User
    attr_reader :display_name, :role

    def initialize(display_name, role = :editor)
      @display_name = display_name
      @role = role
    end

    def can_publish?
      [:admin, :publisher].include?(@role)
    end

    def to_s
      "CMS::User(#{@display_name}, #{@role})"
    end
  end

  class Article
    attr_accessor :title, :content, :author
    attr_reader :created_at, :published_at

    def initialize(title, content, author)
      @title = title
      @content = content
      @author = author
      @created_at = Time.now
      @published = false
    end

    def publish!(publisher)
      raise "Not authorized" unless publisher.can_publish?
      @published = true
      @published_at = Time.now
      puts "Published: #{@title}"
    end

    def published?
      @published
    end

    def to_s
      status = @published ? "published" : "draft"
      "Article: #{@title} (#{status}) by #{@author.display_name}"
    end
  end
end

# ทั้งสองใช้ชื่อ User แต่ต่าง namespace
auth_user = Authentication::User.new("alice", "alice@example.com", "secret123")
cms_user = CMS::User.new("Alice Smith", :publisher)

puts auth_user
puts cms_user
puts cms_user.can_publish?

article = CMS::Article.new("Ruby Modules", "Modules are great...", cms_user)
article.publish!(cms_user)
puts article
```

### Nested Modules

```ruby
module Company
  NAME = "RubyCorp"
  VERSION = "1.0.0"

  module HR
    class Employee
      def initialize(name)
        @name = name
        @company = Company::NAME
      end

      def to_s
        "#{@name} at #{@company}"
      end
    end

    module Payroll
      def self.calculate(salary, hours = 160)
        base = salary
        overtime = hours > 160 ? (hours - 160) * (salary / 160.0) * 1.5 : 0
        base + overtime
      end
    end
  end

  module Finance
    class Invoice
      def initialize(number, amount)
        @number = number
        @amount = amount
      end

      def to_s
        "Invoice ##{@number}: #{@amount}"
      end
    end
  end
end

emp = Company::HR::Employee.new("Alice")
puts emp

pay = Company::HR::Payroll.calculate(50000, 180)
puts "Salary with overtime: #{pay}"

invoice = Company::Finance::Invoice.new("001", 10000)
puts invoice
puts Company::NAME
puts Company::VERSION
```

---

## Step 258: include (Adds as Instance Methods)

`include` นำ module methods เข้ามาเป็น instance methods ของ class

```ruby
module Printable
  def print_info
    puts "=== #{self.class.name} ==="
    instance_variables.each do |var|
      puts "  #{var}: #{instance_variable_get(var)}"
    end
    puts "===================="
  end
end

module Serializable
  def to_json
    pairs = instance_variables.map do |var|
      name = var.to_s.delete('@')
      value = instance_variable_get(var)
      "\"#{name}\": #{value.inspect}"
    end
    "{#{pairs.join(', ')}}"
  end

  def to_csv_row
    instance_variables.map { |v| instance_variable_get(v) }.join(",")
  end
end

module Validatable
  def valid?
    validate.empty?
  end

  def invalid?
    !valid?
  end

  def errors
    validate
  end

  def validate
    []  # subclass/includer ควร override
  end
end

class Product
  include Printable
  include Serializable
  include Validatable

  attr_accessor :name, :price, :quantity

  def initialize(name, price, quantity)
    @name = name
    @price = price
    @quantity = quantity
  end

  def validate
    errors = []
    errors << "Name cannot be blank" if @name.nil? || @name.strip.empty?
    errors << "Price must be positive" unless @price && @price > 0
    errors << "Quantity cannot be negative" if @quantity && @quantity < 0
    errors
  end
end

laptop = Product.new("Laptop", 25000, 10)
laptop.print_info
puts laptop.to_json
puts laptop.valid?

bad_product = Product.new("", -100, -5)
puts bad_product.valid?
bad_product.errors.each { |e| puts "  - #{e}" }
```

### include กับ method resolution

```ruby
module A
  def hello
    "Hello from A"
  end
end

module B
  def hello
    "Hello from B"
  end
end

class MyClass
  include A
  include B  # B included last, จะ override A
end

obj = MyClass.new
puts obj.hello          # => Hello from B (B ถูก include หลัง)
puts MyClass.ancestors.inspect
# => [MyClass, B, A, Object, Kernel, BasicObject]
```

---

## Step 259: extend (Adds as Class Methods)

`extend` นำ module methods เข้ามาเป็น class methods

```ruby
module ClassMethods
  def create_from_hash(hash)
    obj = new
    hash.each do |key, value|
      setter = "#{key}="
      obj.send(setter, value) if obj.respond_to?(setter)
    end
    obj
  end

  def find_by(attribute, value, collection)
    collection.find { |obj| obj.send(attribute) == value }
  end

  def all_with(attribute, value, collection)
    collection.select { |obj| obj.send(attribute) == value }
  end

  def description
    "This is the #{name} class"
  end
end

module InstanceMethods
  def deep_dup
    Marshal.load(Marshal.dump(self))
  end
end

class User
  extend ClassMethods
  include InstanceMethods

  attr_accessor :name, :email, :role

  def initialize(name = nil, email = nil, role = "user")
    @name = name
    @email = email
    @role = role
  end

  def to_s
    "#{@name} <#{@email}> [#{@role}]"
  end
end

# เรียก class methods
puts User.description  # => This is the User class

alice = User.create_from_hash(name: "Alice", email: "alice@example.com", role: "admin")
bob = User.create_from_hash(name: "Bob", email: "bob@example.com")

puts alice
puts bob

users = [alice, bob, User.create_from_hash(name: "Charlie", email: "charlie@example.com")]

found = User.find_by(:name, "Bob", users)
puts "Found: #{found}"

admins = User.all_with(:role, "admin", users)
puts "Admins: #{admins.map(&:name).inspect}"

# instance method จาก module
alice_copy = alice.deep_dup
alice_copy.name = "Alice (copy)"
puts alice        # ไม่เปลี่ยน
puts alice_copy
```

### extend กับ singleton class

```ruby
module Singleton
  def instance
    @instance ||= new
  end

  private :new
end

class DatabaseConnection
  extend Singleton

  attr_reader :connected, :queries_run

  def initialize
    @connected = false
    @queries_run = 0
    @host = "localhost"
  end

  def connect(host = "localhost")
    @host = host
    @connected = true
    puts "Connected to #{@host}"
    self
  end

  def query(sql)
    raise "Not connected!" unless @connected
    @queries_run += 1
    puts "Query ##{@queries_run}: #{sql}"
    "result_#{@queries_run}"
  end

  def disconnect
    @connected = false
    puts "Disconnected"
  end
end

# ได้ instance เดียวกันเสมอ
db1 = DatabaseConnection.instance
db2 = DatabaseConnection.instance
puts db1.equal?(db2)  # => true

db1.connect("db.example.com")
db1.query("SELECT * FROM users")
db2.query("SELECT * FROM products")  # ใช้ connection เดียวกัน

puts "Total queries: #{DatabaseConnection.instance.queries_run}"
```

---

## Step 260: prepend (Adds Before Class in Method Lookup)

`prepend` ใส่ module ไว้หน้า class ใน method lookup chain

```ruby
module Logging
  def greet(name)
    puts "[LOG] #{self.class.name}#greet called with #{name.inspect}"
    result = super
    puts "[LOG] #{self.class.name}#greet returned"
    result
  end
end

module Timing
  def greet(name)
    start = Time.now
    result = super
    elapsed = Time.now - start
    puts "[TIMING] greet took #{(elapsed * 1000).round(3)}ms"
    result
  end
end

class Greeter
  prepend Logging
  prepend Timing

  def greet(name)
    puts "Hello, #{name}!"
    "greeting complete"
  end
end

puts Greeter.ancestors.inspect
# => [Timing, Logging, Greeter, Object, Kernel, BasicObject]
# Timing และ Logging อยู่ก่อน Greeter!

g = Greeter.new
g.greet("Alice")
```

### prepend สำหรับ Method Decoration

```ruby
module Memoizable
  def self.prepended(base)
    base.instance_variable_set(:@memo_cache, {})
    base.extend(ClassMethods)
  end

  module ClassMethods
    def memoize(method_name)
      original = instance_method(method_name)
      cache = {}

      define_method(method_name) do |*args|
        cache[args] ||= original.bind(self).call(*args)
      end
    end
  end
end

class Calculator
  prepend Memoizable

  def fibonacci(n)
    puts "Computing fib(#{n})"
    return n if n <= 1
    fibonacci(n - 1) + fibonacci(n - 2)
  end

  memoize :fibonacci
end

calc = Calculator.new
puts calc.fibonacci(10)  # computes
puts calc.fibonacci(10)  # ได้จาก cache (ไม่ print "Computing")
```

### prepend กับ around-method pattern

```ruby
module Validating
  def save
    if valid?
      super
    else
      puts "Validation failed: #{errors.join(', ')}"
      false
    end
  end
end

module Timestamping
  def save
    @updated_at = Time.now
    @created_at ||= Time.now
    super
  end
end

class Model
  prepend Validating
  prepend Timestamping

  attr_reader :updated_at, :created_at

  def save
    puts "Saving to database..."
    true
  end

  def valid?
    true  # override in subclass
  end

  def errors
    []
  end
end

class UserModel < Model
  attr_accessor :name, :email

  def valid?
    errors.empty?
  end

  def errors
    errs = []
    errs << "Name required" if @name.nil? || @name.empty?
    errs << "Email required" if @email.nil? || @email.empty?
    errs
  end
end

user = UserModel.new
user.name = "Alice"
user.email = "alice@example.com"
user.save  # success

invalid_user = UserModel.new
invalid_user.save  # validation fails
```

---

## Step 261: module_function

`module_function` ทำให้ method ใช้ได้ทั้งแบบ module method และ instance method (แต่ private)

```ruby
module StringHelpers
  module_function

  def titleize(str)
    str.split.map(&:capitalize).join(' ')
  end

  def truncate(str, length = 30, omission = "...")
    return str if str.length <= length
    str[0, length - omission.length] + omission
  end

  def word_count(str)
    str.split.length
  end

  def slugify(str)
    str.downcase
       .gsub(/[^\w\s-]/, '')
       .gsub(/\s+/, '-')
       .gsub(/-+/, '-')
       .strip
  end

  def palindrome?(str)
    clean = str.downcase.gsub(/[^a-z0-9]/, '')
    clean == clean.reverse
  end
end

# ใช้แบบ module method
puts StringHelpers.titleize("hello world ruby")
puts StringHelpers.truncate("This is a very long string that needs truncation")
puts StringHelpers.slugify("Hello World! This is a Slug")
puts StringHelpers.palindrome?("A man a plan a canal Panama")

# ใช้แบบ include (แต่ method จะเป็น private)
class Article
  include StringHelpers

  attr_reader :title, :slug

  def initialize(title)
    @title = titleize(title)  # private method
    @slug = slugify(title)    # private method
  end

  def summary(content, length = 100)
    truncate(content, length)  # private method
  end
end

article = Article.new("hello world my first ruby article")
puts article.title
puts article.slug
puts article.summary("Ruby is a dynamic, open source programming language with a focus on simplicity and productivity. It has an elegant syntax.")
```

---

## Step 262: Comparable Module

Comparable ช่วยให้ class มี comparison operators ทั้งหมดโดย implement เพียง `<=>`

```ruby
class Weight
  include Comparable

  attr_reader :value, :unit

  CONVERSIONS = {
    kg: 1.0,
    g:  0.001,
    lb: 0.453592,
    oz: 0.0283495
  }

  def initialize(value, unit = :kg)
    raise ArgumentError, "Unknown unit: #{unit}" unless CONVERSIONS.key?(unit)
    raise ArgumentError, "Weight cannot be negative" if value < 0
    @value = value.to_f
    @unit = unit
  end

  def in_kg
    @value * CONVERSIONS[@unit]
  end

  def convert_to(new_unit)
    raise ArgumentError, "Unknown unit: #{new_unit}" unless CONVERSIONS.key?(new_unit)
    new_value = in_kg / CONVERSIONS[new_unit]
    Weight.new(new_value, new_unit)
  end

  def +(other)
    Weight.new(in_kg + other.in_kg, :kg)
  end

  def -(other)
    diff = in_kg - other.in_kg
    raise "Result cannot be negative" if diff < 0
    Weight.new(diff, :kg)
  end

  def <=>(other)
    in_kg <=> other.in_kg
  end

  def to_s
    "#{format('%.2f', @value)} #{@unit}"
  end

  def inspect
    "#<Weight #{to_s} = #{format('%.4f', in_kg)} kg>"
  end
end

w1 = Weight.new(5, :kg)
w2 = Weight.new(3000, :g)
w3 = Weight.new(10, :lb)
w4 = Weight.new(2, :kg)

puts "#{w1} = #{w1.in_kg} kg"
puts "#{w2} = #{w2.in_kg} kg"
puts "#{w3} = #{w3.in_kg} kg"

puts "\nComparisons:"
puts "5kg > 3000g: #{w1 > w2}"       # true (5 > 3)
puts "10lb < 5kg: #{w3 < w1}"        # true (4.5 < 5)
puts "between? 2kg and 5kg: #{w3.between?(w4, w1)}"  # true

puts "\nSorted:"
weights = [w1, w2, w3, w4, Weight.new(500, :g)]
weights.sort.each { |w| puts "  #{w} = #{format('%.3f', w.in_kg)} kg" }

puts "\nMin: #{weights.min}"
puts "Max: #{weights.max}"

puts "\nConversion: #{w1.convert_to(:lb)}"
puts "Addition: #{w1 + w2}"
```

---

## Step 263: Enumerable Module

Enumerable ช่วยให้ class ใช้ collection methods ทั้งหมด โดย implement เพียง `each`

```ruby
class NumberList
  include Enumerable
  include Comparable

  def initialize(*numbers)
    @data = numbers
  end

  def each(&block)
    @data.each(&block)
  end

  def <=>(other)
    to_a <=> other.to_a
  end

  def to_s
    "NumberList[#{@data.join(', ')}]"
  end
end

list = NumberList.new(5, 3, 8, 1, 9, 2, 7, 4, 6)

# Enumerable methods ทั้งหมดใช้ได้
puts list.min              # => 1
puts list.max              # => 9
puts list.sum              # => 45
puts list.count            # => 9
puts list.sort.inspect     # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
puts list.select(&:odd?).inspect  # => [5, 3, 1, 9, 7]
puts list.map { |n| n * 2 }.inspect
puts list.reduce(:+)       # => 45
puts list.min_by { |n| -n }  # => 9
puts list.sort_by { |n| -n }.first(3).inspect  # => [9, 8, 7]
puts list.include?(7)      # => true
puts list.any? { |n| n > 8 }   # => true
puts list.all? { |n| n > 0 }   # => true
puts list.none? { |n| n > 10 } # => true
puts list.find { |n| n.even? && n > 5 }  # => 8
puts list.group_by(&:even?).inspect
puts list.tally.inspect
puts list.each_slice(3).to_a.inspect
puts list.each_cons(3).to_a.first(3).inspect
puts list.flat_map { |n| [n, n * 10] }.first(6).inspect
puts list.zip([10,20,30,40,50,60,70,80,90]).first(3).inspect
```

### Enumerable ใน Custom Collection

```ruby
class Playlist
  include Enumerable

  Song = Struct.new(:title, :artist, :duration_seconds, :genre) do
    def duration_str
      "#{duration_seconds / 60}:#{format('%02d', duration_seconds % 60)}"
    end

    def to_s
      "#{title} - #{artist} (#{duration_str}) [#{genre}]"
    end
  end

  def initialize(name)
    @name = name
    @songs = []
  end

  def add(title, artist, duration, genre = "Pop")
    @songs << Song.new(title, artist, duration, genre)
    self
  end

  def each(&block)
    @songs.each(&block)
  end

  def total_duration
    sum(&:duration_seconds)
  end

  def total_duration_str
    total = total_duration
    h = total / 3600
    m = (total % 3600) / 60
    s = total % 60
    h > 0 ? "#{h}:#{format('%02d', m)}:#{format('%02d', s)}" : "#{m}:#{format('%02d', s)}"
  end

  def by_genre
    group_by(&:genre)
  end

  def shuffle_play
    to_a.shuffle
  end

  def show
    puts "=== #{@name} (#{count} songs, #{total_duration_str}) ==="
    each_with_index do |song, i|
      puts "  #{i + 1}. #{song}"
    end
  end
end

playlist = Playlist.new("My Favorites")
playlist.add("Bohemian Rhapsody", "Queen", 354, "Rock")
        .add("Stairway to Heaven", "Led Zeppelin", 482, "Rock")
        .add("Hotel California", "Eagles", 391, "Rock")
        .add("Shape of You", "Ed Sheeran", 234, "Pop")
        .add("Blinding Lights", "The Weeknd", 200, "Pop")
        .add("Lose Yourself", "Eminem", 326, "Hip-Hop")

playlist.show

puts "\nRock songs:"
playlist.select { |s| s.genre == "Rock" }.each { |s| puts "  #{s.title}" }

puts "\nSorted by duration (longest first):"
playlist.sort_by { |s| -s.duration_seconds }.each { |s| puts "  #{s}" }

puts "\nBy genre:"
playlist.by_genre.each do |genre, songs|
  puts "#{genre}: #{songs.map(&:title).join(', ')}"
end

puts "\nTotal duration: #{playlist.total_duration_str}"
puts "Longest song: #{playlist.max_by(&:duration_seconds).title}"
puts "Average duration: #{(playlist.sum(&:duration_seconds).to_f / playlist.count).round}s"
```

---

## Step 264: Forwardable Module

Forwardable ช่วย delegate method calls ไปยัง object อื่น

```ruby
require 'forwardable'

class Printer
  def print_doc(doc)
    puts "Printing: #{doc}"
  end

  def print_photo(photo)
    puts "Printing photo: #{photo}"
  end

  def status
    "Online"
  end
end

class Scanner
  def scan(document)
    puts "Scanning: #{document}"
    "scanned_#{document}"
  end

  def resolution
    "600 DPI"
  end
end

class AllInOneMachine
  extend Forwardable

  # delegate methods to @printer
  def_delegators :@printer, :print_doc, :print_photo
  def_delegator :@printer, :status, :printer_status

  # delegate methods to @scanner
  def_delegators :@scanner, :scan
  def_delegator :@scanner, :resolution, :scan_resolution

  def initialize
    @printer = Printer.new
    @scanner = Scanner.new
  end

  def copy(document)
    scanned = scan(document)
    print_doc(scanned)
    "Copied: #{document}"
  end

  def info
    puts "Printer: #{printer_status}"
    puts "Scanner resolution: #{scan_resolution}"
  end
end

machine = AllInOneMachine.new
machine.print_doc("report.pdf")
machine.scan("invoice.pdf")
machine.copy("contract.pdf")
machine.info
```

### Forwardable กับ Composition

```ruby
require 'forwardable'

class Logger
  def log(level, msg)
    puts "[#{level.upcase}] #{msg}"
  end
end

class Cache
  def initialize
    @store = {}
  end

  def get(key)
    @store[key]
  end

  def set(key, value, ttl = nil)
    @store[key] = { value: value, expires: ttl ? Time.now + ttl : nil }
    value
  end

  def fetch(key)
    entry = get(key)
    entry && (entry[:expires].nil? || entry[:expires] > Time.now) ? entry[:value] : nil
  end

  def clear
    @store.clear
  end

  def size
    @store.size
  end
end

class DataService
  extend Forwardable

  def_delegators :@logger, :log
  def_delegators :@cache, :clear
  def_delegator :@cache, :size, :cache_size

  def initialize
    @logger = Logger.new
    @cache = Cache.new
  end

  def fetch_user(id)
    cached = @cache.fetch("user_#{id}")
    if cached
      log("debug", "Cache hit for user #{id}")
      cached
    else
      log("info", "Fetching user #{id} from database")
      user = { id: id, name: "User #{id}", email: "user#{id}@example.com" }
      @cache.set("user_#{id}", user, 300)
      user
    end
  end
end

service = DataService.new
user1 = service.fetch_user(1)
user2 = service.fetch_user(1)  # from cache
user3 = service.fetch_user(2)

puts "Cache size: #{service.cache_size}"
service.clear
puts "Cache size after clear: #{service.cache_size}"
```

---

## Step 265: Composition Over Inheritance

```ruby
# ปัญหาของ deep inheritance
class Animal
  def breathe; "breathe"; end
end

class Pet < Animal
  def be_petted; "purred"; end
end

class SwimmingPet < Pet
  def swim; "swims"; end
end

class FlyingSwimmingPet < SwimmingPet  # ลึกมากเกินไป!
  def fly; "flies"; end
end

# Composition แก้ปัญหาได้ดีกว่า
module Swimming
  def swim
    "#{self.class.name} swims!"
  end

  def dive(depth)
    "#{self.class.name} dives to #{depth}m"
  end
end

module Flying
  def fly
    "#{self.class.name} flies!"
  end

  def soar(altitude)
    "#{self.class.name} soars at #{altitude}m"
  end
end

module Running
  def run
    "#{self.class.name} runs!"
  end

  def sprint
    "#{self.class.name} sprints at full speed!"
  end
end

module Domestic
  def respond_to_name(name)
    "#{self.class.name} responds to #{name}"
  end
end

# Mix ตามที่ต้องการ
class Duck
  include Swimming
  include Flying
  include Domestic

  def quack
    "Quack!"
  end
end

class Dog
  include Running
  include Swimming  # บางสุนัขว่ายน้ำได้
  include Domestic
end

class Eagle
  include Flying
  include Running  # เดินได้
end

class Fish
  include Swimming
end

duck = Duck.new
puts duck.swim
puts duck.fly
puts duck.quack

dog = Dog.new
puts dog.run
puts dog.swim

eagle = Eagle.new
puts eagle.fly
puts eagle.run

fish = Fish.new
puts fish.swim
puts fish.dive(10)

# ตรวจสอบ capabilities
puts "\nCan duck fly? #{duck.is_a?(Duck) && duck.respond_to?(:fly)}"
puts "Can dog fly? #{dog.respond_to?(:fly)}"
puts "Can eagle swim? #{eagle.respond_to?(:swim)}"
```

---

## Step 266: Module Hooks

```ruby
# self.included - เรียกเมื่อ module ถูก include
module Trackable
  def self.included(base)
    puts "Trackable included in #{base.name}"
    base.extend(ClassMethods)
    base.instance_variable_set(:@tracked_instances, [])
  end

  module ClassMethods
    def tracked_instances
      @tracked_instances
    end

    def track_all
      @tracked_instances.each do |instance|
        puts "  #{instance}"
      end
    end
  end

  def initialize(*)
    super
    self.class.tracked_instances << self
  end
end

class Robot
  include Trackable

  attr_reader :id, :model

  def initialize(id, model)
    @id = id
    @model = model
    super()
  end

  def to_s
    "Robot[#{@id}]: #{@model}"
  end
end

r1 = Robot.new(1, "R2D2")
r2 = Robot.new(2, "C3PO")
r3 = Robot.new(3, "HAL-9000")

puts "Tracked robots:"
Robot.track_all
puts "Total: #{Robot.tracked_instances.length}"
```

### self.extended

```ruby
module ClassFinder
  def self.extended(base)
    puts "ClassFinder extended by #{base.name}"
    base.instance_variable_set(:@registry, {})
  end

  def register(key, value)
    @registry[key] = value
  end

  def find(key)
    @registry[key]
  end

  def registered
    @registry.keys
  end
end

class Plugin
  extend ClassFinder

  def self.load_all(names)
    names.map { |name| find(name) }.compact
  end
end

Plugin.register(:csv, "CSVPlugin")
Plugin.register(:json, "JSONPlugin")
Plugin.register(:xml, "XMLPlugin")

puts Plugin.registered.inspect
puts Plugin.find(:csv)
plugins = Plugin.load_all([:csv, :json, :pdf])  # :pdf ไม่มี
puts plugins.inspect
```

### self.prepended

```ruby
module Profiling
  def self.prepended(base)
    puts "Profiling prepended to #{base.name}"
    base.instance_variable_set(:@profile_data, Hash.new { |h, k| h[k] = [] })
    base.extend(ClassMethods)
  end

  module ClassMethods
    def profile_data
      @profile_data
    end

    def method_stats
      @profile_data.map do |method, times|
        avg = times.sum / times.length.to_f
        puts "#{method}: #{times.length} calls, avg #{format('%.4f', avg)}s"
      end
    end
  end
end

class SlowCalculator
  prepend Profiling

  def complex_calc(n)
    sum = 0
    n.times { |i| sum += Math.sqrt(i) }
    sum
  end
end

# เพิ่ม profiling wrapper
module Profiling
  def complex_calc(n)
    start = Time.now
    result = super
    elapsed = Time.now - start
    self.class.profile_data[:complex_calc] << elapsed
    result
  end
end

calc = SlowCalculator.new
3.times { |i| calc.complex_calc(1000 * (i + 1)) }
SlowCalculator.method_stats
```

---

## Step 267: Building Greetable Mixin

```ruby
module Greetable
  GREETINGS = {
    en: { hello: "Hello", goodbye: "Goodbye", morning: "Good morning",
          evening: "Good evening", how_are_you: "How are you?" },
    th: { hello: "สวัสดี", goodbye: "ลาก่อน", morning: "สวัสดีตอนเช้า",
          evening: "สวัสดีตอนเย็น", how_are_you: "สบายดีไหม?" },
    jp: { hello: "こんにちは", goodbye: "さようなら", morning: "おはようございます",
          evening: "こんばんは", how_are_you: "お元気ですか?" }
  }

  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def default_language(lang)
      @default_language = lang
    end

    def language
      @default_language || :en
    end
  end

  def greet(person_name = nil, lang: nil)
    selected_lang = lang || self.class.language
    greetings = GREETINGS[selected_lang] || GREETINGS[:en]
    target = person_name ? ", #{person_name}" : ""
    my_name = respond_to?(:name) ? " I'm #{name}." : ""
    "#{greetings[:hello]}#{target}!#{my_name}"
  end

  def say_goodbye(person_name = nil, lang: nil)
    selected_lang = lang || self.class.language
    greetings = GREETINGS[selected_lang] || GREETINGS[:en]
    target = person_name ? ", #{person_name}" : ""
    "#{greetings[:goodbye]}#{target}!"
  end

  def good_morning(lang: nil)
    selected_lang = lang || self.class.language
    (GREETINGS[selected_lang] || GREETINGS[:en])[:morning] + "!"
  end

  def introduce
    parts = ["My name is #{name}."] if respond_to?(:name)
    parts ||= []
    parts << "I am #{age} years old." if respond_to?(:age)
    parts << "I work as a #{job}." if respond_to?(:job)
    parts.join(" ")
  end
end

class ThaiStudent
  include Greetable
  default_language :th

  attr_reader :name, :age

  def initialize(name, age)
    @name = name
    @age = age
  end
end

class JapaneseTeacher
  include Greetable
  default_language :jp

  attr_reader :name, :job

  def initialize(name)
    @name = name
    @job = "Teacher"
  end
end

class InternationalPerson
  include Greetable

  attr_reader :name, :age, :job

  def initialize(name, age, job)
    @name = name
    @age = age
    @job = job
  end
end

student = ThaiStudent.new("สมชาย", 20)
teacher = JapaneseTeacher.new("Tanaka")
person = InternationalPerson.new("Alice", 30, "Developer")

puts student.greet("อาจารย์")
puts student.good_morning
puts student.say_goodbye

puts teacher.greet
puts teacher.introduce

puts person.greet("World")
puts person.greet("World", lang: :th)
puts person.introduce
```

---

## Step 268: Building Serializable Mixin

```ruby
require 'json'
require 'yaml'

module Serializable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@serializable_attrs, [])
  end

  module ClassMethods
    def serializable(*attrs)
      @serializable_attrs = attrs
    end

    def serializable_attrs
      @serializable_attrs
    end

    def from_json(json_string)
      data = JSON.parse(json_string, symbolize_names: true)
      from_hash(data)
    end

    def from_hash(hash)
      obj = allocate
      hash.each do |key, value|
        setter = "#{key}="
        obj.send(setter, value) if obj.respond_to?(setter)
      end
      obj
    end

    def from_yaml(yaml_string)
      data = YAML.safe_load(yaml_string, symbolize_names: true)
      from_hash(data)
    end
  end

  def to_h
    attrs = self.class.serializable_attrs
    if attrs.empty?
      instance_variables.each_with_object({}) do |var, hash|
        key = var.to_s.delete('@').to_sym
        hash[key] = instance_variable_get(var)
      end
    else
      attrs.each_with_object({}) do |attr, hash|
        hash[attr] = send(attr) if respond_to?(attr)
      end
    end
  end

  def to_json(*args)
    to_h.to_json(*args)
  end

  def to_yaml
    to_h.to_yaml
  end

  def to_csv_row(delimiter = ",")
    to_h.values.map { |v| "\"#{v}\"" }.join(delimiter)
  end

  def serialize(format = :json)
    case format
    when :json then to_json
    when :yaml then to_yaml
    when :hash then to_h
    else raise ArgumentError, "Unknown format: #{format}"
    end
  end

  def ==(other)
    other.is_a?(self.class) && to_h == other.to_h
  end
end

class Product
  include Serializable
  serializable :id, :name, :price, :category, :in_stock

  attr_accessor :id, :name, :price, :category, :in_stock

  def initialize(id, name, price, category, in_stock = true)
    @id = id
    @name = name
    @price = price
    @category = category
    @in_stock = in_stock
  end

  def to_s
    "Product[#{@id}]: #{@name} (฿#{@price})"
  end
end

class UserProfile
  include Serializable
  serializable :id, :username, :email, :created_at

  attr_accessor :id, :username, :email, :created_at

  def initialize(id, username, email)
    @id = id
    @username = username
    @email = email
    @created_at = Time.now.iso8601
  end
end

# ทดสอบ
laptop = Product.new(1, "Laptop", 25000, "Electronics")

puts "=== Serialization ==="
puts "To Hash: #{laptop.to_h.inspect}"
puts "To JSON: #{laptop.to_json}"
puts "To CSV: #{laptop.to_csv_row}"

json_str = laptop.to_json
restored = Product.from_json(json_str)
puts "\nRestored from JSON: #{restored}"
puts "Equal? #{laptop == restored}"

user = UserProfile.new(1, "alice", "alice@example.com")
puts "\nUser JSON: #{user.to_json}"
puts "User YAML:\n#{user.to_yaml}"
```

---

## Step 269: Building Validatable Mixin

```ruby
module Validatable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@validations, [])
  end

  module ClassMethods
    def validations
      @validations
    end

    def validates(attribute, **options)
      @validations << { attribute: attribute, options: options }
    end

    def validates_presence_of(*attributes)
      attributes.each { |attr| validates(attr, presence: true) }
    end

    def validates_length_of(attribute, min: nil, max: nil)
      validates(attribute, length: { min: min, max: max })
    end

    def validates_format_of(attribute, with:)
      validates(attribute, format: with)
    end

    def validates_numericality_of(attribute, greater_than: nil, less_than: nil, greater_than_or_equal_to: nil)
      validates(attribute, numericality: {
        greater_than: greater_than,
        less_than: less_than,
        greater_than_or_equal_to: greater_than_or_equal_to
      })
    end

    def validates_inclusion_of(attribute, in:)
      validates(attribute, inclusion: binding.local_variable_get(:in))
    end
  end

  def valid?
    @errors = []
    run_validations
    @errors.empty?
  end

  def invalid?
    !valid?
  end

  def errors
    @errors || []
  end

  def validate!
    raise ValidationError, errors.join(", ") unless valid?
    self
  end

  private

  def run_validations
    self.class.validations.each do |validation|
      attr = validation[:attribute]
      opts = validation[:options]
      value = respond_to?(attr) ? send(attr) : instance_variable_get("@#{attr}")

      check_presence(attr, value) if opts[:presence]
      check_length(attr, value, opts[:length]) if opts[:length]
      check_format(attr, value, opts[:format]) if opts[:format]
      check_numericality(attr, value, opts[:numericality]) if opts[:numericality]
      check_inclusion(attr, value, opts[:inclusion]) if opts[:inclusion]
    end

    # Custom validate method
    validate if respond_to?(:validate, true)
  end

  def check_presence(attr, value)
    if value.nil? || (value.respond_to?(:empty?) && value.empty?)
      @errors << "#{attr} can't be blank"
    end
  end

  def check_length(attr, value, opts)
    return unless value.respond_to?(:length)
    if opts[:min] && value.length < opts[:min]
      @errors << "#{attr} is too short (minimum #{opts[:min]} characters)"
    end
    if opts[:max] && value.length > opts[:max]
      @errors << "#{attr} is too long (maximum #{opts[:max]} characters)"
    end
  end

  def check_format(attr, value, pattern)
    @errors << "#{attr} is invalid" if value && value !~ pattern
  end

  def check_numericality(attr, value, opts)
    return unless value.is_a?(Numeric)
    if opts[:greater_than] && value <= opts[:greater_than]
      @errors << "#{attr} must be greater than #{opts[:greater_than]}"
    end
    if opts[:less_than] && value >= opts[:less_than]
      @errors << "#{attr} must be less than #{opts[:less_than]}"
    end
    if opts[:greater_than_or_equal_to] && value < opts[:greater_than_or_equal_to]
      @errors << "#{attr} must be >= #{opts[:greater_than_or_equal_to]}"
    end
  end

  def check_inclusion(attr, value, list)
    @errors << "#{attr} is not included in the list" unless list.include?(value)
  end
end

class ValidationError < StandardError; end

# ใช้งาน
class User
  include Validatable

  attr_accessor :name, :email, :age, :role, :password

  validates_presence_of :name, :email, :password
  validates_length_of :name, min: 2, max: 50
  validates_length_of :password, min: 8
  validates_format_of :email, with: /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates_numericality_of :age, greater_than_or_equal_to: 0, less_than: 150
  validates_inclusion_of :role, in: %w[admin user moderator]

  def initialize(attrs = {})
    attrs.each { |k, v| send("#{k}=", v) if respond_to?("#{k}=") }
  end

  private

  def validate
    if password && name && password.include?(name.downcase)
      @errors << "password cannot contain your name"
    end
  end
end

# Valid user
alice = User.new(
  name: "Alice Smith",
  email: "alice@example.com",
  age: 30,
  role: "admin",
  password: "SuperSecure123!"
)

puts "Alice valid? #{alice.valid?}"

# Invalid user
bad_user = User.new(
  name: "A",
  email: "not-an-email",
  age: 200,
  role: "superuser",
  password: "short"
)

puts "Bad user valid? #{bad_user.valid?}"
puts "Errors:"
bad_user.errors.each { |e| puts "  - #{e}" }

# validate!
begin
  bad_user.validate!
rescue ValidationError => e
  puts "Validation failed: #{e.message}"
end
```

---

## แบบฝึกหัด 25 ข้อ พร้อมเฉลย

### ข้อที่ 1-5

```ruby
# ข้อที่ 1: สร้าง Taggable module
module Taggable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def tagged_with(tag, collection)
      collection.select { |item| item.respond_to?(:tags) && item.tags.include?(tag) }
    end
  end

  def tags
    @tags ||= []
  end

  def add_tag(*new_tags)
    @tags = (tags + new_tags).uniq
    self
  end

  def remove_tag(tag)
    @tags = tags.reject { |t| t == tag }
    self
  end

  def tagged_with?(tag)
    tags.include?(tag)
  end

  def tags_string
    tags.map { |t| "##{t}" }.join(" ")
  end
end

class Article
  include Taggable
  attr_reader :title

  def initialize(title)
    @title = title
  end

  def to_s
    "#{@title} [#{tags_string}]"
  end
end

a1 = Article.new("Ruby OOP")
a2 = Article.new("Python Basics")
a3 = Article.new("Ruby Modules")

a1.add_tag("ruby", "oop", "programming")
a2.add_tag("python", "beginner", "programming")
a3.add_tag("ruby", "modules", "programming")

puts a1
puts a2
puts a3

articles = [a1, a2, a3]
ruby_articles = Article.tagged_with("ruby", articles)
puts "\nRuby articles:"
ruby_articles.each { |a| puts "  #{a.title}" }
```

```ruby
# ข้อที่ 2: Auditable module
module Auditable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def audit_log
      @audit_log ||= []
    end

    def recent_changes(n = 10)
      audit_log.last(n)
    end
  end

  def audit_change(action, details = {})
    entry = {
      action: action,
      object: "#{self.class.name}##{respond_to?(:id) ? id : object_id}",
      details: details,
      timestamp: Time.now
    }
    self.class.audit_log << entry
    puts "[AUDIT] #{entry[:action]}: #{entry[:object]}"
  end
end

class BankAccount
  include Auditable
  attr_reader :id, :balance, :owner

  @@next_id = 0

  def initialize(owner, balance = 0)
    @@next_id += 1
    @id = @@next_id
    @owner = owner
    @balance = balance
    audit_change(:account_created, owner: owner, initial_balance: balance)
  end

  def deposit(amount)
    @balance += amount
    audit_change(:deposit, amount: amount, new_balance: @balance)
    self
  end

  def withdraw(amount)
    raise "Insufficient funds" if amount > @balance
    @balance -= amount
    audit_change(:withdrawal, amount: amount, new_balance: @balance)
    self
  end
end

acc = BankAccount.new("Alice", 1000)
acc.deposit(500)
acc.withdraw(200)
acc.deposit(1000)

puts "\nAudit Log:"
BankAccount.recent_changes.each do |entry|
  puts "  #{entry[:timestamp].strftime('%H:%M:%S')} - #{entry[:action]}: #{entry[:details]}"
end
```

```ruby
# ข้อที่ 3: Cacheable module
module Cacheable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@cache, {})
    base.instance_variable_set(:@cache_ttl, 300)
  end

  module ClassMethods
    def cache
      @cache
    end

    def cache_ttl(seconds)
      @cache_ttl = seconds
    end

    def get_ttl
      @cache_ttl
    end

    def cached(key, &block)
      entry = @cache[key]
      if entry && entry[:expires] > Time.now
        puts "[CACHE HIT] #{key}"
        entry[:value]
      else
        puts "[CACHE MISS] #{key}"
        value = block.call
        @cache[key] = { value: value, expires: Time.now + @cache_ttl }
        value
      end
    end

    def invalidate(key)
      @cache.delete(key)
      puts "[CACHE INVALIDATED] #{key}"
    end

    def clear_cache
      @cache.clear
      puts "[CACHE CLEARED]"
    end
  end
end

class ProductService
  include Cacheable
  cache_ttl 60

  def self.find(id)
    cached("product_#{id}") do
      # Simulate slow database query
      sleep(0.01)
      { id: id, name: "Product #{id}", price: id * 100 }
    end
  end

  def self.featured
    cached("featured_products") do
      sleep(0.01)
      [find(1), find(2), find(3)]
    end
  end
end

p1 = ProductService.find(1)
p2 = ProductService.find(1)  # cache hit
p3 = ProductService.find(2)
featured = ProductService.featured

puts p1.inspect
puts "Cache size: #{ProductService.cache.size}"

ProductService.invalidate("product_1")
p4 = ProductService.find(1)  # cache miss again
```

```ruby
# ข้อที่ 4: Observable module (ปรับปรุงจาก Part 11)
module Observable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@event_handlers, Hash.new { |h, k| h[k] = [] })
  end

  module ClassMethods
    def on(event, &handler)
      @event_handlers[event] << handler
    end

    def event_handlers
      @event_handlers
    end
  end

  def on(event, &handler)
    @event_handlers ||= Hash.new { |h, k| h[k] = [] }
    @event_handlers[event] << handler
    self
  end

  def emit(event, *args)
    # Class-level handlers
    (self.class.event_handlers[event] || []).each { |h| h.call(*args) }
    # Instance-level handlers
    (@event_handlers || {})[event]&.each { |h| h.call(*args) }
  end
end

class EventBus
  include Observable

  def publish(topic, payload)
    emit(topic, payload)
  end
end

class OrderSystem
  include Observable

  attr_reader :orders

  def initialize
    @orders = []
  end

  def create_order(items, customer)
    order = { id: @orders.length + 1, items: items, customer: customer, status: :pending }
    @orders << order
    emit(:order_created, order)
    order
  end

  def fulfill_order(order_id)
    order = @orders.find { |o| o[:id] == order_id }
    return unless order
    order[:status] = :fulfilled
    emit(:order_fulfilled, order)
    order
  end
end

# Setup listeners
OrderSystem.on(:order_created) do |order|
  puts "[EMAIL] Order confirmation sent for Order ##{order[:id]}"
end

OrderSystem.on(:order_created) do |order|
  puts "[INVENTORY] Reserving items for Order ##{order[:id]}"
end

OrderSystem.on(:order_fulfilled) do |order|
  puts "[SHIPPING] Preparing shipment for Order ##{order[:id]}"
end

system = OrderSystem.new
order = system.create_order(["item1", "item2"], "Alice")
system.fulfill_order(order[:id])
```

```ruby
# ข้อที่ 5: Searchable module
module Searchable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@search_fields, [])
  end

  module ClassMethods
    def search_by(*fields)
      @search_fields = fields
    end

    def search_fields
      @search_fields
    end

    def search(query, collection)
      query = query.to_s.downcase
      collection.select do |item|
        @search_fields.any? do |field|
          value = item.respond_to?(field) ? item.send(field).to_s : ""
          value.downcase.include?(query)
        end
      end
    end
  end
end

class Book
  include Searchable
  search_by :title, :author, :genre

  attr_accessor :title, :author, :genre, :year

  def initialize(title, author, genre, year)
    @title = title
    @author = author
    @genre = genre
    @year = year
  end

  def to_s
    "\"#{@title}\" by #{@author} (#{@genre}, #{@year})"
  end
end

books = [
  Book.new("The Ruby Way", "Fulton", "Programming", 2015),
  Book.new("Clean Code", "Martin", "Programming", 2008),
  Book.new("Design Patterns", "Gang of Four", "Programming", 1994),
  Book.new("Harry Potter", "Rowling", "Fiction", 1997),
  Book.new("The Hobbit", "Tolkien", "Fantasy", 1937),
]

puts "Search 'ruby':"
Book.search("ruby", books).each { |b| puts "  #{b}" }

puts "\nSearch 'programming':"
Book.search("programming", books).each { |b| puts "  #{b}" }

puts "\nSearch 'tolkien':"
Book.search("tolkien", books).each { |b| puts "  #{b}" }
```

### ข้อที่ 6-15

```ruby
# ข้อที่ 6: Pageable module
module Pageable
  def paginate(collection, page: 1, per_page: 10)
    total = collection.length
    total_pages = (total.to_f / per_page).ceil

    start_idx = (page - 1) * per_page
    items = collection[start_idx, per_page] || []

    {
      items: items,
      page: page,
      per_page: per_page,
      total: total,
      total_pages: total_pages,
      has_next: page < total_pages,
      has_prev: page > 1
    }
  end

  def display_page(result)
    puts "Page #{result[:page]}/#{result[:total_pages]} " \
         "(#{result[:total]} total, #{result[:per_page]} per page)"
    result[:items].each_with_index do |item, i|
      idx = (result[:page] - 1) * result[:per_page] + i + 1
      puts "  #{idx}. #{item}"
    end
    nav = []
    nav << "[← prev]" if result[:has_prev]
    nav << "[next →]" if result[:has_next]
    puts nav.join(" ") unless nav.empty?
  end
end

class ProductCatalog
  include Pageable

  def initialize
    @products = (1..45).map { |i| "Product #{format('%03d', i)}" }
  end

  def list(page: 1, per_page: 10)
    display_page(paginate(@products, page: page, per_page: per_page))
  end
end

catalog = ProductCatalog.new
puts "=== Page 1 ==="
catalog.list(page: 1, per_page: 10)
puts "\n=== Page 3 ==="
catalog.list(page: 3, per_page: 10)
puts "\n=== Last Page ==="
catalog.list(page: 5, per_page: 10)
```

```ruby
# ข้อที่ 7: Configurable module
module Configurable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@config, {})
  end

  module ClassMethods
    def config
      @config
    end

    def configure
      yield @config
    end

    def set(key, value)
      @config[key] = value
    end

    def get(key, default = nil)
      @config.fetch(key, default)
    end
  end

  def config
    self.class.config
  end

  def get_config(key, default = nil)
    self.class.get(key, default)
  end
end

class PaymentGateway
  include Configurable

  configure do |c|
    c[:api_url] = "https://api.payment.com"
    c[:timeout] = 30
    c[:currency] = "THB"
    c[:max_retries] = 3
  end

  def charge(amount, card_token)
    puts "Charging #{get_config(:currency)} #{amount}"
    puts "Using API: #{get_config(:api_url)}"
    puts "Timeout: #{get_config(:timeout)}s"
    { success: true, transaction_id: "TXN#{rand(100000)}" }
  end
end

gateway = PaymentGateway.new
result = gateway.charge(500, "tok_abc123")
puts result.inspect

# Override config for testing
PaymentGateway.set(:api_url, "https://sandbox.payment.com")
PaymentGateway.set(:currency, "USD")
puts "\nTest mode:"
gateway.charge(100, "tok_test")
```

```ruby
# ข้อที่ 8: Hookable module
module Hookable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def before_hooks
      @before_hooks ||= Hash.new { |h, k| h[k] = [] }
    end

    def after_hooks
      @after_hooks ||= Hash.new { |h, k| h[k] = [] }
    end

    def before(method_name, &block)
      before_hooks[method_name] << block
    end

    def after(method_name, &block)
      after_hooks[method_name] << block
    end

    def hookable(method_name)
      original = instance_method(method_name)

      define_method(method_name) do |*args, &blk|
        self.class.before_hooks[method_name].each { |h| instance_exec(*args, &h) }
        result = original.bind(self).call(*args, &blk)
        self.class.after_hooks[method_name].each { |h| instance_exec(result, &h) }
        result
      end
    end
  end
end

class Order
  include Hookable

  attr_reader :id, :total, :status

  before(:complete) { puts "[Before] Validating order #{@id}" }
  before(:complete) { puts "[Before] Checking inventory" }
  after(:complete) { |result| puts "[After] Sending confirmation email" if result }

  def initialize(id, items)
    @id = id
    @items = items
    @total = items.sum { |i| i[:price] }
    @status = :pending
  end

  def complete
    puts "Processing order #{@id}..."
    @status = :completed
    puts "Order #{@id} completed! Total: ฿#{@total}"
    true
  end

  hookable :complete
end

order = Order.new(1, [
  { name: "Book", price: 350 },
  { name: "Pen", price: 50 }
])

order.complete
```

```ruby
# ข้อที่ 9: Memoizable module
module Memoizable
  def memoize(*method_names)
    method_names.each do |method_name|
      original = instance_method(method_name)
      cache_var = "@_memo_#{method_name}"

      define_method(method_name) do |*args|
        cache = instance_variable_get(cache_var) ||
                instance_variable_set(cache_var, {})
        key = args
        cache.key?(key) ? cache[key] : (cache[key] = original.bind(self).call(*args))
      end

      define_method("clear_memo_#{method_name}") do
        remove_instance_variable(cache_var) if instance_variable_defined?(cache_var)
      end
    end
  end
end

class ExpensiveService
  extend Memoizable

  def fetch_user(id)
    puts "Fetching user #{id} from database..."
    sleep(0.01)
    { id: id, name: "User #{id}", email: "user#{id}@example.com" }
  end

  def calculate_report(year, month)
    puts "Calculating report for #{year}/#{month}..."
    sleep(0.01)
    { year: year, month: month, revenue: rand(100000..999999) }
  end

  memoize :fetch_user, :calculate_report
end

service = ExpensiveService.new

puts "First call:"
u1 = service.fetch_user(1)
puts "Second call (memoized):"
u2 = service.fetch_user(1)
puts "Same result? #{u1 == u2}"

puts "\nReport calls:"
r1 = service.calculate_report(2024, 1)
r2 = service.calculate_report(2024, 1)  # memoized
r3 = service.calculate_report(2024, 2)  # new calculation

puts r1.inspect
puts r2.inspect
puts r3.inspect
```

```ruby
# ข้อที่ 10: Filterable module  
module Filterable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@filters, [])
  end

  module ClassMethods
    def filter_by(*attributes)
      @filters = attributes
      attributes.each do |attr|
        define_method("with_#{attr}") do |collection, value|
          collection.select { |item| item.respond_to?(attr) && item.send(attr) == value }
        end
      end
    end
  end

  def filter(collection, **criteria)
    result = collection
    criteria.each do |attr, value|
      result = result.select { |item| item.respond_to?(attr) && item.send(attr) == value }
    end
    result
  end

  def sort_collection(collection, by:, direction: :asc)
    sorted = collection.sort_by { |item| item.respond_to?(by) ? item.send(by) : 0 }
    direction == :desc ? sorted.reverse : sorted
  end
end

class ProductFilter
  include Filterable
  filter_by :category, :brand, :in_stock

  def price_range(collection, min:, max:)
    collection.select { |p| p.price.between?(min, max) }
  end
end

Product = Struct.new(:name, :category, :brand, :price, :in_stock)

products = [
  Product.new("MacBook", "Laptop", "Apple", 45000, true),
  Product.new("ThinkPad", "Laptop", "Lenovo", 35000, true),
  Product.new("iPhone", "Phone", "Apple", 30000, false),
  Product.new("Galaxy S24", "Phone", "Samsung", 28000, true),
  Product.new("iPad", "Tablet", "Apple", 22000, true),
]

pf = ProductFilter.new

puts "Apple products:"
pf.with_brand(products, "Apple").each { |p| puts "  #{p.name}: ฿#{p.price}" }

puts "\nLaptops in stock:"
pf.filter(products, category: "Laptop", in_stock: true).each { |p| puts "  #{p.name}" }

puts "\nPhones sorted by price:"
phones = pf.filter(products, category: "Phone")
pf.sort_collection(phones, by: :price).each { |p| puts "  #{p.name}: ฿#{p.price}" }
```

### ข้อที่ 11-25

```ruby
# ข้อที่ 11: Retry mechanism module
module Retryable
  def with_retry(max_attempts: 3, wait: 1, exceptions: [StandardError], &block)
    attempts = 0

    begin
      attempts += 1
      block.call
    rescue *exceptions => e
      if attempts < max_attempts
        puts "[RETRY] Attempt #{attempts} failed: #{e.message}. Retrying in #{wait}s..."
        sleep(wait)
        retry
      else
        puts "[RETRY] All #{max_attempts} attempts failed."
        raise
      end
    end
  end
end

class APIClient
  include Retryable

  def initialize(success_after = 3)
    @attempt = 0
    @success_after = success_after
  end

  def fetch_data(endpoint)
    with_retry(max_attempts: 4, wait: 0) do
      @attempt += 1
      raise "Connection timeout" if @attempt < @success_after
      puts "Successfully fetched: #{endpoint}"
      { status: "ok", data: "response_data" }
    end
  end
end

client = APIClient.new(3)
result = client.fetch_data("/api/users")
puts result.inspect
```

```ruby
# ข้อที่ 12: Immutable module
module Immutable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def immutable(*attrs)
      attrs.each do |attr|
        define_method("#{attr}=") do |_|
          raise "Cannot modify immutable attribute: #{attr}"
        end
      end
    end
  end

  def freeze_all
    freeze
  end
end

class Currency
  include Immutable
  immutable :code, :symbol, :decimal_places

  attr_reader :code, :symbol, :decimal_places

  def initialize(code, symbol, decimal_places)
    @code = code
    @symbol = symbol
    @decimal_places = decimal_places
  end

  def format_amount(amount)
    "#{@symbol}#{format("%.#{@decimal_places}f", amount)}"
  end

  def to_s
    "#{@code} (#{@symbol})"
  end
end

thb = Currency.new("THB", "฿", 2)
usd = Currency.new("USD", "$", 2)
jpy = Currency.new("JPY", "¥", 0)

puts thb.format_amount(1234.50)
puts usd.format_amount(99.99)
puts jpy.format_amount(15000)

begin
  thb.code = "HACK"
rescue => e
  puts "Error: #{e.message}"
end
```

```ruby
# ข้อที่ 13: Exportable module
require 'json'
require 'csv'

module Exportable
  def export_to_json(filename = nil)
    data = respond_to?(:to_export_data) ? to_export_data : to_h
    json = JSON.pretty_generate(data)
    if filename
      File.write(filename, json)
      puts "Exported to #{filename}"
    end
    json
  end

  def export_to_csv(filename = nil)
    data = respond_to?(:to_export_data) ? to_export_data : [to_h]
    data = [data] unless data.is_a?(Array)
    return "" if data.empty?

    headers = data.first.keys
    csv_string = CSV.generate do |csv|
      csv << headers
      data.each { |row| csv << headers.map { |h| row[h] } }
    end

    if filename
      File.write(filename, csv_string)
      puts "Exported to #{filename}"
    end
    csv_string
  end
end

class SalesReport
  include Exportable

  def initialize(period)
    @period = period
    @data = []
  end

  def add_sale(product, quantity, amount)
    @data << { product: product, quantity: quantity, amount: amount }
    self
  end

  def to_export_data
    {
      period: @period,
      total_sales: @data.sum { |r| r[:amount] },
      total_items: @data.sum { |r| r[:quantity] },
      records: @data
    }
  end
end

report = SalesReport.new("2024-Q1")
report.add_sale("Laptop", 5, 125000)
      .add_sale("Mouse", 20, 10000)
      .add_sale("Keyboard", 15, 18000)

puts report.export_to_json
puts report.export_to_csv
```

```ruby
# ข้อที่ 14: Plugin system
module Pluggable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@plugins, [])
  end

  module ClassMethods
    def plugins
      @plugins
    end

    def use(plugin, **options)
      @plugins << { plugin: plugin, options: options }
      plugin.setup(self, **options) if plugin.respond_to?(:setup)
    end
  end
end

module AuthPlugin
  def self.setup(base, token_expiry: 3600)
    puts "Setting up AuthPlugin (expiry: #{token_expiry}s)"
    base.include(InstanceMethods)
  end

  module InstanceMethods
    def authenticate(token)
      puts "Authenticating with token: #{token[0..7]}..."
      true
    end
  end
end

module RateLimitPlugin
  def self.setup(base, max_requests: 100)
    puts "Setting up RateLimitPlugin (max: #{max_requests} req/hour)"
    base.include(InstanceMethods)
    base.instance_variable_set(:@rate_limit, max_requests)
  end

  module InstanceMethods
    def rate_limited?
      false  # simplified
    end
  end
end

class APIServer
  include Pluggable

  use AuthPlugin, token_expiry: 1800
  use RateLimitPlugin, max_requests: 500

  def handle_request(path, token)
    return { error: "Rate limited" } if rate_limited?
    return { error: "Unauthorized" } unless authenticate(token)
    { path: path, status: 200, data: "response" }
  end
end

server = APIServer.new
result = server.handle_request("/api/users", "secret_token_xyz")
puts result.inspect

puts "\nPlugins loaded:"
APIServer.plugins.each { |p| puts "  #{p[:plugin].name}: #{p[:options]}" }
```

```ruby
# ข้อที่ 15-25: Advanced modules

# ข้อที่ 15: Decorator mixin
module Decoratable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def decorate_method(method_name, with:)
      original = instance_method(method_name)
      decorator = with

      define_method(method_name) do |*args, &block|
        decorator.call(original.bind(self), args, block)
      end
    end
  end
end

class MeasuredCache
  include Decoratable

  def initialize
    @store = {}
    @hit_count = 0
    @miss_count = 0
  end

  def get(key)
    @store[key]
  end

  def set(key, value)
    @store[key] = value
  end

  decorate_method :get, with: ->(original, args, _) {
    result = original.call(*args)
    if result.nil?
      puts "[CACHE MISS] #{args[0]}"
    else
      puts "[CACHE HIT] #{args[0]}"
    end
    result
  }
end

cache = MeasuredCache.new
cache.set("user:1", { name: "Alice" })
cache.set("user:2", { name: "Bob" })

cache.get("user:1")
cache.get("user:1")
cache.get("user:3")
cache.get("user:2")
```

```ruby
# ข้อที่ 16-20

# ข้อที่ 16: Pipeline module
module Pipelineable
  def pipe(*operations)
    operations.reduce(self) do |result, operation|
      case operation
      when Proc, Method
        operation.call(result)
      when Symbol
        result.send(operation)
      end
    end
  end
end

class DataTransform
  include Pipelineable

  attr_reader :data

  def initialize(data)
    @data = data
  end

  def filter_positive
    DataTransform.new(@data.select { |n| n > 0 })
  end

  def double
    DataTransform.new(@data.map { |n| n * 2 })
  end

  def sort_desc
    DataTransform.new(@data.sort.reverse)
  end

  def take_first(n)
    DataTransform.new(@data.first(n))
  end

  def to_s
    "DataTransform(#{@data.inspect})"
  end
end

transform = DataTransform.new([-3, 7, -1, 4, 2, -5, 8, 1, 6])

result = transform
  .filter_positive
  .double
  .sort_desc
  .take_first(4)

puts result
```

```ruby
# ข้อที่ 17: EventEmitter module
module EventEmitter
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def on(event, &handler)
      class_level_handlers[event] ||= []
      class_level_handlers[event] << handler
    end

    def class_level_handlers
      @handlers ||= {}
    end
  end

  def on(event, &handler)
    @handlers ||= {}
    @handlers[event] ||= []
    @handlers[event] << handler
    self
  end

  def off(event, handler = nil)
    @handlers ||= {}
    if handler
      @handlers[event]&.delete(handler)
    else
      @handlers.delete(event)
    end
    self
  end

  def emit(event, *data)
    handlers = (@handlers || {})[event] || []
    class_handlers = self.class.class_level_handlers[event] || []
    (class_handlers + handlers).each { |h| h.call(*data) }
    self
  end

  def once(event, &handler)
    wrapper = nil
    wrapper = lambda do |*args|
      handler.call(*args)
      off(event, wrapper)
    end
    on(event, &wrapper)
  end
end

class FileWatcher
  include EventEmitter

  def initialize(path)
    @path = path
    @watching = false
  end

  def start
    @watching = true
    emit(:start, @path)
    self
  end

  def simulate_change(filename)
    emit(:change, filename, Time.now)
  end

  def simulate_delete(filename)
    emit(:delete, filename)
  end

  def stop
    @watching = false
    emit(:stop)
    self
  end
end

watcher = FileWatcher.new("/home/user/docs")

watcher.on(:start) { |path| puts "Started watching: #{path}" }
watcher.on(:change) { |file, time| puts "Changed: #{file} at #{time.strftime('%H:%M:%S')}" }
watcher.on(:delete) { |file| puts "Deleted: #{file}" }
watcher.on(:stop) { puts "Stopped watching" }

watcher.start
watcher.simulate_change("document.txt")
watcher.simulate_change("notes.md")
watcher.simulate_delete("temp.tmp")
watcher.stop
```

```ruby
# ข้อที่ 18-25: Final exercises

# ข้อที่ 18: Lazy evaluation
module Lazy
  def lazy_attr(*names)
    names.each do |name|
      define_method(name) do
        var = "@_lazy_#{name}"
        unless instance_variable_defined?(var)
          instance_variable_set(var, yield_if_block(name))
        end
        instance_variable_get(var)
      end
    end
  end

  def yield_if_block(name)
    nil  # override to provide lazy initialization
  end
end

class ExpensiveObject
  def initialize
    puts "Creating ExpensiveObject (fast)"
  end

  def expensive_data
    @expensive_data ||= begin
      puts "Loading expensive data (slow)..."
      sleep(0.01)
      (1..1000).map { |i| i * i }
    end
  end

  def report
    @report ||= begin
      puts "Generating report (slow)..."
      sleep(0.01)
      "Report: #{expensive_data.sum} total"
    end
  end
end

obj = ExpensiveObject.new
puts "Object created, no expensive ops yet"
puts "First report: #{obj.report}"
puts "Second report (cached): #{obj.report}"
puts "Data sum: #{obj.expensive_data.sum}"
```

```ruby
# ข้อที่ 19-25: More modules showcase

# ข้อที่ 19: Role-based access control
module RBAC
  PERMISSIONS = {}

  def self.define_role(role, *permissions)
    PERMISSIONS[role] = permissions
  end

  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def require_permission(method_name, permission)
      original = instance_method(method_name)
      define_method(method_name) do |*args, &block|
        unless has_permission?(permission)
          raise "Access denied: #{permission} required"
        end
        original.bind(self).call(*args, &block)
      end
    end
  end

  def has_permission?(permission)
    role = respond_to?(:role) ? self.role : :guest
    (RBAC::PERMISSIONS[role] || []).include?(permission)
  end
end

RBAC.define_role(:guest,  :read)
RBAC.define_role(:user,   :read, :create)
RBAC.define_role(:editor, :read, :create, :update)
RBAC.define_role(:admin,  :read, :create, :update, :delete)

class ContentManager
  include RBAC

  attr_reader :name, :role

  def initialize(name, role)
    @name = name
    @role = role
  end

  def view_content
    puts "#{@name} viewing content"
  end
  require_permission :view_content, :read

  def create_post(title)
    puts "#{@name} created: #{title}"
  end
  require_permission :create_post, :create

  def delete_post(id)
    puts "#{@name} deleted post #{id}"
  end
  require_permission :delete_post, :delete
end

admin = ContentManager.new("Admin Alice", :admin)
user = ContentManager.new("Regular Bob", :user)

admin.view_content
admin.create_post("Hello World")
admin.delete_post(42)

user.view_content
user.create_post("My Post")

begin
  user.delete_post(42)
rescue => e
  puts "Error: #{e.message}"
end
```

```ruby
# ข้อที่ 20-25

# ข้อที่ 20: State machine mixin
module StateMachine
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@states, {})
    base.instance_variable_set(:@transitions, {})
    base.instance_variable_set(:@initial_state, nil)
  end

  module ClassMethods
    def state(name, **callbacks)
      @states[name] = callbacks
    end

    def transition(from:, to:, on:, guard: nil)
      @transitions[on] ||= []
      @transitions[on] << { from: from, to: to, guard: guard }
    end

    def initial_state(state)
      @initial_state = state
    end

    def all_states; @states; end
    def all_transitions; @transitions; end
    def get_initial_state; @initial_state; end
  end

  def initialize(*)
    super
    @state = self.class.get_initial_state
    callbacks = self.class.all_states[@state]
    send(callbacks[:on_enter]) if callbacks&.key?(:on_enter) && respond_to?(callbacks[:on_enter], true)
  end

  def state
    @state
  end

  def trigger(event)
    possible = self.class.all_transitions[event] || []
    transition = possible.find do |t|
      (t[:from] == :any || t[:from] == @state) &&
      (t[:guard].nil? || send(t[:guard]))
    end

    unless transition
      puts "Cannot trigger #{event} from state #{@state}"
      return false
    end

    old_state = @state
    exit_callbacks = self.class.all_states[@state]
    send(exit_callbacks[:on_exit]) if exit_callbacks&.key?(:on_exit) && respond_to?(exit_callbacks[:on_exit], true)

    @state = transition[:to]
    enter_callbacks = self.class.all_states[@state]
    send(enter_callbacks[:on_enter]) if enter_callbacks&.key?(:on_enter) && respond_to?(enter_callbacks[:on_enter], true)

    puts "#{self.class.name}: #{old_state} → #{@state} (#{event})"
    true
  end

  def can?(event)
    (self.class.all_transitions[event] || []).any? do |t|
      (t[:from] == :any || t[:from] == @state) &&
      (t[:guard].nil? || send(t[:guard]))
    end
  end
end

class TrafficLight
  include StateMachine

  initial_state :red
  state :red,    on_enter: :turn_red
  state :yellow, on_enter: :turn_yellow
  state :green,  on_enter: :turn_green

  transition from: :red,    to: :green,  on: :go
  transition from: :green,  to: :yellow, on: :slow
  transition from: :yellow, to: :red,    on: :stop

  def turn_red;    puts "🔴 RED - Stop!"; end
  def turn_yellow; puts "🟡 YELLOW - Slow down!"; end
  def turn_green;  puts "🟢 GREEN - Go!"; end
end

light = TrafficLight.new
puts "State: #{light.state}"
light.trigger(:go)
light.trigger(:slow)
light.trigger(:stop)
light.trigger(:stop)  # invalid transition
puts "Can go? #{light.can?(:go)}"
```

```ruby
# ข้อที่ 21-25 combined showcase

# ข้อที่ 21: Aspect-Oriented Programming style
module Aspect
  def self.around(object, method_name, &wrapper)
    original = object.method(method_name)
    object.define_singleton_method(method_name) do |*args, &block|
      wrapper.call(original, args, block)
    end
  end
end

class UserService
  def create_user(name, email)
    puts "Creating user: #{name} <#{email}>"
    { id: rand(1000), name: name, email: email }
  end

  def delete_user(id)
    puts "Deleting user #{id}"
    true
  end
end

service = UserService.new

# Add logging aspect
Aspect.around(service, :create_user) do |original, args, block|
  puts "[LOG] Before create_user: #{args.inspect}"
  start = Time.now
  result = original.call(*args, &block)
  puts "[LOG] After create_user (#{((Time.now - start) * 1000).round(2)}ms): #{result.inspect}"
  result
end

# Add validation aspect
Aspect.around(service, :create_user) do |original, args, block|
  name, email = args
  raise ArgumentError, "Invalid email" unless email =~ /@/
  original.call(*args, &block)
end

user = service.create_user("Alice", "alice@example.com")
puts user.inspect

begin
  service.create_user("Bob", "invalid-email")
rescue ArgumentError => e
  puts "Error: #{e.message}"
end
```

```ruby
# ข้อที่ 22-25: Mixed module showcase
module Persistable
  def save(filename)
    require 'yaml'
    data = to_h
    File.write(filename, YAML.dump(data))
    puts "Saved to #{filename}"
    self
  end

  def self.load(klass, filename)
    require 'yaml'
    return nil unless File.exist?(filename)
    data = YAML.safe_load(File.read(filename), symbolize_names: true)
    obj = klass.new
    data.each { |k, v| obj.send("#{k}=", v) if obj.respond_to?("#{k}=") }
    obj
  end
end

module Diffable
  def diff(other)
    my_hash = to_h
    their_hash = other.to_h
    changes = {}
    (my_hash.keys | their_hash.keys).each do |key|
      if my_hash[key] != their_hash[key]
        changes[key] = { was: their_hash[key], now: my_hash[key] }
      end
    end
    changes
  end
end

class AppSettings
  include Persistable
  include Diffable

  attr_accessor :theme, :language, :font_size, :auto_save, :notifications

  def initialize
    @theme = "dark"
    @language = "en"
    @font_size = 14
    @auto_save = true
    @notifications = true
  end

  def to_h
    {
      theme: @theme, language: @language, font_size: @font_size,
      auto_save: @auto_save, notifications: @notifications
    }
  end
end

settings = AppSettings.new
old_settings = AppSettings.new

settings.theme = "light"
settings.font_size = 16
settings.language = "th"

puts "Changes:"
settings.diff(old_settings).each do |attr, change|
  puts "  #{attr}: #{change[:was]} → #{change[:now]}"
end
```

```ruby
# ข้อที่ 23: Type coercion module
module Coercible
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def coerce_attribute(name, type)
      var = "@#{name}"
      original_setter = instance_method("#{name}=") rescue nil

      define_method("#{name}=") do |value|
        coerced = coerce_value(value, type)
        instance_variable_set(var, coerced)
      end
    end
  end

  def coerce_value(value, type)
    return nil if value.nil?
    case type
    when :integer then Integer(value)
    when :float then Float(value)
    when :string then String(value)
    when :boolean
      case value
      when true, "true", "1", 1, :true then true
      when false, "false", "0", 0, :false then false
      else !!value
      end
    when :date
      value.is_a?(Date) ? value : Date.parse(value.to_s)
    else value
    end
  rescue ArgumentError, TypeError
    raise TypeError, "Cannot coerce #{value.inspect} to #{type}"
  end
end

class FormData
  include Coercible

  attr_accessor :name, :age, :price, :active, :birth_date

  coerce_attribute :age, :integer
  coerce_attribute :price, :float
  coerce_attribute :active, :boolean

  def initialize(params = {})
    params.each { |k, v| send("#{k}=", v) if respond_to?("#{k}=") }
  end

  def to_h
    { name: @name, age: @age, price: @price, active: @active }
  end
end

form = FormData.new(
  name: "Product A",
  age: "25",        # string → integer
  price: "299.99",  # string → float
  active: "true"    # string → boolean
)

puts form.to_h.inspect
puts form.age.class    # => Integer
puts form.price.class  # => Float
puts form.active.class # => TrueClass
```

```ruby
# ข้อที่ 24: DSL builder module
module DSLBuilder
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def build(&block)
      instance = new
      instance.instance_eval(&block)
      instance
    end
  end
end

class HtmlBuilder
  include DSLBuilder

  def initialize
    @elements = []
    @indent = 0
  end

  def method_missing(tag, content = nil, **attrs, &block)
    attr_str = attrs.map { |k, v| " #{k}=\"#{v}\"" }.join
    if block
      @elements << "#{' ' * @indent}<#{tag}#{attr_str}>"
      @indent += 2
      instance_eval(&block)
      @indent -= 2
      @elements << "#{' ' * @indent}</#{tag}>"
    else
      @elements << "#{' ' * @indent}<#{tag}#{attr_str}>#{content}</#{tag}>"
    end
    self
  end

  def to_s
    @elements.join("\n")
  end
end

html = HtmlBuilder.build do
  html do
    head { title "My Page" }
    body do
      h1 "Hello World"
      p "Welcome to Ruby!", class: "intro"
      ul do
        li "Item 1"
        li "Item 2"
        li "Item 3"
      end
    end
  end
end

puts html
```

```ruby
# ข้อที่ 25: Final comprehensive module example
module FullFeatured
  module ClassMethods
    def create(**attrs)
      obj = new
      attrs.each { |k, v| obj.send("#{k}=", v) if obj.respond_to?("#{k}=") }
      obj.after_initialize if obj.respond_to?(:after_initialize, true)
      obj
    end
  end

  def self.included(base)
    base.extend(ClassMethods)
  end

  def attributes
    instance_variables.each_with_object({}) do |var, hash|
      hash[var.to_s.delete('@').to_sym] = instance_variable_get(var)
    end
  end

  def update(**attrs)
    attrs.each { |k, v| send("#{k}=", v) if respond_to?("#{k}=") }
    self
  end

  def to_s
    pairs = attributes.map { |k, v| "#{k}: #{v.inspect}" }.join(", ")
    "#<#{self.class.name} #{pairs}>"
  end
end

class Task
  include FullFeatured

  attr_accessor :title, :description, :priority, :status, :due_date

  def initialize
    @status = :pending
    @priority = :medium
    @created_at = Time.now
  end

  def complete!
    @status = :completed
    @completed_at = Time.now
    self
  end

  def overdue?
    @due_date && Time.now > @due_date && @status != :completed
  end
end

t1 = Task.create(title: "Write tests", priority: :high, description: "Add unit tests")
t2 = Task.create(title: "Deploy app", priority: :medium)
t3 = Task.create(title: "Update docs", priority: :low)

puts t1
t1.complete!
puts "Completed: #{t1.status}"

t1.update(description: "Added 100% test coverage", priority: :low)
puts t1

tasks = [t1, t2, t3]
pending = tasks.select { |t| t.status == :pending }
puts "\nPending tasks: #{pending.map(&:title).inspect}"
```

---

## สรุปตอนที่ 13

ในตอนนี้เราได้เรียนรู้:

1. **Module คืออะไร** - ต่างจาก class ตรงไหน, ไม่สามารถ instantiate ได้
2. **Namespacing** - จัดกลุ่มโค้ด, หลีกเลี่ยง name collision
3. **include** - เพิ่มเป็น instance methods, method lookup
4. **extend** - เพิ่มเป็น class methods, singleton pattern
5. **prepend** - ใส่ก่อน class ใน lookup chain, method decoration
6. **module_function** - ใช้ได้ทั้ง module และ instance
7. **Comparable** - implement `<=>` แล้วได้ operators ทั้งหมด
8. **Enumerable** - implement `each` แล้วได้ collection methods
9. **Forwardable** - delegate methods ไปยัง object อื่น
10. **Composition** - ใช้ modules แทน deep inheritance
11. **Module Hooks** - `self.included`, `self.extended`, `self.prepended`
12. **Greetable, Serializable, Validatable** - ตัวอย่าง mixins จริง
13. **Design Patterns** - Observer, Cache, Retry, RBAC, State Machine

---

*ตอนต่อไป: Part 14 - Error Handling*
