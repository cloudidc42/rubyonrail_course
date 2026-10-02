# ตอนที่ 5: Arrays (Steps 61-80)

## บทนำ

Array คือ Collection ที่เก็บข้อมูลหลายค่าในลำดับที่กำหนด ใน Ruby, Array มีความยืดหยุ่นสูงมาก - สามารถเก็บข้อมูลหลายประเภทในอาร์เรย์เดียว, ปรับขนาดได้ dynamic, และมีเมธอดที่ทรงพลังกว่า 100 รายการสำหรับการจัดการข้อมูล

Array เป็น backbone ของการเขียนโปรแกรม Ruby สมัยใหม่ เพราะ Enumerable module ที่ Array ใช้นั้นมีเมธอดที่ช่วยให้เขียน functional-style code ได้อย่างสวยงาม

---

## Step 61: Creating Arrays

### วิธีสร้าง Array

```ruby
# วิธีที่ 1: Array literal []
empty_array = []
numbers = [1, 2, 3, 4, 5]
mixed = [1, "hello", :symbol, true, nil, 3.14]
nested = [[1, 2], [3, 4], [5, 6]]

puts empty_array.inspect  # => []
puts numbers.inspect      # => [1, 2, 3, 4, 5]
puts mixed.inspect        # => [1, "hello", :symbol, true, nil, 3.14]
```

```ruby
# วิธีที่ 2: Array.new
empty = Array.new         # => []
sized = Array.new(5)      # => [nil, nil, nil, nil, nil]
filled = Array.new(5, 0)  # => [0, 0, 0, 0, 0]
filled_str = Array.new(3, "hi")  # => ["hi", "hi", "hi"]

# ข้อระวัง: Array.new(3, []) สร้าง array เดียวกัน 3 ครั้ง
shared = Array.new(3, [])
shared[0] << 1
puts shared.inspect  # => [[1], [1], [1]] (ทั้ง 3 ชี้ array เดียวกัน!)

# ใช้ block แทน
independent = Array.new(3) { [] }
independent[0] << 1
puts independent.inspect  # => [[1], [], []] (แต่ละตัวอิสระ)
```

```ruby
# วิธีที่ 3: Array.new กับ block
squares = Array.new(10) { |i| i ** 2 }
puts squares.inspect
# => [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

countdown = Array.new(5) { |i| 5 - i }
puts countdown.inspect
# => [5, 4, 3, 2, 1]

fibonacci = Array.new(10) { |i| i <= 1 ? i : nil }
# (สร้าง fibonacci จริงๆ ต้องใช้ reduce หรือ loop)
```

```ruby
# วิธีที่ 4: %w[] - Word array (strings)
fruits = %w[apple banana cherry date elderberry]
puts fruits.inspect
# => ["apple", "banana", "cherry", "date", "elderberry"]

# ดีกว่าการพิมพ์ quotes ทุกตัว
# เทียบกับ: ["apple", "banana", "cherry"]

# %w ไม่รองรับ interpolation
name = "ruby"
arr = %w[hello #{name} world]
puts arr.inspect  # => ["hello", "\#{name}", "world"] (ไม่แปลง)
```

```ruby
# วิธีที่ 5: %i[] - Symbol array
statuses = %i[pending processing completed failed cancelled]
puts statuses.inspect
# => [:pending, :processing, :completed, :failed, :cancelled]

# ใช้บ่อยใน Rails
# validates :status, inclusion: { in: %i[active inactive] }
```

```ruby
# วิธีที่ 6: Kernel#Array()  
puts Array(nil).inspect     # => []
puts Array(1).inspect       # => [1]
puts Array([1,2]).inspect   # => [1, 2]
puts Array("hello").inspect # => ["hello"]

# มีประโยชน์ในการ normalize input
def process(items)
  Array(items).each { |item| puts item }
end

process("single item")    # => "single item"
process(["a", "b", "c"])  # => a, b, c
process(nil)              # => (ไม่มี output)
```

---

## Step 62: Array Accessing

### การเข้าถึงข้อมูล

```ruby
fruits = ["apple", "banana", "cherry", "date", "elderberry"]

# ด้วย index (เริ่มที่ 0)
puts fruits[0]      # => apple
puts fruits[1]      # => banana
puts fruits[-1]     # => elderberry (นับจากหลัง)
puts fruits[-2]     # => date

# ด้วย range
puts fruits[1..3].inspect    # => ["banana", "cherry", "date"]
puts fruits[1...3].inspect   # => ["banana", "cherry"] (exclusive)
puts fruits[2..].inspect     # => ["cherry", "date", "elderberry"]
puts fruits[..2].inspect     # => ["apple", "banana", "cherry"]
```

```ruby
# at - เหมือน [] แต่รับ index เท่านั้น
puts fruits.at(0)    # => apple
puts fruits.at(-1)   # => elderberry
puts fruits.at(10)   # => nil (ไม่ raise error)

# fetch - คล้าย [] แต่ raise error ถ้าไม่พบ
puts fruits.fetch(0)          # => apple
# fruits.fetch(10)            # => IndexError!
puts fruits.fetch(10, "N/A")  # => N/A (default value)
puts fruits.fetch(10) { |i| "index #{i} not found" }  # => index 10 not found
```

```ruby
# first / last
puts fruits.first         # => apple
puts fruits.first(3).inspect  # => ["apple", "banana", "cherry"]
puts fruits.last          # => elderberry
puts fruits.last(2).inspect   # => ["date", "elderberry"]

# sample - สุ่ม
puts fruits.sample           # => สุ่ม 1 ตัว
puts fruits.sample(3).inspect # => สุ่ม 3 ตัว

# take / drop
puts fruits.take(3).inspect      # => ["apple", "banana", "cherry"]
puts fruits.drop(3).inspect      # => ["date", "elderberry"]
puts fruits.take_while { |f| f.length < 7 }.inspect  # => ["apple"]
puts fruits.drop_while { |f| f.length < 7 }.inspect  # => ["banana", "cherry", "date", "elderberry"]
```

---

## Step 63: Modifying Arrays

### การแก้ไข Array

```ruby
numbers = [1, 2, 3, 4, 5]

# push / << - เพิ่มท้าย
numbers.push(6)
numbers << 7
numbers.push(8, 9, 10)
puts numbers.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# pop - ลบและคืนค่าตัวสุดท้าย
last = numbers.pop
puts last     # => 10
puts numbers.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]

# pop หลายตัว
popped = numbers.pop(3)
puts popped.inspect   # => [7, 8, 9]
```

```ruby
arr = [1, 2, 3, 4, 5]

# unshift - เพิ่มที่ต้น
arr.unshift(0)
puts arr.inspect  # => [0, 1, 2, 3, 4, 5]

arr.unshift(-2, -1)
puts arr.inspect  # => [-2, -1, 0, 1, 2, 3, 4, 5]

# shift - ลบและคืนค่าตัวแรก
first = arr.shift
puts first    # => -2
puts arr.inspect  # => [-1, 0, 1, 2, 3, 4, 5]

shifted = arr.shift(2)
puts shifted.inspect   # => [-1, 0]
```

```ruby
arr = [1, 2, 3, 4, 5]

# insert - แทรกที่ตำแหน่งใดก็ได้
arr.insert(2, 99)
puts arr.inspect  # => [1, 2, 99, 3, 4, 5]

arr.insert(-1, 100)  # -1 = ท้าย
puts arr.inspect  # => [1, 2, 99, 3, 4, 5, 100]

arr.insert(0, 0)  # 0 = ต้น
puts arr.inspect  # => [0, 1, 2, 99, 3, 4, 5, 100]
```

```ruby
arr = [1, 2, 3, 2, 4, 2, 5]

# delete - ลบค่าที่กำหนดทั้งหมด
arr.delete(2)
puts arr.inspect  # => [1, 3, 4, 5]

# delete_at - ลบที่ index ที่กำหนด
arr2 = [10, 20, 30, 40, 50]
arr2.delete_at(2)
puts arr2.inspect  # => [10, 20, 40, 50]

# delete_if - ลบตามเงื่อนไข (เหมือน reject!)
arr3 = [1, 2, 3, 4, 5, 6]
arr3.delete_if { |x| x.even? }
puts arr3.inspect  # => [1, 3, 5]

# keep_if - เก็บตามเงื่อนไข (เหมือน select!)
arr4 = [1, 2, 3, 4, 5, 6]
arr4.keep_if { |x| x.even? }
puts arr4.inspect  # => [2, 4, 6]
```

