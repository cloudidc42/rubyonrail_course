# ส่วนที่ 3: Strings - ขั้นตอนที่ 26-45

## บทนำ

String (สตริง) เป็นหนึ่งในชนิดข้อมูลที่ใช้บ่อยที่สุดในการเขียนโปรแกรม Ruby มี String methods มากมายที่ช่วยให้การจัดการข้อความเป็นเรื่องง่าย ในส่วนนี้เราจะเรียนรู้ทุกแง่มุมของ String ใน Ruby อย่างละเอียด

---

## ขั้นตอนที่ 26: String Creation - การสร้าง String

### Single Quotes vs Double Quotes

```ruby
# Single quotes - ตรงไปตรงมา ไม่แปลง escape sequences
name = 'Alice'
greeting = 'Hello, World!'
path = 'C:\Users\Alice'  # \ ไม่ต้อง escape

# ข้อยกเว้น: ต้อง escape ' และ \ ใน single quote
puts 'It\'s a beautiful day'   # => It's a beautiful day
puts 'C:\\Users\\Alice'        # => C:\Users\Alice

# Double quotes - รองรับ escape sequences และ interpolation
full_name = "Alice Smith"
newline_example = "Line 1\nLine 2"     # \n = newline
tab_example = "Column1\tColumn2"       # \t = tab
bell = "\a"                             # \a = bell sound
escape = "Say \"Hello\""              # \" = double quote

# Escape sequences ทั้งหมดใน double quotes
puts "\n"   # newline
puts "\t"   # tab
puts "\r"   # carriage return
puts "\\"   # backslash
puts "\""   # double quote
puts "\a"   # bell
puts "\b"   # backspace
puts "\f"   # form feed
puts "\v"   # vertical tab
puts "\e"   # escape
puts "\s"   # space
puts "\0"   # null

# Unicode
puts "\u0041"     # => A (Unicode character)
puts "\u0E2A"     # => ส (Thai character)
puts "\u{1F600}"  # => 😀 (Emoji)

# String interpolation - เฉพาะ double quotes
user = "Ruby"
version = 3.3
puts "Hello #{user} #{version}!"  # => Hello Ruby 3.3!
puts 'Hello #{user} #{version}!'  # => Hello #{user} #{version}! (ไม่แปล!)

# %q และ %Q
str1 = %q(Single quote style - no interpolation #{1+1})
str2 = %Q(Double quote style - with interpolation #{1+1})
str3 = %(Same as %Q - #{1+1})

puts str1  # => Single quote style - no interpolation #{1+1}
puts str2  # => Double quote style - with interpolation 2
puts str3  # => Same as %Q - 2

# เหตุผลที่ใช้ %q/%Q: เมื่อ string มีทั้ง ' และ "
quote = %q("It's wonderful," she said.)
puts quote  # => "It's wonderful," she said.
```

### String Constructor

```ruby
# String.new
s1 = String.new("hello")
s2 = String.new  # empty string

puts s1           # => hello
puts s2.empty?    # => true
puts s2.inspect   # => ""

# String method
s3 = String("42")   # เหมือน "42".to_s
puts s3.class        # => String
```

---

## ขั้นตอนที่ 27: String Methods พื้นฐาน

### Methods เกี่ยวกับ Case

```ruby
str = "Hello, Ruby World!"

# Case transformation
puts str.upcase      # => HELLO, RUBY WORLD!
puts str.downcase    # => hello, ruby world!
puts str.capitalize  # => Hello, ruby world! (พิมพ์ใหญ่เฉพาะตัวแรก)
puts str.swapcase    # => hELLO, rUBY wORLD! (สลับ case)

# Bang methods (แก้ไข in-place)
str2 = "hello world"
str2.upcase!
puts str2   # => HELLO WORLD (แก้ไขตัวแปรเดิม)

# Methods ที่ได้ใน Rails (activesupport)
# "hello_world".camelize    # => HelloWorld
# "HelloWorld".underscore   # => hello_world
# "hello world".titleize    # => Hello World

# titlecase ใน pure Ruby
def titlecase(str)
  str.split(' ').map(&:capitalize).join(' ')
end

puts titlecase("hello ruby world")  # => Hello Ruby World
```

### Methods เกี่ยวกับ Length และ Size

```ruby
str = "Hello, สวัสดี!"

puts str.length       # => 14 (characters)
puts str.size         # => 14 (เหมือน length)
puts str.bytesize     # => 32 (bytes - UTF-8 Thai = 3 bytes/char)
puts str.empty?       # => false
puts "".empty?        # => true
puts "   ".empty?     # => false (whitespace ไม่นับว่า empty)
puts "   ".strip.empty?  # => true

# count characters
puts "hello".count("l")     # => 2 (นับตัว 'l')
puts "hello".count("aeiou") # => 2 (นับสระ)
puts "hello world".count("lo")  # => 5 (นับ l และ o)
```

### Methods เกี่ยวกับ Searching

```ruby
str = "Hello, Ruby World! Ruby is great."

# include? - ตรวจสอบว่ามีหรือไม่
puts str.include?("Ruby")    # => true
puts str.include?("Python")  # => false

# start_with? / end_with?
puts str.start_with?("Hello")   # => true
puts str.start_with?("Hi", "Hello", "Hey")  # => true (any match)
puts str.end_with?("great.")     # => true

# index / rindex - หา position
puts str.index("Ruby")     # => 7 (ตำแหน่งแรก)
puts str.rindex("Ruby")    # => 19 (ตำแหน่งสุดท้าย)
puts str.index("Python")   # => nil (ไม่พบ)

# การใช้ index เพื่อ substring
puts str[0, 5]       # => "Hello" (start, length)
puts str[7, 4]       # => "Ruby"
puts str[7..10]      # => "Ruby" (range)
puts str[-5..-1]     # => "eat." (จากท้าย)
puts str[-5, 5]      # => "reat." (จากท้าย, length)

# scan - หาทุก occurrence
puts str.scan("Ruby").inspect     # => ["Ruby", "Ruby"]
puts "abc123def456".scan(/\d+/).inspect  # => ["123", "456"]

# match - regex matching
if m = str.match(/(\w+) is (\w+)/)
  puts m[0]  # => "Ruby is great"
  puts m[1]  # => "Ruby"
  puts m[2]  # => "great"
end
```

---

## ขั้นตอนที่ 28: String Interpolation ขั้นสูง

```ruby
# Basic interpolation
name = "Alice"
age = 25
puts "ชื่อ: #{name}, อายุ: #{age}"

# Expression ใน interpolation
puts "2 + 2 = #{2 + 2}"
puts "Upper: #{"hello".upcase}"
puts "Array sum: #{[1,2,3].sum}"

# Multi-line expression
result = "ผล: #{
  numbers = [1, 2, 3, 4, 5]
  numbers.select(&:odd?).sum
}"
puts result  # => ผล: 9

# Nested interpolation
greeting = "Hello, #{
  first = "Ruby"
  last = "World"
  "#{first} #{last}"
}!"
puts greeting  # => Hello, Ruby World!

# การจัดรูปแบบใน interpolation
price = 1234567.89
puts "ราคา: #{format('%.2f', price)}"
puts "ราคา: #{sprintf('%,.2f', price)}"  # ไม่มี comma ใน sprintf ปกติ

# Format numbers ด้วย custom method
def format_number(n)
  n.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse
end

puts "ราคา: #{format_number(price.to_i)} บาท"

# String multiplication
puts "=" * 30
puts "-" * 30

# Building strings efficiently
parts = []
parts << "Hello"
parts << "World"
puts parts.join(", ")

# สร้าง String จาก Array
puts ["Alice", "Bob", "Charlie"].join(" and ")
# => Alice and Bob and Charlie
```

