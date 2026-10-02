# ตอนที่ 3: Strings

## Ruby Programming Course สำหรับผู้เริ่มต้นภาษาไทย

---

## บทนำ

String คือลำดับของตัวอักษร (Sequence of Characters) และเป็นหนึ่งในประเภทข้อมูลที่ใช้งานบ่อยที่สุดในการเขียนโปรแกรม Ruby มี String Methods ที่ครบครันและทรงพลังมาก

**สิ่งที่จะได้เรียนรู้:**
- การสร้าง String แบบต่างๆ
- String Interpolation
- String Methods ทุก Method ที่สำคัญ
- String Formatting
- Multiline Strings และ Heredoc
- Frozen String Literals
- การค้นหา เปรียบเทียบ และแทนที่ใน String
- การแยกและรวม String
- String Encoding
- String as Array

---

## Step 26: การสร้าง String

### 26.1 Single vs Double Quotes

```ruby
# Single Quotes - ตีความแค่ \\ และ \'
single1 = 'Hello, World!'
single2 = 'It\'s a beautiful day'
single3 = 'เส้นทาง: C:\\Users\\User'

puts single1  # => Hello, World!
puts single2  # => It's a beautiful day
puts single3  # => เส้นทาง: C:\Users\User

# Double Quotes - ตีความ Escape Sequences ทั้งหมดและ Interpolation
double1 = "Hello, World!"
double2 = "บรรทัดที่ 1\nบรรทัดที่ 2"
double3 = "Tab:\there"
double4 = "คำพูด: \"Ruby is fun!\""

puts double1  # => Hello, World!
puts double2  # => บรรทัดที่ 1 / บรรทัดที่ 2
puts double3  # => Tab:    here
puts double4  # => คำพูด: "Ruby is fun!"
```

```ruby
# Escape Sequences ใน Double Quotes
puts "\n"   # Newline
puts "\t"   # Tab
puts "\r"   # Carriage Return
puts "\\"   # Backslash
puts "\""   # Double Quote
puts "\'"   # Single Quote
puts "\a"   # Bell
puts "\b"   # Backspace
puts "\f"   # Form Feed
puts "\v"   # Vertical Tab
puts "\e"   # Escape (ASCII 27)
puts "\s"   # Space

# Unicode Escape
puts "\u0E2A"   # => ส (Thai character)
puts "\u0E27\u0E31\u0E14\u0E35"  # => วัดี

# Hex Escape
puts "\x41"     # => A (ASCII 65 in hex)
puts "\x4F"     # => O
```

### 26.2 String Literals

```ruby
# %q{} - เหมือน Single Quotes
str1 = %q{Hello 'World' and "friends"}
puts str1  # => Hello 'World' and "friends"

# %Q{} หรือ %{} - เหมือน Double Quotes
name = "สมชาย"
str2 = %Q{สวัสดี #{name}!}
str3 = %{สวัสดี #{name}!}
puts str2  # => สวัสดี สมชาย!
puts str3  # => สวัสดี สมชาย!

# สามารถใช้ delimiter อื่นได้
str4 = %q[ใช้ square brackets]
str5 = %q(ใช้ parentheses)
str6 = %q<ใช้ angle brackets>
str7 = %q!ใช้ exclamation!

puts str4  # => ใช้ square brackets
puts str5  # => ใช้ parentheses
```

---

## Step 27: String Interpolation

### 27.1 การใช้ #{}

```ruby
name = "Ruby"
version = 3.2
year = 2023

puts "ภาษา #{name} เวอร์ชัน #{version} ปี #{year}"
# => ภาษา Ruby เวอร์ชัน 3.2 ปี 2023

# ใส่ Expression ได้
price = 100
tax_rate = 0.07
puts "ราคา #{price} บาท ภาษี #{price * tax_rate} บาท รวม #{price + price * tax_rate} บาท"
# => ราคา 100 บาท ภาษี 7.0 บาท รวม 107.0 บาท

# ใส่ Method Call ได้
items = ["แอปเปิ้ล", "กล้วย", "มะม่วง"]
puts "มีสินค้า #{items.length} รายการ: #{items.join(', ')}"
# => มีสินค้า 3 รายการ: แอปเปิ้ล, กล้วย, มะม่วง
```

```ruby
# Interpolation เรียก to_s อัตโนมัติ
class Product
  def initialize(name, price)
    @name = name
    @price = price
  end
  
  def to_s
    "#{@name} (#{@price} บาท)"
  end
end

product = Product.new("กาแฟ", 65)
puts "สั่ง: #{product}"  # => สั่ง: กาแฟ (65 บาท)

# Multi-line Interpolation
result = "สรุป:\n" \
         "  จำนวน: #{items.length}\n" \
         "  รายการ: #{items.join(', ')}"
puts result
```

```ruby
# String Concatenation vs Interpolation
first_name = "สม"
last_name = "ชาย"

# Concatenation (+) - สร้าง String ใหม่
full_name_concat = first_name + " " + last_name
puts full_name_concat  # => สมชาย

# Interpolation - เร็วกว่าและอ่านง่ายกว่า
full_name_interp = "#{first_name} #{last_name}"
puts full_name_interp  # => สมชาย

# Append (<<) - แก้ไข String ต้นฉบับ (เร็วกว่า +)
result = "Hello"
result << ", " << "World" << "!"
puts result  # => Hello, World!
```

---

## Step 28: String Methods - Case และ Length

### 28.1 Case Methods

