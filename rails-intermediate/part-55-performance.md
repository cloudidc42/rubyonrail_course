# ตอนที่ 55: Performance Optimization (Steps 1201-1220)

## บทนำ

Performance optimization เป็นศาสตร์และศิลป์ในการทำให้ Rails application ทำงานได้เร็วขึ้น ใช้ทรัพยากรน้อยลง และ scale ได้ดีขึ้น ในบทนี้เราจะเรียนรู้ techniques สำคัญตั้งแต่การแก้ปัญหา N+1 queries จนถึง database connection pooling

---

## Step 1201: N+1 Queries Problem

### ตัวอย่าง N+1 Problem

```ruby
# ❌ N+1 Query Problem
# SQL: SELECT * FROM articles LIMIT 10
@articles = Article.published.limit(10)

# ใน view:
@articles.each do |article|
  # SQL: SELECT * FROM users WHERE id = 1
  # SQL: SELECT * FROM users WHERE id = 2
  # SQL: SELECT * FROM users WHERE id = 3
  # ... รวม 10 queries สำหรับ users!
  puts article.user.name
  
  # SQL: SELECT COUNT(*) FROM comments WHERE article_id = 1
  # SQL: SELECT COUNT(*) FROM comments WHERE article_id = 2
  # ... อีก 10 queries!
  puts article.comments.count
end

# รวม: 1 + 10 + 10 = 21 queries สำหรับแค่ 10 articles!
```

### การแก้ไขด้วย Eager Loading

```ruby
# ✅ แก้ด้วย includes
# SQL: SELECT * FROM articles LIMIT 10
# SQL: SELECT * FROM users WHERE id IN (1, 2, 3, ...)
# SQL: SELECT * FROM comments WHERE article_id IN (1, 2, 3, ...)
@articles = Article.published.includes(:user, :comments).limit(10)

# รวม: 3 queries เท่านั้น!
@articles.each do |article|
  puts article.user.name     # ไม่มี query เพิ่ม (cached)
  puts article.comments.count  # ไม่มี query เพิ่ม (cached)
end
```

### Deep Nested N+1

```ruby
# ❌ N+1 ลึกหลายระดับ
@articles.each do |article|
  article.comments.each do |comment|
    puts comment.user.name   # N+1 ใน comments!
    puts comment.likes.count # อีก N+1!
  end
end

# ✅ แก้ด้วย nested includes
@articles = Article.includes(comments: [:user, :likes]).published.limit(10)
```

---

## Step 1202: Bullet Gem

### Setup

```ruby
# Gemfile
group :development do
  gem 'bullet'
end
```

```ruby
# config/environments/development.rb
config.after_initialize do
  Bullet.enable        = true
  Bullet.alert         = true  # JavaScript alert
  Bullet.bullet_logger = true  # log ไปยัง bullet.log
  Bullet.console       = true  # log ใน Rails console
  Bullet.rails_logger  = true  # log ไปยัง Rails logger
  Bullet.add_footer    = true  # แสดงใน page footer
  
  # Notify ด้วย email
  Bullet.smtp_settings = {
    address: 'localhost', port: 25,
    from_email: 'bullet@example.com'
  }
  Bullet.send_email_async(:email, 'dev@example.com')
  
  # Ignore specific N+1
  Bullet.add_safelist type: :n_plus_one_query, class_name: "User", association: :articles
end
```

### Bullet Output Example

```
USE eager loading detected
  Article => [:user]
  Add to your query: .includes([:user])
  
AVOID eager loading detected  
  Article => [:tags]
  Remove from your query: .includes([:tags])

N+1 Query detected
  Article => [:comments]
  Add to your query: .includes([:comments])
```

---

## Step 1203: Database Indexes

### ตรวจสอบ Queries ที่ต้องการ Index

```ruby
# ใน rails console
# ดู queries ที่ไม่มี index
User.where(email: "test@example.com").explain
# QUERY PLAN
# Seq Scan on users  (ไม่ดี! Full table scan)

# หลัง add index
add_index :users, :email, unique: true
User.where(email: "test@example.com").explain
# Index Scan using index_users_on_email on users  ✓
```

