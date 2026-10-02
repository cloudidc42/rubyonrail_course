# ตอนที่ 9: Methods (ขั้นตอนที่ 146-170)

## บทนำ

Methods เป็นองค์ประกอบพื้นฐานในการเขียน Ruby ทุก ๆ operation ใน Ruby เป็น method call Methods ช่วยให้โค้ด reusable, readable และ maintainable

ในบทนี้เราจะเรียนรู้:
- การ define และ call methods
- Parameters และ arguments ทุกรูปแบบ
- Default parameters
- Keyword arguments
- Splat operators
- Return values
- Method visibility
- Bang methods และ Predicate methods
- Method objects
- Recursive methods
- Method chaining
- Memoization

---

## ขั้นตอนที่ 146: การ Define และ Call Methods พื้นฐาน

```ruby
# การ define method ด้วย def/end
def greet
  puts "สวัสดี!"
end

# การ call method
greet       # => สวัสดี!
greet()     # => สวัสดี! (เหมือนกัน, parentheses optional)

# method ที่รับ arguments
def greet_person(name)
  puts "สวัสดี #{name}!"
end

greet_person("Alice")   # => สวัสดี Alice!
greet_person "Bob"      # => สวัสดี Bob! (ไม่ต้อง parentheses)

# method ที่คืนค่า
def add(a, b)
  a + b   # implicit return
end

result = add(3, 4)
puts result   # => 7

# method ที่มีหลาย parameters
def create_user(name, age, email)
  "User: #{name}, อายุ: #{age}, อีเมล: #{email}"
end

puts create_user("Alice", 25, "alice@example.com")

# naming convention: snake_case
def calculate_total_price
  # ...
end

def get_user_by_id
  # ...
end
```

---

## ขั้นตอนที่ 147: Return Values

```ruby
# Implicit return - Ruby คืนค่าบรรทัดสุดท้าย
def multiply(a, b)
  a * b   # คืนค่านี้โดยอัตโนมัติ
end

puts multiply(4, 5)   # => 20

# Explicit return
def divide(a, b)
  return "หารด้วยศูนย์ไม่ได้" if b == 0
  a.to_f / b
end

puts divide(10, 3)    # => 3.3333...
puts divide(10, 0)    # => หารด้วยศูนย์ไม่ได้

# Return หลาย values (tuple as Array)
def min_max(array)
  [array.min, array.max]
end

min, max = min_max([3, 1, 4, 1, 5, 9])
puts "Min: #{min}, Max: #{max}"   # => Min: 1, Max: 9

# คืนค่า Hash
def statistics(numbers)
  {
    count: numbers.size,
    sum: numbers.sum,
    average: numbers.sum.to_f / numbers.size,
    min: numbers.min,
    max: numbers.max
  }
end

stats = statistics([1, 2, 3, 4, 5])
puts stats[:average]   # => 3.0

# Method ที่ไม่มี return value ชัดเจน คืน nil
def side_effect_only
  puts "กำลังทำอะไรบางอย่าง"
  # ไม่มี return statement
end

result = side_effect_only
puts result.inspect   # => nil
```

---

## ขั้นตอนที่ 148: Default Parameters

```ruby
# Default parameters
def greet(name = "Anonymous", greeting = "สวัสดี")
  "#{greeting} #{name}!"
end

puts greet                      # => สวัสดี Anonymous!
puts greet("Alice")             # => สวัสดี Alice!
puts greet("Bob", "Hello")      # => Hello Bob!

# Default values สามารถ reference parameter ก่อนหน้า
def create_rectangle(width, height = width)
  { width: width, height: height, area: width * height }
end

puts create_rectangle(5).inspect         # => {:width=>5, :height=>5, :area=>25}
puts create_rectangle(4, 6).inspect     # => {:width=>4, :height=>6, :area=>24}

# Default values สามารถเป็น expression
def log(message, level = :info, timestamp = Time.now)
  "[#{level.upcase}] #{timestamp}: #{message}"
end

# Default ที่ depend บน environment
def connect(host = ENV["DB_HOST"] || "localhost", port = 5432)
  "Connecting to #{host}:#{port}"
end

# ระวัง: Default ที่เป็น mutable object
# ไม่ดี!
def add_item(item, list = [])
  list << item   # ปัญหา: list เดียวกันทุกครั้ง!
  list
end

# ดีกว่า
def add_item_safe(item, list = nil)
  list = list ? list.dup : []
  list << item
  list
end
```

---

## ขั้นตอนที่ 149: Keyword Arguments

```ruby
# Keyword arguments ทำให้ method call อ่านง่ายขึ้น
def create_user(name:, age:, email:)
  { name: name, age: age, email: email }
end

# ต้องระบุ keyword
create_user(name: "Alice", age: 25, email: "alice@example.com")

# ลำดับไม่สำคัญ
create_user(email: "bob@example.com", name: "Bob", age: 30)

# Keyword arguments กับ default values
def setup_server(host: "localhost", port: 3000, ssl: false)
  puts "Server: #{host}:#{port} (SSL: #{ssl})"
end

setup_server                          # => Server: localhost:3000 (SSL: false)
setup_server(port: 8080)             # => Server: localhost:8080 (SSL: false)
setup_server(host: "0.0.0.0", ssl: true)  # => Server: 0.0.0.0:3000 (SSL: true)

# Mix ของ positional และ keyword arguments
def process(data, verbose: false, timeout: 30)
  puts "Processing #{data.class} (timeout: #{timeout}, verbose: #{verbose})"
end

process([1, 2, 3])
process("text", verbose: true)
process({key: "val"}, timeout: 60, verbose: true)

# Required keyword arguments (Ruby 2.1+)
def connect(host:, port:, database:)
  "#{host}:#{port}/#{database}"
end

# connect(host: "localhost")  # ArgumentError: missing keyword: port, database

# **kwargs - รับ keyword arguments ที่ไม่รู้ชื่อล่วงหน้า
def flexible(**options)
  puts options.inspect
end

flexible(color: "red", size: :large, weight: 50)
# => {:color=>"red", :size=>:large, :weight=>50}
```

