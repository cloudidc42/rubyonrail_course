# ตอนที่ 38: Associations (Steps 831-860)

## บทนำ

Associations (ความสัมพันธ์) ใน Active Record คือการเชื่อมโยงระหว่าง models ต่างๆ ทำให้การทำงานกับ related data ง่ายขึ้นมาก แทนที่จะต้องเขียน SQL JOIN เองทุกครั้ง Rails จัดการให้เราด้วย methods ที่ intuitive

---

## ขั้นตอนที่ 831: belongs_to

### belongs_to พื้นฐาน

`belongs_to` ใช้เมื่อ model มี foreign key ที่ชี้ไปยัง model อื่น

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  # comment.post_id → posts.id
end

# app/models/post.rb
class Post < ApplicationRecord
  # ไม่จำเป็นต้องมี has_many ก็ได้ แต่ควรมีเพื่อ full association
  has_many :comments
end

# Migration
class CreateComments < ActiveRecord::Migration[7.1]
  def change
    create_table :comments do |t|
      t.text :body
      t.references :post, null: false, foreign_key: true
      # สร้าง post_id column + foreign key constraint
    end
  end
end
```

### belongs_to Options ทั้งหมด

```ruby
class Comment < ApplicationRecord
  # class_name - ชื่อ class ต่างจาก convention
  belongs_to :author, class_name: "User"
  # comment.author_id → users.id

  # foreign_key - ชื่อ foreign key ต่างจาก convention
  belongs_to :post, foreign_key: :article_id
  # comment.article_id → posts.id

  # primary_key - primary key ของ association (default: id)
  belongs_to :post, primary_key: :uuid

  # optional - ไม่บังคับ (Rails 5+: belongs_to ต้อง required by default)
  belongs_to :category, optional: true
  # เท่ากับ validates :category, presence: false

  # touch - touch parent timestamps
  belongs_to :post, touch: true
  # เมื่อ update comment → update post.updated_at ด้วย
  belongs_to :post, touch: :comments_updated_at
  # touch column ที่ระบุแทน updated_at

  # counter_cache
  belongs_to :post, counter_cache: true
  # post.comments_count อัปเดตอัตโนมัติ

  belongs_to :post, counter_cache: :total_comments
  # ใช้ column ชื่อ total_comments

  # dependent (ไม่ค่อยใช้กับ belongs_to)
  belongs_to :user

  # inverse_of - ระบุ inverse association
  belongs_to :post, inverse_of: :comments

  # scope
  belongs_to :post, -> { where(published: true) }

  # polymorphic
  belongs_to :commentable, polymorphic: true
end
```

### การใช้งาน belongs_to Methods

```ruby
comment = Comment.find(1)

# Access associated object
comment.post          # return Post object
comment.post_id       # return post_id integer

# Build new association
comment.build_post(title: "New Post")  # new Post ที่ยังไม่ save
comment.create_post(title: "New Post") # create + save Post

# Assign association
comment.post = Post.find(5)
comment.post_id = 5

# Check if association loaded
comment.post.loaded?   # false (belum loaded)
comment.post           # query database
comment.post.loaded?   # true

# Reload association
comment.reload_post    # clear cache + reload
```

---

## ขั้นตอนที่ 832: has_many

### has_many พื้นฐาน

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments
  # posts.id ← comments.post_id
end

# ใช้งาน
post = Post.find(1)
post.comments         # ActiveRecord::Associations::CollectionProxy
post.comments.count   # SELECT COUNT(*) FROM comments WHERE post_id = 1
post.comments.all     # SELECT * FROM comments WHERE post_id = 1
```

### has_many Options ทั้งหมด

