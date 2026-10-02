# ตอนที่ 24: Gems และ Bundler (ขั้นตอนที่ 521-540)

RubyGems เป็นระบบจัดการ Package สำหรับ Ruby ที่ทำให้การแชร์และใช้ Library (เรียกว่า Gem) เป็นเรื่องง่าย Bundler เป็นเครื่องมือที่ช่วยจัดการ Dependencies ของโปรเจกต์

---

## ขั้นตอนที่ 521: Gems คืออะไร?

### นิยามของ Gem

Gem คือ Package หรือ Library ของ Ruby ที่:
- มีโค้ดที่สามารถ Reuse ได้
- มี Metadata (ชื่อ, เวอร์ชัน, Dependencies)
- ถูกเผยแพร่ใน RubyGems.org
- ติดตั้งและใช้งานได้ง่าย

### โครงสร้าง Gem

```
my_gem/
├── lib/
│   ├── my_gem.rb          # Entry point หลัก
│   └── my_gem/
│       ├── version.rb     # เวอร์ชัน Constant
│       ├── core.rb
│       └── helpers.rb
├── spec/
│   ├── spec_helper.rb
│   └── my_gem_spec.rb
├── Gemfile
├── Gemfile.lock
├── my_gem.gemspec         # Metadata ของ Gem
├── README.md
├── LICENSE.txt
└── CHANGELOG.md
```

### ทำไม Gem ถึงสำคัญ?

```ruby
# แทนที่จะเขียน HTTP Client เอง:
require 'net/http'
uri = URI('https://api.example.com/data')
http = Net::HTTP.new(uri.host, uri.port)
http.use_ssl = true
request = Net::HTTP::Get.new(uri)
response = http.request(request)
data = JSON.parse(response.body)

# ใช้ Gem 'httparty' ได้เลย:
require 'httparty'
data = HTTParty.get('https://api.example.com/data').parsed_response
```

---

## ขั้นตอนที่ 522: gem install, list, update

### การใช้งาน gem Command

```bash
# ติดตั้ง Gem
gem install rails
gem install pry --version '0.14.2'
gem install rails --version '>= 7.0'

# ดู Gem ที่ติดตั้งแล้ว
gem list
gem list --local
gem list rails  # ค้นหา Gem ที่มีชื่อ rails

# ดู Gem ที่มีใน Remote
gem list --remote rails

# ข้อมูลรายละเอียด Gem
gem info pry
gem specification pry

# อัพเดท Gem
gem update rails
gem update --system  # อัพเดท RubyGems เอง

# ลบ Gem
gem uninstall pry
gem uninstall pry --version '0.14.0'  # ลบเวอร์ชันเฉพาะ

# ค้นหา Gem
gem search "json"
gem search "csv" --remote

# Cleanup (ลบเวอร์ชันเก่า)
gem cleanup
gem cleanup pry
```

### gem Environment

```bash
# ดู Environment ของ RubyGems
gem environment

# ตัวอย่าง Output:
# RubyGems Environment:
#   - RUBYGEMS VERSION: 3.4.10
#   - RUBY VERSION: 3.2.2
#   - INSTALLATION DIRECTORY: /usr/local/lib/ruby/gems/3.2.0
#   - USER INSTALLATION DIRECTORY: /home/user/.gem/ruby/3.2.0
#   - GEM PATHS:
#      - /usr/local/lib/ruby/gems/3.2.0
#      - /home/user/.gem/ruby/3.2.0
```

---

## ขั้นตอนที่ 523: Gemfile และ Gemfile.lock

### Gemfile พื้นฐาน

```ruby
# Gemfile
source 'https://rubygems.org'

# กำหนดเวอร์ชัน Ruby
ruby '3.2.2'

# Gem ทั่วไป (ทุก Environment)
gem 'rails', '~> 7.1.0'
gem 'pg',    '>= 1.1'
gem 'puma',  '>= 5.0'

# Asset Pipeline
gem 'importmap-rails'
gem 'turbo-rails'
gem 'stimulus-rails'
gem 'tailwindcss-rails'

group :development do
  gem 'web-console'
  gem 'listen'
  gem 'spring'
  gem 'rubocop', require: false
  gem 'rubocop-rails', require: false
end

group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
  gem 'webdrivers'
end

group :development, :test do
  gem 'rspec-rails', '~> 6.0'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'pry-rails'
  gem 'dotenv-rails'
end

group :production do
  gem 'aws-sdk-s3', require: false
  gem 'cloudflare-rails'
end
```

