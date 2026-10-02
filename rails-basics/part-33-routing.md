# Part 33: Routing (ขั้นตอนที่ 706-730)

## บทนำ

Routing ใน Rails คือระบบที่แมป HTTP requests ไปยัง controller actions ถ้า application เปรียบเป็นอาคาร routing ก็คือแผนผังที่บอกว่า request ไหนควรไปห้องไหน

---

## ขั้นตอนที่ 706: routes.rb Basics

### โครงสร้างพื้นฐาน

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # ทุก routes จะอยู่ในบล็อกนี้

  # GET /articles → ArticlesController#index
  get "/articles", to: "articles#index"

  # GET /articles/new → ArticlesController#new
  get "/articles/new", to: "articles#new", as: :new_article

  # POST /articles → ArticlesController#create
  post "/articles", to: "articles#create"

  # GET /articles/:id → ArticlesController#show
  get "/articles/:id", to: "articles#show", as: :article

  # GET /articles/:id/edit → ArticlesController#edit
  get "/articles/:id/edit", to: "articles#edit", as: :edit_article

  # PATCH /articles/:id → ArticlesController#update
  patch "/articles/:id", to: "articles#update"

  # PUT /articles/:id → ArticlesController#update (Rails support ทั้งคู่)
  put "/articles/:id", to: "articles#update"

  # DELETE /articles/:id → ArticlesController#destroy
  delete "/articles/:id", to: "articles#destroy"
end
```

### HTTP Verbs ใน Rails

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # GET - อ่านข้อมูล
  get "/profile", to: "profiles#show"

  # POST - สร้างข้อมูลใหม่
  post "/articles", to: "articles#create"

  # PUT - อัพเดทข้อมูลทั้งหมด (replace)
  put "/articles/:id", to: "articles#update"

  # PATCH - อัพเดทข้อมูลบางส่วน (partial update)
  patch "/articles/:id", to: "articles#update"

  # DELETE - ลบข้อมูล
  delete "/articles/:id", to: "articles#destroy"

  # HEAD - เหมือน GET แต่ไม่มี body
  # (Rails handle ให้อัตโนมัติเมื่อมี GET route)

  # OPTIONS - ตรวจสอบ CORS
  # (จัดการผ่าน rack-cors middleware)
end
```

---

## ขั้นตอนที่ 707: RESTful Routes (resources)

### resources helper

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :articles
end
```

คำสั่งเดียวนี้สร้าง 7 routes:

```
GET    /articles          articles#index   articles_path
GET    /articles/new      articles#new     new_article_path
POST   /articles          articles#create  articles_path
GET    /articles/:id      articles#show    article_path(id)
GET    /articles/:id/edit articles#edit    edit_article_path(id)
PATCH  /articles/:id      articles#update  article_path(id)
DELETE /articles/:id      articles#destroy article_path(id)
```

### only และ except

```ruby
Rails.application.routes.draw do
  # เฉพาะ actions ที่ระบุ
  resources :articles, only: [:index, :show]
  # สร้างเฉพาะ:
  # GET /articles        → index
  # GET /articles/:id    → show

  # ยกเว้น actions ที่ระบุ
  resources :categories, except: [:destroy]
  # สร้างทุก route ยกเว้น DELETE

  # Read-only resources
  resources :tags, only: [:index, :show]

  # Write-only resources (ไม่ค่อยใช้)
  resources :events, only: [:create, :update, :destroy]
end
```

### resource (singular)

```ruby
Rails.application.routes.draw do
  # resource (singular) - ไม่มี :id
  # ใช้เมื่อ resource มีแค่ 1 ต่อ user เช่น profile
  resource :profile
  # สร้าง:
  # GET    /profile/new   → new
  # POST   /profile       → create
  # GET    /profile       → show (ไม่มี :id!)
  # GET    /profile/edit  → edit
  # PATCH  /profile       → update
  # DELETE /profile       → destroy

  # ตัวอย่างการใช้:
  # current_user.profile → แค่ 1 profile ต่อ user
end
```

---

## ขั้นตอนที่ 708: Named Routes

### การตั้งชื่อ Routes

```ruby
Rails.application.routes.draw do
  # as: กำหนดชื่อ route helper
  get "/about", to: "pages#about", as: :about_us
  # สร้าง: about_us_path, about_us_url

  get "/contact", to: "pages#contact", as: :contact_page
  # สร้าง: contact_page_path, contact_page_url

  # Default names จาก resources:
  resources :articles
  # articles_path, article_path(id), new_article_path, edit_article_path(id)

  # Custom names ด้วย as:
  resources :blog_posts, as: :posts
  # posts_path, post_path(id), new_post_path, edit_post_path(id)
end
```

### การใช้ Route Helpers

```ruby
# ใน Controllers
class ArticlesController < ApplicationController
  def create
    @article = Article.create!(article_params)
    redirect_to article_path(@article)  # หรือ redirect_to @article
    # หรือ
    redirect_to articles_path  # ไปที่ index
  end
end

# ใน Views
# _path → relative path (แนะนำสำหรับ links ใน app)
# _url  → absolute URL (ต้องใช้ในอีเมล, external links)

