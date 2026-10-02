# Project 4: RESTful API Backend

## Rails API Mode ที่สมบูรณ์

---

## ภาพรวม

เราจะสร้าง **BookStore API** - RESTful API สำหรับร้านหนังสือออนไลน์ที่มี:
- Rails API mode
- JWT Authentication
- CRUD สำหรับ resources หลัก
- Pagination และ Filtering
- Rate Limiting
- API Versioning
- Swagger Documentation
- Deployment to Heroku

---

## ขั้นตอนที่ 1: Setup Project

```bash
# สร้าง Rails API project
rails new bookstore_api \
  --api \
  --database=postgresql \
  --skip-test

cd bookstore_api

# เพิ่ม gems
bundle add jwt
bundle add bcrypt
bundle add kaminari
bundle add rack-attack
bundle add rswag        # Swagger documentation
bundle add rswag-api
bundle add rswag-ui
bundle add active_model_serializers
bundle add rack-cors
bundle add dotenv-rails

# Development gems
bundle add rspec-rails --group="development,test"
bundle add factory_bot_rails --group="development,test"
bundle add faker --group="development,test"
bundle add shoulda-matchers --group="test"
```

---

## ขั้นตอนที่ 2: Database Schema

```ruby
# db/migrate/001_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users do |t|
      t.string  :email,              null: false
      t.string  :password_digest,    null: false
      t.string  :name,               null: false
      t.string  :role,               null: false, default: 'customer'
      t.boolean :active,             null: false, default: true
      t.datetime :last_login_at
      t.timestamps

      t.index :email, unique: true
    end
  end
end

# db/migrate/002_create_authors.rb
class CreateAuthors < ActiveRecord::Migration[7.1]
  def change
    create_table :authors do |t|
      t.string :name,    null: false
      t.text   :bio
      t.string :country
      t.date   :born_on
      t.timestamps
    end
  end
end

# db/migrate/003_create_categories.rb
class CreateCategories < ActiveRecord::Migration[7.1]
  def change
    create_table :categories do |t|
      t.string     :name,     null: false
      t.string     :slug,     null: false
      t.text       :description
      t.references :parent,   foreign_key: { to_table: :categories }
      t.timestamps

      t.index :slug, unique: true
    end
  end
end

# db/migrate/004_create_books.rb
class CreateBooks < ActiveRecord::Migration[7.1]
  def change
    create_table :books do |t|
      t.string     :title,      null: false
      t.string     :isbn,       null: false
      t.text       :description
      t.decimal    :price,      null: false, precision: 10, scale: 2
      t.integer    :stock,      null: false, default: 0
      t.string     :language,   null: false, default: 'th'
      t.date       :published_on
      t.string     :cover_url
      t.boolean    :active,     null: false, default: true
      t.references :category,   foreign_key: true
      t.timestamps

      t.index :isbn, unique: true
      t.index :title
      t.index [:active, :price]
    end
  end
end

# db/migrate/005_create_book_authors.rb
class CreateBookAuthors < ActiveRecord::Migration[7.1]
  def change
    create_table :book_authors do |t|
      t.references :book,   null: false, foreign_key: true
      t.references :author, null: false, foreign_key: true
      t.string     :role,   null: false, default: 'primary'
      t.timestamps

      t.index [:book_id, :author_id], unique: true
    end
  end
end

# db/migrate/006_create_orders.rb
class CreateOrders < ActiveRecord::Migration[7.1]
  def change
    create_table :orders do |t|
      t.references :user,     null: false, foreign_key: true
      t.string     :status,   null: false, default: 'pending'
      t.decimal    :subtotal, null: false, precision: 10, scale: 2
      t.decimal    :discount, null: false, precision: 10, scale: 2, default: 0
      t.decimal    :total,    null: false, precision: 10, scale: 2
      t.string     :payment_method
      t.string     :payment_status, null: false, default: 'unpaid'
      t.jsonb      :shipping_address, default: {}
      t.timestamps

      t.index [:user_id, :status]
      t.index :created_at
    end
  end
end

# db/migrate/007_create_order_items.rb
class CreateOrderItems < ActiveRecord::Migration[7.1]
  def change
    create_table :order_items do |t|
      t.references :order, null: false, foreign_key: true
      t.references :book,  null: false, foreign_key: true
      t.integer    :quantity,   null: false
      t.decimal    :unit_price, null: false, precision: 10, scale: 2
      t.timestamps
    end
  end
end

# db/migrate/008_create_reviews.rb
class CreateReviews < ActiveRecord::Migration[7.1]
  def change
    create_table :reviews do |t|
      t.references :user,   null: false, foreign_key: true
      t.references :book,   null: false, foreign_key: true
      t.integer    :rating, null: false  # 1-5
      t.text       :content
      t.boolean    :approved, null: false, default: false
      t.timestamps

      t.index [:user_id, :book_id], unique: true
    end
  end
end

# db/migrate/009_create_api_tokens.rb
class CreateApiTokens < ActiveRecord::Migration[7.1]
  def change
    create_table :api_tokens do |t|
      t.references :user, null: false, foreign_key: true
      t.string     :token,      null: false
      t.string     :name,       null: false  # "Mobile App", "Web App"
      t.datetime   :expires_at
      t.datetime   :last_used_at
      t.boolean    :active, null: false, default: true
      t.timestamps

      t.index :token, unique: true
    end
  end
end
```