---

## Step 64: Array Information Methods

### การตรวจสอบ Array

```ruby
arr = [1, 2, 3, 4, 5]

# ขนาด
puts arr.length   # => 5
puts arr.size     # => 5 (เหมือนกัน)
puts arr.count    # => 5
puts arr.count { |x| x.even? }  # => 2 (นับตามเงื่อนไข)

# ตรวจสอบว่างหรือไม่
puts arr.empty?   # => false
puts [].empty?    # => true
```

```ruby
arr = [1, 2, 3, 4, 5]

# include? - ตรวจสอบว่ามีค่านั้นหรือไม่
puts arr.include?(3)   # => true
puts arr.include?(10)  # => false

# any? - มีอย่างน้อย 1 ตัวที่ตรงเงื่อนไข
puts arr.any? { |x| x > 4 }   # => true (5 > 4)
puts arr.any? { |x| x > 10 }  # => false

# all? - ทุกตัวต้องตรงเงื่อนไข
puts arr.all? { |x| x > 0 }   # => true
puts arr.all? { |x| x > 3 }   # => false

# none? - ไม่มีตัวใดที่ตรงเงื่อนไข
puts arr.none? { |x| x > 10 } # => true
puts arr.none? { |x| x > 3 }  # => false

# one? - มีแค่ 1 ตัวที่ตรงเงื่อนไข
puts arr.one? { |x| x == 3 }  # => true
puts arr.one? { |x| x > 3 }   # => false (2 ตัว: 4 และ 5)
```

```ruby
# ตรวจสอบ flatten
nested = [[1, 2], [3, [4, 5]]]
puts nested.flatten.inspect      # => [1, 2, 3, 4, 5]
puts nested.flatten(1).inspect   # => [1, 2, 3, [4, 5]] (1 level only)

# ตรวจสอบว่า flat หรือไม่
def flat_array?(arr)
  arr.none? { |x| x.is_a?(Array) }
end

puts flat_array?([1, 2, 3])      # => true
puts flat_array?([[1, 2], 3])    # => false
```

---

## Step 65: Transforming Arrays

### การแปลง Array

```ruby
numbers = [1, 2, 3, 4, 5]

# map / collect - แปลงทุกตัว (สร้าง array ใหม่)
squares = numbers.map { |n| n ** 2 }
puts squares.inspect  # => [1, 4, 9, 16, 25]

doubled = numbers.map { |n| n * 2 }
puts doubled.inspect  # => [2, 4, 6, 8, 10]

strings = numbers.map(&:to_s)
puts strings.inspect  # => ["1", "2", "3", "4", "5"]
```

```ruby
# map กับ index
letters = %w[a b c d e]
with_index = letters.map.with_index { |char, i| "#{i}:#{char}" }
puts with_index.inspect  # => ["0:a", "1:b", "2:c", "3:d", "4:e"]

# map.with_index starting from 1
with_index2 = letters.map.with_index(1) { |char, i| "#{i}:#{char}" }
puts with_index2.inspect  # => ["1:a", "2:b", "3:c", "4:d", "5:e"]
```

```ruby
# select / filter - เลือกตัวที่ตรงเงื่อนไข
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
evens = numbers.select { |n| n.even? }
puts evens.inspect  # => [2, 4, 6, 8, 10]

big_numbers = numbers.select { |n| n > 5 }
puts big_numbers.inspect  # => [6, 7, 8, 9, 10]

# reject - ตรงข้าม select
odds = numbers.reject { |n| n.even? }
puts odds.inspect  # => [1, 3, 5, 7, 9]
```

```ruby
# flat_map - map แล้ว flatten 1 level
words = ["hello world", "foo bar", "ruby programming"]
all_words = words.flat_map { |s| s.split(" ") }
puts all_words.inspect
# => ["hello", "world", "foo", "bar", "ruby", "programming"]

# เทียบกับ map แล้ว flatten
result = words.map { |s| s.split(" ") }.flatten
puts result.inspect  # ผลเหมือนกัน แต่ flat_map เร็วกว่า

# ใช้กับ nested data
users = [
  { name: "Alice", hobbies: ["reading", "coding"] },
  { name: "Bob", hobbies: ["gaming", "cooking", "reading"] }
]
all_hobbies = users.flat_map { |u| u[:hobbies] }
puts all_hobbies.inspect
# => ["reading", "coding", "gaming", "cooking", "reading"]
```

---

## Step 66: Reducing Arrays

### การรวมค่าใน Array

```ruby
numbers = [1, 2, 3, 4, 5]

# reduce / inject - รวมค่าทีละตัว
sum = numbers.reduce(0) { |acc, n| acc + n }
puts sum  # => 15

product = numbers.reduce(1) { |acc, n| acc * n }
puts product  # => 120

# ใช้ Symbol shorthand
puts numbers.reduce(:+)   # => 15
puts numbers.reduce(:*)   # => 120

# inject (alias ของ reduce)
puts numbers.inject(:+)   # => 15
```

```ruby
# reduce กับ initial value
words = ["hello", "world", "ruby"]
total_length = words.reduce(0) { |sum, w| sum + w.length }
puts total_length  # => 14 (5 + 5 + 4)

# ไม่ระบุ initial value - ใช้ตัวแรกเป็น accumulator
first_and_rest = [1, 2, 3, 4, 5].reduce { |acc, n| acc + n }
puts first_and_rest  # => 15 (เหมือนกัน แต่ไม่มี initial)

# สร้าง Hash จาก Array ด้วย reduce
pairs = [["a", 1], ["b", 2], ["c", 3]]
hash = pairs.reduce({}) { |h, (k, v)| h.merge(k => v) }
puts hash.inspect  # => {"a"=>1, "b"=>2, "c"=>3}
```

```ruby
# each_with_object - คล้าย reduce แต่ใช้ object เป็น accumulator
numbers = [1, 2, 3, 4, 5]

# สร้าง hash จาก array
result = numbers.each_with_object({}) do |n, hash|
  hash[n] = n ** 2
end
puts result.inspect  # => {1=>1, 2=>4, 3=>9, 4=>16, 5=>25}

# แบ่งตาม odd/even
result2 = numbers.each_with_object({ odd: [], even: [] }) do |n, groups|
  key = n.odd? ? :odd : :even
  groups[key] << n
end
puts result2.inspect  # => {:odd=>[1, 3, 5], :even=>[2, 4]}
```

```ruby
# sum - รวมทั้งหมด
puts [1, 2, 3, 4, 5].sum        # => 15
puts [1.5, 2.5, 3.0].sum        # => 7.0

# sum กับ block
puts [1, 2, 3, 4, 5].sum { |n| n ** 2 }  # => 55 (1+4+9+16+25)

# sum กับ initial value
puts [1, 2, 3].sum(10)           # => 16 (10 + 1 + 2 + 3)
```

---

## Step 67: Sorting Arrays

### การเรียงลำดับ

```ruby
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5]

# sort - เรียงลำดับน้อยไปมาก
puts numbers.sort.inspect     # => [1, 1, 2, 3, 4, 5, 5, 6, 9]

# sort กับ block (spaceship operator)
puts numbers.sort { |a, b| a <=> b }.inspect   # เรียง ascending
puts numbers.sort { |a, b| b <=> a }.inspect   # เรียง descending

# reverse - กลับลำดับ
puts numbers.sort.reverse.inspect  # => [9, 6, 5, 5, 4, 3, 2, 1, 1]
```

```ruby
# sort_by - เรียงตาม key ที่กำหนด
people = [
  { name: "Charlie", age: 30 },
  { name: "Alice", age: 25 },
  { name: "Bob", age: 35 }
]

by_name = people.sort_by { |p| p[:name] }
by_name.each { |p| puts "#{p[:name]} (#{p[:age]})" }
# => Alice (25), Bob (35), Charlie (30)

by_age = people.sort_by { |p| p[:age] }
by_age.each { |p| puts "#{p[:name]} (#{p[:age]})" }
# => Alice (25), Charlie (30), Bob (35)

# เรียง descending
by_age_desc = people.sort_by { |p| -p[:age] }
by_age_desc.each { |p| puts "#{p[:name]}: #{p[:age]}" }
# => Bob: 35, Charlie: 30, Alice: 25
```

