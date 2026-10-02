# ตอนที่ 27: Performance Optimization (ขั้นตอนที่ 586-605)

Performance Optimization คือกระบวนการทำให้โปรแกรมทำงานเร็วขึ้น ใช้ Memory น้อยลง หรือมีประสิทธิภาพมากขึ้น สิ่งสำคัญคือต้อง **วัดก่อน Optimize** เสมอ

---

## ขั้นตอนที่ 586: Benchmarking ด้วย Benchmark Module

### Benchmark พื้นฐาน

```ruby
require 'benchmark'

# วัด Execution Time
Benchmark.bm do |x|
  x.report("sort:") do
    data = (1..100_000).to_a.shuffle
    data.sort
  end
  
  x.report("sort!:") do
    data = (1..100_000).to_a.shuffle
    data.sort!
  end
end

# Output:
#       user     system      total        real
# sort:  0.054876   0.000000   0.054876 (  0.055238)
# sort!: 0.038591   0.000000   0.038591 (  0.038843)

# bmbm - เพิ่ม Rehearsal เพื่อ Warmup
Benchmark.bmbm do |x|
  x.report("string concat:") do
    result = ""
    10_000.times { result += "a" }
  end
  
  x.report("string shovel:") do
    result = ""
    10_000.times { result << "a" }
  end
  
  x.report("string join:") do
    parts = []
    10_000.times { parts << "a" }
    result = parts.join
  end
end

# ips - Iterations Per Second (ต้องติดตั้ง benchmark-ips)
require 'benchmark/ips'

Benchmark.ips do |x|
  x.report("symbol:") { :hello }
  x.report("string:") { "hello" }
  
  x.compare!
end

# Output:
# Comparison:
#    symbol:  28561234.5 i/s
#    string:  15234123.4 i/s - 1.88x  slower
```

### เปรียบเทียบ Algorithms

```ruby
require 'benchmark'

def linear_search(arr, target)
  arr.each_with_index do |val, i|
    return i if val == target
  end
  -1
end

def binary_search(sorted_arr, target)
  low, high = 0, sorted_arr.length - 1
  
  while low <= high
    mid = (low + high) / 2
    if sorted_arr[mid] == target
      return mid
    elsif sorted_arr[mid] < target
      low = mid + 1
    else
      high = mid - 1
    end
  end
  -1
end

n   = 1_000_000
arr = (1..n).to_a
target = n / 2

Benchmark.bm(15) do |x|
  x.report("linear search:") do
    1000.times { linear_search(arr, target) }
  end
  
  x.report("binary search:") do
    1000.times { binary_search(arr, target) }
  end
  
  x.report("include?:") do
    1000.times { arr.include?(target) }
  end
end
```

---

## ขั้นตอนที่ 587: Profiling ด้วย ruby-prof

```bash
gem install ruby-prof
```

```ruby
require 'ruby-prof'

# Profile Code Block
result = RubyProf.profile do
  # โค้ดที่ต้องการ Profile
  1000.times do
    data = Array.new(1000) { rand(1000) }
    data.sort.first(10)
  end
end

# Print Result
printer = RubyProf::FlatPrinter.new(result)
printer.print(STDOUT)

# หรือ Graph Printer (แสดง Call Graph)
printer = RubyProf::GraphPrinter.new(result)
printer.print(STDOUT)

# HTML Printer
File.open('/tmp/profile.html', 'w') do |file|
  printer = RubyProf::GraphHtmlPrinter.new(result)
  printer.print(file)
end

# Output ตัวอย่าง:
# %self      total      self      wait     child     calls  name
# 45.23      0.232     0.232     0.000     0.000     1000   Array#sort
# 23.11      0.119     0.119     0.000     0.000    1000000 Kernel#rand
```

### Stackprof (Sampling Profiler)

```bash
gem install stackprof
```

```ruby
require 'stackprof'

# Sampling Profile
StackProf.run(mode: :cpu, out: 'tmp/stackprof.dump', interval: 1000) do
  # Code to profile
  ProcessData.new.run
end

# ดูผลลัพธ์
# stackprof tmp/stackprof.dump --text
# stackprof tmp/stackprof.dump --flamegraph > /tmp/flamegraph.html
```

---

## ขั้นตอนที่ 588: Memory Optimization

### ObjectSpace สำหรับ Memory Analysis

