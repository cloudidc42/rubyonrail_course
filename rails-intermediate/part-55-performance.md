# Part 55: Performance Optimization ใน Rails

## ขั้นตอนที่ 1201-1220: ปรับแต่งประสิทธิภาพ

---

## ขั้นตอนที่ 1201: N+1 Query Problem

```ruby
# ❌ N+1 Query Problem
posts = Post.all
posts.each do |post|
  puts post.user.name    # Query แยกสำหรับแต่ละ post
  puts post.comments.count
end
# 1 query for posts + N queries for users + N queries for comments = 2N+1 queries!
```

```ruby
# ✅ Eager loading แก้ N+1
posts = Post.includes(:user, :comments).all
posts.each do |post|
  puts post.user.name     # ใช้ข้อมูลที่โหลดแล้ว
  puts post.comments.count
end
# 3 queries total (posts + users + comments)

# includes vs preload vs eager_load
Post.includes(:user)    # Rails เลือก preload หรือ eager_load อัตโนมัติ
Post.preload(:user)     # เสมอใช้ 2 queries แยก
Post.eager_load(:user)  # เสมอใช้ LEFT OUTER JOIN
Post.joins(:user)       # INNER JOIN (ไม่ load user objects)
```

## ขั้นตอนที่ 1202: Bullet Gem

```ruby
# Gemfile
group :development do
  gem 'bullet'
end

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true          # JavaScript alert
  Bullet.rails_logger = true   # log ใน development.log
  Bullet.add_footer = true     # แสดงใน footer
  
  # Notifications
  Bullet.raise = true           # raise exception (สำหรับ test)
  # Bullet.slack = { webhook_url: ENV['SLACK_WEBHOOK'] }
  
  # แจ้งเตือน
  Bullet.n_plus_one_query_enable = true
  Bullet.unused_eager_loading_enable = true
  Bullet.counter_cache_enable = true
end
```

```ruby
# สิ่งที่ Bullet แจ้งเตือน:
# N+1 queries
# Unused eager loading (โหลด association แต่ไม่ได้ใช้)
# Missing counter cache

# Example output:
# N+1 Query: USE eager loading for Post => [:user, :comments]
# Unused eager loading detected: Post => [:tags]
# Need Counter Cache for Post => [:comments]
```

## ขั้นตอนที่ 1203: Database Indexes

```ruby
# สร้าง indexes สำคัญ
class AddIndexes < ActiveRecord::Migration[7.0]
  def change
    # Foreign keys
    add_index :posts, :user_id
    add_index :posts, :category_id
    add_index :comments, :post_id
    add_index :comments, :user_id
    
    # Commonly queried columns
    add_index :posts, :published_at
    add_index :posts, :status
    add_index :users, :email, unique: true
    
    # Composite indexes (column ที่ใช้ร่วมกันบ่อย)
    add_index :posts, [:user_id, :published_at]
    add_index :posts, [:status, :published_at]
    
    # Partial index (index เฉพาะ subset ของข้อมูล)
    add_index :posts, :published_at, 
              where: "status = 'published'",
              name: "index_posts_on_published_published_at"
    
    # Full-text search index (PostgreSQL)
    add_index :posts, :title, using: :gin,
              opclass: :gin_trgm_ops,
              name: "index_posts_on_title_trigram"
  end
end
```

```sql
-- ตรวจสอบ indexes ที่มีอยู่
SELECT tablename, indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;

-- ตรวจสอบ slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

## ขั้นตอนที่ 1204: Query Optimization

```ruby
# ❌ Slow: โหลดข้อมูลทั้งหมด
users = User.all.select { |u| u.active? }  # ดึงทุก row แล้ว filter ใน Ruby

# ✅ Fast: filter ที่ database
users = User.where(active: true)

# ❌ count ใน Ruby
User.all.count { |u| u.admin? }

# ✅ count ที่ database
User.where(admin: true).count

# ❌ pluck ทีละ field
User.all.map(&:email)