---

## ขั้นตอนที่ 3: Models

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  ROLES = %w[admin customer].freeze

  has_many :orders,    dependent: :nullify
  has_many :reviews,   dependent: :destroy
  has_many :api_tokens, dependent: :destroy

  validates :email, presence: true, uniqueness: { case_sensitive: false },
                    format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name,  presence: true
  validates :role,  inclusion: { in: ROLES }

  before_validation { email&.downcase! }

  scope :active, -> { where(active: true) }

  def admin?    = role == 'admin'
  def customer? = role == 'customer'

  def generate_auth_token
    payload = {
      user_id: id,
      email:   email,
      role:    role,
      exp:     30.days.from_now.to_i,
      iat:     Time.current.to_i,
      jti:     SecureRandom.uuid  # JWT ID สำหรับ blacklisting
    }
    JWT.encode(payload, ENV.fetch('JWT_SECRET'), 'HS256')
  end

  def self.authenticate_with_token(token)
    decoded = JWT.decode(
      token,
      ENV.fetch('JWT_SECRET'),
      true,
      { algorithm: 'HS256' }
    ).first

    find_by(id: decoded['user_id'], active: true)
  rescue JWT::DecodeError, JWT::ExpiredSignature
    nil
  end
end

# app/models/book.rb
class Book < ApplicationRecord
  belongs_to :category
  has_many   :book_authors, dependent: :destroy
  has_many   :authors,      through: :book_authors
  has_many   :reviews,      dependent: :destroy
  has_many   :order_items,  dependent: :nullify

  validates :title,  presence: true
  validates :isbn,   presence: true, uniqueness: true,
                     format: { with: /\A(?:\d{9}[\dX]|\d{13})\z/ }
  validates :price,  presence: true, numericality: { greater_than: 0 }
  validates :stock,  numericality: { greater_than_or_equal_to: 0 }
  validates :language, inclusion: { in: %w[th en ja ko zh fr de es] }

  scope :active,      -> { where(active: true) }
  scope :in_stock,    -> { where('stock > 0') }
  scope :by_price,    ->(min, max) { where(price: min..max) }
  scope :by_language, ->(lang) { where(language: lang) }
  scope :search,      ->(q) { where('title ILIKE ? OR description ILIKE ?', "%#{q}%", "%#{q}%") }

  def average_rating
    reviews.approved.average(:rating)&.round(1) || 0
  end

  def in_stock?
    stock > 0
  end

  def decrease_stock!(quantity)
    raise InsufficientStockError, "Not enough stock" if stock < quantity
    decrement!(:stock, quantity)
  end
end

# app/models/order.rb
class Order < ApplicationRecord
  STATUSES = %w[pending confirmed processing shipped delivered cancelled].freeze
  PAYMENT_STATUSES = %w[unpaid paid refunded partially_refunded].freeze

  belongs_to :user
  has_many   :order_items, dependent: :destroy
  has_many   :books,       through: :order_items

  validates :status,         inclusion: { in: STATUSES }
  validates :payment_status, inclusion: { in: PAYMENT_STATUSES }
  validates :total,          numericality: { greater_than_or_equal_to: 0 }

  scope :recent, -> { order(created_at: :desc) }
  scope :for_user, ->(user) { where(user: user) }

  def calculate_totals!
    self.subtotal = order_items.sum { |item| item.unit_price * item.quantity }
    self.total    = subtotal - discount
    save!
  end

  def can_cancel?
    status.in?(%w[pending confirmed])
  end

  def cancel!
    raise "Cannot cancel #{status} order" unless can_cancel?

    transaction do
      update!(status: 'cancelled')
      order_items.each do |item|
        item.book.increment!(:stock, item.quantity)
      end
    end
  end
