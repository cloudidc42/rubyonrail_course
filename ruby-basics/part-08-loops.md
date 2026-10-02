# ตอนที่ 8: Loops and Iterators (ขั้นตอนที่ 121-145)

## บทนำ

การวนลูปเป็นหัวใจหลักของการเขียนโปรแกรม Ruby มีวิธีการวนลูปที่หลากหลาย ตั้งแต่ `while` loop แบบดั้งเดิม ไปจนถึง functional iterators ที่ทรงพลังอย่าง `map`, `select`, `reduce` ซึ่งเป็นรูปแบบที่ Rubyist นิยมใช้มากกว่า

ในบทนี้เราจะเรียนรู้:
- while, until, for loop
- loop do
- times, upto, downto
- each และตัวแปรของมัน
- Functional iterators: map, select, reject, reduce
- Advanced iterators: find, all?, any?, flat_map, zip
- Lazy enumerators
- การ chain iterators

---

## ขั้นตอนที่ 121: while Loop

```ruby
# while loop ทำงานตราบเท่าที่ condition เป็น true
count = 0
while count < 5
  puts count
  count += 1
end
# => 0, 1, 2, 3, 4

# while กับ break
i = 0
while true
  break if i >= 5
  puts i
  i += 1
end

# while กับ next (skip iteration)
i = 0
while i < 10
  i += 1
  next if i.even?   # ข้ามเลขคู่
  puts i
end
# => 1, 3, 5, 7, 9

# while loop คืนค่า nil โดยปกติ
# แต่คืนค่าหลัง break ถ้ามี break value
result = while true
  break "done" if count >= 5
  count += 1
end
puts result   # => done

# Nested while loops
i = 1
while i <= 3
  j = 1
  while j <= 3
    print "#{i}*#{j}=#{i*j}  "
    j += 1
  end
  puts
  i += 1
end
```

---

## ขั้นตอนที่ 122: until Loop

`until condition` ทำงานตราบเท่าที่ condition เป็น false (ตรงข้าม while)

```ruby
# until loop
count = 0
until count >= 5
  puts count
  count += 1
end
# => 0, 1, 2, 3, 4

# until กับ user input simulation
password = ""
until password == "secret"
  print "ใส่รหัสผ่าน: "
  password = "secret"  # simulate input
  puts password
end
puts "รหัสผ่านถูกต้อง!"

# begin/end until (do-while equivalent)
# ทำงานอย่างน้อย 1 ครั้งก่อนตรวจสอบ condition
attempts = 0
begin
  puts "พยายาม #{attempts + 1}"
  attempts += 1
end until attempts >= 3
# => พยายาม 1, พยายาม 2, พยายาม 3

# while vs until เมื่อไหรใช้อะไร?
# ใช้ while เมื่อ condition เป็น positive (ทำในขณะที่...)
# ใช้ until เมื่อ condition เป็น negative (ทำจนกว่า...)

# ตัวอย่าง: menunggu response
retries = 0
max_retries = 3
until retries >= max_retries
  response = simulate_api_call
  break if response[:success]
  retries += 1
  puts "Retry #{retries}..."
end
```

---

## ขั้นตอนที่ 123: for Loop

for loop ใน Ruby ไม่ค่อยได้ใช้ เพราะ `each` ดีกว่า แต่ควรรู้จัก

```ruby
# for loop กับ Range
for i in 1..5
  puts i
end
# => 1, 2, 3, 4, 5

# for loop กับ Array
fruits = ["apple", "banana", "cherry"]
for fruit in fruits
  puts fruit
end

# ข้อแตกต่างสำคัญ: for loop ไม่สร้าง new scope!
for i in 1..3
  x = i * 2
end
puts x   # => 6 (accessible outside!)

# แต่ each สร้าง new scope
[1, 2, 3].each do |i|
  y = i * 2
end
# puts y  # NameError! (ไม่ accessible)

# เพราะนี้ Rubyist แนะนำให้ใช้ each แทน for
# for ใช้ each ภายในอยู่แล้ว
# for i in array เหมือน array.each { |i| ... }
```

---

## ขั้นตอนที่ 124: loop do

`loop do` คือ infinite loop ที่ต้องใช้ `break` เพื่อออก

```ruby
# loop do - infinite loop
count = 0
loop do
  puts count
  count += 1
  break if count >= 5
end
# => 0, 1, 2, 3, 4

# loop do สำหรับ game loop
# score = 0
# loop do
#   input = get_user_input
#   break if input == "quit"
#   score += process_input(input)
#   puts "Score: #{score}"
# end

# loop do กับ next
count = 0
loop do
  count += 1
  break if count > 10
  next if count.even?
  puts count
end
# => 1, 3, 5, 7, 9

# loop do เป็น method รับ block
# เทียบเท่ากับ while true
count = 0
Kernel.loop do
  break if count >= 3
  puts count
  count += 1
end

# ใช้กับ Enumerator
enum = [1, 2, 3].each
loop do
  puts enum.next
end
# StopIteration จะถูก rescue โดยอัตโนมัติ
# => 1, 2, 3 แล้วหยุด
```

---

## ขั้นตอนที่ 125: break, next, redo

```ruby
# break - ออกจาก loop ทันที
[1, 2, 3, 4, 5].each do |n|
  break if n == 3
  puts n
end
# => 1, 2

# break กับ value
result = [1, 2, 3, 4, 5].each do |n|
  break n * 10 if n == 3
end
puts result   # => 30

# next - ข้าม iteration ปัจจุบัน
[1, 2, 3, 4, 5].each do |n|
  next if n.even?
  puts n
end
# => 1, 3, 5

# redo - ทำ iteration ปัจจุบันซ้ำ
count = 0
(1..3).each do |i|
  count += 1
  redo if count == 2 && i == 1  # ทำ i=1 ซ้ำอีกครั้ง
  puts "i=#{i}, count=#{count}"
  break if count > 5  # safety
end

# break/next ใน nested loops
(1..3).each do |i|
  (1..3).each do |j|
    next if j == 2     # ข้าม j=2 ใน inner loop
    break if i == 2    # ออก inner loop เมื่อ i=2
    puts "#{i}, #{j}"
  end
end
# => 1,1  1,3  (i=2 ออก inner loop ทันที)

# catch/throw สำหรับ breaking outer loop
catch(:done) do
  (1..5).each do |i|
    (1..5).each do |j|
      throw :done if i * j > 10
      puts "#{i} * #{j} = #{i * j}"
    end
  end
end
```

