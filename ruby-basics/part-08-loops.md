# ตอนที่ 8: Loops และ Iterators (Steps 121-145)

> **เป้าหมาย**: เรียนรู้การวนซ้ำทุกรูปแบบใน Ruby ตั้งแต่ while loop พื้นฐานไปจนถึง lazy enumerators ขั้นสูง

---

## Step 121: while loop — วนซ้ำตามเงื่อนไข

`while` วนซ้ำตราบเท่าที่เงื่อนไขยังเป็น true

```ruby
# รูปแบบพื้นฐาน
count = 1
while count <= 5
  puts "นับ: #{count}"
  count += 1
end
# นับ: 1
# นับ: 2
# นับ: 3
# นับ: 4
# นับ: 5

# while เป็น expression (มีค่า return)
i = 0
result = while i < 3
           i += 1
         end
# result = nil (while คืน nil เสมอ)
```

### while กับ Complex Conditions

```ruby
# อ่านข้อมูลจนกว่าจะครบเงื่อนไข
def read_positive_numbers(limit)
  numbers = []
  while numbers.length < limit
    print "ใส่ตัวเลขบวก (#{numbers.length + 1}/#{limit}): "
    input = gets.chomp.to_i
    if input > 0
      numbers << input
    else
      puts "กรุณาใส่ตัวเลขบวกเท่านั้น"
    end
  end
  numbers
end

# Binary search ด้วย while
def binary_search(arr, target)
  left = 0
  right = arr.length - 1

  while left <= right
    mid = (left + right) / 2
    if arr[mid] == target
      return mid
    elsif arr[mid] < target
      left = mid + 1
    else
      right = mid - 1
    end
  end
  -1  # ไม่พบ
end

sorted = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
puts binary_search(sorted, 11)   # 5
puts binary_search(sorted, 6)    # -1
```

### while modifier

```ruby
# post-condition: ทำงานก่อนแล้วค่อยตรวจสอบ
begin
  puts "ทำงานอย่างน้อยหนึ่งครั้ง"
  count -= 1
end while count > 0

# one-line while
i = 0
i += 1 while i < 5
puts i  # 5
```

---

## Step 122: until loop — วนซ้ำจนกว่าเงื่อนไขจะเป็นจริง

`until` ตรงข้ามกับ `while` — วนซ้ำตราบเท่าที่เงื่อนไข**ยังเป็น false**

```ruby
# until = while not
count = 1
until count > 5
  puts "นับ: #{count}"
  count += 1
end
# นับ: 1 ถึง 5

# until modifier
i = 0
i += 1 until i >= 10
puts i  # 10

# ตัวอย่างจริง: รอจนกว่าจะ ready
retries = 0
until service_ready? || retries >= 3
  puts "รอ service... (#{retries + 1})"
  sleep(1)
  retries += 1
end
```

### เปรียบเทียบ while กับ until

```ruby
# ทั้งสองทำงานเหมือนกัน
x = 0
while x < 5   do x += 1 end  # while: วนซ้ำตราบเท่าที่ true
x = 0
until x >= 5  do x += 1 end  # until: วนซ้ำตราบเท่าที่ false

# เลือกใช้อันที่อ่านง่ายกว่า
# ✅ ชัดเจน
until queue.empty?
  process(queue.pop)
end

# ✅ ก็ชัดเจนเหมือนกัน
while !queue.empty?
  process(queue.pop)
end
```

---

## Step 123: for loop — ไม่ค่อยใช้ใน Ruby

`for` loop มีอยู่ใน Ruby แต่ไม่ค่อยนิยมใช้เพราะ Ruby มี iterator ที่ดีกว่า ข้อสำคัญคือ `for` **ไม่สร้าง scope ใหม่**

```ruby
# for...in
for i in 1..5
  puts i
end

for fruit in ["apple", "banana", "cherry"]
  puts fruit
end

# ⚠️ for ไม่สร้าง scope ใหม่!
for x in [1, 2, 3]
  last = x  # ตัวแปร last ยังอยู่หลัง loop จบ
end
puts last  # 3  (ยังเข้าถึงได้!)
puts x     # 3  (ยังเข้าถึงได้!)

# ✅ each สร้าง scope ใหม่
[1, 2, 3].each do |x|
  last = x
end
# puts last  # NameError: undefined local variable
# puts x     # NameError: undefined local variable
```

### ทำไมไม่นิยมใช้ for

```ruby
# ❌ for loop (Ruby style ไม่ดี)
for i in 0...array.length
  puts array[i]
end

# ✅ each (Ruby way ที่ถูกต้อง)
array.each do |item|
  puts item
end

# ✅ each_with_index ถ้าต้องการ index
array.each_with_index do |item, index|
  puts "#{index}: #{item}"
end
```

---

## Step 124: loop do...end — Infinite Loop

`loop` สร้าง infinite loop ที่ต้องใช้ `break` เพื่อออก

```ruby
# infinite loop พื้นฐาน
loop do
  puts "วนซ้ำตลอดไป"
  break  # ออกทันที (ตัวอย่างเท่านั้น)
end

# รับข้อมูลจนกว่าจะถูกต้อง
def get_valid_age
  loop do
    print "ใส่อายุ (1-120): "
    age = gets.chomp.to_i
    return age if (1..120).cover?(age)
    puts "อายุไม่ถูกต้อง กรุณาลองใหม่"
  end
end

# Game loop
def game_loop
  score = 0
  lives = 3
  
  loop do
    action = get_player_action
    
    case action
    when :quit
      puts "จบเกม! คะแนน: #{score}"
      break
    when :correct
      score += 10
      puts "ถูกต้อง! +10 คะแนน"
    when :wrong
      lives -= 1
      puts "ผิด! เหลือ #{lives} ชีวิต"
      if lives <= 0
        puts "Game Over! คะแนน: #{score}"
        break
      end
    end
  end
end
```

### loop กับ break value

```ruby
# loop สามารถคืนค่าจาก break ได้
result = loop do
  input = gets.chomp
  break input.to_i if input.match?(/^\d+$/)
  puts "กรุณาใส่ตัวเลข"
end

puts "คุณใส่: #{result}"
```

---

## Step 125: times — วนซ้ำตามจำนวนครั้ง

`times` เป็น method ของ Integer ที่ใช้วนซ้ำตามจำนวนครั้งที่กำหนด

