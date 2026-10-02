# Common Rails Patterns

## รูปแบบการเขียนโค้ดที่ดีใน Ruby on Rails

---

## 1. Service Object Pattern

**จุดประสงค์:** แยก business logic ออกจาก Controllers และ Models

### โครงสร้าง

```
app/
  services/
    users/
      registration_service.rb
      password_reset_service.rb
    orders/
      checkout_service.rb
      refund_service.rb
    base_service.rb
```

### Implementation

```ruby
# app/services/base_service.rb
class BaseService
  Result = Data.define(:success, :data, :errors) do
    def success? = success
    def failure? = !success
  end

  def self.call(...)
    new(...).call
  end

  private

  def success(data = nil)
    Result.new(success: true, data: data, errors: [])
  end

  def failure(errors)
    errors = Array(errors)
    Result.new(success: false, data: nil, errors: errors)
  end
end

# app/services/users/registration_service.rb
module Users
  class RegistrationService < BaseService
    def initialize(params:, ip_address: nil)
      @params     = params
      @ip_address = ip_address
    end

    def call
      return failure("Email already taken") if email_taken?

      user = build_user

      return failure(user.errors.full_messages) unless user.valid?

      User.transaction do
        user.save!
        create_profile(user)
        send_welcome_email(user)
        track_signup(user)
      end

      success(user)
    rescue => e
      Rails.logger.error "Registration failed: #{e.message}"
      failure("Registration failed. Please try again.")
    end

    private

    def email_taken?
      User.exists?(email: @params[:email]&.downcase)
    end

    def build_user
      User.new(
        email:    @params[:email]&.downcase&.strip,
        password: @params[:password],
        name:     @params[:name]&.strip
      )
    end

    def create_profile(user)
      UserProfile.create!(user: user)
    end

    def send_welcome_email(user)
      UserMailer.welcome(user).deliver_later
    end

    def track_signup(user)
      Analytics.track(
        user_id:    user.id,
        event:      'signed_up',
        ip_address: @ip_address
      )
    end
  end
end

# Controller ใช้งาน
class UsersController < ApplicationController
  def create
    result = Users::RegistrationService.call(
      params:     user_params,
      ip_address: request.remote_ip
    )

    if result.success?
      sign_in(result.data)
      redirect_to dashboard_path, notice: "Welcome!"
    else
      @errors = result.errors
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :email, :password, :password_confirmation)
  end
end
```

### Testing Service Objects

```ruby
# spec/services/users/registration_service_spec.rb
RSpec.describe Users::RegistrationService do
  describe '.call' do
    let(:valid_params) do
      { name: "Alice", email: "alice@example.com", password: "secret123" }
    end

    context 'with valid params' do
      it 'creates a user' do
        expect {
          described_class.call(params: valid_params)
        }.to change(User, :count).by(1)
      end

      it 'returns success result' do
        result = described_class.call(params: valid_params)
        expect(result).to be_success
        expect(result.data).to be_a(User)
      end

      it 'sends welcome email' do
        expect {
          described_class.call(params: valid_params)
        }.to have_enqueued_mail(UserMailer, :welcome)
      end
    end

    context 'with duplicate email' do
      before { create(:user, email: valid_params[:email]) }

      it 'returns failure' do
        result = described_class.call(params: valid_params)
        expect(result).to be_failure
        expect(result.errors).to include("Email already taken")
      end
    end
  end
end
```

---

## 2. Repository Pattern

**จุดประสงค์:** แยก data access logic ออกจาก business logic

```ruby
# app/repositories/base_repository.rb
class BaseRepository
  def initialize(model_class)
    @model = model_class
  end

  def find(id)
    @model.find(id)
  end

  def find_by(**conditions)
    @model.find_by(**conditions)
  end

  def all
    @model.all
  end

  def create(attributes)
    @model.create(attributes)
  end

  def update(record, attributes)
    record.update(attributes)
    record
  end

  def delete(record)
    record.destroy
  end

  def transaction(&block)
    @model.transaction(&block)
  end
end

# app/repositories/user_repository.rb
class UserRepository < BaseRepository
  def initialize
    super(User)
  end

  def find_by_email(email)
    @model.find_by(email: email.downcase)
  end

  def active_users
    @model.where(active: true).order(name: :asc)
  end

  def admins
    @model.where(role: 'admin')
  end

  def search(query)
    @model.where(
      "name ILIKE :q OR email ILIKE :q",
      q: "%#{query}%"
    )
  end

  def with_recent_activity(days: 30)
    @model.where('last_active_at > ?', days.days.ago)
  end

  def paginated(page:, per_page: 20)
    @model.page(page).per(per_page)
  end
end

# app/repositories/order_repository.rb
class OrderRepository < BaseRepository
  def initialize
    super(Order)
  end

  def pending_for_user(user)
    @model.where(user: user, status: 'pending')
           .order(created_at: :desc)
  end

  def completed_in_range(start_date, end_date)
    @model.where(status: 'completed')
           .where(completed_at: start_date..end_date)
  end

  def revenue_by_month
    @model.where(status: 'completed')
           .group("DATE_TRUNC('month', completed_at)")
           .sum(:total_amount)
  end

  def with_items
    @model.includes(:order_items, :products)
  end
end

# Service ใช้ Repository
class OrderProcessingService
  def initialize(order_repo: OrderRepository.new)
    @orders = order_repo
  end

  def process_pending
    pending = @orders.all.where(status: 'pending')
    pending.each { |order| process_order(order) }
  end

  private

  def process_order(order)
    # ...
  end
end
```