---

## ขั้นตอนที่ 29: String Formatting

### printf / sprintf / format

```ruby
name = "Alice"
age = 25
score = 98.765
balance = 1234567.89

# %s - String
printf("Name: %s\n", name)
# Name: Alice

# %d - Integer
printf("Age: %d\n", age)
# Age: 25

# %f - Float
printf("Score: %f\n", score)
# Score: 98.765000

# %.2f - Float with 2 decimal places
printf("Score: %.2f\n", score)
# Score: 98.77

# %e - Scientific notation
printf("Balance: %e\n", balance)
# Balance: 1.234568e+06

# %-20s - Left align with width 20
printf("%-20s %5d %8.2f\n", name, age, score)
# Alice                   25    98.77

# %05d - Zero-padded
printf("ID: %05d\n", 42)
# ID: 00042

# %+d - Show sign
printf("Temperature: %+d°C\n", -5)
# Temperature: -5°C
printf("Temperature: %+d°C\n", 25)
# Temperature: +25°C

# sprintf/format returns string
msg = sprintf("Hello, %s! You scored %.1f%%", name, score)
puts msg  # => Hello, Alice! You scored 98.8%

msg2 = format("Balance: $%,.2f", balance)
puts msg2  # => Balance: $1234567.89 (format ไม่มี comma ใน Ruby stdlib)

# % operator
puts "Hello, %s!" % name
puts "Score: %.2f%%" % score
puts "%s is %d years old" % [name, age]

# การ format table
headers = ["ชื่อ", "อายุ", "คะแนน"]
data = [
  ["Alice", 25, 98.5],
  ["Bob", 30, 87.3],
  ["Charlie", 22, 92.1]
]

printf("%-15s %5s %8s\n", *headers)
puts "-" * 30
data.each do |row|
  printf("%-15s %5d %8.1f\n", *row)
end
```

---

## ขั้นตอนที่ 30: Multiline Strings และ Heredoc

### Heredoc

```ruby
# Basic heredoc (<<IDENTIFIER)
message = <<HEREDOC
สวัสดีครับ!
นี่คือ multiline string
ที่เก็บไว้ใน heredoc
HEREDOC

puts message

# Squiggly heredoc (<<~) - ลบ indentation
sql = <<~SQL
  SELECT users.name, orders.total
  FROM users
  JOIN orders ON users.id = orders.user_id
  WHERE users.active = true
  ORDER BY orders.total DESC
SQL

puts sql

# Interpolation ใน heredoc (default)
user = "Alice"
count = 5
greeting = <<~MSG
  สวัสดี #{user}!
  คุณมี #{count} ข้อความใหม่
  #{"-" * 30}
  วันที่: #{Time.now.strftime("%d/%m/%Y")}
MSG

puts greeting

# Heredoc ไม่มี interpolation (ใส่ quote รอบ identifier)
raw = <<~'CODE'
  x = #{variable}
  puts "No interpolation here"
CODE
puts raw  # แสดงตรงๆ ไม่แปล

# Heredoc สำหรับ methods
def generate_html(title, body)
  <<~HTML
    <!DOCTYPE html>
    <html>
      <head>
        <title>#{title}</title>
      </head>
      <body>
        <h1>#{title}</h1>
        <p>#{body}</p>
      </body>
    </html>
  HTML
end

puts generate_html("My Page", "Welcome to Ruby!")

# Heredoc inline
puts <<~TEXT.upcase
  hello world
TEXT
# => HELLO WORLD

# Multiple heredocs (Ruby 2.3+)
a, b = <<~A, <<~B
  First string
A
  Second string
B
puts a  # => First string
puts b  # => Second string
```

### Multiline Strings ด้วยวิธีอื่น

```ruby
# การต่อ String ข้ามบรรทัด (\ ท้ายบรรทัด)
long_string = "This is a very long string that " \
              "spans multiple lines in code " \
              "but is one string in memory"
puts long_string

# การต่อ String ด้วย +
another = "Hello " +
          "World " +
          "from Ruby!"
puts another

# Array join
lines = [
  "Line 1",
  "Line 2",
  "Line 3"
]
puts lines.join("\n")
```

---

## ขั้นตอนที่ 31: String Comparison

```ruby
# == เปรียบเทียบเนื้อหา
puts "hello" == "hello"   # => true
puts "hello" == "Hello"   # => false (case sensitive)

# eql? - เหมือน ==
puts "hello".eql?("hello")  # => true

# equal? - เปรียบเทียบ object identity
s1 = "hello"
s2 = "hello"
s3 = s1

puts s1.equal?(s2)  # => false (คนละ object)
puts s1.equal?(s3)  # => true (object เดียวกัน)
puts s1.object_id == s2.object_id  # => false

# <=> Spaceship operator
puts "apple" <=> "apple"   # => 0 (เท่ากัน)
puts "apple" <=> "banana"  # => -1 (น้อยกว่า)
puts "banana" <=> "apple"  # => 1 (มากกว่า)

# Comparable methods (String รวม Comparable)
puts "apple" < "banana"    # => true
puts "apple" <= "apple"    # => true
puts "cherry" > "banana"   # => true
puts "apple".between?("aaa", "zzz")  # => true

# Case-insensitive comparison
puts "Hello".casecmp("hello")   # => 0 (เท่ากัน)
puts "Hello".casecmp?("hello")  # => true (Ruby 2.4+)

# Sorting strings
words = ["banana", "apple", "cherry", "date"]
puts words.sort.inspect
# => ["apple", "banana", "cherry", "date"]

puts words.sort_by { |w| w.length }.inspect
# => ["date", "apple", "banana", "cherry"]

# Sort Thai strings
thai_words = ["สมชาย", "กมล", "อรุณ", "ชาลี"]
puts thai_words.sort.inspect
```

---

## ขั้นตอนที่ 32: String Manipulation - gsub, sub, split, join

### gsub และ sub

