# Part 57: Microservices และ Service Objects ใน Ruby on Rails

## ขั้นตอนที่ 1246-1270: สถาปัตยกรรม Microservices

---

## ขั้นตอนที่ 1246: Monolith vs Microservices

### Monolithic Architecture

Monolith คือแอปพลิเคชันที่ทุกส่วนอยู่รวมกันในโค้ดเดียว

```
rails-app/
├── app/
│   ├── models/
│   │   ├── user.rb
│   │   ├── product.rb
│   │   ├── order.rb
│   │   └── payment.rb
│   ├── controllers/
│   ├── services/
│   └── jobs/
└── config/
```

**ข้อดีของ Monolith:**
- ง่ายต่อการพัฒนาและ Deploy
- ไม่มีปัญหา Network latency ระหว่าง services
- Transaction แบบ ACID ทำได้ง่าย
- Debug ง่ายกว่า

**ข้อเสียของ Monolith:**
- Scale ยาก (ต้อง scale ทั้งระบบ)
- เมื่อโค้ดใหญ่ขึ้น ทำให้ development ช้าลง
- Technology lock-in
- Deployment เสี่ยงสูง

### Microservices Architecture

```
                    ┌─────────────────────┐
                    │      API Gateway     │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
    ┌─────▼──────┐      ┌──────▼─────┐      ┌──────▼──────┐
    │   User     │      │  Product   │      │   Order     │
    │  Service   │      │  Service   │      │   Service   │
    └────────────┘      └────────────┘      └─────────────┘
          │                    │                    │
    ┌─────▼──────┐      ┌──────▼─────┐      ┌──────▼──────┐
    │  Users DB  │      │ Products DB│      │  Orders DB  │
    └────────────┘      └────────────┘      └─────────────┘
```

**ข้อดีของ Microservices:**
- Scale แต่ละ service แยกกัน
- Deploy อิสระต่อกัน
- ใช้ Technology ต่างกันได้
- Fault isolation

**ข้อเสียของ Microservices:**
- ซับซ้อนกว่า Monolith
- Network latency
- Distributed transactions ยาก
- ต้องการ DevOps expertise มากกว่า

### เมื่อไรควรใช้ Microservices?

```
✅ ควรใช้เมื่อ:
- ทีมใหญ่ (10+ developers)
- ระบบซับซ้อนมาก
- ต้องการ scale แต่ละส่วนแยกกัน
- มี Domain ที่ชัดเจน

❌ ไม่ควรใช้เมื่อ:
- ทีมเล็ก
- ระบบยังเล็กหรือยังพัฒนาอยู่
- ไม่มี expertise ด้าน DevOps
- ยังไม่รู้ boundary ของ domain ชัดเจน
```

---

## ขั้นตอนที่ 1247: Service Object Pattern

Service Object คือ class ที่ encapsulate business logic

### ปัญหาที่ Service Objects แก้ไข

```ruby
# BAD: Logic อยู่ใน Controller
class OrdersController < ApplicationController
  def create
    @order = Order.new(order_params)
    
    # Business logic ใน controller
    @order.calculate_total
    @order.apply_discount(current_user)
    
    if @order.save
      PaymentGateway.charge(@order.total, current_user.payment_method)
      OrderMailer.confirmation(@order).deliver_later
      InventoryService.reduce_stock(@order.items)
      redirect_to @order
    else
      render :new
    end
  end
end
```

```ruby
# GOOD: Logic อยู่ใน Service Object
class OrdersController < ApplicationController
  def create
    result = Orders::CreateService.call(
      user: current_user,
      params: order_params
    )
    
    if result.success?
      redirect_to result.order
    else
      @errors = result.errors
      render :new
    end
  end
end
```

### โครงสร้าง Service Object

```ruby
# app/services/application_service.rb
class ApplicationService
  def self.call(*args, **kwargs, &block)
    new(*args, **kwargs).call
  end
  
  def call
    raise NotImplementedError, "#{self.class}#call ยังไม่ได้ implement"
  end
end
```

### Service Result Object

```ruby
# app/services/service_result.rb
class ServiceResult
  attr_reader :data, :errors, :error_code
  
  def initialize(success:, data: nil, errors: [], error_code: nil)
    @success = success
    @data = data
    @errors = Array(errors)
    @error_code = error_code
  end
  
  def success?
    @success
  end
  
  def failure?
    !success?
  end
  
  def self.success(data = nil)
    new(success: true, data: data)
  end
  
  def self.failure(errors, error_code: nil)
    new(success: false, errors: errors, error_code: error_code)
  end
end
```

### ตัวอย่าง Service Objects จริง

```ruby
# app/services/orders/create_service.rb
module Orders
  class CreateService < ApplicationService
    def initialize(user:, cart:, payment_params:)
      @user = user
      @cart = cart
      @payment_params = payment_params
    end
    
    def call
      ActiveRecord::Base.transaction do
        order = create_order
        process_payment(order)
        clear_cart
        send_confirmation(order)
        
        ServiceResult.success(order)
      end
    rescue PaymentError => e
      ServiceResult.failure(e.message, error_code: :payment_failed)
    rescue ActiveRecord::RecordInvalid => e
      ServiceResult.failure(e.record.errors.full_messages)
    end
    
    private
    
    def create_order
      order = Order.create!(
        user: @user,
        items: @cart.items.map { |item| 
          OrderItem.new(
            product: item.product,
            quantity: item.quantity,
            unit_price: item.product.price
          )
        },
        subtotal: @cart.subtotal,
        discount: calculate_discount,
        total: calculate_total
      )
      
      # ลด stock
      @cart.items.each do |item|
        item.product.decrement!(:stock, item.quantity)
      end
      
      order
    end
    
    def process_payment(order)
      charge = Stripe::Charge.create(
        amount: (order.total * 100).to_i,
        currency: 'thb',
        source: @payment_params[:token],
        description: "Order ##{order.id}"
      )
      
      order.update!(
        payment_id: charge.id,
        payment_status: 'paid'
      )
    end
    
    def clear_cart
      @cart.items.destroy_all
    end
    
    def send_confirmation(order)
      OrderMailer.confirmation(order).deliver_later
    end
    
    def calculate_discount
      # ตัวอย่าง: 10% สำหรับ member มากกว่า 1 ปี
      if @user.member_since < 1.year.ago
        @cart.subtotal * 0.10
      else
        0
      end
    end
    
    def calculate_total
      @cart.subtotal - calculate_discount
    end
  end
end
```

