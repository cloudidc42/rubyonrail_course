# Part 18: Enumerables ใน Ruby

## ขั้นตอนที่ 366-390: Enumerable Module อย่างละเอียด

---

## บทนำ

Enumerable module เป็นหนึ่งในเครื่องมือที่ทรงพลังที่สุดใน Ruby มี method มากกว่า 50 ตัวที่ทำงานกับ collection ใดๆ ก็ตามที่ implement `each` ไว้ การเข้าใจ Enumerable จะทำให้คุณเขียน Ruby code ได้อย่างสวยงามและมีประสิทธิภาพ

---

## ขั้นตอนที่ 366: Enumerable Module คืออะไร?

```ruby
# Enumerable เป็น module ที่ Array, Hash, Range, Set และ class อื่นๆ include ไว้
puts Array.ancestors.include?(Enumerable)    # => true
puts Hash.ancestors.include?(Enumerable)     # => true
puts Range.ancestors.include?(Enumerable)    # => true

# Enumerable ต้องการแค่ each method เป็นพื้นฐาน
# แล้วจะได้ method อื่นๆ อีกกว่า 50 ตัวมาฟรี!

# ดู methods ทั้งหมดของ Enumerable
puts Enumerable.instance_methods.sort.inspect

# ตัวอย่างง่ายๆ
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

puts numbers.sort.inspect            # [1, 1, 2, 3, 3, 4, 5, 5, 5, 6, 9]
puts numbers.min                     # 1
puts numbers.max                     # 9
puts numbers.sum                     # 44
puts numbers.count                   # 11
puts numbers.uniq.inspect            # [3, 1, 4, 5, 9, 2, 6]
puts numbers.select(&:odd?).inspect  # [3, 1, 1, 5, 9, 5, 3, 5]
```

---

## ขั้นตอนที่ 367: map / collect

`map` แปลงทุก element และคืนค่า Array ใหม่

```ruby
numbers = [1, 2, 3, 4, 5]

# map พื้นฐาน
squares = numbers.map { |n| n ** 2 }
puts squares.inspect   # => [1, 4, 9, 16, 25]

# map กับ string
words = ["hello", "world", "ruby"]
puts words.map(&:upcase).inspect      # => ["HELLO", "WORLD", "RUBY"]
puts words.map(&:length).inspect      # => [5, 5, 4]
puts words.map(&:reverse).inspect     # => ["olleh", "dlrow", "ybur"]

# map กับ index (each_with_index หรือ map.with_index)
words.map.with_index(1) { |word, i| "#{i}. #{word}" }.each { |w| puts w }
# => 1. hello
# => 2. world
# => 3. ruby

# map กับ Hash
users = [
  { name: "สมชาย", age: 25 },
  { name: "สมหญิง", age: 30 },
  { name: "สมศรี", age: 22 }
]

names = users.map { |u| u[:name] }
puts names.inspect   # => ["สมชาย", "สมหญิง", "สมศรี"]

# map! แก้ไข in-place
arr = [1, 2, 3]
arr.map! { |n| n * 10 }
puts arr.inspect   # => [10, 20, 30]

# ตัวอย่างจริง: แปลงข้อมูล
class Product
  attr_accessor :name, :price_thb
  
  def initialize(name, price_thb)
    @name = name
    @price_thb = price_thb
  end
  
  def to_usd(rate = 35.0)
    (@price_thb / rate).round(2)
  end
end

products = [
  Product.new("หนังสือ", 350),
  Product.new("กระเป๋า", 1500),
  Product.new("นาฬิกา", 5000)
]

usd_prices = products.map do |p|
  { name: p.name, price_usd: p.to_usd }
end

usd_prices.each { |p| puts "#{p[:name]}: $#{p[:price_usd]}" }
```

---

## ขั้นตอนที่ 368: select / reject / filter

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# select - เลือก element ที่ block คืน true
evens = numbers.select { |n| n.even? }
puts evens.inspect   # => [2, 4, 6, 8, 10]

# ใช้ shorthand
puts numbers.select(&:odd?).inspect    # => [1, 3, 5, 7, 9]

# reject - เลือก element ที่ block คืน false (ตรงข้ามกับ select)
odds = numbers.reject { |n| n.even? }
puts odds.inspect   # => [1, 3, 5, 7, 9]

# filter เป็น alias ของ select (Ruby 2.5+)
puts numbers.filter { |n| n > 5 }.inspect   # => [6, 7, 8, 9, 10]

# select กับ Hash
hash = { a: 1, b: 2, c: 3, d: 4, e: 5 }
even_pairs = hash.select { |k, v| v.even? }
puts even_pairs.inspect   # => {:b=>2, :d=>4}

# ตัวอย่างจริง
users = [
  { name: "สมชาย", age: 17, active: true },
  { name: "สมหญิง", age: 25, active: true },
  { name: "สมศรี", age: 15, active: false },
  { name: "สมบัติ", age: 30, active: true }
]

# ผู้ใช้ที่ active และอายุ >= 18
eligible = users.select { |u| u[:active] && u[:age] >= 18 }
puts eligible.map { |u| u[:name] }.inspect
# => ["สมหญิง", "สมบัติ"]

# ผู้ใช้ที่ไม่ active
inactive = users.reject { |u| u[:active] }
puts inactive.map { |u| u[:name] }.inspect
# => ["สมศรี"]

# filter_map - select + map ใน step เดียว (Ruby 2.7+)
numbers = [1, 2, 3, 4, 5, 6, 7, 8]
result = numbers.filter_map { |n| n * 2 if n.even? }
puts result.inspect   # => [4, 8, 12, 16]
```

---

## ขั้นตอนที่ 369: reduce / inject

`reduce` สะสมค่าจากทุก element

```ruby
numbers = [1, 2, 3, 4, 5]

# reduce พื้นฐาน
sum = numbers.reduce(0) { |total, n| total + n }
puts sum   # => 15

# ไม่ใส่ initial value (ใช้ element แรกเป็น accumulator)
sum = numbers.reduce { |total, n| total + n }
puts sum   # => 15

# ใช้ Symbol shorthand
puts numbers.reduce(:+)   # => 15
puts numbers.reduce(:*)   # => 120
puts numbers.reduce(:-)   # => -13 (1-2-3-4-5)

# inject เป็น alias ของ reduce
puts numbers.inject(:+)   # => 15

# ตัวอย่าง: สร้าง Hash จาก Array
keys = [:name, :age, :city]
values = ["สมชาย", 25, "กรุงเทพ"]

hash = keys.zip(values).reduce({}) do |h, (k, v)|
  h[k] = v
  h
end
puts hash.inspect   # => {:name=>"สมชาย", :age=>25, :city=>"กรุงเทพ"}

# ตัวอย่าง: factorial
def factorial(n)
  (1..n).reduce(:*)
end

puts factorial(5)    # => 120
puts factorial(10)   # => 3628800

# ตัวอย่าง: สร้าง Word frequency count
text = "the quick brown fox jumps over the lazy dog the fox"
word_count = text.split.reduce(Hash.new(0)) do |counts, word|
  counts[word] += 1
  counts
