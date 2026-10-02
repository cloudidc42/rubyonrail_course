# ตอนที่ 36: Models และ Active Record (Steps 781-810)

## บทนำ

Active Record เป็น ORM (Object-Relational Mapping) ของ Rails ที่ทำให้เราทำงานกับฐานข้อมูลได้ด้วย Ruby objects แทนที่จะเขียน SQL โดยตรง Active Record ใช้ Pattern ที่ชื่อ "Active Record Pattern" ซึ่ง Martin Fowler เป็นผู้คิดค้น

---

## Step 781: Active Record Pattern คืออะไร?

Active Record Pattern คือ design pattern ที่:
1. **แต่ละ class แทนตาราง** - `User` class แทนตาราง `users`
2. **แต่ละ instance แทน row** - `User.new` แทน row ในตาราง
3. **แต่ละ attribute แทน column** - `user.name` แทน column `name`

```ruby
# Database table: users
# | id | name    | email              | created_at |
# |----|---------|---------------------|------------|
# | 1  | สมชาย  | somchai@example.com | 2024-01-01 |

# Ruby code
user = User.find(1)
user.name    # => "สมชาย"
user.email   # => "somchai@example.com"

# เปลี่ยนค่า
user.name = "สมหญิง"
user.save  # UPDATE users SET name='สมหญิง' WHERE id=1
```

### Rails Naming Conventions

| Model Name | Table Name |
|------------|-----------|
| User | users |
| Post | posts |
| Category | categories |
| OrderItem | order_items |
| Person | people |

---

## Step 782: สร้าง Model

### rails g model

```bash
# สร้าง model
rails g model User name:string email:string:uniq age:integer bio:text active:boolean

# สร้างไฟล์:
# app/models/user.rb
# db/migrate/20240101120000_create_users.rb
# test/models/user_test.rb
# test/fixtures/users.yml
```

### Migration ที่สร้างขึ้น

```ruby
# db/migrate/20240101120000_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :name
      t.string :email
      t.integer :age
      t.text :bio
      t.boolean :active, default: true
      
      t.timestamps  # สร้าง created_at และ updated_at
    end
    add_index :users, :email, unique: true
  end
end
```

### Model Class

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # ApplicationRecord สืบทอดจาก ActiveRecord::Base
  # ทุก features ของ Active Record พร้อมใช้งาน
end
```

### ApplicationRecord

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class
  
  # ใส่ code ที่ share ระหว่าง models ทั้งหมด
  include GlobalId::Identification
end
```

---

## Step 783: CRUD Operations

### Create (สร้าง record ใหม่)

```ruby
# วิธีที่ 1: new + save
user = User.new
user.name = "สมชาย"
user.email = "somchai@example.com"
user.save   # => true ถ้าสำเร็จ, false ถ้า validation ไม่ผ่าน

# วิธีที่ 2: new ด้วย hash
user = User.new(name: "สมชาย", email: "somchai@example.com")
user.save

# วิธีที่ 3: create (new + save รวมกัน)
user = User.create(name: "สมชาย", email: "somchai@example.com")
# => User object (อาจไม่ได้ save ถ้า validation ไม่ผ่าน)

# วิธีที่ 4: create! (raise exception ถ้าไม่สำเร็จ)
user = User.create!(name: "สมชาย", email: "somchai@example.com")
# raise ActiveRecord::RecordInvalid ถ้า validation ไม่ผ่าน

# สร้างหลาย records พร้อมกัน
users = User.create([
  { name: "สมชาย", email: "a@example.com" },
  { name: "สมหญิง", email: "b@example.com" }
])
```

### Read (อ่านข้อมูล)

```ruby
# ดึงทั้งหมด
users = User.all

# ดึงทีละ record
user = User.find(1)           # ด้วย id, raise RecordNotFound ถ้าไม่เจอ
user = User.find_by(email: "a@example.com")  # nil ถ้าไม่เจอ
user = User.find_by!(email: "a@example.com") # raise ถ้าไม่เจอ

# อื่นๆ
user = User.first              # record แรก
user = User.last               # record สุดท้าย
user = User.first(5)           # 5 records แรก
users = User.find([1, 2, 3])   # หลาย records

# ด้วย conditions
users = User.where(active: true)
users = User.where("age > ?", 18)
users = User.where("name LIKE ?", "%สม%")
```

### Update (อัปเดตข้อมูล)

```ruby
user = User.find(1)

# วิธีที่ 1: set attribute แล้ว save
user.name = "ชื่อใหม่"
user.save

# วิธีที่ 2: update (set + save รวมกัน)
user.update(name: "ชื่อใหม่", email: "new@example.com")
# => true ถ้าสำเร็จ

# วิธีที่ 3: update! (raise ถ้าไม่สำเร็จ)
user.update!(name: "ชื่อใหม่")

# อัปเดตทุก records ที่ match condition
User.where(active: false).update_all(deleted_at: Time.current)

# update_columns (ข้าม validations และ callbacks)
user.update_columns(email_verified: true)
user.update_column(:email_verified, true)  # single attribute
```

