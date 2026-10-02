# Part 46: API Development ใน Rails

## ขั้นตอนที่ 1011-1035: การสร้าง RESTful API

---

## ขั้นตอนที่ 1011: Rails API Mode

Rails มี API mode พิเศษที่ตัด middleware ที่ไม่จำเป็นออก

```bash
# สร้าง API-only application
rails new myapi --api

# หรือเพิ่ม flag อื่นๆ
rails new myapi --api \
  --database=postgresql \
  --skip-test \
  --skip-asset-pipeline
```

```ruby
# config/application.rb
module Myapi
  class Application < Rails::Application
    # API mode
    config.api_only = true
    
    # Middleware configuration
    config.middleware.use ActionDispatch::Cookies
    config.middleware.use ActionDispatch::Session::CookieStore
    config.middleware.use ActionDispatch::Flash
    
    # Default format
    config.api_only = true
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include ActionController::HttpAuthentication::Token::ControllerMethods
  
  before_action :authenticate_request!
  
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
  rescue_from ActionController::ParameterMissing, with: :bad_request
  
  private
  
  def authenticate_request!
    # JWT authentication หรือ API key
  end
  
  def not_found(exception)
    render json: {
      error: "Not Found",
      message: "ไม่พบทรัพยากรที่ต้องการ",
      resource: exception.message
    }, status: :not_found
  end
  
  def unprocessable_entity(exception)
    render json: {
      error: "Unprocessable Entity",
      message: "ข้อมูลไม่ถูกต้อง",
      details: exception.record.errors.as_json
    }, status: :unprocessable_entity
  end
  
  def bad_request(exception)
    render json: {
      error: "Bad Request",
      message: exception.message
    }, status: :bad_request
  end
end
```

## ขั้นตอนที่ 1012: RESTful API Design

```
HTTP Methods และความหมาย:
GET     /posts          - รายการทั้งหมด (index)
GET     /posts/:id      - รายการเดียว (show)
POST    /posts          - สร้างใหม่ (create)
PUT     /posts/:id      - อัพเดททั้งหมด (update)
PATCH   /posts/:id      - อัพเดทบางส่วน (partial update)
DELETE  /posts/:id      - ลบ (destroy)
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :posts do
        collection do
          get :search
          get :featured
        end
        
        member do
          post :publish
          post :unpublish
          post :like
        end
        
        resources :comments, only: [:index, :create, :destroy]
      end
      
      resources :users, only: [:index, :show, :create, :update] do
        get :profile, on: :member
        get :posts, on: :member
      end
      
      resources :categories, only: [:index, :show]
      resources :tags, only: [:index]
      
      # Authentication
      post 'auth/login', to: 'auth#login'
      post 'auth/refresh', to: 'auth#refresh'
      delete 'auth/logout', to: 'auth#logout'
    end
  end
  
  # Health check
  get '/health', to: proc { [200, {}, [{ status: 'ok' }.to_json]] }
end
```

## ขั้นตอนที่ 1013: API Versioning

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # URL versioning (most common)
  namespace :api do
    namespace :v1 do
      resources :posts
    end
    
    namespace :v2 do
      resources :posts
    end
  end
  
  # Header versioning
  # Accept: application/vnd.myapp.v1+json
  
  # Subdomain versioning
  # v1.api.myapp.com
end
```

```ruby
# app/controllers/api/v1/posts_controller.rb
module Api
  module V1
    class PostsController < Api::BaseController
      def index
        @posts = Post.published
                     .includes(:user, :tags)
                     .order(created_at: :desc)
        
        render json: PostSerializer.new(@posts).serializable_hash
      end
      
      def show
        @post = Post.find(params[:id])
        render json: PostSerializer.new(@post).serializable_hash
      end
      
      def create
        @post = current_user.posts.build(post_params)
        
        if @post.save
          render json: PostSerializer.new(@post).serializable_hash,
                 status: :created,
                 location: api_v1_post_url(@post)
        else
          render json: { errors: @post.errors }, 
                 status: :unprocessable_entity
        end
      end
      
      def update
        @post = Post.find(params[:id])
        authorize @post
        
        if @post.update(post_params)
          render json: PostSerializer.new(@post).serializable_hash
        else
          render json: { errors: @post.errors },
                 status: :unprocessable_entity
        end
      end
      
      def destroy
        @post = Post.find(params[:id])
        authorize @post
        @post.destroy
        head :no_content
      end
      
      private
      
      def post_params
        params.require(:post).permit(:title, :content, :category_id, tag_ids: [])
      end
    end
  end
end
```

```ruby
# app/controllers/api/v2/posts_controller.rb (อัพเดท version)
module Api
  module V2
    class PostsController < Api::V1::PostsController
      # Override หรือเพิ่ม methods ที่เปลี่ยนใน V2
      def index
        @posts = super_query
                   .with_reading_time  # feature ใหม่ใน V2
        
        render json: V2::PostSerializer.new(@posts).serializable_hash
      end
      
      private
      
      def super_query
        Post.published.includes(:user, :tags, :reading_times)
      end
    end
  end
