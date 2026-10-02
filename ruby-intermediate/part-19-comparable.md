# ตอนที่ 19: Comparable และ Enumerable Modules (Steps 391-410)

## บทนำ

ใน Ruby มี modules สำคัญสองตัวที่ช่วยให้ class ของเราได้รับความสามารถเพิ่มเติมโดยไม่ต้องเขียนโค้ดมาก ได้แก่:

- **Comparable**: ให้ความสามารถในการเปรียบเทียบ (`<`, `<=`, `==`, `>=`, `>`, `between?`, `clamp`)
- **Enumerable**: ให้ความสามารถในการวนซ้ำและ transform collection (กว่า 50 methods)

ตอนนี้เราจะเรียนรู้วิธี implement ทั้งสองอย่างในสมบูรณ์

---

## Step 391: Comparable Module คืออะไร?

**Comparable** module ให้ methods สำหรับการเปรียบเทียบ โดยต้องการแค่ implement `<=>` (spaceship operator) หนึ่ง method เท่านั้น

```ruby
# Comparable module ถูก include ใน Integer, String, Float, ฯลฯ
puts Integer.include?(Comparable)  # true
puts String.include?(Comparable)   # true
puts Float.include?(Comparable)    # true

# ดู methods ที่ Comparable ให้
puts Comparable.instance_methods.sort.inspect
# [:between?, :clamp, :<=>, :<, :<=, :==, :>, :>=]

# ทดสอบกับ Integer
puts 3 > 1    # true
puts 3 >= 3   # true
puts 3 < 5    # true
puts 3 <= 3   # true
puts 3 == 3   # true
puts 3.between?(1, 5)    # true
puts 3.clamp(1, 5)       # 3
puts 10.clamp(1, 5)      # 5 (clamp ไว้ที่ max)
puts (-5).clamp(1, 5)    # 1 (clamp ไว้ที่ min)
```

---

## Step 392: Implementing <=> (Spaceship Operator)

`<=>` คือหัวใจของ Comparable - ต้องคืนค่า -1, 0, หรือ 1:

```ruby
# <=> rules:
# คืน -1 ถ้า self < other
# คืน  0 ถ้า self == other
# คืน  1 ถ้า self > other
# คืน nil ถ้าเปรียบเทียบกันไม่ได้

# ตัวอย่าง Integer
puts (1 <=> 2)  # -1
puts (2 <=> 2)  # 0
puts (3 <=> 2)  # 1
puts (1 <=> "hello")  # nil (ไม่สามารถเปรียบเทียบได้)

# ตัวอย่าง String
puts ("apple" <=> "banana")  # -1
puts ("banana" <=> "banana") # 0
puts ("cherry" <=> "banana") # 1

# สร้าง class ที่ใช้ Comparable
class Temperature
  include Comparable

  attr_reader :degrees, :unit

  def initialize(degrees, unit = :celsius)
    @degrees = degrees
    @unit = unit
  end

  def to_celsius
    case @unit
    when :celsius    then @degrees
    when :fahrenheit then (@degrees - 32) * 5.0 / 9
    when :kelvin     then @degrees - 273.15
    end
  end

  def <=>(other)
    to_celsius <=> other.to_celsius
  end

  def to_s
    "#{@degrees}°#{@unit.to_s.upcase[0]}"
  end
end

t1 = Temperature.new(100, :celsius)
t2 = Temperature.new(212, :fahrenheit)  # 100°C
t3 = Temperature.new(373.15, :kelvin)   # 100°C
t4 = Temperature.new(37, :celsius)

puts t1 == t2  # true (ทั้งคู่คือ 100°C)
puts t1 == t3  # true
puts t4 < t1   # true (37 < 100)
puts t1 > t4   # true

puts [t1, t4, t2].sort
# 37°C, 100°C, 100°F (หรือ Kelvin)
```

---

## Step 393: ได้รับอะไรจาก Comparable

เมื่อ include Comparable และ implement `<=>` คุณได้รับ methods เหล่านี้ฟรี:

```ruby
class Score
  include Comparable

  attr_reader :value, :subject

  def initialize(value, subject)
    @value = value
    @subject = subject
  end

  def <=>(other)
    @value <=> other.value
  end

  def to_s
    "#{@subject}: #{@value}"
  end
end

math = Score.new(85, "Math")
english = Score.new(92, "English")
science = Score.new(78, "Science")
history = Score.new(85, "History")

# < <= == >= >
puts math < english    # true  (85 < 92)
puts math <= history   # true  (85 <= 85)
puts math == history   # true  (85 == 85)
puts english >= math   # true  (92 >= 85)
puts english > science # true  (92 > 78)

# between?
puts math.between?(science, english)  # true  (78 <= 85 <= 92)
puts science.between?(math, english)  # false (78 < 78 is false)

# clamp
score = Score.new(105, "Extra")  # คะแนนเกิน 100
clamped = score.clamp(Score.new(0, "min"), Score.new(100, "max"))
puts clamped.value  # 100

# sort - ใช้ <=> โดยอัตโนมัติ
scores = [math, english, science, history]
puts scores.sort  # เรียงตาม value
puts scores.min   # science (78)
puts scores.max   # english (92)
```

---

## Step 394: Building Sortable Classes