```ruby
class User < ApplicationRecord
  # class_name
  has_many :authored_posts, class_name: "Post", foreign_key: :author_id

  # foreign_key
  has_many :posts, foreign_key: :author_id

  # primary_key
  has_many :posts, primary_key: :uuid

  # through (join via another table)
  has_many :post_tags, through: :posts
  has_many :tags, through: :post_tags

  # source (ใช้กับ through)
  has_many :followers_users, through: :follows, source: :follower

  # dependent - จัดการเมื่อ parent ถูกลบ
  has_many :posts, dependent: :destroy       # destroy each (run callbacks)
  has_many :posts, dependent: :delete_all    # single SQL DELETE (no callbacks)
  has_many :posts, dependent: :nullify       # set foreign key = NULL
  has_many :posts, dependent: :restrict_with_error    # error ถ้ามี posts
  has_many :posts, dependent: :restrict_with_exception  # exception ถ้ามี posts

  # as (polymorphic)
  has_many :comments, as: :commentable

  # inverse_of
  has_many :posts, inverse_of: :user

  # scope (lambda)
  has_many :published_posts, -> { where(published: true) }, class_name: "Post"
  has_many :recent_posts, -> { order(created_at: :desc).limit(5) }, class_name: "Post"

  # autosave
  has_many :posts, autosave: true
  # save posts ด้วยเมื่อ save user

  # validate
  has_many :posts, validate: true
  # validate posts เมื่อ save user
end
```

### has_many Collection Methods

```ruby
user = User.find(1)

# ดึงข้อมูล
user.posts              # all posts
user.posts.first        # first post
user.posts.last         # last post
user.posts.count        # COUNT query
user.posts.size         # uses counter cache ถ้ามี
user.posts.length       # load all + count
user.posts.empty?       # check if empty
user.posts.any?         # check if any
user.posts.many?        # check if more than 1

# เพิ่ม records
user.posts << post            # เพิ่ม post เข้า collection
user.posts.push(post)         # เหมือนกัน
user.posts.append(post)       # เหมือนกัน
user.posts.create(title: "New") # สร้าง + เพิ่ม
user.posts.build(title: "New")  # สร้างแต่ยังไม่ save

# ลบ records
user.posts.delete(post)        # ลบ association (set FK = null หรือ destroy ตาม dependent)
user.posts.destroy(post)       # destroy post (run callbacks)
user.posts.delete_all          # DELETE SQL
user.posts.destroy_all         # destroy each post

# Query ใน collection
user.posts.where(published: true)
user.posts.order(:title)
user.posts.limit(5)
user.posts.find(1)             # find ใน context ของ user

# Check membership
user.posts.include?(post)

# Clear
user.posts.clear               # ขึ้นอยู่กับ dependent option
```

---

## ขั้นตอนที่ 833: has_one

### has_one พื้นฐาน

```ruby
class User < ApplicationRecord
  has_one :profile
  # users.id ← profiles.user_id
end

class Profile < ApplicationRecord
  belongs_to :user
end

# Migration
class CreateProfiles < ActiveRecord::Migration[7.1]
  def change
    create_table :profiles do |t|
      t.string :bio
      t.string :website
      t.string :avatar_url
      t.references :user, null: false, foreign_key: true, index: { unique: true }
    end
  end
end
```

### has_one Options

```ruby
class User < ApplicationRecord
  # class_name
  has_one :main_address, class_name: "Address"

  # foreign_key
  has_one :profile, foreign_key: :account_id

  # dependent
  has_one :profile, dependent: :destroy
  has_one :settings, dependent: :delete
  has_one :profile, dependent: :nullify

  # through
  has_one :account_details, through: :profile

  # scope
  has_one :active_subscription, -> { where(active: true) },
          class_name: "Subscription"
end
```

### has_one Methods

```ruby
user = User.find(1)

# Access
user.profile          # Profile object หรือ nil

# Build
user.build_profile(bio: "Hello")   # new Profile ยังไม่ save
user.create_profile(bio: "Hello")  # create + save

# Assign
user.profile = Profile.new(bio: "Hello")

# Check
user.profile.nil?     # ไม่มี profile
user.profile.present? # มี profile
```

---

## ขั้นตอนที่ 834: has_many :through

### has_many :through พื้นฐาน

