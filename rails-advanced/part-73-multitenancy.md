# Part 73: Multi-tenancy

## Steps 1581-1600

---

## Step 1581: Multi-tenancy Concepts

Multi-tenancy คือ architecture ที่ให้ application เดียวรองรับ organizations/tenants หลายราย โดยแต่ละ tenant มีข้อมูลแยกจากกัน

### ประเภทของ Multi-tenancy

```
1. Schema-based (PostgreSQL)
   - แต่ละ tenant มี schema แยก
   - แยกข้อมูลชัดเจนที่สุด
   - จัดการยากกว่า

2. Row-based (Shared Tables)
   - ทุก tenant ใช้ตารางเดียวกัน
   - แยกด้วย tenant_id column
   - ง่ายต่อการ implement
   - ต้อง query filtering ทุกครั้ง

3. Database-based
   - แต่ละ tenant มี database แยก
   - แยกชัดเจนที่สุด
   - ราคาแพงและจัดการยาก
```

---

## Step 1582: Row-based Multi-tenancy

### ตั้งค่า Tenant Model

```ruby
# migration
class CreateTenants < ActiveRecord::Migration[7.0]
  def change
    create_table :tenants do |t|
      t.string :name, null: false
      t.string :subdomain, null: false, index: { unique: true }
      t.string :plan, default: 'free'
      t.boolean :active, default: true
      t.jsonb :settings, default: {}
      t.datetime :trial_ends_at
      t.timestamps
    end
  end
end

# เพิ่ม tenant_id ใน models ต่างๆ
class AddTenantToUsers < ActiveRecord::Migration[7.0]
  def change
    add_reference :users, :tenant, foreign_key: true, null: true
    add_reference :products, :tenant, foreign_key: true, null: true
    add_reference :orders, :tenant, foreign_key: true, null: true
    add_reference :customers, :tenant, foreign_key: true, null: true
    
    # Index สำคัญมาก
    add_index :users, :tenant_id
    add_index :products, :tenant_id
    add_index :orders, :tenant_id
  end
end

# app/models/tenant.rb
class Tenant < ApplicationRecord
  has_many :users, dependent: :destroy
  has_many :products, dependent: :destroy
  has_many :orders, dependent: :destroy
  
  validates :name, presence: true
  validates :subdomain, presence: true, 
            uniqueness: { case_sensitive: false },
            format: { with: /\A[a-z0-9-]+\z/, message: 'ต้องเป็นตัวพิมพ์เล็กและตัวเลขเท่านั้น' }
  
  def trial_active?
    trial_ends_at.present? && trial_ends_at > Time.current
  end
  
  def active_subscription?
    active? && (trial_active? || plan != 'free')
  end
end
```

### Current Tenant (Thread-safe)

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :tenant
  attribute :user
  attribute :request_id
  attribute :ip_address
  attribute :user_agent
  
  def tenant=(tenant)
    super
    # ตั้งค่า default scope สำหรับ tenant
    self.class.tenant_models.each do |model|
      model.current_tenant = tenant
    end
  end
  
  def self.tenant_models
    @tenant_models ||= []
  end
  
  def self.register_tenant_model(model)
    @tenant_models ||= []
    @tenant_models << model
  end
end

# Concern สำหรับ tenant-scoped models
module TenantScoped
  extend ActiveSupport::Concern
  
  included do
    belongs_to :tenant
    
    # Default scope ใช้ current tenant
    default_scope do
      if Current.tenant
        where(tenant: Current.tenant)
      else
        all
      end
    end
    
    # Register กับ Current
    Current.register_tenant_model(self)
    
    validates :tenant, presence: true
  end
  
  class_methods do
    attr_accessor :current_tenant
    
    def for_tenant(tenant)
      unscoped.where(tenant: tenant)
    end
    
    def across_tenants
      unscoped
    end
  end
end

# ใช้ใน Models
class User < ApplicationRecord
  include TenantScoped
end

class Product < ApplicationRecord
  include TenantScoped
end

class Order < ApplicationRecord
  include TenantScoped
