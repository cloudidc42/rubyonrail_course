# Part 91: Ruby Performance Mastery

## บทนำ

Performance optimization เป็นทักษะสำคัญสำหรับ Ruby developer ระดับสูง บทนี้จะครอบคลุม memory profiling, CPU profiling, object allocation tracking, GC tuning และการเปรียบเทียบ Ruby implementations ต่างๆ

## 1. Ruby Performance Optimization ขั้นสูง

### 1.1 ทำความเข้าใจ Ruby Performance

```ruby
# Ruby มี bottlenecks หลัก 3 ประเภท:
# 1. CPU-bound: คำนวณมาก
# 2. Memory-bound: สร้าง objects มากเกินไป
# 3. I/O-bound: รอ database, network, disk

# เครื่องมือวัด performance
require 'benchmark'

# Basic benchmark
result = Benchmark.measure do
  100_000.times { "hello".upcase }
end

puts result
# =>   0.020000   0.000000   0.020000 (  0.021234)
# user CPU   system CPU   total CPU   real time

# Comparison benchmark
Benchmark.bm(15) do |x|
  x.report("Array#push:") do
    arr = []
    100_000.times { arr.push(1) }
  end
  
  x.report("Array#<<:") do
    arr = []
    100_000.times { arr << 1 }
  end
  
  x.report("Array.new:") do
    100_000.times { Array.new(1000, 0) }
  end
end
```

### 1.2 Benchmark IPS (Iterations Per Second)

```ruby
# Gemfile
gem 'benchmark-ips'

require 'benchmark/ips'

Benchmark.ips do |x|
  x.config(time: 5, warmup: 2)
  
  x.report("String interpolation") { "Hello #{1 + 1}" }
  x.report("String concatenation") { "Hello " + (1 + 1).to_s }
  x.report("String format") { "Hello %d" % (1 + 1) }
  
  x.compare!
end

# ผลลัพธ์ (approximate):
# String interpolation: 14,521,343.5 i/s
# String concatenation:  8,234,521.2 i/s - 1.76x slower
# String format:         6,891,234.1 i/s - 2.11x slower

# More complex benchmarks
array = (1..1000).to_a

Benchmark.ips do |x|
  x.report("Array#find") { array.find { |n| n == 999 } }
  x.report("Array#detect") { array.detect { |n| n == 999 } }
  x.report("Hash#[]") { 
    h = array.each_with_index.to_h
    h[999] 
  }
  
  x.compare!
end
```

### 1.3 String Performance Optimization

```ruby
# การสร้าง frozen strings
# frozen_string_literal: true

# ด้านบนของไฟล์ - strings จะเป็น frozen โดยอัตโนมัติ
str = "hello"  # frozen
str << " world"  # RuntimeError: can't modify frozen String

# ถ้าต้องการ mutable string
str = +"hello"  # Ruby 3.x syntax สำหรับ mutable string
str << " world"  # "hello world"

# String operations performance
require 'benchmark/ips'

Benchmark.ips do |x|
  # การสร้าง String จาก Array
  words = Array.new(1000) { |i| "word#{i}" }
  
  x.report("join") { words.join(" ") }
  x.report("inject +") { words.inject { |memo, w| "#{memo} #{w}" } }
  x.report("each_with_object") do
    words.each_with_object([]) { |w, arr| arr << w }.join(" ")
  end
  
  x.compare!
end

# String encoding
str = "Hello, World!"
puts str.encoding  # UTF-8

# Encoding conversion
utf8_str = "สวัสดี"
ascii_str = utf8_str.encode("ASCII", undef: :replace, replace: "?")

# Binary operations (faster สำหรับ byte manipulation)
binary = "test".b
puts binary.encoding  # ASCII-8BIT
```

## 2. memory_profiler Gem

### 2.1 การติดตั้งและใช้งาน memory_profiler

```ruby
# Gemfile
gem 'memory_profiler'

require 'memory_profiler'

# Basic usage
report = MemoryProfiler.report do
  # code ที่ต้องการ profile
  1000.times do
    user = { name: "John", email: "john@example.com" }
    profile = { user: user, created_at: Time.now }
  end
end

report.pretty_print

# Output ตัวอย่าง:
# Total allocated: 3.21 MB (40020 objects)
# Total retained:  0 B (0 objects)
# 
# allocated memory by gem
# -----------------------------------
#    3216840  other
# 
# allocated objects by class
# -----------------------------------
#      20000  String
#      10000  Hash
#       5000  Time
#       5000  Array
```

### 2.2 Memory Profiling Rails App

```ruby
# lib/tasks/memory_profile.rake
namespace :profile do
  desc "Profile memory usage of key operations"
  task memory: :environment do
    require 'memory_profiler'
    
    # Profile user creation
    puts "=== User Creation Memory Profile ==="
    report = MemoryProfiler.report do
      100.times do
        User.create!(
          name: Faker::Name.name,
          email: Faker::Internet.email,
          password: "password123"
        )
      end
    end
    
    puts "Total allocated: #{report.total_allocated_memsize.to_s(:human_size)}"
    puts "Total retained: #{report.total_retained_memsize.to_s(:human_size)}"
    
    puts "\nTop 5 allocated classes:"
    report.allocated_objects_by_class
          .sort_by { |_, count| -count }
          .first(5)
          .each { |klass, count| puts "  #{klass}: #{count}" }
    
    # Profile query
    puts "\n=== Database Query Memory Profile ==="
    report = MemoryProfiler.report do
      User.includes(:posts, :comments).limit(100).to_a
    end
    
    puts "Total allocated: #{report.total_allocated_memsize.to_s(:human_size)}"
  end
end
```

### 2.3 Detecting Memory Leaks

```ruby
require 'memory_profiler'
require 'objspace'

# ตรวจหา memory leaks
class MemoryLeakDetector
  def initialize(threshold_mb: 5)
    @threshold_mb = threshold_mb
    @snapshots = []
  end
  
  def take_snapshot(label = "")
    GC.start  # Force GC before snapshot
    
    @snapshots << {
      label: label,
      memory: current_memory_mb,
      objects: ObjectSpace.count_objects,
      time: Time.now
    }
  end
  
  def analyze
    return if @snapshots.length < 2
    
    first = @snapshots.first
    last = @snapshots.last
    
    memory_diff = last[:memory] - first[:memory]
    
    puts "Memory Analysis:"
    puts "  Initial: #{first[:memory].round(2)} MB"
    puts "  Final: #{last[:memory].round(2)} MB"
    puts "  Difference: #{memory_diff.round(2)} MB"
    
    if memory_diff > @threshold_mb
      puts "  WARNING: Potential memory leak detected!"
      
      # Object count comparison
      puts "\nObject Count Changes:"
      last[:objects].each do |type, count|
        diff = count - (first[:objects][type] || 0)
        puts "  #{type}: #{diff > 0 ? '+' : ''}#{diff}" if diff.abs > 100
      end
    end
  end
  
  def monitor(&block)
    take_snapshot("before")
    block.call
    take_snapshot("after")
    analyze
  end
  
  private
  
  def current_memory_mb
    # สำหรับ Linux/Mac
    if File.exist?("/proc/#{Process.pid}/status")
      File.read("/proc/#{Process.pid}/status")
          .match(/VmRSS:\s+(\d+) kB/)[1].to_f / 1024
    else
      `ps -o rss= -p #{Process.pid}`.to_f / 1024
    end
  end
end

# ใช้งาน
detector = MemoryLeakDetector.new(threshold_mb: 1)

detector.monitor do
  1000.times do
    # Simulate potential memory leak
    $global_array ||= []
    $global_array << { data: "x" * 1000 }
  end
end
```

## 3. stackprof CPU Profiler

### 3.1 การใช้งาน stackprof

```ruby
# Gemfile
gem 'stackprof'

require 'stackprof'

# CPU profiling
StackProf.run(mode: :cpu, out: '/tmp/cpu.dump') do
  # Code ที่ต้องการ profile
  10_000.times do
    data = Array.new(100) { rand(1000) }
    data.sort
    data.select { |n| n > 500 }
    data.map { |n| n * 2 }
  end
end

# อ่านผล
result = StackProf::Report.new(Marshal.load(File.read('/tmp/cpu.dump')))
result.print_text  # แสดงผลใน terminal
result.print_graphviz  # สร้าง DOT graph
result.print_flamegraph  # สร้าง flamegraph data

# Sampling modes:
# :cpu     - สุ่มทุก 1ms ของ CPU time
# :wall    - สุ่มทุก 1ms ของ wall clock time  
# :object  - สุ่มทุกครั้งที่มี object allocation
```

### 3.2 stackprof ใน Rails

```ruby
# config/initializers/stackprof.rb (สำหรับ development)
if Rails.env.development?
  require 'stackprof'
  
  # Middleware สำหรับ profile ทุก request
  class StackProfMiddleware
    def initialize(app)
      @app = app
    end
    
    def call(env)
      return @app.call(env) unless profile?(env)
      
      result = nil
      StackProf.run(mode: :wall, raw: true) do
        result = @app.call(env)
      end
      
      # Save profile
      filename = "tmp/stackprof/#{Time.now.to_i}_#{env['PATH_INFO'].gsub('/', '-')}.dump"
      FileUtils.mkdir_p(File.dirname(filename))
      StackProf::Report.new(StackProf.results).save(filename)
      
      result
    end
    
    private
    
    def profile?(env)
      env["HTTP_X_PROFILE"] == "true" || env["QUERY_STRING"].include?("_profile=1")
    end
  end
  
  # config/application.rb
  # config.middleware.use StackProfMiddleware