end
```

## ขั้นตอนที่ 1014: Serializers ด้วย Active Model Serializers

```ruby
# Gemfile
gem 'active_model_serializers'
```

```ruby
# app/serializers/post_serializer.rb
class PostSerializer < ActiveModel::Serializer
  attributes :id, :title, :content, :excerpt, :published, 
             :created_at, :updated_at, :reading_time
  
  belongs_to :user
  has_many :tags
  has_many :comments
  
  # Custom attribute
  attribute :url do
    Rails.application.routes.url_helpers.api_v1_post_url(object)
  end
  
  attribute :likes_count do
    object.likes.count
  end
  
  # Conditional attribute
  attribute :admin_notes, if: :admin_viewing? do
    object.admin_notes
  end
  
  def admin_viewing?
    scope&.admin?
  end
  
  def excerpt
    object.content&.truncate(150)
  end
end
```

```ruby
# app/serializers/user_serializer.rb
class UserSerializer < ActiveModel::Serializer
  attributes :id, :name, :email, :avatar_url, :created_at
  
  has_many :posts, serializer: PostSummarySerializer
  
  attribute :avatar_url do
    if object.avatar.attached?
      Rails.application.routes.url_helpers.url_for(object.avatar)
    else
      "https://ui-avatars.com/api/?name=#{URI.encode_www_form_component(object.name)}"
    end
  end
  
  # ซ่อน email ถ้าไม่ใช่ตัวเอง
  attribute :email, if: -> { scope == object || scope&.admin? }
end
```

## ขั้นตอนที่ 1015: jsonapi-serializer (Fast JSONAPI)

```ruby
# Gemfile
gem 'jsonapi-serializer'
```

```ruby
# app/serializers/post_serializer.rb
class PostSerializer
  include JSONAPI::Serializer
  
  # Attributes
  attributes :title, :content, :published, :created_at
  
  # Computed attributes
  attribute :excerpt do |object|
    object.content&.truncate(200)
  end
  
  attribute :reading_time do |object|
    (object.content.to_s.split.length / 200.0).ceil
  end
  
  attribute :formatted_date do |object|
    object.created_at.strftime("%d %B %Y")
  end
  
  # Relationships
  belongs_to :user
  has_many :tags
  has_many :comments
  
  # Links
  link :self do |object|
    Rails.application.routes.url_helpers.api_v1_post_url(object)
  end
  
  # Meta
  meta do |object|
    { likes: object.likes_count, views: object.views_count }
  end
end
```

```ruby
# การใช้งานใน controller
class Api::V1::PostsController < ApplicationController
  def index
    @posts = Post.published.includes(:user, :tags)
    
    render json: PostSerializer.new(
      @posts,
      include: [:user, :tags],
      params: { current_user: current_user }
    ).serializable_hash
  end
  
  def show
    @post = Post.includes(:user, :tags, :comments).find(params[:id])
    
    render json: PostSerializer.new(
      @post,
      include: ['user', 'tags', 'comments.user'],
      meta: { 
        generated_at: Time.current,
        api_version: 'v1'
      }
    ).serializable_hash
  end
end
```

## ขั้นตอนที่ 1016: Pagination ด้วย Kaminari

```ruby
# Gemfile
gem 'kaminari'
gem 'api-pagination'  # integration กับ Link header
```

```ruby
# config/initializers/kaminari_config.rb
Kaminari.configure do |config|
  config.default_per_page = 20
  config.max_per_page = 100
  config.window = 4
  config.outer_window = 0
  config.left = 0
  config.right = 0
  config.page_method_name = :page
  config.param_name = :page
end
```

```ruby
class Api::V1::PostsController < ApplicationController
  def index
    @posts = Post.published
                 .includes(:user, :tags)
                 .order(created_at: :desc)
                 .page(params[:page])
                 .per(params[:per_page] || 20)
    
    render json: {
      data: PostSerializer.new(@posts).serializable_hash[:data],
      meta: {
        current_page: @posts.current_page,
        total_pages: @posts.total_pages,
        total_count: @posts.total_count,
        per_page: @posts.limit_value,
        next_page: @posts.next_page,
        prev_page: @posts.prev_page
      },
      links: pagination_links(@posts)
    }
  end
  
  private
  
  def pagination_links(collection)
    base_url = request.base_url + request.path
    
    {
      self: "#{base_url}?page=#{collection.current_page}",
      first: "#{base_url}?page=1",
      last: "#{base_url}?page=#{collection.total_pages}",
      next: collection.next_page ? "#{base_url}?page=#{collection.next_page}" : nil,
      prev: collection.prev_page ? "#{base_url}?page=#{collection.prev_page}" : nil
    }.compact
  end
end
```

## ขั้นตอนที่ 1017: Pagination ด้วย Pagy

```ruby
# Gemfile
gem 'pagy'
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include Pagy::Backend
  
  private
  
  def pagy_get_vars(collection, vars)
    vars[:count] ||= collection.count
    vars
  end
