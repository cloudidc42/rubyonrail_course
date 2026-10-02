# ตอนที่ 29: Security ใน Ruby (ขั้นตอนที่ 626-645)

Security เป็นหนึ่งในเรื่องที่สำคัญที่สุดในการพัฒนาซอฟต์แวร์ การไม่ให้ความสำคัญกับ Security อาจนำไปสู่การรั่วไหลของข้อมูล การถูก Hack หรือความเสียหายทางธุรกิจ

---

## ขั้นตอนที่ 626: Input Validation

### การตรวจสอบ Input อย่างปลอดภัย

```ruby
# Input Validation พื้นฐาน
class UserValidator
  def self.validate_email(email)
    return nil unless email.is_a?(String)
    
    email = email.strip.downcase
    
    unless email.match?(/\A[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\z/)
      raise ArgumentError, "Email ไม่ถูกต้อง"
    end
    
    email
  end
  
  def self.validate_phone(phone)
    return nil unless phone.is_a?(String)
    
    phone = phone.gsub(/[\s\-\(\)]/, '')
    
    unless phone.match?(/\A(\+66|0)\d{8,9}\z/)
      raise ArgumentError, "เบอร์โทรศัพท์ไม่ถูกต้อง"
    end
    
    phone
  end
  
  def self.validate_age(age)
    age = Integer(age)  # Strict conversion
    raise ArgumentError, "อายุต้องเป็น 0-150" unless (0..150).include?(age)
    age
  rescue ArgumentError, TypeError
    raise ArgumentError, "อายุไม่ถูกต้อง"
  end
  
  def self.validate_username(username)
    return nil unless username.is_a?(String)
    
    username = username.strip
    
    if username.length < 3 || username.length > 30
      raise ArgumentError, "Username ต้องมี 3-30 ตัวอักษร"
    end
    
    unless username.match?(/\A[a-zA-Z0-9_\-]+\z/)
      raise ArgumentError, "Username ใช้ได้เฉพาะ a-z, 0-9, _, -"
    end
    
    username.downcase
  end
end

# Whitelist vs Blacklist
# Whitelist (ดีกว่า): กำหนดสิ่งที่อนุญาต
ALLOWED_SORT_COLUMNS = %w[name email created_at updated_at].freeze

def safe_sort_column(column)
  ALLOWED_SORT_COLUMNS.include?(column) ? column : 'created_at'
end

# Blacklist (อันตราย): กำหนดสิ่งที่ไม่อนุญาต
FORBIDDEN_CHARS = ['<', '>', '"', "'", '\\', '/'].freeze

def blacklist_filter(input)  # ไม่แนะนำ - มีช่องโหว่
  FORBIDDEN_CHARS.each { |char| input.delete!(char) }
  input
end
```

### Strong Parameters ใน Rails

```ruby
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    
    if @user.save
      redirect_to @user
    else
      render :new
    end
  end
  
  def update
    @user = User.find(params[:id])
    
    if @user.update(user_update_params)
      redirect_to @user
    else
      render :edit
    end
  end
  
  private
  
  # กรอง Parameters อย่างเข้มงวด
  def user_params
    params.require(:user).permit(
      :name,
      :email,
      :password,
      :password_confirmation,
      :phone,
      address: [:street, :city, :province, :postcode]
    )
  end
  
  # Update อาจมีบาง Field ที่ต่างออกไป
  def user_update_params
    params.require(:user).permit(
      :name,
      :phone,
      :avatar,
      address: [:street, :city, :province, :postcode]
    )
    # ไม่อนุญาต :email, :password ใน Update ปกติ
  end
end
```

---

## ขั้นตอนที่ 627: SQL Injection Prevention

### SQL Injection คืออะไร

