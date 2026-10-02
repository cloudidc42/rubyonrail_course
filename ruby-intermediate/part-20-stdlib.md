# ตอนที่ 20: Ruby Standard Library (Steps 411-435)

## บทนำ

Ruby มาพร้อมกับ **Standard Library** ที่ครอบคลุมและมีประโยชน์มาก ซึ่งรวมถึง classes และ modules สำหรับงานทั่วไปต่างๆ ตั้งแต่การจัดการวันที่/เวลา การคำนวณทางคณิตศาสตร์ การทำงานกับ network ไปจนถึงการทำ benchmarking

---

## Step 411: Date, Time, DateTime Classes

### Date Class

```ruby
require 'date'

# สร้าง Date
today = Date.today
puts today                     # 2024-01-15 (ขึ้นอยู่กับวันปัจจุบัน)
puts today.class               # Date

# สร้างด้วย arguments
specific = Date.new(2024, 12, 25)
puts specific                  # 2024-12-25

# สร้างจาก String
from_string = Date.parse("2024-01-15")
puts from_string               # 2024-01-15

# Various formats
puts Date.parse("15/01/2024")  # ปัญหา! อาจ parse ผิด
puts Date.strptime("15/01/2024", "%d/%m/%Y")  # ถูกต้อง

# Date properties
d = Date.new(2024, 3, 15)
puts d.year       # 2024
puts d.month      # 3
puts d.day        # 15
puts d.wday       # วันในสัปดาห์ (0=Sun, 1=Mon, ..., 6=Sat)
puts d.yday       # วันที่ในปี (1-366)
puts d.cwday      # วันใน ISO week (1=Mon, ..., 7=Sun)
puts d.cweek      # ISO week number
puts d.cwyear     # ISO week year

# Day of week methods
puts d.monday?    # false
puts d.friday?    # true

# Arithmetic
tomorrow = today + 1
yesterday = today - 1
next_week = today + 7
next_month = today >> 1   # เพิ่ม 1 เดือน
last_month = today << 1   # ลด 1 เดือน
next_year = today >> 12

puts "Tomorrow: #{tomorrow}"
puts "Next week: #{next_week}"
puts "Next month: #{next_month}"

# Days difference
d1 = Date.new(2024, 1, 1)
d2 = Date.new(2024, 12, 31)
puts "Days in 2024: #{d2 - d1 + 1}"  # 366 (leap year)

# Formatting
d = Date.new(2024, 3, 15)
puts d.strftime("%Y-%m-%d")     # 2024-03-15
puts d.strftime("%d/%m/%Y")     # 15/03/2024
puts d.strftime("%B %d, %Y")    # March 15, 2024
puts d.strftime("%A, %B %d")    # Friday, March 15
puts d.strftime("%b %d '%y")    # Mar 15 '24

# Date ranges
jan = Date.new(2024, 1, 1)..Date.new(2024, 1, 31)
weekdays_in_jan = jan.select { |d| d.wday.between?(1, 5) }
puts "Weekdays in January 2024: #{weekdays_in_jan.count}"
```

### Time Class

```ruby
# Time - แม่นยำถึง nanosecond
now = Time.now
puts now                     # 2024-01-15 10:30:45 +0700
puts now.class               # Time

# สร้าง Time
t = Time.new(2024, 3, 15, 10, 30, 0)
puts t                       # 2024-03-15 10:30:00 +0700

# UTC
utc_now = Time.now.utc
puts utc_now
puts utc_now.utc?            # true

# Unix timestamp
timestamp = Time.now.to_i
puts "Unix timestamp: #{timestamp}"
puts Time.at(timestamp)      # แปลงกลับ

# Time properties
t = Time.now
puts t.year    # ปี
puts t.month   # เดือน
puts t.day     # วัน
puts t.hour    # ชั่วโมง
puts t.min     # นาที
puts t.sec     # วินาที
puts t.nsec    # nanosecond
puts t.wday    # day of week
puts t.zone    # timezone string

# Arithmetic
future = Time.now + 3600     # 1 ชั่วโมงข้างหน้า
past   = Time.now - 86400    # 1 วันที่แล้ว
puts "1 hour later: #{future}"
puts "Yesterday: #{past}"

# Difference in seconds
t1 = Time.new(2024, 1, 1, 0, 0, 0)
t2 = Time.new(2024, 1, 2, 12, 0, 0)
diff_seconds = (t2 - t1).to_i
puts "Difference: #{diff_seconds} seconds"
puts "= #{diff_seconds / 3600} hours"
puts "= #{diff_seconds / 86400} days"

# Formatting
puts t.strftime("%Y-%m-%d %H:%M:%S")   # 2024-01-15 10:30:45
puts t.strftime("%I:%M %p")             # 10:30 AM
puts t.strftime("%d %b %Y at %H:%M")   # 15 Jan 2024 at 10:30
```

### DateTime Class

```ruby
require 'date'

# DateTime รวม Date และ Time
dt = DateTime.now
puts dt
puts dt.class   # DateTime

# สร้าง DateTime
dt = DateTime.new(2024, 3, 15, 10, 30, 45)
puts dt

# Convert ระหว่าง Date, Time, DateTime
date = Date.today
time = Time.now
datetime = DateTime.now

# Date -> DateTime
d_to_dt = date.to_datetime
puts d_to_dt

# Time -> DateTime
t_to_dt = time.to_datetime
puts t_to_dt

# DateTime -> Time
dt_to_t = datetime.to_time
puts dt_to_t

# ตัวอย่างจริง: คำนวณอายุ
def calculate_age(birthdate_str)
  birth = Date.parse(birthdate_str)
  today = Date.today
  age = today.year - birth.year
  age -= 1 if today < Date.new(today.year, birth.month, birth.day)
  age
end

puts "Age: #{calculate_age("1990-05-15")} years"

# Business days calculation
def business_days_between(start_date, end_date)
  (start_date..end_date).count { |d| d.wday.between?(1, 5) }
end

d1 = Date.new(2024, 1, 1)
d2 = Date.new(2024, 1, 31)
puts "Business days in January: #{business_days_between(d1, d2)}"
```

---

## Step 412: Math Module