end
```

### Middleware สำหรับ Tenant Resolution

```ruby
# app/middlewares/tenant_resolver.rb
class TenantResolver
  def initialize(app)
    @app = app
  end
  
  def call(env)
    request = ActionDispatch::Request.new(env)
    
    tenant = resolve_tenant(request)
    
    if tenant
      Current.tenant = tenant
      @app.call(env)
    else
      render_not_found
    end
  ensure
    Current.tenant = nil
  end
  
  private
  
  def resolve_tenant(request)
    # Subdomain-based
    subdomain = extract_subdomain(request.host)
    return Tenant.find_by(subdomain: subdomain, active: true) if subdomain
    
    # Header-based (สำหรับ API)
    if (tenant_id = request.headers['X-Tenant-ID'])
      return Tenant.find_by(id: tenant_id, active: true)
    end
    
    nil
  end
  
  def extract_subdomain(host)
    parts = host.split('.')
    # myapp.example.com -> myapp
    # app.example.com -> nil (main app)
    return nil if parts.length < 3
    return nil if parts.first == 'www'
    
    parts.first
  end
  
  def render_not_found
    [404, { 'Content-Type' => 'application/json' }, ['{"error": "Tenant not found"}']]
  end
end

# config/application.rb
config.middleware.use TenantResolver
```

### ApplicationController

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_current_tenant
  before_action :authenticate_user!
  before_action :check_tenant_access
  
  private
  
  def set_current_tenant
    Current.tenant = current_tenant
  end
  
  def current_tenant
    @current_tenant ||= resolve_tenant_from_request
  end
  
  def resolve_tenant_from_request
    subdomain = request.subdomain
    return nil if subdomain.blank? || subdomain == 'www'
    
    Tenant.active.find_by(subdomain: subdomain)
  end
  
  def check_tenant_access
    return if current_tenant.nil?
    
    unless current_tenant.active_subscription?
      redirect_to subscription_expired_path
    end
  end
  
  helper_method :current_tenant
end
```

---

## Step 1583: ActsAsTenant Gem

```ruby
# Gemfile
gem 'acts_as_tenant'

# app/models/tenant.rb
class Tenant < ApplicationRecord
  # ตามเดิม
end

# app/models/user.rb
class User < ApplicationRecord
  acts_as_tenant :tenant
  
  # acts_as_tenant จะ:
  # 1. เพิ่ม default_scope โดยอัตโนมัติ
  # 2. Validate tenant presence
  # 3. ตรวจสอบ tenant ใน associations
end

# ApplicationController
class ApplicationController < ActionController::Base
  set_current_tenant_through_filter
  before_action :set_tenant
  
  def set_tenant
    # หา tenant จาก subdomain
    current_tenant = Tenant.find_by(subdomain: request.subdomain)
    set_current_tenant(current_tenant)
  end
  
  # หรือ
  def set_tenant_by_header
    tenant = Tenant.find_by(
      api_key: request.headers['X-API-Key']
    )
    set_current_tenant(tenant) if tenant
  end
end

# การใช้งาน
ActsAsTenant.with_tenant(tenant) do
  # code ใน block นี้จะ scope ไปยัง tenant นั้น
  User.all  # SELECT * FROM users WHERE tenant_id = ?
end

# ระวัง! ข้ามทุก tenant
ActsAsTenant.without_tenant do
  User.all  # SELECT * FROM users (ทุก tenant!)
end
```

---

## Step 1584: Schema-based Multi-tenancy

```ruby
# Gemfile
gem 'apartment'

# config/initializers/apartment.rb
Apartment.configure do |config|
  # ชื่อ column ที่เก็บ tenant name
  config.tenant_names = -> { Tenant.pluck(:subdomain) }
  
  # Tables ที่ share กัน (ไม่แยก schema)
  config.excluded_models = %w[Tenant Plan AdminUser]
  
  # ใช้ PostgreSQL schema
  config.use_schemas = true
  
  # สร้าง seed data ในแต่ละ tenant
  config.seed_after_create = true
  
  # Middleware
  config.middleware = true
end

# config/application.rb
require 'apartment/elevators/subdomain'

config.middleware.use Apartment::Elevators::Subdomain

# สร้าง tenant ใหม่
Apartment::Tenant.create('mycompany')

# Switch tenant
Apartment::Tenant.switch('mycompany') do
  User.count  # ใน schema ของ mycompany
end

# Current tenant
Apartment::Tenant.current  # => 'mycompany'

# Drop tenant (ระวัง!)
Apartment::Tenant.drop('mycompany')
```

