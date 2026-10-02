# Part 67: Advanced Active Record

## Steps 1451-1475

---

## Step 1451: Custom SQL ด้วย find_by_sql

Rails Active Record มี methods สำหรับ execute SQL โดยตรงเมื่อ Query builder ไม่เพียงพอ

### find_by_sql

```ruby
# การใช้งานพื้นฐาน
users = User.find_by_sql("SELECT * FROM users WHERE active = true ORDER BY name")

# ใช้กับ interpolation (safe)
users = User.find_by_sql(
  ["SELECT * FROM users WHERE created_at > ? AND role = ?", 30.days.ago, 'admin']
)

# ใช้ named parameters
users = User.find_by_sql(
  ["SELECT * FROM users WHERE created_at > :date AND role = :role",
   { date: 30.days.ago, role: 'admin' }]
)

# ผลลัพธ์เป็น User objects (ไม่ใช่ Hash)
users.each do |user|
  puts user.name    # สามารถเข้าถึง attributes ได้ปกติ
  puts user.email
end

# JOIN กับ subquery ซับซ้อน
orders_with_revenue = Order.find_by_sql(<<~SQL)
  SELECT 
    orders.*,
    SUM(order_items.quantity * order_items.unit_price) as total_revenue,
    COUNT(order_items.id) as items_count
  FROM orders
  INNER JOIN order_items ON order_items.order_id = orders.id
  WHERE orders.status = 'completed'
    AND orders.created_at >= NOW() - INTERVAL '30 days'
  GROUP BY orders.id
  ORDER BY total_revenue DESC
  LIMIT 100
SQL

orders_with_revenue.each do |order|
  puts "#{order.id}: #{order.total_revenue}"  # total_revenue เป็น virtual attribute
end
```

### connection.execute

```ruby
# Execute raw SQL (ไม่ return ActiveRecord objects)
result = ActiveRecord::Base.connection.execute(
  "UPDATE products SET stock = stock - 1 WHERE id = #{product_id}"
)

# ปลอดภัยกว่า: ใช้ sanitize
sql = ActiveRecord::Base.send(
  :sanitize_sql_array,
  ["UPDATE products SET stock = stock - ? WHERE id = ?", 1, product_id]
)
ActiveRecord::Base.connection.execute(sql)

# Select ด้วย connection.execute (return PG::Result)
result = ActiveRecord::Base.connection.execute(
  "SELECT id, name, email FROM users WHERE role = 'admin'"
)

# แปลงเป็น Hash array
result.to_a
# => [{"id"=>"1", "name"=>"Admin", "email"=>"admin@example.com"}, ...]

# หรือใช้ select_all สำหรับ Hash results
result = ActiveRecord::Base.connection.select_all(
  "SELECT COUNT(*) as count, role FROM users GROUP BY role"
)
result.each { |row| puts "#{row['role']}: #{row['count']}" }
```

### select_value และ select_values

```ruby
# select_value: คืนค่าเดียว
count = ActiveRecord::Base.connection.select_value(
  "SELECT COUNT(*) FROM users WHERE active = true"
)
puts count  # => "42" (string)
puts count.to_i  # => 42

# select_values: คืน array ของค่าแรกของแต่ละ row
ids = ActiveRecord::Base.connection.select_values(
  "SELECT id FROM users WHERE role = 'admin' ORDER BY id"
)
puts ids  # => ["1", "3", "7"]

# select_rows: คืน array of arrays
rows = ActiveRecord::Base.connection.select_rows(
  "SELECT id, name, email FROM users LIMIT 5"
)
rows.each do |id, name, email|
  puts "#{id}: #{name} (#{email})"
end
```

---

## Step 1452: Window Functions

Window functions เป็นหนึ่งใน features ที่ทรงพลังที่สุดใน SQL สำหรับ analytics

### ROW_NUMBER

```ruby
# หา rank ของ users ตามยอดซื้อ
users_with_rank = User.select(
  "users.*",
  "ROW_NUMBER() OVER (ORDER BY total_spent DESC) as spending_rank"
).joins(
  "LEFT JOIN (
    SELECT user_id, SUM(total) as total_spent
    FROM orders WHERE status = 'completed'
    GROUP BY user_id
  ) user_totals ON user_totals.user_id = users.id"
)

users_with_rank.each do |user|
  puts "#{user.spending_rank}. #{user.name}: #{user.total_spent || 0}"
end

# PARTITION BY: rank ภายในแต่ละ category
products_with_category_rank = Product.select(
  "products.*",
  "ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) as rank_in_category"
)
```

### LAG และ LEAD

```ruby
# เปรียบเทียบยอดขายกับเดือนก่อน
monthly_sales = Order.select(<<~SQL)
  EXTRACT(YEAR FROM created_at) as year,
  EXTRACT(MONTH FROM created_at) as month,
  SUM(total) as revenue,
  LAG(SUM(total)) OVER (ORDER BY EXTRACT(YEAR FROM created_at), EXTRACT(MONTH FROM created_at)) as prev_month_revenue,
  SUM(total) - LAG(SUM(total)) OVER (ORDER BY EXTRACT(YEAR FROM created_at), EXTRACT(MONTH FROM created_at)) as revenue_change
SQL
.where(status: 'completed')
.group(
  "EXTRACT(YEAR FROM created_at)",
  "EXTRACT(MONTH FROM created_at)"
)

# LEAD: ดูค่าถัดไป
next_month_revenue = Order.select(<<~SQL)
  EXTRACT(MONTH FROM created_at) as month,
  SUM(total) as this_month,
  LEAD(SUM(total)) OVER (ORDER BY EXTRACT(MONTH FROM created_at)) as next_month_forecast
SQL
.where(created_at: 1.year.ago..)
.group("EXTRACT(MONTH FROM created_at)")
```

### RANK และ DENSE_RANK

```ruby
# RANK: ถ้า rank เท่ากัน ข้าม rank ถัดไป (1, 2, 2, 4)
# DENSE_RANK: ไม่ข้าม rank (1, 2, 2, 3)
top_sellers = Product.find_by_sql(<<~SQL)
  SELECT 
    products.*,
    SUM(order_items.quantity) as total_sold,
    RANK() OVER (ORDER BY SUM(order_items.quantity) DESC) as rank,
    DENSE_RANK() OVER (ORDER BY SUM(order_items.quantity) DESC) as dense_rank
  FROM products
  INNER JOIN order_items ON order_items.product_id = products.id
  INNER JOIN orders ON orders.id = order_items.order_id
  WHERE orders.status = 'completed'
  GROUP BY products.id
  ORDER BY rank
  LIMIT 20
SQL
```