```ruby
# min / max
puts [3, 1, 4, 1, 5, 9, 2, 6].min   # => 1
puts [3, 1, 4, 1, 5, 9, 2, 6].max   # => 9
puts [3, 1, 4, 1, 5, 9, 2, 6].minmax.inspect  # => [1, 9]

# min_by / max_by
words = ["apple", "fig", "banana", "date"]
puts words.min_by(&:length)   # => fig (shortest)
puts words.max_by(&:length)   # => banana (longest)
puts words.minmax_by(&:length).inspect  # => ["fig", "banana"]

# min(n) / max(n) - เอา n ตัวแรก/สุดท้าย
puts [5, 3, 8, 1, 9, 2, 7].min(3).inspect  # => [1, 2, 3]
puts [5, 3, 8, 1, 9, 2, 7].max(3).inspect  # => [9, 8, 7]
```

```ruby
# shuffle - สับไพ่ (randomize)
deck = (1..10).to_a
puts deck.shuffle.inspect  # => สุ่มลำดับ

# shuffle กับ seed (reproducible)
puts deck.shuffle(random: Random.new(42)).inspect
puts deck.shuffle(random: Random.new(42)).inspect  # เหมือนกัน (seed เดิม)
```

---

## Step 68: Grouping Arrays

### การจัดกลุ่ม

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# group_by - จัดกลุ่มตาม key
grouped = numbers.group_by { |n| n.even? ? :even : :odd }
puts grouped.inspect
# => {:odd=>[1, 3, 5, 7, 9], :even=>[2, 4, 6, 8, 10]}

# กลุ่มตาม remainder
by_mod3 = numbers.group_by { |n| n % 3 }
puts by_mod3.inspect
# => {1=>[1, 4, 7, 10], 2=>[2, 5, 8], 0=>[3, 6, 9]}
```

```ruby
# tally - นับจำนวนของแต่ละค่า
colors = %w[red blue red green blue red purple green]
puts colors.tally.inspect
# => {"red"=>3, "blue"=>2, "green"=>2, "purple"=>1}

# tally_by (Ruby 3.1+)
# words = %w[hello world ruby hi]
# puts words.tally_by(&:length).inspect
# แทน:
words = %w[hello world ruby hi bye]
count_by_length = words.group_by(&:length).transform_values(&:count)
puts count_by_length.inspect  # => {5=>2, 4=>2, 2=>1, 3=>1}
```

```ruby
# partition - แบ่งออกเป็น 2 กลุ่ม (true/false)
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
evens, odds = numbers.partition { |n| n.even? }
puts "Evens: #{evens.inspect}"   # => [2, 4, 6, 8, 10]
puts "Odds: #{odds.inspect}"     # => [1, 3, 5, 7, 9]

# ใช้งานจริง: แยก pass/fail
students = [
  { name: "Alice", score: 85 },
  { name: "Bob", score: 55 },
  { name: "Charlie", score: 90 },
  { name: "Dave", score: 45 }
]

passed, failed = students.partition { |s| s[:score] >= 60 }
puts "ผ่าน: #{passed.map { |s| s[:name] }.join(', ')}"
puts "ไม่ผ่าน: #{failed.map { |s| s[:name] }.join(', ')}"
```

```ruby
# chunk - จัดกลุ่มค่าต่อเนื่อง
numbers = [1, 1, 2, 2, 3, 1, 1, 4, 4, 4]
chunks = numbers.chunk { |n| n }
chunks.each { |key, arr| puts "#{key}: #{arr.inspect}" }
# => 1: [1, 1]
# => 2: [2, 2]
# => 3: [3]
# => 1: [1, 1]
# => 4: [4, 4, 4]

# chunk_while - groupby consecutive
nums = [1, 2, 3, 5, 6, 10, 11, 12]
runs = nums.chunk_while { |i, j| j - i == 1 }.to_a
puts runs.inspect
# => [[1, 2, 3], [5, 6], [10, 11, 12]]
```

```ruby
# each_slice - แบ่งเป็น slices ขนาดที่กำหนด
(1..10).each_slice(3) { |slice| puts slice.inspect }
# => [1, 2, 3]
# => [4, 5, 6]
# => [7, 8, 9]
# => [10]

# each_cons - sliding window
(1..5).each_cons(3) { |cons| puts cons.inspect }
# => [1, 2, 3]
# => [2, 3, 4]
# => [3, 4, 5]
```

---

## Step 69: Searching in Arrays

### การค้นหาใน Array

```ruby
fruits = ["apple", "banana", "cherry", "date", "elderberry"]

# find / detect - หาตัวแรกที่ตรงเงื่อนไข
puts fruits.find { |f| f.length > 5 }    # => banana
puts fruits.detect { |f| f.include?("a") } # => apple

# find_index / index - หา index
puts fruits.find_index("cherry")           # => 2
puts fruits.index("cherry")               # => 2 (alias)
puts fruits.find_index { |f| f.length > 8 } # => 4 (elderberry)

# rindex - index สุดท้าย
arr = [1, 2, 3, 2, 1]
puts arr.rindex(2)   # => 3 (ตัวสุดท้ายที่เป็น 2)
```

```ruby
# include? - ตรวจสอบว่ามีค่านั้นหรือไม่ (linear search)
puts fruits.include?("cherry")  # => true
puts fruits.include?("mango")   # => false

# bsearch - binary search (array ต้องเรียงลำดับแล้ว!)
sorted_nums = [1, 3, 5, 7, 9, 11, 13, 15]

# find mode
result = sorted_nums.bsearch { |n| n >= 7 }
puts result  # => 7 (หาค่า >= 7 ตัวแรก)

# bsearch มี 2 mode:
# find-minimum mode: block คืน true/false
# find-any mode: block คืน 0, +, - (ด้วย <=>)
```

```ruby
# grep - หาด้วย === (pattern matching)
words = %w[hello world ruby rails programming]
puts words.grep(/^r/).inspect    # => ["ruby", "rails"]
puts words.grep(/ing$/).inspect  # => ["programming"]

numbers = [1, 2, 3, "four", 5, :six, 7.0]
puts numbers.grep(Integer).inspect  # => [1, 2, 3, 5]
puts numbers.grep(String).inspect   # => ["four"]
puts numbers.grep(Symbol).inspect   # => [:six]

# grep_v - ตรงข้าม grep
puts words.grep_v(/^r/).inspect   # => ["hello", "world", "programming"]
```

---

## Step 70: Combining Arrays

### การรวม Array

```ruby
a = [1, 2, 3]
b = [3, 4, 5]

# + - รวม (สร้าง array ใหม่)
puts (a + b).inspect   # => [1, 2, 3, 3, 4, 5] (อาจมีซ้ำ)

# - - ลบค่าที่มีใน b ออกจาก a
puts (a - b).inspect   # => [1, 2]

# | - Union (ไม่มีซ้ำ)
puts (a | b).inspect   # => [1, 2, 3, 4, 5]

# & - Intersection (ค่าที่มีในทั้งสอง)
puts (a & b).inspect   # => [3]
```

```ruby
# concat - ต่อเข้าหากัน (แก้ไข array เดิม)
arr = [1, 2, 3]
arr.concat([4, 5, 6])
puts arr.inspect  # => [1, 2, 3, 4, 5, 6]

# * - repeat หรือ join
puts ([1, 2, 3] * 3).inspect  # => [1, 2, 3, 1, 2, 3, 1, 2, 3]
puts ([1, 2, 3] * ", ")       # => "1, 2, 3" (join ด้วย separator)
```

```ruby
# flatten - ทำให้ nested array กลายเป็น flat
nested = [1, [2, 3], [4, [5, 6]]]
puts nested.flatten.inspect    # => [1, 2, 3, 4, 5, 6] (all levels)
puts nested.flatten(1).inspect # => [1, 2, 3, 4, [5, 6]] (1 level only)

# compact - ลบ nil values ออก
arr = [1, nil, 2, nil, 3, nil, nil, 4]
puts arr.compact.inspect  # => [1, 2, 3, 4]

