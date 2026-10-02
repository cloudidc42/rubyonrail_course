# ตอนที่ 37: Migrations (Steps 811-830)

## บทนำ

Migrations เป็นวิธีที่ Rails ใช้ในการเปลี่ยนแปลงโครงสร้างฐานข้อมูล (database schema) อย่างมีระบบ แต่ละ migration เป็นไฟล์ Ruby ที่บอกว่าจะทำอะไรกับ database และสามารถ undo ได้ด้วยการ rollback

---

## Step 811: Migration คืออะไร?

```ruby
# Migration คือไฟล์ที่:
# 1. มี timestamp ใน filename เพื่อ ordering
# 2. มี change method (หรือ up/down)
# 3. สามารถ migrate และ rollback ได้

# ตัวอย่าง migration
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :name
      t.string :email
      t.timestamps
    end
  end
end
```

### ทำไมต้องใช้ Migration?

1. **Version control for database** - track การเปลี่ยนแปลง schema
2. **Team collaboration** - ทุกคนใน team ใช้ schema เดียวกัน
3. **Deploy safely** - deploy schema changes พร้อมกับ code
4. **Rollback** - ยกเลิกการเปลี่ยนแปลงได้

### ชื่อไฟล์ Migration

```
db/migrate/
├── 20240101120000_create_users.rb
├── 20240102130000_add_email_to_users.rb
├── 20240103140000_create_posts.rb
└── 20240104150000_add_index_to_posts.rb
```

Format: `YYYYMMDDHHMMSS_description.rb`

---

## Step 812: rails generate migration

```bash
# สร้าง empty migration
rails g migration CreateUsers

# สร้างพร้อม columns
rails g migration CreatePosts title:string content:text published:boolean

# Add column
rails g migration AddEmailToUsers email:string
# สร้าง: add_column :users, :email, :string

# Add column with index
rails g migration AddEmailToUsers email:string:index
# สร้าง: add_column + add_index

# Add column unique index
rails g migration AddEmailToUsers email:string:uniq
# สร้าง: add_column + add_index unique: true

# Remove column
rails g migration RemoveEmailFromUsers email:string
# สร้าง: remove_column :users, :email, :string

# Add reference (foreign key)
rails g migration AddUserToPosts user:references
# สร้าง: add_reference :posts, :user, foreign_key: true

# Rename table
rails g migration RenameUsersToMembers
# สร้าง empty migration (ต้องเขียนเอง)

# Redo migration (add column + index ในครั้งเดียว)
rails g migration AddSlugToPostsAndIndex slug:string:uniq
```

---

## Step 813: Column Types

```ruby
class CreateExamples < ActiveRecord::Migration[7.0]
  def change
    create_table :examples do |t|
      # String types
      t.string  :name               # VARCHAR(255)
      t.string  :code, limit: 10    # VARCHAR(10)
      t.text    :description        # TEXT (unlimited)
      t.text    :notes, limit: 65535  # MEDIUMTEXT in MySQL

      # Numeric types
      t.integer  :count             # INT
      t.integer  :big_number, limit: 8  # BIGINT
      t.bigint   :external_id       # BIGINT (alias)
      t.float    :price             # FLOAT
      t.decimal  :amount            # DECIMAL
      t.decimal  :precise_amount, precision: 10, scale: 2  # DECIMAL(10,2)
      
      # Boolean
      t.boolean  :active, default: true
      t.boolean  :verified, default: false, null: false
      
      # Date/Time
      t.date      :birthday         # DATE
      t.time      :start_time       # TIME
      t.datetime  :published_at     # DATETIME
      t.timestamp :deleted_at       # TIMESTAMP (alias for datetime)
      t.timestamps                  # สร้าง created_at AND updated_at
      
      # Binary
      t.binary   :data              # BLOB
      t.binary   :image, limit: 2.megabytes
      
      # UUID (PostgreSQL)
      t.uuid     :external_id, default: -> { "gen_random_uuid()" }
      
      # JSON (PostgreSQL, MySQL 5.7+)
      t.json     :metadata
      t.jsonb    :settings          # PostgreSQL only (faster)
      
      # Array (PostgreSQL only)
      t.integer  :scores, array: true, default: []
      t.string   :tags, array: true
      
      # Other types
      t.inet     :ip_address        # PostgreSQL: IP address
      t.cidr     :subnet            # PostgreSQL: network
      t.point    :coordinates       # PostgreSQL: geometric
      t.hstore   :preferences       # PostgreSQL: key-value
    end
  end
end
```