---

## 3. Presenter / Decorator Pattern

**จุดประสงค์:** แยก presentation logic ออกจาก Models

### Draper-style Decorator

```ruby
# app/presenters/base_presenter.rb
class BasePresenter
  include ActionView::Helpers::NumberHelper
  include ActionView::Helpers::DateHelper
  include Rails.application.routes.url_helpers

  def initialize(object, view_context = nil)
    @object  = object
    @context = view_context
  end

  def self.decorate(collection, view_context = nil)
    collection.map { |obj| new(obj, view_context) }
  end

  private

  def h
    @context
  end

  def method_missing(method, *args, &block)
    if @object.respond_to?(method)
      @object.send(method, *args, &block)
    else
      super
    end
  end

  def respond_to_missing?(method, include_private = false)
    @object.respond_to?(method, include_private) || super
  end
end

# app/presenters/user_presenter.rb
class UserPresenter < BasePresenter
  def display_name
    @object.display_name.presence || @object.username
  end

  def avatar_url(size: 64)
    if @object.avatar.attached?
      Rails.application.routes.url_helpers.rails_representation_path(
        @object.avatar.variant(resize_to_fill: [size, size]),
        only_path: true
      )
    else
      gravatar_url(size: size)
    end
  end

  def gravatar_url(size:)
    hash = Digest::MD5.hexdigest(@object.email.downcase)
    "https://www.gravatar.com/avatar/#{hash}?s=#{size}&d=identicon"
  end

  def member_since
    time_ago_in_words(@object.created_at) + " ago"
  end

  def last_active
    return "Never" if @object.last_active_at.nil?
    @object.last_active_at > 5.minutes.ago ? "Online" : time_ago_in_words(@object.last_active_at) + " ago"
  end

  def role_badge
    case @object.role
    when 'admin'   then '<span class="badge badge-red">Admin</span>'
    when 'mod'     then '<span class="badge badge-yellow">Mod</span>'
    else                '<span class="badge badge-gray">Member</span>'
    end.html_safe
  end

  def formatted_created_at
    @object.created_at.strftime("%B %d, %Y")
  end

  def post_count_display
    count = @object.posts.count
    "#{number_with_delimiter(count)} #{count == 1 ? 'post' : 'posts'}"
  end
end

# app/presenters/article_presenter.rb
class ArticlePresenter < BasePresenter
  def reading_time
    words   = @object.body.split.size
    minutes = (words / 200.0).ceil
    "#{minutes} min read"
  end

  def excerpt(length: 200)
    ActionView::Base.full_sanitizer.sanitize(@object.body)
                    .truncate(length, separator: ' ')
  end

  def status_badge
    classes = case @object.status
              when 'published' then 'bg-green-100 text-green-800'
              when 'draft'     then 'bg-yellow-100 text-yellow-800'
              when 'archived'  then 'bg-gray-100 text-gray-800'
              end

    "<span class=\"inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium #{classes}\">
       #{@object.status.capitalize}
     </span>".html_safe
  end

  def author_link
    "<a href=\"/users/#{@object.user.id}\">#{@object.user.display_name}</a>".html_safe
  end

  def published_at_formatted
    @object.published_at&.strftime("%b %d, %Y") || "Draft"
  end
end

# Controller
class ArticlesController < ApplicationController
  def index
    articles = Article.published.recent.limit(20)
    @articles = ArticlePresenter.decorate(articles, view_context)
  end

  def show
    @article = ArticlePresenter.new(Article.find(params[:id]), view_context)
  end
end

# View
# <%= @article.reading_time %>
# <%= @article.status_badge %>
# <%= @article.author_link %>
```

---

## 4. Form Object Pattern

**จุดประสงค์:** Handle complex forms ที่ update หลาย models