# link_to helpers
link_to "Articles", articles_path
link_to "New Article", new_article_path
link_to "Show", article_path(@article)
link_to "Edit", edit_article_path(@article)

# Form action
form_with(url: articles_path, method: :post)
form_with(url: article_path(@article), method: :patch)

# ใน Mailers (ต้องใช้ _url)
article_url(@article)  # https://myapp.com/articles/1
```

### url_for

```ruby
# url_for เป็น flexible way สร้าง URL
url_for(controller: "articles", action: "show", id: 1)
# => "/articles/1"

url_for(@article)
# => "/articles/1"

url_for(action: :new)
# => "/articles/new" (ถ้าอยู่ใน ArticlesController)

# ใน models (ต้องการ Rails.application.routes.url_helpers)
class Article < ApplicationRecord
  include Rails.application.routes.url_helpers

  def share_url
    article_url(self, host: "myapp.com")
  end
end
```

---

## ขั้นตอนที่ 709: Nested Routes

### Basic Nesting

```ruby
Rails.application.routes.draw do
  resources :articles do
    resources :comments
  end
end

# สร้าง routes:
# GET    /articles/:article_id/comments          comments#index
# GET    /articles/:article_id/comments/new      comments#new
# POST   /articles/:article_id/comments          comments#create
# GET    /articles/:article_id/comments/:id      comments#show
# GET    /articles/:article_id/comments/:id/edit comments#edit
# PATCH  /articles/:article_id/comments/:id      comments#update
# DELETE /articles/:article_id/comments/:id      comments#destroy

# Route helpers:
# article_comments_path(@article)
# new_article_comment_path(@article)
# article_comment_path(@article, @comment)
# edit_article_comment_path(@article, @comment)
```

### Controller สำหรับ Nested Routes

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  before_action :set_article
  before_action :set_comment, only: [:show, :edit, :update, :destroy]

  def index
    @comments = @article.comments.includes(:user).order(created_at: :asc)
  end

  def new
    @comment = @article.comments.build
  end

  def create
    @comment = @article.comments.build(comment_params)
    @comment.user = current_user

    if @comment.save
      redirect_to article_path(@article), notice: "Comment added!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @comment.update(comment_params)
      redirect_to article_path(@article), notice: "Comment updated!"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @comment.destroy
    redirect_to article_path(@article), notice: "Comment deleted!"
  end

  private

  def set_article
    @article = Article.find(params[:article_id])
  end

  def set_comment
    @comment = @article.comments.find(params[:id])
  end

  def comment_params
    params.require(:comment).permit(:body)
  end
end
```

### Shallow Nesting

```ruby
Rails.application.routes.draw do
  # Shallow nesting - ลด URL complexity
  resources :articles do
    resources :comments, shallow: true
  end
end

# สร้าง routes:
# GET    /articles/:article_id/comments      comments#index
# GET    /articles/:article_id/comments/new  comments#new
# POST   /articles/:article_id/comments      comments#create
# GET    /comments/:id                       comments#show     (shallow!)
# GET    /comments/:id/edit                  comments#edit     (shallow!)
# PATCH  /comments/:id                       comments#update   (shallow!)
# DELETE /comments/:id                       comments#destroy  (shallow!)

# หรือใช้ shallow block:
resources :articles do
  shallow do
    resources :comments
    resources :likes
  end
end
```

### Deep Nesting (ไม่แนะนำ)

```ruby
# ❌ ไม่แนะนำ - nested มากกว่า 2 ระดับ
resources :users do
  resources :articles do
    resources :comments do
      resources :likes  # URL ยาวเกินไป: /users/1/articles/2/comments/3/likes/4
    end
  end
end

# ✅ แนะนำ - ใช้ shallow หรือ restructure
resources :articles do
  resources :comments, shallow: true
end

resources :comments do
  resources :likes, shallow: true
end
```

---

## ขั้นตอนที่ 710: Namespace และ Scope

### Namespace

```ruby
Rails.application.routes.draw do
  namespace :admin do
    root "dashboard#index"
    resources :articles
    resources :users
    resources :categories
  end
end

# สร้าง routes:
# GET /admin/articles → Admin::ArticlesController#index
# URL prefix: /admin/
# Module prefix: Admin::
# Helper prefix: admin_articles_path

# Controller:
# app/controllers/admin/articles_controller.rb
module Admin
  class ArticlesController < ApplicationController
    # ...
  end
end

# หรือ:
class Admin::ArticlesController < ApplicationController
  # ...
end
```

### Scope

```ruby
Rails.application.routes.draw do
  # scope - เพิ่ม URL prefix เท่านั้น (ไม่เปลี่ยน module/helper)
  scope "/admin" do
    resources :articles  # URL: /admin/articles
    # Controller: ArticlesController (ไม่มี Admin::)
    # Helper: articles_path (ไม่มี admin_)
  end

  # scope :module - เปลี่ยน module เท่านั้น (ไม่เปลี่ยน URL)
  scope module: :admin do
    resources :articles  # URL: /articles
    # Controller: Admin::ArticlesController
    # Helper: articles_path
  end

  # scope :as - เปลี่ยน helper prefix เท่านั้น
  scope as: :admin do
    resources :articles  # URL: /articles
    # Controller: ArticlesController
    # Helper: admin_articles_path
  end

  # รวมทั้งหมด = namespace
  scope "/admin", module: :admin, as: :admin do
    resources :articles
    # URL: /admin/articles
    # Controller: Admin::ArticlesController
    # Helper: admin_articles_path
  end
end
```