---

## ขั้นตอนที่ 126: times

```ruby
# times - ทำ n ครั้ง
5.times do
  puts "Hello!"
end

# times กับ block parameter (0-based index)
5.times do |i|
  puts "การทำซ้ำ ##{i + 1}"
end
# => การทำซ้ำ #1, #2, #3, #4, #5

# times คืนค่า receiver (Integer)
result = 3.times { |i| puts i }
puts result   # => 3

# ใช้บ่อยสำหรับ repetitive tasks
def create_test_data(n)
  n.times.map { |i| { id: i + 1, name: "User #{i + 1}" } }
end

users = create_test_data(5)
puts users.first.inspect   # => {:id=>1, :name=>"User 1"}

# times.map - สร้าง array
squares = 10.times.map { |i| i ** 2 }
puts squares.inspect
# => [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# Enumerator
enum = 3.times
puts enum.next   # => 0
puts enum.next   # => 1
puts enum.next   # => 2
```

---

## ขั้นตอนที่ 127: upto, downto, step

```ruby
# upto - นับขึ้น
1.upto(5) { |n| print "#{n} " }
puts
# => 1 2 3 4 5

# downto - นับลง
5.downto(1) { |n| print "#{n} " }
puts
# => 5 4 3 2 1

# step - นับตาม step ที่กำหนด
1.step(10, 2) { |n| print "#{n} " }
puts
# => 1 3 5 7 9

10.step(1, -3) { |n| print "#{n} " }
puts
# => 10 7 4 1

# Range step
(1..10).step(2) { |n| print "#{n} " }
puts
# => 1 3 5 7 9

(0.0..1.0).step(0.25) { |n| print "#{n} " }
puts
# => 0.0 0.25 0.5 0.75 1.0

# ตัวอย่างใช้งาน
# สร้างตาราง multiplication
puts "ตาราง 2"
1.upto(10) { |n| puts "2 x #{n} = #{2 * n}" }

# นับถอยหลัง
puts "นับถอยหลัง:"
10.downto(1) { |n| print "#{n}... " }
puts "Go!"

# ขั้นบันได
steps = 1.step(100, 10).to_a
puts steps.inspect
# => [1, 11, 21, 31, 41, 51, 61, 71, 81, 91]
```

---

## ขั้นตอนที่ 128: each - Iterator หลัก

`each` เป็น iterator ที่ใช้บ่อยที่สุดใน Ruby

```ruby
# each กับ Array
fruits = ["apple", "banana", "cherry"]
fruits.each do |fruit|
  puts fruit
end

# each กับ Hash
person = { name: "Alice", age: 30, city: "Bangkok" }
person.each do |key, value|
  puts "#{key}: #{value}"
end

# each กับ Range
(1..5).each { |n| puts n }

# each กับ String (iterates characters)
"hello".each_char { |c| print "#{c}-" }
puts
# => h-e-l-l-o-

# each_byte
"hello".each_byte { |b| print "#{b} " }
puts
# => 104 101 108 108 111

# each_line
"line1\nline2\nline3".each_line { |l| puts l.chomp }

# each คืนค่า receiver
arr = [1, 2, 3]
result = arr.each { |n| n * 2 }
puts result.equal?(arr)   # => true (same object)

# ใช้ each กับ external iterator
enum = [1, 2, 3].each
puts enum.next   # => 1
puts enum.next   # => 2
puts enum.next   # => 3
# enum.next  # => StopIteration
```

---

## ขั้นตอนที่ 129: each_with_index

```ruby
# each_with_index - วนลูปพร้อม index
fruits = ["apple", "banana", "cherry"]

fruits.each_with_index do |fruit, index|
  puts "#{index}: #{fruit}"
end
# => 0: apple
# => 1: banana
# => 2: cherry

# map.with_index - เพิ่ม index ให้ map
result = fruits.map.with_index(1) do |fruit, index|
  "#{index}. #{fruit.capitalize}"
end
puts result.inspect
# => ["1. Apple", "2. Banana", "3. Cherry"]

# each_with_index กับ Hash
{ a: 1, b: 2, c: 3 }.each_with_index do |(key, value), index|
  puts "#{index}: #{key}=#{value}"
end

# ใช้สำหรับสร้าง numbered lists
def numbered_list(items, start: 1)
  items.each_with_index.map do |item, i|
    "#{i + start}. #{item}"
  end
end

items = ["เรียน Ruby", "เรียน Rails", "สร้าง project"]
puts numbered_list(items).join("\n")
```

---

## ขั้นตอนที่ 130: each_with_object

```ruby
# each_with_object - วนลูปพร้อม accumulator object
numbers = [1, 2, 3, 4, 5]

# สร้าง Hash จาก Array
hash = numbers.each_with_object({}) do |n, h|
  h[n] = n ** 2
end
puts hash.inspect
# => {1=>1, 2=>4, 3=>9, 4=>16, 5=>25}

# สร้าง Array จาก Array (filter)
evens = numbers.each_with_object([]) do |n, arr|
  arr << n if n.even?
end
puts evens.inspect   # => [2, 4]

# ความแตกต่างจาก reduce:
# each_with_object: object ไม่เปลี่ยน (mutates ใน place)
# reduce: คืนค่า accumulator ใหม่ในแต่ละรอบ

# ตัวอย่างจริง: สร้าง lookup table
words = ["hello", "world", "ruby", "programming"]
length_groups = words.each_with_object(Hash.new { |h, k| h[k] = [] }) do |word, groups|
  groups[word.length] << word
end
puts length_groups.inspect
# => {5=>["hello", "world"], 4=>["ruby"], 11=>["programming"]}

# จัดกลุ่ม orders by status
orders = [
  { id: 1, status: :pending },
  { id: 2, status: :completed },
  { id: 3, status: :pending },
  { id: 4, status: :cancelled }
]

grouped = orders.each_with_object(Hash.new { |h, k| h[k] = [] }) do |order, groups|
  groups[order[:status]] << order[:id]
end
puts grouped.inspect
```