---

## ขั้นตอนที่ 150: Splat Operator (*args)

```ruby
# *args - รับ arguments จำนวนเท่าไรก็ได้
def sum(*numbers)
  numbers.reduce(0, :+)
end

puts sum(1, 2, 3)            # => 6
puts sum(1, 2, 3, 4, 5)     # => 15
puts sum                      # => 0

# *args ใน middle
def wrap(first, *middle, last)
  "[#{first}] #{middle.join(', ')} [#{last}]"
end

puts wrap("start", "a", "b", "c", "end")
# => [start] a, b, c [end]

# Splat ใน method call
def add(a, b, c)
  a + b + c
end

nums = [1, 2, 3]
puts add(*nums)   # => 6 (splat ใน call)

# Splat กับ Array operations
first, *rest = [1, 2, 3, 4, 5]
puts first         # => 1
puts rest.inspect  # => [2, 3, 4, 5]

*head, last = [1, 2, 3, 4, 5]
puts head.inspect  # => [1, 2, 3, 4]
puts last          # => 5

a, *b, c = [1, 2, 3, 4, 5]
puts a             # => 1
puts b.inspect     # => [2, 3, 4]
puts c             # => 5

# Double splat (**) สำหรับ Hash
def merge_options(base, **overrides)
  base.merge(overrides)
end

defaults = { color: "blue", size: :medium }
result = merge_options(defaults, color: "red", weight: 100)
puts result.inspect
# => {:color=>"red", :size=>:medium, :weight=>100}
```

---

## ขั้นตอนที่ 151: ทุก Parameter ในคราวเดียว

```ruby
# Ruby 2.7+: รองรับทุก argument types
def complex_method(required, optional = "default", *args, keyword:, kw_default: "kw_def", **kwargs, &block)
  puts "required: #{required}"
  puts "optional: #{optional}"
  puts "args: #{args.inspect}"
  puts "keyword: #{keyword}"
  puts "kw_default: #{kw_default}"
  puts "kwargs: #{kwargs.inspect}"
  puts "block: #{block.call}" if block
end

complex_method(
  "req",          # required
  "opt",          # optional
  1, 2, 3,       # *args
  keyword: "kw", # keyword:
  extra: true,   # **kwargs
  &-> { "block result" }  # &block
)

# ลำดับ parameters:
# 1. required positional
# 2. optional positional (with defaults)
# 3. splat (*args)
# 4. required keyword
# 5. optional keyword (with defaults)
# 6. double splat (**kwargs)
# 7. block (&block)

# Ruby 3.0: Separated positional and keyword arguments
# ใน Ruby 2 บางกรณี hash สุดท้ายถูก treat เป็น keyword args
# ใน Ruby 3 ต้องแยกชัดเจน
```

---

## ขั้นตอนที่ 152: Method Aliases

```ruby
class Calculator
  def add(a, b)
    a + b
  end

  # สร้าง alias
  alias_method :plus, :add
  alias :sum, :add
end

calc = Calculator.new
puts calc.add(3, 4)    # => 7
puts calc.plus(3, 4)   # => 7
puts calc.sum(3, 4)    # => 7

# alias ที่ built-in Ruby methods
class Array
  alias_method :size_alias, :size
end

arr = [1, 2, 3]
puts arr.size         # => 3
puts arr.size_alias   # => 3

# alias ใน Module
module Greetable
  def greet(name)
    "สวัสดี #{name}"
  end

  alias_method :hello, :greet
  alias_method :hi, :greet
end

class Person
  include Greetable

  def initialize(name)
    @name = name
  end

  def introduce
    greet(@name)
  end
end

person = Person.new("Alice")
puts person.greet("Bob")    # => สวัสดี Bob
puts person.hello("Bob")    # => สวัสดี Bob
puts person.hi("Bob")       # => สวัสดี Bob

# Method aliasing pattern: save original, override, call original
class Logger
  def log(message)
    puts "[LOG] #{message}"
  end
end

class TimestampLogger < Logger
  alias_method :original_log, :log

  def log(message)
    original_log("[#{Time.now}] #{message}")
  end
end
```

---

## ขั้นตอนที่ 153: Method Visibility - public, private, protected

