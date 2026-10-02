# Part 68: API Authentication

## Steps 1476-1495

---

## Step 1476: JWT (JSON Web Tokens) จาก Scratch

JWT คือ standard สำหรับส่ง claims ระหว่าง parties อย่างปลอดภัย ประกอบด้วย 3 ส่วน:

```
Header.Payload.Signature
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxLCJleHAiOjE2OTkwMDAwMDB9.abc123
```

### โครงสร้าง JWT

```
Header:  { "alg": "HS256", "typ": "JWT" }
Payload: { "user_id": 1, "exp": 1699000000, "iat": 1696000000 }
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

### สร้าง JWT จาก Scratch (ไม่ใช้ gem)

```ruby
# app/lib/jwt_helper.rb
require 'openssl'
require 'base64'
require 'json'

module JwtHelper
  SECRET_KEY = ENV.fetch('JWT_SECRET_KEY') { 
    raise "JWT_SECRET_KEY not set!" 
  }
  
  def self.encode(payload, expiry: 24.hours.from_now)
    payload = payload.merge(
      exp: expiry.to_i,
      iat: Time.current.to_i,
      jti: SecureRandom.uuid  # JWT ID สำหรับ revocation
    )
    
    header = Base64.urlsafe_encode64(
      { alg: 'HS256', typ: 'JWT' }.to_json,
      padding: false
    )
    
    body = Base64.urlsafe_encode64(payload.to_json, padding: false)
    
    data = "#{header}.#{body}"
    signature = Base64.urlsafe_encode64(
      OpenSSL::HMAC.digest('SHA256', SECRET_KEY, data),
      padding: false
    )
    
    "#{data}.#{signature}"
  end
  
  def self.decode(token)
    parts = token.split('.')
    raise 'Invalid token format' unless parts.length == 3
    
    header, body, signature = parts
    
    # Verify signature
    expected_sig = Base64.urlsafe_encode64(
      OpenSSL::HMAC.digest('SHA256', SECRET_KEY, "#{header}.#{body}"),
      padding: false
    )
    
    unless ActiveSupport::SecurityUtils.secure_compare(signature, expected_sig)
      raise 'Invalid signature'
    end
    
    # Decode payload
    payload = JSON.parse(Base64.urlsafe_decode64(body), symbolize_names: true)
    
    # Check expiry
    if payload[:exp] && payload[:exp] < Time.current.to_i
      raise 'Token has expired'
    end
    
    payload
  rescue JSON::ParserError
    raise 'Invalid token'
  end
end
```

---

## Step 1477: jwt Gem Implementation

```ruby
# Gemfile
gem 'jwt'

# config/initializers/jwt.rb
JWT_SECRET = ENV.fetch('JWT_SECRET_KEY') { 
  Rails.env.development? ? 'dev-secret-key-change-in-production' : raise('JWT_SECRET_KEY not set!')
}

# app/lib/json_web_token.rb
class JsonWebToken
  SECRET_KEY = JWT_SECRET
  ALGORITHM = 'HS256'.freeze
  
  def self.encode(payload, expiry: 24.hours)
    payload = payload.with_indifferent_access
    payload[:exp] = expiry.from_now.to_i
    payload[:iat] = Time.current.to_i
    payload[:jti] = SecureRandom.uuid
    
    JWT.encode(payload, SECRET_KEY, ALGORITHM)
  end
  
  def self.decode(token)
    decoded = JWT.decode(
      token,
      SECRET_KEY,
      true,  # verify signature
      {
        algorithm: ALGORITHM,
        verify_expiration: true,
        verify_iat: true
      }
    )
    
    HashWithIndifferentAccess.new(decoded.first)
  rescue JWT::ExpiredSignature
    raise AuthenticationError, 'Token หมดอายุแล้ว'
  rescue JWT::DecodeError => e
    raise AuthenticationError, "Token ไม่ถูกต้อง: #{e.message}"
  end
  
  def self.valid?(token)
    decode(token)
    true
  rescue
    false
  end
end

