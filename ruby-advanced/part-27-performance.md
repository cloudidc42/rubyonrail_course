# ตอนที่ 27: Performance Optimization ใน Ruby (Steps 586-605)

## บทนำ

Performance Optimization คือกระบวนการทำให้โปรแกรมทำงานได้เร็วขึ้น ใช้ memory น้อยลง หรือทำงานได้ดีขึ้นโดยรวม ก่อนจะ optimize ควรต้อง **measure** ก่อนเสมอ ไม่ควร optimize แบบ "เดา" เพราะอาจทำให้เสียเวลาและโค้ดอ่านยากขึ้นโดยไม่จำเป็น

---

## Step 586: Benchmarking ด้วย Benchmark Module

### Benchmark พื้นฐาน

```ruby
require 'benchmark'

# วัดเวลาการทำงานพื้นฐาน
Benchmark.measure do
  100_000.times { "hello" + " world" }
end

# => #<Benchmark::Tms:0x... @label="", @real=0.123, @cstime=0.0, @cutime=0.0, @stime=0.01, @utime=0.11>
```

### Benchmark.bm - เปรียบเทียบหลายวิธี

```ruby
require 'benchmark'

n = 1_000_000

Benchmark.bm(20) do |x|
  x.report("String + concat:") do
    n.times { "hello" + " world" }
  end

  x.report("String interpolation:") do
    n.times { "hello #{"world"}" }
  end

  x.report("String <<:") do
    n.times do
      s = "hello"
      s << " world"
    end
  end
end

# ผลลัพธ์:
#                            user     system      total        real
# String + concat:       0.320000   0.010000   0.330000 (  0.340000)
# String interpolation:  0.180000   0.000000   0.180000 (  0.190000)
# String <<:             0.150000   0.000000   0.150000 (  0.160000)
```

### Benchmark.bmbm - ลด JIT warmup noise

```ruby
require 'benchmark'

n = 500_000
data = (1..1000).to_a

Benchmark.bmbm(30) do |x|
  x.report("Array#each + push:") do
    n.times do
      result = []
      data.each { |i| result.push(i * 2) }
    end
  end

  x.report("Array#map:") do
    n.times { data.map { |i| i * 2 } }
  end

  x.report("Array#collect (alias):") do
    n.times { data.collect { |i| i * 2 } }
  end

  x.report("Array#map with Symbol:") do
    n.times { data.map(&method(:itself)) }
  end
end
```

### Benchmark IPS (Iterations Per Second)

```ruby
# gem 'benchmark-ips'
require 'benchmark/ips'

Benchmark.ips do |x|
  x.config(warmup: 2, time: 5)

  x.report("Hash access :symbol") {
    h = { name: "Ruby", version: 3 }
    h[:name]
  }

  x.report("Hash access 'string'") {
    h = { "name" => "Ruby", "version" => 3 }
    h["name"]
  }

  x.compare!
end

# ผลลัพธ์บอก x faster comparison
```

---

## Step 587: Memory Profiler

### memory_profiler gem

```ruby
# gem 'memory_profiler'
require 'memory_profiler'

report = MemoryProfiler.report do
  # โค้ดที่ต้องการ profile
  1000.times do
    arr = Array.new(100) { |i| "item_#{i}" }
    arr.map { |s| s.upcase }
  end
end

report.pretty_print

# ผลลัพธ์:
# Total allocated: 12.88 MB (204000 objects)
# Total retained:  0 bytes (0 objects)
#
# allocated memory by gem
# -----------------------------------
#   12.88 MB  other
#
# allocated objects by class
# -----------------------------------
#    204000  String
```

### ObjectSpace สำหรับ memory tracking

```ruby
require 'objspace'

# นับ object ก่อน
before = ObjectSpace.count_objects

# รันโค้ด
10_000.times { "hello world" }

# นับหลัง
after = ObjectSpace.count_objects

puts "Strings created: #{after[:T_STRING] - before[:T_STRING]}"
puts "Total objects: #{after[:TOTAL] - before[:TOTAL]}"

# ดู memory ที่ใช้
puts ObjectSpace.memsize_of("hello world")  # bytes ที่ใช้

# Trace object allocation
ObjectSpace.trace_object_allocations_start
arr = Array.new(5) { |i| "item #{i}" }
ObjectSpace.trace_object_allocations_stop

arr.each do |obj|
  puts "#{obj} allocated at #{ObjectSpace.allocation_sourcefile(obj)}:#{ObjectSpace.allocation_sourceline(obj)}"
end
```

---

## Step 588: ruby-prof Profiling