```ruby
# Math module สำหรับ mathematical functions
include Math  # หรือใช้ Math::PI เป็น prefix

# Constants
puts Math::PI    # 3.141592653589793
puts Math::E     # 2.718281828459045

# Square root
puts Math.sqrt(16)    # 4.0
puts Math.sqrt(2)     # 1.4142135623730951

# Power/Exponents
puts Math.exp(1)      # e^1 = 2.718...
puts Math.exp(2)      # e^2 = 7.389...
puts 2 ** 10          # 1024 (Ruby power operator)

# Logarithms
puts Math.log(Math::E)   # 1.0 (natural log)
puts Math.log(100, 10)   # 2.0 (log base 10)
puts Math.log2(8)         # 3.0 (log base 2)
puts Math.log10(1000)     # 3.0

# Trigonometry (input in radians)
puts Math.sin(0)                # 0.0
puts Math.sin(Math::PI / 2)     # 1.0
puts Math.cos(0)                # 1.0
puts Math.cos(Math::PI)         # -1.0
puts Math.tan(Math::PI / 4)     # ~1.0

# Convert degrees to radians
def degrees_to_radians(degrees)
  degrees * Math::PI / 180
end

puts Math.sin(degrees_to_radians(30))  # 0.5
puts Math.cos(degrees_to_radians(60))  # 0.5

# Inverse trig
puts Math.asin(1)   # PI/2
puts Math.acos(1)   # 0
puts Math.atan(1)   # PI/4
puts Math.atan2(1, 1)  # PI/4

# Hyperbolic
puts Math.sinh(1)   # 1.175...
puts Math.cosh(1)   # 1.543...
puts Math.tanh(1)   # 0.761...

# Round, ceil, floor (ใช้ built-in numeric methods)
puts 3.7.round    # 4
puts 3.7.ceil     # 4
puts 3.7.floor    # 3
puts 3.7.truncate # 3

# GCD, LCM
puts 12.gcd(8)    # 4
puts 12.lcm(8)    # 24

# ตัวอย่างจริง: Distance calculation (Haversine formula)
def haversine_distance(lat1, lon1, lat2, lon2)
  r = 6371  # Earth radius in km
  phi1 = degrees_to_radians(lat1)
  phi2 = degrees_to_radians(lat2)
  dphi = degrees_to_radians(lat2 - lat1)
  dlambda = degrees_to_radians(lon2 - lon1)

  a = Math.sin(dphi/2)**2 + Math.cos(phi1) * Math.cos(phi2) * Math.sin(dlambda/2)**2
  c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a))
  r * c
end

# Bangkok to Chiang Mai
dist = haversine_distance(13.7563, 100.5018, 18.7883, 98.9853)
puts "Bangkok to Chiang Mai: #{dist.round(0)} km"  # ~687 km
```

---

## Step 413: Set Class

```ruby
require 'set'

# สร้าง Set
numbers = Set.new([1, 2, 3, 4, 5])
puts numbers.inspect  # #<Set: {1, 2, 3, 4, 5}>

# Set ไม่มี duplicates
with_dups = Set.new([1, 2, 2, 3, 3, 3])
puts with_dups.inspect  # #<Set: {1, 2, 3}>

# Add elements
numbers.add(6)
numbers << 7
puts numbers.inspect

# Delete elements
numbers.delete(1)
numbers.discard(99)  # ไม่ error ถ้าไม่มี

# Membership check (O(1) vs Array O(n))
puts numbers.include?(3)  # true
puts numbers.member?(99)  # false

# Set operations
a = Set.new([1, 2, 3, 4, 5])
b = Set.new([3, 4, 5, 6, 7])

# Union (|)
puts (a | b).inspect  # {1, 2, 3, 4, 5, 6, 7}
puts (a.union(b)).inspect

# Intersection (&)
puts (a & b).inspect  # {3, 4, 5}
puts (a.intersection(b)).inspect

# Difference (-)
puts (a - b).inspect  # {1, 2}
puts (a.difference(b)).inspect

# Symmetric difference (^)
puts (a ^ b).inspect  # {1, 2, 6, 7}

# Subset/Superset
c = Set.new([3, 4])
puts c.subset?(a)    # true
puts a.superset?(c)  # true
puts a.subset?(c)    # false
puts c.proper_subset?(a)  # true (strict subset)

# Disjoint
d = Set.new([8, 9])
puts a.disjoint?(d)  # true (ไม่มี element ร่วมกัน)
puts a.intersect?(b) # true

# Convert
arr = [1, 2, 3, 4, 5]
set = arr.to_set
puts set.class  # Set
puts set.to_a.inspect  # Array

# ตัวอย่างจริง: Unique visitors
class WebAnalytics
  def initialize
    @visitors = Set.new
    @page_views = Hash.new(0)
  end

  def track_visit(user_id, page)
    @visitors.add(user_id)
    @page_views[page] += 1
  end

  def unique_visitor_count
    @visitors.size
  end

  def popular_pages(n = 5)
    @page_views.sort_by { |_, v| -v }.first(n)
  end
end

analytics = WebAnalytics.new
["user1", "user2", "user1", "user3", "user2", "user1"].zip(
  ["/home", "/about", "/home", "/contact", "/home", "/products"]
).each { |user, page| analytics.track_visit(user, page) }

puts "Unique visitors: #{analytics.unique_visitor_count}"
puts "Popular pages: #{analytics.popular_pages.inspect}"
```

---

## Step 414: Queue และ SizedQueue

```ruby
require 'thread'

# Queue - Thread-safe FIFO data structure
q = Queue.new

# Enqueue
q << "first"
q << "second"
q.push("third")
q.enq("fourth")

puts "Size: #{q.size}"
puts "Empty? #{q.empty?}"

# Dequeue (blocks ถ้า empty)
puts q.pop   # "first"
puts q.shift # "second" (alias)
puts q.deq   # "third" (alias)

# Non-blocking dequeue
item = q.pop(true) rescue nil  # returns nil ถ้า empty
puts item.inspect  # "fourth"

item = q.pop(true) rescue nil  # empty queue
puts item.inspect  # nil

# Queue ใน Threading
queue = Queue.new

# Producer thread
producer = Thread.new do
  5.times do |i|
    queue << "item_#{i}"
    puts "Produced: item_#{i}"
    sleep 0.1
  end
end

# Consumer thread
consumer = Thread.new do
  5.times do
    item = queue.pop
    puts "Consumed: #{item}"
    sleep 0.15
  end
end

producer.join
consumer.join

# SizedQueue - Queue ที่มี max size
sq = SizedQueue.new(3)
sq << "a"
sq << "b"
sq << "c"
# sq << "d"  # blocks จนกว่าจะมีที่ว่าง!

puts "SizedQueue size: #{sq.size}"
puts "SizedQueue max: #{sq.max}"

# Non-blocking push
begin
  sq.push("d", true)  # raises ThreadError ถ้า full
rescue ThreadError => e
  puts "Queue full: #{e.message}"
end
```

