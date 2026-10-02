# Part 38: Associations (ขั้นตอนที่ 831-860)

## บทนำ

Associations คือความสัมพันธ์ระหว่าง models Active Record รองรับ association types หลากหลาย ทำให้เราสามารถแสดง relationships ระหว่าง objects ได้อย่างสมจริงและทำงานกับ related data ได้ง่าย

---

## ขั้นตอนที่ 831: belongs_to

### belongs_to คืออะไร?

`belongs_to` ใช้เมื่อ model หนึ่งมี foreign key อ้างไปยัง model อื่น

```ruby
# Migration: articles table มี user_id column
class CreateArticles < ActiveRecord::Migration[7.1]
  def change
    create_table :articles do |t|
      t.string :title
      t.references :user, null: false, foreign_key: true  # เพิ่ม user_id column
      t.timestamps
    end
  end
end

# Model
class Article < ApplicationRecord
  belongs_to :user

  # หรือกับ options:
  belongs_to :user, optional: true           # ไม่บังคับต้องมี user
  belongs_to :author, class_name: "User"     # custom class name
  belongs_to :category, counter_cache: true  # counter cache
  belongs_to :parent, class_name: "Article", optional: true  # self-referential
end

# Methods ที่ได้จาก belongs_to:
article = Article.find(1)
article.user          # ดึง User object
article.user_id       # ดึง foreign key value
article.user = user   # assign user
article.build_user(name: "Test")  # build user without save
article.create_user!(name: "Test") # create user with save

# Validation:
# belongs_to ใน Rails 5+ จะ validate ว่า user ต้องมีอยู่จริง
# article.user_id = 999  # user ที่ไม่มีอยู่
# article.valid?  # => false
# article.errors[:user]  # => ["must exist"]
```

### belongs_to Options

```ruby
class Article < ApplicationRecord
  # class_name - กำหนด class name ที่ต่างจาก convention
  belongs_to :author, class_name: "User", foreign_key: "user_id"

  # foreign_key - กำหนด foreign key column
  belongs_to :writer, class_name: "User", foreign_key: "writer_user_id"

  # primary_key - กำหนด primary key ของ associated model
  belongs_to :user, primary_key: "uuid"

  # optional - ไม่บังคับมี association
  belongs_to :category, optional: true

  # counter_cache - increment/decrement counter
  belongs_to :user, counter_cache: true  # เพิ่ม articles_count ใน users
  belongs_to :user, counter_cache: :published_articles_count  # custom counter

  # touch - update timestamps ของ parent เมื่อ save
  belongs_to :article, touch: true
  belongs_to :article, touch: :last_commented_at

  # validate - validate association (default true ใน Rails 5+)
  belongs_to :user, validate: true

  # dependent - behavior เมื่อ associated object ถูกลบ
  # (ไม่ค่อยใช้ใน belongs_to)
  belongs_to :user, dependent: :destroy  # ถ้า user ถูกลบ จะลบ article ด้วย

  # inverse_of - ระบุ inverse association
  belongs_to :user, inverse_of: :articles
end
```

---

## ขั้นตอนที่ 832: has_many

### has_many คืออะไร?

`has_many` ใช้เมื่อ model หนึ่งมี model อื่นหลายตัว

```ruby
class User < ApplicationRecord
  has_many :articles
  has_many :comments
  has_many :likes
end

# Methods ที่ได้จาก has_many:
user = User.find(1)
user.articles                    # ดึง articles ทั้งหมด (SQL query)
user.articles.count              # COUNT query
user.articles.build(title: "x") # build article without save
user.articles.create!(title: "x", body: "content") # create and save
user.articles.where(status: "published")  # scope on association
user.articles.find(1)           # หา article by id (ต้องเป็นของ user)
user.articles.first
user.articles.last
user.articles.any?
user.articles.empty?
user.articles << article        # append article
user.articles.delete(article)   # remove association
user.articles.destroy(article)  # destroy article
user.articles.clear             # remove all
user.articles.reload            # reload from database
```

### has_many Options

