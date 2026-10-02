# ตอนที่ 4: Numbers และ Math (Steps 46-60)

## บทนำ

Ruby รองรับตัวเลขหลายประเภท ตั้งแต่จำนวนเต็ม (Integer) จำนวนทศนิยม (Float) จนถึงจำนวนตรรกยะ (Rational) และจำนวนเชิงซ้อน (Complex) มี Math module ที่ให้ฟังก์ชันทางคณิตศาสตร์ครบครัน รวมถึง BigDecimal สำหรับการคำนวณทางการเงินที่ต้องการความแม่นยำสูง

---

## Step 46: Integer Literals

### รูปแบบการเขียน Integer

Ruby รองรับการเขียน Integer ในหลายรูปแบบ

```ruby
# ทศนิยม (Decimal) - รูปแบบปกติ
puts 42           # => 42
puts 1000000      # => 1000000
puts -255         # => -255

# ใช้ underscore เพื่อให้อ่านง่าย (ไม่มีผลต่อค่า)
puts 1_000_000    # => 1000000
puts 1_234_567    # => 1234567
puts 9_999_999.99 # => 9999999.99 (ใช้กับ Float ได้ด้วย)
```

```ruby
# เลขฐานสอง (Binary) - ขึ้นต้นด้วย 0b
puts 0b1010       # => 10
puts 0b1111_1111  # => 255
puts 0b0000_0001  # => 1
puts 0b1000_0000  # => 128

# แปลงกลับ
puts 10.to_s(2)   # => "1010" (แสดง binary representation)
puts 255.to_s(2)  # => "11111111"
```

```ruby
# เลขฐานสิบหก (Hexadecimal) - ขึ้นต้นด้วย 0x
puts 0xFF         # => 255
puts 0x1F         # => 31
puts 0xDEAD_BEEF  # => 3735928559
puts 0x00FF_FFFF  # => 16777215 (สีขาวใน RGB)

# แปลงกลับ
puts 255.to_s(16)  # => "ff"
puts 255.to_s(16).upcase  # => "FF"
```

```ruby
# เลขฐานแปด (Octal) - ขึ้นต้นด้วย 0o
puts 0o777    # => 511
puts 0o644    # => 420 (Unix file permission: rw-r--r--)
puts 0o755    # => 493 (Unix file permission: rwxr-xr-x)

# แปลงกลับ
puts 511.to_s(8)  # => "777"
puts 420.to_s(8)  # => "644"
```

```ruby
# ขนาดของ Integer ใน Ruby
puts 2**31 - 1   # => 2147483647 (Max 32-bit signed)
puts 2**63 - 1   # => 9223372036854775807 (Max 64-bit signed)
puts 2**100      # => 1267650600228229401496703205376 (Ruby รองรับ BigInteger!)

# Ruby จัดการ Integer ขนาดใหญ่ได้โดยอัตโนมัติ (ไม่ overflow)
factorial_20 = (1..20).reduce(:*)
puts factorial_20  # => 2432902008176640000
```

---

## Step 47: Float Literals

### จำนวนทศนิยม

```ruby
# Float ปกติ
puts 3.14         # => 3.14
puts -2.718       # => -2.718
puts 0.5          # => 0.5
puts 100.0        # => 100.0

# ต้องมีตัวเลขทั้งสองด้านของ .
# puts 1.  # SyntaxError
# puts .5  # SyntaxError
puts 1.0  # => 1.0 (ถูกต้อง)
puts 0.5  # => 0.5 (ถูกต้อง)
```

```ruby
# Scientific notation
puts 1.0e10    # => 10000000000.0
puts 1.5e-3    # => 0.0015
puts 2.5e+8    # => 250000000.0
puts 6.022e23  # => 6.022e+23 (Avogadro's number)

# ความแม่นยำของ Float
puts 0.1 + 0.2         # => 0.30000000000000004 (floating point issue!)
puts (0.1 + 0.2).round(1)  # => 0.3
```

```ruby
# Float precision limits
puts Float::MAX         # => 1.7976931348623157e+308
puts Float::MIN         # => 2.2250738585072014e-308
puts Float::EPSILON     # => 2.220446049250313e-16 (ความแม่นยำสูงสุด)
puts Float::DIG         # => 15 (จำนวนหลักทศนิยมที่แม่นยำ)
puts Float::INFINITY    # => Infinity
puts -Float::INFINITY   # => -Infinity
puts Float::NAN         # => NaN
```

---

## Step 48: Arithmetic Operations

### การคำนวณพื้นฐาน

```ruby
# บวก ลบ คูณ หาร
puts 10 + 3   # => 13
puts 10 - 3   # => 7
puts 10 * 3   # => 30
puts 10 / 3   # => 3 (Integer division!)
puts 10.0 / 3 # => 3.3333333333333335
puts 10 / 3.0 # => 3.3333333333333335
```

```ruby
# Integer vs Float Division - สำคัญมาก!
# Integer / Integer = Integer (ตัดทศนิยมออก)
puts 7 / 2    # => 3 (ไม่ใช่ 3.5!)
puts 7 / 2.0  # => 3.5
puts 7.0 / 2  # => 3.5
puts 7.fdiv(2) # => 3.5 (Float division method)

# ข้อควรระวัง: ใน Rails คำนวณ percentage
students = 30
passed = 25
# ผิด:
rate = passed / students * 100
puts rate  # => 83 (ไม่ถูกต้อง, ควรเป็น 83.33...)

# ถูก:
rate = passed.to_f / students * 100
puts rate.round(2)  # => 83.33
```

```ruby
# Modulo (%)  - เศษจากการหาร
puts 10 % 3    # => 1
puts 17 % 5    # => 2
puts -7 % 3    # => 2 (Ruby modulo: ผลลัพธ์มีเครื่องหมายเหมือน divisor)
puts 7 % -3    # => -2

# ใช้ modulo ตรวจสอบคู่/คี่
puts 4 % 2 == 0  # => true (คู่)
puts 5 % 2 == 0  # => false (คี่)

# ใช้ modulo ใน circular indexing
days = ["จันทร์", "อังคาร", "พุธ", "พฤหัส", "ศุกร์", "เสาร์", "อาทิตย์"]
(0..13).each do |i|
  puts "วันที่ #{i}: #{days[i % 7]}"
end
```

```ruby
# ยกกำลัง (**)
puts 2 ** 10   # => 1024
puts 3 ** 3    # => 27
puts 2 ** 0.5  # => 1.4142135623730951 (square root)
puts 8 ** (1.0/3)  # => 2.0 (cube root)

# Integer#pow กับ modulus (สำหรับ cryptography)
puts 2.pow(10, 1000)  # => 24 (2^10 mod 1000) - เร็วกว่า (2**10) % 1000 มาก
```

---

## Step 49: Integer Methods

### เมธอดของ Integer

```ruby
# Odd and Even
puts 4.even?   # => true
puts 5.even?   # => false
puts 4.odd?    # => false
puts 5.odd?    # => true

# Zero
puts 0.zero?   # => true
puts 1.zero?   # => false

# Sign methods
puts 5.positive?   # => true
puts -5.positive?  # => false
puts -5.negative?  # => true
puts 0.positive?   # => false
puts 0.negative?   # => false
puts 0.zero?       # => true
```