```ruby
# ตัวอย่างจริง: Product class ที่เปรียบเทียบได้
class Product
  include Comparable

  attr_accessor :name, :price, :rating, :stock

  def initialize(name, price, rating, stock)
    @name   = name
    @price  = price
    @rating = rating
    @stock  = stock
  end

  # เปรียบเทียบตาม price เป็นหลัก
  def <=>(other)
    result = @price <=> other.price
    # ถ้า price เท่ากัน เปรียบเทียบตาม rating (มากกว่า = ดีกว่า)
    result == 0 ? other.rating <=> @rating : result
  end

  def to_s
    "#{@name} ($#{@price}, #{@rating}★, stock: #{@stock})"
  end
end

products = [
  Product.new("MacBook Pro",   1299, 4.8, 15),
  Product.new("Dell XPS",      999,  4.5, 20),
  Product.new("HP Spectre",    999,  4.7, 8),
  Product.new("Lenovo ThinkPad", 849, 4.6, 25),
  Product.new("ASUS ZenBook",  1099, 4.4, 12)
]

puts "=== Sorted by Price (then Rating) ==="
products.sort.each { |p| puts p }

puts "\n=== Cheapest ==="
puts products.min

puts "\n=== Most Expensive ==="
puts products.max

puts "\n=== Products $900-$1100 range ==="
budget_range = Product.new("min", 900, 0, 0)..Product.new("max", 1100, 5, 0)
puts products.select { |p| budget_range.cover?(p) }
```

---

## Step 395: ตัวอย่าง Comparable กับ Version Numbers

```ruby
class Version
  include Comparable

  attr_reader :major, :minor, :patch

  def initialize(version_string)
    parts = version_string.to_s.split('.').map(&:to_i)
    @major = parts[0] || 0
    @minor = parts[1] || 0
    @patch = parts[2] || 0
  end

  def <=>(other)
    return @major <=> other.major unless @major == other.major
    return @minor <=> other.minor unless @minor == other.minor
    @patch <=> other.patch
  end

  def to_s
    "#{@major}.#{@minor}.#{@patch}"
  end

  def inspect
    "Version(#{to_s})"
  end
end

v1 = Version.new("1.0.0")
v2 = Version.new("1.2.0")
v3 = Version.new("2.0.0")
v4 = Version.new("1.2.3")
v5 = Version.new("1.2.0")

puts v1 < v2   # true
puts v3 > v2   # true
puts v2 == v5  # true
puts v4.between?(v2, v3)  # true (1.2.3 is between 1.2.0 and 2.0.0)

versions = [v3, v1, v4, v2, v5]
puts versions.sort.inspect
# [Version(1.0.0), Version(1.2.0), Version(1.2.0), Version(1.2.3), Version(2.0.0)]

# ตรวจสอบ minimum required version
required = Version.new("1.1.0")
installed = Version.new("1.2.3")
puts installed >= required ? "Compatible!" : "Upgrade needed!"
```

---

## Step 396: ตัวอย่าง Comparable - Money Class

```ruby
class Money
  include Comparable

  attr_reader :amount, :currency

  EXCHANGE_RATES = {
    USD: 1.0,
    EUR: 1.08,
    GBP: 1.27,
    THB: 0.028,
    JPY: 0.0067
  }.freeze

  def initialize(amount, currency = :USD)
    @amount   = amount.to_f
    @currency = currency.to_sym
  end

  def to_usd
    @amount * EXCHANGE_RATES[@currency]
  end

  def <=>(other)
    to_usd <=> other.to_usd
  end

  def +(other)
    usd_total = to_usd + other.to_usd
    Money.new(usd_total / EXCHANGE_RATES[@currency], @currency)
  end

  def *(factor)
    Money.new(@amount * factor, @currency)
  end

  def to_s
    "#{format('%.2f', @amount)} #{@currency}"
  end
end

prices = [
  Money.new(50, :USD),
  Money.new(4500, :THB),
  Money.new(45, :EUR),
  Money.new(7000, :JPY),
  Money.new(35, :GBP)
]

puts "=== Prices sorted by value ==="
prices.sort.each { |p| puts "#{p} = #{format('%.2f', p.to_usd)} USD" }

puts "\nCheapest: #{prices.min}"
puts "Most expensive: #{prices.max}"

usd_50 = Money.new(50, :USD)
thb_range_low  = Money.new(1500, :THB)
thb_range_high = Money.new(2000, :THB)
puts "\n50 USD between 1500-2000 THB? #{usd_50.between?(thb_range_low, thb_range_high)}"
```

---

## Step 397: Enumerable Module - Implementation

```ruby
# เพื่อ implement Enumerable คุณต้องการแค่ each method
class NumberList
  include Enumerable

  def initialize(*numbers)
    @data = numbers
  end

  def each
    @data.each { |n| yield n }
  end

  # Optional: เพิ่ม convenience methods
  def +(other)
    NumberList.new(*(@data + other.to_a))
  end

  def to_s
    "NumberList[#{@data.join(', ')}]"
  end
end

list = NumberList.new(5, 2, 8, 1, 9, 3, 7, 4, 6)

# ได้ methods ทั้งหมดจาก Enumerable ฟรี
puts list.sort.inspect        # [1, 2, 3, 4, 5, 6, 7, 8, 9]
puts list.min                 # 1
puts list.max                 # 9
puts list.sum                 # 45
puts list.count               # 9
puts list.include?(7)         # true
puts list.select(&:even?).inspect  # [2, 8, 4, 6]
puts list.map { |n| n * 2 }.inspect
puts list.sort_by { |n| -n }.inspect  # descending
puts list.any? { |n| n > 8 }  # true
puts list.all? { |n| n > 0 }  # true
puts list.none? { |n| n < 0 } # true
puts list.group_by(&:odd?).inspect
puts list.first(3).inspect    # [5, 2, 8]
puts list.min_by { |n| n % 3 }  # first element with smallest remainder
```

---

## Step 398: Building Custom Collection Class - Part 1