```ruby
# gem 'ruby-prof'
require 'ruby-prof'

# Profile โค้ด
RubyProf.start

# โค้ดที่ต้องการ profile
def slow_method
  result = []
  10_000.times do |i|
    result << i * 2
  end
  result
end

1000.times { slow_method }

result = RubyProf.stop

# แสดงผลแบบ flat
printer = RubyProf::FlatPrinter.new(result)
printer.print(STDOUT)

# แสดงผลแบบ call tree
printer = RubyProf::CallTreePrinter.new(result)
printer.print(STDOUT)

# สร้างไฟล์ HTML
File.open("profile.html", "w") do |f|
  printer = RubyProf::CallStackPrinter.new(result)
  printer.print(f)
end
```

### Profile เฉพาะส่วน

```ruby
# ruby-prof แบบ block
result = RubyProf.profile do
  100.times do
    # การดำเนินการที่ต้องการ test
    string = ""
    1000.times { |i| string += "item #{i}" }
  end
end

printer = RubyProf::FlatPrinter.new(result)
printer.print(STDOUT, min_percent: 2)  # แสดงเฉพาะที่ใช้เวลา > 2%
```

---

## Step 589: Identifying Bottlenecks

### ขั้นตอนการ Profile

```
1. Measure ก่อน optimize (baseline)
2. หา hotspot ด้วย profiler
3. ทำความเข้าใจว่าทำไมถึงช้า
4. Optimize
5. Measure อีกครั้ง (ตรวจสอบผล)
6. ทำซ้ำถ้าจำเป็น
```

### ตัวอย่างการหา Bottleneck

```ruby
require 'benchmark'

# โปรแกรมช้า
class SlowProgram
  def process(data)
    result = []
    data.each do |item|
      # Bottleneck 1: String concatenation ใน loop
      log = ""
      item[:tags].each { |tag| log += tag + "," }

      # Bottleneck 2: ค้นหา database ซ้ำซ้อน
      category = find_category(item[:category_id])

      # Bottleneck 3: Sort ซ้ำซ้อน
      sorted_tags = item[:tags].sort

      result << {
        log: log,
        category: category,
        tags: sorted_tags
      }
    end
    result
  end

  def find_category(id)
    # จำลอง DB query ช้า
    sleep(0.001)
    "Category #{id}"
  end
end

# โปรแกรมที่ optimize แล้ว
class FastProgram
  def process(data)
    # Optimize 1: Preload categories (N+1 query fix)
    category_ids = data.map { |i| i[:category_id] }.uniq
    categories = load_categories(category_ids)

    data.map do |item|
      # Optimize 2: ใช้ join แทน string concat
      log = item[:tags].join(",")

      # Optimize 3: ใช้ preloaded data
      category = categories[item[:category_id]]

      # Optimize 4: Sort ครั้งเดียว ถ้า tags เรียงแล้ว
      sorted_tags = item[:tags].sort

      {
        log: log,
        category: category,
        tags: sorted_tags
      }
    end
  end

  def load_categories(ids)
    # Load ทีเดียว
    ids.each_with_object({}) { |id, h| h[id] = "Category #{id}" }
  end
end
```

---

## Step 590: Algorithm Complexity (Big O Notation)

### O(1) - Constant Time

```ruby
# Hash lookup - O(1)
def find_by_id(records_hash, id)
  records_hash[id]  # ไม่ขึ้นกับขนาด hash
end

users = { 1 => "สมชาย", 2 => "สมหญิง", 3 => "สมศรี" }
puts find_by_id(users, 2)  # เร็วเท่ากันไม่ว่าจะมีกี่ record
```

### O(n) - Linear Time

```ruby
# Linear search - O(n)
def find_user_linear(users_array, name)
  users_array.find { |u| u[:name] == name }
end

# ยิ่ง array ใหญ่ยิ่งช้า
small_array = 1000.times.map { |i| { id: i, name: "User #{i}" } }
large_array = 1_000_000.times.map { |i| { id: i, name: "User #{i}" } }

# large_array ช้ากว่า 1000 เท่า
```

### O(n²) - Quadratic Time

```ruby
# Bubble sort - O(n²) - ไม่ควรใช้!
def bubble_sort(arr)
  n = arr.length
  n.times do |i|
    (n - i - 1).times do |j|
      if arr[j] > arr[j + 1]
        arr[j], arr[j + 1] = arr[j + 1], arr[j]
      end
    end
  end
  arr
end

# Ruby built-in sort - O(n log n) - ดีกว่ามาก!
arr = (1..1000).to_a.shuffle
arr.sort  # เร็วกว่า bubble_sort มาก!

# ตัวอย่าง O(n²) ที่พบบ่อย
def find_duplicates_slow(arr)
  duplicates = []
  arr.each_with_index do |item, i|
    arr.each_with_index do |other, j|
      duplicates << item if i != j && item == other
    end
  end
  duplicates.uniq
end

# O(n) solution ที่ดีกว่า
def find_duplicates_fast(arr)
  seen = {}
  duplicates = []
  arr.each do |item|
    duplicates << item if seen[item]
    seen[item] = true
  end
  duplicates
end
```

