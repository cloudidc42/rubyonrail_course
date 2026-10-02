# ตอนที่ 46: API Development (Steps 1011-1035)

## บทนำ

API (Application Programming Interface) คือชุดของ endpoints ที่ให้บริการข้อมูลและการกระทำต่างๆ ผ่าน HTTP protocol โดย Rails มีความสามารถในการสร้าง API ได้อย่างยอดเยี่ยมทั้งในโหมด Full Stack และ API-only mode

ในบทนี้เราจะเรียนรู้การสร้าง RESTful API ด้วย Rails 7 ตั้งแต่พื้นฐานจนถึงการ authentication, versioning, pagination และการทดสอบ

---

## Step 1011: Rails API Mode

### การสร้างโปรเจกต์ Rails API

```bash
# สร้าง Rails API application
rails new myapi --api --database=postgresql

# เข้าไปในโฟลเดอร์
cd myapi
```

### ความแตกต่างระหว่าง API Mode กับ Full Stack

เมื่อใช้ `--api` flag Rails จะ:

1. ไม่รวม middleware ที่ไม่จำเป็นสำหรับ browser
2. ApplicationController extends `ActionController::API` แทน `ActionController::Base`
3. ไม่มี views, helpers, assets
4. เร็วกว่าเพราะ middleware stack เล็กกว่า

```ruby
# config/application.rb - API Mode
module Myapi
  class Application < Rails::Application
    config.load_defaults 7.0
    # API-only mode ถูก set อัตโนมัติ
    config.api_only = true
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  # ใช้ ActionController::API แทน ActionController::Base
  include ActionController::MimeResponds
  
  before_action :set_default_response_format
  
  private
  
  def set_default_response_format
    request.format = :json
  end
end
```

### Gemfile สำหรับ API Project

```ruby
# Gemfile
source "https://rubygems.org"

gem "rails", "~> 7.0"
gem "pg", "~> 1.1"
gem "puma", "~> 5.0"

# Authentication
gem "jwt"
gem "bcrypt", "~> 3.1.7"

# Serialization
gem "active_model_serializers"

# Pagination
gem "pagy"

# CORS
gem "rack-cors"

# Rate Limiting
gem "rack-attack"

# Documentation
gem "rswag"

group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end
```

---

## Step 1012: RESTful API Design Principles

### หลักการออกแบบ RESTful API

REST (Representational State Transfer) มีหลักการสำคัญดังนี้:

1. **Stateless** - แต่ละ request ต้องมีข้อมูลครบ ไม่พึ่ง session
2. **Resource-based** - URL แทน resource ไม่ใช่ action
3. **HTTP Methods** - ใช้ GET, POST, PUT/PATCH, DELETE ตามความหมาย
4. **Uniform Interface** - รูปแบบสม่ำเสมอทั้ง API

### URL Design Best Practices

```
# ดี - Resource-based URLs
GET    /api/v1/articles          # ดึงรายการ articles ทั้งหมด
GET    /api/v1/articles/1        # ดึง article id=1
POST   /api/v1/articles          # สร้าง article ใหม่
PUT    /api/v1/articles/1        # อัปเดต article id=1 (ทั้งหมด)
PATCH  /api/v1/articles/1        # อัปเดต article id=1 (บางส่วน)
DELETE /api/v1/articles/1        # ลบ article id=1

# Nested resources
GET    /api/v1/articles/1/comments     # comments ของ article 1
POST   /api/v1/articles/1/comments     # สร้าง comment ใน article 1

# ไม่ดี - Action-based URLs
GET    /api/v1/getArticles
POST   /api/v1/createArticle
POST   /api/v1/deleteArticle/1
```

---

## Step 1013: HTTP Status Codes

### Status Codes ที่สำคัญสำหรับ API

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      
      # Helper methods สำหรับ response
      private
      
      def render_success(data, status: :ok)
        render json: {
          success: true,
          data: data
        }, status: status
      end
      
      def render_created(data)
        render json: {
          success: true,
          data: data
        }, status: :created  # 201
      end
      
      def render_no_content
        head :no_content  # 204
      end
      
      def render_bad_request(errors)
        render json: {
          success: false,
          errors: errors
        }, status: :bad_request  # 400
      end
      
      def render_unauthorized(message = "Unauthorized")
        render json: {
          success: false,
          error: message
        }, status: :unauthorized  # 401
      end
      
      def render_forbidden(message = "Forbidden")
        render json: {
          success: false,
          error: message
        }, status: :forbidden  # 403
      end
      
      def render_not_found(message = "Not Found")
        render json: {
          success: false,
          error: message
        }, status: :not_found  # 404
      end
      
      def render_unprocessable(errors)
        render json: {
          success: false,
          errors: errors
        }, status: :unprocessable_entity  # 422
      end
      
      def render_server_error(message = "Internal Server Error")
        render json: {
          success: false,
          error: message
        }, status: :internal_server_error  # 500
      end
    end
  end
