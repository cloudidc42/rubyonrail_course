# ตอนที่ 24: Gems และ Bundler (Steps 521-540)

## บทนำ

**RubyGems** เป็นระบบ package manager สำหรับ Ruby ที่ช่วยให้คุณแชร์และใช้งาน code จากชุมชน Ruby ทั่วโลก **Bundler** เป็นเครื่องมือที่จัดการ dependencies ของ project ให้แน่ใจว่าทุกคนในทีมใช้ version เดียวกัน

---

## Step 521: RubyGems คืออะไร?

**Gem** คือ package ของ Ruby code ที่มีโครงสร้างมาตรฐาน ประกอบด้วย:
- Ruby code (library หรือ program)
- Documentation
- Tests
- Gemspec (metadata)

```bash
# ตรวจสอบ RubyGems version
gem --version
# 3.5.x

# ดู help
gem help
gem help commands

# ดูว่า gem อยู่ที่ไหน
gem env
# หรือดู specific info
gem env GEM_HOME  # ที่เก็บ gems
gem env GEM_PATH  # search paths
```

### โครงสร้างของ Gem

```
awesome_gem-1.0.0/
├── lib/
│   ├── awesome_gem.rb        # main file
│   └── awesome_gem/
│       ├── version.rb        # version constant
│       ├── calculator.rb     # classes
│       └── formatter.rb
├── spec/ (หรือ test/)
│   ├── spec_helper.rb
│   └── awesome_gem_spec.rb
├── bin/
│   └── awesome_gem           # executable (optional)
├── awesome_gem.gemspec       # metadata
├── Gemfile
├── Gemfile.lock
├── LICENSE.txt
└── README.md
```

---

## Step 522: gem Commands

```bash
# ==========================================
# ค้นหาและดู gems
# ==========================================
gem search rails              # ค้นหา gem ที่มี "rails"
gem search ^rails$            # ค้นหา exact match
gem list                      # รายการ gems ที่ install แล้ว
gem list rails                # กรองด้วยชื่อ
gem info rails                # ข้อมูลละเอียดของ gem

# ==========================================
# Install
# ==========================================
gem install rails             # install latest version
gem install rails -v 7.1.0   # install specific version
gem install rails --version "~> 7.1"  # install compatible version
gem install rails --no-document  # ไม่ install docs (เร็วกว่า)
gem install rails --pre         # install pre-release
gem install rails -N            # shorthand ของ --no-document

# ==========================================
# Update
# ==========================================
gem update                    # update all gems
gem update rails              # update specific gem
gem update --system           # update RubyGems itself

# ==========================================
# Uninstall
# ==========================================
gem uninstall rails           # uninstall latest
gem uninstall rails -v 7.0.0  # uninstall specific version
gem uninstall -aIx            # uninstall all (a=all versions, I=ignore deps, x=exec files)

# ==========================================
# Inspect installed gems
# ==========================================
gem contents rails            # รายการไฟล์ใน gem
gem open rails                # เปิด source ใน editor
gem which rails               # หา path ของ gem file
gem dependency rails          # ดู dependencies

# ==========================================
# Local gem cache
# ==========================================
gem fetch rails               # download .gem file ไม่ install
gem install --local rails     # install จาก local .gem file
gem unpack rails              # extract .gem เป็น directory

# ==========================================
# Server
# ==========================================
gem server                    # start local gem documentation server
# เปิด http://localhost:8808

# ==========================================
# Cleanup
# ==========================================
gem cleanup                   # ลบ old versions
gem cleanup rails             # ลบเฉพาะ old versions ของ rails
gem cleanup -d                # dry run (แสดงว่าจะลบอะไร)
```

---

## Step 523: Gemfile Structure

```ruby
# Gemfile

# Ruby version
ruby '3.2.0'

# Source - ที่ดาวน์โหลด gems
source 'https://rubygems.org'

# ==========================================
# Basic gems
# ==========================================
gem 'rails', '7.1.2'          # specific version
gem 'pg'                      # latest version
gem 'puma'                    # web server

# ==========================================
# Gems with options
# ==========================================
gem 'rails', path: '../rails'         # local path (development)
gem 'private_gem', git: 'https://github.com/user/gem.git'  # from git
gem 'private_gem', git: 'https://...', branch: 'main'       # specific branch
gem 'private_gem', git: 'https://...', tag: 'v1.0.0'        # specific tag
gem 'private_gem', git: 'https://...', ref: 'abc1234'       # specific commit

# ==========================================
# Gem Groups
# ==========================================
group :development do
  gem 'rubocop'               # code linter
  gem 'rubocop-rails'
  gem 'spring'                # faster startup
  gem 'web-console'
end

group :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'shoulda-matchers'
  gem 'simplecov'
end

group :development, :test do
  gem 'pry'                   # debugger
  gem 'pry-byebug'
  gem 'dotenv-rails'          # env variables
end

group :production do
  gem 'newrelic_rpm'          # monitoring
  gem 'lograge'               # logging
end

# ==========================================
# Conditional inclusion
# ==========================================
gem 'platform_gem', platforms: :ruby   # only on MRI Ruby
gem 'jruby_gem', platforms: :jruby     # only on JRuby
gem 'windows_gem', platforms: :x64_mingw  # only on Windows 64-bit

# ==========================================
# Gemspec integration (สำหรับ gem development)
# ==========================================
gemspec  # ใช้ dependencies จาก .gemspec file
```