### Running Total และ Moving Average

```ruby
# Running total (ยอดสะสม)
daily_revenue_with_cumulative = ActiveRecord::Base.connection.select_all(<<~SQL)
  SELECT
    DATE(created_at) as date,
    SUM(total) as daily_revenue,
    SUM(SUM(total)) OVER (ORDER BY DATE(created_at)) as cumulative_revenue,
    AVG(SUM(total)) OVER (
      ORDER BY DATE(created_at)
      ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as rolling_7day_avg
  FROM orders
  WHERE status = 'completed'
    AND created_at >= NOW() - INTERVAL '90 days'
  GROUP BY DATE(created_at)
  ORDER BY date
SQL
```

### NTILE (ทำ percentile)

```ruby
# แบ่ง users ออกเป็น quartiles ตามยอดซื้อ
user_quartiles = User.find_by_sql(<<~SQL)
  SELECT
    users.*,
    COALESCE(total_spent, 0) as total_spent,
    NTILE(4) OVER (ORDER BY COALESCE(total_spent, 0)) as quartile,
    NTILE(10) OVER (ORDER BY COALESCE(total_spent, 0)) as decile,
    PERCENT_RANK() OVER (ORDER BY COALESCE(total_spent, 0)) as percent_rank
  FROM users
  LEFT JOIN (
    SELECT user_id, SUM(total) as total_spent
    FROM orders WHERE status = 'completed'
    GROUP BY user_id
  ) spending ON spending.user_id = users.id
  ORDER BY total_spent DESC
SQL

user_quartiles.each do |user|
  puts "#{user.name}: Q#{user.quartile}, P#{(user.percent_rank.to_f * 100).round(1)}%"
end
```

---

## Step 1453: CTEs (Common Table Expressions)

CTEs ทำให้ query ซับซ้อนอ่านง่ายขึ้นมากและ reuse ได้

### Basic CTE

```ruby
# ตัวอย่าง: หา users ที่มี orders ในช่วง 30 วันที่ผ่านมา
active_users_query = <<~SQL
  WITH recent_orders AS (
    SELECT DISTINCT user_id
    FROM orders
    WHERE created_at >= NOW() - INTERVAL '30 days'
      AND status = 'completed'
  ),
  high_value_orders AS (
    SELECT user_id, SUM(total) as total_spent
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
    HAVING SUM(total) > 5000
  )
  SELECT 
    users.*,
    COALESCE(hvo.total_spent, 0) as lifetime_value,
    CASE WHEN ro.user_id IS NOT NULL THEN true ELSE false END as recently_active
  FROM users
  LEFT JOIN recent_orders ro ON ro.user_id = users.id
  LEFT JOIN high_value_orders hvo ON hvo.user_id = users.id
  WHERE ro.user_id IS NOT NULL OR hvo.user_id IS NOT NULL
SQL

result = User.find_by_sql(active_users_query)
```

### Recursive CTE (สำหรับ hierarchical data)

```ruby
# องค์กร: หา sub-tree ทั้งหมดของ department
department_tree_query = <<~SQL
  WITH RECURSIVE dept_tree AS (
    -- Base case: root department
    SELECT id, name, parent_id, 0 AS depth, name::text AS path
    FROM departments
    WHERE id = :root_dept_id
    
    UNION ALL
    
    -- Recursive case
    SELECT 
      d.id, 
      d.name, 
      d.parent_id,
      dt.depth + 1,
      dt.path || ' > ' || d.name
    FROM departments d
    INNER JOIN dept_tree dt ON dt.id = d.parent_id
    WHERE dt.depth < 10  -- ป้องกัน infinite loop
  )
  SELECT * FROM dept_tree ORDER BY path
SQL

departments = Department.find_by_sql(
  [department_tree_query, root_dept_id: params[:id]]
)

# Comment threads (ข้อความตอบกลับซ้อนกัน)
threaded_comments_query = <<~SQL
  WITH RECURSIVE comment_tree AS (
    SELECT *, 0 AS depth, ARRAY[id] AS path
    FROM comments
    WHERE post_id = :post_id AND parent_id IS NULL
    
    UNION ALL
    
    SELECT 
      c.*, 
      ct.depth + 1,
      ct.path || c.id
    FROM comments c
    INNER JOIN comment_tree ct ON ct.id = c.parent_id
  )
  SELECT * FROM comment_tree ORDER BY path
SQL

comments = Comment.find_by_sql(
  [threaded_comments_query, post_id: @post.id]
)
```

### Arel CTE (Rails 7+)

```ruby
# Rails 7+ รองรับ CTE ผ่าน Arel
recent_active = Arel::Table.new(:recent_active_users)

cte = Arel::Nodes::As.new(
  recent_active,
  User.where('last_sign_in_at > ?', 30.days.ago).arel
)

query = User.from(
  User.arel_table
    .join(recent_active, Arel::Nodes::InnerJoin)
    .on(User.arel_table[:id].eq(recent_active[:id]))
    .with(cte)
    .compile_update(
      User.arel_table.columns.map { |c| [c.name, c] }.to_h
    )
)
```

---

## Step 1454: JSON Queries ใน PostgreSQL

PostgreSQL มี JSON support ที่ทรงพลังมาก Rails ก็รองรับได้ดี

### เก็บข้อมูล JSON ใน Column

```ruby
# migration
class AddMetadataToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :preferences, :jsonb, default: {}
    add_column :users, :metadata, :jsonb, default: {}
    
    # เพิ่ม index สำหรับ JSONB
    add_index :users, :preferences, using: :gin
    add_index :users, :metadata, using: :gin
  end
end

# Model
class User < ApplicationRecord
  # store_accessor ให้ access JSON fields แบบ attribute
  store_accessor :preferences, :theme, :language, :notifications_enabled
  store_accessor :metadata, :source, :referral_code, :utm_campaign
  
  # Default values
  attribute :preferences, default: {
    'theme' => 'light',
    'language' => 'th',
    'notifications_enabled' => true
  }
end
```

### Query JSON Fields

