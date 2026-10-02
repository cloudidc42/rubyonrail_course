# ตอนที่ 5: Arrays

## Ruby Programming Course สำหรับผู้เริ่มต้นภาษาไทย

---

## บทนำ

Array คือโครงสร้างข้อมูลที่เก็บลำดับของ objects ใน Ruby Array มีความยืดหยุ่นสูงมาก สามารถเก็บข้อมูลต่างประเภทกันได้ และมี Methods ที่ทรงพลังมากมาย

**สิ่งที่จะได้เรียนรู้:**
- การสร้าง Array แบบต่างๆ
- การเข้าถึง Elements
- Methods สำหรับแก้ไข Array
- Methods สำหรับตรวจสอบ
- Methods สำหรับ Transform (map, select, reject, reduce)
- Methods สำหรับ Sort
- Methods สำหรับจัดกลุ่ม
- Methods สำหรับรวม Arrays
- Multi-dimensional Arrays
- Array Destructuring
- Array เป็น Stack/Queue
- Lazy Enumerator

---

## Step 61: การสร้าง Array

### 61.1 วิธีสร้าง Array แบบต่างๆ

```ruby
# 1. Array Literal - ง่ายที่สุด
numbers = [1, 2, 3, 4, 5]
mixed   = [1, "hello", :symbol, true, nil, 3.14]
empty   = []

puts numbers.inspect  # => [1, 2, 3, 4, 5]
puts mixed.inspect    # => [1, "hello", :symbol, true, nil, 3.14]
puts empty.inspect    # => []
puts empty.empty?     # => true

# 2. Array.new
arr1 = Array.new(3)          # => [nil, nil, nil]
arr2 = Array.new(3, 0)       # => [0, 0, 0]
arr3 = Array.new(5) { |i| i * 2 }  # => [0, 2, 4, 6, 8]
arr4 = Array.new(3) { [] }   # => [[], [], []] (แต่ละ element เป็น object ต่างกัน)

puts arr1.inspect  # => [nil, nil, nil]
puts arr2.inspect  # => [0, 0, 0]
puts arr3.inspect  # => [0, 2, 4, 6, 8]

# ระวัง! Array.new(3, []) ทำให้ทุก element ชี้ไปที่ Array เดียวกัน
bad  = Array.new(3, [])
bad[0] << 1
puts bad.inspect   # => [[1], [1], [1]] (ผิดที่ต้องการ!)

good = Array.new(3) { [] }
good[0] << 1
puts good.inspect  # => [[1], [], []] (ถูกต้อง)
```

```ruby
# 3. %w - Word Array (String array โดย whitespace)
colors = %w[red green blue yellow orange]
puts colors.inspect  # => ["red", "green", "blue", "yellow", "orange"]
puts colors.class    # => Array
puts colors[0].class # => String

# %w รองรับ whitespace ใน element ด้วย backslash
paths = %w[/usr/bin /usr/local/bin /home/user\ name/bin]
puts paths.inspect

# 4. %i - Symbol Array
statuses = %i[active inactive pending blocked]
puts statuses.inspect  # => [:active, :inactive, :pending, :blocked]
puts statuses[0].class # => Symbol
```

```ruby
# 5. Range to Array
nums = (1..10).to_a
puts nums.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

letters = ("a".."z").to_a
puts letters.inspect  # => ["a", "b", ..., "z"]

# 6. Splat เพื่อสร้าง Array จาก argument
def make_array(*args)
  args
end

arr = make_array(1, 2, 3, "hello")
puts arr.inspect  # => [1, 2, 3, "hello"]

# 7. Array() method - แปลง object เป็น Array
puts Array(nil).inspect        # => []
puts Array(1).inspect          # => [1]
puts Array([1, 2]).inspect     # => [1, 2]
puts Array(1..5).inspect       # => [1, 2, 3, 4, 5]
puts Array({a: 1}).inspect     # => [[:a, 1]]
```

---

## Step 62: การเข้าถึง Elements

### 62.1 Index Access

```ruby
fruits = ["แอปเปิ้ล", "กล้วย", "มะม่วง", "ส้ม", "องุ่น"]

# Positive index (เริ่มจาก 0)
puts fruits[0]   # => แอปเปิ้ล (ตัวแรก)
puts fruits[1]   # => กล้วย
puts fruits[4]   # => องุ่น (ตัวสุดท้าย)
puts fruits[10]  # => nil (ไม่มี index นี้)

# Negative index (นับจากท้าย)
puts fruits[-1]  # => องุ่น (ตัวสุดท้าย)
puts fruits[-2]  # => ส้ม
puts fruits[-5]  # => แอปเปิ้ล (ตัวแรก)

# Range index - slice
puts fruits[1, 3].inspect    # => ["กล้วย", "มะม่วง", "ส้ม"] (start, length)
puts fruits[1..3].inspect    # => ["กล้วย", "มะม่วง", "ส้ม"] (inclusive range)
puts fruits[1...3].inspect   # => ["กล้วย", "มะม่วง"] (exclusive range)
puts fruits[2..].inspect     # => ["มะม่วง", "ส้ม", "องุ่น"] (endless range)
puts fruits[..2].inspect     # => ["แอปเปิ้ล", "กล้วย", "มะม่วง"] (beginless range)
```

```ruby
# Methods สำหรับเข้าถึง
arr = [10, 20, 30, 40, 50]

puts arr.first      # => 10
puts arr.first(3).inspect  # => [10, 20, 30]
puts arr.last       # => 50
puts arr.last(2).inspect   # => [40, 50]

# at - เหมือน []
puts arr.at(2)      # => 30
puts arr.at(-1)     # => 50

# fetch - raise Error ถ้า out of bounds
puts arr.fetch(1)   # => 20
begin
  arr.fetch(10)
rescue IndexError => e
  puts "Error: #{e.message}"  # => index 10 outside of array bounds: -5...5
end

# fetch ด้วย default
puts arr.fetch(10, "ไม่มี")  # => ไม่มี
puts arr.fetch(10) { |i| "Index #{i} ไม่มี" }  # => Index 10 ไม่มี

# sample - สุ่ม
puts arr.sample       # => random element
puts arr.sample(2).inspect  # => 2 random elements
```

```ruby
# dig - เข้าถึง nested array
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
puts matrix.dig(1, 2)   # => 6 (row 1, col 2)
puts matrix.dig(0, 0)   # => 1

nested = [[[1, 2], [3, 4]], [[5, 6], [7, 8]]]
puts nested.dig(1, 0, 1)  # => 6
puts nested.dig(0, 1, 0)  # => 3
puts nested.dig(0, 5)     # => nil (ไม่ raise error)
```

---

## Step 63: Array Methods - Modifying

### 63.1 เพิ่ม Elements

```ruby
arr = [1, 2, 3]

# push / << - เพิ่มท้าย
arr.push(4)     # => [1, 2, 3, 4]
arr << 5        # => [1, 2, 3, 4, 5]
arr.push(6, 7)  # => [1, 2, 3, 4, 5, 6, 7]

# append (Ruby 2.5+) - เหมือน push
arr.append(8)   # => [1, 2, 3, 4, 5, 6, 7, 8]

# unshift - เพิ่มหน้า
arr = [3, 4, 5]
arr.unshift(2)     # => [2, 3, 4, 5]
arr.unshift(0, 1)  # => [0, 1, 2, 3, 4, 5]

# prepend (Ruby 2.5+) - เหมือน unshift
arr.prepend(-1)   # => [-1, 0, 1, 2, 3, 4, 5]

# insert - เพิ่มตรง position
arr = [1, 2, 5, 6]
arr.insert(2, 3, 4)  # => [1, 2, 3, 4, 5, 6] (insert at index 2)
arr.insert(-1, 7)    # => [1, 2, 3, 4, 5, 6, 7] (insert before last)
puts arr.inspect
```

