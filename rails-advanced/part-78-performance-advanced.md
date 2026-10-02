# Part 78: Advanced Performance ใน Ruby on Rails

## ขั้นตอนที่ 1681-1700

---

## ขั้นตอนที่ 1681: Load Testing ด้วย wrk

wrk เป็น HTTP benchmarking tool ที่รวดเร็วและทรงพลัง

### ติดตั้ง wrk

```bash
# Ubuntu/Debian
sudo apt-get install wrk

# macOS
brew install wrk

# Build from source
git clone https://github.com/wg/wrk.git
cd wrk && make
```

### Basic Load Testing

```bash
# Test พื้นฐาน: 10 threads, 100 connections, 30 วินาที
wrk -t10 -c100 -d30s http://localhost:3000/api/products

# ผลลัพธ์
Running 30s test @ http://localhost:3000/api/products
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    45.23ms   15.67ms 312.45ms   89.23%
    Req/Sec   220.15     42.33   412.00     78.45%
  65,823 requests in 30.02s, 45.67MB read
Requests/sec:   2192.45
Transfer/sec:      1.52MB
```

### Advanced wrk Scripts

```lua
-- scripts/post_test.lua
-- Test POST request ด้วย JSON body

wrk.method = "POST"
wrk.headers["Content-Type"] = "application/json"
wrk.body = '{"username": "test@example.com", "password": "password123"}'

-- Script ที่ complex กว่า
local counter = 0

function request()
  counter = counter + 1
  local path = "/api/users/" .. (counter % 1000 + 1)
  return wrk.format("GET", path)
end

function response(status, headers, body)
  if status ~= 200 then
    print("Non-200 response: " .. status)
  end
end

function done(summary, latency, requests)
  io.write("------------------------------\n")
  for _, p in pairs({ 50, 75, 90, 99, 99.9 }) do
    n = latency:percentile(p)
    io.write(string.format("%g%%\t%d\n", p, n))
  end
end
```

```bash
# ใช้ script
wrk -t10 -c100 -d30s -s scripts/post_test.lua http://localhost:3000

# Test endpoints ต่างๆ
wrk -t4 -c50 -d60s --timeout 10s \
  -H "Authorization: Bearer YOUR_TOKEN" \
  http://localhost:3000/api/orders
```

### Interpreting Results

```
Latency: เวลาต่อ request
- Avg < 100ms = ดี
- Avg < 500ms = พอใช้
- Avg > 1000ms = ต้อง optimize

Req/Sec: throughput
- Higher is better
- Watch for Stdev (ความสม่ำเสมอ)

Error rate: ควร 0%
```

---

## ขั้นตอนที่ 1682: Load Testing ด้วย Locust

Locust เป็น Python-based load testing tool ที่ใช้งานง่าย

### ติดตั้ง Locust

```bash
pip install locust
```

### สร้าง Locust Test File

```python
# locustfile.py
from locust import HttpUser, task, between
import json
import random

class RailsApiUser(HttpUser):
    wait_time = between(1, 3)  # รอ 1-3 วินาทีระหว่าง requests

    def on_start(self):
        """Login ก่อนเริ่ม test"""
        response = self.client.post("/api/v1/auth/login", json={
            "email": "test@example.com",
            "password": "password123"
        })
        if response.status_code == 200:
            self.token = response.json()["token"]
            self.client.headers["Authorization"] = f"Bearer {self.token}"
        else:
            self.token = None

    @task(3)  # weight 3 = เรียก 3x บ่อยกว่า task อื่น
    def list_products(self):
        """GET products list"""
        page = random.randint(1, 10)
        self.client.get(f"/api/v1/products?page={page}&per_page=20",
                        name="/api/v1/products")

    @task(2)
    def get_product(self):
        """GET specific product"""
        product_id = random.randint(1, 1000)
        with self.client.get(f"/api/v1/products/{product_id}",
                             name="/api/v1/products/[id]",
                             catch_response=True) as response:
            if response.status_code == 404:
                response.success()  # 404 เป็น expected

    @task(1)
    def create_order(self):
        """POST create order"""
        if not self.token:
            return

        self.client.post("/api/v1/orders", json={
            "items": [
                {"product_id": random.randint(1, 100), "quantity": random.randint(1, 5)}
            ],
            "shipping_address": {
                "street": "123 Test St",
                "city": "Bangkok",
                "postal_code": "10100",
                "country": "TH"
            }
        })

    @task(1)
    def search_products(self):
        """GET search"""
        terms = ["laptop", "phone", "tablet", "watch"]
        term  = random.choice(terms)
        self.client.get(f"/api/v1/products/search?q={term}")


class AdminUser(HttpUser):
    wait_time = between(5, 10)

    @task
    def view_dashboard(self):
        self.client.get("/admin/dashboard")

    @task
    def export_report(self):
        self.client.get("/admin/reports/sales?format=json")
```

