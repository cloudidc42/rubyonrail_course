# Part 47: JSON และ Serialization ใน Rails

## ขั้นตอนที่ 1036-1055: การจัดการ JSON Data

---

## ขั้นตอนที่ 1036: JSON Basics ใน Rails

```ruby
# to_json - แปลง object เป็น JSON string
user = User.find(1)
user.to_json  
# => '{"id":1,"name":"สมชาย","email":"somchai@example.com",...}'

# ควบคุม fields ที่แสดง
user.to_json(only: [:id, :name])
# => '{"id":1,"name":"สมชาย"}'

user.to_json(except: [:password_digest, :remember_digest])

# เพิ่ม methods
user.to_json(methods: [:full_name, :avatar_url])

# Include associations
user.to_json(include: :posts)
user.to_json(include: { posts: { only: [:id, :title] } })
```

```ruby
# as_json - แปลงเป็น Ruby Hash (ก่อน serialize)
user.as_json  
# => {"id"=>1, "name"=>"สมชาย", ...}

user.as_json(only: [:id, :name, :email])

# Collection
users = User.all
users.as_json  # Array of hashes
users.to_json  # JSON string ของ array

# render json ใน controller
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    render json: @user.as_json(
      only: [:id, :name, :email],
      methods: [:avatar_url],
      include: { posts: { only: [:id, :title, :published] } }
    )
  end
end
```

## ขั้นตอนที่ 1037: Override as_json

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def as_json(options = {})
    super(options.merge(
      only: [:id, :name, :email, :created_at],
      methods: [:avatar_url, :posts_count],
      except: [:password_digest, :remember_digest, :reset_digest]
    ))
  end
  
  def avatar_url
    if avatar.attached?
      Rails.application.routes.url_helpers.url_for(avatar)
    else
      "https://ui-avatars.com/api/?name=#{URI.encode_www_form_component(name)}&background=random"
    end
  end
  
  def posts_count
    posts.published.count
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  def as_json(options = {})
    base = super(options.merge(
      only: [:id, :title, :content, :published, :created_at, :updated_at],
      methods: [:excerpt, :reading_time]
    ))
    
    # เพิ่ม computed fields
    base.merge(
      'author' => user&.as_json(only: [:id, :name]),
      'tags' => tags.map { |t| t.as_json(only: [:id, :name]) }
    )
  end
  
  def excerpt
    content&.truncate(200)
  end
  
  def reading_time
    words = content.to_s.split.length
    (words / 200.0).ceil
  end
end
```

## ขั้นตอนที่ 1038: to_xml

```ruby
# แปลงเป็น XML
user.to_xml
# => <?xml version="1.0" encoding="UTF-8"?>
#    <user>
#      <id type="integer">1</id>
#      <name>สมชาย</name>
#      ...
#    </user>

# ควบคุม output
user.to_xml(
  only: [:id, :name],
  root: 'member',
  skip_types: true,
  indent: 2
)

# Include associations
user.to_xml(include: :posts)

# Custom builder
user.to_xml do |xml|
  xml.profile do
    xml.name user.name
    xml.email user.email
    xml.joined user.created_at.strftime("%Y-%m-%d")
  end
end
```

## ขั้นตอนที่ 1039: Jbuilder Views

Jbuilder เป็น gem ที่มาพร้อมกับ Rails ใช้สร้าง JSON ใน view templates

```ruby
# Gemfile (มาพร้อม Rails แล้ว)
gem 'jbuilder'
```

```ruby
# app/views/api/v1/posts/index.json.jbuilder
json.posts @posts do |post|
  json.id post.id
  json.title post.title
  json.excerpt post.content.truncate(200)
  json.published post.published
  json.created_at post.created_at.iso8601
  
  json.author do
    json.id post.user.id
    json.name post.user.name
    json.avatar_url post.user.avatar_url
  end
  
  json.tags post.tags, :id, :name
  
  json.url api_v1_post_url(post)
end

json.meta do
  json.total @posts.total_count
  json.pages @posts.total_pages
  json.current_page @posts.current_page
end
```

```ruby
# app/views/api/v1/posts/show.json.jbuilder
json.id @post.id
json.title @post.title
json.content @post.content
json.published @post.published
json.published_at @post.published_at&.iso8601
json.created_at @post.created_at.iso8601
json.updated_at @post.updated_at.iso8601
json.reading_time @post.reading_time
json.views_count @post.views_count
json.likes_count @post.likes_count

