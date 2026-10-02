# Part 16: Regular Expressions (Regex) ใน Ruby

## ขั้นตอนที่ 321-345: การใช้ Regular Expressions

---

## บทนำ

Regular Expressions (Regex หรือ RegExp) คือรูปแบบ (pattern) ที่ใช้สำหรับค้นหา จับคู่ และแปลงข้อความ ใน Ruby นั้น Regex เป็น first-class citizen มีซินแทกซ์ที่ชัดเจนและมีประสิทธิภาพสูง

---

## ขั้นตอนที่ 321: ซินแทกซ์พื้นฐานของ Regex

ใน Ruby เราเขียน Regex โดยใช้ `/pattern/` หรือ `Regexp.new("pattern")`

```ruby
# การสร้าง Regex
pattern1 = /hello/
pattern2 = Regexp.new("hello")
pattern3 = /hello/i  # i = case insensitive

puts pattern1.class   # => Regexp
puts pattern2.class   # => Regexp

# การตรวจสอบว่า string ตรงกับ pattern หรือไม่
str = "Hello, World!"
puts str.match?(/Hello/)   # => true
puts str.match?(/hello/)   # => false
puts str.match?(/hello/i)  # => true (case insensitive)
```

### Flags / Modifiers

```ruby
# i - case insensitive (ไม่สนใจตัวพิมพ์ใหญ่-เล็ก)
puts "Hello" =~ /hello/i    # => 0

# m - multiline (. จับคู่ newline ด้วย)
text = "line1\nline2"
puts text.match?(/line1.line2/)   # => false
puts text.match?(/line1.line2/m)  # => true

# x - extended (อนุญาตให้ใส่ comment และ whitespace)
pattern = /
  \d+   # ตัวเลข
  \.    # จุดทศนิยม
  \d+   # ตัวเลขหลังจุด
/x
puts "3.14".match?(pattern)  # => true

# Combining flags
puts "HELLO\nWORLD".match?(/hello.*world/im)  # => true
```

---

## ขั้นตอนที่ 322: ตัวดำเนินการ `=~` และ `match`

### ตัวดำเนินการ `=~`

```ruby
str = "Ruby is awesome"

# =~ คืนค่า index ที่พบ หรือ nil ถ้าไม่พบ
result = str =~ /is/
puts result   # => 5 (ตำแหน่งเริ่มต้น)

result = str =~ /python/
puts result.nil?   # => true

# การใช้ใน if statement
if str =~ /awesome/
  puts "พบคำว่า awesome!"
end

# ตัวอย่างใน conditional
words = ["ruby", "python", "javascript", "java"]
words.each do |word|
  if word =~ /java/
    puts "#{word} มีคำว่า java"
  end
end
# => javascript มีคำว่า java
# => java มีคำว่า java
```

### เมธอด `match`

```ruby
str = "John is 25 years old"

# match คืนค่า MatchData object หรือ nil
md = str.match(/(\w+) is (\d+)/)

if md
  puts md[0]    # => "John is 25" (การจับคู่ทั้งหมด)
  puts md[1]    # => "John" (capture group 1)
  puts md[2]    # => "25" (capture group 2)
  puts md.pre_match   # => "" (ส่วนก่อนการจับคู่)
  puts md.post_match  # => " years old" (ส่วนหลังการจับคู่)
end

# match? - ตรวจสอบโดยไม่เก็บ MatchData (เร็วกว่า)
puts str.match?(/\d+/)  # => true
```

---

## ขั้นตอนที่ 323: เมธอด `scan`

`scan` หาการจับคู่ทั้งหมดและคืนค่าเป็น Array

```ruby
str = "cat bat hat mat"

# หาทุกคำที่ลงท้ายด้วย at
puts str.scan(/\w+at/).inspect
# => ["cat", "bat", "hat", "mat"]

# ตัวอย่างกับตัวเลข
text = "ราคา 100 บาท, ราคา 200 บาท, ราคา 350 บาท"
prices = text.scan(/\d+/)
puts prices.inspect   # => ["100", "200", "350"]
puts prices.map(&:to_i).sum   # => 650

# scan กับ capture groups
str = "John:25, Jane:30, Bob:22"
result = str.scan(/(\w+):(\d+)/)
puts result.inspect
# => [["John", "25"], ["Jane", "30"], ["Bob", "22"]]

result.each do |name, age|
  puts "#{name} อายุ #{age} ปี"
end

# scan กับ block
"hello world".scan(/\w+/) do |word|
  puts word.upcase
end
# => HELLO
# => WORLD
```

---

## ขั้นตอนที่ 324: เมธอด `gsub` กับ Regex

```ruby
# gsub แทนที่ทุกการจับคู่
str = "Hello, World! Hello, Ruby!"
puts str.gsub(/Hello/, "Hi")
# => "Hi, World! Hi, Ruby!"

# gsub กับ block
text = "the quick brown fox"
result = text.gsub(/\b\w/) { |match| match.upcase }
puts result   # => "The Quick Brown Fox"

# gsub กับ hash
text = "I have cats and dogs"
replacements = { "cats" => "แมว", "dogs" => "สุนัข" }
result = text.gsub(/cats|dogs/, replacements)
puts result   # => "I have แมว and สุนัข"

# gsub กับ back-references
# \1 อ้างอิง capture group แรก
str = "2024-01-15"
puts str.gsub(/(\d{4})-(\d{2})-(\d{2})/, '\3/\2/\1')
# => "15/01/2024"

# sub - แทนที่เฉพาะการจับคู่แรก
str = "banana"
puts str.sub(/a/, "o")    # => "bonana"
puts str.gsub(/a/, "o")   # => "bonono"
```

---

## ขั้นตอนที่ 325: Character Classes

Character classes กำหนดชุดของตัวอักษรที่สามารถจับคู่ได้

```ruby
# [abc] - จับคู่ a, b, หรือ c
puts "cat".match?(/[abc]/)   # => true
puts "dog".match?(/[abc]/)   # => false

# [a-z] - จับคู่ตัวอักษรพิมพ์เล็ก
puts "hello".match?(/[a-z]+/)   # => true

# [A-Z] - จับคู่ตัวอักษรพิมพ์ใหญ่
puts "Hello".scan(/[A-Z]/).inspect   # => ["H"]

# [0-9] - จับคู่ตัวเลข (เหมือนกับ \d)
puts "abc123".scan(/[0-9]+/).inspect   # => ["123"]

# [^abc] - ไม่จับคู่ a, b, หรือ c
puts "dog".match?(/[^abc]/)   # => true (d ไม่ใช่ a, b, c)
puts "abc".scan(/[^abc]/).inspect   # => []

# ตัวอย่างจริง
emails = ["user@example.com", "invalid-email", "test@test.org"]
emails.each do |email|
  if email =~ /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/
    puts "#{email} - ถูกต้อง"
  else
    puts "#{email} - ไม่ถูกต้อง"
  end
end
```

