# ตอนที่ 14: Error Handling (Steps 281-300)

## บทนำ

Error Handling เป็นส่วนสำคัญในการเขียนโปรแกรมที่แข็งแกร่งและน่าเชื่อถือ Ruby มีระบบ exception handling ที่ยืดหยุ่นและทรงพลัง ช่วยให้เราจัดการกับข้อผิดพลาดได้อย่างมีประสิทธิภาพ

---

## Step 281: Ruby Exception Hierarchy

Ruby มี exception hierarchy ที่มีโครงสร้างชัดเจน

```
Exception
├── ScriptError
│   ├── LoadError
│   ├── NotImplementedError
│   └── SyntaxError
├── SignalException
│   └── Interrupt
├── SystemExit
└── StandardError  ← rescue ส่วนใหญ่จับ StandardError
    ├── ArgumentError
    │   └── UncaughtThrowError
    ├── EncodingError
    ├── FiberError
    ├── IOError
    │   └── EOFError
    ├── IndexError
    │   ├── KeyError
    │   └── StopIteration
    ├── LocalJumpError
    ├── Math::DomainError
    ├── NameError
    │   └── NoMethodError
    ├── RangeError
    │   └── FloatDomainError
    ├── RegexpError
    ├── RuntimeError  ← default raise/fail ใช้อัน
    ├── SystemCallError
    │   └── Errno::* (ENOENT, EACCES, etc.)
    ├── ThreadError
    ├── TypeError
    ├── ZeroDivisionError
    └── ...
```

```ruby
# ดู hierarchy ด้วย superclass
puts ArgumentError.superclass    # => StandardError
puts StandardError.superclass    # => Exception
puts Exception.superclass        # => Object

puts TypeError.superclass        # => StandardError
puts NoMethodError.superclass    # => NameError
puts NameError.superclass        # => StandardError

puts ZeroDivisionError.superclass  # => StandardError
puts IOError.superclass            # => StandardError
puts LoadError.superclass          # => ScriptError

# ตรวจสอบ inheritance
begin
  1 / 0
rescue ZeroDivisionError => e
  puts e.class                       # => ZeroDivisionError
  puts e.is_a?(ZeroDivisionError)    # => true
  puts e.is_a?(StandardError)        # => true
  puts e.is_a?(Exception)            # => true
  puts e.is_a?(RuntimeError)         # => false
end

# ดู exceptions ทั้งหมดของ StandardError
def subclasses_of(klass)
  ObjectSpace.each_object(Class).select { |c| c < klass }
end

puts "\nStandardError subclasses (sample):"
subclasses_of(StandardError).sort_by(&:name).first(10).each do |klass|
  puts "  #{klass}"
end
```

---

## Step 282: begin / rescue / ensure / else / end

โครงสร้างพื้นฐานของ exception handling ใน Ruby

```ruby
# โครงสร้างพื้นฐาน
begin
  # code ที่อาจเกิด exception
rescue
  # จัดการ exception
ensure
  # เรียกเสมอ ไม่ว่าจะมี exception หรือไม่
end

# ตัวอย่างครบ
begin
  puts "Trying..."
  result = 10 / 2
  puts "Result: #{result}"
rescue ZeroDivisionError => e
  puts "Error: #{e.message}"
else
  # เรียกเมื่อ ไม่มี exception
  puts "Success! No errors."
ensure
  # เรียกเสมอ
  puts "This always runs."
end

puts "\n--- With error ---"
begin
  puts "Trying..."
  result = 10 / 0
  puts "Result: #{result}"
rescue ZeroDivisionError => e
  puts "Error: #{e.message}"
else
  puts "Success! (won't reach here)"
ensure
  puts "This always runs."
end
```

### begin/rescue ครบสูตร

```ruby
def safe_read_file(filename)
  puts "Opening #{filename}..."
  file = File.open(filename, 'r')
  content = file.read
  puts "Read #{content.length} characters"
  content
rescue Errno::ENOENT => e
  puts "File not found: #{e.message}"
  nil
rescue Errno::EACCES => e
  puts "Permission denied: #{e.message}"
  nil
rescue IOError => e
  puts "IO Error: #{e.message}"
  nil
rescue StandardError => e
  puts "Unexpected error: #{e.class}: #{e.message}"
  nil
else
  puts "File read successfully"
  content
ensure
  # ปิด file เสมอ แม้จะเกิด error
  if file && !file.closed?
    file.close
    puts "File closed"
  end
end

result = safe_read_file("existing_file.txt")
result2 = safe_read_file("nonexistent.txt")
```

### ensure กับ cleanup

```ruby
class DatabaseConnection
  def initialize(host)
    @host = host
    @connected = false
  end

  def connect
    puts "Connecting to #{@host}..."
    @connected = true
    puts "Connected!"
  end

  def disconnect
    if @connected
      @connected = false
      puts "Disconnected from #{@host}"
    end
  end

  def query(sql)
    raise "Not connected" unless @connected
    puts "Executing: #{sql}"
    # simulate result
    [{ id: 1, name: "Alice" }]
  end

  def connected?
    @connected
  end
end

db = DatabaseConnection.new("localhost:5432")

begin
  db.connect
  users = db.query("SELECT * FROM users WHERE active = true")
  puts "Found #{users.length} users"
  db.query("UPDATE invalid_table SET x = 1")  # ไม่ error ในที่นี้
rescue RuntimeError => e
  puts "Database error: #{e.message}"
ensure
  db.disconnect  # ปิด connection เสมอ
end

puts "Connection status: #{db.connected?}"
```

---

## Step 283: rescue Specific Exceptions

```ruby
def parse_config(config_string)
  begin
    # อาจเกิด JSON::ParseError หรือ TypeError
    require 'json'
    data = JSON.parse(config_string)

    # อาจเกิด ArgumentError
    port = Integer(data["port"])
    raise ArgumentError, "Port must be between 1-65535" unless (1..65535).include?(port)

    # อาจเกิด KeyError
    host = data.fetch("host")

    { host: host, port: port }
  rescue JSON::ParserError => e
    puts "Invalid JSON: #{e.message}"
    nil
  rescue ArgumentError => e
    puts "Invalid argument: #{e.message}"
    nil
  rescue KeyError => e
    puts "Missing required key: #{e.message}"
    nil
  end
end

puts parse_config('{"host": "localhost", "port": 3000}').inspect
puts parse_config('invalid json').inspect
puts parse_config('{"host": "localhost", "port": "abc"}').inspect
puts parse_config('{"port": 3000}').inspect  # missing host
```

### rescue ตาม severity

```ruby
def process_order(order_id)
  order = fetch_order(order_id)
  validate_order!(order)
  charge_customer(order)
  ship_order(order)

rescue ActiveRecord::RecordNotFound => e
  # ไม่พบ order - อาจเป็นปัญหาของ user
  log_warning("Order not found: #{order_id}")
  { error: :not_found, message: "Order #{order_id} not found" }

rescue ValidationError => e
  # Order ไม่ valid - ปัญหาของ data
  log_warning("Order validation failed: #{e.message}")
  { error: :invalid, message: e.message }

rescue PaymentError => e
  # การชำระเงินล้มเหลว - retry ได้
  log_error("Payment failed for order #{order_id}: #{e.message}")
  notify_customer_payment_failed(order_id)
  { error: :payment_failed, message: "Payment processing failed" }

rescue NetworkError => e
  # Network error - retry ได้
  log_error("Network error: #{e.message}")
  { error: :network_error, message: "Service temporarily unavailable" }

rescue StandardError => e
  # Error ที่ไม่คาดหวัง - log อย่างละเอียด
  log_critical("Unexpected error for order #{order_id}: #{e.class}: #{e.message}")
  log_critical(e.backtrace.first(5).join("\n"))
  { error: :internal_error, message: "An unexpected error occurred" }
end

# ตัวอย่างที่ทำงานได้จริง
def divide_numbers(a, b)
  begin
    result = a / b
    puts "#{a} / #{b} = #{result}"
    result
  rescue ZeroDivisionError
    puts "Cannot divide by zero!"
    0
  rescue TypeError => e
    puts "Type error: #{e.message}"
    nil
  end
end

divide_numbers(10, 2)
divide_numbers(10, 0)
divide_numbers(10, "two")
```

