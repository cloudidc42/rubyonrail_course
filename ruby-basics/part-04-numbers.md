# ตอนที่ 4: Numbers และ Math

## Ruby Programming Course สำหรับผู้เริ่มต้นภาษาไทย

---

## บทนำ

ในตอนที่ 4 เราจะเรียนรู้เกี่ยวกับตัวเลขและการคำนวณใน Ruby อย่างละเอียด Ruby มีการรองรับตัวเลขหลายประเภทและมี Math module ที่ทรงพลัง

**สิ่งที่จะได้เรียนรู้:**
- Integer และ Float อย่างละเอียด
- การดำเนินการทางคณิตศาสตร์
- Math module
- BigDecimal สำหรับการเงิน
- เลขฐานต่างๆ
- Random numbers
- Numeric Conversions

---

## Step 46: Integer อย่างละเอียด

### 46.1 Integer Literals

```ruby
# ตัวเลขจำนวนเต็มธรรมดา
age = 25
negative = -100
zero = 0

# ใช้ underscore แทน comma เพื่อให้อ่านง่าย
population = 70_000_000     # 70 ล้าน
distance   = 384_400        # ระยะทางไปดวงจันทร์ กม.
bytes      = 1_073_741_824  # 1 GiB

puts population  # => 70000000
puts distance    # => 384400
puts bytes       # => 1073741824

# Integer ฐานต่างๆ
binary  = 0b1010_1010   # ฐาน 2 = 170
octal   = 0o377         # ฐาน 8 = 255
decimal = 255           # ฐาน 10
hex     = 0xFF          # ฐาน 16 = 255

puts binary   # => 170
puts octal    # => 255
puts hex      # => 255
```

```ruby
# Ruby Integer ไม่มีขีดจำกัด (Arbitrary Precision)
# ใน Ruby เก่า: Fixnum (<= 2^62-1) และ Bignum (ใหญ่กว่า)
# ใน Ruby 2.4+: รวมเป็น Integer เดียว

puts 2**100
# => 1267650600228229401496703205376

factorial = 1
(1..20).each { |n| factorial *= n }
puts factorial  # => 2432902008176640000

# ตรวจสอบ class
puts 42.class          # => Integer
puts (2**100).class    # => Integer
```

### 46.2 Integer Methods

```ruby
n = -42

# ค่าสัมบูรณ์
puts n.abs    # => 42

# เครื่องหมาย
puts n.sign rescue nil  # ไม่มี method นี้
puts n.negative?  # => true
puts n.positive?  # => false
puts n.zero?      # => false
puts n.nonzero?   # => -42 (คืนค่าตัวเองถ้าไม่ใช่ 0, คืน nil ถ้า 0)
puts 0.nonzero?.inspect  # => nil

# Ceiling Division
puts 7.ceildiv(3)   # => 3 (Ruby 3.2+, เหมือน (7.0/3).ceil)

# นับ bits
puts 255.bit_length  # => 8
puts 256.bit_length  # => 9
puts 1.bit_length    # => 1
```

```ruby
# Integer Arithmetic
a = 17
b = 5

puts a + b    # => 22 (addition)
puts a - b    # => 12 (subtraction)
puts a * b    # => 85 (multiplication)
puts a / b    # => 3  (integer division - ตัดทศนิยม!)
puts a % b    # => 2  (modulo - เศษ)
puts a ** b   # => 1419857 (exponentiation)
puts -a % b   # => 3  (ระวัง: -17 % 5 = 3 ใน Ruby)

# divmod - ได้ทั้ง quotient และ remainder
quotient, remainder = 17.divmod(5)
puts "#{quotient}, #{remainder}"  # => 3, 2

# remainder vs modulo
puts (-7).remainder(3)  # => -1 (เก็บเครื่องหมาย dividend)
puts (-7) % 3           # => 2  (ผลลัพธ์เป็น positive เสมอ)
```

```ruby
# Integer ที่ใช้บ่อย
# GCD (Greatest Common Divisor / ห.ร.ม.)
puts 12.gcd(8)    # => 4
puts 15.gcd(10)   # => 5
puts 100.gcd(75)  # => 25

# LCM (Least Common Multiple / ค.ร.น.)
puts 4.lcm(6)    # => 12
puts 3.lcm(5)    # => 15
puts 12.lcm(18)  # => 36

# กำลังสอง
puts 9.integer_sqrt rescue nil  # ไม่มี
require 'isqrt' rescue nil
puts Math.sqrt(9).to_i   # => 3 (ทางอ้อม)
puts Integer.sqrt(9)     # => 3 (Ruby 2.5+)
puts Integer.sqrt(10)    # => 3 (floor square root)
puts Integer.sqrt(16)    # => 4
```

---

## Step 47: Float อย่างละเอียด

### 47.1 Float Literals

```ruby
# Float literals
pi    = 3.14159265358979
e     = 2.71828182845905
price = 29.99
small = 0.001
large = 1.5e10   # 1.5 × 10^10

puts pi     # => 3.14159265358979
puts small  # => 0.001
puts large  # => 15000000000.0

# Scientific notation
puts 1.5e10   # => 15000000000.0
puts 1.5e-3   # => 0.0015
puts 2.5E8    # => 250000000.0

# Float precision
puts 1.0 / 3.0   # => 0.3333333333333333
puts (1.0 / 3.0) * 3  # => 1.0
```

### 47.2 Float Constants

```ruby
puts Float::INFINITY        # => Infinity
puts -Float::INFINITY       # => -Infinity
puts Float::NAN             # => NaN
puts Float::DIG             # => 15 (decimal digits of precision)
puts Float::EPSILON         # => 2.220446049250313e-16 (smallest diff from 1.0)
puts Float::MAX             # => 1.7976931348623157e+308
puts Float::MIN             # => 5.0e-324 (smallest positive)
puts Float::MANT_DIG        # => 53 (mantissa digits)
puts Float::MAX_EXP         # => 1024
puts Float::MIN_EXP         # => -1021
```

### 47.3 Float Methods

```ruby
num = 3.75

# Rounding
puts num.ceil      # => 4 (ปัดขึ้น)
puts num.floor     # => 3 (ปัดลง)
puts num.round     # => 4 (ปัดใกล้สุด)
puts num.truncate  # => 3 (ตัดทศนิยม)

# ปัดหลายตำแหน่ง
pi = 3.14159
puts pi.ceil(2)      # => 3.15
puts pi.floor(2)     # => 3.14
puts pi.round(2)     # => 3.14
puts pi.round(4)     # => 3.1416
puts pi.truncate(3)  # => 3.141

# Checking
puts 1.0.finite?         # => true
puts (1.0/0).finite?     # => false
puts Float::NAN.nan?     # => true
puts 1.0.nan?            # => false
puts (1.0/0).infinite?   # => 1
puts (-1.0/0).infinite?  # => -1
puts 1.0.infinite?       # => nil
```

---

## Step 48: Arithmetic Operations

### 48.1 Basic Operations