### Shorthand Character Classes

```ruby
# \d - ตัวเลข [0-9]
puts "abc123".scan(/\d/).inspect   # => ["1", "2", "3"]

# \D - ไม่ใช่ตัวเลข [^0-9]
puts "abc123".scan(/\D/).inspect   # => ["a", "b", "c"]

# \w - ตัวอักษร ตัวเลข และ _ [a-zA-Z0-9_]
puts "hello_world 123".scan(/\w+/).inspect
# => ["hello_world", "123"]

# \W - ไม่ใช่ word character
puts "hello world!".scan(/\W/).inspect   # => [" ", "!"]

# \s - whitespace (space, tab, newline)
puts "hello   world".split(/\s+/).inspect   # => ["hello", "world"]

# \S - ไม่ใช่ whitespace
puts "hello world".scan(/\S+/).inspect   # => ["hello", "world"]

# . - อะไรก็ได้ยกเว้น newline
puts "abc123!@#".scan(/./).inspect
# => ["a", "b", "c", "1", "2", "3", "!", "@", "#"]

# การใช้ในชีวิตจริง
phone = "081-234-5678"
digits_only = phone.gsub(/\D/, "")
puts digits_only   # => "0812345678"
```

---

## ขั้นตอนที่ 326: Quantifiers (ตัวระบุจำนวน)

```ruby
# * - 0 ครั้งหรือมากกว่า
puts "colour".match?(/colou*r/)    # => true
puts "color".match?(/colou*r/)     # => true
puts "colouur".match?(/colou*r/)   # => true

# + - 1 ครั้งหรือมากกว่า
puts "color".match?(/colou+r/)     # => false
puts "colour".match?(/colou+r/)    # => true
puts "colouur".match?(/colou+r/)   # => true

# ? - 0 หรือ 1 ครั้ง (optional)
puts "color".match?(/colou?r/)     # => true
puts "colour".match?(/colou?r/)    # => true

# {n} - ตรงกัน n ครั้ง
puts "aaa".match?(/a{3}/)   # => true
puts "aa".match?(/a{3}/)    # => false

# {n,} - n ครั้งหรือมากกว่า
puts "aaa".match?(/a{2,}/)   # => true
puts "a".match?(/a{2,}/)     # => false

# {n,m} - ระหว่าง n ถึง m ครั้ง
puts "aa".match?(/a{2,4}/)    # => true
puts "aaaa".match?(/a{2,4}/)  # => true
puts "aaaaa".match?(/a{2,4}/) # => false (เกิน 4)

# ตัวอย่างจริง
# ตรวจสอบรหัสไปรษณีย์ไทย (5 หลัก)
postal_codes = ["10110", "12345", "1234", "123456", "abcde"]
postal_codes.each do |code|
  if code =~ /^\d{5}$/
    puts "#{code} - ถูกต้อง"
  else
    puts "#{code} - ไม่ถูกต้อง"
  end
end
```

---

## ขั้นตอนที่ 327: Anchors (จุดยึด)

```ruby
# ^ - จุดเริ่มต้นของบรรทัด
# $ - จุดสิ้นสุดของบรรทัด
# \A - จุดเริ่มต้นของ string
# \Z - จุดสิ้นสุดของ string (ก่อน newline ท้าย)
# \z - จุดสิ้นสุดของ string (เด็ดขาด)
# \b - word boundary
# \B - ไม่ใช่ word boundary

# ^ และ $
lines = ["Hello World", "  Hello Ruby", "Ruby is great"]
lines.each do |line|
  puts "#{line} - เริ่มด้วย Hello" if line =~ /^Hello/
end
# => "Hello World - เริ่มด้วย Hello"

# \A และ \z
text = "Ruby\nPython"
puts text.match?(/\ARuby/)   # => true (เริ่มต้น string)
puts text.match?(/\ARuby\z/) # => false (Ruby ไม่ใช่ท้าย string)

# \b - word boundary
str = "cats concatenate"
puts str.scan(/\bcat\b/).inspect   # => ["cats"] wait
puts str.scan(/\bcat\b/).inspect   # ไม่ใช่ - ลองใหม่
puts "cat cats catch".scan(/\bcat\b/).inspect   # => ["cat"]
puts "cat cats catch".scan(/\bcat/).inspect     # => ["cat", "cat", "cat"]

# ตัวอย่างที่ใช้จริง
# ตรวจสอบว่า string เป็นตัวเลขทั้งหมด
def all_digits?(str)
  str.match?(/\A\d+\z/)
end

puts all_digits?("12345")    # => true
puts all_digits?("123a5")    # => false
puts all_digits?("  123  ")  # => false

# ตรวจสอบ username
def valid_username?(username)
  username.match?(/\A[a-zA-Z][a-zA-Z0-9_]{2,19}\z/)
end

puts valid_username?("john_doe")    # => true
puts valid_username?("jo")          # => false (สั้นเกิน)
puts valid_username?("1john")       # => false (ขึ้นต้นด้วยตัวเลข)
```

---

## ขั้นตอนที่ 328: Capture Groups (กลุ่มจับ)

```ruby
# วงเล็บ () สร้าง capture group
str = "2024-01-15"
md = str.match(/(\d{4})-(\d{2})-(\d{2})/)
if md
  year  = md[1]   # => "2024"
  month = md[2]   # => "01"
  day   = md[3]   # => "15"
  puts "ปี: #{year}, เดือน: #{month}, วัน: #{day}"
end

# Non-capturing group (?:...)
str = "2024-01-15"
md = str.match(/(?:\d{4})-(\d{2})-(\d{2})/)
if md
  puts md[0]   # => "2024-01-15"
  puts md[1]   # => "01" (ปีไม่ถูกจับ)
  puts md[2]   # => "15"
end

# Alternation กับ groups
str = "I like cats and dogs"
md = str.match(/(cats|dogs)/)
puts md[1] if md   # => "cats"

# ตัวอย่าง URL parsing
url = "https://www.example.com:8080/path?key=value"
pattern = /^(https?):\/\/([^:\/]+)(?::(\d+))?(\/[^?]*)?(?:\?(.*))?$/
md = url.match(pattern)
if md
  puts "Protocol: #{md[1]}"  # => https
  puts "Host: #{md[2]}"      # => www.example.com
  puts "Port: #{md[3]}"      # => 8080
  puts "Path: #{md[4]}"      # => /path
  puts "Query: #{md[5]}"     # => key=value
end
```

