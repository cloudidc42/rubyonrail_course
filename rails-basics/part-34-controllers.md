# ตอนที่ 34: Controllers (Steps 731-755)

## บทนำ

Controllers เป็นส่วนที่เชื่อมระหว่าง Models (data) และ Views (presentation) ใน MVC pattern Controller รับ HTTP request, ทำงานกับ Model และส่งข้อมูลไปยัง View หรือส่ง response กลับ

---

## Step 731: ApplicationController

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # ทุก controller สืบทอดจาก ApplicationController
  
  # CSRF protection (เปิดโดย default)
  protect_from_forgery with: :exception
  
  # Helper methods ที่ใช้ได้ทุก controller
  helper_method :current_user, :user_signed_in?
  
  before_action :set_locale
  before_action :authenticate_user!
  
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
  
  def set_locale
    I18n.locale = params[:locale] || :th
  end
end
```

### API Controller

```ruby
# API-only ApplicationController
class ApplicationController < ActionController::API
  # ไม่มี CSRF, cookies, views
  
  before_action :authenticate_token!
  
  private
  
  def authenticate_token!
    token = request.headers["Authorization"]&.split(" ")&.last
    @current_user = User.find_by_auth_token(token)
    
    unless @current_user
      render json: { error: "Unauthorized" }, status: :unauthorized
    end
  end
  
  def current_user
    @current_user
  end
end
```

---

## Step 732: 7 RESTful Actions

### Index - แสดงรายการ

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.all
    # เมื่อใช้ pagination
    @posts = Post.order(created_at: :desc).page(params[:page]).per(10)
    
    # ด้วย filtering
    @posts = Post.published
    @posts = @posts.where(category_id: params[:category_id]) if params[:category_id]
    @posts = @posts.search(params[:q]) if params[:q]
    
    respond_to do |format|
      format.html
      format.json { render json: @posts }
    end
  end
```

### Show - แสดง record เฉพาะ

```ruby
  def show
    @post = Post.find(params[:id])
    # find จะ raise ActiveRecord::RecordNotFound ถ้าไม่เจอ
    
    # ตาม best practices
    @post = Post.find(params[:id])
    @comments = @post.comments.order(created_at: :asc)
    
    # Increment view count
    @post.increment!(:views_count)
  end
```

### New - แสดงฟอร์มสร้างใหม่

```ruby
  def new
    @post = Post.new
    # กำหนดค่า default
    @post = Post.new(status: "draft")
    # สำหรับ nested form
    @post.comments.build
  end
```

### Create - สร้าง record ใหม่

```ruby
  def create
    @post = Post.new(post_params)
    @post.user = current_user
    
    if @post.save
      redirect_to @post, notice: "สร้างบทความสำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end
```

### Edit - แสดงฟอร์มแก้ไข

```ruby
  def edit
    @post = Post.find(params[:id])
    # ตรวจสอบ authorization
    authorize @post  # ถ้าใช้ Pundit
  end
```

### Update - อัปเดต record

```ruby
  def update
    @post = Post.find(params[:id])
    
    if @post.update(post_params)
      redirect_to @post, notice: "อัปเดตสำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end
```

### Destroy - ลบ record

```ruby
  def destroy
    @post = Post.find(params[:id])
    @post.destroy
    redirect_to posts_path, notice: "ลบสำเร็จ", status: :see_other
  end
```

### Complete Controller

```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :authorize_post!, only: [:edit, :update, :destroy]
  
  def index
    @posts = Post.published.order(created_at: :desc)
    @posts = @posts.page(params[:page]).per(10) if defined?(Kaminari)
  end
  
  def show
    @comments = @post.comments.includes(:user).order(created_at: :asc)
  end
  
  def new
    @post = Post.new
  end
  
  def create
    @post = current_user.posts.new(post_params)
    
    if @post.save
      redirect_to @post, notice: "สร้างบทความสำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  def edit
  end
  
  def update
    if @post.update(post_params)
      redirect_to @post, notice: "อัปเดตสำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end
  
  def destroy
    @post.destroy
    redirect_to posts_path, notice: "ลบบทความสำเร็จ", status: :see_other
  end
  
  private
  
  def set_post
    @post = Post.find(params[:id])
  rescue ActiveRecord::RecordNotFound
    redirect_to posts_path, alert: "ไม่พบบทความ"
  end
  
  def authorize_post!
    unless @post.user == current_user || current_user.admin?
      redirect_to @post, alert: "ไม่มีสิทธิ์"
    end
  end
  
  def post_params
    params.require(:post).permit(:title, :content, :status, :category_id,
                                  :published_at, tag_ids: [])
  end
end
```

---

## Step 733: Strong Parameters

Strong Parameters ป้องกัน mass assignment vulnerability

### require และ permit

```ruby
# require - กำหนด root key ที่ต้องมี
# permit - กำหนด attributes ที่อนุญาต
def user_params
  params.require(:user).permit(:name, :email, :password, :password_confirmation)
end

# ถ้าไม่มี :user key จะ raise ActionController::ParameterMissing
# ถ้ามี attribute ที่ไม่ได้ permit จะถูกกรองออก (ไม่ raise error)
```

### ตัวอย่างต่างๆ

