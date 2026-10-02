# ตอนที่ 6: Hashes (ขั้นตอนที่ 81-100)

## บทนำ

Hash เป็นโครงสร้างข้อมูลที่สำคัญมากใน Ruby ที่เก็บข้อมูลแบบ key-value pairs คล้ายกับ dictionary ในภาษาอื่น ๆ เช่น Python หรือ Object ใน JavaScript Hash ช่วยให้เราสามารถเก็บและเข้าถึงข้อมูลโดยใช้ชื่อ (key) แทนที่จะเป็น index ตัวเลขแบบ Array

ในตอนนี้เราจะเรียนรู้:
- การสร้าง Hash ด้วยวิธีต่าง ๆ
- การเข้าถึงและแก้ไขค่าใน Hash
- Methods ที่สำคัญของ Hash
- การรวม Hash
- ค่า default
- Symbol keys vs String keys
- Nested hashes
- การแปลงระหว่าง Hash และ Array
- Pattern ที่ใช้บ่อยใน Ruby

---

## ขั้นตอนที่ 81: การสร้าง Hash - Old Syntax (Rocket Syntax)

ก่อน Ruby 1.9 การสร้าง Hash ใช้สัญลักษณ์ `=>` (เรียกว่า rocket หรือ hash rocket)

```ruby
# วิธีเก่า (Rocket Syntax)
person = { "name" => "สมชาย", "age" => 25, "city" => "กรุงเทพ" }

puts person["name"]   # => สมชาย
puts person["age"]    # => 25
puts person["city"]   # => กรุงเทพ

# Hash ที่ใช้ Integer เป็น key
scores = { 1 => "หนึ่ง", 2 => "สอง", 3 => "สาม" }
puts scores[1]   # => หนึ่ง
puts scores[2]   # => สอง

# Hash ที่มี key หลายประเภท
mixed = { "string_key" => 1, 2 => "integer_key", :symbol => "symbol_key" }
puts mixed["string_key"]   # => 1
puts mixed[2]              # => integer_key
puts mixed[:symbol]        # => symbol_key
```

Rocket syntax ยังคงใช้ได้ในปัจจุบัน และจำเป็นต้องใช้เมื่อ key ไม่ใช่ Symbol

---

## ขั้นตอนที่ 82: การสร้าง Hash - New Syntax (Symbol Shorthand)

ตั้งแต่ Ruby 1.9 เป็นต้นมา มีวิธีใหม่ที่กระชับกว่าสำหรับ Symbol keys

```ruby
# วิธีใหม่ (Symbol Shorthand) - สำหรับ Symbol keys เท่านั้น
person = { name: "สมชาย", age: 25, city: "กรุงเทพ" }

puts person[:name]   # => สมชาย
puts person[:age]    # => 25
puts person[:city]   # => กรุงเทพ

# เปรียบเทียบทั้งสองวิธี - ผลลัพธ์เหมือนกัน
old_syntax = { :name => "สมหญิง", :age => 30 }
new_syntax = { name: "สมหญิง", age: 30 }

puts old_syntax == new_syntax   # => true
puts old_syntax[:name]          # => สมหญิง
puts new_syntax[:name]          # => สมหญิง

# Hash ว่างเปล่า
empty_hash1 = {}
empty_hash2 = Hash.new

puts empty_hash1.class   # => Hash
puts empty_hash2.class   # => Hash
puts empty_hash1 == empty_hash2   # => true
```

**หมายเหตุ:** New syntax ใช้ได้เฉพาะเมื่อ key เป็น Symbol เท่านั้น หากต้องการ key ประเภทอื่น ต้องใช้ rocket syntax

---

## ขั้นตอนที่ 83: Ruby 3.1+ Symbol Shorthand สำหรับ Variables

ใน Ruby 3.1 มีการเพิ่ม shorthand ใหม่สำหรับการสร้าง Hash จาก variables

```ruby
# Ruby 3.1+ shorthand
name = "สมชาย"
age = 25
city = "กรุงเทพ"

# วิธีเดิม
person_old = { name: name, age: age, city: city }

# Ruby 3.1+ shorthand (ชื่อ variable กลายเป็น key อัตโนมัติ)
person_new = { name:, age:, city: }

puts person_old == person_new   # => true
puts person_new[:name]          # => สมชาย
puts person_new[:age]           # => 25
```

---

## ขั้นตอนที่ 84: การเข้าถึงค่าใน Hash

มีหลายวิธีในการเข้าถึงค่าใน Hash

```ruby
product = {
  name: "MacBook Pro",
  price: 75000,
  brand: "Apple",
  in_stock: true
}

# วิธีที่ 1: ใช้ [] operator
puts product[:name]     # => MacBook Pro
puts product[:price]    # => 75000

# วิธีที่ 2: ใช้ fetch method (safe - throw error ถ้าไม่พบ key)
puts product.fetch(:brand)   # => Apple

# fetch พร้อม default value (ไม่ throw error)
puts product.fetch(:color, "ไม่ระบุ")   # => ไม่ระบุ

# fetch พร้อม block (ไม่ throw error)
puts product.fetch(:weight) { |key| "ไม่พบ key: #{key}" }
# => ไม่พบ key: weight

# วิธีที่ 3: dig - เข้าถึง nested hash
nested = { user: { profile: { name: "สมชาย" } } }
puts nested.dig(:user, :profile, :name)   # => สมชาย
puts nested.dig(:user, :address, :city)   # => nil (ไม่ error)

# เปรียบเทียบ [] กับ fetch
puts product[:color]           # => nil (ไม่ error แต่คืนค่า nil)
# puts product.fetch(:color)   # KeyError: key not found: :color
```

---

## ขั้นตอนที่ 85: การเพิ่มและแก้ไขค่าใน Hash

