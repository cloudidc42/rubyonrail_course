# ตอนที่ 32: Rails Setup และโครงสร้าง (Steps 686-705)

## บทนำ

Ruby on Rails เป็น web framework ที่ถูกออกแบบมาตาม convention over configuration ซึ่งหมายความว่า Rails มีโครงสร้างที่กำหนดไว้แล้ว และเราควรปฏิบัติตาม convention เหล่านั้น ในบทนี้เราจะเรียนรู้วิธีการสร้าง Rails application ใหม่ ทำความเข้าใจโครงสร้าง directory และการ configure ต่างๆ

---

## Step 686: rails new Options

### การสร้าง Rails Application ใหม่

```bash
# สร้าง application แบบ default
rails new myapp

# ดู options ทั้งหมด
rails new --help
```

### Options ที่สำคัญ

#### --api
```bash
# สร้าง API-only application (ไม่มี views, assets)
rails new myapi --api

# เหมาะสำหรับ:
# - REST API backends
# - JSON API สำหรับ React/Vue/Angular frontend
# - Mobile app backends
```

ความแตกต่างของ --api mode:
- ไม่สร้าง views และ assets
- ApplicationController ใช้ ActionController::API แทน ActionController::Base
- Middleware stack เล็กกว่า (ไม่มี cookie, session, flash)
- เร็วกว่า standard app

```ruby
# app/controllers/application_controller.rb ใน API mode
class ApplicationController < ActionController::API
  # ไม่มี protect_from_forgery
  # ไม่มี helper methods สำหรับ views
end
```

#### --database
```bash
# PostgreSQL
rails new myapp --database=postgresql
rails new myapp -d postgresql

# MySQL
rails new myapp --database=mysql
rails new myapp -d mysql

# SQLite (default)
rails new myapp --database=sqlite3

# Oracle
rails new myapp --database=oracle

# SQL Server
rails new myapp --database=sqlserver
```

#### --css
```bash
# ใช้ Tailwind CSS
rails new myapp --css=tailwind

# ใช้ Bootstrap
rails new myapp --css=bootstrap

# ใช้ Bulma
rails new myapp --css=bulma

# ไม่ใช้ CSS framework
rails new myapp --css=sass
```

#### --javascript
```bash
# ใช้ esbuild (แนะนำ)
rails new myapp --javascript=esbuild

# ใช้ Vite
rails new myapp --javascript=vite

# ใช้ webpack
rails new myapp --javascript=webpack

# ไม่ใช้ JS bundler
rails new myapp --javascript=importmap
```

#### --skip-* options
```bash
# ไม่ใช้ Action Mailer
rails new myapp --skip-action-mailer

# ไม่ใช้ Active Job
rails new myapp --skip-action-cable

# ไม่ใช้ ActiveStorage
rails new myapp --skip-active-storage

# ไม่ใช้ Hotwire (Turbo + Stimulus)
rails new myapp --skip-hotwire

# ไม่ใช้ Test framework
rails new myapp --skip-test

# ไม่ใช้ system tests
rails new myapp --skip-system-test

# ไม่ใช้ Bundler
rails new myapp --skip-bundle

# ไม่ใช้ Git
rails new myapp --skip-git

# ไม่ใช้ Docker files
rails new myapp --skip-docker

# รวม options
rails new myapp \
  --database=postgresql \
  --css=tailwind \
  --javascript=esbuild \
  --skip-action-mailer \
  --skip-action-cable
```

---

## Step 687: โครงสร้าง Directory ทั้งหมด

เมื่อสร้าง Rails application ใหม่จะได้โครงสร้างดังนี้:

```
myapp/
├── app/
│   ├── assets/
│   │   ├── config/
│   │   │   └── manifest.js
│   │   ├── images/
│   │   └── stylesheets/
│   │       └── application.css
│   ├── channels/
│   │   └── application_cable/
│   │       ├── channel.rb
│   │       └── connection.rb
│   ├── controllers/
│   │   ├── application_controller.rb
│   │   └── concerns/
│   ├── helpers/
│   │   └── application_helper.rb
│   ├── javascript/
│   │   ├── application.js
│   │   └── controllers/
│   ├── jobs/
│   │   └── application_job.rb
│   ├── mailers/
│   │   └── application_mailer.rb
│   ├── models/
│   │   ├── application_record.rb
│   │   └── concerns/
│   └── views/
│       ├── layouts/
│       │   ├── application.html.erb
│       │   ├── mailer.html.erb
│       │   └── mailer.text.erb
│       └── (controller views)
├── bin/
│   ├── bundle
│   ├── dev
│   ├── rails
│   ├── rake
│   └── setup
├── config/
│   ├── application.rb
│   ├── boot.rb
│   ├── cable.yml
│   ├── credentials.yml.enc
│   ├── database.yml
│   ├── environment.rb
│   ├── environments/
│   │   ├── development.rb
│   │   ├── production.rb
│   │   └── test.rb
│   ├── initializers/
│   │   ├── assets.rb
│   │   ├── content_security_policy.rb
│   │   ├── filter_parameter_logging.rb
│   │   └── permissions_policy.rb
│   ├── locales/
│   │   └── en.yml
│   ├── master.key
│   ├── puma.rb
│   └── routes.rb
├── db/
│   ├── migrate/
│   ├── schema.rb
│   └── seeds.rb
├── lib/
│   ├── assets/
│   └── tasks/
├── log/
│   ├── development.log
│   └── test.log
├── public/
│   ├── 404.html
│   ├── 422.html
│   ├── 500.html
│   └── favicon.ico
├── storage/
├── test/ (หรือ spec/ สำหรับ RSpec)
│   ├── application_system_test_case.rb
│   ├── controllers/
│   ├── fixtures/
│   ├── helpers/
│   ├── integration/
│   ├── mailers/
│   ├── models/
│   ├── system/
│   └── test_helper.rb
├── tmp/
│   ├── cache/
│   ├── pids/
│   └── storage/
├── vendor/
├── .gitignore
├── .ruby-version
├── Dockerfile
├── Gemfile
├── Gemfile.lock
├── Procfile.dev
├── README.md
├── Rakefile
└── config.ru
```