end
puts word_count.sort_by { |_, v| -v }.first(5).to_h.inspect
# => {"the"=>3, "fox"=>2, "quick"=>1, ...}

# each_with_object - คล้าย reduce แต่ object ที่ส่ง pass ไม่ต้องคืนค่า
result = [1, 2, 3, 4, 5].each_with_object([]) do |n, arr|
  arr << n * 2 if n.odd?
end
puts result.inspect   # => [2, 6, 10]
```

---

## ขั้นตอนที่ 370: flat_map / flatten

```ruby
# flat_map = map + flatten(1)
arrays = [[1, 2], [3, 4], [5, 6]]

puts arrays.map { |a| a }.inspect
# => [[1, 2], [3, 4], [5, 6]]

puts arrays.flat_map { |a| a }.inspect
# => [1, 2, 3, 4, 5, 6]

# ตัวอย่างที่ใช้บ่อย
sentences = ["Hello World", "Ruby is Great", "I Love Code"]
words = sentences.flat_map { |s| s.split }
puts words.inspect
# => ["Hello", "World", "Ruby", "is", "Great", "I", "Love", "Code"]

# flat_map กับ range
result = (1..5).flat_map { |n| [n, n * n] }
puts result.inspect
# => [1, 1, 2, 4, 3, 9, 4, 16, 5, 25]

# ตัวอย่างจริง: แต่ละ user มีหลาย orders
users_with_orders = [
  { name: "สมชาย", orders: ["order1", "order2"] },
  { name: "สมหญิง", orders: ["order3"] },
  { name: "สมศรี", orders: ["order4", "order5", "order6"] }
]

all_orders = users_with_orders.flat_map { |u| u[:orders] }
puts all_orders.inspect
# => ["order1", "order2", "order3", "order4", "order5", "order6"]

# flatten กับ depth
nested = [[1, [2, 3]], [4, [5, [6]]]]
puts nested.flatten.inspect     # => [1, 2, 3, 4, 5, 6]
puts nested.flatten(1).inspect  # => [1, [2, 3], 4, [5, [6]]]
```

---

## ขั้นตอนที่ 371: group_by

`group_by` จัดกลุ่ม elements โดยค่าที่ block คืนมา

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# group by odd/even
groups = numbers.group_by { |n| n.even? ? :even : :odd }
puts groups.inspect
# => {:odd=>[1, 3, 5, 7, 9], :even=>[2, 4, 6, 8, 10]}

# group by modulo
groups = numbers.group_by { |n| n % 3 }
puts groups.inspect
# => {1=>[1, 4, 7, 10], 2=>[2, 5, 8], 0=>[3, 6, 9]}

# group by string length
words = %w[cat elephant dog hippopotamus ox bee]
by_length = words.group_by(&:length)
puts by_length.sort.to_h.inspect

# ตัวอย่างจริง: จัดกลุ่มนักศึกษาตามเกรด
students = [
  { name: "ก", score: 95 },
  { name: "ข", score: 82 },
  { name: "ค", score: 73 },
  { name: "ง", score: 88 },
  { name: "จ", score: 65 },
  { name: "ฉ", score: 91 }
]

by_grade = students.group_by do |s|
  case s[:score]
  when 90..100 then "A"
  when 80...90 then "B"
  when 70...80 then "C"
  else "D"
  end
end

by_grade.sort.each do |grade, students|
  names = students.map { |s| s[:name] }
  puts "#{grade}: #{names.join(', ')}"
end

# group_by กับ Date
require 'date'
events = [
  { date: Date.new(2024, 1, 5), name: "Event A" },
  { date: Date.new(2024, 1, 5), name: "Event B" },
  { date: Date.new(2024, 1, 10), name: "Event C" }
]

by_date = events.group_by { |e| e[:date] }
by_date.each do |date, events|
  puts "#{date}: #{events.map { |e| e[:name] }.join(', ')}"
end
```

---

## ขั้นตอนที่ 372: chunk และ chunk_while

```ruby
# chunk - จัดกลุ่ม consecutive elements ที่มีค่าเดียวกัน
numbers = [1, 1, 2, 2, 3, 1, 1, 2]

chunks = numbers.chunk { |n| n }.to_a
puts chunks.inspect
# => [[1, [1, 1]], [2, [2, 2]], [3, [3]], [1, [1, 1]], [2, [2]]]

# chunk_while - จัดกลุ่ม elements ที่ consecutive และ predicate เป็น true
numbers = [1, 2, 3, 5, 6, 10, 11, 12]

# จัดกลุ่มตัวเลขต่อเนื่อง
groups = numbers.chunk_while { |i, j| j - i == 1 }.to_a
puts groups.inspect
# => [[1, 2, 3], [5, 6], [10, 11, 12]]

# ตัวอย่าง: แบ่ง array ตามการเพิ่มขึ้น
data = [1, 2, 3, 1, 2, 5, 6, 7, 3]
ascending_groups = data.chunk_while { |i, j| j > i }.to_a
puts ascending_groups.inspect
# => [[1, 2, 3], [1, 2, 5, 6, 7], [3]]

# slice_when - ตรงข้ามกับ chunk_while
numbers = [1, 2, 3, 5, 6, 10, 11, 12]
groups = numbers.slice_when { |i, j| j - i > 1 }.to_a
puts groups.inspect
# => [[1, 2, 3], [5, 6], [10, 11, 12]]

# chunk_while สำหรับ run-length encoding
def run_length_encode(str)
  str.chars.chunk_while { |a, b| a == b }.map do |group|
    [group.length, group.first]
  end
end

puts run_length_encode("aabbbccddddee").inspect
# => [[2, "a"], [3, "b"], [2, "c"], [4, "d"], [2, "e"]]
```

---

## ขั้นตอนที่ 373: each_slice และ each_cons

```ruby
# each_slice - แบ่งเป็น chunks ขนาด n
(1..10).each_slice(3) do |slice|
  puts slice.inspect
end
# => [1, 2, 3]
# => [4, 5, 6]
# => [7, 8, 9]
# => [10]

# each_cons - sliding window ขนาด n
(1..8).each_cons(3) do |window|
  puts window.inspect
end
# => [1, 2, 3]
# => [2, 3, 4]
# => [3, 4, 5]
# => [4, 5, 6]
# => [5, 6, 7]
# => [6, 7, 8]

# ตัวอย่างจริง: batch processing
def process_in_batches(items, batch_size: 100)
  items.each_slice(batch_size) do |batch|
    puts "Processing #{batch.size} items..."
    # simulate processing
  end
end

process_in_batches((1..350).to_a, batch_size: 100)
# => Processing 100 items...
# => Processing 100 items...
# => Processing 100 items...
# => Processing 50 items...

# ตัวอย่าง: calculate moving average
def moving_average(data, window_size)
  data.each_cons(window_size).map do |window|
    window.sum.to_f / window_size
  end
end

temperatures = [22, 24, 23, 25, 27, 26, 28, 29, 27, 26]
avg = moving_average(temperatures, 3)
puts avg.map { |t| t.round(1) }.inspect
# => [23.0, 24.0, 25.0, 26.0, 27.0, 27.7, 28.0, 27.3]

# each_slice สำหรับ progress tracking
data = (1..1000).to_a
total = data.size
data.each_slice(100).with_index(1) do |batch, i|
  progress = (i * 100.0 / (total / 100)).round
  print "\rProgress: #{[progress, 100].min}%"
end
puts
```

