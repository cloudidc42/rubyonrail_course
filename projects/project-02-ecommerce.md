# Project 2: E-Commerce Store - สร้างร้านค้าออนไลน์ด้วย Ruby on Rails

## เป้าหมายของโปรเจค

สร้าง E-Commerce application ที่มีฟีเจอร์ครบถ้วน ได้แก่:
- สินค้าพร้อม Variants (ขนาด, สี)
- ตะกร้าสินค้า
- Checkout flow
- ชำระเงินด้วย Stripe
- จัดการออร์เดอร์
- ติดตามสต็อก
- แจ้งเตือนทางอีเมล
- Admin Dashboard

---

## ขั้นตอนที่ 1: สร้าง Project

```bash
rails new ecommerce_app \
  --database=postgresql \
  --css=tailwind \
  --javascript=importmap

cd ecommerce_app
```

### Gemfile

```ruby
# Gemfile
source "https://rubygems.org"

ruby "3.2.2"

gem "rails", "~> 7.1.0"
gem "pg", "~> 1.1"
gem "puma", ">= 5.0"
gem "importmap-rails"
gem "turbo-rails"
gem "stimulus-rails"
gem "tailwindcss-rails"
gem "jbuilder"

# Authentication
gem "devise", "~> 4.9"

# Authorization
gem "pundit", "~> 2.3"

# Payment
gem "stripe", "~> 10.0"
gem "money-rails", "~> 1.15"

# File uploads
gem "image_processing", "~> 1.2"
gem "aws-sdk-s3", require: false

# Pagination
gem "pagy", "~> 6.0"

# Search
gem "pg_search"

# Background jobs
gem "sidekiq", "~> 7.0"
gem "redis", "~> 5.0"
gem "sidekiq-scheduler"

# PDF generation
gem "prawn"
gem "prawn-table"

# Barcode/QR
gem "rqrcode"

# Admin
gem "administrate", "~> 0.20"

# Monitoring
gem "sentry-ruby"
gem "sentry-rails"
gem "sentry-sidekiq"

# Security
gem "rack-attack"

# Charts
gem "chartkick"
gem "groupdate"

group :development, :test do
  gem "rspec-rails", "~> 6.0"
  gem "factory_bot_rails"
  gem "faker"
  gem "byebug"
  gem "stripe-ruby-mock", require: "stripe_mock"
end

group :development do
  gem "web-console"
  gem "rack-mini-profiler"
  gem "letter_opener"
  gem "rubocop-rails", require: false
  gem "brakeman", require: false
  gem "annotate"
end

group :test do
  gem "capybara"
  gem "selenium-webdriver"
  gem "shoulda-matchers"
  gem "simplecov"
  gem "vcr"
  gem "webmock"
end
```

```bash
bundle install
rails db:create
```

---

## ขั้นตอนที่ 2: Database Design

### Migrations

```bash
# Users (Devise)
rails generate devise:install
rails generate devise User

# Addresses
rails generate model Address \
  user:references \
  first_name:string \
  last_name:string \
  address_line1:string \
  address_line2:string \
  city:string \
  state:string \
  postal_code:string \
  country:string:default:TH \
  phone:string \
  is_default:boolean:default:false

# Categories
rails generate model Category \
  name:string \
  slug:string \
  description:text \
  parent:references \
  position:integer:default:0

# Products
rails generate model Product \
  name:string \
  slug:string \
  description:text \
  status:integer:default:0 \
  category:references \
  base_price_cents:integer:default:0 \
  compare_at_price_cents:integer \
  sku_prefix:string \
  weight:decimal \
  featured:boolean:default:false

# Product Variants
rails generate model ProductVariant \
  product:references \
  sku:string \
  price_cents:integer:default:0 \
  compare_at_price_cents:integer \
  stock_quantity:integer:default:0 \
  weight:decimal \
  position:integer:default:0

# Variant Options (เช่น Color=Red, Size=M)
rails generate model VariantOption \
  product_variant:references \
  name:string \
  value:string

# Carts
rails generate model Cart \
  user:references \
  session_id:string

# Cart Items
rails generate model CartItem \
  cart:references \
  product_variant:references \
  quantity:integer:default:1

# Orders
rails generate model Order \
  user:references \
  order_number:string \
  status:integer:default:0 \
  subtotal_cents:integer:default:0 \
  tax_cents:integer:default:0 \
  shipping_cents:integer:default:0 \
  discount_cents:integer:default:0 \
  total_cents:integer:default:0 \
  currency:string:default:THB \
  payment_status:integer:default:0 \
  stripe_payment_intent_id:string \
  notes:text

# Order Items
rails generate model OrderItem \
  order:references \
  product_variant:references \
  quantity:integer \
  unit_price_cents:integer \
  total_price_cents:integer \
  product_name:string \
  variant_description:string

# Shipping Addresses (snapshot at order time)
rails generate model OrderAddress \
  order:references \
  first_name:string \
  last_name:string \
  address_line1:string \
  address_line2:string \
  city:string \
  state:string \
  postal_code:string \
  country:string \
  phone:string

# Coupons
rails generate model Coupon \
  code:string \
  discount_type:integer:default:0 \
  discount_value:decimal \
  minimum_order_cents:integer:default:0 \
  usage_limit:integer \
  used_count:integer:default:0 \
  expires_at:datetime \
  active:boolean:default:true

# Reviews
rails generate model Review \
  user:references \
  product:references \
  rating:integer \
  title:string \
  body:text \
  verified_purchase:boolean:default:false

# Inventory Movements
rails generate model InventoryMovement \
  product_variant:references \
  quantity:integer \
  movement_type:string \
  reference_type:string \
  reference_id:integer \
  notes:string
```

### Indexes Migration

```ruby
# db/migrate/xxxx_add_all_indexes.rb
class AddAllIndexes < ActiveRecord::Migration[7.1]
  def change
    # Products
    add_index :products, :slug, unique: true
    add_index :products, :status
    add_index :products, :featured
    add_index :products, [:category_id, :status]

    # Product Variants
    add_index :product_variants, :sku, unique: true
    add_index :product_variants, [:product_id, :position]

    # Carts
    add_index :carts, :session_id
    add_index :carts, [:user_id, :session_id]

    # Cart Items
    add_index :cart_items, [:cart_id, :product_variant_id], unique: true

    # Orders
    add_index :orders, :order_number, unique: true
    add_index :orders, :status
    add_index :orders, :payment_status
    add_index :orders, :stripe_payment_intent_id
    add_index :orders, [:user_id, :created_at]

    # Coupons
    add_index :coupons, :code, unique: true

    # Reviews
    add_index :reviews, [:product_id, :rating]
    add_index :reviews, [:user_id, :product_id], unique: true

    # Categories
    add_index :categories, :slug, unique: true
    add_index :categories, [:parent_id, :position]
  end
end
```

```bash
rails db:migrate
```

---

## ขั้นตอนที่ 3: Models

### Product Model

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  include PgSearch::Model

  belongs_to :category, optional: true
  has_many :product_variants, -> { order(:position) }, dependent: :destroy
  has_many :order_items, through: :product_variants
  has_many :reviews, dependent: :destroy

  has_many_attached :images

  # Money
  monetize :base_price_cents, with_model_currency: :currency
  monetize :compare_at_price_cents, with_model_currency: :currency

  # Enum
  enum status: { draft: 0, active: 1, archived: 2 }

  # Callbacks
  before_validation :generate_slug, if: :name_changed?

  # Validations
  validates :name, presence: true, length: { minimum: 2, maximum: 200 }
  validates :slug, presence: true, uniqueness: true
  validates :base_price_cents, numericality: { greater_than_or_equal_to: 0 }

  # Search
  pg_search_scope :search_by_keyword,
    against: { name: "A", description: "B" },
    using: {
      tsearch: { prefix: true },
      trigram: { threshold: 0.3 }
    }

  # Scopes
  scope :active, -> { where(status: :active) }
  scope :featured, -> { where(featured: true) }
  scope :in_stock, -> {
    joins(:product_variants)
      .where("product_variants.stock_quantity > 0")
      .distinct
  }
  scope :by_price_range, ->(min, max) {
    where(base_price_cents: (min * 100)..(max * 100))
  }

  def to_param
    slug
  end

  def in_stock?
    product_variants.any? { |v| v.stock_quantity > 0 }
  end

  def total_stock
    product_variants.sum(:stock_quantity)
  end

  def average_rating
    reviews.average(:rating)&.round(1) || 0
  end

  def on_sale?
    compare_at_price_cents.present? && compare_at_price_cents > base_price_cents
  end

  def discount_percentage
    return 0 unless on_sale?
    ((1 - base_price_cents.to_f / compare_at_price_cents) * 100).round
  end

  def primary_image
    images.first
  end

  private

  def generate_slug
    return if name.blank?
    base = name.parameterize
    slug = base
    counter = 1
    while Product.where(slug: slug).where.not(id: id).exists?
      slug = "#{base}-#{counter}"
      counter += 1
    end
    self.slug = slug
  end
end
```

### ProductVariant Model

```ruby
# app/models/product_variant.rb
class ProductVariant < ApplicationRecord
  belongs_to :product
  has_many :variant_options, dependent: :destroy
  has_many :cart_items, dependent: :destroy
  has_many :order_items
  has_many :inventory_movements, dependent: :destroy

  monetize :price_cents, with_model_currency: :currency
  monetize :compare_at_price_cents, allow_nil: true, with_model_currency: :currency

  accepts_nested_attributes_for :variant_options, allow_destroy: true

  before_validation :generate_sku, if: :new_record?

  validates :sku, presence: true, uniqueness: true
  validates :price_cents, numericality: { greater_than_or_equal_to: 0 }
  validates :stock_quantity, numericality: { greater_than_or_equal_to: 0 }

  scope :in_stock, -> { where("stock_quantity > 0") }
  scope :out_of_stock, -> { where(stock_quantity: 0) }
  scope :low_stock, -> { where("stock_quantity > 0 AND stock_quantity <= 5") }

  def display_name
    opts = variant_options.map { |o| "#{o.name}: #{o.value}" }.join(", ")
    opts.present? ? "#{product.name} (#{opts})" : product.name
  end

  def in_stock?
    stock_quantity > 0
  end

  def low_stock?
    stock_quantity > 0 && stock_quantity <= 5
  end

  def reserve_stock!(qty)
    raise "สต็อกไม่เพียงพอ" if stock_quantity < qty
    decrement!(:stock_quantity, qty)
    inventory_movements.create!(
      quantity: -qty,
      movement_type: "sale",
      notes: "Reserved for order"
    )
  end

  def return_stock!(qty, reason: "return")
    increment!(:stock_quantity, qty)
    inventory_movements.create!(
      quantity: qty,
      movement_type: reason,
      notes: "Stock returned"
    )
  end

  private

  def generate_sku
    prefix = product&.sku_prefix || "SKU"
    self.sku = "#{prefix}-#{SecureRandom.hex(4).upcase}"
  end