```ruby
class BankAccount
  def initialize(balance)
    @balance = balance
    @owner = "Default Owner"
  end

  # Public methods - accessible จากทุกที่
  def deposit(amount)
    validate_amount(amount)
    @balance += amount
    "เพิ่ม #{amount} บาท"
  end

  def withdraw(amount)
    validate_amount(amount)
    check_sufficient_funds(amount)
    @balance -= amount
    "ถอน #{amount} บาท"
  end

  def balance
    @balance
  end

  # Protected methods - accessible จาก instance ของ class เดียวกัน
  protected

  def transfer_to(other_account, amount)
    @balance -= amount
    other_account.receive(amount)
  end

  def receive(amount)
    @balance += amount
  end

  # Private methods - accessible เฉพาะใน instance เดียวกัน
  private

  def validate_amount(amount)
    raise ArgumentError, "จำนวนต้องเป็นบวก" unless amount > 0
  end

  def check_sufficient_funds(amount)
    raise "เงินไม่พอ" if amount > @balance
  end
end

account = BankAccount.new(1000)
puts account.deposit(500)     # => เพิ่ม 500 บาท
puts account.balance          # => 1500
# account.validate_amount(100)  # NoMethodError: private method

# Protected vs Private
# Private: ไม่สามารถ call ด้วย explicit receiver (แม้แต่ self)
# Protected: สามารถ call ด้วย instance อื่นของ class เดียวกัน
```

---

## ขั้นตอนที่ 154: Private Methods แบบละเอียด

```ruby
class User
  def initialize(name, age)
    @name = name
    @age = age
  end

  def greeting
    "สวัสดี! ฉันชื่อ #{@name}, #{format_age}"
  end

  def adult?
    @age >= 18
  end

  # Inline private (Ruby 2.7+)
  private def format_age
    "อายุ #{@age} ปี"
  end

  private

  def internal_method
    "ใช้ภายในเท่านั้น"
  end
end

user = User.new("Alice", 25)
puts user.greeting    # => สวัสดี! ฉันชื่อ Alice, อายุ 25 ปี
puts user.adult?      # => true
# user.format_age     # => NoMethodError

# Private method ใน Ruby 2.7+
# สามารถ call ด้วย self. ได้แล้ว (ใน instance เดิม)
class Calculator
  def compute(x)
    double(x) + triple(x)  # เรียก private methods
  end

  private

  def double(n)
    n * 2
  end

  def triple(n)
    n * 3
  end
end

puts Calculator.new.compute(5)   # => 25

# Method ที่ private โดย default ใน Ruby
# initialize
# initialize_copy

class Document
  def initialize(title, content)
    @title = title
    @content = content
  end
  
  # initialize เป็น private โดยอัตโนมัติ
  # ไม่สามารถ call ได้โดยตรง
end
```

---

## ขั้นตอนที่ 155: Bang Methods (!) และ Predicate Methods (?)

```ruby
# Bang methods (!) - เป็น convention ไม่ใช่ rule
# มักหมายถึง: "อันตราย" หรือ "แก้ไข in-place"

# ตัวอย่าง pairs ของ ! methods
arr = [3, 1, 4, 1, 5, 9]

sorted = arr.sort      # คืนค่า Array ใหม่
arr.sort!              # แก้ไข arr in-place
puts arr.inspect       # => [1, 1, 3, 4, 5, 9]

str = "hello"
upcased = str.upcase   # คืน String ใหม่
str.upcase!            # แก้ไข str in-place
puts str               # => HELLO

# ! methods ใน Ruby standard library
[1, nil, 2, nil, 3].compact   # คืนค่าใหม่
[1, nil, 2, nil, 3].compact!  # in-place

# สร้าง bang method เอง
class TextProcessor
  def initialize(text)
    @text = text
  end

  def normalize
    # คืน instance ใหม่ หรือ string ใหม่
    TextProcessor.new(@text.strip.downcase)
  end

  def normalize!
    # แก้ไข in-place
    @text = @text.strip.downcase
    self  # คืน self สำหรับ chaining
  end

  def to_s
    @text
  end
end

t = TextProcessor.new("  HELLO WORLD  ")
puts t.normalize.to_s    # => hello world (t ไม่เปลี่ยน)
puts t.to_s              # => "  HELLO WORLD  "
t.normalize!
puts t.to_s              # => "hello world"

# Predicate methods (?) - คืนค่า boolean
puts [].empty?            # => true
puts "hello".include?("ell")  # => true
puts 5.between?(1, 10)   # => true
puts nil.nil?            # => true
puts 3.odd?              # => true
puts 4.even?             # => true
puts "abc".start_with?("ab")  # => true
puts "abc".end_with?("bc")    # => true

# สร้าง predicate method เอง
class User
  def initialize(role, active)
    @role = role
    @active = active
  end

  def admin?
    @role == :admin
  end

  def active?
    @active
  end

  def can_access?(resource)
    active? && (admin? || resource.public?)
  end
end
```

---

## ขั้นตอนที่ 156: Method Objects

```ruby
# Method เป็น object ใน Ruby
def greet(name)
  "สวัสดี #{name}!"
end

m = method(:greet)
puts m.class           # => Method
puts m.call("Alice")   # => สวัสดี Alice!
puts m.("Bob")         # => สวัสดี Bob! (syntactic sugar)

# ส่ง method เป็น argument
names = ["Alice", "Bob", "Carol"]
puts names.map(&method(:greet)).inspect
# => ["สวัสดี Alice!", "สวัสดี Bob!", "สวัสดี Carol!"]

# Method object จาก instance
class Calculator
  def double(n)
    n * 2
  end
end

calc = Calculator.new
double_method = calc.method(:double)
puts [1, 2, 3].map(&double_method).inspect   # => [2, 4, 6]

# UnboundMethod - method ที่ไม่ผูกกับ instance
unbound = Calculator.instance_method(:double)
puts unbound.class   # => UnboundMethod

bound = unbound.bind(Calculator.new)
puts bound.call(5)   # => 10

# Method#to_proc
m = "hello".method(:upcase)
puts m.to_proc.call   # => HELLO

# Proc ของ built-in methods
puts [1, -2, 3, -4].select(&method(:positive?))   # ไม่ทำงาน
puts [1, -2, 3, -4].map(&:abs).inspect   # => [1, 2, 3, 4]
```