```ruby
require 'objspace'

# ดูจำนวน Object ทั้งหมด
puts ObjectSpace.count_objects

# ดู Live Objects ตาม Type
puts ObjectSpace.count_objects[:T_STRING]  # String objects
puts ObjectSpace.count_objects[:T_ARRAY]   # Array objects
puts ObjectSpace.count_objects[:T_HASH]    # Hash objects

# Trace Memory Allocations
require 'objspace'

ObjectSpace.trace_object_allocations_start

def create_many_strings
  100.times.map { "hello" * 1000 }
end

result = create_many_strings

ObjectSpace.trace_object_allocations_stop

# ดู Allocation ของ Object ที่เจาะจง
result.first(5).each do |str|
  puts "#{str.length} chars, allocated at #{ObjectSpace.allocation_sourcefile(str)}:#{ObjectSpace.allocation_sourceline(str)}"
end

# Memory Size
GC.compact  # Compact heap
puts ObjectSpace.memsize_of("hello")       # Size ของ String
puts ObjectSpace.memsize_of([1, 2, 3])     # Size ของ Array
puts ObjectSpace.memsize_of({ a: 1 })      # Size ของ Hash
```

### ลด Memory Allocation

```ruby
# ตัวอย่าง: String สร้างบ่อยมาก

# Bad: สร้าง String ใหม่ทุกครั้ง
def bad_method
  100.times do
    # String ใหม่ถูกสร้างทุก iteration
    user_type = "admin"
    process(user_type)
  end
end

# Good: ใช้ Symbol หรือ Frozen String
USER_TYPE = "admin".freeze

def good_method
  100.times do
    process(USER_TYPE)  # Reference เดิม
  end
end

# Frozen String Literal
# # frozen_string_literal: true

# String ทั้งหมดในไฟล์จะเป็น Frozen

# ลด Array/Hash Allocation
# Bad
def process_items(items)
  result = []
  items.each { |item| result << item * 2 }
  result
end

# Good: ใช้ map แทน
def process_items(items)
  items.map { |item| item * 2 }
end

# Avoid Object Creation ใน Hot Loops
# Bad: สร้าง Regex ใหม่ทุกครั้ง
def valid_email?(email)
  email.match?(/\A[^@\s]+@[^@\s]+\z/)  # Regex compiled ทุกครั้ง!
end

# Good: Cache Regex
EMAIL_REGEX = /\A[^@\s]+@[^@\s]+\z/

def valid_email?(email)
  email.match?(EMAIL_REGEX)
end
```

---

## ขั้นตอนที่ 589: GC Tuning

### ทำความเข้าใจ Ruby GC

```ruby
# ดูข้อมูล GC
puts GC.stat.inspect

# ตัวอย่าง Output:
# {
#   count: 150,               # GC Runs
#   heap_allocated_pages: 520,
#   heap_sorted_length: 520,
#   heap_allocatable_pages: 0,
#   heap_available_slots: 212018,
#   heap_live_slots: 195132,
#   heap_free_slots: 16886,
#   heap_final_slots: 0,
#   heap_marked_slots: 193985,
#   heap_eden_pages: 520,
#   heap_tomb_pages: 0,
#   total_allocated_pages: 521,
#   total_freed_pages: 1,
#   total_allocated_objects: 5234871,
#   total_freed_objects: 5039739,
#   malloc_increase_bytes: 1049684,
#   ...
# }

# GC Profiling
GC::Profiler.enable

# Code ที่ต้องการดู
1000.times do
  Array.new(1000) { Object.new }
end

puts GC::Profiler.report
GC::Profiler.disable
```

### Environment Variables สำหรับ GC Tuning

```bash
# Heap Initial Size (Default: 8MB)
RUBY_GC_HEAP_INIT_SLOTS=10000

# Growth Factor
RUBY_GC_HEAP_GROWTH_FACTOR=1.8

# Free Slots Threshold
RUBY_GC_HEAP_FREE_SLOTS=4096

# Malloc Limit
RUBY_GC_MALLOC_LIMIT=67108864  # 64MB

# Oldmalloc Limit
RUBY_GC_OLDMALLOC_LIMIT=134217728  # 128MB

# ตัวอย่างการตั้งค่าสำหรับ Rails App ขนาดใหญ่
export RUBY_GC_HEAP_INIT_SLOTS=600000
export RUBY_GC_HEAP_FREE_SLOTS=600000
export RUBY_GC_HEAP_GROWTH_FACTOR=1.25
export RUBY_GC_HEAP_GROWTH_MAX_SLOTS=300000
export RUBY_GC_MALLOC_LIMIT=90000000
export RUBY_GC_OLDMALLOC_LIMIT=90000000
```

