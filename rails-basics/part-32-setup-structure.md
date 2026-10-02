# Part 32: Rails Setup and Structure (ขั้นตอนที่ 686-705)

## บทนำ

การตั้งค่า Rails project อย่างถูกต้องตั้งแต่แรกเริ่มเป็นสิ่งสำคัญมาก ในบทนี้เราจะเจาะลึกเกี่ยวกับ options ต่างๆ ของ `rails new`, โครงสร้าง directory, และการ configure application

---

## ขั้นตอนที่ 686: Rails New Options แบบละเอียด

### ตัวเลือก Database

```bash
# SQLite (default) - เหมาะสำหรับ development และ small apps
rails new myapp --database=sqlite3
rails new myapp -d sqlite3

# PostgreSQL - แนะนำสำหรับ production
rails new myapp --database=postgresql
rails new myapp -d postgresql

# MySQL
rails new myapp --database=mysql
rails new myapp -d mysql

# MariaDB
rails new myapp --database=mysql  # ใช้ mysql adapter เหมือนกัน

# Oracle
rails new myapp --database=oracle

# Microsoft SQL Server
rails new myapp --database=sqlserver
```

### ตัวเลือก JavaScript

```bash
# Import Maps (default ใน Rails 7) - ไม่ต้องการ bundler
rails new myapp --javascript=importmap

# Bun - JavaScript runtime ใหม่ที่เร็วมาก
rails new myapp --javascript=bun

# Webpack (legacy)
rails new myapp --javascript=webpack

# ESBuild - เร็ว, simple
rails new myapp --javascript=esbuild

# Rollup - สำหรับ library
rails new myapp --javascript=rollup

# Vite (ผ่าน vite_ruby gem)
rails new myapp
# แล้ว add vite_ruby ใน Gemfile
```

### ตัวเลือก CSS

```bash
# Tailwind CSS (แนะนำ)
rails new myapp --css=tailwind

# Bootstrap
rails new myapp --css=bootstrap

# Bulma
rails new myapp --css=bulma

# PostCSS
rails new myapp --css=postcss

# Sass/SCSS
rails new myapp --css=sass
```

### ตัวเลือก Skip

```bash
# Skip Action Mailer
rails new myapp --skip-action-mailer

# Skip Active Storage (file uploads)
rails new myapp --skip-active-storage

# Skip Action Cable (WebSockets)
rails new myapp --skip-action-cable

# Skip Hotwire (Turbo + Stimulus)
rails new myapp --skip-hotwire

# Skip JavaScript entirely
rails new myapp --skip-javascript

# Skip Test framework
rails new myapp --skip-test

# Skip Gemfile bundle install
rails new myapp --skip-bundle

# Skip git initialization
rails new myapp --skip-git

# ข้ามหลายอย่างพร้อมกัน
rails new myapp \
  --skip-action-mailer \
  --skip-action-cable \
  --skip-active-storage \
  --database=postgresql

# API Mode (skip views, assets, cookies)
rails new myapp --api --database=postgresql
```

### ตัวเลือก Advanced

```bash
# สร้างจาก template
rails new myapp --template=https://raw.githubusercontent.com/user/template/master/template.rb

# Force overwrite existing files
rails new myapp --force

# ไม่ run bundle install
rails new myapp --skip-bundle

# Minimal app (ขั้นต่ำสุด)
rails new myapp --minimal

# Pretty-print output
rails new myapp --pretend  # แสดงว่าจะทำอะไรโดยไม่ทำจริง
```

### Template สำหรับสร้าง App

```ruby
# template.rb - Rails Application Template
# รัน: rails new myapp --template=template.rb

# เพิ่ม gems
gem "devise"
gem "pundit"
gem "pagy"
gem "sidekiq"
gem "redis"
gem "dotenv-rails", groups: [:development, :test]

gem_group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
  gem "pry-rails"
end

gem_group :development do
  gem "better_errors"
  gem "binding_of_caller"
  gem "annotate"
  gem "bullet"
end

gem_group :test do
  gem "capybara"
  gem "shoulda-matchers"
  gem "webmock"
end

# รัน bundle install
after_bundle do
  # Generate Devise
  generate "devise:install"
  generate "devise", "User"

  # Generate RSpec
  generate "rspec:install"

  # Create database
  rails_command "db:create"
  rails_command "db:migrate"

  # Initialize git
  git :init
  git add: "."
  git commit: %Q{ -m "Initial commit" }

  say "Application created successfully!", :green
end
```

---

## ขั้นตอนที่ 687: Directory Structure แบบละเอียด

### app/ Directory