---

## Step 814: create_table

```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    # Basic
    create_table :users do |t|
      t.string :name, null: false
      t.string :email, null: false
      t.string :password_digest
      t.boolean :active, default: true
      t.timestamps
    end
    
    # ด้วย options
    create_table :users, 
      id: :bigint,          # primary key type (default)
      primary_key: :id,     # custom primary key name
      comment: "ตาราง users",
      force: true do |t|    # drop table ก่อนถ้ามีอยู่แล้ว
      t.string :name
    end
    
    # UUID primary key
    create_table :posts, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.string :title
      t.timestamps
    end
    
    # Composite primary key
    create_table :order_items, primary_key: [:order_id, :product_id] do |t|
      t.references :order, foreign_key: true
      t.references :product, foreign_key: true
      t.integer :quantity
    end
    
    # ไม่มี primary key
    create_table :sessions, id: false do |t|
      t.string :session_id, null: false, primary_key: true
      t.text :data
      t.timestamps
    end
  end
end
```

---

## Step 815: add_column, remove_column

### add_column

```ruby
class AddEmailToUsers < ActiveRecord::Migration[7.0]
  def change
    # Basic
    add_column :users, :email, :string
    
    # ด้วย options
    add_column :users, :email, :string, null: false, default: ""
    add_column :users, :age, :integer, default: 0
    add_column :users, :verified_at, :datetime
    add_column :users, :metadata, :jsonb, default: {}
    
    # ระบุตำแหน่ง (MySQL only)
    add_column :users, :phone, :string, after: :email
    add_column :users, :prefix, :string, first: true
    
    # ด้วย comment
    add_column :users, :score, :integer, 
      default: 0, 
      comment: "คะแนนสะสมของผู้ใช้"
  end
end
```

### remove_column

```ruby
class RemoveEmailFromUsers < ActiveRecord::Migration[7.0]
  # ต้องระบุ type เพื่อให้ rollback ได้
  def change
    remove_column :users, :email, :string
    remove_column :users, :age, :integer, default: 0
  end
  
  # หรือใช้ up/down แยกกัน
  def up
    remove_column :users, :email
  end
  
  def down
    add_column :users, :email, :string
  end
end
```

---

## Step 816: rename_column, rename_table

### rename_column

```ruby
class RenameUsernameToName < ActiveRecord::Migration[7.0]
  def change
    rename_column :users, :username, :name
    rename_column :posts, :body, :content
  end
end
```

### rename_table

```ruby
class RenameUsersToMembers < ActiveRecord::Migration[7.0]
  def change
    rename_table :users, :members
  end
end
```

---

## Step 817: change_column

```ruby
class ChangeAgeTypeInUsers < ActiveRecord::Migration[7.0]
  # change_column ไม่ reversible โดย default
  # ต้องใช้ up/down
  
  def up
    change_column :users, :age, :float
    change_column :users, :name, :string, limit: 100
    change_column :users, :bio, :text, null: false, default: ""
  end
  
  def down
    change_column :users, :age, :integer
    change_column :users, :name, :string
    change_column :users, :bio, :text, null: true, default: nil
  end
end

# เปลี่ยนเฉพาะ default value
class ChangeDefaultActiveInUsers < ActiveRecord::Migration[7.0]
  def change
    change_column_default :users, :active, from: nil, to: true
    change_column_default :posts, :status, from: nil, to: "draft"
  end
end

# เปลี่ยนเฉพาะ null constraint
class ChangeNullOnEmail < ActiveRecord::Migration[7.0]
  def change
    change_column_null :users, :email, false  # NOT NULL
    change_column_null :users, :bio, true     # Allow NULL
  end
end
```