---

## Step 284: rescue Multiple Exceptions

```ruby
# rescue หลาย exceptions ในบรรทัดเดียว
def safe_convert(value)
  Integer(value)
rescue ArgumentError, TypeError
  puts "Cannot convert #{value.inspect} to integer"
  nil
end

puts safe_convert("123")   # => 123
puts safe_convert("abc").inspect   # => nil (converted failed)
puts safe_convert(nil).inspect     # => nil

# rescue หลาย exceptions แยกกัน
def process_data(data)
  result = JSON.parse(data)
  result.fetch("required_key")
rescue JSON::ParserError => e
  puts "JSON parse error: #{e.message}"
  {}
rescue KeyError => e
  puts "Missing key: #{e.message}"
  {}
rescue StandardError => e
  puts "Unexpected: #{e.class}: #{e.message}"
  {}
end

# ลำดับ rescue สำคัญ - specific ก่อน general
def careful_rescue(operation)
  begin
    case operation
    when :zero_div then 1 / 0
    when :no_method then nil.upcase
    when :type_error then "a" + 1
    when :name_error then undefined_variable
    end
  rescue ZeroDivisionError => e
    puts "ZeroDivisionError: #{e.message}"
  rescue NoMethodError => e
    puts "NoMethodError: #{e.message}"
  rescue TypeError => e
    puts "TypeError: #{e.message}"
  rescue NameError => e
    puts "NameError: #{e.message}"
  rescue StandardError => e
    puts "StandardError (catch-all): #{e.class} - #{e.message}"
  end
end

careful_rescue(:zero_div)
careful_rescue(:no_method)
careful_rescue(:type_error)
careful_rescue(:name_error)
```

### rescue กับ raise ใหม่

```ruby
def risky_operation
  begin
    # some risky code
    raise "Something went wrong"
  rescue RuntimeError => e
    puts "Caught RuntimeError, converting to custom error"
    raise CustomError, "Wrapped: #{e.message}"
  end
end

class CustomError < StandardError
  def initialize(msg = "Custom error occurred")
    super
  end
end

begin
  risky_operation
rescue CustomError => e
  puts "CustomError: #{e.message}"
end
```

---

## Step 285: Exception Message และ Backtrace

```ruby
begin
  raise RuntimeError, "Something went wrong"
rescue RuntimeError => e
  puts "Class:   #{e.class}"
  puts "Message: #{e.message}"
  puts "Inspect: #{e.inspect}"

  puts "\nFirst 3 lines of backtrace:"
  e.backtrace&.first(3)&.each { |line| puts "  #{line}" }
end

# Exception กับ cause (Ruby 2.0+)
def outer
  inner
rescue RuntimeError => e
  raise StandardError, "Outer error"
end

def inner
  raise RuntimeError, "Inner error"
end

begin
  outer
rescue StandardError => e
  puts "Error: #{e.message}"
  puts "Caused by: #{e.cause&.message}"
  puts "Cause class: #{e.cause&.class}"
end
```

### Custom Exception กับ metadata

```ruby
class AppError < StandardError
  attr_reader :code, :context, :timestamp

  def initialize(message, code: :unknown, context: {})
    super(message)
    @code = code
    @context = context
    @timestamp = Time.now
  end

  def to_h
    {
      error: self.class.name,
      message: message,
      code: @code,
      context: @context,
      timestamp: @timestamp.iso8601
    }
  end

  def to_json
    require 'json'
    to_h.to_json
  end

  def log_message
    "[#{@timestamp.strftime('%Y-%m-%d %H:%M:%S')}] #{self.class.name} (#{@code}): #{message}"
  end
end

class ValidationError < AppError
  attr_reader :field, :value

  def initialize(field, value, message)
    super(message, code: :validation_failed, context: { field: field, value: value })
    @field = field
    @value = value
  end
end

class NetworkError < AppError
  attr_reader :url, :status_code

  def initialize(url, status_code, message)
    super(message, code: :network_error, context: { url: url, status: status_code })
    @url = url
    @status_code = status_code
  end
end

# ทดสอบ
begin
  raise ValidationError.new("email", "invalid-email", "Email format is invalid")
rescue ValidationError => e
  puts e.log_message
  puts "Field: #{e.field}, Value: #{e.value.inspect}"
  puts "JSON: #{e.to_json}"
end

begin
  raise NetworkError.new("https://api.example.com", 503, "Service Unavailable")
rescue NetworkError => e
  puts e.log_message
  puts "URL: #{e.url}, Status: #{e.status_code}"
end
```

---

## Step 286: raise vs fail

`raise` และ `fail` เป็น method เดียวกัน (aliases) แต่มี convention ต่างกัน

```ruby
# raise - ใช้สำหรับ throw exception อย่างตั้งใจ
def find_user!(id)
  user = database_lookup(id)
  raise "User not found: #{id}" unless user
  user
end

# fail - ใช้สำหรับแสดงว่า method ล้มเหลว (convention เท่านั้น)
def divide(a, b)
  fail ArgumentError, "Divisor cannot be zero" if b == 0
  a / b
end

# วิธีการ raise แบบต่างๆ
begin
  raise              # raise RuntimeError ว่างเปล่า
rescue RuntimeError => e
  puts "1. #{e.message}"
end

begin
  raise "Something wrong"  # raise RuntimeError กับ message
rescue RuntimeError => e
  puts "2. #{e.message}"
end

begin
  raise ArgumentError      # raise class
rescue ArgumentError => e
  puts "3. #{e.message}"   # => ArgumentError
end

begin
  raise ArgumentError, "Bad argument"  # raise class กับ message
rescue ArgumentError => e
  puts "4. #{e.message}"
end

begin
  error = RuntimeError.new("Pre-built error")
  raise error              # raise instance
rescue RuntimeError => e
  puts "5. #{e.message}"
end

# raise ใน rescue - re-raise same exception
begin
  begin
    raise "Original error"
  rescue RuntimeError => e
    puts "Caught and re-raising: #{e.message}"
    raise  # re-raise ตัวเดิม
  end
rescue RuntimeError => e
  puts "Final catch: #{e.message}"
end
```

---

## Step 287: Custom Exception Classes

