# ตอนที่ 18: Enumerables (Steps 366-390)

## บทนำ

**Enumerable** เป็น module ที่ทรงพลังที่สุดตัวหนึ่งใน Ruby ซึ่งมี methods สำหรับการทำงานกับ collections (เช่น Array, Hash) มากกว่า 50 methods มันเป็นหัวใจของ functional programming style ใน Ruby

---

## Step 366: Enumerable Module คืออะไร?

```ruby
# Enumerable module ถูก include ใน Array และ Hash โดยอัตโนมัติ
puts Array.include?(Enumerable)  # true
puts Hash.include?(Enumerable)   # true
puts Range.include?(Enumerable)  # true

# ดู methods ทั้งหมด
puts Enumerable.instance_methods.sort.inspect
# [:all?, :any?, :chunk, :chunk_while, :collect, :collect_concat,
#  :count, :cycle, :detect, :drop, :drop_while, :each_cons,
#  :each_entry, :each_slice, :each_with_index, :each_with_object,
#  :entries, :filter, :filter_map, :find, :find_all, :find_index,
#  :first, :flat_map, :grep, :grep_v, :group_by, :include?,
#  :inject, :lazy, :map, :max, :max_by, :min, :min_by, :minmax,
#  :minmax_by, :none?, :one?, :reduce, :reject, :reverse_each,
#  :select, :sort, :sort_by, :sum, :take, :take_while, :tally,
#  :to_a, :to_h, :uniq, :zip]

# Enumerable ต้องการแค่ method each
class SimpleCollection
  include Enumerable

  def initialize(*items)
    @items = items
  end

  def each(&block)
    @items.each(&block)
  end
end

col = SimpleCollection.new(3, 1, 4, 1, 5, 9, 2, 6)
puts col.sort.inspect     # [1, 1, 2, 3, 4, 5, 6, 9]
puts col.min              # 1
puts col.max              # 9
puts col.select(&:odd?).inspect  # [3, 1, 1, 5, 9]
```

---

## Step 367: map/collect - Transform Each Element

`map` (หรือ `collect`) แปลง collection โดยประมวลผลแต่ละ element:

```ruby
# map พื้นฐาน
numbers = [1, 2, 3, 4, 5]
squares = numbers.map { |n| n ** 2 }
puts squares.inspect  # [1, 4, 9, 16, 25]

# map กับ String
words = ["hello", "world", "ruby"]
upcased = words.map(&:upcase)
puts upcased.inspect  # ["HELLO", "WORLD", "RUBY"]

# map กับ Hash
prices = { apple: 10, banana: 5, cherry: 25 }
discounted = prices.map { |item, price| [item, price * 0.9] }.to_h
puts discounted.inspect
# {:apple=>9.0, :banana=>4.5, :cherry=>22.5}

# map กับ index (each_with_index)
items = ["a", "b", "c", "d"]
numbered = items.each_with_index.map { |item, idx| "#{idx + 1}. #{item}" }
puts numbered.inspect
# ["1. a", "2. b", "3. c", "4. d"]

# map! แก้ไข in-place
numbers = [1, 2, 3, 4, 5]
numbers.map! { |n| n * 10 }
puts numbers.inspect  # [10, 20, 30, 40, 50]

# ตัวอย่างจริง: แปลงข้อมูล API
users_json = [
  { "id" => 1, "first_name" => "John", "last_name" => "Doe", "email" => "john@example.com" },
  { "id" => 2, "first_name" => "Jane", "last_name" => "Smith", "email" => "jane@example.com" }
]

users = users_json.map do |u|
  {
    id:   u["id"],
    name: "#{u["first_name"]} #{u["last_name"]}",
    email: u["email"]
  }
end
users.each { |u| puts "#{u[:id]}: #{u[:name]} (#{u[:email]})" }
```

---

## Step 368: select/filter - Keep Matching Elements

`select` (หรือ `filter`) เก็บเฉพาะ elements ที่ตรงกับเงื่อนไข:

```ruby
# select พื้นฐาน
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
evens = numbers.select(&:even?)
odds = numbers.select(&:odd?)
puts evens.inspect  # [2, 4, 6, 8, 10]
puts odds.inspect   # [1, 3, 5, 7, 9]

# select กับ String
words = ["apple", "banana", "cherry", "date", "elderberry", "fig"]
long_words = words.select { |w| w.length > 5 }
puts long_words.inspect  # ["banana", "cherry", "elderberry"]

short_words = words.filter { |w| w.length <= 4 }
puts short_words.inspect  # ["date", "fig"]

# select กับ Hash
inventory = {
  apple:  { count: 50, price: 10 },
  banana: { count: 0,  price: 5 },
  cherry: { count: 20, price: 25 },
  date:   { count: 0,  price: 15 }
}

available = inventory.select { |_, info| info[:count] > 0 }
puts available.keys.inspect  # [:apple, :cherry]

# filter_map - select + map ในขั้นตอนเดียว (Ruby 2.7+)
numbers = [1, 2, 3, 4, 5, 6]
even_squares = numbers.filter_map { |n| n ** 2 if n.even? }
puts even_squares.inspect  # [4, 16, 36]

# เทียบกับการเขียนแบบเก่า
old_way = numbers.select(&:even?).map { |n| n ** 2 }
puts old_way.inspect  # [4, 16, 36]
```

---

## Step 369: reject - Remove Matching Elements

`reject` เป็นตรงข้ามของ `select`:

```ruby
# reject พื้นฐาน
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
non_evens = numbers.reject(&:even?)
puts non_evens.inspect  # [1, 3, 5, 7, 9]

# reject กับ nil/empty
mixed = [1, nil, 2, "", 3, false, 4, 0]
compact = mixed.reject(&:nil?)
puts compact.inspect  # [1, 2, "", 3, false, 4, 0]

# หยุด nil และ false
truthy = mixed.reject { |x| !x }
puts truthy.inspect  # [1, 2, "", 3, 4, 0]

# compact เป็น alias สำหรับ reject nil
arr = [1, nil, 2, nil, 3]
puts arr.compact.inspect  # [1, 2, 3]

# ตัวอย่างจริง: กรองรายการ invalid
emails = ["alice@example.com", "invalid", "bob@test.org", "@broken", "charlie@company.co"]
valid_emails = emails.reject { |e| !e.match?(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i) }
puts valid_emails.inspect
# ["alice@example.com", "bob@test.org", "charlie@company.co"]

# ตัวอย่าง: กรองคำ stop words ออก
stop_words = %w[the a an is are was were be been being]
text = "the quick brown fox is a very fast animal"
words = text.split.reject { |w| stop_words.include?(w) }
puts words.join(" ")
# quick brown fox very fast animal
```

---

## Step 370: reduce/inject - Accumulate Values

`reduce` (หรือ `inject`) สะสมค่าจาก collection:

```ruby
# ผลรวม
numbers = [1, 2, 3, 4, 5]
sum = numbers.reduce(0) { |acc, n| acc + n }
puts sum  # 15

# ใช้ Symbol shorthand
sum = numbers.reduce(:+)
puts sum  # 15

product = numbers.reduce(:*)
puts product  # 120

# ใช้ initial value
sum_plus_100 = numbers.reduce(100, :+)
puts sum_plus_100  # 115

# หา maximum ด้วย reduce
max = numbers.reduce { |max, n| n > max ? n : max }
puts max  # 5

# เปรียบเทียบกับ max method
puts numbers.max  # 5

# ตัวอย่างซับซ้อน: สร้าง Hash จาก Array
pairs = [[:name, "Alice"], [:age, 30], [:city, "Bangkok"]]
hash = pairs.reduce({}) do |result, (key, value)|
  result[key] = value
  result
end
puts hash.inspect  # {:name=>"Alice", :age=>30, :city=>"Bangkok"}

# หรือใช้ each_with_object (อ่านง่ายกว่า)
hash2 = pairs.each_with_object({}) do |(key, value), result|
  result[key] = value
end
puts hash2.inspect

# ตัวอย่าง: Word count
words = "hello world hello ruby world world"
word_count = words.split.reduce(Hash.new(0)) do |counts, word|
  counts[word] += 1
  counts
end
puts word_count.sort_by { |_, v| -v }.inspect
# [["world", 3], ["hello", 2], ["ruby", 1]]

# ตัวอย่าง: Flattening nested structure
nested = [[1, 2], [3, 4], [5, 6]]
flat = nested.reduce([]) { |arr, group| arr + group }
puts flat.inspect  # [1, 2, 3, 4, 5, 6]

# Factorial ด้วย reduce
def factorial(n)
  (1..n).reduce(:*)
end

puts factorial(5)   # 120
puts factorial(10)  # 3628800

# Running total
transactions = [100, -20, 50, -30, 200]
running_total = transactions.reduce([]) do |totals, t|
  totals << (totals.empty? ? t : totals.last + t)
end
puts running_total.inspect  # [100, 80, 130, 100, 300]
```

---

## Step 371: flat_map - Map Then Flatten

```ruby
# flat_map = map + flatten(1)
numbers = [[1, 2], [3, 4], [5, 6]]
flat = numbers.flat_map { |group| group }
puts flat.inspect  # [1, 2, 3, 4, 5, 6]

# เทียบกับ map + flatten
same = numbers.map { |group| group }.flatten
puts same.inspect  # [1, 2, 3, 4, 5, 6]

# ตัวอย่างจริง: แยก words จากหลาย sentences
sentences = [
  "Hello World Ruby",
  "Enumerable is powerful",
  "Learn Ruby today"
]
all_words = sentences.flat_map(&:split)
puts all_words.inspect
# ["Hello", "World", "Ruby", "Enumerable", "is", "powerful", "Learn", "Ruby", "today"]

# ตัวอย่าง: Expand product variants
products = [
  { name: "T-Shirt", sizes: ["S", "M", "L", "XL"] },
  { name: "Hoodie",  sizes: ["M", "L", "XL", "XXL"] }
]

variants = products.flat_map do |product|
  product[:sizes].map { |size| "#{product[:name]} - #{size}" }
end
variants.each { |v| puts v }
# T-Shirt - S
# T-Shirt - M
# T-Shirt - L
# T-Shirt - XL
# Hoodie - M
# Hoodie - L
# Hoodie - XL
# Hoodie - XXL

# ตัวอย่าง: Friends of friends (unique)
network = {
  alice: [:bob, :charlie],
  bob:   [:alice, :diana],
  charlie: [:alice, :eve]
}

friends_of_alice = network[:alice]
friends_of_friends = friends_of_alice.flat_map { |friend| network[friend] }
                                      .uniq
                                      .reject { |f| f == :alice || friends_of_alice.include?(f) }
puts "Friends of Alice's friends: #{friends_of_friends.inspect}"
# [:diana, :eve]
```

---

## Step 372: group_by - Group Elements

```ruby
# group_by พื้นฐาน
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
grouped = numbers.group_by { |n| n.even? ? :even : :odd }
puts grouped.inspect
# {:odd=>[1, 3, 5, 7, 9], :even=>[2, 4, 6, 8, 10]}

# group_by กับ String
words = ["one", "two", "three", "four", "five", "six"]
by_length = words.group_by(&:length)
puts by_length.inspect
# {3=>["one", "two", "six"], 5=>["three"], 4=>["four", "five"]}

# ตัวอย่างจริง: จัดกลุ่มนักศึกษาตามเกรด
students = [
  { name: "Alice", score: 92 },
  { name: "Bob",   score: 78 },
  { name: "Charlie", score: 85 },
  { name: "Diana",  score: 67 },
  { name: "Eve",    score: 95 },
  { name: "Frank",  score: 55 }
]

def letter_grade(score)
  case score
  when 90..100 then "A"
  when 80...90 then "B"
  when 70...80 then "C"
  when 60...70 then "D"
  else "F"
  end
end

by_grade = students.group_by { |s| letter_grade(s[:score]) }
by_grade.each do |grade, students_list|
  names = students_list.map { |s| s[:name] }.join(", ")
  puts "Grade #{grade}: #{names}"
end

# ตัวอย่าง: Log analysis
logs = [
  { time: "09:00", level: :info,    message: "Server started" },
  { time: "09:05", level: :warning, message: "High memory" },
  { time: "09:10", level: :error,   message: "Connection failed" },
  { time: "09:15", level: :info,    message: "Request processed" },
  { time: "09:20", level: :error,   message: "Timeout" }
]

by_level = logs.group_by { |log| log[:level] }
puts "Errors: #{by_level[:error].length}"
puts "Warnings: #{by_level[:warning].length}"
puts "Info: #{by_level[:info].length}"
```

---

## Step 373: chunk - Group Consecutive Elements

```ruby
# chunk จัดกลุ่ม elements ที่ติดกันและ return ค่าเหมือนกัน
numbers = [1, 1, 2, 2, 3, 1, 1, 3, 3]
chunked = numbers.chunk { |n| n }.to_a
puts chunked.inspect
# [[1, [1, 1]], [2, [2, 2]], [3, [3]], [1, [1, 1]], [3, [3, 3]]]

# ต่างจาก group_by ตรงที่ chunk ดู "consecutive" เท่านั้น
grouped = numbers.group_by { |n| n }
puts grouped.inspect
# {1=>[1, 1, 1, 1], 2=>[2, 2], 3=>[3, 3, 3]}  <- รวมทุก occurrence

# ตัวอย่าง: Run-length encoding
def run_length_encode(str)
  str.chars.chunk { |c| c }.map { |char, chars| "#{chars.length}#{char}" }.join
end

puts run_length_encode("AAABBBCCDDDDEE")  # 3A3B2C4D2E
puts run_length_encode("AABBAAB")         # 2A2B2A1B

# chunk_while - ใช้ condition บน consecutive pairs
numbers = [1, 2, 3, 5, 6, 7, 10, 11]
consecutive_groups = numbers.chunk_while { |a, b| b == a + 1 }.to_a
puts consecutive_groups.inspect
# [[1, 2, 3], [5, 6, 7], [10, 11]]

# ตัวอย่าง: หา consecutive working days
require 'date'
dates = [
  Date.new(2024, 1, 1),
  Date.new(2024, 1, 2),
  Date.new(2024, 1, 3),
  Date.new(2024, 1, 5),  # gap
  Date.new(2024, 1, 6),
  Date.new(2024, 1, 10)  # gap
]

date_groups = dates.chunk_while { |a, b| b == a + 1 }.to_a
date_groups.each do |group|
  if group.length > 1
    puts "#{group.first} ถึง #{group.last} (#{group.length} วัน)"
  else
    puts group.first.to_s
  end
end
```