---

## Step 818: add_index, remove_index

### add_index

```ruby
class AddIndexesToUsers < ActiveRecord::Migration[7.0]
  def change
    # Simple index
    add_index :users, :email
    
    # Unique index
    add_index :users, :email, unique: true
    
    # Composite index
    add_index :users, [:last_name, :first_name]
    
    # Index ด้วยชื่อเฉพาะ
    add_index :users, :email, name: "index_users_on_email_unique", unique: true
    
    # Partial index (PostgreSQL)
    add_index :posts, :user_id, where: "published = true"
    
    # Index บน expression (PostgreSQL)
    add_index :users, "LOWER(email)", name: "index_users_on_lower_email", unique: true
    
    # Concurrent index (PostgreSQL - ไม่ lock table)
    add_index :posts, :user_id, algorithm: :concurrently
    
    # String prefix index (MySQL)
    add_index :posts, :content, length: 100  # INDEX ใน 100 chars แรก
  end
end
```

### remove_index

```ruby
class RemoveIndexFromUsers < ActiveRecord::Migration[7.0]
  def change
    # ลบด้วย column name
    remove_index :users, :email
    
    # ลบด้วย index name
    remove_index :users, name: "index_users_on_email_unique"
    
    # ลบด้วย column array
    remove_index :users, column: [:last_name, :first_name]
    
    # ลบ concurrent (PostgreSQL)
    remove_index :posts, :user_id, algorithm: :concurrently
  end
end
```

---

## Step 819: add_foreign_key, remove_foreign_key

### add_foreign_key

```ruby
class AddForeignKeysToPosts < ActiveRecord::Migration[7.0]
  def change
    # Basic foreign key
    add_foreign_key :posts, :users
    # posts.user_id REFERENCES users(id)
    
    # ด้วย custom column
    add_foreign_key :posts, :users, column: :author_id
    # posts.author_id REFERENCES users(id)
    
    # ด้วย ON DELETE
    add_foreign_key :posts, :users, on_delete: :cascade
    # ลบ posts เมื่อ user ถูกลบ
    
    add_foreign_key :posts, :users, on_delete: :nullify
    # set user_id = NULL เมื่อ user ถูกลบ
    
    add_foreign_key :posts, :users, on_delete: :restrict
    # ไม่ให้ลบ user ถ้ายังมี posts
    
    # ด้วย ON UPDATE
    add_foreign_key :posts, :users, on_update: :cascade
    
    # ด้วยชื่อ constraint
    add_foreign_key :posts, :users, name: "fk_posts_users"
    
    # Deferrable (PostgreSQL)
    add_foreign_key :posts, :users, deferrable: :deferred
  end
end
```

### remove_foreign_key

```ruby
class RemoveForeignKeyFromPosts < ActiveRecord::Migration[7.0]
  def change
    remove_foreign_key :posts, :users
    remove_foreign_key :posts, column: :author_id
    remove_foreign_key :posts, name: "fk_posts_users"
  end
end
```

---

## Step 820: add_reference

```ruby
class AddUserToComments < ActiveRecord::Migration[7.0]
  def change
    # เพิ่ม user_id column + index + foreign key
    add_reference :comments, :user, null: false, foreign_key: true
    
    # ด้วย polymorphic
    add_reference :comments, :commentable, polymorphic: true, null: false
    # สร้าง commentable_id (integer) + commentable_type (string)
    
    # ด้วย custom index
    add_reference :posts, :user, foreign_key: true, index: { name: "idx_posts_user_id" }
    
    # ไม่สร้าง index
    add_reference :logs, :user, index: false
    
    # UUID reference
    add_reference :posts, :user, type: :uuid, foreign_key: true
  end
end
```

---

## Step 821: Running Migrations

