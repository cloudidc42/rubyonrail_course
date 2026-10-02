# ตอนที่ 25: Concurrency และ Threads (ขั้นตอนที่ 541-565)

Concurrency คือความสามารถของโปรแกรมในการดำเนินหลายงานพร้อมกัน Ruby มีหลายวิธีในการทำ Concurrency ตั้งแต่ Threads พื้นฐาน ไปจนถึง Fibers, Ractors และ Async Programming

---

## ขั้นตอนที่ 541: Thread Basics

### สร้างและใช้งาน Thread

```ruby
# Thread พื้นฐาน
thread = Thread.new do
  puts "Thread เริ่มทำงาน"
  sleep(1)
  puts "Thread เสร็จแล้ว"
end

puts "Main Thread ทำงานต่อไป"
thread.join  # รอให้ Thread เสร็จ
puts "ทุก Thread เสร็จแล้ว"

# สร้างหลาย Threads
threads = []

5.times do |i|
  threads << Thread.new(i) do |num|
    sleep(rand(0.1..0.5))
    puts "Thread #{num} เสร็จแล้ว"
  end
end

threads.each(&:join)
puts "ทุก Thread เสร็จแล้ว"
```

### Thread Values และ Return Values

```ruby
# Thread คืนค่าได้
result_thread = Thread.new do
  # ทำงานหนัก
  sleep(1)
  "ผลลัพธ์จาก Thread"
end

value = result_thread.value  # รอและรับค่า
puts value  # "ผลลัพธ์จาก Thread"

# ใช้ Parallel Calculation
def fetch_data(url)
  # Simulate HTTP request
  sleep(0.5)
  "Data from #{url}"
end

urls = [
  'https://api1.example.com/data',
  'https://api2.example.com/data',
  'https://api3.example.com/data'
]

start = Time.now

# Sequential (ช้า)
sequential_results = urls.map { |url| fetch_data(url) }
puts "Sequential: #{Time.now - start:.2f}s"

# Parallel with Threads (เร็วกว่า)
start = Time.now
parallel_threads = urls.map { |url| Thread.new { fetch_data(url) } }
parallel_results = parallel_threads.map(&:value)
puts "Parallel: #{Time.now - start:.2f}s"
```

### Thread States และ Management

```ruby
# สร้าง Thread และดู State
t = Thread.new { sleep(5) }

puts t.alive?   # true
puts t.status   # "sleep"

t.kill          # หยุด Thread
puts t.alive?   # false
puts t.status   # false (killed/completed)

# Thread Priority
low_priority  = Thread.new { puts "Low Priority" }
high_priority = Thread.new { puts "High Priority" }

low_priority.priority  = -1  # ลำดับต่ำ
high_priority.priority = 1   # ลำดับสูง

# Thread Local Variables
thread1 = Thread.new do
  Thread.current[:user_id] = 1
  sleep(0.1)
  puts "Thread 1 User: #{Thread.current[:user_id]}"
end

thread2 = Thread.new do
  Thread.current[:user_id] = 2
  puts "Thread 2 User: #{Thread.current[:user_id]}"
end

[thread1, thread2].each(&:join)

# Thread Group
group = ThreadGroup.new

5.times do |i|
  t = Thread.new { sleep(10) }
  group.add(t)
end

puts "Threads in group: #{group.list.size}"
group.list.each(&:kill)
```

---

## ขั้นตอนที่ 542: Mutex และ Synchronization

### Mutex พื้นฐาน

```ruby
# ปัญหา Race Condition
counter = 0

threads = 100.times.map do
  Thread.new do
    current = counter
    sleep(0.0001)  # Simulate some work
    counter = current + 1
  end
end

threads.each(&:join)
puts counter  # ไม่ใช่ 100! (Race Condition)

# แก้ด้วย Mutex
mutex = Mutex.new
counter = 0

threads = 100.times.map do
  Thread.new do
    mutex.synchronize do
      current = counter
      sleep(0.0001)
      counter = current + 1
    end
  end
end

threads.each(&:join)
puts counter  # 100 (ถูกต้อง)
```

### Mutex Methods