end
```

### Status Code Reference

| Code | Symbol | ความหมาย | ใช้เมื่อ |
|------|--------|----------|---------|
| 200 | `:ok` | สำเร็จ | GET, PUT, PATCH สำเร็จ |
| 201 | `:created` | สร้างสำเร็จ | POST สำเร็จ |
| 204 | `:no_content` | ไม่มีข้อมูล | DELETE สำเร็จ |
| 400 | `:bad_request` | request ผิด | params ไม่ถูกต้อง |
| 401 | `:unauthorized` | ยังไม่ได้ login | ไม่มี token |
| 403 | `:forbidden` | ไม่มีสิทธิ์ | login แล้วแต่ห้าม |
| 404 | `:not_found` | ไม่พบ | resource ไม่มี |
| 422 | `:unprocessable_entity` | validation fail | model validation error |
| 500 | `:internal_server_error` | server error | unexpected error |

---

## Step 1014: API Versioning

### วิธีการ Versioning

มี 3 วิธีหลัก:

**1. URL Versioning (แนะนำ)**
```
/api/v1/articles
/api/v2/articles
```

**2. Header Versioning**
```
Accept: application/vnd.myapi.v1+json
```

**3. Query Parameter Versioning**
```
/api/articles?version=1
```

### URL Versioning (วิธีที่แนะนำ)

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :articles
      resources :users, only: [:index, :show, :create, :update, :destroy]
      resources :comments, only: [:index, :create, :destroy]
      
      post '/auth/login', to: 'auth#login'
      post '/auth/register', to: 'auth#register'
      delete '/auth/logout', to: 'auth#logout'
    end
    
    namespace :v2 do
      resources :articles do
        member do
          post :like
          delete :unlike
        end
      end
    end
  end
end
```

```
app/controllers/
  api/
    v1/
      base_controller.rb
      articles_controller.rb
      users_controller.rb
      auth_controller.rb
    v2/
      base_controller.rb
      articles_controller.rb
```

---

## Step 1015: สร้าง Complete Versioned API

### Models

```bash
# สร้าง models
rails generate model User name:string email:string password_digest:string role:integer
rails generate model Article title:string body:text user:references published:boolean views_count:integer
rails generate model Comment body:text user:references article:references
rails db:migrate
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  
  has_many :articles, dependent: :destroy
  has_many :comments, dependent: :destroy
  
  enum role: { reader: 0, author: 1, admin: 2 }
  
  validates :name, presence: true, length: { minimum: 2, maximum: 50 }
  validates :email, presence: true, uniqueness: { case_sensitive: false },
                    format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, length: { minimum: 6 }, if: -> { new_record? || !password.nil? }
  
  before_save { email.downcase! }
  
  def self.find_by_email_and_password(email, password)
    user = find_by(email: email.downcase)
    user&.authenticate(password) ? user : nil
  end
end
```

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy
  
  validates :title, presence: true, length: { minimum: 5, maximum: 200 }
  validates :body, presence: true, length: { minimum: 10 }
  
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_author, ->(user_id) { where(user_id: user_id) }
  
  before_create { self.views_count ||= 0 }
  
  def increment_views!
    increment!(:views_count)
  end
end
```

### Controllers V1

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      before_action :authenticate_request!
      
      attr_reader :current_user
      
      rescue_from ActiveRecord::RecordNotFound, with: :handle_not_found
      rescue_from ActiveRecord::RecordInvalid, with: :handle_invalid_record
      rescue_from ActionController::ParameterMissing, with: :handle_bad_request
      
      private
      
      def authenticate_request!
        header = request.headers['Authorization']
        
        if header.nil?
          return render_unauthorized("Missing Authorization header")
        end
        
        token = header.split(' ').last
        decoded = JwtService.decode(token)
        
        if decoded.nil?
          return render_unauthorized("Invalid or expired token")
        end
        
        @current_user = User.find(decoded['user_id'])
      rescue ActiveRecord::RecordNotFound
        render_unauthorized("User not found")
      end
      
      def handle_not_found(e)
        render json: {
          success: false,
          error: "Resource not found",
          message: e.message
        }, status: :not_found
      end
      
      def handle_invalid_record(e)
        render json: {
          success: false,
          errors: e.record.errors.full_messages
        }, status: :unprocessable_entity
      end
      
      def handle_bad_request(e)
        render json: {
          success: false,
          error: "Bad Request",
          message: e.message
        }, status: :bad_request
      end
      
      def render_success(data, meta: nil, status: :ok)
        response = { success: true, data: data }
        response[:meta] = meta if meta
        render json: response, status: status
      end
      
      def render_created(data)
        render json: { success: true, data: data }, status: :created
      end
      
      def render_unauthorized(message = "Unauthorized")
        render json: { success: false, error: message }, status: :unauthorized
      end
      
      def render_forbidden(message = "Forbidden")
        render json: { success: false, error: message }, status: :forbidden
      end
      
      def render_unprocessable(errors)
        render json: { success: false, errors: errors }, status: :unprocessable_entity
      end
    end
  end
end
```