```
app/
├── assets/
│   ├── config/
│   │   └── manifest.js          # Asset manifest
│   ├── images/                  # รูปภาพ
│   └── stylesheets/
│       └── application.css      # Main CSS file
│
├── channels/
│   ├── application_cable/
│   │   ├── channel.rb           # Base channel class
│   │   └── connection.rb        # WebSocket connection auth
│   └── chat_channel.rb          # Example channel
│
├── controllers/
│   ├── concerns/                # Shared controller modules
│   │   ├── authenticatable.rb
│   │   └── paginatable.rb
│   ├── application_controller.rb
│   └── articles_controller.rb
│
├── helpers/
│   ├── application_helper.rb   # Global helpers
│   └── articles_helper.rb      # Article-specific helpers
│
├── javascript/
│   ├── controllers/             # Stimulus controllers
│   │   ├── index.js             # Controller registry
│   │   └── dropdown_controller.js
│   ├── channels/                # Action Cable channels
│   └── application.js           # JS entry point
│
├── jobs/
│   ├── application_job.rb       # Base job class
│   └── send_email_job.rb
│
├── mailers/
│   ├── application_mailer.rb    # Base mailer class
│   └── user_mailer.rb
│
├── models/
│   ├── concerns/                # Shared model modules
│   │   ├── sluggable.rb
│   │   └── soft_deletable.rb
│   ├── application_record.rb   # Base model class
│   └── article.rb
│
└── views/
    ├── articles/
    │   ├── index.html.erb
    │   ├── show.html.erb
    │   ├── new.html.erb
    │   ├── edit.html.erb
    │   └── _form.html.erb      # Partial (เริ่มด้วย _)
    ├── layouts/
    │   ├── application.html.erb # Default layout
    │   ├── admin.html.erb       # Admin layout
    │   └── mailer.html.erb      # Email layout
    └── shared/
        ├── _navigation.html.erb
        └── _flash_messages.html.erb
```

### config/ Directory

```
config/
├── environments/
│   ├── development.rb           # Development settings
│   ├── test.rb                  # Test settings
│   └── production.rb            # Production settings
│
├── initializers/
│   ├── assets.rb                # Asset configuration
│   ├── devise.rb                # Devise configuration
│   ├── inflections.rb           # Custom inflections
│   ├── sidekiq.rb               # Sidekiq configuration
│   └── cors.rb                  # CORS configuration
│
├── locales/
│   ├── en.yml                   # English translations
│   └── th.yml                   # Thai translations
│
├── application.rb               # Main application configuration
├── boot.rb                      # Bundler and path setup
├── cable.yml                    # Action Cable configuration
├── credentials.yml.enc          # Encrypted credentials
├── database.yml                 # Database configuration
├── environment.rb               # Load application
├── importmap.rb                 # Import map configuration
├── master.key                   # Encryption key (ห้าม commit!)
├── puma.rb                      # Puma web server config
├── routes.rb                    # URL routing
├── storage.yml                  # Active Storage config
└── tailwind.config.js           # Tailwind CSS config (ถ้าใช้)
```

### db/ Directory

```
db/
├── migrate/
│   ├── 20240101000001_create_users.rb
│   ├── 20240101000002_create_articles.rb
│   └── 20240115_add_slug_to_articles.rb
├── schema.rb                    # Current database schema
└── seeds.rb                     # Seed data
```

### test/ หรือ spec/ Directory

```
spec/                            # ถ้าใช้ RSpec
├── controllers/
│   └── articles_controller_spec.rb
├── factories/
│   ├── users.rb
│   └── articles.rb
├── models/
│   └── article_spec.rb
├── requests/
│   └── articles_spec.rb
├── support/
│   ├── factory_bot.rb
│   ├── shoulda_matchers.rb
│   └── database_cleaner.rb
├── system/
│   └── articles_spec.rb
└── rails_helper.rb
```

---

## ขั้นตอนที่ 688: config/ Directory Deep Dive

### config/application.rb

```ruby
# config/application.rb
require_relative "boot"
require "rails/all"

Bundler.require(*Rails.groups)

module MyBlogApp
  class Application < Rails::Application
    # Rails version defaults
    config.load_defaults 7.1

    # ===== Time Zone =====
    config.time_zone = "Bangkok"
    # ทั้งหมดที่ Active Record บันทึกจะเป็น UTC
    # config.active_record.default_timezone = :local  # ถ้าต้องการ local time

    # ===== Internationalization =====
    config.i18n.default_locale = :th
    config.i18n.available_locales = [:th, :en]
    config.i18n.fallbacks = [:en]  # fallback ถ้าไม่มี translation

    # ===== Autoloading =====
    # เพิ่ม custom paths สำหรับ autoloading
    config.autoload_paths += [
      Rails.root.join("lib"),
      Rails.root.join("app/services"),
      Rails.root.join("app/forms"),
      Rails.root.join("app/presenters"),
      Rails.root.join("app/queries"),
      Rails.root.join("app/decorators")
    ]

    # ===== Generators =====
    config.generators do |g|
      g.orm :active_record, primary_key_type: :uuid  # UUID as primary key
      g.test_framework :rspec,
        fixtures: false,
        view_specs: false,
        helper_specs: false,
        routing_specs: false
      g.fixture_replacement :factory_bot, dir: "spec/factories"
      g.stylesheets false
      g.helper false
      g.jbuilder false
    end

    # ===== Middleware =====
    config.middleware.use Rack::Deflater  # Gzip compression
    config.middleware.insert_before 0, Rack::Cors do
      allow do
        origins "*"
        resource "*",
          headers: :any,
          methods: [:get, :post, :put, :patch, :delete, :options, :head]
      end
    end

    # ===== Logging =====
    config.log_formatter = ::Logger::Formatter.new
    config.colorize_logging = true

    # ===== Active Job =====
    config.active_job.queue_adapter = :sidekiq

    # ===== Action Mailer =====
    config.action_mailer.default_url_options = { host: "localhost", port: 3000 }

    # ===== Active Storage =====
    config.active_storage.variant_processor = :vips  # ใช้ libvips แทน ImageMagick
  end
end
```

### config/boot.rb