---

## ขั้นตอนที่ 157: Recursive Methods

```ruby
# Recursion - method เรียกตัวเอง
def factorial(n)
  return 1 if n <= 1
  n * factorial(n - 1)
end

puts factorial(5)   # => 120
puts factorial(10)  # => 3628800

# Fibonacci recursive
def fibonacci(n)
  return n if n <= 1
  fibonacci(n - 1) + fibonacci(n - 2)
end

puts fibonacci(10)  # => 55
# หมายเหตุ: การ implement นี้ช้ามาก O(2^n)

# Fibonacci ด้วย memoization
def fibonacci_memo(n, memo = {})
  return n if n <= 1
  memo[n] ||= fibonacci_memo(n - 1, memo) + fibonacci_memo(n - 2, memo)
end

puts fibonacci_memo(50)  # => 12586269025

# Binary Search recursive
def binary_search(arr, target, low = 0, high = arr.length - 1)
  return -1 if low > high
  mid = (low + high) / 2

  case arr[mid] <=> target
  when 0 then mid
  when -1 then binary_search(arr, target, mid + 1, high)
  when 1 then binary_search(arr, target, low, mid - 1)
  end
end

sorted = [1, 3, 5, 7, 9, 11, 13, 15]
puts binary_search(sorted, 7)    # => 3
puts binary_search(sorted, 6)    # => -1

# Tower of Hanoi
def hanoi(n, from = "A", to = "C", via = "B")
  if n == 1
    puts "ย้ายจาก #{from} ไป #{to}"
    return
  end
  hanoi(n - 1, from, via, to)
  puts "ย้ายจาก #{from} ไป #{to}"
  hanoi(n - 1, via, to, from)
end

hanoi(3)

# Flatten recursive
def my_flatten(arr)
  arr.each_with_object([]) do |item, result|
    if item.is_a?(Array)
      result.concat(my_flatten(item))
    else
      result << item
    end
  end
end

puts my_flatten([1, [2, [3, [4]]]]).inspect   # => [1, 2, 3, 4]
```

---

## ขั้นตอนที่ 158: Method Chaining

```ruby
# Method chaining - เรียก methods ต่อ ๆ กัน
class QueryBuilder
  def initialize
    @conditions = []
    @order = nil
    @limit_val = nil
    @table = nil
  end

  def from(table)
    @table = table
    self  # คืน self เพื่อ chaining
  end

  def where(condition)
    @conditions << condition
    self
  end

  def order(field, direction = :asc)
    @order = "#{field} #{direction.to_s.upcase}"
    self
  end

  def limit(n)
    @limit_val = n
    self
  end

  def build
    sql = "SELECT * FROM #{@table}"
    sql += " WHERE #{@conditions.join(' AND ')}" unless @conditions.empty?
    sql += " ORDER BY #{@order}" if @order
    sql += " LIMIT #{@limit_val}" if @limit_val
    sql
  end
end

query = QueryBuilder.new
  .from("users")
  .where("age > 18")
  .where("active = true")
  .order(:name)
  .limit(10)
  .build

puts query
# => SELECT * FROM users WHERE age > 18 AND active = true ORDER BY name ASC LIMIT 10

# String method chaining
result = "  hello world  "
  .strip
  .split
  .map(&:capitalize)
  .join(" ")
puts result   # => Hello World

# Array method chaining
top_earners = [
  { name: "Alice", salary: 80000 },
  { name: "Bob", salary: 60000 },
  { name: "Carol", salary: 90000 },
  { name: "Dave", salary: 55000 }
]
  .select { |e| e[:salary] > 60000 }
  .sort_by { |e| -e[:salary] }
  .map { |e| e[:name] }

puts top_earners.inspect   # => ["Carol", "Alice"]
```

---

## ขั้นตอนที่ 159: Memoization

```ruby
# Memoization - cache ผลลัพธ์เพื่อไม่ต้องคำนวณซ้ำ

# วิธีที่ 1: ||=
class Calculator
  def fibonacci(n)
    @memo ||= {}
    @memo[n] ||= if n <= 1
      n
    else
      fibonacci(n - 1) + fibonacci(n - 2)
    end
  end
end

calc = Calculator.new
puts calc.fibonacci(50)   # => คำนวณเร็วมาก!

# วิธีที่ 2: Instance variable
class DataFetcher
  def user_count
    @user_count ||= expensive_db_query
  end

  def product_list
    @product_list ||= fetch_from_api
  end

  private

  def expensive_db_query
    sleep(1)  # simulate slow query
    1000
  end

  def fetch_from_api
    sleep(2)  # simulate API call
    ["product1", "product2"]
  end
end

# วิธีที่ 3: Hash-based memoization
module Memoizable
  def memoize(method_name)
    original = instance_method(method_name)
    cache_var = "@_memo_#{method_name}"

    define_method(method_name) do |*args|
      cache = instance_variable_get(cache_var) || instance_variable_set(cache_var, {})
      cache[args] ||= original.bind(self).call(*args)
    end
  end
end

class ExpensiveCalculator
  extend Memoizable

  def complex_calculation(n)
    puts "กำลังคำนวณ #{n}..."
    n ** 3
  end

  memoize :complex_calculation
end

calc = ExpensiveCalculator.new
puts calc.complex_calculation(5)   # => กำลังคำนวณ 5... => 125
puts calc.complex_calculation(5)   # => 125 (ไม่คำนวณซ้ำ)
puts calc.complex_calculation(3)   # => กำลังคำนวณ 3... => 27
```

