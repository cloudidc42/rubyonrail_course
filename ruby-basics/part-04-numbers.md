# ส่วนที่ 4: Numbers - ขั้นตอนที่ 46-60

## บทนำ

ตัวเลขใน Ruby มีความสามารถมากกว่าภาษาอื่นๆ มาก ไม่ว่าจะเป็น Integer ขนาดใหญ่ไม่จำกัด, Float ที่รองรับ scientific notation, BigDecimal สำหรับการเงิน, และ Rational สำหรับเศษส่วน ในส่วนนี้เราจะเรียนรู้ทุกแง่มุมของตัวเลขใน Ruby

---

## ขั้นตอนที่ 46: Integer vs Float

### Integer (จำนวนเต็ม)

```ruby
# Integer ใน Ruby ไม่มีขีดจำกัดขนาด! (Bignum)
small = 42
big = 1_000_000_000
huge = 2 ** 100   # 1267650600228229401496703205376

puts small.class   # => Integer
puts big.class     # => Integer
puts huge.class    # => Integer
puts huge          # => 1267650600228229401496703205376

# Underscore ช่วยอ่านง่าย
population = 7_900_000_000
price = 1_234_567
hex_color = 0xFF_FF_FF

# Integer ในระบบเลขต่างๆ
decimal = 255
binary  = 0b11111111  # 0b = binary
octal   = 0o377       # 0o = octal (หรือ 0377)
hex     = 0xFF        # 0x = hexadecimal

puts decimal  # => 255
puts binary   # => 255
puts octal    # => 255
puts hex      # => 255

# ตรวจสอบ class
puts 42.class          # => Integer
puts 42.is_a?(Integer) # => true
puts 42.is_a?(Numeric) # => true (Integer < Numeric)
puts 42.instance_of?(Integer) # => true
puts 42.instance_of?(Numeric) # => false
```

### Float (จำนวนทศนิยม)

```ruby
# Float ใน Ruby ใช้ IEEE 754 double precision
pi = 3.14159265358979
e  = 2.71828182845905

puts pi.class   # => Float
puts e.class    # => Float

# Scientific notation
light_speed = 3.0e8    # 300000000.0
electron_mass = 9.1e-31  # 0.00000000000000000000000000000091

puts light_speed    # => 300000000.0
puts electron_mass  # => 9.1e-31

# Float limits
puts Float::MAX       # => 1.7976931348623157e+308
puts Float::MIN       # => 2.2250738585072014e-308
puts Float::EPSILON   # => 2.220446049250313e-16 (ความแม่นยำสูงสุด)
puts Float::DIG       # => 15 (จำนวน significant digits)

# Float constants พิเศษ
puts Float::INFINITY    # => Infinity
puts -Float::INFINITY   # => -Infinity
puts Float::NAN         # => NaN

# สร้าง Infinity
puts 1.0 / 0     # => Infinity
puts -1.0 / 0    # => -Infinity
puts 0.0 / 0     # => NaN

# ตรวจสอบ special values
puts (1.0/0).infinite?    # => 1 (positive infinity)
puts (-1.0/0).infinite?   # => -1 (negative infinity)
puts (0.0/0).nan?         # => true
puts 3.14.finite?         # => true
puts (1.0/0).finite?      # => false
```

### Integer Division vs Float Division

```ruby
# Integer / Integer = Integer (ปัดลง)
puts 10 / 3     # => 3 (ไม่ใช่ 3.333...)
puts 7 / 2      # => 3
puts -7 / 2     # => -4 (ปัดไปทางลบอินฟินิตี้)
puts -7 / -2    # => 3

# Float / Integer = Float
puts 10.0 / 3   # => 3.3333333333333335
puts 7.0 / 2    # => 3.5

# Integer / Float = Float
puts 10 / 3.0   # => 3.3333333333333335

# แปลงก่อน divide
puts 10.to_f / 3        # => 3.3333333333333335
puts Rational(10, 3)    # => 10/3 (exact!)
puts Rational(10, 3).to_f  # => 3.3333333333333335

# divmod - คืนทั้ง quotient และ remainder
q, r = 17.divmod(5)
puts "17 ÷ 5 = #{q} เศษ #{r}"   # => 17 ÷ 5 = 3 เศษ 2

# Modulo (%)
puts 17 % 5    # => 2
puts -17 % 5   # => 3 (Ruby: เครื่องหมายตามตัวหาร)
puts 17 % -5   # => -3

# Integer#remainder (เครื่องหมายตามตัวตั้ง)
puts 17.remainder(5)    # => 2
puts (-17).remainder(5) # => -2
```

---

## ขั้นตอนที่ 47: Arithmetic Operations

### การดำเนินการพื้นฐาน

```ruby
a = 20
b = 7

# พื้นฐาน
puts a + b    # => 27 (บวก)
puts a - b    # => 13 (ลบ)
puts a * b    # => 140 (คูณ)
puts a / b    # => 2 (หาร integer)
puts a % b    # => 6 (modulo)
puts a ** b   # => 1280000000 (ยกกำลัง)

# Assignment operators
x = 10
x += 5    # x = x + 5
puts x    # => 15

x -= 3    # x = x - 3
puts x    # => 12

x *= 2    # x = x * 2
puts x    # => 24

x /= 4    # x = x / 4
puts x    # => 6

x %= 4    # x = x % 4
puts x    # => 2

x **= 3   # x = x ** 3
puts x    # => 8

# Order of operations (PEMDAS/BODMAS)
puts 2 + 3 * 4      # => 14 (คูณก่อน)
puts (2 + 3) * 4    # => 20 (วงเล็บก่อน)
puts 2 ** 3 ** 2    # => 512 (right-associative: 2^(3^2) = 2^9)
puts (2 ** 3) ** 2  # => 64

# Floating point arithmetic
puts 0.1 + 0.2          # => 0.30000000000000004 (!)
puts (0.1 + 0.2).round(1) == 0.3  # => true

# Precise arithmetic
require 'bigdecimal'
a = BigDecimal("0.1")
b = BigDecimal("0.2")
puts (a + b).to_s    # => "0.3e0"
puts (a + b) == BigDecimal("0.3")  # => true

# Integer overflow (Ruby ไม่มี overflow!)
puts 2 ** 62           # => 4611686018427387904
puts 2 ** 63           # => 9223372036854775808
puts 2 ** 64           # => 18446744073709551616
puts 2 ** 1000         # ใหญ่ได้ไม่จำกัด!

# Numeric methods
puts 42.zero?       # => false
puts 0.zero?        # => true
puts 42.nonzero?    # => 42 (คืนตัวเอง ถ้าไม่เป็น 0)
puts 0.nonzero?     # => nil
puts 42.positive?   # => true
puts (-42).negative? # => true
puts 42.abs         # => 42
puts (-42).abs      # => 42

# Divmod and related
puts 17.gcd(12)    # => 1 (Greatest Common Divisor)
puts 4.gcd(6)      # => 2
puts 4.lcm(6)      # => 12 (Least Common Multiple)
puts 17.gcdlcm(12).inspect  # => [1, 204]
```

---

