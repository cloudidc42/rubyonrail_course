# Part 80: Contributing to Open Source Ruby/Rails

## ขั้นตอนที่ 1721-1740

---

## ขั้นตอนที่ 1721: การหา Issues ที่เหมาะสม

การ contribute ไปยัง open source projects เริ่มต้นจากการหา issue ที่เหมาะสมกับ level ของเรา

### ป้ายกำกับ Issues ที่ควรมองหา

- **good first issue** - เหมาะสำหรับผู้เริ่มต้น
- **help wanted** - ต้องการความช่วยเหลือจาก community
- **bug** - bug ที่ต้องแก้ไข
- **enhancement** - feature ใหม่ที่ต้องการ
- **documentation** - ปรับปรุง docs (เหมาะสำหรับเริ่มต้น)

### แหล่งหา Issues

```bash
# GitHub Search
# Rails issues สำหรับ beginners:
# https://github.com/rails/rails/labels/good%20first%20issue

# Ruby issues:
# https://github.com/ruby/ruby/labels/good%20first%20issue

# Tools ช่วยหา
# - https://goodfirstissue.dev/?language=ruby
# - https://github.com/explore
# - https://up-for-grabs.net/#/filters?tags=ruby
```

### ตัวอย่างการประเมิน Issue

```
Good Issue ✅:
- มี description ชัดเจน
- มี steps to reproduce (สำหรับ bugs)
- Discussion ใน comments ยังเงียบ (ไม่มีคนรับ PR แล้ว)
- ขนาดพอเหมาะ (ไม่ใหญ่เกิน)

Bad Issue ❌:
- Description ไม่ชัด
- มี PR pending อยู่แล้ว
- เป็น feature ใหญ่ที่ต้องการ RFC ก่อน
- เกี่ยวกับ security vulnerability
```

---

## ขั้นตอนที่ 1722: Setup Rails Development Environment

```bash
# 1. Fork Rails repository
# ไปที่ https://github.com/rails/rails
# กด Fork button

# 2. Clone fork
git clone https://github.com/YOUR_USERNAME/rails.git
cd rails

# 3. Add upstream remote
git remote add upstream https://github.com/rails/rails.git

# 4. ติดตั้ง dependencies
bundle install

# หรือเฉพาะ gem ที่ต้องการ
cd activerecord && bundle install

# 5. ตั้งค่า database สำหรับ testing
cd activerecord
cp test/config.example.yml test/config.yml
# แก้ไข config ตาม database ที่มี

# 6. รัน tests
bundle exec rake test  # รัน test ทั้งหมด
bundle exec rake test TEST=test/cases/migration_test.rb  # รัน test เฉพาะไฟล์
bundle exec rake test TEST=test/cases/migration_test.rb TESTOPTS="-n test_add_column"  # รัน test เดียว
```

### Ruby Development Environment

```bash
# 1. Fork และ Clone
git clone https://github.com/YOUR_USERNAME/ruby.git
cd ruby

# 2. ติดตั้ง dependencies
sudo apt-get install -y autoconf bison build-essential libssl-dev libreadline-dev \
  zlib1g-dev libyaml-dev libncurses5-dev libgdbm-dev libffi-dev

# 3. Build Ruby
autoconf
./configure --prefix=$HOME/.rubies/ruby-dev
make
make install

# 4. รัน tests
make test
make test-basic
```

---

## ขั้นตอนที่ 1723: Git Workflow สำหรับ Contributions

```bash
# 1. Sync กับ upstream ก่อนเริ่มงาน
git fetch upstream
git checkout main
git merge upstream/main

# 2. สร้าง branch ใหม่สำหรับแต่ละ contribution
git checkout -b fix/issue-1234-correct-migration-error
# หรือ
git checkout -b feature/add-bulk-insert-support

# Naming conventions
# fix/issue-<number>-<short-description>
# feature/<short-description>
# docs/<what-you-documenting>
# refactor/<what-you-refactoring>

# 3. ทำการแก้ไข code
# ... edit files ...

# 4. ตรวจสอบ changes
git status
git diff

# 5. Stage และ commit
git add -p  # ดีกว่า git add . เพราะ review ทีละ hunk
git commit -m "Fix migration error when column type changes

Previously, when changing a column from string to integer,
the migration would raise an ArgumentError incorrectly.
This fix checks the current column type before raising.

Fixes #1234"

# 6. Push ไปยัง fork
git push origin fix/issue-1234-correct-migration-error

# 7. สร้าง Pull Request
# ไปที่ GitHub และสร้าง PR
```

### Commit Message Format

```
# Format
<type>(<scope>): <short description>

<body - optional>

<footer - optional>

# Examples
fix(active_record): correct column type detection in migrations

When adding an index on a polymorphic column, Rails was raising
a NoMethodError because the column object didn't respond to #type.

This commit adds a nil check before calling #type on the column.

Fixes #44123

---

feat(action_cable): add support for binary messages in WebSocket

Previously, Action Cable only supported text messages. This adds
support for binary messages by detecting the MessagePack format.

Closes #45678
```

---