```ruby
# ลบ Elements
arr = [1, 2, 3, 4, 5]

# pop - ลบและคืนค่าตัวสุดท้าย
last = arr.pop        # => 5
puts arr.inspect      # => [1, 2, 3, 4]
puts "popped: #{last}"

# pop n elements
last_two = arr.pop(2) # => [3, 4]
puts arr.inspect      # => [1, 2]

# shift - ลบและคืนค่าตัวแรก
first = arr.shift     # => 1
puts arr.inspect      # => [2]

# delete - ลบค่าที่ระบุ (ทุก occurrence)
arr = [1, 2, 3, 2, 4, 2]
arr.delete(2)
puts arr.inspect  # => [1, 3, 4]

# delete ด้วย Block
arr = [1, 2, 3, 4, 5]
result = arr.delete(10) { "ไม่พบ" }
puts result  # => ไม่พบ

# delete_at - ลบที่ index
arr = [1, 2, 3, 4, 5]
removed = arr.delete_at(2)
puts removed       # => 3
puts arr.inspect   # => [1, 2, 4, 5]

# delete_if - ลบตาม condition
arr = [1, 2, 3, 4, 5, 6]
arr.delete_if { |n| n.even? }
puts arr.inspect  # => [1, 3, 5]
```

```ruby
# slice! - ลบและคืนค่า
arr = [1, 2, 3, 4, 5]

removed = arr.slice!(1, 2)  # ลบ 2 elements เริ่มที่ index 1
puts removed.inspect  # => [2, 3]
puts arr.inspect      # => [1, 4, 5]

# reject! - ลบตาม condition (mutate in place)
arr = [1, 2, 3, 4, 5, 6]
arr.reject! { |n| n > 4 }
puts arr.inspect  # => [1, 2, 3, 4]

# keep_if - เก็บเฉพาะที่ match (เหมือน select! )
arr = [1, 2, 3, 4, 5]
arr.keep_if { |n| n.odd? }
puts arr.inspect  # => [1, 3, 5]

# clear - ล้างทั้ง Array
arr = [1, 2, 3]
arr.clear
puts arr.inspect  # => []
puts arr.empty?   # => true
```

---

## Step 64: Array Methods - Checking

### 64.1 Predicate Methods

```ruby
arr = [1, 2, 3, 4, 5]
empty_arr = []

# empty? - ตรวจว่าว่างหรือไม่
puts arr.empty?        # => false
puts empty_arr.empty?  # => true

# include? - ตรวจว่ามี element นั้นหรือไม่
puts arr.include?(3)   # => true
puts arr.include?(10)  # => false

# any? - ตรวจว่ามีสักตัวที่ตรงตาม condition
puts arr.any? { |n| n > 3 }    # => true (4 และ 5)
puts arr.any? { |n| n > 10 }   # => false
puts [false, nil].any?          # => false (ไม่มี truthy element)
puts [false, nil, 0].any?       # => true (0 เป็น truthy)

# all? - ตรวจว่าทุกตัวตรงตาม condition
puts arr.all? { |n| n > 0 }    # => true
puts arr.all? { |n| n.even? }  # => false (1, 3, 5 ไม่ใช่ even)
puts [1, 2, 3].all?             # => true (ทุกตัวเป็น truthy)
puts [1, nil, 3].all?           # => false (nil เป็น falsy)

# none? - ตรวจว่าไม่มีสักตัวที่ตรงตาม condition
puts arr.none? { |n| n > 10 }  # => true
puts arr.none? { |n| n.even? } # => false
puts [false, nil].none?         # => true (ไม่มี truthy)

# one? - ตรวจว่ามีแค่ตัวเดียวที่ตรง
puts arr.one? { |n| n > 4 }   # => true (เฉพาะ 5)
puts arr.one? { |n| n > 3 }   # => false (4 และ 5 ทั้งสอง)
puts arr.one?                   # => false (มีหลายตัว truthy)
```

```ruby
# count - นับจำนวน
arr = [1, 2, 3, 2, 1, 4, 2]

puts arr.count          # => 7 (นับทุกตัว)
puts arr.count(2)       # => 3 (นับตัวที่เท่ากับ 2)
puts arr.count { |n| n.even? }  # => 4 (นับ even)

# length / size
puts arr.length  # => 7
puts arr.size    # => 7 (เหมือนกัน)

# tally - นับความถี่
puts arr.tally.inspect  # => {1=>2, 2=>3, 3=>1, 4=>1}

# สรุป: predicate methods
data = [3, 1, 4, 1, 5, 9, 2, 6]
puts "มีเลขคู่: #{data.any?(&:even?)}"
puts "ทุกตัวบวก: #{data.all?(&:positive?)}"
puts "ไม่มีเลข > 10: #{data.none? { |n| n > 10 }}"
puts "มีแค่ตัวเดียว > 8: #{data.one? { |n| n > 8 }}"
```

---

## Step 65: Array Methods - Transforming

### 65.1 map / collect

```ruby
numbers = [1, 2, 3, 4, 5]

# map - แปลงทุก element ได้ Array ใหม่
squares = numbers.map { |n| n ** 2 }
puts squares.inspect  # => [1, 4, 9, 16, 25]

# map ด้วย method reference (&:method_name)
words = %w[hello world ruby]
upcase_words = words.map(&:upcase)
puts upcase_words.inspect  # => ["HELLO", "WORLD", "RUBY"]

# map! - mutate in place
words.map!(&:capitalize)
puts words.inspect  # => ["Hello", "World", "Ruby"]

# map ซ้อนกัน
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
doubled = matrix.map { |row| row.map { |n| n * 2 } }
puts doubled.inspect  # => [[2, 4, 6], [8, 10, 12], [14, 16, 18]]
```

```ruby
# select / filter - เลือก elements ที่ตรง condition
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

evens = numbers.select { |n| n.even? }
puts evens.inspect  # => [2, 4, 6, 8, 10]

# filter เหมือน select (Ruby 2.6+)
odds = numbers.filter { |n| n.odd? }
puts odds.inspect  # => [1, 3, 5, 7, 9]

# filter_map (Ruby 2.7+) - map แล้ว filter nil
results = numbers.filter_map { |n| n * 2 if n.odd? }
puts results.inspect  # => [2, 6, 10, 14, 18]

# reject - ตรงข้าม select
big_nums = numbers.reject { |n| n <= 5 }
puts big_nums.inspect  # => [6, 7, 8, 9, 10]
```

```ruby
# reduce / inject - Accumulate
numbers = [1, 2, 3, 4, 5]

# Sum
sum = numbers.reduce(0) { |acc, n| acc + n }
puts sum  # => 15

# ใช้ symbol
sum2 = numbers.reduce(:+)
puts sum2  # => 15

# Product
product = numbers.inject(1, :*)
puts product  # => 120

# Max ด้วย reduce
max = numbers.reduce { |max, n| n > max ? n : max }
puts max  # => 5

# สร้าง Hash จาก Array
words = %w[apple banana cherry]
word_lengths = words.reduce({}) do |hash, word|
  hash[word] = word.length
  hash
end
puts word_lengths.inspect  # => {"apple"=>5, "banana"=>6, "cherry"=>6}

# each_with_object (บางครั้งง่ายกว่า reduce)
word_lengths2 = words.each_with_object({}) do |word, hash|
  hash[word] = word.length
end
puts word_lengths2.inspect  # => {"apple"=>5, "banana"=>6, "cherry"=>6}
```