```ruby
# Integer Operations
puts 10 + 3    # => 13
puts 10 - 3    # => 7
puts 10 * 3    # => 30
puts 10 / 3    # => 3 (Integer division!)
puts 10 % 3    # => 1
puts 10 ** 3   # => 1000

# Float Operations
puts 10.0 + 3  # => 13.0
puts 10.0 - 3  # => 7.0
puts 10.0 * 3  # => 30.0
puts 10.0 / 3  # => 3.3333333333333335
puts 10.0 % 3  # => 1.0
puts 10.0 ** 3 # => 1000.0
```

```ruby
# Compound Assignment Operators
x = 10
x += 5   # x = x + 5  => 15
x -= 3   # x = x - 3  => 12
x *= 2   # x = x * 2  => 24
x /= 4   # x = x / 4  => 6
x %= 4   # x = x % 4  => 2
x **= 3  # x = x ** 3 => 8

puts x  # => 8

# Integer ไม่มี ++ และ -- (เหมือน C/Java)
# ต้องใช้ += 1 หรือ -= 1
count = 0
count += 1
count += 1
puts count  # => 2
```

### 48.2 Integer vs Float Division

```ruby
# Integer Division (ตัดทศนิยม - Truncation)
puts 7 / 2     # => 3 (ไม่ใช่ 3.5!)
puts -7 / 2    # => -4 (Ruby rounds toward -infinity)
puts 7 / -2    # => -4

# Float Division
puts 7.0 / 2   # => 3.5
puts 7 / 2.0   # => 3.5
puts 7.0 / 2.0 # => 3.5

# แปลงเป็น Float ก่อนหาร
a = 7
b = 2
puts a.to_f / b  # => 3.5
puts a / b.to_f  # => 3.5

# Rational (เศษส่วน)
puts 7.to_r / 2   # => 7/2 (Rational)
puts (7.to_r / 2).to_f  # => 3.5
```

```ruby
# ระวัง Integer Division Bug
total_price = 100  # บาท
num_items   = 3

# ผิด
avg_price_wrong = total_price / num_items
puts avg_price_wrong  # => 33 (ผิด!)

# ถูก
avg_price_correct = total_price.to_f / num_items
puts avg_price_correct  # => 33.333...

# หรือ
avg_price_float = total_price / num_items.to_f
puts avg_price_float  # => 33.333...

# ตัวอย่างจริง: คำนวณเปอร์เซ็นต์
correct_answers = 17
total_questions = 20

# ผิด
percent_wrong = correct_answers / total_questions * 100
puts percent_wrong  # => 80 (อาจถูกต้องบังเอิญ แต่ไม่น่าเชื่อถือ)

# ถูก
percent_correct = correct_answers.to_f / total_questions * 100
puts percent_correct  # => 85.0

# หรือ
percent_reliable = (correct_answers * 100) / total_questions
puts percent_reliable  # => 85 (Integer แต่แม่นยำ)
```

---

## Step 49: Math Module

### 49.1 Math Constants

```ruby
puts Math::PI   # => 3.141592653589793
puts Math::E    # => 2.718281828459045

# ใช้ include Math เพื่อไม่ต้องพิมพ์ Math::
include Math
puts PI         # => 3.141592653589793
puts E          # => 2.718281828459045
```

### 49.2 Math Methods

```ruby
# Square Root
puts Math.sqrt(9)      # => 3.0
puts Math.sqrt(2)      # => 1.4142135623730951
puts Math.sqrt(0)      # => 0.0
puts Math.sqrt(0.25)   # => 0.5

# Cube Root (ไม่มี method ตรงๆ)
puts 8 ** (1.0/3)    # => 2.0
puts 27 ** (1.0/3)   # => 3.0 (หรือ 2.9999...)

# Exponent
puts Math.exp(1)    # => 2.718281828459045 (e^1)
puts Math.exp(2)    # => 7.38905609893065  (e^2)
puts Math.exp(0)    # => 1.0               (e^0)

# Logarithm
puts Math.log(Math::E)   # => 1.0 (natural log)
puts Math.log(1)         # => 0.0
puts Math.log(100, 10)   # => 2.0 (log base 10)
puts Math.log10(100)     # => 2.0 (log base 10)
puts Math.log2(8)        # => 3.0 (log base 2)
puts Math.log2(1024)     # => 10.0
```

```ruby
# Trigonometry (angles in radians)
puts Math.sin(0)               # => 0.0
puts Math.sin(Math::PI / 2)    # => 1.0
puts Math.cos(0)               # => 1.0
puts Math.cos(Math::PI)        # => -1.0
puts Math.tan(Math::PI / 4)    # => 0.9999... (≈1.0)

# Inverse Trig
puts Math.asin(1)   # => π/2 = 1.5707963...
puts Math.acos(1)   # => 0.0
puts Math.atan(1)   # => π/4 = 0.7853981...
puts Math.atan2(1, 1)  # => π/4 (atan with x,y)

# Convert Degrees to Radians
def deg_to_rad(degrees)
  degrees * Math::PI / 180
end

def rad_to_deg(radians)
  radians * 180 / Math::PI
end

puts Math.sin(deg_to_rad(30)).round(4)  # => 0.5
puts Math.cos(deg_to_rad(60)).round(4)  # => 0.5
puts rad_to_deg(Math::PI).round(2)      # => 180.0
```

```ruby
# Hyperbolic Functions
puts Math.sinh(1)   # => 1.1752011936438014
puts Math.cosh(1)   # => 1.5430806348152437
puts Math.tanh(0)   # => 0.0
puts Math.tanh(1)   # => 0.7615941559557649

# Power and Roots
puts Math.hypot(3, 4)   # => 5.0 (hypotenuse: sqrt(3^2 + 4^2))
puts Math.hypot(5, 12)  # => 13.0

# Ceiling/Floor for float in different contexts
puts Math.cbrt(27)   # => 3.0 (cube root, Ruby 2.4+) หรือ 27**(1.0/3)

# Frexp and Ldexp (Float decomposition)
mantissa, exponent = Math.frexp(0.5)
puts "mantissa=#{mantissa}, exp=#{exponent}"  # => 0.5, 0

puts Math.ldexp(0.5, 0)  # => 0.5
```

---

## Step 50: Number Formatting

### 50.1 Rounding

```ruby
num = 3.14159265358979

# round - ปัดเศษ
puts num.round      # => 3 (ปัดเป็น Integer)
puts num.round(0)   # => 3.0 (ปัดเป็น Float ทศนิยม 0 ตำแหน่ง)
puts num.round(1)   # => 3.1
puts num.round(2)   # => 3.14
puts num.round(3)   # => 3.142
puts num.round(4)   # => 3.1416

# ceil - ปัดขึ้น
puts 3.1.ceil    # => 4
puts 3.9.ceil    # => 4
puts (-3.1).ceil # => -3 (ปัดขึ้นสำหรับ negative = ใกล้ 0)

# floor - ปัดลง
puts 3.9.floor    # => 3
puts (-3.1).floor # => -4

# truncate - ตัดทศนิยม
puts 3.9.truncate   # => 3
puts (-3.9).truncate # => -3 (แตกต่างจาก floor)

# round ด้วย half: mode
puts 2.5.round                    # => 3 (default: half up)
puts 2.5.round(half: :up)         # => 3
puts 2.5.round(half: :down)       # => 2
puts 2.5.round(half: :even)       # => 2 (Banker's rounding)
puts 3.5.round(half: :even)       # => 4 (ปัดเป็นเลขคู่)
```

