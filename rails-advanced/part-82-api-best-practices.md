# Part 82: API Best Practices ใน Rails

## บทนำ

การออกแบบ API ที่ดีเป็นหัวใจสำคัญของการพัฒนาแอปพลิเคชันสมัยใหม่ ในบทนี้เราจะเรียนรู้แนวทางปฏิบัติที่ดีที่สุดสำหรับการสร้าง REST API ด้วย Rails ตั้งแต่การออกแบบ versioning ไปจนถึงการทดสอบและ documentation

---

## 1. REST API Design: Versioning

การทำ versioning ใน API เป็นสิ่งจำเป็นเพื่อรองรับการเปลี่ยนแปลงโดยไม่ทำให้ client เก่าพัง

### 1.1 URL Path Versioning

วิธีที่พบบ่อยที่สุดคือการใส่ version ใน URL path

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :users
      resources :products
      resources :orders
    end
    
    namespace :v2 do
      resources :users
      resources :products
    end
  end
end
```

```ruby
# app/controllers/api/v1/users_controller.rb
module Api
  module V1
    class UsersController < ApplicationController
      def index
        users = User.all.limit(params[:per_page] || 25)
        render json: {
          data: users.map { |u| serialize_user(u) },
          meta: {
            total: User.count,
            version: 'v1'
          }
        }
      end
      
      def show
        user = User.find(params[:id])
        render json: { data: serialize_user(user) }
      rescue ActiveRecord::RecordNotFound
        render json: { error: 'User not found' }, status: :not_found
      end
      
      private
      
      def serialize_user(user)
        {
          id: user.id,
          name: user.name,
          email: user.email,
          created_at: user.created_at.iso8601
        }
      end
    end
  end
end
```

```ruby
# app/controllers/api/v2/users_controller.rb
module Api
  module V2
    class UsersController < ApplicationController
      def index
        users = User.all.limit(params[:per_page] || 25)
        render json: {
          data: users.map { |u| serialize_user(u) },
          meta: {
            total: User.count,
            version: 'v2',
            links: {
              self: api_v2_users_url,
              next: api_v2_users_url(page: 2)
            }
          }
        }
      end
      
      private
      
      def serialize_user(user)
        {
          id: user.id,
          type: 'user',
          attributes: {
            name: user.name,
            email: user.email,
            full_name: "#{user.first_name} #{user.last_name}",
            created_at: user.created_at.iso8601
          },
          links: {
            self: api_v2_user_url(user)
          }
        }
      end
    end
  end
end
```

### 1.2 Header-based Versioning

ใช้ HTTP Header เพื่อระบุ version

```ruby
# config/routes.rb
Rails.application.routes.draw do
  scope :api do
    resources :users, controller: 'api/users'
    resources :products, controller: 'api/products'
  end
end
```

```ruby
# app/controllers/api/base_controller.rb
module Api
  class BaseController < ApplicationController
    before_action :set_api_version
    
    private
    
    def set_api_version
      @api_version = request.headers['Accept-Version'] || 
                     request.headers['Api-Version'] ||
                     'v1'
    end
    
    def api_version
      @api_version
    end
  end
end
```

```ruby
# app/controllers/api/users_controller.rb
module Api
  class UsersController < Api::BaseController
    def index
      case api_version
      when 'v1'
        render_v1_response
      when 'v2'
        render_v2_response
      else
        render json: { error: "API version #{api_version} not supported" },
               status: :not_acceptable
      end
    end
    
    private
    
    def render_v1_response
      users = User.select(:id, :name, :email)
      render json: users
    end
    
    def render_v2_response
      users = User.all.page(params[:page]).per(params[:per_page])
      render json: {
        users: users.as_json(only: [:id, :name, :email, :created_at]),
        pagination: {
          current_page: users.current_page,
          total_pages: users.total_pages,
          total_count: users.total_count
        }
      }
    end
  end
end
```

### 1.3 Query Parameter Versioning

```ruby
# config/routes.rb
Rails.application.routes.draw do
  scope :api do
    resources :users
  end
end
```

```ruby
# app/controllers/api_controller.rb
class ApiController < ApplicationController
  before_action :validate_version
  
  SUPPORTED_VERSIONS = %w[v1 v2 v3].freeze
  DEFAULT_VERSION = 'v1'
  
  private
  
  def validate_version
    version = params[:version] || DEFAULT_VERSION
    
    unless SUPPORTED_VERSIONS.include?(version)
      render json: {
        error: "Unsupported API version: #{version}",
        supported_versions: SUPPORTED_VERSIONS
      }, status: :bad_request
      return
    end
    
    @version = version
  end
  
  def current_version
    @version || DEFAULT_VERSION
  end
end
```

### 1.4 เปรียบเทียบวิธี Versioning

| วิธี | ข้อดี | ข้อเสีย |
|------|-------|---------|
| URL Path | ง่ายใช้ง่าย, ชัดเจน, cacheable | URL ยาว, ต้องเปลี่ยน URL |
| Header | Clean URL, flexible | ยากต่อการทดสอบใน browser |
| Query Param | ง่ายทดสอบ | มักถือว่าไม่ใช่ best practice |

---

## 2. HATEOAS Implementation ใน Rails

HATEOAS (Hypermedia As The Engine Of Application State) เป็นหลักการที่ API response ควรมี links ที่บอก client ว่าจะทำอะไรได้ต่อไป

### 2.1 พื้นฐาน HATEOAS

```ruby
# app/serializers/base_serializer.rb
class BaseSerializer
  def initialize(resource, options = {})
    @resource = resource
    @options = options
    @request = options[:request]
  end
  
  private
  
  attr_reader :resource, :options, :request
  
  def base_url
    @request&.base_url || 'http://api.example.com'
  end
end
```

```ruby
# app/serializers/user_serializer.rb
class UserSerializer < BaseSerializer
  def serialize
    {
      data: user_data,
      links: user_links,
      meta: meta_data
    }
  end
  
  def serialize_collection(users, total:)
    {
      data: users.map { |u| user_data_for(u) },
      links: collection_links(total),
      meta: {
        total: total,
        page: options[:page] || 1,
        per_page: options[:per_page] || 25
      }
    }
  end
  
  private
  
  def user_data
    user_data_for(resource)
  end
  
  def user_data_for(user)
    {
      id: user.id,
      type: 'user',
      attributes: {
        name: user.name,
        email: user.email,
        role: user.role,
        created_at: user.created_at.iso8601,
        updated_at: user.updated_at.iso8601
      },
      relationships: relationships_for(user),
      links: {
        self: "#{base_url}/api/v1/users/#{user.id}"
      }
    }
  end
  
  def relationships_for(user)
    {
      posts: {
        links: {
          related: "#{base_url}/api/v1/users/#{user.id}/posts"
        },
        meta: {
          count: user.posts_count
        }
      },
      profile: {
        links: {
          related: "#{base_url}/api/v1/users/#{user.id}/profile"
        }
      }
    }
  end
  
  def user_links
    {
      self: "#{base_url}/api/v1/users/#{resource.id}",
      edit: "#{base_url}/api/v1/users/#{resource.id}/edit",
      posts: "#{base_url}/api/v1/users/#{resource.id}/posts",
      avatar: "#{base_url}/api/v1/users/#{resource.id}/avatar"
    }
  end
  
  def collection_links(total)
    page = options[:page] || 1
    per_page = options[:per_page] || 25
    total_pages = (total.to_f / per_page).ceil
    
    links = {
      self: "#{base_url}/api/v1/users?page=#{page}&per_page=#{per_page}",
      first: "#{base_url}/api/v1/users?page=1&per_page=#{per_page}",
      last: "#{base_url}/api/v1/users?page=#{total_pages}&per_page=#{per_page}"
    }
    
    links[:prev] = "#{base_url}/api/v1/users?page=#{page - 1}&per_page=#{per_page}" if page > 1
    links[:next] = "#{base_url}/api/v1/users?page=#{page + 1}&per_page=#{per_page}" if page < total_pages
    
    links
  end
  
  def meta_data
    {
      actions: available_actions
    }
  end
  
  def available_actions
    actions = ['read']
    actions << 'update' if options[:can_update]
    actions << 'delete' if options[:can_delete]
    actions
  end
end
```

### 2.2 JSON:API Standard

```ruby
# Gemfile
gem 'jsonapi-serializer'

# app/serializers/user_serializer.rb
class UserSerializer
  include JSONAPI::Serializer
  
  set_type :user
  set_id :id
  
  attributes :name, :email, :role, :created_at
  
  attribute :full_name do |user|
    "#{user.first_name} #{user.last_name}"
  end
  
  belongs_to :organization
  has_many :posts
  has_many :comments
  
  link :self do |user|
    "/api/v1/users/#{user.id}"
  end
  
  meta do |user|
    {
      created_days_ago: (Date.current - user.created_at.to_date).to_i,
      posts_count: user.posts_count,
      active: user.active?
    }
  end
end
```

```ruby
# app/controllers/api/v1/users_controller.rb
module Api
  module V1
    class UsersController < ApplicationController
      def index
        users = User.includes(:organization).page(params[:page]).per(params[:per_page])
        
        render json: UserSerializer.new(
          users,
          meta: {
            total: User.count,
            page: params[:page] || 1
          },
          links: {
            self: request.url
          }
        ).serializable_hash
      end
      
      def show
        user = User.find(params[:id])
        render json: UserSerializer.new(user).serializable_hash
      end
      
      def create
        user = User.new(user_params)
        
        if user.save
          render json: UserSerializer.new(user).serializable_hash, 
                 status: :created,
                 location: api_v1_user_url(user)
        else
          render json: {
            errors: format_errors(user.errors)
          }, status: :unprocessable_entity
        end
      end
      
      private
      
      def user_params
        params.require(:user).permit(:name, :email, :password, :role)
      end
      
      def format_errors(errors)
        errors.map do |error|
          {
            source: { pointer: "/data/attributes/#{error.attribute}" },
            title: error.message,
            detail: "#{error.attribute.to_s.humanize} #{error.message}"
          }
        end
      end
    end
  end