### Migration สำหรับ Indexes

```ruby
class AddPerformanceIndexes < ActiveRecord::Migration[7.0]
  def change
    # Single column indexes
    add_index :articles, :published
    add_index :articles, :created_at
    add_index :articles, :user_id  # (ถ้ายังไม่มี)
    add_index :articles, :views_count
    
    # Composite index (ใช้สำหรับ queries ที่ filter หลาย fields)
    add_index :articles, [:published, :created_at]
    add_index :articles, [:user_id, :published, :created_at]
    
    # Unique index
    add_index :users, :email, unique: true
    add_index :users, :username, unique: true
    
    # Partial index (PostgreSQL)
    add_index :articles, :user_id, 
              where: "published = true",
              name: "index_articles_on_user_id_published"
    
    # GIN index สำหรับ JSONB
    add_index :products, :metadata, using: :gin
    
    # Full-text search index
    execute <<-SQL
      CREATE INDEX idx_articles_title ON articles USING GIN(to_tsvector('english', title));
    SQL
  end
end
```

### เมื่อควรเพิ่ม Index

```
✅ เพิ่ม index เมื่อ:
- ใช้ WHERE clause บ่อย
- ใช้ JOIN conditions
- ใช้ ORDER BY
- Foreign keys

❌ ไม่ต้องเพิ่ม index เมื่อ:
- ตารางเล็กมาก (< 1000 rows)
- Update/Insert บ่อยมาก (index ช้าลง writes)
- Column ที่มี cardinality ต่ำ (เช่น boolean ที่ส่วนใหญ่เป็น true)
```

---

## Step 1204: Query Optimization

### includes vs preload vs eager_load

```ruby
# includes - Rails เลือกวิธีที่เหมาะสม
Article.includes(:user)  

# preload - ใช้ separate SQL queries (แยก queries)
# SQL: SELECT * FROM articles
# SQL: SELECT * FROM users WHERE id IN (...)
Article.preload(:user)

# eager_load - ใช้ LEFT OUTER JOIN (single query)
# SQL: SELECT articles.*, users.* FROM articles LEFT OUTER JOIN users ON ...
Article.eager_load(:user)

# เมื่อใดใช้อะไร:
# - preload: ส่วนใหญ่ใช้ได้ดี
# - eager_load: เมื่อต้องการ filter/order ด้วย association columns
Article.eager_load(:user).where(users: { role: 'admin' })
```

### Select เฉพาะ Columns ที่ต้องการ

```ruby
# ❌ SELECT *
articles = Article.all

# ✅ SELECT เฉพาะที่ต้องการ
articles = Article.select(:id, :title, :created_at, :user_id)

# กับ association
articles = Article.includes(:user).select(
  "articles.id, articles.title, articles.created_at, articles.user_id"
)

# กับ joins
articles = Article.joins(:user)
                  .select("articles.id, articles.title, users.name as author_name")
```

### Query Scopes ที่ดี

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  # Scopes ที่ใช้บ่อย
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :popular, -> { order(views_count: :desc) }
  scope :by_author, ->(user) { where(user: user) }
  scope :from_last, ->(days) { where("created_at > ?", days.days.ago) }
  
  # Combined scope
  scope :trending, -> {
    published
      .from_last(7)
      .order(views_count: :desc)
      .limit(10)
  }
  
  # Scope กับ index usage
  # ✅ ใช้ index: [:published, :created_at]
  scope :recent_published, -> {
    where(published: true).order(created_at: :desc)
  }
end
```

---

## Step 1205: counter_cache

### Setup

```ruby
# Migration
class AddCounterCaches < ActiveRecord::Migration[7.0]
  def change
    add_column :articles, :comments_count, :integer, default: 0, null: false
    add_column :users, :articles_count, :integer, default: 0, null: false
    add_column :articles, :likes_count, :integer, default: 0, null: false
    
    # Reset existing counts
    Article.find_each do |article|
      Article.reset_counters(article.id, :comments)
    end
    User.find_each do |user|
      User.reset_counters(user.id, :articles)
    end
  end
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :article, counter_cache: true
  # เมื่อสร้าง comment → articles.comments_count + 1
  # เมื่อลบ comment → articles.comments_count - 1