### อธิบาย Directory แต่ละส่วน

#### app/ - แกนหลักของ Application

```
app/
├── assets/       - รูปภาพ, CSS, JavaScript (ผ่าน asset pipeline)
├── channels/     - Action Cable (WebSocket)
├── controllers/  - Controller classes
├── helpers/      - View helper methods
├── javascript/   - JavaScript files (Importmap/Webpacker)
├── jobs/         - Background jobs (Active Job)
├── mailers/      - Email senders (Action Mailer)
├── models/       - Model classes (Active Record)
└── views/        - View templates (ERB, Haml, Slim)
```

#### bin/ - Executable Scripts

```bash
# bin/rails - rails command
bin/rails server
bin/rails console
bin/rails generate model User

# bin/bundle - bundler command
bin/bundle install
bin/bundle update

# bin/setup - app setup script
bin/setup

# bin/dev - development server (Procfile.dev)
bin/dev
```

#### config/ - Configuration Files

อธิบายในส่วนถัดไป

#### db/ - Database Files

```ruby
# db/migrate/ - Migration files
# ชื่อ format: YYYYMMDDHHMMSS_create_users.rb
20240101120000_create_users.rb
20240101130000_add_email_to_users.rb

# db/schema.rb - Current database schema
ActiveRecord::Schema[7.0].define(version: 2024_01_01_120000) do
  create_table "users", force: :cascade do |t|
    t.string "name"
    t.string "email"
    t.timestamps
  end
end

# db/seeds.rb - Seed data
User.create!(name: "Admin", email: "admin@example.com")
```

#### lib/ - Library Code

```ruby
# lib/tasks/ - Custom Rake tasks
# lib/tasks/import.rake
namespace :import do
  desc "Import users from CSV"
  task users: :environment do
    CSV.foreach("users.csv") do |row|
      User.create!(name: row[0], email: row[1])
    end
  end
end

# รัน: rails import:users
```

#### log/ - Log Files

```
log/development.log  - Development log
log/test.log         - Test log
log/production.log   - Production log
```

#### public/ - Static Files

Files ใน public/ สามารถ access ได้ตรงๆ โดยไม่ผ่าน Rails:
- public/favicon.ico → /favicon.ico
- public/robots.txt → /robots.txt
- public/404.html → หน้า error page

#### storage/ - Active Storage

```ruby
# เก็บ uploaded files เมื่อใช้ Local storage
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
```

#### test/ หรือ spec/ - Test Files

```ruby
# test/models/user_test.rb
class UserTest < ActiveSupport::TestCase
  test "should not save user without email" do
    user = User.new
    assert_not user.save
  end
end

# spec/models/user_spec.rb (RSpec)
RSpec.describe User, type: :model do
  it "requires email" do
    user = User.new
    expect(user).not_to be_valid
  end
end
```

---

## Step 688: config/ Directory Deep Dive

### config/application.rb

```ruby
# config/application.rb
require_relative "boot"
require "rails/all"

# Require gems ที่ต้องการ
# require "sprockets/railtie"

Bundler.require(*Rails.groups)

module Myapp
  class Application < Rails::Application
    # Rails version configuration
    config.load_defaults 7.0

    # Time zone
    config.time_zone = "Bangkok"
    # หรือ
    config.time_zone = "Asia/Bangkok"

    # Default locale
    config.i18n.default_locale = :th

    # Available locales
    config.i18n.available_locales = [:th, :en]

    # Autoload paths
    config.autoload_paths << Rails.root.join("lib")

    # Eager load paths
    config.eager_load_paths << Rails.root.join("lib")

    # Active Job queue adapter
    config.active_job.queue_adapter = :sidekiq

    # Log level
    # config.log_level = :debug

    # API only
    # config.api_only = true

    # Middleware
    config.middleware.use Rack::Attack

    # Generators configuration
    config.generators do |g|
      g.test_framework :rspec
      g.fixture_replacement :factory_bot, dir: "spec/factories"
      g.orm :active_record, primary_key_type: :uuid
    end

    # Active Record configuration
    config.active_record.default_timezone = :utc

    # Action Mailer
    config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
  end
end
```