end
```

---

## 3. API Rate Limiting ด้วย Rack::Attack

Rate limiting ป้องกัน API จากการถูกใช้งานเกินขอบเขต

### 3.1 การติดตั้ง Rack::Attack

```ruby
# Gemfile
gem 'rack-attack'

# config/application.rb
config.middleware.use Rack::Attack
```

### 3.2 การตั้งค่า Rate Limiting

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # ใช้ Redis สำหรับ production
  Rack::Attack.cache.store = ActiveSupport::Cache::RedisCacheStore.new(
    url: ENV['REDIS_URL'] || 'redis://localhost:6379/1'
  )
  
  # ============================================================
  # Throttles (Rate Limits)
  # ============================================================
  
  # จำกัด requests ทั่วไปจาก IP เดียว: 300 requests ต่อ 5 นาที
  throttle('req/ip', limit: 300, period: 5.minutes) do |req|
    req.ip unless req.path.start_with?('/assets')
  end
  
  # จำกัด login attempts: 5 ต่อ 20 วินาที
  throttle('logins/ip', limit: 5, period: 20.seconds) do |req|
    if req.path == '/api/v1/auth/login' && req.post?
      req.ip
    end
  end
  
  # จำกัดตาม API key: 1000 requests ต่อชั่วโมง
  throttle('api/key', limit: 1000, period: 1.hour) do |req|
    req.env['HTTP_X_API_KEY'] if req.path.start_with?('/api/')
  end
  
  # จำกัด signup: 3 accounts ต่อ IP ต่อวัน
  throttle('sign_up/ip', limit: 3, period: 24.hours) do |req|
    if req.path == '/api/v1/users' && req.post?
      req.ip
    end
  end
  
  # จำกัด password reset: 5 requests ต่อ 10 นาที ต่อ email
  throttle('password_reset/email', limit: 5, period: 10.minutes) do |req|
    if req.path == '/api/v1/auth/password_reset' && req.post?
      req.params['email'].to_s.downcase.gsub(/\s+/, '')
    end
  end
  
  # ============================================================
  # Blocklists
  # ============================================================
  
  # Block IPs จาก environment variable
  blocklist('block_ips') do |req|
    blocked_ips = ENV['BLOCKED_IPS'].to_s.split(',')
    blocked_ips.include?(req.ip)
  end
  
  # Block bots ที่รู้จัก
  blocklist('block_bots') do |req|
    user_agent = req.user_agent.to_s.downcase
    malicious_agents = ['scrapy', 'python-requests/2.x', 'curl/bad-bot']
    malicious_agents.any? { |agent| user_agent.include?(agent) }
  end
  
  # ============================================================
  # Allowlists
  # ============================================================
  
  # อนุญาต localhost ในทุกกรณี
  safelist('allow-localhost') do |req|
    ['127.0.0.1', '::1'].include?(req.ip)
  end
  
  # อนุญาต internal network
  safelist('allow-internal') do |req|
    req.ip.start_with?('10.0.') || req.ip.start_with?('192.168.')
  end
  
  # ============================================================
  # Custom Response สำหรับ Rate Limit
  # ============================================================
  
  self.throttled_responder = lambda do |env|
    now = Time.now
    match_data = env['rack.attack.match_data']
    
    headers = {
      'Content-Type' => 'application/json',
      'X-RateLimit-Limit' => match_data[:limit].to_s,
      'X-RateLimit-Remaining' => '0',
      'X-RateLimit-Reset' => (now + (match_data[:period] - now.to_i % match_data[:period])).to_i.to_s,
      'Retry-After' => (match_data[:period] - now.to_i % match_data[:period]).to_s
    }
    
    body = {
      error: {
        code: 'rate_limit_exceeded',
        message: 'API rate limit exceeded. Please retry after some time.',
        limit: match_data[:limit],
        reset_at: Time.at(headers['X-RateLimit-Reset'].to_i).iso8601
      }
    }.to_json
    
    [429, headers, [body]]
  end
  
  # ============================================================
  # Logging
  # ============================================================
  
  ActiveSupport::Notifications.subscribe('rack.attack') do |name, start, finish, request_id, payload|
    req = payload[:request]
    
    if req.env['rack.attack.matched']
      Rails.logger.warn(
        "[Rack::Attack] #{req.env['rack.attack.match_type']} " \
        "#{req.env['rack.attack.matched']}: " \
        "#{req.ip} #{req.request_method} #{req.fullpath}"
      )
    end
  end
end
```

### 3.3 Dynamic Rate Limiting ตาม User Plan

```ruby
# config/initializers/rack_attack.rb (เพิ่มเติม)

# Throttle ตาม user tier
throttle('api/user_tier', limit: ->(req) {
  api_key = req.env['HTTP_X_API_KEY']
  return 100 unless api_key
  
  # ดึง rate limit จาก cache
  Rails.cache.fetch("rate_limit:#{api_key}", expires_in: 1.hour) do
    key = ApiKey.find_by(key: api_key)
    case key&.plan
    when 'enterprise' then 10_000
    when 'professional' then 3_000
    when 'basic' then 500
    else 100
    end
  end
}, period: 1.hour) do |req|
  req.env['HTTP_X_API_KEY'] if req.path.start_with?('/api/')
end
```

### 3.4 Rack::Attack Controller Integration

```ruby
# app/controllers/concerns/rate_limit_headers.rb
module RateLimitHeaders
  extend ActiveSupport::Concern
  
  included do
    before_action :add_rate_limit_headers
  end
  
  private
  
  def add_rate_limit_headers
    api_key = request.headers['X-Api-Key']
    return unless api_key
    
    limit = get_rate_limit(api_key)
    used = get_rate_limit_used(api_key)
    reset_at = get_rate_limit_reset(api_key)
    
    response.headers['X-RateLimit-Limit'] = limit.to_s
    response.headers['X-RateLimit-Remaining'] = [limit - used, 0].max.to_s
    response.headers['X-RateLimit-Reset'] = reset_at.to_i.to_s
  end
  
  def get_rate_limit(api_key)
    Rails.cache.fetch("rate_limit:#{api_key}:limit", expires_in: 1.hour) do
      ApiKey.find_by(key: api_key)&.rate_limit || 1000
    end
  end
  
  def get_rate_limit_used(api_key)
    cache_key = "rack::attack:#{Time.now.utc.strftime('%Y-%m-%dT%H')}:api/key:#{api_key}"
    Rails.cache.read(cache_key).to_i
  end
  
  def get_rate_limit_reset(api_key)
    Time.now.utc.end_of_hour
  end
end
```

---

## 4. API Documentation ด้วย rswag (Swagger/OpenAPI)

### 4.1 การติดตั้ง rswag

```ruby
# Gemfile
gem 'rswag-api'
gem 'rswag-ui'

group :development, :test do
  gem 'rswag-specs'
end

# bundle install
# rails generate rswag:install
```

```ruby
# config/initializers/rswag_ui.rb
Rswag::Ui.configure do |c|
  c.swagger_endpoint '/api-docs/v1/swagger.yaml', 'API V1 Docs'
  c.swagger_endpoint '/api-docs/v2/swagger.yaml', 'API V2 Docs'
  
  # เปิด authentication ใน Swagger UI
  c.basic_auth_enabled = true
  c.basic_auth_credentials 'admin', Rails.application.credentials.swagger_password
  
  # custom CSS
  c.config_object = {
    deepLinking: true,
    displayRequestDuration: true,
    docExpansion: 'list',
    filter: true
  }
end
```

### 4.2 Swagger Spec สำหรับ Users API

```ruby
# spec/integration/users_spec.rb
require 'swagger_helper'

RSpec.describe 'Users API', type: :request do
  path '/api/v1/users' do
    get 'Returns list of users' do
      tags 'Users'
      description 'Retrieves all users with pagination support'
      operationId 'listUsers'
      produces 'application/json'
      
      security [{ bearerAuth: [] }]
      
      parameter name: :page, 
                in: :query, 
                type: :integer, 
                description: 'Page number',
                default: 1,
                required: false
                
      parameter name: :per_page, 
                in: :query, 
                type: :integer, 
                description: 'Items per page (max: 100)',
                default: 25,
                required: false
                
      parameter name: :sort,
                in: :query,
                type: :string,
                enum: ['name', '-name', 'created_at', '-created_at'],
                description: 'Sort order (prefix - for descending)',
                required: false
      
      response '200', 'Successful response' do
        schema type: :object,
               properties: {
                 data: {
                   type: :array,
                   items: {
                     '$ref' => '#/components/schemas/User'
                   }
                 },
                 meta: {
                   type: :object,
                   properties: {
                     total: { type: :integer },
                     page: { type: :integer },
                     per_page: { type: :integer },
                     total_pages: { type: :integer }
                   }
                 },
                 links: {
                   '$ref' => '#/components/schemas/PaginationLinks'
                 }
               },
               required: ['data', 'meta']
        
        let(:Authorization) { "Bearer #{create_auth_token}" }
        run_test!
      end
      
      response '401', 'Unauthorized' do
        schema '$ref' => '#/components/schemas/UnauthorizedError'
        run_test!
      end
      
      response '429', 'Too Many Requests' do
        schema '$ref' => '#/components/schemas/RateLimitError'
        run_test!
      end
    end
    
    post 'Creates a user' do
      tags 'Users'
      description 'Creates a new user account'
      consumes 'application/json'
      produces 'application/json'
      
      security [{ bearerAuth: [] }]
      
      parameter name: :user, in: :body, schema: {
        type: :object,
        properties: {
          name: { type: :string, example: 'John Doe' },
          email: { type: :string, format: :email, example: 'john@example.com' },
          password: { type: :string, format: :password, minLength: 8 },
          role: { type: :string, enum: ['admin', 'user', 'guest'], default: 'user' }
        },
        required: ['name', 'email', 'password']
      }
      
      response '201', 'User created' do
        schema type: :object,
               properties: {
                 data: { '$ref' => '#/components/schemas/User' }
               }
        
        let(:Authorization) { "Bearer #{create_auth_token(role: 'admin')}" }
        let(:user) { { name: 'Test User', email: 'test@example.com', password: 'password123' } }
        run_test!
      end
      
      response '422', 'Validation failed' do
        schema '$ref' => '#/components/schemas/ValidationError'
        
        let(:Authorization) { "Bearer #{create_auth_token(role: 'admin')}" }
        let(:user) { { name: '', email: 'invalid-email', password: '123' } }
        run_test!
      end
    end
  end
  
  path '/api/v1/users/{id}' do
    get 'Returns a user' do
      tags 'Users'
      produces 'application/json'
      
      security [{ bearerAuth: [] }]
      
      parameter name: :id, in: :path, type: :integer, required: true
      
      response '200', 'User found' do
        schema type: :object,
               properties: {
                 data: { '$ref' => '#/components/schemas/User' }
               }
        
        let(:Authorization) { "Bearer #{create_auth_token}" }
        let(:id) { create(:user).id }
        run_test!
      end
      
      response '404', 'User not found' do
        schema '$ref' => '#/components/schemas/NotFoundError'
        
        let(:Authorization) { "Bearer #{create_auth_token}" }
        let(:id) { 999999 }
        run_test!
      end
    end
  end
end
```