```ruby
# วนซ้ำ 5 ครั้ง
5.times do
  puts "Hello!"
end

# มี index (เริ่มจาก 0)
5.times do |i|
  puts "ครั้งที่ #{i + 1}"
end
# ครั้งที่ 1
# ครั้งที่ 2
# ...
# ครั้งที่ 5

# แบบ one-line
3.times { puts "Ruby!" }

# ใช้สร้าง array
squares = []
5.times { |i| squares << (i + 1) ** 2 }
puts squares.inspect  # [1, 4, 9, 16, 25]

# หรือสั้นกว่า
squares = 5.times.map { |i| (i + 1) ** 2 }
puts squares.inspect  # [1, 4, 9, 16, 25]
```

### ตัวอย่างจริง

```ruby
# สร้าง test data
users = 5.times.map do |i|
  {
    id: i + 1,
    name: "User #{i + 1}",
    email: "user#{i + 1}@example.com"
  }
end

users.each { |u| puts "#{u[:id]}: #{u[:name]} <#{u[:email]}>" }
# 1: User 1 <user1@example.com>
# 2: User 2 <user2@example.com>
# ...

# Retry mechanism
def fetch_with_retry(url, max_retries: 3)
  max_retries.times do |attempt|
    begin
      return http_get(url)
    rescue => e
      puts "ครั้งที่ #{attempt + 1} ล้มเหลว: #{e.message}"
      sleep(2 ** attempt)  # exponential backoff
    end
  end
  raise "ไม่สามารถเชื่อมต่อได้หลังจากลองแล้ว #{max_retries} ครั้ง"
end
```

---

## Step 126: upto และ downto — นับขึ้นและนับลง

```ruby
# upto: นับขึ้น
1.upto(5) { |i| print "#{i} " }
# 1 2 3 4 5

# downto: นับลง
5.downto(1) { |i| print "#{i} " }
# 5 4 3 2 1

# countdown
10.downto(0) do |i|
  if i == 0
    puts "ปล่อย! 🚀"
  else
    puts "#{i}..."
  end
  sleep(0.1)
end

# สร้าง multiplication table
1.upto(5) do |i|
  1.upto(5) do |j|
    printf "%4d", i * j
  end
  puts
end
#    1   2   3   4   5
#    2   4   6   8  10
#    3   6   9  12  15
#    4   8  12  16  20
#    5  10  15  20  25

# upto ใช้กับ string ได้ด้วย!
"a".upto("e") { |c| print "#{c} " }
# a b c d e

"A".upto("Z") { |c| print c }
# ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

---

## Step 127: step — วนซ้ำแบบกำหนด step size

```ruby
# step ใช้กับ Numeric
1.step(10, 2) { |i| print "#{i} " }
# 1 3 5 7 9

0.step(1, 0.25) { |i| print "#{i} " }
# 0.0 0.25 0.5 0.75 1.0

10.step(1, -2) { |i| print "#{i} " }
# 10 8 6 4 2

# Range#step
(1..10).step(3) { |i| print "#{i} " }
# 1 4 7 10

# สร้าง array
evens = 0.step(20, 2).to_a
puts evens.inspect  # [0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# ตัวอย่างจริง: สร้าง time slots
def generate_time_slots(start_hour, end_hour, interval_minutes)
  slots = []
  current_minutes = start_hour * 60
  end_minutes = end_hour * 60

  current_minutes.step(end_minutes - interval_minutes, interval_minutes) do |m|
    hour = m / 60
    min = m % 60
    next_m = m + interval_minutes
    next_hour = next_m / 60
    next_min = next_m % 60
    slots << format("%02d:%02d - %02d:%02d", hour, min, next_hour, next_min)
  end
  slots
end

slots = generate_time_slots(9, 17, 30)
slots.each { |s| puts s }
# 09:00 - 09:30
# 09:30 - 10:00
# ...
# 16:30 - 17:00
```

---

## Step 128: each — Iterator พื้นฐาน

`each` เป็น iterator ที่ใช้บ่อยที่สุดใน Ruby วนซ้ำผ่านแต่ละ element

```ruby
# each กับ Array
fruits = ["apple", "banana", "cherry"]
fruits.each do |fruit|
  puts fruit
end
# apple
# banana
# cherry

# each กับ Hash
person = { name: "Alice", age: 30, city: "Bangkok" }
person.each do |key, value|
  puts "#{key}: #{value}"
end
# name: Alice
# age: 30
# city: Bangkok

# each กับ Range
(1..5).each { |n| print "#{n} " }
# 1 2 3 4 5

# each กับ String (แต่ละตัวอักษร)
"Hello".each_char { |c| print "#{c}-" }
# H-e-l-l-o-

# each กับ Integer (each_digit)
12345.digits.reverse.each { |d| print "#{d} " }
# 1 2 3 4 5
```

### each กับ Custom Objects

```ruby
class NumberRange
  include Enumerable  # ทำให้ได้ each และ methods อื่นๆ ฟรี

  def initialize(from, to)
    @from = from
    @to = to
  end

  def each
    current = @from
    while current <= @to
      yield current
      current += 1
    end
  end
end

range = NumberRange.new(1, 5)
range.each { |n| print "#{n} " }  # 1 2 3 4 5
puts range.map { |n| n * 2 }.inspect    # [2, 4, 6, 8, 10]
puts range.select(&:odd?).inspect       # [1, 3, 5]
puts range.sum                          # 15
```

---

## Step 129: each_with_index — วนซ้ำพร้อม Index

```ruby
# each_with_index
fruits = ["apple", "banana", "cherry"]
fruits.each_with_index do |fruit, index|
  puts "#{index + 1}. #{fruit}"
end
# 1. apple
# 2. banana
# 3. cherry

# กำหนด offset ของ index
fruits.each_with_index do |fruit, index|
  puts "#{index + 10}. #{fruit}"  # เริ่มจาก 10
end

# Hash ก็ใช้ได้
hash = { a: 1, b: 2, c: 3 }
hash.each_with_index do |(key, value), index|
  puts "#{index}: #{key} => #{value}"
end
# 0: a => 1
# 1: b => 2
# 2: c => 3

# ตัวอย่างจริง: สร้าง numbered list
def numbered_list(items, start: 1)
  items.each_with_index.map do |item, i|
    "#{i + start}. #{item}"
  end
end

puts numbered_list(["กาแฟ", "ชา", "น้ำเปล่า"])
# 1. กาแฟ
# 2. ชา
# 3. น้ำเปล่า