### config/boot.rb

```ruby
# config/boot.rb
ENV["BUNDLE_GEMFILE"] ||= File.expand_path("../Gemfile", __dir__)

require "bundler/setup" # Set up gems listed in the Gemfile.
require "bootsnap/setup" # Speed up boot time by caching expensive operations.
```

### config/environment.rb

```ruby
# config/environment.rb
# Load the Rails application.
require_relative "application"

# Initialize the Rails application.
Rails.application.initialize!
```

### config/routes.rb

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Define routes here
  resources :posts
  root "pages#home"
end
```

### config/puma.rb

```ruby
# config/puma.rb
# Puma web server configuration

# จำนวน threads per worker
max_threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

# Port
port ENV.fetch("PORT") { 3000 }

# Environment
environment ENV.fetch("RAILS_ENV") { "development" }

# PID file
pidfile ENV.fetch("PIDFILE") { "tmp/pids/server.pid" }

# Workers (สำหรับ production)
workers ENV.fetch("WEB_CONCURRENCY") { 2 }

# Preload app ใน workers
preload_app!

# Before fork
before_fork do
  ActiveRecord::Base.connection_pool.disconnect! if defined?(ActiveRecord)
end

# On worker boot
on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end

# Allow puma to be restarted
plugin :tmp_restart
```

---

## Step 689: Environments

### Development Environment

```ruby
# config/environments/development.rb
Rails.application.configure do
  # ไม่ cache code ระหว่าง requests (reload ทุก request)
  config.cache_classes = false

  # Eager load เฉพาะเมื่อ cache เปิด
  config.eager_load = false

  # แสดง error details
  config.consider_all_requests_local = true

  # Action Controller
  config.action_controller.perform_caching = false
  config.action_controller.raise_on_missing_translations = true

  # Caching - ปิดโดย default
  # เปิดได้ด้วย: rails dev:cache
  if Rails.root.join("tmp/caching-dev.txt").exist?
    config.action_controller.perform_caching = true
    config.action_controller.enable_fragment_cache_logging = true
    config.cache_store = :memory_store
    config.public_file_server.headers = {
      "Cache-Control" => "public, max-age=#{2.days.to_i}"
    }
  else
    config.action_controller.perform_caching = false
    config.cache_store = :null_store
  end

  # Email - ไม่ส่ง email จริง
  config.action_mailer.raise_delivery_errors = false
  config.action_mailer.perform_caching = false

  # Logger
  config.log_level = :debug

  # Assets
  config.assets.server = "http://localhost:3035"

  # Annotations
  config.active_record.migration_error = :page_load
  config.active_record.verbose_query_logs = true

  # Raises error for missing translations
  config.i18n.raise_on_missing_translations = true

  # Annotate template rendering
  config.action_view.annotate_rendered_view_with_filenames = true
end
```

### Test Environment

```ruby
# config/environments/test.rb
Rails.application.configure do
  # Code ถูก reload อยู่แล้วระหว่าง test run
  config.cache_classes = true

  # Eager load สำหรับ code coverage
  config.eager_load = ENV["CI"].present?

  # ไม่แสดง error details
  config.consider_all_requests_local = true

  # ไม่ cache
  config.action_controller.perform_caching = false
  config.cache_store = :null_store

  # Raise exception สำหรับ delivery errors
  config.action_mailer.delivery_method = :test
  config.action_mailer.perform_caching = false

  # Logger
  config.log_level = :debug

  # Database cleaner
  config.active_record.maintain_test_schema = true

  # ไม่แสดง SQL logs ยาวๆ
  config.active_record.verbose_query_logs = false
end
```

### Production Environment

```ruby
# config/environments/production.rb
Rails.application.configure do
  # Code ถูก cache - ไม่ reload
  config.cache_classes = true

  # Eager load ทุก code
  config.eager_load = true

  # ไม่แสดง error details ให้ user
  config.consider_all_requests_local = false

  # เปิด caching
  config.action_controller.perform_caching = true

  # Force SSL
  config.force_ssl = true

  # Logger
  config.log_level = :info
  config.log_tags = [:request_id]

  # Cache store - Redis
  config.cache_store = :redis_cache_store, {
    url: ENV["REDIS_URL"],
    expires_in: 90.minutes
  }

  # Email
  config.action_mailer.perform_caching = false

  # Assets
  config.public_file_server.enabled = ENV["RAILS_SERVE_STATIC_FILES"].present?
  config.assets.compile = false

  # Active Storage
  config.active_storage.service = :amazon

  # Active Job
  config.active_job.queue_adapter = :sidekiq

  # Logging
  if ENV["RAILS_LOG_TO_STDOUT"].present?
    logger = ActiveSupport::Logger.new($stdout)
    logger.formatter = config.log_formatter
    config.logger = ActiveSupport::TaggedLogging.new(logger)
  end

  # Health check path
  config.health_check_application = ->(env) {
    [200, { "Content-Type" => "text/html" }, ["OK"]]
  }