---

## ขั้นตอนที่ 131: map / collect

`map` (หรือ `collect`) แปลงทุก element และคืนค่า Array ใหม่

```ruby
numbers = [1, 2, 3, 4, 5]

# map พื้นฐาน
squares = numbers.map { |n| n ** 2 }
puts squares.inspect   # => [1, 4, 9, 16, 25]

# map กับ String
names = ["alice", "bob", "charlie"]
capitalized = names.map(&:capitalize)
puts capitalized.inspect   # => ["Alice", "Bob", "Charlie"]

# map กับ Hash
people = [
  { name: "Alice", age: 25 },
  { name: "Bob", age: 30 },
  { name: "Carol", age: 35 }
]

names_only = people.map { |p| p[:name] }
puts names_only.inspect   # => ["Alice", "Bob", "Carol"]

# map กับ index
indexed = names.map.with_index(1) { |name, i| "#{i}. #{name}" }
puts indexed.inspect   # => ["1. alice", "2. bob", "3. charlie"]

# map! - แปลง in-place
arr = [1, 2, 3]
arr.map! { |n| n * 10 }
puts arr.inspect   # => [10, 20, 30]

# Practical examples
prices_usd = [10.0, 25.0, 50.0]
exchange_rate = 35
prices_thb = prices_usd.map { |p| (p * exchange_rate).round(2) }
puts prices_thb.inspect   # => [350.0, 875.0, 1750.0]

# แปลงข้อมูลสำหรับ API response
users = [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }]
formatted = users.map do |user|
  { userId: user[:id], displayName: user[:name].upcase }
end
puts formatted.inspect
```

---

## ขั้นตอนที่ 132: select / filter และ reject

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# select / filter - เลือก elements ที่ตรงเงื่อนไข
evens = numbers.select { |n| n.even? }
puts evens.inspect   # => [2, 4, 6, 8, 10]

odds = numbers.select(&:odd?)
puts odds.inspect    # => [1, 3, 5, 7, 9]

# reject - ตรงข้ามกับ select
non_evens = numbers.reject { |n| n.even? }
puts non_evens.inspect   # => [1, 3, 5, 7, 9]

# select กับ Hash
scores = { Alice: 85, Bob: 62, Carol: 91, Dave: 47 }
passing = scores.select { |_, score| score >= 70 }
puts passing.inspect   # => {:Alice=>85, :Carol=>91}

# select! - modify in-place
arr = [1, 2, 3, 4, 5]
arr.select! { |n| n > 3 }
puts arr.inspect   # => [4, 5]

# Practical examples
products = [
  { name: "Apple", price: 20, in_stock: true },
  { name: "Banana", price: 15, in_stock: false },
  { name: "Cherry", price: 50, in_stock: true },
  { name: "Date", price: 100, in_stock: false }
]

available = products.select { |p| p[:in_stock] }
affordable = products.select { |p| p[:price] < 60 }
in_stock_and_affordable = products.select { |p| p[:in_stock] && p[:price] < 60 }

puts available.map { |p| p[:name] }.inspect       # => ["Apple", "Cherry"]
puts in_stock_and_affordable.map { |p| p[:name] }.inspect  # => ["Apple"]
```

---

## ขั้นตอนที่ 133: reduce / inject

`reduce` (หรือ `inject`) รวบรวมทุก elements เป็นค่าเดียว

```ruby
numbers = [1, 2, 3, 4, 5]

# reduce พื้นฐาน - sum
total = numbers.reduce(0) { |sum, n| sum + n }
puts total   # => 15

# reduce โดยไม่มี initial value
total2 = numbers.reduce { |sum, n| sum + n }
puts total2   # => 15

# ใช้ symbol สำหรับ operator
puts numbers.reduce(:+)   # => 15
puts numbers.reduce(:*)   # => 120
puts numbers.reduce(10, :+)  # => 25 (10 + 15)

# inject เป็น alias ของ reduce
puts numbers.inject(:+)   # => 15

# หาค่ามากที่สุดด้วย reduce
max = numbers.reduce { |m, n| m > n ? m : n }
puts max   # => 5

# หาค่าน้อยที่สุด
min = numbers.reduce { |m, n| m < n ? m : n }
puts min   # => 1

# สร้าง Hash ด้วย reduce
squared = numbers.reduce({}) { |hash, n| hash.merge(n => n ** 2) }
puts squared.inspect
# => {1=>1, 2=>4, 3=>9, 4=>16, 5=>25}

# Practical: คำนวณราคาสินค้า
cart = [
  { name: "Apple", price: 20, qty: 3 },
  { name: "Banana", price: 15, qty: 5 },
  { name: "Cherry", price: 50, qty: 2 }
]

total = cart.reduce(0) { |sum, item| sum + item[:price] * item[:qty] }
puts total   # => 235

# คำนวณ factorial
def factorial(n)
  (1..n).reduce(1, :*)
end

puts factorial(5)    # => 120
puts factorial(10)   # => 3628800
```

---

## ขั้นตอนที่ 134: find / detect

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# find / detect - หา element แรกที่ตรงเงื่อนไข
first_even = numbers.find { |n| n.even? }
puts first_even   # => 2

first_big = numbers.find { |n| n > 7 }
puts first_big   # => 8

# find_index - หา index ของ element
index = numbers.find_index { |n| n > 7 }
puts index   # => 7

# find กับ Hash
products = [
  { id: 1, name: "Apple" },
  { id: 2, name: "Banana" },
  { id: 3, name: "Cherry" }
]

product = products.find { |p| p[:id] == 2 }
puts product.inspect   # => {:id=>2, :name=>"Banana"}

# find ที่ไม่พบ - คืนค่า nil
not_found = numbers.find { |n| n > 100 }
puts not_found.inspect   # => nil

# find พร้อม default (using ifnone proc)
ifnone = -> { "ไม่พบ" }
result = numbers.find(ifnone) { |n| n > 100 }
puts result   # => ไม่พบ

# find_all เหมือน select
find_all_evens = numbers.find_all { |n| n.even? }
puts find_all_evens.inspect   # => [2, 4, 6, 8, 10]
```

---

## ขั้นตอนที่ 135: all?, any?, none?, count

