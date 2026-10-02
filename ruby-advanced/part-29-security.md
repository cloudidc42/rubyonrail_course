# ตอนที่ 29: Security ใน Ruby (Steps 626-645)

## บทนำ

Security เป็นสิ่งที่ต้องคำนึงถึงตั้งแต่เริ่มพัฒนา ไม่ใช่แค่ "เพิ่มทีหลัง" บทนี้จะครอบคลุมการรักษาความปลอดภัยใน Ruby และ Rails อย่างละเอียด

---

## Step 626: Input Validation Patterns

### หลักการ Input Validation

```ruby
# Never trust user input!
# ตรวจสอบ input ทุกครั้งก่อนใช้งาน

class UserValidator
  def initialize(params)
    @params = params
    @errors = []
  end

  def valid?
    validate_name
    validate_email
    validate_age
    validate_phone
    @errors.empty?
  end

  def errors
    @errors
  end

  private

  def validate_name
    name = @params[:name].to_s.strip
    @errors << "ชื่อต้องไม่ว่าง" if name.empty?
    @errors << "ชื่อต้องไม่เกิน 100 ตัวอักษร" if name.length > 100
    # ป้องกัน XSS และ injection ด้วยการ whitelist characters
    @errors << "ชื่อมีอักขระที่ไม่อนุญาต" unless name.match?(/\A[\p{L}\s\-'.]+\z/)
  end

  def validate_email
    email = @params[:email].to_s.strip.downcase
    @errors << "Email ต้องไม่ว่าง" if email.empty?
    unless email.match?(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/)
      @errors << "รูปแบบ Email ไม่ถูกต้อง"
    end
  end

  def validate_age
    age = @params[:age]
    return @errors << "อายุต้องเป็นตัวเลข" unless age.is_a?(Integer) || age.to_s.match?(/\A\d+\z/)
    age = age.to_i
    @errors << "อายุต้องระหว่าง 0-150" unless (0..150).include?(age)
  end

  def validate_phone
    return unless @params[:phone]
    phone = @params[:phone].to_s.gsub(/[\s\-\(\)]/, '')
    @errors << "เบอร์โทรไม่ถูกต้อง" unless phone.match?(/\A(0[0-9]{9}|(\+66)[0-9]{9})\z/)
  end
end

validator = UserValidator.new({
  name: "สมชาย ใจดี",
  email: "somchai@example.com",
  age: "25",
  phone: "0812345678"
})

if validator.valid?
  puts "ข้อมูลถูกต้อง"
else
  puts validator.errors.inspect
end
```

### Whitelist vs Blacklist

```ruby
# Whitelist - ระบุสิ่งที่อนุญาต (ปลอดภัยกว่า)
ALLOWED_SORT_COLUMNS = %w[name email created_at updated_at].freeze
ALLOWED_SORT_DIRECTIONS = %w[asc desc].freeze

def safe_sort(column, direction)
  column = ALLOWED_SORT_COLUMNS.include?(column) ? column : "name"
  direction = ALLOWED_SORT_DIRECTIONS.include?(direction) ? direction : "asc"
  "#{column} #{direction}"
end

# Blacklist - ระบุสิ่งที่ห้าม (ไม่ปลอดภัย เพราะอาจขาด)
# BAD:
def unsafe_sort(column, direction)
  # อาจลืม block บาง cases
  column = "name" if column.include?(";") || column.include?("--")
  "#{column} #{direction}"  # ยังอันตราย!
end
```

---

## Step 627: SQL Injection Prevention

### SQL Injection คืออะไร?

```ruby
# DANGER - SQL Injection vulnerability!
def find_user_unsafe(email)
  User.where("email = '#{email}'")
end

# ถ้า email = "' OR '1'='1"
# SQL: SELECT * FROM users WHERE email = '' OR '1'='1'
# ได้ users ทั้งหมด!

# อันตรายยิ่งกว่า:
# email = "'; DROP TABLE users; --"
# SQL: SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
```

### วิธีป้องกัน SQL Injection

```ruby
# 1. Parameterized queries (วิธีที่ดีที่สุด)
def find_user_safe(email)
  User.where("email = ?", email)
  # หรือ
  User.where(email: email)
end

# 2. Named parameters
def search_users(name, min_age, max_age)
  User.where(
    "name LIKE :name AND age BETWEEN :min AND :max",
    name: "%#{name}%",
    min: min_age,
    max: max_age
  )
end

# 3. Arel สำหรับ complex queries
users = User.arel_table
query = users[:email].eq(email).and(users[:active].eq(true))
User.where(query)

# 4. sanitize_sql_like สำหรับ LIKE queries
def search_by_name(name)
  sanitized = ActiveRecord::Base.sanitize_sql_like(name)
  User.where("name LIKE ?", "%#{sanitized}%")
end
```

### Dynamic Column Names

```ruby
# DANGER - Dynamic column names!
def sort_users_unsafe(column)
  User.order(column)  # อันตราย ถ้า column มาจาก user input!
end

# SAFE - Whitelist columns
SORTABLE_COLUMNS = {
  "name" => :name,
  "email" => :email,
  "created_at" => :created_at
}.freeze

def sort_users_safe(column)
  sort_column = SORTABLE_COLUMNS[column] || :name
  User.order(sort_column)
end
```

