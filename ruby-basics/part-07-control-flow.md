# ตอนที่ 7: Control Flow (Steps 101-120)

> **เป้าหมาย**: เรียนรู้การควบคุมทิศทางการทำงานของโปรแกรม Ruby ด้วย conditions, case statements, pattern matching และ logical operators

---

## Step 101: if / elsif / else / end — พื้นฐานการตัดสินใจ

`if` คือคำสั่งพื้นฐานที่สุดในการควบคุมการทำงานของโปรแกรม Ruby ใช้สำหรับตรวจสอบเงื่อนไขและสั่งให้ทำงานเฉพาะเมื่อเงื่อนไขเป็นจริง

### รูปแบบพื้นฐาน

```ruby
# รูปแบบที่ 1: if เดี่ยว
age = 20
if age >= 18
  puts "คุณเป็นผู้ใหญ่แล้ว"
end
# Output: คุณเป็นผู้ใหญ่แล้ว

# รูปแบบที่ 2: if / else
score = 75
if score >= 60
  puts "ผ่าน"
else
  puts "ไม่ผ่าน"
end
# Output: ผ่าน

# รูปแบบที่ 3: if / elsif / else
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
# Output: B
```

### if เป็น Expression (มีค่า return)

ใน Ruby, `if` เป็น expression หมายความว่ามันมีค่า return เสมอ

```ruby
# if statement มีค่า return
age = 25
result = if age >= 18
           "ผู้ใหญ่"
         else
           "เด็ก"
         end
puts result  # Output: ผู้ใหญ่

# ใช้กับ assignment โดยตรง
temperature = 35
weather = if temperature > 30
            "ร้อนมาก"
          elsif temperature > 25
            "ร้อน"
          elsif temperature > 20
            "อบอุ่น"
          else
            "เย็น"
          end
puts weather  # Output: ร้อนมาก
```

### ตัวอย่างจริง: ตรวจสอบราคาสมาชิก

```ruby
def membership_price(age, is_student)
  if age < 18
    price = 100
    type = "เยาวชน"
  elsif is_student
    price = 150
    type = "นักศึกษา"
  elsif age >= 60
    price = 120
    type = "ผู้สูงอายุ"
  else
    price = 300
    type = "ทั่วไป"
  end
  "ราคาสมาชิกแบบ#{type}: #{price} บาท/เดือน"
end

puts membership_price(15, false)   # ราคาสมาชิกแบบเยาวชน: 100 บาท/เดือน
puts membership_price(22, true)    # ราคาสมาชิกแบบนักศึกษา: 150 บาท/เดือน
puts membership_price(65, false)   # ราคาสมาชิกแบบผู้สูงอายุ: 120 บาท/เดือน
puts membership_price(35, false)   # ราคาสมาชิกแบบทั่วไป: 300 บาท/เดือน
```

### ข้อควรระวัง: nested if

```ruby
# หลีกเลี่ยง nested if ลึกเกินไป (ทำให้อ่านยาก)
# ❌ ไม่ดี
if user
  if user.active?
    if user.admin?
      puts "Admin ที่ active"
    end
  end
end

# ✅ ดีกว่า: ใช้ && เชื่อมเงื่อนไข
if user && user.active? && user.admin?
  puts "Admin ที่ active"
end
```

---

## Step 102: unless — ตรงข้ามกับ if

`unless` ทำงานเมื่อเงื่อนไขเป็น **เท็จ** (false) ตรงข้ามกับ `if` ที่ทำงานเมื่อเงื่อนไขเป็นจริง

```ruby
# unless = if not
logged_in = false

unless logged_in
  puts "กรุณาเข้าสู่ระบบก่อน"
end
# Output: กรุณาเข้าสู่ระบบก่อน

# เทียบเท่ากับ
if !logged_in
  puts "กรุณาเข้าสู่ระบบก่อน"
end
```

### unless / else

```ruby
is_weekend = false

unless is_weekend
  puts "วันทำงาน — ไปทำงานกัน!"
else
  puts "วันหยุด — พักผ่อนได้แล้ว!"
end
# Output: วันทำงาน — ไปทำงานกัน!
```

### เมื่อไหร่ควรใช้ unless?

```ruby
# ✅ ใช้ unless เมื่อเงื่อนไขเป็นการ "ไม่มี" หรือ "ไม่ได้"
items = []
unless items.empty?
  puts "มีสินค้าในตะกร้า"
end

# ✅ ใช้ unless กับ boolean flag
maintenance_mode = false
unless maintenance_mode
  start_application
end

# ❌ อย่าใช้ unless กับ else เมื่อ logic ซับซ้อน (ทำให้งงได้)
# เพราะมันจะกลายเป็น "unless...else" ซึ่งอ่านยาก

# ❌ อย่าใช้ unless กับ && หรือ ||
# unless x && y  <-- อ่านยากมาก ใช้ if !(x && y) แทน
```

### ตัวอย่างจริง: ตรวจสอบ input

```ruby
def process_order(items, payment_confirmed)
  unless items && !items.empty?
    puts "ไม่มีสินค้าในตะกร้า"
    return
  end

  unless payment_confirmed
    puts "การชำระเงินยังไม่ได้รับการยืนยัน"
    return
  end

  puts "กำลังประมวลผลออเดอร์ #{items.length} รายการ..."
end

process_order([], true)
# Output: ไม่มีสินค้าในตะกร้า

process_order(["กาแฟ", "เค้ก"], false)
# Output: การชำระเงินยังไม่ได้รับการยืนยัน

process_order(["กาแฟ", "เค้ก"], true)
# Output: กำลังประมวลผลออเดอร์ 2 รายการ...
```

---

## Step 103: Ternary Operator — เงื่อนไขในบรรทัดเดียว

Ternary operator ช่วยเขียน if/else แบบสั้นในบรรทัดเดียว รูปแบบคือ:
`condition ? value_if_true : value_if_false`

```ruby
# รูปแบบพื้นฐาน
age = 20
status = age >= 18 ? "ผู้ใหญ่" : "เด็ก"
puts status  # Output: ผู้ใหญ่

# เทียบเท่ากับ
status = if age >= 18 then "ผู้ใหญ่" else "เด็ก" end

# ใช้ใน string interpolation
score = 85
puts "คะแนน: #{score} — #{score >= 60 ? 'ผ่าน' : 'ไม่ผ่าน'}"
# Output: คะแนน: 85 — ผ่าน
```

### ตัวอย่างการใช้งานจริง