### เปรียบเทียบ Complexity

```ruby
require 'benchmark'

data = (1..10_000).to_a.shuffle

Benchmark.bm(25) do |x|
  x.report("O(n²) - nested loops:") do
    100.times do
      find_duplicates_slow(data.first(100))
    end
  end

  x.report("O(n) - hash lookup:") do
    100.times do
      find_duplicates_fast(data.first(100))
    end
  end
end
```

---

## Step 591: String Optimization

### frozen_string_literal

```ruby
# frozen_string_literal: true
# เพิ่มที่บรรทัดแรกของทุกไฟล์ Ruby

# String literals ทั้งหมดจะ frozen และถูก reuse
str1 = "hello"
str2 = "hello"
puts str1.equal?(str2)  # => true (same object!)

# ไม่ frozen
# str1 << " world"  # => FrozenError

# สร้าง mutable string
mutable = +"hello"  # หรือ String.new("hello")
mutable << " world"
puts mutable  # => hello world
```

### String Concatenation Performance

```ruby
require 'benchmark'

n = 100_000
parts = ["Hello", " ", "World", " ", "from", " ", "Ruby"]

Benchmark.bm(30) do |x|
  x.report("String +:") do
    n.times { parts.inject(:+) }
  end

  x.report("String join:") do
    n.times { parts.join }
  end

  x.report("String interpolation:") do
    n.times { "#{parts[0]} #{parts[2]} #{parts[4]} #{parts[6]}" }
  end

  x.report("String <<:") do
    n.times do
      s = ""
      parts.each { |p| s << p }
    end
  end
end

# ผลลัพธ์: join เร็วที่สุดสำหรับ array
# << เร็วที่สุดสำหรับ sequential append
```

### String ที่ใช้บ่อย - Use Symbols

```ruby
# ใช้ symbol แทน string สำหรับ hash keys
# Bad - สร้าง string ใหม่ทุกครั้ง
hash = { "name" => "Ruby", "version" => "3.2" }

# Good - symbol คือ immutable และ reused
hash = { name: "Ruby", version: "3.2" }

# ตรวจสอบ memory
require 'objspace'
str = "hello"
sym = :hello
puts ObjectSpace.memsize_of(str)  # > 0
puts ObjectSpace.memsize_of(sym)  # น้อยกว่า
```

---

## Step 592: Array/Hash Performance Tips

### Array Tips

```ruby
# 1. ใช้ flat_map แทน map + flatten
# Bad
result = [[1, 2], [3, 4]].map { |arr| arr.map { |n| n * 2 } }.flatten
# Good
result = [[1, 2], [3, 4]].flat_map { |arr| arr.map { |n| n * 2 } }

# 2. ใช้ each_with_object แทน inject สำหรับ build collection
# Bad (inject สร้าง intermediate objects)
result = [1, 2, 3].inject([]) { |arr, n| arr + [n * 2] }

# Good
result = [1, 2, 3].each_with_object([]) { |n, arr| arr << n * 2 }

# 3. ใช้ any?/all?/none? แทน count
# Bad
[1, 2, 3, 4].select { |n| n.even? }.size > 0
# Good
[1, 2, 3, 4].any?(&:even?)

# 4. Pre-size array ถ้าทราบขนาด
n = 1_000_000
# Bad
arr = []
n.times { |i| arr << i }

# Good  
arr = Array.new(n) { |i| i }

# 5. ใช้ include? กับ Set แทน Array สำหรับ large data
require 'set'
large_array = (1..100_000).to_a
large_set = large_array.to_set

# O(n) - ช้า
large_array.include?(50_000)

# O(1) - เร็ว
large_set.include?(50_000)
```

### Hash Tips

```ruby
# 1. Hash#fetch พร้อม default (ดีกว่า || ในบางกรณี)
config = { timeout: 30 }
timeout = config.fetch(:timeout, 60)
debug = config.fetch(:debug, false)

# 2. Hash#merge vs Hash#update
base = { a: 1, b: 2 }

# merge สร้าง hash ใหม่ (immutable)
new_hash = base.merge({ c: 3 })

# update/merge! เปลี่ยน in-place (mutable แต่เร็วกว่า)
base.update({ c: 3 })

# 3. Lazy hash initialization
class Config
  def initialize
    @data = {}
  end

  def [](key)
    @data[key]
  end

  def []=(key, value)
    @data[key] = value
  end
end

# 4. Hash.new กับ default value
word_count = Hash.new(0)
"hello world hello ruby".split.each { |w| word_count[w] += 1 }
puts word_count.inspect
# => {"hello"=>2, "world"=>1, "ruby"=>1}

# 5. Hash.new กับ default block
graph = Hash.new { |h, k| h[k] = [] }
graph[:a] << :b
graph[:a] << :c
puts graph.inspect
# => {a: [:b, :c]}
```