---

## Step 1585: Subdomain Routing

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Main app routes
  root to: 'home#index'
  
  # Tenant-specific routes
  constraints subdomain: /^(?!www)(?!api)/ do
    # ทุก subdomain ยกเว้น www และ api
    scope module: 'tenant' do
      root to: 'dashboard#index'
      resources :products
      resources :orders
      resources :customers
    end
  end
  
  # API routes
  constraints subdomain: 'api' do
    namespace :api, path: '' do
      namespace :v1 do
        resources :products
        resources :orders
      end
    end
  end
end

# สร้าง URL helper ที่รองรับ subdomain
module TenantUrlHelper
  def tenant_root_url(tenant, options = {})
    root_url(subdomain: tenant.subdomain, **options)
  end
  
  def product_url_for_tenant(product, tenant, options = {})
    product_url(product, subdomain: tenant.subdomain, **options)
  end
end
```

### Subdomain development (localhost)

```ruby
# config/environments/development.rb
Rails.application.configure do
  # ตั้งค่า subdomain ใน development
  # ใช้ lvh.me (mirrors to localhost) หรือ .test TLD
  
  config.action_mailer.default_url_options = {
    host: 'lvh.me',
    port: 3000
  }
end

# hosts.json (เพิ่มใน /etc/hosts)
# 127.0.0.1 mycompany.lvh.me
# 127.0.0.1 acme.lvh.me
# 127.0.0.1 lvh.me

# หรือใช้ Pow ใน macOS
# หรือใช้ dnsmasq
```

---

## Step 1586: Tenant Switching

```ruby
# app/services/tenant_switch_service.rb
class TenantSwitchService
  def self.switch_to(tenant_id:, user:)
    new(tenant_id: tenant_id, user: user).switch
  end
  
  def initialize(tenant_id:, user:)
    @tenant = Tenant.find(tenant_id)
    @user = user
  end
  
  def switch
    validate_access!
    
    # บันทึก tenant ใน session
    # (จัดการโดย Controller หลังจาก return)
    @tenant
  end
  
  private
  
  def validate_access!
    # ตรวจสอบว่า user มีสิทธิ์เข้าถึง tenant นี้
    unless @user.tenants.include?(@tenant)
      raise AuthorizationError, "ไม่มีสิทธิ์เข้าถึง #{@tenant.name}"
    end
    
    unless @tenant.active?
      raise TenantInactiveError, "#{@tenant.name} ถูกระงับการใช้งาน"
    end
  end
end

# Admin: สลับ tenant โดย admin
class Admin::TenantImpersonationController < ApplicationController
  before_action :require_superadmin!
  
  def create
    tenant = Tenant.find(params[:tenant_id])
    session[:admin_impersonating_tenant] = current_tenant.id
    session[:tenant_override] = tenant.subdomain
    
    redirect_to root_url(subdomain: tenant.subdomain), 
      notice: "กำลังดู app ในฐานะ tenant: #{tenant.name}"
  end
  
  def destroy
    original_tenant_id = session.delete(:admin_impersonating_tenant)
    session.delete(:tenant_override)
    
    redirect_to admin_tenants_path
  end
end
```

---

## Step 1587: Shared vs Isolated Data

```ruby
# ข้อมูลที่ share กัน (Global)
class Plan < ApplicationRecord
  # Plans สำหรับทุก tenant
  has_many :tenants
  
  PLANS = {
    free:       { price: 0,    users: 3,    storage: 1 },
    starter:    { price: 299,  users: 10,   storage: 10 },
    pro:        { price: 799,  users: 50,   storage: 100 },
    enterprise: { price: 1999, users: -1,   storage: -1 }
  }.freeze
end