```ruby
# Doctor → Appointment → Patient
class Doctor < ApplicationRecord
  has_many :appointments
  has_many :patients, through: :appointments
end

class Appointment < ApplicationRecord
  belongs_to :doctor
  belongs_to :patient
end

class Patient < ApplicationRecord
  has_many :appointments
  has_many :doctors, through: :appointments
end

# Migration
class CreateAppointments < ActiveRecord::Migration[7.1]
  def change
    create_table :appointments do |t|
      t.datetime :scheduled_at
      t.string :status
      t.references :doctor, null: false, foreign_key: true
      t.references :patient, null: false, foreign_key: true
      t.timestamps
    end
  end
end
```

### ตัวอย่าง Blog: Post → PostTag → Tag

```ruby
class Post < ApplicationRecord
  has_many :post_tags, dependent: :destroy
  has_many :tags, through: :post_tags
end

class PostTag < ApplicationRecord
  belongs_to :post
  belongs_to :tag
end

class Tag < ApplicationRecord
  has_many :post_tags, dependent: :destroy
  has_many :posts, through: :post_tags
end

# การใช้งาน
post = Post.find(1)

# เพิ่ม tags
post.tags << Tag.find_or_create_by(name: "ruby")
post.tags.create(name: "rails")

# ดู tags
post.tags                          # all tags
post.tags.map(&:name)              # ["ruby", "rails"]
post.tags.where(featured: true)    # ใช้ conditions ได้

# ลบ tags
post.tags.delete(tag)              # ลบแค่ join record
post.tags.clear                    # ลบ join records ทั้งหมด

# Access ผ่าน join model
post.post_tags                     # join records
post.post_tags.first.tag           # tag ผ่าน join
```

### has_many :through กับ Extra Attributes

```ruby
class User < ApplicationRecord
  has_many :memberships
  has_many :groups, through: :memberships
end

class Membership < ApplicationRecord
  belongs_to :user
  belongs_to :group

  # extra attributes บน join model
  t.string :role
  t.datetime :joined_at
  t.boolean :admin
end

class Group < ApplicationRecord
  has_many :memberships
  has_many :members, through: :memberships, source: :user
  has_many :admins, -> { where(memberships: { admin: true }) },
           through: :memberships, source: :user
end

# การใช้งาน
user = User.find(1)
group = Group.find(1)

# เพิ่ม membership พร้อม attributes
user.memberships.create(group: group, role: "admin", joined_at: Time.now)

# หรือ
Membership.create(user: user, group: group, role: "member")

# Access
user.memberships.where(role: "admin")
user.groups
group.members
group.admins
```

---

## ขั้นตอนที่ 835: has_one :through

```ruby
class User < ApplicationRecord
  has_one :account
  has_one :account_history, through: :account
end

class Account < ApplicationRecord
  belongs_to :user
  has_one :account_history
end

class AccountHistory < ApplicationRecord
  belongs_to :account
end

# การใช้งาน
user = User.find(1)
user.account          # Account
user.account_history  # AccountHistory ผ่าน account
```

### ตัวอย่างที่ใช้งานจริง

```ruby
# Country → State → City
class Country < ApplicationRecord
  has_many :states
  has_many :cities, through: :states
  # ผ่าน states ไปยัง cities
end

class State < ApplicationRecord
  belongs_to :country
  has_many :cities
end

class City < ApplicationRecord
  belongs_to :state
  has_one :country, through: :state
end

# supplier → account → account_history
class Supplier < ApplicationRecord
  has_one :account
  has_one :account_history, through: :account
end
```

---

## ขั้นตอนที่ 836: has_and_belongs_to_many (HABTM)

### HABTM พื้นฐาน

```ruby
class Post < ApplicationRecord
  has_and_belongs_to_many :tags
end

class Tag < ApplicationRecord
  has_and_belongs_to_many :posts
end

# Migration - ต้องสร้าง join table เอง
class CreatePostsTags < ActiveRecord::Migration[7.1]
  def change
    create_join_table :posts, :tags do |t|
      t.index :post_id
      t.index :tag_id
      t.index [:post_id, :tag_id], unique: true
    end
  end
end
```

### HABTM vs has_many :through