```ruby
str = "Hello, Ruby! Ruby is great!"

# sub - แทนที่แค่ครั้งแรก
puts str.sub("Ruby", "Python")
# => Hello, Python! Ruby is great!

# gsub - แทนที่ทั้งหมด
puts str.gsub("Ruby", "Python")
# => Hello, Python! Python is great!

# sub/gsub กับ Regex
puts "hello world".gsub(/[aeiou]/, "*")
# => h*ll* w*rld

# gsub กับ block
puts "hello world".gsub(/\w+/) { |word| word.capitalize }
# => Hello World

# gsub กับ hash
puts "hello world".gsub(/\w+/, "hello" => "goodbye", "world" => "earth")
# => goodbye earth

# Bang versions (in-place)
text = "Hello, World!"
text.sub!("Hello", "Hi")
puts text  # => Hi, World!

text.gsub!(/[aeiou]/, "")
puts text  # => H, Wrld!

# Practical examples
# ลบ HTML tags
html = "<p>Hello <b>World</b>!</p>"
puts html.gsub(/<[^>]+>/, "")  # => Hello World!

# แทนที่หลายอย่างพร้อมกัน
corrections = {
  "teh" => "the",
  "recieve" => "receive",
  "definately" => "definitely"
}
text2 = "I teh definately need to recieve this"
result = text2.gsub(/\b(#{corrections.keys.join('|')})\b/, corrections)
puts result  # => I the definitely need to receive this

# tr - transliterate (เปลี่ยน character ต่อ character)
puts "hello".tr('aeiou', '*')        # => h*ll*
puts "hello".tr('a-y', 'b-z')       # => ifmmp (shift +1)
puts "Hello World".tr('A-Z', 'a-z') # => hello world (เหมือน downcase)
puts "hello".tr_s('l', 'r')         # => hero (tr + squeeze)

# delete - ลบ characters
puts "hello world".delete('aeiou')  # => hll wrld
puts "hello 123 world".delete('0-9')  # => hello  world
```

### split และ join

```ruby
# split - แบ่ง string เป็น array
sentence = "Hello World Ruby Programming"

puts sentence.split.inspect
# => ["Hello", "World", "Ruby", "Programming"]

puts "a,b,c,d".split(",").inspect
# => ["a", "b", "c", "d"]

puts "a,,b,,c".split(",").inspect
# => ["a", "", "b", "", "c"]

puts "a,,b,,c".split(",", -1).inspect
# => ["a", "", "b", "", "c"] (เก็บ empty strings ท้าย)

puts "a,,b,,c".split(/,+/).inspect
# => ["a", "b", "c"] (regex: หนึ่งหรือมากกว่า comma)

# split กับ limit
puts "a,b,c,d,e".split(",", 3).inspect
# => ["a", "b", "c,d,e"] (แบ่งแค่ 2 ครั้ง)

# split ตัวอักษร
puts "hello".split("").inspect
# => ["h", "e", "l", "l", "o"]

puts "hello".chars.inspect  # เหมือนกัน
# => ["h", "e", "l", "l", "o"]

# join - รวม array เป็น string
arr = ["Hello", "World", "Ruby"]
puts arr.join        # => HelloWorldRuby
puts arr.join(" ")   # => Hello World Ruby
puts arr.join(", ")  # => Hello, World, Ruby
puts arr.join(" | ") # => Hello | World | Ruby

# Practical: CSV parsing
csv_line = "Alice,25,alice@example.com,Bangkok"
fields = csv_line.split(",")
puts "Name: #{fields[0]}"
puts "Age: #{fields[1]}"
puts "Email: #{fields[2]}"
puts "City: #{fields[3]}"

# Practical: Word count
text = "the quick brown fox jumps over the lazy dog"
word_count = text.split.each_with_object(Hash.new(0)) do |word, counts|
  counts[word] += 1
end
puts word_count.sort_by { |_, count| -count }.first(5).to_h.inspect
```

---

## ขั้นตอนที่ 33: String Methods เพิ่มเติม

### Trimming และ Padding

```ruby
# strip, lstrip, rstrip
puts "  hello  ".strip    # => "hello" (ลบทั้งสองข้าง)
puts "  hello  ".lstrip   # => "hello  " (ลบซ้าย)
puts "  hello  ".rstrip   # => "  hello" (ลบขวา)

# chomp - ลบ newline ท้าย
puts "hello\n".chomp      # => "hello"
puts "hello\r\n".chomp    # => "hello"
puts "hello".chomp        # => "hello" (ไม่มีอะไรลบ)
puts "hello\n\n".chomp    # => "hello\n" (ลบแค่ครั้งเดียว)

# chop - ลบตัวอักษรสุดท้าย
puts "hello".chop         # => "hell"
puts "hello\n".chop       # => "hello"

# squeeze - บีบ repeated characters
puts "aaabbbccc".squeeze  # => "abc"
puts "  hello   world  ".squeeze(" ")  # => " hello world "

# center, ljust, rjust - alignment
puts "hello".center(20)         # => "       hello        "
puts "hello".center(20, "-")    # => "-------hello--------"
puts "hello".ljust(20)          # => "hello               "
puts "hello".ljust(20, ".")     # => "hello..............."
puts "hello".rjust(20)          # => "               hello"
puts "hello".rjust(20, "0")     # => "000000000000000hello"
puts "42".rjust(5, "0")         # => "00042" (zero padding)

# insert
str = "Hello World"
puts str.insert(5, ",")   # => "Hello, World" (แทรกที่ position 5)
puts str.insert(-1, "!")  # => "Hello, World!" (แทรกท้าย)

# prepend (เพิ่มหน้า) - แก้ไข in-place!
str2 = "World"
str2.prepend("Hello, ")
puts str2  # => "Hello, World"

# concat / <<
str3 = "Hello"
str3 << " World"  # เร็วกว่า += เพราะแก้ in-place
puts str3  # => "Hello World"

str3.concat("!", " Ruby!")
puts str3  # => "Hello World! Ruby!"
```

### Slicing และ Substring

```ruby
str = "Hello, Ruby World!"

# [] - access by index
puts str[0]       # => "H"
puts str[-1]      # => "!"
puts str[0, 5]    # => "Hello" (index, length)
puts str[7, 4]    # => "Ruby"
puts str[7..10]   # => "Ruby" (range)
puts str[7...11]  # => "Ruby" (exclusive range)

# slice - เหมือน []
puts str.slice(0, 5)    # => "Hello"
puts str.slice(/\w+/)   # => "Hello" (regex)

# first/last characters
puts str[0]      # first char
puts str[-1]     # last char

# substring check
puts str[7..10] == "Ruby"  # => true

# Getting parts
puts str.chars.first(5).join  # => "Hello"
puts str.chars.last(6).join   # => "orld!"

# each_char
str.each_char do |char|
  print char if char =~ /[A-Z]/
end
puts  # => HRW
```

---

## ขั้นตอนที่ 34: Regular Expressions กับ Strings

### Ruby Regex Basics

