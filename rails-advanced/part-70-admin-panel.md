# Part 70: Admin Panels

## Steps 1521-1540

---

## Step 1521: ActiveAdmin Gem (Complete Setup)

ActiveAdmin เป็น gem ยอดนิยมสำหรับสร้าง admin interface อย่างรวดเร็ว

### การติดตั้ง

```ruby
# Gemfile
gem 'activeadmin'
gem 'devise'           # สำหรับ authentication
gem 'cancancan'        # สำหรับ authorization (optional)
gem 'ransack'          # search (included กับ activeadmin)

# Terminal
bundle install
rails generate active_admin:install
rails db:migrate

# สร้าง admin user
rails generate active_admin:install --use-devise

# หรือสร้าง admin resource แยก
rails generate active_admin:install AdminUser
```

### Configuration

```ruby
# config/initializers/active_admin.rb
ActiveAdmin.setup do |config|
  # Site title
  config.site_title = 'MyApp Admin'
  config.site_title_link = '/'
  
  # Authentication
  config.authentication_method = :authenticate_admin_user!
  config.current_user_method    = :current_admin_user
  config.logout_link_path       = :destroy_admin_user_session_path
  config.logout_link_method     = :delete
  
  # Root
  config.root_to = 'dashboard#index'
  
  # Localize dates
  config.localize_format = :long
  
  # Comments
  config.comments_menu = { parent: 'Admin', priority: 1 }
  
  # Pagination
  config.default_per_page = 30
  
  # Filters
  config.maximum_association_filter_arity = :unlimited
end
```

### สร้าง Admin Resources

```ruby
# app/admin/users.rb
ActiveAdmin.register User do
  # Permit parameters
  permit_params :name, :email, :role, :active, :phone
  
  # Menu
  menu priority: 1, label: 'ผู้ใช้งาน', icon: 'people'
  
  # Filters
  filter :name, label: 'ชื่อ'
  filter :email
  filter :role, as: :select, collection: User.roles
  filter :active, label: 'สถานะ'
  filter :created_at, label: 'วันที่สมัคร'
  
  # Index table
  index do
    selectable_column
    id_column
    
    column 'ชื่อ', :name do |user|
      link_to user.name, admin_user_path(user)
    end
    
    column 'อีเมล', :email
    column 'บทบาท', :role do |user|
      status_tag user.role, class: user.admin? ? 'yes' : 'no'
    end
    
    column 'สถานะ' do |user|
      status_tag user.active? ? 'Active' : 'Inactive',
        class: user.active? ? 'green' : 'red'
    end
    
    column 'วันที่สมัคร', :created_at
    column 'ยอดซื้อรวม' do |user|
      number_to_currency(user.orders.paid.sum(:total), unit: '฿')
    end
    
    actions defaults: true do |user|
      unless user == current_admin_user
        item 'Stop', stop_admin_user_path(user), 
          method: :put, 
          data: { confirm: 'ต้องการระงับ user นี้?' }
      end
    end
  end
  
  # Show page
  show do
    tabs do
      tab 'ข้อมูลทั่วไป' do
        attributes_table do
          row :id
          row :name
          row :email
          row :role
          row :active
          row :created_at
          row :last_sign_in_at
        end
      end
      
      tab "Orders (#{resource.orders.count})" do
        table_for resource.orders.recent.limit(20) do
          column :number
          column :status do |o|
            status_tag o.status
          end
          column :total do |o|
            number_to_currency(o.total, unit: '฿')
          end
          column :created_at
          column '' do |o|
            link_to 'View', admin_order_path(o)
          end
        end
      end
      
      tab 'Profile' do
        attributes_table_for resource.profile do
          row :bio
          row :phone
          row :avatar do |p|
            image_tag p.avatar_url, height: 100 if p.avatar_url
          end
        end
      end
    end
  end
  
  # Form
  form do |f|
    f.inputs 'ข้อมูลผู้ใช้' do
      f.input :name, label: 'ชื่อ'
      f.input :email, label: 'อีเมล'
      f.input :role, as: :select, collection: User.roles.map { |r| [r.humanize, r] }
      f.input :active, label: 'สถานะ'
      f.input :phone, label: 'โทรศัพท์'
    end
    
    f.actions
  end
  
  # Custom actions
  member_action :stop, method: :put do
    resource.update!(active: false)
    redirect_to admin_users_path, notice: 'ระงับ user สำเร็จ'
  end
  
  member_action :activate, method: :put do
    resource.update!(active: true)
    redirect_to admin_users_path, notice: 'เปิดใช้งาน user สำเร็จ'
  end
  
  # Batch actions
  batch_action :activate do |ids|
    User.where(id: ids).update_all(active: true)
    redirect_to collection_path, notice: "เปิดใช้งาน #{ids.count} users สำเร็จ"
  end
  
  batch_action :deactivate do |ids|
    User.where(id: ids).update_all(active: false)
    redirect_to collection_path, notice: "ระงับ #{ids.count} users สำเร็จ"
  end
  
  # Scopes
  scope :all, default: true
  scope :active
  scope :inactive
  scope 'Admins', :admin
  scope 'New This Month' do |users|
    users.where('created_at > ?', 1.month.ago)
  end
  
  # Controller customization
  controller do
    def scoped_collection
      super.includes(:profile, :orders)
    end
  end
end
```

