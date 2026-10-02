# Part 14: Error Handling - Steps 281-300

## บทนำ

Error Handling (การจัดการข้อผิดพลาด) เป็นส่วนสำคัญของการเขียนโปรแกรมที่ดี Ruby มีระบบ exception handling ที่ทรงพลังผ่านคีย์เวิร์ด `begin`, `rescue`, `ensure`, `else`, `raise`

---

## Step 281: Exception Hierarchy ใน Ruby

```
Exception
├── ScriptError
│   ├── LoadError
│   ├── NotImplementedError
│   └── SyntaxError
├── SignalException
│   └── Interrupt
├── SystemExit
└── StandardError (ส่วนใหญ่ที่เราจัดการ)
    ├── ArgumentError
    ├── EncodingError
    ├── IOError
    │   └── EOFError
    ├── IndexError
    │   ├── KeyError
    │   └── StopIteration
    ├── Math::DomainError
    ├── NameError
    │   └── NoMethodError
    ├── RangeError
    │   └── FloatDomainError
    ├── RegexpError
    ├── RuntimeError  (raise "message" ใช้อันนี้)
    ├── SystemCallError
    ├── ThreadError
    ├── TypeError
    ├── ZeroDivisionError
    └── FrozenError
```

```ruby
# ทดสอบ exceptions ต่างๆ

# ZeroDivisionError
begin
  1 / 0
rescue ZeroDivisionError => e
  puts "ZeroDivisionError: #{e.message}"
end

# NoMethodError
begin
  nil.upcase
rescue NoMethodError => e
  puts "NoMethodError: #{e.message}"
end

# TypeError
begin
  "hello" + 5
rescue TypeError => e
  puts "TypeError: #{e.message}"
end

# NameError
begin
  puts undefined_variable
rescue NameError => e
  puts "NameError: #{e.message}"
end

# ArgumentError
begin
  Integer("abc")
rescue ArgumentError => e
  puts "ArgumentError: #{e.message}"
end

# IndexError
begin
  [1, 2, 3].fetch(10)
rescue IndexError => e
  puts "IndexError: #{e.message}"
end

# KeyError
begin
  { a: 1 }.fetch(:z)
rescue KeyError => e
  puts "KeyError: #{e.message}"
end
```

---

## Step 282: begin/rescue/ensure/else/raise

### โครงสร้างพื้นฐาน

```ruby
begin
  # โค้ดที่อาจเกิด error
  result = 10 / 2
rescue ZeroDivisionError => e
  # จัดการ error
  puts "เกิดข้อผิดพลาด: #{e.message}"
else
  # ทำงานเมื่อไม่มี error (optional)
  puts "ผลลัพธ์: #{result}"
ensure
  # ทำงานเสมอไม่ว่าจะเกิด error หรือไม่ (optional)
  puts "จบการทำงาน"
end
```

### ตัวอย่างสมบูรณ์

```ruby
def divide(a, b)
  begin
    result = a / b
  rescue ZeroDivisionError => e
    puts "Error: ไม่สามารถหารด้วยศูนย์ได้"
    result = nil
  rescue TypeError => e
    puts "Error: ประเภทข้อมูลไม่ถูกต้อง - #{e.message}"
    result = nil
  else
    puts "คำนวณสำเร็จ: #{a} / #{b} = #{result}"
  ensure
    puts "--- สิ้นสุดการคำนวณ ---"
  end
  
  result
end

puts divide(10, 2).inspect
puts divide(10, 0).inspect
puts divide("10", 2).inspect
```

### rescue หลาย Exceptions ในบรรทัดเดียว

```ruby
def process_input(input)
  begin
    value = Integer(input)
    result = 100 / value
    puts "ผลลัพธ์: #{result}"
  rescue ArgumentError, TypeError => e
    puts "Input ไม่ถูกต้อง: #{e.message}"
  rescue ZeroDivisionError => e
    puts "ห้ามหารด้วย 0"
  end
end

process_input("10")     # => ผลลัพธ์: 10
process_input("0")      # => ห้ามหารด้วย 0
process_input("abc")    # => Input ไม่ถูกต้อง
process_input(nil)      # => Input ไม่ถูกต้อง
```

---

## Step 283: rescue Specific Exceptions

### ลำดับการ rescue

