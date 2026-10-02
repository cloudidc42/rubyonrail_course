# Part 15: File I/O - Steps 301-320

## บทนำ

File I/O (Input/Output) เป็นส่วนสำคัญในการพัฒนาโปรแกรม Ruby มีเครื่องมือที่ทรงพลังสำหรับอ่านและเขียนไฟล์ ทำงานกับ CSV, JSON, YAML และจัดการ directory

---

## Step 301: File.open, File.read, File.write

### การอ่านไฟล์

```ruby
# วิธีที่ 1: File.read - อ่านไฟล์ทั้งหมดเป็น String
content = File.read("/etc/hostname") rescue "ไม่พบไฟล์"
puts content

# วิธีที่ 2: File.readlines - อ่านเป็น Array ของบรรทัด
lines = File.readlines("/etc/hosts") rescue []
lines.first(3).each { |line| puts line.chomp }

# วิธีที่ 3: File.open กับ block
File.open("example.txt", "w") do |file|
  file.puts "บรรทัดที่ 1"
  file.puts "บรรทัดที่ 2"
  file.puts "บรรทัดที่ 3"
end

# อ่านไฟล์ที่เพิ่งสร้าง
File.open("example.txt", "r") do |file|
  file.each_line do |line|
    puts line.chomp
  end
end

# วิธีที่ 4: File.write - เขียนไฟล์ (สร้างใหม่หรือทับของเดิม)
bytes_written = File.write("output.txt", "Hello, Ruby!\nสวัสดี Ruby!")
puts "เขียน #{bytes_written} bytes"
```

### File.open กับ Modes

```ruby
# r  - อ่านอย่างเดียว (default)
# w  - เขียนอย่างเดียว (ล้างเนื้อหาเดิม)
# a  - append (ต่อท้าย)
# r+ - อ่านและเขียน
# w+ - อ่านและเขียน (ล้างเนื้อหาเดิม)
# a+ - อ่านและเขียน (ต่อท้าย)
# b  - binary mode (เช่น rb, wb)

# เขียนไฟล์ใหม่
File.open("test.txt", "w") do |f|
  f.write("บรรทัดแรก\n")
  f.write("บรรทัดสอง\n")
end

# ต่อท้ายไฟล์ (append)
File.open("test.txt", "a") do |f|
  f.puts "บรรทัดที่สาม (ต่อท้าย)"
end

# อ่านทั้งหมด
puts File.read("test.txt")

# เขียน binary
data = [1, 2, 3, 255].pack("C*")
File.open("binary.bin", "wb") { |f| f.write(data) }

# อ่าน binary
binary_data = File.open("binary.bin", "rb") { |f| f.read }
puts binary_data.unpack("C*").inspect  # => [1, 2, 3, 255]
```

---

## Step 302: Reading Line by Line

```ruby
# อ่านทีละบรรทัด - ประหยัดหน่วยความจำสำหรับไฟล์ใหญ่

# วิธีที่ 1: each_line
File.open("test.txt") do |file|
  file.each_line.with_index(1) do |line, num|
    puts "#{num}: #{line.chomp}"
  end
end

# วิธีที่ 2: gets
File.open("test.txt") do |file|
  while (line = file.gets)
    puts line.chomp
  end
end

# วิธีที่ 3: foreach (ไม่ต้องเปิด/ปิดไฟล์)
File.foreach("test.txt") do |line|
  next if line.start_with?("#")  # ข้าม comments
  puts line.chomp
end

# วิธีที่ 4: lazy reading
result = File.foreach("test.txt")
             .lazy
             .select { |line| line.include?("บรรทัด") }
             .map(&:chomp)
             .first(2)
puts result.inspect
```

### อ่านไฟล์ขนาดใหญ่

```ruby
# สร้างไฟล์ทดสอบขนาดใหญ่
File.open("large_file.txt", "w") do |f|
  1000.times do |i|
    f.puts "Line #{i}: #{"x" * 100}"
  end
end

# นับบรรทัดโดยไม่โหลดทั้งหมดเข้าหน่วยความจำ
line_count = 0
File.foreach("large_file.txt") { line_count += 1 }
puts "จำนวนบรรทัด: #{line_count}"

# Process ทีละ chunk
def process_large_file(filename, chunk_size = 1024)
  File.open(filename) do |f|
    while (chunk = f.read(chunk_size))
      yield chunk
    end
  end
end

total_chars = 0
process_large_file("large_file.txt") { |chunk| total_chars += chunk.length }
puts "ขนาดไฟล์: #{total_chars} ตัวอักษร"
```

---

## Step 303: Writing to Files

```ruby
# เขียนข้อมูลแบบต่างๆ

# 1. เขียน String
File.write("string_file.txt", "Hello World!")

# 2. เขียน Array
lines = ["บรรทัด 1", "บรรทัด 2", "บรรทัด 3"]
File.write("array_file.txt", lines.join("\n"))

# 3. เขียนด้วย puts (เพิ่ม newline อัตโนมัติ)
File.open("puts_file.txt", "w") do |f|
  f.puts "Hello"
  f.puts "World"
  f.puts ["line1", "line2", "line3"]  # Array
end

# 4. เขียนด้วย print (ไม่เพิ่ม newline)
File.open("print_file.txt", "w") do |f|
  f.print "Hello"
  f.print " "
  f.print "World"
  f.print "\n"
end

# 5. เขียน formatted
File.open("formatted.txt", "w") do |f|
  f.printf("%-20s %10s\n", "ชื่อ", "ราคา")
  f.printf("%-20s %10.2f\n", "MacBook Pro", 59900)
  f.printf("%-20s %10.2f\n", "iPhone 15", 32900)
  f.printf("%-20s %10.2f\n", "AirPods Pro", 8900)
end

puts File.read("formatted.txt")
```

