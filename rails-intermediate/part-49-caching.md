# Part 49: Caching ใน Rails

## ขั้นตอนที่ 1081-1100: การ Cache ข้อมูลเพื่อประสิทธิภาพ

---

## ขั้นตอนที่ 1081: ทำไมต้องใช้ Cache?

Cache ช่วยลดเวลาในการตอบสนองโดย:
- เก็บผลลัพธ์ที่คำนวณแล้ว
- ลด database queries
- ลด CPU usage
- รองรับ traffic สูงขึ้น

```
Without caching:
User Request → Controller → Database → Serialize → Response (500ms)

With caching:
User Request → Controller → Cache HIT → Response (10ms)
                         → Cache MISS → Database → Cache SET → Response (510ms)
```

## ขั้นตอนที่ 1082: Cache Types ใน Rails

### 1. Page Caching (ไม่มีใน Rails ตั้งแต่ v4)
```ruby
# gem 'actionpack-page_caching'
# cache ทั้งหน้าเป็น HTML file
```

### 2. Action Caching (ไม่มีใน Rails ตั้งแต่ v4)
```ruby
# gem 'actionpack-action_caching'
```

### 3. Fragment Caching (ใช้บ่อยที่สุด)
```erb
<%# Cache ส่วนหนึ่งของ view %>
<% cache @post do %>
  <article>
    <h1><%= @post.title %></h1>
    <p><%= @post.content %></p>
  </article>
<% end %>
```

### 4. Low-level Caching
```ruby
# Cache ข้อมูลใดๆ ใน Rails.cache
value = Rails.cache.fetch("key") { compute_value }
```

### 5. HTTP Caching
```ruby
# ETag และ Last-Modified headers
def show
  @post = Post.find(params[:id])
  if stale?(@post)
    render :show
  end
end
```

## ขั้นตอนที่ 1083: Cache Store Configuration

```ruby
# config/environments/development.rb
config.cache_store = :memory_store, { size: 64.megabytes }

# config/environments/test.rb
config.cache_store = :null_store  # ไม่ cache ใน test

# config/environments/production.rb
# Memory store (single server)
config.cache_store = :memory_store, { size: 256.megabytes }

# File store
config.cache_store = :file_store, "/path/to/cache/directory"

# Redis cache store (recommended สำหรับ production)
config.cache_store = :redis_cache_store, {
  url: ENV['REDIS_URL'],
  connect_timeout: 30,
  read_timeout: 0.2,
  write_timeout: 0.2,
  reconnect_attempts: 1
}

# Memcache
config.cache_store = :mem_cache_store, "cache-1.example.com", "cache-2.example.com", {
  connect_timeout: 2,
  read_timeout: 1,
  write_timeout: 1
}
```

```ruby
# Redis cache store ขั้นสูง
config.cache_store = :redis_cache_store, {
  url: ENV['REDIS_URL'],
  pool_size: 5,
  pool_timeout: 5,
  namespace: "myapp_cache",
  expires_in: 1.day,
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error "Redis cache error: #{exception.message}"
    Sentry.capture_exception(exception)
  }
}
```

## ขั้นตอนที่ 1084: Rails.cache API

```ruby
# เขียน cache
Rails.cache.write("user:1", user_data)
Rails.cache.write("user:1", user_data, expires_in: 30.minutes)
Rails.cache.write("user:1", user_data, namespace: "v2")

# อ่าน cache
value = Rails.cache.read("user:1")
values = Rails.cache.read_multi("user:1", "user:2", "user:3")

# fetch - อ่านหรือคำนวณถ้าไม่มี
user_data = Rails.cache.fetch("user:#{user.id}", expires_in: 1.hour) do
  # Block นี้จะทำงานเมื่อ cache miss
  UserSerializer.new(user).serializable_hash
end

# delete
Rails.cache.delete("user:1")
Rails.cache.delete_matched("user:*")  # ลบตาม pattern

# exist?
Rails.cache.exist?("user:1")

# increment/decrement (atomic)
Rails.cache.increment("page_views:#{post.id}")
Rails.cache.decrement("countdown")
Rails.cache.increment("downloads", 1, expires_in: 1.day)

# fetch_multi
data = Rails.cache.fetch_multi(*user_ids.map { |id| "user:#{id}" }) do |key|
  user_id = key.split(':').last
  User.find(user_id).as_json
end
```