### Destroy (ลบข้อมูล)

```ruby
user = User.find(1)

# ลบ record เดียว (รัน callbacks)
user.destroy
user.destroy!  # raise ถ้าไม่สำเร็จ

# ลบด้วย id
User.destroy(1)

# ลบหลาย records
User.destroy([1, 2, 3])

# ลบด้วย condition (ไม่รัน callbacks)
User.where(active: false).delete_all
User.delete_all  # ลบทั้งหมด (ระวัง!)

# ลบ record เดียวโดยไม่รัน callbacks
User.delete(1)
user.delete
```

---

## Step 784: Finders

```ruby
# find - ด้วย id, raise RecordNotFound ถ้าไม่เจอ
user = User.find(1)
users = User.find([1, 2, 3])  # หลาย ids

# find_by - ด้วย attribute, nil ถ้าไม่เจอ
user = User.find_by(email: "a@example.com")
user = User.find_by(name: "สมชาย", active: true)

# find_by! - raise RecordNotFound ถ้าไม่เจอ
user = User.find_by!(email: "a@example.com")

# find_or_create_by
user = User.find_or_create_by(email: "a@example.com") do |u|
  u.name = "New User"
end

# find_or_initialize_by (ไม่ save)
user = User.find_or_initialize_by(email: "a@example.com")

# all
users = User.all

# first, last
user = User.first
user = User.last
users = User.first(3)  # 3 records แรก
users = User.last(3)   # 3 records สุดท้าย

# count, sum, average, minimum, maximum
User.count
User.where(active: true).count
User.sum(:age)
User.average(:age)
User.minimum(:age)
User.maximum(:age)

# exists?
User.exists?(1)
User.exists?(email: "a@example.com")
User.where(active: true).exists?

# any?, none?
User.any?
User.where(active: true).any?
User.none?

# pluck - ดึงเฉพาะ columns
User.pluck(:id, :name)    # => [[1, "สมชาย"], [2, "สมหญิง"]]
User.pluck(:email)        # => ["a@example.com", "b@example.com"]

# ids - ดึงเฉพาะ ids
User.ids  # => [1, 2, 3]

# pick - ดึง columns จาก record แรก
User.pick(:name, :email)  # => ["สมชาย", "a@example.com"]
```

---

## Step 785: Query Interface

### where

```ruby
# Simple equality
User.where(name: "สมชาย")
User.where(active: true, role: "admin")

# SQL fragment
User.where("age > ?", 18)
User.where("created_at > ?", 1.week.ago)
User.where("name LIKE ?", "%สม%")

# Multiple conditions
User.where("age > ? AND active = ?", 18, true)
User.where("age BETWEEN ? AND ?", 18, 30)

# With hash
User.where(role: ["admin", "moderator"])  # IN clause
User.where(id: [1, 2, 3])

# NOT
User.where.not(active: false)
User.where.not(role: "banned")

# OR (Rails 5+)
User.where(role: "admin").or(User.where(role: "moderator"))

# Named placeholders
User.where("name = :name AND role = :role", name: "สมชาย", role: "admin")
```

### order

```ruby
User.order(:name)                   # ASC ค่า default
User.order(name: :asc)
User.order(name: :desc)
User.order("created_at DESC")
User.order(:last_name, :first_name) # หลาย columns
User.order(age: :desc, name: :asc)
User.reorder(:email)                # replace การ order ก่อนหน้า
```

### limit, offset

```ruby
User.limit(10)              # ดึง 10 records
User.limit(10).offset(20)  # ดึง 10 records เริ่มจาก record ที่ 21
User.offset(20).limit(10)  # เหมือนกัน

# Pagination
page = params[:page].to_i || 1
per_page = 10
users = User.limit(per_page).offset((page - 1) * per_page)
```

### select

```ruby
# ดึงเฉพาะ columns ที่ต้องการ
User.select(:id, :name, :email)
User.select("id, name, email")
User.select("name, COUNT(*) as post_count")

# ใช้กับ joins
User.select("users.id, users.name, COUNT(posts.id) as posts_count")
    .joins(:posts)
    .group("users.id")
```

### joins

```ruby
# INNER JOIN
User.joins(:posts)
User.joins(:posts, :comments)

# หลาย levels
User.joins(posts: :comments)

# SQL fragment
User.joins("LEFT OUTER JOIN posts ON posts.user_id = users.id")

# joins พร้อม where
User.joins(:posts).where(posts: { status: "published" })

# includes (Eager loading) - ดูในส่วน Associations
User.includes(:posts).where(posts: { published: true })
```

### group, having