### Dashboard

```ruby
# app/admin/dashboard.rb
ActiveAdmin.register_page 'Dashboard' do
  menu priority: 1, label: proc { I18n.t('active_admin.dashboard') }
  
  content title: proc { I18n.t('active_admin.dashboard') } do
    # Stats row
    div class: 'dashboard-stats' do
      columns do
        column do
          panel 'ผู้ใช้งาน' do
            div class: 'stat-number' do
              User.count
            end
            div class: 'stat-label' do
              "#{User.where('created_at > ?', 30.days.ago).count} ใหม่ใน 30 วัน"
            end
          end
        end
        
        column do
          panel 'Orders วันนี้' do
            div class: 'stat-number' do
              Order.where('created_at >= ?', Date.today.beginning_of_day).count
            end
          end
        end
        
        column do
          panel 'รายได้วันนี้' do
            div class: 'stat-number' do
              number_to_currency(
                Order.paid.where('paid_at >= ?', Date.today.beginning_of_day).sum(:total),
                unit: '฿'
              )
            end
          end
        end
        
        column do
          panel 'รายได้เดือนนี้' do
            div class: 'stat-number' do
              number_to_currency(
                Order.paid.where('paid_at >= ?', Date.today.beginning_of_month).sum(:total),
                unit: '฿'
              )
            end
          end
        end
      end
    end
    
    # Recent Orders
    columns do
      column do
        panel 'Orders ล่าสุด' do
          table_for Order.recent.limit(10).includes(:user) do
            column('Order') { |o| link_to o.number, admin_order_path(o) }
            column('ลูกค้า') { |o| o.user.name }
            column(:status) { |o| status_tag o.status }
            column(:total) { |o| number_to_currency(o.total, unit: '฿') }
            column(:created_at)
          end
          
          div class: 'see-all' do
            link_to 'ดูทั้งหมด', admin_orders_path
          end
        end
      end
      
      column do
        panel 'Users ล่าสุด' do
          table_for User.order(created_at: :desc).limit(10) do
            column('ชื่อ') { |u| link_to u.name, admin_user_path(u) }
            column(:email)
            column('สมัครเมื่อ') { |u| u.created_at.strftime('%d/%m/%Y') }
          end
        end
      end
    end
  end
end
```

---

## Step 1522: Administrate Gem

Administrate เป็น gem ที่ยืดหยุ่นกว่าและ customize ได้ง่ายกว่า ActiveAdmin

```ruby
# Gemfile
gem 'administrate'
gem 'administrate-field-belongs_to_search'  # Search in belongs_to

# Terminal
rails generate administrate:install
rails generate administrate:dashboard User
rails generate administrate:dashboard Product
rails generate administrate:dashboard Order
```

