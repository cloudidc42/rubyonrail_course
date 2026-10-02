# Part 34: Controllers (ขั้นตอนที่ 731-755)

## บทนำ

Controllers เป็นส่วนที่ประสานงานระหว่าง Models กับ Views รับ HTTP request ดึงข้อมูลจาก model และส่งไปแสดงใน view ถ้า Rails เป็นร้านอาหาร controller ก็คือพนักงานเสิร์ฟที่รับออเดอร์และส่งอาหาร

---

## ขั้นตอนที่ 731: ApplicationController

### ApplicationController คืออะไร?

`ApplicationController` เป็น base class ที่ controllers ทุกตัวใน application inherit มา

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # ===== Security =====
  protect_from_forgery with: :exception

  # ===== Authentication =====
  before_action :authenticate_user!
  helper_method :current_user, :user_signed_in?

  # ===== Locale =====
  before_action :set_locale

  # ===== Logging =====
  before_action :log_request

  # ===== Error Handling =====
  rescue_from ActiveRecord::RecordNotFound, with: :handle_not_found
  rescue_from Pundit::NotAuthorizedError, with: :handle_unauthorized
  rescue_from ActionController::InvalidAuthenticityToken, with: :handle_invalid_token

  private

  def authenticate_user!
    unless user_signed_in?
      respond_to do |format|
        format.html { redirect_to login_path, alert: "กรุณาเข้าสู่ระบบก่อน" }
        format.json { render json: { error: "Unauthorized" }, status: :unauthorized }
      end
    end
  end

  def current_user
    @current_user ||= User.find(session[:user_id]) if session[:user_id]
  end

  def user_signed_in?
    current_user.present?
  end

  def set_locale
    I18n.locale = params[:locale] || session[:locale] || I18n.default_locale
    session[:locale] = I18n.locale
  end

  def log_request
    Rails.logger.info("#{request.method} #{request.path} by User #{current_user&.id}")
  end

  def handle_not_found(exception)
    Rails.logger.warn("Record not found: #{exception.message}")
    respond_to do |format|
      format.html { render "errors/not_found", status: :not_found }
      format.json { render json: { error: "Not found" }, status: :not_found }
    end
  end

  def handle_unauthorized(exception)
    respond_to do |format|
      format.html { redirect_to root_path, alert: "ไม่มีสิทธิ์เข้าถึง" }
      format.json { render json: { error: "Forbidden" }, status: :forbidden }
    end
  end

  def handle_invalid_token
    redirect_to root_path, alert: "Session หมดอายุ กรุณาลองใหม่อีกครั้ง"
  end
end
```

### ActionController::API vs ActionController::Base

```ruby
# สำหรับ API-only applications
# app/controllers/api/base_controller.rb
module Api
  class BaseController < ActionController::API
    include ActionController::HttpAuthentication::Token::ControllerMethods

    before_action :authenticate_api_token!

    rescue_from ActiveRecord::RecordNotFound do |e|
      render json: { error: e.message }, status: :not_found
    end

    rescue_from ActiveRecord::RecordInvalid do |e|
      render json: { errors: e.record.errors.full_messages }, status: :unprocessable_entity
    end

    private

    def authenticate_api_token!
      authenticate_or_request_with_http_token do |token, options|
        @current_api_user = User.find_by(api_token: token)
      end
    end

    def current_api_user
      @current_api_user
    end
  end
end
```

---

## ขั้นตอนที่ 732: CRUD Actions

### Complete CRUD Controller

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  before_action :authorize_article!, only: [:edit, :update, :destroy]

  # GET /articles
  def index
    @articles = Article.published
                       .includes(:user, :tags, :comments)
                       .order(created_at: :desc)

    # Filtering
    @articles = @articles.where(category_id: params[:category_id]) if params[:category_id].present?
    @articles = @articles.search(params[:q]) if params[:q].present?
    @articles = @articles.tagged_with(params[:tag]) if params[:tag].present?

    # Pagination
    @articles = @articles.page(params[:page]).per(params[:per_page] || 12)

    respond_to do |format|
      format.html
      format.json { render json: @articles }
    end
  end

  # GET /articles/:id
  def show
    @article.increment!(:views_count)
    @comments = @article.comments.includes(:user).order(created_at: :asc)
    @new_comment = Comment.new
    @related_articles = Article.published
                                .where.not(id: @article.id)
                                .tagged_with(@article.tag_list)
                                .limit(5)
  end

  # GET /articles/new
  def new
    @article = current_user.articles.build
    @article.build_seo_meta  # สร้าง associated record
  end

  # POST /articles
  def create
    @article = current_user.articles.build(article_params)

    respond_to do |format|
      if @article.save
        format.html { redirect_to @article, notice: "บทความถูกสร้างเรียบร้อยแล้ว" }
        format.json { render json: @article, status: :created, location: @article }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: { errors: @article.errors }, status: :unprocessable_entity }
      end
    end
  end

  # GET /articles/:id/edit
  def edit
    @article.build_seo_meta unless @article.seo_meta
  end

  # PATCH/PUT /articles/:id
  def update
    respond_to do |format|
      if @article.update(article_params)
        format.html { redirect_to @article, notice: "บทความถูกอัพเดทเรียบร้อยแล้ว" }
        format.json { render json: @article }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: { errors: @article.errors }, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /articles/:id
  def destroy
    @article.destroy!

    respond_to do |format|
      format.html { redirect_to articles_path, notice: "บทความถูกลบเรียบร้อยแล้ว" }
      format.json { head :no_content }
    end
  end

  private

  def set_article
    @article = Article.find(params[:id])
  end

  def article_params
    params.require(:article).permit(
      :title,
      :body,
      :status,
      :category_id,
      :featured_image,
      tag_ids: [],
      seo_meta_attributes: [:id, :title, :description, :keywords]
    )
  end

  def authorize_article!
    unless @article.user == current_user || current_user.admin?
      redirect_to articles_path, alert: "ไม่มีสิทธิ์แก้ไขบทความนี้"
    end
  end
end
```

