# ตอนที่ 8: Loops และ Iterators (Steps 121-145)

## บทนำ

Loops และ Iterators เป็นหัวใจสำคัญของการเขียนโปรแกรม ใน Ruby มีวิธีการวนซ้ำหลากหลายมาก ตั้งแต่ loop แบบดั้งเดิมอย่าง `while` และ `for` จนถึง iterators ที่ทันสมัยอย่าง `map`, `select`, `reduce` ที่เป็น functional programming style

Ruby เน้นใช้ iterators มากกว่า loop แบบดั้งเดิม เพราะอ่านง่ายกว่า, ปลอดภัยกว่า, และแสดงเจตนาได้ชัดเจนกว่า

---

## Step 121: while Loop

### Loop พื้นฐาน

```ruby
# while loop - วนซ้ำตราบที่เงื่อนไขเป็น true
count = 0
while count < 5
  puts "Count: #{count}"
  count += 1
end
# => Count: 0
# => Count: 1
# => Count: 2
# => Count: 3
# => Count: 4
```

```ruby
# while loop สำหรับ user input
puts "ใส่ตัวเลขที่ต้องการ (0 เพื่อออก):"
sum = 0
count = 0

while true
  print "ตัวเลข: "
  num = gets.chomp.to_i
  break if num == 0
  
  sum += num
  count += 1
  puts "  ผลรวมตอนนี้: #{sum}"
end

puts "\nจำนวนที่ใส่: #{count}"
puts "ผลรวม: #{sum}"
puts "เฉลี่ย: #{count > 0 ? sum.to_f / count : 0}"
```

```ruby
# การใช้ break เพื่อออกจาก loop
i = 0
while i < 100
  break if i > 5  # ออกเมื่อ i > 5
  puts i
  i += 1
end
# => 0 1 2 3 4 5

# การใช้ next เพื่อข้ามรอบ
i = 0
while i < 10
  i += 1
  next if i.even?  # ข้ามเลขคู่
  puts i
end
# => 1 3 5 7 9
```

```ruby
# while loop กับ complex condition
temperature = 100
cooling_rate = 0.9

while temperature > 1
  temperature *= cooling_rate
  puts "อุณหภูมิ: #{temperature.round(2)}"
end

# Fibonacci sequence ด้วย while
a, b = 0, 1
while a < 100
  print "#{a} "
  a, b = b, a + b
end
puts
# => 0 1 1 2 3 5 8 13 21 34 55 89
```

---

## Step 122: until Loop

### Loop แบบ Until

```ruby
# until - วนซ้ำตราบที่เงื่อนไขเป็น false (ตรงข้าม while)
count = 0
until count >= 5
  puts "Count: #{count}"
  count += 1
end
# => Count: 0 ถึง Count: 4

# เปรียบเทียบ while และ until
# while:  วนต่อเมื่อ condition เป็น true
# until:  วนต่อเมื่อ condition เป็น false
```

```ruby
# until ดีกว่าสำหรับ negative conditions
queue = ["task1", "task2", "task3"]

until queue.empty?
  task = queue.shift
  puts "กำลังทำ: #{task}"
end
# อ่านว่า: "จนกว่า queue จะว่าง"

# เทียบกับ while
while !queue.empty?  # อ่านยากกว่า
  # ...
end
```

```ruby
# until กับ begin...end (do while equivalent)
# วนอย่างน้อยหนึ่งครั้ง
x = 10

begin
  puts "x = #{x}"
  x += 1
end until x > 15

# => x = 10
# => x = 11
# => x = 12
# => x = 13
# => x = 14
# => x = 15

# begin...end with while ก็ได้เหมือนกัน
y = 10
begin
  puts "y = #{y}"
  y += 1
end while y <= 15
```

---

## Step 123: loop do...end

### Infinite Loop

```ruby
# loop do...end - วนไม่รู้จบ (ต้องใช้ break)
counter = 0
loop do
  counter += 1
  puts counter
  break if counter >= 5
end
# => 1 2 3 4 5

# loop ดีกว่า while true สำหรับ explicit infinite loops
loop do
  print "คำสั่ง: "
  input = gets.chomp
  
  case input
  when "quit", "exit", "q"
    puts "Goodbye!"
    break
  when "help"
    puts "คำสั่งที่รองรับ: quit, help"
  else
    puts "คุณพิมพ์: #{input}"
  end
end
```

```ruby
# loop กับ break returning value
result = loop do
  x = rand(100)
  break x if x > 90  # break คืนค่ากลับ
end

puts "ได้ตัวเลข: #{result}"  # ตัวเลข > 90

# Retry pattern
attempts = 0
result = loop do
  attempts += 1
  outcome = rand < 0.3 ? :success : :fail  # 30% chance
  
  if outcome == :success
    break "สำเร็จใน #{attempts} ครั้ง"
  elsif attempts >= 10
    break "ล้มเหลวหลัง #{attempts} ครั้ง"
  end
end

puts result
```

```ruby
# Event loop pattern
events = ["click", "keypress", "scroll", "click", "quit"]
index = 0

loop do
  event = events[index]
  index += 1
  
  puts "Event: #{event}"
  
  case event
  when "quit"
    puts "Stopping event loop"
    break
  when "click"
    puts "  Handling click..."
  when "keypress"
    puts "  Handling keypress..."
  end
end
```

---

## Step 124: for...in Loop

### for Loop (ไม่นิยมใน Ruby)

```ruby
# for...in loop - Ruby มี แต่ไม่นิยมใช้
for i in 1..5
  puts i
end
# => 1 2 3 4 5

for fruit in ["apple", "banana", "cherry"]
  puts fruit
end

# ทำไม for ไม่นิยม?
# 1. for ไม่สร้าง scope ใหม่ (variable leak)
for x in [1, 2, 3]
  y = x * 2  # y ยังอยู่นอก loop!
end
puts y  # => 6 (leaked variable!)
puts x  # => 3 (leaked variable!)

# แต่ each สร้าง scope ใหม่
[1, 2, 3].each do |x|
  z = x * 2  # z ไม่ leak
end
# puts z  # => NameError!
```

```ruby
# เปรียบเทียบ for vs each
numbers = [1, 2, 3, 4, 5]

# for loop (ไม่นิยม)
for n in numbers
  puts n * 2
end

# each (นิยมกว่า)
numbers.each do |n|
  puts n * 2
end

# ผลลัพธ์เหมือนกัน แต่ each ดีกว่าเพราะ:
# - สร้าง scope ใหม่
# - Rubyist ทำ
# - รองรับ Enumerable methods อื่นๆ
# - อ่านง่ายกว่าในบางบริบท
```

---

## Step 125: Integer#times

### วนซ้ำตามจำนวนครั้ง

```ruby
# times - วนซ้ำ n ครั้ง
5.times { puts "Hello!" }
# => Hello! (5 ครั้ง)

# กับ block variable (0-based index)
5.times do |i|
  puts "ครั้งที่ #{i + 1}"
end
# => ครั้งที่ 1 ถึง ครั้งที่ 5

# สร้าง array ด้วย times
squares = []
10.times { |i| squares << i ** 2 }
puts squares.inspect  # => [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

```ruby
# ใช้งานจริง: Retry mechanism
MAX_RETRIES = 3

def connect_to_server
  3.times do |attempt|
    puts "พยายามเชื่อมต่อครั้งที่ #{attempt + 1}..."
    
    # Simulate random success/failure
    if rand > 0.5
      puts "เชื่อมต่อสำเร็จ!"
      return true
    end
    
    puts "ล้มเหลว, รอก่อน..."
    sleep(0.1) if attempt < 2  # ไม่ sleep ครั้งสุดท้าย
  end
  
  puts "ไม่สามารถเชื่อมต่อได้"
  false
end

connect_to_server
```

```ruby
# times กับ parallel processing simulation
results = []
mutex_like = []  # simulated

10.times do |i|
  result = i ** 3
  results << result
end

puts results.inspect
# => [0, 1, 8, 27, 64, 125, 216, 343, 512, 729]

# times ส่งคืน receiver (Integer)
return_value = 3.times { |i| puts i }
puts return_value  # => 3
```

---

## Step 126: Integer#upto / downto / step

### วนซ้ำแบบ Range

```ruby
# upto - วนจากน้อยไปมาก
1.upto(10) { |i| print "#{i} " }
puts  # => 1 2 3 4 5 6 7 8 9 10

# downto - วนจากมากไปน้อย
10.downto(1) { |i| print "#{i} " }
puts  # => 10 9 8 7 6 5 4 3 2 1

# step - วนด้วย step ที่กำหนด
1.step(20, 3) { |i| print "#{i} " }
puts  # => 1 4 7 10 13 16 19