```ruby
# Format ตัวเลขสำหรับแสดงผล
def format_currency(amount, symbol = "฿")
  "#{symbol}#{"%.2f" % amount}"
end

def format_number_with_commas(number)
  # แบบง่าย (สำหรับ Integer)
  number.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\1,').reverse
end

puts format_currency(1234.5)     # => ฿1234.50
puts format_currency(0.99)       # => ฿0.99
puts format_number_with_commas(1234567)  # => 1,234,567
puts format_number_with_commas(1234)     # => 1,234

# ใช้ sprintf
puts sprintf("%.2f", 3.14159)       # => 3.14
puts sprintf("%10.2f", 3.14)        # => "      3.14"
puts sprintf("%-10.2f", 3.14)       # => "3.14      "
puts sprintf("%+.2f", 3.14)         # => +3.14
puts sprintf("%+.2f", -3.14)        # => -3.14
```

---

## Step 51: Integer Methods

### 51.1 Odd, Even, Times

```ruby
# Odd และ Even
puts 5.odd?    # => true
puts 4.odd?    # => false
puts 4.even?   # => true
puts 5.even?   # => false
puts 0.even?   # => true
puts 0.odd?    # => false

# times - วนซ้ำ n ครั้ง
5.times { |i| print "#{i} " }  # => 0 1 2 3 4
puts

3.times { print "Ruby! " }  # => Ruby! Ruby! Ruby!
puts

# upto - วนจาก n ถึง m
1.upto(5) { |i| print "#{i} " }   # => 1 2 3 4 5
puts

# downto - วนจาก n ลงถึง m
5.downto(1) { |i| print "#{i} " }  # => 5 4 3 2 1
puts

# step - วนด้วย increment
1.step(10, 2) { |i| print "#{i} " }   # => 1 3 5 7 9
puts

10.step(1, -3) { |i| print "#{i} " }  # => 10 7 4 1
puts
```

```ruby
# abs - ค่าสัมบูรณ์
puts (-42).abs   # => 42
puts 42.abs      # => 42
puts 0.abs       # => 0

# digits - แยกหลัก (จากขวาไปซ้าย)
puts 12345.digits.inspect     # => [5, 4, 3, 2, 1]
puts 12345.digits(10).inspect # => [5, 4, 3, 2, 1]

# digits ฐานต่างๆ
puts 255.digits(16).inspect   # => [15, 15] (0xFF)
puts 8.digits(2).inspect      # => [0, 0, 0, 1] (0b1000)

# to_s ฐานต่างๆ
puts 255.to_s(2)   # => "11111111" (binary)
puts 255.to_s(16)  # => "ff" (hex)
puts 255.to_s(8)   # => "377" (octal)
```

```ruby
# pow - ยกกำลัง
puts 2.pow(10)     # => 1024
puts 2 ** 10       # => 1024 (เหมือนกัน)

# pow with modulo (modular exponentiation - ใช้ใน cryptography)
puts 2.pow(10, 1000)  # => 24 (2^10 mod 1000)

# Integer.sqrt (Ruby 2.5+)
puts Integer.sqrt(9)    # => 3
puts Integer.sqrt(10)   # => 3 (floor)
puts Integer.sqrt(100)  # => 10
puts Integer.sqrt(0)    # => 0
```

---

## Step 52: Float Methods

### 52.1 NaN, Infinite, Finite

```ruby
# สร้าง Special Float values
infinity     = Float::INFINITY
neg_infinity = -Float::INFINITY
not_a_number = Float::NAN
regular      = 3.14

# nan?
puts not_a_number.nan?  # => true
puts infinity.nan?      # => false
puts regular.nan?       # => false
puts (0.0/0.0).nan?     # => true

# infinite?
puts infinity.infinite?      # => 1
puts neg_infinity.infinite?  # => -1
puts regular.infinite?       # => nil
puts (1.0/0.0).infinite?     # => 1

# finite?
puts regular.finite?         # => true
puts infinity.finite?        # => false
puts not_a_number.finite?    # => false
```

```ruby
# Float Comparison Edge Cases
nan = Float::NAN

# NaN ไม่เท่ากับอะไร รวมถึงตัวเอง!
puts nan == nan    # => false
puts nan != nan    # => true
puts nan < 1       # => false
puts nan > 1       # => false

# ตรวจสอบ NaN
puts nan.nan?      # => true

# Infinity Arithmetic
inf = Float::INFINITY
puts inf + 1       # => Infinity
puts inf + inf     # => Infinity
puts inf - inf     # => NaN
puts inf * 2       # => Infinity
puts inf / inf     # => NaN
puts 1 / inf       # => 0.0
puts inf * 0       # => NaN
```

```ruby
# Float เปรียบเทียบ
a = 0.1 + 0.2
b = 0.3

puts a == b         # => false! (floating point issue)
puts a.round(10) == b.round(10)  # => true
puts (a - b).abs < Float::EPSILON * 10  # => true

# ฟังก์ชัน compare_float
def float_equal?(a, b, tolerance = Float::EPSILON * 100)
  (a - b).abs <= tolerance
end

puts float_equal?(0.1 + 0.2, 0.3)  # => true
```

---

## Step 53: BigDecimal สำหรับการเงิน

### 53.1 BigDecimal Basics

```ruby
require 'bigdecimal'
require 'bigdecimal/util'  # เพิ่ม .to_d method

# สร้าง BigDecimal
price1 = BigDecimal("19.99")
price2 = BigDecimal("5.01")
tax_rate = BigDecimal("0.07")

puts price1 + price2   # => 0.25e2 (25.0)
puts price1 * tax_rate # => 0.13993e1 (1.3993)

# แสดงผลที่อ่านง่าย
total = price1 + price2
puts total.to_s("F")   # => 25.0 (Fixed notation)
puts total.to_f        # => 25.0

tax = price1 * tax_rate
puts tax.to_s("F")     # => 1.3993
```

