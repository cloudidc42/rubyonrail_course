# ตอนที่ 33: Routing (Steps 706-730)

## บทนำ

Routing เป็นส่วนที่กำหนดว่า URL ไหนจะถูก handle โดย controller และ action ไหน ใน Rails routing ถูกกำหนดในไฟล์ `config/routes.rb` ซึ่งเป็น Ruby DSL (Domain Specific Language) ที่อ่านง่ายและทรงพลัง

---

## Step 706: routes.rb - ไฟล์หลักของ Routing

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Routes ทั้งหมดถูกกำหนดใน block นี้
  
  # RESTful resources
  resources :posts
  
  # Root route
  root "pages#home"
  
  # Custom route
  get "/about", to: "pages#about"
end
```

### ดู Routes ทั้งหมด

```bash
# ดู routes ทั้งหมด
rails routes

# ดูเฉพาะ routes ที่เกี่ยวกับ posts
rails routes -c posts

# ดู routes ที่ match pattern
rails routes | grep post

# ดูใน browser (development เท่านั้น)
# http://localhost:3000/rails/info/routes
```

---

## Step 707: resources :posts (7 RESTful Routes)

```ruby
# config/routes.rb
resources :posts
```

คำสั่งนี้สร้าง routes 7 เส้นทาง:

| HTTP Method | Path | Controller#Action | Named Route | ความหมาย |
|------------|------|-------------------|-------------|----------|
| GET | /posts | posts#index | posts_path | แสดงรายการ posts ทั้งหมด |
| GET | /posts/new | posts#new | new_post_path | แสดงฟอร์มสร้าง post ใหม่ |
| POST | /posts | posts#create | posts_path | สร้าง post ใหม่ |
| GET | /posts/:id | posts#show | post_path | แสดง post เฉพาะ |
| GET | /posts/:id/edit | posts#edit | edit_post_path | แสดงฟอร์มแก้ไข post |
| PATCH/PUT | /posts/:id | posts#update | post_path | อัปเดต post เฉพาะ |
| DELETE | /posts/:id | posts#destroy | post_path | ลบ post เฉพาะ |

```bash
# Output จาก rails routes
   Prefix  Verb    URI Pattern                 Controller#Action
    posts  GET     /posts(.:format)            posts#index
           POST    /posts(.:format)            posts#create
 new_post  GET     /posts/new(.:format)        posts#new
edit_post  GET     /posts/:id/edit(.:format)   posts#edit
     post  GET     /posts/:id(.:format)        posts#show
           PATCH   /posts/:id(.:format)        posts#update
           PUT     /posts/:id(.:format)        posts#update
           DELETE  /posts/:id(.:format)        posts#destroy
```

### ใช้ Named Routes ใน Views

```erb
<!-- app/views/posts/index.html.erb -->

<!-- Link to index -->
<%= link_to "All Posts", posts_path %>
<!-- => <a href="/posts">All Posts</a> -->

<!-- Link to show -->
<%= link_to "Show", post_path(@post) %>
<%= link_to "Show", post_path(id: @post.id) %>
<!-- => <a href="/posts/1">Show</a> -->

<!-- Link to new -->
<%= link_to "New Post", new_post_path %>
<!-- => <a href="/posts/new">New Post</a> -->

<!-- Link to edit -->
<%= link_to "Edit", edit_post_path(@post) %>
<!-- => <a href="/posts/1/edit">Edit</a> -->

<!-- Delete -->
<%= link_to "Delete", post_path(@post), 
    data: { turbo_method: :delete, turbo_confirm: "แน่ใจ?" } %>
```

### ใช้ Named Routes ใน Controllers

```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    if @post.save
      redirect_to @post         # => /posts/:id
      redirect_to post_path(@post)  # เหมือนกัน
      redirect_to posts_path    # => /posts
    end
  end
  
  def destroy
    @post.destroy
    redirect_to posts_path, notice: "ลบสำเร็จ"
  end
end
```

---

## Step 708: resources with only/except

### only: - ระบุ actions ที่ต้องการ

```ruby
# config/routes.rb
# สร้างเฉพาะ routes ที่ระบุ
resources :posts, only: [:index, :show]
# สร้าง:
# GET /posts       => posts#index
# GET /posts/:id   => posts#show

