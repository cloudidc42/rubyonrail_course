# ตอนที่ 3: Strings และการจัดการข้อความ (Steps 26-45)

## บทนำ

String (สตริง) คือข้อมูลประเภทข้อความใน Ruby ซึ่งเป็นหนึ่งในประเภทข้อมูลที่ใช้บ่อยที่สุดในการเขียนโปรแกรม ไม่ว่าจะเป็นการรับข้อมูลจากผู้ใช้, การแสดงผล, การประมวลผลข้อความ หรือการสร้าง HTML Ruby มี String API ที่ทรงพลังมากพร้อมเมธอดหลายสิบตัวที่ช่วยให้จัดการข้อความได้อย่างสะดวก

ในตอนนี้เราจะเรียนรู้ทุกอย่างที่จำเป็นเกี่ยวกับ String ตั้งแต่พื้นฐานจนถึงขั้นสูง

---

## Step 26: Single Quotes vs Double Quotes

### ความแตกต่างหลัก

Ruby รองรับการสร้าง String ได้ 2 วิธีหลัก คือใช้เครื่องหมาย Single Quote (`'`) หรือ Double Quote (`"`) ความแตกต่างที่สำคัญที่สุดคือ:

- **Single Quote**: แสดงข้อความตรงๆ ไม่มีการแปลง escape sequence (ยกเว้น `\\` และ `\'`)
- **Double Quote**: รองรับ String Interpolation และ Escape Sequences ทั้งหมด

```ruby
# Single Quote - แสดงข้อความตรงๆ
name = 'สมชาย'
greeting = 'สวัสดี'
puts greeting  # => สวัสดี

# Double Quote - รองรับ interpolation
puts "สวัสดี #{name}"  # => สวัสดี สมชาย

# Single Quote ไม่ทำ interpolation
puts 'สวัสดี #{name}'  # => สวัสดี #{name} (แสดงตรงๆ)
```

```ruby
# ความแตกต่างของ escape sequences
puts 'บรรทัดที่ 1\nบรรทัดที่ 2'
# => บรรทัดที่ 1\nบรรทัดที่ 2 (ไม่ขึ้นบรรทัดใหม่)

puts "บรรทัดที่ 1\nบรรทัดที่ 2"
# => บรรทัดที่ 1
# => บรรทัดที่ 2 (ขึ้นบรรทัดใหม่จริงๆ)
```

```ruby
# การใช้ single quote ภายใน double quote และในทางกลับกัน
message1 = "He said 'hello'"    # ใส่ ' ใน " ได้เลย
message2 = 'She said "goodbye"' # ใส่ " ใน ' ได้เลย

puts message1  # => He said 'hello'
puts message2  # => She said "goodbye"
```

```ruby
# Performance: Single quote เร็วกว่าเล็กน้อย (Ruby ไม่ต้องตรวจสอบ interpolation)
# แต่ในทางปฏิบัติไม่มีความแตกต่างที่สังเกตได้

# Best Practice: ใช้ single quote เมื่อไม่ต้องการ interpolation
config_key = 'database_host'
config_value = "#{ENV['DB_HOST'] || 'localhost'}"

puts config_key    # => database_host
puts config_value  # => localhost (ถ้าไม่มี ENV variable)
```

---

## Step 27: String Interpolation

### การแทรกค่าลงใน String

String Interpolation คือการแทรกค่าของตัวแปรหรือ expression ลงไปใน String โดยใช้ `#{...}` ซึ่งเป็นฟีเจอร์ที่ทรงพลังและใช้บ่อยมากใน Ruby

```ruby
# การแทรกตัวแปร
first_name = "สมชาย"
last_name = "ใจดี"
age = 25

puts "ชื่อ: #{first_name} #{last_name}"  # => ชื่อ: สมชาย ใจดี
puts "อายุ: #{age} ปี"                    # => อายุ: 25 ปี
```

```ruby
# การแทรก expression (การคำนวณ)
price = 100
quantity = 5
discount = 0.1

total = price * quantity
discounted = total * (1 - discount)

puts "ราคาต่อชิ้น: #{price} บาท"
puts "จำนวน: #{quantity} ชิ้น"
puts "ราคารวม: #{total} บาท"
puts "หลังหักส่วนลด #{discount * 100}%: #{discounted} บาท"
# => ราคาต่อชิ้น: 100 บาท
# => จำนวน: 5 ชิ้น
# => ราคารวม: 500 บาท
# => หลังหักส่วนลด 10.0%: 450.0 บาท
```

```ruby
# การแทรกผลลัพธ์จากเมธอด
text = "hello world"
puts "ข้อความ: '#{text}'"
puts "ตัวพิมพ์ใหญ่: '#{text.upcase}'"
puts "ความยาว: #{text.length} ตัวอักษร"
# => ข้อความ: 'hello world'
# => ตัวพิมพ์ใหญ่: 'HELLO WORLD'
# => ความยาว: 11 ตัวอักษร
```

```ruby
# การแทรก conditional expression
score = 85
puts "ผลการสอบ: #{score} - #{score >= 80 ? 'ผ่าน' : 'ไม่ผ่าน'}"
# => ผลการสอบ: 85 - ผ่าน

# การใช้งานซ้อนกัน (nested interpolation)
items = ["แอปเปิล", "กล้วย", "ส้ม"]
puts "มีสินค้า #{items.length} ชนิด: #{items.join(', ')}"
# => มีสินค้า 3 ชนิด: แอปเปิล, กล้วย, ส้ม
```

```ruby
# การ format ตัวเลขใน interpolation
pi = Math::PI
puts "ค่า PI = #{pi}"           # => ค่า PI = 3.141592653589793
puts "ค่า PI = #{pi.round(2)}"  # => ค่า PI = 3.14
puts "ค่า PI = #{'%.4f' % pi}"  # => ค่า PI = 3.1416
```

---

## Step 28: Escape Sequences

### ลำดับหนีอักขระ