```ruby
# Simple attributes
def post_params
  params.require(:post).permit(:title, :content, :published)
end

# Array values (checkboxes)
def post_params
  params.require(:post).permit(:title, tag_ids: [])
end

# Nested attributes (accepts_nested_attributes_for)
def user_params
  params.require(:user).permit(
    :name,
    :email,
    addresses_attributes: [:id, :street, :city, :_destroy]
  )
end

# Hash values
def settings_params
  params.require(:settings).permit(
    preferences: [:theme, :language, :notifications]
  )
end

# Permit all (ใช้ด้วยความระวัง - dev/test เท่านั้น)
params.require(:post).permit!  # อนุญาตทุก attribute

# Optional require
def search_params
  params.permit(:q, :category, :sort, :page)  # ไม่มี require
end
```

### Handling Missing Parameters

```ruby
def create
  # ถ้า params[:post] ไม่มี จะ raise ActionController::ParameterMissing
  @post = Post.new(post_params)
rescue ActionController::ParameterMissing => e
  render json: { error: e.message }, status: :bad_request
end

# หรือ rescue ใน ApplicationController
class ApplicationController < ActionController::Base
  rescue_from ActionController::ParameterMissing, with: :handle_bad_request
  
  private
  
  def handle_bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end
end
```

---

## Step 734: before_action / after_action / around_action

### before_action

```ruby
class PostsController < ApplicationController
  # รันก่อนทุก action
  before_action :authenticate_user!
  
  # รันก่อนเฉพาะ actions ที่ระบุ
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  
  # รันก่อนทุก action ยกเว้นที่ระบุ
  before_action :log_action, except: [:index]
  
  def index; end
  def show; end
  
  private
  
  def authenticate_user!
    redirect_to login_path unless current_user
  end
  
  def set_post
    @post = Post.find(params[:id])
  end
  
  def log_action
    Rails.logger.info "User #{current_user&.id} accessed #{action_name}"
  end
end
```

### after_action

```ruby
class PostsController < ApplicationController
  after_action :log_access, only: [:show]
  after_action :update_stats
  
  def show
    @post = Post.find(params[:id])
  end
  
  private
  
  def log_access
    AccessLog.create!(
      user: current_user,
      resource: @post,
      action: "view"
    )
  end
  
  def update_stats
    # รันหลังทุก action
    Stats.increment("request_count")
  end
end
```

### around_action

```ruby
class PostsController < ApplicationController
  around_action :time_action
  around_action :catch_errors
  
  def index
    @posts = Post.all
  end
  
  private
  
  def time_action
    start = Time.current
    yield  # รัน action จริงๆ
    elapsed = Time.current - start
    Rails.logger.info "#{action_name} took #{elapsed.round(3)}s"
  end
  
  def catch_errors
    yield
  rescue ActiveRecord::RecordNotFound
    render json: { error: "Not found" }, status: :not_found
  rescue StandardError => e
    Sentry.capture_exception(e)
    render json: { error: "Server error" }, status: :internal_server_error
  end
end
```

### Filter Ordering

```ruby
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
end

class PostsController < ApplicationController
  before_action :set_post, only: [:show]
  
  # ลำดับการรัน:
  # 1. authenticate_user! (จาก ApplicationController)
  # 2. set_post (จาก PostsController)
  # 3. action จริง (show, index, etc.)
end
```

---

## Step 735: skip_before_action

```ruby
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  before_action :check_maintenance_mode
end

class SessionsController < ApplicationController
  # ข้าม authentication สำหรับ login หน้า
  skip_before_action :authenticate_user!, only: [:new, :create]
  
  def new
    # หน้า login - ไม่ต้อง authenticate
  end
  
  def create
    # สร้าง session - ไม่ต้อง authenticate
  end
  
  def destroy
    # Logout - ต้อง authenticate (ไม่ skip)
  end
end

class PublicController < ApplicationController
  skip_before_action :authenticate_user!
  skip_before_action :check_maintenance_mode
  
  def index
    # Public page
  end
end
```

---

## Step 736: render vs redirect_to

### render

```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    
    if @post.save
      redirect_to @post
    else
      # render แสดง view โดยไม่ทำ HTTP request ใหม่
      render :new, status: :unprocessable_entity
    end
  end
  
  def show
    @post = Post.find(params[:id])
    
    # render template ต่างๆ
    render :show                              # default - แสดง views/posts/show.html.erb
    render "show"                             # เหมือนกัน
    render template: "posts/show"             # explicit template path
    render "posts/index"                      # template อื่น
    
    # render ด้วย layout ต่างๆ
    render :show, layout: false               # ไม่ใช้ layout
    render :show, layout: "special"           # ใช้ layout อื่น
    render layout: "mobile"                   # เปลี่ยน layout
    
    # render JSON
    render json: @post
    render json: @post, status: :ok
    render json: { error: "Not found" }, status: :not_found
    render json: @post, only: [:id, :title]
    render json: @post.to_json(include: :comments)
    
    # render text
    render plain: "Hello World"
    render html: "<h1>Hello</h1>".html_safe
    
    # render ด้วย status
    render :show, status: 200
    render :new, status: :unprocessable_entity  # 422
    render :show, status: :not_found            # 404
    render :show, status: :forbidden            # 403
    
    # render XML
    render xml: @post
    
    # render nothing
    head :ok
    head :no_content   # 204
    head :not_found    # 404
  end
end
```

