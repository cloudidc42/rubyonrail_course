# ตอนที่ 47: JSON และ Serialization (Steps 1036-1055)

## บทนำ

Serialization คือกระบวนการแปลง Ruby objects ให้เป็น JSON format ที่ส่งผ่าน API ได้ Rails มีหลายวิธีในการทำ serialization ตั้งแต่วิธีพื้นฐานอย่าง `as_json` จนถึง library ที่ซับซ้อนอย่าง `jsonapi-serializer`

ในบทนี้เราจะเรียนรู้แต่ละวิธีพร้อมข้อดีข้อเสีย เพื่อให้เลือกใช้ได้เหมาะสมกับแต่ละโปรเจกต์

---

## Step 1036: to_json และ as_json

### ความแตกต่างระหว่าง to_json และ as_json

```ruby
# to_json - แปลง object เป็น JSON string โดยตรง
article = Article.first
article.to_json
# => '{"id":1,"title":"Hello","body":"World","created_at":"2024-01-01T00:00:00Z"}'

# as_json - แปลง object เป็น Hash ก่อน (ก่อนจะ serialize เป็น JSON)
article.as_json
# => {"id"=>1, "title"=>"Hello", "body"=>"World", "created_at"=>"2024-01-01T00:00:00Z"}
```

### as_json Options

```ruby
# only: แสดงเฉพาะ fields ที่ระบุ
article.as_json(only: [:id, :title, :created_at])
# => {"id"=>1, "title"=>"Hello", "created_at"=>"2024-01-01T00:00:00Z"}

# except: แสดงทุก field ยกเว้นที่ระบุ
article.as_json(except: [:body, :updated_at])

# include: รวม associations
article.as_json(include: :user)
article.as_json(include: { user: { only: [:id, :name] } })

# methods: เรียก instance methods
article.as_json(methods: [:excerpt, :reading_time])

# ผสมกัน
article.as_json(
  only: [:id, :title],
  include: { 
    user: { only: [:id, :name, :email] },
    comments: { 
      only: [:id, :body, :created_at],
      include: { user: { only: [:id, :name] } }
    }
  },
  methods: [:comments_count, :excerpt]
)
```

### Override as_json ใน Model

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :user
  has_many :comments
  
  def as_json(options = {})
    default_options = {
      only: [:id, :title, :body, :published, :views_count, :created_at, :updated_at],
      include: {
        user: { only: [:id, :name, :email] }
      },
      methods: [:excerpt, :reading_time, :comments_count]
    }
    
    super(default_options.deep_merge(options))
  end
  
  def excerpt
    body&.truncate(200, separator: ' ')
  end
  
  def reading_time
    words_per_minute = 200
    words = body&.split&.length || 0
    minutes = [(words / words_per_minute.to_f).ceil, 1].max
    "#{minutes} นาที"
  end
  
  def comments_count
    comments.count
  end
end
```

---

## Step 1037: Jbuilder

### การใช้ Jbuilder

Jbuilder ทำให้สร้าง JSON ใน view files ได้ คล้าย ERB views

```ruby
# Gemfile (รวมอยู่ใน Rails แล้ว สำหรับ non-API mode)
gem 'jbuilder'
```

### Jbuilder View Files

```ruby
# app/views/api/v1/articles/index.json.jbuilder
json.success true
json.data do
  json.array! @articles do |article|
    json.extract! article, :id, :title, :body, :published, :views_count, :created_at
    json.user do
      json.extract! article.user, :id, :name, :email
    end
    json.excerpt article.body&.truncate(150)
    json.comments_count article.comments.count
  end
end
json.meta do
  json.total_items @pagy.count
  json.total_pages @pagy.pages
  json.current_page @pagy.page
  json.per_page @pagy.items
end
```

```ruby
# app/views/api/v1/articles/show.json.jbuilder
json.success true
json.data do
  json.extract! @article, :id, :title, :body, :published, :views_count, :created_at, :updated_at
  
  json.user do
    json.extract! @article.user, :id, :name, :email
    json.articles_count @article.user.articles.count
  end
  
  json.comments do
    json.array! @article.comments.includes(:user) do |comment|
      json.extract! comment, :id, :body, :created_at
      json.user do
        json.extract! comment.user, :id, :name
      end
    end
  end
  
  json.meta do
    json.reading_time @article.reading_time
    json.comments_count @article.comments.count
    json.is_published @article.published?
  end