---

## ขั้นตอนที่ 590: String Optimization

### Frozen String Literals

```ruby
# # frozen_string_literal: true
# ใส่ที่บรรทัดแรกของทุกไฟล์

# ทำงานอย่างไร
str = "hello"  # Frozen ทันที
str << " world"  # FrozenError!

# สำหรับ String ที่ต้องเปลี่ยน ให้ +
mutable = +"hello"  # Mutable String
mutable << " world"  # OK

# String ทั่วไป vs Frozen
require 'benchmark'

n = 1_000_000

Benchmark.bm do |x|
  x.report("frozen:") do
    n.times { "frozen string".freeze }
  end
  
  x.report("regular:") do
    n.times { "regular string" }
  end
end

# String Building
# Bad: String Concatenation (O(n²))
def bad_string_build(n)
  result = ""
  n.times { result += "x" }
  result
end

# Good: Shovel Operator (O(n))
def good_string_build(n)
  result = ""
  n.times { result << "x" }
  result
end

# Best: Array + Join
def best_string_build(n)
  parts = []
  n.times { parts << "x" }
  parts.join
end

# หรือ each_with_object
def each_with_obj_build(n)
  n.times.each_with_object("") { |_, s| s << "x" }
end

# String Format
# ช้ากว่า
name = "สมชาย"
result = "สวัสดี " + name + " !"

# เร็วกว่า
result = "สวัสดี #{name} !"

# เร็วที่สุดสำหรับ Template ซ้ำๆ
GREETING_TEMPLATE = "สวัสดี %s !"
result = GREETING_TEMPLATE % name
```

---

## ขั้นตอนที่ 591: Algorithm Complexity (Big O)

### Big O Notation

```ruby
# O(1) - Constant Time
def get_first(arr)
  arr[0]
end

def hash_lookup(hash, key)
  hash[key]
end

# O(log n) - Logarithmic Time
def binary_search(arr, target)
  low, high = 0, arr.length - 1
  while low <= high
    mid = (low + high) / 2
    return mid if arr[mid] == target
    arr[mid] < target ? (low = mid + 1) : (high = mid - 1)
  end
  -1
end

# O(n) - Linear Time
def linear_search(arr, target)
  arr.each_with_index { |val, i| return i if val == target }
  -1
end

def sum(arr)
  arr.sum
end

# O(n log n) - Log-linear Time
def merge_sort(arr)
  return arr if arr.length <= 1
  
  mid   = arr.length / 2
  left  = merge_sort(arr[0...mid])
  right = merge_sort(arr[mid..])
  
  merge(left, right)
end

def merge(left, right)
  result = []
  while !left.empty? && !right.empty?
    result << (left.first <= right.first ? left.shift : right.shift)
  end
  result + left + right
end

# O(n²) - Quadratic Time
def bubble_sort(arr)
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

# ตัวอย่าง: Find Duplicates
data = Array.new(10_000) { rand(20_000) }

# O(n²) - Bad
def find_duplicates_slow(arr)
  duplicates = []
  arr.each_with_index do |val, i|
    arr.each_with_index do |other, j|
      duplicates << val if i != j && val == other
    end
  end
  duplicates.uniq
end

# O(n) - Good
def find_duplicates_fast(arr)
  seen = {}
  duplicates = []
  arr.each do |val|
    duplicates << val if seen[val]
    seen[val] = true
  end
  duplicates.uniq
end

require 'benchmark'
Benchmark.bm do |x|
  x.report("O(n²):") { find_duplicates_slow(data) }
  x.report("O(n):") { find_duplicates_fast(data) }
end
```

---

## ขั้นตอนที่ 592: Memoization และ Caching

