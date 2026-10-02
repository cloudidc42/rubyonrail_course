# Part 56: GraphQL API ใน Ruby on Rails

## ขั้นตอนที่ 1221-1245: สร้าง API ด้วย GraphQL

---

## ขั้นตอนที่ 1221: GraphQL คืออะไร?

GraphQL คือภาษา Query สำหรับ API ที่พัฒนาโดย Facebook (Meta) ในปี 2012 และเปิดเผยเป็น Open Source ในปี 2015 GraphQL แตกต่างจาก REST API ตรงที่:

### REST vs GraphQL เปรียบเทียบ

**REST API:**
```
GET /users/1          → ได้ข้อมูล user ทั้งหมด
GET /users/1/posts    → ได้ posts ของ user
GET /users/1/friends  → ได้ friends ของ user
```

**GraphQL:**
```graphql
query {
  user(id: 1) {
    name
    email
    posts {
      title
      createdAt
    }
    friends {
      name
    }
  }
}
```

ด้วย GraphQL เราส่ง Query เดียวได้ข้อมูลทั้งหมดที่ต้องการ

### ข้อดีของ GraphQL

1. **ดึงข้อมูลได้ตรงตามต้องการ** - ไม่ Over-fetching หรือ Under-fetching
2. **Single Endpoint** - ใช้ endpoint เดียว `/graphql`
3. **Type System** - มี Schema ที่ชัดเจน
4. **Introspection** - ค้นหา Schema ได้จาก API เอง
5. **Real-time** - รองรับ Subscriptions

### ข้อเสียของ GraphQL

1. **Complexity** - ซับซ้อนกว่า REST
2. **N+1 Problem** - ต้องจัดการเองด้วย DataLoader/graphql-batch
3. **Caching** - ยากกว่า REST
4. **Learning Curve** - ใช้เวลาเรียนรู้มากกว่า

---

## ขั้นตอนที่ 1222: ติดตั้ง graphql-ruby gem

### สร้าง Rails Project ใหม่

```bash
rails new graphql_api --api --database=postgresql
cd graphql_api
```

### เพิ่ม Gems

```ruby
# Gemfile
gem 'graphql', '~> 2.0'
gem 'graphiql-rails', group: :development

# สำหรับ authentication
gem 'devise'
gem 'jwt'

# สำหรับ N+1 prevention
gem 'graphql-batch'

# สำหรับ pagination
gem 'pagy'
```

```bash
bundle install
```

### รัน GraphQL Generator

```bash
rails generate graphql:install
```

คำสั่งนี้จะสร้างไฟล์ต่างๆ:
```
app/graphql/
├── types/
│   ├── base_argument.rb
│   ├── base_enum.rb
│   ├── base_field.rb
│   ├── base_input_object.rb
│   ├── base_interface.rb
│   ├── base_object.rb
│   ├── base_scalar.rb
│   ├── base_union.rb
│   ├── mutation_type.rb
│   └── query_type.rb
├── mutations/
│   └── base_mutation.rb
└── my_app_schema.rb
```

### เพิ่ม Route

```ruby
# config/routes.rb
Rails.application.routes.draw do
  if Rails.env.development?
    mount GraphiQL::Rails::Engine, at: "/graphiql", graphql_path: "/graphql"
  end
  
  post "/graphql", to: "graphql#execute"
end
```

### GraphQL Controller

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  # ปิด CSRF สำหรับ API
  protect_from_forgery with: :null_session
  
  def execute
    variables = prepare_variables(params[:variables])
    query = params[:query]
    operation_name = params[:operationName]
    context = {
      current_user: current_user
    }
    
    result = GraphqlApiSchema.execute(
      query,
      variables: variables,
      context: context,
      operation_name: operation_name
    )
    
    render json: result
  rescue => e
    raise e unless Rails.env.development?
    handle_error_in_development(e)
  end
  
  private
  
  def prepare_variables(variables_param)
    case variables_param
    when String
      if variables_param.present?
        JSON.parse(variables_param) || {}
      else
        {}
      end
    when Hash
      variables_param
    when ActionController::Parameters
      variables_param.to_unsafe_hash
    when nil
      {}
    else
      raise ArgumentError, "Unexpected parameter: #{variables_param}"
    end
  end
  
  def handle_error_in_development(e)
    logger.error e.message
    logger.error e.backtrace.join("\n")
    
    render json: {
      errors: [{ message: e.message, backtrace: e.backtrace }],
      data: {}
    }, status: 500
  end
  
  def current_user
    token = request.headers['Authorization']&.split(' ')&.last
    return nil unless token
    
    begin
      decoded = JWT.decode(token, Rails.application.credentials.secret_key_base, true, algorithm: 'HS256')
      User.find(decoded[0]['user_id'])
    rescue JWT::DecodeError, ActiveRecord::RecordNotFound
      nil
    end
  end
end
```

---

## ขั้นตอนที่ 1223: กำหนด Schema

### Schema หลัก

```ruby
# app/graphql/graphql_api_schema.rb
class GraphqlApiSchema < GraphQL::Schema
  mutation(Types::MutationType)
  query(Types::QueryType)
  subscription(Types::SubscriptionType)
  
  # Enable batch loading
  use GraphQL::Batch
  
  # Error handling
  rescue_from(ActiveRecord::RecordNotFound) do |err, obj, args, ctx, field|
    raise GraphQL::ExecutionError, "#{field.type.unwrap.graphql_name} not found"
  end
  
  rescue_from(ActiveRecord::RecordInvalid) do |err, obj, args, ctx, field|
    raise GraphQL::ExecutionError, err.message
  end
  
  # Authorization
  def self.unauthorized_object(error)
    raise GraphQL::ExecutionError, "คุณไม่มีสิทธิ์เข้าถึงข้อมูลนี้"
  end
  
  # Pagination
  default_page_size 25
  max_page_size 100
end
```

---

## ขั้นตอนที่ 1224: Object Types

Object Type คือประเภทที่ใช้แทนข้อมูลจาก Model

### Base Object

```ruby
# app/graphql/types/base_object.rb
module Types
  class BaseObject < GraphQL::Schema::Object
    field_class Types::BaseField
    
    # Helper สำหรับ authorization
    def self.authorized?(object, context)
      super
    end
  end