```ruby
# ตัวอย่างที่ครบสมบูรณ์: Catalog class สำหรับร้านค้า
class ProductCatalog
  include Enumerable
  include Comparable  # เปรียบเทียบ catalog กันได้

  attr_reader :name, :items

  def initialize(name)
    @name  = name
    @items = []
  end

  def add(product)
    @items << product
    self
  end

  def remove(product_name)
    @items.reject! { |p| p[:name] == product_name }
    self
  end

  # Enumerable requirement
  def each(&block)
    @items.each(&block)
  end

  # Comparable requirement (เปรียบเทียบตามจำนวน item)
  def <=>(other)
    count <=> other.count
  end

  # Additional methods
  def total_value
    sum { |p| p[:price] * p[:stock] }
  end

  def available
    select { |p| p[:stock] > 0 }
  end

  def search(query)
    select { |p| p[:name].downcase.include?(query.downcase) }
  end

  def by_category(category)
    select { |p| p[:category] == category }
  end

  def price_range(min, max)
    select { |p| p[:price].between?(min, max) }
  end

  def to_s
    "Catalog: #{@name} (#{count} items, value: #{total_value})"
  end
end

catalog = ProductCatalog.new("Electronics")
catalog
  .add({ name: "iPhone 15", price: 32000, stock: 50, category: :phone })
  .add({ name: "Samsung S24", price: 28000, stock: 30, category: :phone })
  .add({ name: "MacBook Air", price: 42000, stock: 15, category: :laptop })
  .add({ name: "iPad Pro", price: 25000, stock: 20, category: :tablet })
  .add({ name: "AirPods Pro", price: 8500, stock: 100, category: :audio })
  .add({ name: "Sony WH-1000XM5", price: 12000, stock: 0, category: :audio })

puts catalog
puts "Available: #{catalog.available.count} items"
puts "Phones: #{catalog.by_category(:phone).map { |p| p[:name] }.inspect}"
puts "10k-30k range: #{catalog.price_range(10000, 30000).map { |p| p[:name] }.inspect}"
puts "Search 'air': #{catalog.search('air').map { |p| p[:name] }.inspect}"
puts "Most expensive: #{catalog.max_by { |p| p[:price] }[:name]}"
puts "Sorted by price: #{catalog.sort_by { |p| p[:price] }.map { |p| p[:name] }.inspect}"
```

---

## Step 399: Building Custom Collection Class - Part 2

```ruby
# ตัวอย่าง: EventCalendar class
require 'date'

class EventCalendar
  include Enumerable

  Event = Struct.new(:name, :date, :duration_hours, :attendees) do
    def to_s
      "#{name} (#{date.strftime('%Y-%m-%d')}, #{duration_hours}h, #{attendees} attendees)"
    end
  end

  def initialize
    @events = []
  end

  def schedule(name, date, duration_hours = 1, attendees = 0)
    event = Event.new(name, date, duration_hours, attendees)
    @events << event
    @events.sort_by!(&:date)
    self
  end

  def each(&block)
    @events.each(&block)
  end

  def today
    today = Date.today
    select { |e| e.date == today }
  end

  def this_week
    today = Date.today
    week_end = today + 7
    select { |e| e.date.between?(today, week_end) }
  end

  def upcoming(days = 30)
    today = Date.today
    future = today + days
    select { |e| e.date >= today && e.date <= future }
  end

  def by_month(year, month)
    select { |e| e.date.year == year && e.date.month == month }
  end

  def total_attendees
    sum(&:attendees)
  end

  def busiest_day
    group_by(&:date).max_by { |_, events| events.length }&.first
  end

  def conflict_check(date, duration)
    end_time = date + (duration.to_f / 24)
    any? { |e| date < (e.date + (e.duration_hours.to_f / 24)) && end_time > e.date }
  end
end

cal = EventCalendar.new
base = Date.today

cal.schedule("Team Standup", base, 0.5, 10)
   .schedule("Sprint Review", base + 1, 2, 20)
   .schedule("1-on-1", base + 2, 1, 2)
   .schedule("All-hands", base + 7, 3, 100)
   .schedule("Client Meeting", base + 14, 1.5, 8)

puts "Events this month: #{cal.by_month(base.year, base.month).count}"
puts "Total attendees: #{cal.total_attendees}"
puts "Busiest day: #{cal.busiest_day}"
puts "Upcoming (7 days): #{cal.upcoming(7).count}"
cal.sort_by(&:attendees).reverse.first(3).each { |e| puts e }
```

---

## Step 400: Implementing each as Foundation

`each` คือ method เดียวที่ต้องมีสำหรับ Enumerable ทุกอย่างสร้างบน each:

```ruby
# แสดงให้เห็นว่า Enumerable สร้างทุกอย่างจาก each
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

  def my_reduce(initial = nil, &block)
    acc = initial
    each do |item|
      acc = acc.nil? ? item : block.call(acc, item)
    end
    acc
  end

  def my_find
    each { |item| return item if yield(item) }
    nil
  end

  def my_any?
    each { |item| return true if yield(item) }
    false
  end

  def my_all?
    each { |item| return false unless yield(item) }
    true
  end

  def my_count
    n = 0
    each { |_| n += 1 }
    n
  end

  def my_sort_by(&key_fn)
    to_a.sort { |a, b| key_fn.call(a) <=> key_fn.call(b) }
  end
end

class SimpleList
  include MyEnumerable

  def initialize(*items)
    @items = items
  end

  def each(&block)
    @items.each(&block)
  end
end

list = SimpleList.new(3, 1, 4, 1, 5, 9, 2, 6)
puts list.my_map { |n| n * 2 }.inspect         # [6, 2, 8, 2, 10, 18, 4, 12]
puts list.my_select { |n| n > 3 }.inspect      # [4, 5, 9, 6]
puts list.my_reduce(0) { |s, n| s + n }        # 31
puts list.my_find { |n| n > 5 }                # 9
puts list.my_any? { |n| n > 8 }                # true
puts list.my_all? { |n| n > 0 }                # true
puts list.my_count                              # 8
puts list.my_sort_by { |n| -n }.inspect        # [9, 6, 5, 4, 3, 2, 1, 1]
```

---

## Step 401: Practical Example - GradeBook

