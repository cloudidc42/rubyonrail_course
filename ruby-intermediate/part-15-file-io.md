# ตอนที่ 15: File I/O (Steps 301-320)

## บทนำ

File I/O คือความสามารถในการอ่านและเขียนไฟล์บนระบบ Ruby มีเครื่องมือครบครันสำหรับการจัดการไฟล์ ตั้งแต่การอ่านเขียนพื้นฐาน ไปจนถึงการจัดการ CSV, JSON, YAML และ directories

---

## Step 301: File.read, File.write (Simplest Way)

```ruby
# เขียนไฟล์ - วิธีง่ายที่สุด
File.write("/tmp/hello.txt", "Hello, World!\n")

# อ่านไฟล์ - วิธีง่ายที่สุด
content = File.read("/tmp/hello.txt")
puts content           # => Hello, World!
puts content.length    # => 14

# write กับ mode
File.write("/tmp/hello.txt", "First line\n")
File.write("/tmp/hello.txt", "Second line\n", mode: "a")  # append
content = File.read("/tmp/hello.txt")
puts content
# => First line
# => Second line

# write binary
File.write("/tmp/data.bin", "\x00\x01\x02\x03\xFF", mode: "wb")
binary = File.read("/tmp/data.bin", mode: "rb")
puts binary.bytes.inspect  # => [0, 1, 2, 3, 255]

# เขียนหลาย lines
lines = ["Ruby is great!", "File I/O is easy.", "Let's learn more!"]
File.write("/tmp/multi.txt", lines.join("\n") + "\n")
puts File.read("/tmp/multi.txt")

# อ่าน n bytes
first_10 = File.read("/tmp/hello.txt", 10)
puts first_10.inspect  # อ่านแค่ 10 bytes

# อ่านจาก offset
from_offset = File.read("/tmp/hello.txt", 5, 6)
puts from_offset.inspect  # อ่าน 5 bytes เริ่มจาก offset 6
```

---

## Step 302: File.open with Block

```ruby
# File.open with block - ปิดอัตโนมัติ
File.open("/tmp/sample.txt", "w") do |file|
  file.puts "Line 1"
  file.puts "Line 2"
  file.puts "Line 3"
  file.write("No newline here")
end
# file ปิดอัตโนมัติเมื่อ block จบ

# อ่านด้วย block
File.open("/tmp/sample.txt", "r") do |file|
  puts "File size: #{file.size} bytes"
  content = file.read
  puts content
end

# File.open ไม่ใช้ block - ต้องปิดเอง
file = File.open("/tmp/sample.txt", "r")
begin
  puts file.readline
  puts file.readline
ensure
  file.close
  puts "File closed: #{file.closed?}"
end

# ใช้ File.open กับ readlines
File.open("/tmp/sample.txt") do |f|
  lines = f.readlines
  puts "Total lines: #{lines.length}"
  lines.each_with_index do |line, i|
    puts "#{i + 1}: #{line.chomp}"
  end
end
```

### File.open เพื่อ append

```ruby
# เปิดและ append
3.times do |i|
  File.open("/tmp/log.txt", "a") do |f|
    f.puts "#{Time.now.strftime('%H:%M:%S')} - Log entry #{i + 1}"
  end
end

puts File.read("/tmp/log.txt")
```

---

## Step 303: File Modes

```ruby
# File modes:
# "r"  - Read only (default) - file ต้องมีอยู่
# "w"  - Write only - สร้างใหม่หรือลบเนื้อหาเดิม
# "a"  - Append - สร้างใหม่หรือเพิ่มที่ท้าย
# "r+" - Read/Write - file ต้องมีอยู่, cursor ที่ต้น
# "w+" - Read/Write - สร้างใหม่หรือลบเนื้อหาเดิม
# "a+" - Read/Append - สร้างใหม่หรือ read จากต้น, write ที่ท้าย
# "b"  - Binary mode (ใช้ร่วมกับ r/w/a)
# "rb", "wb", "ab" - Binary read/write/append

# ทดสอบแต่ละ mode
puts "=== Write mode (w) ==="
File.open("/tmp/mode_test.txt", "w") do |f|
  f.write("Original content")
end
puts File.read("/tmp/mode_test.txt")

puts "\n=== Append mode (a) ==="
File.open("/tmp/mode_test.txt", "a") do |f|
  f.write("\nAppended line")
end
puts File.read("/tmp/mode_test.txt")

puts "\n=== Read/Write mode (r+) ==="
File.open("/tmp/mode_test.txt", "r+") do |f|
  puts "Current pos: #{f.pos}"
  puts "Reading: #{f.read(8)}"
  puts "After read pos: #{f.pos}"
  f.seek(0)  # กลับต้น
  f.write("Modified")
end
puts File.read("/tmp/mode_test.txt")

puts "\n=== Binary mode ==="
# เขียน binary data
File.open("/tmp/binary.bin", "wb") do |f|
  f.write([1, 2, 3, 255, 0].pack("C*"))  # pack as unsigned bytes
end
File.open("/tmp/binary.bin", "rb") do |f|
  bytes = f.read.unpack("C*")
  puts "Binary data: #{bytes.inspect}"
end

# Encoding modes
File.open("/tmp/utf8.txt", "w:UTF-8") do |f|
  f.puts "สวัสดีครับ"
  f.puts "Hello, World!"
end

File.open("/tmp/utf8.txt", "r:UTF-8") do |f|
  f.each_line { |line| puts line.chomp }
end
```

---

## Step 304: Reading Line by Line

```ruby
# สร้างไฟล์ทดสอบ
content = <<~TEXT
  Alice,30,alice@example.com,Developer
  Bob,25,bob@example.com,Designer
  Charlie,35,charlie@example.com,Manager
  Diana,28,diana@example.com,Developer
  Eve,32,eve@example.com,Designer
TEXT
File.write("/tmp/people.csv", content)

puts "=== each_line ==="
File.open("/tmp/people.csv") do |f|
  f.each_line.with_index(1) do |line, num|
    puts "#{num}: #{line.chomp}"
  end
end

puts "\n=== readlines ==="
lines = File.readlines("/tmp/people.csv")
puts "Total: #{lines.length} lines"
puts "First: #{lines.first.chomp}"
puts "Last: #{lines.last.chomp}"

# chomp option
lines_chomped = File.readlines("/tmp/people.csv", chomp: true)
puts "\nWith chomp: #{lines_chomped.inspect}"

puts "\n=== gets ==="
File.open("/tmp/people.csv") do |f|
  while (line = f.gets)
    name, age = line.chomp.split(",")
    puts "#{name} is #{age} years old"
  end
end

puts "\n=== foreach ==="
File.foreach("/tmp/people.csv", chomp: true) do |line|
  parts = line.split(",")
  puts "#{parts[0]}: #{parts[3]}"
end

# อ่านแบบ lazy (memory efficient สำหรับไฟล์ใหญ่)
puts "\n=== Lazy reading ==="
developers = File.foreach("/tmp/people.csv", chomp: true)
  .lazy
  .map { |line| line.split(",") }
  .select { |parts| parts[3] == "Developer" }
  .map { |parts| "#{parts[0]} <#{parts[2]}>" }
  .to_a

puts "Developers: #{developers.inspect}"
```

### อ่านไฟล์ใหญ่ทีละ chunk

```ruby
# อ่านทีละ chunk (memory efficient)
def process_large_file(filename, chunk_size: 4096)
  total_bytes = 0
  chunk_count = 0

  File.open(filename, "rb") do |f|
    while (chunk = f.read(chunk_size))
      total_bytes += chunk.length
      chunk_count += 1
      # process chunk here
    end
  end

  puts "Processed #{chunk_count} chunks, #{total_bytes} bytes total"
end

# ตัวอย่าง: นับจำนวน lines ในไฟล์ใหญ่
def count_lines_efficient(filename)
  count = 0
  File.open(filename) do |f|
    f.each_line { count += 1 }
  end
  count
end

File.write("/tmp/test_large.txt", (1..1000).map { |i| "Line #{i}" }.join("\n"))
puts "Lines: #{count_lines_efficient("/tmp/test_large.txt")}"
```

---

## Step 305: Writing to Files

```ruby
# สร้างไฟล์และเขียน
File.open("/tmp/output.txt", "w") do |f|
  # puts เพิ่ม newline อัตโนมัติ
  f.puts "Hello, World!"
  f.puts "Ruby File I/O"
  f.puts ["Line 3", "Line 4", "Line 5"]  # puts กับ array

  # write ไม่เพิ่ม newline
  f.write("No newline")
  f.write(" here\n")

  # print เหมือน write แต่แปลง object เป็น string
  f.print "Printed: "
  f.print 42
  f.print "\n"

  # printf สำหรับ formatted output
  f.printf("%-20s %8.2f\n", "Product", 1234.56)
  f.printf("%-20s %8d\n", "Count", 42)
end

puts File.read("/tmp/output.txt")

# เขียนทีละบรรทัด
def write_report(filename, data)
  File.open(filename, "w") do |f|
    f.puts "=== Sales Report ==="
    f.puts "Generated: #{Time.now.strftime('%Y-%m-%d %H:%M:%S')}"
    f.puts "=" * 40
    f.printf("%-20s %10s %10s\n", "Product", "Quantity", "Revenue")
    f.puts "-" * 40

    data.each do |item|
      f.printf("%-20s %10d %10.2f\n",
        item[:product], item[:quantity], item[:revenue])
    end

    f.puts "-" * 40
    total = data.sum { |i| i[:revenue] }
    f.printf("%-20s %10s %10.2f\n", "TOTAL", "", total)
  end
end

sales_data = [
  { product: "Laptop", quantity: 10, revenue: 250000 },
  { product: "Mouse", quantity: 50, revenue: 25000 },
  { product: "Keyboard", quantity: 30, revenue: 36000 },
  { product: "Monitor", quantity: 8, revenue: 68000 },
]

write_report("/tmp/sales_report.txt", sales_data)
puts File.read("/tmp/sales_report.txt")
```

---

## Step 306: File Metadata