---

## Step 524: Version Constraints

```ruby
# ==========================================
# Version constraints ใน Gemfile
# ==========================================

# Exact version
gem 'rails', '7.1.2'        # exactly 7.1.2

# Greater than or equal to
gem 'pg', '>= 1.0'          # 1.0 หรือมากกว่า

# Less than or equal to
gem 'ruby-debug', '<= 0.10.4'

# Greater than
gem 'rails', '> 5.0'

# Less than
gem 'rails', '< 8.0'

# Pessimistic version constraint (~>)
gem 'rails', '~> 7.1'       # >= 7.1, < 8.0
gem 'rails', '~> 7.1.2'     # >= 7.1.2, < 7.2.0
gem 'activesupport', '~> 7.0.0'  # >= 7.0.0, < 7.1.0

# Multiple constraints
gem 'rails', '>= 7.0', '< 8.0'

# ==========================================
# Semantic Versioning (SemVer): MAJOR.MINOR.PATCH
# ==========================================
# 1.2.3
# ^---- MAJOR: breaking changes
#   ^-- MINOR: new features (backward compatible)
#     ^- PATCH: bug fixes (backward compatible)

# ~> 7.1 หมายถึง: "compatible with 7.1"
# = >=7.1.0, < 8.0.0 (MINOR และ PATCH อาจเปลี่ยน)

# ~> 7.1.2 หมายถึง: "compatible with 7.1.2"
# = >=7.1.2, < 7.2.0 (เฉพาะ PATCH อาจเปลี่ยน)

# Examples:
gem 'devise', '~> 4.9'      # >= 4.9, < 5.0
gem 'sidekiq', '~> 7.2.0'   # >= 7.2.0, < 7.3.0
gem 'nokogiri', '>= 1.13.0'  # อย่างน้อย 1.13.0
```

---

## Step 525: Gemfile.lock

```bash
# Gemfile.lock ถูกสร้างอัตโนมัติโดย bundle install
# เก็บ exact versions ของ gems ทั้งหมด (รวม dependencies)

# ตัวอย่าง Gemfile.lock
```

```
GEM
  remote: https://rubygems.org/
  specs:
    actioncable (7.1.2)
      actionpack (= 7.1.2)
      activesupport (= 7.1.2)
    actionmailer (7.1.2)
      actionpack (= 7.1.2)
      actionview (= 7.1.2)
      activejob (= 7.1.2)
    rails (7.1.2)
      actioncable (= 7.1.2)
      actionmailer (= 7.1.2)
      actionpack (= 7.1.2)
    pg (1.5.4)
    puma (6.4.0)

PLATFORMS
  x86_64-linux
  arm64-darwin-23

DEPENDENCIES
  pg
  puma (~> 6.0)
  rails (~> 7.1.0)

BUNDLED WITH
   2.5.3
```

```bash
# ==========================================
# ทำไม Gemfile.lock สำคัญ?
# ==========================================
# 1. ทุกคนในทีมใช้ version เดียวกัน
# 2. Deploy ไป production ใช้ version เดิม
# 3. Reproducible builds

# Commit Gemfile.lock เสมอสำหรับ applications
# อย่า commit Gemfile.lock สำหรับ gems (libraries)

# ==========================================
# Update lock file
# ==========================================
bundle update                # update all gems (เปลี่ยน lock file)
bundle update rails          # update only rails
bundle update --minor        # update only minor versions
bundle update --patch        # update only patch versions
```

---

## Step 526: bundle Commands

```bash
# ==========================================
# bundle install
# ==========================================
bundle install               # install all gems in Gemfile
bundle install --without production  # skip production group
bundle install --path vendor/bundle  # install locally
bundle install --jobs 4      # parallel install
bundle install --retry 3     # retry on failure
bundle install --frozen      # fail ถ้า Gemfile.lock ล้าสมัย

# ==========================================
# bundle exec
# ==========================================
bundle exec rails server     # รัน command ใน bundle context
bundle exec rspec            # รัน rspec จาก bundle
bundle exec rake             # รัน rake tasks
bundle exec ruby my_script.rb

# ทำไมต้องใช้ bundle exec?
# - ใช้ gems จาก Gemfile แทน global gems
# - หลีกเลี่ยง version conflicts

# ==========================================
# bundle update
# ==========================================
bundle update                # update all (careful!)
bundle update rails          # update specific gem
bundle update rails activesupport  # update multiple
bundle update --conservative  # minimal updates

# ==========================================
# bundle check
# ==========================================
bundle check                 # ตรวจสอบว่า gems พร้อมใช้งาน

# ==========================================
# bundle show
# ==========================================
bundle show                  # รายการ gems ทั้งหมด
bundle show rails            # path ของ rails gem
bundle info rails            # ข้อมูลละเอียด

# ==========================================
# bundle outdated
# ==========================================
bundle outdated              # gems ที่มี update
bundle outdated --minor      # เฉพาะ minor updates
bundle outdated --strict     # เฉพาะ versions ที่ตรง constraints

# ==========================================
# bundle console
# ==========================================
bundle console               # irb กับ gems loaded

# ==========================================
# bundle clean
# ==========================================
bundle clean                 # ลบ gems ที่ไม่ได้ใช้
bundle clean --force         # force clean

# ==========================================
# bundle binstubs
# ==========================================
bundle binstubs rspec-core   # สร้าง bin/rspec
bundle binstubs rails        # สร้าง bin/rails
# หลังจากนี้ใช้ ./bin/rspec แทน bundle exec rspec

# ==========================================
# bundle config
# ==========================================
bundle config list           # ดู config ทั้งหมด
bundle config set path vendor/bundle  # set path
bundle config set without development test  # skip groups
bundle config unset path     # ลบ config
```