### User Dashboard

```ruby
# app/dashboards/user_dashboard.rb
require 'administrate/base_dashboard'

class UserDashboard < Administrate::BaseDashboard
  ATTRIBUTE_TYPES = {
    id: Field::Number,
    name: Field::String,
    email: Field::Email,
    role: Field::Select.with_options(
      collection: User.roles.keys
    ),
    active: Field::Boolean,
    phone: Field::String,
    orders: Field::HasMany,
    profile: Field::HasOne,
    created_at: Field::DateTime,
    updated_at: Field::DateTime,
    last_sign_in_at: Field::DateTime,
    orders_count: Field::Number.with_options(decimals: 0),
    total_spent: Field::Number.with_options(prefix: '฿', decimals: 2)
  }.freeze
  
  SHOW_PAGE_ATTRIBUTES = %i[
    id name email role active phone
    orders_count total_spent
    created_at last_sign_in_at
    orders profile
  ].freeze
  
  INDEX_ATTRIBUTES = %i[
    id name email role active created_at orders_count
  ].freeze
  
  FORM_ATTRIBUTES = %i[
    name email role active phone
  ].freeze
  
  COLLECTION_FILTERS = {
    active: ->(resources) { resources.where(active: true) },
    inactive: ->(resources) { resources.where(active: false) },
    admin: ->(resources) { resources.where(role: 'admin') }
  }.freeze
  
  def display_resource(user)
    "#{user.name} (#{user.email})"
  end
end
```

### Custom Controller

```ruby
# app/controllers/admin/users_controller.rb
module Admin
  class UsersController < Admin::ApplicationController
    before_action :require_superadmin!, only: [:destroy]
    
    def index
      search_term = params[:search].to_s.strip
      
      resources = User.includes(:profile, :orders)
      
      if search_term.present?
        resources = resources.where(
          'name ILIKE :q OR email ILIKE :q', 
          q: "%#{search_term}%"
        )
      end
      
      @pagy, @resources = pagy(resources.order(created_at: :desc))
      
      render locals: {
        resources: @resources,
        page: Administrate::Page::Collection.new(dashboard, order: order)
      }
    end
    
    def impersonate
      user = User.find(params[:id])
      session[:admin_impersonating] = session[:admin_user_id] = current_admin.id
      sign_in(user)
      redirect_to root_path, notice: "กำลัง impersonate #{user.email}"
    end
    
    private
    
    def require_superadmin!
      redirect_to admin_root_path, alert: 'ต้องการสิทธิ์ Superadmin' unless current_admin.superadmin?
    end
  end
end
```

---

## Step 1523: Custom Admin Controller Approach

สร้าง admin interface เองโดยไม่ใช้ gem