```ruby
str = "hello, ruby world!"

puts str.upcase      # => HELLO, RUBY WORLD!
puts str.downcase    # => hello, ruby world!
puts str.capitalize  # => Hello, ruby world! (ตัวแรกพิมพ์ใหญ่)
puts str.swapcase    # => HELLO, RUBY WORLD! (สลับ case)

# Thai doesn't have case, but works with ASCII parts
mixed = "Hello สวัสดี WORLD"
puts mixed.downcase  # => hello สวัสดี world
puts mixed.upcase    # => HELLO สวัสดี WORLD

# Bang versions (modify in place)
str2 = "hello world"
str2.upcase!
puts str2  # => HELLO WORLD
```

### 28.2 Length Methods

```ruby
str = "Hello, สวัสดี!"

puts str.length    # => 14 (จำนวนตัวอักษร)
puts str.size      # => 14 (เหมือน length)
puts str.bytesize  # => 28 (จำนวน bytes ใน UTF-8)
puts str.empty?    # => false
puts "".empty?     # => true
puts "   ".empty?  # => false (มีช่องว่าง)

# เช็คว่า String ว่างหรือ whitespace เท่านั้น
puts "   ".strip.empty?  # => true
```

---

## Step 29: String Methods - Stripping และ Padding

### 29.1 Strip Methods

```ruby
str = "  \t Hello, World! \n  "

puts str.strip.inspect   # => "Hello, World!"
puts str.lstrip.inspect  # => "Hello, World! \n  "
puts str.rstrip.inspect  # => "  \t Hello, World!"

# chomp - ลบ newline ท้าย
line = "Hello\n"
puts line.chomp.inspect  # => "Hello"

line2 = "Hello\r\n"
puts line2.chomp.inspect  # => "Hello"

# chop - ลบตัวสุดท้ายเสมอ
puts "Hello!".chop   # => "Hello"
puts "Hello\n".chop  # => "Hello"

# delete
puts "Hello, World!".delete("lo")   # => "He, Wrd!"
puts "Hello, World!".delete("a-e")  # => "Hllo, Worl!"  (ลบ range a-e)
```

### 29.2 Padding Methods

```ruby
str = "Hello"

# ljust - ชิดซ้าย (padding ขวา)
puts str.ljust(10)        # => "Hello     "
puts str.ljust(10, '-')   # => "Hello-----"

# rjust - ชิดขวา (padding ซ้าย)
puts str.rjust(10)        # => "     Hello"
puts str.rjust(10, '0')   # => "00000Hello"

# center - กึ่งกลาง
puts str.center(11)       # => "   Hello   "
puts str.center(11, '*')  # => "***Hello***"

# ใช้จัดรูปแบบตาราง
items = [["สินค้า", "ราคา", "จำนวน"],
         ["แอปเปิ้ล", "50", "100"],
         ["กล้วย", "20", "200"],
         ["มะม่วง", "80", "50"]]

items.each do |row|
  puts "#{row[0].ljust(12)} #{row[1].rjust(8)} #{row[2].rjust(8)}"
end
```

---

## Step 30: String Methods - squeeze และ tr

### 30.1 squeeze

```ruby
# squeeze - ลดตัวอักษรซ้ำๆ ที่ติดกัน
puts "aaabbbccc".squeeze       # => "abc"
puts "hello    world".squeeze  # => "hello world"
puts "aabbccdd".squeeze("a-c") # => "abccdd" (squeeze เฉพาะ a-c)

# ใช้ในการ normalize
def normalize_spaces(text)
  text.strip.squeeze(" ")
end

puts normalize_spaces("  hello    world  ")  # => "hello world"
```

### 30.2 tr (Transliterate)

```ruby
# tr - แทนที่ตัวอักษร (คล้าย tr command ใน Unix)
puts "hello".tr('el', 'ip')      # => "hippo"
puts "hello".tr('aeiou', '*')    # => "h*ll*"
puts "Hello, World".tr('a-y', 'b-z')  # => "Ifmmp, Xpsme"

# tr_s - tr แล้ว squeeze
puts "hello".tr_s('l', 'r')   # => "hero"

# นำไปใช้จริง - ROT13 cipher
puts "Hello World".tr('A-Za-z', 'N-ZA-Mn-za-m')  # => "Uryyb Jbeyq"
# Decode กลับ
puts "Uryyb Jbeyq".tr('A-Za-z', 'N-ZA-Mn-za-m')  # => "Hello World"

# แปลง Thai transliteration (ตัวอย่าง)
thai_vowels = "กขคงจฉชซ"
substitutes  = "ABCDEFGH"
puts thai_vowels.tr("กขคง", "ABCD")  # => ABCDจฉชซ
```

---

## Step 31: String Methods - Count และ Checking

### 31.1 count

```ruby
str = "Hello, World!"

puts str.count("l")       # => 3 (นับ 'l')
puts str.count("a-e")     # => 1 (นับ range a-e)
puts str.count("aeiou")   # => 3 (นับ vowels)
puts str.count("^aeiou")  # => 10 (นับ non-vowels)

# นับจำนวนคำในประโยค
def count_words(text)
  text.split.length
end

puts count_words("Ruby is a beautiful programming language")  # => 6
```

### 31.2 Checking Methods

```ruby
str = "Hello, World!"

puts str.include?("World")    # => true
puts str.include?("world")    # => false (case sensitive)
puts str.start_with?("Hello") # => true
puts str.start_with?("World") # => false
puts str.end_with?("!")       # => true
puts str.end_with?("World")   # => false

# Multiple arguments
puts str.start_with?("Hello", "Hi", "Hey")  # => true (ตรวจทุก arg)
puts str.end_with?("?", "!", ".")            # => true

# match? - ตรวจ regex
puts "hello123".match?(/\d+/)  # => true
puts "hello".match?(/\d+/)     # => false
```