json.author do
  json.id @post.user.id
  json.name @post.user.name
  json.bio @post.user.bio
  json.avatar_url @post.user.avatar_url
  json.posts_count @post.user.posts.published.count
end

json.tags @post.tags do |tag|
  json.id tag.id
  json.name tag.name
  json.slug tag.slug
  json.posts_count tag.posts.published.count
end

json.comments @post.comments.approved.recent.limit(10) do |comment|
  json.id comment.id
  json.body comment.body
  json.created_at comment.created_at.iso8601
  
  json.user do
    json.id comment.user.id
    json.name comment.user.name
    json.avatar_url comment.user.avatar_url
  end
end

json.related_posts @post.related_posts.limit(5) do |related|
  json.id related.id
  json.title related.title
  json.excerpt related.content.truncate(150)
  json.url api_v1_post_url(related)
end
```

```ruby
# Jbuilder partials
# app/views/api/v1/posts/_post.json.jbuilder
json.cache! ['v1', post], expires_in: 10.minutes do
  json.id post.id
  json.title post.title
  json.excerpt post.content.truncate(200)
  json.author post.user.name
  json.url api_v1_post_url(post)
end
```

```ruby
# ใช้ partial
# app/views/api/v1/posts/index.json.jbuilder
json.posts @posts do |post|
  json.partial! post
end
```

## ขั้นตอนที่ 1040: Blueprinter Gem

Blueprinter เป็น serialization library ที่เร็วและง่ายต่อการใช้

```ruby
# Gemfile
gem 'blueprinter'
```

```ruby
# app/blueprints/user_blueprint.rb
class UserBlueprint < Blueprinter::Base
  identifier :id
  
  # Default view
  fields :name, :email, :created_at
  
  field :avatar_url do |user, _options|
    user.avatar_url
  end
  
  # Normal view
  view :normal do
    fields :name, :email, :bio, :created_at
  end
  
  # Extended view (includes more data)
  view :extended do
    include_view :normal
    
    fields :phone, :timezone, :locale
    
    field :posts_count do |user|
      user.posts.published.count
    end
    
    association :posts, blueprint: PostBlueprint, view: :summary
  end
  
  # Admin view (everything)
  view :admin do
    include_view :extended
    fields :role, :last_sign_in_at, :sign_in_count, :active
  end
end
```

```ruby
# app/blueprints/post_blueprint.rb
class PostBlueprint < Blueprinter::Base
  identifier :id
  
  view :summary do
    fields :title, :excerpt, :published_at, :reading_time
    
    field :excerpt do |post|
      post.content.truncate(200)
    end
    
    field :url do |post, options|
      options[:url_helpers].api_v1_post_url(post)
    end
  end
  
  view :normal do
    include_view :summary
    fields :content, :views_count, :likes_count
    
    association :user, blueprint: UserBlueprint, view: :normal
    association :tags, blueprint: TagBlueprint
  end
  
  view :full do
    include_view :normal
    
    association :comments, blueprint: CommentBlueprint do |post|
      post.comments.approved.order(created_at: :desc).limit(20)
    end
  end
end
```

```ruby
# การใช้งาน Blueprinter ใน controller
class Api::V1::PostsController < ApplicationController
  def index
    @posts = Post.published.page(params[:page])
    
    render json: PostBlueprint.render_as_hash(
      @posts,
      view: :normal,
      root: :posts,
      url_helpers: Rails.application.routes.url_helpers
    )
  end
  
  def show
    @post = Post.find(params[:id])
    
    render json: PostBlueprint.render_as_hash(
      @post,
      view: :full,
      url_helpers: Rails.application.routes.url_helpers
    )
  end