```ruby
# app/controllers/admin/base_controller.rb
class Admin::BaseController < ApplicationController
  layout 'admin'
  
  before_action :authenticate_admin!
  before_action :set_current_admin
  
  helper_method :current_admin
  
  private
  
  def authenticate_admin!
    unless current_user&.admin?
      redirect_to root_path, alert: 'Access denied'
    end
  end
  
  def set_current_admin
    @current_admin = current_user
  end
  
  attr_reader :current_admin
end

# app/controllers/admin/dashboard_controller.rb
class Admin::DashboardController < Admin::BaseController
  def index
    @stats = {
      total_users: User.count,
      new_users_today: User.where('created_at >= ?', Date.today).count,
      total_orders: Order.count,
      pending_orders: Order.pending.count,
      revenue_today: Order.paid.where('paid_at >= ?', Date.today).sum(:total),
      revenue_this_month: Order.paid.where('paid_at >= ?', Date.today.beginning_of_month).sum(:total)
    }
    
    @recent_orders = Order.includes(:user).order(created_at: :desc).limit(10)
    @top_products = Product.joins(:order_items).group(:id).order('COUNT(order_items.id) DESC').limit(5)
  end
end

# app/controllers/admin/users_controller.rb
class Admin::UsersController < Admin::BaseController
  before_action :set_user, only: [:show, :edit, :update, :destroy, :ban, :activate]
  
  def index
    @q = User.ransack(params[:q])
    @users = @q.result(distinct: true)
                .includes(:profile, :orders)
                .page(params[:page])
                .per(25)
  end
  
  def show
    @recent_orders = @user.orders.order(created_at: :desc).limit(10)
    @total_spent = @user.orders.paid.sum(:total)
  end
  
  def new
    @user = User.new
  end
  
  def create
    result = UserRegistration.call(
      name: user_params[:name],
      email: user_params[:email],
      password: SecureRandom.hex(16),
      password_confirmation: SecureRandom.hex(16),
      role: user_params[:role]
    )
    
    if result.success?
      redirect_to admin_user_path(result.user), notice: 'สร้าง user สำเร็จ'
    else
      @user = result.user || User.new(user_params)
      flash.now[:alert] = result.errors.join(', ')
      render :new
    end
  end
  
  def update
    if @user.update(user_params)
      redirect_to admin_user_path(@user), notice: 'อัพเดทสำเร็จ'
    else
      render :edit
    end
  end
  
  def ban
    @user.update!(active: false, banned_at: Time.current, ban_reason: params[:reason])
    UserMailer.account_banned(@user).deliver_later
    redirect_to admin_user_path(@user), notice: 'ระงับ user สำเร็จ'
  end
  
  def activate
    @user.update!(active: true, banned_at: nil, ban_reason: nil)
    redirect_to admin_user_path(@user), notice: 'เปิดใช้งาน user สำเร็จ'
  end
  
  def destroy
    @user.destroy
    redirect_to admin_users_path, notice: 'ลบ user สำเร็จ'
  end
  
  private
  
  def set_user
    @user = User.find(params[:id])
  end
  
  def user_params
    params.require(:user).permit(:name, :email, :role, :active, :phone)
  end
end
```

### Admin Routes

```ruby
# config/routes.rb
namespace :admin do
  root to: 'dashboard#index'
  
  resources :users do
    member do
      put :ban
      put :activate
      get :impersonate
    end
    collection do
      post :export
    end
  end
  
  resources :orders do
    member do
      put :refund
      put :mark_shipped
    end
  end
  
  resources :products do
    member do
      put :toggle_featured
    end
    collection do
      post :import
    end
  end
  
  resources :categories
  resources :coupons
  
  get 'analytics', to: 'analytics#index'
  get 'reports', to: 'reports#index'
  post 'reports/generate', to: 'reports#generate'
end
```

---

## Step 1524: Admin Authentication

```ruby
# app/models/admin_user.rb
class AdminUser < ApplicationRecord
  devise :database_authenticatable, :rememberable, :trackable, :lockable
  
  enum role: { admin: 0, superadmin: 1, moderator: 2 }
  
  validates :email, presence: true, uniqueness: true
  validates :role, presence: true
  
  def can_manage?(resource)
    case resource
    when User, Product then admin? || superadmin?
    when Order then true
    when AdminUser then superadmin?
    else false
    end
  end
end

# app/controllers/admin/base_controller.rb
class Admin::BaseController < ApplicationController
  layout 'admin'
  before_action :authenticate_admin_user!
  before_action :authorize_admin!
  
  helper_method :current_admin_user
  
  rescue_from CanCan::AccessDenied do |exception|
    redirect_to admin_root_path, alert: "ไม่มีสิทธิ์: #{exception.message}"
  end
  
  private
  
  def authenticate_admin_user!
    unless current_admin_user
      redirect_to new_admin_user_session_path
    end
  end
  
  def authorize_admin!
    unless current_admin_user.can_manage?(resource_class)
      raise CanCan::AccessDenied
    end
  end
  
  def resource_class
    self.class.name.demodulize.gsub('Controller', '').singularize.constantize
  rescue NameError
    nil
  end
end
```

---

## Step 1525: CRUD with Admin