---

## Step 32: String Searching

### 32.1 index และ rindex

```ruby
str = "Hello, World! Hello, Ruby!"

puts str.index("Hello")     # => 0 (ตำแหน่งแรก)
puts str.rindex("Hello")    # => 14 (ตำแหน่งสุดท้าย)
puts str.index("Hello", 1)  # => 14 (เริ่มค้นจาก position 1)
puts str.index("xyz")       # => nil (ไม่พบ)

# index ด้วย Regex
str2 = "phone: 02-123-4567, mobile: 081-234-5678"
puts str2.index(/\d{2,3}-\d{3}-\d{4}/)  # => 7

# scan - หาทุก occurrence
puts str.scan("Hello").inspect  # => ["Hello", "Hello"]
phones = str2.scan(/\d{2,3}-\d{3}-\d{4}/)
puts phones.inspect  # => ["02-123-4567", "081-234-5678"]
```

---

## Step 33: String Replacing

### 33.1 sub และ gsub

```ruby
str = "Hello, World! Hello, Ruby!"

# sub - แทนที่ครั้งแรกที่พบ
puts str.sub("Hello", "Hi")    # => "Hi, World! Hello, Ruby!"

# gsub - แทนที่ทุก occurrence
puts str.gsub("Hello", "Hi")   # => "Hi, World! Hi, Ruby!"

# ใช้ Regex
puts str.gsub(/Hello/, "Hi")   # => "Hi, World! Hi, Ruby!"

# ใช้ Block ใน gsub
result = "hello world ruby".gsub(/\b\w/) { |match| match.upcase }
puts result  # => "Hello World Ruby"

# gsub ด้วย Hash
codes = { "US" => "สหรัฐ", "TH" => "ไทย", "JP" => "ญี่ปุ่น" }
text = "US ส่งสินค้าไป TH และ JP"
puts text.gsub(/US|TH|JP/, codes)  # => "สหรัฐ ส่งสินค้าไป ไทย และ ญี่ปุ่น"
```

```ruby
# sub! และ gsub! - แก้ไข in-place
str = "Hello, World!"
str.gsub!(/[aeiou]/, "*")
puts str  # => "H*ll*, W*rld!"

# ใช้ Capture Groups ใน gsub
phone = "0812345678"
formatted = phone.gsub(/(\d{3})(\d{3})(\d{4})/, '\1-\2-\3')
puts formatted  # => 081-234-5678

# เปลี่ยน Word Case
"hello_world_ruby".gsub(/_(\w)/) { $1.upcase }  # => "helloWorldRuby"
```

---

## Step 34: String Splitting

### 34.1 split

```ruby
# split ด้วย delimiter
csv = "แอปเปิ้ล,กล้วย,มะม่วง,ส้ม"
fruits = csv.split(",")
puts fruits.inspect  # => ["แอปเปิ้ล", "กล้วย", "มะม่วง", "ส้ม"]

# split ด้วย whitespace (default)
sentence = "Hello   Ruby   World"
words = sentence.split
puts words.inspect  # => ["Hello", "Ruby", "World"]

# split ด้วย Regex
data = "one  two   three    four"
parts = data.split(/\s+/)
puts parts.inspect  # => ["one", "two", "three", "four"]

# กำหนดจำนวนสูงสุด
puts "a,b,c,d,e".split(",", 3).inspect  # => ["a", "b", "c,d,e"]

# split แต่ละตัวอักษร
puts "hello".split("").inspect   # => ["h", "e", "l", "l", "o"]
puts "hello".chars.inspect       # => ["h", "e", "l", "l", "o"] (เหมือนกัน)
```

```ruby
# bytes - แยกเป็น bytes
str = "Hi!"
puts str.bytes.inspect   # => [72, 105, 33]

# chars - แยกเป็นตัวอักษร (Unicode aware)
thai = "สวัสดี"
puts thai.chars.inspect  # => ["ส", "ว", "ั", "ส", "ด", "ี"]
puts thai.length         # => 6

# lines - แยกเป็นบรรทัด
multiline = "บรรทัด 1\nบรรทัด 2\nบรรทัด 3"
puts multiline.lines.inspect
# => ["บรรทัด 1\n", "บรรทัด 2\n", "บรรทัด 3"]
```

---

## Step 35: String Joining

### 35.1 Array Join

```ruby
words = ["Ruby", "is", "awesome"]
puts words.join(" ")      # => "Ruby is awesome"
puts words.join(", ")     # => "Ruby, is, awesome"
puts words.join(" and ")  # => "Ruby and is and awesome"
puts words.join          # => "Rubyisawesome" (no separator)

# สร้าง CSV
headers = ["ชื่อ", "อายุ", "เมือง"]
data = ["สมชาย", "25", "กรุงเทพ"]
puts headers.join(",")  # => ชื่อ,อายุ,เมือง
puts data.join(",")     # => สมชาย,25,กรุงเทพ
```

---

## Step 36: String Formatting

### 36.1 % Operator