```ruby
# app/controllers/api/v1/articles_controller.rb
module Api
  module V1
    class ArticlesController < BaseController
      skip_before_action :authenticate_request!, only: [:index, :show]
      before_action :set_article, only: [:show, :update, :destroy]
      before_action :authorize_article!, only: [:update, :destroy]
      
      include Pagy::Backend
      
      # GET /api/v1/articles
      def index
        articles = Article.published.recent.includes(:user, :comments)
        
        # Filtering
        articles = articles.by_author(params[:user_id]) if params[:user_id].present?
        articles = articles.where("title ILIKE ?", "%#{params[:search]}%") if params[:search].present?
        
        # Sorting
        if params[:sort].present?
          direction = params[:direction] == 'desc' ? :desc : :asc
          articles = articles.order(params[:sort] => direction)
        end
        
        # Pagination
        pagy, articles = pagy(articles, items: params[:per_page] || 10)
        
        render_success(
          articles.map { |a| ArticleSerializer.new(a).as_json },
          meta: pagy_metadata(pagy)
        )
      end
      
      # GET /api/v1/articles/:id
      def show
        @article.increment_views!
        render_success(ArticleSerializer.new(@article).as_json)
      end
      
      # POST /api/v1/articles
      def create
        article = current_user.articles.build(article_params)
        
        if article.save
          render_created(ArticleSerializer.new(article).as_json)
        else
          render_unprocessable(article.errors.full_messages)
        end
      end
      
      # PATCH /api/v1/articles/:id
      def update
        if @article.update(article_params)
          render_success(ArticleSerializer.new(@article).as_json)
        else
          render_unprocessable(@article.errors.full_messages)
        end
      end
      
      # DELETE /api/v1/articles/:id
      def destroy
        @article.destroy
        head :no_content
      end
      
      private
      
      def set_article
        @article = Article.find(params[:id])
      end
      
      def authorize_article!
        unless @article.user == current_user || current_user.admin?
          render_forbidden("You are not authorized to perform this action")
        end
      end
      
      def article_params
        params.require(:article).permit(:title, :body, :published)
      end
    end
  end
end
```

```ruby
# app/controllers/api/v1/auth_controller.rb
module Api
  module V1
    class AuthController < ApplicationController
      skip_before_action :authenticate_request!
      
      # POST /api/v1/auth/register
      def register
        user = User.new(register_params)
        
        if user.save
          token = JwtService.encode({ user_id: user.id })
          render json: {
            success: true,
            data: {
              user: UserSerializer.new(user).as_json,
              token: token
            }
          }, status: :created
        else
          render json: {
            success: false,
            errors: user.errors.full_messages
          }, status: :unprocessable_entity
        end
      end
      
      # POST /api/v1/auth/login
      def login
        user = User.find_by(email: login_params[:email]&.downcase)
        
        if user&.authenticate(login_params[:password])
          token = JwtService.encode({ 
            user_id: user.id,
            exp: 24.hours.from_now.to_i 
          })
          
          render json: {
            success: true,
            data: {
              user: UserSerializer.new(user).as_json,
              token: token,
              expires_at: 24.hours.from_now
            }
          }
        else
          render json: {
            success: false,
            error: "Invalid email or password"
          }, status: :unauthorized
        end
      end
      
      private
      
      def register_params
        params.require(:user).permit(:name, :email, :password, :password_confirmation)
      end
      
      def login_params
        params.require(:user).permit(:email, :password)
      end
    end
  end
end
```

---

## Step 1016: JWT Authentication

### JWT Service

```ruby
# app/services/jwt_service.rb
class JwtService
  SECRET_KEY = Rails.application.credentials.jwt_secret || ENV['JWT_SECRET'] || 'default_secret_key_change_in_production'
  ALGORITHM = 'HS256'
  
  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] ||= exp.to_i
    JWT.encode(payload, SECRET_KEY, ALGORITHM)
  end
  
  def self.decode(token)
    decoded = JWT.decode(token, SECRET_KEY, true, { algorithm: ALGORITHM })
    decoded.first
  rescue JWT::ExpiredSignature
    Rails.logger.warn "JWT expired"
    nil
  rescue JWT::DecodeError => e
    Rails.logger.warn "JWT decode error: #{e.message}"
    nil
  end
  
  def self.valid?(token)
    !decode(token).nil?
  end
end
```

### ตั้งค่า JWT Secret

```bash
# สร้าง credentials
EDITOR="nano" rails credentials:edit

# เพิ่มใน credentials.yml.enc
jwt_secret: your_very_secure_secret_key_here_at_least_32_characters
```

---

## Step 1017: Serializers

### Active Model Serializers

```bash
# ติดตั้ง gem
# Gemfile: gem 'active_model_serializers'
bundle install
rails generate serializer Article
```

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer < ActiveModel::Serializer
  attributes :id, :title, :body, :published, :views_count, :created_at, :updated_at
  
  belongs_to :user, serializer: UserSerializer
  has_many :comments, serializer: CommentSerializer
  
  attribute :excerpt do
    object.body.truncate(150) if object.body
  end
  
  attribute :comments_count do
    object.comments.count
  end
  
  attribute :reading_time do
    words = object.body.split.length
    minutes = (words / 200.0).ceil
    "#{minutes} min read"
  end
end
```

```ruby
# app/serializers/user_serializer.rb
class UserSerializer < ActiveModel::Serializer
  attributes :id, :name, :email, :role, :created_at
  
  # ไม่แสดง password_digest เด็ดขาด
  
  attribute :articles_count do
    object.articles.count
  end