end
```

```ruby
# config/initializers/pagy.rb
Pagy::DEFAULT[:items] = 20
Pagy::DEFAULT[:max_pages] = 100
```

```ruby
class Api::V1::PostsController < ApplicationController
  def index
    @pagy, @posts = pagy(
      Post.published.order(created_at: :desc),
      items: params[:per_page] || 20,
      page: params[:page] || 1
    )
    
    render json: {
      data: serialize_collection(@posts),
      pagination: pagy_metadata(@pagy)
    }
  end
  
  private
  
  def pagy_metadata(pagy)
    {
      page: pagy.page,
      items: pagy.items,
      count: pagy.count,
      pages: pagy.pages,
      next: pagy.next,
      prev: pagy.prev
    }
  end
end
```

## ขั้นตอนที่ 1018: Filtering และ Sorting

```ruby
# app/controllers/api/v1/posts_controller.rb
class Api::V1::PostsController < ApplicationController
  def index
    @posts = Post.all
    @posts = filter_posts(@posts)
    @posts = sort_posts(@posts)
    @posts = @posts.page(params[:page]).per(params[:per_page] || 20)
    
    render json: PostSerializer.new(@posts).serializable_hash
  end
  
  private
  
  def filter_posts(posts)
    posts = posts.where(published: true) unless admin_viewing?
    posts = posts.where(user_id: params[:user_id]) if params[:user_id]
    posts = posts.where(category_id: params[:category_id]) if params[:category_id]
    posts = posts.joins(:tags).where(tags: { id: params[:tag_ids] }) if params[:tag_ids]
    posts = posts.where("title LIKE ?", "%#{params[:search]}%") if params[:search]
    posts = posts.where("created_at >= ?", params[:from].to_date) if params[:from]
    posts = posts.where("created_at <= ?", params[:to].to_date) if params[:to]
    posts
  end
  
  def sort_posts(posts)
    allowed_sorts = %w[created_at updated_at title views_count likes_count]
    sort_field = params[:sort] if allowed_sorts.include?(params[:sort])
    sort_field ||= 'created_at'
    
    direction = %w[asc desc].include?(params[:direction]) ? params[:direction] : 'desc'
    
    posts.order("#{sort_field} #{direction}")
  end
  
  def admin_viewing?
    current_user&.admin?
  end
end
```

```ruby
# app/models/concerns/filterable.rb
module Filterable
  extend ActiveSupport::Concern
  
  included do
    scope :filter_by, ->(filters) {
      results = all
      filters.each do |key, value|
        results = results.public_send("filter_by_#{key}", value) if value.present?
      end
      results
    }
  end
end

# app/models/post.rb
class Post < ApplicationRecord
  include Filterable
  
  scope :filter_by_category, ->(category_id) { where(category_id: category_id) }
  scope :filter_by_user, ->(user_id) { where(user_id: user_id) }
  scope :filter_by_published, ->(published) { where(published: published) }
  scope :filter_by_tag, ->(tag_id) { joins(:tags).where(tags: { id: tag_id }) }
  scope :filter_by_search, ->(query) {
    where("title ILIKE :q OR content ILIKE :q", q: "%#{query}%")
  }
end
```

## ขั้นตอนที่ 1019: Rate Limiting

```ruby
# Gemfile
gem 'rack-attack'
```

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Cache store (ใช้ Redis)
  Rack::Attack.cache.store = ActiveSupport::Cache::RedisCacheStore.new(
    url: ENV['REDIS_URL']
  )
  
  # Throttle requests to login (prevent brute force)
  throttle('logins/ip', limit: 5, period: 20.seconds) do |req|
    if req.path == '/api/v1/auth/login' && req.post?
      req.ip
    end
  end
  
  # Throttle requests to API
  throttle('api/ip', limit: 100, period: 1.minute) do |req|
    req.ip if req.path.start_with?('/api/')
  end
  
  # Throttle per user (ถ้ามี API token)
  throttle('api/user', limit: 1000, period: 1.hour) do |req|
    if req.path.start_with?('/api/')
      token = req.env['HTTP_AUTHORIZATION']&.split(' ')&.last
      token if token.present?
    end
  end
  
  # Block bad actors
  blocklist('block/bad-actor') do |req|
    BlockedIp.exists?(ip: req.ip)
  end
  
  # Allow admin to bypass
  safelist('allow/admin') do |req|
    AdminIp.exists?(ip: req.ip)
  end
  
  # Custom response
  self.throttled_responder = lambda do |request|
    match_data = request.env['rack.attack.match_data']
    
    now = match_data[:epoch_time]
    headers = {
      'RateLimit-Limit' => match_data[:limit].to_s,
      'RateLimit-Remaining' => '0',
      'RateLimit-Reset' => (now + (match_data[:period] - now % match_data[:period])).to_s
    }
    
    [429, headers, [{ error: 'Rate limit exceeded', message: 'ส่ง request มากเกินไป' }.to_json]]
  end
end
```