end
```

### Cart Model

```ruby
# app/models/cart.rb
class Cart < ApplicationRecord
  belongs_to :user, optional: true
  has_many :cart_items, dependent: :destroy
  has_many :product_variants, through: :cart_items

  def add_item(variant, quantity: 1)
    item = cart_items.find_or_initialize_by(product_variant: variant)
    item.quantity = (item.quantity || 0) + quantity
    item.save!
    item
  end

  def remove_item(variant)
    cart_items.where(product_variant: variant).destroy_all
  end

  def update_quantity(variant, quantity)
    if quantity <= 0
      remove_item(variant)
    else
      item = cart_items.find_by!(product_variant: variant)
      item.update!(quantity: quantity)
    end
  end

  def clear!
    cart_items.destroy_all
  end

  def subtotal_cents
    cart_items.sum { |item| item.total_price_cents }
  end

  def subtotal
    Money.new(subtotal_cents, "THB")
  end

  def item_count
    cart_items.sum(:quantity)
  end

  def empty?
    cart_items.empty?
  end

  def merge_with!(other_cart)
    other_cart.cart_items.each do |item|
      add_item(item.product_variant, quantity: item.quantity)
    end
    other_cart.destroy
  end
end
```

### Order Model

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  belongs_to :user
  has_many :order_items, dependent: :destroy
  has_one :order_address, dependent: :destroy

  monetize :subtotal_cents, with_model_currency: :currency
  monetize :tax_cents, with_model_currency: :currency
  monetize :shipping_cents, with_model_currency: :currency
  monetize :discount_cents, with_model_currency: :currency
  monetize :total_cents, with_model_currency: :currency

  enum status: {
    pending: 0,
    confirmed: 1,
    processing: 2,
    shipped: 3,
    delivered: 4,
    cancelled: 5,
    refunded: 6
  }

  enum payment_status: {
    unpaid: 0,
    paid: 1,
    partially_refunded: 2,
    fully_refunded: 3
  }

  before_create :generate_order_number

  validates :order_number, presence: true, uniqueness: true
  validates :status, presence: true
  validates :payment_status, presence: true

  scope :recent, -> { order(created_at: :desc) }
  scope :this_month, -> { where(created_at: Time.current.beginning_of_month..Time.current.end_of_month) }
  scope :revenue, -> { paid.where.not(status: [:cancelled, :refunded]) }

  def calculate_totals!(coupon: nil)
    self.subtotal_cents = order_items.sum(:total_price_cents)
    self.tax_cents = (subtotal_cents * 0.07).round  # VAT 7%
    self.shipping_cents = calculate_shipping
    self.discount_cents = coupon ? calculate_discount(coupon) : 0
    self.total_cents = subtotal_cents + tax_cents + shipping_cents - discount_cents
    save!
  end

  def cancel!
    return false unless can_cancel?
    transaction do
      update!(status: :cancelled)
      order_items.each do |item|
        item.product_variant.return_stock!(item.quantity, reason: "cancellation")
      end
      OrderMailer.order_cancelled(self).deliver_later
    end
    true
  end

  def can_cancel?
    pending? || confirmed?
  end

  def total_items
    order_items.sum(:quantity)
  end

  def tracking_url
    "https://track.example.com/#{tracking_number}" if tracking_number.present?
  end

  private

  def generate_order_number
    loop do
      num = "ORD-#{Time.current.strftime('%Y%m')}-#{SecureRandom.hex(4).upcase}"
      unless Order.exists?(order_number: num)
        self.order_number = num
        break
      end
    end
  end

  def calculate_shipping
    return 0 if subtotal_cents >= 50000  # Free shipping over 500 THB
    7000  # 70 THB flat rate
  end

  def calculate_discount(coupon)
    case coupon.discount_type.to_sym
    when :percentage
      (subtotal_cents * coupon.discount_value / 100).round
    when :fixed
      [coupon.discount_value * 100, subtotal_cents].min.to_i
    else
      0
    end
  end
end
```

---

## ขั้นตอนที่ 4: Controllers

### Products Controller

```ruby
# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  include Pagy::Backend

  def index
    @products = Product.active.includes(:product_variants, :category, images_attachments: :blob)

    # Filtering
    @products = @products.search_by_keyword(params[:q]) if params[:q].present?
    @products = @products.joins(:category).where(categories: { slug: params[:category] }) if params[:category].present?
    @products = @products.by_price_range(params[:min_price].to_i, params[:max_price].to_i) if params[:min_price].present? && params[:max_price].present?
    @products = @products.in_stock if params[:in_stock].present?

    # Sorting
    @products = case params[:sort]
                when "price_asc"   then @products.order(base_price_cents: :asc)
                when "price_desc"  then @products.order(base_price_cents: :desc)
                when "newest"      then @products.order(created_at: :desc)
                when "popular"     then @products.joins(:order_items).group("products.id").order("COUNT(order_items.id) DESC")
                else                    @products.order(featured: :desc, created_at: :desc)
                end

    @pagy, @products = pagy(@products, items: 12)
    @categories = Category.where(parent_id: nil).includes(:children)
  end

  def show
    @product = Product.active.find_by!(slug: params[:id])
    @variants = @product.product_variants.includes(:variant_options)
    @reviews = @product.reviews.includes(:user).order(created_at: :desc)
    @related = Product.active.where(category: @product.category)
                      .where.not(id: @product.id)
                      .limit(4)
  end
end
```

### Carts Controller

```ruby
# app/controllers/carts_controller.rb
class CartsController < ApplicationController
  before_action :set_cart

  def show
    @cart_items = @cart.cart_items.includes(product_variant: [:product, :variant_options, { product: { images_attachments: :blob } }])
  end

  def add_item
    variant = ProductVariant.find(params[:variant_id])
    quantity = params[:quantity].to_i.clamp(1, 10)

    if variant.stock_quantity >= quantity
      @cart.add_item(variant, quantity: quantity)
      respond_to do |format|
        format.turbo_stream do
          render turbo_stream: [
            turbo_stream.update("cart-count", @cart.item_count),
            turbo_stream.update("cart-mini", partial: "layouts/cart_mini", locals: { cart: @cart })
          ]
        end
        format.html { redirect_back fallback_location: root_path, notice: "เพิ่มสินค้าลงตะกร้าแล้ว" }
      end
    else
      render json: { error: "สต็อกไม่เพียงพอ" }, status: :unprocessable_entity
    end
  end

  def update_item
    variant = ProductVariant.find(params[:variant_id])
    quantity = params[:quantity].to_i

    @cart.update_quantity(variant, quantity)

    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: [
          turbo_stream.update("cart-count", @cart.item_count),
          turbo_stream.replace("cart-item-#{variant.id}", partial: "carts/cart_item",
                               locals: { item: @cart.cart_items.find_by(product_variant: variant) }),
          turbo_stream.update("cart-subtotal", @cart.subtotal.format)
        ]
      end
    end
  end

  def remove_item
    variant = ProductVariant.find(params[:variant_id])
    @cart.remove_item(variant)

    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: [
          turbo_stream.remove("cart-item-#{variant.id}"),
          turbo_stream.update("cart-count", @cart.item_count),
          turbo_stream.update("cart-subtotal", @cart.subtotal.format)
        ]
      end
    end
  end

  private

  def set_cart
    @cart = current_cart
  end
end
```

### Checkout Controller

```ruby
# app/controllers/checkouts_controller.rb
class CheckoutsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_cart
  before_action :ensure_cart_not_empty

  def index
    @order = Order.new
    @address = current_user.addresses.default.first || Address.new
    @coupon_code = session[:coupon_code]
    @coupon = Coupon.find_by(code: @coupon_code) if @coupon_code.present?

    calculate_totals
  end

  def create
    result = Orders::CreateService.call(
      user: current_user,
      cart: @cart,
      address_params: address_params,
      coupon_code: session[:coupon_code]
    )

    if result.success?
      @order = result.order
      session[:last_order_id] = @order.id
      session.delete(:coupon_code)

      redirect_to payment_order_path(@order)
    else
      @order = result.order
      flash[:alert] = result.errors.join(", ")
      render :index, status: :unprocessable_entity
    end
  end

  def apply_coupon
    coupon = Coupon.find_by(code: params[:coupon_code]&.upcase)

    if coupon&.valid_for_use?
      session[:coupon_code] = coupon.code
      flash[:notice] = "ใช้คูปองสำเร็จ! ประหยัด #{coupon.discount_description}"
    else
      session.delete(:coupon_code)
      flash[:alert] = "คูปองไม่ถูกต้องหรือหมดอายุแล้ว"
    end

    redirect_to checkout_path
  end

  private

  def set_cart
    @cart = current_cart
  end

  def ensure_cart_not_empty
    redirect_to cart_path, alert: "ตะกร้าสินค้าว่างเปล่า" if @cart.empty?
  end

  def address_params
    params.require(:address).permit(
      :first_name, :last_name, :address_line1, :address_line2,
      :city, :state, :postal_code, :country, :phone, :save_address
    )
  end

  def calculate_totals
    @subtotal = @cart.subtotal
    @tax = Money.new((@cart.subtotal_cents * 0.07).round, "THB")
    @shipping = @cart.subtotal_cents >= 50000 ? Money.new(0, "THB") : Money.new(7000, "THB")
    @discount = @coupon ? calculate_coupon_discount(@coupon) : Money.new(0, "THB")
    @total = @subtotal + @tax + @shipping - @discount
  end

  def calculate_coupon_discount(coupon)
    case coupon.discount_type.to_sym
    when :percentage
      Money.new((@cart.subtotal_cents * coupon.discount_value / 100).round, "THB")
    when :fixed
      Money.new([coupon.discount_value * 100, @cart.subtotal_cents].min.to_i, "THB")
    end
  end
end
```

### Payments Controller

```ruby
# app/controllers/payments_controller.rb
class PaymentsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_order

  def show
    # สร้าง PaymentIntent ใน Stripe
    @payment_intent = Stripe::PaymentIntent.create(
      amount: @order.total_cents,
      currency: @order.currency.downcase,
      metadata: {
        order_id: @order.id,
        order_number: @order.order_number,
        user_id: current_user.id
      }
    )

    @order.update!(stripe_payment_intent_id: @payment_intent.id)
    @stripe_public_key = ENV['STRIPE_PUBLIC_KEY']
  end

  def confirm
    payment_intent = Stripe::PaymentIntent.retrieve(@order.stripe_payment_intent_id)

    if payment_intent.status == "succeeded"
      @order.update!(payment_status: :paid, status: :confirmed)

      # ส่งอีเมลยืนยัน
      OrderMailer.order_confirmed(@order).deliver_later

      # ลดสต็อก
      @order.order_items.each do |item|
        item.product_variant.reserve_stock!(item.quantity)
      end

      # ล้างตะกร้า
      current_cart.clear!

      redirect_to order_path(@order), notice: "ชำระเงินสำเร็จ! ขอบคุณที่สั่งซื้อ"
    else
      redirect_to payment_order_path(@order), alert: "การชำระเงินไม่สำเร็จ กรุณาลองใหม่"
    end
  rescue Stripe::StripeError => e
    redirect_to payment_order_path(@order), alert: "เกิดข้อผิดพลาด: #{e.message}"
  end

  private

  def set_order
    @order = current_user.orders.find(params[:order_id])
  end
end
```