## ขั้นตอนที่ 1085: Fragment Caching

```erb
<%# Cache ด้วย object key %>
<% cache @post do %>
  <article>
    <h1><%= @post.title %></h1>
    <%= @post.content %>
  </article>
<% end %>
```

```ruby
# cache key จะเป็น "posts/1-20240101120000"
# ซึ่งจะ expire อัตโนมัติเมื่อ post.updated_at เปลี่ยน
```

```erb
<%# Cache collection %>
<% cache @posts do %>
  <% @posts.each do |post| %>
    <%= render post %>
  <% end %>
<% end %>
```

```erb
<%# Custom cache key %>
<% cache ["user_nav", current_user] do %>
  <nav>
    <% current_user.menu_items.each do |item| %>
      <%= link_to item.name, item.path %>
    <% end %>
  </nav>
<% end %>
```

```erb
<%# Cache with expiry %>
<% cache @post, expires_in: 1.hour do %>
  <%= render @post %>
<% end %>
```

## ขั้นตอนที่ 1086: Low-level Caching

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  def similar_posts
    Rails.cache.fetch("#{cache_key}/similar_posts", expires_in: 1.hour) do
      # Query ที่ expensive
      Post.where(category: category)
          .where.not(id: id)
          .order(views_count: :desc)
          .limit(5)
          .to_a  # จำเป็นต้อง materialize array ก่อน cache
    end
  end
  
  def tag_cloud
    Rails.cache.fetch("tag_cloud", expires_in: 30.minutes) do
      Tag.joins(:posts)
         .group('tags.id')
         .select('tags.*, COUNT(posts.id) as posts_count')
         .order('posts_count DESC')
         .to_a
    end
  end
  
  def self.trending(period: 7.days, limit: 10)
    cache_key = "trending_posts:#{period.to_i}:#{limit}"
    
    Rails.cache.fetch(cache_key, expires_in: 1.hour) do
      joins(:post_views)
        .where('post_views.created_at > ?', period.ago)
        .group('posts.id')
        .order('COUNT(post_views.id) DESC')
        .limit(limit)
        .to_a
    end
  end
end
```

```ruby
# Caching ใน controller
class HomeController < ApplicationController
  def index
    @featured_posts = cache_featured_posts
    @recent_posts = cache_recent_posts
    @statistics = cache_statistics
  end
  
  private
  
  def cache_featured_posts
    Rails.cache.fetch("featured_posts", expires_in: 30.minutes) do
      Post.featured.published.includes(:user, :tags).limit(5).to_a
    end
  end
  
  def cache_recent_posts
    Rails.cache.fetch("recent_posts", expires_in: 10.minutes) do
      Post.published.order(created_at: :desc).limit(10).to_a
    end
  end
  
  def cache_statistics
    Rails.cache.fetch("site_stats", expires_in: 1.hour) do
      {
        total_posts: Post.published.count,
        total_users: User.active.count,
        total_comments: Comment.approved.count
      }
    end
  end
end
```

## ขั้นตอนที่ 1087: Russian Doll Caching

Russian Doll Caching คือการ cache ซ้อนกันหลายชั้น

```erb
<%# app/views/posts/index.html.erb %>
<% cache @posts do %>
  <% @posts.each do |post| %>
    <% cache post do %>
      <%# Inner cache - expire เมื่อ post เปลี่ยน %>
      <article>
        <h2><%= post.title %></h2>
        
        <% cache [post, "comments"] do %>
          <%# Innermost cache - expire เมื่อ comments เปลี่ยน %>
          <p><%= post.comments.approved.count %> ความเห็น</p>
          
          <% post.comments.approved.each do |comment| %>
            <% cache comment do %>
              <div>
                <strong><%= comment.user.name %>:</strong>
                <%= comment.body %>
              </div>
            <% end %>
          <% end %>
        <% end %>
      </article>
    <% end %>
  <% end %>
<% end %>
```

```ruby
# touch: true - update parent เมื่อ child เปลี่ยน
class Comment < ApplicationRecord
  belongs_to :post, touch: true  # อัพเดท post.updated_at เมื่อ comment เปลี่ยน
  belongs_to :user
