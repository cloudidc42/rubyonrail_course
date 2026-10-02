# ตอนที่ 7: Control Flow (ขั้นตอนที่ 101-120)

## บทนำ

Control Flow คือการควบคุมทิศทางการทำงานของโปรแกรม Ruby มีเครื่องมือหลากหลายในการตัดสินใจและแยกทิศทางการทำงาน ตั้งแต่ `if/else` พื้นฐาน ไปจนถึง Pattern Matching ที่ทรงพลังใน Ruby 3+

ในบทนี้เราจะเรียนรู้:
- if/elsif/else
- unless
- Ternary operator
- case/when
- Inline conditions
- Guard clauses
- Logical operators
- Short-circuit evaluation
- Truthiness ใน Ruby
- Flip-flop operator
- Pattern matching (Ruby 3+)

---

## ขั้นตอนที่ 101: if/elsif/else - พื้นฐาน

```ruby
# โครงสร้างพื้นฐาน
age = 25

if age >= 18
  puts "ผู้ใหญ่"
end

# if-else
score = 75

if score >= 60
  puts "ผ่าน"
else
  puts "ไม่ผ่าน"
end

# if-elsif-else
grade = 85

if grade >= 90
  puts "A"
elsif grade >= 80
  puts "B"
elsif grade >= 70
  puts "C"
elsif grade >= 60
  puts "D"
else
  puts "F"
end

# การใช้ then (optional แต่ไม่แนะนำ)
if age >= 18 then puts "ผู้ใหญ่" end
```

---

## ขั้นตอนที่ 102: if เป็น Expression ที่คืนค่า

ใน Ruby, `if` คือ expression ที่คืนค่าได้ ไม่ใช่แค่ statement

```ruby
# if expression คืนค่า
age = 25
status = if age >= 18
  "ผู้ใหญ่"
else
  "เด็ก"
end
puts status   # => ผู้ใหญ่

# กำหนดค่าจาก if
message = if score >= 90
  "ยอดเยี่ยม"
elsif score >= 70
  "ดี"
else
  "ต้องพัฒนา"
end

# ถ้า branch ที่ทำงานไม่มี explicit value
result = if false
  "ไม่ถึง"
end
puts result.inspect   # => nil

# ใช้กับ method ที่คืนค่า
def classify(n)
  if n > 0
    "บวก"
  elsif n < 0
    "ลบ"
  else
    "ศูนย์"
  end
end

puts classify(5)    # => บวก
puts classify(-3)   # => ลบ
puts classify(0)    # => ศูนย์
```

---

## ขั้นตอนที่ 103: unless - ตรงข้ามกับ if

`unless condition` เหมือนกับ `if !condition` แต่อ่านง่ายกว่า

```ruby
logged_in = false

# unless = if not
unless logged_in
  puts "กรุณาเข้าสู่ระบบ"
end

# เปรียบเทียบกับ if
if !logged_in
  puts "กรุณาเข้าสู่ระบบ"
end

# unless with else
unless logged_in
  puts "กรุณาเข้าสู่ระบบ"
else
  puts "ยินดีต้อนรับ!"
end

# Inline unless
puts "กรุณาเข้าสู่ระบบ" unless logged_in
redirect_to_home unless logged_in  # Rails pattern

# หลีกเลี่ยงการใช้ unless กับ complex conditions
# ไม่ดี - อ่านยาก
unless !logged_in && age >= 18
  puts "ผ่านการตรวจสอบ"
end

# ดีกว่า
if logged_in || age < 18
  # ...
end

# ไม่ควรใช้ unless กับ else (สับสน)
# หลีกเลี่ยง:
unless condition
  # ทำถ้า false
else
  # ทำถ้า true  # อ่านยากมาก!
end
```

---

## ขั้นตอนที่ 104: Ternary Operator

สำหรับ conditions ง่าย ๆ บนบรรทัดเดียว

```ruby
# syntax: condition ? true_value : false_value
age = 20
status = age >= 18 ? "ผู้ใหญ่" : "เด็ก"
puts status   # => ผู้ใหญ่

# ใช้ใน string interpolation
name = "Alice"
greeting = "สวัสดี #{name.length > 5 ? "คุณ" : ""}#{name}!"
puts greeting   # => สวัสดี Alice!

# ใน method call
def discount_price(price, member)
  price * (member ? 0.9 : 1.0)
end

puts discount_price(1000, true)   # => 900.0
puts discount_price(1000, false)  # => 1000.0

# ซ้อนกัน (แต่หลีกเลี่ยง!)
score = 85
grade = score >= 90 ? "A" : score >= 80 ? "B" : score >= 70 ? "C" : "F"
puts grade   # => B
# ดีกว่าถ้าใช้ case/when แทน

# ใช้กับ puts
puts (temperature > 30 ? "ร้อน" : "เย็น")

# เมื่อไรควรใช้ Ternary?
# - เมื่อ condition ง่ายมาก
# - เมื่อ ternary ทำให้โค้ดอ่านง่ายขึ้น
# - เมื่อใช้ใน string interpolation หรือ argument

# เมื่อไรไม่ควรใช้?
# - เมื่อ condition ซับซ้อน
# - เมื่อ true/false branch ยาว
```

---

## ขั้นตอนที่ 105: case/when - Pattern Matching แบบ Classic