```ruby
def fetch_data(url)
  begin
    # จำลองการเชื่อมต่อ
    raise Timeout::Error, "Connection timeout" if url.include?("timeout")
    raise SocketError, "Cannot connect" if url.include?("error")
    raise ArgumentError, "URL ไม่ถูกต้อง" if url.empty?
    
    "ข้อมูลจาก #{url}"
  rescue ArgumentError => e
    # จัดการก่อน - error เฉพาะเจาะจงที่สุด
    puts "URL Error: #{e.message}"
    nil
  rescue SocketError => e
    puts "Network Error: #{e.message}"
    nil
  rescue StandardError => e
    # จัดการทั่วไป - ต้องอยู่หลัง specific ones
    puts "Unexpected Error: #{e.class} - #{e.message}"
    nil
  end
end

puts fetch_data("https://example.com").inspect
puts fetch_data("https://timeout.com").inspect
puts fetch_data("https://error.com").inspect
puts fetch_data("").inspect
```

### rescue StandardError vs rescue Exception

```ruby
# StandardError - ส่วนใหญ่ที่เราจัดการ
begin
  raise RuntimeError, "runtime error"
rescue StandardError => e
  puts "StandardError จับได้: #{e.message}"
end

# Exception - จับทุกอย่าง (ระวัง!)
begin
  raise Interrupt, "user interrupted"
rescue Exception => e
  puts "Exception จับได้: #{e.class} - #{e.message}"
end

# ไม่ระบุ class - จับ RuntimeError เป็น default
begin
  raise "something went wrong"
rescue => e
  puts "Default rescue: #{e.class} - #{e.message}"
end
```

---

## Step 284: Custom Exceptions

```ruby
# สร้าง custom exception
class AppError < StandardError
  attr_reader :code
  
  def initialize(message = "เกิดข้อผิดพลาด", code: 500)
    super(message)
    @code = code
  end
end

class ValidationError < AppError
  attr_reader :field, :errors
  
  def initialize(field, errors = [])
    @field = field
    @errors = errors
    super("Validation ล้มเหลว: #{errors.join(', ')}", code: 422)
  end
  
  def to_s
    "ValidationError(#{@field}): #{@errors.join(', ')}"
  end
end

class NotFoundError < AppError
  attr_reader :resource, :id
  
  def initialize(resource, id = nil)
    @resource = resource
    @id = id
    msg = id ? "ไม่พบ #{resource} id=#{id}" : "ไม่พบ #{resource}"
    super(msg, code: 404)
  end
end

class AuthenticationError < AppError
  def initialize(message = "ยืนยันตัวตนไม่สำเร็จ")
    super(message, code: 401)
  end
end

class AuthorizationError < AppError
  attr_reader :action, :resource
  
  def initialize(action, resource)
    @action = action
    @resource = resource
    super("ไม่มีสิทธิ์ #{action} #{resource}", code: 403)
  end
end

# ใช้งาน
def find_user(id)
  raise NotFoundError.new("User", id) if id > 100
  { id: id, name: "ผู้ใช้ #{id}" }
end

def update_user(current_user, user_id, data)
  raise AuthorizationError.new("แก้ไข", "User") unless current_user[:admin]
  raise ValidationError.new("email", ["รูปแบบอีเมลไม่ถูกต้อง"]) unless data[:email]&.include?("@")
  
  "อัปเดตสำเร็จ"
end

# ทดสอบ
begin
  user = find_user(200)
rescue NotFoundError => e
  puts "#{e.class}: #{e.message} (code: #{e.code})"
end

begin
  update_user({ admin: false }, 1, { email: "new@example.com" })
rescue AuthorizationError => e
  puts "#{e.class}: #{e.message} (code: #{e.code})"
end

begin
  update_user({ admin: true }, 1, { email: "invalid-email" })
rescue ValidationError => e
  puts "#{e.class}: #{e.to_s}"
  puts "Field: #{e.field}, Errors: #{e.errors.inspect}"
end
```

### Exception Hierarchy ของ Custom Errors