### Version Specifiers

```ruby
# Exact Version
gem 'rails', '7.1.0'

# Greater Than or Equal
gem 'pg', '>= 1.1'

# Pessimistic Operator (~>) - Version Constraint ที่นิยมใช้
gem 'rails', '~> 7.1'    # >= 7.1.0, < 8.0 (เปลี่ยนได้แค่ Minor)
gem 'pry',   '~> 0.14.2' # >= 0.14.2, < 0.15 (เปลี่ยนได้แค่ Patch)

# Range
gem 'rake', '>= 12.0', '< 14.0'

# ไม่ระบุเวอร์ชัน (ใช้ Latest)
gem 'json'
```

### Gemfile.lock

```
GEM
  remote: https://rubygems.org/
  specs:
    actioncable (7.1.0)
      actionpack (= 7.1.0)
      activesupport (= 7.1.0)
      nio4r (~> 2.0)
      websocket-driver (>= 0.6.1)
    rails (7.1.0)
      actioncable (= 7.1.0)
      actionmailer (= 7.1.0)
      actionpack (= 7.1.0)
      ...

PLATFORMS
  x86_64-linux

DEPENDENCIES
  rails (~> 7.1.0)
  puma (>= 5.0)
```

Gemfile.lock ควร Commit ไปยัง Git เสมอ เพื่อให้ทุกคนในทีมใช้ Gem เวอร์ชันเดียวกัน

---

## ขั้นตอนที่ 524: Bundler Commands

### คำสั่ง Bundler พื้นฐาน

```bash
# ติดตั้ง Bundler
gem install bundler

# ติดตั้ง Gems จาก Gemfile
bundle install
bundle install --without production  # ข้าม group production
bundle install --path vendor/bundle  # ติดตั้งใน local directory

# อัพเดท Gems
bundle update              # อัพเดท Gem ทั้งหมด
bundle update rails        # อัพเดทเฉพาะ rails
bundle update rails pry    # อัพเดทหลาย Gem

# รันคำสั่งใน Bundler Environment
bundle exec rspec
bundle exec rake db:migrate
bundle exec rails server

# ดู Dependencies
bundle list
bundle show rails           # แสดง path ของ rails gem
bundle show --paths         # แสดง paths ทั้งหมด

# ตรวจสอบ
bundle check               # ตรวจสอบว่า Gems ทั้งหมดติดตั้งครบ
bundle outdated            # แสดง Gems ที่มีเวอร์ชันใหม่

# สร้าง Gemfile ใหม่
bundle init

# Cleanup
bundle clean               # ลบ Gems ที่ไม่ได้ใช้ออกจาก cache
bundle clean --force
```

### bundle exec

```bash
# ใช้ bundle exec เสมอเพื่อใช้ Version ที่ถูกต้อง
bundle exec rspec spec/
bundle exec rails server
bundle exec rake test
bundle exec rubocop

# สร้าง Binstubs (shortcut)
bundle binstubs rspec-core
./bin/rspec spec/  # ไม่ต้องพิมพ์ bundle exec แล้ว!

# สร้าง Binstubs ทั้งหมด
bundle binstubs --all
```

### Bundler Configuration

```bash
# ตั้งค่า Global
bundle config set --global path 'vendor/bundle'
bundle config set --global without 'development test'

# ตั้งค่าเฉพาะโปรเจกต์ (.bundle/config)
bundle config set --local path 'vendor/bundle'

# ดูการตั้งค่า
bundle config
```

---

## ขั้นตอนที่ 525: Semantic Versioning

### SemVer: MAJOR.MINOR.PATCH