---

## Step 374: each_slice และ each_cons

```ruby
# each_slice - แบ่งเป็นกลุ่มๆ ละ N
(1..12).each_slice(3) do |group|
  puts group.inspect
end
# [1, 2, 3]
# [4, 5, 6]
# [7, 8, 9]
# [10, 11, 12]

# กลุ่มสุดท้ายอาจไม่เต็ม
(1..10).each_slice(3) do |group|
  puts group.inspect
end
# [1, 2, 3]
# [4, 5, 6]
# [7, 8, 9]
# [10]

# each_cons - sliding window ขนาด N
(1..6).each_cons(3) do |window|
  puts window.inspect
end
# [1, 2, 3]
# [2, 3, 4]
# [3, 4, 5]
# [4, 5, 6]

# ตัวอย่างจริง: Batch processing
def process_in_batches(items, batch_size)
  items.each_slice(batch_size).each_with_index do |batch, i|
    puts "Processing batch #{i + 1}: #{batch.inspect}"
    # ทำ database insert หรือ API call ที่นี่
  end
end

users_to_create = (1..15).map { |i| { id: i, name: "User #{i}" } }
process_in_batches(users_to_create, 5)

# ตัวอย่าง: Moving average (each_cons)
stock_prices = [100, 102, 98, 105, 110, 108, 115, 112, 120, 118]
moving_avg = stock_prices.each_cons(3).map { |group| group.sum.to_f / group.size }
puts moving_avg.map { |v| v.round(2) }.inspect
# [100.0, 101.67, 104.33, 107.67, 111.0, 111.67, 115.67, 116.67]
```

---

## Step 375: min_by, max_by, minmax_by

```ruby
products = [
  { name: "Apple",  price: 10, rating: 4.5 },
  { name: "Banana", price: 5,  rating: 4.0 },
  { name: "Cherry", price: 25, rating: 4.8 },
  { name: "Date",   price: 15, rating: 4.2 }
]

# min_by
cheapest = products.min_by { |p| p[:price] }
puts "Cheapest: #{cheapest[:name]} (#{cheapest[:price]})"

# max_by
priciest = products.max_by { |p| p[:price] }
puts "Priciest: #{priciest[:name]} (#{priciest[:price]})"

best_rated = products.max_by { |p| p[:rating] }
puts "Best rated: #{best_rated[:name]} (#{best_rated[:rating]})"

# minmax_by
min_max = products.minmax_by { |p| p[:price] }
puts "Price range: #{min_max.first[:name]} to #{min_max.last[:name]}"

# หลาย values
top_3_expensive = products.max_by(3) { |p| p[:price] }
puts "Top 3 expensive: #{top_3_expensive.map { |p| p[:name] }.inspect}"

# ตัวอย่าง: String operations
words = ["elephant", "cat", "rhinoceros", "fox", "hippopotamus"]
puts words.min_by(&:length)  # cat
puts words.max_by(&:length)  # hippopotamus

sorted = words.sort_by(&:length)
puts sorted.inspect
# ["cat", "fox", "elephant", "rhinoceros", "hippopotamus"]
```

---

## Step 376: sort_by

```ruby
# sort_by พื้นฐาน
numbers = [3, 1, 4, 1, 5, 9, 2, 6]
puts numbers.sort.inspect      # [1, 1, 2, 3, 4, 5, 6, 9]
puts numbers.sort_by { |n| -n }.inspect  # [9, 6, 5, 4, 3, 2, 1, 1]

# sort_by กับ String
words = ["banana", "apple", "cherry", "date"]
puts words.sort.inspect        # alphabetical: ["apple", "banana", "cherry", "date"]
puts words.sort_by(&:length).inspect  # by length: ["date", "apple", "banana", "cherry"]
puts words.sort_by { |w| [-w.length, w] }.inspect  # length desc, then alpha

# sort_by กับ Object
students = [
  { name: "Charlie", gpa: 3.5, year: 2 },
  { name: "Alice",   gpa: 3.8, year: 3 },
  { name: "Bob",     gpa: 3.5, year: 1 },
  { name: "Diana",   gpa: 3.9, year: 2 }
]

# เรียงตาม GPA มากไปน้อย
by_gpa = students.sort_by { |s| -s[:gpa] }
by_gpa.each { |s| puts "#{s[:name]}: #{s[:gpa]}" }

# Multi-key sort: GPA มากไปน้อย, ถ้า GPA เท่ากันให้เรียงตามชื่อ
multi_sort = students.sort_by { |s| [-s[:gpa], s[:name]] }
multi_sort.each { |s| puts "#{s[:name]}: #{s[:gpa]}" }

# Schwartzian transform - เร็วกว่าสำหรับ expensive operations
files = ["file10.txt", "file2.txt", "file1.txt", "file20.txt"]
natural_sorted = files.sort_by { |f| f.scan(/\d+/).map(&:to_i) }
puts natural_sorted.inspect
# ["file1.txt", "file2.txt", "file10.txt", "file20.txt"]
```

---

## Step 377: count กับ Block

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# count ไม่มี argument - นับทั้งหมด
puts numbers.count  # 10

# count กับ value - นับที่เท่ากับ value
arr = [1, 2, 2, 3, 3, 3, 4]
puts arr.count(3)  # 3

# count กับ block
puts numbers.count(&:even?)  # 5
puts numbers.count { |n| n > 5 }  # 5

# ตัวอย่างจริง
votes = [:yes, :no, :yes, :yes, :no, :abstain, :yes]
puts "Yes: #{votes.count(:yes)}"
puts "No: #{votes.count(:no)}"
puts "Abstain: #{votes.count(:abstain)}"

# นับ nil
data = [1, nil, 2, nil, 3, nil, 4]
puts "Nil count: #{data.count(&:nil?)}"
puts "Non-nil count: #{data.count { |x| !x.nil? }}"
```

---

## Step 378: tally - Count Occurrences (Ruby 2.7+)

```ruby
# tally - สร้าง Hash นับจำนวนแต่ละ element
fruits = ["apple", "banana", "apple", "cherry", "banana", "apple"]
counts = fruits.tally
puts counts.inspect
# {"apple"=>3, "banana"=>2, "cherry"=>1}