```bash
# รัน Locust Web UI
locust --host=http://localhost:3000

# Headless mode
locust --host=http://localhost:3000 \
       --users 100 \
       --spawn-rate 10 \
       --run-time 5m \
       --headless

# เปิด Web UI ที่ http://localhost:8089
```

---

## ขั้นตอนที่ 1683: Rack Mini Profiler

rack-mini-profiler แสดง SQL queries และ timing ใน browser

### ติดตั้ง

```ruby
# Gemfile
gem 'rack-mini-profiler'
gem 'stackprof'    # สำหรับ CPU profiling
gem 'memory_profiler'  # สำหรับ memory profiling

# config/initializers/rack_profiler.rb
if Rails.env.development? || (Rails.env.staging? && current_user&.admin?)
  require 'rack-mini-profiler'
  Rack::MiniProfiler.config.position = 'bottom-right'
  Rack::MiniProfiler.config.start_hidden = false
  Rack::MiniProfiler.config.skip_paths = ['/assets', '/packs']
end
```

### การใช้งาน

```ruby
# ใช้ใน code เพื่อ profile specific section
Rack::MiniProfiler.step("Load products") do
  @products = Product.includes(:category, :images).page(params[:page])
end

Rack::MiniProfiler.step("Calculate prices") do
  @prices = @products.map { |p| calculate_discounted_price(p) }
end

# Force enable สำหรับ specific requests
# Add ?pp=enable to URL
# Add ?pp=flamegraph สำหรับ flamegraph
# Add ?pp=profile-memory สำหรับ memory profile
```

### Flamegraph

```bash
# เปิด URL พร้อม ?pp=flamegraph
http://localhost:3000/api/products?pp=flamegraph
```

---

## ขั้นตอนที่ 1684: Skylight (Production Profiling)

Skylight เป็น APM tool สำหรับ production Rails apps

### ติดตั้ง

```ruby
# Gemfile
gem 'skylight'

# Terminal
bundle exec skylight setup
```

### การตั้งค่า

```yaml
# config/skylight.yml
authentication: your_skylight_token_here

# environments ที่ต้องการ monitor
environments:
  - production
  - staging
```

```ruby
# config/application.rb
config.skylight.environments = %w[production staging]
```

### Custom Instrumentation

```ruby
# เพิ่ม custom tracing
class OrderService
  def place_order(params)
    Skylight.instrument(title: "Place Order", description: "Order #{params[:order_id]}") do
      # code ที่ต้องการ monitor
      validate_order(params)
      create_order(params)
      charge_payment(params)
    end
  end
end

# ใน Model
class Product < ApplicationRecord
  def calculate_recommendation_score
    Skylight.instrument(title: "Recommendation Score Calculation") do
      # complex calculation
      views_score    = view_count * 0.3
      purchase_score = purchase_count * 0.5
      rating_score   = average_rating * 0.2
      views_score + purchase_score + rating_score
    end
  end
end
```

---

## ขั้นตอนที่ 1685: Memory Profiler

### memory_profiler gem

```ruby
# Gemfile
gem 'memory_profiler', require: false

# Profile specific code
require 'memory_profiler'

report = MemoryProfiler.report do
  # code ที่ต้องการ profile
  1000.times do
    user = User.new(name: "Test", email: "test@example.com")
    user.valid?
  end
end

report.pretty_print

# Output บางส่วน:
# Total allocated: 12345678 bytes (54321 objects)
# Total retained:  1234 bytes (12 objects)
```

### การ Profile ใน Rails

```ruby
# lib/tasks/memory_profile.rake
namespace :profile do
  task memory: :environment do
    require 'memory_profiler'

    puts "Profiling memory usage..."

    report = MemoryProfiler.report do
      # simulate production workload
      100.times do
        products = Product.includes(:category).limit(20).to_a
        products.map { |p| ProductSerializer.new(p).as_json }
      end
    end

    report.pretty_print(to_file: Rails.root.join('tmp', 'memory_profile.txt'))
    puts "Report saved to tmp/memory_profile.txt"
  end
end
```

### Finding Memory Leaks

```ruby
# config/initializers/memory_leak_detector.rb
if Rails.env.development?
  class MemoryTracker
    def self.track
      before  = GC.stat[:heap_live_slots]
      result  = yield
      after   = GC.stat[:heap_live_slots]
      growth  = after - before

      if growth > 1000
        Rails.logger.warn "Memory growth: +#{growth} objects"
        # Log caller info
        Rails.logger.warn caller.first(5).join("\n")
      end

      result
    end
  end
end

# ใน Controller เพื่อ detect leaks
class ApplicationController < ActionController::Base
  around_action :track_memory

  private

  def track_memory
    before = ObjectSpace.count_objects[:T_OBJECT]
    yield
    after  = ObjectSpace.count_objects[:T_OBJECT]

    if (after - before) > 1000
      Rails.logger.warn "Controller #{controller_name}##{action_name}: +#{after - before} objects"
    end
  end
end
```