### 4.3 Swagger Components/Schemas

```ruby
# spec/swagger_helper.rb
require 'rails_helper'

RSpec.configure do |config|
  config.swagger_root = Rails.root.join('swagger').to_s
  
  config.swagger_docs = {
    'v1/swagger.yaml' => {
      openapi: '3.0.1',
      info: {
        title: 'My App API',
        version: 'v1',
        description: 'API documentation for My App',
        contact: {
          name: 'API Support',
          email: 'api@example.com',
          url: 'https://example.com/api-support'
        },
        license: {
          name: 'MIT',
          url: 'https://opensource.org/licenses/MIT'
        }
      },
      servers: [
        { url: 'https://api.example.com', description: 'Production' },
        { url: 'https://staging-api.example.com', description: 'Staging' },
        { url: 'http://localhost:3000', description: 'Development' }
      ],
      components: {
        securitySchemes: {
          bearerAuth: {
            type: :http,
            scheme: :bearer,
            bearerFormat: :JWT
          },
          apiKey: {
            type: :apiKey,
            name: 'X-Api-Key',
            in: :header
          }
        },
        schemas: {
          User: {
            type: :object,
            properties: {
              id: { type: :integer, example: 1 },
              type: { type: :string, example: 'user' },
              attributes: {
                type: :object,
                properties: {
                  name: { type: :string, example: 'John Doe' },
                  email: { type: :string, example: 'john@example.com' },
                  role: { type: :string, enum: ['admin', 'user', 'guest'] },
                  created_at: { type: :string, format: 'date-time' }
                }
              },
              links: {
                type: :object,
                properties: {
                  self: { type: :string }
                }
              }
            }
          },
          PaginationLinks: {
            type: :object,
            properties: {
              self: { type: :string },
              first: { type: :string },
              last: { type: :string },
              prev: { type: :string, nullable: true },
              next: { type: :string, nullable: true }
            }
          },
          ValidationError: {
            type: :object,
            properties: {
              errors: {
                type: :array,
                items: {
                  type: :object,
                  properties: {
                    source: {
                      type: :object,
                      properties: {
                        pointer: { type: :string }
                      }
                    },
                    title: { type: :string },
                    detail: { type: :string }
                  }
                }
              }
            }
          },
          UnauthorizedError: {
            type: :object,
            properties: {
              error: { type: :string, example: 'Unauthorized' },
              message: { type: :string }
            }
          },
          NotFoundError: {
            type: :object,
            properties: {
              error: { type: :string, example: 'Not Found' },
              message: { type: :string }
            }
          },
          RateLimitError: {
            type: :object,
            properties: {
              error: {
                type: :object,
                properties: {
                  code: { type: :string, example: 'rate_limit_exceeded' },
                  message: { type: :string },
                  reset_at: { type: :string, format: 'date-time' }
                }
              }
            }
          }
        }
      },
      security: [{ bearerAuth: [] }]
    }
  }
  
  config.swagger_format = :yaml
end
```

---

## 5. API Testing ด้วย RSpec Request Specs

### 5.1 โครงสร้าง Request Spec

```ruby
# spec/requests/api/v1/users_spec.rb
require 'rails_helper'

RSpec.describe 'Api::V1::Users', type: :request do
  let(:headers) { { 'Content-Type': 'application/json' } }
  let(:auth_headers) { headers.merge('Authorization' => "Bearer #{token}") }
  let(:user) { create(:user) }
  let(:token) { JwtService.encode(user_id: user.id) }
  
  describe 'GET /api/v1/users' do
    context 'เมื่อ authenticated' do
      before do
        create_list(:user, 5)
      end
      
      it 'คืน list of users' do
        get '/api/v1/users', headers: auth_headers
        
        expect(response).to have_http_status(:ok)
        json = JSON.parse(response.body)
        expect(json['data']).to be_an(Array)
        expect(json['data'].length).to eq(6) # 5 + current user
      end
      
      it 'รองรับ pagination' do
        get '/api/v1/users', params: { page: 1, per_page: 2 }, headers: auth_headers
        
        json = JSON.parse(response.body)
        expect(json['data'].length).to eq(2)
        expect(json['meta']['total']).to eq(6)
        expect(json['meta']['total_pages']).to eq(3)
      end
      
      it 'มี pagination links' do
        get '/api/v1/users', params: { page: 2, per_page: 2 }, headers: auth_headers
        
        json = JSON.parse(response.body)
        expect(json['links']).to have_key('prev')
        expect(json['links']).to have_key('next')
        expect(json['links']['prev']).to include('page=1')
        expect(json['links']['next']).to include('page=3')
      end
      
      it 'สามารถ filter ตาม role' do
        create(:user, role: 'admin')
        create(:user, role: 'guest')
        
        get '/api/v1/users', params: { role: 'admin' }, headers: auth_headers
        
        json = JSON.parse(response.body)
        expect(json['data'].all? { |u| u['attributes']['role'] == 'admin' }).to be true
      end
    end
    
    context 'เมื่อ not authenticated' do
      it 'คืน 401' do
        get '/api/v1/users'
        expect(response).to have_http_status(:unauthorized)
      end
      
      it 'response มี error message' do
        get '/api/v1/users'
        json = JSON.parse(response.body)
        expect(json['error']).to be_present
      end
    end
    
    context 'เมื่อ token หมดอายุ' do
      let(:token) { JwtService.encode(user_id: user.id, exp: 1.hour.ago.to_i) }
      
      it 'คืน 401 with token expired error' do
        get '/api/v1/users', headers: auth_headers
        
        expect(response).to have_http_status(:unauthorized)
        json = JSON.parse(response.body)
        expect(json['error']['code']).to eq('token_expired')
      end
    end
  end
  
  describe 'POST /api/v1/users' do
    let(:valid_params) do
      {
        user: {
          name: 'New User',
          email: 'newuser@example.com',
          password: 'securepassword123'
        }
      }
    end
    
    context 'ด้วย valid params' do
      it 'สร้าง user ใหม่' do
        expect {
          post '/api/v1/users', 
               params: valid_params.to_json,
               headers: auth_headers
        }.to change(User, :count).by(1)
      end
      
      it 'คืน 201 status' do
        post '/api/v1/users', params: valid_params.to_json, headers: auth_headers
        expect(response).to have_http_status(:created)
      end
      
      it 'คืน user data' do
        post '/api/v1/users', params: valid_params.to_json, headers: auth_headers
        
        json = JSON.parse(response.body)
        expect(json['data']['attributes']['name']).to eq('New User')
        expect(json['data']['attributes']['email']).to eq('newuser@example.com')
      end
      
      it 'ส่ง welcome email' do
        expect {
          post '/api/v1/users', params: valid_params.to_json, headers: auth_headers
        }.to have_enqueued_mail(UserMailer, :welcome).with(
          a_hash_including(email: 'newuser@example.com')
        )
      end
    end
    
    context 'ด้วย invalid params' do
      it 'คืน 422 เมื่อ email ซ้ำ' do
        create(:user, email: 'newuser@example.com')
        post '/api/v1/users', params: valid_params.to_json, headers: auth_headers
        
        expect(response).to have_http_status(:unprocessable_entity)
      end
      
      it 'มี validation errors' do
        invalid_params = { user: { name: '', email: 'not-an-email', password: '123' } }
        post '/api/v1/users', params: invalid_params.to_json, headers: auth_headers
        
        json = JSON.parse(response.body)
        expect(json['errors']).to be_an(Array)
        expect(json['errors'].length).to be >= 3
      end
    end
  end
  
  describe 'PATCH /api/v1/users/:id' do
    context 'owner updating own profile' do
      it 'อัพเดต name' do
        patch "/api/v1/users/#{user.id}",
              params: { user: { name: 'Updated Name' } }.to_json,
              headers: auth_headers
        
        expect(response).to have_http_status(:ok)
        expect(user.reload.name).to eq('Updated Name')
      end
    end
    
    context 'updating another user without permission' do
      let(:other_user) { create(:user) }
      
      it 'คืน 403 Forbidden' do
        patch "/api/v1/users/#{other_user.id}",
              params: { user: { name: 'Hacker Name' } }.to_json,
              headers: auth_headers
        
        expect(response).to have_http_status(:forbidden)
      end
    end
    
    context 'admin updating any user' do
      let(:admin) { create(:user, role: 'admin') }
      let(:token) { JwtService.encode(user_id: admin.id) }
      let(:other_user) { create(:user) }
      
      it 'สามารถอัพเดต user อื่นได้' do
        patch "/api/v1/users/#{other_user.id}",
              params: { user: { name: 'Admin Updated' } }.to_json,
              headers: auth_headers
        
        expect(response).to have_http_status(:ok)
      end
    end
  end
end
```

