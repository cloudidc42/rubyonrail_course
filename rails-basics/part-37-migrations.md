# ตอนที่ 37: Migrations (Steps 811-830)

## บทนำ

Migrations คือวิธีที่ Rails ใช้จัดการการเปลี่ยนแปลงโครงสร้างฐานข้อมูลอย่างเป็นระบบ แทนที่จะเขียน SQL โดยตรง เราใช้ Ruby DSL ที่อ่านง่าย และ Rails จะแปลงเป็น SQL ที่เหมาะสมกับแต่ละ database

Migrations มีประโยชน์เพราะ:
1. **Version control สำหรับ database** - ทุกการเปลี่ยนแปลงมี record
2. **Team collaboration** - ทุกคนใน team ใช้ database โครงสร้างเดียวกัน
3. **Environment consistency** - development, test, production มีโครงสร้างเหมือนกัน
4. **Reversible** - ส่วนใหญ่สามารถ rollback ได้

---

## ขั้นตอนที่ 811: Migrations คืออะไรและทำไมต้องใช้

### ปัญหาที่ Migration แก้ไข

```
ก่อนมี Migrations:
- นักพัฒนา A เพิ่ม column email ในฐานข้อมูล local
- นักพัฒนา B ไม่รู้ว่าต้องเพิ่ม column ด้วย
- เกิด error เมื่อ B pull code ของ A มาใช้
- Production server ไม่มี column นั้น → app พัง

หลังมี Migrations:
- A สร้าง migration file เพิ่มเข้า git
- B pull มาแล้วรัน db:migrate
- Production รัน db:migrate ก่อน deploy
- ทุกคนมีโครงสร้างฐานข้อมูลเหมือนกัน
```

### Migration File Structure

```ruby
# db/migrate/20240115103000_create_posts.rb
# ชื่อไฟล์: [timestamp]_[description].rb

class CreatePosts < ActiveRecord::Migration[7.1]
  def change
    create_table :posts do |t|
      t.string :title, null: false
      t.text :body
      t.boolean :published, default: false
      t.references :user, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

---

## ขั้นตอนที่ 812: rails generate migration

### Command พื้นฐาน

```bash
# สร้าง migration เปล่า
rails generate migration AddEmailToUsers

# สร้าง migration พร้อม columns
rails generate migration AddEmailToUsers email:string

# สร้าง migration ลบ column
rails generate migration RemoveEmailFromUsers email:string

# สร้าง migration เพิ่มหลาย columns
rails generate migration AddProfileToUsers \
  bio:text \
  website:string \
  location:string \
  birth_date:date

# สร้าง table ใหม่
rails generate migration CreateComments \
  body:text \
  approved:boolean \
  post:references \
  user:references

# สร้าง join table
rails generate migration CreateJoinTablePostsTagsPostsTags posts tags
# สร้าง: create_join_table :posts, :tags
```

### Naming Conventions ที่ทำให้ Migration Auto-generate

```bash
# Add[Column]To[Table]
rails g migration AddSlugToPosts slug:string:uniq
# → add_column :posts, :slug, :string + add_index :posts, :slug, unique: true

# Remove[Column]From[Table]
rails g migration RemoveSlugFromPosts slug:string
# → remove_column :posts, :slug, :string

# Create[Table]
rails g migration CreateTags name:string
# → create_table :tags

