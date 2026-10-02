# ตอนที่ 38: Associations (Steps 831-860)

## บทนำ

Associations (ความสัมพันธ์) ใน Active Record ช่วยให้เราเชื่อมต่อ Models เข้าหากันและทำงานกับข้อมูลที่เกี่ยวข้องได้ง่ายขึ้น Rails มี association types หลายประเภทที่ครอบคลุม use cases ส่วนใหญ่

---

## Step 831: belongs_to

`belongs_to` ใช้เมื่อ model มี foreign key ไปยัง model อื่น

```ruby
# Post belongs_to User
# posts table มี column: user_id

class Post < ApplicationRecord
  belongs_to :user
end

# ใช้งาน
post = Post.find(1)
post.user          # => User object
post.user_id       # => 1
post.user = User.find(2)  # เปลี่ยน user
post.build_user(name: "New User")  # สร้าง user ใหม่
post.create_user!(name: "New User")  # สร้างและ save
```

### belongs_to options

```ruby
class Post < ApplicationRecord
  # optional: true - อนุญาตให้ user_id เป็น nil
  belongs_to :user, optional: true
  
  # class_name: กำหนด class ที่ใช้
  belongs_to :author, class_name: "User"
  # post.author => User
  
  # foreign_key: กำหนด foreign key column
  belongs_to :author, class_name: "User", foreign_key: "author_id"
  
  # primary_key: กำหนด column ที่ reference
  belongs_to :user, primary_key: :username, foreign_key: :author_username
  
  # touch: อัปเดต updated_at ของ parent เมื่อ child เปลี่ยน
  belongs_to :post, touch: true
  belongs_to :category, touch: :category_updated_at  # custom column
  
  # counter_cache: นับ children อัตโนมัติ
  belongs_to :user, counter_cache: true
  belongs_to :user, counter_cache: :articles_count
  
  # inverse_of: กำหนด inverse association
  belongs_to :user, inverse_of: :posts
  
  # polymorphic:
  belongs_to :commentable, polymorphic: true
  
  # dependent:
  belongs_to :user, dependent: :destroy  # ลบ user เมื่อ post ถูกลบ
end
```

### belongs_to Validation

```ruby
class Post < ApplicationRecord
  belongs_to :user  # Rails 5+ validates presence of user_id โดย default
  
  # ปิด validation
  belongs_to :user, optional: true
  
  # Manual validation
  validates :user, presence: true
end
```

---

## Step 832: has_many

`has_many` ใช้เมื่อ model มี children หลายตัว

```ruby
# User has_many Posts
# posts table มี column: user_id

class User < ApplicationRecord
  has_many :posts
end

# ใช้งาน
user = User.find(1)
user.posts                        # => [Post, Post, ...]
user.posts.count                  # => 5
user.posts.create(title: "New")  # สร้าง post ใหม่
user.posts.build(title: "Draft") # initialize แต่ไม่ save
user.posts.where(status: "published")  # query บน collection
user.posts.first                  # post แรก
user.posts << post                # เพิ่ม post ที่มีอยู่
user.posts.destroy(post)          # ลบ post ออกจาก collection
user.posts.delete(post)           # remove association (ไม่ destroy)
user.posts.clear                  # ลบทุก posts
user.posts.empty?                 # => false
user.posts.any?                   # => true
user.posts.size                   # => 5
user.posts.ids                    # => [1, 2, 3, 4, 5]
```

### has_many options

```ruby
class User < ApplicationRecord
  # class_name: กำหนด class
  has_many :written_posts, class_name: "Post"
  has_many :authored_articles, class_name: "Article"
  
  # foreign_key: กำหนด foreign key
  has_many :posts, foreign_key: "author_id"
  
  # dependent: จัดการ children เมื่อ parent ถูกลบ
  has_many :posts, dependent: :destroy    # ลบ posts ด้วย (รัน callbacks)
  has_many :posts, dependent: :delete_all # ลบ posts ด้วย (ไม่รัน callbacks)
  has_many :posts, dependent: :nullify    # set user_id = nil
  has_many :posts, dependent: :restrict_with_error  # ไม่ให้ลบถ้ามี posts
  has_many :posts, dependent: :restrict_with_exception  # raise exception
  
  # order: เรียงลำดับ
  has_many :posts, -> { order(created_at: :desc) }
  
  # conditions/scope
  has_many :published_posts, -> { where(status: "published") }, class_name: "Post"
  has_many :recent_posts, -> { order(created_at: :desc).limit(5) }, class_name: "Post"
  
  # through: ผ่าน join table
  has_many :comments, through: :posts
  
  # source: กำหนด source association
  has_many :followers, through: :follow_relationships, source: :follower
  
  # counter_cache
  has_many :posts  # user.posts_count ถ้า posts มี counter_cache
  
  # inverse_of
  has_many :posts, inverse_of: :user
  
  # primary_key
  has_many :posts, primary_key: :username, foreign_key: :author_username
  
  # strict_loading
  has_many :posts, strict_loading: true
end
```

