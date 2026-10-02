# ตอนที่ 9: Methods / Functions (Steps 146-170)

> **เป้าหมาย**: เรียนรู้การสร้างและใช้งาน methods ใน Ruby อย่างครบถ้วน ตั้งแต่พื้นฐานไปจนถึง advanced patterns

---

## Step 146: การ Define Method — def / end

Method คือกลุ่มของโค้ดที่มีชื่อ สามารถเรียกใช้ซ้ำได้ ใน Ruby ใช้ `def` และ `end`

```ruby
# รูปแบบพื้นฐาน
def greet
  puts "สวัสดี!"
end

greet  # Output: สวัสดี!

# Method ที่รับ argument
def greet_person(name)
  puts "สวัสดี, #{name}!"
end

greet_person("Alice")  # Output: สวัสดี, Alice!

# Method ที่คืนค่า
def add(a, b)
  a + b  # ค่าสุดท้ายเป็น return value อัตโนมัติ
end

result = add(3, 4)
puts result  # Output: 7
```

### ชื่อ Method ใน Ruby

```ruby
# ชื่อ method ใช้ lowercase + underscore (snake_case)
def calculate_total_price
def get_user_by_id
def is_valid_email?      # predicate method ลงท้าย ?
def save!                # bang method ลงท้าย !
def convert_to_string    # ชัดเจน
```

### Method ที่ไม่มี argument

```ruby
def current_time
  Time.now.strftime("%H:%M:%S")
end

def ruby_version
  RUBY_VERSION
end

puts current_time   # 14:30:25
puts ruby_version   # 3.2.0
```

---

## Step 147: Calling Methods — การเรียกใช้ Method

```ruby
# วิธีการเรียก method ต่างๆ
def hello
  "Hello, World!"
end

# วิธีที่ 1: เรียกตรงๆ
hello

# วิธีที่ 2: เก็บใน variable
result = hello
puts result

# วิธีที่ 3: ใน string interpolation
puts "ผลลัพธ์: #{hello}"

# เรียก method บน object
"hello".upcase
[1, 2, 3].length
42.to_s

# เรียก method แบบ explicit receiver
self.hello  # บน current object
obj.hello   # บน obj
```

### Method Parentheses

```ruby
# ใน Ruby วงเล็บเป็น optional สำหรับ method calls
puts "Hello"    # ✅ ไม่ใส่วงเล็บ
puts("Hello")   # ✅ ใส่วงเล็บ

# แต่ควรใส่วงเล็บเมื่อมี argument เพื่อความชัดเจน
greet "Alice"      # ✅ ทำงานได้
greet("Alice")     # ✅ ชัดเจนกว่า (recommended)

# ต้องใส่วงเล็บเมื่อ chain method
result = add(3, 4).to_s
# result = add 3, 4 .to_s  # ❌ ambiguous
```

---

## Step 148: Parameters และ Arguments

```ruby
# Parameters = ชื่อที่ใช้ใน method definition
# Arguments = ค่าจริงที่ส่งเข้าไปตอนเรียก

def add(a, b)  # a, b คือ parameters
  a + b
end

add(3, 4)  # 3, 4 คือ arguments

# Multiple parameters
def create_user(name, age, email)
  { name: name, age: age, email: email }
end

user = create_user("Alice", 30, "alice@example.com")
puts user.inspect
# {:name=>"Alice", :age=>30, :email=>"alice@example.com"}

# Order matters!
def introduce(name, job)
  "ผมชื่อ #{name} ทำงานเป็น #{job}"
end

puts introduce("Alice", "Developer")  # ผมชื่อ Alice ทำงานเป็น Developer
puts introduce("Developer", "Alice")  # ผมชื่อ Developer ทำงานเป็น Alice (ผิด!)
```

---

## Step 149: Default Parameters — ค่าเริ่มต้น

```ruby
# Default parameter
def greet(name, greeting = "สวัสดี")
  "#{greeting}, #{name}!"
end

puts greet("Alice")             # สวัสดี, Alice!
puts greet("Bob", "Hello")      # Hello, Bob!
puts greet("Charlie", "ดีจ้า")  # ดีจ้า, Charlie!

# Default ที่ซับซ้อนขึ้น
def create_user(name, age = 18, role = :user, active = true)
  { name: name, age: age, role: role, active: active }
end

puts create_user("Alice").inspect
# {:name=>"Alice", :age=>18, :role=>:user, :active=>true}

puts create_user("Bob", 25, :admin).inspect
# {:name=>"Bob", :age=>25, :role=>:admin, :active=>true}
```

### Default กับ Expression

```ruby
# Default สามารถเป็น expression ได้
def log_message(msg, timestamp = Time.now)
  "[#{timestamp.strftime('%H:%M:%S')}] #{msg}"
end

# Default อ้างอิง parameter ก่อนหน้าได้
def multiply(a, b = a * 2)  # b default คือ a * 2
  a * b
end

puts multiply(3)     # 3 * 6 = 18
puts multiply(3, 4)  # 3 * 4 = 12

# Default กับ method call
def connect(host, port = default_port, timeout = 30)
  # ...
end
```

---

## Step 150: Keyword Arguments — Named Parameters

```ruby
# Keyword arguments ทำให้เรียก method ชัดเจนขึ้น
def create_order(product:, quantity:, price:)
  {
    product: product,
    quantity: quantity,
    price: price,
    total: quantity * price
  }
end

# ต้องระบุชื่อตอนเรียก
order = create_order(product: "กาแฟ", quantity: 3, price: 50)
puts order[:total]  # 150

# ลำดับไม่สำคัญ!
order = create_order(price: 50, product: "กาแฟ", quantity: 3)
puts order[:total]  # 150 (เหมือนกัน)
```

### Keyword กับ Default

```ruby
def send_email(to:, subject:, body:, cc: nil, bcc: nil, html: false)
  puts "ถึง: #{to}"
  puts "เรื่อง: #{subject}"
  puts "สำเนา: #{cc}" if cc
  puts "HTML: #{html}"
end

send_email(
  to: "alice@example.com",
  subject: "ทดสอบ",
  body: "เนื้อหา"
)
# ถึง: alice@example.com
# เรื่อง: ทดสอบ
# HTML: false

send_email(
  to: "bob@example.com",
  subject: "ประชุม",
  body: "รายละเอียด",
  html: true,
  cc: "charlie@example.com"
)
```