---

## Step 593: Memoization สำหรับ Performance

### Instance Variable Memoization

```ruby
class ProductService
  def initialize(user)
    @user = user
  end

  # Memoize ด้วย ||=
  def recommended_products
    @recommended_products ||= calculate_recommendations
  end

  def total_price(product_ids)
    # Memoize ที่รับ arguments
    @price_cache ||= {}
    @price_cache[product_ids] ||= compute_price(product_ids)
  end

  private

  def calculate_recommendations
    # expensive calculation
    puts "คำนวณ recommendations..."
    Product.where(category: @user.preferred_category)
           .order(rating: :desc)
           .limit(10)
  end

  def compute_price(ids)
    Product.where(id: ids).sum(:price)
  end
end

service = ProductService.new(user)
service.recommended_products  # คำนวณครั้งแรก
service.recommended_products  # ใช้ cache
```

### Thread-safe Memoization

```ruby
require 'monitor'

class ThreadSafeCache
  include MonitorMixin

  def initialize
    super
    @cache = {}
  end

  def fetch(key)
    # Double-checked locking
    return @cache[key] if @cache.key?(key)

    synchronize do
      return @cache[key] if @cache.key?(key)
      @cache[key] = yield
    end
  end
end

cache = ThreadSafeCache.new
result = cache.fetch(:expensive_computation) { sleep(1); 42 }
```

---

## Step 594: Lazy Evaluation

### Lazy Enumerator

```ruby
# ไม่ lazy - คำนวณทั้งหมดก่อน
result = (1..1_000_000)
  .select { |n| n % 3 == 0 }
  .map { |n| n * n }
  .first(10)

# Lazy - คำนวณเท่าที่จำเป็น
result = (1..Float::INFINITY).lazy
  .select { |n| n % 3 == 0 }
  .map { |n| n * n }
  .first(10)

puts result.inspect
# => [9, 36, 81, 144, 225, 324, 441, 576, 729, 900]
```

### Custom Lazy Computation

```ruby
class LazyLoader
  def initialize(loader)
    @loader = loader
    @loaded = false
    @value = nil
  end

  def value
    unless @loaded
      @value = @loader.call
      @loaded = true
    end
    @value
  end

  def loaded?
    @loaded
  end
end

# ใช้งาน
expensive_data = LazyLoader.new(-> {
  puts "Loading data..."
  sleep(1)
  { users: [1, 2, 3], products: [4, 5, 6] }
})

puts "Before access"
puts expensive_data.loaded?  # => false

# เข้าถึงเมื่อต้องการจริงๆ
puts expensive_data.value[:users].inspect  # "Loading data..." แล้ว => [1, 2, 3]
puts expensive_data.loaded?  # => true
puts expensive_data.value[:users].inspect  # ไม่ print "Loading" อีก
```

---

## Step 595: หลีกเลี่ยงการสร้าง Object ใน Loop

### Object Allocation ใน Loop

```ruby
require 'benchmark'

n = 1_000_000

# BAD - สร้าง object ใหม่ทุก iteration
Benchmark.measure do
  n.times do |i|
    # สร้าง Range ใหม่ทุกครั้ง
    (1..100).each { |j| i + j }
  end
end

# GOOD - สร้าง Range ไว้ข้างนอก
range = (1..100)
Benchmark.measure do
  n.times do |i|
    range.each { |j| i + j }
  end
end

# BAD - Regex compilation ใน loop
data = ["hello@email.com", "invalid", "test@example.org"]
100_000.times do
  data.select { |s| s.match(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i) }
end

# GOOD - Compile regex ครั้งเดียว
EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i.freeze
100_000.times do
  data.select { |s| EMAIL_REGEX.match?(s) }
end
```

### Object Pool Pattern

```ruby
# สำหรับ object ที่ expensive ในการสร้าง
class ConnectionPool
  def initialize(size, &block)
    @pool = Array.new(size, &block)
    @available = @pool.dup
    @mutex = Mutex.new
  end

  def acquire
    connection = nil
    @mutex.synchronize do
      connection = @available.pop
      raise "ไม่มี connection ว่าง" unless connection
    end
    begin
      yield(connection)
    ensure
      @mutex.synchronize { @available.push(connection) }
    end
  end
end

pool = ConnectionPool.new(5) { DatabaseConnection.new }
pool.acquire do |conn|
  conn.query("SELECT * FROM users")
end
```

---

## Step 596: GC Tuning

### เข้าใจ Ruby GC

