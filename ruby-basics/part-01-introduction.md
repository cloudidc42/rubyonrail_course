# ส่วนที่ 1: รู้จักกับ Ruby - ขั้นตอนที่ 1-10

## บทนำ

ยินดีต้อนรับสู่คอร์ส Ruby Programming! ในส่วนนี้เราจะเริ่มต้นจากพื้นฐานที่สุด เพื่อให้คุณเข้าใจภาษา Ruby อย่างถ่องแท้ก่อนที่จะก้าวไปสู่ Ruby on Rails

---

## ขั้นตอนที่ 1: Ruby คืออะไร? ประวัติและความเป็นมา

### กำเนิดของ Ruby

Ruby เป็นภาษาโปรแกรมที่สร้างโดย **Yukihiro "Matz" Matsumoto** (มัตสึโมโตะ ยูกิฮิโระ) โปรแกรมเมอร์ชาวญี่ปุ่น เขาเริ่มพัฒนา Ruby ในปี **1993** และเผยแพร่เวอร์ชันแรก (Ruby 0.95) ในปี **1995**

Matz ต้องการสร้างภาษาที่:
- **สนุกในการเขียน** - เขาบอกว่า "Ruby ถูกออกแบบมาเพื่อทำให้โปรแกรมเมอร์มีความสุข"
- **อ่านง่าย** - โค้ดต้องอ่านเหมือนภาษาอังกฤษธรรมชาติ
- **ทรงพลัง** - ต้องทำงานได้หลากหลาย
- **ยืดหยุ่น** - โปรแกรมเมอร์ควรมีอิสระในการแก้ปัญหา

### ปรัชญาของ Ruby: "Principle of Least Surprise"

Matz ออกแบบ Ruby ตามหลักการ **"หลักการแห่งความไม่แปลกใจ"** (Principle of Least Surprise) ซึ่งหมายความว่าภาษาควรทำงานตามที่โปรแกรมเมอร์คาดหวัง ไม่ใช่ตามที่คอมพิวเตอร์สะดวก

```ruby
# ตัวอย่างที่แสดงถึงความเป็นธรรมชาติของ Ruby

# นับจาก 1 ถึง 5
1.upto(5) do |number|
  puts number
end

# ทำซ้ำ 3 ครั้ง
3.times do
  puts "สวัสดี Ruby!"
end

# เช็คว่าเลขคู่หรือไม่
puts 10.even?   # => true
puts 7.odd?     # => true
puts -5.abs     # => 5

# สตริงที่อ่านง่าย
sentence = "Ruby is awesome"
puts sentence.upcase      # => RUBY IS AWESOME
puts sentence.reverse     # => emosewa si ybuR
puts sentence.length      # => 15
```

### เวอร์ชันสำคัญของ Ruby

| เวอร์ชัน | ปี    | สิ่งสำคัญ                              |
|---------|-------|---------------------------------------|
| 1.0     | 1996  | เวอร์ชันแรกอย่างเป็นทางการ              |
| 1.8     | 2003  | เป็นที่นิยมมาก, Rails ใช้เวอร์ชันนี้ตอนแรก |
| 1.9     | 2007  | ปรับปรุงประสิทธิภาพครั้งใหญ่             |
| 2.0     | 2013  | Keyword arguments, lazy enumerators    |
| 2.5     | 2017  | rescue/else/ensure ใน blocks           |
| 2.7     | 2019  | Pattern matching (experimental)        |
| 3.0     | 2020  | 3x faster, Ractors, Type signatures   |
| 3.1     | 2021  | YJIT compiler, Hash shorthand         |
| 3.2     | 2022  | YJIT enabled by default               |
| 3.3     | 2023  | RJIT, M:N threading                   |

---

## ขั้นตอนที่ 2: ทำไมถึงเลือก Ruby?

### เปรียบเทียบ Ruby กับภาษาอื่น

#### Ruby vs Python

```python
# Python - Hello World
print("Hello, World!")

# Python - List comprehension
squares = [x**2 for x in range(10)]
```

```ruby
# Ruby - Hello World
puts "Hello, World!"

# Ruby - ทำแบบเดียวกัน
squares = (0...10).map { |x| x**2 }
# หรือแบบนี้ก็ได้
squares = Array.new(10) { |i| i**2 }
```

**ความแตกต่างหลัก:**
- Ruby เน้นความสวยงามและ "เวทมนตร์" มากกว่า
- Python เน้นความชัดเจนและ "มีทางเดียวที่ถูก"
- Ruby มี Rails ที่แข็งแกร่งสำหรับ Web Development
- Python มีระบบนิเวศ Data Science/AI ที่ใหญ่กว่า

#### Ruby vs JavaScript