```ruby
# =~ operator - หา match
if "hello world" =~ /world/
  puts "Found!"  # => Found!
  puts $~.inspect   # MatchData
  puts $~.begin(0)  # position ที่เริ่ม
end

# match method - คืน MatchData หรือ nil
if m = "John Doe, 25".match(/(\w+) (\w+), (\d+)/)
  puts "Full name: #{m[1]} #{m[2]}"  # => Full name: John Doe
  puts "Age: #{m[3]}"                 # => Age: 25
  puts "Named: #{m.named_captures}"
end

# Named captures
if m = "2024-01-15".match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/)
  puts "Year: #{m[:year]}"    # => Year: 2024
  puts "Month: #{m[:month]}"  # => Month: 01
  puts "Day: #{m[:day]}"      # => Day: 15
end

# match? - คืน true/false (เร็วกว่า match)
puts "hello".match?(/ell/)   # => true
puts "hello".match?(/xyz/)   # => false

# Regex patterns ที่ใช้บ่อย
email_regex = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
url_regex = /\Ahttps?:\/\//
phone_regex = /\A0[0-9]{8,9}\z/
thai_phone = /\A(0[689]\d{8}|0[2-9]\d{7})\z/

# ตัวอย่าง validation
def valid_email?(email)
  !!(email =~ /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
end

puts valid_email?("test@example.com")  # => true
puts valid_email?("invalid-email")     # => false

# scan - หาทุก match
text = "Call 086-123-4567 or 02-456-7890 for info"
phones = text.scan(/\d[\d\-]+\d/)
puts phones.inspect  # => ["086-123-4567", "02-456-7890"]

# Extract all emails
emails_text = "Contact alice@a.com or bob@b.com for help"
emails = emails_text.scan(/\b[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\b/i)
puts emails.inspect

# gsub กับ regex และ block
"hello world foo bar".gsub(/\b\w/) { |match| match.upcase }
# => "Hello World Foo Bar"

# split กับ regex
"one1two2three3four".split(/\d/)
# => ["one", "two", "three", "four"]
```

---

## ขั้นตอนที่ 35: String Encoding

```ruby
# Ruby 3.x ใช้ UTF-8 เป็น default

# ตรวจสอบ encoding
str = "Hello"
puts str.encoding          # => UTF-8

thai = "สวัสดีครับ"
puts thai.encoding         # => UTF-8
puts thai.length           # => 10 (characters)
puts thai.bytesize         # => 30 (bytes - 3 bytes/char)

# เปลี่ยน encoding
ascii = "Hello".encode("ASCII")
puts ascii.encoding        # => ASCII

# force_encoding vs encode
binary_str = "\xFF\xFE"
binary_str.force_encoding("UTF-16LE")  # บอก Ruby ว่า encoding คืออะไร (ไม่แปลง)
puts binary_str.encoding   # => UTF-16LE

# encode - แปลง encoding จริงๆ
utf8_str = "Hello สวัสดี"
begin
  ascii_str = utf8_str.encode("ASCII")
rescue Encoding::UndefinedConversionError => e
  puts "Error: #{e.message}"
  # ใช้ option เพื่อแทนที่ตัวที่แปลงไม่ได้
  ascii_str = utf8_str.encode("ASCII", undef: :replace, replace: "?")
  puts ascii_str  # => Hello ????
end

# valid_encoding?
puts "Hello".valid_encoding?  # => true

# scrub - แก้ไข invalid encoding
invalid = "\xFF\xFE Hello"
puts invalid.scrub("?")  # แทนที่ invalid bytes ด้วย ?

# String encoding ที่ต้องระวังกับไฟล์
File.open("thai.txt", "w:UTF-8") do |f|
  f.write("สวัสดีครับ")
end

File.open("thai.txt", "r:UTF-8") do |f|
  content = f.read
  puts content.encoding  # => UTF-8
  puts content           # => สวัสดีครับ
end

# Magic comment ที่บรรทัดแรกของไฟล์
# # encoding: UTF-8
# หรือ
# # -*- coding: UTF-8 -*-
```

---

## ขั้นตอนที่ 36: Frozen String Literal

```ruby
# ปกติ String ใน Ruby เป็น mutable
str = "hello"
str << " world"   # แก้ไขได้
puts str          # => hello world

# frozen_string_literal: true
# ใส่ที่บรรทัดแรกของไฟล์ ทำให้ string literals ทั้งหมดถูก freeze

# หลังจากใส่ magic comment:
# str = "hello"
# str << " world"  # FrozenError!

# ประโยชน์ของ Frozen String Literals:
# 1. ประหยัด memory (string เดียวกันใช้ object เดียว)
# 2. Thread safety
# 3. ป้องกัน mutation bugs

# ตัวอย่าง memory เปรียบเทียบ
n = 100_000

# Mutable strings
mutable_time = Time.now
n.times { "hello".object_id }  # สร้าง string ใหม่ทุกครั้ง
mutable_duration = Time.now - mutable_time

# Frozen strings (ใช้ .freeze)
frozen_time = Time.now
frozen = "hello".freeze
n.times { frozen.object_id }   # ใช้ object เดิม
frozen_duration = Time.now - frozen_time

puts "Mutable: #{mutable_duration.round(4)}s"
puts "Frozen: #{frozen_duration.round(4)}s"

# String pool - Symbol ทำงานแบบนี้เสมอ
puts "hello".freeze.object_id == "hello".freeze.object_id  # อาจ true

# การทำ frozen string ที่ยืดหยุ่น
GREETING = "Hello".freeze  # Constant

def greet(name)
  "#{GREETING}, #{name}!"  # สร้าง string ใหม่จาก frozen
end

puts greet("Alice")  # => Hello, Alice!
```

---

## ขั้นตอนที่ 37: Format Strings

```ruby
# printf format specifiers
printf("%-10s %5d %8.2f\n", "Alice", 25, 98.5)

# สร้าง table
def print_table(headers, rows)
  # คำนวณ widths
  widths = headers.map.with_index do |h, i|
    [h.length, rows.map { |r| r[i].to_s.length }.max].max
  end

  # Header
  header_row = headers.each_with_index.map { |h, i| h.ljust(widths[i]) }.join(" | ")
  separator = widths.map { |w| "-" * w }.join("-+-")

  puts header_row
  puts separator

  # Data rows
  rows.each do |row|
    data_row = row.each_with_index.map { |cell, i| cell.to_s.ljust(widths[i]) }.join(" | ")
    puts data_row
  end
end

headers = ["ชื่อ", "อายุ", "เมือง", "คะแนน"]
data = [
  ["Alice", 25, "กรุงเทพ", 95.5],
  ["Bob", 30, "เชียงใหม่", 87.3],
  ["Charlie", 22, "ภูเก็ต", 92.1],
  ["Diana", 28, "ขอนแก่น", 89.7]
]

print_table(headers, data)

# Number formatting
def format_currency(amount, currency: "฿", decimal_places: 2)
  formatted = sprintf("%.#{decimal_places}f", amount)
  integer_part, decimal_part = formatted.split(".")
  with_commas = integer_part.reverse.gsub(/(\d{3})(?=\d)/, '\1,').reverse
  "#{currency}#{with_commas}.#{decimal_part}"
end

puts format_currency(1234567.89)           # => ฿1,234,567.89
puts format_currency(1234.5, currency: "$")  # => $1,234.50
puts format_currency(42)                   # => ฿42.00
```

---

## ขั้นตอนที่ 38: String และ Enumerable