---

## Step 527: Gem Groups

```ruby
# Gemfile กับ groups ที่ชัดเจน
source 'https://rubygems.org'
ruby '3.2.0'

# ===========================
# Core gems (ทุก environment)
# ===========================
gem 'rails', '~> 7.1'
gem 'pg', '>= 1.0'
gem 'redis', '~> 5.0'
gem 'sidekiq', '~> 7.0'
gem 'image_processing', '~> 1.2'

# ===========================
# Development only
# ===========================
group :development do
  gem 'web-console'
  gem 'spring'
  gem 'spring-watcher-listen'
  gem 'listen'
  gem 'letter_opener'         # preview emails in browser
  gem 'rack-mini-profiler'    # performance profiler
  gem 'bullet'                # N+1 query detector
  gem 'annotate'              # add DB schema comments
end

# ===========================
# Test only
# ===========================
group :test do
  gem 'rspec-rails'
  gem 'capybara'
  gem 'selenium-webdriver'
  gem 'webdrivers'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'shoulda-matchers'
  gem 'simplecov', require: false
  gem 'vcr'
  gem 'webmock'
  gem 'database_cleaner-active_record'
end

# ===========================
# Development + Test
# ===========================
group :development, :test do
  gem 'pry-rails'
  gem 'pry-byebug'
  gem 'dotenv-rails'
  gem 'rubocop-rails-omakase', require: false
end

# ===========================
# Production only
# ===========================
group :production do
  gem 'aws-sdk-s3'            # file storage
  gem 'newrelic_rpm'          # monitoring
  gem 'lograge'               # structured logging
  gem 'rack-timeout'          # request timeout
end
```

```bash
# Install without certain groups
bundle install --without development test

# หรือ ใน config
bundle config set without development:test
bundle install

# BUNDLE_WITHOUT environment variable
BUNDLE_WITHOUT=development:test bundle install
```

---

## Step 528: สร้าง Gem ของตัวเอง - Bundle Gem

```bash
# สร้าง gem structure ด้วย bundler
bundle gem my_awesome_gem

# สร้าง:
# my_awesome_gem/
# ├── bin/
# │   ├── console     (interactive console)
# │   └── setup       (setup script)
# ├── lib/
# │   ├── my_awesome_gem.rb
# │   └── my_awesome_gem/
# │       └── version.rb
# ├── spec/
# │   ├── my_awesome_gem_spec.rb
# │   └── spec_helper.rb
# ├── .git/
# ├── .gitignore
# ├── .rspec
# ├── CHANGELOG.md
# ├── CODE_OF_CONDUCT.md
# ├── Gemfile
# ├── LICENSE.txt
# ├── README.md
# ├── Rakefile
# └── my_awesome_gem.gemspec

# Options
bundle gem my_gem --mit      # MIT license
bundle gem my_gem --coc      # Code of Conduct
bundle gem my_gem --test rspec  # RSpec testing
bundle gem my_gem --ci github   # GitHub Actions CI
bundle gem my_gem --exe      # executable
```

---

## Step 529: Gemspec File

```ruby
# my_awesome_gem.gemspec

require_relative "lib/my_awesome_gem/version"

Gem::Specification.new do |spec|
  # ==========================================
  # Required fields
  # ==========================================
  spec.name    = "my_awesome_gem"
  spec.version = MyAwesomeGem::VERSION
  spec.authors = ["Your Name"]
  spec.email   = ["your.email@example.com"]

  # ==========================================
  # Description
  # ==========================================
  spec.summary     = "A short description (one sentence)"
  spec.description = "A longer description of what this gem does, what problems it solves, and how to use it."
  spec.homepage    = "https://github.com/yourusername/my_awesome_gem"
  spec.license     = "MIT"

  # ==========================================
  # Ruby version requirement
  # ==========================================
  spec.required_ruby_version = ">= 3.0.0"

  # ==========================================
  # Files to include
  # ==========================================
  spec.files = Dir.chdir(__dir__) do
    `git ls-files -z`.split("\x0").reject do |f|
      (File.expand_path(f) == __FILE__) ||
        f.start_with?(*%w[bin/ test/ spec/ features/ .git .circleci appveyor Gemfile])
    end
  end

  spec.bindir        = "exe"
  spec.executables   = spec.files.grep(%r{\Aexe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  # ==========================================
  # Metadata
  # ==========================================
  spec.metadata = {
    "homepage_uri"    => spec.homepage,
    "source_code_uri" => "https://github.com/yourusername/my_awesome_gem",
    "changelog_uri"   => "https://github.com/yourusername/my_awesome_gem/blob/main/CHANGELOG.md",
    "bug_tracker_uri" => "https://github.com/yourusername/my_awesome_gem/issues",
    "documentation_uri" => "https://rubydoc.info/gems/my_awesome_gem",
    "rubygems_mfa_required" => "true"  # require MFA for publishing
  }

  # ==========================================
  # Runtime dependencies
  # ==========================================
  spec.add_dependency "activesupport", "~> 7.0"
  spec.add_dependency "faraday", "~> 2.0"

  # ==========================================
  # Development dependencies
  # ==========================================
  spec.add_development_dependency "rspec", "~> 3.12"
  spec.add_development_dependency "rake", "~> 13.0"
  spec.add_development_dependency "rubocop", "~> 1.50"
end
```