```ruby
# VULNERABLE: SQL Injection!
def find_user_vulnerable(email)
  User.find_by("email = '#{email}'")
  # ถ้า email = "' OR '1'='1"
  # SQL จะเป็น: email = '' OR '1'='1' (เข้าถึงได้ทั้งหมด!)
end

# SAFE: Parameterized Query
def find_user_safe(email)
  User.find_by("email = ?", email)
  # หรือ
  User.where(email: email).first
  # หรือ
  User.find_by(email: email)
end

# More Examples
# VULNERABLE
query = "SELECT * FROM orders WHERE user_id = #{params[:user_id]}"
ActiveRecord::Base.connection.execute(query)

# SAFE
Order.where(user_id: params[:user_id])
# หรือ
Order.where("user_id = ?", params[:user_id].to_i)

# Named Parameters
User.where("created_at > :start_date AND status = :status", {
  start_date: 1.month.ago,
  status: 'active'
})

# LIKE Query - ต้องระวัง
# VULNERABLE
User.where("name LIKE '%#{search_term}%'")

# SAFE
escaped_term = ActiveRecord::Base.sanitize_sql_like(search_term)
User.where("name LIKE ?", "%#{escaped_term}%")

# Raw SQL (ต้องใช้ด้วยความระมัดระวัง)
ActiveRecord::Base.connection.execute(
  ActiveRecord::Base.sanitize_sql(
    ["SELECT * FROM users WHERE email = ?", email]
  )
)
```

---

## ขั้นตอนที่ 628: XSS Prevention

### Cross-Site Scripting

```ruby
# XSS ใน Rails
# Rails auto-escape HTML ใน Views โดยค่าเริ่มต้น

# SAFE: Rails escape โดยอัตโนมัติ
# <%= @user.name %>
# แสดง: &lt;script&gt;alert('XSS')&lt;/script&gt;

# UNSAFE: ใช้ html_safe หรือ raw
# <%= @user.name.html_safe %>  # อันตราย!
# <%= raw @user.description %> # อันตราย!

# ใช้ sanitize เมื่อต้องการ HTML
# <%= sanitize @article.body %>
# เก็บเฉพาะ HTML Tags ที่ปลอดภัย

# กำหนด Allowed Tags
allowed_tags = %w[p br b i strong em a ul ol li blockquote]
allowed_attrs = %w[href title class]

sanitized = ActionView::Helpers::SanitizeHelper.sanitize(
  user_content,
  tags: allowed_tags,
  attributes: allowed_attrs
)

# Content Security Policy (CSP)
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https, :unsafe_inline
  
  # Add nonce support for inline scripts
  policy.script_src :self, :https, -> { "'nonce-#{SecureRandom.base64(16)}'" }
end

# Ruby String Escaping
require 'cgi'
CGI.escapeHTML("<script>alert('xss')</script>")
# => "&lt;script&gt;alert('xss')&lt;/script&gt;"

require 'erb'
ERB::Util.html_escape("<script>")
# => "&lt;script&gt;"
```

---

## ขั้นตอนที่ 629: CSRF Protection

```ruby
# Rails CSRF Protection (Auto-enabled)
# ApplicationController มี protect_from_forgery โดยค่าเริ่มต้น

class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  # หรือ
  protect_from_forgery with: :null_session  # สำหรับ API
end

# สำหรับ API Endpoints
class ApiController < ActionController::Base
  protect_from_forgery with: :null_session
  
  before_action :authenticate_api_token!
  
  private
  
  def authenticate_api_token!
    token = request.headers['X-API-Token'] || request.headers['Authorization']&.split(' ')&.last
    
    unless valid_token?(token)
      render json: { error: 'Unauthorized' }, status: :unauthorized
    end
  end
  
  def valid_token?(token)
    return false unless token
    ApiToken.active.find_by(token: token).present?
  end
end

# CSRF Token ใน HTML Forms
# Rails เพิ่ม Token อัตโนมัติ:
# <input type="hidden" name="authenticity_token" value="<token>">

# ใน JavaScript (Axios)
# axios.defaults.headers.common['X-CSRF-Token'] = 
#   document.querySelector('meta[name="csrf-token"]').getAttribute('content')
```

---

## ขั้นตอนที่ 630: Secure Random Tokens