```ruby
# config/boot.rb
ENV["BUNDLE_GEMFILE"] ||= File.expand_path("../Gemfile", __dir__)

require "bundler/setup"  # Set up gems listed in the Gemfile.
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

---

## ขั้นตอนที่ 689: Environments

### Development Environment

```ruby
# config/environments/development.rb
Rails.application.configure do
  # Code reload
  config.enable_reloading = true

  # Eager loading
  config.eager_load = false

  # Error pages
  config.consider_all_requests_local = true

  # Caching
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

  # Mailer
  config.action_mailer.raise_delivery_errors = false
  config.action_mailer.perform_caching = false
  config.action_mailer.delivery_method = :letter_opener  # ถ้าใช้ gem letter_opener
  # หรือ
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.smtp_settings = { address: "localhost", port: 1025 }

  # Active Record
  config.active_record.migration_error = :page_load
  config.active_record.verbose_query_logs = true

  # Assets
  config.assets.quiet = true

  # Logging
  config.log_level = :debug
  config.log_tags = [:request_id]

  # Active Support
  config.active_support.deprecation = :log
  config.active_support.disallowed_deprecation = :raise
  config.active_support.disallowed_deprecation_warnings = []

  # Bullet gem สำหรับ N+1 detection
  config.after_initialize do
    if defined?(Bullet)
      Bullet.enable = true
      Bullet.rails_logger = true
      Bullet.add_footer = true
      Bullet.alert = false
    end
  end
end
```

### Test Environment

```ruby
# config/environments/test.rb
Rails.application.configure do
  config.enable_reloading = false
  config.eager_load = ENV["CI"].present?

  # Error handling
  config.consider_all_requests_local = true
  config.action_controller.raise_on_open_redirects = true
  config.action_controller.allow_forgery_protection = false

  # Caching
  config.cache_store = :null_store

  # Active Record
  config.active_record.maintain_test_schema = true
  config.active_record.encryption.support_unencrypted_data = true

  # Mailer
  config.action_mailer.perform_deliveries = true
  config.action_mailer.delivery_method = :test
  config.action_mailer.raise_delivery_errors = true

  # Active Support
  config.active_support.deprecation = :stderr
  config.active_support.disallowed_deprecation = :raise
  config.active_support.disallowed_deprecation_warnings = []
end
```

### Production Environment

```ruby
# config/environments/production.rb
Rails.application.configure do
  config.enable_reloading = false
  config.eager_load = true
  config.consider_all_requests_local = false

  # SSL
  config.force_ssl = true

  # Logging
  config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info")
  config.log_tags = [:request_id]
  config.logger = ActiveSupport::Logger.new(STDOUT)
                    .tap { |l| l.formatter = Logger::Formatter.new }
                    .then { |l| ActiveSupport::TaggedLogging.new(l) }

  # Caching
  config.action_controller.perform_caching = true
  config.public_file_server.enabled = ENV["RAILS_SERVE_STATIC_FILES"].present?

  config.cache_store = :redis_cache_store, {
    url: ENV["REDIS_URL"],
    pool_size: ENV.fetch("RAILS_MAX_THREADS", 5),
    pool_timeout: 5
  }

  # Assets
  config.assets.compile = false
  config.assets.js_compressor = :terser

  # Active Storage
  config.active_storage.service = :amazon  # หรือ :google, :azure

  # Mailer
  config.action_mailer.perform_caching = false
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.smtp_settings = {
    address: "smtp.sendgrid.net",
    port: 587,
    domain: ENV["MAIL_DOMAIN"],
    user_name: ENV["SENDGRID_USERNAME"],
    password: ENV["SENDGRID_PASSWORD"],
    authentication: "plain",
    enable_starttls_auto: true
  }

  # Active Support
  config.active_support.report_deprecations = false

  # Health checks
  config.active_record.dump_schema_after_migration = false
end
```

---

## ขั้นตอนที่ 690: database.yml แบบละเอียด

### SQLite Configuration

```yaml
# config/database.yml (SQLite)
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: db/development.sqlite3

test:
  <<: *default
  database: db/test.sqlite3

production:
  <<: *default
  database: db/production.sqlite3
```

### PostgreSQL Configuration

```yaml
# config/database.yml (PostgreSQL)
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: myapp_development
  username: <%= ENV.fetch("DB_USERNAME", "postgres") %>
  password: <%= ENV["DB_PASSWORD"] %>
  host: <%= ENV.fetch("DB_HOST", "localhost") %>
  port: <%= ENV.fetch("DB_PORT", 5432) %>

test:
  <<: *default
  database: myapp_test
  username: <%= ENV.fetch("DB_USERNAME", "postgres") %>
  password: <%= ENV["DB_PASSWORD"] %>
  host: <%= ENV.fetch("DB_HOST", "localhost") %>

production:
  <<: *default
  url: <%= ENV["DATABASE_URL"] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  prepared_statements: false  # สำคัญสำหรับ PgBouncer
```

### MySQL Configuration

```yaml
# config/database.yml (MySQL)
default: &default
  adapter: mysql2
  encoding: utf8mb4
  collation: utf8mb4_unicode_ci
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: <%= ENV.fetch("DB_USERNAME", "root") %>
  password: <%= ENV["DB_PASSWORD"] %>
  host: <%= ENV.fetch("DB_HOST", "127.0.0.1") %>
  port: 3306

development:
  <<: *default
  database: myapp_development

test:
  <<: *default
  database: myapp_test

production:
  <<: *default
  url: <%= ENV["DATABASE_URL"] %>
```

### Multiple Databases (Rails 6+)

```yaml
# config/database.yml (Multiple Databases)
default: &default
  adapter: postgresql
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  primary:
    <<: *default
    database: myapp_development
  primary_replica:
    <<: *default
    database: myapp_development
    replica: true
  analytics:
    <<: *default
    database: myapp_analytics_development
    migrations_paths: db/analytics_migrate