```ruby
class User < ApplicationRecord
  # dependent - behavior เมื่อ user ถูกลบ
  has_many :articles, dependent: :destroy   # ลบ articles ด้วย (runs callbacks)
  has_many :articles, dependent: :delete_all # ลบด้วย SQL (เร็วกว่า, ไม่ run callbacks)
  has_many :articles, dependent: :nullify   # set user_id เป็น NULL
  has_many :articles, dependent: :restrict_with_exception  # raise error
  has_many :articles, dependent: :restrict_with_error     # add error to model

  # class_name - กำหนด class
  has_many :published_articles, class_name: "Article"

  # foreign_key - กำหนด foreign key
  has_many :articles, foreign_key: "author_id"

  # primary_key - กำหนด primary key
  has_many :articles, primary_key: "uuid"

  # scope - เพิ่ม default scope
  has_many :published_articles, -> { where(status: "published") }, class_name: "Article"
  has_many :recent_articles, -> { order(created_at: :desc).limit(5) }, class_name: "Article"
  has_many :articles, -> { where(deleted_at: nil) }

  # inverse_of - specify inverse
  has_many :articles, inverse_of: :user

  # through - ผ่าน join model
  has_many :comments, through: :articles

  # source - กำหนด source สำหรับ through
  has_many :commenters, through: :articles, source: :user

  # as - polymorphic
  has_many :images, as: :imageable
end
```

---

## ขั้นตอนที่ 833: has_one

### has_one คืออะไร?

`has_one` ใช้เมื่อ model หนึ่งมีอีก model เดียว (1-to-1 จากฝั่ง parent)

```ruby
class User < ApplicationRecord
  has_one :profile
  has_one :setting
  has_one :subscription
end

class Profile < ApplicationRecord
  belongs_to :user
end

# Methods:
user.profile          # ดึง Profile
user.profile = profile  # assign
user.build_profile(bio: "test")   # build without save
user.create_profile!(bio: "test") # create and save
user.profile&.destroy             # ลบ profile
```

### has_one Options

```ruby
class User < ApplicationRecord
  has_one :profile, dependent: :destroy
  has_one :profile, class_name: "UserProfile"
  has_one :profile, foreign_key: "account_id"

  # scope
  has_one :latest_subscription, -> { order(created_at: :desc) }, class_name: "Subscription"

  # through
  has_one :address, through: :profile
end
```

---

## ขั้นตอนที่ 834: has_many :through

### has_many :through

ใช้สำหรับ many-to-many ที่มี join model ที่มี attributes เพิ่มเติม

```ruby
# Models:
class Article < ApplicationRecord
  has_many :article_tags
  has_many :tags, through: :article_tags
end

class Tag < ApplicationRecord
  has_many :article_tags
  has_many :articles, through: :article_tags
end

class ArticleTag < ApplicationRecord
  belongs_to :article
  belongs_to :tag
  
  # join model สามารถมี attributes เพิ่มเติม
  # t.integer :position
  # t.boolean :featured
end

# Migration:
create_table :article_tags do |t|
  t.references :article, null: false, foreign_key: true
  t.references :tag, null: false, foreign_key: true
  t.timestamps
end
add_index :article_tags, [:article_id, :tag_id], unique: true

# การใช้งาน:
article = Article.find(1)
article.tags                  # ดึง tags ทั้งหมด
article.tags.create!(name: "Rails")  # สร้าง tag ใหม่และ associate
article.tag_ids               # ดึง array ของ tag IDs
article.tag_ids = [1, 2, 3]  # assign tags ด้วย IDs
article.tags << Tag.find(1)  # เพิ่ม tag
article.tags.delete(tag)     # ลบ association (ไม่ลบ tag จริง)
```

### has_many :through กับ Business Logic

```ruby
# ตัวอย่าง: User follows Article
class User < ApplicationRecord
  has_many :follows
  has_many :followed_articles, through: :follows, source: :article

  has_many :written_articles, class_name: "Article"
end

class Article < ApplicationRecord
  has_many :follows
  has_many :followers, through: :follows, source: :user
end

class Follow < ApplicationRecord
  belongs_to :user
  belongs_to :article

  # Extra attributes:
  # t.string :reason    # ทำไม follow
  # t.boolean :notify   # แจ้งเตือนหรือเปล่า
end

# ตัวอย่าง: Order with LineItems
class Order < ApplicationRecord
  has_many :order_items
  has_many :products, through: :order_items

  def total
    order_items.sum { |item| item.quantity * item.unit_price }
  end
end

class OrderItem < ApplicationRecord
  belongs_to :order
  belongs_to :product

  # attributes: quantity, unit_price, discount
end

class Product < ApplicationRecord
  has_many :order_items
  has_many :orders, through: :order_items
end
```

---