```ruby
student = { name: "สมชาย", grade: "A" }

# เพิ่ม key ใหม่
student[:age] = 20
student[:school] = "มหาวิทยาลัยกรุงเทพ"
puts student
# => {:name=>"สมชาย", :grade=>"A", :age=>20, :school=>"มหาวิทยาลัยกรุงเทพ"}

# แก้ไขค่าที่มีอยู่แล้ว
student[:grade] = "A+"
puts student[:grade]   # => A+

# ลบ key
student.delete(:school)
puts student
# => {:name=>"สมชาย", :grade=>"A+", :age=>20}

# delete พร้อม block (ถ้าไม่พบ key)
result = student.delete(:address) { |key| "ไม่พบ #{key}" }
puts result   # => ไม่พบ address

# store method (เหมือน []=)
student.store(:email, "somchai@example.com")
puts student[:email]   # => somchai@example.com
```

---

## ขั้นตอนที่ 86: Hash Methods - keys, values, length

```ruby
inventory = {
  apple: 50,
  banana: 30,
  orange: 20,
  grape: 10
}

# keys - คืนค่า Array ของ keys ทั้งหมด
puts inventory.keys.inspect
# => [:apple, :banana, :orange, :grape]

# values - คืนค่า Array ของ values ทั้งหมด
puts inventory.values.inspect
# => [50, 30, 20, 10]

# length / size - จำนวน pairs
puts inventory.length   # => 4
puts inventory.size     # => 4

# empty? - ตรวจสอบว่าว่างเปล่า
puts inventory.empty?   # => false
puts {}.empty?          # => true

# has_key? / key? / include? / member? - ตรวจสอบ key
puts inventory.has_key?(:apple)    # => true
puts inventory.key?(:mango)        # => false
puts inventory.include?(:banana)   # => true

# has_value? / value? - ตรวจสอบ value
puts inventory.has_value?(50)   # => true
puts inventory.value?(100)      # => false

# key(value) - หา key จาก value
puts inventory.key(30)   # => banana
puts inventory.key(99)   # => nil
```

---

## ขั้นตอนที่ 87: Hash Methods - each, each_pair

```ruby
prices = { rice: 45, noodle: 55, soup: 65 }

# each - วนลูปทุก key-value pairs
prices.each do |key, value|
  puts "#{key}: #{value} บาท"
end
# => rice: 45 บาท
# => noodle: 55 บาท
# => soup: 65 บาท

# each_pair - เหมือน each
prices.each_pair do |food, price|
  puts "ราคา#{food} คือ #{price} บาท"
end

# each_key - วนลูปเฉพาะ keys
prices.each_key do |key|
  puts key
end

# each_value - วนลูปเฉพาะ values
prices.each_value do |value|
  puts value
end

# each_with_object - วนลูปพร้อมสะสมผลลัพธ์
total = prices.each_with_object(0) do |(key, value), sum|
  sum + value
end
# หมายเหตุ: each_with_object ไม่ดีสำหรับสะสม Integer
# ใช้ reduce แทนดีกว่า

# ใช้ each_with_object สำหรับ Array หรือ Hash
expensive = prices.each_with_object([]) do |(food, price), arr|
  arr << food if price > 50
end
puts expensive.inspect   # => [:noodle, :soup]
```

---

## ขั้นตอนที่ 88: Hash Methods - map, transform_values, transform_keys

```ruby
prices = { rice: 45, noodle: 55, soup: 65 }

# map - แปลง Hash เป็น Array of arrays
result = prices.map { |key, value| [key, value * 2] }
puts result.inspect
# => [[:rice, 90], [:noodle, 110], [:soup, 130]]

# map.to_h - แปลงกลับเป็น Hash
doubled_prices = prices.map { |key, value| [key, value * 2] }.to_h
puts doubled_prices.inspect
# => {:rice=>90, :noodle=>110, :soup=>130}

# transform_values - แปลงเฉพาะ values (Ruby 2.4+)
discounted = prices.transform_values { |v| (v * 0.9).round }
puts discounted.inspect
# => {:rice=>41, :noodle=>50, :soup=>59}

# transform_values! - แปลง in-place
prices_copy = prices.dup
prices_copy.transform_values! { |v| v + 10 }
puts prices_copy.inspect
# => {:rice=>55, :noodle=>65, :soup=>75}

# transform_keys - แปลงเฉพาะ keys (Ruby 2.5+)
string_keys = prices.transform_keys { |k| k.to_s }
puts string_keys.inspect
# => {"rice"=>45, "noodle"=>55, "soup"=>65}

# transform_keys! - แปลง in-place
symbol_keys = string_keys.transform_keys(&:to_sym)
puts symbol_keys.inspect
# => {:rice=>45, :noodle=>55, :soup=>65}
```

---

## ขั้นตอนที่ 89: Hash Methods - select, reject, filter

```ruby
students = {
  alice: 85,
  bob: 62,
  carol: 91,
  dave: 47,
  eve: 78
}

# select / filter - เลือก pairs ที่ตรงเงื่อนไข
passing = students.select { |name, score| score >= 70 }
puts passing.inspect
# => {:alice=>85, :carol=>91, :eve=>78}

# reject - ตรงข้ามกับ select
failing = students.reject { |name, score| score >= 70 }
puts failing.inspect
# => {:bob=>62, :dave=>47}

# filter_map - เลือกและแปลงพร้อมกัน (Ruby 2.7+)
high_scorers = students.filter_map do |name, score|
  "#{name}(#{score})" if score >= 80
end
puts high_scorers.inspect
# => ["alice(85)", "carol(91)"]

# any? - ตรวจว่ามีอย่างน้อยหนึ่งที่ตรงเงื่อนไข
has_failing = students.any? { |name, score| score < 50 }
puts has_failing   # => true

# all? - ตรวจว่าทุกอันตรงเงื่อนไข
all_passing = students.all? { |name, score| score >= 50 }
puts all_passing   # => true

# none? - ตรวจว่าไม่มีอันไหนตรงเงื่อนไข
no_perfect = students.none? { |name, score| score == 100 }
puts no_perfect   # => true

# count - นับจำนวนที่ตรงเงื่อนไข
high_count = students.count { |name, score| score >= 80 }
puts high_count   # => 2
```