# Custom Errors
class AuthenticationError < StandardError; end
class AuthorizationError < StandardError; end
```

### Authentication Controller

```ruby
# app/controllers/api/v1/auth_controller.rb
class Api::V1::AuthController < ApplicationController
  skip_before_action :authenticate_user!, only: [:login, :register]
  
  # POST /api/v1/auth/login
  def login
    user = User.find_by(email: params[:email].downcase)
    
    if user&.authenticate(params[:password])
      if user.active?
        access_token = generate_access_token(user)
        refresh_token = generate_refresh_token(user)
        
        render json: {
          access_token: access_token,
          refresh_token: refresh_token,
          token_type: 'Bearer',
          expires_in: 24.hours.to_i,
          user: UserSerializer.new(user).as_json
        }, status: :ok
      else
        render json: { error: 'บัญชีถูกระงับ' }, status: :forbidden
      end
    else
      # Generic error เพื่อป้องกัน user enumeration
      render json: { error: 'อีเมลหรือรหัสผ่านไม่ถูกต้อง' }, status: :unauthorized
    end
  end
  
  # POST /api/v1/auth/register
  def register
    result = UserRegistration.call(
      name: params[:name],
      email: params[:email],
      password: params[:password],
      password_confirmation: params[:password_confirmation]
    )
    
    if result.success?
      user = result.user
      access_token = generate_access_token(user)
      refresh_token = generate_refresh_token(user)
      
      render json: {
        access_token: access_token,
        refresh_token: refresh_token,
        user: UserSerializer.new(user).as_json,
        message: 'สมัครสมาชิกสำเร็จ'
      }, status: :created
    else
      render json: { errors: result.errors }, status: :unprocessable_entity
    end
  end
  
  # POST /api/v1/auth/logout
  def logout
    # เพิ่ม token ลงใน blacklist
    token = extract_token_from_header
    TokenBlacklist.add(token) if token.present?
    
    render json: { message: 'ออกจากระบบสำเร็จ' }, status: :ok
  end
  
  # POST /api/v1/auth/refresh
  def refresh
    refresh_token = params[:refresh_token]
    
    begin
      payload = JsonWebToken.decode(refresh_token)
      
      unless payload[:type] == 'refresh'
        return render json: { error: 'Token ไม่ถูกต้อง' }, status: :unauthorized
      end
      
      user = User.find(payload[:user_id])
      
      # Rotate refresh token (สร้างใหม่)
      new_access_token = generate_access_token(user)
      new_refresh_token = generate_refresh_token(user)
      
      # Invalidate old refresh token
      TokenBlacklist.add(refresh_token)
      
      render json: {
        access_token: new_access_token,
        refresh_token: new_refresh_token,
        token_type: 'Bearer',
        expires_in: 24.hours.to_i
      }, status: :ok
      
    rescue AuthenticationError => e
      render json: { error: e.message }, status: :unauthorized
    end
  end
  
  private
  
  def generate_access_token(user)
    JsonWebToken.encode(
      {
        user_id: user.id,
        email: user.email,
        role: user.role,
        type: 'access'
      },
      expiry: 24.hours
    )
  end
  
  def generate_refresh_token(user)
    JsonWebToken.encode(
      {
        user_id: user.id,
        type: 'refresh'
      },
      expiry: 30.days
    )
  end
end
```

### ApplicationController สำหรับ JWT Authentication

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  before_action :authenticate_user!
  
  attr_reader :current_user
  
  rescue_from AuthenticationError, with: :handle_auth_error
  rescue_from AuthorizationError, with: :handle_authz_error
  rescue_from ActiveRecord::RecordNotFound, with: :handle_not_found
  
  private
  
  def authenticate_user!
    token = extract_token_from_header
    
    raise AuthenticationError, 'Token จำเป็นต้องใส่' unless token
    raise AuthenticationError, 'Token ถูก revoke แล้ว' if TokenBlacklist.include?(token)
    
    payload = JsonWebToken.decode(token)
    
    raise AuthenticationError, 'Token ประเภทไม่ถูกต้อง' unless payload[:type] == 'access'
    
    @current_user = User.find(payload[:user_id])
    raise AuthenticationError, 'User ไม่พบ' unless @current_user
    raise AuthenticationError, 'บัญชีถูกระงับ' unless @current_user.active?
    
  rescue ActiveRecord::RecordNotFound
    raise AuthenticationError, 'User ไม่พบ'
  end
  
  def extract_token_from_header
    auth_header = request.headers['Authorization']
    return nil unless auth_header&.start_with?('Bearer ')
    auth_header.split(' ').last
  end
  
  def handle_auth_error(error)
    render json: { error: error.message }, status: :unauthorized
  end
  
  def handle_authz_error(error)
    render json: { error: error.message }, status: :forbidden
  end
  
  def handle_not_found(error)
    render json: { error: 'ไม่พบข้อมูล' }, status: :not_found
  end
  
  def require_admin!
    raise AuthorizationError, 'ต้องมีสิทธิ์ Admin' unless current_user.admin?
  end
end
```

---

## Step 1478: Refresh Tokens

Refresh Token ให้ผู้ใช้อยู่ใน session นานขึ้นโดยไม่ต้อง login ใหม่

### Token Blacklist ด้วย Redis

```ruby
# app/services/token_blacklist.rb
class TokenBlacklist
  PREFIX = 'blacklisted_token:'.freeze
  
  def self.add(token)
    payload = JsonWebToken.decode(token)
    
    # หมดอายุ blacklist พร้อมกับ token
    ttl = payload[:exp] - Time.current.to_i
    return if ttl <= 0
    
    Redis.current.setex("#{PREFIX}#{token}", ttl, '1')
  rescue AuthenticationError
    # ถ้า token invalid อยู่แล้ว ไม่ต้อง blacklist
  end
  
  def self.include?(token)
    Redis.current.exists?("#{PREFIX}#{token}")
  end
  
  def self.clear_expired
    # Redis จัดการ expiry อัตโนมัติ
    # Method นี้ใช้สำหรับ manual cleanup ถ้าต้องการ
    keys = Redis.current.keys("#{PREFIX}*")
    keys.each { |k| Redis.current.del(k) unless Redis.current.ttl(k) > 0 }
  end
end

# config/initializers/redis.rb
Redis.current = Redis.new(url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0'))
```