end
```

```ruby
# app/views/api/v1/articles/_article.json.jbuilder (partial)
json.extract! article, :id, :title, :published, :views_count, :created_at
json.excerpt article.body&.truncate(150)
json.author article.user.name

# ใช้ partial ใน index
# app/views/api/v1/articles/index.json.jbuilder
json.array! @articles, partial: 'api/v1/articles/article', as: :article
```

### Controller ที่ใช้ Jbuilder

```ruby
# app/controllers/api/v1/articles_controller.rb
def index
  @articles = Article.published.includes(:user, :comments)
  @pagy, @articles = pagy(@articles)
  # Jbuilder จะ render index.json.jbuilder อัตโนมัติ
end

def show
  @article = Article.includes(:user, comments: :user).find(params[:id])
  @article.increment_views!
  # Jbuilder จะ render show.json.jbuilder
end
```

---

## Step 1038: Active Model Serializers (AMS)

### Setup

```ruby
# Gemfile
gem 'active_model_serializers', '~> 0.10.0'
```

```bash
bundle install

# Generate serializer
rails generate serializer Article
rails generate serializer User
rails generate serializer Comment
```

```ruby
# config/initializers/ams.rb
ActiveModelSerializers.config.adapter = :json  # หรือ :json_api
ActiveModelSerializers.config.key_transform = :camel_lower  # หรือ :dash, :underscore
```

### Serializer Classes

```ruby
# app/serializers/user_serializer.rb
class UserSerializer < ActiveModel::Serializer
  attributes :id, :name, :email, :role, :created_at
  
  has_many :articles
  
  attribute :full_name do
    "#{object.first_name} #{object.last_name}"
  end
  
  attribute :articles_count do
    object.articles.count
  end
  
  # Conditional attribute
  attribute :admin_data, if: :admin? do
    {
      permissions: object.permissions,
      last_login: object.last_sign_in_at
    }
  end
  
  private
  
  def admin?
    scope&.admin?
  end
end
```

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :body, :published, :views_count, :created_at, :updated_at
  
  belongs_to :user
  has_many :comments
  
  attribute :excerpt do
    object.body&.truncate(200, separator: ' ')
  end
  
  attribute :reading_time do
    words = object.body&.split&.length || 0
    minutes = [(words / 200.0).ceil, 1].max
    "#{minutes} นาที"
  end
  
  attribute :comments_count do
    object.comments.size
  end
  
  attribute :url do
    Rails.application.routes.url_helpers.api_v1_article_url(object)
  end
  
  # แสดง body เฉพาะ detail view
  attribute :body, if: -> { instance_options[:include_body] }
end
```

### ใช้ใน Controller

```ruby
# app/controllers/api/v1/articles_controller.rb
def index
  articles = Article.published.includes(:user, :comments)
  render json: articles, each_serializer: ArticleSerializer
end

def show
  article = Article.find(params[:id])
  # ส่ง options ไปยัง serializer
  render json: article, 
         serializer: ArticleSerializer,
         include_body: true,
         scope: current_user,
         scope_name: :current_user
end

def create
  article = current_user.articles.build(article_params)
  if article.save
    render json: article, 
           serializer: ArticleSerializer,
           status: :created
  else
    render json: { errors: article.errors }, status: :unprocessable_entity
  end
end
```

### Nested Associations

```ruby
# app/serializers/article_with_comments_serializer.rb
class ArticleWithCommentsSerializer < ArticleSerializer
  has_many :comments, serializer: CommentWithUserSerializer
  
  attributes :full_text  # override ของ parent
  
  def full_text
    object.body  # แสดง body เต็ม (ไม่ truncate)
  end
end

# app/serializers/comment_with_user_serializer.rb
class CommentWithUserSerializer < ActiveModel::Serializer
  attributes :id, :body, :created_at
  belongs_to :user, serializer: UserBriefSerializer
end

# app/serializers/user_brief_serializer.rb
class UserBriefSerializer < ActiveModel::Serializer
  attributes :id, :name, :email
end
```

---

## Step 1039: Blueprinter

### Setup

```ruby
# Gemfile
gem 'blueprinter'
```

### Blueprint Classes