### redirect_to

```ruby
class PostsController < ApplicationController
  def create
    @post = Post.create(post_params)
    
    # redirect สร้าง HTTP 302 response
    redirect_to posts_path                    # => GET /posts
    redirect_to @post                         # => GET /posts/:id
    redirect_to post_path(@post)              # เหมือนกัน
    
    # redirect ด้วย flash message
    redirect_to @post, notice: "สร้างสำเร็จ"
    redirect_to @post, alert: "มีข้อผิดพลาด"
    redirect_to @post, flash: { success: "OK", info: "Note" }
    
    # redirect ด้วย status code
    redirect_to posts_path, status: :moved_permanently  # 301
    redirect_to posts_path, status: 302                 # default
    redirect_to posts_path, status: :see_other          # 303
    
    # redirect ภายนอก
    redirect_to "https://google.com"
    redirect_to "https://google.com", allow_other_host: true
    
    # redirect กลับไปหน้าก่อนหน้า
    redirect_back(fallback_location: root_path)
    redirect_back_or_to root_path  # Rails 7+
  end
end
```

### ความแตกต่าง render vs redirect_to

```
render    - แสดง view โดยไม่เพิ่ม request count
           - ข้อมูลใน @variables ยังคงอยู่
           - URL ไม่เปลี่ยน
           - ใช้สำหรับ errors

redirect_to - สร้าง HTTP redirect response
            - Browser ส่ง request ใหม่
            - URL เปลี่ยน
            - ใช้หลัง create/update/destroy สำเร็จ
            - ป้องกัน double submit
```

---

## Step 737: Flash Messages

### Flash ปกติ

```ruby
# Controller
class PostsController < ApplicationController
  def create
    @post = Post.create(post_params)
    # flash[:notice] persist ข้าม request (1 request)
    flash[:notice] = "สร้างสำเร็จ"
    redirect_to @post
  end
  
  def destroy
    @post.destroy
    flash[:alert] = "ลบสำเร็จ"
    redirect_to posts_path
  end
end

# Shorthand
redirect_to @post, notice: "สร้างสำเร็จ"
redirect_to posts_path, alert: "ลบสำเร็จ"
```

### flash.now (สำหรับ render)

```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    
    if @post.save
      redirect_to @post, notice: "สร้างสำเร็จ"
    else
      # flash.now ใช้ใน request ปัจจุบัน (ไม่ persist)
      flash.now[:alert] = "กรุณาตรวจสอบข้อมูล"
      render :new, status: :unprocessable_entity
    end
  end
end
```

### Custom Flash Types

```ruby
# config/application.rb
class Application < Rails::Application
  config.action_dispatch.default_headers = {
    "X-Frame-Options" => "SAMEORIGIN"
  }
end

# ApplicationController
class ApplicationController < ActionController::Base
  add_flash_types :success, :info, :warning, :danger
end

# ใช้ใน controller
redirect_to @post, success: "บันทึกสำเร็จ"
redirect_to @post, warning: "โปรดระวัง"
redirect_to @post, danger: "มีข้อผิดพลาด"
```

### แสดง Flash ใน Views

```erb
<!-- app/views/layouts/application.html.erb -->
<% flash.each do |type, message| %>
  <div class="alert alert-<%= type %>" role="alert">
    <%= message %>
  </div>
<% end %>

<!-- Bootstrap classes -->
<% flash.each do |type, message| %>
  <% css_class = { notice: "success", alert: "danger", warning: "warning" }[type.to_sym] || "info" %>
  <div class="alert alert-<%= css_class %> alert-dismissible fade show" role="alert">
    <%= message %>
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
  </div>
<% end %>
```

---

## Step 738: respond_to

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
    
    respond_to do |format|
      format.html   # renders views/posts/index.html.erb
      format.json { render json: @posts }
      format.xml  { render xml: @posts }
      format.csv  do
        send_data @posts.to_csv, filename: "posts.csv",
                                  type: "text/csv"
      end
    end
  end
  
  def show
    @post = Post.find(params[:id])
    
    respond_to do |format|
      format.html
      format.json { render json: @post, include: :comments }
      format.pdf  do
        pdf = PostPdfGenerator.new(@post).generate
        send_data pdf, filename: "post-#{@post.id}.pdf",
                       type: "application/pdf",
                       disposition: "inline"
      end
    end
  end
  
  def create
    @post = Post.new(post_params)
    
    respond_to do |format|
      if @post.save
        format.html { redirect_to @post, notice: "สร้างสำเร็จ" }
        format.json { render json: @post, status: :created }
        format.turbo_stream  # Rails 7 Turbo
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @post.errors, status: :unprocessable_entity }
      end
    end
  end
end
```

### Turbo Stream response (Rails 7)

```ruby
def create
  @comment = @post.comments.new(comment_params)
  
  respond_to do |format|
    if @comment.save
      format.turbo_stream  # renders views/comments/create.turbo_stream.erb
      format.html { redirect_to @post }
    else
      format.html { render :new, status: :unprocessable_entity }
    end
  end
end
```

```erb
<!-- views/comments/create.turbo_stream.erb -->
<%= turbo_stream.append "comments" do %>
  <%= render @comment %>