end

# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :user, counter_cache: true
end

# app/models/like.rb
class Like < ApplicationRecord
  belongs_to :article, counter_cache: true
end
```

```ruby
# ใช้ counter_cache แทนการ query
# ❌ ช้า - ทำ query
article.comments.count  

# ✅ เร็ว - อ่านจาก column
article.comments_count  # ไม่มี SQL query
```

---

## Step 1206: Background Jobs สำหรับ Heavy Tasks

```ruby
# app/controllers/reports_controller.rb
class ReportsController < ApplicationController
  def create
    report = Report.create!(
      status: 'pending',
      user: current_user,
      params: report_params
    )
    
    # ทำใน background แทน
    GenerateReportJob.perform_later(report.id)
    
    # Response ทันที
    render json: { 
      success: true, 
      report_id: report.id,
      status_url: report_status_url(report) 
    }
  end
  
  def status
    report = current_user.reports.find(params[:id])
    render json: { 
      status: report.status,
      progress: report.progress_percent,
      download_url: report.completed? ? report_download_url(report) : nil
    }
  end
end

# app/jobs/generate_report_job.rb
class GenerateReportJob < ApplicationJob
  queue_as :default
  
  def perform(report_id)
    report = Report.find(report_id)
    report.update!(status: 'processing')
    
    # Heavy processing
    data = ReportGenerator.new(report).generate
    
    report.update!(
      status: 'completed',
      data: data,
      completed_at: Time.current
    )
  rescue => e
    report.update!(status: 'failed', error: e.message)
    raise
  end
end
```

---

## Step 1207: Fragment Caching สำหรับ Performance

```ruby
# app/views/articles/index.html.erb
<%# Cache ทั้ง list %>
<% cache ["articles/list", @articles.maximum(:updated_at), I18n.locale] do %>
  <% @articles.each do |article| %>
    <%# Cache แต่ละ article %>
    <% cache article do %>
      <%= render 'article', article: article %>
    <% end %>
  <% end %>
<% end %>
```

```ruby
# Low-level caching สำหรับ expensive calculations
class User < ApplicationRecord
  def analytics_summary
    Rails.cache.fetch("user/#{id}/analytics/#{Date.today}", expires_in: 1.hour) do
      {
        total_views: articles.sum(:views_count),
        total_articles: articles.published.count,
        avg_views: articles.average(:views_count).to_f.round(2),
        top_article: articles.order(views_count: :desc).first&.title
      }
    end
  end
end
```

---

## Step 1208: HTTP Caching

```ruby
# app/controllers/articles_controller.rb
def index
  @articles = Article.published.recent.includes(:user)
  
  # HTTP Cache-Control
  expires_in 10.minutes, public: true
  
  # หรือใช้ ETag
  if stale?(etag: @articles.maximum(:updated_at), public: true)
    render :index
  end
end

def show
  @article = Article.find(params[:id])
  
  # Cache สำหรับ public articles
  if @article.published?
    expires_in 1.hour, public: true
  else
    expires_now
  end
  
  if stale?(@article)
    render :show
  end
end
```

---

## Step 1209: Rack Mini Profiler

### Setup

```ruby
# Gemfile
group :development do
  gem 'rack-mini-profiler'
  gem 'flamegraph'
  gem 'stackprof'
  gem 'memory_profiler'
end
```

```ruby
# config/initializers/mini_profiler.rb (development)
if Rails.env.development?
  require 'rack-mini-profiler'
  Rack::MiniProfiler.config.position = 'bottom-right'
  Rack::MiniProfiler.config.show_sensitive_sql_warning = true
  Rack::MiniProfiler.config.skip_paths = ['/assets', '/packs']