```ruby
# ทำไมต้องใช้ BigDecimal สำหรับการเงิน?
# ปัญหาของ Float
puts 0.1 + 0.2 == 0.3  # => false!

# BigDecimal แก้ปัญหา
a = BigDecimal("0.1")
b = BigDecimal("0.2")
c = BigDecimal("0.3")

puts a + b == c  # => true!
puts (a + b).to_s("F")  # => 0.3

# การคำนวณราคา
item_price = BigDecimal("99.95")
quantity   = 3
tax_rate   = BigDecimal("0.07")
discount   = BigDecimal("0.1")  # 10%

subtotal = item_price * quantity
discounted = subtotal * (1 - discount)
tax = discounted * tax_rate
total = discounted + tax

puts "ราคาต่อชิ้น: #{item_price.to_s("F")}"
puts "จำนวน: #{quantity}"
puts "ก่อนส่วนลด: #{subtotal.to_s("F")}"
puts "หลังส่วนลด 10%: #{discounted.to_s("F")}"
puts "ภาษี 7%: #{tax.to_s("F")}"
puts "รวมทั้งสิ้น: #{total.to_s("F")}"
```

```ruby
# BigDecimal Rounding Modes
require 'bigdecimal'

amount = BigDecimal("2.5")

puts amount.round(0, BigDecimal::ROUND_UP)        # => 3
puts amount.round(0, BigDecimal::ROUND_DOWN)      # => 2
puts amount.round(0, BigDecimal::ROUND_CEILING)   # => 3
puts amount.round(0, BigDecimal::ROUND_FLOOR)     # => 2
puts amount.round(0, BigDecimal::ROUND_HALF_UP)   # => 3
puts amount.round(0, BigDecimal::ROUND_HALF_DOWN) # => 2
puts amount.round(0, BigDecimal::ROUND_HALF_EVEN) # => 2 (Banker's rounding)

# ใช้ .to_d จาก bigdecimal/util
"99.99".to_d   # => BigDecimal("99.99")
99.99.to_d     # => BigDecimal (จาก Float - ระวัง precision)
puts "99.99".to_d.to_s("F")  # => 99.99 (แม่นยำ)
puts 99.99.to_d.to_s("F")    # => 99.989999... (ไม่แม่นยำ!)
```

---

## Step 54: Number Bases

### 54.1 Binary, Octal, Hexadecimal

```ruby
# Literals
binary  = 0b1010_1100   # ฐาน 2
octal   = 0o377         # ฐาน 8
decimal = 172           # ฐาน 10
hex     = 0xAC          # ฐาน 16

# ทั้งหมดเท่ากับ 172
puts binary   # => 172
puts octal    # => 255 (ไม่เท่ากัน! 0o377 = 255)
puts decimal  # => 172
puts hex      # => 172

# แปลงเป็นฐานต่างๆ
n = 255
puts n.to_s(2)   # => "11111111" (binary)
puts n.to_s(8)   # => "377" (octal)
puts n.to_s(10)  # => "255" (decimal)
puts n.to_s(16)  # => "ff" (hex)
puts n.to_s(36)  # => "73" (base 36)

# กลับจาก String
puts "11111111".to_i(2)   # => 255 (from binary)
puts "377".to_i(8)        # => 255 (from octal)
puts "ff".to_i(16)        # => 255 (from hex)
puts "FF".to_i(16)        # => 255 (case insensitive)
```

```ruby
# Bitwise Operations
a = 0b1010  # 10
b = 0b1100  # 12

puts (a & b).to_s(2)   # => "1000" (AND) = 8
puts (a | b).to_s(2)   # => "1110" (OR)  = 14
puts (a ^ b).to_s(2)   # => "0110" (XOR) = 6
puts (~a)              # => -11 (NOT, two's complement)

# Bit Shifting
puts (1 << 4)   # => 16 (เลื่อนซ้าย 4 bit = คูณ 2^4)
puts (16 >> 4)  # => 1  (เลื่อนขวา 4 bit = หาร 2^4)

# ใช้ Bitwise ในทางปฏิบัติ
# Flags
READ    = 0b001  # 1
WRITE   = 0b010  # 2
EXECUTE = 0b100  # 4

permissions = READ | WRITE  # 3
puts "Can read?   #{(permissions & READ) != 0}"     # => true
puts "Can write?  #{(permissions & WRITE) != 0}"    # => true
puts "Can execute? #{(permissions & EXECUTE) != 0}" # => false

# เพิ่ม permission
permissions |= EXECUTE
puts "Can execute now? #{(permissions & EXECUTE) != 0}"  # => true
```

---

## Step 55: Random Numbers

### 55.1 rand

```ruby
# rand ไม่มี argument - Float ระหว่าง 0.0 และ 1.0
puts rand           # => 0.5488135 (random)

# rand(n) - Integer ระหว่าง 0 และ n-1
puts rand(10)       # => 0 ถึง 9
puts rand(100)      # => 0 ถึง 99

# rand(range) - ตัวเลขใน range
puts rand(1..6)     # => 1 ถึง 6 (dice)
puts rand(1...6)    # => 1 ถึง 5 (exclusive end)
puts rand(1.0..2.0) # => Float ระหว่าง 1.0 ถึง 2.0

# ตัวอย่างการใช้งาน
def roll_dice(sides = 6)
  rand(1..sides)
end

puts roll_dice        # => 1-6
puts roll_dice(20)    # => 1-20 (D&D style)

# สุ่ม element จาก Array
fruits = ["แอปเปิ้ล", "กล้วย", "มะม่วง", "ส้ม"]
puts fruits.sample           # => random fruit
puts fruits.sample(2).inspect  # => 2 random fruits
```

```ruby
# Random.new - สร้าง Random object ด้วย seed
rng = Random.new(42)  # seed 42
puts rng.rand         # => เสมอได้ค่าเดิมถ้า seed เดิม
puts rng.rand(100)    # => เสมอได้ค่าเดิมถ้า seed เดิม
puts rng.rand(1..10)  # => เสมอได้ค่าเดิมถ้า seed เดิม

# ใช้ seed สำหรับ reproducible results (testing)
rng1 = Random.new(42)
rng2 = Random.new(42)

5.times do
  puts "#{rng1.rand(100)} == #{rng2.rand(100)}: #{rng1.rand(100) == rng2.rand(100)}"
end

# srand - ตั้ง global seed
srand(42)
puts rand(100)  # => เสมอได้ค่าเดิมเมื่อ seed เดิม

srand          # reset to random seed
puts rand(100) # => random
```

```ruby
# การสุ่มจาก Distribution ต่างๆ
# Uniform distribution (rand ปกติ)
# ใน Ruby built-in ไม่มี Gaussian โดยตรง แต่สร้างได้

def gaussian_rand(mean = 0, std_dev = 1)
  # Box-Muller transform
  u1 = rand
  u2 = rand
  z0 = Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math::PI * u2)
  z0 * std_dev + mean
end

# สร้างตัวเลขที่กระจายตาม Normal Distribution
samples = 1000.times.map { gaussian_rand(0, 1) }
mean = samples.sum / samples.length
puts "Mean: #{mean.round(2)}"  # ≈ 0
puts "StdDev: #{Math.sqrt(samples.map { |x| (x - mean)**2 }.sum / samples.length).round(2)}"  # ≈ 1
```

---

## Step 56: Numeric Conversions

### 56.1 การแปลงระหว่างประเภท