---

## ขั้นตอนที่ 1686: GC Tuning

Ruby GC tuning สามารถเพิ่ม performance ได้มาก

### Understanding Ruby GC

```ruby
# ดู GC statistics ปัจจุบัน
puts GC.stat.inspect

# ผลลัพธ์บางส่วน:
{
  count: 100,              # จำนวนครั้ง GC ทำงาน
  heap_allocated_pages: 500,
  heap_sorted_length: 510,
  heap_allocatable_pages: 10,
  heap_available_slots: 200000,
  heap_live_slots: 180000,
  heap_free_slots: 20000,
  heap_final_slots: 0,
  heap_marked_slots: 150000,
  heap_eden_pages: 450,
  heap_tomb_pages: 50,
  total_allocated_objects: 5000000,
  total_freed_objects: 4820000
}
```

### Environment Variables สำหรับ GC Tuning

```bash
# config/environments/production.rb หรือ .env

# Heap growth (default: 1.8 = 80% growth)
RUBY_GC_HEAP_GROWTH_FACTOR=1.1          # เพิ่มทีละ 10% (conservative)

# Min heap slots
RUBY_GC_HEAP_INIT_SLOTS=600000          # initial heap ใหญ่ขึ้น

# Heap free slots ratio
RUBY_GC_HEAP_FREE_SLOTS_MIN_RATIO=0.10  # รักษา free slots อย่างน้อย 10%
RUBY_GC_HEAP_FREE_SLOTS_MAX_RATIO=0.25  # ไม่เกิน 25%
RUBY_GC_HEAP_FREE_SLOTS_GOAL_RATIO=0.20 # เป้าหมาย 20%

# Malloc limit
RUBY_GC_MALLOC_LIMIT=64000000          # 64MB ก่อน major GC
RUBY_GC_MALLOC_LIMIT_MAX=256000000     # max 256MB
RUBY_GC_MALLOC_LIMIT_GROWTH_FACTOR=1.4

# Old malloc limit
RUBY_GC_OLDMALLOC_LIMIT=16000000
RUBY_GC_OLDMALLOC_LIMIT_MAX=128000000
```

### GC Compact (Ruby 3.x)

```ruby
# ใช้ GC.compact เพื่อลด memory fragmentation
# เหมาะสำหรับ long-running processes

# config/initializers/gc_compact.rb
if Rails.env.production?
  # Compact หลัง boot
  GC.compact

  # Optional: compact ทุกๆ N requests (แต่มี overhead)
  class CompactOnInterval
    COMPACT_INTERVAL = 1000

    def initialize(app)
      @app     = app
      @request_count = 0
    end

    def call(env)
      @request_count += 1

      if @request_count % COMPACT_INTERVAL == 0
        GC.compact
      end

      @app.call(env)
    end
  end
end
```

### Ractors สำหรับ Parallelism (Ruby 3.x)

```ruby
# ใช้ Ractors สำหรับ CPU-intensive work แบบ parallel
results = (1..4).map do |i|
  Ractor.new(i) do |n|
    # Heavy computation in parallel
    (1..1_000_000).sum { |x| x * n }
  end
end.map(&:take)
```

---

## ขั้นตอนที่ 1687: Database Connection Pooling

### PgBouncer Setup

```bash
# ติดตั้ง PgBouncer
sudo apt-get install pgbouncer

# /etc/pgbouncer/pgbouncer.ini
[databases]
myapp_production = host=localhost port=5432 dbname=myapp_production

[pgbouncer]
listen_port = 6432
listen_addr = localhost
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction    # transaction-level pooling
max_client_conn = 1000     # max client connections
default_pool_size = 20     # connections to database per pool
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
log_connections = 0
log_disconnections = 0
```

```ruby
# config/database.yml (ใช้ PgBouncer)
production:
  adapter:  postgresql
  host:     localhost
  port:     6432          # PgBouncer port (ไม่ใช่ 5432)
  database: myapp_production
  username: myapp
  password: <%= ENV['DATABASE_PASSWORD'] %>
  pool:     5             # ลดลงเพราะ PgBouncer จัดการ
  checkout_timeout: 5
  connect_timeout: 5

  # PgBouncer transaction mode
  prepared_statements: false  # ต้อง disable สำหรับ transaction mode
  advisory_locks: false
```

### Rails Connection Pool Tuning

```ruby
# config/database.yml
production:
  pool: <%= ENV.fetch('RAILS_MAX_THREADS', 5) %>
  checkout_timeout: 5   # วินาทีที่รอ connection
  reaping_frequency: 10 # ตรวจสอบ dead connections ทุก 10 วินาที
  idle_timeout: 300     # คืน connection หลัง idle 5 นาที
```