resources :articles, only: :index
# สร้าง:
# GET /articles    => articles#index
```

### except: - ระบุ actions ที่ไม่ต้องการ

```ruby
# ยกเว้น actions ที่ระบุ
resources :posts, except: [:destroy]
# สร้างทุก routes ยกเว้น DELETE /posts/:id

resources :tags, except: [:new, :edit, :create, :update, :destroy]
# เหมือนกับ only: [:index, :show]
```

### ตัวอย่างการใช้งานจริง

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Blog - ผู้ใช้อ่านได้อย่างเดียว
  resources :posts, only: [:index, :show]
  
  # Comments - สร้างและลบได้ แต่ไม่ edit
  resources :comments, except: [:edit, :update]
  
  # Tags - เฉพาะ list
  resources :tags, only: :index
  
  # Admin can do everything
  namespace :admin do
    resources :posts  # ทุก actions
  end
end
```

---

## Step 709: Named Routes (as:)

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # กำหนดชื่อ route
  get "/home", to: "pages#home", as: :homepage
  # => homepage_path, homepage_url
  
  get "/about-us", to: "pages#about", as: :about_us
  # => about_us_path, about_us_url
  
  get "/contact", to: "contacts#new", as: :contact_form
  # => contact_form_path
  
  post "/contact", to: "contacts#create"
  
  # Login/Logout routes
  get "/login", to: "sessions#new", as: :login
  delete "/logout", to: "sessions#destroy", as: :logout
  
  # Profile
  get "/profile", to: "users#profile", as: :profile
  patch "/profile", to: "users#update_profile"
}
```

### ใช้ named routes

```ruby
# ใน controller
redirect_to homepage_path
redirect_to login_url  # full URL รวม host

# ใน views
<%= link_to "Home", homepage_path %>
<%= link_to "Login", login_path %>
<%= link_to "Contact Us", contact_form_path %>

# ใน mailers (ต้องใช้ _url)
link = homepage_url  # http://example.com/home
```

---

## Step 710: Nested Resources

```ruby
# config/routes.rb
resources :posts do
  resources :comments
end
```

สร้าง routes:

```
                      Prefix  Verb    URI Pattern                                   Controller#Action
         post_comments  GET     /posts/:post_id/comments(.:format)                  comments#index
                        POST    /posts/:post_id/comments(.:format)                  comments#create
      new_post_comment  GET     /posts/:post_id/comments/new(.:format)              comments#new
     edit_post_comment  GET     /posts/:post_id/comments/:id/edit(.:format)         comments#edit
          post_comment  GET     /posts/:post_id/comments/:id(.:format)              comments#show
                        PATCH   /posts/:post_id/comments/:id(.:format)              comments#update
                        PUT     /posts/:post_id/comments/:id(.:format)              comments#update
                        DELETE  /posts/:post_id/comments/:id(.:format)              comments#destroy
```

### ใช้ nested routes

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  before_action :set_post
  
  def index
    @comments = @post.comments
  end
  
  def create
    @comment = @post.comments.new(comment_params)
    if @comment.save
      redirect_to post_comments_path(@post)
    end
  end
  
  private
  
  def set_post
    @post = Post.find(params[:post_id])
  end
end
```

```erb
<!-- ใน views -->
<%= link_to "Comments", post_comments_path(@post) %>
<%= link_to "New Comment", new_post_comment_path(@post) %>
<%= link_to "Show", post_comment_path(@post, @comment) %>
```

### หลายระดับของ Nesting

```ruby
# config/routes.rb
resources :blogs do
  resources :posts do
    resources :comments
  end
end

# สร้าง routes เช่น:
# /blogs/:blog_id/posts/:post_id/comments
# blog_post_comments_path(blog, post)
```

**แนะนำ:** อย่า nest เกิน 2 ระดับ เพราะจะทำให้ URL ยาวและ code ซับซ้อน

---

## Step 711: Shallow Nested Resources

