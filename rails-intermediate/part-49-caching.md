# ตอนที่ 49: Caching (Steps 1081-1100)

## บทนำ

Caching คือการเก็บข้อมูลที่คำนวณแล้วไว้ใน storage ที่เข้าถึงได้เร็ว เพื่อไม่ต้องคำนวณซ้ำในครั้งถัดไป Rails มีระบบ caching ที่ครอบคลุมหลายระดับตั้งแต่ HTTP caching จนถึง low-level fragment caching

---

## Step 1081: ประเภทของ Cache ใน Rails

### 1. Page Caching (เก่า, ไม่แนะนำใน Rails 7)
เก็บ HTML ทั้งหน้าเป็น static file ไม่ผ่าน Rails stack

### 2. Action Caching (เก่า, ไม่แนะนำ)
เก็บ output ของ controller action

### 3. Fragment Caching (แนะนำ)
เก็บบางส่วนของ view

### 4. HTTP Caching
ใช้ HTTP headers (ETags, Last-Modified, Cache-Control)

### 5. Low-level Caching
`Rails.cache.fetch`, `read`, `write`, `delete`

---

## Step 1082: Cache Store Options

### Memory Store (Development default)

```ruby
# config/environments/development.rb
config.cache_store = :memory_store, { size: 64.megabytes }
```

### File Store

```ruby
config.cache_store = :file_store, "/tmp/rails_cache"
```

### Redis Cache Store (Production recommended)

```ruby
# Gemfile
gem 'redis'
gem 'redis-client'

# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV.fetch("REDIS_URL"),
  expires_in: 1.hour,
  namespace: "myapp_cache",
  
  # Connection pool
  pool_size: 5,
  pool_timeout: 5,
  
  # Error handling
  error_handler: -> (method:, returning:, exception:) {
    Sentry.capture_exception(exception, 
      extra: { method: method, returning: returning }
    )
  }
}
```

### Memcache Store

```ruby
# Gemfile
gem 'dalli'

config.cache_store = :mem_cache_store, 
  "localhost:11211",
  { expires_in: 1.hour, compress: true }
```

### Null Store (Testing)

```ruby
# ไม่ cache อะไรเลย - สำหรับ testing
config.cache_store = :null_store
```

---

## Step 1083: เปิด Cache ใน Development

```bash
# เปิด caching ใน development mode
rails dev:cache
# => Development mode is now being cached.

# ปิด caching
rails dev:cache
# => Development mode is no longer being cached.
```

```ruby
# config/environments/development.rb
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
```

---

## Step 1084: Rails.cache API

### Basic Operations

```ruby
# Write to cache
Rails.cache.write("key", "value")
Rails.cache.write("key", "value", expires_in: 1.hour)
Rails.cache.write("key", { data: complex_object }, expires_in: 30.minutes)

# Read from cache
Rails.cache.read("key")
# => "value" หรือ nil ถ้าหมดอายุหรือไม่มี

# Delete from cache
Rails.cache.delete("key")

# Check existence
Rails.cache.exist?("key")

# Fetch (Read or Write)
Rails.cache.fetch("expensive_calculation") do
  # Block นี้จะทำงานเฉพาะเมื่อ cache miss
  perform_expensive_calculation()
end

# Fetch with expiry
Rails.cache.fetch("user_stats_#{user.id}", expires_in: 30.minutes) do
  calculate_user_statistics(user)
end
```

### Cache Fetch Pattern

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  def self.popular_articles
    Rails.cache.fetch("popular_articles", expires_in: 15.minutes) do
      includes(:user)
        .where(published: true)
        .order(views_count: :desc)
        .limit(10)
        .to_a  # .to_a สำคัญมาก! เพื่อ materialize query
    end
  end
  
  def statistics
    Rails.cache.fetch("article_stats_#{id}_#{updated_at.to_i}", expires_in: 1.hour) do
      {
        views: views_count,
        comments: comments.count,
        likes: likes.count,
        reading_time: calculate_reading_time,
        share_count: social_shares.sum(:count)
      }
    end
  end
end
```

### Atomic Operations

```ruby
# Increment counter
Rails.cache.increment("page_views")
Rails.cache.increment("page_views", 5)  # เพิ่มทีละ 5

# Decrement counter
Rails.cache.decrement("stock_count")

# Read Multiple
values = Rails.cache.read_multi("key1", "key2", "key3")
# => { "key1" => "v1", "key2" => nil, "key3" => "v3" }

# Write Multiple
Rails.cache.write_multi({
  "key1" => "value1",
  "key2" => "value2"
}, expires_in: 1.hour)