production:
  primary:
    <<: *default
    url: <%= ENV["DATABASE_URL"] %>
  primary_replica:
    <<: *default
    url: <%= ENV["DATABASE_REPLICA_URL"] %>
    replica: true
  analytics:
    <<: *default
    url: <%= ENV["ANALYTICS_DATABASE_URL"] %>
    migrations_paths: db/analytics_migrate
```

---

## ขั้นตอนที่ 691: Initializers

### สร้าง Initializers

```ruby
# config/initializers/devise.rb (auto-generated)
Devise.setup do |config|
  config.mailer_sender = "noreply@myapp.com"
  config.secret_key = Rails.application.credentials.devise_secret_key
  config.stretches = Rails.env.test? ? 1 : 12
  config.reconfirmable = true
  config.expire_all_remember_me_on_sign_out = true
  config.password_length = 8..128
  config.email_regexp = /\A[^@\s]+@[^@\s]+\z/
  config.reset_password_within = 6.hours
  config.sign_out_via = :delete
  config.navigational_formats = ["*/*", :html, :turbo_stream]
end
```

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }

  config.on(:startup) do
    Rails.logger.info "Sidekiq started with #{Sidekiq.options[:concurrency]} workers"
  end
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end
```

```ruby
# config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  # Custom plural forms
  inflect.irregular "person", "people"
  inflect.plural /datum$/i, "data"

  # Custom singular forms
  inflect.singular /data$/i, "datum"

  # Acronyms
  inflect.acronym "API"
  inflect.acronym "URL"
  inflect.acronym "HTML"
  inflect.acronym "JSON"

  # Uncountable words
  inflect.uncountable %w[information equipment]
end
```

```ruby
# config/initializers/assets.rb
Rails.application.config.assets.version = "1.0"
Rails.application.config.assets.paths << Rails.root.join("vendor/assets/images")
Rails.application.config.assets.precompile += %w[
  admin.css
  admin.js
  email.css
]
```

```ruby
# config/initializers/cors.rb (สำหรับ API)
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins Rails.env.production? ? "https://myapp.com" : "*"

    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,
      max_age: 600
  end
end
```

```ruby
# config/initializers/pagy.rb (Pagination)
require "pagy/extras/bootstrap"
require "pagy/extras/items"
require "pagy/extras/overflow"

Pagy::DEFAULT[:items] = 20
Pagy::DEFAULT[:size] = [1, 4, 4, 1]
Pagy::DEFAULT[:overflow] = :last_page
```

```ruby
# config/initializers/money.rb (ถ้าใช้ money-rails)
MoneyRails.configure do |config|
  config.default_currency = :thb
  config.no_cents_if_whole = false
  config.rounding_mode = BigDecimal::ROUND_HALF_UP
end
```

```ruby
# config/initializers/time_formats.rb
# Custom time formats
Time::DATE_FORMATS[:thai_date] = "%d/%m/%Y"
Time::DATE_FORMATS[:thai_datetime] = "%d/%m/%Y %H:%M"
Time::DATE_FORMATS[:thai_time] = "%H:%M"

# ใช้:
# Time.current.to_fs(:thai_date)     # => "15/01/2024"
# Time.current.to_fs(:thai_datetime) # => "15/01/2024 10:30"
```

---

## ขั้นตอนที่ 692: Credentials และ Secrets

### Rails Credentials (Rails 5.2+)

```bash
# แก้ไข credentials
rails credentials:edit

# สำหรับ specific environment
rails credentials:edit --environment production
rails credentials:edit --environment development

# ดู credentials
rails credentials:show
```

```yaml
# ตัวอย่าง credentials (config/credentials.yml.enc)
secret_key_base: abc123...

# Database
database:
  password: my_db_password

# AWS
aws:
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  region: ap-southeast-1
  bucket: myapp-production

# Stripe
stripe:
  public_key: pk_live_...
  secret_key: sk_live_...
  webhook_secret: whsec_...

# SendGrid
sendgrid:
  username: apikey
  api_key: SG.xxx...

# Devise
devise_secret_key: def456...
```

```ruby
# การเรียกใช้ credentials ใน code
Rails.application.credentials.secret_key_base
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.stripe[:secret_key]

# ด้วย dig (safe navigation)
Rails.application.credentials.dig(:aws, :access_key_id)

# ใช้ใน initializer
Aws.config.update({
  credentials: Aws::Credentials.new(
    Rails.application.credentials.dig(:aws, :access_key_id),
    Rails.application.credentials.dig(:aws, :secret_access_key)
  ),
  region: Rails.application.credentials.dig(:aws, :region)
})
```

### Environment-specific Credentials

```bash
# สร้าง production credentials
rails credentials:edit --environment production
# สร้าง config/credentials/production.yml.enc
# และ config/credentials/production.key

# สร้าง development credentials
rails credentials:edit --environment development
```

```yaml
# config/credentials/production.yml.enc
database:
  url: postgres://user:password@production-db.example.com/myapp_production
redis_url: redis://production-redis.example.com:6379
```

---

## ขั้นตอนที่ 693: .env Files

### dotenv-rails gem

```ruby
# Gemfile
gem "dotenv-rails", groups: [:development, :test]
```