```ruby
# กำหนดค่า default
name = nil
display_name = name ? name : "ไม่ระบุชื่อ"
puts display_name  # Output: ไม่ระบุชื่อ

# (Ruby มี || สำหรับกรณีนี้ด้วย)
display_name = name || "ไม่ระบุชื่อ"
puts display_name  # Output: ไม่ระบุชื่อ

# ใช้กับ method
def discount_price(price, is_member)
  discounted = is_member ? price * 0.8 : price
  "ราคา: #{discounted.round(2)} บาท #{is_member ? '(ลด 20%)' : ''}"
end

puts discount_price(100, true)   # ราคา: 80.0 บาท (ลด 20%)
puts discount_price(100, false)  # ราคา: 100 บาท

# Nested ternary (ใช้ด้วยความระมัดระวัง)
score = 75
grade = score >= 80 ? "ดีมาก" : (score >= 60 ? "ผ่าน" : "ไม่ผ่าน")
puts grade  # Output: ผ่าน
```

### ข้อแนะนำในการใช้ Ternary

```ruby
# ✅ ดี: เงื่อนไขสั้น ค่าสั้น
label = active ? "เปิด" : "ปิด"

# ✅ ดี: ใน string interpolation
puts "สถานะ: #{connected ? 'ออนไลน์' : 'ออฟไลน์'}"

# ❌ หลีกเลี่ยง: ternary ที่ซับซ้อนเกินไป
# result = a > b ? (c > d ? "x" : "y") : (e > f ? "p" : "q")
# ควรใช้ if/elsif แทน
```

---

## Step 104: One-line Conditions — เงื่อนไขท้ายคำสั่ง

Ruby อนุญาตให้เขียนเงื่อนไข **หลัง** คำสั่งในบรรทัดเดียว ซึ่งเป็นสไตล์ที่นิยมมากใน Ruby

```ruby
# รูปแบบ: statement if condition
puts "สวัสดี!" if true
puts "ผ่านแล้ว!" if score >= 60
puts "ยังไม่ผ่าน" unless score >= 60

# ตัวอย่างจริง
age = 17
puts "คุณยังเด็กอยู่" if age < 18
puts "ยังไม่บรรลุนิติภาวะ" unless age >= 18

# ใช้กับ assignment
discount = nil
discount = 0.2 if is_member
discount = 0.1 if order_total > 500
```

### การใช้งานทั่วไป

```ruby
# Return early
def login(username, password)
  return "กรุณาระบุ username" if username.nil? || username.empty?
  return "กรุณาระบุ password" if password.nil? || password.empty?
  return "Username/Password ไม่ถูกต้อง" unless authenticate(username, password)
  
  "เข้าสู่ระบบสำเร็จ! ยินดีต้อนรับ #{username}"
end

# Raise errors
def divide(a, b)
  raise ArgumentError, "ห้ามหารด้วยศูนย์" if b == 0
  a / b.to_f
end

# Object modification
items = [1, 2, 3, 4, 5]
items.delete(3) if items.include?(3)
puts items.inspect  # [1, 2, 4, 5]
```

### One-line unless

```ruby
numbers = [1, 2, 3, 4, 5]
total = 0
numbers.each { |n| total += n unless n.even? }
puts total  # Output: 9 (1 + 3 + 5)

# ใช้ใน Guard clause
def process(data)
  return unless data
  return unless data.valid?
  data.save!
end
```

---

## Step 105: case / when — Ruby's Switch Statement

`case/when` ใช้เมื่อต้องการเปรียบเทียบค่าหนึ่งกับหลายๆ ค่า มีความสะอาดกว่าการใช้ if/elsif หลายชั้น

```ruby
# รูปแบบพื้นฐาน
day = "Monday"

case day
when "Monday"
  puts "วันจันทร์ — เริ่มต้นสัปดาห์ใหม่"
when "Friday"
  puts "วันศุกร์ — ใกล้จะหยุดแล้ว!"
when "Saturday", "Sunday"
  puts "วันหยุดสุดสัปดาห์!"
else
  puts "วันธรรมดา"
end
# Output: วันจันทร์ — เริ่มต้นสัปดาห์ใหม่
```

### case เป็น Expression

```ruby
# case สามารถคืนค่าได้
month = 8
season = case month
         when 12, 1, 2
           "ฤดูหนาว"
         when 3, 4, 5
           "ฤดูใบไม้ผลิ"
         when 6, 7, 8
           "ฤดูร้อน"
         when 9, 10, 11
           "ฤดูใบไม้ร่วง"
         end
puts season  # Output: ฤดูร้อน
```

### case ใช้ === (triple equals)

ความพิเศษของ Ruby คือ `case/when` ใช้ `===` (triple equals) แทน `==` ซึ่งทำให้ยืดหยุ่นมาก

```ruby
# === กับ Integer จะเป็น == ธรรมดา
case 5
when 5
  puts "เป็น 5"
end

# === กับ String จะเป็น == ธรรมดา
case "hello"
when "hello"
  puts "สวัสดี"
end

# === กับ Symbol
status = :active
case status
when :active
  puts "กำลังทำงาน"
when :inactive
  puts "หยุดทำงาน"
when :pending
  puts "รอดำเนินการ"
end
# Output: กำลังทำงาน
```

### ตัวอย่างจริง: HTTP Status Codes

```ruby
def handle_response(status_code)
  message = case status_code
            when 200
              "สำเร็จ"
            when 201
              "สร้างข้อมูลสำเร็จ"
            when 400
              "Bad Request — ข้อมูลไม่ถูกต้อง"
            when 401
              "Unauthorized — กรุณาเข้าสู่ระบบ"
            when 403
              "Forbidden — ไม่มีสิทธิ์"
            when 404
              "Not Found — ไม่พบข้อมูล"
            when 500
              "Internal Server Error"
            else
              "Unknown Status: #{status_code}"
            end
  puts "Status #{status_code}: #{message}"
end

handle_response(200)  # Status 200: สำเร็จ
handle_response(404)  # Status 404: Not Found — ไม่พบข้อมูล
handle_response(403)  # Status 403: Forbidden — ไม่มีสิทธิ์
```

---

## Step 106: case กับ Range — ตรวจสอบช่วงค่า

`case/when` กับ Range ใช้ `===` ซึ่งใน Range หมายความว่า "อยู่ในช่วงนี้ไหม"

```ruby
# ตรวจสอบคะแนน
score = 78

grade = case score
        when 90..100
          "A"
        when 80...90
          "B"
        when 70...80
          "C"
        when 60...70
          "D"
        when 0...60
          "F"
        else
          "คะแนนไม่ถูกต้อง"
        end
puts "เกรด: #{grade}"  # เกรด: C
```

### ความแตกต่างระหว่าง .. และ ...

```ruby
# .. (inclusive) รวมค่าสุดท้าย
(1..5).include?(5)   # true

# ... (exclusive) ไม่รวมค่าสุดท้าย
(1...5).include?(5)  # false
(1...5).include?(4)  # true
```

### ตัวอย่างจริง: ราคาสินค้าตามปริมาณ