<% end %>
```

---

## Step 739: params Hash

```ruby
class PostsController < ApplicationController
  def index
    # URL: /posts?page=2&q=ruby&sort=created_at
    params[:page]     # => "2" (string)
    params[:q]        # => "ruby"
    params[:sort]     # => "created_at"
    
    # params.to_h - convert to regular hash
    params.to_h
    
    # ตรวจสอบ
    params[:page].present?
    params.key?(:page)
  end
  
  def show
    # URL: /posts/1
    params[:id]       # => "1" (string)
    params[:id].to_i  # => 1 (integer)
    
    # Route parameters
    params[:controller]  # => "posts"
    params[:action]      # => "show"
    params[:format]      # => nil หรือ "json"
  end
  
  def create
    # Form data หรือ JSON body
    params[:post]         # => ActionController::Parameters
    params[:post][:title] # => "My Post"
    
    # Nested params
    params[:user][:address][:city]
  end
  
  def search
    # Array params
    # URL: /search?tags[]=ruby&tags[]=rails
    params[:tags]     # => ["ruby", "rails"]
    
    # Hash params
    # URL: /search?filters[status]=published&filters[year]=2024
    params[:filters][:status]  # => "published"
    params[:filters][:year]    # => "2024"
  end
end
```

### ประเภท params

```ruby
# Query parameters (URL)
# GET /posts?page=1&q=ruby
params[:page]  # => "1"

# Path parameters
# GET /posts/1
params[:id]    # => "1"

# Form parameters (POST body)
# POST /posts { post: { title: "Hello" } }
params[:post][:title]  # => "Hello"

# Multipart (file upload)
params[:avatar]  # => ActionDispatch::Http::UploadedFile
```

---

## Step 740: request Object

```ruby
class ApplicationController < ActionController::Base
  def debug
    # HTTP method
    request.method          # => "GET", "POST", etc.
    request.get?            # => true
    request.post?           # => false
    request.patch?          # => false
    request.delete?         # => false
    
    # URL info
    request.url             # => "http://example.com/posts?page=1"
    request.path            # => "/posts"
    request.query_string    # => "page=1"
    request.fullpath        # => "/posts?page=1"
    request.host            # => "example.com"
    request.port            # => 3000
    request.protocol        # => "http://"
    
    # Format
    request.format          # => Mime::Type
    request.format.html?    # => true
    request.format.json?    # => false
    request.xhr?            # => true ถ้า AJAX request
    request.format.turbo_stream?  # Rails 7
    
    # IP
    request.ip              # => "127.0.0.1"
    request.remote_ip       # => real IP (through proxies)
    
    # Headers
    request.headers["User-Agent"]
    request.headers["Authorization"]
    request.headers["Content-Type"]
    request.headers["Accept"]
    
    # Body
    request.body.read       # raw body
    
    # Environment
    request.env             # Rack environment hash
    
    # SSL
    request.ssl?            # => false in development
    
    # Referrer
    request.referrer        # => URL ที่ user มาจาก
    request.referer         # alias
  end
end
```

### ตัวอย่างการใช้งาน

```ruby
class ApplicationController < ActionController::Base
  before_action :set_current_request_details
  
  private
  
  def set_current_request_details
    Current.ip = request.remote_ip
    Current.user_agent = request.user_agent
    Current.request_id = request.request_id
  end
  
  def mobile_device?
    request.user_agent =~ /Mobile|Android|iPhone|iPad/i
  end
  
  def api_request?
    request.format.json? || request.headers["Accept"] == "application/json"
  end
end
```

---

## Step 741: response Object

```ruby
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    
    # Set headers
    response.headers["X-Post-Id"] = @post.id.to_s
    response.headers["Cache-Control"] = "public, max-age=3600"
    response.headers["Content-Type"] = "application/json"
    
    # Set status
    response.status = 200
    response.status = :ok
    
    # Set body
    response.body = "Hello World"
    
    # Set cookies
    response.set_cookie("seen_post_#{@post.id}", value: "true", expires: 1.day.from_now)
    
    # ETag สำหรับ caching
    if stale?(@post)
      respond_to do |format|
        format.html
        format.json { render json: @post }
      end
    end
  end
  
  def download
    send_file Rails.root.join("storage/files/document.pdf"),
      filename: "document.pdf",
      type: "application/pdf",
      disposition: "attachment"  # force download
      # disposition: "inline"   # แสดงใน browser
  end
  
  def stream
    response.headers["Content-Type"] = "text/event-stream"
    response.headers["X-Accel-Buffering"] = "no"
    
    sse = SSE.new(response.stream, retry: 300, event: "update")
    
    begin
      10.times do |i|
        sse.write({ message: "Update #{i}" })
        sleep 1
      end
    ensure
      sse.close
    end
  end
end
```

---

## Step 742: Session ใน Controllers

```ruby
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])
    
    if user&.authenticate(params[:password])
      # เก็บข้อมูลใน session
      session[:user_id] = user.id
      session[:login_time] = Time.current.to_s
      session[:remember_token] = SecureRandom.hex(20)
      
      redirect_to root_path, notice: "เข้าสู่ระบบสำเร็จ"
    else
      flash.now[:alert] = "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
      render :new, status: :unprocessable_entity
    end
  end
  
  def destroy
    # ล้าง session ทั้งหมด
    reset_session
    # หรือล้างแบบ selective
    session.delete(:user_id)
    
    redirect_to login_path, notice: "ออกจากระบบสำเร็จ"
  end