Shallow nested routes ช่วยลดความซับซ้อนของ URLs ที่ไม่จำเป็น

```ruby
# config/routes.rb
resources :posts do
  resources :comments, shallow: true
end
```

สร้าง:
- `/posts/:post_id/comments` => comments#index (ต้องการ post_id)
- `/posts/:post_id/comments/new` => comments#new (ต้องการ post_id)
- `/posts/:post_id/comments` (POST) => comments#create (ต้องการ post_id)
- `/comments/:id` => comments#show (ไม่ต้องการ post_id)
- `/comments/:id/edit` => comments#edit (ไม่ต้องการ post_id)
- `/comments/:id` (PATCH) => comments#update (ไม่ต้องการ post_id)
- `/comments/:id` (DELETE) => comments#destroy (ไม่ต้องการ post_id)

```ruby
# อีกวิธี - ใช้ shallow block
shallow do
  resources :posts do
    resources :comments
  end
end
```

---

## Step 712: Namespace (Admin Namespace)

```ruby
# config/routes.rb
namespace :admin do
  resources :posts
  resources :users
  resources :categories
end
```

สร้าง routes:
- `/admin/posts` => `admin/posts#index`
- `/admin/posts/:id` => `admin/posts#show`
- ฯลฯ

Named routes: `admin_posts_path`, `admin_post_path(post)`

### Admin Controller Structure

```ruby
# app/controllers/admin/application_controller.rb
module Admin
  class ApplicationController < ActionController::Base
    before_action :authenticate_admin!
    
    layout "admin"
    
    private
    
    def authenticate_admin!
      redirect_to root_path unless current_user&.admin?
    end
  end
end

# app/controllers/admin/posts_controller.rb
module Admin
  class PostsController < Admin::ApplicationController
    def index
      @posts = Post.all
    end
    
    def destroy
      @post = Post.find(params[:id])
      @post.destroy
      redirect_to admin_posts_path, notice: "Post deleted"
    end
  end
end
```

### ใช้ใน views

```erb
<!-- Admin navigation -->
<%= link_to "Posts", admin_posts_path %>
<%= link_to "Users", admin_users_path %>
<%= link_to "Edit", edit_admin_post_path(@post) %>
```

---

## Step 713: Scope

Scope ช่วย group routes โดยไม่สร้าง module ใหม่

```ruby
# config/routes.rb

# scope path - เพิ่ม prefix ใน URL แต่ controller ยังอยู่ที่เดิม
scope "/admin" do
  resources :posts
end
# URL: /admin/posts  => PostsController#index
# Named: posts_path (ไม่มี admin prefix)

# scope module - เพิ่ม module prefix ใน controller แต่ URL เหมือนเดิม
scope module: "admin" do
  resources :posts
end
# URL: /posts  => Admin::PostsController#index

# scope as - เพิ่ม prefix ใน named routes
scope as: "admin" do
  resources :posts
end
# URL: /posts  => PostsController#index
# Named: admin_posts_path

# รวม path + module + as
scope path: "/admin", module: "admin", as: "admin" do
  resources :posts  # เหมือน namespace แต่ควบคุมได้มากกว่า
end
```

### ตัวอย่างการใช้งาน scope

```ruby
# API versioning
scope "/api" do
  scope "/v1" do
    resources :users
    resources :posts
  end
  
  scope "/v2" do
    resources :users
    resources :posts
  end
end

# /api/v1/users  => UsersController#index
# /api/v2/users  => UsersController#index (ยัง controller เดิม!)

# ดีกว่าใช้ namespace สำหรับ versioning
namespace :api do
  namespace :v1 do
    resources :users
    resources :posts
  end
end
# /api/v1/users  => Api::V1::UsersController#index
```

---

## Step 714: Custom Routes

### GET Routes

```ruby
# config/routes.rb
get "/about", to: "pages#about"
get "/contact", to: "pages#contact"
get "/privacy-policy", to: "pages#privacy", as: :privacy_policy
get "/terms", to: "pages#terms", as: :terms

# ด้วย block
get "/profile" do
  redirect_to "/users/me"
end
```

### POST Routes