end

class Post < ApplicationRecord
  belongs_to :category, touch: true  # อัพเดท category.updated_at
end
```

## ขั้นตอนที่ 1088: HTTP Caching

```ruby
# ETag-based caching
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    
    # ตรวจสอบว่า client มี version ล่าสุดหรือไม่
    if stale?(@post, public: false)
      render :show
    end
    # ถ้า not stale จะ return 304 Not Modified อัตโนมัติ
  end
  
  def index
    @posts = Post.published.order(updated_at: :desc)
    
    if stale?(last_modified: @posts.maximum(:updated_at), etag: @posts)
      render :index
    end
  end
end
```

```ruby
# Manual ETags
class ApiPostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    
    # Custom ETag
    etag = Digest::MD5.hexdigest("#{@post.updated_at}-#{current_user&.id}")
    
    if request.fresh?(etag: etag)
      head :not_modified
    else
      response.headers['ETag'] = etag
      response.headers['Last-Modified'] = @post.updated_at.httpdate
      response.headers['Cache-Control'] = 'private, max-age=0, must-revalidate'
      
      render json: PostSerializer.new(@post).serializable_hash
    end
  end
end
```

```ruby
# Public cache headers
def show
  @post = Post.find(params[:id])
  
  if @post.published?
    # Public content - CDN สามารถ cache ได้
    expires_in 1.hour, public: true
    fresh_when @post, public: true
  else
    # Private content - เฉพาะ browser
    expires_in 0, must_revalidate: true
    fresh_when @post, public: false
  end
  
  render :show
end
```

## ขั้นตอนที่ 1089: Cache Expiration Strategies

```ruby
# 1. Time-based expiration
Rails.cache.write("key", value, expires_in: 30.minutes)

# 2. Version-based expiration (Russian Doll)
cache_key = "post/#{post.id}/#{post.updated_at.to_i}"
Rails.cache.fetch(cache_key) { compute_value }

# 3. Manual expiration
# หลัง update ให้ expire cache
after_save :expire_caches

def expire_caches
  Rails.cache.delete("post/#{id}/similar")
  Rails.cache.delete("post/#{id}/tags")
  ActionController::Base.new.expire_fragment("post_#{id}")
end

# 4. Counter-based expiration
def cache_key_with_version
  "#{super}/#{comments_count}"
end
```

```ruby
# app/models/concerns/cacheable.rb
module Cacheable
  extend ActiveSupport::Concern
  
  included do
    after_commit :expire_cache
  end
  
  def cache_fragment(name = nil, **options, &block)
    key = cache_key_for(name)
    Rails.cache.fetch(key, **options, &block)
  end
  
  def expire_cache
    Rails.cache.delete_matched("#{cache_key_prefix}*")
  end
  
  private
  
  def cache_key_for(name)
    parts = [cache_key_prefix, name].compact
    parts.join('/')
  end
  
  def cache_key_prefix
    "#{self.class.name.underscore}/#{id}"
  end
end
```

## ขั้นตอนที่ 1090: Redis Cache Store

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV['REDIS_URL'],
  
  # Connection pool
  pool_size: ActiveRecord::Base.connection_pool.size,
  pool_timeout: 5,
  
  # Expiration
  expires_in: 1.day,
  
  # Compression
  compress: true,
  compress_threshold: 1.kilobyte,
  
  # Error handling
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error "Redis cache error on #{method}: #{exception}"
  },
  
  # SSL (สำหรับ production)
  ssl_params: { verify_mode: OpenSSL::SSL::VERIFY_NONE }
}
```

```ruby
# Redis-specific operations
redis = Redis.new(url: ENV['REDIS_URL'])

# Lists
redis.rpush("queue", "item1")
redis.lpop("queue")

# Sets
redis.sadd("users_online", user_id)
redis.smembers("users_online")

# Sorted sets (leaderboard)
redis.zadd("leaderboard", score, user_id)
redis.zrevrange("leaderboard", 0, 9, with_scores: true)

# Pub/Sub
redis.publish("channel", { event: "new_post", id: post.id }.to_json)
```

## ขั้นตอนที่ 1091: Cache Keys Best Practices