---

## Step 415: OpenStruct และ Struct เปรียบเทียบ

```ruby
require 'ostruct'

# Struct - fast, memory efficient, compile-time structure
Point = Struct.new(:x, :y) do
  def distance_to(other)
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end

  def to_s
    "(#{x}, #{y})"
  end
end

p1 = Point.new(0, 0)
p2 = Point.new(3, 4)
puts p1
puts p2
puts p1.distance_to(p2)  # 5.0

# Struct members
puts Point.members.inspect  # [:x, :y]

# Struct equality
p3 = Point.new(3, 4)
puts p2 == p3  # true

# Struct as value object
Person = Struct.new(:name, :age, :email) do
  def adult?
    age >= 18
  end

  def greeting
    "Hello, I'm #{name}!"
  end
end

alice = Person.new("Alice", 30, "alice@example.com")
puts alice.name     # Alice
puts alice.adult?   # true
puts alice.greeting # Hello, I'm Alice!
alice.age = 31      # mutable by default

# Keyword init (Ruby 3.2+)
Config = Struct.new(:host, :port, :timeout, keyword_init: true)
config = Config.new(host: "localhost", port: 3000, timeout: 30)
puts config.host  # localhost

# OpenStruct - flexible, dynamic
person = OpenStruct.new(name: "Bob", age: 25)
puts person.name    # Bob
puts person.age     # 25

# เพิ่ม attribute ทีหลังได้
person.email = "bob@example.com"
person.admin = true
puts person.email   # bob@example.com
puts person.admin   # true

# ตรวจสอบ attribute ที่มี
puts person.respond_to?(:name)    # true
puts person.respond_to?(:salary)  # false

# แปลง Hash เป็น OpenStruct
data = { name: "Charlie", city: "Bangkok", score: 95 }
obj = OpenStruct.new(data)
puts obj.name   # Charlie
puts obj.city   # Bangkok

# เปรียบเทียบ Struct vs OpenStruct
# Struct:
#   + เร็วกว่า
#   + ใช้ memory น้อยกว่า
#   + ระบุ members ล่วงหน้า
#   + สนับสนุน == comparison
#   - ไม่ flexible

# OpenStruct:
#   + ยืดหยุ่น เพิ่ม attribute ทีหลังได้
#   + เหมาะสำหรับ prototype
#   - ช้ากว่า (Hash-based)
#   - ไม่ type-safe
```

---

## Step 416: SecureRandom

```ruby
require 'securerandom'

# UUID (Universally Unique Identifier)
uuid = SecureRandom.uuid
puts uuid  # e.g., "550e8400-e29b-41d4-a716-446655440000"
puts uuid.length  # 36

# Hex string
hex_16 = SecureRandom.hex(16)  # 32 character hex string
puts hex_16
puts hex_16.length  # 32

hex_8 = SecureRandom.hex(8)
puts hex_8  # 16 characters

# Base64
b64 = SecureRandom.base64(24)
puts b64   # e.g., "rT3qiuBDEV8bBPCTjgRdDA=="
puts b64.length  # ~32 (base64 overhead)

# URL-safe Base64
url_safe = SecureRandom.urlsafe_base64(16)
puts url_safe  # ไม่มี +/= ที่ทำให้ URL มีปัญหา

# Random bytes
bytes = SecureRandom.random_bytes(16)
puts bytes.bytes.inspect

# Random number
puts SecureRandom.random_number(100)     # 0..99
puts SecureRandom.random_number(1000)    # 0..999

# Alphanumeric string
alpha = SecureRandom.alphanumeric(20)
puts alpha  # ตัวอักษรและตัวเลขแบบ random

# ตัวอย่างจริง: Token generation
class TokenGenerator
  def self.session_token
    SecureRandom.urlsafe_base64(32)
  end

  def self.api_key(prefix = "sk")
    "#{prefix}_#{SecureRandom.hex(20)}"
  end

  def self.otp(digits = 6)
    SecureRandom.random_number(10**digits).to_s.rjust(digits, '0')
  end

  def self.password_reset_token
    SecureRandom.urlsafe_base64(24)
  end
end

puts "Session: #{TokenGenerator.session_token}"
puts "API Key: #{TokenGenerator.api_key}"
puts "OTP: #{TokenGenerator.otp}"
puts "Reset: #{TokenGenerator.password_reset_token}"
```

---

## Step 417: Base64

```ruby
require 'base64'

# Encode
original = "Hello, World! สวัสดีชาวโลก"
encoded = Base64.encode64(original)
puts "Encoded: #{encoded}"

# Decode
decoded = Base64.decode64(encoded)
puts "Decoded: #{decoded}"
puts original == decoded  # true

# Strict encoding (ไม่มี newline)
strict = Base64.strict_encode64(original)
puts "Strict: #{strict}"

strict_decoded = Base64.strict_decode64(strict)
puts strict_decoded

# URL-safe Base64 (ใช้ - แทน + และ _ แทน /)
url_safe = Base64.urlsafe_encode64(original)
puts "URL-safe: #{url_safe}"
puts Base64.urlsafe_decode64(url_safe)

# ตัวอย่างจริง: Encode image
# binary_data = File.read("image.png", mode: "rb")
# encoded_image = Base64.strict_encode64(binary_data)
# puts "data:image/png;base64,#{encoded_image}"

# JWT-like token (simplified)
def create_simple_token(payload)
  header = Base64.urlsafe_encode64('{"alg":"none","typ":"JWT"}').tr('=', '')
  body   = Base64.urlsafe_encode64(payload.to_json).tr('=', '')
  "#{header}.#{body}"
end

require 'json'
token = create_simple_token({ user_id: 1, role: "admin", exp: Time.now.to_i + 3600 })
puts "Token: #{token}"

# Decode
parts = token.split('.')
payload = JSON.parse(Base64.urlsafe_decode64(parts[1] + '=='))
puts "Payload: #{payload.inspect}"
```