---

## ขั้นตอนที่ 733: Strong Parameters

### Strong Parameters คืออะไร?

Strong Parameters ป้องกัน **Mass Assignment Vulnerability** - ป้องกันไม่ให้ user ส่งข้อมูลที่ไม่ต้องการมา update

```ruby
# ❌ อันตราย! (Rails 3 แบบเก่า)
# @user = User.new(params[:user])
# ถ้า user ส่ง { admin: true } มาด้วย จะถูก assign!

# ✅ ปลอดภัย (Rails 4+)
def user_params
  params.require(:user).permit(:name, :email, :password)
  # :admin ไม่ถูก permit จึงถูก filter ออกไป
end
```

### require และ permit

```ruby
# params.require(:model_name) → ต้องมี parameter นี้ (raise error ถ้าไม่มี)
# params.permit(:field_name) → อนุญาต parameter นี้

# Basic usage
def article_params
  params.require(:article).permit(:title, :body, :status)
end

# Nested attributes
def user_params
  params.require(:user).permit(
    :name,
    :email,
    :password,
    :password_confirmation,
    address_attributes: [:id, :street, :city, :country, :_destroy],
    profile_attributes: [:id, :bio, :avatar, :website]
  )
end

# Array parameters
def article_params
  params.require(:article).permit(
    :title,
    :body,
    tag_ids: [],           # Array of integers
    category_ids: [],      # Array of integers
    images: []             # Array of files
  )
end

# Hash parameters
def settings_params
  params.require(:settings).permit(
    :email_notifications,
    preferences: {},       # Hash ที่ไม่รู้โครงสร้าง
    notification_settings: [:new_comments, :new_followers]
  )
end

# Optional require (ไม่ raise error ถ้าไม่มี)
def search_params
  params.permit(:q, :category, :sort, :page, :per_page)
end

# ตรวจสอบ Strong Parameters
def create
  ap article_params  # ด้วย awesome_print gem
  # หรือ
  puts article_params.inspect
end
```

### Strong Parameters กับ Nested Forms

```ruby
# Model:
class Order < ApplicationRecord
  has_many :order_items, dependent: :destroy
  accepts_nested_attributes_for :order_items, allow_destroy: true, reject_if: :all_blank
end

# Controller:
def order_params
  params.require(:order).permit(
    :customer_name,
    :shipping_address,
    :payment_method,
    order_items_attributes: [
      :id,
      :product_id,
      :quantity,
      :unit_price,
      :_destroy  # อนุญาตการลบ nested record
    ]
  )
end
```

---

## ขั้นตอนที่ 734: Callbacks (before_action, after_action, around_action)

### before_action

```ruby
class ArticlesController < ApplicationController
  # รันก่อน ทุก action
  before_action :authenticate_user!

  # รันก่อน เฉพาะบาง actions
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  before_action :check_article_owner, only: [:edit, :update, :destroy]

  # ยกเว้นบาง actions
  before_action :require_admin!, except: [:index, :show]

  # รันแบบมีเงื่อนไข
  before_action :redirect_if_banned, if: :user_banned?
  before_action :show_maintenance_notice, unless: :maintenance_mode_off?

  def index
    @articles = Article.all
  end

  def show
    # @article พร้อมใช้แล้วจาก before_action
  end

  private

  def set_article
    @article = Article.find(params[:id])
  rescue ActiveRecord::RecordNotFound
    redirect_to articles_path, alert: "ไม่พบบทความ"
  end

  def check_article_owner
    unless @article.user == current_user
      redirect_to @article, alert: "คุณไม่ใช่เจ้าของบทความนี้"
    end
  end

  def require_admin!
    unless current_user&.admin?
      redirect_to root_path, alert: "ต้องเป็น Admin เท่านั้น"
    end
  end

  def user_banned?
    current_user&.banned?
  end

  def maintenance_mode_off?
    !Rails.application.config.maintenance_mode
  end
end
```

### after_action

```ruby
class ArticlesController < ApplicationController
  # รันหลัง action เสร็จ
  after_action :track_page_view, only: [:show]
  after_action :notify_subscribers, only: [:create]
  after_action :log_api_usage, if: :api_request?

  def show
    @article = Article.find(params[:id])
  end

  def create
    @article = Article.create!(article_params)
    redirect_to @article
  end

  private

  def track_page_view
    PageView.create!(
      path: request.path,
      user: current_user,
      ip_address: request.remote_ip,
      user_agent: request.user_agent
    )
  end

  def notify_subscribers
    return unless @article&.persisted?
    ArticleNotificationJob.perform_later(@article.id)
  end

  def log_api_usage
    ApiLog.create!(
      endpoint: request.path,
      method: request.method,
      user: current_user,
      response_status: response.status
    )
  end

  def api_request?
    request.format.json?
  end
end
```