---

## ขั้นตอนที่ 90: Hash Methods - find, min_by, max_by, sort_by

```ruby
products = {
  laptop: 35000,
  phone: 15000,
  tablet: 18000,
  watch: 8000,
  headphone: 5000
}

# find / detect - หา pair แรกที่ตรงเงื่อนไข
first_expensive = products.find { |item, price| price > 20000 }
puts first_expensive.inspect   # => [:laptop, 35000]

# min_by - หา pair ที่มีค่าน้อยที่สุด
cheapest = products.min_by { |item, price| price }
puts cheapest.inspect   # => [:headphone, 5000]

# max_by - หา pair ที่มีค่ามากที่สุด
most_expensive = products.max_by { |item, price| price }
puts most_expensive.inspect   # => [:laptop, 35000]

# minmax_by - หาทั้ง min และ max พร้อมกัน
range = products.minmax_by { |item, price| price }
puts range.inspect
# => [[:headphone, 5000], [:laptop, 35000]]

# sort_by - เรียงลำดับ (คืนค่าเป็น Array of arrays)
sorted_by_price = products.sort_by { |item, price| price }
puts sorted_by_price.inspect
# => [[:headphone, 5000], [:watch, 8000], ...]

# sort_by กลับด้าน
sorted_desc = products.sort_by { |item, price| -price }
puts sorted_desc.first.inspect   # => [:laptop, 35000]

# sum - รวมค่า
total = products.sum { |item, price| price }
puts total   # => 81000
```

---

## ขั้นตอนที่ 91: การรวม Hash - merge, update

```ruby
defaults = { color: "blue", size: "medium", weight: 100 }
custom = { color: "red", weight: 150 }

# merge - รวม hash (ไม่เปลี่ยน original)
merged = defaults.merge(custom)
puts merged.inspect
# => {:color=>"red", :size=>"medium", :weight=>150}
puts defaults.inspect   # ไม่เปลี่ยน

# merge! / update - รวม in-place (เปลี่ยน original)
defaults_copy = defaults.dup
defaults_copy.merge!(custom)
puts defaults_copy.inspect
# => {:color=>"red", :size=>"medium", :weight=>150}

# merge พร้อม block - กำหนดว่าจะใช้ค่าไหนเมื่อ key ซ้ำ
h1 = { a: 1, b: 2, c: 3 }
h2 = { b: 20, c: 30, d: 40 }

# block ได้รับ (key, old_value, new_value)
merged_custom = h1.merge(h2) do |key, old_val, new_val|
  old_val + new_val  # รวมค่าเมื่อ key ซ้ำ
end
puts merged_custom.inspect
# => {:a=>1, :b=>22, :c=>33, :d=>40}

# รวมหลาย Hash พร้อมกัน
h3 = { e: 5 }
all_merged = h1.merge(h2, h3)
puts all_merged.inspect

# ** operator (splat) - อีกวิธีในการรวม Hash (Ruby 2.0+)
combined = { **defaults, **custom }
puts combined.inspect
# => {:color=>"red", :size=>"medium", :weight=>150}
```

---

## ขั้นตอนที่ 92: Default Values ใน Hash

```ruby
# วิธีที่ 1: Hash.new พร้อม default value
counter = Hash.new(0)
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]

words.each { |word| counter[word] += 1 }
puts counter.inspect
# => {"apple"=>3, "banana"=>2, "cherry"=>1}

puts counter["mango"]   # => 0 (ไม่ error, คืน default)

# วิธีที่ 2: Hash.new พร้อม block
grouped = Hash.new { |hash, key| hash[key] = [] }
data = [["admin", "alice"], ["user", "bob"], ["admin", "carol"]]

data.each { |role, name| grouped[role] << name }
puts grouped.inspect
# => {"admin"=>["alice", "carol"], "user"=>["bob"]}

# วิธีที่ 3: fetch พร้อม default
config = { host: "localhost", port: 3000 }
puts config.fetch(:host, "0.0.0.0")       # => localhost
puts config.fetch(:timeout, 30)            # => 30
puts config.fetch(:debug, false)           # => false

# วิธีที่ 4: || operator
settings = { theme: "dark" }
font_size = settings[:font_size] || 14
puts font_size   # => 14

# วิธีที่ 5: fetch_values - เอาหลาย values พร้อมกัน
db = { user: "admin", pass: "secret", host: "db.local" }
user, pass = db.fetch_values(:user, :pass)
puts user   # => admin
puts pass   # => secret
```

---

## ขั้นตอนที่ 93: Symbol Keys vs String Keys

```ruby
# Symbol keys - เร็วกว่า ประหยัด memory กว่า
symbol_hash = { name: "Alice", age: 30 }

# String keys - ใช้เมื่อ key มาจาก user input หรือ JSON
string_hash = { "name" => "Alice", "age" => 30 }

# สำคัญ: Symbol และ String keys ต่างกัน!
puts symbol_hash[:name]    # => Alice
puts symbol_hash["name"]   # => nil (ไม่พบ!)

puts string_hash["name"]   # => Alice
puts string_hash[:name]    # => nil (ไม่พบ!)

# แปลง String keys เป็น Symbol keys
string_hash.transform_keys(&:to_sym)
# => {:name=>"Alice", :age=>30}

# แปลง Symbol keys เป็น String keys
symbol_hash.transform_keys(&:to_s)
# => {"name"=>"Alice", "age"=>30}

# HashWithIndifferentAccess (Rails) - ใช้ได้ทั้ง Symbol และ String
# ในโค้ด Rails จะใช้แบบนี้บ่อย
# params[:name] == params["name"]

# Frozen String Keys (Ruby 3.0+ optimization)
frozen_key_hash = { "name": "Alice" }   # ใช้ Symbol ที่ frozen
puts frozen_key_hash[:name]   # => Alice

# ตัวอย่างการใช้งานจริง - JSON parsing
require 'json'

json_string = '{"name": "Alice", "age": 30}'
parsed = JSON.parse(json_string)
puts parsed["name"]   # => Alice (String keys)

# แปลงเป็น Symbol keys
symbolized = parsed.transform_keys(&:to_sym)
puts symbolized[:name]   # => Alice (Symbol keys)
```