```
Version: 2.4.1
          │ │ └── PATCH: แก้ Bug (Backward Compatible)
          │ └──── MINOR: เพิ่ม Feature (Backward Compatible)
          └────── MAJOR: เปลี่ยน Breaking Changes
```

### ตัวอย่างการเปลี่ยนแปลง

```ruby
# PATCH: 1.0.0 → 1.0.1
# แก้ Bug ที่ไม่กระทบ API

# MINOR: 1.0.1 → 1.1.0
# เพิ่ม Method ใหม่ ยังใช้งาน Old Code ได้

# MAJOR: 1.1.0 → 2.0.0
# เปลี่ยน Method Signature
# ลบ Deprecated Methods
# เปลี่ยนพฤติกรรมหลัก
```

### Pre-release Versions

```
1.0.0-alpha    # Alpha - ยังไม่เสถียร
1.0.0-beta.1   # Beta - ใกล้จะ Release
1.0.0-rc.1     # Release Candidate - เกือบ Final
1.0.0          # Stable Release
```

---

## ขั้นตอนที่ 526-530: สร้าง Gem ของตัวเอง

### สร้าง Gem Structure

```bash
# สร้าง Gem ด้วย bundler
bundle gem my_awesome_gem

# โครงสร้างที่ได้
my_awesome_gem/
├── lib/
│   ├── my_awesome_gem.rb
│   └── my_awesome_gem/
│       └── version.rb
├── spec/
│   ├── spec_helper.rb
│   └── my_awesome_gem_spec.rb
├── .github/
│   └── workflows/
│       └── main.yml
├── .gitignore
├── .rspec
├── Gemfile
├── LICENSE.txt
├── my_awesome_gem.gemspec
├── Rakefile
└── README.md
```

### gemspec File

```ruby
# my_awesome_gem.gemspec
require_relative 'lib/my_awesome_gem/version'

Gem::Specification.new do |spec|
  spec.name          = 'my_awesome_gem'
  spec.version       = MyAwesomeGem::VERSION
  spec.authors       = ['สมชาย ดีใจ']
  spec.email         = ['somchai@example.com']
  
  spec.summary       = 'Gem ที่ยอดเยี่ยมสำหรับทำ...'
  spec.description   = <<~DESC
    คำอธิบายโดยละเอียดของ Gem นี้
    รองรับ Ruby 2.7+
    ใช้สำหรับ...
  DESC
  
  spec.homepage      = 'https://github.com/somchai/my_awesome_gem'
  spec.license       = 'MIT'
  
  # Version Constraints
  spec.required_ruby_version = '>= 2.7.0'
  
  # URLs
  spec.metadata = {
    'homepage_uri'      => spec.homepage,
    'source_code_uri'   => spec.homepage,
    'changelog_uri'     => "#{spec.homepage}/blob/main/CHANGELOG.md",
    'documentation_uri' => "https://rubydoc.info/gems/#{spec.name}",
    'bug_tracker_uri'   => "#{spec.homepage}/issues"
  }
  
  # Files ที่รวมใน Gem
  spec.files = Dir.glob('{lib,exe}/**/*') + 
               ['README.md', 'LICENSE.txt', 'CHANGELOG.md']
  
  spec.bindir        = 'exe'
  spec.executables   = spec.files.grep(%r{\Aexe/}) { |f| File.basename(f) }
  spec.require_paths = ['lib']
  
  # Runtime Dependencies
  spec.add_dependency 'activesupport', '>= 6.0'
  spec.add_dependency 'faraday',       '~> 2.0'
  
  # Development Dependencies
  spec.add_development_dependency 'rspec',     '~> 3.12'
  spec.add_development_dependency 'rubocop',   '~> 1.50'
  spec.add_development_dependency 'simplecov', '~> 0.22'
end
```

### Gem Implementation