## ขั้นตอนที่ 48: Math Module

### Ruby's Built-in Math Library

```ruby
require 'cmath'  # สำหรับ Complex numbers

# Trigonometric functions (radians)
puts Math::PI          # => 3.141592653589793
puts Math::E           # => 2.718281828459045

puts Math.sin(0)              # => 0.0
puts Math.sin(Math::PI / 2)   # => 1.0
puts Math.cos(0)              # => 1.0
puts Math.cos(Math::PI)       # => -1.0
puts Math.tan(Math::PI / 4)   # => 1.0

# Inverse trig
puts Math.asin(1)             # => π/2
puts Math.acos(1)             # => 0.0
puts Math.atan(1)             # => π/4
puts Math.atan2(1, 1)         # => π/4 (y, x)

# ตัวอย่าง: หาระยะทาง Pythagorean
def distance(x1, y1, x2, y2)
  Math.sqrt((x2 - x1)**2 + (y2 - y1)**2)
end

puts distance(0, 0, 3, 4).round(2)   # => 5.0
puts distance(1, 1, 4, 5).round(2)   # => 5.0

# Exponential and logarithm
puts Math.exp(1)          # => e^1 = 2.718...
puts Math.exp(2)          # => e^2 = 7.389...

puts Math.log(Math::E)    # => 1.0 (natural log)
puts Math.log(100, 10)    # => 2.0 (log base 10)
puts Math.log2(8)         # => 3.0 (log base 2)
puts Math.log10(1000)     # => 3.0

# Square root and powers
puts Math.sqrt(9)         # => 3.0
puts Math.sqrt(2)         # => 1.4142135623730951
puts Math.cbrt(27)        # => 3.0 (cube root - Ruby 2.7+)

# Hyperbolic functions
puts Math.sinh(0)   # => 0.0
puts Math.cosh(0)   # => 1.0
puts Math.tanh(0)   # => 0.0

# Math module ใน class
class Circle
  include Math  # include เพื่อใช้โดยไม่ต้อง Math.

  attr_reader :radius

  def initialize(radius)
    @radius = radius
  end

  def area
    PI * @radius ** 2
  end

  def circumference
    2 * PI * @radius
  end

  def diagonal
    2 * @radius
  end
end

c = Circle.new(5)
printf("รัศมี: %.1f\n", c.radius)
printf("พื้นที่: %.4f\n", c.area)
printf("เส้นรอบวง: %.4f\n", c.circumference)
printf("เส้นผ่านศูนย์กลาง: %.1f\n", c.diagonal)
```

### การคำนวณทางคณิตศาสตร์

```ruby
# Fibonacci sequence
def fibonacci(n)
  return [0, 1].first(n) if n <= 2
  fibs = [0, 1]
  (n - 2).times { fibs << fibs[-1] + fibs[-2] }
  fibs
end

puts fibonacci(10).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Factorial
def factorial(n)
  return 1 if n <= 1
  n * factorial(n - 1)
end

# หรือแบบ iterative
def factorial_iter(n)
  (1..n).reduce(1, :*)
end

puts factorial(10)       # => 3628800
puts factorial_iter(10)  # => 3628800
puts factorial(20)       # => 2432902008176640000

# Prime numbers
def sieve_of_eratosthenes(max)
  primes = Array.new(max + 1, true)
  primes[0] = primes[1] = false

  (2..Math.sqrt(max).to_i).each do |i|
    if primes[i]
      (i*i..max).step(i) { |j| primes[j] = false }
    end
  end

  primes.each_index.select { |i| primes[i] }
end

primes_100 = sieve_of_eratosthenes(100)
puts "จำนวนเฉพาะ 1-100: #{primes_100.join(', ')}"
puts "จำนวน: #{primes_100.length} ตัว"

# Combinatorics
def permutations(n, r)
  factorial(n) / factorial(n - r)
end

def combinations(n, r)
  factorial(n) / (factorial(r) * factorial(n - r))
end

puts "P(5,3) = #{permutations(5, 3)}"  # => 60
puts "C(5,3) = #{combinations(5, 3)}"  # => 10
```

---

## ขั้นตอนที่ 49: Number Formatting

```ruby
# Basic formatting
num = 1234567.8901

printf("Default:     %f\n", num)      # => 1234567.890100
printf("2 decimals:  %.2f\n", num)    # => 1234567.89
printf("Scientific:  %e\n", num)      # => 1.234568e+06
printf("Width 15:    %15.2f\n", num)  # =>       1234567.89
printf("Zero pad:    %015.2f\n", num) # => 0000001234567.89

# Thousands separator (Ruby ไม่มีใน stdlib - ต้องเขียนเอง)
def number_with_commas(number, decimals: 2)
  parts = sprintf("%.#{decimals}f", number).split(".")
  integer_part = parts[0].reverse.gsub(/(\d{3})(?=\d)/, '\1,').reverse
  decimals > 0 ? "#{integer_part}.#{parts[1]}" : integer_part
end

puts number_with_commas(1234567.89)       # => 1,234,567.89
puts number_with_commas(1234567, decimals: 0)  # => 1,234,567
puts number_with_commas(0.5)              # => 0.50

# Currency formatting
def format_currency(amount, currency: "฿", decimals: 2)
  formatted = number_with_commas(amount.abs, decimals: decimals)
  sign = amount < 0 ? "-" : ""
  "#{sign}#{currency}#{formatted}"
end

puts format_currency(1234567.89)          # => ฿1,234,567.89
puts format_currency(-500.5, currency: "$")  # => -$500.50
puts format_currency(0)                   # => ฿0.00

# Percentage
def format_percent(value, decimals: 1)
  "#{sprintf("%.#{decimals}f", value * 100)}%"
end

puts format_percent(0.1234)     # => 12.3%
puts format_percent(0.9999, decimals: 2)  # => 99.99%

# Scientific notation
def scientific_notation(n, sig_digits: 3)
  sprintf("%.#{sig_digits - 1}e", n)
end

puts scientific_notation(12345.678)     # => 1.23e+04
puts scientific_notation(0.000123, sig_digits: 4)  # => 1.230e-04

# Human-readable file sizes
def humanize_bytes(bytes)
  units = ['B', 'KB', 'MB', 'GB', 'TB', 'PB']
  return "0 B" if bytes == 0

  exp = (Math.log(bytes) / Math.log(1024)).floor
  exp = [exp, units.length - 1].min

  value = bytes.to_f / (1024 ** exp)
  "#{value.round(2)} #{units[exp]}"
end

[0, 512, 1024, 1536, 1_048_576, 1_073_741_824, 5_368_709_120].each do |size|
  puts "#{size.to_s.rjust(15)} bytes = #{humanize_bytes(size)}"
end
```

---

## ขั้นตอนที่ 50: Integer Methods