```ruby
# Basic case/when
day = "Monday"

case day
when "Monday", "Tuesday", "Wednesday", "Thursday", "Friday"
  puts "วันทำงาน"
when "Saturday", "Sunday"
  puts "วันหยุด"
else
  puts "ไม่รู้จัก"
end

# case กับ Range
score = 85

grade = case score
when 90..100
  "A"
when 80..89
  "B"
when 70..79
  "C"
when 60..69
  "D"
else
  "F"
end

puts grade   # => B

# case กับ Regex
input = "user@example.com"

case input
when /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  puts "เป็น Email"
when /\A\d{10}\z/
  puts "เป็นหมายเลขโทรศัพท์"
else
  puts "ไม่รู้จักรูปแบบ"
end

# case กับ Class
def describe(value)
  case value
  when String
    puts "เป็น String: #{value}"
  when Integer
    puts "เป็น Integer: #{value}"
  when Array
    puts "เป็น Array ที่มี #{value.length} elements"
  when Hash
    puts "เป็น Hash ที่มี #{value.size} pairs"
  when NilClass
    puts "เป็น nil"
  else
    puts "ไม่รู้จักประเภท: #{value.class}"
  end
end

describe("hello")   # => เป็น String: hello
describe(42)        # => เป็น Integer: 42
describe([1,2,3])   # => เป็น Array ที่มี 3 elements
describe(nil)       # => เป็น nil
```

---

## ขั้นตอนที่ 106: case/when ที่ซับซ้อนกว่า

```ruby
# case โดยไม่มี expression (เหมือน if-elsif)
temperature = 35

case
when temperature > 35
  puts "ร้อนมาก"
when temperature > 28
  puts "ร้อน"
when temperature > 20
  puts "อากาศดี"
when temperature > 10
  puts "เย็น"
else
  puts "หนาว"
end

# case กับ Proc/Lambda
valid_email = ->(s) { s.include?("@") }
valid_phone = ->(s) { s.match?(/^\d{10}$/) }

contact = "0812345678"

case contact
when valid_email
  puts "Email"
when valid_phone
  puts "โทรศัพท์"   # => โทรศัพท์
end

# case คืนค่า
def http_status_message(code)
  case code
  when 200 then "OK"
  when 201 then "Created"
  when 400 then "Bad Request"
  when 401 then "Unauthorized"
  when 403 then "Forbidden"
  when 404 then "Not Found"
  when 500 then "Internal Server Error"
  else "Unknown Status"
  end
end

puts http_status_message(404)   # => Not Found
puts http_status_message(200)   # => OK

# case กับ then บน single line
status = 200
message = case status
  when 200 then "Success"
  when 404 then "Not Found"
  when 500 then "Server Error"
  else "Unknown"
end
puts message
```

---

## ขั้นตอนที่ 107: Inline Conditions (One-liners)

```ruby
# Inline if
age = 20
puts "ผู้ใหญ่" if age >= 18
return if user.nil?  # ใช้บ่อยใน methods

# Inline unless
puts "ยังไม่เข้าสู่ระบบ" unless logged_in

# เหมาะกับ:
# 1. Guard clauses
raise ArgumentError, "Name required" if name.nil?
return [] if items.empty?

# 2. Simple conditions
x += 1 if condition
arr << item unless arr.include?(item)

# 3. ใน loop
items.each { |item| puts item if item.positive? }

# Inline while / until (ไม่ค่อยใช้)
count = 0
count += 1 while count < 5
puts count   # => 5

count = 10
count -= 1 until count <= 5
puts count   # => 5

# begin/end while (do-while equivalent)
count = 0
begin
  puts count
  count += 1
end while count < 3
# 0, 1, 2

# begin/end until
count = 0
begin
  puts count
  count += 1
end until count >= 3
# 0, 1, 2
```

---

## ขั้นตอนที่ 108: Guard Clauses

Guard clauses เป็น pattern ที่ดีในการลด nesting และทำให้โค้ดอ่านง่าย

```ruby
# ไม่ดี - nested conditions
def process_order(order)
  if order
    if order[:items]
      if order[:items].any?
        if order[:total] > 0
          # process order
          puts "Processing order..."
        else
          puts "Total must be positive"
        end
      else
        puts "Cart is empty"
      end
    else
      puts "No items"
    end
  else
    puts "No order"
  end
end

# ดีกว่า - Guard clauses
def process_order(order)
  return "No order" unless order
  return "No items" unless order[:items]
  return "Cart is empty" unless order[:items].any?
  return "Total must be positive" unless order[:total] > 0

  # ตอนนี้เราแน่ใจว่าทุกอย่างถูกต้อง
  puts "Processing order..."
  "Order processed"
end

# อีกตัวอย่าง
def calculate_discount(user, product)
  return 0 unless user
  return 0 unless user[:member]
  return 0 unless product[:discountable]
  return 0 if product[:price] < 100

  # คำนวณ discount
  product[:price] * 0.1
end

# Guard clauses ใน Rails
def create
  @user = User.new(user_params)
  return render :new, status: :unprocessable_entity unless @user.valid?
  
  @user.save
  redirect_to @user, notice: "สร้าง user สำเร็จ"
end
```

---

## ขั้นตอนที่ 109: Logical Operators - && และ ||