# ✅ pluck (ดึง specific columns)
User.pluck(:email)
User.pluck(:id, :name, :email)

# ❌ select ทุก column
Post.all.each { |p| puts p.title }

# ✅ select เฉพาะ columns ที่ต้องการ
Post.select(:id, :title, :user_id).each { |p| puts p.title }

# ❌ load ข้อมูลทั้งหมดเพื่อ check exists
if Post.where(status: 'published').count > 0

# ✅ exists? query (faster)
if Post.where(status: 'published').exists?
```

```ruby
# Batch processing - ป้องกัน memory issues
# ❌ โหลดทั้งหมด
User.all.each { |user| send_email(user) }  # อาจ OOM!

# ✅ find_each - process ทีละ 1000
User.find_each(batch_size: 500) { |user| send_email(user) }

# ✅ find_in_batches - process เป็น batches
User.find_in_batches(batch_size: 1000) do |users|
  users.each { |user| send_email(user) }
end

# ✅ in_batches (Rails 5+)
User.in_batches(of: 500) do |batch|
  batch.update_all(status: 'inactive')  # batch SQL update
end
```

## ขั้นตอนที่ 1205: Rack Mini Profiler

```ruby
# Gemfile
gem 'rack-mini-profiler'
gem 'flamegraph'  # สำหรับ flame graph
gem 'stackprof'   # สำหรับ CPU profiling

# config/initializers/mini_profiler.rb
Rack::MiniProfiler.config.authorization_mode = :allow_authorized
Rack::MiniProfiler.config.position = 'bottom-right'
Rack::MiniProfiler.config.start_hidden = false

# เปิด/ปิดตาม environment
if Rails.env.production?
  Rack::MiniProfiler.config.authorization_mode = :allow_authorized
  
  Rack::MiniProfiler.config.should_profile = lambda do |env|
    env['warden']&.user&.admin?
  end
end
```

```
Query parameters สำหรับ profiling:
?pp=help          - แสดง options
?pp=env           - environment info
?pp=skip          - ข้ามหน้านี้
?pp=flamegraph    - flame graph
?pp=profile-memory - memory profiling
```

## ขั้นตอนที่ 1206: Puma Configuration

```ruby
# config/puma.rb
# Workers = จำนวน processes
workers ENV.fetch("WEB_CONCURRENCY") { 2 }

# Threads = concurrent requests ต่อ worker
max_threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

# Port
port ENV.fetch("PORT") { 3000 }

# Environment
environment ENV.fetch("RAILS_ENV") { "development" }

# Worker timeout
worker_timeout 3600 if ENV.fetch("RAILS_ENV", "development") == "development"

# Preload app สำหรับ Copy-on-Write
preload_app!

# Database connection ใน each worker
on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end

# Before fork
before_fork do
  ActiveRecord::Base.connection_pool.disconnect! if defined?(ActiveRecord)
end
```

## ขั้นตอนที่ 1207: Memory Optimization

```ruby
# ตรวจสอบ memory usage
class ApplicationController < ActionController::Base
  around_action :log_memory if Rails.env.development?
  
  private
  
  def log_memory
    before = GetProcessMem.new.mb
    yield
    after = GetProcessMem.new.mb
    
    if (after - before) > 10  # เกิน 10MB
      Rails.logger.warn "Memory increased #{after - before}MB for #{request.path}"
    end
  end
end
```

```ruby
# Memory leaks prevention

# ❌ สะสม objects
class DataProcessor
  @@cache = {}  # Class variable สะสมตลอด
  
  def process(id)
    @@cache[id] = fetch_data(id)
  end
end

# ✅ ใช้ LRU cache
require 'lru_redux'

class DataProcessor
  CACHE = LruRedux::Cache.new(1000)  # จำกัดที่ 1000 entries
  
  def process(id)
    CACHE[id] ||= fetch_data(id)
  end
end

# ✅ ล้าง references
def process_large_dataset
  data = load_huge_dataset
  result = transform(data)
  data = nil  # Allow GC
  GC.compact   # Compact heap
  result
