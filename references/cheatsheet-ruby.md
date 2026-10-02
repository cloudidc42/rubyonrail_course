# Ruby Cheatsheet — ฉบับสมบูรณ์

> อ้างอิงด่วนสำหรับ Ruby syntax ทั้งหมด

---

## 1. Variables

```ruby
# Local variable
name = "Alice"
age  = 30

# Instance variable (ใน class)
@name = "Alice"

# Class variable
@@count = 0

# Global variable
$app_name = "MyApp"

# Constant
MAX_SIZE = 100
PI = 3.14159
```

## 2. Data Types

```ruby
# String
str = "Hello"
str = 'World'

# Integer
n = 42
n = 1_000_000  # underscore for readability

# Float
f = 3.14
f = 1.0e10

# Boolean
t = true
f = false

# Nil
x = nil

# Symbol
sym = :hello
sym = :"hello world"

# Array
arr = [1, 2, 3]
arr = %w[a b c]  # => ["a", "b", "c"]
arr = %i[a b c]  # => [:a, :b, :c]

# Hash
h = { name: "Alice", age: 30 }
h = { "name" => "Alice" }

# Range
r = (1..10)   # inclusive
r = (1...10)  # exclusive
```

## 3. String Methods

```ruby
s = "Hello, World!"

s.length          # => 13
s.size            # => 13
s.upcase          # => "HELLO, WORLD!"
s.downcase        # => "hello, world!"
s.capitalize      # => "Hello, world!"
s.reverse         # => "!dlroW ,olleH"
s.strip           # => removes leading/trailing whitespace
s.lstrip          # => removes leading whitespace
s.rstrip          # => removes trailing whitespace
s.chomp           # => removes trailing newline
s.chop            # => removes last character
s.squeeze         # => removes consecutive duplicates
s.split(",")      # => ["Hello", " World!"]
s.split(", ", 2)  # => ["Hello", "World!"]
s.include?("World")  # => true
s.start_with?("Hello")  # => true
s.end_with?("!")     # => true
s.gsub("World", "Ruby")  # => "Hello, Ruby!"
s.sub("l", "L")          # => "HeLlo, World!"
s.tr("aeiou", "*")       # => "H*ll*, W*rld!"
s.count("l")             # => 3
s.delete("l")            # => "Heo, Word!"
s.squeeze("l")           # => "Helo, World!"
s[0]                     # => "H"
s[0, 5]                  # => "Hello"
s[7..]                   # => "World!"
s[-1]                    # => "!"

# Interpolation (double quotes only)
name = "Alice"
"Hello, #{name}!"    # => "Hello, Alice!"
"2 + 2 = #{2 + 2}"  # => "2 + 2 = 4"

# Heredoc
text = <<~HEREDOC
  Hello
  World
HEREDOC

# Format
"%.2f" % 3.14159   # => "3.14"
"%s is %d" % ["Alice", 30]  # => "Alice is 30"
format("Hello, %s!", name)
sprintf("Pi = %.4f", Math::PI)

# Frozen String
str = "hello".freeze
str.frozen?  # => true
```

## 4. Integer Methods

```ruby
n = 42

n.odd?        # => false
n.even?       # => true
n.zero?       # => false
n.positive?   # => true
n.negative?   # => false
n.abs         # => 42
n.to_f        # => 42.0
n.to_s        # => "42"
n.to_s(2)     # => "101010" (binary)
n.to_s(16)    # => "2a" (hex)
n.digits      # => [2, 4] (reversed digits)
n.gcd(18)     # => 6
n.lcm(18)     # => 126
n.pow(3)      # => 74088

# Iteration
5.times { |i| puts i }           # 0,1,2,3,4
1.upto(5) { |i| puts i }         # 1,2,3,4,5
5.downto(1) { |i| puts i }       # 5,4,3,2,1
1.step(10, 2) { |i| puts i }     # 1,3,5,7,9
```

## 5. Float Methods