### Keyword Arguments ช่วยอ่านง่าย

```ruby
# ❌ อ่านยาก: positional arguments
connect("localhost", 5432, "mydb", "admin", "secret", true, 30)

# ✅ อ่านง่าย: keyword arguments
connect(
  host: "localhost",
  port: 5432,
  database: "mydb",
  username: "admin",
  password: "secret",
  ssl: true,
  timeout: 30
)
```

---

## Step 151: Required Keyword Arguments

```ruby
# Ruby 2.1+: keyword ที่ไม่มี default = required
def create_user(name:, email:, age: nil)
  { name: name, email: email, age: age }
end

# ✅ ถูกต้อง
create_user(name: "Alice", email: "alice@example.com")

# ❌ Error: missing keyword: name
# create_user(email: "alice@example.com")

# ❌ Error: missing keyword: email
# create_user(name: "Alice")

# ตัวอย่างจริง
def transfer_money(from:, to:, amount:, note: "")
  return "จำนวนเงินต้องเป็นบวก" unless amount > 0
  
  puts "โอน #{amount} บาท"
  puts "จาก: #{from}"
  puts "ไปยัง: #{to}"
  puts "หมายเหตุ: #{note}" unless note.empty?
end

transfer_money(from: "Alice", to: "Bob", amount: 1000, note: "ค่าอาหาร")
```

---

## Step 152: Splat Operator (*args) — Variable Arguments

```ruby
# * รับ arguments จำนวนไม่แน่นอน
def sum(*numbers)
  numbers.sum
end

puts sum(1, 2, 3)         # 6
puts sum(1, 2, 3, 4, 5)   # 15
puts sum                   # 0

# *args เป็น Array ใน method
def greet_all(*names)
  puts "สวัสดี: #{names.join(', ')}!"
end

greet_all("Alice", "Bob", "Charlie")
# สวัสดี: Alice, Bob, Charlie!

# ผสมกับ required parameters
def log(level, *messages)
  messages.each do |msg|
    puts "[#{level.upcase}] #{msg}"
  end
end

log("info", "เริ่มต้นระบบ", "เชื่อมต่อ Database", "โหลด Config")
# [INFO] เริ่มต้นระบบ
# [INFO] เชื่อมต่อ Database
# [INFO] โหลด Config
```

### Splat ใน Method Call

```ruby
# ใช้ * ขยาย array เป็น arguments
def add(a, b, c)
  a + b + c
end

numbers = [1, 2, 3]
puts add(*numbers)  # 6

# รวม arrays
def combine(*arrays)
  arrays.flatten
end

a = [1, 2, 3]
b = [4, 5, 6]
c = [7, 8, 9]
puts combine(a, b, c).inspect  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

## Step 153: Double Splat (**kwargs) — Variable Keyword Arguments

```ruby
# ** รับ keyword arguments จำนวนไม่แน่นอน
def create_tag(tag, **attributes)
  attr_string = attributes.map { |k, v| "#{k}=\"#{v}\"" }.join(" ")
  "<#{tag} #{attr_string}>"
end

puts create_tag("a", href: "https://example.com", class: "link")
# <a href="https://example.com" class="link">

puts create_tag("img", src: "photo.jpg", alt: "รูปภาพ", width: "100")
# <img src="photo.jpg" alt="รูปภาพ" width="100">

# ผสม positional, splat, double splat
def complex_method(required, *optional, key: "default", **options)
  puts "required: #{required}"
  puts "optional: #{optional.inspect}"
  puts "key: #{key}"
  puts "options: #{options.inspect}"
end

complex_method("a", "b", "c", key: "custom", x: 1, y: 2)
# required: a
# optional: ["b", "c"]
# key: custom
# options: {:x=>1, :y=>2}
```

### Double Splat กับ Hash

```ruby
# ** ขยาย Hash เป็น keyword arguments
def configure(debug: false, log_level: :info, timeout: 30)
  puts "debug: #{debug}, log: #{log_level}, timeout: #{timeout}"
end

config = { debug: true, log_level: :debug }
configure(**config)
# debug: true, log: debug, timeout: 30

# merge hashes ด้วย **
defaults = { color: "blue", size: "medium" }
overrides = { color: "red" }
merged = { **defaults, **overrides }
puts merged.inspect  # {:color=>"red", :size=>"medium"}
```

---

## Step 154: Mixed Parameters — ผสมทุกประเภท

```ruby
# Ruby parameters ทุกประเภทผสมกัน
def everything(
  required,          # required positional
  optional = "opt",  # optional positional
  *rest,             # splat
  keyword:,          # required keyword
  key_opt: "def",    # optional keyword
  **options,         # double splat
  &block             # block
)
  puts "required: #{required}"
  puts "optional: #{optional}"
  puts "rest: #{rest.inspect}"
  puts "keyword: #{keyword}"
  puts "key_opt: #{key_opt}"
  puts "options: #{options.inspect}"
  block.call if block
end

everything("a", "b", "c", "d",
           keyword: "req", x: 1) { puts "Block!" }
# required: a
# optional: b
# rest: ["c", "d"]
# keyword: req
# key_opt: def
# options: {:x=>1}
# Block!
```

### ลำดับ Parameters ที่ถูกต้อง

```ruby
# ลำดับที่ Ruby กำหนด:
# 1. required positional
# 2. optional positional
# 3. *splat
# 4. required keyword
# 5. optional keyword
# 6. **double_splat
# 7. &block

def correct_order(req, opt = "opt", *splat, kwreq:, kwopt: "kd", **dbl, &blk)
  # ...
end
```

---

## Step 155: Return Values — Implicit vs Explicit

```ruby
# Implicit return: ค่าสุดท้ายของ method
def add(a, b)
  a + b  # return โดยอัตโนมัติ
end

puts add(3, 4)  # 7

# Explicit return
def divide(a, b)
  return "หารด้วยศูนย์ไม่ได้" if b == 0
  a.to_f / b
end

puts divide(10, 2)  # 5.0
puts divide(10, 0)  # หารด้วยศูนย์ไม่ได้

# Implicit return ของ if/case
def classify_age(age)
  if age < 13
    "เด็ก"
  elsif age < 18
    "วัยรุ่น"
  elsif age < 60
    "ผู้ใหญ่"
  else
    "ผู้สูงอายุ"
  end  # ค่าสุดท้ายจาก if block