```bash
# รัน migrations ที่ยังไม่ได้รัน
rails db:migrate

# รัน migrations ถึง version ที่กำหนด
rails db:migrate VERSION=20240101120000

# Rollback migration ล่าสุด
rails db:rollback

# Rollback หลาย steps
rails db:rollback STEP=3

# Rollback ถึง version ที่กำหนด
rails db:migrate:down VERSION=20240101120000

# Migrate ขึ้นไปถึง version
rails db:migrate:up VERSION=20240101120000

# Rollback แล้ว migrate ใหม่ (redo)
rails db:migrate:redo

# Redo หลาย steps
rails db:migrate:redo STEP=2

# รัน migration เฉพาะ environment
RAILS_ENV=test rails db:migrate
RAILS_ENV=production rails db:migrate

# สร้าง database
rails db:create

# ลบ database
rails db:drop

# Drop + Create + Migrate + Seed
rails db:reset

# Load schema.rb ลงใน database (เร็วกว่า migrate ทั้งหมด)
rails db:schema:load

# Dump schema ปัจจุบัน
rails db:schema:dump

# Setup database ใหม่
rails db:setup  # create + schema:load + seed

# Prepare database (migrate หรือ schema:load อัตโนมัติ)
rails db:prepare
```

---

## Step 822: Migration Status

```bash
# ดู status ของ migrations ทั้งหมด
rails db:migrate:status

# Output:
# database: myapp_development
# 
#  Status   Migration ID    Migration Name
# --------------------------------------------------
#    up     20240101120000  Create users
#    up     20240102130000  Create posts
#    down   20240103140000  Add tags to posts
#    down   20240104150000  Add indexes

# ดู pending migrations
rails db:migrate:status | grep "down"
```

---

## Step 823: schema.rb vs structure.sql

### schema.rb

```ruby
# db/schema.rb
# Auto-generated ห้ามแก้ไขโดยตรง

ActiveRecord::Schema[7.0].define(version: 2024_01_04_150000) do
  # PostgreSQL extensions
  enable_extension "plpgsql"
  enable_extension "pg_trgm"
  
  create_table "users", force: :cascade do |t|
    t.string "name", null: false
    t.string "email", null: false
    t.boolean "active", default: true
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["email"], name: "index_users_on_email", unique: true
  end
  
  create_table "posts", force: :cascade do |t|
    t.string "title"
    t.text "content"
    t.bigint "user_id", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["user_id"], name: "index_posts_on_user_id"
  end
  
  add_foreign_key "posts", "users"
end
```

### ใช้ structure.sql แทน schema.rb

```ruby
# config/application.rb
config.active_record.schema_format = :sql
# สร้าง db/structure.sql แทน db/schema.rb
```

```sql
-- db/structure.sql (PostgreSQL specific SQL)
SET statement_timeout = 0;
SET lock_timeout = 0;

CREATE EXTENSION IF NOT EXISTS plpgsql;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE users (
  id bigserial PRIMARY KEY,
  name character varying NOT NULL,
  email character varying NOT NULL UNIQUE,
  active boolean DEFAULT true,
  created_at timestamp(6) NOT NULL,
  updated_at timestamp(6) NOT NULL
);

-- Triggers, functions, etc.
CREATE FUNCTION update_updated_at() ...
```

**เมื่อไหร่ใช้อะไร:**
- `schema.rb` - ส่วนใหญ่ใช้นี้ portable, อ่านง่าย
- `structure.sql` - ถ้าใช้ database-specific features (triggers, procedures, custom types)

---

## Step 824: Data Migrations

Data migrations คือ migration ที่เปลี่ยนแปลงข้อมูลในฐานข้อมูล ไม่ใช่แค่ schema

```ruby
class PopulateUserSlugs < ActiveRecord::Migration[7.0]
  def up
    # ดึงทีละ batch เพื่อประหยัด memory
    User.find_each(batch_size: 1000) do |user|
      user.update_column(:slug, user.name.parameterize)
    end
  end
  
  def down
    User.update_all(slug: nil)
  end
end

# Migration ที่ backfill ข้อมูลจาก JSON column
class ExtractEmailFromMetadata < ActiveRecord::Migration[7.0]
  def up
    add_column :users, :email, :string
    
    User.find_each do |user|
      metadata = JSON.parse(user.metadata || "{}")
      user.update_column(:email, metadata["email"])
    end
    
    # เพิ่ม constraint หลังจาก backfill
    change_column_null :users, :email, false
    add_index :users, :email, unique: true
  end
  
  def down
    remove_index :users, :email
    remove_column :users, :email
  end
end

# ข้อควรระวัง:
# 1. อย่าใช้ model class โดยตรงใน migration (validation อาจเปลี่ยน)
# 2. ใช้ execute หรือ update_column แทน
# 3. ใช้ find_each สำหรับข้อมูลเยอะ
```