```ruby
# app/services/users/registration_service.rb
module Users
  class RegistrationService < ApplicationService
    def initialize(params:)
      @params = params
    end
    
    def call
      user = User.new(@params)
      
      if user.save
        send_welcome_email(user)
        create_default_settings(user)
        track_registration(user)
        
        ServiceResult.success(user)
      else
        ServiceResult.failure(user.errors.full_messages)
      end
    end
    
    private
    
    def send_welcome_email(user)
      UserMailer.welcome(user).deliver_later
    end
    
    def create_default_settings(user)
      UserSettings.create!(
        user: user,
        email_notifications: true,
        public_profile: false,
        language: 'th'
      )
    end
    
    def track_registration(user)
      Analytics.track('user_registered', {
        user_id: user.id,
        email: user.email,
        registered_at: Time.current
      })
    end
  end
end
```

---

## ขั้นตอนที่ 1248: Interactor Pattern

Interactor gem ทำให้การสร้าง Service Objects ง่ายขึ้น

### ติดตั้ง

```ruby
# Gemfile
gem 'interactor'
gem 'interactor-rails'
```

```bash
bundle install
```

### Interactor พื้นฐาน

```ruby
# app/interactors/authenticate_user.rb
class AuthenticateUser
  include Interactor
  
  def call
    user = User.find_by(email: context.email)
    
    if user&.authenticate(context.password)
      context.user = user
      context.token = generate_token(user)
    else
      context.fail!(message: "อีเมลหรือรหัสผ่านไม่ถูกต้อง")
    end
  end
  
  private
  
  def generate_token(user)
    JWT.encode(
      { user_id: user.id, exp: 7.days.from_now.to_i },
      Rails.application.credentials.secret_key_base
    )
  end
end
```

```ruby
# ใช้งาน
result = AuthenticateUser.call(email: "user@example.com", password: "password123")

if result.success?
  puts "Token: #{result.token}"
  puts "User: #{result.user.name}"
else
  puts "Error: #{result.message}"
end
```

### Organizer - รวมหลาย Interactors

```ruby
# app/interactors/place_order.rb
class PlaceOrder
  include Interactor::Organizer
  
  organize ValidateCart, 
           CreateOrder, 
           ChargePayment, 
           SendConfirmation, 
           UpdateInventory
end
```

```ruby
# app/interactors/validate_cart.rb
class ValidateCart
  include Interactor
  
  def call
    cart = context.cart
    
    if cart.items.empty?
      context.fail!(message: "ตะกร้าสินค้าว่างเปล่า")
    end
    
    cart.items.each do |item|
      if item.product.stock < item.quantity
        context.fail!(
          message: "สินค้า '#{item.product.name}' มีไม่เพียงพอ",
          field: :quantity
        )
      end
    end
  end
end
```

```ruby
# app/interactors/create_order.rb
class CreateOrder
  include Interactor
  
  def call
    order = Order.create!(
      user: context.user,
      cart: context.cart,
      total: context.cart.total
    )
    
    context.order = order
  rescue ActiveRecord::RecordInvalid => e
    context.fail!(message: e.message)
  end
  
  def rollback
    context.order&.destroy
  end
end
```

```ruby
# app/interactors/charge_payment.rb
class ChargePayment
  include Interactor
  
  def call
    charge = Stripe::Charge.create(
      amount: context.order.total_cents,
      currency: 'thb',
      source: context.payment_token
    )
    
    context.charge = charge
    context.order.update!(payment_id: charge.id, paid: true)
  rescue Stripe::CardError => e
    context.fail!(message: "การชำระเงินล้มเหลว: #{e.message}")
  end
  
  def rollback
    # Refund ถ้าได้รับเงินไปแล้วแต่มีข้อผิดพลาดหลังจากนั้น
    if context.charge
      Stripe::Refund.create(charge: context.charge.id)
    end
  end
end
```

```ruby
# ใช้งานใน Controller
class OrdersController < ApplicationController
  def create
    result = PlaceOrder.call(
      user: current_user,
      cart: current_cart,
      payment_token: params[:payment_token]
    )
    
    if result.success?
      redirect_to order_path(result.order), 
                  notice: "สั่งซื้อสำเร็จ!"
    else
      flash.now[:alert] = result.message
      render :checkout
    end
  end
end
```

---

## ขั้นตอนที่ 1249: Form Objects

Form Object ใช้ encapsulate form validation

### ปัญหา: Virtual attributes ใน Model

```ruby
# BAD: Model รับผิดชอบมากเกินไป
class User < ApplicationRecord
  attr_accessor :current_password, :password_confirmation
  
  validate :current_password_matches, if: :changing_password?
  
  # Model รู้ว่ากำลัง change password หรือไม่ (ไม่ควร)
  def changing_password?
    current_password.present?
  end
end
```