end

puts classify_age(8)   # เด็ก
puts classify_age(25)  # ผู้ใหญ่
```

### Return ใน Guard Clauses

```ruby
def process_user(user)
  # guard clauses ใช้ explicit return
  return nil unless user
  return { error: "inactive" } unless user[:active]
  return { error: "too_young" } if user[:age] < 18
  
  # happy path
  {
    success: true,
    user_id: user[:id],
    message: "ดำเนินการสำเร็จ"
  }
end
```

---

## Step 156: Multiple Return Values

```ruby
# Ruby methods คืนค่าได้ค่าเดียว แต่ใช้ Array ได้
def min_max(numbers)
  [numbers.min, numbers.max]
end

min, max = min_max([3, 1, 4, 1, 5, 9, 2, 6])
puts "min: #{min}, max: #{max}"  # min: 1, max: 9

# คืนเป็น Hash เพื่อความชัดเจน
def statistics(numbers)
  sum = numbers.sum
  avg = sum.to_f / numbers.length
  {
    sum: sum,
    average: avg.round(2),
    min: numbers.min,
    max: numbers.max,
    count: numbers.length
  }
end

stats = statistics([1, 2, 3, 4, 5])
puts "รวม: #{stats[:sum]}, เฉลี่ย: #{stats[:average]}"
# รวม: 15, เฉลี่ย: 3.0

# Destructuring assignment
def parse_name(full_name)
  parts = full_name.split
  first = parts.first
  last = parts.last
  [first, last]
end

first, last = parse_name("John Doe")
puts "ชื่อ: #{first}, นามสกุล: #{last}"
# ชื่อ: John, นามสกุล: Doe
```

---

## Step 157: Method Visibility — public, private, protected

```ruby
class BankAccount
  def initialize(balance)
    @balance = balance
  end

  # public methods — เรียกจากนอก class ได้
  def deposit(amount)
    validate_amount!(amount)
    @balance += amount
    log_transaction("deposit", amount)
    @balance
  end

  def withdraw(amount)
    validate_amount!(amount)
    check_sufficient_funds!(amount)
    @balance -= amount
    log_transaction("withdraw", amount)
    @balance
  end

  def balance
    @balance
  end

  private

  # private methods — เรียกได้แค่ใน class เดียวกัน
  def validate_amount!(amount)
    raise ArgumentError, "จำนวนเงินต้องเป็นบวก" unless amount > 0
  end

  def check_sufficient_funds!(amount)
    raise "ยอดเงินไม่เพียงพอ" if amount > @balance
  end

  def log_transaction(type, amount)
    puts "[LOG] #{type}: #{amount} บาท | ยอดคงเหลือ: #{@balance} บาท"
  end

  protected

  # protected methods — เรียกได้จาก subclass และ instance เดียวกัน
  def transfer_to(other_account, amount)
    withdraw(amount)
    other_account.deposit(amount)
  end
end

account = BankAccount.new(1000)
puts account.deposit(500)   # 1500
puts account.withdraw(200)  # 1300

# account.validate_amount!(100)  # NoMethodError: private method
```

### Private Method Ruby 2.7+ Style

```ruby
class User
  def initialize(name, email)
    @name = name
    @email = email
  end

  def display
    "#{formatted_name} <#{masked_email}>"
  end

  private def formatted_name  # Ruby 2.7+ inline private
    @name.split.map(&:capitalize).join(" ")
  end

  private def masked_email
    local, domain = @email.split("@")
    "#{local[0]}***@#{domain}"
  end
end

user = User.new("john doe", "johndoe@example.com")
puts user.display  # John Doe <j***@example.com>
```

---

## Step 158: Bang Methods (!) — Mutating Methods

```ruby
# Bang methods มักแก้ไข object ที่เรียก (in-place)
name = "hello world"
puts name.upcase    # "HELLO WORLD" — คืนค่าใหม่, name ไม่เปลี่ยน
puts name           # "hello world" — ยังเหมือนเดิม

name.upcase!        # แก้ไข name โดยตรง
puts name           # "HELLO WORLD"

# ตัวอย่างกับ Array
arr = [3, 1, 4, 1, 5, 9, 2, 6]
sorted = arr.sort    # คืน array ใหม่
puts arr.inspect     # [3, 1, 4, 1, 5, 9, 2, 6] — ยังเหมือนเดิม

arr.sort!            # แก้ไข array ใน place
puts arr.inspect     # [1, 1, 2, 3, 4, 5, 6, 9]

# map vs map!
numbers = [1, 2, 3, 4, 5]
doubled = numbers.map { |n| n * 2 }    # คืน array ใหม่
puts numbers.inspect                    # [1, 2, 3, 4, 5] — ไม่เปลี่ยน

numbers.map! { |n| n * 2 }             # แก้ไข in-place
puts numbers.inspect                    # [2, 4, 6, 8, 10]
```

### สร้าง Bang Methods เอง

```ruby
class Product
  attr_accessor :name, :price, :stock

  def initialize(name, price, stock)
    @name = name
    @price = price
    @stock = stock
  end

  # Non-bang: คืน object ใหม่
  def apply_discount(percent)
    new_price = @price * (1 - percent / 100.0)
    Product.new(@name, new_price, @stock)
  end

  # Bang: แก้ไข object นี้
  def apply_discount!(percent)
    @price = @price * (1 - percent / 100.0)
    self  # คืน self เพื่อให้ chain ได้
  end

  # Non-bang: คืน boolean
  def save
    # บันทึกไปยัง database
    @id = SecureRandom.uuid
    true
  rescue
    false
  end

  # Bang: raise error ถ้าล้มเหลว
  def save!
    save or raise "บันทึกไม่สำเร็จ: #{name}"
  end
end

coffee = Product.new("กาแฟ", 100, 50)
cheap_coffee = coffee.apply_discount(20)
puts cheap_coffee.price  # 80.0
puts coffee.price        # 100 — ยังเหมือนเดิม

coffee.apply_discount!(20)
puts coffee.price        # 80.0 — เปลี่ยนแล้ว!
```

---

## Step 159: Predicate Methods (?) — Boolean Methods

```ruby
# Method ลงท้าย ? คืน true/false
class User
  def initialize(age, active, admin)
    @age = age
    @active = active
    @admin = admin
  end

  def adult?
    @age >= 18
  end

  def active?
    @active
  end

  def admin?
    @admin
  end

  def teenager?
    (13..17).cover?(@age)
  end

  def can_access?(feature)
    case feature
    when :admin_panel  then admin?
    when :reports      then active? && adult?
    when :dashboard    then active?
    else false
    end
  end