```ruby
# String เป็น Enumerable ผ่าน each_char, each_line, each_byte
str = "Hello\nWorld\nRuby"

# Iterate ทีละบรรทัด
str.each_line do |line|
  puts line.chomp.upcase
end
# => HELLO
# => WORLD
# => RUBY

# Iterate ทีละ character
"abc".each_char do |char|
  print "#{char}-"
end  # => a-b-c-

# Iterate ทีละ byte
"Hi".each_byte do |byte|
  print "#{byte} "
end  # => 72 105

# lines, chars, bytes - คืน Array
puts "Hello\nWorld".lines.inspect
# => ["Hello\n", "World"]

puts "Hello".chars.inspect
# => ["h", "e", "l", "l", "o"]  (แต่ uppercase เพราะ "Hello")
# แก้: puts "Hello".chars.inspect
# => ["H", "e", "l", "l", "o"]

puts "Hi".bytes.inspect
# => [72, 105]

# ใช้ Enumerable methods บน String
vowels = "Hello World".chars.select { |c| "aeiouAEIOU".include?(c) }
puts vowels.inspect  # => ["e", "o", "o"]

uppercase_count = "Hello Ruby World".count("A-Z")
puts uppercase_count  # => 3

# zip characters
"hello".chars.zip("world".chars).each do |pair|
  puts pair.join(" -> ")
end
```

---

## ขั้นตอนที่ 39: Advanced String Techniques

### String Builder Pattern

```ruby
# สร้าง String ที่ซับซ้อนด้วย StringIO
require 'stringio'

def generate_report(data)
  buffer = StringIO.new

  buffer.puts "=" * 50
  buffer.puts "SALES REPORT - #{Time.now.strftime('%B %Y')}"
  buffer.puts "=" * 50
  buffer.puts

  data.each do |item|
    buffer.printf("%-20s %10s %10s\n",
                  item[:name],
                  "#{item[:qty]} units",
                  "฿#{format('%.2f', item[:price] * item[:qty])}")
  end

  buffer.puts "-" * 50
  total = data.sum { |item| item[:price] * item[:qty] }
  buffer.printf("%-20s %21s\n", "TOTAL:", "฿#{format('%.2f', total)}")

  buffer.string  # คืน string ทั้งหมด
end

sales_data = [
  { name: "Product A", qty: 10, price: 100.0 },
  { name: "Product B", qty: 5,  price: 250.0 },
  { name: "Product C", qty: 20, price: 50.0 }
]

puts generate_report(sales_data)

# Template pattern
class Template
  def initialize(template)
    @template = template
    @variables = {}
  end

  def set(key, value)
    @variables[key] = value
    self  # ให้ chain ได้
  end

  def render
    result = @template.dup
    @variables.each do |key, value|
      result.gsub!("{{#{key}}}", value.to_s)
    end
    result
  end
end

template = Template.new(<<~HTML)
  <h1>{{title}}</h1>
  <p>สวัสดี {{name}}!</p>
  <p>วันที่: {{date}}</p>
HTML

output = template
  .set(:title, "หน้าแรก")
  .set(:name, "Alice")
  .set(:date, "01/01/2024")
  .render

puts output
```

---

## ขั้นตอนที่ 40: String Performance

```ruby
require 'benchmark'

n = 100_000
str = "Hello, World! " * 100

# String concatenation methods
Benchmark.bm(20) do |x|
  # + operator (สร้าง object ใหม่ทุกครั้ง - ช้า)
  x.report("+ operator:") do
    result = ""
    n.times { result = result + "x" }
  end

  # << operator (แก้ไข in-place - เร็วกว่า)
  x.report("<< operator:") do
    result = ""
    n.times { result << "x" }
  end

  # Array join (มักจะเร็วที่สุด)
  x.report("Array join:") do
    parts = []
    n.times { parts << "x" }
    result = parts.join
  end

  # concat method
  x.report("concat:") do
    result = ""
    n.times { result.concat("x") }
  end
end

# String duplication vs creation
Benchmark.bm(20) do |x|
  frozen = "hello world".freeze

  x.report("new string:") do
    n.times { "hello world".upcase }
  end

  x.report("frozen dup:") do
    n.times { frozen.dup.upcase }
  end
end
```

---

## ขั้นตอนที่ 41-45: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: Word Count Program

```ruby
def word_frequency(text)
  words = text.downcase.gsub(/[^a-z\s]/, '').split
  frequency = Hash.new(0)
  words.each { |word| frequency[word] += 1 }
  frequency.sort_by { |_, count| -count }
end

text = "the quick brown fox jumps over the lazy dog the fox"
puts "=== ความถี่คำ ==="
word_frequency(text).first(5).each do |word, count|
  puts "#{word.ljust(15)} #{count} ครั้ง"
end
```

### แบบฝึกหัดที่ 2: Password Strength Checker

```ruby
def password_strength(password)
  score = 0
  feedback = []

  if password.length >= 8
    score += 1
  else
    feedback << "ต้องมีอย่างน้อย 8 ตัวอักษร"
  end

  if password.length >= 12
    score += 1
    feedback << "ยาวดี!"
  end

  if password =~ /[A-Z]/
    score += 1
  else
    feedback << "ควรมีตัวพิมพ์ใหญ่"
  end

  if password =~ /[a-z]/
    score += 1
  else
    feedback << "ควรมีตัวพิมพ์เล็ก"
  end

  if password =~ /\d/
    score += 1
  else
    feedback << "ควรมีตัวเลข"
  end

  if password =~ /[!@#$%^&*(),.?":{}|<>]/
    score += 1
  else
    feedback << "ควรมีตัวอักษรพิเศษ"
  end

  strength = case score
             when 0..2 then "อ่อนแอมาก"
             when 3..4 then "พอใช้"
             when 5    then "ดี"
             when 6    then "แข็งแกร่งมาก"
             end

  { score: score, strength: strength, feedback: feedback }
end

passwords = ["password", "P@ssw0rd!", "Ruby1234!", "R!uby#2024$Secure"]
passwords.each do |pwd|
  result = password_strength(pwd)
  puts "Password: #{pwd.ljust(20)} ความแข็งแกร่ง: #{result[:strength]}"
  result[:feedback].each { |f| puts "  - #{f}" }
  puts
end
```

### แบบฝึกหัดที่ 3: Text Formatter

```ruby
class TextFormatter
  def self.wrap(text, width: 60)
    words = text.split
    lines = []
    current_line = []
    current_length = 0

    words.each do |word|
      if current_length + word.length + (current_line.empty? ? 0 : 1) <= width
        current_line << word
        current_length += word.length + (current_line.length > 1 ? 1 : 0)
      else
        lines << current_line.join(" ")
        current_line = [word]
        current_length = word.length
      end
    end

    lines << current_line.join(" ") unless current_line.empty?
    lines.join("\n")
  end

  def self.center_block(text, width: 60)
    text.lines.map { |line| line.chomp.center(width) }.join("\n")
  end

  def self.box(text, width: nil)
    lines = text.lines.map(&:chomp)
    max_width = width || lines.map(&:length).max
    border = "+" + "-" * (max_width + 2) + "+"

    result = [border]
    lines.each do |line|
      result << "| #{line.ljust(max_width)} |"
    end
    result << border
    result.join("\n")
  end
end

long_text = "Ruby เป็นภาษาโปรแกรมที่ถูกออกแบบมาเพื่อให้โปรแกรมเมอร์มีความสุขในการเขียนโค้ด ด้วยความสามารถที่หลากหลาย"

puts TextFormatter.wrap(long_text, width: 40)
puts
puts TextFormatter.box("Hello\nWorld\nfrom Ruby!")
```