```ruby
mutex = Mutex.new

# synchronize - Lock และ Unlock อัตโนมัติ
mutex.synchronize do
  # Critical Section
  # Mutex จะ Lock ขณะอยู่ใน Block
end

# lock/unlock - Manual Control
mutex.lock
begin
  # Critical Section
ensure
  mutex.unlock
end

# try_lock - Non-blocking
if mutex.try_lock
  begin
    # Critical Section
  ensure
    mutex.unlock
  end
else
  puts "Mutex ถูก Lock อยู่ ข้ามไป"
end

# ตัวอย่างจริง: Thread-safe Counter
class ThreadSafeCounter
  def initialize
    @value = 0
    @mutex = Mutex.new
  end
  
  def increment
    @mutex.synchronize { @value += 1 }
  end
  
  def decrement
    @mutex.synchronize { @value -= 1 }
  end
  
  def value
    @mutex.synchronize { @value }
  end
  
  def reset!
    @mutex.synchronize { @value = 0 }
  end
end

counter = ThreadSafeCounter.new

threads = 1000.times.map do
  Thread.new { counter.increment }
end

threads.each(&:join)
puts counter.value  # 1000
```

### Deadlock

```ruby
# Deadlock ตัวอย่าง (อย่าใช้ใน Production!)
mutex_a = Mutex.new
mutex_b = Mutex.new

thread1 = Thread.new do
  mutex_a.lock
  sleep(0.01)
  mutex_b.lock  # รอ mutex_b ที่ thread2 lock อยู่
  puts "Thread 1 เสร็จ"
  mutex_b.unlock
  mutex_a.unlock
end

thread2 = Thread.new do
  mutex_b.lock
  sleep(0.01)
  mutex_a.lock  # รอ mutex_a ที่ thread1 lock อยู่ → DEADLOCK!
  puts "Thread 2 เสร็จ"
  mutex_a.unlock
  mutex_b.unlock
end

# ป้องกัน Deadlock: Lock ตามลำดับเสมอ
def safe_operation(resource_a, resource_b)
  # Lock ตามลำดับ ID เสมอ
  first, second = [resource_a, resource_b].sort_by(&:object_id)
  
  first.synchronize do
    second.synchronize do
      yield
    end
  end
end
```

### Monitor

```ruby
require 'monitor'

# Monitor คล้าย Mutex แต่ Reentrant (Lock ซ้ำได้)
class SafeQueue
  include MonitorMixin
  
  def initialize
    super
    @queue = []
    @condition = new_cond
  end
  
  def push(item)
    synchronize do
      @queue.push(item)
      @condition.signal
    end
  end
  
  def pop
    synchronize do
      @condition.wait_while { @queue.empty? }
      @queue.shift
    end
  end
  
  def size
    synchronize { @queue.size }
  end
end

queue = SafeQueue.new

producer = Thread.new do
  5.times do |i|
    queue.push("Item #{i}")
    sleep(0.1)
  end
end

consumer = Thread.new do
  5.times do
    item = queue.pop
    puts "Consumed: #{item}"
  end
end

[producer, consumer].each(&:join)
```

---

## ขั้นตอนที่ 543: Thread Safety

### Thread-safe Data Structures

```ruby
# Queue - Thread-safe Queue built-in
queue = Queue.new

producer = Thread.new do
  10.times do |i|
    queue.push(i)
    sleep(0.05)
  end
  queue.push(:done)
end

consumer = Thread.new do
  loop do
    item = queue.pop
    break if item == :done
    puts "Processing: #{item}"
  end
end

[producer, consumer].each(&:join)

# SizedQueue - Queue ที่มีขนาดจำกัด
sized_queue = SizedQueue.new(5)  # จำกัด 5 items

producers = 3.times.map do |i|
  Thread.new do
    5.times do |j|
      sized_queue.push("P#{i}-Item#{j}")
      puts "Produced: P#{i}-Item#{j}"
    end
  end
end

consumer = Thread.new do
  15.times do
    item = sized_queue.pop
    sleep(0.1)
    puts "Consumed: #{item}"
  end
end

producers.each(&:join)
consumer.join
```

### Thread-safe Patterns