```ruby
# Absolute value
puts (-5).abs   # => 5
puts 5.abs      # => 5
puts (-100).abs # => 100

# GCD and LCM
puts 12.gcd(8)   # => 4 (Greatest Common Divisor)
puts 12.lcm(8)   # => 24 (Least Common Multiple)
puts 15.gcd(10)  # => 5
puts 15.lcm(10)  # => 30

# กรณีใช้งานจริง: ทำ fraction ให้ง่ายที่สุด
def simplify_fraction(num, den)
  g = num.gcd(den)
  [num / g, den / g]
end

puts simplify_fraction(12, 8).inspect   # => [3, 2]
puts simplify_fraction(6, 9).inspect    # => [2, 3]
```

```ruby
# digits - แยกตัวเลขเป็น array (จากหลักน้อยไปหลักมาก)
puts 1234.digits.inspect     # => [4, 3, 2, 1]
puts 1234.digits(10).inspect # => [4, 3, 2, 1] (base 10)
puts 255.digits(16).inspect  # => [15, 15] (base 16: FF)

# ใช้งาน: หาผลรวมของทุก digit
def digit_sum(n)
  n.abs.digits.sum
end

puts digit_sum(1234)    # => 10 (1+2+3+4)
puts digit_sum(99)      # => 18 (9+9)
puts digit_sum(12345)   # => 15
```

```ruby
# to_s กับ base - แปลงเป็น string ในฐานต่างๆ
puts 255.to_s     # => "255"
puts 255.to_s(2)  # => "11111111" (binary)
puts 255.to_s(8)  # => "377" (octal)
puts 255.to_s(16) # => "ff" (hexadecimal)
puts 10.to_s(36)  # => "a" (base 36)
puts 35.to_s(36)  # => "z"

# แปลงกลับ: String#to_i กับ base
puts "11111111".to_i(2)  # => 255
puts "ff".to_i(16)       # => 255
puts "377".to_i(8)       # => 255
```

```ruby
# Integer iteration methods
# times - วนซ้ำตามจำนวนครั้ง
3.times { |i| puts "ครั้งที่ #{i + 1}" }
# => ครั้งที่ 1
# => ครั้งที่ 2
# => ครั้งที่ 3

# upto - วนจากน้อยไปมาก
1.upto(5) { |i| print "#{i} " }
puts  # => 1 2 3 4 5

# downto - วนจากมากไปน้อย
5.downto(1) { |i| print "#{i} " }
puts  # => 5 4 3 2 1

# step - วนด้วย step ที่กำหนด
1.step(10, 2) { |i| print "#{i} " }
puts  # => 1 3 5 7 9
```

---

## Step 50: Float Methods

### เมธอดของ Float

```ruby
pi = 3.14159265358979

# round - ปัดเศษ
puts pi.round      # => 3 (ปัดเป็น integer)
puts pi.round(2)   # => 3.14
puts pi.round(4)   # => 3.1416
puts 2.5.round     # => 3 (หลักปัด: >= 0.5 ปัดขึ้น)
puts 2.4.round     # => 2

# ceil - ปัดขึ้นเสมอ
puts pi.ceil       # => 4
puts pi.ceil(2)    # => 3.15
puts 2.1.ceil      # => 3

# floor - ปัดลงเสมอ
puts pi.floor      # => 3
puts pi.floor(2)   # => 3.14
puts 2.9.floor     # => 2
```

```ruby
# truncate - ตัดทศนิยมออก (ไม่ปัด)
puts 3.9.truncate   # => 3
puts (-3.9).truncate # => -3 (ต่างจาก floor ซึ่งจะได้ -4)
puts 3.14.truncate(1) # => 3.1

# abs
puts (-3.14).abs  # => 3.14
puts 3.14.abs     # => 3.14
```

```ruby
# การตรวจสอบ special values
puts (1.0/0).infinite?    # => 1 (บวก infinity)
puts (-1.0/0).infinite?   # => -1 (ลบ infinity)
puts 3.14.infinite?       # => nil (ไม่ใช่ infinity)

puts (0.0/0).nan?         # => true (NaN - Not a Number)
puts 3.14.nan?            # => false

puts 3.14.finite?         # => true
puts (1.0/0).finite?      # => false

# ป้องกัน NaN ใน calculation
def safe_divide(a, b)
  return 0.0 if b.zero?
  result = a.to_f / b
  result.nan? || result.infinite? ? 0.0 : result
end

puts safe_divide(10, 3)    # => 3.3333333333333335
puts safe_divide(10, 0)    # => 0.0
puts safe_divide(0, 0)     # => 0.0
```

```ruby
# การแปลง Float
puts 3.9.to_i    # => 3 (เหมือน truncate)
puts 3.9.floor   # => 3
puts 3.9.ceil    # => 4
puts 3.9.round   # => 4

# Float arithmetic precision issue
puts 0.1 + 0.2   # => 0.30000000000000004

# วิธีแก้: ใช้ round
result = (0.1 + 0.2).round(10)
puts result        # => 0.3
puts result == 0.3 # => true
```

---

## Step 51: Math Module

### ฟังก์ชันทางคณิตศาสตร์

```ruby
# Constants
puts Math::PI    # => 3.141592653589793
puts Math::E     # => 2.718281828459045

# Square root และ Cube root
puts Math.sqrt(16)    # => 4.0
puts Math.sqrt(2)     # => 1.4142135623730951
puts Math.cbrt(27)    # => 3.0
puts Math.cbrt(8)     # => 2.0

# หรือใช้ ** operator
puts 16 ** 0.5        # => 4.0
puts 27 ** (1.0/3)    # => 3.0
```

```ruby
# Logarithms
puts Math.log(Math::E)    # => 1.0 (natural log, base e)
puts Math.log(100, 10)    # => 2.0 (log base 10)
puts Math.log2(8)         # => 3.0 (log base 2)
puts Math.log10(1000)     # => 3.0 (log base 10)
puts Math.exp(1)          # => 2.718281828459045 (e^1)
puts Math.exp(2)          # => 7.38905609893065 (e^2)
```

```ruby
# Trigonometric functions (ใช้ Radian)
# แปลง degree เป็น radian
def deg_to_rad(degrees)
  degrees * Math::PI / 180
end

def rad_to_deg(radians)
  radians * 180 / Math::PI
end

puts Math.sin(deg_to_rad(90))   # => 1.0
puts Math.cos(deg_to_rad(0))    # => 1.0
puts Math.tan(deg_to_rad(45))   # => 0.9999999999999999 (≈ 1.0)

puts Math.sin(Math::PI / 2)     # => 1.0
puts Math.cos(Math::PI)         # => -1.0
```

```ruby
# Inverse trigonometric
puts rad_to_deg(Math.asin(1.0))   # => 90.0
puts rad_to_deg(Math.acos(1.0))   # => 0.0
puts rad_to_deg(Math.atan(1.0))   # => 45.0
puts rad_to_deg(Math.atan2(1, 1)) # => 45.0 (atan2 บอก quadrant ด้วย)

# Hyperbolic functions
puts Math.sinh(1)  # => 1.1752011936438014
puts Math.cosh(1)  # => 1.5430806348152437
puts Math.tanh(1)  # => 0.7615941559557649
```