end

# Rack middleware สำหรับ production profiling (ระวัง overhead)
# GET /admin/profiling/start -> เริ่ม profiling
# GET /admin/profiling/stop  -> หยุดและ download dump
```

### 3.3 Flame Graph Analysis

```ruby
# สร้าง flame graph สำหรับ analysis
require 'stackprof'

# Collect data
StackProf.run(mode: :cpu, raw: true, out: 'tmp/profile.dump') do
  # Heavy computation
  users = User.includes(:orders, :products).limit(1000).to_a
  users.each do |user|
    total = user.orders.sum(&:total)
    avg_order = total / [user.orders.count, 1].max
    user.calculate_lifetime_value
  end
end

# Convert to flamegraph
# stackprof --flamegraph tmp/profile.dump > /tmp/flamegraph.json
# แล้วเปิดใน https://speedscope.app/

# ผล: เห็น call stack hierarchy และ เวลาที่ใช้ในแต่ละ method

# วิเคราะห์ด้วย Ruby
result = StackProf::Report.new(Marshal.load(File.binread('tmp/profile.dump')))

puts "Top 10 most called methods:"
result.frames
      .sort_by { |_, frame| -frame[:total_samples] }
      .first(10)
      .each do |name, frame|
  puts "  #{name}: #{frame[:total_samples]} samples (#{frame[:total_percent].round(1)}%)"
end
```

## 4. Object Allocation Tracking

### 4.1 ObjectSpace

```ruby
require 'objspace'

# นับ objects ทั้งหมด
puts ObjectSpace.count_objects.inspect
# => {:TOTAL=>142847, :FREE=>7621, :T_OBJECT=>12543, :T_CLASS=>1234, ...}

# หา objects ทุกตัวของ class ที่ระบุ
string_count = 0
ObjectSpace.each_object(String) { string_count += 1 }
puts "Total strings: #{string_count}"

# Object allocation tracing
ObjectSpace.trace_object_allocations_start

# Code ที่ต้องการ track
def create_user(name)
  user = { name: name, created_at: Time.now }
  user
end

result = create_user("John")
puts ObjectSpace.allocation_sourcefile(result)  # ไฟล์ที่สร้าง object
puts ObjectSpace.allocation_sourceline(result)  # บรรทัดที่สร้าง object
puts ObjectSpace.allocation_class_path(result)  # class ที่สร้าง object

ObjectSpace.trace_object_allocations_stop
ObjectSpace.trace_object_allocations_clear
```

### 4.2 allocation_stats Gem

```ruby
# Gemfile
gem 'allocation_stats'

require 'allocation_stats'

# Track allocations
stats = AllocationStats.trace do
  array = []
  1000.times { array << "item" }
  array.map(&:upcase)
end

puts stats.allocations(alias_paths: true)
          .group_by(:sourcefile, :sourceline, :class)
          .sort_by_count
          .to_text
```

### 4.3 ObjectSpace สำหรับ Memory Optimization

```ruby
# ตรวจหา retained objects (objects ที่ไม่ถูก GC)
require 'objspace'

class LeakFinder
  def initialize
    @before_objects = {}
    @after_objects = {}
  end
  
  def capture_before
    GC.start(full_mark: true, immediate_sweep: true)
    ObjectSpace.each_object do |obj|
      @before_objects[obj.__id__] = obj.class.name
    end
  end
  
  def capture_after
    GC.start(full_mark: true, immediate_sweep: true)
    ObjectSpace.each_object do |obj|
      @after_objects[obj.__id__] = obj.class.name
    end
  end
  
  def report
    new_objects = @after_objects.reject { |id, _| @before_objects.key?(id) }
    
    by_class = new_objects.values.tally.sort_by { |_, count| -count }
    
    puts "New objects after operation:"
    by_class.first(10).each do |klass, count|
      puts "  #{klass}: #{count}"
    end
    
    puts "\nTotal new objects: #{new_objects.length}"
  end
end

# ใช้งาน
finder = LeakFinder.new
finder.capture_before

# Code ที่ต้องการตรวจ
1000.times { User.new(name: "test") }

finder.capture_after
finder.report
```

### 4.4 Reducing Object Allocation

```ruby
# Bad: สร้าง objects มากเกินไป
def process_data_bad(data)
  data.select { |item| item[:active] }
      .map { |item| item[:name].upcase }
      .reject { |name| name.empty? }
      .join(", ")
end

# Good: ใช้ lazy evaluation
def process_data_good(data)
  data.lazy
      .select { |item| item[:active] }
      .map { |item| item[:name].upcase }
      .reject { |name| name.empty? }
      .force
      .join(", ")
end

# Better: ใช้ each_with_object
def process_data_better(data)
  data.each_with_object([]) do |item, result|
    next unless item[:active]
    name = item[:name].upcase
    result << name unless name.empty?
  end.join(", ")
end

# Benchmark
require 'benchmark/ips'
data = 10_000.times.map { |i| { name: "item#{i}", active: i.even? } }

Benchmark.ips do |x|
  x.report("bad (chained)") { process_data_bad(data) }
  x.report("good (lazy)") { process_data_good(data) }
  x.report("better (each_with_object)") { process_data_better(data) }
  x.compare!
end

# Struct vs Hash vs OpenStruct performance
require 'benchmark/ips'
require 'ostruct'

UserStruct = Struct.new(:name, :email, :age)

Benchmark.ips do |x|
  x.report("Hash") { { name: "John", email: "john@example.com", age: 30 } }
  x.report("Struct") { UserStruct.new("John", "john@example.com", 30) }
  x.report("OpenStruct") { OpenStruct.new(name: "John", email: "john@example.com", age: 30) }
  x.compare!
end

# Struct เร็วกว่า Hash เล็กน้อยสำหรับ access
# OpenStruct ช้าที่สุด - ไม่ควรใช้ใน hot paths
```

## 5. GC Tuning

### 5.1 Ruby Garbage Collector

```ruby
# Ruby ใช้ Tri-color mark-and-sweep GC
# Ruby 3.x มี incremental GC และ compaction GC

# ดู GC stats
GC.stat.each { |key, val| puts "#{key}: #{val}" }

# GC stats ที่สำคัญ:
# :count          - จำนวนครั้งที่ GC ทำงาน
# :heap_allocated_pages - จำนวน heap pages
# :heap_live_slots   - จำนวน live objects
# :heap_free_slots   - จำนวน free slots
# :heap_sorted_length - ขนาด heap
# :major_gc_count   - จำนวน major GC
# :minor_gc_count   - จำนวน minor GC
# :total_allocated_objects - จำนวน objects ที่สร้างทั้งหมด

# ดู GC.stat ก่อนและหลัง code
before = GC.stat.dup
run_my_code
after = GC.stat

puts "GC runs: #{after[:count] - before[:count]}"
puts "Objects allocated: #{after[:total_allocated_objects] - before[:total_allocated_objects]}"
```

### 5.2 GC.compact (Ruby 3.x)

```ruby
# GC.compact รวบรวม live objects ให้อยู่ใกล้กัน
# ลด memory fragmentation

# Before compaction
puts "Before: #{GC.stat[:heap_sorted_length]}"

GC.compact

puts "After: #{GC.stat[:heap_sorted_length]}"

# ใน Rails: เรียก GC.compact หลัง boot
# config/initializers/gc_compact.rb
if defined?(GC.compact) && Rails.env.production?
  Rails.application.config.after_initialize do
    GC.compact
    Rails.logger.info "GC compact performed after initialization"
  end
end

# Copy-on-write friendly
# Ruby 3.x รองรับ copy_on_write_friendly compaction
GC.compact
# ใช้ตอน fork processes (Unicorn, Puma)
```

### 5.3 GC Environment Variables

```bash
# สำหรับ production, tune GC ผ่าน environment variables

# RUBY_GC_HEAP_INIT_SLOTS - จำนวน slots เริ่มต้น (default: 10000)
# เพิ่มถ้า app สร้าง objects เยอะตั้งแต่เริ่ม
export RUBY_GC_HEAP_INIT_SLOTS=600000

# RUBY_GC_HEAP_GROWTH_FACTOR - อัตราการ grow heap (default: 1.8)
# ลดถ้าต้องการ memory footprint น้อยลง
export RUBY_GC_HEAP_GROWTH_FACTOR=1.25

# RUBY_GC_HEAP_FREE_SLOTS - จำนวน free slots หลัง GC (default: 4096)
export RUBY_GC_HEAP_FREE_SLOTS=600000

# RUBY_GC_MALLOC_LIMIT - malloc threshold (default: 16MB)
export RUBY_GC_MALLOC_LIMIT=90000000

# RUBY_GC_OLDMALLOC_LIMIT - old malloc threshold (default: 16MB)
export RUBY_GC_OLDMALLOC_LIMIT=90000000