```ruby
# Exception hierarchy สำหรับ application
module MyApp
  class Error < StandardError
    def initialize(msg = "An error occurred in MyApp")
      super
    end
  end

  # Database errors
  class DatabaseError < Error
    attr_reader :query

    def initialize(msg, query: nil)
      super(msg)
      @query = query
    end
  end

  class RecordNotFound < DatabaseError
    attr_reader :model, :id

    def initialize(model, id)
      super("#{model} with id=#{id} not found", query: "SELECT FROM #{model.downcase}s WHERE id=#{id}")
      @model = model
      @id = id
    end
  end

  class DuplicateRecord < DatabaseError
    attr_reader :model, :field, :value

    def initialize(model, field, value)
      super("Duplicate #{field} '#{value}' in #{model}")
      @model = model
      @field = field
      @value = value
    end
  end

  # Authentication errors
  class AuthError < Error; end

  class UnauthorizedError < AuthError
    def initialize(msg = "Authentication required")
      super
    end
  end

  class ForbiddenError < AuthError
    attr_reader :required_permission

    def initialize(permission)
      super("You don't have permission: #{permission}")
      @required_permission = permission
    end
  end

  # Validation errors
  class ValidationError < Error
    attr_reader :errors

    def initialize(errors = {})
      @errors = errors
      messages = errors.map { |field, msgs| "#{field}: #{Array(msgs).join(', ')}" }
      super("Validation failed: #{messages.join('; ')}")
    end

    def full_messages
      @errors.flat_map { |field, msgs| Array(msgs).map { |m| "#{field} #{m}" } }
    end
  end

  # Network errors
  class NetworkError < Error
    attr_reader :url, :status_code, :response_body

    def initialize(url, status_code: nil, body: nil, msg: nil)
      message = msg || "Network error for #{url}"
      message += " (HTTP #{status_code})" if status_code
      super(message)
      @url = url
      @status_code = status_code
      @response_body = body
    end

    def client_error?
      @status_code && @status_code.between?(400, 499)
    end

    def server_error?
      @status_code && @status_code.between?(500, 599)
    end
  end
end

# ทดสอบ custom exceptions
def simulate_database_ops
  operations = [
    -> { raise MyApp::RecordNotFound.new("User", 999) },
    -> { raise MyApp::DuplicateRecord.new("User", "email", "alice@example.com") },
    -> { raise MyApp::ValidationError.new(name: ["can't be blank", "is too short"], email: ["is invalid"]) },
    -> { raise MyApp::ForbiddenError.new("admin:delete") },
    -> { raise MyApp::NetworkError.new("https://api.example.com/users", status_code: 503, msg: "Service unavailable") }
  ]

  operations.each do |op|
    begin
      op.call
    rescue MyApp::RecordNotFound => e
      puts "Not Found: #{e.message}"
      puts "  Query: #{e.query}"
    rescue MyApp::DuplicateRecord => e
      puts "Duplicate: #{e.message}"
    rescue MyApp::ValidationError => e
      puts "Validation failed:"
      e.full_messages.each { |m| puts "  - #{m}" }
    rescue MyApp::ForbiddenError => e
      puts "Forbidden: #{e.message}"
      puts "  Required: #{e.required_permission}"
    rescue MyApp::NetworkError => e
      puts "Network Error: #{e.message}"
      puts "  Server error: #{e.server_error?}"
    end
    puts
  end
end

simulate_database_ops
```

---

## Step 288: retry Mechanism

`retry` เรียก begin block ใหม่อีกครั้ง

```ruby
# retry พื้นฐาน
attempts = 0
begin
  attempts += 1
  puts "Attempt #{attempts}"
  raise "Network timeout" if attempts < 3
  puts "Success on attempt #{attempts}!"
rescue RuntimeError => e
  retry if attempts < 3
  puts "Failed after 3 attempts: #{e.message}"
end

# retry กับ exponential backoff
def with_exponential_backoff(max_retries: 5, base_delay: 0.1)
  retries = 0
  begin
    yield
  rescue StandardError => e
    retries += 1
    if retries <= max_retries
      delay = base_delay * (2 ** (retries - 1))
      puts "Retry #{retries}/#{max_retries} after #{delay.round(2)}s: #{e.message}"
      sleep(delay)
      retry
    else
      raise
    end
  end
end

# Simulate flaky API
call_count = 0
begin
  with_exponential_backoff(max_retries: 4, base_delay: 0.01) do
    call_count += 1
    raise "API Error" if call_count < 4
    puts "API call succeeded on attempt #{call_count}"
  end
rescue => e
  puts "All retries exhausted: #{e.message}"
end
```

### retry ใน HTTP client simulation

```ruby
class RetryableHTTPClient
  MAX_RETRIES = 3
  RETRYABLE_CODES = [429, 500, 502, 503, 504].freeze

  class HTTPError < StandardError
    attr_reader :status_code
    def initialize(status_code, message)
      super(message)
      @status_code = status_code
    end
  end

  def get(url, timeout: 5)
    retries = 0
    begin
      response = simulate_request(url)
      response
    rescue HTTPError => e
      if RETRYABLE_CODES.include?(e.status_code) && retries < MAX_RETRIES
        retries += 1
        wait_time = 2 ** retries  # exponential backoff: 2, 4, 8 seconds
        puts "HTTP #{e.status_code} - Retry #{retries}/#{MAX_RETRIES} in #{wait_time}s"
        sleep(0.001)  # ใช้ 0.001 เพื่อทดสอบให้เร็ว
        retry
      else
        raise
      end
    end
  end

  private

  def simulate_request(url)
    # Simulate different scenarios
    case url
    when /success/ then { status: 200, body: "OK" }
    when /timeout/
      raise Timeout::Error, "Request timed out"
    when /server_error/
      @attempt_count ||= 0
      @attempt_count += 1
      raise HTTPError.new(503, "Service Unavailable") if @attempt_count < 3
      { status: 200, body: "Recovered!" }
    else
      raise HTTPError.new(404, "Not Found")
    end
  end
end

client = RetryableHTTPClient.new

puts "=== Success request ==="
result = client.get("http://api.example.com/success")
puts "Result: #{result}"

puts "\n=== Server error with retry ==="
client2 = RetryableHTTPClient.new
result2 = client2.get("http://api.example.com/server_error")
puts "Result: #{result2}"

puts "\n=== Not Found (no retry) ==="
begin
  client3 = RetryableHTTPClient.new
  client3.get("http://api.example.com/nonexistent")
rescue RetryableHTTPClient::HTTPError => e
  puts "Failed: HTTP #{e.status_code} - #{e.message}"
end
```

---

## Step 289: rescue ใน Method Definitions (Inline)

```ruby
# rescue ใน method definition (ไม่ต้องใช้ begin)
def risky_method(n)
  result = 10 / n
  "Result: #{result}"
rescue ZeroDivisionError
  "Cannot divide by zero"
rescue TypeError => e
  "Type error: #{e.message}"
end

puts risky_method(2)      # => Result: 5
puts risky_method(0)      # => Cannot divide by zero
puts risky_method("two")  # => Type error: ...

# inline rescue ใน expression (ไม่แนะนำสำหรับทุกกรณี)
value = Integer("abc") rescue 0
puts value  # => 0

hash = { a: 1 }
result = hash.fetch(:b) rescue "default"
puts result  # => default

# method กับ ensure
def read_and_process(filename)
  file = File.open(filename)
  content = file.read
  process(content)
rescue Errno::ENOENT
  puts "File not found: #{filename}"
  nil
ensure
  file&.close
end

# แยก error handling ออกมาชัดเจน
class FileProcessor
  def process_file(path)
    validate_path!(path)
    content = read_file(path)
    parse_content(content)
  rescue PathError => e
    handle_path_error(e)
  rescue ReadError => e
    handle_read_error(e)
  rescue ParseError => e
    handle_parse_error(e)
  end

  private

  def validate_path!(path)
    raise PathError, "Path cannot be empty" if path.nil? || path.empty?
    raise PathError, "File must have .txt extension" unless path.end_with?('.txt')
  end

  def read_file(path)
    File.read(path)
  rescue Errno::ENOENT => e
    raise ReadError, "Cannot find file: #{e.message}"
  rescue Errno::EACCES => e
    raise ReadError, "Cannot read file: #{e.message}"
  end

  def parse_content(content)
    # parse logic
    content.split("\n")
  rescue => e
    raise ParseError, "Parse failed: #{e.message}"
  end

  def handle_path_error(e)
    puts "Path Error: #{e.message}"
    nil
  end

  def handle_read_error(e)
    puts "Read Error: #{e.message}"
    nil
  end

  def handle_parse_error(e)
    puts "Parse Error: #{e.message}"
    nil
  end
end

class PathError < StandardError; end
class ReadError < StandardError; end
class ParseError < StandardError; end
```

---

## Step 290: Global Exception Handlers