```ruby
# หา users ที่ theme เป็น dark
dark_theme_users = User.where("preferences->>'theme' = ?", 'dark')

# หา users ที่เปิด notifications
notif_enabled = User.where("(preferences->>'notifications_enabled')::boolean = true")

# ตรวจสอบ key มีอยู่หรือไม่
has_language = User.where("preferences ? 'language'")

# JSONB contains operator (@>)
thai_users = User.where("preferences @> ?", { language: 'th' }.to_json)

# ค้นหาใน nested JSON
# { settings: { email: { daily_digest: true } } }
daily_digest_users = User.where(
  "preferences->'settings'->'email'->>'daily_digest' = ?", 'true'
)

# หา users ที่มี metadata บางอย่าง
users_from_google = User.where("metadata->>'source' = ?", 'google')
users_with_utm = User.where("metadata ? 'utm_campaign'")

# Update JSON field
user.update!(preferences: user.preferences.merge('theme' => 'dark'))

# หรือใช้ jsonb_set
User.where(id: user.id).update_all(
  "preferences = jsonb_set(preferences, '{theme}', '\"dark\"')"
)

# เพิ่ม key ใหม่ใน array ใน JSON
User.where(id: user.id).update_all(
  "metadata = metadata || '{\"last_feature_used\": \"dashboard\"}'::jsonb"
)
```

### JSON Aggregation

```ruby
# รวม JSON objects
user_stats = User.select(<<~SQL)
  users.*,
  (
    SELECT json_agg(
      json_build_object(
        'id', orders.id,
        'total', orders.total,
        'status', orders.status,
        'created_at', orders.created_at
      ) ORDER BY orders.created_at DESC
    )
    FROM orders
    WHERE orders.user_id = users.id
    LIMIT 5
  ) as recent_orders_json
SQL

# json_object_agg: สร้าง JSON object จาก key-value pairs
category_stats = Order.select(<<~SQL)
  json_object_agg(
    categories.name,
    json_build_object(
      'count', COUNT(orders.id),
      'revenue', SUM(orders.total)
    )
  ) as stats_by_category
SQL
.joins(items: { product: :category })
.where(status: 'completed')
.first
```

---

## Step 1455: Full-Text Search

PostgreSQL มี full-text search ที่ทรงพลัง Rails รองรับผ่าน built-in methods

### ตั้งค่า Full-Text Search

```ruby
# Migration: เพิ่ม search vector column
class AddSearchVectorToProducts < ActiveRecord::Migration[7.0]
  def change
    add_column :products, :search_vector, :tsvector
    
    # สร้าง index
    add_index :products, :search_vector, using: :gin
    
    # สร้าง trigger ให้ update อัตโนมัติ
    execute <<~SQL
      CREATE OR REPLACE FUNCTION update_product_search_vector()
      RETURNS TRIGGER AS $$
      BEGIN
        NEW.search_vector := 
          to_tsvector('thai', COALESCE(NEW.name, '')) ||
          to_tsvector('english', COALESCE(NEW.description, '')) ||
          to_tsvector('english', COALESCE(NEW.sku, ''));
        RETURN NEW;
      END;
      $$ LANGUAGE plpgsql;
      
      CREATE TRIGGER update_product_search_vector_trigger
        BEFORE INSERT OR UPDATE ON products
        FOR EACH ROW EXECUTE FUNCTION update_product_search_vector();
    SQL
    
    # Populate existing records
    execute "UPDATE products SET search_vector = to_tsvector('english', COALESCE(name, '') || ' ' || COALESCE(description, ''))"
  end
end
```

### ใช้งาน Full-Text Search

```ruby
# Model
class Product < ApplicationRecord
  scope :search, ->(query) {
    return all if query.blank?
    
    where(
      "search_vector @@ plainto_tsquery('english', ?)", 
      query
    )
    .select(
      "products.*",
      "ts_rank(search_vector, plainto_tsquery('english', #{connection.quote(query)})) as rank"
    )
    .order('rank DESC')
  }
  
  scope :search_with_highlight, ->(query) {
    return all if query.blank?
    
    tsquery = "plainto_tsquery('english', #{connection.quote(query)})"
    
    where("search_vector @@ #{tsquery}")
    .select(
      "products.*",
      "ts_headline('english', products.description, #{tsquery}, 
       'MaxWords=50, MinWords=20, StartSel=<mark>, StopSel=</mark>') as highlighted_description",
      "ts_rank(search_vector, #{tsquery}) as rank"
    )
    .order('rank DESC')
  }
end

# การใช้งาน
products = Product.search('iPhone case')
products = Product.search_with_highlight('leather wallet')

# ใน Controller
class ProductsController < ApplicationController
  def index
    @products = Product
      .search(params[:q])
      .active
      .page(params[:page])
      .per(20)
  end
end
```

### pg_search Gem (ทางเลือก)

```ruby
# Gemfile
gem 'pg_search'

# Model
class Product < ApplicationRecord
  include PgSearch::Model
  
  pg_search_scope :search_full,
    against: {
      name: 'A',        # น้ำหนักสูงสุด
      sku: 'B',
      description: 'C'  # น้ำหนักน้อยที่สุด
    },
    using: {
      tsearch: {
        prefix: true,      # รองรับ prefix search
        any_word: true,
        dictionary: 'english'
      },
      trigram: {
        only: [:name]      # fuzzy search สำหรับชื่อ
      }
    }
  
  pg_search_scope :search_trigram,
    against: [:name, :description],
    using: {
      trigram: {
        threshold: 0.1
      }
    }
end
```

---

## Step 1456: Optimistic Locking

ป้องกัน race conditions โดยตรวจสอบว่า record ไม่ได้ถูกแก้ไขระหว่างที่กำลัง save

### ตั้งค่า Optimistic Locking

```ruby
# Migration
class AddLockVersionToOrders < ActiveRecord::Migration[7.0]
  def change
    add_column :orders, :lock_version, :integer, default: 0, null: false
  end
end

# Model (Rails จัดการอัตโนมัติเมื่อมี lock_version column)
class Order < ApplicationRecord
  # Rails จะ check lock_version อัตโนมัติ
end
```

### การใช้งาน

```ruby
# User A และ User B โหลด order เดียวกัน
order_a = Order.find(1)  # lock_version = 0
order_b = Order.find(1)  # lock_version = 0

# User A update ก่อน
order_a.update!(status: 'processing')  # lock_version เป็น 1

# User B พยายาม update เดิม (lock_version ยังเป็น 0)
begin
  order_b.update!(status: 'cancelled')  # StaleObjectError!
rescue ActiveRecord::StaleObjectError => e
  puts "Order ถูกแก้ไขโดยคนอื่นแล้ว กรุณาโหลดใหม่"
  # โหลด order ใหม่และแสดงความขัดแย้ง
  current_order = Order.find(1)
  render :edit, locals: { conflict: true, fresh_order: current_order }
end

# ตัวอย่างใน Service Object
class UpdateOrderStatus
  def call(order:, new_status:, user:)
    retries = 0
    
    begin
      order.update!(status: new_status, updated_by: user)
    rescue ActiveRecord::StaleObjectError => e
      retries += 1
      if retries <= 3
        order.reload
        retry
      else
        raise "ไม่สามารถ update ได้หลังลองซ้ำ 3 ครั้ง"
      end
    end
  end
end
```

