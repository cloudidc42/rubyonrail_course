# ตอนที่ 81: Advanced Rails Patterns

## สารบัญ
1. Decorator Pattern
2. Presenter Pattern
3. Repository Pattern
4. Query Object Pattern
5. Form Object Pattern ขั้นสูง
6. Policy Object Pattern
7. Domain Model Patterns
8. Anti-corruption Layer
9. Service Object ขั้นสูง
10. แบบฝึกหัด 20 ข้อพร้อมเฉลย

---

## 1. Decorator Pattern

### ทำความเข้าใจ Decorator Pattern

Decorator Pattern คือ design pattern ที่ช่วยให้เราสามารถเพิ่ม behavior หรือ responsibility ให้กับ object ได้แบบ dynamic โดยไม่ต้องแก้ไข class เดิม เหมาะสำหรับกรณีที่ต้องการ wrap object เพื่อเพิ่มฟีเจอร์

ใน Rails มักใช้ Decorator Pattern เพื่อ:
- จัดการ view-related logic ออกจาก Model
- เพิ่ม formatting methods
- Compose behaviors หลายอย่างเข้าด้วยกัน

### การสร้าง Decorator ด้วย SimpleDelegator

```ruby
# app/decorators/base_decorator.rb
class BaseDecorator < SimpleDelegator
  def initialize(object)
    super(object)
    @object = object
  end

  def self.decorate(object)
    new(object)
  end

  def self.decorate_collection(collection)
    collection.map { |obj| new(obj) }
  end

  # เข้าถึง object ต้นฉบับ
  def object
    __getobj__
  end

  def class
    __getobj__.class
  end

  def is_a?(klass)
    __getobj__.is_a?(klass) || super
  end
end
```

```ruby
# app/decorators/user_decorator.rb
class UserDecorator < BaseDecorator
  def full_name
    "#{first_name} #{last_name}".strip
  end

  def display_name
    if full_name.present?
      full_name
    else
      email
    end
  end

  def formatted_created_at
    created_at.strftime("%d %B %Y เวลา %H:%M")
  end

  def avatar_url
    if avatar.attached?
      Rails.application.routes.url_helpers.url_for(avatar)
    else
      "https://ui-avatars.com/api/?name=#{CGI.escape(display_name)}"
    end
  end

  def role_badge
    case role
    when "admin"
      '<span class="badge badge-danger">ผู้ดูแลระบบ</span>'.html_safe
    when "moderator"
      '<span class="badge badge-warning">ผู้ดูแล</span>'.html_safe
    else
      '<span class="badge badge-secondary">สมาชิก</span>'.html_safe
    end
  end

  def status_indicator
    if active?
      "🟢 ออนไลน์"
    elsif last_seen_at&.> 30.minutes.ago
      "🟡 ไม่อยู่"
    else
      "🔴 ออฟไลน์"
    end
  end

  def account_age
    days = (Date.today - created_at.to_date).to_i
    if days < 30
      "#{days} วันที่แล้ว"
    elsif days < 365
      "#{days / 30} เดือนที่แล้ว"
    else
      "#{days / 365} ปีที่แล้ว"
    end
  end
end
```

```ruby
# app/decorators/post_decorator.rb
class PostDecorator < BaseDecorator
  include ActionView::Helpers::TextHelper
  include ActionView::Helpers::UrlHelper

  def truncated_body(length: 200)
    truncate(body, length: length, separator: " ")
  end

  def reading_time
    words = body.split.size
    minutes = (words / 200.0).ceil
    "#{minutes} นาที"
  end

  def formatted_published_at
    return "ยังไม่ได้เผยแพร่" unless published_at

    if published_at.today?
      "วันนี้ #{published_at.strftime('%H:%M')}"
    elsif published_at.yesterday?
      "เมื่อวาน #{published_at.strftime('%H:%M')}"
    else
      published_at.strftime("%d %B %Y")
    end
  end

  def status_text
    case status
    when "draft"    then "ร่าง"
    when "published" then "เผยแพร่แล้ว"
    when "archived"  then "เก็บถาวร"
    end
  end

  def cover_image_url(size: :medium)
    if cover_image.attached?
      Rails.application.routes.url_helpers.url_for(
        cover_image.variant(resize_to_limit: size_for(size))
      )
    else
      "/images/default_cover.jpg"
    end
  end

  def author_info
    UserDecorator.decorate(author)
  end

  private

  def size_for(size)
    case size
    when :thumbnail then [150, 150]
    when :medium    then [600, 400]
    when :large     then [1200, 800]
    end
  end
end
```

### การใช้ Decorator ใน Controller และ View

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = PostDecorator.decorate_collection(
      Post.published.includes(:author).page(params[:page])
    )
  end

  def show
    @post = PostDecorator.decorate(Post.find(params[:id]))
    @author = UserDecorator.decorate(@post.author)
  end
end
```

```erb
<%# app/views/posts/show.html.erb %>
<article>
  <h1><%= @post.title %></h1>
  <p>โดย <%= @post.author_info.display_name %></p>
  <p>เวลาอ่าน: <%= @post.reading_time %></p>
  <p>เผยแพร่: <%= @post.formatted_published_at %></p>
  
  <img src="<%= @post.cover_image_url(size: :large) %>" alt="<%= @post.title %>">
  
  <div class="content">
    <%= @post.body %>
  </div>
</article>
```

### Decorator ด้วย Draper Gem

```ruby
# Gemfile
gem 'draper'
```

```ruby
# app/decorators/article_decorator.rb
class ArticleDecorator < Draper::Decorator
  delegate_all

  def formatted_date
    object.created_at.strftime("%d/%m/%Y")
  end

  def summary(length = 100)
    h.truncate(object.content, length: length)
  end

  def author_link
    h.link_to object.author.name, h.user_path(object.author)
  end

  def category_badges
    object.categories.map do |cat|
      h.content_tag(:span, cat.name, class: "badge badge-#{cat.color}")
    end.join(" ").html_safe
  end
end

# app/decorators/article_collection_decorator.rb
class ArticleCollectionDecorator < Draper::CollectionDecorator
  def total_reading_time
    sum { |article| article.estimated_minutes }
  end
end
```

```ruby
# การใช้งานใน controller
class ArticlesController < ApplicationController
  def index
    @articles = ArticleCollectionDecorator.decorate(
      Article.published.recent
    )
  end

  def show
    @article = Article.find(params[:id]).decorate
  end
end
```

---

## 2. Presenter Pattern

### ความแตกต่างระหว่าง Decorator และ Presenter

**Decorator**: ขยาย behavior ของ single object
**Presenter**: จัดการ view logic สำหรับ view template โดยเฉพาะ และอาจรวม data จากหลาย model

```ruby
# app/presenters/base_presenter.rb
class BasePresenter
  include ActionView::Helpers::TagHelper
  include ActionView::Helpers::TextHelper
  include ActionView::Helpers::UrlHelper
  include Rails.application.routes.url_helpers

  def initialize(view_context)
    @view = view_context
  end

  protected

  def h
    @view
  end
end
```

```ruby
# app/presenters/dashboard_presenter.rb
class DashboardPresenter < BasePresenter
  def initialize(user, view_context)
    super(view_context)
    @user = user
  end

  def greeting
    hour = Time.current.hour
    name = @user.first_name

    if hour < 12
      "สวัสดีตอนเช้า #{name}!"
    elsif hour < 17
      "สวัสดีตอนบ่าย #{name}!"
    else
      "สวัสดีตอนเย็น #{name}!"
    end
  end

  def stats
    @stats ||= {
      total_posts: @user.posts.count,
      published_posts: @user.posts.published.count,
      total_views: @user.posts.sum(:views_count),
      total_comments: Comment.where(post: @user.posts).count,
      followers_count: @user.followers.count,
      following_count: @user.following.count
    }
  end

  def recent_activity
    @user.activities
         .includes(:subject)
         .order(created_at: :desc)
         .limit(10)
         .map { |activity| format_activity(activity) }
  end

  def notifications_summary
    unread = @user.notifications.unread.count
    return "ไม่มีการแจ้งเตือนใหม่" if unread.zero?

    "คุณมีการแจ้งเตือนใหม่ #{unread} รายการ"
  end

  def chart_data
    {
      labels: last_7_days.map { |d| d.strftime("%d/%m") },
      datasets: [
        {
          label: "การเข้าชม",
          data: views_per_day
        }
      ]
    }
  end

  private

  def format_activity(activity)
    case activity.action
    when "created_post"
      "สร้างบทความ: #{activity.subject.title}"
    when "got_comment"
      "ได้รับความคิดเห็นใน: #{activity.subject.post.title}"
    when "got_follower"
      "#{activity.subject.follower.name} ติดตามคุณ"
    else
      activity.description
    end
  end

  def last_7_days
    7.downto(0).map { |i| i.days.ago.to_date }
  end

  def views_per_day
    last_7_days.map do |date|
      @user.posts.joins(:page_views)
           .where(page_views: { created_at: date.all_day })
           .count
    end
  end