end

class ApplicationController < ActionController::Base
  helper_method :current_user
  
  private
  
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  
  def authenticate_user!
    unless current_user
      session[:return_to] = request.fullpath
      redirect_to login_path, alert: "กรุณาเข้าสู่ระบบ"
    end
  end
  
  def after_login_redirect
    # redirect กลับไปหน้าที่ต้องการ
    redirect_to session.delete(:return_to) || root_path
  end
end
```

### Session Configuration

```ruby
# config/initializers/session_store.rb

# Cookie session (default - ข้อมูลอยู่ใน cookie)
Rails.application.config.session_store :cookie_store, 
  key: "_myapp_session",
  expire_after: 14.days,
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax

# Redis session
Rails.application.config.session_store :redis_store,
  servers: [{ host: "localhost", port: 6379, db: 0 }],
  expire_in: 90.minutes,
  key: "_myapp_session"
```

---

## Step 743: Cookies ใน Controllers

```ruby
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    
    # อ่าน cookie
    viewed_posts = JSON.parse(cookies[:viewed_posts] || "[]")
    
    # เขียน cookie
    viewed_posts << @post.id
    viewed_posts = viewed_posts.last(10)  # เก็บ 10 อัน
    cookies[:viewed_posts] = {
      value: viewed_posts.to_json,
      expires: 30.days.from_now,
      httponly: true
    }
    
    # Signed cookie (ป้องกัน tampering)
    cookies.signed[:user_id] = {
      value: current_user.id,
      expires: 30.days.from_now
    }
    
    # อ่าน signed cookie
    user_id = cookies.signed[:user_id]
    
    # Encrypted cookie (ป้องกันทั้ง tampering และ reading)
    cookies.encrypted[:preferences] = {
      value: { theme: "dark" }.to_json,
      expires: 1.year.from_now
    }
    preferences = JSON.parse(cookies.encrypted[:preferences] || "{}")
    
    # ลบ cookie
    cookies.delete(:viewed_posts)
    
    # Permanent cookie
    cookies.permanent[:theme] = "dark"
    cookies.permanent.signed[:remember_token] = remember_token
  end
end
```

---

## Step 744: Concerns (Shared Controller Code)

```ruby
# app/controllers/concerns/authenticatable.rb
module Authenticatable
  extend ActiveSupport::Concern
  
  included do
    before_action :authenticate_user!
    helper_method :current_user, :user_signed_in?
  end
  
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  
  def user_signed_in?
    current_user.present?
  end
  
  private
  
  def authenticate_user!
    redirect_to login_path, alert: "กรุณาเข้าสู่ระบบ" unless user_signed_in?
  end
end

# app/controllers/concerns/paginatable.rb
module Paginatable
  extend ActiveSupport::Concern
  
  private
  
  def paginate(scope)
    scope.page(params[:page]).per(per_page)
  end
  
  def per_page
    [(params[:per_page] || 25).to_i, 100].min
  end
end

# app/controllers/concerns/sortable.rb
module Sortable
  extend ActiveSupport::Concern
  
  private
  
  def sort_column(model)
    allowed_columns = model.column_names
    allowed_columns.include?(params[:sort]) ? params[:sort] : "created_at"
  end
  
  def sort_direction
    %w[asc desc].include?(params[:direction]) ? params[:direction] : "desc"
  end
end

# ใช้ concerns ใน controller
class PostsController < ApplicationController
  include Paginatable
  include Sortable
  
  def index
    @posts = paginate(
      Post.order(sort_column(Post) => sort_direction)
    )
  end
end

class ApplicationController < ActionController::Base
  include Authenticatable
end
```

---

## Step 745: Testing Controllers

### Minitest

```ruby
# test/controllers/posts_controller_test.rb
require "test_helper"

class PostsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:one)
    @post = posts(:one)
    sign_in @user  # หรือ session[:user_id] = @user.id
  end
  
  # Test index
  test "GET /posts returns success" do
    get posts_path
    assert_response :success
    assert_template :index
  end
  
  # Test show
  test "GET /posts/:id returns success" do
    get post_path(@post)
    assert_response :success
    assert_assigns(:post, @post)
  end
  
  # Test create
  test "POST /posts with valid params" do
    assert_difference("Post.count", 1) do
      post posts_path, params: {
        post: { title: "New Post", content: "Content" }
      }
    end
    assert_redirected_to post_path(Post.last)
    assert_equal "สร้างบทความสำเร็จ", flash[:notice]
  end
  
  test "POST /posts with invalid params" do
    assert_no_difference("Post.count") do
      post posts_path, params: {
        post: { title: "", content: "" }
      }
    end
    assert_response :unprocessable_entity
    assert_template :new
  end
  
  # Test update
  test "PATCH /posts/:id with valid params" do
    patch post_path(@post), params: {
      post: { title: "Updated Title" }
    }
    assert_redirected_to post_path(@post)
    @post.reload
    assert_equal "Updated Title", @post.title
  end
  
  # Test destroy
  test "DELETE /posts/:id" do
    assert_difference("Post.count", -1) do
      delete post_path(@post)
    end
    assert_redirected_to posts_path
  end
  
  # Test unauthorized
  test "redirect to login when not authenticated" do
    sign_out
    get posts_path
    assert_redirected_to login_path
  end