---

## Step 304: File Modes

```ruby
# ทดสอบ modes ต่างๆ

# w - สร้างใหม่หรือล้างเนื้อหาเดิม
File.open("modes_test.txt", "w") { |f| f.puts "Initial content" }

# a - ต่อท้าย
File.open("modes_test.txt", "a") { |f| f.puts "Appended content" }
puts "After append:\n#{File.read('modes_test.txt')}"

# r+ - อ่านและเขียน (ต้องมีไฟล์อยู่แล้ว)
File.open("modes_test.txt", "r+") do |f|
  content = f.read
  f.seek(0)        # กลับไปต้นไฟล์
  f.write("Modified: " + content)
  f.truncate(f.pos)  # ตัดส่วนที่เหลือ
end
puts "After r+ modify:\n#{File.read('modes_test.txt')}"

# ตรวจสอบ mode
File.open("modes_test.txt", "r") do |f|
  puts "Mode: #{f.path}"
  puts "Closed? #{f.closed?}"
end
```

---

## Step 305: CSV Reading/Writing

```ruby
require 'csv'

# === เขียน CSV ===

# วิธีที่ 1: สร้าง CSV string แล้วเขียนไฟล์
csv_data = CSV.generate do |csv|
  csv << ["ชื่อ", "อายุ", "เมือง", "เงินเดือน"]  # header
  csv << ["สมชาย", 25, "กรุงเทพฯ", 50000]
  csv << ["สมหญิง", 30, "เชียงใหม่", 65000]
  csv << ["สมศักดิ์", 28, "ภูเก็ต", 55000]
  csv << ["สมพร", 35, "ขอนแก่น", 45000]
end

File.write("employees.csv", csv_data)
puts "สร้าง CSV สำเร็จ"

# วิธีที่ 2: เขียนตรงลงไฟล์
CSV.open("products.csv", "w") do |csv|
  csv << ["รหัส", "ชื่อสินค้า", "ราคา", "สต็อก"]
  [
    [1, "MacBook Pro", 59900, 5],
    [2, "iPhone 15", 32900, 10],
    [3, "iPad", 25900, 8],
    [4, "AirPods", 7900, 20]
  ].each { |row| csv << row }
end

# === อ่าน CSV ===

puts "\n=== อ่านไฟล์ employees.csv ==="

# วิธีที่ 1: อ่านทั้งหมด
rows = CSV.read("employees.csv")
rows.each { |row| puts row.inspect }

# วิธีที่ 2: อ่านพร้อม headers
puts "\n=== อ่านพร้อม headers ==="
CSV.foreach("employees.csv", headers: true) do |row|
  puts "#{row['ชื่อ']} - อายุ #{row['อายุ']} - เงินเดือน #{row['เงินเดือน']}"
end

# วิธีที่ 3: แปลงเป็น Array of Hashes
employees = CSV.read("employees.csv", headers: true).map(&:to_h)
puts "\n=== แปลงเป็น Hash ==="
employees.each { |emp| puts emp.inspect }

# วิธีที่ 4: วิเคราะห์ข้อมูล
salaries = employees.map { |e| e["เงินเดือน"].to_i }
puts "\nเงินเดือนเฉลี่ย: #{salaries.sum.to_f / salaries.length}"
puts "เงินเดือนสูงสุด: #{salaries.max}"
puts "เงินเดือนต่ำสุด: #{salaries.min}"
```

### CSV ขั้นสูง

```ruby
require 'csv'

# CSV กับ options ต่างๆ
CSV.open("custom.csv", "w", 
  col_sep: ";",          # ใช้ ; แทน ,
  row_sep: "\r\n",       # Windows line ending
  quote_char: '"'        # quote character
) do |csv|
  csv << ["Product", "Price", "Description"]
  csv << ["MacBook", 59900, "Powerful laptop, great for coding"]
  csv << ["iPhone", 32900, "Smartphone; latest model"]
end

# อ่าน CSV ที่มี custom separator
CSV.foreach("custom.csv",
  col_sep: ";",
  headers: true,
  converters: [:numeric]  # แปลงตัวเลขอัตโนมัติ
) do |row|
  puts "#{row['Product']}: #{row['Price'].class} = #{row['Price']}"
end

# CSV Transformation
class CSVTransformer
  def initialize(source_file)
    @source = source_file
    @transformations = []
  end
  
  def filter(&block)
    @transformations << [:filter, block]
    self
  end
  
  def map_field(field, &block)
    @transformations << [:map_field, field, block]
    self
  end
  
  def add_field(name, &block)
    @transformations << [:add_field, name, block]
    self
  end
  
  def write_to(dest_file)
    rows = CSV.read(@source, headers: true)
    
    @transformations.each do |transform|
      case transform[0]
      when :filter
        rows = rows.select { |row| transform[1].call(row) }
      when :map_field
        rows.each { |row| row[transform[1]] = transform[2].call(row[transform[1]]) }
      when :add_field
        name = transform[1]
        rows.each { |row| row[name] = transform[2].call(row) }
      end
    end
    
    CSV.open(dest_file, "w") do |csv|
      csv << rows.first.headers if rows.first
      rows.each { |row| csv << row.fields }
    end
    
    puts "เขียน #{rows.length} แถวไปยัง #{dest_file}"
    self
  end
end

CSVTransformer.new("employees.csv")
  .filter { |row| row["เงินเดือน"].to_i >= 50000 }
  .map_field("เงินเดือน") { |v| (v.to_i * 1.1).to_i.to_s }
  .add_field("ระดับ") { |row| row["เงินเดือน"].to_i > 60000 ? "อาวุโส" : "ทั่วไป" }
  .write_to("employees_filtered.csv")
```