```ruby
# at_exit - รันเมื่อโปรแกรมจบ
at_exit do
  puts "\n[at_exit] Application shutting down..."
  # cleanup resources
end

# trap signals
begin
  trap("INT") do
    puts "\n[SIGNAL] Received Ctrl+C, cleaning up..."
    exit(0)
  end
rescue ArgumentError
  # บาง environments ไม่รองรับ
end

# Global exception handler กับ Thread
Thread.current.abort_on_exception = true

# ใช้ at_exit สำหรับ cleanup
class Application
  def initialize
    @resources = []

    at_exit do
      cleanup!
    end
  end

  def start
    puts "Application starting..."
    @resources << "database_connection"
    @resources << "cache_connection"
    @resources << "file_handles"
    puts "Resources acquired: #{@resources.inspect}"
  end

  def cleanup!
    puts "Cleaning up #{@resources.length} resources..."
    @resources.each do |r|
      puts "  Releasing: #{r}"
    end
    @resources.clear
  end
end

# $stderr สำหรับ error output
$stderr.puts "This goes to stderr" if false

# Kernel#warn - output ไปที่ stderr
# warn "This is a warning"

# Exception handling ด้วย Proc
error_handler = ->(e) {
  puts "[ERROR HANDLER] #{e.class}: #{e.message}"
}

[
  -> { raise ArgumentError, "bad arg" },
  -> { raise RuntimeError, "runtime error" },
  -> { raise TypeError, "type error" }
].each do |operation|
  begin
    operation.call
  rescue => e
    error_handler.call(e)
  end
end
```

---

## Step 291: Defensive Programming Patterns

### Guard Clauses

```ruby
# Anti-pattern: nested conditions
def process_order_bad(order)
  if order
    if order[:items]
      if order[:items].any?
        if order[:customer]
          if order[:customer][:email]
            # ทำงานหลัก
            puts "Processing order for #{order[:customer][:email]}"
          else
            raise "Missing customer email"
          end
        else
          raise "Missing customer"
        end
      else
        raise "Order has no items"
      end
    else
      raise "Order has no items field"
    end
  else
    raise "Order is nil"
  end
end

# Good practice: Guard Clauses
def process_order(order)
  raise ArgumentError, "Order cannot be nil" unless order
  raise ArgumentError, "Order must have items" unless order[:items]&.any?
  raise ArgumentError, "Missing customer info" unless order[:customer]
  raise ArgumentError, "Missing customer email" unless order[:customer][:email]

  # ทำงานหลัก - code ชัดเจนกว่ามาก
  puts "Processing order for #{order[:customer][:email]}"
  total = order[:items].sum { |item| item[:price] * item[:qty] }
  puts "Order total: ฿#{total}"
  { success: true, total: total }
end

# ทดสอบ guard clauses
begin
  process_order(nil)
rescue ArgumentError => e
  puts "Error: #{e.message}"
end

begin
  process_order({ items: [], customer: { email: "alice@example.com" } })
rescue ArgumentError => e
  puts "Error: #{e.message}"
end

result = process_order({
  items: [{ name: "Book", price: 350, qty: 2 }, { name: "Pen", price: 50, qty: 5 }],
  customer: { name: "Alice", email: "alice@example.com" }
})
puts result.inspect
```

### Null Object Pattern

```ruby
class NullUser
  def name; "Guest"; end
  def email; nil; end
  def admin?; false; end
  def logged_in?; false; end
  def to_s; "Guest (not logged in)"; end
  def null?; true; end
end

class User
  attr_reader :name, :email, :role

  def initialize(name, email, role = :user)
    @name = name
    @email = email
    @role = role
  end

  def admin?
    @role == :admin
  end

  def logged_in?
    true
  end

  def null?
    false
  end

  def to_s
    "#{@name} <#{@email}>"
  end
end

def find_user(id)
  users = {
    1 => User.new("Alice", "alice@example.com", :admin),
    2 => User.new("Bob", "bob@example.com")
  }
  users[id] || NullUser.new  # ไม่ return nil
end

# ไม่ต้องเช็ค nil ทุกที่
[1, 2, 999].each do |id|
  user = find_user(id)
  puts "User: #{user}"
  puts "  Admin: #{user.admin?}"
  puts "  Logged in: #{user.logged_in?}"
  # ไม่ต้องเช็ค user.nil? ทุกที่
end
```

### Result Object Pattern

```ruby
class Result
  attr_reader :value, :error

  def self.success(value)
    new(value: value, success: true)
  end

  def self.failure(error)
    new(error: error, success: false)
  end

  def initialize(value: nil, error: nil, success:)
    @value = value
    @error = error
    @success = success
  end

  def success?
    @success
  end

  def failure?
    !@success
  end

  def on_success
    yield(@value) if @success
    self
  end

  def on_failure
    yield(@error) if !@success
    self
  end

  def map
    return self if failure?
    begin
      Result.success(yield(@value))
    rescue => e
      Result.failure(e.message)
    end
  end

  def unwrap
    raise @error if failure?
    @value
  end

  def unwrap_or(default)
    success? ? @value : default
  end

  def to_s
    success? ? "Success(#{@value})" : "Failure(#{@error})"
  end
end

# ใช้ Result แทน throwing exceptions
def parse_integer(str)
  Result.success(Integer(str))
rescue ArgumentError
  Result.failure("'#{str}' is not a valid integer")
end

def divide(a, b)
  return Result.failure("Division by zero") if b == 0
  Result.success(a.to_f / b)
end

def calculate(a_str, b_str)
  parse_integer(a_str)
    .map { |a| [a, parse_integer(b_str)] }
    .map { |a, b_result| b_result.success? ? divide(a, b_result.value) : b_result }
    .on_success { |result| puts "Result: #{result.value}" }
    .on_failure { |error| puts "Error: #{error}" }
end

# ทดสอบ
puts parse_integer("42")     # => Success(42)
puts parse_integer("hello")  # => Failure(...)
puts divide(10, 2)           # => Success(5.0)
puts divide(10, 0)           # => Failure(Division by zero)

result = parse_integer("100")
result.on_success { |v| puts "Got value: #{v}" }
      .on_failure { |e| puts "Got error: #{e}" }

# Chaining
chain_result = parse_integer("50")
  .map { |n| n * 2 }
  .map { |n| n + 10 }

puts chain_result.unwrap  # => 110
puts parse_integer("abc").unwrap_or(0)  # => 0
```

### Fail Fast Pattern

```ruby
class OrderProcessor
  def initialize(payment_gateway, inventory, notifier)
    @gateway = payment_gateway
    @inventory = inventory
    @notifier = notifier
  end

  def process(order)
    # Fail fast: validate everything before starting
    validate!(order)

    # All validations passed, now process
    result = charge_payment(order)
    update_inventory(order)
    send_confirmation(order)

    { success: true, transaction_id: result[:transaction_id] }
  rescue ValidationError => e
    { success: false, error: :validation, message: e.message }
  rescue PaymentError => e
    # Payment failed - no inventory update needed
    { success: false, error: :payment, message: e.message }
  rescue InventoryError => e
    # Inventory failed - refund payment
    refund_payment(order)
    { success: false, error: :inventory, message: e.message }
  end

  private

  def validate!(order)
    raise ValidationError, "Order is nil" unless order
    raise ValidationError, "No items in order" unless order[:items]&.any?
    raise ValidationError, "Invalid total" unless order[:total]&.positive?
    raise ValidationError, "No customer email" unless order[:customer_email]
  end

  def charge_payment(order)
    # Payment processing
    puts "Charging ฿#{order[:total]}"
    { transaction_id: "TXN#{rand(10000)}" }
  end

  def update_inventory(order)
    puts "Updating inventory"
  end

  def send_confirmation(order)
    puts "Sending confirmation to #{order[:customer_email]}"
  end

  def refund_payment(order)
    puts "Refunding payment for order"
  end
end

class ValidationError < StandardError; end
class PaymentError < StandardError; end
class InventoryError < StandardError; end

processor = OrderProcessor.new(nil, nil, nil)

# Valid order
result = processor.process({
  items: [{name: "Book", qty: 1, price: 350}],
  total: 350,
  customer_email: "alice@example.com"
})
puts result.inspect

# Invalid order
result = processor.process({ items: [] })
puts result.inspect
```