---

## Step 833: has_one

`has_one` ใช้เมื่อ model มี child หนึ่งตัวเท่านั้น

```ruby
# User has_one Profile
# profiles table มี column: user_id

class User < ApplicationRecord
  has_one :profile
  has_one :avatar
end

class Profile < ApplicationRecord
  belongs_to :user
end

# ใช้งาน
user = User.find(1)
user.profile                        # => Profile object หรือ nil
user.profile = Profile.new(bio: "...")  # set profile
user.create_profile(bio: "...")    # สร้างและ save
user.build_profile(bio: "...")     # initialize แต่ไม่ save
user.profile.destroy               # ลบ profile
```

### has_one options

```ruby
class User < ApplicationRecord
  # ตัวเลือกคล้ายกับ has_many
  has_one :profile, dependent: :destroy
  has_one :profile, dependent: :nullify
  has_one :recent_post, -> { order(created_at: :desc) }, class_name: "Post"
  has_one :billing_address, -> { where(type: "billing") }, class_name: "Address"
end
```

---

## Step 834: has_many :through

`has_many :through` ใช้เชื่อมต่อผ่าน join model

```ruby
# User has_many Courses through Enrollments
# users table
# enrollments table (user_id, course_id, enrolled_at, grade)
# courses table

class User < ApplicationRecord
  has_many :enrollments
  has_many :courses, through: :enrollments
end

class Enrollment < ApplicationRecord
  belongs_to :user
  belongs_to :course
end

class Course < ApplicationRecord
  has_many :enrollments
  has_many :students, through: :enrollments, source: :user
end

# ใช้งาน
user = User.find(1)
user.courses          # => [Course, Course, ...]
user.enrollments      # => [Enrollment, ...]

# สร้าง enrollment (join)
user.courses << course  # สร้าง Enrollment record
user.courses.create    # สร้าง Course + Enrollment
user.enrollments.create(course: course, enrolled_at: Time.current)

# ลบ enrollment
user.courses.delete(course)  # ลบ Enrollment record
user.courses.destroy(course)  # ลบ Course + Enrollment (ระวัง!)

# ดึง join model data
enrollment = user.enrollments.find_by(course: course)
enrollment.grade  # => "A"
enrollment.enrolled_at
```

### has_many :through ที่ซับซ้อน

```ruby
# Blog system
class User < ApplicationRecord
  has_many :posts
  has_many :taggings, through: :posts
  has_many :tags, through: :taggings
  # User.find(1).tags => tags จาก posts ทั้งหมดของ user
  
  has_many :comments
  has_many :commented_posts, through: :comments, source: :post
end

# Doctor and Patient
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
```

---

## Step 835: has_one :through

```ruby
# User has_one Account through Profile
class User < ApplicationRecord
  has_one :profile
  has_one :account, through: :profile
end

class Profile < ApplicationRecord
  belongs_to :user
  has_one :account
end

class Account < ApplicationRecord
  belongs_to :profile
end

# ใช้งาน
user.account  # => Account ผ่าน Profile
```

---

## Step 836: has_and_belongs_to_many (HABTM)

HABTM ใช้ join table ที่ไม่มี model (ไม่แนะนำสำหรับ Rails ใหม่ๆ)

```ruby
# Posts and Tags
# join table: posts_tags (post_id, tag_id) - ไม่มี id, ไม่มี timestamps

class Post < ApplicationRecord
  has_and_belongs_to_many :tags
end

class Tag < ApplicationRecord
  has_and_belongs_to_many :posts
end

# Migration สำหรับ join table
class CreatePostsTags < ActiveRecord::Migration[7.0]
  def change
    create_table :posts_tags, id: false do |t|  # ไม่มี id
      t.bigint :post_id, null: false
      t.bigint :tag_id, null: false
    end
    
    add_index :posts_tags, :post_id
    add_index :posts_tags, :tag_id
    add_index :posts_tags, [:post_id, :tag_id], unique: true
    
    add_foreign_key :posts_tags, :posts
    add_foreign_key :posts_tags, :tags
  end
end

# ใช้งาน
post = Post.find(1)
post.tags           # => [Tag, Tag, ...]
post.tags << tag    # เพิ่ม tag
post.tags.delete(tag)  # ลบ tag
post.tags = [tag1, tag2]  # set tags ทั้งหมด
```