```bash
# .env (development)
# ห้าม commit ไฟล์นี้!
DATABASE_URL=postgres://localhost/myapp_development
REDIS_URL=redis://localhost:6379/0
SECRET_KEY_BASE=development_secret_key

# Stripe
STRIPE_PUBLIC_KEY=pk_test_...
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_test_...

# AWS
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=ap-southeast-1
AWS_BUCKET=myapp-development

# Email
SENDGRID_API_KEY=SG.xxx...
MAIL_FROM=noreply@myapp.com
MAIL_DOMAIN=myapp.com

# App Settings
APP_HOST=localhost:3000
ADMIN_EMAIL=admin@myapp.com
```

```bash
# .env.test
DATABASE_URL=postgres://localhost/myapp_test
REDIS_URL=redis://localhost:6379/1  # DB 1 สำหรับ test

# .env.example (เก็บ template ไว้ใน git)
DATABASE_URL=postgres://localhost/myapp_development
REDIS_URL=redis://localhost:6379/0
SECRET_KEY_BASE=
STRIPE_PUBLIC_KEY=
STRIPE_SECRET_KEY=
```

```ruby
# เรียกใช้ ENV variables
# config/database.yml
database:
  url: <%= ENV["DATABASE_URL"] %>

# config/initializers/stripe.rb
Stripe.api_key = ENV["STRIPE_SECRET_KEY"]

# ใน Ruby code
class SomeService
  def initialize
    @api_key = ENV.fetch("SOME_API_KEY") do
      raise "SOME_API_KEY is not set"
    end
  end
end
```

---

## ขั้นตอนที่ 694: Rails Generators Overview

### Generator Types

```bash
# ===== Model Generator =====
rails g model Article title:string body:text status:string user:references

# Column types:
# :string        - VARCHAR (255)
# :text          - TEXT
# :integer       - INT
# :float         - FLOAT
# :decimal       - DECIMAL
# :datetime      - DATETIME
# :time          - TIME
# :date          - DATE
# :boolean       - BOOLEAN
# :binary        - BLOB
# :json          - JSON (PostgreSQL/MySQL)
# :jsonb         - JSONB (PostgreSQL เท่านั้น)
# :uuid          - UUID

# Special modifiers:
# :uniq          - สร้าง unique index
# :index         - สร้าง index
# :references    - foreign key + index

# ตัวอย่าง:
rails g model Product \
  name:string \
  description:text \
  price:decimal{10,2} \
  sku:string:uniq \
  stock:integer \
  active:boolean \
  category:references \
  metadata:jsonb

# ===== Controller Generator =====
rails g controller Articles index show new create edit update destroy

# Options:
rails g controller Articles \
  --skip-routes \    # ไม่สร้าง routes
  --no-helper \      # ไม่สร้าง helper
  --no-assets        # ไม่สร้าง asset files

# ===== Scaffold Generator =====
rails g scaffold Article title:string body:text status:string user:references

# API Scaffold:
rails g scaffold Article title:string body:text --api

# ===== Migration Generator =====
rails g migration AddSlugToArticles slug:string:uniq
rails g migration RemoveStatusFromArticles status:string
rails g migration CreateJoinTableArticlesTags articles tags
rails g migration AddIndexToUsersEmail
rails g migration ChangeArticlesBodyToText

# ===== Resource Generator =====
# สร้าง model + controller + routes (ไม่มี views)
rails g resource Article title:string body:text

# ===== Scaffold Controller Generator =====
# สร้าง controller + views (ไม่มี model + migration)
rails g scaffold_controller Article title:string body:text
```

### Custom Generator

```ruby
# lib/generators/service/service_generator.rb
class ServiceGenerator < Rails::Generators::NamedBase
  source_root File.expand_path("templates", __dir__)

  def create_service_file
    template "service.rb.tt", "app/services/#{file_name}_service.rb"
  end

  def create_spec_file
    template "service_spec.rb.tt", "spec/services/#{file_name}_service_spec.rb"
  end
end

# lib/generators/service/templates/service.rb.tt
class <%= class_name %>Service
  def initialize(params = {})
    @params = params
  end

  def call
    # TODO: Implement service logic
  end

  private

  attr_reader :params
end

# รัน:
# rails g service UserRegistration
# สร้าง:
# app/services/user_registration_service.rb
# spec/services/user_registration_service_spec.rb
```

---

## ขั้นตอนที่ 695: config/routes.rb แบบเบื้องต้น

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Root
  root "home#index"

  # Resources
  resources :articles
  resources :users, only: [:index, :show]
  resources :categories, except: [:destroy]

  # Devise
  devise_for :users

  # Admin namespace
  namespace :admin do
    root "dashboard#index"
    resources :articles
    resources :users
  end

  # API namespace
  namespace :api do
    namespace :v1 do
      resources :articles, only: [:index, :show, :create, :update, :destroy]
    end
  end

  # Custom routes
  get "/about", to: "pages#about", as: :about
  get "/contact", to: "pages#contact", as: :contact

  # Health check
  get "/health", to: "health#show"
end
```

---

## ขั้นตอนที่ 696-705: Advanced Configuration

### Puma Configuration

```ruby
# config/puma.rb
# Threads
max_threads_count = ENV.fetch("RAILS_MAX_THREADS", 5)
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

# Workers (สำหรับ production)
workers ENV.fetch("WEB_CONCURRENCY", 2)

# Worker timeout
worker_timeout 3600 if ENV.fetch("RAILS_ENV", "development") == "development"

# Port
port ENV.fetch("PORT", 3000)

# Environment
environment ENV.fetch("RAILS_ENV", "development")

# PID file
pidfile ENV.fetch("PIDFILE", "tmp/pids/server.pid")

# Allow puma to be restarted
plugin :tmp_restart