# เรียงตามความถี่
sorted_counts = counts.sort_by { |_, v| -v }
puts sorted_counts.inspect
# [["apple", 3], ["banana", 2], ["cherry", 1]]

# ก่อน Ruby 2.7 ต้องใช้ reduce หรือ each_with_object
old_way = fruits.each_with_object(Hash.new(0)) { |f, h| h[f] += 1 }
puts old_way.inspect

# tally_by (ใช้ group_by + transform_values)
words = ["hello", "world", "hi", "how", "hey"]
by_first_letter = words.group_by { |w| w[0] }.transform_values(&:tally)
puts by_first_letter.inspect

# ตัวอย่าง: Word frequency analysis
text = "the quick brown fox jumps over the lazy dog the fox"
word_freq = text.split.tally.sort_by { |_, v| -v }
word_freq.first(5).each { |word, count| puts "#{word}: #{count}" }
```

---

## Step 379: zip - Combine Arrays

```ruby
# zip รวม arrays เข้าด้วยกัน
names = ["Alice", "Bob", "Charlie"]
ages  = [30, 25, 35]
cities = ["Bangkok", "Chiang Mai", "Phuket"]

zipped = names.zip(ages, cities)
puts zipped.inspect
# [["Alice", 30, "Bangkok"], ["Bob", 25, "Chiang Mai"], ["Charlie", 35, "Phuket"]]

# แปลงเป็น Hash
people = names.zip(ages).map { |name, age| { name: name, age: age } }
puts people.inspect

# zip กับ block
names.zip(ages) do |name, age|
  puts "#{name} อายุ #{age} ปี"
end

# zip ที่มีความยาวไม่เท่ากัน - ส่วนที่ขาดเป็น nil
short = [1, 2, 3]
long  = [4, 5, 6, 7, 8]
puts short.zip(long).inspect
# [[1, 4], [2, 5], [3, 6]]  <- ตัดตาม array แรก

puts long.zip(short).inspect
# [[4, 1], [5, 2], [6, 3], [7, nil], [8, nil]]  <- nil เติม

# Unzip (ด้วย transpose)
pairs = [["a", 1], ["b", 2], ["c", 3]]
letters, numbers_array = pairs.transpose
puts letters.inspect  # ["a", "b", "c"]
puts numbers_array.inspect  # [1, 2, 3]
```

---

## Step 380: cycle - วนซ้ำแบบ Circular

```ruby
# cycle วนซ้ำ collection N รอบ
[1, 2, 3].cycle(3) { |n| print "#{n} " }
puts
# 1 2 3 1 2 3 1 2 3

# cycle โดยไม่กำหนดจำนวน (infinite loop)
# ต้องใช้ break เพื่อหยุด
count = 0
[:red, :green, :blue].cycle do |color|
  puts color
  count += 1
  break if count >= 9
end

# ตัวอย่างจริง: Round-robin assignment
servers = ["server1", "server2", "server3"]
requests = 10.times.map { |i| "request_#{i}" }

assigned = {}
server_cycle = servers.cycle
requests.each do |req|
  assigned[req] = server_cycle.next
end

assigned.each { |req, server| puts "#{req} -> #{server}" }

# ตัวอย่าง: สร้าง color pattern
colors = %w[red blue green]
pattern = []
colors.cycle do |c|
  pattern << c
  break if pattern.length >= 10
end
puts pattern.inspect
```

---

## Step 381: find/detect - ค้นหา Element แรก

```ruby
numbers = [3, 7, 1, 8, 4, 2, 6, 5]

# find หา element แรกที่ตรง
first_even = numbers.find(&:even?)
puts first_even  # 8

first_gt_5 = numbers.find { |n| n > 5 }
puts first_gt_5  # 7

# detect เป็น alias ของ find
puts numbers.detect(&:odd?)  # 3

# ถ้าหาไม่พบ return nil
puts numbers.find { |n| n > 100 }.inspect  # nil

# find กับ default (ifnone)
not_found = numbers.find(-> { "not found" }) { |n| n > 100 }
puts not_found  # "not found"

# ตัวอย่างจริง
users = [
  { id: 1, name: "Alice", admin: false },
  { id: 2, name: "Bob",   admin: true },
  { id: 3, name: "Charlie", admin: false }
]

admin = users.find { |u| u[:admin] }
puts "Admin user: #{admin[:name]}"

user_by_id = users.find { |u| u[:id] == 2 }
puts "User with id 2: #{user_by_id[:name]}"
```

---

## Step 382: find_index - ค้นหา Index

```ruby
arr = [10, 20, 30, 40, 50]

# find_index ด้วย value
puts arr.find_index(30)  # 2
puts arr.find_index(99)  # nil

# find_index ด้วย block
puts arr.find_index { |n| n > 25 }  # 2 (first match)

# ตัวอย่าง
words = ["apple", "banana", "cherry", "date"]
idx = words.find_index { |w| w.start_with?("c") }
puts "First 'c' word at index: #{idx}"  # 2

# index เป็น alias
puts arr.index(20)  # 1
puts arr.index { |n| n.even? }  # 0

# rindex - หาจากด้านหลัง
arr2 = [1, 2, 3, 2, 1]
puts arr2.find_index(2)   # 1 (first occurrence)
puts arr2.rindex(2)       # 3 (last occurrence)
```

---

## Step 383: any?, all?, none?, one?

```ruby
numbers = [1, 2, 3, 4, 5]

# any? - อย่างน้อยหนึ่งตรงเงื่อนไข
puts numbers.any?(&:even?)           # true
puts numbers.any? { |n| n > 10 }    # false

# all? - ทั้งหมดตรงเงื่อนไข
puts numbers.all? { |n| n > 0 }     # true
puts numbers.all? { |n| n > 3 }     # false

# none? - ไม่มีตัวใดตรงเงื่อนไข
puts numbers.none? { |n| n > 10 }   # true
puts numbers.none? { |n| n > 3 }    # false

# one? - มีแค่หนึ่งตัวเท่านั้นที่ตรง
puts numbers.one? { |n| n == 3 }    # true
puts numbers.one?(&:even?)          # false (มี 2 และ 4)

# ใช้กับ Hash
users = { alice: true, bob: false, charlie: true }
puts users.any? { |_, active| active }   # true
puts users.all? { |_, active| active }   # false
puts users.none? { |_, active| active }  # false

# ไม่มี block - ตรวจสอบว่ามี truthy element หรือไม่
puts [nil, false, nil].any?    # false
puts [nil, 1, nil].any?        # true
puts [1, 2, 3].all?            # true
puts [1, nil, 3].all?          # false
puts [nil, false, nil].none?   # true
puts [nil, 1, nil].none?       # false
```

---

## Step 384: first, take, take_while

```ruby
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# first
puts arr.first      # 1
puts arr.first(3).inspect  # [1, 2, 3]