```ruby
# Application error hierarchy
module MyApp
  class Error < StandardError; end
  
  module Database
    class Error < MyApp::Error; end
    class ConnectionError < Error; end
    class QueryError < Error; end
    class TransactionError < Error; end
  end
  
  module Network
    class Error < MyApp::Error; end
    class TimeoutError < Error; end
    class ConnectionRefusedError < Error; end
  end
  
  module Business
    class Error < MyApp::Error; end
    class InsufficientFundsError < Error
      attr_reader :amount, :balance
      def initialize(amount, balance)
        @amount = amount
        @balance = balance
        super("ยอดเงินไม่พอ: ต้องการ #{amount}, มี #{balance}")
      end
    end
    class OrderLimitExceededError < Error; end
  end
end

# ใช้งาน
def transfer_money(from_balance, amount)
  raise MyApp::Business::InsufficientFundsError.new(amount, from_balance) if amount > from_balance
  from_balance - amount
end

begin
  new_balance = transfer_money(1000, 1500)
rescue MyApp::Business::InsufficientFundsError => e
  puts "ธุรกรรมล้มเหลว: #{e.message}"
  puts "ขาดอยู่: #{e.amount - e.balance} บาท"
rescue MyApp::Business::Error => e
  puts "Business Error: #{e.message}"
rescue MyApp::Error => e
  puts "App Error: #{e.message}"
end
```

---

## Step 285: retry

`retry` ลองดำเนินการใหม่เมื่อเกิด error

```ruby
def fetch_with_retry(url, max_attempts = 3)
  attempts = 0
  
  begin
    attempts += 1
    puts "พยายามครั้งที่ #{attempts}..."
    
    # จำลองการเชื่อมต่อที่ล้มเหลวในครั้งแรกๆ
    raise "Connection failed" if attempts < 3
    
    "ข้อมูลจาก #{url}"
  rescue RuntimeError => e
    if attempts < max_attempts
      puts "Error: #{e.message}, รอ #{attempts} วินาที..."
      sleep(attempts * 0.1)  # exponential backoff
      retry
    else
      puts "ล้มเหลวหลังจาก #{max_attempts} ครั้ง"
      raise
    end
  end
end

begin
  result = fetch_with_retry("https://example.com")
  puts "สำเร็จ: #{result}"
rescue RuntimeError => e
  puts "สุดท้ายล้มเหลว: #{e.message}"
end
```

### retry กับ Exponential Backoff

```ruby
module Retryable
  def self.with_retry(max_attempts: 3, wait: 1, exceptions: [StandardError], &block)
    attempts = 0
    
    begin
      attempts += 1
      block.call(attempts)
    rescue *exceptions => e
      if attempts < max_attempts
        wait_time = wait * (2 ** (attempts - 1))  # exponential backoff
        puts "Attempt #{attempts} failed: #{e.message}. Retrying in #{wait_time}s..."
        sleep(wait_time * 0.01)  # ลดเวลาสำหรับ demo
        retry
      else
        puts "All #{max_attempts} attempts failed"
        raise
      end
    end
  end
  
  def self.with_circuit_breaker(max_failures: 3, reset_timeout: 60, &block)
    @failures ||= 0
    @last_failure ||= nil
    
    # ตรวจสอบว่า circuit เปิดอยู่
    if @failures >= max_failures
      if @last_failure && Time.now - @last_failure > reset_timeout
        @failures = 0
        puts "Circuit reset"
      else
        raise "Circuit breaker open! รอ #{reset_timeout - (Time.now - @last_failure).to_i} วินาที"
      end
    end
    
    begin
      result = block.call
      @failures = 0  # success - reset failures
      result
    rescue => e
      @failures += 1
      @last_failure = Time.now
      raise
    end
  end
end

# ทดสอบ retry
call_count = 0
begin
  Retryable.with_retry(max_attempts: 3, wait: 0.1) do |attempt|
    call_count += 1
    raise "Temporary error" if call_count < 3
    "Success on attempt #{attempt}"
  end
rescue => e
  puts "Final error: #{e.message}"
else
  puts "Result: completed successfully"
end
```

---

## Step 286: raise vs fail

ทั้ง `raise` และ `fail` เหมือนกันทุกประการ ใช้ตามสไตล์ของทีม

```ruby
# raise - ใช้บ่อยกว่า
raise "Something went wrong"
raise ArgumentError, "Invalid argument"
raise ArgumentError.new("Invalid argument")

# fail - semantically เหมือน raise แต่บาง style guide ใช้ใน rescue block
def process(value)
  fail ArgumentError, "value ต้องเป็น Integer" unless value.is_a?(Integer)
  value * 2
end

# re-raise - raise โดยไม่มี argument ใน rescue block
def safe_process(value)
  begin
    process(value)
  rescue ArgumentError => e
    puts "จัดการ error: #{e.message}"
    raise  # re-raise exception เดิม
  end
end

begin
  safe_process("not an integer")
rescue ArgumentError => e
  puts "ถูก re-raise: #{e.message}"
end
```

### raise กับ custom message