```ruby
numbers = [1, 2, 3, 4, 5, 6]

# all? - ทุก element ตรงเงื่อนไขหรือไม่
puts numbers.all? { |n| n > 0 }     # => true
puts numbers.all? { |n| n.even? }   # => false

# any? - มีอย่างน้อยหนึ่ง element ที่ตรงเงื่อนไข
puts numbers.any? { |n| n > 5 }     # => true
puts numbers.any? { |n| n > 10 }    # => false

# none? - ไม่มี element ใดตรงเงื่อนไข
puts numbers.none? { |n| n > 10 }   # => true
puts numbers.none? { |n| n > 3 }    # => false

# one? - มีเพียงหนึ่ง element ที่ตรงเงื่อนไข
puts numbers.one? { |n| n == 3 }    # => true
puts numbers.one? { |n| n.even? }   # => false

# count - นับ elements ที่ตรงเงื่อนไข
puts numbers.count                   # => 6
puts numbers.count { |n| n.even? }  # => 3
puts numbers.count(3)                # => 1

# ตัวอย่างจริง
students = [
  { name: "Alice", grade: 95, passed: true },
  { name: "Bob", grade: 55, passed: false },
  { name: "Carol", grade: 80, passed: true },
  { name: "Dave", grade: 40, passed: false }
]

puts students.all? { |s| s[:name].length > 2 }     # => true
puts students.any? { |s| s[:grade] > 90 }           # => true
puts students.none? { |s| s[:grade] == 100 }        # => true
puts students.count { |s| s[:passed] }              # => 2

# all? / any? / none? โดยไม่มี block
# ตรวจสอบว่าทุก element เป็น truthy
[1, 2, 3].all?         # => true
[1, nil, 3].all?       # => false
[nil, false].any?      # => false
[1, nil].any?          # => true
[nil, false].none?     # => true
```

---

## ขั้นตอนที่ 136: flat_map

```ruby
# flat_map = map + flatten (1 level)
nested = [[1, 2], [3, 4], [5, 6]]

# map ให้ nested
mapped = nested.map { |arr| arr.map { |n| n * 2 } }
puts mapped.inspect   # => [[2, 4], [6, 8], [10, 12]]

# flat_map ให้ flat
flat_mapped = nested.flat_map { |arr| arr.map { |n| n * 2 } }
puts flat_mapped.inspect   # => [2, 4, 6, 8, 10, 12]

# flat_map กับ single element
sentences = ["hello world", "foo bar", "ruby programming"]
words = sentences.flat_map { |s| s.split }
puts words.inspect
# => ["hello", "world", "foo", "bar", "ruby", "programming"]

# ตัวอย่างจริง: แปลง categories
categories = [
  { name: "Fruit", items: ["apple", "banana"] },
  { name: "Veggie", items: ["carrot", "broccoli", "spinach"] }
]

all_items = categories.flat_map { |cat| cat[:items] }
puts all_items.inspect
# => ["apple", "banana", "carrot", "broccoli", "spinach"]

# flat_map vs flatten vs map
users_with_roles = [
  { name: "Alice", roles: ["admin", "editor"] },
  { name: "Bob", roles: ["viewer"] }
]

all_roles = users_with_roles.flat_map { |u| u[:roles] }
puts all_roles.inspect   # => ["admin", "editor", "viewer"]

# chain_map_flatten
chain_result = users_with_roles.map { |u| u[:roles] }.flatten
puts chain_result.inspect   # => ["admin", "editor", "viewer"]
```

---

## ขั้นตอนที่ 137: zip

```ruby
# zip - รวม arrays เข้าด้วยกัน
names = ["Alice", "Bob", "Carol"]
scores = [85, 92, 78]
grades = ["B", "A", "C"]

# zip กับ arrays
combined = names.zip(scores)
puts combined.inspect
# => [["Alice", 85], ["Bob", 92], ["Carol", 78]]

combined2 = names.zip(scores, grades)
puts combined2.inspect
# => [["Alice", 85, "B"], ["Bob", 92, "A"], ["Carol", 78, "C"]]

# zip กับ block
names.zip(scores) do |name, score|
  puts "#{name}: #{score}"
end

# แปลงเป็น Hash
name_score_hash = names.zip(scores).to_h
puts name_score_hash.inspect
# => {"Alice"=>85, "Bob"=>92, "Carol"=>78}

# zip กับ arrays ที่ยาวไม่เท่ากัน
a = [1, 2, 3, 4, 5]
b = ["a", "b", "c"]
puts a.zip(b).inspect
# => [[1, "a"], [2, "b"], [3, "c"], [4, nil], [5, nil]]

# ตัวอย่าง: สร้าง report
months = %w[ม.ค. ก.พ. มี.ค.]
sales = [100000, 120000, 95000]
expenses = [80000, 90000, 85000]

months.zip(sales, expenses).each do |month, sale, expense|
  profit = sale - expense
  puts "#{month}: รายได้ #{sale}, ค่าใช้จ่าย #{expense}, กำไร #{profit}"
end
```

---

## ขั้นตอนที่ 138: Chaining Iterators

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Chain map และ select
result = numbers
  .select { |n| n.odd? }
  .map { |n| n ** 2 }
puts result.inspect   # => [1, 9, 25, 49, 81]

# Chain multiple operations
words = ["  hello  ", "WORLD", "  Ruby  ", "programming  "]
clean = words
  .map(&:strip)
  .map(&:downcase)
  .select { |w| w.length > 4 }
  .sort
puts clean.inspect
# => ["hello", "programming", "world"]

# Complex pipeline
students = [
  { name: "Alice", score: 85, active: true },
  { name: "Bob", score: 62, active: false },
  { name: "Carol", score: 91, active: true },
  { name: "Dave", score: 75, active: true },
  { name: "Eve", score: 55, active: false }
]

top_active_students = students
  .select { |s| s[:active] }
  .select { |s| s[:score] >= 70 }
  .sort_by { |s| -s[:score] }
  .first(2)
  .map { |s| "#{s[:name]} (#{s[:score]})" }

puts top_active_students.inspect
# => ["Carol (91)", "Alice (85)"]

# then / yield_self สำหรับ pipeline (Ruby 2.6+)
result = [1, 2, 3, 4, 5]
  .select(&:odd?)
  .then { |arr| arr.sum }
  .then { |sum| "Sum of odds: #{sum}" }