```
HABTM:
- ง่าย ไม่มี join model
- ไม่สามารถเพิ่ม extra attributes ได้
- ไม่สามารถ validate join
- เหมาะกับ simple many-to-many

has_many :through:
- มี join model
- สามารถเพิ่ม extra attributes ได้
- สามารถ validate join ได้
- ยืดหยุ่นกว่า
- แนะนำให้ใช้เสมอ
```

---

## ขั้นตอนที่ 837: Polymorphic Associations

### Polymorphic คืออะไร?

Polymorphic association ให้ model หนึ่ง belongs_to หลาย model ที่ต่างชนิดกัน

```
Comment สามารถ belongs_to: Post, Article, Photo, Video
```

### Implement Polymorphic

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
  # ต้องมี: commentable_type, commentable_id columns
end

# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

# app/models/article.rb
class Article < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

# app/models/photo.rb
class Photo < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

# Migration
class CreateComments < ActiveRecord::Migration[7.1]
  def change
    create_table :comments do |t|
      t.text :body
      t.string :commentable_type, null: false  # ชื่อ class: "Post", "Article", "Photo"
      t.bigint :commentable_id, null: false    # id ของ object
      t.references :user, foreign_key: true
      t.timestamps
    end

    # หรือใช้ references แบบ polymorphic
    add_reference :comments, :commentable, polymorphic: true, null: false
    # สร้าง commentable_type และ commentable_id
    add_index :comments, [:commentable_type, :commentable_id]
  end
end
```

### Database Structure

```
comments table:
+----+------------------+-----------------+-----------+
| id | commentable_type | commentable_id  | body      |
+----+------------------+-----------------+-----------+
| 1  | "Post"           | 5               | "Great!"  |
| 2  | "Article"        | 3               | "Thanks"  |
| 3  | "Photo"          | 8               | "Nice"    |
+----+------------------+-----------------+-----------+
```

### การใช้งาน Polymorphic

```ruby
# สร้าง comments
post = Post.find(1)
post.comments.create(body: "Great post!", user: current_user)

article = Article.find(1)
article.comments.create(body: "Interesting!", user: current_user)

# ดู comments
post.comments        # Comments สำหรับ post นี้
article.comments     # Comments สำหรับ article นี้

# Access commentable จาก comment
comment = Comment.find(1)
comment.commentable        # return Post object
comment.commentable_type   # => "Post"
comment.commentable_id     # => 5

# Polymorphic route helper
link_to "ดู", [comment.commentable]  # ไปยัง post หรือ article
```

### Polymorphic ตัวอย่างอื่น

```ruby
# Taggable - Tag หลาย types
class Tag < ApplicationRecord
  has_many :taggings
  has_many :taggables, through: :taggings, source: :taggable
end

class Tagging < ApplicationRecord
  belongs_to :tag
  belongs_to :taggable, polymorphic: true
end

class Post < ApplicationRecord
  has_many :taggings, as: :taggable
  has_many :tags, through: :taggings
end

# Attachable - Attachment หลาย types
class Attachment < ApplicationRecord
  belongs_to :attachable, polymorphic: true
end

class User < ApplicationRecord
  has_many :attachments, as: :attachable
end

class Post < ApplicationRecord
  has_many :attachments, as: :attachable
end
```

---

## ขั้นตอนที่ 838: Self-referential Associations

### Tree Structure

```ruby
# Employee hierarchy: Employee has many subordinates, belongs_to manager
class Employee < ApplicationRecord
  belongs_to :manager, class_name: "Employee", optional: true
  has_many :subordinates, class_name: "Employee", foreign_key: :manager_id

  def ancestors
    result = []
    current = self
    while current.manager
      result << current.manager
      current = current.manager
    end
    result
  end

  def descendants
    result = []
    queue = subordinates.to_a
    while queue.any?
      employee = queue.shift
      result << employee
      queue.concat(employee.subordinates.to_a)
    end
    result
  end
end

# Migration
class CreateEmployees < ActiveRecord::Migration[7.1]
  def change
    create_table :employees do |t|
      t.string :name
      t.references :manager, foreign_key: { to_table: :employees }, null: true
      t.timestamps
    end
  end
end