```ruby
# hypot - คำนวณ hypotenuse (ด้านตรงข้ามมุมฉาก)
puts Math.hypot(3, 4)    # => 5.0 (Pythagorean: 3² + 4² = 5²)
puts Math.hypot(5, 12)   # => 13.0

# ใช้งานจริง: หาระยะห่างระหว่าง 2 จุด
def distance(x1, y1, x2, y2)
  Math.hypot(x2 - x1, y2 - y1)
end

puts distance(0, 0, 3, 4)    # => 5.0
puts distance(1, 1, 4, 5)    # => 5.0
```

---

## Step 52: Number Formatting

### การจัดรูปแบบตัวเลข

```ruby
# ใช้ sprintf / format
puts sprintf("%.2f", 3.14159)      # => 3.14
puts sprintf("%d", 42.9)           # => 42
puts sprintf("%05d", 42)           # => 00042
puts sprintf("%+.2f", 3.14)       # => +3.14
puts sprintf("%-10s|", "hello")   # => hello     |
puts sprintf("%10s|", "hello")    # => "     hello|"
```

```ruby
# Number separator (ด้วย monkey patching หรือ custom method)
def number_with_commas(n)
  n.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse
end

puts number_with_commas(1234567)      # => 1,234,567
puts number_with_commas(9876543210)   # => 9,876,543,210
puts number_with_commas(42)           # => 42

# สำหรับ Float
def format_currency(amount, currency = "฿")
  integer_part, decimal_part = ("%.2f" % amount).split(".")
  "#{currency}#{number_with_commas(integer_part.to_i)}.#{decimal_part}"
end

puts format_currency(1234567.89)      # => ฿1,234,567.89
puts format_currency(42.5)            # => ฿42.50
puts format_currency(9999999.0)       # => ฿9,999,999.00
```

```ruby
# การ format เปอร์เซ็นต์
def format_percent(value, decimals = 2)
  "#{(value * 100).round(decimals)}%"
end

puts format_percent(0.1234)    # => 12.34%
puts format_percent(0.5)       # => 50.0%
puts format_percent(1.0)       # => 100.0%
puts format_percent(0.1234, 0) # => 12%

# การ format filesize
def format_filesize(bytes)
  units = %w[B KB MB GB TB PB]
  size = bytes.to_f
  unit_index = 0
  
  while size >= 1024 && unit_index < units.length - 1
    size /= 1024
    unit_index += 1
  end
  
  if unit_index == 0
    "#{size.to_i} #{units[unit_index]}"
  else
    "#{size.round(2)} #{units[unit_index]}"
  end
end

puts format_filesize(1023)           # => 1023 B
puts format_filesize(1024)           # => 1.0 KB
puts format_filesize(1_048_576)      # => 1.0 MB
puts format_filesize(1_073_741_824)  # => 1.0 GB
puts format_filesize(5_368_709_120)  # => 5.0 GB
```

---

## Step 53: BigDecimal

### การคำนวณที่ต้องการความแม่นยำสูง

```ruby
require 'bigdecimal'
require 'bigdecimal/util'

# ปัญหาของ Float ในการเงิน
price1 = 0.1
price2 = 0.2
puts price1 + price2              # => 0.30000000000000004 (ผิด!)
puts (price1 + price2) == 0.3     # => false (ผิด!)

# แก้ด้วย BigDecimal
bd_price1 = BigDecimal("0.1")
bd_price2 = BigDecimal("0.2")
sum = bd_price1 + bd_price2
puts sum                          # => 0.3e0 (= 0.3)
puts sum.to_s("F")                # => "0.3"
puts sum == BigDecimal("0.3")     # => true (ถูกต้อง!)
```

```ruby
require 'bigdecimal'

# การคำนวณทางการเงิน
def calculate_interest(principal, rate, years)
  p = BigDecimal(principal.to_s)
  r = BigDecimal(rate.to_s)
  n = BigDecimal(years.to_s)
  
  # Compound interest: A = P(1 + r)^n
  amount = p * (1 + r) ** n.to_i
  amount.round(2, BigDecimal::ROUND_HALF_UP)
end

principal = 100_000
rate = 0.05  # 5%
years = 10

result = calculate_interest(principal, rate, years)
puts "เงินต้น: #{principal.to_s}"
puts "ดอกเบี้ย: #{(rate * 100)}% ต่อปี"
puts "ระยะเวลา: #{years} ปี"
puts "ยอดเงิน: #{result.to_s("F")} บาท"
```

```ruby
require 'bigdecimal'

# VAT calculation
def calculate_vat(amount_before_vat, vat_rate = BigDecimal("0.07"))
  amount = BigDecimal(amount_before_vat.to_s)
  rate = BigDecimal(vat_rate.to_s)
  
  vat = (amount * rate).round(2, BigDecimal::ROUND_HALF_UP)
  total = amount + vat
  
  {
    before_vat: amount,
    vat: vat,
    total: total
  }
end

result = calculate_vat(1000)
puts "ก่อน VAT: #{result[:before_vat].to_s("F")} บาท"
puts "VAT 7%:   #{result[:vat].to_s("F")} บาท"
puts "รวม:      #{result[:total].to_s("F")} บาท"
```

```ruby
require 'bigdecimal'
require 'bigdecimal/math'

# BigDecimal Math operations
include BigMath

puts BigMath::PI(50).to_s  # PI ด้วยความแม่นยำ 50 หลัก
puts BigMath::E(20).to_s   # E ด้วยความแม่นยำ 20 หลัก
```

---

## Step 54: Random Numbers

### การสร้างตัวเลขสุ่ม

```ruby
# rand - สุ่มตัวเลข
puts rand        # => 0.12345... (Float ระหว่าง 0.0 และ 1.0)
puts rand(10)    # => 0-9 (Integer ระหว่าง 0 ถึง 9)
puts rand(1..6)  # => 1-6 (สุ่มลูกเต๋า)
puts rand(50..100) # => 50-100

# สุ่มซ้ำได้ (reproducible) ด้วย seed
srand(42)        # ตั้ง seed
puts rand(100)   # => เลขเดิมทุกครั้งถ้า seed เหมือนกัน
puts rand(100)
puts rand(100)
```

```ruby
# Random class - สร้าง random number generator แยก
rng = Random.new        # seed สุ่ม
puts rng.rand(10)       # => 0-9
puts rng.rand(1.0)      # => 0.0 ถึง 1.0

rng2 = Random.new(42)   # seed คงที่
puts rng2.rand(100)     # => เลขเดิมทุกครั้ง
puts rng2.rand(100)

# Random::DEFAULT - global RNG
puts Random::DEFAULT.rand(10)
```

