# Part 87: Ruby Internals เชิงลึก

## บทนำ

การเข้าใจ Ruby internals ช่วยให้เขียนโค้ดได้มีประสิทธิภาพมากขึ้น เข้าใจ behavior ที่คาดไม่ถึง และ debug ปัญหาที่ซับซ้อนได้ ในบทนี้เราจะเจาะลึกกลไกภายใน Ruby

---

## 1. Ruby Object Model เชิงลึก

### 1.1 ทุกอย่างเป็น Object

```ruby
# ใน Ruby ทุกอย่างเป็น Object รวมถึง numbers, nil, true, false
1.class          # => Integer
nil.class        # => NilClass
true.class       # => TrueClass
false.class      # => FalseClass
"hello".class    # => String
:symbol.class    # => Symbol
[].class         # => Array
{}.class         # => Hash

# แม้แต่ Class เองก็เป็น Object
String.class     # => Class
Class.class      # => Class  (Class เป็น instance ของ Class เอง!)
Module.class     # => Class
Object.class     # => Class

# และ Method ก็เป็น Object
method = 1.method(:+)
method.class     # => Method
method.call(2)   # => 3

# Proc และ Lambda
proc_obj = proc { |x| x * 2 }
proc_obj.class   # => Proc
```

### 1.2 Object Identity และ Value Equality

```ruby
# object_id คือ unique identifier ของแต่ละ object
a = "hello"
b = "hello"

a.object_id == b.object_id  # => false (คนละ object)
a == b                       # => true (value เหมือนกัน)
a.equal?(b)                  # => false (ไม่ใช่ object เดียวกัน)

# Special objects มี fixed object_id
nil.object_id    # => 8
true.object_id   # => 2
false.object_id  # => 0
1.object_id      # => 3 (small integers: n * 2 + 1)
2.object_id      # => 5

# Symbols ใน Ruby < 3.0 - same symbol = same object
:foo.object_id == :foo.object_id  # => true
"foo".to_sym.object_id == :foo.object_id  # => true (ใน most cases)

# Frozen strings share object
a = "hello".freeze
b = "hello".freeze
# ไม่จำเป็นต้อง share object id แต่ runtime optimization อาจทำ
```

### 1.3 Ruby Object Structure (C-level)

```ruby
# Ruby object ใน C มี structure นี้ (simplified):
# struct RObject {
#   struct RBasic basic;  // flags, klass
#   union {
#     struct {
#       uint32_t numiv;   // number of instance variables
#       VALUE *ivptr;     // pointer to instance variable array
#       void *iv_index_tbl; // lookup table
#     };
#     VALUE ary[3];       // small objects optimization
#   };
# };

# ดู internals ด้วย ObjectSpace
ObjectSpace.each_object(String).count  # นับ String objects ทั้งหมด

# ดู memory usage
require 'objspace'
str = "hello world"
ObjectSpace.memsize_of(str)  # Memory ที่ใช้ (bytes)
ObjectSpace.memsize_of_all(String)  # Total memory ของ String objects

# Allocation tracing
ObjectSpace.trace_object_allocations_start

user = User.new(name: 'John')

puts ObjectSpace.allocation_sourcefile(user)  # ไฟล์ที่สร้าง object
puts ObjectSpace.allocation_sourceline(user)  # บรรทัดที่สร้าง object
puts ObjectSpace.allocation_class_path(user)  # Class ที่สร้าง

ObjectSpace.trace_object_allocations_stop
```

### 1.4 Eigenclass / Singleton Class

```ruby
# ทุก object มี eigenclass (singleton class) ของตัวเอง
class Dog
  def speak
    "Woof!"
  end
end

rex = Dog.new
spot = Dog.new

# สร้าง method เฉพาะสำหรับ rex เท่านั้น
def rex.fetch
  "I fetched it!"
end

rex.fetch   # => "I fetched it!"
# spot.fetch  # => NoMethodError

# เข้าถึง singleton class
rex_singleton = class << rex
  self  # ใน block นี้ self คือ singleton class ของ rex
end

rex_singleton  # => #<Class:#<Dog:0x...>>
rex.singleton_class  # => #<Class:#<Dog:0x...>> (Ruby 1.9.2+)

# Class methods คือ instance methods ของ singleton class ของ Class
class Cat
  def self.species  # method นี้อยู่ใน Cat's singleton class
    "Felis catus"
  end
end

Cat.singleton_class.instance_methods(false)  # => [:species]
```

---

## 2. Method Lookup Chain เชิงลึก

### 2.1 Method Resolution Order (MRO)