# Production config (e.g., Heroku Puma-based apps)
# สำหรับ Puma workers:
export MALLOC_ARENA_MAX=2
export RUBY_GC_HEAP_GROWTH_FACTOR=1.1
export RUBY_GC_HEAP_FREE_SLOTS_MIN_RATIO=0.10
export RUBY_GC_HEAP_FREE_SLOTS_GOAL_RATIO=0.20
```

### 5.4 Generational GC

```ruby
# Ruby 2.1+ ใช้ generational GC (minor + major GC)
# Objects ถูกแบ่งเป็น:
# - Young generation: objects ที่เพิ่งสร้าง
# - Old generation: objects ที่รอด minor GC

# Minor GC: เร็ว, เก็บแค่ young generation
# Major GC: ช้า, เก็บทุก generation

# Monitor GC activity
class GCMonitor
  def self.track
    before_stat = GC.stat.dup
    start_time = Time.now
    
    result = yield
    
    duration = Time.now - start_time
    after_stat = GC.stat
    
    {
      result: result,
      duration: duration,
      minor_gc_count: after_stat[:minor_gc_count] - before_stat[:minor_gc_count],
      major_gc_count: after_stat[:major_gc_count] - before_stat[:major_gc_count],
      objects_allocated: after_stat[:total_allocated_objects] - before_stat[:total_allocated_objects],
      objects_freed: after_stat[:total_freed_objects] - before_stat[:total_freed_objects]
    }
  end
end

result = GCMonitor.track do
  1_000_000.times { "hello".upcase }
end

puts "Duration: #{result[:duration].round(3)}s"
puts "Minor GC: #{result[:minor_gc_count]}"
puts "Major GC: #{result[:major_gc_count]}"
puts "Objects allocated: #{result[:objects_allocated]}"
puts "Objects freed: #{result[:objects_freed]}"
```

### 5.5 GC-friendly Code Patterns

```ruby
# Pattern 1: Object reuse
# Bad
def process_items(items)
  items.map { |item| { id: item.id, name: item.name } }  # สร้าง Hash ใหม่ทุก item
end

# Good: reuse hash (ถ้า thread-safe)
def process_items_good(items)
  buffer = {}  # สร้างครั้งเดียว
  items.map do |item|
    buffer[:id] = item.id
    buffer[:name] = item.name
    buffer.dup  # ต้อง dup ถ้าต้องการ independent copies
  end
end

# Pattern 2: String freezing
MODULE_CONSTANT = "hello".freeze  # frozen string

def greet(name)
  MODULE_CONSTANT + " " + name  # OK แต่ยัง allocate strings
end

# Pattern 3: Symbol vs String
# Symbols เหมาะสำหรับ hash keys (ไม่ถูก GC เพราะเป็น global)
options = { timeout: 30, retry: true }  # ดีกว่า { "timeout" => 30 }

# Pattern 4: Lazy evaluation
# Bad: evaluate ทั้งหมดแม้ต้องการแค่บางส่วน
def expensive_options
  {
    data1: compute_expensive_thing_1,
    data2: compute_expensive_thing_2,
    data3: compute_expensive_thing_3
  }
end

# Good: lazy evaluation
class LazyOptions
  def initialize
    @cache = {}
  end
  
  def [](key)
    @cache[key] ||= send(:"compute_#{key}")
  end
  
  private
  
  def compute_data1
    # expensive computation
  end
end
```

## 6. JRuby vs TruffleRuby Performance

### 6.1 JRuby Overview

```ruby
# JRuby รัน Ruby code บน JVM
# ข้อดี:
# - True multithreading (JVM threads, no GIL)
# - Java interop
# - JIT compilation (เร็วกว่า CRuby สำหรับ long-running processes)
# - Large heap sizes

# การติดตั้ง JRuby
# rbenv install jruby-9.4.6.0
# rbenv global jruby-9.4.6.0

# ตรวจสอบ
ruby --version  # jruby 9.4.6.0 (3.1.4) ...

# JRuby + Concurrency (ไม่มี GIL!)
threads = 10.times.map do
  Thread.new do
    result = 0
    1_000_000.times { result += 1 }
    result
  end
end

total = threads.sum(&:value)
puts total  # 10,000,000 (correct, no race conditions with simple += on JVM)

# Java Integration
require 'java'

java_import 'java.util.ArrayList'
java_import 'java.util.HashMap'

list = ArrayList.new
list.add("Hello")
list.add("World")
puts list.size  # 2

# ใช้ Java libraries
java_import 'org.apache.commons.lang3.StringUtils'
puts StringUtils.capitalize("hello world")
```

### 6.2 TruffleRuby Overview

```ruby
# TruffleRuby รัน Ruby บน GraalVM
# ข้อดี:
# - Polyglot programming (Ruby + Python + JS + Java ในโปรแกรมเดียว)
# - AOT compilation (native image)
# - Excellent peak performance
# - Low memory footprint (native mode)

# การติดตั้ง
# rbenv install truffleruby+graalvm-23.1.1
# rbenv global truffleruby+graalvm-23.1.1

# ตรวจสอบ
ruby --version  # truffleruby 23.1.1, like ruby 3.2.2, ...

# Polyglot - รัน JavaScript จาก Ruby
require 'polyglot'
require 'truffle/interop'

js_code = Polyglot.eval("js", "1 + 1")
puts js_code  # 2

# Python จาก Ruby
python_result = Polyglot.eval("python", "list(range(5))")
puts python_result.inspect

# Native Image (AOT compilation)
# truffleruby --native script.rb
# หรือ native-image -H:Class=org.truffleruby.Main
```

### 6.3 Performance Benchmarks

```ruby
# เปรียบเทียบ CRuby vs JRuby vs TruffleRuby
# (ตัวเลขโดยประมาณ)

# Test 1: Fibonacci (CPU intensive)
def fib(n)
  return n if n <= 1
  fib(n - 1) + fib(n - 2)
end

# CRuby 3.3:    fib(35) ~ 1.2s
# JRuby 9.4:    fib(35) ~ 0.8s (after warmup)
# TruffleRuby:  fib(35) ~ 0.1s (fully compiled)

# Test 2: Object allocation (memory heavy)
# CRuby:       Best GC
# JRuby:       Large heap, JVM GC (generational)
# TruffleRuby: Efficient with AOT

# Test 3: Multi-threading
# CRuby:       GIL - threads ไม่ run parallel สำหรับ Ruby code
# JRuby:       True parallel threads
# TruffleRuby: True parallel threads

# Test 4: Startup time
# CRuby:       ~50ms
# JRuby:       ~1-3s (JVM startup)
# TruffleRuby: ~1s (native mode: ~50ms)

# Recommendation:
# CRuby:       Web apps, scripts, gems compatibility
# JRuby:       High-throughput services, Java integration
# TruffleRuby: Peak performance, polyglot, containers
```

### 6.4 Ractors สำหรับ True Parallelism (CRuby)

```ruby
# Ruby 3.x มี Ractors - parallel execution units
# ช่วยหลีกเลี่ยง GIL สำหรับ CPU-intensive tasks

# Simple Ractor example
r1 = Ractor.new do
  Ractor.yield "Hello from Ractor 1"
end

r2 = Ractor.new do
  Ractor.yield "Hello from Ractor 2"
end

puts r1.take
puts r2.take

# Parallel computation
def parallel_sum(numbers, n_ractors: 4)
  chunk_size = (numbers.length / n_ractors.to_f).ceil
  chunks = numbers.each_slice(chunk_size).to_a
  
  ractors = chunks.map do |chunk|
    Ractor.new(chunk) do |data|
      data.sum
    end
  end
  
  ractors.sum(&:take)
end

numbers = (1..1_000_000).to_a
puts parallel_sum(numbers, n_ractors: 4)  # 500000500000

# Ractor limitations:
# - Objects ต้อง shareable หรือ frozen
# - ไม่ทุก gem compatible กับ Ractor
# - Still experimental ใน Ruby 3.x

# Shareable objects
frozen_hash = { key: "value" }.freeze
frozen_hash_shareable = Ractor.make_shareable(frozen_hash)

r = Ractor.new(frozen_hash_shareable) do |data|
  puts data[:key]
end
r.take
```

## 7. ActiveRecord Performance

### 7.1 N+1 Query Detection

```ruby
# Gemfile
gem 'bullet'

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
  Bullet.unused_eager_loading_enable = true
  Bullet.counter_cache_enable = true
end

# N+1 ตัวอย่าง
# Bad
posts = Post.all
posts.each { |post| puts post.user.name }  # N+1!

# Good: eager loading
posts = Post.includes(:user).all
posts.each { |post| puts post.user.name }  # 2 queries total

# More complex N+1
# Bad
users = User.all
users.each do |user|
  puts user.posts.count  # N+1!
  puts user.comments.recent.limit(5)  # Another N+1!
end

# Good
users = User.includes(:posts, comments: :post).all
users.each do |user|
  puts user.posts.size  # uses loaded association
  puts user.comments.select { |c| c.created_at > 1.week.ago }.first(5)
end
```

### 7.2 Query Optimization

```ruby
# Select only needed columns
User.select(:id, :name, :email).all  # แทน User.all

# Use pluck for simple values
emails = User.pluck(:email)  # เร็วกว่า User.all.map(&:email) มาก

# Batch processing
User.find_each(batch_size: 1000) do |user|
  user.send_newsletter
end

# Use joins instead of includes when not accessing associations
User.joins(:orders)
    .where(orders: { status: 'completed' })
    .distinct

# Counter caches
# add_column :users, :posts_count, :integer, default: 0
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end