### ตัวอย่าง Namespace ใน Production App

```ruby
Rails.application.routes.draw do
  # Public routes
  root "home#index"
  resources :articles, only: [:index, :show]

  # Authentication
  devise_for :users, controllers: {
    sessions: "users/sessions",
    registrations: "users/registrations",
    passwords: "users/passwords"
  }

  # User dashboard
  namespace :dashboard do
    root "overview#index"
    resources :articles
    resources :profile, only: [:show, :edit, :update]
    resources :notifications, only: [:index, :update]
  end

  # Admin panel
  namespace :admin do
    root "dashboard#index"
    resources :articles
    resources :users
    resources :categories
    resources :tags
    resources :settings, only: [:index, :update]

    namespace :reports do
      get :traffic
      get :revenue
      get :users
    end
  end

  # API
  namespace :api do
    namespace :v1 do
      resources :articles, only: [:index, :show, :create, :update, :destroy]
      resources :users, only: [:show]

      namespace :auth do
        post :login
        post :logout
        post :refresh
      end
    end

    namespace :v2 do
      resources :articles
    end
  end
end
```

---

## ขั้นตอนที่ 711: Custom Routes

### Collection Routes

```ruby
Rails.application.routes.draw do
  resources :articles do
    # Collection routes - ไม่ต้องการ :id
    collection do
      get :search        # GET /articles/search
      get :popular       # GET /articles/popular
      get :trending      # GET /articles/trending
      post :bulk_delete  # POST /articles/bulk_delete
      patch :bulk_update # PATCH /articles/bulk_update
    end
  end
end

# Controller:
class ArticlesController < ApplicationController
  def search
    @articles = Article.search(params[:q])
    render :index
  end

  def popular
    @articles = Article.popular.limit(20)
    render :index
  end

  def bulk_delete
    Article.where(id: params[:ids]).destroy_all
    redirect_to articles_path, notice: "Articles deleted"
  end
end
```

### Member Routes

```ruby
Rails.application.routes.draw do
  resources :articles do
    # Member routes - ต้องการ :id
    member do
      post :publish      # POST /articles/:id/publish
      post :unpublish    # POST /articles/:id/unpublish
      post :archive      # POST /articles/:id/archive
      get  :preview      # GET /articles/:id/preview
      post :duplicate    # POST /articles/:id/duplicate
    end
  end
end

# Short form:
resources :articles do
  post :publish, on: :member
  post :unpublish, on: :member
  get :preview, on: :member
  get :search, on: :collection
end

# Controller:
class ArticlesController < ApplicationController
  before_action :set_article, only: [:publish, :unpublish, :archive, :preview, :duplicate]

  def publish
    @article.publish!
    redirect_to @article, notice: "Article published!"
  end

  def unpublish
    @article.update!(status: "draft")
    redirect_to @article, notice: "Article unpublished"
  end

  def preview
    render :show  # แสดง preview โดยไม่ต้อง publish
  end

  def duplicate
    new_article = @article.dup
    new_article.title = "Copy of #{@article.title}"
    new_article.status = "draft"
    new_article.save!
    redirect_to edit_article_path(new_article), notice: "Article duplicated!"
  end
end
```

---

## ขั้นตอนที่ 712: Root Route

```ruby
Rails.application.routes.draw do
  # Root route
  root "home#index"
  # GET / → HomeController#index
  # root_path, root_url

  # Authenticated root
  # ใช้ Devise helper authenticated:
  authenticated :user do
    root "dashboard#index", as: :authenticated_root
  end

  unauthenticated do
    root "home#index"
  end

  # หรือ handle ใน controller:
  root "home#index"
  # ใน HomeController:
  # def index
  #   redirect_to dashboard_path if user_signed_in?
  # end
end
```

---

## ขั้นตอนที่ 713: Constraints

### Pattern Constraints

```ruby
Rails.application.routes.draw do
  # Format constraint - :id ต้องเป็นตัวเลขเท่านั้น
  resources :articles, constraints: { id: /\d+/ }

  # Custom path constraint
  get "/users/:username",
    to: "users#show",
    constraints: { username: /[a-zA-Z0-9_]+/ },
    as: :user_profile

  # Subdomain constraint
  constraints subdomain: "api" do
    namespace :api do
      resources :articles
    end
  end

  # Format constraint
  resources :articles, constraints: { format: "json" }
end
```

### Custom Constraint Class

```ruby
# app/constraints/admin_constraint.rb
class AdminConstraint
  def matches?(request)
    return false unless request.session[:user_id]
    user = User.find_by(id: request.session[:user_id])
    user&.admin?
  end
end

# config/routes.rb
Rails.application.routes.draw do
  namespace :admin, constraints: AdminConstraint.new do
    resources :articles
    resources :users
  end
end
```