end
```

## ขั้นตอนที่ 1041: Fast JSONAPI (jsonapi-serializer)

```ruby
# Gemfile
gem 'jsonapi-serializer'
```

```ruby
# app/serializers/post_serializer.rb
class PostSerializer
  include JSONAPI::Serializer
  
  set_type :post
  set_id :id
  
  attributes :title, :content, :published
  
  # Computed attributes
  attribute :excerpt do |object|
    object.content&.truncate(200)
  end
  
  attribute :reading_time do |object|
    (object.content.to_s.split.length / 200.0).ceil
  end
  
  attribute :url do |object, params|
    params[:url_helpers].api_v1_post_url(object) if params[:url_helpers]
  end
  
  # Conditional attributes
  attribute :admin_notes do |object, params|
    object.admin_notes if params[:current_user]&.admin?
  end
  
  # Relationships
  belongs_to :user
  has_many :tags
  has_many :comments do |object|
    object.comments.approved.limit(10)
  end
  
  # Links
  link :self do |object|
    Rails.application.routes.url_helpers.api_v1_post_url(object)
  end
  
  # Meta
  meta do |object, params|
    {
      likes_count: object.likes.count,
      comments_count: object.comments.approved.count
    }
  end
end
```

```ruby
# app/serializers/user_serializer.rb
class UserSerializer
  include JSONAPI::Serializer
  
  set_type :user
  
  attributes :name, :created_at
  
  # ซ่อน email ตาม permission
  attribute :email do |object, params|
    object.email if params[:current_user] == object || params[:current_user]&.admin?
  end
  
  attribute :avatar_url do |object|
    object.avatar_url
  end
  
  has_many :posts do |object, params|
    scope = params[:include_drafts] ? object.posts : object.posts.published
    scope.order(created_at: :desc).limit(10)
  end
end
```

```ruby
# การใช้งานใน controller
class Api::V1::PostsController < ApplicationController
  def index
    @posts = Post.published
                 .includes(:user, :tags)
                 .order(created_at: :desc)
    
    # กำหนด params สำหรับ serializer
    serializer_params = {
      current_user: current_user,
      url_helpers: Rails.application.routes.url_helpers
    }
    
    render json: PostSerializer.new(
      @posts,
      include: [:user, :tags],
      params: serializer_params,
      meta: { total: @posts.count }
    ).serializable_hash
  end
  
  def show
    @post = Post.includes(:user, :tags, :comments).find(params[:id])
    
    render json: PostSerializer.new(
      @post,
      include: ['user', 'tags', 'comments.user'],
      params: { current_user: current_user }
    ).serializable_hash
  end
end
```

## ขั้นตอนที่ 1042: Custom Serializers (ไม่ใช้ gem)

```ruby
# app/serializers/base_serializer.rb
class BaseSerializer
  def initialize(resource, options = {})
    @resource = resource
    @options = options
  end
  
  def as_json
    raise NotImplementedError
  end
  
  def to_json
    as_json.to_json
  end
  
  private
  
  attr_reader :resource, :options
  
  def current_user
    options[:current_user]
  end
  
  def admin?
    current_user&.admin?
  end
end
```

```ruby
# app/serializers/post_detail_serializer.rb
class PostDetailSerializer < BaseSerializer
  def as_json
    {
      id: resource.id,
      type: "post",
      attributes: attributes,
      relationships: relationships,
      links: links,
      meta: meta
    }
  end
  
  private
  
  def attributes
    data = {
      title: resource.title,
      content: resource.content,
      excerpt: resource.content&.truncate(200),
      published: resource.published,
      reading_time: reading_time,
      created_at: resource.created_at.iso8601,
      updated_at: resource.updated_at.iso8601
    }
    
    # Admin-only fields
    data[:admin_notes] = resource.admin_notes if admin?
    data[:internal_id] = resource.internal_id if admin?
    
    data
  end
  
  def relationships
    {
      author: {
        data: { type: "user", id: resource.user_id },
        attributes: {
          name: resource.user.name,
          avatar_url: resource.user.avatar_url
        }
      },
      tags: resource.tags.map { |tag| 
        { id: tag.id, name: tag.name, slug: tag.slug }
      },
      comments: {
        data: resource.comments.approved.count,
        recent: resource.comments.approved.order(created_at: :desc).limit(3).map { |c|
          { id: c.id, body: c.body.truncate(100), user: c.user.name }
        }
      }
    }
  end
  
  def links
    {
      self: Rails.application.routes.url_helpers.api_v1_post_url(resource)
    }
  end
  
  def meta
    {
      views: resource.views_count,
      likes: resource.likes_count
    }
  end
  
  def reading_time
    words = resource.content.to_s.split.length
    "#{(words / 200.0).ceil} นาที"
  end