```ruby
f = 3.14159

f.round        # => 3
f.round(2)     # => 3.14
f.ceil         # => 4
f.floor        # => 3
f.truncate     # => 3
f.abs          # => 3.14159
f.to_i         # => 3
f.nan?         # => false
f.infinite?    # => nil (nil/1/-1)
f.finite?      # => true
```

## 6. Array Methods

```ruby
arr = [3, 1, 4, 1, 5, 9, 2, 6]

# Access
arr[0]          # => 3
arr[-1]         # => 6
arr[1, 3]       # => [1, 4, 1]
arr[1..3]       # => [1, 4, 1]
arr.first       # => 3
arr.first(2)    # => [3, 1]
arr.last        # => 6
arr.last(2)     # => [2, 6]
arr.sample      # random element
arr.sample(2)   # 2 random elements

# Info
arr.length      # => 8
arr.size        # => 8
arr.count       # => 8
arr.count(1)    # => 2 (count occurrences)
arr.empty?      # => false
arr.include?(5) # => true
arr.any? { |x| x > 8 }   # => true
arr.all? { |x| x > 0 }   # => true
arr.none? { |x| x > 10 } # => true
arr.one? { |x| x == 9 }  # => true

# Modify (mutates)
arr.push(7)       # add to end
arr << 7          # same as push
arr.pop           # remove from end
arr.unshift(0)    # add to beginning
arr.shift         # remove from beginning
arr.insert(2, 99) # insert at index
arr.delete(1)     # delete by value
arr.delete_at(0)  # delete by index
arr.clear         # empty the array

# Transform (returns new array)
arr.map { |x| x * 2 }
arr.collect { |x| x * 2 }  # alias
arr.select { |x| x.even? }
arr.filter { |x| x.even? }  # alias
arr.reject { |x| x.even? }
arr.reduce(0) { |sum, x| sum + x }
arr.inject(:+)  # same using symbol
arr.sort
arr.sort { |a, b| b <=> a }  # descending
arr.sort_by { |x| -x }       # descending
arr.reverse
arr.flatten           # flatten nested
arr.flatten(1)        # flatten one level
arr.compact           # remove nils
arr.uniq              # remove duplicates
arr.zip([1,2,3])      # zip with another
arr.take(3)           # first 3
arr.drop(3)           # all but first 3
arr.take_while { |x| x < 5 }
arr.drop_while { |x| x < 5 }
arr.each_slice(2).to_a    # groups of 2
arr.each_cons(3).to_a     # sliding windows

# Set operations
a = [1, 2, 3]
b = [2, 3, 4]
a | b    # => [1, 2, 3, 4] (union)
a & b    # => [2, 3] (intersection)
a - b    # => [1] (difference)
a + b    # => [1, 2, 3, 2, 3, 4] (concatenation)

# Useful
arr.sum         # => sum of elements
arr.min         # => minimum
arr.max         # => maximum
arr.minmax      # => [min, max]
arr.min_by { |x| x.to_s }
arr.max_by { |x| x }
arr.tally       # count occurrences => {3=>1, 1=>2, ...}
arr.flatten.map(&:to_s)
arr.each_with_index { |val, idx| ... }
arr.each_with_object({}) { |x, h| h[x] = x**2 }
arr.group_by { |x| x % 2 == 0 ? :even : :odd }
arr.partition { |x| x.even? }  # => [[evens], [odds]]
arr.flat_map { |x| [x, x*2] }
arr.chunk { |x| x.even? }
arr.zip([10,20,30]).map { |a, b| a + b }

# Destructuring
first, *rest = [1, 2, 3, 4]
# first => 1, rest => [2, 3, 4]
a, b, c = [1, 2, 3]
```

## 7. Hash Methods