```ruby
# Lambda constraint
Rails.application.routes.draw do
  constraints ->(req) { req.env["HTTP_USER_AGENT"] !~ /MSIE/ } do
    resources :articles
  end

  # IP constraint
  constraints ip: /127\.0\.0\.1/ do
    get "/debug", to: "debug#index"
  end
end
```

---

## ขั้นตอนที่ 714: Route Helpers (_path, _url)

### _path vs _url

```ruby
# _path = relative path (เริ่มด้วย /)
articles_path          # => "/articles"
article_path(1)        # => "/articles/1"
new_article_path       # => "/articles/new"
edit_article_path(1)   # => "/articles/1/edit"

# _url = absolute URL (มี protocol + host)
articles_url           # => "http://localhost:3000/articles"
article_url(1)         # => "http://localhost:3000/articles/1"

# เมื่อไหร่ใช้อะไร?
# _path: ทั่วไปใน views และ controllers
# _url: ใน emails, external redirects, API responses

# ตัวอย่างใน Mailer (ต้องใช้ _url):
class ArticleMailer < ApplicationMailer
  def new_article(article)
    @article = article
    @article_url = article_url(article)  # ต้องใช้ _url
    mail(to: "user@example.com", subject: "New Article")
  end
end
```

### Route Helpers กับ Parameters

```ruby
# ส่ง ID โดยตรง
article_path(1)
# => "/articles/1"

# ส่ง object (Rails ใช้ to_param)
article_path(@article)
# => "/articles/42"

# Nested routes
article_comment_path(@article, @comment)
# => "/articles/1/comments/5"

# เพิ่ม query parameters
articles_path(page: 2, sort: "title")
# => "/articles?page=2&sort=title"

# สร้าง path สำหรับ named routes
about_path
# => "/about"

# format
article_path(@article, format: :json)
# => "/articles/1.json"

# Anchor
article_path(@article, anchor: "comments")
# => "/articles/1#comments"
```

### url_options

```ruby
# ตั้งค่า default_url_options
class ApplicationController < ActionController::Base
  def default_url_options
    { host: ENV["APP_HOST"] || "localhost:3000" }
  end
end

# config/environments/development.rb
config.action_mailer.default_url_options = {
  host: "localhost",
  port: 3000
}

# config/environments/production.rb
config.action_mailer.default_url_options = {
  host: "myapp.com",
  protocol: "https"
}
```

---

## ขั้นตอนที่ 715: Advanced Routes

### Routes with Format

```ruby
Rails.application.routes.draw do
  resources :articles, defaults: { format: :json }
  # GET /articles → articles#index (JSON)

  # Format-specific routes
  resources :articles do
    get :feed, on: :collection, defaults: { format: :atom }
    # GET /articles/feed.atom
  end
end

# Controller:
class ArticlesController < ApplicationController
  def index
    @articles = Article.published

    respond_to do |format|
      format.html
      format.json { render json: @articles }
      format.atom { render layout: false }
    end
  end

  def feed
    @articles = Article.published.limit(20)
    respond_to do |format|
      format.atom { render layout: false }
    end
  end
end
```

### Routes with Redirect

```ruby
Rails.application.routes.draw do
  # Simple redirect
  get "/home", to: redirect("/")

  # Redirect กับ dynamic path
  get "/articles/:id", to: redirect("/posts/%{id}")

  # Redirect ด้วย proc
  get "/search",
    to: redirect { |params, request|
      "/articles?q=#{request.query_parameters[:q]}"
    }

  # Redirect ด้วย status code (default 301)
  get "/old-path", to: redirect("/new-path", status: 302)
end
```

### Routes with Constraints และ Wildcards

```ruby
Rails.application.routes.draw do
  # Wildcard segment
  get "/articles/*slug", to: "articles#show_by_slug"
  # จะ match: /articles/2024/my-article-title
  # params[:slug] = "2024/my-article-title"

  # Optional segment
  get "/articles(/:year(/:month))", to: "articles#archive"
  # จะ match:
  # /articles
  # /articles/2024
  # /articles/2024/01

  # Multiple optional segments
  resources :articles do
    get ":year/:month", on: :collection, to: "articles#archive", as: :archive
    # GET /articles/2024/01 → articles#archive
    # archive_articles_path(year: 2024, month: "01")
  end
end
```

---

## ขั้นตอนที่ 716: Route Concerns

```ruby
Rails.application.routes.draw do
  # กำหนด concern
  concern :commentable do
    resources :comments
  end

  concern :taggable do
    resources :tags, only: [:index, :create, :destroy]
  end

  concern :likeable do
    member do
      post :like
      post :unlike
    end
  end

  # ใช้ concerns
  resources :articles, concerns: [:commentable, :taggable, :likeable]
  resources :photos, concerns: [:commentable, :likeable]
  resources :videos, concerns: [:commentable, :taggable]
end

# สร้าง routes:
# GET /articles/:article_id/comments
# GET /articles/:article_id/tags
# POST /articles/:id/like
# GET /photos/:photo_id/comments
# POST /photos/:id/like
# etc.
```