# uniq - ลบค่าซ้ำ
arr2 = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4]
puts arr2.uniq.inspect  # => [1, 2, 3, 4]

# uniq กับ block
words = %w[Apple apple APPLE banana BANANA]
puts words.uniq { |w| w.downcase }.inspect  # => ["Apple", "banana"]
```

```ruby
# zip - รวมหลาย array element by element
names = %w[Alice Bob Charlie]
ages = [25, 30, 35]
cities = %w[Bangkok Chiang\ Mai Phuket]

zipped = names.zip(ages, cities)
zipped.each { |name, age, city| puts "#{name} (#{age}) - #{city}" }
# => Alice (25) - Bangkok
# => Bob (30) - Chiang Mai
# => Charlie (35) - Phuket

# zip เป็น inverse ของ transpose
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
puts matrix.transpose.inspect  # => [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
```

---

## Step 71: Array Iteration

### การวนซ้ำ Array

```ruby
fruits = %w[apple banana cherry]

# each - วนดูทุกตัว
fruits.each { |f| puts f }
# หรือ
fruits.each do |fruit|
  puts "I like #{fruit}"
end

# each_with_index
fruits.each_with_index do |fruit, i|
  puts "#{i + 1}. #{fruit}"
end
# => 1. apple, 2. banana, 3. cherry
```

```ruby
# each_with_object
fruits = %w[apple banana cherry]
result = fruits.each_with_object({}) do |fruit, hash|
  hash[fruit] = fruit.length
end
puts result.inspect  # => {"apple"=>5, "banana"=>6, "cherry"=>6}

# reverse_each - วนย้อนหลัง
fruits.reverse_each { |f| puts f }
# => cherry, banana, apple

# each_slice - วนทีละกลุ่ม
(1..9).each_slice(3) do |group|
  puts group.inspect
end
# => [1, 2, 3], [4, 5, 6], [7, 8, 9]
```

```ruby
# Chaining iterators
numbers = (1..10).to_a

result = numbers
  .select { |n| n.odd? }       # เลือกคี่: [1, 3, 5, 7, 9]
  .map { |n| n ** 2 }          # ยกกำลัง: [1, 9, 25, 49, 81]
  .reject { |n| n > 50 }       # ตัด > 50: [1, 9, 25, 49]
  .reduce(:+)                   # รวม: 84

puts result  # => 84
```

---

## Step 72: Multi-dimensional Arrays

### Array หลายมิติ

```ruby
# 2D array (matrix)
matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]

# เข้าถึง
puts matrix[1][2]  # => 6 (row 1, col 2)

# วนซ้ำ
matrix.each_with_index do |row, i|
  row.each_with_index do |val, j|
    print "#{val} "
  end
  puts
end
# => 1 2 3
# => 4 5 6
# => 7 8 9
```

```ruby
# สร้าง 2D array ที่ไม่แชร์ references
rows, cols = 3, 4
grid = Array.new(rows) { Array.new(cols, 0) }

# ใส่ค่า
grid[1][2] = 5
puts grid.inspect
# => [[0, 0, 0, 0], [0, 0, 5, 0], [0, 0, 0, 0]]

# Matrix operations
def matrix_sum(a, b)
  a.map.with_index { |row, i| row.map.with_index { |val, j| val + b[i][j] } }
end

a = [[1, 2], [3, 4]]
b = [[5, 6], [7, 8]]
puts matrix_sum(a, b).inspect  # => [[6, 8], [10, 12]]
```

```ruby
# ใช้งานจริง: Game board (Tic-Tac-Toe)
class TicTacToe
  def initialize
    @board = Array.new(3) { Array.new(3, ".") }
  end
  
  def place(row, col, player)
    @board[row][col] = player
  end
  
  def display
    @board.each_with_index do |row, i|
      puts row.join(" | ")
      puts "---------" if i < 2
    end
  end
  
  def winner?
    # ตรวจ rows
    @board.each { |row| return row[0] if row.uniq.length == 1 && row[0] != "." }
    
    # ตรวจ columns
    3.times do |j|
      col = @board.map { |row| row[j] }
      return col[0] if col.uniq.length == 1 && col[0] != "."
    end
    
    # ตรวจ diagonals
    diag1 = [0, 1, 2].map { |i| @board[i][i] }
    return diag1[0] if diag1.uniq.length == 1 && diag1[0] != "."
    
    diag2 = [0, 1, 2].map { |i| @board[i][2 - i] }
    return diag2[0] if diag2.uniq.length == 1 && diag2[0] != "."
    
    nil
  end
end

game = TicTacToe.new
game.place(0, 0, "X")
game.place(1, 1, "X")
game.place(2, 2, "X")
game.display
puts "Winner: #{game.winner?}"
```

---

## Step 73: Array as Stack and Queue

### Array เป็น Stack และ Queue

```ruby
# Stack (LIFO - Last In First Out)
stack = []

# push
stack.push(1)
stack.push(2)
stack.push(3)
puts "Stack: #{stack.inspect}"  # => [1, 2, 3]

# pop (ดึงจากบน/ท้าย)
puts stack.pop  # => 3
puts stack.pop  # => 2
puts "Stack: #{stack.inspect}"  # => [1]

# peek (ดูบนสุดโดยไม่ลบ)
puts stack.last  # => 1
```

```ruby
# Queue (FIFO - First In First Out)
queue = []

# enqueue (เพิ่มท้าย)
queue.push("task 1")
queue.push("task 2")
queue.push("task 3")
puts "Queue: #{queue.inspect}"

# dequeue (ดึงจากต้น)
puts queue.shift  # => task 1
puts queue.shift  # => task 2
puts "Queue: #{queue.inspect}"  # => ["task 3"]
```

```ruby
# ใช้งานจริง: Undo/Redo ด้วย Stack
class TextEditor
  def initialize
    @text = ""
    @history = []
    @redo_stack = []
  end
  
  def type(chars)
    save_state
    @text += chars
    @redo_stack.clear
  end
  
  def delete(n = 1)
    save_state
    @text = @text[0...-n]
    @redo_stack.clear
  end
  
  def undo
    return if @history.empty?
    @redo_stack.push(@text)
    @text = @history.pop
  end
  
  def redo
    return if @redo_stack.empty?
    @history.push(@text)
    @text = @redo_stack.pop
  end
  
  def content
    @text
  end
  
  private
  
  def save_state
    @history.push(@text.dup)
  end
end

editor = TextEditor.new
editor.type("Hello")
editor.type(", World")
editor.type("!")
puts editor.content   # => Hello, World!

editor.undo
puts editor.content   # => Hello, World

editor.undo
puts editor.content   # => Hello

editor.redo
puts editor.content   # => Hello, World
```

---

## Step 74: Array Destructuring and Splat

### การ Destructure และ Splat

```ruby
# Multiple assignment (destructuring)
a, b, c = [1, 2, 3]
puts a  # => 1
puts b  # => 2
puts c  # => 3

# ไม่ตรงจำนวน
x, y = [1, 2, 3, 4]
puts x  # => 1
puts y  # => 2 (ที่เหลือถูกละทิ้ง)

a, b = [1]
puts a  # => 1
puts b  # => nil
```

```ruby
# Splat operator (*)
first, *rest = [1, 2, 3, 4, 5]
puts first       # => 1
puts rest.inspect # => [2, 3, 4, 5]

*init, last = [1, 2, 3, 4, 5]
puts init.inspect  # => [1, 2, 3, 4]
puts last          # => 5

head, *middle, tail = [1, 2, 3, 4, 5]
puts head            # => 1
puts middle.inspect  # => [2, 3, 4]
puts tail            # => 5
```

```ruby
# การใช้ splat ใน method calls
def sum(*numbers)
  numbers.sum
end

puts sum(1, 2, 3)        # => 6
puts sum(*[4, 5, 6])     # => 15
puts sum(1, *[2, 3], 4)  # => 10

# แปลง array เป็น arguments
args = [1, 2, 3]
puts [args.min, args.max]  # => [1, 3]
# หรือ
min, max = args.minmax
puts "#{min} to #{max}"   # => 1 to 3
```

```ruby
# Nested destructuring
((a, b), c) = [[1, 2], 3]
puts a  # => 1
puts b  # => 2
puts c  # => 3