# take - เหมือน first แต่ return [] ถ้า n=0
puts arr.take(4).inspect   # [1, 2, 3, 4]
puts arr.take(0).inspect   # []

# take_while - take จนกว่า condition เป็น false
ascending = arr.take_while { |n| n < 6 }
puts ascending.inspect  # [1, 2, 3, 4, 5]

# ต่างจาก select - take_while หยุดเมื่อ condition แรกที่เป็น false
mixed = [2, 4, 6, 3, 8, 10]
evens_from_start = mixed.take_while(&:even?)
puts evens_from_start.inspect  # [2, 4, 6]  <- หยุดที่ 3

all_evens = mixed.select(&:even?)
puts all_evens.inspect  # [2, 4, 6, 8, 10]  <- select ทั้งหมด

# ตัวอย่างจริง: Top N items
scores = [95, 88, 72, 91, 85, 67, 78, 90]
top_3 = scores.sort.reverse.first(3)
puts "Top 3 scores: #{top_3.inspect}"
```

---

## Step 385: drop, drop_while

```ruby
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# drop - ข้าม N elements แรก
puts arr.drop(3).inspect   # [4, 5, 6, 7, 8, 9, 10]
puts arr.drop(0).inspect   # [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# drop_while - ข้ามจนกว่า condition เป็น false
result = arr.drop_while { |n| n < 5 }
puts result.inspect  # [5, 6, 7, 8, 9, 10]

# ใช้ drop + take เพื่อ pagination
def paginate(items, page, per_page)
  items.drop((page - 1) * per_page).take(per_page)
end

items = (1..50).to_a
puts paginate(items, 1, 10).inspect  # [1..10]
puts paginate(items, 2, 10).inspect  # [11..20]
puts paginate(items, 5, 10).inspect  # [41..50]

# drop_while ใช้กับ stream ที่มี header
log_lines = [
  "# Header 1",
  "# Header 2",
  "# End of headers",
  "actual data line 1",
  "actual data line 2",
  "# This is data, not header!"
]

data_lines = log_lines.drop_while { |line| line.start_with?("#") }
puts data_lines.inspect
```

---

## Step 386: Lazy Enumerators

```ruby
# Lazy evaluation - คำนวณเมื่อต้องการเท่านั้น
# มีประโยชน์มากเมื่อ collection ใหญ่หรือ infinite

# ปกติ (eager)
result = (1..Float::INFINITY).select { |n| n.odd? }.first(5)
# ❌ infinite loop! ไม่สามารถทำได้

# ด้วย lazy
result = (1..Float::INFINITY).lazy.select { |n| n.odd? }.first(5)
puts result.inspect  # [1, 3, 5, 7, 9]

# lazy + map + select
result = (1..Float::INFINITY).lazy
                              .map { |n| n * n }
                              .select { |n| n % 3 == 0 }
                              .first(5)
puts result.inspect  # [9, 36, 81, 144, 225]

# lazy กับ finite collection
# ไม่คำนวณทั้งหมด - หยุดเมื่อได้ค่าตามต้องการ
numbers = (1..1_000_000)
first_10_squares = numbers.lazy.map { |n| n * n }.first(10)
puts first_10_squares.inspect

# force/to_a แปลงกลับเป็น Array ปกติ
lazy_enum = (1..10).lazy.select(&:even?).map { |n| n * 2 }
puts lazy_enum.class  # Enumerator::Lazy
puts lazy_enum.to_a.inspect  # [4, 8, 12, 16, 20]

# ตัวอย่าง: Fibonacci sequence
fib = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

puts fib.lazy.first(10).inspect
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

puts fib.lazy.select(&:even?).first(5).inspect
# [0, 2, 8, 34, 144]
```

---

## Step 387: each_with_object

```ruby
# each_with_object - วนซ้ำพร้อมสะสมผลลัพธ์ใน object เดิม
numbers = [1, 2, 3, 4, 5]

# สร้าง Hash
result = numbers.each_with_object({}) do |n, hash|
  hash[n] = n ** 2
end
puts result.inspect  # {1=>1, 2=>4, 3=>9, 4=>16, 5=>25}

# สร้าง Array
doubled = numbers.each_with_object([]) do |n, arr|
  arr << n * 2
end
puts doubled.inspect  # [2, 4, 6, 8, 10]

# ต่างจาก reduce
# reduce: return value ของ block เป็น accumulator ใหม่
# each_with_object: object เดิมถูก pass เข้า block เสมอ

# ตัวอย่างจริง: Grouping with transformation
students = [
  { name: "Alice", grade: "A", score: 95 },
  { name: "Bob",   grade: "B", score: 82 },
  { name: "Charlie", grade: "A", score: 91 },
  { name: "Diana",  grade: "B", score: 78 }
]

grade_scores = students.each_with_object(Hash.new { |h, k| h[k] = [] }) do |s, h|
  h[s[:grade]] << s[:score]
end
puts grade_scores.inspect
# {"A"=>[95, 91], "B"=>[82, 78]}

# คำนวณ average per grade
averages = grade_scores.transform_values { |scores| scores.sum.to_f / scores.size }
puts averages.inspect
# {"A"=>93.0, "B"=>80.0}
```

---

## Step 388: sum, tally, uniq กับ blocks

```ruby
# sum พื้นฐาน
puts [1, 2, 3, 4, 5].sum  # 15
puts [1.5, 2.5, 3.0].sum  # 7.0

# sum กับ block
puts [1, 2, 3, 4, 5].sum { |n| n * 2 }  # 30
puts ["hello", "world", "ruby"].sum(0) { |w| w.length }  # 14

# sum กับ initial value
puts [1, 2, 3].sum(100)  # 106

# ตัวอย่างจริง: คำนวณ order total
cart_items = [
  { name: "Book", price: 250, qty: 2 },
  { name: "Pen",  price: 15,  qty: 5 },
  { name: "Notebook", price: 80, qty: 3 }
]

total = cart_items.sum { |item| item[:price] * item[:qty] }
puts "Total: #{total} บาท"  # 815 บาท

# uniq กับ block
people = [
  { name: "Alice", city: "Bangkok" },
  { name: "Bob",   city: "Bangkok" },
  { name: "Charlie", city: "Chiang Mai" }
]

unique_cities = people.map { |p| p[:city] }.uniq
puts unique_cities.inspect  # ["Bangkok", "Chiang Mai"]

# เลือก unique โดย criteria
unique_by_city = people.uniq { |p| p[:city] }
puts unique_by_city.map { |p| p[:name] }.inspect  # ["Alice", "Charlie"]

# flat_map + uniq
nested = [[1, 2, 3], [2, 3, 4], [3, 4, 5]]
all_unique = nested.flat_map { |a| a }.uniq
puts all_unique.sort.inspect  # [1, 2, 3, 4, 5]
```

---

## Step 389: Custom Enumerable Class

```ruby
# สร้าง class ที่ include Enumerable
class NumberSet
  include Enumerable

  def initialize(*numbers)
    @numbers = numbers
  end

  # ต้อง implement each
  def each(&block)
    @numbers.each(&block)
  end

  def to_s
    "NumberSet(#{@numbers.join(', ')})"
  end