end
```

```ruby
# app/controllers/dashboards_controller.rb
class DashboardsController < ApplicationController
  def show
    @presenter = DashboardPresenter.new(current_user, view_context)
  end
end
```

```erb
<%# app/views/dashboards/show.html.erb %>
<div class="dashboard">
  <h1><%= @presenter.greeting %></h1>
  
  <div class="stats-grid">
    <div class="stat">
      <span class="number"><%= @presenter.stats[:total_posts] %></span>
      <span class="label">บทความทั้งหมด</span>
    </div>
    <div class="stat">
      <span class="number"><%= @presenter.stats[:total_views] %></span>
      <span class="label">การเข้าชมทั้งหมด</span>
    </div>
  </div>
  
  <p class="notifications"><%= @presenter.notifications_summary %></p>
  
  <h2>กิจกรรมล่าสุด</h2>
  <ul>
    <% @presenter.recent_activity.each do |activity| %>
      <li><%= activity %></li>
    <% end %>
  </ul>
</div>
```

---

## 3. Repository Pattern

### ทำความเข้าใจ Repository Pattern

Repository Pattern คือ abstraction layer ระหว่าง domain logic กับ data access logic ช่วยให้:
- แยก business logic ออกจาก database queries
- ง่ายต่อการ test (ใช้ mock repository)
- เปลี่ยน data source ได้ง่าย

```ruby
# app/repositories/base_repository.rb
class BaseRepository
  def initialize(model_class)
    @model = model_class
  end

  def find(id)
    @model.find(id)
  rescue ActiveRecord::RecordNotFound
    nil
  end

  def find!(id)
    @model.find(id)
  end

  def all
    @model.all
  end

  def create(attributes)
    @model.create(attributes)
  end

  def update(id, attributes)
    record = find!(id)
    record.update(attributes)
    record
  end

  def delete(id)
    find!(id).destroy
  end

  def count
    @model.count
  end

  protected

  def scope
    @model.all
  end
end
```

```ruby
# app/repositories/user_repository.rb
class UserRepository < BaseRepository
  def initialize
    super(User)
  end

  def find_by_email(email)
    scope.find_by(email: email.downcase)
  end

  def find_active
    scope.where(active: true)
  end

  def find_by_role(role)
    scope.where(role: role)
  end

  def find_with_posts
    scope.includes(:posts)
  end

  def find_recent(limit: 10)
    scope.order(created_at: :desc).limit(limit)
  end

  def search(query)
    scope.where(
      "first_name ILIKE :q OR last_name ILIKE :q OR email ILIKE :q",
      q: "%#{query}%"
    )
  end

  def find_by_username(username)
    scope.find_by(username: username)
  end

  def admins
    find_by_role("admin")
  end

  def active_in_last_days(days)
    scope.where(last_seen_at: days.days.ago..)
  end

  def stats
    {
      total: count,
      active: find_active.count,
      admins: admins.count,
      new_this_month: scope.where(created_at: Time.current.beginning_of_month..).count
    }
  end
end
```

```ruby
# app/repositories/post_repository.rb
class PostRepository < BaseRepository
  def initialize
    super(Post)
  end

  def published
    scope.where(status: "published").where("published_at <= ?", Time.current)
  end

  def drafts_for(user)
    scope.where(author: user, status: "draft")
  end

  def find_by_slug(slug)
    scope.find_by(slug: slug)
  end

  def find_popular(limit: 10)
    published.order(views_count: :desc).limit(limit)
  end

  def find_by_category(category)
    published.joins(:categories).where(categories: { id: category.id })
  end

  def find_by_tag(tag_name)
    published.tagged_with(tag_name)
  end

  def search(query)
    published.where(
      "title ILIKE :q OR body ILIKE :q",
      q: "%#{query}%"
    )
  end

  def related_to(post, limit: 5)
    published
      .where.not(id: post.id)
      .joins(:tags)
      .where(tags: { id: post.tag_ids })
      .group("posts.id")
      .order("COUNT(tags.id) DESC")
      .limit(limit)
  end

  def archive_old(days: 365)
    scope.where(
      status: "published",
      published_at: ...days.days.ago
    ).update_all(status: "archived")
  end
end
```

```ruby
# การใช้งาน Repository ใน Service
class PublishPostService
  def initialize(post_repo: PostRepository.new, user_repo: UserRepository.new)
    @post_repo = post_repo
    @user_repo = user_repo
  end

  def call(post_id, publisher_id)
    post = @post_repo.find!(post_id)
    publisher = @user_repo.find!(publisher_id)

    unless publisher.can_publish?
      return ServiceResult.failure("ไม่มีสิทธิ์เผยแพร่บทความ")
    end

    @post_repo.update(post_id, {
      status: "published",
      published_at: Time.current,
      published_by: publisher
    })

    ServiceResult.success(post)
  end
end
```

```ruby
# การ inject Repository สำหรับ testing
class PostsController < ApplicationController
  def initialize
    @post_repo = PostRepository.new
    @user_repo = UserRepository.new
    super
  end

  def index
    @posts = @post_repo.published
    @popular = @post_repo.find_popular(limit: 5)
  end

  def show
    @post = @post_repo.find_by_slug(params[:slug])
    return head :not_found unless @post

    @related = @post_repo.related_to(@post)
  end
end
```

---

## 4. Query Object Pattern

### ความสำคัญของ Query Object

Query Object ช่วยแยก complex database queries ออกจาก Model และ Controller ทำให้:
- Query logic นำกลับมาใช้ใหม่ได้
- ง่ายต่อการ test
- อ่านง่ายขึ้น

```ruby
# app/queries/base_query.rb
class BaseQuery
  def initialize(scope = nil)
    @scope = scope || default_scope
  end

  def call
    raise NotImplementedError, "#{self.class}#call ต้อง implement"
  end

  def self.call(scope = nil)
    new(scope).call
  end

  private

  def default_scope
    raise NotImplementedError, "#{self.class}#default_scope ต้อง implement"
  end
end
```

```ruby
# app/queries/posts/published_query.rb
module Posts
  class PublishedQuery < BaseQuery
    def initialize(scope = nil, options = {})
      super(scope)
      @options = options
    end

    def call
      scope
        .where(status: :published)
        .where("published_at <= ?", Time.current)
        .order(published_at: :desc)
    end

    private

    def default_scope
      Post.all
    end
  end
end
```

```ruby
# app/queries/posts/search_query.rb
module Posts
  class SearchQuery < BaseQuery
    def initialize(scope = nil, query:, filters: {})
      super(scope)
      @query = query
      @filters = filters
    end

    def call
      result = scope
      result = apply_text_search(result)
      result = apply_category_filter(result)
      result = apply_date_filter(result)
      result = apply_tag_filter(result)
      result
    end

    private

    def default_scope
      Post.published
    end

    def apply_text_search(scope)
      return scope if @query.blank?

      scope.where(
        "to_tsvector('thai', title || ' ' || body) @@ plainto_tsquery('thai', ?)",
        @query
      )
    end

    def apply_category_filter(scope)
      return scope unless @filters[:category_id].present?

      scope.joins(:categories)
           .where(categories: { id: @filters[:category_id] })
    end

    def apply_date_filter(scope)
      return scope unless @filters[:from_date].present?

      from = Date.parse(@filters[:from_date]) rescue nil
      to = @filters[:to_date].present? ? Date.parse(@filters[:to_date]) : Date.current

      return scope unless from

      scope.where(published_at: from..to)
    end

    def apply_tag_filter(scope)
      return scope unless @filters[:tags].present?

      scope.tagged_with(@filters[:tags], any: true)
    end
  end