---

## ขั้นตอนที่ 94: Nested Hashes

```ruby
# Nested Hash - Hash ซ้อนกัน
user = {
  name: "สมชาย",
  age: 25,
  address: {
    street: "123 ถนนสุขุมวิท",
    city: "กรุงเทพ",
    zip: "10110"
  },
  contacts: {
    email: "somchai@example.com",
    phone: "081-234-5678"
  }
}

# เข้าถึงข้อมูลใน nested hash
puts user[:name]                    # => สมชาย
puts user[:address][:city]          # => กรุงเทพ
puts user[:contacts][:email]        # => somchai@example.com

# ใช้ dig method (ปลอดภัยกว่า)
puts user.dig(:address, :city)      # => กรุงเทพ
puts user.dig(:address, :country)   # => nil (ไม่ error)
puts user.dig(:social, :twitter)    # => nil (ไม่ error แม้ :social ไม่มี)

# เพิ่มข้อมูลใน nested hash
user[:address][:country] = "ไทย"
puts user[:address][:country]   # => ไทย

# แก้ไขข้อมูลใน nested hash
user[:contacts][:phone] = "082-345-6789"

# Deep merge - รวม nested hash
require 'active_support/core_ext/hash/deep_merge'
# หรือเขียน deep_merge เอง
def deep_merge(hash1, hash2)
  hash1.merge(hash2) do |key, old_val, new_val|
    if old_val.is_a?(Hash) && new_val.is_a?(Hash)
      deep_merge(old_val, new_val)
    else
      new_val
    end
  end
end

config1 = { server: { host: "localhost", port: 3000 }, debug: false }
config2 = { server: { port: 8080, ssl: true }, log: true }

merged = deep_merge(config1, config2)
puts merged.inspect
# => {:server=>{:host=>"localhost", :port=>8080, :ssl=>true}, :debug=>false, :log=>true}
```

---

## ขั้นตอนที่ 95: การแปลงระหว่าง Hash และ Array

```ruby
# Hash เป็น Array
h = { a: 1, b: 2, c: 3 }

# to_a - แปลงเป็น Array of pairs
arr = h.to_a
puts arr.inspect   # => [[:a, 1], [:b, 2], [:c, 3]]

# flatten - ทำให้แบนราบ
flat = h.to_a.flatten
puts flat.inspect   # => [:a, 1, :b, 2, :c, 3]

# Array เป็น Hash
pairs = [[:x, 10], [:y, 20], [:z, 30]]

# to_h method
hash1 = pairs.to_h
puts hash1.inspect   # => {:x=>10, :y=>20, :z=>30}

# Hash[] constructor
hash2 = Hash[pairs]
puts hash2.inspect   # => {:x=>10, :y=>20, :z=>30}

# Hash[] จาก flat array
hash3 = Hash[:a, 1, :b, 2, :c, 3]
puts hash3.inspect   # => {:a=>1, :b=>2, :c=>3}

# keys และ values เป็น Array
h = { name: "Alice", age: 30, city: "Bangkok" }
keys = h.keys     # => [:name, :age, :city]
values = h.values # => ["Alice", 30, "Bangkok"]

# zip keys และ values กลับมาเป็น Hash
zipped = keys.zip(values).to_h
puts zipped.inspect   # => {:name=>"Alice", :age=>30, :city=>"Bangkok"}

# map พร้อม to_h
doubled = h.map { |k, v| [k, v.is_a?(Integer) ? v * 2 : v] }.to_h
puts doubled.inspect
# => {:name=>"Alice", :age=>60, :city=>"Bangkok"}

# each_with_object สร้าง Hash ใหม่
inverted_style = (1..5).each_with_object({}) do |n, hash|
  hash[n] = n ** 2
end
puts inverted_style.inspect
# => {1=>1, 2=>4, 3=>9, 4=>16, 5=>25}
```

---

## ขั้นตอนที่ 96: Hash Methods เพิ่มเติม - flatten, invert, any?, all?

```ruby
h = { a: 1, b: 2, c: 3 }

# invert - สลับ key และ value
inverted = h.invert
puts inverted.inspect   # => {1=>:a, 2=>:b, 3=>:c}

# flatten - ทำให้ nested hash แบนราบ
nested = { a: [1, 2], b: [3, 4] }
puts nested.to_a.flatten.inspect   # => [:a, 1, 2, :b, 3, 4]
puts nested.flatten.inspect        # => [:a, 1, 2, :b, 3, 4]
puts nested.flatten(1).inspect     # => [:a, [1, 2], :b, [3, 4]]

# slice - เอาเฉพาะ keys ที่ระบุ (Ruby 2.5+)
person = { name: "Alice", age: 30, email: "alice@example.com", phone: "111" }
contact_info = person.slice(:email, :phone)
puts contact_info.inspect
# => {:email=>"alice@example.com", :phone=>"111"}

# except - ยกเว้น keys ที่ระบุ (Ruby 3.0+)
without_sensitive = person.except(:email, :phone)
puts without_sensitive.inspect
# => {:name=>"Alice", :age=>30}

# group_by - จัดกลุ่ม
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
grouped = numbers.group_by { |n| n % 3 }
puts grouped.inspect
# => {1=>[1, 4, 7, 10], 2=>[2, 5, 8], 0=>[3, 6, 9]}

# tally - นับความถี่ (Ruby 2.7+)
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
count = words.tally
puts count.inspect
# => {"apple"=>3, "banana"=>2, "cherry"=>1}
```