---

## Step 418: Digest (MD5, SHA1, SHA256)

```ruby
require 'digest'

message = "Hello, Ruby!"

# MD5 (ไม่แนะนำสำหรับ security)
md5 = Digest::MD5.hexdigest(message)
puts "MD5: #{md5}"
puts "MD5 length: #{md5.length}"  # 32

# SHA1 (ไม่แนะนำสำหรับ security)
sha1 = Digest::SHA1.hexdigest(message)
puts "SHA1: #{sha1}"
puts "SHA1 length: #{sha1.length}"  # 40

# SHA256 (แนะนำ)
sha256 = Digest::SHA256.hexdigest(message)
puts "SHA256: #{sha256}"
puts "SHA256 length: #{sha256.length}"  # 64

# SHA512
sha512 = Digest::SHA512.hexdigest(message)
puts "SHA512 length: #{sha512.length}"  # 128

# Digest object (incremental)
d = Digest::SHA256.new
d.update("Hello")
d.update(", ")
d.update("Ruby!")
puts d.hexdigest  # เหมือนกับ SHA256.hexdigest("Hello, Ruby!")

# Reset และใช้ใหม่
d.reset
d << "Different message"
puts d.hexdigest

# Binary digest
binary = Digest::SHA256.digest(message)
puts "Binary length: #{binary.length}"  # 32 bytes

# File digest
# file_hash = Digest::MD5.file("important.txt").hexdigest

# ตัวอย่างจริง: Password hashing (ใช้ bcrypt ในโปรแกรมจริง แต่นี่เป็น demo)
def simple_hash_password(password, salt = nil)
  salt ||= SecureRandom.hex(16)
  hash = Digest::SHA256.hexdigest("#{salt}:#{password}")
  "#{salt}:#{hash}"
end

def verify_password(password, stored_hash)
  salt, hash = stored_hash.split(':')
  Digest::SHA256.hexdigest("#{salt}:#{password}") == hash
end

require 'securerandom'
stored = simple_hash_password("mypassword")
puts "Stored: #{stored}"
puts verify_password("mypassword", stored)    # true
puts verify_password("wrongpassword", stored) # false

# Checksum สำหรับ file integrity
def file_checksum(content)
  Digest::SHA256.hexdigest(content)
end

data = "Important data content"
checksum = file_checksum(data)
puts "Checksum: #{checksum}"
puts "Integrity OK: #{file_checksum(data) == checksum}"  # true
puts "Integrity OK: #{file_checksum(data + "!") == checksum}"  # false
```

---

## Step 419: URI Module

```ruby
require 'uri'

# Parse URI
url = URI.parse("https://www.example.com:8080/path/to/page?name=Alice&age=30#section1")
puts url.scheme    # "https"
puts url.host      # "www.example.com"
puts url.port      # 8080
puts url.path      # "/path/to/page"
puts url.query     # "name=Alice&age=30"
puts url.fragment  # "section1"
puts url.userinfo  # nil (no auth)

# Extract query parameters
params = URI.decode_www_form(url.query).to_h
puts params.inspect  # {"name"=>"Alice", "age"=>"30"}

# Build URI
uri = URI::HTTP.build(
  host:  "api.example.com",
  path:  "/v1/users",
  query: URI.encode_www_form({ page: 1, per_page: 20 })
)
puts uri.to_s  # http://api.example.com/v1/users?page=1&per_page=20

# HTTPS
uri = URI::HTTPS.build(host: "api.example.com", path: "/users")
puts uri.to_s

# Encode/Decode
raw = "Hello World! สวัสดี"
encoded = URI.encode_www_form_component(raw)
puts encoded  # Hello+World%21+%E0%B8%AA...

decoded = URI.decode_www_form_component(encoded)
puts decoded  # Hello World! สวัสดี

# ตัวอย่างจริง: URL builder
class ApiUrlBuilder
  def initialize(base_url)
    @base = URI.parse(base_url)
  end

  def build(path, params = {})
    uri = @base.dup
    uri.path = path
    uri.query = URI.encode_www_form(params) unless params.empty?
    uri.to_s
  end
end

builder = ApiUrlBuilder.new("https://api.example.com")
puts builder.build("/users")
puts builder.build("/users", { page: 2, per_page: 10, sort: "name" })
puts builder.build("/users/search", { q: "Alice", role: "admin" })
```

---

## Step 420: Net::HTTP

```ruby
require 'net/http'
require 'uri'
require 'json'

# GET request
def http_get(url)
  uri = URI.parse(url)
  response = Net::HTTP.get_response(uri)
  {
    status: response.code.to_i,
    body: response.body,
    headers: response.to_hash
  }
end

# ตัวอย่าง (ต้องมี internet connection)
# result = http_get("https://jsonplaceholder.typicode.com/posts/1")
# puts result[:status]
# puts JSON.parse(result[:body])

# POST request
def http_post(url, data)
  uri = URI.parse(url)
  http = Net::HTTP.new(uri.host, uri.port)
  http.use_ssl = (uri.scheme == 'https')

  request = Net::HTTP::Post.new(uri.path)
  request.content_type = "application/json"
  request.body = data.to_json

  response = http.request(request)
  {
    status: response.code.to_i,
    body: JSON.parse(response.body)
  }
rescue => e
  { error: e.message }
end

# Advanced: HTTP client class
class HttpClient
  def initialize(base_url, headers = {})
    @base_uri = URI.parse(base_url)
    @default_headers = {
      "Content-Type" => "application/json",
      "Accept" => "application/json"
    }.merge(headers)
  end

  def get(path, params = {})
    uri = build_uri(path, params)
    request = Net::HTTP::Get.new(uri)
    execute(request)
  end

  def post(path, body = {})
    uri = build_uri(path)
    request = Net::HTTP::Post.new(uri)
    request.body = body.to_json
    execute(request)
  end

  def put(path, body = {})
    uri = build_uri(path)
    request = Net::HTTP::Put.new(uri)
    request.body = body.to_json
    execute(request)
  end

  def delete(path)
    uri = build_uri(path)
    request = Net::HTTP::Delete.new(uri)
    execute(request)
  end

  private

  def build_uri(path, params = {})
    uri = @base_uri.dup
    uri.path = path
    uri.query = URI.encode_www_form(params) unless params.empty?
    uri
  end

  def execute(request)
    @default_headers.each { |k, v| request[k] = v }

    http = Net::HTTP.new(@base_uri.host, @base_uri.port)
    http.use_ssl = (@base_uri.scheme == 'https')
    http.read_timeout = 10

    response = http.request(request)
    {
      status: response.code.to_i,
      ok: response.code.to_i.between?(200, 299),
      body: parse_body(response.body),
      headers: response.to_hash
    }
  rescue Net::TimeoutError
    { status: 408, ok: false, error: "Request timeout" }
  rescue => e
    { status: 0, ok: false, error: e.message }
  end

  def parse_body(body)
    return nil if body.nil? || body.empty?
    JSON.parse(body)
  rescue JSON::ParserError
    body
  end
end

# ใช้งาน
# client = HttpClient.new("https://jsonplaceholder.typicode.com")
# response = client.get("/posts", { userId: 1 })
# puts response[:status]
# puts response[:body].length
```