## ขั้นตอนที่ 835: has_one :through

### has_one :through

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

# ใช้งาน:
user.account_history  # User → Account → AccountHistory

# ตัวอย่าง Real-world:
class Customer < ApplicationRecord
  has_one :order, -> { order(created_at: :desc) }
  has_one :latest_purchase, through: :order, source: :product
end

class Doctor < ApplicationRecord
  has_many :appointments
  has_many :patients, through: :appointments
  has_one :first_patient, through: :appointments, source: :patient
end
```

---

## ขั้นตอนที่ 836: has_and_belongs_to_many (HABTM)

### HABTM - Simple Many-to-Many

```ruby
# HABTM ใช้สำหรับ many-to-many ที่ join table ไม่มี attributes เพิ่มเติม
# (ปัจจุบันแนะนำให้ใช้ has_many :through แทน - ยืดหยุ่นกว่า)

class Article < ApplicationRecord
  has_and_belongs_to_many :tags
end

class Tag < ApplicationRecord
  has_and_belongs_to_many :articles
end

# Migration: ต้องสร้าง join table
class CreateArticlesTags < ActiveRecord::Migration[7.1]
  def change
    create_join_table :articles, :tags do |t|
      t.index [:article_id, :tag_id]
      t.index [:tag_id, :article_id]
    end
  end
end

# การใช้งาน:
article.tags                    # ดึง tags
article.tags << Tag.find(1)    # เพิ่ม tag
article.tags.delete(tag)       # ลบ association
article.tags = [tag1, tag2]    # set tags
article.tag_ids = [1, 2, 3]   # set by IDs

tag.articles                   # ดึง articles
```

### HABTM vs has_many :through

```ruby
# HABTM:
class Article < ApplicationRecord
  has_and_belongs_to_many :tags
end

# has_many :through (แนะนำ):
class Article < ApplicationRecord
  has_many :article_tags
  has_many :tags, through: :article_tags
end

# เหตุผลที่ prefer has_many :through:
# 1. สามารถเพิ่ม attributes ใน join model ได้
# 2. สามารถ validate join model ได้
# 3. สามารถ create/update join model ได้
# 4. Rails documentation แนะนำ
```

---

## ขั้นตอนที่ 837: Polymorphic Associations

### Polymorphic คืออะไร?

Polymorphic ทำให้ model หนึ่ง belong_to หลาย models ด้วย single association

```ruby
# Migration:
class CreateComments < ActiveRecord::Migration[7.1]
  def change
    create_table :comments do |t|
      t.text :body, null: false
      t.references :commentable, polymorphic: true, null: false
      # เพิ่ม 2 columns: commentable_type (string) และ commentable_id (bigint)
      t.references :user, null: false, foreign_key: true
      t.timestamps
    end
    add_index :comments, [:commentable_type, :commentable_id]
  end
end

# Comment model
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
  belongs_to :user

  # commentable_type: "Article", "Photo", "Video"
  # commentable_id: 1, 2, 3, ...
end

# Article model
class Article < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

# Photo model
class Photo < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

# Video model
class Video < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

# การใช้งาน:
article = Article.find(1)
article.comments.create!(body: "Great article!", user: current_user)

photo = Photo.find(1)
photo.comments.create!(body: "Beautiful photo!", user: current_user)

comment = Comment.find(1)
comment.commentable  # ดึง Article หรือ Photo หรือ Video
comment.commentable_type  # => "Article"
comment.commentable_id    # => 1
```

### Real-world Polymorphic Examples

```ruby
# Likes - ทุก resource สามารถถูก like ได้
class Like < ApplicationRecord
  belongs_to :likeable, polymorphic: true
  belongs_to :user
end

class Article < ApplicationRecord
  has_many :likes, as: :likeable
  def liked_by?(user)
    likes.where(user: user).exists?
  end
end

class Comment < ApplicationRecord
  has_many :likes, as: :likeable
end

# Images - หลาย models มี images
class Image < ApplicationRecord
  belongs_to :imageable, polymorphic: true
  has_one_attached :file
end

class User < ApplicationRecord
  has_many :images, as: :imageable
  has_one :avatar, -> { where(role: "avatar") }, class_name: "Image", as: :imageable
end

class Product < ApplicationRecord
  has_many :images, as: :imageable
end

# Addresses - หลาย models มี addresses
class Address < ApplicationRecord
  belongs_to :addressable, polymorphic: true
end

