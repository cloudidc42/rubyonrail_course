# ตอนที่ 25: Concurrency และ Threads (Steps 541-565)

## บทนำ

**Concurrency** คือความสามารถในการทำงานหลายอย่างพร้อมกัน ใน Ruby มีหลายวิธีในการทำงานแบบ concurrent:

1. **Threads** - lightweight processes ที่แชร์ memory
2. **Fibers** - cooperative concurrency (coroutines)
3. **Ractors** - parallel execution (Ruby 3.0+)
4. **Processes** - isolated processes

ความเข้าใจเรื่อง concurrency สำคัญมากสำหรับการเขียน programs ที่มีประสิทธิภาพ

---

## Step 541: Thread Basics

```ruby
# สร้าง Thread ด้วย Thread.new
t = Thread.new do
  puts "Thread is running"
  sleep(1)
  puts "Thread finished"
end

puts "Main thread continues"
t.join  # รอให้ thread เสร็จ
puts "After thread"

# Output:
# Main thread continues
# Thread is running
# Thread finished
# After thread
```

### Thread.new กับ Block

```ruby
# Thread รับ arguments
t = Thread.new(1, 2, 3) do |a, b, c|
  puts "Arguments: #{a}, #{b}, #{c}"
  a + b + c
end

t.join
puts "Thread value: #{t.value}"  # 6

# หลาย threads พร้อมกัน
threads = 5.times.map do |i|
  Thread.new do
    sleep(rand * 0.1)
    puts "Thread #{i} done"
  end
end

threads.each(&:join)
puts "All threads finished"

# Thread กับ shared data
counter = 0
threads = 10.times.map do
  Thread.new { counter += 1 }
end
threads.each(&:join)
puts "Counter: #{counter}"  # อาจไม่ถึง 10! (race condition)
```

---

## Step 542: Thread Methods

```ruby
# สร้าง thread
t = Thread.new { sleep(5); "result" }

# join - รอให้ thread เสร็จ
t.join           # รอไม่มี timeout
t.join(2.0)      # รอสูงสุด 2 วินาที

# value - รับค่า return ของ thread
result = t.value  # block จนกว่า thread จะเสร็จ
puts result       # "result"

# alive? - thread ยังทำงานอยู่ไหม?
t = Thread.new { sleep(1) }
puts t.alive?   # true
t.join
puts t.alive?   # false

# status - สถานะของ thread
t = Thread.new { sleep(0.5) }
puts t.status   # "sleep" หรือ "run"
t.join
puts t.status   # false (terminated normally)

# abort_on_exception - crash program เมื่อ thread crash
Thread.abort_on_exception = true  # global setting
t = Thread.new { raise "Thread error" }
# ถ้าไม่ set นี้ thread error จะถูก ignore!

# per-thread setting
t = Thread.new do
  Thread.current.abort_on_exception = true
  raise "Error in thread"
end

# kill - หยุด thread
t = Thread.new { sleep(10) }
sleep(0.1)
t.kill
puts t.alive?  # false

# Thread.current - thread ปัจจุบัน
Thread.new do
  puts "Current thread: #{Thread.current.inspect}"
  puts "Main thread? #{Thread.current == Thread.main}"
end.join

# Thread.main - main thread
puts Thread.main.inspect

# Thread.list - รายการ threads ทั้งหมด
5.times { Thread.new { sleep(5) } }
puts Thread.list.count  # 6 (main + 5 threads)
Thread.list.each { |t| puts t.status }
```

---

## Step 543: Thread Safety Issues

```ruby
# Race Condition - ปัญหาหลักของ threads
class UnsafeCounter
  attr_reader :count

  def initialize
    @count = 0
  end

  def increment
    # ไม่ thread-safe! (read-modify-write ไม่ atomic)
    temp = @count
    sleep(0.001)  # simulate delay
    @count = temp + 1
  end
end

counter = UnsafeCounter.new
threads = 100.times.map { Thread.new { counter.increment } }
threads.each(&:join)
puts "Expected: 100, Got: #{counter.count}"  # อาจได้ < 100!

# Memory Visibility
# Thread A เปลี่ยน variable แต่ Thread B อาจไม่เห็นการเปลี่ยนแปลง
# เพราะ CPU cache

# Deadlock - threads ต่างรอกันอยู่
mutex1 = Mutex.new
mutex2 = Mutex.new

t1 = Thread.new do
  mutex1.lock
  sleep(0.1)
  mutex2.lock  # รอ mutex2 ที่ t2 lock ไว้
  mutex1.unlock
  mutex2.unlock
end

t2 = Thread.new do
  mutex2.lock
  sleep(0.1)
  mutex1.lock  # รอ mutex1 ที่ t1 lock ไว้ -> DEADLOCK!
  mutex2.unlock
  mutex1.unlock
end

# ป้องกัน deadlock: lock mutexes ในลำดับเดิมเสมอ
# หรือใช้ try_lock

# LiveLock - threads ทำงานแต่ไม่ progress
# Starvation - thread บางตัวไม่ได้รับ CPU time
```

---

## Step 544: Mutex - Mutual Exclusion