```ruby
class Config
  attr_reader :settings
  
  def initialize(settings = {})
    @settings = settings
  end
  
  def get(key)
    unless @settings.key?(key)
      raise KeyError, "Configuration key '#{key}' ไม่พบ (มีเฉพาะ: #{@settings.keys.join(', ')})"
    end
    @settings[key]
  end
  
  def require_all!(*keys)
    missing = keys - @settings.keys
    unless missing.empty?
      raise "Configuration ขาด: #{missing.join(', ')}"
    end
    self
  end
end

config = Config.new(host: "localhost", port: 3000)

begin
  config.require_all!(:host, :port, :database)
rescue RuntimeError => e
  puts "Config Error: #{e.message}"
end

begin
  config.get(:unknown_key)
rescue KeyError => e
  puts "Key Error: #{e.message}"
end
```

---

## Step 287: Exception Message และ Backtrace

```ruby
def level3
  raise RuntimeError, "ข้อผิดพลาดที่ level 3"
end

def level2
  level3
end

def level1
  level2
end

begin
  level1
rescue RuntimeError => e
  puts "Message: #{e.message}"
  puts "Class: #{e.class}"
  puts "\nBacktrace (5 บรรทัดแรก):"
  e.backtrace.first(5).each { |line| puts "  #{line}" }
  
  # ใช้ cause เพื่อดู original exception
  puts "\nCause: #{e.cause.inspect}"
end
```

### Exception Wrapping

```ruby
class DatabaseError < StandardError
  def initialize(msg, original: nil)
    super(msg)
    @original = original
  end
  
  def original_error
    @original
  end
end

def query_database(sql)
  begin
    raise "MySQL: Table 'users' doesn't exist" if sql.include?("users")
    "ผลลัพธ์จาก query"
  rescue => e
    raise DatabaseError.new(
      "Database query ล้มเหลว: #{sql}",
      original: e
    )
  end
end

begin
  query_database("SELECT * FROM users")
rescue DatabaseError => e
  puts "DB Error: #{e.message}"
  puts "Original: #{e.original_error.message}"
end
```

---

## Step 288: Global Exception Handlers

```ruby
# at_exit handler
at_exit do
  if $!
    puts "\n=== Program crashed! ==="
    puts "Error: #{$!.class}: #{$!.message}"
  else
    puts "\n=== Program ended normally ==="
  end
end

# Exception handler ด้วย Thread
Thread.current.report_on_exception = true

# Global rescue ด้วย method
def safe_execute(operation_name, &block)
  block.call
rescue ArgumentError => e
  puts "[#{operation_name}] Input Error: #{e.message}"
rescue RuntimeError => e
  puts "[#{operation_name}] Runtime Error: #{e.message}"
rescue StandardError => e
  puts "[#{operation_name}] Unexpected Error: #{e.class} - #{e.message}"
  raise  # re-raise ถ้าจำเป็น
end

safe_execute("การคำนวณ") do
  raise ArgumentError, "ค่าไม่ถูกต้อง"
end

safe_execute("การประมวลผล") do
  raise RuntimeError, "เกิดข้อผิดพลาดทั่วไป"
end

safe_execute("ปกติ") do
  puts "ทำงานปกติ"
end
```

---

## Step 289: rescue ใน Methods (Inline)

```ruby
# rescue ใน method body โดยตรง (ไม่ต้องมี begin)
def safe_divide(a, b)
  a / b
rescue ZeroDivisionError
  puts "ไม่สามารถหารด้วยศูนย์"
  0
end

def parse_json(json_string)
  require 'json'
  JSON.parse(json_string)
rescue JSON::ParserError => e
  puts "JSON ไม่ถูกต้อง: #{e.message}"
  {}
end

def fetch_user(id)
  raise NotFoundError, "ไม่พบผู้ใช้" unless id > 0
  { id: id, name: "ผู้ใช้ #{id}" }
rescue NotFoundError
  nil
end

puts safe_divide(10, 2)   # => 5
puts safe_divide(10, 0)   # => 0
puts parse_json('{"key": "value"}').inspect
puts parse_json('invalid json').inspect
```

### rescue ใน Block

```ruby
# rescue ใน do...end block
[1, 0, 2, -1, 3].each do |n|
  result = 10 / n
  puts "10 / #{n} = #{result}"
rescue ZeroDivisionError
  puts "10 / #{n} = Error (หารด้วย 0)"
end
```

---

## Step 290: Defensive Programming