### around_action

```ruby
class ArticlesController < ApplicationController
  around_action :track_execution_time
  around_action :wrap_in_transaction, only: [:create, :update, :destroy]

  private

  def track_execution_time
    start_time = Time.current
    yield  # รัน action
    end_time = Time.current

    duration = (end_time - start_time) * 1000
    Rails.logger.info("#{action_name} took #{duration.round(2)}ms")

    response.headers["X-Execution-Time"] = duration.to_s
  end

  def wrap_in_transaction
    ActiveRecord::Base.transaction do
      yield
    end
  rescue ActiveRecord::RecordInvalid => e
    flash[:alert] = e.message
    redirect_to :back
  end
end
```

### Skipping Callbacks

```ruby
class AdminArticlesController < ArticlesController
  # Skip inherited before_action
  skip_before_action :authenticate_user!, only: [:index]
  skip_before_action :check_article_owner
end
```

---

## ขั้นตอนที่ 735: render vs redirect_to

### render

```ruby
class ArticlesController < ApplicationController
  def new
    @article = Article.new
    # Default: render :new  (app/views/articles/new.html.erb)
  end

  def create
    @article = Article.new(article_params)
    if @article.save
      redirect_to @article  # ส่ง HTTP 302 แล้วให้ browser load URL ใหม่
    else
      render :new, status: :unprocessable_entity  # แสดง new form อีกครั้ง
    end
  end

  def show
    @article = Article.find(params[:id])

    # render template ต่างๆ
    render :show               # app/views/articles/show.html.erb
    render "articles/show"     # ระบุ path เต็ม
    render "shared/error"      # render template จาก folder อื่น

    # render inline HTML
    render html: "<h1>Hello</h1>".html_safe

    # render JSON
    render json: @article

    # render text
    render plain: "Hello World"

    # render file
    render file: Rails.root.join("public/404.html"), status: :not_found

    # render nothing
    head :ok

    # render กับ layout
    render :show, layout: "admin"
    render :show, layout: false  # ไม่มี layout
    render layout: "application"

    # render กับ status
    render :show, status: :ok       # 200
    render :new, status: :unprocessable_entity  # 422
    render :show, status: 404
  end
end
```

### redirect_to

```ruby
class ArticlesController < ApplicationController
  def create
    @article = Article.create!(article_params)

    # redirect กับ path helper
    redirect_to articles_path

    # redirect กับ object
    redirect_to @article  # Rails รู้ว่าต้อง redirect ไป article_path(@article)

    # redirect กับ URL
    redirect_to "https://example.com"

    # redirect กับ flash message
    redirect_to @article, notice: "สร้างเรียบร้อย!"
    redirect_to articles_path, alert: "เกิดข้อผิดพลาด"
    redirect_to root_path, flash: { custom_type: "Custom message" }

    # redirect back
    redirect_back fallback_location: root_path
    redirect_back_or_to root_path  # Rails 7.1+

    # redirect กับ status
    redirect_to articles_path, status: :moved_permanently  # 301
    redirect_to articles_path, status: :found              # 302 (default)
    redirect_to articles_path, status: :see_other          # 303
    redirect_to articles_path, status: :temporary_redirect  # 307

    # redirect กับ allow_other_host (Rails 7.1)
    redirect_to "https://external.com", allow_other_host: true
  end
end
```

### ความแตกต่างระหว่าง render และ redirect_to

```ruby
# render: ไม่ส่ง HTTP redirect
# - แสดง template โดยตรง
# - instance variables พร้อมใช้ใน template
# - URL ใน browser ไม่เปลี่ยน
# - ใช้เมื่อ: form errors, หน้าพิเศษ

# redirect_to: ส่ง HTTP 302 response
# - browser จะ load URL ใหม่
# - instance variables หายไป (ต้องใช้ flash)
# - URL ใน browser เปลี่ยน
# - ใช้เมื่อ: หลัง create/update/destroy สำเร็จ

# PRG Pattern (Post-Redirect-Get)
def create
  @article = Article.create!(article_params)
  redirect_to @article  # ✅ PRG Pattern
  # ไม่ render โดยตรง เพราะถ้า user refresh จะ POST ซ้ำ
end
```

---

## ขั้นตอนที่ 736: Flash Messages

### การใช้ Flash

```ruby
# ใน Controller
class ArticlesController < ApplicationController
  def create
    @article = Article.create!(article_params)

    # Flash กับ redirect
    redirect_to @article, notice: "บทความถูกสร้างเรียบร้อยแล้ว"
    redirect_to @article, alert: "มีข้อผิดพลาด"
    redirect_to @article, flash: {
      notice: "สำเร็จ",
      alert: "คำเตือน",
      info: "ข้อมูล",
      success: "สำเร็จ"
    }
  end

  def update
    if @article.update(article_params)
      flash[:notice] = "อัพเดทสำเร็จ"
      redirect_to @article
    else
      flash.now[:alert] = "เกิดข้อผิดพลาด"  # flash.now สำหรับ render (ไม่ใช่ redirect)
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @article.destroy
    flash[:notice] = "ลบบทความเรียบร้อยแล้ว"
    redirect_to articles_path
  end
end

# ใน View
# app/views/layouts/application.html.erb
<% flash.each do |type, message| %>
  <% css_class = case type.to_sym
                 when :notice, :success then "alert-success"
                 when :alert, :error    then "alert-danger"
                 when :warning          then "alert-warning"
                 when :info             then "alert-info"
                 else "alert-secondary"
                 end %>
  <div class="alert <%= css_class %> alert-dismissible fade show" role="alert">
    <%= message %>
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
<% end %>
```