puts numbered_list(["A", "B", "C"], start: 0).join(", ")
# 0. A, 1. B, 2. C
```

---

## Step 130: each_with_object — วนซ้ำพร้อมสะสมผลลัพธ์

`each_with_object` วนซ้ำและส่ง object ผ่านทุก iteration เพื่อสะสมผลลัพธ์

```ruby
# สะสมใน Hash
words = ["apple", "banana", "cherry", "avocado", "blueberry"]
word_by_letter = words.each_with_object({}) do |word, hash|
  first_letter = word[0]
  hash[first_letter] ||= []
  hash[first_letter] << word
end

puts word_by_letter.inspect
# {"a"=>["apple", "avocado"], "b"=>["banana", "blueberry"], "c"=>["cherry"]}

# สะสมใน Array
numbers = [1, 2, 3, 4, 5, 6]
result = numbers.each_with_object({ odd: [], even: [] }) do |n, acc|
  if n.odd?
    acc[:odd] << n
  else
    acc[:even] << n
  end
end
puts result.inspect  # {:odd=>[1, 3, 5], :even=>[2, 4, 6]}

# เปรียบเทียบกับ reduce
# each_with_object: ดีสำหรับ mutable objects (Array, Hash)
# reduce/inject: ดีสำหรับ immutable values (Integer, String)

# นับความถี่
words2 = ["the", "quick", "brown", "fox", "the", "lazy", "the"]
frequency = words2.each_with_object(Hash.new(0)) do |word, counts|
  counts[word] += 1
end
puts frequency.inspect  # {"the"=>3, "quick"=>1, "brown"=>1, "fox"=>1, "lazy"=>1}
```

---

## Step 131: map / collect — แปลงค่าทุกตัว

`map` (หรือชื่อเดิม `collect`) คืน Array ใหม่ที่ผ่านการแปลงค่าทุกตัว

```ruby
# map พื้นฐาน
numbers = [1, 2, 3, 4, 5]
squares = numbers.map { |n| n ** 2 }
puts squares.inspect  # [1, 4, 9, 16, 25]

# แปลง string
names = ["alice", "bob", "charlie"]
upper = names.map(&:upcase)
puts upper.inspect  # ["ALICE", "BOB", "CHARLIE"]

# แปลง Hash
users = [
  { name: "Alice", age: 30 },
  { name: "Bob", age: 25 },
  { name: "Charlie", age: 35 }
]

# ดึงแค่ชื่อ
names = users.map { |u| u[:name] }
puts names.inspect  # ["Alice", "Bob", "Charlie"]

# เพิ่มข้อมูล
users_with_label = users.map do |u|
  u.merge(label: u[:age] >= 30 ? "Senior" : "Junior")
end
users_with_label.each { |u| puts "#{u[:name]}: #{u[:label]}" }
# Alice: Senior
# Bob: Junior
# Charlie: Senior
```

### map กับ index

```ruby
words = ["hello", "world", "ruby"]
indexed = words.map.with_index(1) { |word, i| "#{i}. #{word}" }
puts indexed.inspect  # ["1. hello", "2. world", "3. ruby"]

# transform_values สำหรับ Hash
prices = { apple: 50, banana: 30, cherry: 80 }
discounted = prices.transform_values { |v| (v * 0.9).round }
puts discounted.inspect  # {:apple=>45, :banana=>27, :cherry=>72}

# transform_keys สำหรับ Hash
symbolize = { "name" => "Alice", "age" => 30 }
result = symbolize.transform_keys(&:to_sym)
puts result.inspect  # {:name=>"Alice", :age=>30}
```

---

## Step 132: select / filter — กรองตามเงื่อนไข

`select` (หรือ `filter`) คืน Array ของ elements ที่ผ่านเงื่อนไข (block คืน true)

```ruby
# select พื้นฐาน
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

evens = numbers.select { |n| n.even? }
puts evens.inspect  # [2, 4, 6, 8, 10]

odds = numbers.select(&:odd?)
puts odds.inspect   # [1, 3, 5, 7, 9]

big = numbers.select { |n| n > 5 }
puts big.inspect    # [6, 7, 8, 9, 10]

# select กับ String array
words = ["apple", "banana", "cherry", "date", "elderberry"]
long_words = words.select { |w| w.length > 5 }
puts long_words.inspect  # ["banana", "cherry", "elderberry"]

# select กับ Hash
inventory = { apple: 5, banana: 0, cherry: 3, date: 0, elderberry: 8 }
in_stock = inventory.select { |_, qty| qty > 0 }
puts in_stock.inspect  # {:apple=>5, :cherry=>3, :elderberry=>8}

# ตัวอย่างจริง: filter users
users = [
  { name: "Alice", age: 28, active: true },
  { name: "Bob", age: 17, active: true },
  { name: "Charlie", age: 35, active: false },
  { name: "Diana", age: 22, active: true }
]

active_adults = users.select { |u| u[:active] && u[:age] >= 18 }
active_adults.each { |u| puts u[:name] }
# Alice
# Diana
```

---

## Step 133: reject — กรองออก (ตรงข้าม select)

`reject` คืน elements ที่ **ไม่ผ่าน** เงื่อนไข

```ruby
numbers = [1, 2, 3, 4, 5, 6]

odds = numbers.reject(&:even?)  # ตรงข้าม select(&:even?)
puts odds.inspect  # [1, 3, 5]

# reject กับ nil values
data = [1, nil, 2, nil, 3, nil, 4]
clean = data.reject(&:nil?)
# หรือ
clean = data.compact  # เหมือนกัน
puts clean.inspect  # [1, 2, 3, 4]

# reject กับ empty strings
mixed = ["hello", "", "world", "  ", "ruby", ""]
non_empty = mixed.reject { |s| s.strip.empty? }
puts non_empty.inspect  # ["hello", "world", "ruby"]

# ตัวอย่างจริง
spam_words = ["buy", "free", "click", "win"]
messages = [
  "Hello, how are you?",
  "Click here to win a free prize!",
  "Meeting at 3pm",
  "Buy now, limited time offer!"
]

clean_messages = messages.reject do |msg|
  spam_words.any? { |word| msg.downcase.include?(word) }
end
clean_messages.each { |m| puts m }
# Hello, how are you?
# Meeting at 3pm
```

---

## Step 134: reduce / inject — สะสมค่าเป็นผลเดียว

`reduce` (หรือ `inject`) รวมทุก element เป็นค่าเดียวโดยใช้ block

```ruby
# reduce พื้นฐาน
numbers = [1, 2, 3, 4, 5]

# หาผลรวม
sum = numbers.reduce(0) { |total, n| total + n }
puts sum  # 15