### Form Object Solution

```ruby
# app/forms/registration_form.rb
class RegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes
  
  attribute :email, :string
  attribute :name, :string
  attribute :username, :string
  attribute :password, :string
  attribute :password_confirmation, :string
  attribute :terms_accepted, :boolean, default: false
  
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name, presence: true, length: { minimum: 2, maximum: 100 }
  validates :username, presence: true, 
            format: { with: /\A[a-z0-9_]+\z/, message: "ใช้ได้แค่ a-z, 0-9, _" },
            uniqueness_check: true
  validates :password, length: { minimum: 8 }
  validates :password_confirmation, presence: true
  validates :terms_accepted, acceptance: true
  
  validate :password_matches_confirmation
  validate :email_not_taken
  
  def save
    return false unless valid?
    
    ActiveRecord::Base.transaction do
      user = User.create!(
        email: email,
        name: name,
        username: username,
        password: password
      )
      
      @user = user
      true
    end
  rescue ActiveRecord::RecordInvalid => e
    errors.add(:base, e.message)
    false
  end
  
  def user
    @user
  end
  
  private
  
  def password_matches_confirmation
    if password != password_confirmation
      errors.add(:password_confirmation, "ไม่ตรงกับรหัสผ่าน")
    end
  end
  
  def email_not_taken
    if User.exists?(email: email)
      errors.add(:email, "ถูกใช้ไปแล้ว")
    end
  end
end
```

### ใช้ Form Object ใน Controller

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  def new
    @form = RegistrationForm.new
  end
  
  def create
    @form = RegistrationForm.new(registration_params)
    
    if @form.save
      sign_in(@form.user)
      redirect_to dashboard_path, notice: "สมัครสมาชิกสำเร็จ!"
    else
      render :new
    end
  end
  
  private
  
  def registration_params
    params.require(:registration).permit(
      :email, :name, :username, 
      :password, :password_confirmation, 
      :terms_accepted
    )
  end
end
```

### Form Object สำหรับ Multi-step Form

```ruby
# app/forms/checkout_form.rb
class CheckoutForm
  include ActiveModel::Model
  
  STEPS = [:address, :payment, :review]
  
  attr_accessor :step, :user, :cart
  
  # Step 1: Address
  attr_accessor :shipping_name, :shipping_address, :shipping_city,
                :shipping_province, :shipping_postal_code
                
  # Step 2: Payment
  attr_accessor :payment_method, :card_token
  
  validates :shipping_name, :shipping_address, presence: true, if: :address_step?
  validates :payment_method, presence: true, if: :payment_step?
  
  def initialize(attrs = {})
    super
    @step ||= STEPS.first
  end
  
  def current_step
    @step.to_sym
  end
  
  def last_step?
    current_step == STEPS.last
  end
  
  def next_step
    next_index = STEPS.index(current_step) + 1
    STEPS[next_index]
  end
  
  def address_step?
    current_step == :address
  end
  
  def payment_step?
    current_step == :payment
  end
  
  def save
    return false unless valid?
    return place_order if last_step?
    true
  end
  
  private
  
  def place_order
    PlaceOrder.call(
      user: user,
      cart: cart,
      shipping: shipping_params,
      payment_token: card_token
    ).success?
  end
  
  def shipping_params
    {
      name: shipping_name,
      address: shipping_address,
      city: shipping_city,
      province: shipping_province,
      postal_code: shipping_postal_code
    }
  end
end
```

---

## ขั้นตอนที่ 1250: Query Objects

Query Object ช่วย encapsulate database queries

### ปัญหา: Scope ที่ซับซ้อนใน Model

```ruby
# BAD: Model มี scopes มากเกินไป
class Post < ApplicationRecord
  scope :published, -> { where(status: 'published') }
  scope :by_author, ->(author_id) { where(user_id: author_id) }
  scope :with_tags, ->(tag_ids) { joins(:tags).where(tags: { id: tag_ids }) }
  scope :recent, -> { order(created_at: :desc) }
  scope :popular, -> { order(likes_count: :desc) }
  scope :with_comments_count, -> { 
    left_joins(:comments)
      .select('posts.*, COUNT(comments.id) as comments_count')
      .group('posts.id') 
  }
  # ... อีกหลาย scopes
end
```

### Query Object Solution

```ruby
# app/queries/posts_query.rb
class PostsQuery
  attr_reader :scope
  
  def initialize(scope = Post.all)
    @scope = scope
  end
  
  def call(params = {})
    result = scope
    result = filter_by_status(result, params[:status])
    result = filter_by_author(result, params[:author_id])
    result = filter_by_tags(result, params[:tag_ids])
    result = search_by_keyword(result, params[:search])
    result = filter_by_date_range(result, params[:from_date], params[:to_date])
    result = sort(result, params[:sort_by], params[:direction])
    result
  end
  
  private
  
  def filter_by_status(scope, status)
    return scope if status.blank?
    scope.where(status: status)
  end
  
  def filter_by_author(scope, author_id)
    return scope if author_id.blank?
    scope.where(user_id: author_id)
  end
  
  def filter_by_tags(scope, tag_ids)
    return scope if tag_ids.blank?
    scope.joins(:tags)
         .where(tags: { id: tag_ids })
         .distinct
  end
  
  def search_by_keyword(scope, keyword)
    return scope if keyword.blank?
    scope.where(
      "title ILIKE :q OR body ILIKE :q",
      q: "%#{keyword}%"
    )
  end
  
  def filter_by_date_range(scope, from_date, to_date)
    scope = scope.where("created_at >= ?", from_date) if from_date.present?
    scope = scope.where("created_at <= ?", to_date) if to_date.present?
    scope
  end
  
  def sort(scope, sort_by, direction = "desc")
    direction = %w[asc desc].include?(direction.to_s.downcase) ? direction : "desc"
    
    case sort_by.to_s
    when "title"
      scope.order("title #{direction}")
    when "likes_count"
      scope.order("likes_count #{direction}")
    when "comments_count"
      scope.left_joins(:comments)
           .select("posts.*, COUNT(comments.id) as comments_count")
           .group("posts.id")
           .order("comments_count #{direction}")
    else
      scope.order("created_at #{direction}")
    end
  end