```ruby
# app/views/admin/products/index.html.erb
<div class="admin-page">
  <div class="admin-header">
    <h1>สินค้า (<%= @products.total_count %>)</h1>
    <div class="admin-actions">
      <%= link_to 'เพิ่มสินค้าใหม่', new_admin_product_path, class: 'btn btn-primary' %>
      <%= link_to 'Export CSV', admin_products_path(format: :csv), class: 'btn btn-secondary' %>
      <%= link_to 'Import', import_admin_products_path, class: 'btn btn-secondary' %>
    </div>
  </div>
  
  <%# Search and Filters %>
  <%= search_form_for @q, url: admin_products_path do |f| %>
    <div class="filters">
      <%= f.search_field :name_cont, placeholder: 'ค้นหาชื่อสินค้า' %>
      <%= f.select :category_id_eq, 
          Category.all.map { |c| [c.name, c.id] },
          include_blank: 'ทุก Category' %>
      <%= f.number_field :price_gteq, placeholder: 'ราคาต่ำสุด' %>
      <%= f.number_field :price_lteq, placeholder: 'ราคาสูงสุด' %>
      <%= f.check_box :active_eq_true %>
      <%= f.label :active_eq_true, 'เฉพาะที่ active' %>
      <%= f.submit 'ค้นหา', class: 'btn' %>
      <%= link_to 'ล้าง', admin_products_path, class: 'btn btn-link' %>
    </div>
  <% end %>
  
  <%# Table %>
  <table class="admin-table">
    <thead>
      <tr>
        <th><%= sort_link(@q, :id, 'ID') %></th>
        <th>รูป</th>
        <th><%= sort_link(@q, :name, 'ชื่อสินค้า') %></th>
        <th><%= sort_link(@q, :price, 'ราคา') %></th>
        <th><%= sort_link(@q, :stock, 'Stock') %></th>
        <th>Status</th>
        <th>Actions</th>
      </tr>
    </thead>
    <tbody>
      <% @products.each do |product| %>
        <tr>
          <td><%= product.id %></td>
          <td>
            <% if product.cover_image.attached? %>
              <%= image_tag product.cover_image.variant(resize_to_limit: [50, 50]) %>
            <% end %>
          </td>
          <td>
            <%= link_to product.name, admin_product_path(product) %>
            <small><%= product.sku %></small>
          </td>
          <td><%= number_to_currency(product.price, unit: '฿') %></td>
          <td>
            <span class="<%= product.stock < 10 ? 'text-danger' : '' %>">
              <%= product.stock %>
            </span>
          </td>
          <td>
            <span class="badge <%= product.active? ? 'badge-success' : 'badge-danger' %>">
              <%= product.active? ? 'Active' : 'Inactive' %>
            </span>
          </td>
          <td>
            <%= link_to 'ดู', admin_product_path(product), class: 'btn btn-sm' %>
            <%= link_to 'แก้ไข', edit_admin_product_path(product), class: 'btn btn-sm btn-primary' %>
            <%= link_to 'ลบ', admin_product_path(product), 
                method: :delete,
                data: { confirm: 'ต้องการลบสินค้านี้?' },
                class: 'btn btn-sm btn-danger' %>
          </td>
        </tr>
      <% end %>
    </tbody>
  </table>
  
  <%# Pagination %>
  <%= paginate @products %>
</div>
```

---

## Step 1526: Filtering และ Search ใน Admin

```ruby
# app/controllers/admin/products_controller.rb
class Admin::ProductsController < Admin::BaseController
  def index
    @q = Product.ransack(params[:q])
    
    @products = @q.result(distinct: true)
      .includes(:category, :images)
      .tap { |r| apply_additional_filters(r) }
      .order(order_column => order_direction)
      .page(params[:page])
      .per(params[:per_page] || 25)
  end
  
  private
  
  def apply_additional_filters(relation)
    relation
      .then { |r| params[:low_stock] == '1' ? r.where('stock < 10') : r }
      .then { |r| params[:featured] == '1' ? r.where(featured: true) : r }
      .then { |r| params[:out_of_stock] == '1' ? r.where(stock: 0) : r }
  end
  
  def order_column
    allowed = %w[id name price stock created_at]
    allowed.include?(params[:sort]) ? params[:sort] : 'created_at'
  end
  
  def order_direction
    params[:direction] == 'asc' ? 'asc' : 'desc'
  end
end
```