end

user = User.new(25, true, false)
puts user.adult?          # true
puts user.active?         # true
puts user.admin?          # false
puts user.can_access?(:reports)    # true
puts user.can_access?(:admin_panel) # false
```

### Built-in Predicate Methods

```ruby
# String
puts "hello".empty?         # false
puts "".empty?              # true
puts "hello".include?("ll") # true
puts "hello".start_with?("he") # true
puts "hello".end_with?("lo")   # true

# Array
puts [].empty?              # true
puts [1, 2, 3].any? { |n| n > 2 } # true
puts [2, 4, 6].all?(&:even?)      # true

# Numeric
puts 5.zero?       # false
puts 0.zero?       # true
puts 5.positive?   # true
puts (-3).negative? # true
puts 4.even?       # true
puts 5.odd?        # true
puts 5.between?(1, 10) # true

# Object
puts nil.nil?      # true
puts 5.nil?        # false
puts 5.is_a?(Integer) # true
puts 5.respond_to?(:to_s) # true
```

---

## Step 160: Method Aliases — alias และ alias_method

```ruby
class Array
  # alias สร้างชื่อเรียกแทนได้
  alias_method :contains?, :include?
  alias_method :size, :length
end

arr = [1, 2, 3, 4, 5]
puts arr.contains?(3)  # true
puts arr.size          # 5

# alias keyword (แบบเก่า)
class String
  alias old_reverse reverse
  
  def reverse
    super.swapcase  # override แต่ยังใช้ original ได้ผ่าน old_reverse
  end
end

# สร้าง aliases ใน class เอง
class Calculator
  def add(a, b)
    a + b
  end
  alias plus add  # plus เป็น alias ของ add
  alias :sum :add # รูปแบบ symbol
end

calc = Calculator.new
puts calc.add(3, 4)   # 7
puts calc.plus(3, 4)  # 7
puts calc.sum(3, 4)   # 7
```

### เหตุที่ใช้ alias_method

```ruby
class EmailNotifier
  def send_notification(message)
    # ส่ง email
    puts "Email: #{message}"
  end
  
  # สร้าง alias เพื่อ backward compatibility
  alias_method :notify, :send_notification
  alias_method :alert,  :send_notification
end

notifier = EmailNotifier.new
notifier.send_notification("ประชุม")  # Email: ประชุม
notifier.notify("ประชุม")            # Email: ประชุม
notifier.alert("ประชุม")             # Email: ประชุม
```

---

## Step 161: Method Objects — method(:name)

```ruby
# Method ใน Ruby เป็น object ได้!
def square(n)
  n ** 2
end

m = method(:square)
puts m.class      # Method
puts m.call(5)    # 25
puts m.(5)        # 25 (shorthand)
puts m[5]         # 25 (shorthand)

# Method object มี arity
puts m.arity  # 1

# ใช้กับ built-in methods
upcase_method = "hello".method(:upcase)
puts upcase_method.call  # HELLO

# map ด้วย method object
numbers = [1, 2, 3, 4, 5]
squares = numbers.map(&method(:square))
puts squares.inspect  # [1, 4, 9, 16, 25]
```

### UnboundMethod

```ruby
# UnboundMethod: method ที่ยังไม่ผูกกับ object
class Greeter
  def hello
    "Hello, I'm #{@name}"
  end
end

unbound = Greeter.instance_method(:hello)
puts unbound.class  # UnboundMethod

# ต้อง bind กับ object ก่อนเรียก
alice = Greeter.new
alice.instance_variable_set(:@name, "Alice")
bound = unbound.bind(alice)
puts bound.call  # Hello, I'm Alice
```

---

## Step 162: Passing Methods as Blocks (&method(:name))

```ruby
# & แปลง method เป็น block
numbers = [-3, -1, 0, 2, 4, 6]

# แบบปกติ
positives = numbers.select { |n| n.positive? }

# ด้วย method object
positives = numbers.select(&method(:positive?))  # ถ้า positive? เป็น method

# ด้วย Symbol#to_proc
positives = numbers.select(&:positive?)  # สั้นกว่า!
puts positives.inspect  # [2, 4, 6]

# ตัวอย่างการใช้
words = ["hello", "WORLD", "ruby", "PROGRAMMING"]

# แบบปกติ
upcased = words.map { |w| w.upcase }

# ด้วย Symbol#to_proc
upcased = words.map(&:upcase)
puts upcased.inspect  # ["HELLO", "WORLD", "RUBY", "PROGRAMMING"]

# map กับ custom method
def double(n)
  n * 2
end

numbers = [1, 2, 3, 4, 5]
puts numbers.map(&method(:double)).inspect  # [2, 4, 6, 8, 10]
```

### Symbol to Proc (&:method_name)

```ruby
# & บน symbol เรียก to_proc เพื่อสร้าง block
# :upcase เหมือนกับ { |s| s.upcase }
# :to_i เหมือนกับ { |s| s.to_i }

["1", "2", "3"].map(&:to_i)     # [1, 2, 3]
[1, 2, 3].map(&:to_s)           # ["1", "2", "3"]
["a", "b", "c"].map(&:upcase)   # ["A", "B", "C"]
[1, nil, 2, nil, 3].compact     # [1, 2, 3]
[1, nil, 2, nil, 3].select(&:itself)  # [1, 2, 3]
```

---

## Step 163: Recursive Methods — การเรียกตัวเอง

```ruby
# Factorial
def factorial(n)
  return 1 if n <= 1
  n * factorial(n - 1)
end

puts factorial(5)  # 120
puts factorial(10) # 3628800

# Fibonacci
def fib(n)
  return n if n <= 1
  fib(n - 1) + fib(n - 2)
end

puts fib(10)  # 55
# หมายเหตุ: naive recursion ช้า ใช้ memoization แทน

# Tower of Hanoi
def hanoi(n, from = "A", to = "C", via = "B")
  return if n == 0
  hanoi(n - 1, from, via, to)
  puts "เคลื่อน disk #{n} จาก #{from} ไป #{to}"
  hanoi(n - 1, via, to, from)