end
```

```ruby
# ใช้งาน
class PostsController < ApplicationController
  def index
    @posts = PostsQuery.new.call(
      status: params[:status],
      author_id: params[:author_id],
      tag_ids: params[:tag_ids],
      search: params[:q],
      from_date: params[:from_date],
      to_date: params[:to_date],
      sort_by: params[:sort],
      direction: params[:direction]
    ).page(params[:page]).per(20)
  end
end
```

### Query Object แบบ Chainable

```ruby
# app/queries/base_query.rb
class BaseQuery
  def initialize(scope)
    @scope = scope
  end
  
  def self.call(scope = default_scope, **options)
    new(scope).call(**options)
  end
  
  def self.default_scope
    raise NotImplementedError
  end
end

# app/queries/products_query.rb
class ProductsQuery < BaseQuery
  def self.default_scope
    Product.all
  end
  
  def call(search: nil, category_id: nil, min_price: nil, max_price: nil, 
           in_stock: nil, sort: "created_at", direction: "desc")
    
    @scope
      .then { |s| search.present? ? s.search(search) : s }
      .then { |s| category_id ? s.where(category_id: category_id) : s }
      .then { |s| min_price ? s.where("price >= ?", min_price) : s }
      .then { |s| max_price ? s.where("price <= ?", max_price) : s }
      .then { |s| in_stock ? s.where("stock > 0") : s }
      .order("#{sort} #{direction}")
  end
end
```

---

## ขั้นตอนที่ 1251: Presenter Objects

Presenter ช่วย format ข้อมูลสำหรับ View

### View Presenter

```ruby
# app/presenters/post_presenter.rb
class PostPresenter
  delegate_missing_to :post
  
  def initialize(post, view_context)
    @post = post
    @h = view_context
  end
  
  def status_badge
    @h.content_tag(:span, status_text, class: "badge badge-#{status_color}")
  end
  
  def formatted_date
    @h.time_tag(created_at, created_at.strftime("%d %B %Y เวลา %H:%M"), 
               datetime: created_at.iso8601)
  end
  
  def truncated_body(length: 200)
    @h.truncate(body, length: length, separator: ' ')
  end
  
  def reading_time
    words = body.split.length
    minutes = (words / 200.0).ceil
    "#{minutes} นาที"
  end
  
  def cover_image_or_placeholder
    if cover_image.attached?
      @h.image_tag(@h.rails_blob_url(cover_image), class: "post-cover")
    else
      @h.image_tag("post-placeholder.jpg", class: "post-cover placeholder")
    end
  end
  
  def author_link
    @h.link_to(user.name, @h.user_path(user), class: "author-link")
  end
  
  private
  
  def post
    @post
  end
  
  def status_text
    case status
    when "published" then "เผยแพร่แล้ว"
    when "draft" then "แบบร่าง"
    when "archived" then "เก็บถาวร"
    end
  end
  
  def status_color
    case status
    when "published" then "success"
    when "draft" then "warning"
    when "archived" then "secondary"
    end
  end
end
```

### Presenter ใน View

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def present(model)
    presenter_class = "#{model.class.name}Presenter".constantize
    presenter = presenter_class.new(model, self)
    yield(presenter) if block_given?
    presenter
  end
end
```

```erb
<%# app/views/posts/show.html.erb %>
<% present @post do |post| %>
  <article>
    <%= post.cover_image_or_placeholder %>
    
    <h1><%= post.title %></h1>
    
    <div class="meta">
      <%= post.status_badge %>
      <span>โดย <%= post.author_link %></span>
      <%= post.formatted_date %>
      <span>เวลาอ่าน: <%= post.reading_time %></span>
    </div>
    
    <div class="content">
      <%= post.body %>
    </div>
  </article>
<% end %>
```

### Decorator Pattern (draper gem)

```ruby
# Gemfile
gem 'draper'
```

```ruby
# app/decorators/user_decorator.rb
class UserDecorator < Draper::Decorator
  delegate_all
  
  def full_name
    "#{first_name} #{last_name}"
  end
  
  def avatar_or_initials
    if object.avatar.attached?
      h.image_tag(object.avatar, class: "avatar")
    else
      h.content_tag(:div, initials, class: "avatar-initials")
    end
  end
  
  def member_since
    h.time_ago_in_words(object.created_at) + " ที่ผ่านมา"
  end
  
  def formatted_posts_count
    count = object.posts.count
    "#{count} #{"บทความ".pluralize(count)}"
  end
  
  private
  
  def initials
    name.split.map(&:first).join.upcase.first(2)
  end
end

# ใช้งาน
user = User.find(1).decorate
user.full_name  # => "สมชาย ใจดี"
user.member_since  # => "2 ปี ที่ผ่านมา"
```

---

## ขั้นตอนที่ 1252: Service Communication ด้วย HTTP

### HTTP Client สำหรับเรียก External APIs