---

## Step 1527: Charts และ Statistics ใน Admin

```ruby
# app/controllers/admin/analytics_controller.rb
class Admin::AnalyticsController < Admin::BaseController
  def index
    @date_range = parse_date_range
    
    @revenue_chart = revenue_by_day(@date_range)
    @orders_chart = orders_by_status
    @top_products = top_selling_products
    @user_registrations = user_registrations_by_day(@date_range)
    @conversion_funnel = conversion_funnel_data
  end
  
  private
  
  def revenue_by_day(range)
    Order.paid
      .where(paid_at: range)
      .group("DATE(paid_at)")
      .order("DATE(paid_at)")
      .sum(:total)
  end
  
  def orders_by_status
    Order.group(:status).count
  end
  
  def top_selling_products
    Product
      .joins(:order_items)
      .group(:id, :name)
      .order('COUNT(order_items.id) DESC')
      .limit(10)
      .select('products.id, products.name, COUNT(order_items.id) as sales_count, SUM(order_items.quantity) as total_quantity')
  end
  
  def user_registrations_by_day(range)
    User.where(created_at: range)
        .group("DATE(created_at)")
        .order("DATE(created_at)")
        .count
  end
  
  def conversion_funnel_data
    {
      visitors: PageView.where('created_at > ?', 30.days.ago).count,
      product_views: Event.where(name: 'product_view').where('created_at > ?', 30.days.ago).count,
      add_to_cart: Event.where(name: 'add_to_cart').where('created_at > ?', 30.days.ago).count,
      checkout_started: Order.where('created_at > ?', 30.days.ago).count,
      purchases: Order.paid.where('paid_at > ?', 30.days.ago).count
    }
  end
  
  def parse_date_range
    start_date = params[:start_date]&.to_date || 30.days.ago.to_date
    end_date = params[:end_date]&.to_date || Date.today
    start_date.beginning_of_day..end_date.end_of_day
  end
end
```

```erb
<%# app/views/admin/analytics/index.html.erb %>
<div class="admin-analytics">
  <div class="date-range-picker">
    <%= form_with url: admin_analytics_path, method: :get do |f| %>
      <%= f.date_field :start_date, value: 30.days.ago.to_date %>
      <%= f.date_field :end_date, value: Date.today %>
      <%= f.submit 'อัพเดท' %>
    <% end %>
  </div>
  
  <div class="chart-container">
    <h3>รายได้ตามวัน</h3>
    <%= line_chart @revenue_chart, 
        prefix: '฿',
        thousands: ',',
        curve: false,
        colors: ['#4CAF50'] %>
  </div>
  
  <div class="chart-row">
    <div class="chart-half">
      <h3>Orders ตาม Status</h3>
      <%= pie_chart @orders_chart %>
    </div>
    
    <div class="chart-half">
      <h3>สมาชิกใหม่ตามวัน</h3>
      <%= bar_chart @user_registrations,
          colors: ['#2196F3'] %>
    </div>
  </div>
</div>
```

---

## Step 1528: Export Data