end
```

### User Type

```ruby
# app/graphql/types/user_type.rb
module Types
  class UserType < Types::BaseObject
    description "ผู้ใช้งานระบบ"
    
    field :id, ID, null: false, description: "รหัสผู้ใช้"
    field :email, String, null: false, description: "อีเมล"
    field :name, String, null: false, description: "ชื่อ"
    field :username, String, null: false, description: "ชื่อผู้ใช้"
    field :avatar_url, String, null: true, description: "URL รูปโปรไฟล์"
    field :bio, String, null: true, description: "ประวัติย่อ"
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
    
    # Nested fields
    field :posts, [Types::PostType], null: false, description: "บทความของผู้ใช้"
    field :posts_count, Integer, null: false, description: "จำนวนบทความ"
    field :followers, [Types::UserType], null: false, description: "ผู้ติดตาม"
    field :following, [Types::UserType], null: false, description: "ที่ติดตาม"
    field :followers_count, Integer, null: false
    field :following_count, Integer, null: false
    
    # Computed fields
    field :is_following, Boolean, null: false, description: "กำลังติดตามหรือไม่"
    
    def posts_count
      object.posts.count
    end
    
    def followers_count
      object.followers.count
    end
    
    def following_count
      object.following.count
    end
    
    def is_following
      context[:current_user]&.following?(object) || false
    end
    
    # เปิดเผย email เฉพาะเจ้าของบัญชี
    def email
      if context[:current_user]&.id == object.id
        object.email
      else
        nil
      end
    end
  end
end
```

### Post Type

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    description "บทความ"
    
    field :id, ID, null: false
    field :title, String, null: false, description: "หัวข้อบทความ"
    field :body, String, null: false, description: "เนื้อหา"
    field :excerpt, String, null: true, description: "สรุปย่อ"
    field :slug, String, null: false, description: "URL slug"
    field :status, Types::PostStatusEnum, null: false, description: "สถานะ"
    field :published_at, GraphQL::Types::ISO8601DateTime, null: true
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
    
    field :author, Types::UserType, null: false, description: "ผู้เขียน"
    field :tags, [Types::TagType], null: false, description: "แท็ก"
    field :comments, [Types::CommentType], null: false, description: "ความคิดเห็น"
    field :comments_count, Integer, null: false
    field :likes_count, Integer, null: false
    field :is_liked, Boolean, null: false
    
    def comments_count
      object.comments.count
    end
    
    def likes_count
      object.likes.count
    end
    
    def is_liked
      return false unless context[:current_user]
      object.likes.exists?(user: context[:current_user])
    end
  end
end
```

---

## ขั้นตอนที่ 1225: Scalar Types

Scalar Type คือประเภทพื้นฐานที่ไม่มี sub-fields

### ประเภท Scalar ที่มีในตัว

```graphql
String    # ข้อความ
Int       # จำนวนเต็ม
Float     # จำนวนทศนิยม
Boolean   # true/false
ID        # รหัสเฉพาะ
```

### Custom Scalar - JSON

```ruby
# app/graphql/types/json_type.rb
module Types
  class JsonType < GraphQL::Schema::Scalar
    description "JSON object (arbitrary key-value pairs)"
    
    def self.coerce_input(input_value, context)
      case input_value
      when String
        JSON.parse(input_value)
      when Hash, Array
        input_value
      else
        raise GraphQL::CoercionError, "#{input_value.inspect} is not valid JSON"
      end
    end
    
    def self.coerce_result(ruby_value, context)
      ruby_value
    end
  end
end
```

### Custom Scalar - URL

```ruby
# app/graphql/types/url_type.rb
module Types
  class UrlType < GraphQL::Schema::Scalar
    description "A valid URL string"
    
    def self.coerce_input(input_value, context)
      uri = URI.parse(input_value)
      unless uri.is_a?(URI::HTTP) || uri.is_a?(URI::HTTPS)
        raise GraphQL::CoercionError, "#{input_value.inspect} is not a valid URL"
      end
      input_value
    rescue URI::InvalidURIError
      raise GraphQL::CoercionError, "#{input_value.inspect} is not a valid URL"
    end
    
    def self.coerce_result(ruby_value, context)
      ruby_value.to_s
    end
  end
end
```

### Custom Scalar - Date

```ruby
# app/graphql/types/date_type.rb
module Types
  class DateType < GraphQL::Schema::Scalar
    description "วันที่ในรูปแบบ YYYY-MM-DD"
    
    def self.coerce_input(input_value, context)
      Date.parse(input_value)
    rescue Date::Error
      raise GraphQL::CoercionError, "#{input_value.inspect} ไม่ใช่รูปแบบวันที่ที่ถูกต้อง"
    end
    
    def self.coerce_result(ruby_value, context)
      ruby_value.strftime('%Y-%m-%d')
    end
  end
end
```

---

## ขั้นตอนที่ 1226: Enum Types

Enum Type คือประเภทที่มีค่าได้แค่ค่าที่กำหนดไว้

```ruby
# app/graphql/types/post_status_enum.rb
module Types
  class PostStatusEnum < Types::BaseEnum
    description "สถานะของบทความ"
    
    value "DRAFT", "แบบร่าง", value: "draft"
    value "PUBLISHED", "เผยแพร่แล้ว", value: "published"
    value "ARCHIVED", "เก็บถาวร", value: "archived"
    value "DELETED", "ลบแล้ว", value: "deleted"
  end
end
```

```ruby
# app/graphql/types/order_direction_enum.rb
module Types
  class OrderDirectionEnum < Types::BaseEnum
    description "ทิศทางการเรียงลำดับ"
    
    value "ASC", "น้อยไปมาก"
    value "DESC", "มากไปน้อย"
  end
end
```

```ruby
# app/graphql/types/sort_by_enum.rb
module Types
  class PostSortByEnum < Types::BaseEnum
    description "เรียงลำดับบทความตาม"
    
    value "CREATED_AT", "วันที่สร้าง"
    value "UPDATED_AT", "วันที่แก้ไข"
    value "TITLE", "หัวข้อ"
    value "LIKES_COUNT", "จำนวน likes"
    value "COMMENTS_COUNT", "จำนวน comments"
  end
end
```

---

## ขั้นตอนที่ 1227: Input Types

Input Type ใช้รับค่าที่ส่งมากับ Mutations

```ruby
# app/graphql/types/create_post_input.rb
module Types
  class CreatePostInput < Types::BaseInputObject
    description "ข้อมูลสำหรับสร้างบทความใหม่"
    
    argument :title, String, required: true, description: "หัวข้อบทความ"
    argument :body, String, required: true, description: "เนื้อหาบทความ"
    argument :excerpt, String, required: false, description: "สรุปย่อ"
    argument :status, Types::PostStatusEnum, required: false, default_value: "draft"
    argument :tag_ids, [ID], required: false, default_value: [], description: "รหัสแท็ก"
    argument :published_at, GraphQL::Types::ISO8601DateTime, required: false
  end
end
```

```ruby
# app/graphql/types/update_post_input.rb
module Types
  class UpdatePostInput < Types::BaseInputObject
    description "ข้อมูลสำหรับแก้ไขบทความ"
    
    argument :title, String, required: false
    argument :body, String, required: false
    argument :excerpt, String, required: false
    argument :status, Types::PostStatusEnum, required: false
    argument :tag_ids, [ID], required: false
    argument :published_at, GraphQL::Types::ISO8601DateTime, required: false
  end
end
```

