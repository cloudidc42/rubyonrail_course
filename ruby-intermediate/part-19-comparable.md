# Part 19: Comparable และ Enumerable (Custom Implementation)

## ขั้นตอนที่ 391-410

---

## บทนำ

ในบทที่แล้วเราเรียนรู้การใช้ Enumerable methods ที่มีอยู่แล้ว ในบทนี้เราจะเรียนรู้วิธีสร้าง class ของตัวเองที่ implement Comparable และ Enumerable ซึ่งจะทำให้ objects ของเรามีความสามารถเทียบเท่ากับ Array และ Hash ที่ Ruby มีในตัว

---

## ขั้นตอนที่ 391: Comparable Module คืออะไร?

```ruby
# Comparable module ต้องการแค่ <=> operator
# และจะให้ >, >=, <, <=, between?, clamp มาฟรี!

# ตรวจดู Comparable methods
puts Comparable.instance_methods.sort.inspect
# => [:between?, :clamp, :>=, :>, :<=, :<]

# Integer ใช้ Comparable
puts 5.between?(1, 10)     # => true
puts 15.clamp(1, 10)       # => 10
puts 1 <=> 2               # => -1
puts 2 <=> 2               # => 0
puts 3 <=> 2               # => 1

# <=> (spaceship operator) คืน:
# -1 ถ้า self < other
#  0 ถ้า self == other
#  1 ถ้า self > other
# nil ถ้า เปรียบเทียบไม่ได้
```

---

## ขั้นตอนที่ 392: Implementing Comparable

```ruby
# สร้าง Temperature class
class Temperature
  include Comparable
  
  attr_reader :degrees, :scale
  
  def initialize(degrees, scale = :celsius)
    @degrees = degrees.to_f
    @scale = scale
  end
  
  # Method เดียวที่ต้องกำหนด!
  def <=>(other)
    to_celsius <=> other.to_celsius
  end
  
  def to_celsius
    case @scale
    when :celsius    then @degrees
    when :fahrenheit then (@degrees - 32) * 5.0 / 9
    when :kelvin     then @degrees - 273.15
    end
  end
  
  def to_fahrenheit
    to_celsius * 9.0 / 5 + 32
  end
  
  def to_kelvin
    to_celsius + 273.15
  end
  
  def to_s
    "#{@degrees}°#{@scale.to_s[0].upcase}"
  end
  
  def inspect
    "Temperature(#{to_s})"
  end
end

# ทดสอบ
boiling = Temperature.new(100, :celsius)
body    = Temperature.new(98.6, :fahrenheit)
room    = Temperature.new(293.15, :kelvin)
freezing = Temperature.new(32, :fahrenheit)

puts boiling > body      # => true
puts body.between?(room, boiling)  # => true
puts freezing == Temperature.new(0, :celsius)  # => true

# sort temperatures!
temps = [boiling, body, room, freezing]
sorted = temps.sort
sorted.each { |t| puts "#{t} = #{t.to_celsius.round(1)}°C" }

# min / max ได้ฟรี
puts "Coldest: #{temps.min}"
puts "Hottest: #{temps.max}"

# clamp
very_hot = Temperature.new(150, :celsius)
normal_range = Temperature.new(20, :celsius)..Temperature.new(30, :celsius)
# puts very_hot.clamp(normal_range)  # => 30°C
```

---

## ขั้นตอนที่ 393: Comparable กับ sort

```ruby
class Product
  include Comparable
  
  attr_reader :name, :price, :rating
  
  def initialize(name, price, rating)
    @name = name
    @price = price
    @rating = rating
  end
  
  # เปรียบเทียบตาม price ก่อน แล้ว rating
  def <=>(other)
    result = @price <=> other.price
    result == 0 ? other.rating <=> @rating : result
  end
  
  def to_s
    "#{@name} (฿#{@price}, ★#{@rating})"
  end
end

products = [
  Product.new("หนังสือ A", 300, 4.5),
  Product.new("หนังสือ B", 200, 4.8),
  Product.new("หนังสือ C", 300, 4.2),
  Product.new("หนังสือ D", 150, 3.9),
  Product.new("หนังสือ E", 200, 4.5)
]

puts "เรียงตาม price (น้อยไปมาก), rating (มากไปน้อย):"
products.sort.each { |p| puts "  #{p}" }

puts "\nราคาถูกสุด: #{products.min}"
puts "ราคาแพงสุด: #{products.max}"
puts "ระหว่าง ฿150-฿250: #{products.select { |p| p.between?(products.min, products.sort[1]) }.map(&:to_s).inspect}"
```

---

## ขั้นตอนที่ 394: Comparable - Custom Sort Orders

```ruby
# สร้าง SemVer (Semantic Version) class
class SemVer
  include Comparable
  
  attr_reader :major, :minor, :patch, :pre_release
  
  VERSION_REGEX = /\A(\d+)\.(\d+)\.(\d+)(?:-(.+))?\z/
  
  def initialize(version_string)
    md = version_string.match(VERSION_REGEX)
    raise ArgumentError, "Invalid version: #{version_string}" unless md
    
    @major = md[1].to_i
    @minor = md[2].to_i
    @patch = md[3].to_i
    @pre_release = md[4]
  end
  
  def <=>(other)
    return major <=> other.major unless major == other.major
    return minor <=> other.minor unless minor == other.minor
    return patch <=> other.patch unless patch == other.patch
    
    # pre-release versions are lower than release
    # "1.0.0-alpha" < "1.0.0"
    case [pre_release.nil?, other.pre_release.nil?]
    when [true, false]  then 1   # no pre-release > has pre-release
    when [false, true]  then -1  # has pre-release < no pre-release
    when [true, true]   then 0
    else pre_release <=> other.pre_release
    end
  end
  
  def to_s
    base = "#{major}.#{minor}.#{patch}"
    pre_release ? "#{base}-#{pre_release}" : base
  end
  
  def next_patch = SemVer.new("#{major}.#{minor}.#{patch + 1}")
  def next_minor = SemVer.new("#{major}.#{minor + 1}.0")
  def next_major = SemVer.new("#{major + 1}.0.0")
end

versions = %w[1.0.0 2.1.0 1.5.3 2.0.0-beta 2.0.0-alpha 2.0.0 1.0.1].map { |v| SemVer.new(v) }

puts "เรียงลำดับ:"
versions.sort.each { |v| puts "  #{v}" }

v1 = SemVer.new("1.2.3")
v2 = SemVer.new("1.2.4")

puts "#{v1} < #{v2}: #{v1 < v2}"
puts "Next: #{v1.next_patch}"
puts "Next minor: #{v1.next_minor}"
```