# หาผลคูณ
product = numbers.reduce(1) { |total, n| total * n }
puts product  # 120

# ใช้ symbol แทน block
sum = numbers.reduce(:+)     # 15
product = numbers.reduce(:*) # 120
max = numbers.reduce { |a, b| a > b ? a : b }  # 5
min = numbers.reduce { |a, b| a < b ? a : b }  # 1

# สร้าง string
words = ["Hello", "World", "from", "Ruby"]
sentence = words.reduce { |s, w| "#{s} #{w}" }
puts sentence  # "Hello World from Ruby"
```

### reduce กับ initial value

```ruby
# กับ initial value
words = ["apple", "banana", "cherry"]
result = words.reduce({}) do |hash, word|
  hash[word] = word.length
  hash
end
puts result.inspect  # {"apple"=>5, "banana"=>6, "cherry"=>6}

# คำนวณสถิติ
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]
stats = numbers.reduce({ sum: 0, min: nil, max: nil, count: 0 }) do |acc, n|
  acc[:sum] += n
  acc[:count] += 1
  acc[:min] = [acc[:min], n].compact.min
  acc[:max] = [acc[:max], n].compact.max
  acc
end
stats[:avg] = stats[:sum].to_f / stats[:count]
puts stats.inspect
# {:sum=>39, :min=>1, :max=>9, :count=>10, :avg=>3.9}
```

### inject เหมือน reduce 100%

```ruby
# inject และ reduce เป็น alias กัน
[1, 2, 3].inject(:+)    # 6
[1, 2, 3].reduce(:+)    # 6

# ทั้งสองเหมือนกันทุกประการ
```

---

## Step 135: find / detect — ค้นหา element แรกที่ตรงเงื่อนไข

`find` (หรือ `detect`) คืน element แรกที่ผ่านเงื่อนไข

```ruby
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

# หา element แรกที่ > 4
first_big = numbers.find { |n| n > 4 }
puts first_big  # 5

# หา element แรกที่เป็นคู่
first_even = numbers.find(&:even?)
puts first_even  # 4

# ถ้าไม่พบคืน nil
result = numbers.find { |n| n > 100 }
puts result.inspect  # nil

# find กับ default value (ใช้ proc)
default = -> { "ไม่พบ" }
result = [1, 2, 3].find(default) { |n| n > 10 }
puts result  # ไม่พบ

# find กับ objects
users = [
  { id: 1, name: "Alice", role: :admin },
  { id: 2, name: "Bob", role: :user },
  { id: 3, name: "Charlie", role: :admin }
]

admin = users.find { |u| u[:role] == :admin }
puts admin[:name]  # Alice (แรกที่พบ)

# find_index ถ้าต้องการ index
idx = users.find_index { |u| u[:name] == "Bob" }
puts idx  # 1
```

---

## Step 136: all? / any? / none? / one? — ตรวจสอบ Collection

```ruby
numbers = [2, 4, 6, 8, 10]

# all? — ทุกตัวผ่านเงื่อนไขไหม
puts numbers.all?(&:even?)       # true
puts numbers.all? { |n| n > 5 } # false

# any? — มีอย่างน้อยหนึ่งตัวที่ผ่านไหม
puts numbers.any? { |n| n > 9 } # true
puts numbers.any? { |n| n > 20 }# false

# none? — ไม่มีตัวไหนผ่านเงื่อนไขเลยไหม
puts numbers.none?(&:odd?)       # true
puts numbers.none? { |n| n > 5 }# false

# one? — มีแค่หนึ่งตัวเท่านั้นที่ผ่านไหม
puts numbers.one? { |n| n == 6 } # true
puts numbers.one?(&:even?)        # false (มีหลายตัว)

# ตัวอย่างจริง
def validate_scores(scores)
  return "ต้องมีคะแนนอย่างน้อย 1 ข้อ" if scores.none? { true }
  return "มีคะแนนติดลบ" if scores.any? { |s| s < 0 }
  return "มีคะแนนเกิน 100" if scores.any? { |s| s > 100 }
  return "ผ่านทุกข้อ" if scores.all? { |s| s >= 50 }
  failed = scores.count { |s| s < 50 }
  "ไม่ผ่าน #{failed} ข้อจาก #{scores.length} ข้อ"
end

puts validate_scores([80, 75, 90])       # ผ่านทุกข้อ
puts validate_scores([80, 45, 90])       # ไม่ผ่าน 1 ข้อจาก 3 ข้อ
puts validate_scores([80, -5, 90])       # มีคะแนนติดลบ
```

---

## Step 137: flat_map — map แล้ว flatten

`flat_map` ทำงานเหมือน `map` แต่ flatten ผลลัพธ์ 1 ระดับ

```ruby
# map ธรรมดา
words = ["hello world", "ruby programming", "flat map"]
result = words.map { |s| s.split(" ") }
puts result.inspect
# [["hello", "world"], ["ruby", "programming"], ["flat", "map"]]

# flat_map
result = words.flat_map { |s| s.split(" ") }
puts result.inspect
# ["hello", "world", "ruby", "programming", "flat", "map"]

# เหมือนกับ map + flatten
puts words.map { |s| s.split }.flatten.inspect
# ["hello", "world", "ruby", "programming", "flat", "map"]

# ตัวอย่างจริง
categories = [
  { name: "Electronics", items: ["Phone", "Laptop", "Tablet"] },
  { name: "Food", items: ["Apple", "Banana", "Cherry"] },
  { name: "Books", items: ["Ruby", "Python"] }
]

all_items = categories.flat_map { |cat| cat[:items] }
puts all_items.inspect
# ["Phone", "Laptop", "Tablet", "Apple", "Banana", "Cherry", "Ruby", "Python"]

# นับ items ต่อ category
categories.each do |cat|
  puts "#{cat[:name]}: #{cat[:items].length} items"
end

# flat_map กับ range
result = [1, 2, 3].flat_map { |n| [n, n * 2] }
puts result.inspect  # [1, 2, 2, 4, 3, 6]
```

---

## Step 138: zip — รวม Arrays เข้าด้วยกัน

`zip` รวม elements จากหลาย arrays เป็น array ของ arrays

```ruby
# zip พื้นฐาน
a = [1, 2, 3]
b = ["a", "b", "c"]
c = [:x, :y, :z]

result = a.zip(b)
puts result.inspect  # [[1, "a"], [2, "b"], [3, "c"]]

result = a.zip(b, c)
puts result.inspect  # [[1, "a", :x], [2, "b", :y], [3, "c", :z]]