```ruby
# app/graphql/types/user_filter_input.rb
module Types
  class UserFilterInput < Types::BaseInputObject
    description "ตัวกรองผู้ใช้"
    
    argument :search, String, required: false, description: "ค้นหาชื่อหรืออีเมล"
    argument :has_posts, Boolean, required: false
    argument :created_after, GraphQL::Types::ISO8601DateTime, required: false
  end
end
```

---

## ขั้นตอนที่ 1228: Union Types

Union Type ใช้เมื่อ field อาจคืนค่าได้หลายประเภท

```ruby
# app/graphql/types/search_result_union.rb
module Types
  class SearchResultUnion < Types::BaseUnion
    description "ผลการค้นหา"
    
    possible_types Types::PostType, Types::UserType, Types::TagType
    
    def self.resolve_type(object, context)
      case object
      when Post
        Types::PostType
      when User
        Types::UserType
      when Tag
        Types::TagType
      else
        raise "ไม่รู้จักประเภท: #{object.class}"
      end
    end
  end
end
```

```ruby
# app/graphql/types/mutation_result_union.rb
module Types
  class MutationResultUnion < Types::BaseUnion
    description "ผลการทำ Mutation"
    
    possible_types Types::PostType, Types::ValidationErrorType
    
    def self.resolve_type(object, context)
      case object
      when Post
        Types::PostType
      when Hash
        Types::ValidationErrorType
      end
    end
  end
end
```

### ValidationError Type

```ruby
# app/graphql/types/validation_error_type.rb
module Types
  class ValidationErrorType < Types::BaseObject
    description "ข้อผิดพลาดจากการ validate"
    
    field :field, String, null: false, description: "ชื่อ field ที่มีปัญหา"
    field :messages, [String], null: false, description: "ข้อความ error"
  end
end
```

---

## ขั้นตอนที่ 1229: Interface Types

Interface คือ contract ที่กำหนดว่า Type ต้องมี fields อะไรบ้าง

```ruby
# app/graphql/types/node_interface.rb
module Types
  module NodeInterface
    include Types::BaseInterface
    description "Object ที่มี global ID"
    
    field :id, ID, null: false, description: "Global ID"
    
    definition_methods do
      def resolve_type(object, context)
        case object
        when User then Types::UserType
        when Post then Types::PostType
        when Comment then Types::CommentType
        when Tag then Types::TagType
        end
      end
    end
  end
end
```

```ruby
# app/graphql/types/timestampable_interface.rb
module Types
  module TimestampableInterface
    include Types::BaseInterface
    description "Object ที่มี timestamps"
    
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
  end
end
```

### ใช้ Interface ใน Type

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    implements Types::NodeInterface
    implements Types::TimestampableInterface
    
    description "บทความ"
    
    field :id, ID, null: false
    field :title, String, null: false
    # ... other fields
  end
end
```

---

## ขั้นตอนที่ 1230: Query Type

Query Type กำหนด entry points สำหรับการดึงข้อมูล

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    description "Root query type"
    
    # ===== USERS =====
    field :me, Types::UserType, null: true, description: "ข้อมูลผู้ใช้ปัจจุบัน"
    
    field :user, Types::UserType, null: true, description: "ค้นหาผู้ใช้" do
      argument :id, ID, required: false
      argument :username, String, required: false
    end
    
    field :users, Types::UserConnectionType, null: false, 
          description: "รายการผู้ใช้",
          connection: true do
      argument :filter, Types::UserFilterInput, required: false
      argument :sort_by, String, required: false, default_value: "created_at"
      argument :direction, Types::OrderDirectionEnum, required: false, default_value: "DESC"
    end
    
    # ===== POSTS =====
    field :post, Types::PostType, null: true, description: "ค้นหาบทความ" do
      argument :id, ID, required: false
      argument :slug, String, required: false
    end
    
    field :posts, Types::PostConnectionType, null: false,
          description: "รายการบทความ",
          connection: true do
      argument :status, Types::PostStatusEnum, required: false
      argument :author_id, ID, required: false
      argument :tag_id, ID, required: false
      argument :search, String, required: false
      argument :sort_by, Types::PostSortByEnum, required: false, default_value: "CREATED_AT"
      argument :direction, Types::OrderDirectionEnum, required: false, default_value: "DESC"
    end
    
    # ===== SEARCH =====
    field :search, [Types::SearchResultUnion], null: false, description: "ค้นหาทั่วไป" do
      argument :query, String, required: true
      argument :limit, Integer, required: false, default_value: 10
    end
    
    # ===== TAGS =====
    field :tags, [Types::TagType], null: false, description: "รายการแท็ก"
    field :popular_tags, [Types::TagType], null: false, description: "แท็กยอดนิยม" do
      argument :limit, Integer, required: false, default_value: 10
    end
    
    # Resolver methods
    def me
      context[:current_user]
    end
    
    def user(id: nil, username: nil)
      if id
        User.find_by(id: id)
      elsif username
        User.find_by(username: username)
      end
    end
    
    def users(filter: nil, sort_by: "created_at", direction: "DESC")
      scope = User.all
      
      if filter
        if filter[:search].present?
          scope = scope.where("name ILIKE ? OR email ILIKE ?", 
                             "%#{filter[:search]}%", "%#{filter[:search]}%")
        end
        
        if filter[:has_posts]
          scope = scope.joins(:posts).distinct
        end
        
        if filter[:created_after]
          scope = scope.where("created_at >= ?", filter[:created_after])
        end
      end
      
      column = sort_by.downcase
      scope.order("#{column} #{direction}")
    end
    
    def post(id: nil, slug: nil)
      if id
        Post.find_by(id: id)
      elsif slug
        Post.find_by(slug: slug)
      end
    end
    
    def posts(status: nil, author_id: nil, tag_id: nil, search: nil, 
              sort_by: "CREATED_AT", direction: "DESC")
      scope = Post.all
      scope = scope.where(status: status) if status
      scope = scope.where(user_id: author_id) if author_id
      scope = scope.joins(:tags).where(tags: { id: tag_id }) if tag_id
      
      if search.present?
        scope = scope.where("title ILIKE ? OR body ILIKE ?", 
                           "%#{search}%", "%#{search}%")
      end
      
      column = case sort_by
               when "CREATED_AT" then "created_at"
               when "UPDATED_AT" then "updated_at"
               when "TITLE" then "title"
               when "LIKES_COUNT" then "likes_count"
               when "COMMENTS_COUNT" then "comments_count"
               else "created_at"
               end
      
      scope.order("#{column} #{direction}")
    end
    
    def search(query:, limit: 10)
      posts = Post.published.where("title ILIKE ?", "%#{query}%").limit(limit)
      users = User.where("name ILIKE ?", "%#{query}%").limit(limit)
      tags = Tag.where("name ILIKE ?", "%#{query}%").limit(limit)
      
      (posts + users + tags).first(limit)
    end
    
    def tags
      Tag.all.order(:name)
    end
    
    def popular_tags(limit: 10)
      Tag.joins(:posts).group("tags.id").order("COUNT(posts.id) DESC").limit(limit)
    end
  end
end
```