end
```

---

## Step 690: database.yml Configuration

### SQLite (Default)

```yaml
# config/database.yml
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: storage/development.sqlite3

test:
  <<: *default
  database: storage/test.sqlite3

production:
  <<: *default
  database: storage/production.sqlite3
```

### PostgreSQL

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: myapp_development
  username: myapp
  password: <%= ENV["MYAPP_DATABASE_PASSWORD"] %>
  host: localhost
  port: 5432

test:
  <<: *default
  database: myapp_test
  username: myapp
  password: <%= ENV["MYAPP_DATABASE_PASSWORD"] %>

production:
  <<: *default
  url: <%= ENV["DATABASE_URL"] %>
  # หรือแบบแยก fields
  database: myapp_production
  username: myapp
  password: <%= ENV["MYAPP_DATABASE_PASSWORD"] %>
  host: <%= ENV["DB_HOST"] %>
  port: <%= ENV["DB_PORT"] || 5432 %>
  # SSL
  sslmode: require
```

### MySQL

```yaml
# config/database.yml
default: &default
  adapter: mysql2
  encoding: utf8mb4
  collation: utf8mb4_unicode_ci
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: root
  password:
  host: localhost
  port: 3306

development:
  <<: *default
  database: myapp_development

test:
  <<: *default
  database: myapp_test

production:
  <<: *default
  database: myapp_production
  username: <%= ENV["DB_USERNAME"] %>
  password: <%= ENV["DB_PASSWORD"] %>
  host: <%= ENV["DB_HOST"] %>
  socket: /var/run/mysqld/mysqld.sock
```

### Connection Pool Configuration

```yaml
# config/database.yml
production:
  adapter: postgresql
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  # ตั้ง pool size ตาม Puma workers * threads
  # ถ้า 3 workers * 5 threads = 15 connections ต่อ server
  # pool: 15
  checkout_timeout: 5      # วินาที รอ connection
  idle_timeout: 300        # วินาที ก่อน idle connection ถูก reclaim
  connect_timeout: 5       # วินาที รอ server connect
```

---

## Step 691: Initializers

Initializers คือ code ที่รันเมื่อ Rails application boot ขึ้นมา

### สร้าง Initializer

```ruby
# config/initializers/app_config.rb
Rails.application.config.app_name = "My Application"
Rails.application.config.admin_email = "admin@example.com"
Rails.application.config.max_upload_size = 10.megabytes
```

### Initializer ที่สำคัญที่มาพร้อม Rails

```ruby
# config/initializers/assets.rb
# Precompile assets
Rails.application.config.assets.version = "1.0"
Rails.application.config.assets.paths << Rails.root.join("node_modules")

# config/initializers/filter_parameter_logging.rb
# ซ่อน sensitive parameters ใน logs
Rails.application.config.filter_parameters += [
  :passw, :secret, :token, :_key, :crypt, :salt, :certificate, :otp, :ssn
]

# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https
  policy.connect_src :self, :https, "http://localhost:3035", "ws://localhost:3035"
end
```

### Custom Initializers

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV["REDIS_URL"] }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV["REDIS_URL"] }
end

# config/initializers/devise.rb (Devise gem)
Devise.setup do |config|
  config.mailer_sender = "noreply@example.com"
  config.secret_key = Rails.application.credentials.devise_secret_key
end

# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.stripe[:secret_key]

# config/initializers/carrierwave.rb
CarrierWave.configure do |config|
  config.storage = :fog
  config.fog_provider = "fog/aws"
  config.fog_credentials = {
    provider:              "AWS",
    aws_access_key_id:     Rails.application.credentials.aws[:access_key_id],
    aws_secret_access_key: Rails.application.credentials.aws[:secret_access_key]
  }
  config.fog_directory = ENV["AWS_BUCKET"]
end
```

### Initializers Load Order

```ruby
# config/application.rb
# Initializers รันตามลำดับ alphabetical
# หากต้องการกำหนดลำดับ:
config.railties_order = [:main_app, :engines, :all]

# หรือใช้ append_after ใน initializer เอง:
# config/initializers/z_last.rb
Rails.application.config.after_initialize do
  # Code นี้รันหลัง initializers ทั้งหมด
end
```

---

## Step 692: Credentials and Secrets Management

### Rails Credentials (Rails 7 way)

```bash
# เปิด credentials editor
EDITOR=nano rails credentials:edit