---

## ขั้นตอนที่ 329: Named Captures (การจับชื่อ)

Named captures ช่วยให้ code อ่านง่ายขึ้นมาก

```ruby
# (?<name>pattern) - named capture
str = "John Smith, age 30"
md = str.match(/(?<first>\w+)\s+(?<last>\w+),\s+age\s+(?<age>\d+)/)

if md
  puts md[:first]   # => "John"
  puts md[:last]    # => "Smith"
  puts md[:age]     # => "30"
  
  # md.named_captures คืนค่า Hash
  puts md.named_captures.inspect
  # => {"first"=>"John", "last"=>"Smith", "age"=>"30"}
end

# Named captures กับ gsub
str = "2024-01-15"
result = str.gsub(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/) do
  "#{$~[:day]}/#{$~[:month]}/#{$~[:year]}"
end
puts result   # => "15/01/2024"

# ตัวอย่าง: parsing log file
log = "[2024-01-15 10:30:45] ERROR: Connection timeout"
pattern = /\[(?<date>\d{4}-\d{2}-\d{2})\s+(?<time>\d{2}:\d{2}:\d{2})\]\s+(?<level>\w+):\s+(?<message>.*)/

md = log.match(pattern)
if md
  puts "วันที่: #{md[:date]}"
  puts "เวลา: #{md[:time]}"
  puts "ระดับ: #{md[:level]}"
  puts "ข้อความ: #{md[:message]}"
end

# Named captures กับ =~ จะสร้าง local variables
if /(?<year>\d{4})-(?<month>\d{2})/ =~ "2024-01"
  puts year    # => "2024"  (local variable อัตโนมัติ!)
  puts month   # => "01"
end
```

---

## ขั้นตอนที่ 330: Lookahead และ Lookbehind

### Lookahead

```ruby
# Positive lookahead (?=...)
# จับคู่ที่ตามด้วย pattern ที่ระบุ (ไม่รวมส่วนที่ระบุ)

# หาตัวเลขที่ตามด้วย "px"
str = "width: 100px, height: 200px, margin: 10em"
puts str.scan(/\d+(?=px)/).inspect   # => ["100", "200"]

# Negative lookahead (?!...)
# จับคู่ที่ไม่ตามด้วย pattern ที่ระบุ
str = "file1.rb file2.txt file3.rb"
puts str.scan(/\w+(?!\.rb)\.\w+/).inspect  # ซับซ้อน ลองวิธีอื่น

# หาตัวเลขที่ไม่ตามด้วย "px"
str = "100px 200em 300px 400%"
puts str.scan(/\d+(?!px)/).inspect   # => ["200", "400", ...]

# ตัวอย่างที่ดีกว่า
words = "cats catnip catalog"
puts words.scan(/cat(?=nip|alog)/).inspect   # => ["cat", "cat"]
puts words.scan(/cat(?=s|\b)/).inspect       # => ["cat"]
```

### Lookbehind

```ruby
# Positive lookbehind (?<=...)
# จับคู่ที่นำหน้าด้วย pattern ที่ระบุ

# หาตัวเลขที่นำหน้าด้วย "$"
str = "price: $100, discount: $20, total: $80"
puts str.scan(/(?<=\$)\d+/).inspect   # => ["100", "20", "80"]

# Negative lookbehind (?<!...)
# จับคู่ที่ไม่ได้นำหน้าด้วย pattern ที่ระบุ
str = "100 $200 300 $400"
puts str.scan(/(?<!\$)\d+/).inspect   # => ["100", "300"]

# ตัวอย่างจริง: แยก domain จาก email
emails = ["user@example.com", "admin@company.org"]
emails.each do |email|
  domain = email.match(/(?<=@)[^.]+\.\w+/)
  puts "Domain: #{domain[0]}" if domain
end
# => Domain: example.com
# => Domain: company.org

# ตัวอย่าง: ตรวจหาคำที่อยู่หลัง "not"
text = "I am not happy and not sad"
puts text.scan(/(?<=not\s)\w+/).inspect   # => ["happy", "sad"]
```

---

## ขั้นตอนที่ 331: Global Variables `$~`, `$1`, `$2`

```ruby
# หลังจาก match สำเร็จ Ruby จะตั้งค่า global variables อัตโนมัติ

str = "Ruby 3.2.0 released"
if str =~ /(\w+)\s+(\d+\.\d+\.\d+)/
  puts $~.inspect  # MatchData object ทั้งหมด
  puts $~[0]       # => "Ruby 3.2.0" (การจับคู่ทั้งหมด)
  puts $1          # => "Ruby" (capture group 1)
  puts $2          # => "3.2.0" (capture group 2)
  puts $`          # => "" (pre-match)
  puts $'          # => " released" (post-match)
  puts $&          # => "Ruby 3.2.0" (การจับคู่ทั้งหมด)
end

# $~ เปลี่ยนทุกครั้งที่มีการ match
"hello" =~ /(\w+)/
first_match = $1   # => "hello"

"world 42" =~ /(\d+)/
second_match = $1  # => "42"

puts first_match   # => "hello" (ยังคงค่าเดิม)
puts second_match  # => "42"

# ตัวอย่างการใช้งาน
def extract_version(str)
  if str =~ /version\s+(\d+)\.(\d+)\.(\d+)/i
    { major: $1.to_i, minor: $2.to_i, patch: $3.to_i }
  end
end

info = extract_version("Ruby version 3.2.1 is out")
puts info.inspect   # => {:major=>3, :minor=>2, :patch=>1}
```

---

## ขั้นตอนที่ 332: Multiline Matching

```ruby
# โดยปกติ . ไม่จับคู่ newline
text = "line 1\nline 2\nline 3"

# ไม่ใช้ m flag
puts text.match?(/line 1.line 2/)   # => false

# ใช้ m flag
puts text.match?(/line 1.line 2/m)  # => true

# ^ และ $ vs \A และ \z
multiline_text = "first line\nsecond line\nthird line"

# ^ จับคู่จุดเริ่มต้นของแต่ละบรรทัด
puts multiline_text.scan(/^\w+/).inspect
# => ["first", "second", "third"]