end
```

```ruby
# app/queries/users/active_contributors_query.rb
module Users
  class ActiveContributorsQuery < BaseQuery
    def initialize(scope = nil, since: 30.days.ago, min_posts: 3)
      super(scope)
      @since = since
      @min_posts = min_posts
    end

    def call
      scope
        .joins(:posts)
        .where(posts: { status: :published, published_at: @since.. })
        .group("users.id")
        .having("COUNT(posts.id) >= ?", @min_posts)
        .select("users.*, COUNT(posts.id) as posts_count")
        .order("posts_count DESC")
    end

    private

    def default_scope
      User.active
    end
  end
end
```

```ruby
# app/queries/analytics/revenue_query.rb
module Analytics
  class RevenueQuery < BaseQuery
    def initialize(scope = nil, period:, group_by: :day)
      super(scope)
      @period = period
      @group_by = group_by
    end

    def call
      scope
        .where(created_at: @period)
        .where(status: :completed)
        .group(group_expression)
        .sum(:amount)
    end

    def total
      scope
        .where(created_at: @period)
        .where(status: :completed)
        .sum(:amount)
    end

    def by_product
      scope
        .where(created_at: @period)
        .joins(:product)
        .group("products.name")
        .sum(:amount)
    end

    private

    def default_scope
      Order.all
    end

    def group_expression
      case @group_by
      when :day
        "DATE(created_at AT TIME ZONE 'Asia/Bangkok')"
      when :week
        "DATE_TRUNC('week', created_at AT TIME ZONE 'Asia/Bangkok')"
      when :month
        "DATE_TRUNC('month', created_at AT TIME ZONE 'Asia/Bangkok')"
      end
    end
  end
end
```

```ruby
# การใช้งาน Query Objects
class PostsController < ApplicationController
  def index
    @posts = Posts::PublishedQuery.call
                                  .page(params[:page])
                                  .per(20)
  end

  def search
    @posts = Posts::SearchQuery.call(
      query: params[:q],
      filters: {
        category_id: params[:category],
        from_date: params[:from],
        to_date: params[:to],
        tags: params[:tags]
      }
    )
  end
end

class AnalyticsController < ApplicationController
  def revenue
    query = Analytics::RevenueQuery.new(
      period: params[:start_date]..params[:end_date],
      group_by: params[:group_by]&.to_sym || :day
    )

    @revenue_by_period = query.call
    @total_revenue = query.total
    @revenue_by_product = query.by_product
  end
end
```

---

## 5. Form Object Pattern ขั้นสูง

### Form Object คืออะไร

Form Object คือ object ที่ represent form data โดยเฉพาะ แยกออกจาก Model ช่วยให้:
- จัดการ multi-model forms ได้
- Validation logic แยกจาก Model
- เหมาะสำหรับ complex business workflows

```ruby
# app/forms/base_form.rb
class BaseForm
  include ActiveModel::Model
  include ActiveModel::Attributes
  include ActiveModel::Validations

  def self.from_params(params)
    new(params.permit(*attribute_names).to_h)
  end

  def submit
    return false unless valid?

    persist!
    true
  rescue ActiveRecord::RecordInvalid => e
    errors.merge!(e.record.errors)
    false
  end

  private

  def persist!
    raise NotImplementedError, "#{self.class}#persist! ต้อง implement"
  end
end
```

```ruby
# app/forms/registration_form.rb
class RegistrationForm < BaseForm
  attribute :first_name, :string
  attribute :last_name, :string
  attribute :email, :string
  attribute :password, :string
  attribute :password_confirmation, :string
  attribute :date_of_birth, :date
  attribute :terms_accepted, :boolean

  validates :first_name, presence: true, length: { minimum: 2, maximum: 50 }
  validates :last_name, presence: true, length: { minimum: 2, maximum: 50 }
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 },
            format: {
              with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
              message: "ต้องมีตัวพิมพ์เล็ก ตัวพิมพ์ใหญ่ และตัวเลข"
            }
  validates :password_confirmation, presence: true
  validates :terms_accepted, acceptance: true

  validate :passwords_match
  validate :email_not_taken
  validate :age_requirement

  def user
    @user
  end

  private

  def persist!
    @user = User.create!(
      first_name: first_name,
      last_name: last_name,
      email: email,
      password: password,
      date_of_birth: date_of_birth
    )

    UserMailer.welcome_email(@user).deliver_later
    ActivityLog.record(@user, "registered")
  end

  def passwords_match
    errors.add(:password_confirmation, "รหัสผ่านไม่ตรงกัน") if password != password_confirmation
  end

  def email_not_taken
    errors.add(:email, "อีเมลนี้ถูกใช้งานแล้ว") if User.exists?(email: email)
  end

  def age_requirement
    return unless date_of_birth

    age = ((Date.current - date_of_birth) / 365).floor
    errors.add(:date_of_birth, "ต้องมีอายุอย่างน้อย 13 ปี") if age < 13
  end
end
```

```ruby
# app/forms/checkout_form.rb
class CheckoutForm < BaseForm
  # Billing info
  attribute :billing_first_name, :string
  attribute :billing_last_name, :string
  attribute :billing_address, :string
  attribute :billing_city, :string
  attribute :billing_postal_code, :string
  attribute :billing_country, :string

  # Shipping info
  attribute :same_as_billing, :boolean, default: true
  attribute :shipping_first_name, :string
  attribute :shipping_last_name, :string
  attribute :shipping_address, :string
  attribute :shipping_city, :string
  attribute :shipping_postal_code, :string
  attribute :shipping_country, :string

  # Payment
  attribute :payment_method, :string
  attribute :card_number, :string
  attribute :card_expiry, :string
  attribute :card_cvv, :string

  # Order
  attribute :cart_id, :integer
  attribute :coupon_code, :string
  attribute :notes, :string

  validates :billing_first_name, :billing_last_name, :billing_address,
            :billing_city, :billing_postal_code, :billing_country, presence: true
  validates :payment_method, presence: true, inclusion: { in: %w[credit_card bank_transfer promptpay] }

  with_options if: :credit_card? do
    validates :card_number, presence: true, format: { with: /\A\d{16}\z/ }
    validates :card_expiry, presence: true, format: { with: /\A\d{2}\/\d{2}\z/ }
    validates :card_cvv, presence: true, format: { with: /\A\d{3,4}\z/ }
  end

  validate :cart_must_exist
  validate :coupon_valid_if_present

  def order
    @order
  end

  private

  def persist!
    ActiveRecord::Base.transaction do
      @order = Order.create!(order_attributes)
      @order.create_shipping_address!(shipping_address_attributes)
      process_payment!
      clear_cart!
      apply_coupon! if coupon_code.present?
    end
  end

  def order_attributes
    {
      user: current_user,
      cart: cart,
      billing_address: billing_address_attributes,
      payment_method: payment_method,
      notes: notes
    }
  end

  def billing_address_attributes
    {
      first_name: billing_first_name,
      last_name: billing_last_name,
      address: billing_address,
      city: billing_city,
      postal_code: billing_postal_code,
      country: billing_country
    }
  end

  def shipping_address_attributes
    if same_as_billing?
      billing_address_attributes
    else
      {
        first_name: shipping_first_name,
        last_name: shipping_last_name,
        address: shipping_address,
        city: shipping_city,
        postal_code: shipping_postal_code,
        country: shipping_country
      }
    end
  end

  def credit_card?
    payment_method == "credit_card"
  end

  def cart
    @cart ||= Cart.find(cart_id)
  end

  def cart_must_exist
    unless Cart.exists?(cart_id)
      errors.add(:cart_id, "ตะกร้าสินค้าไม่พบ")
    end
  end

  def coupon_valid_if_present
    return if coupon_code.blank?

    coupon = Coupon.find_by(code: coupon_code)
    unless coupon&.valid_for_use?
      errors.add(:coupon_code, "รหัสคูปองไม่ถูกต้องหรือหมดอายุ")
    end
  end

  def process_payment!
    PaymentService.charge(
      order: @order,
      method: payment_method,
      card_number: card_number,
      card_expiry: card_expiry,
      card_cvv: card_cvv
    )
  end

  def clear_cart!
    cart.clear!
  end

  def apply_coupon!
    coupon = Coupon.find_by(code: coupon_code)
    coupon&.apply_to!(@order)
  end