# Preload app (better memory usage with workers)
preload_app!

on_worker_boot do
  # Worker-specific setup (reconnect databases)
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end
```

### Active Record Configuration

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # Connection pool
    config.active_record.warn_on_records_fetched_greater_than = 1500

    # Encryption (Rails 7+)
    config.active_record.encryption.primary_key = Rails.application.credentials.dig(:active_record_encryption, :primary_key)
    config.active_record.encryption.deterministic_key = Rails.application.credentials.dig(:active_record_encryption, :deterministic_key)
    config.active_record.encryption.key_derivation_salt = Rails.application.credentials.dig(:active_record_encryption, :key_derivation_salt)
  end
end
```

### Action Cable Configuration

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

### Active Storage Configuration

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

# Amazon S3
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: ap-southeast-1
  bucket: <%= Rails.application.credentials.dig(:aws, :bucket) %>

# Google Cloud Storage
google:
  service: GCS
  credentials: <%= Rails.root.join("path/to/keyfile.json") %>
  project: my-project
  bucket: my-bucket

# Azure
microsoft:
  service: AzureStorage
  storage_account_name: my_account
  storage_access_key: my_access_key
  container: my_container
```

---

## แบบฝึกหัด Part 32 (ขั้นตอนที่ 686-705)

### แบบฝึกหัดที่ 1-5: Rails New Options

**ข้อ 1:** สร้าง Rails API application พร้อม PostgreSQL database

```bash
# คำตอบ:
rails new api_app \
  --api \
  --database=postgresql \
  --skip-test

cd api_app
bundle install
rails db:create
```

**ข้อ 2:** สร้าง Rails app ด้วย Tailwind CSS และ PostgreSQL

```bash
# คำตอบ:
rails new blog_app \
  --database=postgresql \
  --css=tailwind \
  --javascript=importmap

cd blog_app
bundle install
rails db:create
rails server
```

**ข้อ 3:** สร้าง minimal Rails app ที่ skip Action Mailer, Active Storage, Action Cable

```bash
# คำตอบ:
rails new myapp \
  --skip-action-mailer \
  --skip-active-storage \
  --skip-action-cable \
  --skip-hotwire \
  --database=postgresql
```

**ข้อ 4:** สร้าง Rails template ที่ auto-install Devise และ RSpec

```ruby
# คำตอบ:
# template.rb
gem "devise"
gem_group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end

after_bundle do
  generate "devise:install"
  generate "devise", "User"
  generate "rspec:install"

  rails_command "db:create"
  rails_command "db:migrate"

  git :init
  git add: "."
  git commit: %Q{ -m "Initial commit with Devise + RSpec" }
end

# รัน:
# rails new myapp --template=template.rb
```

**ข้อ 5:** อธิบายความแตกต่างระหว่าง `rails new myapp --api` กับ `rails new myapp`

```
คำตอบ:
API Mode (--api):
- ไม่มี View layer (ไม่มี ERB templates)
- ไม่มี Asset Pipeline
- ไม่มี Session middleware (cookies)
- ไม่มี Flash messages
- Controller inherit จาก ActionController::API (เร็วกว่า)
- เหมาะสำหรับ JSON API backend

Standard Mode:
- มี View layer (ERB, Haml, etc.)
- มี Asset Pipeline
- มี Session/Cookie support
- มี Flash messages
- Controller inherit จาก ActionController::Base
- เหมาะสำหรับ full-stack web app
```

### แบบฝึกหัดที่ 6-10: Directory Structure

**ข้อ 6:** อธิบายความแตกต่างระหว่าง `app/controllers/concerns/` กับ `app/models/concerns/`

```
คำตอบ:
app/controllers/concerns/:
- เก็บ modules ที่ share behavior ระหว่าง controllers
- ตัวอย่าง: Authenticatable, Paginatable, Trackable

app/models/concerns/:
- เก็บ modules ที่ share behavior ระหว่าง models
- ตัวอย่าง: Sluggable, SoftDeletable, Searchable

ทั้งคู่ใช้ ActiveSupport::Concern
```

**ข้อ 7:** สร้าง Concern ที่เพิ่ม search functionality ให้กับ model

```ruby
# คำตอบ:
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  included do
    scope :search, ->(query) {
      return all if query.blank?

      searchable_columns = self.class.instance_variable_get(:@searchable_columns) || []
      conditions = searchable_columns.map { |col| "#{col} ILIKE :query" }.join(" OR ")

      where(conditions, query: "%#{query}%")
    }
  end

  class_methods do
    def searchable_by(*columns)
      @searchable_columns = columns.map(&:to_s)
    end
  end
end

# ใช้:
class Article < ApplicationRecord
  include Searchable
  searchable_by :title, :body
end

# Article.search("Rails")
```

**ข้อ 8:** สร้างโครงสร้าง directory สำหรับ feature-based organization

```bash
# คำตอบ:
mkdir -p app/services
mkdir -p app/queries
mkdir -p app/presenters
mkdir -p app/forms
mkdir -p app/decorators
mkdir -p app/policies

# เพิ่มใน config/application.rb:
config.autoload_paths += [
  Rails.root.join("app/services"),
  Rails.root.join("app/queries"),
  Rails.root.join("app/presenters"),
  Rails.root.join("app/forms"),
  Rails.root.join("app/decorators"),
  Rails.root.join("app/policies")
]
```

**ข้อ 9:** อธิบายไฟล์ใน bin/ directory และการใช้งาน

```bash
# คำตอบ:

# bin/bundle - Bundler wrapper
./bin/bundle install
./bin/bundle exec rails server