**แนะนำ:** ใช้ `has_many :through` แทน HABTM เพราะ:
- มี model กลาง ทำให้เพิ่ม attributes ได้
- Query ง่ายกว่า
- รองรับ validations, callbacks

---

## Step 837: Polymorphic Associations

```ruby
# Comment ถูก comment บน Post และ Photo

class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
end

class Post < ApplicationRecord
  has_many :comments, as: :commentable
end

class Photo < ApplicationRecord
  has_many :comments, as: :commentable
end

class Video < ApplicationRecord
  has_many :comments, as: :commentable
end

# Migration สำหรับ polymorphic
class CreateComments < ActiveRecord::Migration[7.0]
  def change
    create_table :comments do |t|
      t.text :content
      t.bigint :user_id
      t.references :commentable, polymorphic: true, null: false
      # สร้าง commentable_id (bigint) และ commentable_type (string)
      t.timestamps
    end
  end
end

# ใช้งาน
# สร้าง comment
post = Post.find(1)
post.comments.create(content: "ความคิดเห็น", user: current_user)

photo.comments.create(content: "สวยมาก!")

# ดึง comments
post.comments   # => [Comment, ...] ที่เป็น commentable_type = "Post"
photo.comments  # => [Comment, ...] ที่เป็น commentable_type = "Photo"

# ดึงจาก Comment
comment = Comment.find(1)
comment.commentable       # => Post หรือ Photo
comment.commentable_type  # => "Post"
comment.commentable_id    # => 1
```

### ตัวอย่างอื่นๆ ของ Polymorphic

```ruby
# Like system
class Like < ApplicationRecord
  belongs_to :user
  belongs_to :likeable, polymorphic: true
end

class Post < ApplicationRecord
  has_many :likes, as: :likeable
end

class Comment < ApplicationRecord
  has_many :likes, as: :likeable
end

# Image attachment
class Image < ApplicationRecord
  belongs_to :imageable, polymorphic: true
end

class User < ApplicationRecord
  has_many :images, as: :imageable
end

class Product < ApplicationRecord
  has_many :images, as: :imageable
end
```

---

## Step 838: Self-referential Associations

```ruby
# Category tree
class Category < ApplicationRecord
  belongs_to :parent, class_name: "Category", optional: true
  has_many :children, class_name: "Category", foreign_key: "parent_id",
           dependent: :destroy
  
  scope :roots, -> { where(parent_id: nil) }
  
  def ancestors
    result = []
    current = self
    while current.parent
      result << current.parent
      current = current.parent
    end
    result.reverse
  end
  
  def descendants
    children.flat_map { |child| [child] + child.descendants }
  end
  
  def root?
    parent_id.nil?
  end
  
  def leaf?
    children.empty?
  end
end

# ใช้งาน
root = Category.create(name: "Electronics")
sub = Category.create(name: "Smartphones", parent: root)
sub_sub = Category.create(name: "Apple", parent: sub)

root.children    # => [sub]
sub.parent       # => root
sub_sub.ancestors  # => [root, sub]
Category.roots   # => [root]

# Social network - followers
class User < ApplicationRecord
  has_many :followed_relationships, foreign_key: "follower_id",
           class_name: "Follow", dependent: :destroy
  has_many :following, through: :followed_relationships, source: :followed
  
  has_many :follower_relationships, foreign_key: "followed_id",
           class_name: "Follow", dependent: :destroy
  has_many :followers, through: :follower_relationships, source: :follower
end

class Follow < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"
end

# Migration
class CreateFollows < ActiveRecord::Migration[7.0]
  def change
    create_table :follows do |t|
      t.bigint :follower_id, null: false
      t.bigint :followed_id, null: false
      t.timestamps
    end
    
    add_index :follows, :follower_id
    add_index :follows, :followed_id
    add_index :follows, [:follower_id, :followed_id], unique: true
    add_foreign_key :follows, :users, column: :follower_id
    add_foreign_key :follows, :users, column: :followed_id
  end
end

# ใช้งาน
user1.following << user2  # user1 follows user2
user2.followers           # => [user1]
user1.following           # => [user2]
```

---

## Step 839: Association Options