---

## ขั้นตอนที่ 97: Hash Destructuring

```ruby
# Basic destructuring ใน block parameters
hash = { name: "Alice", age: 30, city: "Bangkok" }

# วิธีเดิม
hash.each do |pair|
  puts pair.inspect
end

# Destructuring ใน block
hash.each do |key, value|
  puts "#{key}: #{value}"
end

# Pattern matching destructuring (Ruby 2.7+)
case hash
in { name: String => name, age: (18..) => age }
  puts "ผู้ใหญ่: #{name}, อายุ #{age}"
end

# Rightward assignment (Ruby 3.0+)
# hash => { name:, age: }

# Destructuring ใน method arguments
def greet(name:, age:, **rest)
  puts "สวัสดี #{name}! อายุ #{age} ปี"
  puts "ข้อมูลเพิ่มเติม: #{rest}" unless rest.empty?
end

greet(name: "Alice", age: 30)
greet(name: "Bob", age: 25, city: "Chiang Mai")

# Multiple assignment จาก Hash
data = { x: 10, y: 20, z: 30 }
x, y = data.values_at(:x, :y)
puts x   # => 10
puts y   # => 20

# values_at
coords = data.values_at(:x, :y, :z)
puts coords.inspect   # => [10, 20, 30]

# assoc - หา pair จาก key
found = hash.assoc(:name)
puts found.inspect   # => [:name, "Alice"]

# rassoc - หา pair จาก value
found2 = hash.rassoc(30)
puts found2.inspect  # => [:age, 30]
```

---

## ขั้นตอนที่ 98: Common Patterns ใน Ruby - Hash

```ruby
# Pattern 1: Counting frequency
def count_frequency(items)
  items.tally
  # Ruby 2.7+: items.tally
  # หรือ: items.each_with_object(Hash.new(0)) { |item, h| h[item] += 1 }
end

words = ["the", "quick", "brown", "fox", "the", "quick", "the"]
puts count_frequency(words).inspect
# => {"the"=>3, "quick"=>2, "brown"=>1, "fox"=>1}

# Pattern 2: Grouping
def group_by_first_letter(words)
  words.group_by { |word| word[0].upcase }
end

fruits = ["apple", "banana", "avocado", "blueberry", "cherry"]
puts group_by_first_letter(fruits).inspect
# => {"A"=>["apple", "avocado"], "B"=>["banana", "blueberry"], "C"=>["cherry"]}

# Pattern 3: Building lookup table
def build_lookup(items, key_method)
  items.each_with_object({}) do |item, hash|
    hash[item.send(key_method)] = item
  end
end

# Pattern 4: Config with defaults
DEFAULT_CONFIG = {
  host: "localhost",
  port: 3000,
  debug: false,
  timeout: 30
}.freeze

def start_server(options = {})
  config = DEFAULT_CONFIG.merge(options)
  puts "Starting server on #{config[:host]}:#{config[:port]}"
  puts "Debug: #{config[:debug]}"
end

start_server
start_server(port: 8080, debug: true)

# Pattern 5: Conditional Hash building
def build_query(filters = {})
  query = {}
  query[:name] = filters[:name] if filters[:name]
  query[:status] = filters[:status] if filters[:status]
  query[:limit] = filters[:limit] || 10
  query
end

puts build_query(name: "Alice", limit: 5).inspect
# => {:name=>"Alice", :limit=>5}
```

---

## ขั้นตอนที่ 99: reduce กับ Hash

```ruby
# reduce / inject กับ Hash
orders = [
  { product: "apple", qty: 3, price: 20 },
  { product: "banana", qty: 5, price: 15 },
  { product: "orange", qty: 2, price: 25 }
]

# คำนวณยอดรวม
total = orders.reduce(0) { |sum, order| sum + (order[:qty] * order[:price]) }
puts total   # => 175

# สร้าง Hash สรุปจาก Array of Hashes
summary = orders.reduce({}) do |hash, order|
  hash[order[:product]] = order[:qty] * order[:price]
  hash
end
puts summary.inspect
# => {"apple"=>60, "banana"=>75, "orange"=>50}

# each_with_object ดีกว่าสำหรับ Hash building
summary2 = orders.each_with_object({}) do |order, hash|
  hash[order[:product]] = order[:qty] * order[:price]
end
puts summary2.inspect

# ตัวอย่างที่ซับซ้อนกว่า: nested aggregation
sales_data = [
  { region: "เหนือ", month: "ม.ค.", amount: 50000 },
  { region: "ใต้", month: "ม.ค.", amount: 30000 },
  { region: "เหนือ", month: "ก.พ.", amount: 60000 },
  { region: "ใต้", month: "ก.พ.", amount: 35000 }
]

by_region = sales_data.each_with_object(Hash.new(0)) do |sale, hash|
  hash[sale[:region]] += sale[:amount]
end
puts by_region.inspect
# => {"เหนือ"=>110000, "ใต้"=>65000}
```

---

## ขั้นตอนที่ 100: Hash Performance และ Best Practices