class User < ApplicationRecord
  has_one :billing_address, -> { where(type: "billing") },
    class_name: "Address", as: :addressable
  has_one :shipping_address, -> { where(type: "shipping") },
    class_name: "Address", as: :addressable
end

class Company < ApplicationRecord
  has_one :address, as: :addressable
end

# Notifications - polymorphic source
class Notification < ApplicationRecord
  belongs_to :notifiable, polymorphic: true
  belongs_to :recipient, class_name: "User"

  # source types: "Article", "Comment", "Follow", etc.
end

class Article < ApplicationRecord
  has_many :notifications, as: :notifiable
end

class Comment < ApplicationRecord
  has_many :notifications, as: :notifiable
  after_create :notify_article_author

  private

  def notify_article_author
    Notification.create!(
      notifiable: self,
      recipient: article.user,
      message: "#{user.name} commented on your article"
    )
  end
end
```

---

## ขั้นตอนที่ 838: Self-Referential Associations

### Self-Referential

```ruby
# Categories กับ subcategories
class Category < ApplicationRecord
  belongs_to :parent, class_name: "Category", optional: true
  has_many :children, class_name: "Category", foreign_key: "parent_id", dependent: :destroy

  # Navigation
  def root?
    parent_id.nil?
  end

  def leaf?
    children.empty?
  end

  def ancestors
    node = self
    result = []
    while node.parent.present?
      result.unshift(node.parent)
      node = node.parent
    end
    result
  end

  def descendants
    result = []
    children.each do |child|
      result << child
      result.concat(child.descendants)
    end
    result
  end

  def self.tree
    where(parent_id: nil).includes(children: [:children])
  end
end

# ใช้งาน:
tech = Category.create!(name: "Technology")
rails = Category.create!(name: "Rails", parent: tech)
models = Category.create!(name: "Models", parent: rails)

models.parent        # => rails
models.ancestors     # => [tech, rails]
tech.children        # => [rails]
tech.descendants     # => [rails, models]
```

### User Follows User

```ruby
class User < ApplicationRecord
  has_many :active_follows, class_name: "Follow",
    foreign_key: "follower_id", dependent: :destroy
  has_many :passive_follows, class_name: "Follow",
    foreign_key: "followed_id", dependent: :destroy

  has_many :following, through: :active_follows, source: :followed
  has_many :followers, through: :passive_follows, source: :follower

  def follow!(other_user)
    active_follows.create!(followed: other_user) unless following?(other_user)
  end

  def unfollow!(other_user)
    active_follows.find_by(followed: other_user)&.destroy
  end

  def following?(other_user)
    following.include?(other_user)
  end

  def followed_by?(other_user)
    followers.include?(other_user)
  end
end

class Follow < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"

  validates :follower_id, uniqueness: { scope: :followed_id }
  validate :cannot_follow_self

  private

  def cannot_follow_self
    errors.add(:base, "ไม่สามารถ follow ตัวเองได้") if follower_id == followed_id
  end
end

# Migration:
create_table :follows do |t|
  t.references :follower, null: false, foreign_key: { to_table: :users }
  t.references :followed, null: false, foreign_key: { to_table: :users }
  t.timestamps
end
add_index :follows, [:follower_id, :followed_id], unique: true
```

---

## ขั้นตอนที่ 839: Association Methods และ Options

### Association Methods

```ruby
class Article < ApplicationRecord
  has_many :comments, dependent: :destroy
  belongs_to :user
end

# Collection methods (has_many):
@article.comments             # SELECT * FROM comments WHERE article_id = X
@article.comments.count       # SELECT COUNT(*) FROM comments WHERE article_id = X
@article.comments.size        # ใช้ counter_cache ถ้ามี
@article.comments.length      # load ทั้งหมดแล้ว count (ไม่แนะนำ)
@article.comments.empty?      # true/false
@article.comments.any?        # true/false
@article.comments.many?       # มีมากกว่า 1
@article.comments.include?(comment)  # ตรวจสอบว่ามีหรือเปล่า

# Modification:
@article.comments.build(body: "test")   # new comment ที่ link กับ article (ไม่ save)
@article.comments.create(body: "test")  # save ทันที
@article.comments.create!(body: "test") # save + raise exception
@article.comments << new_comment        # append
@article.comments.concat(comments_arr)  # append หลายตัว
@article.comments.push(new_comment)     # เหมือน <<