```ruby
# GROUP BY
Post.group(:status)
Post.group(:status).count
# => {"draft" => 5, "published" => 10, "archived" => 2}

Post.group(:category_id).sum(:views_count)

# HAVING
Post.group(:user_id).having("COUNT(*) > 5")
```

### distinct

```ruby
User.select(:name).distinct
Post.joins(:tags).select(:category_id).distinct
```

### Chaining Queries

```ruby
# Queries สามารถ chain ได้
users = User.where(active: true)
            .where("age > ?", 18)
            .order(:name)
            .limit(10)
            .offset(20)
            .includes(:posts)

# Lazy evaluation - query ไม่รันจนกว่าจะ iterate
relation = User.where(active: true)  # ยังไม่รัน SQL
relation.each { |u| puts u.name }    # รัน SQL ตอนนี้

# none - return empty relation (ไม่ query DB)
users = User.none
```

---

## Step 786: Scopes

```ruby
class Post < ApplicationRecord
  # Named scope
  scope :published, -> { where(status: "published") }
  scope :draft, -> { where(status: "draft") }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_user, ->(user) { where(user: user) }
  
  # Scope ที่ condition-based
  scope :active, -> { where(active: true) }
  
  # Scope ด้วย time
  scope :this_week, -> { where("created_at > ?", 1.week.ago) }
  scope :this_month, -> { where("created_at > ?", 1.month.ago) }
  
  # Scope ที่รับ parameters
  scope :by_category, ->(cat) { where(category_id: cat) if cat.present? }
  scope :search, ->(q) { where("title ILIKE ?", "%#{q}%") if q.present? }
  scope :limit_to, ->(n) { limit(n) }
  
  # Merging scopes
  scope :featured_recent, -> { published.recent.limit(5) }
  
  # Scope with includes
  scope :with_author, -> { includes(:user) }
  scope :with_details, -> { includes(:user, :tags, :category) }
end
```

### ใช้ Scopes

```ruby
# Simple scopes
Post.published
Post.draft
Post.recent

# Chaining scopes
Post.published.recent.limit(10)

# Scope ด้วย parameter
Post.by_user(current_user)
Post.by_category(params[:category_id])
Post.search(params[:q])

# Scope ใน controller
@posts = Post.published
             .search(params[:q])
             .by_category(params[:category_id])
             .with_author
             .recent
             .page(params[:page])

# Chaining scope กับ where
Post.published.where("views_count > ?", 100)

# Count ด้วย scope
Post.published.count
Post.draft.where(user: current_user).count
```

---

## Step 787: Default Scope

```ruby
class Post < ApplicationRecord
  # Default scope - ใช้ทุก query
  default_scope { order(created_at: :desc) }
  
  # Default scope สำหรับ soft delete
  default_scope { where(deleted_at: nil) }
end

# ใช้
Post.all           # => SELECT * FROM posts WHERE deleted_at IS NULL ORDER BY created_at DESC
Post.where(active: true)  # => เพิ่ม default scope อัตโนมัติ

# ยกเลิก default scope
Post.unscoped.all
Post.unscoped.where(deleted_at: [nil, 0])
```

**ข้อควรระวัง:** Default scope อาจทำให้เกิด unexpected behavior แนะนำให้ใช้ named scope แทน

```ruby
# แทนที่จะใช้ default_scope
class Post < ApplicationRecord
  scope :active, -> { where(deleted_at: nil) }
  scope :recent, -> { order(created_at: :desc) }
  
  # ใช้ใน controller แทน
  # Post.active.recent
end
```

---

## Step 788: Callbacks

Callbacks คือ hooks ที่รันก่อน/หลังการ save, create, update, destroy

### Callback ทั้งหมด

```ruby
class User < ApplicationRecord
  # Validation callbacks
  before_validation :normalize_email
  after_validation :log_validation_errors
  
  # Create callbacks
  before_create :generate_slug
  after_create :send_welcome_email
  around_create :with_logging
  
  # Update callbacks
  before_update :track_changes
  after_update :clear_cache
  
  # Save callbacks (create + update)
  before_save :set_full_name
  after_save :sync_to_external_service
  around_save :with_transaction_logging
  
  # Destroy callbacks
  before_destroy :check_dependencies
  after_destroy :cleanup_files
  around_destroy :with_audit_log
  
  # Commit callbacks (หลัง DB transaction commit)
  after_commit :send_notification, on: [:create, :update]
  after_create_commit :enqueue_indexing_job
  after_update_commit :update_cache
  after_destroy_commit :remove_from_search_index
  
  # Rollback callbacks
  after_rollback :cleanup_temp_files
  
  private
  
  def normalize_email
    self.email = email.to_s.downcase.strip
  end
  
  def generate_slug
    self.slug = name.parameterize
  end
  
  def send_welcome_email
    UserMailer.welcome_email(self).deliver_later
  end
  
  def set_full_name
    self.full_name = "#{first_name} #{last_name}".strip
  end
  
  def check_dependencies
    if orders.any?
      errors.add(:base, "ไม่สามารถลบผู้ใช้ที่มี orders ได้")
      throw(:abort)  # หยุด destroy
    end
  end
  
  def log_validation_errors
    Rails.logger.warn("Validation errors: #{errors.full_messages}") if errors.any?
  end
  
  def track_changes
    @changed_attributes_before_save = changes.dup
  end
end
```