### แบบฝึกหัดที่ 4: Cipher Program

```ruby
class CaesarCipher
  def initialize(shift = 3)
    @shift = shift
  end

  def encrypt(text)
    transform(text, @shift)
  end

  def decrypt(text)
    transform(text, -@shift)
  end

  private

  def transform(text, shift)
    text.chars.map do |char|
      if char =~ /[A-Z]/
        ((char.ord - 65 + shift) % 26 + 65).chr
      elsif char =~ /[a-z]/
        ((char.ord - 97 + shift) % 26 + 97).chr
      else
        char
      end
    end.join
  end
end

cipher = CaesarCipher.new(13)  # ROT13

messages = ["Hello, World!", "Ruby is awesome!", "Secret message"]
messages.each do |msg|
  encrypted = cipher.encrypt(msg)
  decrypted = cipher.decrypt(encrypted)
  puts "Original:  #{msg}"
  puts "Encrypted: #{encrypted}"
  puts "Decrypted: #{decrypted}"
  puts "-" * 40
end
```

### แบบฝึกหัดที่ 5: Template Engine

```ruby
class SimpleTemplate
  VARIABLE_PATTERN = /\{\{(\w+)\}\}/
  LOOP_START = /\{%\s*for\s+(\w+)\s+in\s+(\w+)\s*%\}/
  LOOP_END   = /\{%\s*endfor\s*%\}/
  IF_PATTERN = /\{%\s*if\s+(\w+)\s*%\}/
  ENDIF_PATTERN = /\{%\s*endif\s*%\}/

  def initialize(template)
    @template = template
  end

  def render(context = {})
    result = @template.dup

    # แทนที่ variables
    result.gsub!(VARIABLE_PATTERN) do |match|
      context[$1.to_sym] || context[$1] || ""
    end

    result
  end
end

template = SimpleTemplate.new(<<~TEMPLATE)
  สวัสดี {{name}}!
  อายุ: {{age}} ปี
  Email: {{email}}
TEMPLATE

output = template.render(
  name: "Alice",
  age: 25,
  email: "alice@example.com"
)

puts output
```

### แบบฝึกหัดที่ 6: URL Parser

```ruby
def parse_url(url)
  pattern = /\A
    (?<scheme>[a-z]+):\/\/
    (?:(?<user>[^:@]+)(?::(?<password>[^@]+))?@)?
    (?<host>[^\/\?#:]+)
    (?::(?<port>\d+))?
    (?<path>\/[^?#]*)?
    (?:\?(?<query>[^#]*))?
    (?:#(?<fragment>.*))?
  \z/xi

  m = url.match(pattern)
  return nil unless m

  {
    scheme:   m[:scheme],
    user:     m[:user],
    password: m[:password],
    host:     m[:host],
    port:     m[:port]&.to_i,
    path:     m[:path] || "/",
    query:    parse_query(m[:query]),
    fragment: m[:fragment]
  }
end

def parse_query(query_string)
  return {} unless query_string
  query_string.split("&").each_with_object({}) do |pair, hash|
    key, value = pair.split("=", 2)
    hash[key] = value
  end
end

urls = [
  "https://www.example.com/path/to/page?name=Alice&age=25#section1",
  "http://user:pass@db.example.com:5432/mydb",
  "ftp://files.example.com/downloads/file.zip"
]

urls.each do |url|
  parsed = parse_url(url)
  puts "URL: #{url}"
  parsed.each { |key, value| puts "  #{key}: #{value.inspect}" unless value.nil? || value == {} }
  puts
end
```

### แบบฝึกหัดที่ 7: Markdown to HTML

```ruby
def simple_markdown_to_html(markdown)
  html = markdown.dup

  # Headers
  html.gsub!(/^###### (.+)$/, '<h6>\1</h6>')
  html.gsub!(/^##### (.+)$/,  '<h5>\1</h5>')
  html.gsub!(/^#### (.+)$/,   '<h4>\1</h4>')
  html.gsub!(/^### (.+)$/,    '<h3>\1</h3>')
  html.gsub!(/^## (.+)$/,     '<h2>\1</h2>')
  html.gsub!(/^# (.+)$/,      '<h1>\1</h1>')

  # Bold, Italic
  html.gsub!(/\*\*\*(.+?)\*\*\*/, '<strong><em>\1</em></strong>')
  html.gsub!(/\*\*(.+?)\*\*/,     '<strong>\1</strong>')
  html.gsub!(/\*(.+?)\*/,         '<em>\1</em>')

  # Links
  html.gsub!(/\[(.+?)\]\((.+?)\)/, '<a href="\2">\1</a>')

  # Code
  html.gsub!(/`(.+?)`/, '<code>\1</code>')

  # Paragraphs
  paragraphs = html.split(/\n\n+/)
  paragraphs.map! do |para|
    if para =~ /^<h[1-6]>/ || para.empty?
      para
    else
      "<p>#{para.gsub("\n", "<br>")}</p>"
    end
  end

  paragraphs.join("\n\n")
end

markdown = <<~MD
  # Ruby Programming

  Ruby เป็นภาษาที่ **สวยงาม** และ *ทรงพลัง*

  เรียนรู้ได้ที่ [Ruby Official Site](https://www.ruby-lang.org)

  ใช้คำสั่ง `puts "Hello"` เพื่อแสดงผล

  ## ทำไมต้อง Ruby?

  - **เขียนน้อย** ทำได้มาก
  - *อ่านง่าย* เหมือนภาษาอังกฤษ
MD

puts simple_markdown_to_html(markdown)
```

### แบบฝึกหัดที่ 8: String Statistics

```ruby
def analyze_text(text)
  # การนับพื้นฐาน
  chars = text.length
  words = text.split.length
  sentences = text.scan(/[.!?]+/).length
  paragraphs = text.split(/\n\n+/).length

  # ความถี่ตัวอักษร
  letter_freq = text.downcase.scan(/[a-z]/).each_with_object(Hash.new(0)) do |char, freq|
    freq[char] += 1
  end
  most_common = letter_freq.max_by { |_, count| count }

  # คำที่ไม่ซ้ำ
  unique_words = text.downcase.scan(/\b\w+\b/).uniq.length

  # ความยาวเฉลี่ยของคำ
  word_lengths = text.scan(/\b\w+\b/).map(&:length)
  avg_word_length = word_lengths.sum.to_f / word_lengths.length

  {
    characters: chars,
    characters_no_spaces: text.gsub(/\s/, '').length,
    words: words,
    unique_words: unique_words,
    sentences: sentences,
    paragraphs: paragraphs,
    most_common_letter: most_common[0],
    most_common_count: most_common[1],
    avg_word_length: avg_word_length.round(2)
  }
end

text = <<~TEXT
  Ruby is a dynamic, open source programming language with a focus on simplicity and productivity.
  It has an elegant syntax that is natural to read and easy to write.
  Ruby was created by Yukihiro Matsumoto in the 1990s.
TEXT

stats = analyze_text(text)
puts "=== Text Statistics ==="
stats.each do |key, value|
  puts "#{key.to_s.gsub('_', ' ').capitalize}: #{value}"
end
```