```ruby
# && (and) - ทั้งคู่ต้องเป็น true
puts true && true    # => true
puts true && false   # => false
puts false && true   # => false
puts false && false  # => false

# || (or) - อย่างน้อยหนึ่งเป็น true
puts true || true    # => true
puts true || false   # => true
puts false || true   # => true
puts false || false  # => false

# ! (not)
puts !true    # => false
puts !false   # => true
puts !nil     # => true

# ใน conditions
age = 25
has_id = true

if age >= 18 && has_id
  puts "เข้าได้"
end

member = false
vip = true

if member || vip
  puts "รับสิทธิพิเศษ"
end

# && และ || return actual values (ไม่ใช่แค่ true/false)
puts (nil && "hello")     # => nil (คืน left เพราะ falsy)
puts (false && "hello")   # => false
puts ("world" && "hello") # => hello (คืน right เพราะ left truthy)

puts (nil || "default")   # => default (คืน right เพราะ left falsy)
puts (false || "value")   # => value
puts ("first" || "second") # => first (คืน left เพราะ truthy)

# ใช้ || สำหรับ default values
name = params[:name] || "Anonymous"
config = user_config || DEFAULT_CONFIG
```

---

## ขั้นตอนที่ 110: and, or, not - Low Precedence Operators

```ruby
# and, or, not มี precedence ต่ำกว่า &&, ||, !
# ใช้เป็น flow control ไม่ใช่ boolean operations

# and - เหมือน && แต่ precedence ต่ำกว่า
x = true and false    # x = true (เพราะ = มี precedence สูงกว่า and)
x = true && false     # x = false (เพราะ && มี precedence สูงกว่า =)

puts x   # แสดงให้เห็นความแตกต่าง

# การใช้ and สำหรับ flow control
result = find_user(id) and return result
# เหมือน:
result = find_user(id)
return result if result

# การใช้ or สำหรับ fallback
record = find_in_cache(id) or record = find_in_db(id)
# หรือ
user = User.find_by(id: id) or raise "User not found"

# not
not true    # => false
not false   # => true

# ตัวอย่างการใช้ not ที่สร้างสับสน
# x = not true  # SyntaxError
x = (not true)  # ต้องมี parentheses

# Recommendation:
# ใช้ && || ! สำหรับ boolean operations
# ใช้ and or not สำหรับ flow control (เฉพาะกรณีที่ชัดเจน)
```

---

## ขั้นตอนที่ 111: Short-Circuit Evaluation

```ruby
# && short-circuits: ถ้า left เป็น falsy จะไม่ evaluate right
def expensive_check
  puts "กำลังตรวจสอบ..."
  true
end

false && expensive_check   # "กำลังตรวจสอบ..." ไม่แสดง
nil && expensive_check     # ไม่ evaluate

# || short-circuits: ถ้า left เป็น truthy จะไม่ evaluate right
true || expensive_check    # "กำลังตรวจสอบ..." ไม่แสดง
"value" || expensive_check # ไม่ evaluate

# ประโยชน์ของ Short-circuit

# 1. ป้องกัน nil errors
user = nil
# user.name  # NoMethodError!
user && user.name  # => nil (ปลอดภัย)
user&.name  # Safe navigation operator (Ruby 2.3+)

# 2. Default values
def greet(name = nil)
  name ||= "Anonymous"
  puts "สวัสดี #{name}!"
end

greet("Alice")  # => สวัสดี Alice!
greet           # => สวัสดี Anonymous!

# 3. Conditional execution
config[:debug] && puts("Debug mode on")
admin? && send_notification

# 4. ||= operator (assign unless truthy)
@cache ||= {}            # initialize เฉพาะครั้งแรก
result ||= compute_result  # memoization

# 5. &&= operator (assign only if truthy)
user &&= user.update(name: "New Name")  # เฉพาะถ้า user ไม่ nil

# ตัวอย่างจริง
def find_user_email(user_id)
  user = User.find_by(id: user_id)
  user && user.email
  # หรือ: user&.email
end
```

---

## ขั้นตอนที่ 112: Truthiness ใน Ruby

ใน Ruby มีเพียง `nil` และ `false` เท่านั้นที่เป็น falsy ทุกอย่างอื่นเป็น truthy

```ruby
# Falsy values ใน Ruby (มีแค่ 2 อย่าง!)
puts "nil is falsy" unless nil     # => nil is falsy
puts "false is falsy" unless false # => false is falsy

# Truthy values (ทุกอย่างที่ไม่ใช่ nil และ false)
puts "0 is truthy" if 0           # => 0 is truthy (ต่างจาก JavaScript!)
puts "'' is truthy" if ""         # => '' is truthy (ต่างจาก Python!)
puts "[] is truthy" if []         # => [] is truthy
puts "{} is truthy" if {}         # => {} is truthy

# ตัวอย่างที่อาจสร้างความสับสน
value = 0
if value   # 0 เป็น truthy ใน Ruby!
  puts "0 เป็น truthy"
end

empty_string = ""
if empty_string  # "" เป็น truthy ใน Ruby!
  puts "String ว่างเป็น truthy"
end

# การตรวจสอบที่ถูกต้อง
puts value.zero?                # => true
puts empty_string.empty?        # => true
puts [].empty?                  # => true
puts {}.empty?                  # => true
puts nil.nil?                   # => true

# !! (double bang) - แปลงเป็น boolean
puts !!nil    # => false
puts !!false  # => false
puts !!0      # => true
puts !!""     # => true
puts !![]     # => true

# เปรียบเทียบกับภาษาอื่น
# Python: 0, "", [], {} เป็น falsy
# JavaScript: 0, "", null, undefined, NaN เป็น falsy
# Ruby: nil, false เท่านั้น!
```

---

## ขั้นตอนที่ 113: Comparison Operators