## ขั้นตอนที่ 1020: CORS Configuration

```ruby
# Gemfile
gem 'rack-cors'
```

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    # Development
    origins 'http://localhost:3000', 'http://localhost:3001'
    
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      expose: ['Authorization', 'X-Request-Id'],
      max_age: 600
  end
  
  allow do
    # Production
    origins 'https://myapp.com', 'https://admin.myapp.com',
            /https:\/\/.*\.myapp\.com\z/
    
    resource '/api/*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options],
      credentials: true,
      expose: ['Authorization']
  end
  
  allow do
    # Public API (no CORS restriction)
    origins '*'
    
    resource '/api/v1/public/*',
      headers: :any,
      methods: [:get]
  end
end
```

## ขั้นตอนที่ 1021: JWT Authentication สำหรับ API

```ruby
# Gemfile
gem 'jwt'
gem 'bcrypt'
```

```ruby
# app/services/jwt_service.rb
class JwtService
  SECRET = Rails.application.credentials.jwt_secret || ENV['JWT_SECRET']
  ALGORITHM = 'HS256'
  EXPIRY = 24.hours.from_now
  
  def self.encode(payload, expiry: EXPIRY)
    payload = payload.merge(
      exp: expiry.to_i,
      iat: Time.current.to_i,
      jti: SecureRandom.uuid  # JWT ID สำหรับ revocation
    )
    JWT.encode(payload, SECRET, ALGORITHM)
  end
  
  def self.decode(token)
    decoded = JWT.decode(token, SECRET, true, { algorithm: ALGORITHM })
    HashWithIndifferentAccess.new(decoded.first)
  rescue JWT::ExpiredSignature
    raise AuthenticationError, "Token หมดอายุ"
  rescue JWT::DecodeError => e
    raise AuthenticationError, "Token ไม่ถูกต้อง: #{e.message}"
  end
  
  def self.refresh(token)
    decoded = decode(token)
    payload = decoded.except('exp', 'iat', 'jti')
    encode(payload)
  end
end
```

```ruby
# app/controllers/api/v1/auth_controller.rb
class Api::V1::AuthController < ApplicationController
  skip_before_action :authenticate_request!, only: [:login, :register]
  
  def login
    user = User.find_by(email: params[:email]&.downcase)
    
    if user&.authenticate(params[:password])
      if user.active?
        tokens = generate_tokens(user)
        
        render json: {
          user: UserSerializer.new(user).serializable_hash,
          tokens: tokens
        }, status: :ok
      else
        render json: { error: "บัญชีถูก deactivate" }, status: :forbidden
      end
    else
      render json: { 
        error: "Authentication failed",
        message: "อีเมลหรือรหัสผ่านไม่ถูกต้อง" 
      }, status: :unauthorized
    end
  end
  
  def register
    user = User.new(registration_params)
    
    if user.save
      tokens = generate_tokens(user)
      
      render json: {
        message: "ลงทะเบียนสำเร็จ",
        user: UserSerializer.new(user).serializable_hash,
        tokens: tokens
      }, status: :created
    else
      render json: {
        error: "Registration failed",
        details: user.errors.as_json
      }, status: :unprocessable_entity
    end
  end
  
  def refresh
    new_token = JwtService.refresh(refresh_token_from_header)
    render json: { access_token: new_token }
  rescue AuthenticationError => e
    render json: { error: e.message }, status: :unauthorized
  end
  
  def logout
    # Revoke token (add to blacklist)
    decoded = JwtService.decode(bearer_token)
    RevokedToken.create!(jti: decoded[:jti], expires_at: Time.at(decoded[:exp]))
    
    render json: { message: "ออกจากระบบแล้ว" }
  rescue AuthenticationError
    render json: { message: "ออกจากระบบแล้ว" }
  end
  
  private
  
  def generate_tokens(user)
    access_token = JwtService.encode(
      { user_id: user.id, role: user.role },
      expiry: 1.hour.from_now
    )
    
    refresh_token = JwtService.encode(
      { user_id: user.id, token_type: 'refresh' },
      expiry: 30.days.from_now
    )
    
    # บันทึก refresh token
    user.update!(refresh_token: BCrypt::Password.create(refresh_token))
    
    {
      access_token: access_token,
      refresh_token: refresh_token,
      token_type: 'Bearer',
      expires_in: 3600
    }
  end
  
  def registration_params
    params.permit(:name, :email, :password, :password_confirmation)
  end
  
  def refresh_token_from_header
    request.headers['X-Refresh-Token'] || params[:refresh_token]
  end
  
  def bearer_token
    request.headers['Authorization']&.split(' ')&.last
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  before_action :authenticate_request!
  
  attr_reader :current_user
  
  private
  
  def authenticate_request!
    token = bearer_token
    
    if token.blank?
      return render json: { 
        error: "Unauthorized",
        message: "กรุณาระบุ Authorization header" 
      }, status: :unauthorized
    end
    
    decoded = JwtService.decode(token)
    
    # ตรวจสอบ token revocation
    if RevokedToken.exists?(jti: decoded[:jti])
      return render json: { 
        error: "Unauthorized", 
        message: "Token ถูก revoke แล้ว" 
      }, status: :unauthorized
    end
    
    @current_user = User.find_by(id: decoded[:user_id])
    
    unless @current_user&.active?
      render json: { error: "บัญชีถูก deactivate" }, status: :forbidden
    end
  rescue AuthenticationError => e
    render json: { error: e.message }, status: :unauthorized
  end
  
  def bearer_token
    request.headers['Authorization']&.split(' ')&.last
  end