end

ns = NumberSet.new(5, 2, 8, 1, 9, 3, 7, 4, 6)

# ได้รับ Enumerable methods ทั้งหมดฟรี!
puts ns.sort.inspect         # [1, 2, 3, 4, 5, 6, 7, 8, 9]
puts ns.min                  # 1
puts ns.max                  # 9
puts ns.sum                  # 45
puts ns.select(&:odd?).inspect  # [5, 1, 9, 3, 7]
puts ns.map { |n| n * 2 }.inspect  # [10, 4, 16, 2, 18, 6, 14, 8, 12]
puts ns.include?(7)          # true
puts ns.count                # 9

# ตัวอย่างที่ซับซ้อนขึ้น: WordCollection
class WordCollection
  include Enumerable

  def initialize(text)
    @words = text.downcase.scan(/\w+/)
  end

  def each(&block)
    @words.each(&block)
  end

  def word_frequencies
    tally.sort_by { |_, v| -v }
  end

  def unique_words
    uniq
  end

  def average_word_length
    sum(&:length).to_f / count
  end
end

text = "the quick brown fox jumps over the lazy dog the quick fox"
wc = WordCollection.new(text)

puts "Word count: #{wc.count}"
puts "Unique words: #{wc.unique_words.sort.inspect}"
puts "Average length: #{wc.average_word_length.round(2)}"
puts "Top 3 words:"
wc.word_frequencies.first(3).each { |word, freq| puts "  #{word}: #{freq}" }
puts "Words longer than 4 chars: #{wc.select { |w| w.length > 4 }.uniq.sort.inspect}"
```

---

## Step 390: ตัวอย่างขั้นสูงและ Pattern ที่พบบ่อย

```ruby
# Pattern 1: Data Pipeline
class DataPipeline
  def initialize(data)
    @data = data
  end

  def filter(&block)
    DataPipeline.new(@data.select(&block))
  end

  def transform(&block)
    DataPipeline.new(@data.map(&block))
  end

  def aggregate(&block)
    @data.reduce(&block)
  end

  def result
    @data
  end
end

orders = [
  { id: 1, amount: 150, status: :completed, category: :food },
  { id: 2, amount: 350, status: :pending,   category: :electronics },
  { id: 3, amount: 250, status: :completed, category: :food },
  { id: 4, amount: 500, status: :completed, category: :electronics },
  { id: 5, amount: 75,  status: :cancelled, category: :food }
]

# Chain operations
total_completed_food = DataPipeline.new(orders)
  .filter { |o| o[:status] == :completed }
  .filter { |o| o[:category] == :food }
  .transform { |o| o[:amount] }
  .aggregate(:+)

puts "Total completed food orders: #{total_completed_food}"  # 400

# Pattern 2: Chaining Enumerable methods
result = (1..100)
  .select { |n| n % 3 == 0 || n % 5 == 0 }
  .map { |n| n * n }
  .reject { |n| n > 500 }
  .sum

puts "Result: #{result}"

# Pattern 3: Lazy + Complex transforms
def process_log_file(lines)
  lines.lazy
       .reject { |line| line.strip.empty? }
       .reject { |line| line.start_with?("#") }
       .map(&:strip)
       .map { |line| line.split(":", 2) }
       .select { |parts| parts.length == 2 }
       .map { |key, value| [key.strip.to_sym, value.strip] }
       .to_a
       .to_h
end

config_lines = [
  "# Configuration file",
  "",
  "host: localhost",
  "port: 3000",
  "# Database settings",
  "db_host: 127.0.0.1",
  "db_port: 5432"
]

config = process_log_file(config_lines)
puts config.inspect
# {:host=>"localhost", :port=>"3000", :db_host=>"127.0.0.1", :db_port=>"5432"}
```

---

## แบบฝึกหัด: Enumerables (25 ข้อ)

### ข้อที่ 1-5: map, select, reject

**ข้อ 1:** สร้าง array ของ squared numbers สำหรับเลข 1-20 ที่เป็นจำนวนคี่เท่านั้น

```ruby
# เฉลย
result = (1..20).select(&:odd?).map { |n| n ** 2 }
puts result.inspect
# [1, 9, 25, 49, 81, 121, 169, 225, 289, 361]
```

**ข้อ 2:** จาก array ของชื่อ ["Alice", "Bob", "Charlie", "David", "Eve"] ให้ map เป็น Hash ที่มี name และ name length

```ruby
# เฉลย
names = ["Alice", "Bob", "Charlie", "David", "Eve"]
result = names.map { |name| { name: name, length: name.length } }
result.each { |r| puts "#{r[:name]}: #{r[:length]}" }
```

**ข้อ 3:** กรองเฉพาะ positive numbers และหาผลรวม

```ruby
# เฉลย
numbers = [-5, 3, -2, 8, -1, 4, -7, 9, -3, 6]
sum = numbers.select { |n| n > 0 }.sum
puts "Sum of positives: #{sum}"  # 30
```

**ข้อ 4:** ใช้ filter_map เพื่อแปลง String เป็น Integer เฉพาะที่เป็นเลขเท่านั้น

```ruby
# เฉลย
inputs = ["42", "hello", "17", "abc", "99", "3.14", "0"]
integers = inputs.filter_map { |s| Integer(s) rescue nil }
puts integers.inspect  # [42, 17, 99, 0]
```

**ข้อ 5:** สร้าง method ที่ reject nil และ empty strings

```ruby
# เฉลย
def clean_array(arr)
  arr.reject { |x| x.nil? || (x.is_a?(String) && x.strip.empty?) }
end

data = [1, nil, "hello", "", "  ", 2, nil, "world"]
puts clean_array(data).inspect  # [1, "hello", 2, "world"]
```

### ข้อที่ 6-10: reduce, group_by, sort_by

**ข้อ 6:** ใช้ reduce สร้าง word frequency count

```ruby
# เฉลย
sentence = "apple banana apple cherry banana apple"
freq = sentence.split.reduce(Hash.new(0)) { |h, w| h[w] += 1; h }
puts freq.sort_by { |_, v| -v }.inspect
```

**ข้อ 7:** group_by เพื่อจัดกลุ่มนักศึกษาตามชั้นปี และคำนวณ average GPA ต่อปี

```ruby
# เฉลย
students = [
  { name: "Alice", year: 1, gpa: 3.5 },
  { name: "Bob",   year: 2, gpa: 3.2 },
  { name: "Charlie", year: 1, gpa: 3.8 },
  { name: "Diana",  year: 2, gpa: 3.6 },
  { name: "Eve",    year: 3, gpa: 3.9 }
]