end
```

---

## 6. Policy Object Pattern

### การใช้ Policy Object สำหรับ Authorization

Policy Object ช่วยจัดการ authorization logic ให้เป็นระเบียบ แยกออกจาก Controller และ Model

```ruby
# app/policies/base_policy.rb
class BasePolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def index?
    false
  end

  def show?
    false
  end

  def create?
    false
  end

  def update?
    false
  end

  def destroy?
    false
  end

  def scope
    Pundit.policy_scope!(user, record.class)
  end

  class Scope
    attr_reader :user, :scope

    def initialize(user, scope)
      @user = user
      @scope = scope
    end

    def resolve
      raise NotImplementedError, "#{self.class}#resolve ต้อง implement"
    end
  end

  protected

  def admin?
    user&.admin?
  end

  def owner_or_admin?
    admin? || owner?
  end

  def owner?
    return false unless record.respond_to?(:user)
    record.user == user
  end
end
```

```ruby
# app/policies/post_policy.rb
class PostPolicy < BasePolicy
  def index?
    true # ทุกคนดูรายการได้
  end

  def show?
    return true if record.published?
    return false unless user

    owner? || admin? || moderator?
  end

  def create?
    user.present? && user.active? && !user.banned?
  end

  def update?
    return false unless user
    return true if admin?
    return true if moderator?

    owner? && !record.archived?
  end

  def destroy?
    return false unless user
    return true if admin?

    owner? && record.draft?
  end

  def publish?
    return false unless user

    (owner? || admin?) && record.draft? && record.ready_to_publish?
  end

  def feature?
    admin? || moderator?
  end

  def archive?
    admin?
  end

  class Scope < BasePolicy::Scope
    def resolve
      if user&.admin? || user&.moderator?
        scope.all
      elsif user
        scope.where(status: :published)
             .or(scope.where(author: user))
      else
        scope.where(status: :published)
      end
    end
  end

  private

  def moderator?
    user&.moderator?
  end
end
```

```ruby
# app/policies/comment_policy.rb
class CommentPolicy < BasePolicy
  def create?
    user.present? && user.active? && !user.banned? && record.post.allows_comments?
  end

  def update?
    return false unless user
    return true if admin?

    owner? && within_edit_window?
  end

  def destroy?
    return false unless user
    return true if admin?

    owner? || post_author?
  end

  def approve?
    admin? || moderator?
  end

  def flag?
    user.present?
  end

  class Scope < BasePolicy::Scope
    def resolve
      if user&.admin?
        scope.all
      else
        scope.where(approved: true).or(scope.where(user: user))
      end
    end
  end

  private

  def within_edit_window?
    record.created_at > 15.minutes.ago
  end

  def post_author?
    record.post.author == user
  end

  def moderator?
    user&.moderator?
  end
end
```

```ruby
# การใช้งานใน Controller
class PostsController < ApplicationController
  include Pundit::Authorization

  def show
    @post = Post.find(params[:id])
    authorize @post
  end

  def update
    @post = Post.find(params[:id])
    authorize @post

    if @post.update(post_params)
      redirect_to @post, notice: "อัปเดตบทความสำเร็จ"
    else
      render :edit
    end
  end

  def publish
    @post = Post.find(params[:id])
    authorize @post, :publish?

    @post.publish!
    redirect_to @post, notice: "เผยแพร่บทความสำเร็จ"
  end

  def index
    @posts = policy_scope(Post)
  end

  private

  def post_params
    params.require(:post).permit(:title, :body, :category_id)
  end
end
```

---

## 7. Domain Model Patterns

### Value Object

Value Object คือ object ที่ defined ด้วย value ของมัน ไม่ใช่ identity เช่น Money, Address, Email

```ruby
# app/models/value_objects/money.rb
class Money
  include Comparable

  attr_reader :amount, :currency

  def initialize(amount, currency = "THB")
    @amount = BigDecimal(amount.to_s)
    @currency = currency.upcase
  end

  def +(other)
    ensure_same_currency!(other)
    Money.new(amount + other.amount, currency)
  end

  def -(other)
    ensure_same_currency!(other)
    Money.new(amount - other.amount, currency)
  end

  def *(multiplier)
    Money.new(amount * multiplier, currency)
  end

  def /(divisor)
    Money.new(amount / divisor, currency)
  end

  def <=>(other)
    ensure_same_currency!(other)
    amount <=> other.amount
  end

  def zero?
    amount.zero?
  end

  def positive?
    amount.positive?
  end

  def negative?
    amount.negative?
  end

  def to_s
    formatted = amount.truncate(2).to_s("F")
    "#{formatted} #{currency}"
  end

  def format_thai
    amount_int = amount.to_i
    formatted = number_to_currency(amount_int, unit: "฿", precision: 2, separator: ".", delimiter: ",")
    formatted
  end

  def ==(other)
    other.is_a?(Money) && amount == other.amount && currency == other.currency
  end

  def eql?(other)
    self == other
  end

  def hash
    [amount, currency].hash
  end

  private

  def ensure_same_currency!(other)
    raise ArgumentError, "ไม่สามารถดำเนินการกับสกุลเงินต่างกัน" unless currency == other.currency
  end
end
```

```ruby
# app/models/value_objects/address.rb
class Address
  attr_reader :street, :district, :city, :province, :postal_code, :country

  def initialize(attributes = {})
    @street      = attributes[:street]
    @district    = attributes[:district]
    @city        = attributes[:city]
    @province    = attributes[:province]
    @postal_code = attributes[:postal_code]
    @country     = attributes[:country] || "TH"
  end

  def full_address
    [street, district, city, province, postal_code, country_name]
      .compact
      .reject(&:blank?)
      .join(", ")
  end

  def country_name
    ISO3166::Country[country]&.translations["th"] || country
  end

  def ==(other)
    other.is_a?(Address) &&
      street == other.street &&
      city == other.city &&
      postal_code == other.postal_code
  end

  def to_h
    {
      street: street,
      district: district,
      city: city,
      province: province,
      postal_code: postal_code,
      country: country
    }
  end
end
```

### Entity

Entity คือ object ที่มี unique identity และมี lifecycle

```ruby
# app/models/entities/order_aggregate.rb
class OrderAggregate
  attr_reader :order, :events

  def initialize(order)
    @order = order
    @events = []
  end

  def add_item(product, quantity)
    raise DomainError, "ไม่สามารถเพิ่มสินค้าในออเดอร์ที่ยืนยันแล้ว" unless order.pending?

    line_item = order.line_items.build(
      product: product,
      quantity: quantity,
      price: product.current_price
    )

    if line_item.valid?
      order.recalculate_totals
      events << ItemAddedEvent.new(order: order, product: product, quantity: quantity)
    else
      raise DomainError, line_item.errors.full_messages.join(", ")
    end
  end

  def remove_item(product)
    raise DomainError, "ไม่สามารถลบสินค้าในออเดอร์ที่ยืนยันแล้ว" unless order.pending?

    item = order.line_items.find_by(product: product)
    raise DomainError, "ไม่พบสินค้าในออเดอร์" unless item

    item.destroy
    order.recalculate_totals
    events << ItemRemovedEvent.new(order: order, product: product)
  end

  def confirm
    raise DomainError, "ออเดอร์ว่างเปล่า" if order.line_items.empty?
    raise DomainError, "ออเดอร์ถูกยืนยันแล้ว" unless order.pending?

    order.update!(
      status: :confirmed,
      confirmed_at: Time.current
    )
    events << OrderConfirmedEvent.new(order: order)
  end

  def publish_events!
    events.each { |event| EventBus.publish(event) }
    events.clear
  end