```ruby
require 'thread'

# Mutex ป้องกัน race condition
class SafeCounter
  attr_reader :count

  def initialize
    @count = 0
    @mutex = Mutex.new
  end

  def increment
    @mutex.synchronize do
      @count += 1
    end
  end

  def decrement
    @mutex.synchronize do
      @count -= 1
    end
  end

  def value
    @mutex.synchronize { @count }
  end
end

counter = SafeCounter.new
threads = 1000.times.map { Thread.new { counter.increment } }
threads.each(&:join)
puts "Count: #{counter.count}"  # 1000 เสมอ!

# Mutex methods
mutex = Mutex.new

# synchronize - lock, yield, unlock
mutex.synchronize { "critical section" }

# lock / unlock manual (ระวัง!)
mutex.lock
begin
  # critical section
ensure
  mutex.unlock  # unlock เสมอ!
end

# try_lock - ไม่ block
if mutex.try_lock
  begin
    # critical section
  ensure
    mutex.unlock
  end
else
  puts "Mutex is busy, skipping"
end

# owned? - thread นี้เป็นเจ้าของ mutex หรือไม่
mutex.synchronize { puts mutex.owned? }  # true

# locked? - mutex ถูก lock อยู่หรือไม่
puts mutex.locked?  # false (เราไม่ได้ lock)

# ตัวอย่างจริง: Thread-safe cache
class ThreadSafeCache
  def initialize
    @store = {}
    @mutex = Mutex.new
  end

  def get(key)
    @mutex.synchronize { @store[key] }
  end

  def set(key, value)
    @mutex.synchronize { @store[key] = value }
  end

  def fetch(key, &block)
    @mutex.synchronize do
      @store[key] ||= block.call
    end
  end

  def delete(key)
    @mutex.synchronize { @store.delete(key) }
  end

  def size
    @mutex.synchronize { @store.size }
  end
end

cache = ThreadSafeCache.new
threads = 10.times.map do |i|
  Thread.new do
    cache.set("key_#{i}", i * 100)
    sleep(0.01)
    puts "key_#{i} = #{cache.get("key_#{i}")}"
  end
end
threads.each(&:join)
puts "Cache size: #{cache.size}"
```

---

## Step 545: Monitor

```ruby
require 'monitor'

# Monitor คล้าย Mutex แต่ reentrant (ล็อคซ้ำได้จาก thread เดิม)
class SafeQueue
  include MonitorMixin

  def initialize
    super  # ต้องเรียก super!
    @data = []
    @not_empty = new_cond  # condition variable
  end

  def push(item)
    synchronize do
      @data.push(item)
      @not_empty.signal
    end
  end

  def pop
    synchronize do
      @not_empty.wait_while { @data.empty? }
      @data.shift
    end
  end

  def size
    synchronize { @data.size }
  end

  def empty?
    synchronize { @data.empty? }
  end
end

queue = SafeQueue.new

# Producer
producer = Thread.new do
  5.times do |i|
    sleep(0.1)
    queue.push("item_#{i}")
    puts "Pushed: item_#{i}"
  end
end

# Consumer
consumer = Thread.new do
  5.times do
    item = queue.pop
    puts "Popped: #{item}"
  end
end

[producer, consumer].each(&:join)

# MonitorMixin กับ reentrant locking
class Counter
  include MonitorMixin

  def initialize
    super
    @count = 0
  end

  def increment
    synchronize do
      @count += 1
      # !! สามารถเรียก method อื่นที่ใช้ synchronize ได้!
      check_limit
    end
  end

  def value
    synchronize { @count }
  end

  private

  def check_limit
    synchronize do  # reentrant - thread เดิมสามารถ lock ได้อีก!
      puts "Count is #{@count}" if @count % 10 == 0
    end
  end
end

# กับ Mutex ธรรมดา check_limit จะ deadlock!
# Monitor ใช้ได้เพราะ reentrant
```

---

## Step 546: Thread Variables (Thread-local)

```ruby
# Thread-local variables - แต่ละ thread มีค่าของตัวเอง
Thread.current[:name] = "Main Thread"

t = Thread.new do
  Thread.current[:name] = "Worker Thread"
  puts Thread.current[:name]  # "Worker Thread"
end

t.join
puts Thread.current[:name]  # "Main Thread" (ไม่เปลี่ยน)

# thread_variable_get และ thread_variable_set (Ruby 2.0+)
Thread.current.thread_variable_set(:user_id, 1)
puts Thread.current.thread_variable_get(:user_id)  # 1

# ตัวอย่างจริง: Request tracking
class RequestContext
  def self.set(user_id:, request_id:)
    Thread.current[:user_id]    = user_id
    Thread.current[:request_id] = request_id
  end

  def self.user_id
    Thread.current[:user_id]
  end

  def self.request_id
    Thread.current[:request_id]
  end

  def self.clear
    Thread.current[:user_id]    = nil
    Thread.current[:request_id] = nil
  end
end

# Simulate web request handling
def handle_request(user_id, request_id)
  Thread.new do
    RequestContext.set(user_id: user_id, request_id: request_id)
    # ทุกที่ใน code stack สามารถเข้าถึง context ได้
    puts "Processing request #{RequestContext.request_id} for user #{RequestContext.user_id}"
    sleep(0.1)
    log_action("viewed_page")
  end
end

def log_action(action)
  puts "[#{RequestContext.request_id}] User #{RequestContext.user_id}: #{action}"
end

threads = 3.times.map { |i| handle_request(i + 1, "REQ-#{1000 + i}") }
threads.each(&:join)

# Fiber-local variables (ต่างจาก Thread-local)
# Thread.current[:key] = Thread-local
# Fiber.yield  - Fiber local (ไม่มีใน simple case)
```

---

## Step 547: Thread Status and Priority