```ruby
# สร้างไฟล์ทดสอบ
File.write("/tmp/metadata_test.txt", "Hello, Ruby!\nThis is a test file.\n" * 10)

path = "/tmp/metadata_test.txt"

# ขนาดไฟล์
puts "=== File Size ==="
puts "Size: #{File.size(path)} bytes"
puts "Size: #{File.size?(path)} bytes (nil if not exist)"

# เวลา
puts "\n=== File Times ==="
puts "Modified: #{File.mtime(path)}"
puts "Changed:  #{File.ctime(path)}"   # metadata changed time
puts "Accessed: #{File.atime(path)}"

# Path operations
puts "\n=== Path Operations ==="
full_path = "/home/user/projects/myapp/config/database.yml"
puts "dirname:  #{File.dirname(full_path)}"    # => /home/user/projects/myapp/config
puts "basename: #{File.basename(full_path)}"   # => database.yml
puts "basename (no ext): #{File.basename(full_path, '.yml')}"  # => database
puts "extname:  #{File.extname(full_path)}"    # => .yml
puts "expand:   #{File.expand_path('~')}"      # => /home/user (home directory)

# Path joining
puts "\n=== Path Joining ==="
puts File.join("home", "user", "docs", "file.txt")
# => home/user/docs/file.txt

puts File.join("/", "var", "log", "app.log")
# => /var/log/app.log

# File stats
stat = File.stat(path)
puts "\n=== Stat ==="
puts "Size:  #{stat.size}"
puts "Mode:  #{stat.mode.to_s(8)}"  # octal permission
puts "ino:   #{stat.ino}"           # inode number
puts "nlink: #{stat.nlink}"         # number of links
puts "Regular file? #{stat.file?}"
puts "Symlink? #{stat.symlink?}"
```

---

## Step 307: File Existence Checks

```ruby
# สร้างไฟล์และ directory ทดสอบ
File.write("/tmp/test_exist.txt", "test")
Dir.mkdir("/tmp/test_dir_exist") unless Dir.exist?("/tmp/test_dir_exist")

path_file = "/tmp/test_exist.txt"
path_dir = "/tmp/test_dir_exist"
path_none = "/tmp/this_does_not_exist_xyz"

puts "=== Existence Checks ==="
puts "exist?    file: #{File.exist?(path_file)}"     # => true
puts "exist?    dir:  #{File.exist?(path_dir)}"      # => true
puts "exist?    none: #{File.exist?(path_none)}"     # => false

puts "\n=== Type Checks ==="
puts "file?     file: #{File.file?(path_file)}"      # => true
puts "file?     dir:  #{File.file?(path_dir)}"       # => false
puts "directory? file: #{File.directory?(path_file)}" # => false
puts "directory? dir:  #{File.directory?(path_dir)}"  # => true

puts "\n=== Permission Checks ==="
puts "readable? file: #{File.readable?(path_file)}"  # => true
puts "writable? file: #{File.writable?(path_file)}"  # => true
puts "executable? file: #{File.executable?(path_file)}" # => false (text file)

puts "\n=== Content Checks ==="
puts "zero?:    #{File.zero?(path_file)}"            # => false (has content)
File.write("/tmp/empty.txt", "")
puts "zero? empty: #{File.zero?("/tmp/empty.txt")}"  # => true
puts "size?:    #{File.size?(path_file)}"            # => file size or nil

# ใช้ใน method
def safe_read(path)
  unless File.exist?(path)
    puts "File not found: #{path}"
    return nil
  end
  unless File.readable?(path)
    puts "Cannot read: #{path}"
    return nil
  end
  if File.zero?(path)
    puts "File is empty: #{path}"
    return ""
  end

  File.read(path)
end

puts "\n=== safe_read tests ==="
safe_read(path_file)
safe_read(path_none)
puts "Content: #{safe_read(path_file)}"
```

---

## Step 308: Directory Operations

```ruby
require 'fileutils'

puts "=== Current Directory ==="
puts "pwd: #{Dir.pwd}"
puts "home: #{Dir.home}"

# สร้าง directories
puts "\n=== Creating Directories ==="
Dir.mkdir("/tmp/ruby_test") unless Dir.exist?("/tmp/ruby_test")
FileUtils.mkdir_p("/tmp/ruby_test/subdir1/subdir2")  # สร้างทั้ง path
puts "Created: /tmp/ruby_test/subdir1/subdir2"

# สร้างไฟล์ทดสอบ
["file1.txt", "file2.rb", "file3.csv", "subdir1/nested.txt"].each do |f|
  path = "/tmp/ruby_test/#{f}"
  FileUtils.mkdir_p(File.dirname(path))
  File.write(path, "Content of #{f}")
end

puts "\n=== Dir.entries ==="
# entries รวม . และ ..
entries = Dir.entries("/tmp/ruby_test")
puts entries.inspect
# ไม่รวม . และ ..
real_entries = Dir.entries("/tmp/ruby_test").reject { |e| e.start_with?('.') }
puts "Real entries: #{real_entries.inspect}"

puts "\n=== Dir.glob patterns ==="
# glob - wildcard matching
puts "All txt files:"
Dir.glob("/tmp/ruby_test/**/*.txt").each { |f| puts "  #{f}" }

puts "\nAll files (not dirs):"
Dir.glob("/tmp/ruby_test/**/*").select { |f| File.file?(f) }.each { |f| puts "  #{f}" }

puts "\nRuby files:"
Dir.glob("/tmp/ruby_test/*.rb").each { |f| puts "  #{f}" }

# glob patterns
puts "\n=== Glob Patterns ==="
File.write("/tmp/ruby_test/image1.jpg", "")
File.write("/tmp/ruby_test/image2.png", "")
File.write("/tmp/ruby_test/image3.gif", "")

puts "Images (jpg/png):"
Dir.glob("/tmp/ruby_test/*.{jpg,png}").each { |f| puts "  #{File.basename(f)}" }

puts "Files starting with 'file':"
Dir.glob("/tmp/ruby_test/file*").each { |f| puts "  #{File.basename(f)}" }

puts "\n=== Dir.foreach ==="
Dir.foreach("/tmp/ruby_test") do |entry|
  next if entry.start_with?('.')
  type = File.directory?("/tmp/ruby_test/#{entry}") ? "[DIR]" : "[FILE]"
  puts "  #{type} #{entry}"
end

# chdir และ pwd
puts "\n=== Dir.chdir ==="
original_dir = Dir.pwd
Dir.chdir("/tmp") do
  puts "Inside block: #{Dir.pwd}"
end
puts "After block: #{Dir.pwd}"  # กลับมาที่เดิม

puts "\n=== Cleanup ==="
FileUtils.rm_rf("/tmp/ruby_test")
puts "Removed /tmp/ruby_test"
```

### Recursive Directory Traversal

```ruby
def list_directory_tree(path, indent = 0)
  prefix = "  " * indent
  entries = Dir.entries(path).reject { |e| e.start_with?('.') }.sort

  entries.each do |entry|
    full_path = File.join(path, entry)
    if File.directory?(full_path)
      puts "#{prefix}📁 #{entry}/"
      list_directory_tree(full_path, indent + 1)
    else
      size = File.size(full_path)
      puts "#{prefix}📄 #{entry} (#{size} bytes)"
    end
  end
end

# สร้าง structure
FileUtils.mkdir_p("/tmp/demo_tree/src/controllers")
FileUtils.mkdir_p("/tmp/demo_tree/src/models")
FileUtils.mkdir_p("/tmp/demo_tree/tests")

[
  "/tmp/demo_tree/README.md",
  "/tmp/demo_tree/Gemfile",
  "/tmp/demo_tree/src/app.rb",
  "/tmp/demo_tree/src/controllers/users_controller.rb",
  "/tmp/demo_tree/src/controllers/posts_controller.rb",
  "/tmp/demo_tree/src/models/user.rb",
  "/tmp/demo_tree/src/models/post.rb",
  "/tmp/demo_tree/tests/user_test.rb",
  "/tmp/demo_tree/tests/post_test.rb",
].each { |f| File.write(f, "# #{File.basename(f)}\n") }

puts "Project Structure:"
list_directory_tree("/tmp/demo_tree")

FileUtils.rm_rf("/tmp/demo_tree")
```

---

## Step 309: CSV Reading/Writing

```ruby
require 'csv'

# ข้อมูลตัวอย่าง
students = [
  { name: "Alice Smith", age: 22, grade: "A", gpa: 3.9, major: "Computer Science" },
  { name: "Bob Johnson", age: 24, grade: "B", gpa: 3.2, major: "Mathematics" },
  { name: "Charlie Brown", age: 21, grade: "A-", gpa: 3.7, major: "Physics" },
  { name: "Diana Prince", age: 23, grade: "B+", gpa: 3.5, major: "Computer Science" },
  { name: "Eve Williams", age: 25, grade: "A", gpa: 3.8, major: "Statistics" },
]

# เขียน CSV
puts "=== Writing CSV ==="
CSV.open("/tmp/students.csv", "w") do |csv|
  # header row
  csv << ["Name", "Age", "Grade", "GPA", "Major"]
  students.each do |s|
    csv << [s[:name], s[:age], s[:grade], s[:gpa], s[:major]]
  end
end
puts "Written to students.csv"
puts File.read("/tmp/students.csv")

# เขียนด้วย headers
puts "\n=== Writing with headers ==="
CSV.open("/tmp/students2.csv", "w", headers: true) do |csv|
  csv << ["Name", "Age", "Grade", "GPA", "Major"]
  students.each do |s|
    csv << [s[:name], s[:age], s[:grade], s[:gpa], s[:major]]
  end
end

# อ่าน CSV พื้นฐาน
puts "=== Reading CSV (basic) ==="
CSV.foreach("/tmp/students.csv") do |row|
  puts row.inspect
end

# อ่านด้วย headers
puts "\n=== Reading CSV with headers ==="
CSV.foreach("/tmp/students.csv", headers: true) do |row|
  puts "#{row['Name']}: GPA #{row['GPA']}, Major: #{row['Major']}"
end

# อ่านทั้งหมดเป็น array
puts "\n=== Read all as array ==="
rows = CSV.read("/tmp/students.csv", headers: true)
puts "Total students: #{rows.length}"
cs_students = rows.select { |r| r['Major'] == 'Computer Science' }
puts "CS students: #{cs_students.map { |r| r['Name'] }.inspect}"

# อ่านพร้อม type conversion
puts "\n=== Reading with converters ==="
CSV.foreach("/tmp/students.csv", headers: true, converters: :numeric) do |row|
  puts "#{row['Name']}: Age #{row['Age'].class} = #{row['Age']}"
  break  # แสดงแค่แถวแรก
end

# CSV จาก string
puts "\n=== Parse CSV string ==="
csv_string = "Alice,30\nBob,25\nCharlie,35"
CSV.parse(csv_string) do |row|
  puts "Name: #{row[0]}, Age: #{row[1]}"
end

# เขียน CSV จาก objects
puts "\n=== Writing objects to CSV ==="
class Product
  attr_reader :name, :price, :quantity, :category

  def initialize(name, price, quantity, category)
    @name = name
    @price = price
    @quantity = quantity
    @category = category
  end

  def to_csv_row
    [name, price, quantity, category]
  end
end

products = [
  Product.new("Laptop", 25000, 10, "Electronics"),
  Product.new("Book", 350, 100, "Education"),
  Product.new("Headphones", 1500, 30, "Electronics"),
]

CSV.open("/tmp/products.csv", "w", headers: ["Name", "Price", "Qty", "Category"], write_headers: true) do |csv|
  products.each { |p| csv << p.to_csv_row }
end
puts File.read("/tmp/products.csv")
```