```ruby
class GradeBook
  include Enumerable
  include Comparable

  GRADE_POINTS = { "A" => 4.0, "B+" => 3.5, "B" => 3.0, "C+" => 2.5,
                   "C" => 2.0, "D+" => 1.5, "D" => 1.0, "F" => 0.0 }.freeze

  Entry = Struct.new(:subject, :credits, :score) do
    def letter_grade
      case score
      when 80..100 then "A"
      when 75...80 then "B+"
      when 70...75 then "B"
      when 65...70 then "C+"
      when 60...65 then "C"
      when 55...60 then "D+"
      when 50...55 then "D"
      else "F"
      end
    end

    def grade_point
      GRADE_POINTS[letter_grade]
    end

    def quality_points
      credits * grade_point
    end

    def to_s
      "#{subject} (#{credits} cr): #{score} -> #{letter_grade}"
    end
  end

  attr_reader :student_name

  def initialize(student_name)
    @student_name = student_name
    @entries = []
  end

  def add_entry(subject, credits, score)
    @entries << Entry.new(subject, credits, score)
    self
  end

  def each(&block)
    @entries.each(&block)
  end

  def <=>(other)
    gpa <=> other.gpa
  end

  def total_credits
    sum(&:credits)
  end

  def gpa
    return 0.0 if total_credits == 0
    total_quality_points = sum(&:quality_points)
    (total_quality_points / total_credits).round(2)
  end

  def passed_subjects
    select { |e| e.score >= 50 }
  end

  def failed_subjects
    reject { |e| e.score >= 50 }
  end

  def honor_roll?
    gpa >= 3.5 && failed_subjects.empty?
  end

  def summary
    puts "=" * 40
    puts "Grade Report: #{@student_name}"
    puts "=" * 40
    each { |e| puts "  #{e}" }
    puts "-" * 40
    puts "  Total Credits: #{total_credits}"
    puts "  GPA: #{gpa}"
    puts "  Honor Roll: #{honor_roll? ? 'Yes 🎉' : 'No'}"
    puts "=" * 40
  end
end

# สร้าง GradeBook สำหรับนักศึกษา
alice = GradeBook.new("Alice")
alice.add_entry("Calculus", 3, 88)
     .add_entry("Physics", 4, 75)
     .add_entry("Chemistry", 3, 92)
     .add_entry("English", 2, 85)
     .add_entry("History", 2, 78)

bob = GradeBook.new("Bob")
bob.add_entry("Calculus", 3, 62)
   .add_entry("Physics", 4, 55)
   .add_entry("Chemistry", 3, 70)
   .add_entry("English", 2, 80)
   .add_entry("History", 2, 68)

alice.summary
bob.summary

# เปรียบเทียบ GradeBook
puts "Alice has better GPA? #{alice > bob}"

# เรียงลำดับ class
class_grades = [alice, bob]
class_grades.sort.reverse.each_with_index do |gb, i|
  puts "Rank #{i + 1}: #{gb.student_name} (GPA: #{gb.gpa})"
end
```

---

## Step 402: PriorityQueue Implementation

```ruby
class PriorityQueue
  include Enumerable

  Item = Struct.new(:value, :priority) do
    include Comparable

    def <=>(other)
      other.priority <=> priority  # สูงกว่า = ออกก่อน
    end
  end

  def initialize
    @heap = []
  end

  def push(value, priority = 0)
    @heap << Item.new(value, priority)
    @heap.sort!
    self
  end

  alias << push

  def pop
    @heap.shift&.value
  end

  def peek
    @heap.first&.value
  end

  def each(&block)
    @heap.each { |item| block.call(item.value) }
  end

  def empty?
    @heap.empty?
  end

  def size
    @heap.size
  end

  def to_s
    @heap.map { |item| "#{item.value}(#{item.priority})" }.join(" -> ")
  end
end

# ทดสอบ PriorityQueue
pq = PriorityQueue.new
pq.push("Low priority task", 1)
pq.push("Critical bug fix", 10)
pq.push("Normal task", 5)
pq.push("Emergency!", 15)
pq.push("Nice to have", 2)

puts "Queue: #{pq}"
puts "Items: #{pq.count}"

puts "\nProcessing by priority:"
until pq.empty?
  puts "  Processing: #{pq.pop}"
end

# ตัวอย่างจริง: Task scheduler
scheduler = PriorityQueue.new
scheduler.push("Send newsletter",  priority: 3)
scheduler.push("Fix production bug", priority: 10)
scheduler.push("Update docs",       priority: 1)
scheduler.push("Security patch",    priority: 9)
scheduler.push("Code review",       priority: 5)

# Enumerable methods ทำงานได้!
puts "\nAll tasks sorted by priority:"
scheduler.sort_by { |t| -scheduler.instance_variable_get(:@heap).find { |i| i.value == t }.priority }
         .each { |t| puts "  - #{t}" }
```

---

## Step 403: Combining Comparable และ Enumerable

