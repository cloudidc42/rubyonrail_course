# Part 61: Database Optimization ใน Ruby on Rails

## ขั้นตอนที่ 1331-1350: เพิ่มประสิทธิภาพ Database

---

## ขั้นตอนที่ 1331: EXPLAIN ANALYZE

### ใช้ EXPLAIN ใน PostgreSQL

```sql
-- ดูแผนการรัน query
EXPLAIN SELECT * FROM posts WHERE status = 'published';

-- ดูแผนพร้อมข้อมูลจริง (รัน query จริงๆ)
EXPLAIN ANALYZE SELECT * FROM posts WHERE status = 'published';

-- ดูรายละเอียดเพิ่มเติม
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT * FROM posts WHERE status = 'published';
```

### ตัวอย่าง Output

```
                         QUERY PLAN
---------------------------------------------------------------
Seq Scan on posts  (cost=0.00..25.88 rows=6 width=289) 
                   (actual time=0.014..0.027 rows=6 loops=1)
  Filter: ((status)::text = 'published'::text)
  Rows Removed by Filter: 38
Planning Time: 0.078 ms
Execution Time: 0.042 ms
```

**อ่านผล:**
- `Seq Scan` = Sequential Scan (ไม่ใช้ index, ช้า)
- `cost=0.00..25.88` = estimated cost
- `actual time=0.014..0.027` = เวลาจริง
- `rows=6` = จำนวน rows ที่ได้

### ใช้ EXPLAIN ใน Rails

```ruby
# Rails Console
Post.where(status: 'published').explain
# หรือ
puts Post.where(status: 'published').explain

# แบบ verbose
ActiveRecord::Base.connection.execute(
  "EXPLAIN ANALYZE #{Post.where(status: 'published').to_sql}"
).each { |row| puts row['QUERY PLAN'] }
```

### ติดตั้ง pg_query gem

```ruby
# Gemfile
gem 'rails-pg-extras'
```

```ruby
# ดู slow queries
RailsPgExtras.calls
RailsPgExtras.long_running_queries
RailsPgExtras.index_usage
RailsPgExtras.bloat
```

---

## ขั้นตอนที่ 1332: Index Types ใน PostgreSQL

### B-tree Index (default)

B-tree เป็น index ที่ใช้บ่อยที่สุด เหมาะกับ equality และ range queries

```ruby
# Migration
add_index :posts, :status              # B-tree by default
add_index :posts, :created_at
add_index :posts, [:user_id, :status]  # Composite B-tree

# ใช้ที่ไหน:
# - WHERE status = 'published'
# - WHERE created_at > '2024-01-01'
# - ORDER BY created_at DESC
```

### GIN Index (Generalized Inverted Index)

GIN เหมาะกับ arrays, JSONB, full-text search

```ruby
# Full-text search
add_index :posts, :title, using: :gin
add_index :posts, :tsv_body, using: :gin  # tsvector

# JSONB
add_index :products, :metadata, using: :gin

# Array column
add_index :posts, :tag_ids, using: :gin
```

```sql
-- Full-text search ด้วย GIN
CREATE INDEX posts_search_idx ON posts 
USING GIN (to_tsvector('english', title || ' ' || body));

-- ค้นหา
SELECT * FROM posts
WHERE to_tsvector('english', title || ' ' || body) 
  @@ to_tsquery('english', 'rails & tutorial');
```

### GiST Index (Generalized Search Tree)

GiST เหมาะกับ geometric data, ranges, fuzzy text search

```sql
-- Range index
CREATE INDEX orders_date_range_idx ON orders 
USING GiST (daterange(start_date, end_date));

-- Geometric
CREATE INDEX locations_idx ON places 
USING GiST (location);

-- pg_trgm (trigram) สำหรับ LIKE queries
CREATE EXTENSION pg_trgm;
CREATE INDEX posts_title_trgm ON posts 
USING GiST (title gist_trgm_ops);

-- ค้นหาด้วย LIKE
SELECT * FROM posts WHERE title LIKE '%rails%';
-- จะใช้ GiST index นี้!
```

### BRIN Index (Block Range INdex)

BRIN เหมาะกับ large tables ที่มี physical order ตาม column (เช่น timestamps)

```ruby
# สำหรับ timestamp columns ใน tables ขนาดใหญ่
add_index :logs, :created_at, using: :brin

# ข้อดี: ขนาดเล็กมาก (เทียบกับ B-tree)
# ข้อเสีย: ไม่แม่นยำ 100% (ต้อง heap scan บางส่วน)
```

```sql
CREATE INDEX logs_created_at_brin ON logs 
USING BRIN (created_at) 
WITH (pages_per_range = 128);
```

---

## ขั้นตอนที่ 1333: Partial Indexes

Partial Index ใช้ indexing เฉพาะ subset ของ rows