---

## Step 421: JSON

```ruby
require 'json'

# Parse JSON string เป็น Ruby object
json_str = '{"name":"Alice","age":30,"hobbies":["reading","coding"],"address":{"city":"Bangkok","country":"Thailand"}}'

data = JSON.parse(json_str)
puts data["name"]                  # Alice
puts data["age"]                   # 30
puts data["hobbies"].inspect       # ["reading", "coding"]
puts data["address"]["city"]       # Bangkok

# Parse กับ symbolize_names
sym_data = JSON.parse(json_str, symbolize_names: true)
puts sym_data[:name]               # Alice
puts sym_data[:address][:city]     # Bangkok

# Generate JSON จาก Ruby object
person = {
  name: "Bob",
  age: 25,
  scores: [90, 85, 92],
  metadata: { created_at: Time.now.to_s }
}

json = JSON.generate(person)
puts json

# Pretty print
pretty = JSON.pretty_generate(person)
puts pretty

# Compact (no whitespace)
compact = person.to_json
puts compact

# JSON ที่รองรับ custom types
class User
  attr_reader :name, :email, :created_at

  def initialize(name, email)
    @name       = name
    @email      = email
    @created_at = Time.now
  end

  def as_json
    {
      name: @name,
      email: @email,
      created_at: @created_at.iso8601
    }
  end

  def to_json(*args)
    as_json.to_json(*args)
  end
end

user = User.new("Alice", "alice@example.com")
puts user.to_json
puts JSON.pretty_generate(JSON.parse(user.to_json))

# Parse with error handling
def safe_parse(json_str)
  JSON.parse(json_str)
rescue JSON::ParserError => e
  puts "JSON parse error: #{e.message}"
  nil
end

puts safe_parse('{"valid": true}').inspect
puts safe_parse('invalid json').inspect
```

---

## Step 422: CSV

```ruby
require 'csv'

# อ่าน CSV จาก string
csv_data = <<~CSV
  name,age,city,email
  Alice,30,Bangkok,alice@example.com
  Bob,25,Chiang Mai,bob@example.com
  Charlie,35,Phuket,charlie@example.com
CSV

# Parse CSV
rows = CSV.parse(csv_data, headers: true)
rows.each do |row|
  puts "#{row['name']} (#{row['age']}) - #{row['city']}"
end

# Access by header
rows.each do |row|
  puts row.to_h.inspect
end

# เขียน CSV
CSV.generate do |csv|
  csv << ["Name", "Score", "Grade"]
  csv << ["Alice", 95, "A"]
  csv << ["Bob", 78, "B"]
  csv << ["Charlie", 85, "B"]
end

# เขียนไฟล์ (ตัวอย่าง)
# CSV.open("output.csv", "w") do |csv|
#   csv << ["header1", "header2"]
#   csv << ["data1", "data2"]
# end

# อ่านจาก string ที่ซับซ้อนกว่า
complex_csv = '"Alice Smith","30","Bangkok, Thailand","alice@example.com"'
row = CSV.parse_line(complex_csv)
puts row.inspect
# ["Alice Smith", "30", "Bangkok, Thailand", "alice@example.com"]

# สร้าง CSV กับ options
data = [
  { name: "Alice", score: 95 },
  { name: "Bob", score: 78 }
]

output = CSV.generate(headers: true) do |csv|
  csv << data.first.keys
  data.each { |row| csv << row.values }
end
puts output

# ตัวอย่างจริง: แปลง array of hashes เป็น CSV
class CsvExporter
  def self.export(data, headers: nil)
    return "" if data.empty?
    headers ||= data.first.keys

    CSV.generate do |csv|
      csv << headers
      data.each { |row| csv << headers.map { |h| row[h] } }
    end
  end

  def self.import(csv_string, symbolize: false)
    CSV.parse(csv_string, headers: true).map do |row|
      if symbolize
        row.to_h.transform_keys(&:to_sym)
      else
        row.to_h
      end
    end
  end
end

users = [
  { name: "Alice", age: 30, city: "Bangkok" },
  { name: "Bob",   age: 25, city: "Chiang Mai" }
]

csv_output = CsvExporter.export(users)
puts csv_output

imported = CsvExporter.import(csv_output, symbolize: true)
puts imported.inspect
```

---

## Step 423: Benchmark

```ruby
require 'benchmark'

# วัดเวลา code block
time = Benchmark.measure do
  (1..1_000_000).sum
end
puts time
# => 0.050000   0.000000   0.050000 (  0.051234)
# user CPU time / system CPU time / total / real time

# bm - multiple benchmarks
n = 1_000_000
Benchmark.bm do |x|
  x.report("Array.new(n):") { Array.new(n) }
  x.report("Array.new block:") { Array.new(n) { |i| i } }
  x.report("(0...n).to_a:") { (0...n).to_a }
end

# bmbm - ทำ rehearsal + real benchmark
puts "\nBenchmark with rehearsal:"
Benchmark.bmbm(15) do |x|
  x.report("Symbol compare:") { n.times { :hello == :hello } }
  x.report("String compare:") { n.times { "hello" == "hello" } }
end

# realtime - วัดเฉพาะ wall clock time
elapsed = Benchmark.realtime do
  sleep 0.1
end
puts "Real time: #{elapsed.round(3)}s"

# ตัวอย่าง: เปรียบเทียบ algorithms
def bubble_sort(arr)
  arr = arr.dup
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

data = Array.new(1000) { rand(10000) }

Benchmark.bm(15) do |x|
  x.report("Ruby sort:") { data.sort }
  x.report("Bubble sort:") { bubble_sort(data) }
end

# measure + report
puts "\nSingle measurement:"
puts Benchmark.measure { (1..100000).map { |n| n ** 2 }.sum }
```