# Delete Multiple
Rails.cache.delete_multi("key1", "key2")

# Delete by Pattern (Redis only)
Rails.cache.delete_matched("user_*")
```

---

## Step 1085: Fragment Caching ใน Views

### Basic Fragment Cache

```erb
<%# app/views/articles/index.html.erb %>

<% cache do %>
  <%# Block นี้จะถูก cache %>
  <% @articles.each do |article| %>
    <%= render article %>
  <% end %>
<% end %>
```

### Cache with Key

```erb
<%# Cache ด้วย key ที่ specific %>
<% cache "articles/all" do %>
  ...
<% end %>

<%# Cache ด้วย object (ใช้ cache_key) %>
<% cache @article do %>
  <h1><%= @article.title %></h1>
  <div><%= @article.body %></div>
<% end %>
```

### cache_key ของ ActiveRecord

```ruby
# Rails สร้าง cache_key จาก model name + id + updated_at
article.cache_key
# => "articles/1-20240101120000000000"

# Cache expires อัตโนมัติเมื่อ record update
@article.touch  # อัปเดต updated_at → cache invalidate
```

### Fragment Cache ใน Partial

```erb
<%# app/views/articles/_article.html.erb %>
<% cache article do %>
  <article>
    <h2><%= article.title %></h2>
    <p><%= article.excerpt %></p>
    <span>By <%= article.user.name %></span>
    <time><%= article.created_at.strftime("%d %b %Y") %></time>
  </article>
<% end %>
```

---

## Step 1086: Russian Doll Caching

### หลักการ

Russian Doll Caching คือการ nest cache ซ้อนกัน โดย inner cache จะ invalidate เมื่อ record update และจะ invalidate outer cache ด้วย

```erb
<%# Outer cache - article list %>
<% cache ["articles/list", @articles.maximum(:updated_at)] do %>
  
  <% @articles.each do |article| %>
    
    <%# Inner cache - individual article %>
    <% cache article do %>
      <div class="article">
        <h2><%= article.title %></h2>
        
        <%# Deeper cache - comments count %>
        <% cache ["comments/count", article.id, article.comments.maximum(:updated_at)] do %>
          <span><%= article.comments.count %> comments</span>
        <% end %>
      </div>
    <% end %>
    
  <% end %>
  
<% end %>
```

### Touch สำหรับ Propagation

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :article, touch: true  # อัปเดต article.updated_at เมื่อ comment เปลี่ยน
end

# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :user, touch: true  # อัปเดต user.updated_at
  has_many :comments
end
```

เมื่อ comment ถูกสร้าง → article.updated_at update → article cache invalidate → outer cache invalidate

---

## Step 1087: Low-level Caching

### Cache Expensive Queries

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def self.leaderboard
    Rails.cache.fetch("users/leaderboard", expires_in: 5.minutes) do
      joins(:articles)
        .group("users.id")
        .select("users.*, COUNT(articles.id) as articles_count, SUM(articles.views_count) as total_views")
        .order("total_views DESC")
        .limit(20)
        .to_a
    end
  end
  
  def monthly_stats
    key = "user/#{id}/monthly_stats/#{Date.today.strftime('%Y-%m')}"
    Rails.cache.fetch(key, expires_in: 1.hour) do
      articles_this_month = articles.where(created_at: Time.current.beginning_of_month..Time.current.end_of_month)
      {
        articles_count: articles_this_month.count,
        total_views: articles_this_month.sum(:views_count),
        avg_views: articles_this_month.average(:views_count).to_f.round(2),
        top_article: articles_this_month.order(views_count: :desc).first
      }
    end
  end
end
```

### Service Object Caching

```ruby
# app/services/analytics_service.rb
class AnalyticsService
  CACHE_DURATION = 30.minutes
  
  def initialize(date_range = 30.days)
    @date_range = date_range
  end
  
  def overview
    Rails.cache.fetch(cache_key("overview"), expires_in: CACHE_DURATION) do
      {
        total_users: User.count,
        new_users: User.where("created_at > ?", @date_range.ago).count,
        total_articles: Article.count,
        published_articles: Article.published.count,
        total_views: Article.sum(:views_count),
        avg_views_per_article: Article.average(:views_count).to_f.round(2)
      }
    end
  end
  
  def popular_articles
    Rails.cache.fetch(cache_key("popular_articles"), expires_in: CACHE_DURATION) do
      Article.published
             .where("created_at > ?", @date_range.ago)
             .order(views_count: :desc)
             .limit(10)
             .includes(:user)
             .map { |a| { id: a.id, title: a.title, views: a.views_count, author: a.user.name } }
    end
  end
  
  def invalidate_all!
    Rails.cache.delete_matched("analytics/*")
  end
  
  private
  
  def cache_key(stat)
    "analytics/#{stat}/#{@date_range.inspect}"
  end