```ruby
# Immutable Objects
class ImmutableConfig
  attr_reader :host, :port, :database
  
  def initialize(host:, port:, database:)
    @host     = host.freeze
    @port     = port
    @database = database.freeze
    freeze  # Prevent modification
  end
  
  def with(changes)
    ImmutableConfig.new(
      host:     changes.fetch(:host, @host),
      port:     changes.fetch(:port, @port),
      database: changes.fetch(:database, @database)
    )
  end
end

# Thread-safe Singleton
class ServiceLocator
  @instance = nil
  @mutex    = Mutex.new
  
  def self.instance
    @mutex.synchronize { @instance ||= new }
  end
  
  private_class_method :new
  
  def initialize
    @services = {}
    @lock     = Mutex.new
  end
  
  def register(name, service)
    @lock.synchronize { @services[name] = service }
  end
  
  def resolve(name)
    @lock.synchronize { @services[name] }
  end
end

# Thread-safe Lazy Initialization
class ExpensiveService
  def self.instance
    @instance ||= begin
      # Double-checked locking
      @mutex ||= Mutex.new
      @mutex.synchronize { @instance ||= new }
    end
  end
  
  private_class_method :new
  
  def initialize
    puts "กำลัง Initialize ExpensiveService..."
    sleep(0.5)  # Simulate expensive initialization
    @ready = true
  end
end
```

---

## ขั้นตอนที่ 544: Fiber และ Coroutines

### Fiber พื้นฐาน

```ruby
# Fiber - Lightweight Cooperative Concurrency
fiber = Fiber.new do
  puts "Fiber เริ่มทำงาน"
  Fiber.yield "ผลลัพธ์ที่ 1"
  
  puts "Fiber กลับมาทำงานต่อ"
  Fiber.yield "ผลลัพธ์ที่ 2"
  
  puts "Fiber เสร็จสิ้น"
  "ผลลัพธ์สุดท้าย"
end

puts fiber.resume   # "Fiber เริ่มทำงาน" -> "ผลลัพธ์ที่ 1"
puts fiber.resume   # "Fiber กลับมาทำงานต่อ" -> "ผลลัพธ์ที่ 2"
puts fiber.resume   # "Fiber เสร็จสิ้น" -> "ผลลัพธ์สุดท้าย"
puts fiber.alive?   # false

# Fiber สำหรับ Generator
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

# Infinite Sequence Generator
def range_generator(start, stop = nil, step = 1)
  Fiber.new do
    current = start
    loop do
      break if stop && current >= stop
      Fiber.yield current
      current += step
    end
  end
end

gen = range_generator(1, 10, 2)
while gen.alive?
  val = gen.resume
  print "#{val} " if val
end
# 1 3 5 7 9
```

### Fiber::Scheduler (Ruby 3.0+)

```ruby
# Ruby 3.0+ Fiber Scheduler สำหรับ Async I/O
class MyScheduler
  def initialize
    @readable = {}
    @writable = {}
    @timers   = []
    @count    = 0
  end
  
  def io_wait(io, events, timeout)
    # บอก Scheduler ว่า Fiber รอ I/O
    fiber = Fiber.current
    
    if events == IO::READABLE
      @readable[io] = fiber
    elsif events == IO::WRITABLE
      @writable[io] = fiber
    end
    
    Fiber.yield
    events
  end
  
  def run
    until @readable.empty? && @writable.empty? && @timers.empty?
      readable, writable, = IO.select(@readable.keys, @writable.keys, [], 0.1)
      
      readable&.each do |io|
        @readable.delete(io)&.resume
      end
      
      writable&.each do |io|
        @writable.delete(io)&.resume
      end
    end
  end
  
  def close
    # Cleanup
  end
  
  def kernel_sleep(duration)
    # Sleep implementation
    Fiber.yield
  end
  
  def fiber(&block)
    fiber = Fiber.new(&block)
    fiber.resume
    fiber
  end
end
```

---

## ขั้นตอนที่ 545: Ractors (Ruby 3.0+)

### Ractor พื้นฐาน