# ใช้ posts_count แทน posts.count
user.posts_count  # ไม่ query database

# Database indexes
# db/migrate/20240101000000_add_indexes.rb
class AddIndexes < ActiveRecord::Migration[7.1]
  def change
    add_index :users, :email, unique: true
    add_index :orders, [:user_id, :status]
    add_index :posts, [:user_id, :published_at]
    add_index :comments, :post_id
    
    # Partial indexes
    add_index :orders, :user_id, where: "status = 'pending'"
    
    # Composite index for common queries
    add_index :sessions, [:user_id, :created_at]
  end
end

# Raw SQL สำหรับ complex queries
result = ActiveRecord::Base.connection.execute(<<~SQL)
  SELECT u.id, u.name, COUNT(o.id) as order_count, SUM(o.total) as revenue
  FROM users u
  LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
  WHERE u.created_at > NOW() - INTERVAL '30 days'
  GROUP BY u.id, u.name
  ORDER BY revenue DESC
  LIMIT 10
SQL
```

### 7.3 Database Connection Pooling

```ruby
# config/database.yml
production:
  adapter: postgresql
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  checkout_timeout: 5  # seconds to wait for connection
  reaping_frequency: 10  # check for dead connections every 10s
  idle_timeout: 300  # remove idle connections after 5 min

# PgBouncer (connection pooler)
# ใช้ PgBouncer ระหว่าง Rails app และ PostgreSQL
# รองรับ transaction-mode pooling (ประหยัดหน่วยความจำ)

# config/database.yml (เมื่อใช้ PgBouncer)
production:
  url: postgresql://pgbouncer_host:6432/myapp
  # PgBouncer mode: transaction
  # ไม่ควรใช้ prepared statements ใน transaction mode
  prepared_statements: false
  advisory_locks: false
```

## 8. Caching Strategies

### 8.1 Russian Doll Caching

```ruby
# app/views/posts/show.html.erb
<% cache @post do %>
  <article>
    <h1><%= @post.title %></h1>
    
    <% cache [@post, "comments"] do %>
      <%= render @post.comments %>
    <% end %>
    
    <% cache [@post, "related"] do %>
      <%= render @post.related_posts %>
    <% end %>
  </article>
<% end %>

# Cache keys
# Post cache: "views/posts/1-20240101120000123456789"
# Comments cache: "views/posts/1-20240101120000123456789/comments"

# เมื่อ post update -> post cache invalid -> comments/related cache invalid ด้วย

# Caching ใน controller
class PostsController < ApplicationController
  def index
    @posts = Rails.cache.fetch("posts/all", expires_in: 5.minutes) do
      Post.includes(:user, :tags).published.recent.limit(20).to_a
    end
  end
end

# Fragment caching
class Post < ApplicationRecord
  def cache_key_with_version
    "#{cache_key}/v#{updated_at.to_i}"
  end
end
```

### 8.2 Low-level Caching

```ruby
# Redis caching
Rails.cache.write("user:#{id}:stats", stats, expires_in: 1.hour)
user_stats = Rails.cache.read("user:#{id}:stats")

# Fetch with block
user = Rails.cache.fetch("user:#{id}", expires_in: 10.minutes) do
  User.find(id)
end

# Multi-fetch
users = Rails.cache.fetch_multi(
  "user:1", "user:2", "user:3"
) do |key|
  id = key.split(":").last.to_i
  User.find(id)
end

# Counter caching
Rails.cache.increment("page_views:#{path}", 1, expires_in: 1.day)
views = Rails.cache.read("page_views:#{path}") || 0

# Cache versioning
class User < ApplicationRecord
  def cache_version
    "#{updated_at.to_i}-#{posts.maximum(:updated_at).to_i}"
  end
end
```

## 9. Profiling Real Applications

### 9.1 Rack Mini Profiler

```ruby
# Gemfile
gem 'rack-mini-profiler'
gem 'flamegraph'
gem 'stackprof'
gem 'memory_profiler'

# config/initializers/rack_mini_profiler.rb
if Rails.env.development? || Rails.env.staging?
  require 'rack-mini-profiler'
  
  Rack::MiniProfiler.config.position = 'bottom-right'
  Rack::MiniProfiler.config.skip_paths = ['/assets', '/cable']
  
  # Enable flamegraph: append ?pp=flamegraph to URL
  # Enable memory profiling: append ?pp=memory to URL
  # Enable trace: append ?pp=trace to URL
end

# Usage:
# GET /products?pp=flamegraph    -> CPU flamegraph
# GET /products?pp=memory        -> Memory profiling
# GET /products?pp=trace         -> method tracing
# GET /products?pp=analyze-memory -> ObjectSpace analysis
```

### 9.2 Application Performance Monitoring

```ruby
# Using scout_apm หรือ datadog
gem 'scout_apm'

# config/scout_apm.yml
common: &defaults
  name: <%= ENV['APP_NAME'] || 'MyApp' %>
  monitor: true

development:
  <<: *defaults
  monitor: false

production:
  <<: *defaults
  key: <%= ENV['SCOUT_KEY'] %>

# Custom instrumentation
ScoutApm::Instrument.use do
  def process_payment(order)
    # Scout จะ track method นี้
    super
  end
end

# Manual tracing
ScoutApm::Tracer.instrument("Payment", "process") do |span|
  span.tag("order_id", order.id)
  process_stripe_payment(order)
end
```

## 10. Performance Testing

### 10.1 Load Testing ด้วย k6

```javascript
// load_test.js
import http from 'k6/http'
import { check, sleep } from 'k6'

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp up
    { duration: '1m', target: 20 },    // Steady state
    { duration: '10s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% requests < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
}

export default function() {
  const res = http.get('http://localhost:3000/api/v1/products')
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  })
  
  sleep(1)
}
```

### 10.2 Performance Tests ใน RSpec

```ruby
# spec/performance/api_performance_spec.rb
require 'rails_helper'
require 'benchmark/ips'

RSpec.describe "API Performance", type: :request do
  before { create_list(:product, 100) }
  
  it "lists products within acceptable time" do
    time = Benchmark.measure { get '/api/v1/products' }.real
    expect(time).to be < 0.1  # 100ms
  end
  
  it "handles concurrent requests efficiently" do
    threads = 10.times.map do
      Thread.new { get '/api/v1/products' }
    end
    
    threads.each(&:join)
    
    expect(response).to have_http_status(:ok)
  end
  
  it "doesn't have N+1 queries for product list" do
    query_count = 0
    
    counter = -> (*, **) { query_count += 1 }
    ActiveSupport::Notifications.subscribed(counter, "sql.active_record") do
      get '/api/v1/products'
    end
    
    # Should be 2 queries max: products + associations
    expect(query_count).to be <= 2
  end
end
```

---

## แบบฝึกหัดบทที่ 91

### แบบฝึกหัดที่ 1: Benchmark String Operations
**โจทย์:** เปรียบเทียบ performance ของการต่อ strings วิธีต่างๆ และหาวิธีที่เร็วที่สุด

**เฉลย:**
```ruby
require 'benchmark/ips'

WORDS = %w[hello world foo bar baz qux]

Benchmark.ips do |x|
  x.config(time: 3, warmup: 1)
  
  x.report("String#+") do
    WORDS.inject("") { |result, w| result + w + " " }
  end
  
  x.report("String#concat") do
    result = ""
    WORDS.each { |w| result.concat(w).concat(" ") }
    result
  end
  
  x.report("Array#join") do
    WORDS.join(" ")
  end
  
  x.report("StringIO") do
    require 'stringio'
    sio = StringIO.new
    WORDS.each { |w| sio << w << " " }
    sio.string
  end
  
  x.report("String interpolation") do
    "#{WORDS[0]} #{WORDS[1]} #{WORDS[2]} #{WORDS[3]} #{WORDS[4]} #{WORDS[5]}"
  end
  
  x.compare!
end
# Array#join และ String#concat ที่ใช้ << เร็วที่สุด
```

### แบบฝึกหัดที่ 2: Memory Profiling
**โจทย์:** Profile memory usage ของ method ที่สร้าง objects มากเกินไป และ optimize

**เฉลย:**
```ruby
require 'memory_profiler'
require 'benchmark/ips'

# Version 1: Memory-heavy
def parse_logs_v1(lines)
  lines.map do |line|
    parts = line.split(" ")
    {
      timestamp: Time.parse(parts[0] + " " + parts[1]),
      level: parts[2].upcase,
      message: parts[3..].join(" "),
      tags: parts[3..].grep(/\[.*\]/).map { |t| t[1..-2] }
    }
  end
end

# Version 2: Memory-optimized
LogEntry = Struct.new(:timestamp, :level, :message, :tags)

def parse_logs_v2(lines)
  lines.map do |line|
    timestamp_end = line.index(" ", 11)
    level_end = line.index(" ", timestamp_end + 1)
    
    timestamp_str = line[0...timestamp_end]
    level = line[timestamp_end + 1...level_end]
    message = line[level_end + 1..]
    
    tags = message.scan(/\[([^\]]+)\]/).flatten
    
    LogEntry.new(
      Time.parse(timestamp_str),
      level.upcase!,
      message,
      tags
    )
  end
end

# Generate test data
lines = 10_000.times.map do |i|
  "2024-01-01 12:00:#{i % 60} INFO Processing request [web] [api] message #{i}"