### Stripe Webhooks Controller

```ruby
# app/controllers/stripe_webhooks_controller.rb
class StripeWebhooksController < ActionController::API
  def create
    payload = request.body.read
    sig_header = request.env['HTTP_STRIPE_SIGNATURE']
    endpoint_secret = ENV['STRIPE_WEBHOOK_SECRET']

    begin
      event = Stripe::Webhook.construct_event(payload, sig_header, endpoint_secret)
    rescue JSON::ParserError
      return render plain: "Invalid payload", status: :bad_request
    rescue Stripe::SignatureVerificationError
      return render plain: "Invalid signature", status: :bad_request
    end

    case event.type
    when "payment_intent.succeeded"
      handle_payment_succeeded(event.data.object)
    when "payment_intent.payment_failed"
      handle_payment_failed(event.data.object)
    when "charge.refunded"
      handle_charge_refunded(event.data.object)
    end

    render plain: "OK", status: :ok
  end

  private

  def handle_payment_succeeded(payment_intent)
    order = Order.find_by(stripe_payment_intent_id: payment_intent.id)
    return unless order

    order.transaction do
      order.update!(payment_status: :paid, status: :confirmed)
      order.order_items.each do |item|
        item.product_variant.reserve_stock!(item.quantity)
      end
      OrderMailer.order_confirmed(order).deliver_later
    end
  end

  def handle_payment_failed(payment_intent)
    order = Order.find_by(stripe_payment_intent_id: payment_intent.id)
    return unless order

    order.update!(payment_status: :unpaid)
    OrderMailer.payment_failed(order).deliver_later
  end

  def handle_charge_refunded(charge)
    payment_intent = Stripe::PaymentIntent.retrieve(charge.payment_intent)
    order = Order.find_by(stripe_payment_intent_id: payment_intent.id)
    return unless order

    refunded_amount = charge.amount_refunded
    if refunded_amount >= order.total_cents
      order.update!(status: :refunded, payment_status: :fully_refunded)
    else
      order.update!(payment_status: :partially_refunded)
    end

    OrderMailer.order_refunded(order).deliver_later
  end
end
```

### Orders Controller

```ruby
# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  include Pagy::Backend
  before_action :authenticate_user!
  before_action :set_order, only: [:show, :cancel, :invoice]

  def index
    @pagy, @orders = pagy(
      current_user.orders.includes(:order_items, :order_address).recent,
      items: 10
    )
  end

  def show
    @order_items = @order.order_items.includes(product_variant: [:product, :variant_options])
  end

  def cancel
    if @order.cancel!
      redirect_to @order, notice: "ยกเลิกออร์เดอร์แล้ว"
    else
      redirect_to @order, alert: "ไม่สามารถยกเลิกออร์เดอร์นี้ได้"
    end
  end

  def invoice
    pdf = InvoicePdf.new(@order)
    send_data pdf.render,
      filename: "invoice-#{@order.order_number}.pdf",
      type: "application/pdf",
      disposition: "inline"
  end

  private

  def set_order
    @order = current_user.orders.find(params[:id])
  end
end
```

---

## ขั้นตอนที่ 5: Service Objects

### Orders::CreateService

```ruby
# app/services/orders/create_service.rb
module Orders
  class CreateService
    Result = Struct.new(:success?, :order, :errors)

    def self.call(**args)
      new(**args).call
    end

    def initialize(user:, cart:, address_params:, coupon_code: nil)
      @user = user
      @cart = cart
      @address_params = address_params
      @coupon_code = coupon_code
      @errors = []
    end

    def call
      validate_cart
      return failure if @errors.any?

      ActiveRecord::Base.transaction do
        coupon = find_and_validate_coupon
        order = build_order(coupon)
        create_order_items(order)
        create_order_address(order)
        order.calculate_totals!(coupon: coupon)
        coupon&.increment!(:used_count)

        Result.new(true, order, [])
      end
    rescue ActiveRecord::RecordInvalid => e
      failure(e.record.errors.full_messages)
    rescue => e
      Rails.logger.error "Order creation failed: #{e.message}"
      failure(["เกิดข้อผิดพลาดในการสร้างออร์เดอร์"])
    end

    private

    def validate_cart
      @errors << "ตะกร้าสินค้าว่างเปล่า" if @cart.empty?

      @cart.cart_items.includes(:product_variant).each do |item|
        variant = item.product_variant
        unless variant.stock_quantity >= item.quantity
          @errors << "#{variant.display_name} มีสต็อกเหลือแค่ #{variant.stock_quantity} ชิ้น"
        end
      end
    end

    def find_and_validate_coupon
      return nil if @coupon_code.blank?

      coupon = Coupon.find_by(code: @coupon_code.upcase)
      unless coupon&.valid_for_use?
        @errors << "คูปองไม่ถูกต้อง"
        raise ActiveRecord::Rollback
      end

      coupon
    end

    def build_order(coupon)
      @user.orders.create!(
        status: :pending,
        payment_status: :unpaid,
        currency: "THB"
      )
    end

    def create_order_items(order)
      @cart.cart_items.includes(product_variant: [:product, :variant_options]).each do |item|
        variant = item.product_variant
        order.order_items.create!(
          product_variant: variant,
          quantity: item.quantity,
          unit_price_cents: variant.price_cents,
          total_price_cents: variant.price_cents * item.quantity,
          product_name: variant.product.name,
          variant_description: variant.variant_options.map { |o| "#{o.name}: #{o.value}" }.join(", ")
        )
      end
    end

    def create_order_address(order)
      source_address = if @address_params[:address_id].present?
                         @user.addresses.find(@address_params[:address_id])
                       else
                         build_address_from_params
                       end

      order.create_order_address!(
        first_name: source_address[:first_name],
        last_name: source_address[:last_name],
        address_line1: source_address[:address_line1],
        address_line2: source_address[:address_line2],
        city: source_address[:city],
        state: source_address[:state],
        postal_code: source_address[:postal_code],
        country: source_address[:country] || "TH",
        phone: source_address[:phone]
      )

      if @address_params[:save_address] == "1"
        @user.addresses.create!(source_address.except(:save_address, :address_id))
      end
    end

    def build_address_from_params
      @address_params.to_h.symbolize_keys
    end

    def failure(errors = @errors)
      Result.new(false, nil, errors)
    end
  end
end
```

---

## ขั้นตอนที่ 6: Views

### Layout

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= content_for?(:title) ? "#{yield :title} | ShopRails" : "ShopRails - ร้านค้าออนไลน์" %></title>
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= stylesheet_link_tag "application", media: "all" %>
  <%= javascript_importmap_tags %>
</head>
<body class="bg-gray-50" data-controller="cart">
  <%= render "layouts/header" %>
  
  <main class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
    <%= render "layouts/flash" %>
    <%= yield %>
  </main>
  
  <%= render "layouts/footer" %>
</body>
</html>
```

```erb
<%# app/views/layouts/_header.html.erb %>
<header class="bg-white shadow-sm sticky top-0 z-40">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex items-center justify-between h-16">
      
      <!-- Logo -->
      <%= link_to root_path, class: "flex items-center" do %>
        <span class="text-2xl font-bold text-indigo-600">🛒 ShopRails</span>
      <% end %>
      
      <!-- Search -->
      <div class="flex-1 max-w-lg mx-8">
        <%= form_with url: products_path, method: :get, class: "relative" do |f| %>
          <%= f.text_field :q, value: params[:q],
              placeholder: "ค้นหาสินค้า...",
              class: "w-full pl-10 pr-4 py-2 border border-gray-300 rounded-full focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
          <div class="absolute left-3 top-2.5 text-gray-400">🔍</div>
        <% end %>
      </div>
      
      <!-- Actions -->
      <div class="flex items-center space-x-4">
        <!-- Wishlist -->
        <button class="text-gray-500 hover:text-indigo-600">❤️</button>
        
        <!-- Cart -->
        <%= link_to cart_path, class: "relative text-gray-500 hover:text-indigo-600" do %>
          🛒
          <span id="cart-count"
                class="absolute -top-2 -right-2 bg-red-500 text-white text-xs rounded-full w-5 h-5 flex items-center justify-center">
            <%= current_cart.item_count %>
          </span>
        <% end %>
        
        <!-- User -->
        <% if user_signed_in? %>
          <div class="relative" data-controller="dropdown">
            <button data-action="click->dropdown#toggle" class="flex items-center space-x-1">
              <span class="text-sm text-gray-700">สวัสดี <%= current_user.first_name %></span>
              <span>▼</span>
            </button>
            <div data-dropdown-target="menu" class="hidden absolute right-0 mt-2 w-48 bg-white rounded-md shadow-lg">
              <%= link_to "ออร์เดอร์ของฉัน", orders_path, class: "block px-4 py-2 text-sm text-gray-700 hover:bg-gray-50" %>
              <%= link_to "ที่อยู่", addresses_path, class: "block px-4 py-2 text-sm text-gray-700 hover:bg-gray-50" %>
              <%= link_to "ตั้งค่าบัญชี", edit_user_registration_path, class: "block px-4 py-2 text-sm text-gray-700 hover:bg-gray-50" %>
              <hr>
              <%= button_to "ออกจากระบบ", destroy_user_session_path, method: :delete,
                  class: "block w-full text-left px-4 py-2 text-sm text-red-600 hover:bg-gray-50" %>
            </div>
          </div>
        <% else %>
          <%= link_to "เข้าสู่ระบบ", new_user_session_path, class: "text-sm text-gray-600 hover:text-indigo-600" %>
          <%= link_to "สมัครสมาชิก", new_user_registration_path,
              class: "bg-indigo-600 text-white px-4 py-2 rounded-md text-sm hover:bg-indigo-700" %>
        <% end %>
      </div>
    </div>
    
    <!-- Category Nav -->
    <nav class="border-t flex space-x-6 py-2 text-sm">
      <% Category.where(parent_id: nil).each do |category| %>
        <%= link_to category.name, products_path(category: category.slug),
            class: "text-gray-600 hover:text-indigo-600" %>
      <% end %>
    </nav>
  </div>