```ruby
# class ที่ implement ทั้งสอง
class Student
  include Comparable

  attr_accessor :name, :scores

  def initialize(name, scores)
    @name   = name
    @scores = scores
  end

  def average
    @scores.sum.to_f / @scores.size
  end

  def <=>(other)
    average <=> other.average
  end

  def to_s
    "#{@name}: avg=#{average.round(1)}"
  end
end

class Classroom
  include Enumerable

  def initialize(name)
    @name     = name
    @students = []
  end

  def enroll(student)
    @students << student
    self
  end

  def each(&block)
    @students.each(&block)
  end

  def top_student
    max
  end

  def bottom_student
    min
  end

  def honor_students(threshold = 80)
    select { |s| s.average >= threshold }
  end

  def class_average
    sum(&:average) / count
  end

  def ranking
    sort.reverse.each_with_index.map { |s, i| [i + 1, s] }
  end

  def grade_distribution
    group_by do |s|
      case s.average
      when 80..100 then "A"
      when 70...80 then "B"
      when 60...70 then "C"
      else "F"
      end
    end.transform_values(&:count)
  end
end

classroom = Classroom.new("Ruby101")
classroom.enroll(Student.new("Alice",   [90, 85, 92, 88]))
         .enroll(Student.new("Bob",     [70, 68, 75, 72]))
         .enroll(Student.new("Charlie", [85, 90, 88, 91]))
         .enroll(Student.new("Diana",   [60, 65, 58, 62]))
         .enroll(Student.new("Eve",     [95, 98, 92, 96]))

puts "Class: #{classroom.count} students"
puts "Class average: #{classroom.class_average.round(1)}"
puts "Top student: #{classroom.top_student}"
puts "Bottom student: #{classroom.bottom_student}"
puts "\nHonor roll:"
classroom.honor_students.each { |s| puts "  #{s}" }
puts "\nClass ranking:"
classroom.ranking.each { |rank, student| puts "  #{rank}. #{student}" }
puts "\nGrade distribution: #{classroom.grade_distribution}"
```

---

## Step 404: Custom Iterator with Enumerator

```ruby
# สร้าง Enumerator จาก method
class BinaryTree
  include Enumerable

  Node = Struct.new(:value, :left, :right)

  def initialize
    @root = nil
  end

  def insert(value)
    @root = insert_node(@root, value)
    self
  end

  def each(&block)
    return to_enum(:each) unless block_given?
    inorder(@root, &block)
  end

  def each_preorder(&block)
    return to_enum(:each_preorder) unless block_given?
    preorder(@root, &block)
  end

  def each_postorder(&block)
    return to_enum(:each_postorder) unless block_given?
    postorder(@root, &block)
  end

  def height
    tree_height(@root)
  end

  private

  def insert_node(node, value)
    return Node.new(value, nil, nil) if node.nil?

    if value < node.value
      node.left = insert_node(node.left, value)
    elsif value > node.value
      node.right = insert_node(node.right, value)
    end
    node
  end

  def inorder(node, &block)
    return if node.nil?
    inorder(node.left, &block)
    block.call(node.value)
    inorder(node.right, &block)
  end

  def preorder(node, &block)
    return if node.nil?
    block.call(node.value)
    preorder(node.left, &block)
    preorder(node.right, &block)
  end

  def postorder(node, &block)
    return if node.nil?
    postorder(node.left, &block)
    postorder(node.right, &block)
    block.call(node.value)
  end

  def tree_height(node)
    return 0 if node.nil?
    [tree_height(node.left), tree_height(node.right)].max + 1
  end
end

tree = BinaryTree.new
[5, 3, 7, 1, 4, 6, 8, 2].each { |n| tree.insert(n) }

puts "In-order (sorted): #{tree.to_a.inspect}"
puts "Pre-order: #{tree.each_preorder.to_a.inspect}"
puts "Post-order: #{tree.each_postorder.to_a.inspect}"
puts "Height: #{tree.height}"

# Enumerable methods ทำงาน!
puts "Sum: #{tree.sum}"
puts "Min: #{tree.min}"
puts "Max: #{tree.max}"
puts "Evens: #{tree.select(&:even?).inspect}"
```

---

## Step 405: ตัวอย่าง - Observable Collection

```ruby
# Collection ที่แจ้ง observers เมื่อมีการเปลี่ยนแปลง
module Observable
  def self.included(base)
    base.instance_variable_set(:@observers, [])
    base.extend(ClassMethods)
  end

  module ClassMethods
    def observers
      @observers
    end

    def add_observer(observer)
      @observers << observer
    end
  end

  def notify_observers(event, data = nil)
    self.class.observers.each { |obs| obs.update(event, data) }
  end
end

class LogObserver
  def update(event, data)
    puts "[LOG] Event: #{event}, Data: #{data.inspect}"
  end
end

class CountObserver
  attr_reader :counts

  def initialize
    @counts = Hash.new(0)
  end

  def update(event, _data)
    @counts[event] += 1
  end
end

class WatchedCollection
  include Enumerable
  include Observable

  def initialize
    @items = []
  end

  def add(item)
    @items << item
    notify_observers(:added, item)
    self
  end

  def remove(item)
    if @items.delete(item)
      notify_observers(:removed, item)
    end
    self
  end

  def each(&block)
    @items.each(&block)
  end
end

log_obs   = LogObserver.new
count_obs = CountObserver.new

WatchedCollection.add_observer(log_obs)
WatchedCollection.add_observer(count_obs)

collection = WatchedCollection.new
collection.add(1).add(2).add(3)
collection.remove(2)
collection.add(4)

puts "\nEvent counts: #{count_obs.counts}"
puts "Final collection: #{collection.to_a.inspect}"
puts "Sum: #{collection.sum}"
```

---

## Step 406-410: แบบฝึกหัด (20 ข้อ)

### ข้อที่ 1-5: Comparable

**ข้อ 1:** สร้าง `Rectangle` class ที่ Comparable โดยเปรียบเทียบด้วย area

```ruby
# เฉลย
class Rectangle
  include Comparable

  attr_reader :width, :height

  def initialize(width, height)
    @width  = width
    @height = height
  end

  def area
    @width * @height
  end

  def perimeter
    2 * (@width + @height)
  end

  def <=>(other)
    area <=> other.area
  end

  def to_s
    "Rectangle(#{@width}x#{@height}, area=#{area})"
  end
end

rectangles = [
  Rectangle.new(3, 4),
  Rectangle.new(5, 2),
  Rectangle.new(6, 6),
  Rectangle.new(2, 8)
]

puts "Sorted by area:"
rectangles.sort.each { |r| puts "  #{r}" }
puts "Largest: #{rectangles.max}"
puts "Smallest: #{rectangles.min}"
```