```ruby
# Ractor - True Parallelism ใน Ruby 3.0+
# ทำงานใน Separate Memory Space

# สร้าง Ractor
ractor = Ractor.new do
  puts "Ractor กำลังทำงาน"
  "ผลลัพธ์จาก Ractor"
end

result = ractor.take
puts result  # "ผลลัพธ์จาก Ractor"

# Ractor ที่รับ Input
ractor = Ractor.new do
  input = Ractor.receive  # รอรับ Message
  "ประมวลผล: #{input}"
end

ractor.send("Hello World")
puts ractor.take  # "ประมวลผล: Hello World"

# Parallel Computation ด้วย Ractors
def cpu_intensive_task(n)
  # คำนวณ Fibonacci (CPU Bound)
  return n if n <= 1
  cpu_intensive_task(n - 1) + cpu_intensive_task(n - 2)
end

numbers = [35, 36, 37, 38]

# Sequential
start = Time.now
sequential = numbers.map { |n| cpu_intensive_task(n) }
puts "Sequential: #{Time.now - start:.2f}s"

# Parallel with Ractors
start = Time.now
ractors = numbers.map do |n|
  Ractor.new(n) { |num| cpu_intensive_task(num) }
end
parallel = ractors.map(&:take)
puts "Parallel: #{Time.now - start:.2f}s"

puts sequential == parallel  # true
```

### Ractor Communication

```ruby
# Pipeline ด้วย Ractors
pipeline = [
  Ractor.new do
    loop do
      value = Ractor.receive
      Ractor.yield value * 2
    end
  end,
  
  Ractor.new do
    loop do
      value = Ractor.receive
      Ractor.yield value + 10
    end
  end
]

# Producer
producer = Ractor.new(*pipeline) do |stage1, stage2|
  [1, 2, 3, 4, 5].each do |n|
    stage1.send(n)
    result = stage1.take
    stage2.send(result)
    final = stage2.take
    Ractor.yield final
  end
end

5.times { puts producer.take }
# 12, 14, 16, 18, 20
```

---

## ขั้นตอนที่ 546: Async Programming

### concurrent-ruby Gem

```ruby
# Gemfile: gem 'concurrent-ruby'
require 'concurrent'

# Future - Async Computation
future = Concurrent::Future.new do
  sleep(1)
  "ผลลัพธ์หลังจาก 1 วินาที"
end

future.execute

# ทำงานอื่นระหว่างรอ
puts "ทำงานอื่น..."
sleep(0.5)
puts "ยังรออยู่..."

result = future.value  # รอจนเสร็จ
puts result

# Promise - Composable Async
def fetch_user(id)
  Concurrent::Promise.new {
    sleep(0.5)  # Simulate DB query
    { id: id, name: "User #{id}" }
  }.execute
end

def fetch_orders(user_id)
  Concurrent::Promise.new {
    sleep(0.3)  # Simulate DB query
    [{ id: 1, user_id: user_id }, { id: 2, user_id: user_id }]
  }.execute
end

# Chain Promises
fetch_user(1).then do |user|
  puts "Got user: #{user[:name]}"
  fetch_orders(user[:id])
end.then do |orders|
  puts "Got #{orders.size} orders"
end.rescue do |error|
  puts "Error: #{error.message}"
end.wait  # รอให้เสร็จ
```

### ThreadPool

```ruby
# Thread Pool สำหรับจัดการ Threads อย่างมีประสิทธิภาพ
require 'concurrent'

pool = Concurrent::ThreadPoolExecutor.new(
  min_threads: 2,
  max_threads: 10,
  max_queue: 100,
  fallback_policy: :abort
)

results = []
mutex = Mutex.new

20.times do |i|
  pool.post do
    sleep(rand(0.1..0.5))
    result = "Task #{i} เสร็จ"
    mutex.synchronize { results << result }
  end
end

pool.shutdown
pool.wait_for_termination(30)

puts "เสร็จทั้งหมด #{results.size} tasks"

# FixedThreadPool
fixed_pool = Concurrent::FixedThreadPool.new(4)

10.times do |i|
  fixed_pool.post { puts "Job #{i} on thread #{Thread.current.object_id}" }
end

fixed_pool.shutdown
fixed_pool.wait_for_termination

# CachedThreadPool
cached_pool = Concurrent::CachedThreadPool.new

5.times do |i|
  cached_pool.post { "Task #{i}" }
end
```