```ruby
def calculate_price(quantity, unit_price)
  discount = case quantity
             when 1..9
               0.0     # ไม่ลด
             when 10..49
               0.05    # ลด 5%
             when 50..99
               0.10    # ลด 10%
             when 100..499
               0.15    # ลด 15%
             when 500..Float::INFINITY
               0.20    # ลด 20%
             else
               0.0
             end

  total = quantity * unit_price * (1 - discount)
  discount_percent = (discount * 100).to_i

  puts "จำนวน: #{quantity} ชิ้น"
  puts "ส่วนลด: #{discount_percent}%"
  puts "ราคารวม: #{total.round(2)} บาท"
  puts "---"
end

calculate_price(5, 100)
# จำนวน: 5 ชิ้น
# ส่วนลด: 0%
# ราคารวม: 500.0 บาท

calculate_price(25, 100)
# จำนวน: 25 ชิ้น
# ส่วนลด: 5%
# ราคารวม: 2375.0 บาท

calculate_price(100, 100)
# จำนวน: 100 ชิ้น
# ส่วนลด: 15%
# ราคารวม: 8500.0 บาท
```

### case กับ Range สำหรับ BMI

```ruby
def bmi_category(bmi)
  case bmi
  when Float::INFINITY..Float::INFINITY
    "ค่า BMI ไม่ถูกต้อง"
  when 0...18.5
    "น้ำหนักน้อยกว่าเกณฑ์"
  when 18.5...25.0
    "น้ำหนักปกติ"
  when 25.0...30.0
    "น้ำหนักเกิน"
  when 30.0..Float::INFINITY
    "อ้วน"
  else
    "ค่า BMI ไม่ถูกต้อง"
  end
end

puts bmi_category(17.5)  # น้ำหนักน้อยกว่าเกณฑ์
puts bmi_category(22.0)  # น้ำหนักปกติ
puts bmi_category(27.5)  # น้ำหนักเกิน
puts bmi_category(32.0)  # อ้วน
```

---

## Step 107: case กับ Regex — Pattern Matching บน String

`case/when` กับ Regular Expression (Regex) ใช้ `===` ซึ่งใน Regex หมายความว่า "ตรงกับ pattern นี้ไหม"

```ruby
# ตรวจสอบรูปแบบ string
phone = "0812345678"

phone_type = case phone
             when /^0[689]\d{8}$/
               "มือถือ"
             when /^0[2-5]\d{7}$/
               "บ้าน/ออฟฟิศ"
             when /^1\d{3}$/
               "เบอร์สายด่วน"
             else
               "ไม่ทราบประเภท"
             end
puts phone_type  # Output: มือถือ
```

### ตัวอย่างจริง: ตรวจสอบ Email Format

```ruby
def classify_email(email)
  case email
  when /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
    case email
    when /@gmail\.com$/i
      "Gmail account"
    when /@yahoo\.com$/i
      "Yahoo account"
    when /@hotmail\.com$/i, /@outlook\.com$/i
      "Microsoft account"
    when /\.(ac|edu)\./i, /\.edu$/i
      "Educational institution"
    when /\.(co\.th|or\.th|ac\.th)$/i
      "Thai domain"
    else
      "Email ทั่วไป"
    end
  else
    "รูปแบบ Email ไม่ถูกต้อง"
  end
end

puts classify_email("user@gmail.com")         # Gmail account
puts classify_email("student@ku.ac.th")       # Thai domain
puts classify_email("not-an-email")           # รูปแบบ Email ไม่ถูกต้อง
```

### ตัวอย่างจริง: Parser อย่างง่าย

```ruby
def parse_command(input)
  case input.strip.downcase
  when /^help(\s.*)?$/
    show_help($1&.strip)
  when /^quit|^exit|^bye/
    puts "ลาก่อน! 👋"
    exit
  when /^add\s+(.+)$/
    item = $1
    puts "เพิ่ม: #{item}"
  when /^delete\s+(\d+)$/
    id = $1.to_i
    puts "ลบรายการ ##{id}"
  when /^list(\s+all)?$/
    show_all = !$1.nil?
    puts show_all ? "แสดงทั้งหมด" : "แสดงรายการ"
  else
    puts "ไม่รู้จักคำสั่ง: '#{input}'"
    puts "พิมพ์ 'help' เพื่อดูคำสั่งที่ใช้ได้"
  end
end

parse_command("add กาแฟ")     # เพิ่ม: กาแฟ
parse_command("delete 3")     # ลบรายการ #3
parse_command("list all")     # แสดงทั้งหมด
```

---

## Step 108: case กับ Class — Type Checking

`case/when` ใช้กับ Class เพื่อตรวจสอบประเภทของ object โดยใช้ `===` ซึ่งใน Class หมายความว่า `is_a?`

```ruby
# ตรวจสอบประเภทข้อมูล
def describe(value)
  case value
  when Integer
    "#{value} เป็นจำนวนเต็ม"
  when Float
    "#{value} เป็นทศนิยม"
  when String
    "\"#{value}\" เป็น String (ยาว #{value.length} ตัว)"
  when Array
    "Array มี #{value.length} สมาชิก"
  when Hash
    "Hash มี #{value.length} คู่ key-value"
  when Symbol
    ":#{value} เป็น Symbol"
  when NilClass
    "nil (ไม่มีค่า)"
  when TrueClass, FalseClass
    "#{value} เป็น Boolean"
  else
    "#{value.class} — ไม่รู้จักประเภท"
  end
end

puts describe(42)              # 42 เป็นจำนวนเต็ม
puts describe(3.14)            # 3.14 เป็นทศนิยม
puts describe("สวัสดี")        # "สวัสดี" เป็น String (ยาว 7 ตัว)
puts describe([1, 2, 3])       # Array มี 3 สมาชิก
puts describe({a: 1})          # Hash มี 1 คู่ key-value
puts describe(:name)           # :name เป็น Symbol
puts describe(nil)             # nil (ไม่มีค่า)
puts describe(true)            # true เป็น Boolean
```

### ตัวอย่างจริง: Polymorphic Calculator

```ruby
def calculate(a, b, operation)
  # ตรวจสอบประเภทของ operation
  result = case operation
           when String
             case operation
             when "+"   then a + b
             when "-"   then a - b
             when "*"   then a * b
             when "/"   then b != 0 ? a.to_f / b : "หารด้วยศูนย์ไม่ได้"
             else "ไม่รู้จัก operation: #{operation}"
             end
           when Symbol
             case operation
             when :add      then a + b
             when :subtract then a - b
             when :multiply then a * b
             when :divide   then b != 0 ? a.to_f / b : "หารด้วยศูนย์ไม่ได้"
             end
           when Proc, Method
             operation.call(a, b)
           else
             "operation ต้องเป็น String, Symbol, หรือ Proc"
           end
  result
end

puts calculate(10, 5, "+")          # 15
puts calculate(10, 5, :multiply)    # 50
puts calculate(10, 5, ->(a, b) { a ** b })  # 100000
```

### ใช้ case กับ inheritance