```ruby
# app/services/external/payment_client.rb
module External
  class PaymentClient
    BASE_URL = ENV['PAYMENT_API_URL']
    
    def initialize
      @connection = build_connection
    end
    
    def create_charge(amount:, currency:, source:, description:)
      response = @connection.post('/charges') do |req|
        req.body = {
          amount: amount,
          currency: currency,
          source: source,
          description: description
        }
      end
      
      handle_response(response)
    end
    
    def create_refund(charge_id:, amount: nil)
      response = @connection.post("/charges/#{charge_id}/refunds") do |req|
        req.body = { amount: amount }.compact
      end
      
      handle_response(response)
    end
    
    private
    
    def build_connection
      Faraday.new(BASE_URL) do |conn|
        conn.request :json
        conn.response :json
        conn.response :logger if Rails.env.development?
        conn.adapter Faraday.default_adapter
        
        conn.headers['Authorization'] = "Bearer #{ENV['PAYMENT_API_KEY']}"
        conn.headers['Content-Type'] = 'application/json'
        
        # Retry logic
        conn.request :retry, max: 3, interval: 0.5,
                     retry_statuses: [500, 502, 503]
      end
    end
    
    def handle_response(response)
      case response.status
      when 200..299
        response.body
      when 401
        raise AuthenticationError, "Invalid API key"
      when 402
        raise PaymentError, response.body['error']['message']
      when 422
        raise ValidationError, response.body['errors']
      else
        raise APIError, "HTTP #{response.status}: #{response.body}"
      end
    end
  end
end
```

### Message Bus Pattern

```ruby
# Gemfile
gem 'message_bus'
```

```ruby
# config/initializers/message_bus.rb
MessageBus.configure(
  backend: :redis,
  url: ENV['REDIS_URL']
)
```

```ruby
# app/services/event_publisher.rb
class EventPublisher
  def self.publish(channel, data)
    MessageBus.publish(channel, data.to_json)
  end
  
  def self.publish_order_created(order)
    publish("/orders/created", {
      event: "order.created",
      order_id: order.id,
      user_id: order.user_id,
      total: order.total,
      timestamp: Time.current.iso8601
    })
  end
  
  def self.publish_payment_processed(payment)
    publish("/payments/processed", {
      event: "payment.processed",
      payment_id: payment.id,
      order_id: payment.order_id,
      amount: payment.amount,
      timestamp: Time.current.iso8601
    })
  end
end
```

```ruby
# ใช้งาน
# ใน OrdersController
def create
  result = PlaceOrder.call(...)
  
  if result.success?
    EventPublisher.publish_order_created(result.order)
    redirect_to result.order
  end
end
```

---

## ขั้นตอนที่ 1253: Rails Engines

Rails Engine คือ mini Rails app ที่ embed ใน app หลัก

### สร้าง Engine

```bash
rails plugin new admin_dashboard --mountable --database=postgresql
```

### โครงสร้าง Engine

```
admin_dashboard/
├── app/
│   ├── assets/
│   ├── controllers/admin_dashboard/
│   ├── models/admin_dashboard/
│   └── views/admin_dashboard/
├── config/
│   └── routes.rb
├── lib/
│   ├── admin_dashboard/
│   │   ├── engine.rb
│   │   └── version.rb
│   └── admin_dashboard.rb
└── admin_dashboard.gemspec
```

### Engine Configuration

```ruby
# lib/admin_dashboard/engine.rb
module AdminDashboard
  class Engine < ::Rails::Engine
    isolate_namespace AdminDashboard
    
    config.generators do |g|
      g.test_framework :rspec
      g.fixture_replacement :factory_bot
    end
    
    initializer "admin_dashboard.assets" do |app|
      app.config.assets.precompile += %w[
        admin_dashboard/application.css
        admin_dashboard/application.js
      ]
    end
  end
end
```

### Mount Engine

```ruby
# config/routes.rb (ใน main app)
Rails.application.routes.draw do
  mount AdminDashboard::Engine => "/admin"
end
```

### Engine Routes

```ruby
# admin_dashboard/config/routes.rb
AdminDashboard::Engine.routes.draw do
  root to: "dashboard#index"
  
  resources :users
  resources :posts
  resources :orders do
    member do
      post :refund
      post :cancel
    end
  end
  
  namespace :reports do
    get :sales
    get :users
    get :inventory
  end
end
```

---

## ขั้นตอนที่ 1254: API Clients

### Base API Client

```ruby
# app/services/api_clients/base_client.rb
module ApiClients
  class BaseClient
    class Error < StandardError; end
    class AuthenticationError < Error; end
    class NotFoundError < Error; end
    class RateLimitError < Error; end
    class ServerError < Error; end
    
    attr_reader :connection
    
    def initialize(base_url:, api_key:)
      @connection = Faraday.new(base_url) do |conn|
        conn.request :json
        conn.response :json
        conn.adapter Faraday.default_adapter
        conn.headers['Authorization'] = "Bearer #{api_key}"
        conn.headers['User-Agent'] = "RailsApp/1.0"
      end
    end
    
    protected
    
    def get(path, params = {})
      request(:get, path, params: params)
    end
    
    def post(path, body = {})
      request(:post, path, body: body)
    end
    
    def put(path, body = {})
      request(:put, path, body: body)
    end
    
    def delete(path)
      request(:delete, path)
    end
    
    private
    
    def request(method, path, params: nil, body: nil)
      response = @connection.send(method, path) do |req|
        req.params = params if params
        req.body = body.to_json if body
      end
      
      handle_response(response)
    end
    
    def handle_response(response)
      case response.status
      when 200..299
        response.body
      when 401
        raise AuthenticationError, "Unauthorized"
      when 404
        raise NotFoundError, "Not found"
      when 429
        raise RateLimitError, "Rate limit exceeded"
      when 500..599
        raise ServerError, "Server error: #{response.status}"
      else
        raise Error, "Unexpected status: #{response.status}"
      end
    end
  end
end
```