---

## ขั้นตอนที่ 395: Comparable - between? และ clamp

```ruby
class Score
  include Comparable
  
  attr_reader :value, :max
  
  def initialize(value, max: 100)
    @value = [[value, 0].max, max].min  # clamp to 0..max
    @max = max
  end
  
  def <=>(other)
    @value <=> other.value
  end
  
  def percentage
    (@value.to_f / @max * 100).round(1)
  end
  
  def grade
    case percentage
    when 90..100 then "A"
    when 80...90 then "B"
    when 70...80 then "C"
    when 60...70 then "D"
    else "F"
    end
  end
  
  def to_s
    "#{@value}/#{@max} (#{percentage}% - #{grade})"
  end
end

scores = [
  Score.new(85),
  Score.new(92),
  Score.new(73),
  Score.new(58),
  Score.new(100)
]

# between? ทำงานได้ทันที!
passing = Score.new(60)
honor   = Score.new(90)

scores.each do |s|
  status = if s >= honor
    "เกียรตินิยม"
  elsif s >= passing
    "ผ่าน"
  else
    "ไม่ผ่าน"
  end
  puts "#{s} - #{status}"
end

# clamp
min_score = Score.new(50)
max_score = Score.new(80)
test = Score.new(95)
clamped = test.clamp(min_score, max_score)
puts "Clamped #{test.value} to: #{clamped.value}"

puts "\nMax score: #{scores.max}"
puts "Min score: #{scores.min}"
puts "Average: #{scores.sum(&:value).to_f / scores.size}"
```

---

## ขั้นตอนที่ 396: Building Sortable Collection

```ruby
# สร้าง collection ที่ sort ได้ด้วย Comparable
class SortedArray
  include Enumerable
  include Comparable
  
  def initialize(comparator = nil)
    @data = []
    @comparator = comparator || method(:default_compare)
  end
  
  def insert(item)
    index = @data.bsearch_index { |x| @comparator.call(x, item) >= 0 } || @data.size
    @data.insert(index, item)
    self
  end
  
  def <<(item)
    insert(item)
  end
  
  def each(&block)
    @data.each(&block)
  end
  
  def <=>(other)
    @data <=> other.to_a
  end
  
  def [](index)
    @data[index]
  end
  
  def size = @data.size
  
  def to_s = @data.inspect
  
  private
  
  def default_compare(a, b)
    a <=> b
  end
end

# sorted array ปกติ
sorted = SortedArray.new
[5, 3, 8, 1, 9, 2, 7, 4, 6].each { |n| sorted << n }
puts sorted.to_s   # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
puts "First 3: #{sorted.first(3).inspect}"
puts "Evens: #{sorted.select(&:even?).inspect}"

# sorted array กับ custom comparator (ตามความยาว string)
str_sorted = SortedArray.new(->(a, b) { a.length <=> b.length })
["banana", "apple", "fig", "cherry", "kiwi"].each { |w| str_sorted << w }
puts str_sorted.to_s   # => ["fig", "kiwi", "apple", "banana", "cherry"]
```

---

## ขั้นตอนที่ 397: Enumerable Module - Foundation

```ruby
# Enumerable ต้องการแค่ each method
# จากนั้นจะได้ methods มากกว่า 50 ตัวฟรี

# ตัวอย่างง่ายๆ: class ที่ implement each
class Countdown
  include Enumerable
  
  def initialize(start)
    @start = start
  end
  
  def each
    return to_enum(:each) unless block_given?
    @start.downto(0) { |n| yield n }
  end
end

cd = Countdown.new(5)

# ได้ methods ทั้งหมดฟรี!
puts cd.to_a.inspect         # => [5, 4, 3, 2, 1, 0]
puts cd.min                  # => 0
puts cd.max                  # => 5
puts cd.select(&:even?).inspect   # => [4, 2, 0]
puts cd.map { |n| n * 2 }.inspect # => [10, 8, 6, 4, 2, 0]
puts cd.sum                  # => 15
puts cd.include?(3)          # => true
puts cd.sort.inspect         # => [0, 1, 2, 3, 4, 5]
puts cd.first(3).inspect     # => [5, 4, 3]
puts cd.count                # => 6
```

---

## ขั้นตอนที่ 398: Custom Collection - Stack

```ruby
class Stack
  include Enumerable
  
  def initialize
    @data = []
  end
  
  def push(item)
    @data.push(item)
    self
  end
  
  alias_method :<<, :push
  
  def pop
    @data.pop
  end
  
  def peek
    @data.last
  end
  
  def empty?
    @data.empty?
  end
  
  def size
    @data.size
  end
  
  # each iterates from top to bottom
  def each(&block)
    return to_enum(:each) unless block_given?
    @data.reverse_each(&block)
  end
  
  def to_s
    "Stack(top: #{peek.inspect}, size: #{size})"
  end
  
  def inspect
    "#<Stack [#{@data.reverse.inspect}]>"
  end
end

stack = Stack.new
stack.push(1).push(2).push(3)
stack << 4 << 5

puts stack.to_s         # Stack(top: 5, size: 5)
puts stack.peek         # 5
puts stack.pop          # 5
puts stack.to_a.inspect # [4, 3, 2, 1] (top to bottom)

# ได้ Enumerable methods ฟรี!
puts stack.include?(3)     # => true
puts stack.max             # => 4
puts stack.sum             # => 10
puts stack.select { |n| n.even? }.inspect  # => [4, 2]
```

---

## ขั้นตอนที่ 399: Custom Collection - LinkedList