```ruby
class Post < ApplicationRecord
  # dependent - จัดการ children เมื่อ parent ถูกลบ
  has_many :comments, dependent: :destroy    # ลบ comments ด้วย
  has_many :comments, dependent: :delete_all # SQL DELETE (เร็วกว่า ไม่รัน callback)
  has_many :comments, dependent: :nullify    # set post_id = NULL
  has_many :comments, dependent: :restrict_with_error
  has_many :comments, dependent: :restrict_with_exception
  
  # class_name - ชื่อ model ที่ใช้
  has_many :reviews, class_name: "Comment"
  belongs_to :creator, class_name: "User"
  
  # foreign_key - ชื่อ column
  has_many :comments, foreign_key: "article_id"
  belongs_to :author, foreign_key: "author_id", class_name: "User"
  
  # primary_key - ชื่อ column ที่ reference
  has_many :comments, primary_key: :slug, foreign_key: :post_slug
  
  # source - ใช้กับ has_many through
  has_many :tags, through: :taggings, source: :tag
  
  # source_type - ใช้กับ polymorphic through
  has_many :comments, through: :commentings, source: :commentable, source_type: "Post"
  
  # as - polymorphic association name
  has_many :images, as: :imageable
  
  # through - join model
  has_many :tags, through: :taggings
  
  # inverse_of - ระบุ inverse
  has_many :comments, inverse_of: :post
  
  # validate - validate children เมื่อ save parent
  has_many :comments, validate: false  # default: true
  
  # autosave - save children อัตโนมัติ
  has_many :comments, autosave: true  # default: false
  
  # strict_loading - raise error ถ้า lazy load
  has_many :comments, strict_loading: true
  
  # extend - extend association proxy
  has_many :comments do
    def spam
      where(spam: true)
    end
    
    def by_user(user)
      where(user: user)
    end
  end
end
```

---

## Step 840: Association Methods

```ruby
class User < ApplicationRecord
  has_many :posts
end

user = User.find(1)

# Collection methods
user.posts                     # ดึง all posts
user.posts(reload)             # reload จาก DB
user.posts.reload              # เหมือนกัน

# Create/Build
new_post = user.posts.build(title: "Draft")  # new ไม่ save
new_post = user.posts.new(title: "Draft")    # เหมือน build
saved_post = user.posts.create(title: "Post")  # สร้างและ save
saved_post = user.posts.create!(title: "Post") # raise ถ้าไม่สำเร็จ

# Add existing
user.posts << post1             # เพิ่ม post ที่มีอยู่
user.posts << [post1, post2]    # เพิ่มหลาย posts

# Remove
user.posts.delete(post)         # remove (set user_id = nil)
user.posts.destroy(post)        # ลบ post จาก DB
user.posts.delete([post1, post2]) # remove หลาย posts

# Check
user.posts.include?(post)       # => true/false
user.posts.loaded?              # => true ถ้า loaded แล้ว
user.posts.empty?               # => false
user.posts.any?                 # => true
user.posts.none?                # => false
user.posts.one?                 # => true/false
user.posts.many?                # => true ถ้ามีมากกว่า 1

# Count
user.posts.count                # SQL COUNT
user.posts.size                 # count แบบ smart (ใช้ cache ถ้าโหลดแล้ว)
user.posts.length               # load และนับ

# Clear
user.posts.clear                # ลบ association

# Reset (unload cache)
user.posts.reset

# Scoping
user.posts.published
user.posts.where(status: "draft")
user.posts.order(created_at: :desc).limit(5)

# Aggregate
user.posts.sum(:views_count)
user.posts.average(:views_count)
user.posts.maximum(:views_count)
user.posts.minimum(:views_count)

# IDs
user.posts.ids                  # => [1, 2, 3]
user.post_ids                   # เหมือนกัน
user.post_ids = [1, 2, 3]       # set posts by ids
```

---

## Step 841: Eager Loading

```ruby
# N+1 Problem
posts = Post.all
posts.each do |post|
  puts post.user.name  # ทุก post สร้าง 1 query => N+1 queries!
end

# Solution: includes
posts = Post.includes(:user)
posts.each do |post|
  puts post.user.name  # ไม่มี N+1 แล้ว
end
```

### includes

```ruby
# Load association ด้วย separate query (default)
Post.includes(:user)
Post.includes(:user, :category)
Post.includes(:user, :tags, comments: :user)
Post.includes(comments: [:user, :likes])

# ตัวอย่างซับซ้อน
Post.published
    .includes(:user, :category, :tags, comments: :user)
    .order(created_at: :desc)
    .limit(10)
```

### preload

```ruby
# ใช้ separate queries เสมอ
Post.preload(:user)
Post.preload(:user, :comments)
```

### eager_load

```ruby
# ใช้ LEFT OUTER JOIN
Post.eager_load(:user)
Post.eager_load(:user, :comments)

# ดีกว่า includes เมื่อต้องการ filter บน association
Post.eager_load(:user).where(users: { active: true })
```

---

## Step 842: N+1 Problem

### ปัญหา N+1

```ruby
# ❌ N+1 Problem
posts = Post.all          # 1 query: SELECT * FROM posts
posts.each do |post|
  puts post.user.name     # N queries: SELECT * FROM users WHERE id = ?
end
# รวม N+1 queries!

# ❌ N+1 กับ count
posts.each do |post|
  puts post.comments.count  # N queries!
end
```