```ruby
# Guard clauses
def process_payment(amount, card_number, cvv)
  # Guard clauses ที่ด้านบนสุด
  raise ArgumentError, "จำนวนเงินต้องมากกว่า 0" if amount <= 0
  raise ArgumentError, "หมายเลขบัตรไม่ถูกต้อง" unless card_number.to_s.length == 16
  raise ArgumentError, "CVV ไม่ถูกต้อง" unless cvv.to_s.match?(/^\d{3,4}$/)
  
  # Business logic
  puts "ชำระเงิน #{amount} บาท ด้วยบัตร ****#{card_number.to_s[-4..]}"
  { success: true, amount: amount }
end

begin
  process_payment(-100, "1234567890123456", "123")
rescue ArgumentError => e
  puts "Error: #{e.message}"
end

begin
  process_payment(1000, "12345", "123")
rescue ArgumentError => e
  puts "Error: #{e.message}"
end

begin
  result = process_payment(500, "1234567890123456", "456")
  puts "สำเร็จ: #{result.inspect}"
rescue ArgumentError => e
  puts "Error: #{e.message}"
end
```

### Result Pattern (No Exceptions)

```ruby
class Result
  attr_reader :value, :error
  
  def initialize(success:, value: nil, error: nil)
    @success = success
    @value = value
    @error = error
  end
  
  def success?
    @success
  end
  
  def failure?
    !@success
  end
  
  def self.ok(value)
    new(success: true, value: value)
  end
  
  def self.err(error)
    new(success: false, error: error)
  end
  
  def map(&block)
    return self if failure?
    begin
      Result.ok(block.call(@value))
    rescue => e
      Result.err(e.message)
    end
  end
  
  def flat_map(&block)
    return self if failure?
    block.call(@value)
  end
  
  def on_success(&block)
    block.call(@value) if success?
    self
  end
  
  def on_failure(&block)
    block.call(@error) if failure?
    self
  end
  
  def or_else(default)
    success? ? @value : default
  end
  
  def to_s
    success? ? "Ok(#{@value})" : "Err(#{@error})"
  end
end

# ใช้งาน
def divide_safe(a, b)
  return Result.err("หารด้วย 0 ไม่ได้") if b == 0
  Result.ok(a.to_f / b)
end

def sqrt_safe(n)
  return Result.err("ไม่สามารถหารากที่สองของจำนวนลบ") if n < 0
  Result.ok(Math.sqrt(n))
end

# Chain operations
result = divide_safe(100, 4)
  .map { |v| v * 2 }
  .flat_map { |v| sqrt_safe(v) }
  .map { |v| v.round(4) }

result.on_success { |v| puts "ผลลัพธ์: #{v}" }
      .on_failure { |e| puts "Error: #{e}" }

# Error case
divide_safe(10, 0)
  .on_success { |v| puts "ผลลัพธ์: #{v}" }
  .on_failure { |e| puts "Error: #{e}" }

puts divide_safe(10, 0).or_else("N/A")
```

---

## Step 291-300: ตัวอย่างขั้นสูง

### Error Handling ใน Web Application Context