### Flash กับ Turbo Streams (Rails 7)

```ruby
# app/views/articles/create.turbo_stream.erb
<%= turbo_stream.prepend "articles", partial: "articles/article", locals: { article: @article } %>
<%= turbo_stream.update "flash", partial: "shared/flash_message", locals: { message: "บทความถูกสร้างแล้ว", type: "success" } %>

# Controller:
def create
  @article = Article.create!(article_params)

  respond_to do |format|
    format.turbo_stream
    format.html { redirect_to @article, notice: "สร้างสำเร็จ" }
  end
end
```

---

## ขั้นตอนที่ 737: respond_to

### respond_to สำหรับ Multiple Formats

```ruby
class ArticlesController < ApplicationController
  def index
    @articles = Article.published.includes(:user)

    respond_to do |format|
      format.html  # render app/views/articles/index.html.erb
      format.json do
        render json: @articles.as_json(
          only: [:id, :title, :status, :created_at],
          include: {
            user: { only: [:id, :name, :email] }
          }
        )
      end
      format.atom do
        # RSS/Atom feed
        render layout: false
      end
      format.csv do
        send_data Article.to_csv(@articles),
          filename: "articles-#{Date.current}.csv",
          type: "text/csv"
      end
      format.pdf do
        pdf = ArticlePDF.new(@articles)
        send_data pdf.render,
          filename: "articles.pdf",
          type: "application/pdf"
      end
      format.xlsx do
        response.headers["Content-Disposition"] = 'attachment; filename="articles.xlsx"'
        render xlsx: "index"
      end
    end
  end

  def show
    @article = Article.find(params[:id])

    respond_to do |format|
      format.html
      format.json { render json: @article }
      format.pdf do
        pdf = ArticlePDF.new(@article)
        send_data pdf.render,
          filename: "#{@article.slug}.pdf",
          type: "application/pdf",
          disposition: "inline"  # inline = แสดงในหน้าต่างเบราว์เซอร์
          # disposition: "attachment" = download
      end
    end
  end
end
```

### respond_with (Rails API)

```ruby
# API Controller
class Api::V1::ArticlesController < Api::BaseController
  def index
    @articles = Article.published

    render json: {
      data: @articles.map { |a| serialize_article(a) },
      meta: {
        total: @articles.count,
        page: params[:page] || 1
      }
    }
  end

  def show
    @article = Article.find(params[:id])
    render json: { data: serialize_article(@article) }
  end

  def create
    @article = Article.new(article_params)
    @article.user = current_api_user

    if @article.save
      render json: { data: serialize_article(@article) }, status: :created
    else
      render json: {
        errors: @article.errors.as_json,
        message: "Validation failed"
      }, status: :unprocessable_entity
    end
  end

  private

  def serialize_article(article)
    {
      id: article.id,
      type: "article",
      attributes: {
        title: article.title,
        body: article.body,
        status: article.status,
        created_at: article.created_at.iso8601,
        updated_at: article.updated_at.iso8601
      },
      relationships: {
        user: {
          data: {
            id: article.user_id,
            type: "user"
          }
        }
      }
    }
  end
end
```

---

## ขั้นตอนที่ 738: Filters และ Callbacks

### Filter Chain

```ruby
class ApplicationController < ActionController::Base
  before_action :log_access
  before_action :authenticate_user!

  private

  def log_access
    Rails.logger.info "[#{Time.current}] #{request.method} #{request.path}"
  end
end

class ArticlesController < ApplicationController
  # เพิ่ม filter ใหม่
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  before_action :check_ownership, only: [:edit, :update, :destroy]

  # Filter order:
  # 1. ApplicationController#log_access
  # 2. ApplicationController#authenticate_user!
  # 3. ArticlesController#set_article (ถ้า action ตรง)
  # 4. ArticlesController#check_ownership (ถ้า action ตรง)
  # 5. Action itself (index/show/new/create/etc.)

  # Skip inherited filters
  skip_before_action :authenticate_user!, only: [:index, :show]

  # Prepend filter (รันก่อน filter chain ทั้งหมด)
  prepend_before_action :redirect_maintenance_mode
end
```

### Conditional Filters

```ruby
class ArticlesController < ApplicationController
  # ใช้ method name เป็น condition
  before_action :premium_required, if: :premium_content?
  before_action :notify_admin, unless: :admin_action?

  # ใช้ lambda เป็น condition
  before_action :check_ip, if: -> { request.remote_ip == "127.0.0.1" }
  before_action :redirect_old_browsers, unless: -> { request.user_agent =~ /Chrome/ }

  private

  def premium_content?
    action_name == "show" && @article&.premium?
  end

  def admin_action?
    current_user&.admin?
  end

  def premium_required
    unless current_user&.premium?
      redirect_to upgrade_path, alert: "บทความนี้สำหรับสมาชิก Premium เท่านั้น"
    end
  end
end
```