```ruby
# app/forms/base_form.rb
class BaseForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  def self.model_name
    ActiveModel::Name.new(self)
  end
end

# app/forms/user_registration_form.rb
class UserRegistrationForm < BaseForm
  attribute :name,                  :string
  attribute :email,                 :string
  attribute :password,              :string
  attribute :password_confirmation, :string
  attribute :company_name,          :string
  attribute :company_size,          :string
  attribute :newsletter,            :boolean, default: false

  validates :name,     presence: true, length: { minimum: 2 }
  validates :email,    presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 },
                       confirmation: true

  validate :email_not_taken
  validate :password_complexity

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      @user    = create_user
      @company = create_company if company_name.present?
      subscribe_to_newsletter if newsletter
    end

    true
  rescue => e
    errors.add(:base, e.message)
    false
  end

  def user    = @user
  def company = @company

  private

  def email_not_taken
    errors.add(:email, "is already taken") if User.exists?(email: email)
  end

  def password_complexity
    return if password.blank?
    unless password.match?(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
      errors.add(:password, "must include uppercase, lowercase, and number")
    end
  end

  def create_user
    User.create!(name: name, email: email, password: password)
  end

  def create_company
    company = Company.create!(name: company_name, size: company_size)
    company.memberships.create!(user: @user, role: 'owner')
    company
  end

  def subscribe_to_newsletter
    NewsletterService.subscribe(@user.email)
  end
end

# Controller
class RegistrationsController < ApplicationController
  def new
    @form = UserRegistrationForm.new
  end

  def create
    @form = UserRegistrationForm.new(registration_params)

    if @form.save
      sign_in(@form.user)
      redirect_to dashboard_path, notice: "Welcome!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def registration_params
    params.require(:user_registration_form)
          .permit(:name, :email, :password, :password_confirmation,
                  :company_name, :company_size, :newsletter)
  end
end
```

---

## 5. Query Object Pattern

**จุดประสงค์:** Encapsulate complex database queries

```ruby
# app/queries/base_query.rb
class BaseQuery
  def initialize(relation = nil)
    @relation = relation || default_relation
  end

  def call
    @relation
  end

  private

  def default_relation
    raise NotImplementedError
  end
end

# app/queries/articles/published_query.rb
module Articles
  class PublishedQuery < BaseQuery
    def initialize(relation = Article.all)
      super(relation)
    end

    def call
      @relation.where(status: 'published')
               .where('published_at <= ?', Time.current)
    end

    private

    def default_relation
      Article.all
    end
  end
end

# app/queries/articles/search_query.rb
module Articles
  class SearchQuery < BaseQuery
    def initialize(query:, relation: Article.all)
      @query    = query
      super(relation)
    end

    def call
      return @relation if @query.blank?

      @relation.where(
        "title ILIKE :q OR body ILIKE :q OR tags ILIKE :q",
        q: "%#{@query}%"
      )
    end
  end
end

# app/queries/articles/trending_query.rb
module Articles
  class TrendingQuery < BaseQuery
    def initialize(period: 7.days, limit: 10, relation: Article.all)
      @period   = period
      @limit    = limit
      super(relation)
    end

    def call
      @relation
        .joins(:article_views)
        .where('article_views.created_at > ?', @period.ago)
        .group('articles.id')
        .order('COUNT(article_views.id) DESC')
        .limit(@limit)
    end
  end
end

# Composable queries
class ArticleFilter
  def initialize(params)
    @params = params
  end

  def results
    scope = Article.includes(:user, :tags)

    scope = Articles::PublishedQuery.new(scope).call
    scope = Articles::SearchQuery.new(query: @params[:q], relation: scope).call if @params[:q]
    scope = scope.where(category: @params[:category]) if @params[:category]
    scope = scope.where(user_id: @params[:author_id]) if @params[:author_id]
    scope = apply_sort(scope)
    scope = scope.page(@params[:page]).per(@params[:per_page] || 20)

    scope
  end

  private

  def apply_sort(scope)
    case @params[:sort]
    when 'popular'    then scope.order(views_count: :desc)
    when 'oldest'     then scope.order(created_at: :asc)
    else                   scope.order(created_at: :desc)
    end
  end
end

# Usage in controller
def index
  @articles = ArticleFilter.new(params.permit(:q, :category, :author_id, :sort, :page)).results
end
```

---

## 6. Value Object Pattern

**จุดประสงค์:** Represent domain concepts as immutable objects