```ruby
# config/puma.rb
# ปรับ threads ให้ match กับ database pool
max_threads_count = ENV.fetch("RAILS_MAX_THREADS", 5)
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

# Workers (processes) x threads = database connections needed
workers_count = ENV.fetch("WEB_CONCURRENCY", 2)
workers workers_count

# Warmup database connection pool
on_worker_boot do
  ActiveRecord::Base.establish_connection
end
```

---

## ขั้นตอนที่ 1688: Horizontal Scaling

### Stateless Application

```ruby
# ❌ ไม่ควรเก็บ state ใน application server
class UsersController < ApplicationController
  @@connected_users = []  # ❌ ไม่ work กับ multiple servers

  def login
    @@connected_users << current_user.id
  end
end

# ✅ เก็บ state ใน Redis
class UsersController < ApplicationController
  def login
    Redis.current.sadd("connected_users", current_user.id)
    Redis.current.expire("connected_users", 24.hours.to_i)
  end
end
```

### Session Storage

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store(
  :redis_store,
  servers:     [ENV['REDIS_URL']],
  expire_after: 2.hours,
  key:          '_myapp_session',
  threadsafe:   false
)

# หรือใช้ cookie store (ง่ายกว่า)
Rails.application.config.session_store :cookie_store,
  key:    '_myapp_session',
  secure: Rails.env.production?
```

### Load Balancer Health Check

```ruby
# config/routes.rb
get '/health', to: 'health#check'

# app/controllers/health_controller.rb
class HealthController < ApplicationController
  skip_before_action :authenticate_user!

  def check
    checks = {
      database: database_healthy?,
      redis:    redis_healthy?,
      storage:  storage_healthy?
    }

    if checks.values.all?
      render json: { status: 'ok', checks: checks }, status: :ok
    else
      render json: {
        status: 'degraded',
        checks: checks
      }, status: :service_unavailable
    end
  end

  private

  def database_healthy?
    ActiveRecord::Base.connection.execute("SELECT 1")
    true
  rescue => e
    Rails.logger.error "Database health check failed: #{e.message}"
    false
  end

  def redis_healthy?
    Redis.current.ping == 'PONG'
  rescue => e
    Rails.logger.error "Redis health check failed: #{e.message}"
    false
  end

  def storage_healthy?
    # Check S3 or local storage
    true
  rescue => e
    false
  end
end
```

---

## ขั้นตอนที่ 1689: Read Replicas

### Configure Multiple Databases

```ruby
# config/database.yml
production:
  primary:
    adapter:  postgresql
    database: myapp_production
    host:     primary-db.example.com

  primary_replica:
    adapter:  postgresql
    database: myapp_production
    host:     replica-db.example.com
    replica:  true        # สำคัญ! mark เป็น replica

  analytics:
    adapter:  postgresql
    database: myapp_analytics
    host:     analytics-db.example.com
```

### ตั้งค่า Models

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to database: {
    writing: :primary,
    reading: :primary_replica
  }
end

# Model ที่ใช้ analytics database
class AnalyticsRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to database: {
    writing: :analytics,
    reading: :analytics
  }
end
```

### ใช้ Read Replica

```ruby
# อ่านจาก replica โดยอัตโนมัติ
ActiveRecord::Base.connected_to(role: :reading) do
  @products = Product.all.to_a  # อ่านจาก replica
end

# ใน Controller - read requests ไปที่ replica
class ProductsController < ApplicationController
  def index
    ActiveRecord::Base.connected_to(role: :reading) do
      @products = Product.includes(:category).page(params[:page])
    end
  end

  def create
    # write ไปที่ primary
    @product = Product.create!(product_params)
  end
end

# Middleware สำหรับ automatic routing
# config/application.rb
config.active_record.database_selector = {
  delay: 2.seconds  # รอ 2 วินาทีหลัง write ก่อน route ไป replica
}
config.active_record.database_resolver = ActiveRecord::Middleware::DatabaseSelector::Resolver
config.active_record.database_resolver_context = ActiveRecord::Middleware::DatabaseSelector::Resolver::Session
```

---

## ขั้นตอนที่ 1690: Database Sharding

Sharding แบ่ง data ออกเป็น shards เพื่อ scale horizontal

### Basic Sharding Setup

```ruby
# config/database.yml
production:
  primary:
    adapter: postgresql
    database: myapp_shard0
    host: shard0.example.com

  primary_shard_one:
    adapter: postgresql
    database: myapp_shard1
    host: shard1.example.com

  primary_shard_two:
    adapter: postgresql
    database: myapp_shard2
    host: shard2.example.com
```

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to shards: {
    default:   { writing: :primary },
    shard_one: { writing: :primary_shard_one },
    shard_two: { writing: :primary_shard_two }
  }
end