```ruby
# Memoization ระดับ Method
class PriceCalculator
  def initialize(products)
    @products = products
    @cache    = {}
  end
  
  def total_revenue(category = nil)
    cache_key = category || :all
    @cache[cache_key] ||= calculate_revenue(category)
  end
  
  def average_price(category = nil)
    cache_key = "avg_#{category || :all}"
    @cache[cache_key] ||= calculate_average(category)
  end
  
  def invalidate_cache!
    @cache.clear
  end
  
  private
  
  def calculate_revenue(category)
    products = category ? @products.select { |p| p[:category] == category } : @products
    products.sum { |p| p[:price] * p[:quantity] }
  end
  
  def calculate_average(category)
    products = category ? @products.select { |p| p[:category] == category } : @products
    return 0 if products.empty?
    products.sum { |p| p[:price] } / products.size.to_f
  end
end

# Rails Caching
class ProductsController < ApplicationController
  def index
    @products = Rails.cache.fetch('all_products', expires_in: 1.hour) do
      Product.includes(:category).active.order(:name).to_a
    end
  end
  
  def expensive_report
    @stats = Rails.cache.fetch(
      "product_stats_#{Date.today}",
      expires_in: 24.hours
    ) do
      {
        total:    Product.count,
        by_category: Product.group(:category).count,
        avg_price:   Product.average(:price),
        top_sellers: Product.order(sales: :desc).limit(10).to_a
      }
    end
  end
end

# Fragment Caching ใน Rails Views
# <% cache @product do %>
#   <%= render @product %>
# <% end %>

# Russian Doll Caching
# <% cache @user do %>
#   <%= @user.name %>
#   <% cache @user.orders do %>
#     <%= render @user.orders %>
#   <% end %>
# <% end %>
```

---

## ขั้นตอนที่ 593: Lazy Evaluation

```ruby
# Lazy Enumerator ลด Memory Usage
require 'benchmark'

n = 10_000_000

# Eager - โหลด Memory ทั้งหมด
Benchmark.bm do |x|
  x.report("eager:") do
    (1..n).select { |x| x % 2 == 0 }.map { |x| x ** 2 }.first(10)
  end
  
  x.report("lazy:") do
    (1..n).lazy.select { |x| x % 2 == 0 }.map { |x| x ** 2 }.first(10)
  end
end

# File Processing with Lazy
def process_log_file(filepath, pattern, limit = 100)
  File.each_line(filepath)
    .lazy
    .map(&:chomp)
    .select { |line| line.match?(pattern) }
    .map { |line| parse_log_line(line) }
    .reject { |entry| entry.nil? }
    .first(limit)
end

# Lazy + Infinite Sequence
def primes
  Enumerator.new do |yielder|
    primes_found = []
    n = 2
    loop do
      if primes_found.none? { |p| n % p == 0 }
        primes_found << n
        yielder << n
      end
      n += 1
    end
  end.lazy
end

puts primes.first(10).inspect
# [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]

puts primes.select { |p| p > 100 }.first(5).inspect
# [101, 103, 107, 109, 113]
```

---

## ขั้นตอนที่ 594: Database Query Optimization

```ruby
# N+1 Query Problem
# Bad - N+1 Queries
def list_orders_bad
  Order.all.each do |order|
    puts "#{order.id}: #{order.user.name}"  # User query ทุก iteration!
  end
end

# Good - Eager Loading
def list_orders_good
  Order.includes(:user).each do |order|
    puts "#{order.id}: #{order.user.name}"  # ใช้ Preloaded User
  end
end

# More complex associations
Order.includes(:user, items: :product).all.each do |order|
  order.items.each do |item|
    puts "#{item.product.name}: #{item.quantity}"
  end
end

# Select เฉพาะ Columns ที่ต้องการ
# Bad
User.all.each { |u| puts u.email }  # โหลดทุก Column

# Good
User.select(:id, :email).each { |u| puts u.email }

# Pluck - ดึงค่า Specific Column
emails = User.pluck(:email)         # [String, String, ...]
ids    = User.pluck(:id)            # [Integer, Integer, ...]
pairs  = User.pluck(:id, :email)    # [[id, email], ...]

# Counter Cache
class Order < ApplicationRecord
  belongs_to :user, counter_cache: true
end

# แทนที่จะ user.orders.count (Query)
# ใช้ user.orders_count (ใช้ counter cache)

# Bulk Operations
# Bad - N Queries
users.each do |user|
  user.update!(status: :active)
end

# Good - Single Query
User.where(id: users.map(&:id)).update_all(status: :active)

# Bulk Insert
# ActiveRecord 6+
User.insert_all([
  { name: 'สมชาย', email: 'a@example.com' },
  { name: 'สมหญิง', email: 'b@example.com' },
  { name: 'มานี',   email: 'c@example.com' }
])

# Upsert
User.upsert_all(
  [{ id: 1, name: 'อัพเดท' }, { id: 999, name: 'ใหม่' }],
  unique_by: :id
)

# Indexes
# db/migrate
class AddIndexesToOrders < ActiveRecord::Migration[7.1]
  def change
    add_index :orders, :user_id           # Foreign Key Index
    add_index :orders, :status            # Filter Index
    add_index :orders, [:user_id, :status] # Composite Index
    add_index :orders, :created_at        # Sort Index
    
    # Partial Index (ประหยัดพื้นที่)
    add_index :orders, :user_id,
      where: "status = 'pending'",
      name: 'index_orders_pending'
  end
end
```