```ruby
# ดูสถานะ GC
puts GC.stat.inspect

# ผลลัพธ์ (ตัวอย่าง)
# {
#   :count => 15,
#   :heap_allocated_pages => 102,
#   :heap_sorted_length => 102,
#   :heap_allocated_slots => 41616,
#   :heap_live_slots => 29876,
#   :heap_free_slots => 11740,
#   :heap_final_slots => 0,
#   :heap_marked_slots => 20036,
#   :heap_swept_slots => 11740,
#   :heap_eden_pages => 102,
#   ...
# }
```

### GC.compact (Ruby 2.7+)

```ruby
# Compact heap - ย้าย object ที่กระจัดกระจายมารวมกัน
GC.compact

# ผลลัพธ์บอก object ที่ถูก move
# { :considered => 12345, :moved => 5678 }
```

### RUBY_GC Environment Variables

```bash
# ปรับ GC ผ่าน environment variables

# เพิ่ม heap size เพื่อลด GC frequency (ใช้ memory มากขึ้น)
RUBY_GC_HEAP_INIT_SLOTS=800000

# ปัจจัยการขยาย heap
RUBY_GC_HEAP_GROWTH_FACTOR=1.25

# Minimum heap size
RUBY_GC_HEAP_FREE_SLOTS=4096

# ตัวอย่างการตั้งค่าสำหรับ production Rails
export RUBY_GC_HEAP_INIT_SLOTS=600000
export RUBY_GC_HEAP_FREE_SLOTS=600000
export RUBY_GC_HEAP_GROWTH_FACTOR=1.25
export RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR=1.3
```

### GC.disable / GC.enable

```ruby
# ปิด GC ชั่วคราวสำหรับ performance critical section
GC.disable
begin
  # โค้ดที่ critical ด้าน performance
  result = huge_computation()
ensure
  GC.enable
  GC.start  # manual GC
end
```

---

## Step 597: Jemalloc Memory Allocator

### ทำไมต้องใช้ Jemalloc?

Ruby ใช้ system malloc เป็น default ซึ่งอาจมี memory fragmentation สูง Jemalloc ช่วยลด fragmentation และปรับปรุง memory efficiency

```bash
# ติดตั้ง jemalloc บน Ubuntu/Debian
sudo apt-get install libjemalloc-dev

# บน macOS
brew install jemalloc

# Compile Ruby ด้วย jemalloc
rbenv install 3.2.0 RUBY_CONFIGURE_OPTS="--with-jemalloc"

# หรือ run Ruby ด้วย jemalloc (LD_PRELOAD)
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ruby myapp.rb

# ตรวจสอบว่าใช้ jemalloc
ruby -r fiddle -e "puts Fiddle::Handle::DEFAULT['malloc_usable_size'] ? 'using jemalloc' : 'using system malloc'"
```

### วัดผลต่าง

```bash
# ทดสอบ Rails app
# ไม่มี jemalloc
ruby -e "puts 'Memory usage without jemalloc'"
# RSS: 180MB

# มี jemalloc
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ruby -e "puts 'Memory usage with jemalloc'"
# RSS: 120MB (ประมาณ 30-40% ลดลง)
```

---

## Step 598: Benchmark Comparison Examples

### เปรียบเทียบ Data Structures

```ruby
require 'benchmark/ips'
require 'set'

data = (1..10_000).to_a
target = 5_000

Benchmark.ips do |x|
  x.report("Array#include?") { data.include?(target) }
  x.report("Set#include?") do
    set = Set.new(data)
    set.include?(target)
  end

  # Pre-built set
  set = Set.new(data)
  x.report("Set#include? (pre-built)") { set.include?(target) }

  x.compare!
end
```

### String Processing

```ruby
require 'benchmark/ips'

text = "The quick brown fox jumps over the lazy dog " * 1000

Benchmark.ips do |x|
  x.report("gsub with regex") { text.gsub(/[aeiou]/, '') }
  x.report("tr") { text.tr('aeiou', '') }
  x.report("delete") { text.delete('aeiou') }
  x.compare!
end

# tr และ delete เร็วกว่า gsub สำหรับ simple replacement
```

### Numeric Computation

```ruby
require 'benchmark/ips'

Benchmark.ips do |x|
  x.report("Float arithmetic") { 3.14 * 2.72 * 100 }
  x.report("Integer arithmetic") { 314 * 272 * 100 }
  x.report("Bignum") { (10**100) * (10**100) }
  x.compare!
end
```

---

## Step 599-605: Advanced Optimization Techniques

### Database Query Optimization

```ruby
# N+1 Query Problem
# BAD
users = User.all
users.each do |user|
  puts user.posts.count  # Query per user!
end

# GOOD - Eager loading
users = User.includes(:posts)
users.each do |user|
  puts user.posts.size  # ใช้ data ที่ load ไว้แล้ว
end

# Counter Cache
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end

# จากนั้น users จะมี posts_count column
users.each do |user|
  puts user.posts_count  # ไม่ต้อง query เพิ่ม!
end
```