```ruby
require 'securerandom'

# Generate Secure Random Tokens
SecureRandom.hex(32)       # 64 char hex string
SecureRandom.base64(32)    # Base64 string
SecureRandom.urlsafe_base64(32)  # URL-safe Base64
SecureRandom.uuid          # UUID v4
SecureRandom.alphanumeric(20)   # Alphanumeric string

# Token สำหรับ Password Reset
class User < ApplicationRecord
  before_create :generate_confirmation_token
  
  def generate_reset_password_token!
    raw_token = SecureRandom.urlsafe_base64(32)
    self.reset_password_token      = Digest::SHA256.hexdigest(raw_token)
    self.reset_password_sent_at    = Time.current
    save!
    raw_token  # ส่ง raw token ให้ User, เก็บ hashed version ใน DB
  end
  
  def reset_password_token_valid?
    reset_password_sent_at > 2.hours.ago
  end
  
  def self.find_by_reset_token(raw_token)
    hashed = Digest::SHA256.hexdigest(raw_token)
    find_by(reset_password_token: hashed)
  end
  
  private
  
  def generate_confirmation_token
    self.confirmation_token = SecureRandom.urlsafe_base64(32)
  end
end

# Timing-safe Comparison (ป้องกัน Timing Attack)
def secure_compare(a, b)
  return false if a.bytesize != b.bytesize
  
  l = a.unpack("C#{a.bytesize}")
  r = 0
  
  b.each_byte { |byte| r |= byte ^ l.shift }
  r == 0
end

# หรือใช้ ActiveSupport::SecurityUtils
require 'active_support/security_utils'
ActiveSupport::SecurityUtils.secure_compare(a, b)
```

---

## ขั้นตอนที่ 631: Password Hashing

```ruby
# ใช้ bcrypt สำหรับ Hash Password
gem 'bcrypt'

require 'bcrypt'

class UserAccount
  attr_reader :email
  
  def initialize(email, plain_password)
    @email          = email
    @password_digest = BCrypt::Password.create(plain_password, cost: 12)
  end
  
  def authenticate(plain_password)
    BCrypt::Password.new(@password_digest) == plain_password
  end
  
  def password_digest
    @password_digest.to_s
  end
end

# การใช้งาน
user = UserAccount.new('test@example.com', 'MySecureP@ssw0rd!')

puts user.authenticate('MySecureP@ssw0rd!')  # true
puts user.authenticate('wrong_password')       # false

# Rails has_secure_password
class User < ApplicationRecord
  has_secure_password  # ต้องมี password_digest column
  
  validates :password, 
    length: { minimum: 8 },
    format: { 
      with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*])/,
      message: "ต้องมีตัวพิมพ์ใหญ่ พิมพ์เล็ก ตัวเลข และอักขระพิเศษ"
    }
end

# ใช้งาน
user = User.new(email: 'test@example.com', password: 'SecureP@ss1')
user.save

user.authenticate('SecureP@ss1')   # คืน user object
user.authenticate('wrong')          # คืน false

# Argon2 (แนะนำมากกว่า bcrypt)
gem 'argon2'

require 'argon2'

password_hasher = Argon2::Password.new(t_cost: 2, m_cost: 16)
hashed = password_hasher.create("my_password")

puts Argon2::Password.verify_password("my_password", hashed)  # true
puts Argon2::Password.verify_password("wrong", hashed)         # false
```

---

## ขั้นตอนที่ 632: Secrets Management

### Rails Credentials

```bash
# สร้าง/แก้ไข Credentials
rails credentials:edit

# แก้ไขสำหรับ Environment เฉพาะ
rails credentials:edit --environment production
rails credentials:edit --environment staging
```

```yaml
# config/credentials.yml.enc (หลัง decrypt)
aws:
  access_key_id: "AKIAIOSFODNN7EXAMPLE"
  secret_access_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

stripe:
  publishable_key: "pk_test_xxx"
  secret_key: "sk_test_xxx"
  webhook_secret: "whsec_xxx"

sendgrid:
  api_key: "SG.xxx"

jwt_secret: "super_secret_key_for_jwt"

database:
  password: "secure_db_password"
```