</header>
```

### Product Index View

```erb
<%# app/views/products/index.html.erb %>
<% content_for :title, "สินค้าทั้งหมด" %>

<div class="flex gap-8">
  
  <!-- Sidebar Filter -->
  <aside class="w-64 flex-shrink-0">
    <div class="bg-white rounded-lg shadow-sm p-6 space-y-6">
      <h3 class="font-semibold text-gray-900">ตัวกรอง</h3>
      
      <%= form_with url: products_path, method: :get, data: { turbo_frame: "products-grid" } do |f| %>
        
        <!-- Category -->
        <div>
          <h4 class="text-sm font-medium text-gray-700 mb-3">หมวดหมู่</h4>
          <% @categories.each do |cat| %>
            <label class="flex items-center space-x-2 mb-2">
              <%= f.radio_button :category, cat.slug, checked: params[:category] == cat.slug %>
              <span class="text-sm text-gray-600"><%= cat.name %></span>
            </label>
          <% end %>
        </div>
        
        <!-- Price Range -->
        <div>
          <h4 class="text-sm font-medium text-gray-700 mb-3">ช่วงราคา</h4>
          <div class="flex items-center space-x-2">
            <%= f.number_field :min_price, value: params[:min_price], placeholder: "ต่ำสุด",
                class: "w-full text-sm border rounded p-2" %>
            <span>-</span>
            <%= f.number_field :max_price, value: params[:max_price], placeholder: "สูงสุด",
                class: "w-full text-sm border rounded p-2" %>
          </div>
        </div>
        
        <!-- In Stock -->
        <div>
          <label class="flex items-center space-x-2">
            <%= f.check_box :in_stock, checked: params[:in_stock].present? %>
            <span class="text-sm text-gray-600">มีสินค้าเท่านั้น</span>
          </label>
        </div>
        
        <%= f.submit "กรอง", class: "w-full bg-indigo-600 text-white rounded-md py-2 text-sm hover:bg-indigo-700" %>
      <% end %>
    </div>
  </aside>
  
  <!-- Main Content -->
  <div class="flex-1">
    <!-- Sort & Results -->
    <div class="flex items-center justify-between mb-6">
      <p class="text-sm text-gray-500">พบ <%= @pagy.count %> สินค้า</p>
      
      <%= form_with url: products_path, method: :get, class: "flex items-center space-x-2" do |f| %>
        <%= f.hidden_field :q, value: params[:q] %>
        <%= f.hidden_field :category, value: params[:category] %>
        <label class="text-sm text-gray-500">เรียงโดย:</label>
        <%= f.select :sort, [
              ["แนะนำ", ""],
              ["ราคาต่ำ-สูง", "price_asc"],
              ["ราคาสูง-ต่ำ", "price_desc"],
              ["ใหม่ล่าสุด", "newest"],
              ["ยอดนิยม", "popular"]
            ], { selected: params[:sort] },
            class: "text-sm border rounded-md p-1",
            onchange: "this.form.submit()" %>
      <% end %>
    </div>
    
    <!-- Products Grid -->
    <%= turbo_frame_tag "products-grid" do %>
      <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
        <% @products.each do |product| %>
          <%= render "products/card", product: product %>
        <% end %>
      </div>
      
      <!-- Pagination -->
      <div class="mt-8 flex justify-center">
        <%== pagy_nav(@pagy) %>
      </div>
    <% end %>
  </div>
</div>
```

```erb
<%# app/views/products/_card.html.erb %>
<div class="bg-white rounded-lg shadow-sm overflow-hidden hover:shadow-md transition-shadow group">
  <%= link_to product_path(product) do %>
    <div class="relative aspect-square overflow-hidden bg-gray-100">
      <% if product.images.attached? %>
        <%= image_tag product.primary_image.variant(resize_to_fill: [300, 300]),
            class: "w-full h-full object-cover group-hover:scale-105 transition-transform duration-300" %>
      <% else %>
        <div class="w-full h-full flex items-center justify-center text-gray-400 text-5xl">🛍️</div>
      <% end %>
      
      <% if product.on_sale? %>
        <span class="absolute top-2 left-2 bg-red-500 text-white text-xs px-2 py-1 rounded">
          -<%= product.discount_percentage %>%
        </span>
      <% end %>
      
      <% unless product.in_stock? %>
        <div class="absolute inset-0 bg-black bg-opacity-40 flex items-center justify-center">
          <span class="text-white font-bold">หมดสต็อก</span>
        </div>
      <% end %>
    </div>
  <% end %>
  
  <div class="p-4">
    <p class="text-xs text-indigo-600 mb-1"><%= product.category&.name %></p>
    <h3 class="text-sm font-medium text-gray-900 line-clamp-2 mb-2">
      <%= link_to product.name, product_path(product), class: "hover:text-indigo-600" %>
    </h3>
    
    <!-- Rating -->
    <div class="flex items-center mb-2">
      <% rating = product.average_rating %>
      <% 5.times do |i| %>
        <span class="text-xs <%= i < rating.floor ? 'text-yellow-400' : 'text-gray-300' %>">★</span>
      <% end %>
      <span class="text-xs text-gray-400 ml-1">(<%= product.reviews.count %>)</span>
    </div>
    
    <!-- Price -->
    <div class="flex items-center space-x-2">
      <span class="font-bold text-gray-900">
        <%= Money.new(product.base_price_cents, "THB").format %>
      </span>
      <% if product.on_sale? %>
        <span class="text-sm text-gray-400 line-through">
          <%= Money.new(product.compare_at_price_cents, "THB").format %>
        </span>
      <% end %>
    </div>
    
    <!-- Add to Cart -->
    <% if product.in_stock? %>
      <% variant = product.product_variants.in_stock.first %>
      <%= button_to add_item_cart_path, 
          params: { variant_id: variant.id, quantity: 1 },
          method: :post,
          class: "mt-3 w-full bg-indigo-600 text-white text-sm rounded-md py-2 hover:bg-indigo-700 transition-colors",
          data: { turbo_method: :post } do %>
        + เพิ่มลงตะกร้า
      <% end %>
    <% else %>
      <button disabled class="mt-3 w-full bg-gray-200 text-gray-500 text-sm rounded-md py-2 cursor-not-allowed">
        หมดสต็อก
      </button>
    <% end %>
  </div>
</div>
```

### Product Show View

```erb
<%# app/views/products/show.html.erb %>
<% content_for :title, @product.name %>

<div class="grid grid-cols-1 lg:grid-cols-2 gap-12">
  
  <!-- Images -->
  <div>
    <div class="aspect-square bg-gray-100 rounded-xl overflow-hidden mb-4" id="main-image">
      <% if @product.images.attached? %>
        <%= image_tag @product.primary_image.variant(resize_to_fill: [600, 600]),
            class: "w-full h-full object-cover", id: "main-product-image" %>
      <% end %>
    </div>
    
    <!-- Thumbnails -->
    <div class="flex space-x-2 overflow-x-auto">
      <% @product.images.each_with_index do |img, i| %>
        <div class="w-20 h-20 flex-shrink-0 rounded-lg overflow-hidden cursor-pointer border-2 <%= i == 0 ? 'border-indigo-500' : 'border-transparent' %>"
             data-controller="image-switcher"
             data-action="click->image-switcher#switch"
             data-image-url="<%= rails_blob_url(img) %>">
          <%= image_tag img.variant(resize_to_fill: [80, 80]), class: "w-full h-full object-cover" %>
        </div>
      <% end %>
    </div>
  </div>
  
  <!-- Product Info -->
  <div>
    <div class="mb-4">
      <p class="text-sm text-indigo-600 mb-1"><%= @product.category&.name %></p>
      <h1 class="text-2xl font-bold text-gray-900"><%= @product.name %></h1>
    </div>
    
    <!-- Rating -->
    <div class="flex items-center mb-4">
      <% rating = @product.average_rating %>
      <% 5.times do |i| %>
        <span class="text-lg <%= i < rating.floor ? 'text-yellow-400' : 'text-gray-300' %>">★</span>
      <% end %>
      <span class="text-sm text-gray-500 ml-2"><%= rating %> (<%= @reviews.count %> รีวิว)</span>
    </div>
    
    <!-- Price -->
    <div class="mb-6">
      <div class="flex items-baseline space-x-3">
        <span class="text-3xl font-bold text-gray-900" id="current-price">
          <%= Money.new(@product.base_price_cents, "THB").format %>
        </span>
        <% if @product.on_sale? %>
          <span class="text-lg text-gray-400 line-through">
            <%= Money.new(@product.compare_at_price_cents, "THB").format %>
          </span>
          <span class="text-sm text-red-500 font-medium">
            ลด <%= @product.discount_percentage %>%
          </span>
        <% end %>
      </div>
    </div>
    
    <!-- Variants -->
    <% if @variants.any? %>
      <%= form_with url: add_item_cart_path, method: :post,
          data: { controller: "variant-selector", turbo: false } do |f| %>
        
        <!-- Group by option type -->
        <% option_names = @variants.flat_map { |v| v.variant_options.map(&:name) }.uniq %>
        <% option_names.each do |option_name| %>
          <div class="mb-4">
            <label class="text-sm font-medium text-gray-700 mb-2 block"><%= option_name %></label>
            <div class="flex flex-wrap gap-2">
              <% @variants.each do |variant| %>
                <% opt = variant.variant_options.find { |o| o.name == option_name } %>
                <% next unless opt %>
                <button type="button"
                        class="px-4 py-2 text-sm border rounded-md <%= variant.in_stock? ? 'hover:border-indigo-500 hover:text-indigo-600' : 'opacity-50 cursor-not-allowed line-through' %>"
                        data-variant-id="<%= variant.id %>"
                        data-price="<%= variant.price_cents %>"
                        data-stock="<%= variant.stock_quantity %>"
                        data-action="click->variant-selector#select">
                  <%= opt.value %>
                </button>
              <% end %>
            </div>
          </div>
        <% end %>
        
        <!-- Quantity -->
        <div class="mb-6">
          <label class="text-sm font-medium text-gray-700 mb-2 block">จำนวน</label>
          <div class="flex items-center border rounded-md w-32">
            <button type="button" class="px-3 py-2 text-gray-600 hover:text-gray-900" 
                    data-action="click->variant-selector#decrementQty">-</button>
            <input type="number" name="quantity" value="1" min="1" max="10"
                   class="w-12 text-center border-0 focus:ring-0 text-sm"
                   data-variant-selector-target="quantity">
            <button type="button" class="px-3 py-2 text-gray-600 hover:text-gray-900"
                    data-action="click->variant-selector#incrementQty">+</button>
          </div>
        </div>
        
        <%= f.hidden_field :variant_id, value: @variants.first.id, 
            data: { "variant-selector-target": "variantId" } %>
        
        <!-- Add to Cart Button -->
        <div class="flex gap-4">
          <%= f.submit "เพิ่มลงตะกร้า 🛒",
              class: "flex-1 bg-indigo-600 text-white py-3 rounded-lg font-medium hover:bg-indigo-700 transition-colors" %>
          <button type="button" class="border border-gray-300 px-4 py-3 rounded-lg hover:bg-gray-50">❤️</button>
        </div>
      <% end %>
    <% end %>
    
    <!-- Features -->
    <div class="mt-6 grid grid-cols-3 gap-4 border-t pt-6">
      <div class="text-center">
        <div class="text-2xl mb-1">🚚</div>
        <p class="text-xs text-gray-600">จัดส่งฟรี<br>เมื่อซื้อครบ 500฿</p>
      </div>
      <div class="text-center">
        <div class="text-2xl mb-1">🔄</div>
        <p class="text-xs text-gray-600">คืนสินค้า<br>ภายใน 30 วัน</p>
      </div>
      <div class="text-center">
        <div class="text-2xl mb-1">🔒</div>
        <p class="text-xs text-gray-600">ชำระเงิน<br>ปลอดภัย</p>
      </div>
    </div>
  </div>