# step ลง
20.step(1, -3) { |i| print "#{i} " }
puts  # => 20 17 14 11 8 5 2
```

```ruby
# ใช้งานจริง: countdown timer
puts "Countdown:"
10.downto(0) do |second|
  if second == 0
    puts "LAUNCH! 🚀"
  else
    puts "#{second}..."
    sleep(0.1)  # ลดเวลาสำหรับ demo
  end
end

# ตาราง multiplication
print "   "
1.upto(5) { |i| print "#{i.to_s.rjust(4)}" }
puts

1.upto(5) do |i|
  print "#{i}: "
  1.upto(5) { |j| print "#{(i*j).to_s.rjust(4)}" }
  puts
end
```

```ruby
# step กับ Float
0.0.step(1.0, 0.25) { |x| print "#{x} " }
puts
# => 0.0 0.25 0.5 0.75 1.0

# ใช้กับ Numeric#step
(1..5).step(0.5) { |x| print "#{x} " }
puts
# => 1.0 1.5 2.0 2.5 3.0 3.5 4.0 4.5 5.0
```

---

## Step 127: Array#each และ Hash#each

### Iterator พื้นฐาน

```ruby
# Array each
fruits = ["apple", "banana", "cherry"]
fruits.each do |fruit|
  puts "I like #{fruit}"
end

# Hash each
person = { name: "สมชาย", age: 25, city: "กรุงเทพ" }
person.each do |key, value|
  puts "#{key}: #{value}"
end
# => name: สมชาย
# => age: 25
# => city: กรุงเทพ
```

```ruby
# each ส่งคืน receiver
original = [1, 2, 3]
returned = original.each { |x| x * 2 }
puts returned.equal?(original)  # => true (คืน array เดิม)

# เทียบกับ map ที่สร้าง array ใหม่
mapped = original.map { |x| x * 2 }
puts original.inspect  # => [1, 2, 3] (ไม่เปลี่ยน)
puts mapped.inspect    # => [2, 4, 6] (ใหม่)
```

```ruby
# each กับ hash - destructuring
inventory = { apple: 50, banana: 30, cherry: 100 }

inventory.each do |fruit, quantity|
  status = quantity > 40 ? "มีพอ" : "ใกล้หมด"
  puts "#{fruit}: #{quantity} ชิ้น (#{status})"
end

# each_pair (alias ของ each สำหรับ Hash)
{a: 1, b: 2}.each_pair { |k, v| puts "#{k}=#{v}" }
```

---

## Step 128: each_with_index

### วนซ้ำพร้อม Index

```ruby
fruits = ["apple", "banana", "cherry"]

# each_with_index
fruits.each_with_index do |fruit, index|
  puts "#{index + 1}. #{fruit}"
end
# => 1. apple
# => 2. banana
# => 3. cherry

# ค้นหาด้วย each_with_index
target = "banana"
fruits.each_with_index do |fruit, i|
  if fruit == target
    puts "พบ '#{target}' ที่ index #{i}"
  end
end
```

```ruby
# each_with_index สำหรับ Hash
person = { name: "Alice", age: 30, city: "Bangkok" }
person.each_with_index do |(key, value), index|
  puts "#{index}: #{key} = #{value}"
end
# => 0: name = Alice
# => 1: age = 30
# => 2: city = Bangkok

# ใช้ map.with_index เพื่อ transform
result = fruits.map.with_index(1) do |fruit, i|
  "#{i}. #{fruit.capitalize}"
end
puts result.inspect
# => ["1. Apple", "2. Banana", "3. Cherry"]
```

```ruby
# เมื่อต้องการ index ที่เริ่มต้นที่ค่าอื่น
headers = %w[Name Age Email City]
headers.each_with_index do |header, i|
  puts "Column #{i + 65}.chr: #{header}"  # A, B, C, D
end

# หรือ:
headers.each.with_index(65) do |header, ascii|
  puts "Column #{ascii.chr}: #{header}"
end
```

---

## Step 129: each_with_object

### วนซ้ำพร้อมสะสมค่า

```ruby
numbers = [1, 2, 3, 4, 5]

# สะสมค่าลงใน array ใหม่
result = numbers.each_with_object([]) do |n, arr|
  arr << n * 2 if n.odd?
end
puts result.inspect  # => [2, 6, 10]

# สร้าง Hash
word_lengths = %w[hello world ruby].each_with_object({}) do |word, hash|
  hash[word] = word.length
end
puts word_lengths.inspect  # => {"hello"=>5, "world"=>5, "ruby"=>4}
```

```ruby
# เปรียบเทียบกับ reduce
numbers = [1, 2, 3, 4, 5]

# ด้วย reduce (acc ต้อง return ทุกครั้ง)
result_reduce = numbers.reduce([]) do |arr, n|
  arr << n * 2 if n.odd?
  arr  # ต้อง return arr!
end

# ด้วย each_with_object (ไม่ต้อง return object)
result_ewo = numbers.each_with_object([]) do |n, arr|
  arr << n * 2 if n.odd?
  # ไม่ต้อง return arr - มันรู้เองว่า object คือ arr
end

puts result_reduce == result_ewo  # => true
```

```ruby
# ใช้งานจริง: สร้าง grouped data
students = [
  { name: "Alice", grade: "A", score: 95 },
  { name: "Bob", grade: "B", score: 75 },
  { name: "Charlie", grade: "A", score: 90 },
  { name: "Dave", grade: "C", score: 65 },
  { name: "Eve", grade: "B", score: 80 }
]

by_grade = students.each_with_object(Hash.new { |h, k| h[k] = [] }) do |student, groups|
  groups[student[:grade]] << student[:name]
end

by_grade.sort.each do |grade, names|
  puts "Grade #{grade}: #{names.join(', ')}"
end
# => Grade A: Alice, Charlie
# => Grade B: Bob, Eve
# => Grade C: Dave
```

---

## Step 130: map / collect

### แปลง Collection

```ruby
numbers = [1, 2, 3, 4, 5]

# map - แปลงทุก element
squares = numbers.map { |n| n ** 2 }
puts squares.inspect  # => [1, 4, 9, 16, 25]

# collect เป็น alias ของ map
cubes = numbers.collect { |n| n ** 3 }
puts cubes.inspect  # => [1, 8, 27, 64, 125]

# map ส่งคืน array ใหม่เสมอ
puts numbers.inspect  # => [1, 2, 3, 4, 5] (ไม่เปลี่ยน)
```

```ruby
# map กับ method symbol
words = ["hello", "world", "RUBY"]
puts words.map(&:upcase).inspect       # => ["HELLO", "WORLD", "RUBY"]
puts words.map(&:capitalize).inspect   # => ["Hello", "World", "Ruby"]
puts words.map(&:length).inspect       # => [5, 5, 4]

# map กับ conversion
puts ["1", "2", "3"].map(&:to_i).inspect  # => [1, 2, 3]
puts [1, 2, 3].map(&:to_s).inspect        # => ["1", "2", "3"]
puts [1, 2, 3].map(&:to_f).inspect        # => [1.0, 2.0, 3.0]
```

```ruby
# map กับ ข้อมูลซับซ้อน
users = [
  { id: 1, name: "Alice", email: "alice@example.com" },
  { id: 2, name: "Bob", email: "bob@example.com" },
  { id: 3, name: "Charlie", email: "charlie@example.com" }
]

# ดึง field เฉพาะ
emails = users.map { |u| u[:email] }
puts emails.inspect
# => ["alice@example.com", "bob@example.com", "charlie@example.com"]

# แปลง data structure
names_by_id = users.map { |u| [u[:id], u[:name]] }.to_h
puts names_by_id.inspect
# => {1=>"Alice", 2=>"Bob", 3=>"Charlie"}

# map กับ index
indexed = users.map.with_index(1) { |u, i| "#{i}. #{u[:name]}" }
puts indexed.inspect
# => ["1. Alice", "2. Bob", "3. Charlie"]
```

---

## Step 131: select/filter และ reject

### กรอง Collection

```ruby
numbers = (1..10).to_a

# select / filter - เลือกตัวที่ตรงเงื่อนไข
evens = numbers.select { |n| n.even? }
puts evens.inspect  # => [2, 4, 6, 8, 10]

# filter เป็น alias ของ select (Ruby 2.6+)
odds = numbers.filter { |n| n.odd? }
puts odds.inspect   # => [1, 3, 5, 7, 9]

# reject - ตรงข้าม select (ตัดที่ตรงเงื่อนไขออก)
no_evens = numbers.reject { |n| n.even? }
puts no_evens.inspect  # => [1, 3, 5, 7, 9]
```

```ruby
# select กับ string
words = %w[apple Banana CHERRY date Elderberry]
lowercase_words = words.select { |w| w == w.downcase }
puts lowercase_words.inspect  # => ["apple", "date"]

# select กับ nil removal
data = [1, nil, 2, false, 3, nil, 4]
# compact ทำงานเหมือน select { |x| !x.nil? }
puts data.compact.inspect   # => [1, 2, false, 3, 4]