### วิธีแก้ไข

```ruby
# ✅ includes
posts = Post.includes(:user)
posts.each do |post|
  puts post.user.name  # 2 queries total
end

# ✅ includes กับ where บน association
Post.includes(:user).where(users: { active: true })
# หรือ
Post.eager_load(:user).where(users: { active: true })

# ✅ counter_cache สำหรับ count
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end
# post.comments_count  # อ่านจาก column - ไม่ต้อง query

# ✅ includes หลายระดับ
Post.includes(comments: :user)

# ✅ select เฉพาะ columns ที่ต้องการ
Post.includes(:user).select("posts.*, users.name as author_name")

# ✅ joins แทน includes เมื่อต้องการ filter เท่านั้น
Post.joins(:user).where(users: { active: true })
```

### ใช้ Bullet gem ตรวจหา N+1

```ruby
# Gemfile
group :development do
  gem "bullet"
end

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
end
```

---

## Step 843: Counter Cache

```ruby
# Migration - เพิ่ม counter cache column
class AddCommentsCountToPosts < ActiveRecord::Migration[7.0]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false
    
    # Reset counter cache สำหรับ existing data
    Post.find_each do |post|
      Post.reset_counters(post.id, :comments)
    end
  end
end

# Model
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
  # counter_cache: true => ใช้ posts.comments_count
  
  # หรือ custom column name
  belongs_to :post, counter_cache: :total_comments
end

class Post < ApplicationRecord
  has_many :comments
  # posts.comments_count อัปเดตอัตโนมัติ
end

# ใช้งาน
post = Post.find(1)
post.comments_count  # => 5 (อ่านจาก column)
post.comments.count  # => 5 (รัน SQL COUNT)

# Reset counter (ถ้า count ไม่ sync)
Post.reset_counters(post.id, :comments)
Post.all.each { |p| Post.reset_counters(p.id, :comments) }

# Bulk reset
Post.find_each { |p| Post.reset_counters(p.id, :comments) }
```

---

## Step 844: Touch Option

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true
  # เมื่อ comment ถูก create/update/destroy
  # จะ update post.updated_at อัตโนมัติ
  
  belongs_to :post, touch: :comments_updated_at
  # อัปเดต post.comments_updated_at แทน
end

# ใช้งาน
comment = Comment.find(1)
post = comment.post
post.updated_at  # => เก่า

comment.update!(content: "Updated")
post.reload
post.updated_at  # => ใหม่ (touched!)

# Manual touch
post.touch                  # อัปเดต updated_at
post.touch(:published_at)  # อัปเดต published_at
```

---

## ตัวอย่าง Associations ที่สมบูรณ์

```ruby
# Blog system
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy
  has_many :likes, dependent: :destroy
  has_one :profile, dependent: :destroy
  
  # Indirect associations
  has_many :post_comments, through: :posts, source: :comments
  has_many :liked_posts, through: :likes, source: :likeable, 
           source_type: "Post"
  
  # Followers/Following (self-referential)
  has_many :active_relationships, foreign_key: "follower_id",
           class_name: "Follow", dependent: :destroy
  has_many :passive_relationships, foreign_key: "followed_id",
           class_name: "Follow", dependent: :destroy
  has_many :following, through: :active_relationships, source: :followed
  has_many :followers, through: :passive_relationships, source: :follower
  
  def follow(other_user)
    following << other_user unless following?(other_user)
  end
  
  def unfollow(other_user)
    following.delete(other_user)
  end
  
  def following?(other_user)
    following.include?(other_user)
  end
end

class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
  belongs_to :category, optional: true
  has_many :comments, as: :commentable, dependent: :destroy
  has_many :likes, as: :likeable, dependent: :destroy
  has_many :taggings, dependent: :destroy
  has_many :tags, through: :taggings
  has_one_attached :cover_image
  has_many_attached :images
  
  scope :with_associations, -> {
    includes(:user, :category, :tags)
  }
end

class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :commentable, polymorphic: true, touch: true
  has_many :likes, as: :likeable, dependent: :destroy
  has_many :replies, class_name: "Comment", foreign_key: "parent_id",
           dependent: :destroy
  belongs_to :parent, class_name: "Comment", optional: true
end

class Like < ApplicationRecord
  belongs_to :user
  belongs_to :likeable, polymorphic: true, touch: true
  
  validates :user_id, uniqueness: { scope: [:likeable_type, :likeable_id] }
end

class Tag < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :posts, through: :taggings
end

class Tagging < ApplicationRecord
  belongs_to :tag, counter_cache: true
  belongs_to :post, touch: true
end