```ruby
# to_i - แปลงเป็น Integer
puts "42".to_i       # => 42
puts "3.14".to_i     # => 3 (ตัดทศนิยม)
puts "0xFF".to_i(16) # => 255
puts "0b1010".to_i(2) # => 10 (หรือ "1010".to_i(2))
puts 3.14.to_i       # => 3
puts nil.to_i        # => 0
puts true.to_i rescue puts "ไม่สามารถแปลงได้"

# to_f - แปลงเป็น Float
puts "3.14".to_f     # => 3.14
puts "42".to_f       # => 42.0
puts nil.to_f        # => 0.0
puts 42.to_f         # => 42.0

# to_r - แปลงเป็น Rational
puts "3/4".to_r      # => 3/4
puts 0.5.to_r        # => 1/2
puts 42.to_r         # => 42/1
puts 3.14.to_r       # => 7070651414971679/2251799813685248 (exact)
puts 3.14.rationalize(0.001)  # => 22/7 (approximate)

# to_c - แปลงเป็น Complex
puts 42.to_c         # => 42+0i
puts 3.14.to_c       # => 3.14+0i
puts "3+4i".to_c     # => 3+4i
```

```ruby
# Integer() และ Float() - Strict Conversion
puts Integer("42")      # => 42
puts Integer("0xFF")    # => 255
puts Integer("0b1010")  # => 10
puts Float("3.14")      # => 3.14

begin
  Integer("abc")
rescue ArgumentError => e
  puts "Error: #{e.message}"  # invalid value for Integer()
end

begin
  Float("abc")
rescue ArgumentError => e
  puts "Error: #{e.message}"  # invalid value for Float()
end

# Kernel#Integer vs to_i
puts "42abc".to_i       # => 42 (ไม่ error)
Integer("42abc") rescue puts "Error!"  # => Error!
```

```ruby
# Rational Numbers
r1 = Rational(1, 3)    # 1/3
r2 = Rational(1, 6)    # 1/6

puts r1 + r2    # => 1/2
puts r1 * r2    # => 1/18
puts r1 / r2    # => 2/1
puts r1.to_f    # => 0.3333...
puts r2.to_f    # => 0.1666...

# สำหรับการคำนวณที่ต้องการความแม่นยำ
puts Rational(1, 3) + Rational(1, 3) + Rational(1, 3)  # => 1/1 (= 1 แน่นอน!)
puts 1.0/3 + 1.0/3 + 1.0/3  # => 0.9999... (ไม่แม่นยำ)

# Complex Numbers
c1 = Complex(3, 4)    # 3 + 4i
c2 = Complex(1, -2)   # 1 - 2i

puts c1 + c2    # => 4+2i
puts c1 * c2    # => 11-2i
puts c1.abs     # => 5.0 (magnitude)
puts c1.angle   # => 0.9272... (in radians)
puts c1.conjugate  # => 3-4i
```

---

## Step 57: Numeric Comparisons

### 57.1 Comparison Operators

```ruby
puts 5 == 5     # => true
puts 5 == 5.0   # => true (Integer == Float if values equal)
puts 5.eql?(5)   # => true
puts 5.eql?(5.0) # => false (eql? checks type too)

puts 5 <=> 3    # => 1
puts 5 <=> 5    # => 0
puts 5 <=> 7    # => -1

# between? และ clamp
puts 5.between?(1, 10)  # => true
puts 15.between?(1, 10) # => false

puts 5.clamp(1, 10)   # => 5
puts 0.clamp(1, 10)   # => 1
puts 15.clamp(1, 10)  # => 10

# clamp with range (Ruby 2.7+)
puts 5.clamp(1..10)   # => 5
puts 0.clamp(1..10)   # => 1
puts 15.clamp(1..10)  # => 10

# clamp with endless/beginless range
puts (-5).clamp(0..)  # => 0 (minimum 0)
puts 5.clamp(..10)    # => 5
puts 15.clamp(..10)   # => 10
```

---

## Step 58: Number Utilities

### 58.1 เพิ่มเติม

```ruby
# Infinity arithmetic
puts Float::INFINITY + 1         # => Infinity
puts Float::INFINITY > 9999999   # => true

# ใช้ infinity เพื่อหา min/max โดยไม่มี initial value
numbers = [3, 1, 4, 1, 5, 9, 2, 6]
min_val = Float::INFINITY
max_val = -Float::INFINITY

numbers.each do |n|
  min_val = n if n < min_val
  max_val = n if n > max_val
end

puts "Min: #{min_val}, Max: #{max_val}"  # => Min: 1, Max: 9

# Ruby way ที่ง่ายกว่า
puts numbers.min  # => 1
puts numbers.max  # => 9
```

```ruby
# Numeric Utilities
# divmod
puts 13.divmod(4).inspect  # => [3, 1] (quotient, remainder)

# fdiv - Float division
puts 7.fdiv(2)    # => 3.5 (Float division สำหรับ Integer)

# modulo และ %
puts 13.modulo(4)  # => 1
puts 13 % 4        # => 1

# remainder (ต่างจาก modulo สำหรับ negative)
puts (-13).remainder(4)  # => -1
puts (-13) % 4           # => 3

# ceil, floor สำหรับ Integer
puts 13.ceil(-1)    # => 20 (ปัดขึ้นเป็น 10s)
puts 13.floor(-1)   # => 10
puts 17.ceil(-1)    # => 20
puts 13.round(-1)   # => 10
puts 15.round(-1)   # => 20 (ปัดขึ้น)
puts 15.round(-1, half: :even)  # => 20 (banker's rounding)
```

---

## Step 59: Performance Tips

### 59.1 ประสิทธิภาพการคำนวณ

```ruby
require 'benchmark'

n = 1_000_000

# Integer vs Float
Benchmark.bm(15) do |x|
  x.report("integer +") { n.times { 1 + 1 } }
  x.report("float +")   { n.times { 1.0 + 1.0 } }
  x.report("integer *") { n.times { 100 * 100 } }
  x.report("float *")   { n.times { 100.0 * 100.0 } }
  x.report("integer /") { n.times { 100 / 3 } }
  x.report("float /")   { n.times { 100.0 / 3.0 } }
end
```

```ruby
# Memoization สำหรับการคำนวณซ้ำ
# โดยไม่มี memoization
def fib_slow(n)
  return n if n <= 1
  fib_slow(n-1) + fib_slow(n-2)
end

# ด้วย memoization
def fib_fast(n, memo = {})
  return memo[n] if memo.key?(n)
  return n if n <= 1
  memo[n] = fib_fast(n-1, memo) + fib_fast(n-2, memo)
end

# แตกต่างมาก!
require 'benchmark'
Benchmark.bm do |x|
  x.report("slow:") { fib_slow(30) }
  x.report("fast:") { fib_fast(50) }  # 50 ทำได้เพราะ memoization
end
```

---

## Step 60: แบบฝึกหัด

### แบบฝึกหัดที่ 1
คำนวณดอกเบี้ยทบต้น