end
```

## ขั้นตอนที่ 1208: Database Connection Pooling

```ruby
# config/database.yml
production:
  adapter: postgresql
  pool: <%= ENV.fetch("DB_POOL") { 10 } %>
  timeout: 5000
  checkout_timeout: 5
  
# ควร match กับ Puma threads
# pool >= (workers * threads)
```

```ruby
# Monitor connection pool
ActiveRecord::Base.connection_pool.stat
# => { size: 10, connections: 3, busy: 1, dead: 0, idle: 2, waiting: 0, checkout_timeout: 5 }

# ป้องกัน connection leak
def some_operation
  ActiveRecord::Base.connection_pool.with_connection do |conn|
    conn.execute("SELECT 1")
  end
end
```

## ขั้นตอนที่ 1209: Caching ที่ Application Level

```ruby
# ดู Part 49 สำหรับรายละเอียด Caching
# สรุปสำหรับ Performance:

# 1. HTTP Caching - ลด server load มากที่สุด
def index
  @posts = Post.published.order(updated_at: :desc)
  
  if stale?(last_modified: @posts.maximum(:updated_at), public: true)
    expires_in 5.minutes, public: true
    render :index
  end
end

# 2. Fragment Cache - ลด rendering time
<% cache @post, expires_in: 1.hour do %>
  <%= render @post %>
<% end %>

# 3. Counter Cache - ลด COUNT queries
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# 4. Memoization - ลด repeated computation
def user_with_roles
  @user_with_roles ||= current_user.includes(:roles)
end
```

## ขั้นตอนที่ 1210: Background Jobs สำหรับ Performance

```ruby
# ย้าย heavy operations ไปทำใน background

# ❌ ทำใน request cycle
def create
  @post = Post.create!(post_params)
  
  # Heavy operations - ทำให้ response ช้า
  ImageOptimizer.optimize_all(@post.images)
  SearchIndexer.index(@post)
  NotificationSender.send_to_followers(@post)
  
  redirect_to @post
end

# ✅ ทำใน background
def create
  @post = Post.create!(post_params)
  
  PostCreatedJob.perform_later(@post.id)
  
  redirect_to @post  # ตอบ user ทันที
end

class PostCreatedJob < ApplicationJob
  def perform(post_id)
    post = Post.find(post_id)
    ImageOptimizationJob.perform_later(post_id)
    SearchIndexJob.perform_later(post_id)
    NotificationJob.perform_later(post_id)
  end
end
```

## ขั้นตอนที่ 1211: Asset Performance

```ruby
# config/environments/production.rb
config.assets.compress = true
config.assets.css_compressor = :sass
config.assets.js_compressor = :terser
config.assets.digest = true  # fingerprinting

# CDN
config.asset_host = ENV['CDN_HOST']  # "https://cdn.myapp.com"

# Precompile assets
config.assets.precompile += %w[admin.css admin.js email.css]
```

## ขั้นตอนที่ 1212: Measuring Performance

```ruby
# Benchmark
require 'benchmark'

result = Benchmark.measure do
  Post.all.map(&:title)
end
puts result
# =>  0.100000   0.010000   0.110000 (  0.125000)
# CPU User  CPU System  CPU Total  Real Time

# Benchmark.bm - compare methods
Benchmark.bm(20) do |x|
  x.report("map:") { Post.all.map(&:title) }
  x.report("pluck:") { Post.pluck(:title) }
end

# ActiveSupport::Notifications
ActiveSupport::Notifications.subscribe('sql.active_record') do |name, start, finish, id, payload|
  duration = finish - start
  if duration > 0.1  # 100ms
    Rails.logger.warn "Slow query (#{(duration * 1000).round}ms): #{payload[:sql]}"
  end
end
```

## ขั้นตอนที่ 1213: Performance Monitoring in Production

```ruby
# config/initializers/performance_monitoring.rb