### Async/Await Pattern

```ruby
# Ruby ไม่มี async/await built-in แต่เลียนแบบได้

require 'concurrent'

class AsyncHTTPClient
  def self.get(url)
    Concurrent::Future.new do
      require 'open-uri'
      URI.open(url).read
    rescue => e
      raise "Failed to fetch #{url}: #{e.message}"
    end.execute
  end
end

# ใช้งาน
future1 = AsyncHTTPClient.get('https://jsonplaceholder.typicode.com/posts/1')
future2 = AsyncHTTPClient.get('https://jsonplaceholder.typicode.com/posts/2')

# รอทั้งสอง
result1 = future1.value!
result2 = future2.value!
```

---

## ขั้นตอนที่ 547: GIL (Global Interpreter Lock)

### ทำความเข้าใจ GIL ใน MRI Ruby

```ruby
# GIL (Global VM Lock ใน Ruby) ป้องกัน Thread หลายตัว
# รัน Ruby Code พร้อมกันบน MRI Ruby

# Thread ยังมีประโยชน์สำหรับ:
# 1. I/O Bound Operations (ไฟล์, เครือข่าย)
# 2. Sleep Operations

# I/O Bound - Thread ได้ประโยชน์
def read_file(path)
  File.read(path)  # GIL release ขณะรอ I/O
end

start = Time.now

# Sequential
['file1.txt', 'file2.txt', 'file3.txt'].each do |f|
  read_file(f) rescue nil
end
puts "Sequential: #{Time.now - start:.3f}s"

# Concurrent (เร็วกว่าสำหรับ I/O Bound)
start = Time.now
threads = ['file1.txt', 'file2.txt', 'file3.txt'].map do |f|
  Thread.new { read_file(f) rescue nil }
end
threads.each(&:join)
puts "Concurrent: #{Time.now - start:.3f}s"

# CPU Bound - Thread ไม่ได้ประโยชน์จาก GIL
def calculate(n)
  n.times.sum { |i| Math.sqrt(i) }
end

start = Time.now

# Sequential CPU
(1..4).each { calculate(1_000_000) }
puts "Sequential CPU: #{Time.now - start:.2f}s"

# Concurrent CPU (ไม่เร็วขึ้นเพราะ GIL)
start = Time.now
threads = 4.times.map { Thread.new { calculate(1_000_000) } }
threads.each(&:join)
puts "Concurrent CPU: #{Time.now - start:.2f}s"

# ใช้ Ractor หรือ Process.fork สำหรับ CPU Bound
```

### JRuby และ TruffleRuby

```ruby
# ใน JRuby หรือ TruffleRuby ไม่มี GIL
# Thread ทำงานแบบ True Parallel ได้

# ตรวจสอบ Ruby Implementation
puts RUBY_ENGINE          # "ruby", "jruby", "truffleruby"
puts RUBY_ENGINE_VERSION  # Version ของ Engine
puts RUBY_VERSION         # Ruby Version

if RUBY_ENGINE == 'ruby'
  puts "MRI Ruby - มี GIL"
elsif RUBY_ENGINE == 'jruby'
  puts "JRuby - ไม่มี GIL"
elsif RUBY_ENGINE == 'truffleruby'
  puts "TruffleRuby - ไม่มี GIL"
end
```

---

## ขั้นตอนที่ 548-550: Practical Concurrent Patterns

### Producer-Consumer Pattern

```ruby
require 'thread'

class ProducerConsumer
  def initialize(buffer_size = 10)
    @queue    = SizedQueue.new(buffer_size)
    @finished = false
  end
  
  def produce(items)
    Thread.new do
      items.each do |item|
        @queue.push(item)
        puts "Produced: #{item}"
      end
      @queue.push(nil)  # Sentinel value
    end
  end
  
  def consume
    Thread.new do
      loop do
        item = @queue.pop
        break if item.nil?
        
        process(item)
        puts "Consumed: #{item}"
      end
    end
  end
  
  def run(items, consumer_count = 3)
    consumers = consumer_count.times.map { consume }
    producer  = produce(items)
    
    producer.join
    consumers.each(&:join)
  end
  
  private
  
  def process(item)
    sleep(rand(0.1..0.3))  # Simulate processing
  end
end

pc = ProducerConsumer.new(5)
pc.run((1..20).to_a, 3)
```