```ruby
# Index เฉพาะ published posts
add_index :posts, :created_at, 
          where: "status = 'published'",
          name: "index_posts_on_created_at_published"

# Index เฉพาะ non-deleted records
add_index :users, :email, 
          where: "deleted_at IS NULL",
          name: "index_users_on_email_active"

# Index เฉพาะ null values
add_index :jobs, :started_at,
          where: "started_at IS NULL",
          name: "index_jobs_pending"
```

```sql
-- ตัวอย่าง SQL
CREATE INDEX CONCURRENTLY posts_published_idx 
ON posts (created_at DESC) 
WHERE status = 'published' AND deleted_at IS NULL;

-- Query ที่ใช้ index นี้
SELECT * FROM posts 
WHERE status = 'published' 
AND deleted_at IS NULL
ORDER BY created_at DESC;
```

**ข้อดี:**
- Index มีขนาดเล็กกว่า
- Update เร็วกว่า (เพราะ index น้อย rows)
- ค้นหาเร็วกว่าสำหรับ filtered queries

---

## ขั้นตอนที่ 1334: Composite Indexes

Composite Index ใช้ได้กับ WHERE, ORDER BY, GROUP BY

```ruby
# Composite indexes
add_index :posts, [:user_id, :status]
add_index :posts, [:user_id, :created_at]
add_index :orders, [:user_id, :status, :created_at]
```

### กฎของ Leftmost Prefix

```sql
-- index: (user_id, status, created_at)

-- ใช้ index ได้:
WHERE user_id = 1                              -- ใช้ user_id
WHERE user_id = 1 AND status = 'published'    -- ใช้ user_id, status
WHERE user_id = 1 AND status = 'published' AND created_at > '2024-01-01'

-- ไม่ใช้ index (เต็มๆ):
WHERE status = 'published'                    -- ข้าม user_id
WHERE created_at > '2024-01-01'               -- ข้าม user_id, status
```

### Order ใน Composite Index

```ruby
# สำหรับ query: WHERE user_id = 1 ORDER BY created_at DESC
add_index :posts, [:user_id, :created_at]

# สำหรับ query: WHERE status = 'published' ORDER BY created_at DESC
# ต้องการ status ก่อน
add_index :posts, [:status, :created_at]
```

---

## ขั้นตอนที่ 1335: Avoiding Index Bloat

### ตรวจสอบ Bloat

```sql
-- ดู index size
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- ดู table bloat
SELECT
  relname as table_name,
  n_dead_tup as dead_tuples,
  n_live_tup as live_tuples,
  round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;
```

### VACUUM และ REINDEX

```sql
-- VACUUM - ล้าง dead tuples
VACUUM posts;
VACUUM ANALYZE posts;

-- VACUUM FULL - ล้างและคืน disk space (LOCK table!)
VACUUM FULL posts;

-- REINDEX - rebuild index
REINDEX INDEX index_posts_on_status;
REINDEX TABLE posts;

-- ไม่ lock table (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY index_posts_on_status;
```

```ruby
# Rails migration สำหรับ reindex
class ReindexPosts < ActiveRecord::Migration[7.1]
  def up
    execute "REINDEX TABLE CONCURRENTLY posts"
  end
  
  def down
    # ไม่จำเป็นต้อง rollback
  end
end
```

---

## ขั้นตอนที่ 1336: Query Optimization Techniques

### N+1 Query Problem

```ruby
# BAD: N+1 queries
posts = Post.all
posts.each do |post|
  puts post.user.name  # โหลด user ทุก post!
end

# GOOD: Eager loading
posts = Post.includes(:user).all
posts.each do |post|
  puts post.user.name  # ใช้ข้อมูลที่ load ไว้แล้ว
end

# GOOD: joins สำหรับ filter
Post.joins(:user).where(users: { admin: true })

# GOOD: preload สำหรับ association แยก query
Post.preload(:user, :tags, :comments)

# GOOD: eager_load สำหรับ query + join
Post.eager_load(:user).where(users: { active: true })
```

### Using bullet gem เพื่อตรวจจับ N+1

```ruby
# Gemfile
gem 'bullet', group: :development

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
  Bullet.n_plus_one_query_enable = true
  Bullet.unused_eager_loading_enable = true
  Bullet.counter_cache_enable = true
end
```

### select() เพื่อ limit columns

```ruby
# BAD: โหลด columns ทั้งหมด
users = User.all

# GOOD: โหลดแค่ที่ต้องการ
users = User.select(:id, :name, :email)

# สำหรับ aggregation
User.select(:country, "COUNT(*) as count").group(:country)
```

### find_each สำหรับ bulk processing