```ruby
# ❌ Bad: cache key ที่อาจ conflict
Rails.cache.write("posts", posts_data)  # ไม่ระบุ version

# ✅ Good: cache key ที่มี version และ scope
Rails.cache.write("v2/posts/index/page_1", posts_data, expires_in: 5.minutes)

# ✅ ใช้ cache_key method ของ model
post.cache_key
# => "posts/1-20240101120000000"

post.cache_key_with_version
# => "posts/1-20240101120000000/5"

# ✅ Namespace
cache_key = [
  "api", "v2",
  "posts",
  "user", current_user&.id || "guest",
  "locale", I18n.locale,
  @post.cache_key
].join("/")
```

```ruby
# Auto-expiring keys
def cache_key_for_user(user)
  [
    "user",
    user.id,
    user.updated_at.to_i,
    "profile"
  ].join("/")
end

# Versioned keys
def cache_key_v2(resource)
  "v2/#{resource.class.name.underscore}/#{resource.id}/#{resource.updated_at.to_i}"
end
```

## ขั้นตอนที่ 1092: Cache Warming

```ruby
# app/jobs/warm_cache_job.rb
class WarmCacheJob < ApplicationJob
  queue_as :low
  
  def perform
    warm_homepage_cache
    warm_popular_posts_cache
    warm_tag_cloud_cache
  end
  
  private
  
  def warm_homepage_cache
    featured = Post.featured.published.includes(:user, :tags).limit(5).to_a
    Rails.cache.write("featured_posts", featured, expires_in: 1.hour)
    
    recent = Post.published.order(created_at: :desc).limit(10).to_a
    Rails.cache.write("recent_posts", recent, expires_in: 30.minutes)
  end
  
  def warm_popular_posts_cache
    Post.popular.limit(20).each do |post|
      Rails.cache.fetch("post/#{post.id}/similar", expires_in: 2.hours) do
        post.similar_posts
      end
    end
  end
  
  def warm_tag_cloud_cache
    Rails.cache.fetch("tag_cloud", expires_in: 1.hour) do
      Tag.with_posts_count.order(posts_count: :desc).limit(50).to_a
    end
  end
end

# Warm cache หลัง deploy
# config/deploy/after_deploy.rb
after :publishing do
  WarmCacheJob.perform_later
end
```

## ขั้นตอนที่ 1093: Conditional Caching

```ruby
# Cache เฉพาะเมื่อเหมาะสม
def show
  @post = Post.find(params[:id])
  
  if should_cache?
    @related_posts = Rails.cache.fetch("post/#{@post.id}/related", expires_in: 1.hour) do
      @post.related_posts.to_a
    end
  else
    @related_posts = @post.related_posts
  end
end

private

def should_cache?
  # ไม่ cache ถ้า admin อาจจะเห็น draft posts
  !current_user&.admin? && 
  # ไม่ cache ถ้า preview mode
  !params[:preview] &&
  # ไม่ cache ใน development
  !Rails.env.development?
end
```

## ขั้นตอนที่ 1094: Cache Performance Monitoring

```ruby
# config/initializers/cache_monitoring.rb
ActiveSupport::Notifications.subscribe(/cache_(.*)\.active_support/) do |event|
  name = event.name.split('.').first.split('_').last
  
  case name
  when 'read'
    StatsD.increment("cache.read.#{event.payload[:hit] ? 'hit' : 'miss'}")
  when 'write'
    StatsD.increment("cache.write")
  when 'delete'
    StatsD.increment("cache.delete")
  end
  
  StatsD.timing("cache.#{name}.duration", event.duration)
end

# Cache stats endpoint
class Admin::CacheController < AdminController
  def stats
    render json: {
      memory_size: Rails.cache.instance_variable_get(:@data)&.size,
      hit_rate: CacheStats.hit_rate,
      top_keys: CacheStats.top_accessed_keys
    }
  end
  
  def clear
    Rails.cache.clear
    redirect_to admin_cache_path, notice: "Cache ล้างแล้ว"
  end
  
  def delete_key
    Rails.cache.delete(params[:key])
    redirect_to admin_cache_path, notice: "ลบ key แล้ว"
  end
end
```

## ขั้นตอนที่ 1095: Caching Strategies ขั้นสูง