```ruby
# Ruby ค้นหา method ตาม path นี้:
# 1. Singleton class (ถ้ามี)
# 2. Class ของ object
# 3. Mixins (Modules) ที่ include ล่าสุดก่อน
# 4. Superclass
# 5. ทำขั้นตอน 2-4 ซ้ำจนถึง BasicObject

module Flyable
  def fly
    "I can fly! (from Flyable)"
  end
  
  def describe
    "I'm Flyable"
  end
end

module Swimmable
  def swim
    "I can swim! (from Swimmable)"
  end
  
  def describe
    "I'm Swimmable"
  end
end

class Animal
  def breathe
    "Breathing (from Animal)"
  end
  
  def describe
    "I'm an Animal"
  end
end

class Duck < Animal
  include Swimmable
  include Flyable  # include ล่าสุด จะถูกค้นหาก่อน
  
  def describe
    "I'm a Duck"
  end
end

duck = Duck.new

# Method lookup chain:
puts Duck.ancestors.inspect
# => [Duck, Flyable, Swimmable, Animal, Object, Kernel, BasicObject]

# Duck#describe จะถูกเรียกก่อน (ใกล้ที่สุด)
duck.describe    # => "I'm a Duck"

# ดู ancestors chain
Duck.ancestors.each do |ancestor|
  puts "#{ancestor}: #{ancestor.instance_methods(false).include?(:describe)}"
end
# Duck: true      <- ค้นพบที่นี่ หยุด
# Flyable: true
# Swimmable: true
# Animal: true
# ...
```

### 2.2 super การค้นหาใน Chain

```ruby
module Logging
  def save
    puts "Logging: before save"
    result = super  # เรียก next in chain
    puts "Logging: after save"
    result
  end
end

module Validating
  def save
    puts "Validating: checking..."
    raise "Invalid!" unless valid?
    super
  end
  
  def valid?
    true
  end
end

class Record
  include Validating
  include Logging
  
  def save
    puts "Record: saving to database"
    true
  end
end

record = Record.new
record.save
# Output:
# Logging: before save
# Validating: checking...
# Record: saving to database
# Logging: after save

# super ส่ง args ทั้งหมดไปด้วย
class MathHelper
  def add(a, b)
    result = super  # ส่ง a, b ไปให้ super
    result
  end
end

# super() เรียก super โดยไม่ส่ง args
# super เรียก super พร้อมส่ง args ทั้งหมด
# super(a) เรียก super พร้อมส่งแค่ a
```

### 2.3 Method Visibility

```ruby
class BankAccount
  def balance
    @balance || 0
  end
  
  def deposit(amount)
    validate_amount!(amount)
    @balance = balance + amount
  end
  
  def transfer_to(other_account, amount)
    validate_amount!(amount)
    # private method เรียกได้ภายใน class
    debit(amount)
    other_account.credit(amount)  # protected method เรียกจาก same class/subclass
  end
  
  protected
  
  def credit(amount)
    @balance = balance + amount
  end
  
  private
  
  def debit(amount)
    raise InsufficientFundsError if amount > balance
    @balance = balance - amount
  end
  
  def validate_amount!(amount)
    raise ArgumentError, "Amount must be positive" unless amount > 0
  end
end

account = BankAccount.new
account.deposit(1000)
# account.debit(100)    # => NoMethodError (private)
# account.credit(100)   # => NoMethodError from outside class (protected)

# Ruby 2.7+: private method ด้วย explicit receiver
class Foo
  def bar
    self.baz  # Error ก่อน Ruby 2.7 สำหรับ private method
               # แต่ Ruby 2.7+ อนุญาตสำหรับ self explicitly
  end
  
  private
  
  def baz
    "baz"
  end
end
```

### 2.4 BasicObject และ Object

```ruby
# BasicObject คือ root ของ hierarchy
# มี methods น้อยมาก (เหมาะสำหรับ proxy objects)
BasicObject.instance_methods.sort
# => [:!, :!=, :==, :__id__, :__send__, :equal?, :instance_eval, :instance_exec]

# Object มี methods เพิ่มจาก Kernel module
Object.include?(Kernel)  # => true
Object.ancestors  # => [Object, Kernel, BasicObject]

# Blank Slate / Proxy pattern ด้วย BasicObject
class BlankSlate < BasicObject
  def initialize(target)
    @target = target
  end
  
  def method_missing(name, *args, &block)
    @target.send(name, *args, &block)
  end
  
  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private)
  end
end

proxy = BlankSlate.new("hello world")
proxy.upcase    # => "HELLO WORLD"
proxy.length    # => 11
proxy.respond_to?(:upcase)  # => true
```

---

## 3. Ruby GC (Garbage Collector) ทำงานอย่างไร

### 3.1 Tri-color Mark and Sweep

```ruby
# Ruby ใช้ Incremental Generational GC (ตั้งแต่ Ruby 2.2+)
# Tri-color marking:
# - White: ยังไม่ถูก mark (อาจถูก collect)
# - Gray: ถูก mark แต่ children ยังไม่ถูก mark
# - Black: ถูก mark และ children ก็ถูก mark แล้ว

# Generational GC:
# - Young generation (eden): objects ใหม่
# - Old generation: objects ที่รอดมาหลาย GC cycles

# ดู GC statistics
GC.stat
# => {
#   :count=>12,                 # จำนวนครั้งที่ run GC
#   :heap_allocated_pages=>100,  # จำนวน heap pages ที่ allocated
#   :heap_sorted_length=>100,
#   :heap_allocatable_pages=>0,
#   :heap_available_slots=>40800,
#   :heap_live_slots=>35000,     # objects ที่ active
#   :heap_free_slots=>5800,      # slots ที่ว่าง
#   :heap_final_slots=>0,
#   :heap_marked_slots=>25000,
#   :heap_eden_pages=>100,
#   :heap_tomb_pages=>0,
#   :total_allocated_pages=>100,
#   :total_freed_pages=>0,
#   :total_allocated_objects=>500000,
#   :total_freed_objects=>465000,
#   :malloc_increase_bytes=>2000000,
#   :malloc_increase_bytes_limit=>16000000,
#   :minor_gc_count=>10,         # Young generation GC
#   :major_gc_count=>2,          # Full GC
#   ...
# }

# Force GC และ measure
before = GC.stat[:total_freed_objects]
GC.start
after = GC.stat[:total_freed_objects]
puts "Freed #{after - before} objects"
```