```ruby
# ใช้งาน Credentials ใน Code
class PaymentService
  def self.stripe_client
    Stripe.api_key = Rails.application.credentials.stripe[:secret_key]
    Stripe
  end
end

# Environment Variables ทางเลือก
class Config
  def self.stripe_secret_key
    ENV['STRIPE_SECRET_KEY'] || 
      Rails.application.credentials.dig(:stripe, :secret_key)
  end
  
  def self.jwt_secret
    ENV['JWT_SECRET'] || 
      Rails.application.credentials.jwt_secret ||
      raise("JWT_SECRET ไม่ได้ตั้งค่า!")
  end
end

# .env file (ไม่ควร Commit ใน Git!)
# .gitignore
# .env
# .env.*

# ใช้ dotenv gem
require 'dotenv'
Dotenv.load

stripe_key = ENV['STRIPE_SECRET_KEY']
```

---

## ขั้นตอนที่ 633: SSL/TLS Basics

```ruby
# Force HTTPS ใน Rails
class ApplicationController < ActionController::Base
  force_ssl if Rails.env.production?
end

# config/environments/production.rb
config.force_ssl = true

# SSL/TLS Configuration ใน Nginx
# server {
#   listen 80;
#   server_name example.com;
#   return 301 https://$server_name$request_uri;
# }
#
# server {
#   listen 443 ssl http2;
#   server_name example.com;
#   ssl_certificate     /etc/ssl/example.com.crt;
#   ssl_certificate_key /etc/ssl/example.com.key;
#   ssl_protocols       TLSv1.2 TLSv1.3;
#   ssl_ciphers         HIGH:!aNULL:!MD5;
# }

# HSTS (HTTP Strict Transport Security)
# config/environments/production.rb
config.force_ssl = true
# Rails จะเพิ่ม Strict-Transport-Security Header โดยอัตโนมัติ

# Verify SSL Certificate ใน HTTP Requests
require 'net/https'

uri  = URI('https://api.example.com')
http = Net::HTTP.new(uri.host, uri.port)
http.use_ssl = true
http.verify_mode = OpenSSL::SSL::VERIFY_PEER  # Default
# ห้ามใช้ VERIFY_NONE ใน Production!

# Certificate Pinning
http.ca_file = '/path/to/cert-bundle.crt'
```

---

## ขั้นตอนที่ 634: Bundler-audit

```bash
# ติดตั้ง
gem install bundler-audit

# อัพเดท Advisory Database
bundle-audit update

# ตรวจสอบ
bundle-audit check

# Example Output:
# Name: rack
# Version: 2.2.6
# Advisory: CVE-2022-44570
# Criticality: Medium
# URL: https://github.com/advisories/GHSA-65f5-mfpf-vfhj
# Title: Denial of Service Vulnerability in Rack multipart
# Solution: upgrade to >= 2.2.6.3

# แก้ไขโดยอัพเดท Gem
bundle update rack

# เพิ่มใน CI/CD
# .github/workflows/security.yml
name: Security Audit
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: gem install bundler-audit
      - run: bundle-audit check --update
```

### Brakeman (Static Analysis)

```bash
gem install brakeman

# วิเคราะห์ Rails App
brakeman

# Output HTML
brakeman -o brakeman.html

# ตรวจสอบเฉพาะ
brakeman --checks SqlInjection,XSS,CSRF
```

---

## ขั้นตอนที่ 635-640: Security Best Practices

### Secure Headers

```ruby
# Gemfile
gem 'secure_headers'

# config/initializers/secure_headers.rb
SecureHeaders::Configuration.default do |config|
  config.cookies = {
    secure: true,
    httponly: true,
    samesite: {
      lax: true
    }
  }
  
  config.x_frame_options          = "SAMEORIGIN"
  config.x_content_type_options   = "nosniff"
  config.x_xss_protection         = "1; mode=block"
  config.x_download_options       = "noopen"
  config.x_permitted_cross_domain_policies = "none"
  
  config.referrer_policy          = "strict-origin-when-cross-origin"
  
  config.csp = {
    default_src: %w('self'),
    script_src:  %w('self' https://cdn.jsdelivr.net),
    style_src:   %w('self' 'unsafe-inline' https://fonts.googleapis.com),
    img_src:     %w('self' data: https:),
    font_src:    %w('self' https://fonts.gstatic.com),
    connect_src: %w('self' https://api.example.com),
    frame_src:   %w('none'),
    object_src:  %w('none')
  }
end
```