```ruby
# app/value_objects/money.rb
class Money
  include Comparable

  attr_reader :amount, :currency

  EXCHANGE_RATES = {
    'USD' => 1.0,
    'EUR' => 0.92,
    'THB' => 35.0,
    'GBP' => 0.79
  }.freeze

  def initialize(amount, currency = 'USD')
    @amount   = BigDecimal(amount.to_s)
    @currency = currency.upcase
    freeze
  end

  def +(other)
    same_currency!(other)
    Money.new(amount + other.amount, currency)
  end

  def -(other)
    same_currency!(other)
    Money.new(amount - other.amount, currency)
  end

  def *(factor)
    Money.new(amount * factor, currency)
  end

  def /(divisor)
    raise ArgumentError, "Cannot divide by zero" if divisor.zero?
    Money.new(amount / divisor, currency)
  end

  def <=>(other)
    return nil unless other.is_a?(Money)
    to_base <=> other.to_base
  end

  def ==(other)
    other.is_a?(Money) && amount == other.amount && currency == other.currency
  end

  def to_base
    amount * BigDecimal(EXCHANGE_RATES[currency].to_s)
  end

  def convert_to(target_currency)
    base   = to_base
    rate   = EXCHANGE_RATES[target_currency.upcase]
    raise ArgumentError, "Unknown currency: #{target_currency}" unless rate

    Money.new(base / BigDecimal(rate.to_s), target_currency)
  end

  def format
    "#{formatted_amount} #{currency}"
  end

  def formatted_amount
    case currency
    when 'USD' then "$#{formatted_number}"
    when 'EUR' then "€#{formatted_number}"
    when 'GBP' then "£#{formatted_number}"
    when 'THB' then "฿#{formatted_number}"
    else            "#{currency} #{formatted_number}"
    end
  end

  def zero? = amount.zero?
  def positive? = amount.positive?
  def negative? = amount.negative?

  def to_s = format
  def inspect = "#<Money #{format}>"

  private

  def formatted_number
    "%.2f" % amount
  end

  def same_currency!(other)
    raise ArgumentError, "Currency mismatch: #{currency} vs #{other.currency}" \
      unless currency == other.currency
  end
end

# app/value_objects/email_address.rb
class EmailAddress
  attr_reader :value

  PATTERN = /\A[^@\s]+@[^@\s]+\.[^@\s]+\z/

  def initialize(value)
    @value = value.to_s.downcase.strip
    validate!
    freeze
  end

  def domain
    @value.split('@').last
  end

  def local_part
    @value.split('@').first
  end

  def ==(other)
    other.is_a?(EmailAddress) && value == other.value
  end

  def to_s = @value
  def inspect = "#<EmailAddress #{@value}>"

  private

  def validate!
    raise ArgumentError, "Invalid email: #{@value}" unless @value.match?(PATTERN)
  end
end

# app/value_objects/address.rb
class Address
  attr_reader :street, :city, :state, :zip, :country

  def initialize(street:, city:, state: nil, zip:, country: 'TH')
    @street  = street.to_s.strip
    @city    = city.to_s.strip
    @state   = state&.to_s&.strip
    @zip     = zip.to_s.strip
    @country = country.to_s.upcase
    freeze
  end

  def full_address
    parts = [@street, @city, @state, @zip, @country].compact.reject(&:empty?)
    parts.join(', ')
  end

  def ==(other)
    other.is_a?(Address) &&
      street == other.street && city == other.city &&
      zip == other.zip && country == other.country
  end

  def to_s = full_address
end

# Using Value Objects in ActiveRecord
class Order < ApplicationRecord
  composed_of :total_price,
              class_name:  'Money',
              mapping:     [['price_amount', 'amount'], ['price_currency', 'currency']],
              constructor: ->(amount, currency) { Money.new(amount, currency) }

  def formatted_total
    total_price.formatted_amount
  end
end
```

---

## 7. Policy Object Pattern (Pundit)

**จุดประสงค์:** Encapsulate authorization logic

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user   = user
    @record = record
  end

  def index?  = user.present?
  def show?   = user.present?
  def create? = user.present?
  def update? = user.present? && owner_or_admin?
  def destroy? = admin?

  def scope
    Scope.new(user, record.class).resolve
  end

  class Scope
    attr_reader :user, :scope

    def initialize(user, scope)
      @user  = user
      @scope = scope
    end

    def resolve
      scope.all
    end
  end

  private

  def admin?         = user&.admin?
  def owner?         = user == record.try(:user)
  def owner_or_admin? = owner? || admin?
end

# app/policies/article_policy.rb
class ArticlePolicy < ApplicationPolicy
  def show?
    record.published? || owner? || admin?
  end

  def create?
    user.present? && user.verified_email?
  end

  def update?
    owner? || admin?
  end

  def destroy?
    owner? || admin?
  end

  def publish?
    owner? || admin?
  end

  def feature?
    admin?
  end

  class Scope < Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user.present?
        scope.where(published: true).or(scope.where(user: user))
      else
        scope.where(published: true)
      end
    end
  end
end

# app/policies/organization_policy.rb
class OrganizationPolicy < ApplicationPolicy
  def show?    = member? || admin?
  def update?  = org_admin? || admin?
  def destroy? = admin?
  def invite?  = org_admin? || admin?

  def manage_billing?
    org_admin? && record.on_paid_plan?
  end

  private

  def membership
    @membership ||= record.memberships.find_by(user: user)
  end

  def member?    = membership.present?
  def org_admin? = membership&.role.in?(%w[admin owner])
end

# Controller with Pundit
class ArticlesController < ApplicationController
  include Pundit::Authorization

  after_action :verify_authorized

  def index
    @articles = policy_scope(Article).recent.limit(20)
    skip_authorization  # index uses scope only
  end

  def show
    @article = Article.find(params[:id])
    authorize @article
  end

  def publish
    @article = Article.find(params[:id])
    authorize @article, :publish?

    @article.update!(published: true, published_at: Time.current)
    redirect_to @article, notice: "Article published!"
  end

  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

  private

  def user_not_authorized
    flash[:alert] = "You are not authorized to perform this action."
    redirect_back(fallback_location: root_path)
  end