puts result   # => Sum of odds: 9
```

---

## ขั้นตอนที่ 139: Lazy Enumerators

Lazy enumerators ไม่คำนวณค่าจนกว่าจะถูกขอ ช่วยประหยัด memory

```ruby
# ปัญหากับ eager evaluation
# (1..Float::INFINITY).select { |n| n.even? }.first(5)
# ↑ จะวนลูปไปเรื่อย ๆ ไม่มีสิ้นสุด!

# lazy evaluation แก้ปัญหานี้
result = (1..Float::INFINITY).lazy.select { |n| n.even? }.first(5)
puts result.inspect   # => [2, 4, 6, 8, 10]

# lazy กับ map
result = (1..Float::INFINITY).lazy
  .select { |n| n % 3 == 0 }
  .map { |n| n ** 2 }
  .first(5)
puts result.inspect   # => [9, 36, 81, 144, 225]

# lazy.take แทน first
result = (1..Float::INFINITY).lazy
  .select(&:prime_like?)  
  .take(5)
  .to_a

# Enumerator::Lazy
lazy_odds = (1..Float::INFINITY).lazy.select(&:odd?)
puts lazy_odds.first(5).inspect   # => [1, 3, 5, 7, 9]

# Custom lazy enumerator
def fibonacci
  Enumerator.new do |y|
    a, b = 0, 1
    loop do
      y << a
      a, b = b, a + b
    end
  end
end

fibs = fibonacci.lazy
puts fibs.first(10).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

puts fibs.select { |n| n.even? }.first(5).inspect
# => [0, 2, 8, 34, 144]

# ประโยชน์ของ lazy:
# 1. ทำงานกับ infinite collections
# 2. ประหยัด memory สำหรับ large datasets
# 3. หยุดทำงานเมื่อได้ผลลัพธ์ที่ต้องการ
```

---

## ขั้นตอนที่ 140: Custom Enumerators

```ruby
# สร้าง Enumerator ด้วย Enumerator.new
counter = Enumerator.new do |y|
  i = 0
  loop do
    y << i
    i += 1
  end
end

puts counter.next   # => 0
puts counter.next   # => 1
puts counter.next   # => 2
puts counter.take(5).inspect  # => [3, 4, 5, 6, 7]

# สร้าง method ที่ return Enumerator ถ้าไม่มี block
def my_each(arr)
  return to_enum(:my_each, arr) unless block_given?
  arr.each { |item| yield item }
end

# ใช้แบบ block
my_each([1, 2, 3]) { |n| puts n }

# ใช้แบบ Enumerator
enum = my_each([1, 2, 3])
puts enum.map { |n| n * 2 }.inspect   # => [2, 4, 6]

# Chained Enumerators
letters = ('a'..'z').each
numbers = (1..26).each

result = letters.zip(numbers).first(5)
puts result.inspect
# => [["a", 1], ["b", 2], ["c", 3], ["d", 4], ["e", 5]]

# Enumerator::Chain (Ruby 2.6+)
chain = [1, 2, 3].each + [4, 5, 6].each
puts chain.to_a.inspect   # => [1, 2, 3, 4, 5, 6]

# หรือใช้ chain method
nums = [1, 2].chain([3, 4], [5, 6])
puts nums.to_a.inspect   # => [1, 2, 3, 4, 5, 6]
```

---

## ขั้นตอนที่ 141: Enumerable Module

```ruby
# Enumerable มี methods มากกว่า 50+ methods
# ต้องการแค่ implement each

class WordCollection
  include Enumerable

  def initialize
    @words = []
  end

  def add(word)
    @words << word
    self
  end

  def each(&block)
    @words.each(&block)
  end
end

collection = WordCollection.new
collection.add("ruby").add("rails").add("sinatra").add("hanami")

# ตอนนี้ได้ methods ทั้งหมดจาก Enumerable!
puts collection.map(&:upcase).inspect
# => ["RUBY", "RAILS", "SINATRA", "HANAMI"]

puts collection.select { |w| w.length > 4 }.inspect
# => ["rails", "sinatra", "hanami"]

puts collection.sort.inspect
# => ["hanami", "rails", "ruby", "sinatra"]

puts collection.min_by(&:length)   # => ruby
puts collection.max_by(&:length)   # => sinatra
puts collection.count { |w| w.include?("a") }  # => 3

# Comparable module ทำให้ compare ได้
class Student
  include Comparable
  attr_accessor :name, :gpa

  def initialize(name, gpa)
    @name = name
    @gpa = gpa
  end

  def <=>(other)
    gpa <=> other.gpa
  end

  def to_s
    "#{name}(#{gpa})"
  end
end

students = [
  Student.new("Alice", 3.8),
  Student.new("Bob", 3.5),
  Student.new("Carol", 3.9)
]

puts students.sort.map(&:to_s).inspect
puts students.max.to_s   # => Carol(3.9)
puts students.min.to_s   # => Bob(3.5)
```

---

## ขั้นตอนที่ 142: group_by, partition, tally

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# group_by - จัดกลุ่มตามเงื่อนไข
grouped = numbers.group_by { |n| n % 3 }
puts grouped.inspect
# => {1=>[1, 4, 7, 10], 2=>[2, 5, 8], 0=>[3, 6, 9]}

# partition - แบ่งเป็น 2 กลุ่ม
evens, odds = numbers.partition { |n| n.even? }
puts evens.inspect   # => [2, 4, 6, 8, 10]
puts odds.inspect    # => [1, 3, 5, 7, 9]

# tally - นับความถี่ (Ruby 2.7+)
fruits = ["apple", "banana", "apple", "cherry", "banana", "apple"]
counts = fruits.tally
puts counts.inspect
# => {"apple"=>3, "banana"=>2, "cherry"=>1}

# tally_by (ไม่มีใน Ruby - ใช้ group_by แทน)
# ใช้ group_by แทน
scores = [85, 92, 78, 95, 62, 88, 71]
grade_groups = scores.group_by do |score|
  case score
  when 90..100 then "A"
  when 80..89 then "B"
  when 70..79 then "C"
  else "F"
  end
end
puts grade_groups.inspect
# => {"B"=>[85, 88], "A"=>[92, 95], "C"=>[78, 71], "F"=>[62]}

# each_slice - แบ่ง array เป็นกลุ่ม ๆ
(1..10).each_slice(3) { |slice| puts slice.inspect }
# => [1, 2, 3]
# => [4, 5, 6]
# => [7, 8, 9]
# => [10]

# each_cons - sliding window
(1..5).each_cons(3) { |window| puts window.inspect }
# => [1, 2, 3]
# => [2, 3, 4]
# => [3, 4, 5]
```