---

## Step 530: Gem Structure และ Code

```ruby
# lib/my_awesome_gem/version.rb
module MyAwesomeGem
  VERSION = "0.1.0"
end
```

```ruby
# lib/my_awesome_gem.rb - Main entry point
require "my_awesome_gem/version"
require "my_awesome_gem/configuration"
require "my_awesome_gem/errors"
require "my_awesome_gem/client"

module MyAwesomeGem
  class Error < StandardError; end
  class ConfigurationError < Error; end
  class ApiError < Error; end

  class << self
    attr_reader :configuration

    def configure
      @configuration ||= Configuration.new
      yield(@configuration) if block_given?
      @configuration
    end

    def reset!
      @configuration = nil
    end
  end
end
```

```ruby
# lib/my_awesome_gem/configuration.rb
module MyAwesomeGem
  class Configuration
    attr_accessor :api_key, :api_url, :timeout, :logger

    def initialize
      @api_url = "https://api.example.com"
      @timeout = 30
      @logger  = Logger.new($stdout)
    end

    def valid?
      !@api_key.nil? && !@api_key.empty?
    end
  end
end
```

```ruby
# lib/my_awesome_gem/client.rb
require 'net/http'
require 'json'

module MyAwesomeGem
  class Client
    def initialize(config = MyAwesomeGem.configuration)
      raise ConfigurationError, "API key is required" unless config&.valid?
      @config = config
    end

    def get(endpoint, params = {})
      uri = build_uri(endpoint, params)
      response = make_request(:get, uri)
      parse_response(response)
    end

    def post(endpoint, body = {})
      uri = build_uri(endpoint)
      response = make_request(:post, uri, body)
      parse_response(response)
    end

    private

    def build_uri(endpoint, params = {})
      uri = URI.parse("#{@config.api_url}#{endpoint}")
      uri.query = URI.encode_www_form(params) unless params.empty?
      uri
    end

    def make_request(method, uri, body = nil)
      http = Net::HTTP.new(uri.host, uri.port)
      http.use_ssl = uri.scheme == 'https'
      http.read_timeout = @config.timeout

      request = case method
        when :get  then Net::HTTP::Get.new(uri)
        when :post then Net::HTTP::Post.new(uri)
      end

      request['Authorization'] = "Bearer #{@config.api_key}"
      request['Content-Type']  = 'application/json'
      request.body = body.to_json if body

      @config.logger.debug "#{method.upcase} #{uri}"
      http.request(request)
    rescue Net::TimeoutError
      raise ApiError, "Request timed out after #{@config.timeout}s"
    rescue => e
      raise ApiError, "Request failed: #{e.message}"
    end

    def parse_response(response)
      body = JSON.parse(response.body) rescue response.body

      case response.code.to_i
      when 200..299
        body
      when 401
        raise ApiError, "Unauthorized: check your API key"
      when 404
        raise ApiError, "Resource not found"
      when 422
        raise ApiError, "Validation failed: #{body['errors']}"
      when 500..599
        raise ApiError, "Server error: #{response.code}"
      else
        raise ApiError, "Unexpected response: #{response.code}"
      end
    end
  end
end
```

---

## Step 531: Writing Gem Code

```ruby
# ตัวอย่าง gem ที่สมบูรณ์กว่า: text_analyzer gem

# lib/text_analyzer.rb
require "text_analyzer/version"
require "text_analyzer/analyzer"
require "text_analyzer/formatter"

module TextAnalyzer
  class << self
    def analyze(text)
      Analyzer.new(text)
    end

    def configure
      @config ||= Configuration.new
      yield(@config) if block_given?
    end
  end
end

# lib/text_analyzer/analyzer.rb
module TextAnalyzer
  class Analyzer
    attr_reader :text

    def initialize(text)
      @text = text.to_s
    end

    def word_count
      words.size
    end

    def sentence_count
      @text.scan(/[.!?]+/).size
    end

    def char_count(include_spaces: true)
      include_spaces ? @text.length : @text.gsub(/\s/, '').length
    end

    def words
      @words ||= @text.downcase.scan(/\b[a-z']+\b/)
    end

    def unique_words
      words.uniq
    end

    def word_frequency
      words.tally.sort_by { |_, v| -v }.to_h
    end

    def most_common_words(n = 10)
      word_frequency.first(n)
    end

    def reading_time(wpm: 200)
      (word_count.to_f / wpm).ceil
    end

    def flesch_kincaid_grade
      avg_sentence_length = word_count.to_f / sentence_count
      avg_syllables = syllable_count.to_f / word_count
      0.39 * avg_sentence_length + 11.8 * avg_syllables - 15.59
    end

    def summary(max_sentences: 3)
      sentences = @text.split(/[.!?]+/).map(&:strip).reject(&:empty?)
      important = sentences.first(max_sentences)
      important.join('. ') + '.'
    end

    def to_h
      {
        word_count:     word_count,
        unique_words:   unique_words.size,
        sentence_count: sentence_count,
        char_count:     char_count,
        reading_time:   reading_time
      }
    end

    private

    def syllable_count
      words.sum { |w| count_syllables(w) }
    end

    def count_syllables(word)
      word.downcase.gsub(/[^aeiouy]/, ' ').split.size.then { |n| [n, 1].max }
    end
  end
end
```