```ruby
class Animal
  def speak; "..."; end
end

class Dog < Animal
  def speak; "โฮ่ง!"; end
end

class Cat < Animal
  def speak; "เมี้ยว!"; end
end

class Bird < Animal
  def speak; "จีจ๊า!"; end
end

def describe_animal(animal)
  sound = animal.speak
  type = case animal
         when Dog   then "สุนัข"
         when Cat   then "แมว"
         when Bird  then "นก"
         when Animal then "สัตว์ทั่วไป"
         else "ไม่รู้จัก"
         end
  puts "#{type} พูดว่า: #{sound}"
end

describe_animal(Dog.new)    # สุนัข พูดว่า: โฮ่ง!
describe_animal(Cat.new)    # แมว พูดว่า: เมี้ยว!
describe_animal(Bird.new)   # นก พูดว่า: จีจ๊า!
```

---

## Step 109: Pattern Matching (Ruby 3+) — in operator

Pattern Matching เป็นฟีเจอร์ใหม่ใน Ruby 2.7+ (stable ใน Ruby 3.0) ช่วยให้ตรวจสอบ structure ของ data ได้อย่างทรงพลัง

### รูปแบบพื้นฐาน: case/in

```ruby
# ใช้ in แทน when
data = { name: "สมชาย", age: 25, role: "admin" }

case data
in { name: String => name, role: "admin" }
  puts "Admin: #{name}"
in { name: String => name, role: "user" }
  puts "User: #{name}"
in { name: String => name }
  puts "Unknown role: #{name}"
end
# Output: Admin: สมชาย
```

### Pattern Matching กับ Array

```ruby
# Match array patterns
point = [3, 4]

case point
in [0, 0]
  puts "จุดกำเนิด"
in [x, 0]
  puts "บนแกน X ที่ #{x}"
in [0, y]
  puts "บนแกน Y ที่ #{y}"
in [x, y]
  puts "จุดที่ (#{x}, #{y})"
end
# Output: จุดที่ (3, 4)
```

### Pattern Matching กับ Hash

```ruby
# Match hash patterns
response = { status: 200, body: { users: ["Alice", "Bob"] } }

case response
in { status: 200, body: { users: [first, *rest] } }
  puts "สำเร็จ! User แรก: #{first}, เหลือ: #{rest.length} คน"
in { status: 404 }
  puts "ไม่พบข้อมูล"
in { status: (500..), body: { error: message } }
  puts "Server error: #{message}"
in { status: Integer => code }
  puts "Status: #{code}"
end
# Output: สำเร็จ! User แรก: Alice, เหลือ: 1 คน
```

### Find Pattern

```ruby
# หา pattern ในกลาง array
users = [
  { name: "Alice", role: :user },
  { name: "Bob", role: :admin },
  { name: "Charlie", role: :user }
]

case users
in [*, { name: String => admin_name, role: :admin }, *]
  puts "พบ admin: #{admin_name}"
end
# Output: พบ admin: Bob
```

### One-line Pattern Matching (Ruby 3.0+)

```ruby
# => pattern (จะ raise error ถ้าไม่ match)
{ name: "Alice", age: 30 } => { name: String => name }
puts name  # Alice

# in pattern (return true/false)
if { name: "Alice", age: 30 } in { name: /^A/ => name }
  puts "ชื่อขึ้นต้นด้วย A: #{name}"
end
```

---

## Step 110: Logical Operators — &&, ||, !, and, or, not

Ruby มี logical operators 2 ชุด: `&&, ||, !` (operator ทั่วไป) และ `and, or, not` (คำสงวน)

### && (AND) และ || (OR) และ ! (NOT)

```ruby
# && (AND) — ทั้งสองต้องจริง
puts true && true    # true
puts true && false   # false
puts false && true   # false
puts false && false  # false

# || (OR) — อย่างน้อยหนึ่งอย่างต้องจริง
puts true || false   # true
puts false || true   # true
puts false || false  # false
puts true || true    # true

# ! (NOT) — กลับค่า
puts !true   # false
puts !false  # true
puts !nil    # true
puts !0      # false (0 เป็น truthy ใน Ruby!)
```

### การใช้งานจริง

```ruby
# ตรวจสอบหลายเงื่อนไขพร้อมกัน
age = 25
income = 50000
has_job = true

eligible_for_loan = age >= 20 && income >= 30000 && has_job
puts "สิทธิ์กู้เงิน: #{eligible_for_loan ? 'ใช่' : 'ไม่ใช่'}"
# Output: สิทธิ์กู้เงิน: ใช่

# ตรวจสอบว่า valid
def valid_user?(user)
  !user.nil? && !user[:name].nil? && user[:age] >= 18
end

puts valid_user?({ name: "Alice", age: 25 })  # true
puts valid_user?({ name: "Bob", age: 15 })    # false
puts valid_user?(nil)                          # false
```

### and, or, not — ใช้สำหรับ control flow

```ruby
# and, or, not มี precedence ต่ำกว่า = จึงนิยมใช้กับ control flow

# Pattern: do_something or raise_error
result = find_user or raise "ไม่พบ user"

# Pattern: do_something and do_next
user = find_user and user.activate!

# ข้อควรระวัง: and/or มี precedence ต่ำกว่า &&/||
x = true && false   # x = (true && false) = false
x = true and false  # x = true   (x = true ทำก่อน แล้วค่อย and false)
```

### ตารางความแตกต่าง

```ruby
# &&, || มี precedence สูงกว่า and, or
# ใช้ &&, || สำหรับ boolean expressions
# ใช้ and, or สำหรับ control flow (แต่ไม่แนะนำใน code สมัยใหม่)

# ตัวอย่างที่อาจงงได้
a = false || true    # a = true (|| มี precedence สูงกว่า =)
a = false or true    # a = false! (= มี precedence สูงกว่า or)

puts a  # false (เพราะ a = false รัน ก่อน แล้ว or true ไม่มีผล)
```

---

## Step 111: Short-circuit Evaluation — การประเมินแบบ lazy

Short-circuit evaluation หมายความว่า Ruby จะหยุดประเมินเงื่อนไขทันทีที่รู้ผลแล้ว

```ruby
# && หยุดเมื่อพบ false ตัวแรก
def check_a
  puts "ตรวจสอบ A"
  false
end

def check_b
  puts "ตรวจสอบ B"
  true
end

result = check_a && check_b
# Output: ตรวจสอบ A  (check_b ไม่ถูกเรียก!)
puts result  # false

# || หยุดเมื่อพบ true ตัวแรก
result = check_b || check_a
# Output: ตรวจสอบ B  (check_a ไม่ถูกเรียก!)
puts result  # true
```

### การใช้ Short-circuit สำหรับ Safety Check

```ruby
# Guard against nil
user = nil
# ❌ อันตราย — จะ raise NoMethodError
# puts user.name

# ✅ ปลอดภัย — short-circuit จะหยุดก่อน
puts user && user.name  # nil (ไม่ raise error)

# ใช้ && chain
puts user && user.address && user.address.city  # nil

# ✅ หรือใช้ &. (safe navigation)
puts user&.name  # nil

# Default values ด้วย ||
config = nil
timeout = config || 30
puts timeout  # 30

# Multiple fallbacks
env_value = ENV["TIMEOUT"]
config_value = nil
default_value = 30
timeout = env_value || config_value || default_value
```