# zip กับ block
[1, 2, 3].zip([4, 5, 6]) { |pair| puts pair.sum }
# 5
# 7
# 9

# ตัวอย่างจริง: สร้าง Hash จาก 2 arrays
keys = [:name, :age, :city]
values = ["Alice", 30, "Bangkok"]
person = keys.zip(values).to_h
puts person.inspect  # {:name=>"Alice", :age=>30, :city=>"Bangkok"}

# สร้าง report
students = ["Alice", "Bob", "Charlie"]
scores = [92, 78, 85]
grades = ["A", "C", "B"]

students.zip(scores, grades).each do |name, score, grade|
  puts "#{name}: #{score} คะแนน (#{grade})"
end
# Alice: 92 คะแนน (A)
# Bob: 78 คะแนน (C)
# Charlie: 85 คะแนน (B)
```

---

## Step 139: Method Chaining — เชื่อม methods ต่อกัน

Ruby Enumerable methods ทุกตัวสามารถ chain ต่อกันได้

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Chain หลาย methods
result = numbers
  .select(&:even?)     # [2, 4, 6, 8, 10]
  .map { |n| n ** 2 }  # [4, 16, 36, 64, 100]
  .reject { |n| n > 50 }  # [4, 16, 36]
  .sum  # 56

puts result  # 56

# ตัวอย่างจริง
orders = [
  { id: 1, product: "กาแฟ", price: 50, qty: 3, status: :completed },
  { id: 2, product: "เค้ก", price: 80, qty: 1, status: :pending },
  { id: 3, product: "ชา", price: 40, qty: 2, status: :completed },
  { id: 4, product: "คุกกี้", price: 60, qty: 5, status: :cancelled },
  { id: 5, product: "น้ำ", price: 20, qty: 4, status: :completed }
]

# คำนวณรายได้จาก completed orders เท่านั้น
revenue = orders
  .select { |o| o[:status] == :completed }
  .map { |o| o[:price] * o[:qty] }
  .sum

puts "รายได้รวม: #{revenue} บาท"
# รายได้รวม: 430 บาท

# สรุปแยกตาม status
summary = orders
  .group_by { |o| o[:status] }
  .transform_values { |group| group.map { |o| o[:price] * o[:qty] }.sum }

puts summary.inspect
# {:completed=>430, :pending=>80, :cancelled=>300}
```

---

## Step 140: Lazy Enumerators — ประมวลผลแบบ lazy

`lazy` ทำให้ enumeration ทำงานแบบ lazy (ไม่ประมวลผลจนกว่าจะจำเป็น) มีประโยชน์มากกับ infinite sequences

```ruby
# ปัญหากับ eager evaluation
# ❌ นี้จะคำนวณทั้ง array ก่อน แล้วค่อยเลือก 5 ตัว
result = (1..Float::INFINITY).map { |n| n * 2 }.first(5)
# ❌ จะวนซ้ำตลอดไปไม่หยุด!

# ✅ ใช้ lazy
result = (1..Float::INFINITY).lazy.map { |n| n * 2 }.first(5)
puts result.inspect  # [2, 4, 6, 8, 10]

# ✅ ค้นหาใน infinite sequence
first_triple = (1..Float::INFINITY).lazy
  .map { |n| n ** 3 }
  .find { |n| n > 1000 }
puts first_triple  # 1331 (11^3)

# lazy select
squares_over_50 = (1..Float::INFINITY).lazy
  .map { |n| n * n }
  .select { |n| n > 50 }
  .first(5)
puts squares_over_50.inspect  # [64, 81, 100, 121, 144]
```

### lazy กับ large data

```ruby
# อ่านไฟล์ขนาดใหญ่แบบ lazy
def process_large_file(filename)
  File.open(filename).lazy
    .map(&:chomp)
    .select { |line| line.start_with?("ERROR") }
    .first(10)
end

# สร้าง fibonacci sequence ด้วย lazy
def fibonacci
  Enumerator.new do |yielder|
    a, b = 0, 1
    loop do
      yielder << a
      a, b = b, a + b
    end
  end.lazy
end

# 10 ตัวแรกของ Fibonacci
puts fibonacci.first(10).inspect
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Fibonacci ที่มากกว่า 100 ตัวแรก 5 ตัว
puts fibonacci.select { |n| n > 100 }.first(5).inspect
# [144, 233, 377, 610, 987]

# Prime numbers ด้วย lazy
def primes
  Enumerator.new do |yielder|
    n = 2
    loop do
      yielder << n if (2...n).none? { |i| n % i == 0 }
      n += 1
    end
  end.lazy
end

puts primes.first(10).inspect
# [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

---

## Step 141: break, next, redo, return ใน Loops

### break — ออกจาก loop ทันที

```ruby
# break พื้นฐาน
[1, 2, 3, 4, 5].each do |n|
  break if n == 3
  puts n
end
# 1
# 2

# break กับ value
result = [1, 2, 3, 4, 5].each do |n|
  break "พบ #{n}" if n == 3
  puts n
end
puts result  # "พบ 3"

# break ใน while
i = 0
while true
  break if i >= 5
  i += 1
end
puts i  # 5

# ค้นหาแล้วหยุด
users = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
  { id: 3, name: "Charlie" }
]

found = nil
users.each do |user|
  if user[:id] == 2
    found = user
    break
  end
end
puts found[:name]  # Bob
# (ใช้ find แทนได้: users.find { |u| u[:id] == 2 })
```

### next — ข้ามไป iteration ถัดไป

```ruby
# next พื้นฐาน
[1, 2, 3, 4, 5].each do |n|
  next if n.even?  # ข้ามเลขคู่
  puts n
end
# 1
# 3
# 5

# next กับ complex logic
data = [
  { name: "Alice", score: 85 },
  { name: nil, score: 70 },
  { name: "Charlie", score: -5 },
  { name: "Diana", score: 92 }
]

data.each do |item|
  next if item[:name].nil?        # ข้ามถ้าไม่มีชื่อ
  next if item[:score] < 0        # ข้ามถ้าคะแนนติดลบ
  puts "#{item[:name]}: #{item[:score]}"
end
# Alice: 85
# Diana: 92
```

### redo — ทำ iteration นี้ใหม่

```ruby
# redo ทำ iteration ปัจจุบันใหม่โดยไม่เพิ่ม counter
# ใช้ระวัง! อาจเกิด infinite loop ได้
attempts = 0
[1, 2, 3].each do |n|
  attempts += 1
  if n == 2 && attempts < 5
    redo  # ทำ iteration ของ n=2 ใหม่
  end
  puts "n=#{n}, attempts=#{attempts}"