```ruby
class LinkedList
  include Enumerable
  
  Node = Struct.new(:value, :next_node)
  
  def initialize
    @head = nil
    @size = 0
  end
  
  def prepend(value)
    @head = Node.new(value, @head)
    @size += 1
    self
  end
  
  def append(value)
    new_node = Node.new(value, nil)
    
    if @head.nil?
      @head = new_node
    else
      current = @head
      current = current.next_node while current.next_node
      current.next_node = new_node
    end
    
    @size += 1
    self
  end
  
  def delete(value)
    return if @head.nil?
    
    if @head.value == value
      @head = @head.next_node
      @size -= 1
      return
    end
    
    current = @head
    while current.next_node
      if current.next_node.value == value
        current.next_node = current.next_node.next_node
        @size -= 1
        return
      end
      current = current.next_node
    end
  end
  
  def each
    return to_enum(:each) unless block_given?
    current = @head
    while current
      yield current.value
      current = current.next_node
    end
  end
  
  def size = @size
  
  def to_s
    "LinkedList(#{to_a.join(' -> ')})"
  end
end

list = LinkedList.new
list.append(1).append(2).append(3).append(4).append(5)
list.prepend(0)

puts list.to_s
# => LinkedList(0 -> 1 -> 2 -> 3 -> 4 -> 5)

puts list.to_a.inspect         # => [0, 1, 2, 3, 4, 5]
puts list.select(&:even?).inspect # => [0, 2, 4]
puts list.map { |n| n * n }.inspect # => [0, 1, 4, 9, 16, 25]
puts list.include?(3)   # => true
puts list.min           # => 0
puts list.max           # => 5
puts list.sum           # => 15

list.delete(3)
puts list.to_s   # => LinkedList(0 -> 1 -> 2 -> 4 -> 5)
```

---

## ขั้นตอนที่ 400: Custom Collection - Tree

```ruby
class BinaryTree
  include Enumerable
  
  attr_accessor :value, :left, :right
  
  def initialize(value = nil)
    @value = value
    @left = nil
    @right = nil
  end
  
  def insert(val)
    if @value.nil?
      @value = val
    elsif val <= @value
      @left ? @left.insert(val) : @left = BinaryTree.new(val)
    else
      @right ? @right.insert(val) : @right = BinaryTree.new(val)
    end
    self
  end
  
  # in-order traversal (sorted)
  def each(&block)
    return to_enum(:each) unless block_given?
    @left&.each(&block)
    yield @value if @value
    @right&.each(&block)
  end
  
  # pre-order traversal
  def pre_order(&block)
    return to_enum(:pre_order) unless block_given?
    yield @value if @value
    @left&.pre_order(&block)
    @right&.pre_order(&block)
  end
  
  # search
  def search(val)
    return self if @value == val
    return nil if @value.nil?
    
    if val < @value
      @left&.search(val)
    else
      @right&.search(val)
    end
  end
  
  def height
    return 0 if @value.nil?
    left_h  = @left ? @left.height : 0
    right_h = @right ? @right.height : 0
    1 + [left_h, right_h].max
  end
  
  def to_s
    "BinaryTree(#{to_a.inspect})"
  end
end

tree = BinaryTree.new
[5, 3, 7, 1, 4, 6, 8, 2].each { |n| tree.insert(n) }

puts "In-order: #{tree.to_a.inspect}"
# => [1, 2, 3, 4, 5, 6, 7, 8]

# Enumerable methods!
puts "Max: #{tree.max}"          # => 8
puts "Min: #{tree.min}"          # => 1
puts "Sum: #{tree.sum}"          # => 36
puts "Evens: #{tree.select(&:even?).inspect}"  # => [2, 4, 6, 8]
puts "Height: #{tree.height}"

found = tree.search(4)
puts "Found 4: #{found.value}"
```

---

## ขั้นตอนที่ 401: Comparable กับ Enumerable ร่วมกัน

```ruby
# สร้าง SortedSet class ที่ implement ทั้งสอง modules
class SortedSet
  include Enumerable
  include Comparable
  
  def initialize(*items)
    @data = items.flatten.uniq.sort
  end
  
  def add(item)
    unless @data.include?(item)
      @data = (@data + [item]).sort
    end
    self
  end
  
  def delete(item)
    @data.delete(item)
    self
  end
  
  def each(&block)
    @data.each(&block)
  end
  
  # Comparable
  def <=>(other)
    @data <=> other.to_a
  end
  
  # Set operations
  def |(other)
    SortedSet.new(@data + other.to_a)
  end
  
  def &(other)
    SortedSet.new(@data & other.to_a)
  end
  
  def -(other)
    SortedSet.new(@data - other.to_a)
  end
  
  def subset?(other)
    @data.all? { |item| other.include?(item) }
  end
  
  def superset?(other)
    other.subset?(self)
  end
  
  def to_s
    "{#{@data.join(', ')}}"
  end
  
  def inspect
    "SortedSet#{to_s}"
  end
end

s1 = SortedSet.new(1, 2, 3, 4, 5)
s2 = SortedSet.new(3, 4, 5, 6, 7)

puts "S1: #{s1}"
puts "S2: #{s2}"
puts "Union: #{(s1 | s2)}"
puts "Intersection: #{(s1 & s2)}"
puts "Difference: #{(s1 - s2)}"

puts "S1 evens: #{s1.select(&:even?).inspect}"
puts "S1 sum: #{s1.sum}"
puts "S1 max: #{s1.max}"

small_set = SortedSet.new(3, 4)
puts "#{small_set} subset of #{s1}? #{small_set.subset?(s1)}"
```

---

## ขั้นตอนที่ 402: ตัวอย่างจริง - Priority Queue

```ruby
# Priority Queue ด้วย Comparable และ Enumerable
class PriorityQueue
  include Enumerable
  
  Item = Struct.new(:value, :priority) do
    include Comparable
    
    def <=>(other)
      other.priority <=> priority  # high priority first
    end
  end
  
  def initialize
    @heap = []
  end
  
  def enqueue(value, priority)
    item = Item.new(value, priority)
    @heap << item
    bubble_up(@heap.size - 1)
    self
  end
  
  def dequeue
    return nil if @heap.empty?
    
    swap(0, @heap.size - 1)
    item = @heap.pop
    sink_down(0) unless @heap.empty?
    item.value
  end
  
  def peek
    @heap.first&.value
  end
  
  def each(&block)
    @heap.sort.each { |item| block.call(item.value) }
  end
  
  def size = @heap.size
  def empty? = @heap.empty?
  
  private
  
  def bubble_up(index)
    while index > 0
      parent = (index - 1) / 2
      break if @heap[parent] <= @heap[index]
      swap(parent, index)
      index = parent
    end
  end
  
  def sink_down(index)
    loop do
      smallest = index
      left  = 2 * index + 1
      right = 2 * index + 2
      
      smallest = left  if left < @heap.size  && @heap[left] < @heap[smallest]
      smallest = right if right < @heap.size && @heap[right] < @heap[smallest]
      
      break if smallest == index
      swap(index, smallest)
      index = smallest
    end
  end
  
  def swap(i, j)
    @heap[i], @heap[j] = @heap[j], @heap[i]
  end
end

pq = PriorityQueue.new
pq.enqueue("งานต่ำ", 1)
pq.enqueue("งานฉุกเฉิน", 10)
pq.enqueue("งานปานกลาง", 5)
pq.enqueue("งานสูง", 8)
pq.enqueue("งานต่ำมาก", 2)

puts "Priority Queue (#{pq.size} items)"
puts "Next: #{pq.peek}"

while !pq.empty?
  puts "Processing: #{pq.dequeue}"
end
```