```ruby
# 1. ใช้ Symbol keys สำหรับ internal data (เร็วกว่า String keys)
# Good
user = { name: "Alice", age: 30 }

# ไม่ดีเท่า (สำหรับ internal data)
user_str = { "name" => "Alice", "age" => 30 }

# 2. freeze Hashes ที่ไม่ต้องการเปลี่ยนแปลง (constants)
COLORS = {
  red: "#FF0000",
  green: "#00FF00",
  blue: "#0000FF"
}.freeze

# COLORS[:yellow] = "#FFFF00"   # FrozenError!

# 3. ใช้ Hash.new(default) สำหรับ counting/accumulating
counter = Hash.new(0)
["a", "b", "a", "c", "b", "a"].each { |x| counter[x] += 1 }

# 4. ใช้ fetch สำหรับ required keys
def process(options)
  host = options.fetch(:host)   # raise KeyError ถ้าไม่มี :host
  port = options.fetch(:port, 80)
  "#{host}:#{port}"
end

# 5. slice เพื่อดึง subset
def create_user(params)
  allowed = params.slice(:name, :email, :age)
  User.new(allowed)
end

# 6. dig สำหรับ nested access (ปลอดภัย)
config = { db: { primary: { host: "db1.local" } } }
host = config.dig(:db, :primary, :host)   # ปลอดภัยกว่า config[:db][:primary][:host]

# 7. compact - ลบ nil values
data = { name: "Alice", age: nil, city: "Bangkok", phone: nil }
clean = data.compact
puts clean.inspect   # => {:name=>"Alice", :city=>"Bangkok"}

# 8. transform_values สำหรับ value transformation
prices_in_usd = { apple: 1.5, banana: 0.8, orange: 2.0 }
exchange_rate = 35
prices_in_thb = prices_in_usd.transform_values { |usd| (usd * exchange_rate).round(2) }
puts prices_in_thb.inspect
# => {:apple=>52.5, :banana=>28.0, :orange=>70.0}
```

---

## แบบฝึกหัด (ขั้นตอนที่ 81-100)

### ข้อที่ 1: สร้าง Hash หนังสือ
สร้าง Hash ที่เก็บข้อมูลหนังสือ (ชื่อ, ผู้แต่ง, ปีพิมพ์, ราคา) แล้วแสดงทุก key-value

```ruby
# เฉลย
book = {
  title: "Ruby Programming",
  author: "Matz",
  year: 2023,
  price: 450
}

book.each { |key, value| puts "#{key}: #{value}" }
```

### ข้อที่ 2: นับคำ
รับ String แล้วนับว่าแต่ละคำปรากฏกี่ครั้ง

```ruby
# เฉลย
def count_words(text)
  text.downcase.split.tally
end

result = count_words("the quick brown fox jumps over the lazy dog the")
puts result.sort_by { |_, count| -count }.first(3).to_h.inspect
# => {"the"=>3, "quick"=>1, "brown"=>1}
```

### ข้อที่ 3: จัดกลุ่มนักเรียน
จัดกลุ่มนักเรียนตามเกรด (A: 90+, B: 80+, C: 70+, F: ต่ำกว่า 70)

```ruby
# เฉลย
def grade_students(students)
  students.group_by do |name, score|
    case score
    when 90..100 then "A"
    when 80..89  then "B"
    when 70..79  then "C"
    else "F"
    end
  end.transform_values { |pairs| pairs.map(&:first) }
end

students = { Alice: 95, Bob: 82, Carol: 71, Dave: 65, Eve: 88 }
result = grade_students(students)
puts result.inspect
```

### ข้อที่ 4: แปลง Inventory
แปลง Array ของ products เป็น Hash โดยใช้ product_id เป็น key

```ruby
# เฉลย
products = [
  { id: "P001", name: "Apple", price: 20 },
  { id: "P002", name: "Banana", price: 15 },
  { id: "P003", name: "Orange", price: 25 }
]

inventory = products.each_with_object({}) do |product, hash|
  hash[product[:id]] = product.except(:id)
end

puts inventory.inspect
# => {"P001"=>{:name=>"Apple", :price=>20}, ...}
```

### ข้อที่ 5: Merge Configs
เขียน method ที่รวม config หลายชั้น โดย deeper level override shallow level

```ruby
# เฉลย
def merge_configs(*configs)
  configs.reduce({}) { |result, config| result.merge(config) }
end

base = { debug: false, port: 3000, host: "localhost" }
env = { port: 8080 }
custom = { debug: true }

final_config = merge_configs(base, env, custom)
puts final_config.inspect
# => {:debug=>true, :port=>8080, :host=>"localhost"}
```

### ข้อที่ 6: Invert และ Find
สร้าง Hash ของ country codes แล้วหาชื่อประเทศจาก code

```ruby
# เฉลย
country_codes = { TH: "ไทย", JP: "ญี่ปุ่น", US: "อเมริกา", UK: "อังกฤษ" }

def find_country(codes_hash, code)
  codes_hash.fetch(code, "ไม่พบ")
end

def find_code(codes_hash, country_name)
  codes_hash.key(country_name) || "ไม่พบ"
end

puts find_country(country_codes, :TH)    # => ไทย
puts find_code(country_codes, "ญี่ปุ่น")  # => JP
```

### ข้อที่ 7: Nested Hash Access
เข้าถึงข้อมูลใน nested hash อย่างปลอดภัย

```ruby
# เฉลย
company = {
  name: "Tech Corp",
  offices: {
    bangkok: { address: "123 Silom", employees: 50 },
    chiang_mai: { address: "456 Nimman", employees: 20 }
  }
}

# ปลอดภัย
puts company.dig(:offices, :bangkok, :employees)       # => 50
puts company.dig(:offices, :phuket, :employees)        # => nil (ไม่ error)

# ดึง office ทั้งหมด
puts company[:offices].map { |city, info| "#{city}: #{info[:employees]} คน" }
```

### ข้อที่ 8: Hash Statistics
คำนวณ max, min, average จาก Hash ของ scores