```javascript
// JavaScript - Array iteration
[1, 2, 3].forEach(num => console.log(num * 2));

// JavaScript - Object
const person = { name: "Alice", age: 30 };
```

```ruby
# Ruby - Array iteration
[1, 2, 3].each { |num| puts num * 2 }

# Ruby - Hash (คล้าย Object)
person = { name: "Alice", age: 30 }
puts person[:name]  # => Alice
```

**ความแตกต่างหลัก:**
- Ruby รันบน Server, JavaScript รันได้ทั้ง Server และ Browser
- Ruby มี syntax ที่สะอาดกว่า
- JavaScript มี ecosystem ที่ใหญ่กว่ามาก (npm)

#### Ruby vs Java/C#

```java
// Java - Hello World (verbose มาก)
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

```ruby
# Ruby - Hello World (simple มาก)
puts "Hello, World!"
```

**ความแตกต่างหลัก:**
- Ruby เขียนโค้ดน้อยกว่ามาก (ประมาณ 5-10 เท่า)
- Java/C# เร็วกว่าและเหมาะกับ Enterprise มากกว่า
- Ruby เหมาะกับ Startup และ Rapid Prototyping

### จุดเด่นของ Ruby

1. **Expressive** - โค้ดอ่านเหมือนภาษาอังกฤษ
2. **Object-Oriented** - ทุกอย่างเป็น Object จริงๆ (แม้แต่ตัวเลข)
3. **Dynamic Typing** - ไม่ต้องประกาศชนิดตัวแปร
4. **Metaprogramming** - เขียนโค้ดที่สร้างโค้ดได้
5. **Blocks & Closures** - ทำ Functional Programming ได้สวยงาม
6. **Gems** - ไลบรารีที่อุดมสมบูรณ์
7. **Rails** - Framework ที่ยอดเยี่ยมสำหรับ Web Development
8. **Testing Culture** - ชุมชน Ruby รักการทดสอบ

```ruby
# ตัวอย่างความสวยงามของ Ruby

# อ่านเหมือนภาษาอังกฤษ
5.times { print "Ruby! " }
# Output: Ruby! Ruby! Ruby! Ruby! Ruby!

# Conditional ที่อ่านง่าย
age = 20
puts "ผู้ใหญ่แล้ว" if age >= 18
puts "เด็กอยู่" unless age >= 18

# Method chaining
"hello world"
  .split(" ")
  .map(&:capitalize)
  .join(" ")
# => "Hello World"

# Range
(1..10).select(&:odd?).sum  # => 25
```

---

## ขั้นตอนที่ 3: การติดตั้ง Ruby

### วิธีที่ 1: rbenv (แนะนำสำหรับ Mac/Linux)

rbenv เป็น Version Manager ที่เบาและจัดการได้ดี

#### ติดตั้งบน macOS

```bash
# ขั้นตอนที่ 1: ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ขั้นตอนที่ 2: ติดตั้ง rbenv
brew install rbenv ruby-build

# ขั้นตอนที่ 3: เพิ่ม rbenv ลงใน shell configuration
echo 'eval "$(rbenv init - bash)"' >> ~/.bashrc
# หรือถ้าใช้ zsh
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc

# ขั้นตอนที่ 4: Restart terminal หรือ reload config
source ~/.zshrc

# ขั้นตอนที่ 5: ติดตั้ง Ruby
rbenv install 3.3.0

# ขั้นตอนที่ 6: ตั้งค่าเป็น Ruby เริ่มต้น
rbenv global 3.3.0

# ตรวจสอบ
ruby --version
# => ruby 3.3.0 (2023-12-25 revision 5124f9ac75) [arm64-darwin23]
```

#### ติดตั้งบน Linux (Ubuntu/Debian)

```bash
# ขั้นตอนที่ 1: ติดตั้ง dependencies
sudo apt update
sudo apt install -y git curl libssl-dev libreadline-dev zlib1g-dev \
  autoconf bison build-essential libyaml-dev libreadline-dev \
  libncurses5-dev libffi-dev libgdbm-dev

# ขั้นตอนที่ 2: ติดตั้ง rbenv
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash

# ขั้นตอนที่ 3: เพิ่มลงใน PATH
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init - bash)"' >> ~/.bashrc
source ~/.bashrc

# ขั้นตอนที่ 4: ติดตั้ง Ruby
rbenv install 3.3.0
rbenv global 3.3.0

# ตรวจสอบ
ruby --version
gem --version
```

### วิธีที่ 2: RVM (Ruby Version Manager)

RVM เป็น Version Manager ที่เก่าแก่และ feature-rich กว่า

```bash
# ติดตั้ง RVM
\curl -sSL https://get.rvm.io | bash -s stable

# Reload terminal
source ~/.rvm/scripts/rvm