### ลำดับ Callbacks

```
สร้าง record ใหม่:
1. before_validation
2. after_validation
3. before_save
4. before_create
5. (database INSERT)
6. after_create
7. after_save
8. after_commit / after_create_commit

อัปเดต record:
1. before_validation
2. after_validation
3. before_save
4. before_update
5. (database UPDATE)
6. after_update
7. after_save
8. after_commit / after_update_commit

ลบ record:
1. before_destroy
2. (database DELETE)
3. after_destroy
4. after_commit / after_destroy_commit
```

### Conditional Callbacks

```ruby
class Post < ApplicationRecord
  before_save :generate_excerpt, if: :content_changed?
  after_create :notify_followers, if: -> { published? }
  before_destroy :check_admin_approval, unless: :admin_approved?
  
  after_create :send_notification, if: :should_notify?
  
  private
  
  def should_notify?
    published? && !draft?
  end
  
  def generate_excerpt
    self.excerpt = content.truncate(200)
  end
end
```

### throw(:abort) เพื่อหยุด callbacks chain

```ruby
class Post < ApplicationRecord
  before_destroy :prevent_if_published
  
  private
  
  def prevent_if_published
    if published?
      errors.add(:base, "ไม่สามารถลบบทความที่เผยแพร่แล้ว")
      throw(:abort)  # หยุด destroy
    end
  end
end

# ใน controller
if @post.destroy
  # ลบสำเร็จ
else
  # ลบไม่สำเร็จ - @post.errors มีข้อมูล
  render :show, alert: @post.errors.full_messages.join(", ")
end
```

---

## Step 789: Dirty Tracking

```ruby
class User < ApplicationRecord
end

user = User.find(1)
user.name = "ชื่อใหม่"

# changed? - มีการเปลี่ยนแปลงหรือเปล่า
user.changed?          # => true
user.name_changed?     # => true
user.email_changed?    # => false

# changes - hash ของการเปลี่ยนแปลง
user.changes           # => {"name" => ["ชื่อเดิม", "ชื่อใหม่"]}
user.name_change       # => ["ชื่อเดิม", "ชื่อใหม่"]

# previous_changes (หลัง save)
user.save
user.previous_changes  # => {"name" => ["ชื่อเดิม", "ชื่อใหม่"], "updated_at" => [...]}

# changed_attributes - attribute ที่เปลี่ยน
user.changed_attributes  # => {"name" => "ชื่อเดิม"}

# will_save_change_to? (ก่อน save - ดีกว่า _changed?)
user.will_save_change_to_name?    # => true
user.will_save_change_to_email?   # => false

# saved_change_to? (หลัง save)
user.save
user.saved_change_to_name?  # => true
user.saved_change_to_name   # => ["ชื่อเดิม", "ชื่อใหม่"]
```

### ใช้ใน Callbacks

```ruby
class Post < ApplicationRecord
  after_save :reindex, if: :saved_change_to_content?
  after_save :clear_cache, if: -> { saved_change_to_title? || saved_change_to_content? }
  
  before_save :update_word_count, if: :will_save_change_to_content?
  
  private
  
  def reindex
    SearchIndex.update(self)
  end
  
  def clear_cache
    Rails.cache.delete("post_#{id}")
  end
  
  def update_word_count
    self.word_count = content.to_s.split.size
  end
end
```

---

## Step 790: Counter Cache

Counter cache ช่วยเก็บจำนวน associated records ไว้ใน column พิเศษ เพื่อลด queries

```ruby
# Migration
class AddCommentsCountToPosts < ActiveRecord::Migration[7.0]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false
  end
end

# Model
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
  # หรือ
  belongs_to :post, counter_cache: :comments_count
end

class Post < ApplicationRecord
  has_many :comments
  # comments_count column จะ update อัตโนมัติ
end

# ใช้งาน
post = Post.find(1)
post.comments_count  # => 5 (ดึงจาก column ไม่ต้อง query)

# แทนที่จะ
post.comments.count  # => รัน COUNT query

# Reset counter cache (ถ้า count ไม่ sync)
Post.reset_counters(post.id, :comments)
Post.all.each { |p| Post.reset_counters(p.id, :comments) }
```

---

## Step 791: Soft Delete Pattern