### 5.2 Shared Examples สำหรับ API Testing

```ruby
# spec/support/shared_examples/api_authentication.rb
RSpec.shared_examples 'requires authentication' do
  context 'เมื่อไม่มี token' do
    it 'คืน 401' do
      make_request
      expect(response).to have_http_status(:unauthorized)
    end
  end
  
  context 'เมื่อ token invalid' do
    let(:token) { 'invalid-token' }
    
    it 'คืน 401' do
      make_request
      expect(response).to have_http_status(:unauthorized)
    end
  end
end

RSpec.shared_examples 'requires admin role' do
  context 'เมื่อเป็น regular user' do
    let(:user) { create(:user, role: 'user') }
    
    it 'คืน 403 Forbidden' do
      make_request
      expect(response).to have_http_status(:forbidden)
    end
  end
end

RSpec.shared_examples 'paginated response' do
  it 'มี pagination meta' do
    make_request
    json = JSON.parse(response.body)
    expect(json['meta']).to include('total', 'page', 'per_page', 'total_pages')
  end
  
  it 'มี pagination links' do
    make_request
    json = JSON.parse(response.body)
    expect(json['links']).to include('self', 'first', 'last')
  end
end

# การใช้งาน shared examples
RSpec.describe 'Api::V1::Posts', type: :request do
  let(:user) { create(:user) }
  let(:token) { JwtService.encode(user_id: user.id) }
  let(:auth_headers) { { 'Authorization' => "Bearer #{token}", 'Content-Type' => 'application/json' } }
  
  describe 'GET /api/v1/posts' do
    def make_request
      get '/api/v1/posts', headers: auth_headers
    end
    
    it_behaves_like 'requires authentication'
    it_behaves_like 'paginated response'
    
    it 'คืน posts list' do
      create_list(:post, 3, user: user)
      make_request
      
      json = JSON.parse(response.body)
      expect(json['data'].length).to eq(3)
    end
  end
end
```

### 5.3 API Test Helpers

```ruby
# spec/support/api_helpers.rb
module ApiHelpers
  def json_response
    @json_response ||= JSON.parse(response.body, symbolize_names: true)
  end
  
  def json_data
    json_response[:data]
  end
  
  def json_errors
    json_response[:errors]
  end
  
  def json_meta
    json_response[:meta]
  end
  
  def create_auth_token(user = nil)
    user ||= create(:user)
    JwtService.encode(user_id: user.id)
  end
  
  def auth_headers(user = nil)
    token = create_auth_token(user)
    {
      'Authorization' => "Bearer #{token}",
      'Content-Type' => 'application/json',
      'Accept' => 'application/json'
    }
  end
  
  def expect_validation_error(field, message = nil)
    errors = json_errors
    field_errors = errors.select { |e| e.dig(:source, :pointer) == "/data/attributes/#{field}" }
    expect(field_errors).not_to be_empty
    expect(field_errors.first[:detail]).to include(message) if message
  end
end

RSpec.configure do |config|
  config.include ApiHelpers, type: :request
end
```

---

## 6. Contract Testing ด้วย Pact

Contract testing ช่วยให้มั่นใจว่า API ที่ consumer ใช้งานจะยังคงทำงานได้เมื่อ provider มีการเปลี่ยนแปลง

### 6.1 การติดตั้ง Pact

```ruby
# Gemfile
group :test do
  gem 'pact'
  gem 'pact-support'
end
```

### 6.2 Consumer-side Contract

```ruby
# spec/service_consumers/user_service_consumer_spec.rb
require 'pact/consumer/rspec'

Pact.service_consumer 'User Frontend' do
  has_pact_with 'User API' do
    mock_service :user_api do
      port 1234
    end
  end
end

RSpec.describe 'User Service Consumer', pact: true do
  subject(:client) { UserApiClient.new('http://localhost:1234') }
  
  describe 'getting users' do
    before do
      user_api
        .given('users exist')
        .upon_receiving('a request for users')
        .with(
          method: :get,
          path: '/api/v1/users',
          headers: { 'Accept' => 'application/json' }
        )
        .will_respond_with(
          status: 200,
          headers: { 'Content-Type' => 'application/json' },
          body: {
            data: Pact.each_like(
              id: Pact.like(1),
              type: 'user',
              attributes: {
                name: Pact.like('John Doe'),
                email: Pact.like('john@example.com'),
                role: Pact.term(matcher: /^(admin|user|guest)$/, generate: 'user')
              }
            ),
            meta: {
              total: Pact.like(100)
            }
          }
        )
    end
    
    it 'คืน users list' do
      result = client.get_users
      expect(result[:data]).to be_an(Array)
      expect(result[:data].first[:attributes][:name]).to be_a(String)
    end
  end
  
  describe 'creating a user' do
    before do
      user_api
        .given('admin is authenticated')
        .upon_receiving('a request to create a user')
        .with(
          method: :post,
          path: '/api/v1/users',
          headers: {
            'Content-Type' => 'application/json',
            'Authorization' => Pact.term(matcher: /^Bearer .+$/, generate: 'Bearer token123')
          },
          body: {
            user: {
              name: 'Jane Smith',
              email: 'jane@example.com',
              password: Pact.like('password123')
            }
          }
        )
        .will_respond_with(
          status: 201,
          headers: { 'Content-Type' => 'application/json' },
          body: {
            data: {
              id: Pact.like(2),
              type: 'user',
              attributes: {
                name: 'Jane Smith',
                email: 'jane@example.com'
              }
            }
          }
        )
    end
    
    it 'สร้าง user ใหม่และคืน data' do
      result = client.create_user(
        name: 'Jane Smith',
        email: 'jane@example.com',
        password: 'password123'
      )
      
      expect(result[:data][:attributes][:name]).to eq('Jane Smith')
    end
  end
end
```

### 6.3 Provider Verification

```ruby
# spec/service_providers/user_api_spec.rb
require 'rails_helper'
require 'pact/provider/rspec'

Pact.service_provider 'User API' do
  honours_pact_with 'User Frontend' do
    pact_uri './spec/pacts/user_frontend-user_api.json'
  end
end

Pact.provider_states_for 'User Frontend' do
  provider_state 'users exist' do
    set_up do
      User.delete_all
      create_list(:user, 5)
    end
    
    tear_down do
      User.delete_all
    end
  end
  
  provider_state 'admin is authenticated' do
    set_up do
      @admin = create(:user, role: 'admin')
      # Setup auth token ใน test environment
      allow_any_instance_of(ApplicationController)
        .to receive(:current_user).and_return(@admin)
    end
  end
  
  provider_state 'a user with id 1 exists' do
    set_up do
      create(:user, id: 1, name: 'John Doe', email: 'john@example.com')
    end
  end
end
```

### 6.4 Pact Broker Integration

```ruby
# Rakefile หรือ lib/tasks/pact.rake
namespace :pact do
  desc 'Publish pacts to Pact Broker'
  task :publish do
    require 'pact_broker-client'
    
    PactBroker::Client::PublishPacts.call(
      pact_broker_base_url: ENV['PACT_BROKER_URL'],
      consumer_version: ENV['GIT_COMMIT'] || `git rev-parse HEAD`.strip,
      pact_files: Dir['spec/pacts/*.json'],
      tag_with_git_branch: true,
      pact_broker_token: ENV['PACT_BROKER_TOKEN']
    )
  end
  
  desc 'Verify pacts from Pact Broker'
  task :verify do
    ENV['PACT_URL'] = "#{ENV['PACT_BROKER_URL']}/pacts/provider/User%20API/consumer/User%20Frontend/latest"
    RSpec::Core::RakeTask.new(:pact_verify) do |t|
      t.pattern = 'spec/service_providers/**/*_spec.rb'
    end
    Rake::Task['pact_verify'].invoke
  end
end
```

---

## 7. Advanced API Patterns

### 7.1 API Response Caching

```ruby
# app/controllers/api/v1/products_controller.rb
module Api
  module V1
    class ProductsController < ApplicationController
      # HTTP Caching
      def index
        @products = Product.active.order(:name)
        
        # ETag-based caching
        fresh_when(
          etag: @products.cache_key_with_version,
          last_modified: @products.maximum(:updated_at),
          public: false
        )
        
        render json: ProductSerializer.new(@products).serializable_hash
      end
      
      def show
        @product = Product.find(params[:id])
        
        fresh_when(
          etag: @product,
          last_modified: @product.updated_at,
          public: true
        )
        
        render json: ProductSerializer.new(@product).serializable_hash
      end
      
      # Fragment Caching
      def catalog
        products = Rails.cache.fetch('api/v1/products/catalog', expires_in: 15.minutes) do
          Product.includes(:category, :images).active.as_json(
            include: { category: { only: [:id, :name] } },
            methods: [:primary_image_url]
          )
        end
        
        render json: { data: products }
      end
    end
  end
end
```

### 7.2 Bulk Operations API