```ruby
# BAD: โหลด records ทั้งหมดในครั้งเดียว (memory issue!)
User.all.each do |user|
  NewsletterService.send(user)
end

# GOOD: process เป็น batches
User.find_each(batch_size: 1000) do |user|
  NewsletterService.send(user)
end

# find_in_batches
User.find_in_batches(batch_size: 1000) do |users|
  BulkNewsletterJob.perform_later(users.map(&:id))
end
```

### Counter Cache

```ruby
# Migration
class AddPostsCountToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :posts_count, :integer, default: 0, null: false
    
    # ถ้าเพิ่ม column ในตาราง existing:
    User.find_each do |user|
      User.reset_counters(user.id, :posts)
    end
  end
end

# Model
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end

# ใช้งาน (ไม่มี DB query!)
user.posts_count  # ใช้ cached value
```

---

## ขั้นตอนที่ 1337: Database Partitioning

### Range Partitioning (ตาม date/time)

```sql
-- สร้าง partitioned table
CREATE TABLE logs (
  id BIGSERIAL,
  user_id BIGINT NOT NULL,
  action VARCHAR NOT NULL,
  created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- สร้าง partitions
CREATE TABLE logs_2024_q1 PARTITION OF logs
  FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE logs_2024_q2 PARTITION OF logs
  FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

CREATE TABLE logs_2024_q3 PARTITION OF logs
  FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');

CREATE TABLE logs_2024_q4 PARTITION OF logs
  FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');
```

### Partitioning ใน Rails

```ruby
# lib/tasks/partitioning.rake
namespace :db do
  namespace :partitions do
    desc "สร้าง partitions สำหรับ quarter ต่อไป"
    task create_next: :environment do
      next_quarter_start = Date.today.next_quarter.beginning_of_quarter
      next_quarter_end = next_quarter_start.next_quarter
      
      partition_name = "logs_#{next_quarter_start.strftime('%Y_q%q')}"
      
      ActiveRecord::Base.connection.execute(<<~SQL)
        CREATE TABLE IF NOT EXISTS #{partition_name} 
        PARTITION OF logs
        FOR VALUES FROM ('#{next_quarter_start}') TO ('#{next_quarter_end}')
      SQL
      
      puts "สร้าง partition: #{partition_name}"
    end
    
    desc "ลบ partitions เก่ากว่า 2 ปี"
    task cleanup_old: :environment do
      cutoff = 2.years.ago.beginning_of_quarter
      
      # ค้นหา partitions เก่า
      old_partitions = ActiveRecord::Base.connection.execute(<<~SQL)
        SELECT tablename FROM pg_tables
        WHERE tablename LIKE 'logs_%'
        AND tablename < 'logs_#{cutoff.strftime('%Y_q%q')}'
      SQL
      
      old_partitions.each do |row|
        ActiveRecord::Base.connection.execute(
          "DROP TABLE IF EXISTS #{row['tablename']}"
        )
        puts "ลบ partition: #{row['tablename']}"
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1338: Read Replicas

### ตั้งค่า Read Replica ใน Rails

```yaml
# config/database.yml
production:
  primary:
    adapter: postgresql
    url: <%= ENV['DATABASE_PRIMARY_URL'] %>
    
  primary_replica:
    adapter: postgresql
    url: <%= ENV['DATABASE_REPLICA_URL'] %>
    replica: true
```

```ruby
# config/application.rb
config.active_record.database_selector = { delay: 2.seconds }
config.active_record.database_resolver = ActiveRecord::Middleware::DatabaseSelector::Resolver
config.active_record.database_resolver_context = ActiveRecord::Middleware::DatabaseSelector::Resolver::Session
```

### ใช้ Read Replica manually

```ruby
# อ่านจาก replica
ActiveRecord::Base.connected_to(role: :reading) do
  Post.all  # ใช้ replica
end

# เขียนลง primary
ActiveRecord::Base.connected_to(role: :writing) do
  Post.create!(title: "New Post")  # ใช้ primary
end

# Model-level replica
class Post < ApplicationRecord
  connects_to database: { writing: :primary, reading: :primary_replica }
end
```

---

## ขั้นตอนที่ 1339: Connection Pooling (PgBouncer)

### ทำไมต้องใช้ PgBouncer?

```
ปัญหา: Rails app มี DB connections มาก
- 5 dynos × 5 threads = 25 connections
- PostgreSQL จัดการ connections ได้จำกัด (ค่า default = 100)

แก้ปัญหาด้วย PgBouncer:
- PgBouncer เป็น connection pooler
- Rails → PgBouncer → PostgreSQL
- PgBouncer ใช้ connections ซ้ำ
```

### PgBouncer Configuration

```ini
# pgbouncer.ini
[databases]
myapp = host=postgres-host port=5432 dbname=myapp_production

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction      # transaction หรือ session
max_client_conn = 100         # connections จาก clients
default_pool_size = 25        # connections ไป PostgreSQL
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
log_connections = 0
log_disconnections = 0
server_idle_timeout = 600
```

### ใช้ PgBouncer กับ Rails

```yaml
# config/database.yml
production:
  adapter: postgresql
  url: <%= ENV['PGBOUNCER_URL'] %>  # ชี้ไปที่ PgBouncer แทน
  pool: 5
  prepared_statements: false  # ต้องปิดสำหรับ transaction mode
  advisory_locks: false       # ต้องปิดสำหรับ transaction mode