```ruby
# Equality
puts 1 == 1      # => true
puts 1 == "1"    # => false (ต่างประเภท)
puts 1.eql?(1)   # => true (type-strict)
puts 1.eql?(1.0) # => false (ต่างประเภท)
puts 1.equal?(1) # => true (same object)

# Comparison
puts 1 < 2    # => true
puts 2 > 1    # => true
puts 1 <= 1   # => true
puts 1 >= 2   # => false
puts 1 != 2   # => true

# Spaceship operator <=>
puts 1 <=> 2   # => -1 (น้อยกว่า)
puts 2 <=> 2   # => 0 (เท่ากัน)
puts 3 <=> 2   # => 1 (มากกว่า)
puts 1 <=> "a" # => nil (ไม่สามารถเปรียบเทียบได้)

# ใช้ <=> กับ sort
names = ["Charlie", "Alice", "Bob"]
puts names.sort { |a, b| a <=> b }.inspect
# => ["Alice", "Bob", "Charlie"]

# ใช้ sort_by แทน (สะดวกกว่า)
puts names.sort_by { |name| name }.inspect

# === operator (used in case/when)
puts (1..10) === 5      # => true (range membership)
puts String === "hello" # => true (type check)
puts /\d+/ === "abc123" # => true (regex match)
puts :foo === :foo      # => true

# Comparable module
class Temperature
  include Comparable
  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  def <=>(other)
    degrees <=> other.degrees
  end
end

temps = [Temperature.new(30), Temperature.new(25), Temperature.new(35)]
puts temps.sort.map(&:degrees).inspect   # => [25, 30, 35]
puts temps.max.degrees   # => 35
```

---

## ขั้นตอนที่ 114: Safe Navigation Operator (&.)

Ruby 2.3 เพิ่ม `&.` (safe navigation operator หรือ "lonely operator")

```ruby
# ปัญหาก่อน Ruby 2.3
user = nil
# user.name  # NoMethodError!

# วิธีเดิม
puts user && user.name  # => nil (ปลอดภัย แต่ verbose)

# Safe Navigation Operator
puts user&.name   # => nil (เรียบง่าย)

# ตัวอย่างใช้งาน
class User
  attr_accessor :name, :address

  def initialize(name, address = nil)
    @name = name
    @address = address
  end
end

class Address
  attr_accessor :city

  def initialize(city)
    @city = city
  end
end

alice = User.new("Alice", Address.new("Bangkok"))
bob = User.new("Bob")

puts alice&.name              # => Alice
puts alice&.address&.city     # => Bangkok
puts bob&.name                # => Bob
puts bob&.address&.city       # => nil (ไม่ error!)

# ใน chaining
users = [alice, nil, bob, nil]
names = users.map { |u| u&.name }
puts names.inspect   # => ["Alice", nil, "Bob", nil]

# กับ compact
names_clean = users.map { |u| u&.name }.compact
puts names_clean.inspect   # => ["Alice", "Bob"]

# เปรียบเทียบกับ dig
data = { user: { name: "Alice" } }
puts data.dig(:user, :name)    # => Alice
puts data.dig(:user, :email)   # => nil
```

---

## ขั้นตอนที่ 115: Conditional Assignment Operators

```ruby
# ||= (or-assign) - assign ถ้า current value เป็น nil หรือ false
cache = nil
cache ||= {}
puts cache.inspect   # => {}

cache ||= { key: "value" }  # ไม่ assign อีกแล้วเพราะ {} truthy
puts cache.inspect   # => {}

# ใช้บ่อยสำหรับ memoization
def expensive_calculation
  @result ||= begin
    # คำนวณที่ใช้เวลานาน
    sleep(1)
    42
  end
end

# ครั้งแรก: คำนวณ
# ครั้งถัดไป: ใช้ค่าที่ cache ไว้

# &&= (and-assign) - assign ถ้า current value เป็น truthy
name = "Alice"
name &&= name.upcase
puts name   # => ALICE

name = nil
name &&= name.upcase  # ไม่ทำอะไรเพราะ nil เป็น falsy
puts name.inspect   # => nil

# ใช้สำหรับ conditional transformation
user = { name: "alice", age: 25 }
user[:name] &&= user[:name].capitalize
puts user[:name]   # => Alice

# ||= กับ Hash defaults
scores = {}
scores[:alice] ||= 0
scores[:alice] += 10
scores[:bob] ||= 0
scores[:bob] += 20
puts scores.inspect   # => {:alice=>10, :bob=>20}
```

---

## ขั้นตอนที่ 116: Flip-Flop Operator (Advanced)

Flip-flop เป็น operator ที่ไม่ค่อยถูกใช้ แต่มีประโยชน์ในบางกรณี

```ruby
# Flip-flop ใช้ .. หรือ ... ใน if condition
# เปิดใช้เมื่อ left condition เป็น true
# ปิดใช้เมื่อ right condition เป็น true

# ตัวอย่าง: แสดง lines ระหว่าง "START" และ "END"
lines = [
  "header",
  "START",
  "line 1",
  "line 2",
  "END",
  "footer"
]

lines.each do |line|
  if (line == "START")..(line == "END")
    puts line
  end
end
# => START
# => line 1
# => line 2
# => END

# .. ทดสอบ end condition หลัง flip เป็น true
# ... ไม่ทดสอบ end condition จนกว่าจะถึง iteration ถัดไป

lines.each do |line|
  if (line == "START")...(line == "END")
    puts line
  end
end
# => START
# => line 1
# => line 2
# (ไม่มี END เพราะใช้ ...)

# ตัวอย่างใช้ $. (line number)
# มักใช้กับ STDIN
# STDIN.each_line { |line| puts line if ($. == 3)..($. == 7) }

# ปัจจุบัน flip-flop เป็น deprecated ใน Ruby 2.6
# และถูก un-deprecated ใน Ruby 2.7
# แต่ยังไม่ค่อยแนะนำให้ใช้ในโค้ด production
```