```ruby
post "/login", to: "sessions#create"
post "/register", to: "registrations#create"
post "/newsletter/subscribe", to: "newsletters#subscribe"
```

### PATCH/PUT Routes

```ruby
patch "/profile", to: "profiles#update"
put "/settings", to: "settings#update"
```

### DELETE Routes

```ruby
delete "/logout", to: "sessions#destroy"
delete "/account", to: "accounts#destroy"
```

### Route ที่รับ parameter

```ruby
# Named parameter
get "/users/:username", to: "users#show", as: :user_profile
# => user_profile_path("john") => /users/john

# Multiple parameters
get "/posts/:year/:month/:day/:slug", to: "posts#show", as: :dated_post
# => dated_post_path(2024, 1, 15, "my-post")
```

---

## Step 715: Root Route

```ruby
# config/routes.rb
root "pages#home"
# หรือ
root to: "pages#home"
# หรือ
root "posts#index"

# GET / => pages#home
# Named: root_path, root_url
```

### ใช้ root route ใน code

```ruby
# Controller
redirect_to root_path

# View
<%= link_to "Home", root_path %>

# หลังจาก login
after_sign_in_path_for(resource) = root_path
```

---

## Step 716: Member Routes

Member routes ใช้กับ record เฉพาะ (ต้องมี :id)

```ruby
# config/routes.rb
resources :posts do
  member do
    post :publish      # POST /posts/:id/publish
    delete :unpublish  # DELETE /posts/:id/unpublish
    get :preview       # GET /posts/:id/preview
    patch :archive     # PATCH /posts/:id/archive
  end
end

# หรือ inline syntax
resources :posts do
  post :publish, on: :member
  get :preview, on: :member
end
```

Routes ที่สร้าง:
- `POST /posts/:id/publish` => posts#publish
- Named: `publish_post_path(@post)`
- `GET /posts/:id/preview` => posts#preview
- Named: `preview_post_path(@post)`

### ใช้ใน Controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def publish
    @post = Post.find(params[:id])
    @post.update!(published_at: Time.current, status: "published")
    redirect_to @post, notice: "เผยแพร่แล้ว"
  end
  
  def preview
    @post = Post.find(params[:id])
    render :show, layout: "preview"
  end
  
  def archive
    @post = Post.find(params[:id])
    @post.archive!
    redirect_to posts_path
  end
end
```

```erb
<!-- ใน views -->
<%= link_to "Preview", preview_post_path(@post) %>

<%= button_to "Publish", publish_post_path(@post), method: :post %>

<%= button_to "Archive", archive_post_path(@post), 
    method: :patch,
    data: { confirm: "แน่ใจ?" } %>
```

---

## Step 717: Collection Routes

Collection routes ใช้กับ collection ทั้งหมด (ไม่มี :id)

```ruby
# config/routes.rb
resources :posts do
  collection do
    get :published    # GET /posts/published
    get :drafts       # GET /posts/drafts
    get :archived     # GET /posts/archived
    delete :bulk_destroy  # DELETE /posts/bulk_destroy
  end
end

# หรือ inline
resources :posts do
  get :published, on: :collection
  get :drafts, on: :collection
end
```

Routes ที่สร้าง:
- `GET /posts/published` => posts#published
- Named: `published_posts_path`
- `GET /posts/drafts` => posts#drafts
- Named: `drafts_posts_path`

### ใช้ใน Controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def published
    @posts = Post.where(status: "published").order(published_at: :desc)
    render :index
  end
  
  def drafts
    @posts = current_user.posts.where(status: "draft")
    render :index
  end
  
  def archived
    @posts = Post.where(archived: true)
    render :index
  end
  
  def bulk_destroy
    ids = params[:post_ids]
    Post.where(id: ids).destroy_all
    redirect_to posts_path, notice: "ลบ #{ids.count} posts สำเร็จ"
  end
end
```

---

## Step 718: Constraints on Routes

### Format Constraints