# \A จับคู่เฉพาะจุดเริ่มต้นของ string
puts multiline_text.match(/\A\w+/)[0]   # => "first"

# $ จับคู่จุดสิ้นสุดของแต่ละบรรทัด
puts multiline_text.scan(/\w+$/).inspect
# => ["line", "line", "line"]

# ตัวอย่างจริง: การ parse ข้อมูลหลายบรรทัด
config = """
name = Ruby
version = 3.2
author = Matz
"""

config.scan(/^(\w+)\s*=\s*(.+)$/) do |key, value|
  puts "#{key}: #{value}"
end
# => name: Ruby
# => version: 3.2
# => author: Matz
```

---

## ขั้นตอนที่ 333: Non-Greedy Matching

```ruby
# โดยปกติ Quantifiers เป็น greedy (จับคู่มากที่สุด)
str = "<h1>Title</h1><p>Paragraph</p>"

# Greedy - จับคู่มากที่สุด
puts str.match(/<.+>/)[0]
# => "<h1>Title</h1><p>Paragraph</p>" (ทั้งหมดระหว่าง < แรก และ > สุดท้าย)

# Non-greedy (lazy) - จับคู่น้อยที่สุด โดยใช้ ?
puts str.match(/<.+?>/)[0]
# => "<h1>" (แค่ tag แรก)

puts str.scan(/<.+?>/).inspect
# => ["<h1>", "</h1>", "<p>", "</p>"]

# *? - non-greedy *
str = "aXXXb aYb"
puts str.match(/a.*b/)[0]    # => "aXXXb aYb" (greedy)
puts str.match(/a.*?b/)[0]   # => "aXXXb" (non-greedy)

# +? - non-greedy +
str = "aaabbbccc"
puts str.match(/a+/)[0]    # => "aaa" (greedy)
puts str.match(/a+?/)[0]   # => "a" (non-greedy - แค่ 1 ตัว)

# {n,m}? - non-greedy range
str = "aaaaaaa"
puts str.match(/a{2,5}/)[0]    # => "aaaaa" (greedy - เต็ม 5)
puts str.match(/a{2,5}?/)[0]   # => "aa" (non-greedy - แค่ 2)

# ตัวอย่างจริง: extract HTML content
html = "<p>First paragraph</p><p>Second paragraph</p>"
paragraphs = html.scan(/<p>(.*?)<\/p>/)
puts paragraphs.inspect
# => [["First paragraph"], ["Second paragraph"]]
```

---

## ขั้นตอนที่ 334: เมธอด Regex อื่นๆ

```ruby
# split กับ Regex
str = "one two  three   four"
puts str.split(/\s+/).inspect
# => ["one", "two", "three", "four"]

# split กับ limit
puts str.split(/\s+/, 2).inspect
# => ["one", "two  three   four"]

# start_with? และ end_with? (ไม่ใช่ regex แต่มีประโยชน์)
puts "hello".start_with?("hel")   # => true
puts "hello".end_with?("llo")     # => true

# tr (translate characters)
puts "hello".tr('aeiou', '*')   # => "h*ll*"
puts "hello".tr('a-y', 'b-z')   # => "ifmmp"

# squeeze - ลดตัวซ้ำ
puts "aaabbbccc".squeeze   # => "abc"
puts "hello world".squeeze(' ')   # => "hello world"

# ตัวอย่าง: เมธอด scan กับ named captures
text = "Alice: 25, Bob: 30, Charlie: 22"
pattern = /(?<name>\w+):\s*(?<age>\d+)/
text.scan(pattern).each do |name, age|
  puts "#{name} อายุ #{age} ปี"
end
# Note: scan กับ named captures จะคืนเป็น array ของ arrays