---

## Step 628: XSS Prevention

### XSS คืออะไร?

Cross-Site Scripting (XSS) คือการแทรก JavaScript ที่เป็นอันตรายลงในหน้าเว็บ

```ruby
# XSS Attack example
# ถ้าผู้ใช้ส่ง: name = "<script>alert('hacked!')</script>"
# แล้วแสดงใน HTML โดยตรง จะรัน JavaScript!
```

### html_escape

```ruby
require 'cgi'

# ป้องกัน XSS ด้วย html_escape
user_input = "<script>alert('xss')</script>"
safe_output = CGI.escapeHTML(user_input)
puts safe_output
# => &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;

# ใน Rails helpers
include ActionView::Helpers::OutputSafetyHelper

user_input = "<script>alert('xss')</script>"
safe_output = html_escape(user_input)
puts safe_output
# => &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;
```

### Rails Auto-escaping

```erb
<!-- Rails auto-escapes ERB output ด้วย <%= %> -->
<!-- ปลอดภัย -->
<p><%=  @user.name %></p>

<!-- อันตราย! - ปิด auto-escape -->
<p><%= @user.name.html_safe %></p>    <!-- อย่าทำ! -->
<p><%== @user.name %></p>              <!-- อย่าทำ! (alias html_safe) -->
<p><%= raw(@user.name) %></p>          <!-- อย่าทำ! -->
```

### sanitize Helper

```ruby
# ใน Rails
require 'action_view'
include ActionView::Helpers::SanitizeHelper

html = "<p>Hello <b>World</b> <script>alert('xss')</script></p>"

# อนุญาตเฉพาะ tags ที่ระบุ
safe_html = sanitize(html, tags: %w[p b i em strong], attributes: %w[href class])
puts safe_html
# => <p>Hello <b>World</b> </p>

# ลบ HTML ทั้งหมด
plain_text = strip_tags(html)
puts plain_text
# => Hello World alert('xss')
```

### Content Security Policy

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https

  # ใช้ nonce สำหรับ inline scripts
  policy.script_src :self, :https, :unsafe_inline if Rails.env.development?
end
```

---

## Step 629: CSRF Basics

### CSRF คืออะไร?

Cross-Site Request Forgery คือการหลอกให้ผู้ใช้ส่ง request ที่ไม่ต้องการ

```ruby
# Rails CSRF Protection (built-in)
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception  # เปิด CSRF protection

  # หรือ
  protect_from_forgery with: :reset_session  # สำหรับ API
  protect_from_forgery with: :null_session   # สำหรับ JSON API
end

# ใน Form (auto-included ด้วย form_with/form_for)
# <%= form_with(model: @user) do |f| %>
#   <%= hidden_field_tag :authenticity_token, form_authenticity_token %>
# <% end %>

# ข้ามสำหรับ API controllers
class Api::V1::BaseController < ApplicationController
  skip_before_action :verify_authenticity_token
end
```

### CSRF Token Management

```ruby
# ตรวจสอบ CSRF token ด้วยตนเอง
def verify_csrf
  unless valid_authenticity_token?(session, params[:authenticity_token] || request.headers["X-CSRF-Token"])
    raise ActionController::InvalidAuthenticityToken
  end
end

# สำหรับ SPA (ใช้ cookie)
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'https://my-spa.com'
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true
  end
end
```

---

## Step 630: Secure Random Tokens

### SecureRandom

```ruby
require 'securerandom'

# UUID - unique identifier
token = SecureRandom.uuid
puts token  # => "550e8400-e29b-41d4-a716-446655440000"

# Hex string
token = SecureRandom.hex(32)    # 64 hex chars
puts token  # => "a1b2c3d4..."

# Base64 URL-safe
token = SecureRandom.urlsafe_base64(32)
puts token  # => "abc123..."

# Random bytes
bytes = SecureRandom.bytes(16)  # binary

# Random number
puts SecureRandom.random_number(1000)  # 0 - 999
```

### สร้าง Secure Token สำหรับ Applications

```ruby
# Password Reset Token
class User < ApplicationRecord
  def generate_password_reset_token!
    token = SecureRandom.urlsafe_base64(32)
    update!(
      reset_password_token: Digest::SHA256.hexdigest(token),
      reset_password_sent_at: Time.current
    )
    token  # คืน raw token (ส่งให้ user)
  end

  def reset_password_token_valid?(token, expiry: 2.hours)
    return false if reset_password_sent_at < expiry.ago
    hashed = Digest::SHA256.hexdigest(token)
    ActiveSupport::SecurityUtils.secure_compare(reset_password_token, hashed)
  end
end

# API Token
class ApiKey < ApplicationRecord
  before_create :generate_key

  private

  def generate_key
    self.token = SecureRandom.hex(32)
    self.prefix = token.first(8)  # สำหรับ lookup
  end
end
```

---

## Step 631: Password Hashing ด้วย bcrypt

### bcrypt คืออะไร?

bcrypt เป็น cryptographic hash function ที่ออกแบบมาสำหรับ passwords โดยเฉพาะ มีคุณสมบัติ:
- **Slow by design** - ป้องกัน brute force
- **Salted** - ป้องกัน rainbow table attacks
- **Adaptive** - ปรับ cost factor ได้

```ruby
# gem 'bcrypt'
require 'bcrypt'