class Category < ApplicationRecord
  belongs_to :parent, class_name: "Category", optional: true
  has_many :children, class_name: "Category", foreign_key: "parent_id"
  has_many :posts, dependent: :nullify
  
  scope :roots, -> { where(parent_id: nil) }
end
```

---

## แบบฝึกหัด (Steps 845-860)

### แบบฝึกหัดที่ 1
สร้าง User has_many Posts และ Post belongs_to User

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
end

class Post < ApplicationRecord
  belongs_to :user
end
```

### แบบฝึกหัดที่ 2
สร้าง User has_one Profile

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_one :profile, dependent: :destroy
end

class Profile < ApplicationRecord
  belongs_to :user
end
```

### แบบฝึกหัดที่ 3
สร้าง Post has_many Tags ผ่าน Taggings (has_many :through)

**เฉลย:**
```ruby
class Post < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :tags, through: :taggings
end

class Tagging < ApplicationRecord
  belongs_to :post
  belongs_to :tag
end

class Tag < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :posts, through: :taggings
end
```

### แบบฝึกหัดที่ 4
implement polymorphic Comments สำหรับ Post และ Photo

**เฉลย:**
```ruby
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
  belongs_to :user
end

class Post < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

class Photo < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end
```

### แบบฝึกหัดที่ 5
สร้าง self-referential association สำหรับ Category (tree structure)

**เฉลย:**
```ruby
class Category < ApplicationRecord
  belongs_to :parent, class_name: "Category", optional: true
  has_many :children, class_name: "Category", foreign_key: "parent_id",
           dependent: :destroy
  
  scope :roots, -> { where(parent_id: nil) }
  
  def root?
    parent_id.nil?
  end
end
```

### แบบฝึกหัดที่ 6
สร้าง User followers/following ด้วย self-referential

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_many :active_relationships, class_name: "Follow",
           foreign_key: :follower_id, dependent: :destroy
  has_many :following, through: :active_relationships, source: :followed
  
  has_many :passive_relationships, class_name: "Follow",
           foreign_key: :followed_id, dependent: :destroy
  has_many :followers, through: :passive_relationships, source: :follower
end

class Follow < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"
end
```

### แบบฝึกหัดที่ 7
implement counter_cache สำหรับ posts_count ใน User

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
```

### แบบฝึกหัดที่ 8
แก้ไข N+1 problem ใน posts#index

**เฉลย:**
```ruby
# ❌ N+1
@posts = Post.all

# ✅ แก้ไข
@posts = Post.includes(:user, :category, :tags)
```

### แบบฝึกหัดที่ 9
สร้าง Post has_many :through User (ผ่าน Favorites)

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_many :favorites, dependent: :destroy
  has_many :favorite_posts, through: :favorites, source: :post
end

class Favorite < ApplicationRecord
  belongs_to :user
  belongs_to :post
end

class Post < ApplicationRecord
  has_many :favorites, dependent: :destroy
  has_many :favorited_by, through: :favorites, source: :user
end
```

### แบบฝึกหัดที่ 10
ใช้ eager_load สำหรับ filter บน association

**เฉลย:**
```ruby
# ดึง posts ที่ author active
posts = Post.eager_load(:user)
            .where(users: { active: true, role: "admin" })
            .order("posts.created_at DESC")
```

### แบบฝึกหัดที่ 11
สร้าง association ที่ custom foreign_key และ class_name

**เฉลย:**
```ruby
class Post < ApplicationRecord
  belongs_to :author, class_name: "User", foreign_key: "author_id"
  belongs_to :editor, class_name: "User", foreign_key: "editor_id", optional: true
  
  has_many :revisions, class_name: "PostRevision", foreign_key: "article_id"
end
```

### แบบฝึกหัดที่ 12
implement touch option เพื่ออัปเดต post.updated_at เมื่อ comment เปลี่ยน

**เฉลย:**
```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end
```

### แบบฝึกหัดที่ 13
สร้าง has_many :through ที่มี conditions

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_many :enrollments
  has_many :active_courses, -> { where(enrollments: { active: true }) },
           through: :enrollments, source: :course
  has_many :completed_courses, -> { where(enrollments: { completed: true }) },
           through: :enrollments, source: :course
end
```

### แบบฝึกหัดที่ 14
ใช้ includes ในหลาย levels

**เฉลย:**
```ruby
# ดึง posts พร้อม user, comments, และ user ของ comments
@posts = Post.includes(:user, comments: :user)

# ดึง order พร้อม items, products, และ images ของ products
@orders = Order.includes(items: { product: :images })
```

### แบบฝึกหัดที่ 15
สร้าง Doctor-Patient many-to-many ผ่าน Appointment

**เฉลย:**
```ruby
class Doctor < ApplicationRecord
  has_many :appointments
  has_many :patients, through: :appointments