```ruby
# Migration
class AddDeletedAtToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :deleted_at, :datetime
    add_index :users, :deleted_at
  end
end

# User Model
class User < ApplicationRecord
  scope :active, -> { where(deleted_at: nil) }
  scope :deleted, -> { where.not(deleted_at: nil) }
  
  def soft_delete!
    update!(deleted_at: Time.current)
  end
  
  def restore!
    update!(deleted_at: nil)
  end
  
  def deleted?
    deleted_at.present?
  end
  
  def active?
    !deleted?
  end
end

# ใช้งาน
user.soft_delete!   # ซ่อน user
user.restore!       # กู้คืน user
user.deleted?       # => true

User.active         # ดึงเฉพาะ users ที่ active
User.deleted        # ดึงเฉพาะ users ที่ soft deleted
User.unscoped.all   # ดึงทั้งหมด รวม deleted
```

---

## Step 792: Virtual Attributes

Virtual attributes คือ attributes ที่ไม่ได้เก็บใน database

```ruby
class User < ApplicationRecord
  # Virtual attribute ด้วย attr_accessor
  attr_accessor :password, :password_confirmation, :current_password
  
  # Virtual attribute ที่ compute จาก database columns
  def full_name
    "#{first_name} #{last_name}".strip
  end
  
  def full_name=(name)
    parts = name.split(" ", 2)
    self.first_name = parts[0]
    self.last_name = parts[1]
  end
  
  # Virtual attribute ที่เป็น boolean
  def adult?
    age.to_i >= 18
  end
  
  # Virtual attribute ด้วย ActiveModel::Attributes (Rails 6+)
  attribute :terms_accepted, :boolean, default: false
  attribute :referral_code, :string
end

# ใช้งาน
user = User.new
user.full_name = "สมชาย รักษ์ดี"
user.first_name  # => "สมชาย"
user.last_name   # => "รักษ์ดี"
user.full_name   # => "สมชาย รักษ์ดี"

user.password = "secret123"  # ไม่บันทึก DB แต่ใช้ใน validation
user.adult?       # => true/false
user.terms_accepted = true
```

---

## Step 793: Computed Attributes / Custom Select

```ruby
class User < ApplicationRecord
  # Custom select ด้วย SQL expression
  def self.with_post_count
    select("users.*, COUNT(posts.id) AS posts_count")
      .left_joins(:posts)
      .group("users.id")
  end
  
  def self.with_stats
    select("users.*,
            COUNT(DISTINCT posts.id) AS posts_count,
            COUNT(DISTINCT comments.id) AS comments_count")
      .left_joins(:posts, :comments)
      .group("users.id")
  end
end

# ใช้งาน
users = User.with_post_count
users.each do |user|
  puts "#{user.name}: #{user.posts_count} posts"
end

# ใช้กับ find_by_sql
users = User.find_by_sql(
  "SELECT users.*, COUNT(posts.id) as posts_count 
   FROM users 
   LEFT JOIN posts ON posts.user_id = users.id 
   GROUP BY users.id"
)
```

---

## Step 794: acts_as_paranoid Overview

acts_as_paranoid เป็น gem สำหรับ soft delete ที่ elegant กว่า manual implementation

```ruby
# Gemfile
gem "acts_as_paranoid"

# Model
class User < ApplicationRecord
  acts_as_paranoid
  
  # สร้าง scope :only_deleted, :with_deleted, :without_deleted
  # Override destroy ให้ set deleted_at แทน
end

# Migration
class AddDeletedAtToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :deleted_at, :datetime
    add_index :users, :deleted_at
  end
end
```

```ruby
# ใช้งาน
user = User.find(1)
user.destroy              # soft delete (set deleted_at)
user.deleted_at           # => Time
User.find(1)              # raise RecordNotFound (ถูก soft deleted)
User.with_deleted.find(1) # => User (รวม soft deleted)
User.only_deleted         # => [user] (เฉพาะ soft deleted)
user.restore              # กู้คืน

# Destroy จริงๆ
user.really_destroy!
```

---