### แบบฝึกหัดที่ 9: Name Formatter

```ruby
class NameFormatter
  def self.format(full_name, style: :standard)
    parts = full_name.strip.split(/\s+/)

    case style
    when :standard
      parts.map(&:capitalize).join(" ")
    when :last_first
      last = parts.last.capitalize
      first = parts[0..-2].map(&:capitalize).join(" ")
      "#{last}, #{first}"
    when :initials
      parts.map { |p| "#{p[0].upcase}." }.join("")
    when :first_last_initial
      first = parts.first.capitalize
      last_initial = "#{parts.last[0].upcase}."
      "#{first} #{last_initial}"
    when :formal
      salutation = parts.length > 2 ? "" : "Mr./Ms. "
      "#{salutation}#{parts.map(&:capitalize).join(' ')}"
    end
  end
end

names = [
  "john doe smith",
  "alice johnson",
  "CHARLIE BROWN"
]

styles = [:standard, :last_first, :initials, :first_last_initial]

names.each do |name|
  puts "Input: \"#{name}\""
  styles.each do |style|
    puts "  #{style}: #{NameFormatter.format(name, style: style)}"
  end
  puts
end
```

### แบบฝึกหัดที่ 10: Multi-line String Builder

```ruby
class HtmlBuilder
  def initialize
    @elements = []
    @indent_level = 0
  end

  def tag(name, content = nil, **attrs, &block)
    attr_str = attrs.map { |k, v| " #{k}=\"#{v}\"" }.join

    if block
      @elements << "#{indent}<#{name}#{attr_str}>"
      @indent_level += 1
      block.call
      @indent_level -= 1
      @elements << "#{indent}</#{name}>"
    elsif content
      @elements << "#{indent}<#{name}#{attr_str}>#{content}</#{name}>"
    else
      @elements << "#{indent}<#{name}#{attr_str}>"
    end

    self
  end

  def text(content)
    @elements << "#{indent}#{content}"
    self
  end

  def to_s
    @elements.join("\n")
  end

  private

  def indent
    "  " * @indent_level
  end
end

# Helper methods
def method_missing(name, *args, **kwargs, &block)
  super
end

builder = HtmlBuilder.new
builder.tag("div", class: "container") do
  builder.tag("h1", "Welcome to Ruby")
  builder.tag("p", "Ruby is a beautiful language")
  builder.tag("ul") do
    ["Fast", "Elegant", "Powerful"].each do |item|
      builder.tag("li", item)
    end
  end
end

puts builder.to_s
```

### แบบฝึกหัดที่ 11: Email Template System

```ruby
class EmailTemplate
  TEMPLATES = {
    welcome: <<~TEMPLATE,
      เรียน คุณ {{name}},

      ยินดีต้อนรับสู่ {{app_name}}!

      บัญชีของคุณถูกสร้างเรียบร้อยแล้วด้วยรายละเอียดต่อไปนี้:
        • Email: {{email}}
        • วันที่สมัคร: {{date}}

      กรุณายืนยัน email ของคุณที่: {{confirm_url}}

      ขอบคุณที่เลือกใช้บริการ,
      ทีมงาน {{app_name}}
    TEMPLATE

    reset_password: <<~TEMPLATE
      เรียน คุณ {{name}},

      เราได้รับคำขอรีเซ็ตรหัสผ่านสำหรับบัญชีของคุณ

      คลิกลิงค์ด้านล่างเพื่อรีเซ็ตรหัสผ่าน:
      {{reset_url}}

      ลิงค์นี้จะหมดอายุใน {{expiry_hours}} ชั่วโมง

      หากคุณไม่ได้ขอรีเซ็ตรหัสผ่าน กรุณาเพิกเฉยต่ออีเมลนี้

      ขอบคุณ,
      ทีมงาน {{app_name}}
    TEMPLATE
  }

  def self.render(template_name, variables = {})
    template = TEMPLATES[template_name] or raise "Template not found: #{template_name}"

    result = template.dup
    variables.each do |key, value|
      result.gsub!("{{#{key}}}", value.to_s)
    end

    # ตรวจสอบว่ายังมี placeholder ที่ยังไม่ถูกแทนที่
    remaining = result.scan(/\{\{\w+\}\}/)
    raise "Missing variables: #{remaining.join(', ')}" unless remaining.empty?

    result
  end
end

email = EmailTemplate.render(:welcome,
  name: "Alice",
  app_name: "RubyApp",
  email: "alice@example.com",
  date: "01/01/2024",
  confirm_url: "https://app.com/confirm/abc123"
)

puts email
```

### แบบฝึกหัดที่ 12: String Compression

```ruby
# Run-length encoding
def compress(str)
  return str if str.empty?

  compressed = ""
  count = 1
  (1...str.length).each do |i|
    if str[i] == str[i-1]
      count += 1
    else
      compressed += count > 1 ? "#{count}#{str[i-1]}" : str[i-1]
      count = 1
    end
  end
  compressed += count > 1 ? "#{count}#{str[-1]}" : str[-1]

  compressed.length < str.length ? compressed : str
end

def decompress(str)
  decompressed = ""
  str.scan(/(\d*)([a-zA-Z])/).each do |count, char|
    decompressed += char * (count.empty? ? 1 : count.to_i)
  end
  decompressed
end

strings = ["aabbbcccc", "abcdef", "aaaaaaaaaa", "aabbaabb"]
strings.each do |s|
  compressed = compress(s)
  puts "Original:   #{s} (#{s.length} chars)"
  puts "Compressed: #{compressed} (#{compressed.length} chars)"
  puts "Decompressed: #{decompress(compressed)}" if compressed != s
  puts
end
```

### แบบฝึกหัดที่ 13: Phone Number Formatter

```ruby
def format_phone(number)
  # ลบตัวอักษรพิเศษทั้งหมด
  digits = number.gsub(/\D/, '')

  case digits.length
  when 10
    if digits.start_with?("08", "09", "06")
      # Mobile: 08X-XXX-XXXX
      "#{digits[0..2]}-#{digits[3..5]}-#{digits[6..9]}"
    else
      # Landline: 0X-XXX-XXXX or 02-XXX-XXXX
      if digits.start_with?("02")
        "#{digits[0..1]}-#{digits[2..5]}-#{digits[6..9]}"
      else
        "#{digits[0..2]}-#{digits[3..5]}-#{digits[6..9]}"
      end
    end
  when 9
    "#{digits[0..1]}-#{digits[2..5]}-#{digits[6..8]}"
  else
    number  # คืนค่าเดิมถ้าไม่รู้จัก format
  end
end

phone_numbers = [
  "0812345678",
  "02-123-4567",
  "089 876 5432",
  "(02)5551234",
  "02.555.6789"
]

phone_numbers.each do |phone|
  puts "#{phone.ljust(20)} → #{format_phone(phone)}"
end
```

### แบบฝึกหัดที่ 14: Anagram Checker