---

## ขั้นตอนที่ 374: min, max, min_by, max_by, minmax

```ruby
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]

# min / max
puts numbers.min    # => 1
puts numbers.max    # => 9

# min / max กับ argument (n smallest/largest)
puts numbers.min(3).inspect   # => [1, 1, 2]
puts numbers.max(3).inspect   # => [9, 6, 5]

# minmax
puts numbers.minmax.inspect   # => [1, 9]

# min_by / max_by - เปรียบเทียบด้วยค่าที่ block คืนมา
words = %w[ant elephant fox bee hippopotamus]
puts words.min_by(&:length)   # => "ant"
puts words.max_by(&:length)   # => "hippopotamus"

# min_by กับ n
puts words.min_by(2, &:length).inspect   # => ["ant", "bee"]
puts words.max_by(2, &:length).inspect   # => ["hippopotamus", "elephant"]

# minmax_by
puts words.minmax_by(&:length).inspect
# => ["ant", "hippopotamus"]

# ตัวอย่างจริง: หาสินค้าราคาถูกสุดและแพงสุด
products = [
  { name: "หนังสือ", price: 350 },
  { name: "กระเป๋า", price: 1500 },
  { name: "นาฬิกา", price: 5000 },
  { name: "ปากกา", price: 50 },
  { name: "แล็ปท็อป", price: 35000 }
]

cheapest = products.min_by { |p| p[:price] }
most_expensive = products.max_by { |p| p[:price] }

puts "ถูกสุด: #{cheapest[:name]} (฿#{cheapest[:price]})"
puts "แพงสุด: #{most_expensive[:name]} (฿#{most_expensive[:price]})"

# top 3 most expensive
top3 = products.max_by(3) { |p| p[:price] }
top3.each { |p| puts "#{p[:name]}: ฿#{p[:price]}" }
```

---

## ขั้นตอนที่ 375: sort_by

```ruby
# sort_by - เรียงลำดับด้วยค่าที่ block คืนมา
words = %w[banana apple cherry date elderberry]

puts words.sort_by(&:length).inspect
# => ["date", "apple", "banana", "cherry", "elderberry"]

# sort_by กับ multiple keys (Schwartzian transform)
puts words.sort_by { |w| [w.length, w] }.inspect
# => ["date", "apple", "banana", "cherry", "elderberry"]

# sort กับ custom comparator
puts words.sort { |a, b| a.length <=> b.length }.inspect

# sort ตาม multiple criteria
students = [
  { name: "ก", grade: "A", score: 95 },
  { name: "ข", grade: "B", score: 85 },
  { name: "ค", grade: "A", score: 92 },
  { name: "ง", grade: "B", score: 88 }
]

# เรียงตาม grade ก่อน แล้วตาม score มากไปน้อย
sorted = students.sort_by { |s| [s[:grade], -s[:score]] }
sorted.each { |s| puts "#{s[:name]}: #{s[:grade]} (#{s[:score]})" }
# => ก: A (95)
# => ค: A (92)
# => ง: B (88)
# => ข: B (85)

# sort_by! แก้ไข in-place
arr = ["banana", "apple", "cherry"]
arr.sort_by!(&:length)
puts arr.inspect   # => ["apple", "banana", "cherry"]

# ตัวอย่าง: sort files by multiple criteria
files = [
  { name: "z_file.rb", size: 100, modified: "2024-01-10" },
  { name: "a_file.rb", size: 500, modified: "2024-01-15" },
  { name: "m_file.rb", size: 100, modified: "2024-01-12" }
]

# เรียงตาม size ก่อน แล้วตาม name
sorted_files = files.sort_by { |f| [f[:size], f[:name]] }
sorted_files.each { |f| puts "#{f[:name]} (#{f[:size]})" }
```

---

## ขั้นตอนที่ 376: count และ tally

```ruby
numbers = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4, 5]

# count - นับจำนวน elements
puts numbers.count          # => 11 (ทั้งหมด)
puts numbers.count(3)       # => 3 (นับ element ที่เท่ากับ 3)
puts numbers.count(&:even?) # => 6 (นับ element ที่ block คืน true)

# tally - นับความถี่ของแต่ละค่า (Ruby 2.7+)
puts numbers.tally.inspect
# => {1=>1, 2=>2, 3=>3, 4=>4, 5=>1}

# tally กับ string
words = "the quick brown fox jumps over the lazy dog".split
puts words.tally.select { |_, v| v > 1 }.inspect
# => {"the"=>2}

# tally_by (Ruby 3.1+) หรือ group_by + transform_values
colors = ["red", "blue", "red", "green", "blue", "red"]
puts colors.tally.inspect
# => {"red"=>3, "blue"=>2, "green"=>1}

# Sort by frequency
puts colors.tally.sort_by { |_, v| -v }.to_h.inspect
# => {"red"=>3, "blue"=>2, "green"=>1}

# ตัวอย่างจริง: vote counting
votes = %w[Alice Bob Alice Charlie Bob Alice Charlie Bob Alice]
result = votes.tally.sort_by { |_, v| -v }
puts "ผลการเลือกตั้ง:"
result.each_with_index do |(candidate, votes), rank|
  puts "#{rank + 1}. #{candidate}: #{votes} คะแนน"
end
```

---

## ขั้นตอนที่ 377: zip

```ruby
# zip - รวม arrays โดยจับคู่ตาม index
a = [1, 2, 3]
b = ["a", "b", "c"]
c = [:x, :y, :z]

puts a.zip(b).inspect        # => [[1, "a"], [2, "b"], [3, "c"]]
puts a.zip(b, c).inspect     # => [[1, "a", :x], [2, "b", :y], [3, "c", :z]]

# zip กับ arrays ที่ยาวไม่เท่ากัน
puts [1, 2, 3].zip([4, 5]).inspect
# => [[1, 4], [2, 5], [3, nil]]

# zip กับ block
[1, 2, 3].zip([4, 5, 6]) do |pair|
  puts "#{pair[0]} + #{pair[1]} = #{pair[0] + pair[1]}"
end

# ตัวอย่างจริง: สร้าง Hash จาก 2 arrays
keys = [:name, :age, :city]
values = ["สมชาย", 25, "กรุงเทพ"]

hash = keys.zip(values).to_h
puts hash.inspect
# => {:name=>"สมชาย", :age=>25, :city=>"กรุงเทพ"}

# ตัวอย่าง: แสดงข้อมูลเปรียบเทียบ
before = [100, 200, 300]
after = [120, 180, 350]
labels = ["Product A", "Product B", "Product C"]

labels.zip(before, after).each do |label, b, a|
  change = ((a - b) / b.to_f * 100).round(1)
  direction = change > 0 ? "↑" : "↓"
  puts "#{label}: #{b} → #{a} (#{direction}#{change.abs}%)"
end
```