---

## ขั้นตอนที่ 1231: Mutations

Mutations ใช้สำหรับเปลี่ยนแปลงข้อมูล (Create, Update, Delete)

### Base Mutation

```ruby
# app/graphql/mutations/base_mutation.rb
module Mutations
  class BaseMutation < GraphQL::Schema::Mutation
    argument_class Types::BaseArgument
    field_class Types::BaseField
    input_object_class Types::BaseInputObject
    object_class Types::BaseObject
    
    # Helper สำหรับตรวจสอบว่า login หรือยัง
    def current_user
      context[:current_user]
    end
    
    def require_authentication!
      unless current_user
        raise GraphQL::ExecutionError, "กรุณาเข้าสู่ระบบก่อน"
      end
    end
    
    # แปลง ActiveRecord errors เป็น GraphQL errors
    def record_errors(record)
      record.errors.map do |error|
        { field: error.attribute.to_s, messages: [error.message] }
      end
    end
  end
end
```

### Create Post Mutation

```ruby
# app/graphql/mutations/create_post.rb
module Mutations
  class CreatePost < BaseMutation
    description "สร้างบทความใหม่"
    
    argument :input, Types::CreatePostInput, required: true
    
    field :post, Types::PostType, null: true, description: "บทความที่สร้าง"
    field :errors, [String], null: false, description: "ข้อผิดพลาด"
    
    def resolve(input:)
      require_authentication!
      
      post = current_user.posts.build(
        title: input[:title],
        body: input[:body],
        excerpt: input[:excerpt],
        status: input[:status] || "draft",
        published_at: input[:published_at]
      )
      
      # เพิ่ม tags
      if input[:tag_ids].present?
        tags = Tag.where(id: input[:tag_ids])
        post.tags = tags
      end
      
      if post.save
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    end
  end
end
```

### Update Post Mutation

```ruby
# app/graphql/mutations/update_post.rb
module Mutations
  class UpdatePost < BaseMutation
    description "แก้ไขบทความ"
    
    argument :id, ID, required: true
    argument :input, Types::UpdatePostInput, required: true
    
    field :post, Types::PostType, null: true
    field :errors, [String], null: false
    
    def resolve(id:, input:)
      require_authentication!
      
      post = Post.find(id)
      
      # ตรวจสอบสิทธิ์
      unless post.user_id == current_user.id || current_user.admin?
        raise GraphQL::ExecutionError, "คุณไม่มีสิทธิ์แก้ไขบทความนี้"
      end
      
      update_params = input.to_h.compact
      
      if input[:tag_ids]
        tags = Tag.where(id: input[:tag_ids])
        post.tags = tags
        update_params.delete(:tag_ids)
      end
      
      if post.update(update_params)
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    end
  end
end
```

### Delete Post Mutation

```ruby
# app/graphql/mutations/delete_post.rb
module Mutations
  class DeletePost < BaseMutation
    description "ลบบทความ"
    
    argument :id, ID, required: true
    
    field :success, Boolean, null: false
    field :message, String, null: true
    
    def resolve(id:)
      require_authentication!
      
      post = Post.find(id)
      
      unless post.user_id == current_user.id || current_user.admin?
        raise GraphQL::ExecutionError, "คุณไม่มีสิทธิ์ลบบทความนี้"
      end
      
      if post.destroy
        { success: true, message: "ลบบทความสำเร็จ" }
      else
        { success: false, message: "ไม่สามารถลบบทความได้" }
      end
    end
  end
end
```

### Authentication Mutations

```ruby
# app/graphql/mutations/sign_in.rb
module Mutations
  class SignIn < BaseMutation
    description "เข้าสู่ระบบ"
    
    argument :email, String, required: true
    argument :password, String, required: true
    
    field :token, String, null: true, description: "JWT Token"
    field :user, Types::UserType, null: true
    field :errors, [String], null: false
    
    def resolve(email:, password:)
      user = User.find_by(email: email.downcase)
      
      if user&.valid_password?(password)
        token = generate_jwt(user)
        { token: token, user: user, errors: [] }
      else
        { token: nil, user: nil, errors: ["อีเมลหรือรหัสผ่านไม่ถูกต้อง"] }
      end
    end
    
    private
    
    def generate_jwt(user)
      payload = {
        user_id: user.id,
        email: user.email,
        exp: 7.days.from_now.to_i
      }
      
      JWT.encode(payload, Rails.application.credentials.secret_key_base, 'HS256')
    end
  end
end
```

```ruby
# app/graphql/mutations/sign_up.rb
module Mutations
  class SignUp < BaseMutation
    description "สมัครสมาชิก"
    
    argument :email, String, required: true
    argument :password, String, required: true
    argument :name, String, required: true
    argument :username, String, required: true
    
    field :token, String, null: true
    field :user, Types::UserType, null: true
    field :errors, [String], null: false
    
    def resolve(email:, password:, name:, username:)
      user = User.new(
        email: email.downcase,
        password: password,
        name: name,
        username: username
      )
      
      if user.save
        token = generate_jwt(user)
        { token: token, user: user, errors: [] }
      else
        { token: nil, user: nil, errors: user.errors.full_messages }
      end
    end
    
    private
    
    def generate_jwt(user)
      payload = {
        user_id: user.id,
        email: user.email,
        exp: 7.days.from_now.to_i
      }
      JWT.encode(payload, Rails.application.credentials.secret_key_base, 'HS256')
    end
  end
end
```

### Mutation Type

```ruby
# app/graphql/types/mutation_type.rb
module Types
  class MutationType < Types::BaseObject
    description "Root mutation type"
    
    # Authentication
    field :sign_in, mutation: Mutations::SignIn
    field :sign_up, mutation: Mutations::SignUp
    
    # Posts
    field :create_post, mutation: Mutations::CreatePost
    field :update_post, mutation: Mutations::UpdatePost
    field :delete_post, mutation: Mutations::DeletePost
    field :publish_post, mutation: Mutations::PublishPost
    field :like_post, mutation: Mutations::LikePost
    field :unlike_post, mutation: Mutations::UnlikePost
    
    # Comments
    field :create_comment, mutation: Mutations::CreateComment
    field :update_comment, mutation: Mutations::UpdateComment
    field :delete_comment, mutation: Mutations::DeleteComment
    
    # Users
    field :update_profile, mutation: Mutations::UpdateProfile
    field :follow_user, mutation: Mutations::FollowUser
    field :unfollow_user, mutation: Mutations::UnfollowUser
  end
end
```