# สำหรับ environment เฉพาะ
EDITOR=nano rails credentials:edit --environment production
EDITOR=nano rails credentials:edit --environment development
```

### โครงสร้าง credentials.yml.enc

```yaml
# config/credentials.yml.enc (decrypted view)
# หลังจาก decrypt ด้วย master.key

# Database
db_password: mysecretpassword

# External APIs
stripe:
  public_key: pk_live_xxx
  secret_key: sk_live_xxx

aws:
  access_key_id: AKIA...
  secret_access_key: xxx
  region: ap-southeast-1
  bucket: my-bucket

# JWT secret
jwt_secret: mysecretjwtkey

# API keys
sendgrid_api_key: SG.xxx

# Secret key base (auto-generated)
secret_key_base: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### ใช้ Credentials ใน Code

```ruby
# อ่าน credentials
Rails.application.credentials.db_password
# => "mysecretpassword"

# Nested credentials
Rails.application.credentials.stripe[:secret_key]
# => "sk_live_xxx"

Rails.application.credentials.dig(:aws, :access_key_id)
# => "AKIA..."

# ใน config files
config.action_mailer.smtp_settings = {
  password: Rails.application.credentials.sendgrid_api_key
}

# ใน initializers
Stripe.api_key = Rails.application.credentials.stripe[:secret_key]
```

### master.key

```bash
# config/master.key - ไม่ commit ขึ้น Git!
# ต้อง add ใน .gitignore

# .gitignore
config/master.key
config/credentials/*.key
```

### Environment-specific Credentials

```bash
# สร้าง production credentials
EDITOR=nano rails credentials:edit --environment production
# สร้าง: config/credentials/production.yml.enc
# สร้าง: config/credentials/production.key

# ใช้ใน production
# Environment variable: RAILS_MASTER_KEY=xxx
```

---

## Step 693: .env Files กับ dotenv gem

### ติดตั้ง dotenv-rails

```ruby
# Gemfile
group :development, :test do
  gem "dotenv-rails"
end
```

```bash
bundle install
```

### สร้าง .env files

```bash
# .env (shared ระหว่าง environments)
DATABASE_HOST=localhost
REDIS_URL=redis://localhost:6379/0
APP_URL=http://localhost:3000

# .env.development
DATABASE_NAME=myapp_development
LOG_LEVEL=debug

# .env.test
DATABASE_NAME=myapp_test

# .env.production (อย่า commit!)
DATABASE_URL=postgresql://user:pass@host/dbname
SECRET_KEY_BASE=xxxxxxxxxxxx
```

### ใช้ .env ใน code

```ruby
# ใน application code
ENV["DATABASE_HOST"]    # => "localhost"
ENV["REDIS_URL"]        # => "redis://localhost:6379/0"

# config/database.yml
development:
  adapter: postgresql
  database: <%= ENV.fetch("DATABASE_NAME", "myapp_development") %>
  host: <%= ENV.fetch("DATABASE_HOST", "localhost") %>
```

### .gitignore ที่ควรมี

```bash
# .gitignore
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
.env.production
config/master.key
config/credentials/*.key
```

### dotenv vs credentials - เลือกใช้อะไร?

| Feature | dotenv | credentials |
|---------|--------|-------------|
| Encryption | ไม่มี | มี (AES-256-GCM) |
| Git-safe | ไม่ (ถ้า commit) | ใช่ |
| Dev experience | ง่าย | ต้องมี key |
| Team sharing | ยาก | แชร์ key |
| CI/CD | env vars | env var (RAILS_MASTER_KEY) |

แนะนำ: ใช้ credentials สำหรับ production secrets, dotenv สำหรับ development convenience

---

## Step 694: Rails Generators Overview

### List Generators

```bash
# ดู generators ทั้งหมด
rails generate --help
rails g --help

# ดู options ของ generator เฉพาะ
rails g model --help
rails g controller --help
```

### Model Generator

```bash
# สร้าง model
rails g model User name:string email:string age:integer

# สร้างไฟล์:
# app/models/user.rb
# db/migrate/20240101120000_create_users.rb
# test/models/user_test.rb
# test/fixtures/users.yml

# Column types
rails g model Product \
  name:string \
  description:text \
  price:decimal \
  stock:integer \
  active:boolean \
  published_at:datetime \
  image:attachment
```

### Controller Generator

```bash
# สร้าง controller
rails g controller Users index show new create edit update destroy

# สร้างไฟล์:
# app/controllers/users_controller.rb
# app/views/users/index.html.erb
# app/views/users/show.html.erb
# ... (views ตาม actions ที่กำหนด)
# test/controllers/users_controller_test.rb
# app/helpers/users_helper.rb
```

### Scaffold Generator