### Refresh Token ใน Database

```ruby
# migration
class CreateRefreshTokens < ActiveRecord::Migration[7.0]
  def change
    create_table :refresh_tokens do |t|
      t.references :user, null: false, foreign_key: true
      t.string :token, null: false, index: { unique: true }
      t.string :device_info
      t.string :ip_address
      t.datetime :expires_at, null: false
      t.datetime :last_used_at
      t.boolean :revoked, default: false
      t.timestamps
    end
    
    add_index :refresh_tokens, :expires_at
  end
end

# app/models/refresh_token.rb
class RefreshToken < ApplicationRecord
  belongs_to :user
  
  before_create :generate_token
  
  scope :active, -> { where(revoked: false).where('expires_at > ?', Time.current) }
  
  def self.cleanup_expired
    where('expires_at < ?', Time.current).delete_all
  end
  
  def revoke!
    update!(revoked: true)
  end
  
  def expired?
    expires_at < Time.current
  end
  
  def valid_token?
    !revoked && !expired?
  end
  
  private
  
  def generate_token
    self.token = SecureRandom.hex(64)
    self.expires_at ||= 30.days.from_now
  end
end
```

---

## Step 1479: Devise with JWT

devise-jwt gem รวม Devise กับ JWT authentication

```ruby
# Gemfile
gem 'devise'
gem 'devise-jwt'

# Terminal
rails generate devise:install
rails generate devise User

# config/initializers/devise.rb
Devise.setup do |config|
  config.jwt do |jwt|
    jwt.secret = ENV.fetch('DEVISE_JWT_SECRET_KEY')
    jwt.dispatch_requests = [
      ['POST', %r{^/api/v1/users/sign_in$}],
      ['POST', %r{^/api/v1/users$}]
    ]
    jwt.revocation_requests = [
      ['DELETE', %r{^/api/v1/users/sign_out$}]
    ]
    jwt.expiration_time = 1.day.to_i
  end
end

# migration (เพิ่ม JTI สำหรับ revocation)
class AddJtiToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :jti, :string, null: false
    add_index :users, :jti, unique: true
  end
end

# app/models/user.rb
class User < ApplicationRecord
  include Devise::JWT::RevocationStrategies::JTIMatcher
  
  devise :database_authenticatable,
         :registerable,
         :recoverable,
         :validatable,
         :jwt_authenticatable,
         jwt_revocation_strategy: self
  
  before_create :generate_jti
  
  private
  
  def generate_jti
    self.jti = SecureRandom.uuid
  end
end

# config/routes.rb
namespace :api do
  namespace :v1 do
    devise_for :users,
               path: '',
               path_names: {
                 sign_in: 'login',
                 sign_out: 'logout',
                 registration: 'signup'
               },
               controllers: {
                 sessions: 'api/v1/sessions',
                 registrations: 'api/v1/registrations'
               }
  end
end

# app/controllers/api/v1/sessions_controller.rb
class Api::V1::SessionsController < Devise::SessionsController
  respond_to :json
  
  private
  
  def respond_with(resource, _opts = {})
    render json: {
      message: 'เข้าสู่ระบบสำเร็จ',
      user: UserSerializer.new(resource).as_json
    }, status: :ok
  end
  
  def respond_to_on_destroy
    if current_user
      render json: { message: 'ออกจากระบบสำเร็จ' }, status: :ok
    else
      render json: { message: 'ออกจากระบบสำเร็จ' }, status: :ok
    end
  end
end
```

---

## Step 1480: API Key Authentication

สำหรับ machine-to-machine authentication