Escape Sequences คืออักขระพิเศษที่ใช้แทนอักขระที่ไม่สามารถพิมพ์ได้โดยตรง โดยใช้ backslash (`\`) นำหน้า

```ruby
# Escape sequences ที่ใช้บ่อย
puts "Hello\nWorld"    # \n = newline (ขึ้นบรรทัดใหม่)
# => Hello
# => World

puts "Column1\tColumn2"  # \t = tab (เว้นระยะ)
# => Column1    Column2

puts "Say \"Hello\""  # \" = double quote ภายใน double quote string
# => Say "Hello"

puts 'That\'s great'  # \' = single quote ภายใน single quote string
# => That's great

puts "Back\\slash"   # \\ = backslash จริงๆ
# => Back\slash
```

```ruby
# Escape sequences อื่นๆ
puts "\a"  # \a = alert/bell (เสียงบี๊ป - ใน terminal บางตัว)
puts "\b"  # \b = backspace
puts "\r"  # \r = carriage return
puts "\0"  # \0 = null character

# Unicode escape
puts "\u0E2A\u0E27\u0E31\u0E2A\u0E14\u0E35"  # => สวัสดี
puts "\u2665"  # => ♥ (heart symbol)
puts "\u{1F600}"  # => 😀 (emoji - ใช้ \u{} สำหรับ code points มากกว่า 4 หลัก)
```

```ruby
# การใช้ escape ใน string paths (Windows style paths)
windows_path = "C:\\Users\\John\\Documents"
puts windows_path  # => C:\Users\John\Documents

# แบบ Unix/Mac ไม่ต้อง escape
unix_path = "/home/john/documents"
puts unix_path  # => /home/john/documents
```

```ruby
# การเปรียบเทียบ single vs double quote กับ escape
single = '\n\t\\"'  # แสดงตรงๆ (ยกเว้น \\)
double = "\n\t\""   # แปลงเป็น newline, tab, quote จริงๆ

puts single.length  # => 6 (แสดง \n\t\\ และ " เป็น 4 ตัวอักษร จริงๆ 6: \, n, \, t, \, ")
puts double.length  # => 3 (newline + tab + quote = 3 ตัวอักษร)
puts single.inspect  # => "\\n\\t\\\\\""
puts double.inspect  # => "\"\\n\\t\\\"\""
```

---

## Step 29: String Methods - Case Conversion

### การเปลี่ยนตัวพิมพ์

Ruby มีเมธอดที่ใช้เปลี่ยนตัวพิมพ์ (case) ของ String ได้หลายแบบ

```ruby
text = "hello WORLD ruby"

# upcase - เปลี่ยนเป็นตัวพิมพ์ใหญ่ทั้งหมด
puts text.upcase    # => HELLO WORLD RUBY

# downcase - เปลี่ยนเป็นตัวพิมพ์เล็กทั้งหมด
puts text.downcase  # => hello world ruby

# capitalize - ตัวแรกพิมพ์ใหญ่ ที่เหลือพิมพ์เล็ก
puts text.capitalize  # => Hello world ruby

# swapcase - สลับ: ใหญ่เป็นเล็ก เล็กเป็นใหญ่
puts text.swapcase  # => HELLO world RUBY
```

```ruby
# เมธอดแบบ bang (!) แก้ไข string ต้นฉบับ
text = "hello world"
text.upcase!
puts text  # => HELLO WORLD (ต้นฉบับถูกแก้ไข)

# เมธอดที่ไม่มี ! สร้าง string ใหม่
original = "hello"
upper = original.upcase
puts original  # => hello (ไม่เปลี่ยน)
puts upper     # => HELLO (string ใหม่)
```

```ruby
# การใช้งานจริง: Normalize input จากผู้ใช้
def normalize_name(name)
  name.strip.split.map(&:capitalize).join(' ')
end

puts normalize_name("  john   doe  ")   # => John Doe
puts normalize_name("MARY JANE")         # => Mary Jane
puts normalize_name("bob smith jones")   # => Bob Smith Jones
```

```ruby
# Case insensitive comparison
username1 = "Admin"
username2 = "admin"

# ผิด: ไม่ตรงกัน
puts username1 == username2  # => false

# ถูก: downcase ก่อน compare
puts username1.downcase == username2.downcase  # => true

# หรือใช้ casecmp
puts username1.casecmp(username2)   # => 0 (เท่ากัน)
puts username1.casecmp?("ADMIN")    # => true
```

---

## Step 30: String Whitespace Methods

### การจัดการช่องว่าง

```ruby
text = "  Hello, World!  "

# strip - ตัด whitespace ทั้งหน้าและหลัง
puts text.strip    # => "Hello, World!"

# lstrip - ตัด whitespace ด้านหน้า (left)
puts text.lstrip   # => "Hello, World!  "

# rstrip - ตัด whitespace ด้านหลัง (right)
puts text.rstrip   # => "  Hello, World!"

# ใน Ruby 2.5+ มี delete_prefix และ delete_suffix
text2 = "Hello, World!"
puts text2.delete_prefix("Hello, ")  # => World!
puts text2.delete_suffix("!")        # => Hello, World
```

```ruby
# chomp - ตัด newline ท้าย string (ใช้บ่อยกับ gets)
line = "Hello\n"
puts line.chomp    # => Hello (ไม่มี newline)
puts line.chomp.length  # => 5

line2 = "Hello\r\n"  # Windows line ending
puts line2.chomp    # => Hello

# chop - ตัดอักขระสุดท้ายออก (ไม่ว่าจะเป็นอะไร)
puts "Hello".chop   # => Hell
puts "Hello\n".chop # => Hello
```

```ruby
# squeeze - รวมอักขระที่ซ้ำติดกันเป็นอันเดียว
puts "aaabbbccc".squeeze         # => abc
puts "hello   world".squeeze(" ") # => hello world (รวม space)
puts "aabbccdd".squeeze("a-c")    # => abcdd (รวมเฉพาะ a, b, c)

# ใช้ squeeze เพื่อ normalize ข้อมูล
user_input = "I  love   Ruby!!!"
normalized = user_input.squeeze(" ").squeeze("!")
puts normalized  # => I love Ruby!
```

```ruby
# การจัดการ whitespace ใน data processing
data = ["  apple  ", "\tbanana\n", "  cherry  "]
cleaned = data.map(&:strip)
puts cleaned.inspect  # => ["apple", "banana", "cherry"]

# ตรวจสอบว่า string ว่างหรือมีแต่ whitespace
puts "".empty?        # => true
puts "   ".empty?     # => false (มี space อยู่)
puts "   ".strip.empty?  # => true
```

---

## Step 31: String Length and Checking Methods

### การตรวจสอบ String

```ruby
text = "Hello, สวัสดี Ruby!"

# length / size - นับจำนวน characters
puts text.length  # => 19
puts text.size    # => 19 (เหมือนกัน)

# นับ bytes (แตกต่างกันสำหรับ UTF-8)
puts text.bytesize  # => 30 (ภาษาไทยใช้ 3 bytes ต่อตัวอักษร)
```

```ruby
# empty? - ตรวจสอบว่าว่างหรือไม่
puts "".empty?      # => true
puts "hello".empty? # => false

# include? - ตรวจสอบว่ามีข้อความย่อยหรือไม่
sentence = "The quick brown fox"
puts sentence.include?("quick")   # => true
puts sentence.include?("slow")    # => false
puts sentence.include?("Q")       # => false (case sensitive)
puts sentence.downcase.include?("q") # => true
```

```ruby
# start_with? - ตรวจสอบขึ้นต้นด้วย
filename = "document.pdf"
puts filename.start_with?("doc")   # => true
puts filename.start_with?("img")   # => false
puts filename.start_with?("doc", "img", "vid")  # => true (รับหลายค่า)

# end_with? - ตรวจสอบลงท้ายด้วย
puts filename.end_with?(".pdf")    # => true
puts filename.end_with?(".jpg")    # => false
puts filename.end_with?(".pdf", ".jpg", ".png")  # => true
```

```ruby
# การใช้งานจริง: ตรวจสอบประเภทไฟล์
def image_file?(filename)
  filename.end_with?(".jpg", ".jpeg", ".png", ".gif", ".webp")
end

def video_file?(filename)
  filename.end_with?(".mp4", ".avi", ".mkv", ".mov")
end

files = ["photo.jpg", "movie.mp4", "data.csv", "avatar.png"]
files.each do |file|
  type = if image_file?(file)
    "รูปภาพ"
  elsif video_file?(file)
    "วิดีโอ"
  else
    "ไฟล์อื่นๆ"
  end
  puts "#{file}: #{type}"
end
```

---

## Step 32: String Search and Replace

### การค้นหาและแทนที่

```ruby
text = "I love Ruby. Ruby is great. Ruby is fun."

# sub - แทนที่ครั้งแรกที่พบ
puts text.sub("Ruby", "Python")
# => I love Python. Ruby is great. Ruby is fun.

# gsub - แทนที่ทุกครั้งที่พบ
puts text.gsub("Ruby", "Python")
# => I love Python. Python is great. Python is fun.
```

```ruby
# การใช้ Regular Expression กับ sub และ gsub
email = "user@example.com"
censored = email.gsub(/@.*/, "@***")
puts censored  # => user@***

# แทนที่พร้อม capture group
phone = "081-234-5678"
formatted = phone.gsub(/(\d{3})-(\d{3})-(\d{4})/, "(\\1) \\2-\\3")
puts formatted  # => (081) 234-5678

# ใช้ block กับ gsub เพื่อการแปลงซับซ้อน
text = "hello world ruby"
result = text.gsub(/\b\w/) { |match| match.upcase }
puts result  # => Hello World Ruby
```

```ruby
# gsub กับ Hash สำหรับการแทนที่หลายค่าพร้อมกัน
text = "I have cats and dogs"
animals = {"cats" => "แมว", "dogs" => "สุนัข"}
puts text.gsub(/cats|dogs/, animals)
# => I have แมว and สุนัข

# การ escape HTML
def html_escape(str)
  str.gsub(/[&<>"']/) do |char|
    case char
    when "&" then "&amp;"
    when "<" then "&lt;"
    when ">" then "&gt;"
    when '"' then "&quot;"
    when "'" then "&#39;"
    end
  end
end

puts html_escape("<script>alert('XSS')</script>")
# => &lt;script&gt;alert(&#39;XSS&#39;)&lt;/script&gt;
```

```ruby
# tr - แทนที่ทีละตัวอักษร (translate)
puts "hello".tr('aeiou', '*')     # => h*ll*
puts "hello".tr('el', 'ip')       # => hippo
puts "hello world".tr('a-m', 'A-M')  # => HELLo worLD

# tr_s - tr แล้ว squeeze ด้วย
puts "hello".tr_s('l', 'r')  # => hero (ll -> r -> r)
```

---

## Step 33: String Splitting and Joining

### การแบ่งและรวม String

```ruby
# split - แบ่ง string เป็น array
sentence = "The quick brown fox"
words = sentence.split
puts words.inspect  # => ["The", "quick", "brown", "fox"]

# split ด้วย delimiter
csv = "apple,banana,cherry"
fruits = csv.split(",")
puts fruits.inspect  # => ["apple", "banana", "cherry"]

# split ด้วย regex
text = "one1two2three3four"
parts = text.split(/\d/)
puts parts.inspect  # => ["one", "two", "three", "four"]
```

```ruby
# split กับ limit
text = "a:b:c:d:e"
puts text.split(":", 3).inspect   # => ["a", "b", "c:d:e"]
puts text.split(":", -1).inspect  # => ["a", "b", "c", "d", "e"]

# split ข้อความว่าง
puts "".split(",").inspect        # => []
puts "a,,b".split(",").inspect    # => ["a", "", "b"]
puts "a,,b".split(",", -1).inspect # => ["a", "", "b"]
```

```ruby
# chars - แบ่งเป็น array ของตัวอักษร
puts "hello".chars.inspect  # => ["h", "e", "l", "l", "o"]
puts "สวัสดี".chars.inspect # => ["ส", "ว", "ั", "ส", "ด", "ี"]

# bytes - array ของ byte values
puts "hello".bytes.inspect  # => [104, 101, 108, 108, 111]
puts "ส".bytes.inspect       # => [224, 184, 170] (UTF-8 encoding)

# lines - แบ่งตาม newlines
text = "line 1\nline 2\nline 3"
puts text.lines.inspect  # => ["line 1\n", "line 2\n", "line 3"]
```

```ruby
# join - รวม array กลับเป็น string (เมธอดของ Array)
words = ["สวัสดี", "โลก", "Ruby"]
puts words.join(", ")  # => สวัสดี, โลก, Ruby
puts words.join(" - ") # => สวัสดี - โลก - Ruby
puts words.join         # => สวัสดีโลกRuby

# การใช้งานจริง: สร้าง SQL IN clause
ids = [1, 2, 3, 4, 5]
puts "SELECT * FROM users WHERE id IN (#{ids.join(', ')})"
# => SELECT * FROM users WHERE id IN (1, 2, 3, 4, 5)
```

---

## Step 34: String Navigation Methods

### การนำทางและดึงข้อมูลจาก String

```ruby
text = "Hello, World!"

# index - หา index แรกที่พบ
puts text.index("o")     # => 4
puts text.index("o", 5)  # => 8 (เริ่มหาจาก index 5)
puts text.index("xyz")   # => nil (ไม่พบ)

# rindex - หา index สุดท้ายที่พบ
puts text.rindex("o")    # => 8
puts text.rindex("l")    # => 10
```

```ruby
# slice / [] - ดึงส่วนของ string
text = "Hello, World!"

puts text[0]       # => H (ตัวอักษรที่ index 0)
puts text[-1]      # => ! (ตัวสุดท้าย)
puts text[0, 5]    # => Hello (เริ่มที่ 0 ยาว 5 ตัว)
puts text[7..]     # => World! (จาก index 7 ถึงสุดท้าย)
puts text[7..11]   # => World (จาก 7 ถึง 11)
puts text[7...11]  # => Worl (exclusive end)
puts text["World"] # => World (ดึง matching substring)
```

```ruby
# slice! - ดึงแล้วลบออกจาก string ต้นฉบับ
text = "Hello, World!"
extracted = text.slice!(0, 7)
puts extracted  # => Hello, 
puts text       # => World! (ถูกตัดออกแล้ว)
```

```ruby
# การใช้งานจริง: Parse ข้อมูล
date_string = "2024-03-15"
year = date_string[0, 4]
month = date_string[5, 2]
day = date_string[8, 2]

puts "ปี: #{year}, เดือน: #{month}, วัน: #{day}"
# => ปี: 2024, เดือน: 03, วัน: 15

# แบบอื่น
parts = date_string.split("-")
puts "ปี: #{parts[0]}, เดือน: #{parts[1]}, วัน: #{parts[2]}"
```

---

## Step 35: String Formatting and Alignment

### การจัดรูปแบบและการจัดวาง

```ruby
text = "Ruby"

# center - จัดกึ่งกลาง
puts text.center(20)        # => "        Ruby        "
puts text.center(20, "-")   # => "--------Ruby--------"

# ljust - จัดชิดซ้าย (left justified)
puts text.ljust(20)         # => "Ruby                "
puts text.ljust(20, ".")    # => "Ruby................"

# rjust - จัดชิดขวา (right justified)
puts text.rjust(20)         # => "                Ruby"
puts text.rjust(20, "0")    # => "0000000000000000Ruby"
```

```ruby
# การสร้างตาราง
headers = ["ชื่อ", "อายุ", "เมือง"]
data = [
  ["สมชาย", "25", "กรุงเทพ"],
  ["สมหญิง", "30", "เชียงใหม่"],
  ["สมศักดิ์", "22", "ขอนแก่น"]
]

# พิมพ์ header
puts headers.map { |h| h.ljust(12) }.join(" | ")
puts "-" * 45

# พิมพ์ข้อมูล
data.each do |row|
  puts row.map { |cell| cell.ljust(12) }.join(" | ")
end
```

```ruby
# String % operator สำหรับ formatting
puts "Hello, %s!" % "World"           # => Hello, World!
puts "%.2f" % 3.14159                 # => 3.14
puts "%05d" % 42                      # => 00042
puts "%10s" % "right"                 # =>      right
puts "%-10s" % "left"                 # => left      
puts "Name: %s, Age: %d" % ["Ruby", 30]  # => Name: Ruby, Age: 30
```

```ruby
# sprintf / format - เหมือน % แต่อ่านง่ายกว่า
result = sprintf("ราคา: %,.2f บาท", 1234567.89)
puts result  # => ราคา: 1234567.89 บาท

result2 = format("สวัสดี %s! คุณอายุ %d ปี", "สมชาย", 25)
puts result2  # => สวัสดี สมชาย! คุณอายุ 25 ปี

# ใช้ named references
result3 = format("Hello, %{name}! You are %{age} years old.",
                 name: "Alice", age: 30)
puts result3  # => Hello, Alice! You are 30 years old.
```

---

## Step 36: Heredoc

### Heredoc สำหรับ Multiline Strings

Heredoc ใช้สร้าง String หลายบรรทัดโดยไม่ต้องใส่ escape sequence

```ruby
# Basic heredoc
message = <<~HEREDOC
  สวัสดีครับ
  นี่คือข้อความ
  หลายบรรทัด
HEREDOC

puts message
# => สวัสดีครับ
# => นี่คือข้อความ
# => หลายบรรทัด
```

```ruby
# Heredoc รองรับ interpolation (เหมือน double quote)
name = "สมชาย"
age = 25

bio = <<~HEREDOC
  ชื่อ: #{name}
  อายุ: #{age} ปี
  วันที่สมัคร: #{Time.now.strftime("%d/%m/%Y")}
HEREDOC

puts bio
```

```ruby
# Heredoc แบบ single quote - ไม่ทำ interpolation
template = <<~'HEREDOC'
  Dear #{name},
  Your order #{order_id} is ready.
  Total: #{total} baht
HEREDOC

puts template
# พิมพ์ตรงๆ ไม่แปลง #{...}
```

```ruby
# การใช้ heredoc กับ method chaining
html = <<~HTML.strip.upcase
  <div>
    <p>Hello World</p>
  </div>
HTML

puts html
# => <DIV>
# =>   <P>HELLO WORLD</P>
# => </DIV>
```

```ruby
# Heredoc สำหรับ SQL query
user_id = 42

query = <<~SQL
  SELECT 
    users.id,
    users.name,
    orders.total
  FROM users
  LEFT JOIN orders ON orders.user_id = users.id
  WHERE users.id = #{user_id}
    AND orders.status = 'completed'
  ORDER BY orders.created_at DESC
SQL

puts query
```

---

## Step 37: String Multiplication and Concatenation

### การคูณและการรวม String

```ruby
# การคูณ string ด้วย * 
puts "Ruby! " * 3    # => Ruby! Ruby! Ruby! 
puts "-" * 40        # => ----------------------------------------
puts "ha" * 5        # => hahahahahaha (ไม่, เป็น hahahahaha)

# ใช้งานจริง: สร้าง separator lines
def section_header(title)
  width = 50
  separator = "=" * width
  "#{separator}\n#{title.center(width)}\n#{separator}"
end

puts section_header("บทที่ 3: Strings")
```

```ruby
# String Concatenation ด้วย + 
first = "Hello"
second = ", "
third = "World!"

result = first + second + third
puts result  # => Hello, World!

# ข้อควรระวัง: + สร้าง string ใหม่ทุกครั้ง (inefficient ใน loop)
```

```ruby
# << (shovel operator) - ต่อท้าย string ต้นฉบับ (efficient)
buffer = ""
buffer << "Hello"
buffer << ", "
buffer << "World!"
puts buffer  # => Hello, World!

# เปรียบเทียบ performance
require 'benchmark'

n = 10000

Benchmark.bm do |x|
  x.report("plus:") do
    str = ""
    n.times { str = str + "a" }  # สร้าง string ใหม่ทุกครั้ง
  end
  
  x.report("shovel:") do
    str = ""
    n.times { str << "a" }  # ต่อท้าย string เดิม (เร็วกว่า)
  end
end
```

```ruby
# concat method
str = "Hello"
str.concat(", ", "World", "!")
puts str  # => Hello, World!

# prepend - เพิ่มที่ต้น string
str = "World!"
str.prepend("Hello, ")
puts str  # => Hello, World!

# insert - แทรกที่ตำแหน่งใดก็ได้
str = "Hello World"
str.insert(5, ",")
puts str  # => Hello, World
```

---

## Step 38: String Comparison

### การเปรียบเทียบ String

```ruby
# == ตรวจสอบความเท่ากัน
puts "hello" == "hello"   # => true
puts "hello" == "Hello"   # => false (case sensitive)
puts "hello" == "world"   # => false

# != ตรวจสอบความไม่เท่ากัน
puts "hello" != "world"   # => true
```

```ruby
# <=> (spaceship operator) - เปรียบเทียบ lexicographic
puts "apple" <=> "banana"  # => -1 (apple มาก่อน)
puts "cherry" <=> "banana" # => 1 (cherry มาหลัง)
puts "apple" <=> "apple"   # => 0 (เท่ากัน)

# ใช้กับ sort
fruits = ["cherry", "apple", "banana", "date"]
puts fruits.sort.inspect
# => ["apple", "banana", "cherry", "date"]
```

```ruby
# < > <= >= เปรียบเทียบแบบ alphabetical
puts "apple" < "banana"   # => true
puts "zoo" > "ant"        # => true

# casecmp - เปรียบเทียบโดยไม่สนใจตัวพิมพ์
puts "Hello".casecmp("hello")   # => 0 (เท่ากัน)
puts "Hello".casecmp?("HELLO")  # => true

# equal? - ตรวจสอบว่าเป็น object เดียวกัน (object identity)
a = "hello"
b = "hello"
c = a

puts a == b      # => true (ค่าเหมือนกัน)
puts a.equal?(b) # => false (คนละ object)
puts a.equal?(c) # => true (เป็น object เดียวกัน)
```

---

## Step 39: Advanced String Methods

### เมธอดขั้นสูง

```ruby
# reverse - กลับ string
puts "Hello".reverse   # => olleH
puts "12345".reverse   # => 54321

# การตรวจสอบ palindrome
def palindrome?(str)
  clean = str.downcase.gsub(/[^a-z0-9]/, "")
  clean == clean.reverse
end

puts palindrome?("racecar")   # => true
puts palindrome?("A man a plan a canal Panama")  # => true
puts palindrome?("hello")     # => false
```

```ruby
# count - นับจำนวนตัวอักษรที่กำหนด
text = "Hello, World!"
puts text.count("l")      # => 3
puts text.count("aeiou")  # => 3 (นับสระทั้งหมด)
puts text.count("a-z")    # => 8 (นับอักษรพิมพ์เล็กทั้งหมด)

# delete - ลบตัวอักษรที่กำหนดออก
puts text.delete("l")         # => Heo, Word!
puts text.delete("aeiouAEIOU") # => Hll, Wrld!
puts text.delete("a-z")        # => H, W!
```

```ruby
# scan - หาทุก match ของ pattern
text = "The phone is 081-234-5678 or 02-345-6789"
phones = text.scan(/\d+-\d+-\d+/)
puts phones.inspect  # => ["081-234-5678", "02-345-6789"]

# scan กับ capture group
email_text = "Contact john@example.com or jane@test.org"
emails = email_text.scan(/[\w.]+@[\w.]+/)
puts emails.inspect  # => ["john@example.com", "jane@test.org"]
```

```ruby
# succ / next - string ถัดไป
puts "a".succ    # => b
puts "z".succ    # => aa
puts "Az".succ   # => Ba
puts "zz".succ   # => aaa
puts "hello".succ # => hellp

# ใช้สร้าง unique IDs
counter = "AA01"
5.times do
  puts counter
  counter = counter.succ
end
# => AA01, AA02, AA03, AA04, AA05
```

---

## Step 40: String Encoding

### String Encoding และ Unicode

```ruby
# ตรวจสอบ encoding
text = "Hello สวัสดี"
puts text.encoding  # => UTF-8

# แปลง encoding
ascii_text = "Hello".encode("ASCII")
puts ascii_text.encoding  # => ASCII

# force_encoding - เปลี่ยน encoding โดยไม่แปลง bytes
raw_bytes = "\xE0\xB8\xAA\xE0\xB8\xA7\xE0\xB8\xB1\xE0\xB8\xAA\xE0\xB8\x94\xE0\xB8\xB5"
text = raw_bytes.force_encoding("UTF-8")
puts text  # => สวัสดี
```

```ruby
# valid_encoding? - ตรวจสอบว่า encoding ถูกต้อง
puts "Hello".valid_encoding?       # => true
puts "สวัสดี".valid_encoding?      # => true

# b - แปลงเป็น binary (ASCII-8BIT)
binary = "Hello".b
puts binary.encoding  # => ASCII-8BIT

# การจัดการ string ที่มีปัญหา encoding
def safe_string(str)
  str.encode("UTF-8", invalid: :replace, undef: :replace, replace: "?")
end
```

---

## Step 41: frozen_string_literal

### Performance Optimization ด้วย frozen_string_literal

```ruby
# ไฟล์ Ruby ปกติ
str = "hello"
str << " world"  # OK, string สามารถแก้ไขได้
puts str  # => hello world
```

```ruby
# frozen_string_literal: true
# เพิ่ม magic comment นี้บนสุดของไฟล์

# frozen_string_literal: true

str = "hello"
# str << " world"  # => FrozenError: can't modify frozen String

# ต้องสร้าง string ใหม่แทน
str = str + " world"
puts str  # => hello world

# หรือ unfreeze ด้วย dup
mutable = "hello".dup
mutable << " world"
puts mutable  # => hello world
```

```ruby
# frozen_string_literal: true

# เหตุผลที่ใช้ frozen_string_literal:
# 1. Performance: Ruby ใช้ string เดียวกันสำหรับ string literals เหมือนกัน
# 2. Memory: ลดการสร้าง string objects ซ้ำ
# 3. Safety: ป้องกันการแก้ไข string โดยไม่ตั้งใจ

GREETING = "Hello"  # ถูก freeze โดยอัตโนมัติ
puts GREETING.frozen?  # => true

# การสร้าง mutable string เมื่อต้องการ
dynamic = +"Hello"  # + prefix สร้าง mutable string
dynamic << " World"
puts dynamic  # => Hello World
```

---

## Step 42: String Patterns และ Regular Expressions

### การใช้ Regex กับ String

```ruby
# match - ตรวจสอบว่า match หรือไม่
email = "user@example.com"
if email.match(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
  puts "Email ถูกต้อง"
else
  puts "Email ไม่ถูกต้อง"
end

# =~ operator
puts ("hello" =~ /ell/)   # => 1 (index ที่พบ)
puts ("hello" =~ /xyz/)   # => nil (ไม่พบ)
```

```ruby
# match กับ capture groups
date = "15/03/2024"
if match = date.match(/(\d{2})\/(\d{2})\/(\d{4})/)
  day, month, year = match[1], match[2], match[3]
  puts "วัน: #{day}, เดือน: #{month}, ปี: #{year}"
end

# Named capture groups
if match = date.match(/(?<day>\d{2})\/(?<month>\d{2})\/(?<year>\d{4})/)
  puts "วัน: #{match[:day]}"
  puts "เดือน: #{match[:month]}"
  puts "ปี: #{match[:year]}"
end
```

```ruby
# gsub กับ Regex และ block
text = "Hello World Ruby"
# CamelCase to snake_case
snake = text.gsub(/([A-Z])/) { "_#{$1.downcase}" }.sub(/^_/, "")
puts snake  # => _hello _world _ruby -> ต้องปรับ

# snake_case จริงๆ
def to_snake_case(str)
  str.gsub(/([A-Z]+)([A-Z][a-z])/, '\1_\2')
     .gsub(/([a-z\d])([A-Z])/, '\1_\2')
     .downcase
end

puts to_snake_case("HelloWorld")       # => hello_world
puts to_snake_case("CamelCaseString")  # => camel_case_string
puts to_snake_case("HTMLParser")       # => html_parser
```

---

## Step 43: String เพิ่มเติม

### String Methods อื่นๆ ที่มีประโยชน์

```ruby
# replace - แทนที่เนื้อหาทั้งหมดของ string
str = "Hello"
str.replace("Goodbye")
puts str  # => Goodbye (แก้ไข object เดิม)

# ต่างจาก assignment
str2 = "Hello"
str2 = "Goodbye"  # สร้าง object ใหม่
```

```ruby
# each_char - iterate ทีละตัวอักษร
"hello".each_char { |c| print "#{c}." }
# => h.e.l.l.o.
puts

# each_byte - iterate ทีละ byte
"hi".each_byte { |b| print "#{b} " }
# => 104 105
puts

# each_line - iterate ทีละบรรทัด
"line1\nline2\nline3".each_line do |line|
  print line.chomp + " | "
end
# => line1 | line2 | line3 |
puts
```

```ruby
# unpack / pack (binary operations)
# แปลง string เป็น binary data
packed = [65, 66, 67].pack("C*")
puts packed          # => ABC
puts packed.unpack("C*").inspect  # => [65, 66, 67]

# ใช้กับ IP addresses
ip = [192, 168, 1, 1].pack("C4")
puts ip.unpack("C4").join(".")  # => 192.168.1.1
```

```ruby
# Comparable - String ใช้ module Comparable
puts "apple".between?("aardvark", "banana")  # => true
puts "zebra".clamp("a", "m")   # => m (ถ้าเกิน clamp ที่ขอบเขต)

# frozen? - ตรวจสอบว่า frozen หรือไม่
puts "hello".frozen?          # => false (ปกติ)
puts "hello".freeze.frozen?   # => true
puts :hello.to_s.frozen?      # => false

# String interpolation สร้าง string ใหม่เสมอ
str = "frozen".freeze
new_str = "#{str} string"
puts new_str.frozen?  # => false
```

---

## Step 44: String Multiline Patterns

### รูปแบบ String หลายบรรทัด

```ruby
# วิธีที่ 1: ใช้ \ ต่อบรรทัด
long_string = "This is a very long string that " \
              "spans multiple lines in source code " \
              "but is a single line string"
puts long_string

# วิธีที่ 2: ใช้ + ต่อ string
long_string2 = "This is a very long string that " +
               "spans multiple lines in source code"
puts long_string2

# วิธีที่ 3: ใช้ parentheses
long_string3 = (
  "This is a very long string that "
  "spans multiple lines in source code"
)
puts long_string3
```

```ruby
# Heredoc สำหรับข้อความจริงๆ หลายบรรทัด
letter = <<~LETTER
  Dear Customer,

  Thank you for your order. Your items will be shipped within 3-5 business days.

  Order Details:
  - Item 1: Ruby Book
  - Item 2: Keyboard
  
  Total: 1,500 THB

  Best regards,
  The Team
LETTER

puts letter
```

```ruby
# String สำหรับ configuration
config_template = <<~CONFIG
  database:
    host: #{ENV.fetch('DB_HOST', 'localhost')}
    port: #{ENV.fetch('DB_PORT', '5432')}
    name: #{ENV.fetch('DB_NAME', 'myapp_development')}
  
  redis:
    host: #{ENV.fetch('REDIS_HOST', 'localhost')}
    port: #{ENV.fetch('REDIS_PORT', '6379')}
CONFIG

puts config_template
```

---

## Step 45: String Performance Tips

### เคล็ดลับประสิทธิภาพ

```ruby
# 1. ใช้ << แทน + ใน loop
require 'benchmark'

n = 100_000

time1 = Benchmark.realtime do
  str = ""
  n.times { str = str + "x" }
end

time2 = Benchmark.realtime do
  str = ""
  n.times { str << "x" }
end

puts "Plus: #{time1.round(4)} seconds"
puts "Shovel: #{time2.round(4)} seconds"
```

```ruby
# 2. ใช้ String#freeze สำหรับ constant strings
# ไม่ดี: สร้าง string object ใหม่ทุกครั้ง
1000.times do
  status = "active"  # object ใหม่ทุก iteration
end

# ดีกว่า: ใช้ string เดิม
STATUS = "active".freeze
1000.times do
  status = STATUS  # ใช้ object เดิม
end
```

```ruby
# 3. ใช้ Array#join แทนการต่อ string ซ้ำๆ
# ไม่ดี:
parts = []
100.times { |i| parts << "item #{i}" }
result = parts[0]
99.times { |i| result += ", #{parts[i+1]}" }

# ดีกว่า:
parts = 100.times.map { |i| "item #{i}" }
result = parts.join(", ")
```

```ruby
# 4. ใช้ gsub อย่างระมัดระวัง - regex มี overhead
# สำหรับการแทนที่ง่ายๆ ใช้ sub/gsub กับ string แทน regex
text = "hello world hello"
puts text.gsub("hello", "hi")   # เร็วกว่า
puts text.gsub(/hello/, "hi")   # ช้ากว่า (compile regex)

# แต่ถ้าต้องการ pattern ที่ซับซ้อน regex ก็ยังจำเป็น
```

---

## แบบฝึกหัดตอนที่ 3: Strings (20 ข้อ)

### ข้อ 1-5: พื้นฐาน

**ข้อ 1**: เขียนโปรแกรมรับชื่อจากผู้ใช้แล้วแสดงผลใน format ต่างๆ (ตัวพิมพ์ใหญ่, ตัวพิมพ์เล็ก, capitalize)

```ruby
# เฉลย
print "กรุณาใส่ชื่อ: "
name = gets.chomp

puts "ตัวพิมพ์ใหญ่: #{name.upcase}"
puts "ตัวพิมพ์เล็ก: #{name.downcase}"
puts "Capitalize: #{name.capitalize}"
puts "Swap case: #{name.swapcase}"
```

**ข้อ 2**: เขียนฟังก์ชันที่รับ String แล้วตรวจสอบว่าเป็น Palindrome หรือไม่ (ไม่สนใจ space และตัวพิมพ์)

```ruby
# เฉลย
def palindrome?(str)
  # ลบอักขระที่ไม่ใช่ตัวอักษรและตัวเลข แล้วทำเป็นตัวพิมพ์เล็ก
  cleaned = str.downcase.gsub(/[^a-z0-9]/, "")
  cleaned == cleaned.reverse
end

test_cases = [
  "racecar",
  "A man a plan a canal Panama",
  "hello",
  "Was it a car or a cat I saw",
  "Ruby"
]

test_cases.each do |str|
  result = palindrome?(str) ? "ใช่" : "ไม่ใช่"
  puts "'#{str}' #{result} palindrome"
end
```

**ข้อ 3**: เขียนโปรแกรมนับจำนวน vowels (a, e, i, o, u) ใน string

```ruby
# เฉลย
def count_vowels(str)
  str.downcase.count("aeiou")
end

sentences = [
  "Hello World",
  "Ruby is great",
  "The quick brown fox"
]

sentences.each do |s|
  puts "'#{s}' มี #{count_vowels(s)} vowels"
end
```

**ข้อ 4**: เขียนฟังก์ชันแปลง snake_case เป็น CamelCase

```ruby
# เฉลย
def snake_to_camel(str)
  str.split("_").map(&:capitalize).join
end

def snake_to_lower_camel(str)
  parts = str.split("_")
  parts[0] + parts[1..].map(&:capitalize).join
end

puts snake_to_camel("hello_world")          # => HelloWorld
puts snake_to_camel("my_variable_name")     # => MyVariableName
puts snake_to_lower_camel("hello_world")    # => helloWorld
puts snake_to_lower_camel("my_var_name")    # => myVarName
```

**ข้อ 5**: เขียนโปรแกรม word frequency counter

```ruby
# เฉลย
def word_frequency(text)
  words = text.downcase.gsub(/[^a-z\s]/, "").split
  frequency = Hash.new(0)
  words.each { |word| frequency[word] += 1 }
  frequency.sort_by { |_, count| -count }
end

text = "the quick brown fox jumps over the lazy dog the fox"
results = word_frequency(text)

puts "Word Frequency:"
results.each do |word, count|
  puts "  '#{word}': #{count} ครั้ง"
end
```

### ข้อ 6-10: String Methods

**ข้อ 6**: เขียนฟังก์ชัน truncate ที่ตัด string ให้สั้นลงถ้ายาวเกินไป

```ruby
# เฉลย
def truncate(str, max_length = 30, ellipsis = "...")
  return str if str.length <= max_length
  str[0, max_length - ellipsis.length] + ellipsis
end

long_text = "This is a very long text that needs to be truncated"
puts truncate(long_text)           # => This is a very long text tha...
puts truncate(long_text, 20)       # => This is a very lo...
puts truncate("Short text")        # => Short text (ไม่ตัด)
puts truncate(long_text, 20, " →") # => This is a very  →
```

**ข้อ 7**: เขียนฟังก์ชัน mask สำหรับ credit card number

```ruby
# เฉลย
def mask_credit_card(number)
  # ลบ space และ dash
  digits = number.gsub(/[\s-]/, "")
  
  # แสดงเฉพาะ 4 ตัวสุดท้าย
  masked = "*" * (digits.length - 4) + digits[-4..]
  
  # จัดรูปแบบเป็นกลุ่มๆ ละ 4
  masked.chars.each_slice(4).map(&:join).join("-")
end

cards = ["4111111111111111", "5500 0000 0000 0004", "3782-822463-10005"]
cards.each do |card|
  puts "#{card} => #{mask_credit_card(card)}"
end
```

**ข้อ 8**: เขียนฟังก์ชัน count_words ที่นับจำนวนคำใน string

```ruby
# เฉลย
def count_words(text)
  text.strip.split(/\s+/).reject(&:empty?).length
end

texts = [
  "Hello World",
  "  spaces   between   words  ",
  "",
  "one",
  "The quick brown fox jumps"
]

texts.each do |t|
  puts "'#{t}' มี #{count_words(t)} คำ"
end
```

**ข้อ 9**: เขียนฟังก์ชัน email validator อย่างง่าย

```ruby
# เฉลย
def valid_email?(email)
  # Pattern: local@domain.tld
  pattern = /\A[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\z/
  !email.nil? && !email.empty? && email.match?(pattern)
end

emails = [
  "user@example.com",
  "user.name+tag@example.co.th",
  "invalid-email",
  "@no-local.com",
  "no-domain@",
  "valid@test.org"
]

emails.each do |email|
  status = valid_email?(email) ? "ถูกต้อง" : "ไม่ถูกต้อง"
  puts "#{email}: #{status}"
end
```

**ข้อ 10**: เขียนฟังก์ชันสร้าง slug จาก title

```ruby
# เฉลย
def slugify(title)
  title
    .downcase
    .gsub(/[^\w\s-]/, "")  # ลบอักขระพิเศษ
    .strip
    .gsub(/[\s_-]+/, "-")  # แทน space ด้วย -
    .gsub(/^-+|-+$/, "")   # ลบ - ที่ขึ้นต้นหรือลงท้าย
end

titles = [
  "Hello World!",
  "Ruby on Rails - Getting Started",
  "  Leading and trailing spaces  ",
  "Multiple   spaces   here"
]

titles.each do |title|
  puts "#{title.inspect} => #{slugify(title)}"
end
```

### ข้อ 11-15: Intermediate

**ข้อ 11**: เขียนโปรแกรม Caesar Cipher (เลื่อนอักษรไปตามจำนวนที่กำหนด)

```ruby
# เฉลย
def caesar_encrypt(text, shift)
  text.chars.map do |char|
    if char =~ /[a-zA-Z]/
      base = char =~ /[A-Z]/ ? "A".ord : "a".ord
      ((char.ord - base + shift) % 26 + base).chr
    else
      char
    end
  end.join
end

def caesar_decrypt(text, shift)
  caesar_encrypt(text, 26 - shift)
end

message = "Hello, World!"
shift = 3

encrypted = caesar_encrypt(message, shift)
decrypted = caesar_decrypt(encrypted, shift)

puts "ต้นฉบับ:  #{message}"
puts "เข้ารหัส: #{encrypted}"
puts "ถอดรหัส: #{decrypted}"
```

**ข้อ 12**: เขียนฟังก์ชัน wrap_text ที่ตัดบรรทัดไม่ให้ยาวเกินที่กำหนด

```ruby
# เฉลย
def wrap_text(text, max_width = 80)
  words = text.split(" ")
  lines = []
  current_line = ""
  
  words.each do |word|
    if current_line.empty?
      current_line = word
    elsif (current_line + " " + word).length <= max_width
      current_line += " " + word
    else
      lines << current_line
      current_line = word
    end
  end
  
  lines << current_line unless current_line.empty?
  lines.join("\n")
end

long_text = "This is a long text that should be wrapped at a specific width to make it more readable on narrow screens or terminals"
puts wrap_text(long_text, 40)
```

**ข้อ 13**: เขียนฟังก์ชัน parse_query_string สำหรับ URL

```ruby
# เฉลย
def parse_query_string(query)
  return {} if query.nil? || query.empty?
  
  # ลบ ? ถ้ามี
  query = query.sub(/^\?/, "")
  
  query.split("&").each_with_object({}) do |pair, hash|
    key, value = pair.split("=", 2)
    # URL decode (อย่างง่าย)
    key = key.gsub("%20", " ").gsub("+", " ") if key
    value = value.gsub("%20", " ").gsub("+", " ") if value
    hash[key] = value if key
  end
end

queries = [
  "name=John&age=30&city=Bangkok",
  "?search=ruby+on+rails&page=2",
  "empty=&key=value",
  ""
]

queries.each do |q|
  puts "'#{q}' =>"
  p parse_query_string(q)
  puts
end
```

**ข้อ 14**: เขียนโปรแกรมแสดง string สวยงาม (pretty print string with visible whitespace)

```ruby
# เฉลย
def inspect_whitespace(str)
  str
    .gsub(" ", "·")   # space เป็น ·
    .gsub("\t", "→")  # tab เป็น →
    .gsub("\n", "↵\n")  # newline เป็น ↵ แล้วขึ้นบรรทัดจริง
    .gsub("\r", "←")  # carriage return
end

examples = [
  "Hello World",
  "Tab\there",
  "Line 1\nLine 2\nLine 3",
  "  leading spaces",
  "trailing spaces  "
]

examples.each do |s|
  puts "Input:  #{s.inspect}"
  puts "Visual: #{inspect_whitespace(s)}"
  puts
end
```

**ข้อ 15**: เขียนฟังก์ชัน highlight ที่เน้นคำใน string

```ruby
# เฉลย
def highlight(text, keyword, open_tag = "**", close_tag = "**")
  text.gsub(keyword, "#{open_tag}#{keyword}#{close_tag}")
end

def highlight_case_insensitive(text, keyword, open_tag = "**", close_tag = "**")
  text.gsub(/#{Regexp.escape(keyword)}/i) do |match|
    "#{open_tag}#{match}#{close_tag}"
  end
end

text = "Ruby is a great language. I love Ruby programming."
puts highlight(text, "Ruby")
puts highlight(text, "Ruby", "[", "]")
puts highlight_case_insensitive("Hello hello HELLO", "hello", ">>>", "<<<")
```

### ข้อ 16-20: Advanced

**ข้อ 16**: เขียนฟังก์ชัน levenshtein_distance (ระยะห่างระหว่าง 2 strings)

```ruby
# เฉลย - Levenshtein Distance Algorithm
def levenshtein_distance(s1, s2)
  m, n = s1.length, s2.length
  
  # สร้าง matrix
  dp = Array.new(m + 1) { Array.new(n + 1, 0) }
  
  # Initialize
  (0..m).each { |i| dp[i][0] = i }
  (0..n).each { |j| dp[0][j] = j }
  
  # Fill matrix
  (1..m).each do |i|
    (1..n).each do |j|
      if s1[i-1] == s2[j-1]
        dp[i][j] = dp[i-1][j-1]
      else
        dp[i][j] = 1 + [dp[i-1][j], dp[i][j-1], dp[i-1][j-1]].min
      end
    end
  end
  
  dp[m][n]
end

pairs = [
  ["kitten", "sitting"],
  ["Saturday", "Sunday"],
  ["hello", "hello"],
  ["ruby", "rudy"]
]

pairs.each do |s1, s2|
  dist = levenshtein_distance(s1, s2)
  puts "'#{s1}' -> '#{s2}': distance = #{dist}"
end
```

**ข้อ 17**: เขียนโปรแกรม simple template engine

```ruby
# เฉลย
def render_template(template, variables = {})
  template.gsub(/\{\{(\w+)\}\}/) do |match|
    key = $1.to_sym
    variables.fetch(key, match)  # ถ้าไม่พบ ใช้ match เดิม
  end
end

template = <<~TEMPLATE
  สวัสดีคุณ {{name}},
  
  คุณสมัครสมาชิกเมื่อ {{date}} ด้วย email {{email}}
  Package ของคุณ: {{package}}
  
  ขอบคุณที่ใช้บริการ
TEMPLATE

data = {
  name: "สมชาย ใจดี",
  date: "15 มีนาคม 2567",
  email: "somchai@example.com",
  package: "Gold"
}

puts render_template(template, data)
```

**ข้อ 18**: เขียนฟังก์ชัน format_thai_phone สำหรับเบอร์โทรศัพท์ไทย

```ruby
# เฉลย
def format_thai_phone(phone)
  # ลบอักขระที่ไม่ใช่ตัวเลข
  digits = phone.gsub(/\D/, "")
  
  case digits.length
  when 10  # มือถือหรือโทรศัพท์พื้นฐาน 10 หลัก
    if digits.start_with?("0")
      if digits[1] == "2"  # กรุงเทพ 02-xxx-xxxx
        "#{digits[0,2]}-#{digits[2,3]}-#{digits[5,4]}"
      else  # มือถือ 0xx-xxx-xxxx
        "#{digits[0,3]}-#{digits[3,3]}-#{digits[6,4]}"
      end
    end
  when 9   # โทรศัพท์บ้าน 9 หลัก
    "#{digits[0,2]}-#{digits[2,3]}-#{digits[5,4]}"
  else
    phone  # ไม่รู้จัก format คืนค่าเดิม
  end
end

phones = ["0812345678", "0234567890", "021234567", "08 1234 5678", "081-234-5678"]
phones.each do |phone|
  puts "#{phone.inspect.ljust(20)} => #{format_thai_phone(phone)}"
end
```

**ข้อ 19**: เขียน simple JSON serializer สำหรับ Hash และ Array

```ruby
# เฉลย
def to_json_simple(obj, indent = 0)
  spaces = "  " * indent
  inner_spaces = "  " * (indent + 1)
  
  case obj
  when Hash
    if obj.empty?
      "{}"
    else
      pairs = obj.map do |k, v|
        "#{inner_spaces}#{k.to_s.inspect}: #{to_json_simple(v, indent + 1)}"
      end
      "{\n#{pairs.join(",\n")}\n#{spaces}}"
    end
  when Array
    if obj.empty?
      "[]"
    else
      items = obj.map { |v| "#{inner_spaces}#{to_json_simple(v, indent + 1)}" }
      "[\n#{items.join(",\n")}\n#{spaces}]"
    end
  when String
    obj.inspect
  when NilClass
    "null"
  when TrueClass, FalseClass
    obj.to_s
  when Numeric
    obj.to_s
  else
    obj.to_s.inspect
  end
end

data = {
  name: "สมชาย",
  age: 25,
  active: true,
  address: nil,
  hobbies: ["อ่านหนังสือ", "เขียนโปรแกรม"],
  scores: { math: 90, english: 85 }
}

puts to_json_simple(data)
```

**ข้อ 20**: เขียนโปรแกรม text statistics analyzer

```ruby
# เฉลย
def analyze_text(text)
  # ทำความสะอาด text
  clean_text = text.strip
  
  # นับประโยค (จบด้วย . ! ?)
  sentences = clean_text.scan(/[.!?]+/).length
  sentences = 1 if sentences == 0 && !clean_text.empty?
  
  # แยกคำ
  words = clean_text.downcase.scan(/\b[a-zA-Zก-๙]+\b/)
  
  # นับตัวอักษร (ไม่นับ space)
  chars_no_space = clean_text.gsub(/\s/, "").length
  
  # หาคำที่ใช้บ่อย
  word_freq = Hash.new(0)
  words.each { |w| word_freq[w] += 1 }
  top_words = word_freq.sort_by { |_, v| -v }.first(5)
  
  # ประมาณเวลาอ่าน (เฉลี่ย 200 คำ/นาที)
  reading_time = (words.length / 200.0).ceil
  
  {
    characters: clean_text.length,
    characters_no_spaces: chars_no_space,
    words: words.length,
    sentences: sentences,
    paragraphs: clean_text.split(/\n\n+/).length,
    average_word_length: words.empty? ? 0 : (words.sum(&:length).to_f / words.length).round(2),
    average_sentence_length: words.empty? ? 0 : (words.length.to_f / sentences).round(2),
    reading_time_minutes: reading_time,
    top_words: top_words
  }
end

sample_text = <<~TEXT
  Ruby is a dynamic, interpreted, reflective, object-oriented, general-purpose programming language.
  It was designed and developed in the mid-1990s by Yukihiro Matsumoto in Japan.
  
  Ruby embodies the principle of least astonishment, meaning it minimizes confusion for experienced users.
  Ruby is known for its elegant syntax that is natural to read and easy to write.
  
  The language has influenced many other languages and has a vibrant community.
TEXT

stats = analyze_text(sample_text)
puts "=== Text Statistics ==="
puts "จำนวนตัวอักษร: #{stats[:characters]}"
puts "ตัวอักษร (ไม่นับ space): #{stats[:characters_no_spaces]}"
puts "จำนวนคำ: #{stats[:words]}"
puts "จำนวนประโยค: #{stats[:sentences]}"
puts "จำนวนย่อหน้า: #{stats[:paragraphs]}"
puts "ความยาวคำเฉลี่ย: #{stats[:average_word_length]} ตัวอักษร"
puts "ความยาวประโยคเฉลี่ย: #{stats[:average_sentence_length]} คำ"
puts "เวลาอ่านประมาณ: #{stats[:reading_time_minutes]} นาที"
puts "\nคำที่ใช้บ่อยสุด:"
stats[:top_words].each_with_index do |(word, count), i|
  puts "  #{i+1}. '#{word}': #{count} ครั้ง"
end
```

---

## สรุป

ในตอนนี้เราได้เรียนรู้เกี่ยวกับ Ruby Strings อย่างครบถ้วน:

| เรื่อง | สิ่งที่เรียนรู้ |
|--------|----------------|
| การสร้าง String | Single/Double quotes, Heredoc, Multiplication |
| Interpolation | `#{}` syntax, Expression interpolation |
| Escape Sequences | `\n`, `\t`, `\\`, `\"`, Unicode |
| Case Methods | upcase, downcase, capitalize, swapcase |
| Whitespace | strip, lstrip, rstrip, chomp, chop, squeeze |
| การค้นหา | include?, start_with?, end_with?, index, rindex |
| การแทนที่ | sub, gsub, tr, delete |
| การแบ่ง | split, chars, bytes, lines |
| Formatting | center, ljust, rjust, %, sprintf, format |
| Performance | frozen_string_literal, << vs + |
| Comparison | ==, <=>, casecmp |
| Encoding | UTF-8, encode, force_encoding |

ตอนต่อไปจะเป็นเรื่อง **Numbers และ Math** ที่จะเรียนรู้การทำงานกับตัวเลขใน Ruby อย่างละเอียด