---

## Step 306: JSON Reading/Writing

```ruby
require 'json'

# === เขียน JSON ===

data = {
  users: [
    { id: 1, name: "สมชาย", email: "somchai@example.com", age: 25,
      address: { city: "กรุงเทพฯ", district: "ลาดพร้าว" } },
    { id: 2, name: "สมหญิง", email: "somying@example.com", age: 30,
      address: { city: "เชียงใหม่", district: "เมือง" } }
  ],
  metadata: {
    total: 2,
    created_at: Time.now.iso8601
  }
}

# วิธีที่ 1: JSON.generate (compact)
compact_json = JSON.generate(data)
File.write("compact.json", compact_json)

# วิธีที่ 2: JSON.pretty_generate (formatted)
pretty_json = JSON.pretty_generate(data)
File.write("pretty.json", pretty_json)

puts "Pretty JSON:"
puts pretty_json

# === อ่าน JSON ===

puts "\n=== อ่านไฟล์ JSON ==="

# วิธีที่ 1: JSON.parse
json_string = File.read("pretty.json")
parsed = JSON.parse(json_string)

puts "Users:"
parsed["users"].each do |user|
  puts "  #{user['name']} (#{user['email']}) - #{user['address']['city']}"
end

# วิธีที่ 2: symbolize_names
parsed_sym = JSON.parse(json_string, symbolize_names: true)
puts "\nSymbolized:"
parsed_sym[:users].each do |user|
  puts "  #{user[:name]} - #{user[:address][:city]}"
end
```

### JSON กับ Custom Objects

```ruby
require 'json'

class User
  attr_accessor :name, :email, :age, :created_at
  
  def initialize(name, email, age)
    @name = name
    @email = email
    @age = age
    @created_at = Time.now.iso8601
  end
  
  # Custom serialization
  def to_json(*options)
    {
      name: @name,
      email: @email,
      age: @age,
      created_at: @created_at
    }.to_json(*options)
  end
  
  def self.from_json(json_string)
    data = JSON.parse(json_string, symbolize_names: true)
    user = new(data[:name], data[:email], data[:age])
    user.created_at = data[:created_at]
    user
  end
  
  def self.load_from_file(filename)
    json = File.read(filename)
    data = JSON.parse(json, symbolize_names: true)
    data.map { |u| new(u[:name], u[:email], u[:age]) }
  end
  
  def to_s
    "User(#{@name}, #{@email}, #{@age})"
  end
end

users = [
  User.new("สมชาย", "somchai@example.com", 25),
  User.new("สมหญิง", "somying@example.com", 30)
]

# บันทึก
File.write("users.json", JSON.pretty_generate(users))

# โหลด
json_content = File.read("users.json")
loaded_users = JSON.parse(json_content, symbolize_names: true).map do |u|
  User.new(u[:name], u[:email], u[:age])
end

loaded_users.each { |u| puts u }
```

---

## Step 307: YAML Reading/Writing

```ruby
require 'yaml'

# === เขียน YAML ===

config = {
  "app" => {
    "name" => "MyRubyApp",
    "version" => "1.0.0",
    "debug" => false
  },
  "database" => {
    "host" => "localhost",
    "port" => 5432,
    "name" => "myapp_production",
    "pool" => 5
  },
  "redis" => {
    "host" => "localhost",
    "port" => 6379,
    "db" => 0
  },
  "email" => {
    "smtp_host" => "smtp.gmail.com",
    "smtp_port" => 587,
    "from" => "app@example.com"
  }
}

File.write("config.yml", config.to_yaml)

puts "YAML:"
puts config.to_yaml

# === อ่าน YAML ===

loaded_config = YAML.load_file("config.yml")
puts "\nDatabase host: #{loaded_config['database']['host']}"
puts "App name: #{loaded_config['app']['name']}"
puts "Debug: #{loaded_config['app']['debug']}"
```

### YAML กับ Ruby Objects

```ruby
require 'yaml'

class ServerConfig
  attr_accessor :host, :port, :timeout, :max_connections
  
  def initialize(host:, port:, timeout: 30, max_connections: 100)
    @host = host
    @port = port
    @timeout = timeout
    @max_connections = max_connections
  end
  
  def to_yaml_data
    {
      "host" => @host,
      "port" => @port,
      "timeout" => @timeout,
      "max_connections" => @max_connections
    }
  end
  
  def self.from_yaml_file(filename)
    data = YAML.load_file(filename)
    new(
      host: data["host"],
      port: data["port"],
      timeout: data.fetch("timeout", 30),
      max_connections: data.fetch("max_connections", 100)
    )
  end
  
  def save(filename)
    File.write(filename, to_yaml_data.to_yaml)
    puts "บันทึก config ไปยัง #{filename}"
  end
  
  def to_s
    "Server(#{@host}:#{@port}, timeout: #{@timeout}s)"
  end
end

server = ServerConfig.new(
  host: "production.example.com",
  port: 8080,
  timeout: 60,
  max_connections: 500
)

server.save("server_config.yml")
puts File.read("server_config.yml")

loaded = ServerConfig.from_yaml_file("server_config.yml")
puts loaded
```