```bash
# สร้าง CRUD ครบชุด
rails g scaffold Post title:string body:text published:boolean user:references

# สร้างไฟล์:
# app/models/post.rb
# app/controllers/posts_controller.rb
# app/views/posts/ (ทุก views สำหรับ CRUD)
# db/migrate/xxx_create_posts.rb
# test/models/post_test.rb
# test/controllers/posts_controller_test.rb
# app/helpers/posts_helper.rb
# test/system/posts_test.rb
```

### Migration Generator

```bash
# สร้าง migration
rails g migration AddEmailToUsers email:string:uniq

# สร้างไฟล์:
# db/migrate/20240101120000_add_email_to_users.rb

# Content ที่ auto-generate
class AddEmailToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :email, :string
    add_index :users, :email, unique: true
  end
end
```

### Mailer Generator

```bash
rails g mailer UserMailer welcome_email reset_password

# สร้างไฟล์:
# app/mailers/user_mailer.rb
# app/views/user_mailer/welcome_email.html.erb
# app/views/user_mailer/welcome_email.text.erb
# test/mailers/user_mailer_test.rb
```

### Job Generator

```bash
rails g job ProcessPayment

# สร้างไฟล์:
# app/jobs/process_payment_job.rb
# test/jobs/process_payment_job_test.rb
```

### Destroy Generator (ยกเลิกการ generate)

```bash
rails destroy model User
rails d controller Users
rails d scaffold Post
```

---

## Step 695: Gemfile Structure

### โครงสร้าง Gemfile

```ruby
# Gemfile

# Ruby version
ruby "3.2.0"

# Rails core
gem "rails", "~> 7.0.0"

# Database
gem "pg", "~> 1.1"               # PostgreSQL
# gem "mysql2", "~> 0.5"         # MySQL
# gem "sqlite3", "~> 1.4"        # SQLite

# Web server
gem "puma", "~> 5.0"

# CSS
gem "tailwindcss-rails"          # Tailwind

# JavaScript
gem "importmap-rails"            # Importmap
gem "turbo-rails"                # Hotwire Turbo
gem "stimulus-rails"             # Hotwire Stimulus

# Assets
gem "sprockets-rails"
gem "jbuilder"                   # JSON templates

# Authentication
gem "devise"                     # User auth
gem "jwt"                        # JWT tokens

# Authorization
gem "pundit"                     # Policy-based auth

# File uploads
gem "active_storage_validations"
gem "image_processing", "~> 1.2"

# Background jobs
gem "sidekiq"                    # Job processor
gem "redis", ">= 4.0.1"         # Job queue

# Search
gem "pg_search"                  # PostgreSQL full-text search
gem "ransack"                    # Search/sort

# Pagination
gem "kaminari"                   # Pagination
# gem "pagy"                     # Fast pagination

# API
gem "rack-cors"                  # CORS
gem "jsonapi-serializer"         # JSON API serialization

# Money
gem "money-rails"

# Misc
gem "bootsnap", require: false   # Speed up boot
gem "tzinfo-data"                # Timezone data

# Development & Test
group :development, :test do
  gem "debug", platforms: %i[mri mingw x64_mingw]
  gem "rspec-rails"              # Testing
  gem "factory_bot_rails"       # Test factories
  gem "faker"                    # Fake data
  gem "dotenv-rails"             # Environment variables
end

group :development do
  gem "web-console"              # Console in browser
  gem "rack-mini-profiler"       # Profiling
  gem "bullet"                   # N+1 detection
  gem "rubocop-rails"            # Linting
  gem "rubocop-rspec"
  gem "annotate"                 # Annotate models
  gem "letter_opener"            # Email preview
  gem "pry-rails"                # Better console
end

group :test do
  gem "capybara"                 # System tests
  gem "selenium-webdriver"
  gem "webmock"                  # HTTP request mocking
  gem "vcr"                      # Record HTTP interactions
  gem "shoulda-matchers"         # Additional matchers
  gem "database_cleaner-active_record"
  gem "simplecov"                # Code coverage
end

group :production do
  gem "sentry-rails"             # Error tracking
  gem "lograge"                  # Better logging
end
```

### Gemfile Syntax

```ruby
# เลือก version
gem "rails", "7.0.4"           # Exact version
gem "rails", "~> 7.0.4"       # >=7.0.4, <7.1
gem "rails", ">= 7.0"         # 7.0 ขึ้นไป
gem "rails", "~> 7.0", ">= 7.0.4"  # รวมกัน

# Git source
gem "rails", git: "https://github.com/rails/rails.git"
gem "rails", git: "https://github.com/rails/rails.git", branch: "main"
gem "rails", git: "https://github.com/rails/rails.git", tag: "v7.0.4"

# Local path
gem "mygem", path: "../mygem"

# Platform specific
gem "tzinfo-data", platforms: %i[mingw mswin x64_mingw jruby]

# Require false
gem "bootsnap", require: false  # ต้อง require เองใน code
```

---

## Step 696-705: Additional Configuration Topics