```ruby
# app/blueprints/user_blueprint.rb
class UserBlueprint < Blueprinter::Base
  identifier :id
  
  # Default view
  fields :name, :email, :created_at
  
  field :role do |user, _options|
    user.role.to_s.capitalize
  end
  
  # Named views
  view :extended do
    fields :name, :email, :role, :created_at, :updated_at
    
    field :articles_count do |user|
      user.articles.count
    end
    
    association :articles, blueprint: ArticleBlueprint, view: :brief
  end
  
  view :admin do
    include_view :extended
    fields :id  # Admin ได้เห็น id
    
    field :total_comments do |user|
      user.comments.count
    end
  end
end
```

```ruby
# app/blueprints/article_blueprint.rb
class ArticleBlueprint < Blueprinter::Base
  identifier :id
  
  view :brief do
    fields :id, :title, :published, :created_at
    
    field :excerpt do |article|
      article.body&.truncate(150)
    end
    
    association :user, blueprint: UserBlueprint
  end
  
  view :normal do
    include_view :brief
    fields :body, :views_count, :updated_at
    
    field :reading_time do |article|
      words = article.body&.split&.length || 0
      "#{[(words / 200.0).ceil, 1].max} นาที"
    end
  end
  
  view :full do
    include_view :normal
    association :comments, blueprint: CommentBlueprint, view: :with_user
    
    field :comments_count do |article|
      article.comments.size
    end
  end
end
```

### ใช้ Blueprinter ใน Controller

```ruby
# app/controllers/api/v1/articles_controller.rb
def index
  articles = Article.published.includes(:user)
  pagy, articles = pagy(articles)
  
  render json: {
    success: true,
    data: ArticleBlueprint.render_as_hash(articles, view: :brief),
    meta: {
      total: pagy.count,
      page: pagy.page
    }
  }
end

def show
  article = Article.includes(:user, comments: :user).find(params[:id])
  
  render json: {
    success: true,
    data: ArticleBlueprint.render_as_hash(article, view: :full)
  }
end

# render เป็น JSON string โดยตรง
def export
  articles = Article.all
  render json: ArticleBlueprint.render(articles, view: :normal)
end
```

---

## Step 1040: jsonapi-serializer (fast_jsonapi)

### Setup

```ruby
# Gemfile
gem 'jsonapi-serializer'
```

### Serializer Classes

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer
  include JSONAPI::Serializer
  
  set_type :article
  set_id :id
  
  attributes :title, :body, :published, :views_count, :created_at, :updated_at
  
  attribute :excerpt do |article|
    article.body&.truncate(200)
  end
  
  attribute :reading_time do |article|
    words = article.body&.split&.length || 0
    "#{[(words / 200.0).ceil, 1].max} นาที"
  end
  
  belongs_to :user
  has_many :comments
  
  # Cache control
  cache_options store: Rails.cache, namespace: 'jsonapi-serializer', expires_in: 1.hour
end
```

```ruby
# app/serializers/user_serializer.rb
class UserSerializer
  include JSONAPI::Serializer
  
  set_type :user
  
  attributes :name, :email, :role, :created_at
  
  has_many :articles
  
  # Conditional attribute
  attribute :admin_notes, if: proc { |record, params|
    params[:current_user]&.admin?
  }
end
```

### ใช้ใน Controller

```ruby
# app/controllers/api/v1/articles_controller.rb
def index
  articles = Article.published.includes(:user, :comments)
  pagy, articles = pagy(articles)
  
  options = {
    meta: {
      total_pages: pagy.pages,
      current_page: pagy.page,
      total_items: pagy.count
    },
    params: { current_user: current_user }
  }
  
  render json: ArticleSerializer.new(articles, options).serializable_hash
end

def show
  article = Article.includes(:user, comments: :user).find(params[:id])
  
  options = {
    include: [:user, :'comments.user'],
    params: { current_user: current_user }
  }
  
  render json: ArticleSerializer.new(article, options).serializable_hash
end
```

### JSON:API Format Output

```json
{
  "data": {
    "id": "1",
    "type": "article",
    "attributes": {
      "title": "Hello World",
      "body": "Content here",
      "excerpt": "Content...",
      "reading_time": "2 นาที"
    },
    "relationships": {
      "user": {
        "data": { "id": "1", "type": "user" }
      },
      "comments": {
        "data": [
          { "id": "1", "type": "comment" }
        ]
      }
    }
  },
  "included": [
    {
      "id": "1",
      "type": "user",
      "attributes": {
        "name": "John Doe",
        "email": "john@example.com"
      }
    }
  ],
  "meta": {
    "total_pages": 5,
    "current_page": 1
  }
}
```

---

## Step 1041: Custom Serializers

### Pure Ruby Serializer

```ruby
# app/serializers/base_serializer.rb
class BaseSerializer
  def initialize(object, options = {})
    @object = object
    @options = options
    @current_user = options[:current_user]
  end
  
  def as_json
    raise NotImplementedError, "Subclasses must implement as_json"
  end
  
  def self.serialize(object, options = {})
    new(object, options).as_json
  end
  
  def self.serialize_collection(objects, options = {})
    objects.map { |obj| new(obj, options).as_json }
  end
  
  private
  
  attr_reader :object, :options, :current_user