end
```

## ขั้นตอนที่ 1022: API Key Authentication

```ruby
# Migration
class CreateApiKeys < ActiveRecord::Migration[7.0]
  def change
    create_table :api_keys do |t|
      t.references :user, null: false, foreign_key: true
      t.string :key, null: false
      t.string :name
      t.text :description
      t.datetime :expires_at
      t.datetime :last_used_at
      t.string :last_used_ip
      t.boolean :active, default: true
      t.json :permissions, default: {}
      
      t.timestamps
    end
    
    add_index :api_keys, :key, unique: true
  end
end
```

```ruby
# app/models/api_key.rb
class ApiKey < ApplicationRecord
  belongs_to :user
  
  before_create :generate_key
  
  scope :active, -> { where(active: true).where("expires_at IS NULL OR expires_at > ?", Time.current) }
  
  def expired?
    expires_at.present? && expires_at < Time.current
  end
  
  def touch_last_used!(ip)
    update_columns(last_used_at: Time.current, last_used_ip: ip)
  end
  
  private
  
  def generate_key
    self.key = "mk_#{SecureRandom.urlsafe_base64(32)}"
  end
end
```

```ruby
# ใช้ API key ใน controller
class ApplicationController < ActionController::API
  private
  
  def authenticate_with_api_key!
    api_key = ApiKey.active.find_by(key: api_key_from_header)
    
    if api_key
      api_key.touch_last_used!(request.remote_ip)
      @current_user = api_key.user
    else
      render json: { error: "Invalid API key" }, status: :unauthorized
    end
  end
  
  def api_key_from_header
    # X-API-Key: mk_xxx
    request.headers['X-API-Key'] ||
    # Authorization: ApiKey mk_xxx
    request.headers['Authorization']&.gsub(/^ApiKey /, '')
  end
end
```

## ขั้นตอนที่ 1023: API Response Format

```ruby
# app/controllers/concerns/api_response.rb
module ApiResponse
  extend ActiveSupport::Concern
  
  def success(data: nil, message: nil, status: :ok, meta: {})
    response = { success: true }
    response[:message] = message if message
    response[:data] = data if data
    response[:meta] = meta unless meta.empty?
    
    render json: response, status: status
  end
  
  def error(message:, details: nil, status: :unprocessable_entity, code: nil)
    response = { success: false, error: message }
    response[:details] = details if details
    response[:code] = code if code
    
    render json: response, status: status
  end
  
  def paginated(collection, serializer_class: nil, meta: {})
    data = if serializer_class
      serializer_class.new(collection).serializable_hash
    else
      collection
    end
    
    render json: {
      success: true,
      data: data,
      meta: {
        current_page: collection.current_page,
        total_pages: collection.total_pages,
        total_count: collection.total_count,
        per_page: collection.limit_value
      }.merge(meta)
    }
  end
end
```

```ruby
class Api::V1::PostsController < ApplicationController
  include ApiResponse
  
  def index
    @posts = Post.published.page(params[:page])
    paginated(@posts, serializer_class: PostSerializer)
  end
  
  def create
    @post = current_user.posts.build(post_params)
    
    if @post.save
      success(
        data: PostSerializer.new(@post).serializable_hash, 
        message: "สร้างบทความสำเร็จ",
        status: :created
      )
    else
      error(
        message: "ไม่สามารถสร้างบทความได้",
        details: @post.errors.as_json
      )
    end
  end
end
```

## ขั้นตอนที่ 1024: API Documentation ด้วย Swagger/OpenAPI

```ruby
# Gemfile
gem 'rswag'
gem 'rswag-api'
gem 'rswag-ui'
gem 'rswag-specs'
```

```bash
rails generate rswag:install
```

```ruby
# spec/swagger_helper.rb
RSpec.configure do |config|
  config.swagger_root = Rails.root.join('swagger').to_s
  
  config.swagger_docs = {
    'v1/swagger.yaml' => {
      openapi: '3.0.1',
      info: {
        title: 'My API',
        version: 'v1',
        description: 'API สำหรับ My Rails Application',
        contact: {
          name: 'API Support',
          email: 'api@myapp.com'
        }
      },
      paths: {},
      servers: [
        { url: 'http://localhost:3000', description: 'Development' },
        { url: 'https://api.myapp.com', description: 'Production' }
      ],
      components: {
        securitySchemes: {
          bearerAuth: {
            type: :http,
            scheme: :bearer,
            bearerFormat: :JWT
          }
        }
      }
    }
  }