end
```

### ใช้งาน

```
เปิด http://localhost:3000/articles
→ เห็น profiler badge ที่มุมหน้าจอ
→ คลิกเพื่อดู:
  - Total time
  - SQL queries
  - Duration ของแต่ละ query
  - Memory usage

URL params พิเศษ:
?pp=flamegraph  → Flame graph
?pp=profile-gc  → GC profiling
?pp=profile-memory → Memory profiling
?pp=analyze-memory → Allocation analysis
```

---

## Step 1210: Database Connection Pooling

### Puma Configuration

```ruby
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 2 }  # Number of worker processes
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

preload_app!

rackup DefaultRackup if defined?(DefaultRackup)
port ENV.fetch("PORT") { 3000 }
environment ENV.fetch("RAILS_ENV") { "development" }

on_worker_boot do
  # Re-establish connections after fork
  ActiveRecord::Base.establish_connection
end
```

### Database Pool Configuration

```yaml
# config/database.yml
production:
  adapter: postgresql
  database: myapp_production
  host: <%= ENV['DB_HOST'] %>
  username: <%= ENV['DB_USERNAME'] %>
  password: <%= ENV['DB_PASSWORD'] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  checkout_timeout: 5
  reaping_frequency: 10
  idle_timeout: 300
```

### PgBouncer สำหรับ Connection Pooling

```ini
# pgbouncer.ini
[databases]
myapp_production = host=localhost port=5432 dbname=myapp_production

[pgbouncer]
listen_port = 6432
listen_addr = *

auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction  # session | transaction | statement
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
server_idle_timeout = 600
```

---

## Step 1211: Memory Optimization

### Avoid Loading Too Much Data

```ruby
# ❌ Loading ทั้ง table ไปใน memory
articles = Article.all.map { |a| process(a) }

# ✅ Process in batches
Article.find_each(batch_size: 100) do |article|
  process(article)
end

# ✅ กับ update_all
Article.where("created_at < ?", 1.year.ago).update_all(archived: true)

# ✅ กับ bulk insert
Article.insert_all(articles_data)
```

### String Optimization

```ruby
# ❌ String concatenation ใน loop
result = ""
articles.each do |a|
  result += a.title + "\n"  # สร้าง string ใหม่ทุกครั้ง
end

# ✅ Use array join
lines = articles.map(&:title)
result = lines.join("\n")

# ✅ Use heredoc
result = <<~TEXT
  Title: #{article.title}
  Body: #{article.body}
TEXT
```

---

## Step 1212: Query Caching และ Memoization

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def expensive_calculation
    @expensive_calculation ||= begin
      # คำนวณครั้งเดียว เก็บใน instance variable
      Article.where(user: self).sum(:views_count)
    end
  end
  
  def reset_memoization!
    @expensive_calculation = nil
  end
end
```

```ruby
# app/services/dashboard_service.rb
class DashboardService
  def initialize(user)
    @user = user
  end
  
  def data
    @data ||= {
      stats: compute_stats,
      recent_articles: recent_articles,
      trending: trending_articles
    }
  end
  
  private
  
  def compute_stats
    Rails.cache.fetch("dashboard/#{@user.id}/stats/#{Date.today}", expires_in: 30.minutes) do
      {
        total_views: @user.articles.sum(:views_count),
        total_comments: @user.articles.joins(:comments).count,
        followers_count: @user.followers.count,
        avg_reading_time: calculate_avg_reading_time
      }
    end
  end
  
  def recent_articles
    @recent_articles ||= @user.articles.published.recent.limit(5).includes(:tags)
  end
  
  def trending_articles
    @trending_articles ||= Article.trending.includes(:user).limit(10)
  end
end
```

---

## Step 1213: ActiveRecord Performance Tips

### select ลด Memory Usage

```ruby
# ❌ Load ทุก column (waste memory)
users = User.all
users.each { |u| puts u.name }  # แต่ใช้แค่ name

# ✅ เลือกเฉพาะที่ต้องการ
users = User.select(:id, :name, :email)
users.each { |u| puts u.name }

# ✅ pluck สำหรับ raw values
names = User.pluck(:name)
user_data = User.pluck(:id, :name, :email)

# ✅ ids สำหรับ just IDs
user_ids = User.where(role: 'admin').ids
```