---

## ขั้นตอนที่ 378: cycle

```ruby
# cycle - วนซ้ำ n รอบ หรือวนตลอดไป
days = ["จันทร์", "อังคาร", "พุธ", "พฤหัส", "ศุกร์"]

# วน 2 รอบ
days.cycle(2) { |day| print "#{day} " }
puts

# ตัวอย่าง: Round Robin scheduling
team = ["สมชาย", "สมหญิง", "สมศรี", "สมบัติ"]
tasks = (1..10).to_a

assignments = {}
team.cycle.each_with_index do |person, i|
  break if i >= tasks.size
  assignments[tasks[i]] = person
end

assignments.each { |task, person| puts "Task #{task}: #{person}" }

# cycle สำหรับ animation
colors = ["🔴", "🟡", "🟢"]
5.times do |i|
  color = colors[i % colors.size]
  print "#{color} "
end
puts

# cycle กับ Enumerator
counter = [1, 2, 3].cycle
puts counter.next   # => 1
puts counter.next   # => 2
puts counter.next   # => 3
puts counter.next   # => 1 (วนกลับ)

# ใช้ cycle สำหรับ pagination
def paginate_data(data, page_size: 5)
  data.each_slice(page_size).with_index(1).map do |page, num|
    { page: num, data: page }
  end
end

items = (1..23).to_a
pages = paginate_data(items)
puts "Total pages: #{pages.size}"
puts "Page 3: #{pages[2][:data].inspect}"
```

---

## ขั้นตอนที่ 379: first, take, drop

```ruby
data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# first
puts data.first         # => 1
puts data.first(3).inspect   # => [1, 2, 3]

# last
puts data.last          # => 10
puts data.last(3).inspect    # => [8, 9, 10]

# take - เหมือน first แต่ใช้กับ Enumerable ทั่วไป
puts data.take(4).inspect        # => [1, 2, 3, 4]
puts data.take_while { |n| n < 6 }.inspect   # => [1, 2, 3, 4, 5]

# drop
puts data.drop(3).inspect        # => [4, 5, 6, 7, 8, 9, 10]
puts data.drop_while { |n| n < 5 }.inspect   # => [5, 6, 7, 8, 9, 10]

# ตัวอย่างจริง: pagination
def paginate(collection, page:, per_page:)
  collection.drop((page - 1) * per_page).take(per_page)
end

items = (1..100).to_a
puts "Page 3 (5 per page): #{paginate(items, page: 3, per_page: 5).inspect}"
# => [11, 12, 13, 14, 15]

# take_while / drop_while
transactions = [
  { date: "2024-01-01", amount: 100 },
  { date: "2024-01-05", amount: 200 },
  { date: "2024-01-10", amount: 150 },
  { date: "2024-02-01", amount: 300 }
]

# เอาแค่ transactions ในเดือน January
jan_txns = transactions.take_while { |t| t[:date].start_with?("2024-01") }
puts jan_txns.length   # => 3
```

---

## ขั้นตอนที่ 380: find / detect

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# find (alias: detect) - หา element แรกที่ block คืน true
first_even = numbers.find { |n| n.even? }
puts first_even   # => 2

first_over_5 = numbers.find { |n| n > 5 }
puts first_over_5   # => 6

# ถ้าไม่พบคืน nil
result = numbers.find { |n| n > 100 }
puts result.nil?   # => true

# find กับ default value (proc)
result = numbers.find(-> { "not found" }) { |n| n > 100 }
puts result   # => "not found"

# find_index - คืน index
puts numbers.find_index { |n| n > 5 }   # => 5
puts numbers.find_index(7)               # => 6

# ตัวอย่างจริง: หา user ด้วย condition
users = [
  { id: 1, name: "สมชาย", role: :admin },
  { id: 2, name: "สมหญิง", role: :user },
  { id: 3, name: "สมศรี", role: :admin }
]

admin = users.find { |u| u[:role] == :admin }
puts admin[:name]   # => "สมชาย"

user = users.find { |u| u[:id] == 2 }
puts user[:name]   # => "สมหญิง"

# find_all เป็น alias ของ select
puts users.find_all { |u| u[:role] == :admin }.map { |u| u[:name] }.inspect
# => ["สมชาย", "สมศรี"]
```

---

## ขั้นตอนที่ 381: any?, all?, none?, one?

```ruby
numbers = [1, 2, 3, 4, 5]

# any? - มีอย่างน้อย 1 element ที่ block คืน true?
puts numbers.any? { |n| n > 3 }    # => true
puts numbers.any? { |n| n > 10 }   # => false
puts numbers.any?(&:even?)          # => true

# all? - ทุก element ต้อง block คืน true?
puts numbers.all? { |n| n > 0 }    # => true
puts numbers.all? { |n| n > 3 }    # => false
puts numbers.all?(&:positive?)      # => true

# none? - ไม่มี element ใดที่ block คืน true?
puts numbers.none? { |n| n > 10 }  # => true
puts numbers.none? { |n| n > 3 }   # => false
puts numbers.none?(&:negative?)     # => true

# one? - มีแค่ 1 element ที่ block คืน true?
puts numbers.one? { |n| n == 3 }   # => true
puts numbers.one? { |n| n > 3 }    # => false (มี 2 ตัวคือ 4 และ 5)

# ตัวอย่างจริง
def valid_user_input?(inputs)
  # ตรวจว่าไม่มี blank fields
  inputs.none?(&:empty?) &&
  # ตรวจว่ามีอย่างน้อยหนึ่งค่าที่ไม่ใช่ nil
  inputs.any? { |i| !i.nil? }
end

puts valid_user_input?(["สมชาย", "25", "Bangkok"])  # => true
puts valid_user_input?(["สมชาย", "", "Bangkok"])    # => false

# ใช้กับ Hash
users = [
  { name: "ก", verified: true },
  { name: "ข", verified: true },
  { name: "ค", verified: false }
]

puts users.all? { |u| u[:verified] }    # => false
puts users.any? { |u| u[:verified] }    # => true
puts users.none? { |u| u[:verified] }   # => false
puts users.one? { |u| !u[:verified] }   # => true
```

---

## ขั้นตอนที่ 382: Lazy Enumerators

```ruby
# Lazy enumerator จะไม่ evaluate จนกว่าจะถูกเรียกจริงๆ

# ไม่มี lazy - จะ enumerate ทุก element
# (1..Float::INFINITY).select { |n| n.odd? }.first(5)  # infinite loop!

# ใช้ lazy - evaluate เมื่อต้องการเท่านั้น
result = (1..Float::INFINITY).lazy.select { |n| n.odd? }.first(5)
puts result.inspect   # => [1, 3, 5, 7, 9]

# lazy chain
result = (1..Float::INFINITY)
  .lazy
  .map { |n| n * n }
  .select { |n| n % 3 == 0 }
  .reject { |n| n % 9 == 0 }
  .first(5)
puts result.inspect