```ruby
# flat_map / collect_concat
nested = [[1, 2], [3, 4], [5, 6]]

# map แล้ว flatten(1)
flat = nested.flat_map { |arr| arr.map { |n| n * 2 } }
puts flat.inspect  # => [2, 4, 6, 8, 10, 12]

# ใช้กับ String
sentences = ["Hello World", "Ruby is Fun"]
words = sentences.flat_map { |s| s.split }
puts words.inspect  # => ["Hello", "World", "Ruby", "is", "Fun"]

# zip ก่อน flat_map
names = ["Alice", "Bob"]
scores = [90, 85]
combined = names.zip(scores).flat_map { |name, score| "#{name}: #{score}" }
puts combined.inspect  # => ["Alice: 90", "Bob: 85"]
```

---

## Step 66: Array Methods - Sorting

### 66.1 sort

```ruby
numbers = [5, 3, 8, 1, 9, 2, 7, 4, 6]

# sort - เรียงขึ้น (ASC)
puts numbers.sort.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]

# sort_by - เรียงตาม criteria
words = %w[banana apple cherry date elderberry]
puts words.sort.inspect       # => alphabetical
puts words.sort_by(&:length).inspect  # => ["date", "apple", "banana", "cherry", "elderberry"]

# sort ด้วย Block (Spaceship operator)
puts numbers.sort { |a, b| b <=> a }.inspect  # => [9, 8, 7, 6, 5, 4, 3, 2, 1] (DESC)

# sort_by ซับซ้อน
students = [
  { name: "Alice", grade: "B", score: 85 },
  { name: "Bob", grade: "A", score: 92 },
  { name: "Charlie", grade: "B", score: 88 },
  { name: "Dave", grade: "A", score: 90 }
]

# เรียงตาม grade แล้ว score
sorted = students.sort_by { |s| [s[:grade], -s[:score]] }
sorted.each { |s| puts "#{s[:name]}: #{s[:grade]} #{s[:score]}" }
# Alice: A... (เรียง grade ก่อน, score DESC)
```

```ruby
# min, max, minmax
arr = [5, 3, 8, 1, 9, 2]

puts arr.min       # => 1
puts arr.max       # => 9
puts arr.minmax.inspect  # => [1, 9]

# min_by, max_by, minmax_by
words = %w[cherry apple banana date]
puts words.min_by(&:length)  # => "date"
puts words.max_by(&:length)  # => "banana" หรือ "cherry"
puts words.minmax_by(&:length).inspect  # => ["date", "banana"]

# min_by ด้วย multiple criteria
students = [
  { name: "Alice", score: 85 },
  { name: "Bob", score: 92 },
  { name: "Charlie", score: 85 }
]

# นักเรียนที่คะแนนน้อยที่สุด (ถ้าเท่ากัน เรียกตามชื่อ)
worst = students.min_by { |s| [s[:score], s[:name]] }
puts worst[:name]  # => Alice (score 85, ชื่อ A ก่อน C)
```

```ruby
# reverse - กลับลำดับ
arr = [1, 2, 3, 4, 5]
puts arr.reverse.inspect  # => [5, 4, 3, 2, 1]

# shuffle - สลับแบบสุ่ม
puts arr.shuffle.inspect  # => random order

# shuffle ด้วย seed
puts arr.shuffle(random: Random.new(42)).inspect  # reproducible

# rotate - หมุน
puts arr.rotate.inspect      # => [2, 3, 4, 5, 1] (หมุนซ้าย 1)
puts arr.rotate(2).inspect   # => [3, 4, 5, 1, 2] (หมุนซ้าย 2)
puts arr.rotate(-1).inspect  # => [5, 1, 2, 3, 4] (หมุนขวา 1)
```

---

## Step 67: Array Methods - Grouping

### 67.1 group_by

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# group_by - จัดกลุ่มตาม criteria
grouped = numbers.group_by { |n| n.even? ? :even : :odd }
puts grouped.inspect
# => {:odd=>[1, 3, 5, 7, 9], :even=>[2, 4, 6, 8, 10]}

# group_by ด้วย modulo
by_mod3 = numbers.group_by { |n| n % 3 }
puts by_mod3.inspect
# => {1=>[1, 4, 7, 10], 2=>[2, 5, 8], 0=>[3, 6, 9]}

# ตัวอย่างจริง
words = %w[apple banana cherry apricot blueberry avocado]
by_first_letter = words.group_by { |w| w[0] }
puts by_first_letter.inspect
# => {"a"=>["apple", "apricot", "avocado"], "b"=>["banana", "blueberry"], "c"=>["cherry"]}
```

```ruby
# partition - แบ่งเป็น 2 กลุ่ม (true/false)
numbers = (1..10).to_a

evens, odds = numbers.partition(&:even?)
puts "Evens: #{evens.inspect}"  # => [2, 4, 6, 8, 10]
puts "Odds: #{odds.inspect}"    # => [1, 3, 5, 7, 9]

# adults, minors
people = [
  { name: "Alice", age: 25 },
  { name: "Bob", age: 17 },
  { name: "Charlie", age: 30 },
  { name: "Dave", age: 15 }
]

adults, minors = people.partition { |p| p[:age] >= 18 }
puts "ผู้ใหญ่: #{adults.map { |p| p[:name] }.inspect}"
puts "เยาวชน: #{minors.map { |p| p[:name] }.inspect}"
```

```ruby
# each_slice - แบ่งเป็น chunks ขนาดที่กำหนด
arr = (1..10).to_a

arr.each_slice(3) { |slice| puts slice.inspect }
# => [1, 2, 3]
# => [4, 5, 6]
# => [7, 8, 9]
# => [10]

# เก็บเป็น Array
slices = arr.each_slice(3).to_a
puts slices.inspect  # => [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

# each_cons - sliding window
arr.each_cons(3) { |cons| puts cons.inspect }
# => [1, 2, 3]
# => [2, 3, 4]
# => [3, 4, 5]
# ...
# => [8, 9, 10]

# ใช้ each_cons คำนวณ moving average
prices = [10, 11, 12, 11, 13, 14, 12, 15]
moving_avg = prices.each_cons(3).map { |trio| trio.sum / 3.0 }
puts moving_avg.map { |v| v.round(2) }.inspect
```

```ruby
# chunk - จัดกลุ่มตาม consecutive values
data = [1, 1, 2, 2, 3, 1, 1, 4, 4, 4]

chunks = data.chunk { |n| n }.map { |key, arr| [key, arr.length] }
puts chunks.inspect  # => [[1, 2], [2, 2], [3, 1], [1, 2], [4, 3]]

# chunk_while - จัดกลุ่ม consecutive โดย condition
sorted_data = [1, 2, 4, 9, 10, 11, 12, 15, 16, 19, 20, 21]
consecutive_groups = sorted_data.chunk_while { |a, b| b == a + 1 }.to_a
puts consecutive_groups.inspect
# => [[1, 2], [4], [9, 10, 11, 12], [15, 16], [19, 20, 21]]

# slice_when - ตรงข้าม chunk_while
consecutive_groups2 = sorted_data.slice_when { |a, b| b != a + 1 }.to_a
puts consecutive_groups2.inspect  # เหมือนกัน

# tally - นับความถี่
fruits = %w[apple banana apple cherry banana apple]
puts fruits.tally.inspect  # => {"apple"=>3, "banana"=>2, "cherry"=>1}
```

---

## Step 68: Array Methods - Combining

### 68.1 zip

```ruby
names  = ["Alice", "Bob", "Charlie"]
scores = [90, 85, 92]
grades = ["A", "B+", "A+"]