```ruby
module ErrorHandling
  class AppError < StandardError
    attr_reader :status_code, :details
    
    def initialize(message, status_code: 500, details: nil)
      super(message)
      @status_code = status_code
      @details = details
    end
    
    def to_h
      {
        error: {
          type: self.class.name,
          message: message,
          status: @status_code,
          details: @details
        }
      }
    end
    
    def to_json
      require 'json'
      to_h.to_json
    end
  end
  
  class NotFoundError < AppError
    def initialize(resource, id = nil)
      msg = id ? "#{resource} ##{id} ไม่พบ" : "#{resource} ไม่พบ"
      super(msg, status_code: 404)
    end
  end
  
  class ValidationError < AppError
    attr_reader :field_errors
    
    def initialize(field_errors = {})
      @field_errors = field_errors
      messages = field_errors.map { |f, e| "#{f}: #{e.join(', ')}" }.join('; ')
      super(messages, status_code: 422, details: field_errors)
    end
  end
  
  class UnauthorizedError < AppError
    def initialize(message = "กรุณาเข้าสู่ระบบ")
      super(message, status_code: 401)
    end
  end
  
  class ForbiddenError < AppError
    def initialize(message = "ไม่มีสิทธิ์เข้าถึง")
      super(message, status_code: 403)
    end
  end
  
  class RateLimitError < AppError
    attr_reader :retry_after
    
    def initialize(retry_after = 60)
      @retry_after = retry_after
      super("คำขอมากเกินไป กรุณารอ #{retry_after} วินาที", status_code: 429)
    end
  end
  
  # Error handler middleware
  module Handler
    def self.handle(context = {}, &block)
      begin
        block.call
      rescue ValidationError => e
        puts "422 Validation Error: #{e.message}"
        puts "Details: #{e.field_errors.inspect}"
        { success: false, status: 422, errors: e.field_errors }
      rescue NotFoundError => e
        puts "404 Not Found: #{e.message}"
        { success: false, status: 404, message: e.message }
      rescue UnauthorizedError => e
        puts "401 Unauthorized: #{e.message}"
        { success: false, status: 401, message: e.message }
      rescue ForbiddenError => e
        puts "403 Forbidden: #{e.message}"
        { success: false, status: 403, message: e.message }
      rescue RateLimitError => e
        puts "429 Rate Limit: #{e.message}"
        { success: false, status: 429, retry_after: e.retry_after }
      rescue AppError => e
        puts "#{e.status_code} App Error: #{e.message}"
        { success: false, status: e.status_code, message: e.message }
      rescue StandardError => e
        puts "500 Internal Error: #{e.message}"
        { success: false, status: 500, message: "Internal server error" }
      end
    end
  end
end

# ทดสอบ
include ErrorHandling

puts "=== ทดสอบ Error Handlers ==="

result1 = Handler.handle do
  raise ValidationError.new({
    name: ["ห้ามว่าง"],
    email: ["รูปแบบไม่ถูกต้อง", "ต้องไม่ซ้ำ"]
  })
end
puts "Result: #{result1}\n\n"

result2 = Handler.handle do
  raise NotFoundError.new("User", 42)
end
puts "Result: #{result2}\n\n"

result3 = Handler.handle do
  raise RateLimitError.new(120)
end
puts "Result: #{result3}\n\n"

result4 = Handler.handle do
  # ทำงานปกติ
  { success: true, data: "ข้อมูล" }
end
puts "Result: #{result4}"
```

### Exception Safety ใน Transaction

```ruby
class Database
  def initialize
    @data = {}
    @transaction_log = []
  end
  
  def transaction
    backup = @data.dup
    @transaction_log << "BEGIN"
    
    begin
      result = yield self
      @transaction_log << "COMMIT"
      puts "Transaction committed"
      result
    rescue => e
      @data = backup  # Rollback
      @transaction_log << "ROLLBACK: #{e.message}"
      puts "Transaction rolled back: #{e.message}"
      raise
    end
  end
  
  def set(key, value)
    @data[key] = value
    puts "  SET #{key} = #{value}"
  end
  
  def get(key)
    @data[key]
  end
  
  def delete(key)
    @data.delete(key)
    puts "  DELETE #{key}"
  end
  
  def log
    @transaction_log
  end
  
  def to_s
    "DB: #{@data.inspect}"
  end
end

db = Database.new

# Transaction สำเร็จ
db.transaction do |db|
  db.set(:account_a, 1000)
  db.set(:account_b, 500)
end

puts "\nหลัง transaction 1: #{db}"

# Transaction ที่ล้มเหลว
begin
  db.transaction do |db|
    db.set(:account_a, 900)
    db.set(:account_b, 600)
    raise "ยอดเงินไม่สมดุล!"
  end
rescue RuntimeError => e
  puts "Error: #{e.message}"
end

puts "\nหลัง transaction 2 (failed): #{db}"
puts "Transaction log: #{db.log.inspect}"
```

---

## แบบฝึกหัด Part 14 (20 ข้อ)

### ระดับง่าย (ข้อ 1-7)

**ข้อ 1:** เขียนฟังก์ชัน safe_sqrt ที่จัดการ negative number

```ruby
# เฉลย
def safe_sqrt(n)
  raise ArgumentError, "ไม่สามารถหารากที่สองของ #{n}" if n < 0
  Math.sqrt(n)
rescue ArgumentError => e
  puts "Error: #{e.message}"
  nil
end

puts safe_sqrt(16)   # => 4.0
puts safe_sqrt(-4).inspect  # => nil
puts safe_sqrt(0)    # => 0.0
```

**ข้อ 2:** เขียน parse_integer ที่จัดการ input ที่ไม่ใช่ตัวเลข

```ruby
# เฉลย
def parse_integer(input, default: nil)
  Integer(input)
rescue ArgumentError, TypeError => e
  puts "Warning: '#{input}' ไม่ใช่จำนวนเต็ม, ใช้ค่า default: #{default}"
  default
end

puts parse_integer("42")          # => 42
puts parse_integer("abc")         # => nil (with warning)
puts parse_integer("3.14")        # => nil (with warning)
puts parse_integer(nil, default: 0) # => 0 (with warning)
```