```ruby
# app/controllers/admin/users_controller.rb - Export CSV
def export
  @users = User.includes(:profile, :orders).all
  
  respond_to do |format|
    format.csv do
      response.headers['Content-Type'] = 'text/csv; charset=utf-8'
      response.headers['Content-Disposition'] = 
        "attachment; filename=users_#{Date.today}.csv"
      
      render template: 'admin/users/export'
    end
    
    format.xlsx do
      render xlsx: 'export', filename: "users_#{Date.today}.xlsx"
    end
  end
end

# app/views/admin/users/export.csv.erb
<% 
  headers = ['ID', 'ชื่อ', 'อีเมล', 'บทบาท', 'สถานะ', 'วันที่สมัคร', 'Orders', 'ยอดซื้อรวม']
%>
<%= CSV.generate(headers: headers, write_headers: true, encoding: 'UTF-8') do |csv|
  @users.each do |user|
    csv << [
      user.id,
      user.name,
      user.email,
      user.role,
      user.active? ? 'Active' : 'Inactive',
      user.created_at.strftime('%d/%m/%Y'),
      user.orders.count,
      user.orders.paid.sum(:total)
    ]
  end
end %>

# ใช้ axlsx gem สำหรับ Excel
# app/views/admin/users/export.xlsx.axlsx
wb = xlsx_package.workbook

wb.add_worksheet(name: 'Users') do |sheet|
  # Header row with styling
  header_style = sheet.styles.add_style(
    b: true,
    bg_color: '4472C4',
    fg_color: 'FFFFFF'
  )
  
  sheet.add_row(
    ['ID', 'ชื่อ', 'อีเมล', 'บทบาท', 'วันที่สมัคร', 'ยอดซื้อรวม'],
    style: header_style
  )
  
  @users.each do |user|
    sheet.add_row [
      user.id,
      user.name,
      user.email,
      user.role,
      user.created_at,
      user.orders.paid.sum(:total)
    ]
  end
  
  # Auto-filter
  sheet.auto_filter = "A1:F1"
end
```

---

## Step 1529-1540: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: ActiveAdmin Dashboard Widget

```ruby
# เพิ่ม custom chart widget ใน dashboard
ActiveAdmin.register_page 'Dashboard' do
  content do
    # Revenue trend chart (ใช้ chartkick)
    panel 'Revenue Trend (30 วันล่าสุด)' do
      revenue_data = Order.paid
        .where('paid_at >= ?', 30.days.ago)
        .group("DATE(paid_at)")
        .order("DATE(paid_at)")
        .sum(:total)
      
      para line_chart(revenue_data, prefix: '฿', thousands: ',')
    end
    
    # Top categories
    panel 'Top Categories' do
      category_data = OrderItem.joins(product: :category)
        .where('order_items.created_at >= ?', 30.days.ago)
        .group('categories.name')
        .sum(:quantity)
        .sort_by { |_, v| -v }
        .first(10)
      
      para pie_chart(category_data)
    end
  end
end
```

### แบบฝึกหัดที่ 2: Custom Admin Filter

```ruby
# app/admin/orders.rb
ActiveAdmin.register Order do
  filter :status, as: :select, collection: Order.statuses.keys
  filter :user_name_cont, label: 'ชื่อลูกค้า'
  filter :user_email_cont, label: 'อีเมลลูกค้า'
  filter :created_at
  filter :total_gt, label: 'ยอดรวม มากกว่า'
  filter :total_lt, label: 'ยอดรวม น้อยกว่า'
  
  # Custom filter: จำนวน items
  filter :items_count_gt,
    as: :number,
    label: 'จำนวน items มากกว่า',
    filters: ['gt']
  
  # Scope filters
  scope :all, default: true
  scope :pending
  scope :paid
  scope :processing
  scope :shipped
  scope 'High Value', :high_value do
    Order.where('total > ?', Order.average(:total))
  end
  
  # Batch refund
  batch_action :refund, 
    confirm: 'ต้องการ refund orders ที่เลือก?',
    if: proc { can?(:refund, Order) } do |ids|
    
    Order.find(ids).each do |order|
      ProcessRefund.call(order: order, reason: 'bulk_refund')
    end
    
    redirect_to collection_path, notice: "Refunded #{ids.count} orders"
  end
end
```

### แบบฝึกหัดที่ 3: Admin สำหรับ Import CSV