# Track slow requests
ActiveSupport::Notifications.subscribe("process_action.action_controller") do |event|
  if event.duration > 1000  # > 1 second
    Sentry.with_scope do |scope|
      scope.set_tags(
        controller: event.payload[:controller],
        action: event.payload[:action]
      )
      scope.set_context("request", {
        path: event.payload[:path],
        duration_ms: event.duration.round
      })
      
      Sentry.capture_message("Slow request: #{event.duration.round}ms")
    end
  end
end

# Track slow database queries
ActiveSupport::Notifications.subscribe("sql.active_record") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  
  if event.duration > 500  # > 500ms
    Rails.logger.warn "[SLOW SQL] #{event.duration.round}ms: #{event.payload[:sql]}"
  end
end
```

---

## แบบฝึกหัด: Performance (20 ข้อ)

### ข้อที่ 1: Fix N+1 Query
```ruby
# ❌ มี N+1
@posts = Post.all
@posts.each { |p| p.user.name }

# ✅ แก้ด้วย
@posts = Post.includes(:user).all
```

### ข้อที่ 2: Setup Bullet
```ruby
# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.n_plus_one_query_enable = true
  Bullet.rails_logger = true
end
```

### ข้อที่ 3: Add Indexes
```
เพิ่ม indexes สำหรับ foreign keys และ commonly queried columns
```

### ข้อที่ 4: Use pluck instead of map
```ruby
# แทน User.all.map(&:email)
User.pluck(:email)
```

### ข้อที่ 5: Batch Processing
```
ส่ง email ไปยัง users 100,000 คน โดยใช้ find_each
```

### ข้อที่ 6-20 (แบบสรุป)

**ข้อ 6:** Setup Rack Mini Profiler
**ข้อ 7:** Puma configuration
**ข้อ 8:** Counter cache สำหรับ comments
**ข้อ 9:** Move image processing to background job
**ข้อ 10:** HTTP caching ใน API endpoint
**ข้อ 11:** Fragment caching ใน views
**ข้อ 12:** Database connection pool tuning
**ข้อ 13:** Query ด้วย select แค่ columns ที่ต้องการ
**ข้อ 14:** Benchmark เปรียบเทียบ 2 implementation
**ข้อ 15:** Partial indexes
**ข้อ 16:** Monitor slow queries ใน production
**ข้อ 17:** Memory leak detection
**ข้อ 18:** CDN setup
**ข้อ 19:** Asset compression
**ข้อ 20:** Load testing ด้วย k6 หรือ Apache Bench

---

## สรุป: Performance Optimization

| ปัญหา | แนวทางแก้ |
|-------|-----------|
| N+1 Queries | `includes`, `eager_load`, `preload` |
| Slow queries | Database indexes, query optimization |
| Memory issues | Batch processing, streaming |
| Slow rendering | Fragment caching, CDN |
| Heavy operations | Background jobs |
| DB bottleneck | Connection pooling, read replicas |
| Asset loading | Compression, CDN, browser caching |

**Key Takeaways:**
1. ใช้ Bullet gem ตรวจหา N+1 ใน development
2. Measure ก่อน optimize - อย่า guess
3. Database indexes ช่วยได้มาก
4. Cache aggressively
5. ย้าย heavy work ไป background jobs
6. Monitor performance ใน production

---

## Step 1206: Query Optimization เชิงลึก

### includes vs preload vs eager_load

```ruby
# includes - Rails เลือก strategy เอง
User.includes(:posts).where(posts: { published: true })

# preload - ใช้ separate queries เสมอ
User.preload(:posts)
# SQL: SELECT * FROM users
# SQL: SELECT * FROM posts WHERE user_id IN (1,2,3...)

# eager_load - ใช้ LEFT OUTER JOIN เสมอ
User.eager_load(:posts)
# SQL: SELECT users.*, posts.* FROM users LEFT OUTER JOIN posts ON posts.user_id = users.id

# ใช้ eager_load เมื่อต้อง filter ด้วย association
User.eager_load(:posts).where(posts: { published: true })
```

### joins สำหรับ filtering ไม่ต้องโหลด data

```ruby
# เร็วกว่า includes เมื่อไม่ต้องใช้ associated data
User.joins(:posts).where(posts: { published: true }).distinct
# SQL: SELECT DISTINCT users.* FROM users INNER JOIN posts ON posts.user_id = users.id WHERE posts.published = true