end
```

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer < BaseSerializer
  def as_json
    base_data.tap do |data|
      data[:user] = UserSerializer.serialize(object.user) if object.association(:user).loaded?
      data[:comments] = CommentSerializer.serialize_collection(object.comments) if options[:include_comments]
    end
  end
  
  private
  
  def base_data
    {
      id: object.id,
      title: object.title,
      body: include_full_body? ? object.body : nil,
      excerpt: object.body&.truncate(200),
      published: object.published,
      views_count: object.views_count,
      reading_time: calculate_reading_time,
      created_at: object.created_at.iso8601,
      updated_at: object.updated_at.iso8601
    }.compact
  end
  
  def include_full_body?
    options[:include_body] == true
  end
  
  def calculate_reading_time
    words = object.body&.split&.length || 0
    minutes = [(words / 200.0).ceil, 1].max
    "#{minutes} นาที"
  end
end
```

---

## Step 1042: Serializing Associations

### Handling N+1 ด้วย includes

```ruby
# app/controllers/api/v1/articles_controller.rb
def index
  # ป้องกัน N+1 queries
  articles = Article.published
                    .includes(:user, :comments, :tags)  # Eager load
                    .order(created_at: :desc)
  
  pagy, articles = pagy(articles)
  
  render json: {
    data: articles.map { |article| ArticleSerializer.new(article).as_json }
  }
end
```

### Conditional Associations

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :published, :created_at
  
  # Association ที่แสดงเฉพาะเมื่อ include ขอ
  has_many :comments, if: -> { instance_options[:include_comments] }
  belongs_to :user
  
  # Conditional nested fields
  attribute :user_email, if: -> { 
    scope&.admin? || object.user == scope 
  } do
    object.user.email
  end
end
```

### Polymorphic Associations

```ruby
# app/serializers/activity_serializer.rb
class ActivitySerializer < ActiveModel::Serializer
  attributes :id, :action, :created_at
  
  attribute :subject do
    case object.subject_type
    when 'Article'
      ArticleSerializer.new(object.subject).as_json
    when 'Comment'
      CommentSerializer.new(object.subject).as_json
    when 'User'
      UserSerializer.new(object.subject).as_json
    end
  end
end
```

---

## Step 1043: Conditional Attributes

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :published, :created_at
  
  # แสดงเฉพาะเมื่อ published
  attribute :published_at, if: -> { object.published? }
  
  # แสดงเฉพาะ admin
  attribute :internal_notes, if: -> { scope&.admin? } do
    object.internal_notes
  end
  
  # แสดงเฉพาะ owner หรือ admin
  attribute :draft_content, if: -> {
    scope&.admin? || object.user_id == scope&.id
  } do
    object.draft_body
  end
  
  # แสดงตาม options ที่ส่งมา
  attribute :full_body, if: -> { instance_options[:detailed_view] } do
    object.body
  end
end
```

---

## Step 1044: Camelize vs Snake_case

### Key Transform

```ruby
# config/initializers/ams.rb
# ใช้ camelCase (สำหรับ JavaScript frontend)
ActiveModelSerializers.config.key_transform = :camel_lower
# { "createdAt": "...", "viewsCount": 0 }

# ใช้ dash (สำหรับ JSON:API)
ActiveModelSerializers.config.key_transform = :dash
# { "created-at": "...", "views-count": 0 }

# ใช้ underscore (default Rails)
ActiveModelSerializers.config.key_transform = :underscore
# { "created_at": "...", "views_count": 0 }
```

### Manual Camelize

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer
  include JSONAPI::Serializer
  
  set_key_transform :camel_lower
  
  attributes :title, :body, :created_at
  # Output: { "title": "...", "createdAt": "..." }