# force evaluation ด้วย to_a หรือ force
lazy_result = (1..100).lazy.select(&:odd?)
puts lazy_result.class   # => Enumerator::Lazy
puts lazy_result.to_a.first(5).inspect   # => [1, 3, 5, 7, 9]

# ตัวอย่าง: หา Fibonacci ที่เป็น perfect square
fib = Enumerator.new do |y|
  a, b = 0, 1
  loop { y << a; a, b = b, a + b }
end

def perfect_square?(n)
  n >= 0 && Math.sqrt(n) == Math.sqrt(n).to_i
end

fib_squares = fib.lazy
  .select { |n| perfect_square?(n) }
  .first(5)
puts fib_squares.inspect   # => [0, 1, 1, 144, ...]

# lazy กับ custom Enumerator
power_of_2 = Enumerator.new do |y|
  n = 1
  loop { y << n; n *= 2 }
end

puts power_of_2.lazy.select { |n| n % 3 == 0 }.first(5).inspect
```

---

## ขั้นตอนที่ 383: each_with_index และ each_with_object

```ruby
# each_with_index
fruits = ["apple", "banana", "cherry"]

fruits.each_with_index do |fruit, index|
  puts "#{index + 1}. #{fruit}"
end

# map.with_index สำหรับ map พร้อม index
numbered = fruits.map.with_index(1) { |fruit, i| "#{i}. #{fruit}" }
puts numbered.inspect
# => ["1. apple", "2. banana", "3. cherry"]

# each_with_object - accumulate ผลลัพธ์ใน object
# object ที่ส่งไปจะไม่เปลี่ยน reference
words = ["hello", "world", "foo", "bar"]

word_lengths = words.each_with_object({}) do |word, hash|
  hash[word] = word.length
end
puts word_lengths.inspect
# => {"hello"=>5, "world"=>5, "foo"=>3, "bar"=>3}

# ตัวอย่าง: สร้าง lookup table
numbers = (1..5).each_with_object({}) do |n, hash|
  hash[n] = { square: n**2, cube: n**3 }
end
numbers.each { |n, v| puts "#{n}: square=#{v[:square]}, cube=#{v[:cube]}" }

# เปรียบเทียบกับ reduce
# each_with_object ไม่ต้อง return object (ต่างจาก reduce)
result1 = [1, 2, 3].each_with_object([]) { |n, arr| arr << n * 2 }
result2 = [1, 2, 3].reduce([]) { |arr, n| arr << n * 2; arr }  # ต้อง return arr
puts result1.inspect   # => [2, 4, 6]
puts result2.inspect   # => [2, 4, 6]
```

---

## ขั้นตอนที่ 384: sum, inject สำหรับ complex aggregation

```ruby
# sum
puts [1, 2, 3, 4, 5].sum   # => 15
puts [1.5, 2.5, 3.0].sum   # => 7.0

# sum กับ block (Ruby 2.4+)
words = ["hello", "world", "ruby"]
total_chars = words.sum(&:length)
puts total_chars   # => 14

# sum กับ initial value
puts [1, 2, 3].sum(100)   # => 106

# ตัวอย่างจริง
orders = [
  { product: "A", qty: 3, price: 100 },
  { product: "B", qty: 2, price: 250 },
  { product: "C", qty: 5, price: 75 }
]

total = orders.sum { |o| o[:qty] * o[:price] }
puts "ยอดรวม: ฿#{total}"   # => ฿1,425

# inject สำหรับ complex aggregation
# สร้าง running total
running_totals = [10, 20, 30, 40].inject([]) do |totals, n|
  totals << (totals.last || 0) + n
end
puts running_totals.inspect   # => [10, 30, 60, 100]

# nested aggregation
transactions = [
  { month: "Jan", amount: 1000 },
  { month: "Jan", amount: 500 },
  { month: "Feb", amount: 800 },
  { month: "Feb", amount: 1200 },
  { month: "Mar", amount: 600 }
]

monthly_totals = transactions.inject(Hash.new(0)) do |totals, t|
  totals[t[:month]] += t[:amount]
  totals
end
puts monthly_totals.inspect
# => {"Jan"=>1500, "Feb"=>2000, "Mar"=>600}
```

---

## ขั้นตอนที่ 385: flat_map สำหรับ nested collections

```ruby
# ตัวอย่างจริงของ flat_map
categories = [
  { name: "ผลไม้", items: ["แอปเปิ้ล", "กล้วย", "มะม่วง"] },
  { name: "ผัก", items: ["แครอท", "มะเขือ"] },
  { name: "เนื้อสัตว์", items: ["หมู", "ไก่", "เนื้อ"] }
]

# หา items ทั้งหมด
all_items = categories.flat_map { |cat| cat[:items] }
puts all_items.inspect
# => ["แอปเปิ้ล", "กล้วย", "มะม่วง", "แครอท", "มะเขือ", "หมู", "ไก่", "เนื้อ"]

# ตัวอย่าง: database-like query
users = [
  { name: "สมชาย", posts: [
    { title: "Ruby 101", likes: 10 },
    { title: "Rails Guide", likes: 25 }
  ]},
  { name: "สมหญิง", posts: [
    { title: "Python Tips", likes: 15 }
  ]}
]

# หา posts ทั้งหมดที่ likes > 12
popular_posts = users.flat_map { |u| u[:posts] }
  .select { |p| p[:likes] > 12 }
  .map { |p| p[:title] }

puts popular_posts.inspect
# => ["Rails Guide", "Python Tips"]

# flat_map vs map + flatten
matrix = [[1, 2], [3, 4], [5, 6]]

# map ธรรมดา
puts matrix.map { |row| row.map { |n| n * 2 } }.inspect
# => [[2, 4], [6, 8], [10, 12]]

# flat_map
puts matrix.flat_map { |row| row.map { |n| n * 2 } }.inspect
# => [2, 4, 6, 8, 10, 12]
```

---

## ขั้นตอนที่ 386: Enumerable สำหรับ Hash

```ruby
hash = { a: 1, b: 2, c: 3, d: 4, e: 5 }

# map บน Hash คืน Array of arrays
puts hash.map { |k, v| [k, v * 2] }.inspect
# => [[:a, 2], [:b, 4], [:c, 6], [:d, 8], [:e, 10]]

# transform_values คืน Hash ใหม่
puts hash.transform_values { |v| v * 2 }.inspect
# => {:a=>2, :b=>4, :c=>6, :d=>8, :e=>10}

# transform_keys คืน Hash ใหม่
puts hash.transform_keys { |k| k.to_s }.inspect
# => {"a"=>1, "b"=>2, "c"=>3, "d"=>4, "e"=>5}

# select บน Hash
puts hash.select { |k, v| v > 3 }.inspect
# => {:d=>4, :e=>5}

# reject บน Hash
puts hash.reject { |k, v| v.even? }.inspect
# => {:a=>1, :c=>3, :e=>5}

# any? / all? / none? บน Hash
puts hash.any? { |k, v| v > 4 }    # => true
puts hash.all? { |k, v| v > 0 }    # => true
puts hash.none? { |k, v| v < 0 }   # => true