### Background Jobs

```ruby
# แทนที่จะทำงานหนักใน request cycle
class OrdersController < ApplicationController
  def create
    @order = Order.create!(order_params)

    # BAD - ทำใน request thread
    # SendConfirmationEmail.new(@order).deliver
    # GenerateInvoice.new(@order).generate
    # UpdateInventory.new(@order).update

    # GOOD - ทำใน background
    OrderConfirmationJob.perform_later(@order.id)
    InvoiceGenerationJob.perform_later(@order.id)
    InventoryUpdateJob.perform_later(@order.id)

    redirect_to @order
  end
end
```

### Caching Strategies

```ruby
# Fragment caching ใน Rails
# app/views/products/index.html.erb
# <% cache @products do %>
#   ...product list...
# <% end %>

# Russian Doll Caching
# <% cache @product do %>
#   <%= @product.name %>
#   <% cache @product.reviews do %>
#     ... reviews ...
#   <% end %>
# <% end %>

# Low-level caching
class ProductService
  def featured_products
    Rails.cache.fetch("featured_products", expires_in: 1.hour) do
      Product.featured.includes(:category).limit(10).to_a
    end
  end

  def invalidate_cache
    Rails.cache.delete("featured_products")
  end
end
```

---

## แบบฝึกหัด 20 ข้อ

### ข้อที่ 1: Benchmark การเรียง Array
เขียน benchmark เปรียบเทียบวิธีการเรียงลำดับ

**เฉลย:**
```ruby
require 'benchmark'

n = 10_000
data = n.times.map { rand(10_000) }

Benchmark.bm(30) do |x|
  x.report("Array#sort:") { data.sort }
  x.report("Array#sort_by (numeric):") { data.sort_by { |n| n } }
  x.report("Array#sort with block:") { data.sort { |a, b| a <=> b } }
  x.report("Bubble sort (O(n²)):") do
    arr = data.dup
    n = arr.length
    n.times { |i| (n-i-1).times { |j| arr[j], arr[j+1] = arr[j+1], arr[j] if arr[j] > arr[j+1] } }
  end
end
```

### ข้อที่ 2: Memory Profile String Operations
```ruby
require 'memory_profiler'

report = MemoryProfiler.report do
  # Method 1: String +
  1000.times do
    result = ""
    100.times { |i| result = result + "item #{i}, " }
  end
end
report.pretty_print

report2 = MemoryProfiler.report do
  # Method 2: Array join
  1000.times do
    parts = 100.times.map { |i| "item #{i}" }
    result = parts.join(", ")
  end
end
report2.pretty_print
```

### ข้อที่ 3: Optimize N+1 Query
```ruby
# เฉลย
# ก่อน optimize
def slow_user_summary
  User.all.map do |user|
    {
      name: user.name,
      post_count: user.posts.count,        # N queries
      comment_count: user.comments.count   # N queries
    }
  end
end

# หลัง optimize
def fast_user_summary
  User.includes(:posts, :comments).map do |user|
    {
      name: user.name,
      post_count: user.posts.size,
      comment_count: user.comments.size
    }
  end
end
```

### ข้อที่ 4: ใช้ Set สำหรับ Lookup
```ruby
# เฉลย
require 'benchmark'
require 'set'

banned_words = File.read('/usr/share/dict/words').split("\n").first(10000)
banned_set = Set.new(banned_words)

test_words = ["hello", "ruby", "programming", "badword"] * 1000

Benchmark.bm(20) do |x|
  x.report("Array include?:") { test_words.each { |w| banned_words.include?(w) } }
  x.report("Set include?:") { test_words.each { |w| banned_set.include?(w) } }
end
```

### ข้อที่ 5: Memoize Fibonacci
```ruby
# เฉลย
class FibCalculator
  def initialize
    @cache = {}
  end

  def fib(n)
    return n if n <= 1
    @cache[n] ||= fib(n - 1) + fib(n - 2)
  end
end

require 'benchmark'

calc = FibCalculator.new

Benchmark.bm do |x|
  x.report("fib(40) first call:") { calc.fib(40) }
  x.report("fib(40) cached:") { calc.fib(40) }
end
```

### ข้อที่ 6: Lazy vs Eager Evaluation
```ruby
# เฉลย
require 'benchmark'

n = 1_000_000

Benchmark.bm(20) do |x|
  x.report("Eager:") do
    (1..n).select { |x| x % 3 == 0 }.map { |x| x ** 2 }.first(10)
  end

  x.report("Lazy:") do
    (1..n).lazy.select { |x| x % 3 == 0 }.map { |x| x ** 2 }.first(10)
  end
end
```