end
```

---

## 8. Observer Pattern

**จุดประสงค์:** React to model changes without coupling

### ActiveSupport::Notifications

```ruby
# app/observers/order_observer.rb
class OrderObserver
  def self.subscribe!
    ActiveSupport::Notifications.subscribe('order.status_changed') do |*args|
      event = ActiveSupport::Notifications::Event.new(*args)
      new.handle_status_change(event.payload)
    end

    ActiveSupport::Notifications.subscribe('order.completed') do |*args|
      event = ActiveSupport::Notifications::Event.new(*args)
      new.handle_completion(event.payload)
    end
  end

  def handle_status_change(payload)
    order     = Order.find(payload[:order_id])
    old_status = payload[:old_status]
    new_status = payload[:new_status]

    Rails.logger.info "Order #{order.id}: #{old_status} -> #{new_status}"
    send_status_notification(order, old_status, new_status)
  end

  def handle_completion(payload)
    order = Order.find(payload[:order_id])

    OrderMailer.receipt(order).deliver_later
    UpdateInventoryJob.perform_later(order.id)
    Analytics.track(event: 'order_completed', order_id: order.id, total: order.total)
  end

  private

  def send_status_notification(order, old_status, new_status)
    # Only notify on significant changes
    notify = %w[processing shipped delivered cancelled]
    return unless new_status.in?(notify)

    OrderMailer.status_update(order, new_status).deliver_later
  end
end

# Model instruments events
class Order < ApplicationRecord
  before_update :track_status_change
  after_commit  :instrument_events

  private

  def track_status_change
    @status_changed_from = status_was if will_save_change_to_status?
  end

  def instrument_events
    if @status_changed_from
      ActiveSupport::Notifications.instrument('order.status_changed', {
        order_id:   id,
        old_status: @status_changed_from,
        new_status: status
      })

      if status == 'completed'
        ActiveSupport::Notifications.instrument('order.completed', { order_id: id })
      end
    end
  end
end

# initializer
# config/initializers/observers.rb
OrderObserver.subscribe!
```

### Custom Event System

```ruby
# lib/event_bus.rb
module EventBus
  @subscribers = Hash.new { |h, k| h[k] = [] }

  def self.subscribe(event_name, &handler)
    @subscribers[event_name.to_s] << handler
  end

  def self.publish(event_name, payload = {})
    event_name = event_name.to_s
    @subscribers[event_name].each do |handler|
      handler.call(payload)
    end
  rescue => e
    Rails.logger.error "EventBus error for #{event_name}: #{e.message}\n#{e.backtrace.first(5).join("\n")}"
  end

  def self.clear!
    @subscribers.clear
  end
end

# Subscribe
EventBus.subscribe('user.created') { |p| UserMailer.welcome(p[:user]).deliver_later }
EventBus.subscribe('user.created') { |p| AnalyticsJob.perform_later(:user_created, p[:user_id]) }

# Publish
EventBus.publish('user.created', user: user, user_id: user.id)
```

---

## 9. Strategy Pattern

**จุดประสงค์:** Define a family of algorithms, make them interchangeable

```ruby
# app/strategies/payment/base_strategy.rb
module Payment
  class BaseStrategy
    def initialize(amount:, currency: 'THB')
      @amount   = amount
      @currency = currency
    end

    def charge(payment_details)
      raise NotImplementedError
    end

    def refund(transaction_id, amount: nil)
      raise NotImplementedError
    end

    def name
      self.class.name.demodulize.underscore.gsub('_strategy', '')
    end

    private

    def validate_amount!
      raise ArgumentError, "Amount must be positive" unless @amount.positive?
    end
  end
end

# app/strategies/payment/stripe_strategy.rb
module Payment
  class StripeStrategy < BaseStrategy
    def charge(payment_details)
      validate_amount!

      charge = Stripe::PaymentIntent.create(
        amount:   (@amount * 100).to_i,  # Stripe uses cents
        currency: @currency.downcase,
        payment_method: payment_details[:payment_method_id],
        confirm: true,
        metadata: { user_id: payment_details[:user_id] }
      )

      {
        success:        true,
        transaction_id: charge.id,
        amount:         @amount,
        currency:       @currency
      }
    rescue Stripe::CardError => e
      { success: false, error: e.message }
    rescue Stripe::StripeError => e
      Rails.logger.error "Stripe error: #{e.message}"
      { success: false, error: "Payment processing failed" }
    end

    def refund(transaction_id, amount: nil)
      params = { payment_intent: transaction_id }
      params[:amount] = (amount * 100).to_i if amount

      refund = Stripe::Refund.create(params)
      { success: true, refund_id: refund.id }
    rescue => e
      { success: false, error: e.message }
    end
  end
end

# app/strategies/payment/paypal_strategy.rb
module Payment
  class PaypalStrategy < BaseStrategy
    def charge(payment_details)
      validate_amount!

      response = PaypalClient.charge(
        amount:   @amount,
        currency: @currency,
        token:    payment_details[:paypal_token]
      )

      {
        success:        response.approved?,
        transaction_id: response.transaction_id,
        amount:         @amount,
        currency:       @currency
      }
    end

    def refund(transaction_id, amount: nil)
      PaypalClient.refund(transaction_id, amount: amount)
    end
  end
end