### Safe Migration Pattern

```ruby
class SafeDataMigration < ActiveRecord::Migration[7.0]
  # suppress_messages ซ่อน output จาก say
  def up
    say_with_time "Backfilling user slugs" do
      User.where(slug: nil).find_each(batch_size: 500) do |user|
        slug = user.name.to_s.parameterize
        user.update_column(:slug, slug)
      end
    end
  end
  
  def down
    # ไม่จำเป็นต้อง down สำหรับ data migration บางอย่าง
    raise ActiveRecord::IrreversibleMigration
  end
end
```

---

## Step 825: db:seed

```ruby
# db/seeds.rb
# ข้อมูล initial สำหรับ development

# ล้างข้อมูลเก่าก่อน (สำหรับ development)
if Rails.env.development?
  Post.destroy_all
  User.destroy_all
end

# สร้าง admin user
admin = User.find_or_create_by(email: "admin@example.com") do |u|
  u.name = "Admin User"
  u.password = "password123"
  u.role = "admin"
end

puts "Created admin: #{admin.email}"

# สร้าง users ด้วย Faker
require "faker"

10.times do |i|
  User.find_or_create_by(email: Faker::Internet.unique.email) do |u|
    u.name = Faker::Name.name
    u.bio = Faker::Lorem.paragraph
  end
end

puts "Created #{User.count} users"

# สร้าง categories
categories = ["เทคโนโลยี", "ท่องเที่ยว", "อาหาร", "กีฬา", "บันเทิง"]
categories.each do |name|
  Category.find_or_create_by(name: name)
end

# สร้าง posts
User.all.each do |user|
  5.times do
    Post.create!(
      title: Faker::Lorem.sentence,
      content: Faker::Lorem.paragraphs(number: 3).join("\n\n"),
      status: ["draft", "published"].sample,
      user: user,
      category: Category.all.sample
    )
  end
end

puts "Created #{Post.count} posts"
```

```bash
# รัน seeds
rails db:seed

# Reset + seed
rails db:reset

# รัน specific seed file
rails runner db/seeds/users.rb
```

### EnvironmentSeed

```ruby
# db/seeds.rb
puts "Seeding #{Rails.env}..."

# Common seeds
load(Rails.root.join("db", "seeds", "categories.rb"))

# Environment-specific
seed_file = Rails.root.join("db", "seeds", "#{Rails.env}.rb")
load(seed_file) if File.exist?(seed_file)

puts "Done!"
```

```ruby
# db/seeds/development.rb
require "faker"
Faker::Config.locale = :en

50.times do
  User.create!(
    name: Faker::Name.name,
    email: Faker::Internet.unique.email,
    password: "password"
  )
end

100.times do
  Post.create!(
    title: Faker::Lorem.sentence,
    content: Faker::Lorem.paragraphs(number: 5).join("\n\n"),
    user: User.all.sample,
    status: "published"
  )
end
```

---

## ตัวอย่าง Migration ที่สมบูรณ์ (E-commerce)