```ruby
# เฉลย
require 'bigdecimal'
require 'bigdecimal/util'

def compound_interest(principal, rate, years, compounds_per_year = 12)
  r = rate.to_d / 100
  n = compounds_per_year
  t = years
  p = principal.to_d
  
  # A = P(1 + r/n)^(nt)
  amount = p * (1 + r/n) ** (n * t)
  interest = amount - p
  
  {
    principal: p.round(2).to_s("F"),
    total: amount.round(2).to_s("F"),
    interest: interest.round(2).to_s("F")
  }
end

result = compound_interest(100_000, 3.5, 10)
puts "เงินต้น: #{result[:principal]} บาท"
puts "ดอกเบี้ย: #{result[:interest]} บาท"
puts "รวม: #{result[:total]} บาท"
```

### แบบฝึกหัดที่ 2
เขียน Calculator ด้วย BigDecimal

```ruby
# เฉลย
require 'bigdecimal'
require 'bigdecimal/util'

class Calculator
  def initialize
    @memory = BigDecimal("0")
  end
  
  def calculate(expression)
    # ง่ายๆ: parse 2 operands และ 1 operator
    if (m = expression.match(/^([\d.]+)\s*([+\-*\/])\s*([\d.]+)$/))
      a = m[1].to_d
      op = m[2]
      b = m[3].to_d
      
      result = case op
               when "+" then a + b
               when "-" then a - b
               when "*" then a * b
               when "/" then b.zero? ? "Error: Division by zero" : a / b
               end
      
      result.is_a?(BigDecimal) ? result.to_s("F") : result
    else
      "Error: Invalid expression"
    end
  end
  
  def store(value)
    @memory = value.to_d
  end
  
  def recall
    @memory.to_s("F")
  end
end

calc = Calculator.new
puts calc.calculate("19.99 + 5.01")  # => 25.0
puts calc.calculate("100 * 0.07")    # => 7.0
puts calc.calculate("10 / 3")        # => 3.333333...
```

### แบบฝึกหัดที่ 3
หาจำนวนเฉพาะ (Prime Numbers)

```ruby
# เฉลย
def prime?(n)
  return false if n < 2
  return true if n == 2
  return false if n.even?
  
  (3..Math.sqrt(n).to_i).step(2).none? { |i| n % i == 0 }
end

def primes_up_to(limit)
  (2..limit).select { |n| prime?(n) }
end

# Sieve of Eratosthenes (เร็วกว่า)
def sieve_of_eratosthenes(limit)
  is_prime = Array.new(limit + 1, true)
  is_prime[0] = is_prime[1] = false
  
  (2..Math.sqrt(limit).to_i).each do |i|
    if is_prime[i]
      (i*i..limit).step(i) { |j| is_prime[j] = false }
    end
  end
  
  (2..limit).select { |i| is_prime[i] }
end

primes = sieve_of_eratosthenes(100)
puts "จำนวนเฉพาะถึง 100: #{primes.inspect}"
puts "จำนวนทั้งหมด: #{primes.length}"
```

### แบบฝึกหัดที่ 4
ลำดับ Fibonacci

```ruby
# เฉลย
def fibonacci(n)
  return [] if n <= 0
  return [0] if n == 1
  return [0, 1] if n == 2
  
  fib = [0, 1]
  (n - 2).times { fib << fib[-1] + fib[-2] }
  fib
end

# Fibonacci ด้วย Enumerator (lazy)
fib_enum = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

puts fibonacci(10).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

puts fib_enum.first(10).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# หา Fibonacci ที่น้อยกว่า 1000
big_fib = fib_enum.take_while { |n| n < 1000 }
puts big_fib.inspect
```

### แบบฝึกหัดที่ 5
Statistical Functions

```ruby
# เฉลย
module Statistics
  def self.mean(data)
    return 0.0 if data.empty?
    data.sum.to_f / data.length
  end
  
  def self.median(data)
    return nil if data.empty?
    sorted = data.sort
    n = sorted.length
    
    if n.odd?
      sorted[n / 2].to_f
    else
      (sorted[n/2 - 1] + sorted[n/2]) / 2.0
    end
  end
  
  def self.mode(data)
    frequency = data.tally
    max_freq = frequency.values.max
    frequency.select { |_, v| v == max_freq }.keys
  end
  
  def self.variance(data)
    return 0.0 if data.length <= 1
    m = mean(data)
    data.sum { |x| (x - m) ** 2 } / (data.length - 1)
  end
  
  def self.std_dev(data)
    Math.sqrt(variance(data))
  end
  
  def self.summary(data)
    {
      count: data.length,
      min: data.min,
      max: data.max,
      mean: mean(data).round(4),
      median: median(data),
      mode: mode(data),
      std_dev: std_dev(data).round(4)
    }
  end
end

data = [4, 7, 13, 16, 21, 4, 7, 7, 12, 4]
stats = Statistics.summary(data)
stats.each { |k, v| puts "#{k}: #{v}" }
```

### แบบฝึกหัดที่ 6
Unit Converter

```ruby
# เฉลย
class UnitConverter
  CONVERSIONS = {
    length: {
      meter: 1.0,
      kilometer: 1000.0,
      mile: 1609.344,
      yard: 0.9144,
      foot: 0.3048,
      inch: 0.0254,
      centimeter: 0.01,
      millimeter: 0.001
    },
    weight: {
      kilogram: 1.0,
      gram: 0.001,
      pound: 0.453592,
      ounce: 0.0283495,
      ton: 1000.0
    },
    temperature: :special
  }
  
  def self.convert(value, from, to, type)
    table = CONVERSIONS[type]
    return "ไม่รู้จัก type: #{type}" unless table
    
    if type == :temperature
      convert_temperature(value, from, to)
    else
      return "ไม่รู้จักหน่วย" unless table[from] && table[to]
      value * table[from] / table[to]
    end
  end
  
  def self.convert_temperature(value, from, to)
    celsius = case from
    when :celsius    then value
    when :fahrenheit then (value - 32) * 5.0 / 9
    when :kelvin     then value - 273.15
    end
    
    case to
    when :celsius    then celsius
    when :fahrenheit then celsius * 9.0 / 5 + 32
    when :kelvin     then celsius + 273.15
    end
  end
end

puts UnitConverter.convert(1, :kilometer, :mile, :length).round(4)   # => 0.6214
puts UnitConverter.convert(100, :celsius, :fahrenheit, :temperature)  # => 212.0
puts UnitConverter.convert(1, :kilogram, :pound, :weight).round(4)    # => 2.2046
```

### แบบฝึกหัดที่ 7
Geometry Calculator