end

hanoi(3)
# เคลื่อน disk 1 จาก A ไป C
# เคลื่อน disk 2 จาก A ไป B
# เคลื่อน disk 1 จาก C ไป B
# เคลื่อน disk 3 จาก A ไป C
# เคลื่อน disk 1 จาก B ไป A
# เคลื่อน disk 2 จาก B ไป C
# เคลื่อน disk 1 จาก A ไป C
```

### Recursion กับ Accumulator (Tail Recursion Pattern)

```ruby
# ปกติ Ruby ไม่ optimize tail recursion แต่เขียนให้ชัดเจนได้
def factorial_tail(n, acc = 1)
  return acc if n <= 1
  factorial_tail(n - 1, n * acc)
end

puts factorial_tail(10)  # 3628800

# Flatten ด้วย recursion
def deep_flatten(arr)
  arr.each_with_object([]) do |item, result|
    if item.is_a?(Array)
      result.concat(deep_flatten(item))
    else
      result << item
    end
  end
end

puts deep_flatten([1, [2, [3, [4, 5]], 6]]).inspect  # [1, 2, 3, 4, 5, 6]
```

---

## Step 164: Method Chaining — เชื่อม Methods ต่อกัน

```ruby
# Method chaining ทำได้เมื่อ method คืน self หรือ object ที่มี method ต่อ

# String chaining
"  hello world  ".strip.split.map(&:capitalize).join(" ")
# => "Hello World"

# Array chaining
[1, 2, 3, 4, 5]
  .map { |n| n * 2 }
  .select { |n| n > 4 }
  .sum
# => 24

# สร้าง class ที่ chain ได้
class QueryBuilder
  def initialize
    @table = nil
    @conditions = []
    @order = nil
    @limit = nil
  end

  def from(table)
    @table = table
    self  # คืน self เพื่อให้ chain ได้
  end

  def where(condition)
    @conditions << condition
    self
  end

  def order_by(column, direction = :asc)
    @order = "#{column} #{direction.upcase}"
    self
  end

  def limit(n)
    @limit = n
    self
  end

  def to_sql
    sql = "SELECT * FROM #{@table}"
    sql += " WHERE #{@conditions.join(' AND ')}" unless @conditions.empty?
    sql += " ORDER BY #{@order}" if @order
    sql += " LIMIT #{@limit}" if @limit
    sql
  end
end

query = QueryBuilder.new
          .from("users")
          .where("age >= 18")
          .where("active = true")
          .order_by(:name)
          .limit(10)
          .to_sql

puts query
# SELECT * FROM users WHERE age >= 18 AND active = true ORDER BY name ASC LIMIT 10
```

---

## Step 165: Memoization Pattern — Cache Results

```ruby
# Memoization คือการ cache ผลลัพธ์ของการคำนวณ
class Fibonacci
  def initialize
    @cache = {}
  end

  def calculate(n)
    return n if n <= 1
    @cache[n] ||= calculate(n - 1) + calculate(n - 2)
  end
end

fib = Fibonacci.new
puts fib.calculate(50)  # 12586269025 (เร็วมาก!)

# Memoization ด้วย ||=
class ExpensiveCalculator
  def initialize(data)
    @data = data
  end

  def result
    @result ||= compute_expensive_result
  end

  def summary
    @summary ||= build_summary
  end

  private

  def compute_expensive_result
    puts "กำลังคำนวณ..." 
    @data.map { |x| x ** 3 }.sum
  end

  def build_summary
    puts "กำลังสร้าง summary..."
    { total: result, count: @data.length, average: result.to_f / @data.length }
  end
end

calc = ExpensiveCalculator.new([1, 2, 3, 4, 5])
puts calc.result  # กำลังคำนวณ... 225
puts calc.result  # 225 (ไม่คำนวณซ้ำ!)
puts calc.summary.inspect  # กำลังสร้าง summary... {:total=>225, :count=>5, :average=>45.0}
```

### Thread-safe Memoization

```ruby
# ใน multi-threaded environment ต้องระวัง
class SafeMemoizer
  def initialize
    @mutex = Mutex.new
    @cache = {}
  end

  def fetch(key, &computation)
    @cache[key] || @mutex.synchronize do
      @cache[key] ||= computation.call
    end
  end
end
```

---

## Step 166–170: Advanced Method Patterns

### Step 166: Method Missing

```ruby
class FlexibleHash
  def initialize
    @data = {}
  end

  # เรียก method ที่ไม่มี
  def method_missing(name, *args)
    key = name.to_s
    
    if key.end_with?("=")
      # setter: obj.name = "Alice"
      @data[key.chomp("=")] = args.first
    elsif key.end_with?("?")
      # predicate: obj.admin?
      !@data[key.chomp("?")].nil?
    else
      # getter: obj.name
      @data[key]
    end
  end

  def respond_to_missing?(name, include_private = false)
    true
  end
end

person = FlexibleHash.new
person.name = "Alice"
person.age = 30
person.admin = true

puts person.name    # Alice
puts person.age     # 30
puts person.admin?  # true
puts person.email?  # false
```

### Step 167: Callable Objects

```ruby
# Ruby มีหลายวิธีในการสร้าง callable objects

# 1. Method object
def double(n) = n * 2
m = method(:double)
m.call(5)  # 10

# 2. Proc
p = Proc.new { |n| n * 2 }
p.call(5)  # 10

# 3. Lambda
l = lambda { |n| n * 2 }
l.call(5)  # 10

# 4. Stabby lambda
sl = ->(n) { n * 2 }
sl.call(5)  # 10

# ทุกตัว respond to call
[m, p, l, sl].each do |callable|
  puts callable.call(5)  # 10 ทั้งหมด
end
```

### Step 168: Method Decoration Pattern

```ruby
# เพิ่มความสามารถให้ method โดยไม่แก้ต้นฉบับ
module Logging
  def self.included(base)
    base.instance_methods(false).each do |method_name|
      original = base.instance_method(method_name)
      base.define_method(method_name) do |*args, &block|
        puts "[LOG] Calling #{method_name}(#{args.inspect})"
        result = original.bind(self).call(*args, &block)
        puts "[LOG] #{method_name} returned #{result.inspect}"
        result
      end
    end
  end
end

class Calculator
  def add(a, b) = a + b
  def multiply(a, b) = a * b
  
  # include หลัง method definitions
  include Logging