# bin/rails - Rails CLI
./bin/rails server
./bin/rails generate model Article
./bin/rails db:migrate

# bin/rake - Rake task runner  
./bin/rake db:seed
./bin/rake assets:precompile

# bin/setup - Project setup script
#!/usr/bin/env ruby
require "fileutils"

APP_ROOT = File.expand_path("..", __dir__)

def system!(*args)
  system(*args, exception: true)
end

FileUtils.chdir APP_ROOT do
  puts "== Installing dependencies =="
  system! "gem install bundler --conservative"
  system("bundle check") || system!("bundle install")

  puts "\n== Copying sample files =="
  unless File.exist?("config/database.yml")
    FileUtils.cp "config/database.yml.sample", "config/database.yml"
  end

  puts "\n== Preparing database =="
  system! "bin/rails db:prepare"

  puts "\n== Setup complete! =="
end
```

**ข้อ 10:** สร้าง Custom Rake task สำหรับ database maintenance

```ruby
# คำตอบ:
# lib/tasks/db_maintenance.rake
namespace :db do
  namespace :maintenance do
    desc "ลบ sessions เก่ากว่า 30 วัน"
    task clear_old_sessions: :environment do
      count = ActiveRecord::SessionStore::Session
        .where("updated_at < ?", 30.days.ago)
        .delete_all
      puts "ลบ #{count} sessions"
    end

    desc "Vacuum analyze PostgreSQL database"
    task vacuum: :environment do
      ActiveRecord::Base.connection.execute("VACUUM ANALYZE")
      puts "Database vacuum completed"
    end

    desc "ดู database statistics"
    task stats: :environment do
      tables = ActiveRecord::Base.connection.tables
      puts "\n=== Database Statistics ==="
      tables.sort.each do |table|
        count = ActiveRecord::Base.connection.execute("SELECT COUNT(*) FROM #{table}").first["count"]
        puts "#{table.ljust(30)} #{count.rjust(10)} rows"
      end
    end
  end
end
```

### แบบฝึกหัดที่ 11-15: Configuration

**ข้อ 11:** ตั้งค่า credentials สำหรับ Stripe payment

```bash
# คำตอบ:
# รัน:
rails credentials:edit

# เพิ่มใน credentials.yml.enc:
# stripe:
#   public_key: pk_test_xxx
#   secret_key: sk_test_xxx
#   webhook_secret: whsec_xxx

# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
Stripe.webhook_secret = Rails.application.credentials.dig(:stripe, :webhook_secret)
```

**ข้อ 12:** สร้าง initializer ที่ตั้งค่า global date/time formats

```ruby
# คำตอบ:
# config/initializers/datetime_formats.rb
Time::DATE_FORMATS.merge!(
  thai_date: "%d/%m/%Y",
  thai_datetime: "%d/%m/%Y %H:%M น.",
  thai_time: "%H:%M น.",
  iso_date: "%Y-%m-%d",
  full_date: "%A, %d %B %Y",
  month_year: "%B %Y"
)

Date::DATE_FORMATS.merge!(
  thai: "%d/%m/%Y",
  short_thai: "%d/%m/%y"
)
```

**ข้อ 13:** ตั้งค่า multiple databases ใน database.yml

```yaml
# คำตอบ:
# config/database.yml
default: &default
  adapter: postgresql
  pool: 5

development:
  primary:
    <<: *default
    database: myapp_development
  analytics:
    <<: *default
    database: myapp_analytics_development
    migrations_paths: db/analytics_migrate

# app/models/analytics_record.rb
class AnalyticsRecord < ApplicationRecord
  self.abstract_class = true
  connects_to database: { writing: :analytics, reading: :analytics }
end

# app/models/page_view.rb
class PageView < AnalyticsRecord
  validates :path, presence: true
end
```

**ข้อ 14:** ตั้งค่า Puma สำหรับ production ด้วย 4 workers และ 5 threads

```ruby
# คำตอบ:
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY", 4)
threads ENV.fetch("RAILS_MIN_THREADS", 1), ENV.fetch("RAILS_MAX_THREADS", 5)

environment ENV.fetch("RAILS_ENV", "production")
port ENV.fetch("PORT", 3000)
pidfile ENV.fetch("PIDFILE", "tmp/pids/server.pid")

preload_app!

on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
  Redis.current = Redis.new(url: ENV["REDIS_URL"]) if defined?(Redis)
end

plugin :tmp_restart
```

**ข้อ 15:** เขียน .env.example ที่สมบูรณ์สำหรับ Rails app

```bash
# คำตอบ:
# .env.example
# Copy this file to .env and fill in your values

# Database
DATABASE_URL=postgres://localhost/myapp_development
DB_USERNAME=postgres
DB_PASSWORD=

# Redis
REDIS_URL=redis://localhost:6379/0

# Application
SECRET_KEY_BASE=
APP_HOST=localhost:3000
ADMIN_EMAIL=admin@example.com

# AWS S3
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-southeast-1
AWS_BUCKET=myapp-development

# Stripe
STRIPE_PUBLIC_KEY=pk_test_
STRIPE_SECRET_KEY=sk_test_
STRIPE_WEBHOOK_SECRET=whsec_

# SendGrid
SENDGRID_API_KEY=
MAIL_FROM=noreply@example.com

# Pusher (ถ้าใช้)
PUSHER_APP_ID=
PUSHER_KEY=
PUSHER_SECRET=
PUSHER_CLUSTER=ap1