```ruby
# % - String formatting (คล้าย printf ใน C)
printf_format = "ชื่อ: %s, อายุ: %d, GPA: %.2f"
puts printf_format % ["สมชาย", 25, 3.85]
# => ชื่อ: สมชาย, อายุ: 25, GPA: 3.85

# Format Specifiers
puts "%d" % 42          # => 42 (decimal)
puts "%f" % 3.14        # => 3.140000
puts "%.2f" % 3.14      # => 3.14
puts "%e" % 1234567     # => 1.234567e+06
puts "%s" % "hello"     # => hello
puts "%10s" % "hello"   # => "     hello" (right-aligned, width 10)
puts "%-10s" % "hello"  # => "hello     " (left-aligned)
puts "%010d" % 42       # => 0000000042 (zero-padded)
puts "%+d" % 42         # => +42
puts "%x" % 255         # => ff (hexadecimal)
puts "%X" % 255         # => FF
puts "%o" % 8           # => 10 (octal)
puts "%b" % 10          # => 1010 (binary)
```

```ruby
# sprintf / format - เหมือน % แต่อ่านง่ายกว่า
name = "สมหญิง"
age = 22
score = 95.5

result = sprintf("ชื่อ: %-10s อายุ: %3d คะแนน: %5.1f", name, age, score)
puts result  # => ชื่อ: สมหญิง      อายุ:  22 คะแนน:  95.5

# format เหมือน sprintf
result2 = format("ราคา: %,.2f บาท", 1234567.89)
puts result2  # => ราคา: 1234567.89 บาท (ไม่มี comma ใน Ruby โดย default)

# Named format
puts "%{name} มีอายุ %{age} ปี" % { name: "สมชาย", age: 25 }
# => สมชาย มีอายุ 25 ปี
```

---

## Step 37: Multiline Strings (Heredoc)

### 37.1 Heredoc

```ruby
# <<IDENTIFIER - ไม่ strip indent
text1 = <<HEREDOC
บรรทัดที่ 1
บรรทัดที่ 2
บรรทัดที่ 3
HEREDOC

puts text1
# => บรรทัดที่ 1
# => บรรทัดที่ 2
# => บรรทัดที่ 3

# <<~IDENTIFIER - strip indent (Ruby 2.3+)
text2 = <<~HEREDOC
  บรรทัดที่ 1
  บรรทัดที่ 2
  บรรทัดที่ 3
HEREDOC

puts text2  # เหมือนกัน แต่ strip indent ออก

# <<~'HEREDOC' - no interpolation
name = "Ruby"
text3 = <<~'HEREDOC'
  Hello #{name}!
  ไม่มี interpolation
HEREDOC
puts text3  # => Hello #{name}!
```

```ruby
# Heredoc ใน Method Call
sql = <<~SQL
  SELECT users.name, orders.total
  FROM users
  JOIN orders ON users.id = orders.user_id
  WHERE orders.total > 1000
  ORDER BY orders.total DESC
SQL

puts sql

# HTML Template
name = "สมชาย"
items = ["แอปเปิ้ล", "กล้วย"]

html = <<~HTML
  <div class="user-card">
    <h1>#{name}</h1>
    <ul>
      #{items.map { |item| "<li>#{item}</li>" }.join("\n      ")}
    </ul>
  </div>
HTML

puts html
```

```ruby
# Heredoc ใน Array
messages = [
  <<~MSG1,
    ข้อความที่ 1
    บรรทัด 2 ของข้อความที่ 1
  MSG1
  <<~MSG2
    ข้อความที่ 2
    บรรทัด 2 ของข้อความที่ 2
  MSG2
]

messages.each.with_index(1) do |msg, i|
  puts "Message #{i}:"
  puts msg
end
```

---

## Step 38: Frozen String Literal

### 38.1 # frozen_string_literal: true

```ruby
# เพิ่มที่บรรทัดแรกของไฟล์เพื่อทำให้ทุก String literal เป็น frozen
# frozen_string_literal: true

str = "Hello"
puts str.frozen?  # => true (ถ้า frozen_string_literal: true)

# ข้อดี:
# 1. ประหยัด memory (ไม่สร้าง String object ซ้ำ)
# 2. Thread safe
# 3. Performance ดีขึ้น

# ถ้าต้องการ mutable string ให้ใช้ + "" หรือ String.new
mutable = +"Hello"         # unary + ทำให้เป็น mutable ใน Ruby 2.3+
mutable2 = "Hello".dup    # dup ทำให้ได้ mutable copy
mutable3 = String.new("Hello")

mutable << " World"
puts mutable  # => Hello World
```

```ruby
# ประโยชน์ของ Frozen String
# String เดียวกันใช้ memory เดียวกัน
str1 = "hello"
str2 = "hello"

# ใน frozen_string_literal: true mode
# str1.object_id == str2.object_id => true!

# Performance comparison
require 'benchmark'

iterations = 1_000_000

Benchmark.bm do |x|
  x.report("mutable:") {
    iterations.times { s = "hello"; s << " world" }
  }
  x.report("frozen:") {
    iterations.times { s = "hello"; t = s + " world" }
  }
end
```

---

## Step 39: String Encoding

### 39.1 Encoding Methods

```ruby
# ดู encoding ของ String
str_ascii = "Hello"
str_thai = "สวัสดี"

puts str_ascii.encoding  # => UTF-8
puts str_thai.encoding   # => UTF-8

# ตรวจสอบ valid encoding
puts str_ascii.valid_encoding?  # => true
puts str_thai.valid_encoding?   # => true

# แปลง encoding
str_utf8 = "สวัสดี"
# str_iso = str_utf8.encode("ISO-8859-1")  # => จะ raise error เพราะไม่รองรับ Thai

# encode ด้วย options
# str_safe = str_utf8.encode("ISO-8859-1", invalid: :replace, undef: :replace, replace: "?")

# bytesize vs length
puts str_thai.length    # => 6 (6 ตัวอักษร)
puts str_thai.bytesize  # => 18 (6 * 3 bytes per Thai char ใน UTF-8)
```