---

## ขั้นตอนที่ 403: ตัวอย่างจริง - Student Roster

```ruby
class Student
  include Comparable
  
  attr_reader :name, :scores
  
  def initialize(name, *scores)
    @name = name
    @scores = scores
  end
  
  def average
    @scores.sum.to_f / @scores.size
  end
  
  def grade
    case average
    when 90..100 then "A"
    when 80...90 then "B"
    when 70...80 then "C"
    when 60...70 then "D"
    else "F"
    end
  end
  
  def <=>(other)
    average <=> other.average
  end
  
  def to_s
    "#{@name}: avg=#{average.round(1)} (#{grade})"
  end
end

class Roster
  include Enumerable
  
  def initialize(course_name)
    @course_name = course_name
    @students = []
  end
  
  def add(student)
    @students << student
    self
  end
  
  def each(&block)
    @students.each(&block)
  end
  
  def rank
    sort.reverse
  end
  
  def honor_roll
    select { |s| s.grade == "A" }
  end
  
  def failing
    select { |s| s.grade == "F" }
  end
  
  def class_average
    sum(&:average) / count
  end
  
  def grade_distribution
    group_by(&:grade).transform_values(&:count)
  end
  
  def report
    puts "=" * 50
    puts "วิชา: #{@course_name}"
    puts "จำนวนนักเรียน: #{count}"
    puts "คะแนนเฉลี่ยของห้อง: #{class_average.round(1)}"
    puts ""
    puts "อันดับ:"
    rank.each_with_index do |s, i|
      puts "  #{i + 1}. #{s}"
    end
    puts ""
    puts "การกระจายเกรด:"
    grade_distribution.sort.each { |grade, count| puts "  #{grade}: #{count} คน" }
    puts "=" * 50
  end
end

roster = Roster.new("Ruby Programming")

[
  Student.new("สมชาย",   85, 92, 88, 90),
  Student.new("สมหญิง",  72, 78, 80, 75),
  Student.new("สมศรี",   95, 98, 92, 96),
  Student.new("สมบัติ",  55, 60, 58, 62),
  Student.new("สมปอง",   88, 85, 91, 87)
].each { |s| roster.add(s) }

roster.report

puts "\nนักเรียนเกียรตินิยม:"
roster.honor_roll.each { |s| puts "  #{s.name}" }
```

---

## ขั้นตอนที่ 404: ตัวอย่างจริง - Product Catalog

```ruby
class Product
  include Comparable
  
  attr_reader :id, :name, :category, :price, :stock
  
  def initialize(id:, name:, category:, price:, stock:)
    @id = id
    @name = name
    @category = category
    @price = price
    @stock = stock
  end
  
  def <=>(other)
    @price <=> other.price
  end
  
  def available?
    @stock > 0
  end
  
  def discount_price(percent)
    @price * (1 - percent / 100.0)
  end
  
  def to_s
    "#{@name} (฿#{@price})"
  end
  
  def inspect
    "#<Product id=#{@id} name=#{@name} price=#{@price}>"
  end
end

class Catalog
  include Enumerable
  
  def initialize
    @products = []
  end
  
  def add(product)
    @products << product
    self
  end
  
  def each(&block)
    @products.each(&block)
  end
  
  def by_category(cat)
    Catalog.new.tap do |c|
      select { |p| p.category == cat }.each { |p| c.add(p) }
    end
  end
  
  def in_stock
    Catalog.new.tap do |c|
      select(&:available?).each { |p| c.add(p) }
    end
  end
  
  def price_range(min, max)
    Catalog.new.tap do |c|
      select { |p| (min..max).include?(p.price) }.each { |p| c.add(p) }
    end
  end
  
  def sorted_by_price
    sort.reverse
  end
  
  def cheapest(n = 1)
    min_by(n, &:price)
  end
  
  def most_expensive(n = 1)
    max_by(n, &:price)
  end
  
  def categories
    map(&:category).uniq.sort
  end
  
  def total_value
    sum { |p| p.price * p.stock }
  end
  
  def summary
    puts "สินค้าทั้งหมด: #{count} รายการ"
    puts "หมวดหมู่: #{categories.join(', ')}"
    puts "ราคาถูกสุด: #{min}"
    puts "ราคาแพงสุด: #{max}"
    puts "มูลค่ารวม: ฿#{total_value.round(2)}"
  end
end

catalog = Catalog.new
[
  Product.new(id: 1, name: "หนังสือ Ruby", category: "หนังสือ", price: 350, stock: 50),
  Product.new(id: 2, name: "หนังสือ Rails", category: "หนังสือ", price: 400, stock: 30),
  Product.new(id: 3, name: "เมาส์", category: "อิเล็กทรอนิกส์", price: 800, stock: 15),
  Product.new(id: 4, name: "คีย์บอร์ด", category: "อิเล็กทรอนิกส์", price: 1500, stock: 10),
  Product.new(id: 5, name: "กระเป๋า", category: "เสื้อผ้า", price: 600, stock: 0),
  Product.new(id: 6, name: "ปากกา", category: "เครื่องเขียน", price: 50, stock: 200)
].each { |p| catalog.add(p) }

catalog.summary

puts "\nหนังสือทั้งหมด:"
catalog.by_category("หนังสือ").each { |p| puts "  #{p}" }

puts "\nสินค้ามีสต็อก:"
catalog.in_stock.sort.each { |p| puts "  #{p} (stock: #{p.stock})" }

puts "\nสินค้าราคา 300-1000:"
catalog.price_range(300, 1000).each { |p| puts "  #{p}" }
```

---

## ขั้นตอนที่ 405: Implementing Enumerable - เพิ่ม efficiency