```ruby
# config/routes.rb

# เฉพาะ integer
get "/posts/:id", to: "posts#show", constraints: { id: /\d+/ }

# Subdomain
constraints subdomain: "api" do
  resources :users
end
# api.example.com/users => users#index

# ด้วย class
class AdminSubdomainConstraint
  def matches?(request)
    request.subdomain == "admin"
  end
end

constraints AdminSubdomainConstraint.new do
  namespace :admin do
    resources :users
  end
end
```

### IP Constraints

```ruby
# เฉพาะ local network
constraints ip: /127\.0\.0\.\d+/ do
  get "/debug", to: "debug#index"
end
```

### Custom Constraints

```ruby
# config/routes.rb
class AuthenticatedConstraint
  def matches?(request)
    user = User.find_by(auth_token: request.cookies["auth_token"])
    user.present?
  end
end

constraints AuthenticatedConstraint.new do
  resources :dashboard
end

# หรือ lambda
constraints lambda { |req| req.env["warden"].user.admin? } do
  resources :admin_panel
end
```

---

## Step 719: Redirect Routes

```ruby
# config/routes.rb

# Redirect ไปยัง path ใหม่
get "/old-path", to: redirect("/new-path")

# Redirect ด้วย code (default 301)
get "/old-posts", to: redirect("/posts", status: 302)

# Dynamic redirect
get "/users/:username", to: redirect { |params, req|
  "/profiles/#{params[:username]}"
}

# Redirect ด้วย method
get "/shop", to: redirect { |params, req|
  "/products?#{req.query_string}"
}

# Redirect ภายนอก
get "/google", to: redirect("https://google.com")
```

---

## Step 720: Route Helpers (_path vs _url)

### _path - Relative URL

```ruby
# ส่งคืน path เช่น /posts/1
post_path(@post)         # => "/posts/1"
posts_path               # => "/posts"
new_post_path            # => "/posts/new"
edit_post_path(@post)    # => "/posts/1/edit"
```

### _url - Absolute URL

```ruby
# ส่งคืน full URL เช่น http://example.com/posts/1
post_url(@post)          # => "http://example.com/posts/1"
posts_url                # => "http://example.com/posts"
```

### เมื่อไหร่ใช้อะไร

- ใช้ `_path` ใน views และ controllers (ส่วนใหญ่)
- ใช้ `_url` ใน mailers (email ต้องการ full URL)
- ใช้ `_url` เมื่อ redirect ข้าม domain

```ruby
# ใน mailer (ต้องใช้ _url)
class UserMailer < ApplicationMailer
  def welcome_email(user)
    @user = user
    @login_url = login_url  # http://example.com/login
    mail(to: user.email)
  end
end
```

### Helper Methods ใน Routes

```ruby
# url_for
url_for(@post)           # => "/posts/1"
url_for(controller: "posts", action: "index")  # => "/posts"

# polymorphic_path (สำหรับ polymorphic associations)
polymorphic_path(@post)  # => "/posts/1"
polymorphic_path([:admin, @post])  # => "/admin/posts/1"

# link_to ด้วย model
link_to "Show", @post    # ใช้ post_path(@post)
link_to "Edit", [:edit, @post]  # ใช้ edit_post_path(@post)
```

---

## Step 721: rails routes Command

```bash
# ดู routes ทั้งหมด
rails routes

# ดูเฉพาะ controller
rails routes -c posts
rails routes -c admin/users

# ค้นหา routes ด้วย pattern
rails routes -g post

# ดูใน format ต่างๆ
rails routes --expanded

# Output ทั้งหมดเป็น grep-friendly
rails routes | grep DELETE

# ดู routes ที่เกี่ยวกับ path เฉพาะ
rails routes -g /posts/new
```

### ตัวอย่าง Output

```
$ rails routes
   Prefix Verb   URI Pattern               Controller#Action
    posts GET    /posts(.:format)          posts#index
          POST   /posts(.:format)          posts#create
 new_post GET    /posts/new(.:format)      posts#new
edit_post GET    /posts/:id/edit(.:format) posts#edit
     post GET    /posts/:id(.:format)      posts#show
          PATCH  /posts/:id(.:format)      posts#update
          PUT    /posts/:id(.:format)      posts#update
          DELETE /posts/:id(.:format)      posts#destroy
     root GET    /                         pages#home
```