# Shard Routing
class ShardRouter
  SHARDS = [:default, :shard_one, :shard_two].freeze

  def self.shard_for(tenant_id)
    shard_index = Zlib::crc32(tenant_id.to_s) % SHARDS.count
    SHARDS[shard_index]
  end
end

# ใช้งาน
class TenantsController < ApplicationController
  def show
    shard = ShardRouter.shard_for(params[:tenant_id])

    ActiveRecord::Base.connected_to(shard: shard) do
      @tenant = Tenant.find(params[:tenant_id])
      @data   = TenantData.where(tenant_id: @tenant.id).to_a
    end
  end
end

# Middleware สำหรับ automatic shard selection
class ShardMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    request = ActionDispatch::Request.new(env)
    tenant_id = extract_tenant_id(request)

    if tenant_id
      shard = ShardRouter.shard_for(tenant_id)
      ActiveRecord::Base.connected_to(shard: shard) do
        @app.call(env)
      end
    else
      @app.call(env)
    end
  end

  private

  def extract_tenant_id(request)
    # จาก subdomain, header, หรือ JWT
    request.headers['X-Tenant-ID'] ||
      Tenant.find_by(subdomain: request.subdomain)&.id
  end
end
```

---

## ขั้นตอนที่ 1691: CDN สำหรับ Assets

### Asset Pipeline กับ CDN

```ruby
# config/environments/production.rb
config.asset_host = ENV['CDN_HOST']  # 'https://cdn.example.com'

# หรือใช้ function สำหรับ dynamic asset host
config.asset_host = proc { |source|
  # ใช้ multiple CDN domains สำหรับ browser parallelism
  number = (source.length % 4) + 1
  "https://cdn#{number}.example.com"
}
```

### Active Storage กับ CDN

```ruby
# config/environments/production.rb
config.active_storage.service = :amazon

# config/storage.yml
amazon:
  service: S3
  access_key_id:     <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region:            <%= ENV['AWS_REGION'] %>
  bucket:            <%= ENV['AWS_BUCKET'] %>

# CDN distribution สำหรับ S3
# ใช้ CloudFront หน้า S3
```

### ตั้งค่า CloudFront

```ruby
# config/initializers/active_storage_cdn.rb
if Rails.env.production?
  module ActiveStorage
    module Variant
      def processed
        super.tap do |variant|
          # Warm CDN cache
          CDNWarmer.enqueue(url_for(variant)) if CDNWarmer.respond_to?(:enqueue)
        end
      end
    end
  end
end

# ใช้ Signed URLs สำหรับ private files
class DocumentsController < ApplicationController
  def show
    document = Document.find(params[:id])
    authorize! :read, document

    # สร้าง signed URL ที่หมดอายุใน 5 นาที
    url = document.file.url(expires_in: 5.minutes)
    redirect_to url
  end
end
```

---

## ขั้นตอนที่ 1692: HTTP/2

### Nginx HTTP/2 Configuration

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 443 ssl http2;  # เปิดใช้ HTTP/2
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # SSL optimization
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # HTTP/2 Push (Server Push)
    location / {
        http2_push /assets/application.css;
        http2_push /assets/application.js;

        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Cache static assets
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        gzip_static on;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### HTTP/2 Server Push ใน Rails

```ruby
# config/environments/production.rb
config.action_dispatch.default_headers = {
  'X-XSS-Protection'         => '1; mode=block',
  'X-Content-Type-Options'   => 'nosniff',
  'X-Download-Options'       => 'noopen',
  'X-Permitted-Cross-Domain-Policies' => 'none',
  'Referrer-Policy'          => 'strict-origin-when-cross-origin'
}

# ใน Controller สำหรับ HTTP/2 Push
class ApplicationController < ActionController::Base
  before_action :push_critical_assets

  private

  def push_critical_assets
    response.set_header('Link', [
      "</assets/application.css>; rel=preload; as=style",
      "</assets/application.js>; rel=preload; as=script"
    ].join(', '))
  end
end
```

---

## ขั้นตอนที่ 1693: Query Optimization เพิ่มเติม

```ruby
# 1. Bulk Operations
# ❌ N+1 inserts
100.times { |i| Product.create!(name: "Product #{i}", price: i * 10) }

# ✅ Bulk insert
products_data = 100.times.map { |i|
  { name: "Product #{i}", price: i * 10, created_at: Time.current, updated_at: Time.current }
}
Product.insert_all(products_data)  # Rails 6+

# With unique constraint handling
Product.upsert_all(
  products_data,
  unique_by: :name,
  on_duplicate: :update
)

# 2. Explain Analyze
result = ActiveRecord::Base.connection.execute(
  "EXPLAIN ANALYZE #{Product.where('price > 100').to_sql}"
)
puts result.to_a.map { |r| r['QUERY PLAN'] }.join("\n")

# 3. Partial Indexes
add_index :orders, :user_id, where: "status = 'pending'", name: 'index_pending_orders'
add_index :products, :category_id, where: "active = true"