### 3.2 Object Lifecycle และ Finalizers

```ruby
# Finalizer ทำงานเมื่อ object ถูก GC
class ResourceHolder
  def initialize(resource)
    @resource = resource
    @id = object_id
    
    # Finalizer ต้องเป็น Proc ที่ไม่ reference object ตัวเอง
    ObjectSpace.define_finalizer(self, self.class.create_finalizer(@id))
  end
  
  def self.create_finalizer(id)
    proc do
      # Cleanup ที่นี่
      puts "Object #{id} is being garbage collected"
      # Close file handles, database connections, etc.
    end
  end
end

holder = ResourceHolder.new("some_resource")
holder = nil  # Remove reference
GC.start     # Force GC - finalizer จะถูกเรียก
```

### 3.3 Weak References

```ruby
# WeakRef อนุญาตให้ reference object โดยไม่ป้องกัน GC
require 'weakref'

class Cache
  def initialize
    @store = {}
  end
  
  def set(key, value)
    @store[key] = WeakRef.new(value)
  end
  
  def get(key)
    ref = @store[key]
    return nil unless ref
    
    begin
      ref.__getobj__  # อาจ raise WeakRef::RefError ถ้าถูก GC แล้ว
    rescue WeakRef::RefError
      @store.delete(key)
      nil
    end
  end
end

cache = Cache.new
obj = { data: "important" }
cache.set(:data, obj)

cache.get(:data)  # => { data: "important" }

obj = nil  # Remove strong reference
GC.start

cache.get(:data)  # => nil (object ถูก GC แล้ว)
```

### 3.4 GC Tuning

```ruby
# Environment variables สำหรับ tune GC:
# RUBY_GC_HEAP_INIT_SLOTS      - initial slots
# RUBY_GC_HEAP_FREE_SLOTS      - minimum free slots หลัง GC
# RUBY_GC_HEAP_GROWTH_FACTOR   - growth factor เมื่อ heap ขยาย
# RUBY_GC_HEAP_GROWTH_MAX_SLOTS - maximum slots to grow
# RUBY_GC_MALLOC_LIMIT         - malloc limit ก่อน minor GC

# ตัวอย่างการ tune สำหรับ web server (ลด GC pauses):
# RUBY_GC_HEAP_INIT_SLOTS=600000
# RUBY_GC_HEAP_FREE_SLOTS=600000
# RUBY_GC_HEAP_GROWTH_FACTOR=1.25
# RUBY_GC_MALLOC_LIMIT=67108864

# Measure GC impact
require 'gc_instrumentation'

class GCTracker
  def self.measure
    before_stats = GC.stat.dup
    before_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    
    yield
    
    after_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    after_stats = GC.stat
    
    {
      elapsed_ms: ((after_time - before_time) * 1000).round(2),
      gc_runs: after_stats[:count] - before_stats[:count],
      objects_freed: after_stats[:total_freed_objects] - before_stats[:total_freed_objects]
    }
  end
end

stats = GCTracker.measure do
  # Code to measure
  10_000.times { String.new("temporary string #{rand}") }
end

puts stats
# => {:elapsed_ms=>45.23, :gc_runs=>2, :objects_freed=>9850}
```

---

## 4. Ruby VM (YARV) Overview

### 4.1 YARV Architecture

```ruby
# YARV = Yet Another Ruby VM
# คือ bytecode interpreter ที่ถูกใช้ใน Ruby 1.9+

# ดู bytecode ที่ Ruby generate
puts RubyVM::InstructionSequence.compile('1 + 2').disasm
# == disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(1,5)>
# 0000 putobject_INT2FIX 1                                              (   1)[Li]
# 0002 putobject_INT2FIX 2
# 0004 opt_plus                        <calldata!mid:+, argc:1, ARGS_SIMPLE>[CcCr]
# 0006 leave

# ดู bytecode ของ method
def greet(name)
  "Hello, #{name}!"
end

iseq = RubyVM::InstructionSequence.of(method(:greet))
puts iseq.disasm
```

### 4.2 Bytecode Instructions

```ruby
# ตัวอย่าง bytecode instructions ที่สำคัญ:

# putself - push self onto stack
# putnil - push nil
# putobject - push literal object
# setlocal/getlocal - set/get local variable
# setinstancevariable/getinstancevariable - instance variables
# opt_send_without_block - optimized method call
# leave - return from method

# Method call mechanics
code = <<~RUBY
  def add(a, b)
    a + b
  end
  
  result = add(1, 2)
RUBY

iseq = RubyVM::InstructionSequence.compile(code)
puts iseq.disasm
# แสดง bytecode ทั้งหมดรวมถึง method definitions
```

### 4.3 Compilation Process