**ข้อ 2:** สร้าง `Employee` class ที่ Comparable ตาม salary และมี `between?` check สำหรับ salary range

```ruby
# เฉลย
class Employee
  include Comparable

  attr_reader :name, :salary, :department

  def initialize(name, salary, department)
    @name       = name
    @salary     = salary
    @department = department
  end

  def <=>(other)
    @salary <=> other.salary
  end

  def to_s
    "#{@name} (#{@department}): #{@salary} บาท"
  end
end

employees = [
  Employee.new("Alice", 85000, "Engineering"),
  Employee.new("Bob", 65000, "Marketing"),
  Employee.new("Charlie", 95000, "Engineering"),
  Employee.new("Diana", 75000, "HR"),
  Employee.new("Eve", 110000, "Engineering")
]

mid_range = Employee.new("low", 70000, "")..Employee.new("high", 90000, "")
in_range = employees.select { |e| mid_range.cover?(e) }
puts "Employees with salary 70k-90k:"
in_range.each { |e| puts "  #{e}" }

puts "\nHighest paid: #{employees.max}"
puts "Lowest paid: #{employees.min}"
puts "\nAll sorted:"
employees.sort.each { |e| puts "  #{e}" }
```

**ข้อ 3:** สร้าง `Priority` enum-like class ที่เปรียบเทียบได้

```ruby
# เฉลย
class Priority
  include Comparable

  LEVELS = { low: 1, medium: 2, high: 3, critical: 4 }.freeze

  attr_reader :level

  def initialize(level)
    raise ArgumentError, "Invalid level" unless LEVELS.key?(level)
    @level = level
  end

  def <=>(other)
    LEVELS[@level] <=> LEVELS[other.level]
  end

  def to_s
    @level.to_s.capitalize
  end

  def self.low      = new(:low)
  def self.medium   = new(:medium)
  def self.high     = new(:high)
  def self.critical = new(:critical)
end

tasks = [
  { name: "Update docs",     priority: Priority.low },
  { name: "Fix bug",         priority: Priority.high },
  { name: "Security patch",  priority: Priority.critical },
  { name: "Code review",     priority: Priority.medium },
  { name: "Server down!!",   priority: Priority.critical }
]

sorted = tasks.sort_by { |t| t[:priority] }.reverse
sorted.each { |t| puts "[#{t[:priority]}] #{t[:name]}" }
```

**ข้อ 4:** สร้าง class Fraction ที่ Comparable

```ruby
# เฉลย
class Fraction
  include Comparable

  attr_reader :numerator, :denominator

  def initialize(numerator, denominator)
    raise ArgumentError, "Denominator cannot be zero" if denominator == 0
    gcd = numerator.gcd(denominator)
    sign = denominator < 0 ? -1 : 1
    @numerator   = sign * numerator / gcd
    @denominator = sign * denominator / gcd
  end

  def to_f
    @numerator.to_f / @denominator
  end

  def <=>(other)
    to_f <=> other.to_f
  end

  def +(other)
    Fraction.new(
      @numerator * other.denominator + other.numerator * @denominator,
      @denominator * other.denominator
    )
  end

  def to_s
    @denominator == 1 ? @numerator.to_s : "#{@numerator}/#{@denominator}"
  end
end

fractions = [
  Fraction.new(1, 2),
  Fraction.new(3, 4),
  Fraction.new(1, 3),
  Fraction.new(2, 3),
  Fraction.new(5, 8)
]

puts "Sorted fractions: #{fractions.sort.inspect}"
puts "Largest: #{fractions.max}"
puts "Smallest: #{fractions.min}"

f1 = Fraction.new(1, 3)
f2 = Fraction.new(1, 2)
puts "1/3 < 1/2: #{f1 < f2}"
puts "Sum: #{(f1 + f2)}"
```

**ข้อ 5:** `clamp` ด้วย Temperature class

```ruby
# เฉลย
class Temperature
  include Comparable
  attr_reader :celsius

  def initialize(celsius)
    @celsius = celsius
  end

  def <=>(other)
    @celsius <=> other.celsius
  end

  def to_s
    "#{@celsius}°C"
  end
end

# Comfortable room temperature range
min_comfort = Temperature.new(18)
max_comfort = Temperature.new(26)

readings = [
  Temperature.new(10),
  Temperature.new(20),
  Temperature.new(30),
  Temperature.new(22),
  Temperature.new(5)
]

puts "Clamped temperatures:"
readings.each do |t|
  clamped = t.clamp(min_comfort, max_comfort)
  adjusted = clamped == t ? "OK" : "adjusted"
  puts "  #{t} -> #{clamped} (#{adjusted})"
end
```

### ข้อที่ 6-10: Enumerable

**ข้อ 6:** สร้าง `Library` class ที่ Enumerable สำหรับจัดการหนังสือ

```ruby
# เฉลย
class Library
  include Enumerable

  Book = Struct.new(:title, :author, :genre, :year, :available)

  def initialize(name)
    @name  = name
    @books = []
  end

  def add_book(title, author, genre, year, available = true)
    @books << Book.new(title, author, genre, year, available)
    self
  end

  def each(&block)
    @books.each(&block)
  end

  def available_books
    select(&:available)
  end

  def by_genre(genre)
    select { |b| b.genre == genre }
  end

  def by_author(author)
    select { |b| b.author.downcase.include?(author.downcase) }
  end

  def recent(years = 5)
    current_year = Time.now.year
    select { |b| b.year >= current_year - years }
  end

  def statistics
    {
      total: count,
      available: available_books.count,
      genres: map(&:genre).tally,
      avg_year: (sum(&:year).to_f / count).round
    }
  end
end

lib = Library.new("City Library")
lib.add_book("The Ruby Way", "Fulton", :programming, 2015)
   .add_book("POODR", "Metz", :programming, 2018)
   .add_book("Clean Code", "Martin", :programming, 2008)
   .add_book("Design Patterns", "GoF", :programming, 1994, false)
   .add_book("Sapiens", "Harari", :history, 2011)
   .add_book("1984", "Orwell", :fiction, 1949)

puts lib.statistics.inspect
puts "\nAvailable programming books:"
lib.available_books.select { |b| b.genre == :programming }.each do |b|
  puts "  #{b.title} by #{b.author} (#{b.year})"
end
```