---

## ขั้นตอนที่ 143: min, max, sort, minmax

```ruby
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

# min / max
puts numbers.min   # => 1
puts numbers.max   # => 9

# min_by / max_by
words = ["apple", "fig", "banana", "cherry"]
puts words.min_by { |w| w.length }   # => fig
puts words.max_by { |w| w.length }   # => banana

# minmax - คืน [min, max]
puts numbers.minmax.inspect    # => [1, 9]
puts words.minmax_by { |w| w.length }.inspect
# => ["fig", "banana"]

# sort - เรียงลำดับ
puts numbers.sort.inspect
# => [1, 1, 2, 3, 3, 4, 5, 5, 6, 9]

puts numbers.sort.reverse.inspect
# => [9, 6, 5, 5, 4, 3, 3, 2, 1, 1]

# sort กับ block
puts words.sort { |a, b| a.length <=> b.length }.inspect
# => ["fig", "apple", "banana", "cherry"]

# sort_by
puts words.sort_by { |w| [-w.length, w] }.inspect
# => ["banana", "cherry", "apple", "fig"]

# Stable sort (preserves order for equal elements)
students = [
  { name: "Alice", grade: "A" },
  { name: "Bob", grade: "B" },
  { name: "Carol", grade: "A" }
]

# sort_by ใน Ruby เป็น stable sort
sorted = students.sort_by { |s| s[:grade] }
puts sorted.map { |s| s[:name] }.inspect
# => ["Alice", "Carol", "Bob"]
```

---

## ขั้นตอนที่ 144: sum, product, flatten, uniq

```ruby
numbers = [1, 2, 3, 4, 5]

# sum
puts numbers.sum      # => 15
puts numbers.sum { |n| n * 2 }  # => 30

# product (ไม่ใช่ method ตรง ๆ สำหรับ Array)
product = numbers.reduce(:*)
puts product   # => 120

# flatten
nested = [1, [2, 3], [4, [5, 6]]]
puts nested.flatten.inspect      # => [1, 2, 3, 4, 5, 6]
puts nested.flatten(1).inspect   # => [1, 2, 3, 4, [5, 6]]

# uniq - ลบ duplicates
with_dups = [1, 2, 2, 3, 3, 3, 4]
puts with_dups.uniq.inspect   # => [1, 2, 3, 4]

# uniq กับ block
words = ["apple", "Apple", "APPLE", "banana"]
puts words.uniq { |w| w.downcase }.inspect
# => ["apple", "banana"]

# compact - ลบ nil values
with_nils = [1, nil, 2, nil, 3, nil]
puts with_nils.compact.inspect   # => [1, 2, 3]

# combination - สร้าง combinations
puts [1, 2, 3].combination(2).to_a.inspect
# => [[1, 2], [1, 3], [2, 3]]

# permutation
puts [1, 2, 3].permutation(2).to_a.inspect
# => [[1, 2], [1, 3], [2, 1], [2, 3], [3, 1], [3, 2]]

# product (cartesian product)
puts [1, 2].product([3, 4]).inspect
# => [[1, 3], [1, 4], [2, 3], [2, 4]]
```

---

## ขั้นตอนที่ 145: Performance Tips สำหรับ Loops

```ruby
# 1. ใช้ lazy สำหรับ large datasets
(1..1_000_000).lazy
  .select { |n| n.odd? }
  .map { |n| n ** 2 }
  .first(5)
# เร็วกว่าการ evaluate ทั้ง array

# 2. ใช้ each_with_object แทน reduce สำหรับ mutable accumulator
# ช้ากว่า (สร้าง Hash ใหม่ทุก iteration)
result1 = (1..5).reduce({}) { |h, n| h.merge(n => n**2) }

# เร็วกว่า (mutate Hash เดิม)
result2 = (1..5).each_with_object({}) { |n, h| h[n] = n**2 }

# 3. ใช้ map แทน each ที่เก็บค่า
# ไม่ดี
results = []
[1, 2, 3].each { |n| results << n * 2 }

# ดีกว่า
results = [1, 2, 3].map { |n| n * 2 }

# 4. ใช้ count แทน select.length
# ช้า
count1 = [1, 2, 3, 4, 5].select { |n| n.even? }.length

# เร็วกว่า
count2 = [1, 2, 3, 4, 5].count { |n| n.even? }

# 5. ใช้ any? แทน select.any?
# ช้า (scan ทั้ง array)
has_big1 = (1..1000).select { |n| n > 999 }.any?

# เร็วกว่า (หยุดเมื่อพบ)
has_big2 = (1..1000).any? { |n| n > 999 }

# 6. ใช้ flat_map แทน map.flatten
# ช้ากว่า
result1 = [[1,2],[3,4]].map { |a| a.map { |n| n * 2 } }.flatten

# เร็วกว่า
result2 = [[1,2],[3,4]].flat_map { |a| a.map { |n| n * 2 } }

# 7. Symbol to proc (&:method_name) เร็วกว่า block
# ช้ากว่า
names = ["alice", "bob"].map { |n| n.upcase }

# เร็วกว่า
names = ["alice", "bob"].map(&:upcase)
```

---

## แบบฝึกหัด (ขั้นตอนที่ 121-145)

### ข้อที่ 1: FizzBuzz แบบ Functional
เขียน FizzBuzz โดยไม่ใช้ loop ปกติ

```ruby
# เฉลย
result = (1..30).map do |n|
  case
  when (n % 15).zero? then "FizzBuzz"
  when (n % 3).zero? then "Fizz"
  when (n % 5).zero? then "Buzz"
  else n
  end
end
puts result.inspect
```

### ข้อที่ 2: Running Average
คำนวณ running average ของ array