```ruby
# ใช้งานจริง: สุ่มชื่อ, รหัส, etc.
def generate_password(length = 12)
  chars = ('A'..'Z').to_a + ('a'..'z').to_a + ('0'..'9').to_a + %w[! @ # $ %]
  length.times.map { chars.sample }.join
end

puts generate_password        # => e.g., "xK3@mN9!rP2#"
puts generate_password(8)     # => e.g., "aB3!xY7z"

# สุ่มลำดับ
items = ["แอปเปิล", "กล้วย", "ส้ม", "องุ่น", "มะม่วง"]
puts items.shuffle.inspect         # => สุ่มลำดับใหม่ทุกครั้ง
puts items.sample                  # => สุ่ม 1 ชิ้น
puts items.sample(3).inspect       # => สุ่ม 3 ชิ้น
```

```ruby
# Gaussian random (ใกล้เคียงการกระจายปกติ)
def gaussian_rand(mean = 0, std_dev = 1)
  theta = 2 * Math::PI * rand
  rho = Math.sqrt(-2 * Math.log(1 - rand))
  mean + std_dev * rho * Math.cos(theta)
end

# สร้างข้อมูลทดสอบที่มีการกระจายปกติ
scores = 100.times.map { gaussian_rand(75, 10).round.clamp(0, 100) }
puts "เฉลี่ย: #{(scores.sum.to_f / scores.length).round(2)}"
puts "ต่ำสุด: #{scores.min}"
puts "สูงสุด: #{scores.max}"
```

---

## Step 55: Numeric Conversions

### การแปลงประเภทตัวเลข

```ruby
# to_i - แปลงเป็น Integer
puts "42".to_i         # => 42
puts "42.9".to_i       # => 42 (ตัดทศนิยม)
puts "3.14".to_i       # => 3
puts "hello".to_i      # => 0 (ถ้าแปลงไม่ได้)
puts "12abc".to_i      # => 12 (อ่านได้เท่าไหร่ก็เอามา)
puts "0xff".to_i(16)   # => 255 (แปลง hex string)
puts "0b1010".to_i(2)  # => 10 (แปลง binary string)
```

```ruby
# to_f - แปลงเป็น Float
puts "3.14".to_f      # => 3.14
puts "42".to_f        # => 42.0
puts "hello".to_f     # => 0.0
puts "1.5e3".to_f     # => 1500.0

# Integer() vs to_i - ต่างกันยังไง?
puts Integer("42")    # => 42
puts Integer("42.9")  # => ArgumentError! (ไม่ยอมรับ float string)
puts Integer("0xFF")  # => 255 (รับ hex ได้)
# puts Integer("hello") # => ArgumentError (เข้มงวดกว่า to_i)

# Float() vs to_f
puts Float("3.14")    # => 3.14
# puts Float("hello") # => ArgumentError
```

```ruby
# to_r - แปลงเป็น Rational (จำนวนตรรกยะ)
puts 3.to_r         # => 3/1
puts 3.5.to_r       # => 7/2
puts "2/3".to_r     # => 2/3
puts 0.1.to_r       # => 3602879701896397/36028797018963968 (exact float representation!)

# Rational literal
r = Rational(2, 3)
puts r              # => 2/3
puts r.to_f         # => 0.6666666666666666
puts r + Rational(1, 3)  # => 1/1

# ใช้ Rational สำหรับการคำนวณที่แม่นยำ
puts Rational(1, 10) + Rational(2, 10)  # => 3/10 (ไม่มี floating point error!)
```

```ruby
# to_c - แปลงเป็น Complex (จำนวนเชิงซ้อน)
puts 3.to_c         # => (3+0i)
puts Complex(3, 4)  # => (3+4i)

c = Complex(3, 4)
puts c.real         # => 3
puts c.imaginary    # => 4
puts c.abs          # => 5.0 (magnitude = sqrt(3²+4²))
puts c.conjugate    # => (3-4i)
puts c * Complex(1, -1)  # => (7+(-1)i)
```

---

## Step 56: Numeric Comparison

### การเปรียบเทียบตัวเลข

```ruby
# เปรียบเทียบพื้นฐาน
puts 5 == 5.0    # => true (Integer == Float ถ้าค่าเท่ากัน)
puts 5.eql?(5.0) # => false (eql? ตรวจสอบ type ด้วย!)
puts 5.equal?(5) # => true (small integers ชี้ไปที่ object เดียวกัน)

# Spaceship operator
puts 1 <=> 2    # => -1
puts 2 <=> 2    # => 0
puts 3 <=> 2    # => 1
puts 1 <=> "a"  # => nil (ต่างประเภท)

# Comparable methods (ได้จาก Integer/Float ที่ include Comparable)
puts 5.between?(1, 10)  # => true
puts 5.between?(6, 10)  # => false
puts 5.clamp(1, 10)     # => 5
puts 0.clamp(1, 10)     # => 1 (ต่ำกว่า min)
puts 15.clamp(1, 10)    # => 10 (สูงกว่า max)
```

```ruby
# Float comparison pitfall
a = 0.1 + 0.2
b = 0.3
puts a == b          # => false! (floating point error)
puts (a - b).abs < 1e-10  # => true (วิธีที่ถูกต้อง)

# Helper method สำหรับ float comparison
def float_equal?(a, b, epsilon = 1e-10)
  (a - b).abs < epsilon
end

puts float_equal?(0.1 + 0.2, 0.3)   # => true
puts float_equal?(1.0, 1.0000000001) # => true
puts float_equal?(1.0, 1.001)        # => false
```

---

## Step 57: Infinity และ NaN

### ค่าพิเศษของ Float

```ruby
# Infinity
pos_inf = Float::INFINITY
neg_inf = -Float::INFINITY

puts pos_inf             # => Infinity
puts neg_inf             # => -Infinity
puts 1.0 / 0             # => Infinity
puts -1.0 / 0            # => -Infinity

# Operations with Infinity
puts pos_inf + 1         # => Infinity
puts pos_inf * 2         # => Infinity
puts pos_inf - pos_inf   # => NaN
puts 1 / pos_inf         # => 0.0

# Comparison
puts pos_inf > 1_000_000   # => true
puts neg_inf < -1_000_000  # => true
puts pos_inf.infinite?     # => 1
puts neg_inf.infinite?     # => -1
puts 3.14.infinite?        # => nil
```

```ruby
# NaN - Not a Number
nan = Float::NAN
puts nan              # => NaN
puts 0.0 / 0.0        # => NaN

# NaN operations
puts nan + 1          # => NaN
puts nan * 0          # => NaN
puts nan == nan       # => false! (NaN ไม่เท่ากับตัวเอง!)
puts nan.nan?         # => true
puts nan.equal?(nan)  # => true (เป็น object เดียวกัน แต่ == ยังคืน false)

# ตรวจสอบ NaN อย่างถูกต้อง
value = 0.0 / 0.0
puts value.nan?       # => true (วิธีที่ถูกต้อง)
puts value != value   # => true (อีกวิธี: NaN เป็นค่าเดียวที่ != ตัวเอง)
```

```ruby
# ใช้งานจริง: ป้องกัน Infinity และ NaN
def safe_calculation(a, b)
  result = a.to_f / b.to_f
  
  if result.nan?
    raise ArgumentError, "การคำนวณได้ผล NaN"
  elsif result.infinite?
    raise ArgumentError, "การคำนวณได้ผล Infinity"
  end
  
  result
end

begin
  puts safe_calculation(10, 3)    # => 3.333...
  puts safe_calculation(10, 0)    # => ArgumentError
rescue ArgumentError => e
  puts "ข้อผิดพลาด: #{e.message}"
end
```