```ruby
# เฉลย
test_scores = { Alice: 85, Bob: 92, Carol: 78, Dave: 95, Eve: 88 }

stats = {
  max: test_scores.max_by { |_, score| score },
  min: test_scores.min_by { |_, score| score },
  average: (test_scores.values.sum.to_f / test_scores.size).round(2),
  passing: test_scores.count { |_, score| score >= 80 }
}

puts "คะแนนสูงสุด: #{stats[:max][0]} (#{stats[:max][1]})"
puts "คะแนนต่ำสุด: #{stats[:min][0]} (#{stats[:min][1]})"
puts "คะแนนเฉลี่ย: #{stats[:average]}"
puts "จำนวนคนผ่าน: #{stats[:passing]}"
```

### ข้อที่ 9: สร้าง Frequency Table
รับ Array แล้วสร้าง frequency table เรียงจากมากไปน้อย

```ruby
# เฉลย
def frequency_table(items)
  items.tally.sort_by { |_, count| -count }.to_h
end

data = ["red", "blue", "red", "green", "blue", "red", "yellow", "blue"]
table = frequency_table(data)
table.each { |color, count| puts "#{color}: #{count}" }
```

### ข้อที่ 10: Deep Merge
เขียน deep_merge method ที่รวม nested hashes อย่างถูกต้อง

```ruby
# เฉลย
def deep_merge(h1, h2)
  h1.merge(h2) do |key, old, new|
    (old.is_a?(Hash) && new.is_a?(Hash)) ? deep_merge(old, new) : new
  end
end

config1 = { server: { host: "localhost", port: 3000 }, debug: false }
config2 = { server: { port: 4000, ssl: true }, log: "info" }

result = deep_merge(config1, config2)
puts result.inspect
# => {:server=>{:host=>"localhost", :port=>4000, :ssl=>true}, :debug=>false, :log=>"info"}
```

### ข้อที่ 11: Transform Sales Data
แปลง sales data ให้แสดงยอดขายรายภูมิภาค

```ruby
# เฉลย
sales = [
  { region: "เหนือ", product: "ข้าว", amount: 5000 },
  { region: "ใต้", product: "ยาง", amount: 8000 },
  { region: "เหนือ", product: "ผัก", amount: 3000 },
  { region: "กลาง", product: "ผลไม้", amount: 6000 },
  { region: "ใต้", product: "ปลา", amount: 4000 }
]

by_region = sales.each_with_object(Hash.new(0)) do |sale, totals|
  totals[sale[:region]] += sale[:amount]
end

puts by_region.sort_by { |_, total| -total }.to_h.inspect
```

### ข้อที่ 12: Hash Validation
เขียน method ตรวจสอบว่า Hash มี required keys ครบ

```ruby
# เฉลย
def validate_hash(hash, required_keys)
  missing = required_keys - hash.keys
  return { valid: true } if missing.empty?
  { valid: false, missing: missing }
end

required = [:name, :email, :age]
user1 = { name: "Alice", email: "alice@example.com", age: 30 }
user2 = { name: "Bob", email: "bob@example.com" }

puts validate_hash(user1, required).inspect   # => {:valid=>true}
puts validate_hash(user2, required).inspect   # => {:valid=>false, :missing=>[:age]}
```

### ข้อที่ 13: สร้าง Phone Book
สร้าง phone book ที่รองรับการเพิ่ม ค้นหา และลบรายการ

```ruby
# เฉลย
class PhoneBook
  def initialize
    @contacts = {}
  end

  def add(name, phone)
    @contacts[name.downcase] = phone
    puts "เพิ่ม #{name} แล้ว"
  end

  def find(name)
    @contacts[name.downcase] || "ไม่พบ #{name}"
  end

  def delete(name)
    if @contacts.delete(name.downcase)
      puts "ลบ #{name} แล้ว"
    else
      puts "ไม่พบ #{name}"
    end
  end

  def list
    @contacts.each { |name, phone| puts "#{name}: #{phone}" }
  end
end

book = PhoneBook.new
book.add("Alice", "081-111-1111")
book.add("Bob", "082-222-2222")
puts book.find("Alice")
book.delete("Bob")
book.list
```

### ข้อที่ 14: ค้นหา Common Keys
หา keys ที่มีใน hashes ทุกอัน

```ruby
# เฉลย
def common_keys(*hashes)
  hashes.map(&:keys).reduce(:&)
end

h1 = { a: 1, b: 2, c: 3 }
h2 = { b: 20, c: 30, d: 40 }
h3 = { b: 200, c: 300, e: 500 }

puts common_keys(h1, h2, h3).inspect   # => [:b, :c]
```

### ข้อที่ 15: Pivot Table
สร้าง pivot table จาก data

```ruby
# เฉลย
def pivot(data, row_key, col_key, value_key)
  data.each_with_object(Hash.new { |h, k| h[k] = {} }) do |row, table|
    table[row[row_key]][row[col_key]] = row[value_key]
  end
end

sales = [
  { region: "เหนือ", month: "ม.ค.", sales: 100 },
  { region: "เหนือ", month: "ก.พ.", sales: 120 },
  { region: "ใต้", month: "ม.ค.", sales: 80 },
  { region: "ใต้", month: "ก.พ.", sales: 90 }
]

table = pivot(sales, :region, :month, :sales)
table.each do |region, months|
  puts "#{region}: #{months.inspect}"
end
```

### ข้อที่ 16: Flatten Nested Hash
แปลง nested hash ให้เป็น flat hash โดยใช้ dot notation เป็น key