end

calc = Calculator.new
calc.add(3, 4)
# [LOG] Calling add([3, 4])
# [LOG] add returned 7
```

### Step 169: Functional Methods

```ruby
# Ruby รองรับ functional programming style
# compose methods

def compose(*fns)
  fns.reduce { |f, g| ->(x) { f.call(g.call(x)) } }
end

double = ->(n) { n * 2 }
square = ->(n) { n ** 2 }
add_one = ->(n) { n + 1 }

double_then_square = compose(square, double)  # square(double(x))
puts double_then_square.call(3)  # (3*2)^2 = 36

pipeline = compose(add_one, square, double)  # add_one(square(double(x)))
puts pipeline.call(3)  # add_one(square(6)) = add_one(36) = 37

# Method pipeline ด้วย >> และ <<  (Ruby 2.6+)
triple = ->(n) { n * 3 }
square_f = ->(n) { n ** 2 }
to_s_f = ->(n) { n.to_s }

pipeline = triple >> square_f >> to_s_f
puts pipeline.call(2)  # ((2*3)^2).to_s = "36"
```

### Step 170: Mixin Methods และ Module Functions

```ruby
# Module methods
module MathHelper
  def self.circle_area(radius)
    Math::PI * radius ** 2
  end

  def self.distance(x1, y1, x2, y2)
    Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2)
  end
end

puts MathHelper.circle_area(5).round(2)  # 78.54
puts MathHelper.distance(0, 0, 3, 4).round(2)  # 5.0

# Module ที่ include ได้
module Greetable
  def greet
    "สวัสดี! ฉันชื่อ #{name}"
  end

  def farewell
    "ลาก่อน! ขอบคุณ #{name}"
  end
end

class Person
  include Greetable
  attr_reader :name

  def initialize(name)
    @name = name
  end
end

alice = Person.new("Alice")
puts alice.greet    # สวัสดี! ฉันชื่อ Alice
puts alice.farewell # ลาก่อน! ขอบคุณ Alice
```

---

## แบบฝึกหัดตอนที่ 9 (25 ข้อ)

### ข้อ 1-5: พื้นฐาน Methods

```ruby
# ข้อ 1: String Calculator
def string_calculator(expression)
  # รับ string เช่น "3 + 4" แล้วคืนผลลัพธ์
  parts = expression.split
  a = parts[0].to_f
  op = parts[1]
  b = parts[2].to_f
  
  case op
  when "+" then a + b
  when "-" then a - b
  when "*" then a * b
  when "/" then b != 0 ? a / b : "Error: Division by zero"
  when "%" then a % b
  when "**" then a ** b
  else "Error: Unknown operator"
  end
end

puts string_calculator("10 + 5")    # 15.0
puts string_calculator("20 / 4")    # 5.0
puts string_calculator("3 ** 4")    # 81.0
puts string_calculator("10 / 0")    # Error: Division by zero

# ข้อ 2: Titlecase
def titlecase(str)
  # แปลง "hello world" -> "Hello World"
  # แต่ words เช่น "a", "an", "the", "in", "of" ไม่ capitalize (ยกเว้นตัวแรก)
  small_words = %w[a an the in of on at to for and but or nor]
  words = str.downcase.split
  words.each_with_index.map do |word, i|
    (i == 0 || !small_words.include?(word)) ? word.capitalize : word
  end.join(" ")
end

puts titlecase("the quick brown fox")    # The Quick Brown Fox
puts titlecase("lord of the rings")      # Lord of the Rings
puts titlecase("a tale of two cities")   # A Tale of Two Cities

# ข้อ 3: Caesar Cipher
def caesar_cipher(text, shift)
  text.chars.map do |char|
    if char.match?(/[A-Za-z]/)
      base = char.match?(/[A-Z]/) ? "A".ord : "a".ord
      ((char.ord - base + shift) % 26 + base).chr
    else
      char
    end
  end.join
end

puts caesar_cipher("Hello, World!", 3)   # Khoor, Zruog!
puts caesar_cipher("Khoor, Zruog!", -3)  # Hello, World!

# ข้อ 4: Pangram checker
def pangram?(sentence)
  ("a".."z").all? { |c| sentence.downcase.include?(c) }
end

puts pangram?("The quick brown fox jumps over the lazy dog")  # true
puts pangram?("Hello World")  # false

# ข้อ 5: สร้าง Matrix
def create_matrix(rows, cols, &filler)
  filler ||= ->(r, c) { 0 }
  Array.new(rows) { |r| Array.new(cols) { |c| filler.call(r, c) } }
end

identity = create_matrix(3, 3) { |r, c| r == c ? 1 : 0 }
identity.each { |row| puts row.inspect }
# [1, 0, 0]
# [0, 1, 0]
# [0, 0, 1]

multiplication = create_matrix(5, 5) { |r, c| (r + 1) * (c + 1) }
multiplication.each { |row| puts row.map { |n| n.to_s.rjust(4) }.join }
```

### ข้อ 6-10: Keyword Arguments และ Splat

```ruby
# ข้อ 6: URL Builder
def build_url(base_url, path = "", **params)
  url = "#{base_url.chomp('/')}/#{path.gsub(/^\//, '')}"
  unless params.empty?
    query = params.map { |k, v| "#{k}=#{URI.encode_www_form_component(v.to_s)}" }.join("&")
    url += "?#{query}"
  end
  url
end

require "uri"
puts build_url("https://api.example.com", "users", page: 1, limit: 10, sort: "name")
# https://api.example.com/users?page=1&limit=10&sort=name

# ข้อ 7: Flexible Logger
def log(*messages, level: :info, timestamp: true, prefix: nil)
  ts = timestamp ? "[#{Time.now.strftime('%Y-%m-%d %H:%M:%S')}] " : ""
  pre = prefix ? "[#{prefix}] " : ""
  icon = { info: "ℹ", warn: "⚠", error: "✗", debug: "⚙" }[level] || "?"
  
  messages.each do |msg|
    puts "#{ts}#{icon} #{pre}#{msg}"
  end
end

log("เริ่มต้นระบบ", level: :info)
log("CPU สูง", "Memory เต็ม", level: :warn, prefix: "SYSTEM")
log("Database connection failed", level: :error, timestamp: false)