end
```

```ruby
# Custom camelize helper
module CamelizeHelper
  def camelize_keys(hash)
    hash.transform_keys { |key| key.to_s.camelize(:lower).to_sym }
  end
  
  def deep_camelize_keys(obj)
    case obj
    when Hash
      obj.transform_keys { |key| key.to_s.camelize(:lower) }
         .transform_values { |val| deep_camelize_keys(val) }
    when Array
      obj.map { |item| deep_camelize_keys(item) }
    else
      obj
    end
  end
end
```

---

## Step 1045: Performance Comparison

### Benchmark ต่างๆ

```ruby
# lib/tasks/benchmark.rake
namespace :benchmark do
  desc "Compare serializer performance"
  task serializers: :environment do
    require 'benchmark'
    
    articles = Article.includes(:user, :comments).first(100)
    
    Benchmark.bm(30) do |x|
      x.report("as_json:") do
        100.times { articles.as_json(include: { user: {}, comments: {} }) }
      end
      
      x.report("AMS:") do
        100.times { 
          ActiveModelSerializers::SerializableResource.new(articles).as_json 
        }
      end
      
      x.report("Blueprinter:") do
        100.times { ArticleBlueprint.render_as_hash(articles, view: :normal) }
      end
      
      x.report("jsonapi-serializer:") do
        100.times { ArticleSerializer.new(articles).serializable_hash }
      end
    end
  end
end
```

### ผลการ Benchmark (approximate)

| Gem | Speed | Memory | Use Case |
|-----|-------|--------|---------|
| as_json | Medium | Low | Simple, quick |
| Jbuilder | Slow | Medium | Complex views |
| AMS | Slow | High | Full-featured |
| Blueprinter | Fast | Low | Production |
| jsonapi-serializer | Very Fast | Low | JSON:API |

---

## Step 1046: Best Practices

### การเลือก Serializer

```ruby
# สำหรับ Simple API
# ใช้ as_json หรือ custom Hash serializer

# สำหรับ Complex API ที่ต้องการ flexibility
# ใช้ Active Model Serializers

# สำหรับ High Performance API
# ใช้ Blueprinter หรือ jsonapi-serializer

# สำหรับ JSON:API spec compliance
# ใช้ jsonapi-serializer
```

### Caching Serialization Results

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer < ActiveModel::Serializer
  cache key: 'article', expires_in: 1.hour
  
  attributes :id, :title, :body, :created_at
  belongs_to :user
end
```

### Avoiding N+1 ใน Serializers

```ruby
# ไม่ดี - N+1 query
articles.each do |article|
  article.user.name  # query สำหรับทุก article
  article.comments.count  # อีก query
end

# ดี - Eager load
articles = Article.includes(:user, :comments).all
articles.each do |article|
  article.user.name  # ไม่มี N+1
  article.comments.count  # ใช้ counter_cache
end

# ดีกว่า - ใช้ counter_cache
add_column :articles, :comments_count, :integer, default: 0

class Comment < ApplicationRecord
  belongs_to :article, counter_cache: true
end

# ใน serializer
def comments_count
  object.comments_count  # ไม่ต้อง query
end
```

---

## แบบฝึกหัด (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** เขียน as_json ที่แสดงเฉพาะ id, name, email ของ User
```ruby
# เฉลย
user.as_json(only: [:id, :name, :email])
```

**ข้อ 2:** เขียน as_json ที่รวม User's articles พร้อม title และ created_at
```ruby
# เฉลย
user.as_json(
  only: [:id, :name],
  include: { articles: { only: [:id, :title, :created_at] } }
)
```

**ข้อ 3:** สร้าง Jbuilder view สำหรับ product list ที่แสดง id, name, price, category name
```ruby
# เฉลย
# app/views/api/v1/products/index.json.jbuilder
json.array! @products do |product|
  json.extract! product, :id, :name, :price
  json.category_name product.category&.name
end
```

**ข้อ 4:** สร้าง ProductSerializer ด้วย AMS ที่มี :brief view แสดง id, name, price
```ruby
# เฉลย
class ProductSerializer < ActiveModel::Serializer
  attributes :id, :name, :price
  
  attribute :in_stock do
    object.stock_quantity > 0
  end
end
```

**ข้อ 5:** สร้าง Blueprint สำหรับ User ที่มี :with_stats view แสดงจำนวน articles และ comments
```ruby
# เฉลย
class UserBlueprint < Blueprinter::Base
  identifier :id
  fields :name, :email
  
  view :with_stats do
    fields :name, :email, :created_at
    
    field :articles_count do |user|
      user.articles.count
    end
    
    field :comments_count do |user|
      user.comments.count
    end
  end
end
```