### Handling Conflicts ใน Controller

```ruby
class OrdersController < ApplicationController
  def update
    @order = Order.find(params[:id])
    
    begin
      @order.update!(order_params)
      redirect_to @order, notice: 'อัพเดทสำเร็จ'
    rescue ActiveRecord::StaleObjectError
      # ดึง order เวอร์ชันใหม่
      fresh_order = Order.find(params[:id])
      
      flash.now[:alert] = "Order ถูกแก้ไขโดยผู้อื่นแล้ว ข้อมูลด้านล่างคือเวอร์ชันปัจจุบัน"
      @order = fresh_order
      render :edit
    end
  end
  
  private
  
  def order_params
    params.require(:order).permit(:status, :notes, :lock_version)
    # ต้องส่ง lock_version ผ่าน hidden field ใน form
  end
end
```

```erb
<%# ใน form %> 
<%= f.hidden_field :lock_version %>
```

---

## Step 1457: Pessimistic Locking

Lock record ระหว่าง transaction เพื่อป้องกัน concurrent writes

### lock! Method

```ruby
# lock! เพิ่ม FOR UPDATE ใน SQL query
Order.transaction do
  order = Order.lock.find(params[:id])
  # หรือ
  order = Order.find(params[:id])
  order.lock!
  
  # ตอนนี้ record ถูก lock จนกว่า transaction จะจบ
  # ถ้า user อื่นพยายาม lock เดิม จะรอจนกว่า lock จะถูก release
  
  if order.stock > 0
    order.decrement!(:stock)
  end
end

# SKIP LOCKED: ข้าม records ที่ถูก lock
# มีประโยชน์สำหรับ job queues
Order.where(status: 'pending')
     .order(:created_at)
     .limit(10)
     .lock('FOR UPDATE SKIP LOCKED')
     .each do |order|
       ProcessOrderJob.perform_now(order)
     end

# FOR SHARE: อนุญาตให้อ่านแต่ไม่ให้ write
Order.transaction do
  order = Order.lock('FOR SHARE').find(params[:id])
  # อ่านได้ แต่ write ไม่ได้จนกว่า transaction จะจบ
  puts order.total
end
```

### ตัวอย่างจริง: Inventory Management

```ruby
# ป้องกัน overselling
class ReserveInventory
  def self.call(product_id:, quantity:, order:)
    Product.transaction do
      # Lock product row
      product = Product.lock.find(product_id)
      
      if product.stock >= quantity
        product.decrement!(:stock, quantity)
        
        Reservation.create!(
          product: product,
          order: order,
          quantity: quantity,
          reserved_at: Time.current,
          expires_at: 30.minutes.from_now
        )
        
        { success: true, product: product }
      else
        { 
          success: false, 
          error: "สินค้าไม่เพียงพอ (มีเพียง #{product.stock} ชิ้น)"
        }
      end
    end
  end
end
```

---

## Step 1458: Transactions

Transactions ทำให้หลาย operations เป็น atomic (ทำสำเร็จทั้งหมด หรือ rollback ทั้งหมด)

### Basic Transaction

```ruby
# transaction do: ถ้ามี exception ใดๆ จะ rollback ทั้งหมด
ActiveRecord::Base.transaction do
  order = Order.create!(user: current_user, total: cart.total)
  
  cart.items.each do |cart_item|
    OrderItem.create!(
      order: order,
      product: cart_item.product,
      quantity: cart_item.quantity,
      unit_price: cart_item.product.price
    )
    
    # ลด stock
    cart_item.product.decrement!(:stock, cart_item.quantity)
  end
  
  # ชำระเงิน (ถ้า fail จะ rollback ทั้งหมด)
  payment = stripe_charge(order)
  order.update!(status: 'paid', payment_id: payment.id)
end

# หรือ call จาก model
class Order < ApplicationRecord
  def self.create_with_items!(user:, cart:)
    transaction do
      order = create!(user: user, total: cart.total, status: 'pending')
      cart.items.each do |item|
        order.items.create!(
          product: item.product,
          quantity: item.quantity,
          unit_price: item.product.current_price
        )
      end
      order
    end
  end
end
```

### Transaction Callbacks

```ruby
class Order < ApplicationRecord
  after_commit :send_confirmation_email, on: :create
  after_commit :notify_status_change, on: :update
  after_rollback :log_failed_transaction
  
  # after_commit ดีกว่า after_create สำหรับ side effects
  # เพราะ commit เกิดขึ้นหลัง transaction สำเร็จจริงๆ
  
  def send_confirmation_email
    OrderMailer.confirmation(self).deliver_later
  end
  
  def notify_status_change
    if saved_change_to_status?
      OrderStatusChangeJob.perform_later(id, status)
    end
  end
  
  def log_failed_transaction
    Rails.logger.error "Transaction rolled back for Order ##{id}"
  end
end
```

### Transaction Isolation Levels

```ruby
# READ COMMITTED (default)
Order.transaction do
  # ...
end

# REPEATABLE READ: ป้องกัน non-repeatable reads
Order.transaction(isolation: :repeatable_read) do
  count1 = Order.count  # 100
  # แม้ user อื่น insert order ใหม่ระหว่างนี้
  count2 = Order.count  # ยังคง 100 (same snapshot)
end

# SERIALIZABLE: isolation level สูงสุด
Order.transaction(isolation: :serializable) do
  # ป้องกันทุก anomalies แต่ช้าที่สุด
end
```

---

## Step 1459: Nested Transactions

Rails รองรับ nested transactions แต่มีข้อควรระวัง

```ruby
# Nested transaction ด้วย requires_new: true
User.transaction do
  User.create!(name: 'ผู้ใช้ 1', email: 'user1@example.com')
  
  # Sub-transaction แยกออกไป (ถ้า fail ไม่กระทบ outer transaction)
  User.transaction(requires_new: true) do
    User.create!(name: 'ผู้ใช้ 2', email: 'INVALID_EMAIL')
  end
rescue ActiveRecord::RecordInvalid => e
  # nested transaction rollback แต่ outer transaction ยังดำเนินต่อ
  Rails.logger.warn "Sub-transaction failed: #{e.message}"
end
# ผู้ใช้ 1 ถูก save แต่ ผู้ใช้ 2 ไม่ถูก save

# ⚠️ ไม่ใช้ requires_new: จะเป็น savepoint แทน
User.transaction do
  User.create!(name: 'สมชาย')
  
  User.transaction do  # <-- นี่ไม่ใช่ transaction ใหม่จริงๆ!
    User.create!(name: 'invalid', email: 'INVALID')
  end  # Exception จะ propagate ไปยัง outer transaction
end
```