end
```

```ruby
# spec/integration/posts_spec.rb
require 'swagger_helper'

RSpec.describe 'Posts API', type: :request do
  path '/api/v1/posts' do
    get 'รายการบทความทั้งหมด' do
      tags 'Posts'
      security [bearerAuth: []]
      produces 'application/json'
      
      parameter name: :page, in: :query, type: :integer, description: 'หน้าที่'
      parameter name: :per_page, in: :query, type: :integer, description: 'จำนวนต่อหน้า'
      parameter name: :search, in: :query, type: :string, description: 'ค้นหา'
      
      response '200', 'รายการบทความ' do
        schema type: :object,
          properties: {
            data: {
              type: :array,
              items: { '$ref': '#/components/schemas/Post' }
            },
            meta: {
              type: :object,
              properties: {
                current_page: { type: :integer },
                total_pages: { type: :integer },
                total_count: { type: :integer }
              }
            }
          }
        
        run_test!
      end
    end
    
    post 'สร้างบทความใหม่' do
      tags 'Posts'
      security [bearerAuth: []]
      consumes 'application/json'
      produces 'application/json'
      
      parameter name: :post, in: :body, schema: {
        type: :object,
        properties: {
          title: { type: :string, description: 'หัวข้อบทความ' },
          content: { type: :string, description: 'เนื้อหา' },
          published: { type: :boolean, description: 'เผยแพร่หรือไม่' }
        },
        required: [:title, :content]
      }
      
      response '201', 'สร้างสำเร็จ' do
        let(:post) { { title: "Test Post", content: "Content here" } }
        run_test!
      end
      
      response '422', 'ข้อมูลไม่ถูกต้อง' do
        let(:post) { { title: "" } }
        run_test!
      end
    end
  end
  
  path '/api/v1/posts/{id}' do
    parameter name: :id, in: :path, type: :integer, description: 'Post ID'
    
    get 'ดูบทความ' do
      tags 'Posts'
      produces 'application/json'
      
      response '200', 'บทความ' do
        let(:id) { create(:post, :published).id }
        run_test!
      end
      
      response '404', 'ไม่พบ' do
        let(:id) { 0 }
        run_test!
      end
    end
  end
end
```

## ขั้นตอนที่ 1025: Error Handling

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  rescue_from StandardError, with: :handle_standard_error
  rescue_from ActiveRecord::RecordNotFound, with: :handle_not_found
  rescue_from ActiveRecord::RecordInvalid, with: :handle_validation_error
  rescue_from ActionController::ParameterMissing, with: :handle_parameter_missing
  rescue_from Pundit::NotAuthorizedError, with: :handle_unauthorized
  rescue_from AuthenticationError, with: :handle_unauthenticated
  
  private
  
  def handle_not_found(exception)
    render json: api_error(
      status: 404,
      error: "Not Found",
      message: "ไม่พบทรัพยากรที่ต้องการ",
      details: exception.message
    ), status: :not_found
  end
  
  def handle_validation_error(exception)
    render json: api_error(
      status: 422,
      error: "Validation Error",
      message: "ข้อมูลไม่ถูกต้อง",
      details: exception.record.errors.as_json
    ), status: :unprocessable_entity
  end
  
  def handle_unauthorized
    render json: api_error(
      status: 403,
      error: "Forbidden",
      message: "ไม่มีสิทธิ์ดำเนินการนี้"
    ), status: :forbidden
  end
  
  def handle_unauthenticated(exception)
    render json: api_error(
      status: 401,
      error: "Unauthorized",
      message: exception.message
    ), status: :unauthorized
  end
  
  def handle_parameter_missing(exception)
    render json: api_error(
      status: 400,
      error: "Bad Request",
      message: "พารามิเตอร์ที่จำเป็นหายไป: #{exception.param}"
    ), status: :bad_request
  end
  
  def handle_standard_error(exception)
    Rails.logger.error(exception.message)
    Rails.logger.error(exception.backtrace.join("\n"))
    
    render json: api_error(
      status: 500,
      error: "Internal Server Error",
      message: Rails.env.development? ? exception.message : "เกิดข้อผิดพลาดภายใน"
    ), status: :internal_server_error
  end
  
  def api_error(status:, error:, message:, details: nil)
    {
      success: false,
      status: status,
      error: error,
      message: message,
      details: details,
      timestamp: Time.current.iso8601,
      request_id: request.request_id
    }.compact
  end
end
```

## ขั้นตอนที่ 1026: API Testing