### Worker Pool Pattern

```ruby
class WorkerPool
  Job = Struct.new(:task, :callback)
  
  def initialize(size)
    @jobs    = Queue.new
    @workers = size.times.map { create_worker }
  end
  
  def submit(task, &callback)
    @jobs.push(Job.new(task, callback))
  end
  
  def shutdown
    @workers.size.times { @jobs.push(:shutdown) }
    @workers.each(&:join)
  end
  
  def wait
    # ง่ายๆ: รอจนกว่า Queue ว่าง
    sleep(0.1) until @jobs.empty?
  end
  
  private
  
  def create_worker
    Thread.new do
      loop do
        job = @jobs.pop
        break if job == :shutdown
        
        begin
          result = job.task.call
          job.callback&.call(result)
        rescue => e
          puts "Worker Error: #{e.message}"
        end
      end
    end
  end
end

# การใช้งาน
pool = WorkerPool.new(4)
results = []
mutex = Mutex.new

20.times do |i|
  pool.submit(-> { sleep(0.1); i * 2 }) do |result|
    mutex.synchronize { results << result }
  end
end

pool.wait
pool.shutdown
puts results.sort.inspect
```

### Event-Driven Pattern

```ruby
class EventBus
  def initialize
    @handlers = Hash.new { |h, k| h[k] = [] }
    @mutex    = Mutex.new
  end
  
  def subscribe(event, &handler)
    @mutex.synchronize { @handlers[event] << handler }
  end
  
  def publish(event, data = nil)
    handlers = @mutex.synchronize { @handlers[event].dup }
    
    threads = handlers.map do |handler|
      Thread.new { handler.call(data) }
    end
    
    threads.each(&:join)
  end
end

bus = EventBus.new

# Subscribe
bus.subscribe(:user_created) do |user|
  puts "Sending welcome email to #{user[:email]}"
  sleep(0.1)
end

bus.subscribe(:user_created) do |user|
  puts "Creating profile for #{user[:name]}"
  sleep(0.1)
end

bus.subscribe(:user_created) do |user|
  puts "Logging user creation: #{user[:id]}"
end

# Publish
user = { id: 1, name: 'สมชาย', email: 'somchai@example.com' }
bus.publish(:user_created, user)
```

### Rate Limiter

```ruby
class TokenBucket
  def initialize(capacity, refill_rate)
    @capacity    = capacity.to_f
    @tokens      = capacity.to_f
    @refill_rate = refill_rate.to_f  # tokens per second
    @last_refill = Time.now
    @mutex       = Mutex.new
  end
  
  def consume(tokens = 1)
    @mutex.synchronize do
      refill!
      
      if @tokens >= tokens
        @tokens -= tokens
        true
      else
        false
      end
    end
  end
  
  def wait_and_consume(tokens = 1, timeout: nil)
    deadline = timeout ? Time.now + timeout : nil
    
    loop do
      return true if consume(tokens)
      
      raise Timeout::Error if deadline && Time.now > deadline
      
      sleep(0.01)
    end
  end
  
  private
  
  def refill!
    now     = Time.now
    elapsed = now - @last_refill
    
    @tokens = [@tokens + elapsed * @refill_rate, @capacity].min
    @last_refill = now
  end
end

# Rate Limiter: 10 requests per second
limiter = TokenBucket.new(10, 10)

threads = 20.times.map do |i|
  Thread.new do
    if limiter.wait_and_consume(1, timeout: 5)
      puts "Request #{i}: OK"
    else
      puts "Request #{i}: Rate Limited"
    end
  end
end

threads.each(&:join)
```

---

## ขั้นตอนที่ 551: Parallel Gem