## ขั้นตอนที่ 1724: การเขียน Tests สำหรับ Contribution

### Rails Test Structure

```ruby
# test/cases/migration_test.rb (ใน ActiveRecord)
require "cases/helper"

class MigrationColumnTest < ActiveRecord::TestCase
  def setup
    @connection = ActiveRecord::Base.connection
    @connection.create_table :test_columns, force: true do |t|
      t.string :name
    end
  end

  def teardown
    @connection.drop_table :test_columns, if_exists: true
  end

  # ตั้งชื่อ test ให้ชัดเจน: test_<what>_<when>_<expected>
  def test_change_column_preserves_null_constraint
    @connection.add_column :test_columns, :age, :string, null: false
    @connection.change_column :test_columns, :age, :integer

    column = @connection.columns(:test_columns).find { |c| c.name == 'age' }
    assert_not column.null
  end

  def test_add_column_with_default_value
    @connection.add_column :test_columns, :score, :integer, default: 0

    column = @connection.columns(:test_columns).find { |c| c.name == 'score' }
    assert_equal 0, column.default
    assert_equal :integer, column.type
  end

  def test_rename_column_updates_index
    @connection.add_index :test_columns, :name, name: 'index_test_on_name'
    @connection.rename_column :test_columns, :name, :full_name

    assert_not @connection.index_exists?(:test_columns, :name)
    assert @connection.index_exists?(:test_columns, :full_name)
  end
end
```

### Testing ด้วย RSpec สำหรับ Gems

```ruby
# spec/my_gem/feature_spec.rb
RSpec.describe MyGem::Feature do
  subject(:feature) { described_class.new(options) }
  let(:options)     { { timeout: 30, retry: 3 } }

  describe '#initialize' do
    it 'sets default values' do
      feature = described_class.new

      expect(feature.timeout).to eq(60)  # default
      expect(feature.retry).to eq(1)     # default
    end

    it 'accepts custom options' do
      expect(feature.timeout).to eq(30)
      expect(feature.retry).to eq(3)
    end

    it 'raises ArgumentError for invalid timeout' do
      expect { described_class.new(timeout: -1) }
        .to raise_error(ArgumentError, /timeout must be positive/)
    end
  end

  describe '#perform' do
    context 'when successful' do
      before do
        allow(feature).to receive(:execute).and_return('result')
      end

      it 'returns the result' do
        expect(feature.perform).to eq('result')
      end
    end

    context 'when it fails' do
      before do
        call_count = 0
        allow(feature).to receive(:execute) do
          call_count += 1
          raise NetworkError if call_count < 3
          'result'
        end
      end

      it 'retries on failure' do
        expect(feature.perform).to eq('result')
        expect(feature).to have_received(:execute).exactly(3).times
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1725: Code Review Process

### Code ที่ดีสำหรับ Rails Contribution

```ruby
# ✅ สิ่งที่ Rails maintainers ชอบ

# 1. Follow existing code style
# ดูไฟล์ใกล้เคียงแล้ว match style

# 2. Add/update documentation
# lib/active_record/connection_adapters/abstract/schema_statements.rb

# Adds a new column to the named table.
#
#   add_column(:suppliers, :name, :string, limit: 30)
#   # ALTER TABLE "suppliers" ADD "name" varchar(30)
#
# The +type+ parameter is normally one of the migrations native types,
# which is one of the following:
# <tt>:primary_key</tt>, <tt>:string</tt>, <tt>:text</tt>,
# ...
#
# ==== Options
#
# [+:limit+]
#   The length and precision of a string/decimal type field.
def add_column(table_name, column_name, type, **options)
  # implementation
end

# 3. Tests ครอบคลุมทุก paths
def test_add_column_with_various_types
  [:string, :integer, :float, :decimal, :boolean, :date, :datetime, :text].each do |type|
    @connection.add_column :test_table, :"col_#{type}", type
    column = @connection.columns(:test_table).find { |c| c.name == "col_#{type}" }
    assert_equal type, column.type, "Expected #{type} but got #{column.type}"
  end
end

# 4. ไม่ break backward compatibility
# ถ้าต้อง deprecate สิ่งเก่า ต้องทำ deprecation notice ก่อน
def old_method
  ActiveSupport::Deprecation.warn(
    "`old_method` is deprecated and will be removed in Rails 8.0. " \
    "Use `new_method` instead."
  )
  new_method
end
```

### Checklist ก่อน Submit PR

```markdown
## PR Checklist

### Code
- [ ] Code follows style guide ของ project
- [ ] ไม่มี unnecessary changes (whitespace, formatting)
- [ ] มี tests ที่ cover changes ครบ
- [ ] Tests ผ่านทั้งหมด
- [ ] ไม่ break existing tests
- [ ] ไม่ introduce new deprecation warnings

### Documentation
- [ ] Code comments อัพเดทแล้ว (ถ้ามี)
- [ ] CHANGELOG อัพเดทแล้ว (ถ้า project มี)
- [ ] README อัพเดทแล้ว (ถ้า API เปลี่ยน)