```ruby
# เมื่อ implement Enumerable เราสามารถ override บาง methods
# เพื่อเพิ่ม performance

class FastSortedArray
  include Enumerable
  
  def initialize
    @data = []
  end
  
  def <<(item)
    # binary search insertion - O(log n) instead of O(n)
    index = @data.bsearch_index { |x| x >= item } || @data.size
    @data.insert(index, item)
    self
  end
  
  def each(&block)
    @data.each(&block)
  end
  
  # Override sort - already sorted!
  def sort
    @data.dup
  end
  
  def sort_by(&block)
    @data.sort_by(&block)
  end
  
  # Override include? - use binary search O(log n)
  def include?(item)
    index = @data.bsearch_index { |x| x >= item }
    index && @data[index] == item
  end
  
  # Override min/max - O(1) since already sorted
  def min = @data.first
  def max = @data.last
  def first(n = nil) = n ? @data.first(n) : @data.first
  def last(n = nil) = n ? @data.last(n) : @data.last
  
  def size = @data.size
  def empty? = @data.empty?
  
  def to_s = @data.inspect
end

fsa = FastSortedArray.new
[5, 2, 8, 1, 9, 3, 7, 4, 6].each { |n| fsa << n }

puts fsa.to_s          # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
puts fsa.min           # => 1 (O(1))
puts fsa.max           # => 9 (O(1))
puts fsa.include?(7)   # => true (O(log n))
puts fsa.include?(10)  # => false

# Enumerable methods ทำงานได้ปกติ
puts fsa.select(&:even?).inspect  # => [2, 4, 6, 8]
puts fsa.sum                       # => 45
```

---

## ขั้นตอนที่ 406: ตัวอย่าง DSL ด้วย Comparable และ Enumerable

```ruby
# DSL สำหรับ query data
class Dataset
  include Enumerable
  
  def initialize(data = [])
    @data = data
  end
  
  def each(&block)
    @data.each(&block)
  end
  
  def where(**conditions, &block)
    filtered = select do |item|
      conditions_met = conditions.all? do |key, value|
        case value
        when Proc   then value.call(item[key])
        when Range  then value.include?(item[key])
        else item[key] == value
        end
      end
      block_given? ? (conditions_met && block.call(item)) : conditions_met
    end
    Dataset.new(filtered)
  end
  
  def order_by(*keys, desc: false)
    sorted = sort_by { |item| keys.map { |k| item[k] } }
    Dataset.new(desc ? sorted.reverse : sorted)
  end
  
  def limit(n)
    Dataset.new(first(n))
  end
  
  def pluck(*keys)
    map { |item| keys.size == 1 ? item[keys.first] : keys.map { |k| item[k] } }
  end
  
  def count_by(key)
    group_by { |item| item[key] }.transform_values(&:count)
  end
  
  def average(key)
    sum { |item| item[key] }.to_f / count
  end
end

# ทดสอบ
employees = Dataset.new([
  { name: "ก", dept: "IT", salary: 80000, age: 28 },
  { name: "ข", dept: "HR", salary: 60000, age: 35 },
  { name: "ค", dept: "IT", salary: 95000, age: 32 },
  { name: "ง", dept: "Sales", salary: 70000, age: 27 },
  { name: "จ", dept: "IT", salary: 75000, age: 24 },
  { name: "ฉ", dept: "HR", salary: 65000, age: 42 }
])

# Query DSL
puts "พนักงาน IT:"
employees.where(dept: "IT")
  .order_by(:salary, desc: true)
  .pluck(:name, :salary)
  .each { |name, sal| puts "  #{name}: ฿#{sal}" }

puts "\nพนักงานอายุ 25-35:"
employees.where(age: (25..35))
  .pluck(:name, :age)
  .each { |name, age| puts "  #{name}: #{age} ปี" }

puts "\nเงินเดือนเฉลี่ย IT: ฿#{employees.where(dept: "IT").average(:salary).round(0)}"
puts "\nจำนวนพนักงานแต่ละแผนก:"
employees.count_by(:dept).each { |dept, count| puts "  #{dept}: #{count} คน" }
```

---

## ขั้นตอนที่ 407: Design Patterns กับ Comparable

```ruby
# Strategy Pattern กับ Comparable
module SortStrategy
  module ByName
    def <=>(other)
      name <=> other.name
    end
  end
  
  module ByPrice
    def <=>(other)
      price <=> other.price
    end
  end
  
  module ByRating
    def <=>(other)
      other.rating <=> rating  # descending
    end
  end
end

class ProductWithStrategy
  attr_reader :name, :price, :rating
  
  def initialize(name, price, rating)
    @name = name
    @price = price
    @rating = rating
  end
  
  def sort_by_name
    self.class.prepend(SortStrategy::ByName)
    self
  end
  
  def to_s
    "#{name} (฿#{price}, ★#{rating})"
  end
end

# Decorator Pattern
class SortableCollection
  include Enumerable
  
  def initialize(items, &comparator)
    @items = items
    @comparator = comparator
  end
  
  def each(&block)
    @items.each(&block)
  end
  
  def sort
    @comparator ? @items.sort(&@comparator) : @items.sort
  end
  
  def sort_desc
    @comparator ? @items.sort(&@comparator).reverse : @items.sort.reverse
  end
end

books = [
  { title: "Ruby", price: 350, rating: 4.8 },
  { title: "Rails", price: 400, rating: 4.5 },
  { title: "Python", price: 300, rating: 4.6 }
]

by_price = SortableCollection.new(books) { |a, b| a[:price] <=> b[:price] }
puts "By price:"
by_price.sort.each { |b| puts "  #{b[:title]}: ฿#{b[:price]}" }

by_rating = SortableCollection.new(books) { |a, b| b[:rating] <=> a[:rating] }
puts "By rating (desc):"
by_rating.sort.each { |b| puts "  #{b[:title]}: ★#{b[:rating]}" }
```

---

## ขั้นตอนที่ 408: Testing Custom Comparable/Enumerable

```ruby
# ตัวอย่างการ test ด้วย minitest
require 'minitest/autorun'

class Money
  include Comparable
  
  attr_reader :amount, :currency
  
  RATES = { USD: 1.0, THB: 35.0, EUR: 1.1 }.freeze
  
  def initialize(amount, currency = :USD)
    @amount = amount.to_f
    @currency = currency.to_sym
  end
  
  def to_usd
    @amount / RATES[@currency]
  end
  
  def <=>(other)
    to_usd <=> other.to_usd
  end
  
  def +(other)
    Money.new(to_usd + other.to_usd, :USD)
  end
  
  def *(factor)
    Money.new(@amount * factor, @currency)
  end
  
  def to_s
    "#{@currency} #{@amount.round(2)}"
  end
end

class TestMoney < Minitest::Test
  def test_comparison
    usd_10 = Money.new(10, :USD)
    thb_350 = Money.new(350, :THB)
    
    assert_equal 0, (usd_10 <=> thb_350)
    assert usd_10 == thb_350
  end
  
  def test_ordering
    prices = [
      Money.new(100, :THB),
      Money.new(5, :USD),
      Money.new(3, :EUR)
    ]
    
    sorted = prices.sort
    assert sorted.first < sorted.last
  end
  
  def test_between
    price = Money.new(200, :THB)
    low   = Money.new(100, :THB)
    high  = Money.new(300, :THB)
    
    assert price.between?(low, high)
  end
end
```