---

## ขั้นตอนที่ 160: Method Missing และ respond_to_missing?

```ruby
# method_missing - จัดการ method ที่ไม่มี
class DynamicProxy
  def initialize(target)
    @target = target
  end

  def method_missing(name, *args, &block)
    if @target.respond_to?(name)
      puts "Calling #{name} on target"
      @target.send(name, *args, &block)
    else
      super  # ส่งต่อไปยัง default behavior
    end
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private) || super
  end
end

proxy = DynamicProxy.new([1, 2, 3])
puts proxy.length       # => Calling length on target => 3
puts proxy.map { |x| x * 2 }.inspect   # => [2, 4, 6]

# Dynamic Finder (Rails style)
class Database
  def initialize
    @data = {
      users: [
        { id: 1, name: "Alice", role: "admin" },
        { id: 2, name: "Bob", role: "user" }
      ]
    }
  end

  def method_missing(name, *args)
    if name.to_s =~ /^find_(\w+)_by_(\w+)$/
      table = $1.to_sym
      field = $2.to_sym
      value = args.first
      @data[table]&.find { |r| r[field] == value }
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    name.to_s =~ /^find_\w+_by_\w+$/ || super
  end
end

db = Database.new
puts db.find_users_by_name("Alice").inspect
# => {:id=>1, :name=>"Alice", :role=>"admin"}
puts db.find_users_by_role("user").inspect
# => {:id=>2, :name=>"Bob", :role=>"user"}
```

---

## ขั้นตอนที่ 161: define_method

```ruby
# define_method - สร้าง method โดยใช้ string/symbol
class MyClass
  # สร้าง methods dynamically
  %w[red green blue].each do |color|
    define_method("#{color}_text") do |text|
      "\e[#{case color when 'red' then 31 when 'green' then 32 else 34 end}m#{text}\e[0m"
    end
  end
end

obj = MyClass.new
# puts obj.red_text("Error!")   # แสดงเป็นสีแดง
# puts obj.green_text("OK!")    # แสดงเป็นสีเขียว

# สร้าง getters/setters dynamically
class Config
  SETTINGS = %i[host port database username]

  SETTINGS.each do |setting|
    define_method(setting) { instance_variable_get("@#{setting}") }
    define_method("#{setting}=") { |val| instance_variable_set("@#{setting}", val) }
  end

  def initialize(opts = {})
    opts.each { |k, v| send("#{k}=", v) if SETTINGS.include?(k) }
  end
end

config = Config.new(host: "localhost", port: 5432, database: "myapp")
puts config.host      # => localhost
puts config.port      # => 5432
config.database = "newdb"
puts config.database  # => newdb

# define_method กับ closure
def create_multiplier(factor)
  define_method("multiply_by_#{factor}") do |n|
    n * factor
  end
end

class Calculator
end

calc = Calculator.new
# create_multiplier บน Calculator...
# หรือใช้ lambda
multiplier = ->(factor) { ->(n) { n * factor } }
double = multiplier.(2)
puts double.(5)   # => 10
```

---

## ขั้นตอนที่ 162: Methods with Blocks (yield และ block_given?)

```ruby
# Accepting blocks
def repeat(n)
  n.times { yield }
end

repeat(3) { puts "Hello!" }
# => Hello! x 3

# block_given? - ตรวจสอบว่ามี block ส่งมา
def greet(name)
  if block_given?
    yield name
  else
    "สวัสดี #{name}"
  end
end

puts greet("Alice")                     # => สวัสดี Alice
puts greet("Bob") { |n| "Hi #{n}!" }   # => Hi Bob!

# yield ด้วยข้อมูล
def transform(data)
  if block_given?
    yield data
  else
    data
  end
end

puts transform([1, 2, 3]) { |a| a.map { |n| n * 2 } }.inspect
# => [2, 4, 6]

# &block - explicit block parameter
def log_time(&block)
  start = Time.now
  result = block.call
  elapsed = Time.now - start
  puts "ใช้เวลา: #{elapsed.round(4)} วินาที"
  result
end

result = log_time { (1..1000).to_a.map { |n| n ** 2 }.sum }
puts result

# ส่ง block ต่อ
def wrapper(&block)
  puts "ก่อน"
  inner(&block)
  puts "หลัง"
end

def inner
  puts "ใน inner"
  yield
end

wrapper { puts "Block content" }
```

---

## ขั้นตอนที่ 163: Method Access Control Detail

```ruby
# attr_accessor, attr_reader, attr_writer
class Person
  attr_accessor :name    # getter + setter
  attr_reader :age       # getter only
  attr_writer :email     # setter only

  def initialize(name, age, email)
    @name = name
    @age = age
    @email = email
  end

  def info
    "#{@name}, #{@age} years"
  end

  private :info  # ทำ method ที่มีอยู่ให้ private
end

p = Person.new("Alice", 25, "alice@example.com")
puts p.name        # => Alice (getter)
p.name = "Bob"     # (setter)
puts p.age         # => 25 (getter)
# p.age = 30      # NoMethodError
p.email = "new@example.com"  # (setter)
# p.email          # NoMethodError (no getter)

# module_function
module MathHelper
  module_function  # เป็นทั้ง instance method และ module method

  def square(n)
    n ** 2
  end

  def cube(n)
    n ** 3
  end
end

puts MathHelper.square(4)   # => 16 (as module method)
puts MathHelper.cube(3)     # => 27

include MathHelper
puts square(5)   # => 25 (as instance method)

# protected - สำหรับ comparison
class Temperature
  include Comparable

  def initialize(degrees)
    @degrees = degrees
  end

  def <=>(other)
    degrees <=> other.degrees
  end

  protected

  def degrees
    @degrees
  end
end
```