end

class Patient < ApplicationRecord
  has_many :appointments
  has_many :doctors, through: :appointments
end

class Appointment < ApplicationRecord
  belongs_to :doctor
  belongs_to :patient
  
  validates :appointment_date, presence: true
  validates :doctor_id, uniqueness: { scope: :patient_id,
    message: "Doctor already has appointment with this patient" }
end
```

### แบบฝึกหัดที่ 16
implement HABTM สำหรับ Post และ Tag

**เฉลย:**
```ruby
class Post < ApplicationRecord
  has_and_belongs_to_many :tags
end

class Tag < ApplicationRecord
  has_and_belongs_to_many :posts
end

# Migration
class CreatePostsTags < ActiveRecord::Migration[7.0]
  def change
    create_table :posts_tags, id: false do |t|
      t.bigint :post_id, null: false
      t.bigint :tag_id, null: false
    end
    add_index :posts_tags, [:post_id, :tag_id], unique: true
  end
end
```

### แบบฝึกหัดที่ 17
สร้าง has_many association ที่ extend ด้วย custom methods

**เฉลย:**
```ruby
class Post < ApplicationRecord
  has_many :comments do
    def spam
      where(spam: true)
    end
    
    def by_user(user)
      where(user: user)
    end
    
    def recent(limit = 5)
      order(created_at: :desc).limit(limit)
    end
  end
end

# ใช้งาน
post.comments.spam
post.comments.by_user(current_user)
post.comments.recent(10)
```

### แบบฝึกหัดที่ 18
implement dependent: :restrict_with_error

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_many :posts, dependent: :restrict_with_error
  
  # ถ้า user มี posts จะ add error และไม่ให้ลบ
  # user.destroy => false
  # user.errors[:base] => ["Cannot delete record because dependent posts exist"]
end
```

### แบบฝึกหัดที่ 19
เขียน test สำหรับ User has_many Posts association

**เฉลย:**
```ruby
# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  describe "associations" do
    it { is_expected.to have_many(:posts).dependent(:destroy) }
    it { is_expected.to have_one(:profile) }
  end
  
  describe "#posts" do
    let(:user) { create(:user) }
    
    it "can have multiple posts" do
      create_list(:post, 3, user: user)
      expect(user.posts.count).to eq(3)
    end
    
    it "destroys posts when user is destroyed" do
      create_list(:post, 2, user: user)
      expect { user.destroy }.to change(Post, :count).by(-2)
    end
  end
end
```

### แบบฝึกหัดที่ 20
สร้าง polymorphic Like system

**เฉลย:**
```ruby
class Like < ApplicationRecord
  belongs_to :user
  belongs_to :likeable, polymorphic: true
  
  validates :user_id, uniqueness: {
    scope: [:likeable_type, :likeable_id],
    message: "ไม่สามารถกดถูกใจซ้ำได้"
  }
end

class Post < ApplicationRecord
  has_many :likes, as: :likeable, dependent: :destroy
  
  def liked_by?(user)
    likes.exists?(user: user)
  end
  
  def like_count
    likes.count
  end
end

class Comment < ApplicationRecord
  has_many :likes, as: :likeable, dependent: :destroy
end
```

### แบบฝึกหัดที่ 21
ใช้ association scope สำหรับ published posts เท่านั้น

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_many :posts
  has_many :published_posts, -> { where(status: :published) }, class_name: "Post"
  has_many :draft_posts, -> { where(status: :draft) }, class_name: "Post"
end

# ใช้งาน
user.published_posts  # เฉพาะ published
user.draft_posts      # เฉพาะ drafts
```

### แบบฝึกหัดที่ 22
implement Order has_many OrderItems และ total calculation

**เฉลย:**
```ruby
class Order < ApplicationRecord
  has_many :order_items, dependent: :destroy
  has_many :products, through: :order_items
  
  def subtotal
    order_items.sum { |item| item.unit_price * item.quantity }
  end
  
  def recalculate_total!
    update!(total: subtotal + tax + shipping - discount)
  end
end

class OrderItem < ApplicationRecord
  belongs_to :order, touch: true
  belongs_to :product
  
  before_create :set_unit_price
  after_save :update_order_total
  
  private
  
  def set_unit_price
    self.unit_price = product.price
    self.total_price = unit_price * quantity
  end
  
  def update_order_total
    order.recalculate_total!
  end
end
```

### แบบฝึกหัดที่ 23
สร้าง Company has_many Employees ที่มี different roles

**เฉลย:**
```ruby
class Company < ApplicationRecord
  has_many :employees, dependent: :destroy
  has_many :managers, -> { where(role: "manager") }, class_name: "Employee"
  has_many :developers, -> { where(role: "developer") }, class_name: "Employee"
  
  def employee_count
    employees.count
  end