# ใช้ gsub กับ block เพื่อ transform
sentence = "the quick brown fox"
title_case = sentence.gsub(/\b\w+/) { |word| word.capitalize }
puts title_case   # => "The Quick Brown Fox"
```

---

## ขั้นตอนที่ 335: การตรวจสอบ Email

```ruby
# Email validation - regex มาตรฐาน
EMAIL_REGEX = /\A[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\z/

def valid_email?(email)
  email.match?(EMAIL_REGEX)
end

# ทดสอบ
emails = [
  "user@example.com",
  "user.name+tag@example.co.th",
  "invalid-email",
  "@example.com",
  "user@",
  "user@.com",
  "user@example",
  "User.Name@EXAMPLE.COM"
]

emails.each do |email|
  status = valid_email?(email) ? "✓ ถูกต้อง" : "✗ ไม่ถูกต้อง"
  puts "#{email.ljust(35)} #{status}"
end

# Extract emails จาก text
text = "ติดต่อ john@example.com หรือ admin@company.org สำหรับข้อมูลเพิ่มเติม"
emails = text.scan(/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/)
puts emails.inspect
# => ["john@example.com", "admin@company.org"]
```

---

## ขั้นตอนที่ 336: การ Parse เบอร์โทรศัพท์

```ruby
# Phone number parsing
def parse_thai_phone(phone)
  # ลบอักขระที่ไม่ใช่ตัวเลขออก
  digits = phone.gsub(/\D/, "")
  
  case digits
  when /\A0[689]\d{8}\z/
    # เบอร์มือถือ: 08x, 09x, 06x
    { type: "มือถือ", formatted: "#{digits[0,3]}-#{digits[3,3]}-#{digits[6,4]}" }
  when /\A0[2-5]\d{7}\z/
    # เบอร์บ้าน 2 หลัก
    { type: "บ้าน", formatted: "#{digits[0,2]}-#{digits[2,3]}-#{digits[5,4]}" }
  when /\A66[689]\d{8}\z/
    # รูปแบบ +66
    { type: "มือถือ (+66)", formatted: "+66-#{digits[2,2]}-#{digits[4,3]}-#{digits[7,4]}" }
  else
    { type: "ไม่ทราบรูปแบบ", formatted: phone }
  end
end

phones = [
  "081-234-5678",
  "0812345678",
  "+66 81 234 5678",
  "02-345-6789",
  "02-3456789",
  "555-1234"
]

phones.each do |phone|
  result = parse_thai_phone(phone)
  puts "#{phone.ljust(20)} => #{result[:type]}: #{result[:formatted]}"
end

# ดึงเบอร์โทรจากข้อความ
text = "ติดต่อได้ที่ 081-234-5678 หรือ 02-345-6789 ขอบคุณ"
phones = text.scan(/\d[\d\s\-]{8,12}\d/)
puts phones.inspect
```

---

## ขั้นตอนที่ 337: การ Extract URL

```ruby
# URL extraction
URL_REGEX = %r{
  https?://         # protocol
  [a-zA-Z0-9.-]+   # domain
  (?::\d+)?        # port (optional)
  (?:/[^\s]*)?     # path (optional)
}x

text = """
เยี่ยมชมเว็บไซต์ที่ https://www.example.com และ
ดาวน์โหลดจาก http://files.example.org/download?id=123
หรือ API ที่ https://api.service.com:8080/v1/data
"""

urls = text.scan(URL_REGEX)
urls.each { |url| puts url }

# URL components extraction
def parse_url(url)
  pattern = %r{
    ^(?<protocol>https?)://    # protocol
    (?<host>[^/:]+)            # host
    (?::(?<port>\d+))?         # port
    (?<path>/[^?#]*)?          # path
    (?:\?(?<query>[^#]*))?     # query
    (?:\#(?<fragment>.*))?     # fragment
  }x
  
  md = url.match(pattern)
  return nil unless md
  
  md.named_captures
end

url = "https://www.example.com:8080/path/to/page?name=ruby&version=3#section"
parts = parse_url(url)
parts.each { |key, val| puts "#{key}: #{val}" } if parts
```

---

## ขั้นตอนที่ 338: ตัวอย่างการใช้งานจริง - Log Parser

```ruby
# Log file parser
class LogParser
  LOG_PATTERN = /
    \[(?<timestamp>\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2})\]  # timestamp
    \s+
    (?<level>DEBUG|INFO|WARN|ERROR|FATAL)  # log level
    \s+
    (?<message>.+)  # message
  /x

  def initialize(log_text)
    @log_text = log_text
  end

  def entries
    @entries ||= parse_entries
  end

  def filter_by_level(level)
    entries.select { |e| e[:level] == level.to_s.upcase }
  end

  def errors
    filter_by_level("ERROR") + filter_by_level("FATAL")
  end

  private

  def parse_entries
    @log_text.lines.filter_map do |line|
      md = line.match(LOG_PATTERN)
      next unless md
      md.named_captures.transform_keys(&:to_sym)
    end
  end
end

# ทดสอบ
log_text = """
[2024-01-15 10:00:01] INFO Application started
[2024-01-15 10:00:05] DEBUG Loading configuration
[2024-01-15 10:01:00] WARN Memory usage high: 85%
[2024-01-15 10:02:30] ERROR Database connection failed
[2024-01-15 10:03:00] INFO Retry connection...
[2024-01-15 10:03:05] INFO Connection restored
[2024-01-15 10:05:00] ERROR Timeout on request /api/data
"""

parser = LogParser.new(log_text)
puts "ทั้งหมด: #{parser.entries.size} entries"
puts "Errors: #{parser.errors.size}"
parser.errors.each do |e|
  puts "  [#{e[:timestamp]}] #{e[:message]}"
end
```

---

## ขั้นตอนที่ 339: ตัวอย่างการใช้งานจริง - String Transformation

```ruby
# CamelCase ↔ snake_case conversion
def to_snake_case(str)
  str
    .gsub(/([A-Z]+)([A-Z][a-z])/, '\1_\2')
    .gsub(/([a-z\d])([A-Z])/, '\1_\2')
    .downcase
end

def to_camel_case(str)
  str.gsub(/_([a-z])/) { $1.upcase }
end

def to_pascal_case(str)
  to_camel_case(str).sub(/^[a-z]/) { $&.upcase }
end

# ทดสอบ
puts to_snake_case("CamelCaseString")      # => "camel_case_string"
puts to_snake_case("HTMLParser")           # => "html_parser"
puts to_snake_case("simpleXMLParser")      # => "simple_xml_parser"
puts to_camel_case("snake_case_string")    # => "snakeCaseString"
puts to_pascal_case("snake_case_string")   # => "SnakeCaseString"

# Slug generation
def to_slug(str)
  str
    .downcase
    .gsub(/[àáâãäå]/, 'a')
    .gsub(/[èéêë]/, 'e')
    .gsub(/[ìíîï]/, 'i')
    .gsub(/[òóôõö]/, 'o')
    .gsub(/[ùúûü]/, 'u')
    .gsub(/[^a-z0-9\s-]/, '')
    .gsub(/\s+/, '-')
    .gsub(/-+/, '-')
    .strip
end

puts to_slug("Hello World!")          # => "hello-world"
puts to_slug("Ruby on Rails 7.0")    # => "ruby-on-rails-70"
puts to_slug("  Multiple   Spaces  ") # => "multiple-spaces"
```

---

## ขั้นตอนที่ 340: ตัวอย่างการใช้งานจริง - CSV-like Parsing

```ruby
# Simple CSV parser
def parse_csv_line(line)
  fields = []
  field = ""
  in_quotes = false
  
  line.chars.each do |char|
    case char
    when '"'
      in_quotes = !in_quotes
    when ','
      if in_quotes
        field += char
      else
        fields << field
        field = ""
      end
    else
      field += char
    end
  end
  fields << field
  fields
end

# หรือใช้ regex
def parse_csv_line_regex(line)
  line.scan(/(?:"([^"]*)")|([^,]+)|(?:,(?=,|$))/)
      .map { |a, b| a || b || "" }
end

# Markdown link extraction
def extract_links(markdown)
  pattern = /\[(?<text>[^\]]+)\]\((?<url>[^)]+)\)/
  markdown.scan(pattern).map do |text, url|
    { text: text, url: url }
  end
end

markdown = """
ดูเพิ่มเติมที่ [Ruby Docs](https://ruby-doc.org) และ
[Rails Guide](https://guides.rubyonrails.org)
"""

links = extract_links(markdown)
links.each do |link|
  puts "#{link[:text]} -> #{link[:url]}"
end
```

---

## ขั้นตอนที่ 341: Regex Performance Tips

```ruby
# Tip 1: ใช้ match? แทน match เมื่อไม่ต้องการ MatchData
require 'benchmark'

str = "Hello, World! This is a test string."

Benchmark.bm do |x|
  x.report("match:  ") { 1_000_000.times { str.match(/\w+/) } }
  x.report("match?: ") { 1_000_000.times { str.match?(/\w+/) } }
end

# Tip 2: compile regex ไว้นอก loop
WORD_PATTERN = /\w+/  # Compiled once

big_array = ["hello", "world"] * 10000

# ดีกว่า (compiled once)
big_array.each { |s| s.match?(WORD_PATTERN) }

# ช้ากว่า (compile ใหม่ทุก iteration)
big_array.each { |s| s.match?(/\w+/) }

# Tip 3: ใช้ Non-capturing group เมื่อไม่ต้องการ capture
# (?:...) เร็วกว่า (...) เล็กน้อย

# Tip 4: หลีกเลี่ยง catastrophic backtracking
# ระวัง pattern แบบ (a+)+ หรือ (a|aa)+
# สำหรับ input ยาวๆ อาจช้ามาก

# Tip 5: ใช้ anchors เพื่อลด backtracking
str = "the quick brown fox"
# ดีกว่า - anchor บอกว่าต้องเริ่มที่ไหน
str.match?(/\Athe/)
# ช้ากว่า - ต้องลองทุกตำแหน่ง
str.match?(/the/)
```

---

## ขั้นตอนที่ 342: Regex Patterns ที่ใช้บ่อย

```ruby
# Collection ของ Regex patterns ที่มีประโยชน์

module Patterns
  # ตัวเลข
  INTEGER     = /\A-?\d+\z/
  FLOAT       = /\A-?\d+\.\d+\z/
  NUMBER      = /\A-?\d+(\.\d+)?\z/
  
  # ข้อมูลส่วนตัว
  EMAIL       = /\A[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\z/
  THAI_PHONE  = /\A0[2-9]\d{7,8}\z/
  
  # วันที่
  DATE_ISO    = /\A\d{4}-\d{2}-\d{2}\z/
  DATE_THAI   = /\A\d{1,2}\/\d{1,2}\/\d{4}\z/
  
  # เว็บ
  URL         = %r{\Ahttps?://[a-zA-Z0-9.-]+(?:/[^\s]*)?\z}
  IP_ADDRESS  = /\A(?:\d{1,3}\.){3}\d{1,3}\z/
  
  # Username/Password
  USERNAME    = /\A[a-zA-Z][a-zA-Z0-9_]{2,19}\z/
  STRONG_PW   = /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}\z/
  
  # Thai specific
  THAI_ID     = /\A\d{13}\z/  # เลขบัตรประชาชน
  POSTAL_CODE = /\A\d{5}\z/
end

# ทดสอบ
test_cases = {
  "123" => Patterns::INTEGER,
  "3.14" => Patterns::FLOAT,
  "user@test.com" => Patterns::EMAIL,
  "0812345678" => Patterns::THAI_PHONE,
  "2024-01-15" => Patterns::DATE_ISO,
  "john_doe123" => Patterns::USERNAME
}

test_cases.each do |value, pattern|
  result = value.match?(pattern) ? "✓" : "✗"
  puts "#{result} #{value}"
end
```

---

## ขั้นตอนที่ 343: Advanced Regex - Conditional Patterns

```ruby
# Backreferences ใน pattern
# \1 อ้างอิง capture group ที่ 1

# หา palindromes แบบง่าย
words = ["level", "ruby", "radar", "hello", "noon", "world"]
words.each do |word|
  if word == word.reverse
    puts "#{word} เป็น palindrome"
  end
end

# Regex backreference - หา HTML tags ที่ matching
html = "<b>bold</b> <i>italic</i> <b>another bold</i>"
html.scan(/<(\w+)>.*?<\/\1>/) do |tag|
  puts "Valid tag: #{tag[0]}"
end
# => Valid tag: b
# => Valid tag: i

# ตัวอย่าง: ตรวจสอบ quotes matching
def balanced_quotes?(str)
  str.match?(/\A(?:[^'"]*(?:"[^"]*"[^'"]*|'[^']*'[^'"]*)*)*\z/)
end

# ตัวอย่าง: หาคำที่ซ้ำ (repeated words)
text = "the the quick brown fox fox jumped"
puts text.scan(/\b(\w+)\s+\1\b/).map(&:first).inspect
# => ["the", "fox"]
```

---

## ขั้นตอนที่ 344: Regex กับ String Methods อื่นๆ

```ruby
# String#delete กับ pattern (ไม่ใช่ regex แต่คล้าย)
puts "hello, world!".delete("aeiou")   # => "hll, wrld!"

# String#count - นับตัวอักษร
puts "hello, world!".count("aeiou")   # => 3

# String#squeeze - ลดตัวซ้ำ
puts "aaabbbccc".squeeze   # => "abc"

# การ validate password แบบซับซ้อน
def validate_password(password)
  errors = []
  
  errors << "ต้องมีอย่างน้อย 8 ตัวอักษร" unless password.length >= 8
  errors << "ต้องมีตัวพิมพ์ใหญ่" unless password =~ /[A-Z]/
  errors << "ต้องมีตัวพิมพ์เล็ก" unless password =~ /[a-z]/
  errors << "ต้องมีตัวเลข" unless password =~ /\d/
  errors << "ต้องมีอักขระพิเศษ" unless password =~ /[@$!%*?&]/
  
  errors.empty? ? "รหัสผ่านถูกต้อง" : errors.join(", ")
end

passwords = ["weak", "Better1", "Good1@pass", "MyP@ssw0rd!"]
passwords.each do |pw|
  puts "#{pw}: #{validate_password(pw)}"
end

# ใช้ Regex กับ heredoc
pattern = <<~REGEX.chomp
  (?x)
  \d{4}    # year
  -\d{2}   # month
  -\d{2}   # day
REGEX

puts Regexp.new(pattern).match?("2024-01-15")  # => true
```

---

## ขั้นตอนที่ 345: สรุปและ Best Practices

```ruby
# 1. ใช้ %r{} เมื่อ pattern มี /
url_pattern = %r{https?://[^/]+(/\S*)?}
# ดีกว่า /https?:\/\/[^\/]+\/\S*/ (อ่านยาก)

# 2. ใช้ x flag สำหรับ pattern ซับซ้อน
email_pattern = /
  \A                    # จุดเริ่มต้น
  [a-zA-Z0-9._%+-]+    # local part
  @                     # @
  [a-zA-Z0-9.-]+       # domain
  \.                    # จุด
  [a-zA-Z]{2,}         # TLD
  \z                    # จุดสิ้นสุด
/x

# 3. ตั้งชื่อ capture groups เสมอสำหรับ complex patterns
date_pattern = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/

# 4. Test regex ก่อนใช้งาน
def test_regex(pattern, test_cases)
  test_cases.each do |input, expected|
    result = input.match?(pattern)
    status = result == expected ? "✓" : "✗"
    puts "#{status} #{input.inspect} => #{result} (expected: #{expected})"
  end
end

test_regex(/\A\d{5}\z/, {
  "12345" => true,
  "1234" => false,
  "123456" => false,
  "abcde" => false
})

# 5. ระวัง ReDoS (Regular Expression Denial of Service)
# หลีกเลี่ยง nested quantifiers เช่น (a+)+
```

---

## แบบฝึกหัด 25 ข้อ

### ข้อ 1-5: พื้นฐาน

**ข้อ 1:** เขียน regex เพื่อตรวจสอบว่า string มีตัวเลขอย่างน้อย 1 ตัวหรือไม่
```ruby
# เฉลย
def has_digit?(str)
  str.match?(/\d/)
end

puts has_digit?("hello123")  # => true
puts has_digit?("hello")     # => false
```

**ข้อ 2:** เขียน regex เพื่อ extract ตัวเลขทั้งหมดจาก string
```ruby
# เฉลย
def extract_numbers(str)
  str.scan(/\d+/).map(&:to_i)
end

puts extract_numbers("I have 3 cats and 5 dogs").inspect
# => [3, 5]
```

**ข้อ 3:** ใช้ gsub เพื่อแทนที่ช่องว่างหลายๆ ช่องด้วยช่องว่างเดียว
```ruby
# เฉลย
def normalize_spaces(str)
  str.gsub(/\s+/, ' ').strip
end

puts normalize_spaces("hello   world   ruby")
# => "hello world ruby"
```

**ข้อ 4:** เขียน regex ตรวจสอบว่า string เป็น palindrome หรือไม่
```ruby
# เฉลย
def palindrome?(str)
  clean = str.downcase.gsub(/[^a-z]/, '')
  clean == clean.reverse
end

puts palindrome?("racecar")    # => true
puts palindrome?("A man a plan a canal Panama")  # => true
```

**ข้อ 5:** ใช้ scan เพื่อหาทุก word ที่ขึ้นต้นด้วยตัวใหญ่
```ruby
# เฉลย
str = "Hello World from Ruby Programming"
capitals = str.scan(/\b[A-Z]\w*/)
puts capitals.inspect
# => ["Hello", "World", "Ruby", "Programming"]
```

### ข้อ 6-10: Capture Groups

**ข้อ 6:** Parse วันที่รูปแบบ DD/MM/YYYY และแปลงเป็น YYYY-MM-DD
```ruby
# เฉลย
def reformat_date(date_str)
  if date_str =~ /(\d{2})\/(\d{2})\/(\d{4})/
    "#{$3}-#{$2}-#{$1}"
  end
end

puts reformat_date("15/01/2024")  # => "2024-01-15"
```

**ข้อ 7:** Extract ชื่อและนามสกุลจาก "Smith, John"
```ruby
# เฉลย
def parse_name(name_str)
  md = name_str.match(/(?<last>\w+),\s*(?<first>\w+)/)
  return nil unless md
  { first: md[:first], last: md[:last] }
end

result = parse_name("Smith, John")
puts "#{result[:first]} #{result[:last]}"  # => "John Smith"
```

**ข้อ 8:** ดึง domain name จาก email addresses
```ruby
# เฉลย
def extract_domain(email)
  email.match(/(?<=@)[^.]+\.[a-z]+/)&.to_s
end

puts extract_domain("user@example.com")   # => "example.com"
puts extract_domain("admin@test.co.th")   # => "test.co"
```

**ข้อ 9:** หาคำทั้งหมดที่มีความยาว 4-6 ตัวอักษร
```ruby
# เฉลย
str = "I like Ruby very much and enjoy coding"
words = str.scan(/\b\w{4,6}\b/)
puts words.inspect
# => ["like", "Ruby", "very", "much", "enjoy", "coding"]
```

**ข้อ 10:** Replace ตัวเลขในข้อความด้วยคำว่า [NUMBER]
```ruby
# เฉลย
def hide_numbers(str)
  str.gsub(/\d+/, '[NUMBER]')
end

puts hide_numbers("I have 3 cats and 10 dogs worth $500")
# => "I have [NUMBER] cats and [NUMBER] dogs worth $[NUMBER]"
```

### ข้อ 11-15: Named Captures และ Advanced

**ข้อ 11:** Parse IP address และแยกแต่ละ octet
```ruby
# เฉลย
def parse_ip(ip)
  pattern = /\A(?<a>\d{1,3})\.(?<b>\d{1,3})\.(?<c>\d{1,3})\.(?<d>\d{1,3})\z/
  md = ip.match(pattern)
  return nil unless md
  [md[:a], md[:b], md[:c], md[:d]].map(&:to_i)
end

puts parse_ip("192.168.1.100").inspect  # => [192, 168, 1, 100]
```

**ข้อ 12:** ตรวจสอบ strong password (ต้องมีตัวใหญ่, เล็ก, เลข, สัญลักษณ์, 8+ ตัว)
```ruby
# เฉลย
def strong_password?(password)
  return false if password.length < 8
  return false unless password =~ /[A-Z]/
  return false unless password =~ /[a-z]/
  return false unless password =~ /\d/
  return false unless password =~ /[@$!%*?&]/
  true
end

puts strong_password?("MyP@ss1")      # => false (7 chars)
puts strong_password?("MyP@ssw0rd")   # => true
```

**ข้อ 13:** Extract hashtags จาก tweet
```ruby
# เฉลย
def extract_hashtags(tweet)
  tweet.scan(/#\w+/).map { |tag| tag[1..] }
end

tweet = "เรียน #Ruby และ #Rails กับ #Programming ดีมาก!"
puts extract_hashtags(tweet).inspect
# => ["Ruby", "Rails", "Programming"]
```

**ข้อ 14:** Mask credit card number (เช่น 4111111111111111 => ****-****-****-1111)
```ruby
# เฉลย
def mask_card(number)
  digits = number.gsub(/\D/, '')
  return "Invalid" unless digits.length == 16
  digits.gsub(/\A(\d{12})(\d{4})\z/, '****-****-****-\2')
end

puts mask_card("4111111111111111")  # => "****-****-****-1111"
puts mask_card("4111-1111-1111-1111")  # => "****-****-****-1111"
```

**ข้อ 15:** หาประโยคทั้งหมดจาก paragraph
```ruby
# เฉลย
def extract_sentences(text)
  text.scan(/[^.!?]+[.!?]/).map(&:strip)
end

para = "Ruby is great! I love it. Do you use Ruby? It's awesome!"
puts extract_sentences(para).inspect
```

### ข้อ 16-20: Lookahead/Lookbehind

**ข้อ 16:** หาราคาทั้งหมด (ตัวเลขที่นำหน้าด้วย ฿ หรือ $)
```ruby
# เฉลย
def extract_prices(text)
  text.scan(/(?<=[฿$])\d+(?:\.\d{2})?/)
end

text = "ราคา ฿100.00 หรือ $3.50 ลดเหลือ ฿89"
puts extract_prices(text).inspect
# => ["100.00", "3.50", "89"]
```

**ข้อ 17:** หาคำที่ตามด้วย "ing" แต่ไม่รวม "ing" ด้วย
```ruby
# เฉลย
text = "I am running and jumping while singing"
words = text.scan(/\w+(?=ing\b)/)
puts words.inspect  # => ["runn", "jump", "sing"]
```

**ข้อ 18:** แทนที่ตัวเลขทีอยู่หน้า "%" ด้วย percentage format
```ruby
# เฉลย
def format_percentages(text)
  text.gsub(/(\d+(?:\.\d+)?)(?=%)/) { |n| "%.1f" % n.to_f }
end

puts format_percentages("discount 20% and tax 7%")
# => "discount 20.0% and tax 7.0%"
```

**ข้อ 19:** ลบ HTML comments ออกจาก string
```ruby
# เฉลย
def remove_html_comments(html)
  html.gsub(/<!--.*?-->/m, '')
end

html = "<p>Hello <!-- this is a comment --> World</p>"
puts remove_html_comments(html)
# => "<p>Hello  World</p>"
```

**ข้อ 20:** Extract สี hex จาก CSS
```ruby
# เฉลย
def extract_hex_colors(css)
  css.scan(/#(?:[0-9a-fA-F]{3}|[0-9a-fA-F]{6})\b/)
end

css = "color: #fff; background: #FF5733; border: #abc"
puts extract_hex_colors(css).inspect
# => ["#fff", "#FF5733", "#abc"]
```

### ข้อ 21-25: Advanced Applications

**ข้อ 21:** สร้าง simple tokenizer สำหรับ math expression
```ruby
# เฉลย
def tokenize(expr)
  tokens = []
  expr.scan(/\d+(?:\.\d+)?|[+\-*\/()]|\w+/) do |token|
    type = case token
    when /\A\d+(?:\.\d+)?\z/ then :number
    when /\A[+\-*\/]\z/ then :operator
    when /\A[()]\z/ then :paren
    else :variable
    end
    tokens << { type: type, value: token }
  end
  tokens
end

puts tokenize("3 + 4 * (x - 2)").inspect
```

**ข้อ 22:** Parse key=value config file
```ruby
# เฉลย
def parse_config(content)
  result = {}
  content.lines.each do |line|
    line.strip!
    next if line.empty? || line.start_with?('#')
    if md = line.match(/^(?<key>\w+)\s*=\s*(?<value>.+)$/)
      result[md[:key]] = md[:value].strip
    end
  end
  result
end

config_text = """
# Database settings
host = localhost
port = 5432
name = myapp_db
"""

puts parse_config(config_text).inspect
# => {"host"=>"localhost", "port"=>"5432", "name"=>"myapp_db"}
```

**ข้อ 23:** แปลง markdown bold/italic เป็น HTML
```ruby
# เฉลย
def markdown_to_html(text)
  text
    .gsub(/\*\*(.+?)\*\*/, '<strong>\1</strong>')
    .gsub(/\*(.+?)\*/, '<em>\1</em>')
    .gsub(/_(.+?)_/, '<em>\1</em>')
end

md = "Hello **World** and *Ruby* is _awesome_"
puts markdown_to_html(md)
# => "Hello <strong>World</strong> and <em>Ruby</em> is <em>awesome</em>"
```

**ข้อ 24:** หาคำที่ปรากฏซ้ำในข้อความ
```ruby
# เฉลย
def find_repeated_words(text)
  words = text.downcase.scan(/\b\w+\b/)
  freq = words.tally
  freq.select { |_, count| count > 1 }.sort_by { |_, v| -v }.to_h
end

text = "the quick brown fox jumps over the lazy dog the fox"
puts find_repeated_words(text).inspect
# => {"the"=>3, "fox"=>2}
```

**ข้อ 25:** สร้าง simple template engine
```ruby
# เฉลย
class SimpleTemplate
  def initialize(template)
    @template = template
  end

  def render(vars = {})
    @template.gsub(/\{\{(\w+)\}\}/) do |match|
      key = $1.to_sym
      vars.fetch(key, match)
    end
  end
end

template = SimpleTemplate.new(
  "สวัสดี {{name}}! คุณอายุ {{age}} ปี อาศัยอยู่ที่ {{city}}"
)

result = template.render(name: "สมชาย", age: 25, city: "กรุงเทพ")
puts result
# => "สวัสดี สมชาย! คุณอายุ 25 ปี อาศัยอยู่ที่ กรุงเทพ"
```

---

## สรุปบทที่ 16

ในบทนี้เราได้เรียนรู้:
- **ซินแทกซ์พื้นฐาน**: `/pattern/`, flags (i, m, x)
- **ตัวดำเนินการ**: `=~`, `match`, `match?`, `scan`, `gsub`, `sub`
- **Character Classes**: `[abc]`, `\d`, `\w`, `\s` และ negations
- **Quantifiers**: `*`, `+`, `?`, `{n,m}` และ non-greedy versions
- **Anchors**: `^`, `$`, `\A`, `\z`, `\b`
- **Capture Groups**: `()`, `(?:)`, `(?<name>)`
- **Named Captures**: `(?<year>\d{4})` และการใช้ `md[:name]`
- **Lookahead/Lookbehind**: `(?=)`, `(?!)`, `(?<=)`, `(?<!)`
- **Global Variables**: `$~`, `$1`, `$2`, `$&`, `$'`, `` $` ``
- **Multiline**: การใช้ `m` flag
- **Non-greedy**: `*?`, `+?`, `{n,m}?`
- **ตัวอย่างจริง**: Email, phone, URL, log parsing, template engine

Regular Expressions เป็นเครื่องมือที่ทรงพลังมาก การฝึกฝนบ่อยๆ จะช่วยให้ใช้งานได้อย่างเชี่ยวชาญ