### CSV ที่ซับซ้อนกว่า

```ruby
require 'csv'

# อ่าน CSV กับ custom separator
data = "Alice|30|alice@example.com\nBob|25|bob@example.com"
File.write("/tmp/pipe.csv", data)

CSV.foreach("/tmp/pipe.csv", col_sep: "|") do |row|
  puts "#{row[0]} (#{row[1]}): #{row[2]}"
end

# CSV กับ special characters
special_data = [
  ["Alice, Jr.", "Developer", "She said \"Hello\""],
  ["Bob\nSmith", "Designer", "Multi\nline"],
]

CSV.open("/tmp/special.csv", "w") do |csv|
  special_data.each { |row| csv << row }
end

puts "\nSpecial CSV content:"
puts File.read("/tmp/special.csv")

puts "\nRead back:"
CSV.foreach("/tmp/special.csv") do |row|
  puts row.map(&:inspect).join(" | ")
end

# Merge CSV files
def merge_csv_files(files, output, headers: true)
  all_rows = []
  header_row = nil

  files.each do |file|
    rows = CSV.read(file, headers: headers)
    if headers && header_row.nil?
      header_row = rows.headers
    end
    all_rows.concat(rows.map(&:to_a))
  end

  CSV.open(output, "w") do |csv|
    csv << header_row if header_row
    all_rows.each { |row| csv << row.map(&:last) }
  end
  puts "Merged #{files.length} files into #{output}"
end
```

---

## Step 310: JSON Reading/Writing

```ruby
require 'json'

# ข้อมูล Ruby
config = {
  app_name: "MyRubyApp",
  version: "2.1.0",
  database: {
    host: "localhost",
    port: 5432,
    name: "myapp_production",
    pool_size: 10
  },
  redis: {
    host: "localhost",
    port: 6379,
    db: 0
  },
  features: {
    dark_mode: true,
    beta_features: false,
    max_upload_mb: 100
  },
  allowed_origins: ["https://example.com", "https://api.example.com"],
  admin_emails: ["admin@example.com", "ops@example.com"]
}

# เขียน JSON
puts "=== Writing JSON ==="
json_string = JSON.generate(config)
File.write("/tmp/config.json", json_string)
puts "Written (compact)"

# เขียน pretty JSON
pretty_json = JSON.pretty_generate(config)
File.write("/tmp/config_pretty.json", pretty_json)
puts "Written (pretty)"
puts pretty_json

# อ่าน JSON
puts "\n=== Reading JSON ==="
raw = File.read("/tmp/config.json")
parsed = JSON.parse(raw)
puts "App: #{parsed['app_name']} v#{parsed['version']}"
puts "DB: #{parsed['database']['host']}:#{parsed['database']['port']}/#{parsed['database']['name']}"

# อ่านด้วย symbolize_names
parsed_sym = JSON.parse(raw, symbolize_names: true)
puts "DB Pool: #{parsed_sym[:database][:pool_size]}"
puts "Features: #{parsed_sym[:features].inspect}"

# JSON.load vs JSON.parse
# JSON.parse - อ่าน string
# JSON.load - อ่าน string หรือ IO object

File.open("/tmp/config.json") do |f|
  data = JSON.load(f)
  puts "Loaded from file: #{data['app_name']}"
end

# แปลง objects เป็น JSON
class User
  attr_reader :id, :name, :email, :created_at

  def initialize(id, name, email)
    @id = id
    @name = name
    @email = email
    @created_at = Time.now.iso8601
  end

  def to_json(*args)
    {
      id: @id,
      name: @name,
      email: @email,
      created_at: @created_at
    }.to_json(*args)
  end

  def to_h
    { id: @id, name: @name, email: @email, created_at: @created_at }
  end
end

users = [
  User.new(1, "Alice Smith", "alice@example.com"),
  User.new(2, "Bob Jones", "bob@example.com"),
  User.new(3, "Charlie Brown", "charlie@example.com"),
]

puts "\n=== Writing objects to JSON ==="
json = JSON.pretty_generate(users.map(&:to_h))
File.write("/tmp/users.json", json)
puts json

# อ่าน JSON array
puts "\n=== Reading JSON array ==="
loaded_users = JSON.parse(File.read("/tmp/users.json"), symbolize_names: true)
loaded_users.each do |u|
  puts "#{u[:id]}. #{u[:name]} <#{u[:email]}>"
end

# JSON กับ complex nested structure
puts "\n=== Complex JSON ==="
api_response = {
  status: "success",
  data: {
    users: users.map(&:to_h),
    pagination: { page: 1, per_page: 10, total: 3, total_pages: 1 },
    meta: { request_id: "req-#{rand(10000)}", timestamp: Time.now.iso8601 }
  }
}

File.write("/tmp/api_response.json", JSON.pretty_generate(api_response))
puts "API response written to api_response.json"

# อ่านและ process
data = JSON.parse(File.read("/tmp/api_response.json"), symbolize_names: true)
puts "Status: #{data[:status]}"
puts "Total users: #{data[:data][:pagination][:total]}"
data[:data][:users].each do |u|
  puts "  - #{u[:name]}"
end
```

---

## Step 311: YAML Reading/Writing

```ruby
require 'yaml'

# YAML (YAML Ain't Markup Language) - human readable config format

config = {
  application: {
    name: "MyApp",
    version: "1.0.0",
    environment: "development"
  },
  server: {
    host: "0.0.0.0",
    port: 3000,
    workers: 4,
    timeout: 30
  },
  database: {
    adapter: "postgresql",
    host: "localhost",
    port: 5432,
    database: "myapp_dev",
    username: "myapp",
    password: "secret",
    pool: 5
  },
  logging: {
    level: "debug",
    file: "/var/log/myapp.log",
    rotate: true,
    max_size_mb: 100
  },
  features: ["authentication", "authorization", "api", "admin_panel"]
}

# เขียน YAML
yaml_string = YAML.dump(config)
File.write("/tmp/config.yml", yaml_string)
puts "=== Written YAML ==="
puts yaml_string

# อ่าน YAML
puts "=== Reading YAML ==="
loaded = YAML.safe_load(File.read("/tmp/config.yml"), symbolize_names: true)
puts "App: #{loaded[:application][:name]} v#{loaded[:application][:version]}"
puts "DB: #{loaded[:database][:host]}:#{loaded[:database][:port]}"
puts "Features: #{loaded[:features].inspect}"

# YAML.load_file
loaded2 = YAML.safe_load(File.read("/tmp/config.yml"))
puts "\nLoaded as string keys:"
puts "Server port: #{loaded2['server']['port']}"

# YAML string literals
yaml_text = <<~YAML
  name: Alice Smith
  age: 30
  hobbies:
    - Programming
    - Reading
    - Hiking
  address:
    city: Bangkok
    country: Thailand
  active: true
YAML

person = YAML.safe_load(yaml_text)
puts "\n=== Parsed YAML literal ==="
puts "Name: #{person['name']}"
puts "Age: #{person['age']}"
puts "Hobbies: #{person['hobbies'].inspect}"
puts "City: #{person['address']['city']}"

# YAML สำหรับ objects
class AppConfig
  attr_accessor :debug, :log_level, :max_connections, :api_key

  def initialize
    @debug = false
    @log_level = :info
    @max_connections = 10
    @api_key = nil
  end

  def to_yaml_hash
    {
      debug: @debug,
      log_level: @log_level.to_s,
      max_connections: @max_connections,
      api_key: @api_key
    }
  end

  def self.from_yaml(filename)
    data = YAML.safe_load(File.read(filename), symbolize_names: true)
    config = new
    config.debug = data[:debug]
    config.log_level = data[:log_level]&.to_sym
    config.max_connections = data[:max_connections]
    config.api_key = data[:api_key]
    config
  end

  def save_yaml(filename)
    File.write(filename, YAML.dump(to_yaml_hash))
  end
end

app_config = AppConfig.new
app_config.debug = true
app_config.log_level = :debug
app_config.max_connections = 20
app_config.api_key = "sk-abc123"

app_config.save_yaml("/tmp/app_config.yml")
puts "\n=== Saved AppConfig ==="
puts File.read("/tmp/app_config.yml")

loaded_config = AppConfig.from_yaml("/tmp/app_config.yml")
puts "Loaded debug: #{loaded_config.debug}"
puts "Loaded log_level: #{loaded_config.log_level}"
```

### YAML กับ Complex Types

```ruby
require 'yaml'

# YAML รองรับ Ruby objects บางประเภท
data = {
  created_at: Time.now,
  tags: [:ruby, :programming, :tutorial],
  nested: {
    array_of_hashes: [
      { name: "Alice", score: 95 },
      { name: "Bob", score: 87 }
    ]
  },
  multiline_string: "This is a\nmultiline\nstring"
}

yaml = YAML.dump(data)
puts yaml

# โหลดกลับ (ใช้ permitted_classes สำหรับ custom types)
loaded = YAML.safe_load(yaml, permitted_classes: [Symbol, Time])
puts "Created at: #{loaded['created_at']}"
puts "Tags: #{loaded['tags'].inspect}"
```

---

## Step 312: Pathname Class

```ruby
require 'pathname'

# Pathname เป็น OOP wrapper สำหรับ File paths
home = Pathname.new(Dir.home)
puts "Home: #{home}"
puts "Class: #{home.class}"

# เชื่อมต่อ paths ด้วย /
docs = home / "Documents"
project = docs / "my_project"
puts "Project: #{project}"

# Pathname methods
path = Pathname.new("/home/user/projects/myapp/src/main.rb")

puts "\n=== Pathname methods ==="
puts "dirname:   #{path.dirname}"
puts "basename:  #{path.basename}"
puts "basename without ext: #{path.basename('.rb')}"
puts "extname:   #{path.extname}"
puts "to_s:      #{path.to_s}"

# Relative paths
current = Pathname.new("/home/user/projects")
file = Pathname.new("/home/user/projects/myapp/src/main.rb")
puts "\nRelative: #{file.relative_path_from(current)}"

# ตรวจสอบ
tmp = Pathname.new("/tmp")
test_path = tmp / "pathname_test.txt"

test_path.write("Hello from Pathname!")
puts "\n=== File operations ==="
puts "exists?:    #{test_path.exist?}"
puts "file?:      #{test_path.file?}"
puts "directory?: #{test_path.directory?}"
puts "size:       #{test_path.size}"
puts "mtime:      #{test_path.mtime}"
puts "readable?:  #{test_path.readable?}"
puts "writable?:  #{test_path.writable?}"

# อ่านไฟล์
content = test_path.read
puts "Content: #{content}"

# iterate directory
puts "\n=== Directory iteration ==="
Dir.mkdir("/tmp/pathname_dir") unless Dir.exist?("/tmp/pathname_dir")
["a.txt", "b.rb", "c.csv"].each { |f| (Pathname.new("/tmp/pathname_dir") / f).write("") }

dir = Pathname.new("/tmp/pathname_dir")
dir.each_child do |child|
  puts "  #{child.basename} (#{child.extname})"
end

# glob
puts "\n=== Glob ==="
ruby_files = dir.glob("*.rb")
puts "Ruby files: #{ruby_files.map(&:basename).inspect}"

# ทำ path manipulation
puts "\n=== Path manipulation ==="
path2 = Pathname.new("relative/path/to/file.txt")
puts "absolute?: #{path2.absolute?}"
puts "relative?: #{path2.relative?}"
puts "absolute:  #{path2.expand_path}"

# Cleanup
test_path.delete
FileUtils.rm_rf("/tmp/pathname_dir")
```