```ruby
# Predicate methods
puts 42.even?     # => true
puts 43.odd?      # => true
puts 0.zero?      # => true
puts 42.positive? # => true
puts (-5).negative? # => true

# Math-related
puts 42.abs       # => 42
puts (-42).abs    # => 42
puts 42.ceil      # => 42 (เหมือนกัน สำหรับ Integer)
puts 42.floor     # => 42

# Integer arithmetic
puts 42.gcd(18)   # => 6 (GCF)
puts 12.lcm(18)   # => 36 (LCM)

# Conversion
puts 42.to_f      # => 42.0
puts 42.to_r      # => 42/1 (Rational)
puts 42.to_c      # => (42+0i) (Complex)

# Base conversion
puts 42.to_s      # => "42"
puts 42.to_s(2)   # => "101010" (binary)
puts 42.to_s(8)   # => "52" (octal)
puts 42.to_s(16)  # => "2a" (hex)
puts 42.to_s(36)  # => "16" (base 36)

# digits - แยกหลัก
puts 1234.digits.inspect     # => [4, 3, 2, 1] (จากขวาไปซ้าย)
puts 1234.digits(10).inspect # => [4, 3, 2, 1]
puts 1234.digits(16).inspect # => [2, 4, 4] (4*256 + 4*16 + 2)

# bit operations
puts 0b1010 & 0b1100   # => 8  (AND: 0b1000)
puts 0b1010 | 0b1100   # => 14 (OR:  0b1110)
puts 0b1010 ^ 0b1100   # => 6  (XOR: 0b0110)
puts ~0b1010           # => -11 (NOT)
puts 0b0101 << 2       # => 20 (left shift)
puts 0b1000 >> 2       # => 2  (right shift)

# Integer iteration
3.times { |i| print "#{i} " }     # 0 1 2
puts
1.upto(5) { |i| print "#{i} " }   # 1 2 3 4 5
puts
5.downto(1) { |i| print "#{i} " } # 5 4 3 2 1
puts
1.step(10, 2) { |i| print "#{i} " }  # 1 3 5 7 9
puts
10.step(1, -3) { |i| print "#{i} " } # 10 7 4 1
puts

# Chaining
result = 5.times.map { |i| i * 2 }
puts result.inspect  # => [0, 2, 4, 6, 8]

sum_of_squares = 10.times.map { |i| i**2 }.sum
puts sum_of_squares  # => 285

# Integer.sqrt (Ruby 2.5+)
puts Integer.sqrt(9)   # => 3
puts Integer.sqrt(10)  # => 3 (floor of sqrt)

# pow with modulo (faster for large numbers)
puts 2.pow(100, 1_000_000_007)  # modular exponentiation
```

---

## ขั้นตอนที่ 51: Float Methods และ Precision Issues

```ruby
# Float rounding
n = 3.14159265

puts n.ceil          # => 4 (ปัดขึ้น)
puts n.floor         # => 3 (ปัดลง)
puts n.round         # => 3 (ปัดสี่)
puts n.round(2)      # => 3.14
puts n.round(4)      # => 3.1416
puts n.truncate      # => 3 (ตัดทศนิยมทิ้ง)
puts n.truncate(2)   # => 3.14

# รูปแบบการปัด
puts 2.5.round       # => 3 (ปัดขึ้น)
puts 3.5.round       # => 4 (ปัดขึ้น)
puts 2.5.round(half: :up)    # => 3
puts 2.5.round(half: :down)  # => 2
puts 2.5.round(half: :even)  # => 2 (banker's rounding)
puts 3.5.round(half: :even)  # => 4

# Absolute value
puts (-3.14).abs     # => 3.14

# Float::INFINITY operations
puts Float::INFINITY + 1     # => Infinity
puts Float::INFINITY * -1    # => -Infinity
puts Float::INFINITY - Float::INFINITY  # => NaN

# Float precision problems - สิ่งที่ต้องระวัง!
puts 0.1 + 0.2                    # => 0.30000000000000004
puts 0.1 + 0.2 == 0.3             # => false !!!

# วิธีเปรียบเทียบ Float
def approximately_equal?(a, b, epsilon: Float::EPSILON)
  (a - b).abs < epsilon
end

puts approximately_equal?(0.1 + 0.2, 0.3)  # => false (epsilon เล็กเกินไป)
puts approximately_equal?(0.1 + 0.2, 0.3, epsilon: 1e-10)  # => true

# หรือใช้ round
puts (0.1 + 0.2).round(10) == 0.3.round(10)  # => true

# Float arithmetic gotchas
money = 0.0
1000.times { money += 0.001 }
puts money           # => 0.9999999999999998 (ไม่ใช่ 1.0!)
puts money.round(10) # => 1.0

# Division by zero
puts 1.0 / 0    # => Infinity (ไม่ error!)
puts -1.0 / 0   # => -Infinity
puts 0.0 / 0    # => NaN

# 0.0/0 ≠ 0.0/0 (NaN ≠ NaN)
nan = 0.0 / 0
puts nan == nan   # => false!
puts nan.nan?     # => true
```

---

## ขั้นตอนที่ 52: BigDecimal สำหรับการเงิน

```ruby
require 'bigdecimal'
require 'bigdecimal/util'  # เพิ่ม to_d method

# สร้าง BigDecimal
price = BigDecimal("99.99")
tax_rate = BigDecimal("0.07")  # 7% VAT

# ควรสร้างจาก string ไม่ใช่ float!
# BigDecimal(0.1)   => 0.1000000000000000055511... (ผิด!)
# BigDecimal("0.1") => 0.1 (ถูก!)

puts BigDecimal("0.1") + BigDecimal("0.2")  # => 0.3e0 (ถูกต้อง!)
puts (BigDecimal("0.1") + BigDecimal("0.2")) == BigDecimal("0.3")  # => true!

# การคำนวณเงิน
class Money
  include Comparable

  attr_reader :amount, :currency

  def initialize(amount, currency: "THB")
    @amount = BigDecimal(amount.to_s)
    @currency = currency
  end

  def +(other)
    check_currency!(other)
    Money.new(@amount + other.amount, currency: @currency)
  end

  def -(other)
    check_currency!(other)
    Money.new(@amount - other.amount, currency: @currency)
  end

  def *(multiplier)
    Money.new(@amount * multiplier, currency: @currency)
  end

  def /(divisor)
    Money.new(@amount / divisor, currency: @currency)
  end

  def <=>(other)
    check_currency!(other)
    @amount <=> other.amount
  end

  def to_s
    formatted = @amount.round(2).to_s("F")
    integer, decimal = formatted.split(".")
    with_commas = integer.reverse.gsub(/(\d{3})(?=\d)/, '\1,').reverse
    "#{@currency} #{with_commas}.#{decimal}"
  end

  def inspect
    "#<Money: #{to_s}>"
  end

  private

  def check_currency!(other)
    raise "Currency mismatch: #{@currency} vs #{other.currency}" unless @currency == other.currency
  end
end

# ตัวอย่างการใช้งาน
price = Money.new("1299.00")
discount = Money.new("100.00")
tax_rate = BigDecimal("0.07")

subtotal = price - discount
tax = subtotal * tax_rate
total = subtotal + tax

puts "ราคา:      #{price}"
puts "ส่วนลด:    -#{discount}"
puts "ราคารวม:   #{subtotal}"
puts "VAT 7%:   #{tax}"
puts "ยอดรวม:   #{total}"

# BigDecimal precision
require 'bigdecimal/math'
include BigMath

# คำนวณ PI ด้วยความแม่นยำสูง
pi = PI(50)
puts pi.to_s  # แสดง PI ด้วย 50 digits

# sqrt ที่แม่นยำ
sqrt2 = BigDecimal("2").sqrt(30)
puts sqrt2.to_s  # sqrt(2) ด้วย 30 digits

# Financial calculations
def compound_interest(principal:, rate:, years:, compounds_per_year: 12)
  p = BigDecimal(principal.to_s)
  r = BigDecimal(rate.to_s)
  n = BigDecimal(compounds_per_year.to_s)
  t = BigDecimal(years.to_s)

  p * (1 + r / n) ** (n * t)
end

result = compound_interest(
  principal: 100_000,
  rate: 0.05,  # 5% per year
  years: 10
)

puts "เงินต้น: ฿100,000"
puts "อัตราดอกเบี้ย: 5% ต่อปี"
puts "ระยะเวลา: 10 ปี"
puts "เงินสุดท้าย: ฿#{result.round(2).to_s('F')}"
```