---

## ขั้นตอนที่ 164: Proc การส่ง methods เป็น Arguments

```ruby
# Symbol#to_proc - แปลง symbol เป็น proc
# :method_name.to_proc เทียบกับ { |x| x.method_name }

names = ["alice", "bob", "carol"]

# ยาว
upcase_names = names.map { |n| n.upcase }

# สั้นกว่า
upcase_names = names.map(&:upcase)

# works with any method
puts [1, 2, 3].map(&:to_s).inspect     # => ["1", "2", "3"]
puts [-1, 2, -3].map(&:abs).inspect    # => [1, 2, 3]
puts [1, 2, 3].select(&:odd?).inspect  # => [1, 3]

# method(:name) - สำหรับ methods ที่ต้อง argument
def double(n)
  n * 2
end

puts [1, 2, 3].map(&method(:double)).inspect   # => [2, 4, 6]

# Chaining กับ &:method
result = ["  hello  ", "  world  ", "  ruby  "]
  .map(&:strip)
  .map(&:capitalize)
  .sort
puts result.inspect   # => ["Hello", "Ruby", "World"]

# Higher-order methods
def apply_twice(func, value)
  func.call(func.call(value))
end

double_func = method(:double)
puts apply_twice(double_func, 3)   # => 12

# Passing method references
sorters = {
  by_name: ->(a, b) { a[:name] <=> b[:name] },
  by_age: ->(a, b) { a[:age] <=> b[:age] },
  by_salary: ->(a, b) { b[:salary] <=> a[:salary] }  # descending
}

people = [
  { name: "Charlie", age: 35, salary: 70000 },
  { name: "Alice", age: 25, salary: 80000 },
  { name: "Bob", age: 30, salary: 60000 }
]

people.sort(&sorters[:by_name]).each { |p| puts p[:name] }
```

---

## ขั้นตอนที่ 165: Callable Objects

```ruby
# Objects ที่ callable ได้ต้องมี #call method
# Proc, Lambda, Method ล้วนเป็น callable

# Proc
double_proc = Proc.new { |n| n * 2 }
puts double_proc.call(5)   # => 10
puts double_proc.(5)        # => 10
puts double_proc[5]         # => 10

# Lambda
double_lambda = lambda { |n| n * 2 }
# หรือ
double_lambda2 = ->(n) { n * 2 }
puts double_lambda.call(5)   # => 10

# Method
def triple(n)
  n * 3
end

triple_method = method(:triple)
puts triple_method.call(5)   # => 15

# Custom callable class
class Formatter
  def initialize(template)
    @template = template
  end

  def call(data)
    @template % data
  end
end

currency_formatter = Formatter.new("฿%,d")
puts currency_formatter.(50000)   # => ฿50,000

# ใช้ callable objects ใน higher-order programming
def process_numbers(numbers, *operations)
  operations.reduce(numbers) do |nums, op|
    nums.map { |n| op.call(n) }
  end
end

result = process_numbers(
  [1, 2, 3, 4, 5],
  method(:double),   # ไม่มี method double ในตัวอย่างนี้
  ->(n) { n + 1 },
  ->(n) { n * 3 }
)
```

---

## ขั้นตอนที่ 166-170: Advanced Method Concepts

```ruby
# ขั้นตอนที่ 166: Method Introspection
class MyClass
  def public_method; end
  protected
  def protected_method; end
  private
  def private_method; end
end

obj = MyClass.new
puts obj.methods.sort.first(5).inspect
puts obj.public_methods(false).inspect
puts obj.protected_methods(false).inspect
puts obj.private_methods(false).inspect

puts obj.respond_to?(:public_method)    # => true
puts obj.respond_to?(:private_method)  # => false
puts obj.respond_to?(:private_method, true)  # => true (include private)

# ขั้นตอนที่ 167: send
class Greeter
  private

  def secret_greet(name)
    "ลับมาก! สวัสดี #{name}"
  end
end

g = Greeter.new
puts g.send(:secret_greet, "Alice")   # send สามารถ call private methods
# g.public_send(:secret_greet, "Alice")  # NoMethodError

# ขั้นตอนที่ 168: Method Wrapping (Decorator pattern)
module Timing
  def self.included(base)
    base.instance_methods(false).each do |method|
      original = instance_method(method)
      define_method(method) do |*args, &block|
        start = Time.now
        result = original.bind(self).call(*args, &block)
        puts "#{method} ใช้เวลา #{Time.now - start} วินาที"
        result
      end
    end
  end
end

# ขั้นตอนที่ 169: Functional Composition
double = ->(n) { n * 2 }
increment = ->(n) { n + 1 }

# compose กับ >> และ << (Ruby 2.6+)
double_then_increment = double >> increment
increment_then_double = double << increment

puts double_then_increment.(5)   # => 11 (5*2+1)
puts increment_then_double.(5)   # => 12 (5+1)*2

# ขั้นตอนที่ 170: Curry
add = ->(a, b) { a + b }
add5 = add.curry.(5)   # partial application

puts add5.(3)    # => 8
puts add5.(10)   # => 15

multiply = ->(a, b) { a * b }
triple = multiply.curry.(3)

puts [1, 2, 3, 4, 5].map(&triple).inspect
# => [3, 6, 9, 12, 15]
```