# 4. Generated Columns (PostgreSQL)
execute <<-SQL
  ALTER TABLE products
  ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(name, '') || ' ' || coalesce(description, ''))
  ) STORED;

  CREATE INDEX index_products_search_vector ON products USING GIN(search_vector);
SQL

# การค้นหา
Product.where("search_vector @@ plainto_tsquery('english', ?)", params[:q])

# 5. JSON Columns
# Migration
add_column :users, :preferences, :jsonb, default: {}
add_index  :users, :preferences, using: :gin

# Query JSON
User.where("preferences->>'theme' = ?", 'dark')
User.where("preferences @> ?", { notifications: true }.to_json)
```

---

## ขั้นตอนที่ 1694: Caching Strategy

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url:                  ENV['REDIS_URL'],
  connect_timeout:      30,
  read_timeout:         0.2,
  write_timeout:        0.2,
  reconnect_attempts:   1,
  error_handler:        -> (method:, returning:, exception:) {
    Sentry.capture_exception(exception, level: 'warning',
      tags: { method: method, returning: returning })
  }
}

# Fragment Caching
# app/views/products/index.html.erb
<% cache ["products-v1", @products.maximum(:updated_at)] do %>
  <% @products.each do |product| %>
    <% cache ["product-card-v1", product] do %>
      <%= render 'product_card', product: product %>
    <% end %>
  <% end %>
<% end %>

# HTTP Caching
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])

    # HTTP Cache-Control headers
    fresh_when(
      etag:          @product,
      last_modified: @product.updated_at,
      public:        true
    )
  end

  def index
    @products = Product.all

    # Expires ใน 5 นาที
    expires_in 5.minutes, public: true

    respond_to do |format|
      format.json { render json: @products }
      format.html
    end
  end
end

# Low-Level Caching
class ProductRecommendationService
  def recommendations_for(user_id)
    Rails.cache.fetch("recommendations/#{user_id}", expires_in: 1.hour) do
      # expensive computation
      User.find(user_id)
          .orders
          .recent
          .flat_map(&:products)
          .tally
          .sort_by { |_, count| -count }
          .first(10)
          .map(&:first)
    end
  end
end
```

---

## ขั้นตอนที่ 1695: Background Job Optimization

```ruby
# Sidekiq configuration
# config/sidekiq.yml
:concurrency: 10
:max_retries: 3
:queues:
  - [critical, 3]   # priority queue
  - [default, 2]
  - [low, 1]

# ปรับ memory ใน worker
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV['REDIS_URL'], size: 15 }

  # ลด GC pressure
  config.on(:startup) do
    GC.compact
  end
end

# Batch processing
class BulkEmailJob < ApplicationJob
  queue_as :default

  BATCH_SIZE = 100

  def perform(user_ids)
    # Process ใน batches เพื่อลด memory usage
    user_ids.each_slice(BATCH_SIZE) do |batch|
      users = User.where(id: batch).to_a

      users.each do |user|
        UserMailer.newsletter(user).deliver_now
      end

      # ช่วยให้ GC ทำงาน
      GC.start if batch.last == user_ids.last
    end
  end
end

# Job deduplication
class ProcessOrderJob < ApplicationJob
  include SidekiqUniqueJobs::Worker

  sidekiq_options(
    unique: :until_executed,
    unique_args: ->(args) { [args.first] }  # unique ต่อ order_id
  )

  def perform(order_id)
    Order.find(order_id).process!
  end
end
```

---

## ขั้นตอนที่ 1696: Profiling Tools และ Benchmarks

```ruby
# Benchmark Ruby code
require 'benchmark'
require 'benchmark/ips'

# Basic benchmark
Benchmark.bm(20) do |x|
  x.report("Array#map:") do
    1000.times { (1..1000).map { |i| i * 2 } }
  end

  x.report("Array#flat_map:") do
    1000.times { (1..1000).flat_map { |i| [i * 2] } }
  end
end

# Iterations per second
Benchmark.ips do |x|
  x.report("string interpolation") { "Hello #{user.name}!" }
  x.report("string concatenation") { "Hello " + user.name + "!" }
  x.compare!
end

# Stackprof CPU profiling
require 'stackprof'

StackProf.run(mode: :cpu, out: 'tmp/stackprof.dump') do
  # code ที่ต้องการ profile
  1000.times do
    Product.includes(:category, :images)
           .where(active: true)
           .limit(20)
           .to_a
  end
end

# วิเคราะห์ผล
# stackprof tmp/stackprof.dump --text
# stackprof tmp/stackprof.dump --flamegraph > tmp/flamegraph.json

# rbspy - profile Ruby processes โดยไม่ต้อง modify code
# sudo rbspy record --pid $(cat tmp/pids/server.pid)
```

---

## ขั้นตอนที่ 1697: Response Compression