```ruby
# migration
class CreateApiKeys < ActiveRecord::Migration[7.0]
  def change
    create_table :api_keys do |t|
      t.references :user, null: false, foreign_key: true
      t.string :key_hash, null: false, index: { unique: true }
      t.string :key_prefix, null: false  # แสดงในหน้า UI
      t.string :name, null: false
      t.string :description
      t.string :scopes, array: true, default: []
      t.datetime :last_used_at
      t.string :last_used_ip
      t.integer :request_count, default: 0
      t.datetime :expires_at
      t.boolean :active, default: true
      t.datetime :revoked_at
      t.timestamps
    end
  end
end

# app/models/api_key.rb
class ApiKey < ApplicationRecord
  belongs_to :user
  
  SCOPES = %w[read write admin webhook].freeze
  
  attr_reader :raw_key  # เก็บ plain text key ชั่วคราว
  
  before_create :generate_key
  
  scope :active, -> { where(active: true, revoked_at: nil) }
  scope :valid, -> {
    active.where('expires_at IS NULL OR expires_at > ?', Time.current)
  }
  
  validates :name, presence: true
  validates :scopes, inclusion: { in: SCOPES }
  
  def self.authenticate(raw_key)
    return nil unless raw_key.present?
    
    # key format: prefix_randomhex (e.g., sk_abc123...)
    prefix = raw_key.split('_').first
    key_record = valid.find_by(key_prefix: prefix)
    
    return nil unless key_record
    return nil unless key_record.matches_key?(raw_key)
    
    key_record.track_usage!
    key_record
  end
  
  def matches_key?(raw_key)
    BCrypt::Password.new(key_hash) == raw_key
  end
  
  def has_scope?(scope)
    scopes.include?(scope.to_s)
  end
  
  def revoke!
    update!(active: false, revoked_at: Time.current)
  end
  
  def track_usage!
    update_columns(
      last_used_at: Time.current,
      last_used_ip: Current.ip_address,
      request_count: request_count + 1
    )
  end
  
  def expired?
    expires_at.present? && expires_at < Time.current
  end
  
  private
  
  def generate_key
    raw = "sk_#{SecureRandom.hex(32)}"
    @raw_key = raw
    self.key_prefix = raw.split('_').first + '_' + raw[3, 8]
    self.key_hash = BCrypt::Password.create(raw)
  end
end

# ใน ApplicationController
class ApplicationController < ActionController::API
  before_action :authenticate!
  
  private
  
  def authenticate!
    if request.headers['X-API-Key'].present?
      authenticate_with_api_key!
    elsif request.headers['Authorization'].present?
      authenticate_with_jwt!
    else
      render json: { error: 'Authentication required' }, status: :unauthorized
    end
  end
  
  def authenticate_with_api_key!
    api_key = ApiKey.authenticate(request.headers['X-API-Key'])
    
    if api_key
      @current_user = api_key.user
      @current_api_key = api_key
    else
      render json: { error: 'Invalid API key' }, status: :unauthorized
    end
  end
end

# Controller สำหรับ manage API keys
class Api::V1::ApiKeysController < ApplicationController
  def index
    @api_keys = current_user.api_keys.order(created_at: :desc)
    render json: @api_keys
  end
  
  def create
    @api_key = current_user.api_keys.create!(api_key_params)
    
    # ส่ง raw key กลับไปครั้งเดียวเท่านั้น!
    render json: {
      api_key: @api_key.as_json.merge(raw_key: @api_key.raw_key),
      message: 'เก็บ API key นี้ไว้ให้ดี จะไม่แสดงอีกครั้ง'
    }, status: :created
  end
  
  def destroy
    @api_key = current_user.api_keys.find(params[:id])
    @api_key.revoke!
    render json: { message: 'ยกเลิก API key สำเร็จ' }
  end
  
  private
  
  def api_key_params
    params.require(:api_key).permit(:name, :description, :expires_at, scopes: [])
  end
end
```

---

## Step 1481: OAuth2 กับ Doorkeeper Gem

```ruby
# Gemfile
gem 'doorkeeper'

# Terminal
rails generate doorkeeper:install
rails generate doorkeeper:migration
rails db:migrate

# config/initializers/doorkeeper.rb
Doorkeeper.configure do
  orm :active_record
  
  resource_owner_authenticator do
    current_user || warden.authenticate!(scope: :user)
  end
  
  resource_owner_from_credentials do |routes|
    user = User.find_by(email: params[:username])
    user if user&.valid_password?(params[:password])
  end
  
  # Allowed grant types
  grant_flows %w[authorization_code client_credentials password refresh_token]
  
  # Token expiry
  access_token_expires_in 2.hours
  refresh_token_expires_in 1.month
  
  # Custom token response
  custom_access_token_expires_in do |context|
    case context.client&.application&.scopes
    when /admin/  then 1.hour
    when /mobile/ then 30.days
    else          2.hours
    end
  end
  
  # Scopes
  default_scopes :public
  optional_scopes :read, :write, :admin, :email, :profile
  
  # Base controller
  base_controller 'ApplicationController'
  
  # Use PKCE for public clients
  use_pkce_with_authorization_code_flow true
  
  # Handle revocation
  revoke_previous_client_credentials_token
end

# app/controllers/api/v1/protected_controller.rb
class Api::V1::ProtectedController < ApplicationController
  before_action :doorkeeper_authorize!
  
  private
  
  def current_user
    User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
  end
  
  def current_scopes
    doorkeeper_token.scopes
  end
  
  def require_scope!(scope)
    unless doorkeeper_token.scopes.include?(scope.to_s)
      render json: { error: "Requires #{scope} scope" }, status: :forbidden
    end
  end
end
```

### Client Application สำหรับ OAuth2