end
```

## ขั้นตอนที่ 1043: Serializing Associations

```ruby
# app/serializers/post_with_nested_serializer.rb
class PostWithNestedSerializer
  include JSONAPI::Serializer
  
  attributes :title, :content
  
  # Nested serializer สำหรับ belongs_to
  belongs_to :user, serializer: UserBasicSerializer
  
  # Nested serializer สำหรับ has_many
  has_many :tags, serializer: TagSerializer
  
  # Nested serializer แบบ conditional
  has_many :comments, serializer: CommentSerializer do |post, params|
    if params[:include_all_comments]
      post.comments.approved
    else
      post.comments.approved.order(created_at: :desc).limit(5)
    end
  end
  
  # Polymorphic association
  belongs_to :commentable, polymorphic: true
end
```

```ruby
# Serialize deeply nested data
class UserWithPostsSerializer
  include JSONAPI::Serializer
  
  attributes :name, :email
  
  has_many :posts do |user, params|
    user.posts.published.includes(:tags, :comments)
  end
end

# ใน controller
render json: UserWithPostsSerializer.new(
  user,
  include: ['posts', 'posts.tags', 'posts.comments']
).serializable_hash
```

## ขั้นตอนที่ 1044: Performance Tips สำหรับ Serialization

```ruby
# 1. Eager loading associations
class Api::V1::PostsController < ApplicationController
  def index
    @posts = Post.published
                 .includes(:user, :tags, :comments)  # ป้องกัน N+1
                 .page(params[:page])
    
    render json: PostSerializer.new(@posts).serializable_hash
  end
end

# 2. Caching serialized data
class PostSerializer
  include JSONAPI::Serializer
  
  cache_options store: Rails.cache, namespace: 'jsonapi', expires_in: 1.hour
  
  attributes :title, :content, :published_at
  belongs_to :user
  has_many :tags
end

# 3. Select เฉพาะ columns ที่ต้องการ
@posts = Post.select(:id, :title, :content, :user_id, :published_at)
             .published
             .page(params[:page])

# 4. ใช้ batch loading
# gem 'batch-loader'
class PostSerializer
  include JSONAPI::Serializer
  
  attribute :comments_count do |post|
    BatchLoader::GraphQL.for(post.id).batch do |post_ids, loader|
      Comment.where(post_id: post_ids)
             .group(:post_id)
             .count
             .each { |id, count| loader.call(id, count) }
    end
  end
end

# 5. Streaming large datasets
class Api::V1::ExportsController < ApplicationController
  def posts
    response.headers['Content-Type'] = 'application/json'
    response.headers['Transfer-Encoding'] = 'chunked'
    
    self.response_body = Enumerator.new do |y|
      y << '{"posts":['
      
      Post.published.find_each.with_index do |post, index|
        y << ',' if index > 0
        y << PostSerializer.new(post).to_json
      end
      
      y << ']}'
    end
  end
end
```

## ขั้นตอนที่ 1045: JSON Schema Validation

```ruby
# app/validators/json_schema_validator.rb
class JsonSchemaValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank?
    
    schema = options[:with]
    errors = JSON::Validator.fully_validate(schema, value)
    
    if errors.any?
      errors.each { |error| record.errors.add(attribute, error) }
    end
  end
end

# app/models/product.rb
class Product < ApplicationRecord
  METADATA_SCHEMA = {
    "type" => "object",
    "properties" => {
      "color" => { "type" => "string" },
      "size" => { "type" => "string", "enum" => ["S", "M", "L", "XL"] },
      "weight" => { "type" => "number", "minimum" => 0 }
    },
    "required" => ["color", "size"]
  }
  
  validates :metadata, json_schema: { with: METADATA_SCHEMA }
  
  store_accessor :metadata, :color, :size, :weight
end
```

## ขั้นตอนที่ 1046: JSON Columns ใน Database

```ruby
# Migration
class AddMetadataToProducts < ActiveRecord::Migration[7.0]
  def change
    add_column :products, :metadata, :jsonb, default: {}
    add_column :products, :attributes_data, :json
    
    # JSON index (PostgreSQL)
    add_index :products, :metadata, using: :gin
    add_index :products, "(metadata->>'color')"
    add_index :products, "((metadata->>'price')::numeric)"
  end