end

# Profile v1
report_v1 = MemoryProfiler.report { parse_logs_v1(lines) }
# Profile v2
report_v2 = MemoryProfiler.report { parse_logs_v2(lines) }

puts "V1 Memory: #{report_v1.total_allocated_memsize / 1024 / 1024}MB"
puts "V2 Memory: #{report_v2.total_allocated_memsize / 1024 / 1024}MB"
puts "Savings: #{((1 - report_v2.total_allocated_memsize.to_f / report_v1.total_allocated_memsize) * 100).round(1)}%"

Benchmark.ips do |x|
  x.report("v1") { parse_logs_v1(lines) }
  x.report("v2") { parse_logs_v2(lines) }
  x.compare!
end
```

### แบบฝึกหัดที่ 3: GC Tuning
**โจทย์:** ทำ GC tuning สำหรับ Rails app ที่มีปัญหา memory

**เฉลย:**
```ruby
# config/initializers/gc_tuning.rb

# ตรวจสอบ current GC settings
current_settings = {
  heap_init_slots: ENV.fetch('RUBY_GC_HEAP_INIT_SLOTS', 10000).to_i,
  heap_growth_factor: ENV.fetch('RUBY_GC_HEAP_GROWTH_FACTOR', 1.8).to_f,
  heap_free_slots: ENV.fetch('RUBY_GC_HEAP_FREE_SLOTS', 4096).to_i,
  malloc_limit: ENV.fetch('RUBY_GC_MALLOC_LIMIT', 16_000_000).to_i
}

Rails.logger.info "GC Settings: #{current_settings}"

# Monitor GC pauses
class GCProfiler
  def self.setup
    GC::Profiler.enable
    
    at_exit do
      Rails.logger.info "GC Profile:"
      Rails.logger.info GC::Profiler.result
      GC::Profiler.disable
    end
  end
  
  def self.report
    data = GC::Profiler.raw_data
    
    {
      gc_count: data.length,
      total_gc_time: data.sum { |d| d[:GC_TIME] },
      avg_gc_time: data.empty? ? 0 : data.sum { |d| d[:GC_TIME] } / data.length,
      heap_use: data.last&.dig(:HEAP_USE_SLOT_NAME) || 0
    }
  end
end

if Rails.env.production? && ENV['ENABLE_GC_PROFILING']
  GCProfiler.setup
end

# Recommended settings per app type:
RECOMMENDED_SETTINGS = {
  # Small app (< 10 req/s)
  small: {
    RUBY_GC_HEAP_INIT_SLOTS: 200_000,
    RUBY_GC_HEAP_GROWTH_FACTOR: 1.5,
    RUBY_GC_HEAP_FREE_SLOTS: 200_000,
    RUBY_GC_MALLOC_LIMIT: 32_000_000
  },
  # Medium app (10-100 req/s)
  medium: {
    RUBY_GC_HEAP_INIT_SLOTS: 600_000,
    RUBY_GC_HEAP_GROWTH_FACTOR: 1.25,
    RUBY_GC_HEAP_FREE_SLOTS: 600_000,
    RUBY_GC_MALLOC_LIMIT: 64_000_000
  },
  # Large app (> 100 req/s)
  large: {
    RUBY_GC_HEAP_INIT_SLOTS: 2_000_000,
    RUBY_GC_HEAP_GROWTH_FACTOR: 1.1,
    RUBY_GC_HEAP_FREE_SLOTS: 1_000_000,
    RUBY_GC_MALLOC_LIMIT: 128_000_000
  }
}
```

### แบบฝึกหัดที่ 4: stackprof Analysis
**โจทย์:** ใช้ stackprof เพื่อหา bottleneck ใน slow endpoint

**เฉลย:**
```ruby
# spec/performance/slow_endpoint_spec.rb
require 'rails_helper'
require 'stackprof'

RSpec.describe "Dashboard endpoint performance" do
  before { create_test_data }
  
  it "profiles dashboard endpoint" do
    profile = StackProf.run(mode: :wall, raw: true) do
      100.times { get '/dashboard' }
    end
    
    report = StackProf::Report.new(profile)
    
    # Print top methods
    puts "\n=== Top 20 methods by total time ==="
    report.print_text(20)
    
    # Find methods over 10% of total time
    expensive = report.frames
      .select { |_, f| f[:total_percent] > 10 }
      .sort_by { |_, f| -f[:total_percent] }
    
    puts "\n=== Methods over 10% ==="
    expensive.each do |name, frame|
      puts "  #{name}: #{frame[:total_percent].round(1)}%"
    end
    
    # Assert no single method takes more than 50%
    max_percent = report.frames.values.map { |f| f[:total_percent] }.max
    expect(max_percent).to be < 50
  end
  
  private
  
  def create_test_data
    users = create_list(:user, 10)
    users.each do |user|
      create_list(:order, 5, user: user)
      create_list(:post, 3, user: user)
    end
  end
end
```

### แบบฝึกหัดที่ 5: Ractor Parallel Processing
**โจทย์:** ใช้ Ractors สำหรับ parallel image processing

**เฉลย:**
```ruby
require 'ractor'

# Parallel image resizing (conceptual - ต้องใช้ thread-safe gem)
class ParallelImageProcessor
  def initialize(n_workers: 4)
    @n_workers = n_workers
  end
  
  def process(image_paths)
    # Distribute work across Ractors
    chunk_size = (image_paths.length / @n_workers.to_f).ceil
    chunks = image_paths.each_slice(chunk_size).to_a
    
    ractors = chunks.map do |chunk|
      Ractor.new(chunk) do |paths|
        paths.map do |path|
          # Process each image
          result = process_image(path)
          [path, result]
        end
      end
    end
    
    # Collect results
    results = {}
    ractors.each do |r|
      r.take.each { |path, result| results[path] = result }
    end
    
    results
  end
  
  private
  
  def process_image(path)
    # Simulate image processing
    { 
      original: path,
      thumbnail: path.gsub('.jpg', '_thumb.jpg'),
      size: File.size(path)
    }
  rescue
    { error: "Failed to process #{path}" }
  end
end

# Word count สำหรับ Ractor (ตัวอย่างที่ใช้งานได้จริง)
def parallel_word_count(texts, n_ractors: 4)
  chunk_size = (texts.length / n_ractors.to_f).ceil
  chunks = texts.each_slice(chunk_size).to_a
  
  ractors = chunks.map do |chunk|
    Ractor.new(chunk) do |text_chunk|
      text_chunk.each_with_object(Hash.new(0)) do |text, counts|
        text.split.each { |word| counts[word.downcase] += 1 }
      end
    end
  end
  
  # Merge results
  ractors.each_with_object(Hash.new(0)) do |r, total|
    r.take.each { |word, count| total[word] += count }
  end
end

texts = [
  "hello world hello ruby",
  "world is great ruby is great",
  "hello everyone ruby world"
]

result = parallel_word_count(texts)
puts result.sort_by { |_, count| -count }.first(5).inspect
```

### แบบฝึกหัดที่ 6: Database Query Optimization
**โจทย์:** Optimize slow queries ใน Rails app

**เฉลย:**
```ruby
# lib/tasks/query_optimization.rake
namespace :optimize do
  desc "Find and optimize slow queries"
  task queries: :environment do
    # Enable query logging
    slow_queries = []
    
    ActiveSupport::Notifications.subscribe("sql.active_record") do |*args|
      event = ActiveSupport::Notifications::Event.new(*args)
      
      if event.duration > 100  # queries over 100ms
        slow_queries << {
          sql: event.payload[:sql],
          duration: event.duration,
          binds: event.payload[:binds]
        }
      end
    end
    
    # Run application scenarios
    puts "Testing common scenarios..."
    
    # Scenario 1: User dashboard
    User.includes(:posts, :comments, :orders).limit(10).each do |user|
      user.posts.published.recent
      user.orders.completed.sum(:total)
    end
    
    # Report
    puts "\n=== Slow Queries Found ==="
    slow_queries.sort_by { |q| -q[:duration] }.each do |query|
      puts "Duration: #{query[:duration].round(2)}ms"
      puts "SQL: #{query[:sql][0..200]}"
      puts "---"
    end
    
    # Suggest indexes
    puts "\n=== Suggested Indexes ==="
    slow_queries.each do |query|
      suggest_indexes(query[:sql])
    end
  end
  
  private
  
  def suggest_indexes(sql)
    # Simple heuristic: look for WHERE clauses without indexes
    where_columns = sql.scan(/WHERE\s+(\w+)\.(\w+)\s*=/).map { |_, col| col }
    
    where_columns.each do |col|
      puts "Consider adding index on: #{col}"
    end
  end
end

# app/models/concerns/query_logging.rb
module QueryLogging
  extend ActiveSupport::Concern
  
  included do
    around_action :log_query_count, if: -> { Rails.env.development? }
  end
  
  private
  
  def log_query_count
    query_count = 0
    
    counter = -> (*, **) { query_count += 1 }
    ActiveSupport::Notifications.subscribed(counter, "sql.active_record") do
      yield
    end
    
    if query_count > 10
      Rails.logger.warn "#{controller_name}##{action_name}: #{query_count} queries - potential N+1!"
    end
  end