```ruby
# db/migrate/20240101000001_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :email, null: false
      t.string :password_digest, null: false
      t.string :phone
      t.boolean :active, default: true, null: false
      t.boolean :email_verified, default: false, null: false
      t.integer :role, default: 0, null: false
      t.datetime :last_sign_in_at
      t.string :remember_digest
      t.string :reset_password_token
      t.datetime :reset_password_sent_at
      t.timestamps
    end
    
    add_index :users, :email, unique: true
    add_index :users, :phone, unique: true, where: "phone IS NOT NULL"
    add_index :users, :role
    add_index :users, :reset_password_token, unique: true, 
              where: "reset_password_token IS NOT NULL"
  end
end

# db/migrate/20240101000002_create_categories.rb
class CreateCategories < ActiveRecord::Migration[7.0]
  def change
    create_table :categories do |t|
      t.string :name, null: false
      t.string :slug, null: false
      t.text :description
      t.integer :parent_id
      t.integer :position, default: 0
      t.boolean :active, default: true
      t.timestamps
    end
    
    add_index :categories, :slug, unique: true
    add_index :categories, :parent_id
    add_foreign_key :categories, :categories, column: :parent_id
  end
end

# db/migrate/20240101000003_create_products.rb
class CreateProducts < ActiveRecord::Migration[7.0]
  def change
    create_table :products do |t|
      t.string :name, null: false
      t.string :sku, null: false
      t.text :description
      t.decimal :price, precision: 10, scale: 2, null: false, default: 0
      t.decimal :sale_price, precision: 10, scale: 2
      t.integer :stock, default: 0, null: false
      t.integer :status, default: 0, null: false
      t.boolean :featured, default: false
      t.bigint :category_id
      t.jsonb :metadata, default: {}
      t.timestamps
    end
    
    add_index :products, :sku, unique: true
    add_index :products, :category_id
    add_index :products, :status
    add_index :products, :featured
    add_index :products, :price
    add_foreign_key :products, :categories
  end
end

# db/migrate/20240101000004_create_orders.rb
class CreateOrders < ActiveRecord::Migration[7.0]
  def change
    create_table :orders do |t|
      t.bigint :user_id, null: false
      t.string :number, null: false
      t.integer :status, default: 0, null: false
      t.decimal :subtotal, precision: 10, scale: 2, default: 0
      t.decimal :tax, precision: 10, scale: 2, default: 0
      t.decimal :shipping, precision: 10, scale: 2, default: 0
      t.decimal :discount, precision: 10, scale: 2, default: 0
      t.decimal :total, precision: 10, scale: 2, default: 0
      t.string :payment_method
      t.string :payment_status
      t.text :notes
      t.jsonb :shipping_address, default: {}
      t.datetime :paid_at
      t.datetime :shipped_at
      t.datetime :delivered_at
      t.timestamps
    end
    
    add_index :orders, :user_id
    add_index :orders, :number, unique: true
    add_index :orders, :status
    add_foreign_key :orders, :users
  end
end

# db/migrate/20240101000005_create_order_items.rb
class CreateOrderItems < ActiveRecord::Migration[7.0]
  def change
    create_table :order_items do |t|
      t.bigint :order_id, null: false
      t.bigint :product_id, null: false
      t.integer :quantity, null: false, default: 1
      t.decimal :unit_price, precision: 10, scale: 2, null: false
      t.decimal :total_price, precision: 10, scale: 2, null: false
      t.string :product_name  # snapshot ณ เวลาที่สั่ง
      t.timestamps
    end
    
    add_index :order_items, :order_id
    add_index :order_items, :product_id
    add_foreign_key :order_items, :orders, on_delete: :cascade
    add_foreign_key :order_items, :products
  end
end
```

---

## แบบฝึกหัด (Steps 826-830)

### แบบฝึกหัดที่ 1
สร้าง migration ที่เพิ่ม column `phone` ใน users table

**เฉลย:**
```bash
rails g migration AddPhoneToUsers phone:string
```

```ruby
class AddPhoneToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :phone, :string
  end
end
```

### แบบฝึกหัดที่ 2
สร้าง migration ที่เพิ่ม unique index บน email ใน users

**เฉลย:**
```ruby
class AddUniqueIndexToUsersEmail < ActiveRecord::Migration[7.0]
  def change
    add_index :users, :email, unique: true
  end
end
```

### แบบฝึกหัดที่ 3
สร้าง migration สำหรับ posts table ที่มี title, content, status, และ foreign key ไป users