---

## Step 424: ObjectSpace

```ruby
require 'objspace'

# นับ objects ทุกประเภท
count = ObjectSpace.count_objects
puts count.inspect
# {:TOTAL=>..., :FREE=>..., :T_OBJECT=>..., :T_CLASS=>..., ...}

# วนซ้ำ objects ทั้งหมด (ระวัง! ช้ามาก)
string_count = 0
ObjectSpace.each_object(String) { |_| string_count += 1 }
puts "Total Strings: #{string_count}"

# นับเฉพาะ type
puts "Classes: #{ObjectSpace.each_object(Class).count}"
puts "Modules: #{ObjectSpace.each_object(Module).count}"

# ขนาดของ object
str = "Hello, World!"
puts ObjectSpace.memsize_of(str)  # bytes

arr = [1, 2, 3, 4, 5]
puts ObjectSpace.memsize_of(arr)

hash = { a: 1, b: 2, c: 3 }
puts ObjectSpace.memsize_of(hash)

# trace object lifecycle
ObjectSpace.trace_object_allocations_start

def create_objects
  strs = (1..100).map { |i| "string_#{i}" }
  strs
end

objects = create_objects
ObjectSpace.trace_object_allocations_stop

# allocation info
objects.first(3).each do |obj|
  source = ObjectSpace.allocation_sourcefile(obj)
  line   = ObjectSpace.allocation_sourceline(obj)
  puts "Allocated at #{source}:#{line}"
end

# ทำ GC และดูผล
before = GC.stat[:heap_live_slots]
GC.start
after = GC.stat[:heap_live_slots]
puts "Objects freed: #{before - after}"

# ตัวอย่างจริง: Memory profiler ง่ายๆ
class MemoryProfiler
  def self.profile(name = "block", &block)
    before = GC.stat[:heap_allocated_pages]
    time_before = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    result = block.call

    time_after = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    after = GC.stat[:heap_allocated_pages]

    elapsed = ((time_after - time_before) * 1000).round(2)
    pages_used = after - before

    puts "[#{name}] Time: #{elapsed}ms, Heap pages allocated: #{pages_used}"
    result
  end
end

MemoryProfiler.profile("Array creation") do
  Array.new(10_000) { |i| { id: i, value: i * 2 } }
end

MemoryProfiler.profile("String concat") do
  result = ""
  10_000.times { |i| result += i.to_s }
  result
end

MemoryProfiler.profile("Array join (efficient)") do
  parts = []
  10_000.times { |i| parts << i.to_s }
  parts.join
end
```

---

## Step 425: ตัวอย่างการใช้ Standard Library รวมกัน

```ruby
# ตัวอย่าง: Web scraper simulation
require 'uri'
require 'json'
require 'csv'
require 'digest'
require 'date'

class DataProcessor
  def initialize
    @cache = {}
    @processed_count = 0
  end

  def process_api_response(json_string)
    # Parse JSON
    data = JSON.parse(json_string, symbolize_names: true)

    # Validate and transform
    data.map do |record|
      next unless record[:id] && record[:name]

      {
        id:         record[:id],
        name:       record[:name].strip,
        email:      record[:email]&.downcase,
        created_at: parse_date(record[:created_at]),
        checksum:   Digest::MD5.hexdigest(record.to_json)
      }
    end.compact
  end

  def export_to_csv(records)
    CSV.generate(headers: true) do |csv|
      csv << ["ID", "Name", "Email", "Created At", "Checksum"]
      records.each do |r|
        csv << [r[:id], r[:name], r[:email], r[:created_at], r[:checksum]]
      end
    end
  end

  def generate_report(records)
    {
      total: records.count,
      date_range: {
        earliest: records.map { |r| r[:created_at] }.min,
        latest:   records.map { |r| r[:created_at] }.max
      },
      domains: records.group_by { |r| r[:email]&.split('@')&.last }
                      .transform_values(&:count)
                      .sort_by { |_, v| -v }
                      .first(5)
    }
  end

  private

  def parse_date(date_str)
    return nil unless date_str
    Date.parse(date_str) rescue nil
  end
end

# Test data
api_data = JSON.generate([
  { id: 1, name: " Alice Smith ", email: "ALICE@GMAIL.COM", created_at: "2024-01-15" },
  { id: 2, name: "Bob Jones",     email: "bob@yahoo.com",  created_at: "2024-02-20" },
  { id: 3, name: "Charlie Brown", email: "charlie@gmail.com", created_at: "2024-03-01" }
])

processor = DataProcessor.new
records = processor.process_api_response(api_data)
puts "Processed #{records.count} records"

csv_output = processor.export_to_csv(records)
puts "\nCSV Output:"
puts csv_output

report = processor.generate_report(records)
puts "\nReport:"
puts JSON.pretty_generate(report)
```

---

## แบบฝึกหัด: Ruby Standard Library (25 ข้อ)

**ข้อ 1:** สร้าง method หา weekdays ระหว่างวันที่สอง

```ruby
# เฉลย
require 'date'

def count_weekdays(start_date, end_date)
  (Date.parse(start_date)..Date.parse(end_date))
    .count { |d| d.wday.between?(1, 5) }
end

puts count_weekdays("2024-01-01", "2024-01-31")  # 23
```

**ข้อ 2:** คำนวณ compound interest ด้วย Math

```ruby
# เฉลย
def compound_interest(principal, annual_rate, years, n = 12)
  principal * (1 + annual_rate / n) ** (n * years)
end

puts compound_interest(10000, 0.05, 10).round(2)  # 16470.09
```

**ข้อ 3:** ใช้ Set หา common elements ระหว่าง arrays

```ruby
# เฉลย
require 'set'

def common_elements(*arrays)
  arrays.map(&:to_set).reduce(:&).to_a.sort
end

puts common_elements([1,2,3,4], [2,3,4,5], [3,4,5,6]).inspect  # [3, 4]
```

**ข้อ 4:** สร้าง secure random password generator