# reject nil AND false
truthy = data.select { |x| x }
puts truthy.inspect  # => [1, 2, 3, 4]
```

```ruby
# ใช้งานจริง: Filter products
products = [
  { name: "Laptop", price: 35000, in_stock: true },
  { name: "Phone", price: 15000, in_stock: false },
  { name: "Tablet", price: 20000, in_stock: true },
  { name: "Watch", price: 8000, in_stock: true },
  { name: "Headphones", price: 5000, in_stock: false }
]

# เฉพาะสินค้าที่มีในสต็อกและราคา <= 25000
affordable_in_stock = products
  .select { |p| p[:in_stock] && p[:price] <= 25000 }
  .map { |p| "#{p[:name]} (#{p[:price]} บาท)" }

puts affordable_in_stock.inspect
# => ["Tablet (20000 บาท)", "Watch (8000 บาท)"]
```

---

## Step 132: reduce / inject

### สะสมค่า

```ruby
numbers = [1, 2, 3, 4, 5]

# reduce / inject - สะสมค่า
sum = numbers.reduce(0) { |acc, n| acc + n }
puts sum  # => 15

# ไม่ระบุ initial value
product = numbers.reduce { |acc, n| acc * n }
puts product  # => 120

# Symbol shorthand
puts numbers.reduce(:+)   # => 15
puts numbers.reduce(:*)   # => 120
puts numbers.inject(:+)   # => 15 (inject = reduce)
```

```ruby
# reduce กับ initial value ประเภทอื่น
words = ["hello", "world", "ruby"]

# สร้าง hash
word_map = words.reduce({}) { |h, w| h.merge(w => w.length) }
puts word_map.inspect  # => {"hello"=>5, "world"=>5, "ruby"=>4}

# สร้าง string
sentence = words.reduce { |s, w| "#{s} #{w}" }
puts sentence  # => hello world ruby

# flatten nested array
nested = [[1, 2], [3, 4], [5, 6]]
flat = nested.reduce([]) { |arr, sub| arr + sub }
puts flat.inspect  # => [1, 2, 3, 4, 5, 6]
```

```ruby
# reduce เพื่อหา max/min โดยไม่ใช้ built-in
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5]

max = numbers.reduce { |m, n| n > m ? n : m }
min = numbers.reduce { |m, n| n < m ? n : m }

puts "Max: #{max}"  # => 9
puts "Min: #{min}"  # => 1

# Compose functions ด้วย reduce
double = ->(x) { x * 2 }
increment = ->(x) { x + 1 }
square = ->(x) { x ** 2 }

pipeline = [double, increment, square]
result = pipeline.reduce(3) { |val, fn| fn.call(val) }
puts result  # => ((3 * 2) + 1)^2 = 7^2 = 49
```

---

## Step 133: flat_map

### map แล้ว flatten

```ruby
# flat_map = map + flatten(1)
sentences = ["hello world", "foo bar baz", "ruby programming"]

# แบบ map แล้ว flatten
words_v1 = sentences.map { |s| s.split }.flatten
puts words_v1.inspect
# => ["hello", "world", "foo", "bar", "baz", "ruby", "programming"]

# แบบ flat_map (เหมือนกัน แต่เร็วกว่า)
words_v2 = sentences.flat_map { |s| s.split }
puts words_v2.inspect  # ผลเหมือนกัน
```

```ruby
# flat_map กับ nested data
users = [
  { name: "Alice", tags: ["admin", "developer"] },
  { name: "Bob", tags: ["developer", "designer"] },
  { name: "Charlie", tags: ["admin"] }
]

# รวม tags ทั้งหมด (มีซ้ำ)
all_tags = users.flat_map { |u| u[:tags] }
puts all_tags.inspect  # => ["admin", "developer", "developer", "designer", "admin"]

# unique tags
unique_tags = all_tags.uniq.sort
puts unique_tags.inspect  # => ["admin", "developer", "designer"]

# flat_map สร้าง pairs
pairs = [1, 2, 3].flat_map { |n| [n, n * 2] }
puts pairs.inspect  # => [1, 2, 2, 4, 3, 6]
```

---

## Step 134: zip

### รวม Array หลายตัว

```ruby
names = ["Alice", "Bob", "Charlie"]
ages  = [25, 30, 35]
cities = ["Bangkok", "Chiang Mai", "Phuket"]

# zip - รวม element by element
combined = names.zip(ages, cities)
puts combined.inspect
# => [["Alice", 25, "Bangkok"], ["Bob", 30, "Chiang Mai"], ["Charlie", 35, "Phuket"]]

# zip กับ block
names.zip(ages) do |name, age|
  puts "#{name} is #{age} years old"
end
```

```ruby
# zip ที่มีความยาวไม่เท่ากัน
a = [1, 2, 3, 4]
b = ["a", "b"]

puts a.zip(b).inspect
# => [[1, "a"], [2, "b"], [3, nil], [4, nil]]

# transpose - inverse ของ zip
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
puts matrix.transpose.inspect
# => [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
```

```ruby
# ใช้งานจริง: สร้าง hash จาก 2 arrays
keys = [:name, :age, :city]
values = ["Alice", 25, "Bangkok"]

hash = keys.zip(values).to_h
puts hash.inspect  # => {:name=>"Alice", :age=>25, :city=>"Bangkok"}

# หรือสั้นกว่า:
hash2 = keys.zip(values).each_with_object({}) { |(k, v), h| h[k] = v }

# แปลง CSV
headers = %w[Name Age Email]
rows = [
  %w[Alice 25 alice@example.com],
  %w[Bob 30 bob@example.com]
]

records = rows.map { |row| headers.zip(row).to_h }
records.each { |r| puts r.inspect }
```

---

## Step 135: find / detect

### ค้นหาตัวแรก

```ruby
numbers = [1, 3, 5, 8, 10, 12, 15]

# find / detect - คืน element แรกที่ตรงเงื่อนไข
first_even = numbers.find { |n| n.even? }
puts first_even  # => 8

# ถ้าไม่พบ คืน nil
puts numbers.find { |n| n > 100 }.inspect  # => nil

# detect เป็น alias ของ find
puts numbers.detect { |n| n > 10 }  # => 12
```

```ruby
# find กับ default value (Proc)
# ถ้าไม่พบ เรียก proc
default = -> { "ไม่พบ" }
result = [1, 3, 5].find(default) { |n| n.even? }
puts result  # => ไม่พบ

# find_index - คืน index แทน element
puts numbers.find_index { |n| n.even? }  # => 3
puts numbers.find_index(10)              # => 4

# rindex - index สุดท้าย
arr = [1, 2, 3, 2, 1]
puts arr.rindex(2)   # => 3
puts arr.rindex { |n| n > 1 }  # => 3
```

```ruby
# ใช้งานจริง
users = [
  { id: 1, name: "Alice", role: "admin" },
  { id: 2, name: "Bob", role: "user" },
  { id: 3, name: "Charlie", role: "admin" }
]

# หา admin คนแรก
first_admin = users.find { |u| u[:role] == "admin" }
puts first_admin[:name]  # => Alice

# หาด้วย ID
def find_user(users, id)
  users.find { |u| u[:id] == id }
end

user = find_user(users, 2)
puts user&.fetch(:name, "Unknown")  # => Bob
```

---

## Step 136: count กับ Block

### นับแบบมีเงื่อนไข

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# count - นับทั้งหมด
puts numbers.count        # => 10

# count ด้วย value
arr = [1, 2, 2, 3, 3, 3]
puts arr.count(3)         # => 3

# count ด้วย block
puts numbers.count { |n| n.even? }   # => 5
puts numbers.count { |n| n > 5 }     # => 5
puts numbers.count { |n| n.odd? && n > 5 }  # => 3 (7, 9)
```

```ruby
# ใช้งานจริง
students = [
  { name: "Alice", score: 92, passed: true },
  { name: "Bob", score: 45, passed: false },
  { name: "Charlie", score: 78, passed: true },
  { name: "Dave", score: 55, passed: false },
  { name: "Eve", score: 88, passed: true }
]

total = students.count
passed = students.count { |s| s[:passed] }
failed = students.count { |s| !s[:passed] }
high_score = students.count { |s| s[:score] >= 80 }

puts "รวม: #{total}"
puts "ผ่าน: #{passed}"
puts "ไม่ผ่าน: #{failed}"
puts "คะแนนสูง (>=80): #{high_score}"
puts "Pass rate: #{(passed.to_f / total * 100).round(1)}%"
```

---

## Step 137: sum กับ Block

### รวมค่าแบบมีเงื่อนไข

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# sum ธรรมดา
puts numbers.sum   # => 55

# sum กับ block (Ruby 2.4+)
puts numbers.sum { |n| n ** 2 }   # => 385 (sum of squares)
puts numbers.sum { |n| n.even? ? n : 0 }  # => 30 (sum of evens)