```ruby
# Ruby source -> Tokens -> AST -> Bytecode -> Execution
# 1. Lexer: Source code -> Tokens
# 2. Parser: Tokens -> Abstract Syntax Tree (AST)
# 3. Compiler: AST -> YARV Bytecode
# 4. VM: Execute bytecode

# ดู AST (ใช้ ruby_parser หรือ ripper)
require 'ripper'
require 'pp'

code = "def hello(name)\n  \"Hello, #{name}!\"\nend"
pp Ripper.sexp(code)
# [:program,
#  [[:def,
#    [:@ident, "hello", [1, 4]],
#    [:paren, [:params, [[:@ident, "name", [1, 10]]], nil, nil, nil, nil, nil, nil]],
#    [:bodystmt,
#     [[:string_literal,
#       ...

# Inline caching - YARV cache method lookups
# Ruby caches the result of method lookups สำหรับ performance
# ทุก call site มี inline cache ที่เก็บ:
# - Class ที่ call ครั้งล่าสุด
# - Method ที่ resolve แล้ว

# JIT Compilation (Ruby 3.1+ YJIT)
# YJIT แปลง hot bytecode เป็น native machine code
# ดู YJIT stats:
RubyVM::YJIT.runtime_stats if defined?(RubyVM::YJIT)
```

### 4.4 Fiber และ Coroutines

```ruby
# Fiber คือ lightweight concurrency primitive ใน Ruby
# สร้างโดยใช้ setjmp/longjmp หรือ platform coroutines

# Fiber ทำงานอย่างไร:
# - Each fiber มี separate stack (default 4KB)
# - Fiber.yield หยุด execution และคืน control
# - Fiber#resume เริ่ม/ต่อ execution

fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield "yielded from step 1"
  puts "Step 2"
  "final result"
end

result1 = fiber.resume  # "Step 1" ถูก print, คืน "yielded from step 1"
puts result1  # => "yielded from step 1"

result2 = fiber.resume  # "Step 2" ถูก print, คืน "final result"
puts result2  # => "final result"

# Enumerator ใช้ Fiber ภายใน
enum = Enumerator.new do |yielder|
  yielder << 1
  yielder << 2
  yielder << 3
end

enum.next  # => 1
enum.next  # => 2
enum.next  # => 3

# Fiber Scheduler (Ruby 3.0+) สำหรับ non-blocking I/O
class MyScheduler
  def io_wait(io, events, timeout)
    # Custom I/O waiting logic
    IO.select([io], nil, nil, timeout)
  end
  
  def kernel_sleep(duration = nil)
    sleep duration
  end
  
  def block(blocker, timeout = nil)
    # Custom blocking logic
  end
end
```

---

## 5. Object Shapes (Ruby 3.2+)

### 5.1 ความเข้าใจ Object Shapes

```ruby
# Object Shape คือ metadata ที่บอก layout ของ instance variables
# Objects ที่มี same shape จะ share struct definition
# ช่วยให้ inline caching ทำงานได้ดีขึ้น

# ก่อน Ruby 3.2:
# ทุก object ต้อง lookup iv_index_tbl เพื่อหา instance variable
# แต่ละ object มี hash map ของ instance variable indices

# Ruby 3.2+ ด้วย Object Shapes:
# Objects ที่มี same shape share shape ID
# JIT/inline cache ใช้ shape ID สำหรับ fast path

class Point
  def initialize(x, y)
    @x = x  # Shape 1: Point with @x
    @y = y  # Shape 2: Point with @x, @y
  end
  
  def x; @x; end
  def y; @y; end
end

# p1 และ p2 มี same shape -> สามารถ share optimized code ได้
p1 = Point.new(1, 2)
p2 = Point.new(3, 4)

# ปัญหาของ Dynamic instance variables (anti-pattern):
class BadClass
  def setup_a
    @a = 1
    @b = 2
  end
  
  def setup_b
    @b = 2
    @a = 1  # ลำดับต่างกัน -> different shape!
  end
end

# Best practice: ตั้งค่า instance variables ตาม order เดิมเสมอ
class GoodClass
  def initialize
    @a = nil
    @b = nil
    @c = nil
    # กำหนดทุก iv ใน initialize เพื่อ consistent shape
  end
end
```

### 5.2 Shape-based Optimization

```ruby
# ดู shape ของ object (Ruby 3.2+)
if RUBY_VERSION >= '3.2'
  require 'ruby/vm'
  
  obj = Object.new
  obj.instance_variable_set(:@x, 1)
  
  # Shape ID (internal, may change between versions)
  # สามารถ introspect ผ่าน ObjectSpace
end

# Benchmark: consistent vs inconsistent shapes
require 'benchmark'

class ConsistentShape
  def initialize(n)
    @a = n
    @b = n * 2
    @c = n * 3
  end
  
  def sum
    @a + @b + @c
  end
end

class InconsistentShape
  def initialize(n)
    # บางครั้งสร้าง @d บางครั้งไม่สร้าง
    if n.odd?
      @a = n
      @b = n * 2
      @c = n * 3
    else
      @a = n
      @c = n * 3
      @b = n * 2
      @d = n * 4  # เพิ่ม @d ทำให้ shape ต่างกัน
    end
  end
  
  def sum
    @a + @b + @c
  end
end

N = 100_000

Benchmark.bm(20) do |x|
  x.report('Consistent:') do
    N.times { |i| ConsistentShape.new(i).sum }
  end
  
  x.report('Inconsistent:') do
    N.times { |i| InconsistentShape.new(i).sum }
  end
end
```