end
```

```ruby
# app/serializers/comment_serializer.rb
class CommentSerializer < ActiveModel::Serializer
  attributes :id, :body, :created_at
  
  belongs_to :user, serializer: UserSerializer
end
```

### as_json แบบ Custom

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  def as_json(options = {})
    super(options.merge(
      only: [:id, :title, :body, :published, :views_count, :created_at],
      include: {
        user: { only: [:id, :name, :email] },
        comments: {
          only: [:id, :body, :created_at],
          include: { user: { only: [:id, :name] } }
        }
      },
      methods: [:excerpt, :comments_count]
    ))
  end
  
  def excerpt
    body.truncate(150) if body
  end
  
  def comments_count
    comments.count
  end
end
```

---

## Step 1018: Pagination ด้วย Pagy

### Setup Pagy

```ruby
# Gemfile
gem 'pagy'
```

```ruby
# config/initializers/pagy.rb
require 'pagy/extras/metadata'
require 'pagy/extras/headers'

Pagy::DEFAULT[:items] = 10        # default items per page
Pagy::DEFAULT[:max_items] = 100   # maximum items per page
Pagy::DEFAULT[:size] = [1, 4, 4, 1]
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include Pagy::Backend
end
```

### ใช้ Pagy ใน Controller

```ruby
# app/controllers/api/v1/articles_controller.rb
def index
  articles = Article.published.recent.includes(:user)
  
  pagy, articles = pagy(
    articles, 
    items: [params.fetch(:per_page, 10).to_i, 100].min  # max 100 per page
  )
  
  render json: {
    success: true,
    data: articles.map { |a| ArticleSerializer.new(a).as_json },
    meta: {
      total_items: pagy.count,
      total_pages: pagy.pages,
      current_page: pagy.page,
      per_page: pagy.items,
      has_next_page: pagy.next.present?,
      has_prev_page: pagy.prev.present?
    }
  }
end
```

---

## Step 1019: CORS Configuration

### rack-cors Setup

```ruby
# Gemfile
gem 'rack-cors'
```

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    # Development: อนุญาตทุก origin
    origins 'http://localhost:3000', 'http://localhost:5173', 'http://localhost:4200'
    
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,
      expose: ['Authorization', 'X-Total-Count', 'X-Page', 'X-Per-Page']
  end
  
  # Production: ระบุ domain จริง
  allow do
    origins 'https://myfrontend.com', 'https://app.myfrontend.com'
    
    resource '/api/*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: false,
      max_age: 600
  end
end
```

---

## Step 1020: Rate Limiting ด้วย Rack::Attack

### Setup

```ruby
# Gemfile
gem 'rack-attack'
```

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Cache store สำหรับ tracking
  Rack::Attack.cache.store = ActiveSupport::Cache::RedisCacheStore.new(
    url: ENV['REDIS_URL'] || 'redis://localhost:6379/1'
  )
  
  # Throttle: 60 requests ต่อ minute ต่อ IP
  throttle('req/ip', limit: 60, period: 1.minute) do |req|
    req.ip if req.path.start_with?('/api/')
  end
  
  # Throttle login attempts: 5 ครั้งต่อ 20 วินาที ต่อ IP
  throttle('login/ip', limit: 5, period: 20.seconds) do |req|
    if req.path == '/api/v1/auth/login' && req.post?
      req.ip
    end
  end
  
  # Throttle login attempts ต่อ email
  throttle('login/email', limit: 5, period: 20.seconds) do |req|
    if req.path == '/api/v1/auth/login' && req.post?
      req.params['user']['email'].to_s.downcase.gsub(/\s+/, "") if req.params['user']
    end
  end
  
  # Blocklist: IP ที่น่าสงสัย
  blocklist('block_attackers') do |req|
    Ahoy::Visit.where(ip: req.ip).where("created_at > ?", 1.hour.ago).count > 100
  end
  
  # Response สำหรับ throttled requests
  self.throttled_responder = lambda do |req|
    retry_after = (req.env['rack.attack.match_data'] || {})[:period]
    [
      429,
      {
        'Content-Type' => 'application/json',
        'Retry-After' => retry_after.to_s
      },
      [{ 
        success: false, 
        error: 'Too Many Requests',
        retry_after: retry_after
      }.to_json]
    ]
  end
end
```

---

## Step 1021: Error Handling