# Hash password
password = "my_secret_password"
hashed = BCrypt::Password.create(password)
puts hashed
# => $2a$12$abc123...  (BCrypt hash)

# ตรวจสอบ password
bcrypt_password = BCrypt::Password.new(hashed)
puts bcrypt_password == password  # => true
puts bcrypt_password == "wrong"   # => false

# Cost factor (default 12)
# ยิ่งสูง ยิ่งช้า ยิ่งปลอดภัย
fast_hash = BCrypt::Password.create("password", cost: 4)   # เร็ว (สำหรับ test)
slow_hash = BCrypt::Password.create("password", cost: 14)  # ช้า (สำหรับ production)
```

### bcrypt-ruby Gem Usage

```ruby
class User
  attr_accessor :email
  attr_reader :password_digest

  def password=(plain_password)
    @password_digest = BCrypt::Password.create(plain_password)
  end

  def authenticate(plain_password)
    BCrypt::Password.new(@password_digest) == plain_password
  end
end

user = User.new
user.email = "user@example.com"
user.password = "SecurePass123!"

puts user.authenticate("SecurePass123!")  # => true
puts user.authenticate("wrongpass")       # => false
```

### has_secure_password ใน Rails

```ruby
# Migration
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.string :password_digest, null: false  # ต้องมี column นี้
      t.timestamps
    end
  end
end

# Model
class User < ApplicationRecord
  has_secure_password  # ใช้ bcrypt อัตโนมัติ

  validates :email, presence: true, uniqueness: true,
            format: { with: /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i }
  validates :password, length: { minimum: 8 }, if: :password_required?

  private

  def password_required?
    new_record? || password.present?
  end
end

# ใช้งาน
user = User.create!(email: "user@example.com", password: "SecurePass123!", password_confirmation: "SecurePass123!")

# Authenticate
user.authenticate("SecurePass123!")  # => user object (truthy)
user.authenticate("wrong")           # => false

# ใน Sessions Controller
def create
  user = User.find_by(email: params[:email])
  if user&.authenticate(params[:password])
    session[:user_id] = user.id
    redirect_to dashboard_path
  else
    flash[:alert] = "Email หรือ Password ไม่ถูกต้อง"
    render :new
  end
end
```

---

## Step 632: Storing Secrets - Environment Variables

### ทำไมต้องใช้ Environment Variables?

```ruby
# BAD - อย่า hardcode secrets ใน code!
DATABASE_PASSWORD = "my_secret_password"  # จะถูก commit ขึ้น Git!

API_KEY = "sk-abc123..."  # อันตราย!
```

### dotenv gem

```ruby
# Gemfile
gem 'dotenv-rails', groups: [:development, :test]

# .env file (ไม่ commit ขึ้น Git!)
DATABASE_URL=postgres://localhost/myapp
SECRET_KEY=abc123...
STRIPE_SECRET_KEY=sk_test_...
SENDGRID_API_KEY=SG.abc...
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=abc...

# .gitignore
.env
.env.local
.env.*.local

# ใช้งาน
database_url = ENV['DATABASE_URL']
secret_key = ENV.fetch('SECRET_KEY') { raise "SECRET_KEY ต้องตั้งค่า!" }
```

### Rails Credentials

```ruby
# Rails built-in encrypted credentials
# config/credentials.yml.enc (encrypted, ปลอดภัย commit ได้)
# config/master.key (ไม่ commit ขึ้น Git!)

# แก้ไข credentials
# EDITOR=vim rails credentials:edit

# credentials.yml.enc content:
# secret_key_base: abc123...
# database:
#   password: my_db_password
# stripe:
#   publishable_key: pk_test_...
#   secret_key: sk_test_...

# ใช้งาน
Rails.application.credentials.secret_key_base
Rails.application.credentials.dig(:stripe, :secret_key)

# Per-environment credentials
# rails credentials:edit --environment production
```

---

## Step 633: bundler-audit

### ตรวจสอบ Vulnerable Gems

```bash
# ติดตั้ง
gem install bundler-audit

# อัปเดต vulnerability database
bundle-audit update

# ตรวจสอบ Gemfile.lock
bundle-audit check

# ผลลัพธ์ตัวอย่าง:
# Name: rails
# Version: 5.2.0
# Advisory: CVE-2019-5418
# Criticality: High
# URL: https://groups.google.com/forum/#!topic/rubyonrails-security/pFRKI96Sm8Q
# Title: File Content Disclosure in Action View
# Solution: upgrade to >= 5.2.2.1
```

### ใน CI/CD Pipeline

```yaml
# .github/workflows/security.yml
name: Security

on: [push, pull_request]

jobs:
  bundle-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.2
          bundler-cache: true
      - name: Bundle Audit
        run: |
          gem install bundler-audit
          bundle-audit update
          bundle-audit check --ignore CVE-2022-XXXX  # ignore ที่จัดการแล้ว
```

### Brakeman - Static Analysis Security Scanner

```bash
# ติดตั้ง
gem install brakeman