---

## 6. Frozen String Literal Optimization

### 6.1 String Freezing

```ruby
# String ใน Ruby เป็น mutable โดย default
str = "hello"
str << " world"  # OK

frozen_str = "hello".freeze
# frozen_str << " world"  # => FrozenError

# frozen_string_literal comment
# เมื่อเพิ่ม # frozen_string_literal: true ที่หัวไฟล์
# String literals ทั้งหมดในไฟล์นั้นจะถูก frozen อัตโนมัติ

# # frozen_string_literal: true
# 
# str = "hello"  # frozen
# str << " world"  # => FrozenError
# str = +"hello"  # Mutable string (+ prefix)
# str << " world"  # OK
```

### 6.2 ประโยชน์ของ Frozen Strings

```ruby
# 1. Memory optimization: same frozen string สามารถ share memory ได้
# 2. Performance: no copy-on-write overhead
# 3. Thread safety: frozen objects ปลอดภัยสำหรับ multi-thread

# ตัวอย่าง memory usage
require 'objspace'

# Non-frozen strings
100.times { "hello world" }  # สร้าง 100 objects แยกกัน

# Frozen strings
FROZEN = "hello world".freeze
100.times { FROZEN }  # ใช้ object เดียวกัน

# Symbol vs frozen string
# Ruby 3.0+ frozen_string_literal ช่วยลด garbage

# String deduplication
# Ruby 2.3+: String#-@ (ทำให้ frozen)
str = -"hello"  # Equivalent to "hello".freeze
str.frozen?  # => true

# String literals ที่ identical ใน source จะ deduplicate อัตโนมัติ (Ruby 2.5+)
a = -"test"
b = -"test"
a.equal?(b)  # => true (same object!) ใน most implementations
```

### 6.3 Benchmark: Frozen vs Mutable Strings

```ruby
require 'benchmark'

N = 1_000_000

Benchmark.bm(25) do |x|
  x.report('Regular string:') do
    N.times { "hello world".length }
  end
  
  x.report('Frozen constant:') do
    frozen = "hello world".freeze
    N.times { frozen.length }
  end
  
  x.report('Symbol:') do
    N.times { :hello_world.to_s.length }
  end
end

# การเปรียบเทียบ string operations
x.report('String concat (<<):') do
  result = String.new  # Mutable
  N.times { result << "x" }
end

x.report('String concat (+):') do
  result = ""
  N.times { result = result + "x" }  # สร้าง object ใหม่ทุกครั้ง!
end
```

### 6.4 String Memory Management

```ruby
# String encoding
str = "สวัสดี"
str.encoding     # => UTF-8
str.bytesize     # => 21 (UTF-8: 3 bytes ต่อตัวอักษรไทย)
str.length       # => 7 (7 characters)

# ASCII-8BIT (binary)
binary = "\xFF\xFE".b
binary.encoding  # => ASCII-8BIT

# String#encode สำหรับ conversion
str.encode('TIS-620')  # Convert to Thai encoding

# ใช้ String#encode กับ error handling
begin
  str.encode('ASCII', invalid: :replace, undef: :replace, replace: '?')
rescue Encoding::UndefinedConversionError => e
  puts "Encoding error: #{e.message}"
end

# String interning (Ruby 3.0+)
# String#frozen? สำหรับ check
# String#-@ สำหรับ intern

# Memory-efficient string operations
require 'stringio'

# แทน String concatenation:
buffer = StringIO.new
buffer << "Part 1"
buffer << "Part 2"
buffer << "Part 3"
result = buffer.string  # Combine ท้ายสุดครั้งเดียว
```

---

## 7. Advanced Ruby Internals

### 7.1 Binding และ Closures

```ruby
# Binding คือ object ที่ encapsulates execution context
def get_binding(value)
  binding  # capture local scope
end

b = get_binding(42)
eval("value", b)  # => 42

# Closure เก็บ reference ไปยัง outer scope
def make_counter(start = 0)
  count = start
  
  incrementer = -> { count += 1; count }
  decrementer = -> { count -= 1; count }
  getter = -> { count }
  
  [incrementer, decrementer, getter]
end

inc, dec, get = make_counter(10)
inc.call  # => 11
inc.call  # => 12
dec.call  # => 11
get.call  # => 11
# count ถูก share ระหว่าง 3 lambdas

# Lambda vs Proc
my_lambda = lambda { |x| x * 2 }
my_proc = proc { |x| x * 2 }

# Lambda: strict argument checking
my_lambda.call(5)      # => 10
# my_lambda.call(5, 6)  # => ArgumentError

# Proc: lenient argument checking
my_proc.call(5)        # => 10
my_proc.call(5, 6)     # => 10 (ignores extra args)
my_proc.call           # => nil * 2 => NoMethodError for complex procs

# Return behavior
def test_lambda
  l = lambda { return 10 }
  result = l.call
  puts "After lambda: #{result}"
  20
end

def test_proc
  p = proc { return 10 }  # return ออกจาก method!
  p.call
  puts "After proc: never reaches here"
  20
end

test_lambda  # => prints "After lambda: 10", returns 20
test_proc    # => returns 10 (proc's return exits the method)
```

### 7.2 Kernel Methods และ Method Visibility