```ruby
# app/controllers/api/v1/bulk_controller.rb
module Api
  module V1
    class BulkController < ApplicationController
      MAX_BULK_SIZE = 100
      
      def create
        items = bulk_params[:items]
        
        if items.length > MAX_BULK_SIZE
          return render json: {
            error: "Maximum bulk size is #{MAX_BULK_SIZE}"
          }, status: :bad_request
        end
        
        results = items.map.with_index do |item, index|
          process_item(item, index)
        end
        
        successful = results.count { |r| r[:success] }
        failed = results.count { |r| !r[:success] }
        
        render json: {
          summary: {
            total: items.length,
            successful: successful,
            failed: failed
          },
          results: results
        }, status: failed > 0 ? :multi_status : :ok
      end
      
      private
      
      def process_item(item, index)
        user = User.new(item.permit(:name, :email, :role))
        
        if user.save
          { index: index, success: true, data: UserSerializer.new(user).serializable_hash }
        else
          { index: index, success: false, errors: user.errors.full_messages }
        end
      rescue => e
        { index: index, success: false, errors: [e.message] }
      end
      
      def bulk_params
        params.permit(items: [:name, :email, :role, :password])
      end
    end
  end
end
```

### 7.3 API Versioning Strategy

```ruby
# lib/api_version_router.rb
class ApiVersionRouter
  VERSIONS = {
    'v1' => '2022-01-01',
    'v2' => '2023-06-01',
    'v3' => '2024-01-01'
  }.freeze
  
  DEPRECATED_VERSIONS = %w[v1].freeze
  SUNSET_DATES = {
    'v1' => Date.new(2025, 12, 31)
  }.freeze
  
  def self.deprecation_headers_for(version)
    headers = {}
    
    if DEPRECATED_VERSIONS.include?(version)
      headers['Deprecation'] = 'true'
      headers['Sunset'] = SUNSET_DATES[version]&.httpdate
      headers['Link'] = '<https://api.example.com/v3>; rel="successor-version"'
      headers['Warning'] = "299 - \"This API version is deprecated. Please upgrade to v3\""
    end
    
    headers
  end
end

# app/controllers/api/base_controller.rb
module Api
  class BaseController < ApplicationController
    after_action :add_deprecation_headers
    
    private
    
    def add_deprecation_headers
      version = params[:api_version] || detect_version_from_path
      headers.merge!(ApiVersionRouter.deprecation_headers_for(version))
    end
    
    def detect_version_from_path
      request.path.match(%r{/api/(v\d+)/})&.captures&.first
    end
  end
end
```

---

## แบบฝึกหัด (20 ข้อพร้อมเฉลย)

### ข้อ 1: URL Path Versioning
**คำถาม:** สร้าง routes configuration สำหรับ API ที่มี v1 และ v2 โดย v2 มี endpoints เพิ่มเติม `GET /api/v2/users/:id/analytics`

**เฉลย:**
```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :users, only: [:index, :show, :create, :update, :destroy]
      resources :products, only: [:index, :show]
    end
    
    namespace :v2 do
      resources :users, only: [:index, :show, :create, :update, :destroy] do
        member do
          get :analytics
        end
      end
      resources :products
    end
  end
end
```

### ข้อ 2: Header Versioning Middleware
**คำถาม:** สร้าง middleware ที่อ่าน `Api-Version` header และ redirect ไปยัง controller ที่ถูกต้อง

**เฉลย:**
```ruby
# lib/middleware/api_version_middleware.rb
class ApiVersionMiddleware
  SUPPORTED_VERSIONS = %w[v1 v2 v3].freeze
  DEFAULT_VERSION = 'v2'
  
  def initialize(app)
    @app = app
  end
  
  def call(env)
    request = Rack::Request.new(env)
    
    if request.path.start_with?('/api')
      version = env['HTTP_API_VERSION'] || 
                env['HTTP_ACCEPT']&.match(/version=(\w+)/)&.captures&.first ||
                DEFAULT_VERSION
      
      unless SUPPORTED_VERSIONS.include?(version)
        return [
          406,
          { 'Content-Type' => 'application/json' },
          [{ error: "API version #{version} not supported", 
             supported: SUPPORTED_VERSIONS }.to_json]
        ]
      end
      
      env['api.version'] = version
      # Rewrite path
      env['PATH_INFO'] = env['PATH_INFO'].sub('/api/', "/api/#{version}/")
    end
    
    @app.call(env)
  end
end

# config/application.rb
config.middleware.insert_before Rack::Runtime, ApiVersionMiddleware
```

### ข้อ 3: HATEOAS Links
**คำถาม:** เขียน serializer สำหรับ Order ที่มี HATEOAS links รวมถึง actions ที่ทำได้ตาม status

**เฉลย:**
```ruby
# app/serializers/order_serializer.rb
class OrderSerializer
  def initialize(order, current_user:, base_url: '')
    @order = order
    @current_user = current_user
    @base_url = base_url
  end
  
  def serialize
    {
      data: {
        id: order.id,
        type: 'order',
        attributes: {
          status: order.status,
          total: order.total,
          created_at: order.created_at.iso8601
        },
        links: links,
        actions: available_actions
      }
    }
  end
  
  private
  
  attr_reader :order, :current_user, :base_url
  
  def links
    {
      self: "#{base_url}/api/v1/orders/#{order.id}",
      customer: "#{base_url}/api/v1/users/#{order.user_id}",
      items: "#{base_url}/api/v1/orders/#{order.id}/items"
    }
  end
  
  def available_actions
    actions = {}
    
    if order.pending? && current_user.owns?(order)
      actions[:cancel] = {
        href: "#{base_url}/api/v1/orders/#{order.id}/cancel",
        method: 'POST'
      }
      actions[:pay] = {
        href: "#{base_url}/api/v1/orders/#{order.id}/payment",
        method: 'POST'
      }
    end
    
    if order.shipped? 
      actions[:track] = {
        href: "#{base_url}/api/v1/orders/#{order.id}/tracking",
        method: 'GET'
      }
    end
    
    if order.delivered? && current_user.owns?(order)
      actions[:return] = {
        href: "#{base_url}/api/v1/orders/#{order.id}/return",
        method: 'POST'
      }
      actions[:review] = {
        href: "#{base_url}/api/v1/orders/#{order.id}/review",
        method: 'POST'
      }
    end
    
    actions
  end
end
```

### ข้อ 4: Rate Limiting ตาม Endpoint
**คำถาม:** ตั้งค่า Rack::Attack ให้มี rate limit ต่างกันสำหรับ read vs write operations

**เฉลย:**
```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Read operations: 1000 requests ต่อชั่วโมง
  throttle('api/read', limit: 1000, period: 1.hour) do |req|
    if req.path.start_with?('/api/') && req.get?
      req.env['HTTP_X_API_KEY'] || req.ip
    end
  end
  
  # Write operations: 100 requests ต่อชั่วโมง  
  throttle('api/write', limit: 100, period: 1.hour) do |req|
    if req.path.start_with?('/api/') && (req.post? || req.put? || req.patch? || req.delete?)
      req.env['HTTP_X_API_KEY'] || req.ip
    end
  end
  
  # Search endpoint: 50 ต่อนาที
  throttle('api/search', limit: 50, period: 1.minute) do |req|
    if req.path.include?('/search') && req.get?
      req.env['HTTP_X_API_KEY'] || req.ip
    end
  end
  
  # File upload: 10 ต่อชั่วโมง
  throttle('api/upload', limit: 10, period: 1.hour) do |req|
    if req.path.include?('/uploads') && req.post?
      req.env['HTTP_X_API_KEY'] || req.ip
    end
  end
end
```

### ข้อ 5: Swagger Schema
**คำถาม:** เขียน Swagger/OpenAPI schema สำหรับ endpoint POST /api/v1/orders ที่รับ items array

**เฉลย:**
```ruby
# spec/integration/orders_spec.rb
path '/api/v1/orders' do
  post 'Creates an order' do
    tags 'Orders'
    consumes 'application/json'
    produces 'application/json'
    
    security [{ bearerAuth: [] }]
    
    parameter name: :order, in: :body, schema: {
      type: :object,
      properties: {
        order: {
          type: :object,
          properties: {
            delivery_address: {
              type: :object,
              properties: {
                street: { type: :string },
                city: { type: :string },
                zip: { type: :string },
                country: { type: :string, default: 'TH' }
              },
              required: ['street', 'city', 'zip']
            },
            items: {
              type: :array,
              minItems: 1,
              items: {
                type: :object,
                properties: {
                  product_id: { type: :integer },
                  quantity: { type: :integer, minimum: 1 },
                  variant_id: { type: :integer, nullable: true }
                },
                required: ['product_id', 'quantity']
              }
            },
            payment_method: {
              type: :string,
              enum: ['credit_card', 'bank_transfer', 'promptpay', 'cod']
            },
            notes: { type: :string, nullable: true }
          },
          required: ['delivery_address', 'items', 'payment_method']
        }
      }
    }
    
    response '201', 'Order created' do
      schema '$ref' => '#/components/schemas/Order'
      let(:Authorization) { "Bearer #{user_token}" }
      let(:order) do
        {
          order: {
            delivery_address: { street: '123 Main St', city: 'Bangkok', zip: '10100' },
            items: [{ product_id: 1, quantity: 2 }],
            payment_method: 'promptpay'
          }
        }
      end
      run_test!
    end
  end
end
```

### ข้อ 6: Request Spec สำหรับ Nested Resources
**คำถาม:** เขียน request spec สำหรับ `GET /api/v1/users/:user_id/posts` ที่ test pagination, filtering, และ authorization