```ruby
# Thread states: "run", "sleep", "aborting", false (dead), nil (killed)
t = Thread.new do
  puts "Status while running: #{Thread.current.status}"  # "run"
  sleep(1)
end

sleep(0.1)
puts "Status while sleeping: #{t.status}"  # "sleep"
t.join
puts "Status after join: #{t.status}"      # false

# Priority (-3 to 3, higher = more CPU time)
t = Thread.new do
  Thread.current.priority = 2
  # high priority work
  1_000_000.times { |i| i * i }
end

puts t.priority  # 2
t.join

# ตัวอย่าง: background vs foreground
Thread.new do
  Thread.current.priority = -1  # low priority
  loop do
    # background cleanup
    sleep(1)
  end
end

# Main work (default priority 0)
10.times { |i| puts "Main work #{i}" }

# Thread.pass - hint to scheduler
Thread.new do
  100.times do |i|
    Thread.pass if i % 10 == 0  # yield CPU
    process_item(i)
  end
end

# stop และ run
t = Thread.new { Thread.stop; puts "Resumed!" }
sleep(0.1)
t.run  # wake up stopped thread
t.join

# wakeup
t = Thread.new do
  sleep(10)
  puts "Woke up!"
end
sleep(0.1)
t.wakeup  # interrupt sleep
t.join
```

---

## Step 548: Fiber และ Coroutines

```ruby
# Fiber - cooperative multitasking (ไม่ใช่ preemptive)
# Fiber เลือกว่าจะ yield เมื่อไหร่เอง

# สร้าง Fiber
fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield   # หยุดและส่ง control กลับ
  puts "Step 2"
  Fiber.yield   # หยุดอีกครั้ง
  puts "Step 3"
  # จบ Fiber
end

puts "Before first resume"
fiber.resume    # รัน "Step 1", yield
puts "Between steps"
fiber.resume    # รัน "Step 2", yield
puts "Almost done"
fiber.resume    # รัน "Step 3", finish
puts "Done"

# Output:
# Before first resume
# Step 1
# Between steps
# Step 2
# Almost done
# Step 3
# Done

# Fiber กับ values
fiber = Fiber.new do |first|
  puts "Received: #{first}"
  second = Fiber.yield first + 10
  puts "Received: #{second}"
  second + 20
end

result1 = fiber.resume(1)    # ส่ง 1 เข้าไป
puts "Got: #{result1}"       # 11 (1 + 10)

result2 = fiber.resume(100)  # ส่ง 100 เข้าไป
puts "Got: #{result2}"       # 120 (100 + 20)

# Fiber ใช้สร้าง generators
def counter_generator(start = 0)
  Fiber.new do
    n = start
    loop do
      Fiber.yield n
      n += 1
    end
  end
end

counter = counter_generator(5)
puts counter.resume  # 5
puts counter.resume  # 6
puts counter.resume  # 7

# Infinite sequence
def fibonacci_generator
  Fiber.new do
    a, b = 0, 1
    loop do
      Fiber.yield a
      a, b = b, a + b
    end
  end
end

fib = fibonacci_generator
10.times { print "#{fib.resume} " }
puts
# 0 1 1 2 3 5 8 13 21 34

# Fiber.alive?
f = Fiber.new { "done" }
puts f.alive?   # true
f.resume
puts f.alive?   # false
```

---

## Step 549: Enumerator as Fiber

```ruby
# Enumerator ใช้ Fiber ภายใน
enum = Enumerator.new do |yielder|
  yielder << 1
  yielder << 2
  yielder << 3
end

puts enum.next  # 1
puts enum.next  # 2
puts enum.next  # 3
begin
  enum.next
rescue StopIteration
  puts "End of enumerator"
end

# Rewind
enum.rewind
puts enum.next  # 1 อีกครั้ง

# External iteration (ด้วย Fiber-like behavior)
enum = [1, 2, 3].each
puts enum.next  # 1
puts enum.next  # 2
puts enum.next  # 3

# Custom Enumerator ที่ซับซ้อน
def file_line_reader(filename)
  Enumerator.new do |yielder|
    File.foreach(filename) do |line|
      yielder.yield line.chomp
    end
  end
end

# ใช้ lazy evaluation
# lines = file_line_reader("large_file.txt")
# first_10 = lines.lazy.first(10)

# Fiber-based producer-consumer
producer = Fiber.new do
  5.times do |i|
    puts "Producing #{i}"
    Fiber.yield i
  end
  nil  # signal done
end

consumer = Fiber.new do |item|
  loop do
    break if item.nil?
    puts "Consuming #{item}"
    item = Fiber.yield
  end
end

# Connect producer and consumer
consumer.resume
loop do
  item = producer.resume
  break if item.nil?
  consumer.resume(item)
end
puts "Pipeline complete"
```

---

## Step 550: Ractor - Parallel Execution (Ruby 3.0+)