---

## Step 292-300: แบบฝึกหัด 20 ข้อ พร้อมเฉลย

### ข้อที่ 1: Safe Calculator

```ruby
class SafeCalculator
  class CalculatorError < StandardError; end
  class DivisionByZeroError < CalculatorError; end
  class OverflowError < CalculatorError; end
  class DomainError < CalculatorError; end

  MAX_VALUE = 10**15

  def add(a, b)
    result = a + b
    raise OverflowError, "Result #{result} exceeds maximum" if result.abs > MAX_VALUE
    result
  rescue TypeError => e
    raise CalculatorError, "Type error: #{e.message}"
  end

  def subtract(a, b)
    add(a, -b)
  end

  def multiply(a, b)
    result = a * b
    raise OverflowError, "Result exceeds maximum" if result.abs > MAX_VALUE
    result
  rescue TypeError => e
    raise CalculatorError, "Type error: #{e.message}"
  end

  def divide(a, b)
    raise DivisionByZeroError, "Cannot divide by zero" if b == 0
    a.to_f / b
  end

  def sqrt(n)
    raise DomainError, "Cannot take sqrt of negative number" if n < 0
    Math.sqrt(n)
  end

  def safe_eval(expression)
    # Only allow numbers and basic operators
    unless expression =~ /\A[\d\s\+\-\*\/\.\(\)]+\z/
      raise CalculatorError, "Invalid expression: #{expression}"
    end
    eval(expression)
  rescue SyntaxError => e
    raise CalculatorError, "Syntax error: #{e.message}"
  end
end

calc = SafeCalculator.new

# Test various scenarios
[
  -> { puts "10 + 5 = #{calc.add(10, 5)}" },
  -> { puts "10 / 2 = #{calc.divide(10, 2)}" },
  -> { puts "10 / 0 raises error" ; calc.divide(10, 0) },
  -> { puts "sqrt(-1) raises error" ; calc.sqrt(-1) },
  -> { puts "sqrt(16) = #{calc.sqrt(16)}" },
].each do |op|
  begin
    op.call
  rescue SafeCalculator::CalculatorError => e
    puts "  CalculatorError: #{e.class.name.split('::').last} - #{e.message}"
  end
end
```

### ข้อที่ 2: File Operations with Error Handling

```ruby
class SafeFileOps
  class FileError < StandardError; end
  class FileNotFoundError < FileError; end
  class PermissionError < FileError; end
  class FileSizeError < FileError; end

  MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB

  def read(filename, encoding: 'UTF-8')
    validate_filename!(filename)

    File.open(filename, "r:#{encoding}") do |f|
      size = f.size
      raise FileSizeError, "File too large: #{size} bytes" if size > MAX_FILE_SIZE
      f.read
    end
  rescue Errno::ENOENT
    raise FileNotFoundError, "File not found: #{filename}"
  rescue Errno::EACCES
    raise PermissionError, "Cannot read: #{filename}"
  rescue Encoding::InvalidByteSequenceError => e
    raise FileError, "Encoding error: #{e.message}"
  end

  def write(filename, content, mode: 'w')
    validate_filename!(filename)
    File.write(filename, content, mode: mode)
    true
  rescue Errno::EACCES
    raise PermissionError, "Cannot write: #{filename}"
  rescue Errno::ENOENT
    raise FileNotFoundError, "Directory not found for: #{filename}"
  end

  def copy(source, destination)
    content = read(source)
    write(destination, content)
    puts "Copied #{source} → #{destination}"
    true
  rescue FileError => e
    puts "Copy failed: #{e.message}"
    false
  end

  def safe_delete(filename)
    File.delete(filename)
    puts "Deleted: #{filename}"
    true
  rescue Errno::ENOENT
    puts "File already gone: #{filename}"
    false
  rescue Errno::EACCES => e
    raise PermissionError, "Cannot delete: #{e.message}"
  end

  private

  def validate_filename!(filename)
    raise ArgumentError, "Filename cannot be nil" if filename.nil?
    raise ArgumentError, "Filename cannot be empty" if filename.strip.empty?
    raise ArgumentError, "Invalid filename characters" if filename =~ /[<>:"|?*]/
  end
end

ops = SafeFileOps.new

# Write
ops.write("/tmp/test_error_handling.txt", "Hello, World!\nLine 2\nLine 3\n")

# Read
begin
  content = ops.read("/tmp/test_error_handling.txt")
  puts "Read #{content.length} chars"
rescue SafeFileOps::FileError => e
  puts "Error: #{e.message}"
end

# Read non-existent
begin
  ops.read("/tmp/nonexistent_file_xyz.txt")
rescue SafeFileOps::FileNotFoundError => e
  puts "#{e.class.name.split('::').last}: #{e.message}"
end

# Cleanup
ops.safe_delete("/tmp/test_error_handling.txt")
ops.safe_delete("/tmp/test_error_handling.txt")  # already gone
```

### ข้อที่ 3: Network Client with Retry

```ruby
class NetworkClient
  class NetworkError < StandardError
    attr_reader :code
    def initialize(msg, code: nil); super(msg); @code = code; end
  end
  class TimeoutError < NetworkError; end
  class ServerError < NetworkError; end
  class ClientError < NetworkError; end

  DEFAULT_RETRY_CONFIG = {
    max_attempts: 3,
    initial_delay: 0.5,
    max_delay: 30,
    backoff_factor: 2,
    retryable_codes: [429, 500, 502, 503, 504]
  }.freeze

  def initialize(retry_config = {})
    @config = DEFAULT_RETRY_CONFIG.merge(retry_config)
    @request_count = 0
    @error_count = 0
  end

  def get(url, **options)
    with_retry { make_request(:get, url, options) }
  end

  def post(url, body: {}, **options)
    with_retry { make_request(:post, url, options.merge(body: body)) }
  end

  def stats
    {
      total_requests: @request_count,
      errors: @error_count,
      success_rate: @request_count > 0 ?
        "#{((@request_count - @error_count).to_f / @request_count * 100).round(1)}%" : "N/A"
    }
  end

  private

  def make_request(method, url, options)
    @request_count += 1
    # Simulate request
    case url
    when /timeout/ then raise TimeoutError.new("Request timed out", code: nil)
    when /500/ then raise ServerError.new("Internal Server Error", code: 500)
    when /404/ then raise ClientError.new("Not Found", code: 404)
    when /503/
      @attempt_count ||= 0
      @attempt_count += 1
      raise ServerError.new("Service Unavailable", code: 503) if @attempt_count < 3
      { status: 200, body: { data: "recovered" } }
    else
      { status: 200, body: { url: url, method: method } }
    end
  rescue NetworkError
    @error_count += 1
    raise
  end

  def with_retry(&block)
    attempts = 0
    delay = @config[:initial_delay]

    begin
      block.call
    rescue TimeoutError, ServerError => e
      attempts += 1
      max = @config[:max_attempts]

      if attempts < max && should_retry?(e)
        actual_delay = [delay, @config[:max_delay]].min
        puts "  Retry #{attempts}/#{max - 1} after #{actual_delay.round(2)}s: #{e.class.name.split('::').last}"
        sleep(0.001)  # simulated
        delay *= @config[:backoff_factor]
        retry
      end
      raise
    end
  end

  def should_retry?(error)
    case error
    when TimeoutError then true
    when ServerError then @config[:retryable_codes].include?(error.code)
    else false
    end
  end
end

client = NetworkClient.new(initial_delay: 0.001)

puts "=== Success ==="
result = client.get("https://api.example.com/users")
puts "Response: #{result.inspect}"

puts "\n=== With retry (503) ==="
result = client.get("https://api.example.com/503/users")
puts "Response: #{result.inspect}"

puts "\n=== Client error (no retry) ==="
begin
  client.get("https://api.example.com/404/resource")
rescue NetworkClient::ClientError => e
  puts "Client error (not retried): #{e.message}"
end

puts "\n=== Stats ==="
puts client.stats.inspect
```