end
```

---

## 8. Anti-corruption Layer (ACL)

### ทำความเข้าใจ ACL

Anti-corruption Layer เป็น pattern ที่ป้องกันไม่ให้ concept จาก external system เข้ามา "ปนเปื้อน" domain model ของเรา

```ruby
# app/adapters/payment_gateway_adapter.rb
class PaymentGatewayAdapter
  class PaymentResult
    attr_reader :success, :transaction_id, :error_message, :amount

    def initialize(attrs = {})
      @success = attrs[:success]
      @transaction_id = attrs[:transaction_id]
      @error_message = attrs[:error_message]
      @amount = attrs[:amount]
    end

    def success?
      @success
    end

    def failure?
      !success?
    end
  end

  def charge(order:, payment_method:, amount:)
    response = external_gateway.charge(
      amount: (amount * 100).to_i, # แปลงเป็น satang
      currency: "THB",
      payment_method_id: payment_method.gateway_id,
      metadata: {
        order_id: order.id,
        customer_email: order.user.email
      }
    )

    translate_response(response)
  rescue ExternalGateway::NetworkError => e
    PaymentResult.new(success: false, error_message: "ไม่สามารถเชื่อมต่อกับระบบชำระเงิน")
  rescue ExternalGateway::InvalidCardError => e
    PaymentResult.new(success: false, error_message: "ข้อมูลบัตรไม่ถูกต้อง")
  end

  def refund(transaction_id:, amount: nil)
    response = external_gateway.refund(
      payment_intent_id: transaction_id,
      amount: amount ? (amount * 100).to_i : nil
    )

    PaymentResult.new(
      success: response.status == "succeeded",
      transaction_id: response.id
    )
  end

  private

  def translate_response(response)
    if response.status == "succeeded"
      PaymentResult.new(
        success: true,
        transaction_id: response.id,
        amount: response.amount / 100.0
      )
    else
      PaymentResult.new(
        success: false,
        error_message: translate_error(response.last_payment_error&.code)
      )
    end
  end

  def translate_error(code)
    error_messages = {
      "card_declined"       => "บัตรถูกปฏิเสธ กรุณาติดต่อธนาคาร",
      "insufficient_funds"  => "ยอดเงินในบัญชีไม่เพียงพอ",
      "expired_card"        => "บัตรหมดอายุ กรุณาใช้บัตรใหม่",
      "incorrect_cvc"       => "รหัส CVC ไม่ถูกต้อง",
      "processing_error"    => "เกิดข้อผิดพลาดในการประมวลผล กรุณาลองใหม่"
    }

    error_messages[code] || "เกิดข้อผิดพลาดในการชำระเงิน"
  end

  def external_gateway
    @external_gateway ||= Stripe::PaymentIntents
  end
end
```

```ruby
# app/adapters/weather_api_adapter.rb
class WeatherApiAdapter
  WeatherData = Struct.new(:temperature, :humidity, :description, :wind_speed, :feels_like) do
    def hot?
      temperature > 35
    end

    def rainy?
      description.include?("rain") || description.include?("shower")
    end
  end

  def current_weather(city)
    response = fetch_weather(city)
    translate_to_weather_data(response)
  rescue StandardError
    nil
  end

  private

  def fetch_weather(city)
    conn = Faraday.new("https://api.openweathermap.org") do |f|
      f.response :json
      f.response :raise_error
    end

    conn.get("/data/2.5/weather", {
      q: city,
      appid: ENV["OPENWEATHER_API_KEY"],
      units: "metric",
      lang: "th"
    })
  end

  def translate_to_weather_data(response)
    data = response.body
    WeatherData.new(
      data.dig("main", "temp"),
      data.dig("main", "humidity"),
      data.dig("weather", 0, "description"),
      data.dig("wind", "speed"),
      data.dig("main", "feels_like")
    )
  end
end
```

---

## 9. Service Object ขั้นสูง

### Result Object Pattern

```ruby
# app/services/result.rb
class Result
  attr_reader :value, :errors

  def initialize(success:, value: nil, errors: [])
    @success = success
    @value = value
    @errors = Array(errors)
  end

  def self.success(value = nil)
    new(success: true, value: value)
  end

  def self.failure(errors = [])
    new(success: false, errors: Array(errors))
  end

  def success?
    @success
  end

  def failure?
    !success?
  end

  def on_success
    yield(value) if success?
    self
  end

  def on_failure
    yield(errors) if failure?
    self
  end

  def map
    return self if failure?
    Result.success(yield(value))
  end

  def flat_map
    return self if failure?
    yield(value)
  end
end
```

```ruby
# app/services/create_post_service.rb
class CreatePostService
  def initialize(user:, params:)
    @user = user
    @params = params
  end

  def call
    return Result.failure("ผู้ใช้ไม่มีสิทธิ์สร้างบทความ") unless @user.can_create_post?

    post = Post.new(post_attributes)

    if post.save
      after_save(post)
      Result.success(post)
    else
      Result.failure(post.errors.full_messages)
    end
  end

  private

  def post_attributes
    {
      title:      @params[:title],
      body:       @params[:body],
      author:     @user,
      status:     :draft,
      slug:       generate_slug(@params[:title]),
      category:   Category.find(@params[:category_id])
    }
  end

  def generate_slug(title)
    base_slug = title.parameterize
    slug = base_slug
    counter = 1

    while Post.exists?(slug: slug)
      slug = "#{base_slug}-#{counter}"
      counter += 1
    end

    slug
  end

  def after_save(post)
    NotificationService.notify_followers(@user, post)
    SearchIndexJob.perform_later(post.id)
    ActivityLog.record(@user, :created_post, post)
  end
end
```

```ruby
# Pipeline pattern สำหรับ service composition
class ServicePipeline
  def initialize
    @steps = []
  end

  def pipe(service_class, **options)
    @steps << [service_class, options]
    self
  end

  def call(initial_value)
    @steps.reduce(Result.success(initial_value)) do |result, (service_class, options)|
      result.flat_map do |value|
        service_class.new(**options, value: value).call
      end
    end
  end
end

# การใช้งาน
result = ServicePipeline.new
  .pipe(ValidateOrderService)
  .pipe(ReserveInventoryService)
  .pipe(ProcessPaymentService)
  .pipe(SendConfirmationService)
  .call(order)

result
  .on_success { |order| redirect_to order, notice: "สั่งซื้อสำเร็จ" }
  .on_failure { |errors| flash[:error] = errors.join(", ") }
```

---

## แบบฝึกหัด 20 ข้อพร้อมเฉลย

### ข้อที่ 1: สร้าง Product Decorator
**โจทย์**: สร้าง ProductDecorator ที่มี methods:
- `formatted_price` - แสดงราคาในรูปแบบ "1,250.00 บาท"
- `discount_badge` - แสดง badge เมื่อมีส่วนลด
- `availability_status` - แสดงสถานะสินค้า

**เฉลย**:
```ruby
class ProductDecorator < BaseDecorator
  def formatted_price
    "#{number_with_precision(price, precision: 2, delimiter: ',', separator: '.')} บาท"
  end

  def original_price_formatted
    return nil unless original_price

    "#{number_with_precision(original_price, precision: 2, delimiter: ',', separator: '.')} บาท"
  end

  def discount_badge
    return nil unless on_sale?

    percentage = ((1 - price / original_price) * 100).round
    "<span class='badge badge-danger'>ลด #{percentage}%</span>".html_safe
  end

  def availability_status
    if stock_quantity > 10
      { text: "มีสินค้า", color: "success" }
    elsif stock_quantity > 0
      { text: "สินค้าใกล้หมด (เหลือ #{stock_quantity})", color: "warning" }
    else
      { text: "สินค้าหมด", color: "danger" }
    end
  end

  def in_stock?
    stock_quantity > 0
  end

  private

  def number_with_precision(number, precision:, delimiter:, separator:)
    parts = ("%.#{precision}f" % number).split(".")
    integer_part = parts[0].chars.reverse.each_slice(3).map(&:join).join(delimiter).reverse
    "#{integer_part}#{separator}#{parts[1]}"
  end
end
```

### ข้อที่ 2: Order Presenter
**โจทย์**: สร้าง OrderPresenter ที่รวม order, customer, products data แสดงสรุปออเดอร์

**เฉลย**:
```ruby
class OrderPresenter < BasePresenter
  def initialize(order, view_context)
    super(view_context)
    @order = order
  end

  def order_number
    "#ORD-#{@order.id.to_s.rjust(8, '0')}"
  end

  def status_badge
    colors = {
      pending:    "warning",
      confirmed:  "info",
      shipped:    "primary",
      delivered:  "success",
      cancelled:  "danger"
    }
    status_labels = {
      pending:    "รอยืนยัน",
      confirmed:  "ยืนยันแล้ว",
      shipped:    "จัดส่งแล้ว",
      delivered:  "ส่งถึงแล้ว",
      cancelled:  "ยกเลิก"
    }
    color = colors[@order.status.to_sym]
    label = status_labels[@order.status.to_sym]
    h.content_tag(:span, label, class: "badge badge-#{color}")
  end

  def customer_info
    customer = @order.user
    {
      name: "#{customer.first_name} #{customer.last_name}",
      email: customer.email,
      phone: customer.phone
    }
  end

  def items_summary
    @order.line_items.map do |item|
      {
        name: item.product.name,
        quantity: item.quantity,
        unit_price: format_currency(item.price),
        subtotal: format_currency(item.subtotal)
      }
    end
  end

  def totals
    {
      subtotal: format_currency(@order.subtotal),
      shipping: format_currency(@order.shipping_cost),
      discount: @order.discount > 0 ? format_currency(@order.discount) : nil,
      total: format_currency(@order.total)
    }
  end

  def estimated_delivery
    return "ไม่ระบุ" unless @order.shipped_at

    delivery_date = @order.shipped_at + @order.shipping_days.days
    delivery_date.strftime("%d %B %Y")
  end

  private

  def format_currency(amount)
    "฿#{number_with_delimiter(amount.round(2))}"
  end

  def number_with_delimiter(number)
    parts = number.to_s.split(".")
    parts[0] = parts[0].chars.reverse.each_slice(3).map(&:join).join(",").reverse
    parts.join(".")
  end