---

## ขั้นตอนที่ 53: Number Bases

```ruby
# Base literals
decimal = 255
binary  = 0b11111111  # 0b prefix
octal   = 0o377       # 0o prefix
hex     = 0xFF        # 0x prefix

puts decimal  # => 255
puts binary   # => 255
puts octal    # => 255
puts hex      # => 255

# Conversion
puts 255.to_s(2)   # => "11111111" (decimal to binary string)
puts 255.to_s(8)   # => "377"      (decimal to octal string)
puts 255.to_s(16)  # => "ff"       (decimal to hex string)

# Parse strings in different bases
puts "11111111".to_i(2)   # => 255 (binary string to decimal)
puts "377".to_i(8)         # => 255
puts "ff".to_i(16)         # => 255
puts "FF".to_i(16)         # => 255

# Integer() with base prefix in string
puts Integer("0b11111111")  # => 255
puts Integer("0xFF")        # => 255
puts Integer("0o377")       # => 255

# ตัวอย่าง: Color converter
def hex_to_rgb(hex_color)
  hex = hex_color.gsub("#", "")
  r = hex[0..1].to_i(16)
  g = hex[2..3].to_i(16)
  b = hex[4..5].to_i(16)
  { r: r, g: g, b: b }
end

def rgb_to_hex(r, g, b)
  "#%02X%02X%02X" % [r, g, b]
end

colors = ["#FF0000", "#00FF00", "#0000FF", "#FFFFFF", "#000000"]
colors.each do |hex|
  rgb = hex_to_rgb(hex)
  back_to_hex = rgb_to_hex(rgb[:r], rgb[:g], rgb[:b])
  puts "#{hex} => RGB(#{rgb[:r]}, #{rgb[:g]}, #{rgb[:b]}) => #{back_to_hex}"
end

# IP Address ใน binary/hex
def ip_to_binary(ip)
  ip.split(".").map { |octet| octet.to_i.to_s(2).rjust(8, '0') }.join(".")
end

def ip_to_hex(ip)
  ip.split(".").map { |octet| octet.to_i.to_s(16).rjust(2, '0') }.join("")
end

ip = "192.168.1.100"
puts "IP: #{ip}"
puts "Binary: #{ip_to_binary(ip)}"
puts "Hex: 0x#{ip_to_hex(ip).upcase}"

# Bitwise operations ตัวอย่างจริง
# Flags
READ  = 0b001  # 1
WRITE = 0b010  # 2
EXEC  = 0b100  # 4

permissions = READ | WRITE  # User has read and write
puts permissions.to_s(2)     # => "11"
puts (permissions & READ).positive?   # => true (has read)
puts (permissions & EXEC).positive?   # => false (no exec)

# Set EXEC permission
permissions |= EXEC
puts permissions.to_s(2)  # => "111"

# Remove WRITE permission
permissions &= ~WRITE
puts permissions.to_s(2)  # => "101"
```

---

## ขั้นตอนที่ 54: Numeric Conversions

```ruby
# String to Number
puts "42".to_i          # => 42
puts "42.5".to_f        # => 42.5
puts "42.5".to_i        # => 42 (truncate)
puts "42abc".to_i       # => 42 (หยุดที่ไม่ใช่ตัวเลข)
puts "abc42".to_i       # => 0 (ขึ้นต้นด้วยไม่ใช่ตัวเลข)
puts "  42  ".to_i      # => 42 (strip whitespace)
puts "3.14e2".to_f      # => 314.0 (scientific notation)

# Strict conversion
begin
  Integer("42abc")      # ArgumentError!
rescue ArgumentError => e
  puts "Error: #{e.message}"
end

puts Integer("42")      # => 42
puts Integer(42.9)      # => 42 (truncate)
puts Float("3.14")      # => 3.14

# Number to String
puts 42.to_s            # => "42"
puts 42.to_s(2)         # => "101010" (binary)
puts 42.to_s(16)        # => "2a" (hex)
puts 3.14.to_s          # => "3.14"

# Integer to Float/Rational/Complex
puts 42.to_f            # => 42.0
puts 42.to_r            # => 42/1
puts 42.to_c            # => (42+0i)

# Float to Integer
puts 3.7.to_i           # => 3 (truncate)
puts 3.7.ceil           # => 4
puts 3.7.floor          # => 3
puts 3.7.round          # => 4

# Rational arithmetic
a = Rational(1, 3)
b = Rational(1, 6)

puts a + b    # => 1/2
puts a - b    # => 1/6
puts a * b    # => 1/18
puts a / b    # => 2/1

puts (a + b).to_f  # => 0.5

# สร้าง Rational
puts Rational(2, 4)    # => 1/2 (auto-reduce)
puts 1.to_r / 3        # => 1/3
puts "1/3".to_r        # => 1/3

# Complex numbers
c1 = Complex(3, 4)     # 3 + 4i
c2 = Complex(1, -2)    # 1 - 2i

puts c1 + c2   # => (4+2i)
puts c1 * c2   # => (11-2i)
puts c1.abs    # => 5.0 (magnitude)
puts c1.real   # => 3
puts c1.imaginary  # => 4
puts c1.conjugate  # => (3-4i)

# Coerce - Ruby's implicit conversion
class Celsius
  attr_reader :value

  def initialize(value)
    @value = value.to_f
  end

  def +(other)
    case other
    when Celsius
      Celsius.new(@value + other.value)
    when Numeric
      Celsius.new(@value + other)
    end
  end

  def coerce(other)
    [Celsius.new(other), self]
  end

  def to_f
    @value
  end

  def to_s
    "#{@value}°C"
  end
end

temp = Celsius.new(20)
puts temp + 5              # => 25.0°C
puts temp + Celsius.new(3) # => 23.0°C
puts 5 + temp              # => 25.0°C (coerce!)
```

---

## ขั้นตอนที่ 55: Random Numbers