```ruby
# spec/requests/api/v1/posts_spec.rb
require 'rails_helper'

RSpec.describe "Api::V1::Posts", type: :request do
  let(:user) { create(:user) }
  let(:token) { JwtService.encode(user_id: user.id) }
  let(:headers) { { 'Authorization' => "Bearer #{token}" } }
  
  describe "GET /api/v1/posts" do
    let!(:published_posts) { create_list(:post, 5, :published) }
    let!(:draft_posts) { create_list(:post, 3, :draft) }
    
    context "without authentication" do
      it "returns published posts" do
        get "/api/v1/posts"
        
        expect(response).to have_http_status(:ok)
        data = JSON.parse(response.body)
        expect(data['data'].length).to eq(5)
      end
    end
    
    context "with pagination" do
      before { create_list(:post, 25, :published) }
      
      it "returns paginated results" do
        get "/api/v1/posts", params: { page: 1, per_page: 10 }
        
        body = JSON.parse(response.body)
        expect(body['data'].length).to eq(10)
        expect(body['meta']['total_pages']).to be > 1
      end
    end
    
    context "with filtering" do
      let(:category) { create(:category) }
      let!(:category_posts) { create_list(:post, 3, :published, category: category) }
      
      it "filters by category" do
        get "/api/v1/posts", params: { category_id: category.id }
        
        body = JSON.parse(response.body)
        expect(body['data'].length).to eq(3)
      end
    end
  end
  
  describe "POST /api/v1/posts" do
    let(:valid_params) do
      { post: { title: "New Post", content: "Content here" } }
    end
    
    context "authenticated" do
      it "creates a post" do
        expect {
          post "/api/v1/posts", 
               params: valid_params,
               headers: headers,
               as: :json
        }.to change(Post, :count).by(1)
        
        expect(response).to have_http_status(:created)
      end
      
      it "returns post data" do
        post "/api/v1/posts",
             params: valid_params,
             headers: headers,
             as: :json
        
        body = JSON.parse(response.body)
        expect(body['data']['attributes']['title']).to eq("New Post")
      end
    end
    
    context "with invalid data" do
      it "returns error" do
        post "/api/v1/posts",
             params: { post: { title: "" } },
             headers: headers,
             as: :json
        
        expect(response).to have_http_status(:unprocessable_entity)
        body = JSON.parse(response.body)
        expect(body['errors']).to be_present
      end
    end
    
    context "unauthenticated" do
      it "returns 401" do
        post "/api/v1/posts", params: valid_params, as: :json
        expect(response).to have_http_status(:unauthorized)
      end
    end
  end
end
```

## ขั้นตอนที่ 1027: API Versioning Strategy

```ruby
# lib/api_constraints.rb
class ApiConstraints
  def initialize(options)
    @version = options[:version]
    @default = options[:default]
  end
  
  def matches?(req)
    @default || req.headers['Accept'].include?("application/vnd.myapp.v#{@version}+json")
  end
end
```

```ruby
# config/routes.rb - Header-based versioning
Rails.application.routes.draw do
  scope :api do
    scope module: :v2, constraints: ApiConstraints.new(version: 2) do
      resources :posts
    end
    
    scope module: :v1, constraints: ApiConstraints.new(version: 1, default: true) do
      resources :posts
    end
  end
end
```

## ขั้นตอนที่ 1028: Content Negotiation

```ruby
class Api::V1::PostsController < ApplicationController
  def index
    @posts = Post.published
    
    respond_to do |format|
      format.json { render json: PostSerializer.new(@posts).serializable_hash }
      format.xml  { render xml: @posts.to_xml }
      format.csv  { send_data posts_to_csv, filename: "posts-#{Date.current}.csv" }
    end
  end
  
  private
  
  def posts_to_csv
    CSV.generate(headers: true) do |csv|
      csv << ["ID", "Title", "Author", "Published At"]
      @posts.each do |post|
        csv << [post.id, post.title, post.user.name, post.published_at]
      end
    end
  end
end
```

## ขั้นตอนที่ 1029: API Caching

```ruby
# app/controllers/api/v1/posts_controller.rb
class Api::V1::PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    
    # HTTP caching
    if stale?(last_modified: @post.updated_at, etag: @post)
      render json: PostSerializer.new(@post).serializable_hash
    end
  end
  
  def index
    @posts = Post.published.order(updated_at: :desc)
    
    # Cache response
    cache_key = "api/v1/posts/#{params.slice(:page, :per_page, :category_id).to_h.sort.to_s}"
    
    if stale?(etag: cache_key, last_modified: @posts.maximum(:updated_at))
      render json: PostSerializer.new(@posts).serializable_hash
    end
  end
end
```

## ขั้นตอนที่ 1030: Webhooks