---

## Step 532: Testing Gems

```ruby
# spec/spec_helper.rb
require "bundler/setup"
require "text_analyzer"

RSpec.configure do |config|
  config.expect_with :rspec do |c|
    c.syntax = :expect
  end
  config.order = :random
end

# spec/text_analyzer/analyzer_spec.rb
RSpec.describe TextAnalyzer::Analyzer do
  let(:sample_text) do
    "The quick brown fox jumps over the lazy dog. " \
    "This is a simple sentence. Ruby is a great language!"
  end

  subject(:analyzer) { described_class.new(sample_text) }

  describe "#word_count" do
    it "counts all words" do
      expect(analyzer.word_count).to eq(18)
    end

    it "returns 0 for empty text" do
      expect(described_class.new("").word_count).to eq(0)
    end
  end

  describe "#sentence_count" do
    it "counts sentences" do
      expect(analyzer.sentence_count).to eq(3)
    end
  end

  describe "#word_frequency" do
    it "counts word occurrences" do
      freq = analyzer.word_frequency
      expect(freq["the"]).to eq(2)
    end

    it "sorts by frequency" do
      freq = analyzer.word_frequency
      values = freq.values
      expect(values).to eq(values.sort.reverse)
    end
  end

  describe "#most_common_words" do
    it "returns N most common words" do
      result = analyzer.most_common_words(3)
      expect(result.size).to eq(3)
    end
  end

  describe "#reading_time" do
    it "estimates reading time in minutes" do
      expect(analyzer.reading_time).to be_a(Integer)
      expect(analyzer.reading_time).to be >= 1
    end
  end

  describe "#to_h" do
    it "returns hash with all stats" do
      hash = analyzer.to_h
      expect(hash).to include(:word_count, :unique_words, :sentence_count)
    end
  end
end
```

---

## Step 533: Publishing to RubyGems.org

```bash
# 1. สร้าง account บน rubygems.org

# 2. Setup credentials
gem signin
# หรือ
gem push --key rubygems

# 3. Build gem
gem build my_awesome_gem.gemspec
# สร้าง my_awesome_gem-0.1.0.gem

# 4. Push to RubyGems.org
gem push my_awesome_gem-0.1.0.gem

# 5. ตรวจสอบ
gem info my_awesome_gem

# ==========================================
# Update version
# ==========================================
# แก้ lib/my_awesome_gem/version.rb
# VERSION = "0.2.0"

# Build และ push อีกครั้ง
gem build my_awesome_gem.gemspec
gem push my_awesome_gem-0.2.0.gem

# ==========================================
# Yank (ถอน) version
# ==========================================
gem yank my_awesome_gem -v 0.1.0

# ==========================================
# Rake tasks สำหรับ gem release
# ==========================================
# Rakefile
require "bundler/gem_tasks"
require "rspec/core/rake_task"

RSpec::Core::RakeTask.new(:spec)

task default: :spec
```

```ruby
# Rakefile สำหรับ gem development
require "bundler/gem_tasks"
require "rspec/core/rake_task"
require "rubocop/rake_task"

RSpec::Core::RakeTask.new(:spec)
RuboCop::RakeTask.new

desc "Run specs and rubocop"
task default: %i[spec rubocop]

# รัน: rake release  # bump version, tag, push to rubygems
# รัน: rake spec     # run tests
# รัน: rake rubocop  # lint code
```

---

## Step 534: Popular Ruby Gems Overview