**เฉลย:**
```ruby
class CreatePosts < ActiveRecord::Migration[7.0]
  def change
    create_table :posts do |t|
      t.string :title, null: false
      t.text :content
      t.integer :status, default: 0, null: false
      t.bigint :user_id, null: false
      t.timestamps
    end
    
    add_index :posts, :user_id
    add_index :posts, :status
    add_foreign_key :posts, :users
  end
end
```

### แบบฝึกหัดที่ 4
rollback migration ล่าสุด

**เฉลย:**
```bash
rails db:rollback
```

### แบบฝึกหัดที่ 5
ดู status ของ migrations ทั้งหมด

**เฉลย:**
```bash
rails db:migrate:status
```

### แบบฝึกหัดที่ 6
เพิ่ม column `deleted_at` สำหรับ soft delete ใน posts

**เฉลย:**
```ruby
class AddDeletedAtToPosts < ActiveRecord::Migration[7.0]
  def change
    add_column :posts, :deleted_at, :datetime
    add_index :posts, :deleted_at
  end
end
```

### แบบฝึกหัดที่ 7
เปลี่ยน column type ของ age จาก string เป็น integer

**เฉลย:**
```ruby
class ChangeAgeTypeInUsers < ActiveRecord::Migration[7.0]
  def up
    change_column :users, :age, :integer, using: "age::integer"
  end
  
  def down
    change_column :users, :age, :string
  end
end
```

### แบบฝึกหัดที่ 8
เพิ่ม composite index บน (user_id, status) ใน posts

**เฉลย:**
```ruby
class AddCompositeIndexToPosts < ActiveRecord::Migration[7.0]
  def change
    add_index :posts, [:user_id, :status]
  end
end
```

### แบบฝึกหัดที่ 9
สร้าง data migration ที่ set default value สำหรับ column ที่มีอยู่

**เฉลย:**
```ruby
class BackfillUsersRole < ActiveRecord::Migration[7.0]
  def up
    User.where(role: nil).update_all(role: 0)  # default: member
    change_column_null :users, :role, false
    change_column_default :users, :role, from: nil, to: 0
  end
  
  def down
    change_column_default :users, :role, from: 0, to: nil
    change_column_null :users, :role, true
  end
end
```

### แบบฝึกหัดที่ 10
สร้าง seed file ที่สร้าง admin user และ categories

**เฉลย:**
```ruby
# db/seeds.rb
admin = User.find_or_create_by(email: "admin@example.com") do |u|
  u.name = "Admin"
  u.password = "Admin1234!"
  u.role = "admin"
end
puts "Admin created: #{admin.email}"

["เทคโนโลยี", "ท่องเที่ยว", "อาหาร", "สุขภาพ"].each do |name|
  Category.find_or_create_by(name: name)
end
puts "Categories created: #{Category.count}"
```

### แบบฝึกหัดที่ 11
rename column old_name เป็น name ใน users

**เฉลย:**
```ruby
class RenameOldNameToNameInUsers < ActiveRecord::Migration[7.0]
  def change
    rename_column :users, :old_name, :name
  end
end
```

### แบบฝึกหัดที่ 12
เพิ่ม foreign key แบบ ON DELETE CASCADE

**เฉลย:**
```ruby
class AddForeignKeyWithCascade < ActiveRecord::Migration[7.0]
  def change
    add_foreign_key :comments, :posts, on_delete: :cascade
  end
end
```

### แบบฝึกหัดที่ 13
สร้าง migration สำหรับ polymorphic association

**เฉลย:**
```ruby
class CreateComments < ActiveRecord::Migration[7.0]
  def change
    create_table :comments do |t|
      t.text :content, null: false
      t.bigint :user_id, null: false
      t.references :commentable, polymorphic: true, null: false
      t.timestamps
    end
    
    add_index :comments, :user_id
    add_foreign_key :comments, :users
  end
end
```

### แบบฝึกหัดที่ 14
ลบ column bio จาก users

**เฉลย:**
```ruby
class RemoveBioFromUsers < ActiveRecord::Migration[7.0]
  def change
    remove_column :users, :bio, :text  # ต้องระบุ type สำหรับ rollback
  end
end
```

### แบบฝึกหัดที่ 15
สร้าง migration ที่ใช้ up/down แทน change