# สแกน Rails app
brakeman

# สแกนพร้อม output formats
brakeman -o report.html  # HTML report
brakeman -o report.json  # JSON report
brakeman -A             # สแกนทุก checks

# ผลลัพธ์
# == Warnings ==
# Confidence: High
# Category: SQL Injection
# Message: Possible SQL injection
# File: app/models/user.rb
# Line: 25
```

---

## Step 634: Mass Assignment Risks

### Mass Assignment คืออะไร?

```ruby
# DANGER - Mass assignment
class User < ApplicationRecord
  # ถ้าไม่มี protection
end

# ผู้ใช้ส่ง params:
# { user: { name: "สมชาย", email: "test@test.com", admin: true } }

User.create(params[:user])  # admin: true จะถูก set!
```

### Strong Parameters

```ruby
# Rails Strong Parameters - whitelist approach
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
    if @user.update(user_params)
      redirect_to @user
    else
      render :edit
    end
  end

  private

  def user_params
    # ระบุเฉพาะ fields ที่อนุญาต
    params.require(:user).permit(:name, :email, :password, :password_confirmation)
    # ไม่อนุญาต: admin, role, balance, etc.
  end

  # Admin สามารถตั้งค่า role
  def admin_user_params
    if current_user.admin?
      params.require(:user).permit(:name, :email, :role)
    else
      user_params
    end
  end
end
```

### Nested Attributes

```ruby
class UsersController < ApplicationController
  private

  def user_params
    params.require(:user).permit(
      :name,
      :email,
      :password,
      # nested
      address_attributes: [:street, :city, :zip, :_destroy],
      # array
      tag_ids: [],
      # nested array
      posts_attributes: [:title, :body, :_destroy, :id]
    )
  end
end
```

---

## Step 635: Timing Attacks

### Timing Attack คืออะไร?

```ruby
# DANGER - Timing attack vulnerable
def authenticate(provided_token, actual_token)
  provided_token == actual_token  # String comparison short-circuits!
  # ถ้า first character ตรง จะใช้เวลานานกว่า
  # ผู้โจมตีวัดเวลาได้!
end

# SAFE - Constant-time comparison
require 'active_support/security_utils'

def authenticate_safe(provided_token, actual_token)
  ActiveSupport::SecurityUtils.secure_compare(provided_token, actual_token)
  # ใช้เวลาเท่ากันเสมอ ไม่ว่า token จะตรงหรือไม่
end

# หรือ
require 'openssl'
def secure_compare(a, b)
  return false unless a.length == b.length
  OpenSSL.fixed_length_secure_compare(a, b)
end
```

### Timing Attack ใน Password Reset

```ruby
class PasswordResetsController < ApplicationController
  def update
    # SAFE: ค้นหาด้วย token hash (ไม่ใช่ raw token)
    token_hash = Digest::SHA256.hexdigest(params[:token])
    @user = User.find_by(reset_password_token: token_hash)

    # SAFE: ตรวจสอบด้วย secure_compare
    if @user && @user.reset_password_token_valid?(params[:token])
      @user.update!(
        password: params[:password],
        reset_password_token: nil,
        reset_password_sent_at: nil
      )
      redirect_to login_path, notice: "รีเซ็ตรหัสผ่านสำเร็จ"
    else
      # IMPORTANT: ใช้เวลาเดียวกันสำหรับ invalid cases
      render :new, alert: "Token ไม่ถูกต้องหรือหมดอายุ"
    end
  end
end
```

---

## Step 636: SSL/TLS Basics

### HTTPS ใน Rails

```ruby
# config/environments/production.rb
config.force_ssl = true  # บังคับ HTTPS

# HSTS (HTTP Strict Transport Security)
config.force_ssl = true
config.ssl_options = {
  hsts: {
    expires: 1.year,
    subdomains: true,
    preload: true
  }
}
```

### SSL Certificate Verification

```ruby
require 'net/http'
require 'openssl'

# GOOD - ตรวจสอบ SSL certificate
def secure_http_request(url)
  uri = URI(url)
  http = Net::HTTP.new(uri.host, uri.port)
  http.use_ssl = true
  http.verify_mode = OpenSSL::SSL::VERIFY_PEER  # default, ดีอยู่แล้ว
  http.get(uri.path)
end

# BAD - ปิด SSL verification (อย่าทำ!)
def insecure_http_request(url)
  uri = URI(url)
  http = Net::HTTP.new(uri.host, uri.port)
  http.use_ssl = true
  http.verify_mode = OpenSSL::SSL::VERIFY_NONE  # อันตราย! MITM attacks!
  http.get(uri.path)
end
```

### Faraday ด้วย SSL

```ruby
# gem 'faraday'
require 'faraday'

# GOOD
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.adapter Faraday.default_adapter
  # SSL verification เปิดอัตโนมัติ
end

# ตั้งค่า SSL options ถ้าจำเป็น
conn = Faraday.new(url: 'https://api.example.com') do |f|
  f.ssl[:verify] = true
  f.ssl[:ca_file] = '/path/to/ca.crt'  # custom CA certificate
end
```

---

## Step 637: Secure Headers

```ruby
# gem 'secure_headers'