---

## ขั้นตอนที่ 117: Pattern Matching - Basic (Ruby 2.7+)

Pattern matching เป็น feature ที่ทรงพลังมากใน Ruby 3+

```ruby
# ใช้ case/in แทน case/when
data = { name: "Alice", age: 30, role: "admin" }

case data
in { name: String => name, role: "admin" }
  puts "Admin: #{name}"
in { name: String => name, role: "user" }
  puts "User: #{name}"
in { name: String => name }
  puts "Unknown role: #{name}"
end
# => Admin: Alice

# Pattern matching กับ Array
numbers = [1, 2, 3, 4, 5]

case numbers
in [Integer => first, *rest]
  puts "First: #{first}, Rest: #{rest.inspect}"
end
# => First: 1, Rest: [2, 3, 4, 5]

# Pattern matching กับ types
def process(value)
  case value
  in Integer => n if n > 0
    puts "บวก Integer: #{n}"
  in Integer => n
    puts "ลบหรือศูนย์ Integer: #{n}"
  in String => s if s.empty?
    puts "String ว่าง"
  in String => s
    puts "String: #{s}"
  in nil
    puts "nil"
  end
end

process(42)     # => บวก Integer: 42
process(-5)     # => ลบหรือศูนย์ Integer: -5
process("hi")   # => String: hi
process(nil)    # => nil
```

---

## ขั้นตอนที่ 118: Pattern Matching - Advanced (Ruby 3.0+)

```ruby
# Find pattern - หา pattern ใน array
numbers = [1, 2, "three", 4, 5]

case numbers
in [*, String => s, *]
  puts "พบ String: #{s}"
end
# => พบ String: three

# Deconstruct keys (Hash patterns)
user = { name: "Alice", age: 30, address: { city: "Bangkok" } }

case user
in { name:, address: { city: } }
  puts "#{name} อยู่ที่ #{city}"
end
# => Alice อยู่ที่ Bangkok

# Pin operator (^) - ใช้ค่าที่มีอยู่แล้ว
expected = 42
result = 42

case result
in ^expected
  puts "ตรงกับที่คาดหวัง!"
end

# Rightward assignment (Ruby 3.0+)
# value => pattern
{ name: "Alice", age: 30 } => { name:, age: }
puts name   # => Alice
puts age    # => 30

# One-line pattern matching (Ruby 3.0+)
# in แทน case..in สำหรับ single pattern
{ name: "Bob", role: "admin" } in { name:, role: "admin" }
puts name if $~   # ตรวจสอบว่า match สำเร็จ

# Pattern matching กับ class deconstruct
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
  end

  def deconstruct
    [@x, @y]  # สำหรับ Array pattern
  end

  def deconstruct_keys(keys)
    { x: @x, y: @y }  # สำหรับ Hash pattern
  end
end

point = Point.new(1, 2)

case point
in [x, y]
  puts "Array: (#{x}, #{y})"
end

case point
in { x:, y: }
  puts "Hash: x=#{x}, y=#{y}"
end
```

---

## ขั้นตอนที่ 119: Pattern Matching - Guard Clauses และ Error Handling

```ruby
# Guard clauses ใน pattern matching (if)
scores = [85, 92, 67, 98, 75]

scores.each do |score|
  case score
  in Integer => n if n >= 90
    puts "#{n}: เกรด A"
  in Integer => n if n >= 80
    puts "#{n}: เกรด B"
  in Integer => n if n >= 70
    puts "#{n}: เกรด C"
  in Integer => n
    puts "#{n}: เกรด F"
  end
end

# NoMatchingPatternError
begin
  case 42
  in String
    puts "String"
  end
rescue NoMatchingPatternError => e
  puts "ไม่ match: #{e.message}"
end

# in? หรือ case ที่ไม่ raise (Ruby 3.0+)
data = { type: "unknown" }
if data in { type: "admin" | "superadmin" }
  puts "สิทธิ์สูง"
else
  puts "สิทธิ์ปกติ"
end

# Pattern matching กับ Response objects
def parse_response(response)
  case response
  in { status: 200, body: String => body }
    { success: true, data: body }
  in { status: 404 }
    { success: false, error: "Not found" }
  in { status: 500, error: String => msg }
    { success: false, error: "Server error: #{msg}" }
  in { status: Integer => code }
    { success: false, error: "HTTP #{code}" }
  end
end

puts parse_response({ status: 200, body: "OK" }).inspect
# => {:success=>true, :data=>"OK"}

puts parse_response({ status: 404 }).inspect
# => {:success=>false, :error=>"Not found"}
```

---

## ขั้นตอนที่ 120: Control Flow Best Practices