```ruby
# เฉลย
def flatten_hash(hash, prefix = "")
  hash.each_with_object({}) do |(key, value), flat|
    full_key = prefix.empty? ? key.to_s : "#{prefix}.#{key}"
    if value.is_a?(Hash)
      flat.merge!(flatten_hash(value, full_key))
    else
      flat[full_key] = value
    end
  end
end

nested = {
  server: {
    host: "localhost",
    port: 3000,
    ssl: { enabled: true, cert: "cert.pem" }
  },
  debug: false
}

puts flatten_hash(nested).inspect
# => {"server.host"=>"localhost", "server.port"=>3000, "server.ssl.enabled"=>true, ...}
```

### ข้อที่ 17: สร้าง Cache
สร้าง simple cache ด้วย Hash

```ruby
# เฉลย
class SimpleCache
  def initialize(max_size = 100)
    @cache = {}
    @max_size = max_size
  end

  def fetch(key, &block)
    return @cache[key] if @cache.key?(key)

    value = block.call
    @cache.delete(@cache.keys.first) if @cache.size >= @max_size
    @cache[key] = value
  end

  def clear
    @cache.clear
  end

  def size
    @cache.size
  end
end

cache = SimpleCache.new(3)
result1 = cache.fetch("user_1") { "Alice" }
result2 = cache.fetch("user_2") { "Bob" }
result3 = cache.fetch("user_1") { "Should not call this" }  # cached!

puts result1   # => Alice
puts result3   # => Alice
puts cache.size  # => 2
```

### ข้อที่ 18: Score Aggregator
รวมคะแนนจากหลาย rounds

```ruby
# เฉลย
def aggregate_scores(*rounds)
  rounds.each_with_object(Hash.new(0)) do |round, totals|
    round.each { |player, score| totals[player] += score }
  end.sort_by { |_, total| -total }.to_h
end

round1 = { Alice: 85, Bob: 90, Carol: 78 }
round2 = { Alice: 92, Bob: 88, Carol: 95 }
round3 = { Alice: 79, Bob: 94, Carol: 82 }

final = aggregate_scores(round1, round2, round3)
final.each_with_index do |(player, total), rank|
  puts "อันดับ #{rank + 1}: #{player} - #{total} คะแนน"
end
```

### ข้อที่ 19: Word Frequency Analyzer
วิเคราะห์ความถี่คำใน text โดยละเว้น stop words

```ruby
# เฉลย
STOP_WORDS = %w[the a an is are was were in on at to of and or but].freeze

def analyze_text(text)
  words = text.downcase.gsub(/[^a-z\s]/, '').split
  meaningful_words = words.reject { |w| STOP_WORDS.include?(w) }
  
  freq = meaningful_words.tally.sort_by { |_, count| -count }.to_h
  
  {
    total_words: words.length,
    unique_words: freq.keys.length,
    top_5: freq.first(5).to_h
  }
end

text = "Ruby is a dynamic programming language with a focus on simplicity and productivity. Ruby was designed to make programmers happy."
result = analyze_text(text)
puts result.inspect
```

### ข้อที่ 20: Config Manager
สร้าง config manager ที่ support environments

```ruby
# เฉลย
class ConfigManager
  DEFAULTS = {
    database: { host: "localhost", port: 5432, pool: 5 },
    cache: { host: "localhost", port: 6379, ttl: 3600 },
    app: { debug: false, log_level: "info" }
  }.freeze

  ENVIRONMENTS = {
    development: {
      database: { name: "myapp_dev" },
      app: { debug: true, log_level: "debug" }
    },
    production: {
      database: { host: "db.prod.com", name: "myapp_prod", pool: 20 },
      cache: { host: "redis.prod.com" },
      app: { log_level: "warn" }
    }
  }.freeze

  def self.for(env)
    env_config = ENVIRONMENTS[env.to_sym] || {}
    deep_merge(DEFAULTS, env_config)
  end

  def self.deep_merge(base, override)
    base.merge(override) do |_, old, new|
      old.is_a?(Hash) && new.is_a?(Hash) ? deep_merge(old, new) : new
    end
  end
end

dev_config = ConfigManager.for(:development)
prod_config = ConfigManager.for(:production)

puts "Dev DB host: #{dev_config.dig(:database, :host)}"    # => localhost
puts "Prod DB host: #{prod_config.dig(:database, :host)}"  # => db.prod.com
puts "Dev debug: #{dev_config.dig(:app, :debug)}"          # => true
puts "Prod debug: #{prod_config.dig(:app, :debug)}"        # => false
```

---

## สรุปบทที่ 6

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Hash ซึ่งเป็นโครงสร้างข้อมูลที่สำคัญมากใน Ruby:

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| การสร้าง | Rocket syntax (`=>`), Symbol shorthand (`:key => val` หรือ `key: val`) |
| การเข้าถึง | `[]`, `fetch`, `dig`, `values_at` |
| การแก้ไข | `[]=`, `store`, `delete`, `merge!` |
| iteration | `each`, `map`, `select`, `reject`, `reduce` |
| transformation | `transform_values`, `transform_keys`, `flatten`, `invert` |
| การรวม | `merge`, `merge!`, `**splat` |
| Default values | `Hash.new(default)`, `fetch(key, default)`, `||` |
| Symbol vs String | ต่างกัน! ใช้ `transform_keys` เพื่อแปลง |
| Nested | `dig`, deep_merge |
| Performance | Symbol keys เร็วกว่า, ใช้ `freeze` สำหรับ constants |

**Key Takeaways:**
1. Symbol keys (`:key`) ดีกว่า String keys (`"key"`) สำหรับ internal data
2. ใช้ `fetch` แทน `[]` เมื่อ key ต้องมีอยู่เสมอ
3. ใช้ `dig` สำหรับ nested hash access อย่างปลอดภัย
4. `transform_values` และ `transform_keys` มีประโยชน์มากสำหรับ transformation
5. `Hash.new(0)` หรือ `Hash.new([])` ช่วยในการ counting/grouping

---

*ถัดไป: ตอนที่ 7 - Control Flow (ขั้นตอนที่ 101-120)*