**เฉลย:**
```ruby
# spec/requests/api/v1/user_posts_spec.rb
RSpec.describe 'Api::V1::UserPosts', type: :request do
  let(:user) { create(:user) }
  let(:other_user) { create(:user) }
  let(:token) { JwtService.encode(user_id: user.id) }
  let(:headers) { { 'Authorization' => "Bearer #{token}", 'Content-Type' => 'application/json' } }
  
  before do
    create_list(:post, 10, user: user, published: true)
    create_list(:post, 3, user: user, published: false)
    create_list(:post, 5, user: other_user, published: true)
  end
  
  describe 'GET /api/v1/users/:user_id/posts' do
    context 'ดู posts ของตัวเอง' do
      it 'คืน posts ทั้งหมดรวม drafts' do
        get "/api/v1/users/#{user.id}/posts", headers: headers
        
        json = JSON.parse(response.body)
        expect(json['meta']['total']).to eq(13)
      end
      
      it 'สามารถ filter เฉพาะ published' do
        get "/api/v1/users/#{user.id}/posts",
            params: { published: true },
            headers: headers
        
        json = JSON.parse(response.body)
        expect(json['meta']['total']).to eq(10)
        expect(json['data'].all? { |p| p['attributes']['published'] == true }).to be true
      end
    end
    
    context 'ดู posts ของ user อื่น' do
      it 'คืน published posts เท่านั้น' do
        get "/api/v1/users/#{other_user.id}/posts", headers: headers
        
        json = JSON.parse(response.body)
        expect(json['meta']['total']).to eq(5)
      end
    end
    
    context 'pagination' do
      it 'แบ่งหน้าถูกต้อง' do
        get "/api/v1/users/#{user.id}/posts",
            params: { page: 2, per_page: 5 },
            headers: headers
        
        json = JSON.parse(response.body)
        expect(json['data'].length).to eq(5)
        expect(json['meta']['current_page']).to eq(2)
        expect(json['links']['prev']).to include('page=1')
        expect(json['links']['next']).to include('page=3')
      end
    end
    
    context 'ไม่มี authentication' do
      it 'คืน 401' do
        get "/api/v1/users/#{user.id}/posts"
        expect(response).to have_http_status(:unauthorized)
      end
    end
  end
end
```

### ข้อ 7: Contract Test Provider State
**คำถาม:** เขียน Pact provider states สำหรับ Order API ที่มี states: "order exists", "order is pending payment", "order is shipped"

**เฉลย:**
```ruby
# spec/service_providers/order_api_spec.rb
Pact.provider_states_for 'Order Frontend' do
  provider_state 'order exists' do
    set_up do
      @user = create(:user)
      @order = create(:order, user: @user, status: 'confirmed')
      create_list(:order_item, 3, order: @order)
    end
    
    tear_down do
      OrderItem.delete_all
      Order.delete_all
      User.delete_all
    end
  end
  
  provider_state 'order is pending payment' do
    set_up do
      @user = create(:user)
      @order = create(:order, 
                      user: @user, 
                      status: 'pending_payment',
                      total: 1500.00)
      allow_any_instance_of(ApplicationController)
        .to receive(:current_user).and_return(@user)
    end
    
    tear_down do
      Order.delete_all
      User.delete_all
    end
  end
  
  provider_state 'order is shipped' do
    set_up do
      @user = create(:user)
      @order = create(:order, 
                      user: @user,
                      status: 'shipped',
                      tracking_number: 'TH123456789',
                      shipped_at: 2.days.ago)
      allow_any_instance_of(ApplicationController)
        .to receive(:current_user).and_return(@user)
    end
    
    tear_down do
      Order.delete_all
      User.delete_all
    end
  end
  
  provider_state 'no orders exist' do
    set_up do
      Order.delete_all
    end
  end
end
```

### ข้อ 8: Custom Error Handler
**คำถาม:** สร้าง centralized error handler สำหรับ API ที่จัดการ errors ต่างๆ ให้อยู่ในรูปแบบมาตรฐาน

**เฉลย:**
```ruby
# app/controllers/concerns/error_handler.rb
module ErrorHandler
  extend ActiveSupport::Concern
  
  included do
    rescue_from StandardError, with: :handle_internal_server_error
    rescue_from ActiveRecord::RecordNotFound, with: :handle_not_found
    rescue_from ActiveRecord::RecordInvalid, with: :handle_unprocessable_entity
    rescue_from ActionController::ParameterMissing, with: :handle_bad_request
    rescue_from Pundit::NotAuthorizedError, with: :handle_forbidden
    rescue_from JWT::ExpiredSignature, with: :handle_token_expired
    rescue_from JWT::DecodeError, with: :handle_unauthorized
  end
  
  private
  
  def handle_not_found(exception)
    render_error(
      status: :not_found,
      code: 'resource_not_found',
      message: exception.message,
      detail: "The requested #{exception.model} could not be found"
    )
  end
  
  def handle_unprocessable_entity(exception)
    render json: {
      errors: format_record_errors(exception.record.errors)
    }, status: :unprocessable_entity
  end
  
  def handle_bad_request(exception)
    render_error(
      status: :bad_request,
      code: 'bad_request',
      message: exception.message
    )
  end
  
  def handle_forbidden(exception)
    render_error(
      status: :forbidden,
      code: 'forbidden',
      message: 'You are not authorized to perform this action'
    )
  end
  
  def handle_token_expired(exception)
    render_error(
      status: :unauthorized,
      code: 'token_expired',
      message: 'Your session has expired. Please login again.'
    )
  end
  
  def handle_unauthorized(exception)
    render_error(
      status: :unauthorized,
      code: 'unauthorized',
      message: 'Authentication required'
    )
  end
  
  def handle_internal_server_error(exception)
    Rails.logger.error("#{exception.class}: #{exception.message}\n#{exception.backtrace.first(10).join("\n")}")
    
    render_error(
      status: :internal_server_error,
      code: 'internal_server_error',
      message: 'An unexpected error occurred'
    )
  end
  
  def render_error(status:, code:, message:, detail: nil)
    error = { code: code, message: message }
    error[:detail] = detail if detail
    
    render json: { error: error }, status: status
  end
  
  def format_record_errors(errors)
    errors.map do |error|
      {
        source: { pointer: "/data/attributes/#{error.attribute}" },
        title: 'Validation Error',
        detail: error.full_message
      }
    end
  end
end
```

### ข้อ 9: API Key Authentication
**คำถาม:** สร้างระบบ API Key authentication ที่รองรับ key rotation และ scope-based permissions

**เฉลย:**
```ruby
# app/models/api_key.rb
class ApiKey < ApplicationRecord
  belongs_to :user
  
  SCOPES = %w[read write admin].freeze
  
  before_create :generate_key
  
  validates :name, presence: true
  validates :scopes, presence: true
  
  scope :active, -> { where(revoked_at: nil) }
  scope :with_scope, ->(scope) { where('scopes @> ARRAY[?]::varchar[]', scope) }
  
  def revoke!
    update!(revoked_at: Time.current)
  end
  
  def revoked?
    revoked_at.present?
  end
  
  def has_scope?(scope)
    scopes.include?(scope.to_s)
  end
  
  def rotate!
    transaction do
      old_key = key
      generate_key
      save!
      ApiKeyRotationLog.create!(api_key: self, old_key_prefix: old_key[0..7])
    end
    key
  end
  
  private
  
  def generate_key
    self.key = "#{prefix}_#{SecureRandom.hex(32)}"
  end
  
  def prefix
    case scopes
    when ['read'] then 'ro'
    when ['write'] then 'wo'
    else 'sk'
    end
  end
end

# app/controllers/concerns/api_key_authenticatable.rb
module ApiKeyAuthenticatable
  extend ActiveSupport::Concern
  
  included do
    before_action :authenticate_with_api_key
  end
  
  private
  
  def authenticate_with_api_key
    key = extract_api_key
    return render_unauthorized unless key
    
    @api_key = ApiKey.active.find_by(key: key)
    
    unless @api_key
      render json: { error: { code: 'invalid_api_key', message: 'Invalid API key' } }, 
             status: :unauthorized
      return
    end
    
    @current_user = @api_key.user
    @api_key.update_columns(last_used_at: Time.current, usage_count: @api_key.usage_count + 1)
  end
  
  def require_scope(scope)
    unless @api_key&.has_scope?(scope)
      render json: {
        error: {
          code: 'insufficient_scope',
          message: "This action requires '#{scope}' scope"
        }
      }, status: :forbidden
    end
  end
  
  def extract_api_key
    request.headers['X-Api-Key'] ||
      request.headers['Authorization']&.sub(/^ApiKey /, '')
  end
  
  def render_unauthorized
    render json: { error: { code: 'missing_api_key', message: 'API key required' } },
           status: :unauthorized
  end
end
```

### ข้อ 10: Pagination Helper
**คำถาม:** สร้าง reusable pagination module สำหรับ API controllers ที่รองรับ cursor-based pagination

**เฉลย:**
```ruby
# app/controllers/concerns/cursor_pagination.rb
module CursorPagination
  extend ActiveSupport::Concern
  
  DEFAULT_LIMIT = 25
  MAX_LIMIT = 100
  
  def paginate_with_cursor(scope, default_order: :id)
    limit = [params[:limit].to_i.presence || DEFAULT_LIMIT, MAX_LIMIT].min
    cursor = decode_cursor(params[:cursor])
    direction = params[:direction] == 'before' ? :before : :after
    
    if cursor && direction == :after
      scope = scope.where("#{default_order} > ?", cursor)
    elsif cursor && direction == :before
      scope = scope.where("#{default_order} < ?", cursor).order("#{default_order} DESC")
    end
    
    records = scope.order(default_order => :asc).limit(limit + 1).to_a
    
    has_more = records.length > limit
    records = records.first(limit)
    records.reverse! if direction == :before
    
    {
      data: records,
      pagination: {
        has_more: has_more,
        next_cursor: has_more ? encode_cursor(records.last.send(default_order)) : nil,
        prev_cursor: cursor ? encode_cursor(records.first.send(default_order)) : nil,
        limit: limit
      }
    }
  end
  
  private
  
  def encode_cursor(value)
    Base64.urlsafe_encode64(value.to_s)
  end
  
  def decode_cursor(cursor)
    return nil unless cursor
    Base64.urlsafe_decode64(cursor)
  rescue ArgumentError
    nil
  end
end
```