```ruby
# Read-through cache
class UserRepository
  def find(id)
    Rails.cache.fetch("user/#{id}", expires_in: 30.minutes) do
      User.includes(:roles, :settings).find(id)
    end
  end
  
  def update(user, attributes)
    user.update!(attributes)
    Rails.cache.delete("user/#{user.id}")
    user
  end
end

# Write-through cache
class PostService
  def create(user, attributes)
    post = user.posts.create!(attributes)
    cache_post(post)
    expire_related_caches(post)
    post
  end
  
  private
  
  def cache_post(post)
    Rails.cache.write(
      "post/#{post.id}",
      PostSerializer.new(post).serializable_hash,
      expires_in: 1.hour
    )
  end
  
  def expire_related_caches(post)
    Rails.cache.delete("user/#{post.user_id}/posts")
    Rails.cache.delete("category/#{post.category_id}/posts")
    Rails.cache.delete("featured_posts") if post.featured?
    Rails.cache.delete_matched("tag_cloud*")
  end
end
```

## ขั้นตอนที่ 1096: View Caching Best Practices

```erb
<%# เพิ่ม version ใน cache key %>
<% cache ["v2", @post] do %>
  <%# ... %>
<% end %>

<%# Cache ตาม role %>
<% cache ["post", @post, current_user&.role || "guest"] do %>
  <%# content ที่แตกต่างตาม role %>
<% end %>

<%# Cache ตาม locale %>
<% cache [@post, I18n.locale] do %>
  <%# translated content %>
<% end %>

<%# ไม่ cache เมื่อ logged in (personalized content) %>
<% if user_signed_in? %>
  <%# Personalized - ไม่ cache %>
  <%= render "personalized_post", post: @post %>
<% else %>
  <%# Public - cache ได้ %>
  <% cache @post do %>
    <%= render "public_post", post: @post %>
  <% end %>
<% end %>
```

## ขั้นตอนที่ 1097: Counter Caches

```ruby
# Counter cache ใน database
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true  # อัพเดท posts.comments_count
end

class Migration < ActiveRecord::Migration[7.0]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false
    
    # Reset existing counters
    Post.reset_column_information
    Post.pluck(:id).each do |id|
      Post.reset_counters(id, :comments)
    end
  end
end
```

```ruby
# Custom counter cache ด้วย Redis
class Post < ApplicationRecord
  def increment_views!
    Redis.current.incr("post:#{id}:views")
    
    # Sync ไปยัง database ทุกๆ 100 views
    views = Redis.current.get("post:#{id}:views").to_i
    if views % 100 == 0
      update_column(:views_count, views)
    end
  end
  
  def views_count
    Redis.current.get("post:#{id}:views").to_i
  end
end
```

## ขั้นตอนที่ 1098: Cache Testing

```ruby
# spec/support/cache_helpers.rb
module CacheHelpers
  def with_caching
    original = ActionController::Base.perform_caching
    ActionController::Base.perform_caching = true
    yield
  ensure
    ActionController::Base.perform_caching = original
    Rails.cache.clear
  end
end

RSpec.configure do |config|
  config.include CacheHelpers
end
```

```ruby
# spec/models/post_spec.rb
RSpec.describe Post do
  describe "#similar_posts" do
    let(:post) { create(:post, :published) }
    
    it "caches result" do
      # First call - cache miss
      expect(Rails.cache).to receive(:fetch).and_call_original
      post.similar_posts
      
      # Second call - cache hit
      expect(Rails.cache).not_to receive(:fetch)
      post.similar_posts
    end
    
    it "expires cache when post updated" do
      post.similar_posts  # cache it
      
      # Update the post
      post.touch
      
      # Should re-fetch
      expect(Rails.cache).to receive(:fetch).and_call_original
      post.similar_posts
    end
  end
end
```

## ขั้นตอนที่ 1099: Full-page Caching Alternative

```ruby
# ใช้ Rack::Cache หรือ Varnish สำหรับ full-page cache
# config/environments/production.rb
config.action_dispatch.rack_cache = {
  metastore: "redis://localhost:6379/cache/meta",
  entitystore: "redis://localhost:6379/cache/entity"
}

# ใน controller
def index
  @posts = Post.published.order(created_at: :desc).limit(20)
  
  expires_in 10.minutes, public: true
  
  fresh_when last_modified: @posts.maximum(:updated_at),
             etag: Digest::MD5.hexdigest(@posts.map(&:cache_key).join)
end
```