---

## แบบฝึกหัด (ขั้นตอนที่ 146-170)

### ข้อที่ 1: Method ที่รองรับ Multiple Input Formats

```ruby
# เฉลย
def parse_date(input)
  case input
  when String
    parts = input.split(/[-\/]/).map(&:to_i)
    Date.new(*parts) rescue "Invalid date format"
  when Array
    Date.new(*input) rescue "Invalid date array"
  when Hash
    Date.new(input[:year], input[:month], input[:day]) rescue "Invalid date hash"
  else
    raise ArgumentError, "Unsupported input type"
  end
end
```

### ข้อที่ 2: Pipeline สำหรับ Text Processing

```ruby
# เฉลย
class TextPipeline
  def initialize(text)
    @text = text
    @operations = []
  end

  def strip
    @operations << :strip
    self
  end

  def downcase
    @operations << :downcase
    self
  end

  def remove_punctuation
    @operations << ->(t) { t.gsub(/[^\w\s]/, '') }
    self
  end

  def split_words
    @operations << :split
    self
  end

  def process
    @operations.reduce(@text) do |text, op|
      case op
      when Symbol then text.send(op)
      when Proc then op.call(text)
      end
    end
  end
end

result = TextPipeline.new("  Hello, World! This is Ruby.  ")
  .strip
  .downcase
  .remove_punctuation
  .split_words
  .process

puts result.inspect
# => ["hello", "world", "this", "is", "ruby"]
```

### ข้อที่ 3: Retry Logic

```ruby
# เฉลย
def with_retry(max_attempts: 3, wait: 1, exceptions: [StandardError])
  attempts = 0
  begin
    attempts += 1
    yield attempts
  rescue *exceptions => e
    puts "Attempt #{attempts} failed: #{e.message}"
    if attempts < max_attempts
      sleep(wait)
      retry
    else
      raise "Failed after #{max_attempts} attempts: #{e.message}"
    end
  end
end

# ใช้งาน:
result = with_retry(max_attempts: 3) do |attempt|
  raise "Simulated error" if attempt < 3
  "Success on attempt #{attempt}"
end
puts result
```

### ข้อที่ 4: Memoized Fibonacci

```ruby
# เฉลย
class FibonacciCalculator
  def initialize
    @cache = { 0 => 0, 1 => 1 }
  end

  def compute(n)
    @cache[n] ||= compute(n - 1) + compute(n - 2)
  end

  def sequence(n)
    (0..n).map { |i| compute(i) }
  end
end

calc = FibonacciCalculator.new
puts calc.sequence(15).inspect
```

### ข้อที่ 5: Flexible Logger

```ruby
# เฉลย
class Logger
  LEVELS = { debug: 0, info: 1, warn: 2, error: 3, fatal: 4 }

  def initialize(min_level: :info, prefix: nil)
    @min_level = min_level
    @prefix = prefix
    @logs = []
  end

  LEVELS.each do |level, value|
    define_method(level) do |message|
      log(level, message)
    end
  end

  private

  def log(level, message)
    return unless LEVELS[level] >= LEVELS[@min_level]
    entry = {
      level: level,
      message: "#{@prefix ? "[#{@prefix}] " : ''}#{message}",
      time: Time.now
    }
    @logs << entry
    puts "[#{level.upcase}] #{entry[:message]}"
  end
end

logger = Logger.new(min_level: :info, prefix: "APP")
logger.debug("ไม่แสดง")
logger.info("เริ่มระบบ")
logger.warn("ระวัง!")
logger.error("เกิดข้อผิดพลาด")
```

### ข้อที่ 6-25: แบบฝึกหัดเพิ่มเติม