### YAML Environment Config Pattern

```ruby
require 'yaml'
require 'erb'

# config.yml ด้วย ERB template
yaml_content = <<~YAML
  common: &common
    app_name: MyApp
    secret_key: secret123
    
  development:
    <<: *common
    database:
      host: localhost
      name: myapp_development
    debug: true
    log_level: debug
    
  production:
    <<: *common
    database:
      host: prod-db.example.com
      name: myapp_production
    debug: false
    log_level: warn
YAML

File.write("app_config.yml", yaml_content)

class AppConfig
  def initialize(env = "development")
    raw = YAML.load(File.read("app_config.yml"))
    @config = raw[env] || raw["development"]
    @env = env
  end
  
  def [](key)
    @config[key.to_s]
  end
  
  def get(*keys)
    keys.reduce(@config) { |config, key| config[key.to_s] }
  end
  
  def development?
    @env == "development"
  end
  
  def production?
    @env == "production"
  end
  
  def to_s
    "AppConfig(#{@env})"
  end
end

dev_config = AppConfig.new("development")
prod_config = AppConfig.new("production")

puts dev_config.get("database", "host")   # => localhost
puts prod_config.get("database", "host")  # => prod-db.example.com
puts dev_config["debug"]                  # => true
puts prod_config["debug"]                 # => false
```

---

## Step 308: Directory Operations

```ruby
# สร้าง directory
Dir.mkdir("test_dir") unless Dir.exist?("test_dir")
Dir.mkdir("test_dir/subdir") unless Dir.exist?("test_dir/subdir")

# สร้างหลาย directories
require 'fileutils'
FileUtils.mkdir_p("deep/nested/directory/path")

# List files in directory
puts "\nFiles in current directory:"
Dir.entries(".").sort.first(5).each { |f| puts "  #{f}" }

# Glob patterns
puts "\n.txt files:"
Dir.glob("*.txt").each { |f| puts "  #{f}" }

puts "\nAll Ruby files (recursive):"
Dir.glob("**/*.rb").first(5).each { |f| puts "  #{f}" }

# เปลี่ยน directory
current = Dir.pwd
puts "\nCurrent directory: #{current}"

Dir.chdir("test_dir") do
  puts "Inside test_dir: #{Dir.pwd}"
  File.write("inside.txt", "created inside test_dir")
end

puts "Back to: #{Dir.pwd}"

# ลบ directory
FileUtils.rm_rf("test_dir")  # ลบพร้อม contents
puts "ลบ test_dir แล้ว"
FileUtils.rm_rf("deep")
```

### Directory Traversal

```ruby
require 'find'
require 'fileutils'

# สร้าง directory structure สำหรับทดสอบ
structure = {
  "project/lib" => ["helper.rb", "utils.rb"],
  "project/lib/models" => ["user.rb", "product.rb"],
  "project/spec" => ["helper_spec.rb", "utils_spec.rb"],
  "project/config" => ["config.yml", "database.yml"],
  "project" => ["Gemfile", "README.md", "app.rb"]
}

structure.each do |dir, files|
  FileUtils.mkdir_p(dir)
  files.each do |file|
    File.write("#{dir}/#{file}", "# #{file}\n")
  end
end

puts "=== Find ทุกไฟล์ ==="
Find.find("project") do |path|
  if File.directory?(path)
    puts "DIR:  #{path}"
  else
    puts "FILE: #{path} (#{File.size(path)} bytes)"
  end
end

puts "\n=== เฉพาะไฟล์ .rb ==="
Find.find("project") do |path|
  puts path if File.file?(path) && path.end_with?(".rb")
end

puts "\n=== ขนาดรวม ==="
total_size = Find.find("project").sum do |path|
  File.file?(path) ? File.size(path) : 0
end
puts "รวม: #{total_size} bytes"

FileUtils.rm_rf("project")
```

---

## Step 309: File.exist?, File.directory?

```ruby
require 'fileutils'

# สร้างไฟล์และ directory สำหรับทดสอบ
FileUtils.mkdir_p("test_workspace")
File.write("test_workspace/sample.txt", "test content")
File.write("test_workspace/data.json", '{"key": "value"}')

# === File checking methods ===

path = "test_workspace/sample.txt"
dir_path = "test_workspace"

# ตรวจสอบการมีอยู่
puts "=== File Existence ==="
puts "File.exist?('#{path}'): #{File.exist?(path)}"
puts "File.exists?('#{path}'): #{File.exist?(path)}"  # alias
puts "File.file?('#{path}'): #{File.file?(path)}"
puts "File.directory?('#{dir_path}'): #{File.directory?(dir_path)}"
puts "File.directory?('#{path}'): #{File.directory?(path)}"

# ตรวจสอบ permissions
puts "\n=== Permissions ==="
puts "File.readable?('#{path}'): #{File.readable?(path)}"
puts "File.writable?('#{path}'): #{File.writable?(path)}"
puts "File.executable?('#{path}'): #{File.executable?(path)}"

# ตรวจสอบขนาด
puts "\n=== File Info ==="
puts "File.size('#{path}'): #{File.size(path)} bytes"
puts "File.zero?('#{path}'): #{File.zero?(path)}"
puts "File.empty?('#{path}'): #{File.empty?(path)}"

# ตรวจสอบเวลา
puts "\n=== Timestamps ==="
puts "File.mtime: #{File.mtime(path)}"
puts "File.ctime: #{File.ctime(path)}"
puts "File.atime: #{File.atime(path)}"

# File information
stat = File.stat(path)
puts "\n=== Stat ==="
puts "Size: #{stat.size}"
puts "Mode: #{stat.mode.to_s(8)}"
puts "Modified: #{stat.mtime}"

# เปรียบเทียบเวลา
f1 = "test_workspace/sample.txt"
f2 = "test_workspace/data.json"
puts "\n#{f1} newer than #{f2}? #{File.mtime(f1) > File.mtime(f2)}"

FileUtils.rm_rf("test_workspace")
```