### ข้อที่ 7: Frozen String Performance
```ruby
# ก่อน
def slow_greeting(name)
  greeting = "Hello, " + name + "!"
  greeting
end

# หลัง (frozen_string_literal + interpolation)
# frozen_string_literal: true

def fast_greeting(name)
  "Hello, #{name}!"
end

require 'benchmark/ips'
Benchmark.ips do |x|
  x.report("String +:") { slow_greeting("Ruby") }
  x.report("Interpolation:") { fast_greeting("Ruby") }
  x.compare!
end
```

### ข้อที่ 8: Object Allocation Reduction
```ruby
# ลด object allocation ใน loop
require 'benchmark'

n = 1_000_000

Benchmark.bm(30) do |x|
  x.report("Create regex in loop:") do
    n.times { |i| "user_#{i}".match(/user_\d+/) }
  end

  x.report("Regex outside loop:") do
    pattern = /user_\d+/
    n.times { |i| "user_#{i}".match(pattern) }
  end

  x.report("match? (no MatchData):") do
    pattern = /user_\d+/
    n.times { |i| "user_#{i}".match?(pattern) }
  end
end
```

### ข้อที่ 9: Hash vs Case/When
```ruby
require 'benchmark/ips'

# Lookup table vs case/when
STATUS_LABELS = {
  0 => "pending",
  1 => "active",
  2 => "inactive",
  3 => "banned"
}.freeze

def label_case(status)
  case status
  when 0 then "pending"
  when 1 then "active"
  when 2 then "inactive"
  when 3 then "banned"
  end
end

def label_hash(status)
  STATUS_LABELS[status]
end

Benchmark.ips do |x|
  x.report("case/when:") { label_case(2) }
  x.report("hash lookup:") { label_hash(2) }
  x.compare!
end
```

### ข้อที่ 10: Profiling Rails Controller
```ruby
# เฉลย - วิธี profile Rails action

# ใน config/environments/development.rb
# config.middleware.insert_before 0, StackProf::Middleware, enabled: true, mode: :wall

# หรือใน controller
class ProductsController < ApplicationController
  def index
    require 'stackprof'

    result = StackProf.run(mode: :cpu, out: '/tmp/stackprof.dump') do
      @products = Product.includes(:category, :images)
                        .where(active: true)
                        .order(created_at: :desc)
                        .page(params[:page])
    end

    render :index
  end
end

# วิเคราะห์ด้วย
# stackprof /tmp/stackprof.dump --text
```

### ข้อที่ 11: เปรียบเทียบ each_with_object vs inject
```ruby
require 'benchmark/ips'

data = (1..1000).to_a

Benchmark.ips do |x|
  x.report("inject with +:") do
    data.inject([]) { |arr, n| arr + [n * 2] }
  end

  x.report("each_with_object:") do
    data.each_with_object([]) { |n, arr| arr << n * 2 }
  end

  x.report("map:") do
    data.map { |n| n * 2 }
  end

  x.compare!
end
```

### ข้อที่ 12: Database Index Optimization
```ruby
# Migration ที่เพิ่ม index
class AddIndexesToPosts < ActiveRecord::Migration[7.0]
  def change
    # เพิ่ม index บน column ที่ query บ่อย
    add_index :posts, :user_id
    add_index :posts, :created_at
    add_index :posts, [:status, :created_at]  # composite index
    add_index :posts, :title  # สำหรับ full-text search แบบง่าย

    # สำหรับ polymorphic associations
    add_index :comments, [:commentable_type, :commentable_id]
  end
end
```

### ข้อที่ 13: Caching ด้วย Redis
```ruby
# เฉลย
class UserDashboardService
  def initialize(user)
    @user = user
  end

  def statistics
    Rails.cache.fetch(cache_key, expires_in: 30.minutes) do
      {
        total_posts: @user.posts.count,
        total_comments: @user.comments.count,
        total_likes: @user.liked_posts.count,
        recent_activity: fetch_recent_activity
      }
    end
  end

  def invalidate!
    Rails.cache.delete(cache_key)
  end

  private

  def cache_key
    "user_dashboard/#{@user.id}/#{@user.updated_at.to_i}"
  end

  def fetch_recent_activity
    @user.activities.recent.limit(10).to_a
  end
end
```

### ข้อที่ 14: GC Profiling
```ruby
# เฉลย
GC::Profiler.enable

# รันโค้ด
1_000_000.times do
  obj = Object.new
  str = "hello #{obj.object_id}"
end

GC.start
puts GC::Profiler.result
# แสดงจำนวน GC runs และเวลาที่ใช้

GC::Profiler.disable
GC::Profiler.clear
```