```ruby
# เฉลย
def running_average(numbers)
  numbers.each_with_index.map do |_, i|
    numbers[0..i].sum.to_f / (i + 1)
  end
end

data = [10, 20, 30, 40, 50]
puts running_average(data).map { |v| v.round(2) }.inspect
# => [10.0, 15.0, 20.0, 25.0, 30.0]
```

### ข้อที่ 3: Matrix Transpose
Transpose matrix โดยใช้ iterators

```ruby
# เฉลย
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

transposed = matrix.first.zip(*matrix[1..])
puts transposed.inspect
# => [[1, 4, 7], [2, 5, 8], [3, 6, 9]]

# หรือ
transposed2 = matrix.transpose
```

### ข้อที่ 4: Fibonacci Sequence
สร้าง Fibonacci sequence ด้วย Enumerator

```ruby
# เฉลย
fib = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

puts fib.take(10).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

puts fib.lazy.select { |n| n.even? }.first(5).inspect
# => [0, 2, 8, 34, 144]
```

### ข้อที่ 5: Word Frequency
นับความถี่คำและเรียงลำดับ

```ruby
# เฉลย
def word_frequency(text)
  text.downcase
      .gsub(/[^a-z\s]/, '')
      .split
      .tally
      .sort_by { |_, count| -count }
      .to_h
end

text = "the quick brown fox jumps over the lazy dog the fox"
result = word_frequency(text)
result.each { |word, count| puts "#{word}: #{count}" if count > 1 }
```

### ข้อที่ 6: Group Anagrams
จัดกลุ่ม words ที่เป็น anagram ด้วยกัน

```ruby
# เฉลย
def group_anagrams(words)
  words.group_by { |w| w.chars.sort.join }
       .values
       .select { |group| group.size > 1 }
end

words = ["eat", "tea", "tan", "ate", "nat", "bat"]
puts group_anagrams(words).inspect
# => [["eat", "tea", "ate"], ["tan", "nat"]]
```

### ข้อที่ 7: Pagination
แบ่ง array เป็น pages

```ruby
# เฉลย
def paginate(items, page_size)
  pages = items.each_slice(page_size).to_a
  {
    total: items.size,
    page_count: pages.size,
    pages: pages
  }
end

items = (1..25).to_a
result = paginate(items, 10)
puts "Total: #{result[:total]}"
puts "Pages: #{result[:page_count]}"
result[:pages].each_with_index do |page, i|
  puts "Page #{i + 1}: #{page.inspect}"
end
```

### ข้อที่ 8: Pipeline
สร้าง data pipeline ที่ process records

```ruby
# เฉลย
records = [
  { name: "  Alice  ", age: "25", salary: "50000" },
  { name: "BOB", age: "30", salary: "invalid" },
  { name: "Carol", age: "25", salary: "75000" }
]

processed = records
  .map { |r| r.transform_values { |v| v.to_s.strip } }
  .map { |r| r.merge(name: r[:name].capitalize) }
  .select { |r| r[:salary].match?(/^\d+$/) }
  .map { |r| r.merge(salary: r[:salary].to_i, age: r[:age].to_i) }
  .sort_by { |r| -r[:salary] }

puts processed.inspect
```

### ข้อที่ 9: Intersection และ Difference
หา intersection, union, difference ของ collections

```ruby
# เฉลย
def set_operations(a, b)
  {
    union: (a | b).sort,
    intersection: (a & b).sort,
    difference_a: (a - b).sort,
    difference_b: (b - a).sort
  }
end

set_a = [1, 2, 3, 4, 5]
set_b = [3, 4, 5, 6, 7]
ops = set_operations(set_a, set_b)
ops.each { |operation, result| puts "#{operation}: #{result.inspect}" }
```

### ข้อที่ 10: Recursive Flatten
เขียน recursive flatten เอง

```ruby
# เฉลย
def my_flatten(arr, depth = Float::INFINITY)
  arr.each_with_object([]) do |item, result|
    if item.is_a?(Array) && depth > 0
      result.concat(my_flatten(item, depth - 1))
    else
      result << item
    end
  end
end

nested = [1, [2, [3, [4, [5]]]]]
puts my_flatten(nested).inspect          # => [1, 2, 3, 4, 5]
puts my_flatten(nested, 2).inspect      # => [1, 2, 3, [4, [5]]]
```

### ข้อที่ 11-25: แบบฝึกหัดเพิ่มเติม

**ข้อที่ 11:** Binary Search ด้วย loop

```ruby
def binary_search(arr, target)
  low, high = 0, arr.length - 1

  loop do
    return -1 if low > high
    mid = (low + high) / 2
    
    case arr[mid] <=> target
    when 0 then return mid
    when -1 then low = mid + 1
    when 1 then high = mid - 1
    end
  end
end

sorted = [1, 3, 5, 7, 9, 11, 13, 15]
puts binary_search(sorted, 7)    # => 3
puts binary_search(sorted, 6)    # => -1
```

**ข้อที่ 12:** Bubble Sort

```ruby
def bubble_sort(arr)
  arr = arr.dup
  n = arr.length
  loop do
    swapped = false
    (n - 1).times do |i|
      if arr[i] > arr[i + 1]
        arr[i], arr[i + 1] = arr[i + 1], arr[i]
        swapped = true
      end
    end
    break unless swapped
  end
  arr
end

puts bubble_sort([64, 34, 25, 12, 22, 11, 90]).inspect
```

**ข้อที่ 13:** Sliding Window Maximum

```ruby
def sliding_window_max(arr, k)
  arr.each_cons(k).map(&:max)
end

puts sliding_window_max([1, 3, -1, -3, 5, 3, 6, 7], 3).inspect
# => [3, 3, 5, 5, 6, 7]
```

**ข้อที่ 14:** Flatten Hash to Array

```ruby
def hash_to_sorted_pairs(hash)
  hash.flat_map { |k, v| [k, v] }
end

h = { a: 1, b: 2, c: 3 }
puts hash_to_sorted_pairs(h).inspect   # => [:a, 1, :b, 2, :c, 3]
```

**ข้อที่ 15:** Generate Combinations

```ruby
def power_set(arr)
  [[]].concat(
    (1..arr.length).flat_map { |n| arr.combination(n).to_a }
  )
end

puts power_set([1, 2, 3]).inspect
```