# Joins กับ conditions
Post.joins(:comments).where(comments: { approved: true }).group("posts.id")
```

---

## Step 1207: Database Connection Pooling

```yaml
# config/database.yml
production:
  adapter: postgresql
  database: myapp_production
  pool: <%= ENV.fetch("DB_POOL", 10) %>
  checkout_timeout: 5
  idle_timeout: 300
  connect_timeout: 5
```

```ruby
# config/puma.rb
# ปรับ threads ให้สัมพันธ์กับ DB pool
threads_count = Integer(ENV.fetch("RAILS_MAX_THREADS", 5))
threads threads_count, threads_count
workers Integer(ENV.fetch("WEB_CONCURRENCY", 2))

# DB pool = threads * workers
# ENV["DB_POOL"] = threads_count * workers_count
```

```ruby
# ตรวจสอบ connection pool
ActiveRecord::Base.connection_pool.stat
# => {size: 10, connections: 3, busy: 1, dead: 0, idle: 2, waiting: 0, checkout_timeout: 5.0}

# ป้องกัน connection leak
ActiveRecord::Base.connection_pool.with_connection do |conn|
  # ใช้ connection ที่นี่
end
```

---

## Step 1208: Memory Optimization

```ruby
# ใช้ find_each แทน all สำหรับ large datasets
# BAD: โหลดทั้งหมดเข้า memory
User.all.each { |u| process(u) }

# GOOD: โหลดทีละ batch (default 1000)
User.find_each { |u| process(u) }

# GOOD: custom batch size
User.find_each(batch_size: 500) { |u| process(u) }

# GOOD: find_in_batches สำหรับ batch processing
User.find_in_batches(batch_size: 1000) do |batch|
  batch.each { |u| process(u) }
end
```

```ruby
# Select only needed columns
User.select(:id, :name, :email).find_each do |user|
  # user จะมีแค่ id, name, email - ประหยัด memory
  send_email(user.email)
end

# pluck สำหรับ single/multiple columns
User.pluck(:email)
# => ["user1@example.com", "user2@example.com"]

User.pluck(:id, :name)
# => [[1, "Alice"], [2, "Bob"]]
```

---

## Step 1209: Database Indexes เชิงลึก

```ruby
# Composite index
add_index :orders, [:user_id, :status, :created_at]
# เหมาะสำหรับ: User.joins(:orders).where(orders: { status: 'pending' }).order(:created_at)

# Partial index - index เฉพาะ subset
add_index :users, :email, where: "deleted_at IS NULL", unique: true
add_index :orders, :created_at, where: "status = 'pending'"

# Covering index - index ครอบคลุม columns ที่ SELECT
add_index :products, [:category_id, :price, :name]
# Query จะใช้ index only scan เมื่อ SELECT id, category_id, price, name

# Expression index
add_index :users, "lower(email)", unique: true, name: "idx_users_lower_email"
# User.where("lower(email) = lower(?)", params[:email])
```

```sql
-- ตรวจสอบ index usage ใน PostgreSQL
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- ดู indexes ที่ไม่ได้ใช้
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY schemaname, tablename;
```

---

## Step 1210: Caching Strategies

```ruby
# Russian Doll Caching
# app/views/posts/index.html.erb
<% cache ["posts-list", @posts.maximum(:updated_at)] do %>
  <% @posts.each do |post| %>
    <% cache post do %>  <%# cache_key = "posts/#{id}-#{updated_at}" %>
      <%= render post %>
    <% end %>
  <% end %>
<% end %>

# Low-level caching
def popular_products
  Rails.cache.fetch("popular_products", expires_in: 15.minutes) do
    Product.popular.includes(:category).limit(10).to_a
  end
end

# Fragment caching manual invalidation
def invalidate_product_cache(product)
  Rails.cache.delete("product_#{product.id}_details")
  Rails.cache.delete_matched("products_category_#{product.category_id}*")
