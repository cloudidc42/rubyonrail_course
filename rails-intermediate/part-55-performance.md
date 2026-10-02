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