```ruby
# ข้อ 6: Decorator Method
def memoize(func)
  cache = {}
  ->(n) { cache[n] ||= func.call(n) }
end

slow_square = ->(n) { sleep(0.01); n ** 2 }
fast_square = memoize(slow_square)
puts fast_square.(5)   # => 25 (คำนวณ)
puts fast_square.(5)   # => 25 (cache)

# ข้อ 7: Compose Functions
def compose(*funcs)
  funcs.reduce { |f, g| ->(x) { f.(g.(x)) } }
end

pipeline = compose(
  ->(x) { x * 2 },
  ->(x) { x + 1 },
  ->(x) { x ** 2 }
)
puts pipeline.(3)   # => (3^2 + 1) * 2 = 20

# ข้อ 8: Safe Division
def safe_divide(a, b, default: nil)
  return default if b.zero?
  a.to_f / b
rescue => e
  default
end

puts safe_divide(10, 2)          # => 5.0
puts safe_divide(10, 0)          # => nil
puts safe_divide(10, 0, default: 0)  # => 0

# ข้อ 9: Named Parameters Validator
def validated_method(name:, age: nil, email: nil)
  errors = []
  errors << "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร" if name.length < 2
  errors << "อายุต้อง 0-120" if age && !(0..120).include?(age)
  errors << "Email ไม่ถูกต้อง" if email && !email.include?("@")
  
  return yield(name, age, email) if errors.empty? && block_given?
  errors.empty? ? { success: true } : { success: false, errors: errors }
end

result = validated_method(name: "A")
puts result.inspect   # => {:success=>false, :errors=>["ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"]}

# ข้อ 10: Fluent Interface
class EmailBuilder
  def initialize
    @to = []
    @cc = []
    @subject = ""
    @body = ""
  end

  def to(*addresses)
    @to.concat(addresses)
    self
  end

  def cc(*addresses)
    @cc.concat(addresses)
    self
  end

  def subject(text)
    @subject = text
    self
  end

  def body(text)
    @body = text
    self
  end

  def send!
    puts "To: #{@to.join(', ')}"
    puts "CC: #{@cc.join(', ')}" unless @cc.empty?
    puts "Subject: #{@subject}"
    puts "---"
    puts @body
    puts "Email sent!"
    self
  end
end

EmailBuilder.new
  .to("alice@example.com", "bob@example.com")
  .cc("manager@example.com")
  .subject("รายงานประจำเดือน")
  .body("สวัสดีทุกคน นี่คือรายงานประจำเดือน...")
  .send!

# ข้อ 11: Method Rate Limiter
class RateLimiter
  def initialize(calls_per_second)
    @interval = 1.0 / calls_per_second
    @last_call = Time.now - @interval
  end

  def call
    elapsed = Time.now - @last_call
    sleep(@interval - elapsed) if elapsed < @interval
    @last_call = Time.now
    yield
  end
end

# ข้อ 12: Recursive Data Traversal
def traverse(data, &block)
  case data
  when Hash
    data.each_with_object({}) do |(k, v), result|
      result[k] = traverse(v, &block)
    end
  when Array
    data.map { |item| traverse(item, &block) }
  else
    block.call(data)
  end
end

nested = { a: 1, b: [2, 3, { c: 4 }], d: "hello" }
doubled = traverse(nested) { |v| v.is_a?(Integer) ? v * 2 : v }
puts doubled.inspect
# => {:a=>2, :b=>[4, 6, {:c=>8}], :d=>"hello"}

# ข้อ 13-25: (สรุป patterns)

# ข้อ 13: Template Method Pattern
class Report
  def generate
    setup
    header = generate_header
    body = generate_body
    footer = generate_footer
    teardown
    [header, body, footer].join("\n")
  end

  private

  def setup; end
  def teardown; end

  def generate_header
    raise NotImplementedError, "Subclass must implement generate_header"
  end

  def generate_body
    raise NotImplementedError
  end

  def generate_footer
    "--- End of Report ---"
  end
end

class SalesReport < Report
  def generate_header
    "=== Sales Report ==="
  end

  def generate_body
    "Total Sales: ฿100,000"
  end
end

puts SalesReport.new.generate

# ข้อ 14: Optional Chaining
class Config
  def initialize(data)
    @data = data
  end

  def dig_safe(*keys)
    keys.reduce(@data) do |current, key|
      return nil unless current.is_a?(Hash)
      current[key]
    end
  end
end

config = Config.new({ db: { host: "localhost", port: 5432 } })
puts config.dig_safe(:db, :host)       # => localhost
puts config.dig_safe(:db, :name).inspect      # => nil (ไม่ error)
puts config.dig_safe(:cache, :host).inspect   # => nil (ไม่ error)

# ข้อ 15: Dynamic Validators
class Validator
  RULES = {
    required: ->(val, _) { !val.nil? && val.to_s != '' },
    min_length: ->(val, n) { val.to_s.length >= n },
    max_length: ->(val, n) { val.to_s.length <= n },
    min: ->(val, n) { val.to_i >= n },
    max: ->(val, n) { val.to_i <= n },
    format: ->(val, regex) { val.to_s.match?(regex) }
  }

  def validate(value, **rules)
    errors = rules.each_with_object([]) do |(rule, param), errs|
      unless RULES[rule]&.call(value, param)
        errs << "ไม่ผ่าน rule: #{rule}(#{param})"
      end
    end
    errors.empty? ? { valid: true } : { valid: false, errors: errors }
  end
end

v = Validator.new
puts v.validate("hello", required: true, min_length: 3, max_length: 10).inspect
puts v.validate("", required: true).inspect
puts v.validate("user@test.com", format: /\A\S+@\S+\z/).inspect
```

---

## สรุปบทที่ 9

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| การ define | `def name`, implicit return |
| Parameters | required, optional (`= default`), splat (`*args`), keyword (`key:`) |
| Return | implicit, explicit `return` |
| Visibility | `public`, `private`, `protected` |
| Bang (!) | แก้ไข in-place หรือ "อันตราย" |
| Predicate (?) | คืนค่า boolean |
| Method objects | `method(:name)`, `&:symbol` |
| Recursion | self-call พร้อม base case |
| Chaining | return `self` |
| Memoization | `@var ||= expensive_call` |
| method_missing | จัดการ undefined methods |
| define_method | สร้าง methods dynamically |

**Key Takeaways:**
1. Ruby method คืนค่าบรรทัดสุดท้ายเสมอ (implicit return)
2. ใช้ keyword arguments เพื่อทำให้ method calls อ่านง่าย
3. Bang methods (!) เป็น convention ไม่ใช่ rule
4. Predicate methods (?) ควรคืนค่า boolean เสมอ
5. Return `self` จาก methods สำหรับ method chaining
6. `||=` เป็นวิธีที่ง่ายที่สุดสำหรับ memoization

---

*ถัดไป: ตอนที่ 10 - Blocks, Procs, Lambdas (ขั้นตอนที่ 171-200)*