### ระวัง: 0 และ "" เป็น truthy ใน Ruby!

```ruby
# ใน Ruby: false และ nil เท่านั้นที่ falsy!
# ทุกอย่างอื่น (รวมถึง 0, "", [], {}) เป็น truthy

value = 0
result = value || "default"
puts result  # 0  (ไม่ใช่ "default"!)

value = ""
result = value || "default"
puts result  # ""  (ไม่ใช่ "default"!)

# ถ้าต้องการ default เมื่อ nil เท่านั้น
value = 0
result = value.nil? ? "default" : value
puts result  # 0

# หรือใช้
result = value || "default"  # ใช้ได้เฉพาะเมื่อ 0 ไม่ใช่ค่า valid
```

---

## Step 112: Truthiness ใน Ruby — false และ nil เท่านั้นที่ falsy

นี่คือหนึ่งในสิ่งที่สำคัญที่สุดใน Ruby: **มีแค่ `false` และ `nil` เท่านั้นที่เป็น falsy**

```ruby
# Falsy values ใน Ruby (มีแค่ 2)
puts !!false   # false
puts !!nil     # false

# Truthy values ใน Ruby (ทุกอย่างอื่น!)
puts !!true    # true
puts !!0       # true  ← ต่างจาก JavaScript!
puts !!""      # true  ← ต่างจาก JavaScript!
puts !![]      # true
puts !!{}      # true
puts !!:sym    # true
```

### ความแตกต่างจาก JavaScript

```ruby
# JavaScript: 0, "", null, undefined, NaN, false เป็น falsy
# Ruby: nil, false เท่านั้นที่ falsy

# ดังนั้นต้องระวัง!
items = []
if items
  puts "มีรายการ"  # ← พิมพ์นี้! เพราะ [] เป็น truthy
else
  puts "ไม่มีรายการ"
end

# ควรใช้
if items.any?
  puts "มีรายการ"
else
  puts "ไม่มีรายการ"
end
```

### ตัวอย่างที่อาจงงได้

```ruby
# ระวัง: 0 เป็น truthy
count = 0
if count
  puts "count มีค่า"  # ← พิมพ์นี้! (0 เป็น truthy)
end

# ถ้าต้องการตรวจสอบว่า "มีค่าและไม่ใช่ 0"
if count && count > 0
  puts "count มากกว่า 0"
end

# ระวัง: "" เป็น truthy
name = ""
if name
  puts "มีชื่อ"  # ← พิมพ์นี้! ("" เป็น truthy)
end

# ถ้าต้องการตรวจสอบว่า "มีชื่อและไม่ว่างเปล่า"
if name && !name.empty?
  puts "มีชื่อ"
end

# หรือ
if name.present?  # Rails method
  puts "มีชื่อ"
end
```

---

## Step 113: Guard Clauses Pattern — เขียนโค้ดที่สะอาดขึ้น

Guard clause คือการตรวจสอบเงื่อนไข "ล้มเหลว" ก่อน แล้ว return เลย เพื่อลด nesting และทำให้โค้ดอ่านง่ายขึ้น

### ปัญหา: Nested Conditions

```ruby
# ❌ ไม่ดี: deep nesting
def process_payment(user, amount, card)
  if user
    if user.active?
      if amount > 0
        if card
          if card.valid?
            if card.has_sufficient_funds?(amount)
              charge_card(card, amount)
              send_receipt(user, amount)
              "ชำระเงินสำเร็จ"
            else
              "ยอดเงินในบัตรไม่เพียงพอ"
            end
          else
            "บัตรไม่ถูกต้อง"
          end
        else
          "ไม่พบข้อมูลบัตร"
        end
      else
        "จำนวนเงินต้องมากกว่า 0"
      end
    else
      "บัญชีไม่ได้ใช้งาน"
    end
  else
    "ไม่พบผู้ใช้"
  end
end
```

### วิธีแก้: Guard Clauses

```ruby
# ✅ ดี: Guard clauses
def process_payment(user, amount, card)
  return "ไม่พบผู้ใช้" unless user
  return "บัญชีไม่ได้ใช้งาน" unless user.active?
  return "จำนวนเงินต้องมากกว่า 0" unless amount > 0
  return "ไม่พบข้อมูลบัตร" unless card
  return "บัตรไม่ถูกต้อง" unless card.valid?
  return "ยอดเงินในบัตรไม่เพียงพอ" unless card.has_sufficient_funds?(amount)

  charge_card(card, amount)
  send_receipt(user, amount)
  "ชำระเงินสำเร็จ"
end
```

### ตัวอย่างจริง: User Validation

```ruby
def create_user(params)
  return { error: "params ต้องเป็น Hash" } unless params.is_a?(Hash)
  return { error: "ต้องระบุชื่อ" } if params[:name].nil? || params[:name].empty?
  return { error: "ต้องระบุ email" } if params[:email].nil? || params[:email].empty?
  return { error: "Email ไม่ถูกต้อง" } unless params[:email].include?("@")
  return { error: "อายุต้องมากกว่า 0" } if params[:age] && params[:age] <= 0
  return { error: "Email นี้ถูกใช้แล้ว" } if User.exists?(email: params[:email])

  user = User.new(params)
  user.save!
  { success: true, user: user }
end
```

### Guard Clauses ใน loops

```ruby
# ข้ามรายการที่ไม่ต้องการ
def process_orders(orders)
  results = []
  orders.each do |order|
    next if order[:status] == :cancelled  # ข้ามที่ยกเลิกแล้ว
    next unless order[:payment_confirmed]  # ข้ามที่ยังไม่ชำระ
    next if order[:items].empty?           # ข้ามที่ไม่มีสินค้า
    
    results << process_single_order(order)
  end
  results
end
```

---

## Step 114: Safe Navigation Operator (&.) — Nil-safe Method Calls

`&.` (ampersand dot หรือ "safe navigation operator") ช่วยเรียก method บน object ที่อาจเป็น nil ได้อย่างปลอดภัย

```ruby
# ❌ อันตราย
user = nil
user.name  # NoMethodError: undefined method 'name' for nil:NilClass

# ✅ ปลอดภัยด้วย &.
user = nil
puts user&.name   # nil (ไม่ raise error)

user = { name: "Alice", age: 25 }
# user.name จะ error เพราะ Hash ไม่มี method .name
# ถ้าเป็น object ที่มี .name method:
# puts user&.name  # "Alice"
```

### ตัวอย่างการใช้งานจริง