```

---

## ขั้นตอนที่ 1340: Database Monitoring

### pg_stat_statements

```sql
-- เปิด extension
CREATE EXTENSION pg_stat_statements;

-- ดู top 10 slow queries
SELECT 
  query,
  calls,
  total_exec_time / 1000 as total_seconds,
  mean_exec_time / 1000 as mean_seconds,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- ดู queries ที่เรียกบ่อยที่สุด
SELECT 
  left(query, 100) as query,
  calls,
  mean_exec_time::numeric(10,2) as mean_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;
```

### Rails Performance Monitoring

```ruby
# Gemfile
gem 'prosopite'  # N+1 detection

# config/environments/development.rb
config.after_initialize do
  Prosopite.rails_logger = true
  Prosopite.raise = true  # ใน test
end

# ในแต่ละ request
around_action :detect_n_plus_one

def detect_n_plus_one
  Prosopite.scan
  yield
ensure
  Prosopite.finish
end
```

### Query Logging

```ruby
# config/initializers/query_logger.rb
if Rails.env.development?
  ActiveSupport::Notifications.subscribe('sql.active_record') do |*args|
    event = ActiveSupport::Notifications::Event.new(*args)
    
    if event.duration > 100  # slow query > 100ms
      Rails.logger.warn "[SLOW QUERY] #{event.duration.round(2)}ms: #{event.payload[:sql]}"
      Rails.logger.warn event.payload[:binds].inspect
    end
  end
end
```

---

## แบบฝึกหัดที่ 1-20

### แบบฝึกหัดที่ 1
ใช้ EXPLAIN ANALYZE วิเคราะห์ slow query ใน Blog app

### แบบฝึกหัดที่ 2
เพิ่ม B-tree indexes ให้กับ commonly queried columns

### แบบฝึกหัดที่ 3
สร้าง GIN index สำหรับ full-text search

### แบบฝึกหัดที่ 4
สร้าง Partial index เฉพาะ active/published records

### แบบฝึกหัดที่ 5
แก้ N+1 queries ด้วย includes/preload/eager_load

### แบบฝึกหัดที่ 6
เพิ่ม Counter Cache สำหรับ posts_count ใน User

### แบบฝึกหัดที่ 7
ใช้ find_each สำหรับ bulk email processing

### แบบฝึกหัดที่ 8
Implement Database Partitioning สำหรับ logs table

### แบบฝึกหัดที่ 9
ตั้งค่า Read Replica สำหรับ reporting queries

### แบบฝึกหัดที่ 10
Install และตั้งค่า PgBouncer

### แบบฝึกหัดที่ 11
ตั้งค่า pg_stat_statements และวิเคราะห์ slow queries

### แบบฝึกหัดที่ 12
Install bullet gem และแก้ N+1 warnings

### แบบฝึกหัดที่ 13
Optimize query ด้วย select() เพื่อ reduce data transfer

### แบบฝึกหัดที่ 14
สร้าง Composite indexes สำหรับ common filter combinations

### แบบฝึกหัดที่ 15
ทำ VACUUM ANALYZE และ REINDEX บน production

### แบบฝึกหัดที่ 16
ตั้งค่า Query Logging สำหรับ queries ที่ช้ากว่า 100ms

### แบบฝึกหัดที่ 17
Benchmark queries ก่อนและหลัง optimization

### แบบฝึกหัดที่ 18
สร้าง Database Health Check

### แบบฝึกหัดที่ 19
Implement caching layer สำหรับ expensive queries

### แบบฝึกหัดที่ 20
Performance audit ทั้ง app:
- ค้นหา all slow queries
- ค้นหา all N+1
- เพิ่ม missing indexes
- วัดผลก่อนและหลัง

---

## สรุป Part 61

เราได้เรียนรู้:
1. EXPLAIN ANALYZE วิเคราะห์ query plans
2. Index types: B-tree, GIN, GiST, BRIN
3. Partial indexes สำหรับ subsets
4. Composite indexes และ leftmost prefix rule
5. หลีกเลี่ยง index bloat
6. Query optimization: N+1, eager loading, counter cache
7. Database partitioning
8. Read replicas
9. PgBouncer connection pooling
10. Database monitoring

Database optimization เป็นทักษะสำคัญที่ช่วยให้ app ทำงานได้เร็วขึ้นหลายเท่าโดยไม่ต้องเพิ่ม hardware