```ruby
# Encoding Aware Operations
mixed = "Hello สวัสดี World"

# length นับ characters (ไม่ใช่ bytes)
puts mixed.length    # => 18

# bytes - ดู bytes
puts "A".bytes.inspect     # => [65]
puts "ส".bytes.inspect     # => [224, 185, 170] (3 bytes ใน UTF-8)

# encode สำหรับ external systems
str = "Hello, World!"
puts str.encode("ASCII")   # => Hello, World!
puts str.b.encoding        # => ASCII-8BIT (binary encoding)
```

---

## Step 40: String as Array

### 40.1 การเข้าถึง String ด้วย Index

```ruby
str = "Hello, Ruby!"

# ใช้ [] operator
puts str[0]      # => H
puts str[-1]     # => !
puts str[7, 4]   # => Ruby (start, length)
puts str[7..10]  # => Ruby (range)
puts str[7..]    # => Ruby! (endless range)
puts str[..4]    # => Hello

# first method
puts str.slice(0, 5)  # => Hello (เหมือน str[0, 5])

# อ่านไม่พบ
puts str[100].inspect   # => nil

# each_char - วนลูปแต่ละตัวอักษร
str.each_char do |char|
  print "#{char} "
end
puts
```

```ruby
# String เหมือน Array of Characters
str = "Ruby"

# แก้ไขด้วย []
str[0] = "B"
puts str  # => Buby

str[1..2] = "ir"
puts str  # => Bird

# insert
str.insert(2, "th")
puts str  # => Birthd

# slice! - ตัดและคืนค่า
removed = str.slice!(0, 5)
puts removed  # => Birth
puts str      # => d
```

---

## Step 41: String Comparison

### 41.1 การเปรียบเทียบ String

```ruby
# == และ != - ตรวจค่า
puts "hello" == "hello"   # => true
puts "hello" == "Hello"   # => false (case sensitive)
puts "hello" != "world"   # => true

# eql? - เหมือน == สำหรับ String
puts "hello".eql?("hello")  # => true

# equal? - ตรวจ object identity (object_id)
str1 = "hello"
str2 = "hello"
puts str1.equal?(str2)    # => false (คนละ object)
puts str1.equal?(str1)    # => true

# <=> - Spaceship operator (เปรียบเทียบ lexicographically)
puts "apple" <=> "banana"  # => -1 (apple น้อยกว่า)
puts "banana" <=> "apple"  # => 1
puts "apple" <=> "apple"   # => 0

# เรียงตัวอักษร
words = %w[cherry apple banana date elderberry]
puts words.sort.inspect  # => ["apple", "banana", "cherry", "date", "elderberry"]

# case-insensitive comparison
puts "Hello".casecmp("hello")   # => 0 (เท่ากัน)
puts "Hello".casecmp?("hello")  # => true (Ruby 2.4+)
```

---

## Step 42: String Methods รวม

### 42.1 Methods ที่ใช้บ่อย

```ruby
str = "The quick brown fox jumps over the lazy dog"

# scan - หาทุก match
vowels = str.scan(/[aeiou]/)
puts "สระ: #{vowels.uniq.sort.inspect}"  # => ["a", "e", "i", "o", "u"]

# gsub ด้วย Block
result = str.gsub(/\b\w+\b/) do |word|
  word.length > 3 ? word.upcase : word
end
puts result
# => "THE QUICK BROWN FOX JUMPS OVER THE LAZY DOG"

# squeeze + strip + downcase สำหรับ normalize
user_input = "  Hello   WORLD  "
normalized = user_input.strip.squeeze(" ").downcase
puts normalized  # => "hello world"
```

```ruby
# String Methods ที่น่าสนใจอื่นๆ
str = "Hello, World!"

# succ / next - ตัวอักษรถัดไป
puts "a".succ    # => "b"
puts "z".succ    # => "aa"
puts "Az".succ   # => "Ba"
puts "zz".succ   # => "aaa"
puts "9".succ    # => "10"

# upto - วนซ้ำ
"a".upto("e") { |c| print "#{c} " }  # => a b c d e

# oct - แปลง Octal String เป็น Integer
puts "0177".oct   # => 127
puts "0xFF".hex   # => 255

# unpack - แกะ binary data
puts "ABC".unpack("C*").inspect  # => [65, 66, 67] (ASCII codes)
puts [65, 66, 67].pack("C*")    # => "ABC"
```

```ruby
# String Multiplication
puts "Ha" * 3      # => "HaHaHa"
puts "-" * 40      # => "----------------------------------------"
puts "=" * 40      # => "========================================"

# เปรียบเทียบ แสดงหัวตาราง
def divider(char = "-", width = 40)
  char * width
end

puts divider("=")
puts "| รายการ | ราคา | จำนวน |"
puts divider("=")
puts "| แอปเปิ้ล | 50 | 100 |"
puts divider("-")
```

---

## Step 43: String Utility Methods

### 43.1 Methods เพิ่มเติม

```ruby
str = "Hello, World!"

# replace - แทนที่ content ทั้งหมด (mutate in place)
str2 = "Old Content"
str2.replace("New Content")
puts str2  # => New Content

# clear - เคลียร์ content
str3 = "Hello"
str3.clear
puts str3.empty?  # => true

# encode
puts "Hello".encode("UTF-8")  # => Hello

# match - ตรวจ Regex และคืน MatchData
m = "John 25".match(/(\w+) (\d+)/)
if m
  puts m[0]   # => "John 25" (full match)
  puts m[1]   # => "John" (first capture)
  puts m[2]   # => "25" (second capture)
end

# Named Captures
m2 = "John 25".match(/(?<name>\w+) (?<age>\d+)/)
if m2
  puts m2[:name]  # => John
  puts m2[:age]   # => 25
end
```