# zip - รวม Arrays คู่ขนาน
combined = names.zip(scores, grades)
puts combined.inspect
# => [["Alice", 90, "A"], ["Bob", 85, "B+"], ["Charlie", 92, "A+"]]

combined.each do |name, score, grade|
  puts "#{name}: #{score} (#{grade})"
end

# zip ด้วย Block
names.zip(scores) { |name, score| puts "#{name} ได้ #{score} คะแนน" }
```

```ruby
# flatten - ทำ Array หลายมิติให้แบน
nested = [1, [2, 3], [4, [5, 6]], 7]

puts nested.flatten.inspect      # => [1, 2, 3, 4, 5, 6, 7] (flatten ทั้งหมด)
puts nested.flatten(1).inspect   # => [1, 2, 3, 4, [5, 6], 7] (flatten 1 level)
puts nested.flatten(2).inspect   # => [1, 2, 3, 4, 5, 6, 7] (flatten 2 levels)

# compact - ลบ nil
arr = [1, nil, 2, nil, 3, nil, 4]
puts arr.compact.inspect  # => [1, 2, 3, 4]

# uniq - ลบ duplicates
arr2 = [1, 2, 2, 3, 3, 3, 4, 1]
puts arr2.uniq.inspect  # => [1, 2, 3, 4]

# uniq ด้วย Block
people = [
  { name: "Alice", age: 25 },
  { name: "Bob", age: 25 },
  { name: "Charlie", age: 30 }
]
unique_ages = people.uniq { |p| p[:age] }
puts unique_ages.inspect  # => [{:name=>"Alice", :age=>25}, {:name=>"Charlie", :age=>30}]
```

```ruby
# Set Operations
a = [1, 2, 3, 4, 5]
b = [3, 4, 5, 6, 7]

# Union (|) - รวม ไม่ซ้ำ
puts (a | b).inspect    # => [1, 2, 3, 4, 5, 6, 7]

# Intersection (&) - ส่วนร่วม
puts (a & b).inspect    # => [3, 4, 5]

# Difference (-) - ส่วนต่าง
puts (a - b).inspect    # => [1, 2]
puts (b - a).inspect    # => [6, 7]

# Concatenation (+)
puts (a + b).inspect    # => [1, 2, 3, 4, 5, 3, 4, 5, 6, 7] (ซ้ำได้)

# concat - เหมือน += แต่ mutate in place
arr = [1, 2, 3]
arr.concat([4, 5])
puts arr.inspect  # => [1, 2, 3, 4, 5]
```

---

## Step 69: Array Methods - Searching

### 69.1 find / detect

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# find / detect - หา element แรกที่ตรง
first_even = numbers.find { |n| n.even? }
puts first_even  # => 2

first_big = numbers.detect { |n| n > 7 }
puts first_big   # => 8

# ถ้าไม่พบ คืน nil
puts numbers.find { |n| n > 100 }.inspect  # => nil

# find ด้วย default (Proc)
not_found_handler = -> { "ไม่พบ" }
result = numbers.find(not_found_handler) { |n| n > 100 }
puts result  # => ไม่พบ
```

```ruby
# find_index / index - หา index ของ element
arr = ["apple", "banana", "cherry", "banana"]

puts arr.find_index("banana")              # => 1 (ตัวแรกที่พบ)
puts arr.find_index { |f| f.start_with?("c") }  # => 2
puts arr.index("cherry")                    # => 2

# rindex - หาจากขวา
puts arr.rindex("banana")  # => 3 (ตัวหลังสุด)

# bsearch - Binary Search (Array ต้องเรียงแล้ว!)
sorted = [1, 3, 5, 7, 9, 11, 13, 15]

# find mode - หา element ที่ตรง predicate
result = sorted.bsearch { |n| n >= 7 }
puts result  # => 7

# หา range
result2 = sorted.bsearch { |n| n >= 10 }
puts result2  # => 11

# minimize mode
result3 = sorted.bsearch_index { |n| n >= 7 }
puts result3  # => 3
```

---

## Step 70: Array Methods - Iteration

### 70.1 each และ Variants

```ruby
fruits = ["แอปเปิ้ล", "กล้วย", "มะม่วง"]

# each - วนซ้ำ
fruits.each { |fruit| puts fruit }

# each_with_index - วนซ้ำพร้อม index
fruits.each_with_index do |fruit, index|
  puts "#{index + 1}. #{fruit}"
end
# => 1. แอปเปิ้ล
# => 2. กล้วย
# => 3. มะม่วง

# each_with_object - วนซ้ำพร้อม accumulator
result = fruits.each_with_object([]) do |fruit, arr|
  arr << fruit.upcase
end
puts result.inspect  # => ["แอปเปิ้ล", "กล้วย", "มะม่วง"] (ยังเป็น Thai)

# each_index - วนเฉพาะ index
fruits.each_index { |i| print "#{i} " }
puts  # => 0 1 2

# reverse_each - วนย้อนกลับ
fruits.reverse_each { |fruit| puts fruit }
# => มะม่วง, กล้วย, แอปเปิ้ล
```

```ruby
# cycle - วนซ้ำวนเวียน
arr = [1, 2, 3]
arr.cycle(2) { |n| print "#{n} " }  # => 1 2 3 1 2 3
puts

# cycle ไม่มีจำนวนจะวนไปเรื่อยๆ (ระวัง infinite loop!)
# arr.cycle { |n| print n }  # => infinite loop!

# combination - สุ่มเลือก k items จาก n items
arr = [1, 2, 3, 4]
arr.combination(2).each { |c| print "#{c} " }
puts
# => [1, 2] [1, 3] [1, 4] [2, 3] [2, 4] [3, 4]

# permutation - เรียงลำดับ k items
arr.permutation(2).first(5).each { |p| print "#{p} " }
puts
# => [1, 2] [1, 3] [1, 4] [2, 1] [2, 3]

# repeated_combination
[1, 2, 3].repeated_combination(2).to_a.inspect
# => [[1, 1], [1, 2], [1, 3], [2, 2], [2, 3], [3, 3]]
```

```ruby
# inject/reduce ขั้นสูง
data = [1, 2, 3, 4, 5]

# Build running total
running_totals = data.inject([]) { |acc, n| acc + [acc.last.to_i + n] }
puts running_totals.inspect  # => [1, 3, 6, 10, 15]

# Build frequency map
words = %w[apple banana apple cherry banana apple]
freq = words.inject(Hash.new(0)) { |h, w| h[w] += 1; h }
puts freq.inspect  # => {"apple"=>3, "banana"=>2, "cherry"=>1}

# sum ด้วย initial value
puts data.sum      # => 15
puts data.sum(100) # => 115 (เพิ่ม initial value)
puts data.sum { |n| n ** 2 }  # => 55 (sum of squares)
```

---

## Step 71: Multi-dimensional Arrays

### 71.1 2D Arrays (Matrix)

```ruby
# สร้าง 2D Array
matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]

# เข้าถึง
puts matrix[1][2]     # => 6 (row 1, col 2)
puts matrix[0][0]     # => 1
puts matrix[-1][-1]   # => 9

# วน loop
matrix.each_with_index do |row, i|
  row.each_with_index do |val, j|
    print "#{val} "
  end
  puts
end

# Transpose (สลับแถวและคอลัมน์)
transposed = matrix.transpose
puts "Transposed:"
transposed.each { |row| puts row.inspect }
# [1, 4, 7]
# [2, 5, 8]
# [3, 6, 9]
```