---

## Step 310: Pathname

```ruby
require 'pathname'

# Pathname เป็น OOP wrapper สำหรับ file paths

home = Pathname.new("/home/user")
project = home + "projects" + "my_app"

puts project        # => /home/user/projects/my_app
puts project.to_s   # => /home/user/projects/my_app

# Pathname operations
path = Pathname.new("/home/user/documents/report.pdf")
puts path.dirname    # => /home/user/documents
puts path.basename   # => report.pdf
puts path.extname    # => .pdf
puts path.basename(".pdf")  # => report
puts path.parent     # => /home/user/documents
puts path.root?      # => false
puts path.absolute?  # => true
puts path.relative?  # => false

# Relative path
rel = Pathname.new("./lib/models/user.rb")
puts rel.relative?   # => true
puts rel.dirname     # => lib/models
puts rel.basename    # => user.rb

# Join paths
base = Pathname.new("/var/www")
log_dir = base / "app" / "log"
log_file = log_dir / "production.log"
puts log_file  # => /var/www/app/log/production.log

# Cleanup paths
messy = Pathname.new("/usr//local/../local/bin")
puts messy.cleanpath  # => /usr/local/bin

# Glob
puts "\n=== Pathname Glob ==="
current = Pathname.new(".")
current.glob("*.txt").each { |f| puts f }
```

### Pathname ใน Real Code

```ruby
require 'pathname'

class FileManager
  def initialize(base_dir)
    @base = Pathname.new(base_dir).expand_path
    @base.mkpath unless @base.exist?
  end
  
  def path_for(filename)
    (@base + filename).cleanpath
  end
  
  def write(filename, content)
    full_path = path_for(filename)
    full_path.parent.mkpath  # สร้าง subdirectories ถ้าจำเป็น
    full_path.write(content)
    puts "เขียนไฟล์: #{full_path}"
    full_path
  end
  
  def read(filename)
    full_path = path_for(filename)
    raise Errno::ENOENT, "ไม่พบไฟล์: #{filename}" unless full_path.exist?
    full_path.read
  end
  
  def list(pattern = "**/*")
    @base.glob(pattern).select(&:file?).map do |p|
      p.relative_path_from(@base).to_s
    end
  end
  
  def delete(filename)
    path_for(filename).delete
    puts "ลบไฟล์: #{filename}"
  end
  
  def move(from, to)
    full_from = path_for(from)
    full_to = path_for(to)
    full_to.parent.mkpath
    full_from.rename(full_to)
    puts "ย้ายไฟล์: #{from} -> #{to}"
  end
  
  def size
    @base.glob("**/*").select(&:file?).sum { |f| f.size }
  end
  
  def to_s
    "FileManager(#{@base})"
  end
end

fm = FileManager.new("./file_manager_test")
fm.write("documents/report.txt", "รายงานสำคัญ")
fm.write("images/photo.jpg", "fake image data")
fm.write("data.json", '{"key": "value"}')

puts "\nไฟล์ทั้งหมด:"
fm.list.each { |f| puts "  #{f}" }

puts "\nอ่านไฟล์:"
puts fm.read("documents/report.txt")

puts "\nขนาดรวม: #{fm.size} bytes"

require 'fileutils'
FileUtils.rm_rf("./file_manager_test")
```

---

## Step 311: Tempfile

```ruby
require 'tempfile'

# Tempfile - ไฟล์ชั่วคราวที่ถูกลบอัตโนมัติ

# วิธีที่ 1: ใช้ block (แนะนำ)
Tempfile.create("my_temp") do |f|
  f.write("ข้อมูลชั่วคราว")
  f.flush
  f.rewind
  
  puts "Temp file path: #{f.path}"
  puts "Content: #{f.read}"
end

# ไฟล์ถูกลบอัตโนมัติหลัง block

# วิธีที่ 2: manual
temp = Tempfile.new("temp_data")
begin
  temp.write("ข้อมูล 1\n")
  temp.write("ข้อมูล 2\n")
  temp.rewind
  
  puts "\nTemp file: #{temp.path}"
  puts temp.read
ensure
  temp.close
  temp.unlink  # ลบไฟล์
end

# Tempfile ด้วย extension
Tempfile.create(["data", ".json"]) do |f|
  f.write('{"temp": true}')
  puts "\nJSON temp: #{f.path}"
end

# ใช้งานจริง: ประมวลผลไฟล์ใหญ่
def process_large_dataset(data_array)
  Tempfile.create("dataset") do |temp|
    # เขียนข้อมูลลง temp file ก่อน
    data_array.each { |item| temp.puts item.to_s }
    temp.rewind
    
    # ประมวลผลทีละบรรทัด
    results = []
    temp.each_line do |line|
      results << line.strip.upcase
    end
    
    results
  end
end

data = (1..100).map { |i| "item_#{i}" }
processed = process_large_dataset(data)
puts "\nProcessed first 5: #{processed.first(5).inspect}"
```

---