```ruby
# Gemfile: gem 'parallel'
require 'parallel'

items = (1..20).to_a

# Map แบบ Parallel
results = Parallel.map(items, in_threads: 4) do |item|
  sleep(0.1)  # Simulate I/O work
  item * 2
end
puts results.inspect

# Each แบบ Parallel
Parallel.each(items, in_processes: 4) do |item|
  # CPU Bound work - ใช้ processes
  puts "Processing #{item}"
end

# กำหนด Progress
Parallel.map(items, progress: "Processing") do |item|
  sleep(0.1)
  item
end

# ควบคุม Worker Count
result = Parallel.map(
  (1..100).to_a,
  in_threads: Parallel.processor_count
) do |n|
  n ** 2
end
```

---

## ขั้นตอนที่ 552-555: Async I/O

### async Gem (Ruby Async)

```ruby
# Gemfile: gem 'async'
require 'async'
require 'async/http/internet'

# Async HTTP Requests
Async do
  internet = Async::HTTP::Internet.new
  
  tasks = [
    'https://jsonplaceholder.typicode.com/posts/1',
    'https://jsonplaceholder.typicode.com/posts/2',
    'https://jsonplaceholder.typicode.com/posts/3'
  ].map do |url|
    Async do
      response = internet.get(url)
      JSON.parse(response.read)
    end
  end
  
  results = tasks.map(&:wait)
  results.each { |r| puts r['title'] }
ensure
  internet&.close
end
```

### Concurrent File Operations

```ruby
require 'concurrent'

class AsyncFileProcessor
  def initialize(thread_count = 4)
    @pool = Concurrent::FixedThreadPool.new(thread_count)
  end
  
  def process_files(file_paths)
    promises = file_paths.map do |path|
      Concurrent::Promise.new(executor: @pool) do
        process_file(path)
      end.execute
    end
    
    promises.map do |promise|
      promise.value || promise.reason
    end
  end
  
  def shutdown
    @pool.shutdown
    @pool.wait_for_termination
  end
  
  private
  
  def process_file(path)
    content = File.read(path)
    {
      path:       path,
      size:       File.size(path),
      lines:      content.lines.count,
      words:      content.split.count,
      processed: true
    }
  rescue Errno::ENOENT => e
    { path: path, error: e.message, processed: false }
  end
end

processor = AsyncFileProcessor.new(8)
results = processor.process_files(Dir.glob('**/*.rb'))

results.each do |result|
  if result[:processed]
    puts "#{result[:path]}: #{result[:lines]} lines"
  else
    puts "Error: #{result[:error]}"
  end
end

processor.shutdown
```

---

## แบบฝึกหัดบทที่ 25 (25 ข้อ)

### ระดับพื้นฐาน (ข้อ 1-8)

**ข้อ 1:** สร้าง Thread Pool ขนาด 5 threads สำหรับประมวลผล Array ของตัวเลข 100 ตัว

**ข้อ 2:** Implement Thread-safe Stack ด้วย Mutex

**ข้อ 3:** สร้าง Producer-Consumer ด้วย Queue ที่มี 1 Producer และ 3 Consumers

**ข้อ 4:** ใช้ Fiber สร้าง Infinite Sequence Generator สำหรับ Prime Numbers

**ข้อ 5:** สร้าง Countdown Timer ที่ใช้ Thread และหยุดได้

**ข้อ 6:** ทดสอบ Race Condition และแก้ด้วย Mutex

**ข้อ 7:** สร้าง Parallel File Reader ที่อ่านไฟล์หลายไฟล์พร้อมกัน

**ข้อ 8:** ใช้ SizedQueue Implement Bounded Buffer

### ระดับกลาง (ข้อ 9-17)

**ข้อ 9:** Implement Rate Limiter ด้วย Token Bucket Algorithm และ Thread Safety

**ข้อ 10:** สร้าง Async HTTP Client ที่ Request หลาย URLs พร้อมกัน

**ข้อ 11:** Implement Event Bus ที่ Thread-safe

**ข้อ 12:** สร้าง Thread-safe Cache ด้วย TTL (Time-To-Live)

**ข้อ 13:** ใช้ concurrent-ruby Future สำหรับ Parallel Data Processing

**ข้อ 14:** สร้าง Worker Pool ที่มี Error Handling และ Retry Logic

**ข้อ 15:** Implement Semaphore สำหรับ Limit Concurrent Access