---

## Step 58: เมธอดเพิ่มเติมของตัวเลข

### Numeric Methods อื่นๆ

```ruby
# divmod - หารแล้วได้ทั้ง quotient และ remainder
puts 13.divmod(4).inspect  # => [3, 1] (13 = 3*4 + 1)
puts 17.divmod(5).inspect  # => [3, 2] (17 = 3*5 + 2)
puts (-7).divmod(3).inspect # => [-3, 2] (-7 = -3*3 + 2)

# fdiv - Float division
puts 7.fdiv(2)      # => 3.5
puts 7.div(2)       # => 3 (integer division)

# modulo vs remainder
puts 13.modulo(4)   # => 1 (เหมือน %)
puts 13.remainder(4) # => 1
puts (-13).modulo(4)  # => 3 (modulo: เครื่องหมายตาม divisor)
puts (-13).remainder(4) # => -1 (remainder: เครื่องหมายตาม dividend)
```

```ruby
# floor_div - integer division ที่ floor แทน truncate
puts 7.div(2)          # => 3
puts (-7).div(2)       # => -4 (floor toward -infinity)
puts (-7).fdiv(2).ceil # => -3 (truncate toward zero)

# pow กับ modulus
puts 2.pow(10)          # => 1024
puts 2.pow(10, 1000)    # => 24 (2^10 mod 1000)

# integer? - ตรวจสอบว่าเป็น integer
puts 42.integer?     # => true
puts 42.0.integer?   # => false (Float ไม่มีเมธอดนี้ใน Ruby)
puts 42.is_a?(Integer)    # => true
puts 42.0.is_a?(Float)    # => true
puts 42.is_a?(Numeric)    # => true (ทั้ง Integer และ Float เป็น Numeric)
```

```ruby
# Numeric#nonzero? - คืนค่า self ถ้าไม่ใช่ 0, คืน nil ถ้าเป็น 0
puts 5.nonzero?    # => 5
puts 0.nonzero?    # => nil (ดีกว่า nil สำหรับ conditional)

# ใช้งาน: หาร ถ้าไม่ใช่ศูนย์
denominator = 0
result = 10.0 / (denominator.nonzero? || 1)
puts result  # => 10.0 (แทนที่จะ divide by zero)

# coerce - แปลงประเภทสำหรับ mixed arithmetic
puts 1.coerce(2.5).inspect  # => [2.5, 1.0] (แปลงทั้งคู่เป็น Float)
```

---

## Step 59: Number Iteration Patterns

### รูปแบบการวนซ้ำด้วยตัวเลข

```ruby
# Fibonacci sequence
def fibonacci(n)
  return n if n <= 1
  
  a, b = 0, 1
  (n - 1).times { a, b = b, a + b }
  b
end

(0..10).each { |i| print "#{fibonacci(i)} " }
puts
# => 0 1 1 2 3 5 8 13 21 34 55

# ด้วย Enumerator::Lazy
fib = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

puts fib.first(10).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

```ruby
# Prime numbers (Sieve of Eratosthenes)
def primes_up_to(n)
  sieve = Array.new(n + 1, true)
  sieve[0] = sieve[1] = false
  
  (2..Math.sqrt(n)).each do |i|
    if sieve[i]
      (i*i..n).step(i) { |j| sieve[j] = false }
    end
  end
  
  (2..n).select { |i| sieve[i] }
end

puts primes_up_to(50).inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]
```

```ruby
# Numeric ranges และ iteration
(1..5).each { |i| print "#{i} " }
puts  # => 1 2 3 4 5

(1...5).each { |i| print "#{i} " }
puts  # => 1 2 3 4

# step
(0..1).step(0.25) { |i| print "#{i} " }
puts  # => 0.0 0.25 0.5 0.75 1.0

# map กับ Range
squares = (1..10).map { |i| i ** 2 }
puts squares.inspect
# => [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
```

---

## Step 60: Numeric Utility Patterns

### รูปแบบการใช้งานตัวเลขในชีวิตจริง

```ruby
# Unit conversion
module UnitConverter
  CONVERSIONS = {
    km_to_miles: 0.621371,
    miles_to_km: 1.60934,
    kg_to_lbs: 2.20462,
    lbs_to_kg: 0.453592,
    celsius_to_fahrenheit: ->(c) { c * 9.0/5 + 32 },
    fahrenheit_to_celsius: ->(f) { (f - 32) * 5.0/9 }
  }
  
  def self.convert(value, type)
    conversion = CONVERSIONS[type]
    case conversion
    when Numeric then value * conversion
    when Proc then conversion.call(value)
    else raise ArgumentError, "Unknown conversion: #{type}"
    end
  end
end

puts UnitConverter.convert(100, :km_to_miles).round(2)      # => 62.14
puts UnitConverter.convert(70, :kg_to_lbs).round(2)         # => 154.32
puts UnitConverter.convert(100, :celsius_to_fahrenheit)      # => 212.0
puts UnitConverter.convert(37, :celsius_to_fahrenheit)       # => 98.6
```

```ruby
# Statistics helper
module Statistics
  def self.mean(numbers)
    numbers.sum.to_f / numbers.length
  end
  
  def self.median(numbers)
    sorted = numbers.sort
    n = sorted.length
    n.odd? ? sorted[n / 2] : (sorted[n/2 - 1] + sorted[n/2]) / 2.0
  end
  
  def self.mode(numbers)
    freq = Hash.new(0)
    numbers.each { |n| freq[n] += 1 }
    max_freq = freq.values.max
    freq.select { |_, v| v == max_freq }.keys
  end
  
  def self.std_dev(numbers)
    m = mean(numbers)
    variance = numbers.sum { |n| (n - m) ** 2 } / numbers.length
    Math.sqrt(variance)
  end
  
  def self.range_stat(numbers)
    numbers.max - numbers.min
  end
end

data = [4, 8, 15, 16, 23, 42, 8, 15, 16, 16]
puts "ข้อมูล: #{data.sort.inspect}"
puts "Mean: #{Statistics.mean(data).round(2)}"
puts "Median: #{Statistics.median(data)}"
puts "Mode: #{Statistics.mode(data).inspect}"
puts "Std Dev: #{Statistics.std_dev(data).round(2)}"
puts "Range: #{Statistics.range_stat(data)}"
```

---

## แบบฝึกหัดตอนที่ 4: Numbers (15 ข้อ)

### ข้อ 1-5: พื้นฐาน

**ข้อ 1**: เขียนโปรแกรมแปลงอุณหภูมิระหว่าง Celsius, Fahrenheit, และ Kelvin

```ruby
# เฉลย
def celsius_to_fahrenheit(c)
  (c * 9.0 / 5) + 32
end

def celsius_to_kelvin(c)
  c + 273.15
end

def fahrenheit_to_celsius(f)
  (f - 32) * 5.0 / 9
end