end
```

---

## ขั้นตอนที่ 4: Authentication

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include ActionController::HttpAuthentication::Token::ControllerMethods

  before_action :authenticate_user!

  attr_reader :current_user

  private

  def authenticate_user!
    token = extract_token_from_request
    return unauthorized_response unless token

    @current_user = User.authenticate_with_token(token)
    unauthorized_response unless @current_user
  end

  def authenticate_admin!
    authenticate_user!
    forbidden_response unless current_user&.admin?
  end

  def extract_token_from_request
    request.headers['Authorization']&.split('Bearer ')&.last
  end

  def unauthorized_response
    render json: {
      error:   'Unauthorized',
      message: 'Valid authentication token required'
    }, status: :unauthorized
  end

  def forbidden_response
    render json: {
      error:   'Forbidden',
      message: 'Insufficient permissions'
    }, status: :forbidden
  end

  def pagination_meta(collection)
    {
      current_page: collection.current_page,
      total_pages:  collection.total_pages,
      total_count:  collection.total_count,
      per_page:     collection.limit_value,
      next_page:    collection.next_page,
      prev_page:    collection.prev_page
    }
  end

  def render_error(message, status: :unprocessable_entity, errors: nil)
    response = { error: message }
    response[:errors] = errors if errors
    render json: response, status: status
  end
end

# app/controllers/api/v1/auth_controller.rb
module Api
  module V1
    class AuthController < ApplicationController
      skip_before_action :authenticate_user!

      # POST /api/v1/auth/register
      def register
        user = User.new(register_params)
        user.role = 'customer'

        if user.save
          token = user.generate_auth_token
          user.update_column(:last_login_at, Time.current)

          render json: {
            message: 'Registration successful',
            token:   token,
            user:    UserSerializer.new(user)
          }, status: :created
        else
          render_error('Registration failed', errors: user.errors.full_messages)
        end
      end

      # POST /api/v1/auth/login
      def login
        user = User.find_by(email: params[:email]&.downcase)

        if user&.authenticate(params[:password])
          unless user.active?
            return render_error('Account suspended', status: :forbidden)
          end

          token = user.generate_auth_token
          user.update_column(:last_login_at, Time.current)

          render json: {
            message:    'Login successful',
            token:      token,
            expires_at: 30.days.from_now.iso8601,
            user:       UserSerializer.new(user)
          }
        else
          render_error('Invalid email or password', status: :unauthorized)
        end
      end

      # GET /api/v1/auth/me
      def me
        authenticate_user!
        render json: { user: UserSerializer.new(current_user) }
      end

      # PUT /api/v1/auth/change_password
      def change_password
        authenticate_user!

        unless current_user.authenticate(params[:current_password])
          return render_error('Current password is incorrect', status: :unprocessable_entity)
        end

        if current_user.update(
          password: params[:new_password],
          password_confirmation: params[:new_password_confirmation]
        )
          render json: { message: 'Password changed successfully' }
        else
          render_error('Password change failed', errors: current_user.errors.full_messages)
        end
      end

      private

      def register_params
        params.require(:user).permit(:email, :password, :password_confirmation, :name)
      end
    end
  end
end
```

---

## ขั้นตอนที่ 5: Books API