### Global Error Handler

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      rescue_from StandardError, with: :handle_standard_error
      rescue_from ActiveRecord::RecordNotFound, with: :handle_not_found
      rescue_from ActiveRecord::RecordInvalid, with: :handle_record_invalid
      rescue_from ActionController::ParameterMissing, with: :handle_parameter_missing
      rescue_from JWT::DecodeError, with: :handle_unauthorized
      rescue_from Pundit::NotAuthorizedError, with: :handle_forbidden
      
      private
      
      def handle_standard_error(e)
        Rails.logger.error "#{e.class}: #{e.message}\n#{e.backtrace.first(5).join("\n")}"
        
        if Rails.env.development?
          render json: {
            success: false,
            error: e.message,
            backtrace: e.backtrace.first(10)
          }, status: :internal_server_error
        else
          render json: {
            success: false,
            error: "Something went wrong"
          }, status: :internal_server_error
        end
      end
      
      def handle_not_found(e)
        render json: {
          success: false,
          error: "Resource not found",
          message: e.message
        }, status: :not_found
      end
      
      def handle_record_invalid(e)
        render json: {
          success: false,
          errors: e.record.errors.full_messages
        }, status: :unprocessable_entity
      end
      
      def handle_parameter_missing(e)
        render json: {
          success: false,
          error: "Required parameter missing: #{e.param}"
        }, status: :bad_request
      end
      
      def handle_unauthorized(e)
        render json: {
          success: false,
          error: "Unauthorized",
          message: e.message
        }, status: :unauthorized
      end
      
      def handle_forbidden(e)
        render json: {
          success: false,
          error: "Forbidden",
          message: "You don't have permission to perform this action"
        }, status: :forbidden
      end
    end
  end
end
```

---

## Step 1022: API Documentation ด้วย Swagger/rswag

### Setup rswag

```ruby
# Gemfile
gem 'rswag-api'
gem 'rswag-ui'

group :development, :test do
  gem 'rswag-specs'
end
```

```bash
bundle install
rails generate rswag:api:install
rails generate rswag:ui:install
rails generate rswag:specs:install
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  mount Rswag::Ui::Engine => '/api-docs'
  mount Rswag::Api::Engine => '/api-docs'
  
  namespace :api do
    namespace :v1 do
      resources :articles
    end
  end
end
```

### เขียน Swagger Spec

```ruby
# spec/integration/api/v1/articles_spec.rb
require 'swagger_helper'

RSpec.describe 'Articles API', type: :request do
  path '/api/v1/articles' do
    get 'Retrieves list of articles' do
      tags 'Articles'
      produces 'application/json'
      parameter name: :page, in: :query, type: :integer, description: 'Page number'
      parameter name: :per_page, in: :query, type: :integer, description: 'Items per page'
      parameter name: :search, in: :query, type: :string, description: 'Search query'
      
      response '200', 'articles found' do
        schema type: :object,
               properties: {
                 success: { type: :boolean },
                 data: {
                   type: :array,
                   items: {
                     type: :object,
                     properties: {
                       id: { type: :integer },
                       title: { type: :string },
                       body: { type: :string },
                       published: { type: :boolean },
                       created_at: { type: :string, format: :datetime }
                     }
                   }
                 },
                 meta: {
                   type: :object,
                   properties: {
                     total_items: { type: :integer },
                     total_pages: { type: :integer },
                     current_page: { type: :integer }
                   }
                 }
               }
        
        let(:page) { 1 }
        run_test!
      end
    end
    
    post 'Creates an article' do
      tags 'Articles'
      consumes 'application/json'
      produces 'application/json'
      security [bearerAuth: []]
      
      parameter name: :article, in: :body, schema: {
        type: :object,
        properties: {
          article: {
            type: :object,
            properties: {
              title: { type: :string, example: 'My Article Title' },
              body: { type: :string, example: 'Article content here' },
              published: { type: :boolean, example: true }
            },
            required: ['title', 'body']
          }
        }
      }
      
      response '201', 'article created' do
        let(:article) { { article: { title: 'Test', body: 'Content here' } } }
        run_test!
      end
      
      response '422', 'invalid request' do
        let(:article) { { article: { title: '' } } }
        run_test!
      end
      
      response '401', 'unauthorized' do
        run_test!
      end
    end
  end
end
```

---

## Step 1023: Testing APIs กับ RSpec

### Setup Request Specs

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
  config.include RequestSpecHelper, type: :request
end
```

```ruby
# spec/support/request_spec_helper.rb
module RequestSpecHelper
  def json_response
    JSON.parse(response.body, symbolize_names: true)
  end
  
  def auth_headers(user)
    token = JwtService.encode({ user_id: user.id })
    { 'Authorization' => "Bearer #{token}" }
  end
end
```

```ruby
# spec/factories/user.rb
FactoryBot.define do
  factory :user do
    name { Faker::Name.full_name }
    sequence(:email) { |n| "user#{n}@example.com" }
    password { 'password123' }
    password_confirmation { 'password123' }
    role { :reader }
    
    trait :author do
      role { :author }
    end
    
    trait :admin do
      role { :admin }
    end
  end
end
```

```ruby
# spec/factories/article.rb
FactoryBot.define do
  factory :article do
    title { Faker::Lorem.sentence(word_count: 5) }
    body { Faker::Lorem.paragraphs(number: 3).join("\n\n") }
    published { true }
    views_count { 0 }
    association :user
    
    trait :unpublished do
      published { false }
    end
    
    trait :popular do
      views_count { Faker::Number.between(from: 100, to: 10000) }
    end
  end
end
```

### Request Specs