### Bulk Operations

```ruby
# ❌ N queries
articles.each { |a| a.update!(published: true) }

# ✅ Single query
Article.where(id: article_ids).update_all(published: true)

# ✅ insert_all (Rails 6+)
Article.insert_all(
  articles_data,
  unique_by: :slug  # optional
)

# ✅ upsert_all
Article.upsert_all(articles_data, unique_by: :slug)
```

---

## Step 1214: Connection Pooling Monitoring

```ruby
# ดู connection pool stats ใน production
ActiveRecord::Base.connection_pool.stat
# => { size: 5, connections: 3, busy: 1, dead: 0, idle: 2, waiting: 0, checkout_timeout: 5.0 }

# Log pool stats
class ApplicationController < ActionController::Base
  after_action :log_db_stats
  
  private
  
  def log_db_stats
    pool = ActiveRecord::Base.connection_pool
    stats = pool.stat
    
    if stats[:waiting] > 0
      Rails.logger.warn "DB Pool waiting: #{stats.inspect}"
    end
  end
end
```

---

## Step 1215: Performance Testing

### การวัด Performance ด้วย Benchmark

```ruby
# lib/tasks/benchmark.rake
namespace :benchmark do
  task articles: :environment do
    require 'benchmark'
    
    # Setup data
    user = User.first
    articles = Article.where(user: user).limit(100)
    
    Benchmark.bm(30) do |x|
      x.report("Without includes:") do
        articles.reload.each { |a| a.user.name; a.comments.count }
      end
      
      x.report("With includes:") do
        articles.includes(:user, :comments).reload.each { |a| a.user.name; a.comments_count }
      end
      
      x.report("With counter_cache:") do
        articles.includes(:user).reload.each { |a| a.user.name; a.comments_count }
      end
    end
  end
end
```

---

## แบบฝึกหัด (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** ระบุและแก้ไข N+1 query ในโค้ดต่อไปนี้
```ruby
# Original (N+1)
@posts = Post.all
@posts.each { |p| puts p.author.name }

# เฉลย
@posts = Post.includes(:author).all
@posts.each { |p| puts p.author.name }  # ไม่มี N+1
```

**ข้อ 2:** เพิ่ม index สำหรับ column ที่ใช้บ่อยใน WHERE clause
```ruby
# เฉลย
# Migration
add_index :articles, :published
add_index :articles, [:published, :created_at]
add_index :users, :email, unique: true
```

**ข้อ 3:** setup Bullet gem และอ่านการแจ้งเตือน
```ruby
# เฉลย
# Gemfile: gem 'bullet'
# config/environments/development.rb:
Bullet.enable = true
Bullet.rails_logger = true
Bullet.add_footer = true
```

**ข้อ 4:** เพิ่ม counter_cache สำหรับ comments count
```ruby
# เฉลย
# Migration:
add_column :articles, :comments_count, :integer, default: 0

# Model:
class Comment < ApplicationRecord
  belongs_to :article, counter_cache: true
end
```

**ข้อ 5:** ใช้ pluck แทน map เมื่อต้องการ raw values
```ruby
# เฉลย
# ❌ ช้า
User.all.map(&:email)

# ✅ เร็ว
User.pluck(:email)
```

### ระดับกลาง

**ข้อ 6:** แก้ไข query ให้ใช้ eager_load เมื่อต้อง filter ด้วย association
```ruby
# เฉลย
# ต้องการ filter articles โดย user role
# ❌ ช้า
Article.includes(:user).where(users: { role: 'author' })
# อาจ error

# ✅ ใช้ eager_load
Article.eager_load(:user).where(users: { role: 'author' })
```