```ruby
# lib/my_awesome_gem/version.rb
module MyAwesomeGem
  VERSION = '0.1.0'
end

# lib/my_awesome_gem.rb
require 'my_awesome_gem/version'
require 'my_awesome_gem/thai_text'
require 'my_awesome_gem/formatter'
require 'my_awesome_gem/validator'

module MyAwesomeGem
  class Error < StandardError; end
  class ConfigurationError < Error; end
  
  class << self
    attr_reader :configuration
    
    def configure
      @configuration = Configuration.new
      yield(@configuration)
    end
    
    def reset!
      @configuration = nil
    end
  end
  
  class Configuration
    attr_accessor :api_key, :timeout, :debug
    
    def initialize
      @timeout = 30
      @debug   = false
    end
  end
end

# lib/my_awesome_gem/thai_text.rb
module MyAwesomeGem
  module ThaiText
    def self.to_digits(number)
      THAI_DIGITS = {
        '0' => '๐', '1' => '๑', '2' => '๒', '3' => '๓', '4' => '๔',
        '5' => '๕', '6' => '๖', '7' => '๗', '8' => '๘', '9' => '๙'
      }
      
      number.to_s.chars.map { |c| THAI_DIGITS[c] || c }.join
    end
    
    def self.count_words(text)
      # นับคำในภาษาไทย (แบบง่าย)
      text.split(/[\s,]+/).reject(&:empty?).size
    end
    
    def self.truncate(text, max_length)
      return text if text.length <= max_length
      "#{text[0...max_length]}..."
    end
  end
end
```

### Testing Gem

```ruby
# spec/my_awesome_gem_spec.rb
require 'spec_helper'

RSpec.describe MyAwesomeGem do
  it 'has a version number' do
    expect(MyAwesomeGem::VERSION).not_to be nil
  end
  
  describe '.configure' do
    it 'ตั้งค่า API Key ได้' do
      MyAwesomeGem.configure do |config|
        config.api_key = 'test_key_123'
        config.timeout = 60
      end
      
      expect(MyAwesomeGem.configuration.api_key).to eq('test_key_123')
      expect(MyAwesomeGem.configuration.timeout).to eq(60)
    end
    
    after { MyAwesomeGem.reset! }
  end
end

RSpec.describe MyAwesomeGem::ThaiText do
  describe '.to_digits' do
    it 'แปลงตัวเลขเป็นตัวเลขไทย' do
      expect(MyAwesomeGem::ThaiText.to_digits(12345)).to eq('๑๒๓๔๕')
      expect(MyAwesomeGem::ThaiText.to_digits(0)).to eq('๐')
    end
  end
  
  describe '.truncate' do
    it 'ตัดข้อความที่ยาวเกิน' do
      long_text = "สวัสดีชาวโลกทุกท่าน"
      result    = MyAwesomeGem::ThaiText.truncate(long_text, 5)
      expect(result).to eq("สวัสดี...")
    end
    
    it 'ไม่ตัดข้อความที่ไม่เกิน Limit' do
      short_text = "สวัสดี"
      expect(MyAwesomeGem::ThaiText.truncate(short_text, 10)).to eq("สวัสดี")
    end
  end
end
```

---

## ขั้นตอนที่ 531-535: Publishing Gem ไปยัง RubyGems.org

### ขั้นตอนการ Publish

```bash
# 1. สมัครสมาชิก RubyGems.org
# ไปที่ https://rubygems.org/sign_up

# 2. ตั้งค่า Credentials
gem signin
# ใส่ Email และ Password

# 3. Build Gem
gem build my_awesome_gem.gemspec
# ได้ไฟล์ my_awesome_gem-0.1.0.gem

# 4. ตรวจสอบ Gem ก่อน Publish
gem contents my_awesome_gem-0.1.0.gem

# 5. Publish
gem push my_awesome_gem-0.1.0.gem

# ตรวจสอบว่า Publish แล้ว
gem list my_awesome_gem --remote
```

### การทำ Release ด้วย rake-release

```ruby
# Rakefile
require 'bundler/gem_tasks'
require 'rspec/core/rake_task'
require 'rubocop/rake_task'

RSpec::RakeTask.new(:spec)
RuboCop::RakeTask.new

task default: %i[spec rubocop]

# คำสั่งที่ได้จาก bundler/gem_tasks:
# rake build      - Build Gem
# rake install    - Build และติดตั้ง Local
# rake release    - Build, Tag, Push ไปยัง GitHub และ RubyGems
```