---

## Step 313: Tempfile

```ruby
require 'tempfile'

# Tempfile - ไฟล์ชั่วคราวที่ลบตัวเองอัตโนมัติ

puts "=== Basic Tempfile ==="
temp = Tempfile.new("myapp")
puts "Path: #{temp.path}"
puts "Name: #{File.basename(temp.path)}"

temp.write("Temporary data\nMore data\n")
temp.flush  # ทำให้ data ถูกเขียนลงดิสก์
temp.rewind

puts "Content:"
puts temp.read

temp.close
temp.unlink  # ลบไฟล์

# ใช้ Tempfile กับ block
puts "\n=== Tempfile with block ==="
Tempfile.create("prefix") do |f|
  puts "Temp file: #{f.path}"
  f.puts "Line 1"
  f.puts "Line 2"
  f.puts "Line 3"
  f.rewind
  puts f.read
end
# ลบอัตโนมัติหลัง block

# Tempfile กับ extension
Tempfile.create(["data", ".csv"]) do |f|
  puts "CSV temp: #{f.path}"
  require 'csv'
  CSV.new(f) do |csv|
    csv << ["name", "age"]
    csv << ["Alice", 30]
    csv << ["Bob", 25]
  end
  f.rewind
  puts f.read
end

# ใช้งาน tempfile ใน processing pipeline
def process_large_data(data_array)
  Tempfile.create(["processing", ".txt"]) do |temp|
    # เขียนลง temp file
    data_array.each { |item| temp.puts item }
    temp.flush
    temp.rewind

    # ประมวลผลจาก temp file
    result = []
    temp.each_line do |line|
      processed = line.chomp.upcase
      result << processed
    end
    result
  end
end

data = ["hello world", "ruby programming", "file io"]
result = process_large_data(data)
puts "\n=== Processing result ==="
result.each { |r| puts "  #{r}" }

# Tempfile กับ binary data
puts "\n=== Binary Tempfile ==="
Tempfile.create(["image", ".png"], mode: File::BINARY) do |f|
  # Simulate writing binary data
  f.write([137, 80, 78, 71].pack("C*"))  # PNG magic bytes
  f.write("\x00" * 100)  # padding
  puts "Binary file size: #{f.size} bytes"
end
```

---

## Step 314: StringIO (In-Memory IO)

```ruby
require 'stringio'

# StringIO คือ IO object ที่ทำงานกับ String ใน memory แทน file

puts "=== Basic StringIO ==="
sio = StringIO.new
sio.puts "Line 1"
sio.puts "Line 2"
sio.puts "Line 3"
sio.rewind

puts "Content:"
puts sio.read

# อ่านทีละบรรทัด
sio.rewind
sio.each_line { |line| print "  > #{line}" }

# StringIO เหมือน File
puts "\n=== StringIO like File ==="
sio2 = StringIO.new("Hello, World!\nRuby IO\nTest\n")
puts "Size: #{sio2.size}"
puts "First line: #{sio2.readline.chomp}"
puts "Position: #{sio2.pos}"
sio2.seek(0)
puts "After seek: #{sio2.readline.chomp}"

# ใช้ StringIO แทน $stdout สำหรับ capture output
puts "\n=== Capture output ==="
captured = StringIO.new

old_stdout = $stdout
$stdout = captured

puts "This goes to StringIO"
puts "Not to the terminal"
print "Final line"

$stdout = old_stdout
puts "Back to real stdout"
puts "Captured: #{captured.string.inspect}"

# ใช้ใน test
def capture_output
  original = $stdout
  $stdout = StringIO.new
  yield
  output = $stdout.string
  $stdout = original
  output
end

output = capture_output do
  puts "Test output 1"
  puts "Test output 2"
  print "No newline"
end

puts "Captured output: #{output.inspect}"

# StringIO กับ CSV
puts "\n=== StringIO with CSV ==="
require 'csv'

csv_io = StringIO.new
CSV(csv_io) do |csv|
  csv << ["Name", "Score"]
  csv << ["Alice", 95]
  csv << ["Bob", 87]
  csv << ["Charlie", 92]
end

puts "CSV content:"
puts csv_io.string

# parse จาก StringIO
csv_io.rewind
CSV(csv_io, headers: true) do |csv|
  csv.each { |row| puts "  #{row['Name']}: #{row['Score']}" }
end

# StringIO กับ JSON streaming
puts "\n=== StringIO with JSON ==="
require 'json'

json_io = StringIO.new
json_io.write(JSON.generate({
  users: [
    { id: 1, name: "Alice" },
    { id: 2, name: "Bob" }
  ],
  count: 2
}))

json_io.rewind
data = JSON.load(json_io)
puts "Users: #{data['users'].map { |u| u['name'] }.inspect}"

# StringIO เป็น drop-in replacement สำหรับ File
def read_lines_from_io(io)
  lines = []
  io.each_line { |line| lines << line.chomp }
  lines
end

# ทำงานกับ real file
File.write("/tmp/test_sio.txt", "Line A\nLine B\nLine C\n")
file_lines = File.open("/tmp/test_sio.txt") { |f| read_lines_from_io(f) }
puts "\nFrom file: #{file_lines.inspect}"

# ทำงานกับ StringIO (เช่น ใน test)
sio3 = StringIO.new("Line X\nLine Y\nLine Z\n")
sio_lines = read_lines_from_io(sio3)
puts "From StringIO: #{sio_lines.inspect}"
```

---

## Step 315-320: แบบฝึกหัด 20 ข้อ พร้อมเฉลย

### ข้อที่ 1: File Line Counter

```ruby
class FileAnalyzer
  def initialize(filepath)
    @filepath = filepath
  end

  def analyze
    return nil unless File.exist?(@filepath)

    stats = {
      filename: File.basename(@filepath),
      size_bytes: File.size(@filepath),
      total_lines: 0,
      blank_lines: 0,
      code_lines: 0,
      comment_lines: 0,
      words: 0,
      chars: 0
    }

    File.foreach(@filepath, chomp: true) do |line|
      stats[:total_lines] += 1
      stats[:words] += line.split.length
      stats[:chars] += line.length

      if line.strip.empty?
        stats[:blank_lines] += 1
      elsif line.strip.start_with?('#')
        stats[:comment_lines] += 1
      else
        stats[:code_lines] += 1
      end
    end

    stats
  end

  def report
    stats = analyze
    return puts "File not found: #{@filepath}" unless stats

    puts "=== File Analysis: #{stats[:filename]} ==="
    puts "Size:          #{stats[:size_bytes]} bytes"
    puts "Total lines:   #{stats[:total_lines]}"
    puts "Code lines:    #{stats[:code_lines]}"
    puts "Comment lines: #{stats[:comment_lines]}"
    puts "Blank lines:   #{stats[:blank_lines]}"
    puts "Words:         #{stats[:words]}"
    puts "Characters:    #{stats[:chars]}"
  end
end

# สร้างไฟล์ทดสอบ
test_content = <<~RUBY
  # This is a Ruby file
  # Author: Test

  class Greeter
    # initialize method
    def initialize(name)
      @name = name
    end

    def greet
      puts "Hello, #\{@name\}!"
    end
  end

  # Main program
  greeter = Greeter.new("World")
  greeter.greet
RUBY

File.write("/tmp/analyzer_test.rb", test_content)
analyzer = FileAnalyzer.new("/tmp/analyzer_test.rb")
analyzer.report
```

### ข้อที่ 2: Log File Parser

```ruby
class LogParser
  LOG_PATTERN = /\[(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\] \[(\w+)\] (.+)/

  def initialize(log_file)
    @log_file = log_file
  end

  def parse_all
    File.foreach(@log_file, chomp: true).filter_map do |line|
      parse_line(line)
    end
  end

  def by_level(level)
    parse_all.select { |entry| entry[:level] == level.to_s.upcase }
  end

  def errors
    by_level("ERROR")
  end

  def summary
    entries = parse_all
    counts = entries.group_by { |e| e[:level] }.transform_values(&:length)
    {
      total: entries.length,
      by_level: counts,
      first_entry: entries.first&.dig(:timestamp),
      last_entry: entries.last&.dig(:timestamp)
    }
  end

  def search(pattern)
    parse_all.select { |entry| entry[:message] =~ Regexp.new(pattern, 'i') }
  end

  private

  def parse_line(line)
    match = LOG_PATTERN.match(line)
    return nil unless match

    {
      timestamp: match[1],
      level: match[2],
      message: match[3]
    }
  end
end

# สร้าง log ทดสอบ
log_content = <<~LOG
  [2024-01-15 09:00:01] [INFO] Application started
  [2024-01-15 09:00:02] [DEBUG] Loading configuration
  [2024-01-15 09:01:00] [INFO] Connected to database
  [2024-01-15 09:01:30] [WARN] Low memory: 85% used
  [2024-01-15 09:02:00] [ERROR] Failed to connect to cache server
  [2024-01-15 09:02:01] [INFO] Retrying connection...
  [2024-01-15 09:02:05] [INFO] Connected to cache server
  [2024-01-15 09:05:00] [ERROR] Database query timeout after 30s
  [2024-01-15 09:05:01] [WARN] High CPU usage: 95%
  [2024-01-15 09:10:00] [INFO] Performing backup
  [2024-01-15 09:10:30] [INFO] Backup completed
  [2024-01-15 09:15:00] [ERROR] Out of disk space: /var/log
LOG

File.write("/tmp/app.log", log_content)

parser = LogParser.new("/tmp/app.log")

puts "=== Log Summary ==="
summary = parser.summary
puts "Total entries: #{summary[:total]}"
summary[:by_level].each { |level, count| puts "  #{level}: #{count}" }

puts "\n=== Errors ==="
parser.errors.each { |e| puts "  #{e[:timestamp]}: #{e[:message]}" }

puts "\n=== Search 'connection' ==="
parser.search("connection").each { |e| puts "  [#{e[:level]}] #{e[:message]}" }
```