# Add[Reference]To[Table]
rails g migration AddCategoryToArticles category:references
# → add_reference :articles, :category, foreign_key: true
```

---

## ขั้นตอนที่ 813: Column Types ทั้งหมด

### Standard Types

```ruby
class CreateExamples < ActiveRecord::Migration[7.1]
  def change
    create_table :examples do |t|
      # String Types
      t.string   :name              # VARCHAR(255)
      t.string   :code, limit: 10   # VARCHAR(10)
      t.text     :description       # TEXT
      t.text     :content, limit: 65535  # MEDIUMTEXT (MySQL)

      # Numeric Types
      t.integer  :count             # INT
      t.integer  :big_number, limit: 8  # BIGINT
      t.float    :rating            # FLOAT
      t.decimal  :price             # DECIMAL
      t.decimal  :exact_price, precision: 10, scale: 2  # DECIMAL(10,2)
      t.bigint   :large_id          # BIGINT

      # Boolean
      t.boolean  :active, default: true

      # Date/Time
      t.date     :birth_date        # DATE
      t.time     :event_time        # TIME
      t.datetime :published_at      # DATETIME
      t.timestamp :recorded_at      # TIMESTAMP

      # Binary
      t.binary   :attachment        # BLOB
      t.binary   :data, limit: 1.megabyte

      # Special Types
      t.json     :metadata          # JSON (MySQL 5.7+, PostgreSQL)
      t.jsonb    :settings          # JSONB (PostgreSQL only - faster)
      t.hstore   :properties        # HSTORE (PostgreSQL only)
      t.uuid     :token             # UUID

      # Array (PostgreSQL only)
      t.string   :tags, array: true, default: []
      t.integer  :scores, array: true

      # Inet (PostgreSQL only)
      t.inet     :ip_address
      t.cidr     :network

      # Auto timestamps
      t.timestamps                  # created_at, updated_at
      t.timestamps null: false      # NOT NULL timestamps
    end
  end
end
```

### Column Options

```ruby
create_table :users do |t|
  # null: false - NOT NULL constraint
  t.string :email, null: false

  # default - ค่า default
  t.boolean :active, default: true
  t.integer :views_count, default: 0
  t.string :role, default: "member"

  # limit - ขนาด
  t.string :name, limit: 100

  # precision และ scale สำหรับ decimal
  t.decimal :price, precision: 10, scale: 2

  # index - เพิ่ม index
  t.string :slug, index: { unique: true }

  # comment - comment สำหรับ column
  t.string :code, comment: "รหัสสินค้า"
end
```

---

## ขั้นตอนที่ 814: create_table

### create_table พื้นฐาน

```ruby
class CreatePosts < ActiveRecord::Migration[7.1]
  def change
    create_table :posts do |t|
      t.string  :title, null: false
      t.text    :body
      t.boolean :published, default: false
      t.integer :views_count, default: 0
      t.string  :slug, index: { unique: true }

      t.references :user, null: false, foreign_key: true
      t.references :category, foreign_key: true

      t.timestamps null: false
    end
  end
end
```

### create_table Options

```ruby
# กำหนด primary key เอง
create_table :posts, primary_key: :post_id do |t|
  t.string :title
end

# ไม่มี primary key (สำหรับ join tables)
create_table :posts_tags, id: false do |t|
  t.integer :post_id, null: false
  t.integer :tag_id, null: false
end

# UUID primary key (PostgreSQL)
create_table :posts, id: :uuid do |t|
  t.string :title
end

# comment สำหรับ table
create_table :posts, comment: "ตารางบทความ" do |t|
  t.string :title
end

# force: :cascade - ลบ table เดิมก่อน (ระวังใน production!)
create_table :posts, force: :cascade do |t|
  t.string :title
end
```

### create_join_table

```ruby
# สร้าง join table สำหรับ HABTM
create_join_table :posts, :tags do |t|
  t.index :post_id
  t.index :tag_id
  t.index [:post_id, :tag_id], unique: true
end
# สร้าง table: posts_tags (alphabetically sorted)

# กำหนด table name เอง
create_join_table :posts, :tags, table_name: :taggings
```

---

## ขั้นตอนที่ 815: change_table

```ruby
class ModifyPosts < ActiveRecord::Migration[7.1]
  def change
    change_table :posts do |t|
      # เพิ่ม column
      t.string :subtitle
      t.integer :likes_count, default: 0

      # ลบ column
      t.remove :old_field

      # เปลี่ยน type
      t.change :views_count, :bigint

      # เพิ่ม index
      t.index :published_at
      t.index [:user_id, :published_at]

      # เพิ่ม foreign key
      t.references :editor, foreign_key: { to_table: :users }

      # เพิ่ม column เป็น group
      t.string :first_name, :last_name, :middle_name
    end
  end
end
```

---

## ขั้นตอนที่ 816: Column Operations

### add_column

```ruby
class AddFieldsToPosts < ActiveRecord::Migration[7.1]
  def change
    # เพิ่ม column เดียว
    add_column :posts, :subtitle, :string
    add_column :posts, :views_count, :integer, default: 0, null: false

    # เพิ่มพร้อมกำหนดตำแหน่ง (MySQL)
    add_column :posts, :featured, :boolean, after: :published
    add_column :posts, :code, :string, first: true

    # เพิ่ม reference column
    add_reference :posts, :category, foreign_key: true
    add_reference :posts, :editor, foreign_key: { to_table: :users }
  end