```bash
# Workflow การ Release
# 1. อัพเดทเวอร์ชันใน version.rb
# 2. อัพเดท CHANGELOG.md
# 3. Commit changes
# 4. รัน rake release
rake release

# Output:
# my_awesome_gem 0.1.0 built to pkg/my_awesome_gem-0.1.0.gem
# Tagged v0.1.0
# Pushed git commits and release tag
# Pushed my_awesome_gem 0.1.0 to rubygems.org
```

### Private Gem Server

```ruby
# Gemfile สำหรับ Private Gem Server
source 'https://rubygems.org'

# Private gem server
source 'https://gems.example.com' do
  gem 'our_internal_gem'
  gem 'another_private_gem'
end

gem 'rails', '~> 7.1'
```

---

## ขั้นตอนที่ 536-538: Popular Gems Overview

### Development Tools

```ruby
# pry - Interactive Ruby Shell ที่ทรงพลัง
gem 'pry'

# การใช้งาน
require 'pry'

def complex_method
  data = fetch_data
  binding.pry  # Pause และเปิด pry session
  process_data(data)
end
```

```ruby
# awesome_print - Pretty Print Ruby Objects
gem 'awesome_print'

require 'awesome_print'

# แทนที่ p หรือ pp
ap { name: 'สมชาย', age: 30, roles: [:admin, :user] }
# Output สวยงาม มีสี และ Indentation

# ตั้งค่า Default
AwesomePrint.defaults = {
  indent: 2,
  sort_keys: true,
  color: {
    string: :yellow,
    symbol: :green,
    integer: :cyan
  }
}
```

```ruby
# irb (built-in) vs pry
# pry มีฟีเจอร์เพิ่มเติม:
# - Syntax Highlighting
# - Method introspection (show-method)
# - Debugging (binding.pry)
# - History
# - Plugins

# pry-byebug - Debugging
gem 'pry-byebug'

binding.pry
# Commands:
# next - ไปบรรทัดถัดไป
# step - เข้าไปใน Method
# finish - ออกจาก Method ปัจจุบัน
# continue - ทำงานต่อจนถึง breakpoint ถัดไป
```

### HTTP Clients

```ruby
# httparty - Simple HTTP Client
gem 'httparty'

class WeatherClient
  include HTTParty
  base_uri 'https://api.weatherapi.com/v1'
  
  def initialize(api_key)
    @options = { query: { key: api_key } }
  end
  
  def current(city)
    self.class.get('/current.json', @options.merge(
      query: @options[:query].merge(q: city)
    ))
  end
end

client = WeatherClient.new('your_api_key')
weather = client.current('Bangkok')
puts weather['current']['temp_c']

# faraday - Flexible HTTP Client
gem 'faraday'
gem 'faraday-retry'
gem 'faraday-follow_redirects'

connection = Faraday.new('https://api.example.com') do |conn|
  conn.use Faraday::Retry::Middleware, max: 3
  conn.response :json
  conn.adapter Faraday.default_adapter
end

response = connection.get('/users', { page: 1, per_page: 20 })
puts response.body
```

### JSON/Data Processing

```ruby
# oj - Fast JSON Parser (Optimized JSON)
gem 'oj'

require 'oj'

# Fast JSON
json_string = '{"name":"สมชาย","age":30}'
data = Oj.load(json_string)
puts data  # {"name"=>"สมชาย", "age"=>30}

output = Oj.dump({ name: 'สมชาย', age: 30 })
puts output  # {"name":"สมชาย","age":30}

# dry-types - Type System สำหรับ Ruby
gem 'dry-types'

module Types
  include Dry.Types()
  
  Email    = String.constrained(format: /\A[^@\s]+@[^@\s]+\z/)
  Age      = Integer.constrained(gteq: 0, lteq: 150)
  ThaiName = String.constrained(min_size: 2, max_size: 100)
end

# ใช้งาน
Types::Email.('valid@example.com')   # OK
Types::Email.('invalid')              # Error!
Types::Age.(25)                       # OK
Types::Age.(200)                      # Error!
```