**ข้อ 7:** สร้าง `Portfolio` class สำหรับหุ้น

```ruby
# เฉลย
class Portfolio
  include Enumerable
  include Comparable

  Holding = Struct.new(:symbol, :shares, :purchase_price, :current_price) do
    def market_value
      shares * current_price
    end

    def cost_basis
      shares * purchase_price
    end

    def gain_loss
      market_value - cost_basis
    end

    def return_pct
      (gain_loss / cost_basis * 100).round(2)
    end
  end

  def initialize(owner)
    @owner    = owner
    @holdings = []
  end

  def buy(symbol, shares, price)
    existing = @holdings.find { |h| h.symbol == symbol }
    if existing
      total_shares = existing.shares + shares
      avg_price = (existing.cost_basis + shares * price) / total_shares
      existing.shares = total_shares
      existing.purchase_price = avg_price
    else
      @holdings << Holding.new(symbol, shares, price, price)
    end
    self
  end

  def update_price(symbol, price)
    holding = @holdings.find { |h| h.symbol == symbol }
    holding.current_price = price if holding
    self
  end

  def each(&block)
    @holdings.each(&block)
  end

  def <=>(other)
    total_value <=> other.total_value
  end

  def total_value
    sum(&:market_value)
  end

  def total_return
    sum(&:gain_loss)
  end

  def total_return_pct
    (total_return / sum(&:cost_basis) * 100).round(2)
  end

  def winners
    select { |h| h.gain_loss > 0 }.sort_by { |h| -h.return_pct }
  end

  def losers
    select { |h| h.gain_loss < 0 }.sort_by(&:return_pct)
  end

  def summary
    puts "Portfolio: #{@owner}"
    puts "Total value: #{total_value.round(2)}"
    puts "Total return: #{total_return.round(2)} (#{total_return_pct}%)"
    puts "\nHoldings:"
    sort_by { |h| -h.market_value }.each do |h|
      puts "  #{h.symbol}: #{h.shares} shares @ #{h.current_price} = #{h.market_value.round(2)} (#{h.return_pct}%)"
    end
  end
end

portfolio = Portfolio.new("Alice")
portfolio.buy("AAPL", 100, 150.0)
         .buy("GOOGL", 50, 2800.0)
         .buy("MSFT", 75, 300.0)
         .update_price("AAPL", 175.0)
         .update_price("GOOGL", 2650.0)
         .update_price("MSFT", 325.0)

portfolio.summary
```

**ข้อ 8:** สร้าง Matrix class ที่ Enumerable

```ruby
# เฉลย
class Matrix
  include Enumerable

  def initialize(rows)
    @rows = rows
    @num_rows = rows.size
    @num_cols = rows.first&.size || 0
  end

  def self.zeros(rows, cols)
    new(Array.new(rows) { Array.new(cols, 0) })
  end

  def [](row, col)
    @rows[row][col]
  end

  def []=(row, col, value)
    @rows[row][col] = value
  end

  def each(&block)
    @rows.each do |row|
      row.each(&block)
    end
  end

  def each_row(&block)
    @rows.each(&block)
  end

  def row(i)
    @rows[i]
  end

  def col(j)
    @rows.map { |row| row[j] }
  end

  def transpose
    Matrix.new(@rows.transpose)
  end

  def +(other)
    result = @rows.zip(other.instance_variable_get(:@rows)).map do |row1, row2|
      row1.zip(row2).map { |a, b| a + b }
    end
    Matrix.new(result)
  end

  def to_s
    @rows.map { |row| row.inspect }.join("\n")
  end
end

m = Matrix.new([[1, 2, 3], [4, 5, 6], [7, 8, 9]])

puts "Sum: #{m.sum}"
puts "Max: #{m.max}"
puts "Min: #{m.min}"
puts "Evens: #{m.select(&:even?).inspect}"
puts "Row 0: #{m.row(0).inspect}"
puts "Col 1: #{m.col(1).inspect}"
puts "\nMatrix:"
puts m
puts "\nTransposed:"
puts m.transpose
```

**ข้อ 9:** ใช้ Comparable ใน Sorting Algorithm

```ruby
# เฉลย - Insertion Sort กับ Comparable objects
def insertion_sort(arr)
  arr = arr.dup
  (1...arr.size).each do |i|
    key = arr[i]
    j = i - 1
    while j >= 0 && arr[j] > key
      arr[j + 1] = arr[j]
      j -= 1
    end
    arr[j + 1] = key
  end
  arr
end

# ทดสอบกับ Integer
puts insertion_sort([5, 2, 8, 1, 9, 3]).inspect

# ทดสอบกับ Version
class Version
  include Comparable
  def initialize(v)
    @parts = v.split('.').map(&:to_i)
  end
  def <=>(other)
    @parts <=> other.instance_variable_get(:@parts)
  end
  def to_s
    @parts.join('.')
  end
end

versions = %w[2.1.0 1.0.0 2.0.3 1.5.2 3.0.0].map { |v| Version.new(v) }
puts insertion_sort(versions).inspect
```

**ข้อ 10:** สร้าง `TimeSlot` class ที่ Comparable สำหรับ scheduling