**ข้อ 7:** เพิ่ม fragment caching สำหรับ expensive view
```erb
<%# เฉลย %>
<% cache ["sidebar", Article.maximum(:updated_at)] do %>
  <%= render 'shared/popular_articles' %>
<% end %>
```

**ข้อ 8:** ใช้ find_each สำหรับ batch processing
```ruby
# เฉลย
# ❌ Load ทั้งหมดใน memory
User.all.each { |u| send_newsletter(u) }

# ✅ Process in batches
User.find_each(batch_size: 100) { |u| send_newsletter(u) }
```

**ข้อ 9:** setup Rack Mini Profiler
```ruby
# เฉลย
# Gemfile (development): gem 'rack-mini-profiler'
# ไม่ต้องมี config - ทำงานได้เลย
# เห็น profiler badge ที่ http://localhost:3000
```

**ข้อ 10:** ปรับ database pool size ให้เหมาะกับ Puma threads
```yaml
# เฉลย - config/database.yml
production:
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

### ระดับสูง

**ข้อ 11-20:** (แบบฝึกหัดเพิ่มเติม)

```ruby
# เฉลย ข้อ 11 - Query analysis
Article.where(published: true).explain
# ดู Seq Scan vs Index Scan

# เพิ่ม index ที่เหมาะสม
add_index :articles, :published
add_index :articles, [:published, :created_at]
```

```ruby
# เฉลย ข้อ 14 - Memory profiling
require 'memory_profiler'

report = MemoryProfiler.report do
  Article.includes(:user, :comments).limit(100).each do |a|
    a.user.name
    a.comments.count
  end
end

report.pretty_print
```

```ruby
# เฉลย ข้อ 17 - SQL bulk insert
# ❌ N queries
articles_data.each do |data|
  Article.create!(data)
end

# ✅ Single insert
Article.insert_all(articles_data)

# ✅ Upsert
Article.upsert_all(articles_data, unique_by: :slug)
```

```ruby
# เฉลย ข้อ 19 - Database view สำหรับ complex queries
# Migration
execute <<-SQL
  CREATE VIEW article_stats AS
  SELECT 
    a.id,
    a.title,
    a.user_id,
    COUNT(c.id) as comments_count,
    SUM(l.id) as likes_count,
    a.views_count
  FROM articles a
  LEFT JOIN comments c ON c.article_id = a.id
  LEFT JOIN likes l ON l.article_id = a.id
  WHERE a.published = true
  GROUP BY a.id, a.title, a.user_id, a.views_count;
SQL

# Model
class ArticleStat < ApplicationRecord
  self.table_name = 'article_stats'
  belongs_to :user
  
  def readonly?
    true  # View ไม่ write ได้
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **N+1 Queries** - ปัญหาที่พบบ่อยที่สุด แก้ด้วย includes
2. **Bullet Gem** - auto-detect N+1 queries
3. **Database Indexes** - เพิ่ม indexes ที่เหมาะสม
4. **Query Optimization** - includes vs preload vs eager_load
5. **counter_cache** - ลด queries ด้วย cached counters
6. **Background Jobs** - ย้าย heavy tasks ออกจาก request cycle
7. **Fragment Caching** - cache views ที่ render ช้า
8. **HTTP Caching** - ETags และ Cache-Control headers
9. **Rack Mini Profiler** - วัด performance ใน development
10. **Connection Pooling** - Puma + PgBouncer
11. **Memory Optimization** - find_each, bulk operations

**Performance Checklist:**
- [ ] ไม่มี N+1 queries (ใช้ Bullet)
- [ ] มี indexes ที่เหมาะสม
- [ ] ใช้ counter_cache สำหรับ counts ที่ใช้บ่อย
- [ ] Cache expensive queries และ view fragments
- [ ] Heavy tasks อยู่ใน background jobs
- [ ] Database pool size เหมาะกับ Puma threads
- [ ] Monitor performance ด้วย tools อย่าง New Relic หรือ Skylight

Performance optimization เป็นกระบวนการต่อเนื่อง ควร measure ก่อน optimize เสมอ อย่า optimize สิ่งที่ยังไม่ได้วัด