# การใช้งาน
ceo = Employee.create(name: "CEO")
vp1 = Employee.create(name: "VP Engineering", manager: ceo)
vp2 = Employee.create(name: "VP Marketing", manager: ceo)
dev1 = Employee.create(name: "Dev 1", manager: vp1)
dev2 = Employee.create(name: "Dev 2", manager: vp1)

ceo.subordinates      # [VP Engineering, VP Marketing]
vp1.subordinates      # [Dev 1, Dev 2]
dev1.manager          # VP Engineering
dev1.ancestors        # [VP Engineering, CEO]
ceo.descendants       # [VP Engineering, VP Marketing, Dev 1, Dev 2]
```

### Friendship (Bidirectional)

```ruby
class User < ApplicationRecord
  # Friendships ที่ user เป็น initiator
  has_many :friendships, dependent: :destroy
  has_many :friends, through: :friendships

  # Friendships ที่ user เป็น recipient
  has_many :inverse_friendships, class_name: "Friendship",
           foreign_key: :friend_id, dependent: :destroy
  has_many :inverse_friends, through: :inverse_friendships, source: :user

  # ดู friends ทั้งสองทิศทาง
  def all_friends
    (friends + inverse_friends).uniq
  end
end

class Friendship < ApplicationRecord
  belongs_to :user
  belongs_to :friend, class_name: "User"
end

# Migration
create_table :friendships do |t|
  t.references :user, null: false, foreign_key: true
  t.references :friend, null: false, foreign_key: { to_table: :users }
  t.string :status, default: "pending"
  t.timestamps
end
add_index :friendships, [:user_id, :friend_id], unique: true
```

### Category (Nested Categories)

```ruby
class Category < ApplicationRecord
  belongs_to :parent, class_name: "Category", optional: true
  has_many :children, class_name: "Category", foreign_key: :parent_id, dependent: :destroy

  scope :roots, -> { where(parent_id: nil) }
  scope :leaves, -> { where.not(id: select(:parent_id).where.not(parent_id: nil)) }

  def root?
    parent_id.nil?
  end

  def leaf?
    children.empty?
  end

  def depth
    return 0 if root?
    parent.depth + 1
  end

  def path
    return [self] if root?
    parent.path + [self]
  end
end
```

---

## ขั้นตอนที่ 839: N+1 Problem และ Eager Loading

### N+1 Problem คืออะไร?

```ruby
# N+1 Problem
posts = Post.all
posts.each do |post|
  puts post.user.name  # query แยกสำหรับแต่ละ post!
end

# SQL queries:
# SELECT * FROM posts                    (1 query)
# SELECT * FROM users WHERE id = 1      (1 query)
# SELECT * FROM users WHERE id = 2      (1 query)
# SELECT * FROM users WHERE id = 3      (1 query)
# ... N+1 queries สำหรับ N posts
```

### แก้ด้วย Eager Loading

```ruby
# includes - โหลด association ล่วงหน้า
posts = Post.includes(:user)
posts.each do |post|
  puts post.user.name  # ไม่ query เพิ่มแล้ว!
end

# SQL queries:
# SELECT * FROM posts
# SELECT * FROM users WHERE id IN (1, 2, 3, ...)
# เพียง 2 queries เสมอ
```

### includes, preload, eager_load

```ruby
# includes - ให้ Rails เลือกวิธีที่ดีที่สุด
Post.includes(:user, :comments)
Post.includes(comments: :author)
Post.includes(:user, comments: [:author, :likes])

# preload - บังคับใช้ 2 separate queries
Post.preload(:user)
# SELECT * FROM posts
# SELECT * FROM users WHERE id IN (...)

# eager_load - บังคับใช้ LEFT OUTER JOIN
Post.eager_load(:user)
# SELECT posts.*, users.* FROM posts LEFT OUTER JOIN users ON ...

# ใช้ includes กับ where บน association
# ต้องใช้ references หรือ eager_load
Post.includes(:comments).where(comments: { approved: true })
# ถ้าใช้ string condition ต้องใช้ references
Post.includes(:comments)
    .where("comments.approved = ?", true)
    .references(:comments)