---

## ขั้นตอนที่ 595: Jemalloc

### ทำไมต้อง Jemalloc?

Jemalloc เป็น Memory Allocator ที่มีประสิทธิภาพสูงกว่า glibc malloc ช่วยลด Memory Fragmentation

```bash
# ติดตั้ง Jemalloc
# Ubuntu/Debian
apt-get install libjemalloc-dev

# macOS
brew install jemalloc

# ตรวจสอบว่า Ruby ใช้ Jemalloc
ruby -e "require 'rbconfig'; puts RbConfig::CONFIG['LIBS']"

# รัน Ruby ด้วย Jemalloc
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ruby app.rb

# สำหรับ Rails
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 bundle exec rails server

# Dockerfile
FROM ruby:3.2.2
RUN apt-get update && apt-get install -y libjemalloc2
ENV LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2
```

---

## ขั้นตอนที่ 596-600: Advanced Optimization

### Object Pooling

```ruby
# Object Pool ลดการสร้าง/ทำลาย Object ซ้ำๆ

class ConnectionPool
  def initialize(size, &factory)
    @size     = size
    @factory  = factory
    @pool     = Array.new(size) { factory.call }
    @mutex    = Mutex.new
    @available = @pool.dup
  end
  
  def checkout(timeout: 5)
    deadline = Time.now + timeout
    
    loop do
      @mutex.synchronize do
        unless @available.empty?
          return @available.pop
        end
      end
      
      raise "Connection Timeout" if Time.now > deadline
      sleep(0.01)
    end
  end
  
  def checkin(connection)
    @mutex.synchronize { @available.push(connection) }
  end
  
  def with_connection
    conn = checkout
    begin
      yield conn
    ensure
      checkin(conn)
    end
  end
end

# การใช้งาน
db_pool = ConnectionPool.new(10) { DatabaseConnection.new }

threads = 20.times.map do |i|
  Thread.new do
    db_pool.with_connection do |conn|
      result = conn.query("SELECT * FROM users LIMIT 10")
      puts "Thread #{i}: #{result.size} rows"
    end
  end
end

threads.each(&:join)
```

### Flyweight Pattern

```ruby
# Flyweight ลด Memory โดย Share Immutable State
class CharacterStyle
  POOL = {}
  
  attr_reader :font, :size, :color, :bold, :italic
  
  def self.create(font:, size:, color:, bold: false, italic: false)
    key = [font, size, color, bold, italic]
    POOL[key] ||= new(font: font, size: size, color: color, bold: bold, italic: italic)
  end
  
  def initialize(font:, size:, color:, bold:, italic:)
    @font   = font
    @size   = size
    @color  = color
    @bold   = bold
    @italic = italic
    freeze
  end
  
  def to_s
    "#{@font} #{@size}px #{@color}#{@bold ? ' bold' : ''}#{@italic ? ' italic' : ''}"
  end
end

class Character
  attr_reader :char, :style
  
  def initialize(char, style)
    @char  = char
    @style = style
  end
end

# ใน Document ขนาดใหญ่
# แทนที่จะสร้าง Style Object สำหรับทุก Character
# ใช้ Share Style Object แทน

normal_style = CharacterStyle.create(font: 'Sarabun', size: 12, color: 'black')
bold_style   = CharacterStyle.create(font: 'Sarabun', size: 12, color: 'black', bold: true)

# Characters แชร์ Style Objects
document = []
"สวัสดีโลก".each_char { |c| document << Character.new(c, normal_style) }
"สำคัญมาก".each_char  { |c| document << Character.new(c, bold_style) }

puts CharacterStyle::POOL.size  # 2 Style Objects สำหรับทุก Character
puts document.size              # 15 Characters แต่ใช้ 2 Style Objects
```

---

## แบบฝึกหัดบทที่ 27 (20 ข้อ)

**ข้อ 1:** ใช้ Benchmark เปรียบเทียบ `map` vs `each_with_object` vs `inject` สำหรับสร้าง Hash

**ข้อ 2:** Benchmark การใช้ String Concatenation 3 วิธี: `+`, `<<`, `join`