### config/cable.yml (Action Cable)

```yaml
# config/cable.yml
development:
  adapter: async

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: myapp_production
```

### config/storage.yml (Active Storage)

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: ap-southeast-1
  bucket: <%= Rails.application.credentials.dig(:aws, :bucket) %>

google:
  service: GCS
  project: your-gcs-project
  credentials: <%= Rails.root.join("path/to/gcs.keyfile") %>
  bucket: your-gcs-bucket

azure:
  service: AzureStorage
  storage_account_name: your_account_name
  storage_access_key: <%= Rails.application.credentials.dig(:azure_storage, :storage_access_key) %>
  container: your-container-name
```

### config/locales/th.yml (Internationalization)

```yaml
# config/locales/th.yml
th:
  hello: "สวัสดี"
  
  activerecord:
    models:
      user: "ผู้ใช้"
      post: "บทความ"
    attributes:
      user:
        name: "ชื่อ"
        email: "อีเมล"
        password: "รหัสผ่าน"
    errors:
      messages:
        blank: "ไม่สามารถเว้นว่างได้"
        taken: "ถูกใช้งานแล้ว"
        too_short: "สั้นเกินไป (ขั้นต่ำ %{count} ตัวอักษร)"
        too_long: "ยาวเกินไป (สูงสุด %{count} ตัวอักษร)"
  
  date:
    formats:
      default: "%d/%m/%Y"
      short: "%d %b"
      long: "%d %B %Y"
    abbr_month_names: [~, ม.ค., ก.พ., มี.ค., เม.ย., พ.ค., มิ.ย., ก.ค., ส.ค., ก.ย., ต.ค., พ.ย., ธ.ค.]
    month_names: [~, มกราคม, กุมภาพันธ์, มีนาคม, เมษายน, พฤษภาคม, มิถุนายน, กรกฎาคม, สิงหาคม, กันยายน, ตุลาคม, พฤศจิกายน, ธันวาคม]
  
  time:
    formats:
      default: "%a, %d %b %Y %H:%M:%S %z"
      short: "%d %b %H:%M"
      long: "%d %B %Y %H:%M"
  
  number:
    currency:
      format:
        unit: "฿"
        precision: 2
        separator: "."
        delimiter: ","
        format: "%u%n"
```

### Rakefile

```ruby
# Rakefile
require_relative "config/application"
Rails.application.load_tasks

# Custom tasks จะ autoload จาก lib/tasks/
```

### config.ru (Rack Entry Point)

```ruby
# config.ru
require_relative "config/environment"

run Rails.application
Rails.application.load_server
```

### bin/setup

```ruby
#!/usr/bin/env ruby
require "fileutils"

# path to your application root
APP_ROOT = File.expand_path("..", __dir__)

def system!(*args)
  system(*args, exception: true)
end

FileUtils.chdir APP_ROOT do
  # Install gems
  puts "== Installing dependencies =="
  system! "gem install bundler --conservative"
  system("bundle check") || system!("bundle install")

  # Setup database
  puts "\n== Preparing database =="
  system! "bin/rails db:prepare"

  # Remove old logs and tempfiles
  puts "\n== Removing old logs and tempfiles =="
  system! "bin/rails log:clear tmp:clear"

  # Restart app server
  puts "\n== Restarting application server =="
  system! "bin/rails restart"
end
```

---

## แบบฝึกหัด (Steps 696-705)

### แบบฝึกหัดที่ 1
สร้าง Rails application ชื่อ `blog_app` ที่ใช้ PostgreSQL, Tailwind CSS และ esbuild

**เฉลย:**
```bash
rails new blog_app \
  --database=postgresql \
  --css=tailwind \
  --javascript=esbuild
```

### แบบฝึกหัดที่ 2
สร้าง API-only Rails application ชื่อ `blog_api`

**เฉลย:**
```bash
rails new blog_api --api --database=postgresql
```

### แบบฝึกหัดที่ 3
ใน `config/application.rb` ตั้งค่า timezone เป็น Bangkok และ default locale เป็น Thai

**เฉลย:**
```ruby
# config/application.rb
config.time_zone = "Bangkok"
config.i18n.default_locale = :th
```

### แบบฝึกหัดที่ 4
สร้าง initializer ชื่อ `stripe.rb` ที่ตั้งค่า Stripe API key จาก credentials

**เฉลย:**
```ruby
# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.stripe[:secret_key]
```

### แบบฝึกหัดที่ 5
เพิ่ม credentials สำหรับ AWS โดยมี access_key_id และ secret_access_key

**เฉลย:**
```bash
EDITOR=nano rails credentials:edit
# เพิ่ม:
# aws:
#   access_key_id: AKIA...
#   secret_access_key: xxx
```

### แบบฝึกหัดที่ 6
configure `database.yml` สำหรับ PostgreSQL ที่อ่าน password จาก environment variable

**เฉลย:**
```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: myapp_development
  username: <%= ENV["DB_USERNAME"] %>
  password: <%= ENV["DB_PASSWORD"] %>
  host: localhost