---

## ขั้นตอนที่ 409: Advanced - Comparable ใน Real Applications

```ruby
# Task management system
class Task
  include Comparable
  
  PRIORITIES = { low: 1, medium: 2, high: 3, critical: 4 }.freeze
  
  attr_reader :id, :title, :priority, :due_date, :created_at
  attr_accessor :completed
  
  def initialize(id, title, priority: :medium, due_date: nil)
    @id = id
    @title = title
    @priority = priority
    @due_date = due_date
    @created_at = Time.now
    @completed = false
  end
  
  def <=>(other)
    # Compare by priority first (higher = more urgent)
    prio_compare = PRIORITIES[other.priority] <=> PRIORITIES[@priority]
    return prio_compare unless prio_compare == 0
    
    # Then by due date (earlier = more urgent)
    if @due_date && other.due_date
      @due_date <=> other.due_date
    elsif @due_date
      -1  # has due date is more urgent
    elsif other.due_date
      1
    else
      @created_at <=> other.created_at
    end
  end
  
  def urgent?
    PRIORITIES[@priority] >= PRIORITIES[:high]
  end
  
  def overdue?
    @due_date && @due_date < Date.today && !@completed
  end
  
  def to_s
    status = @completed ? "[✓]" : "[ ]"
    due = @due_date ? " (due: #{@due_date})" : ""
    "#{status} [#{@priority.upcase}] #{@title}#{due}"
  end
end

class TaskManager
  include Enumerable
  
  def initialize
    @tasks = []
    @next_id = 1
  end
  
  def add(title, **options)
    task = Task.new(@next_id, title, **options)
    @tasks << task
    @next_id += 1
    task
  end
  
  def each(&block)
    @tasks.each(&block)
  end
  
  def pending
    reject(&:completed).sort
  end
  
  def urgent_tasks
    pending.select(&:urgent?)
  end
  
  def complete(id)
    find { |t| t.id == id }&.tap { |t| t.completed = true }
  end
  
  def summary
    total = count
    done  = count(&:completed)
    puts "งานทั้งหมด: #{total} (เสร็จ: #{done}, คงเหลือ: #{total - done})"
  end
end

require 'date'
manager = TaskManager.new

manager.add("เขียนรายงาน", priority: :high, due_date: Date.today + 2)
manager.add("ประชุมทีม", priority: :medium, due_date: Date.today + 1)
manager.add("ส่ง email", priority: :low)
manager.add("Fix critical bug", priority: :critical, due_date: Date.today)
manager.add("Code review", priority: :medium)

puts "งานที่ต้องทำ (เรียงตามความสำคัญ):"
manager.pending.each { |t| puts "  #{t}" }

puts "\nงานด่วน:"
manager.urgent_tasks.each { |t| puts "  #{t}" }

manager.complete(3)
puts "\nหลังเสร็จงาน:"
manager.summary
```

---

## ขั้นตอนที่ 410: สรุปและ Best Practices

```ruby
# Best practices สำหรับ Comparable และ Enumerable

# 1. Comparable: เพิ่ม guard สำหรับ comparison กับ type อื่น
class SafeComparable
  include Comparable
  
  attr_reader :value
  
  def initialize(value)
    @value = value
  end
  
  def <=>(other)
    return nil unless other.is_a?(self.class)
    @value <=> other.value
  end
end

# 2. Enumerable: พิจารณา override methods สำหรับ performance
class OptimizedCollection
  include Enumerable
  
  def initialize(data)
    @data = data.sort  # keep sorted
  end
  
  def each(&block) = @data.each(&block)
  
  # Override สำหรับ O(1) แทน O(n)
  def min = @data.first
  def max = @data.last
  
  # Override สำหรับ O(log n) แทน O(n)
  def include?(item)
    idx = @data.bsearch_index { |x| x >= item }
    idx && @data[idx] == item
  end
  
  # Override sort - already sorted
  def sort = @data.dup
  def sort_by(&block) = @data.sort_by(&block)
end

# 3. Lazy evaluation สำหรับ infinite collections
class InfiniteCounter
  include Enumerable
  
  def each
    return to_enum(:each) unless block_given?
    n = 0
    loop { yield n; n += 1 }
  end
  
  # ต้องใช้ lazy!
  def first(n)
    lazy.first(n)
  end
  
  def take(n)
    lazy.first(n)
  end
end

counter = InfiniteCounter.new
puts counter.first(5).inspect  # => [0, 1, 2, 3, 4]
puts counter.lazy.select(&:even?).first(5).inspect  # => [0, 2, 4, 6, 8]
```

---

## แบบฝึกหัด 20 ข้อ

### ข้อ 1-5: Comparable

**ข้อ 1:** สร้าง Rectangle class ที่ implement Comparable ตาม area
```ruby
# เฉลย
class Rectangle
  include Comparable
  attr_reader :width, :height
  
  def initialize(width, height)
    @width = width
    @height = height
  end
  
  def area = @width * @height
  def perimeter = 2 * (@width + @height)
  
  def <=>(other) = area <=> other.area
  
  def to_s = "#{@width}x#{@height} (area: #{area})"
end

rects = [Rectangle.new(3, 4), Rectangle.new(2, 10), Rectangle.new(5, 5)]
puts rects.sort.map(&:to_s).inspect
puts "Largest: #{rects.max}"
```

**ข้อ 2:** สร้าง Weight class รองรับหลาย units
```ruby
# เฉลย
class Weight
  include Comparable
  
  CONVERSIONS = { kg: 1.0, g: 0.001, lb: 0.453592, oz: 0.0283495 }
  
  def initialize(amount, unit = :kg)
    @amount = amount
    @unit = unit
  end
  
  def in_kg = @amount * CONVERSIONS[@unit]
  
  def <=>(other) = in_kg <=> other.in_kg
  
  def to_s = "#{@amount}#{@unit}"
  def inspect = "Weight(#{to_s} = #{in_kg.round(3)}kg)"
end

weights = [Weight.new(1000, :g), Weight.new(2, :lb), Weight.new(0.5, :kg)]
puts weights.sort.map(&:inspect).inspect
puts "Heaviest: #{weights.max}"
```