```ruby
# 1. ใช้ Guard clauses แทน nested if
# ไม่ดี
def process(user)
  if user
    if user.active?
      if user.has_permission?(:admin)
        do_admin_work
      end
    end
  end
end

# ดีกว่า
def process(user)
  return unless user
  return unless user.active?
  return unless user.has_permission?(:admin)

  do_admin_work
end

# 2. ใช้ case/when แทน if-elsif ยาว ๆ
# ไม่ดี
def categorize(value)
  if value.is_a?(String)
    "string"
  elsif value.is_a?(Integer)
    "integer"
  elsif value.is_a?(Float)
    "float"
  elsif value.is_a?(Array)
    "array"
  else
    "other"
  end
end

# ดีกว่า
def categorize(value)
  case value
  when String then "string"
  when Integer then "integer"
  when Float then "float"
  when Array then "array"
  else "other"
  end
end

# 3. ใช้ &&= และ ||= อย่างเหมาะสม
# สำหรับ memoization
def user_name
  @user_name ||= fetch_user_name
end

# 4. ใช้ Safe Navigation Operator
# แทน: user && user.profile && user.profile.avatar
user&.profile&.avatar

# 5. ระวัง unless กับ complex condition
# ไม่ดี
unless !active && !admin
  process
end

# ดีกว่า
if active || admin
  process
end

# 6. Ternary สำหรับ simple assignment
label = count == 1 ? "item" : "items"

# ไม่ใช้ ternary สำหรับ side effects
# ไม่ดี
condition ? puts("yes") : puts("no")

# ดีกว่า
if condition
  puts "yes"
else
  puts "no"
end

# 7. Pattern matching สำหรับ complex data destructuring
def handle_event(event)
  case event
  in { type: "user.created", data: { name:, email: } }
    welcome_user(name, email)
  in { type: "order.placed", data: { order_id:, total: } }
    process_payment(order_id, total)
  in { type: "error", message: }
    log_error(message)
  end
end
```

---

## แบบฝึกหัด (ขั้นตอนที่ 101-120)

### ข้อที่ 1: BMI Calculator
คำนวณ BMI และแสดงผลการประเมิน

```ruby
# เฉลย
def calculate_bmi(weight_kg, height_m)
  bmi = weight_kg / (height_m ** 2)
  
  category = case bmi
  when 0...18.5 then "น้ำหนักต่ำกว่าเกณฑ์"
  when 18.5...25 then "น้ำหนักปกติ"
  when 25...30 then "น้ำหนักเกิน"
  else "อ้วน"
  end
  
  { bmi: bmi.round(2), category: category }
end

result = calculate_bmi(70, 1.75)
puts "BMI: #{result[:bmi]} - #{result[:category]}"
```

### ข้อที่ 2: FizzBuzz
ปัญหา FizzBuzz คลาสสิก

```ruby
# เฉลย
(1..30).each do |n|
  result = case
  when (n % 15).zero? then "FizzBuzz"
  when (n % 3).zero? then "Fizz"
  when (n % 5).zero? then "Buzz"
  else n.to_s
  end
  print "#{result} "
end
puts
```

### ข้อที่ 3: Temperature Converter
แปลงอุณหภูมิระหว่าง Celsius, Fahrenheit, Kelvin

```ruby
# เฉลย
def convert_temperature(value, from:, to:)
  celsius = case from
  when :celsius then value
  when :fahrenheit then (value - 32) * 5.0 / 9
  when :kelvin then value - 273.15
  end

  case to
  when :celsius then celsius.round(2)
  when :fahrenheit then (celsius * 9.0 / 5 + 32).round(2)
  when :kelvin then (celsius + 273.15).round(2)
  end
end

puts convert_temperature(100, from: :celsius, to: :fahrenheit)   # => 212.0
puts convert_temperature(32, from: :fahrenheit, to: :celsius)    # => 0.0
puts convert_temperature(0, from: :celsius, to: :kelvin)         # => 273.15
```

### ข้อที่ 4: Day Checker
ตรวจสอบว่าวันนั้นเป็นวันทำงาน วันหยุด หรืออื่น ๆ

```ruby
# เฉลย
WEEKDAYS = %w[Monday Tuesday Wednesday Thursday Friday]
WEEKENDS = %w[Saturday Sunday]

def day_type(day)
  case day.capitalize
  when *WEEKDAYS
    "วันทำงาน"
  when *WEEKENDS
    "วันหยุดสุดสัปดาห์"
  else
    "ไม่รู้จักวัน"
  end
end

puts day_type("Monday")   # => วันทำงาน
puts day_type("sunday")   # => วันหยุดสุดสัปดาห์
puts day_type("Holiday")  # => ไม่รู้จักวัน
```

### ข้อที่ 5: Input Validator
ตรวจสอบความถูกต้องของ input

```ruby
# เฉลย
def validate_input(value, type:)
  case type
  when :email
    value.match?(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
  when :phone
    value.match?(/\A\d{10}\z/)
  when :age
    value.is_a?(Integer) && (0..120).include?(value)
  when :name
    value.is_a?(String) && !value.empty? && value.length <= 100
  else
    false
  end
end

puts validate_input("user@example.com", type: :email)  # => true
puts validate_input("invalid", type: :email)            # => false
puts validate_input("0812345678", type: :phone)         # => true
puts validate_input(25, type: :age)                     # => true
```

### ข้อที่ 6: Safe User Access
เขียน method ที่ safely access nested user data

```ruby
# เฉลย
def user_city(user)
  user&.dig(:profile, :address, :city) || "ไม่ระบุ"
end

users = [
  { name: "Alice", profile: { address: { city: "Bangkok" } } },
  { name: "Bob", profile: { address: {} } },
  { name: "Carol", profile: {} },
  nil
]

users.each do |user|
  name = user&.dig(:name) || "Unknown"
  city = user_city(user)
  puts "#{name}: #{city}"
end
```