# ข้อ 8: Table Formatter
def format_table(headers, rows, **options)
  col_widths = headers.length.times.map do |i|
    [headers[i].length, *rows.map { |r| r[i].to_s.length }].max
  end
  
  separator = "+#{col_widths.map { |w| "-" * (w + 2) }.join("+")}+"
  
  format_row = ->(row) {
    "|#{row.each_with_index.map { |cell, i| " #{cell.to_s.ljust(col_widths[i])} " }.join("|")}|"
  }
  
  lines = [separator, format_row.call(headers), separator]
  rows.each { |row| lines << format_row.call(row) }
  lines << separator
  lines.join("\n")
end

headers = ["ชื่อ", "อายุ", "เมือง"]
rows = [
  ["Alice", 30, "กรุงเทพ"],
  ["Bob", 25, "เชียงใหม่"],
  ["Charlie", 35, "ภูเก็ต"]
]
puts format_table(headers, rows)
# +-------+-----+-----------+
# | ชื่อ  | อายุ | เมือง     |
# +-------+-----+-----------+
# | Alice | 30  | กรุงเทพ   |
# | Bob   | 25  | เชียงใหม่ |
# | Charlie | 35 | ภูเก็ต  |
# +-------+-----+-----------+

# ข้อ 9: Method Profiler
def profile(method_name, *args, runs: 100, **kwargs)
  times = runs.times.map do
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    method(method_name).call(*args, **kwargs)
    Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
  end
  
  {
    method: method_name,
    runs: runs,
    total_ms: (times.sum * 1000).round(3),
    avg_ms: (times.sum / runs * 1000).round(3),
    min_ms: (times.min * 1000).round(3),
    max_ms: (times.max * 1000).round(3)
  }
end

def slow_sum(n)
  (1..n).sum
end

result = profile(:slow_sum, 10000, runs: 50)
puts result.inspect

# ข้อ 10: Deep Merge
def deep_merge(hash1, hash2)
  hash1.merge(hash2) do |_, old_val, new_val|
    if old_val.is_a?(Hash) && new_val.is_a?(Hash)
      deep_merge(old_val, new_val)
    elsif old_val.is_a?(Array) && new_val.is_a?(Array)
      old_val + new_val
    else
      new_val
    end
  end
end

config1 = { db: { host: "localhost", port: 5432 }, debug: false, tags: ["v1"] }
config2 = { db: { port: 5433, name: "mydb" }, debug: true, tags: ["v2"] }
merged = deep_merge(config1, config2)
puts merged.inspect
# {:db=>{:host=>"localhost", :port=>5433, :name=>"mydb"}, :debug=>true, :tags=>["v1", "v2"]}
```

### ข้อ 11-15: Return Values และ Visibility

```ruby
# ข้อ 11: Result Pattern
class Result
  attr_reader :value, :error

  def self.ok(value)
    new(value: value)
  end

  def self.err(error)
    new(error: error)
  end

  def initialize(value: nil, error: nil)
    @value = value
    @error = error
    @success = error.nil?
  end

  def success?
    @success
  end

  def failure?
    !@success
  end

  def on_success(&block)
    block.call(@value) if success?
    self
  end

  def on_failure(&block)
    block.call(@error) if failure?
    self
  end

  def map(&block)
    success? ? Result.ok(block.call(@value)) : self
  end
end

def divide_safe(a, b)
  return Result.err("หารด้วยศูนย์ไม่ได้") if b == 0
  Result.ok(a.to_f / b)
end

divide_safe(10, 2)
  .on_success { |v| puts "ผลลัพธ์: #{v}" }
  .on_failure { |e| puts "Error: #{e}" }
# ผลลัพธ์: 5.0

divide_safe(10, 0)
  .on_success { |v| puts "ผลลัพธ์: #{v}" }
  .on_failure { |e| puts "Error: #{e}" }
# Error: หารด้วยศูนย์ไม่ได้

# ข้อ 12: Memoized Properties
class User
  def initialize(id)
    @id = id
  end

  def name
    @name ||= fetch_from_db(:name)
  end

  def email
    @email ||= fetch_from_db(:email)
  end

  def full_profile
    @full_profile ||= {
      id: @id,
      name: name,
      email: email,
      created_at: Time.now
    }
  end

  private

  def fetch_from_db(field)
    # simulate DB query
    sleep(0.001)
    { name: "User #{@id}", email: "user#{@id}@example.com" }[field]
  end
end

user = User.new(42)
3.times { puts user.name }  # DB query แค่ครั้งเดียว!

# ข้อ 13: Decorator Pattern
module Cacheable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def cache_method(*method_names, ttl: 60)
      method_names.each do |name|
        original = instance_method(name)
        cache = {}
        
        define_method(name) do |*args|
          key = [name, args].hash
          cached = cache[key]
          
          if cached && Time.now - cached[:time] < ttl
            cached[:value]
          else
            result = original.bind(self).call(*args)
            cache[key] = { value: result, time: Time.now }
            result
          end
        end
      end
    end
  end
end

class WeatherService
  include Cacheable
  
  def temperature(city)
    puts "Fetching weather for #{city}..."
    rand(20..35)
  end
  
  cache_method :temperature, ttl: 300  # cache 5 นาที
end

weather = WeatherService.new
puts weather.temperature("Bangkok")  # Fetching weather... 28
puts weather.temperature("Bangkok")  # ใช้ cache (ไม่ fetch ซ้ำ!)

# ข้อ 14: Chain of Responsibility
class Validator
  def initialize(name, &rule)
    @name = name
    @rule = rule
    @next_validator = nil
  end

  def then(validator)
    @next_validator = validator
    validator
  end

  def validate(value)
    unless @rule.call(value)
      return { valid: false, failed_at: @name }
    end
    @next_validator ? @next_validator.validate(value) : { valid: true }
  end
end

not_nil    = Validator.new("not_nil") { |v| !v.nil? }
not_empty  = Validator.new("not_empty") { |v| !v.to_s.empty? }
min_length = Validator.new("min_length_6") { |v| v.to_s.length >= 6 }
has_upper  = Validator.new("has_uppercase") { |v| v.to_s.match?(/[A-Z]/) }
has_digit  = Validator.new("has_digit") { |v| v.to_s.match?(/\d/) }

not_nil.then(not_empty).then(min_length).then(has_upper).then(has_digit)