### ข้อที่ 4-10: เพิ่มเติม

```ruby
# ข้อที่ 4: Input Validator
class InputValidator
  class ValidationError < StandardError
    attr_reader :field, :value, :rule
    def initialize(field, value, rule, message)
      super(message)
      @field = field
      @value = value
      @rule = rule
    end
  end

  RULES = {
    required: ->(v, _) { !v.nil? && (v.respond_to?(:empty?) ? !v.empty? : true) },
    min_length: ->(v, n) { !v.respond_to?(:length) || v.length >= n },
    max_length: ->(v, n) { !v.respond_to?(:length) || v.length <= n },
    email: ->(v, _) { !v || v =~ /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i },
    numeric: ->(v, _) { !v || v.to_s =~ /\A\d+(\.\d+)?\z/ },
    min: ->(v, n) { !v || v.to_f >= n },
    max: ->(v, n) { !v || v.to_f <= n },
  }

  def initialize
    @schema = {}
    @errors = {}
  end

  def define(field, **rules)
    @schema[field] = rules
    self
  end

  def validate(data)
    @errors = {}
    @schema.each do |field, rules|
      value = data[field]
      rules.each do |rule, param|
        validator = RULES[rule]
        next unless validator
        unless validator.call(value, param)
          @errors[field] ||= []
          @errors[field] << rule_message(rule, field, param)
        end
      end
    end
    @errors.empty?
  end

  def validate!(data)
    raise ValidationError.new(nil, nil, nil, error_summary) unless validate(data)
    data
  end

  def errors
    @errors.dup
  end

  private

  def rule_message(rule, field, param)
    messages = {
      required: "#{field} is required",
      min_length: "#{field} must be at least #{param} characters",
      max_length: "#{field} must be at most #{param} characters",
      email: "#{field} must be a valid email",
      numeric: "#{field} must be a number",
      min: "#{field} must be at least #{param}",
      max: "#{field} must be at most #{param}",
    }
    messages[rule] || "#{field} is invalid"
  end

  def error_summary
    @errors.map { |f, msgs| msgs.join(', ') }.join('; ')
  end
end

validator = InputValidator.new
validator.define(:name, required: true, min_length: 2, max_length: 50)
         .define(:email, required: true, email: true)
         .define(:age, required: true, numeric: true, min: 18, max: 120)

test_cases = [
  { name: "Alice Smith", email: "alice@example.com", age: "30" },
  { name: "A", email: "bad-email", age: "15" },
  { email: "alice@example.com", age: "30" },  # missing name
]

test_cases.each_with_index do |data, i|
  puts "\n--- Test #{i + 1} ---"
  if validator.validate(data)
    puts "Valid: #{data.inspect}"
  else
    puts "Invalid:"
    validator.errors.each { |f, msgs| msgs.each { |m| puts "  - #{m}" } }
  end
end
```

```ruby
# ข้อที่ 5: Circuit Breaker Pattern
class CircuitBreaker
  STATES = [:closed, :open, :half_open].freeze

  class CircuitOpenError < StandardError
    def initialize
      super("Circuit breaker is open - service unavailable")
    end
  end

  def initialize(name:, failure_threshold: 5, recovery_timeout: 30, success_threshold: 2)
    @name = name
    @failure_threshold = failure_threshold
    @recovery_timeout = recovery_timeout
    @success_threshold = success_threshold

    @state = :closed
    @failure_count = 0
    @success_count = 0
    @last_failure_time = nil
    @total_calls = 0
    @total_failures = 0
  end

  def call
    @total_calls += 1

    case @state
    when :open
      if Time.now - @last_failure_time >= @recovery_timeout
        transition_to(:half_open)
        puts "[#{@name}] Circuit HALF-OPEN - testing service"
      else
        @total_failures += 1
        raise CircuitOpenError
      end
    when :half_open, :closed
      # proceed
    end

    begin
      result = yield
      on_success
      result
    rescue => e
      on_failure(e)
      raise
    end
  end

  def state
    @state
  end

  def stats
    {
      name: @name,
      state: @state,
      failures: @failure_count,
      total_calls: @total_calls,
      total_failures: @total_failures
    }
  end

  private

  def on_success
    case @state
    when :half_open
      @success_count += 1
      if @success_count >= @success_threshold
        transition_to(:closed)
        puts "[#{@name}] Circuit CLOSED - service recovered"
      end
    when :closed
      @failure_count = 0  # reset on success
    end
  end

  def on_failure(error)
    @total_failures += 1
    @last_failure_time = Time.now

    case @state
    when :closed
      @failure_count += 1
      if @failure_count >= @failure_threshold
        transition_to(:open)
        puts "[#{@name}] Circuit OPEN - too many failures!"
      end
    when :half_open
      transition_to(:open)
      puts "[#{@name}] Circuit re-opened - service still failing"
    end
  end

  def transition_to(new_state)
    @state = new_state
    @failure_count = 0 if new_state == :closed
    @success_count = 0
  end
end

# Simulate a flaky service
service_calls = 0
breaker = CircuitBreaker.new(
  name: "UserService",
  failure_threshold: 3,
  recovery_timeout: 1
)

def call_service(breaker, call_num, should_fail)
  breaker.call do
    raise "Service error!" if should_fail
    "Response from call #{call_num}"
  end
rescue CircuitBreaker::CircuitOpenError => e
  "Circuit open: #{e.message}"
rescue RuntimeError => e
  "Service error: #{e.message}"
end

# Successful calls
puts "=== Normal operation ==="
puts call_service(breaker, 1, false)
puts call_service(breaker, 2, false)

# Failures - should open circuit
puts "\n=== Failures ==="
puts call_service(breaker, 3, true)
puts call_service(breaker, 4, true)
puts call_service(breaker, 5, true)

# Circuit should be open
puts "\n=== Circuit open ==="
puts call_service(breaker, 6, false)
puts call_service(breaker, 7, false)

puts "\nStats: #{breaker.stats.inspect}"
```

```ruby
# ข้อที่ 6: Exception Logger
class ExceptionLogger
  LOG_LEVELS = { debug: 0, info: 1, warn: 2, error: 3, fatal: 4 }

  def initialize(output: $stdout, min_level: :error)
    @output = output
    @min_level = min_level
    @entries = []
  end

  def log_exception(exception, level: :error, context: {})
    entry = build_entry(exception, level, context)
    @entries << entry

    if LOG_LEVELS[level] >= LOG_LEVELS[@min_level]
      write_entry(entry)
    end

    entry
  end

  def entries
    @entries.dup
  end

  def recent_errors(n = 10)
    @entries.select { |e| LOG_LEVELS[e[:level]] >= LOG_LEVELS[:error] }.last(n)
  end

  def summary
    by_class = @entries.group_by { |e| e[:class] }
    {
      total: @entries.length,
      by_class: by_class.transform_values(&:length).sort_by { |_, v| -v }.first(5).to_h
    }
  end

  private

  def build_entry(exception, level, context)
    {
      timestamp: Time.now.iso8601,
      level: level,
      class: exception.class.name,
      message: exception.message,
      backtrace: exception.backtrace&.first(3),
      context: context
    }
  end

  def write_entry(entry)
    @output.puts format_entry(entry)
  end

  def format_entry(entry)
    parts = [
      "[#{entry[:timestamp]}]",
      "[#{entry[:level].upcase.ljust(5)}]",
      "#{entry[:class]}: #{entry[:message]}"
    ]
    parts << "Context: #{entry[:context]}" unless entry[:context].empty?
    parts.join(" ")
  end
end

logger = ExceptionLogger.new(min_level: :warn)

# Log various exceptions
begin
  raise ArgumentError, "Invalid argument provided"
rescue => e
  logger.log_exception(e, level: :error, context: { user_id: 123, action: "create_post" })
end

begin
  Integer("not a number")
rescue => e
  logger.log_exception(e, level: :warn, context: { input: "not a number" })
end

begin
  raise RuntimeError, "Unexpected state"
rescue => e
  logger.log_exception(e, level: :fatal, context: { component: "payment_processor" })
end

puts "\n=== Exception Summary ==="
summary = logger.summary
puts "Total exceptions: #{summary[:total]}"
puts "By class:"
summary[:by_class].each { |klass, count| puts "  #{klass}: #{count}" }
```