```ruby
# เฉลย
module Geometry
  PI = Math::PI
  
  module Circle
    def self.area(r)       = PI * r ** 2
    def self.circumference(r) = 2 * PI * r
    def self.diameter(r)   = 2 * r
  end
  
  module Rectangle
    def self.area(w, h)      = w * h
    def self.perimeter(w, h) = 2 * (w + h)
    def self.diagonal(w, h)  = Math.hypot(w, h)
  end
  
  module Triangle
    def self.area(b, h)  = 0.5 * b * h
    
    def self.area_from_sides(a, b, c)
      # Heron's formula
      s = (a + b + c) / 2.0
      Math.sqrt(s * (s-a) * (s-b) * (s-c))
    end
    
    def self.perimeter(a, b, c) = a + b + c
  end
  
  module Sphere
    def self.volume(r)              = (4.0/3) * PI * r ** 3
    def self.surface_area(r)        = 4 * PI * r ** 2
  end
end

puts Geometry::Circle.area(5).round(2)               # => 78.54
puts Geometry::Rectangle.diagonal(3, 4).round(2)     # => 5.0
puts Geometry::Triangle.area_from_sides(3,4,5).round(2)  # => 6.0
puts Geometry::Sphere.volume(3).round(2)              # => 113.1
```

### แบบฝึกหัดที่ 8
Binary Number Operations

```ruby
# เฉลย
def to_binary(n)
  n.to_s(2).rjust(8, '0')
end

def binary_add(a, b)
  result = []
  carry = 0
  
  [a.length, b.length].max.times do |i|
    bit_a = i < a.length ? a[-(i+1)].to_i : 0
    bit_b = i < b.length ? b[-(i+1)].to_i : 0
    sum = bit_a + bit_b + carry
    result.unshift(sum % 2)
    carry = sum / 2
  end
  
  result.unshift(1) if carry > 0
  result.join
end

a = to_binary(42)   # => "00101010"
b = to_binary(27)   # => "00011011"
sum_bin = binary_add(a, b)
sum_dec = sum_bin.to_i(2)

puts "#{42} = #{a}"
puts "#{27} = #{b}"
puts "#{42} + #{27} = #{sum_dec} = #{sum_bin}"
# => 42 + 27 = 69 = 1000101
```

### แบบฝึกหัดที่ 9
Loan Calculator

```ruby
# เฉลย
require 'bigdecimal'
require 'bigdecimal/util'

class LoanCalculator
  def initialize(principal, annual_rate, months)
    @principal = principal.to_d
    @monthly_rate = (annual_rate / 100.0 / 12).to_d
    @months = months
  end
  
  def monthly_payment
    if @monthly_rate.zero?
      @principal / @months
    else
      @principal * @monthly_rate / (1 - (1 + @monthly_rate) ** (-@months))
    end
  end
  
  def total_payment
    monthly_payment * @months
  end
  
  def total_interest
    total_payment - @principal
  end
  
  def amortization_schedule(show_months = 6)
    payment = monthly_payment
    balance = @principal
    
    puts "%-6s %-12s %-12s %-12s %-12s" % ["เดือน", "ผ่อน", "ดอกเบี้ย", "เงินต้น", "คงเหลือ"]
    puts "-" * 60
    
    [@months, show_months].min.times do |i|
      interest = balance * @monthly_rate
      principal_paid = payment - interest
      balance -= principal_paid
      
      printf "%-6d %-12.2f %-12.2f %-12.2f %-12.2f\n",
             i+1, payment.to_f, interest.to_f, principal_paid.to_f, [balance.to_f, 0].max
    end
    
    puts "..." if @months > show_months
  end
end

loan = LoanCalculator.new(500_000, 5.5, 120)  # 500k, 5.5%, 10 years
puts "ผ่อนรายเดือน: #{loan.monthly_payment.round(2)}"
puts "รวมที่จ่าย: #{loan.total_payment.round(2)}"
puts "ดอกเบี้ยรวม: #{loan.total_interest.round(2)}"
puts
loan.amortization_schedule
```

### แบบฝึกหัดที่ 10
Number Theory

```ruby
# เฉลย
module NumberTheory
  def self.prime_factors(n)
    factors = []
    d = 2
    
    while d * d <= n
      while n % d == 0
        factors << d
        n /= d
      end
      d += 1
    end
    
    factors << n if n > 1
    factors
  end
  
  def self.euler_phi(n)
    # Euler's totient function
    count = 0
    (1..n).each { |k| count += 1 if n.gcd(k) == 1 }
    count
  end
  
  def self.perfect?(n)
    return false if n <= 1
    divisors = (1...n).select { |i| n % i == 0 }
    divisors.sum == n
  end
  
  def self.abundant?(n)
    (1...n).select { |i| n % i == 0 }.sum > n
  end
  
  def self.deficient?(n)
    (1...n).select { |i| n % i == 0 }.sum < n
  end
end

puts NumberTheory.prime_factors(60).inspect   # => [2, 2, 3, 5]
puts NumberTheory.euler_phi(10)               # => 4
puts NumberTheory.perfect?(28)                # => true (1+2+4+7+14=28)
puts NumberTheory.perfect?(6)                 # => true (1+2+3=6)

puts "Perfect numbers ถึง 1000:"
(2..1000).each do |n|
  puts n if NumberTheory.perfect?(n)
end
# 6, 28, 496
```

### แบบฝึกหัดที่ 11
Roman Numerals

```ruby
# เฉลย
def to_roman(n)
  values = [
    [1000, "M"], [900, "CM"], [500, "D"], [400, "CD"],
    [100, "C"],  [90, "XC"],  [50, "L"],  [40, "XL"],
    [10, "X"],   [9, "IX"],   [5, "V"],   [4, "IV"], [1, "I"]
  ]
  
  result = ""
  values.each do |value, numeral|
    while n >= value
      result += numeral
      n -= value
    end
  end
  result
end

def from_roman(str)
  values = { "I" => 1, "V" => 5, "X" => 10, "L" => 50,
             "C" => 100, "D" => 500, "M" => 1000 }
  
  total = 0
  str.chars.each_with_index do |char, i|
    curr = values[char]
    next_val = values[str[i+1]] || 0
    
    if curr < next_val
      total -= curr
    else
      total += curr
    end
  end
  total
end

(1..20).each { |n| puts "#{n} = #{to_roman(n)}" }
puts from_roman("MCMXCIX")  # => 1999
puts to_roman(2024)         # => MMXXIV
```

### แบบฝึกหัดที่ 12
Matrix Operations