# ติดตั้ง Ruby
rvm install 3.3.0

# ใช้งาน
rvm use 3.3.0 --default

# ตรวจสอบ
ruby --version

# ดูเวอร์ชันที่ติดตั้งไว้
rvm list

# สร้าง Gemset สำหรับแต่ละโปรเจกต์
rvm gemset create myapp
rvm use 3.3.0@myapp
```

### วิธีที่ 3: asdf (Universal Version Manager)

asdf สามารถจัดการหลายภาษาได้ในที่เดียว

```bash
# ติดตั้ง asdf
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0

# เพิ่มลงใน shell config
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
echo '. "$HOME/.asdf/completions/asdf.bash"' >> ~/.bashrc
source ~/.bashrc

# เพิ่ม Ruby plugin
asdf plugin add ruby https://github.com/asdf-vm/asdf-ruby.git

# ติดตั้ง Ruby
asdf install ruby 3.3.0

# ตั้งค่า global version
asdf global ruby 3.3.0

# ตั้งค่า local version (สำหรับโปรเจกต์เฉพาะ)
cd myproject
asdf local ruby 3.3.0
# สร้างไฟล์ .tool-versions

# ตรวจสอบ
ruby --version
```

### วิธีที่ 4: ติดตั้งบน Windows

#### วิธีที่ดีที่สุดบน Windows: WSL2

```bash
# ขั้นตอนที่ 1: เปิด PowerShell ในฐานะ Administrator
wsl --install

# ขั้นตอนที่ 2: Restart คอมพิวเตอร์

# ขั้นตอนที่ 3: เปิด Ubuntu จาก Start Menu
# แล้วทำตามขั้นตอน Linux ด้านบน
```

#### ใช้ RubyInstaller (Windows native)

```
1. ไปที่ https://rubyinstaller.org/downloads/
2. ดาวน์โหลด Ruby+Devkit 3.3.X (x64)
3. รันไฟล์ติดตั้ง
4. เลือก "Add Ruby executables to your PATH"
5. เปิด Command Prompt และตรวจสอบ: ruby --version
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Ruby
ruby --version
# ruby 3.3.0 (2023-12-25 revision 5124f9ac75)

# ตรวจสอบ RubyGems
gem --version
# 3.5.3

# ตรวจสอบ Bundler
bundler --version
# Bundler version 2.5.3

# รันโค้ด Ruby แบบ inline
ruby -e "puts 'Hello from Ruby!'"
# Hello from Ruby!

# ดูที่อยู่ Ruby
which ruby
# /home/username/.rbenv/shims/ruby
```

---

## ขั้นตอนที่ 4: Ruby Interactive Shell (IRB และ Pry)

### IRB (Interactive Ruby Shell)

IRB คือ shell สำหรับทดลองเขียนโค้ด Ruby แบบ interactive ติดมากับ Ruby ทุกเวอร์ชัน

```bash
# เริ่มต้น IRB
irb
```

```ruby
# ใน IRB ลองพิมพ์คำสั่งเหล่านี้
irb(main):001> 1 + 2
=> 3

irb(main):002> "Hello" + " " + "World"
=> "Hello World"

irb(main):003> [1, 2, 3].sum
=> 6

irb(main):004> name = "Ruby"
=> "Ruby"

irb(main):005> puts "Hello, #{name}!"
Hello, Ruby!
=> nil