# Context class
class PaymentProcessor
  STRATEGIES = {
    'stripe'  => Payment::StripeStrategy,
    'paypal'  => Payment::PaypalStrategy
  }.freeze

  def initialize(provider:, amount:, currency: 'THB')
    strategy_class = STRATEGIES[provider.to_s]
    raise ArgumentError, "Unknown payment provider: #{provider}" unless strategy_class

    @strategy = strategy_class.new(amount: amount, currency: currency)
  end

  def charge(payment_details)
    result = @strategy.charge(payment_details)
    log_transaction(result)
    result
  end

  def refund(transaction_id, amount: nil)
    @strategy.refund(transaction_id, amount: amount)
  end

  private

  def log_transaction(result)
    if result[:success]
      Rails.logger.info "Payment successful: #{result[:transaction_id]}"
    else
      Rails.logger.warn "Payment failed: #{result[:error]}"
    end
  end
end

# Usage
processor = PaymentProcessor.new(provider: 'stripe', amount: 1000, currency: 'THB')
result = processor.charge(payment_method_id: 'pm_xxx', user_id: user.id)
```

---

## 10. Template Method Pattern

**จุดประสงค์:** Define skeleton of algorithm, let subclasses fill in steps

```ruby
# app/importers/base_importer.rb
class BaseImporter
  def import(file_path)
    data    = parse_file(file_path)
    records = transform(data)
    results = []

    ActiveRecord::Base.transaction do
      records.each do |record|
        result = process_record(record)
        results << result
      end

      after_import(results)
    end

    build_report(results)
  rescue => e
    handle_error(e)
  end

  private

  # Template methods - subclasses MUST implement
  def parse_file(file_path)
    raise NotImplementedError, "#{self.class}#parse_file not implemented"
  end

  def process_record(record)
    raise NotImplementedError, "#{self.class}#process_record not implemented"
  end

  # Template methods - subclasses CAN override (hooks)
  def transform(data)
    data
  end

  def after_import(results)
    # Default: do nothing
  end

  def handle_error(error)
    Rails.logger.error "Import failed: #{error.message}"
    raise error
  end

  def build_report(results)
    {
      total:     results.size,
      succeeded: results.count { |r| r[:success] },
      failed:    results.count { |r| !r[:success] }
    }
  end
end

# CSV Importer
class UserCsvImporter < BaseImporter
  private

  def parse_file(file_path)
    CSV.read(file_path, headers: true).map(&:to_h)
  end

  def transform(data)
    data.map do |row|
      {
        name:  row['Name']&.strip,
        email: row['Email']&.downcase&.strip,
        role:  row['Role']&.downcase || 'user'
      }
    end
  end

  def process_record(record)
    user = User.find_or_initialize_by(email: record[:email])
    user.assign_attributes(name: record[:name], role: record[:role])

    if user.save
      { success: true, email: record[:email], action: user.new_record? ? :created : :updated }
    else
      { success: false, email: record[:email], errors: user.errors.full_messages }
    end
  end

  def after_import(results)
    successful = results.select { |r| r[:success] }
    Rails.logger.info "Imported #{successful.count} users"
  end
end

# Excel Importer
class ProductExcelImporter < BaseImporter
  private

  def parse_file(file_path)
    spreadsheet = Roo::Spreadsheet.open(file_path)
    sheet = spreadsheet.sheet(0)
    headers = sheet.row(1)

    sheet.each_row_streaming(offset: 1).map do |row|
      headers.zip(row.map(&:value)).to_h
    end
  end

  def process_record(record)
    product = Product.find_or_initialize_by(sku: record['SKU'])
    product.assign_attributes(
      name:  record['Product Name'],
      price: record['Price'].to_f,
      stock: record['Stock'].to_i
    )

    if product.save
      { success: true, sku: record['SKU'] }
    else
      { success: false, sku: record['SKU'], errors: product.errors.full_messages }
    end
  end
end
```

---

## 11. Null Object Pattern

**จุดประสงค์:** แทน nil ด้วย object ที่มี behavior ที่เหมาะสม

```ruby
# แทนที่จะตรวจ nil ทุกที่
# if current_user
#   current_user.name
# else
#   "Guest"
# end

# Null Object
class NullUser
  def id          = nil
  def name        = "Guest"
  def email       = nil
  def role        = "guest"
  def admin?      = false
  def signed_in?  = false
  def authenticated? = false
  def permissions = []

  def to_s = "Guest"
  def inspect = "#<NullUser>"

  def nil? = true  # ดูเหมือน nil จาก caller's perspective
end

class ApplicationController < ActionController::Base
  helper_method :current_user

  def current_user
    @current_user ||= begin
      if session[:user_id]
        User.find_by(id: session[:user_id]) || NullUser.new
      else
        NullUser.new
      end
    end
  end
end

# ใช้งาน - ไม่ต้องตรวจ nil
current_user.name   # => "Alice" หรือ "Guest"
current_user.admin? # => true หรือ false
current_user.signed_in?  # => true หรือ false

# Null Object สำหรับ Settings
class NullSettings
  def [](key) = nil
  def get(key, default = nil) = default
  def respond_to_missing?(*)  = true
  def method_missing(*) = nil
end

class User < ApplicationRecord
  def settings
    super || NullSettings.new
  end
end