---

## ขั้นตอนที่ 717: Routes สำหรับ API

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      # Resources สำหรับ API
      resources :articles, only: [:index, :show, :create, :update, :destroy]
      resources :users, only: [:show, :create, :update]

      # Authentication
      namespace :auth do
        post :login
        post :logout
        post :register
        post :refresh_token
        post :forgot_password
        put :reset_password
      end

      # Search
      get "/search", to: "search#index"

      # Health check
      get "/health", to: "health#index"
    end

    namespace :v2 do
      resources :articles
    end
  end
end

# API Controller:
class Api::V1::ArticlesController < Api::BaseController
  before_action :authenticate_api_user!
  before_action :set_article, only: [:show, :update, :destroy]

  def index
    @articles = Article.published
                       .page(params[:page])
                       .per(params[:per_page] || 20)

    render json: {
      articles: @articles.as_json(include: :user),
      meta: {
        total: @articles.total_count,
        page: @articles.current_page,
        per_page: @articles.limit_value
      }
    }
  end

  def show
    render json: @article.as_json(include: [:user, :tags, :comments])
  end

  def create
    @article = current_api_user.articles.build(article_params)
    if @article.save
      render json: @article, status: :created
    else
      render json: { errors: @article.errors }, status: :unprocessable_entity
    end
  end

  def update
    if @article.update(article_params)
      render json: @article
    else
      render json: { errors: @article.errors }, status: :unprocessable_entity
    end
  end

  def destroy
    @article.destroy
    head :no_content
  end

  private

  def set_article
    @article = Article.find(params[:id])
  end

  def article_params
    params.require(:article).permit(:title, :body, :status, tag_ids: [])
  end
end
```

---

## ขั้นตอนที่ 718: Testing Routes

### Route Tests ด้วย Minitest

```ruby
# test/routing/articles_routing_test.rb
require "test_helper"

class ArticlesRoutingTest < ActionDispatch::IntegrationTest
  test "routes to articles#index" do
    assert_routing "/articles", controller: "articles", action: "index"
  end

  test "routes to articles#show" do
    assert_routing "/articles/1", controller: "articles", action: "show", id: "1"
  end

  test "routes to articles#new" do
    assert_routing "/articles/new", controller: "articles", action: "new"
  end

  test "routes to articles#create" do
    assert_routing({ method: "post", path: "/articles" },
      { controller: "articles", action: "create" })
  end

  test "routes to articles#edit" do
    assert_routing "/articles/1/edit",
      controller: "articles", action: "edit", id: "1"
  end

  test "routes to articles#update via patch" do
    assert_routing({ method: "patch", path: "/articles/1" },
      { controller: "articles", action: "update", id: "1" })
  end

  test "routes to articles#destroy" do
    assert_routing({ method: "delete", path: "/articles/1" },
      { controller: "articles", action: "destroy", id: "1" })
  end

  test "routes recognizes articles path" do
    assert_recognizes(
      { controller: "articles", action: "index" },
      "/articles"
    )
  end

  test "generates articles path" do
    assert_generates "/articles", controller: "articles", action: "index"
    assert_generates "/articles/1", controller: "articles", action: "show", id: "1"
  end
end
```

### Route Tests ด้วย RSpec

```ruby
# spec/routing/articles_routing_spec.rb
require "rails_helper"

RSpec.describe "Articles routing", type: :routing do
  describe "routing" do
    it "routes GET /articles to articles#index" do
      expect(get: "/articles").to route_to("articles#index")
    end

    it "routes GET /articles/1 to articles#show" do
      expect(get: "/articles/1").to route_to(
        controller: "articles",
        action: "show",
        id: "1"
      )
    end

    it "routes POST /articles to articles#create" do
      expect(post: "/articles").to route_to("articles#create")
    end

    it "routes PATCH /articles/1 to articles#update" do
      expect(patch: "/articles/1").to route_to(
        controller: "articles",
        action: "update",
        id: "1"
      )
    end

    it "routes DELETE /articles/1 to articles#destroy" do
      expect(delete: "/articles/1").to route_to(
        controller: "articles",
        action: "destroy",
        id: "1"
      )
    end

    it "routes to admin namespace" do
      expect(get: "/admin/articles").to route_to("admin/articles#index")
    end

    it "does not route to non-existent paths" do
      expect(get: "/invalid-path").not_to be_routable
    end
  end

  describe "named routes" do
    it "generates articles_path" do
      expect(articles_path).to eq("/articles")
    end

    it "generates article_path" do
      expect(article_path(id: 1)).to eq("/articles/1")
    end

    it "generates new_article_path" do
      expect(new_article_path).to eq("/articles/new")
    end

    it "generates edit_article_path" do
      expect(edit_article_path(id: 1)).to eq("/articles/1/edit")
    end
  end