```ruby
# Kernel module มี methods ที่ดูเหมือน functions
puts "hello"       # Kernel#puts
p "hello"          # Kernel#p
print "hello"      # Kernel#print
pp "hello"         # Kernel#pp
gets               # Kernel#gets
require 'json'     # Kernel#require
rand               # Kernel#rand
sleep 1            # Kernel#sleep
exit               # Kernel#exit
raise "error"      # Kernel#raise

# เหล่านี้คือ private methods ของ Object (ผ่าน Kernel)
Object.include?(Kernel)  # => true
Kernel.instance_methods(false).sort.first(10)
# => [:Array, :Complex, :Float, :Hash, :Integer, :Rational, :String, ...]

# Global functions ที่แท้จริงคือ private instance methods
class Foo
  def bar
    puts "from bar"  # เรียก Kernel#puts
  end
end

# ดู method ว่ามาจากไหน
puts method(:puts).owner  # => Kernel
```

### 7.3 Method Objects

```ruby
# Method object เป็น first-class object
class Calculator
  def add(a, b)
    a + b
  end
  
  def multiply(a, b)
    a * b
  end
end

calc = Calculator.new

# Bound method
add_method = calc.method(:add)
add_method.call(3, 4)  # => 7

# Unbound method
unbound = Calculator.instance_method(:add)
# unbound.call(3, 4)  # Error: ยัง unbound

bound = unbound.bind(calc)
bound.call(3, 4)  # => 7

# Method เป็น Proc/Lambda
add_proc = calc.method(:add).to_proc
add_proc.call(3, 4)  # => 7

# Method composition (Ruby 2.6+)
double = method(:puts).>>(method(:pp))
# หรือ
add = ->(a, b) { a + b }
double = ->(x) { x * 2 }
double_then_add = double >> add.curry.(10)

# Method#curry
add_curried = add_method.curry
add_to_10 = add_curried.(10)
add_to_10.(5)   # => 15
add_to_10.(20)  # => 30
```

### 7.4 Refinements

```ruby
# Refinements ช่วยให้ extend class ในขอบเขตจำกัด
module StringExtensions
  refine String do
    def palindrome?
      self == self.reverse
    end
    
    def word_count
      split.length
    end
  end
  
  refine Integer do
    def factorial
      return 1 if self <= 1
      self * (self - 1).factorial
    end
  end
end

# ก่อน using, methods ไม่พร้อมใช้งาน
# "racecar".palindrome?  # => NoMethodError

module MyModule
  using StringExtensions
  
  def self.test
    puts "racecar".palindrome?   # => true
    puts "hello".palindrome?     # => false
    puts "hello world".word_count  # => 2
    puts 5.factorial              # => 120
  end
end

MyModule.test

# นอก scope ของ using
# "racecar".palindrome?  # => NoMethodError

# Refinements ไม่ส่งผลต่อ method_missing หรือ respond_to?
# แต่ใน Ruby 2.4+ refinements ทำงานใน instance_eval/module_eval
```

---

## แบบฝึกหัด (20 ข้อพร้อมเฉลย)

### ข้อ 1: Object Hierarchy
**คำถาม:** เขียนโค้ดที่แสดง complete ancestry chain ของ String class พร้อม methods count ในแต่ละ level

**เฉลย:**
```ruby
def show_ancestry(klass)
  puts "Ancestry of #{klass}:"
  puts "=" * 40
  
  klass.ancestors.each do |ancestor|
    methods = ancestor.instance_methods(false).count
    puts "#{ancestor.name || ancestor.inspect}: #{methods} methods"
  end
end

show_ancestry(String)
# Ancestry of String:
# ========================================
# String: 150 methods
# Comparable: 5 methods
# Object: 0 methods
# Kernel: 50 methods
# BasicObject: 8 methods
```

### ข้อ 2: Method Lookup Tracing
**คำถาม:** สร้าง module ที่ trace method lookup และแสดงว่า method ถูก found ที่ไหน

**เฉลย:**
```ruby
module MethodTracer
  def self.trace(obj, method_name)
    obj.class.ancestors.each do |ancestor|
      if ancestor.method_defined?(method_name) || 
         ancestor.private_method_defined?(method_name) ||
         (ancestor.respond_to?(:singleton_class) && 
          ancestor.singleton_class.method_defined?(method_name))
        puts "Found '#{method_name}' in #{ancestor}"
        return ancestor
      end
    end
    puts "Method '#{method_name}' not found!"
    nil
  end
end

MethodTracer.trace("hello", :upcase)
# Found 'upcase' in String
MethodTracer.trace("hello", :puts)
# Found 'puts' in Kernel
```

### ข้อ 3: GC Measurement
**คำถาม:** เขียน benchmark ที่ measure GC pressure สำหรับ 3 วิธีสร้าง strings

**เฉลย:**
```ruby
require 'benchmark'

def gc_stat_delta
  before = GC.stat.dup
  yield
  after = GC.stat
  {
    minor_gc: after[:minor_gc_count] - before[:minor_gc_count],
    major_gc: after[:major_gc_count] - before[:major_gc_count],
    freed: after[:total_freed_objects] - before[:total_freed_objects]
  }
end

N = 100_000

puts "String creation approaches:"

puts gc_stat_delta {
  N.times { "hello_#{rand(1000)}" }
}.inspect

puts gc_stat_delta {
  N.times { +"hello" << rand(1000).to_s }
}.inspect

puts gc_stat_delta {
  N.times { String.new("hello") << rand(1000).to_s }
}.inspect
```