end
```

### ข้อที่ 3: สร้าง ProductRepository
**โจทย์**: สร้าง ProductRepository ที่มี methods สำหรับค้นหาสินค้าตามเงื่อนไขต่างๆ

**เฉลย**:
```ruby
class ProductRepository < BaseRepository
  def initialize
    super(Product)
  end

  def find_available
    scope.where(active: true).where("stock_quantity > 0")
  end

  def find_by_category(category_id)
    scope.where(category_id: category_id)
  end

  def find_featured
    scope.where(featured: true, active: true).order(featured_at: :desc)
  end

  def search(query, filters: {})
    result = scope.active
    result = result.where("name ILIKE ? OR description ILIKE ?", "%#{query}%", "%#{query}%") if query.present?
    result = result.where("price >= ?", filters[:min_price]) if filters[:min_price].present?
    result = result.where("price <= ?", filters[:max_price]) if filters[:max_price].present?
    result = result.where(category_id: filters[:category_id]) if filters[:category_id].present?
    result = result.where(brand_id: filters[:brand_id]) if filters[:brand_id].present?
    result
  end

  def find_related(product, limit: 4)
    scope.active
         .where(category_id: product.category_id)
         .where.not(id: product.id)
         .order(sold_count: :desc)
         .limit(limit)
  end

  def low_stock(threshold: 5)
    scope.where("stock_quantity > 0 AND stock_quantity <= ?", threshold)
  end

  def out_of_stock
    scope.where(stock_quantity: 0)
  end

  def bestsellers(limit: 10, period: 30.days)
    scope.joins(:order_items)
         .where(order_items: { created_at: period.ago.. })
         .group("products.id")
         .order("SUM(order_items.quantity) DESC")
         .limit(limit)
  end
end
```

### ข้อที่ 4: Advanced Search Query Object
**โจทย์**: สร้าง Query Object สำหรับค้นหา User แบบ advanced

**เฉลย**:
```ruby
class Users::AdvancedSearchQuery < BaseQuery
  def initialize(scope = nil, filters: {}, sort: {})
    super(scope)
    @filters = filters
    @sort = sort
  end

  def call
    result = scope
    result = filter_by_name(result)
    result = filter_by_role(result)
    result = filter_by_status(result)
    result = filter_by_joined_date(result)
    result = filter_by_activity(result)
    apply_sorting(result)
  end

  private

  def default_scope
    User.all
  end

  def filter_by_name(scope)
    return scope unless @filters[:name].present?

    scope.where(
      "first_name ILIKE :q OR last_name ILIKE :q OR email ILIKE :q",
      q: "%#{@filters[:name]}%"
    )
  end

  def filter_by_role(scope)
    return scope unless @filters[:role].present?
    scope.where(role: @filters[:role])
  end

  def filter_by_status(scope)
    case @filters[:status]
    when "active"   then scope.active
    when "inactive" then scope.inactive
    when "banned"   then scope.banned
    else scope
    end
  end

  def filter_by_joined_date(scope)
    return scope unless @filters[:joined_from].present?

    from = Date.parse(@filters[:joined_from]) rescue nil
    to = @filters[:joined_to].present? ? Date.parse(@filters[:joined_to]) : Date.current
    return scope unless from

    scope.where(created_at: from.beginning_of_day..to.end_of_day)
  end

  def filter_by_activity(scope)
    case @filters[:activity]
    when "active_last_week"
      scope.where(last_seen_at: 1.week.ago..)
    when "active_last_month"
      scope.where(last_seen_at: 1.month.ago..)
    when "never_active"
      scope.where(last_seen_at: nil)
    else
      scope
    end
  end

  def apply_sorting(scope)
    field = @sort[:field] || "created_at"
    direction = @sort[:direction] || "desc"
    allowed_fields = %w[first_name email created_at last_seen_at posts_count]
    return scope unless allowed_fields.include?(field)

    scope.order("#{field} #{direction}")
  end
end
```

### ข้อที่ 5: Profile Update Form Object
**โจทย์**: สร้าง ProfileUpdateForm ที่ validate และ update user profile

**เฉลย**:
```ruby
class ProfileUpdateForm < BaseForm
  attribute :first_name, :string
  attribute :last_name, :string
  attribute :bio, :string
  attribute :website, :string
  attribute :location, :string
  attribute :avatar

  attr_reader :user

  validates :first_name, presence: true, length: { 2..50 }
  validates :last_name, presence: true, length: { 2..50 }
  validates :bio, length: { maximum: 500 }
  validates :website, format: {
    with: /\Ahttps?:\/\/.+\z/,
    message: "ต้องเป็น URL ที่ขึ้นต้นด้วย http:// หรือ https://"
  }, allow_blank: true

  def initialize(user, params = {})
    @user = user
    super(params)
  end

  private

  def persist!
    @user.update!(
      first_name: first_name,
      last_name: last_name,
      bio: bio,
      website: website,
      location: location
    )

    if avatar.present?
      @user.avatar.attach(avatar)
    end
  end
end
```

### ข้อที่ 6: Organization Policy
**โจทย์**: สร้าง OrganizationPolicy สำหรับจัดการ multi-tenant organization

**เฉลย**:
```ruby
class OrganizationPolicy < BasePolicy
  def show?
    member?
  end

  def update?
    owner? || admin?
  end

  def destroy?
    owner?
  end

  def invite_member?
    owner? || admin?
  end

  def remove_member?
    owner? || admin?
  end

  def manage_billing?
    owner?
  end

  def view_analytics?
    owner? || admin?
  end

  class Scope < BasePolicy::Scope
    def resolve
      scope.joins(:memberships)
           .where(memberships: { user: user, status: :active })
    end
  end

  private

  def member?
    record.memberships.exists?(user: user, status: :active)
  end

  def admin?
    record.memberships.exists?(user: user, role: :admin, status: :active)
  end

  def owner?
    record.owner == user
  end
end
```

### ข้อที่ 7: สร้าง Email Value Object
**โจทย์**: สร้าง Email Value Object ที่ validate format และมี utility methods

**เฉลย**:
```ruby
class Email
  VALID_FORMAT = /\A[^@\s]+@[^@\s]+\.[^@\s]+\z/

  attr_reader :value

  def initialize(value)
    @value = value.to_s.downcase.strip
    raise ArgumentError, "อีเมลไม่ถูกต้อง: #{@value}" unless valid?
  end

  def self.try(value)
    new(value)
  rescue ArgumentError
    nil
  end

  def valid?
    value.match?(VALID_FORMAT)
  end

  def domain
    value.split("@").last
  end

  def local_part
    value.split("@").first
  end

  def masked
    local = local_part
    masked_local = local[0] + "*" * (local.length - 2) + local[-1]
    "#{masked_local}@#{domain}"
  end

  def ==(other)
    other.is_a?(Email) && value == other.value
  end

  def to_s
    value
  end

  def hash
    value.hash
  end
end

# การใช้งาน
email = Email.new("user@example.com")
puts email.domain      # "example.com"
puts email.masked      # "u**r@example.com"
puts email             # "user@example.com"
```

### ข้อที่ 8: Content Moderation Service Pipeline
**โจทย์**: สร้าง service pipeline สำหรับตรวจสอบ content ก่อนเผยแพร่

**เฉลย**:
```ruby
class ContentModerationPipeline
  STEPS = [
    SpamCheckService,
    ProfanityFilterService,
    LinkValidationService,
    ImageModerationService
  ].freeze

  def initialize(content)
    @content = content
    @results = []
  end

  def moderate!
    STEPS.each do |service_class|
      result = service_class.new(@content).check
      @results << result

      return ModerationResult.rejected(result.reason) if result.rejected?
    end

    ModerationResult.approved
  end

  def self.moderate!(content)
    new(content).moderate!
  end