@article.comments.delete(comment)       # ลบ association
@article.comments.destroy(comment)      # destroy object + association
@article.comments.clear                 # ลบ associations ทั้งหมด
@article.comments.delete_all            # DELETE SQL (ไม่ run callbacks)
@article.comments.destroy_all           # destroy แต่ละ object

# Querying:
@article.comments.where(approved: true)
@article.comments.order(created_at: :desc)
@article.comments.includes(:user)
@article.comments.limit(5)
@article.comments.find(1)
@article.comments.find_by(user: current_user)
```

---

## ขั้นตอนที่ 840: Eager Loading (N+1 Problem)

### N+1 Problem

```ruby
# ❌ N+1 Problem
# 1 query สำหรับ articles + N queries สำหรับแต่ละ user
@articles = Article.all
@articles.each do |article|
  puts article.user.name  # Query ใหม่ทุกครั้ง!
end
# SQL:
# SELECT * FROM articles;
# SELECT * FROM users WHERE id = 1;
# SELECT * FROM users WHERE id = 2;
# SELECT * FROM users WHERE id = 3;
# ... (N queries!)

# ✅ แก้ด้วย includes
@articles = Article.includes(:user)
@articles.each do |article|
  puts article.user.name  # ใช้ data ที่ preloaded แล้ว
end
# SQL:
# SELECT * FROM articles;
# SELECT * FROM users WHERE id IN (1, 2, 3);  # เพียง 2 queries!

# ตรวจสอบ N+1 ด้วย Bullet gem
# config/environments/development.rb
# config.after_initialize do
#   Bullet.enable = true
#   Bullet.rails_logger = true
#   Bullet.add_footer = true
# end
```

### includes vs preload vs eager_load

```ruby
# 1. includes - อัจฉริยะ เลือกระหว่าง preload/eager_load อัตโนมัติ
Article.includes(:user)
Article.includes(:user, :tags, :comments)
Article.includes(comments: :user)  # nested
Article.includes(:user).where("users.admin = ?", true).references(:users)

# 2. preload - ใช้ separate queries เสมอ
Article.preload(:user)
# SQL:
# SELECT * FROM articles
# SELECT * FROM users WHERE id IN (...)

# 3. eager_load - ใช้ LEFT OUTER JOIN เสมอ
Article.eager_load(:user)
# SQL:
# SELECT articles.*, users.* FROM articles
#   LEFT OUTER JOIN users ON users.id = articles.user_id

# เมื่อไหร่ใช้อะไร:
# includes: ใช้ทั่วไป (Rails เลือกให้)
# preload: เมื่อต้องการ separate queries แน่นอน
# eager_load: เมื่อต้องการ WHERE/ORDER บน joined table

# includes + where บน association ต้องใช้ references:
Article.includes(:user)
       .where("users.role = ?", "admin")
       .references(:users)
# หรือ:
Article.eager_load(:user).where(users: { role: "admin" })
```

### ตัวอย่าง Complex Eager Loading

```ruby
# หน้า article show ที่ต้องการข้อมูลหลายส่วน
@article = Article.includes(
  :user,
  :category,
  :tags,
  comments: { user: :profile }  # nested includes
).find(params[:id])

# Dashboard ที่ต้องการ stats
@users = User.includes(
  :profile,
  articles: [:tags, :comments],
  follows: :followed
).where(role: "author").limit(20)

# Homepage ที่ต้องการ recent popular articles
@articles = Article.published
                   .includes(:user, :tags, :category)
                   .preload(:comments)  # ต้องการ count
                   .order(published_at: :desc)
                   .limit(10)