# ใช้กับ each
pairs = [[1, "one"], [2, "two"], [3, "three"]]
pairs.each do |(num, word)|
  puts "#{num} = #{word}"
end
```

---

## Step 75: Lazy Enumerator

### Lazy Evaluation สำหรับ Array ขนาดใหญ่

```ruby
# Lazy evaluation - คำนวณเมื่อต้องการเท่านั้น
# ปัญหา: array ใหญ่มาก
result = (1..Float::INFINITY)
  .lazy
  .select { |n| n.odd? }
  .map { |n| n ** 2 }
  .first(10)

puts result.inspect
# => [1, 9, 25, 49, 81, 121, 169, 225, 289, 361]
# ไม่ต้องสร้าง array ทั้งหมดก่อน!
```

```ruby
# เปรียบเทียบ eager vs lazy
require 'benchmark'

# Eager: สร้าง array ทั้งหมดก่อน
eager_time = Benchmark.realtime do
  (1..1_000_000).select { |n| n.odd? }.map { |n| n * 2 }.first(5)
end

# Lazy: คำนวณเฉพาะที่ต้องการ
lazy_time = Benchmark.realtime do
  (1..1_000_000).lazy.select { |n| n.odd? }.map { |n| n * 2 }.first(5)
end

puts "Eager: #{eager_time.round(4)}s"
puts "Lazy:  #{lazy_time.round(4)}s"
# Lazy เร็วกว่ามาก!
```

```ruby
# Lazy กับ Enumerator
fib = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y.yield a
    a, b = b, a + b
  end
end

# เอา fibonacci ที่น้อยกว่า 100
puts fib.lazy.select { |n| n < 100 }.to_a.inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]

# หา fibonacci prime 5 ตัวแรก
def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n)).none? { |i| n % i == 0 }
end

fib_primes = fib.lazy.select { |n| prime?(n) }.first(5)
puts fib_primes.inspect  # => [2, 3, 5, 13, 89]
```

---

## Step 76: Array Patterns ขั้นสูง

### รูปแบบการใช้งาน Array ขั้นสูง

```ruby
# แปลง Hash เป็น Array และกลับ
hash = { a: 1, b: 2, c: 3 }
arr = hash.to_a
puts arr.inspect  # => [[:a, 1], [:b, 2], [:c, 3]]

back_to_hash = arr.to_h
puts back_to_hash.inspect  # => {:a=>1, :b=>2, :c=>3}

# ด้วย map
arr2 = hash.map { |k, v| [k, v * 2] }
puts arr2.to_h.inspect  # => {:a=>2, :b=>4, :c=>6}
```

```ruby
# Rotate
arr = [1, 2, 3, 4, 5]
puts arr.rotate.inspect    # => [2, 3, 4, 5, 1]
puts arr.rotate(2).inspect # => [3, 4, 5, 1, 2]
puts arr.rotate(-1).inspect # => [5, 1, 2, 3, 4]

# ใช้สร้าง circular queue
class CircularBuffer
  def initialize(size)
    @buffer = Array.new(size)
    @size = size
    @head = 0
    @count = 0
  end
  
  def push(item)
    @buffer[@head] = item
    @head = (@head + 1) % @size
    @count = [@count + 1, @size].min
  end
  
  def to_a
    @buffer.rotate(@head).last(@count)
  end
end

buf = CircularBuffer.new(5)
(1..8).each { |i| buf.push(i) }
puts buf.to_a.inspect  # => [4, 5, 6, 7, 8] (เก็บ 5 ตัวล่าสุด)
```

```ruby
# cartesian product
colors = %w[red green blue]
sizes = %w[S M L XL]

# ใช้ product
combinations = colors.product(sizes)
puts combinations.length  # => 12 (3 * 4)
combinations.first(4).each { |c, s| puts "#{s} #{c} shirt" }
# => S red shirt, M red shirt, L red shirt, XL red shirt

# Nested product
puts colors.product(colors).reject { |a, b| a == b }.first(6).inspect
```

```ruby
# combinations และ permutations
numbers = [1, 2, 3, 4]

# combinations - เลือก n ตัวโดยไม่สนลำดับ
puts numbers.combination(2).to_a.inspect
# => [[1,2],[1,3],[1,4],[2,3],[2,4],[3,4]]

# permutations - เลือก n ตัวโดยสนใจลำดับ
puts numbers.permutation(2).to_a.length  # => 12 (4 * 3)
puts numbers.permutation(2).first(4).inspect
# => [[1,2],[1,3],[1,4],[2,1]]
```

---

## Step 77: Array Processing Patterns

### รูปแบบการประมวลผล

```ruby
# Pipeline pattern
def process_pipeline(data, *operations)
  operations.reduce(data) { |result, op| op.call(result) }
end

numbers = [5, 2, 8, 1, 9, 3, 7, 4, 6]

result = process_pipeline(
  numbers,
  ->(arr) { arr.select { |n| n > 3 } },
  ->(arr) { arr.sort },
  ->(arr) { arr.map { |n| n * 2 } },
  ->(arr) { arr.first(3) }
)

puts result.inspect  # => [8, 10, 12]
```

```ruby
# Sliding window maximum
def max_sliding_window(arr, k)
  arr.each_cons(k).map(&:max)
end

puts max_sliding_window([1, 3, -1, -3, 5, 3, 6, 7], 3).inspect
# => [3, 3, 5, 5, 6, 7]

# Running average
def running_average(arr, window)
  arr.each_cons(window).map { |w| w.sum.to_f / window }
end

prices = [100, 102, 98, 105, 103, 101, 99, 107]
puts running_average(prices, 3).map { |v| v.round(2) }.inspect
# => [100.0, 101.67, 102.0, 103.0, 101.0, 102.33]
```

```ruby
# Batch processing
def process_in_batches(items, batch_size, &block)
  items.each_slice(batch_size) do |batch|
    block.call(batch)
  end
end

users = (1..25).map { |i| { id: i, name: "User #{i}" } }

process_in_batches(users, 10) do |batch|
  puts "Processing batch of #{batch.length} users (IDs: #{batch.first[:id]}-#{batch.last[:id]})"
  # จริงๆ จะ save to database ที่นี่
end
```

---

## Step 78: Array เพิ่มเติม

### Methods อื่นๆ ที่มีประโยชน์

```ruby
# dig - เข้าถึง nested array อย่างปลอดภัย
nested = [[1, [2, 3]], [4, [5, 6]]]
puts nested.dig(0, 1, 0)    # => 2
puts nested.dig(1, 1, 1)    # => 6
puts nested.dig(2, 0)       # => nil (ไม่ raise error!)
# nested[2][0]              # => NoMethodError!

# flatten! แบบ in-place
arr = [1, [2, [3, [4]]]]
arr.flatten!
puts arr.inspect  # => [1, 2, 3, 4]
```

```ruby
# assoc / rassoc - หา pair ใน array of arrays
pairs = [["a", 1], ["b", 2], ["c", 3]]
puts pairs.assoc("b").inspect    # => ["b", 2] (หาด้วย key)
puts pairs.rassoc(2).inspect     # => ["b", 2] (หาด้วย value)

# flatten ใช้กับ compact
data = [1, nil, [2, nil, 3], nil, [4, [nil, 5]]]
puts data.flatten.compact.inspect  # => [1, 2, 3, 4, 5]
```

```ruby
# repeated_combination / repeated_permutation
puts [1, 2, 3].repeated_combination(2).to_a.inspect
# => [[1,1],[1,2],[1,3],[2,2],[2,3],[3,3]]

puts [1, 2].repeated_permutation(2).to_a.inspect
# => [[1,1],[1,2],[2,1],[2,2]]

# Array#sum กับ initial value
puts [1, 2, 3].sum(10)   # => 16 (10 + 6)
puts ["a", "b"].sum("")  # => "ab"
```

```ruby
# intersection, union, difference (Ruby 2.7+)
a = [1, 2, 3, 4, 5]
b = [3, 4, 5, 6, 7]
c = [5, 6, 7, 8, 9]