```ruby
# โดยปกติต้องเขียนแบบนี้ (ยาวและซ้ำ)
city = user && user.address && user.address.city

# ด้วย &. เขียนสั้นกว่า
city = user&.address&.city
puts city  # nil ถ้า user หรือ address เป็น nil

# ตัวอย่างจริง
class User
  attr_reader :name, :address

  def initialize(name, address = nil)
    @name = name
    @address = address
  end
end

class Address
  attr_reader :city, :zip_code

  def initialize(city, zip_code)
    @city = city
    @zip_code = zip_code
  end
end

# User มี address
alice = User.new("Alice", Address.new("กรุงเทพ", "10100"))
puts alice&.address&.city     # กรุงเทพ
puts alice&.address&.zip_code # 10100

# User ไม่มี address
bob = User.new("Bob")
puts bob&.address&.city  # nil (ไม่ error!)

# User เป็น nil
no_user = nil
puts no_user&.name&.upcase  # nil (ไม่ error!)
```

### ใช้กับ Arrays

```ruby
# &. กับ methods ทั่วไป
names = ["Alice", "Bob", nil, "Charlie"]
processed = names.map { |name| name&.upcase }
puts processed.inspect  # ["ALICE", "BOB", nil, "CHARLIE"]

# &. กับ block
response = nil
items = response&.fetch(:items, []) || []
puts items.inspect  # []
```

### &. กับ Assignment

```ruby
# กำหนดค่าเฉพาะเมื่อ object ไม่เป็น nil
user = nil
user&.name = "Alice"  # ไม่ทำอะไร ไม่ error

user = User.new("Bob")
user&.update(name: "Charlie")  # ทำงานปกติ
```

---

## Step 115: ตัวอย่าง종합 (Comprehensive Examples)

### ตัวอย่างที่ 1: Game State Machine

```ruby
def handle_game_event(state, event)
  case [state, event]
  in [:idle, :start]
    puts "เกมเริ่มต้น!"
    :playing
  in [:playing, :pause]
    puts "หยุดพักชั่วคราว"
    :paused
  in [:paused, :resume]
    puts "เล่นต่อ!"
    :playing
  in [:playing, :game_over]
    puts "เกมจบแล้ว"
    :game_over
  in [:playing, :win]
    puts "ชนะ! ยินดีด้วย!"
    :win
  in [state, event]
    puts "ไม่สามารถ #{event} ใน state: #{state}"
    state
  end
end

state = :idle
state = handle_game_event(state, :start)    # เกมเริ่มต้น!
state = handle_game_event(state, :pause)    # หยุดพักชั่วคราว
state = handle_game_event(state, :resume)   # เล่นต่อ!
state = handle_game_event(state, :win)      # ชนะ! ยินดีด้วย!
```

### ตัวอย่างที่ 2: Form Validator

```ruby
def validate_registration(data)
  errors = []

  # ตรวจสอบ name
  case data[:name]
  in nil | ""
    errors << "ชื่อต้องไม่ว่างเปล่า"
  in String => name if name.length < 2
    errors << "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"
  in String => name if name.length > 50
    errors << "ชื่อต้องไม่เกิน 50 ตัวอักษร"
  end

  # ตรวจสอบ email
  case data[:email]
  in nil | ""
    errors << "Email ต้องไม่ว่างเปล่า"
  in String => email unless email.match?(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
    errors << "รูปแบบ Email ไม่ถูกต้อง"
  end

  # ตรวจสอบ age
  case data[:age]
  in nil
    errors << "กรุณาระบุอายุ"
  in Integer => age if age < 18
    errors << "ต้องมีอายุ 18 ปีขึ้นไป"
  in Integer => age if age > 120
    errors << "อายุไม่ถูกต้อง"
  end

  if errors.empty?
    { valid: true, message: "ข้อมูลถูกต้อง" }
  else
    { valid: false, errors: errors }
  end
end

result = validate_registration({ name: "A", email: "invalid", age: 15 })
puts result[:errors].join(", ")
# ชื่อต้องมีอย่างน้อย 2 ตัวอักษร, รูปแบบ Email ไม่ถูกต้อง, ต้องมีอายุ 18 ปีขึ้นไป

result = validate_registration({ name: "สมชาย", email: "somchai@example.com", age: 25 })
puts result[:message]  # ข้อมูลถูกต้อง
```

---

## Step 116–120: เทคนิคขั้นสูงและ Best Practices

### Step 116: Combining Conditions

```ruby
# ใช้ Array#include? แทน multiple ||
def valid_status?(status)
  [:active, :pending, :inactive].include?(status)
end

puts valid_status?(:active)   # true
puts valid_status?(:deleted)  # false

# ใช้ Range#cover? สำหรับตรวจสอบช่วง
def valid_score?(score)
  (0..100).cover?(score)
end

puts valid_score?(85)   # true
puts valid_score?(105)  # false

# ใช้ Regex match?
def valid_phone?(phone)
  phone.match?(/^0[689]\d{8}$/)
end
```

### Step 117: Conditional Assignment

```ruby
# ||= กำหนดค่าเฉพาะเมื่อ nil หรือ false
@cache ||= {}
@cache[:key] ||= expensive_calculation()

# &&= กำหนดค่าเฉพาะเมื่อไม่ nil และไม่ false
user.name &&= user.name.strip.downcase

# ตัวอย่างจริง: memoization
def expensive_result
  @result ||= begin
    # การคำนวณที่ใช้เวลานาน
    sleep(2)
    42
  end
end

# ครั้งแรกจะใช้เวลา 2 วินาที
# ครั้งถัดไปจะคืนค่าจาก cache ทันที
```

### Step 118: Conditional Methods

```ruby
# สร้าง method ที่ทำงานตามเงื่อนไข
class Order
  attr_reader :status, :total

  def initialize(total)
    @total = total
    @status = :pending
  end

  def process!
    validate! or return false
    calculate_tax!
    apply_discounts! if eligible_for_discount?
    charge_payment! and update_status!(:paid)
    true
  end

  private

  def validate!
    return false unless @total > 0
    true
  end

  def eligible_for_discount?
    @total >= 500
  end

  def calculate_tax!
    @tax = @total * 0.07
  end

  def apply_discounts!
    @discount = @total >= 1000 ? @total * 0.10 : @total * 0.05
  end

  def update_status!(new_status)
    @status = new_status
  end
end
```

### Step 119: Pattern Matching กับ Deconstruct

```ruby
# สร้าง class ที่ support pattern matching
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
  end

  # สำหรับ array pattern matching
  def deconstruct
    [@x, @y]
  end

  # สำหรับ hash pattern matching
  def deconstruct_keys(keys)
    { x: @x, y: @y }
  end
end

point = Point.new(3, 4)

# Array pattern
case point
in [0, 0]
  puts "จุดกำเนิด"
in [x, y]
  puts "จุดที่ (#{x}, #{y})"
end
# Output: จุดที่ (3, 4)

# Hash pattern
case point
in { x: 0, y: }
  puts "บนแกน Y ที่ #{y}"
in { x:, y: 0 }
  puts "บนแกน X ที่ #{x}"
in { x:, y: }
  puts "จุดที่ x=#{x}, y=#{y}"
end
# Output: จุดที่ x=3, y=4
```