end

class Employee < ApplicationRecord
  belongs_to :company
  
  scope :by_role, ->(role) { where(role: role) }
  scope :managers, -> { where(role: "manager") }
end
```

### แบบฝึกหัดที่ 24
implement cascade delete ผ่าน dependent: :destroy

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
  
  # เมื่อ user ถูกลบ:
  # 1. posts ถูกลบ (รัน callbacks)
  # 2. ถ้า post has_many :comments, dependent: :destroy
  #    comments ก็ถูกลบด้วย
end

class Post < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :likes, dependent: :destroy
end
```

### แบบฝึกหัดที่ 25
เขียน integration test สำหรับ N+1 detection

**เฉลย:**
```ruby
# spec/models/post_spec.rb
RSpec.describe Post, type: :model do
  describe "eager loading" do
    before do
      create_list(:post, 5, :with_user_and_tags)
    end
    
    it "does not have N+1 queries" do
      query_count = 0
      counter = ->(*, **) { query_count += 1 }
      
      ActiveSupport::Notifications.subscribed(counter, "sql.active_record") do
        Post.includes(:user, :tags).each do |post|
          post.user.name
          post.tags.each(&:name)
        end
      end
      
      # 1 query for posts + 1 for users + 1 for taggings + 1 for tags = 4
      expect(query_count).to be <= 4
    end
  end
end
```

### แบบฝึกหัดที่ 26
สร้าง Product belongs_to Category ที่ optional

**เฉลย:**
```ruby
class Product < ApplicationRecord
  belongs_to :category, optional: true
  
  scope :uncategorized, -> { where(category_id: nil) }
  scope :in_category, ->(cat) { where(category: cat) if cat }
end
```

### แบบฝึกหัดที่ 27
ใช้ autosave option สำหรับ nested attributes

**เฉลย:**
```ruby
class Order < ApplicationRecord
  has_many :order_items, autosave: true
  accepts_nested_attributes_for :order_items, allow_destroy: true
  
  # เมื่อ save Order จะ save OrderItems ด้วยอัตโนมัติ
end
```

### แบบฝึกหัดที่ 28
implement inverse_of เพื่อ avoid loading ซ้ำ

**เฉลย:**
```ruby
class Post < ApplicationRecord
  has_many :comments, inverse_of: :post
  belongs_to :user, inverse_of: :posts
end

class Comment < ApplicationRecord
  belongs_to :post, inverse_of: :comments
end

# ประโยชน์: ป้องกัน loading ซ้ำ
post = Post.first
post.comments.each do |comment|
  comment.post == post  # true, ไม่ต้อง reload
  comment.post.object_id == post.object_id  # true! same object
end
```

### แบบฝึกหัดที่ 29
สร้าง Book has_many Authors ผ่าน Authorship (many-to-many with extra data)

**เฉลย:**
```ruby
class Book < ApplicationRecord
  has_many :authorships, dependent: :destroy
  has_many :authors, through: :authorships
  
  def primary_author
    authorships.primary.first&.author
  end
end

class Authorship < ApplicationRecord
  belongs_to :book
  belongs_to :author
  
  enum role: { primary: 0, secondary: 1, editor: 2 }
  
  scope :primary, -> { where(role: :primary) }
end

class Author < ApplicationRecord
  has_many :authorships, dependent: :destroy
  has_many :books, through: :authorships
end
```

### แบบฝึกหัดที่ 30
เขียน method ที่ return users ที่ไม่มี posts

**เฉลย:**
```ruby
class User < ApplicationRecord
  has_many :posts
  
  def self.without_posts
    left_joins(:posts).where(posts: { id: nil })
  end
  
  # หรือ
  def self.without_posts
    where.not(id: Post.select(:user_id))
  end
end
```

---

## สรุป

ใน Associations เราได้เรียนรู้:

1. **belongs_to** - foreign key อยู่ที่ตัวเอง
2. **has_many** - มี children หลายตัว
3. **has_one** - มี child หนึ่งตัว
4. **has_many :through** - many-to-many ผ่าน join model
5. **has_one :through** - one-to-one ผ่าน model อื่น
6. **HABTM** - many-to-many ไม่มี model กลาง
7. **Polymorphic** - association กับหลาย models
8. **Self-referential** - association กับตัวเอง
9. **Association options** - dependent, class_name, foreign_key, touch
10. **Association methods** - build, create, delete, destroy
11. **Eager loading** - includes, preload, eager_load
12. **N+1 problem** - ปัญหาและวิธีแก้ไข
13. **Counter cache** - นับ children โดยไม่ต้อง query
14. **Touch option** - อัปเดต parent timestamp