```ruby
# spec/requests/api/v1/articles_spec.rb
require 'rails_helper'

RSpec.describe "Api::V1::Articles", type: :request do
  let!(:user) { create(:user, :author) }
  let!(:articles) { create_list(:article, 15, user: user) }
  let!(:article) { articles.first }
  
  describe "GET /api/v1/articles" do
    context "without authentication" do
      it "returns list of published articles" do
        get "/api/v1/articles"
        
        expect(response).to have_http_status(:ok)
        expect(json_response[:success]).to be true
        expect(json_response[:data]).to be_an(Array)
      end
      
      it "returns paginated results" do
        get "/api/v1/articles", params: { per_page: 5 }
        
        expect(response).to have_http_status(:ok)
        expect(json_response[:data].length).to eq(5)
        expect(json_response[:meta][:total_items]).to eq(15)
        expect(json_response[:meta][:total_pages]).to eq(3)
      end
      
      it "filters by search query" do
        specific_article = create(:article, title: "Ruby on Rails Tutorial", user: user)
        
        get "/api/v1/articles", params: { search: "Ruby" }
        
        expect(json_response[:data].any? { |a| a[:title].include?("Ruby") }).to be true
      end
    end
  end
  
  describe "GET /api/v1/articles/:id" do
    it "returns a specific article" do
      get "/api/v1/articles/#{article.id}"
      
      expect(response).to have_http_status(:ok)
      expect(json_response[:data][:id]).to eq(article.id)
      expect(json_response[:data][:title]).to eq(article.title)
    end
    
    it "increments views_count" do
      expect {
        get "/api/v1/articles/#{article.id}"
      }.to change { article.reload.views_count }.by(1)
    end
    
    it "returns 404 for non-existent article" do
      get "/api/v1/articles/99999"
      
      expect(response).to have_http_status(:not_found)
      expect(json_response[:success]).to be false
    end
  end
  
  describe "POST /api/v1/articles" do
    context "with valid authentication" do
      let(:valid_params) do
        { article: { title: "New Article Title", body: "This is the article body content", published: true } }
      end
      
      it "creates a new article" do
        expect {
          post "/api/v1/articles", params: valid_params, headers: auth_headers(user)
        }.to change(Article, :count).by(1)
        
        expect(response).to have_http_status(:created)
        expect(json_response[:data][:title]).to eq("New Article Title")
      end
    end
    
    context "without authentication" do
      it "returns 401 unauthorized" do
        post "/api/v1/articles", params: { article: { title: "Test" } }
        
        expect(response).to have_http_status(:unauthorized)
      end
    end
    
    context "with invalid params" do
      it "returns 422 with error messages" do
        post "/api/v1/articles", 
             params: { article: { title: "", body: "" } },
             headers: auth_headers(user)
        
        expect(response).to have_http_status(:unprocessable_entity)
        expect(json_response[:errors]).to be_present
      end
    end
  end
  
  describe "DELETE /api/v1/articles/:id" do
    context "as the owner" do
      it "deletes the article" do
        expect {
          delete "/api/v1/articles/#{article.id}", headers: auth_headers(user)
        }.to change(Article, :count).by(-1)
        
        expect(response).to have_http_status(:no_content)
      end
    end
    
    context "as another user" do
      let(:other_user) { create(:user) }
      
      it "returns 403 forbidden" do
        delete "/api/v1/articles/#{article.id}", headers: auth_headers(other_user)
        
        expect(response).to have_http_status(:forbidden)
      end
    end
  end
end
```

---

## Step 1024: Filtering และ Sorting

```ruby
# app/controllers/concerns/filterable.rb
module Filterable
  extend ActiveSupport::Concern
  
  def filter_articles(articles)
    # Filter by published status
    if params[:published].present?
      articles = articles.where(published: params[:published] == 'true')
    end
    
    # Filter by author
    articles = articles.where(user_id: params[:user_id]) if params[:user_id].present?
    
    # Filter by date range
    if params[:from_date].present?
      articles = articles.where("created_at >= ?", Date.parse(params[:from_date]))
    end
    
    if params[:to_date].present?
      articles = articles.where("created_at <= ?", Date.parse(params[:to_date]).end_of_day)
    end
    
    # Full text search
    if params[:search].present?
      search_term = "%#{params[:search].strip}%"
      articles = articles.where("title ILIKE ? OR body ILIKE ?", search_term, search_term)
    end
    
    articles
  end
  
  def sort_articles(articles)
    allowed_sort_columns = %w[created_at updated_at title views_count]
    allowed_directions = %w[asc desc]
    
    sort_by = allowed_sort_columns.include?(params[:sort]) ? params[:sort] : 'created_at'
    direction = allowed_directions.include?(params[:direction]) ? params[:direction] : 'desc'
    
    articles.order(sort_by => direction)
  end
end
```

---

## Step 1025: V2 API กับ Additional Features

```ruby
# app/controllers/api/v2/articles_controller.rb
module Api
  module V2
    class ArticlesController < Api::V1::ArticlesController
      # เพิ่ม features ใหม่ใน V2
      
      # POST /api/v2/articles/:id/like
      def like
        if current_user.liked_articles.include?(@article)
          render json: { success: false, error: "Already liked" }, status: :bad_request
        else
          current_user.liked_articles << @article
          render json: { 
            success: true, 
            data: { likes_count: @article.likes.count }
          }
        end
      end
      
      # DELETE /api/v2/articles/:id/unlike
      def unlike
        current_user.liked_articles.delete(@article)
        render json: { 
          success: true,
          data: { likes_count: @article.likes.count }
        }
      end
      
      private
      
      # V2 ใช้ serializer ที่ครอบคลุมมากกว่า
      def serialize_article(article)
        ArticleV2Serializer.new(article, { 
          current_user: current_user,
          include: [:user, :tags, :comments]
        }).as_json
      end
    end
  end
end
```