```ruby
# สร้าง Matrix ด้วย Array.new
def create_matrix(rows, cols, default = 0)
  Array.new(rows) { Array.new(cols, default) }
end

grid = create_matrix(3, 4)
puts grid.inspect  # => [[0, 0, 0, 0], [0, 0, 0, 0], [0, 0, 0, 0]]

# Tic-Tac-Toe Board
board = create_matrix(3, 3, ".")
board[0][0] = "X"
board[1][1] = "O"
board[2][2] = "X"

board.each { |row| puts row.join(" | ") }
# X | . | .
# . | O | .
# . | . | X

# Flatten 2D to 1D
puts matrix.flatten.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

```ruby
# Array of Hashes (ใช้บ่อยมากใน Rails)
students = [
  { id: 1, name: "Alice",   score: 92, grade: "A" },
  { id: 2, name: "Bob",     score: 85, grade: "B+" },
  { id: 3, name: "Charlie", score: 78, grade: "C+" },
  { id: 4, name: "Dave",    score: 95, grade: "A+" }
]

# ค้นหา
alice = students.find { |s| s[:name] == "Alice" }
puts alice.inspect

# กรอง
honor_students = students.select { |s| s[:score] >= 90 }
puts "นักเรียนเกียรตินิยม: #{honor_students.map { |s| s[:name] }.inspect}"

# เรียงลำดับ
by_score = students.sort_by { |s| -s[:score] }
by_score.each { |s| puts "#{s[:name]}: #{s[:score]}" }

# แปลง
report = students.map { |s| "#{s[:name]} (#{s[:grade]})" }
puts report.join(", ")
```

---

## Step 72: Array Destructuring

### 72.1 Destructuring Assignment

```ruby
# Basic destructuring
first, second, third = [1, 2, 3]
puts "#{first}, #{second}, #{third}"  # => 1, 2, 3

# Splat
head, *tail = [1, 2, 3, 4, 5]
puts head          # => 1
puts tail.inspect  # => [2, 3, 4, 5]

*init, last = [1, 2, 3, 4, 5]
puts init.inspect  # => [1, 2, 3, 4]
puts last          # => 5

first, *middle, last = [1, 2, 3, 4, 5]
puts first         # => 1
puts middle.inspect # => [2, 3, 4]
puts last          # => 5

# Nested destructuring
(a, b), c = [[1, 2], 3]
puts "#{a}, #{b}, #{c}"  # => 1, 2, 3

first, (second, third) = [1, [2, 3]]
puts "#{first}, #{second}, #{third}"  # => 1, 2, 3
```

```ruby
# Destructuring ใน block
pairs = [[1, "one"], [2, "two"], [3, "three"]]
pairs.each do |(num, word)|
  puts "#{num} = #{word}"
end

# หรือแบบนี้
pairs.each do |num, word|
  puts "#{num} = #{word}"
end

# Hash destructuring ใน block
people = [
  { name: "Alice", age: 25 },
  { name: "Bob", age: 30 }
]

people.each do |person|
  name = person[:name]
  age  = person[:age]
  puts "#{name} is #{age}"
end
```

---

## Step 73: Array as Stack และ Queue

### 73.1 Stack (LIFO - Last In First Out)

```ruby
# Stack: push (เพิ่มหัว/ท้าย) และ pop (เอาออก)
stack = []

# Push
stack.push(1)
stack.push(2)
stack.push(3)
puts stack.inspect  # => [1, 2, 3]

# Pop (เอาออกจากท้าย - LIFO)
puts stack.pop   # => 3
puts stack.pop   # => 2
puts stack.inspect  # => [1]

# Peek (ดูโดยไม่เอาออก)
puts stack.last  # => 1
puts stack.inspect  # => [1] (ไม่เปลี่ยน)