end
```

### RSpec

```ruby
# spec/requests/posts_spec.rb
require "rails_helper"

RSpec.describe "Posts", type: :request do
  let(:user) { create(:user) }
  let(:post) { create(:post, user: user) }
  
  before { sign_in user }
  
  describe "GET /posts" do
    it "returns success" do
      get posts_path
      expect(response).to have_http_status(:success)
    end
    
    it "shows all posts" do
      create_list(:post, 3, user: user)
      get posts_path
      expect(response.body).to include(post.title)
    end
  end
  
  describe "POST /posts" do
    context "with valid params" do
      let(:valid_params) { { post: { title: "New Post", content: "Content" } } }
      
      it "creates a post" do
        expect {
          post posts_path, params: valid_params
        }.to change(Post, :count).by(1)
      end
      
      it "redirects to the new post" do
        post posts_path, params: valid_params
        expect(response).to redirect_to(Post.last)
      end
    end
    
    context "with invalid params" do
      let(:invalid_params) { { post: { title: "", content: "" } } }
      
      it "does not create a post" do
        expect {
          post posts_path, params: invalid_params
        }.not_to change(Post, :count)
      end
      
      it "returns unprocessable entity" do
        post posts_path, params: invalid_params
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end
  
  describe "DELETE /posts/:id" do
    it "destroys the post" do
      post_to_delete = create(:post, user: user)
      expect {
        delete post_path(post_to_delete)
      }.to change(Post, :count).by(-1)
    end
  end
end
```

---

## แบบฝึกหัด (Steps 746-755)

### แบบฝึกหัดที่ 1
สร้าง ApplicationController ที่มี `current_user` helper และ `authenticate_user!` before_action

**เฉลย:**
```ruby
class ApplicationController < ActionController::Base
  helper_method :current_user
  
  private
  
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  
  def authenticate_user!
    redirect_to login_path unless current_user
  end
end
```

### แบบฝึกหัดที่ 2
สร้าง `PostsController` ที่มีทุก 7 RESTful actions พร้อม Strong Parameters

**เฉลย:**
```ruby
class PostsController < ApplicationController
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  
  def index
    @posts = Post.all
  end
  
  def show; end
  
  def new
    @post = Post.new
  end
  
  def create
    @post = Post.new(post_params)
    if @post.save
      redirect_to @post, notice: "สร้างสำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  def edit; end
  
  def update
    if @post.update(post_params)
      redirect_to @post, notice: "อัปเดตสำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end
  
  def destroy
    @post.destroy
    redirect_to posts_path, notice: "ลบสำเร็จ", status: :see_other
  end
  
  private
  
  def set_post
    @post = Post.find(params[:id])
  end
  
  def post_params
    params.require(:post).permit(:title, :content)
  end
end
```

### แบบฝึกหัดที่ 3
เพิ่ม before_action ที่ตรวจสอบว่า user เป็น admin เท่านั้นที่เข้าถึง destroy ได้

**เฉลย:**
```ruby
before_action :require_admin!, only: [:destroy]

private

def require_admin!
  redirect_to root_path, alert: "ไม่มีสิทธิ์" unless current_user&.admin?
end
```

### แบบฝึกหัดที่ 4
สร้าง controller ที่ respond ทั้ง HTML และ JSON

**เฉลย:**
```ruby
def index
  @posts = Post.all
  
  respond_to do |format|
    format.html
    format.json { render json: @posts }
  end
end
```

### แบบฝึกหัดที่ 5
ใช้ flash.now เพื่อแสดง error message เมื่อ create ไม่สำเร็จ

**เฉลย:**
```ruby
def create
  @post = Post.new(post_params)
  if @post.save
    redirect_to @post, notice: "สร้างสำเร็จ"
  else
    flash.now[:alert] = "กรุณาตรวจสอบข้อมูล"
    render :new, status: :unprocessable_entity
  end
end
```

### แบบฝึกหัดที่ 6
สร้าง concern `Trackable` ที่ log action ทุกครั้งที่มี request

**เฉลย:**
```ruby
# app/controllers/concerns/trackable.rb
module Trackable
  extend ActiveSupport::Concern
  
  included do
    after_action :log_action
  end
  
  private
  
  def log_action
    Rails.logger.info "[#{Time.current}] #{controller_name}##{action_name} - User: #{current_user&.id}"
  end
end
```

### แบบฝึกหัดที่ 7
implement `skip_before_action` ให้ SessionsController ข้าม authenticate

**เฉลย:**
```ruby
class SessionsController < ApplicationController
  skip_before_action :authenticate_user!, only: [:new, :create]
  
  def new; end
  
  def create
    user = User.find_by(email: params[:email])
    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to root_path
    else
      flash.now[:alert] = "ข้อมูลไม่ถูกต้อง"
      render :new, status: :unprocessable_entity
    end
  end
  
  def destroy
    reset_session
    redirect_to login_path
  end
end
```

### แบบฝึกหัดที่ 8
สร้าง around_action ที่วัดเวลาการรัน action

**เฉลย:**
```ruby
around_action :measure_performance

private