end
# n=1, attempts=1
# n=2, attempts=5  (redo 3 ครั้ง)
# n=3, attempts=6
```

### return ใน loop

```ruby
# return ออกจาก method ทันที (รวมถึง loop ที่อยู่ใน method)
def find_first_negative(numbers)
  numbers.each do |n|
    return n if n < 0  # ออกจาก method ทันที
  end
  nil  # ถ้าไม่พบ
end

puts find_first_negative([1, 2, -3, 4, -5])  # -3
puts find_first_negative([1, 2, 3]).inspect   # nil
```

---

## Step 142–145: Enumerator และ Custom Iterators

### Step 142: Enumerator

```ruby
# สร้าง Enumerator ด้วย to_enum
enum = [1, 2, 3].to_enum
puts enum.next  # 1
puts enum.next  # 2
puts enum.next  # 3
# enum.next  # StopIteration

# สร้าง custom Enumerator
counter = Enumerator.new do |yielder|
  i = 0
  loop do
    yielder << i
    i += 1
  end
end

puts counter.take(5).inspect  # [0, 1, 2, 3, 4]
puts counter.first(3).inspect  # [0, 1, 2]

# Enumerator::Chain
evens = (0..Float::INFINITY).step(2)
odds  = (1..Float::INFINITY).step(2)
# ไม่สามารถ chain infinite enumerators ได้โดยตรง
```

### Step 143: each_slice และ each_cons

```ruby
# each_slice: แบ่งเป็น chunks
(1..10).each_slice(3) { |group| puts group.inspect }
# [1, 2, 3]
# [4, 5, 6]
# [7, 8, 9]
# [10]

# each_cons: sliding window
(1..5).each_cons(3) { |group| puts group.inspect }
# [1, 2, 3]
# [2, 3, 4]
# [3, 4, 5]

# ตัวอย่างจริง: moving average
def moving_average(data, window)
  data.each_cons(window).map do |window_data|
    window_data.sum.to_f / window
  end
end

prices = [100, 102, 98, 105, 103, 107, 104, 108]
ma3 = moving_average(prices, 3)
puts ma3.map { |v| v.round(2) }.inspect
# [100.0, 101.67, 102.0, 105.0, 104.67, 106.33]
```

### Step 144: group_by และ tally

```ruby
# group_by: จัดกลุ่ม
words = ["apple", "ant", "banana", "bear", "cherry", "cat"]
by_first_letter = words.group_by { |w| w[0] }
puts by_first_letter.inspect
# {"a"=>["apple", "ant"], "b"=>["banana", "bear"], "c"=>["cherry", "cat"]}

# จัดกลุ่มตาม type
data = [1, "hello", 2, "world", :sym, 3.14, :other]
by_type = data.group_by(&:class)
by_type.each { |type, items| puts "#{type}: #{items.inspect}" }

# tally: นับความถี่ (Ruby 2.7+)
votes = ["A", "B", "A", "C", "B", "A", "B", "A"]
tally = votes.tally
puts tally.inspect  # {"A"=>4, "B"=>3, "C"=>1}

# tally_by
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
odd_even_count = numbers.tally_by { |n| n.even? ? :even : :odd }
puts odd_even_count.inspect  # {:odd=>5, :even=>5}
```

### Step 145: min, max, sort, sum และ Enumerable ที่มีประโยชน์

```ruby
numbers = [5, 3, 8, 1, 9, 2, 7, 4, 6]

# min/max
puts numbers.min  # 1
puts numbers.max  # 9

# min_by / max_by
words = ["cherry", "apple", "banana", "date"]
puts words.min_by(&:length)  # date
puts words.max_by(&:length)  # cherry

# minmax / minmax_by
puts numbers.minmax.inspect  # [1, 9]
puts words.minmax_by(&:length).inspect  # ["date", "cherry"]

# sort / sort_by
puts numbers.sort.inspect  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
puts words.sort_by(&:length).inspect  # ["date", "apple", "banana", "cherry"]

# sum
puts numbers.sum   # 45
puts numbers.sum { |n| n * 2 }  # 90

# count
puts numbers.count        # 9
puts numbers.count(&:odd?)  # 5
puts numbers.count { |n| n > 5 }  # 4

# take / drop
puts numbers.first(3).inspect  # [5, 3, 8]
puts numbers.take(3).inspect   # [5, 3, 8]
puts numbers.drop(6).inspect   # [7, 4, 6]
puts numbers.last(3).inspect   # [7, 4, 6]

# take_while / drop_while
puts [1, 2, 3, 4, 1, 2].take_while { |n| n < 4 }.inspect  # [1, 2, 3]
puts [1, 2, 3, 4, 1, 2].drop_while { |n| n < 4 }.inspect  # [4, 1, 2]

# uniq / uniq_by
puts [1, 2, 2, 3, 3, 3].uniq.inspect  # [1, 2, 3]
puts ["apple", "Apple", "APPLE"].uniq { |s| s.downcase }.inspect  # ["apple"]

# flatten
nested = [1, [2, 3], [4, [5, 6]]]
puts nested.flatten.inspect    # [1, 2, 3, 4, 5, 6]
puts nested.flatten(1).inspect # [1, 2, 3, 4, [5, 6]]

# chunk
result = [1, 1, 2, 2, 3, 1, 1].chunk_while { |a, b| a == b }.to_a
puts result.inspect  # [[1, 1], [2, 2], [3], [1, 1]]
```

---

## แบบฝึกหัดตอนที่ 8 (25 ข้อ)

### ข้อ 1-5: while / until

```ruby
# ข้อ 1: Collatz Conjecture
# เขียน method ที่คำนวณ Collatz sequence สำหรับ n
# ถ้า n คู่: n / 2, ถ้า n คี่: n * 3 + 1, หยุดเมื่อ n == 1

def collatz(n)
  return [1] if n == 1
  sequence = [n]
  while n != 1
    n = n.even? ? n / 2 : n * 3 + 1
    sequence << n
  end
  sequence
end

puts collatz(6).inspect   # [6, 3, 10, 5, 16, 8, 4, 2, 1]
puts collatz(27).length   # 112 (ยาวมาก!)

# ข้อ 2: กำลังของ 2 ที่ไม่เกิน n
def powers_of_two_up_to(n)
  powers = []
  power = 1
  while power <= n
    powers << power
    power *= 2
  end
  powers
end