```ruby
# app/controllers/api/v1/books_controller.rb
module Api
  module V1
    class BooksController < ApplicationController
      skip_before_action :authenticate_user!, only: [:index, :show, :search]

      before_action :set_book,     only: [:show, :update, :destroy]
      before_action :authenticate_admin!, only: [:create, :update, :destroy]

      # GET /api/v1/books
      def index
        @books = Book.active
                     .includes(:authors, :category, :reviews)

        # Filtering
        @books = apply_filters(@books)

        # Sorting
        @books = apply_sorting(@books)

        # Pagination
        @books = @books.page(params[:page]).per(params[:per_page] || 20)

        render json: {
          books:      ActiveModelSerializers::SerializableResource.new(@books),
          pagination: pagination_meta(@books),
          filters:    applied_filters
        }
      end

      # GET /api/v1/books/:id
      def show
        render json: {
          book: BookDetailSerializer.new(@book, include: [:authors, :category, :reviews])
        }
      end

      # POST /api/v1/books
      def create
        @book = Book.new(book_params)

        if @book.save
          render json: { book: BookSerializer.new(@book) }, status: :created
        else
          render_error('Failed to create book', errors: @book.errors.full_messages)
        end
      end

      # PUT/PATCH /api/v1/books/:id
      def update
        if @book.update(book_params)
          render json: { book: BookSerializer.new(@book) }
        else
          render_error('Failed to update book', errors: @book.errors.full_messages)
        end
      end

      # DELETE /api/v1/books/:id
      def destroy
        @book.update!(active: false)
        head :no_content
      end

      # GET /api/v1/books/search
      def search
        q = params[:q]
        return render_error('Search query required', status: :bad_request) if q.blank?
        return render_error('Query too short', status: :bad_request) if q.length < 2

        @books = Book.active
                     .search(q)
                     .includes(:authors, :category)
                     .page(params[:page])
                     .per(params[:per_page] || 20)

        render json: {
          books:      ActiveModelSerializers::SerializableResource.new(@books),
          query:      q,
          pagination: pagination_meta(@books)
        }
      end

      private

      def set_book
        @book = Book.find(params[:id])
      rescue ActiveRecord::RecordNotFound
        render_error('Book not found', status: :not_found)
      end

      def book_params
        params.require(:book).permit(
          :title, :isbn, :description, :price, :stock,
          :language, :published_on, :cover_url, :active,
          :category_id, author_ids: []
        )
      end

      def apply_filters(books)
        books = books.where(category_id: params[:category_id]) if params[:category_id].present?
        books = books.by_language(params[:language])           if params[:language].present?
        books = books.in_stock                                 if params[:in_stock] == 'true'
        books = books.by_price(params[:min_price], params[:max_price]) if params[:min_price] || params[:max_price]
        books
      end

      def apply_sorting(books)
        case params[:sort]
        when 'price_asc'    then books.order(price: :asc)
        when 'price_desc'   then books.order(price: :desc)
        when 'newest'       then books.order(published_on: :desc)
        when 'title'        then books.order(title: :asc)
        when 'bestselling'  then books.joins(:order_items)
                                      .group('books.id')
                                      .order('COUNT(order_items.id) DESC')
        else books.order(created_at: :desc)
        end
      end

      def applied_filters
        {
          category_id: params[:category_id],
          language:    params[:language],
          in_stock:    params[:in_stock],
          min_price:   params[:min_price],
          max_price:   params[:max_price],
          sort:        params[:sort] || 'newest'
        }.compact
      end
    end
  end
end
```

---

## ขั้นตอนที่ 6: Orders API

```ruby
# app/controllers/api/v1/orders_controller.rb
module Api
  module V1
    class OrdersController < ApplicationController
      before_action :set_order, only: [:show, :cancel, :update_status]
      before_action :ensure_order_owner, only: [:show, :cancel]

      # GET /api/v1/orders
      def index
        @orders = current_user.orders
                              .includes(:order_items => :book)
                              .recent

        @orders = @orders.where(status: params[:status]) if params[:status].present?
        @orders = @orders.page(params[:page]).per(params[:per_page] || 10)

        render json: {
          orders:     ActiveModelSerializers::SerializableResource.new(@orders),
          pagination: pagination_meta(@orders)
        }
      end

      # GET /api/v1/orders/:id
      def show
        render json: { order: OrderDetailSerializer.new(@order) }
      end

      # POST /api/v1/orders
      def create
        ActiveRecord::Base.transaction do
          @order = current_user.orders.build(
            status:   'pending',
            discount: 0,
            shipping_address: shipping_address_params
          )

          items_data.each do |item_params|
            book = Book.find(item_params[:book_id])

            unless book.active? && book.stock >= item_params[:quantity].to_i
              render_error("#{book.title} is not available in requested quantity",
                           status: :unprocessable_entity)
              raise ActiveRecord::Rollback
              return
            end

            @order.order_items.build(
              book:       book,
              quantity:   item_params[:quantity].to_i,
              unit_price: book.price
            )
          end

          @order.save!
          @order.calculate_totals!

          # Decrease stock
          @order.order_items.each do |item|
            item.book.decrease_stock!(item.quantity)
          end
        end

        render json: { order: OrderDetailSerializer.new(@order) }, status: :created
      rescue ActiveRecord::RecordInvalid => e
        render_error('Order creation failed', errors: @order.errors.full_messages)
      rescue InsufficientStockError => e
        render_error(e.message, status: :unprocessable_entity)
      end

      # POST /api/v1/orders/:id/cancel
      def cancel
        if @order.can_cancel?
          @order.cancel!
          render json: { message: 'Order cancelled', order: OrderSerializer.new(@order) }
        else
          render_error("Cannot cancel order with status '#{@order.status}'",
                       status: :unprocessable_entity)
        end
      end

      # PUT /api/v1/orders/:id/status (admin only)
      def update_status
        authenticate_admin!

        new_status = params[:status]
        unless Order::STATUSES.include?(new_status)
          return render_error("Invalid status: #{new_status}")
        end

        @order.update!(status: new_status)
        render json: { order: OrderSerializer.new(@order) }
      end

      private

      def set_order
        @order = Order.find(params[:id])
      rescue ActiveRecord::RecordNotFound
        render_error('Order not found', status: :not_found)
      end

      def ensure_order_owner
        unless @order.user == current_user || current_user.admin?
          forbidden_response
        end
      end

      def items_data
        params.require(:items).map do |item|
          item.permit(:book_id, :quantity)
        end
      end

      def shipping_address_params
        params.permit(
          shipping_address: [:name, :phone, :street, :city, :postal_code, :country]
        )[:shipping_address] || {}
      end
    end
  end
end
```