**เฉลย:**
```ruby
class CreateSpecialTable < ActiveRecord::Migration[7.0]
  def up
    create_table :special_items do |t|
      t.string :type_code, null: false
      t.jsonb :attributes, default: {}
      t.timestamps
    end
    
    execute "CREATE INDEX idx_special_items_gin ON special_items USING gin(attributes)"
  end
  
  def down
    execute "DROP INDEX IF EXISTS idx_special_items_gin"
    drop_table :special_items
  end
end
```

### แบบฝึกหัดที่ 16
เปลี่ยน schema format เป็น sql

**เฉลย:**
```ruby
# config/application.rb
config.active_record.schema_format = :sql
```

```bash
rails db:schema:dump
# สร้าง db/structure.sql
```

### แบบฝึกหัดที่ 17
สร้าง migration ที่มี conditional SQL execution

**เฉลย:**
```ruby
class AddFullTextSearchToPosts < ActiveRecord::Migration[7.0]
  def up
    if ActiveRecord::Base.connection.adapter_name == "PostgreSQL"
      execute <<-SQL
        ALTER TABLE posts ADD COLUMN search_vector tsvector;
        CREATE INDEX posts_search_vector_idx ON posts USING gin(search_vector);
      SQL
    else
      add_column :posts, :search_content, :text
      add_index :posts, :search_content
    end
  end
  
  def down
    if ActiveRecord::Base.connection.adapter_name == "PostgreSQL"
      execute "DROP INDEX IF EXISTS posts_search_vector_idx"
      remove_column :posts, :search_vector
    else
      remove_index :posts, :search_content
      remove_column :posts, :search_content
    end
  end
end
```

### แบบฝึกหัดที่ 18
เพิ่ม timestamps ใน table ที่ไม่มี

**เฉลย:**
```ruby
class AddTimestampsToCategories < ActiveRecord::Migration[7.0]
  def change
    add_column :categories, :created_at, :datetime, null: false, default: -> { "CURRENT_TIMESTAMP" }
    add_column :categories, :updated_at, :datetime, null: false, default: -> { "CURRENT_TIMESTAMP" }
  end
end
```

### แบบฝึกหัดที่ 19
สร้าง migration ที่ backfill data หลังเพิ่ม column

**เฉลย:**
```ruby
class AddSlugToPostsAndBackfill < ActiveRecord::Migration[7.0]
  def up
    add_column :posts, :slug, :string
    
    # Backfill existing records
    Post.find_each(batch_size: 500) do |post|
      post.update_column(:slug, post.title.parameterize)
    end
    
    # Add constraint after backfill
    change_column_null :posts, :slug, false
    add_index :posts, :slug, unique: true
  end
  
  def down
    remove_index :posts, :slug
    remove_column :posts, :slug
  end
end
```

### แบบฝึกหัดที่ 20
reset migrations และ seed ใหม่ใน development

**เฉลย:**
```bash
# Drop + Create + Migrate + Seed
rails db:reset

# หรือทีละขั้น
rails db:drop
rails db:create
rails db:migrate
rails db:seed
```

---

## สรุป

ใน Migrations เราได้เรียนรู้:

1. **Migration คืออะไร** - version control สำหรับ database schema
2. **rails g migration** - สร้าง migration ด้วย generator
3. **Column types** - string, text, integer, decimal, boolean, datetime, json
4. **create_table** - สร้างตาราง
5. **add_column/remove_column** - เพิ่ม/ลบ column
6. **rename_column/rename_table** - เปลี่ยนชื่อ
7. **change_column** - เปลี่ยน type หรือ options
8. **add_index/remove_index** - จัดการ indexes
9. **add_foreign_key** - ความสัมพันธ์ระหว่างตาราง
10. **Running migrations** - migrate, rollback, redo
11. **Migration status** - ดูสถานะ
12. **schema.rb vs structure.sql** - สองแบบของ schema file
13. **Data migrations** - เปลี่ยนแปลงข้อมูล
14. **db:seed** - สร้างข้อมูลเริ่มต้น