puts a.intersection(b).inspect      # => [3, 4, 5]
puts a.intersection(b, c).inspect   # => [5]
puts a.union(b).inspect             # => [1, 2, 3, 4, 5, 6, 7]
puts a.difference(b).inspect        # => [1, 2]
```

---

## Step 79: Array Performance Tips

### เคล็ดลับประสิทธิภาพ

```ruby
# 1. ใช้ include? แทน find สำหรับ membership test
arr = (1..1000).to_a

# ช้า
arr.find { |x| x == 500 }

# เร็วกว่า
arr.include?(500)

# เร็วที่สุด (O(1)) - ใช้ Set ถ้า check บ่อย
require 'set'
set = Set.new(arr)
set.include?(500)  # O(1)
```

```ruby
# 2. ใช้ map! / select! / reject! แทนเพื่อประหยัด memory
arr = [1, 2, 3, 4, 5, 6]

# สร้าง array ใหม่ (ใช้ memory เพิ่ม)
doubled = arr.map { |x| x * 2 }

# แก้ไข in-place (ประหยัด memory)
arr.map! { |x| x * 2 }
puts arr.inspect  # => [2, 4, 6, 8, 10, 12]
```

```ruby
# 3. ใช้ flatten(1) แทน flatten เมื่อรู้จำนวน level
# ปลอดภัยกว่าและชัดเจนกว่า
nested = [[1, 2], [3, 4], [5, 6]]
puts nested.flatten(1).inspect  # => [1, 2, 3, 4, 5, 6]

# 4. sort_by เร็วกว่า sort กับ block ซับซ้อน
users = Array.new(1000) { |i| { name: "User #{i}", age: rand(100) } }

# ช้า: คำนวณ key ทุก comparison
users.sort { |a, b| a[:name].downcase <=> b[:name].downcase }

# เร็วกว่า: คำนวณ key 1 ครั้งต่อ element
users.sort_by { |u| u[:name].downcase }
```

---

## Step 80: Enumerable Module

### เมธอดจาก Enumerable

```ruby
# Array include Enumerable module
# ทำให้ได้เมธอดเหล่านี้ฟรี:

numbers = [5, 3, 8, 1, 9, 2, 7, 4, 6]

# count กับ condition
puts numbers.count { |n| n > 5 }   # => 4

# min/max
puts numbers.min_by { |n| -n }     # => 9 (max by negate)

# flat_map
puts [[1,2],[3,4],[5,6]].flat_map { |arr| arr.map { |n| n * 2 } }.inspect
# => [2, 4, 6, 8, 10, 12]

# each_with_object
result = numbers.each_with_object([]) do |n, arr|
  arr << n * 2 if n.odd?
end
puts result.inspect  # => [10, 6, 2, 14, 8]

# zip กับ block
[1,2,3].zip([4,5,6]) { |a, b| puts "#{a} + #{b} = #{a+b}" }
```

---

## แบบฝึกหัดตอนที่ 5: Arrays (20 ข้อ)

### ข้อ 1-5: พื้นฐาน

**ข้อ 1**: สร้างฟังก์ชันที่หาค่า median ของ array ตัวเลข

```ruby
# เฉลย
def median(arr)
  return nil if arr.empty?
  sorted = arr.sort
  n = sorted.length
  
  if n.odd?
    sorted[n / 2].to_f
  else
    (sorted[n/2 - 1] + sorted[n/2]) / 2.0
  end
end

puts median([3, 1, 4, 1, 5, 9, 2, 6])  # => 3.5
puts median([1, 2, 3, 4, 5])             # => 3.0
puts median([42])                         # => 42.0
puts median([]).inspect                   # => nil
```

**ข้อ 2**: เขียนฟังก์ชัน rotate_array

```ruby
# เฉลย
def rotate_left(arr, n = 1)
  return arr if arr.empty?
  n = n % arr.length
  arr[n..] + arr[0...n]
end

def rotate_right(arr, n = 1)
  rotate_left(arr, -n % arr.length)
end

arr = [1, 2, 3, 4, 5]
puts rotate_left(arr, 2).inspect   # => [3, 4, 5, 1, 2]
puts rotate_right(arr, 2).inspect  # => [4, 5, 1, 2, 3]
puts arr.rotate(2).inspect         # => [3, 4, 5, 1, 2] (built-in)
```

**ข้อ 3**: เขียนฟังก์ชัน two_sum (หา pair ที่รวมกันได้ target)

```ruby
# เฉลย
def two_sum(nums, target)
  seen = {}
  
  nums.each_with_index do |num, i|
    complement = target - num
    if seen.key?(complement)
      return [seen[complement], i]
    end
    seen[num] = i
  end
  
  nil
end

puts two_sum([2, 7, 11, 15], 9).inspect   # => [0, 1]
puts two_sum([3, 2, 4], 6).inspect         # => [1, 2]
puts two_sum([1, 2, 3, 4], 10).inspect     # => nil
```

**ข้อ 4**: เขียนฟังก์ชันหาค่าที่ซ้ำใน array

```ruby
# เฉลย
def find_duplicates(arr)
  freq = Hash.new(0)
  arr.each { |item| freq[item] += 1 }
  freq.select { |_, count| count > 1 }.keys
end

def find_duplicates_fast(arr)
  seen = Set.new
  duplicates = Set.new
  arr.each { |item| seen.add?(item) ? nil : duplicates.add(item) }
  duplicates.to_a
rescue NameError
  # Fallback without Set
  arr.group_by(&:itself).select { |_, v| v.length > 1 }.keys
end

puts find_duplicates([1, 2, 3, 2, 4, 3, 5, 1]).inspect
# => [1, 2, 3]
```

**ข้อ 5**: เขียนฟังก์ชัน chunk array เป็นกลุ่มๆ

```ruby
# เฉลย
def chunk_array(arr, size)
  arr.each_slice(size).to_a
end

def chunk_by_predicate(arr, &pred)
  arr.chunk { |x| pred.call(x) }.map { |_, group| group }
end

nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
puts chunk_array(nums, 3).inspect
# => [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

# chunk by odd/even
chunks = nums.chunk(&:odd?).map { |odd, group| { odd: odd, values: group } }
chunks.each { |c| puts c.inspect }
```

### ข้อ 6-10: Intermediate

**ข้อ 6**: เขียน flatten_hash ที่แปลง nested hash เป็น flat key-value pairs

```ruby
# เฉลย
def flatten_hash(hash, prefix = "", separator = ".")
  hash.each_with_object({}) do |(key, value), result|
    full_key = prefix.empty? ? key.to_s : "#{prefix}#{separator}#{key}"
    
    if value.is_a?(Hash)
      result.merge!(flatten_hash(value, full_key, separator))
    else
      result[full_key] = value
    end
  end
end

config = {
  database: {
    host: "localhost",
    port: 5432,
    credentials: {
      user: "admin",
      password: "secret"
    }
  },
  app: {
    name: "MyApp",
    debug: false
  }
}

flattened = flatten_hash(config)
flattened.each { |k, v| puts "#{k}: #{v}" }
```

**ข้อ 7**: เขียนฟังก์ชัน group_consecutive

```ruby
# เฉลย
def group_consecutive(arr)
  return [] if arr.empty?
  
  arr.sort.each_with_object([[]]) do |num, groups|
    if groups.last.empty? || num == groups.last.last + 1
      groups.last << num
    else
      groups << [num]
    end
  end
end

nums = [1, 2, 3, 5, 6, 10, 11, 12, 20]
groups = group_consecutive(nums)
groups.each do |group|
  if group.length == 1
    puts group.first.to_s
  else
    puts "#{group.first}-#{group.last}"
  end
end
# => 1-3
# => 5-6
# => 10-12
# => 20
```

**ข้อ 8**: Matrix transpose โดยไม่ใช้ built-in

```ruby
# เฉลย
def transpose_matrix(matrix)
  return [] if matrix.empty?
  
  rows = matrix.length
  cols = matrix[0].length
  
  Array.new(cols) do |j|
    Array.new(rows) { |i| matrix[i][j] }
  end
end

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

m = [[1, 2, 3], [4, 5, 6]]
puts "Original:"
m.each { |row| puts row.inspect }

puts "\nTransposed:"
transpose_matrix(m).each { |row| puts row.inspect }