---

## ขั้นตอนที่ 7: Serializers

```ruby
# app/serializers/book_serializer.rb
class BookSerializer < ActiveModel::Serializer
  attributes :id, :title, :isbn, :price, :stock, :language,
             :published_on, :cover_url, :active, :average_rating,
             :in_stock, :created_at, :updated_at

  belongs_to :category
  has_many   :authors

  def in_stock
    object.in_stock?
  end
end

# app/serializers/book_detail_serializer.rb
class BookDetailSerializer < BookSerializer
  attributes :description
  has_many :reviews
end

# app/serializers/user_serializer.rb
class UserSerializer < ActiveModel::Serializer
  attributes :id, :email, :name, :role, :last_login_at, :created_at

  # ไม่ serialize password!
end

# app/serializers/order_serializer.rb
class OrderSerializer < ActiveModel::Serializer
  attributes :id, :status, :payment_status, :subtotal, :discount,
             :total, :shipping_address, :created_at, :updated_at

  belongs_to :user
  has_many   :order_items

  class OrderItemSerializer < ActiveModel::Serializer
    attributes :id, :quantity, :unit_price, :subtotal
    belongs_to :book

    def subtotal
      object.quantity * object.unit_price
    end
  end
end
```

---

## ขั้นตอนที่ 8: Rate Limiting

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Throttle IP addresses
  throttle('req/ip', limit: 300, period: 5.minutes) do |req|
    req.ip unless req.path.start_with?('/assets')
  end

  # ป้องกัน login brute force
  throttle('logins/ip', limit: 10, period: 15.minutes) do |req|
    req.ip if req.path == '/api/v1/auth/login' && req.post?
  end

  # ป้องกัน registration flood
  throttle('registrations/ip', limit: 5, period: 1.hour) do |req|
    req.ip if req.path == '/api/v1/auth/register' && req.post?
  end

  # ป้องกัน search spam
  throttle('search/ip', limit: 60, period: 1.minute) do |req|
    req.ip if req.path.include?('/search')
  end

  # Authenticated users มี limit สูงกว่า
  throttle('req/authenticated', limit: 1000, period: 1.hour) do |req|
    token = req.env['HTTP_AUTHORIZATION']&.split('Bearer ')&.last
    next unless token

    user = User.authenticate_with_token(token)
    user&.id
  end

  # Custom response สำหรับ throttled requests
  self.throttled_responder = ->(env) {
    [
      429,
      {
        'Content-Type'   => 'application/json',
        'Retry-After'    => env['rack.attack.match_data'][:period].to_s
      },
      [{
        error:   'Too Many Requests',
        message: 'Rate limit exceeded. Please slow down.',
        retry_after: env['rack.attack.match_data'][:period]
      }.to_json]
    ]
  }

  # Whitelist
  safelist('allow-localhost') do |req|
    req.ip == '127.0.0.1' && Rails.env.development?
  end
end

# Log throttled requests
ActiveSupport::Notifications.subscribe('throttle.rack_attack') do |name, start, finish, request_id, payload|
  req = payload[:request]
  Rails.logger.warn "Throttled: #{req.ip} #{req.path} [#{payload[:match_discriminator]}]"