---

## แบบฝึกหัด (25 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** สร้าง Rails API project ใหม่ด้วย `--api` flag และอธิบายความแตกต่างจาก full stack Rails
```bash
# เฉลย
rails new blog_api --api --database=postgresql
# ความแตกต่าง:
# - ApplicationController extends ActionController::API
# - ไม่มี views, helpers, assets pipeline
# - middleware stack เล็กกว่า (เร็วกว่า)
# - ไม่มี cookie/session middleware (โดย default)
```

**ข้อ 2:** เขียน routes สำหรับ API v1 ที่มี resources: products, categories, orders
```ruby
# เฉลย
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :products
      resources :categories
      resources :orders, only: [:index, :show, :create, :update]
    end
  end
end
```

**ข้อ 3:** สร้าง model Product ที่มี fields: name, price, description, stock_quantity, category_id
```bash
# เฉลย
rails generate model Product name:string price:decimal description:text stock_quantity:integer category:references
rails db:migrate
```

**ข้อ 4:** เขียน controller action ที่ return JSON response พร้อม status codes ที่ถูกต้อง
```ruby
# เฉลย
def show
  product = Product.find(params[:id])
  render json: { success: true, data: product }, status: :ok
rescue ActiveRecord::RecordNotFound
  render json: { success: false, error: "Product not found" }, status: :not_found
end
```

**ข้อ 5:** สร้าง JwtService class ที่สามารถ encode และ decode token ได้
```ruby
# เฉลย
class JwtService
  SECRET = ENV.fetch('JWT_SECRET') { 'fallback_dev_secret' }
  
  def self.encode(payload)
    payload[:exp] = 24.hours.from_now.to_i
    JWT.encode(payload, SECRET, 'HS256')
  end
  
  def self.decode(token)
    JWT.decode(token, SECRET, true, algorithm: 'HS256').first
  rescue JWT::DecodeError
    nil
  end
end
```

### ระดับกลาง

**ข้อ 6:** เพิ่ม authentication ใน BaseController ที่ตรวจสอบ JWT token จาก Authorization header
```ruby
# เฉลย
def authenticate_request!
  token = request.headers['Authorization']&.split(' ')&.last
  payload = JwtService.decode(token)
  
  unless payload
    return render json: { error: 'Unauthorized' }, status: :unauthorized
  end
  
  @current_user = User.find(payload['user_id'])
rescue ActiveRecord::RecordNotFound
  render json: { error: 'User not found' }, status: :unauthorized
end
```

**ข้อ 7:** เพิ่ม pagination ใน index action ด้วย Pagy gem
```ruby
# เฉลย
include Pagy::Backend

def index
  pagy, products = pagy(Product.all, items: params.fetch(:per_page, 10).to_i)
  render json: {
    data: products,
    meta: {
      total: pagy.count,
      page: pagy.page,
      pages: pagy.pages
    }
  }
end
```

**ข้อ 8:** สร้าง ProductSerializer ด้วย Active Model Serializers ที่แสดง category name
```ruby
# เฉลย
class ProductSerializer < ActiveModel::Serializer
  attributes :id, :name, :price, :description, :stock_quantity
  
  attribute :category_name do
    object.category&.name
  end
  
  attribute :in_stock do
    object.stock_quantity > 0
  end
end
```

**ข้อ 9:** เพิ่ม CORS configuration ที่อนุญาต frontend ที่ port 3000 และ 5173
```ruby
# เฉลย
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'http://localhost:3000', 'http://localhost:5173'
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options]
  end
end
```

**ข้อ 10:** เพิ่ม rate limiting ที่จำกัด 30 requests ต่อ minute ต่อ IP สำหรับ API routes
```ruby
# เฉลย
throttle('api/ip', limit: 30, period: 1.minute) do |req|
  req.ip if req.path.start_with?('/api/')
end
```

### ระดับสูง

**ข้อ 11:** เขียน request spec ที่ทดสอบ CRUD operations ของ Products API ทั้งหมด
```ruby
# เฉลย
RSpec.describe "Products API", type: :request do
  let(:user) { create(:user) }
  let(:headers) { { 'Authorization' => "Bearer #{JwtService.encode(user_id: user.id)}" } }
  let!(:product) { create(:product) }
  
  it "GET /api/v1/products returns products" do
    get '/api/v1/products'
    expect(response).to have_http_status(:ok)
    expect(json_response[:data]).to be_an Array
  end
  
  it "POST /api/v1/products creates product" do
    post '/api/v1/products', 
         params: { product: { name: 'Test', price: 100 } },
         headers: headers
    expect(response).to have_http_status(:created)
  end
  
  it "PUT /api/v1/products/:id updates product" do
    put "/api/v1/products/#{product.id}",
        params: { product: { name: 'Updated' } },
        headers: headers
    expect(response).to have_http_status(:ok)
  end
  
  it "DELETE /api/v1/products/:id deletes product" do
    delete "/api/v1/products/#{product.id}", headers: headers
    expect(response).to have_http_status(:no_content)
  end
end
```