end
```

### remove_column

```ruby
class RemoveFieldsFromPosts < ActiveRecord::Migration[7.1]
  def change
    # ลบ column เดียว (reversible ต้องระบุ type)
    remove_column :posts, :old_field, :string

    # ลบหลาย columns
    remove_columns :posts, :field1, :field2

    # ลบ reference
    remove_reference :posts, :category
  end
end
```

### rename_column

```ruby
class RenameColumnsInPosts < ActiveRecord::Migration[7.1]
  def change
    rename_column :posts, :content, :body
    rename_column :users, :full_name, :name
  end
end
```

### change_column

```ruby
class ChangeColumnTypes < ActiveRecord::Migration[7.1]
  # change_column ไม่ reversible โดยตรง
  def up
    change_column :posts, :views_count, :bigint
    change_column :posts, :title, :string, limit: 500
    change_column_default :posts, :status, from: nil, to: "draft"
    change_column_null :posts, :title, false  # NOT NULL
  end

  def down
    change_column :posts, :views_count, :integer
    change_column :posts, :title, :string, limit: 255
    change_column_default :posts, :status, from: "draft", to: nil
    change_column_null :posts, :title, true
  end
end
```

---

## ขั้นตอนที่ 817: add_index

### ประเภทของ Index

```ruby
class AddIndexesToPosts < ActiveRecord::Migration[7.1]
  def change
    # Basic index
    add_index :posts, :title

    # Unique index
    add_index :posts, :slug, unique: true

    # Composite index (หลาย columns)
    add_index :posts, [:user_id, :created_at]
    add_index :posts, [:user_id, :published], name: "index_posts_user_published"

    # Partial index (PostgreSQL)
    add_index :posts, :published_at,
              where: "published = true",
              name: "index_published_posts_on_published_at"

    # ลบ index
    remove_index :posts, :title
    remove_index :posts, name: "index_posts_on_slug"

    # Rename index
    rename_index :posts, "old_index_name", "new_index_name"
  end
end
```

### Index ใน create_table

```ruby
create_table :users do |t|
  t.string :email, index: { unique: true }
  t.string :username, index: true
  t.string :token, index: { unique: true, name: "idx_user_token" }
  t.timestamps
end
```

---

## ขั้นตอนที่ 818: add_foreign_key

### Foreign Key Constraints

```ruby
class AddForeignKeys < ActiveRecord::Migration[7.1]
  def change
    # Foreign key พื้นฐาน
    add_foreign_key :posts, :users
    # posts.user_id → users.id

    # Foreign key กับ column ที่ไม่ใช่ convention
    add_foreign_key :posts, :users, column: :author_id
    # posts.author_id → users.id

    # Foreign key กับ primary key ต่างกัน
    add_foreign_key :posts, :categories, primary_key: :category_code

    # กำหนด on_delete behavior
    add_foreign_key :comments, :posts, on_delete: :cascade
    # ลบ post → ลบ comments ด้วย

    add_foreign_key :posts, :categories, on_delete: :nullify
    # ลบ category → set category_id เป็น NULL

    # ลบ foreign key
    remove_foreign_key :posts, :users
    remove_foreign_key :posts, column: :author_id
  end
end
```

---

## ขั้นตอนที่ 819: Running Migrations

### Migration Commands

```bash
# รัน migrations ที่ยังไม่ได้รัน
rails db:migrate

# รัน migration สำหรับ test database
rails db:migrate RAILS_ENV=test

# รัน migration ถึง version ที่กำหนด
rails db:migrate VERSION=20240115103000

# ดู status ของ migrations
rails db:migrate:status

# Rollback migration ล่าสุด
rails db:rollback

# Rollback หลาย steps
rails db:rollback STEP=3

# redo (rollback แล้ว migrate)
rails db:migrate:redo
rails db:migrate:redo STEP=3

# รัน migration เฉพาะ version
rails db:migrate:up VERSION=20240115103000
rails db:migrate:down VERSION=20240115103000

# Reset database (drop + create + migrate)
rails db:reset