---

## ขั้นตอนที่ 739: Helper Methods ใน Controllers

### Controller Helper Methods

```ruby
class ApplicationController < ActionController::Base
  # ประกาศ helper_method เพื่อให้ใช้ได้ใน views
  helper_method :current_user, :user_signed_in?, :admin?

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  def user_signed_in?
    current_user.present?
  end

  def admin?
    current_user&.admin?
  end
end

# การใช้ใน Views:
# <% if user_signed_in? %>
#   <span>สวัสดี, <%= current_user.name %></span>
# <% end %>
#
# <% if admin? %>
#   <%= link_to "Admin Panel", admin_root_path %>
# <% end %>
```

### Utility Methods

```ruby
class ArticlesController < ApplicationController
  private

  # Pagination helper
  def paginate_scope(scope)
    scope.page(params[:page]).per(per_page)
  end

  def per_page
    [params[:per_page].to_i, 100].min.nonzero? || 20
  end

  # Sort helper
  def sort_column
    Article.column_names.include?(params[:sort]) ? params[:sort] : "created_at"
  end

  def sort_direction
    %w[asc desc].include?(params[:direction]) ? params[:direction] : "desc"
  end

  # Search helper
  def search_query
    params[:q].presence
  end

  # JSON response helper
  def render_success(data, message: nil, status: :ok)
    response_body = { success: true, data: data }
    response_body[:message] = message if message
    render json: response_body, status: status
  end

  def render_error(errors, message: "Validation failed", status: :unprocessable_entity)
    render json: {
      success: false,
      message: message,
      errors: errors
    }, status: status
  end
end
```

---

## ขั้นตอนที่ 740: Request และ Response Objects

### Request Object

```ruby
class ArticlesController < ApplicationController
  def show
    # Request information
    request.method          # "GET", "POST", etc.
    request.path            # "/articles/1"
    request.url             # "http://localhost:3000/articles/1"
    request.host            # "localhost"
    request.port            # 3000
    request.remote_ip       # "127.0.0.1"
    request.user_agent      # Browser string
    request.referer         # Previous URL
    request.xhr?            # Ajax request?
    request.format          # :html, :json, etc.
    request.content_type    # "application/json"
    request.headers["Authorization"]  # Custom headers

    # Parameters
    params[:id]             # URL params
    request.query_parameters # Query string params
    request.request_parameters # POST body params

    # Body
    request.body.read       # Raw request body

    # Cookies
    cookies[:session_token]
    request.cookies
  end
end
```

### Response Object

```ruby
class ArticlesController < ApplicationController
  def index
    @articles = Article.all

    # Set response headers
    response.headers["X-Custom-Header"] = "value"
    response.headers["Cache-Control"] = "no-cache, no-store, must-revalidate"
    response.headers["Last-Modified"] = Article.maximum(:updated_at).httpdate

    # Set cache headers
    expires_in 10.minutes, public: true

    # Set content type
    response.content_type = "application/json"

    # Status
    response.status = :ok  # หรือ 200

    # ETag caching
    fresh_when etag: @articles, last_modified: @articles.maximum(:updated_at)
  end
end
```

---

## ขั้นตอนที่ 741: Concerns สำหรับ Controllers

### การสร้าง Controller Concern

```ruby
# app/controllers/concerns/paginatable.rb
module Paginatable
  extend ActiveSupport::Concern

  included do
    before_action :set_pagination_params
  end

  private

  def set_pagination_params
    @page = params.fetch(:page, 1).to_i
    @per_page = [params.fetch(:per_page, 20).to_i, 100].min
  end

  def paginate(scope)
    scope.page(@page).per(@per_page)
  end

  def pagination_meta(collection)
    {
      current_page: collection.current_page,
      total_pages: collection.total_pages,
      total_count: collection.total_count,
      per_page: collection.limit_value
    }
  end
end
```

```ruby
# app/controllers/concerns/authenticatable.rb
module Authenticatable
  extend ActiveSupport::Concern

  included do
    before_action :authenticate_request
    helper_method :current_user
  end

  private

  def authenticate_request
    header = request.headers["Authorization"]
    token = header&.split(" ")&.last

    payload = JwtService.decode(token)
    @current_user_id = payload&.dig("user_id")

    unless @current_user_id
      render json: { error: "Unauthorized" }, status: :unauthorized
    end
  end

  def current_user
    @current_user ||= User.find_by(id: @current_user_id)
  end
end
```

```ruby
# app/controllers/concerns/rate_limitable.rb
module RateLimitable
  extend ActiveSupport::Concern

  included do
    before_action :check_rate_limit
  end

  private

  def check_rate_limit
    key = "rate_limit:#{request.remote_ip}:#{action_name}"
    count = Rails.cache.increment(key, 1, expires_in: 1.minute)

    if count > rate_limit_for(action_name)
      render json: { error: "Rate limit exceeded" }, status: :too_many_requests
    end
  end

  def rate_limit_for(action)
    limits = {
      "index" => 100,
      "show" => 100,
      "create" => 10,
      "update" => 20,
      "destroy" => 10
    }
    limits[action] || 60
  end
end
```