```ruby
# Ractor - true parallelism ใน Ruby 3.0+
# แต่ละ Ractor รันบน thread ของตัวเอง และไม่แชร์ memory
# (ทำลาย GIL ได้!)

# สร้าง Ractor
r = Ractor.new do
  puts "Ractor is running"
  "result from ractor"
end

result = r.take  # รับค่า
puts result      # "result from ractor"

# Ractor กับ arguments
r = Ractor.new("Hello", "World") do |a, b|
  "#{a} #{b}"
end
puts r.take  # "Hello World"

# ส่ง message ระหว่าง Ractors
r = Ractor.new do
  message = Ractor.receive  # รับ message
  "Got: #{message}"
end

r.send("Hello Ractor")
puts r.take  # "Got: Hello Ractor"

# Parallel computation (ไม่ติด GIL!)
require 'benchmark'

def prime?(n)
  return false if n < 2
  (2..Math.sqrt(n)).none? { |i| n % i == 0 }
end

numbers = (1..100_000).to_a

# Sequential
sequential_time = Benchmark.realtime do
  numbers.count { |n| prime?(n) }
end

# Parallel with Ractors
parallel_time = Benchmark.realtime do
  # แบ่งงาน
  chunks = numbers.each_slice(numbers.size / 4).to_a

  ractors = chunks.map do |chunk|
    Ractor.new(chunk) do |nums|
      nums.count { |n| prime?(n) }
    end
  end

  # รวมผล
  total = ractors.sum(&:take)
end

puts "Sequential: #{sequential_time.round(2)}s"
puts "Parallel: #{parallel_time.round(2)}s"
# Parallel เร็วกว่า (ถ้า cores มากพอ)!

# Ractor limitations:
# - ไม่สามารถแชร์ mutable objects
# - String, Integer, Float, Symbol, true, false, nil ส่งได้
# - Object ที่ frozen? ส่งได้
# - ต้อง "move" mutable objects (ทำให้ object เดิม unusable)

# Shareable objects
puts Ractor.shareable?(42)      # true
puts Ractor.shareable?("hello") # false (mutable string)
puts Ractor.shareable?("hello".freeze)  # true

# ทำ String shareable
str = "hello"
Ractor.make_shareable(str)
puts Ractor.shareable?(str)  # true
```

---

## Step 551: GIL/GVL Explained

```ruby
# GIL = Global Interpreter Lock
# GVL = Global VM Lock (ชื่อที่ถูกต้องกว่าใน MRI Ruby)

# GVL ทำให้ Ruby threads ไม่ parallel (ใน CRuby/MRI)
# ทำให้ CPU-bound tasks ไม่ได้ประโยชน์จาก threads

# I/O-bound: threads ยังมีประโยชน์ (GVL release ระหว่าง I/O)
# CPU-bound: ไม่ได้ประโยชน์จาก threads (ต้องใช้ Ractors หรือ Processes)

# ตัวอย่าง: I/O-bound (threads มีประโยชน์)
require 'net/http'
require 'benchmark'

urls = ["https://httpbin.org/delay/0.1"] * 10

# Sequential
seq_time = Benchmark.realtime do
  urls.each do |url|
    # Net::HTTP.get(URI(url))  # simulate
    sleep(0.1)  # simulate I/O
  end
end

# Concurrent threads
thread_time = Benchmark.realtime do
  threads = urls.map do |url|
    Thread.new do
      sleep(0.1)  # simulate I/O - GVL released!
    end
  end
  threads.each(&:join)
end

puts "Sequential: #{seq_time.round(2)}s"   # ~1.0s
puts "Threads: #{thread_time.round(2)}s"    # ~0.1s (parallel I/O!)

# ตัวอย่าง: CPU-bound (threads ไม่มีประโยชน์ใน MRI)
def cpu_intensive(n)
  n.times { |i| i ** 2 }
end

# Sequential CPU work
seq_cpu = Benchmark.realtime do
  2.times { cpu_intensive(1_000_000) }
end

# Thread CPU work (ไม่เร็วขึ้น เพราะ GVL)
thread_cpu = Benchmark.realtime do
  threads = 2.times.map { Thread.new { cpu_intensive(1_000_000) } }
  threads.each(&:join)
end

puts "Sequential CPU: #{seq_cpu.round(2)}s"
puts "Thread CPU: #{thread_cpu.round(2)}s"  # ไม่เร็วขึ้น!

# ทางออกสำหรับ CPU-bound:
# 1. Ractors (Ruby 3.0+)
# 2. JRuby/TruffleRuby (ไม่มี GVL)
# 3. fork/processes
# 4. C extensions
```

---

## Step 552: concurrent-ruby Gem

```ruby
# gem 'concurrent-ruby' - thread-safe data structures และ patterns

require 'concurrent-ruby'

# ==========================================
# Atomic operations
# ==========================================
counter = Concurrent::AtomicFixnum.new(0)
threads = 100.times.map { Thread.new { counter.increment } }
threads.each(&:join)
puts counter.value  # 100 เสมอ!

# AtomicBoolean
flag = Concurrent::AtomicBoolean.new(false)
threads = 10.times.map do
  Thread.new { flag.make_true }
end
threads.each(&:join)
puts flag.true?  # true

# AtomicReference - ครอบ object ใดก็ได้
ref = Concurrent::AtomicReference.new({ count: 0 })
threads = 10.times.map do
  Thread.new do
    ref.update { |val| { count: val[:count] + 1 } }
  end
end
threads.each(&:join)
puts ref.value[:count]  # 10

# ==========================================
# Thread-safe data structures
# ==========================================

# Array
arr = Concurrent::Array.new
threads = 10.times.map { |i| Thread.new { arr << i } }
threads.each(&:join)
puts arr.sort.inspect  # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# Hash
hash = Concurrent::Hash.new
threads = 10.times.map { |i| Thread.new { hash[i] = i * 2 } }
threads.each(&:join)
puts hash.sort.to_h.inspect

# Map (like Hash)
map = Concurrent::Map.new
map["key"] = "value"
puts map["key"]
map.compute_if_absent("new_key") { "default" }

# ==========================================
# Future (async computation)
# ==========================================
future = Concurrent::Future.execute do
  sleep(1)
  42
end

puts future.state   # :pending
sleep(1.5)
puts future.state   # :fulfilled
puts future.value   # 42

# Chaining futures
result = Concurrent::Future.execute { 10 }
  .then { |v| v * 2 }
  .then { |v| v + 5 }

puts result.value  # 25

# Error handling
bad_future = Concurrent::Future.execute { raise "error!" }
bad_future.wait
puts bad_future.rejected?   # true
puts bad_future.reason.message  # "error!"

# ==========================================
# Promise
# ==========================================
promise = Concurrent::Promise.execute { "hello" }
  .then { |v| v.upcase }
  .then { |v| "#{v}!" }

promise.wait
puts promise.value  # "HELLO!"

# ==========================================
# Async (simple async mixin)
# ==========================================
class AsyncCalculator
  include Concurrent::Async

  def compute(n)
    sleep(0.1)
    n ** 2
  end
end

calc = AsyncCalculator.new
future = calc.async.compute(10)  # async call
puts future.value  # 100

# Await (blocking)
result = calc.await.compute(5)
puts result.value  # 25
```