## Step 312: StringIO

```ruby
require 'stringio'

# StringIO - ใช้ String เป็นเหมือน file

# เขียนลง StringIO
sio = StringIO.new
sio.puts "Hello"
sio.puts "World"
sio.write "Ruby!"
sio.rewind

puts "StringIO content:"
puts sio.read

# อ่านทีละบรรทัด
sio.rewind
sio.each_line.with_index(1) do |line, i|
  puts "Line #{i}: #{line.chomp}"
end

# ใช้เป็น capture buffer
def capture_output
  original = $stdout
  $stdout = StringIO.new
  yield
  output = $stdout.string
  $stdout = original
  output
end

captured = capture_output do
  puts "This won't appear in terminal"
  puts "It's captured!"
  p [1, 2, 3]
end

puts "\nCaptured output:"
puts captured

# StringIO ใช้สำหรับ test
class Logger
  def initialize(output = $stdout)
    @output = output
  end
  
  def log(message, level = :info)
    @output.puts "[#{level.upcase}] #{message}"
  end
  
  def error(message)
    log(message, :error)
  end
  
  def warn(message)
    log(message, :warn)
  end
end

# ใน tests
buffer = StringIO.new
logger = Logger.new(buffer)
logger.log("Application started")
logger.error("Something went wrong")
logger.warn("Low disk space")

puts "\nLogger output:"
puts buffer.string
puts "Contains error? #{buffer.string.include?('[ERROR]')}"
```

---

## Step 313: FileUtils

```ruby
require 'fileutils'

# === FileUtils operations ===

# สร้าง directory
FileUtils.mkdir_p("test_fu/subdir1/deep")
FileUtils.mkdir_p("test_fu/subdir2")

# สร้างไฟล์
File.write("test_fu/file1.txt", "content 1")
File.write("test_fu/file2.txt", "content 2")
File.write("test_fu/subdir1/nested.txt", "nested content")

# Copy ไฟล์
FileUtils.cp("test_fu/file1.txt", "test_fu/file1_copy.txt")
puts "Copied file1.txt"

# Copy directory
FileUtils.cp_r("test_fu/subdir1", "test_fu/subdir1_copy")
puts "Copied directory"

# Move ไฟล์
FileUtils.mv("test_fu/file2.txt", "test_fu/subdir2/file2_moved.txt")
puts "Moved file2.txt"

# ลบไฟล์
FileUtils.rm("test_fu/file1_copy.txt")
puts "Deleted copy"

# ลบ directory
FileUtils.rm_rf("test_fu/subdir1_copy")
puts "Deleted directory copy"

# เปลี่ยน permissions
FileUtils.chmod(0644, "test_fu/file1.txt")
FileUtils.chmod_R(0755, "test_fu/subdir1")

# Touch (สร้างไฟล์ว่างหรืออัปเดต timestamp)
FileUtils.touch("test_fu/empty_file.txt")
FileUtils.touch(["test_fu/file_a.txt", "test_fu/file_b.txt"])

puts "\nFiles in test_fu:"
Dir.glob("test_fu/**/*").sort.each { |f| puts "  #{f}" }

FileUtils.rm_rf("test_fu")
```

---

## Step 314: Practical File Processing Examples

### Log File Analyzer

```ruby
require 'time'

class LogAnalyzer
  def initialize(log_file)
    @log_file = log_file
    @entries = []
  end
  
  def parse
    File.foreach(@log_file) do |line|
      if match = line.match(/\[(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\] \[(\w+)\] (.+)/)
        @entries << {
          time: Time.parse(match[1]),
          level: match[2],
          message: match[3].chomp
        }
      end
    end
    self
  end
  
  def by_level(level)
    @entries.select { |e| e[:level] == level.upcase }
  end
  
  def errors
    by_level("ERROR")
  end
  
  def warnings
    by_level("WARN")
  end
  
  def time_range(from, to)
    @entries.select { |e| e[:time].between?(from, to) }
  end
  
  def summary
    counts = @entries.group_by { |e| e[:level] }.transform_values(&:count)
    {
      total: @entries.length,
      by_level: counts,
      first: @entries.first&.dig(:time),
      last: @entries.last&.dig(:time)
    }
  end
  
  def to_s
    "LogAnalyzer(#{@log_file}, #{@entries.length} entries)"
  end
end

# สร้าง sample log file
log_content = <<~LOG
  [2024-01-15 09:00:01] [INFO] Application started
  [2024-01-15 09:00:05] [INFO] Database connected
  [2024-01-15 09:01:00] [WARN] Slow query detected (2.3s)
  [2024-01-15 09:02:15] [ERROR] Failed to send email: Connection refused
  [2024-01-15 09:05:00] [INFO] 100 requests processed
  [2024-01-15 09:10:00] [ERROR] Disk space low (89% used)
  [2024-01-15 09:15:00] [WARN] Memory usage high (78%)
  [2024-01-15 09:20:00] [INFO] Scheduled job completed
LOG

File.write("app.log", log_content)

analyzer = LogAnalyzer.new("app.log")
analyzer.parse

puts "Summary: #{analyzer.summary.inspect}"
puts "\nErrors:"
analyzer.errors.each { |e| puts "  #{e[:time]}: #{e[:message]}" }
puts "\nWarnings:"
analyzer.warnings.each { |e| puts "  #{e[:time]}: #{e[:message]}" }

File.delete("app.log")
```

### Configuration File Manager