```

### แบบฝึกหัดที่ 7
สร้าง `.env` file สำหรับ development environment และ configure dotenv gem

**เฉลย:**
```ruby
# Gemfile
group :development, :test do
  gem "dotenv-rails"
end
```

```bash
# .env.development
DATABASE_NAME=myapp_development
DATABASE_HOST=localhost
REDIS_URL=redis://localhost:6379/0
```

### แบบฝึกหัดที่ 8
สร้าง scaffold สำหรับ `Article` ที่มี title (string), content (text), published (boolean)

**เฉลย:**
```bash
rails g scaffold Article title:string content:text published:boolean
rails db:migrate
```

### แบบฝึกหัดที่ 9
เพิ่ม gem `rack-mini-profiler` เฉพาะ development group

**เฉลย:**
```ruby
# Gemfile
group :development do
  gem "rack-mini-profiler"
end
```

### แบบฝึกหัดที่ 10
ใน `config/environments/production.rb` เปิด force_ssl และตั้ง cache store เป็น Redis

**เฉลย:**
```ruby
# config/environments/production.rb
config.force_ssl = true
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  expires_in: 90.minutes
}
```

### แบบฝึกหัดที่ 11
สร้าง custom rake task ที่ print จำนวน users ทั้งหมดในฐานข้อมูล

**เฉลย:**
```ruby
# lib/tasks/stats.rake
namespace :stats do
  desc "Show user count"
  task users: :environment do
    puts "Total users: #{User.count}"
  end
end

# รัน: rails stats:users
```

### แบบฝึกหัดที่ 12
configure generators ใน `application.rb` ให้ใช้ RSpec แทน Minitest

**เฉลย:**
```ruby
# config/application.rb
config.generators do |g|
  g.test_framework :rspec
  g.fixture_replacement :factory_bot, dir: "spec/factories"
end
```

### แบบฝึกหัดที่ 13
สร้าง environment-specific credentials สำหรับ production

**เฉลย:**
```bash
EDITOR=nano rails credentials:edit --environment production
# File ที่สร้าง: config/credentials/production.yml.enc
# Key: config/credentials/production.key
```

### แบบฝึกหัดที่ 14
อ่าน credential `stripe.public_key` ใน controller

**เฉลย:**
```ruby
class CheckoutsController < ApplicationController
  def new
    @stripe_public_key = Rails.application.credentials.stripe[:public_key]
  end
end
```

### แบบฝึกหัดที่ 15
เพิ่ม autoload path สำหรับ `lib/` ใน `application.rb`

**เฉลย:**
```ruby
# config/application.rb
config.autoload_paths << Rails.root.join("lib")
config.eager_load_paths << Rails.root.join("lib")
```

### แบบฝึกหัดที่ 16
configure Puma สำหรับ production ที่มี 4 workers และ 5 threads ต่อ worker

**เฉลย:**
```ruby
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 4 }
threads 5, 5
preload_app!
```

### แบบฝึกหัดที่ 17
สร้าง locale file ภาษาไทยที่กำหนด error message สำหรับ `blank` validation

**เฉลย:**
```yaml
# config/locales/th.yml
th:
  activerecord:
    errors:
      messages:
        blank: "ไม่สามารถเว้นว่างได้"
```

### แบบฝึกหัดที่ 18
configure Active Storage ให้ใช้ S3 ใน production

**เฉลย:**
```yaml
# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: ap-southeast-1
  bucket: my-bucket
```

```ruby
# config/environments/production.rb
config.active_storage.service = :amazon
```

### แบบฝึกหัดที่ 19
สร้าง initializer ที่ filter `credit_card_number` ออกจาก logs

**เฉลย:**
```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [:credit_card_number, :cvv]
```

### แบบฝึกหัดที่ 20
configure Action Mailer ใน development ให้ใช้ `letter_opener` gem

**เฉลย:**
```ruby
# Gemfile
group :development do
  gem "letter_opener"
end

# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.perform_deliveries = true
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **rails new options** - --api, --database, --css, --javascript, --skip-*
2. **โครงสร้าง Directory** - ความหมายของแต่ละ folder และ file
3. **config/ directory** - application.rb, environments, routes
4. **Environments** - development, test, production configuration
5. **database.yml** - SQLite, PostgreSQL, MySQL configuration
6. **Initializers** - การ configure gems และ services
7. **Credentials** - การจัดการ secrets ด้วย encryption
8. **dotenv** - Environment variables สำหรับ development
9. **Generators** - scaffold, model, controller, migration
10. **Gemfile** - การจัดการ gems และ groups

Rails ใช้ **Convention over Configuration** ทำให้เราไม่ต้องตั้งค่าหลายอย่าง แต่ต้องทำความเข้าใจ conventions เหล่านั้นก่อน