# sum กับ initial value
puts [1, 2, 3].sum(10)  # => 16
```

```ruby
# ใช้งานจริง: การเงิน
orders = [
  { product: "Laptop", price: 35000, qty: 2 },
  { product: "Phone", price: 15000, qty: 3 },
  { product: "Tablet", price: 20000, qty: 1 }
]

total_items = orders.sum { |o| o[:qty] }
total_value = orders.sum { |o| o[:price] * o[:qty] }

puts "รวมสินค้า: #{total_items} ชิ้น"
puts "มูลค่ารวม: #{total_value} บาท"

# sum กับ string
puts %w[hello world ruby].sum("")  # => "helloworldruby"
```

---

## Step 138: all?, any?, none?, one?

### ตรวจสอบเงื่อนไข

```ruby
numbers = [2, 4, 6, 8, 10]

# all? - ทุกตัวต้องผ่านเงื่อนไข
puts numbers.all? { |n| n.even? }   # => true
puts numbers.all? { |n| n > 5 }     # => false (2, 4 ไม่ผ่าน)

# any? - อย่างน้อย 1 ตัวผ่าน
puts numbers.any? { |n| n > 8 }     # => true (10)
puts numbers.any? { |n| n > 15 }    # => false

# none? - ไม่มีตัวใดผ่าน
puts numbers.none? { |n| n.odd? }   # => true
puts numbers.none? { |n| n > 8 }    # => false (10)

# one? - มีแค่ 1 ตัวผ่าน
mixed = [2, 3, 4, 6, 8]
puts mixed.one? { |n| n.odd? }      # => true (3 เท่านั้น)
puts mixed.one? { |n| n > 5 }       # => false (6 และ 8)
```

```ruby
# ใช้งานโดยไม่มี block (ตรวจสอบ truthiness)
puts [1, 2, 3].all?       # => true (ทุกตัว truthy)
puts [1, nil, 3].all?     # => false (nil falsy)
puts [1, nil, 3].any?     # => true (1 truthy)
puts [nil, false].any?    # => false (ทุกตัว falsy)
puts [nil, false].none?   # => true (ไม่มี truthy)
puts [nil, 1, false].one? # => true (มีแค่ 1 truthy)
```

```ruby
# ใช้งานจริง: Validation
def valid_order?(order)
  order.all? do |item|
    item[:product] && !item[:product].empty? &&
    item[:quantity] > 0 &&
    item[:price] > 0
  end
end

valid_items = [
  { product: "Apple", quantity: 5, price: 10 },
  { product: "Banana", quantity: 3, price: 8 }
]

invalid_items = [
  { product: "", quantity: 5, price: 10 },
  { product: "Banana", quantity: -1, price: 8 }
]

puts valid_order?(valid_items)    # => true
puts valid_order?(invalid_items)  # => false
```

---

## Step 139: take_while / drop_while

### วนพร้อมเงื่อนไขหยุด

```ruby
numbers = [2, 4, 6, 7, 8, 10]

# take_while - เอาตัวที่ตรงเงื่อนไขจนกว่าจะไม่ตรง
puts numbers.take_while { |n| n.even? }.inspect
# => [2, 4, 6] (หยุดที่ 7 ซึ่งเป็นคี่)

# drop_while - ทิ้งตัวที่ตรงเงื่อนไขจนกว่าจะไม่ตรง
puts numbers.drop_while { |n| n.even? }.inspect
# => [7, 8, 10] (ทิ้ง 2, 4, 6 แล้วเอาที่เหลือทั้งหมด)
```

```ruby
# ใช้กับ sorted data
prices = [5, 10, 15, 20, 25, 30]
affordable = prices.take_while { |p| p <= 20 }
expensive = prices.drop_while { |p| p <= 20 }
puts "ราคาไม่เกิน 20: #{affordable.inspect}"    # => [5, 10, 15, 20]
puts "ราคามากกว่า 20: #{expensive.inspect}"      # => [25, 30]
```

```ruby
# ใช้งานจริง: Skip header/footer ใน data
lines = [
  "# Header",
  "# Comment",
  "data1",
  "data2",
  "data3",
  "# Footer"
]

data_lines = lines
  .drop_while { |l| l.start_with?("#") }
  .take_while { |l| !l.start_with?("#") }

puts data_lines.inspect  # => ["data1", "data2", "data3"]
```

---

## Step 140: each_slice / each_cons

### วนซ้ำแบบกลุ่ม

```ruby
numbers = (1..10).to_a

# each_slice - แบ่งเป็น slices ขนาดที่กำหนด
puts "each_slice(3):"
numbers.each_slice(3) { |slice| puts slice.inspect }
# => [1, 2, 3]
# => [4, 5, 6]
# => [7, 8, 9]
# => [10]

# each_cons - sliding window
puts "\neach_cons(3):"
numbers.each_cons(3) { |window| puts window.inspect }
# => [1, 2, 3]
# => [2, 3, 4]
# ...
# => [8, 9, 10]
```

```ruby
# ใช้งานจริง: Batch processing
users = (1..25).map { |i| "User#{i}" }

puts "Batch processing (5 per batch):"
users.each_slice(5).with_index(1) do |batch, batch_num|
  puts "Batch #{batch_num}: #{batch.first} to #{batch.last}"
  # จริงๆ จะ save หรือ process ที่นี่
end

# Pairwise comparison
scores = [10, 8, 12, 9, 15, 7, 11]
changes = scores.each_cons(2).map { |a, b| b - a }
puts "Score changes: #{changes.inspect}"
# => [-2, 4, -3, 6, -8, 4]
```

```ruby
# Moving average ด้วย each_cons
prices = [100, 102, 98, 105, 103, 101]
window = 3

ma = prices.each_cons(window).map { |w| w.sum.to_f / window }
puts "Prices: #{prices.inspect}"
puts "3-day MA: #{ma.map { |v| v.round(2) }.inspect}"
# => [100.0, 101.67, 102.0, 103.0]
```

---

## Step 141: Loop Control: break, next, redo

### ควบคุม Flow ของ Loop

```ruby
# break - ออกจาก loop
(1..10).each do |i|
  break if i > 5
  puts i
end
# => 1 2 3 4 5

# break ส่งคืนค่า
result = (1..100).each do |i|
  break i * 2 if i > 5
end
puts result  # => 12 (i=6, 6*2=12)
```

```ruby
# next - ข้ามไปรอบถัดไป
(1..10).each do |i|
  next if i.even?  # ข้ามเลขคู่
  puts i
end
# => 1 3 5 7 9

# ใช้ next เพื่อ skip
data = [1, nil, 2, nil, 3, "invalid", 4]
data.each do |item|
  next unless item.is_a?(Integer)
  puts "Processing: #{item}"
end
# => Processing: 1, 2, 3, 4
```

```ruby
# redo - ทำ iteration ปัจจุบันซ้ำ
# ใช้ระวัง! อาจทำให้ loop ไม่สิ้นสุด

count = 0
[1, 2, 3].each do |i|
  count += 1
  redo if count < 2 && i == 1  # ทำซ้ำสำหรับ i=1 ครั้งเดียว
  puts "i=#{i}, count=#{count}"
end
# => i=1, count=2
# => i=2, count=3
# => i=3, count=4

# redo ใช้ในกรณีที่ต้องการ retry operation เดิม
```

```ruby
# break ใน nested loops
found = nil
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

catch(:found) do
  matrix.each_with_index do |row, i|
    row.each_with_index do |val, j|
      if val == 5
        found = [i, j]
        throw :found  # break จาก nested loop ทั้งหมด
      end
    end
  end
end

puts "Found 5 at: #{found.inspect}"  # => [1, 1]
```

---

## Step 142: Chaining Iterators

### การต่อ Iterators

```ruby
numbers = (1..20).to_a

# Chain หลาย operations
result = numbers
  .select { |n| n % 3 == 0 }      # หารด้วย 3 ลงตัว: [3, 6, 9, 12, 15, 18]
  .reject { |n| n % 2 == 0 }      # ไม่ใช่ (หารด้วย 2 ลงตัว): [3, 9, 15]
  .map { |n| n * 10 }             # คูณด้วย 10: [30, 90, 150]
  .sum                             # รวม: 270

puts result  # => 270
```

```ruby
# Chain กับ sort และ group
words = %w[banana apple cherry date elderberry fig grape]

result = words
  .reject { |w| w.length < 4 }   # ตัดคำสั้น
  .sort_by(&:length)              # เรียงตามความยาว
  .each_slice(2)                  # จัดกลุ่มๆ ละ 2
  .map { |group| group.join(", ") } # รวมแต่ละกลุ่ม
  .join(" | ")                    # รวมทั้งหมด