### ข้อที่ 3: Configuration Manager

```ruby
require 'yaml'
require 'json'

class ConfigManager
  attr_reader :config, :source

  def initialize(filepath = nil)
    @config = {}
    @source = filepath
    @watchers = {}
    load_file(filepath) if filepath
  end

  def load_file(filepath)
    raise "File not found: #{filepath}" unless File.exist?(filepath)
    @source = filepath

    @config = case File.extname(filepath)
    when ".yaml", ".yml"
      YAML.safe_load(File.read(filepath), symbolize_names: true) || {}
    when ".json"
      JSON.parse(File.read(filepath), symbolize_names: true)
    else
      raise "Unsupported format: #{File.extname(filepath)}"
    end.transform_keys(&:to_sym)

    self
  end

  def get(key_path, default = nil)
    keys = key_path.to_s.split('.').map(&:to_sym)
    keys.reduce(@config) do |current, key|
      case current
      when Hash then current[key]
      else return default
      end
    end || default
  end

  def set(key_path, value)
    keys = key_path.to_s.split('.').map(&:to_sym)
    target = keys[0..-2].reduce(@config) do |current, key|
      current[key] ||= {}
      current[key]
    end
    target[keys.last] = value
    trigger_watcher(key_path.to_s, value)
    self
  end

  def delete(key_path)
    keys = key_path.to_s.split('.').map(&:to_sym)
    parent = keys[0..-2].reduce(@config) { |c, k| c[k] || {} }
    parent.delete(keys.last)
    self
  end

  def save(filepath = @source)
    raise "No filepath specified" unless filepath
    content = case File.extname(filepath)
    when ".yaml", ".yml" then YAML.dump(@config.transform_keys(&:to_s))
    when ".json" then JSON.pretty_generate(@config)
    else raise "Unsupported format"
    end
    File.write(filepath, content)
    puts "Config saved to #{filepath}"
    self
  end

  def on_change(key_path, &block)
    @watchers[key_path.to_s] = block
    self
  end

  def merge!(other_config)
    case other_config
    when Hash then @config.merge!(other_config)
    when ConfigManager then @config.merge!(other_config.config)
    end
    self
  end

  def to_h
    @config.dup
  end

  def [](key)
    get(key)
  end

  def []=(key, value)
    set(key, value)
  end

  private

  def trigger_watcher(key, value)
    @watchers[key]&.call(value)
  end
end

# ทดสอบ
yaml_config = <<~YAML
  app:
    name: MyApp
    version: 1.0.0
    debug: false
  database:
    host: localhost
    port: 5432
    pool: 10
  features:
    - auth
    - api
    - admin
YAML

File.write("/tmp/test_config.yml", yaml_config)

config = ConfigManager.new("/tmp/test_config.yml")

puts "App: #{config.get('app.name')} v#{config.get('app.version')}"
puts "DB: #{config.get('database.host')}:#{config.get('database.port')}"
puts "Features: #{config.get('features').inspect}"

# Watch for changes
config.on_change("database.host") do |new_value|
  puts "Database host changed to: #{new_value}"
end

config.set("database.host", "production-db.example.com")
config.set("app.debug", true)
config.set("cache.redis.host", "redis.example.com")

puts "\nUpdated config:"
puts "DB host: #{config.get('database.host')}"
puts "Debug: #{config.get('app.debug')}"
puts "Cache Redis: #{config.get('cache.redis.host')}"

config.save("/tmp/test_config_updated.yml")
puts "\nUpdated YAML:"
puts File.read("/tmp/test_config_updated.yml")
```

### ข้อที่ 4: CSV Data Processor

```ruby
require 'csv'

class CSVProcessor
  attr_reader :headers, :rows

  def initialize(filepath_or_string, headers: true, **csv_options)
    @headers = []
    @rows = []
    load_data(filepath_or_string, headers: headers, **csv_options)
  end

  def filter(&block)
    filtered = @rows.select(&block)
    result = self.class.from_rows(@headers, filtered)
    result
  end

  def map_rows(&block)
    mapped = @rows.map(&block)
    self.class.from_rows(@headers, mapped)
  end

  def sort_by(column, direction: :asc)
    sorted = @rows.sort_by { |row| row[column.to_s] || "" }
    sorted.reverse! if direction == :desc
    self.class.from_rows(@headers, sorted)
  end

  def group_by(column)
    @rows.group_by { |row| row[column.to_s] }
  end

  def aggregate(group_col, value_col, operation: :sum)
    grouped = group_by(group_col)
    grouped.transform_values do |rows|
      values = rows.map { |r| r[value_col.to_s].to_f }
      case operation
      when :sum then values.sum
      when :avg then values.sum / values.length
      when :min then values.min
      when :max then values.max
      when :count then values.length
      end
    end
  end

  def to_csv
    require 'stringio'
    sio = StringIO.new
    CSV(sio) do |csv|
      csv << @headers
      @rows.each { |row| csv << @headers.map { |h| row[h] } }
    end
    sio.string
  end

  def save(filepath)
    File.write(filepath, to_csv)
    puts "Saved #{@rows.length} rows to #{filepath}"
  end

  def count
    @rows.length
  end

  def summary
    {
      rows: count,
      columns: @headers.length,
      headers: @headers
    }
  end

  def self.from_rows(headers, rows)
    sio = StringIO.new
    CSV(sio) do |csv|
      csv << headers
      rows.each { |row| csv << headers.map { |h| row[h] } }
    end
    sio.rewind
    new(sio, headers: true)
  end

  private

  def load_data(source, headers:, **opts)
    csv_opts = { headers: headers }.merge(opts)
    data = case source
    when String
      if File.exist?(source)
        CSV.read(source, **csv_opts)
      else
        CSV.parse(source, **csv_opts)
      end
    when StringIO, IO
      CSV.new(source, **csv_opts).read
    end

    if headers
      @headers = data.headers || []
      @rows = data.map(&:to_h)
    else
      @rows = data
    end
  end
end

# สร้าง CSV ทดสอบ
csv_data = <<~CSV
  name,department,salary,years_experience,performance
  Alice Smith,Engineering,85000,5,Excellent
  Bob Jones,Marketing,65000,3,Good
  Charlie Brown,Engineering,95000,8,Excellent
  Diana Prince,HR,60000,2,Good
  Eve Williams,Engineering,75000,4,Average
  Frank Miller,Marketing,70000,6,Good
  Grace Lee,HR,62000,3,Excellent
  Henry Ford,Engineering,110000,12,Excellent
CSV

File.write("/tmp/employees.csv", csv_data)

proc = CSVProcessor.new("/tmp/employees.csv")
puts "=== Summary ==="
puts proc.summary.inspect

puts "\n=== Engineers ==="
engineers = proc.filter { |r| r['department'] == 'Engineering' }
engineers.rows.each { |r| puts "  #{r['name']}: $#{r['salary']}" }

puts "\n=== Sorted by salary (desc) ==="
sorted = proc.sort_by('salary', direction: :desc)
sorted.rows.first(3).each { |r| puts "  #{r['name']}: $#{r['salary']}" }

puts "\n=== Avg salary by department ==="
avg = proc.aggregate('department', 'salary', operation: :avg)
avg.each { |dept, avg_sal| puts "  #{dept}: $#{avg_sal.round(0)}" }

puts "\n=== Excellent performers ==="
excellent = proc.filter { |r| r['performance'] == 'Excellent' }
puts "Count: #{excellent.count}"
excellent.rows.each { |r| puts "  #{r['name']} (#{r['department']})" }
```

### ข้อที่ 5-10: เพิ่มเติม

```ruby
# ข้อที่ 5: JSON Config with Defaults
require 'json'

class JSONConfig
  DEFAULT_CONFIG = {
    "server" => { "host" => "localhost", "port" => 3000, "workers" => 2 },
    "database" => { "host" => "localhost", "port" => 5432, "pool" => 5 },
    "cache" => { "enabled" => true, "ttl" => 3600 },
    "logging" => { "level" => "info", "colorize" => false }
  }.freeze

  def initialize(filepath = nil)
    @config = deep_dup(DEFAULT_CONFIG)
    load(filepath) if filepath && File.exist?(filepath)
  end

  def load(filepath)
    data = JSON.parse(File.read(filepath))
    @config = deep_merge(@config, data)
    self
  end

  def save(filepath)
    File.write(filepath, JSON.pretty_generate(@config))
    puts "Saved to #{filepath}"
  end

  def [](key)
    @config[key.to_s]
  end

  def get(path, default = nil)
    parts = path.to_s.split('.')
    parts.reduce(@config) do |hash, key|
      return default unless hash.is_a?(Hash)
      hash[key]
    end || default
  end

  def to_h
    @config.dup
  end

  private

  def deep_dup(obj)
    case obj
    when Hash then obj.transform_values { |v| deep_dup(v) }
    when Array then obj.map { |v| deep_dup(v) }
    else obj
    end
  end

  def deep_merge(base, override)
    base.merge(override) do |_, old, new_val|
      old.is_a?(Hash) && new_val.is_a?(Hash) ? deep_merge(old, new_val) : new_val
    end
  end
end

config = JSONConfig.new
puts "Default server port: #{config.get('server.port')}"
puts "Default DB pool: #{config.get('database.pool')}"

# Override some settings
override = '{"server": {"port": 8080}, "logging": {"level": "debug", "colorize": true}}'
File.write("/tmp/override_config.json", override)

config2 = JSONConfig.new("/tmp/override_config.json")
puts "Overridden port: #{config2.get('server.port')}"  # 8080
puts "DB pool (default): #{config2.get('database.pool')}"  # 5 (default kept)
puts "Log level: #{config2.get('logging.level')}"  # debug
```