```ruby
# Kernel#rand
puts rand          # => 0.0 to 1.0 (Float)
puts rand(10)      # => 0 to 9 (Integer)
puts rand(1..10)   # => 1 to 10 (inclusive)
puts rand(1...10)  # => 1 to 9 (exclusive)

# Random class
rng = Random.new
puts rng.rand(100)

# Seed สำหรับ reproducible results
rng_seeded = Random.new(42)
5.times { print "#{rng_seeded.rand(100)} " }
puts

# Reload ด้วย seed เดิม = ผลเหมือนเดิม
rng_seeded2 = Random.new(42)
5.times { print "#{rng_seeded2.rand(100)} " }
puts

# Random.srand - set global seed
srand(12345)
5.times { print "#{rand(100)} " }
puts

# Array sampling
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

puts arr.sample         # => random element
puts arr.sample(3).inspect  # => 3 random elements (no duplicates)

shuffled = arr.shuffle
puts shuffled.inspect

# Weighted random selection
def weighted_sample(weights)
  total = weights.values.sum
  threshold = rand * total
  cumulative = 0

  weights.each do |item, weight|
    cumulative += weight
    return item if cumulative >= threshold
  end

  weights.keys.last
end

prizes = {
  "รางวัลที่ 1 (ทอง)" => 1,
  "รางวัลที่ 2 (เงิน)" => 3,
  "รางวัลที่ 3 (ทองแดง)" => 10,
  "ไม่ได้รับรางวัล" => 86
}

results = Hash.new(0)
10000.times { results[weighted_sample(prizes)] += 1 }

puts "ผลการสุ่ม 10,000 ครั้ง:"
results.sort_by { |_, v| -v }.each do |prize, count|
  bar = "█" * (count / 100)
  puts "  #{prize.ljust(25)} #{count.to_s.rjust(5)} (#{(count/100.0).round(1)}%) #{bar}"
end

# Secure random (สำหรับ security)
require 'securerandom'

puts SecureRandom.uuid    # => "a8098c1a-f86e-11da-bd1a-00112444be1e"
puts SecureRandom.hex(16)  # => "16 random hex chars"
puts SecureRandom.base64   # => random base64
puts SecureRandom.random_number(100)  # => 0-99
puts SecureRandom.random_bytes(8).bytes.inspect  # => 8 random bytes

# Monte Carlo Pi estimation
def estimate_pi(samples: 1_000_000)
  inside = samples.times.count do
    x = rand
    y = rand
    x**2 + y**2 <= 1
  end
  4.0 * inside / samples
end

estimated = estimate_pi(samples: 1_000_000)
puts "Estimated PI: #{estimated}"
puts "Actual PI:    #{Math::PI}"
puts "Error: #{((estimated - Math::PI).abs / Math::PI * 100).round(4)}%"
```

---

## ขั้นตอนที่ 56: Mathematical Constants และ Special Values

```ruby
# Math constants
puts Math::PI    # => 3.141592653589793
puts Math::E     # => 2.718281828459045

# Float constants
puts Float::INFINITY   # => Infinity
puts Float::NAN        # => NaN
puts Float::EPSILON    # => 2.220446049250313e-16
puts Float::DIG        # => 15 (significant decimal digits)
puts Float::MANT_DIG   # => 53 (binary mantissa digits)
puts Float::MAX_EXP    # => 1024
puts Float::MIN_EXP    # => -1021
puts Float::MAX        # => 1.7976931348623157e+308
puts Float::MIN        # => 2.2250738585072014e-308
puts Float::RADIX      # => 2

# Integer constants (Ruby 2.0+)
puts Integer::GMP_VERSION  # ถ้า Ruby compiled กับ GMP

# ตัวอย่างการใช้ constants ในการคำนวณ
def circle_properties(radius)
  {
    radius:        radius,
    diameter:      2 * radius,
    circumference: 2 * Math::PI * radius,
    area:          Math::PI * radius**2,
    volume_sphere: (4.0/3) * Math::PI * radius**3
  }
end

props = circle_properties(5)
props.each do |key, value|
  puts "#{key.to_s.ljust(15)}: #{value.is_a?(Float) ? value.round(4) : value}"
end

# Golden ratio
golden_ratio = (1 + Math.sqrt(5)) / 2
puts "\nGolden Ratio φ = #{golden_ratio}"

# Euler's identity: e^(iπ) + 1 = 0
include CMath  # Complex Math
result = exp(Complex(0, 1) * PI) + 1
puts "e^(iπ) + 1 = #{result.real.round(10) + result.imaginary.round(10) * 1i}"

# Natural constants
AVOGADRO   = 6.02214076e23   # Avogadro's number
BOLTZMANN  = 1.380649e-23    # Boltzmann constant
PLANCK     = 6.62607015e-34  # Planck constant
SPEED_LIGHT = 299_792_458    # Speed of light (m/s)

puts "\nกฎของ Einstein: E = mc²"
mass = 1.0  # 1 kg
energy = mass * SPEED_LIGHT**2
puts "1 kg = #{energy} Joules"
puts "      = #{scientific_notation(energy, sig_digits: 4)} J"

def scientific_notation(n, sig_digits: 3)
  sprintf("%.#{sig_digits - 1}e", n)
end
```

---

## ขั้นตอนที่ 57-60: แบบฝึกหัด 15 ข้อ

### แบบฝึกหัดที่ 1: Unit Converter

```ruby
module UnitConverter
  CONVERSIONS = {
    length: {
      meter: 1.0,
      kilometer: 1000.0,
      centimeter: 0.01,
      millimeter: 0.001,
      mile: 1609.344,
      yard: 0.9144,
      foot: 0.3048,
      inch: 0.0254
    },
    weight: {
      kilogram: 1.0,
      gram: 0.001,
      pound: 0.453592,
      ounce: 0.0283495,
      ton: 1000.0
    },
    temperature: {
      celsius: :special,
      fahrenheit: :special,
      kelvin: :special
    }
  }

  def self.convert(value, from_unit, to_unit, type:)
    if type == :temperature
      convert_temperature(value, from_unit, to_unit)
    else
      units = CONVERSIONS[type]
      in_base = value * units[from_unit]
      (in_base / units[to_unit]).round(6)
    end
  end

  def self.convert_temperature(value, from, to)
    celsius = case from
              when :celsius    then value
              when :fahrenheit then (value - 32) * 5.0 / 9
              when :kelvin     then value - 273.15
              end

    case to
    when :celsius    then celsius.round(4)
    when :fahrenheit then (celsius * 9.0 / 5 + 32).round(4)
    when :kelvin     then (celsius + 273.15).round(4)
    end
  end
end

# Length conversions
puts "=== Length ==="
puts "1 mile = #{UnitConverter.convert(1, :mile, :kilometer, type: :length)} km"
puts "1 foot = #{UnitConverter.convert(1, :foot, :centimeter, type: :length)} cm"
puts "180 cm = #{UnitConverter.convert(180, :centimeter, :inch, type: :length).round(2)} inches"

puts "\n=== Temperature ==="
puts "100°C = #{UnitConverter.convert(100, :celsius, :fahrenheit, type: :temperature)}°F"
puts "32°F = #{UnitConverter.convert(32, :fahrenheit, :celsius, type: :temperature)}°C"
puts "0°C = #{UnitConverter.convert(0, :celsius, :kelvin, type: :temperature)}K"
```