```ruby
# String Predicates
puts "hello".frozen?     # => false (ถ้าไม่ได้ freeze)
puts "".empty?           # => true
puts "hello".include?("ell")  # => true

# ASCII?
puts "hello".ascii_only?  # => true
puts "สวัสดี".ascii_only?  # => false

# each_line
"line1\nline2\nline3".each_line do |line|
  puts line.chomp
end

# String ใน Conditional
puts "มีค่า" if "hello"  # String เป็น truthy เสมอ
puts "ว่าง" if !""       # String ว่างก็ยัง truthy!
puts "nil" if !nil        # nil เป็น falsy
```

---

## Step 44: Advanced String Operations

### 44.1 Regular Expressions

```ruby
# match? - เร็วกว่า match เพราะไม่สร้าง MatchData
puts "hello123".match?(/\d+/)  # => true

# scan with groups
str = "First: John 25, Second: Jane 22"
matches = str.scan(/(\w+): (\w+) (\d+)/)
matches.each do |_, name, age|
  puts "ชื่อ: #{name}, อายุ: #{age}"
end

# gsub with regex and capture groups
phone = "โทร: 081-234-5678 หรือ 02-123-4567"
formatted = phone.gsub(/(\d+)-(\d+)-(\d+)/) do |match|
  "(#{$1}) #{$2}-#{$3}"
end
puts formatted
# => โทร: (081) 234-5678 หรือ (02) 123-4567
```

```ruby
# String Template Engine (ตัวอย่าง)
class Template
  def initialize(template)
    @template = template
  end
  
  def render(variables = {})
    @template.gsub(/\{\{(\w+)\}\}/) do |match|
      key = $1.to_sym
      variables[key] || match
    end
  end
end

template = Template.new(<<~HTML)
  <h1>สวัสดี {{name}}!</h1>
  <p>คุณมีอายุ {{age}} ปี</p>
  <p>สมาชิกตั้งแต่: {{year}}</p>
HTML

puts template.render(name: "สมชาย", age: 25, year: 2020)
```

---

## Step 45: แบบฝึกหัด

### แบบฝึกหัดที่ 1
แก้ไข String ให้เป็น Title Case

```ruby
# เฉลย
def title_case(str)
  str.split(" ").map(&:capitalize).join(" ")
end

puts title_case("hello world ruby programming")
# => Hello World Ruby Programming
```

### แบบฝึกหัดที่ 2
นับตัวอักษรในแต่ละ Category

```ruby
# เฉลย
def analyze_string(str)
  {
    total: str.length,
    uppercase: str.count("A-Z"),
    lowercase: str.count("a-z"),
    digits: str.count("0-9"),
    spaces: str.count(" "),
    others: str.length - str.count("A-Za-z0-9 ")
  }
end

result = analyze_string("Hello, World! 123")
result.each do |key, value|
  puts "#{key}: #{value}"
end
```

### แบบฝึกหัดที่ 3
ตรวจสอบ Palindrome

```ruby
# เฉลย
def palindrome?(str)
  cleaned = str.downcase.gsub(/[^a-z0-9]/, "")
  cleaned == cleaned.reverse
end

puts palindrome?("racecar")       # => true
puts palindrome?("A man a plan a canal Panama")  # => true
puts palindrome?("hello")         # => false
```

### แบบฝึกหัดที่ 4
นับความถี่ของคำ

```ruby
# เฉลย
def word_frequency(text)
  words = text.downcase.scan(/\w+/)
  frequency = Hash.new(0)
  words.each { |word| frequency[word] += 1 }
  frequency.sort_by { |_, count| -count }
end

text = "the quick brown fox jumps over the lazy dog the fox"
word_frequency(text).first(5).each do |word, count|
  puts "#{word}: #{count}"
end
# the: 3
# fox: 2
# ...
```

### แบบฝึกหัดที่ 5
Format ตัวเลขให้มี Comma

```ruby
# เฉลย
def format_number(number)
  number.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\1,').reverse
end

puts format_number(1234567)   # => 1,234,567
puts format_number(999)       # => 999
puts format_number(1234)      # => 1,234
puts format_number(12345678)  # => 12,345,678
```

### แบบฝึกหัดที่ 6
Slugify String สำหรับ URL

```ruby
# เฉลย
def slugify(str)
  str.downcase
     .gsub(/[^\w\s-]/, '')
     .gsub(/\s+/, '-')
     .gsub(/-+/, '-')
     .chomp('-')
     .sub(/^-/, '')
end

puts slugify("Hello, World!")           # => hello-world
puts slugify("Ruby on Rails Tutorial")  # => ruby-on-rails-tutorial
puts slugify("  Trim   Spaces  ")       # => trim-spaces
```

### แบบฝึกหัดที่ 7
ตัด String ยาวและเพิ่ม ...

```ruby
# เฉลย
def truncate(str, length: 50, omission: "...")
  return str if str.length <= length
  str[0, length - omission.length] + omission
end

long_text = "Ruby is a dynamic, open source programming language with a focus on simplicity and productivity."
puts truncate(long_text, length: 40)
# => "Ruby is a dynamic, open source prog..."
puts truncate(long_text, length: 20, omission: "…")
# => "Ruby is a dynamic, …"
```