```ruby
# ข้อที่ 6: File Backup System
require 'fileutils'
require 'digest'

class BackupSystem
  def initialize(backup_dir)
    @backup_dir = backup_dir
    FileUtils.mkdir_p(backup_dir)
  end

  def backup(source)
    raise "Source not found: #{source}" unless File.exist?(source)

    timestamp = Time.now.strftime('%Y%m%d_%H%M%S')
    basename = File.basename(source)
    dest = File.join(@backup_dir, "#{timestamp}_#{basename}")

    FileUtils.cp(source, dest)
    checksum = Digest::MD5.file(dest).hexdigest

    record = {
      original: source,
      backup: dest,
      timestamp: timestamp,
      size: File.size(dest),
      checksum: checksum
    }

    log_backup(record)
    puts "Backed up: #{source} → #{dest}"
    record
  end

  def restore(backup_file, destination)
    raise "Backup not found: #{backup_file}" unless File.exist?(backup_file)

    FileUtils.cp(backup_file, destination)
    puts "Restored: #{backup_file} → #{destination}"
    true
  end

  def list_backups(pattern = "*")
    Dir.glob(File.join(@backup_dir, pattern)).map do |f|
      {
        path: f,
        basename: File.basename(f),
        size: File.size(f),
        modified: File.mtime(f)
      }
    end.sort_by { |b| b[:modified] }.reverse
  end

  def verify(backup_file, original_file)
    return false unless File.exist?(backup_file) && File.exist?(original_file)
    Digest::MD5.file(backup_file).hexdigest == Digest::MD5.file(original_file).hexdigest
  end

  def cleanup_old(keep_count: 5)
    backups = list_backups
    to_delete = backups.drop(keep_count)
    to_delete.each do |b|
      File.delete(b[:path])
      puts "Deleted old backup: #{b[:basename]}"
    end
    puts "Cleaned up #{to_delete.length} old backups"
  end

  private

  def log_backup(record)
    log_file = File.join(@backup_dir, "backup_log.json")
    logs = File.exist?(log_file) ? JSON.parse(File.read(log_file)) : []
    logs << record
    File.write(log_file, JSON.pretty_generate(logs))
  end
end

require 'json'

# ทดสอบ
backup_sys = BackupSystem.new("/tmp/backups")

# สร้างไฟล์ทดสอบ
File.write("/tmp/important.txt", "Important data #{Time.now}")

# Backup
record = backup_sys.backup("/tmp/important.txt")
puts "\nBackup record:"
puts record.except(:backup).inspect

# List backups
puts "\nAll backups:"
backup_sys.list_backups.each do |b|
  puts "  #{b[:basename]} (#{b[:size]} bytes)"
end

# Verify
is_valid = backup_sys.verify(record[:backup], "/tmp/important.txt")
puts "\nBackup valid: #{is_valid}"

# Modify original
File.write("/tmp/important.txt", "Modified data!")
is_still_valid = backup_sys.verify(record[:backup], "/tmp/important.txt")
puts "After modification valid: #{is_still_valid}"

# Restore
backup_sys.restore(record[:backup], "/tmp/restored.txt")
puts "Restored content: #{File.read("/tmp/restored.txt")}"

# Cleanup
FileUtils.rm_rf("/tmp/backups")
```

```ruby
# ข้อที่ 7: YAML Settings with Environment Override
require 'yaml'

class Settings
  def initialize(base_file, env_file = nil)
    @settings = {}
    load_base(base_file)
    load_env_override(env_file) if env_file && File.exist?(env_file)
  end

  def get(key_path)
    keys = key_path.split('.')
    keys.reduce(@settings) do |hash, key|
      return nil unless hash.is_a?(Hash)
      hash[key] || hash[key.to_sym]
    end
  end

  def all
    @settings.dup
  end

  def to_yaml
    YAML.dump(@settings)
  end

  private

  def load_base(file)
    @settings = YAML.safe_load(File.read(file)) || {}
  end

  def load_env_override(file)
    overrides = YAML.safe_load(File.read(file)) || {}
    @settings = deep_merge(@settings, overrides)
  end

  def deep_merge(base, override)
    base.merge(override) do |_, old, new_val|
      old.is_a?(Hash) && new_val.is_a?(Hash) ? deep_merge(old, new_val) : new_val
    end
  end
end

base_yaml = <<~YAML
  app:
    name: MyApp
    debug: false
  db:
    host: localhost
    port: 5432
  email:
    host: smtp.gmail.com
    port: 587
YAML

production_yaml = <<~YAML
  app:
    debug: false
  db:
    host: prod-db.example.com
  email:
    host: smtp.sendgrid.net
YAML

File.write("/tmp/base.yml", base_yaml)
File.write("/tmp/production.yml", production_yaml)

base_settings = Settings.new("/tmp/base.yml")
prod_settings = Settings.new("/tmp/base.yml", "/tmp/production.yml")

puts "=== Base Settings ==="
puts "DB Host: #{base_settings.get('db.host')}"
puts "Email: #{base_settings.get('email.host')}"

puts "\n=== Production Settings ==="
puts "DB Host: #{prod_settings.get('db.host')}"
puts "Email: #{prod_settings.get('email.host')}"
puts "App Name: #{prod_settings.get('app.name')}"  # from base
```

```ruby
# ข้อที่ 8: File Watcher (polling)
class FileWatcher
  def initialize(filepath, poll_interval: 1)
    @filepath = filepath
    @poll_interval = poll_interval
    @callbacks = {}
    @last_mtime = File.exist?(filepath) ? File.mtime(filepath) : nil
    @last_size = File.exist?(filepath) ? File.size(filepath) : 0
  end

  def on(event, &handler)
    @callbacks[event] ||= []
    @callbacks[event] << handler
    self
  end

  def check
    if !File.exist?(@filepath)
      if @last_mtime
        @last_mtime = nil
        @last_size = 0
        trigger(:deleted)
      end
      return
    end

    mtime = File.mtime(@filepath)
    size = File.size(@filepath)

    if @last_mtime.nil?
      @last_mtime = mtime
      @last_size = size
      trigger(:created)
    elsif mtime != @last_mtime
      @last_mtime = mtime
      @last_size = size
      trigger(:modified)
    end
  end

  def watch(duration_seconds: 5)
    end_time = Time.now + duration_seconds
    puts "Watching #{@filepath}..."

    while Time.now < end_time
      check
      sleep(@poll_interval)
    end

    puts "Done watching"
  end

  private

  def trigger(event)
    (@callbacks[event] || []).each { |cb| cb.call(@filepath, event) }
  end
end

watcher = FileWatcher.new("/tmp/watched.txt", poll_interval: 0.1)

watcher.on(:created) { |f, _| puts "Created: #{f}" }
watcher.on(:modified) { |f, _| puts "Modified: #{f} (size: #{File.size(f)})" }
watcher.on(:deleted) { |f, _| puts "Deleted: #{f}" }

# Simulate changes in another thread
Thread.new do
  sleep(0.2)
  File.write("/tmp/watched.txt", "Initial content")
  sleep(0.3)
  File.write("/tmp/watched.txt", "Modified content - more data here")
  sleep(0.3)
  File.delete("/tmp/watched.txt")
end

watcher.watch(duration_seconds: 1.5)
```

### ข้อที่ 9-20: Final exercises

```ruby
# ข้อที่ 9: CSV Report Generator
require 'csv'

class ReportGenerator
  def initialize(data)
    @data = data
  end

  def generate_summary_csv(output_file)
    CSV.open(output_file, "w") do |csv|
      csv << ["Report Generated", Time.now.strftime("%Y-%m-%d %H:%M:%S")]
      csv << []
      csv << ["Category", "Count", "Total", "Average", "Min", "Max"]

      groups = @data.group_by { |row| row[:category] }
      groups.each do |category, rows|
        values = rows.map { |r| r[:value] }
        csv << [
          category,
          values.length,
          values.sum.round(2),
          (values.sum / values.length).round(2),
          values.min.round(2),
          values.max.round(2)
        ]
      end

      csv << []
      total_values = @data.map { |r| r[:value] }
      csv << ["TOTAL", total_values.length, total_values.sum.round(2), "", "", ""]
    end
    puts "Summary report saved to #{output_file}"
  end

  def generate_detail_csv(output_file)
    CSV.open(output_file, "w", headers: true) do |csv|
      csv << ["ID", "Category", "Name", "Value", "Date"]
      @data.each_with_index do |row, i|
        csv << [i + 1, row[:category], row[:name], row[:value], row[:date]]
      end
    end
    puts "Detail report saved to #{output_file}"
  end
end

# สร้างข้อมูลทดสอบ
require 'date'
sales_data = [
  { category: "Electronics", name: "Laptop", value: 25000, date: "2024-01-15" },
  { category: "Electronics", name: "Phone", value: 15000, date: "2024-01-16" },
  { category: "Books", name: "Ruby Book", value: 350, date: "2024-01-15" },
  { category: "Books", name: "Design Patterns", value: 450, date: "2024-01-17" },
  { category: "Electronics", name: "Tablet", value: 12000, date: "2024-01-18" },
  { category: "Books", name: "Clean Code", value: 420, date: "2024-01-18" },
]

reporter = ReportGenerator.new(sales_data)
reporter.generate_summary_csv("/tmp/summary_report.csv")
reporter.generate_detail_csv("/tmp/detail_report.csv")

puts "\n=== Summary Report ==="
puts File.read("/tmp/summary_report.csv")
```

```ruby
# ข้อที่ 10: JSON API Response Cacher
require 'json'
require 'digest'

class APIResponseCache
  def initialize(cache_dir: "/tmp/api_cache", ttl: 3600)
    @cache_dir = cache_dir
    @ttl = ttl
    FileUtils.mkdir_p(cache_dir)
  end

  def fetch(url, &request)
    key = Digest::MD5.hexdigest(url)
    cache_file = File.join(@cache_dir, "#{key}.json")

    if cached?(cache_file)
      puts "[CACHE HIT] #{url}"
      data = JSON.parse(File.read(cache_file), symbolize_names: true)
      data[:body]
    else
      puts "[CACHE MISS] #{url}"
      response = request.call(url)
      store(cache_file, url, response)
      response
    end
  end

  def invalidate(url)
    key = Digest::MD5.hexdigest(url)
    cache_file = File.join(@cache_dir, "#{key}.json")
    if File.exist?(cache_file)
      File.delete(cache_file)
      puts "[CACHE INVALIDATED] #{url}"
    end
  end

  def clear_all
    Dir.glob(File.join(@cache_dir, "*.json")).each { |f| File.delete(f) }
    puts "[CACHE CLEARED]"
  end

  def stats
    files = Dir.glob(File.join(@cache_dir, "*.json"))
    {
      entries: files.length,
      total_size: files.sum { |f| File.size(f) },
      oldest: files.map { |f| File.mtime(f) }.min
    }
  end

  private

  def cached?(cache_file)
    return false unless File.exist?(cache_file)
    (Time.now - File.mtime(cache_file)) < @ttl
  end

  def store(cache_file, url, data)
    cache_data = {
      url: url,
      cached_at: Time.now.iso8601,
      expires_at: (Time.now + @ttl).iso8601,
      body: data
    }
    File.write(cache_file, JSON.pretty_generate(cache_data))
  end
end

require 'fileutils'
cache = APIResponseCache.new(ttl: 300)

# Simulate API calls
simulate_api = ->(url) {
  puts "  Making actual API request to: #{url}"
  { status: 200, data: "Response from #{url}", timestamp: Time.now.iso8601 }
}

# First call - cache miss
result1 = cache.fetch("https://api.example.com/users") { |url| simulate_api.call(url) }
puts "Got: #{result1[:data]}"

# Second call - cache hit
result2 = cache.fetch("https://api.example.com/users") { |url| simulate_api.call(url) }
puts "Got: #{result2[:data]}"

# Different URL - cache miss
result3 = cache.fetch("https://api.example.com/posts") { |url| simulate_api.call(url) }
puts "Got: #{result3[:data]}"

puts "\n=== Cache Stats ==="
stats = cache.stats
puts "Entries: #{stats[:entries]}"
puts "Total size: #{stats[:total_size]} bytes"

cache.invalidate("https://api.example.com/users")
result4 = cache.fetch("https://api.example.com/users") { |url| simulate_api.call(url) }
puts "After invalidation: #{result4[:data]}"

FileUtils.rm_rf("/tmp/api_cache")
```