**ข้อ 3:** วัด Memory Usage ก่อน/หลัง Optimization ด้วย `ObjectSpace`

**ข้อ 4:** ใช้ `ruby-prof` Profile Rails Controller Action และหา Bottleneck

**ข้อ 5:** เปรียบเทียบ Eager Loading (`includes`) vs ไม่ใช้ สำหรับ N+1 Query

**ข้อ 6:** Implement Memoization ที่ Thread-safe สำหรับ Expensive Calculation

**ข้อ 7:** สร้าง Benchmark Test สำหรับ Binary Search vs Linear Search

**ข้อ 8:** ใช้ Lazy Enumerable ประมวลผล 1 Million Records โดยใช้ Memory น้อยที่สุด

**ข้อ 9:** Optimize SQL Queries สำหรับ Rails App โดยเพิ่ม Indexes และ Eager Loading

**ข้อ 10:** Implement Connection Pool สำหรับ Database Connections

**ข้อ 11:** เปรียบเทียบ `frozen_string_literal: true` กับ Regular Strings

**ข้อ 12:** ใช้ `stackprof` วิเคราะห์ CPU Usage ของ Algorithm

**ข้อ 13:** Implement Object Pool สำหรับ Expensive Objects

**ข้อ 14:** สร้าง Cache Layer ด้วย Redis สำหรับ Rails API

**ข้อ 15:** เปรียบเทียบ `Array.include?` vs `Set.include?` สำหรับ Large Collections

**ข้อ 16:** Optimize Text Processing ด้วย Regular Expressions ที่มีประสิทธิภาพ

**ข้อ 17:** ใช้ `bulk_insert` และ `update_all` แทน Individual Operations

**ข้อ 18:** Implement Flyweight Pattern สำหรับ Repeated Objects

**ข้อ 19:** สร้าง Memory-efficient CSV Parser ด้วย Streaming

**ข้อ 20:** วิเคราะห์และ Optimize Rails App ให้ Response Time ลดลง 50%

---

### เฉลยตัวอย่าง ข้อ 15: Array vs Set

```ruby
require 'benchmark'
require 'set'

sizes = [100, 1_000, 10_000, 100_000]

sizes.each do |size|
  arr = (1..size).to_a.shuffle
  set = Set.new(arr)
  
  target = size / 2
  n = 10_000
  
  puts "\n=== Size: #{size} ==="
  Benchmark.bm(15) do |x|
    x.report("Array#include?:") do
      n.times { arr.include?(target) }
    end
    
    x.report("Set#include?:") do
      n.times { set.include?(target) }
    end
    
    x.report("Array#member?:") do
      n.times { arr.member?(target) }
    end
  end
end

# Output (เปรียบเทียบ):
# === Size: 100 ===
#                       user     system      total        real
# Array#include?:   0.001234   0.000000   0.001234 (  0.001234)
# Set#include?:     0.000432   0.000000   0.000432 (  0.000432)
#
# === Size: 100000 ===
#                       user     system      total        real
# Array#include?:   5.234234   0.000000   5.234234 (  5.234234)
# Set#include?:     0.000456   0.000000   0.000456 (  0.000456)

# Set: O(1) lookup
# Array: O(n) lookup

# เมื่อต้องการ Lookup บ่อยๆ ให้แปลงเป็น Set
def process_users(users, blocked_ids)
  blocked_set = Set.new(blocked_ids)  # สร้างครั้งเดียว
  
  users.reject { |user| blocked_set.include?(user[:id]) }
end

# Array Membership Test สำหรับ Large Collection
users      = (1..10_000).map { |i| { id: i, name: "User #{i}" } }
blocked_ids = (1..5_000).to_a.shuffle.first(500)

puts "Using Array (slow):"
t = Time.now
result = process_users(users, blocked_ids)
puts "Time: #{Time.now - t:.4f}s"

puts "Using Set (fast):"
t = Time.now
result = process_users(users, blocked_ids)
puts "Time: #{Time.now - t:.4f}s"
```

---

*สรุปบทที่ 27: Performance Optimization ควรเริ่มจากการ Measure ก่อนเสมอ ใช้ Benchmark และ Profiler เพื่อหา Bottleneck จริง หลีกเลี่ยง Premature Optimization Lazy Evaluation, Memoization, Object Pooling และ Database Query Optimization เป็น Techniques สำคัญที่ช่วยปรับปรุงประสิทธิภาพได้มาก*