puts powers_of_two_up_to(100).inspect
# [1, 2, 4, 8, 16, 32, 64]

# ข้อ 3: หา GCD (Greatest Common Divisor)
def gcd(a, b)
  while b != 0
    a, b = b, a % b
  end
  a.abs
end

puts gcd(48, 18)  # 6
puts gcd(100, 75) # 25

# ข้อ 4: สร้าง digital root
# digital_root(942) = 9 + 4 + 2 = 15 -> 1 + 5 = 6
def digital_root(n)
  until n < 10
    n = n.digits.sum
  end
  n
end

puts digital_root(942)   # 6
puts digital_root(9999)  # 9

# ข้อ 5: เกม Number Guessing (simulation)
def simulate_guessing(secret, max_attempts)
  attempts = 0
  low = 1
  high = 100
  
  until low > high
    guess = (low + high) / 2
    attempts += 1
    
    if guess == secret
      return "เดาถูก! คือ #{secret} ใช้เวลา #{attempts} ครั้ง"
    elsif guess < secret
      low = guess + 1
    else
      high = guess - 1
    end
    
    break if attempts >= max_attempts
  end
  "เดาไม่ถูก!"
end

puts simulate_guessing(42, 10)
# เดาถูก! คือ 42 ใช้เวลา 6 ครั้ง
```

### ข้อ 6-10: times / upto / step

```ruby
# ข้อ 6: Pascal's Triangle
def pascals_triangle(rows)
  triangle = [[1]]
  (rows - 1).times do |i|
    prev = triangle.last
    new_row = [1]
    (prev.length - 1).times do |j|
      new_row << prev[j] + prev[j + 1]
    end
    new_row << 1
    triangle << new_row
  end
  triangle
end

pascals_triangle(6).each { |row| puts row.join(" ").center(20) }
#          1         
#         1 1        
#        1 2 1       
#       1 3 3 1      
#      1 4 6 4 1     
#    1 5 10 10 5 1   

# ข้อ 7: สร้าง Multiplication Table
def multiplication_table(n)
  1.upto(n) do |i|
    1.upto(n) do |j|
      printf "%4d", i * j
    end
    puts
  end
end

multiplication_table(5)

# ข้อ 8: ตรวจสอบ Prime
def prime?(n)
  return false if n < 2
  return true if n == 2
  return false if n.even?
  3.step(Math.sqrt(n).to_i, 2).none? { |i| n % i == 0 }
end

primes_under_50 = (2..50).select { |n| prime?(n) }
puts primes_under_50.inspect
# [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]

# ข้อ 9: Simpson's Rule integration
def simpsons_rule(a, b, n, &f)
  raise ArgumentError, "n ต้องเป็นจำนวนคู่" if n.odd?
  h = (b - a).to_f / n
  sum = f.call(a) + f.call(b)
  
  1.step(n - 1, 2) { |i| sum += 4 * f.call(a + i * h) }
  2.step(n - 2, 2) { |i| sum += 2 * f.call(a + i * h) }
  
  (h / 3) * sum
end

# คำนวณ ∫₀^π sin(x) dx ≈ 2
result = simpsons_rule(0, Math::PI, 100) { |x| Math.sin(x) }
puts result.round(6)  # 2.000000

# ข้อ 10: Gray Code
def gray_code(n)
  (2**n).times.map { |i| i ^ (i >> 1) }
end

puts gray_code(3).map { |n| n.to_s(2).rjust(3, "0") }.inspect
# ["000", "001", "011", "010", "110", "111", "101", "100"]
```

### ข้อ 11-15: map / select / reduce

```ruby
# ข้อ 11: Word Frequency Counter
def word_frequency(text)
  text.downcase
      .scan(/\b[a-z]+\b/)
      .tally
      .sort_by { |_, count| -count }
      .first(10)
end

text = "the quick brown fox jumps over the lazy dog the fox"
word_frequency(text).each { |word, count| puts "#{word}: #{count}" }

# ข้อ 12: Matrix Operations
def matrix_multiply(a, b)
  rows_a = a.length
  cols_a = a[0].length
  cols_b = b[0].length
  
  Array.new(rows_a) do |i|
    Array.new(cols_b) do |j|
      (0...cols_a).sum { |k| a[i][k] * b[k][j] }
    end
  end
end

a = [[1, 2], [3, 4]]
b = [[5, 6], [7, 8]]
result = matrix_multiply(a, b)
result.each { |row| puts row.inspect }
# [19, 22]
# [43, 50]

# ข้อ 13: Run-Length Encoding
def run_length_encode(str)
  str.chars
     .chunk_while { |a, b| a == b }
     .map { |group| "#{group.length}#{group.first}" }
     .join
end

def run_length_decode(encoded)
  encoded.scan(/(\d+)([a-zA-Z])/)
         .map { |count, char| char * count.to_i }
         .join
end

puts run_length_encode("AABBBCCDDDDEE")  # 2A3B2C4D2E
puts run_length_decode("2A3B2C4D2E")    # AABBBCCDDDDEE

# ข้อ 14: Anagram Grouper
def group_anagrams(words)
  words.group_by { |w| w.chars.sort.join }
       .values
end

puts group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"]).inspect
# [["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]

# ข้อ 15: Running Statistics
def running_statistics(numbers)
  numbers.each_with_object({ values: [], stats: [] }) do |n, acc|
    acc[:values] << n
    values = acc[:values]
    sorted = values.sort
    n_count = values.length
    mean = values.sum.to_f / n_count
    variance = values.sum { |x| (x - mean) ** 2 } / n_count
    
    acc[:stats] << {
      value: n,
      mean: mean.round(2),
      stddev: Math.sqrt(variance).round(2),
      median: n_count.odd? ? sorted[n_count / 2] : (sorted[n_count / 2 - 1] + sorted[n_count / 2]) / 2.0
    }
  end[:stats]
end

stats = running_statistics([4, 2, 7, 1, 9, 3])
stats.each { |s| puts "#{s[:value]}: mean=#{s[:mean]}, median=#{s[:median]}" }
```

### ข้อ 16-20: Enumerator / Lazy

```ruby
# ข้อ 16: Infinite Sequences
def geometric_sequence(first, ratio)
  Enumerator.new do |y|
    current = first
    loop do
      y << current
      current *= ratio
    end
  end.lazy
end

puts geometric_sequence(1, 2).first(8).inspect    # [1, 2, 4, 8, 16, 32, 64, 128]
puts geometric_sequence(100, 0.5).first(5).inspect # [100, 50.0, 25.0, 12.5, 6.25]