### ข้อที่ 11-20

```ruby
# ข้อที่ 11: Multi-format Data Exporter
require 'json'
require 'csv'
require 'yaml'

class DataExporter
  def initialize(data, headers: nil)
    @data = data
    @headers = headers || (data.first.is_a?(Hash) ? data.first.keys.map(&:to_s) : [])
  end

  def to_json(pretty: true)
    if pretty
      JSON.pretty_generate(@data)
    else
      JSON.generate(@data)
    end
  end

  def to_csv(delimiter: ",")
    sio = StringIO.new
    CSV(sio, col_sep: delimiter) do |csv|
      csv << @headers unless @headers.empty?
      @data.each do |row|
        csv << case row
        when Hash then @headers.map { |h| row[h] || row[h.to_sym] }
        when Array then row
        end
      end
    end
    sio.string
  end

  def to_yaml
    YAML.dump(@data.map { |r| r.is_a?(Hash) ? r.transform_keys(&:to_s) : r })
  end

  def to_markdown_table
    return "" if @data.empty?

    rows = @data.map do |row|
      case row
      when Hash then @headers.map { |h| (row[h] || row[h.to_sym]).to_s }
      when Array then row.map(&:to_s)
      end
    end

    col_widths = @headers.each_with_index.map { |h, i|
      [h.length, rows.map { |r| (r[i] || "").length }.max || 0].max
    }

    header_row = @headers.each_with_index.map { |h, i|
      h.ljust(col_widths[i])
    }.join(" | ")

    separator = col_widths.map { |w| "-" * w }.join("-|-")
    data_rows = rows.map { |row|
      row.each_with_index.map { |cell, i| cell.ljust(col_widths[i]) }.join(" | ")
    }

    ["| #{header_row} |", "|-#{separator}-|", *data_rows.map { |r| "| #{r} |" }].join("\n")
  end

  def export(filename, format: nil)
    format ||= File.extname(filename).delete('.')
    content = case format.to_s
    when "json" then to_json
    when "csv" then to_csv
    when "yaml", "yml" then to_yaml
    when "md" then to_markdown_table
    else raise "Unsupported format: #{format}"
    end
    File.write(filename, content)
    puts "Exported to #{filename} (#{content.length} bytes)"
  end
end

data = [
  { name: "Alice", department: "Engineering", salary: 85000 },
  { name: "Bob", department: "Marketing", salary: 65000 },
  { name: "Charlie", department: "Engineering", salary: 95000 },
  { name: "Diana", department: "HR", salary: 60000 },
]

exporter = DataExporter.new(data)

puts "=== JSON ==="
puts exporter.to_json

puts "\n=== CSV ==="
puts exporter.to_csv

puts "\n=== Markdown Table ==="
puts exporter.to_markdown_table

exporter.export("/tmp/export_test.json")
exporter.export("/tmp/export_test.csv")
exporter.export("/tmp/export_test.md")
```

```ruby
# ข้อที่ 12-20: Final File I/O examples

# ข้อที่ 12: File Search Engine
class FileSearchEngine
  def initialize(root_dir)
    @root_dir = root_dir
  end

  def search_by_name(pattern, case_sensitive: false)
    flags = case_sensitive ? 0 : File::FNM_CASEFOLD
    Dir.glob(File.join(@root_dir, "**", "*")).select do |path|
      File.file?(path) &&
        File.fnmatch(pattern, File.basename(path), flags)
    end
  end

  def search_by_content(text, case_sensitive: false)
    results = []
    Dir.glob(File.join(@root_dir, "**", "*")).each do |path|
      next unless File.file?(path) && text_file?(path)
      begin
        content = File.read(path)
        if case_sensitive ? content.include?(text) : content.downcase.include?(text.downcase)
          matches = find_matches(content, text, case_sensitive)
          results << { file: path, matches: matches }
        end
      rescue
        # skip binary/unreadable files
      end
    end
    results
  end

  def search_by_size(min_bytes: 0, max_bytes: Float::INFINITY)
    Dir.glob(File.join(@root_dir, "**", "*")).select do |path|
      File.file?(path) &&
        File.size(path).between?(min_bytes, max_bytes)
    end
  end

  def search_by_date(after: nil, before: nil)
    Dir.glob(File.join(@root_dir, "**", "*")).select do |path|
      next unless File.file?(path)
      mtime = File.mtime(path)
      (!after || mtime >= after) && (!before || mtime <= before)
    end
  end

  def stats
    all_files = Dir.glob(File.join(@root_dir, "**", "*")).select { |p| File.file?(p) }
    {
      total_files: all_files.length,
      total_size: all_files.sum { |f| File.size(f) },
      by_extension: all_files.group_by { |f| File.extname(f) }.transform_values(&:length)
    }
  end

  private

  def text_file?(path)
    ext = File.extname(path).downcase
    ['.txt', '.rb', '.js', '.py', '.md', '.json', '.yaml', '.yml', '.csv', '.html', '.css', '.xml'].include?(ext)
  end

  def find_matches(content, text, case_sensitive)
    content.each_line.each_with_index.filter_map do |line, i|
      pattern = case_sensitive ? text : text.downcase
      haystack = case_sensitive ? line : line.downcase
      { line: i + 1, content: line.chomp } if haystack.include?(pattern)
    end
  end
end

# สร้างไฟล์ทดสอบ
FileUtils.mkdir_p("/tmp/search_test/src")
FileUtils.mkdir_p("/tmp/search_test/tests")

File.write("/tmp/search_test/README.md", "# My Ruby Project\nA sample project for testing.")
File.write("/tmp/search_test/src/user.rb", "class User\n  attr_reader :name\n  def initialize(name)\n    @name = name\n  end\nend")
File.write("/tmp/search_test/src/post.rb", "class Post\n  attr_reader :title\n  def initialize(title)\n    @title = title\n  end\nend")
File.write("/tmp/search_test/tests/user_test.rb", "require 'minitest'\nclass UserTest < Minitest::Test\n  def test_name\n    user = User.new('Alice')\n    assert_equal 'Alice', user.name\n  end\nend")
File.write("/tmp/search_test/config.yaml", "app:\n  name: TestApp\n  debug: true")

engine = FileSearchEngine.new("/tmp/search_test")

puts "=== Search by name (*.rb) ==="
engine.search_by_name("*.rb").each { |f| puts "  #{File.basename(f)}" }

puts "\n=== Search by content ('User') ==="
results = engine.search_by_content("User")
results.each do |r|
  puts "  #{File.basename(r[:file])}:"
  r[:matches].each { |m| puts "    Line #{m[:line]}: #{m[:content]}" }
end

puts "\n=== File stats ==="
stats = engine.stats
puts "Total files: #{stats[:total_files]}"
stats[:by_extension].each { |ext, count| puts "  #{ext.empty? ? 'no ext' : ext}: #{count}" }

FileUtils.rm_rf("/tmp/search_test")
```

```ruby
# ข้อที่ 13-20: More examples

# ข้อที่ 13: Rotating Log File
class RotatingLogger
  def initialize(base_path, max_size_kb: 100, max_files: 5)
    @base_path = base_path
    @max_size = max_size_kb * 1024
    @max_files = max_files
    ensure_dir
  end

  def log(level, message)
    rotate_if_needed
    File.open(current_log, "a") do |f|
      f.puts format_entry(level, message)
    end
  end

  def debug(msg) = log(:DEBUG, msg)
  def info(msg)  = log(:INFO, msg)
  def warn(msg)  = log(:WARN, msg)
  def error(msg) = log(:ERROR, msg)

  def all_log_files
    Dir.glob("#{@base_path}*.log").sort
  end

  def tail(n = 20)
    File.exist?(current_log) ? File.readlines(current_log).last(n) : []
  end

  private

  def current_log
    "#{@base_path}.log"
  end

  def ensure_dir
    FileUtils.mkdir_p(File.dirname(@base_path))
  end

  def rotate_if_needed
    return unless File.exist?(current_log) && File.size(current_log) >= @max_size
    rotate!
  end

  def rotate!
    timestamp = Time.now.strftime("%Y%m%d_%H%M%S")
    rotated = "#{@base_path}_#{timestamp}.log"
    FileUtils.mv(current_log, rotated)
    puts "Rotated log to #{File.basename(rotated)}"
    cleanup_old_logs
  end

  def cleanup_old_logs
    old_logs = Dir.glob("#{@base_path}_*.log").sort
    while old_logs.length >= @max_files
      oldest = old_logs.shift
      File.delete(oldest)
      puts "Deleted old log: #{File.basename(oldest)}"
    end
  end

  def format_entry(level, message)
    "[#{Time.now.strftime('%Y-%m-%d %H:%M:%S')}] [#{level}] #{message}"
  end
end

logger = RotatingLogger.new("/tmp/rotating_logs/app", max_size_kb: 1, max_files: 3)

50.times do |i|
  case i % 4
  when 0 then logger.info("Processing item #{i}")
  when 1 then logger.debug("Debug: step #{i}")
  when 2 then logger.warn("Warning at #{i}")
  when 3 then logger.error("Error occurred at #{i}")
  end
end

puts "\nLog files:"
logger.all_log_files.each { |f| puts "  #{File.basename(f)} (#{File.size(f)} bytes)" }

puts "\nLast 5 lines:"
logger.tail(5).each { |line| puts "  #{line.chomp}" }

FileUtils.rm_rf("/tmp/rotating_logs")
```