### ข้อ 11: Response Transformer
**คำถาม:** สร้าง response transformer ที่แปลง snake_case เป็น camelCase สำหรับ JSON API responses

**เฉลย:**
```ruby
# app/middleware/camel_case_response_middleware.rb
class CamelCaseResponseMiddleware
  API_PATH_PATTERN = %r{^/api/}
  
  def initialize(app)
    @app = app
  end
  
  def call(env)
    status, headers, body = @app.call(env)
    
    request = Rack::Request.new(env)
    
    if should_transform?(request, headers)
      body_string = body.map { |b| b }.join
      
      begin
        json = JSON.parse(body_string)
        transformed = deep_camelize(json)
        new_body = transformed.to_json
        
        headers['Content-Length'] = new_body.bytesize.to_s
        body = [new_body]
      rescue JSON::ParserError
        # Not JSON, return as-is
      end
    end
    
    [status, headers, body]
  end
  
  private
  
  def should_transform?(request, headers)
    request.path.match?(API_PATH_PATTERN) &&
      headers['Content-Type']&.include?('application/json') &&
      request.env['HTTP_X_RESPONSE_CASE'] != 'snake_case'
  end
  
  def deep_camelize(obj)
    case obj
    when Hash
      obj.transform_keys { |k| camelize(k.to_s) }
         .transform_values { |v| deep_camelize(v) }
    when Array
      obj.map { |v| deep_camelize(v) }
    else
      obj
    end
  end
  
  def camelize(string)
    # "created_at" => "createdAt"
    string.gsub(/_([a-z])/) { $1.upcase }
  end
end

# config/application.rb
config.middleware.use CamelCaseResponseMiddleware
```

### ข้อ 12: API Versioning with Semantic Versioning
**คำถาม:** สร้างระบบ versioning ที่รองรับ semantic versioning (1.0.0, 2.1.3) ผ่าน Accept header

**เฉลย:**
```ruby
# lib/api_version_negotiator.rb
class ApiVersionNegotiator
  VERSION_PATTERN = /application\/vnd\.myapp\.v(\d+)(?:\.(\d+))?(?:\.(\d+))?\+json/
  
  AVAILABLE_VERSIONS = [
    Gem::Version.new('3.0.0'),
    Gem::Version.new('2.5.0'),
    Gem::Version.new('2.0.0'),
    Gem::Version.new('1.2.0'),
    Gem::Version.new('1.0.0')
  ].sort.reverse.freeze
  
  def self.negotiate(accept_header)
    return AVAILABLE_VERSIONS.first unless accept_header
    
    requested = parse_version(accept_header)
    return AVAILABLE_VERSIONS.first unless requested
    
    # หา version ที่ compatible (major version ตรงกัน, minor/patch >= ที่ขอ)
    compatible = AVAILABLE_VERSIONS.select do |v|
      v.segments[0] == requested.segments[0] && v >= requested
    end
    
    compatible.first || AVAILABLE_VERSIONS.first
  end
  
  def self.parse_version(header)
    match = header.match(VERSION_PATTERN)
    return nil unless match
    
    major = match[1].to_i
    minor = match[2].to_i
    patch = match[3].to_i
    
    Gem::Version.new("#{major}.#{minor}.#{patch}")
  end
end

# app/controllers/application_controller.rb
before_action :negotiate_api_version

def negotiate_api_version
  @negotiated_version = ApiVersionNegotiator.negotiate(
    request.headers['Accept']
  )
  
  response.headers['Content-Type'] = 
    "application/vnd.myapp.v#{@negotiated_version}+json"
end
```

### ข้อ 13: Webhook Testing
**คำถาม:** เขียน request spec สำหรับ webhook endpoint ที่ verify signature และ process events

**เฉลย:**
```ruby
# spec/requests/api/v1/webhooks_spec.rb
RSpec.describe 'Api::V1::Webhooks', type: :request do
  let(:secret) { 'webhook_secret_key' }
  let(:payload) { { event: 'payment.completed', order_id: 123, amount: 1500 } }
  let(:payload_json) { payload.to_json }
  let(:signature) do
    "sha256=#{OpenSSL::HMAC.hexdigest('SHA256', secret, payload_json)}"
  end
  
  before do
    allow(Rails.application.credentials).to receive(:webhook_secret).and_return(secret)
  end
  
  describe 'POST /api/v1/webhooks' do
    context 'ด้วย valid signature' do
      it 'process event และคืน 200' do
        expect {
          post '/api/v1/webhooks',
               params: payload_json,
               headers: {
                 'Content-Type' => 'application/json',
                 'X-Webhook-Signature' => signature
               }
        }.to have_enqueued_job(ProcessWebhookJob).with(
          hash_including('event' => 'payment.completed')
        )
        
        expect(response).to have_http_status(:ok)
      end
    end
    
    context 'ด้วย invalid signature' do
      it 'คืน 401' do
        post '/api/v1/webhooks',
             params: payload_json,
             headers: {
               'Content-Type' => 'application/json',
               'X-Webhook-Signature' => 'sha256=invalidsignature'
             }
        
        expect(response).to have_http_status(:unauthorized)
        expect(JSON.parse(response.body)['error']).to eq('Invalid webhook signature')
      end
    end
    
    context 'ไม่มี signature' do
      it 'คืน 401' do
        post '/api/v1/webhooks',
             params: payload_json,
             headers: { 'Content-Type' => 'application/json' }
        
        expect(response).to have_http_status(:unauthorized)
      end
    end
  end
end
```

### ข้อ 14: API Deprecation Middleware
**คำถาม:** สร้าง middleware ที่เพิ่ม deprecation warning headers สำหรับ deprecated endpoints

**เฉลย:**
```ruby
# lib/middleware/api_deprecation_middleware.rb
class ApiDeprecationMiddleware
  DEPRECATED_ENDPOINTS = {
    '/api/v1/users/search' => {
      sunset_date: Date.new(2025, 6, 30),
      successor: '/api/v2/users?query=',
      message: 'Use /api/v2/users?query= instead'
    },
    %r{/api/v1/products/\d+/review} => {
      sunset_date: Date.new(2025, 3, 31),
      successor: '/api/v2/reviews',
      message: 'Use /api/v2/reviews with product_id parameter'
    }
  }.freeze
  
  def initialize(app)
    @app = app
  end
  
  def call(env)
    status, headers, body = @app.call(env)
    
    path = env['PATH_INFO']
    deprecation_info = find_deprecation(path)
    
    if deprecation_info
      headers['Deprecation'] = deprecation_info[:sunset_date].httpdate
      headers['Sunset'] = deprecation_info[:sunset_date].httpdate
      headers['Link'] = build_link_header(deprecation_info)
      headers['Warning'] = "299 - \"#{deprecation_info[:message]}\""
      
      # Log deprecated API usage
      Rails.logger.warn("Deprecated API called: #{path} from #{env['REMOTE_ADDR']}")
    end
    
    [status, headers, body]
  end
  
  private
  
  def find_deprecation(path)
    DEPRECATED_ENDPOINTS.each do |pattern, info|
      if pattern.is_a?(Regexp)
        return info if path.match?(pattern)
      else
        return info if path == pattern
      end
    end
    nil
  end
  
  def build_link_header(info)
    "<#{info[:successor]}>; rel=\"successor-version\""
  end
end
```

### ข้อ 15: Idempotency Keys
**คำถาม:** implement idempotency key สำหรับ POST requests เพื่อป้องกัน duplicate submissions

**เฉลย:**
```ruby
# app/controllers/concerns/idempotency.rb
module Idempotency
  extend ActiveSupport::Concern
  
  included do
    before_action :check_idempotency_key, only: [:create]
    after_action :store_idempotency_response, only: [:create]
  end
  
  private
  
  IDEMPOTENCY_TTL = 24.hours
  
  def check_idempotency_key
    key = request.headers['Idempotency-Key']
    return unless key
    
    cached = Rails.cache.read(idempotency_cache_key(key))
    
    if cached
      # ส่ง cached response กลับไป
      cached_response = JSON.parse(cached)
      response.headers['X-Idempotent-Replayed'] = 'true'
      render json: cached_response['body'],
             status: cached_response['status']
    end
  end
  
  def store_idempotency_response
    key = request.headers['Idempotency-Key']
    return unless key
    return if response.headers['X-Idempotent-Replayed']
    
    # Successful responses เท่านั้น
    if response.status.to_i.between?(200, 299)
      Rails.cache.write(
        idempotency_cache_key(key),
        { status: response.status, body: response.body }.to_json,
        expires_in: IDEMPOTENCY_TTL
      )
    end
  end
  
  def idempotency_cache_key(key)
    user_id = current_user&.id || request.ip
    "idempotency:#{user_id}:#{request.path}:#{key}"
  end
end
```

### ข้อ 16: API Analytics Middleware
**คำถาม:** สร้าง middleware ที่ track API usage analytics (endpoint, status, duration)

**เฉลย:**
```ruby
# lib/middleware/api_analytics_middleware.rb
class ApiAnalyticsMiddleware
  def initialize(app)
    @app = app
  end
  
  def call(env)
    return @app.call(env) unless api_request?(env)
    
    start_time = Time.current
    status, headers, body = @app.call(env)
    duration = ((Time.current - start_time) * 1000).round(2)
    
    track_request(env, status, duration)
    
    [status, headers, body]
  end
  
  private
  
  def api_request?(env)
    env['PATH_INFO'].start_with?('/api/')
  end
  
  def track_request(env, status, duration)
    request = Rack::Request.new(env)
    
    data = {
      endpoint: "#{request.request_method} #{normalize_path(request.path)}",
      status: status,
      duration_ms: duration,
      ip: request.ip,
      user_agent: request.user_agent,
      api_key: env['HTTP_X_API_KEY'],
      user_id: env['current_user_id'],
      timestamp: Time.current.iso8601
    }
    
    # Async tracking
    ApiAnalyticsJob.perform_later(data) if defined?(ApiAnalyticsJob)
    
    # Real-time metrics
    StatsD.increment("api.requests.#{request.request_method.downcase}")
    StatsD.increment("api.responses.#{status}")
    StatsD.timing("api.duration", duration)
    
  rescue => e
    Rails.logger.error("Analytics tracking failed: #{e.message}")
  end
  
  def normalize_path(path)
    # Replace IDs with :id placeholder
    path.gsub(%r{/\d+}, '/:id')
        .gsub(%r{/[0-9a-f-]{36}}, '/:uuid')
  end
end
```