```ruby
# app/models/webhook.rb
class Webhook < ApplicationRecord
  belongs_to :user
  
  validates :url, presence: true, format: { with: URI::DEFAULT_PARSER.make_regexp(%w[http https]) }
  validates :events, presence: true
  
  serialize :events, coder: JSON
  
  EVENTS = %w[post.created post.published post.deleted user.registered].freeze
  
  def deliver!(event, payload)
    WebhookDeliveryJob.perform_later(id, event, payload)
  end
end

# app/jobs/webhook_delivery_job.rb
class WebhookDeliveryJob < ApplicationJob
  queue_as :webhooks
  retry_on StandardError, wait: :exponentially_longer, attempts: 5
  
  def perform(webhook_id, event, payload)
    webhook = Webhook.find(webhook_id)
    
    return unless webhook.events.include?(event)
    
    deliver(webhook, event, payload)
  end
  
  private
  
  def deliver(webhook, event, payload)
    timestamp = Time.current.to_i
    signature = generate_signature(webhook.secret, payload, timestamp)
    
    response = HTTP.post(webhook.url, json: {
      event: event,
      data: payload,
      timestamp: timestamp
    }, headers: {
      'X-Webhook-Signature': signature,
      'X-Webhook-Event': event,
      'X-Webhook-Timestamp': timestamp.to_s
    })
    
    WebhookDelivery.create!(
      webhook: webhook,
      event: event,
      response_code: response.code,
      response_body: response.body.to_s.truncate(1000),
      delivered_at: Time.current
    )
  end
  
  def generate_signature(secret, payload, timestamp)
    body = "#{timestamp}.#{payload.to_json}"
    "sha256=#{OpenSSL::HMAC.hexdigest('SHA256', secret, body)}"
  end
end
```

## ขั้นตอนที่ 1031-1035: Advanced API Features

```ruby
# Batch API requests
class Api::V1::BatchController < ApplicationController
  def create
    results = params[:requests].map do |req|
      process_request(req)
    end
    
    render json: { results: results }
  end
  
  private
  
  def process_request(request_params)
    method = request_params[:method].upcase
    path = request_params[:path]
    body = request_params[:body]
    
    # Simulate request
    env = Rack::MockRequest.env_for(path, method: method, input: body.to_json)
    status, headers, response = Rails.application.call(env)
    
    {
      status: status,
      body: JSON.parse(response.first),
      headers: headers
    }
  rescue => e
    { status: 500, error: e.message }
  end
end
```

---

## แบบฝึกหัด: API Development (25 ข้อ)

### ข้อที่ 1: สร้าง API-only Application
```bash
rails new myapi --api --database=postgresql
```

### ข้อที่ 2: สร้าง RESTful Resources
```
สร้าง products API ที่มี CRUD ครบถ้วน
```

**เฉลย:**
```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :products
  end
end
```

### ข้อที่ 3: JWT Authentication
สร้าง auth endpoints สำหรับ login, register, refresh, logout

### ข้อที่ 4: Serializer
สร้าง ProductSerializer ด้วย jsonapi-serializer

### ข้อที่ 5: Pagination
เพิ่ม pagination ด้วย Pagy

### ข้อที่ 6: Rate Limiting
ตั้งค่า rack-attack สำหรับ 100 requests/minute

### ข้อที่ 7: CORS
กำหนด CORS ให้รับ request จาก localhost:3001

### ข้อที่ 8: Filtering และ Sorting
เพิ่ม filter และ sort สำหรับ products list

### ข้อที่ 9: API Documentation
สร้าง Swagger documentation ด้วย rswag

### ข้อที่ 10: Error Handling
สร้าง centralized error handling ที่ส่ง JSON response

### ข้อที่ 11-25 (แบบสรุป)

**ข้อ 11:** API versioning (v1, v2)
**ข้อ 12:** API key authentication
**ข้อ 13:** Webhook system
**ข้อ 14:** Request/Response logging
**ข้อ 15:** Testing API endpoints
**ข้อ 16:** API health check endpoint
**ข้อ 17:** Nested resource API
**ข้อ 18:** File upload API
**ข้อ 19:** Search API
**ข้อ 20:** Caching API responses
**ข้อ 21:** Batch API requests
**ข้อ 22:** GraphQL integration
**ข้อ 23:** API analytics
**ข้อ 24:** Deprecation headers
**ข้อ 25:** API SDK generation

```ruby
# ข้อ 24: Deprecation headers
module Api
  module Deprecatable
    def deprecate_api(version:, sunset_date:, alternative: nil)
      response.headers['Deprecation'] = "true"
      response.headers['Sunset'] = sunset_date.httpdate
      response.headers['Link'] = "<#{alternative}>; rel=\"successor-version\"" if alternative
    end
  end
end

class Api::V1::OldFeaturesController < ApplicationController
  include Api::Deprecatable
  
  before_action { deprecate_api(version: "v1", sunset_date: 6.months.from_now) }
  
  def index
    render json: { warning: "This endpoint is deprecated" }
  end
end
```

---

## สรุป: API Development

| Feature | Tool |
|---------|------|
| Serialization | jsonapi-serializer, AMS |
| Authentication | JWT, Devise Token Auth |
| Pagination | Pagy, Kaminari |
| Rate Limiting | rack-attack |
| CORS | rack-cors |
| Documentation | rswag (Swagger) |
| Testing | RSpec request specs |

**Key Takeaways:**
1. ใช้ API mode (--api) สำหรับ pure API applications
2. Version API ตั้งแต่เริ่มต้น
3. Authenticate ทุก endpoint ที่ sensitive
4. Rate limit เพื่อป้องกัน abuse
5. Document API ด้วย Swagger
6. Handle errors อย่างสม่ำเสมอ