**ข้อ 3:** สร้าง FileReader class ที่จัดการ errors ต่างๆ

```ruby
# เฉลย
class FileReader
  def initialize(file_path)
    @file_path = file_path
  end
  
  def read
    File.read(@file_path)
  rescue Errno::ENOENT => e
    raise "ไม่พบไฟล์: #{@file_path}"
  rescue Errno::EACCES => e
    raise "ไม่มีสิทธิ์อ่านไฟล์: #{@file_path}"
  rescue => e
    raise "ไม่สามารถอ่านไฟล์ได้: #{e.message}"
  end
  
  def read_lines
    read.split("\n")
  rescue => e
    puts "Error reading lines: #{e.message}"
    []
  end
  
  def read_safe
    read
  rescue => e
    puts "Error: #{e.message}"
    nil
  end
end

reader = FileReader.new("/nonexistent/file.txt")
content = reader.read_safe
puts content.inspect  # => nil
lines = reader.read_lines
puts lines.inspect    # => []
```

**ข้อ 4:** สร้าง custom exceptions สำหรับ Banking application

```ruby
# เฉลย
module BankingErrors
  class BankError < StandardError; end
  class InsufficientFundsError < BankError
    attr_reader :balance, :requested
    def initialize(balance, requested)
      @balance = balance
      @requested = requested
      super("ยอดเงินไม่พอ: มี #{balance} บาท ต้องการ #{requested} บาท")
    end
  end
  class AccountFrozenError < BankError
    def initialize
      super("บัญชีถูกระงับการใช้งาน")
    end
  end
  class DailyLimitExceededError < BankError
    attr_reader :limit, :attempted
    def initialize(limit, attempted)
      @limit = limit
      @attempted = attempted
      super("เกินวงเงินต่อวัน: วงเงิน #{limit} บาท, พยายามถอน #{attempted} บาท")
    end
  end
end

include BankingErrors

class BankAccount
  include BankingErrors
  
  DAILY_LIMIT = 50000
  
  attr_reader :balance
  
  def initialize(initial)
    @balance = initial
    @frozen = false
    @daily_withdrawn = 0
  end
  
  def withdraw(amount)
    raise AccountFrozenError if @frozen
    raise InsufficientFundsError.new(@balance, amount) if amount > @balance
    raise DailyLimitExceededError.new(DAILY_LIMIT, @daily_withdrawn + amount) if @daily_withdrawn + amount > DAILY_LIMIT
    
    @balance -= amount
    @daily_withdrawn += amount
    amount
  end
  
  def freeze!
    @frozen = true
  end
end

acc = BankAccount.new(10000)

begin
  acc.withdraw(20000)
rescue InsufficientFundsError => e
  puts "Error: #{e.message}"
  puts "ขาดอยู่: #{e.requested - e.balance} บาท"
end

acc.freeze!
begin
  acc.withdraw(100)
rescue AccountFrozenError => e
  puts "Error: #{e.message}"
end
```

**ข้อ 5-7:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 5: สร้าง retry mechanism สำหรับ API calls
# ข้อ 6: สร้าง validation framework ด้วย custom exceptions
# ข้อ 7: implement Result/Either monad
```

### ระดับกลาง (ข้อ 8-14)

**ข้อ 8:** สร้าง Error Logger ที่บันทึก exceptions

```ruby
# เฉลย
class ErrorLogger
  attr_reader :errors
  
  def initialize
    @errors = []
    @handlers = {}
  end
  
  def log(error, context = {})
    entry = {
      type: error.class.name,
      message: error.message,
      backtrace: error.backtrace&.first(3),
      context: context,
      timestamp: Time.now
    }
    @errors << entry
    notify_handlers(error, entry)
    entry
  end
  
  def on(error_class, &block)
    @handlers[error_class] = block
  end
  
  def wrap(context = {}, &block)
    block.call
  rescue => e
    log(e, context)
    raise
  end
  
  def statistics
    {
      total: @errors.length,
      by_type: @errors.group_by { |e| e[:type] }.transform_values(&:length)
    }
  end
  
  def recent(n = 10)
    @errors.last(n)
  end
  
  private
  
  def notify_handlers(error, entry)
    @handlers.each do |error_class, handler|
      handler.call(entry) if error.is_a?(error_class)
    end
  end