```ruby
# ==========================================
# Web Frameworks
# ==========================================
gem 'rails'          # Full-stack web framework
gem 'sinatra'        # Minimal web framework
gem 'hanami'         # Clean architecture framework
gem 'grape'          # REST API framework

# ==========================================
# Database
# ==========================================
gem 'activerecord'   # ORM (comes with Rails)
gem 'sequel'         # Alternative ORM
gem 'pg'             # PostgreSQL adapter
gem 'mysql2'         # MySQL adapter
gem 'sqlite3'        # SQLite adapter
gem 'redis'          # Redis client
gem 'mongoid'        # MongoDB ORM

# ==========================================
# Authentication
# ==========================================
gem 'devise'         # User authentication
gem 'bcrypt'         # Password hashing
gem 'jwt'            # JSON Web Tokens
gem 'doorkeeper'     # OAuth 2.0
gem 'omniauth'       # Multi-provider auth
gem 'cancancan'      # Authorization
gem 'pundit'         # Policy-based authorization

# ==========================================
# HTTP Clients
# ==========================================
gem 'faraday'        # HTTP client with middleware
gem 'httparty'       # Simple HTTP client
gem 'rest-client'    # REST client
gem 'typhoeus'       # Parallel HTTP requests
gem 'mechanize'      # Web scraping with cookies

# ==========================================
# Background Jobs
# ==========================================
gem 'sidekiq'        # Background jobs with Redis
gem 'delayed_job'    # Database-backed jobs
gem 'resque'         # Redis-backed jobs
gem 'sucker_punch'   # In-process async jobs
gem 'good_job'       # Database-backed with Postgres

# ==========================================
# Testing
# ==========================================
gem 'rspec'          # Testing framework
gem 'minitest'       # Built-in testing
gem 'capybara'       # Browser testing
gem 'factory_bot'    # Test fixtures
gem 'faker'          # Fake data
gem 'vcr'            # Record HTTP interactions
gem 'webmock'        # Stub HTTP requests
gem 'timecop'        # Time manipulation

# ==========================================
# Code Quality
# ==========================================
gem 'rubocop'        # Style linter
gem 'reek'           # Code smell detector
gem 'flay'           # Code duplication
gem 'flog'           # Code complexity
gem 'brakeman'       # Security scanner

# ==========================================
# Performance
# ==========================================
gem 'rack-mini-profiler'  # Performance profiler
gem 'bullet'              # N+1 query detector
gem 'skylight'            # Performance monitoring
gem 'scout_apm'           # APM solution

# ==========================================
# File Processing
# ==========================================
gem 'carrierwave'    # File uploading
gem 'shrine'         # File uploading (modern)
gem 'image_processing'  # Image resizing
gem 'mini_magick'    # ImageMagick wrapper
gem 'pdf-reader'     # Read PDFs
gem 'prawn'          # Generate PDFs
gem 'axlsx'          # Generate Excel files

# ==========================================
# Email
# ==========================================
gem 'action_mailer'  # Built into Rails
gem 'mail'           # Email composition
gem 'letter_opener'  # Preview emails
gem 'mailgun-ruby'   # Mailgun client

# ==========================================
# Serialization
# ==========================================
gem 'active_model_serializers'  # JSON serialization
gem 'fast_jsonapi'   # Fast JSON:API serializer
gem 'jbuilder'       # JSON builder (Rails)
gem 'blueprinter'    # Object serialization
gem 'alba'           # Fast serialization

# ==========================================
# CLI
# ==========================================
gem 'thor'           # CLI toolkit
gem 'optparse'       # Built-in option parsing
gem 'tty-prompt'     # Interactive CLI
gem 'colorize'       # Colored output
gem 'terminal-table' # ASCII tables

# ==========================================
# Debugging
# ==========================================
gem 'pry'            # Better REPL
gem 'pry-byebug'     # Debugger
gem 'byebug'         # Debugger
gem 'binding_of_caller'  # Access binding
gem 'better_errors'  # Better error pages
```

---

## Step 535: สร้าง Gem ตั้งแต่ต้น - ตัวอย่างสมบูรณ์

```ruby
# สร้าง gem ชื่อ "thai_text" ที่ process ข้อความภาษาไทย

# lib/thai_text.rb
require "thai_text/version"
require "thai_text/analyzer"
require "thai_text/formatter"
require "thai_text/validator"

module ThaiText
  class Error < StandardError; end

  def self.analyze(text)
    Analyzer.new(text)
  end

  def self.validate_id(id)
    Validator.valid_national_id?(id)
  end

  def self.format_phone(phone)
    Formatter.phone(phone)
  end
end

# lib/thai_text/version.rb
module ThaiText
  VERSION = "1.0.0"
end

# lib/thai_text/analyzer.rb
module ThaiText
  class Analyzer
    THAI_RANGE = ("฀".."๿")

    def initialize(text)
      @text = text.to_s
    end

    def thai_char_count
      @text.chars.count { |c| THAI_RANGE.cover?(c) }
    end

    def contains_thai?
      @text.chars.any? { |c| THAI_RANGE.cover?(c) }
    end

    def thai_percentage
      return 0.0 if @text.empty?
      (thai_char_count.to_f / @text.length * 100).round(2)
    end

    def thai_words
      # ตัวอย่างง่ายๆ - real word segmentation ซับซ้อนกว่านี้
      @text.scan(/[฀-๿]+/)
    end

    def stats
      {
        total_chars:      @text.length,
        thai_chars:       thai_char_count,
        thai_percentage:  thai_percentage,
        thai_words:       thai_words.count,
        contains_thai:    contains_thai?
      }
    end
  end
end

# lib/thai_text/validator.rb
module ThaiText
  module Validator
    # ตรวจสอบเลขบัตรประชาชนไทย
    def self.valid_national_id?(id)
      digits = id.to_s.gsub(/\D/, '')
      return false unless digits.length == 13

      sum = 0
      12.times { |i| sum += digits[i].to_i * (13 - i) }
      checksum = (11 - sum % 11) % 10
      checksum == digits[12].to_i
    end

    # ตรวจสอบเบอร์โทรศัพท์ไทย
    def self.valid_phone?(phone)
      cleaned = phone.to_s.gsub(/[\s\-\.]/, '')
      cleaned.match?(/\A(0[689]\d{8}|0[1-5]\d{7})\z/)
    end

    # ตรวจสอบ Thai postal code
    def self.valid_postal_code?(code)
      code.to_s.match?(/\A[1-9]\d{4}\z/)
    end
  end
end

# lib/thai_text/formatter.rb
module ThaiText
  module Formatter
    def self.phone(number)
      digits = number.to_s.gsub(/\D/, '')
      return number unless digits.length.between?(9, 10)

      if digits.length == 10
        "#{digits[0..1]}-#{digits[2..4]}-#{digits[5..9]}"
      else
        "#{digits[0..1]}-#{digits[2..8]}"
      end
    end

    def self.national_id(id)
      digits = id.to_s.gsub(/\D/, '')
      return id unless digits.length == 13
      "#{digits[0]}-#{digits[1..4]}-#{digits[5..9]}-#{digits[10..11]}-#{digits[12]}"
    end

    def self.currency(amount, currency: "฿")
      "#{currency}#{format('%.2f', amount.to_f)}"
    end
  end
end
```