end
```

### แบบฝึกหัดที่ 7: Caching Strategy
**โจทย์:** Implement multi-level caching strategy

**เฉลย:**
```ruby
# app/services/multi_level_cache_service.rb
class MultiLevelCacheService
  # Level 1: In-memory cache (fastest, not shared between processes)
  MEMORY_CACHE = {}
  MEMORY_CACHE_EXPIRY = {}
  
  # Level 2: Redis cache (shared, persistent)
  # Level 3: Database (always available)
  
  def self.fetch(key, expires_in: 5.minutes, &block)
    # Try L1 (memory)
    if (cached = fetch_from_memory(key))
      Rails.logger.debug "Cache HIT (L1-memory): #{key}"
      return cached
    end
    
    # Try L2 (Redis)
    if (cached = Rails.cache.read(key))
      Rails.logger.debug "Cache HIT (L2-redis): #{key}"
      store_in_memory(key, cached, expires_in: [expires_in, 1.minute].min)
      return cached
    end
    
    # Miss - compute and cache
    Rails.logger.debug "Cache MISS: #{key}"
    value = block.call
    
    # Store in both levels
    store_in_memory(key, value, expires_in: [expires_in, 1.minute].min)
    Rails.cache.write(key, value, expires_in: expires_in)
    
    value
  end
  
  def self.invalidate(key)
    MEMORY_CACHE.delete(key)
    MEMORY_CACHE_EXPIRY.delete(key)
    Rails.cache.delete(key)
  end
  
  private
  
  def self.fetch_from_memory(key)
    expiry = MEMORY_CACHE_EXPIRY[key]
    return nil unless expiry && Time.now < expiry
    MEMORY_CACHE[key]
  end
  
  def self.store_in_memory(key, value, expires_in:)
    MEMORY_CACHE[key] = value
    MEMORY_CACHE_EXPIRY[key] = Time.now + expires_in
  end
end

# ใช้งาน
MultiLevelCacheService.fetch("expensive_report", expires_in: 1.hour) do
  ReportService.generate_monthly_report
end
```

### แบบฝึกหัดที่ 8: Profiling Middleware
**โจทย์:** สร้าง custom profiling middleware

**เฉลย:**
```ruby
# lib/middleware/performance_monitor.rb
class PerformanceMonitor
  def initialize(app, options = {})
    @app = app
    @threshold_ms = options.fetch(:threshold_ms, 200)
    @logger = options.fetch(:logger, Rails.logger)
  end
  
  def call(env)
    return @app.call(env) unless monitor?(env)
    
    start_time = Time.now
    query_count = 0
    allocations_before = GC.stat[:total_allocated_objects]
    
    # Count queries
    counter = -> (*, **) { query_count += 1 }
    status, headers, body = nil
    
    ActiveSupport::Notifications.subscribed(counter, "sql.active_record") do
      status, headers, body = @app.call(env)
    end
    
    duration_ms = (Time.now - start_time) * 1000
    allocations = GC.stat[:total_allocated_objects] - allocations_before
    
    log_request(env, duration_ms, query_count, allocations)
    
    [status, headers, body]
  end
  
  private
  
  def monitor?(env)
    !env['PATH_INFO'].start_with?('/assets', '/cable', '/healthz')
  end
  
  def log_request(env, duration_ms, query_count, allocations)
    data = {
      path: env['PATH_INFO'],
      method: env['REQUEST_METHOD'],
      duration_ms: duration_ms.round(2),
      query_count: query_count,
      allocations: allocations,
      slow: duration_ms > @threshold_ms
    }
    
    if data[:slow]
      @logger.warn "SLOW REQUEST: #{data.to_json}"
    else
      @logger.debug "Request: #{data.to_json}"
    end
    
    # StatsD/Prometheus metrics
    StatsD.timing("http.request.duration", duration_ms, tags: ["path:#{env['PATH_INFO']}"])
    StatsD.gauge("http.request.queries", query_count, tags: ["path:#{env['PATH_INFO']}"])
  end
end

# config/application.rb
config.middleware.insert_before(ActionDispatch::RequestId, PerformanceMonitor, 
                                 threshold_ms: 300)
```

### แบบฝึกหัดที่ 9: Lazy Enumerator Optimization
**โจทย์:** Optimize large dataset processing ด้วย Lazy Enumerators

**เฉลย:**
```ruby
require 'benchmark/ips'
require 'memory_profiler'

# Dataset: process large CSV-like data
def generate_data(n = 1_000_000)
  n.times.map { |i| { id: i, value: rand(1000), category: %w[A B C D][i % 4] } }
end

# Bad: load everything into memory
def process_eager(data)
  data.select { |d| d[:category] == 'A' }
      .map { |d| d[:value] * 2 }
      .select { |v| v > 500 }
      .first(100)
end

# Good: lazy evaluation
def process_lazy(data)
  data.lazy
      .select { |d| d[:category] == 'A' }
      .map { |d| d[:value] * 2 }
      .select { |v| v > 500 }
      .first(100)
end

# File-based processing (ดีกว่า - ไม่ต้อง load ทั้งหมด)
def process_file(filename)
  File.foreach(filename).lazy
      .map { |line| JSON.parse(line) }
      .select { |d| d["category"] == "A" }
      .map { |d| d["value"] * 2 }
      .select { |v| v > 500 }
      .first(100)
end

# Infinite sequences
fib = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y.yield a
    a, b = b, a + b
  end
end

# ดึงเฉพาะที่ต้องการ
puts fib.lazy.select { |n| n.even? }.first(10).inspect
puts fib.lazy.find { |n| n > 1_000_000 }

# Memory comparison
data = generate_data(100_000)

report = MemoryProfiler.report { process_eager(data) }
puts "Eager memory: #{report.total_allocated_memsize / 1024}KB"

report = MemoryProfiler.report { process_lazy(data) }
puts "Lazy memory: #{report.total_allocated_memsize / 1024}KB"
```

### แบบฝึกหัดที่ 10: Batch Processing Optimization
**โจทย์:** Optimize bulk operations ใน Rails

**เฉลย:**
```ruby
# Bad: หลาย queries
def update_user_stats_bad
  User.find_each do |user|
    user.update!(
      post_count: user.posts.count,
      order_total: user.orders.sum(:total)
    )
  end
end

# Good: ใช้ bulk SQL update
def update_user_stats_good
  # Single query ด้วย subqueries
  User.connection.execute(<<~SQL)
    UPDATE users
    SET 
      post_count = (
        SELECT COUNT(*) FROM posts WHERE posts.user_id = users.id
      ),
      order_total = (
        SELECT COALESCE(SUM(total), 0) FROM orders WHERE orders.user_id = users.id
      ),
      updated_at = NOW()
  SQL
end

# Better: ใช้ raw SQL bulk insert
def bulk_insert(records)
  return if records.empty?
  
  columns = records.first.keys
  values = records.map do |record|
    "(#{columns.map { |col| ActiveRecord::Base.connection.quote(record[col]) }.join(', ')})"
  end
  
  sql = "INSERT INTO #{table_name} (#{columns.join(', ')}) VALUES #{values.join(', ')}"
  sql += " ON CONFLICT (id) DO UPDATE SET #{update_clause(columns)}"
  
  ActiveRecord::Base.connection.execute(sql)
end

# Benchmark
Benchmark.bm do |x|
  x.report("bad:") { update_user_stats_bad }
  x.report("good:") { update_user_stats_good }
end
```

### แบบฝึกหัดที่ 11: Rails Boot Time Optimization
**โจทย์:** Optimize Rails application boot time

**เฉลย:**
```ruby
# config/boot.rb
ENV["BUNDLE_GEMFILE"] ||= File.expand_path("../Gemfile", __dir__)
require "bundler/setup"

# ไม่ require "bootsnap/setup" ก็ได้ถ้า develop mode ต้องการ fresh code

# config/application.rb
# ใช้ bootsnap สำหรับ faster boot
require "bootsnap/setup"

# Lazy load expensive initializers
config.after_initialize do
  # ย้าย heavy initializations มาไว้ที่นี่
end

# lib/tasks/boot_time.rake
namespace :performance do
  desc "Measure boot time"
  task :boot_time do
    start = Time.now
    require File.expand_path('../config/environment', __dir__)
    duration = Time.now - start
    
    puts "Boot time: #{(duration * 1000).round(0)}ms"
    
    # ตรวจสอบ initializers ที่ช้า
    slow_initializers = []
    
    Rails.application.initializers.each do |init|
      t = Time.now
      # init.run  # ระวัง: อาจ double-initialize
      elapsed = Time.now - t
      
      if elapsed > 0.1  # > 100ms
        slow_initializers << { name: init.name, time: elapsed * 1000 }
      end
    end
    
    if slow_initializers.any?
      puts "\nSlow initializers:"
      slow_initializers.sort_by { |i| -i[:time] }.each do |init|
        puts "  #{init[:name]}: #{init[:time].round(0)}ms"
      end
    end
  end
end
```

### แบบฝึกหัดที่ 12: JRuby Threading
**โจทย์:** ใช้ JRuby สำหรับ parallel HTTP requests

**เฉลย:**
```ruby
# ใช้ JRuby เพื่อ true parallel HTTP requests
require 'net/http'
require 'json'
require 'concurrent-ruby'  # gem 'concurrent-ruby'