### ข้อที่ 7: Roman Numerals
แปลตัวเลข 1-3999 เป็น Roman Numerals

```ruby
# เฉลย
def to_roman(number)
  return "Invalid" unless (1..3999).include?(number)

  numerals = {
    1000 => "M", 900 => "CM", 500 => "D", 400 => "CD",
    100 => "C", 90 => "XC", 50 => "L", 40 => "XL",
    10 => "X", 9 => "IX", 5 => "V", 4 => "IV", 1 => "I"
  }

  result = ""
  numerals.each do |value, numeral|
    while number >= value
      result += numeral
      number -= value
    end
  end
  result
end

puts to_roman(1)     # => I
puts to_roman(4)     # => IV
puts to_roman(9)     # => IX
puts to_roman(2024)  # => MMXXIV
```

### ข้อที่ 8: Ticket Price
คำนวณราคาตั๋วตามอายุและประเภท

```ruby
# เฉลย
def ticket_price(age:, type: :regular)
  base = case type
  when :regular then 300
  when :vip then 500
  when :premium then 800
  else raise ArgumentError, "ประเภทตั๋วไม่ถูกต้อง"
  end

  discount = case age
  when 0..3 then 1.0  # ฟรี
  when 4..12 then 0.5  # เด็ก 50%
  when 13..17 then 0.75 # วัยรุ่น 25%
  when 60.. then 0.8   # ผู้สูงอายุ 20%
  else 1.0             # ราคาเต็ม
  end

  (base * discount).round
end

puts ticket_price(age: 5, type: :regular)    # => 150
puts ticket_price(age: 25, type: :vip)       # => 500
puts ticket_price(age: 65, type: :premium)   # => 640
```

### ข้อที่ 9: Login System
จำลอง login system ด้วย guard clauses

```ruby
# เฉลย
class LoginSystem
  MAX_ATTEMPTS = 3
  
  def initialize
    @users = { "admin" => "password123", "user1" => "pass456" }
    @attempts = Hash.new(0)
    @locked = []
  end

  def login(username, password)
    return "บัญชีถูกล็อก" if @locked.include?(username)
    return "ไม่พบผู้ใช้" unless @users.key?(username)
    
    if @users[username] == password
      @attempts.delete(username)
      "เข้าสู่ระบบสำเร็จ"
    else
      @attempts[username] += 1
      if @attempts[username] >= MAX_ATTEMPTS
        @locked << username
        "บัญชีถูกล็อก (เกินจำนวนครั้งที่กำหนด)"
      else
        "รหัสผ่านไม่ถูกต้อง (#{MAX_ATTEMPTS - @attempts[username]} ครั้งที่เหลือ)"
      end
    end
  end
end

system = LoginSystem.new
puts system.login("admin", "wrong")       # => รหัสผ่านไม่ถูกต้อง (2 ครั้งที่เหลือ)
puts system.login("admin", "wrong")       # => รหัสผ่านไม่ถูกต้อง (1 ครั้งที่เหลือ)
puts system.login("admin", "wrong")       # => บัญชีถูกล็อก
puts system.login("admin", "password123") # => บัญชีถูกล็อก
```

### ข้อที่ 10: Pattern Matching JSON
ใช้ pattern matching เพื่อ parse JSON data

```ruby
# เฉลย
def process_api_response(response)
  case response
  in { status: "success", data: { users: [*, { name:, email: }, *] } }
    puts "พบ user: #{name} (#{email})"
  in { status: "success", data: Hash => data }
    puts "ข้อมูล: #{data.inspect}"
  in { status: "error", message: String => msg }
    puts "Error: #{msg}"
  in { status: "error" }
    puts "Error ไม่มีรายละเอียด"
  else
    puts "Response ไม่รู้จัก"
  end
end

process_api_response({
  status: "success",
  data: { users: [{ name: "Alice", email: "a@example.com" }] }
})

process_api_response({
  status: "error",
  message: "Unauthorized"
})
```

### ข้อที่ 11-20: แบบฝึกหัดเพิ่มเติม

**ข้อที่ 11:** เขียน method `classify_number` ที่รับตัวเลขและบอกว่าเป็นจำนวนบวก/ลบ/ศูนย์/คี่/คู่/ไพรม์

```ruby
def classify_number(n)
  traits = []
  traits << (n > 0 ? "บวก" : n < 0 ? "ลบ" : "ศูนย์")
  traits << (n.even? ? "คู่" : "คี่") unless n.zero?
  traits << "จำนวนเฉพาะ" if prime?(n)
  traits.join(", ")
end

def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n)).none? { |i| n % i == 0 }
end

puts classify_number(7)    # => บวก, คี่, จำนวนเฉพาะ
puts classify_number(10)   # => บวก, คู่
puts classify_number(-3)   # => ลบ, คี่
```

**ข้อที่ 12:** เขียน method `season` ที่รับเดือน (1-12) และคืนค่า season

```ruby
def season(month)
  case month
  when 3..5 then "ฤดูร้อน"
  when 6..8 then "ฤดูฝน"
  when 9..11 then "ฤดูหนาว"
  when 12, 1..2 then "ฤดูหนาวกลาง"
  else "เดือนไม่ถูกต้อง"
  end
end
```

**ข้อที่ 13:** Short-circuit evaluation เพื่อ validate form