---

## ขั้นตอนที่ 1232: Subscriptions

Subscriptions ใช้สำหรับ real-time updates ผ่าน WebSocket

### ติดตั้ง ActionCable สำหรับ Subscriptions

```ruby
# Gemfile
gem 'graphql'
gem 'redis'
```

### Schema สำหรับ Subscription

```ruby
# app/graphql/graphql_api_schema.rb
class GraphqlApiSchema < GraphQL::Schema
  use GraphQL::Subscriptions::ActionCableSubscriptions
  
  mutation(Types::MutationType)
  query(Types::QueryType)
  subscription(Types::SubscriptionType)
end
```

### Subscription Type

```ruby
# app/graphql/types/subscription_type.rb
module Types
  class SubscriptionType < Types::BaseObject
    description "Root subscription type"
    
    field :post_created, Types::PostType, null: false,
          description: "รับการแจ้งเตือนเมื่อมีบทความใหม่" do
      argument :author_id, ID, required: false
    end
    
    field :comment_added, Types::CommentType, null: false,
          description: "รับการแจ้งเตือนเมื่อมี comment ใหม่" do
      argument :post_id, ID, required: true
    end
    
    field :post_liked, Types::PostType, null: false,
          description: "รับการแจ้งเตือนเมื่อบทความได้รับ like" do
      argument :post_id, ID, required: true
    end
    
    def post_created(author_id: nil)
      object
    end
    
    def comment_added(post_id:)
      object
    end
    
    def post_liked(post_id:)
      object
    end
  end
end
```

### Trigger Subscriptions จาก Mutations

```ruby
# app/graphql/mutations/create_post.rb
module Mutations
  class CreatePost < BaseMutation
    def resolve(input:)
      require_authentication!
      
      post = current_user.posts.build(input.to_h)
      
      if post.save
        # Trigger subscription
        GraphqlApiSchema.subscriptions.trigger(:post_created, {}, post)
        
        if input[:author_id]
          GraphqlApiSchema.subscriptions.trigger(
            :post_created, 
            { author_id: post.user_id.to_s }, 
            post
          )
        end
        
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    end
  end
end
```

### ActionCable Channel

```ruby
# app/channels/graphql_channel.rb
class GraphqlChannel < ActionCable::Channel::Base
  def subscribed
    @subscription_ids = []
  end
  
  def execute(data)
    query = data["query"]
    variables = ensure_hash(data["variables"])
    operation_name = data["operationName"]
    context = {
      current_user: current_user,
      channel: self
    }
    
    result = GraphqlApiSchema.execute(
      query,
      context: context,
      variables: variables,
      operation_name: operation_name
    )
    
    payload = {
      result: result.to_h,
      more: result.subscription?
    }
    
    if result.context[:subscription_id]
      @subscription_ids << result.context[:subscription_id]
    end
    
    transmit(payload)
  end
  
  def unsubscribed
    @subscription_ids.each do |sid|
      GraphqlApiSchema.subscriptions.delete_subscription(sid)
    end
  end
  
  private
  
  def current_user
    token = connection.request.headers['Authorization']&.split(' ')&.last
    return nil unless token
    
    decoded = JWT.decode(token, Rails.application.credentials.secret_key_base, true, algorithm: 'HS256')
    User.find(decoded[0]['user_id'])
  rescue
    nil
  end
  
  def ensure_hash(ambiguous_param)
    case ambiguous_param
    when String
      ambiguous_param.present? ? JSON.parse(ambiguous_param) : {}
    when Hash, ActionController::Parameters
      ambiguous_param
    when nil
      {}
    end
  end
end
```

---

## ขั้นตอนที่ 1233: Authentication กับ GraphQL

### JWT Authentication

```ruby
# app/graphql/graphql_controller.rb
class GraphqlController < ApplicationController
  def execute
    variables = prepare_variables(params[:variables])
    query = params[:query]
    operation_name = params[:operationName]
    
    context = {
      current_user: authenticate_user,
      request: request
    }
    
    result = GraphqlApiSchema.execute(
      query,
      variables: variables,
      context: context,
      operation_name: operation_name
    )
    
    render json: result
  end
  
  private
  
  def authenticate_user
    header = request.headers['Authorization']
    return nil unless header
    
    token = header.split(' ').last
    decode_token(token)
  rescue JWT::ExpiredSignature
    raise GraphQL::ExecutionError, "Token หมดอายุแล้ว กรุณาเข้าสู่ระบบใหม่"
  rescue JWT::DecodeError
    raise GraphQL::ExecutionError, "Token ไม่ถูกต้อง"
  end
  
  def decode_token(token)
    secret = Rails.application.credentials.secret_key_base
    decoded = JWT.decode(token, secret, true, algorithm: 'HS256')
    user_id = decoded[0]['user_id']
    User.find(user_id)
  rescue ActiveRecord::RecordNotFound
    nil
  end
end
```

### Authorization ใน Types

```ruby
# app/graphql/types/user_type.rb
module Types
  class UserType < Types::BaseObject
    # Field-level authorization
    field :email, String, null: true
    
    def email
      # เฉพาะเจ้าของ หรือ admin เท่านั้น
      return object.email if context[:current_user]&.id == object.id
      return object.email if context[:current_user]&.admin?
      nil
    end
    
    # Object-level authorization
    def self.authorized?(object, context)
      super && (context[:current_user].present? || object.public_profile?)
    end
  end
end
```

### GraphQL Guard Pattern

```ruby
# app/graphql/concerns/authenticate_user.rb
module AuthenticateUser
  def current_user
    context[:current_user]
  end
  
  def authenticate!
    raise GraphQL::ExecutionError, "กรุณาเข้าสู่ระบบก่อน" unless current_user
  end
  
  def authorize!(object, permission)
    unless Pundit.policy(current_user, object).public_send("#{permission}?")
      raise GraphQL::ExecutionError, "คุณไม่มีสิทธิ์ดำเนินการนี้"
    end
  end
end

# ใช้ใน Mutation
module Mutations
  class DeletePost < BaseMutation
    include AuthenticateUser
    
    def resolve(id:)
      authenticate!
      post = Post.find(id)
      authorize!(post, :destroy)
      
      post.destroy
      { success: true }
    end
  end
end
```

---

## ขั้นตอนที่ 1234: N+1 Problem และ graphql-batch

### ปัญหา N+1 ใน GraphQL

```graphql
# Query นี้จะเกิด N+1 ถ้าไม่จัดการ
query {
  posts {
    nodes {
      title
      author {  # โหลด user ทุก post แยกกัน!
        name
      }
    }
  }
}
```

### ติดตั้ง graphql-batch

```bash
gem 'graphql-batch'
bundle install
```

```ruby
# app/graphql/graphql_api_schema.rb
class GraphqlApiSchema < GraphQL::Schema
  use GraphQL::Batch
  # ...
end
```