puts not_nil.validate(nil).inspect          # {:valid=>false, :failed_at=>"not_nil"}
puts not_nil.validate("hello").inspect      # {:valid=>false, :failed_at=>"min_length_6"}
puts not_nil.validate("Hello1").inspect     # {:valid=>true}

# ข้อ 15: Builder Pattern
class HtmlBuilder
  def initialize(tag, **attrs)
    @tag = tag
    @attrs = attrs
    @children = []
    @text = nil
  end

  def text(content)
    @text = content
    self
  end

  def add(tag, **attrs, &block)
    child = HtmlBuilder.new(tag, **attrs)
    block.call(child) if block
    @children << child
    self
  end

  def build
    attr_str = @attrs.map { |k, v| " #{k}=\"#{v}\"" }.join
    inner = @text || @children.map(&:build).join
    "<#{@tag}#{attr_str}>#{inner}</#{@tag}>"
  end
end

html = HtmlBuilder.new("div", class: "card") do |div|
  # ไม่ใช้ block ที่นี่ (ตัวอย่างอื่น)
end

card = HtmlBuilder.new("div", class: "card")
card.add("h2") { |h| h.text("Ruby Methods") }
card.add("p", class: "description") { |p| p.text("เรียนรู้ methods ใน Ruby") }
card.add("a", href: "/learn") { |a| a.text("อ่านต่อ") }

puts card.build
# <div class="card"><h2>Ruby Methods</h2><p class="description">เรียนรู้ methods ใน Ruby</p><a href="/learn">อ่านต่อ</a></div>
```

### ข้อ 16-25: Advanced Methods

```ruby
# ข้อ 16: Method Overloading Simulation
class Shape
  def area(*args)
    case args
    in [Float | Integer => r] if args.length == 1
      # circle: area(radius)
      Math::PI * r ** 2
    in [Float | Integer => w, Float | Integer => h] if args.length == 2
      # rectangle: area(width, height)
      w * h
    in [Float | Integer => a, Float | Integer => b, Float | Integer => c] if args.length == 3
      # triangle: area(a, b, c) using Heron's formula
      s = (a + b + c) / 2.0
      Math.sqrt(s * (s-a) * (s-b) * (s-c))
    else
      raise ArgumentError, "ไม่รู้จักรูปแบบ"
    end
  end
end

shape = Shape.new
puts shape.area(5).round(2)          # 78.54 (วงกลม radius 5)
puts shape.area(4, 6).round(2)       # 24.0 (สี่เหลี่ยม 4x6)
puts shape.area(3, 4, 5).round(2)    # 6.0 (สามเหลี่ยม 3-4-5)

# ข้อ 17: Lazy Evaluator
class LazyValue
  def initialize(&computation)
    @computation = computation
    @evaluated = false
  end

  def value
    unless @evaluated
      @value = @computation.call
      @evaluated = true
    end
    @value
  end

  def to_s
    value.to_s
  end
end

expensive = LazyValue.new do
  puts "กำลังคำนวณ..."
  sleep(0.1)
  42
end

puts "ยังไม่คำนวณ"
puts expensive.value  # กำลังคำนวณ... 42
puts expensive.value  # 42 (cache)

# ข้อ 18: Method Pipeline
class Pipeline
  def initialize(*steps)
    @steps = steps
  end

  def call(input)
    @steps.reduce(input) { |value, step|
      case step
      when Symbol then value.send(step)
      when Proc, Method then step.call(value)
      else raise "ไม่รู้จัก step type: #{step.class}"
      end
    }
  end

  def >>(other_step)
    Pipeline.new(*@steps, other_step)
  end
end

pipeline = Pipeline.new(
  :strip,
  :downcase,
  ->(s) { s.gsub(/\s+/, "_") },
  :to_sym
)

puts pipeline.call("  Hello World  ").inspect  # :hello_world

# ข้อ 19: Aspect-Oriented Method Wrapping
module Around
  def self.wrap(object, method_name, before: nil, after: nil, rescue_with: nil)
    original = object.method(method_name)
    
    object.define_singleton_method(method_name) do |*args, **kwargs, &block|
      before&.call(method_name, args)
      begin
        result = original.call(*args, **kwargs, &block)
        after&.call(method_name, result)
        result
      rescue => e
        rescue_with ? rescue_with.call(e) : raise
      end
    end
  end
end

class Service
  def process(data)
    puts "Processing: #{data}"
    data.upcase
  end
end

svc = Service.new
Around.wrap(
  svc,
  :process,
  before: ->(name, args) { puts "[BEFORE] #{name}(#{args.inspect})" },
  after:  ->(name, result) { puts "[AFTER] #{name} -> #{result}" }
)

svc.process("hello")
# [BEFORE] process(["hello"])
# Processing: hello
# [AFTER] process -> HELLO

# ข้อ 20-25: โจทย์เพิ่มเติม
# ข้อ 20: สร้าง DSL สำหรับ validation rules
# ข้อ 21: Implement curry ด้วย closures
# ข้อ 22: สร้าง retry mechanism ด้วย exponential backoff
# ข้อ 23: สร้าง Observable pattern ด้วย method hooks
# ข้อ 24: Implement pipe operator (|>) ด้วย method chaining
# ข้อ 25: สร้าง Method introspection tool
```

---

## สรุป Methods ใน Ruby

| Pattern | การใช้งาน |
|---------|----------|
| `def method(req)` | required parameter |
| `def method(opt = val)` | optional parameter |
| `def method(*args)` | variable positional |
| `def method(key:)` | required keyword |
| `def method(key: val)` | optional keyword |
| `def method(**opts)` | variable keyword |
| `def method(&blk)` | explicit block |
| `method!` | bang (mutating) |
| `method?` | predicate (boolean) |
| `alias_method :new, :old` | method alias |
| `method(:name)` | method as object |
| `@var \|\|= value` | memoization |

**Best Practices:**
1. ชื่อ method ใช้ snake_case
2. ใช้ keyword arguments เพื่อความชัดเจน
3. Guard clauses แทน deep nesting
4. Bang method ควรมี non-bang version ด้วย
5. Memoize ผลการคำนวณที่ expensive
6. Private method สำหรับ implementation details
7. Method ควรทำสิ่งเดียว (Single Responsibility)

> ⬅️ [ตอนที่ 8: Loops](part-08-loops.md) | ➡️ [ตอนที่ 10: Blocks, Procs, Lambdas](part-10-blocks-procs-lambdas.md)