end
```

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  # JSON column accessors
  store_accessor :metadata, :color, :size, :weight, :dimensions
  
  # Serializers
  serialize :specifications, coder: JSON
  
  # Validations
  validates :color, presence: true
  validates :size, inclusion: { in: %w[XS S M L XL XXL] }
  
  # Scopes ที่ใช้ JSON column
  scope :red_products, -> { where("metadata->>'color' = ?", "red") }
  scope :large_sizes, -> { where("metadata->>'size' IN (?)", %w[L XL XXL]) }
  scope :heavy_items, -> { where("(metadata->>'weight')::float > ?", 5.0) }
end
```

```ruby
# Querying JSON columns
Post.where("settings->>'language' = ?", "th")
Post.where("(settings->'notifications')::boolean = true")
Post.where("tags @> ?", ["ruby", "rails"].to_json)  # JSONB contains
Post.where("extra_data ? :key", key: "special_field")  # key exists
```

## ขั้นตอนที่ 1047: JSON API Error Responses

```ruby
# Standard JSON:API error format
{
  "errors": [
    {
      "status": "422",
      "source": { "pointer": "/data/attributes/title" },
      "title": "Invalid Attribute",
      "detail": "Title ไม่สามารถเว้นว่างได้"
    },
    {
      "status": "422",
      "source": { "pointer": "/data/attributes/email" },
      "title": "Invalid Attribute",
      "detail": "Email รูปแบบไม่ถูกต้อง"
    }
  ]
}
```

```ruby
# app/services/error_serializer.rb
class ErrorSerializer
  def initialize(model)
    @model = model
  end
  
  def serialize
    {
      errors: @model.errors.map { |error| serialize_error(error) }
    }
  end
  
  private
  
  def serialize_error(error)
    {
      status: "422",
      source: { pointer: "/data/attributes/#{error.attribute}" },
      title: "Validation Error",
      detail: error.full_message
    }
  end
end
```

## ขั้นตอนที่ 1048: Blueprinter ขั้นสูง

```ruby
# Transformations
class PostBlueprint < Blueprinter::Base
  # เปลี่ยน field names
  transform do |hash|
    hash.transform_keys { |k| k.to_s.camelize(:lower) }
  end
  
  # นิยาม field แบบ dynamic
  view :with_metrics do
    include_view :normal
    
    dynamic_field :engagement_score do |post|
      (post.likes_count * 2 + post.comments.count * 3 + post.views_count).to_f / 
        [post.age_in_days, 1].max
    end
  end
end
```

```ruby
# Nested Blueprints
class OrderBlueprint < Blueprinter::Base
  identifier :id
  
  fields :status, :total_amount, :created_at
  
  association :user, blueprint: UserBlueprint
  association :items, blueprint: OrderItemBlueprint
  
  view :with_shipping do
    include_view :normal
    
    field :shipping_details do |order|
      {
        address: order.shipping_address,
        carrier: order.carrier,
        tracking_number: order.tracking_number,
        estimated_delivery: order.estimated_delivery&.strftime("%d/%m/%Y")
      }
    end
  end
end
```

## ขั้นตอนที่ 1049: Testing Serializers

```ruby
# spec/serializers/post_serializer_spec.rb
require 'rails_helper'

RSpec.describe PostSerializer do
  let(:user) { create(:user) }
  let(:post) { create(:post, :published, user: user) }
  let(:tags) { create_list(:tag, 3) }
  
  before { post.tags << tags }
  
  subject(:serialized) do
    described_class.new(post, params: { current_user: user }).serializable_hash
  end
  
  describe "attributes" do
    it "includes required attributes" do
      attrs = serialized[:data][:attributes]
      expect(attrs).to include(:title, :content, :published)
    end
    
    it "includes excerpt" do
      expect(serialized[:data][:attributes][:excerpt]).to eq(post.content.truncate(200))
    end
    
    it "includes reading_time" do
      expect(serialized[:data][:attributes][:reading_time]).to be_a(Integer)
    end
  end
  
  describe "relationships" do
    it "includes user relationship" do
      expect(serialized[:data][:relationships][:user]).to be_present
    end
    
    it "includes tags" do
      expect(serialized[:data][:relationships][:tags][:data].length).to eq(3)
    end
  end
  
  describe "conditional attributes" do
    context "ผู้ใช้ทั่วไป" do
      it "ไม่รวม admin_notes" do
        expect(serialized[:data][:attributes]).not_to have_key(:admin_notes)
      end
    end
    
    context "admin" do
      let(:admin) { create(:user, :admin) }
      
      subject(:serialized) do
        described_class.new(post, params: { current_user: admin }).serializable_hash
      end
      
      it "รวม admin_notes" do
        expect(serialized[:data][:attributes]).to have_key(:admin_notes)
      end
    end
  end
end
```