**ข้อ 12:** สร้าง V2 API ที่มี additional endpoint สำหรับ bulk update products
```ruby
# เฉลย
module Api
  module V2
    class ProductsController < Api::V1::ProductsController
      def bulk_update
        results = params[:products].map do |product_params|
          product = Product.find_by(id: product_params[:id])
          next { id: product_params[:id], error: "Not found" } unless product
          
          if product.update(product_params.permit(:name, :price, :stock_quantity))
            { id: product.id, success: true }
          else
            { id: product.id, errors: product.errors.full_messages }
          end
        end
        
        render json: { success: true, data: results }
      end
    end
  end
end
```

**ข้อ 13:** เพิ่ม search และ filter functionality ใน index action
```ruby
# เฉลย
def index
  products = Product.all
  
  products = products.where("name ILIKE ?", "%#{params[:search]}%") if params[:search].present?
  products = products.where(category_id: params[:category_id]) if params[:category_id].present?
  products = products.where("price >= ?", params[:min_price]) if params[:min_price].present?
  products = products.where("price <= ?", params[:max_price]) if params[:max_price].present?
  products = products.where("stock_quantity > 0") if params[:in_stock] == 'true'
  
  pagy, products = pagy(products.order(created_at: :desc))
  render_success(products.map { |p| serialize_product(p) }, meta: pagy_metadata(pagy))
end
```

**ข้อ 14:** เพิ่ม Swagger documentation สำหรับ Products API
```ruby
# เฉลย
path '/api/v1/products' do
  get 'List products' do
    tags 'Products'
    parameter name: :search, in: :query, type: :string
    parameter name: :page, in: :query, type: :integer
    
    response '200', 'success' do
      schema '$ref' => '#/components/schemas/ProductListResponse'
      run_test!
    end
  end
end
```

**ข้อ 15:** เขียน custom error handler สำหรับ specific exceptions ทั้งหมด
```ruby
# เฉลย
rescue_from ArgumentError do |e|
  render json: { error: "Invalid argument: #{e.message}" }, status: :bad_request
end

rescue_from ActiveRecord::RecordNotUnique do |e|
  render json: { error: "Record already exists" }, status: :conflict
end
```

**ข้อ 16-25:** (แบบฝึกหัดเพิ่มเติม)

**ข้อ 16:** สร้าง API endpoint สำหรับ user profile update พร้อม avatar upload
**ข้อ 17:** implement refresh token mechanism
**ข้อ 18:** เพิ่ม API key authentication แทน JWT สำหรับ B2B API
**ข้อ 19:** เขียน middleware ที่ log ทุก API request พร้อม duration
**ข้อ 20:** สร้าง consistent error response format ด้วย concern module
**ข้อ 21:** implement cursor-based pagination แทน offset-based
**ข้อ 22:** สร้าง webhook endpoint ที่รับ POST requests จาก third party
**ข้อ 23:** เพิ่ม request ID tracking ใน headers และ logs
**ข้อ 24:** implement API deprecation notices ใน headers
**ข้อ 25:** สร้าง health check endpoint `/api/health` ที่ตรวจสอบ database และ redis connectivity

```ruby
# เฉลย ข้อ 25
# app/controllers/api/health_controller.rb
module Api
  class HealthController < ApplicationController
    skip_before_action :authenticate_request!
    
    def index
      checks = {
        status: 'ok',
        timestamp: Time.current.iso8601,
        version: '1.0.0',
        checks: {}
      }
      
      # Database check
      begin
        ActiveRecord::Base.connection.execute("SELECT 1")
        checks[:checks][:database] = { status: 'ok' }
      rescue => e
        checks[:checks][:database] = { status: 'error', message: e.message }
        checks[:status] = 'degraded'
      end
      
      # Redis check
      begin
        Redis.new(url: ENV.fetch('REDIS_URL', 'redis://localhost:6379')).ping
        checks[:checks][:redis] = { status: 'ok' }
      rescue => e
        checks[:checks][:redis] = { status: 'error', message: e.message }
        checks[:status] = 'degraded'
      end
      
      status_code = checks[:status] == 'ok' ? :ok : :service_unavailable
      render json: checks, status: status_code
    end
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Rails API Mode** - การสร้าง API-only Rails app
2. **RESTful Design** - หลักการออกแบบ API ที่ดี
3. **Status Codes** - การใช้ HTTP status codes อย่างถูกต้อง
4. **Versioning** - การทำ API versioning แบบ URL
5. **JWT Authentication** - การ authenticate ด้วย JWT
6. **Serializers** - การจัดการ JSON output
7. **Pagination** - การทำ pagination ด้วย Pagy
8. **CORS** - การตั้งค่า Cross-Origin Resource Sharing
9. **Rate Limiting** - การป้องกัน abuse
10. **Error Handling** - การจัดการ errors อย่างสม่ำเสมอ
11. **Testing** - การทดสอบ API ด้วย RSpec request specs
12. **Documentation** - การทำ API docs ด้วย Swagger

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ JSON Serialization อย่างละเอียดมากขึ้น