class Category < ApplicationRecord
  # Global categories ที่ทุก tenant ใช้ร่วมกัน
  scope :global, -> { where(tenant_id: nil) }
end

# ข้อมูลที่แยกตาม tenant (Isolated)
class Product < ApplicationRecord
  acts_as_tenant :tenant  # แยกตาม tenant
  
  belongs_to :category  # ถ้า category เป็น global ต้องระวัง
  
  # Custom scope ที่รวม global และ tenant-specific
  scope :available_categories, -> {
    Category.where(tenant: [Current.tenant, nil])
  }
end

# Hybrid: บางข้อมูล share บางข้อมูลแยก
class Template < ApplicationRecord
  belongs_to :tenant, optional: true
  
  scope :available_for, ->(tenant) {
    where(tenant: [tenant, nil])
  }
  
  def global?
    tenant_id.nil?
  end
  
  def tenant_specific?
    !global?
  end
end
```

---

## Step 1588: Testing Multi-tenant Apps

```ruby
# spec/support/tenant_helpers.rb
module TenantHelpers
  def with_tenant(tenant)
    old_tenant = Current.tenant
    Current.tenant = tenant
    yield
  ensure
    Current.tenant = old_tenant
  end
  
  def create_tenant_with_user(tenant_attrs = {}, user_attrs = {})
    tenant = create(:tenant, tenant_attrs)
    user = with_tenant(tenant) { create(:user, user_attrs.merge(tenant: tenant)) }
    [tenant, user]
  end
end

RSpec.configure do |config|
  config.include TenantHelpers
end

# spec/models/product_spec.rb
RSpec.describe Product do
  let(:tenant1) { create(:tenant, subdomain: 'tenant1') }
  let(:tenant2) { create(:tenant, subdomain: 'tenant2') }
  
  describe 'tenant isolation' do
    before do
      with_tenant(tenant1) { create_list(:product, 3) }
      with_tenant(tenant2) { create_list(:product, 2) }
    end
    
    it 'only shows products for current tenant' do
      with_tenant(tenant1) do
        expect(Product.count).to eq(3)
      end
      
      with_tenant(tenant2) do
        expect(Product.count).to eq(2)
      end
    end
    
    it 'cannot access other tenant products' do
      tenant1_product = with_tenant(tenant1) { Product.first }
      
      with_tenant(tenant2) do
        expect(Product.find_by(id: tenant1_product.id)).to be_nil
      end
    end
  end
  
  describe 'cross-tenant prevention' do
    it 'raises error when assigning wrong tenant' do
      product = with_tenant(tenant1) { create(:product) }
      
      expect {
        with_tenant(tenant2) do
          product.update!(name: 'Hacked product')
        end
      }.to raise_error(ActiveRecord::RecordNotFound)
    end
  end
end

# spec/requests/api/v1/products_spec.rb
RSpec.describe 'Products API' do
  let(:tenant1) { create(:tenant, subdomain: 'shop1') }
  let(:tenant2) { create(:tenant, subdomain: 'shop2') }
  
  before do
    with_tenant(tenant1) { create_list(:product, 5) }
    with_tenant(tenant2) { create_list(:product, 3) }
  end
  
  it 'returns only tenant1 products' do
    get '/api/v1/products',
      headers: { 
        'Host' => 'shop1.example.com',
        'Authorization' => auth_token(tenant1)
      }
    
    expect(json_body['products'].length).to eq(5)
  end
  
  it 'returns only tenant2 products' do
    get '/api/v1/products',
      headers: {
        'Host' => 'shop2.example.com',
        'Authorization' => auth_token(tenant2)
      }
    
    expect(json_body['products'].length).to eq(3)
  end
end
```

---

## Step 1589-1600: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: Complete Tenant Setup

```ruby
# สร้าง complete multi-tenant SaaS app