```ruby
require 'yaml'
require 'json'

class ConfigManager
  FORMATS = {
    ".yml" => :yaml,
    ".yaml" => :yaml,
    ".json" => :json
  }
  
  def initialize(config_file)
    @config_file = config_file
    @ext = File.extname(config_file).downcase
    @config = {}
    load if File.exist?(config_file)
  end
  
  def load
    format = FORMATS[@ext] || :yaml
    content = File.read(@config_file)
    @config = case format
              when :yaml then YAML.load(content) || {}
              when :json then JSON.parse(content)
              end
    self
  end
  
  def save
    format = FORMATS[@ext] || :yaml
    content = case format
              when :yaml then @config.to_yaml
              when :json then JSON.pretty_generate(@config)
              end
    File.write(@config_file, content)
    puts "บันทึก config: #{@config_file}"
    self
  end
  
  def get(key_path, default = nil)
    keys = key_path.to_s.split(".")
    keys.reduce(@config) do |config, key|
      config.is_a?(Hash) ? config[key] || config[key.to_sym] : nil
    end || default
  end
  
  def set(key_path, value)
    keys = key_path.to_s.split(".")
    last_key = keys.pop
    
    target = keys.reduce(@config) do |config, key|
      config[key] ||= {}
    end
    
    target[last_key] = value
    self
  end
  
  def delete(key_path)
    keys = key_path.to_s.split(".")
    last_key = keys.pop
    
    target = keys.reduce(@config) do |config, key|
      config[key] || {}
    end
    
    target.delete(last_key)
    self
  end
  
  def [](key)
    @config[key.to_s] || @config[key.to_sym]
  end
  
  def []=(key, value)
    @config[key.to_s] = value
  end
  
  def to_s
    "ConfigManager(#{@config_file})"
  end
end

# ทดสอบ
config = ConfigManager.new("app_settings.yml")
config.set("database.host", "localhost")
config.set("database.port", 5432)
config.set("app.name", "MyApp")
config.set("app.debug", true)
config.save

# โหลดใหม่
loaded = ConfigManager.new("app_settings.yml")
puts "DB Host: #{loaded.get('database.host')}"
puts "App Name: #{loaded.get('app.name')}"
puts "Debug: #{loaded.get('app.debug')}"
puts "Missing: #{loaded.get('missing.key', 'default_value')}"

File.delete("app_settings.yml")
```

---

## แบบฝึกหัด Part 15 (20 ข้อ)

### ระดับง่าย (ข้อ 1-7)

**ข้อ 1:** เขียน word counter สำหรับไฟล์ข้อความ

```ruby
# เฉลย
def count_words(filename)
  return {} unless File.exist?(filename)
  
  words = File.read(filename).downcase.scan(/\w+/)
  counts = Hash.new(0)
  words.each { |word| counts[word] += 1 }
  counts.sort_by { |_, v| -v }.to_h
end

# สร้างไฟล์ทดสอบ
File.write("word_test.txt", "ruby is great ruby is fun programming ruby")
counts = count_words("word_test.txt")
counts.first(5).each { |word, count| puts "#{word}: #{count}" }
File.delete("word_test.txt")
```

**ข้อ 2:** สร้าง CSV report generator

```ruby
# เฉลย
require 'csv'

class SalesReport
  def initialize
    @sales = []
  end
  
  def add_sale(product, amount, date = Date.today)
    @sales << { product: product, amount: amount, date: date }
    self
  end
  
  def write_csv(filename)
    CSV.open(filename, "w") do |csv|
      csv << ["วันที่", "สินค้า", "ยอดขาย"]
      @sales.each { |s| csv << [s[:date], s[:product], s[:amount]] }
      csv << []
      csv << ["รวม", "", @sales.sum { |s| s[:amount] }]
    end
    puts "สร้าง report: #{filename}"
  end
  
  def total
    @sales.sum { |s| s[:amount] }
  end
end

report = SalesReport.new
report.add_sale("MacBook Pro", 59900)
      .add_sale("iPhone 15", 32900)
      .add_sale("iPad", 25900)
report.write_csv("sales_report.csv")
puts "รวมยอดขาย: #{report.total}"
puts CSV.read("sales_report.csv").inspect
File.delete("sales_report.csv")
```

**ข้อ 3-7:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 3: สร้าง File Backup utility ที่ backup พร้อม timestamp
# ข้อ 4: สร้าง Directory Size Calculator
# ข้อ 5: สร้าง File Search ที่ค้นหาตาม content
# ข้อ 6: สร้าง CSV to JSON converter
# ข้อ 7: สร้าง Config file merger (รวม YAML หลายไฟล์)
```

### ระดับกลาง (ข้อ 8-14)

**ข้อ 8:** สร้าง Log Rotation System

```ruby
# เฉลย
require 'fileutils'

class LogRotator
  def initialize(log_file, max_size: 1024, max_files: 5)
    @log_file = log_file
    @max_size = max_size  # bytes
    @max_files = max_files
  end
  
  def write(message)
    rotate_if_needed
    
    File.open(@log_file, "a") do |f|
      f.puts "[#{Time.now.strftime('%Y-%m-%d %H:%M:%S')}] #{message}"
    end
  end
  
  def rotate_if_needed
    return unless File.exist?(@log_file)
    return unless File.size(@log_file) >= @max_size
    
    # Rotate files
    (@max_files - 1).downto(1) do |i|
      old = "#{@log_file}.#{i}"
      newer = i == 1 ? @log_file : "#{@log_file}.#{i - 1}"
      FileUtils.mv(newer, old) if File.exist?(newer)
    end
    
    FileUtils.mv(@log_file, "#{@log_file}.1")
    puts "Log rotated"
  end
  
  def files
    [@log_file] + (1...@max_files).map { |i| "#{@log_file}.#{i}" }
      .select { |f| File.exist?(f) }
  end