```ruby
# เฉลย
require 'securerandom'

def generate_password(length = 16, options = {})
  chars = ""
  chars += ('a'..'z').to_a.join unless options[:no_lowercase]
  chars += ('A'..'Z').to_a.join unless options[:no_uppercase]
  chars += ('0'..'9').to_a.join unless options[:no_digits]
  chars += "!@#$%^&*()" unless options[:no_symbols]

  Array.new(length) { chars[SecureRandom.random_number(chars.length)] }.join
end

puts generate_password(16)
puts generate_password(8, no_symbols: true)
```

**ข้อ 5:** Hash ไฟล์ content และตรวจสอบ integrity

```ruby
# เฉลย
require 'digest'

class FileIntegrityChecker
  def initialize
    @checksums = {}
  end

  def register(filename, content)
    @checksums[filename] = Digest::SHA256.hexdigest(content)
  end

  def verify(filename, content)
    stored = @checksums[filename]
    return false unless stored
    Digest::SHA256.hexdigest(content) == stored
  end
end

checker = FileIntegrityChecker.new
checker.register("config.txt", "host=localhost\nport=3000")
puts checker.verify("config.txt", "host=localhost\nport=3000")  # true
puts checker.verify("config.txt", "host=hacked.com\nport=3000") # false
```

**ข้อ 6:** URL parser และ query extractor

```ruby
# เฉลย
require 'uri'

class UrlParser
  def initialize(url)
    @uri = URI.parse(url)
  end

  def params
    return {} unless @uri.query
    URI.decode_www_form(@uri.query).to_h
  end

  def base_url
    "#{@uri.scheme}://#{@uri.host}#{':' + @uri.port.to_s if @uri.port != URI::HTTP::DEFAULT_PORT && @uri.port != URI::HTTPS::DEFAULT_PORT}"
  end

  def path
    @uri.path
  end
end

parser = UrlParser.new("https://example.com/search?q=ruby&page=2&sort=recent")
puts parser.base_url
puts parser.path
puts parser.params.inspect
```

**ข้อ 7:** Benchmark สองวิธีสร้าง Hash

```ruby
# เฉลย
require 'benchmark'

n = 100_000
data = (1..n).to_a

Benchmark.bmbm(20) do |x|
  x.report("each_with_object:") do
    data.each_with_object({}) { |i, h| h[i] = i ** 2 }
  end
  x.report("map + to_h:") do
    data.map { |i| [i, i ** 2] }.to_h
  end
  x.report("zip + to_h:") do
    data.zip(data.map { |i| i ** 2 }).to_h
  end
end
```

**ข้อ 8:** JSON API response processor

```ruby
# เฉลย
require 'json'

def process_users_api(json_str)
  users = JSON.parse(json_str, symbolize_names: true)

  {
    total: users.count,
    active: users.count { |u| u[:active] },
    by_role: users.group_by { |u| u[:role] }.transform_values(&:count),
    average_age: (users.sum { |u| u[:age] }.to_f / users.count).round(1),
    emails: users.map { |u| u[:email] }.compact
  }
end

test_data = [
  { id: 1, name: "Alice", age: 30, role: "admin", active: true, email: "alice@test.com" },
  { id: 2, name: "Bob",   age: 25, role: "user",  active: true, email: "bob@test.com" },
  { id: 3, name: "Charlie", age: 35, role: "user", active: false, email: nil }
].to_json

puts process_users_api(test_data).inspect
```

**ข้อ 9:** CSV importer พร้อม validation

```ruby
# เฉลย
require 'csv'

class CsvImporter
  REQUIRED_HEADERS = %w[name email age].freeze

  def initialize(csv_string)
    @rows = CSV.parse(csv_string, headers: true)
  end

  def valid?
    headers = @rows.headers
    REQUIRED_HEADERS.all? { |h| headers.include?(h) }
  end

  def import
    return { errors: ["Invalid headers"] } unless valid?

    records = []
    errors = []

    @rows.each_with_index do |row, i|
      data = row.to_h
      row_errors = validate_row(data)

      if row_errors.empty?
        records << data.transform_values(&:strip)
      else
        errors << "Row #{i + 2}: #{row_errors.join(', ')}"
      end
    end

    { records: records, errors: errors }
  end

  private

  def validate_row(data)
    errors = []
    errors << "name required" if data["name"].to_s.strip.empty?
    errors << "invalid email" unless data["email"]&.match?(/\A[\w+\-.]+@[a-z\d\-]+\.[a-z]+\z/i)
    errors << "invalid age" unless data["age"]&.match?(/\A\d+\z/)
    errors
  end
end

csv_data = <<~CSV
  name,email,age
  Alice,alice@example.com,30
  ,invalid-email,25
  Charlie,charlie@example.com,abc
  Diana,diana@example.com,28
CSV

importer = CsvImporter.new(csv_data)
result = importer.import
puts "Imported: #{result[:records].count} records"
puts "Errors: #{result[:errors].inspect}"
```

**ข้อ 10-25:** (ตัวอย่างเพิ่มเติมแบบย่อ)