# app/models/tenant.rb
class Tenant < ApplicationRecord
  has_many :users, dependent: :destroy
  has_many :memberships, dependent: :destroy
  
  validates :name, presence: true
  validates :subdomain, presence: true, uniqueness: true,
            format: { with: /\A[a-z0-9][a-z0-9-]{1,60}[a-z0-9]\z/ }
  
  before_validation :normalize_subdomain
  after_create :setup_default_data
  
  def self.resolve_from_request(request)
    subdomain = request.subdomain
    return nil if subdomain.blank? || %w[www api admin].include?(subdomain)
    
    find_by(subdomain: subdomain, active: true)
  end
  
  private
  
  def normalize_subdomain
    self.subdomain = subdomain&.downcase&.strip
  end
  
  def setup_default_data
    # สร้าง default settings, categories, etc.
    DefaultTenantSetup.call(tenant: self)
  end
end

# app/services/default_tenant_setup.rb
class DefaultTenantSetup
  def self.call(tenant:)
    ActsAsTenant.with_tenant(tenant) do
      # สร้าง default categories
      Category.create!([
        { name: 'สินค้าทั่วไป' },
        { name: 'โปรโมชั่น' }
      ])
      
      # สร้าง default settings
      TenantSetting.create!(
        currency: 'THB',
        timezone: 'Bangkok',
        locale: 'th',
        tax_rate: 7
      )
    end
  end
end
```

### แบบฝึกหัดที่ 2: Tenant Onboarding

```ruby
# app/services/tenant_onboarding.rb
class TenantOnboarding
  def self.call(**args)
    new(**args).call
  end
  
  def initialize(name:, subdomain:, owner_name:, owner_email:, plan: 'free')
    @name = name
    @subdomain = subdomain
    @owner_name = owner_name
    @owner_email = owner_email
    @plan = plan
  end
  
  def call
    ActiveRecord::Base.transaction do
      @tenant = Tenant.create!(
        name: @name,
        subdomain: @subdomain,
        plan: @plan,
        trial_ends_at: 14.days.from_now
      )
      
      ActsAsTenant.with_tenant(@tenant) do
        @owner = User.create!(
          name: @owner_name,
          email: @owner_email,
          password: SecureRandom.hex(16),
          role: 'owner',
          tenant: @tenant
        )
      end
      
      TenantMailer.welcome(@tenant, @owner).deliver_later
    end
    
    { tenant: @tenant, owner: @owner }
  end
end

# Controller
class TenantsController < ApplicationController
  skip_before_action :authenticate_user!, only: [:new, :create]
  skip_before_action :set_current_tenant, only: [:new, :create]
  
  def new
    @tenant = Tenant.new
  end
  
  def create
    result = TenantOnboarding.call(
      name: params[:company_name],
      subdomain: params[:subdomain],
      owner_name: params[:name],
      owner_email: params[:email]
    )
    
    if result[:tenant].persisted?
      redirect_to root_url(subdomain: result[:tenant].subdomain),
        notice: "ยินดีต้อนรับสู่ #{result[:tenant].name}!"
    else
      @tenant = result[:tenant]
      render :new
    end
  rescue => e
    flash.now[:alert] = e.message
    render :new
  end
end
```

### แบบฝึกหัดที่ 3: Tenant Resource Limits

```ruby
# app/models/concerns/tenant_resource_limited.rb
module TenantResourceLimited
  extend ActiveSupport::Concern
  
  included do
    before_create :check_resource_limit
  end
  
  class_methods do
    def resource_limit_key
      name.pluralize.downcase.to_sym
    end
  end
  
  private
  
  def check_resource_limit
    return unless Current.tenant
    
    limit = Current.tenant.plan_limits[self.class.resource_limit_key]
    return if limit == -1  # Unlimited
    
    current_count = self.class.for_tenant(Current.tenant).count
    
    if current_count >= limit
      errors.add(:base, "ถึงขีดจำกัด #{self.class.name} สำหรับ plan นี้ (#{limit})")
      throw(:abort)
    end
  end
end