---

## Step 1460: Savepoints

Savepoints ให้ roll back บางส่วนของ transaction ได้

```ruby
# ใช้ savepoints ผ่าน nested transactions
Order.transaction do
  order = Order.create!(status: 'pending', user: current_user)
  
  # Savepoint 1
  Order.transaction(requires_new: true) do
    apply_coupon(order)
  rescue InvalidCouponError
    # Roll back ถึง savepoint 1 เท่านั้น
    raise ActiveRecord::Rollback
  end
  
  # Savepoint 2  
  Order.transaction(requires_new: true) do
    process_payment(order)
  rescue PaymentError => e
    raise ActiveRecord::Rollback
    # Order ยังคงอยู่ แต่ payment ไม่ถูกประมวลผล
  end
  
  order.update!(status: 'processing')
end
```

---

## Step 1461: STI (Single Table Inheritance)

STI เก็บ subclasses หลายชนิดในตาราง database เดียว

### การตั้งค่า STI

```ruby
# migration
class CreateVehicles < ActiveRecord::Migration[7.0]
  def change
    create_table :vehicles do |t|
      t.string :type, null: false  # จำเป็นสำหรับ STI
      t.string :name, null: false
      t.string :make
      t.string :model
      t.integer :year
      t.decimal :price, precision: 10, scale: 2
      
      # Car-specific
      t.integer :doors
      t.string :fuel_type
      
      # Truck-specific
      t.decimal :payload_capacity
      
      # Motorcycle-specific
      t.string :engine_size
      
      t.timestamps
    end
    
    add_index :vehicles, :type
  end
end
```

### Model Classes

```ruby
# app/models/vehicle.rb - Parent class
class Vehicle < ApplicationRecord
  validates :name, :make, :model, :year, presence: true
  
  scope :available, -> { where(available: true) }
  scope :by_price, ->(min, max) { where(price: min..max) }
  
  def display_name
    "#{year} #{make} #{model}"
  end
end

# app/models/car.rb
class Car < Vehicle
  FUEL_TYPES = %w[gasoline diesel electric hybrid].freeze
  
  validates :doors, inclusion: { in: 2..5 }
  validates :fuel_type, inclusion: { in: FUEL_TYPES }
  
  scope :electric, -> { where(fuel_type: 'electric') }
  scope :family_size, -> { where('doors >= ?', 4) }
  
  def electric?
    fuel_type == 'electric'
  end
  
  def display_info
    "#{display_name} - #{doors} doors, #{fuel_type}"
  end
end

# app/models/truck.rb
class Truck < Vehicle
  validates :payload_capacity, presence: true, numericality: { greater_than: 0 }
  
  scope :heavy_duty, -> { where('payload_capacity > ?', 5000) }
  
  def heavy_duty?
    payload_capacity > 5000
  end
end

# app/models/motorcycle.rb
class Motorcycle < Vehicle
  validates :engine_size, presence: true
  
  scope :large_engine, -> { where('engine_size::integer >= 600') }
end
```

### STI ใน Controller

```ruby
# Query ทุก vehicles
all_vehicles = Vehicle.all

# Query เฉพาะ type
cars = Car.all
trucks = Truck.available
motorcycles = Motorcycle.where(year: 2020..)

# STI works with polymorphism
vehicles = Vehicle.where(type: ['Car', 'Truck'])

# ตรวจสอบ type
vehicle = Vehicle.find(1)
vehicle.class  # => Car
vehicle.is_a?(Car)  # => true
vehicle.type  # => "Car"

# class ใน controller
class VehiclesController < ApplicationController
  def index
    @vehicles = case params[:type]
    when 'Car'        then Car.all
    when 'Truck'      then Truck.all
    when 'Motorcycle' then Motorcycle.all
    else              Vehicle.all
    end
  end
end
```

### ข้อดีและข้อเสียของ STI

```
ข้อดี:
✅ ง่ายต่อการ query ข้าม types
✅ Polymorphic associations ทำงานได้ง่าย
✅ ไม่ต้องมี joins ข้าม tables

ข้อเสีย:
❌ Null columns เยอะ (เพราะแต่ละ type ใช้ columns ต่างกัน)
❌ ตาราง scale ไม่ดีเมื่อ types เยอะมาก
❌ Schema changes ยาก (ต้อง add column ให้ทุก types)
```

---

## Step 1462: Delegated Types

Rails 6.1+ แนะนำ Delegated Types เป็นทางเลือกที่ดีกว่า STI

### การตั้งค่า Delegated Types

```ruby
# migration สำหรับ Entry (parent)
class CreateEntries < ActiveRecord::Migration[7.0]
  def change
    create_table :entries do |t|
      t.string :entryable_type, null: false
      t.integer :entryable_id, null: false
      t.string :title
      t.references :author, foreign_key: { to_table: :users }
      t.timestamps
      
      t.index [:entryable_type, :entryable_id], unique: true
    end
  end
end

# migration สำหรับ Message
class CreateMessages < ActiveRecord::Migration[7.0]
  def change
    create_table :messages do |t|
      t.text :content
      t.string :message_type, default: 'text'
      t.timestamps
    end
  end
end

# migration สำหรับ Comment  
class CreateComments < ActiveRecord::Migration[7.0]
  def change
    create_table :comments do |t|
      t.text :body
      t.integer :post_id
      t.boolean :approved, default: false
      t.timestamps
    end
  end
end
```

### Model Classes สำหรับ Delegated Types

```ruby
# app/models/entry.rb
class Entry < ApplicationRecord
  delegated_type :entryable, types: %w[Message Comment Post]
  
  belongs_to :author, class_name: 'User'
  
  scope :recent, -> { order(created_at: :desc) }
  
  # Methods ที่ใช้ร่วมกัน
  def summary
    title || entryable.to_s.truncate(100)
  end
end

# app/models/message.rb
class Message < ApplicationRecord
  include Entryable
  
  TYPES = %w[text image video file].freeze
  validates :content, presence: true
  validates :message_type, inclusion: { in: TYPES }
  
  def text?
    message_type == 'text'
  end
end

# app/models/comment.rb
class Comment < ApplicationRecord
  include Entryable
  
  belongs_to :post
  validates :body, presence: true, length: { minimum: 10 }
  
  scope :approved, -> { where(approved: true) }
end

# app/concerns/entryable.rb
module Entryable
  extend ActiveSupport::Concern
  
  included do
    has_one :entry, as: :entryable, touch: true
  end
end
```