```ruby
# app/models/oauth_application.rb (Doorkeeper extends this)
# Doorkeeper สร้าง Doorkeeper::Application model ให้

# สร้าง OAuth application
app = Doorkeeper::Application.create!(
  name: 'My Mobile App',
  redirect_uri: 'myapp://oauth/callback',
  scopes: 'read write profile',
  confidential: false  # Public client (mobile/SPA)
)

puts "Client ID: #{app.uid}"
puts "Client Secret: #{app.secret}"

# routes
Rails.application.routes.draw do
  use_doorkeeper
  # Doorkeeper จะสร้าง routes:
  # GET /oauth/authorize
  # POST /oauth/authorize
  # DELETE /oauth/authorize
  # POST /oauth/token
  # POST /oauth/revoke
  # GET /oauth/token/info
end
```

---

## Step 1482: OmniAuth (Social Login)

```ruby
# Gemfile
gem 'omniauth'
gem 'omniauth-google-oauth2'
gem 'omniauth-facebook'
gem 'omniauth-github'
gem 'omniauth-rails_csrf_protection'

# config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2,
    ENV['GOOGLE_CLIENT_ID'],
    ENV['GOOGLE_CLIENT_SECRET'],
    {
      scope: 'email,profile',
      prompt: 'select_account',
      image_aspect_ratio: 'square',
      image_size: 200
    }
  
  provider :facebook,
    ENV['FACEBOOK_APP_ID'],
    ENV['FACEBOOK_APP_SECRET'],
    {
      scope: 'email,public_profile',
      info_fields: 'name,email,picture'
    }
  
  provider :github,
    ENV['GITHUB_CLIENT_ID'],
    ENV['GITHUB_CLIENT_SECRET'],
    scope: 'user:email'
end

OmniAuth.config.allowed_request_methods = %i[post]
OmniAuth.config.silence_get_warning = true

# migration
class CreateOauthIdentities < ActiveRecord::Migration[7.0]
  def change
    create_table :oauth_identities do |t|
      t.references :user, null: false, foreign_key: true
      t.string :provider, null: false
      t.string :uid, null: false
      t.string :access_token
      t.string :refresh_token
      t.datetime :token_expires_at
      t.jsonb :raw_info, default: {}
      t.timestamps
      
      t.index [:provider, :uid], unique: true
    end
  end
end

# app/models/oauth_identity.rb
class OauthIdentity < ApplicationRecord
  belongs_to :user
  
  validates :provider, :uid, presence: true
  validates :uid, uniqueness: { scope: :provider }
  
  def self.from_omniauth(auth)
    find_or_initialize_by(provider: auth.provider, uid: auth.uid) do |identity|
      identity.access_token = auth.credentials.token
      identity.refresh_token = auth.credentials.refresh_token
      identity.token_expires_at = auth.credentials.expires_at&.then { |t| Time.at(t) }
      identity.raw_info = auth.info.to_h
    end
  end
end

# app/models/user.rb
class User < ApplicationRecord
  has_many :oauth_identities, dependent: :destroy
  
  def self.from_omniauth(auth)
    # หา existing user จาก OAuth identity
    identity = OauthIdentity.find_by(provider: auth.provider, uid: auth.uid)
    return identity.user if identity
    
    # หา user จาก email
    user = find_or_initialize_by(email: auth.info.email)
    
    if user.new_record?
      user.assign_attributes(
        name: auth.info.name,
        password: SecureRandom.hex(20),
        email_verified_at: Time.current
      )
      user.save!
    end
    
    # สร้าง OAuth identity
    user.oauth_identities.find_or_create_by!(
      provider: auth.provider,
      uid: auth.uid
    ) do |i|
      i.raw_info = auth.info.to_h
    end
    
    user
  end
end

# app/controllers/omniauth_callbacks_controller.rb
class OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def google_oauth2
    handle_auth('Google')
  end
  
  def facebook
    handle_auth('Facebook')
  end
  
  def github
    handle_auth('GitHub')
  end
  
  private
  
  def handle_auth(provider_name)
    auth = request.env['omniauth.auth']
    
    @user = User.from_omniauth(auth)
    
    if @user.persisted?
      sign_in_and_redirect @user, event: :authentication
      
      if is_navigational_format?
        set_flash_message(:notice, :success, kind: provider_name)
      end
    else
      session['devise.omniauth_data'] = auth.except('extra')
      redirect_to new_user_registration_url
    end
  rescue => e
    Rails.logger.error "OmniAuth error: #{e.message}"
    redirect_to root_path, alert: 'เข้าสู่ระบบผ่าน Social ไม่สำเร็จ'
  end
  
  def failure
    redirect_to root_path, alert: "ยกเลิกการเข้าสู่ระบบผ่าน #{failed_strategy.name}"
  end
end
```

---

## Step 1483: Token Rotation and Security