end
```

---

## Step 1088: HTTP Caching

### ETags

```ruby
# app/controllers/api/v1/articles_controller.rb
def show
  @article = Article.find(params[:id])
  
  # Fresh when: ETag matches (stale? returns false)
  if stale?(etag: @article, last_modified: @article.updated_at)
    render json: ArticleSerializer.new(@article).as_json
  end
  # ถ้า stale? returns false → Rails return 304 Not Modified อัตโนมัติ
end

# หรือใช้ fresh_when
def show
  @article = Article.find(params[:id])
  fresh_when(etag: @article, last_modified: @article.updated_at, public: true)
  # ยังคง render view ถ้า not cached
end
```

### Cache-Control Headers

```ruby
# config/environments/production.rb

# Public pages
def show
  @article = Article.find(params[:id])
  expires_in 1.hour, public: true
  render json: @article
end

# Private (per-user)
def dashboard
  expires_in 5.minutes, public: false
  render json: current_user.dashboard_data
end

# No cache
def real_time_data
  expires_now
  render json: @data
end
```

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  def index
    @articles = Article.published
    
    # Cache for 1 hour for all users
    expires_in 1.hour, public: true
    
    # Vary ตาม Accept-Language header
    response.headers['Vary'] = 'Accept-Language'
    
    render json: @articles.as_json
  end
  
  def show
    @article = Article.find(params[:id])
    
    # Cache-Control สำหรับ CDN
    response.headers['Cache-Control'] = 'public, max-age=3600, s-maxage=86400'
    response.headers['Surrogate-Key'] = "article-#{@article.id}"
    
    render json: @article
  end
end
```

---

## Step 1089: Cache Digests

### Cache Digest คืออะไร

Rails สร้าง digest (hash) ของ template file โดยอัตโนมัติ และรวมใน cache key ทำให้เมื่อ template เปลี่ยน → cache key เปลี่ยน → cache invalidate อัตโนมัติ

```erb
<%# Rails สร้าง cache key อัตโนมัติ เช่น %>
<%# views/articles/show:abc123/articles/1-20240101 %>
<% cache @article do %>
  ...
<% end %>
```

### ดู Cache Key ที่สร้าง

```ruby
# ใน rails console
view_cache_dependencies = ActionView::Helpers::CacheHelper
# หรือดูใน logs เมื่อ config.action_controller.enable_fragment_cache_logging = true
```

---

## Step 1090: Redis เป็น Cache Store

### Setup Redis Cache

```ruby
# Gemfile
gem 'redis', '~> 5.0'

# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV.fetch('REDIS_CACHE_URL') { ENV.fetch('REDIS_URL') { 'redis://localhost:6379/1' } },
  
  # Separate DB number จาก Sidekiq (ที่ใช้ DB 0)
  # Redis URLs: redis://localhost:6379/0 (Sidekiq), redis://localhost:6379/1 (Cache)
  
  expires_in: 1.hour,
  namespace: "#{Rails.application.class.module_parent_name.downcase}_cache",
  compress: true,
  compress_threshold: 1.kilobyte,
  
  pool_size: ENV.fetch("RAILS_MAX_THREADS") { 5 }.to_i,
  pool_timeout: 5
}
```

### Cache Namespacing

```ruby
# ป้องกัน key collision ระหว่าง environments
# Development cache keys: dev_myapp_cache:articles/1
# Production cache keys: prod_myapp_cache:articles/1

config.cache_store = :redis_cache_store, {
  url: ENV['REDIS_URL'],
  namespace: "#{Rails.env}_myapp_cache"
}
```

---

## Step 1091: Cache Performance

### Monitoring Cache

```ruby
# config/initializers/cache_monitoring.rb
ActiveSupport::Notifications.subscribe("cache_read.active_support") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  
  if event.payload[:hit]
    StatsD.increment('cache.hit', tags: ["key:#{event.payload[:key]}"])
  else
    StatsD.increment('cache.miss', tags: ["key:#{event.payload[:key]}"])
  end
end
```

### Cache Hit Rate