</div>

<!-- Description -->
<div class="mt-12">
  <div class="border-b mb-6">
    <nav class="flex space-x-8">
      <button class="border-b-2 border-indigo-500 text-indigo-600 px-1 pb-4 text-sm font-medium">รายละเอียด</button>
      <button class="text-gray-500 hover:text-gray-700 px-1 pb-4 text-sm font-medium">รีวิว (<%= @reviews.count %>)</button>
    </nav>
  </div>
  
  <div class="prose max-w-none">
    <%= simple_format @product.description %>
  </div>
</div>

<!-- Reviews -->
<section class="mt-12">
  <h2 class="text-xl font-bold text-gray-900 mb-6">รีวิวจากลูกค้า</h2>
  
  <% if user_signed_in? && !@reviews.exists?(user: current_user) %>
    <%= render "products/review_form", product: @product %>
  <% end %>
  
  <div class="space-y-6 mt-8">
    <% @reviews.each do |review| %>
      <%= render "products/review", review: review %>
    <% end %>
  </div>
</section>

<!-- Related Products -->
<section class="mt-16">
  <h2 class="text-xl font-bold text-gray-900 mb-6">สินค้าที่เกี่ยวข้อง</h2>
  <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
    <% @related.each do |product| %>
      <%= render "products/card", product: product %>
    <% end %>
  </div>
</section>
```

---

## ขั้นตอนที่ 7: Checkout Views

```erb
<%# app/views/checkouts/index.html.erb %>
<% content_for :title, "ชำระเงิน" %>

<div class="max-w-5xl mx-auto">
  <h1 class="text-2xl font-bold text-gray-900 mb-8">ชำระเงิน</h1>
  
  <!-- Progress Steps -->
  <div class="flex items-center mb-8">
    <% steps = [["ตะกร้า", :cart], ["ที่อยู่จัดส่ง", :address], ["ชำระเงิน", :payment], ["เสร็จสิ้น", :done]] %>
    <% steps.each_with_index do |(label, step), i| %>
      <div class="flex items-center">
        <div class="flex items-center justify-center w-8 h-8 rounded-full text-sm font-medium
                    <%= step == :address ? 'bg-indigo-600 text-white' : 'bg-gray-200 text-gray-500' %>">
          <%= i + 1 %>
        </div>
        <span class="ml-2 text-sm font-medium <%= step == :address ? 'text-indigo-600' : 'text-gray-500' %>">
          <%= label %>
        </span>
        <% if i < steps.length - 1 %>
          <div class="mx-4 flex-1 h-px bg-gray-200"></div>
        <% end %>
      </div>
    <% end %>
  </div>
  
  <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
    
    <!-- Address Form -->
    <div class="lg:col-span-2">
      <%= form_with url: checkout_path, method: :post do |f| %>
        
        <div class="bg-white rounded-lg shadow-sm p-6">
          <h2 class="text-lg font-semibold mb-4">ที่อยู่จัดส่ง</h2>
          
          <% if current_user.addresses.any? %>
            <div class="mb-4">
              <h3 class="text-sm font-medium text-gray-700 mb-2">ที่อยู่ที่บันทึกไว้</h3>
              <% current_user.addresses.each do |addr| %>
                <label class="flex items-start space-x-3 p-3 border rounded-lg mb-2 cursor-pointer hover:border-indigo-500">
                  <%= f.radio_button "address[address_id]", addr.id, checked: addr.is_default %>
                  <div class="text-sm">
                    <p class="font-medium"><%= addr.full_name %></p>
                    <p class="text-gray-500"><%= addr.full_address %></p>
                    <p class="text-gray-500">Tel: <%= addr.phone %></p>
                  </div>
                </label>
              <% end %>
              
              <details class="mt-4">
                <summary class="text-sm text-indigo-600 cursor-pointer">+ ใช้ที่อยู่ใหม่</summary>
                <%= render "checkouts/address_fields", f: f %>
              </details>
            </div>
          <% else %>
            <%= render "checkouts/address_fields", f: f %>
          <% end %>
        </div>
        
        <!-- Coupon -->
        <div class="bg-white rounded-lg shadow-sm p-6 mt-4">
          <h2 class="text-lg font-semibold mb-4">คูปองส่วนลด</h2>
          <div class="flex gap-2">
            <%= text_field_tag "coupon_code", @coupon_code,
                placeholder: "รหัสคูปอง",
                class: "flex-1 border rounded-md px-3 py-2 text-sm focus:ring-indigo-500" %>
            <%= button_to "ใช้คูปอง", apply_coupon_checkout_path,
                params: { coupon_code: "WILL_BE_OVERRIDDEN" },
                method: :post,
                class: "bg-gray-800 text-white px-4 py-2 rounded-md text-sm hover:bg-gray-900",
                data: { turbo: false } %>
          </div>
          <% if @coupon %>
            <p class="mt-2 text-green-600 text-sm">✓ คูปอง <%= @coupon.code %> ใช้งานได้</p>
          <% end %>
        </div>
        
        <%= f.submit "ดำเนินการชำระเงิน →",
            class: "mt-6 w-full bg-indigo-600 text-white py-3 rounded-lg font-medium hover:bg-indigo-700" %>
      <% end %>
    </div>
    
    <!-- Order Summary -->
    <div>
      <div class="bg-white rounded-lg shadow-sm p-6 sticky top-24">
        <h2 class="text-lg font-semibold mb-4">สรุปออร์เดอร์</h2>
        
        <div class="space-y-3 mb-4">
          <% @cart.cart_items.includes(product_variant: :product).each do |item| %>
            <div class="flex items-center justify-between text-sm">
              <div class="flex items-center space-x-2">
                <span class="text-gray-600 text-xs bg-gray-100 rounded-full w-5 h-5 flex items-center justify-center">
                  <%= item.quantity %>
                </span>
                <span class="text-gray-700"><%= item.product_variant.display_name %></span>
              </div>
              <span class="font-medium">
                <%= Money.new(item.total_price_cents, "THB").format %>
              </span>
            </div>
          <% end %>
        </div>
        
        <hr class="my-4">
        
        <div class="space-y-2 text-sm">
          <div class="flex justify-between">
            <span class="text-gray-600">ยอดรวม</span>
            <span><%= @subtotal.format %></span>
          </div>
          <div class="flex justify-between">
            <span class="text-gray-600">ค่าจัดส่ง</span>
            <span><%= @shipping.zero? ? "ฟรี!" : @shipping.format %></span>
          </div>
          <div class="flex justify-between">
            <span class="text-gray-600">ภาษีมูลค่าเพิ่ม (7%)</span>
            <span><%= @tax.format %></span>
          </div>
          <% if @discount.positive? %>
            <div class="flex justify-between text-green-600">
              <span>ส่วนลดคูปอง</span>
              <span>-<%= @discount.format %></span>
            </div>
          <% end %>
          
          <hr>
          
          <div class="flex justify-between font-bold text-lg">
            <span>ยอดสุทธิ</span>
            <span><%= @total.format %></span>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

---

## ขั้นตอนที่ 8: Payment View (Stripe)

```erb
<%# app/views/payments/show.html.erb %>
<% content_for :title, "ชำระเงิน" %>

<div class="max-w-2xl mx-auto">
  <h1 class="text-2xl font-bold text-gray-900 mb-8">ชำระเงิน</h1>
  
  <div class="bg-white rounded-lg shadow-sm p-6">
    <!-- Order Info -->
    <div class="mb-6 p-4 bg-gray-50 rounded-lg">
      <p class="text-sm text-gray-600">ออร์เดอร์: <strong><%= @order.order_number %></strong></p>
      <p class="text-2xl font-bold text-gray-900 mt-1"><%= @order.total.format %></p>
    </div>
    
    <!-- Stripe Elements -->
    <form id="payment-form" data-controller="stripe-payment"
          data-stripe-payment-public-key-value="<%= @stripe_public_key %>"
          data-stripe-payment-client-secret-value="<%= @payment_intent.client_secret %>"
          data-stripe-payment-confirm-url-value="<%= confirm_payment_order_path(@order) %>">
      
      <div class="mb-4">
        <label class="block text-sm font-medium text-gray-700 mb-2">ข้อมูลบัตร</label>
        <div id="card-element" class="border rounded-md p-3">
          <!-- Stripe Card Element inserts here -->
        </div>
        <div id="card-errors" class="text-red-500 text-sm mt-2"></div>
      </div>
      
      <div class="mb-4">
        <label class="flex items-center space-x-2 text-sm text-gray-600">
          <input type="checkbox" class="rounded" checked>
          <span>บันทึกบัตรนี้สำหรับการซื้อครั้งต่อไป</span>
        </label>
      </div>
      
      <button id="submit-btn" type="submit"
              class="w-full bg-indigo-600 text-white py-3 rounded-lg font-medium hover:bg-indigo-700 disabled:opacity-50 disabled:cursor-not-allowed">
        <span id="submit-text">ชำระเงิน <%= @order.total.format %></span>
        <span id="submit-loading" class="hidden">⏳ กำลังดำเนินการ...</span>
      </button>
      
      <!-- Security Note -->
      <p class="text-xs text-gray-400 text-center mt-3">
        🔒 ชำระเงินด้วยระบบที่ปลอดภัยจาก Stripe. ข้อมูลบัตรของคุณถูกเข้ารหัสอย่างปลอดภัย.
      </p>
    </form>
  </div>
</div>
```