puts result
# => "date, apple | grape, banana | cherry, elderberry"
```

```ruby
# Method chaining กับ lazy
# สำหรับ large dataset
result = (1..Float::INFINITY)
  .lazy
  .select { |n| n.odd? }
  .map { |n| n ** 2 }
  .select { |n| n.to_s.chars.sum(&:to_i) > 10 }  # digit sum > 10
  .first(5)

puts result.inspect
# => [49, 169, 289, 361, 529] (7², 13², 17², ...)
```

---

## Step 143: Lazy Enumerators

### Lazy Evaluation

```ruby
# ปัญหา: สร้าง array ขนาดใหญ่
# (1..1_000_000).select { |n| n.prime? }.first(10)
# ^ สร้าง array 1 ล้านตัวก่อน แล้วค่อย select

# แก้: ใช้ lazy
require 'benchmark'

def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n)).none? { |i| n % i == 0 }
end

lazy_time = Benchmark.realtime do
  (2..Float::INFINITY).lazy.select { |n| prime?(n) }.first(10)
end

puts "Lazy: #{lazy_time.round(4)}s"
puts (2..Float::INFINITY).lazy.select { |n| prime?(n) }.first(10).inspect
```

```ruby
# Lazy + complex pipeline
result = (1..Float::INFINITY)
  .lazy
  .map { |n| n ** 2 }        # square
  .select { |n| n.odd? }     # only odd
  .reject { |n| n % 3 == 0 } # not divisible by 3
  .first(5)

puts result.inspect
# => [1, 25, 49, 121, 169]

# Lazy กับ Enumerator
def fibonacci
  Enumerator.new do |yielder|
    a, b = 0, 1
    loop do
      yielder.yield a
      a, b = b, a + b
    end
  end
end

# หา fibonacci number ที่มี 10 หลัก
puts fibonacci.lazy.find { |n| n.to_s.length >= 10 }
# => 1134903170
```

```ruby
# Chained Lazy operations
words = ["hello", "world", "ruby", "is", "awesome", "language"]

result = words
  .lazy
  .select { |w| w.length > 3 }
  .map(&:upcase)
  .reject { |w| w.include?("E") }
  .first(3)

puts result.inspect
# => ["WORLD", "RUBY", "LINGO..."] (depends on input)
```

---

## Step 144: Enumerator

### สร้าง Custom Iterator

```ruby
# Enumerator.new - สร้าง iterator เอง
counter = Enumerator.new do |yielder|
  i = 0
  loop do
    yielder << i  # หรือ yielder.yield(i)
    i += 1
  end
end

puts counter.next   # => 0
puts counter.next   # => 1
puts counter.next   # => 2
puts counter.first(5).inspect  # => [0, 1, 2, 3, 4]
```

```ruby
# External vs Internal Iterator
arr = [1, 2, 3]

# Internal: block-based (ปกติ)
arr.each { |x| puts x }

# External: เรียกทีละ step
enum = arr.each
puts enum.next  # => 1
puts enum.next  # => 2
puts enum.next  # => 3
# enum.next     # => StopIteration!

# ใช้กับ loop
enum = arr.each
loop do
  puts enum.next
rescue StopIteration
  break
end
```

```ruby
# ใช้ Enumerator สร้าง infinite sequences
powers_of_two = Enumerator.new do |y|
  n = 1
  loop { y << n; n *= 2 }
end

puts powers_of_two.first(10).inspect
# => [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]

# Enumerator::Chain (Ruby 2.6+)
combined = (1..3).chain(10..12)
puts combined.to_a.inspect  # => [1, 2, 3, 10, 11, 12]

# หรือ:
puts ((1..3) + (10..12)).to_a.inspect rescue nil
```

---

## Step 145: Iterator Patterns ขั้นสูง

### รูปแบบ Iterator ขั้นสูง

```ruby
# Recursive iteration กับ yield
def traverse(tree, &block)
  return unless tree
  
  block.call(tree[:value])
  (tree[:children] || []).each { |child| traverse(child, &block) }
end

tree = {
  value: 1,
  children: [
    { value: 2, children: [
      { value: 4, children: [] },
      { value: 5, children: [] }
    ]},
    { value: 3, children: [
      { value: 6, children: [] }
    ]}
  ]
}

print "Tree traversal: "
traverse(tree) { |v| print "#{v} " }
puts  # => Tree traversal: 1 2 4 5 3 6
```

```ruby
# Enumerable ใน Custom Class
class NumberRange
  include Enumerable
  
  def initialize(start, stop, step = 1)
    @start = start
    @stop = stop
    @step = step
  end
  
  def each
    current = @start
    while current <= @stop
      yield current
      current += @step
    end
  end
end

range = NumberRange.new(1, 20, 3)
puts range.to_a.inspect            # => [1, 4, 7, 10, 13, 16, 19]
puts range.select(&:odd?).inspect  # => [1, 7, 13, 19]
puts range.map { |n| n ** 2 }.inspect # => [1, 16, 49, 100, 169, 256, 361]
puts range.sum                     # => 70
puts range.min                     # => 1
puts range.max                     # => 19
```

```ruby
# Iterator with state
def stateful_counter(start = 0, &transform)
  transform ||= ->(x) { x }
  Enumerator.new do |y|
    n = start
    loop { y << transform.call(n); n += 1 }
  end
end

# ตัวนับธรรมดา
puts stateful_counter(1).first(5).inspect   # => [1, 2, 3, 4, 5]

# ตัวนับแบบ transform
puts stateful_counter(0) { |n| n ** 2 }.first(5).inspect  # => [0, 1, 4, 9, 16]
puts stateful_counter(1) { |n| 2 ** n }.first(8).inspect  # => [2, 4, 8, 16, 32, 64, 128, 256]
```

---

## แบบฝึกหัดตอนที่ 8: Loops และ Iterators (25 ข้อ)

### ข้อ 1-5: while / until / loop

**ข้อ 1**: เขียน Collatz sequence ด้วย while loop

```ruby
# เฉลย
def collatz_sequence(n)
  sequence = [n]
  
  while n != 1
    n = n.even? ? n / 2 : 3 * n + 1
    sequence << n
  end
  
  sequence
end

[6, 11, 27].each do |n|
  seq = collatz_sequence(n)
  puts "#{n}: #{seq.length} steps - #{seq.inspect}"
end
```

**ข้อ 2**: เขียน binary search ด้วย while loop

```ruby
# เฉลย
def binary_search(arr, target)
  left, right = 0, arr.length - 1
  
  while left <= right
    mid = (left + right) / 2
    
    case arr[mid] <=> target
    when 0  then return mid
    when -1 then left = mid + 1
    when 1  then right = mid - 1
    end
  end
  
  -1  # ไม่พบ
end

sorted = [2, 5, 8, 12, 16, 23, 38, 56, 72, 91]
puts binary_search(sorted, 23)   # => 5
puts binary_search(sorted, 100)  # => -1
```

**ข้อ 3**: เขียน exponential backoff ด้วย loop

```ruby
# เฉลย
def with_retry(max_attempts: 5, base_delay: 0.1)
  attempts = 0
  
  loop do
    attempts += 1
    
    begin
      result = yield attempts
      return result  # สำเร็จ
    rescue => e
      if attempts >= max_attempts
        raise "Max attempts reached: #{e.message}"
      end
      
      delay = base_delay * (2 ** (attempts - 1))
      puts "ครั้งที่ #{attempts} ล้มเหลว: #{e.message}. รอ #{delay}s..."
      sleep(delay)
    end
  end
end

# ทดสอบ
call_count = 0
begin
  result = with_retry(max_attempts: 4, base_delay: 0.01) do |attempt|
    call_count += 1
    raise "Network error" if attempt < 3
    "สำเร็จ!"
  end
  puts "ผลลัพธ์: #{result}"
rescue => e
  puts "Error: #{e.message}"