```ruby
# app/services/token_rotation.rb
class TokenRotation
  # Rotate tokens เป็นระยะ เพื่อ limit damage จาก token theft
  
  def self.rotate_for(user)
    new_jti = SecureRandom.uuid
    user.update!(jti: new_jti)
    new_jti
  end
  
  # ตรวจสอบ suspicious activity
  def self.detect_reuse!(token_payload)
    jti = token_payload[:jti]
    user_id = token_payload[:user_id]
    
    user = User.find(user_id)
    
    # ถ้า jti ไม่ตรงกับ user's current jti = token ถูก reuse หลัง rotation
    if user.jti != jti
      # อาจมีการขโมย token - revoke ทุก sessions
      user.update!(jti: SecureRandom.uuid)
      
      SecurityAlert.create!(
        user: user,
        alert_type: 'token_reuse_detected',
        severity: 'high',
        details: {
          reused_jti: jti,
          ip_address: Current.ip_address,
          user_agent: Current.user_agent
        }
      )
      
      UserMailer.security_alert(user, 'possible_account_compromise').deliver_later
      
      raise AuthenticationError, 'Security violation detected. All sessions have been revoked.'
    end
  end
end

# Security Headers Middleware
class SecurityHeadersMiddleware
  def initialize(app)
    @app = app
  end
  
  def call(env)
    status, headers, response = @app.call(env)
    
    headers['X-Content-Type-Options'] = 'nosniff'
    headers['X-Frame-Options'] = 'DENY'
    headers['X-XSS-Protection'] = '1; mode=block'
    headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
    headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'
    
    [status, headers, response]
  end
end

# config/application.rb
config.middleware.insert_before 0, SecurityHeadersMiddleware
```

---

## Step 1484: Rate Limiting API

```ruby
# Gemfile
gem 'rack-attack'

# config/initializers/rack_attack.rb
class Rack::Attack
  # Throttle all requests by IP
  throttle('req/ip', limit: 300, period: 5.minutes) do |req|
    req.ip unless req.path.start_with?('/assets')
  end
  
  # Throttle login attempts
  throttle('login/ip', limit: 5, period: 20.seconds) do |req|
    if req.path == '/api/v1/auth/login' && req.post?
      req.ip
    end
  end
  
  throttle('login/email', limit: 5, period: 20.seconds) do |req|
    if req.path == '/api/v1/auth/login' && req.post?
      req.params['email']&.downcase&.gsub(/\s+/, '')
    end
  end
  
  # Throttle API by token
  throttle('api/token', limit: 1000, period: 1.hour) do |req|
    if req.path.start_with?('/api/')
      req.env['HTTP_AUTHORIZATION']&.split(' ')&.last
    end
  end
  
  # Block suspicious IPs
  blocklist('block/bad-actors') do |req|
    BlockedIp.exists?(ip_address: req.ip)
  end
  
  # Return JSON for throttled requests
  self.throttled_responder = lambda do |env|
    now = Time.now
    match_data = env['rack.attack.match_data']
    
    headers = {
      'Content-Type' => 'application/json',
      'X-RateLimit-Limit' => match_data[:limit].to_s,
      'X-RateLimit-Remaining' => '0',
      'X-RateLimit-Reset' => (now + (match_data[:period] - now.to_i % match_data[:period])).to_s,
      'Retry-After' => match_data[:period].to_s
    }
    
    [429, headers, [{ error: 'Too many requests. Please try again later.', retry_after: match_data[:period] }.to_json]]
  end
  
  # Log throttle events
  ActiveSupport::Notifications.subscribe('rack.attack') do |name, start, finish, request_id, payload|
    req = payload[:request]
    
    if [:throttle, :blocklist].include?(req.env['rack.attack.match_type'])
      Rails.logger.warn "Rack::Attack #{req.env['rack.attack.match_type']}: " \
        "#{req.env['rack.attack.matched']} #{req.ip} #{req.path}"
    end
  end
end

# config/application.rb
config.middleware.use Rack::Attack

# ใน Controller: เพิ่ม rate limit headers
class ApplicationController < ActionController::API
  after_action :set_rate_limit_headers
  
  private
  
  def set_rate_limit_headers
    # Rack::Attack จัดการ headers หลักอยู่แล้ว
    # เพิ่ม custom headers ตามต้องการ
    response.headers['X-API-Version'] = 'v1'
  end
end
```

---

## Step 1485: Two-Factor Authentication (2FA)