def measure_performance
  start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
  yield
  elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
  Rails.logger.info "#{controller_name}##{action_name} took #{(elapsed * 1000).round(2)}ms"
end
```

### แบบฝึกหัดที่ 9
ใช้ Strong Parameters สำหรับ User ที่มี nested address attributes

**เฉลย:**
```ruby
def user_params
  params.require(:user).permit(
    :name,
    :email,
    :password,
    :password_confirmation,
    address_attributes: [:id, :street, :city, :province, :postal_code, :_destroy]
  )
end
```

### แบบฝึกหัดที่ 10
สร้าง before_action ที่ set ค่า @post และ handle RecordNotFound error

**เฉลย:**
```ruby
before_action :set_post, only: [:show, :edit, :update, :destroy]

private

def set_post
  @post = Post.find(params[:id])
rescue ActiveRecord::RecordNotFound
  redirect_to posts_path, alert: "ไม่พบบทความ"
end
```

### แบบฝึกหัดที่ 11
สร้าง action ที่ส่ง redirect_back พร้อม fallback

**เฉลย:**
```ruby
def vote
  @post = Post.find(params[:id])
  @post.votes.create!(user: current_user)
  redirect_back(fallback_location: @post, notice: "โหวตสำเร็จ")
end
```

### แบบฝึกหัดที่ 12
implement cookie-based "remember me" feature

**เฉลย:**
```ruby
def create
  user = User.find_by(email: params[:email])
  
  if user&.authenticate(params[:password])
    session[:user_id] = user.id
    
    if params[:remember_me]
      token = SecureRandom.urlsafe_base64
      user.update!(remember_token: token)
      cookies.permanent.signed[:remember_token] = token
    end
    
    redirect_to root_path
  end
end

# ใน ApplicationController
def current_user
  @current_user ||= 
    if session[:user_id]
      User.find_by(id: session[:user_id])
    elsif cookies.signed[:remember_token]
      user = User.find_by(remember_token: cookies.signed[:remember_token])
      session[:user_id] = user.id if user
      user
    end
end
```

### แบบฝึกหัดที่ 13
เขียน request spec สำหรับ PostsController ที่ test index action

**เฉลย:**
```ruby
RSpec.describe "Posts", type: :request do
  describe "GET /posts" do
    before { create_list(:post, 5) }
    
    it "returns success" do
      get posts_path
      expect(response).to have_http_status(:ok)
    end
    
    it "returns posts in the response" do
      get posts_path
      expect(response.body).to include(Post.first.title)
    end
    
    context "as JSON" do
      it "returns JSON" do
        get posts_path, headers: { "Accept" => "application/json" }
        expect(response.content_type).to include("application/json")
      end
    end
  end
end
```

### แบบฝึกหัดที่ 14
สร้าง action ที่ download CSV file

**เฉลย:**
```ruby
def export
  @posts = Post.all
  
  respond_to do |format|
    format.csv do
      csv_data = CSV.generate(headers: true) do |csv|
        csv << ["ID", "Title", "Created At"]
        @posts.each do |post|
          csv << [post.id, post.title, post.created_at.strftime("%Y-%m-%d")]
        end
      end
      send_data csv_data, 
                filename: "posts-#{Date.today}.csv",
                type: "text/csv"
    end
  end
end
```

### แบบฝึกหัดที่ 15
implement pagination โดยใช้ Concern

**เฉลย:**
```ruby
# app/controllers/concerns/paginatable.rb
module Paginatable
  extend ActiveSupport::Concern
  
  private
  
  def paginate(scope)
    page = [params[:page].to_i, 1].max
    per_page = [[params[:per_page].to_i, 10].max, 100].min
    
    total = scope.count
    offset = (page - 1) * per_page
    
    {
      records: scope.limit(per_page).offset(offset),
      meta: {
        total: total,
        page: page,
        per_page: per_page,
        total_pages: (total.to_f / per_page).ceil
      }
    }
  end
end
```

### แบบฝึกหัดที่ 16
สร้าง after_action ที่ cache response headers

**เฉลย:**
```ruby
after_action :set_cache_headers, only: [:show, :index]

private

def set_cache_headers
  if @post&.published?
    response.headers["Cache-Control"] = "public, max-age=300"
    response.headers["Vary"] = "Accept"
  else
    response.headers["Cache-Control"] = "private, no-cache"
  end
end
```

### แบบฝึกหัดที่ 17
ใช้ request.xhr? เพื่อ respond แตกต่างกันระหว่าง AJAX และ regular request

**เฉลย:**
```ruby
def index
  @posts = Post.all
  
  if request.xhr?
    render partial: "posts", locals: { posts: @posts }
  else
    render :index
  end
end
```

### แบบฝึกหัดที่ 18
implement rate limiting ด้วย before_action

**เฉลย:**
```ruby
before_action :check_rate_limit, only: [:create]

private

def check_rate_limit
  key = "rate_limit:#{request.ip}:#{controller_name}"
  count = Rails.cache.increment(key, 1, expires_in: 1.hour)
  
  if count > 100
    render json: { error: "Rate limit exceeded" }, status: :too_many_requests
  end