# Drop, create, migrate, seed
rails db:setup
```

### Migration Status

```
rails db:migrate:status

database: myapp_development

 Status   Migration ID    Migration Name
--------------------------------------------------
   up     20240101000001  Create users
   up     20240101000002  Create posts
   up     20240110000001  Add email to users
  down    20240115103000  Add subtitle to posts
  down    20240116000001  Create comments
```

---

## ขั้นตอนที่ 820: Reversible Migrations

### change Method (Auto-reversible)

```ruby
class AddColumnToPosts < ActiveRecord::Migration[7.1]
  def change
    # Methods เหล่านี้ auto-reversible:
    add_column :posts, :subtitle, :string
    remove_column :posts, :old_field, :string  # ต้องระบุ type
    rename_column :posts, :content, :body
    add_index :posts, :slug
    remove_index :posts, :old_slug
    create_table :tags
    drop_table :old_tags, force: :cascade
    add_foreign_key :posts, :users
    remove_foreign_key :posts, :users
    change_column_default :posts, :status, from: nil, to: "draft"
    change_column_null :posts, :email, false
  end
end
```

### reversible Block

```ruby
class AddDataToPosts < ActiveRecord::Migration[7.1]
  def change
    add_column :posts, :slug, :string

    reversible do |dir|
      dir.up do
        # รันเมื่อ migrate
        Post.find_each do |post|
          post.update_column(:slug, post.title.parameterize)
        end
      end

      dir.down do
        # รันเมื่อ rollback
        # (อาจไม่ต้องทำอะไร)
      end
    end
  end
end
```

### up/down Method (Manual)

```ruby
class ComplexMigration < ActiveRecord::Migration[7.1]
  def up
    # รันเมื่อ migrate
    execute "CREATE INDEX CONCURRENTLY index_posts_on_search ON posts USING gin(to_tsvector('english', title || ' ' || body))"
    execute "ALTER TABLE posts ADD CONSTRAINT price_positive CHECK (price > 0)"
  end

  def down
    # รันเมื่อ rollback
    execute "DROP INDEX CONCURRENTLY IF EXISTS index_posts_on_search"
    execute "ALTER TABLE posts DROP CONSTRAINT price_positive"
  end
end
```

---

## ขั้นตอนที่ 821: Data Migrations

### เปลี่ยนข้อมูลพร้อม Schema

```ruby
class AddSlugToPostsAndPopulate < ActiveRecord::Migration[7.1]
  def up
    # เพิ่ม column
    add_column :posts, :slug, :string

    # Populate data
    Post.find_each do |post|
      post.update_column(:slug, post.title.parameterize)
    end

    # เพิ่ม constraint หลังจาก populate
    change_column_null :posts, :slug, false
    add_index :posts, :slug, unique: true
  end

  def down
    remove_column :posts, :slug
  end
end
```

### Data Migration แบบ Batch

```ruby
class MigrateUserRoles < ActiveRecord::Migration[7.1]
  def up
    add_column :users, :role_id, :integer

    # Process in batches เพื่อ performance
    User.find_each(batch_size: 1000) do |user|
      role_name = user.read_attribute_before_type_cast(:role)
      role = Role.find_or_create_by(name: role_name)
      user.update_column(:role_id, role.id)
    end

    change_column_null :users, :role_id, false
    add_foreign_key :users, :roles
  end

  def down
    remove_foreign_key :users, :roles
    remove_column :users, :role_id
  end
end
```

### Separate Data Migration (Best Practice)

```ruby
# ใช้ gem data-migrate หรือสร้าง rake task
# db/data/20240115_populate_slugs.rb

namespace :data do
  task populate_slugs: :environment do
    puts "Populating slugs..."
    Post.where(slug: nil).find_each do |post|
      post.update_column(:slug, post.title.parameterize)
    end
    puts "Done!"
  end
end
```

---

## ขั้นตอนที่ 822: schema.rb

### schema.rb คืออะไร?

```ruby
# db/schema.rb - Auto-generated จาก migrations
# ไม่ควรแก้ไขด้วยมือ!