end
```

---

## ขั้นตอนที่ 9: API Versioning

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      # Auth
      post 'auth/register',        to: 'auth#register'
      post 'auth/login',           to: 'auth#login'
      get  'auth/me',              to: 'auth#me'
      put  'auth/change_password', to: 'auth#change_password'

      # Books
      resources :books, only: [:index, :show, :create, :update, :destroy] do
        collection do
          get :search
          get :featured
        end
      end

      # Categories
      resources :categories, only: [:index, :show]

      # Authors
      resources :authors, only: [:index, :show]

      # Orders
      resources :orders, only: [:index, :show, :create] do
        member do
          post :cancel
          put  :update_status
        end
      end

      # Reviews
      resources :books do
        resources :reviews, only: [:index, :create, :destroy]
      end

      # Profile
      resource :profile, only: [:show, :update]
    end

    # V2 (future) - Header-based versioning
    namespace :v2 do
      resources :books
      # ... V2 endpoints
    end
  end

  # Health check
  get '/health', to: 'health#check'
end

# Header-based versioning middleware
# app/middleware/api_version_middleware.rb
class ApiVersionMiddleware
  VALID_VERSIONS = %w[v1 v2].freeze
  DEFAULT_VERSION = 'v1'

  def initialize(app)
    @app = app
  end

  def call(env)
    request = ActionDispatch::Request.new(env)

    # อ่าน version จาก Accept header
    # Accept: application/vnd.bookstore.v2+json
    if request.path.start_with?('/api/')
      version = extract_version_from_header(request) ||
                extract_version_from_path(request) ||
                DEFAULT_VERSION

      env['api.version'] = version
    end

    @app.call(env)
  end

  private

  def extract_version_from_header(request)
    accept = request.headers['Accept']
    return nil unless accept

    match = accept.match(/application\/vnd\.bookstore\.(v\d+)\+json/)
    return nil unless match

    version = match[1]
    VALID_VERSIONS.include?(version) ? version : nil
  end

  def extract_version_from_path(request)
    match = request.path.match(%r{/api/(v\d+)/})
    match && VALID_VERSIONS.include?(match[1]) ? match[1] : nil
  end
end
```

---

## ขั้นตอนที่ 10: Swagger Documentation

```ruby
# spec/swagger_helper.rb
require 'rails_helper'

RSpec.configure do |config|
  config.swagger_root = Rails.root.join('swagger').to_s

  config.swagger_docs = {
    'v1/swagger.yaml' => {
      openapi: '3.0.1',
      info: {
        title: 'BookStore API',
        version: 'v1',
        description: 'RESTful API for BookStore application',
        contact: {
          name:  'API Support',
          email: 'api@bookstore.com'
        }
      },
      servers: [
        { url: 'http://localhost:3000', description: 'Development' },
        { url: 'https://api.bookstore.com', description: 'Production' }
      ],
      components: {
        securitySchemes: {
          bearerAuth: {
            type:   'http',
            scheme: 'bearer',
            bearerFormat: 'JWT'
          }
        },
        schemas: {
          Book: {
            type: 'object',
            properties: {
              id:           { type: 'integer' },
              title:        { type: 'string' },
              isbn:         { type: 'string' },
              price:        { type: 'number' },
              stock:        { type: 'integer' },
              language:     { type: 'string' },
              average_rating: { type: 'number' },
              in_stock:     { type: 'boolean' },
              category:     { '$ref': '#/components/schemas/Category' },
              authors:      { type: 'array', items: { '$ref': '#/components/schemas/Author' } }
            }
          },
          Category: {
            type: 'object',
            properties: {
              id:   { type: 'integer' },
              name: { type: 'string' },
              slug: { type: 'string' }
            }
          },
          Author: {
            type: 'object',
            properties: {
              id:      { type: 'integer' },
              name:    { type: 'string' },
              country: { type: 'string' }
            }
          },
          Error: {
            type: 'object',
            properties: {
              error:   { type: 'string' },
              message: { type: 'string' }
            }
          },
          Pagination: {
            type: 'object',
            properties: {
              current_page: { type: 'integer' },
              total_pages:  { type: 'integer' },
              total_count:  { type: 'integer' },
              per_page:     { type: 'integer' }
            }
          }
        }
      },
      security: [{ bearerAuth: [] }]
    }
  }

  config.swagger_format = :yaml
end

# spec/requests/api/v1/books_spec.rb
require 'swagger_helper'

RSpec.describe 'Books API', type: :request do
  path '/api/v1/books' do
    get 'List books' do
      tags    'Books'
      summary 'Returns paginated list of books'
      security []  # public endpoint - no auth required

      parameter name: :page,       in: :query, schema: { type: :integer }
      parameter name: :per_page,   in: :query, schema: { type: :integer }
      parameter name: :category_id, in: :query, schema: { type: :integer }
      parameter name: :language,   in: :query, schema: { type: :string }
      parameter name: :in_stock,   in: :query, schema: { type: :boolean }
      parameter name: :sort,       in: :query, schema: {
        type: :string,
        enum: ['newest', 'price_asc', 'price_desc', 'bestselling', 'title']
      }

      response '200', 'Books listed successfully' do
        schema type: :object,
               properties: {
                 books:      { type: :array, items: { '$ref': '#/components/schemas/Book' } },
                 pagination: { '$ref': '#/components/schemas/Pagination' }
               }

        let!(:books) { create_list(:book, 3) }
        run_test!
      end
    end

    post 'Create book' do
      tags        'Books'
      summary     'Create a new book (admin only)'
      consumes    'application/json'
      produces    'application/json'

      security [{ bearerAuth: [] }]

      parameter name: :book, in: :body, schema: {
        type: :object,
        required: [:title, :isbn, :price, :category_id],
        properties: {
          title:       { type: :string },
          isbn:        { type: :string },
          price:       { type: :number },
          stock:       { type: :integer },
          language:    { type: :string },
          description: { type: :string },
          category_id: { type: :integer }
        }
      }

      response '201', 'Book created' do
        let(:Authorization) { "Bearer #{admin_user.generate_auth_token}" }
        let(:book) { attributes_for(:book, category_id: create(:category).id) }
        run_test!
      end

      response '401', 'Unauthorized' do
        let(:Authorization) { 'Bearer invalid_token' }
        let(:book)          { attributes_for(:book) }
        run_test!
      end

      response '422', 'Validation failed' do
        let(:Authorization) { "Bearer #{admin_user.generate_auth_token}" }
        let(:book)          { { title: '' } }
        run_test!
      end
    end
  end

  path '/api/v1/books/{id}' do
    parameter name: :id, in: :path, type: :integer, required: true

    get 'Get book details' do
      tags    'Books'
      summary 'Returns book details'
      security []

      response '200', 'Book found' do
        schema '$ref': '#/components/schemas/Book'
        let(:id) { create(:book).id }
        run_test!
      end

      response '404', 'Book not found' do
        let(:id) { 999_999 }
        run_test!
      end
    end
  end
end
```