---

## แบบฝึกหัด: Gems และ Bundler (20 ข้อ)

**ข้อ 1:** สร้าง Gemfile สำหรับ REST API application

```ruby
# เฉลย
source "https://rubygems.org"
ruby "3.2.0"

gem "rails", "~> 7.1"
gem "pg", "~> 1.5"
gem "puma", "~> 6.0"
gem "rack-cors"
gem "jwt", "~> 2.7"
gem "bcrypt", "~> 3.1"
gem "kaminari"                    # pagination
gem "active_model_serializers"    # JSON serialization
gem "sidekiq", "~> 7.0"          # background jobs
gem "redis", "~> 5.0"

group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
  gem "pry-rails"
  gem "dotenv-rails"
end

group :test do
  gem "shoulda-matchers"
  gem "simplecov", require: false
  gem "database_cleaner-active_record"
end
```

**ข้อ 2:** เขียน version constraints สำหรับ Rails project dependencies

```ruby
# เฉลย - เหตุผลของ constraints
gem 'rails', '~> 7.1.0'        # อยู่ใน 7.1.x เท่านั้น (patch updates OK)
gem 'pg', '>= 1.1', '< 3.0'    # 1.1+ แต่ไม่ถึง 3.0
gem 'devise', '~> 4.9'         # >= 4.9, < 5.0
gem 'sidekiq', '~> 7.2'        # >= 7.2, < 8.0
gem 'jwt', '~> 2.7.1'          # >= 2.7.1, < 2.8.0 (strict)
gem 'nokogiri', '>= 1.13.10'   # minimum version สำหรับ security
```

**ข้อ 3:** สร้างโครงสร้าง gem ด้วย bundle gem

```bash
# เฉลย
bundle gem string_utils --mit --test rspec --ci github

# โครงสร้างที่ได้:
# string_utils/
# ├── lib/
# │   ├── string_utils.rb
# │   └── string_utils/
# │       └── version.rb
# ├── spec/
# │   └── string_utils_spec.rb
# ├── string_utils.gemspec
# └── Gemfile
```

**ข้อ 4:** เขียน gemspec สำหรับ simple utility gem

```ruby
# เฉลย
require_relative "lib/string_utils/version"

Gem::Specification.new do |spec|
  spec.name    = "string_utils"
  spec.version = StringUtils::VERSION
  spec.authors = ["Developer Name"]
  spec.email   = ["dev@example.com"]

  spec.summary     = "Useful string utilities for Ruby"
  spec.description = "Collection of helpful string manipulation methods"
  spec.homepage    = "https://github.com/dev/string_utils"
  spec.license     = "MIT"

  spec.required_ruby_version = ">= 3.0.0"

  spec.files         = Dir["lib/**/*.rb", "README.md", "LICENSE.txt"]
  spec.require_paths = ["lib"]

  spec.add_development_dependency "rspec", "~> 3.12"
  spec.add_development_dependency "rake", "~> 13.0"
end
```

**ข้อ 5-10:** (ตัวอย่างย่อ)

```bash
# ข้อ 5: bundle exec vs โดยตรง
# รัน rspec ผ่าน bundle
bundle exec rspec spec/models/

# สร้าง binstub เพื่อสะดวก
bundle binstubs rspec-core
./bin/rspec spec/models/  # เหมือนกัน

# ข้อ 6: update gem อย่างปลอดภัย
bundle outdated --strict
bundle update devise --conservative

# ข้อ 7: install gems locally (CI/CD)
bundle install --path vendor/bundle --jobs 4
bundle exec rails s

# ข้อ 8: skip groups in production
bundle install --without development test

# ข้อ 9: อ่าน gem source
bundle open devise
gem open activerecord

# ข้อ 10: pin version เมื่อ bug
# Gemfile
gem 'activerecord', '7.1.1'  # pin เพราะ 7.1.2 มี bug
```

**ข้อ 11-20:** (สร้าง gem จริง)