---

## Step 553: Async Programming Patterns

```ruby
# ==========================================
# Pattern 1: Thread Pool
# ==========================================
class ThreadPool
  def initialize(size)
    @size    = size
    @queue   = Queue.new
    @threads = Array.new(size) { spawn_worker }
  end

  def submit(&block)
    future = Concurrent::Future.new(&block)
    @queue << future
    future
  end

  def shutdown
    @size.times { @queue << :stop }
    @threads.each(&:join)
  end

  private

  def spawn_worker
    Thread.new do
      loop do
        task = @queue.pop
        break if task == :stop
        task.execute
      end
    end
  end
end

pool = ThreadPool.new(4)

futures = 20.times.map do |i|
  pool.submit { sleep(0.1); i * 2 }
end

pool.shutdown
results = futures.map(&:value)
puts results.inspect

# ==========================================
# Pattern 2: Producer-Consumer
# ==========================================
require 'thread'

class Pipeline
  def initialize(workers: 4)
    @queue   = SizedQueue.new(100)
    @workers = workers
    @results = Queue.new
  end

  def process(items, &work)
    # Start workers
    worker_threads = @workers.times.map do
      Thread.new do
        while (item = @queue.pop) != :done
          result = work.call(item)
          @results << result
        end
      end
    end

    # Feed queue
    producer = Thread.new do
      items.each { |item| @queue << item }
      @workers.times { @queue << :done }
    end

    producer.join
    worker_threads.each(&:join)

    # Collect results
    results = []
    results << @results.pop until @results.empty?
    results
  end
end

pipeline = Pipeline.new(workers: 4)
results = pipeline.process((1..20).to_a) { |n| n ** 2 }
puts results.sort.inspect

# ==========================================
# Pattern 3: Event-based
# ==========================================
class EventEmitter
  def initialize
    @listeners = Hash.new { |h, k| h[k] = [] }
    @mutex     = Mutex.new
  end

  def on(event, &handler)
    @mutex.synchronize { @listeners[event] << handler }
  end

  def emit(event, *args)
    handlers = @mutex.synchronize { @listeners[event].dup }
    handlers.each { |h| Thread.new { h.call(*args) } }
  end

  def once(event, &handler)
    wrapper = nil
    wrapper = lambda do |*args|
      handler.call(*args)
      @mutex.synchronize { @listeners[event].delete(wrapper) }
    end
    on(event, &wrapper)
  end
end

emitter = EventEmitter.new
emitter.on(:data)    { |d| puts "Handler 1: #{d}" }
emitter.on(:data)    { |d| puts "Handler 2: #{d.upcase}" }
emitter.once(:ready) { puts "Ready! (only once)" }

emitter.emit(:ready)
emitter.emit(:ready)  # ไม่ print
emitter.emit(:data, "hello")
sleep(0.1)
```

---

## Step 554: Parallel Gem

```ruby
# gem 'parallel' - simple parallel processing

require 'parallel'

# ==========================================
# Parallel.map - parallel version ของ map
# ==========================================
results = Parallel.map([1, 2, 3, 4, 5], in_threads: 4) do |n|
  sleep(0.1)  # simulate work
  n ** 2
end
puts results.inspect  # [1, 4, 9, 16, 25] - เร็วกว่า sequential!

# ==========================================
# in_processes vs in_threads
# ==========================================
# in_processes: ใช้ fork - ดีสำหรับ CPU-bound (หลีกเลี่ยง GVL)
# in_threads: ใช้ threads - ดีสำหรับ I/O-bound

results = Parallel.map((1..8).to_a, in_processes: 4) do |n|
  # CPU-intensive work
  n.times.sum { |i| i ** 2 }
end
puts results.inspect

# ==========================================
# Parallel.each
# ==========================================
Parallel.each([1, 2, 3, 4, 5], in_threads: 3) do |n|
  puts "Processing #{n} in thread #{Thread.current.object_id}"
end

# ==========================================
# Parallel.inject/reduce
# ==========================================
# ระวัง! ผลลัพธ์อาจไม่เป็น order ที่ต้องการ

# ==========================================
# Progress tracking
# ==========================================
require 'ruby-progressbar'  # gem

results = Parallel.map((1..100).to_a,
                        in_threads: 10,
                        progress: "Processing") do |n|
  sleep(0.05)
  n ** 2
end

# ==========================================
# Error handling
# ==========================================
begin
  Parallel.map([1, 2, "bad", 4], in_threads: 2) do |n|
    raise ArgumentError, "Not a number: #{n}" unless n.is_a?(Integer)
    n * 2
  end
rescue Parallel::UndumpableException => e
  puts "Parallel error: #{e.message}"
end

# ==========================================
# ตัวอย่างจริง: Bulk API requests
# ==========================================
def fetch_user(id)
  # simulate API call
  sleep(0.1)
  { id: id, name: "User #{id}", score: rand(100) }
end

user_ids = (1..50).to_a

# Sequential: ~5 seconds
# seq_time = Benchmark.realtime { user_ids.map { |id| fetch_user(id) } }

# Parallel threads: ~0.5 seconds
par_time = Benchmark.realtime do
  Parallel.map(user_ids, in_threads: 10) { |id| fetch_user(id) }
end

puts "Parallel: #{par_time.round(1)}s"
```