```ruby
def anagram?(word1, word2)
  normalize(word1) == normalize(word2)
end

def normalize(word)
  word.downcase.gsub(/[^a-z]/, '').chars.sort.join
end

def find_anagrams(word, word_list)
  word_list.select { |w| anagram?(word, w) && w.downcase != word.downcase }
end

word_pairs = [
  ["listen", "silent"],
  ["hello", "world"],
  ["astronomer", "moon starer"],
  ["debit card", "bad credit"],
  ["school master", "the classroom"]
]

puts "=== Anagram Checker ==="
word_pairs.each do |w1, w2|
  result = anagram?(w1, w2)
  puts "\"#{w1}\" และ \"#{w2}\": #{result ? '✓ เป็น anagram' : '✗ ไม่ใช่ anagram'}"
end

word_list = ["race", "care", "acre", "nacre", "ocean", "canoe"]
puts "\nAnagrams ของ 'race' ใน [#{word_list.join(', ')}]:"
puts find_anagrams("race", word_list).join(", ")
```

### แบบฝึกหัดที่ 15: Text Statistics Dashboard

```ruby
def generate_text_report(text)
  word_count = text.split.length
  char_count = text.length
  sentence_count = text.scan(/[.!?]/).length
  avg_words_per_sentence = (word_count.to_f / [sentence_count, 1].max).round(1)
  reading_time = (word_count / 200.0).ceil  # ~200 words/minute

  words = text.downcase.scan(/\b[a-z]{3,}\b/)
  word_freq = words.each_with_object(Hash.new(0)) { |w, h| h[w] += 1 }
  top_words = word_freq.sort_by { |_, v| -v }.first(5)

  longest_word = text.scan(/\b\w+\b/).max_by(&:length)

  <<~REPORT
    ╔══════════════════════════════════════╗
    ║         Text Analysis Report         ║
    ╚══════════════════════════════════════╝

    📊 Basic Statistics:
      • Total characters:  #{char_count.to_s.rjust(10)}
      • Total words:       #{word_count.to_s.rjust(10)}
      • Total sentences:   #{sentence_count.to_s.rjust(10)}
      • Unique words:      #{words.uniq.length.to_s.rjust(10)}

    📖 Readability:
      • Words/sentence:    #{avg_words_per_sentence.to_s.rjust(10)}
      • Reading time:      #{reading_time.to_s.rjust(9)} min
      • Longest word:      #{longest_word.rjust(10)}

    🔤 Top 5 Words:
    #{top_words.map.with_index { |(w, c), i| "    #{i+1}. #{w.ljust(15)} (#{c}x)" }.join("\n")}
  REPORT
end

sample_text = <<~TEXT
  Ruby is a dynamic, open source programming language with a focus on simplicity and productivity.
  It has an elegant syntax that is natural to read and easy to write.
  Ruby was created by Yukihiro Matsumoto in the 1990s.
  Ruby is often used for web development, especially with the Ruby on Rails framework.
  Many startups and companies around the world use Ruby to build their applications.
TEXT

puts generate_text_report(sample_text)
```

### แบบฝึกหัดที่ 16-20: โจทย์เพิ่มเติม

```ruby
# แบบฝึกหัดที่ 16: Slugify
def slugify(text)
  text
    .downcase
    .gsub(/[àáâãäåæ]/, 'a')
    .gsub(/[èéêë]/, 'e')
    .gsub(/[ìíîï]/, 'i')
    .gsub(/[òóôõö]/, 'o')
    .gsub(/[ùúûü]/, 'u')
    .gsub(/[^a-z0-9\s-]/, '')
    .gsub(/\s+/, '-')
    .gsub(/-+/, '-')
    .gsub(/\A-|-\z/, '')
end

titles = [
  "Hello World!",
  "Ruby on Rails Tutorial",
  "10 Tips for Better Code",
  "  Extra  Spaces  "
]

titles.each { |t| puts "\"#{t}\" → \"#{slugify(t)}\"" }

# แบบฝึกหัดที่ 17: String to Binary
def to_binary(str)
  str.bytes.map { |b| b.to_s(2).rjust(8, '0') }.join(' ')
end

def from_binary(binary)
  binary.split(' ').map { |b| b.to_i(2).chr }.join
end

puts to_binary("Hi")
puts from_binary(to_binary("Ruby"))

# แบบฝึกหัดที่ 18: Truncate with Ellipsis
def truncate(str, length: 50, omission: "...")
  return str if str.length <= length
  "#{str[0, length - omission.length]}#{omission}"
end

long_text = "Ruby is a dynamic, open source programming language with a focus on simplicity and productivity."
puts truncate(long_text, length: 40)
puts truncate(long_text, length: 60)

# แบบฝึกหัดที่ 19: Parse CSV Line
def parse_csv_line(line, delimiter: ",", quote_char: '"')
  fields = []
  current = ""
  in_quotes = false

  line.each_char do |char|
    if char == quote_char
      in_quotes = !in_quotes
    elsif char == delimiter && !in_quotes
      fields << current
      current = ""
    else
      current << char
    end
  end
  fields << current

  fields
end

csv_lines = [
  'Alice,25,"Bangkok, Thailand",alice@example.com',
  '"Smith, John",30,New York,john.smith@example.com',
  'Bob,22,"He said ""Hello""",bob@example.com'
]

csv_lines.each do |line|
  fields = parse_csv_line(line)
  puts fields.inspect
end

# แบบฝึกหัดที่ 20: String Diff (simple)
def highlight_diff(str1, str2)
  max_len = [str1.length, str2.length].max
  diff_positions = (0...max_len).select { |i| str1[i] != str2[i] }

  puts "String 1: #{str1}"
  puts "String 2: #{str2}"
  puts "Diff:     #{(0...max_len).map { |i| diff_positions.include?(i) ? '^' : ' ' }.join}"
  puts "Changed at positions: #{diff_positions.inspect}"
  puts "Similarity: #{((1 - diff_positions.length.to_f / max_len) * 100).round(1)}%"
end

highlight_diff("hello world", "hello ruby!")
```

---

## สรุปส่วนที่ 3

ในส่วนนี้เราได้เรียนรู้เรื่อง String อย่างครบถ้วน:

1. **String Creation** - single/double quotes, %q/%Q, String.new
2. **Case Methods** - upcase, downcase, capitalize, swapcase
3. **Length/Size** - length, bytesize, count, empty?
4. **String Methods** - reverse, strip, chomp, chop
5. **Searching** - include?, index, scan, match
6. **Interpolation** - #{}, expression, methods
7. **Formatting** - printf, sprintf, %, format
8. **Heredoc** - <<HEREDOC, <<~HEREDOC, frozen heredoc
9. **Comparison** - ==, <=>, casecmp, sort
10. **Manipulation** - gsub, sub, tr, delete, split, join
11. **Regex** - =~, match, scan, gsub with regex
12. **Encoding** - UTF-8, encode, bytesize
13. **Frozen Strings** - freeze, frozen_string_literal
14. **Performance** - <<, concat, Array join

---

*เอกสารนี้เป็นส่วนหนึ่งของคอร์ส Ruby on Rails สำหรับผู้เริ่มต้น*