end

rotator = LogRotator.new("test_rotate.log", max_size: 200)

50.times { |i| rotator.write("Log entry #{i + 1}: #{"data" * 5}") }

puts "\nLog files:"
rotator.files.each do |f|
  puts "  #{f}: #{File.size(f)} bytes"
end

rotator.files.each { |f| File.delete(f) if File.exist?(f) }
```

**ข้อ 9-14:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 9: สร้าง File Watcher ที่ monitor การเปลี่ยนแปลง
# ข้อ 10: สร้าง Database-like File Storage ด้วย JSON
# ข้อ 11: สร้าง Template Engine ที่ใช้ไฟล์ .erb
# ข้อ 12: สร้าง File Encryption/Decryption utility
# ข้อ 13: สร้าง Archive/Extract utility
# ข้อ 14: สร้าง Bulk File Renamer
```

### ระดับยาก (ข้อ 15-20)

**ข้อ 15:** สร้าง Simple File-based Key-Value Store

```ruby
# เฉลย
require 'json'
require 'fileutils'

class FileStore
  def initialize(directory)
    @dir = directory
    FileUtils.mkdir_p(@dir)
    @index = load_index
  end
  
  def set(key, value, ttl: nil)
    key = normalize_key(key)
    expiry = ttl ? Time.now.to_i + ttl : nil
    
    metadata = { key: key, created_at: Time.now.to_i, expires_at: expiry }
    data = { metadata: metadata, value: value }
    
    File.write(path_for(key), JSON.generate(data))
    @index[key] = metadata
    save_index
    
    value
  end
  
  def get(key)
    key = normalize_key(key)
    return nil unless @index.key?(key)
    
    file_path = path_for(key)
    return nil unless File.exist?(file_path)
    
    data = JSON.parse(File.read(file_path), symbolize_names: true)
    
    if data[:metadata][:expires_at] && Time.now.to_i > data[:metadata][:expires_at]
      delete(key)
      return nil
    end
    
    data[:value]
  end
  
  def delete(key)
    key = normalize_key(key)
    return nil unless @index.key?(key)
    
    file_path = path_for(key)
    File.delete(file_path) if File.exist?(file_path)
    @index.delete(key)
    save_index
    true
  end
  
  def exists?(key)
    get(normalize_key(key)) != nil
  end
  
  def keys
    @index.keys
  end
  
  def count
    @index.length
  end
  
  def clear
    @index.each_key { |k| delete(k) }
    @index = {}
    save_index
  end
  
  def to_s
    "FileStore(#{@dir}, #{count} keys)"
  end
  
  private
  
  def normalize_key(key)
    key.to_s.downcase.gsub(/[^a-z0-9_]/, '_')
  end
  
  def path_for(key)
    File.join(@dir, "#{key}.json")
  end
  
  def load_index
    index_path = File.join(@dir, "_index.json")
    return {} unless File.exist?(index_path)
    JSON.parse(File.read(index_path))
  rescue
    {}
  end
  
  def save_index
    index_path = File.join(@dir, "_index.json")
    File.write(index_path, JSON.generate(@index))
  end
end

# ทดสอบ
store = FileStore.new("./kv_store")
store.set("user:1", { name: "สมชาย", age: 25 })
store.set("config:theme", "dark")
store.set("temp_key", "ค่าชั่วคราว", ttl: 1)

puts store.get("user:1").inspect
puts store.get("config:theme")
puts "Keys: #{store.keys.inspect}"
puts store

sleep(0.01)
puts "temp_key: #{store.get('temp_key').inspect}"  # ยังอยู่

puts "\nหลัง 2 วินาที:"
sleep(2)
puts "temp_key: #{store.get('temp_key').inspect}"  # หมดอายุแล้ว

require 'fileutils'
FileUtils.rm_rf("./kv_store")
```

**ข้อ 16-20:** (โจทย์ให้ทำต่อเอง)

```ruby
# ข้อ 16: สร้าง File Sync utility สำหรับ sync 2 directories
# ข้อ 17: สร้าง CSV Database ที่มี CRUD operations
# ข้อ 18: สร้าง Full-text Search index ด้วย files
# ข้อ 19: สร้าง File-based Message Queue
# ข้อ 20: สร้าง Incremental Backup system
```

---

## สรุป Part 15

ใน Part นี้ เราได้เรียนรู้:

1. **File.open, read, write** - การอ่านและเขียนไฟล์พื้นฐาน
2. **Reading line by line** - อ่านทีละบรรทัดสำหรับไฟล์ใหญ่
3. **Writing to files** - วิธีการเขียนไฟล์แบบต่างๆ
4. **File modes** - r, w, a, r+, w+, a+, b
5. **CSV** - อ่านและเขียนไฟล์ CSV
6. **JSON** - serialize/deserialize กับไฟล์ JSON
7. **YAML** - อ่านและเขียน YAML configuration
8. **Directory operations** - Dir, FileUtils
9. **File checking** - exist?, directory?, file?, size
10. **Pathname** - OOP wrapper สำหรับ paths
11. **Tempfile** - ไฟล์ชั่วคราว
12. **StringIO** - String ที่ทำงานเหมือนไฟล์

**ต่อไป:** Part 16 - Advanced Ruby Topics (Ruby advanced section)