user = User.new  # ไม่มี settings
user.settings[:theme]                # => nil (ไม่ raise!)
user.settings.get(:theme, 'light')  # => 'light'
```

---

## 12. Command Pattern

**จุดประสงค์:** Encapsulate requests as objects (undo/redo support)

```ruby
# app/commands/base_command.rb
class BaseCommand
  def self.execute(...)
    cmd = new(...)
    cmd.execute
    cmd
  end

  def execute
    raise NotImplementedError
  end

  def undo
    raise NotImplementedError
  end

  def successful? = @successful
  def error       = @error
end

# app/commands/update_user_role_command.rb
class UpdateUserRoleCommand < BaseCommand
  def initialize(user:, new_role:, performed_by:)
    @user         = user
    @new_role     = new_role
    @performed_by = performed_by
    @old_role     = user.role
  end

  def execute
    @user.update!(role: @new_role)
    log_action("changed role: #{@old_role} → #{@new_role}")
    @successful = true
  rescue => e
    @error      = e.message
    @successful = false
  end

  def undo
    @user.update!(role: @old_role)
    log_action("reverted role: #{@new_role} → #{@old_role}")
  end

  private

  def log_action(description)
    AuditLog.create!(
      user:        @performed_by,
      target_type: 'User',
      target_id:   @user.id,
      action:      'role_change',
      description: description
    )
  end
end

# History for undo
class CommandHistory
  def initialize
    @history = []
  end

  def execute(command)
    command.execute
    @history << command if command.successful?
    command
  end

  def undo_last
    command = @history.pop
    command&.undo
    command
  end

  def history
    @history.dup.freeze
  end
end
```

---

## 13. Builder Pattern

**จุดประสงค์:** สร้าง complex objects ทีละขั้นตอน

```ruby
# app/builders/report_builder.rb
class ReportBuilder
  def initialize
    @filters  = {}
    @columns  = []
    @sorts    = []
    @formats  = [:html]
    @limit    = nil
  end

  def for_user(user)
    @user = user
    self
  end

  def with_date_range(start_date, end_date)
    @filters[:date_range] = start_date..end_date
    self
  end

  def with_status(*statuses)
    @filters[:status] = statuses.flatten
    self
  end

  def include_columns(*columns)
    @columns.concat(columns)
    self
  end

  def sort_by(column, direction: :asc)
    @sorts << { column: column, direction: direction }
    self
  end

  def limit(n)
    @limit = n
    self
  end

  def export_as(*formats)
    @formats = formats.flatten
    self
  end

  def build
    Report.new(
      user:    @user,
      filters: @filters,
      columns: @columns.presence || default_columns,
      sorts:   @sorts.presence || default_sorts,
      formats: @formats,
      limit:   @limit
    )
  end

  private

  def default_columns
    [:id, :created_at, :status, :total]
  end

  def default_sorts
    [{ column: :created_at, direction: :desc }]
  end
end

# Usage
report = ReportBuilder.new
  .for_user(current_user)
  .with_date_range(1.month.ago, Time.current)
  .with_status(:completed, :refunded)
  .include_columns(:id, :customer, :total, :status)
  .sort_by(:total, direction: :desc)
  .limit(100)
  .export_as(:pdf, :csv)
  .build

report.generate
```

---

## 14. Interactor Pattern

**จุดประสงค์:** Organize business logic ด้วย Context object

```ruby
# gem 'interactor'
# app/interactors/authenticate_user.rb
class AuthenticateUser
  include Interactor

  def call
    user = User.find_by(email: context.email)

    unless user&.authenticate(context.password)
      context.fail!(message: "Invalid credentials")
    end

    unless user.active?
      context.fail!(message: "Account is deactivated")
    end

    context.user  = user
    context.token = user.generate_jwt_token
  end
end

# Organizer - chains interactors
class UserSignIn
  include Interactor::Organizer

  organize AuthenticateUser,
           TrackSignIn,
           UpdateLastSeen
end

# class TrackSignIn
#   include Interactor
#   def call
#     SignInLog.create!(user: context.user, ip: context.ip_address)
#   end
# end

# Controller
def create
  result = UserSignIn.call(
    email:      params[:email],
    password:   params[:password],
    ip_address: request.remote_ip
  )

  if result.success?
    render json: { token: result.token }
  else
    render json: { error: result.message }, status: :unauthorized
  end
end
```

---

## 15. Specification Pattern

**จุดประสงค์:** Encapsulate business rules ที่ combinable

```ruby
# app/specifications/base_specification.rb
class BaseSpecification
  def satisfied_by?(candidate)
    raise NotImplementedError
  end

  def &(other)
    AndSpecification.new(self, other)
  end

  def |(other)
    OrSpecification.new(self, other)
  end

  def !
    NotSpecification.new(self)
  end
end

class AndSpecification < BaseSpecification
  def initialize(spec1, spec2)
    @spec1 = spec1
    @spec2 = spec2
  end

  def satisfied_by?(candidate)
    @spec1.satisfied_by?(candidate) && @spec2.satisfied_by?(candidate)
  end
end