```ruby
# Rack Deflater middleware
# config/application.rb
config.middleware.use Rack::Deflater

# Nginx กำหนด gzip
# /etc/nginx/nginx.conf
# gzip on;
# gzip_vary on;
# gzip_proxied any;
# gzip_comp_level 6;
# gzip_types text/plain text/css text/xml application/json
#            application/javascript application/xml+rss application/atom+xml
#            image/svg+xml;

# Brotli compression (ดีกว่า gzip)
# nginx ต้องการ module ngx_brotli
# brotli on;
# brotli_comp_level 6;
# brotli_types text/plain text/css application/json application/javascript;

# API Response compression
class ApplicationController < ActionController::API
  before_action :set_compression_headers

  private

  def set_compression_headers
    # บอก client ว่า server รองรับ compression
    response.headers['Vary'] = 'Accept-Encoding'
  end
end
```

---

## ขั้นตอนที่ 1698: APM Integration

```ruby
# New Relic
# Gemfile
# gem 'newrelic_rpm'

# config/newrelic.yml
# license_key: YOUR_LICENSE_KEY
# app_name: My Rails App

# Datadog
# Gemfile
# gem 'ddtrace'

# config/initializers/datadog.rb
require 'ddtrace'

Datadog.configure do |c|
  c.use :rails, {
    service_name:     'my-rails-app',
    controller_service: 'rails-controller',
    database_service:   'rails-db',
    cache_service:      'rails-cache'
  }

  c.use :sidekiq, service_name: 'sidekiq'
  c.use :redis,   service_name: 'redis'
  c.use :faraday, service_name: 'http'

  c.tracer.enabled = Rails.env.production?
end

# Custom spans
class OrderService
  def place_order(params)
    Datadog::Tracing.trace('order.place', resource: params[:order_id]) do |span|
      span.set_tag('customer.id', params[:customer_id])

      order = build_order(params)
      charge_payment(order)

      span.set_tag('order.total', order.total)
    end
  end
end
```

---

## ขั้นตอนที่ 1699: Performance Monitoring Dashboard

```ruby
# config/initializers/performance_monitoring.rb

# Prometheus metrics
require 'prometheus/client'
require 'prometheus/client/formats/text'

PROMETHEUS_REGISTRY = Prometheus::Client.registry

REQUEST_DURATION = PROMETHEUS_REGISTRY.histogram(
  :http_request_duration_seconds,
  docstring: 'HTTP request duration in seconds',
  labels:    [:method, :path, :status],
  buckets:   [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
)

DB_QUERY_DURATION = PROMETHEUS_REGISTRY.histogram(
  :db_query_duration_seconds,
  docstring: 'Database query duration in seconds',
  labels:    [:operation]
)

CACHE_HITS = PROMETHEUS_REGISTRY.counter(
  :cache_hits_total,
  docstring: 'Total cache hits',
  labels:    [:key_prefix]
)

# Middleware สำหรับ track metrics
class PrometheusMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    path   = ActionDispatch::Request.new(env).path
    method = env['REQUEST_METHOD']

    start    = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start

    REQUEST_DURATION.observe(
      duration,
      labels: { method: method, path: normalize_path(path), status: status }
    )

    [status, headers, body]
  end

  private

  def normalize_path(path)
    # แทน /users/123 ด้วย /users/:id
    path.gsub(/\/\d+/, '/:id')
        .gsub(/\/[a-f0-9-]{36}/, '/:uuid')
  end
end

# Expose metrics endpoint
# config/routes.rb
# get '/metrics', to: PrometheusController
class PrometheusController < ActionController::API
  def index
    render plain: Prometheus::Client::Formats::Text.marshal(PROMETHEUS_REGISTRY),
           content_type: 'text/plain; version=0.0.4'
  end
end
```

---

## ขั้นตอนที่ 1700: แนวปฏิบัติที่ดีสำหรับ Performance

```ruby
# 1. Measure before optimizing
# ใช้ rack-mini-profiler หา bottleneck ก่อน optimize

# 2. Database indexes ครบ
# ทุก foreign key ต้องมี index
# ทุก column ที่ใช้ใน WHERE clause ต้องมี index

# 3. N+1 query elimination
# ใช้ bullet gem เพื่อ detect N+1

# Gemfile (development)
gem 'bullet'

# config/environments/development.rb
config.after_initialize do
  Bullet.enable        = true
  Bullet.alert         = true
  Bullet.rails_logger  = true
  Bullet.add_footer    = true
  Bullet.slack = { webhook_url: ENV['SLACK_WEBHOOK'], channel: '#dev' }
end

# 4. Pagination ทุก list endpoint
class ProductsController < ApplicationController
  def index
    @products = Product.all.page(params[:page]).per(params[:per_page] || 20)
    # ไม่ใช้ Product.all.to_a (อันตราย!)
  end
end

# 5. Background jobs สำหรับ heavy operations
class ReportsController < ApplicationController
  def create
    # ❌ ทำ sync - user รอนาน
    # report = ReportGenerator.generate(params)
    # render json: report

    # ✅ ทำ async - return ทันที
    job = GenerateReportJob.perform_later(
      user_id:    current_user.id,
      params:     report_params
    )
    render json: { job_id: job.job_id, status: 'processing' }
  end
end

# 6. Response serialization caching
class ProductSerializer < ActiveModel::Serializer
  cache key: 'product', expires_in: 10.minutes

  attributes :id, :name, :price, :category

  def cache_key
    [object.id, object.updated_at.to_i].join('/')
  end
end
```