```ruby
# app/controllers/api/v1/articles_controller.rb
class Api::V1::ArticlesController < Api::BaseController
  include Paginatable
  include RateLimitable

  def index
    @articles = paginate(Article.published)

    render json: {
      data: @articles,
      meta: pagination_meta(@articles)
    }
  end
end
```

---

## ขั้นตอนที่ 742-755: Testing Controllers

### Controller Tests ด้วย RSpec

```ruby
# spec/requests/articles_spec.rb (Request specs แนะนำมากกว่า Controller specs)
require "rails_helper"

RSpec.describe "Articles", type: :request do
  let(:user) { create(:user) }
  let(:other_user) { create(:user) }
  let(:article) { create(:article, user: user) }

  describe "GET /articles" do
    before do
      create_list(:article, 5, status: "published")
    end

    it "returns success" do
      get articles_path
      expect(response).to have_http_status(:ok)
    end

    it "displays articles" do
      get articles_path
      expect(response.body).to include(article.title)
    end

    context "with pagination" do
      it "paginates results" do
        get articles_path, params: { page: 1, per_page: 2 }
        expect(response).to have_http_status(:ok)
      end
    end

    context "with search" do
      it "filters by search query" do
        target = create(:article, title: "Ruby on Rails Tips", status: "published")
        other = create(:article, title: "Python Tips", status: "published")

        get articles_path, params: { q: "Ruby" }
        expect(response.body).to include(target.title)
        expect(response.body).not_to include(other.title)
      end
    end
  end

  describe "GET /articles/:id" do
    context "when article exists" do
      let!(:published_article) { create(:article, status: "published") }

      it "returns success" do
        get article_path(published_article)
        expect(response).to have_http_status(:ok)
      end
    end

    context "when article does not exist" do
      it "returns 404" do
        get article_path(id: 999999)
        expect(response).to have_http_status(:not_found)
      end
    end
  end

  describe "POST /articles" do
    context "when authenticated" do
      before { sign_in user }

      context "with valid params" do
        let(:valid_params) do
          { article: { title: "New Article", body: "This is the article body content that is long enough", status: "draft" } }
        end

        it "creates article" do
          expect {
            post articles_path, params: valid_params
          }.to change(Article, :count).by(1)
        end

        it "redirects to article" do
          post articles_path, params: valid_params
          expect(response).to redirect_to(article_path(Article.last))
        end

        it "sets flash notice" do
          post articles_path, params: valid_params
          expect(flash[:notice]).to be_present
        end
      end

      context "with invalid params" do
        let(:invalid_params) do
          { article: { title: "", body: "" } }
        end

        it "does not create article" do
          expect {
            post articles_path, params: invalid_params
          }.not_to change(Article, :count)
        end

        it "renders new template" do
          post articles_path, params: invalid_params
          expect(response).to have_http_status(:unprocessable_entity)
          expect(response).to render_template(:new)
        end
      end
    end

    context "when not authenticated" do
      it "redirects to login" do
        post articles_path, params: { article: attributes_for(:article) }
        expect(response).to redirect_to(login_path)
      end
    end
  end

  describe "PATCH /articles/:id" do
    context "when authenticated as owner" do
      before { sign_in user }

      context "with valid params" do
        it "updates article" do
          patch article_path(article), params: {
            article: { title: "Updated Title" }
          }
          expect(article.reload.title).to eq("Updated Title")
        end
      end
    end

    context "when authenticated as other user" do
      before { sign_in other_user }

      it "redirects with error" do
        patch article_path(article), params: {
          article: { title: "Hacked Title" }
        }
        expect(response).to redirect_to(articles_path)
        expect(flash[:alert]).to be_present
      end
    end
  end

  describe "DELETE /articles/:id" do
    context "when authenticated as owner" do
      before { sign_in user }

      it "destroys article" do
        article  # Ensure article exists
        expect {
          delete article_path(article)
        }.to change(Article, :count).by(-1)
      end

      it "redirects to articles" do
        delete article_path(article)
        expect(response).to redirect_to(articles_path)
      end
    end
  end
end
```

### JSON API Tests

```ruby
# spec/requests/api/v1/articles_spec.rb
require "rails_helper"

RSpec.describe "Api::V1::Articles", type: :request do
  let(:user) { create(:user) }
  let(:headers) { { "Authorization" => "Bearer #{user.api_token}", "Content-Type" => "application/json" } }

  describe "GET /api/v1/articles" do
    before { create_list(:article, 3, status: "published") }

    it "returns articles" do
      get api_v1_articles_path, headers: headers
      expect(response).to have_http_status(:ok)

      json = JSON.parse(response.body)
      expect(json["data"].length).to eq(3)
    end

    it "includes pagination meta" do
      get api_v1_articles_path, headers: headers
      json = JSON.parse(response.body)
      expect(json["meta"]).to include("total", "page")
    end
  end

  describe "POST /api/v1/articles" do
    let(:valid_params) do
      {
        article: {
          title: "New API Article",
          body: "This is a test article body with enough content",
          status: "draft"
        }
      }.to_json
    end

    it "creates article" do
      expect {
        post api_v1_articles_path, params: valid_params, headers: headers
      }.to change(Article, :count).by(1)

      expect(response).to have_http_status(:created)
    end

    context "without authentication" do
      it "returns 401" do
        post api_v1_articles_path, params: valid_params
        expect(response).to have_http_status(:unauthorized)
      end
    end
  end
end
```