```ruby
h = { name: "Alice", age: 30, city: "Bangkok" }

# Access
h[:name]            # => "Alice"
h.fetch(:name)      # => "Alice" (raises error if missing)
h.fetch(:x, "N/A")  # => "N/A" (default)
h.dig(:address, :city)  # nested safe access

# Info
h.keys              # => [:name, :age, :city]
h.values            # => ["Alice", 30, "Bangkok"]
h.length            # => 3
h.size              # => 3
h.empty?            # => false
h.has_key?(:name)   # => true
h.key?(:name)       # => true (alias)
h.include?(:name)   # => true (alias)
h.has_value?("Alice")  # => true
h.value?("Alice")      # => true (alias)

# Modify
h[:email] = "alice@ex.com"  # add
h.delete(:city)             # remove
h.store(:phone, "0800000")  # add (alias)
h.merge({ country: "TH" })  # merge (returns new)
h.merge!({ country: "TH" }) # merge (mutates)
h.update({ age: 31 })       # alias for merge!

# Transform
h.map { |k, v| [k, v.to_s] }.to_h
h.transform_values { |v| v.to_s }
h.transform_keys { |k| k.to_s }
h.select { |k, v| v.is_a?(String) }
h.reject { |k, v| v.is_a?(Integer) }
h.filter_map { |k, v| [k, v*2] if v.is_a?(Integer) }
h.any? { |k, v| v == "Alice" }
h.all? { |k, v| !v.nil? }
h.count { |k, v| v.is_a?(String) }
h.min_by { |k, v| v.to_s }
h.max_by { |k, v| v.to_s }
h.sort_by { |k, v| k.to_s }
h.group_by { |k, v| v.class }
h.flat_map { |k, v| [k, v] }
h.sum { |k, v| v.is_a?(Integer) ? v : 0 }
h.each { |k, v| puts "#{k}: #{v}" }
h.each_with_object([]) { |(k, v), arr| arr << "#{k}=#{v}" }

# Convert
h.to_a              # => [[:name, "Alice"], [:age, 30], ...]
h.flatten           # => [:name, "Alice", :age, 30, ...]
hash.invert         # swap keys/values
```

## 8. Control Flow

```ruby
# if/elsif/else
if condition
  # ...
elsif other_condition
  # ...
else
  # ...
end

# unless (negation of if)
unless condition
  # runs when false
end

# Ternary
result = condition ? "yes" : "no"

# One-liner
do_something if condition
do_something unless condition

# case/when
case value
when 1
  "one"
when 2, 3
  "two or three"
when 4..10
  "four to ten"
when String
  "it's a string"
when /pattern/
  "matches regex"
else
  "other"
end

# Pattern matching (Ruby 3+)
case user
in { name: String => name, age: (18..) }
  puts "Adult: #{name}"
in { name: String => name }
  puts "Minor: #{name}"
end

# Logical operators
&&, ||, !
and, or, not  # lower precedence, avoid in assignments

# Safe navigation
user&.name   # returns nil if user is nil
```

## 9. Loops

```ruby
# while
i = 0
while i < 5
  puts i
  i += 1
end

# until
until i >= 5
  i += 1
end

# loop (infinite, use break)
loop do
  break if condition
end

# for (uncommon)
for i in 1..5
  puts i
end

# times
5.times { |i| puts i }

# each
[1, 2, 3].each { |x| puts x }

# Keywords in loops
next    # skip to next iteration (like continue)
break   # exit loop
redo    # restart current iteration
retry   # retry rescue block
```

## 10. Methods

```ruby
# Basic
def greet(name)
  "Hello, #{name}!"
end

# Default parameter
def greet(name = "World")
  "Hello, #{name}!"
end

# Keyword arguments
def create_user(name:, age:, role: "user")
  { name: name, age: age, role: role }
end
create_user(name: "Alice", age: 30)

# Splat
def sum(*numbers)
  numbers.sum
end
sum(1, 2, 3, 4)  # => 10

# Double splat (keyword splat)
def config(**options)
  options
end
config(debug: true, verbose: false)

# Block parameter
def with_logging(&block)
  puts "Starting"
  result = block.call
  puts "Done"
  result
end

# Implicit return (last expression)
def double(n)
  n * 2  # implicit return
end

# Explicit return
def grade(score)
  return "F" if score < 60
  return "C" if score < 70
  return "B" if score < 80
  "A"
end

# Method aliases
alias_method :say_hello, :greet
```