### ข้อที่ 15: Background Processing Pattern
```ruby
# เฉลย
class ReportGenerator
  include Sidekiq::Worker

  sidekiq_options queue: 'reports', retry: 3

  def perform(user_id, params)
    user = User.find(user_id)
    report = generate_report(user, params)
    save_report(user, report)
    notify_user(user, report)
  end

  private

  def generate_report(user, params)
    # Complex calculation
    {
      summary: calculate_summary(user, params),
      details: calculate_details(user, params)
    }
  end
end

# Enqueue
ReportGenerator.perform_async(current_user.id, report_params)
```

### ข้อที่ 16-20: Final Exercises

**ข้อที่ 16:** สร้าง Benchmark suite สำหรับ string interpolation ทุกรูปแบบ
```ruby
# เฉลย: ทดสอบ sprintf, format, interpolation, %, concat
require 'benchmark/ips'

Benchmark.ips do |x|
  name, num = "Ruby", 3.2
  x.report("interpolation") { "Hello #{name} #{num}" }
  x.report("sprintf") { sprintf("Hello %s %.1f", name, num) }
  x.report("format") { format("Hello %s %.1f", name, num) }
  x.report("% operator") { "Hello %s %.1f" % [name, num] }
  x.compare!
end
```

**ข้อที่ 17:** Implement LRU Cache
```ruby
# เฉลย
class LRUCache
  def initialize(capacity)
    @capacity = capacity
    @cache = {}
    @order = []
  end

  def get(key)
    return nil unless @cache.key?(key)
    @order.delete(key)
    @order.push(key)
    @cache[key]
  end

  def put(key, value)
    if @cache.key?(key)
      @order.delete(key)
    elsif @cache.size >= @capacity
      oldest = @order.shift
      @cache.delete(oldest)
    end
    @cache[key] = value
    @order.push(key)
  end
end

cache = LRUCache.new(3)
cache.put(:a, 1)
cache.put(:b, 2)
cache.put(:c, 3)
cache.get(:a)         # => 1
cache.put(:d, 4)      # evicts :b
puts cache.get(:b)    # => nil
puts cache.get(:d)    # => 4
```

**ข้อที่ 18:** Profiling Rails Mailer Performance
```ruby
# เฉลย
# Benchmark email generation
require 'benchmark'

Benchmark.bm do |x|
  x.report("Simple email:") do
    100.times { OrderMailer.confirmation(Order.first).message }
  end
  x.report("Complex email:") do
    100.times { ReportMailer.weekly_report(User.first).message }
  end
end
```

**ข้อที่ 19:** SQL Query Optimization
```ruby
# เฉลย
# Bad: Multiple queries
def bad_summary
  total = Order.where(status: 'completed').count
  amount = Order.where(status: 'completed').sum(:total)
  { count: total, amount: amount }
end

# Good: Single query
def good_summary
  result = Order.where(status: 'completed')
                .select("COUNT(*) as count, SUM(total) as total_amount")
                .first
  { count: result.count, amount: result.total_amount }
end

# Even better with group
def summary_by_status
  Order.group(:status)
       .select("status, COUNT(*) as count, SUM(total) as total")
       .each_with_object({}) do |row, h|
         h[row.status] = { count: row.count, total: row.total }
       end
end
```

**ข้อที่ 20:** สร้าง Performance Report Generator
```ruby
# เฉลย
class PerformanceReport
  def initialize
    @metrics = {}
  end

  def measure(name, &block)
    before_memory = GC.stat[:heap_live_slots]
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    result = block.call

    elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    after_memory = GC.stat[:heap_live_slots]

    @metrics[name] = {
      time: elapsed,
      memory_delta: after_memory - before_memory
    }

    result
  end

  def report
    puts "\n=== Performance Report ==="
    @metrics.each do |name, data|
      puts "#{name}:"
      puts "  Time: #{(data[:time] * 1000).round(3)}ms"
      puts "  Objects created: #{data[:memory_delta]}"
    end
    puts "=========================="
  end
end

reporter = PerformanceReport.new

reporter.measure("Database query") { User.includes(:posts).limit(100).to_a }
reporter.measure("String processing") { 10_000.times.map { |i| "item_#{i}".upcase } }
reporter.measure("Array sort") { (1..10_000).to_a.shuffle.sort }

reporter.report
```

---

## สรุป

Performance Optimization ใน Ruby:

1. **วัดก่อน optimize** - ใช้ Benchmark และ profiler เสมอ
2. **Algorithm มีผลมากกว่าการ micro-optimize** - O(n²) ไม่ดีกว่า O(n) ไม่ว่าจะ tune แค่ไหน
3. **หลีกเลี่ยง N+1 queries** - bottleneck ที่พบบ่อยที่สุดใน Rails
4. **ใช้ caching ให้ถูกที่** - cache แต่อย่า over-cache
5. **Lazy evaluation** สำหรับ large datasets
6. **frozen_string_literal** ช่วย memory ได้เยอะ

---

*ต่อไป: ตอนที่ 28 - Debugging เชิงลึก*