---

## ขั้นตอนที่ 11: CORS Configuration

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV.fetch('ALLOWED_ORIGINS', 'http://localhost:3001').split(',')

    resource '/api/*',
      headers:     :any,
      methods:     [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: false,
      max_age:     86400,
      expose:      ['X-Total-Count', 'X-Page', 'X-Per-Page']
  end

  allow do
    origins '*'
    resource '/health', headers: :any, methods: [:get]
  end
end
```

---

## ขั้นตอนที่ 12: Error Handling

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  rescue_from ActiveRecord::RecordNotFound do |e|
    render json: { error: 'Not Found', message: e.message }, status: :not_found
  end

  rescue_from ActiveRecord::RecordInvalid do |e|
    render json: {
      error:  'Validation Error',
      errors: e.record.errors.full_messages
    }, status: :unprocessable_entity
  end

  rescue_from ActionController::ParameterMissing do |e|
    render json: {
      error:   'Bad Request',
      message: "Required parameter missing: #{e.param}"
    }, status: :bad_request
  end

  rescue_from ActionController::UnpermittedParameters do |e|
    render json: {
      error:   'Bad Request',
      message: "Unpermitted parameters: #{e.params.join(', ')}"
    }, status: :bad_request
  end

  rescue_from StandardError do |e|
    Rails.logger.error "Unhandled error: #{e.class}: #{e.message}\n#{e.backtrace.first(5).join("\n")}"

    if Rails.env.production?
      render json: { error: 'Internal Server Error' }, status: :internal_server_error
    else
      render json: {
        error:     e.class.name,
        message:   e.message,
        backtrace: e.backtrace.first(10)
      }, status: :internal_server_error
    end
  end
end
```

---

## ขั้นตอนที่ 13: Testing

```ruby
# spec/requests/api/v1/orders_spec.rb
RSpec.describe 'Orders API', type: :request do
  let(:user)    { create(:user) }
  let(:token)   { user.generate_auth_token }
  let(:headers) {{ 'Authorization' => "Bearer #{token}" }}

  describe 'POST /api/v1/orders' do
    let(:book)   { create(:book, price: 299, stock: 10) }
    let(:params) {{
      items: [{ book_id: book.id, quantity: 2 }],
      shipping_address: {
        name:        'Test User',
        phone:       '0812345678',
        street:      '123 Main St',
        city:        'Bangkok',
        postal_code: '10100',
        country:     'TH'
      }
    }}

    it 'creates an order' do
      expect {
        post '/api/v1/orders', params: params, headers: headers
      }.to change(Order, :count).by(1)

      expect(response).to have_http_status(:created)
      json = response.parsed_body

      expect(json['order']['status']).to eq('pending')
      expect(json['order']['total'].to_f).to eq(598.0)
    end

    it 'decreases book stock' do
      expect {
        post '/api/v1/orders', params: params, headers: headers
      }.to change { book.reload.stock }.by(-2)
    end

    it 'returns error for insufficient stock' do
      book.update!(stock: 1)
      post '/api/v1/orders', params: params, headers: headers

      expect(response).to have_http_status(:unprocessable_entity)
      expect(response.parsed_body['error']).to include('not available')
    end

    it 'requires authentication' do
      post '/api/v1/orders', params: params
      expect(response).to have_http_status(:unauthorized)
    end
  end

  describe 'POST /api/v1/orders/:id/cancel' do
    let!(:order) { create(:order, user: user, status: 'pending') }

    it 'cancels pending order' do
      post "/api/v1/orders/#{order.id}/cancel", headers: headers

      expect(response).to have_http_status(:ok)
      expect(order.reload.status).to eq('cancelled')
    end

    it 'restores book stock on cancel' do
      book  = create(:book, stock: 5)
      item  = create(:order_item, order: order, book: book, quantity: 3)
      book.update!(stock: 2)  # simulate decreased stock

      post "/api/v1/orders/#{order.id}/cancel", headers: headers

      expect(book.reload.stock).to eq(5)
    end

    it 'cannot cancel shipped order' do
      order.update!(status: 'shipped')
      post "/api/v1/orders/#{order.id}/cancel", headers: headers

      expect(response).to have_http_status(:unprocessable_entity)
    end
  end
end
```

---

## ขั้นตอนที่ 14: Deployment to Heroku

```bash
# 1. ติดตั้ง Heroku CLI
# https://devcenter.heroku.com/articles/heroku-cli

# 2. Login
heroku login

# 3. สร้าง app
heroku create bookstore-api-prod

# 4. เพิ่ม add-ons
heroku addons:create heroku-postgresql:mini
heroku addons:create heroku-redis:mini
heroku addons:create papertrail:choklad  # logging

# 5. ตั้งค่า environment variables
heroku config:set RAILS_ENV=production
heroku config:set RAILS_MASTER_KEY=$(cat config/master.key)
heroku config:set JWT_SECRET=$(openssl rand -hex 64)
heroku config:set ALLOWED_ORIGINS=https://myfrontend.com,https://app.bookstore.com

# Sidekiq
heroku config:set REDIS_URL=$(heroku config:get REDIS_URL)

# 6. Procfile สำหรับ Heroku
cat > Procfile << 'EOF'
web:    bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -C config/sidekiq.yml
EOF

# 7. Deploy
git push heroku main

# 8. Run migrations
heroku run rails db:migrate

# 9. Seed data (optional)
heroku run rails db:seed

# 10. Scale dynos
heroku ps:scale web=1 worker=1

# 11. ตรวจสอบ
heroku logs --tail
heroku ps
heroku run rails c
```

### Procfile

```
web:    bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -C config/sidekiq.yml
release: bundle exec rails db:migrate
```

### config/puma.rb สำหรับ Heroku

```ruby
max_threads_count = ENV.fetch("RAILS_MAX_THREADS", 5)
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

port        ENV.fetch("PORT", 3000)
environment ENV.fetch("RAILS_ENV", "development")

workers_count = Integer(ENV.fetch("WEB_CONCURRENCY", 2))
workers workers_count if workers_count > 1

preload_app!

on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end

plugin :tmp_restart
```

---

## ขั้นตอนที่ 15: Response Caching

```ruby
# app/controllers/api/v1/books_controller.rb
def index
  cache_key = "books/index/#{params.to_h.sort.to_s}"

  cached_response = Rails.cache.fetch(cache_key, expires_in: 5.minutes) do
    books = Book.active.includes(:authors, :category)
    books = apply_filters(books)
    books = apply_sorting(books)
    books = books.page(params[:page]).per(params[:per_page] || 20)

    {
      books:      ActiveModelSerializers::SerializableResource.new(books).as_json,
      pagination: pagination_meta(books)
    }
  end

  render json: cached_response
end

# HTTP Caching Headers
def show
  @book = Book.find(params[:id])

  if stale?(etag: @book, last_modified: @book.updated_at, public: false)
    render json: { book: BookDetailSerializer.new(@book) }
  end
end
```

---

## สรุป

BookStore RESTful API ครอบคลุม:
- **Rails API mode**: lightweight controller สำหรับ JSON responses
- **JWT Authentication**: stateless auth ด้วย tokens
- **CRUD operations**: Books, Orders, Reviews
- **Pagination & Filtering**: Kaminari + query params
- **Rate Limiting**: Rack::Attack ป้องกัน abuse
- **API Versioning**: URL-based และ Header-based
- **Swagger Docs**: Interactive API documentation
- **Heroku Deployment**: Production-ready deployment

API นี้สามารถใช้เป็น backend สำหรับ mobile apps หรือ SPA frontends ได้ทันที