### การใช้งาน Delegated Types

```ruby
# สร้าง Message entry
message_entry = Entry.create!(
  author: current_user,
  title: 'ข้อความใหม่',
  entryable: Message.new(content: 'สวัสดีทุกคน!')
)

# สร้าง Comment entry
comment_entry = Entry.create!(
  author: current_user,
  entryable: Comment.new(body: 'ความเห็นของฉัน...', post: @post)
)

# Query
Entry.all.each do |entry|
  case entry.entryable_type
  when 'Message'
    puts "Message: #{entry.message.content}"
  when 'Comment'
    puts "Comment: #{entry.comment.body}"
  end
end

# Type checking
entry.message?   # => true/false
entry.comment?   # => true/false
entry.message    # => #<Message> หรือ nil
entry.comment    # => #<Comment> หรือ nil

# Query specific type
Entry.where(entryable_type: 'Message')
Entry.messages   # delegate type creates this scope
Entry.comments
```

---

## Step 1463: Advanced Query Techniques

### Preloading Strategies

```ruby
# includes: lazy loading + 2 queries
users = User.includes(:orders, :profile)
# SELECT * FROM users
# SELECT * FROM orders WHERE user_id IN (...)
# SELECT * FROM profiles WHERE user_id IN (...)

# eager_load: JOIN (1 query, ใช้ WHERE clause ข้าม associations ได้)
users = User.eager_load(:orders).where(orders: { status: 'completed' })
# SELECT users.*, orders.* FROM users LEFT JOIN orders ON ...

# preload: เหมือน includes แต่ force 2 queries เสมอ
users = User.preload(:orders)

# strict_loading: ป้องกัน N+1 โดย raise error
user = User.strict_loading.first
user.orders  # => StrictLoadingViolationError!

# ใช้ strict_loading ใน development
class ApplicationRecord < ActiveRecord::Base
  self.strict_loading_by_default = true if Rails.env.development?
end
```

### Batch Processing

```ruby
# find_each: process records ทีละ batch (ไม่ load ทั้งหมดเข้า memory)
User.find_each(batch_size: 1000) do |user|
  user.recalculate_stats!
end

# find_in_batches: ให้ batch ทั้ง array
User.find_in_batches(batch_size: 500) do |users|
  UserMailer.newsletter(users).deliver_later
end

# in_batches: ให้ relation (Rails 5+)
User.active.in_batches(of: 1000).each_with_index do |batch, index|
  puts "Processing batch #{index + 1}: #{batch.count} users"
  batch.update_all(last_newsletter_at: Time.current)
end

# in_batches.each_record: iterate records ใน batches
User.in_batches(of: 200).each_record do |user|
  user.update!(status: compute_status(user))
end
```

### Scoping และ Default Scopes

```ruby
class Product < ApplicationRecord
  # Default scope (ระวัง: ส่งผลกระทบทุก query)
  default_scope { where(active: true).order(:name) }
  
  # ออก default scope
  Product.unscoped.all  # ไม่มี where active = true
  
  # Named scopes
  scope :available, -> { where('stock > 0') }
  scope :by_price, ->(min, max) { where(price: min..max) }
  scope :recent, -> { order(created_at: :desc).limit(10) }
  scope :featured, -> { where(featured: true).order(sort_order: :asc) }
  scope :for_category, ->(cat) { where(category: cat) if cat.present? }
  
  # Scope chain
  scope :on_sale, -> {
    joins(:promotions)
    .where(promotions: { active: true })
    .distinct
  }
end

# Scope merging
popular_available = Product.available.merge(Product.featured)

# any_of (แบบ OR)
# ต้อง install gem 'activerecord-or'
User.where(role: 'admin').or(User.where(role: 'moderator'))
# SQL: WHERE (role = 'admin' OR role = 'moderator')
```

---

## Step 1464-1475: แบบฝึกหัด 25 ข้อ

### แบบฝึกหัดที่ 1: Custom SQL Query

```ruby
# สร้าง method สำหรับหา users ที่มีกิจกรรมมากที่สุดในแต่ละเดือน
class User < ApplicationRecord
  def self.most_active_by_month(year: Date.current.year)
    find_by_sql([<<~SQL, year: year])
      WITH monthly_activity AS (
        SELECT 
          user_id,
          EXTRACT(MONTH FROM created_at) as month,
          COUNT(*) as action_count,
          RANK() OVER (
            PARTITION BY EXTRACT(MONTH FROM created_at)
            ORDER BY COUNT(*) DESC
          ) as monthly_rank
        FROM user_activities
        WHERE EXTRACT(YEAR FROM created_at) = :year
        GROUP BY user_id, EXTRACT(MONTH FROM created_at)
      )
      SELECT 
        users.*,
        ma.month,
        ma.action_count,
        ma.monthly_rank
      FROM users
      INNER JOIN monthly_activity ma ON ma.user_id = users.id
      WHERE ma.monthly_rank = 1
      ORDER BY ma.month
    SQL
  end
end
```

### แบบฝึกหัดที่ 2: Window Function Report

```ruby
# Revenue report ด้วย window functions
class RevenueReport
  def self.generate(year: Date.current.year)
    ActiveRecord::Base.connection.select_all(<<~SQL)
      SELECT
        EXTRACT(MONTH FROM created_at) as month,
        SUM(total) as monthly_revenue,
        SUM(SUM(total)) OVER (ORDER BY EXTRACT(MONTH FROM created_at)) as ytd_revenue,
        AVG(SUM(total)) OVER () as avg_monthly_revenue,
        SUM(total) - LAG(SUM(total), 1, 0) OVER (
          ORDER BY EXTRACT(MONTH FROM created_at)
        ) as mom_change,
        ROUND(
          (SUM(total) - LAG(SUM(total), 1, SUM(total)) OVER (
            ORDER BY EXTRACT(MONTH FROM created_at)
          )) / NULLIF(LAG(SUM(total)) OVER (
            ORDER BY EXTRACT(MONTH FROM created_at)
          ), 0) * 100, 2
        ) as mom_change_pct
      FROM orders
      WHERE status = 'completed'
        AND EXTRACT(YEAR FROM created_at) = #{year}
      GROUP BY EXTRACT(MONTH FROM created_at)
      ORDER BY month
    SQL
  end
end
```

### แบบฝึกหัดที่ 3: CTE สำหรับ Category Tree