a = [[1, 2], [3, 4]]
b = [[5, 6], [7, 8]]
puts "\nMatrix multiplication:"
matrix_multiply(a, b).each { |row| puts row.inspect }
```

**ข้อ 9**: เขียน merge_sort algorithm

```ruby
# เฉลย
def merge_sort(arr)
  return arr if arr.length <= 1
  
  mid = arr.length / 2
  left = merge_sort(arr[0...mid])
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

unsorted = [38, 27, 43, 3, 9, 82, 10]
puts "ก่อนเรียง: #{unsorted.inspect}"
puts "หลังเรียง: #{merge_sort(unsorted).inspect}"

# ทดสอบ performance
require 'benchmark'

big_arr = (1..10000).to_a.shuffle

Benchmark.bm(15) do |bm|
  bm.report("merge_sort:") { merge_sort(big_arr) }
  bm.report("ruby sort:")  { big_arr.sort }
end
```

**ข้อ 10**: เขียนฟังก์ชัน deep_flatten ที่รองรับ array ลึกมาก

```ruby
# เฉลย
def deep_flatten(arr)
  arr.each_with_object([]) do |elem, result|
    if elem.is_a?(Array)
      result.concat(deep_flatten(elem))
    else
      result << elem
    end
  end
end

# แบบ iterative (ไม่ใช้ recursion, ดีกว่าสำหรับ deeply nested)
def deep_flatten_iterative(arr)
  stack = arr.dup
  result = []
  
  until stack.empty?
    item = stack.shift
    if item.is_a?(Array)
      stack.unshift(*item)  # แทรกกลับเข้าไปที่ต้น
    else
      result << item
    end
  end
  
  result
end

deeply_nested = [1, [2, [3, [4, [5, [6, [7]]]]]]]
puts deep_flatten(deeply_nested).inspect
# => [1, 2, 3, 4, 5, 6, 7]

puts deep_flatten_iterative([1, [2, [3]], [4, [5, [6]]]]).inspect
# => [1, 2, 3, 4, 5, 6]
```

### ข้อ 11-15: Intermediate-Advanced

**ข้อ 11**: เขียน quicksort

```ruby
# เฉลย
def quicksort(arr)
  return arr if arr.length <= 1
  
  pivot = arr[arr.length / 2]
  left = arr.select { |x| x < pivot }
  middle = arr.select { |x| x == pivot }
  right = arr.select { |x| x > pivot }
  
  quicksort(left) + middle + quicksort(right)
end

# In-place quicksort
def quicksort_inplace(arr, low = 0, high = arr.length - 1)
  if low < high
    pivot_idx = partition(arr, low, high)
    quicksort_inplace(arr, low, pivot_idx - 1)
    quicksort_inplace(arr, pivot_idx + 1, high)
  end
  arr
end

def partition(arr, low, high)
  pivot = arr[high]
  i = low - 1
  
  (low...high).each do |j|
    if arr[j] <= pivot
      i += 1
      arr[i], arr[j] = arr[j], arr[i]
    end
  end
  
  arr[i + 1], arr[high] = arr[high], arr[i + 1]
  i + 1
end

puts quicksort([3, 6, 8, 10, 1, 2, 1]).inspect
# => [1, 1, 2, 3, 6, 8, 10]
```

**ข้อ 12**: Implement stack-based calculator

```ruby
# เฉลย
def evaluate_rpn(expression)
  # Reverse Polish Notation: "3 4 + 2 * 7 /" => ((3+4)*2)/7
  stack = []
  
  expression.split.each do |token|
    case token
    when /\A-?\d+\.?\d*\z/  # number
      stack.push(token.include?(".") ? token.to_f : token.to_i)
    when "+", "-", "*", "/"
      b, a = stack.pop, stack.pop
      result = case token
               when "+" then a + b
               when "-" then a - b
               when "*" then a * b
               when "/" then a.to_f / b
               end
      stack.push(result)
    else
      raise ArgumentError, "Unknown token: #{token}"
    end
  end
  
  stack.last
end

expressions = [
  "3 4 +",       # 7
  "5 1 2 + 4 * + 3 -",  # 14
  "2 3 * 4 +",   # 10
  "10 2 /"        # 5.0
]

expressions.each do |expr|
  puts "#{expr} = #{evaluate_rpn(expr)}"
end
```

**ข้อ 13**: เขียน histogram generator

```ruby
# เฉลย
def histogram(data, bins = 10)
  min, max = data.min, data.max
  range = max - min
  bin_width = range.to_f / bins
  
  # สร้าง bins
  counts = Array.new(bins, 0)
  data.each do |value|
    bin_idx = ((value - min) / bin_width).to_i
    bin_idx = bins - 1 if bin_idx >= bins  # Edge case for max value
    counts[bin_idx] += 1
  end
  
  # แสดงผล
  max_count = counts.max
  bar_width = 40
  
  puts "Histogram (#{data.length} data points)"
  puts "-" * 60
  
  bins.times do |i|
    low = min + i * bin_width
    high = low + bin_width
    bar = "#" * (counts[i].to_f / max_count * bar_width).round
    puts "[%6.2f - %6.2f] | %-40s| %d" % [low, high, bar, counts[i]]
  end
end

# Generate sample data
require 'rng' if false  # ถ้ามี
data = 500.times.map { |_| (rand * 2 - 1 + rand * 2 - 1 + rand * 2 - 1) * 20 + 50 }
histogram(data.map { |x| x.round(1) }, 15)
```

**ข้อ 14**: เขียน LRU Cache ด้วย Array และ Hash

```ruby
# เฉลย
class LRUCache
  def initialize(capacity)
    @capacity = capacity
    @cache = {}
    @order = []  # LRU order: newest at end
  end
  
  def get(key)
    if @cache.key?(key)
      @order.delete(key)
      @order.push(key)
      @cache[key]
    else
      -1
    end
  end
  
  def put(key, value)
    if @cache.key?(key)
      @order.delete(key)
    elsif @cache.length >= @capacity
      oldest = @order.shift  # Remove LRU
      @cache.delete(oldest)
    end
    
    @cache[key] = value
    @order.push(key)
  end
  
  def to_s
    @order.map { |k| "#{k}:#{@cache[k]}" }.join(", ")
  end
end

cache = LRUCache.new(3)
cache.put(1, "one")
cache.put(2, "two")
cache.put(3, "three")
puts cache  # => 1:one, 2:two, 3:three

cache.get(1)  # Access 1 (moves to most recent)
cache.put(4, "four")  # Evict LRU (2)
puts cache  # => 3:three, 1:one, 4:four

puts cache.get(2)  # => -1 (evicted)
puts cache.get(1)  # => one
```

**ข้อ 15**: Implement Sudoku validator

```ruby
# เฉลย
def valid_sudoku?(board)
  # ตรวจ rows
  board.each do |row|
    return false unless valid_unit?(row)
  end
  
  # ตรวจ columns
  9.times do |col|
    column = board.map { |row| row[col] }
    return false unless valid_unit?(column)
  end
  
  # ตรวจ 3x3 boxes
  [0, 3, 6].each do |box_row|
    [0, 3, 6].each do |box_col|
      box = []
      (box_row...box_row+3).each do |r|
        (box_col...box_col+3).each do |c|
          box << board[r][c]
        end
      end
      return false unless valid_unit?(box)
    end
  end
  
  true
end

def valid_unit?(unit)
  numbers = unit.reject { |x| x == 0 }
  numbers.length == numbers.uniq.length &&
    numbers.all? { |n| (1..9).include?(n) }
end

valid_board = [
  [5,3,0, 0,7,0, 0,0,0],
  [6,0,0, 1,9,5, 0,0,0],
  [0,9,8, 0,0,0, 0,6,0],
  [8,0,0, 0,6,0, 0,0,3],
  [4,0,0, 8,0,3, 0,0,1],
  [7,0,0, 0,2,0, 0,0,6],
  [0,6,0, 0,0,0, 2,8,0],
  [0,0,0, 4,1,9, 0,0,5],
  [0,0,0, 0,8,0, 0,7,9]
]

puts valid_sudoku?(valid_board) ? "Valid Sudoku!" : "Invalid!"