## ขั้นตอนที่ 1100: Cache Management

```ruby
# lib/tasks/cache.rake
namespace :cache do
  desc "ล้าง cache ทั้งหมด"
  task clear: :environment do
    Rails.cache.clear
    puts "Cache ถูกล้างแล้ว"
  end
  
  desc "ล้าง fragment cache"
  task clear_fragments: :environment do
    ActionController::Base.new.expire_fragment('*')
    puts "Fragment cache ถูกล้างแล้ว"
  end
  
  desc "อุ่น cache"
  task warm: :environment do
    WarmCacheJob.perform_now
    puts "Cache ถูกอุ่นแล้ว"
  end
  
  desc "ดู cache stats"
  task stats: :environment do
    if Rails.cache.respond_to?(:redis)
      info = Rails.cache.redis.info
      puts "Memory used: #{info['used_memory_human']}"
      puts "Hit rate: #{info['keyspace_hits'].to_f / (info['keyspace_hits'].to_f + info['keyspace_misses'].to_f) * 100}%"
    end
  end
end
```

---

## แบบฝึกหัด: Caching (20 ข้อ)

### ข้อที่ 1: ตั้งค่า Redis Cache Store
```ruby
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }
```

### ข้อที่ 2: Fragment Caching
```
Cache post card ใน index view
```

**เฉลย:**
```erb
<% cache post do %>
  <%= render partial: 'post_card', locals: { post: post } %>
<% end %>
```

### ข้อที่ 3: Low-level Caching
```
Cache การ query Posts ที่ trending
```

**เฉลย:**
```ruby
def trending_posts
  Rails.cache.fetch("trending_posts", expires_in: 1.hour) do
    Post.trending.limit(10).to_a
  end
end
```

### ข้อที่ 4: Russian Doll Caching
```
สร้าง nested cache: post → comments → each comment
```

### ข้อที่ 5: touch: true
```
เพิ่ม touch: true ใน Comment model
เพื่อ expire post cache เมื่อ comment เปลี่ยน
```

### ข้อที่ 6: HTTP Caching
```
เพิ่ม ETag caching ใน API endpoint
```

**เฉลย:**
```ruby
def show
  @post = Post.find(params[:id])
  if stale?(@post, public: true)
    render json: PostSerializer.new(@post).serializable_hash
  end
end
```

### ข้อที่ 7: Counter Cache
```
เพิ่ม counter_cache สำหรับ comments_count ใน Post
```

### ข้อที่ 8: Cache Expiration
```
เขียน after_commit callback ที่ expire caches ที่เกี่ยวข้อง
```

### ข้อที่ 9: Testing Cache
```
เขียน test ที่ verify ว่า cache ทำงานถูกต้อง
```

### ข้อที่ 10: Cache Warming Job
```
สร้าง job ที่ warm cache ส่วนสำคัญของ app
```

### ข้อที่ 11-20 (แบบสรุป)

**ข้อ 11:** Cache API responses ด้วย ETag
**ข้อ 12:** Conditional caching ตาม user role
**ข้อ 13:** Cache key versioning
**ข้อ 14:** Rake task สำหรับ cache management
**ข้อ 15:** Cache monitoring
**ข้อ 16:** Redis counter increment
**ข้อ 17:** Multi-level cache (memory + Redis)
**ข้อ 18:** Cache ด้วย namespace
**ข้อ 19:** Expire cache ด้วย events
**ข้อ 20:** Performance testing ก่อน/หลัง caching

---

## สรุป: Caching

| Cache Type | ใช้เมื่อ |
|-----------|---------|
| Fragment cache | ส่วนของ view ที่ expensive |
| Low-level cache | Computed values |
| Russian Doll | Nested objects |
| HTTP cache | API responses |
| Counter cache | Count queries |
| Page cache | Static content |

**Key Takeaways:**
1. Cache เฉพาะสิ่งที่ expensive จริงๆ
2. Russian Doll caching ช่วยให้ cache fine-grained
3. ใช้ Redis สำหรับ production
4. Cache key ต้องรวม version/timestamp
5. ทดสอบ cache behavior เสมอ