# Thread pool ด้วย concurrent-ruby
def parallel_fetch_jruby(urls, timeout: 30)
  pool = Concurrent::FixedThreadPool.new(10)
  futures = urls.map do |url|
    Concurrent::Future.execute(executor: pool) do
      uri = URI(url)
      response = Net::HTTP.get_response(uri)
      JSON.parse(response.body)
    end
  end
  
  results = futures.map { |f| f.value(timeout) }
  pool.shutdown
  results
end

# ใน CRuby, Threads ถูก block ด้วย GIL สำหรับ Ruby code
# แต่ HTTP requests (I/O) ยังคง parallel ได้
# ใน JRuby, ทุกอย่าง truly parallel

urls = [
  "https://api.example.com/users",
  "https://api.example.com/orders",
  "https://api.example.com/products"
]

results = parallel_fetch_jruby(urls)
puts "Fetched #{results.length} resources"
```

### แบบฝึกหัดที่ 13: Custom Memory Allocator
**โจทย์:** ใช้ Object Pool pattern เพื่อลด GC pressure

**เฉลย:**
```ruby
# Object Pool Pattern - reuse objects แทนการสร้างใหม่ทุกครั้ง
class ObjectPool
  def initialize(size: 10, &factory)
    @factory = factory
    @pool = Array.new(size) { factory.call }
    @mutex = Mutex.new
    @available = @pool.dup
  end
  
  def acquire(&block)
    obj = @mutex.synchronize { @available.pop || @factory.call }
    
    begin
      block.call(obj)
    ensure
      @mutex.synchronize { @available.push(obj) }
    end
  end
  
  def stats
    @mutex.synchronize do
      {
        total: @pool.length,
        available: @available.length,
        in_use: @pool.length - @available.length
      }
    end
  end
end

# ตัวอย่าง: Connection pool
class DatabaseConnectionPool < ObjectPool
  def initialize(size: 5)
    super(size: size) { create_connection }
  end
  
  def query(sql)
    acquire do |conn|
      conn.execute(sql)
    end
  end
  
  private
  
  def create_connection
    # สร้าง database connection
    PG.connect(ENV['DATABASE_URL'])
  end
end

# ตัวอย่าง: Buffer pool สำหรับ string building
class StringBufferPool < ObjectPool
  def initialize(size: 20)
    super(size: size) { StringIO.new }
  end
  
  def build(&block)
    acquire do |buffer|
      buffer.truncate(0)
      buffer.rewind
      block.call(buffer)
      buffer.string
    end
  end
end

pool = StringBufferPool.new(size: 10)

threads = 100.times.map do
  Thread.new do
    result = pool.build do |buf|
      buf << "Hello"
      buf << " "
      buf << "World"
    end
    result
  end
end

threads.map(&:value)
puts pool.stats.inspect
```

### แบบฝึกหัดที่ 14: Performance Regression Testing
**โจทย์:** สร้าง performance regression tests

**เฉลย:**
```ruby
# spec/support/performance_helpers.rb
module PerformanceHelpers
  def expect_fast(max_ms: 100, max_queries: 5, max_memory_kb: 1000)
    query_count = 0
    before_memory = current_memory_kb
    start_time = Time.now
    
    # Count queries
    counter = -> (*, **) { query_count += 1 }
    
    ActiveSupport::Notifications.subscribed(counter, "sql.active_record") do
      yield
    end
    
    duration_ms = (Time.now - start_time) * 1000
    memory_used = current_memory_kb - before_memory
    
    aggregate_failures "performance expectations" do
      expect(duration_ms).to be < max_ms,
        "Expected to complete in #{max_ms}ms but took #{duration_ms.round(2)}ms"
      
      expect(query_count).to be <= max_queries,
        "Expected at most #{max_queries} queries but got #{query_count}"
      
      expect(memory_used).to be < max_memory_kb,
        "Expected to use at most #{max_memory_kb}KB but used #{memory_used.round(2)}KB"
    end
  end
  
  private
  
  def current_memory_kb
    `ps -o rss= -p #{Process.pid}`.to_f
  end
end

RSpec.configure do |config|
  config.include PerformanceHelpers, type: :performance
end

# spec/performance/user_service_spec.rb
RSpec.describe UserService, type: :performance do
  let(:users) { create_list(:user, 100, :with_posts) }
  
  it "generates report efficiently" do
    expect_fast(max_ms: 50, max_queries: 3, max_memory_kb: 500) do
      UserService.generate_report(users)
    end
  end
  
  it "loads user with associations efficiently" do
    user = create(:user, :with_full_profile)
    
    expect_fast(max_ms: 10, max_queries: 3) do
      UserService.load_full_profile(user.id)
    end
  end
end
```

### แบบฝึกหัดที่ 15: Concurrent Ruby
**โจทย์:** ใช้ concurrent-ruby สำหรับ async operations

**เฉลย:**
```ruby
require 'concurrent-ruby'

class AsyncReportGenerator
  def initialize
    @executor = Concurrent::ThreadPoolExecutor.new(
      min_threads: 2,
      max_threads: 10,
      max_queue: 100,
      fallback_policy: :caller_runs
    )
  end
  
  def generate_all_reports(user_ids)
    futures = user_ids.map do |user_id|
      Concurrent::Promise.execute(executor: @executor) do
        generate_report(user_id)
      end
    end
    
    # Wait for all with timeout
    Concurrent::Promise.zip(*futures).value!(30)  # 30 second timeout
  end
  
  def generate_with_fallback(user_id)
    Concurrent::Promise.execute(executor: @executor) do
      generate_report(user_id)
    end
    .rescue { |error| fallback_report(user_id, error) }
    .value
  end
  
  def shutdown
    @executor.shutdown
    @executor.wait_for_termination(30)
  end
  
  private
  
  def generate_report(user_id)
    user = User.find(user_id)
    ReportService.generate(user)
  end
  
  def fallback_report(user_id, error)
    Rails.logger.error "Failed to generate report for user #{user_id}: #{error.message}"
    { user_id: user_id, error: error.message, status: 'failed' }
  end
end

# ใช้งาน
generator = AsyncReportGenerator.new
user_ids = User.active.pluck(:id)

reports = generator.generate_all_reports(user_ids)
puts "Generated #{reports.length} reports"

generator.shutdown
```

### แบบฝึกหัดที่ 16: Memory-efficient CSV Processing
**โจทย์:** Process large CSV files โดยไม่ใช้ memory มาก

**เฉลย:**
```ruby
require 'csv'

class LargeCSVProcessor
  BATCH_SIZE = 1000
  
  def self.process(filename, &block)
    processed_count = 0
    
    CSV.foreach(filename, headers: true) do |row|
      block.call(row.to_h)
      processed_count += 1
      
      # GC hint ทุก 10,000 rows
      GC.start if processed_count % 10_000 == 0
    end
    
    processed_count
  end
  
  def self.process_in_batches(filename, batch_size: BATCH_SIZE, &block)
    batch = []
    
    CSV.foreach(filename, headers: true) do |row|
      batch << row.to_h
      
      if batch.length >= batch_size
        block.call(batch)
        batch = []
        GC.compact if defined?(GC.compact)
      end
    end
    
    block.call(batch) unless batch.empty?
  end
  
  def self.to_database(filename, model_class, batch_size: BATCH_SIZE)
    process_in_batches(filename, batch_size: batch_size) do |batch|
      model_class.insert_all(batch.map { |row| sanitize_row(row) })
      print "."
    end
    puts "\nDone!"
  end
  
  private
  
  def self.sanitize_row(row)
    row.transform_values { |v| v.presence }
       .merge(created_at: Time.current, updated_at: Time.current)
  end
end

# ใช้งาน
# แทนที่จะ load ทั้ง CSV ไว้ใน memory
# CSV.read("huge_file.csv").each { ... }  # Bad: loads 1GB ทันที

# Process line by line
LargeCSVProcessor.process("huge_file.csv") do |row|
  # Process each row
end

# หรือ import ทั้ง CSV เข้า database อย่างมีประสิทธิภาพ
LargeCSVProcessor.to_database("users.csv", User)
```

### แบบฝึกหัดที่ 17: Fiber-based Concurrency
**โจทย์:** ใช้ Fibers สำหรับ cooperative multitasking

**เฉลย:**
```ruby
# Fibers ช่วยให้ทำ cooperative multitasking
# เหมาะสำหรับ I/O operations ที่ต้องรอ

class FiberScheduler
  def initialize
    @fibers = []
    @callbacks = {}
  end
  
  def schedule(fiber)
    @fibers << fiber
  end
  
  def run
    until @fibers.empty?
      current = @fibers.shift
      next unless current.alive?
      
      begin
        current.resume
        @fibers << current if current.alive?
      rescue FiberError => e
        puts "Fiber error: #{e.message}"
      end
    end
  end
end

# Producer-Consumer ด้วย Fibers
def producer
  Fiber.new do
    100.times do |i|
      Fiber.yield i  # ส่ง value ออกมา
    end
    nil
  end
end

def consumer(prod)
  results = []
  while (value = prod.resume)
    results << value * 2
  end
  results
end

prod = producer
results = consumer(prod)
puts results.first(10).inspect

# Async HTTP ด้วย Fibers (ใช้ async gem)
# gem 'async'
require 'async'
require 'async/http/internet'