# app/models/tenant.rb
class Tenant < ApplicationRecord
  PLAN_LIMITS = {
    'free'       => { users: 3, products: 100, orders: -1, storage_mb: 500 },
    'starter'    => { users: 10, products: 1000, orders: -1, storage_mb: 5000 },
    'pro'        => { users: 50, products: -1, orders: -1, storage_mb: 50000 },
    'enterprise' => { users: -1, products: -1, orders: -1, storage_mb: -1 }
  }.freeze
  
  def plan_limits
    PLAN_LIMITS[plan] || PLAN_LIMITS['free']
  end
  
  def within_limit?(resource, additional: 1)
    limit = plan_limits[resource]
    return true if limit == -1
    
    current = ActsAsTenant.with_tenant(self) { resource.to_s.pluralize.classify.constantize.count }
    current + additional <= limit
  end
  
  def usage_percentage(resource)
    limit = plan_limits[resource]
    return 0 if limit == -1
    
    current = ActsAsTenant.with_tenant(self) { resource.to_s.pluralize.classify.constantize.count }
    (current.to_f / limit * 100).round(1)
  end
end
```

### แบบฝึกหัดที่ 4: Tenant Dashboard Analytics

```ruby
# app/controllers/tenant/analytics_controller.rb
class Tenant::AnalyticsController < ApplicationController
  def index
    @analytics = TenantAnalytics.new(Current.tenant)
    
    @stats = {
      users: @analytics.user_stats,
      revenue: @analytics.revenue_stats,
      products: @analytics.product_stats,
      plan_usage: Current.tenant.plan_usage_summary
    }
  end
end

# app/services/tenant_analytics.rb
class TenantAnalytics
  def initialize(tenant)
    @tenant = tenant
  end
  
  def user_stats
    ActsAsTenant.with_tenant(@tenant) do
      {
        total: User.count,
        active: User.where('last_sign_in_at > ?', 30.days.ago).count,
        new_this_month: User.where('created_at >= ?', Date.today.beginning_of_month).count,
        limit: @tenant.plan_limits[:users]
      }
    end
  end
  
  def revenue_stats
    ActsAsTenant.with_tenant(@tenant) do
      {
        this_month: Order.paid.where('paid_at >= ?', Date.today.beginning_of_month).sum(:total),
        last_month: Order.paid.where(
          paid_at: 1.month.ago.beginning_of_month..1.month.ago.end_of_month
        ).sum(:total),
        total: Order.paid.sum(:total)
      }
    end
  end
  
  def product_stats
    ActsAsTenant.with_tenant(@tenant) do
      {
        total: Product.count,
        active: Product.where(active: true).count,
        out_of_stock: Product.where(stock: 0).count,
        limit: @tenant.plan_limits[:products]
      }
    end
  end
end
```

### แบบฝึกหัดที่ 5: Cross-Tenant Admin

```ruby
# app/controllers/super_admin/tenants_controller.rb
class SuperAdmin::TenantsController < SuperAdmin::BaseController
  def index
    @tenants = Tenant.all.order(created_at: :desc).page(params[:page])
    
    # Cross-tenant stats
    @stats = {
      total_tenants: Tenant.count,
      active_tenants: Tenant.active.count,
      revenue_all: ActsAsTenant.without_tenant { Order.paid.sum(:total) },
      users_all: ActsAsTenant.without_tenant { User.count }
    }
  end
  
  def show
    @tenant = Tenant.find(params[:id])
    
    # Stats สำหรับ tenant นี้
    ActsAsTenant.with_tenant(@tenant) do
      @tenant_stats = {
        users: User.count,
        products: Product.count,
        orders: Order.count,
        revenue: Order.paid.sum(:total)
      }
    end
  end
  
  def suspend
    @tenant = Tenant.find(params[:id])
    @tenant.update!(active: false)
    
    # ส่งแจ้งเตือน
    TenantMailer.suspended(@tenant).deliver_later
    
    redirect_to super_admin_tenants_path, notice: "ระงับ #{@tenant.name} สำเร็จ"
  end
  
  def reinstate
    @tenant = Tenant.find(params[:id])
    @tenant.update!(active: true)
    
    redirect_to super_admin_tenants_path, notice: "เปิดใช้งาน #{@tenant.name} สำเร็จ"
  end
end
```

---

**จบ Part 73: Multi-tenancy**

*ในส่วนถัดไป Part 74 เราจะเรียนรู้เกี่ยวกับ Analytics and Reporting*