end
```

---

## Step 1211: Rack Mini Profiler

```ruby
# Gemfile
gem 'rack-mini-profiler', require: false
gem 'stackprof'  # CPU profiling
gem 'memory_profiler'  # Memory profiling

# config/initializers/rack_mini_profiler.rb
if defined?(Rack::MiniProfiler)
  Rack::MiniProfiler.config.position = 'bottom-right'
  Rack::MiniProfiler.config.start_hidden = false
  
  # ดู SQL queries
  Rack::MiniProfiler.config.enable_advanced_debugging_tools = true
  
  # ใช้ Redis สำหรับ storage
  Rack::MiniProfiler.config.storage = Rack::MiniProfiler::RedisStore
  Rack::MiniProfiler.config.storage_options = { url: ENV["REDIS_URL"] }
  
  # Allow only admins
  Rack::MiniProfiler.config.authorization_mode = :allow_authorized
end
```

```ruby
# Manual profiling
Rack::MiniProfiler.step("Expensive operation") do
  # code to profile
end

# Flamegraph: เพิ่ม ?pp=flamegraph ใน URL
# Memory profiling: เพิ่ม ?pp=profile-memory
# GC profiling: เพิ่ม ?pp=profile-gc
```

---

## Step 1212: Background Processing สำหรับ Heavy Tasks

```ruby
# แทนที่การทำงานใน request-response cycle
# BAD: ทำงานใน controller (ช้า)
def generate_report
  @report = Reports::Generator.new(params).generate  # ใช้เวลา 30 วินาที
  send_data @report, filename: "report.csv"
end

# GOOD: สร้าง background job
def generate_report
  job = ReportGenerationJob.perform_later(current_user.id, params.to_h)
  redirect_to reports_path, notice: "กำลังสร้างรายงาน จะแจ้งเมื่อเสร็จ"
end

# app/jobs/report_generation_job.rb
class ReportGenerationJob < ApplicationJob
  queue_as :reports
  
  def perform(user_id, params)
    user = User.find(user_id)
    report = Reports::Generator.new(params).generate
    
    # บันทึก report
    file = user.reports.create!(
      name: "Report #{Time.current.strftime('%Y%m%d_%H%M%S')}",
      status: :completed
    )
    file.file.attach(io: StringIO.new(report), filename: "report.csv")
    
    # แจ้งเตือน user
    UserMailer.with(user: user, report: file).report_ready.deliver_later
    ActionCable.server.broadcast("notifications_#{user_id}", {
      type: 'report_ready',
      report_id: file.id,
      message: 'รายงานพร้อมแล้ว'
    })
  end
end
```

---

## Step 1213: HTTP Caching

```ruby
# app/controllers/products_controller.rb
def show
  @product = Product.find(params[:id])
  
  # ETag-based caching
  fresh_when(@product, public: true)
  
  # หรือ manual
  if stale?(@product, public: true)
    render :show
  end
end

def index
  @products = Product.order(updated_at: :desc)
  
  # Cache-Control header
  expires_in 10.minutes, public: true
  
  # Last-Modified
  fresh_when(last_modified: @products.maximum(:updated_at), public: true)
end
```

---

## Step 1214: Performance Testing

```ruby
# spec/support/performance_helpers.rb
module PerformanceHelpers
  def measure_time(&block)
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    result = block.call
    elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    [result, elapsed]
  end
  
  def expect_query_count(count, &block)
    queries = []
    subscriber = ActiveSupport::Notifications.subscribe("sql.active_record") do |_name, _start, _finish, _id, payload|
      queries << payload[:sql] unless payload[:name] == 'SCHEMA'
    end
    
    block.call
    ActiveSupport::Notifications.unsubscribe(subscriber)
    
    expect(queries.length).to eq(count),
      "Expected #{count} queries but got #{queries.length}:\n#{queries.join("\n")}"
  end
end