### Database

```ruby
# sequel - Database Toolkit
gem 'sequel'

DB = Sequel.connect('postgres://localhost/mydb')

class User < Sequel::Model
  many_to_many :roles
  one_to_many  :orders
  
  def before_save
    self.email = email.downcase
    super
  end
end

# Query API คล้าย ActiveRecord
User.where(active: true).order(:name).limit(10)
User.where { age > 18 }.all
DB[:users].insert(name: 'สมชาย', email: 'test@example.com')
```

### Utility Gems

```ruby
# activesupport - Standalone ActiveSupport
gem 'activesupport', require: false

require 'active_support/all'

# Timezone
Time.now.in_time_zone('Bangkok')
'สวัสดี'.truncate(3)    # "สวั..."
2.weeks.ago
1.month.from_now
[1, 2, 3].sum
{ a: 1, b: 2 }.slice(:a)  # {a: 1}

# thor - CLI Framework
gem 'thor'

class MyCLI < Thor
  desc "hello NAME", "สวัสดี NAME"
  option :thai, type: :boolean, default: false
  
  def hello(name)
    greeting = options[:thai] ? "สวัสดี" : "Hello"
    puts "#{greeting}, #{name}!"
  end
  
  desc "list", "แสดงรายการ"
  def list
    puts "รายการ..."
  end
end

MyCLI.start(ARGV)
# ruby my_cli.rb hello สมชาย --thai
# ruby my_cli.rb list
```

### Background Jobs

```ruby
# sidekiq - Background Job Processing
gem 'sidekiq'

class SendEmailWorker
  include Sidekiq::Worker
  
  sidekiq_options queue: :mailers, retry: 3
  
  def perform(user_id, template_name)
    user = User.find(user_id)
    UserMailer.send(template_name, user).deliver_now
  end
end

# Enqueue Job
SendEmailWorker.perform_async(user.id, 'welcome_email')
SendEmailWorker.perform_in(5.minutes, user.id, 'reminder_email')
SendEmailWorker.perform_at(Time.now + 1.hour, user.id, 'followup_email')

# good_job - Multi-threaded Active Job Backend
gem 'good_job'

class ImportDataJob < ApplicationJob
  queue_as :default
  
  def perform(file_path)
    ImportService.new(file_path).import!
  end
end

ImportDataJob.perform_later('data.csv')
ImportDataJob.set(wait: 5.minutes).perform_later('data.csv')
```

---

## ขั้นตอนที่ 539: Gem Security

```bash
# ตรวจสอบ Security Vulnerabilities
gem install bundler-audit

# อัพเดท Advisory Database
bundle audit update

# ตรวจสอบ Gemfile.lock
bundle audit check
# Output:
# Name: rack
# Version: 2.2.6
# Advisory: CVE-2022-44570
# Criticality: Medium
# URL: https://github.com/advisories/GHSA-65f5-mfpf-vfhj
# Title: Denial of Service Vulnerability
# Solution: upgrade to >= 2.2.6.3

# ตรวจสอบพร้อมอัพเดท
bundle audit check --update
```

### การเพิ่ม Security ใน CI/CD

```yaml
# .github/workflows/security.yml
name: Security Check

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
      
      - name: Run bundler-audit
        run: |
          gem install bundler-audit
          bundle audit check --update
      
      - name: Run brakeman (Rails Security)
        run: |
          gem install brakeman
          brakeman -q
```

---

## ขั้นตอนที่ 540: Best Practices สำหรับ Gems

### Gemfile Organization