```ruby
def validate_form(name:, email:, age:)
  errors = []
  errors << "ชื่อจำเป็น" if name.nil? || name.empty?
  errors << "Email ไม่ถูกต้อง" unless email&.include?("@")
  errors << "อายุต้องเป็นตัวเลข" unless age.is_a?(Integer)
  errors << "อายุต้อง 0-120" if age.is_a?(Integer) && !(0..120).include?(age)
  errors.empty? ? { valid: true } : { valid: false, errors: errors }
end
```

**ข้อที่ 14:** Simple calculator ด้วย case/when

```ruby
def calculate(a, operator, b)
  case operator
  when "+" then a + b
  when "-" then a - b
  when "*" then a * b
  when "/" then b.zero? ? "หารด้วยศูนย์ไม่ได้" : a.to_f / b
  when "**" then a ** b
  when "%" then b.zero? ? "หารด้วยศูนย์ไม่ได้" : a % b
  else "ไม่รู้จัก operator"
  end
end

puts calculate(10, "+", 5)    # => 15
puts calculate(10, "/", 0)    # => หารด้วยศูนย์ไม่ได้
puts calculate(2, "**", 10)   # => 1024
```

**ข้อที่ 15:** Traffic light simulation

```ruby
class TrafficLight
  STATES = [:red, :green, :yellow]
  DURATIONS = { red: 30, green: 25, yellow: 5 }

  def initialize
    @state = :red
  end

  def next_state
    current_index = STATES.index(@state)
    @state = STATES[(current_index + 1) % STATES.size]
  end

  def message
    case @state
    when :red then "หยุด"
    when :green then "ไปได้"
    when :yellow then "ระวัง"
    end
  end

  def duration
    DURATIONS[@state]
  end
end

light = TrafficLight.new
5.times do
  puts "#{light.message} (#{light.duration} วินาที)"
  light.next_state
end
```

**ข้อที่ 16-20:** แบบฝึกหัดเพิ่มเติม (สั้น ๆ)

```ruby
# ข้อ 16: Password Strength
def password_strength(password)
  return :invalid if password.length < 8
  has_upper = password =~ /[A-Z]/
  has_lower = password =~ /[a-z]/
  has_digit = password =~ /\d/
  has_symbol = password =~ /[!@#$%^&*]/
  
  score = [has_upper, has_lower, has_digit, has_symbol].count(&:itself)
  case score
  when 4 then :strong
  when 3 then :medium
  when 2 then :weak
  else :very_weak
  end
end

# ข้อ 17: Age Group
def age_group(age)
  case age
  when 0..12 then "เด็ก"
  when 13..17 then "วัยรุ่น"
  when 18..25 then "วัยหนุ่มสาว"
  when 26..60 then "วัยทำงาน"
  else "ผู้สูงอายุ"
  end
end

# ข้อ 18: Direction Calculator
def turn(current, direction)
  directions = [:north, :east, :south, :west]
  idx = directions.index(current)
  case direction
  when :right then directions[(idx + 1) % 4]
  when :left then directions[(idx - 1) % 4]
  when :back then directions[(idx + 2) % 4]
  else current
  end
end

# ข้อ 19: Simple State Machine
class Door
  def initialize
    @state = :closed
  end

  def action(input)
    case [@state, input]
    in [:closed, :open] then @state = :open
    in [:open, :close] then @state = :closed
    in [:closed, :lock] then @state = :locked
    in [:locked, :unlock] then @state = :closed
    else puts "ไม่สามารถทำ #{input} ได้ในสถานะ #{@state}"
    end
    puts "สถานะ: #{@state}"
  end
end

# ข้อ 20: Multi-condition Validator
def validate_age_for_activity(age, activity)
  requirements = {
    driving: 18..,
    voting: 18..,
    alcohol: 20..,
    senior_discount: 60..,
    children_program: 0..12
  }
  
  range = requirements[activity]
  return "ไม่รู้จักกิจกรรม" unless range
  range.include?(age) ? "อนุญาต" : "ไม่อนุญาต"
end

puts validate_age_for_activity(25, :driving)          # => อนุญาต
puts validate_age_for_activity(16, :driving)          # => ไม่อนุญาต
puts validate_age_for_activity(65, :senior_discount)  # => อนุญาต
```

---

## สรุปบทที่ 7

| Operator/Statement | การใช้งาน |
|-------------------|-----------|
| `if/elsif/else` | พื้นฐาน conditional |
| `unless` | ตรงข้าม if, ใช้เมื่อ condition เป็น negative |
| `? :` | Ternary, สำหรับ simple conditional |
| `case/when` | หลาย conditions, range, regex, types |
| `case/in` | Pattern matching (Ruby 2.7+) |
| `&&` / `\|\|` | Boolean AND/OR, short-circuit |
| `and` / `or` | Low precedence, flow control |
| `&.` | Safe navigation, nil-safe |
| `\|\|=` / `&&=` | Conditional assignment |
| Inline conditions | `if` / `unless` หลัง statement |
| Guard clauses | `return unless condition` |

**Key Takeaways:**
1. ใน Ruby มีเพียง `nil` และ `false` เท่านั้นที่เป็น falsy
2. Guard clauses ทำให้โค้ดอ่านง่ายและลด nesting
3. `case/when` ใช้ `===` operator ซึ่งทำงานต่างกันตาม type
4. Pattern matching (Ruby 3+) ทรงพลังมากสำหรับ complex data
5. `&.` ช่วยป้องกัน NoMethodError จาก nil

---

*ถัดไป: ตอนที่ 8 - Loops and Iterators (ขั้นตอนที่ 121-145)*