### Specific API Clients

```ruby
# app/services/api_clients/sms_client.rb
module ApiClients
  class SmsClient < BaseClient
    def initialize
      super(
        base_url: ENV['SMS_API_URL'],
        api_key: ENV['SMS_API_KEY']
      )
    end
    
    def send_otp(phone_number:, code:)
      post('/messages', {
        to: phone_number,
        message: "รหัส OTP ของคุณคือ #{code} (หมดอายุใน 5 นาที)"
      })
    end
    
    def send_notification(phone_number:, message:)
      post('/messages', {
        to: phone_number,
        message: message
      })
    end
    
    def send_bulk(messages)
      post('/bulk', { messages: messages })
    end
  end
end
```

---

## ขั้นตอนที่ 1255: Event-Driven Architecture

```ruby
# app/events/application_event.rb
class ApplicationEvent
  attr_reader :data, :occurred_at
  
  def initialize(data = {})
    @data = data
    @occurred_at = Time.current
  end
  
  def name
    self.class.name.underscore.gsub('/', '.')
  end
end

# app/events/order_placed_event.rb
class OrderPlacedEvent < ApplicationEvent
  def initialize(order:)
    super(
      order_id: order.id,
      user_id: order.user_id,
      total: order.total,
      items_count: order.items.count
    )
  end
end
```

```ruby
# app/services/event_dispatcher.rb
class EventDispatcher
  class << self
    def listeners
      @listeners ||= Hash.new { |h, k| h[k] = [] }
    end
    
    def subscribe(event_class, listener)
      listeners[event_class] << listener
    end
    
    def dispatch(event)
      listeners[event.class].each do |listener|
        listener.call(event)
      end
    end
  end
end
```

```ruby
# config/initializers/event_subscriptions.rb
EventDispatcher.subscribe(OrderPlacedEvent, ->(event) {
  OrderMailer.confirmation_for_user(event.data[:order_id]).deliver_later
})

EventDispatcher.subscribe(OrderPlacedEvent, ->(event) {
  InventoryService.reduce_stock_for_order(event.data[:order_id])
})

EventDispatcher.subscribe(OrderPlacedEvent, ->(event) {
  Analytics.track("order_placed", event.data)
})
```

---

## ขั้นตอนที่ 1256-1258: Advanced Service Patterns

### Policy Objects

```ruby
# app/policies/post_policy.rb
class PostPolicy
  attr_reader :user, :post
  
  def initialize(user, post)
    @user = user
    @post = post
  end
  
  def show?
    post.published? || author? || admin?
  end
  
  def create?
    user.present?
  end
  
  def update?
    author? || admin? || moderator?
  end
  
  def destroy?
    author? || admin?
  end
  
  def publish?
    author? || admin?
  end
  
  def feature?
    admin?
  end
  
  private
  
  def author?
    user == post.user
  end
  
  def admin?
    user&.admin?
  end
  
  def moderator?
    user&.moderator?
  end
end
```

### Value Objects

```ruby
# app/values/money.rb
class Money
  include Comparable
  
  attr_reader :amount, :currency
  
  def initialize(amount, currency = "THB")
    @amount = BigDecimal(amount.to_s)
    @currency = currency
  end
  
  def +(other)
    ensure_same_currency(other)
    Money.new(amount + other.amount, currency)
  end
  
  def -(other)
    ensure_same_currency(other)
    Money.new(amount - other.amount, currency)
  end
  
  def *(scalar)
    Money.new(amount * scalar, currency)
  end
  
  def <=>(other)
    return nil unless other.is_a?(Money)
    ensure_same_currency(other)
    amount <=> other.amount
  end
  
  def to_s
    "#{format('%.2f', amount)} #{currency}"
  end
  
  def to_f
    amount.to_f
  end
  
  def zero?
    amount.zero?
  end
  
  private
  
  def ensure_same_currency(other)
    raise "Currency mismatch: #{currency} vs #{other.currency}" unless currency == other.currency
  end
end

# ใช้งาน
price = Money.new(100, "THB")
tax = Money.new(7, "THB")
total = price + tax  # Money.new(107, "THB")
```

### Repository Pattern

```ruby
# app/repositories/post_repository.rb
class PostRepository
  def find(id)
    Post.find(id)
  end
  
  def find_by_slug(slug)
    Post.find_by!(slug: slug)
  end
  
  def published
    Post.where(status: 'published').order(published_at: :desc)
  end
  
  def by_author(user)
    Post.where(user: user).order(created_at: :desc)
  end
  
  def save(post)
    post.save!
    post
  end
  
  def delete(post)
    post.destroy!
  end
  
  def search(query, options = {})
    scope = Post.all
    scope = scope.where("title ILIKE ?", "%#{query}%") if query.present?
    scope = scope.where(status: options[:status]) if options[:status]
    scope
  end
end
```

---

## ขั้นตอนที่ 1259-1265: Communication Patterns

### Synchronous Communication (HTTP)

```ruby
# app/services/product_service_client.rb
class ProductServiceClient
  include HTTParty
  
  base_uri ENV['PRODUCT_SERVICE_URL']
  headers 'Content-Type' => 'application/json',
          'X-Service-Token' => ENV['SERVICE_TOKEN']
  
  def get_product(id)
    response = self.class.get("/products/#{id}")
    
    if response.success?
      Product.new(response.parsed_response)
    else
      raise "Product not found: #{id}"
    end
  end
  
  def get_products(ids)
    response = self.class.post("/products/batch", body: { ids: ids }.to_json)
    
    if response.success?
      response.parsed_response.map { |p| Product.new(p) }
    else
      []
    end
  end
end
```