### Step 120: Best Practices Summary

```ruby
# 1. ใช้ Guard Clauses แทน deep nesting
def good_example(x, y)
  return "x ต้องเป็นบวก" unless x > 0
  return "y ต้องเป็นบวก" unless y > 0
  x * y
end

# 2. ใช้ ternary สำหรับ simple conditions เท่านั้น
label = enabled ? "เปิด" : "ปิด"  # ✅

# 3. ใช้ unless เมื่อมีเงื่อนไขเดียวและเป็น negation
return unless valid?  # ✅

# 4. ใช้ &. เพื่อหลีกเลี่ยง nil check
city = user&.address&.city  # ✅

# 5. ใช้ case/in (pattern matching) กับ complex data
case response
in { status: 200, data: { users: [*, { admin: true, name: }, *] } }
  puts "มี admin ชื่อ #{name}"
end

# 6. อย่าลืม: 0 และ "" เป็น truthy!
count = 0
if count                  # ❌ จะเป็น true!
if count.positive?        # ✅ ถูกต้อง

# 7. ใช้ ||= สำหรับ memoization
@cache ||= {}  # ✅
```

---

## แบบฝึกหัดตอนที่ 7 (20 ข้อ)

### ข้อ 1: เกรดนักเรียน
เขียน method `calculate_grade(score)` ที่รับคะแนน 0-100 และคืนเกรด A, B, C, D, F พร้อม GPA (4.0, 3.0, 2.0, 1.0, 0.0)

```ruby
# เฉลย
def calculate_grade(score)
  return { grade: "Invalid", gpa: nil } unless score.is_a?(Numeric) && (0..100).cover?(score)
  
  case score
  when 80..100 then { grade: "A", gpa: 4.0 }
  when 70...80 then { grade: "B", gpa: 3.0 }
  when 60...70 then { grade: "C", gpa: 2.0 }
  when 50...60 then { grade: "D", gpa: 1.0 }
  else              { grade: "F", gpa: 0.0 }
  end
end

puts calculate_grade(92).inspect  # {:grade=>"A", :gpa=>4.0}
puts calculate_grade(75).inspect  # {:grade=>"B", :gpa=>3.0}
puts calculate_grade(45).inspect  # {:grade=>"F", :gpa=>0.0}
```

### ข้อ 2: FizzBuzz
เขียน method `fizzbuzz(n)` ที่คืน "Fizz" ถ้าหารด้วย 3 ลงตัว, "Buzz" ถ้าหารด้วย 5 ลงตัว, "FizzBuzz" ถ้าหารด้วย 15 ลงตัว, ไม่งั้นคืนตัวเลขนั้น

```ruby
# เฉลย
def fizzbuzz(n)
  case n
  when ->(x) { x % 15 == 0 } then "FizzBuzz"
  when ->(x) { x % 3 == 0 }  then "Fizz"
  when ->(x) { x % 5 == 0 }  then "Buzz"
  else n.to_s
  end
end

(1..20).each { |i| print "#{fizzbuzz(i)} " }
# 1 2 Fizz 4 Buzz Fizz 7 8 Fizz Buzz 11 Fizz 13 14 FizzBuzz 16 17 Fizz 19 Buzz
```

### ข้อ 3: วันในสัปดาห์ภาษาไทย
เขียน method ที่รับ Integer 1-7 (1=จันทร์) และคืนชื่อวันภาษาไทยพร้อมสีประจำวัน

```ruby
# เฉลย
def day_info(day_number)
  case day_number
  when 1 then { name: "จันทร์", color: "เหลือง" }
  when 2 then { name: "อังคาร", color: "ชมพู" }
  when 3 then { name: "พุธ", color: "เขียว" }
  when 4 then { name: "พฤหัสบดี", color: "ส้ม" }
  when 5 then { name: "ศุกร์", color: "ฟ้า" }
  when 6 then { name: "เสาร์", color: "ม่วง" }
  when 7 then { name: "อาทิตย์", color: "แดง" }
  else { name: "ไม่ทราบ", color: "ไม่ทราบ" }
  end
end

(1..7).each do |d|
  info = day_info(d)
  puts "วัน#{info[:name]}: สี#{info[:color]}"
end
```

### ข้อ 4: ตรวจสอบ Palindrome
เขียน method ที่ตรวจสอบว่า string เป็น palindrome (อ่านหน้าหลังเหมือนกัน) หรือไม่

```ruby
# เฉลย
def palindrome?(str)
  return false unless str.is_a?(String)
  cleaned = str.downcase.gsub(/[^a-z0-9ก-ฮ]/, "")
  cleaned == cleaned.reverse
end

puts palindrome?("racecar")   # true
puts palindrome?("hello")     # false
puts palindrome?("A man a plan a canal Panama".gsub(" ", "").downcase)  # true
```

### ข้อ 5: ระบบ ATM อย่างง่าย
เขียน class ATM ที่มี method withdraw(amount) ที่ตรวจสอบเงื่อนไขต่างๆ

```ruby
# เฉลย
class ATM
  def initialize(balance)
    @balance = balance
  end

  def withdraw(amount)
    return "จำนวนเงินต้องเป็นบวก" unless amount > 0
    return "จำนวนเงินต้องเป็นทวีคูณของ 100" unless amount % 100 == 0
    return "จำนวนเงินเกินวงเงินต่อครั้ง (20,000)" if amount > 20_000
    return "ยอดเงินไม่เพียงพอ" if amount > @balance
    
    @balance -= amount
    "ถอนเงิน #{amount} บาท สำเร็จ ยอดคงเหลือ: #{@balance} บาท"
  end
end

atm = ATM.new(5000)
puts atm.withdraw(1000)   # ถอนเงิน 1000 บาท สำเร็จ ยอดคงเหลือ: 4000 บาท
puts atm.withdraw(150)    # จำนวนเงินต้องเป็นทวีคูณของ 100
puts atm.withdraw(5000)   # ยอดเงินไม่เพียงพอ
```

### ข้อ 6–10: โจทย์ปฏิบัติ