**ข้อ 16:** สร้าง Pipeline Pattern ด้วย Threads และ Queues

**ข้อ 17:** ทดสอบ Deadlock และ Implement วิธีป้องกัน

### ระดับสูง (ข้อ 18-25)

**ข้อ 18:** Implement Reactor Pattern ด้วย Fiber Scheduler

**ข้อ 19:** สร้าง Circuit Breaker ที่ Thread-safe

**ข้อ 20:** Implement Map-Reduce Algorithm ด้วย Threads

**ข้อ 21:** สร้าง Concurrent Web Scraper ด้วย Thread Pool

**ข้อ 22:** ใช้ Ractor สำหรับ CPU-Bound Parallel Computation

**ข้อ 23:** Implement Actor Model Pattern

**ข้อ 24:** สร้าง Backpressure Mechanism สำหรับ Stream Processing

**ข้อ 25:** สร้าง Distributed Task Queue โดยใช้ Redis และ Threads

---

### เฉลยตัวอย่าง ข้อ 12: Thread-safe Cache พร้อม TTL

```ruby
class TTLCache
  Entry = Struct.new(:value, :expires_at)
  
  def initialize(default_ttl: 60)
    @store       = {}
    @default_ttl = default_ttl
    @mutex       = Mutex.new
    start_cleanup_thread
  end
  
  def set(key, value, ttl: nil)
    ttl      ||= @default_ttl
    expires_at = Time.now + ttl
    
    @mutex.synchronize do
      @store[key] = Entry.new(value, expires_at)
    end
    
    value
  end
  
  def get(key)
    @mutex.synchronize do
      entry = @store[key]
      return nil unless entry
      return nil if entry.expires_at < Time.now
      
      entry.value
    end
  end
  
  def delete(key)
    @mutex.synchronize { @store.delete(key) }
  end
  
  def fetch(key, ttl: nil, &block)
    value = get(key)
    return value if value
    
    value = block.call
    set(key, value, ttl: ttl)
    value
  end
  
  def clear
    @mutex.synchronize { @store.clear }
  end
  
  def size
    @mutex.synchronize { @store.count { |_, v| v.expires_at > Time.now } }
  end
  
  def keys
    @mutex.synchronize do
      @store.select { |_, v| v.expires_at > Time.now }.keys
    end
  end
  
  private
  
  def start_cleanup_thread
    Thread.new do
      loop do
        sleep(30)  # Cleanup ทุก 30 วินาที
        cleanup_expired_entries
      end
    end.tap { |t| t.abort_on_exception = false }
  end
  
  def cleanup_expired_entries
    now = Time.now
    @mutex.synchronize do
      @store.delete_if { |_, entry| entry.expires_at < now }
    end
  end
end

# การใช้งาน
cache = TTLCache.new(default_ttl: 5)

# Set values
cache.set('key1', 'value1')
cache.set('key2', 'value2', ttl: 10)

# Get values
puts cache.get('key1')  # "value1"
puts cache.size         # 2

# Fetch with block (สร้างถ้าไม่มี)
result = cache.fetch('expensive_key') do
  sleep(1)  # Expensive computation
  "computed_value"
end
puts result  # "computed_value"

# Second call ใช้ Cache
result2 = cache.fetch('expensive_key') { "won't be called" }
puts result2  # "computed_value" (from cache)

# Thread-safety test
threads = 10.times.map do |i|
  Thread.new do
    value = cache.fetch("shared_key_#{i % 3}") { "value_#{i}" }
    puts "Thread #{i}: #{value}"
  end
end

threads.each(&:join)

# รอ TTL หมด
sleep(6)
puts cache.get('key1')  # nil (expired)
puts cache.size         # 1 (key2 ยังอยู่)
```

---

*สรุปบทที่ 25: Concurrency ใน Ruby มีหลายระดับตั้งแต่ Threads พื้นฐาน ไปจนถึง Fibers, Ractors และ Async Programming การเลือกใช้ขึ้นอยู่กับประเภทงาน (I/O Bound vs CPU Bound) และความต้องการด้าน Parallelism การใช้ Mutex และ Thread-safe Data Structures เป็นสิ่งจำเป็นเมื่อทำงานกับ Shared State*