# min_by / max_by บน Hash
puts hash.min_by { |k, v| v }.inspect   # => [:a, 1]
puts hash.max_by { |k, v| v }.inspect   # => [:e, 5]

# sort_by บน Hash
puts hash.sort_by { |k, v| -v }.to_h.inspect
# => {:e=>5, :d=>4, :c=>3, :b=>2, :a=>1}

# each_with_object บน Hash
result = hash.each_with_object([]) do |(k, v), arr|
  arr << "#{k}=#{v}" if v.odd?
end
puts result.inspect   # => ["a=1", "c=3", "e=5"]
```

---

## ขั้นตอนที่ 387: Custom Enumerable Class

```ruby
# สร้าง class ที่ include Enumerable
class WordCollection
  include Enumerable
  
  def initialize
    @words = []
  end
  
  def add(word)
    @words << word.downcase
    self  # allow chaining
  end
  
  # Method เดียวที่ต้องกำหนด
  def each(&block)
    @words.each(&block)
  end
  
  def to_s
    @words.join(", ")
  end
end

wc = WordCollection.new
wc.add("Ruby").add("Python").add("JavaScript").add("Go").add("Rust")

# ได้ methods ทั้งหมดจาก Enumerable ฟรี!
puts wc.sort.inspect
puts wc.map(&:upcase).inspect
puts wc.select { |w| w.length > 3 }.inspect
puts wc.min
puts wc.max
puts wc.count
puts wc.include?("ruby")   # => true
puts wc.first
puts wc.min_by(&:length)
puts wc.max_by(&:length)

# ตัวอย่างที่ซับซ้อนกว่า: NumberSet
class NumberSet
  include Enumerable
  include Comparable
  
  def initialize(*numbers)
    @numbers = numbers.uniq.sort
  end
  
  def each(&block)
    @numbers.each(&block)
  end
  
  def <=>(other)
    to_a <=> other.to_a
  end
  
  def +(other)
    NumberSet.new(*(@numbers + other.to_a))
  end
  
  def -(other)
    NumberSet.new(*(@numbers - other.to_a))
  end
  
  def &(other)
    NumberSet.new(*(@numbers & other.to_a))
  end
  
  def to_s
    "{#{@numbers.join(', ')}}"
  end
  
  def inspect
    "NumberSet#{to_s}"
  end
end

set1 = NumberSet.new(1, 2, 3, 4, 5)
set2 = NumberSet.new(3, 4, 5, 6, 7)

puts (set1 + set2).to_s    # => {1, 2, 3, 4, 5, 6, 7}
puts (set1 - set2).to_s    # => {1, 2}
puts (set1 & set2).to_s    # => {3, 4, 5}
puts set1.sum               # => 15
puts set1.select(&:even?).inspect   # => [2, 4]
```

---

## ขั้นตอนที่ 388: Enumerable Chaining

```ruby
# Chaining หลาย Enumerable methods
employees = [
  { name: "ก", dept: :engineering, salary: 80000, years: 5 },
  { name: "ข", dept: :marketing, salary: 70000, years: 3 },
  { name: "ค", dept: :engineering, salary: 95000, years: 8 },
  { name: "ง", dept: :marketing, salary: 65000, years: 2 },
  { name: "จ", dept: :engineering, salary: 75000, years: 4 },
  { name: "ฉ", dept: :hr, salary: 60000, years: 6 }
]

# หาเงินเดือนเฉลี่ยของ Engineering
eng_avg = employees
  .select { |e| e[:dept] == :engineering }
  .map { |e| e[:salary] }
  .then { |salaries| salaries.sum.to_f / salaries.size }

puts "Engineering average: ฿#{eng_avg.round(0)}"

# หา top earner ในแต่ละ dept
top_earners = employees
  .group_by { |e| e[:dept] }
  .transform_values { |dept_employees| dept_employees.max_by { |e| e[:salary] } }

top_earners.each do |dept, emp|
  puts "#{dept}: #{emp[:name]} (฿#{emp[:salary]})"
end

# Report: employees with 4+ years sorted by salary
report = employees
  .select { |e| e[:years] >= 4 }
  .sort_by { |e| -e[:salary] }
  .map.with_index(1) { |e, i| "#{i}. #{e[:name]}: ฿#{e[:salary]} (#{e[:years]} ปี)" }

puts "\nพนักงานอาวุโส (4+ ปี) เรียงตามเงินเดือน:"
report.each { |line| puts line }
```

---

## ขั้นตอนที่ 389: Performance Considerations

```ruby
require 'benchmark'

n = 100_000
numbers = (1..n).to_a

Benchmark.bm(20) do |x|
  # map + select vs filter_map
  x.report("map + select:     ") { numbers.map { |n| n * 2 }.select { |n| n > 100000 } }
  x.report("filter_map:       ") { numbers.filter_map { |n| n * 2 if n * 2 > 100000 } }
  
  # each_with_object vs reduce
  x.report("each_with_object: ") { numbers.each_with_object([]) { |n, a| a << n * 2 } }
  x.report("map:              ") { numbers.map { |n| n * 2 } }
  
  # any? short-circuits
  x.report("any? (first):     ") { numbers.any? { |n| n == 1 } }
  x.report("any? (last):      ") { numbers.any? { |n| n == n } }
end

# Lazy vs eager
n = 1_000_000
Benchmark.bm(15) do |x|
  x.report("eager first(10):") { (1..n).select { |n| n.odd? }.first(10) }
  x.report("lazy first(10): ") { (1..n).lazy.select { |n| n.odd? }.first(10) }
end

# Tips สำหรับ performance:
# 1. ใช้ lazy สำหรับ large collections และ short-circuit operations
# 2. ใช้ filter_map แทน map + select เมื่อทำได้
# 3. ใช้ find แทน select.first
# 4. ใช้ any? แทน select.empty? หรือ count > 0
# 5. ใช้ tally แทน group_by + transform_values count
```

---

## ขั้นตอนที่ 390: Real-World Enumerable Patterns

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

  def aggregate(method = :to_a, &block)
    block ? @data.send(method, &block) : @data.send(method)
  end

  def result
    @data
  end
end

sales = [
  { product: "A", amount: 1000, month: "Jan", region: "North" },
  { product: "B", amount: 2500, month: "Jan", region: "South" },
  { product: "A", amount: 1500, month: "Feb", region: "North" },
  { product: "C", amount: 800,  month: "Feb", region: "East" },
  { product: "B", amount: 3000, month: "Mar", region: "South" }
]

# ยอดขายรวมของ Product A
total_a = DataPipeline.new(sales)
  .filter { |s| s[:product] == "A" }
  .aggregate(:sum) { |s| s[:amount] }
puts "Product A total: ฿#{total_a}"

# Pattern 2: Memoization with Enumerable
class MemoizedCollection
  include Enumerable

  def initialize(&generator)
    @generator = generator
    @cache = []
    @enum = nil
  end

  def each
    return to_enum(:each) unless block_given?
    @cache.each { |item| yield item }
    @enum ||= @generator.call
    loop do
      item = @enum.next
      @cache << item
      yield item
    end
  rescue StopIteration
    # done
  end
end

# Pattern 3: Infinite sequence ด้วย Lazy
class InfiniteSequence
  include Enumerable

  def each
    return to_enum(:each) unless block_given?
    n = 0
    loop { yield generate(n); n += 1 }
  end

  def take(n)
    lazy.first(n)
  end

  private

  def generate(n)
    raise NotImplementedError
  end
end

class PrimeSequence < InfiniteSequence
  def generate(n)
    count = 0
    candidate = 2
    loop do
      if prime?(candidate)
        count += 1
        return candidate if count > n
      end
      candidate += 1
    end
  end

  private

  def prime?(n)
    return false if n < 2
    (2..Math.sqrt(n).to_i).none? { |i| n % i == 0 }
  end
end

primes = PrimeSequence.new
puts primes.take(10).inspect
```