### ข้อ 17: Conditional Fields
**คำถาม:** implement sparse fieldsets (ตาม JSON:API spec) ที่ให้ client เลือก fields ที่ต้องการ

**เฉลย:**
```ruby
# app/serializers/concerns/sparse_fieldsets.rb
module SparseFieldsets
  def initialize(resource, options = {})
    @requested_fields = parse_fields(options[:fields])
    super
  end
  
  def serializable_hash(options = {})
    hash = super
    
    return hash unless @requested_fields.any?
    
    filter_fields(hash)
  end
  
  private
  
  def parse_fields(fields_param)
    return {} unless fields_param.is_a?(Hash)
    
    fields_param.transform_values do |fields|
      case fields
      when String then fields.split(',').map(&:strip).map(&:to_sym)
      when Array then fields.map(&:to_sym)
      else []
      end
    end
  end
  
  def filter_fields(hash)
    return hash unless hash.is_a?(Hash)
    
    data = hash[:data] || hash['data']
    return hash unless data
    
    if data.is_a?(Array)
      hash.merge(data: data.map { |item| filter_item_fields(item) })
    else
      hash.merge(data: filter_item_fields(data))
    end
  end
  
  def filter_item_fields(item)
    return item unless item.is_a?(Hash)
    
    type = (item[:type] || item['type'])&.to_sym
    return item unless type
    
    allowed_fields = @requested_fields[type]
    return item unless allowed_fields&.any?
    
    attributes = item[:attributes] || item['attributes'] || {}
    filtered_attrs = attributes.select { |k, _| allowed_fields.include?(k.to_sym) }
    
    item.merge(attributes: filtered_attrs)
  end
end

# การใช้งาน
class UserSerializer
  include JSONAPI::Serializer
  include SparseFieldsets
  
  attributes :name, :email, :role, :bio, :avatar_url, :created_at
end

# Controller
def index
  users = User.all
  fields = params[:fields]
  
  render json: UserSerializer.new(
    users,
    fields: fields  # ?fields[user]=name,email
  ).serializable_hash
end
```

### ข้อ 18: Streaming API Response
**คำถาม:** implement streaming response สำหรับ export endpoint ที่ส่งข้อมูล CSV ขนาดใหญ่

**เฉลย:**
```ruby
# app/controllers/api/v1/exports_controller.rb
module Api
  module V1
    class ExportsController < ApplicationController
      def users_csv
        response.headers['Content-Type'] = 'text/csv'
        response.headers['Content-Disposition'] = 
          'attachment; filename="users_export.csv"'
        response.headers['X-Accel-Buffering'] = 'no'
        
        self.response_body = Enumerator.new do |yielder|
          # CSV header
          yielder << CSV.generate_line(['ID', 'Name', 'Email', 'Role', 'Created At'])
          
          # Stream in batches
          User.in_batches(of: 1000) do |batch|
            batch.each do |user|
              yielder << CSV.generate_line([
                user.id,
                user.name,
                user.email,
                user.role,
                user.created_at.iso8601
              ])
            end
          end
        end
      end
      
      def users_json_stream
        response.headers['Content-Type'] = 'application/x-ndjson'
        response.headers['X-Accel-Buffering'] = 'no'
        
        self.response_body = Enumerator.new do |yielder|
          User.in_batches(of: 500) do |batch|
            batch.each do |user|
              yielder << user.to_json + "\n"
            end
          end
          
          # Final newline
          yielder << ""
        end
      end
    end
  end
end
```

### ข้อ 19: API Compatibility Testing
**คำถาม:** เขียน test ที่ verify backward compatibility ระหว่าง API versions

**เฉลย:**
```ruby
# spec/support/api_compatibility_checker.rb
module ApiCompatibilityChecker
  def self.check_backward_compatibility(v1_response, v2_response)
    v1_keys = extract_keys(v1_response)
    v2_keys = extract_keys(v2_response)
    
    missing_in_v2 = v1_keys - v2_keys
    
    {
      compatible: missing_in_v2.empty?,
      missing_fields: missing_in_v2,
      new_fields: v2_keys - v1_keys
    }
  end
  
  def self.extract_keys(response, prefix = '')
    keys = []
    
    case response
    when Hash
      response.each do |k, v|
        full_key = prefix.empty? ? k.to_s : "#{prefix}.#{k}"
        keys << full_key
        keys.concat(extract_keys(v, full_key))
      end
    when Array
      response.first(1).each do |item|
        keys.concat(extract_keys(item, "#{prefix}[]"))
      end
    end
    
    keys
  end
end

# spec/requests/api_compatibility_spec.rb
RSpec.describe 'API Backward Compatibility', type: :request do
  let(:user) { create(:user) }
  let(:v1_headers) { { 'Accept-Version' => 'v1', 'Authorization' => "Bearer #{token}" } }
  let(:v2_headers) { { 'Accept-Version' => 'v2', 'Authorization' => "Bearer #{token}" } }
  let(:token) { JwtService.encode(user_id: user.id) }
  
  describe 'Users endpoint' do
    before { create_list(:user, 3) }
    
    it 'v2 เป็น backward compatible กับ v1' do
      get '/api/users', headers: v1_headers
      v1_response = JSON.parse(response.body)
      
      get '/api/users', headers: v2_headers
      v2_response = JSON.parse(response.body)
      
      result = ApiCompatibilityChecker.check_backward_compatibility(
        v1_response.dig('data', 0),
        v2_response.dig('data', 0)
      )
      
      expect(result[:compatible]).to be true,
        "v2 is missing fields from v1: #{result[:missing_fields].join(', ')}"
    end
  end
end
```

### ข้อ 20: Full API Integration Test
**คำถาม:** เขียน integration test ที่ test workflow สมบูรณ์ตั้งแต่ register, login, สร้าง resource, และ cleanup

**เฉลย:**
```ruby
# spec/requests/api/full_workflow_spec.rb
RSpec.describe 'Full API Workflow', type: :request do
  let(:headers) { { 'Content-Type' => 'application/json' } }
  
  it 'completes full user lifecycle' do
    # Step 1: Register
    post '/api/v1/auth/register',
         params: {
           user: {
             name: 'Test User',
             email: 'workflow_test@example.com',
             password: 'password123'
           }
         }.to_json,
         headers: headers
    
    expect(response).to have_http_status(:created)
    user_data = JSON.parse(response.body)
    expect(user_data['data']['attributes']['email']).to eq('workflow_test@example.com')
    
    # Step 2: Login
    post '/api/v1/auth/login',
         params: {
           email: 'workflow_test@example.com',
           password: 'password123'
         }.to_json,
         headers: headers
    
    expect(response).to have_http_status(:ok)
    auth_data = JSON.parse(response.body)
    token = auth_data['data']['token']
    expect(token).to be_present
    
    auth_headers = headers.merge('Authorization' => "Bearer #{token}")
    
    # Step 3: Create a post
    post '/api/v1/posts',
         params: {
           post: {
             title: 'My First Post',
             content: 'Hello World!',
             published: true
           }
         }.to_json,
         headers: auth_headers
    
    expect(response).to have_http_status(:created)
    post_data = JSON.parse(response.body)
    post_id = post_data['data']['id']
    
    # Step 4: Get the post
    get "/api/v1/posts/#{post_id}", headers: auth_headers
    
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)['data']['attributes']['title']).to eq('My First Post')
    
    # Step 5: Update
    patch "/api/v1/posts/#{post_id}",
          params: { post: { title: 'Updated Title' } }.to_json,
          headers: auth_headers
    
    expect(response).to have_http_status(:ok)
    
    # Step 6: Delete
    delete "/api/v1/posts/#{post_id}", headers: auth_headers
    expect(response).to have_http_status(:no_content)
    
    # Verify deleted
    get "/api/v1/posts/#{post_id}", headers: auth_headers
    expect(response).to have_http_status(:not_found)
    
    # Step 7: Logout
    delete '/api/v1/auth/logout', headers: auth_headers
    expect(response).to have_http_status(:no_content)
    
    # Verify token invalid after logout
    get '/api/v1/users/me', headers: auth_headers
    expect(response).to have_http_status(:unauthorized)
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **API Versioning** - 3 วิธีหลัก: URL path, Header, Query param พร้อมข้อดีข้อเสีย
2. **HATEOAS** - การเพิ่ม links และ actions ใน API response ตาม JSON:API standard
3. **Rate Limiting** - Rack::Attack สำหรับป้องกัน API abuse พร้อม dynamic limits
4. **API Documentation** - rswag/Swagger สำหรับ generate OpenAPI documentation
5. **Request Specs** - RSpec request specs พร้อม shared examples และ helpers
6. **Contract Testing** - Pact สำหรับ consumer-driven contract testing

แนวทางปฏิบัติที่ดีที่สุด:
- เลือก versioning strategy ที่เหมาะกับ use case
- ใช้ HATEOAS เพื่อให้ API self-documenting
- ตั้ง rate limits ตั้งแต่ต้น อย่ารอจนถูก abuse
- เขียน documentation ควบคู่กับโค้ด (rswag)
- Test ทั้ง happy path และ edge cases
- Contract testing สำคัญมากเมื่อมี multiple clients