---

## แบบฝึกหัด Part 34 (ขั้นตอนที่ 731-755)

**ข้อ 1:** สร้าง ApplicationController ที่มี authentication และ error handling

```ruby
# คำตอบ:
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  helper_method :current_user, :user_signed_in?

  rescue_from ActiveRecord::RecordNotFound do |e|
    respond_to do |format|
      format.html { render "errors/not_found", status: :not_found }
      format.json { render json: { error: "Not found" }, status: :not_found }
    end
  end

  private

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end

  def user_signed_in?
    current_user.present?
  end

  def authenticate_user!
    redirect_to login_path unless user_signed_in?
  end
end
```

**ข้อ 2:** เขียน Strong Parameters สำหรับ User model ที่มี nested profile

```ruby
# คำตอบ:
def user_params
  params.require(:user).permit(
    :name,
    :email,
    :password,
    :password_confirmation,
    :avatar,
    profile_attributes: [
      :id,
      :bio,
      :website,
      :github_username,
      :twitter_username,
      :location,
      :_destroy
    ]
  )
end
```

**ข้อ 3:** เขียน before_action ที่ตรวจสอบว่า user เป็นเจ้าของ resource

```ruby
# คำตอบ:
class ArticlesController < ApplicationController
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  before_action :ensure_owner!, only: [:edit, :update, :destroy]

  private

  def set_article
    @article = Article.find(params[:id])
  end

  def ensure_owner!
    unless @article.user == current_user || current_user.admin?
      respond_to do |format|
        format.html { redirect_to articles_path, alert: "ไม่มีสิทธิ์เข้าถึง" }
        format.json { render json: { error: "Forbidden" }, status: :forbidden }
      end
    end
  end
end
```

**ข้อ 4:** เขียน respond_to ที่รองรับ HTML, JSON, และ CSV

```ruby
# คำตอบ:
def index
  @products = Product.active.order(:name)

  respond_to do |format|
    format.html
    format.json { render json: @products }
    format.csv do
      csv_data = CSV.generate(headers: true) do |csv|
        csv << ["ID", "Name", "Price", "Stock"]
        @products.each do |p|
          csv << [p.id, p.name, p.price, p.stock]
        end
      end
      send_data csv_data,
        filename: "products-#{Date.current}.csv",
        type: "text/csv"
    end
  end
end
```

**ข้อ 5:** สร้าง flash message ที่รองรับหลาย types

```erb
<%# app/views/shared/_flash.html.erb %>
<% flash.each do |type, message| %>
  <% css_class = { notice: "success", alert: "danger", warning: "warning", info: "info" }[type.to_sym] || "secondary" %>
  <div class="alert alert-<%= css_class %> alert-dismissible">
    <%= message %>
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
<% end %>
```

**ข้อ 6:** เขียน around_action สำหรับ measure execution time

```ruby
# คำตอบ:
around_action :measure_execution do
  start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
  yield
  elapsed = (Process.clock_gettime(Process::CLOCK_MONOTONIC) - start) * 1000
  Rails.logger.info("[Performance] #{controller_name}##{action_name}: #{elapsed.round(2)}ms")
  response.headers["X-Runtime-Ms"] = elapsed.round(2).to_s
end
```

**ข้อ 7:** สร้าง Controller Concern สำหรับ sorting

```ruby
# คำตอบ:
# app/controllers/concerns/sortable.rb
module Sortable
  extend ActiveSupport::Concern

  ALLOWED_SORT_COLUMNS = %w[created_at updated_at name title].freeze
  ALLOWED_SORT_DIRECTIONS = %w[asc desc].freeze

  private

  def sort_column(allowed: ALLOWED_SORT_COLUMNS, default: "created_at")
    allowed.include?(params[:sort]) ? params[:sort] : default
  end

  def sort_direction(default: "desc")
    ALLOWED_SORT_DIRECTIONS.include?(params[:dir]) ? params[:dir] : default
  end

  def sorted(scope, allowed: ALLOWED_SORT_COLUMNS, default_col: "created_at", default_dir: "desc")
    scope.order(sort_column(allowed: allowed, default: default_col) => sort_direction(default: default_dir))
  end
end

# ใช้:
class ArticlesController < ApplicationController
  include Sortable

  def index
    @articles = sorted(Article.published, allowed: %w[title created_at views_count])
  end
end
```

**ข้อ 8:** เขียน request spec สำหรับ CRUD actions

```ruby
# คำตอบ:
RSpec.describe "Products", type: :request do
  let(:user) { create(:user) }
  let(:product) { create(:product, user: user) }

  before { sign_in user }

  it "creates a product" do
    expect {
      post products_path, params: {
        product: { name: "Test", price: 100, stock: 10 }
      }
    }.to change(Product, :count).by(1)
    expect(response).to redirect_to(product_path(Product.last))
  end

  it "updates a product" do
    patch product_path(product), params: {
      product: { name: "Updated Name" }
    }
    expect(product.reload.name).to eq("Updated Name")
  end

  it "destroys a product" do
    product
    expect {
      delete product_path(product)
    }.to change(Product, :count).by(-1)
  end
end
```