end
```

---

## ขั้นตอนที่ 719-730: Routes Best Practices

### ตัวอย่าง routes.rb ที่สมบูรณ์

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # ===== Health Check =====
  get "/health", to: "health#show", as: :health_check

  # ===== Static Pages =====
  get "/about",   to: "pages#about",   as: :about
  get "/contact", to: "pages#contact", as: :contact
  get "/privacy", to: "pages#privacy", as: :privacy
  get "/terms",   to: "pages#terms",   as: :terms

  # ===== Authentication =====
  devise_for :users, controllers: {
    sessions: "users/sessions",
    registrations: "users/registrations",
    passwords: "users/passwords",
    confirmations: "users/confirmations"
  }

  # ===== Root =====
  authenticated :user do
    root "dashboard#index", as: :authenticated_root
  end
  root "home#index"

  # ===== Public Resources =====
  resources :articles, only: [:index, :show] do
    collection do
      get :search
      get :popular
    end

    resources :comments, only: [:create, :destroy]
  end

  resources :categories, only: [:index, :show]
  resources :tags, only: [:index, :show]

  # ===== User Dashboard =====
  namespace :dashboard do
    root "overview#index"

    resources :articles do
      member do
        post :publish
        post :unpublish
        post :archive
        get :preview
      end
    end

    resource :profile, only: [:show, :edit, :update]
    resources :notifications, only: [:index, :update] do
      collection do
        patch :mark_all_read
      end
    end
  end

  # ===== Admin Panel =====
  namespace :admin, constraints: AdminConstraint.new do
    root "dashboard#index"

    resources :articles do
      collection do
        get :pending
        post :bulk_approve
      end
    end

    resources :users do
      member do
        post :suspend
        post :activate
        post :make_admin
      end
    end

    resources :categories
    resources :tags
    resources :settings, only: [:index, :update]

    namespace :reports do
      get :traffic,  to: "traffic#index"
      get :revenue,  to: "revenue#index"
      get :users,    to: "users#index"
      get :articles, to: "articles#index"
    end
  end

  # ===== API =====
  namespace :api, defaults: { format: :json } do
    namespace :v1 do
      # Authentication
      namespace :auth do
        post :login
        post :logout
        post :register
        post :refresh
        post :forgot_password
        patch :reset_password
      end

      # Resources
      resources :articles, only: [:index, :show, :create, :update, :destroy] do
        resources :comments, only: [:index, :create, :destroy]
      end

      resources :users, only: [:show, :update]
      resources :categories, only: [:index, :show]
      resources :tags, only: [:index]

      # Search
      get "/search", to: "search#index"
    end
  end
end
```

---

## แบบฝึกหัด Part 33 (ขั้นตอนที่ 706-730)

**ข้อ 1:** เขียน routes ที่สร้าง CRUD ครบสำหรับ Product

```ruby
# คำตอบ:
Rails.application.routes.draw do
  resources :products
end
# สร้าง:
# GET    /products          → index
# GET    /products/new      → new
# POST   /products          → create
# GET    /products/:id      → show
# GET    /products/:id/edit → edit
# PATCH  /products/:id      → update
# DELETE /products/:id      → destroy
```

**ข้อ 2:** สร้าง routes สำหรับ admin namespace

```ruby
# คำตอบ:
namespace :admin do
  root "dashboard#index"
  resources :users
  resources :products
  resources :orders
end
```

**ข้อ 3:** สร้าง nested routes สำหรับ Order มี LineItems

```ruby
# คำตอบ:
resources :orders do
  resources :line_items, shallow: true
end
# สร้าง:
# GET    /orders/:order_id/line_items      → index
# POST   /orders/:order_id/line_items      → create
# GET    /line_items/:id                   → show (shallow)
# PATCH  /line_items/:id                   → update (shallow)
# DELETE /line_items/:id                   → destroy (shallow)
```

**ข้อ 4:** เพิ่ม member routes สำหรับ publish, archive ให้กับ Article

```ruby
# คำตอบ:
resources :articles do
  member do
    post :publish
    post :archive
    get :preview
  end
end
```

**ข้อ 5:** เพิ่ม collection routes สำหรับ search, popular ให้กับ Product

```ruby
# คำตอบ:
resources :products do
  collection do
    get :search
    get :popular
    get :featured
  end
end
```

**ข้อ 6:** สร้าง routes สำหรับ User Profile (singular resource)

```ruby
# คำตอบ:
resource :profile do
  member do
    get :avatar
    delete :avatar
  end
end
```

**ข้อ 7:** สร้าง API routes ใน namespace /api/v1

```ruby
# คำตอบ:
namespace :api do
  namespace :v1 do
    resources :articles, only: [:index, :show, :create, :update, :destroy]
    resources :users, only: [:show, :update]
    post "/auth/login", to: "auth#login"
    post "/auth/logout", to: "auth#logout"
  end
end
```

**ข้อ 8:** เขียน constraint ที่อนุญาตเฉพาะ admin users

```ruby
# คำตอบ:
# app/constraints/admin_constraint.rb
class AdminConstraint
  def matches?(request)
    session_user_id = request.session[:user_id]
    return false unless session_user_id

    User.find_by(id: session_user_id)&.admin?
  end
end

# config/routes.rb
namespace :admin, constraints: AdminConstraint.new do
  resources :users
  resources :articles
end
```