---

## Step 555: ตัวอย่าง Concurrency จริง

```ruby
# ตัวอย่าง 1: Web Crawler แบบ Concurrent
require 'thread'

class WebCrawler
  def initialize(max_threads: 5)
    @max_threads  = max_threads
    @queue        = Queue.new
    @visited      = Concurrent::Set.new
    @results      = Concurrent::Array.new
    @mutex        = Mutex.new
  end

  def crawl(start_url, max_pages: 20)
    @queue << start_url

    threads = @max_threads.times.map do
      Thread.new { worker(max_pages) }
    end

    threads.each(&:join)
    @results.to_a
  end

  private

  def worker(max_pages)
    while @results.size < max_pages
      url = begin
        @queue.pop(true)  # non-blocking
      rescue ThreadError
        break
      end

      next if @visited.include?(url)
      @visited.add(url)

      # Simulate fetching
      page_data = fetch_page(url)
      next unless page_data

      @results << page_data

      # Add new links to queue
      page_data[:links].each { |link| @queue << link }
    end
  end

  def fetch_page(url)
    sleep(0.1)  # simulate HTTP request
    {
      url:   url,
      title: "Page at #{url}",
      links: ["#{url}/page1", "#{url}/page2"]
    }
  rescue
    nil
  end
end

crawler = WebCrawler.new(max_threads: 5)
results = crawler.crawl("https://example.com", max_pages: 10)
puts "Crawled #{results.size} pages"

# ตัวอย่าง 2: Background Job Queue
class JobQueue
  Job = Struct.new(:id, :payload, :attempts, :created_at)

  def initialize(workers: 3)
    @queue   = SizedQueue.new(50)
    @workers = workers
    @results = Concurrent::Hash.new
    @mutex   = Mutex.new
    @running = Concurrent::AtomicBoolean.new(false)
  end

  def start
    @running.make_true
    @worker_threads = @workers.times.map { spawn_worker }
    puts "Job queue started with #{@workers} workers"
    self
  end

  def stop
    @running.make_false
    @workers.times { @queue << :poison_pill }
    @worker_threads.each(&:join)
    puts "Job queue stopped"
  end

  def enqueue(payload)
    job = Job.new(
      SecureRandom.hex(8),
      payload,
      0,
      Time.now
    )
    @queue << job
    job.id
  end

  def result(job_id, timeout: 10)
    deadline = Time.now + timeout
    until @results.key?(job_id) || Time.now > deadline
      sleep(0.1)
    end
    @results.delete(job_id)
  end

  def stats
    {
      queue_size:  @queue.size,
      workers:     @workers,
      completed:   @results.size
    }
  end

  private

  def spawn_worker
    Thread.new do
      while @running.true?
        job = @queue.pop
        break if job == :poison_pill

        process_job(job)
      end
    end
  end

  def process_job(job)
    puts "Processing job #{job.id}"
    result = job.payload.call  # execute the job
    @results[job.id] = { status: :success, result: result }
  rescue => e
    @results[job.id] = { status: :failed, error: e.message }
  end
end

require 'securerandom'
require 'concurrent-ruby'

queue = JobQueue.new(workers: 3).start

# Enqueue jobs
job_ids = 10.times.map do |i|
  queue.enqueue(-> { sleep(0.1); i * 100 })
end

# Wait for results
job_ids.each do |id|
  result = queue.result(id)
  puts "Job #{id}: #{result.inspect}"
end

queue.stop
```

---

## แบบฝึกหัด: Concurrency (25 ข้อ)

**ข้อ 1:** สร้าง Thread-safe bank account

```ruby
# เฉลย
class ThreadSafeBankAccount
  def initialize(balance)
    @balance = balance
    @mutex   = Mutex.new
    @log     = []
  end

  def deposit(amount)
    @mutex.synchronize do
      @balance += amount
      @log << "Deposit: +#{amount} -> #{@balance}"
    end
  end

  def withdraw(amount)
    @mutex.synchronize do
      raise "Insufficient funds" if amount > @balance
      @balance -= amount
      @log << "Withdrawal: -#{amount} -> #{@balance}"
    end
  end

  def balance
    @mutex.synchronize { @balance }
  end

  def transaction_log
    @mutex.synchronize { @log.dup }
  end
end

account = ThreadSafeBankAccount.new(1000)
threads = []
10.times { threads << Thread.new { account.deposit(100) } }
5.times  { threads << Thread.new { account.withdraw(50) } }
threads.each(&:join)

puts "Final balance: #{account.balance}"  # 1750
puts "Transactions: #{account.transaction_log.length}"  # 15
```

**ข้อ 2:** สร้าง Fiber-based generator สำหรับ Fibonacci