# ตัวอย่าง: ตรวจ balanced parentheses
def balanced_parentheses?(str)
  stack = []
  pairs = { ")" => "(", "]" => "[", "}" => "{" }
  
  str.chars.each do |char|
    if %w[( [ {].include?(char)
      stack.push(char)
    elsif pairs.key?(char)
      return false if stack.empty? || stack.last != pairs[char]
      stack.pop
    end
  end
  
  stack.empty?
end

puts balanced_parentheses?("({[]})")  # => true
puts balanced_parentheses?("({[}])")  # => false
puts balanced_parentheses?("((())")   # => false
```

### 73.2 Queue (FIFO - First In First Out)

```ruby
# Queue: push (เพิ่มท้าย) และ shift (เอาออกจากหน้า)
queue = []

# Enqueue
queue.push("task 1")
queue.push("task 2")
queue.push("task 3")
puts queue.inspect  # => ["task 1", "task 2", "task 3"]

# Dequeue (เอาออกจากหน้า - FIFO)
puts queue.shift  # => "task 1"
puts queue.shift  # => "task 2"
puts queue.inspect  # => ["task 3"]

# Ruby มี Queue ใน standard library ด้วย
require 'thread'
q = Queue.new
q.push("item 1")
q.push("item 2")
puts q.pop  # => "item 1"

# ตัวอย่าง: Simple Task Queue
class TaskQueue
  def initialize
    @tasks = []
  end
  
  def add_task(task, priority: :normal)
    if priority == :high
      @tasks.unshift(task)  # เพิ่มหน้า
    else
      @tasks.push(task)     # เพิ่มท้าย
    end
  end
  
  def process_next
    return "ไม่มีงาน" if @tasks.empty?
    task = @tasks.shift
    yield task if block_given?
    task
  end
  
  def size = @tasks.size
  def empty? = @tasks.empty?
end

tq = TaskQueue.new
tq.add_task("อีเมล ทั่วไป")
tq.add_task("รายงานประจำปี")
tq.add_task("งานด่วน!", priority: :high)

tq.process_next { |task| puts "กำลังทำ: #{task}" }
tq.process_next { |task| puts "กำลังทำ: #{task}" }
tq.process_next { |task| puts "กำลังทำ: #{task}" }
```

---

## Step 74: Lazy Enumerator

### 74.1 Lazy Evaluation

```ruby
# ปัญหาของ Eager Evaluation
# ถ้า array ใหญ่มาก เราอาจไม่ต้องการทุก element
numbers = (1..Float::INFINITY)

# ไม่ lazy - จะ hang เพราะต้องประมวลผลทุกตัวก่อน!
# first_5_squares = numbers.map { |n| n**2 }.first(5)  # HANG!

# Lazy - ประมวลผลเฉพาะที่ต้องการ
first_5_squares = numbers.lazy.map { |n| n**2 }.first(5)
puts first_5_squares.inspect  # => [1, 4, 9, 16, 25]

# หาจำนวนเฉพาะ 10 ตัวแรก
def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n)).none? { |i| n % i == 0 }
end

first_10_primes = (2..Float::INFINITY).lazy.select { |n| prime?(n) }.first(10)
puts first_10_primes.inspect  # => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

```ruby
# Lazy Chain
result = (1..Float::INFINITY)
  .lazy
  .select { |n| n.odd? }       # เลขคี่
  .map { |n| n ** 2 }          # ยกกำลัง 2
  .select { |n| n % 3 == 0 }   # หาร 3 ลงตัว
  .first(5)

puts result.inspect  # => [9, 225, 441, 1089, 1521]
# (3^2=9, 15^2=225, 21^2=441, ...)

# สร้าง Lazy Enumerator เอง
fib = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

# หา Fibonacci ที่น้อยกว่า 100
fibs_under_100 = fib.lazy.take_while { |n| n < 100 }.to_a
puts fibs_under_100.inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
```

```ruby
# Lazy vs Eager Performance
require 'benchmark'

n = 10_000_000

Benchmark.bm(15) do |x|
  x.report("eager:") do
    (1..n).select { |i| i.odd? }.map { |i| i ** 2 }.first(10)
  end
  
  x.report("lazy:") do
    (1..n).lazy.select { |i| i.odd? }.map { |i| i ** 2 }.first(10)
  end
end
# lazy เร็วกว่ามาก!
```

---

## Step 75: Advanced Array Techniques

### 75.1 Array Methods รวม

```ruby
# take / drop
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

puts arr.take(3).inspect     # => [1, 2, 3]
puts arr.drop(7).inspect     # => [8, 9, 10]
puts arr.take_while { |n| n < 5 }.inspect  # => [1, 2, 3, 4]
puts arr.drop_while { |n| n < 5 }.inspect  # => [5, 6, 7, 8, 9, 10]

# each_with_object สร้าง Hash
words = %w[apple banana cherry date]
word_hash = words.each_with_object({}) do |word, hash|
  hash[word.to_sym] = word.length
end
puts word_hash.inspect
# => {:apple=>5, :banana=>6, :cherry=>6, :date=>4}

# zip ซับซ้อน
keys   = [:name, :age, :city]
values = ["Alice", 25, "Bangkok"]
person = Hash[keys.zip(values)]
puts person.inspect  # => {:name=>"Alice", :age=>25, :city=>"Bangkok"}

# หรือ
person2 = keys.zip(values).to_h
puts person2.inspect  # เหมือนกัน
```

```ruby
# product - Cartesian Product
colors = [:red, :green, :blue]
sizes  = [:small, :medium, :large]

combinations = colors.product(sizes)
puts combinations.length  # => 9
combinations.each { |c, s| puts "#{c}-#{s}" }
# red-small, red-medium, ..., blue-large

# repeated_permutation
puts [1, 2].repeated_permutation(2).to_a.inspect
# => [[1, 1], [1, 2], [2, 1], [2, 2]]

# flatten_map (flat_map)
result = [[1, 2], [3, 4], [5, 6]].flat_map { |pair| pair.map { |n| n * 2 } }
puts result.inspect  # => [2, 4, 6, 8, 10, 12]
```

---

## Step 76: แบบฝึกหัด

### แบบฝึกหัดที่ 1
หา unique elements และนับความถี่

```ruby
# เฉลย
def frequency_analysis(arr)
  arr.tally
     .sort_by { |_, count| -count }
     .map { |elem, count| "#{elem}: #{count} ครั้ง" }
end

data = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
frequency_analysis(data).each { |r| puts r }
# 5: 3 ครั้ง
# 1: 2 ครั้ง
# 3: 2 ครั้ง
# ...
```

### แบบฝึกหัดที่ 2
สร้าง Pagination

```ruby
# เฉลย
def paginate(array, page:, per_page: 10)
  total_pages = (array.length.to_f / per_page).ceil
  offset = (page - 1) * per_page
  
  {
    items: array[offset, per_page] || [],
    page: page,
    per_page: per_page,
    total_items: array.length,
    total_pages: total_pages,
    has_prev: page > 1,
    has_next: page < total_pages
  }
end

items = (1..100).to_a
page1 = paginate(items, page: 1, per_page: 10)
puts "Page #{page1[:page]}/#{page1[:total_pages]}: #{page1[:items].inspect}"

page3 = paginate(items, page: 3, per_page: 10)
puts "Page #{page3[:page]}/#{page3[:total_pages]}: #{page3[:items].inspect}"
```

### แบบฝึกหัดที่ 3
Flatten Array Manually

```ruby
# เฉลย
def deep_flatten(arr)
  result = []
  arr.each do |element|
    if element.is_a?(Array)
      result.concat(deep_flatten(element))
    else
      result << element
    end
  end
  result
end

nested = [1, [2, [3, [4, [5]]]], 6]
puts deep_flatten(nested).inspect  # => [1, 2, 3, 4, 5, 6]
```

### แบบฝึกหัดที่ 4
Rotate Matrix

```ruby
# เฉลย
def rotate_matrix_90(matrix)
  n = matrix.length
  # Transpose แล้ว reverse แต่ละแถว
  matrix.transpose.map(&:reverse)
end

def rotate_matrix_180(matrix)
  rotate_matrix_90(rotate_matrix_90(matrix))
end

matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
rotated = rotate_matrix_90(matrix)

puts "Original:"
matrix.each { |row| puts row.inspect }
puts "Rotated 90°:"
rotated.each { |row| puts row.inspect }
```

### แบบฝึกหัดที่ 5
สร้าง Moving Average

```ruby
# เฉลย
def moving_average(data, window)
  data.each_cons(window).map { |window_data| window_data.sum.to_f / window }
end

prices = [100, 102, 104, 103, 105, 108, 107, 110, 112, 111]
ma3 = moving_average(prices, 3)
ma5 = moving_average(prices, 5)

puts "Original: #{prices.inspect}"
puts "MA(3): #{ma3.map { |v| v.round(2) }.inspect}"
puts "MA(5): #{ma5.map { |v| v.round(2) }.inspect}"
```

### แบบฝึกหัดที่ 6
Array Set Operations

```ruby
# เฉลย
def compare_arrays(a, b)
  {
    union: a | b,
    intersection: a & b,
    diff_a_minus_b: a - b,
    diff_b_minus_a: b - a,
    symmetric_diff: (a - b) | (b - a),
    a_subset_of_b: (a - b).empty?,
    b_subset_of_a: (b - a).empty?
  }
end

a = [1, 2, 3, 4, 5]
b = [3, 4, 5, 6, 7]

result = compare_arrays(a, b)
result.each { |key, val| puts "#{key}: #{val.inspect}" }
```

### แบบฝึกหัดที่ 7
สร้าง Simple Database

```ruby
# เฉลย
class ArrayDatabase
  def initialize
    @records = []
    @next_id = 1
  end
  
  def insert(data)
    record = data.merge(id: @next_id)
    @records << record
    @next_id += 1
    record
  end
  
  def find(id)
    @records.find { |r| r[:id] == id }
  end
  
  def where(conditions)
    @records.select do |record|
      conditions.all? { |key, value| record[key] == value }
    end
  end
  
  def update(id, updates)
    record = find(id)
    return nil unless record
    record.merge!(updates)
  end
  
  def delete(id)
    @records.reject! { |r| r[:id] == id }
  end
  
  def all = @records.dup
  def count = @records.length
  
  def order_by(field, direction = :asc)
    sorted = @records.sort_by { |r| r[field] }
    direction == :desc ? sorted.reverse : sorted
  end
end

db = ArrayDatabase.new
db.insert({ name: "Alice", age: 25, city: "กรุงเทพ" })
db.insert({ name: "Bob", age: 30, city: "เชียงใหม่" })
db.insert({ name: "Charlie", age: 25, city: "กรุงเทพ" })

bkk_users = db.where(city: "กรุงเทพ")
puts "ผู้ใช้กรุงเทพ: #{bkk_users.map { |u| u[:name] }.inspect}"

age_25 = db.where(age: 25)
puts "อายุ 25: #{age_25.map { |u| u[:name] }.inspect}"

by_age = db.order_by(:age)
by_age.each { |u| puts "#{u[:name]}: #{u[:age]}" }
```

### แบบฝึกหัดที่ 8
สร้าง Array Pipeline

```ruby
# เฉลย
class ArrayPipeline
  def initialize(data)
    @data = data
  end
  
  def filter(&block)
    @data = @data.select(&block)
    self
  end
  
  def transform(&block)
    @data = @data.map(&block)
    self
  end
  
  def sort_by_field(field, direction = :asc)
    @data = @data.sort_by { |item| item[field] }
    @data.reverse! if direction == :desc
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

students = [
  { name: "Alice", score: 85, grade: "B+" },
  { name: "Bob", score: 92, grade: "A" },
  { name: "Charlie", score: 78, grade: "C+" },
  { name: "Dave", score: 95, grade: "A+" },
  { name: "Eve", score: 88, grade: "B+" }
]

result = ArrayPipeline.new(students)
  .filter { |s| s[:score] >= 85 }
  .sort_by_field(:score, :desc)
  .limit(3)
  .transform { |s| "#{s[:name]} (#{s[:score]})" }
  .result

puts result.inspect
```

### แบบฝึกหัดที่ 9
Matrix Multiplication

```ruby
# เฉลย
def matrix_multiply(a, b)
  rows_a = a.length
  cols_a = a[0].length
  cols_b = b[0].length
  
  raise "ขนาด Matrix ไม่ compatible" unless cols_a == b.length
  
  result = Array.new(rows_a) { Array.new(cols_b, 0) }
  
  rows_a.times do |i|
    cols_b.times do |j|
      cols_a.times do |k|
        result[i][j] += a[i][k] * b[k][j]
      end
    end
  end
  
  result
end

a = [[1, 2, 3], [4, 5, 6]]
b = [[7, 8], [9, 10], [11, 12]]

result = matrix_multiply(a, b)
puts "A * B ="
result.each { |row| puts row.inspect }
# [58, 64]
# [139, 154]
```

### แบบฝึกหัดที่ 10
Merge Sort

```ruby
# เฉลย
def merge_sort(arr)
  return arr if arr.length <= 1
  
  mid = arr.length / 2
  left  = merge_sort(arr[0...mid])
  right = merge_sort(arr[mid..])
  
  merge(left, right)
end

def merge(left, right)
  result = []
  
  until left.empty? || right.empty?
    if left.first <= right.first
      result << left.shift
    else
      result << right.shift
    end
  end
  
  result + left + right
end

data = [5, 3, 8, 1, 9, 2, 7, 4, 6]
puts "Before: #{data.inspect}"
puts "After: #{merge_sort(data).inspect}"
```

### แบบฝึกหัดที่ 11
Group and Aggregate

```ruby
# เฉลย
orders = [
  { customer: "Alice", product: "Apple", amount: 50, date: "2024-01" },
  { customer: "Bob", product: "Banana", amount: 30, date: "2024-01" },
  { customer: "Alice", product: "Cherry", amount: 80, date: "2024-02" },
  { customer: "Bob", product: "Apple", amount: 50, date: "2024-02" },
  { customer: "Alice", product: "Banana", amount: 30, date: "2024-01" },
  { customer: "Charlie", product: "Apple", amount: 50, date: "2024-02" }
]

# รายได้ต่อลูกค้า
puts "=== รายได้ต่อลูกค้า ==="
by_customer = orders.group_by { |o| o[:customer] }
by_customer.each do |customer, orders|
  total = orders.sum { |o| o[:amount] }
  puts "#{customer}: #{total} บาท"
end

# ยอดขายต่อเดือน
puts "\n=== ยอดขายต่อเดือน ==="
by_month = orders.group_by { |o| o[:date] }
by_month.sort.each do |month, orders|
  total = orders.sum { |o| o[:amount] }
  count = orders.length
  puts "#{month}: #{total} บาท (#{count} รายการ)"
end
```

### แบบฝึกหัดที่ 12
Implement Zip สำหรับ Multiple Arrays

```ruby
# เฉลย
def my_zip(*arrays)
  max_length = arrays.map(&:length).max
  
  max_length.times.map do |i|
    arrays.map { |arr| arr[i] }
  end
end

a = [1, 2, 3]
b = ["a", "b", "c"]
c = [:x, :y, :z]

puts my_zip(a, b, c).inspect
# => [[1, "a", :x], [2, "b", :y], [3, "c", :z]]

# Test with different lengths
d = [1, 2]
e = ["a", "b", "c"]
puts my_zip(d, e).inspect
# => [[1, "a"], [2, "b"], [nil, "c"]]
```

### แบบฝึกหัดที่ 13
Array Chunker

```ruby
# เฉลย
def chunk_into_balanced_groups(arr, num_groups)
  result = Array.new(num_groups) { [] }
  arr.each_with_index { |item, i| result[i % num_groups] << item }
  result
end

def chunk_by_size(arr, chunk_size)
  arr.each_slice(chunk_size).to_a
end

def chunk_balanced(arr, num_groups)
  # แบ่งให้กลุ่มใหญ่อยู่หน้า
  base_size = arr.length / num_groups
  remainder = arr.length % num_groups
  
  result = []
  offset = 0
  
  num_groups.times do |i|
    size = base_size + (i < remainder ? 1 : 0)
    result << arr[offset, size]
    offset += size
  end
  
  result
end

items = (1..10).to_a
puts chunk_into_balanced_groups(items, 3).inspect
# => [[1, 4, 7, 10], [2, 5, 8], [3, 6, 9]]

puts chunk_by_size(items, 3).inspect
# => [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

puts chunk_balanced(items, 3).inspect
# => [[1, 2, 3, 4], [5, 6, 7], [8, 9, 10]]
```

### แบบฝึกหัดที่ 14
Event System ด้วย Array

```ruby
# เฉลย
class EventEmitter
  def initialize
    @listeners = Hash.new { |h, k| h[k] = [] }
  end
  
  def on(event, &handler)
    @listeners[event] << handler
    self
  end
  
  def once(event, &handler)
    wrapper = nil
    wrapper = proc do |*args|
      handler.call(*args)
      @listeners[event].delete(wrapper)
    end
    @listeners[event] << wrapper
    self
  end
  
  def emit(event, *args)
    @listeners[event].each { |handler| handler.call(*args) }
    self
  end
  
  def off(event)
    @listeners.delete(event)
    self
  end
  
  def listener_count(event)
    @listeners[event].length
  end
end

emitter = EventEmitter.new

emitter.on(:data) { |msg| puts "Handler 1: #{msg}" }
emitter.on(:data) { |msg| puts "Handler 2: #{msg.upcase}" }
emitter.once(:data) { |msg| puts "Once: #{msg} (จะเรียกแค่ครั้งเดียว)" }

puts "Listeners: #{emitter.listener_count(:data)}"
emitter.emit(:data, "hello")
puts "\nหลัง emit ครั้งแรก:"
puts "Listeners: #{emitter.listener_count(:data)}"
emitter.emit(:data, "world")
```

### แบบฝึกหัดที่ 15
Sliding Window Maximum

```ruby
# เฉลย
def sliding_window_max(arr, k)
  return arr if k >= arr.length
  
  # ใช้ deque pattern
  result = []
  deque = []  # เก็บ indices
  
  arr.each_with_index do |num, i|
    # ลบ elements ที่ out of window
    deque.shift if !deque.empty? && deque.first <= i - k
    
    # ลบ elements ที่เล็กกว่า num จากท้าย
    deque.pop while !deque.empty? && arr[deque.last] <= num
    
    deque.push(i)
    
    # เพิ่ม max เมื่อ window ครบแล้ว
    result << arr[deque.first] if i >= k - 1
  end
  
  result
end

nums = [1, 3, -1, -3, 5, 3, 6, 7]
k = 3
puts "Array: #{nums.inspect}"
puts "Window size: #{k}"
puts "Max values: #{sliding_window_max(nums, k).inspect}"
# => [3, 3, 5, 5, 6, 7]
```

### แบบฝึกหัดที่ 16
Array Statistics

```ruby
# เฉลย
module ArrayStats
  def self.percentile(arr, p)
    sorted = arr.sort
    index = (p / 100.0) * (sorted.length - 1)
    
    if index == index.floor
      sorted[index.to_i]
    else
      low  = sorted[index.floor]
      high = sorted[index.ceil]
      low + (high - low) * (index - index.floor)
    end
  end
  
  def self.iqr(arr)
    q75 = percentile(arr, 75)
    q25 = percentile(arr, 25)
    q75 - q25
  end
  
  def self.outliers(arr)
    q1  = percentile(arr, 25)
    q3  = percentile(arr, 75)
    iqr = q3 - q1
    lower = q1 - 1.5 * iqr
    upper = q3 + 1.5 * iqr
    
    arr.select { |n| n < lower || n > upper }
  end
end

data = [2, 3, 3, 4, 5, 6, 6, 7, 8, 10, 100]  # 100 เป็น outlier
puts "P25: #{ArrayStats.percentile(data, 25)}"
puts "Median: #{ArrayStats.percentile(data, 50)}"
puts "P75: #{ArrayStats.percentile(data, 75)}"
puts "IQR: #{ArrayStats.iqr(data)}"
puts "Outliers: #{ArrayStats.outliers(data).inspect}"
```

### แบบฝึกหัดที่ 17
Implement Array Flatten

```ruby
# เฉลย
def my_flatten(arr, depth = Float::INFINITY)
  result = []
  
  arr.each do |element|
    if element.is_a?(Array) && depth > 0
      result.concat(my_flatten(element, depth - 1))
    else
      result << element
    end
  end
  
  result
end

nested = [1, [2, [3, [4, [5]]]]]
puts my_flatten(nested).inspect      # => [1, 2, 3, 4, 5]
puts my_flatten(nested, 1).inspect   # => [1, 2, [3, [4, [5]]]]
puts my_flatten(nested, 2).inspect   # => [1, 2, 3, [4, [5]]]
```

### แบบฝึกหัดที่ 18
Binary Search

```ruby
# เฉลย
def binary_search(arr, target)
  low = 0
  high = arr.length - 1
  
  while low <= high
    mid = (low + high) / 2
    
    case arr[mid] <=> target
    when 0  then return mid    # พบ
    when -1 then low = mid + 1  # target อยู่ทางขวา
    when 1  then high = mid - 1 # target อยู่ทางซ้าย
    end
  end
  
  nil  # ไม่พบ
end

sorted = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
puts binary_search(sorted, 7)    # => 3 (index)
puts binary_search(sorted, 15)   # => 7 (index)
puts binary_search(sorted, 6).inspect  # => nil
```

### แบบฝึกหัดที่ 19
Implement Quick Sort

```ruby
# เฉลย
def quicksort(arr)
  return arr if arr.length <= 1
  
  pivot = arr[arr.length / 2]
  left   = arr.select { |x| x < pivot }
  middle = arr.select { |x| x == pivot }
  right  = arr.select { |x| x > pivot }
  
  quicksort(left) + middle + quicksort(right)
end

data = [3, 6, 8, 10, 1, 2, 1]
puts quicksort(data).inspect  # => [1, 1, 2, 3, 6, 8, 10]

# In-place quicksort
def quicksort_in_place(arr, low = 0, high = arr.length - 1)
  return arr if low >= high
  
  pivot_index = partition!(arr, low, high)
  quicksort_in_place(arr, low, pivot_index - 1)
  quicksort_in_place(arr, pivot_index + 1, high)
  arr
end

def partition!(arr, low, high)
  pivot = arr[high]
  i = low - 1
  
  (low...high).each do |j|
    if arr[j] <= pivot
      i += 1
      arr[i], arr[j] = arr[j], arr[i]
    end
  end
  
  arr[i+1], arr[high] = arr[high], arr[i+1]
  i + 1
end

data2 = [3, 6, 8, 10, 1, 2, 1]
puts quicksort_in_place(data2).inspect  # => [1, 1, 2, 3, 6, 8, 10]
```

### แบบฝึกหัดที่ 20
Shopping Cart

```ruby
# เฉลย
class ShoppingCart
  Item = Struct.new(:name, :price, :quantity) do
    def subtotal = price * quantity
    def to_s     = "#{name} x#{quantity} @ ฿#{price} = ฿#{subtotal}"
  end
  
  include Enumerable
  
  def initialize
    @items = []
    @discount_code = nil
    @discounts = { "SAVE10" => 0.10, "SAVE20" => 0.20 }
  end
  
  def add(name, price, quantity = 1)
    existing = @items.find { |item| item.name == name }
    
    if existing
      existing.quantity += quantity
    else
      @items << Item.new(name, price, quantity)
    end
    
    self
  end
  
  def remove(name)
    @items.reject! { |item| item.name == name }
    self
  end
  
  def apply_discount(code)
    @discount_code = code if @discounts.key?(code)
    self
  end
  
  def subtotal
    @items.sum(&:subtotal)
  end
  
  def discount_amount
    return 0 unless @discount_code
    subtotal * @discounts[@discount_code]
  end
  
  def total
    subtotal - discount_amount
  end
  
  def each(&block)
    @items.each(&block)
  end
  
  def empty? = @items.empty?
  def count  = @items.length
  
  def receipt
    puts "=" * 50
    puts "ใบเสร็จ"
    puts "=" * 50
    @items.each { |item| puts item }
    puts "-" * 50
    puts "ยอดรวม: ฿#{"%.2f" % subtotal}"
    if @discount_code
      puts "ส่วนลด (#{@discount_code}): -฿#{"%.2f" % discount_amount}"
    end
    puts "ยอดสุทธิ: ฿#{"%.2f" % total}"
    puts "=" * 50
  end
end

cart = ShoppingCart.new
cart.add("แอปเปิ้ล", 50, 3)
     .add("กล้วย", 20, 5)
     .add("มะม่วง", 80, 2)
     .apply_discount("SAVE10")

cart.receipt

puts "\nสินค้าทั้งหมด: #{cart.count} รายการ"
puts "สินค้าราคาแพงสุด: #{cart.max_by(&:price).name}"
puts "สินค้าที่สั่งมากสุด: #{cart.max_by(&:quantity).name}"
```

---

## สรุป

ในตอนที่ 5 นี้ เราได้เรียนรู้:

1. **การสร้าง Array** - Literal, Array.new, %w, %i, Range.to_a
2. **การเข้าถึง Elements** - Index, first, last, sample, at, fetch, dig
3. **Modifying** - push/<<, pop, shift, unshift, insert, delete, delete_at
4. **Checking** - empty?, include?, any?, all?, none?, one?, count
5. **Transforming** - map, select/filter, reject, reduce/inject, flat_map, filter_map
6. **Sorting** - sort, sort_by, reverse, shuffle, min, max, minmax
7. **Grouping** - group_by, partition, each_slice, each_cons, chunk, tally
8. **Combining** - zip, flatten, compact, uniq, concat, |, &, -
9. **Searching** - find/detect, find_index, bsearch
10. **Iteration** - each, each_with_index, each_with_object
11. **Multi-dimensional** - 2D Arrays, Matrix operations
12. **Destructuring** - Parallel assignment, Splat operator
13. **Stack/Queue** - push/pop, shift/unshift patterns
14. **Lazy Enumerator** - lazy, take_while, Infinite sequences

ในตอนต่อไปเราจะเรียนรู้เรื่อง **Hashes** ซึ่งเป็น key-value data structure ที่ทรงพลัง

---

*ตอนที่ 5 จบแล้ว - ไปต่อตอนที่ 6: Hashes*