**ข้อ 3:** สร้าง Priority class สำหรับ job scheduling
```ruby
# เฉลย
class Job
  include Comparable
  
  LEVELS = { low: 0, normal: 1, high: 2, urgent: 3 }
  
  attr_reader :name, :priority, :submitted_at
  
  def initialize(name, priority = :normal)
    @name = name
    @priority = priority
    @submitted_at = Time.now
  end
  
  def <=>(other)
    result = LEVELS[other.priority] <=> LEVELS[@priority]
    result == 0 ? @submitted_at <=> other.submitted_at : result
  end
  
  def to_s = "[#{@priority.upcase}] #{@name}"
end

jobs = [
  Job.new("Print document"),
  Job.new("System backup", :high),
  Job.new("Update software", :urgent),
  Job.new("Email check"),
  Job.new("DB Maintenance", :high)
]

puts "Job Queue (priority order):"
jobs.sort.each { |j| puts "  #{j}" }
```

**ข้อ 4:** Comparable กับ Date
```ruby
# เฉลย
require 'date'

class Event
  include Comparable
  
  attr_reader :name, :date, :priority
  
  def initialize(name, date, priority = :normal)
    @name = name
    @date = date
    @priority = priority
  end
  
  def <=>(other)
    result = @date <=> other.date
    result == 0 ? (other.priority == :urgent ? 1 : -1) : result
  end
  
  def upcoming?
    @date >= Date.today
  end
  
  def to_s = "#{@name} on #{@date}"
end

events = [
  Event.new("Meeting", Date.today + 3),
  Event.new("Deadline", Date.today + 1, :urgent),
  Event.new("Party", Date.today + 5),
  Event.new("Review", Date.today + 1)
]

puts "Events sorted:"
events.sort.each { |e| puts "  #{e}" }
```

**ข้อ 5:** Custom Comparable สำหรับ IP Address
```ruby
# เฉลย
class IPAddress
  include Comparable
  
  def initialize(ip_string)
    @octets = ip_string.split('.').map(&:to_i)
    raise ArgumentError, "Invalid IP" unless @octets.size == 4 &&
      @octets.all? { |o| (0..255).include?(o) }
  end
  
  def <=>(other) = @octets <=> other.octets
  
  def to_s = @octets.join('.')
  
  def to_i
    @octets.reduce { |acc, o| acc * 256 + o }
  end
  
  def in_subnet?(network, mask)
    net = IPAddress.new(network)
    (to_i & mask_to_i(mask)) == (net.to_i & mask_to_i(mask))
  end
  
  protected
  def octets = @octets
  
  private
  def mask_to_i(bits)
    ((2**bits - 1) << (32 - bits))
  end
end

ips = ["192.168.1.10", "10.0.0.1", "192.168.1.1", "172.16.0.1"].map { |ip| IPAddress.new(ip) }
puts ips.sort.map(&:to_s).inspect
puts "Smallest: #{ips.min}"
puts "Largest: #{ips.max}"
```

### ข้อ 6-10: Enumerable

**ข้อ 6:** สร้าง Deck of Cards
```ruby
# เฉลย
class Deck
  include Enumerable
  
  SUITS = [:spades, :hearts, :diamonds, :clubs]
  VALUES = (2..10).to_a + [:J, :Q, :K, :A]
  
  Card = Struct.new(:value, :suit) do
    def to_s = "#{value}#{suit.to_s[0].upcase}"
    def numeric_value = VALUES.index(value) + 2
  end
  
  def initialize
    @cards = SUITS.flat_map { |suit| VALUES.map { |val| Card.new(val, suit) } }
    shuffle!
  end
  
  def each(&block) = @cards.each(&block)
  
  def shuffle! = @cards.shuffle!
  
  def deal(n) = @cards.pop(n)
  
  def size = @cards.size
end

deck = Deck.new
puts "Total cards: #{deck.count}"
hand = deck.deal(5)
puts "Your hand: #{hand.map(&:to_s).join(', ')}"
puts "Remaining: #{deck.size}"
puts "Unique suits: #{deck.map(&:suit).uniq.count}"
```

**ข้อ 7:** File System Enumerable
```ruby
# เฉลย
class FileCollection
  include Enumerable
  
  FileInfo = Struct.new(:name, :size, :type) do
    def to_s = "#{name} (#{type}, #{size}B)"
  end
  
  def initialize
    @files = []
  end
  
  def add(name, size, type)
    @files << FileInfo.new(name, size, type)
    self
  end
  
  def each(&block) = @files.each(&block)
  
  def by_type(type) = select { |f| f.type == type }
  def total_size = sum(&:size)
  def largest(n = 1) = max_by(n, &:size)
end

fs = FileCollection.new
fs.add("main.rb", 1024, :ruby)
  .add("helper.rb", 512, :ruby)
  .add("config.json", 256, :json)
  .add("readme.md", 4096, :markdown)
  .add("test.rb", 2048, :ruby)

puts "Ruby files: #{fs.by_type(:ruby).map(&:name).inspect}"
puts "Total size: #{fs.total_size} bytes"
puts "Largest: #{fs.largest(2).map(&:name).inspect}"
puts "By type: #{fs.group_by(&:type).transform_values(&:count).inspect}"
```

**ข้อ 8-10:** (ต่อ)