### Authorization

```ruby
# Pundit สำหรับ Authorization
gem 'pundit'

class ApplicationController < ActionController::Base
  include Pundit::Authorization
  
  rescue_from Pundit::NotAuthorizedError do |exception|
    redirect_to root_path, alert: "ไม่มีสิทธิ์เข้าถึง"
  end
end

class OrderPolicy < ApplicationPolicy
  def show?
    user == record.user || user.admin?
  end
  
  def update?
    user == record.user && record.pending?
  end
  
  def destroy?
    user.admin? || (user == record.user && record.pending?)
  end
  
  class Scope < Scope
    def resolve
      if user.admin?
        scope.all
      else
        scope.where(user: user)
      end
    end
  end
end

class OrdersController < ApplicationController
  def show
    @order = Order.find(params[:id])
    authorize @order  # Check OrderPolicy#show?
  end
  
  def index
    @orders = policy_scope(Order)  # ใช้ OrderPolicy::Scope
  end
end
```

### Mass Assignment Protection

```ruby
# ป้องกัน Mass Assignment Attack
class User < ApplicationRecord
  # attr_accessible เดิม (Rails 3)
  # ใน Rails 4+ ใช้ Strong Parameters แทน
  
  # Internal คือไม่ผ่าน Strong Parameters
  def promote_to_admin!
    update_column(:role, 'admin')
  end
  
  # User ไม่สามารถ Set role โดยตรงผ่าน Web Form
  # ต้องใช้ Method ที่กำหนดแล้ว
end

class UsersController < ApplicationController
  def create
    @user = User.new(user_params)  # user_params ไม่มี :role
    
    if @user.save
      redirect_to @user
    end
  end
  
  private
  
  def user_params
    params.require(:user).permit(:name, :email, :password)
    # ไม่ include :role!
  end
end
```

---

## แบบฝึกหัดบทที่ 29 (20 ข้อ)

**ข้อ 1:** สร้าง Input Validator สำหรับ Thai National ID (13 หลัก)

**ข้อ 2:** Implement SQL Injection Test ด้วย Malicious Inputs และแก้ให้ปลอดภัย

**ข้อ 3:** สร้าง XSS Test ใน Rails View และใช้ sanitize อย่างถูกต้อง

**ข้อ 4:** Implement CSRF Token Validation สำหรับ API Endpoint

**ข้อ 5:** สร้าง Password Strength Validator ที่ตรวจสอบความซับซ้อน

**ข้อ 6:** ใช้ bcrypt Hash Password และ Verify อย่างถูกต้อง

**ข้อ 7:** Setup Rails Credentials สำหรับ API Keys ต่างๆ

**ข้อ 8:** สร้าง Rate Limiter สำหรับ Login Attempts

**ข้อ 9:** ใช้ bundler-audit ตรวจสอบโปรเจกต์และแก้ไข Vulnerabilities

**ข้อ 10:** Implement Secure File Upload ด้วย Whitelist ของ File Types

**ข้อ 11:** สร้าง JWT Authentication ที่ปลอดภัย

**ข้อ 12:** Implement Secure Password Reset Flow

**ข้อ 13:** ตั้งค่า Content Security Policy (CSP) ใน Rails

**ข้อ 14:** สร้าง Authorization System ด้วย Pundit

**ข้อ 15:** Implement Audit Log สำหรับ Sensitive Operations

**ข้อ 16:** ป้องกัน Directory Traversal Attack ใน File Operations

**ข้อ 17:** สร้าง Secure Token Generation และ Comparison

**ข้อ 18:** ใช้ brakeman วิเคราะห์ Rails App และแก้ไข Warnings

**ข้อ 19:** Implement 2FA (Two-Factor Authentication) ด้วย TOTP

**ข้อ 20:** ทำ Security Audit สำหรับ Rails App ขนาดเล็ก

---

### เฉลยตัวอย่าง ข้อ 8: Rate Limiter สำหรับ Login