## ตัวอย่าง Model ที่สมบูรณ์

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  # Associations
  belongs_to :user
  belongs_to :category, optional: true
  has_many :comments, dependent: :destroy
  has_many :taggings, dependent: :destroy
  has_many :tags, through: :taggings
  has_one_attached :cover_image
  
  # Enums
  enum status: { draft: 0, published: 1, archived: 2 }
  
  # Validations
  validates :title, presence: true, length: { minimum: 5, maximum: 200 }
  validates :content, presence: true
  validates :slug, uniqueness: true, allow_blank: true
  
  # Callbacks
  before_validation :generate_slug, if: :title_changed?
  before_save :set_published_at, if: :status_changed_to_published?
  after_commit :notify_subscribers, on: :create, if: :published?
  
  # Scopes
  scope :published, -> { where(status: :published) }
  scope :draft, -> { where(status: :draft) }
  scope :recent, -> { order(created_at: :desc) }
  scope :popular, -> { order(views_count: :desc) }
  scope :featured, -> { where(featured: true) }
  scope :by_category, ->(cat) { where(category: cat) if cat }
  scope :search, ->(q) { where("title ILIKE ? OR content ILIKE ?", "%#{q}%", "%#{q}%") if q.present? }
  scope :with_details, -> { includes(:user, :tags, :category) }
  
  # Class methods
  def self.trending
    published
      .where("created_at > ?", 1.week.ago)
      .order(views_count: :desc)
      .limit(10)
  end
  
  # Instance methods
  def increment_views!
    increment!(:views_count)
  end
  
  def reading_time
    words = content.to_s.split.size
    (words / 200.0).ceil
  end
  
  def excerpt(length = 200)
    content.to_s.truncate(length)
  end
  
  def published_by?(user)
    self.user == user
  end
  
  private
  
  def generate_slug
    base_slug = title.parameterize
    count = 0
    candidate = base_slug
    
    while Post.where(slug: candidate).where.not(id: id).exists?
      count += 1
      candidate = "#{base_slug}-#{count}"
    end
    
    self.slug = candidate
  end
  
  def set_published_at
    self.published_at = Time.current if published?
  end
  
  def status_changed_to_published?
    status_changed? && published?
  end
  
  def notify_subscribers
    NotifySubscribersJob.perform_later(self)
  end
end
```

---

## แบบฝึกหัด (Steps 795-810)

### แบบฝึกหัดที่ 1
สร้าง User model ที่มี name, email, age, และ bio

**เฉลย:**
```bash
rails g model User name:string email:string:uniq age:integer bio:text
rails db:migrate
```

### แบบฝึกหัดที่ 2
สร้าง User record ด้วย 3 วิธีที่แตกต่างกัน

**เฉลย:**
```ruby
# วิธีที่ 1
user = User.new(name: "สมชาย", email: "somchai@example.com")
user.save

# วิธีที่ 2
user = User.create(name: "สมหญิง", email: "somying@example.com")

# วิธีที่ 3
user = User.new
user.name = "ประหยัด"
user.email = "prayat@example.com"
user.save!
```

### แบบฝึกหัดที่ 3
ดึง users ทั้งหมดที่ active = true และ age > 20 เรียงตาม name

**เฉลย:**
```ruby
users = User.where(active: true).where("age > ?", 20).order(:name)
```

### แบบฝึกหัดที่ 4
สร้าง scope `recent` ที่ return posts 7 วันที่ผ่านมา

**เฉลย:**
```ruby
class Post < ApplicationRecord
  scope :recent, -> { where("created_at > ?", 7.days.ago) }
end
```

### แบบฝึกหัดที่ 5
สร้าง scope `search` ที่ filter ด้วย title ถ้ามีค่า

**เฉลย:**
```ruby
scope :search, ->(q) {
  where("title ILIKE ?", "%#{q}%") if q.present?
}
```

### แบบฝึกหัดที่ 6
เพิ่ม before_save callback ที่แปลง email เป็น lowercase

**เฉลย:**
```ruby
class User < ApplicationRecord
  before_save :normalize_email
  
  private
  
  def normalize_email
    self.email = email.to_s.downcase.strip
  end
end
```

### แบบฝึกหัดที่ 7
ใช้ after_create callback เพื่อส่ง welcome email

**เฉลย:**
```ruby
class User < ApplicationRecord
  after_create :send_welcome_email
  
  private
  
  def send_welcome_email
    UserMailer.welcome_email(self).deliver_later
  end
end
```

### แบบฝึกหัดที่ 8
ตรวจสอบ dirty tracking ว่า email เปลี่ยนหรือเปล่า

**เฉลย:**
```ruby
user = User.find(1)
user.email = "new@example.com"
user.email_changed?  # => true
user.email_change    # => ["old@example.com", "new@example.com"]
user.changed?        # => true
```

### แบบฝึกหัดที่ 9
implement soft delete ใน User model

**เฉลย:**
```ruby
class User < ApplicationRecord
  scope :active, -> { where(deleted_at: nil) }
  
  def soft_delete!
    update!(deleted_at: Time.current)
  end
  
  def restore!
    update!(deleted_at: nil)
  end
  
  def deleted?
    deleted_at.present?
  end
end
```

### แบบฝึกหัดที่ 10
สร้าง virtual attribute `full_name` ที่ combine first_name + last_name

**เฉลย:**
```ruby
class User < ApplicationRecord
  def full_name
    [first_name, last_name].compact.join(" ")
  end
  
  def full_name=(name)
    parts = name.to_s.split(" ", 2)
    self.first_name = parts[0]
    self.last_name = parts[1]
  end