```ruby
# ข้อที่ 7: Transactional Operations
class Transaction
  class RollbackError < StandardError; end

  attr_reader :status, :operations

  def initialize
    @operations = []
    @rollback_ops = []
    @status = :pending
  end

  def add_operation(name, &forward)
    @operations << { name: name, action: forward }
    self
  end

  def add_rollback(name, &rollback)
    @rollback_ops << { name: name, action: rollback }
    self
  end

  def execute
    @status = :running
    completed_ops = []

    @operations.each do |op|
      begin
        puts "  Executing: #{op[:name]}"
        op[:action].call
        completed_ops << op[:name]
      rescue => e
        puts "  FAILED: #{op[:name]} - #{e.message}"
        puts "  Rolling back..."

        rollback!(completed_ops)
        @status = :rolled_back
        return false
      end
    end

    @status = :committed
    true
  end

  def commit
    @status = :committed
    puts "Transaction committed!"
    true
  end

  private

  def rollback!(completed)
    @rollback_ops.reverse.each do |op|
      begin
        puts "  Rollback: #{op[:name]}"
        op[:action].call
      rescue => e
        puts "  Rollback failed: #{op[:name]} - #{e.message}"
      end
    end
  end
end

# Simulate a bank transfer transaction
def transfer_money(from_account, to_account, amount)
  tx = Transaction.new

  tx.add_operation("Check balance") {
    raise "Insufficient funds" if from_account[:balance] < amount
    puts "    Balance check passed"
  }
  tx.add_operation("Debit from account") {
    from_account[:balance] -= amount
    puts "    Debited ฿#{amount} from account #{from_account[:id]}"
  }
  tx.add_rollback("Undo debit") {
    from_account[:balance] += amount
    puts "    Reversed debit for account #{from_account[:id]}"
  }
  tx.add_operation("Credit to account") {
    raise "Account frozen" if to_account[:frozen]
    to_account[:balance] += amount
    puts "    Credited ฿#{amount} to account #{to_account[:id]}"
  }
  tx.add_rollback("Undo credit") {
    to_account[:balance] -= amount if to_account[:balance] >= amount
    puts "    Reversed credit for account #{to_account[:id]}"
  }

  success = tx.execute
  puts success ? "Transfer successful!" : "Transfer failed! Status: #{tx.status}"
  success
end

acc1 = { id: "ACC001", balance: 5000 }
acc2 = { id: "ACC002", balance: 1000 }
acc3 = { id: "ACC003", balance: 2000, frozen: true }

puts "=== Successful Transfer ==="
transfer_money(acc1, acc2, 1000)
puts "ACC001 balance: #{acc1[:balance]}"
puts "ACC002 balance: #{acc2[:balance]}"

puts "\n=== Failed Transfer (frozen account) ==="
transfer_money(acc1, acc3, 500)
puts "ACC001 balance: #{acc1[:balance]} (should be unchanged)"
```

```ruby
# ข้อที่ 8: Exception Hierarchy Builder
module ExceptionBuilder
  def self.build_hierarchy(base_class, definitions)
    definitions.each do |name, config|
      klass = Class.new(base_class) do
        if config[:attributes]
          attr_reader(*config[:attributes])
          define_method(:initialize) do |msg = nil, **attrs|
            config[:attributes].each { |a| instance_variable_set("@#{a}", attrs[a]) }
            context = config[:attributes].filter_map { |a| "#{a}=#{attrs[a]}" if attrs[a] }.join(", ")
            full_msg = msg || config[:default_message] || self.class.name
            full_msg += " (#{context})" unless context.empty?
            super(full_msg)
          end
        else
          define_method(:initialize) do |msg = nil|
            super(msg || config[:default_message] || self.class.name)
          end
        end
      end

      base_class.const_set(name, klass)
    end
  end
end

class AppError < StandardError
  ExceptionBuilder.build_hierarchy(self, {
    NotFoundError: {
      default_message: "Resource not found",
      attributes: [:resource, :id]
    },
    ValidationError: {
      default_message: "Validation failed",
      attributes: [:field, :value]
    },
    AuthorizationError: {
      default_message: "Not authorized",
      attributes: [:action, :resource]
    },
    ExternalServiceError: {
      default_message: "External service failed",
      attributes: [:service, :status_code]
    }
  })
end

# ทดสอบ
begin
  raise AppError::NotFoundError.new("User not found", resource: "User", id: 999)
rescue AppError::NotFoundError => e
  puts "#{e.class}: #{e.message}"
  puts "  Resource: #{e.resource}, ID: #{e.id}"
end

begin
  raise AppError::ValidationError.new("Invalid email", field: :email, value: "bad-email")
rescue AppError::ValidationError => e
  puts "#{e.class}: #{e.message}"
  puts "  Field: #{e.field}"
end

begin
  raise AppError::AuthorizationError.new(action: :delete, resource: :admin_panel)
rescue AppError::AuthorizationError => e
  puts "#{e.class}: #{e.message}"
end
```

### ข้อที่ 9-20

```ruby
# ข้อที่ 9: Timeout wrapper
require 'timeout'

class TimeoutWrapper
  class TimeoutError < StandardError
    attr_reader :operation, :limit
    def initialize(operation, limit)
      super("Operation '#{operation}' timed out after #{limit}s")
      @operation = operation
      @limit = limit
    end
  end

  def initialize(default_timeout: 30)
    @default_timeout = default_timeout
    @timeouts = {}
  end

  def set_timeout(operation, seconds)
    @timeouts[operation] = seconds
    self
  end

  def execute(operation, timeout: nil)
    limit = timeout || @timeouts[operation] || @default_timeout

    begin
      Timeout.timeout(limit) do
        yield
      end
    rescue Timeout::Error
      raise TimeoutError.new(operation, limit)
    end
  end
end

wrapper = TimeoutWrapper.new(default_timeout: 5)
wrapper.set_timeout(:database_query, 3)
wrapper.set_timeout(:api_call, 10)

begin
  result = wrapper.execute(:database_query) do
    sleep(0.001)  # Fast operation - success
    "query result"
  end
  puts "Query result: #{result}"
rescue TimeoutWrapper::TimeoutError => e
  puts "Timeout: #{e.message}"
end

begin
  wrapper.execute(:fast_op, timeout: 0.0001) do
    sleep(1)  # Too slow!
  end
rescue TimeoutWrapper::TimeoutError => e
  puts "Timeout: #{e.message}"
end
```

```ruby
# ข้อที่ 10-20: comprehensive examples

# ข้อที่ 10: Error accumulator
class ErrorAccumulator
  attr_reader :errors

  def initialize
    @errors = []
    @warnings = []
  end

  def add_error(message, context: {})
    @errors << { type: :error, message: message, context: context, time: Time.now }
    self
  end

  def add_warning(message, context: {})
    @warnings << { type: :warning, message: message, context: context, time: Time.now }
    self
  end

  def valid?
    @errors.empty?
  end

  def raise_if_errors!
    return if valid?
    messages = @errors.map { |e| e[:message] }
    raise "Multiple errors: #{messages.join('; ')}"
  end

  def full_report
    all = (@errors + @warnings).sort_by { |e| e[:time] }
    all.map { |e| "[#{e[:type].upcase}] #{e[:message]}" }
  end
end

def bulk_validate(users)
  acc = ErrorAccumulator.new

  users.each_with_index do |user, i|
    acc.add_error("User #{i}: name is required", context: { index: i }) if user[:name].nil?
    acc.add_error("User #{i}: invalid email", context: { index: i, email: user[:email] }) if user[:email] !~ /@/
    acc.add_warning("User #{i}: no phone number", context: { index: i }) unless user[:phone]
  end

  acc
end

users = [
  { name: "Alice", email: "alice@example.com", phone: "081-111" },
  { name: nil, email: "bob@example.com" },
  { name: "Charlie", email: "bad-email" },
  { name: "Diana", email: "diana@example.com" }
]

result = bulk_validate(users)
if result.valid?
  puts "All users valid!"
else
  puts "Validation report:"
  result.full_report.each { |line| puts "  #{line}" }

  begin
    result.raise_if_errors!
  rescue => e
    puts "\nRaised: #{e.message}"
  end
end
```