### แบบฝึกหัดที่ 2: Financial Calculator

```ruby
require 'bigdecimal'

class FinancialCalculator
  # Simple Interest: I = P × R × T
  def self.simple_interest(principal:, rate:, years:)
    p = BigDecimal(principal.to_s)
    r = BigDecimal(rate.to_s)
    t = BigDecimal(years.to_s)
    interest = p * r * t
    { interest: interest.round(2), total: (p + interest).round(2) }
  end

  # Compound Interest: A = P(1 + r/n)^(nt)
  def self.compound_interest(principal:, rate:, years:, compounds_per_year: 12)
    p = BigDecimal(principal.to_s)
    r = BigDecimal(rate.to_s)
    n = BigDecimal(compounds_per_year.to_s)
    t = BigDecimal(years.to_s)

    amount = p * (1 + r / n) ** (n * t)
    interest = amount - p

    { interest: interest.round(2), total: amount.round(2) }
  end

  # Monthly Mortgage Payment: M = P[r(1+r)^n]/[(1+r)^n-1]
  def self.mortgage_payment(principal:, annual_rate:, years:)
    p = BigDecimal(principal.to_s)
    annual_r = BigDecimal(annual_rate.to_s)
    monthly_r = annual_r / 12
    n = years * 12  # total months

    payment = p * monthly_r * (1 + monthly_r)**n / ((1 + monthly_r)**n - 1)
    total_paid = payment * n
    total_interest = total_paid - p

    {
      monthly_payment: payment.round(2),
      total_paid: total_paid.round(2),
      total_interest: total_interest.round(2)
    }
  end
end

puts "=== Simple Interest ==="
result = FinancialCalculator.simple_interest(principal: 100_000, rate: 0.05, years: 3)
puts "เงินต้น: ฿100,000 | อัตรา: 5% | ระยะเวลา: 3 ปี"
puts "ดอกเบี้ย: ฿#{result[:interest]}"
puts "เงินรวม: ฿#{result[:total]}"

puts "\n=== Compound Interest ==="
result2 = FinancialCalculator.compound_interest(principal: 100_000, rate: 0.05, years: 10)
puts "เงินต้น: ฿100,000 | อัตรา: 5% | 10 ปี | ทบรายเดือน"
puts "ดอกเบี้ยรวม: ฿#{result2[:interest]}"
puts "เงินรวม: ฿#{result2[:total]}"

puts "\n=== Mortgage ==="
mortgage = FinancialCalculator.mortgage_payment(principal: 2_000_000, annual_rate: 0.035, years: 30)
puts "บ้านราคา: ฿2,000,000 | อัตราดอกเบี้ย: 3.5% ต่อปี | 30 ปี"
puts "ผ่อนรายเดือน: ฿#{mortgage[:monthly_payment]}"
puts "จ่ายรวมตลอด: ฿#{mortgage[:total_paid]}"
puts "ดอกเบี้ยรวม: ฿#{mortgage[:total_interest]}"
```

### แบบฝึกหัดที่ 3: Statistics Calculator

```ruby
module Statistics
  def self.mean(data)
    data.sum.to_f / data.length
  end

  def self.median(data)
    sorted = data.sort
    mid = sorted.length / 2
    sorted.length.odd? ? sorted[mid] : (sorted[mid-1] + sorted[mid]) / 2.0
  end

  def self.mode(data)
    freq = data.each_with_object(Hash.new(0)) { |n, h| h[n] += 1 }
    max_freq = freq.values.max
    freq.select { |_, count| count == max_freq }.keys
  end

  def self.variance(data)
    m = mean(data)
    data.sum { |x| (x - m)**2 } / data.length.to_f
  end

  def self.std_deviation(data)
    Math.sqrt(variance(data))
  end

  def self.summary(data)
    sorted = data.sort
    {
      count:     data.length,
      min:       sorted.first,
      max:       sorted.last,
      range:     sorted.last - sorted.first,
      mean:      mean(data).round(4),
      median:    median(data),
      mode:      mode(data),
      variance:  variance(data).round(4),
      std_dev:   std_deviation(data).round(4),
      q1:        sorted[sorted.length / 4],
      q3:        sorted[3 * sorted.length / 4]
    }
  end
end

scores = [85, 92, 78, 95, 88, 76, 92, 84, 89, 91, 73, 87, 92, 95, 80]

puts "=== Statistics Summary ==="
stats = Statistics.summary(scores)
stats.each do |key, value|
  puts "#{key.to_s.ljust(12)}: #{value}"
end
```

### แบบฝึกหัดที่ 4: Number Pattern Generator

```ruby
# Geometric sequence
def geometric_sequence(first, ratio, count)
  count.times.map { |i| first * (ratio ** i) }
end

# Arithmetic sequence
def arithmetic_sequence(first, diff, count)
  count.times.map { |i| first + i * diff }
end

# Triangular numbers: 1, 3, 6, 10, 15, ...
def triangular_numbers(count)
  (1..count).map { |n| n * (n + 1) / 2 }
end

# Perfect squares
def perfect_squares(count)
  (1..count).map { |n| n**2 }
end

# Perfect cubes
def perfect_cubes(count)
  (1..count).map { |n| n**3 }
end

puts "Geometric (2, ratio=3): #{geometric_sequence(2, 3, 8).inspect}"
puts "Arithmetic (1, diff=4): #{arithmetic_sequence(1, 4, 8).inspect}"
puts "Triangular (10):        #{triangular_numbers(10).inspect}"
puts "Squares (10):           #{perfect_squares(10).inspect}"
puts "Cubes (10):             #{perfect_cubes(10).inspect}"

# Pascal's Triangle
def pascals_triangle(rows)
  triangle = [[1]]
  (rows - 1).times do
    prev = triangle.last
    next_row = [1]
    (1..prev.length - 1).each { |i| next_row << prev[i-1] + prev[i] }
    next_row << 1
    triangle << next_row
  end
  triangle
end

puts "\nPascal's Triangle:"
pascals_triangle(8).each_with_index do |row, i|
  spaces = " " * (7 - i) * 2
  puts spaces + row.map { |n| n.to_s.center(4) }.join
end
```

### แบบฝึกหัดที่ 5: Dice Simulator

```ruby
class Dice
  def initialize(sides: 6)
    @sides = sides
  end

  def roll
    rand(1..@sides)
  end

  def roll_multiple(count)
    count.times.map { roll }
  end

  def roll_with_advantage
    [roll, roll].max
  end

  def roll_with_disadvantage
    [roll, roll].min
  end
end

class DiceSimulator
  def self.run_experiment(dice_count: 2, sides: 6, simulations: 100_000)
    dice = Dice.new(sides: sides)
    results = Hash.new(0)

    simulations.times do
      total = dice.roll_multiple(dice_count).sum
      results[total] += 1
    end

    results.sort.each do |sum, count|
      percentage = (count * 100.0 / simulations).round(1)
      bar = "█" * (percentage * 2).to_i
      printf("%3d: %6d (%5.1f%%) %s\n", sum, count, percentage, bar)
    end
  end
end

puts "=== 2d6 Distribution (100,000 rolls) ==="
DiceSimulator.run_experiment(dice_count: 2, sides: 6, simulations: 100_000)
```