**ข้อที่ 16:** Moving Average

```ruby
def moving_average(data, window)
  data.each_cons(window).map { |window_data| window_data.sum.to_f / window }
end

data = [2, 4, 6, 8, 10, 12]
puts moving_average(data, 3).map { |v| v.round(2) }.inspect
# => [4.0, 6.0, 8.0, 10.0]
```

**ข้อที่ 17:** Nested Hash Builder

```ruby
def build_nested(pairs)
  pairs.each_with_object({}) do |(keys, value), hash|
    keys.reduce(hash) do |h, key|
      h[key] ||= {}
      key == keys.last ? h.tap { h[key] = value } : h[key]
    end
  end
end
```

**ข้อที่ 18:** Caesar Cipher

```ruby
def caesar_cipher(text, shift)
  text.chars.map do |char|
    if char =~ /[a-zA-Z]/
      base = char =~ /[a-z]/ ? 'a'.ord : 'A'.ord
      ((char.ord - base + shift) % 26 + base).chr
    else
      char
    end
  end.join
end

puts caesar_cipher("Hello, World!", 3)   # => Khoor, Zruog!
```

**ข้อที่ 19:** Chunk Array

```ruby
def chunk_by_sign(numbers)
  numbers.chunk_while { |i, j| (i >= 0) == (j >= 0) }.to_a
end

puts chunk_by_sign([1, 2, -1, -2, 3, -4, 5]).inspect
# => [[1, 2], [-1, -2], [3], [-4], [5]]
```

**ข้อที่ 20:** Deep Count

```ruby
def deep_count(nested)
  nested.each_with_object(Hash.new(0)) do |item, counts|
    if item.is_a?(Array)
      deep_count(item).each { |k, v| counts[k] += v }
    else
      counts[item] += 1
    end
  end
end

puts deep_count([1, [2, 3], [2, [3, 3]]]).inspect
# => {1=>1, 2=>2, 3=>3}
```

**ข้อที่ 21:** Lazy Prime Generator

```ruby
def primes
  (2..Float::INFINITY).lazy.select do |n|
    (2..Math.sqrt(n)).none? { |i| n % i == 0 }
  end
end

puts primes.first(10).inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

**ข้อที่ 22:** Stock Price Analyzer

```ruby
def analyze_stocks(prices)
  changes = prices.each_cons(2).map { |a, b| ((b - a).to_f / a * 100).round(2) }
  {
    average: prices.sum.to_f / prices.size,
    max: prices.max,
    min: prices.min,
    max_gain: changes.max,
    max_loss: changes.min,
    volatility: changes.map { |c| c.abs }.sum / changes.size
  }
end

prices = [100, 105, 98, 110, 108, 115, 112]
result = analyze_stocks(prices)
result.each { |k, v| puts "#{k}: #{v.round(2)}" }
```

**ข้อที่ 23:** Text Statistics

```ruby
def text_statistics(text)
  words = text.split(/\s+/)
  sentences = text.split(/[.!?]+/)
  
  {
    chars: text.length,
    words: words.length,
    sentences: sentences.length,
    avg_word_length: (words.map(&:length).sum.to_f / words.length).round(2),
    most_common_word: words.map(&:downcase).tally.max_by { |_, v| v }&.first
  }
end

text = "Ruby is a dynamic programming language. Ruby was designed by Yukihiro Matsumoto. Ruby is great!"
puts text_statistics(text).inspect
```

**ข้อที่ 24:** Interval Merge

```ruby
def merge_intervals(intervals)
  sorted = intervals.sort_by { |interval| interval[0] }
  sorted.each_with_object([]) do |interval, merged|
    if merged.empty? || merged.last[1] < interval[0]
      merged << interval.dup
    else
      merged.last[1] = [merged.last[1], interval[1]].max
    end
  end
end

intervals = [[1,3],[2,6],[8,10],[15,18]]
puts merge_intervals(intervals).inspect
# => [[1, 6], [8, 10], [15, 18]]
```

**ข้อที่ 25:** Number System Converter

```ruby
def convert_base(number, from_base, to_base)
  # แปลงเป็น decimal ก่อน
  decimal = number.to_s.chars.reduce(0) do |result, digit|
    result * from_base + digit.to_i(from_base)
  end
  
  # แปลงจาก decimal เป็น to_base
  return "0" if decimal.zero?
  
  digits = []
  while decimal > 0
    digits.unshift((decimal % to_base).to_s(to_base))
    decimal /= to_base
  end
  digits.join
end

puts convert_base("1010", 2, 10)   # => 10
puts convert_base("FF", 16, 10)    # => 255
puts convert_base("10", 10, 2)     # => 1010
```

---

## สรุปบทที่ 8

| Iterator/Loop | การใช้งาน | คืนค่า |
|---------------|-----------|--------|
| `while` | วนลูปตาม condition | nil หรือ break value |
| `until` | ตรงข้าม while | nil หรือ break value |
| `for` | วนลูปใน range/array | iterable |
| `loop do` | infinite loop | break value |
| `times` | ทำ n ครั้ง | receiver |
| `upto`/`downto` | นับขึ้น/ลง | receiver |
| `each` | วนลูปแต่ละ element | receiver |
| `map` | แปลงทุก element | Array ใหม่ |
| `select` | กรอง elements | Array ใหม่ |
| `reject` | กรองออก elements | Array ใหม่ |
| `reduce` | รวบรวมเป็นค่าเดียว | accumulated value |
| `find` | หา element แรก | element หรือ nil |
| `flat_map` | map + flatten | Array ใหม่ |
| `zip` | รวม arrays | Array of arrays |

**Key Takeaways:**
1. ใช้ `each`, `map`, `select`, `reduce` แทน `while`/`for` เมื่อทำได้
2. `lazy` ช่วยประหยัด memory สำหรับ large datasets
3. Chain iterators ทำให้โค้ดอ่านง่ายและสั้นกว่า
4. `each_with_object` ดีกว่า `reduce` เมื่อต้องการ mutable accumulator
5. `flat_map` เร็วกว่า `map.flatten`

---

*ถัดไป: ตอนที่ 9 - Methods (ขั้นตอนที่ 146-170)*