ActiveRecord::Schema[7.1].define(version: 2024_01_15_103000) do
  # These are extensions that must be enabled in order to support this database
  enable_extension "plpgsql"
  enable_extension "pgcrypto"  # สำหรับ UUID

  create_table "posts", force: :cascade do |t|
    t.string "title", null: false
    t.text "body"
    t.boolean "published", default: false
    t.integer "views_count", default: 0
    t.string "slug"
    t.bigint "user_id", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["slug"], name: "index_posts_on_slug", unique: true
    t.index ["user_id"], name: "index_posts_on_user_id"
  end

  create_table "users", force: :cascade do |t|
    t.string "email", null: false
    t.string "name"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["email"], name: "index_users_on_email", unique: true
  end

  add_foreign_key "posts", "users"
end
```

### db:schema:load

```bash
# Load schema.rb แทนการรัน migrations ทั้งหมด (เร็วกว่า)
rails db:schema:load

# ใช้สำหรับ setup database ใหม่ใน development/test
rails db:create db:schema:load
```

### structure.sql (แทน schema.rb)

```ruby
# config/application.rb
config.active_record.schema_format = :sql
# สร้าง db/structure.sql แทน db/schema.rb
# ใช้เมื่อต้องการ database-specific features

rails db:structure:dump
rails db:structure:load
```

---

## ขั้นตอนที่ 823: seeds.rb

### db/seeds.rb

```ruby
# db/seeds.rb - ข้อมูลเริ่มต้น

# ล้างข้อมูลเก่า (ระวัง!)
# Post.destroy_all
# User.destroy_all

# สร้าง admin user
admin = User.find_or_create_by!(email: "admin@example.com") do |u|
  u.name = "Admin"
  u.password = "password123"
  u.role = "admin"
end

puts "Created admin: #{admin.email}"

# สร้าง categories
categories = ["Ruby", "Rails", "JavaScript", "Database"].map do |name|
  Category.find_or_create_by!(name: name)
end

puts "Created #{categories.count} categories"

# สร้าง sample posts
categories.each do |category|
  5.times do |i|
    Post.find_or_create_by!(
      title: "#{category.name} Post #{i + 1}",
      user: admin
    ) do |p|
      p.body = Faker::Lorem.paragraphs(number: 3).join("\n\n")
      p.category = category
      p.published = true
      p.published_at = rand(30).days.ago
    end
  end
end

puts "Created #{Post.count} posts"
```

### รัน Seeds

```bash
# รัน seeds
rails db:seed

# Setup (create + migrate + seed)
rails db:setup

# Reset + seed
rails db:seed:replant  # Rails 6+
# หรือ
rails db:truncate_all db:seed
```

### Environment-specific Seeds

```ruby
# db/seeds.rb
load(Rails.root.join("db", "seeds", "#{Rails.env}.rb"))

# db/seeds/development.rb
# ข้อมูล mock สำหรับ development

# db/seeds/production.rb
# ข้อมูล reference ที่จำเป็น (admin users, config data)
```

---

## ขั้นตอนที่ 824: Migration Best Practices

### ทำและไม่ควรทำ

```ruby
# ✅ ทำ: ใช้ null: false สำหรับ required fields
add_column :users, :email, :string, null: false

# ✅ ทำ: ใช้ default เสมอถ้ามี
add_column :posts, :views_count, :integer, default: 0, null: false

# ✅ ทำ: เพิ่ม index สำหรับ foreign keys
add_index :posts, :user_id  # ถ้าไม่ได้ใช้ add_reference

# ✅ ทำ: เพิ่ม index สำหรับ columns ที่ query บ่อย
add_index :users, :email, unique: true

# ❌ ไม่ควร: แก้ไข migration เก่าที่ commit แล้ว
# สร้าง migration ใหม่แทน

# ❌ ไม่ควร: ใช้ models ใน migration โดยตรง
# ควรใช้ execute SQL แทน
class BadMigration < ActiveRecord::Migration[7.1]
  def up
    Post.all.each { |p| p.update_column(:slug, p.title.parameterize) }
    # อันตราย! model อาจเปลี่ยนไปในอนาคต
  end
end

# ✅ ควรทำ: define model ชั่วคราวใน migration
class GoodMigration < ActiveRecord::Migration[7.1]
  class Post < ActiveRecord::Base; end  # isolated model

  def up
    Post.find_each do |post|
      post.update_column(:slug, post.title.parameterize)
    end
  end