---

## แบบฝึกหัด 25 ข้อ

### ข้อ 1-5: map และ select

**ข้อ 1:** แปลง array ของ temperatures จาก Celsius เป็น Fahrenheit
```ruby
# เฉลย
celsius = [0, 20, 37, 100]
fahrenheit = celsius.map { |c| (c * 9.0 / 5) + 32 }
puts fahrenheit.inspect   # => [32.0, 68.0, 98.6, 212.0]
```

**ข้อ 2:** filter users ที่อายุ 18-60 และ active
```ruby
# เฉลย
users = [
  { name: "ก", age: 15, active: true },
  { name: "ข", age: 25, active: true },
  { name: "ค", age: 65, active: true },
  { name: "ง", age: 30, active: false }
]

eligible = users.select { |u| (18..60).include?(u[:age]) && u[:active] }
puts eligible.map { |u| u[:name] }.inspect   # => ["ข"]
```

**ข้อ 3:** แปลง array ของ strings เป็น format title case
```ruby
# เฉลย
titles = ["ruby on rails", "the art of war", "clean code"]
title_cased = titles.map { |t| t.split.map(&:capitalize).join(' ') }
puts title_cased.inspect
```

**ข้อ 4:** หา products ที่มีส่วนลดและคำนวณราคาสุทธิ
```ruby
# เฉลย
products = [
  { name: "A", price: 100, discount: 0 },
  { name: "B", price: 200, discount: 10 },
  { name: "C", price: 150, discount: 20 }
]

discounted = products
  .select { |p| p[:discount] > 0 }
  .map { |p| { name: p[:name], final_price: p[:price] * (1 - p[:discount] / 100.0) } }

puts discounted.inspect
```

**ข้อ 5:** flat_map เพื่อสร้าง word list จาก paragraph
```ruby
# เฉลย
paragraphs = [
  "Ruby is elegant",
  "Python is readable",
  "Go is fast"
]

words = paragraphs.flat_map { |p| p.split }
puts words.uniq.sort.inspect
```

### ข้อ 6-10: reduce และ group_by

**ข้อ 6:** คำนวณ running total
```ruby
# เฉลย
payments = [100, 250, 75, 300, 150]
running_total = payments.each_with_object([0]) { |n, acc| acc << acc.last + n }
puts running_total.inspect   # => [0, 100, 350, 425, 725, 875]
```

**ข้อ 7:** จัดกลุ่ม transactions ตามเดือนและหายอดรวม
```ruby
# เฉลย
transactions = [
  { month: 1, amount: 1000 },
  { month: 1, amount: 500 },
  { month: 2, amount: 800 },
  { month: 2, amount: 1200 },
  { month: 3, amount: 600 }
]

monthly = transactions.group_by { |t| t[:month] }
  .transform_values { |txns| txns.sum { |t| t[:amount] } }

puts monthly.inspect
```

**ข้อ 8:** สร้าง inverted index
```ruby
# เฉลย
documents = {
  1 => "ruby is great",
  2 => "python is readable",
  3 => "ruby and python"
}

index = documents.flat_map { |id, text|
  text.split.map { |word| [word, id] }
}.group_by { |word, _| word }
 .transform_values { |pairs| pairs.map { |_, id| id }.uniq.sort }

puts index["ruby"].inspect    # => [1, 3]
puts index["python"].inspect  # => [2, 3]
```

**ข้อ 9:** หา unique visitors per page
```ruby
# เฉลย
visits = [
  { page: "/home", user: "alice" },
  { page: "/about", user: "bob" },
  { page: "/home", user: "alice" },
  { page: "/home", user: "charlie" },
  { page: "/about", user: "alice" }
]

unique_visitors = visits.group_by { |v| v[:page] }
  .transform_values { |vs| vs.map { |v| v[:user] }.uniq.count }

puts unique_visitors.inspect
```

**ข้อ 10:** สร้าง matrix multiplication ด้วย Enumerable
```ruby
# เฉลย
def matrix_multiply(a, b)
  b_t = b.transpose
  a.map { |row|
    b_t.map { |col|
      row.zip(col).sum { |x, y| x * y }
    }
  }
end

a = [[1, 2], [3, 4]]
b = [[5, 6], [7, 8]]
result = matrix_multiply(a, b)
puts result.inspect   # => [[19, 22], [43, 50]]
```

### ข้อ 11-15: Sorting และ Finding

**ข้อ 11:** เรียง students ตาม grade แล้ว name
```ruby
# เฉลย
students = [
  { name: "Zinc", grade: "B", score: 85 },
  { name: "Alpha", grade: "A", score: 95 },
  { name: "Beta", grade: "B", score: 88 },
  { name: "Delta", grade: "A", score: 92 }
]

sorted = students.sort_by { |s| [s[:grade], s[:name]] }
sorted.each { |s| puts "#{s[:grade]} #{s[:name]}: #{s[:score]}" }
```

**ข้อ 12:** หา median ของ array
```ruby
# เฉลย
def median(arr)
  sorted = arr.sort
  n = sorted.size
  n.odd? ? sorted[n / 2] : (sorted[n/2 - 1] + sorted[n/2]) / 2.0
end

puts median([1, 3, 5, 7, 9])    # => 5
puts median([1, 2, 3, 4])       # => 2.5
```

**ข้อ 13:** หา mode (ค่าที่ปรากฏบ่อยที่สุด)
```ruby
# เฉลย
def mode(arr)
  tally = arr.tally
  max_count = tally.values.max
  tally.select { |_, count| count == max_count }.keys
end

puts mode([1, 2, 2, 3, 3, 3, 4]).inspect   # => [3]
puts mode([1, 2, 2, 3, 3]).inspect          # => [2, 3]
```

**ข้อ 14:** หา consecutive sequences ใน array ด้วย chunk_while
```ruby
# เฉลย
numbers = [1, 2, 3, 7, 8, 9, 15, 16]

sequences = numbers.chunk_while { |a, b| b - a == 1 }.to_a
puts sequences.inspect
# => [[1, 2, 3], [7, 8, 9], [15, 16]]

# หา sequence ที่ยาวที่สุด
longest = sequences.max_by(&:length)
puts "Longest: #{longest.inspect}"
```