### สร้าง Loaders

```ruby
# app/graphql/loaders/record_loader.rb
class RecordLoader < GraphQL::Batch::Loader
  def initialize(model)
    @model = model
  end
  
  def perform(ids)
    @model.where(id: ids).each do |record|
      fulfill(record.id, record)
    end
    
    ids.each do |id|
      fulfill(id, nil) unless fulfilled?(id)
    end
  end
end
```

```ruby
# app/graphql/loaders/association_loader.rb
class AssociationLoader < GraphQL::Batch::Loader
  def initialize(model, association_name)
    @model = model
    @association_name = association_name
    validate
  end
  
  def load(record)
    raise TypeError, "#{@model} loader can't load association for #{record.class}" unless record.is_a?(@model)
    super
  end
  
  def cache_key(record)
    record.object_id
  end
  
  def perform(records)
    preload_association(records)
    records.each { |r| fulfill(r, read_association(r)) }
  end
  
  private
  
  def validate
    @model.reflect_on_association(@association_name) ||
      raise("No association #{@association_name} on #{@model}")
  end
  
  def preload_association(records)
    ::ActiveRecord::Associations::Preloader.new(
      records: records,
      associations: @association_name
    ).call
  end
  
  def read_association(record)
    record.public_send(@association_name)
  end
end
```

```ruby
# app/graphql/loaders/count_loader.rb
class CountLoader < GraphQL::Batch::Loader
  def initialize(model, foreign_key)
    @model = model
    @foreign_key = foreign_key
  end
  
  def perform(ids)
    counts = @model.where(@foreign_key => ids)
                   .group(@foreign_key)
                   .count
    
    ids.each do |id|
      fulfill(id, counts[id] || 0)
    end
  end
end
```

### ใช้ Loader ใน Type

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    field :author, Types::UserType, null: false
    field :comments, [Types::CommentType], null: false
    field :comments_count, Integer, null: false
    
    def author
      # แทนที่จะ object.user (N+1)
      RecordLoader.for(User).load(object.user_id)
    end
    
    def comments
      AssociationLoader.for(Post, :comments).load(object)
    end
    
    def comments_count
      CountLoader.for(Comment, :post_id).load(object.id)
    end
  end
end
```

---

## ขั้นตอนที่ 1235: Connections และ Pagination

### Connection Type

```ruby
# app/graphql/types/post_connection_type.rb
module Types
  class PostConnectionType < Types::BaseConnection
    description "Connection สำหรับ posts"
    
    edge_type(Types::PostEdgeType)
    
    field :total_count, Integer, null: false, description: "จำนวนทั้งหมด"
    
    def total_count
      object.items.size
    end
  end
end
```

```ruby
# app/graphql/types/post_edge_type.rb
module Types
  class PostEdgeType < Types::BaseEdge
    node_type(Types::PostType)
    
    field :cursor, String, null: false
    field :node, Types::PostType, null: true
  end
end
```

### การใช้งาน Connection

```graphql
query {
  posts(first: 10, after: "cursor_here") {
    nodes {
      id
      title
    }
    pageInfo {
      hasNextPage
      hasPreviousPage
      startCursor
      endCursor
    }
    totalCount
  }
}
```

---

## ขั้นตอนที่ 1236: Error Handling

```ruby
# app/graphql/types/base_mutation.rb
module Mutations
  class BaseMutation < GraphQL::Schema::Mutation
    # Standard error fields
    field :errors, [String], null: false
    field :user_errors, [Types::UserErrorType], null: false
    
    def resolve(**args)
      perform(**args)
    rescue ActiveRecord::RecordInvalid => e
      { errors: e.record.errors.full_messages, user_errors: [] }
    rescue GraphQL::ExecutionError => e
      raise e
    rescue => e
      Rails.logger.error(e)
      raise GraphQL::ExecutionError, "เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง"
    end
  end
end
```

```ruby
# app/graphql/types/user_error_type.rb
module Types
  class UserErrorType < Types::BaseObject
    description "ข้อผิดพลาดจาก User input"
    
    field :path, [String], null: true, description: "Path ของ field ที่มีปัญหา"
    field :message, String, null: false, description: "ข้อความ error"
  end
end
```

---

## ขั้นตอนที่ 1237: GraphiQL Playground

GraphiQL คือ web IDE สำหรับทดสอบ GraphQL API

### การตั้งค่า

```ruby
# config/routes.rb
Rails.application.routes.draw do
  if Rails.env.development?
    mount GraphiQL::Rails::Engine, at: "/graphiql", graphql_path: "/graphql"
  end
end
```

```ruby
# config/initializers/graphiql.rb
GraphiQL::Rails.config.initial_query = <<~GRAPHQL
  # ยินดีต้อนรับสู่ GraphiQL!
  # ลองรัน query นี้:
  query {
    me {
      id
      name
      email
    }
  }
GRAPHQL

GraphiQL::Rails.config.headers = {
  'Authorization' => proc { |_view| "Bearer #{session[:token]}" }
}
```

### ตัวอย่าง Queries ที่ทดสอบได้

```graphql
# ดึงข้อมูล user ปัจจุบัน
query GetMe {
  me {
    id
    name
    email
    posts {
      id
      title
      status
    }
  }
}