## 11. Blocks, Procs, Lambdas

```ruby
# Block
[1,2,3].each { |n| puts n }      # inline
[1,2,3].each do |n|               # multi-line
  puts n
end

# yield
def run
  yield if block_given?
end
run { puts "Hello" }

# Proc
double = Proc.new { |n| n * 2 }
double.call(5)   # => 10
double.(5)       # => 10
double[5]        # => 10

# Lambda
square = lambda { |n| n ** 2 }
square = ->(n) { n ** 2 }  # stabby lambda
square.call(4)   # => 16

# Differences: Proc vs Lambda
# 1. Lambda checks arity; Proc does not
# 2. return in Lambda returns from lambda;
#    return in Proc returns from enclosing method

# Method object
m = method(:puts)
m.call("hello")
[1,2,3].each(&method(:puts))

# Currying
multiply = ->(a, b) { a * b }
double = multiply.curry.(2)
double.(5)  # => 10
```

## 12. Classes

```ruby
class Animal
  @@count = 0                    # class variable

  attr_accessor :name, :age     # getter + setter
  attr_reader :species           # getter only

  CATEGORY = "Vertebrate"        # constant

  def initialize(name, age, species)
    @name    = name
    @age     = age
    @species = species
    @@count  += 1
  end

  def self.count                 # class method
    @@count
  end

  def speak                      # instance method
    "..."
  end

  def to_s                       # string representation
    "#{@name} (#{@species})"
  end

  def ==(other)                  # equality
    name == other.name && species == other.species
  end

  private

  def secret
    "hidden"
  end
end

# Inheritance
class Dog < Animal
  def initialize(name, age)
    super(name, age, "Dog")
  end

  def speak
    "Woof!"
  end
end
```

## 13. Modules

```ruby
module Greetable
  def greet
    "Hello, I'm #{name}"
  end
end

module Farewell
  def bye
    "Goodbye from #{name}"
  end
end

class Person
  include Greetable    # adds as instance methods
  extend Farewell      # adds as class methods

  attr_reader :name

  def initialize(name)
    @name = name
  end
end

# Namespace
module Payment
  class Processor
    def charge(amount)
      # ...
    end
  end
end

Payment::Processor.new.charge(100)
```

## 14. Error Handling

```ruby
begin
  result = 10 / 0
rescue ZeroDivisionError => e
  puts "Error: #{e.message}"
rescue TypeError, ArgumentError => e
  puts "Type/Arg error: #{e.message}"
rescue => e  # catches StandardError
  puts "Unexpected: #{e.message}"
else
  puts "No error!"
ensure
  puts "Always runs"
end

# Custom exception
class AppError < StandardError
  def initialize(msg = "Application error")
    super(msg)
  end
end

raise AppError, "Something went wrong"
raise AppError.new("Custom message")

# retry
attempts = 0
begin
  attempts += 1
  dangerous_operation()
rescue NetworkError
  retry if attempts < 3
  raise
end
```

## 15. File I/O

```ruby
# Read entire file
content = File.read("file.txt")

# Read lines
lines = File.readlines("file.txt", chomp: true)

# Write
File.write("file.txt", "content")

# Append
File.write("file.txt", "more", mode: "a")

# Open with block (auto-closes)
File.open("file.txt", "r") do |f|
  f.each_line { |line| puts line }
end

# Check existence
File.exist?("file.txt")
File.directory?("dir")
File.file?("file.txt")

# CSV
require 'csv'
CSV.foreach("data.csv", headers: true) do |row|
  puts row["name"]
end

CSV.open("output.csv", "w") do |csv|
  csv << ["name", "age"]
  csv << ["Alice", 30]
end

# JSON
require 'json'
data = JSON.parse(File.read("data.json"))
File.write("output.json", JSON.pretty_generate(data))
```