```ruby
# Gemfile
gem 'rotp'
gem 'rqrcode'

# migration
class AddTwoFactorToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :otp_secret, :string
    add_column :users, :otp_required_for_login, :boolean, default: false
    add_column :users, :otp_backup_codes, :text, array: true, default: []
  end
end

# app/models/user.rb
class User < ApplicationRecord
  BACKUP_CODES_COUNT = 8
  
  def setup_2fa!
    self.otp_secret = ROTP::Base32.random
    save!
    
    {
      secret: otp_secret,
      qr_code_uri: otp_provisioning_uri,
      qr_code_svg: generate_qr_code
    }
  end
  
  def enable_2fa!(otp_code)
    if verify_otp(otp_code)
      codes = generate_backup_codes
      update!(
        otp_required_for_login: true,
        otp_backup_codes: codes.map { |c| BCrypt::Password.create(c) }
      )
      codes  # Return plain text codes (แสดงครั้งเดียว)
    else
      nil
    end
  end
  
  def disable_2fa!
    update!(
      otp_required_for_login: false,
      otp_secret: nil,
      otp_backup_codes: []
    )
  end
  
  def verify_otp(code)
    totp = ROTP::TOTP.new(otp_secret, issuer: 'MyApp')
    totp.verify(code.to_s, drift_behind: 15, drift_ahead: 15)
  end
  
  def verify_backup_code(code)
    otp_backup_codes.each_with_index do |hashed_code, index|
      if BCrypt::Password.new(hashed_code) == code
        # ใช้ backup code แล้ว ลบออก
        remaining_codes = otp_backup_codes.dup
        remaining_codes.delete_at(index)
        update!(otp_backup_codes: remaining_codes)
        return true
      end
    end
    false
  end
  
  private
  
  def otp_provisioning_uri
    ROTP::TOTP.new(otp_secret, issuer: 'MyApp').provisioning_uri(email)
  end
  
  def generate_qr_code
    qrcode = RQRCode::QRCode.new(otp_provisioning_uri)
    qrcode.as_svg(
      offset: 0,
      color: '000',
      shape_rendering: 'crispEdges',
      module_size: 11
    )
  end
  
  def generate_backup_codes
    BACKUP_CODES_COUNT.times.map { SecureRandom.alphanumeric(10).upcase }
  end
end

# app/controllers/api/v1/two_factor_controller.rb
class Api::V1::TwoFactorController < ApplicationController
  # Setup 2FA
  def setup
    result = current_user.setup_2fa!
    render json: result
  end
  
  # Enable 2FA หลัง verify OTP
  def enable
    backup_codes = current_user.enable_2fa!(params[:otp_code])
    
    if backup_codes
      render json: {
        message: 'เปิดใช้งาน 2FA สำเร็จ',
        backup_codes: backup_codes,
        warning: 'เก็บ backup codes เหล่านี้ไว้ให้ดี จะไม่แสดงอีกครั้ง'
      }
    else
      render json: { error: 'รหัส OTP ไม่ถูกต้อง' }, status: :unprocessable_entity
    end
  end
  
  # Verify OTP ตอน login
  def verify
    session_token = params[:session_token]
    otp_code = params[:otp_code]
    
    # ตรวจสอบ session token ที่สร้างหลัง login ครั้งแรก
    pending_user_id = Rails.cache.read("2fa_pending:#{session_token}")
    
    unless pending_user_id
      return render json: { error: 'Session หมดอายุ' }, status: :unauthorized
    end
    
    user = User.find(pending_user_id)
    
    if user.verify_otp(otp_code) || user.verify_backup_code(otp_code)
      Rails.cache.delete("2fa_pending:#{session_token}")
      
      access_token = JsonWebToken.encode(user_id: user.id, type: 'access')
      refresh_token = JsonWebToken.encode(user_id: user.id, type: 'refresh', expiry: 30.days)
      
      render json: {
        access_token: access_token,
        refresh_token: refresh_token,
        user: UserSerializer.new(user).as_json
      }
    else
      render json: { error: 'รหัส OTP ไม่ถูกต้อง' }, status: :unauthorized
    end
  end
end
```

---

## Step 1486-1495: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: JWT Middleware

```ruby
# สร้าง JWT middleware สำหรับ Rails API
# app/middlewares/jwt_authentication_middleware.rb
class JwtAuthenticationMiddleware
  PUBLIC_PATHS = %w[
    /api/v1/auth/login
    /api/v1/auth/register
    /api/v1/auth/refresh
    /health
  ].freeze
  
  def initialize(app)
    @app = app
  end
  
  def call(env)
    request = ActionDispatch::Request.new(env)
    
    if public_path?(request.path)
      @app.call(env)
    else
      authenticate(env, request)
    end
  end
  
  private
  
  def public_path?(path)
    PUBLIC_PATHS.any? { |p| path.start_with?(p) }
  end
  
  def authenticate(env, request)
    token = extract_token(request)
    
    unless token
      return unauthorized_response('Token required')
    end
    
    begin
      payload = JsonWebToken.decode(token)
      env['current_user_id'] = payload[:user_id]
      env['current_user_role'] = payload[:role]
      @app.call(env)
    rescue AuthenticationError => e
      unauthorized_response(e.message)
    end
  end
  
  def extract_token(request)
    header = request.headers['Authorization']
    header&.gsub(/^Bearer\s/, '')
  end
  
  def unauthorized_response(message)
    [
      401,
      { 'Content-Type' => 'application/json' },
      [{ error: message }.to_json]
    ]
  end
end
```

### แบบฝึกหัดที่ 2: Scoped API Keys