end

logger = ErrorLogger.new

logger.on(ArgumentError) { |e| puts "ALERT: ArgumentError - #{e[:message]}" }
logger.on(RuntimeError) { |e| puts "ALERT: RuntimeError - #{e[:message]}" }

begin
  logger.wrap(operation: "calculation") do
    raise ArgumentError, "Invalid input"
  end
rescue
end

begin
  logger.wrap(operation: "process") do
    raise RuntimeError, "Process failed"
  end
rescue
end

puts "\nStatistics: #{logger.statistics.inspect}"
puts "\nRecent errors:"
logger.recent.each do |e|
  puts "  #{e[:type]}: #{e[:message]} (#{e[:context].inspect})"
end
```

**ข้อ 9-14:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 9: สร้าง Circuit Breaker pattern
# ข้อ 10: สร้าง Fault Tolerant Queue
# ข้อ 11: สร้าง Safe Calculator ที่จัดการทุก error
# ข้อ 12: สร้าง Config Validator ด้วย custom exceptions
# ข้อ 13: สร้าง Database Transaction handler
# ข้อ 14: สร้าง HTTP Client ที่มี error handling สมบูรณ์
```

### ระดับยาก (ข้อ 15-20)

**ข้อ 15:** สร้าง Error Aggregator ที่รวม errors หลายอัน

```ruby
# เฉลย
class ErrorAggregator
  class AggregateError < StandardError
    attr_reader :errors
    
    def initialize(errors)
      @errors = errors
      message = "#{errors.length} errors occurred:\n" +
                errors.map.with_index(1) { |e, i| "  #{i}. #{e.class}: #{e.message}" }.join("\n")
      super(message)
    end
  end
  
  def initialize
    @errors = []
  end
  
  def collect(&block)
    begin
      block.call
    rescue => e
      @errors << e
    end
    self
  end
  
  def raise_if_errors!
    raise AggregateError.new(@errors) unless @errors.empty?
    self
  end
  
  def any_errors?
    !@errors.empty?
  end
  
  def errors
    @errors.dup
  end
end

def validate_user(params)
  aggregator = ErrorAggregator.new
  
  aggregator
    .collect { raise ArgumentError, "ชื่อห้ามว่าง" if params[:name].to_s.empty? }
    .collect { raise ArgumentError, "อีเมลไม่ถูกต้อง" unless params[:email].to_s.include?("@") }
    .collect { raise ArgumentError, "อายุต้องระหว่าง 18-120" unless params[:age].to_i.between?(18, 120) }
    .collect { raise ArgumentError, "รหัสผ่านต้องมีอย่างน้อย 8 ตัว" if params[:password].to_s.length < 8 }
  
  begin
    aggregator.raise_if_errors!
  rescue ErrorAggregator::AggregateError => e
    puts "Validation ล้มเหลว:"
    e.errors.each { |err| puts "  - #{err.message}" }
    return false
  end
  
  true
end

result = validate_user({
  name: "",
  email: "not-an-email",
  age: 15,
  password: "short"
})
puts "\nValid? #{result}"

result2 = validate_user({
  name: "สมชาย",
  email: "somchai@example.com",
  age: 25,
  password: "securepassword123"
})
puts "Valid? #{result2}"
```

**ข้อ 16-20:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 16: สร้าง Saga pattern สำหรับ distributed transactions
# ข้อ 17: สร้าง Error Recovery middleware
# ข้อ 18: สร้าง Supervisory restart mechanism
# ข้อ 19: สร้าง Bulkhead pattern
# ข้อ 20: สร้าง Full-featured error handling framework
```

---

## สรุป Part 14

ใน Part นี้ เราได้เรียนรู้:

1. **Exception Hierarchy** - ลำดับชั้นของ exceptions ใน Ruby
2. **begin/rescue/ensure/else** - โครงสร้างจัดการ errors
3. **rescue Specific Exceptions** - จัดการ exceptions เฉพาะเจาะจง
4. **Custom Exceptions** - สร้าง exception classes เอง
5. **retry** - ลองใหม่เมื่อเกิด error
6. **raise vs fail** - สองวิธีการ raise exception
7. **Exception Message และ Backtrace** - ดูรายละเอียด error
8. **Global Exception Handlers** - จัดการ errors ระดับ global
9. **rescue ใน Methods** - inline rescue
10. **Defensive Programming** - เขียนโค้ดป้องกัน errors

**ต่อไป:** Part 15 - File I/O