# ดึงรายการบทความ
query GetPosts($first: Int, $after: String) {
  posts(first: $first, after: $after, status: PUBLISHED) {
    nodes {
      id
      title
      excerpt
      author {
        name
      }
      tags {
        name
      }
      commentsCount
      likesCount
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

# สร้างบทความใหม่
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    post {
      id
      title
      slug
    }
    errors
  }
}

# Variables สำหรับ mutation
# {
#   "input": {
#     "title": "บทความแรกของฉัน",
#     "body": "เนื้อหาของบทความ...",
#     "status": "PUBLISHED"
#   }
# }
```

---

## ขั้นตอนที่ 1238: Testing GraphQL

```ruby
# spec/graphql/queries/posts_spec.rb
require 'rails_helper'

RSpec.describe "Posts Query", type: :request do
  let(:user) { create(:user) }
  let!(:posts) { create_list(:post, 3, user: user, status: 'published') }
  
  let(:query) do
    <<~GRAPHQL
      query GetPosts($first: Int) {
        posts(first: $first, status: PUBLISHED) {
          nodes {
            id
            title
          }
          totalCount
        }
      }
    GRAPHQL
  end
  
  it "ดึงรายการบทความได้" do
    post "/graphql", params: {
      query: query,
      variables: { first: 10 }
    }
    
    data = JSON.parse(response.body)
    expect(data['errors']).to be_nil
    expect(data['data']['posts']['nodes'].length).to eq(3)
    expect(data['data']['posts']['totalCount']).to eq(3)
  end
end
```

```ruby
# spec/graphql/mutations/create_post_spec.rb
require 'rails_helper'

RSpec.describe "CreatePost Mutation", type: :request do
  let(:user) { create(:user) }
  let(:token) { generate_token(user) }
  
  let(:mutation) do
    <<~GRAPHQL
      mutation CreatePost($input: CreatePostInput!) {
        createPost(input: $input) {
          post {
            id
            title
            status
          }
          errors
        }
      }
    GRAPHQL
  end
  
  context "เมื่อ login แล้ว" do
    it "สร้างบทความได้" do
      post "/graphql",
           params: {
             query: mutation,
             variables: {
               input: {
                 title: "Test Post",
                 body: "Test body",
                 status: "PUBLISHED"
               }
             }
           },
           headers: { 'Authorization' => "Bearer #{token}" }
      
      data = JSON.parse(response.body)
      expect(data['errors']).to be_nil
      expect(data['data']['createPost']['post']['title']).to eq("Test Post")
      expect(data['data']['createPost']['errors']).to be_empty
    end
  end
  
  context "เมื่อยังไม่ได้ login" do
    it "ไม่สามารถสร้างบทความได้" do
      post "/graphql",
           params: {
             query: mutation,
             variables: {
               input: {
                 title: "Test Post",
                 body: "Test body"
               }
             }
           }
      
      data = JSON.parse(response.body)
      expect(data['errors']).not_to be_nil
    end
  end
end
```

---

## ขั้นตอนที่ 1239: Connection Pagination แบบ Cursor-based

```ruby
# app/graphql/types/page_info_type.rb
module Types
  class PageInfoType < Types::BaseObject
    description "ข้อมูล pagination"
    
    field :has_next_page, Boolean, null: false
    field :has_previous_page, Boolean, null: false
    field :start_cursor, String, null: true
    field :end_cursor, String, null: true
  end
end
```

### Keyset Pagination

```ruby
# app/graphql/connections/keyset_connection.rb
class KeysetConnection < GraphQL::Pagination::Connection
  def nodes
    # Apply cursor-based pagination
    items = if after_cursor
              object.where("id > ?", decode_cursor(after_cursor))
            elsif before_cursor
              object.where("id < ?", decode_cursor(before_cursor))
            else
              object
            end
    
    items = items.limit(page_size + 1)
    @has_next_page = items.size > page_size
    items.first(page_size)
  end
  
  def has_next_page
    nodes
    @has_next_page
  end
  
  private
  
  def decode_cursor(cursor)
    Base64.decode64(cursor).to_i
  end
  
  def encode_cursor(id)
    Base64.encode64(id.to_s)
  end
end
```

---

## ขั้นตอนที่ 1240: Rate Limiting สำหรับ GraphQL

```ruby
# app/graphql/graphql_api_schema.rb
class GraphqlApiSchema < GraphQL::Schema
  # Complexity limiting
  max_complexity 300
  max_depth 10
  
  # Query timeout
  timeout_max_wait_seconds 10
  
  # Custom complexity
  default_max_page_size 100
end
```

```ruby
# app/graphql/types/base_field.rb
module Types
  class BaseField < GraphQL::Schema::Field
    # เพิ่ม complexity สำหรับ connection fields
    def initialize(*args, **kwargs, &block)
      super
      if connection?
        self.complexity = ->(ctx, args, child_complexity) {
          page_size = args[:first] || args[:last] || 25
          page_size * child_complexity
        }
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1241: GraphQL Fragments

Fragment ช่วยให้ใช้ชุด fields ซ้ำได้

```graphql
# กำหนด Fragment
fragment PostBasicFields on Post {
  id
  title
  slug
  createdAt
}

fragment PostFullFields on Post {
  ...PostBasicFields
  body
  excerpt
  status
  author {
    id
    name
    avatarUrl
  }
  tags {
    id
    name
  }
}

# ใช้งาน Fragment
query GetPosts {
  posts(first: 10) {
    nodes {
      ...PostFullFields
    }
  }
}

query GetPost($id: ID!) {
  post(id: $id) {
    ...PostFullFields
    comments {
      id
      body
      author {
        name
      }
    }
  }
}
```

---

## ขั้นตอนที่ 1242: Directives

Directives ควบคุมการทำงานของ query

```graphql
# @include - แสดง field ถ้า condition เป็น true
query GetUser($showEmail: Boolean!) {
  me {
    name
    email @include(if: $showEmail)
  }
}

# @skip - ข้าม field ถ้า condition เป็น true
query GetPost($includeComments: Boolean!) {
  post(id: "1") {
    title
    comments @skip(if: $includeComments) {
      body
    }
  }
}

# @deprecated - บอกว่า field นี้ deprecated
type Post {
  oldField: String @deprecated(reason: "ใช้ newField แทน")
  newField: String
}
```

### Custom Directive

```ruby
# app/graphql/directives/format_date_directive.rb
class Directives::FormatDateDirective < GraphQL::Schema::Directive
  description "จัดรูปแบบวันที่"
  locations FIELD
  
  argument :format, String, required: false, default_value: "%Y-%m-%d"
  
  def self.resolve(obj, args, ctx)
    value = yield
    if value.respond_to?(:strftime)
      value.strftime(args[:format])
    else
      value
    end
  end
end
```

---

## ขั้นตอนที่ 1243: Introspection

Introspection ช่วยให้ query Schema ได้

```graphql
# ดู types ทั้งหมด
{
  __schema {
    types {
      name
      kind
      description
    }
  }
}

# ดู fields ของ type
{
  __type(name: "Post") {
    name
    fields {
      name
      type {
        name
        kind
      }
      description
    }
  }
}

# ดู queries ที่มี
{
  __schema {
    queryType {
      fields {
        name
        description
      }
    }
  }
}
```

### ปิด Introspection ใน Production

```ruby
# app/graphql/graphql_api_schema.rb
class GraphqlApiSchema < GraphQL::Schema
  disable_introspection_entry_points if Rails.env.production?
end
```

---

## ขั้นตอนที่ 1244: File Upload กับ GraphQL

```ruby
# Gemfile
gem 'apollo-upload-server'
```

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  include ApolloUploadServer::Middleware
end
```

```ruby
# app/graphql/types/file_upload_type.rb
module Types
  class FileUploadType < GraphQL::Schema::Scalar
    description "A file upload"
    
    def self.coerce_input(value, context)
      value
    end
    
    def self.coerce_result(value, context)
      value
    end
  end
end
```

```ruby
# app/graphql/mutations/upload_avatar.rb
module Mutations
  class UploadAvatar < BaseMutation
    argument :avatar, Types::FileUploadType, required: true
    
    field :user, Types::UserType, null: true
    field :errors, [String], null: false
    
    def resolve(avatar:)
      require_authentication!
      
      current_user.avatar.attach(avatar)
      
      if current_user.save
        { user: current_user, errors: [] }
      else
        { user: nil, errors: current_user.errors.full_messages }
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1245: Deploy GraphQL API

### การตั้งค่าสำหรับ Production

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # GraphiQL เฉพาะ development
  if Rails.env.development?
    mount GraphiQL::Rails::Engine, at: "/graphiql", graphql_path: "/graphql"
  end
  
  post "/graphql", to: "graphql#execute"
end
```

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins Rails.env.production? ? 'https://yourdomain.com' : '*'
    
    resource '/graphql',
             headers: :any,
             methods: [:get, :post, :options],
             credentials: true
  end
end
```

```bash
# Heroku deployment
heroku create my-graphql-api
heroku addons:create heroku-postgresql
heroku addons:create heroku-redis

git push heroku main
heroku run rails db:migrate
heroku run rails db:seed
```

---

## แบบฝึกหัดที่ 1-25

### แบบฝึกหัดที่ 1
สร้าง GraphQL API สำหรับระบบ Blog พื้นฐาน มี User, Post, Comment

```
เป้าหมาย:
- สร้าง Schema ด้วย graphql-ruby
- กำหนด Types ให้ครบถ้วน  
- สร้าง Queries พื้นฐาน
```

### แบบฝึกหัดที่ 2
เพิ่ม Authentication ด้วย JWT

```
เป้าหมาย:
- สร้าง SignIn Mutation
- สร้าง SignUp Mutation
- ป้องกัน Mutations ที่ต้องการ authentication
```

### แบบฝึกหัดที่ 3
สร้าง CRUD Mutations สำหรับ Post

```ruby
# ต้องสร้าง:
mutation CreatePost($input: CreatePostInput!) { ... }
mutation UpdatePost($id: ID!, $input: UpdatePostInput!) { ... }
mutation DeletePost($id: ID!) { ... }
```

### แบบฝึกหัดที่ 4
เพิ่ม Pagination ด้วย Cursor-based pagination

```graphql
query {
  posts(first: 10, after: "cursor") {
    nodes { ... }
    pageInfo { hasNextPage, endCursor }
    totalCount
  }
}
```

### แบบฝึกหัดที่ 5
แก้ปัญหา N+1 ด้วย graphql-batch

```ruby
# สร้าง Loaders:
# - RecordLoader สำหรับโหลด User จาก ID
# - AssociationLoader สำหรับโหลด tags ของ Post
# - CountLoader สำหรับนับ comments
```

### แบบฝึกหัดที่ 6
สร้าง Search Query ที่ค้นหาได้หลายประเภท

```graphql
query Search($query: String!) {
  search(query: $query) {
    ... on Post { id, title }
    ... on User { id, name }
    ... on Tag { id, name }
  }
}
```

### แบบฝึกหัดที่ 7
เพิ่ม Enum Types สำหรับ Post status และ sort options

### แบบฝึกหัดที่ 8
สร้าง Interface สำหรับ Node และ Timestampable

### แบบฝึกหัดที่ 9
เพิ่ม File Upload สำหรับรูป avatar

### แบบฝึกหัดที่ 10
สร้าง Subscription สำหรับ real-time comments

```graphql
subscription OnCommentAdded($postId: ID!) {
  commentAdded(postId: $postId) {
    id
    body
    author { name }
  }
}
```

### แบบฝึกหัดที่ 11
เพิ่ม Rate Limiting และ Max Complexity

```ruby
# ตั้งค่า:
max_complexity 200
max_depth 8
```

### แบบฝึกหัดที่ 12
สร้าง Custom Scalar สำหรับ Date และ URL

### แบบฝึกหัดที่ 13
เขียน RSpec tests สำหรับ Queries ทั้งหมด

### แบบฝึกหัดที่ 14
เขียน RSpec tests สำหรับ Mutations ทั้งหมด

### แบบฝึกหัดที่ 15
เพิ่ม Authorization ด้วย Pundit ใน GraphQL

```ruby
# ต้องตรวจสอบสิทธิ์:
# - ใครสามารถลบ Post ได้
# - ใครสามารถดู email ได้
# - ใครสามารถ update Post ได้
```

### แบบฝึกหัดที่ 16
สร้าง GraphQL API สำหรับระบบ E-commerce

```
Models: Product, Cart, Order, OrderItem
Queries: products, cart, orders
Mutations: addToCart, checkout, cancelOrder
```

### แบบฝึกหัดที่ 17
เพิ่ม Error Handling ที่ดีสำหรับทุก Mutation

### แบบฝึกหัดที่ 18
Implement Cursor Pagination แบบ Custom

### แบบฝึกหัดที่ 19
สร้าง Admin-only Queries และ Mutations

### แบบฝึกหัดที่ 20
เพิ่ม Logging และ Performance Monitoring ให้ GraphQL

```ruby
# สร้าง Tracer:
class GraphqlTracer
  def platform_trace(platform_key, key, data)
    Rails.logger.info "[GraphQL] #{key}: #{data}"
    yield
  end
end
```

### แบบฝึกหัดที่ 21
Implement DataLoader สำหรับ Social Network

```
- Loader สำหรับ followers/following counts
- Loader สำหรับ mutual friends
- Loader สำหรับ post stats
```

### แบบฝึกหัดที่ 22
สร้าง GraphQL API Versioning Strategy

### แบบฝึกหัดที่ 23
Deploy GraphQL API บน Heroku พร้อม ActionCable

### แบบฝึกหัดที่ 24
สร้าง GraphQL Subscription สำหรับ Notification System

```graphql
subscription {
  notificationReceived {
    id
    type
    message
    createdAt
  }
}
```

### แบบฝึกหัดที่ 25
สร้าง Full-featured Blog API ด้วย GraphQL:
- Authentication/Authorization
- CRUD Posts
- Comments System
- Tags/Categories
- Search (Full-text)
- Real-time Subscriptions
- File Uploads
- Pagination
- Rate Limiting
- Tests

---

## สรุป Part 56

ในส่วนนี้เราได้เรียนรู้:
1. GraphQL คืออะไรและแตกต่างจาก REST อย่างไร
2. การติดตั้งและตั้งค่า graphql-ruby
3. Types ต่างๆ: Object, Scalar, Enum, Input, Union, Interface
4. การสร้าง Queries และ Mutations
5. Real-time ด้วย Subscriptions
6. Authentication/Authorization ใน GraphQL
7. การแก้ปัญหา N+1 ด้วย graphql-batch
8. GraphiQL playground
9. Testing GraphQL APIs
10. การ Deploy

GraphQL เป็นเครื่องมือที่ทรงพลังสำหรับการสร้าง API ที่ยืดหยุ่น แต่ต้องใช้ความเข้าใจที่ดีในการจัดการปัญหาต่างๆ เช่น N+1, Authorization, และ Rate Limiting