```

---

## ขั้นตอนที่ 841-860: Association Best Practices

### Full Blog Application Associations

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # Authored content
  has_many :articles, dependent: :destroy
  has_many :comments, dependent: :destroy
  has_many :likes, dependent: :destroy
  has_many :bookmarks, dependent: :destroy

  # Published content
  has_many :published_articles,
    -> { published },
    class_name: "Article"

  # Liked articles
  has_many :liked_articles, through: :likes, source: :likeable,
    source_type: "Article"

  # Social
  has_many :active_follows, class_name: "Follow",
    foreign_key: :follower_id, dependent: :destroy
  has_many :passive_follows, class_name: "Follow",
    foreign_key: :followed_id, dependent: :destroy
  has_many :following, through: :active_follows, source: :followed
  has_many :followers, through: :passive_follows, source: :follower

  # Profile
  has_one :profile, dependent: :destroy
  has_one :setting, dependent: :destroy

  # Notifications
  has_many :notifications, foreign_key: :recipient_id, dependent: :destroy
  has_many :unread_notifications, -> { unread },
    class_name: "Notification", foreign_key: :recipient_id
end

# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :user, counter_cache: :articles_count
  belongs_to :category, optional: true, counter_cache: true

  has_many :article_tags, dependent: :destroy
  has_many :tags, through: :article_tags

  has_many :comments, as: :commentable, dependent: :destroy
  has_many :comment_users, through: :comments, source: :user

  has_many :likes, as: :likeable, dependent: :destroy
  has_many :bookmarks, dependent: :destroy

  has_one :seo_meta, dependent: :destroy, autosave: true
  has_one_attached :featured_image
  has_many_attached :attachments

  # Nested attributes
  accepts_nested_attributes_for :seo_meta, allow_destroy: true
  accepts_nested_attributes_for :tags,
    reject_if: :all_blank,
    allow_destroy: true
end

# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
  belongs_to :user

  has_many :likes, as: :likeable, dependent: :destroy
  has_many :replies, class_name: "Comment",
    foreign_key: :parent_id, dependent: :destroy
  belongs_to :parent, class_name: "Comment", optional: true
end

# app/models/tag.rb
class Tag < ApplicationRecord
  has_many :article_tags, dependent: :destroy
  has_many :articles, through: :article_tags

  scope :popular, -> { order(articles_count: :desc) }
end

# app/models/follow.rb
class Follow < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"

  validates :follower_id, uniqueness: { scope: :followed_id }
  validate :cannot_follow_self

  after_create :notify_followed_user

  private

  def cannot_follow_self
    errors.add(:base, "Cannot follow yourself") if follower_id == followed_id
  end

  def notify_followed_user
    Notification.create!(
      recipient: followed,
      notifiable: self,
      message: "#{follower.name} started following you"
    )
  end
end

# app/models/like.rb
class Like < ApplicationRecord
  belongs_to :user
  belongs_to :likeable, polymorphic: true, counter_cache: true

  validates :user_id, uniqueness: {
    scope: [:likeable_type, :likeable_id],
    message: "already liked this"
  }

  after_create :increment_likes_count
  after_destroy :decrement_likes_count

  private

  def increment_likes_count
    likeable.increment!(:likes_count)
  end

  def decrement_likes_count
    likeable.decrement!(:likes_count)
  end
end

# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :recipient, class_name: "User"
  belongs_to :notifiable, polymorphic: true, optional: true

  scope :unread, -> { where(read_at: nil) }
  scope :recent, -> { order(created_at: :desc) }

  def mark_as_read!
    update!(read_at: Time.current)
  end
end
```

---

## แบบฝึกหัด Part 38 (ขั้นตอนที่ 831-860)

**ข้อ 1:** สร้าง has_many :through สำหรับ Student และ Course ผ่าน Enrollment

```ruby
# คำตอบ:
class Student < ApplicationRecord
  has_many :enrollments, dependent: :destroy
  has_many :courses, through: :enrollments
end

class Course < ApplicationRecord
  has_many :enrollments, dependent: :destroy
  has_many :students, through: :enrollments
end

class Enrollment < ApplicationRecord
  belongs_to :student
  belongs_to :course

  # Extra: grade, enrolled_at, status
  validates :student_id, uniqueness: { scope: :course_id }
end
```

**ข้อ 2:** สร้าง polymorphic association สำหรับ Attachments

```ruby
# คำตอบ:
class Attachment < ApplicationRecord
  belongs_to :attachable, polymorphic: true
  has_one_attached :file
  validates :file, presence: true
end

class Article < ApplicationRecord
  has_many :attachments, as: :attachable, dependent: :destroy
end

class Message < ApplicationRecord
  has_many :attachments, as: :attachable, dependent: :destroy
end

# Migration:
create_table :attachments do |t|
  t.references :attachable, polymorphic: true, null: false
  t.string :filename
  t.string :content_type
  t.bigint :file_size
  t.timestamps
end
```

**ข้อ 3:** implement N+1 fix ด้วย includes

```ruby
# คำตอบ:
# ❌ N+1:
@articles = Article.published
@articles.each { |a| puts "#{a.title} by #{a.user.name}" }

# ✅ Fixed:
@articles = Article.published.includes(:user, :tags, :category)
@articles.each { |a| puts "#{a.title} by #{a.user.name}" }
```