```ruby
# ใน rails console
# ดู Redis info
redis = Redis.new
info = redis.info

puts "Used memory: #{info['used_memory_human']}"
puts "Keyspace hits: #{info['keyspace_hits']}"
puts "Keyspace misses: #{info['keyspace_misses']}"

total = info['keyspace_hits'].to_i + info['keyspace_misses'].to_i
hit_rate = total > 0 ? (info['keyspace_hits'].to_f / total * 100).round(2) : 0
puts "Hit rate: #{hit_rate}%"
```

---

## Step 1092: Cache Invalidation Strategies

### Time-based Expiration

```ruby
# Expire หลัง 30 นาที
Rails.cache.write("data", value, expires_in: 30.minutes)
```

### Event-based Invalidation

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  after_save :invalidate_cache
  after_destroy :invalidate_cache
  
  def invalidate_cache
    Rails.cache.delete("article/#{id}/stats")
    Rails.cache.delete("articles/popular")
    Rails.cache.delete("articles/recent")
    Rails.cache.delete("users/#{user_id}/stats")
  end
end
```

### Cache Versioning

```ruby
# แทนที่จะ delete, ใช้ version key
class Article < ApplicationRecord
  def cache_version
    "v#{updated_at.to_i}"
  end
  
  def cached_stats
    Rails.cache.fetch("article/#{id}/stats/#{cache_version}", expires_in: 24.hours) do
      calculate_stats
    end
  end
end
```

---

## Step 1093: Rack Mini Profiler Integration

```ruby
# Gemfile (development)
gem 'rack-mini-profiler'
gem 'flamegraph'  # optional - for flame graphs
gem 'stackprof'   # optional
gem 'memory_profiler' # optional

# config/initializers/profiler.rb (development only)
if Rails.env.development?
  require 'rack-mini-profiler'
  
  Rack::MiniProfiler.config.position = 'bottom-right'
  Rack::MiniProfiler.config.start_hidden = false
  Rack::MiniProfiler.config.skip_paths = ['/assets']
end
```

---

## Step 1094: Complete Caching Example

### Full Application Caching Setup

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  def index
    # Level 1: HTTP Cache
    if stale?(etag: Article.maximum(:updated_at), last_modified: Article.maximum(:updated_at), public: true)
      
      # Level 2: Fragment Cache
      @articles = Rails.cache.fetch("articles/published/page/#{params[:page]}", expires_in: 10.minutes) do
        Article.published.includes(:user).order(created_at: :desc).page(params[:page]).to_a
      end
      
      render :index
    end
  end
  
  def show
    @article = Article.find(params[:id])
    
    # Conditional GET support
    if stale?(etag: @article, last_modified: @article.updated_at, public: @article.published?)
      @article.increment_views!
      render :show
    end
  end
end
```

```erb
<%# app/views/articles/index.html.erb %>
<%# Level 3: View Fragment Cache %>

<% cache ["articles/list", @articles.map(&:cache_key_with_version).join] do %>
  <div class="articles">
    <% @articles.each do |article| %>
      <%# Russian Doll - inner cache %>
      <% cache article do %>
        <%= render 'article', article: article %>
      <% end %>
    <% end %>
  </div>
<% end %>
```

---

## แบบฝึกหัด (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** เปิด caching ใน development และ verify ว่าทำงาน
```bash
# เฉลย
rails dev:cache
# สังเกต log: "Cache read: articles/all"
```

**ข้อ 2:** เขียน code ที่ cache ผลลัพธ์ของ query ที่ใช้เวลานาน
```ruby
# เฉลย
def expensive_data
  Rails.cache.fetch("expensive_data", expires_in: 1.hour) do
    User.joins(:articles).where(articles: { published: true })
        .select("users.*, COUNT(*) as article_count")
        .group("users.id")
        .order("article_count DESC")
        .to_a
  end
end
```

**ข้อ 3:** เพิ่ม Fragment caching ใน article list view
```erb
<%# เฉลย %>
<% cache ["articles", @articles.maximum(:updated_at)] do %>
  <% @articles.each do |article| %>
    <% cache article do %>
      <%= render article %>
    <% end %>
  <% end %>
<% end %>
```

**ข้อ 4:** ตั้งค่า Redis เป็น cache store ใน production
```ruby
# เฉลย
config.cache_store = :redis_cache_store, {
  url: ENV['REDIS_URL'],
  expires_in: 1.hour,
  namespace: "myapp_cache"
}
```