def kelvin_to_celsius(k)
  k - 273.15
end

temperatures = [0, 100, -40, 37]
puts "%-12s | %-12s | %-12s" % ["Celsius", "Fahrenheit", "Kelvin"]
puts "-" * 40
temperatures.each do |c|
  f = celsius_to_fahrenheit(c)
  k = celsius_to_kelvin(c)
  puts "%-12.1f | %-12.1f | %-12.2f" % [c, f, k]
end
```

**ข้อ 2**: เขียนโปรแกรมคำนวณ BMI

```ruby
# เฉลย
def calculate_bmi(weight_kg, height_cm)
  height_m = height_cm / 100.0
  bmi = weight_kg / (height_m ** 2)
  
  category = case bmi
  when 0...18.5 then "ผอมเกินไป"
  when 18.5...23 then "ปกติ"
  when 23...25   then "ท้วม"
  when 25...30   then "อ้วน"
  else "อ้วนมาก"
  end
  
  { bmi: bmi.round(2), category: category }
end

people = [
  { name: "สมชาย", weight: 70, height: 175 },
  { name: "สมหญิง", weight: 55, height: 165 },
  { name: "สมศักดิ์", weight: 90, height: 170 }
]

people.each do |person|
  result = calculate_bmi(person[:weight], person[:height])
  puts "#{person[:name]}: BMI = #{result[:bmi]} (#{result[:category]})"
end
```

**ข้อ 3**: เขียนโปรแกรมคำนวณดอกเบี้ยแบบต่างๆ

```ruby
# เฉลย
require 'bigdecimal'

def simple_interest(principal, rate, years)
  p = BigDecimal(principal.to_s)
  r = BigDecimal(rate.to_s)
  n = BigDecimal(years.to_s)
  interest = p * r * n
  {
    principal: p,
    interest: interest.round(2),
    total: (p + interest).round(2)
  }
end

def compound_interest(principal, rate, years, n_per_year = 12)
  p = BigDecimal(principal.to_s)
  r = BigDecimal(rate.to_s)
  n = BigDecimal(n_per_year.to_s)
  t = BigDecimal(years.to_s)
  
  # A = P(1 + r/n)^(nt)
  total = p * (1 + r/n) ** (n * t).to_i
  total = total.round(2)
  
  {
    principal: p,
    interest: (total - p).round(2),
    total: total
  }
end

principal = 100_000
rate = 0.05
years = 10

simple = simple_interest(principal, rate, years)
compound = compound_interest(principal, rate, years)

puts "=== เงินต้น: #{principal} บาท, อัตราดอกเบี้ย: #{rate*100}%, #{years} ปี ==="
puts "\nดอกเบี้ยแบบง่าย:"
puts "  ดอกเบี้ย: #{simple[:interest]} บาท"
puts "  ยอดรวม: #{simple[:total]} บาท"
puts "\nดอกเบี้ยแบบทบต้น (คิดรายเดือน):"
puts "  ดอกเบี้ย: #{compound[:interest]} บาท"
puts "  ยอดรวม: #{compound[:total]} บาท"
```

**ข้อ 4**: เขียนฟังก์ชัน is_prime? และสร้าง prime factorization

```ruby
# เฉลย
def prime?(n)
  return false if n < 2
  return true if n == 2
  return false if n.even?
  
  (3..Math.sqrt(n)).step(2).none? { |i| n % i == 0 }
end

def prime_factorization(n)
  factors = []
  divisor = 2
  
  while n > 1
    while n % divisor == 0
      factors << divisor
      n /= divisor
    end
    divisor += divisor == 2 ? 1 : 2
  end
  
  factors
end

numbers = [2, 7, 12, 13, 100, 97, 360]
numbers.each do |n|
  if prime?(n)
    puts "#{n}: เป็น prime"
  else
    factors = prime_factorization(n)
    puts "#{n}: ไม่ใช่ prime = #{factors.join(' × ')}"
  end
end
```

**ข้อ 5**: เขียนโปรแกรมแปลงเลขโรมัน

```ruby
# เฉลย
def to_roman(num)
  values = [1000, 900, 500, 400, 100, 90, 50, 40, 10, 9, 5, 4, 1]
  symbols = %w[M CM D CD C XC L XL X IX V IV I]
  
  result = ""
  values.each_with_index do |value, i|
    while num >= value
      result += symbols[i]
      num -= value
    end
  end
  result
end

def from_roman(roman)
  values = { 'I' => 1, 'V' => 5, 'X' => 10, 'L' => 50,
             'C' => 100, 'D' => 500, 'M' => 1000 }
  
  result = 0
  prev = 0
  roman.upcase.chars.reverse.each do |char|
    current = values[char] || 0
    if current >= prev
      result += current
    else
      result -= current
    end
    prev = current
  end
  result
end

[1, 4, 9, 14, 40, 99, 400, 1994, 2024].each do |n|
  roman = to_roman(n)
  back = from_roman(roman)
  puts "#{n} => #{roman} => #{back}"
end
```

### ข้อ 6-10: Intermediate

**ข้อ 6**: สร้าง number guessing game

```ruby
# เฉลย
def number_guessing_game(min = 1, max = 100)
  secret = rand(min..max)
  attempts = 0
  max_attempts = (Math.log2(max - min + 1) + 1).ceil
  
  puts "เดาตัวเลขระหว่าง #{min} ถึง #{max}"
  puts "คุณมี #{max_attempts} ครั้ง"
  
  loop do
    print "เดา: "
    guess = gets.chomp.to_i
    attempts += 1
    
    if guess == secret
      puts "ถูกต้อง! ใช้ #{attempts} ครั้ง"
      break
    elsif guess < secret
      puts "น้อยเกินไป"
    else
      puts "มากเกินไป"
    end
    
    if attempts >= max_attempts
      puts "หมดโอกาสแล้ว! คำตอบคือ #{secret}"
      break
    end
  end
end

# number_guessing_game  # ยกเลิก comment เพื่อเล่น
puts "Game function defined (requires interactive input)"
```

**ข้อ 7**: คำนวณ statistics ของชุดข้อมูล

```ruby
# เฉลย
def statistics(data)
  raise ArgumentError, "Data cannot be empty" if data.empty?
  
  sorted = data.sort
  n = data.length
  
  mean = data.sum.to_f / n
  
  median = if n.odd?
    sorted[n / 2]
  else
    (sorted[n/2 - 1] + sorted[n/2]) / 2.0
  end
  
  freq = Hash.new(0)
  data.each { |x| freq[x] += 1 }
  max_freq = freq.values.max
  mode = freq.select { |_, v| v == max_freq }.keys
  
  variance = data.sum { |x| (x - mean) ** 2 } / n
  std_dev = Math.sqrt(variance)
  
  q1 = sorted[n / 4]
  q3 = sorted[3 * n / 4]
  iqr = q3 - q1
  
  {
    count: n,
    sum: data.sum,
    min: sorted.first,
    max: sorted.last,
    range: sorted.last - sorted.first,
    mean: mean.round(4),
    median: median,
    mode: mode,
    variance: variance.round(4),
    std_dev: std_dev.round(4),
    q1: q1,
    q3: q3,
    iqr: iqr
  }