**ข้อ 4:** สร้าง self-referential association สำหรับ categories

```ruby
# คำตอบ:
class Category < ApplicationRecord
  belongs_to :parent, class_name: "Category", optional: true
  has_many :children, class_name: "Category",
    foreign_key: "parent_id",
    dependent: :destroy

  scope :roots, -> { where(parent_id: nil) }

  def root?
    parent_id.nil?
  end
end
```

**ข้อ 5:** เขียน RSpec tests สำหรับ associations

```ruby
# คำตอบ:
RSpec.describe Article, type: :model do
  it { should belong_to(:user) }
  it { should have_many(:comments).dependent(:destroy) }
  it { should have_many(:tags).through(:article_tags) }
  it { should belong_to(:category).optional }

  describe "comments association" do
    let(:article) { create(:article) }
    let(:user) { create(:user) }

    it "can have many comments" do
      expect {
        create(:comment, article: article, user: user)
        create(:comment, article: article, user: user)
      }.to change { article.comments.count }.by(2)
    end
  end
end
```

**ข้อ 6:** สร้าง association กับ scope

```ruby
# คำตอบ:
class User < ApplicationRecord
  has_many :articles
  has_many :published_articles,
    -> { where(status: "published").order(published_at: :desc) },
    class_name: "Article"
  has_many :recent_articles,
    -> { order(created_at: :desc).limit(5) },
    class_name: "Article"
  has_many :featured_articles,
    -> { where(featured: true) },
    class_name: "Article"
end
```

**ข้อ 7:** implement counter cache

```ruby
# คำตอบ:
# Migration:
add_column :users, :articles_count, :integer, default: 0, null: false

User.find_each do |user|
  User.reset_counters(user.id, :articles)
end

# Model:
class Article < ApplicationRecord
  belongs_to :user, counter_cache: :articles_count
end

# ใช้งาน:
user.articles_count  # เร็ว! ไม่ต้อง COUNT query
```

**ข้อ 8:** สร้าง User ที่ Follow User ด้วย self-referential

```ruby
# คำตอบ:
class User < ApplicationRecord
  has_many :active_follows, class_name: "Follow", foreign_key: :follower_id, dependent: :destroy
  has_many :passive_follows, class_name: "Follow", foreign_key: :followed_id, dependent: :destroy
  has_many :following, through: :active_follows, source: :followed
  has_many :followers, through: :passive_follows, source: :follower

  def follow!(other)
    following << other unless following?(other)
  end

  def unfollow!(other)
    following.delete(other)
  end

  def following?(other)
    following.include?(other)
  end
end
```

**ข้อ 9:** สร้าง belongs_to กับ optional และ validate custom

```ruby
# คำตอบ:
class Article < ApplicationRecord
  belongs_to :category, optional: true
  validates :category_id, presence: true, if: :published?

  def published?
    status == "published"
  end
end
```

**ข้อ 10:** eager load nested associations

```ruby
# คำตอบ:
@articles = Article.published
                   .includes(
                     :user,
                     :category,
                     :tags,
                     comments: { user: :profile }
                   )
                   .order(published_at: :desc)
                   .page(params[:page])
                   .per(10)
```

**ข้อ 11-30:** ดูตัวอย่าง associations ที่สมบูรณ์ในส่วน "Full Blog Application Associations" ด้านบน

---

## สรุป Part 38

ในบทนี้เราได้เรียนรู้:

1. **belongs_to** - foreign key relationship (many side)
2. **has_many** - one-to-many relationship
3. **has_one** - one-to-one relationship
4. **has_many :through** - many-to-many ผ่าน join model
5. **has_one :through** - one-to-one ผ่าน intermediary model
6. **HABTM** - simple many-to-many (ไม่แนะนำ)
7. **Polymorphic** - model หนึ่ง belong_to หลาย models
8. **Self-referential** - model สัมพันธ์กับตัวเอง
9. **Association Options** - dependent, class_name, scope, etc.
10. **Eager Loading** - แก้ N+1 ด้วย includes/preload/eager_load
11. **N+1 Problem** - และวิธีแก้

Associations เป็นหัวใจของ Rails ORM ที่ทำให้การทำงานกับ related data เป็นเรื่องง่ายและ intuitive

---

*ต่อไป: Part 39 - Validations (ขั้นตอนที่ 861-880)*