**ข้อ 5:** เขียน model callback ที่ invalidate cache เมื่อ record update
```ruby
# เฉลย
class Product < ApplicationRecord
  after_save :bust_cache
  after_destroy :bust_cache
  
  private
  
  def bust_cache
    Rails.cache.delete("product_#{id}")
    Rails.cache.delete("products_all")
    Rails.cache.delete("products_featured")
  end
end
```

### ระดับกลาง

**ข้อ 6:** implement Russian Doll caching สำหรับ article กับ comments
```erb
<%# เฉลย %>
<% cache ["article", article, article.comments.maximum(:updated_at)] do %>
  <h1><%= article.title %></h1>
  <% article.comments.each do |comment| %>
    <% cache comment do %>
      <p><%= comment.body %></p>
    <% end %>
  <% end %>
<% end %>
```

**ข้อ 7:** สร้าง HTTP caching ด้วย ETags สำหรับ articles API
```ruby
# เฉลย
def show
  @article = Article.find(params[:id])
  if stale?(etag: @article, last_modified: @article.updated_at, public: true)
    render json: ArticleSerializer.new(@article).as_json
  end
end
```

**ข้อ 8:** เพิ่ม touch: true ใน association เพื่อ propagate cache invalidation
```ruby
# เฉลย
class Comment < ApplicationRecord
  belongs_to :article, touch: true
end

class Article < ApplicationRecord
  belongs_to :user, touch: true
end
```

**ข้อ 9:** implement cache warming ด้วย background job
```ruby
# เฉลย
class WarmCacheJob < ApplicationJob
  def perform
    # Pre-load popular pages
    Article.popular.each do |article|
      Rails.cache.fetch("article_#{article.id}", expires_in: 2.hours) do
        ArticleSerializer.new(article).as_json
      end
    end
  end
end
```

**ข้อ 10:** สร้าง counter cache สำหรับ comments count
```ruby
# เฉลย
# migration
add_column :articles, :comments_count, :integer, default: 0, null: false

class Comment < ApplicationRecord
  belongs_to :article, counter_cache: true
end

# ใช้ใน serializer
def comments_count
  object.comments_count  # ไม่ query database
end
```

### ระดับสูง

**ข้อ 11-20:** (แบบฝึกหัดเพิ่มเติม)

**ข้อ 11:** implement cache stampede prevention (dog-pile effect)
**ข้อ 12:** สร้าง distributed cache invalidation system
**ข้อ 13:** implement conditional caching ตาม user role
**ข้อ 14:** เพิ่ม cache monitoring และ alerting
**ข้อ 15:** implement multi-level caching (memory + Redis)

```ruby
# เฉลย ข้อ 11 - Cache Stampede Prevention
class CacheService
  LOCK_TIMEOUT = 10.seconds
  
  def self.fetch(key, expires_in:, &block)
    # ลอง read ก่อน
    cached = Rails.cache.read(key)
    return cached if cached
    
    # ถ้า miss, lock และ compute
    lock_key = "#{key}_lock"
    if $redis.set(lock_key, 1, nx: true, ex: LOCK_TIMEOUT.to_i)
      begin
        value = block.call
        Rails.cache.write(key, value, expires_in: expires_in)
        value
      ensure
        $redis.del(lock_key)
      end
    else
      # รอ lock release แล้ว read
      sleep(0.1)
      Rails.cache.read(key) || block.call
    end
  end
end
```

```ruby
# เฉลย ข้อ 13 - Conditional Caching
class ArticlesCachingStrategy
  def self.cache_key_for(article, user = nil)
    if user&.admin?
      "article/#{article.id}/admin/#{article.cache_key_with_version}"
    elsif user&.premium?
      "article/#{article.id}/premium/#{article.cache_key_with_version}"
    else
      "article/#{article.id}/public/#{article.cache_key_with_version}"
    end
  end
  
  def self.fetch(article, user = nil, &block)
    key = cache_key_for(article, user)
    ttl = user&.admin? ? 5.minutes : 1.hour
    Rails.cache.fetch(key, expires_in: ttl, &block)
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Cache Types** - page, action, fragment, HTTP
2. **Cache Store** - memory, Redis, memcache
3. **Rails.cache API** - fetch, read, write, delete
4. **Fragment Caching** - cache blocks ใน views
5. **Russian Doll Caching** - nested caches
6. **HTTP Caching** - ETags, Last-Modified
7. **Cache Digests** - automatic expiry on template change
8. **Redis Cache Store** - production setup

**กฎทอง:** Cache ข้อมูลที่คำนวณแล้วและไม่เปลี่ยนบ่อย แต่ระวัง stale data ที่อาจแสดงข้อมูลเก่าให้ users