end

class SpamCheckService
  def initialize(content)
    @content = content
  end

  def check
    spam_score = calculate_spam_score
    if spam_score > 0.8
      CheckResult.rejected("เนื้อหานี้ถูกตรวจพบว่าเป็น spam")
    else
      CheckResult.passed
    end
  end

  private

  def calculate_spam_score
    # logic การตรวจสอบ spam
    spam_indicators = [
      has_too_many_links?,
      has_excessive_caps?,
      has_spam_keywords?
    ]
    spam_indicators.count(true) / spam_indicators.length.to_f
  end

  def has_too_many_links?
    @content.body.scan(/https?:\/\//).length > 10
  end

  def has_excessive_caps?
    letters = @content.body.gsub(/[^a-zA-Z]/, '')
    return false if letters.empty?
    letters.count("A-Z").to_f / letters.length > 0.5
  end

  def has_spam_keywords?
    spam_words = %w[buy now click here free money winner]
    spam_words.any? { |word| @content.body.downcase.include?(word) }
  end
end
```

### ข้อที่ 9: Observer Pattern ใน Domain Events
**โจทย์**: สร้างระบบ domain events สำหรับ Order lifecycle

**เฉลย**:
```ruby
class OrderEventPublisher
  SUBSCRIBERS = Hash.new { |h, k| h[k] = [] }

  def self.subscribe(event_class, &handler)
    SUBSCRIBERS[event_class] << handler
  end

  def self.publish(event)
    SUBSCRIBERS[event.class].each { |handler| handler.call(event) }
  end
end

# Events
OrderPlaced = Struct.new(:order, :placed_at)
OrderShipped = Struct.new(:order, :tracking_number, :shipped_at)
OrderDelivered = Struct.new(:order, :delivered_at)
OrderCancelled = Struct.new(:order, :reason, :cancelled_at)

# Subscribers
OrderEventPublisher.subscribe(OrderPlaced) do |event|
  OrderMailer.confirmation(event.order).deliver_later
  InventoryService.reserve_items(event.order)
end

OrderEventPublisher.subscribe(OrderShipped) do |event|
  OrderMailer.shipping_notification(event.order, event.tracking_number).deliver_later
  TrackingService.create_tracking(event.order, event.tracking_number)
end

OrderEventPublisher.subscribe(OrderCancelled) do |event|
  OrderMailer.cancellation_notice(event.order, event.reason).deliver_later
  InventoryService.release_items(event.order)
  RefundService.process_if_paid(event.order)
end

# การใช้งาน
class OrdersController < ApplicationController
  def create
    result = PlaceOrderService.new(current_user, order_params).call

    if result.success?
      OrderEventPublisher.publish(
        OrderPlaced.new(result.value, Time.current)
      )
      redirect_to result.value, notice: "สั่งซื้อสำเร็จ"
    else
      @errors = result.errors
      render :new
    end
  end
end
```

### ข้อที่ 10: สร้าง ACL สำหรับ SMS Provider
**โจทย์**: สร้าง Anti-corruption Layer สำหรับ SMS provider ที่สามารถเปลี่ยน provider ได้

**เฉลย**:
```ruby
# Domain interface
class SmsMessage
  attr_reader :to, :body, :sender_id

  def initialize(to:, body:, sender_id: nil)
    @to = normalize_phone(to)
    @body = body
    @sender_id = sender_id || "MYAPP"

    validate!
  end

  private

  def normalize_phone(phone)
    phone.gsub(/\D/, "").then do |digits|
      digits.start_with?("0") ? "+66#{digits[1..]}" : digits
    end
  end

  def validate!
    raise ArgumentError, "เบอร์โทรไม่ถูกต้อง" unless to.match?(/\A\+\d{10,15}\z/)
    raise ArgumentError, "ข้อความว่างเปล่า" if body.blank?
    raise ArgumentError, "ข้อความยาวเกินไป" if body.length > 160
  end
end

class SmsDeliveryResult
  attr_reader :message_id, :status, :error

  def initialize(message_id: nil, status:, error: nil)
    @message_id = message_id
    @status = status
    @error = error
  end

  def delivered?
    status == :delivered
  end

  def failed?
    status == :failed
  end
end

# Adapter สำหรับ Twilio
class TwilioSmsAdapter
  def send_message(sms_message)
    client = Twilio::REST::Client.new(
      ENV["TWILIO_ACCOUNT_SID"],
      ENV["TWILIO_AUTH_TOKEN"]
    )

    message = client.messages.create(
      to: sms_message.to,
      from: ENV["TWILIO_PHONE_NUMBER"],
      body: sms_message.body
    )

    SmsDeliveryResult.new(
      message_id: message.sid,
      status: map_status(message.status)
    )
  rescue Twilio::REST::RestError => e
    SmsDeliveryResult.new(status: :failed, error: e.message)
  end

  private

  def map_status(twilio_status)
    case twilio_status
    when "sent", "delivered" then :delivered
    when "failed", "undelivered" then :failed
    else :pending
    end
  end
end

# Service ใช้ interface ไม่ใช่ Twilio โดยตรง
class SmsNotificationService
  def initialize(adapter: nil)
    @adapter = adapter || TwilioSmsAdapter.new
  end

  def send_otp(phone:, otp:)
    message = SmsMessage.new(
      to: phone,
      body: "รหัส OTP ของคุณคือ: #{otp} (หมดอายุใน 5 นาที)"
    )

    @adapter.send_message(message)
  end

  def send_order_confirmation(order)
    message = SmsMessage.new(
      to: order.user.phone,
      body: "ยืนยันออเดอร์ ##{order.id} ยอด #{order.total} บาท ขอบคุณที่ใช้บริการ"
    )

    @adapter.send_message(message)
  end
end
```

### ข้อที่ 11-20: แบบฝึกหัดเพิ่มเติม

**ข้อที่ 11**: สร้าง CartPresenter ที่แสดงรายการสินค้าในตะกร้า พร้อมส่วนลดและภาษี

**เฉลย**:
```ruby
class CartPresenter < BasePresenter
  def initialize(cart, user, view_context)
    super(view_context)
    @cart = cart
    @user = user
  end

  def items
    @cart.cart_items.includes(:product).map do |item|
      {
        product: item.product.decorate,
        quantity: item.quantity,
        unit_price: format_price(item.product.price),
        subtotal: format_price(item.subtotal)
      }
    end
  end

  def summary
    subtotal = @cart.subtotal
    discount = calculate_discount(subtotal)
    vat = ((subtotal - discount) * 0.07).round(2)
    total = subtotal - discount + vat

    {
      subtotal: format_price(subtotal),
      discount: discount > 0 ? "-#{format_price(discount)}" : nil,
      vat: format_price(vat),
      total: format_price(total)
    }
  end

  def empty?
    @cart.cart_items.empty?
  end

  def item_count
    @cart.cart_items.sum(:quantity)
  end

  private

  def calculate_discount(subtotal)
    return 0 unless @user&.premium_member?
    (subtotal * 0.1).round(2)
  end

  def format_price(amount)
    "฿#{'%.2f' % amount}"
  end
end
```

**ข้อที่ 12**: สร้าง BillingRepository สำหรับจัดการ invoice data

**เฉลย**:
```ruby
class BillingRepository < BaseRepository
  def initialize
    super(Invoice)
  end

  def find_for_user(user, page: 1, per_page: 20)
    scope.where(user: user)
         .order(created_at: :desc)
         .page(page).per(per_page)
  end

  def find_overdue
    scope.where(status: :pending)
         .where("due_date < ?", Date.current)
  end

  def find_by_period(start_date, end_date)
    scope.where(created_at: start_date..end_date)
  end

  def monthly_revenue(year)
    scope.where(status: :paid)
         .where("EXTRACT(YEAR FROM paid_at) = ?", year)
         .group("EXTRACT(MONTH FROM paid_at)")
         .sum(:amount)
  end

  def unpaid_total_for(user)
    scope.where(user: user, status: :pending).sum(:amount)
  end
end
```

**ข้อที่ 13**: สร้าง Notification Query Object

**เฉลย**:
```ruby
class Notifications::UnreadQuery < BaseQuery
  def initialize(scope = nil, user:)
    super(scope)
    @user = user
  end

  def call
    scope.where(recipient: @user, read_at: nil)
         .order(created_at: :desc)
  end

  def count
    call.count
  end

  private

  def default_scope
    Notification.all
  end
end
```

**ข้อที่ 14**: สร้าง ImportForm สำหรับ import CSV

**เฉลย**:
```ruby
class ProductImportForm < BaseForm
  attribute :file
  attribute :overwrite_existing, :boolean, default: false
  attribute :notify_on_complete, :boolean, default: true

  validates :file, presence: true
  validate :file_must_be_csv
  validate :file_size_limit

  attr_reader :imported_count, :failed_count, :errors_list

  private

  def persist!
    @imported_count = 0
    @failed_count = 0
    @errors_list = []

    CSV.foreach(file.tempfile, headers: true, encoding: "UTF-8") do |row|
      import_row(row)
    end

    UserMailer.import_complete(
      imported: @imported_count,
      failed: @failed_count
    ).deliver_later if notify_on_complete
  end

  def import_row(row)
    product = find_or_initialize_product(row["sku"])

    if !overwrite_existing && product.persisted?
      @errors_list << "แถว #{$. }: สินค้า #{row['sku']} มีอยู่แล้ว (ข้ามไป)"
      @failed_count += 1
      return
    end

    product.assign_attributes(
      name: row["name"],
      price: row["price"].to_f,
      stock_quantity: row["stock"].to_i
    )

    if product.save
      @imported_count += 1
    else
      @errors_list << "แถว #{$.}: #{product.errors.full_messages.join(', ')}"
      @failed_count += 1
    end
  end

  def find_or_initialize_product(sku)
    Product.find_or_initialize_by(sku: sku)
  end

  def file_must_be_csv
    return unless file.present?
    errors.add(:file, "ต้องเป็นไฟล์ CSV") unless file.content_type == "text/csv"
  end

  def file_size_limit
    return unless file.present?
    errors.add(:file, "ไฟล์ใหญ่เกินไป (สูงสุด 5MB)") if file.size > 5.megabytes
  end
end
```

**ข้อที่ 15**: สร้าง Payment Policy

**เฉลย**:
```ruby
class PaymentPolicy < BasePolicy
  def create?
    user.present? && !user.banned? && order.belongs_to?(user) && order.pending?
  end

  def retry?
    user.present? && order.belongs_to?(user) && record.failed?
  end

  def refund?
    admin? || (order.belongs_to?(user) && within_refund_window?)
  end

  def view_details?
    admin? || order.belongs_to?(user)
  end

  private

  def order
    @order ||= record.order
  end

  def within_refund_window?
    record.created_at > 7.days.ago
  end
end
```

**ข้อที่ 16**: สร้าง DateRange Value Object

**เฉลย**:
```ruby
class DateRange
  attr_reader :start_date, :end_date

  def initialize(start_date, end_date)
    @start_date = start_date.to_date
    @end_date = end_date.to_date
    raise ArgumentError, "วันที่เริ่มต้องไม่เกินวันที่สิ้นสุด" if @start_date > @end_date
  end

  def include?(date)
    (start_date..end_date).include?(date.to_date)
  end

  def overlap?(other_range)
    start_date <= other_range.end_date && end_date >= other_range.start_date
  end

  def duration_in_days
    (end_date - start_date).to_i + 1
  end

  def to_range
    start_date..end_date
  end

  def to_s
    "#{start_date.strftime('%d/%m/%Y')} - #{end_date.strftime('%d/%m/%Y')}"
  end

  def ==(other)
    other.is_a?(DateRange) &&
      start_date == other.start_date &&
      end_date == other.end_date
  end
end
```

**ข้อที่ 17**: สร้าง Statistics Query Object

**เฉลย**:
```ruby
class Stats::SalesReportQuery < BaseQuery
  def initialize(scope = nil, period:)
    super(scope)
    @period = period
  end

  def call
    scope.where(created_at: @period)
         .where(status: :completed)
  end

  def total_revenue
    call.sum(:total_amount)
  end

  def average_order_value
    call.average(:total_amount)&.round(2) || 0
  end

  def orders_per_day
    call.group("DATE(created_at)").count
  end

  def top_products(limit: 5)
    Product.joins(line_items: :order)
           .where(orders: { created_at: @period, status: :completed })
           .group("products.id, products.name")
           .select("products.id, products.name, SUM(line_items.quantity) as total_sold")
           .order("total_sold DESC")
           .limit(limit)
  end

  private

  def default_scope
    Order.all
  end
end
```

**ข้อที่ 18**: สร้าง External API Adapter สำหรับ geolocation

**เฉลย**:
```ruby
class GeolocationAdapter
  Location = Struct.new(:latitude, :longitude, :city, :country, :postal_code)

  def lookup(ip_address)
    response = fetch_location(ip_address)
    return nil if response.nil?

    translate_response(response)
  end

  private

  def fetch_location(ip)
    conn = Faraday.new("https://ipapi.co") do |f|
      f.response :json
      f.options.timeout = 5
    end

    response = conn.get("/#{ip}/json/")
    return nil unless response.success?

    response.body
  rescue Faraday::Error
    nil
  end

  def translate_response(data)
    return nil if data["error"]

    Location.new(
      data["latitude"],
      data["longitude"],
      data["city"],
      data["country_name"],
      data["postal"]
    )
  end
end
```

**ข้อที่ 19**: สร้าง User Session Presenter

**เฉลย**:
```ruby
class UserSessionPresenter < BasePresenter
  def initialize(user, session_data, view_context)
    super(view_context)
    @user = user
    @session = session_data
  end

  def display_name
    @user.full_name.presence || @user.email
  end

  def avatar_tag(size: 40)
    if @user.avatar.attached?
      h.image_tag(
        @user.avatar.variant(resize_to_fill: [size, size]),
        class: "avatar",
        alt: display_name
      )
    else
      initials_avatar(size)
    end
  end

  def menu_items
    items = [
      { label: "โปรไฟล์", path: h.profile_path, icon: "user" },
      { label: "การตั้งค่า", path: h.settings_path, icon: "settings" }
    ]
    items << { label: "จัดการระบบ", path: h.admin_root_path, icon: "shield" } if @user.admin?
    items << { label: "ออกจากระบบ", path: h.logout_path, icon: "logout", method: :delete }
    items
  end

  private

  def initials_avatar(size)
    initials = [@user.first_name&.first, @user.last_name&.first].compact.join
    h.content_tag(:div, initials,
      class: "avatar-initials",
      style: "width:#{size}px;height:#{size}px;font-size:#{size / 2.5}px"
    )
  end
end
```

**ข้อที่ 20**: สร้าง Composite Policy สำหรับ multi-condition authorization

**เฉลย**:
```ruby
class CompositePolicy
  def initialize(user, record)
    @user = user
    @record = record
    @policies = []
  end

  def add_policy(policy_class)
    @policies << policy_class.new(@user, @record)
    self
  end

  def all_allow?(action)
    @policies.all? { |policy| policy.public_send("#{action}?") }
  end

  def any_allow?(action)
    @policies.any? { |policy| policy.public_send("#{action}?") }
  end
end

# การใช้งาน
class PremiumContentPolicy < BasePolicy
  def show?
    return false unless super
    return true if record.free?

    user.premium? || user.admin?
  end
end

class AgeGatingPolicy < BasePolicy
  def show?
    return true unless record.adult_content?
    return false unless user
    return false unless user.date_of_birth

    user.age >= 18
  end
end

class ContentAccessPolicy
  def initialize(user, content)
    @composite = CompositePolicy.new(user, content)
      .add_policy(PremiumContentPolicy)
      .add_policy(AgeGatingPolicy)
  end

  def can_view?
    @composite.all_allow?(:show)
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Advanced Rails Patterns ที่สำคัญ:

1. **Decorator Pattern** - ขยาย behavior ของ object โดยไม่แก้ไข class เดิม
2. **Presenter Pattern** - จัดการ view logic ที่ซับซ้อน
3. **Repository Pattern** - abstraction layer สำหรับ data access
4. **Query Object** - encapsulate database queries
5. **Form Object** - จัดการ form logic และ validation ที่ซับซ้อน
6. **Policy Object** - authorization logic ที่เป็นระเบียบ
7. **Value Object** - object ที่ defined ด้วย value
8. **Anti-corruption Layer** - ป้องกัน external system concepts เข้า domain

Pattern เหล่านี้ช่วยให้ code มีคุณภาพสูง maintainable และ testable มากขึ้น