---

## Step 722: Testing Routes

### ใช้ assert_routing

```ruby
# test/routing/posts_routing_test.rb
require "test_helper"

class PostsRoutingTest < ActionDispatch::IntegrationTest
  test "routes to posts index" do
    assert_routing "/posts", controller: "posts", action: "index"
  end
  
  test "routes to post show" do
    assert_routing "/posts/1", controller: "posts", action: "show", id: "1"
  end
  
  test "routes to new post" do
    assert_routing "/posts/new", controller: "posts", action: "new"
  end
  
  test "routes POST to create" do
    assert_routing({ method: "post", path: "/posts" },
                   { controller: "posts", action: "create" })
  end
end
```

### ใช้ assert_generates

```ruby
test "generates correct path" do
  assert_generates "/posts/1", controller: "posts", action: "show", id: "1"
  assert_generates "/posts", controller: "posts", action: "index"
end
```

### ใช้ RSpec route matchers

```ruby
# spec/routing/posts_routing_spec.rb
require "rails_helper"

RSpec.describe "Posts routing", type: :routing do
  it "routes GET /posts to posts#index" do
    expect(get: "/posts").to route_to("posts#index")
  end
  
  it "routes GET /posts/:id to posts#show" do
    expect(get: "/posts/1").to route_to(
      controller: "posts",
      action: "show",
      id: "1"
    )
  end
  
  it "routes POST /posts to posts#create" do
    expect(post: "/posts").to route_to("posts#create")
  end
  
  it "routes DELETE /posts/:id to posts#destroy" do
    expect(delete: "/posts/1").to route_to(
      controller: "posts",
      action: "destroy",
      id: "1"
    )
  end
end
```

---

## ตัวอย่าง Routes ที่สมบูรณ์

### E-commerce Application

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "home#index"
  
  # Auth
  devise_for :users
  get "/profile", to: "users#profile", as: :profile
  patch "/profile", to: "users#update_profile"
  
  # Public
  resources :products, only: [:index, :show] do
    collection do
      get :search
      get :featured
      get :sale
    end
    member do
      post :add_to_wishlist
      delete :remove_from_wishlist
    end
    resources :reviews, only: [:index, :create, :destroy]
  end
  
  resources :categories, only: [:index, :show]
  
  # Shopping
  resource :cart, only: [:show, :update, :destroy] do
    post :add_item
    delete :remove_item
    patch :update_quantity
  end
  
  resources :orders, only: [:index, :show, :create] do
    member do
      post :cancel
      get :invoice
    end
  end
  
  resources :checkouts, only: [:new, :create]
  
  # API
  namespace :api do
    namespace :v1 do
      resources :products, only: [:index, :show]
      resources :orders, only: [:index, :show, :create]
      post "/auth/login", to: "auth#login"
      post "/auth/logout", to: "auth#logout"
    end
  end
  
  # Admin
  namespace :admin do
    root "dashboard#index"
    
    resources :products
    resources :orders do
      member do
        patch :ship
        patch :complete
        patch :refund
      end
      collection do
        get :pending
        get :shipped
      end
    end
    resources :users do
      member do
        post :ban
        post :unban
      end
    end
    resources :categories
    resources :reviews, only: [:index, :destroy]
  end
  
  # Static pages
  get "/about", to: "pages#about", as: :about
  get "/contact", to: "pages#contact", as: :contact
  get "/terms", to: "pages#terms", as: :terms
  get "/privacy", to: "pages#privacy", as: :privacy
  
  # Health check
  get "/health", to: proc { [200, {}, ["OK"]] }
  
  # Wildcard (ต้องอยู่ท้ายสุด)
  # get "*path", to: "errors#not_found"
end
```

---

## แบบฝึกหัด (Steps 723-730)

### แบบฝึกหัดที่ 1
สร้าง routes สำหรับ `Article` model ที่มีทุก RESTful routes

**เฉลย:**
```ruby
resources :articles
```

### แบบฝึกหัดที่ 2
สร้าง routes สำหรับ `Comment` ที่ nested ใน `Article` แต่เฉพาะ index, create, destroy

**เฉลย:**
```ruby
resources :articles do
  resources :comments, only: [:index, :create, :destroy]