### แบบฝึกหัดที่ 8
สร้าง Simple Cipher

```ruby
# เฉลย
class CaesarCipher
  def initialize(shift)
    @shift = shift % 26
  end
  
  def encrypt(text)
    text.chars.map do |char|
      if char =~ /[a-z]/
        ((char.ord - 97 + @shift) % 26 + 97).chr
      elsif char =~ /[A-Z]/
        ((char.ord - 65 + @shift) % 26 + 65).chr
      else
        char
      end
    end.join
  end
  
  def decrypt(text)
    CaesarCipher.new(26 - @shift).encrypt(text)
  end
end

cipher = CaesarCipher.new(13)
encrypted = cipher.encrypt("Hello, World!")
puts encrypted  # => Uryyb, Jbeyq!
puts cipher.decrypt(encrypted)  # => Hello, World!
```

### แบบฝึกหัดที่ 9
Parse CSV String

```ruby
# เฉลย
def parse_csv(csv_string, delimiter: ",")
  lines = csv_string.strip.split("\n")
  headers = lines.first.split(delimiter).map(&:strip)
  
  lines[1..].map do |line|
    values = line.split(delimiter).map(&:strip)
    headers.zip(values).to_h
  end
end

csv = <<~CSV
  ชื่อ,อายุ,เมือง
  สมชาย,25,กรุงเทพ
  สมหญิง,22,เชียงใหม่
  สมศรี,30,ภูเก็ต
CSV

data = parse_csv(csv)
data.each do |row|
  puts "#{row['ชื่อ']} อายุ #{row['อายุ']} ปี อยู่ที่ #{row['เมือง']}"
end
```

### แบบฝึกหัดที่ 10
Validate Email

```ruby
# เฉลย
def valid_email?(email)
  email.match?(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
end

emails = [
  "valid@example.com",
  "user.name@domain.co.th",
  "invalid@",
  "no-at-sign.com",
  "user@domain"
]

emails.each do |email|
  status = valid_email?(email) ? "ถูกต้อง" : "ไม่ถูกต้อง"
  puts "#{email}: #{status}"
end
```

### แบบฝึกหัดที่ 11
String Compression (Run-Length Encoding)

```ruby
# เฉลย
def compress(str)
  result = ""
  i = 0
  
  while i < str.length
    char = str[i]
    count = 1
    
    while i + count < str.length && str[i + count] == char
      count += 1
    end
    
    result += count > 1 ? "#{char}#{count}" : char
    i += count
  end
  
  result.length < str.length ? result : str
end

puts compress("aabbbccdddd")   # => a2b3c2d4
puts compress("abcd")          # => abcd (ไม่ compress เพราะยาวขึ้น)
puts compress("aaabbaaa")      # => a3b2a3
```

### แบบฝึกหัดที่ 12
Word Wrap

```ruby
# เฉลย
def word_wrap(text, line_length: 60)
  words = text.split(" ")
  lines = []
  current_line = ""
  
  words.each do |word|
    if current_line.empty?
      current_line = word
    elsif (current_line + " " + word).length <= line_length
      current_line += " " + word
    else
      lines << current_line
      current_line = word
    end
  end
  
  lines << current_line unless current_line.empty?
  lines.join("\n")
end

long_text = "Ruby is a dynamic, reflective, object-oriented, general-purpose programming language. It was designed and developed in the mid-1990s by Yukihiro Matsumoto in Japan."
puts word_wrap(long_text, line_length: 50)
```

### แบบฝึกหัดที่ 13
String Tokenizer

```ruby
# เฉลย
def tokenize(expression)
  tokens = []
  i = 0
  
  while i < expression.length
    case expression[i]
    when /\d/
      num = ""
      while i < expression.length && expression[i] =~ /[\d.]/
        num += expression[i]
        i += 1
      end
      tokens << { type: :number, value: num.include?(".") ? num.to_f : num.to_i }
    when /[+\-*\/]/
      tokens << { type: :operator, value: expression[i] }
      i += 1
    when "("
      tokens << { type: :lparen, value: "(" }
      i += 1
    when ")"
      tokens << { type: :rparen, value: ")" }
      i += 1
    when " "
      i += 1
    else
      i += 1
    end
  end
  
  tokens
end

puts tokenize("3 + 4 * (2 - 1)").inspect
```

### แบบฝึกหัดที่ 14
Levenshtein Distance (Edit Distance)

```ruby
# เฉลย
def levenshtein_distance(s, t)
  m = s.length
  n = t.length
  
  # สร้างตาราง
  d = Array.new(m + 1) { Array.new(n + 1) }
  
  (0..m).each { |i| d[i][0] = i }
  (0..n).each { |j| d[0][j] = j }
  
  (1..m).each do |i|
    (1..n).each do |j|
      cost = s[i-1] == t[j-1] ? 0 : 1
      d[i][j] = [
        d[i-1][j] + 1,        # deletion
        d[i][j-1] + 1,        # insertion
        d[i-1][j-1] + cost    # substitution
      ].min
    end
  end
  
  d[m][n]
end

puts levenshtein_distance("kitten", "sitting")   # => 3
puts levenshtein_distance("saturday", "sunday")  # => 3
puts levenshtein_distance("", "hello")           # => 5
```

### แบบฝึกหัดที่ 15
Template Engine ง่ายๆ