```ruby
# Rate Limiter ใน Rails ด้วย Redis

class LoginRateLimiter
  MAX_ATTEMPTS = 5
  LOCKOUT_TIME = 30.minutes
  WINDOW_SIZE  = 1.hour
  
  def initialize(redis: Redis.new)
    @redis = redis
  end
  
  def allow_attempt?(identifier)
    !locked?(identifier)
  end
  
  def record_failure!(identifier)
    key = failure_key(identifier)
    
    @redis.multi do
      @redis.incr(key)
      @redis.expire(key, WINDOW_SIZE.to_i)
    end
    
    if failure_count(identifier) >= MAX_ATTEMPTS
      lock!(identifier)
    end
  end
  
  def record_success!(identifier)
    @redis.del(failure_key(identifier))
    @redis.del(lock_key(identifier))
  end
  
  def failure_count(identifier)
    @redis.get(failure_key(identifier)).to_i
  end
  
  def locked?(identifier)
    @redis.exists(lock_key(identifier)) > 0
  end
  
  def remaining_lockout_time(identifier)
    ttl = @redis.ttl(lock_key(identifier))
    ttl > 0 ? ttl : 0
  end
  
  private
  
  def failure_key(identifier)
    "login_failures:#{identifier}"
  end
  
  def lock_key(identifier)
    "login_lock:#{identifier}"
  end
  
  def lock!(identifier)
    @redis.setex(lock_key(identifier), LOCKOUT_TIME.to_i, '1')
  end
end

# Controller
class SessionsController < ApplicationController
  def create
    identifier = login_identifier
    limiter    = LoginRateLimiter.new
    
    unless limiter.allow_attempt?(identifier)
      remaining = limiter.remaining_lockout_time(identifier)
      
      render json: {
        error: "บัญชีถูกล็อค กรุณารอ #{remaining / 60} นาที"
      }, status: :too_many_requests
      return
    end
    
    user = User.find_by(email: params[:email])
    
    if user&.authenticate(params[:password])
      limiter.record_success!(identifier)
      
      session[:user_id] = user.id
      redirect_to dashboard_path
    else
      limiter.record_failure!(identifier)
      
      attempts = limiter.failure_count(identifier)
      remaining_attempts = LoginRateLimiter::MAX_ATTEMPTS - attempts
      
      flash.now[:error] = if remaining_attempts > 0
        "Email หรือ Password ไม่ถูกต้อง (เหลือ #{remaining_attempts} ครั้ง)"
      else
        "บัญชีถูกล็อค กรุณารอ 30 นาที"
      end
      
      render :new, status: :unprocessable_entity
    end
  end
  
  private
  
  def login_identifier
    # ใช้ IP + Email เพื่อป้องกัน Brute Force
    "#{request.ip}:#{params[:email]&.downcase}"
  end
end

# Rack Middleware สำหรับ Rate Limiting (Rack::Attack)
gem 'rack-attack'

# config/initializers/rack_attack.rb
class Rack::Attack
  # Throttle: 10 requests per minute per IP
  throttle('req/ip', limit: 10, period: 1.minute) do |req|
    req.ip
  end
  
  # Throttle: Login Attempts
  throttle('logins/ip', limit: 5, period: 20.seconds) do |req|
    if req.path == '/session' && req.post?
      req.ip
    end
  end
  
  # Block Known Attackers
  blocklist('block bad actors') do |req|
    BlockedIP.exists?(ip: req.ip)
  end
  
  # Custom Response
  self.throttled_responder = lambda do |req|
    [
      429,
      { 'Content-Type' => 'application/json' },
      [{ error: 'Too Many Requests' }.to_json]
    ]
  end
end
```

---

*สรุปบทที่ 29: Security เป็นเรื่องที่ไม่สามารถมองข้ามได้ การใช้ Input Validation, Parameterized Queries, Secure Password Hashing, CSRF Protection และ Rate Limiting เป็นพื้นฐานที่ต้องทำ bundler-audit และ brakeman ช่วยตรวจสอบ Vulnerabilities โดยอัตโนมัติ ควรทำ Security Audit เป็นประจำ*