end
```

### แบบฝึกหัดที่ 3
สร้าง admin namespace ที่มี `articles`, `users`, `categories`

**เฉลย:**
```ruby
namespace :admin do
  resources :articles
  resources :users
  resources :categories
end
```

### แบบฝึกหัดที่ 4
สร้าง custom route `GET /search` ที่ไปยัง `searches#index` โดยมีชื่อว่า `search`

**เฉลย:**
```ruby
get "/search", to: "searches#index", as: :search
```

### แบบฝึกหัดที่ 5
เพิ่ม member route `publish` (POST) และ `unpublish` (DELETE) ใน articles

**เฉลย:**
```ruby
resources :articles do
  member do
    post :publish
    delete :unpublish
  end
end
```

### แบบฝึกหัดที่ 6
เพิ่ม collection route `featured` (GET) และ `drafts` (GET) ใน articles

**เฉลย:**
```ruby
resources :articles do
  collection do
    get :featured
    get :drafts
  end
end
```

### แบบฝึกหัดที่ 7
สร้าง root route ที่ชี้ไปยัง `home#index`

**เฉลย:**
```ruby
root "home#index"
```

### แบบฝึกหัดที่ 8
สร้าง route constraint ที่ allow เฉพาะ integer สำหรับ `:id` parameter

**เฉลย:**
```ruby
resources :posts, constraints: { id: /\d+/ }
```

### แบบฝึกหัดที่ 9
สร้าง redirect route จาก `/old-blog` ไป `/articles`

**เฉลย:**
```ruby
get "/old-blog", to: redirect("/articles")
```

### แบบฝึกหัดที่ 10
สร้าง `login` และ `logout` routes

**เฉลย:**
```ruby
get "/login", to: "sessions#new", as: :login
post "/login", to: "sessions#create"
delete "/logout", to: "sessions#destroy", as: :logout
```

### แบบฝึกหัดที่ 11
สร้าง shallow nested routes สำหรับ posts > comments

**เฉลย:**
```ruby
resources :posts do
  resources :comments, shallow: true
end
```

### แบบฝึกหัดที่ 12
ใช้ scope module สำหรับ api namespace โดย URL ยังเป็น `/users`

**เฉลย:**
```ruby
scope module: "api" do
  resources :users
end
# URL: /users => Api::UsersController#index
```

### แบบฝึกหัดที่ 13
สร้าง API versioning ด้วย namespace

**เฉลย:**
```ruby
namespace :api do
  namespace :v1 do
    resources :users
    resources :posts
  end
  
  namespace :v2 do
    resources :users
    resources :posts
  end
end
```

### แบบฝึกหัดที่ 14
เขียน test ตรวจสอบว่า `GET /articles` routes ไปยัง `articles#index`

**เฉลย:**
```ruby
require "test_helper"

class ArticlesRoutingTest < ActionDispatch::IntegrationTest
  test "routes to articles index" do
    assert_routing "/articles", controller: "articles", action: "index"
  end
end
```

### แบบฝึกหัดที่ 15
สร้าง routes สำหรับ `singular resource` เช่น user profile (ไม่มี id)

**เฉลย:**
```ruby
resource :profile, only: [:show, :edit, :update]
# GET /profile     => profiles#show
# GET /profile/edit  => profiles#edit
# PATCH /profile   => profiles#update
```

### แบบฝึกหัดที่ 16
สร้าง route ที่ accept ทั้ง GET และ POST ด้วย match

**เฉลย:**
```ruby
match "/contact", to: "contacts#index", via: [:get, :post]
```

### แบบฝึกหัดที่ 17
สร้าง route สำหรับ health check ที่ส่งคืน 200 OK โดยไม่ใช้ controller

**เฉลย:**
```ruby
get "/health", to: proc { [200, { "Content-Type" => "text/plain" }, ["OK"]] }
```

### แบบฝึกหัดที่ 18
แสดงชื่อ helper method สำหรับ route `admin_posts_path`