```ruby
# เฉลย
def render_template(template, variables)
  template.gsub(/\{\{(\w+)\}\}/) do
    key = $1
    variables[key] || variables[key.to_sym] || "{{#{key}}}"
  end
end

template = <<~HTML
  สวัสดี {{name}}!
  คุณมีอีเมล {{email}}
  สถานะ: {{status}}
HTML

puts render_template(template, {
  "name" => "สมชาย",
  "email" => "somchai@example.com",
  "status" => "Active"
})
```

### แบบฝึกหัดที่ 16
String Difference Highlighter

```ruby
# เฉลย
def highlight_differences(str1, str2)
  result = ""
  max_len = [str1.length, str2.length].max
  
  max_len.times do |i|
    c1 = str1[i] || " "
    c2 = str2[i] || " "
    
    if c1 == c2
      result += c1
    else
      result += "[#{c1}→#{c2}]"
    end
  end
  
  result
end

puts highlight_differences("Hello World", "Hello Ruby!")
# => Hello [W→R][o→u][r→b][l→y][d→!]
```

### แบบฝึกหัดที่ 17
Thai Number to Text

```ruby
# เฉลย
def thai_number(n)
  digits = %w[ศูนย์ หนึ่ง สอง สาม สี่ ห้า หก เจ็ด แปด เก้า]
  units = ["", "สิบ", "ร้อย", "พัน", "หมื่น", "แสน", "ล้าน"]
  
  return digits[0] if n == 0
  
  result = ""
  n.to_s.chars.reverse.each_with_index do |digit, i|
    d = digit.to_i
    next if d == 0
    result = digits[d] + units[i] + result
  end
  
  result
end

puts thai_number(0)     # => ศูนย์
puts thai_number(5)     # => ห้า
puts thai_number(42)    # => สี่สิบสอง
puts thai_number(1234)  # => หนึ่งพันสองร้อยสามสิบสี่
```

### แบบฝึกหัดที่ 18
String Pattern Matching

```ruby
# เฉลย
class StringPattern
  PATTERNS = {
    email: /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i,
    phone: /\A(\+66|0)\d{8,9}\z/,
    thai_id: /\A\d{13}\z/,
    url: /\Ahttps?:\/\/[\w\-.]+\.[a-z]{2,}(\/\S*)?\z/i,
    ip: /\A(\d{1,3}\.){3}\d{1,3}\z/
  }
  
  def self.validate(value, type)
    pattern = PATTERNS[type]
    return "ไม่รู้จัก type: #{type}" unless pattern
    
    if value.match?(pattern)
      "#{value}: ถูกต้อง (#{type})"
    else
      "#{value}: ไม่ถูกต้อง (#{type})"
    end
  end
end

puts StringPattern.validate("user@example.com", :email)    # => ถูกต้อง
puts StringPattern.validate("0812345678", :phone)           # => ถูกต้อง
puts StringPattern.validate("1234567890123", :thai_id)     # => ถูกต้อง
puts StringPattern.validate("https://www.ruby-lang.org", :url)  # => ถูกต้อง
puts StringPattern.validate("192.168.1.1", :ip)            # => ถูกต้อง
```

### แบบฝึกหัดที่ 19
Multi-language Greeting

```ruby
# เฉลย
GREETINGS = {
  th: "สวัสดี",
  en: "Hello",
  ja: "こんにちは",
  zh: "你好",
  ko: "안녕하세요",
  fr: "Bonjour",
  de: "Hallo",
  es: "Hola"
}.freeze

def greet(name, lang = :th)
  greeting = GREETINGS[lang] || GREETINGS[:en]
  "#{greeting}, #{name}!"
end

GREETINGS.each_key do |lang|
  puts greet("สมชาย", lang)
end
```

### แบบฝึกหัดที่ 20
String Statistics

```ruby
# เฉลย
def string_stats(text)
  words = text.split(/\s+/).reject(&:empty?)
  sentences = text.split(/[.!?]/).reject(&:empty?)
  
  {
    characters: text.length,
    characters_no_spaces: text.gsub(/\s/, "").length,
    words: words.length,
    sentences: sentences.length,
    avg_word_length: words.empty? ? 0 : (words.sum(&:length).to_f / words.length).round(2),
    avg_sentence_length: sentences.empty? ? 0 : (words.length.to_f / sentences.length).round(2),
    most_common_word: words.group_by(&:downcase).max_by { |_, v| v.length }&.first,
    unique_words: words.map(&:downcase).uniq.length
  }
end

sample_text = "Ruby is a powerful language. Ruby makes programming fun! Ruby is used in many applications."
stats = string_stats(sample_text)
stats.each do |key, value|
  puts "#{key}: #{value}"
end
```

---

## สรุป

ในตอนที่ 3 นี้ เราได้เรียนรู้:

1. **การสร้าง String** - Single/Double quotes, %q{}, %Q{}
2. **String Interpolation** - การใส่ Expression ใน String
3. **Case Methods** - upcase, downcase, capitalize, swapcase
4. **Stripping/Padding** - strip, ljust, rjust, center
5. **Searching** - include?, index, rindex, scan, match
6. **Replacing** - sub, gsub ด้วย String, Regex, และ Block
7. **Splitting/Joining** - split, chars, bytes, lines, join
8. **Formatting** - %, sprintf, format, heredoc
9. **Frozen Strings** - freeze, frozen_string_literal
10. **Encoding** - encoding, bytesize, valid_encoding?
11. **String as Array** - [], slice, each_char
12. **Comparison** - ==, <=>, casecmp

ในตอนต่อไปเราจะเรียนรู้เรื่อง **Numbers และ Math** ในเชิงลึก

---

*ตอนที่ 3 จบแล้ว - ไปต่อตอนที่ 4: Numbers และ Math*