end
```

### แบบฝึกหัดที่ 11
ใช้ pluck เพื่อดึง email ของ users ทั้งหมดที่ active

**เฉลย:**
```ruby
emails = User.where(active: true).pluck(:email)
# => ["a@example.com", "b@example.com", ...]
```

### แบบฝึกหัดที่ 12
count posts ตาม status ด้วย group

**เฉลย:**
```ruby
counts = Post.group(:status).count
# => {"draft" => 5, "published" => 10, "archived" => 2}
```

### แบบฝึกหัดที่ 13
สร้าง before_destroy callback ที่ป้องกันลบ admin user

**เฉลย:**
```ruby
class User < ApplicationRecord
  before_destroy :prevent_admin_deletion
  
  private
  
  def prevent_admin_deletion
    if admin?
      errors.add(:base, "ไม่สามารถลบ admin user ได้")
      throw(:abort)
    end
  end
end
```

### แบบฝึกหัดที่ 14
ใช้ find_or_create_by เพื่อสร้าง tag ถ้ายังไม่มี

**เฉลย:**
```ruby
tag = Tag.find_or_create_by(name: "ruby") do |t|
  t.description = "Ruby programming language"
  t.color = "#CC342D"
end
```

### แบบฝึกหัดที่ 15
สร้าง scope ที่ join กับ table อื่นและ filter

**เฉลย:**
```ruby
class Post < ApplicationRecord
  scope :by_author_email, ->(email) {
    joins(:user).where(users: { email: email })
  }
end

# ใช้งาน
Post.by_author_email("admin@example.com")
```

### แบบฝึกหัดที่ 16
ใช้ update_all เพื่ออัปเดต posts เก่าให้เป็น archived

**เฉลย:**
```ruby
Post.where("created_at < ?", 1.year.ago)
    .where(status: "draft")
    .update_all(status: "archived", archived_at: Time.current)
```

### แบบฝึกหัดที่ 17
สร้าง class method ที่ return top 5 users ที่มี posts มากที่สุด

**เฉลย:**
```ruby
class User < ApplicationRecord
  def self.most_prolific(limit = 5)
    joins(:posts)
      .group("users.id")
      .order("COUNT(posts.id) DESC")
      .limit(limit)
      .select("users.*, COUNT(posts.id) as posts_count")
  end
end

# ใช้งาน
User.most_prolific(10).each do |user|
  puts "#{user.name}: #{user.posts_count} posts"
end
```

### แบบฝึกหัดที่ 18
ใช้ after_commit callback เพื่อ clear cache หลัง update

**เฉลย:**
```ruby
class Post < ApplicationRecord
  after_commit :clear_post_cache, on: [:update, :destroy]
  
  private
  
  def clear_post_cache
    Rails.cache.delete("post_#{id}")
    Rails.cache.delete("posts_list")
  end
end
```

### แบบฝึกหัดที่ 19
สร้าง counter cache สำหรับ posts count ใน User

**เฉลย:**
```ruby
# Migration
class AddPostsCountToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :posts_count, :integer, default: 0, null: false
  end
end

# Post model
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end

# ใช้งาน
user.posts_count  # ดึงจาก column - ไม่ต้อง query
```

### แบบฝึกหัดที่ 20
implement pagination แบบ manual โดยไม่ใช้ gem

**เฉลย:**
```ruby
class Post < ApplicationRecord
  def self.paginate(page: 1, per_page: 10)
    offset = (page.to_i - 1) * per_page.to_i
    limit(per_page).offset(offset)
  end
end

# ใช้งาน
@posts = Post.published.paginate(page: params[:page], per_page: 15)
```

### แบบฝึกหัดที่ 21
สร้าง model method ที่คำนวณ age จาก birthdate

**เฉลย:**
```ruby
class User < ApplicationRecord
  def age
    return unless birthdate
    today = Date.today
    years = today.year - birthdate.year
    years -= 1 if today < birthdate + years.years
    years
  end
  
  scope :adults, -> { where("birthdate <= ?", 18.years.ago) }
end
```

### แบบฝึกหัดที่ 22
ใช้ exists? เพื่อตรวจสอบก่อน create

**เฉลย:**
```ruby
unless User.exists?(email: params[:email])
  @user = User.create!(email: params[:email], name: params[:name])
end

# หรือ
User.find_or_create_by(email: params[:email])
```

### แบบฝึกหัดที่ 23
สร้าง scope สำหรับ date range

**เฉลย:**
```ruby
class Post < ApplicationRecord
  scope :created_between, ->(start_date, end_date) {
    where(created_at: start_date.beginning_of_day..end_date.end_of_day)
  }
  
  scope :this_month, -> {
    where(created_at: Time.current.beginning_of_month..Time.current.end_of_month)
  }
end