# Invalid board (duplicate in row 1)
invalid_board = valid_board.map(&:dup)
invalid_board[0][1] = 5  # Duplicate 5 in row 0
puts valid_sudoku?(invalid_board) ? "Valid Sudoku!" : "Invalid!"
```

### ข้อ 16-20: Advanced

**ข้อ 16**: เขียน graph BFS/DFS ด้วย Array

```ruby
# เฉลย
def bfs(graph, start)
  visited = []
  queue = [start]
  
  until queue.empty?
    node = queue.shift
    next if visited.include?(node)
    
    visited << node
    (graph[node] || []).each { |neighbor| queue << neighbor }
  end
  
  visited
end

def dfs(graph, start, visited = [])
  visited << start
  
  (graph[start] || []).each do |neighbor|
    dfs(graph, neighbor, visited) unless visited.include?(neighbor)
  end
  
  visited
end

graph = {
  "A" => ["B", "C"],
  "B" => ["A", "D", "E"],
  "C" => ["A", "F"],
  "D" => ["B"],
  "E" => ["B", "F"],
  "F" => ["C", "E"]
}

puts "BFS from A: #{bfs(graph, 'A').join(' -> ')}"
puts "DFS from A: #{dfs(graph, 'A').join(' -> ')}"
```

**ข้อ 17**: เขียน array-based heap sort

```ruby
# เฉลย
def heap_sort(arr)
  n = arr.length
  arr = arr.dup
  
  # Build max heap
  (n/2 - 1).downto(0) { |i| heapify(arr, n, i) }
  
  # Extract elements
  (n-1).downto(1) do |i|
    arr[0], arr[i] = arr[i], arr[0]
    heapify(arr, i, 0)
  end
  
  arr
end

def heapify(arr, n, i)
  largest = i
  left = 2 * i + 1
  right = 2 * i + 2
  
  largest = left if left < n && arr[left] > arr[largest]
  largest = right if right < n && arr[right] > arr[largest]
  
  if largest != i
    arr[i], arr[largest] = arr[largest], arr[i]
    heapify(arr, n, largest)
  end
end

unsorted = [12, 11, 13, 5, 6, 7]
puts "Before: #{unsorted.inspect}"
puts "After:  #{heap_sort(unsorted).inspect}"
```

**ข้อ 18**: เขียนฟังก์ชัน array_diff ที่บอกความแตกต่างระหว่าง 2 arrays

```ruby
# เฉลย
def array_diff(original, modified)
  {
    added: modified - original,
    removed: original - modified,
    common: original & modified,
    unchanged: original.select { |x| modified.include?(x) },
    changed_count: (original | modified).length - (original & modified).length
  }
end

# Myers diff algorithm (simplified)
def compute_diff(old_arr, new_arr)
  diff = []
  old_idx = new_idx = 0
  
  while old_idx < old_arr.length || new_idx < new_arr.length
    old_item = old_arr[old_idx]
    new_item = new_arr[new_idx]
    
    if old_item == new_item
      diff << { op: :equal, value: old_item }
      old_idx += 1
      new_idx += 1
    elsif old_idx >= old_arr.length
      diff << { op: :insert, value: new_item }
      new_idx += 1
    elsif new_idx >= new_arr.length
      diff << { op: :delete, value: old_item }
      old_idx += 1
    elsif new_arr.include?(old_item)
      diff << { op: :insert, value: new_item }
      new_idx += 1
    else
      diff << { op: :delete, value: old_item }
      old_idx += 1
    end
  end
  
  diff
end

old = %w[apple banana cherry date]
new_arr = %w[apple cherry elderberry date fig]

diff = array_diff(old, new_arr)
puts "Added: #{diff[:added].inspect}"
puts "Removed: #{diff[:removed].inspect}"
puts "Common: #{diff[:common].inspect}"
```

**ข้อ 19**: เขียน polynomial arithmetic ด้วย Array

```ruby
# เฉลย
# Polynomial represented as array of coefficients [a0, a1, a2, ...] = a0 + a1*x + a2*x^2 ...
class Polynomial
  attr_reader :coefficients
  
  def initialize(coeffs)
    @coefficients = coeffs.dup
    @coefficients.pop while @coefficients.last == 0 && @coefficients.length > 1
  end
  
  def +(other)
    max_len = [coefficients.length, other.coefficients.length].max
    result = max_len.times.map do |i|
      (coefficients[i] || 0) + (other.coefficients[i] || 0)
    end
    Polynomial.new(result)
  end
  
  def -(other)
    max_len = [coefficients.length, other.coefficients.length].max
    result = max_len.times.map do |i|
      (coefficients[i] || 0) - (other.coefficients[i] || 0)
    end
    Polynomial.new(result)
  end
  
  def *(other)
    result = Array.new(coefficients.length + other.coefficients.length - 1, 0)
    coefficients.each_with_index do |c1, i|
      other.coefficients.each_with_index do |c2, j|
        result[i + j] += c1 * c2
      end
    end
    Polynomial.new(result)
  end
  
  def evaluate(x)
    coefficients.each_with_index.reduce(0) do |sum, (c, i)|
      sum + c * (x ** i)
    end
  end
  
  def degree
    coefficients.length - 1
  end
  
  def to_s
    terms = coefficients.each_with_index.reverse_each.map do |c, i|
      next if c == 0
      case i
      when 0 then c.to_s
      when 1 then c == 1 ? "x" : "#{c}x"
      else c == 1 ? "x^#{i}" : "#{c}x^#{i}"
      end
    end.compact
    terms.empty? ? "0" : terms.join(" + ").gsub("+ -", "- ")
  end
end

p1 = Polynomial.new([1, 2, 1])   # 1 + 2x + x^2
p2 = Polynomial.new([1, -1])      # 1 - x

puts "p1 = #{p1}"
puts "p2 = #{p2}"
puts "p1 + p2 = #{p1 + p2}"
puts "p1 * p2 = #{p1 * p2}"
puts "p1(3) = #{p1.evaluate(3)}"  # 1 + 6 + 9 = 16
```

**ข้อ 20**: เขียน moving average และ Bollinger Bands

```ruby
# เฉลย
def simple_moving_average(prices, period)
  prices.each_cons(period).map { |window| window.sum.to_f / period }
end

def standard_deviation(arr)
  mean = arr.sum.to_f / arr.length
  variance = arr.sum { |x| (x - mean) ** 2 } / arr.length
  Math.sqrt(variance)
end

def bollinger_bands(prices, period = 20, num_std = 2)
  sma = simple_moving_average(prices, period)
  
  bands = prices.each_cons(period).map.with_index do |window, i|
    std = standard_deviation(window)
    {
      middle: sma[i],
      upper: sma[i] + num_std * std,
      lower: sma[i] - num_std * std,
      bandwidth: 4 * num_std * std / sma[i] * 100
    }
  end
  
  bands
end

# Generate fake stock prices
prices = [100.0]
50.times { prices << (prices.last * (1 + (rand * 0.04 - 0.02))).round(2) }

puts "=== Bollinger Bands (last 5 values) ==="
bands = bollinger_bands(prices)
bands.last(5).each_with_index do |band, i|
  puts "Upper: #{band[:upper].round(2)}, Middle: #{band[:middle].round(2)}, Lower: #{band[:lower].round(2)}, BW: #{band[:bandwidth].round(2)}%"
end
```

---

## สรุป

| เรื่อง | เมธอดสำคัญ |
|--------|------------|
| สร้าง | [], Array.new, %w[], %i[], Array() |
| เข้าถึง | [], at, fetch, first, last, sample, take, drop |
| แก้ไข | push/<<, pop, shift, unshift, insert, delete |
| ตรวจสอบ | length, empty?, include?, any?, all?, none? |
| แปลง | map, select/filter, reject, flat_map |
| รวมค่า | reduce/inject, sum, each_with_object |
| เรียง | sort, sort_by, min, max, shuffle |
| จัดกลุ่ม | group_by, tally, partition, chunk, each_slice |
| ค้นหา | find, find_index, bsearch, grep |
| รวม | +, -, &, |, zip, flatten, compact, uniq |
| Lazy | .lazy สำหรับ large datasets |
| Destructuring | Splat (*) และ multiple assignment |

ตอนต่อไปจะเป็นเรื่อง **Hashes** ที่เป็น key-value collection ที่ทรงพลังมากใน Ruby