**ข้อ 15:** สร้าง top-N leaderboard
```ruby
# เฉลย
scores = [
  { player: "ก", score: 850 },
  { player: "ข", score: 1200 },
  { player: "ค", score: 750 },
  { player: "ง", score: 1500 },
  { player: "จ", score: 950 }
]

def leaderboard(scores, top_n: 3)
  scores.sort_by { |s| -s[:score] }
        .first(top_n)
        .map.with_index(1) { |s, rank| { rank: rank, **s } }
end

leaderboard(scores).each do |entry|
  puts "##{entry[:rank]} #{entry[:player]}: #{entry[:score]}"
end
```

### ข้อ 16-20: Custom Enumerable

**ข้อ 16:** สร้าง PlayList class ที่ใช้ Enumerable
```ruby
# เฉลย
class PlayList
  include Enumerable
  
  Song = Struct.new(:title, :artist, :duration_secs)
  
  def initialize
    @songs = []
  end

  def add(title, artist, duration_secs)
    @songs << Song.new(title, artist, duration_secs)
    self
  end

  def each(&block)
    @songs.each(&block)
  end

  def total_duration
    sum(&:duration_secs)
  end

  def by_artist(name)
    select { |s| s.artist == name }
  end
end

pl = PlayList.new
pl.add("Song A", "Artist 1", 210)
  .add("Song B", "Artist 2", 185)
  .add("Song C", "Artist 1", 240)

puts "Total duration: #{pl.total_duration}s"
puts "Artist 1 songs: #{pl.by_artist("Artist 1").map(&:title).inspect}"
puts "Shortest: #{pl.min_by(&:duration_secs).title}"
```

**ข้อ 17:** สร้าง FamilyTree class
```ruby
# เฉลย
class FamilyTree
  include Enumerable
  
  Person = Struct.new(:name, :age, :generation)
  
  def initialize
    @members = []
  end

  def add_member(name, age, generation)
    @members << Person.new(name, age, generation)
    self
  end

  def each(&block)
    @members.each(&block)
  end

  def by_generation(gen)
    select { |p| p.generation == gen }
  end

  def oldest
    max_by(&:age)
  end

  def youngest
    min_by(&:age)
  end
end

tree = FamilyTree.new
tree.add_member("ปู่", 80, 1)
    .add_member("ย่า", 75, 1)
    .add_member("พ่อ", 55, 2)
    .add_member("แม่", 52, 2)
    .add_member("ฉัน", 25, 3)

puts "ผู้เฒ่าที่สุด: #{tree.oldest.name}"
puts "คนรุ่น 2: #{tree.by_generation(2).map(&:name).inspect}"
```

### ข้อ 18-25: Applications

```ruby
# ข้อ 18: Statistics class
class Statistics
  def self.analyze(data)
    sorted = data.sort
    n = data.size
    
    {
      count:  n,
      sum:    data.sum,
      mean:   data.sum.to_f / n,
      median: n.odd? ? sorted[n/2] : (sorted[n/2-1] + sorted[n/2]) / 2.0,
      mode:   data.tally.max_by { |_, c| c }.first,
      min:    sorted.first,
      max:    sorted.last,
      range:  sorted.last - sorted.first,
      variance: data.sum { |x| (x - data.sum.to_f/n) ** 2 } / n
    }.tap { |s| s[:std_dev] = Math.sqrt(s[:variance]).round(4) }
  end
end

data = [23, 45, 12, 67, 23, 89, 34, 56, 23, 78]
stats = Statistics.analyze(data)
stats.each { |key, val| puts "#{key}: #{val}" }

# ข้อ 19: ใช้ lazy สำหรับ infinite fibonacci
def fibonacci_lazy
  Enumerator.new do |y|
    a, b = 0, 1
    loop { y << a; a, b = b, a + b }
  end.lazy
end

# หา fib ที่ < 1000
puts fibonacci_lazy.select { |n| n < 1000 }.to_a.inspect

# ข้อ 20-25: ฝึกเอง
# สร้าง shopping cart ที่ใช้ Enumerable
class ShoppingCart
  include Enumerable
  
  Item = Struct.new(:name, :price, :qty)
  
  def initialize
    @items = []
  end
  
  def add(name, price, qty: 1)
    existing = @items.find { |i| i.name == name }
    if existing
      existing.qty += qty
    else
      @items << Item.new(name, price, qty)
    end
    self
  end
  
  def each(&block) = @items.each(&block)
  
  def subtotal = sum { |i| i.price * i.qty }
  
  def discount(pct)
    subtotal * (1 - pct / 100.0)
  end
  
  def tax(rate = 7)
    subtotal * rate / 100.0
  end
  
  def total(discount_pct: 0, tax_rate: 7)
    base = subtotal * (1 - discount_pct / 100.0)
    base + base * tax_rate / 100.0
  end
  
  def summary
    map { |i| "#{i.name} x#{i.qty} = ฿#{(i.price * i.qty).round(2)}" }
  end
end

cart = ShoppingCart.new
cart.add("หนังสือ", 300, qty: 2)
    .add("ปากกา", 50, qty: 5)
    .add("สมุด", 120, qty: 3)

puts "รายการ:"
cart.summary.each { |line| puts "  #{line}" }
puts "รวม: ฿#{cart.subtotal}"
puts "ภาษี 7%: ฿#{cart.tax.round(2)}"
puts "ทั้งหมด: ฿#{cart.total.round(2)}"
puts "กับส่วนลด 10%: ฿#{cart.total(discount_pct: 10).round(2)}"
puts "สินค้าแพงที่สุด: #{cart.max_by(&:price).name}"
puts "จำนวนสินค้า: #{cart.count} ชิ้น"
puts "ทุกชิ้นราคา > 40: #{cart.all? { |i| i.price > 40 }}"
```

---

## สรุปบทที่ 18

ในบทนี้เราได้เรียนรู้ Enumerable module อย่างละเอียด:

**Transformation:**
- `map` / `collect` - แปลง elements
- `flat_map` - map + flatten
- `filter_map` - select + map ใน step เดียว

**Filtering:**
- `select` / `filter` - เลือก elements
- `reject` - ตรงข้ามกับ select

**Aggregation:**
- `reduce` / `inject` - สะสมค่า
- `sum` - รวมค่า
- `each_with_object` - accumulate ใน object

**Grouping:**
- `group_by` - จัดกลุ่ม
- `chunk` / `chunk_while` - จัดกลุ่ม consecutive
- `tally` - นับความถี่

**Slicing:**
- `each_slice` - แบ่ง chunks
- `each_cons` - sliding window
- `first` / `take` / `drop` - ตัด collection

**Finding:**
- `find` / `detect` - หา element แรก
- `any?` / `all?` / `none?` / `one?` - boolean checks

**Sorting:**
- `sort_by` - เรียงด้วย key
- `min_by` / `max_by` / `minmax_by` - หาค่าสุด

**Advanced:**
- `lazy` - lazy evaluation สำหรับ infinite sequences
- Custom Enumerable classes