```ruby
# ข้อที่ 14-20: Final showcase

# ข้อที่ 14: Data Serialization Pipeline
require 'json'
require 'yaml'
require 'csv'

module SerializationPipeline
  def self.convert(data, from:, to:, **options)
    intermediate = parse(data, format: from)
    serialize(intermediate, format: to, **options)
  end

  def self.parse(data, format:)
    case format.to_sym
    when :json  then JSON.parse(data, symbolize_names: true)
    when :yaml  then YAML.safe_load(data, symbolize_names: true)
    when :csv
      rows = CSV.parse(data, headers: true)
      rows.map(&:to_h)
    when :ruby  then data  # already Ruby object
    end
  end

  def self.serialize(data, format:, **opts)
    case format.to_sym
    when :json  then opts[:pretty] ? JSON.pretty_generate(data) : JSON.generate(data)
    when :yaml  then YAML.dump(data.is_a?(Array) ? data.map { |r| r.transform_keys(&:to_s) } : data)
    when :csv
      return "" unless data.is_a?(Array) && !data.empty?
      headers = (data.first.keys rescue [])
      sio = StringIO.new
      CSV(sio) do |csv|
        csv << headers
        data.each { |row| csv << headers.map { |h| row[h] || row[h.to_s] } }
      end
      sio.string
    when :ruby  then data
    end
  end
end

# ทดสอบ conversion
json_data = '[{"name":"Alice","age":30},{"name":"Bob","age":25}]'
puts "=== JSON to CSV ==="
csv = SerializationPipeline.convert(json_data, from: :json, to: :csv)
puts csv

puts "=== JSON to YAML ==="
yaml = SerializationPipeline.convert(json_data, from: :json, to: :yaml)
puts yaml

puts "=== CSV to JSON ==="
csv_input = "name,age\nAlice,30\nBob,25\n"
json = SerializationPipeline.convert(csv_input, from: :csv, to: :json, pretty: true)
puts json
```

```ruby
# ข้อที่ 15-20 combined

# ข้อที่ 15: File Template Renderer
class TemplateRenderer
  def initialize(template_dir)
    @template_dir = template_dir
  end

  def render(template_name, variables = {})
    path = File.join(@template_dir, "#{template_name}.txt.erb")
    raise "Template not found: #{template_name}" unless File.exist?(path)

    template = File.read(path)
    render_erb(template, variables)
  end

  def render_to_file(template_name, output_file, variables = {})
    content = render(template_name, variables)
    File.write(output_file, content)
    puts "Rendered #{template_name} → #{output_file}"
    content
  end

  private

  def render_erb(template, vars)
    # Simple template replacement ({{var}})
    vars.each do |key, value|
      template = template.gsub("{{#{key}}}", value.to_s)
    end
    template
  end
end

# สร้าง templates
FileUtils.mkdir_p("/tmp/templates")

File.write("/tmp/templates/email.txt.erb", <<~TEMPLATE)
Dear {{name}},

Thank you for registering with {{app_name}}!

Your account details:
  Username: {{username}}
  Email: {{email}}

To activate your account, please click the link below:
{{activation_link}}

Best regards,
{{app_name}} Team
TEMPLATE

File.write("/tmp/templates/invoice.txt.erb", <<~TEMPLATE)
================================
INVOICE #{{invoice_number}}
================================
Date: {{date}}
Customer: {{customer_name}}

Items:
{{items}}

Subtotal: {{subtotal}}
Tax (7%): {{tax}}
TOTAL: {{total}}
================================
Thank you for your business!
================================
TEMPLATE

renderer = TemplateRenderer.new("/tmp/templates")

email = renderer.render("email", {
  name: "Alice Smith",
  app_name: "MyApp",
  username: "alice",
  email: "alice@example.com",
  activation_link: "https://myapp.com/activate/abc123"
})
puts email

FileUtils.rm_rf("/tmp/templates")
```

```ruby
# ข้อที่ 16-20: Advanced file operations

# ข้อที่ 16: Checksum verification
require 'digest'

class FileIntegrityChecker
  ALGORITHMS = { md5: Digest::MD5, sha1: Digest::SHA1, sha256: Digest::SHA256 }

  def compute(filepath, algorithm: :sha256)
    raise "File not found: #{filepath}" unless File.exist?(filepath)
    hasher = ALGORITHMS[algorithm] || Digest::SHA256

    hasher.file(filepath).hexdigest
  end

  def verify(filepath, expected_hash, algorithm: :sha256)
    actual = compute(filepath, algorithm: algorithm)
    actual == expected_hash
  end

  def create_manifest(directory, algorithm: :sha256)
    manifest = {}
    Dir.glob(File.join(directory, "**", "*")).each do |path|
      next unless File.file?(path)
      relative = path.sub("#{directory}/", "")
      manifest[relative] = compute(path, algorithm: algorithm)
    end
    manifest
  end

  def verify_manifest(directory, manifest, algorithm: :sha256)
    results = { valid: [], invalid: [], missing: [] }

    manifest.each do |relative_path, expected|
      full_path = File.join(directory, relative_path)
      if File.exist?(full_path)
        if verify(full_path, expected, algorithm: algorithm)
          results[:valid] << relative_path
        else
          results[:invalid] << relative_path
        end
      else
        results[:missing] << relative_path
      end
    end

    results
  end
end

checker = FileIntegrityChecker.new
FileUtils.mkdir_p("/tmp/integrity_test")

File.write("/tmp/integrity_test/file1.txt", "Content 1")
File.write("/tmp/integrity_test/file2.txt", "Content 2")

manifest = checker.create_manifest("/tmp/integrity_test")
puts "=== Manifest ==="
manifest.each { |f, h| puts "  #{f}: #{h[0..16]}..." }

# Verify (should all pass)
results = checker.verify_manifest("/tmp/integrity_test", manifest)
puts "\n=== Verification ==="
puts "Valid: #{results[:valid].inspect}"
puts "Invalid: #{results[:invalid].inspect}"
puts "Missing: #{results[:missing].inspect}"

# Modify a file and re-verify
File.write("/tmp/integrity_test/file1.txt", "MODIFIED CONTENT!")
results2 = checker.verify_manifest("/tmp/integrity_test", manifest)
puts "\n=== After modification ==="
puts "Invalid files: #{results2[:invalid].inspect}"

FileUtils.rm_rf("/tmp/integrity_test")
```

```ruby
# ข้อที่ 17-20: Final exercises

# ข้อที่ 17: Stream processing
class StreamProcessor
  def initialize(chunk_size: 4096)
    @chunk_size = chunk_size
    @transformers = []
    @filters = []
  end

  def transform(&block)
    @transformers << block
    self
  end

  def filter(&block)
    @filters << block
    self
  end

  def process(input_file, output_file)
    line_count = 0
    written_count = 0

    File.open(output_file, "w") do |out|
      File.foreach(input_file, chomp: true) do |line|
        line_count += 1
        processed = apply_transformers(line)
        next unless apply_filters(processed)
        out.puts processed
        written_count += 1
      end
    end

    puts "Processed #{line_count} lines, wrote #{written_count}"
    written_count
  end

  private

  def apply_transformers(line)
    @transformers.reduce(line) { |l, t| t.call(l) }
  end

  def apply_filters(line)
    @filters.all? { |f| f.call(line) }
  end
end

# สร้างไฟล์ทดสอบ
log_lines = (1..20).map { |i| "#{Time.now.strftime('%H:%M:%S')} INFO Request #{i} - status=#{[200, 404, 500].sample} path=/api/item/#{i}" }
File.write("/tmp/stream_input.log", log_lines.join("\n") + "\n")

processor = StreamProcessor.new
processor
  .transform { |line| line.gsub(/\d{2}:\d{2}:\d{2}/, "[TIME]") }
  .filter { |line| line.include?("status=200") }

processor.process("/tmp/stream_input.log", "/tmp/stream_output.log")

puts "\nOutput (200 OK only):"
File.foreach("/tmp/stream_output.log") { |line| puts "  #{line.chomp}" }
```

```ruby
# ข้อที่ 18-20: Summary examples

# ข้อที่ 18: Multi-file merge and split
class FileSplitter
  def split(filepath, lines_per_chunk)
    basename = File.basename(filepath, ".*")
    ext = File.extname(filepath)
    dir = File.dirname(filepath)
    chunks = []
    current_chunk = []
    chunk_num = 1

    File.foreach(filepath, chomp: true) do |line|
      current_chunk << line
      if current_chunk.length >= lines_per_chunk
        chunk_file = File.join(dir, "#{basename}_#{chunk_num.to_s.rjust(3, '0')}#{ext}")
        File.write(chunk_file, current_chunk.join("\n") + "\n")
        chunks << chunk_file
        current_chunk = []
        chunk_num += 1
      end
    end

    unless current_chunk.empty?
      chunk_file = File.join(dir, "#{basename}_#{chunk_num.to_s.rjust(3, '0')}#{ext}")
      File.write(chunk_file, current_chunk.join("\n") + "\n")
      chunks << chunk_file
    end

    puts "Split into #{chunks.length} chunks"
    chunks
  end

  def merge(chunk_files, output_file)
    File.open(output_file, "w") do |out|
      chunk_files.sort.each do |chunk|
        out.write(File.read(chunk))
      end
    end
    puts "Merged #{chunk_files.length} chunks into #{output_file}"
  end
end

big_file = "/tmp/big_file.txt"
File.write(big_file, (1..100).map { |i| "Record #{i}: #{("a".."z").to_a.sample(10).join}" }.join("\n"))

splitter = FileSplitter.new
chunks = splitter.split(big_file, 25)
puts "Created chunks: #{chunks.map { |c| File.basename(c) }.inspect}"

splitter.merge(chunks, "/tmp/merged_file.txt")
original_lines = File.readlines(big_file).length
merged_lines = File.readlines("/tmp/merged_file.txt").length
puts "Original: #{original_lines} lines, Merged: #{merged_lines} lines"
puts "Files match: #{original_lines == merged_lines}"

chunks.each { |c| File.delete(c) }
File.delete("/tmp/merged_file.txt")
```

---

## สรุปตอนที่ 15

ในตอนนี้เราได้เรียนรู้:

1. **File.read / File.write** - วิธีง่ายที่สุด
2. **File.open with block** - จัดการ lifecycle อัตโนมัติ
3. **File modes** - r, w, a, r+, w+, a+, b และ encoding
4. **Reading line by line** - each_line, readlines, gets, foreach, lazy
5. **Writing** - puts, write, print, printf
6. **File metadata** - size, mtime, dirname, basename, extname
7. **Existence checks** - exist?, file?, directory?, readable?, writable?
8. **Directory operations** - mkdir, glob, entries, foreach, chdir
9. **CSV** - reading/writing, headers, converters, custom sep
10. **JSON** - generate/parse, pretty, symbolize_names, objects
11. **YAML** - dump/safe_load, config files, custom objects
12. **Pathname** - OOP path manipulation
13. **Tempfile** - temporary files, auto-cleanup
14. **StringIO** - in-memory IO, capture output, testing

---

*ตอนต่อไป: Part 16 - Regular Expressions*