# spec/requests/products_spec.rb
RSpec.describe "Products", type: :request do
  include PerformanceHelpers
  
  describe "GET /products" do
    it "renders in under 200ms" do
      create_list(:product, 10, :with_category)
      
      _, elapsed = measure_time { get products_path }
      
      expect(elapsed).to be < 0.2
    end
    
    it "uses N+1 safe queries" do
      create_list(:product, 5, :with_category)
      
      expect_query_count(2) { get products_path }  # 1 products + 1 categories
    end
  end
end
```

---

## Step 1215: Monitoring และ APM

```ruby
# Gemfile
gem 'scout_apm'  # หรือ
gem 'newrelic_rpm'  # หรือ
gem 'datadog'

# config/initializers/scout.rb
ScoutApm::Agent.config(
  name: 'MyApp',
  key: ENV['SCOUT_KEY'],
  monitor: Rails.env.production?
)

# Custom instrumentation
class ProductService
  include ScoutApm::Tracer
  
  def complex_calculation
    instrument("ProductService", "complex_calculation") do
      # code here
    end
  end
end

# Custom metrics
StatsD.gauge("cache.hit_rate", cache_hit_rate)
StatsD.timing("db.query.time", query_time)
StatsD.increment("api.requests", tags: ["endpoint:products"])
```

---

## แบบฝึกหัดเพิ่มเติม: Performance

### ข้อ 1: Optimize N+1 Query

```ruby
# มี N+1 - แก้ไข
class Order < ApplicationRecord
  belongs_to :user
  has_many :order_items
  has_many :products, through: :order_items
end

# BAD
Order.all.each do |order|
  puts "#{order.user.name}: #{order.order_items.sum(:total)}"
end

# GOOD
Order.includes(:user).each do |order|
  puts "#{order.user.name}: #{order.order_items.sum(:total)}"
end

# BETTER - precompute sum
Order.includes(:user, :order_items)
     .select("orders.*, SUM(order_items.total) as items_total")
     .joins(:order_items)
     .group("orders.id")
     .each do |order|
  puts "#{order.user.name}: #{order.items_total}"
end
```

### ข้อ 2: Database Index สำหรับ Common Queries

```ruby
# เพิ่ม indexes สำหรับ queries เหล่านี้
# User.where(email: x)  
# Order.where(user_id: x).order(:created_at)
# Product.where(category_id: x, active: true).order(:price)

class AddPerformanceIndexes < ActiveRecord::Migration[7.0]
  def change
    add_index :users, :email, unique: true
    add_index :orders, [:user_id, :created_at]
    add_index :products, [:category_id, :active, :price]
  end
end
```

### ข้อ 3: Counter Cache

```ruby
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end

# Migration
add_column :users, :posts_count, :integer, default: 0
User.find_each { |u| User.reset_counters(u.id, :posts) }

# ใช้งาน - ไม่ต้องนับ COUNT query
user.posts_count  # ดึงจาก column โดยตรง
```

---

## สรุป Performance Optimization

| เทคนิค | Impact | Difficulty |
|--------|--------|-----------|
| แก้ N+1 queries | สูง | ต่ำ |
| เพิ่ม indexes | สูง | ต่ำ |
| Fragment caching | กลาง | ต่ำ |
| Background jobs | สูง | กลาง |
| Counter cache | กลาง | ต่ำ |
| find_each | กลาง | ต่ำ |
| HTTP caching | กลาง | กลาง |
| Connection pooling | กลาง | กลาง |
| CDN for assets | สูง | ต่ำ |
| Query optimization | สูง | สูง |

**Performance Checklist:**
1. ติดตั้ง Bullet gem ใน development
2. ตรวจสอบ query count ใน specs
3. เพิ่ม indexes สำหรับทุก foreign key และ query columns
4. ใช้ find_each สำหรับ large datasets
5. Cache ผลลัพธ์ที่ compute แพง
6. ใช้ background jobs สำหรับงานที่ใช้เวลานาน
7. Monitor ด้วย APM tool ใน production
8. Profile ด้วย Rack Mini Profiler ก่อน optimize