```ruby
# ข้อ 10: Date ค้นหาวันหยุด public holiday ที่ใกล้ที่สุด
require 'date'

def nearest_holiday(from_date = Date.today)
  holidays_2024 = [
    Date.new(2024, 1, 1),
    Date.new(2024, 4, 13),
    Date.new(2024, 4, 14),
    Date.new(2024, 4, 15),
    Date.new(2024, 5, 1),
    Date.new(2024, 12, 25)
  ]

  future = holidays_2024.select { |h| h >= from_date }
  nearest = future.min
  days_away = (nearest - from_date).to_i
  { date: nearest, days_away: days_away }
end

result = nearest_holiday(Date.new(2024, 4, 10))
puts "Nearest holiday: #{result[:date]} (#{result[:days_away]} days away)"

# ข้อ 11: Math - คำนวณ statistics
def descriptive_stats(numbers)
  n    = numbers.size
  mean = numbers.sum.to_f / n
  sorted = numbers.sort
  median = sorted.length.odd? ? sorted[n/2] : (sorted[n/2-1] + sorted[n/2]) / 2.0
  variance = numbers.sum { |x| (x - mean)**2 } / n
  {
    n: n, mean: mean.round(2), median: median,
    std_dev: Math.sqrt(variance).round(2),
    min: sorted.first, max: sorted.last
  }
end

puts descriptive_stats([4, 7, 13, 2, 1, 9, 5, 12, 8, 6]).inspect

# ข้อ 12: SecureRandom - สร้าง invite codes
def generate_invite_code(prefix = "INV")
  code = SecureRandom.alphanumeric(8).upcase
  "#{prefix}-#{code[0..3]}-#{code[4..7]}"
end

5.times { puts generate_password(12) rescue puts generate_invite_code }

# ข้อ 13: Set operations สำหรับ permissions
require 'set'

class PermissionManager
  PERMISSIONS = %i[read write delete admin].freeze

  def initialize
    @role_permissions = {}
  end

  def set_permissions(role, *perms)
    @role_permissions[role] = Set.new(perms)
  end

  def can?(role, permission)
    @role_permissions[role]&.include?(permission) || false
  end

  def merge_permissions(*roles)
    roles.map { |r| @role_permissions[r] || Set.new }.reduce(:+)
  end
end

pm = PermissionManager.new
pm.set_permissions(:viewer, :read)
pm.set_permissions(:editor, :read, :write)
pm.set_permissions(:admin, :read, :write, :delete, :admin)

puts pm.can?(:viewer, :read)    # true
puts pm.can?(:viewer, :write)   # false
puts pm.can?(:editor, :delete)  # false
puts pm.merge_permissions(:viewer, :editor).to_a.sort.inspect

# ข้อ 14: Digest สำหรับ API signature
require 'digest'
require 'base64'

class ApiSigner
  def initialize(secret_key)
    @secret = secret_key
  end

  def sign(payload)
    data = payload.sort.map { |k, v| "#{k}=#{v}" }.join("&")
    signature = Digest::SHA256.hexdigest("#{@secret}:#{data}")
    { payload: payload, signature: signature }
  end

  def verify(payload, signature)
    data = payload.sort.map { |k, v| "#{k}=#{v}" }.join("&")
    expected = Digest::SHA256.hexdigest("#{@secret}:#{data}")
    expected == signature
  end
end

signer = ApiSigner.new("mysecret123")
signed = signer.sign({ user_id: 1, action: "purchase", amount: 100 })
puts "Signature: #{signed[:signature]}"
puts "Valid: #{signer.verify(signed[:payload], signed[:signature])}"
puts "Tampered: #{signer.verify({ user_id: 1, action: "purchase", amount: 999 }, signed[:signature])}"

# ข้อ 15: URI building สำหรับ OAuth
require 'uri'
require 'securerandom'

def build_oauth_url(client_id, redirect_uri, scopes)
  URI::HTTPS.build(
    host: "auth.example.com",
    path: "/oauth/authorize",
    query: URI.encode_www_form(
      client_id:     client_id,
      redirect_uri:  redirect_uri,
      response_type: "code",
      scope:         scopes.join(" "),
      state:         SecureRandom.urlsafe_base64(16)
    )
  ).to_s
end

puts build_oauth_url(
  "my_app",
  "https://myapp.com/callback",
  ["read:user", "write:posts"]
)

# ข้อ 16-25: brief examples
# ข้อ 16: CSV ของนักศึกษาพร้อมคำนวณ GPA
# ข้อ 17: JSON config file reader/writer
# ข้อ 18: Benchmark string operations
# ข้อ 19: Date calculation - working hours
# ข้อ 20: Set ใน tag system
# ข้อ 21: Queue สำหรับ job processing
# ข้อ 22: Struct vs OpenStruct benchmarking
# ข้อ 23: Math - draw sine wave as ASCII
# ข้อ 24: Digest - password strength checker
# ข้อ 25: Full CRUD with JSON + CSV backup

# ข้อ 25 - Full CRUD
class DataStore
  require 'json'

  def initialize(filename)
    @filename = filename
    @data = load_data
    @next_id = (@data.map { |r| r[:id] }.max || 0) + 1
  end

  def create(attrs)
    record = attrs.merge(id: @next_id)
    @next_id += 1
    @data << record
    save_data
    record
  end

  def read(id)
    @data.find { |r| r[:id] == id }
  end

  def update(id, attrs)
    record = read(id)
    return nil unless record
    record.merge!(attrs)
    save_data
    record
  end

  def delete(id)
    record = read(id)
    return nil unless record
    @data.reject! { |r| r[:id] == id }
    save_data
    record
  end

  def all
    @data.dup
  end

  private

  def load_data
    return [] unless File.exist?(@filename)
    JSON.parse(File.read(@filename), symbolize_names: true)
  rescue
    []
  end

  def save_data
    File.write(@filename, JSON.pretty_generate(@data))
  end
end

# ใช้ tempfile สำหรับ demo
require 'tempfile'
tmpfile = Tempfile.new(['data', '.json'])

store = DataStore.new(tmpfile.path)
store.create({ name: "Alice", email: "alice@test.com" })
store.create({ name: "Bob",   email: "bob@test.com" })
puts store.all.inspect
record = store.update(1, { name: "Alice Smith" })
puts "Updated: #{record.inspect}"
store.delete(2)
puts "After delete: #{store.all.inspect}"
tmpfile.close
tmpfile.unlink
```

---

## สรุป

Ruby Standard Library ที่สำคัญ:

| Library | ใช้สำหรับ |
|---------|----------|
| `Date` | จัดการวันที่ (ไม่มีเวลา) |
| `Time` | วันที่และเวลา |
| `DateTime` | วันที่ เวลา และ timezone |
| `Math` | คณิตศาสตร์ขั้นสูง |
| `Set` | ชุดข้อมูล ไม่ซ้ำ, set operations |
| `Queue/SizedQueue` | Thread-safe queue |
| `Struct` | Value objects ที่รวดเร็ว |
| `OpenStruct` | Dynamic objects ที่ยืดหยุ่น |
| `SecureRandom` | Random values สำหรับ security |
| `Base64` | Encode/decode binary data |
| `Digest` | Hash functions (MD5, SHA) |
| `URI` | Parse/build URLs |
| `Net::HTTP` | HTTP requests |
| `JSON` | JSON parse/generate |
| `CSV` | CSV read/write |
| `Benchmark` | วัดประสิทธิภาพ |
| `ObjectSpace` | Inspect Ruby runtime |

---

*ตอนถัดไป: ตอนที่ 21-22 (Modules ขั้นสูง และ Design Patterns)*