**เฉลย:**
```ruby
# admin_posts_path  => /admin/posts (GET)
# admin_post_path(@post)  => /admin/posts/:id (GET)
# new_admin_post_path  => /admin/posts/new (GET)
# edit_admin_post_path(@post)  => /admin/posts/:id/edit (GET)
```

### แบบฝึกหัดที่ 19
สร้าง route สำหรับ download article PDF โดยเป็น member route

**เฉลย:**
```ruby
resources :articles do
  get :download_pdf, on: :member
end
# GET /articles/:id/download_pdf
# Named: download_pdf_article_path(@article)
```

### แบบฝึกหัดที่ 20
กำหนด route constraint ให้รับเฉพาะ request จาก subdomain "api"

**เฉลย:**
```ruby
constraints subdomain: "api" do
  namespace :api do
    resources :users
    resources :posts
  end
end
```

### แบบฝึกหัดที่ 21
ใช้ scope path เพิ่ม prefix `/v1` โดยไม่สร้าง module ใหม่

**เฉลย:**
```ruby
scope "/v1" do
  resources :users
  resources :posts
end
# /v1/users => UsersController#index (ไม่ใช่ V1::UsersController)
```

### แบบฝึกหัดที่ 22
สร้าง named route `dashboard` ที่ชี้ไปยัง `dashboards#index`

**เฉลย:**
```ruby
get "/dashboard", to: "dashboards#index", as: :dashboard
# dashboard_path => /dashboard
```

### แบบฝึกหัดที่ 23
สร้าง route สำหรับ `bulk_delete` ใน articles collection

**เฉลย:**
```ruby
resources :articles do
  delete :bulk_delete, on: :collection
end
# DELETE /articles/bulk_delete
# Named: bulk_delete_articles_path
```

### แบบฝึกหัดที่ 24
อธิบายความแตกต่างระหว่าง `namespace` และ `scope`

**เฉลย:**
- `namespace` เปลี่ยนทั้ง URL path, module path, และ named route prefix
- `scope` ให้เราควบคุมแต่ละส่วนแยกกัน
  - `scope path:` เปลี่ยน URL prefix เท่านั้น
  - `scope module:` เปลี่ยน module path เท่านั้น
  - `scope as:` เปลี่ยน named route prefix เท่านั้น

### แบบฝึกหัดที่ 25
สร้าง complete routes สำหรับ blog application ที่มี posts, categories, tags และ admin section

**เฉลย:**
```ruby
Rails.application.routes.draw do
  root "posts#index"
  
  # Public
  resources :posts, only: [:index, :show] do
    resources :comments, only: [:create, :destroy]
  end
  resources :categories, only: [:index, :show]
  resources :tags, only: [:index, :show]
  
  # Search
  get "/search", to: "searches#index", as: :search
  
  # Auth
  get "/login", to: "sessions#new", as: :login
  post "/login", to: "sessions#create"
  delete "/logout", to: "sessions#destroy", as: :logout
  
  # Admin
  namespace :admin do
    root "dashboard#index"
    resources :posts do
      member do
        post :publish
        post :unpublish
      end
    end
    resources :categories
    resources :tags
    resources :users
    resources :comments, only: [:index, :destroy]
  end
end
```

---

## สรุป

ใน Rails Routing เราได้เรียนรู้:

1. **resources** - สร้าง 7 RESTful routes อัตโนมัติ
2. **only/except** - จำกัด routes ที่สร้าง
3. **Named routes** - กำหนดชื่อให้ routes ด้วย `as:`
4. **Nested resources** - routes ที่ซ้อนกัน
5. **Shallow nesting** - ลด URL complexity
6. **namespace** - สร้าง admin section
7. **scope** - ควบคุม URL, module, named routes แยกกัน
8. **Custom routes** - get, post, put, patch, delete
9. **Member routes** - actions บน individual record
10. **Collection routes** - actions บน collection
11. **Constraints** - จำกัด routes ด้วย regex หรือ class
12. **Redirect routes** - redirect จาก URL เก่า
13. **Route helpers** - _path vs _url