```ruby
# เฉลย
class Matrix2D
  def initialize(data)
    @data = data
    @rows = data.length
    @cols = data[0].length
  end
  
  def [](i, j) = @data[i][j]
  def rows = @rows
  def cols = @cols
  
  def +(other)
    raise "ขนาดไม่เท่ากัน" unless @rows == other.rows && @cols == other.cols
    
    result = Array.new(@rows) { Array.new(@cols) }
    @rows.times do |i|
      @cols.times do |j|
        result[i][j] = @data[i][j] + other[i, j]
      end
    end
    Matrix2D.new(result)
  end
  
  def *(other)
    raise "ขนาดไม่ compatible" unless @cols == other.rows
    
    result = Array.new(@rows) { Array.new(other.cols, 0) }
    @rows.times do |i|
      other.cols.times do |j|
        @cols.times do |k|
          result[i][j] += @data[i][k] * other[k, j]
        end
      end
    end
    Matrix2D.new(result)
  end
  
  def determinant
    raise "ต้องเป็น square matrix" unless @rows == @cols
    
    if @rows == 2
      @data[0][0] * @data[1][1] - @data[0][1] * @data[1][0]
    else
      # Expansion along first row
      @cols.times.sum do |j|
        sign = j.even? ? 1 : -1
        @data[0][j] * sign * minor(0, j).determinant
      end
    end
  end
  
  def minor(row, col)
    data = @data.each_with_index.reject { |_, i| i == row }
                .map { |r, _| r.each_with_index.reject { |_, j| j == col }.map(&:first) }
    Matrix2D.new(data)
  end
  
  def to_s
    @data.map { |row| row.map { |v| "%6.2f" % v }.join(" ") }.join("\n")
  end
end

a = Matrix2D.new([[1, 2], [3, 4]])
b = Matrix2D.new([[5, 6], [7, 8]])
puts "A * B ="
puts (a * b).to_s
puts "det(A) = #{a.determinant}"
```

### แบบฝึกหัดที่ 13
Numerical Integration

```ruby
# เฉลย
module NumericalIntegration
  # Trapezoidal Rule
  def self.trapezoidal(func, a, b, n = 1000)
    h = (b - a).to_f / n
    sum = func.call(a) / 2.0 + func.call(b) / 2.0
    
    (1...n).each do |i|
      sum += func.call(a + i * h)
    end
    
    sum * h
  end
  
  # Simpson's Rule
  def self.simpsons(func, a, b, n = 1000)
    n = n.even? ? n : n + 1
    h = (b - a).to_f / n
    sum = func.call(a) + func.call(b)
    
    (1...n).each do |i|
      sum += (i.even? ? 2 : 4) * func.call(a + i * h)
    end
    
    sum * h / 3.0
  end
end

# คำนวณ ∫₀¹ x² dx = 1/3
f = ->(x) { x ** 2 }
puts NumericalIntegration.trapezoidal(f, 0, 1).round(6)  # ≈ 0.333333
puts NumericalIntegration.simpsons(f, 0, 1).round(6)     # ≈ 0.333333

# คำนวณ ∫₀^π sin(x) dx = 2
g = ->(x) { Math.sin(x) }
puts NumericalIntegration.simpsons(g, 0, Math::PI).round(6)  # ≈ 2.0
```

### แบบฝึกหัดที่ 14
Number Guessing Game

```ruby
# เฉลย
class NumberGuessingGame
  def initialize(min: 1, max: 100)
    @min = min
    @max = max
    @secret = rand(min..max)
    @attempts = 0
    @max_attempts = Math.log2(max - min + 1).ceil + 1
  end
  
  def guess(number)
    @attempts += 1
    
    if number == @secret
      puts "ถูกต้อง! ใช้ #{@attempts} ครั้ง (optimal: #{@max_attempts})"
      :correct
    elsif @attempts >= @max_attempts
      puts "หมดโอกาสแล้ว! ตัวเลขคือ #{@secret}"
      :game_over
    elsif number < @secret
      puts "มากกว่า #{number}"
      :low
    else
      puts "น้อยกว่า #{number}"
      :high
    end
  end
  
  # Optimal strategy (binary search)
  def optimal_guess
    (@min + @max) / 2
  end
  
  def auto_play
    low = @min
    high = @max
    
    until (result = guess(mid = (low + high) / 2)) == :correct
      break if result == :game_over
      result == :low ? low = mid + 1 : high = mid - 1
    end
  end
end

game = NumberGuessingGame.new
puts "เล่นอัตโนมัติ (binary search):"
game.auto_play
```

### แบบฝึกหัดที่ 15
Financial Portfolio

```ruby
# เฉลย
require 'bigdecimal'
require 'bigdecimal/util'

class Portfolio
  Investment = Struct.new(:name, :shares, :buy_price, :current_price)
  
  def initialize
    @investments = []
  end
  
  def add(name, shares, buy_price, current_price)
    @investments << Investment.new(name, shares.to_d, buy_price.to_d, current_price.to_d)
  end
  
  def total_cost
    @investments.sum { |inv| inv.shares * inv.buy_price }
  end
  
  def total_value
    @investments.sum { |inv| inv.shares * inv.current_price }
  end
  
  def total_gain_loss
    total_value - total_cost
  end
  
  def return_percent
    return BigDecimal("0") if total_cost.zero?
    (total_gain_loss / total_cost * 100).round(2)
  end
  
  def report
    puts "=" * 70
    puts "%-15s %8s %10s %10s %10s %10s" % ["ชื่อ", "จำนวน", "ราคาซื้อ", "ราคาตลาด", "ต้นทุน", "มูลค่า"]
    puts "=" * 70
    
    @investments.each do |inv|
      cost = inv.shares * inv.buy_price
      value = inv.shares * inv.current_price
      printf "%-15s %8.0f %10.2f %10.2f %10.2f %10.2f\n",
             inv.name, inv.shares.to_f, inv.buy_price.to_f,
             inv.current_price.to_f, cost.to_f, value.to_f
    end
    
    puts "=" * 70
    puts "ต้นทุนรวม:  #{total_cost.round(2).to_f}"
    puts "มูลค่ารวม:  #{total_value.round(2).to_f}"
    puts "กำไร/ขาดทุน: #{total_gain_loss.round(2).to_f}"
    puts "ผลตอบแทน:   #{return_percent}%"
  end
end

portfolio = Portfolio.new
portfolio.add("Apple Inc.", 100, 150.00, 175.50)
portfolio.add("Google", 50, 2800.00, 2950.00)
portfolio.add("Tesla", 200, 900.00, 750.00)
portfolio.add("Microsoft", 75, 300.00, 325.00)

portfolio.report
```

---

## สรุป

ในตอนที่ 4 นี้ เราได้เรียนรู้:

1. **Integer** - Literals, Methods, Bitwise Operations, Large Numbers
2. **Float** - Precision, Special Values (NaN, Infinity), Methods
3. **Arithmetic** - Integer vs Float Division (ระวัง!)
4. **Math Module** - sqrt, exp, log, Trigonometry
5. **BigDecimal** - สำหรับการเงินที่ต้องการความแม่นยำสูง
6. **Number Bases** - Binary, Octal, Hexadecimal
7. **Random** - rand, Random.new, Seeding
8. **Conversions** - to_i, to_f, to_r, to_c, Integer(), Float()
9. **Formatting** - round, ceil, floor, sprintf

ในตอนต่อไปเราจะเรียนรู้เรื่อง **Arrays** ซึ่งเป็นโครงสร้างข้อมูลที่ทรงพลังมากใน Ruby

---

*ตอนที่ 4 จบแล้ว - ไปต่อตอนที่ 5: Arrays*