### ข้อ 4: Eigenclass Exploration
**คำถาม:** สร้าง object และเพิ่ม methods ผ่าน eigenclass ในหลายวิธี

**เฉลย:**
```ruby
obj = Object.new

# วิธีที่ 1: def
def obj.greet
  "Hello from singleton method!"
end

# วิธีที่ 2: class << self
class << obj
  def farewell
    "Goodbye from eigenclass!"
  end
  
  def info
    "I am #{self.class} ##{object_id}"
  end
end

# วิธีที่ 3: define_singleton_method
obj.define_singleton_method(:compute) do |x|
  x * 2
end

puts obj.greet
puts obj.farewell
puts obj.info
puts obj.compute(21)
puts obj.singleton_class.instance_methods(false).inspect
# => [:greet, :farewell, :info, :compute]
```

### ข้อ 5: Frozen String Performance
**คำถาม:** benchmark แสดง performance difference ระหว่าง frozen และ mutable strings ในสถานการณ์ต่างๆ

**เฉลย:**
```ruby
require 'benchmark/ips'

Benchmark.ips do |x|
  x.config(time: 5, warmup: 2)
  
  x.report('mutable concat (<<)') do
    str = String.new
    100.times { str << "x" }
  end
  
  x.report('frozen constant') do
    FROZEN = "hello world".freeze
    FROZEN.length
  end
  
  x.report('frozen interned (-@)') do
    str = -"hello world"
    str.length
  end
  
  x.report('symbol to_s') do
    :hello_world.to_s.length
  end
  
  x.compare!
end
```

### ข้อ 6-20 (สรุปเฉลย):

**ข้อ 6: YARV Bytecode Analysis**
```ruby
# วิเคราะห์ bytecode ของ method และ compare
simple = RubyVM::InstructionSequence.compile('1 + 2').disasm
complex = RubyVM::InstructionSequence.compile('
  result = 0
  10.times { |i| result += i }
  result
').disasm

puts "Simple:\n#{simple}"
puts "Complex:\n#{complex}"
```

**ข้อ 7: Method Visibility**
```ruby
class SecureService
  def public_action
    internal_process
  end
  
  protected
  def shared_action; "shared"; end
  
  private
  def internal_process; "internal"; end
end

# Test visibility
svc = SecureService.new
puts svc.public_methods(false).inspect
puts svc.protected_methods(false).inspect
puts svc.private_methods(false).grep(/internal/).inspect
```

**ข้อ 8: Refinement Scope**
```ruby
module TimeHelpers
  refine Integer do
    def seconds; self; end
    def minutes; self * 60; end
    def hours; self * 3600; end
    def days; self * 86400; end
  end
end

module TimeCalculator
  using TimeHelpers
  
  def self.duration_in_hours(days:, hours:, minutes:)
    (days.days + hours.hours + minutes.minutes) / 1.hour
  end
end

puts TimeCalculator.duration_in_hours(days: 1, hours: 2, minutes: 30)
# => 26.5
```

**ข้อ 9: WeakRef Cache**
```ruby
require 'weakref'

class SmartCache
  def initialize
    @cache = {}
    @mutex = Mutex.new
  end
  
  def fetch(key)
    @mutex.synchronize do
      ref = @cache[key]
      if ref
        begin
          value = ref.__getobj__
          return value
        rescue WeakRef::RefError
          @cache.delete(key)
        end
      end
      
      value = yield
      @cache[key] = WeakRef.new(value)
      value
    end
  end
  
  def size
    @cache.size
  end
end
```

**ข้อ 10: Object Shape Optimization**
```ruby
# Demonstrate shape consistency benefit
require 'benchmark'

class Consistent
  def initialize(a, b, c)
    @a = a; @b = b; @c = c
  end
  def sum; @a + @b + @c; end
end

class Inconsistent
  def initialize(a, b, c)
    if a > 0
      @a = a; @b = b; @c = c
    else
      @c = c; @a = a; @b = b; @d = 0  # Different shape!
    end
  end
  def sum; @a + @b + @c; end
end

N = 500_000
Benchmark.bm(15) do |x|
  x.report('Consistent:') { N.times { |i| Consistent.new(i, i+1, i+2).sum } }
  x.report('Inconsistent:') { N.times { |i| Inconsistent.new(i, i+1, i+2).sum } }
end
```

**ข้อ 11: Closure Variable Capture**
```ruby
def make_adders
  adders = []
  [1, 2, 3, 4, 5].each do |i|
    adders << ->(x) { x + i }  # i ถูก captured ถูกต้อง
  end
  adders
end

adders = make_adders
puts adders[0].call(10)   # => 11
puts adders[2].call(10)   # => 13
puts adders[4].call(10)   # => 15
```

**ข้อ 12: Binding Exploration**
```ruby
def create_context(x, y)
  z = x + y
  binding
end

ctx = create_context(10, 20)

eval("puts x", ctx)  # => 10
eval("puts y", ctx)  # => 20
eval("puts z", ctx)  # => 30

# Modify variable through binding
eval("x = 100", ctx)
eval("puts x", ctx)  # => 100
```