## ขั้นตอนที่ 1050: Jbuilder Best Practices

```ruby
# ใช้ partial สำหรับ reusability
# app/views/api/v1/shared/_user.json.jbuilder
json.id user.id
json.name user.name
json.avatar_url user.avatar_url

# app/views/api/v1/posts/show.json.jbuilder
json.id @post.id
json.title @post.title

json.author do
  json.partial! 'api/v1/shared/user', user: @post.user
end

# ใช้ cache
json.cache! [@post, 'v2'], expires_in: 5.minutes do
  json.id @post.id
  json.title @post.title
  json.content @post.content
end

# Conditional rendering
json.secret_key @post.secret_key if current_user&.admin?

# Null handling
json.published_at @post.published_at&.iso8601
```

## ขั้นตอนที่ 1051: JSONB PostgreSQL

```ruby
# การค้นหาใน JSONB
# ค้นหา records ที่มี key
Post.where("settings ? 'dark_mode'")

# ค้นหาค่าใน JSONB
Post.where("settings ->> 'theme' = ?", 'dark')

# JSONB contains
Post.where("tags @> ?", '["ruby"]')

# Path operator
Post.where("settings #>> '{notifications, email}' = ?", 'true')

# Update JSONB
Post.where(id: 1).update_all("settings = settings || '#{{"featured": true}}'.to_json")

# Aggregate JSON
Post.group("settings->>'category'").count
```

```ruby
# Migration สำหรับ JSONB
class AddSettingsToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :settings, :jsonb, default: {
      theme: 'light',
      language: 'th',
      notifications: {
        email: true,
        push: false,
        sms: false
      }
    }
    
    # Index สำหรับ JSONB
    add_index :users, :settings, using: :gin
    
    # Index สำหรับ specific path
    add_index :users, "(settings->>'theme')", name: 'index_users_on_settings_theme'
  end
end
```

## ขั้นตอนที่ 1052: Serialization สำหรับ CSV/Excel

```ruby
# app/serializers/post_csv_serializer.rb
class PostCsvSerializer
  def self.serialize(posts)
    CSV.generate(headers: true) do |csv|
      csv << headers
      posts.each { |post| csv << row(post) }
    end
  end
  
  private
  
  def self.headers
    ["ID", "หัวข้อ", "ผู้เขียน", "สถานะ", "วันที่สร้าง", "ยอดวิว", "ยอดถูกใจ"]
  end
  
  def self.row(post)
    [
      post.id,
      post.title,
      post.user.name,
      post.published? ? "เผยแพร่" : "ร่าง",
      post.created_at.strftime("%d/%m/%Y %H:%M"),
      post.views_count,
      post.likes_count
    ]
  end
end

# ใน controller
class Admin::PostsController < ApplicationController
  def export
    @posts = Post.includes(:user).order(created_at: :desc)
    
    respond_to do |format|
      format.csv do
        send_data PostCsvSerializer.serialize(@posts),
          filename: "posts-#{Date.current}.csv",
          type: "text/csv"
      end
      format.xlsx do
        render xlsx: 'export', filename: "posts-#{Date.current}.xlsx"
      end
    end
  end
end
```

## ขั้นตอนที่ 1053: Real-time Data Serialization

```ruby
# ActionCable serialization
class PostsChannel < ApplicationCable::Channel
  def subscribed
    stream_from "posts_channel"
  end
  
  def receive(data)
    post = Post.find(data['id'])
    broadcast_post(post)
  end
  
  private
  
  def broadcast_post(post)
    ActionCable.server.broadcast "posts_channel",
      PostSerializer.new(post).serializable_hash
  end
end
```

## ขั้นตอนที่ 1054: Serialization Middleware