by_year = students.group_by { |s| s[:year] }
by_year.each do |year, group|
  avg = group.sum { |s| s[:gpa] } / group.size
  puts "Year #{year}: avg GPA = #{avg.round(2)}"
end
```

**ข้อ 8:** sort_by multiple criteria: ราคาน้อยไปมาก ถ้าราคาเท่ากันให้เรียงตามชื่อ A-Z

```ruby
# เฉลย
items = [
  { name: "Pen",    price: 20 },
  { name: "Apple",  price: 15 },
  { name: "Book",   price: 20 },
  { name: "Eraser", price: 10 },
  { name: "Ruler",  price: 15 }
]

sorted = items.sort_by { |i| [i[:price], i[:name]] }
sorted.each { |i| puts "#{i[:name]}: #{i[:price]}" }
```

**ข้อ 9:** ใช้ flat_map เพื่อ expand tag lists

```ruby
# เฉลย
articles = [
  { title: "Ruby basics",   tags: ["ruby", "programming", "beginner"] },
  { title: "Web with Rails", tags: ["rails", "ruby", "web"] },
  { title: "Testing guide",  tags: ["testing", "rspec", "ruby"] }
]

all_tags = articles.flat_map { |a| a[:tags] }.uniq.sort
puts all_tags.inspect
# ["beginner", "programming", "rails", "rspec", "ruby", "testing", "web"]
```

**ข้อ 10:** หา top 3 สินค้าที่ขายดีที่สุด

```ruby
# เฉลย
sales = [
  { product: "A", quantity: 150 },
  { product: "B", quantity: 320 },
  { product: "C", quantity: 89  },
  { product: "D", quantity: 445 },
  { product: "E", quantity: 201 }
]

top3 = sales.max_by(3) { |s| s[:quantity] }
top3.each_with_index do |s, i|
  puts "#{i + 1}. #{s[:product]}: #{s[:quantity]} units"
end
```

### ข้อที่ 11-15: each_slice, each_cons, chunk

**ข้อ 11:** ส่ง email เป็น batch ทีละ 10 คน

```ruby
# เฉลย
subscribers = (1..47).map { |i| "user#{i}@example.com" }

subscribers.each_slice(10).each_with_index do |batch, i|
  puts "Sending batch #{i + 1} to #{batch.length} subscribers"
  # ส่ง email จริงที่นี่
end
```

**ข้อ 12:** หา maximum sum ของ window ขนาด 3 ใน array

```ruby
# เฉลย
numbers = [4, 2, 7, 1, 8, 3, 5, 6]
max_window_sum = numbers.each_cons(3).map(&:sum).max
puts "Max window sum (size 3): #{max_window_sum}"

# หาตำแหน่งด้วย
windows = numbers.each_cons(3).to_a
max_sum = windows.map(&:sum).max
max_window = windows.find { |w| w.sum == max_sum }
puts "Window: #{max_window.inspect}"
```

**ข้อ 13:** ใช้ chunk_while เพื่อหา longest increasing sequence

```ruby
# เฉลย
numbers = [1, 2, 3, 2, 3, 4, 5, 1, 2, 3, 4, 5, 6]
sequences = numbers.chunk_while { |a, b| b > a }.to_a
longest = sequences.max_by(&:length)
puts "Longest increasing sequence: #{longest.inspect}"
puts "Length: #{longest.length}"
```

**ข้อ 14:** chunk ข้อมูล log ตาม log level

```ruby
# เฉลย
logs = [
  "INFO: Server started",
  "INFO: Loading config",
  "WARN: Memory high",
  "WARN: Disk space low",
  "ERROR: Connection failed",
  "INFO: Reconnecting",
  "INFO: Connected"
]

chunked = logs.chunk { |line| line.split(":").first }.to_a
chunked.each do |level, messages|
  puts "#{level} (#{messages.length} messages):"
  messages.each { |m| puts "  #{m}" }
end
```

**ข้อ 15:** สร้าง sliding window average สำหรับ temperature readings

```ruby
# เฉลย
temps = [20, 22, 19, 23, 25, 24, 26, 28, 27, 25, 23, 22]
window_size = 3

moving_avg = temps.each_cons(window_size).map do |window|
  (window.sum.to_f / window_size).round(1)
end

puts "Raw temperatures: #{temps.inspect}"
puts "Moving average (#{window_size}-day): #{moving_avg.inspect}"
```

### ข้อที่ 16-20: any?, all?, none?, find, zip

**ข้อ 16:** ตรวจสอบ password requirements

```ruby
# เฉลย
def validate_password(password)
  checks = {
    length:     password.length >= 8,
    uppercase:  password.match?(/[A-Z]/),
    lowercase:  password.match?(/[a-z]/),
    digit:      password.match?(/\d/),
    special:    password.match?(/[^a-zA-Z\d]/)
  }

  failed = checks.reject { |_, v| v }.keys
  if failed.empty?
    puts "Password valid!"
  else
    puts "Password invalid. Missing: #{failed.join(', ')}"
  end
end

validate_password("abc")           # too short, no upper, no digit, no special
validate_password("Password1!")    # valid
validate_password("password123")   # no uppercase, no special
```

**ข้อ 17:** ใช้ zip เพื่อสร้าง HTML table

```ruby
# เฉลย
headers = ["ชื่อ", "อายุ", "เมือง"]
rows = [
  ["Alice", 30, "Bangkok"],
  ["Bob",   25, "Chiang Mai"],
  ["Charlie", 35, "Phuket"]
]

puts "<table>"
puts "  <tr>#{headers.map { |h| "<th>#{h}</th>" }.join}</tr>"
rows.each do |row|
  cells = headers.zip(row).map { |_, val| "<td>#{val}</td>" }.join
  puts "  <tr>#{cells}</tr>"
end
puts "</table>"
```

**ข้อ 18:** ใช้ lazy enumerable สร้าง prime numbers

```ruby
# เฉลย
def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n)).none? { |i| n % i == 0 }
end

first_20_primes = (2..Float::INFINITY).lazy.select { |n| prime?(n) }.first(20)
puts first_20_primes.inspect
```

**ข้อ 19:** สร้าง DataAnalyzer class ด้วย Enumerable

```ruby
# เฉลย
class DataAnalyzer
  include Enumerable

  def initialize(data)
    @data = data
  end

  def each(&block)
    @data.each(&block)
  end

  def mean
    sum.to_f / count
  end

  def median
    sorted = sort
    mid = sorted.length / 2
    sorted.length.odd? ? sorted[mid] : (sorted[mid - 1] + sorted[mid]) / 2.0
  end

  def variance
    m = mean
    sum { |x| (x - m) ** 2 } / count
  end

  def std_dev
    Math.sqrt(variance)
  end

  def summary
    {
      count:   count,
      sum:     sum,
      min:     min,
      max:     max,
      mean:    mean.round(2),
      median:  median,
      std_dev: std_dev.round(2)
    }
  end
end