# ข้อ 17: Sieve of Eratosthenes
def sieve_of_eratosthenes(limit)
  composite = Array.new(limit + 1, false)
  composite[0] = composite[1] = true
  
  2.upto(Math.sqrt(limit).to_i) do |i|
    unless composite[i]
      (i * i).step(limit, i) { |j| composite[j] = true }
    end
  end
  
  (2..limit).reject { |i| composite[i] }
end

puts sieve_of_eratosthenes(50).inspect
# [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]

# ข้อ 18: Sliding Window Maximum
def sliding_window_max(nums, k)
  nums.each_cons(k).map(&:max)
end

puts sliding_window_max([1, 3, -1, -3, 5, 3, 6, 7], 3).inspect
# [3, 3, 5, 5, 6, 7]

# ข้อ 19: Flatten Nested Array (ไม่ใช้ flatten)
def deep_flatten(arr)
  arr.each_with_object([]) do |item, result|
    if item.is_a?(Array)
      result.concat(deep_flatten(item))
    else
      result << item
    end
  end
end

puts deep_flatten([1, [2, [3, [4]], 5], 6]).inspect  # [1, 2, 3, 4, 5, 6]

# ข้อ 20: Spiral Matrix
def spiral_matrix(n)
  matrix = Array.new(n) { Array.new(n, 0) }
  directions = [[0, 1], [1, 0], [0, -1], [-1, 0]]
  dir = 0
  row, col = 0, 0
  
  (1..n*n).each do |num|
    matrix[row][col] = num
    next_row = row + directions[dir][0]
    next_col = col + directions[dir][1]
    
    if next_row.between?(0, n - 1) && next_col.between?(0, n - 1) && matrix[next_row][next_col] == 0
      row, col = next_row, next_col
    else
      dir = (dir + 1) % 4
      row += directions[dir][0]
      col += directions[dir][1]
    end
  end
  matrix
end

spiral_matrix(4).each { |row| puts row.map { |n| n.to_s.rjust(3) }.join }
#   1  2  3  4
#  12 13 14  5
#  11 16 15  6
#  10  9  8  7
```

### ข้อ 21-25: แบบฝึกหัดขั้นสูง

```ruby
# ข้อ 21: Cartesian Product
def cartesian_product(*arrays)
  arrays.reduce { |acc, arr| acc.flat_map { |x| arr.map { |y| [x, y].flatten } } }
end

puts cartesian_product([1, 2], [3, 4], [5, 6]).length  # 8
puts cartesian_product(["a", "b"], [1, 2]).inspect
# [["a", 1], ["a", 2], ["b", 1], ["b", 2]]

# ข้อ 22: Deep Zip (transpose nested arrays)
def deep_transpose(matrix)
  matrix.transpose
end

m = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
deep_transpose(m).each { |row| puts row.inspect }
# [1, 4, 7]
# [2, 5, 8]
# [3, 6, 9]

# ข้อ 23: Custom Enumerable class
class InfiniteCounter
  include Enumerable
  
  def initialize(start: 0, step: 1)
    @start = start
    @step = step
  end
  
  def each
    current = @start
    loop do
      yield current
      current += @step
    end
  end
  
  def take(n)
    result = []
    each do |val|
      result << val
      break if result.length >= n
    end
    result
  end
end

counter = InfiniteCounter.new(start: 0, step: 5)
puts counter.take(6).inspect  # [0, 5, 10, 15, 20, 25]
puts counter.lazy.select { |n| n % 3 == 0 }.first(5).inspect  # [0, 15, 30, 45, 60]

# ข้อ 24: Memoized Fibonacci ด้วย Enumerator
def fib_memo
  cache = { 0 => 0, 1 => 1 }
  
  Enumerator.new do |y|
    n = 0
    loop do
      cache[n] ||= cache[n - 1] + cache[n - 2]
      y << cache[n]
      n += 1
    end
  end.lazy
end

puts fib_memo.first(15).inspect
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377]

# ข้อ 25: Pipeline Processing
module Pipeline
  def self.process(data, *steps)
    steps.reduce(data) { |result, step| result.then(&step) }
  end
end

data = [1, -2, 3, -4, 5, 6, -7, 8, 9, -10]

result = Pipeline.process(
  data,
  ->(d) { d.select { |n| n > 0 } },          # กรองเฉพาะบวก
  ->(d) { d.map { |n| n ** 2 } },             # ยกกำลัง 2
  ->(d) { d.select { |n| n > 10 } },          # กรองที่ > 10
  ->(d) { d.sort }                             # เรียงลำดับ
)
puts result.inspect  # [25, 36, 64, 81]
```

---

## สรุป Loops และ Iterators ใน Ruby

| Method | การใช้งาน | คืนค่า |
|--------|----------|--------|
| `while` | วนซ้ำตามเงื่อนไข | nil |
| `until` | วนซ้ำจนกว่าเงื่อนไขจะจริง | nil |
| `loop` | infinite loop | nil / break value |
| `times` | วนซ้ำ n ครั้ง | Integer |
| `upto/downto` | นับขึ้น/ลง | Integer |
| `step` | วนซ้ำแบบกำหนด step | Numeric |
| `each` | วนซ้ำผ่าน elements | original collection |
| `map` | แปลงค่า | Array ใหม่ |
| `select` | กรองที่ผ่าน | Array ใหม่ |
| `reject` | กรองที่ไม่ผ่าน | Array ใหม่ |
| `reduce` | สะสมเป็นค่าเดียว | ค่าสุดท้าย |
| `find` | หาตัวแรก | element หรือ nil |
| `flat_map` | map + flatten | Array ใหม่ |
| `zip` | รวม arrays | Array of arrays |
| `group_by` | จัดกลุ่ม | Hash |
| `tally` | นับความถี่ | Hash |
| `lazy` | ประมวลผลแบบ lazy | Lazy Enumerator |

**หลักการที่ควรจำ:**
1. ใน Ruby นิยมใช้ iterator (each, map, select) มากกว่า for/while
2. `lazy` ช่วยประหยัด memory เมื่อทำงานกับ large/infinite data
3. `reduce` ทรงพลังมาก สามารถทำงานได้หลายรูปแบบ
4. Chaining methods ทำให้โค้ดอ่านง่ายและ functional
5. `break` และ `next` ควบคุม flow ใน loops ได้ดี

> ⬅️ [ตอนที่ 7: Control Flow](part-07-control-flow.md) | ➡️ [ตอนที่ 9: Methods](part-09-methods.md)