```javascript
// app/javascript/controllers/stripe_payment_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = {
    publicKey: String,
    clientSecret: String,
    confirmUrl: String
  }

  connect() {
    this.stripe = Stripe(this.publicKeyValue)
    this.elements = this.stripe.elements()
    this.cardElement = this.elements.create('card', {
      style: {
        base: {
          fontSize: '16px',
          color: '#374151',
          '::placeholder': { color: '#9CA3AF' }
        }
      }
    })
    this.cardElement.mount('#card-element')

    this.cardElement.on('change', (event) => {
      const displayError = document.getElementById('card-errors')
      displayError.textContent = event.error ? event.error.message : ''
    })
  }

  async submitPayment(event) {
    event.preventDefault()
    this.setLoading(true)

    const { error, paymentIntent } = await this.stripe.confirmCardPayment(
      this.clientSecretValue,
      { payment_method: { card: this.cardElement } }
    )

    if (error) {
      document.getElementById('card-errors').textContent = error.message
      this.setLoading(false)
    } else if (paymentIntent.status === 'succeeded') {
      // Notify backend
      const response = await fetch(this.confirmUrlValue, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-CSRF-Token': document.querySelector('[name="csrf-token"]').content
        },
        body: JSON.stringify({ payment_intent_id: paymentIntent.id })
      })

      if (response.redirected) {
        window.location.href = response.url
      }
    }
  }

  setLoading(isLoading) {
    const btn = document.getElementById('submit-btn')
    const text = document.getElementById('submit-text')
    const loading = document.getElementById('submit-loading')

    btn.disabled = isLoading
    text.classList.toggle('hidden', isLoading)
    loading.classList.toggle('hidden', !isLoading)
  }
}
```

---

## ขั้นตอนที่ 9: Mailers

```ruby
# app/mailers/order_mailer.rb
class OrderMailer < ApplicationMailer
  def order_confirmed(order)
    @order = order
    @user = order.user
    @order_items = order.order_items.includes(product_variant: :product)

    mail(
      to: @user.email,
      subject: "ยืนยันการสั่งซื้อ ##{@order.order_number}"
    )
  end

  def order_shipped(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "ออร์เดอร์ ##{@order.order_number} จัดส่งแล้ว!"
    )
  end

  def order_delivered(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "ออร์เดอร์ ##{@order.order_number} ส่งถึงแล้ว 🎉"
    )
  end

  def order_cancelled(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "ออร์เดอร์ ##{@order.order_number} ถูกยกเลิก"
    )
  end

  def payment_failed(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "การชำระเงินสำหรับออร์เดอร์ ##{@order.order_number} ไม่สำเร็จ"
    )
  end

  def order_refunded(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "คืนเงินสำหรับออร์เดอร์ ##{@order.order_number}"
    )
  end
end
```

```erb
<%# app/views/order_mailer/order_confirmed.html.erb %>
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>ยืนยันการสั่งซื้อ</title>
  <style>
    body { font-family: 'Sarabun', Arial, sans-serif; color: #333; margin: 0; padding: 0; background: #f9f9f9; }
    .container { max-width: 600px; margin: 20px auto; background: white; border-radius: 8px; overflow: hidden; }
    .header { background: #4F46E5; color: white; padding: 24px; text-align: center; }
    .body { padding: 24px; }
    .order-item { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #f0f0f0; }
    .total-row { display: flex; justify-content: space-between; padding: 8px 0; font-weight: bold; }
    .button { display: inline-block; background: #4F46E5; color: white; padding: 12px 24px; border-radius: 6px; text-decoration: none; }
    .footer { text-align: center; padding: 20px; color: #999; font-size: 12px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>✅ ยืนยันการสั่งซื้อ</h1>
      <p>ออร์เดอร์ #<%= @order.order_number %></p>
    </div>
    
    <div class="body">
      <p>สวัสดีคุณ <%= @user.first_name %>,</p>
      <p>ขอบคุณที่สั่งซื้อสินค้ากับเรา! เราได้รับออร์เดอร์ของคุณแล้ว</p>
      
      <h2>รายการสินค้า</h2>
      <% @order_items.each do |item| %>
        <div class="order-item">
          <span><%= item.product_name %> <%= "- #{item.variant_description}" if item.variant_description.present? %> × <%= item.quantity %></span>
          <span><%= Money.new(item.total_price_cents, "THB").format %></span>
        </div>
      <% end %>
      
      <div style="margin-top: 16px;">
        <div class="total-row">
          <span>ยอดรวม:</span>
          <span><%= @order.subtotal.format %></span>
        </div>
        <div class="total-row">
          <span>ค่าจัดส่ง:</span>
          <span><%= @order.shipping.zero? ? "ฟรี" : @order.shipping.format %></span>
        </div>
        <div class="total-row">
          <span>ภาษีมูลค่าเพิ่ม:</span>
          <span><%= @order.tax.format %></span>
        </div>
        <div class="total-row" style="font-size: 18px; border-top: 2px solid #4F46E5; padding-top: 12px; margin-top: 8px;">
          <span>ยอดสุทธิ:</span>
          <span><%= @order.total.format %></span>
        </div>
      </div>
      
      <h2>ที่อยู่จัดส่ง</h2>
      <% addr = @order.order_address %>
      <p>
        <%= addr.first_name %> <%= addr.last_name %><br>
        <%= addr.address_line1 %><br>
        <% if addr.address_line2.present? %><%= addr.address_line2 %><br><% end %>
        <%= addr.city %>, <%= addr.state %> <%= addr.postal_code %><br>
        โทร: <%= addr.phone %>
      </p>
      
      <div style="text-align: center; margin: 24px 0;">
        <%= link_to "ดูออร์เดอร์", order_url(@order), class: "button" %>
      </div>
      
      <p>หากมีคำถามใดๆ กรุณาติดต่อเราที่ <a href="mailto:support@shopapp.com">support@shopapp.com</a></p>
    </div>
    
    <div class="footer">
      ShopRails | ร้านค้าออนไลน์<br>
      © 2024 ShopRails. All rights reserved.
    </div>
  </div>
</body>
</html>
```

---

## ขั้นตอนที่ 10: Admin Dashboard

```ruby
# app/controllers/admin/dashboard_controller.rb
module Admin
  class DashboardController < ApplicationController
    before_action :authenticate_user!
    before_action :require_admin!

    def index
      @today_orders = Order.where(created_at: Time.current.beginning_of_day..Time.current.end_of_day)
      @today_revenue = @today_orders.paid.sum(:total_cents)

      @this_month_orders = Order.this_month
      @this_month_revenue = @this_month_orders.paid.sum(:total_cents)

      @total_customers = User.count
      @new_customers_this_month = User.where(created_at: Time.current.beginning_of_month..).count

      @recent_orders = Order.includes(:user, :order_items).recent.limit(10)
      @low_stock_variants = ProductVariant.low_stock.includes(:product).limit(10)
      @top_products = Product.joins(:order_items)
                             .group("products.id")
                             .order("COUNT(order_items.id) DESC")
                             .limit(5)
                             .select("products.*, COUNT(order_items.id) as orders_count")

      @revenue_by_day = Order.paid
                             .where(created_at: 30.days.ago..)
                             .group_by_day(:created_at)
                             .sum(:total_cents)
    end

    private

    def require_admin!
      redirect_to root_path, alert: "ไม่มีสิทธิ์เข้าถึง" unless current_user.admin?
    end
  end
end
```

```erb
<%# app/views/admin/dashboard/index.html.erb %>
<% content_for :title, "Admin Dashboard" %>

<h1 class="text-2xl font-bold text-gray-900 mb-8">แดชบอร์ด</h1>

<!-- Stats Cards -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
  
  <div class="bg-white rounded-lg shadow-sm p-6">
    <div class="flex items-center justify-between">
      <div>
        <p class="text-sm font-medium text-gray-500">ออร์เดอร์วันนี้</p>
        <p class="text-3xl font-bold text-gray-900 mt-1"><%= @today_orders.count %></p>
      </div>
      <div class="text-4xl">📦</div>
    </div>
    <p class="text-sm text-gray-500 mt-2">รายได้: <%= Money.new(@today_revenue, "THB").format %></p>
  </div>
  
  <div class="bg-white rounded-lg shadow-sm p-6">
    <div class="flex items-center justify-between">
      <div>
        <p class="text-sm font-medium text-gray-500">ออร์เดอร์เดือนนี้</p>
        <p class="text-3xl font-bold text-gray-900 mt-1"><%= @this_month_orders.count %></p>
      </div>
      <div class="text-4xl">📊</div>
    </div>
    <p class="text-sm text-gray-500 mt-2">รายได้: <%= Money.new(@this_month_revenue, "THB").format %></p>
  </div>
  
  <div class="bg-white rounded-lg shadow-sm p-6">
    <div class="flex items-center justify-between">
      <div>
        <p class="text-sm font-medium text-gray-500">ลูกค้าทั้งหมด</p>
        <p class="text-3xl font-bold text-gray-900 mt-1"><%= @total_customers %></p>
      </div>
      <div class="text-4xl">👥</div>
    </div>
    <p class="text-sm text-gray-500 mt-2">ใหม่เดือนนี้: <%= @new_customers_this_month %></p>
  </div>
  
  <div class="bg-white rounded-lg shadow-sm p-6">
    <div class="flex items-center justify-between">
      <div>
        <p class="text-sm font-medium text-gray-500">สินค้าใกล้หมด</p>
        <p class="text-3xl font-bold text-red-600 mt-1"><%= @low_stock_variants.count %></p>
      </div>
      <div class="text-4xl">⚠️</div>
    </div>
    <p class="text-sm text-gray-500 mt-2">ต้องเติมสต็อก</p>
  </div>
</div>

<!-- Revenue Chart -->
<div class="bg-white rounded-lg shadow-sm p-6 mb-8">
  <h2 class="text-lg font-semibold text-gray-900 mb-4">รายได้ 30 วันล่าสุด</h2>
  <%= line_chart @revenue_by_day.transform_values { |v| v / 100.0 },
      xtitle: "วันที่",
      ytitle: "บาท",
      thousands: ",",
      prefix: "฿",
      height: "300px" %>
</div>

<div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
  
  <!-- Recent Orders -->
  <div class="bg-white rounded-lg shadow-sm p-6">
    <div class="flex justify-between items-center mb-4">
      <h2 class="text-lg font-semibold text-gray-900">ออร์เดอร์ล่าสุด</h2>
      <%= link_to "ดูทั้งหมด", admin_orders_path, class: "text-sm text-indigo-600 hover:text-indigo-700" %>
    </div>
    
    <div class="space-y-3">
      <% @recent_orders.each do |order| %>
        <div class="flex items-center justify-between py-2 border-b">
          <div>
            <p class="text-sm font-medium"><%= order.order_number %></p>
            <p class="text-xs text-gray-500"><%= order.user.name %></p>
          </div>
          <div class="text-right">
            <p class="text-sm font-medium"><%= order.total.format %></p>
            <span class="text-xs px-2 py-0.5 rounded-full
                         <%= order.confirmed? || order.processing? ? 'bg-blue-100 text-blue-700' :
                             order.shipped? ? 'bg-yellow-100 text-yellow-700' :
                             order.delivered? ? 'bg-green-100 text-green-700' :
                             order.cancelled? ? 'bg-red-100 text-red-700' :
                             'bg-gray-100 text-gray-700' %>">
              <%= order.status_label %>
            </span>
          </div>
        </div>
      <% end %>
    </div>
  </div>
  
  <!-- Low Stock -->
  <div class="bg-white rounded-lg shadow-sm p-6">
    <div class="flex justify-between items-center mb-4">
      <h2 class="text-lg font-semibold text-gray-900">สินค้าใกล้หมดสต็อก</h2>
      <%= link_to "จัดการสต็อก", admin_products_path, class: "text-sm text-indigo-600 hover:text-indigo-700" %>
    </div>
    
    <div class="space-y-3">
      <% @low_stock_variants.each do |variant| %>
        <div class="flex items-center justify-between py-2 border-b">
          <div>
            <p class="text-sm font-medium"><%= variant.product.name %></p>
            <p class="text-xs text-gray-500">SKU: <%= variant.sku %></p>
          </div>
          <div class="text-right">
            <span class="text-red-600 font-bold"><%= variant.stock_quantity %></span>
            <span class="text-xs text-gray-500"> ชิ้น</span>
          </div>
        </div>
      <% end %>
    </div>
  </div>
</div>
```