end
```

### Performance ใน Production

```ruby
# เพิ่ม index แบบ concurrent (PostgreSQL) ไม่ lock table
class AddIndexToLargeTable < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!  # ต้องปิด transaction

  def change
    add_index :posts, :title, algorithm: :concurrently
  end
end

# MySQL: ใช้ algorithm และ lock options
class AddIndexMySQL < ActiveRecord::Migration[7.1]
  def change
    add_index :posts, :title, algorithm: :inplace, lock: :none
  end
end
```

---

## แบบฝึกหัดตอนที่ 37 (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** สร้าง migration สำหรับ `Article` table ที่มี fields:
- title (string, required)
- content (text)
- published (boolean, default: false)
- views_count (integer, default: 0)
- slug (string, unique)
- user_id (foreign key)
- timestamps

**ข้อ 2:** สร้าง migration เพิ่ม `subtitle` column ให้ `articles` table

**ข้อ 3:** สร้าง migration ลบ column `old_description` จาก `articles` table (reversible)

**ข้อ 4:** สร้าง migration เปลี่ยนชื่อ column `content` เป็น `body` ใน `articles` table

**ข้อ 5:** สร้าง migration เพิ่ม index ให้ `articles.title` และ composite index สำหรับ `[user_id, published]`

**ข้อ 6:** สร้าง migration เพิ่ม `category_id` foreign key ให้ `articles` ที่ cascade delete

**ข้อ 7:** รัน `rails db:migrate:status` และ `rails db:rollback` แล้วอธิบายผลลัพธ์

**ข้อ 8:** สร้าง join table สำหรับ `articles` และ `tags` พร้อม unique index

**ข้อ 9:** สร้าง `db/seeds.rb` ที่สร้าง 1 admin user และ 10 sample articles

**ข้อ 10:** สร้าง migration เปลี่ยน `views_count` จาก integer เป็น bigint

### ระดับกลาง

**ข้อ 11:** สร้าง migration ที่เพิ่ม column แล้ว populate data (reversible) เช่น เพิ่ม slug แล้ว generate จาก title

**ข้อ 12:** สร้าง migration เพิ่ม `deleted_at` column (soft delete) พร้อม index

**ข้อ 13:** สร้าง migration ที่ใช้ `change_table` เพื่อแก้ไขหลาย columns พร้อมกัน

**ข้อ 14:** สร้าง migration สำหรับ JSON column (metadata) ใน articles

**ข้อ 15:** สร้าง migration ที่ใช้ `reversible` block และ `up/down` method

**ข้อ 16:** เขียน migration เพิ่ม check constraint: price ต้องมากกว่า 0

**ข้อ 17:** ดู `db/schema.rb` และอธิบายโครงสร้างของ database

**ข้อ 18:** สร้าง migration ที่เพิ่ม column nullable แล้วแก้ไขเป็น NOT NULL หลัง populate

**ข้อ 19:** เขียน seeds file ที่ใช้ `find_or_create_by!` เพื่อไม่สร้าง duplicates

**ข้อ 20:** สร้าง migration สำหรับ PostgreSQL ที่ใช้ `add_index` แบบ concurrent

---

## สรุปตอนที่ 37

Migrations เป็นเครื่องมือที่ขาดไม่ได้สำหรับการจัดการ database schema:

| หัวข้อ | สิ่งสำคัญ |
|--------|-----------|
| Generate Migration | rails g migration + naming conventions |
| Column Types | string, text, integer, decimal, boolean, datetime, json |
| create_table | สร้าง table ใหม่พร้อม columns |
| change_table | แก้ไข table ที่มีอยู่ |
| add/remove/rename column | จัดการ columns แต่ละตัว |
| add_index | สร้าง index เพื่อ performance |
| add_foreign_key | database-level constraint |
| Running Migrations | db:migrate, db:rollback, db:status |
| Reversible | change method vs up/down |
| Data Migrations | เปลี่ยนข้อมูลพร้อม schema |
| schema.rb | snapshot ของ database structure |
| seeds.rb | ข้อมูลเริ่มต้น |

**กฎทอง:** อย่าแก้ไข migration ที่ commit ไปแล้ว → สร้าง migration ใหม่เสมอ

ตอนถัดไป: **ตอนที่ 38** - Associations