---

## แบบฝึกหัด: Advanced Performance

### ข้อที่ 1: Load Test Setup

```bash
# สร้าง wrk script สำหรับ test authentication
# และ run test เก็บ baseline metrics

# run_loadtest.sh
#!/bin/bash
echo "=== Load Test Results ===" > results.txt
echo "Date: $(date)" >> results.txt

echo "-- Login Endpoint --" >> results.txt
wrk -t4 -c50 -d30s -s scripts/login.lua http://localhost:3000 >> results.txt

echo "-- Products List --" >> results.txt
wrk -t4 -c50 -d30s \
  -H "Authorization: Bearer $TEST_TOKEN" \
  http://localhost:3000/api/products >> results.txt

echo "-- Product Search --" >> results.txt
wrk -t4 -c50 -d30s \
  "http://localhost:3000/api/products?q=laptop" >> results.txt

cat results.txt
```

### ข้อที่ 2: Profiling กับ rack-mini-profiler
ติดตั้ง rack-mini-profiler และ identify 3 slow endpoints ใน application

### ข้อที่ 3: Memory Profiling
ใช้ memory_profiler gem เพื่อ find top 5 memory-intensive operations

### ข้อที่ 4: N+1 Query Fix

```ruby
# ระบุปัญหาและแก้ไข N+1 queries ด้วย includes/preload/eager_load
# ตัวอย่างปัญหา
def show_orders
  orders = Order.recent.limit(20)
  # N+1 ที่ซ่อนอยู่
  orders.each do |order|
    puts order.customer.name   # N+1
    order.items.each do |item|
      puts item.product.name   # N+1 * M
    end
  end
end

# แก้ไข
def show_orders_fixed
  orders = Order.includes(customer: {}, items: :product).recent.limit(20)
  orders.each do |order|
    puts order.customer.name
    order.items.each { |item| puts item.product.name }
  end
end
```

### ข้อที่ 5: Database Index Optimization
ระบุ queries ที่ใช้ Seq Scan และเพิ่ม indexes ที่เหมาะสม

### ข้อที่ 6: Redis Caching
เพิ่ม caching layer สำหรับ expensive queries ด้วย Redis

### ข้อที่ 7: Read Replica Setup
ตั้งค่า read replica สำหรับ reporting queries

### ข้อที่ 8-20: แบบฝึกหัดเพิ่มเติม

**ข้อ 8:** ตั้งค่า PgBouncer และวัดผลต่าง connection overhead

**ข้อ 9:** Implement HTTP/2 Server Push สำหรับ critical CSS/JS

**ข้อ 10:** สร้าง Prometheus metrics สำหรับ custom business events

**ข้อ 11:** Tune GC settings สำหรับ production environment

**ข้อ 12:** Implement bulk operations แทน individual inserts

**ข้อ 13:** สร้าง CDN configuration สำหรับ uploads ด้วย CloudFront

**ข้อ 14:** Implement response caching ด้วย HTTP Cache-Control headers

**ข้อ 15:** Setup database sharding สำหรับ multi-tenant application

**ข้อ 16:** สร้าง performance budget: ทุก endpoint ต้อง < 200ms p95

**ข้อ 17:** ใช้ Skylight ใน staging และ identify top 10 slow transactions

**ข้อ 18:** Implement background job batching สำหรับ email sending

**ข้อ 19:** สร้าง full performance monitoring dashboard ด้วย Grafana + Prometheus

**ข้อ 20:** ทำ capacity planning: วิเคราะห์ว่าระบบรองรับ user กี่คนก่อน scale

---

## สรุป

Advanced Performance ใน Rails ครอบคลุม:
- **Load Testing**: วัด baseline และ continuous testing
- **Profiling**: ระบุ bottlenecks ใน code และ database
- **GC Tuning**: ลด GC pauses ใน production
- **Connection Pooling**: จัดการ database connections อย่างมีประสิทธิภาพ
- **Horizontal Scaling**: stateless apps กับ load balancers
- **Read Replicas**: scale reads แยกจาก writes
- **CDN**: ลด latency สำหรับ static assets
- **HTTP/2**: multiplexing และ server push

ขั้นตอนต่อไป: Part 79 - Kubernetes and Production Operations