```ruby
# Gemfile ที่จัดระเบียบดี
source 'https://rubygems.org'
ruby File.read('.ruby-version').strip

# Core
gem 'rails',    '~> 7.1.0'
gem 'pg',       '>= 1.1'
gem 'puma',     '>= 5.0'
gem 'redis',    '~> 5.0'
gem 'sidekiq',  '~> 7.0'

# Authentication
gem 'devise',   '~> 4.9'
gem 'jwt',      '~> 2.7'

# API
gem 'jbuilder', '~> 2.11'
gem 'oj',       '~> 3.16'

# Storage
gem 'aws-sdk-s3',      require: false
gem 'image_processing', '~> 1.2'

# Monitoring
gem 'sentry-ruby',     '~> 5.11'
gem 'sentry-rails'
gem 'rack-mini-profiler', require: false

group :development do
  gem 'web-console'
  gem 'letter_opener'         # Preview emails ใน browser
  gem 'bullet'                # Detect N+1 Queries
  gem 'annotate'              # เพิ่ม Schema ใน Model files
  gem 'brakeman', require: false
  gem 'rubocop-rails', require: false
end

group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
  gem 'webdrivers'
  gem 'vcr'                   # Record HTTP Interactions
  gem 'webmock'               # Stub HTTP Requests
  gem 'timecop'               # Control Time ใน Tests
end

group :development, :test do
  gem 'rspec-rails',          '~> 6.0'
  gem 'factory_bot_rails',    '~> 6.2'
  gem 'faker',                '~> 3.2'
  gem 'pry-rails'
  gem 'pry-byebug'
  gem 'simplecov', require: false
  gem 'dotenv-rails'
end
```

---

## แบบฝึกหัดบทที่ 24 (20 ข้อ)

**ข้อ 1:** ติดตั้ง `pry` และ `awesome_print` gems จากนั้นสำรวจฟีเจอร์ต่างๆ ของแต่ละ Gem

**ข้อ 2:** สร้าง Gemfile สำหรับโปรเจกต์ Web Scraper ที่ต้องการ `nokogiri`, `httparty`, `csv` พร้อม Development/Test groups

**ข้อ 3:** ใช้ `bundle outdated` ตรวจสอบ Gems ที่ล้าสมัย และอธิบาย SemVer ของแต่ละ Gem

**ข้อ 4:** สร้าง Gem skeleton ด้วย `bundle gem` และ Implement `ThaiNumberFormatter` ที่แปลงตัวเลขเป็นภาษาไทย

**ข้อ 5:** เขียน gemspec ที่สมบูรณ์พร้อม Dependencies สำหรับ Gem ที่สร้างใน ข้อ 4

**ข้อ 6:** ทดสอบ Gem ที่สร้างด้วย RSpec และเพิ่ม SimpleCov สำหรับ Coverage

**ข้อ 7:** ใช้ `bundle audit` ตรวจสอบ Security Vulnerabilities ในโปรเจกต์ที่มี

**ข้อ 8:** สร้าง CLI Tool ด้วย `thor` gem ที่มีคำสั่ง `greet`, `list`, `create`

**ข้อ 9:** สำรวจ `activesupport` gem โดยใช้ Time extensions, String extensions, Array extensions

**ข้อ 10:** สร้าง HTTP Client Class โดยใช้ `faraday` พร้อม Retry และ Error Handling

**ข้อ 11:** อธิบายความแตกต่างระหว่าง `gem install` และ `bundle install`

**ข้อ 12:** สร้าง Rake task สำหรับ Build และ Test Gem ที่สร้าง

**ข้อ 13:** ใช้ `dry-types` สร้าง Type-safe Data Structures สำหรับ User และ Order

**ข้อ 14:** สำรวจ `oj` gem และเปรียบเทียบ Performance กับ `json` built-in ด้วย Benchmark

**ข้อ 15:** สร้าง Background Worker ด้วย `sidekiq` สำหรับส่ง Email และ Process Image

**ข้อ 16:** เขียน Changelog สำหรับ Gem ที่สร้าง โดยใช้ Keep a Changelog format

**ข้อ 17:** ตั้งค่า GitHub Actions สำหรับ Test Gem อัตโนมัติ

**ข้อ 18:** สำรวจ `timecop` gem และใช้ใน Tests ที่มี Time-dependent Logic

**ข้อ 19:** สร้าง Private Gem ที่มี Configuration Options และ Error Handling ที่สมบูรณ์