```ruby
# สร้างระบบ API keys ที่มี scopes
class ScopedApiKeyAuth
  PERMISSION_MAP = {
    'GET' => :read,
    'POST' => :write,
    'PUT' => :write,
    'PATCH' => :write,
    'DELETE' => :delete
  }.freeze
  
  def self.authorize!(api_key, request)
    required_scope = PERMISSION_MAP[request.method]
    
    unless api_key.has_scope?(required_scope)
      raise AuthorizationError, "API key ต้องการ #{required_scope} scope"
    end
    
    # Path-based scopes
    if request.path.start_with?('/api/v1/admin/')
      unless api_key.has_scope?(:admin)
        raise AuthorizationError, 'ต้องการ admin scope'
      end
    end
  end
end

# ใน Controller
class ApplicationController < ActionController::API
  before_action :check_api_key_scopes, if: :api_key_authenticated?
  
  private
  
  def api_key_authenticated?
    @current_api_key.present?
  end
  
  def check_api_key_scopes
    ScopedApiKeyAuth.authorize!(@current_api_key, request)
  rescue AuthorizationError => e
    render json: { error: e.message }, status: :forbidden
  end
end
```

### แบบฝึกหัดที่ 3: OAuth2 Resource Server

```ruby
# app/controllers/concerns/oauth_resource_server.rb
module OauthResourceServer
  extend ActiveSupport::Concern
  
  included do
    before_action :validate_oauth_token
  end
  
  private
  
  def validate_oauth_token
    token_string = request.headers['Authorization']&.gsub(/^Bearer\s/, '')
    
    unless token_string
      return render_oauth_error('invalid_request', 'Missing token', 401)
    end
    
    @oauth_token = OauthAccessToken.find_by(token: token_string)
    
    if @oauth_token.nil?
      render_oauth_error('invalid_token', 'Token not found', 401)
    elsif @oauth_token.expired?
      render_oauth_error('invalid_token', 'Token expired', 401)
    elsif @oauth_token.revoked?
      render_oauth_error('invalid_token', 'Token has been revoked', 401)
    else
      @current_user = @oauth_token.resource_owner
    end
  end
  
  def require_oauth_scope(scope)
    unless @oauth_token.scopes.include?(scope.to_s)
      render_oauth_error('insufficient_scope', "Required scope: #{scope}", 403)
    end
  end
  
  def render_oauth_error(error, description, status)
    response.headers['WWW-Authenticate'] = 
      "Bearer realm=\"API\", error=\"#{error}\", error_description=\"#{description}\""
    
    render json: { error: error, error_description: description }, status: status
  end
end
```

### แบบฝึกหัดที่ 4: Complete Authentication Test

```ruby
# spec/requests/api/v1/auth_spec.rb
require 'rails_helper'

RSpec.describe 'API Authentication', type: :request do
  describe 'POST /api/v1/auth/login' do
    let!(:user) { create(:user, email: 'test@example.com', password: 'password123') }
    
    context 'with valid credentials' do
      it 'returns access and refresh tokens' do
        post '/api/v1/auth/login', params: {
          email: 'test@example.com',
          password: 'password123'
        }
        
        expect(response).to have_http_status(:ok)
        expect(json_body).to include('access_token', 'refresh_token')
        
        # Verify token is valid
        payload = JsonWebToken.decode(json_body['access_token'])
        expect(payload[:user_id]).to eq(user.id)
      end
    end
    
    context 'with invalid credentials' do
      it 'returns unauthorized' do
        post '/api/v1/auth/login', params: {
          email: 'test@example.com',
          password: 'wrong'
        }
        
        expect(response).to have_http_status(:unauthorized)
        expect(json_body).to have_key('error')
      end
    end
    
    context 'rate limiting' do
      it 'blocks after 5 failed attempts' do
        6.times do
          post '/api/v1/auth/login', params: {
            email: 'test@example.com',
            password: 'wrong'
          }
        end
        
        expect(response).to have_http_status(429)
      end
    end
  end
  
  describe 'accessing protected endpoint' do
    let!(:user) { create(:user) }
    let(:token) { JsonWebToken.encode(user_id: user.id, type: 'access') }
    
    it 'allows access with valid token' do
      get '/api/v1/profile',
        headers: { 'Authorization' => "Bearer #{token}" }
      
      expect(response).to have_http_status(:ok)
    end
    
    it 'denies access without token' do
      get '/api/v1/profile'
      expect(response).to have_http_status(:unauthorized)
    end
    
    it 'denies access with expired token' do
      expired_token = JsonWebToken.encode(user_id: user.id, expiry: 1.hour.ago)
      
      get '/api/v1/profile',
        headers: { 'Authorization' => "Bearer #{expired_token}" }
      
      expect(response).to have_http_status(:unauthorized)
    end
  end
end

def json_body
  JSON.parse(response.body)
end
```

---

**จบ Part 68: API Authentication**

*ในส่วนถัดไป Part 69 เราจะเรียนรู้เกี่ยวกับ Advanced Testing*