### Asynchronous Communication (Sidekiq + Redis)

```ruby
# app/jobs/order_notification_job.rb
class OrderNotificationJob < ApplicationJob
  queue_as :notifications
  
  def perform(order_id)
    order = Order.find(order_id)
    
    # แจ้งเตือน user
    OrderMailer.confirmation(order).deliver_now
    
    # แจ้งเตือน SMS
    SmsClient.new.send_notification(
      phone_number: order.user.phone,
      message: "ยืนยันคำสั่งซื้อ ##{order.id} ยอด #{order.total} บาท"
    )
    
    # อัปเดต inventory
    InventoryUpdateJob.perform_later(order_id)
    
    # ส่งข้อมูลไป analytics
    AnalyticsTrackJob.perform_later("order_placed", order.id)
  end
end
```

### Circuit Breaker Pattern

```ruby
# app/services/circuit_breaker.rb
class CircuitBreaker
  STATES = [:closed, :open, :half_open]
  
  def initialize(service_name, failure_threshold: 5, timeout: 30)
    @service_name = service_name
    @failure_threshold = failure_threshold
    @timeout = timeout
    @failures = 0
    @state = :closed
    @last_failure_time = nil
  end
  
  def call
    case @state
    when :open
      if Time.current - @last_failure_time > @timeout
        @state = :half_open
      else
        raise "Circuit is OPEN for #{@service_name}"
      end
    end
    
    begin
      result = yield
      on_success
      result
    rescue => e
      on_failure
      raise e
    end
  end
  
  private
  
  def on_success
    @failures = 0
    @state = :closed
  end
  
  def on_failure
    @failures += 1
    @last_failure_time = Time.current
    
    if @failures >= @failure_threshold
      @state = :open
      Rails.logger.warn "Circuit OPENED for #{@service_name}"
    end
  end
end

# ใช้งาน
breaker = CircuitBreaker.new("payment_service")

begin
  result = breaker.call { PaymentClient.new.charge(amount: 100) }
rescue => e
  flash[:error] = "ระบบชำระเงินขัดข้อง กรุณาลองใหม่"
end
```

---

## ขั้นตอนที่ 1266-1270: Best Practices

### Dependency Injection

```ruby
# app/services/email_service.rb
class EmailService
  def initialize(mailer: UserMailer, delivery: :deliver_later)
    @mailer = mailer
    @delivery = delivery
  end
  
  def send_welcome(user)
    mail = @mailer.welcome(user)
    mail.public_send(@delivery)
  end
  
  def send_confirmation(user, token)
    mail = @mailer.email_confirmation(user, token)
    mail.public_send(@delivery)
  end
end

# ใช้งาน normal
EmailService.new.send_welcome(user)

# ใช้งานใน tests
EmailService.new(delivery: :deliver_now).send_welcome(user)
```

### Service Object Testing

```ruby
# spec/services/orders/create_service_spec.rb
require 'rails_helper'

RSpec.describe Orders::CreateService do
  let(:user) { create(:user) }
  let(:cart) { create(:cart, user: user) }
  let(:product) { create(:product, price: 100, stock: 10) }
  
  before do
    create(:cart_item, cart: cart, product: product, quantity: 2)
  end
  
  describe "#call" do
    context "เมื่อทุกอย่างปกติ" do
      before do
        allow(Stripe::Charge).to receive(:create).and_return(
          double(id: "charge_123")
        )
      end
      
      it "สร้าง order ได้" do
        result = described_class.call(
          user: user,
          cart: cart,
          payment_params: { token: "tok_test" }
        )
        
        expect(result.success?).to be true
        expect(result.data).to be_a(Order)
        expect(result.data.total).to eq(200)
      end
      
      it "ลด stock" do
        described_class.call(user: user, cart: cart, payment_params: { token: "tok_test" })
        expect(product.reload.stock).to eq(8)
      end
      
      it "ส่ง email" do
        expect(OrderMailer).to receive(:confirmation).and_call_original
        described_class.call(user: user, cart: cart, payment_params: { token: "tok_test" })
      end
    end
    
    context "เมื่อ payment ล้มเหลว" do
      before do
        allow(Stripe::Charge).to receive(:create)
          .and_raise(Stripe::CardError.new("Your card was declined.", nil))
      end
      
      it "คืนค่า failure" do
        result = described_class.call(
          user: user,
          cart: cart,
          payment_params: { token: "tok_fail" }
        )
        
        expect(result.failure?).to be true
        expect(result.error_code).to eq(:payment_failed)
      end
      
      it "ไม่สร้าง order" do
        expect {
          described_class.call(user: user, cart: cart, payment_params: { token: "tok_fail" })
        }.not_to change(Order, :count)
      end
    end
  end
end
```

---

## แบบฝึกหัดที่ 1-25

### แบบฝึกหัดที่ 1
สร้าง Service Object สำหรับ User Registration

```ruby
# สร้าง:
# app/services/users/registration_service.rb
# - Validate params
# - Create user
# - Send welcome email
# - Create default settings
```

### แบบฝึกหัดที่ 2
สร้าง Interactor สำหรับ Blog Post Publication

```ruby
# Interactors:
# - ValidatePost (ตรวจสอบ content)
# - OptimizeImages (ลดขนาดรูป)
# - GenerateSlug
# - NotifyFollowers
# - PublishPost
```