```ruby
# เฉลย
class TimeSlot
  include Comparable

  attr_reader :start_time, :end_time, :title

  def initialize(title, start_hour, start_min, duration_min)
    @title      = title
    @start_time = start_hour * 60 + start_min
    @end_time   = @start_time + duration_min
  end

  def <=>(other)
    @start_time <=> other.start_time
  end

  def overlaps?(other)
    @start_time < other.end_time && @end_time > other.start_time
  end

  def duration
    @end_time - @start_time
  end

  def to_s
    sh, sm = @start_time.divmod(60)
    eh, em = @end_time.divmod(60)
    "#{@title} #{sh}:#{sm.to_s.rjust(2,'0')}-#{eh}:#{em.to_s.rjust(2,'0')}"
  end
end

slots = [
  TimeSlot.new("Standup", 9, 0, 30),
  TimeSlot.new("Lunch", 12, 0, 60),
  TimeSlot.new("Sprint Review", 14, 0, 90),
  TimeSlot.new("1-on-1", 10, 30, 45),
  TimeSlot.new("Planning", 11, 30, 60)
]

puts "Schedule (sorted):"
slots.sort.each { |s| puts "  #{s}" }

puts "\nChecking for conflicts:"
sorted = slots.sort
sorted.each_cons(2) do |a, b|
  if a.overlaps?(b)
    puts "  CONFLICT: #{a.title} overlaps with #{b.title}"
  end
end
```

### ข้อที่ 11-20: ขั้นสูง

**ข้อ 11:** สร้าง Graph class ที่ Enumerable สำหรับ BFS

```ruby
# เฉลย
class Graph
  include Enumerable

  def initialize
    @adjacency = Hash.new { |h, k| h[k] = [] }
  end

  def add_edge(from, to)
    @adjacency[from] << to
    @adjacency[to] << from  # undirected
    self
  end

  def neighbors(node)
    @adjacency[node]
  end

  # BFS traversal starting from first node
  def each(start = @adjacency.keys.first, &block)
    return to_enum(:each, start) unless block_given?

    visited = Set.new
    queue = [start]

    while (node = queue.shift)
      next if visited.include?(node)
      visited.add(node)
      block.call(node)
      queue.concat(@adjacency[node].reject { |n| visited.include?(n) })
    end
  end

  def connected?(a, b)
    include?(b) && find { |n| n == b }
  end
end

require 'set'
g = Graph.new
g.add_edge(1, 2).add_edge(1, 3).add_edge(2, 4).add_edge(3, 4).add_edge(4, 5)

puts "BFS traversal: #{g.to_a.inspect}"
puts "Nodes: #{g.count}"
```

**ข้อ 12-20:** (รวม exercises เพิ่มเติม)

```ruby
# ข้อ 12: Immutable collection
class ImmutableList
  include Enumerable
  include Comparable

  def initialize(*items)
    @items = items.freeze
  end

  def each(&block)
    @items.each(&block)
  end

  def <=>(other)
    to_a <=> other.to_a
  end

  def append(item)
    ImmutableList.new(*@items, item)
  end

  def prepend(item)
    ImmutableList.new(item, *@items)
  end

  def without(item)
    ImmutableList.new(*@items.reject { |i| i == item })
  end

  def to_s
    "ImmutableList#{@items.inspect}"
  end
end

list1 = ImmutableList.new(1, 2, 3)
list2 = list1.append(4).append(5)
list3 = list1.prepend(0)

puts list1  # Original unchanged
puts list2
puts list3
puts "Sum: #{list2.sum}"
puts "List1 < List2: #{list1 < list2}"

# ข้อ 13-20 abbreviated examples
# ข้อ 13: Weighted collection
class WeightedCollection
  include Enumerable

  def initialize
    @items = []
  end

  def add(item, weight)
    @items << { item: item, weight: weight }
    self
  end

  def each
    @items.each { |i| yield i[:item] }
  end

  def weighted_sample
    total = @items.sum { |i| i[:weight] }
    r = rand * total
    cumulative = 0
    @items.each do |i|
      cumulative += i[:weight]
      return i[:item] if r <= cumulative
    end
  end

  def probability_of(item)
    total = @items.sum { |i| i[:weight] }
    found = @items.find { |i| i[:item] == item }
    found ? (found[:weight] / total * 100).round(1) : 0
  end
end

wc = WeightedCollection.new
wc.add("Common", 60).add("Rare", 30).add("Epic", 9).add("Legendary", 1)
puts "All items: #{wc.to_a.inspect}"
puts "Common probability: #{wc.probability_of('Common')}%"
puts "Legendary probability: #{wc.probability_of('Legendary')}%"
10.times { print "#{wc.weighted_sample} " }
puts
```

---

## สรุป

### Comparable Module
- **ต้องการ**: implement `<=>` operator
- **ได้รับ**: `<`, `<=`, `==`, `>=`, `>`, `between?`, `clamp`
- **`<=>` คืนค่า**: -1 (น้อยกว่า), 0 (เท่ากัน), 1 (มากกว่า), nil (เปรียบเทียบไม่ได้)
- **ใช้สำหรับ**: sorting, range checking, comparison operators

### Enumerable Module
- **ต้องการ**: implement `each` method
- **ได้รับ**: 50+ methods รวมถึง map, select, reduce, sort, group_by, find, any?, all? ฯลฯ
- **Lazy evaluation**: `.lazy` สำหรับ infinite sequences หรือ performance
- **ใช้สำหรับ**: collections, custom iterators, data processing

### Pattern ที่ดีที่สุด
1. Include Comparable เมื่อ objects มีลำดับที่ชัดเจน
2. Include Enumerable เมื่อ class เป็น collection of items
3. ทั้งสองทำงานร่วมกันได้ดี
4. `each` เป็น foundation ของ Enumerable ทั้งหมด
5. `<=>` เป็น foundation ของ Comparable ทั้งหมด

---

*ตอนถัดไป: ตอนที่ 20 - Ruby Standard Library*