# Google OAuth (ถ้าใช้)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Web Concurrency
WEB_CONCURRENCY=2
RAILS_MAX_THREADS=5
```

### แบบฝึกหัดที่ 16-20: Advanced Setup

**ข้อ 16:** สร้าง initializer สำหรับ configure CORS

```ruby
# คำตอบ:
# Gemfile
# gem "rack-cors"

# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins do |source, env|
      allowed_origins = [
        "http://localhost:3000",
        "http://localhost:3001",  # React dev server
        Rails.env.production? ? "https://myapp.com" : nil,
        Rails.env.production? ? "https://www.myapp.com" : nil
      ].compact

      allowed_origins.include?(source)
    end

    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,
      expose: ["Authorization", "X-Request-Id"],
      max_age: 600
  end
end
```

**ข้อ 17:** ตั้งค่า logging สำหรับ production

```ruby
# คำตอบ:
# config/environments/production.rb
Rails.application.configure do
  # JSON logging สำหรับ log aggregation
  config.logger = ActiveSupport::Logger.new(STDOUT)
  config.log_formatter = proc do |severity, datetime, progname, msg|
    {
      severity: severity,
      time: datetime.iso8601,
      app: Rails.application.class.module_parent_name,
      message: msg
    }.to_json + "\n"
  end

  config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info").to_sym
  config.log_tags = [:request_id, :remote_ip]
end
```

**ข้อ 18:** ตั้งค่า Active Job ให้ใช้ Sidekiq

```ruby
# คำตอบ:
# Gemfile
# gem "sidekiq"

# config/application.rb
config.active_job.queue_adapter = :sidekiq

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end

# config/sidekiq.yml
:concurrency: 5
:queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]
  - [mailers, 2]

# ใช้ใน job:
class WelcomeEmailJob < ApplicationJob
  queue_as :mailers

  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome_email(user).deliver_now
  end
end
```

**ข้อ 19:** สร้าง custom generator สำหรับ Service Object

```ruby
# คำตอบ:
# lib/generators/service_object/service_object_generator.rb
class ServiceObjectGenerator < Rails::Generators::NamedBase
  source_root File.expand_path("templates", __dir__)

  def create_service_file
    template "service_object.rb.tt",
      File.join("app/services", class_path, "#{file_name}_service.rb")
  end

  def create_spec_file
    template "service_object_spec.rb.tt",
      File.join("spec/services", class_path, "#{file_name}_service_spec.rb")
  end
end

# lib/generators/service_object/templates/service_object.rb.tt
class <%= class_name %>Service
  Result = Struct.new(:success?, :data, :errors, keyword_init: true)

  def initialize(params = {})
    @params = params
  end

  def call
    # TODO: Implement
    Result.new(success?: true, data: nil, errors: [])
  rescue => e
    Result.new(success?: false, data: nil, errors: [e.message])
  end

  private

  attr_reader :params
end

# รัน:
# rails g service_object UserRegistration
```

**ข้อ 20:** ตั้งค่า config/application.rb แบบสมบูรณ์

```ruby
# คำตอบ:
# config/application.rb
require_relative "boot"
require "rails/all"

Bundler.require(*Rails.groups)

module MyApp
  class Application < Rails::Application
    config.load_defaults 7.1

    # Time
    config.time_zone = "Bangkok"

    # Locale
    config.i18n.default_locale = :th
    config.i18n.available_locales = [:th, :en]
    config.i18n.fallbacks = [:en]

    # Autoload paths
    config.autoload_paths += [
      Rails.root.join("app/services"),
      Rails.root.join("app/queries"),
      Rails.root.join("app/forms"),
      Rails.root.join("app/presenters"),
      Rails.root.join("lib")
    ]

    # Generators
    config.generators do |g|
      g.test_framework :rspec, fixtures: false
      g.fixture_replacement :factory_bot, dir: "spec/factories"
      g.helper false
      g.stylesheets false
      g.jbuilder false
    end

    # Active Job
    config.active_job.queue_adapter = :sidekiq
    config.active_job.queue_name_prefix = Rails.env

    # Mailer
    config.action_mailer.default_url_options = {
      host: ENV.fetch("APP_HOST", "localhost:3000")
    }

    # Logging
    config.log_level = :debug

    # Middleware
    config.middleware.use Rack::Deflater

    # Security headers
    config.action_dispatch.default_headers = {
      "X-Frame-Options" => "SAMEORIGIN",
      "X-XSS-Protection" => "1; mode=block",
      "X-Content-Type-Options" => "nosniff",
      "X-Download-Options" => "noopen",
      "X-Permitted-Cross-Domain-Policies" => "none",
      "Referrer-Policy" => "strict-origin-when-cross-origin"
    }
  end
end
```

---

## สรุป Part 32

ในบทนี้เราได้เรียนรู้:

1. **Rails New Options** - API mode, database, CSS, JS, skip options
2. **Directory Structure** - ทุก directory และ file มีหน้าที่ชัดเจน
3. **config/ Directory** - application.rb, environments, initializers
4. **Environments** - development, test, production configurations
5. **database.yml** - PostgreSQL, MySQL, SQLite, Multiple databases
6. **Initializers** - Devise, Sidekiq, Inflections, CORS
7. **Credentials** - การเก็บ secrets อย่างปลอดภัย
8. **dotenv** - การใช้ .env files
9. **Generators** - Model, Controller, Scaffold, Custom generators
10. **Rake Tasks** - Custom maintenance tasks

---

*ต่อไป: Part 33 - Routing (ขั้นตอนที่ 706-730)*