## 16. Enumerable Quick Reference

```ruby
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

arr.map { |x| x * 2 }              # [2,4,6,8,10,12,14,16,18,20]
arr.select { |x| x.even? }         # [2,4,6,8,10]
arr.reject { |x| x.even? }         # [1,3,5,7,9]
arr.reduce(:+)                      # 55
arr.reduce(100, :+)                 # 155
arr.inject { |sum, x| sum + x }    # 55
arr.find { |x| x > 5 }             # 6
arr.find_index { |x| x > 5 }       # 5
arr.count { |x| x.even? }          # 5
arr.sum                             # 55
arr.sum { |x| x * 2 }             # 110
arr.min                             # 1
arr.max                             # 10
arr.minmax                          # [1, 10]
arr.min_by { |x| -x }             # 10
arr.max_by { |x| -x }             # 1
arr.sort                            # sorted
arr.sort_by { |x| -x }            # reverse sorted
arr.group_by { |x| x % 3 }        # {1=>[1,4,7,10], 2=>[2,5,8], 0=>[3,6,9]}
arr.partition { |x| x.even? }      # [[2,4,6,8,10], [1,3,5,7,9]]
arr.tally                           # {1=>1, 2=>1, ...}
arr.flat_map { |x| [x, -x] }      # [1,-1,2,-2,...]
arr.zip([11..20])                   # pairs
arr.each_slice(3).to_a             # [[1,2,3],[4,5,6],[7,8,9],[10]]
arr.each_cons(3).to_a              # [[1,2,3],[2,3,4],...]
arr.take(3)                         # [1,2,3]
arr.drop(7)                         # [8,9,10]
arr.take_while { |x| x < 5 }      # [1,2,3,4]
arr.drop_while { |x| x < 5 }      # [5,6,7,8,9,10]
arr.chunk { |x| x < 5 }.to_a      # grouped by predicate
arr.any? { |x| x > 9 }            # true
arr.all? { |x| x > 0 }            # true
arr.none? { |x| x > 10 }          # true
arr.one? { |x| x > 9 }            # true
arr.first(3)                        # [1,2,3]
arr.last(3)                         # [8,9,10]
arr.sample(3)                       # 3 random
arr.shuffle                         # shuffled
arr.lazy.select(&:even?).first(3)  # lazy evaluation
```

## 17. Ruby 3+ Features

```ruby
# Pattern matching
case [1, 2, 3]
in [Integer => a, Integer => b, Integer => c]
  puts "Three integers: #{a}, #{b}, #{c}"
end

# Hash pattern
case { name: "Alice", role: :admin }
in { name: String => name, role: :admin }
  puts "Admin: #{name}"
end

# Find pattern
case [1, 2, "three", 4, 5]
in [*, String => s, *]
  puts "Found string: #{s}"
end

# One-line pattern (=>)
{ name: "Alice" } => { name: String => name }
puts name  # => "Alice"

# Pin operator (^)
expected = 42
case value
in ^expected
  puts "Matched expected!"
end

# Endless range
[1, 2, 3, 4, 5].select { |x| x in (3..) }  # [3,4,5]

# Hash shorthand (Ruby 3.1)
x = 1; y = 2
{ x:, y: }  # => { x: 1, y: 2 }

# Numbered block parameters (Ruby 2.7)
[1, 2, 3].map { _1 * 2 }   # => [2, 4, 6]
[[1,2],[3,4]].map { _1 + _2 }  # => [3, 7]

# Ractor (experimental parallel execution)
r = Ractor.new { "Hello from Ractor!" }
puts r.take
```

---

*สรุป Ruby syntax ทั้งหมดในไฟล์เดียว — ใช้เป็น reference ขณะเขียนโค้ด*