---

## ขั้นตอนที่ 11: Inventory Management

```ruby
# app/jobs/inventory_alert_job.rb
class InventoryAlertJob < ApplicationJob
  queue_as :default

  def perform
    low_stock_variants = ProductVariant.low_stock.includes(:product)
    out_of_stock_variants = ProductVariant.out_of_stock.includes(:product)

    if low_stock_variants.any? || out_of_stock_variants.any?
      AdminMailer.inventory_alert(
        low_stock: low_stock_variants.to_a,
        out_of_stock: out_of_stock_variants.to_a
      ).deliver_now
    end
  end
end
```

```ruby
# config/sidekiq.yml
:schedule:
  inventory_check:
    cron: "0 9 * * *"    # ทุกวันตอน 9 โมง
    class: InventoryAlertJob
    description: "ตรวจสอบสต็อกสินค้า"
    
  abandoned_cart_reminder:
    cron: "0 */6 * * *"  # ทุก 6 ชั่วโมง
    class: AbandonedCartReminderJob
    description: "เตือนตะกร้าที่ถูกทิ้งไว้"
```

---

## ขั้นตอนที่ 12: Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, controllers: {
    sessions: 'users/sessions',
    registrations: 'users/registrations'
  }

  root "home#index"

  # Products
  resources :products, only: [:index, :show] do
    resources :reviews, only: [:create, :destroy]
  end
  resources :categories, only: [:index, :show]

  # Cart
  resource :cart, only: [:show] do
    post :add_item
    patch :update_item
    delete :remove_item
  end

  # Checkout
  resource :checkout, only: [:index, :create] do
    post :apply_coupon
  end

  # Orders & Payments
  resources :orders, only: [:index, :show] do
    member do
      post :cancel
      get :invoice
      get :payment, to: "payments#show"
      post :confirm_payment, to: "payments#confirm"
    end
  end

  # User Profile
  resources :addresses, except: [:show]
  resource :profile, only: [:show, :edit, :update]

  # Stripe Webhooks
  post "/stripe/webhooks", to: "stripe_webhooks#create"

  # Admin
  namespace :admin do
    root to: "dashboard#index"
    resources :products do
      resources :variants, controller: "product_variants"
    end
    resources :orders do
      member do
        patch :update_status
        post :refund
      end
    end
    resources :categories
    resources :coupons
    resources :users
    resources :inventory, only: [:index] do
      collection do
        post :bulk_update
      end
    end
    get :reports, to: "reports#index"
    get "reports/sales", to: "reports#sales"
    get "reports/products", to: "reports#products"
  end

  # Health
  get "/health", to: "health#show"
end
```

---

## ขั้นตอนที่ 13: PDF Invoice

```ruby
# app/pdfs/invoice_pdf.rb
class InvoicePdf
  include ActionView::Helpers::NumberHelper

  def initialize(order)
    @order = order
    @document = Prawn::Document.new(
      page_size: "A4",
      margin: [36, 36, 36, 36]
    )
  end

  def render
    build_header
    build_order_info
    build_items_table
    build_totals
    build_footer
    @document.render
  end

  private

  def build_header
    @document.bounding_box([0, @document.cursor], width: @document.bounds.width) do
      @document.text "ใบเสร็จรับเงิน / Invoice", size: 22, style: :bold, align: :center
      @document.text "ShopRails Online Store", size: 14, align: :center, color: "4F46E5"
    end
    @document.move_down 20
    @document.stroke_horizontal_rule
    @document.move_down 10
  end

  def build_order_info
    data = [
      ["เลขที่ออร์เดอร์:", @order.order_number],
      ["วันที่สั่งซื้อ:", @order.created_at.strftime("%d/%m/%Y %H:%M")],
      ["สถานะชำระเงิน:", @order.payment_status_label],
      ["ชื่อลูกค้า:", @order.user.name],
      ["อีเมล:", @order.user.email]
    ]

    if (addr = @order.order_address)
      data << ["ที่อยู่จัดส่ง:", "#{addr.first_name} #{addr.last_name}\n#{addr.address_line1}\n#{addr.city} #{addr.postal_code}"]
    end

    @document.table(data, width: @document.bounds.width) do
      column(0).style(font_style: :bold, width: 120)
      cells.borders = []
      rows(0..-1).padding = [4, 8]
    end

    @document.move_down 20
  end

  def build_items_table
    @document.text "รายการสินค้า", size: 14, style: :bold
    @document.move_down 10

    items_data = [["สินค้า", "SKU", "จำนวน", "ราคาต่อชิ้น", "รวม"]]

    @order.order_items.each do |item|
      items_data << [
        "#{item.product_name}\n#{item.variant_description}".strip,
        item.product_variant&.sku || "-",
        item.quantity.to_s,
        Money.new(item.unit_price_cents, "THB").format,
        Money.new(item.total_price_cents, "THB").format
      ]
    end

    @document.table(items_data, width: @document.bounds.width, header: true) do
      row(0).style(font_style: :bold, background_color: "F3F4F6")
      column(2).align = :center
      column(3).align = :right
      column(4).align = :right
      cells.padding = [6, 8]
      cells.borders = [:bottom]
      cells.border_color = "E5E7EB"
    end

    @document.move_down 10
  end

  def build_totals
    totals = [
      ["ยอดรวม:", @order.subtotal.format],
      ["ค่าจัดส่ง:", @order.shipping.zero? ? "ฟรี" : @order.shipping.format],
      ["ภาษีมูลค่าเพิ่ม (7%):", @order.tax.format]
    ]

    if @order.discount_cents > 0
      totals << ["ส่วนลด:", "-#{@order.discount.format}"]
    end

    totals << ["ยอดสุทธิ:", @order.total.format]

    @document.table(totals, position: :right, width: 250) do
      column(0).align = :left
      column(1).align = :right
      column(0).style(font_style: :normal)
      row(-1).style(font_style: :bold, size: 14)
      cells.borders = []
      rows(0..-1).padding = [4, 8]
    end
  end

  def build_footer
    @document.move_down 30
    @document.stroke_horizontal_rule
    @document.move_down 10
    @document.text "ขอบคุณที่ใช้บริการ ShopRails", align: :center, size: 10, color: "9CA3AF"
    @document.text "support@shopapp.com | www.shopapp.com", align: :center, size: 10, color: "9CA3AF"
  end
end
```

---

## ขั้นตอนที่ 14: Tests

```ruby
# spec/models/order_spec.rb
require 'rails_helper'

RSpec.describe Order, type: :model do
  let(:user) { create(:user) }
  let(:order) { create(:order, user: user) }

  describe "validations" do
    it { should belong_to(:user) }
    it { should have_many(:order_items).dependent(:destroy) }
  end

  describe "enums" do
    it { should define_enum_for(:status).with_values(pending: 0, confirmed: 1, processing: 2, shipped: 3, delivered: 4, cancelled: 5, refunded: 6) }
    it { should define_enum_for(:payment_status).with_values(unpaid: 0, paid: 1, partially_refunded: 2, fully_refunded: 3) }
  end

  describe "#generate_order_number" do
    it "สร้างหมายเลขออร์เดอร์อัตโนมัติ" do
      expect(order.order_number).to match(/\AORD-\d{6}-[A-F0-9]{8}\z/)
    end

    it "สร้างหมายเลขที่ไม่ซ้ำกัน" do
      another_order = create(:order, user: user)
      expect(order.order_number).not_to eq(another_order.order_number)
    end
  end

  describe "#cancel!" do
    context "เมื่อออร์เดอร์อยู่ในสถานะ pending" do
      let(:order) { create(:order, user: user, status: :pending) }
      let!(:variant) { create(:product_variant, stock_quantity: 5) }
      let!(:item) { create(:order_item, order: order, product_variant: variant, quantity: 2) }

      it "ยกเลิกออร์เดอร์ได้" do
        order.cancel!
        expect(order.reload.status).to eq("cancelled")
      end

      it "คืนสต็อกสินค้า" do
        expect { order.cancel! }.to change { variant.reload.stock_quantity }.by(2)
      end
    end

    context "เมื่อออร์เดอร์อยู่ในสถานะ shipped" do
      let(:order) { create(:order, user: user, status: :shipped) }

      it "ยกเลิกออร์เดอร์ไม่ได้" do
        expect(order.cancel!).to be_falsy
      end
    end
  end
end
```

```ruby
# spec/services/orders/create_service_spec.rb
require 'rails_helper'

RSpec.describe Orders::CreateService do
  let(:user) { create(:user) }
  let(:cart) { create(:cart, user: user) }
  let(:variant) { create(:product_variant, stock_quantity: 10, price_cents: 50000) }
  let!(:cart_item) { create(:cart_item, cart: cart, product_variant: variant, quantity: 2) }

  let(:address_params) do
    {
      first_name: "สมชาย",
      last_name: "ใจดี",
      address_line1: "123 ถ.สุขุมวิท",
      city: "กรุงเทพ",
      state: "กรุงเทพ",
      postal_code: "10110",
      phone: "0812345678"
    }
  end

  subject(:result) do
    described_class.call(user: user, cart: cart, address_params: address_params)
  end

  context "เมื่อข้อมูลถูกต้อง" do
    it "สร้างออร์เดอร์สำเร็จ" do
      expect(result.success?).to be true
      expect(result.order).to be_a(Order)
      expect(result.order).to be_persisted
    end

    it "สร้าง order items ถูกต้อง" do
      order = result.order
      expect(order.order_items.count).to eq(1)
      expect(order.order_items.first.quantity).to eq(2)
    end

    it "คำนวณยอดรวมถูกต้อง" do
      order = result.order
      expect(order.subtotal_cents).to eq(100000)  # 500 THB × 2
    end
  end

  context "เมื่อสต็อกไม่เพียงพอ" do
    before { variant.update!(stock_quantity: 1) }

    it "สร้างออร์เดอร์ไม่สำเร็จ" do
      expect(result.success?).to be false
      expect(result.errors).to include(match(/สต็อกไม่เพียงพอ/))
    end
  end

  context "เมื่อตะกร้าว่างเปล่า" do
    before { cart.cart_items.destroy_all }

    it "สร้างออร์เดอร์ไม่สำเร็จ" do
      expect(result.success?).to be false
      expect(result.errors).to include("ตะกร้าสินค้าว่างเปล่า")
    end
  end