**ข้อ 9:** สร้าง API controller ที่ return JSON responses แบบสม่ำเสมอ

```ruby
# คำตอบ:
class Api::BaseController < ActionController::API
  before_action :authenticate!

  rescue_from ActiveRecord::RecordNotFound do
    render_error("Not found", status: :not_found)
  end

  rescue_from ActiveRecord::RecordInvalid do |e|
    render_error("Validation failed", errors: e.record.errors.full_messages)
  end

  private

  def render_success(data, message: nil, status: :ok, meta: nil)
    response = { success: true, data: data }
    response[:message] = message if message
    response[:meta] = meta if meta
    render json: response, status: status
  end

  def render_error(message, errors: nil, status: :unprocessable_entity)
    response = { success: false, message: message }
    response[:errors] = errors if errors
    render json: response, status: status
  end

  def authenticate!
    token = request.headers["Authorization"]&.split(" ")&.last
    @current_user = User.find_by(api_token: token)
    render json: { error: "Unauthorized" }, status: :unauthorized unless @current_user
  end
end
```

**ข้อ 10:** สร้าง Dashboard controller ที่ใช้ multiple models

```ruby
# คำตอบ:
class DashboardController < ApplicationController
  before_action :authenticate_user!

  def index
    @stats = {
      total_articles: current_user.articles.count,
      published_articles: current_user.articles.published.count,
      draft_articles: current_user.articles.drafts.count,
      total_views: current_user.articles.sum(:views_count),
      total_comments: Comment.where(article: current_user.articles).count
    }

    @recent_articles = current_user.articles
                                   .includes(:comments)
                                   .order(updated_at: :desc)
                                   .limit(5)

    @popular_articles = current_user.articles
                                    .published
                                    .order(views_count: :desc)
                                    .limit(5)

    @recent_comments = Comment.where(article: current_user.articles)
                               .includes(:user, :article)
                               .order(created_at: :desc)
                               .limit(10)
  end
end
```

**ข้อ 11-25:** ดูใน exercises ขยาย

```ruby
# ข้อ 11: สร้าง controller สำหรับ file download
def download
  @document = Document.find(params[:id])
  authorize! :download, @document

  send_file @document.file_path,
    filename: @document.original_filename,
    type: @document.content_type,
    disposition: :attachment
end

# ข้อ 12: สร้าง search controller
class SearchController < ApplicationController
  skip_before_action :authenticate_user!

  def index
    @query = params[:q]
    return unless @query.present?

    @results = {
      articles: Article.search(@query).published.limit(10),
      users: User.search(@query).limit(5),
      tags: Tag.search(@query).limit(10)
    }
  end
end

# ข้อ 13: สร้าง import controller
class ImportsController < ApplicationController
  def create
    file = params[:file]
    result = ImportService.new(file).call

    if result.success?
      redirect_to root_path, notice: "นำเข้า #{result.count} รายการสำเร็จ"
    else
      redirect_to root_path, alert: "เกิดข้อผิดพลาด: #{result.errors.join(', ')}"
    end
  end
end

# ข้อ 14: สร้าง webhook controller
class WebhooksController < ApplicationController
  skip_before_action :authenticate_user!
  skip_before_action :verify_authenticity_token

  def stripe
    payload = request.body.read
    sig_header = request.env["HTTP_STRIPE_SIGNATURE"]

    begin
      event = Stripe::Webhook.construct_event(
        payload, sig_header, Rails.application.credentials.dig(:stripe, :webhook_secret)
      )
    rescue JSON::ParserError, Stripe::SignatureVerificationError => e
      render json: { error: e.message }, status: :bad_request
      return
    end

    handle_stripe_event(event)
    render json: { received: true }
  end

  private

  def handle_stripe_event(event)
    case event["type"]
    when "payment_intent.succeeded"
      PaymentSuccessService.new(event["data"]["object"]).call
    when "payment_intent.payment_failed"
      PaymentFailedService.new(event["data"]["object"]).call
    end
  end
end

# ข้อ 15-25: additional patterns
# (Omitted for brevity - patterns shown above cover key concepts)
```

---

## สรุป Part 34

ในบทนี้เราได้เรียนรู้:

1. **ApplicationController** - Base class สำหรับ controllers ทั้งหมด
2. **CRUD Actions** - index, show, new, create, edit, update, destroy
3. **Strong Parameters** - ป้องกัน mass assignment vulnerability
4. **Callbacks** - before_action, after_action, around_action
5. **render vs redirect_to** - ความแตกต่างและเมื่อไหร่ใช้อะไร
6. **Flash Messages** - แจ้งเตือน user หลัง action
7. **respond_to** - รองรับหลาย formats (HTML, JSON, CSV)
8. **Controller Concerns** - แชร์ behavior ระหว่าง controllers
9. **Testing** - Request specs ด้วย RSpec

Controllers เป็น "conductor" ของ MVC ที่ประสานงานทุกส่วน เมื่อเข้าใจดีจะทำให้เขียน Rails app ได้อย่างมีประสิทธิภาพ

---

*ต่อไป: Part 35 - Views and ERB (ขั้นตอนที่ 756-780)*