# หรือใช้ eager_load แทน includes
Post.eager_load(:comments).where(comments: { approved: true })
```

### Detecting N+1 ด้วย Bullet gem

```ruby
# Gemfile
gem "bullet", group: :development

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
  Bullet.bullet_logger = true
  Bullet.unused_eager_loading_enable = true
  Bullet.counter_cache_enable = true
end
```

---

## ขั้นตอนที่ 840: Counter Cache

### ติดตั้ง Counter Cache

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# Migration
class AddCommentsCountToPosts < ActiveRecord::Migration[7.1]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false

    # Reset counts สำหรับ data ที่มีอยู่
    Post.find_each do |post|
      Post.reset_counters(post.id, :comments)
    end
  end
end

# ใช้งาน
post = Post.find(1)
post.comments_count   # ดึงจาก column ในตาราง (ไม่ต้อง query)
post.comments.size    # ใช้ counter cache ถ้ามี
post.comments.count   # query database เสมอ
```

### Custom Counter Cache Name

```ruby
class Like < ApplicationRecord
  belongs_to :post, counter_cache: :likes_count
  # ใช้ column likes_count แทน likes_count
end

# Reset specific counter
Post.reset_counters(post_id, :likes)
```

---

## ขั้นตอนที่ 841: Association Callbacks

```ruby
class User < ApplicationRecord
  has_many :posts,
           before_add: :check_post_limit,
           after_add: :send_notification,
           before_remove: :log_removal,
           after_remove: :update_stats
end

class Post < ApplicationRecord
  belongs_to :user
end

# ใน User model
private

def check_post_limit(post)
  if posts.count >= 100
    raise "ไม่สามารถเพิ่มบทความได้ มีมากกว่า 100 แล้ว"
  end
end

def send_notification(post)
  PostNotificationJob.perform_later(post.id)
end

def log_removal(post)
  Rails.logger.info "Removing post #{post.id} from user #{id}"
end

def update_stats(post)
  update_column(:posts_count, posts.count)
end
```

---

## ขั้นตอนที่ 842: Association Extensions

```ruby
class User < ApplicationRecord
  has_many :posts do
    def published
      where(published: true)
    end

    def recent
      order(created_at: :desc).limit(5)
    end

    def search(query)
      where("title LIKE ?", "%#{query}%")
    end

    def publish_all!
      update_all(published: true)
    end
  end
end

# การใช้งาน
user.posts.published
user.posts.recent
user.posts.search("Ruby")
user.posts.published.recent
user.posts.publish_all!
```

---

## ขั้นตอนที่ 843: Strict Loading

### Strict Loading (Rails 6.1+)

```ruby
# ป้องกัน lazy loading - force eager loading
post = Post.strict_loading.find(1)
post.user   # raise ActiveRecord::StrictLoadingViolationError!

# ต้อง eager load ล่วงหน้า
post = Post.includes(:user).strict_loading.find(1)
post.user   # OK

# กำหนด globally ใน model
class Post < ApplicationRecord
  self.strict_loading_by_default = true
end

# กำหนดสำหรับ association
class User < ApplicationRecord
  has_many :posts, strict_loading: true
end
```

---

## ขั้นตอนที่ 844: Association Tips และ Best Practices

### Bidirectional Associations

```ruby
class Post < ApplicationRecord
  belongs_to :user, inverse_of: :posts
  has_many :comments, inverse_of: :post
end

class User < ApplicationRecord
  has_many :posts, inverse_of: :user
end

class Comment < ApplicationRecord
  belongs_to :post, inverse_of: :comments
end
```

### Polymorphic กับ STI

```ruby
# STI (Single Table Inheritance)
class Shape < ApplicationRecord
end

class Circle < Shape
  def area
    Math::PI * radius ** 2
  end
end

class Rectangle < Shape
  def area
    width * height
  end
end

# Migration
create_table :shapes do |t|
  t.string :type  # required for STI
  t.float :radius
  t.float :width
  t.float :height
end
```

---

## แบบฝึกหัดตอนที่ 38 (30 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** สร้าง `User` และ `Profile` models ที่มี has_one/belongs_to relationship พร้อม migration