```ruby
class Category < ApplicationRecord
  belongs_to :parent, class_name: 'Category', optional: true
  has_many :children, class_name: 'Category', foreign_key: :parent_id
  
  def self.full_tree_for(root_id)
    find_by_sql([<<~SQL, root_id: root_id])
      WITH RECURSIVE category_tree AS (
        SELECT id, name, parent_id, 0 AS level, 
               name::text AS breadcrumb,
               ARRAY[id] AS ancestors
        FROM categories
        WHERE id = :root_id
        
        UNION ALL
        
        SELECT 
          c.id, c.name, c.parent_id,
          ct.level + 1,
          ct.breadcrumb || ' > ' || c.name,
          ct.ancestors || c.id
        FROM categories c
        INNER JOIN category_tree ct ON ct.id = c.parent_id
      )
      SELECT *, ARRAY_LENGTH(ancestors, 1) as depth
      FROM category_tree
      ORDER BY breadcrumb
    SQL
  end
  
  def ancestors_list
    Category.find_by_sql([<<~SQL, id: self.id])
      WITH RECURSIVE ancestors AS (
        SELECT * FROM categories WHERE id = :id
        UNION ALL
        SELECT c.* FROM categories c
        INNER JOIN ancestors a ON a.parent_id = c.id
      )
      SELECT * FROM ancestors WHERE id != :id
      ORDER BY id
    SQL
  end
end
```

### แบบฝึกหัดที่ 4: JSON Field Operations

```ruby
# สร้าง User model ที่มี JSONB preferences
class User < ApplicationRecord
  store_accessor :preferences, 
    :theme, :language, :timezone, 
    :email_notifications, :sms_notifications
  
  store_accessor :metadata,
    :source, :referral_code, :device_type
  
  def self.with_preference(key, value)
    where("preferences @> ?", { key => value }.to_json)
  end
  
  def self.missing_preference(key)
    where("NOT (preferences ? ?)", key)
  end
  
  def self.by_language(lang)
    where("preferences->>'language' = ?", lang)
  end
  
  def update_preference(key, value)
    update!(
      preferences: preferences.merge(key.to_s => value)
    )
  end
  
  def toggle_notification(type)
    current = preferences[type.to_s]
    update_preference(type, !current)
  end
end

# ใช้งาน
thai_users = User.by_language('th')
users_without_timezone = User.missing_preference('timezone')
dark_theme_users = User.with_preference('theme', 'dark')
```

### แบบฝึกหัดที่ 5: Full-Text Search Implementation

```ruby
# ตั้งค่า FTS สำหรับ Article model
class Article < ApplicationRecord
  include PgSearch::Model
  
  pg_search_scope :full_text_search,
    against: { title: 'A', summary: 'B', body: 'C' },
    using: {
      tsearch: { dictionary: 'english', prefix: true },
      trigram: { only: [:title], threshold: 0.2 }
    },
    ignoring: :accents
  
  scope :search_and_rank, ->(query) {
    return all if query.blank?
    
    full_text_search(query)
      .with_pg_search_rank
      .where('pg_search_rank > 0.1')
  }
  
  # Custom FTS with highlights
  def self.search_with_highlight(query)
    return none if query.blank?
    
    tsquery = sanitize_sql(["plainto_tsquery('english', ?)", query])
    tsvector = "to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(body, ''))"
    
    select(
      "articles.*",
      "ts_rank(#{tsvector}, #{tsquery}) as search_rank",
      "ts_headline('english', body, #{tsquery}, 'MaxFragments=3,FragmentDelimiter=...,StartSel=<strong>,StopSel=</strong>') as body_highlights"
    )
    .where("#{tsvector} @@ #{tsquery}")
    .order('search_rank DESC')
  end
end
```

### แบบฝึกหัดที่ 6: Pessimistic Locking สำหรับ Tickets

```ruby
# ระบบขายตั๋วที่ป้องกัน overselling
class TicketReservation
  def self.reserve!(event_id:, user:, quantity: 1)
    Event.transaction do
      event = Event.lock.find(event_id)
      
      if event.available_tickets < quantity
        raise InsufficientTicketsError, 
          "เหลือเพียง #{event.available_tickets} ที่นั่ง"
      end
      
      reservation = Reservation.create!(
        event: event,
        user: user,
        quantity: quantity,
        status: 'reserved',
        expires_at: 15.minutes.from_now
      )
      
      event.decrement!(:available_tickets, quantity)
      
      # Schedule cleanup ถ้าไม่ชำระเงิน
      ExpireReservationJob.set(wait_until: reservation.expires_at)
                          .perform_later(reservation.id)
      
      reservation
    end
  rescue ActiveRecord::RecordNotFound
    raise EventNotFoundError, "ไม่พบ event"
  end
end
```

### แบบฝึกหัดที่ 7: STI สำหรับ Payment Methods

```ruby
# migration
class CreatePaymentMethods < ActiveRecord::Migration[7.0]
  def change
    create_table :payment_methods do |t|
      t.string :type, null: false
      t.references :user, null: false, foreign_key: true
      t.string :nickname
      t.boolean :default, default: false
      t.boolean :active, default: true
      
      # Credit card fields
      t.string :card_last4
      t.string :card_brand
      t.integer :card_exp_month
      t.integer :card_exp_year
      t.string :stripe_card_id
      
      # Bank account fields
      t.string :bank_name
      t.string :account_last4
      t.string :account_type
      
      # Wallet fields
      t.string :wallet_type
      t.string :wallet_account
      
      t.timestamps
    end
  end
end

class PaymentMethod < ApplicationRecord
  belongs_to :user
  
  scope :active, -> { where(active: true) }
  scope :default, -> { where(default: true) }
  
  def self.make_default(user, payment_method)
    transaction do
      user.payment_methods.update_all(default: false)
      payment_method.update!(default: true)
    end
  end
end

class CreditCard < PaymentMethod
  validates :card_last4, :card_brand, :stripe_card_id, presence: true
  validates :card_exp_month, inclusion: { in: 1..12 }
  validates :card_exp_year, numericality: { greater_than_or_equal_to: Date.current.year }
  
  def expired?
    Date.new(card_exp_year, card_exp_month).end_of_month < Date.current
  end
  
  def display_name
    "#{card_brand} ****#{card_last4}"
  end
  
  def charge!(amount:, description: nil)
    Stripe::Charge.create(
      amount: (amount * 100).to_i,
      currency: 'thb',
      source: stripe_card_id,
      description: description
    )
  end
end

class BankAccount < PaymentMethod
  validates :bank_name, :account_last4, presence: true
  
  def display_name
    "#{bank_name} ****#{account_last4}"
  end
end

class DigitalWallet < PaymentMethod
  TYPES = %w[promptpay truemoney rabbit_line_pay].freeze
  
  validates :wallet_type, inclusion: { in: TYPES }
  validates :wallet_account, presence: true
  
  def display_name
    "#{wallet_type.humanize}: #{wallet_account}"
  end
end
```