end
```

### แบบฝึกหัดที่ 19
สร้าง controller action ที่ handle JSON API request

**เฉลย:**
```ruby
class Api::PostsController < ApplicationController
  before_action :authenticate_token!
  
  def index
    @posts = Post.all
    render json: {
      data: @posts.map { |post|
        {
          id: post.id,
          type: "post",
          attributes: {
            title: post.title,
            content: post.content,
            created_at: post.created_at.iso8601
          }
        }
      }
    }
  end
  
  private
  
  def authenticate_token!
    token = request.headers["Authorization"]&.remove("Bearer ")
    @current_user = User.find_by(api_token: token)
    render json: { error: "Unauthorized" }, status: 401 unless @current_user
  end
end
```

### แบบฝึกหัดที่ 20
implement การ handle RecordNotFound ใน ApplicationController

**เฉลย:**
```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :handle_not_found
  
  private
  
  def handle_not_found(exception)
    respond_to do |format|
      format.html do
        render "errors/not_found", status: :not_found
      end
      format.json do
        render json: { error: "Resource not found" }, status: :not_found
      end
    end
  end
end
```

### แบบฝึกหัดที่ 21
สร้าง `CommentsController` ที่ nested ใน `PostsController`

**เฉลย:**
```ruby
class CommentsController < ApplicationController
  before_action :set_post
  before_action :set_comment, only: [:destroy]
  
  def create
    @comment = @post.comments.new(comment_params)
    @comment.user = current_user
    
    if @comment.save
      redirect_to @post, notice: "เพิ่มความคิดเห็นสำเร็จ"
    else
      redirect_to @post, alert: "ไม่สามารถเพิ่มความคิดเห็นได้"
    end
  end
  
  def destroy
    @comment.destroy
    redirect_to @post, notice: "ลบความคิดเห็นสำเร็จ"
  end
  
  private
  
  def set_post
    @post = Post.find(params[:post_id])
  end
  
  def set_comment
    @comment = @post.comments.find(params[:id])
  end
  
  def comment_params
    params.require(:comment).permit(:content)
  end
end
```

### แบบฝึกหัดที่ 22
implement content negotiation ด้วย respond_to

**เฉลย:**
```ruby
def show
  @post = Post.find(params[:id])
  
  respond_to do |format|
    format.html
    format.json { render json: @post }
    format.xml  { render xml: @post }
    format.pdf  do
      pdf = WickedPdf.new.pdf_from_string(
        render_to_string("posts/show", layout: "pdf")
      )
      send_data pdf,
                filename: "post-#{@post.id}.pdf",
                type: "application/pdf"
    end
  end
end
```

### แบบฝึกหัดที่ 23
ใช้ session เพื่อ track user's recently viewed posts

**เฉลย:**
```ruby
def show
  @post = Post.find(params[:id])
  
  session[:viewed_posts] ||= []
  session[:viewed_posts] = ([params[:id]] + session[:viewed_posts]).uniq.first(5)
  
  @recently_viewed = Post.where(id: session[:viewed_posts] - [params[:id]])
end
```

### แบบฝึกหัดที่ 24
implement search action ที่ไม่ใช้ model (search across multiple)

**เฉลย:**
```ruby
def search
  query = params[:q].to_s.strip
  
  if query.present?
    @posts = Post.where("title ILIKE ?", "%#{query}%").limit(5)
    @users = User.where("name ILIKE ?", "%#{query}%").limit(5)
    @categories = Category.where("name ILIKE ?", "%#{query}%").limit(5)
  else
    @posts = @users = @categories = []
  end
  
  respond_to do |format|
    format.html
    format.json do
      render json: {
        posts: @posts,
        users: @users,
        categories: @categories
      }
    end
  end
end
```

### แบบฝึกหัดที่ 25
สร้าง Admin::PostsController ที่สืบทอดจาก Admin::ApplicationController

**เฉลย:**
```ruby
# app/controllers/admin/application_controller.rb
module Admin
  class ApplicationController < ::ApplicationController
    before_action :require_admin!
    layout "admin"
    
    private
    
    def require_admin!
      redirect_to root_path unless current_user&.admin?
    end
  end
end

# app/controllers/admin/posts_controller.rb
module Admin
  class PostsController < Admin::ApplicationController
    before_action :set_post, only: [:show, :edit, :update, :destroy]
    
    def index
      @posts = Post.order(created_at: :desc).page(params[:page])
    end
    
    def destroy
      @post.destroy
      redirect_to admin_posts_path, notice: "ลบสำเร็จ"
    end
    
    private
    
    def set_post
      @post = Post.find(params[:id])
    end
  end
end
```

---

## สรุป

ใน Controllers เราได้เรียนรู้:

1. **ApplicationController** - base class สำหรับทุก controller
2. **7 RESTful actions** - index, show, new, create, edit, update, destroy
3. **Strong Parameters** - require, permit เพื่อความปลอดภัย
4. **before/after/around_action** - filters ก่อน/หลัง actions
5. **skip_before_action** - ข้าม filter เฉพาะ actions
6. **render vs redirect_to** - render ใช้ view, redirect สร้าง request ใหม่
7. **Flash messages** - notice, alert, flash.now
8. **respond_to** - handle multiple formats
9. **params hash** - รับ data จาก request
10. **request object** - ข้อมูลเกี่ยวกับ HTTP request
11. **session/cookies** - เก็บ state ระหว่าง requests
12. **Concerns** - shared controller code
13. **Testing** - Minitest และ RSpec