**ข้อ 2:** สร้าง `Post` และ `Comment` models พร้อม has_many/belongs_to relationship ด้วย dependent: :destroy

**ข้อ 3:** สร้าง belongs_to ที่ optional (category ไม่จำเป็น) พร้อม test

**ข้อ 4:** ใช้ has_many :through เพื่อเชื่อม Posts และ Tags ผ่าน PostTag join model

**ข้อ 5:** ใช้ `touch: true` ใน belongs_to เพื่ออัปเดต parent's updated_at

**ข้อ 6:** สร้าง self-referential association สำหรับ Category (parent/children)

**ข้อ 7:** แก้ N+1 problem ด้วย includes ใน posts index page

**ข้อ 8:** สร้าง counter cache สำหรับ comments count ใน posts

**ข้อ 9:** ใช้ class_name และ foreign_key เพื่อสร้าง has_many :authored_posts

**ข้อ 10:** ใช้ collection methods: build, create, << เพื่อเพิ่ม records

### ระดับกลาง

**ข้อ 11:** Implement polymorphic comments ที่ใช้ได้กับทั้ง Post, Article, Photo

**ข้อ 12:** สร้าง Friendship model แบบ self-referential ที่มี status: pending/accepted/rejected

**ข้อ 13:** ใช้ has_many :through กับ extra attributes เช่น Membership ที่มี role และ joined_at

**ข้อ 14:** สร้าง scope ใน has_many เพื่อ filter เช่น published_posts, recent_posts

**ข้อ 15:** ใช้ association extension เพิ่ม custom methods ให้ collection

**ข้อ 16:** Implement Employee hierarchy ด้วย self-referential association

**ข้อ 17:** ใช้ eager_load แทน includes เมื่อต้อง filter บน associated table

**ข้อ 18:** สร้าง association callback เพื่อ validate เมื่อเพิ่ม records เข้า collection

**ข้อ 19:** ใช้ merge เพื่อ combine scopes จาก associated models

**ข้อ 20:** Implement tagging system ด้วย polymorphic many-to-many

### ระดับสูง

**ข้อ 21:** ติดตั้ง Bullet gem และ detect N+1 problems ในระบบ

**ข้อ 22:** Implement threaded comments (comments ที่มี parent comment)

**ข้อ 23:** สร้าง multi-level category tree ที่ query ทุก descendants ด้วย recursive query

**ข้อ 24:** ใช้ strict_loading เพื่อป้องกัน lazy loading ใน production

**ข้อ 25:** สร้าง has_many with complex scope ที่รวม joins

**ข้อ 26:** Implement ระบบ followers/following ด้วย self-referential HABTM

**ข้อ 27:** ใช้ preload vs eager_load อธิบายความแตกต่างด้วย SQL queries

**ข้อ 28:** สร้าง polymorphic likes system สำหรับ Post, Comment, Photo

**ข้อ 29:** Implement has_one through ใน 3 levels: Country → State → City

**ข้อ 30:** เขียน tests ครอบคลุม associations ทั้งหมด รวม N+1 prevention

---

## สรุปตอนที่ 38

| Association | ใช้เมื่อ | Database |
|------------|---------|----------|
| belongs_to | model มี foreign key | FK column ใน table นี้ |
| has_many | model มีหลาย records ใน table อื่น | FK column ในตาราง other |
| has_one | model มี 1 record ใน table อื่น | FK column ในตาราง other |
| has_many :through | many-to-many ที่มี extra attributes | join table |
| has_one :through | one-to-one ผ่าน join | join table |
| HABTM | simple many-to-many | join table ไม่มี model |
| polymorphic | belongs_to หลาย types | type + id columns |
| self-referential | hierarchy/tree | self FK column |

**Key Principles:**
1. **ใช้ has_many :through แทน HABTM** เสมอ (flexible กว่า)
2. **ใช้ includes เสมอ** เมื่อ access association ใน loop
3. **ใช้ dependent:** ป้องกัน orphan records
4. **ใช้ inverse_of** เพื่อ performance

ตอนถัดไป: **ตอนที่ 39** - Validations