```ruby
# ข้อที่ 11-20: Final error handling examples

# ข้อที่ 11: Exception policy
class ExceptionPolicy
  POLICIES = {}

  def self.define(exception_class, &handler)
    POLICIES[exception_class] = handler
  end

  def self.handle(exception)
    handler = POLICIES.find { |klass, _| exception.is_a?(klass) }&.last
    handler ? handler.call(exception) : default_handler(exception)
  end

  def self.default_handler(e)
    puts "[Unhandled] #{e.class}: #{e.message}"
  end
end

ExceptionPolicy.define(ArgumentError) { |e| puts "[ARG_ERROR] #{e.message}" }
ExceptionPolicy.define(RuntimeError) { |e| puts "[RUNTIME] #{e.message}" }
ExceptionPolicy.define(StandardError) { |e| puts "[STANDARD] #{e.class}: #{e.message}" }

[
  ArgumentError.new("bad argument"),
  RuntimeError.new("runtime error"),
  TypeError.new("type mismatch"),
  ZeroDivisionError.new("divided by 0"),
].each do |e|
  ExceptionPolicy.handle(e)
end
```

```ruby
# ข้อที่ 12-20: More patterns

# ข้อที่ 12: Fallback chain
class FallbackChain
  def initialize(*handlers)
    @handlers = handlers
  end

  def execute(*args)
    @handlers.each_with_index do |handler, i|
      begin
        result = handler.call(*args)
        puts "Handler #{i + 1} succeeded"
        return result
      rescue => e
        puts "Handler #{i + 1} failed: #{e.message}"
        raise if i == @handlers.length - 1  # last handler
      end
    end
  end
end

primary_handler = ->(id) {
  raise "Primary DB unavailable" if id % 3 == 0
  "Data from primary: #{id}"
}

replica_handler = ->(id) {
  raise "Replica unavailable" if id % 5 == 0
  "Data from replica: #{id}"
}

cache_handler = ->(id) {
  "Data from cache: #{id} (stale)"
}

chain = FallbackChain.new(primary_handler, replica_handler, cache_handler)

[1, 3, 5, 15].each do |id|
  puts "\nFetching id=#{id}:"
  begin
    result = chain.execute(id)
    puts "  Result: #{result}"
  rescue => e
    puts "  All handlers failed: #{e.message}"
  end
end
```

```ruby
# ข้อที่ 13-20: Final showcase

# ข้อที่ 13: Error boundary (React-inspired)
class ErrorBoundary
  def initialize(name: "ErrorBoundary", fallback: nil)
    @name = name
    @fallback = fallback
    @last_error = nil
  end

  def render(&content)
    content.call
  rescue => e
    @last_error = e
    puts "[#{@name}] Caught error: #{e.class}: #{e.message}"

    if @fallback
      puts "[#{@name}] Rendering fallback"
      @fallback.call(e)
    else
      "Error: #{e.message}"
    end
  end

  def last_error
    @last_error
  end
end

boundary = ErrorBoundary.new(
  name: "UserComponent",
  fallback: ->(e) { "Sorry, failed to load user: #{e.message}" }
)

def render_user(id)
  raise "User #{id} not found" if id > 3
  "User ##{id}: Alice"
end

[1, 2, 5].each do |id|
  result = boundary.render { render_user(id) }
  puts "Rendered: #{result}"
end
```

```ruby
# ข้อที่ 14-20: Comprehensive test
# ข้อที่ 14: Safe delegator
class SafeDelegator
  def initialize(target, default: nil, on_error: nil)
    @target = target
    @default = default
    @on_error = on_error
  end

  def method_missing(method, *args, &block)
    @target.send(method, *args, &block)
  rescue NoMethodError
    @default
  rescue => e
    @on_error&.call(e, method, args)
    @default
  end

  def respond_to_missing?(method, include_private = false)
    @target.respond_to?(method, include_private) || super
  end
end

class FlakyService
  def data
    raise "Random failure!" if rand < 0.5
    "Important data"
  end

  def another_method
    "Works fine"
  end
end

service = FlakyService.new
safe = SafeDelegator.new(
  service,
  default: "default_value",
  on_error: ->(e, method, _) { puts "Error in #{method}: #{e.message}" }
)

5.times do
  result = safe.data
  puts "Got: #{result}"
end

puts safe.another_method
puts safe.nonexistent_method.inspect
```

```ruby
# ข้อที่ 15-20: Exception testing helpers

# ข้อที่ 15: Custom expect_raise
def expect_raise(exception_class, message: nil)
  yield
  puts "FAIL: Expected #{exception_class} but no exception raised"
  false
rescue exception_class => e
  if message && !e.message.include?(message)
    puts "FAIL: Expected message '#{message}' but got '#{e.message}'"
    false
  else
    puts "PASS: #{exception_class} raised with: #{e.message}"
    true
  end
rescue => e
  puts "FAIL: Expected #{exception_class} but got #{e.class}"
  false
end

# ข้อที่ 16-20: More tests
expect_raise(ZeroDivisionError) { 1 / 0 }
expect_raise(ArgumentError) { Integer("abc") }
expect_raise(NoMethodError) { nil.upcase }
expect_raise(TypeError, message: "no implicit") { "a" + 1 }
expect_raise(RuntimeError) { raise "test error" }

# ข้อที่ 17: Exception matchers
class ExceptionMatcher
  def initialize(klass)
    @klass = klass
    @message_pattern = nil
    @context_check = nil
  end

  def with_message(pattern)
    @message_pattern = pattern
    self
  end

  def satisfying(&check)
    @context_check = check
    self
  end

  def match?(exception)
    return false unless exception.is_a?(@klass)
    return false if @message_pattern && exception.message !~ Regexp.new(@message_pattern)
    return false if @context_check && !@context_check.call(exception)
    true
  end
end

matcher = ExceptionMatcher.new(ArgumentError).with_message("Invalid")
e = ArgumentError.new("Invalid email format")
puts "Matches: #{matcher.match?(e)}"

matcher2 = ExceptionMatcher.new(RuntimeError).with_message("not_this")
puts "Matches: #{matcher2.match?(e)}"
```

---

## สรุปตอนที่ 14

ในตอนนี้เราได้เรียนรู้:

1. **Exception Hierarchy** - โครงสร้างจาก Exception ลงมา StandardError
2. **begin/rescue/ensure/else** - โครงสร้างพื้นฐาน, cleanup ใน ensure
3. **rescue specific** - จัดลำดับ specific ก่อน general
4. **rescue multiple** - จับหลาย exceptions พร้อมกัน
5. **message & backtrace** - ข้อมูลสำหรับ debugging
6. **raise vs fail** - convention การใช้, วิธี raise แบบต่างๆ
7. **Custom exceptions** - สร้าง hierarchy, เพิ่ม metadata
8. **retry** - retry กับ backoff, max attempts
9. **inline rescue** - ใน method, ใน expression
10. **Global handlers** - at_exit, trap
11. **Defensive patterns** - Guard clauses, Null Object, Result Object, Fail Fast
12. **Advanced patterns** - Circuit Breaker, Transaction, Fallback Chain

---

*ตอนต่อไป: Part 15 - File I/O*