```ruby
# เฉลย
def fibonacci_fiber
  Fiber.new do
    a, b = 0, 1
    loop do
      Fiber.yield a
      a, b = b, a + b
    end
  end
end

fib = fibonacci_fiber
first_15 = 15.times.map { fib.resume }
puts first_15.inspect
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377]
```

**ข้อ 3:** Implement simple thread pool

```ruby
# เฉลย
class SimpleThreadPool
  def initialize(size)
    @queue   = Queue.new
    @threads = size.times.map do
      Thread.new do
        loop do
          task, promise = @queue.pop
          break if task == :shutdown
          begin
            result = task.call
            promise[:result] = result
          rescue => e
            promise[:error] = e
          ensure
            promise[:done] = true
          end
        end
      end
    end
  end

  def submit(&block)
    promise = { done: false }
    @queue << [block, promise]
    promise
  end

  def shutdown
    @threads.size.times { @queue << [:shutdown, nil] }
    @threads.each(&:join)
  end

  def wait_for(promise, timeout = 10)
    deadline = Time.now + timeout
    until promise[:done] || Time.now > deadline
      sleep(0.01)
    end
    raise promise[:error] if promise[:error]
    promise[:result]
  end
end

pool = SimpleThreadPool.new(4)
promises = 8.times.map { |i| pool.submit { sleep(0.1); i ** 2 } }
results = promises.map { |p| pool.wait_for(p) }
pool.shutdown

puts "Results: #{results.sort.inspect}"
```

**ข้อ 4-10:** (ตัวอย่างย่อ)

```ruby
# ข้อ 4: Read-Write lock
class ReadWriteLock
  def initialize
    @readers       = 0
    @writer_active = false
    @mutex         = Mutex.new
    @read_ok       = ConditionVariable.new
    @write_ok      = ConditionVariable.new
  end

  def read_lock
    @mutex.synchronize do
      @read_ok.wait(@mutex) while @writer_active
      @readers += 1
    end
    begin
      yield
    ensure
      @mutex.synchronize do
        @readers -= 1
        @write_ok.signal if @readers == 0
      end
    end
  end

  def write_lock
    @mutex.synchronize do
      @write_ok.wait(@mutex) while @writer_active || @readers > 0
      @writer_active = true
    end
    begin
      yield
    ensure
      @mutex.synchronize do
        @writer_active = false
        @read_ok.broadcast
        @write_ok.signal
      end
    end
  end
end

rw_lock = ReadWriteLock.new
shared_data = []

readers = 5.times.map do
  Thread.new do
    rw_lock.read_lock { puts "Reading: #{shared_data.length}" }
  end
end

writer = Thread.new do
  rw_lock.write_lock do
    shared_data << "new item"
    puts "Wrote: #{shared_data.length} items"
  end
end

(readers + [writer]).each(&:join)

# ข้อ 5: Parallel map implementation
def parallel_map(collection, threads: 4, &block)
  return collection.map(&block) if collection.empty?

  results = Array.new(collection.size)
  mutex   = Mutex.new
  index   = 0

  thread_pool = threads.times.map do
    Thread.new do
      loop do
        i = mutex.synchronize { index.tap { index += 1 } }
        break if i >= collection.size
        results[i] = block.call(collection[i])
      end
    end
  end

  thread_pool.each(&:join)
  results
end

result = parallel_map([1, 2, 3, 4, 5, 6, 7, 8]) { |n| n ** 2 }
puts result.inspect  # [1, 4, 9, 16, 25, 36, 49, 64]

# ข้อ 6: Fiber coroutine ping-pong
def ping(other_fiber)
  Fiber.new do
    loop do
      puts "Ping!"
      other_fiber.resume rescue Fiber.yield
    end
  end
end

def pong
  Fiber.new do
    loop do
      puts "Pong!"
      Fiber.yield
    end
  end
end

pong_fiber = pong
ping_fiber = ping(pong_fiber)

5.times { ping_fiber.resume }

# ข้อ 7: Thread timeout
def with_timeout(seconds, &block)
  thread = Thread.new(&block)
  result = thread.join(seconds)
  if result.nil?
    thread.kill
    raise Timeout::Error, "Operation timed out after #{seconds}s"
  end
  thread.value
end

begin
  result = with_timeout(1.0) { sleep(2); "done" }
rescue Timeout::Error => e
  puts "Timeout: #{e.message}"
end

# ข้อ 8: Concurrent hash counter
class ConcurrentWordCounter
  def initialize
    @counts = Concurrent::Hash.new(0)
    @mutex  = Mutex.new
  end

  def count_document(text)
    words = text.downcase.scan(/\w+/)
    words.each do |word|
      @mutex.synchronize { @counts[word] += 1 }
    end
  end

  def top_words(n = 10)
    @counts.sort_by { |_, v| -v }.first(n)
  end
end

counter = ConcurrentWordCounter.new
documents = [
  "the quick brown fox jumps over the lazy dog",
  "the fox was very quick and the dog was very lazy",
  "ruby is a great language for concurrency"
]

threads = documents.map do |doc|
  Thread.new { counter.count_document(doc) }
end
threads.each(&:join)

puts counter.top_words(5).inspect

# ข้อ 9: Semaphore ด้วย Mutex
class Semaphore
  def initialize(count)
    @count = count
    @mutex = Mutex.new
    @cond  = ConditionVariable.new
  end

  def acquire
    @mutex.synchronize do
      @cond.wait(@mutex) while @count <= 0
      @count -= 1
    end
  end

  def release
    @mutex.synchronize do
      @count += 1
      @cond.signal
    end
  end

  def with_semaphore
    acquire
    begin
      yield
    ensure
      release
    end
  end
end

sem = Semaphore.new(3)  # max 3 concurrent
threads = 10.times.map do |i|
  Thread.new do
    sem.with_semaphore do
      puts "Thread #{i} running (at most 3 at once)"
      sleep(0.2)
      puts "Thread #{i} done"
    end
  end
end
threads.each(&:join)

# ข้อ 10: Async event processor
class AsyncEventProcessor
  def initialize
    @handlers  = Hash.new { |h, k| h[k] = [] }
    @queue     = Queue.new
    @thread    = Thread.new { process_loop }
    @mutex     = Mutex.new
  end

  def subscribe(event, &handler)
    @mutex.synchronize { @handlers[event] << handler }
  end

  def publish(event, data = nil)
    @queue << [event, data]
  end

  def shutdown
    @queue << :shutdown
    @thread.join
  end

  private

  def process_loop
    loop do
      event, data = @queue.pop
      break if event == :shutdown

      handlers = @mutex.synchronize { @handlers[event].dup }
      handlers.each { |h| h.call(data) }
    end
  end
end

processor = AsyncEventProcessor.new
processor.subscribe(:user_created) { |u| puts "New user: #{u[:name]}" }
processor.subscribe(:user_created) { |u| puts "Send welcome to #{u[:email]}" }
processor.subscribe(:order_placed) { |o| puts "Order #{o[:id]} placed!" }

processor.publish(:user_created, { name: "Alice", email: "alice@test.com" })
processor.publish(:order_placed, { id: 1001, total: 250 })
processor.publish(:user_created, { name: "Bob", email: "bob@test.com" })

sleep(0.1)
processor.shutdown
```