### แบบฝึกหัดที่ 3
สร้าง Form Object สำหรับ User Profile Update

```ruby
# ProfileUpdateForm:
# - name, username, bio, avatar
# - Validation rules
# - Handle avatar upload
```

### แบบฝึกหัดที่ 4
สร้าง Query Object สำหรับ Product Catalog

```ruby
# ProductsQuery:
# - search by keyword
# - filter by category
# - filter by price range
# - filter by availability
# - sort options
```

### แบบฝึกหัดที่ 5
สร้าง Presenter สำหรับ Order Summary

```ruby
# OrderPresenter:
# - formatted_total (แสดงราคาพร้อม currency)
# - status_badge (HTML badge)
# - delivery_estimate (วันที่คาดว่าจะได้รับ)
# - item_count_text (เช่น "3 รายการ")
```

### แบบฝึกหัดที่ 6
Implement Circuit Breaker สำหรับ External Payment API

### แบบฝึกหัดที่ 7
สร้าง Rails Engine สำหรับ Admin Dashboard

```ruby
# AdminDashboard Engine:
# - Dashboard หลัก
# - จัดการ Users
# - จัดการ Posts
# - Reports
```

### แบบฝึกหัดที่ 8
สร้าง Event System สำหรับ E-commerce

```ruby
# Events:
# - OrderPlaced
# - PaymentProcessed
# - OrderShipped
# - OrderDelivered

# Listeners:
# - EmailNotifier
# - SMSNotifier
# - InventoryUpdater
# - AnalyticsTracker
```

### แบบฝึกหัดที่ 9
สร้าง Value Object สำหรับ Address

```ruby
# Address Value Object:
# - house_number, street, district
# - city, province, postal_code, country
# - full_address method
# - comparison methods
```

### แบบฝึกหัดที่ 10
สร้าง Policy Object สำหรับ Comment System

```ruby
# CommentPolicy:
# - can create? (logged in user)
# - can edit? (author only, within 24 hours)
# - can delete? (author or admin)
# - can pin? (post author or admin)
```

### แบบฝึกหัดที่ 11
Refactor Blog Controller ให้ใช้ Service Objects

### แบบฝึกหัดที่ 12
สร้าง HTTP Client สำหรับ Third-party API (e.g., Weather API)

### แบบฝึกหัดที่ 13
Implement Repository Pattern สำหรับ Product

### แบบฝึกหัดที่ 14
สร้าง Dependency Injection สำหรับ Notification Service

### แบบฝึกหัดที่ 15
เขียน Tests สำหรับ Service Objects ทั้งหมด

### แบบฝึกหัดที่ 16
สร้าง Organizer Interactor สำหรับ Complete Checkout Flow

### แบบฝึกหัดที่ 17
สร้าง Form Object สำหรับ Multi-step Registration

### แบบฝึกหัดที่ 18
Implement Observer Pattern สำหรับ Audit Log

```ruby
# บันทึกทุกการเปลี่ยนแปลงของ Post:
# - created, updated, published, deleted
# - ว่าใครทำ, เมื่อไร, เปลี่ยนอะไร
```

### แบบฝึกหัดที่ 19
สร้าง Service สำหรับ Image Processing

```ruby
# ImageProcessingService:
# - resize(width, height)
# - generate_thumbnail
# - add_watermark
# - optimize_for_web
```

### แบบฝึกหัดที่ 20
สร้าง Reporting Service ที่ Generate PDF Report

### แบบฝึกหัดที่ 21
Implement Rate Limiting Service

```ruby
# RateLimiter:
# - check_rate_limit(user_id, action)
# - increment(user_id, action)
# - reset(user_id, action)
```

### แบบฝึกหัดที่ 22
สร้าง Background Job ที่ใช้ Service Objects

### แบบฝึกหัดที่ 23
สร้าง API Client ที่มี Retry Logic และ Exponential Backoff

### แบบฝึกหัดที่ 24
Implement Feature Flags System

```ruby
# FeatureFlag:
# - enabled?(user, :new_dashboard)
# - enable(:new_dashboard)
# - disable(:new_dashboard)
# - enable_for_percentage(:new_feature, 10) # 10% ของ users
```

### แบบฝึกหัดที่ 25
สร้าง Full Microservice Architecture สำหรับ E-commerce:

```
Services:
1. UserService - authentication, profiles
2. ProductService - catalog, inventory
3. OrderService - orders, checkout
4. PaymentService - billing
5. NotificationService - email, SMS

Communication:
- Sync: REST APIs
- Async: Message Queue (Redis/Sidekiq)

เขียน Tests สำหรับทุก Service
Deploy ด้วย Docker Compose
```

---

## สรุป Part 57

ในส่วนนี้เราได้เรียนรู้:

1. **Monolith vs Microservices** - ข้อดีข้อเสียและเมื่อไรควรใช้
2. **Service Objects** - Encapsulate business logic ออกจาก Controller/Model
3. **Interactors** - รวมหลาย Services เป็น Pipeline
4. **Form Objects** - Handle form validation แยกจาก Model
5. **Query Objects** - Encapsulate database queries
6. **Presenter/Decorator** - Format ข้อมูลสำหรับ View
7. **HTTP Communication** - เรียก External APIs
8. **Message Bus** - Asynchronous communication
9. **Rails Engines** - Modular application structure
10. **Best Practices** - Testing, Dependency Injection, Patterns

การใช้ Service Objects และ Related Patterns ช่วยให้โค้ดอ่านง่าย ทดสอบง่าย และบำรุงรักษาง่ายขึ้น แม้ว่าจะยังเป็น Monolith อยู่