# config/initializers/secure_headers.rb
SecureHeaders::Configuration.default do |config|
  config.x_frame_options = "DENY"
  config.x_content_type_options = "nosniff"
  config.x_xss_protection = "1; mode=block"
  config.x_download_options = "noopen"
  config.x_permitted_cross_domain_policies = "none"
  config.referrer_policy = "origin-when-cross-origin"

  config.csp = {
    default_src: %w['self'],
    script_src: %w['self' 'nonce-abc123'],
    style_src: %w['self' https://fonts.googleapis.com],
    img_src: %w['self' data: https:],
    font_src: %w['self' https://fonts.gstatic.com],
    connect_src: %w['self' https://api.example.com],
    object_src: %w['none'],
    frame_ancestors: %w['none']
  }
end
```

---

## Step 638: Common Vulnerabilities

### Directory Traversal

```ruby
# DANGER - Directory traversal
def serve_file(filename)
  File.read("public/#{filename}")
  # filename = "../../etc/passwd" → อ่าน system files!
end

# SAFE
def serve_file_safe(filename)
  # ตรวจสอบว่าอยู่ใน allowed directory
  base_dir = Rails.root.join("public", "uploads")
  requested = base_dir.join(filename).expand_path

  unless requested.to_s.start_with?(base_dir.to_s)
    raise SecurityError, "Access denied"
  end

  File.read(requested)
end
```

### Command Injection

```ruby
# DANGER - Command injection
def resize_image_unsafe(filename, width)
  system("convert #{filename} -resize #{width}x output.jpg")
  # filename = "input.jpg; rm -rf /"  → อันตรายมาก!
end

# SAFE - ใช้ array form (ไม่ผ่าน shell)
def resize_image_safe(filename, width)
  # ตรวจสอบ filename
  raise ArgumentError, "Invalid filename" unless filename.match?(/\A[\w\-\.]+\z/)
  raise ArgumentError, "Invalid width" unless width.to_s.match?(/\A\d+\z/)

  system("convert", filename, "-resize", "#{width}x", "output.jpg")
end

# หรือใช้ shellwords
require 'shellwords'
def resize_image_shellwords(filename, width)
  safe_filename = Shellwords.escape(filename)
  safe_width = width.to_i.to_s
  `convert #{safe_filename} -resize #{safe_width}x output.jpg`
end
```

---

## แบบฝึกหัด 20 ข้อ

### ข้อที่ 1: SQL Injection Prevention
```ruby
# โจทย์: แก้ SQL injection vulnerability นี้
def find_users_unsafe(search_term)
  User.where("name = '#{search_term}'")
end

# เฉลย
def find_users_safe(search_term)
  User.where("name = ?", search_term)
  # หรือ
  User.where(name: search_term)
end

# Test
find_users_safe("'; DROP TABLE users; --")
# จะ query: WHERE name = '''; DROP TABLE users; --'
# ปลอดภัย! ไม่ execute SQL injection
```

### ข้อที่ 2: Password Hashing
```ruby
# เฉลย
require 'bcrypt'

class PasswordManager
  def self.hash_password(plain_password)
    raise ArgumentError, "Password ต้องมีอย่างน้อย 8 ตัวอักษร" if plain_password.length < 8
    BCrypt::Password.create(plain_password, cost: 12)
  end

  def self.verify_password(plain_password, hashed)
    BCrypt::Password.new(hashed) == plain_password
  end

  def self.strong_password?(password)
    return false if password.length < 8
    return false unless password.match?(/[A-Z]/)  # uppercase
    return false unless password.match?(/[a-z]/)  # lowercase
    return false unless password.match?(/\d/)      # digit
    return false unless password.match?(/[!@#$%^&*]/)  # special char
    true
  end
end

password = "MyStr0ng!Pass"
hash = PasswordManager.hash_password(password)
puts PasswordManager.verify_password(password, hash)  # => true
puts PasswordManager.strong_password?(password)        # => true
```

### ข้อที่ 3: Secure Token Generation
```ruby
# เฉลย
class TokenService
  TOKEN_EXPIRY = 24.hours

  def self.generate
    SecureRandom.urlsafe_base64(32)
  end

  def self.generate_with_expiry
    {
      token: generate,
      expires_at: TOKEN_EXPIRY.from_now
    }
  end

  def self.hash_token(raw_token)
    Digest::SHA256.hexdigest(raw_token)
  end

  def self.valid_token?(raw_token, hashed_token, expires_at)
    return false if Time.current > expires_at
    ActiveSupport::SecurityUtils.secure_compare(
      hash_token(raw_token),
      hashed_token
    )
  end
end
```

### ข้อที่ 4: Input Validation
```ruby
# เฉลย
class PostValidator
  attr_reader :errors

  def initialize(params)
    @params = params
    @errors = []
  end

  def valid?
    validate_title
    validate_body
    validate_tags
    @errors.empty?
  end

  private

  def validate_title
    title = @params[:title].to_s.strip
    @errors << "หัวข้อต้องไม่ว่าง" if title.empty?
    @errors << "หัวข้อต้องไม่เกิน 200 ตัวอักษร" if title.length > 200
  end

  def validate_body
    body = @params[:body].to_s.strip
    @errors << "เนื้อหาต้องไม่ว่าง" if body.empty?
    @errors << "เนื้อหาต้องไม่เกิน 50000 ตัวอักษร" if body.length > 50_000
  end

  def validate_tags
    tags = @params[:tags]
    return unless tags
    @errors << "Tags ต้องเป็น array" unless tags.is_a?(Array)
    @errors << "Tags ต้องไม่เกิน 10 รายการ" if tags.length > 10
    tags.each do |tag|
      @errors << "Tag '#{tag}' มีอักขระไม่ถูกต้อง" unless tag.to_s.match?(/\A[\w\-]+\z/)
    end
  end
end
```

### ข้อที่ 5: XSS Prevention
```ruby
# เฉลย
module HtmlSanitizer
  ALLOWED_TAGS = %w[p b i em strong a ul ol li br h1 h2 h3 h4 h5 h6 blockquote].freeze
  ALLOWED_ATTRS = { "a" => ["href", "title"], "p" => ["class"] }.freeze

  def self.sanitize(html)
    # ใช้ Loofah สำหรับ sanitization
    Loofah.fragment(html).scrub!(:strip).to_s
  end

  def self.sanitize_strict(html)
    ActionView::Base.full_sanitizer.sanitize(html)
  end

  def self.escape(text)
    CGI.escapeHTML(text.to_s)
  end
end

unsafe_html = "<p>Hello <b>World</b> <script>alert('xss')</script></p>"
puts HtmlSanitizer.sanitize(unsafe_html)
# => <p>Hello <b>World</b> </p>
```

### ข้อที่ 6: CSRF Token Verification
```ruby
# เฉลย - Custom CSRF implementation
class CSRFProtection
  def self.generate_token
    SecureRandom.base64(32)
  end

  def self.store_in_session(session)
    session[:csrf_token] = generate_token
    session[:csrf_token]
  end

  def self.verify(session, params_token)
    session_token = session[:csrf_token]
    return false unless session_token && params_token
    ActiveSupport::SecurityUtils.secure_compare(session_token, params_token)
  end
end
```

### ข้อที่ 7: Rate Limiting
```ruby
# เฉลย
class RateLimiter
  def initialize(redis, limit: 100, window: 60)
    @redis = redis
    @limit = limit
    @window = window
  end

  def allow?(identifier)
    key = "rate_limit:#{identifier}:#{Time.current.to_i / @window}"
    count = @redis.incr(key)
    @redis.expire(key, @window * 2) if count == 1
    count <= @limit
  end

  def remaining(identifier)
    key = "rate_limit:#{identifier}:#{Time.current.to_i / @window}"
    count = @redis.get(key).to_i
    [@limit - count, 0].max
  end
end
```

### ข้อที่ 8: Secure File Upload
```ruby
# เฉลย
class SecureFileUploader
  ALLOWED_TYPES = %w[image/jpeg image/png image/gif application/pdf].freeze
  MAX_SIZE = 10.megabytes

  def self.validate!(file)
    raise "ไม่มีไฟล์" unless file
    raise "ไฟล์ใหญ่เกินไป (max #{MAX_SIZE / 1.megabyte}MB)" if file.size > MAX_SIZE
    raise "ประเภทไฟล์ไม่อนุญาต" unless ALLOWED_TYPES.include?(file.content_type)
    validate_filename!(file.original_filename)
    validate_file_magic_bytes!(file)
  end

  def self.safe_filename(original_filename)
    extension = File.extname(original_filename).downcase
    "#{SecureRandom.hex(16)}#{extension}"
  end

  private

  def self.validate_filename!(filename)
    raise "ชื่อไฟล์ไม่ถูกต้อง" unless filename.match?(/\A[\w\-. ]+\z/)
    raise "ชื่อไฟล์ต้องไม่เริ่มด้วย ." if filename.start_with?(".")
  end

  def self.validate_file_magic_bytes!(file)
    file.rewind
    header = file.read(4)
    file.rewind

    jpeg_magic = "\xFF\xD8\xFF"
    png_magic = "\x89PNG"
    gif_magic = "GIF8"
    pdf_magic = "%PDF"

    valid = header.start_with?(jpeg_magic) ||
            header.start_with?(png_magic) ||
            header.start_with?(gif_magic) ||
            header.start_with?(pdf_magic)

    raise "ไฟล์ไม่ตรงกับ content type" unless valid
  end
end
```

### ข้อที่ 9: Environment Variables Security
```ruby
# เฉลย
class AppConfig
  REQUIRED_VARS = %w[
    DATABASE_URL
    SECRET_KEY_BASE
    REDIS_URL
  ].freeze

  OPTIONAL_VARS = {
    "LOG_LEVEL" => "info",
    "MAX_CONNECTIONS" => "10",
    "CACHE_TTL" => "3600"
  }.freeze

  def self.load!
    missing = REQUIRED_VARS.reject { |var| ENV[var] }
    if missing.any?
      raise "Missing required environment variables: #{missing.join(', ')}"
    end
    puts "Config loaded successfully"
  end

  def self.[](key)
    ENV[key] || OPTIONAL_VARS[key]
  end

  def self.database_url
    ENV.fetch('DATABASE_URL')
  end
end

# config/application.rb
# AppConfig.load! ใน initializer
```

### ข้อที่ 10: Audit Logging
```ruby
# เฉลย
class AuditLogger
  def self.log(action, user:, resource: nil, details: {})
    entry = {
      timestamp: Time.current.utc.iso8601,
      action: action,
      user_id: user&.id,
      user_email: user&.email,
      resource_type: resource&.class&.name,
      resource_id: resource&.id,
      ip_address: Current.request&.remote_ip,
      details: details
    }

    AuditLog.create!(entry)
    Rails.logger.info("[AUDIT] #{entry.to_json}")
  end
end

# ใน Controller
class UsersController < ApplicationController
  def destroy
    @user = User.find(params[:id])
    @user.destroy
    AuditLogger.log("user.deleted",
      user: current_user,
      resource: @user,
      details: { email: @user.email }
    )
    redirect_to users_path
  end
end
```

### ข้อที่ 11-20: Additional Security Exercises

**ข้อที่ 11:** Implement API Key Authentication
```ruby
# เฉลย
class ApiKeyMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)
    api_key = request.env['HTTP_X_API_KEY']

    unless valid_api_key?(api_key)
      return [401, { 'Content-Type' => 'application/json' },
              ['{"error":"Unauthorized"}']]
    end

    @app.call(env)
  end

  private

  def valid_api_key?(key)
    return false unless key
    ApiKey.find_by_token(key)&.active?
  end
end
```

**ข้อที่ 12:** Secure Session Management
```ruby
# เฉลย
class SessionManager
  SESSION_EXPIRY = 24.hours

  def self.create(user, request)
    token = SecureRandom.hex(32)
    Session.create!(
      user: user,
      token: Digest::SHA256.hexdigest(token),
      user_agent: request.user_agent,
      ip_address: request.remote_ip,
      expires_at: SESSION_EXPIRY.from_now
    )
    token  # คืน raw token
  end

  def self.find_user(raw_token)
    token_hash = Digest::SHA256.hexdigest(raw_token)
    session = Session.find_by(token: token_hash)
    return nil unless session
    return nil if session.expired?
    return nil if session.revoked?
    session.user
  end

  def self.revoke(raw_token)
    token_hash = Digest::SHA256.hexdigest(raw_token)
    Session.find_by(token: token_hash)&.update!(revoked_at: Time.current)
  end
end
```

**ข้อที่ 13:** Encryption/Decryption
```ruby
# เฉลย
require 'openssl'
require 'base64'

class Encryptor
  def initialize(key = Rails.application.credentials.encryption_key)
    @key = key
  end

  def encrypt(data)
    cipher = OpenSSL::Cipher.new('AES-256-GCM')
    cipher.encrypt
    cipher.key = Digest::SHA256.digest(@key)
    iv = cipher.random_iv

    cipher.auth_data = ""
    encrypted = cipher.update(data) + cipher.final
    tag = cipher.auth_tag

    Base64.strict_encode64(iv + tag + encrypted)
  end

  def decrypt(encoded)
    raw = Base64.strict_decode64(encoded)
    iv = raw[0, 12]
    tag = raw[12, 16]
    encrypted = raw[28..]

    cipher = OpenSSL::Cipher.new('AES-256-GCM')
    cipher.decrypt
    cipher.key = Digest::SHA256.digest(@key)
    cipher.iv = iv
    cipher.auth_tag = tag
    cipher.auth_data = ""

    cipher.update(encrypted) + cipher.final
  end
end
```

**ข้อที่ 14:** Prevent Path Traversal
```ruby
# เฉลย
class SafeFileServer
  def initialize(base_dir)
    @base_dir = Pathname.new(base_dir).expand_path
  end

  def read_file(requested_path)
    full_path = @base_dir.join(requested_path).expand_path

    unless full_path.to_s.start_with?(@base_dir.to_s + '/')
      raise SecurityError, "Access denied: path traversal detected"
    end

    raise "File not found" unless full_path.file?
    full_path.read
  end
end

server = SafeFileServer.new("/var/www/uploads")
server.read_file("images/photo.jpg")     # OK
server.read_file("../../etc/passwd")     # => SecurityError!
```

**ข้อที่ 15:** JWT Token Validation
```ruby
# เฉลย
# gem 'jwt'
require 'jwt'

class JwtService
  SECRET_KEY = Rails.application.credentials.secret_key_base.first(32)
  ALGORITHM = 'HS256'
  EXPIRY = 24.hours

  def self.encode(payload)
    payload = payload.merge(exp: EXPIRY.from_now.to_i)
    JWT.encode(payload, SECRET_KEY, ALGORITHM)
  end

  def self.decode(token)
    decoded = JWT.decode(token, SECRET_KEY, true, algorithm: ALGORITHM)
    HashWithIndifferentAccess.new(decoded.first)
  rescue JWT::ExpiredSignature
    raise "Token หมดอายุ"
  rescue JWT::DecodeError => e
    raise "Token ไม่ถูกต้อง: #{e.message}"
  end
end
```

**ข้อที่ 16:** Prevent XML/JSON Injection
```ruby
# เฉลย
class SafeJsonBuilder
  def self.build_response(user_data)
    # ใช้ JSON.generate แทน string interpolation
    {
      id: user_data[:id].to_i,
      name: user_data[:name].to_s,
      email: user_data[:email].to_s
    }.to_json
  end

  def self.parse_safely(json_string)
    JSON.parse(json_string, max_nesting: 10)
  rescue JSON::ParserError => e
    raise "JSON ไม่ถูกต้อง: #{e.message}"
  end
end
```

**ข้อที่ 17:** Two-Factor Authentication
```ruby
# เฉลย
# gem 'rotp'
require 'rotp'

class TotpService
  def self.generate_secret
    ROTP::Base32.random
  end

  def self.provisioning_uri(user, secret)
    totp = ROTP::TOTP.new(secret, issuer: "MyApp")
    totp.provisioning_uri(user.email)
  end

  def self.verify(secret, token)
    totp = ROTP::TOTP.new(secret)
    totp.verify(token, drift_behind: 15, drift_ahead: 15)
  end
end
```

**ข้อที่ 18:** IP Whitelist Middleware
```ruby
# เฉลย
class IpWhitelistMiddleware
  ALLOWED_IPS = Set.new(ENV.fetch('ALLOWED_IPS', '127.0.0.1').split(',').map(&:strip)).freeze

  def initialize(app, paths: ['/admin'])
    @app = app
    @paths = paths
  end

  def call(env)
    request = Rack::Request.new(env)

    if @paths.any? { |p| request.path.start_with?(p) }
      ip = request.ip
      unless ALLOWED_IPS.include?(ip)
        return [403, { 'Content-Type' => 'text/plain' }, ['Access Denied']]
      end
    end

    @app.call(env)
  end
end
```

**ข้อที่ 19:** Secure Random Password Generator
```ruby
# เฉลย
class PasswordGenerator
  LOWERCASE = ('a'..'z').to_a
  UPPERCASE = ('A'..'Z').to_a
  DIGITS = ('0'..'9').to_a
  SPECIAL = %w[! @ # $ % ^ & * ( ) _ + = { } [ ] | ; : , . ?].freeze

  def self.generate(length: 16)
    raise ArgumentError, "Length ต้องอย่างน้อย 12" if length < 12

    # ต้องมีอย่างน้อย 1 ของแต่ละ category
    password_chars = [
      LOWERCASE.sample,
      UPPERCASE.sample,
      DIGITS.sample,
      SPECIAL.sample
    ]

    # เติมจนครบ length
    all_chars = LOWERCASE + UPPERCASE + DIGITS + SPECIAL
    (length - 4).times { password_chars << all_chars.sample }

    # สับตำแหน่ง (Fisher-Yates shuffle)
    password_chars.each_index do |i|
      j = i + SecureRandom.random_number(password_chars.length - i)
      password_chars[i], password_chars[j] = password_chars[j], password_chars[i]
    end

    password_chars.join
  end
end

puts PasswordGenerator.generate         # => "xK3!mP9@qL7#nR2$"
puts PasswordGenerator.generate(length: 20)
```

**ข้อที่ 20:** Security Headers Checker
```ruby
# เฉลย - ตรวจสอบ security headers ของ app ตัวเอง
require 'net/http'

class SecurityHeadersChecker
  RECOMMENDED_HEADERS = {
    'X-Content-Type-Options' => 'nosniff',
    'X-Frame-Options' => ['DENY', 'SAMEORIGIN'],
    'X-XSS-Protection' => '1; mode=block',
    'Strict-Transport-Security' => /max-age=\d+/,
    'Content-Security-Policy' => /.+/,
    'Referrer-Policy' => /.+/
  }.freeze

  def self.check(url)
    uri = URI(url)
    response = Net::HTTP.get_response(uri)
    results = {}

    RECOMMENDED_HEADERS.each do |header, expected|
      value = response[header.downcase]
      results[header] = {
        present: !value.nil?,
        value: value,
        ok: check_value(value, expected)
      }
    end

    results
  end

  def self.report(url)
    results = check(url)
    puts "Security Headers Report for #{url}"
    puts "=" * 50
    results.each do |header, data|
      status = data[:ok] ? "✓" : "✗"
      puts "#{status} #{header}: #{data[:value] || 'MISSING'}"
    end
  end

  private

  def self.check_value(value, expected)
    return false unless value
    case expected
    when Array then expected.any? { |e| value.include?(e) }
    when Regexp then expected.match?(value)
    when String then value.include?(expected)
    end
  end
end
```

---

## สรุป

Security ใน Ruby/Rails:

1. **Input Validation** - ตรวจสอบ input ทุกอย่างจาก user
2. **SQL Injection** - ใช้ parameterized queries เสมอ
3. **XSS** - Rails auto-escapes แต่ระวัง html_safe
4. **Password Security** - bcrypt เสมอ, ห้าม MD5/SHA1
5. **Secrets Management** - Environment variables หรือ Rails credentials
6. **Dependency Security** - bundler-audit และ Brakeman สม่ำเสมอ
7. **HTTPS** - บังคับใช้ใน production เสมอ

---

*ต่อไป: ตอนที่ 30 - Best Practices และ Code Quality*