### แบบฝึกหัดที่ 8-25: ชุดแบบฝึกหัดเพิ่มเติม

```ruby
# แบบฝึกหัดที่ 8: Delegated Types สำหรับ Content
class Content < ApplicationRecord
  delegated_type :contentable, types: %w[Article Video Podcast Infographic]
  belongs_to :author, class_name: 'User'
  has_many :views
  has_many :likes
  has_many :comments
  
  scope :published, -> { where(published_at: ..Time.current) }
  scope :trending, -> { order(views_count: :desc).limit(10) }
  
  def view_count
    views.count
  end
  
  def like_count
    likes.count
  end
end

class Article < ApplicationRecord
  include Contentable
  
  validates :body, presence: true
  validates :reading_time, numericality: { greater_than: 0 }, allow_nil: true
  
  before_save :calculate_reading_time
  
  private
  
  def calculate_reading_time
    self.reading_time = (body.split.count / 200.0).ceil  # 200 words/min
  end
end

class Video < ApplicationRecord
  include Contentable
  
  validates :video_url, presence: true
  validates :duration_seconds, numericality: { greater_than: 0 }
  
  def duration_formatted
    seconds = duration_seconds
    "#{seconds / 60}:#{(seconds % 60).to_s.rjust(2, '0')}"
  end
end

# แบบฝึกหัดที่ 9: Complex Scopes
class Order < ApplicationRecord
  scope :revenue_in_range, ->(min, max) {
    where(total: min..max)
  }
  
  scope :with_customer_info, -> {
    joins(:user)
    .select('orders.*, users.name as customer_name, users.email as customer_email')
  }
  
  scope :abandoned, -> {
    where(status: 'cart')
    .where('updated_at < ?', 24.hours.ago)
    .where(converted_at: nil)
  }
  
  scope :high_value, -> {
    where('total > ?', Order.average(:total))
  }
  
  scope :for_dashboard, -> {
    select(:id, :status, :total, :created_at, :user_id)
    .includes(:user)
    .order(created_at: :desc)
  }
end

# แบบฝึกหัดที่ 10: Efficient Counting
class Analytics
  def self.user_stats
    # Efficient: ใช้ GROUP BY แทน multiple queries
    User
      .select(
        "COUNT(*) as total",
        "COUNT(*) FILTER (WHERE active = true) as active",
        "COUNT(*) FILTER (WHERE created_at > NOW() - INTERVAL '30 days') as new_this_month",
        "COUNT(*) FILTER (WHERE last_sign_in_at > NOW() - INTERVAL '7 days') as active_this_week"
      )
      .first
  end
  
  def self.order_stats_by_status
    Order
      .group(:status)
      .select(:status, "COUNT(*) as count", "SUM(total) as revenue")
      .map { |r| [r.status, { count: r.count, revenue: r.revenue }] }
      .to_h
  end
end

# แบบฝึกหัดที่ 11: Upsert
class ProductSync
  def self.sync_from_api(products_data)
    # Bulk upsert (Rails 6+)
    Product.upsert_all(
      products_data.map { |p|
        {
          external_id: p[:id],
          name: p[:name],
          price: p[:price],
          stock: p[:stock],
          updated_at: Time.current
        }
      },
      unique_by: :external_id,
      update_only: [:name, :price, :stock, :updated_at]
    )
  end
end

# แบบฝึกหัดที่ 12: Virtual Attributes
class User < ApplicationRecord
  # Virtual attribute: ไม่ save ใน DB
  attr_accessor :skip_confirmation_email
  
  after_create :send_confirmation, unless: :skip_confirmation_email
  
  # Virtual attribute สำหรับ form
  attr_writer :full_name
  
  before_validation :split_full_name, if: :full_name_changed?
  
  def full_name
    [first_name, last_name].join(' ').strip
  end
  
  private
  
  def full_name_changed?
    @full_name.present?
  end
  
  def split_full_name
    parts = @full_name.split(' ', 2)
    self.first_name = parts[0]
    self.last_name = parts[1]
  end
  
  def send_confirmation
    UserMailer.confirmation(self).deliver_later
  end
end
```

### แบบฝึกหัดที่ 13: Advanced Callbacks

```ruby
class Order < ApplicationRecord
  # Conditional callbacks
  after_create :notify_admin, if: :high_value?
  after_update :send_status_email, if: :saved_change_to_status?
  before_destroy :check_can_delete
  
  # Callback with method
  after_commit :index_in_elasticsearch, on: [:create, :update]
  after_commit :remove_from_elasticsearch, on: :destroy
  
  private
  
  def high_value?
    total >= 10_000
  end
  
  def notify_admin
    AdminMailer.high_value_order(self).deliver_later
  end
  
  def send_status_email
    previous_status = status_before_last_save
    OrderMailer.status_changed(self, previous_status, status).deliver_later
  end
  
  def check_can_delete
    if completed?
      errors.add(:base, 'ไม่สามารถลบ order ที่เสร็จสิ้นแล้ว')
      throw(:abort)
    end
  end
  
  def index_in_elasticsearch
    ElasticsearchIndexJob.perform_later('Order', id)
  end
  
  def remove_from_elasticsearch
    ElasticsearchRemoveJob.perform_later('Order', id)
  end
end
```

### สรุป Advanced Active Record

```
Key Points:
1. ใช้ find_by_sql และ connection.execute สำหรับ queries ที่ซับซ้อนมาก
2. Window functions เหมาะสำหรับ analytics และ ranking
3. CTEs ช่วยให้ queries ซับซ้อนอ่านง่ายขึ้น
4. Optimistic locking: ดีสำหรับ low-contention scenarios
5. Pessimistic locking: ดีสำหรับ high-contention scenarios เช่น inventory
6. STI: ใช้เมื่อ subtypes มี structure คล้ายกันมาก
7. Delegated Types: ใช้เมื่อ subtypes มี structure แตกต่างกัน
8. ใช้ includes/preload/eager_load อย่างถูกต้องเพื่อป้องกัน N+1
```

---

**จบ Part 67: Advanced Active Record**

*ในส่วนถัดไป Part 68 เราจะเรียนรู้เกี่ยวกับ API Authentication*