analyzer = DataAnalyzer.new([4, 7, 13, 2, 1, 9, 5, 12, 8, 6])
puts analyzer.summary.inspect
```

**ข้อ 20:** Multi-step data transformation pipeline

```ruby
# เฉลย
transactions = [
  { id: 1, user: "alice", amount: 150, type: :purchase, date: "2024-01" },
  { id: 2, user: "bob",   amount: 200, type: :purchase, date: "2024-01" },
  { id: 3, user: "alice", amount: -50, type: :refund,   date: "2024-01" },
  { id: 4, user: "charlie", amount: 300, type: :purchase, date: "2024-02" },
  { id: 5, user: "bob",   amount: 150, type: :purchase, date: "2024-02" },
  { id: 6, user: "alice", amount: 250, type: :purchase, date: "2024-02" }
]

# Monthly revenue (purchases only)
monthly_revenue = transactions
  .select { |t| t[:type] == :purchase }
  .group_by { |t| t[:date] }
  .transform_values { |ts| ts.sum { |t| t[:amount] } }
puts "Monthly revenue: #{monthly_revenue.inspect}"

# Top customers by total spend
customer_spend = transactions
  .select { |t| t[:type] == :purchase }
  .group_by { |t| t[:user] }
  .transform_values { |ts| ts.sum { |t| t[:amount] } }
  .sort_by { |_, v| -v }
puts "Top customers: #{customer_spend.inspect}"
```

### ข้อที่ 21-25: ขั้นสูง

**ข้อ 21:** Implement Enumerable ใน custom LinkedList

```ruby
# เฉลย
class LinkedList
  include Enumerable

  Node = Struct.new(:value, :next_node)

  def initialize
    @head = nil
  end

  def push(value)
    @head = Node.new(value, @head)
    self
  end

  def each
    current = @head
    while current
      yield current.value
      current = current.next_node
    end
  end
end

list = LinkedList.new
list.push(5).push(3).push(8).push(1).push(9).push(2)

puts "List: #{list.to_a.inspect}"
puts "Sorted: #{list.sort.inspect}"
puts "Sum: #{list.sum}"
puts "Min: #{list.min}, Max: #{list.max}"
puts "Evens: #{list.select(&:even?).inspect}"
```

**ข้อ 22:** สร้าง lazy pipeline สำหรับ log processing

```ruby
# เฉลย
def analyze_logs(log_stream)
  log_stream
    .lazy
    .map(&:strip)
    .reject(&:empty?)
    .map { |line| line.match(/\[(.*?)\] (.*?): (.*)/) }
    .compact
    .map { |m| { timestamp: m[1], level: m[2], message: m[3] } }
    .select { |entry| entry[:level] == "ERROR" }
    .first(10)
end

sample_logs = [
  "[2024-01-15 09:00:01] INFO: Server started",
  "[2024-01-15 09:00:05] ERROR: Database connection failed",
  "[2024-01-15 09:00:10] WARN: High memory usage",
  "[2024-01-15 09:00:15] ERROR: Timeout on request",
  "[2024-01-15 09:00:20] INFO: Request processed",
  ""
]

errors = analyze_logs(sample_logs)
errors.each { |e| puts "[#{e[:timestamp]}] #{e[:message]}" }
```

**ข้อ 23:** ใช้ tally เพื่อวิเคราะห์ voting data

```ruby
# เฉลย
votes = %w[Alice Bob Charlie Alice Alice Bob Charlie Alice Diana Bob]

vote_counts = votes.tally
total = votes.count

puts "=== Election Results ==="
vote_counts.sort_by { |_, v| -v }.each do |candidate, count|
  percentage = (count.to_f / total * 100).round(1)
  puts "#{candidate}: #{count} votes (#{percentage}%)"
end

winner = vote_counts.max_by { |_, v| v }
puts "\nWinner: #{winner[0]} with #{winner[1]} votes"
```

**ข้อ 24:** สร้าง method chain ที่ reusable ด้วย lazy enumerator

```ruby
# เฉลย
class QueryBuilder
  def initialize(data)
    @data = data.lazy
  end

  def where(&condition)
    @data = @data.select(&condition)
    self
  end

  def order_by(&key_fn)
    # ต้อง force evaluation ก่อน sort
    @data = @data.sort_by(&key_fn).lazy
    self
  end

  def limit(n)
    @data = @data.first(n).lazy
    self
  end

  def to_a
    @data.to_a
  end
end

people = (1..100).map { |i| { id: i, name: "Person #{i}", age: rand(18..70) } }

result = QueryBuilder.new(people)
  .where { |p| p[:age] >= 30 }
  .where { |p| p[:age] <= 40 }
  .order_by { |p| p[:age] }
  .limit(5)
  .to_a

result.each { |p| puts "#{p[:name]}: #{p[:age]}" }
```

**ข้อ 25:** Implement สถิติ descriptive statistics ด้วย Enumerable

```ruby
# เฉลย
module Statistics
  def self.describe(data)
    sorted = data.sort
    n = data.length

    # Mode
    freq = data.tally
    max_freq = freq.values.max
    modes = freq.select { |_, v| v == max_freq }.keys

    # Percentiles
    p25 = sorted[(n * 0.25).ceil - 1]
    p75 = sorted[(n * 0.75).ceil - 1]

    mean = data.sum.to_f / n
    variance = data.sum { |x| (x - mean) ** 2 } / n
    std_dev = Math.sqrt(variance)

    {
      count:   n,
      mean:    mean.round(3),
      std_dev: std_dev.round(3),
      min:     sorted.first,
      p25:     p25,
      median:  sorted[n / 2],
      p75:     p75,
      max:     sorted.last,
      mode:    modes
    }
  end
end

data = [4, 7, 13, 2, 1, 9, 5, 12, 8, 6, 7, 4, 7]
stats = Statistics.describe(data)
stats.each { |k, v| puts "#{k}: #{v}" }
```

---

## สรุป

Enumerable เป็น module ที่ทรงพลังมาก มี methods สำคัญ:

| Method | ใช้เพื่อ |
|--------|---------|
| `map/collect` | แปลง element แต่ละตัว |
| `select/filter` | กรองเฉพาะที่ตรงเงื่อนไข |
| `reject` | ตรงข้าม select |
| `reduce/inject` | สะสมค่าเป็น single value |
| `flat_map` | map แล้ว flatten |
| `group_by` | จัดกลุ่มตาม criteria |
| `chunk` | จัดกลุ่ม consecutive elements |
| `sort_by` | เรียงลำดับ |
| `min_by/max_by` | หา min/max ตาม criteria |
| `each_slice` | แบ่งเป็น groups ของ N |
| `each_cons` | sliding window |
| `find/detect` | หา element แรก |
| `any?/all?/none?` | ตรวจสอบเงื่อนไข |
| `tally` | นับ occurrences |
| `zip` | รวม arrays |
| `lazy` | lazy evaluation |

---

*ตอนถัดไป: ตอนที่ 19 - Comparable และ Enumerable Modules*