```ruby
# ข้อ 11: สร้าง Color class สำหรับ gem
class Color
  attr_reader :r, :g, :b

  def initialize(r, g, b)
    @r, @g, @b = r.clamp(0, 255), g.clamp(0, 255), b.clamp(0, 255)
  end

  def self.from_hex(hex)
    hex = hex.gsub('#', '')
    r = hex[0..1].to_i(16)
    g = hex[2..3].to_i(16)
    b = hex[4..5].to_i(16)
    new(r, g, b)
  end

  def to_hex
    "##{[@r, @g, @b].map { |c| c.to_s(16).rjust(2, '0') }.join.upcase}"
  end

  def to_hsl
    r, g, b = @r / 255.0, @g / 255.0, @b / 255.0
    max, min = [r, g, b].max, [r, g, b].min
    l = (max + min) / 2.0

    if max == min
      h = s = 0
    else
      d = max - min
      s = l > 0.5 ? d / (2 - max - min) : d / (max + min)
      h = case max
          when r then ((g - b) / d + (g < b ? 6 : 0)) / 6.0
          when g then ((b - r) / d + 2) / 6.0
          when b then ((r - g) / d + 4) / 6.0
          end
    end

    { h: (h * 360).round, s: (s * 100).round, l: (l * 100).round }
  end

  def lighten(amount = 10)
    hsl = to_hsl
    from_hsl(hsl[:h], hsl[:s], [hsl[:l] + amount, 100].min)
  end

  def darken(amount = 10)
    hsl = to_hsl
    from_hsl(hsl[:h], hsl[:s], [hsl[:l] - amount, 0].max)
  end

  def mix(other, weight = 0.5)
    Color.new(
      ((@r * (1 - weight)) + (other.r * weight)).round,
      ((@g * (1 - weight)) + (other.g * weight)).round,
      ((@b * (1 - weight)) + (other.b * weight)).round
    )
  end

  def contrast_color
    luminance = (0.299 * @r + 0.587 * @g + 0.114 * @b) / 255
    luminance > 0.5 ? Color.new(0, 0, 0) : Color.new(255, 255, 255)
  end

  def to_s
    to_hex
  end

  private

  def from_hsl(h, s, l)
    h, s, l = h / 360.0, s / 100.0, l / 100.0
    if s == 0
      rgb = (l * 255).round
      Color.new(rgb, rgb, rgb)
    else
      q = l < 0.5 ? l * (1 + s) : l + s - l * s
      p = 2 * l - q
      r = hue_to_rgb(p, q, h + 1.0/3)
      g = hue_to_rgb(p, q, h)
      b = hue_to_rgb(p, q, h - 1.0/3)
      Color.new((r * 255).round, (g * 255).round, (b * 255).round)
    end
  end

  def hue_to_rgb(p, q, t)
    t += 1 if t < 0
    t -= 1 if t > 1
    return p + (q - p) * 6 * t if t < 1.0/6
    return q if t < 1.0/2
    return p + (q - p) * (2.0/3 - t) * 6 if t < 2.0/3
    p
  end
end

# ข้อ 12-20 specs สำหรับ Color gem
RSpec.describe Color do
  describe ".from_hex" do
    it "creates color from hex" do
      color = Color.from_hex("#FF5733")
      expect(color.r).to eq(255)
      expect(color.g).to eq(87)
      expect(color.b).to eq(51)
    end
  end

  describe "#to_hex" do
    it "converts to hex string" do
      color = Color.new(255, 87, 51)
      expect(color.to_hex).to eq("#FF5733")
    end
  end

  describe "#mix" do
    it "mixes two colors" do
      red = Color.new(255, 0, 0)
      blue = Color.new(0, 0, 255)
      mixed = red.mix(blue, 0.5)
      expect(mixed.r).to eq(128)
      expect(mixed.b).to eq(128)
    end
  end

  describe "#contrast_color" do
    it "returns black for light colors" do
      white = Color.new(255, 255, 255)
      expect(white.contrast_color.to_hex).to eq("#000000")
    end

    it "returns white for dark colors" do
      black = Color.new(0, 0, 0)
      expect(black.contrast_color.to_hex).to eq("#FFFFFF")
    end
  end
end
```

---

## สรุป

### RubyGems Commands สำคัญ
```bash
gem install <name>       # install gem
gem uninstall <name>     # remove gem
gem update <name>        # update gem
gem list                 # list installed gems
gem search <query>       # search rubygems.org
```

### Bundler Commands สำคัญ
```bash
bundle install           # install from Gemfile
bundle update            # update gems
bundle exec <command>    # run with bundle context
bundle outdated          # check for updates
bundle show              # list gem paths
```

### Version Constraints
| Operator | ความหมาย |
|----------|----------|
| `= 1.0` | exactly 1.0 |
| `!= 1.0` | not 1.0 |
| `> 1.0` | greater than 1.0 |
| `>= 1.0` | 1.0 or greater |
| `< 2.0` | less than 2.0 |
| `<= 2.0` | 2.0 or less |
| `~> 1.5` | >= 1.5, < 2.0 |
| `~> 1.5.0` | >= 1.5.0, < 1.6.0 |

### Best Practices
1. **Lock dependencies** - commit Gemfile.lock (applications)
2. **Don't lock** Gemfile.lock (libraries/gems)
3. **Use semantic versioning** - `~>` operator
4. **Group correctly** - :development, :test, :production
5. **bundle exec** - always use for consistency
6. **Keep gems updated** - use bundle outdated regularly

---

*ตอนถัดไป: ตอนที่ 25 - Concurrency และ Threads*