end
```

---

## ขั้นตอนที่ 15: Deployment Configuration

```yaml
# render.yaml
services:
  - type: web
    name: ecommerce-app
    env: ruby
    buildCommand: bundle install && bundle exec rails assets:precompile && bundle exec rails db:migrate
    startCommand: bundle exec puma -C config/puma.rb
    envVars:
      - key: RAILS_ENV
        value: production
      - key: RAILS_MASTER_KEY
        sync: false
      - key: DATABASE_URL
        fromDatabase:
          name: ecommerce-db
          property: connectionString
      - key: REDIS_URL
        fromService:
          name: ecommerce-redis
          type: redis
          property: connectionString
      - key: STRIPE_SECRET_KEY
        sync: false
      - key: STRIPE_PUBLIC_KEY
        sync: false
      - key: STRIPE_WEBHOOK_SECRET
        sync: false
      - key: AWS_ACCESS_KEY_ID
        sync: false
      - key: AWS_SECRET_ACCESS_KEY
        sync: false
      - key: AWS_REGION
        value: ap-southeast-1
      - key: S3_BUCKET
        value: ecommerce-uploads

  - type: worker
    name: ecommerce-worker
    env: ruby
    buildCommand: bundle install
    startCommand: bundle exec sidekiq -C config/sidekiq.yml
    envVars:
      - key: RAILS_ENV
        value: production
      - key: RAILS_MASTER_KEY
        sync: false

databases:
  - name: ecommerce-db
    databaseName: ecommerce_production
    plan: starter

services:
  - type: redis
    name: ecommerce-redis
    plan: starter
```

### Environment Variables

```bash
# .env (ไม่ commit ไปใน git)
# Database
DATABASE_URL=postgresql://localhost/ecommerce_development

# Redis
REDIS_URL=redis://localhost:6379/0

# Stripe
STRIPE_PUBLIC_KEY=pk_test_...
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# AWS S3
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=ap-southeast-1
S3_BUCKET=ecommerce-uploads

# Email
MAILER_FROM=noreply@shopapp.com
SENDGRID_API_KEY=SG...

# App
RAILS_ENV=development
RAILS_MASTER_KEY=...
SECRET_KEY_BASE=...
```

---

## ขั้นตอนที่ 16: Factories (FactoryBot)

```ruby
# spec/factories/products.rb
FactoryBot.define do
  factory :product do
    sequence(:name) { |n| "สินค้าตัวอย่าง #{n}" }
    description { "รายละเอียดสินค้าตัวอย่าง" }
    base_price_cents { 10000 }  # 100 THB
    status { :active }

    trait :with_variants do
      after(:create) do |product|
        create(:product_variant, product: product)
      end
    end

    trait :on_sale do
      compare_at_price_cents { 15000 }  # 150 THB
    end
  end

  factory :product_variant do
    product
    sequence(:sku) { |n| "SKU-TEST-#{n}" }
    price_cents { 10000 }
    stock_quantity { 10 }

    trait :out_of_stock do
      stock_quantity { 0 }
    end

    trait :low_stock do
      stock_quantity { 3 }
    end
  end
end

# spec/factories/orders.rb
FactoryBot.define do
  factory :order do
    user
    status { :pending }
    payment_status { :unpaid }
    currency { "THB" }
    subtotal_cents { 100000 }
    tax_cents { 7000 }
    shipping_cents { 7000 }
    total_cents { 114000 }

    trait :paid do
      status { :confirmed }
      payment_status { :paid }
    end

    trait :with_items do
      after(:create) do |order|
        create(:order_item, order: order)
      end
    end
  end

  factory :order_item do
    order
    product_variant
    quantity { 1 }
    unit_price_cents { 10000 }
    total_price_cents { 10000 }
    product_name { "สินค้าทดสอบ" }
  end
end
```

---

## ขั้นตอนที่ 17: Seeds

```ruby
# db/seeds.rb
puts "Seeding..."

# Admin User
admin = User.find_or_create_by(email: "admin@shopapp.com") do |u|
  u.name = "Admin User"
  u.password = "Admin1234!"
  u.password_confirmation = "Admin1234!"
  u.role = :admin
  u.confirmed_at = Time.current
end
puts "Admin: #{admin.email}"

# Categories
electronics = Category.find_or_create_by(name: "อิเล็กทรอนิกส์") { |c| c.description = "สินค้าอิเล็กทรอนิกส์" }
fashion = Category.find_or_create_by(name: "แฟชั่น") { |c| c.description = "เสื้อผ้าและเครื่องแต่งกาย" }
home = Category.find_or_create_by(name: "บ้านและสวน") { |c| c.description = "สินค้าสำหรับบ้านและสวน" }
books = Category.find_or_create_by(name: "หนังสือ") { |c| c.description = "หนังสือและสื่อการเรียนรู้" }

# Products
puts "Creating products..."

[
  {
    name: "iPhone 15 Pro",
    category: electronics,
    price: 4499900,
    description: "iPhone 15 Pro สมาร์ทโฟนรุ่นล่าสุดจาก Apple",
    variants: [
      { options: [["สี", "ไทเทเนียมดำ"], ["ความจุ", "128GB"]], price: 4499900, stock: 5 },
      { options: [["สี", "ไทเทเนียมดำ"], ["ความจุ", "256GB"]], price: 4999900, stock: 3 },
      { options: [["สี", "ไทเทเนียมขาว"], ["ความจุ", "128GB"]], price: 4499900, stock: 7 },
    ]
  },
  {
    name: "MacBook Air M3",
    category: electronics,
    price: 4199000,
    description: "MacBook Air รุ่นใหม่พร้อม Apple M3 chip",
    variants: [
      { options: [["สี", "สเปซเกรย์"], ["RAM", "8GB"]], price: 4199000, stock: 4 },
      { options: [["สี", "สเปซเกรย์"], ["RAM", "16GB"]], price: 4799000, stock: 2 },
    ]
  },
  {
    name: "เสื้อยืดคอกลม",
    category: fashion,
    price: 29900,
    compare_at_price: 49900,
    description: "เสื้อยืดคอกลมผ้าคอตตอน 100% นุ่มสบาย",
    variants: [
      { options: [["สี", "ขาว"], ["ไซส์", "S"]], price: 29900, stock: 20 },
      { options: [["สี", "ขาว"], ["ไซส์", "M"]], price: 29900, stock: 15 },
      { options: [["สี", "ขาว"], ["ไซส์", "L"]], price: 29900, stock: 10 },
      { options: [["สี", "ดำ"], ["ไซส์", "S"]], price: 29900, stock: 12 },
      { options: [["สี", "ดำ"], ["ไซส์", "M"]], price: 29900, stock: 18 },
      { options: [["สี", "ดำ"], ["ไซส์", "L"]], price: 29900, stock: 8 },
    ]
  },
  {
    name: "Ruby on Rails 7 Bible",
    category: books,
    price: 89900,
    description: "คู่มือ Ruby on Rails ฉบับสมบูรณ์ภาษาไทย",
    variants: [
      { options: [], price: 89900, stock: 50 }
    ]
  }
].each do |product_data|
  variants_data = product_data.delete(:variants)
  compare_at_price = product_data.delete(:compare_at_price)
  price = product_data.delete(:price)

  product = Product.find_or_create_by(name: product_data[:name]) do |p|
    p.category = product_data[:category]
    p.base_price_cents = price
    p.compare_at_price_cents = compare_at_price
    p.description = product_data[:description]
    p.status = :active
    p.featured = true
  end

  variants_data.each do |vd|
    sku = "#{product.name.first(3).upcase}-#{SecureRandom.hex(3).upcase}"
    variant = product.product_variants.create!(
      sku: sku,
      price_cents: vd[:price],
      stock_quantity: vd[:stock]
    )

    vd[:options].each do |name, value|
      variant.variant_options.create!(name: name, value: value)
    end
  end

  puts "  Created: #{product.name}"
end

# Coupons
Coupon.find_or_create_by(code: "WELCOME10") do |c|
  c.discount_type = :percentage
  c.discount_value = 10
  c.usage_limit = 1000
  c.expires_at = 1.year.from_now
  c.active = true
end

Coupon.find_or_create_by(code: "SAVE100") do |c|
  c.discount_type = :fixed
  c.discount_value = 100
  c.minimum_order_cents = 50000
  c.usage_limit = 500
  c.expires_at = 6.months.from_now
  c.active = true
end

puts "\nSeeds completed!"
puts "Admin login: admin@shopapp.com / Admin1234!"
```

---

## สรุปโปรเจค

E-Commerce Store ที่เราสร้างมีฟีเจอร์ครบถ้วน:

1. **สินค้าและ Variants** - สินค้าพร้อมตัวเลือก (ขนาด, สี, ฯลฯ) 
2. **ตะกร้าสินค้า** - Turbo Streams สำหรับ real-time updates
3. **Checkout** - กรอกที่อยู่, ใช้คูปอง
4. **Stripe Payment** - ชำระเงินด้วยบัตรเครดิต, Webhooks
5. **Order Management** - ติดตามออร์เดอร์, ยกเลิก, PDF Invoice
6. **Inventory** - ติดตามสต็อก, แจ้งเตือนสต็อกต่ำ
7. **Email Notifications** - ยืนยันออร์เดอร์, จัดส่ง, ยกเลิก
8. **Admin Dashboard** - สถิติ, จัดการสินค้า, ออร์เดอร์
9. **Search & Filter** - PgSearch, filter ราคา, หมวดหมู่
10. **Reviews** - รีวิวสินค้าพร้อม Rating

เทคโนโลยีที่ใช้:
- Ruby 3.2 + Rails 7.1
- PostgreSQL + Redis + Sidekiq
- Stripe สำหรับ Payment
- AWS S3 สำหรับ File Storage
- Tailwind CSS + Hotwire/Turbo
- RSpec + FactoryBot สำหรับ Testing
- Render.com สำหรับ Deployment