### แบบฝึกหัดที่ 6: Prime Number Tools

```ruby
module PrimeTools
  def self.prime?(n)
    return false if n < 2
    return true if n == 2 || n == 3
    return false if n.even? || n % 3 == 0
    i = 5
    while i * i <= n
      return false if n % i == 0 || n % (i + 2) == 0
      i += 6
    end
    true
  end

  def self.primes_upto(n)
    sieve = Array.new(n + 1, true)
    sieve[0] = sieve[1] = false
    (2..Math.sqrt(n).to_i).each do |i|
      if sieve[i]
        (i*i..n).step(i) { |j| sieve[j] = false }
      end
    end
    sieve.each_index.select { |i| sieve[i] }
  end

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

  def self.goldbach(n)
    return nil unless n.even? && n > 2
    primes = primes_upto(n)
    prime_set = primes.to_set
    primes.each do |p|
      return [p, n - p] if prime_set.include?(n - p)
    end
    nil
  end
end

require 'set'

puts "=== Prime Tests ==="
[1, 2, 17, 100, 97, 1_000_003].each do |n|
  puts "#{n.to_s.rjust(10)} - #{PrimeTools.prime?(n) ? 'เฉพาะ' : 'ไม่เฉพาะ'}"
end

puts "\n=== Prime Factorization ==="
[12, 100, 360, 1024, 9999].each do |n|
  factors = PrimeTools.prime_factors(n)
  puts "#{n} = #{factors.join(' × ')}"
end

puts "\n=== Goldbach's Conjecture ==="
[4, 6, 100, 1000].each do |n|
  p1, p2 = PrimeTools.goldbach(n)
  puts "#{n} = #{p1} + #{p2}"
end
```

### แบบฝึกหัดที่ 7: Binary/Hex Converter

```ruby
class NumberBaseConverter
  DIGITS = ("0".."9").to_a + ("A".."Z").to_a

  def self.to_base(n, base, min_length: 1)
    return "0" if n == 0

    result = ""
    while n > 0
      result = DIGITS[n % base] + result
      n /= base
    end

    result.rjust(min_length, "0")
  end

  def self.from_base(str, base)
    str.upcase.chars.reduce(0) do |sum, char|
      digit = DIGITS.index(char)
      raise "Invalid digit #{char} for base #{base}" if digit.nil? || digit >= base
      sum * base + digit
    end
  end

  def self.convert(value, from_base, to_base)
    decimal = from_base == 10 ? value.to_i : from_base(value.to_s, from_base)
    to_base(decimal, to_base)
  end
end

puts "=== Number Base Converter ==="
[0, 1, 42, 255, 1024, 65535].each do |n|
  puts "Decimal: #{n.to_s.rjust(6)}" \
       " | Binary: #{NumberBaseConverter.to_base(n, 2, min_length: 16)}" \
       " | Hex: #{NumberBaseConverter.to_base(n, 16, min_length: 4)}" \
       " | Oct: #{NumberBaseConverter.to_base(n, 8, min_length: 6)}"
end
```

### แบบฝึกหัดที่ 8: Math Quiz Generator

```ruby
class MathQuiz
  OPERATIONS = [:add, :subtract, :multiply, :divide]

  def initialize(difficulty: :easy)
    @difficulty = difficulty
    @score = 0
    @total = 0
  end

  def generate_question
    op = OPERATIONS.sample

    case @difficulty
    when :easy
      a = rand(1..20)
      b = rand(1..20)
    when :medium
      a = rand(1..100)
      b = rand(1..100)
    when :hard
      a = rand(1..1000)
      b = rand(1..1000)
    end

    # Ensure clean division
    if op == :divide
      b = rand(1..10)
      a = b * rand(1..10)
    end

    answer = calculate(a, op, b)
    { a: a, b: b, operation: op, answer: answer }
  end

  def run(questions: 5)
    puts "=== Math Quiz (#{@difficulty}) ==="
    puts "ตอบคำถาม #{questions} ข้อ\n\n"

    questions.times do |i|
      q = generate_question
      op_symbol = { add: "+", subtract: "-", multiply: "×", divide: "÷" }[q[:operation]]

      print "ข้อที่ #{i+1}: #{q[:a]} #{op_symbol} #{q[:b]} = ? "
      user_answer = gets.chomp

      correct = user_answer.to_f == q[:answer].to_f
      @score += 1 if correct
      @total += 1

      puts correct ? "✓ ถูกต้อง!" : "✗ ผิด! คำตอบที่ถูกต้องคือ #{q[:answer]}"
    end

    puts "\n=== ผลการทดสอบ ==="
    puts "คะแนน: #{@score}/#{@total}"
    puts "เปอร์เซ็นต์: #{(@score * 100.0 / @total).round(1)}%"
  end

  private

  def calculate(a, op, b)
    case op
    when :add      then a + b
    when :subtract then a - b
    when :multiply then a * b
    when :divide   then a.to_f / b
    end
  end
end

# quiz = MathQuiz.new(difficulty: :easy)
# quiz.run(questions: 5)
puts "MathQuiz class created (uncommment เพื่อเล่น)"
```

### แบบฝึกหัดที่ 9: Number Series Analyzer

```ruby
def analyze_series(numbers)
  diffs = numbers.each_cons(2).map { |a, b| b - a }
  ratios = numbers.each_cons(2).map { |a, b| b.to_f / a rescue nil }.compact

  # ตรวจสอบว่าเป็น Arithmetic series
  if diffs.uniq.length == 1
    return {
      type: "Arithmetic Sequence",
      first_term: numbers.first,
      common_difference: diffs.first,
      next_term: numbers.last + diffs.first,
      formula: "a(n) = #{numbers.first} + (n-1)×#{diffs.first}"
    }
  end

  # ตรวจสอบว่าเป็น Geometric series
  unique_ratios = ratios.map { |r| r.round(6) }.uniq
  if unique_ratios.length == 1
    ratio = unique_ratios.first
    return {
      type: "Geometric Sequence",
      first_term: numbers.first,
      common_ratio: ratio,
      next_term: (numbers.last * ratio).round(6),
      formula: "a(n) = #{numbers.first} × #{ratio}^(n-1)"
    }
  end

  { type: "Unknown Series", numbers: numbers }
end

series = [
  [2, 5, 8, 11, 14],
  [3, 6, 12, 24, 48],
  [1, 4, 9, 16, 25],
  [100, 50, 25, 12.5]
]

series.each do |s|
  puts "Series: #{s.inspect}"
  result = analyze_series(s)
  result.each { |k, v| puts "  #{k}: #{v}" }
  puts
end
```

### แบบฝึกหัดที่ 10-15: โจทย์เพิ่มเติม