class OrSpecification < BaseSpecification
  def initialize(spec1, spec2)
    @spec1 = spec1
    @spec2 = spec2
  end

  def satisfied_by?(candidate)
    @spec1.satisfied_by?(candidate) || @spec2.satisfied_by?(candidate)
  end
end

class NotSpecification < BaseSpecification
  def initialize(spec)
    @spec = spec
  end

  def satisfied_by?(candidate)
    !@spec.satisfied_by?(candidate)
  end
end

# Concrete specifications
class ActiveUserSpec < BaseSpecification
  def satisfied_by?(user)
    user.active? && !user.banned?
  end
end

class PremiumUserSpec < BaseSpecification
  def satisfied_by?(user)
    user.subscription&.active?
  end
end

class VerifiedEmailSpec < BaseSpecification
  def satisfied_by?(user)
    user.email_verified?
  end
end

class AdminSpec < BaseSpecification
  def satisfied_by?(user)
    user.role == 'admin'
  end
end

# Usage
active_user      = ActiveUserSpec.new
premium_user     = PremiumUserSpec.new
verified_email   = VerifiedEmailSpec.new
admin            = AdminSpec.new

# Combine
can_create_post  = active_user & verified_email
can_access_api   = active_user & (premium_user | admin)
requires_upgrade = active_user & !premium_user

can_create_post.satisfied_by?(user)
requires_upgrade.satisfied_by?(user)

# Convert to AR scope
class UserQuerySpec
  def self.to_scope(specification)
    # Map spec to AR conditions
    case specification
    when ActiveUserSpec
      User.where(active: true, banned: false)
    when PremiumUserSpec
      User.joins(:subscription).merge(Subscription.active)
    # ...
    end
  end
end
```

---

## 16. Facade Pattern

**จุดประสงค์:** Provide simplified interface to complex subsystem

```ruby
# app/facades/dashboard_facade.rb
class DashboardFacade
  def initialize(user)
    @user = user
  end

  # Single method แทนที่จะ query ทุก subsystem ใน controller
  def stats
    @stats ||= {
      total_orders:    orders_stats,
      notifications:   notification_count,
      recent_activity: recent_activity,
      recommendations: recommendations
    }
  end

  def orders_stats
    {
      total:     @user.orders.count,
      pending:   @user.orders.pending.count,
      completed: @user.orders.completed.count,
      this_month: @user.orders.this_month.sum(:total)
    }
  end

  def notification_count
    @user.notifications.unread.count
  end

  def recent_activity
    @user.activity_logs
         .includes(:target)
         .order(created_at: :desc)
         .limit(5)
  end

  def recommendations
    RecommendationEngine.for_user(@user).top(5)
  end
end

# Controller
class DashboardController < ApplicationController
  def index
    @facade = DashboardFacade.new(current_user)
    # View uses @facade.stats, @facade.orders_stats, etc.
  end
end
```

---

## สรุปและเปรียบเทียบ Patterns

| Pattern | ใช้เมื่อ | ประโยชน์ |
|---------|----------|---------|
| **Service Object** | Business logic ซับซ้อน | ง่ายต่อการ test, single responsibility |
| **Repository** | ต้องการแยก data access | Swap storage backends ง่าย |
| **Presenter/Decorator** | Presentation logic หลาย views | ไม่ให้ logic อยู่ใน view |
| **Form Object** | Form ที่ update หลาย models | Clean validation, no fat models |
| **Query Object** | Complex DB queries | Reusable, testable queries |
| **Value Object** | Domain concepts (Money, Email) | Immutable, equality by value |
| **Policy Object** | Authorization rules | Centralized permissions |
| **Observer** | React to events | Loose coupling |
| **Strategy** | Interchangeable algorithms | Open/Closed principle |
| **Template Method** | Skeleton algorithm | Reuse common steps |
| **Null Object** | Handle nil gracefully | No nil checks everywhere |
| **Command** | Encapsulate actions/undo | Audit trail, reversibility |
| **Builder** | Complex object creation | Fluent API |
| **Specification** | Combinable business rules | Flexible filters |
| **Facade** | Complex subsystem | Simple interface |

### หลักการเลือก Pattern

1. **YAGNI** - อย่าใส่ Pattern ถ้าไม่จำเป็น
2. **Single Responsibility** - แต่ละ class ทำสิ่งเดียว
3. **Testability** - เขียน unit test ง่าย
4. **Readability** - code อ่านเข้าใจง่าย
5. **Start Simple** - เริ่มง่าย refactor ทีหลัง

### Anti-Patterns ที่ควรหลีกเลี่ยง

```ruby
# Fat Controller - ย้าย logic ไป Service
def create
  # 50 lines of business logic...
end

# Fat Model - ย้าง behavior ไป Concern หรือ Service
class User
  # 500 lines...
  def self.import_from_csv(file)
    # 100 lines of import logic
  end
end

# Callback hell - ใช้ Service แทน
class Order
  after_create :send_email
  after_create :update_inventory
  after_create :notify_warehouse
  after_create :track_analytics
  # ยากต่อการ test และ debug
end

# God Object - class ที่รู้เรื่องทุกอย่าง
class ApplicationHelper
  # มี 200 helper methods
end
```