```ruby
# app/controllers/admin/products_controller.rb
def import
  if request.post?
    if params[:file].blank?
      flash.now[:alert] = 'กรุณาเลือกไฟล์ CSV'
      return
    end
    
    result = ProductCsvImport.call(
      file: params[:file],
      performed_by: current_admin_user
    )
    
    if result.success?
      redirect_to admin_products_path, 
        notice: "นำเข้า #{result.created_count} สินค้าใหม่, อัพเดท #{result.updated_count} สินค้า"
    else
      flash.now[:alert] = result.errors.join('<br>')
    end
  end
end

# app/services/product_csv_import.rb
class ProductCsvImport
  attr_reader :created_count, :updated_count, :errors
  
  def self.call(**args) = new(**args).tap(&:call)
  
  def initialize(file:, performed_by:)
    @file = file
    @performed_by = performed_by
    @created_count = 0
    @updated_count = 0
    @errors = []
  end
  
  def call
    require 'csv'
    
    CSV.foreach(@file.path, headers: true, encoding: 'UTF-8') do |row|
      process_row(row, $.)
    end
  rescue CSV::MalformedCSVError => e
    @errors << "CSV format error: #{e.message}"
  end
  
  def success? = @errors.empty?
  
  private
  
  def process_row(row, line_number)
    product = Product.find_or_initialize_by(sku: row['sku'])
    
    product.assign_attributes(
      name: row['name'],
      price: row['price'].to_f,
      stock: row['stock'].to_i,
      description: row['description'],
      category: Category.find_by(name: row['category'])
    )
    
    if product.new_record?
      product.save! ? @created_count += 1 : nil
    else
      product.save! ? @updated_count += 1 : nil
    end
  rescue => e
    @errors << "Line #{line_number}: #{e.message}"
  end
end
```

### แบบฝึกหัดที่ 4: Admin Authorization

```ruby
# app/models/ability.rb (CanCanCan)
class Ability
  include CanCan::Ability
  
  def initialize(admin)
    return unless admin
    
    if admin.superadmin?
      can :manage, :all
    elsif admin.admin?
      can :manage, [User, Product, Order, Category, Coupon]
      can :read, AdminUser
      cannot :destroy, AdminUser
    elsif admin.moderator?
      can :read, [User, Product, Order]
      can :update, Order, status: ['processing', 'shipped']
      can :manage, Product
    end
  end
end

# app/controllers/admin/base_controller.rb
class Admin::BaseController < ApplicationController
  include CanCan::ControllerAdditions
  
  check_authorization unless: :skip_authorization?
  
  rescue_from CanCan::AccessDenied do |exception|
    respond_to do |format|
      format.json { render json: { error: exception.message }, status: :forbidden }
      format.html { redirect_to admin_root_path, alert: exception.message }
    end
  end
  
  private
  
  def current_ability
    @current_ability ||= Ability.new(current_admin_user)
  end
  
  def skip_authorization?
    false
  end
end
```

### แบบฝึกหัดที่ 5: Admin Activity Log

```ruby
# app/models/admin_activity_log.rb
class AdminActivityLog < ApplicationRecord
  belongs_to :admin_user
  belongs_to :resource, polymorphic: true, optional: true
  
  scope :recent, -> { order(created_at: :desc) }
end

# app/controllers/concerns/admin_logging.rb
module AdminLogging
  extend ActiveSupport::Concern
  
  included do
    after_action :log_admin_activity, only: [:create, :update, :destroy]
  end
  
  private
  
  def log_admin_activity
    return unless response.successful? || response.redirect?
    
    AdminActivityLog.create!(
      admin_user: current_admin_user,
      action: action_name,
      controller: controller_name,
      resource_type: resource_class_name,
      resource_id: resource_id,
      changes: tracked_changes,
      ip_address: request.remote_ip,
      user_agent: request.user_agent
    )
  end
  
  def tracked_changes
    return {} unless @resource.respond_to?(:saved_changes)
    @resource.saved_changes.except('updated_at')
  end
end
```

---

**จบ Part 70: Admin Panels**

*ในส่วนถัดไป Part 71 เราจะเรียนรู้เกี่ยวกับ Payment Integration*