end

data = [23, 45, 67, 34, 89, 45, 23, 56, 78, 45, 12, 90, 34, 67, 45]
stats = statistics(data)

puts "=== Statistics ==="
stats.each do |key, value|
  puts "#{key.to_s.ljust(10)}: #{value}"
end
```

**ข้อ 8**: สร้าง number formatter ที่รองรับหลายรูปแบบ

```ruby
# เฉลย
class NumberFormatter
  def initialize(number)
    @number = number
  end
  
  def as_currency(symbol = "฿", decimals = 2)
    formatted = sprintf("%.#{decimals}f", @number)
    integer, decimal = formatted.split(".")
    grouped = integer.gsub(/(\d)(?=(\d{3})+\z)/, '\1,')
    "#{symbol}#{grouped}.#{decimal}"
  end
  
  def as_percent(decimals = 2)
    "#{sprintf("%.#{decimals}f", @number * 100)}%"
  end
  
  def as_ordinal
    n = @number.to_i
    suffix = if (11..13).include?(n % 100)
      "th"
    else
      case n % 10
      when 1 then "st"
      when 2 then "nd"
      when 3 then "rd"
      else "th"
      end
    end
    "#{n}#{suffix}"
  end
  
  def as_words
    # Simple Thai number words
    ones = %w[ศูนย์ หนึ่ง สอง สาม สี่ ห้า หก เจ็ด แปด เก้า]
    n = @number.to_i.abs
    
    return ones[n] if n < 10
    return "#{ones[n/10]}สิบ#{n%10 > 0 ? ones[n%10] : ''}" if n < 100
    "#{n}" # simplified for larger numbers
  end
end

formatter = NumberFormatter.new(1234567.89)
puts formatter.as_currency             # => ฿1,234,567.89
puts formatter.as_currency("$")        # => $1,234,567.89

pct = NumberFormatter.new(0.1567)
puts pct.as_percent                    # => 15.67%

ord = NumberFormatter.new(21)
puts ord.as_ordinal                    # => 21st

thai = NumberFormatter.new(7)
puts thai.as_words                     # => เจ็ด
```

**ข้อ 9**: เขียน matrix operations

```ruby
# เฉลย
class Matrix2D
  attr_reader :rows, :cols, :data
  
  def initialize(data)
    @data = data
    @rows = data.length
    @cols = data[0].length
  end
  
  def [](i, j)
    @data[i][j]
  end
  
  def +(other)
    result = Array.new(@rows) { Array.new(@cols, 0) }
    @rows.times do |i|
      @cols.times do |j|
        result[i][j] = @data[i][j] + other[i, j]
      end
    end
    Matrix2D.new(result)
  end
  
  def *(other)
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
  
  def transpose
    result = Array.new(@cols) { Array.new(@rows, 0) }
    @rows.times do |i|
      @cols.times do |j|
        result[j][i] = @data[i][j]
      end
    end
    Matrix2D.new(result)
  end
  
  def to_s
    @data.map { |row| row.map { |v| v.to_s.rjust(6) }.join(" ") }.join("\n")
  end
end

a = Matrix2D.new([[1, 2], [3, 4]])
b = Matrix2D.new([[5, 6], [7, 8]])

puts "Matrix A:"
puts a
puts "\nMatrix B:"
puts b
puts "\nA + B:"
puts (a + b)
puts "\nA * B:"
puts (a * b)
puts "\nTranspose of A:"
puts a.transpose
```

**ข้อ 10**: สร้าง loan calculator

```ruby
# เฉลย
require 'bigdecimal'

def loan_payment(principal, annual_rate, years)
  p = BigDecimal(principal.to_s)
  r = BigDecimal(annual_rate.to_s) / 12  # monthly rate
  n = years * 12                          # total months
  
  # Monthly payment formula: M = P * [r(1+r)^n] / [(1+r)^n - 1]
  if r == 0
    return p / n
  end
  
  factor = (1 + r) ** n
  monthly = p * (r * factor) / (factor - 1)
  monthly.round(2, BigDecimal::ROUND_HALF_UP)
end

def amortization_schedule(principal, annual_rate, years)
  monthly_payment = loan_payment(principal, annual_rate, years)
  r = BigDecimal(annual_rate.to_s) / 12
  balance = BigDecimal(principal.to_s)
  
  schedule = []
  (years * 12).times do |i|
    interest = (balance * r).round(2, BigDecimal::ROUND_HALF_UP)
    principal_paid = monthly_payment - interest
    balance -= principal_paid
    balance = [balance, BigDecimal("0")].max  # ไม่ให้ติดลบ
    
    schedule << {
      month: i + 1,
      payment: monthly_payment,
      principal: principal_paid.round(2),
      interest: interest,
      balance: balance.round(2)
    }
  end
  
  schedule
end

principal = 1_000_000
rate = 0.06
years = 10

monthly = loan_payment(principal, rate, years)
total = monthly * years * 12
interest_total = total - principal

puts "=== Loan Calculator ==="
puts "เงินกู้: #{principal.to_s} บาท"
puts "อัตราดอกเบี้ย: #{rate * 100}% ต่อปี"
puts "ระยะเวลา: #{years} ปี"
puts "ผ่อนรายเดือน: #{monthly.to_s("F")} บาท"
puts "ดอกเบี้ยรวม: #{interest_total.to_s("F")} บาท"
puts "ยอดชำระรวม: #{total.to_s("F")} บาท"

puts "\n=== ตาราง Amortization (12 เดือนแรก) ==="
schedule = amortization_schedule(principal, rate, years)
puts "%-6s | %-12s | %-12s | %-12s | %-15s" % ["เดือน", "เงินผ่อน", "เงินต้น", "ดอกเบี้ย", "ยอดคงเหลือ"]
puts "-" * 65
schedule.first(12).each do |row|
  puts "%-6d | %-12s | %-12s | %-12s | %-15s" % [
    row[:month],
    row[:payment].to_s("F"),
    row[:principal].to_s("F"),
    row[:interest].to_s("F"),
    row[:balance].to_s("F")
  ]
end
```

### ข้อ 11-15: Advanced

**ข้อ 11**: เขียน base converter (แปลงระหว่างฐานต่างๆ)

```ruby
# เฉลย
def convert_base(number_str, from_base, to_base)
  # แปลงเป็น decimal ก่อน
  decimal = number_str.to_s.upcase.chars.reduce(0) do |sum, digit|
    value = digit.match?(/\d/) ? digit.to_i : digit.ord - 'A'.ord + 10
    raise ArgumentError, "Invalid digit #{digit} for base #{from_base}" if value >= from_base
    sum * from_base + value
  end
  
  # แปลงจาก decimal ไปยัง to_base
  return "0" if decimal == 0
  
  digits = []
  while decimal > 0
    remainder = decimal % to_base
    digits.unshift(remainder < 10 ? remainder.to_s : (remainder - 10 + 'A'.ord).chr)
    decimal /= to_base
  end
  
  digits.join
end