irb(main):006> 10.times.map { |i| i * 2 }
=> [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

# ออกจาก IRB
irb(main):007> exit
# หรือกด Ctrl+D
```

### การตั้งค่า IRB

สร้างไฟล์ `~/.irbrc` เพื่อปรับแต่ง IRB:

```ruby
# ~/.irbrc
require 'irb/completion'      # Auto-complete
require 'irb/ext/save-history'

IRB.conf[:SAVE_HISTORY] = 1000
IRB.conf[:HISTORY_FILE] = "#{ENV['HOME']}/.irb_history"
IRB.conf[:AUTO_INDENT] = true
IRB.conf[:PROMPT_MODE] = :DEFAULT

# เพิ่ม method ที่ใช้บ่อย
class Object
  def ri(method = nil)
    unless method && method.to_s.size > 0
      puts `ri #{self.class}`
      return
    end
    puts `ri #{self.class}##{method}`
  end
end
```

### Pry - IRB ที่ดีกว่า

Pry เป็น alternative ที่ทรงพลังกว่า IRB มาก

```bash
# ติดตั้ง Pry
gem install pry pry-byebug

# เริ่มต้น Pry
pry
```

```ruby
# Pry มี features พิเศษมากมาย

# ดู source code ของ method
pry(main)> show-source Array#map

# ดู documentation
pry(main)> ? String#gsub

# ดูค่าตัวแปรในแต่ละ context
pry(main)> ls String

# Navigate ไปใน object
pry(main)> cd [1,2,3]
pry(#<Array:0x>):1> map { |x| x * 2 }
=> [2, 4, 6]
pry(#<Array:0x>):1> exit

# History
pry(main)> hist
```

```ruby
# ใช้ Pry ใน code เพื่อ debug
require 'pry'

def calculate_total(items)
  binding.pry  # หยุดตรงนี้และเข้า Pry shell
  items.sum
end

calculate_total([1, 2, 3, 4, 5])
# โปรแกรมจะหยุดและเปิด Pry shell ให้เราตรวจสอบตัวแปร
```

---

## ขั้นตอนที่ 5: Hello World แบบต่างๆ

### การแสดงผลพื้นฐาน

```ruby
# วิธีที่ 1: puts - พิมพ์แล้วขึ้นบรรทัดใหม่
puts "Hello, World!"
# Hello, World!

# วิธีที่ 2: print - พิมพ์โดยไม่ขึ้นบรรทัดใหม่
print "Hello"
print ", "
print "World!"
print "\n"   # ต้องใส่ newline เอง
# Hello, World!

# วิธีที่ 3: p - พิมพ์พร้อม inspect (ดีสำหรับ debug)
p "Hello, World!"
# "Hello, World!"  (มี quote)
p 42
# 42
p [1, 2, 3]
# [1, 2, 3]

# วิธีที่ 4: pp - pretty print สำหรับ data ซับซ้อน
require 'pp'
pp({ name: "Ruby", version: 3.3, features: ["fast", "elegant"] })
# {:name=>"Ruby", :version=>3.3, :features=>["fast", "elegant"]}

# วิธีที่ 5: printf - ใช้ format string
printf("Hello, %s! You are %d years old.\n", "World", 25)
# Hello, World! You are 25 years old.

# วิธีที่ 6: sprintf - เหมือน printf แต่ return string
message = sprintf("Hello, %s!", "World")
puts message
# Hello, World!
```

### Hello World หลายภาษา

```ruby
# สวัสดีชาวโลกในหลายภาษา
greetings = {
  thai:     "สวัสดีชาวโลก!",
  english:  "Hello, World!",
  japanese: "こんにちは世界！",
  spanish:  "¡Hola, Mundo!",
  french:   "Bonjour, le Monde!",
  arabic:   "مرحبا بالعالم!"
}

greetings.each do |language, greeting|
  puts "#{language.to_s.capitalize}: #{greeting}"
end
```

### Hello World แบบ OOP

```ruby
# Object-Oriented Hello World
class Greeter
  def initialize(name)
    @name = name
  end

  def greet
    puts "Hello, #{@name}!"
  end

  def greet_multiple_times(times)
    times.times { |i| puts "#{i + 1}. Hello, #{@name}!" }
  end
end

greeter = Greeter.new("World")
greeter.greet
# Hello, World!

greeter.greet_multiple_times(3)
# 1. Hello, World!
# 2. Hello, World!
# 3. Hello, World!
```

### Hello World แบบ Functional

```ruby
# Functional style
hello = ->(name) { "Hello, #{name}!" }
puts hello.call("World")
# Hello, World!

# Method reference
greet = method(:puts)
greet.call("Hello, World!")
# Hello, World!

# Composition
shout = ->(text) { text.upcase }
greet_loudly = ->(name) { shout.call("Hello, #{name}!") }
puts greet_loudly.call("World")
# HELLO, WORLD!
```

---

## ขั้นตอนที่ 6: Ruby Version Manager

### ทำความเข้าใจ Version Manager

ทำไมต้องใช้ Version Manager?
- โปรเจกต์ต่างๆ อาจต้องการ Ruby version ต่างกัน
- สามารถสลับ version ได้ง่าย
- ไม่กระทบกับ system Ruby

### คำสั่ง rbenv ที่ใช้บ่อย

```bash
# ดู Ruby versions ที่ติดตั้ง
rbenv versions
# * 3.3.0 (set by /home/user/.rbenv/version)
#   3.2.2
#   3.1.4

# ดู version ปัจจุบัน
rbenv version
# 3.3.0 (set by /home/user/.rbenv/version)

# ดู versions ที่ติดตั้งได้
rbenv install -l

# ติดตั้ง version ใหม่
rbenv install 3.3.1

# ตั้ง global version
rbenv global 3.3.1

# ตั้ง local version (สำหรับโฟลเดอร์นั้น)
cd /path/to/project
rbenv local 3.2.2
# สร้างไฟล์ .ruby-version

# ลบ version
rbenv uninstall 3.1.4

# Rehash หลังติดตั้ง gem ใหม่
rbenv rehash
```

### .ruby-version file

```bash
# ไฟล์ .ruby-version บอกว่าโปรเจกต์ใช้ Ruby version ไหน
cat .ruby-version
# 3.3.0

# rbenv จะอ่านไฟล์นี้อัตโนมัติเมื่อเข้าโฟลเดอร์
```

### คำสั่ง RVM ที่ใช้บ่อย

```bash
# ดู versions ทั้งหมด
rvm list

# ติดตั้ง
rvm install 3.3.0

# ใช้งาน
rvm use 3.3.0

# ใช้งานเป็น default
rvm use 3.3.0 --default

# สร้าง Gemset
rvm gemset create rails_app

# ใช้ Ruby + Gemset
rvm use 3.3.0@rails_app

# ดู Gemsets
rvm gemset list

# ลบ Gemset
rvm gemset delete rails_app
```

---

## ขั้นตอนที่ 7: Gems เบื้องต้น

### Gem คืออะไร?

Gem คือ Package หรือ Library ของ Ruby เหมือน npm ของ Node.js หรือ pip ของ Python

### RubyGems - Package Manager

```bash
# ดู gems ที่ติดตั้งอยู่
gem list

# ค้นหา gem
gem search rails

# ติดตั้ง gem
gem install rails

# ติดตั้ง version เฉพาะ
gem install rails -v 7.1.0

# อัพเดท gem
gem update rails

# อัพเดท gems ทั้งหมด
gem update

# ลบ gem
gem uninstall rails

# ดูข้อมูล gem
gem info rails

# ดู documentation
gem server  # เปิด web server ดู docs
```

### Bundler - จัดการ Dependencies

Bundler ช่วยจัดการ dependencies ของโปรเจกต์

```bash
# ติดตั้ง Bundler
gem install bundler

# สร้าง Gemfile ใหม่
bundle init
```

```ruby
# Gemfile - กำหนด dependencies ของโปรเจกต์
source 'https://rubygems.org'

ruby '3.3.0'

# Web Framework
gem 'rails', '~> 7.1'

# Database
gem 'pg', '>= 1.1'

# Testing
group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
end

group :development do
  gem 'better_errors'
  gem 'pry-rails'
end
```

```bash
# ติดตั้ง gems จาก Gemfile
bundle install

# อัพเดท gems ทั้งหมด
bundle update

# อัพเดท gem เฉพาะ
bundle update rails

# รันโปรแกรมด้วย environment ของ Bundler
bundle exec ruby myapp.rb
bundle exec rails server
bundle exec rspec

# ดู dependency tree
bundle list
```

### Gems ที่น่าสนใจ

```ruby
# 1. Pry - Better REPL
gem install pry

# 2. Colorize - สีใน terminal
gem install colorize

require 'colorize'
puts "Hello".red
puts "World".green.bold
puts "Ruby".colorize(:blue).on_white

# 3. HTTParty - HTTP requests
gem install httparty

require 'httparty'
response = HTTParty.get('https://api.github.com/users/ruby')
puts response['name']

# 4. Nokogiri - HTML/XML parsing
gem install nokogiri

require 'nokogiri'
require 'open-uri'

doc = Nokogiri::HTML(URI.open('https://example.com'))
puts doc.title

# 5. Faker - สร้าง fake data
gem install faker

require 'faker'
puts Faker::Name.full_name      # => "Alice Johnson"
puts Faker::Internet.email      # => "alice@example.com"
puts Faker::Address.city        # => "Bangkok"
```

---

## ขั้นตอนที่ 8: การรันไฟล์ Ruby

### สร้างและรันไฟล์ Ruby

```bash
# สร้างไฟล์
touch hello.rb
```

```ruby
# hello.rb
#!/usr/bin/env ruby
# encoding: utf-8

puts "สวัสดีครับ!"
puts "นี่คือไฟล์ Ruby แรกของฉัน"

# รับ arguments จาก command line
if ARGV.length > 0
  puts "Arguments ที่ได้รับ: #{ARGV.join(', ')}"
else
  puts "ไม่มี arguments"
end
```

```bash
# รันไฟล์
ruby hello.rb
# สวัสดีครับ!
# นี่คือไฟล์ Ruby แรกของฉัน
# ไม่มี arguments

# รันพร้อม arguments
ruby hello.rb foo bar baz
# สวัสดีครับ!
# นี่คือไฟล์ Ruby แรกของฉัน
# Arguments ที่ได้รับ: foo, bar, baz

# ทำให้ไฟล์รันได้โดยตรง (Unix/Linux/Mac)
chmod +x hello.rb
./hello.rb
```

### Command-line Options

```bash
# -e: รัน code โดยตรง
ruby -e "puts 'Hello!'"

# -w: เปิด warning
ruby -w myfile.rb

# -r: require library
ruby -r json -e "puts JSON.parse('{\"key\":\"value\"}').inspect"

# -v: ดู version
ruby -v

# --check: ตรวจสอบ syntax โดยไม่รัน
ruby --check myfile.rb
# หรือย่อ
ruby -c myfile.rb

# -d: debug mode
ruby -d myfile.rb

# STDIN
echo "hello" | ruby -e "puts gets.upcase"
# HELLO
```

### การรับ Input จากผู้ใช้

```ruby
# รับ input พื้นฐาน
print "กรุณาใส่ชื่อของคุณ: "
name = gets.chomp  # chomp ลบ newline ออก
puts "สวัสดี, #{name}!"

# รับตัวเลข
print "ใส่อายุของคุณ: "
age = gets.chomp.to_i  # แปลงเป็น Integer
puts "คุณจะมีอายุ #{age + 10} ปีใน 10 ปีข้างหน้า"

# รับหลาย inputs
print "ใส่ตัวเลข 3 ตัวคั่นด้วย space: "
numbers = gets.chomp.split(' ').map(&:to_i)
puts "ผลรวม: #{numbers.sum}"
puts "ค่าเฉลี่ย: #{numbers.sum.to_f / numbers.length}"

# ใช้ STDIN
STDIN.each_line do |line|
  puts line.upcase
end
```

---

## ขั้นตอนที่ 9: Ruby File Structure

### โครงสร้างไฟล์ทั่วไป

```
my_ruby_project/
├── lib/                    # โค้ดหลัก
│   ├── my_project.rb       # Entry point
│   └── my_project/
│       ├── module_a.rb
│       └── module_b.rb
├── bin/                    # Executable scripts
│   └── my_script
├── spec/                   # Tests (RSpec)
│   ├── spec_helper.rb
│   └── lib/
│       └── my_project_spec.rb
├── test/                   # Tests (Minitest)
├── config/                 # Configuration files
├── data/                   # Data files
├── doc/                    # Documentation
├── Gemfile                 # Dependencies
├── Gemfile.lock            # Locked versions
├── README.md               # Documentation
├── .ruby-version           # Ruby version
└── Rakefile                # Rake tasks
```

### Require และ Require_relative

```ruby
# require - โหลด library จาก $LOAD_PATH
require 'json'
require 'date'
require 'net/http'

# require_relative - โหลด relative to current file
require_relative 'helpers/string_helper'
require_relative '../models/user'

# autoload - โหลดเมื่อมีการใช้งาน
autoload :User, 'models/user'

# load - โหลดทุกครั้ง (ไม่ cache)
load 'config.rb'
```

### การจัดระเบียบโค้ดด้วย Module และ Class

```ruby
# lib/my_project.rb
require 'my_project/version'
require 'my_project/configuration'
require 'my_project/utils'

module MyProject
  class Error < StandardError; end

  def self.configure
    yield configuration
  end

  def self.configuration
    @configuration ||= Configuration.new
  end
end

# lib/my_project/version.rb
module MyProject
  VERSION = "1.0.0"
end

# lib/my_project/configuration.rb
module MyProject
  class Configuration
    attr_accessor :api_key, :timeout, :debug

    def initialize
      @timeout = 30
      @debug = false
    end
  end
end

# lib/my_project/utils.rb
module MyProject
  module Utils
    def self.format_number(num)
      num.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse
    end

    def self.timestamp
      Time.now.strftime("%Y-%m-%d %H:%M:%S")
    end
  end
end
```

### Gemspec - เมื่อสร้าง Gem เอง

```ruby
# my_project.gemspec
Gem::Specification.new do |spec|
  spec.name          = "my_project"
  spec.version       = MyProject::VERSION
  spec.authors       = ["Your Name"]
  spec.email         = ["your@email.com"]

  spec.summary       = "A brief description"
  spec.description   = "A longer description"
  spec.homepage      = "https://github.com/username/my_project"
  spec.license       = "MIT"

  spec.files         = Dir['lib/**/*', 'bin/*', 'README.md']
  spec.executables   = spec.files.grep(%r{^bin/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  spec.ruby_version  = '>= 3.0.0'

  spec.add_dependency "httparty", "~> 0.21"
  spec.add_development_dependency "rspec", "~> 3.12"
end
```

---

## ขั้นตอนที่ 10: แบบฝึกหัด 10 ข้อ

### แบบฝึกหัดที่ 1: Hello World หลายภาษา

เขียนโปรแกรมที่แสดงคำทักทาย "สวัสดี" ใน 5 ภาษา โดยใช้ Hash

```ruby
# เฉลย
greetings = {
  "ไทย"     => "สวัสดีครับ/ค่ะ",
  "English" => "Hello",
  "日本語"   => "こんにちは",
  "español" => "Hola",
  "français" => "Bonjour"
}

puts "=== คำทักทายใน 5 ภาษา ==="
greetings.each_with_index do |(language, greeting), index|
  puts "#{index + 1}. #{language}: #{greeting}"
end
```

### แบบฝึกหัดที่ 2: Calculator อย่างง่าย

เขียน calculator ที่รับ input 2 ตัวเลขและ operation (+, -, *, /)

```ruby
# เฉลย
def calculate(a, operator, b)
  case operator
  when '+'
    a + b
  when '-'
    a - b
  when '*'
    a * b
  when '/'
    raise "หารด้วยศูนย์ไม่ได้!" if b == 0
    a.to_f / b
  else
    raise "Operation ไม่ถูกต้อง: #{operator}"
  end
end

print "ใส่ตัวเลขแรก: "
a = gets.chomp.to_f

print "ใส่ operation (+, -, *, /): "
op = gets.chomp

print "ใส่ตัวเลขที่สอง: "
b = gets.chomp.to_f

begin
  result = calculate(a, op, b)
  puts "#{a} #{op} #{b} = #{result}"
rescue => e
  puts "เกิดข้อผิดพลาด: #{e.message}"
end
```

### แบบฝึกหัดที่ 3: FizzBuzz

พิมพ์เลข 1-100 แต่ถ้าหาร 3 ลงตัวพิมพ์ "Fizz", หาร 5 ลงตัวพิมพ์ "Buzz", หารทั้งสองพิมพ์ "FizzBuzz"

```ruby
# เฉลย - วิธีที่ 1: if/else
(1..100).each do |n|
  if n % 15 == 0
    puts "FizzBuzz"
  elsif n % 3 == 0
    puts "Fizz"
  elsif n % 5 == 0
    puts "Buzz"
  else
    puts n
  end
end

# เฉลย - วิธีที่ 2: Ruby style
(1..100).each do |n|
  result = ""
  result += "Fizz" if n % 3 == 0
  result += "Buzz" if n % 5 == 0
  puts result.empty? ? n : result
end

# เฉลย - วิธีที่ 3: Functional
puts (1..100).map { |n|
  case
  when n % 15 == 0 then "FizzBuzz"
  when n % 3 == 0  then "Fizz"
  when n % 5 == 0  then "Buzz"
  else n
  end
}.join("\n")
```

### แบบฝึกหัดที่ 4: ดาวพิมพ์รูปสามเหลี่ยม

```ruby
# พิมพ์สามเหลี่ยมดาว
def triangle(height)
  (1..height).each do |i|
    puts "*" * i
  end
end

triangle(5)
# *
# **
# ***
# ****
# *****

# สามเหลี่ยมหัวกลับ
def inverted_triangle(height)
  height.downto(1) do |i|
    puts "*" * i
  end
end

# สามเหลี่ยมกึ่งกลาง
def centered_triangle(height)
  (1..height).each do |i|
    spaces = " " * (height - i)
    stars = "*" * (2 * i - 1)
    puts spaces + stars
  end
end

centered_triangle(5)
#     *
#    ***
#   *****
#  *******
# *********
```

### แบบฝึกหัดที่ 5: ตรวจสอบ Palindrome

```ruby
def palindrome?(word)
  cleaned = word.downcase.gsub(/[^a-z0-9ก-๙]/, '')
  cleaned == cleaned.reverse
end

words = ["racecar", "hello", "level", "ruby", "madam", "สวัสดี", "ทศกัณฐ์"]

words.each do |word|
  status = palindrome?(word) ? "เป็น palindrome ✓" : "ไม่ใช่ palindrome ✗"
  puts "\"#{word}\" #{status}"
end
```

### แบบฝึกหัดที่ 6: หาจำนวนเฉพาะ

```ruby
def prime?(n)
  return false if n < 2
  return true if n == 2
  return false if n.even?

  (3..Math.sqrt(n)).step(2).none? { |i| n % i == 0 }
end

# หาจำนวนเฉพาะ 1-100
primes = (2..100).select { |n| prime?(n) }
puts "จำนวนเฉพาะระหว่าง 1-100:"
puts primes.join(", ")
puts "มีทั้งหมด #{primes.count} ตัว"
```

### แบบฝึกหัดที่ 7: เขียน BMI Calculator

```ruby
def calculate_bmi(weight_kg, height_cm)
  height_m = height_cm / 100.0
  bmi = weight_kg / (height_m ** 2)
  bmi.round(2)
end

def bmi_category(bmi)
  case bmi
  when 0...18.5  then "น้ำหนักต่ำกว่าเกณฑ์"
  when 18.5...25 then "น้ำหนักปกติ ✓"
  when 25...30   then "น้ำหนักเกิน"
  else                "อ้วน"
  end
end

print "น้ำหนัก (กก.): "
weight = gets.chomp.to_f

print "ส่วนสูง (ซม.): "
height = gets.chomp.to_f

bmi = calculate_bmi(weight, height)
category = bmi_category(bmi)

puts "\n=== ผล BMI ของคุณ ==="
puts "BMI: #{bmi}"
puts "หมวดหมู่: #{category}"
```

### แบบฝึกหัดที่ 8: Guessing Game

```ruby
def guessing_game
  secret = rand(1..100)
  attempts = 0
  max_attempts = 10

  puts "=== ทายตัวเลข ==="
  puts "ทายตัวเลขระหว่าง 1-100 (มี #{max_attempts} ครั้ง)"

  max_attempts.times do |i|
    print "ครั้งที่ #{i + 1}: "
    guess = gets.chomp.to_i
    attempts += 1

    if guess == secret
      puts "ถูกต้อง! คุณทายถูกใน #{attempts} ครั้ง!"
      return
    elsif guess < secret
      puts "น้อยเกินไป!"
    else
      puts "มากเกินไป!"
    end

    remaining = max_attempts - attempts
    puts "เหลืออีก #{remaining} ครั้ง" if remaining > 0
  end

  puts "หมดแล้ว! ตัวเลขที่ถูกต้องคือ #{secret}"
end

guessing_game
```

### แบบฝึกหัดที่ 9: Table of Multiplication

```ruby
def multiplication_table(n)
  puts "ตารางสูตรคูณ #{n}"
  puts "-" * 20

  (1..12).each do |i|
    printf("%d × %2d = %3d\n", n, i, n * i)
  end
end

print "ต้องการดูสูตรคูณแม่ที่เท่าไหร่? "
n = gets.chomp.to_i

if n.between?(1, 12)
  multiplication_table(n)
else
  puts "กรุณาใส่เลข 1-12"
end
```

### แบบฝึกหัดที่ 10: Temperature Converter

```ruby
module TemperatureConverter
  def self.celsius_to_fahrenheit(c)
    (c * 9.0 / 5) + 32
  end

  def self.fahrenheit_to_celsius(f)
    (f - 32) * 5.0 / 9
  end

  def self.celsius_to_kelvin(c)
    c + 273.15
  end

  def self.kelvin_to_celsius(k)
    k - 273.15
  end
end

puts "=== แปลงหน่วยอุณหภูมิ ==="
puts "1. Celsius → Fahrenheit"
puts "2. Fahrenheit → Celsius"
puts "3. Celsius → Kelvin"
puts "4. Kelvin → Celsius"

print "เลือก (1-4): "
choice = gets.chomp.to_i

print "ใส่ค่าอุณหภูมิ: "
temp = gets.chomp.to_f

result = case choice
         when 1 then "#{temp}°C = #{TemperatureConverter.celsius_to_fahrenheit(temp).round(2)}°F"
         when 2 then "#{temp}°F = #{TemperatureConverter.fahrenheit_to_celsius(temp).round(2)}°C"
         when 3 then "#{temp}°C = #{TemperatureConverter.celsius_to_kelvin(temp).round(2)}K"
         when 4 then "#{temp}K = #{TemperatureConverter.kelvin_to_celsius(temp).round(2)}°C"
         else "ตัวเลือกไม่ถูกต้อง"
         end

puts result
```

---

## สรุปส่วนที่ 1

ในส่วนนี้เราได้เรียนรู้:

1. **ประวัติ Ruby** - สร้างโดย Matz ในปี 1993 ด้วยปรัชญา "ทำให้โปรแกรมเมอร์มีความสุข"
2. **จุดเด่น** - Expressive, OOP, Dynamic, Metaprogramming
3. **การติดตั้ง** - rbenv, RVM, asdf บน Mac/Linux/Windows
4. **IRB และ Pry** - เครื่องมือสำหรับทดลองโค้ด
5. **Hello World** - หลายรูปแบบ: puts, print, p, pp
6. **Version Manager** - จัดการ Ruby หลาย version
7. **Gems** - Package manager และ Bundler
8. **รันไฟล์** - ruby command, command-line options
9. **File Structure** - การจัดระเบียบโค้ด
10. **แบบฝึกหัด** - 10 โปรแกรมพื้นฐาน

### ขั้นตอนต่อไป

ในส่วนที่ 2 เราจะเรียนรู้เรื่อง **Variables และ Data Types** ซึ่งเป็นรากฐานสำคัญของ Ruby programming

---

*เอกสารนี้เป็นส่วนหนึ่งของคอร์ส Ruby on Rails สำหรับผู้เริ่มต้น*