### ระดับกลาง

**ข้อ 6:** สร้าง jsonapi-serializer สำหรับ Article ที่มี relationships กับ User และ Tags
```ruby
# เฉลย
class ArticleSerializer
  include JSONAPI::Serializer
  
  set_type :article
  attributes :title, :body, :published, :created_at
  
  attribute :excerpt do |article|
    article.body&.truncate(200)
  end
  
  belongs_to :user
  has_many :tags
end
```

**ข้อ 7:** เพิ่ม conditional attribute ใน AMS serializer ที่แสดง admin_notes เฉพาะ admin users
```ruby
# เฉลย
class ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :body
  
  attribute :admin_notes, if: -> { scope&.admin? } do
    object.admin_notes
  end
end
```

**ข้อ 8:** สร้าง custom serializer แบบ Pure Ruby ที่ transform keys เป็น camelCase
```ruby
# เฉลย
class ArticleSerializer
  def initialize(article)
    @article = article
  end
  
  def as_json
    {
      id: @article.id,
      articleTitle: @article.title,
      articleBody: @article.body,
      createdAt: @article.created_at.iso8601,
      viewsCount: @article.views_count
    }
  end
end
```

**ข้อ 9:** เพิ่ม caching ใน AMS serializer
```ruby
# เฉลย
class ProductSerializer < ActiveModel::Serializer
  cache key: 'product', expires_in: 30.minutes
  
  attributes :id, :name, :price, :description
  belongs_to :category
end
```

**ข้อ 10:** เขียน Jbuilder view ที่แสดง nested comments (comment มี sub-comments)
```ruby
# เฉลย
json.comments do
  json.array! @article.root_comments do |comment|
    json.extract! comment, :id, :body, :created_at
    json.replies do
      json.array! comment.replies do |reply|
        json.extract! reply, :id, :body, :created_at
        json.user_name reply.user.name
      end
    end
  end
end
```

### ระดับสูง

**ข้อ 11:** เขียน benchmark เปรียบเทียบ as_json กับ Blueprinter สำหรับ 1000 records
```ruby
# เฉลย
require 'benchmark'
articles = Article.includes(:user).first(1000)

Benchmark.bm do |x|
  x.report("as_json:") { articles.as_json(include: :user) }
  x.report("Blueprint:") { ArticleBlueprint.render_as_hash(articles) }
end
```

**ข้อ 12:** สร้าง serializer ที่ handle N+1 โดย detect ว่า association loaded หรือยัง
```ruby
# เฉลย
class ArticleSerializer
  def as_json
    data = { id: @article.id, title: @article.title }
    
    if @article.association(:user).loaded?
      data[:user] = { id: @article.user.id, name: @article.user.name }
    else
      data[:user_id] = @article.user_id
    end
    
    data
  end
end
```

**ข้อ 13-20:** (แบบฝึกหัดเพิ่มเติม)
- สร้าง serializer ที่รองรับ polymorphic associations
- implement streaming serialization สำหรับ large datasets
- สร้าง serializer ที่ exclude null/empty values
- implement pagination metadata ใน serializer
- สร้าง serializer versioning system
- implement field selection (sparse fieldsets)
- สร้าง serializer ที่รองรับ circular references
- performance optimization ด้วย fragment caching

```ruby
# เฉลย ข้อ 14 - Sparse Fieldsets
class ArticleSerializer
  include JSONAPI::Serializer
  
  attributes :id, :title, :body, :published, :created_at
  
  # ใน controller
  # fields = params[:fields]&.split(',') || []
  # ArticleSerializer.new(article, { fields: { article: fields } })
end

# Request: GET /articles?fields[article]=id,title,created_at
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **as_json/to_json** - วิธีพื้นฐานที่ง่ายแต่ไม่ flexible
2. **Jbuilder** - ดีสำหรับ complex views, template-based
3. **Active Model Serializers** - full-featured, ช้ากว่า
4. **Blueprinter** - เร็ว, clean API, recommended สำหรับ production
5. **jsonapi-serializer** - เร็วมาก, JSON:API spec compliant
6. **Custom Serializers** - maximum control

**แนะนำ:** ใช้ Blueprinter หรือ jsonapi-serializer สำหรับ production APIs เพราะเร็วและ maintain ง่าย