**ข้อ 13: Method Currying**
```ruby
multiply = ->(a, b) { a * b }
double = multiply.curry.(2)
triple = multiply.curry.(3)

[1, 2, 3, 4, 5].map(&double)  # => [2, 4, 6, 8, 10]
[1, 2, 3, 4, 5].map(&triple)  # => [3, 6, 9, 12, 15]

# Method object as block
[1, -2, 3, -4].select(&method(:positive?))  # => [1, 3]
```

**ข้อ 14: Finalizer Pattern**
```ruby
class DatabaseConnection
  @@open_connections = 0
  
  def initialize(url)
    @url = url
    @id = SecureRandom.uuid
    @@open_connections += 1
    connect!
    
    ObjectSpace.define_finalizer(
      self, 
      self.class.create_finalizer(@id)
    )
  end
  
  def self.create_finalizer(id)
    proc do
      @@open_connections -= 1
      puts "Connection #{id} closed by GC"
    end
  end
  
  def self.open_connections
    @@open_connections
  end
  
  private
  def connect!; @connected = true; end
end
```

**ข้อ 15: Fiber Generator**
```ruby
def fibonacci
  Fiber.new do
    a, b = 0, 1
    loop do
      Fiber.yield a
      a, b = b, a + b
    end
  end
end

fib = fibonacci
10.times { print "#{fib.resume} " }
# => 0 1 1 2 3 5 8 13 21 34
```

**ข้อ 16: Instance Variable Introspection**
```ruby
class Inspector
  def initialize
    @name = "test"
    @value = 42
    @data = [1, 2, 3]
  end
  
  def inspect_ivars
    instance_variables.map do |ivar|
      value = instance_variable_get(ivar)
      "#{ivar}: #{value.inspect} (#{value.class})"
    end
  end
  
  def dynamic_ivar(name, value)
    instance_variable_set("@#{name}", value)
  end
end

obj = Inspector.new
obj.inspect_ivars.each { |info| puts info }
obj.dynamic_ivar(:new_field, "dynamic!")
puts obj.instance_variable_get(:@new_field)
```

**ข้อ 17: Protected Method Sharing**
```ruby
class Employee
  def initialize(salary)
    @salary = salary
  end
  
  def >(other)
    salary > other.salary
  end
  
  def compare_with(other)
    if self > other
      "I earn more"
    elsif other > self
      "Other earns more"  
    else
      "Equal salary"
    end
  end
  
  protected
  
  def salary
    @salary
  end
end

emp1 = Employee.new(50000)
emp2 = Employee.new(60000)
puts emp1.compare_with(emp2)  # => "Other earns more"
# emp1.salary  # => NoMethodError (protected)
```

**ข้อ 18: Proxy with BasicObject**
```ruby
class LoggingProxy < BasicObject
  def initialize(target)
    @target = target
  end
  
  def method_missing(name, *args, &block)
    ::Kernel.puts "Calling #{name}(#{args.inspect})"
    result = @target.send(name, *args, &block)
    ::Kernel.puts "Result: #{result.inspect}"
    result
  end
  
  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private)
  end
end

proxy = LoggingProxy.new("hello world")
proxy.upcase
# Calling upcase([])
# Result: "HELLO WORLD"
```

**ข้อ 19: GC Stress Test**
```ruby
require 'objspace'

def measure_allocations(&block)
  before = ObjectSpace.count_objects
  GC.disable
  
  block.call
  
  after = ObjectSpace.count_objects
  GC.enable
  GC.start
  
  diff = after.map { |k, v| [k, v - (before[k] || 0)] }.to_h
  diff.select { |_, v| v > 0 }
end

puts measure_allocations {
  1000.times { { a: 1, b: 2, c: [1, 2, 3] } }
}.inspect
```

**ข้อ 20: Complete Introspection Tool**
```ruby
module RubyInspector
  def self.analyze(obj)
    {
      class: obj.class,
      superclass: obj.class.superclass,
      ancestors: obj.class.ancestors,
      instance_variables: obj.instance_variables,
      public_methods: obj.public_methods(false),
      protected_methods: obj.protected_methods(false),
      private_methods: obj.private_methods(false),
      singleton_methods: obj.singleton_methods,
      frozen: obj.frozen?,
      object_id: obj.object_id,
      memory: ObjectSpace.memsize_of(obj)
    }
  end
  
  def self.compare(obj1, obj2)
    a1 = obj1.class.ancestors
    a2 = obj2.class.ancestors
    {
      shared_ancestors: a1 & a2,
      unique_to_1: a1 - a2,
      unique_to_2: a2 - a1
    }
  end
end

puts RubyInspector.analyze("hello").inspect
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Ruby Object Model** - Eigenclass, ancestry, object identity
2. **Method Lookup Chain** - MRO, super, visibility, BasicObject
3. **Garbage Collector** - Tri-color mark, generational GC, tuning
4. **YARV VM** - Bytecode, compilation, Fiber scheduler
5. **Object Shapes** - Ruby 3.2+ optimization สำหรับ instance variables
6. **Frozen Strings** - Memory optimization, performance benefits

ความเข้าใจ internals ช่วยให้:
- เขียนโค้ดที่ GC-friendly
- ใช้ object shapes อย่างมีประสิทธิภาพ
- Debug memory leaks และ performance issues
- สร้าง metaprogramming ที่ถูกต้อง