conversions = [
  ["1010", 2, 10],    # binary to decimal
  ["FF", 16, 10],     # hex to decimal
  ["255", 10, 2],     # decimal to binary
  ["255", 10, 16],    # decimal to hex
  ["777", 8, 10],     # octal to decimal
]

conversions.each do |num, from, to|
  result = convert_base(num, from, to)
  puts "#{num} (base #{from}) = #{result} (base #{to})"
end
```

**ข้อ 12**: เขียนโปรแกรมคำนวณ geometric progression

```ruby
# เฉลย
def geometric_sequence(first_term, ratio, n_terms)
  n_terms.times.map { |i| first_term * ratio ** i }
end

def geometric_sum(first_term, ratio, n_terms)
  if ratio == 1
    first_term * n_terms
  else
    first_term * (ratio ** n_terms - 1) / (ratio - 1)
  end
end

def geometric_sum_infinite(first_term, ratio)
  raise ArgumentError, "Ratio must be |r| < 1 for convergence" if ratio.abs >= 1
  first_term / (1.0 - ratio)
end

puts "Geometric Sequence (a=2, r=3, n=5):"
seq = geometric_sequence(2, 3, 5)
puts seq.inspect
puts "Sum: #{geometric_sum(2, 3, 5)}"

puts "\nInfinite Sum (a=1, r=0.5):"
puts geometric_sum_infinite(1, 0.5)  # => 2.0

# Compound interest ก็คือ geometric sequence!
puts "\nCompound Interest (1000 baht, 5% per year, 5 years):"
balances = geometric_sequence(1000, 1.05, 6)
balances.each_with_index do |balance, year|
  puts "Year #{year}: #{'%.2f' % balance} baht"
end
```

**ข้อ 13**: เขียน numerical integration

```ruby
# เฉลย
def trapezoidal_integration(a, b, n, &func)
  h = (b - a).to_f / n
  sum = func.call(a) / 2.0 + func.call(b) / 2.0
  
  (1...n).each do |i|
    sum += func.call(a + i * h)
  end
  
  h * sum
end

def simpson_integration(a, b, n, &func)
  raise ArgumentError, "n must be even" if n.odd?
  
  h = (b - a).to_f / n
  sum = func.call(a) + func.call(b)
  
  (1...n).each do |i|
    x = a + i * h
    sum += i.even? ? 2 * func.call(x) : 4 * func.call(x)
  end
  
  h * sum / 3
end

# ∫₀¹ x² dx = 1/3
puts "∫₀¹ x² dx:"
puts "Trapezoidal: #{trapezoidal_integration(0, 1, 1000) { |x| x**2 }.round(6)}"
puts "Simpson:     #{simpson_integration(0, 1, 1000) { |x| x**2 }.round(6)}"
puts "Exact:       #{1.0/3}"

# ∫₀^π sin(x) dx = 2
puts "\n∫₀^π sin(x) dx:"
puts "Trapezoidal: #{trapezoidal_integration(0, Math::PI, 1000) { |x| Math.sin(x) }.round(6)}"
puts "Simpson:     #{simpson_integration(0, Math::PI, 1000) { |x| Math.sin(x) }.round(6)}"
puts "Exact:       2.0"
```

**ข้อ 14**: เขียน binary search ใน sorted numbers

```ruby
# เฉลย
def binary_search(array, target)
  low, high = 0, array.length - 1
  
  while low <= high
    mid = (low + high) / 2
    
    case array[mid] <=> target
    when 0  then return mid  # found
    when -1 then low = mid + 1   # target is larger
    when 1  then high = mid - 1  # target is smaller
    end
  end
  
  nil  # not found
end

def binary_search_range(array, target)
  # หา index แรกและสุดท้ายที่ target ปรากฎ
  first = find_first(array, target)
  return nil if first.nil?
  
  last = find_last(array, target)
  first..last
end

def find_first(array, target)
  low, high = 0, array.length - 1
  result = nil
  
  while low <= high
    mid = (low + high) / 2
    if array[mid] == target
      result = mid
      high = mid - 1
    elsif array[mid] < target
      low = mid + 1
    else
      high = mid - 1
    end
  end
  
  result
end

def find_last(array, target)
  low, high = 0, array.length - 1
  result = nil
  
  while low <= high
    mid = (low + high) / 2
    if array[mid] == target
      result = mid
      low = mid + 1
    elsif array[mid] < target
      low = mid + 1
    else
      high = mid - 1
    end
  end
  
  result
end

sorted = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
puts binary_search(sorted, 7)    # => 3
puts binary_search(sorted, 13)   # => 6
puts binary_search(sorted, 6)    # => nil

nums = [1, 2, 2, 3, 3, 3, 4, 4, 5]
range = binary_search_range(nums, 3)
puts "3 appears at positions: #{range}"  # => 3..5
```

**ข้อ 15**: Monte Carlo Simulation

```ruby
# เฉลย
def estimate_pi(num_samples)
  inside_circle = 0
  
  num_samples.times do
    x = rand * 2 - 1  # -1 to 1
    y = rand * 2 - 1  # -1 to 1
    inside_circle += 1 if x**2 + y**2 <= 1
  end
  
  4.0 * inside_circle / num_samples
end

def birthday_problem_simulation(num_people, num_trials = 10_000)
  successes = 0
  
  num_trials.times do
    birthdays = num_people.times.map { rand(365) }
    successes += 1 if birthdays.length != birthdays.uniq.length
  end
  
  successes.to_f / num_trials
end

# Estimate Pi
puts "=== Monte Carlo Pi Estimation ==="
[100, 1_000, 10_000, 100_000].each do |n|
  estimate = estimate_pi(n)
  error = ((estimate - Math::PI) / Math::PI * 100).abs
  puts "n=#{n.to_s.ljust(8)}: π ≈ #{estimate.round(6)} (error: #{error.round(4)}%)"
end
puts "Actual π = #{Math::PI}"

# Birthday Problem
puts "\n=== Birthday Problem Simulation ==="
[10, 23, 30, 50].each do |n|
  prob = birthday_problem_simulation(n)
  puts "#{n} คน: #{(prob * 100).round(1)}% โอกาสเกิดวันเดียวกัน"
end
```

---

## สรุป

| เรื่อง | สิ่งที่เรียนรู้ |
|--------|----------------|
| Integer | Decimal, Binary (0b), Hex (0x), Octal (0o), ขนาดไม่จำกัด |
| Float | ทศนิยม, Scientific notation, ค่าพิเศษ (NaN, Infinity) |
| Arithmetic | +, -, *, /, %, ** และ Integer/Float division |
| Integer methods | odd?, even?, abs, gcd, lcm, digits, pow, times, upto, downto |
| Float methods | round, ceil, floor, truncate, nan?, infinite? |
| Math module | sqrt, log, sin, cos, tan, atan2, hypot, PI, E |
| BigDecimal | ความแม่นยำสูงสำหรับการเงิน |
| Random | rand, Random.new, seed |
| Conversions | to_i, to_f, to_r, to_c, Integer(), Float() |
| Formatting | sprintf, format, % operator |

ตอนต่อไปจะเป็นเรื่อง **Arrays** ที่เป็น Collection ที่ทรงพลังที่สุดใน Ruby