**ข้อ 9:** สร้าง route redirect จาก /old-articles ไป /articles

```ruby
# คำตอบ:
get "/old-articles", to: redirect("/articles")
get "/old-articles/:id", to: redirect("/articles/%{id}")
```

**ข้อ 10:** ใช้ route concerns สำหรับ commentable resources

```ruby
# คำตอบ:
concern :commentable do
  resources :comments, only: [:index, :create, :destroy]
end

concern :likeable do
  member do
    post :like
    delete :like, action: :unlike
  end
end

resources :articles, concerns: [:commentable, :likeable]
resources :photos,   concerns: [:commentable, :likeable]
resources :videos,   concerns: [:commentable]
```

**ข้อ 11:** เขียน Route test สำหรับ articles#index และ articles#show

```ruby
# คำตอบ:
# spec/routing/articles_routing_spec.rb
RSpec.describe "Articles routing", type: :routing do
  it "routes GET /articles to articles#index" do
    expect(get: "/articles").to route_to("articles#index")
  end

  it "routes GET /articles/1 to articles#show" do
    expect(get: "/articles/1").to route_to(
      controller: "articles",
      action: "show",
      id: "1"
    )
  end
end
```

**ข้อ 12:** สร้าง custom route สำหรับ user profile ด้วย username

```ruby
# คำตอบ:
get "/@:username",
  to: "users#show",
  as: :user_profile,
  constraints: { username: /[a-zA-Z0-9_]+/ }

# หรือ:
get "/users/:username",
  to: "users#show",
  as: :user_by_username,
  constraints: { username: /[a-zA-Z0-9_.-]+/ }
```

**ข้อ 13:** อธิบายความแตกต่างระหว่าง _path และ _url helpers

```
คำตอบ:
_path: สร้าง relative path
- articles_path → "/articles"
- article_path(1) → "/articles/1"
- ใช้สำหรับ links ภายใน app
- เร็วกว่าเล็กน้อย (ไม่ต้องใส่ host)

_url: สร้าง absolute URL
- articles_url → "http://localhost:3000/articles"
- article_url(1) → "http://localhost:3000/articles/1"
- ใช้สำหรับ:
  * Email links (ต้องมี full URL)
  * External redirects
  * JSON API responses
  * Social sharing
```

**ข้อ 14:** สร้าง routes สำหรับ newsletter subscription

```ruby
# คำตอบ:
resource :subscription, only: [:new, :create, :destroy] do
  get :confirm, on: :member
  post :unsubscribe, on: :collection
end

# หรือ:
namespace :newsletter do
  post :subscribe
  delete :unsubscribe
  get :confirm
end
```

**ข้อ 15:** สร้าง wildcard route สำหรับ CMS pages

```ruby
# คำตอบ:
# ต้องอยู่ท้ายสุดใน routes.rb!
get "/*permalink",
  to: "pages#show",
  as: :cms_page,
  constraints: { permalink: /[a-z0-9\-\/]+/ }

# Controller:
class PagesController < ApplicationController
  def show
    @page = Page.find_by!(permalink: params[:permalink])
  rescue ActiveRecord::RecordNotFound
    render :not_found, status: :not_found
  end
end
```

**ข้อ 16:** เพิ่ม routes สำหรับ Devise กับ custom controllers

```ruby
# คำตอบ:
devise_for :users,
  path: "",
  path_names: {
    sign_in: "login",
    sign_out: "logout",
    sign_up: "register"
  },
  controllers: {
    sessions: "users/sessions",
    registrations: "users/registrations",
    passwords: "users/passwords",
    confirmations: "users/confirmations",
    unlocks: "users/unlocks"
  }

# Routes สร้าง:
# GET /login → users/sessions#new
# POST /login → users/sessions#create
# DELETE /logout → users/sessions#destroy
# GET /register → users/registrations#new
```

**ข้อ 17:** สร้าง scope สำหรับ locale-prefixed routes

```ruby
# คำตอบ:
scope "(:locale)", locale: /th|en/ do
  root "home#index"
  resources :articles
  resources :categories
end

# สร้าง routes:
# GET /articles          (ไม่มี locale)
# GET /th/articles       (Thai locale)
# GET /en/articles       (English locale)

# ApplicationController:
class ApplicationController < ActionController::Base
  before_action :set_locale

  def set_locale
    I18n.locale = params[:locale] || I18n.default_locale
  end

  def default_url_options
    { locale: I18n.locale == I18n.default_locale ? nil : I18n.locale }
  end
end
```

**ข้อ 18:** เขียน routes สำหรับ shopping cart

```ruby
# คำตอบ:
resource :cart, only: [:show] do
  resources :cart_items, only: [:create, :update, :destroy]
  post :checkout
  post :apply_coupon
  delete :clear
end

# สร้าง:
# GET    /cart              → show
# POST   /cart/cart_items   → create
# PATCH  /cart/cart_items/:id → update
# DELETE /cart/cart_items/:id → destroy
# POST   /cart/checkout     → checkout
```