```ruby
# แบบฝึกหัดที่ 10: Loan Calculator
def loan_schedule(principal:, annual_rate:, months:)
  monthly_rate = annual_rate / 12.0
  payment = principal * monthly_rate / (1 - (1 + monthly_rate)**(-months))

  printf("%-6s %12s %12s %12s %12s\n", "เดือน", "ยอดค้าง", "ชำระ", "ดอกเบี้ย", "ลดต้น")
  puts "-" * 60

  balance = principal.to_f
  months.times do |month|
    interest = balance * monthly_rate
    principal_paid = payment - interest
    balance -= principal_paid

    printf("%-6d %12.2f %12.2f %12.2f %12.2f\n",
           month + 1, [balance, 0].max, payment, interest, principal_paid)
    break if balance <= 0.01
  end
end

# loan_schedule(principal: 100_000, annual_rate: 0.06, months: 12)

# แบบฝึกหัดที่ 11: Area Calculator
module AreaCalculator
  def self.circle(radius)    = Math::PI * radius**2
  def self.rectangle(w, h)   = w * h
  def self.triangle(b, h)    = 0.5 * b * h
  def self.trapezoid(a, b, h) = 0.5 * (a + b) * h
  def self.ellipse(a, b)     = Math::PI * a * b
  def self.regular_polygon(sides, side_length)
    (sides * side_length**2) / (4 * Math.tan(Math::PI / sides))
  end
end

shapes = [
  [:circle, [5]],
  [:rectangle, [4, 6]],
  [:triangle, [3, 8]],
  [:trapezoid, [5, 8, 4]],
  [:regular_polygon, [6, 4]]
]

puts "=== Area Calculator ==="
shapes.each do |(shape, args)|
  area = AreaCalculator.send(shape, *args)
  puts "#{shape.to_s.capitalize}(#{args.join(', ')}): #{area.round(4)}"
end

# แบบฝึกหัดที่ 12: Roman Numerals
def to_roman(number)
  values = [[1000,"M"],[900,"CM"],[500,"D"],[400,"CD"],[100,"C"],
            [90,"XC"],[50,"L"],[40,"XL"],[10,"X"],[9,"IX"],
            [5,"V"],[4,"IV"],[1,"I"]]

  result = ""
  values.each do |value, numeral|
    while number >= value
      result += numeral
      number -= value
    end
  end
  result
end

def from_roman(roman)
  values = { 'M'=>1000,'D'=>500,'C'=>100,'L'=>50,'X'=>10,'V'=>5,'I'=>1 }
  prev = 0
  roman.upcase.chars.reverse.reduce(0) do |sum, char|
    val = values[char]
    val < prev ? sum - val : (prev = val; sum + val)
  end
end

[1, 4, 9, 14, 40, 90, 399, 2024, 3999].each do |n|
  roman = to_roman(n)
  back = from_roman(roman)
  puts "#{n.to_s.rjust(5)} = #{roman.ljust(15)} = #{back}"
end

# แบบฝึกหัดที่ 13: Number Properties
def number_properties(n)
  require 'set'

  props = []
  props << "บวก" if n > 0
  props << "ลบ" if n < 0
  props << "ศูนย์" if n == 0

  if n.is_a?(Integer) && n > 0
    props << "คู่" if n.even?
    props << "คี่" if n.odd?

    # Perfect square
    sqrt = Math.sqrt(n).to_i
    props << "กำลังสองสมบูรณ์" if sqrt * sqrt == n

    # Perfect cube
    cbrt = Math.cbrt(n).round.to_i
    props << "กำลังสามสมบูรณ์" if cbrt**3 == n

    # Palindrome number
    props << "Palindrome" if n.to_s == n.to_s.reverse

    # Armstrong/Narcissistic number
    digits = n.to_s.chars.map(&:to_i)
    props << "Armstrong" if digits.sum { |d| d**digits.length } == n
  end

  props
end

[0, 1, 4, 9, 8, 153, 371, 121, 125].each do |n|
  puts "#{n}: #{number_properties(n).join(', ')}"
end

# แบบฝึกหัดที่ 14: Percentage Calculator
class PercentageCalculator
  def self.percent_of(percent, total)
    total * percent / 100.0
  end

  def self.what_percent(part, total)
    part * 100.0 / total
  end

  def self.percent_change(from, to)
    (to - from) * 100.0 / from
  end

  def self.add_percent(value, percent)
    value * (1 + percent / 100.0)
  end

  def self.remove_percent(value, percent)
    value * (1 - percent / 100.0)
  end

  def self.tax_inclusive(amount, tax_rate)
    amount / (1 + tax_rate / 100.0)
  end
end

price = 1000.0

puts "=== Percentage Calculator ==="
puts "15% of #{price} = #{PercentageCalculator.percent_of(15, price)}"
puts "150 is what % of #{price} = #{PercentageCalculator.what_percent(150, price)}%"
puts "#{price} → 1200 = #{PercentageCalculator.percent_change(price, 1200).round(1)}% change"
puts "#{price} + 7% VAT = #{PercentageCalculator.add_percent(price, 7)}"
puts "VAT inclusive 1070 = #{PercentageCalculator.tax_inclusive(1070, 7).round(2)} before tax"

# แบบฝึกหัดที่ 15: Number Spiral
def number_spiral(n)
  size = n * 2 - 1
  grid = Array.new(size) { Array.new(size, 0) }

  num = 1
  top, bottom, left, right = 0, size - 1, 0, size - 1

  while num <= size * size
    (left..right).each { |i| grid[top][i] = num; num += 1 }
    top += 1
    (top..bottom).each { |i| grid[i][right] = num; num += 1 }
    right -= 1
    (right..left).reverse_each { |i| grid[bottom][i] = num; num += 1 }  if top <= bottom
    bottom -= 1
    (bottom..top).reverse_each { |i| grid[i][left] = num; num += 1 } if left <= right
    left += 1
  end

  grid.each { |row| puts row.map { |n| n.to_s.rjust(4) }.join }
end

puts "=== 3x3 Number Spiral ==="
number_spiral(3)
```

---

## สรุปส่วนที่ 4

ในส่วนนี้เราได้เรียนรู้เรื่องตัวเลขใน Ruby:

1. **Integer vs Float** - ความแตกต่างและการใช้งาน
2. **Arithmetic Operations** - บวก ลบ คูณ หาร ยกกำลัง
3. **Math Module** - sin, cos, sqrt, log, PI, E
4. **Number Formatting** - printf, sprintf, commas, currency
5. **Integer Methods** - even?, odd?, times, upto, downto, gcd, lcm
6. **Float Precision** - IEEE 754, precision issues, rounding
7. **BigDecimal** - การเงินที่แม่นยำ
8. **Number Bases** - binary, octal, hex, conversion
9. **Conversions** - to_i, to_f, to_r, to_c, Rational, Complex
10. **Random Numbers** - rand, Random, SecureRandom, Monte Carlo
11. **Constants** - Math::PI, Float::INFINITY, Float::NAN

---

*เอกสารนี้เป็นส่วนหนึ่งของคอร์ส Ruby on Rails สำหรับผู้เริ่มต้น*