**ข้อ 11-25:** (ย่อ)

```ruby
# ข้อ 11: Ractor parallel computation
if RUBY_VERSION >= "3.0"
  ractor_results = 4.times.map do |i|
    Ractor.new(i) do |worker_id|
      # CPU-intensive work
      sum = 0
      (worker_id * 250_000)...((worker_id + 1) * 250_000).each do |n|
        sum += n
      end
      sum
    end
  end
  total = ractor_results.sum(&:take)
  puts "Ractor total: #{total}"
end

# ข้อ 12-25 โดยย่อ
# ข้อ 12: Throttled concurrent requests
# ข้อ 13: Circuit breaker pattern
# ข้อ 14: Actor model simulation
# ข้อ 15: Concurrent pipeline
# ข้อ 16: Thread-safe LRU cache
# ข้อ 17: Fiber-based coroutine scheduler
# ข้อ 18: Parallel file processing
# ข้อ 19: Async HTTP batch requests
# ข้อ 20: Worker pool with priority
# ข้อ 21: Distributed lock simulation
# ข้อ 22: Rate limiter กับ threads
# ข้อ 23: Concurrent map-reduce
# ข้อ 24: Async logger
# ข้อ 25: Complete concurrent web server

# ข้อ 25 - Concurrent Web Server (simplified)
class TinyWebServer
  def initialize(port = 8080, threads: 4)
    @port    = port
    @threads = threads
    @pool    = Queue.new
    @router  = {}
  end

  def route(path, &handler)
    @router[path] = handler
  end

  def start
    # Start thread pool
    @workers = @threads.times.map do
      Thread.new do
        loop do
          connection = @pool.pop
          break if connection == :stop
          handle_connection(connection)
        end
      end
    end

    puts "Server running on port #{@port} with #{@threads} threads"
    # Normally would accept TCP connections here
    puts "Router paths: #{@router.keys.inspect}"
  end

  def stop
    @threads.times { @pool << :stop }
    @workers.each(&:join)
    puts "Server stopped"
  end

  private

  def handle_connection(conn)
    path, body = conn
    handler = @router[path]
    if handler
      response = handler.call(body)
      puts "#{path}: #{response}"
    else
      puts "#{path}: 404 Not Found"
    end
  end
end

server = TinyWebServer.new(8080, threads: 3)
server.route("/hello") { |_| "Hello, World!" }
server.route("/time")  { |_| Time.now.to_s }
server.route("/echo")  { |body| "Echo: #{body}" }

server.start

# Simulate requests
requests = [
  ["/hello", nil],
  ["/time", nil],
  ["/echo", "test message"],
  ["/unknown", nil]
]

requests.each { |req| server.instance_variable_get(:@pool) << req }
sleep(0.1)
server.stop
```

---

## สรุป

### เมื่อไหร่ใช้อะไร

| Scenario | วิธีที่แนะนำ |
|----------|------------|
| I/O-bound (HTTP, DB) | Threads |
| CPU-bound tasks | Ractors หรือ Processes |
| Coroutines/generators | Fibers |
| Simple async | concurrent-ruby |
| Bulk parallel | parallel gem |

### Thread Safety Checklist
1. **ใช้ Mutex** สำหรับ shared mutable state
2. **ระวัง race conditions** บน read-modify-write
3. **หลีกเลี่ยง deadlock** - lock ตามลำดับเดิมเสมอ
4. **Thread-local variables** สำหรับ per-request context
5. **ใช้ concurrent-ruby** แทนเขียน data structures เอง

### GVL Summary
- MRI Ruby มี GVL - threads ไม่ parallel สำหรับ Ruby code
- I/O operations release GVL - threads มีประโยชน์สำหรับ I/O
- Ruby 3.0+ Ractors - true parallelism
- JRuby/TruffleRuby - ไม่มี GVL

---

*ตอนถัดไป: ตอนที่ 26 - Functional Programming ใน Ruby*