```ruby
# ข้อ 6: ตรวจสอบปีอธิกสุรทิน (Leap Year)
def leap_year?(year)
  return false unless year.is_a?(Integer) && year > 0
  (year % 400 == 0) || (year % 4 == 0 && year % 100 != 0)
end

puts leap_year?(2000)  # true
puts leap_year?(1900)  # false
puts leap_year?(2024)  # true

# ข้อ 7: คำนวณค่า Shipping
def shipping_cost(weight_kg, distance_km)
  return "น้ำหนักไม่ถูกต้อง" unless weight_kg > 0
  return "ระยะทางไม่ถูกต้อง" unless distance_km > 0
  
  base_cost = case weight_kg
              when 0...1   then 30
              when 1...5   then 50
              when 5...20  then 80
              else              150
              end
  
  distance_multiplier = case distance_km
                        when 0...50     then 1.0
                        when 50...200   then 1.5
                        when 200...500  then 2.0
                        else                 3.0
                        end
  
  (base_cost * distance_multiplier).round(2)
end

puts shipping_cost(0.5, 30)   # 30.0
puts shipping_cost(3, 100)    # 75.0
puts shipping_cost(10, 300)   # 160.0

# ข้อ 8: แปลงอุณหภูมิ
def convert_temperature(value, from:, to:)
  celsius = case from
            when :celsius    then value
            when :fahrenheit then (value - 32) * 5.0 / 9
            when :kelvin     then value - 273.15
            else return "หน่วยต้นทางไม่รู้จัก"
            end
  
  case to
  when :celsius    then celsius.round(2)
  when :fahrenheit then (celsius * 9.0 / 5 + 32).round(2)
  when :kelvin     then (celsius + 273.15).round(2)
  else "หน่วยปลายทางไม่รู้จัก"
  end
end

puts convert_temperature(100, from: :celsius, to: :fahrenheit)  # 212.0
puts convert_temperature(32, from: :fahrenheit, to: :celsius)   # 0.0
puts convert_temperature(0, from: :celsius, to: :kelvin)        # 273.15

# ข้อ 9: ตรวจสอบรหัสผ่าน
def password_strength(password)
  return "ต้องระบุรหัสผ่าน" if password.nil? || password.empty?
  
  score = 0
  score += 1 if password.length >= 8
  score += 1 if password.length >= 12
  score += 1 if password.match?(/[A-Z]/)
  score += 1 if password.match?(/[a-z]/)
  score += 1 if password.match?(/\d/)
  score += 1 if password.match?(/[!@#$%^&*]/)
  
  case score
  when 0..2 then "อ่อนแอมาก"
  when 3    then "อ่อนแอ"
  when 4    then "ปานกลาง"
  when 5    then "แข็งแกร่ง"
  when 6    then "แข็งแกร่งมาก"
  end
end

puts password_strength("abc")              # อ่อนแอมาก
puts password_strength("abcdefgh")         # อ่อนแอ
puts password_strength("Abcdefgh1")        # ปานกลาง
puts password_strength("Abcdefgh1!")       # แข็งแกร่ง
puts password_strength("Ab1!cdefghijkl")  # แข็งแกร่งมาก

# ข้อ 10: Roman Numerals
def to_roman(number)
  return "ตัวเลขต้องอยู่ระหว่าง 1-3999" unless (1..3999).cover?(number)
  
  values = [[1000, "M"], [900, "CM"], [500, "D"], [400, "CD"],
            [100, "C"], [90, "XC"], [50, "L"], [40, "XL"],
            [10, "X"], [9, "IX"], [5, "V"], [4, "IV"], [1, "I"]]
  
  result = ""
  values.each do |value, numeral|
    while number >= value
      result += numeral
      number -= value
    end
  end
  result
end

puts to_roman(2024)  # MMXXIV
puts to_roman(1999)  # MCMXCIX
puts to_roman(42)    # XLII
```

### ข้อ 11–20: โจทย์ขั้นสูง

```ruby
# ข้อ 11: Traffic Light System
def next_traffic_light(current)
  case current
  when :red    then :green
  when :green  then :yellow
  when :yellow then :red
  else raise ArgumentError, "ไฟจราจรไม่รู้จัก: #{current}"
  end
end

light = :red
5.times do
  puts "ไฟ: #{light}"
  light = next_traffic_light(light)
end

# ข้อ 12: JSON-like Data Processor
def extract_value(data, path)
  return nil unless data && path
  
  parts = path.split(".")
  current = data
  
  parts.each do |part|
    case current
    when Hash   then current = current[part.to_sym] || current[part]
    when Array  then current = current[part.to_i]
    else return nil
    end
    return nil unless current
  end
  
  current
end

data = {
  user: {
    name: "Alice",
    address: {
      city: "กรุงเทพ",
      zip: "10100"
    },
    scores: [95, 87, 92]
  }
}

puts extract_value(data, "user.name")            # Alice
puts extract_value(data, "user.address.city")    # กรุงเทพ
puts extract_value(data, "user.scores.0")        # 95
puts extract_value(data, "user.phone")           # nil

# ข้อ 13: Currency Converter
EXCHANGE_RATES = {
  THB: 1.0,
  USD: 0.028,
  EUR: 0.026,
  JPY: 4.2,
  GBP: 0.022
}

def convert_currency(amount, from:, to:)
  return "สกุลเงินต้นทางไม่รู้จัก" unless EXCHANGE_RATES.key?(from)
  return "สกุลเงินปลายทางไม่รู้จัก" unless EXCHANGE_RATES.key?(to)
  return "จำนวนเงินต้องเป็นบวก" unless amount > 0
  
  in_thb = amount / EXCHANGE_RATES[from]
  result = in_thb * EXCHANGE_RATES[to]
  result.round(4)
end

puts convert_currency(1000, from: :THB, to: :USD)  # 28.0
puts convert_currency(100, from: :USD, to: :THB)   # 3571.4286
puts convert_currency(1, from: :USD, to: :EUR)      # 0.9286

# ข้อ 14-20: โจทย์ให้ผู้เรียนลองทำเอง
# ข้อ 14: สร้าง Morse Code converter
# ข้อ 15: เขียน method ตรวจสอบ Credit Card number (Luhn algorithm)
# ข้อ 16: สร้าง simple calculator ที่รับ expression เป็น string เช่น "5 + 3"
# ข้อ 17: เขียน Binary Search ที่ใช้ pattern matching
# ข้อ 18: สร้าง Discount calculator ที่มีหลายประเภทส่วนลด
# ข้อ 19: เขียน method แปลงตัวเลขเป็นภาษาไทย (1 = หนึ่ง, 2 = สอง, ...)
# ข้อ 20: สร้าง simple State Machine ด้วย case/when
```

---

## สรุป Control Flow ใน Ruby

| คำสั่ง | การใช้งาน |
|--------|----------|
| `if/elsif/else` | เงื่อนไขพื้นฐาน |
| `unless` | เงื่อนไขแบบ negation |
| `? :` | ternary สำหรับ simple condition |
| `statement if cond` | one-line condition |
| `case/when` | multiple conditions |
| `case/in` | pattern matching (Ruby 3+) |
| `&&, \|\|, !` | logical operators |
| `&.` | safe navigation |
| `\|\|=` | conditional assignment |

**หลักการที่ควรจำ:**
1. ใน Ruby มีแค่ `false` และ `nil` ที่เป็น falsy
2. ใช้ Guard Clauses เพื่อลด nesting
3. `case/when` ใช้ `===` ที่ยืดหยุ่น (range, regex, class)
4. `&.` ช่วยหลีกเลี่ยง nil errors
5. Short-circuit evaluation ช่วยเพิ่ม performance

> ⬅️ [ตอนที่ 6: Hashes](part-06-hashes.md) | ➡️ [ตอนที่ 8: Loops](part-08-loops.md)