# ใช้งาน
Post.created_between(Date.parse("2024-01-01"), Date.parse("2024-01-31"))
Post.this_month
```

### แบบฝึกหัดที่ 24
ใช้ transaction เพื่อให้แน่ใจว่า operations ทำงานพร้อมกัน

**เฉลย:**
```ruby
def transfer_points(from_user, to_user, amount)
  ActiveRecord::Base.transaction do
    from_user.decrement!(:points, amount)
    to_user.increment!(:points, amount)
    
    PointTransfer.create!(
      from_user: from_user,
      to_user: to_user,
      amount: amount
    )
  end
rescue ActiveRecord::RecordInvalid => e
  false
end
```

### แบบฝึกหัดที่ 25
สร้าง model method ที่ return formatted data สำหรับ API

**เฉลย:**
```ruby
class Post < ApplicationRecord
  def as_api_response
    {
      id: id,
      title: title,
      excerpt: content.truncate(200),
      published_at: published_at&.iso8601,
      author: {
        id: user.id,
        name: user.name
      },
      tags: tags.pluck(:name),
      comments_count: comments_count
    }
  end
end

# ใน controller
render json: @post.as_api_response
```

### แบบฝึกหัดที่ 26
implement enum สำหรับ user role

**เฉลย:**
```ruby
class User < ApplicationRecord
  enum role: { member: 0, moderator: 1, admin: 2 }
  
  scope :admins, -> { where(role: :admin) }
  scope :moderators, -> { where(role: :moderator) }
end

# ใช้งาน
user.admin?        # => true/false
user.role          # => "admin"
User.admins        # scope
user.admin!        # เปลี่ยน role เป็น admin
User.roles         # => {"member" => 0, "moderator" => 1, "admin" => 2}
```

### แบบฝึกหัดที่ 27
ใช้ select เพื่อดึงเฉพาะ columns ที่ต้องการ

**เฉลย:**
```ruby
users = User.select(:id, :name, :email).where(active: true)
# SQL: SELECT id, name, email FROM users WHERE active = true

# ใช้กับ pluck ถ้าต้องการ array
User.where(active: true).pluck(:id, :name, :email)
# => [[1, "สมชาย", "a@example.com"], ...]
```

### แบบฝึกหัดที่ 28
สร้าง around_save callback สำหรับ logging

**เฉลย:**
```ruby
class Post < ApplicationRecord
  around_save :log_save_time
  
  private
  
  def log_save_time
    start = Time.current
    yield
    elapsed = Time.current - start
    Rails.logger.info "Saved #{self.class}##{id} in #{elapsed.round(3)}s. Changed: #{previous_changes.keys.join(', ')}"
  end
end
```

### แบบฝึกหัดที่ 29
สร้าง model ที่ใช้ UUID เป็น primary key

**เฉลย:**
```ruby
# Migration
class CreateArticles < ActiveRecord::Migration[7.0]
  def change
    create_table :articles, id: :uuid do |t|
      t.string :title
      t.text :content
      t.timestamps
    end
  end
end

# Model
class Article < ApplicationRecord
  # id จะเป็น UUID อัตโนมัติ
  # "550e8400-e29b-41d4-a716-446655440000"
end
```

### แบบฝึกหัดที่ 30
เขียน test สำหรับ User model scope

**เฉลย:**
```ruby
# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  describe "scopes" do
    describe ".active" do
      let!(:active_user) { create(:user, deleted_at: nil) }
      let!(:deleted_user) { create(:user, deleted_at: Time.current) }
      
      it "returns only active users" do
        expect(User.active).to include(active_user)
        expect(User.active).not_to include(deleted_user)
      end
    end
    
    describe ".search" do
      let!(:ruby_post) { create(:post, title: "Ruby on Rails") }
      let!(:python_post) { create(:post, title: "Python Tutorial") }
      
      it "filters by title" do
        expect(Post.search("Ruby")).to include(ruby_post)
        expect(Post.search("Ruby")).not_to include(python_post)
      end
      
      it "returns all when query is blank" do
        expect(Post.search("")).to include(ruby_post, python_post)
      end
    end
  end
end
```

---

## สรุป

ใน Models และ Active Record เราได้เรียนรู้:

1. **Active Record Pattern** - Object-Relational Mapping
2. **สร้าง Model** - rails g model, migrations
3. **CRUD** - create, read, update, destroy
4. **Finders** - find, find_by, where, first, last
5. **Query Interface** - where, order, limit, offset, select, joins
6. **Scopes** - named scopes, parameterized scopes
7. **Default scope** - ระวัง gotchas
8. **Callbacks** - before/after/around save, create, update, destroy
9. **Dirty Tracking** - changed?, changes, will_save_change_to?
10. **Counter cache** - ลด COUNT queries
11. **Soft delete** - ซ่อนแทนลบจริง
12. **Virtual attributes** - computed attributes
13. **Enums** - type-safe attributes
14. **Transactions** - atomicity