end
```

**ข้อ 4**: เขียน password generator ด้วย loop

```ruby
# เฉลย
def generate_secure_password(length: 12, min_uppercase: 2, min_digits: 2, min_special: 1)
  uppercase = ('A'..'Z').to_a
  lowercase = ('a'..'z').to_a
  digits = ('0'..'9').to_a
  special = '!@#$%^&*'.chars
  
  password = nil
  
  loop do
    # สร้าง password สุ่ม
    chars = []
    chars += uppercase.sample(min_uppercase)
    chars += digits.sample(min_digits)
    chars += special.sample(min_special)
    
    remaining = length - chars.length
    chars += (uppercase + lowercase + digits + special).sample(remaining)
    
    password = chars.shuffle.join
    
    # ตรวจสอบว่าตรงเงื่อนไข
    break if password.scan(/[A-Z]/).length >= min_uppercase &&
             password.scan(/\d/).length >= min_digits &&
             password.scan(/[!@#$%^&*]/).length >= min_special
  end
  
  password
end

5.times { puts generate_secure_password }
```

**ข้อ 5**: เขียน number guessing game พร้อม hint

```ruby
# เฉลย
def number_guessing_game_v2
  secret = rand(1..100)
  attempts = 0
  history = []
  
  puts "=== Number Guessing Game ==="
  puts "เดาตัวเลข 1-100"
  
  loop do
    print "เดา (หรือ 'quit'): "
    input = gets.chomp
    
    break if input.downcase == "quit"
    
    guess = input.to_i
    unless (1..100).include?(guess)
      puts "กรุณาใส่ตัวเลข 1-100"
      next
    end
    
    attempts += 1
    history << guess
    
    if guess == secret
      puts "ถูกต้อง! #{secret} ใช้ #{attempts} ครั้ง"
      break
    elsif guess < secret
      diff = secret - guess
      hint = diff > 20 ? "น้อยมาก" : diff > 10 ? "น้อยไปหน่อย" : "ใกล้แล้ว"
      puts "#{hint} (#{guess} < คำตอบ)"
    else
      diff = guess - secret
      hint = diff > 20 ? "มากมาก" : diff > 10 ? "มากไปหน่อย" : "ใกล้แล้ว"
      puts "#{hint} (#{guess} > คำตอบ)"
    end
    
    if attempts % 5 == 0
      sorted = history.sort
      puts "Hint: คำตอบอยู่ระหว่าง #{sorted.select { |h| h < secret }.last || 1} และ #{sorted.select { |h| h > secret }.first || 100}"
    end
  end
end

# number_guessing_game_v2
puts "Game ready (requires interactive input)"
```

### ข้อ 6-10: map / select / reduce

**ข้อ 6**: เขียนฟังก์ชัน transform_data

```ruby
# เฉลย
def transform_data(data, transformations)
  data.map do |record|
    transformations.reduce(record) do |current, transform|
      transform.call(current)
    end
  end
end

users = [
  { name: "  alice  ", age: 25, email: "ALICE@EXAMPLE.COM" },
  { name: "  BOB   ", age: 30, email: "Bob@Test.org" }
]

transformations = [
  ->(r) { r.merge(name: r[:name].strip.capitalize) },
  ->(r) { r.merge(email: r[:email].downcase) },
  ->(r) { r.merge(age_group: r[:age] >= 30 ? "senior" : "junior") }
]

puts transform_data(users, transformations).map { |u| u.inspect }.join("\n")
```

**ข้อ 7**: เขียน FizzBuzz เวอร์ชัน functional

```ruby
# เฉลย
def fizzbuzz(n)
  (1..n).map do |i|
    case
    when i % 15 == 0 then "FizzBuzz"
    when i % 3 == 0  then "Fizz"
    when i % 5 == 0  then "Buzz"
    else i.to_s
    end
  end
end

puts fizzbuzz(30).inspect

# แบบ configurable
def custom_fizzbuzz(n, rules)
  (1..n).map do |i|
    result = rules.each_with_object("") do |(divisor, word), str|
      str << word if i % divisor == 0
    end
    result.empty? ? i.to_s : result
  end
end

rules = { 3 => "Fizz", 5 => "Buzz", 7 => "Bazz" }
puts custom_fizzbuzz(15, rules).inspect
```

**ข้อ 8**: เขียน pipeline ประมวลผลข้อมูล

```ruby
# เฉลย
class DataPipeline
  def initialize(data)
    @data = data
    @steps = []
  end
  
  def filter(&block)
    @steps << [:filter, block]
    self
  end
  
  def transform(&block)
    @steps << [:transform, block]
    self
  end
  
  def aggregate(&block)
    @steps << [:aggregate, block]
    self
  end
  
  def execute
    result = @data
    @steps.each do |type, block|
      result = case type
               when :filter    then result.select(&block)
               when :transform then result.map(&block)
               when :aggregate then block.call(result)
               end
    end
    result
  end
end

data = (1..50).to_a

result = DataPipeline.new(data)
  .filter { |n| n % 3 == 0 }
  .transform { |n| n ** 2 }
  .filter { |n| n.to_s.length <= 3 }
  .aggregate { |arr| { count: arr.length, sum: arr.sum, avg: arr.sum.to_f / arr.length } }
  .execute

puts result.inspect
```

**ข้อ 9**: เขียน word frequency analyzer

```ruby
# เฉลย
def word_frequency_analysis(text)
  words = text.downcase
               .gsub(/[^a-z\s]/, "")
               .split
  
  freq = words.each_with_object(Hash.new(0)) { |w, h| h[w] += 1 }
  
  # Statistics
  total_words = words.length
  unique_words = freq.keys.length
  
  # Top words
  top_n = freq.sort_by { |_, v| -v }.first(10)
  
  # Hapax legomena (คำที่ปรากฎครั้งเดียว)
  hapax = freq.select { |_, v| v == 1 }.keys
  
  # Average occurrences
  avg_freq = freq.values.sum.to_f / unique_words
  
  {
    total: total_words,
    unique: unique_words,
    top_words: top_n,
    hapax_count: hapax.length,
    avg_frequency: avg_freq.round(2)
  }
end

text = "Ruby is a dynamic language. Ruby is elegant. Ruby programming is fun. Programming with Ruby is a joy."

result = word_frequency_analysis(text)
puts "รวม: #{result[:total]} คำ"
puts "Unique: #{result[:unique]} คำ"
puts "\nTop words:"
result[:top_words].each { |w, c| puts "  #{w}: #{c}" }
puts "Hapax legomena: #{result[:hapax_count]} คำ"
puts "เฉลี่ย: #{result[:avg_frequency]} ครั้ง/คำ"
```

**ข้อ 10**: เขียน sliding window ค้นหา maximum sum subarray

```ruby
# เฉลย
# Sliding Window Maximum Sum (Fixed size)
def max_sum_fixed_window(arr, k)
  return nil if arr.length < k
  
  windows = arr.each_cons(k)
  max_window = windows.max_by { |w| w.sum }
  
  {
    max_sum: max_window.sum,
    window: max_window,
    start_index: arr.each_cons(k).find_index { |w| w == max_window }
  }
end

# Kadane's Algorithm (any size)
def max_sum_subarray(arr)
  max_sum = current_sum = arr[0]
  max_start = max_end = current_start = 0
  
  (1...arr.length).each do |i|
    if current_sum + arr[i] < arr[i]
      current_sum = arr[i]
      current_start = i
    else
      current_sum += arr[i]
    end
    
    if current_sum > max_sum
      max_sum = current_sum
      max_start = current_start
      max_end = i
    end
  end
  
  {
    max_sum: max_sum,
    subarray: arr[max_start..max_end],
    range: (max_start..max_end)
  }
end

arr = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
puts "Fixed window (k=3): #{max_sum_fixed_window(arr, 3).inspect}"
puts "Max subarray: #{max_sum_subarray(arr).inspect}"
# => max subarray: {max_sum: 6, subarray: [4, -1, 2, 1], range: 3..6}
```

### ข้อ 11-15: เมธอด Enumerable

**ข้อ 11**: เขียน custom each_with_rolling_sum

```ruby
# เฉลย
def each_with_rolling_sum(arr)
  running_sum = 0
  arr.each_with_index do |val, i|
    running_sum += val
    yield val, running_sum, i
  end
end

prices = [10, 25, 15, 30, 20]
each_with_rolling_sum(prices) do |price, total, i|
  puts "Item #{i + 1}: #{price} บาท (รวม #{total} บาท)"
end
```

**ข้อ 12**: เขียน chunk_by_change

```ruby
# เฉลย
def chunk_by_change(arr, &block)
  return [] if arr.empty?
  
  result = []
  current_group = [arr[0]]
  current_key = block ? block.call(arr[0]) : arr[0]
  
  arr[1..].each do |item|
    new_key = block ? block.call(item) : item
    if new_key == current_key
      current_group << item
    else
      result << { key: current_key, values: current_group }
      current_group = [item]
      current_key = new_key
    end
  end
  
  result << { key: current_key, values: current_group }
  result
end

# ตัวอย่าง: จัดกลุ่มตาม positive/negative
nums = [1, 2, -3, -4, 5, -6, 7, 8, 9]
groups = chunk_by_change(nums) { |n| n > 0 ? :positive : :negative }
groups.each { |g| puts "#{g[:key]}: #{g[:values].inspect}" }
```

**ข้อ 13**: เขียน Enumerable method จาก scratch

```ruby
# เฉลย - implement common Enumerable methods
module MyEnumerable
  def my_map
    result = []
    each { |item| result << yield(item) }
    result
  end
  
  def my_select
    result = []
    each { |item| result << item if yield(item) }
    result
  end
  
  def my_reduce(initial = nil)
    accumulator = initial
    each do |item|
      if accumulator.nil? && initial.nil?
        accumulator = item
      else
        accumulator = yield(accumulator, item)
      end
    end
    accumulator
  end
  
  def my_all?
    each { |item| return false unless yield(item) }
    true
  end
  
  def my_find
    each { |item| return item if yield(item) }
    nil
  end
end

class NumberList
  include MyEnumerable
  
  def initialize(*numbers)
    @numbers = numbers
  end
  
  def each(&block)
    @numbers.each(&block)
  end
end

list = NumberList.new(1, 2, 3, 4, 5, 6)
puts list.my_map { |n| n * 2 }.inspect     # => [2, 4, 6, 8, 10, 12]
puts list.my_select { |n| n.even? }.inspect # => [2, 4, 6]
puts list.my_reduce(0) { |sum, n| sum + n } # => 21
puts list.my_all? { |n| n > 0 }            # => true
puts list.my_find { |n| n > 4 }            # => 5
```

**ข้อ 14**: เขียน generator สำหรับ permutations

```ruby
# เฉลย
def permutations(arr, k = nil)
  k ||= arr.length
  return [[]] if k == 0
  
  result = []
  arr.each_with_index do |item, i|
    rest = arr[0...i] + arr[i+1..]
    permutations(rest, k - 1).each do |perm|
      result << [item] + perm
    end
  end
  result
end

def permutations_lazy(arr, k = nil)
  Enumerator.new do |yielder|
    generate_permutations(arr, k || arr.length, [], yielder)
  end
end

def generate_permutations(remaining, k, current, yielder)
  if current.length == k
    yielder << current
    return
  end
  
  remaining.each_with_index do |item, i|
    rest = remaining[0...i] + remaining[i+1..]
    generate_permutations(rest, k, current + [item], yielder)
  end
end

arr = [1, 2, 3]
puts "All permutations:"
permutations(arr).each { |p| puts p.inspect }

puts "\nFirst 3 permutations (lazy):"
permutations_lazy(arr).first(3).each { |p| puts p.inspect }
```

**ข้อ 15**: เขียน Scheduler ด้วย Enumerator

```ruby
# เฉลย
class RoundRobinScheduler
  include Enumerable
  
  def initialize(tasks)
    @tasks = tasks
    @current = 0
  end
  
  def next_task
    task = @tasks[@current % @tasks.length]
    @current += 1
    task
  end
  
  def each
    return to_enum unless block_given?
    loop { yield next_task }
  end
  
  def schedule(n)
    take(n)
  end
end

tasks = ["Task A", "Task B", "Task C", "Task D"]
scheduler = RoundRobinScheduler.new(tasks)

puts "Schedule 10 tasks:"
scheduler.schedule(10).each_with_index do |task, i|
  puts "  Slot #{i + 1}: #{task}"
end
```

### ข้อ 16-20: Advanced Patterns

**ข้อ 16**: เขียน lazy infinite sequence generator

```ruby
# เฉลย
def arithmetic_sequence(start, difference)
  Enumerator.new do |y|
    n = start
    loop { y << n; n += difference }
  end
end

def geometric_sequence(start, ratio)
  Enumerator.new do |y|
    n = start
    loop { y << n; n *= ratio }
  end
end

def sieve_of_primes
  Enumerator.new do |y|
    primes = []
    n = 2
    loop do
      if primes.none? { |p| n % p == 0 }
        primes << n
        y << n
      end
      n += 1
    end
  end
end

# Test
puts "Arithmetic (2, 3): #{arithmetic_sequence(2, 3).first(8).inspect}"
# => [2, 5, 8, 11, 14, 17, 20, 23]

puts "Geometric (2, 3): #{geometric_sequence(2, 3).first(7).inspect}"
# => [2, 6, 18, 54, 162, 486, 1458]

puts "Primes: #{sieve_of_primes.first(10).inspect}"
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

**ข้อ 17**: เขียน concurrent-style map ด้วย Thread

```ruby
# เฉลย
def parallel_map(arr, max_threads: 4)
  result = Array.new(arr.length)
  mutex = Mutex.new
  threads = []
  
  arr.each_slice((arr.length.to_f / max_threads).ceil).each_with_index do |slice, batch_idx|
    start_idx = batch_idx * (arr.length.to_f / max_threads).ceil
    
    threads << Thread.new(slice, start_idx) do |items, offset|
      items.each_with_index do |item, i|
        computed = yield item
        mutex.synchronize { result[offset + i] = computed }
      end
    end
  end
  
  threads.each(&:join)
  result
end

# ทดสอบ (simulate expensive computation)
data = (1..20).to_a
result = parallel_map(data, max_threads: 4) { |n| n ** 2 }
puts result.inspect
# => [1, 4, 9, 16, 25, 36, 49, 64, 81, 100, 121, 144, 169, 196, 225, 256, 289, 324, 361, 400]
```

**ข้อ 18**: เขียน memoized Fibonacci ด้วย lazy enumerator

```ruby
# เฉลย
def memoized_fibonacci
  cache = {}
  
  fib = ->(n) do
    cache[n] ||= case n
                 when 0 then 0
                 when 1 then 1
                 else fib.call(n-1) + fib.call(n-2)
                 end
  end
  
  Enumerator.new do |y|
    n = 0
    loop { y << fib.call(n); n += 1 }
  end
end

fib_gen = memoized_fibonacci

puts "First 20 Fibonacci:"
puts fib_gen.first(20).inspect

puts "\nFibonacci up to 1000:"
puts fib_gen.take_while { |n| n < 1000 }.inspect

puts "\n100th Fibonacci:"
puts fib_gen.first(100).last
```

**ข้อ 19**: เขียน event stream processor

```ruby
# เฉลย
class EventStream
  def initialize
    @events = []
    @handlers = {}
    @transformers = []
    @filters = []
  end
  
  def emit(event_type, data)
    @events << { type: event_type, data: data, timestamp: Time.now }
    self
  end
  
  def on(event_type, &handler)
    @handlers[event_type] ||= []
    @handlers[event_type] << handler
    self
  end
  
  def filter(&block)
    @filters << block
    self
  end
  
  def transform(&block)
    @transformers << block
    self
  end
  
  def process
    @events
      .select { |e| @filters.all? { |f| f.call(e) } }
      .map { |e| @transformers.reduce(e) { |current, t| t.call(current) } }
      .each do |e|
        (@handlers[e[:type]] || []).each { |h| h.call(e[:data]) }
        (@handlers[:all] || []).each { |h| h.call(e) }
      end
  end
end

stream = EventStream.new

stream
  .on(:purchase) { |data| puts "Purchase: #{data[:product]} - #{data[:amount]} บาท" }
  .on(:refund) { |data| puts "Refund: #{data[:product]} - #{data[:amount]} บาท" }
  .filter { |e| e[:data][:amount] > 100 }

stream.emit(:purchase, { product: "Laptop", amount: 35000 })
stream.emit(:purchase, { product: "Pen", amount: 50 })  # กรอง (< 100)
stream.emit(:refund, { product: "Keyboard", amount: 1500 })
stream.emit(:purchase, { product: "Mouse", amount: 500 })

stream.process
```

**ข้อ 20**: เขียน coroutine-style producer-consumer

```ruby
# เฉลย
def producer(n)
  Enumerator.new do |y|
    n.times do |i|
      item = { id: i, value: rand(100), processed: false }
      puts "Producing item #{i}: #{item[:value]}"
      y << item
    end
  end
end

def consumer(source, processor)
  source.map do |item|
    processed_value = processor.call(item[:value])
    item.merge(processed: true, result: processed_value)
  end
end

# Pipeline
items = producer(5)
doubled = consumer(items, ->(v) { v * 2 })
filtered = doubled.select { |item| item[:result] > 50 }

puts "\nProcessed items:"
filtered.each do |item|
  puts "  Item #{item[:id]}: #{item[:value]} -> #{item[:result]}"
end
```

### ข้อ 21-25: Complex Patterns

**ข้อ 21**: เขียน trie data structure ด้วย iterators

```ruby
# เฉลย
class Trie
  def initialize
    @root = {}
  end
  
  def insert(word)
    node = @root
    word.each_char do |char|
      node[char] ||= {}
      node = node[char]
    end
    node[:end] = true
  end
  
  def search(word)
    node = @root
    word.each_char do |char|
      return false unless node.key?(char)
      node = node[char]
    end
    node.key?(:end)
  end
  
  def starts_with(prefix)
    node = @root
    prefix.each_char do |char|
      return [] unless node.key?(char)
      node = node[char]
    end
    collect_words(node, prefix)
  end
  
  private
  
  def collect_words(node, prefix)
    results = []
    results << prefix if node[:end]
    
    node.each do |char, child_node|
      next if char == :end
      results.concat(collect_words(child_node, prefix + char))
    end
    
    results
  end
end

trie = Trie.new
%w[apple app application apply appreciate].each { |w| trie.insert(w) }

puts "Search 'apple': #{trie.search('apple')}"
puts "Search 'app': #{trie.search('app')}"
puts "Search 'apt': #{trie.search('apt')}"
puts "Starts with 'app': #{trie.starts_with('app').inspect}"
puts "Starts with 'appl': #{trie.starts_with('appl').inspect}"
```

**ข้อ 22**: เขียน observer pattern ด้วย iterators

```ruby
# เฉลย
module Observable
  def self.included(base)
    base.instance_variable_set(:@observers, Hash.new { |h, k| h[k] = [] })
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def on(event, &handler)
      @observers[event] << handler
    end
    
    def observers
      @observers
    end
  end
  
  def emit(event, *args)
    self.class.observers[event].each { |handler| handler.call(*args) }
    self.class.observers[:any].each { |handler| handler.call(event, *args) }
  end
end

class OrderProcessor
  include Observable
  
  def process(order)
    emit(:start, order)
    
    order[:items].each do |item|
      emit(:item_processed, item)
    end
    
    total = order[:items].sum { |i| i[:price] }
    emit(:complete, order.merge(total: total))
  end
end

OrderProcessor.on(:start) { |o| puts "เริ่มประมวลผล Order ##{o[:id]}" }
OrderProcessor.on(:item_processed) { |item| puts "  - #{item[:name]}: #{item[:price]}" }
OrderProcessor.on(:complete) { |o| puts "เสร็จสิ้น ยอดรวม: #{o[:total]} บาท" }

processor = OrderProcessor.new
order = {
  id: 101,
  items: [
    { name: "Laptop", price: 35000 },
    { name: "Mouse", price: 500 },
    { name: "Keyboard", price: 1500 }
  ]
}

processor.process(order)
```

**ข้อ 23**: เขียน recursive tree iterator

```ruby
# เฉลย
class TreeNode
  attr_accessor :value, :children
  
  def initialize(value)
    @value = value
    @children = []
  end
  
  def add_child(child)
    @children << child
    self
  end
  
  def each_depth_first(&block)
    block.call(self)
    children.each { |child| child.each_depth_first(&block) }
  end
  
  def each_breadth_first(&block)
    queue = [self]
    until queue.empty?
      node = queue.shift
      block.call(node)
      queue.concat(node.children)
    end
  end
  
  def depth_first_enumerator
    Enumerator.new { |y| each_depth_first { |node| y << node } }
  end
  
  def breadth_first_enumerator
    Enumerator.new { |y| each_breadth_first { |node| y << node } }
  end
  
  def map(&block)
    depth_first_enumerator.map { |node| block.call(node.value) }
  end
end

# สร้าง tree
root = TreeNode.new(1)
node2 = TreeNode.new(2)
node3 = TreeNode.new(3)
node4 = TreeNode.new(4)
node5 = TreeNode.new(5)
node6 = TreeNode.new(6)

root.add_child(node2).add_child(node3)
node2.add_child(node4).add_child(node5)
node3.add_child(node6)

print "DFS: "
root.each_depth_first { |n| print "#{n.value} " }
puts

print "BFS: "
root.each_breadth_first { |n| print "#{n.value} " }
puts

puts "Sum: #{root.map(&:itself).sum}"
```

**ข้อ 24**: เขียน stream processor กับ backpressure

```ruby
# เฉลย
class StreamProcessor
  def initialize(buffer_size: 10)
    @buffer_size = buffer_size
    @buffer = []
    @processors = []
    @sink = nil
  end
  
  def source(enum)
    @source = enum
    self
  end
  
  def pipe(&processor)
    @processors << processor
    self
  end
  
  def sink(&handler)
    @sink = handler
    self
  end
  
  def run
    @source.each do |item|
      # Apply processors
      processed = @processors.reduce(item) do |current, proc|
        proc.call(current)
      end
      
      # Buffer management (backpressure)
      @buffer << processed
      
      if @buffer.length >= @buffer_size
        flush
      end
    end
    
    flush  # Flush remaining
  end
  
  private
  
  def flush
    @buffer.each { |item| @sink&.call(item) } unless @buffer.empty?
    @buffer.clear
    puts "  [Flushed #{@buffer_size} items]" if @buffer_size <= @buffer.length
  end
end

source_data = (1..25).each

StreamProcessor.new(buffer_size: 5)
  .source(source_data)
  .pipe { |n| n * 2 }
  .pipe { |n| { value: n, squared: n ** 2 } }
  .sink { |item| puts "Output: #{item[:value]} (#{item[:squared]})" }
  .run
```

**ข้อ 25**: สร้าง DSL สำหรับ data transformation

```ruby
# เฉลย
class TransformDSL
  def initialize(data)
    @data = data
  end
  
  def self.transform(data, &block)
    instance = new(data)
    instance.instance_eval(&block)
    instance.result
  end
  
  def filter(field = nil, &block)
    if field
      @data = @data.select { |r| block.call(r[field]) }
    else
      @data = @data.select(&block)
    end
    self
  end
  
  def map_field(field, &block)
    @data = @data.map { |r| r.merge(field => block.call(r[field])) }
    self
  end
  
  def add_field(field, &block)
    @data = @data.map { |r| r.merge(field => block.call(r)) }
    self
  end
  
  def sort_by_field(field, direction: :asc)
    @data = if direction == :asc
              @data.sort_by { |r| r[field] }
            else
              @data.sort_by { |r| r[field] }.reverse
            end
    self
  end
  
  def limit(n)
    @data = @data.first(n)
    self
  end
  
  def result
    @data
  end
end

products = [
  { id: 1, name: "Laptop", price: 35000, category: "electronics" },
  { id: 2, name: "Shirt", price: 500, category: "clothing" },
  { id: 3, name: "Phone", price: 15000, category: "electronics" },
  { id: 4, name: "Pants", price: 800, category: "clothing" },
  { id: 5, name: "Tablet", price: 20000, category: "electronics" }
]

result = TransformDSL.transform(products) do
  filter(:category) { |c| c == "electronics" }
  filter { |r| r[:price] < 30000 }
  add_field(:price_with_vat) { |r| (r[:price] * 1.07).round }
  map_field(:name, &:upcase)
  sort_by_field(:price)
end

result.each { |p| puts p.inspect }
```

---

## สรุป

| Loop/Iterator | การใช้งาน | ตัวอย่าง |
|--------------|----------|---------|
| `while` | วนตราบที่เงื่อนไขเป็น true | `while x < 10; ...; end` |
| `until` | วนตราบที่เงื่อนไขเป็น false | `until queue.empty?` |
| `loop` | วนไม่สิ้นสุด + break | `loop { break if done }` |
| `for..in` | ไม่นิยม (ใช้ each แทน) | `for x in arr` |
| `times` | วน n ครั้ง | `5.times { }` |
| `upto/downto` | วนใน range | `1.upto(10)` |
| `step` | วนด้วย step | `1.step(10, 2)` |
| `each` | วนดูทุก element | `arr.each { |x| }` |
| `each_with_index` | วนพร้อม index | `arr.each_with_index { |x, i| }` |
| `each_with_object` | วนพร้อมสะสมค่า | `arr.each_with_object({})` |
| `map/collect` | แปลงทุก element | `arr.map { |x| x * 2 }` |
| `select/filter` | กรอง element | `arr.select { |x| x > 5 }` |
| `reject` | ตัด element ออก | `arr.reject { |x| x.nil? }` |
| `reduce/inject` | สะสมค่าเป็น single value | `arr.reduce(:+)` |
| `flat_map` | map + flatten | `arr.flat_map { |x| [x, x] }` |
| `find/detect` | หาตัวแรกที่ตรงเงื่อนไข | `arr.find { |x| x > 5 }` |
| `all?/any?/none?` | ตรวจสอบเงื่อนไข | `arr.all? { |x| x > 0 }` |
| `count` | นับตามเงื่อนไข | `arr.count { |x| x.even? }` |
| `sum` | รวมค่า | `arr.sum { |x| x ** 2 }` |
| `take_while` | เอาจนกว่าเงื่อนไขไม่ตรง | `arr.take_while { |x| x < 5 }` |
| `drop_while` | ทิ้งจนกว่าเงื่อนไขไม่ตรง | `arr.drop_while { |x| x < 5 }` |
| `each_slice` | วนทีละกลุ่ม | `arr.each_slice(3) { }` |
| `each_cons` | วนแบบ sliding window | `arr.each_cons(3) { }` |
| `lazy` | Lazy evaluation | `(1..inf).lazy.select { }.first(n)` |
| `break` | ออกจาก loop + optional value | `break result` |
| `next` | ข้ามรอบถัดไป | `next if condition` |

ตอนต่อไปจะเป็นเรื่อง **Methods และ Blocks** ที่จะเรียนรู้การสร้างและใช้งาน Method พร้อม Block, Proc, และ Lambda อย่างละเอียด