```ruby
# ข้อ 8: Matrix class กับ Enumerable
class Matrix
  include Enumerable
  
  def initialize(rows)
    @rows = rows
  end
  
  def each(&block)
    @rows.each { |row| row.each(&block) }
  end
  
  def [](i, j) = @rows[i][j]
  def rows = @rows.size
  def cols = @rows.first&.size || 0
  
  def row(i) = @rows[i]
  def col(j) = @rows.map { |row| row[j] }
  
  def transpose
    Matrix.new(@rows.transpose)
  end
  
  def *(other)
    result = Array.new(rows) { Array.new(other.cols, 0) }
    rows.times do |i|
      other.cols.times do |j|
        cols.times { |k| result[i][j] += self[i, k] * other[k, j] }
      end
    end
    Matrix.new(result)
  end
  
  def to_s
    @rows.map { |row| row.map { |n| n.to_s.rjust(4) }.join }.join("\n")
  end
end

m1 = Matrix.new([[1, 2, 3], [4, 5, 6]])
puts "All elements: #{m1.to_a.inspect}"
puts "Sum: #{m1.sum}"
puts "Max: #{m1.max}"
puts "Evens: #{m1.select(&:even?).inspect}"
puts "\nMatrix:\n#{m1}"

# ข้อ 9: Calendar กับ Enumerable
require 'date'

class Calendar
  include Enumerable
  
  def initialize(year, month)
    @start = Date.new(year, month, 1)
    @end   = Date.new(year, month, -1)
  end
  
  def each(&block) = (@start..@end).each(&block)
  
  def weekdays = select { |d| (1..5).include?(d.wday) }
  def weekends = select { |d| [0, 6].include?(d.wday) }
  
  def mondays  = select(&:monday?)
  def fridays  = select(&:friday?)
  
  def to_display
    days_of_week = %w[Su Mo Tu We Th Fr Sa]
    puts days_of_week.join(' ')
    
    first_day = @start.wday
    cells = Array.new(first_day, '  ')
    cells += map { |d| d.day.to_s.rjust(2) }
    
    while cells.size % 7 != 0
      cells << '  '
    end
    
    cells.each_slice(7) { |week| puts week.join(' ') }
  end
end

jan = Calendar.new(2024, 1)
puts "วันทำงานใน Jan 2024: #{jan.weekdays.count}"
puts "วันหยุดสุดสัปดาห์: #{jan.weekends.count}"
jan.to_display

# ข้อ 10: Graph กับ Enumerable
class Graph
  include Enumerable
  
  def initialize
    @edges = Hash.new { |h, k| h[k] = [] }
    @nodes = Set.new
  end
  
  def add_edge(from, to, weight: 1)
    @edges[from] << { node: to, weight: weight }
    @nodes.add(from)
    @nodes.add(to)
    self
  end
  
  def each(&block) = @nodes.each(&block)
  
  def neighbors(node) = @edges[node].map { |e| e[:node] }
  
  def bfs(start)
    visited = []
    queue = [start]
    
    until queue.empty?
      node = queue.shift
      next if visited.include?(node)
      visited << node
      queue.concat(neighbors(node))
    end
    
    visited
  end
  
  def connected?
    return true if empty?
    bfs(first).sort == sort.to_a.sort
  end
end

require 'set'
g = Graph.new
g.add_edge("A", "B").add_edge("B", "C").add_edge("A", "C").add_edge("C", "D")

puts "Nodes: #{g.sort.inspect}"
puts "BFS from A: #{g.bfs("A").inspect}"
puts "Connected: #{g.connected?}"
puts "Node count: #{g.count}"
```

### ข้อ 11-20: Applications

```ruby
# ข้อ 11: Comparable Employee
class Employee
  include Comparable
  
  attr_reader :name, :department, :salary, :years
  
  def initialize(name, department, salary, years)
    @name = name
    @department = department
    @salary = salary
    @years = years
  end
  
  def seniority_score = @salary * (1 + @years * 0.1)
  
  def <=>(other) = seniority_score <=> other.seniority_score
  
  def to_s = "#{@name} (#{@department}, ฿#{@salary}, #{@years}y)"
end

employees = [
  Employee.new("ก", "IT", 80000, 5),
  Employee.new("ข", "HR", 60000, 8),
  Employee.new("ค", "IT", 95000, 3),
  Employee.new("ง", "Sales", 70000, 7)
]

puts "Senior ranking:"
employees.sort.reverse.each { |e| puts "  #{e}" }

# ข้อ 12-20: (ฝึกเอง)
# สร้าง Library ที่รองรับ Comparable และ Enumerable
class Book
  include Comparable
  
  attr_reader :title, :author, :pages, :year, :isbn
  
  def initialize(title:, author:, pages:, year:, isbn:)
    @title  = title
    @author = author
    @pages  = pages
    @year   = year
    @isbn   = isbn
  end
  
  def <=>(other) = @title <=> other.title
  def to_s = "#{@title} by #{@author} (#{@year}, #{@pages}p)"
end

class Library
  include Enumerable
  
  def initialize(name)
    @name = name
    @books = []
  end
  
  def add(book) = @books << book and self
  
  def each(&block) = @books.each(&block)
  
  def search(query)
    select { |b| b.title.downcase.include?(query.downcase) || 
                 b.author.downcase.include?(query.downcase) }
  end
  
  def by_author(name) = select { |b| b.author == name }
  def by_year(year) = select { |b| b.year == year }
  def newest = max_by(&:year)
  def oldest = min_by(&:year)
  def longest = max_by(&:pages)
  def total_pages = sum(&:pages)
  
  def catalog
    puts "#{@name} - #{count} books"
    sort.each { |b| puts "  #{b}" }
  end
end

lib = Library.new("Ruby Library")
lib.add(Book.new(title: "Ruby Programming", author: "Matz", pages: 400, year: 2015, isbn: "111"))
   .add(Book.new(title: "Rails Guides", author: "DHH", pages: 600, year: 2020, isbn: "222"))
   .add(Book.new(title: "Clean Code", author: "Martin", pages: 464, year: 2008, isbn: "333"))
   .add(Book.new(title: "Design Patterns", author: "GoF", pages: 395, year: 1994, isbn: "444"))

lib.catalog
puts "\nSearch 'ruby': #{lib.search('ruby').map(&:title).inspect}"
puts "Newest: #{lib.newest.title}"
puts "Longest: #{lib.longest.title}"
puts "Total pages: #{lib.total_pages}"
```

---

## สรุปบทที่ 19

ในบทนี้เราได้เรียนรู้:

**Comparable Module:**
- implement `<=>` (spaceship operator) เพื่อได้ `>`, `>=`, `<`, `<=`, `between?`, `clamp` ฟรี
- ใช้สำหรับ custom comparison logic
- override เพื่อ compare ด้วย multiple criteria

**Custom Enumerable:**
- implement `each` เพื่อได้ methods กว่า 50 ตัวฟรี
- สร้าง collection classes ที่มีความสามารถเทียบเท่า Array
- override methods เพื่อ optimize performance

**Best Practices:**
- Comparable: guard สำหรับ type mismatch
- Enumerable: override min/max/include? สำหรับ sorted data
- ใช้ lazy เมื่อ collection อาจ infinite หรือ large
- combine ทั้งสอง modules สำหรับ powerful collections