**ข้อ 20:** ทำ Complete Workflow: สร้าง Gem → Test → Build → Mock Publish (ไม่ต้อง Publish จริง)

---

### เฉลยตัวอย่าง ข้อ 4: ThaiNumberFormatter Gem

```ruby
# lib/thai_number_formatter.rb
require 'thai_number_formatter/version'
require 'thai_number_formatter/thai_number'

module ThaiNumberFormatter
  class Error < StandardError; end
  
  def self.format(number, options = {})
    ThaiNumber.new(number, options).format
  end
  
  def self.to_text(number)
    ThaiNumber.new(number).to_text
  end
end

# lib/thai_number_formatter/thai_number.rb
module ThaiNumberFormatter
  class ThaiNumber
    THAI_DIGITS = %w[๐ ๑ ๒ ๓ ๔ ๕ ๖ ๗ ๘ ๙].freeze
    
    ONES = %w[ศูนย์ หนึ่ง สอง สาม สี่ ห้า หก เจ็ด แปด เก้า].freeze
    POSITIONS = %w['' สิบ ร้อย พัน หมื่น แสน ล้าน].freeze
    
    def initialize(number, options = {})
      @number  = number
      @options = options
    end
    
    def format
      digits_string.chars.map { |c| THAI_DIGITS[c.to_i] }.join
    end
    
    def to_text
      return 'ศูนย์' if @number.zero?
      
      parts = []
      n = @number.abs
      
      millions = n / 1_000_000
      parts << "#{convert_below_million(millions)}ล้าน" if millions > 0
      
      remainder = n % 1_000_000
      parts << convert_below_million(remainder) if remainder > 0
      
      result = parts.join('')
      @number < 0 ? "ลบ#{result}" : result
    end
    
    private
    
    def digits_string
      @number.to_s
    end
    
    def convert_below_million(n)
      return '' if n == 0
      
      result = ''
      position = 0
      
      while n > 0
        digit = n % 10
        
        if digit > 0
          digit_name = digit == 1 && position == 1 ? 'เอ็ด' : ONES[digit]
          digit_name = 'ยี่' if digit == 2 && position == 1
          
          result = "#{digit_name}#{POSITIONS[position]}#{result}"
        end
        
        n /= 10
        position += 1
      end
      
      result
    end
  end
end

# spec/thai_number_formatter_spec.rb
RSpec.describe ThaiNumberFormatter do
  describe '.format' do
    it 'แปลงเลข 0-9 เป็นเลขไทย' do
      expect(ThaiNumberFormatter.format(0)).to eq('๐')
      expect(ThaiNumberFormatter.format(5)).to eq('๕')
      expect(ThaiNumberFormatter.format(9)).to eq('๙')
    end
    
    it 'แปลงเลขหลายหลัก' do
      expect(ThaiNumberFormatter.format(2567)).to eq('๒๕๖๗')
      expect(ThaiNumberFormatter.format(12345)).to eq('๑๒๓๔๕')
    end
  end
  
  describe '.to_text' do
    it 'แปลงตัวเลขเป็นตัวอักษรไทย' do
      expect(ThaiNumberFormatter.to_text(0)).to eq('ศูนย์')
      expect(ThaiNumberFormatter.to_text(1)).to eq('หนึ่ง')
      expect(ThaiNumberFormatter.to_text(10)).to eq('สิบ')
      expect(ThaiNumberFormatter.to_text(11)).to eq('สิบเอ็ด')
      expect(ThaiNumberFormatter.to_text(20)).to eq('ยี่สิบ')
      expect(ThaiNumberFormatter.to_text(100)).to eq('ร้อย')
    end
  end
end
```

---

*สรุปบทที่ 24: Gems และ Bundler เป็นส่วนสำคัญของ Ruby Ecosystem การเข้าใจวิธีจัดการ Dependencies, การสร้าง Gem ของตัวเอง และการรักษา Security จะช่วยให้พัฒนา Ruby Application ได้อย่างมีประสิทธิภาพ*