**ข้อ 19:** สร้าง routes สำหรับ Multi-step form (wizard)

```ruby
# คำตอบ:
namespace :registration do
  get :step1
  post :step1, action: :save_step1
  get :step2
  post :step2, action: :save_step2
  get :step3
  post :step3, action: :save_step3
  get :complete
end

# หรือ:
resources :registrations, only: [] do
  collection do
    get "step/:step",     action: :show,   as: :step
    post "step/:step",    action: :update, as: :update_step
    get "complete",       action: :complete
  end
end
```

**ข้อ 20:** สร้าง routes สำหรับ Two-factor authentication

```ruby
# คำตอบ:
namespace :two_factor_auth do
  get  :setup
  post :enable
  post :disable
  get  :verify
  post :verify, action: :confirm
  post :generate_backup_codes
  get  :backup_codes
end

# หรือผ่าน devise:
devise_for :users
devise_scope :user do
  get  "/two_factor/setup",   to: "two_factor#setup"
  post "/two_factor/enable",  to: "two_factor#enable"
  post "/two_factor/verify",  to: "two_factor#verify"
end
```

**ข้อ 21:** สร้าง routes สำหรับ Social features (follow, unfollow)

```ruby
# คำตอบ:
resources :users, only: [:index, :show] do
  member do
    post :follow
    delete :follow, action: :unfollow, as: :unfollow
    get :followers
    get :following
  end
end

# หรือ:
resources :follows, only: [:create, :destroy]
# POST   /follows        { user_id: X }   → follow
# DELETE /follows/:id                     → unfollow
```

**ข้อ 22:** สร้าง routes สำหรับ File upload

```ruby
# คำตอบ:
resources :documents, only: [:index, :show, :create, :destroy] do
  member do
    get :download
    post :process
  end
  collection do
    post :bulk_upload
  end
end

resources :avatars, only: [:create, :destroy]
```

**ข้อ 23:** เขียน integration test สำหรับ routes

```ruby
# คำตอบ:
# test/integration/routing_test.rb
class RoutingTest < ActionDispatch::IntegrationTest
  test "admin routes require admin authentication" do
    get admin_articles_path
    assert_redirected_to new_user_session_path
  end

  test "api routes return json" do
    get api_v1_articles_path, headers: { "Accept" => "application/json" }
    assert_response :unauthorized  # ถ้าต้องการ auth
  end

  test "nested routes work correctly" do
    article = create(:article)
    get article_comments_path(article)
    assert_response :success
  end
end
```

**ข้อ 24:** สร้าง routes สำหรับ Search ที่ซับซ้อน

```ruby
# คำตอบ:
namespace :search do
  get :articles
  get :users
  get :tags
  get :global, to: "search#global", as: :global
end

# หรือ:
get "/search",          to: "search#index",    as: :search
get "/search/articles", to: "search#articles", as: :search_articles
get "/search/users",    to: "search#users",    as: :search_users

# ใน controller:
class SearchController < ApplicationController
  def index
    @query = params[:q]

    if @query.present?
      @articles = Article.search(@query).limit(10)
      @users = User.search(@query).limit(5)
      @tags = Tag.search(@query).limit(10)
    end
  end
end
```

**ข้อ 25:** ดู routes ที่สร้างและอธิบาย route helpers ที่ได้

```bash
# คำตอบ:
rails routes

# Output จะแสดง:
#         Prefix Verb   URI Pattern                    Controller#Action
#       articles GET    /articles(.:format)             articles#index
#                POST   /articles(.:format)             articles#create
#    new_article GET    /articles/new(.:format)         articles#new
#   edit_article GET    /articles/:id/edit(.:format)    articles#edit
#        article GET    /articles/:id(.:format)         articles#show
#                PATCH  /articles/:id(.:format)         articles#update
#                DELETE /articles/:id(.:format)         articles#destroy

# Route helpers:
# articles_path       → GET /articles
# new_article_path    → GET /articles/new
# article_path(id)    → GET /articles/:id
# edit_article_path(id) → GET /articles/:id/edit

# ดู routes เฉพาะ controller:
rails routes -c articles

# ดู routes ที่ match URL:
rails routes -g /admin
```

---

## สรุป Part 33

ในบทนี้เราได้เรียนรู้:

1. **routes.rb basics** - HTTP verbs, path patterns
2. **RESTful routes** - resources helper สร้าง 7 routes อัตโนมัติ
3. **Named routes** - route helpers (_path, _url)
4. **Nested routes** - parent-child relationships, shallow nesting
5. **Namespace & Scope** - admin, api namespacing
6. **Custom routes** - collection, member, redirects
7. **Root route** - authenticated/unauthenticated roots
8. **Constraints** - pattern, class, lambda constraints
9. **Route Concerns** - reusable route sets
10. **Testing routes** - Minitest และ RSpec

Routing เป็นส่วนสำคัญที่เชื่อม request กับ controller Rails routing system มีความยืดหยุ่นสูงและรองรับ patterns หลากหลาย

---

*ต่อไป: Part 34 - Controllers (ขั้นตอนที่ 731-755)*