### PR Description
- [ ] อธิบาย what และ why อย่างชัดเจน
- [ ] Link issue ที่เกี่ยวข้อง
- [ ] Screenshots (ถ้ามี UI changes)
- [ ] Breaking changes อธิบายไว้ชัดเจน
```

---

## ขั้นตอนที่ 1726: สร้าง Open Source Gem

### โครงสร้าง Gem พื้นฐาน

```bash
# ใช้ bundler สร้าง gem skeleton
bundle gem my_awesome_gem

# โครงสร้างที่สร้างขึ้น
my_awesome_gem/
├── bin/
│   ├── console    # interactive console
│   └── setup      # setup script
├── lib/
│   ├── my_awesome_gem/
│   │   └── version.rb
│   └── my_awesome_gem.rb
├── spec/ หรือ test/
│   └── my_awesome_gem_spec.rb
├── .gitignore
├── .rspec (ถ้าเลือก RSpec)
├── CHANGELOG.md
├── CODE_OF_CONDUCT.md
├── Gemfile
├── LICENSE.txt
├── my_awesome_gem.gemspec
└── README.md
```

### Gemspec ที่ดี

```ruby
# my_awesome_gem.gemspec
require_relative "lib/my_awesome_gem/version"

Gem::Specification.new do |spec|
  spec.name        = "my_awesome_gem"
  spec.version     = MyAwesomeGem::VERSION
  spec.authors     = ["Your Name"]
  spec.email       = ["your@email.com"]

  spec.summary     = "Short description of your gem"
  spec.description = "Longer description explaining what the gem does and why"
  spec.homepage    = "https://github.com/yourusername/my_awesome_gem"
  spec.license     = "MIT"

  spec.required_ruby_version = ">= 3.1.0"

  spec.metadata = {
    "bug_tracker_uri"   => "https://github.com/yourusername/my_awesome_gem/issues",
    "changelog_uri"     => "https://github.com/yourusername/my_awesome_gem/blob/main/CHANGELOG.md",
    "documentation_uri" => "https://rubydoc.info/gems/my_awesome_gem",
    "homepage_uri"      => spec.homepage,
    "source_code_uri"   => "https://github.com/yourusername/my_awesome_gem",
    "rubygems_mfa_required" => "true"
  }

  # ไฟล์ที่จะ include ใน gem (ไม่รวม spec/test)
  spec.files = Dir.chdir(__dir__) do
    `git ls-files -z`.split("\x0").reject do |f|
      (File.expand_path(f) == __FILE__) ||
        f.start_with?(*%w[bin/ test/ spec/ features/ .git .circleci appveyor Gemfile])
    end
  end

  spec.bindir        = "exe"
  spec.executables   = spec.files.grep(%r{\Aexe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  # Runtime dependencies
  spec.add_dependency "activesupport", ">= 7.0", "< 9.0"
  spec.add_dependency "faraday", "~> 2.0"

  # Development dependencies
  spec.add_development_dependency "rspec", "~> 3.12"
  spec.add_development_dependency "rubocop", "~> 1.60"
  spec.add_development_dependency "rubocop-rspec", "~> 2.26"
  spec.add_development_dependency "vcr", "~> 6.2"
  spec.add_development_dependency "webmock", "~> 3.23"
end
```

---

## ขั้นตอนที่ 1727: ตัวอย่างสร้าง Rails Gem

### สร้าง rails_activity_tracker gem

```ruby
# lib/rails_activity_tracker.rb
require "rails_activity_tracker/version"
require "rails_activity_tracker/railtie"
require "rails_activity_tracker/tracker"
require "rails_activity_tracker/middleware"
require "rails_activity_tracker/configuration"

module RailsActivityTracker
  class Error < StandardError; end

  class << self
    def configuration
      @configuration ||= Configuration.new
    end

    def configure
      yield(configuration)
    end

    def track(user:, action:, resource:, metadata: {})
      Tracker.new(
        user:     user,
        action:   action,
        resource: resource,
        metadata: metadata
      ).record
    end
  end
end

# lib/rails_activity_tracker/configuration.rb
module RailsActivityTracker
  class Configuration
    VALID_STORAGE_BACKENDS = %i[database redis file].freeze

    attr_accessor :storage_backend, :exclude_paths, :user_identifier,
                  :async, :retention_days

    def initialize
      @storage_backend = :database
      @exclude_paths   = ['/health', '/assets']
      @user_identifier = :id
      @async           = true
      @retention_days  = 90
    end

    def validate!
      unless VALID_STORAGE_BACKENDS.include?(@storage_backend)
        raise ConfigurationError,
              "Invalid storage backend: #{@storage_backend}. " \
              "Must be one of: #{VALID_STORAGE_BACKENDS.join(', ')}"
      end
    end
  end
end

# lib/rails_activity_tracker/tracker.rb
module RailsActivityTracker
  class Tracker
    def initialize(user:, action:, resource:, metadata: {})
      @user     = user
      @action   = action
      @resource = resource
      @metadata = metadata
    end

    def record
      activity_data = build_activity_data

      if RailsActivityTracker.configuration.async
        TrackActivityJob.perform_later(activity_data)
      else
        persist(activity_data)
      end
    end

    private

    def build_activity_data
      {
        user_id:       extract_user_id,
        user_type:     @user.class.name,
        action:        @action.to_s,
        resource_type: @resource.class.name,
        resource_id:   @resource.try(:id),
        metadata:      @metadata,
        ip_address:    Current.request&.remote_ip,
        user_agent:    Current.request&.user_agent,
        occurred_at:   Time.current
      }
    end

    def extract_user_id
      identifier = RailsActivityTracker.configuration.user_identifier
      @user.send(identifier)
    end

    def persist(data)
      storage_backend.save(data)
    end

    def storage_backend
      case RailsActivityTracker.configuration.storage_backend
      when :database then DatabaseStorage.new
      when :redis     then RedisStorage.new
      when :file      then FileStorage.new
      end
    end
  end
end

# lib/rails_activity_tracker/railtie.rb
module RailsActivityTracker
  class Railtie < Rails::Railtie
    initializer "rails_activity_tracker.configure_rails_initialization" do |app|
      app.middleware.use RailsActivityTracker::Middleware
    end

    rake_tasks do
      load "rails_activity_tracker/tasks/cleanup.rake"
    end

    generators do
      require "rails_activity_tracker/generators/install_generator"
    end
  end
end

# lib/generators/rails_activity_tracker/install_generator.rb
module RailsActivityTracker
  module Generators
    class InstallGenerator < Rails::Generators::Base
      include Rails::Generators::Migration

      source_root File.expand_path("templates", __dir__)

      def self.next_migration_number(dirname)
        next_migration_number = current_migration_number(dirname) + 1
        ActiveRecord::Migration.next_migration_number(next_migration_number)
      end

      def create_migration_file
        migration_template(
          "create_activity_logs.rb.erb",
          "db/migrate/create_activity_logs.rb"
        )
      end

      def create_initializer
        template "initializer.rb.erb", "config/initializers/rails_activity_tracker.rb"
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1728: README และ Documentation

```markdown
# Rails Activity Tracker

Track user activities in your Rails application with ease.

[![Gem Version](https://badge.fury.io/rb/rails_activity_tracker.svg)](https://badge.fury.io/rb/rails_activity_tracker)
[![CI](https://github.com/yourusername/rails_activity_tracker/workflows/CI/badge.svg)](https://github.com/yourusername/rails_activity_tracker/actions)
[![Coverage](https://codecov.io/gh/yourusername/rails_activity_tracker/branch/main/graph/badge.svg)](https://codecov.io/gh/yourusername/rails_activity_tracker)

## Features

- 🚀 Automatic request tracking via middleware
- 🔄 Async recording with ActiveJob
- 🗄️ Multiple storage backends (database, Redis, file)
- 🔧 Highly configurable
- 📊 Built-in cleanup tasks

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'rails_activity_tracker'
```

Then execute:
```bash
bundle install
rails generate rails_activity_tracker:install
rails db:migrate
```

## Configuration

```ruby
# config/initializers/rails_activity_tracker.rb
RailsActivityTracker.configure do |config|
  config.storage_backend = :database  # :database, :redis, :file
  config.retention_days  = 90
  config.async           = true
  config.exclude_paths   = ['/health', '/assets', '/packs']
  config.user_identifier = :id
end
```

## Usage

### Manual Tracking

```ruby
# ใน Controller
RailsActivityTracker.track(
  user:     current_user,
  action:   :created,
  resource: @post,
  metadata: { ip: request.remote_ip }
)
```

### Model Tracking

```ruby
class Post < ApplicationRecord
  include RailsActivityTracker::Trackable

  track_activity on: [:create, :update, :destroy]
end
```

### Query Activities

```ruby
# ดู activities ทั้งหมด
ActivityLog.all

# ดู activities ของ user
ActivityLog.for_user(current_user)

# ดู activities ของ resource
ActivityLog.for_resource(@post)

# ดู activities ใน time range
ActivityLog.between(1.week.ago, Time.current)
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## License

The gem is available as open source under the [MIT License](LICENSE.txt).
```

---

## ขั้นตอนที่ 1729: RubyGems Publishing

```bash
# 1. สร้าง account ที่ rubygems.org

# 2. ตั้งค่า credentials
gem signin
# Enter your RubyGems.org credentials.
# Email: your@email.com
# Password: xxxxxxxx
# Signed in.

# 3. Build gem
gem build my_awesome_gem.gemspec
# Successfully built RubyGem
# Name: my_awesome_gem
# Version: 0.1.0
# File: my_awesome_gem-0.1.0.gem

# 4. Push ไปยัง RubyGems
gem push my_awesome_gem-0.1.0.gem

# 5. ตรวจสอบ
gem list --remote my_awesome_gem

# 6. ใช้ rake tasks (ที่ bundler สร้างให้)
bundle exec rake release

# ขั้นตอน rake release ทำให้อัตโนมัติ:
# - build gem
# - create git tag
# - push to rubygems
# - push tag to git
```

### Versioning (Semantic Versioning)

```ruby
# lib/my_awesome_gem/version.rb
module MyAwesomeGem
  # MAJOR.MINOR.PATCH
  # MAJOR - breaking changes
  # MINOR - new features, backward compatible
  # PATCH - bug fixes
  VERSION = "1.2.3"

  # Pre-release versions
  # "1.0.0.alpha"
  # "1.0.0.beta.1"
  # "1.0.0.rc.1"
end
```

### CHANGELOG.md

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] - 2024-01-15

### Added
- Redis storage backend support (#45)
- Configuration validation (#48)
- Batch activity recording for performance (#50)

### Changed
- Improved error messages for invalid configurations

### Deprecated
- `track_activity` method without `on:` option will be removed in 2.0

### Fixed
- Fix memory leak in middleware when request fails (#52)
- Correct timezone handling for occurred_at (#54)

## [1.1.0] - 2023-12-01

### Added
- Async recording with ActiveJob
- Cleanup rake tasks for old activities

### Fixed
- Fix N+1 query in `for_user` scope

## [1.0.0] - 2023-11-01

### Added
- Initial release
- Database storage backend
- Manual and automatic tracking
- Basic query interface
```

---

## ขั้นตอนที่ 1730: Maintaining a Gem

### CI/CD สำหรับ Gem

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        ruby-version: ['3.1', '3.2', '3.3']
        rails-version: ['7.0', '7.1', '8.0']

    steps:
    - uses: actions/checkout@v4

    - name: Set up Ruby ${{ matrix.ruby-version }}
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: ${{ matrix.ruby-version }}
        bundler-cache: true

    - name: Run tests
      env:
        RAILS_VERSION: ${{ matrix.rails-version }}
      run: bundle exec rspec

    - name: Upload coverage
      uses: codecov/codecov-action@v3

  rubocop:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: '3.3'
        bundler-cache: true

    - name: Run RuboCop
      run: bundle exec rubocop

  security:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Set up Ruby
      uses: ruby/setup-ruby@v1

    - name: Run bundle audit
      run: |
        gem install bundler-audit
        bundle audit check --update
```

### Gemfile สำหรับ Testing หลาย Rails versions

```ruby
# Gemfile
source "https://rubygems.org"

gemspec

rails_version = ENV.fetch("RAILS_VERSION", "7.1")
gem "rails", "~> #{rails_version}.0"

group :development, :test do
  gem "rspec-rails", "~> 6.0"
  gem "factory_bot_rails", "~> 6.4"
  gem "faker", "~> 3.2"
  gem "simplecov", "~> 0.22", require: false
  gem "rubocop", "~> 1.60", require: false
  gem "rubocop-rails", "~> 2.23", require: false
  gem "rubocop-rspec", "~> 2.26", require: false
end

group :test do
  gem "database_cleaner-active_record"
  gem "shoulda-matchers", "~> 5.3"
  gem "vcr", "~> 6.2"
  gem "webmock", "~> 3.23"
end
```

---

## ขั้นตอนที่ 1731: Contributing Guidelines

```markdown
# Contributing to Rails Activity Tracker

We love your input! We want to make contributing as easy and transparent as possible.

## Ways to Contribute

- Report bugs
- Propose new features
- Improve documentation
- Fix bugs and submit PRs

## Development Process

1. Fork the repo and create your branch from `main`
2. Make your changes
3. Add tests if you've added code
4. Ensure the test suite passes: `bundle exec rspec`
5. Ensure your code passes RuboCop: `bundle exec rubocop`
6. Issue that pull request!

## Code Style

We use RuboCop with our custom configuration (see `.rubocop.yml`).
Run `bundle exec rubocop -a` to auto-correct many issues.

## Testing

```bash
# Run all tests
bundle exec rspec

# Run specific test file
bundle exec rspec spec/rails_activity_tracker/tracker_spec.rb

# Run with coverage
COVERAGE=true bundle exec rspec
```

## Reporting Bugs

Please include:
- Ruby version
- Rails version
- gem version
- Steps to reproduce
- Expected behavior
- Actual behavior
- Stack trace (if applicable)

## License

By contributing, you agree that your contributions will be licensed under MIT License.
```

---

## ขั้นตอนที่ 1732: Code of Conduct

```markdown
# Contributor Covenant Code of Conduct

## Our Pledge

We as members, contributors, and leaders pledge to make participation in our
community a harassment-free experience for everyone.

## Our Standards

Examples of behavior that contributes to a positive environment:

* Using welcoming and inclusive language
* Being respectful of differing viewpoints
* Gracefully accepting constructive criticism
* Focusing on what is best for the community

Examples of unacceptable behavior:

* Harassment of any kind
* Trolling, insulting/derogatory comments
* Publishing others' private information

## Enforcement

Project maintainers are responsible for clarifying standards and will take
appropriate corrective action in response to any behavior they deem inappropriate.
```

---

## ขั้นตอนที่ 1733: Advanced Gem Patterns

### Lazy Loading

```ruby
# lib/my_gem.rb
module MyGem
  # ไม่ require ทันที - lazy load เมื่อจำเป็น
  autoload :Feature,    'my_gem/feature'
  autoload :Middleware, 'my_gem/middleware'
  autoload :Railtie,    'my_gem/railtie'

  # require เฉพาะสิ่งที่จำเป็นเสมอ
  require 'my_gem/version'
  require 'my_gem/configuration'
end
```

### DSL Pattern

```ruby
# lib/my_gem/dsl.rb
module MyGem
  module DSL
    def self.included(base)
      base.extend(ClassMethods)
    end

    module ClassMethods
      def track_activity(**options)
        include InstanceMethods
        class_attribute :tracking_options
        self.tracking_options = options

        # Define callbacks
        after_create  { track(:created)  } if options[:on]&.include?(:create)
        after_update  { track(:updated)  } if options[:on]&.include?(:update)
        before_destroy { track(:deleted) } if options[:on]&.include?(:destroy)
      end
    end

    module InstanceMethods
      private

      def track(action)
        MyGem.track(
          user:     try(:current_user) || Current.user,
          action:   action,
          resource: self,
          metadata: self.class.tracking_options[:metadata]&.call(self) || {}
        )
      end
    end
  end
end

# การใช้งาน
class Article < ApplicationRecord
  include MyGem::DSL

  track_activity on: [:create, :update, :destroy],
                 metadata: ->(record) { { title: record.title } }
end
```

---

## ขั้นตอนที่ 1734: RuboCop Configuration

```yaml
# .rubocop.yml
require:
  - rubocop-rails
  - rubocop-rspec

AllCops:
  TargetRubyVersion: 3.1
  TargetRailsVersion: 7.0
  NewCops: enable
  Exclude:
    - 'db/**/*'
    - 'bin/**/*'
    - 'vendor/**/*'

Style/Documentation:
  Enabled: false

Metrics/BlockLength:
  AllowedMethods: ['describe', 'context', 'it', 'shared_examples']

Metrics/MethodLength:
  Max: 20

Metrics/ClassLength:
  Max: 200

Naming/MethodParameterName:
  MinNameLength: 2

RSpec/ExampleLength:
  Max: 20

RSpec/MultipleExpectations:
  Max: 5
```

---

## ขั้นตอนที่ 1735: Security Considerations สำหรับ Gems

```ruby
# lib/my_gem/security.rb

module MyGem
  module Security
    # 1. ไม่เก็บ sensitive data ใน logs
    def sanitize_for_logging(data)
      sensitive_keys = %w[password password_confirmation token secret credit_card]
      data.transform_values.with_index do |value, index|
        key = data.keys[index].to_s.downcase
        if sensitive_keys.any? { |s| key.include?(s) }
          '[FILTERED]'
        else
          value
        end
      end
    end

    # 2. SQL Injection prevention
    def safe_query(table, conditions)
      # ✅ ใช้ parameterized queries
      ActiveRecord::Base.connection.exec_query(
        "SELECT * FROM #{ActiveRecord::Base.connection.quote_table_name(table)} WHERE id = $1",
        'SQL',
        [conditions[:id]]
      )
    end

    # 3. ระวัง mass assignment
    ALLOWED_PARAMS = %i[name email role].freeze

    def safe_update(params)
      filtered = params.slice(*ALLOWED_PARAMS)
      update!(filtered)
    end
  end
end
```

---

## ขั้นตอนที่ 1736: Gem Documentation ด้วย YARD

```ruby
# lib/my_gem/tracker.rb

module MyGem
  # Tracks user activities in the application.
  #
  # @example Basic usage
  #   tracker = MyGem::Tracker.new(user: current_user, action: :login)
  #   tracker.record
  #
  # @example With metadata
  #   MyGem::Tracker.new(
  #     user:     current_user,
  #     action:   :purchase,
  #     resource: @order,
  #     metadata: { amount: @order.total }
  #   ).record
  class Tracker
    # @param user [Object] The user performing the action
    # @param action [Symbol, String] The action being performed
    # @param resource [Object, nil] The resource being acted upon
    # @param metadata [Hash] Additional information about the action
    def initialize(user:, action:, resource: nil, metadata: {})
      @user     = user
      @action   = action
      @resource = resource
      @metadata = metadata
    end

    # Records the activity.
    #
    # @return [ActivityLog, nil] The created activity log, or nil if async
    # @raise [MyGem::Error] If the activity cannot be recorded
    def record
      # ...
    end

    private

    # @return [Integer] The user's identifier
    def extract_user_id
      # ...
    end
  end
end
```

```bash
# สร้าง documentation
gem install yard
yard doc lib/

# ดู documentation ใน browser
yard server
# เปิด http://localhost:8808
```

---

## ขั้นตอนที่ 1737: Handling Deprecations

```ruby
# lib/my_gem/deprecator.rb
module MyGem
  module Deprecation
    def self.warn(message, caller_location = nil)
      location = caller_location || caller_locations(1, 1).first

      full_message = "[DEPRECATION] #{message}\n"
      full_message += "  Called from: #{location.path}:#{location.lineno}"

      case Rails.env.to_sym
      when :development, :test
        raise DeprecatedError, full_message if strict_mode?
        Kernel.warn full_message
      when :production
        Rails.logger.warn full_message
      end
    end

    def self.strict_mode?
      ENV.fetch('MY_GEM_STRICT_DEPRECATIONS', 'false') == 'true'
    end
  end
end

# การใช้งาน deprecation
class Tracker
  def track(user, action)  # deprecated signature
    MyGem::Deprecation.warn(
      "`Tracker#track(user, action)` is deprecated. " \
      "Use `Tracker#record(user:, action:)` instead. " \
      "This will be removed in version 3.0.",
      caller_locations(1, 1).first
    )

    record(user: user, action: action)
  end

  def record(user:, action:)  # new signature
    # ...
  end
end
```

---

## ขั้นตอนที่ 1738: Benchmarking Gem Performance

```ruby
# benchmarks/tracker_benchmark.rb
require 'benchmark/ips'
require 'my_gem'

puts "Ruby #{RUBY_VERSION}"
puts "My Gem #{MyGem::VERSION}"
puts

Benchmark.ips do |x|
  x.config(time: 10, warmup: 3)

  user     = OpenStruct.new(id: 1, email: 'test@test.com')
  resource = OpenStruct.new(id: 100, class: OpenStruct.new(name: 'Post'))

  x.report("sync tracking") do
    MyGem.configure { |c| c.async = false }
    MyGem.track(user: user, action: :view, resource: resource)
  end

  x.report("async tracking") do
    MyGem.configure { |c| c.async = true }
    MyGem.track(user: user, action: :view, resource: resource)
  end

  x.report("with metadata") do
    MyGem.track(
      user:     user,
      action:   :view,
      resource: resource,
      metadata: { ip: '127.0.0.1', user_agent: 'Chrome/123' }
    )
  end

  x.compare!
end
```

---

## ขั้นตอนที่ 1739: Community Management

```markdown
# การจัดการ Community

## การตอบ Issues

1. ตอบภายใน 48 ชั่วโมง
2. ขอบคุณผู้รายงาน bug
3. ถามข้อมูลเพิ่มเติมถ้าต้องการ
4. Assign milestone ให้ bug ที่ confirmed

Template ตอบ bug:
---
Thank you for reporting this issue! 🙏

I can reproduce this with:
- Ruby 3.2.0
- Rails 7.1.2

This is definitely a bug. I'll work on a fix for the next patch release (1.2.1).

In the meantime, you can work around it by:
...

---

## การ Review PR

1. Respond ภายใน 1 สัปดาห์
2. Review อย่าง constructive
3. Explain reasoning ของการ request changes
4. ขอบคุณเมื่อ merge

Template review:
---
Thanks for this contribution! The approach looks good overall.

A few things to address before merging:

1. **Tests**: Could you add a test case for when `user` is nil?
2. **Style**: Please run `bundle exec rubocop` - there are a few style violations
3. **Documentation**: The new parameter needs a YARD doc comment

Let me know if you have questions!

---

## Releases

1. Update CHANGELOG.md
2. Bump version ใน version.rb
3. bundle exec rake release
4. Create GitHub Release with notes
5. Tweet/announce ถ้าเป็น significant release
```

---

## ขั้นตอนที่ 1740: แนวปฏิบัติที่ดี

```ruby
# 1. ทำ gem ให้เล็กและ focused
# หลักการ Unix: do one thing well

# ❌ Swiss army knife gem
class SuperGem
  def track_activity; end
  def send_email; end
  def generate_pdf; end
  def process_payment; end
  # มากเกินไป!
end

# ✅ Focused gem
module ActivityTracker
  # ทำแค่ tracking เท่านั้น
end

# 2. ไม่ monkey patch โดยไม่จำเป็น
# ❌ อันตราย
class String
  def to_slug
    # modify built-in class
  end
end

# ✅ ใช้ Module
module MyGem
  module StringExtensions
    def to_slug
      # ...
    end
  end
end

# ให้ผู้ใช้เลือกว่าจะ include หรือไม่
String.include(MyGem::StringExtensions) if defined?(MyGem::StringExtensions)

# 3. Semantic Versioning อย่างเคร่งครัด
# NEVER break API ใน patch/minor versions

# 4. Lock dependencies อย่างเหมาะสม
# Too strict:
spec.add_dependency 'rails', '= 7.1.2'  # ❌

# Too loose:
spec.add_dependency 'rails', '>= 5.0'   # อาจ break กับ Rails 9

# Just right:
spec.add_dependency 'rails', '>= 7.0', '< 9.0'  # ✅

# 5. Document breaking changes อย่างชัดเจน
# BREAKING CHANGE: `track` method renamed to `record`
# Migration guide in CHANGELOG.md
```

---

## แบบฝึกหัด: Contributing to Open Source

### ข้อที่ 1: หา Good First Issue
ไปที่ https://github.com/rails/rails/labels/good%20first%20issue และหา 1 issue ที่คิดว่าทำได้

### ข้อที่ 2: Setup Rails Development Environment
Fork Rails, clone, setup database, และ run tests ให้ผ่าน

### ข้อที่ 3: สร้าง Gem Skeleton

```bash
# สร้าง gem ใหม่: thai_validator
# gem ที่ validate เบอร์โทรศัพท์ไทย, เลขบัตรประชาชน, postal code ไทย

bundle gem thai_validator

# สร้างไฟล์หลัก
# lib/thai_validator.rb
# lib/thai_validator/phone.rb
# lib/thai_validator/id_card.rb
# lib/thai_validator/postal_code.rb
```

### ข้อที่ 4: สร้าง Thai Phone Validator

```ruby
# lib/thai_validator/phone.rb
module ThaiValidator
  class Phone
    # เบอร์โทรไทย: 0X-XXXX-XXXX หรือ 0XXXXXXXXX
    MOBILE_PATTERN  = /\A0[6-9]\d{8}\z/
    LANDLINE_PATTERN = /\A0[2-5]\d{7,8}\z/

    def initialize(number)
      @number = normalize(number)
    end

    def valid?
      mobile? || landline?
    end

    def mobile?
      @number.match?(MOBILE_PATTERN)
    end

    def landline?
      @number.match?(LANDLINE_PATTERN)
    end

    def formatted
      return nil unless valid?
      if mobile?
        "#{@number[0..2]}-#{@number[3..6]}-#{@number[7..9]}"
      else
        "#{@number[0..1]}-#{@number[2..5]}-#{@number[6..]}"
      end
    end

    private

    def normalize(number)
      number.to_s.gsub(/[\s\-\(\)]/, '')
    end
  end
end
```

### ข้อที่ 5: Tests ครบ

```ruby
# spec/thai_validator/phone_spec.rb
RSpec.describe ThaiValidator::Phone do
  describe '#valid?' do
    valid_numbers = [
      '0812345678',   # mobile - dtac
      '0912345678',   # mobile - true
      '0623456789',   # mobile - ais
      '021234567',    # landline - bangkok
      '02-123-4567',  # formatted
      '081 234 5678', # with spaces
    ]

    invalid_numbers = [
      '1234567890',   # ไม่เริ่มด้วย 0
      '051234567',    # prefix ไม่ถูกต้อง
      '081234',       # สั้นเกิน
      'abc',          # ไม่ใช่ตัวเลข
    ]

    valid_numbers.each do |number|
      it "validates #{number} as valid" do
        expect(described_class.new(number).valid?).to be true
      end
    end

    invalid_numbers.each do |number|
      it "validates #{number} as invalid" do
        expect(described_class.new(number).valid?).to be false
      end
    end
  end
end
```

### ข้อที่ 6: ActiveModel Validator Integration

```ruby
# lib/thai_validator/active_model.rb
module ThaiValidator
  class ThaiPhoneValidator < ActiveModel::EachValidator
    def validate_each(record, attribute, value)
      validator = Phone.new(value)
      unless validator.valid?
        record.errors.add(attribute, :invalid_thai_phone,
                          message: options[:message] || "is not a valid Thai phone number")
      end
    end
  end
end

# การใช้งาน
class User < ApplicationRecord
  validates :phone, thai_phone: true
end
```

### ข้อที่ 7: Publish ไปยัง RubyGems

```bash
# Build และ publish
gem build thai_validator.gemspec
gem push thai_validator-0.1.0.gem
```

### ข้อที่ 8-20: แบบฝึกหัดเพิ่มเติม

**ข้อ 8:** เพิ่ม Thai ID Card validator (เลขบัตรประชาชน 13 หลัก)

**ข้อ 9:** เพิ่ม Thai Postal Code validator

**ข้อ 10:** เขียน YARD documentation ครบทุก public methods

**ข้อ 11:** Setup CI/CD ด้วย GitHub Actions

**ข้อ 12:** เพิ่ม multi-version testing (Ruby 3.1, 3.2, 3.3)

**ข้อ 13:** สร้าง CHANGELOG.md ที่ follow Keep a Changelog format

**ข้อ 14:** เพิ่ม code coverage badge ด้วย SimpleCov + Codecov

**ข้อ 15:** เขียน CONTRIBUTING.md ที่ comprehensive

**ข้อ 16:** Release version 0.2.0 พร้อม new features

**ข้อ 17:** ตอบ issue ที่มีคนรายงาน bug (simulate)

**ข้อ 18:** Review PR จาก contributor (simulate)

**ข้อ 19:** Handle deprecation จาก old API ใน version 1.0.0

**ข้อ 20:** สร้าง community ด้วย Discussions, wiki, และ project board

---

## สรุป

การ contribute ไปยัง Open Source:
- **หา issues**: good first issue, help wanted
- **Setup environment**: fork, clone, setup tests
- **Git workflow**: branch, commit messages ที่ดี, PR
- **Tests**: cover ทุก scenarios
- **Code review**: ตอบสนอง feedback อย่าง constructive

การสร้าง Open Source Gem:
- **Design**: focused, well-documented API
- **Testing**: ครบถ้วน, หลาย Ruby/Rails versions
- **Documentation**: README, YARD docs, CHANGELOG
- **Publishing**: RubyGems, semantic versioning
- **Maintenance**: respond to issues, review PRs

จบ Part 80 และจบ Rails Advanced Series!