```ruby
# app/middleware/json_response_formatter.rb
class JsonResponseFormatter
  def initialize(app)
    @app = app
  end
  
  def call(env)
    status, headers, body = @app.call(env)
    
    if headers['Content-Type']&.include?('application/json')
      json_body = body.map { |chunk| chunk }.join
      
      begin
        parsed = JSON.parse(json_body)
        
        # เพิ่ม request metadata
        parsed['_meta'] = {
          timestamp: Time.current.iso8601,
          request_id: env['action_dispatch.request_id'],
          version: 'v1'
        }
        
        body = [parsed.to_json]
        headers['Content-Length'] = body.first.bytesize.to_s
      rescue JSON::ParserError
        # ถ้า parse ไม่ได้ ปล่อยผ่านไปเลย
      end
    end
    
    [status, headers, body]
  end
end
```

## ขั้นตอนที่ 1055: Performance Benchmarks

```ruby
# Benchmark serializers
require 'benchmark'

posts = Post.includes(:user, :tags).limit(100)

Benchmark.bm(25) do |x|
  x.report("to_json:") { posts.to_json }
  x.report("jbuilder:") { ApplicationController.render('api/v1/posts/index.json.jbuilder', assigns: { posts: posts }) }
  x.report("ams:") { ActiveModel::SerializableResource.new(posts).to_json }
  x.report("jsonapi-serializer:") { PostSerializer.new(posts).serializable_hash.to_json }
  x.report("blueprinter:") { PostBlueprint.render(posts) }
end
```

---

## แบบฝึกหัด: JSON Serialization (20 ข้อ)

### ข้อที่ 1: Override as_json
```
Override as_json ใน Post model ให้ส่งคืนเฉพาะ
id, title, excerpt (200 chars), author_name, published_at
```

### ข้อที่ 2: Jbuilder View
```
สร้าง show.json.jbuilder ที่รวม:
- ข้อมูล post
- ผู้เขียน
- tags
- 5 comments ล่าสุด
```

### ข้อที่ 3: Blueprinter
```
สร้าง ProductBlueprint ที่มี views:
- :summary (id, name, price)
- :detail (ทุก field + reviews)
```

### ข้อที่ 4: jsonapi-serializer
```
สร้าง OrderSerializer ที่ serialize:
- order attributes
- customer (belongs_to user)
- items (has_many order_items)
```

### ข้อที่ 5: Conditional Attributes
```
สร้าง serializer ที่แสดง sensitive data เฉพาะ admin
```

**เฉลย:**
```ruby
class UserSerializer
  include JSONAPI::Serializer
  attributes :name, :email
  
  attribute :phone do |user, params|
    user.phone if params[:current_user]&.admin? || params[:current_user] == user
  end
  
  attribute :internal_notes do |user, params|
    user.internal_notes if params[:current_user]&.admin?
  end
end
```

### ข้อที่ 6-20 (แบบสรุป)

**ข้อ 6:** Test serializer ด้วย RSpec
**ข้อ 7:** Performance optimization ด้วย eager loading
**ข้อ 8:** JSON columns ใน PostgreSQL
**ข้อ 9:** CSV serialization
**ข้อ 10:** Nested serializer หลายระดับ
**ข้อ 11:** Cache serialized responses
**ข้อ 12:** Serializer สำหรับ error responses
**ข้อ 13:** to_xml ด้วย custom builder
**ข้อ 14:** Streaming large JSON responses
**ข้อ 15:** JSON schema validation
**ข้อ 16:** ActionCable serialization
**ข้อ 17:** Benchmark serializers
**ข้อ 18:** Polymorphic association serialization
**ข้อ 19:** Dynamic fields ใน serializer
**ข้อ 20:** Internationalization ใน serializer

---

## สรุป: JSON Serialization

| Tool | ข้อดี | ข้อเสีย |
|------|-------|--------|
| as_json/to_json | Built-in, ง่าย | ไม่ structured |
| Jbuilder | Template-based, flexible | ช้า, ยากทดสอบ |
| Blueprinter | เร็ว, views | น้อย features |
| jsonapi-serializer | JSONAPI spec, relationships | ซับซ้อนกว่า |
| AMS | Mature, easy | ช้ากว่า |

**Key Takeaways:**
1. ใช้ Jbuilder สำหรับ views ที่ซับซ้อน
2. jsonapi-serializer เหมาะสำหรับ JSON:API spec
3. Blueprinter เร็วและง่าย
4. Eager loading สำคัญมากสำหรับ performance
5. Cache serialized responses เมื่อทำได้