Async do
  internet = Async::HTTP::Internet.new
  
  # Concurrent requests
  responses = [
    "https://api.example.com/users",
    "https://api.example.com/orders"
  ].map do |url|
    Async do
      internet.get(url)
    end
  end
  
  results = responses.map { |t| t.wait.read }
  puts "Got #{results.length} responses"
ensure
  internet.close
end
```

### แบบฝึกหัดที่ 18: Frozen String Optimization
**โจทย์:** Audit และ fix frozen string issues ใน Rails app

**เฉลย:**
```ruby
# lib/tasks/frozen_string_audit.rake
namespace :performance do
  desc "Find mutable strings that could be frozen"
  task :frozen_string_audit do
    require 'parser/current'
    
    files_audited = 0
    issues_found = []
    
    Dir.glob("app/**/*.rb") do |filename|
      files_audited += 1
      
      begin
        source = File.read(filename)
        ast = Parser::CurrentRuby.parse(source)
        
        find_mutable_strings(ast, filename, issues_found)
      rescue Parser::SyntaxError => e
        puts "Parse error in #{filename}: #{e.message}"
      end
    end
    
    puts "Audited #{files_audited} files"
    puts "Found #{issues_found.length} potential issues:"
    
    issues_found.each do |issue|
      puts "  #{issue[:file]}:#{issue[:line]} - #{issue[:description]}"
    end
  end
end

# frozen_string_test.rb - ทดสอบ frozen string benefits
# frozen_string_literal: true

require 'benchmark/ips'
require 'memory_profiler'

# Test 1: String creation frequency
Benchmark.ips do |x|
  x.report("mutable string") do
    str = "hello world"
    str.upcase
  end
  
  x.report("frozen constant") do
    HELLO = "hello world".freeze
    HELLO.upcase
  end
  
  x.compare!
end

# Test 2: Memory impact
report = MemoryProfiler.report do
  100_000.times { "hello" }
end
puts "Mutable: #{report.total_allocated_memsize}B"

# With frozen_string_literal: true ที่ด้านบนไฟล์
report = MemoryProfiler.report do
  100_000.times { "hello" }  # Same object, frozen
end
puts "Frozen: #{report.total_allocated_memsize}B"
```

### แบบฝึกหัดที่ 19: Active Record Batching
**โจทย์:** Optimize large-scale data processing ใน Rails

**เฉลย:**
```ruby
# Bad: load ทั้งหมด
def process_all_orders_bad
  Order.all.each { |order| process(order) }  # อาจ OOM ถ้า orders เยอะ
end

# Good: find_each (batch_size default 1000)
def process_all_orders_good
  Order.find_each(batch_size: 500) { |order| process(order) }
end

# Better: in_batches สำหรับ batch operations
def process_all_orders_better
  Order.in_batches(of: 500) do |batch|
    # Process ทั้ง batch ในครั้งเดียว
    order_ids = batch.pluck(:id)
    results = bulk_process(order_ids)
    
    # Bulk update
    batch.update_all(processed: true, processed_at: Time.current)
  end
end

# Cursor-based pagination สำหรับ real-time safe
class CursorBasedBatchProcessor
  def initialize(model, batch_size: 1000, cursor_column: :id)
    @model = model
    @batch_size = batch_size
    @cursor_column = cursor_column
  end
  
  def process_all
    cursor = nil
    total = 0
    
    loop do
      batch = fetch_batch(cursor)
      break if batch.empty?
      
      yield batch
      
      total += batch.length
      cursor = batch.last.send(@cursor_column)
      
      puts "Processed #{total} records..."
    end
    
    total
  end
  
  private
  
  def fetch_batch(cursor)
    query = @model.order(@cursor_column)
    query = query.where("#{@cursor_column} > ?", cursor) if cursor
    query.limit(@batch_size).to_a
  end
end

# ใช้งาน
processor = CursorBasedBatchProcessor.new(Order, batch_size: 1000)
total = processor.process_all do |batch|
  batch.each { |order| process_order(order) }
end
puts "Processed #{total} orders"
```

### แบบฝึกหัดที่ 20: Complete Performance Audit
**โจทย์:** ทำ complete performance audit สำหรับ Rails app

**เฉลย:**
```ruby
# lib/performance/audit.rb
class PerformanceAudit
  def initialize(app)
    @app = app
    @results = {}
  end
  
  def run
    puts "=== Performance Audit ==="
    
    audit_memory
    audit_queries
    audit_boot_time
    audit_gc_settings
    
    generate_report
  end
  
  private
  
  def audit_memory
    puts "\n--- Memory Analysis ---"
    
    before = current_memory_mb
    
    # Simulate typical workload
    10.times do
      User.includes(:orders, :posts).limit(100).to_a
    end
    
    GC.start(full_mark: true, immediate_sweep: true)
    after_gc = current_memory_mb
    
    @results[:memory] = {
      baseline: before,
      after_workload: after_gc,
      growth: after_gc - before
    }
    
    puts "Baseline: #{before.round(2)} MB"
    puts "After workload + GC: #{after_gc.round(2)} MB"
    puts "Growth: #{(after_gc - before).round(2)} MB"
  end
  
  def audit_queries
    puts "\n--- Query Analysis ---"
    
    slow_queries = []
    query_counts = Hash.new(0)
    
    ActiveSupport::Notifications.subscribed(-> (*args) {
      event = ActiveSupport::Notifications::Event.new(*args)
      query_counts[event.payload[:name]] += 1
      
      if event.duration > 50
        slow_queries << {
          sql: event.payload[:sql][0..100],
          duration: event.duration
        }
      end
    }, "sql.active_record") do
      simulate_workload
    end
    
    @results[:queries] = {
      slow_count: slow_queries.length,
      top_queries: query_counts.sort_by { |_, c| -c }.first(5)
    }
    
    puts "Slow queries (>50ms): #{slow_queries.length}"
    slow_queries.each { |q| puts "  #{q[:duration].round(1)}ms: #{q[:sql]}" }
  end
  
  def audit_boot_time
    puts "\n--- Boot Time ---"
    puts "Current boot: ~#{estimate_boot_time}ms"
    
    puts "Bootsnap: #{defined?(Bootsnap) ? 'enabled' : 'NOT enabled'}"
    puts "Spring: #{defined?(Spring) ? 'available' : 'not available'}"
  end
  
  def audit_gc_settings
    puts "\n--- GC Settings ---"
    
    gc_stat = GC.stat
    puts "Heap pages: #{gc_stat[:heap_allocated_pages]}"
    puts "Live slots: #{gc_stat[:heap_live_slots]}"
    puts "Free slots: #{gc_stat[:heap_free_slots]}"
    puts "Major GC count: #{gc_stat[:major_gc_count]}"
    puts "Minor GC count: #{gc_stat[:minor_gc_count]}"
    
    usage_percent = (gc_stat[:heap_live_slots].to_f / 
                     (gc_stat[:heap_live_slots] + gc_stat[:heap_free_slots]) * 100).round(1)
    
    if usage_percent > 90
      puts "WARNING: Heap is #{usage_percent}% full - consider increasing RUBY_GC_HEAP_FREE_SLOTS"
    end
  end
  
  def generate_report
    puts "\n=== Recommendations ==="
    
    if @results.dig(:memory, :growth)&.> 50
      puts "• High memory growth detected - check for memory leaks"
      puts "  Run: require 'memory_profiler'; MemoryProfiler.report { ... }.pretty_print"
    end
    
    if @results.dig(:queries, :slow_count)&.> 0
      puts "• Slow queries detected - consider:"
      puts "  - Adding database indexes"
      puts "  - Using eager loading"
      puts "  - Implementing query caching"
    end
    
    puts "\nAll done! Performance audit complete."
  end
  
  def current_memory_mb
    `ps -o rss= -p #{Process.pid}`.to_f / 1024
  end
  
  def simulate_workload
    100.times do
      User.includes(:posts).limit(10).to_a
      Order.where(status: 'pending').limit(20).to_a
    end
  end
  
  def estimate_boot_time
    # Get from bootsnap stats or estimate
    Bootsnap::CompileCache::ISeq.stats rescue { misses: 'unknown' }
    "~#{(Rails.application.initializers.count * 5).round(-1)}"
  end
end

# ใช้งาน
# rake performance:audit
namespace :performance do
  task audit: :environment do
    PerformanceAudit.new(Rails.application).run
  end
end
```

---

## สรุปบทที่ 91

ในบทนี้เราได้เรียนรู้:

1. **Ruby Performance Basics** - Benchmark, benchmark-ips
2. **memory_profiler** - การ profile memory allocation และ retention
3. **stackprof** - CPU profiling และ flamegraph analysis
4. **Object Allocation Tracking** - ObjectSpace, allocation_stats
5. **GC Tuning** - GC.compact, environment variables, monitoring
6. **JRuby** - True multithreading, Java interop
7. **TruffleRuby** - GraalVM, polyglot, AOT compilation
8. **Ractors** - Parallel execution ใน CRuby 3.x
9. **ActiveRecord Optimization** - N+1, bulk operations, caching
10. **Production Monitoring** - rack-mini-profiler, application performance monitoring

### สิ่งที่ควรทำต่อ

- ทำ performance audit กับ project จริง
- ลอง TruffleRuby สำหรับ compute-intensive tasks
- Study GC internals ด้วย `ruby --gc-stats`
- ลอง async/await pattern ด้วย async gem
