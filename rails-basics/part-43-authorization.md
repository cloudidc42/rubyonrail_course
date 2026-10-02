# Part 43: Authorization ใน Rails

## ขั้นตอนที่ 951-970: การกำหนดสิทธิ์การเข้าถึง

---

## ขั้นตอนที่ 951: Authentication vs Authorization

**Authentication (การยืนยันตัวตน):** ตรวจสอบว่า "คุณเป็นใคร?"
- Login ด้วย email/password
- JWT token verification
- OAuth callback

**Authorization (การให้สิทธิ์):** ตรวจสอบว่า "คุณทำอะไรได้บ้าง?"
- Admin สามารถลบ post ได้
- User ทั่วไปแก้ไขได้เฉพาะ post ของตัวเอง
- Guest อ่านได้อย่างเดียว

```ruby
# ตัวอย่างพื้นฐานที่ไม่ใช้ gem
class PostsController < ApplicationController
  before_action :authenticate_user!
  before_action :authorize_user!, only: [:edit, :update, :destroy]
  
  def index
    @posts = Post.all  # ทุกคนดูได้
  end
  
  def show
    @post = Post.find(params[:id])  # ทุกคนดูได้
  end
  
  def edit
    @post = Post.find(params[:id])  # เฉพาะเจ้าของ
  end
  
  def destroy
    @post = Post.find(params[:id])
    @post.destroy  # เฉพาะ admin หรือเจ้าของ
    redirect_to posts_path
  end
  
  private
  
  def authorize_user!
    @post = Post.find(params[:id])
    unless current_user.admin? || @post.user == current_user
      redirect_to root_path, alert: "ไม่มีสิทธิ์ดำเนินการนี้"
    end
  end
end
```

## ขั้นตอนที่ 952: Role-Based Access Control (RBAC)

RBAC คือการกำหนดสิทธิ์ตาม role (บทบาท) ของผู้ใช้

```ruby
# Migration สำหรับ roles
class AddRoleToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :role, :integer, default: 0, null: false
    add_index :users, :role
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # Enum สำหรับ roles
  enum role: {
    guest: 0,
    user: 1,
    moderator: 2,
    admin: 3,
    super_admin: 4
  }, _prefix: :role
  
  # Role hierarchy checks
  def can_moderate?
    role_moderator? || role_admin? || role_super_admin?
  end
  
  def can_admin?
    role_admin? || role_super_admin?
  end
  
  def higher_than?(other_user)
    roles_hierarchy.index(role) > roles_hierarchy.index(other_user.role)
  end
  
  private
  
  def roles_hierarchy
    %w[guest user moderator admin super_admin]
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # Helper methods
  helper_method :current_user, :logged_in?, :admin?
  
  private
  
  def require_login
    redirect_to login_path, alert: "กรุณา login ก่อน" unless logged_in?
  end
  
  def require_admin
    unless current_user&.can_admin?
      redirect_to root_path, alert: "ไม่มีสิทธิ์เข้าถึงส่วนนี้"
    end
  end
  
  def require_moderator
    unless current_user&.can_moderate?
      redirect_to root_path, alert: "ต้องมีสิทธิ์ Moderator ขึ้นไป"
    end
  end
  
  def admin?
    current_user&.can_admin? || false
  end
end
```

## ขั้นตอนที่ 953: Simple Authorization ด้วย before_action

```ruby
# ตัวอย่าง authorization แบบ manual
class ArticlesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  before_action :authorize_article_owner!, only: [:edit, :update]
  before_action :authorize_admin!, only: [:destroy]
  
  def index
    @articles = policy_scope(Article)
  end
  
  def show
    # ทุกคนดูได้ถ้า published
    unless @article.published? || can_manage_article?
      redirect_to articles_path, alert: "บทความนี้ยังไม่เผยแพร่"
    end
  end
  
  def create
    @article = current_user.articles.build(article_params)
    
    if @article.save
      redirect_to @article, notice: "สร้างบทความแล้ว"
    else
      render :new
    end
  end
  
  private
  
  def set_article
    @article = Article.find(params[:id])
  end
  
  def authorize_article_owner!
    unless can_manage_article?
      redirect_to @article, alert: "คุณแก้ไขได้เฉพาะบทความของตัวเอง"
    end
  end
  
  def authorize_admin!
    unless current_user.can_admin?
      redirect_to @article, alert: "เฉพาะ Admin เท่านั้น"
    end
  end
  
  def can_manage_article?
    current_user.can_admin? || @article.user == current_user
  end
end
```

## ขั้นตอนที่ 954: Pundit Gem - การติดตั้ง

Pundit เป็น gem ที่ใช้ Policy Objects สำหรับ authorization

```ruby
# Gemfile
gem 'pundit'
```

```bash
bundle install
rails generate pundit:install
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization
  
  # Error handling
  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized
  
  private
  
  def user_not_authorized(exception)
    policy_name = exception.policy.class.to_s.underscore
    
    flash[:alert] = I18n.t("pundit.#{policy_name}.#{exception.query}",
                           default: "ไม่มีสิทธิ์ดำเนินการนี้")
    redirect_back fallback_location: root_path
  end
end
```

## ขั้นตอนที่ 955: สร้าง Pundit Policy

```bash
# สร้าง policy สำหรับ Post
rails generate pundit:policy Post
```

```ruby
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  # ใครสามารถ index ได้บ้าง?
  def index?
    true  # ทุกคนดูรายการได้
  end
  
  # ใครสามารถ show ได้บ้าง?
  def show?
    record.published? || user_is_owner? || user_is_admin?
  end
  
  # ใครสามารถ create ได้บ้าง?
  def create?
    user.present?  # ต้อง login
  end
  
  # ใครสามารถ update ได้บ้าง?
  def update?
    user_is_owner? || user_is_admin?
  end
  
  # ใครสามารถ destroy ได้บ้าง?
  def destroy?
    user_is_owner? || user_is_admin?
  end
  
  # ใครสามารถ publish ได้บ้าง?
  def publish?
    user_is_owner? || user_is_admin?
  end
  
  # ใครสามารถ feature ได้บ้าง? (ปักหมุดบทความ)
  def feature?
    user_is_admin?
  end
  
  # Scope สำหรับ index
  class Scope < ApplicationPolicy::Scope
    def resolve
      if user&.can_admin?
        scope.all  # Admin เห็นทุก post
      elsif user.present?
        # User เห็น post ที่ publish + post ของตัวเอง
        scope.where(published: true).or(scope.where(user: user))
      else
        # Guest เห็นเฉพาะ published
        scope.where(published: true)
      end
    end
  end
  
  private
  
  def user_is_owner?
    user.present? && record.user == user
  end
  
  def user_is_admin?
    user&.can_admin?
  end
end
```

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
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
  
  def new?
    create?
  end
  
  def update?
    false
  end
  
  def edit?
    update?
  end
  
  def destroy?
    false
  end
  
  class Scope
    attr_reader :user, :scope
    
    def initialize(user, scope)
      @user = user
      @scope = scope
    end
    
    def resolve
      raise NotImplementedError, "#{self.class}#resolve ยังไม่ได้ implement"
    end
  end
end
```

## ขั้นตอนที่ 956: ใช้ Pundit ใน Controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy, :publish]
  after_action :verify_authorized, except: [:index]
  after_action :verify_policy_scoped, only: :index
  
  def index
    @posts = policy_scope(Post)
                .order(created_at: :desc)
                .page(params[:page])
  end
  
  def show
    authorize @post
  end
  
  def new
    @post = Post.new
    authorize @post
  end
  
  def create
    @post = current_user.posts.build(post_params)
    authorize @post
    
    if @post.save
      redirect_to @post, notice: "สร้างบทความสำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  def edit
    authorize @post
  end
  
  def update
    authorize @post
    
    if @post.update(post_params)
      redirect_to @post, notice: "อัพเดทบทความแล้ว"
    else
      render :edit, status: :unprocessable_entity
    end
  end
  
  def destroy
    authorize @post
    @post.destroy
    redirect_to posts_path, notice: "ลบบทความแล้ว"
  end
  
  def publish
    authorize @post
    
    if @post.publish!
      redirect_to @post, notice: "เผยแพร่บทความแล้ว"
    else
      redirect_to @post, alert: "ไม่สามารถเผยแพร่ได้"
    end
  end
  
  private
  
  def set_post
    @post = Post.find(params[:id])
  end
  
  def post_params
    params.require(:post).permit(:title, :content, :published, :category_id)
  end
end
```

## ขั้นตอนที่ 957: Pundit ใน Views

```erb
<%# app/views/posts/show.html.erb %>
<article>
  <h1><%= @post.title %></h1>
  <p><%= @post.content %></p>
  
  <%# แสดงปุ่มเฉพาะผู้ที่มีสิทธิ์ %>
  <div class="actions">
    <% if policy(@post).update? %>
      <%= link_to "แก้ไข", edit_post_path(@post), class: "btn btn-primary" %>
    <% end %>
    
    <% if policy(@post).destroy? %>
      <%= link_to "ลบ", post_path(@post), 
          method: :delete,
          data: { confirm: "แน่ใจหรือไม่?" },
          class: "btn btn-danger" %>
    <% end %>
    
    <% if policy(@post).publish? && !@post.published? %>
      <%= button_to "เผยแพร่", publish_post_path(@post), 
          class: "btn btn-success" %>
    <% end %>
    
    <% if policy(@post).feature? %>
      <%= button_to "ปักหมุด", feature_post_path(@post),
          class: "btn btn-warning" %>
    <% end %>
  </div>
</article>
```

```erb
<%# app/views/posts/index.html.erb %>
<h1>บทความทั้งหมด</h1>

<% if policy(Post).create? %>
  <%= link_to "เขียนบทความใหม่", new_post_path, class: "btn btn-primary mb-3" %>
<% end %>

<% @posts.each do |post| %>
  <div class="card mb-3">
    <div class="card-body">
      <h5><%= post.title %></h5>
      <p><%= truncate(post.content, length: 150) %></p>
      
      <div class="d-flex gap-2">
        <%= link_to "อ่านต่อ", post %>
        
        <% if policy(post).update? %>
          <%= link_to "แก้ไข", edit_post_path(post) %>
        <% end %>
        
        <% if policy(post).destroy? %>
          <%= link_to "ลบ", post, method: :delete, 
              data: { confirm: "ลบบทความนี้?" } %>
        <% end %>
      </div>
    </div>
  </div>
<% end %>
```

## ขั้นตอนที่ 958: Pundit Scopes

```ruby
# app/policies/comment_policy.rb
class CommentPolicy < ApplicationPolicy
  class Scope < ApplicationPolicy::Scope
    def resolve
      if user&.can_admin?
        # Admin เห็นทุก comment รวม spam
        scope.all
      elsif user&.can_moderate?
        # Moderator เห็น comment ที่ approved + pending
        scope.where(status: [:approved, :pending])
      elsif user.present?
        # User เห็น approved comments + comment ของตัวเอง
        scope.approved.or(scope.where(user: user))
      else
        # Guest เห็นเฉพาะ approved
        scope.approved
      end
    end
  end
  
  def create?
    user.present?
  end
  
  def update?
    user_is_owner? && !record.too_old_to_edit?
  end
  
  def destroy?
    user_is_owner? || user&.can_moderate?
  end
  
  def approve?
    user&.can_moderate?
  end
  
  def spam?
    user&.can_moderate?
  end
  
  private
  
  def user_is_owner?
    user == record.user
  end
end
```

## ขั้นตอนที่ 959: Pundit สำหรับ Resource ที่ซับซ้อน

```ruby
# Policy สำหรับ nested resources
class OrderPolicy < ApplicationPolicy
  def index?
    user.present?
  end
  
  def show?
    user_is_owner? || user_is_admin?
  end
  
  def create?
    user.present? && !user.banned?
  end
  
  def update?
    # เฉพาะ admin และ order ที่ยังไม่ ship
    user_is_admin? && record.pending?
  end
  
  def cancel?
    # เจ้าของ cancel ได้ถ้า pending, admin cancel ได้ทุกสถานะ
    (user_is_owner? && record.cancellable?) || user_is_admin?
  end
  
  def refund?
    # เฉพาะ admin
    user_is_admin?
  end
  
  # Permitted attributes ตาม role
  def permitted_attributes
    if user_is_admin?
      [:status, :admin_notes, :refund_amount]
    else
      [:shipping_address, :billing_address]
    end
  end
  
  class Scope < ApplicationPolicy::Scope
    def resolve
      if user.can_admin?
        scope.all
      else
        scope.where(user: user)
      end
    end
  end
  
  private
  
  def user_is_owner?
    record.user == user
  end
  
  def user_is_admin?
    user&.can_admin?
  end
end
```

```ruby
# ใช้ permitted_attributes ใน controller
class OrdersController < ApplicationController
  def update
    @order = Order.find(params[:id])
    authorize @order
    
    # permitted_attributes จะกรองตาม role อัตโนมัติ
    if @order.update(permitted_attributes(@order))
      redirect_to @order
    else
      render :edit
    end
  end
end
```

## ขั้นตอนที่ 960: CanCanCan Gem

CanCanCan เป็นอีกทางเลือกสำหรับ authorization โดยใช้ Ability class

```ruby
# Gemfile
gem 'cancancan'
```

```bash
bundle install
rails generate cancan:ability
```

```ruby
# app/models/ability.rb
class Ability
  include CanCan::Ability
  
  def initialize(user)
    user ||= User.new  # guest user
    
    # กำหนดสิทธิ์ตาม role
    if user.super_admin?
      can :manage, :all
    elsif user.admin?
      can :manage, :all
      cannot :manage, User, role: 'super_admin'
    elsif user.moderator?
      define_moderator_abilities(user)
    elsif user.user?
      define_user_abilities(user)
    else
      define_guest_abilities
    end
  end
  
  private
  
  def define_guest_abilities
    can :read, Post, published: true
    can :read, Comment, status: :approved
    can :read, User
  end
  
  def define_user_abilities(user)
    # สิทธิ์ guest
    define_guest_abilities
    
    # สิทธิ์เพิ่มเติม
    can :create, Post
    can :create, Comment
    
    # จัดการของตัวเอง
    can [:update, :destroy], Post, user_id: user.id
    can [:update, :destroy], Comment, user_id: user.id
    can [:update], User, id: user.id
    
    # เงื่อนไขซับซ้อน
    can :publish, Post do |post|
      post.user_id == user.id && post.ready_to_publish?
    end
  end
  
  def define_moderator_abilities(user)
    define_user_abilities(user)
    
    # Moderator สิทธิ์เพิ่มเติม
    can :read, Post  # เห็นทุก post รวม draft
    can [:approve, :reject, :spam], Comment
    can :ban, User do |target_user|
      !target_user.admin? && !target_user.moderator?
    end
  end
end
```

## ขั้นตอนที่ 961: ใช้ CanCanCan ใน Controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  load_and_authorize_resource  # อ่านและ authorize resource อัตโนมัติ
  
  def index
    # @posts ถูกกำหนดและ filter โดย load_and_authorize_resource
  end
  
  def show
    # @post ถูก authorize แล้ว
  end
  
  def new
    # @post = Post.new (สร้างให้อัตโนมัติ)
  end
  
  def create
    # @post มีอยู่แล้ว และ authorize แล้ว
    @post.user = current_user
    
    if @post.save
      redirect_to @post
    else
      render :new
    end
  end
  
  def edit; end
  
  def update
    if @post.update(post_params)
      redirect_to @post
    else
      render :edit
    end
  end
  
  def destroy
    @post.destroy
    redirect_to posts_path
  end
  
  private
  
  def post_params
    params.require(:post).permit(:title, :content)
  end
end
```

```ruby
# Error handling สำหรับ CanCanCan
class ApplicationController < ActionController::Base
  rescue_from CanCan::AccessDenied do |exception|
    respond_to do |format|
      format.json { render json: { error: "ไม่มีสิทธิ์" }, status: :forbidden }
      format.html { 
        flash[:alert] = exception.message
        redirect_to root_path
      }
    end
  end
end
```

## ขั้นตอนที่ 962: CanCanCan ใน Views

```erb
<%# ใช้ can? helper %>
<% if can? :update, @post %>
  <%= link_to "แก้ไข", edit_post_path(@post) %>
<% end %>

<% if can? :destroy, @post %>
  <%= link_to "ลบ", @post, method: :delete %>
<% end %>

<%# ตรวจสอบ action กับ class %>
<% if can? :create, Post %>
  <%= link_to "เขียนบทความใหม่", new_post_path %>
<% end %>

<%# cannot? helper %>
<% if cannot? :manage, @post %>
  <p>คุณสามารถอ่านได้อย่างเดียว</p>
<% end %>
```

## ขั้นตอนที่ 963: Admin Panel ด้วย Pundit

```ruby
# app/policies/admin/base_policy.rb
module Admin
  class BasePolicy < ApplicationPolicy
    def initialize(user, record)
      raise Pundit::NotAuthorizedError, "ต้องเป็น Admin" unless user&.can_admin?
      super
    end
  end
end
```

```ruby
# app/policies/admin/user_policy.rb
module Admin
  class UserPolicy < Admin::BasePolicy
    def index?
      true
    end
    
    def show?
      true
    end
    
    def update?
      # Super admin แก้ทุกคนได้ ยกเว้น Super admin ด้วยกัน
      if record.super_admin?
        user.super_admin?
      else
        true
      end
    end
    
    def destroy?
      # ไม่ให้ลบตัวเอง
      record != user && !record.super_admin?
    end
    
    def impersonate?
      user.super_admin? && record != user
    end
    
    def ban?
      !record.admin? && record != user
    end
  end
end
```

```ruby
# app/controllers/admin/users_controller.rb
class Admin::UsersController < ApplicationController
  before_action :authenticate_user!
  before_action :set_user, only: [:show, :edit, :update, :destroy, :ban]
  after_action :verify_authorized
  
  def index
    authorize [:admin, User]
    @users = policy_scope([:admin, User])
              .order(created_at: :desc)
              .page(params[:page])
  end
  
  def show
    authorize [:admin, @user]
  end
  
  def edit
    authorize [:admin, @user]
  end
  
  def update
    authorize [:admin, @user]
    
    if @user.update(user_params)
      redirect_to admin_user_path(@user), notice: "อัพเดทผู้ใช้แล้ว"
    else
      render :edit
    end
  end
  
  def destroy
    authorize [:admin, @user]
    @user.destroy
    redirect_to admin_users_path, notice: "ลบผู้ใช้แล้ว"
  end
  
  def ban
    authorize [:admin, @user]
    @user.update!(banned_at: Time.current, banned_reason: params[:reason])
    redirect_to admin_user_path(@user), notice: "แบนผู้ใช้แล้ว"
  end
  
  private
  
  def set_user
    @user = User.find(params[:id])
  end
  
  def user_params
    params.require(:user).permit(:name, :email, :role, :active)
  end
end
```

## ขั้นตอนที่ 964: Resource-based Policies

```ruby
# Policy สำหรับ Document ที่มี permissions หลายระดับ
class DocumentPolicy < ApplicationPolicy
  def index?
    true
  end
  
  def show?
    public_document? || has_access? || user_is_owner? || user_is_admin?
  end
  
  def create?
    user.present?
  end
  
  def update?
    has_write_access? || user_is_owner? || user_is_admin?
  end
  
  def destroy?
    user_is_owner? || user_is_admin?
  end
  
  def share?
    user_is_owner? || user_is_admin?
  end
  
  def download?
    show? # ถ้าดูได้ก็ download ได้
  end
  
  class Scope < ApplicationPolicy::Scope
    def resolve
      if user&.can_admin?
        scope.all
      elsif user.present?
        # Document ที่เป็นของตัวเอง + public + shared กับตัวเอง
        scope.where(user: user)
             .or(scope.public_access)
             .or(scope.shared_with(user))
      else
        scope.public_access
      end
    end
  end
  
  private
  
  def public_document?
    record.public_access?
  end
  
  def has_access?
    user.present? && record.shared_with?(user)
  end
  
  def has_write_access?
    user.present? && record.has_write_access?(user)
  end
  
  def user_is_owner?
    user.present? && record.user == user
  end
  
  def user_is_admin?
    user&.can_admin?
  end
end
```

## ขั้นตอนที่ 965: Testing Authorization

```ruby
# spec/policies/post_policy_spec.rb
require 'rails_helper'

RSpec.describe PostPolicy, type: :policy do
  subject { described_class }
  
  let(:admin) { create(:user, :admin) }
  let(:owner) { create(:user) }
  let(:other_user) { create(:user) }
  let(:guest) { nil }
  
  let(:published_post) { create(:post, user: owner, published: true) }
  let(:draft_post) { create(:post, user: owner, published: false) }
  
  describe "index?" do
    it { is_expected.to permit(admin, Post) }
    it { is_expected.to permit(owner, Post) }
    it { is_expected.to permit(other_user, Post) }
    it { is_expected.to permit(guest, Post) }
  end
  
  describe "show?" do
    context "published post" do
      it { is_expected.to permit(admin, published_post) }
      it { is_expected.to permit(owner, published_post) }
      it { is_expected.to permit(other_user, published_post) }
      it { is_expected.to permit(guest, published_post) }
    end
    
    context "draft post" do
      it { is_expected.to permit(admin, draft_post) }
      it { is_expected.to permit(owner, draft_post) }
      it { is_expected.not_to permit(other_user, draft_post) }
      it { is_expected.not_to permit(guest, draft_post) }
    end
  end
  
  describe "create?" do
    it { is_expected.to permit(admin, Post) }
    it { is_expected.to permit(owner, Post) }
    it { is_expected.not_to permit(guest, Post) }
  end
  
  describe "update?" do
    it { is_expected.to permit(admin, published_post) }
    it { is_expected.to permit(owner, published_post) }
    it { is_expected.not_to permit(other_user, published_post) }
    it { is_expected.not_to permit(guest, published_post) }
  end
  
  describe "destroy?" do
    it { is_expected.to permit(admin, published_post) }
    it { is_expected.to permit(owner, published_post) }
    it { is_expected.not_to permit(other_user, published_post) }
    it { is_expected.not_to permit(guest, published_post) }
  end
  
  describe "PostPolicy::Scope" do
    let!(:published) { create_list(:post, 3, published: true) }
    let!(:drafts) { create_list(:post, 2, published: false, user: owner) }
    
    subject { PostPolicy::Scope.new(user, Post).resolve }
    
    context "admin" do
      let(:user) { admin }
      it "เห็นทุก post" do
        expect(subject).to include(*published, *drafts)
      end
    end
    
    context "owner" do
      let(:user) { owner }
      it "เห็น published + own drafts" do
        expect(subject).to include(*published, *drafts)
      end
    end
    
    context "other user" do
      let(:user) { other_user }
      it "เห็นเฉพาะ published" do
        expect(subject).to include(*published)
        expect(subject).not_to include(*drafts)
      end
    end
    
    context "guest" do
      let(:user) { nil }
      it "เห็นเฉพาะ published" do
        expect(subject).to include(*published)
        expect(subject).not_to include(*drafts)
      end
    end
  end
end
```

## ขั้นตอนที่ 966: Authorization Helpers

```ruby
# app/helpers/authorization_helper.rb
module AuthorizationHelper
  # แสดง content เฉพาะถ้ามีสิทธิ์
  def authorized_content(policy_subject, action, &block)
    if policy(policy_subject).public_send("#{action}?")
      capture(&block)
    end
  end
  
  # สร้าง link เฉพาะถ้ามีสิทธิ์
  def authorized_link_to(text, path, policy_subject, action, **options)
    if policy(policy_subject).public_send("#{action}?")
      link_to text, path, **options
    end
  end
end
```

```erb
<%# ใช้งาน helper %>
<%= authorized_content @post, :update do %>
  <%= link_to "แก้ไข", edit_post_path(@post) %>
<% end %>

<%= authorized_link_to "ลบ", post_path(@post), @post, :destroy,
    method: :delete, class: "btn btn-danger" %>
```

## ขั้นตอนที่ 967: Custom Roles ด้วย Database

```ruby
# สำหรับ application ที่ต้องการ dynamic roles
class CreateRoles < ActiveRecord::Migration[7.0]
  def change
    create_table :roles do |t|
      t.string :name, null: false
      t.text :description
      t.timestamps
    end
    
    create_table :role_assignments do |t|
      t.references :user, null: false, foreign_key: true
      t.references :role, null: false, foreign_key: true
      t.references :resource, polymorphic: true  # สำหรับ scoped roles
      t.datetime :expires_at
      t.timestamps
    end
    
    create_table :permissions do |t|
      t.references :role, null: false, foreign_key: true
      t.string :subject_class, null: false
      t.string :action, null: false
      t.timestamps
    end
    
    add_index :roles, :name, unique: true
    add_index :role_assignments, [:user_id, :role_id], unique: true
    add_index :permissions, [:role_id, :subject_class, :action], unique: true
  end
end
```

```ruby
# app/models/role.rb
class Role < ApplicationRecord
  has_many :role_assignments, dependent: :destroy
  has_many :users, through: :role_assignments
  has_many :permissions, dependent: :destroy
  
  validates :name, presence: true, uniqueness: true
  
  # Pre-defined roles
  ROLES = %w[guest user moderator admin super_admin].freeze
  
  def can?(action, subject_class)
    permissions.exists?(action: action, subject_class: subject_class.to_s)
  end
  
  def grant_permission(action, subject_class)
    permissions.find_or_create_by!(
      action: action.to_s,
      subject_class: subject_class.to_s
    )
  end
  
  def revoke_permission(action, subject_class)
    permissions.where(
      action: action.to_s,
      subject_class: subject_class.to_s
    ).delete_all
  end
end
```

## ขั้นตอนที่ 968: Authorization Logging

```ruby
# app/services/authorization_audit_service.rb
class AuthorizationAuditService
  def self.log(user:, action:, resource:, allowed:, reason: nil)
    AuthorizationLog.create!(
      user: user,
      action: action,
      resource_type: resource.class.name,
      resource_id: resource.id,
      allowed: allowed,
      reason: reason,
      ip_address: Current.request_ip,
      user_agent: Current.request_user_agent
    )
  end
end

# app/policies/application_policy.rb
class ApplicationPolicy
  def authorize_with_logging(action_name)
    result = yield
    
    AuthorizationAuditService.log(
      user: user,
      action: action_name,
      resource: record,
      allowed: result
    )
    
    result
  end
end
```

## ขั้นตอนที่ 969: Authorization ในระดับ API

```ruby
# app/controllers/api/v1/base_controller.rb
class Api::V1::BaseController < ActionController::API
  include Pundit::Authorization
  
  before_action :authenticate_request!
  
  rescue_from Pundit::NotAuthorizedError do |exception|
    render json: { 
      error: "ไม่มีสิทธิ์",
      message: "คุณไม่มีสิทธิ์ดำเนินการ #{exception.query} กับ #{exception.record.class}"
    }, status: :forbidden
  end
  
  private
  
  def authenticate_request!
    token = request.headers['Authorization']&.split(' ')&.last
    
    if token
      decoded = JwtService.decode(token)
      @current_user = User.find_by(id: decoded[:user_id]) if decoded
    end
    
    render json: { error: "ต้อง authenticate ก่อน" }, status: :unauthorized unless @current_user
  end
  
  def current_user
    @current_user
  end
  
  def pundit_user
    current_user
  end
end
```

## ขั้นตอนที่ 970: Best Practices สำหรับ Authorization

```ruby
# 1. Fail Secure - ปฏิเสธก่อน อนุญาตทีหลัง
class ApplicationPolicy
  def initialize(user, record)
    @user = user
    @record = record
  end
  
  # Default ปฏิเสธทุกอย่าง
  def index?   = false
  def show?    = false
  def create?  = false
  def update?  = false
  def destroy? = false
end

# 2. ใช้ verify_authorized ใน after_action
class PostsController < ApplicationController
  after_action :verify_authorized
  after_action :verify_policy_scoped, only: :index
end

# 3. Policy Namespace สำหรับ admin
# app/policies/admin/post_policy.rb
module Admin
  class PostPolicy < ApplicationPolicy
    def index? = user&.can_admin?
    def show?  = user&.can_admin?
    def update? = user&.can_admin?
    def destroy? = user&.can_admin?
  end
end

# 4. Memoize current_user
class ApplicationController < ActionController::Base
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
end

# 5. ใช้ scopes เสมอในการ query
class PostsController < ApplicationController
  def index
    @posts = policy_scope(Post)  # ไม่ใช้ Post.all โดยตรง
  end
end
```

---

## แบบฝึกหัด: Authorization (20 ข้อ)

### ข้อที่ 1: สร้าง Basic Role System
```
สร้าง User model ที่มี roles: guest, user, moderator, admin
ใช้ enum และเพิ่ม helper methods สำหรับตรวจสอบ role
```

**เฉลย:**
```ruby
# migration
add_column :users, :role, :integer, default: 1

# user.rb
enum role: { guest: 0, user: 1, moderator: 2, admin: 3 }

def can_moderate? = moderator? || admin?
def can_admin? = admin?
```

### ข้อที่ 2: Pundit Policy พื้นฐาน
```
สร้าง PostPolicy ที่:
- ทุกคนอ่านได้ (published เท่านั้น)
- ต้อง login ถึงจะ create ได้
- แก้ไข/ลบได้เฉพาะเจ้าของหรือ admin
```

### ข้อที่ 3: Policy Scope
```
เขียน PostPolicy::Scope ที่:
- Admin เห็นทุก post
- User เห็น published + post ตัวเอง
- Guest เห็นเฉพาะ published
```

### ข้อที่ 4: CanCanCan Ability
```
สร้าง Ability class ด้วย CanCanCan
กำหนดสิทธิ์ที่ต่างกันสำหรับ guest, user, admin
```

### ข้อที่ 5: Admin Panel Protection
```
สร้าง Admin namespace controller
ป้องกันเฉพาะ admin เข้าถึงได้
```

**เฉลย:**
```ruby
class Admin::BaseController < ApplicationController
  before_action :authenticate_user!
  before_action :require_admin!
  
  private
  def require_admin!
    redirect_to root_path unless current_user.can_admin?
  end
end
```

### ข้อที่ 6: View Authorization
```
แสดงปุ่ม Edit/Delete เฉพาะผู้ที่มีสิทธิ์
ใช้ policy helper ใน view
```

### ข้อที่ 7: Resource-specific Roles
```
สร้าง ProjectMember model ที่มี role (owner, editor, viewer)
ใช้ Pundit policy ตรวจสอบสิทธิ์ตาม membership role
```

### ข้อที่ 8: Policy Testing
```
เขียน RSpec tests สำหรับ PostPolicy
ครอบคลุมทุก action และทุก role
```

### ข้อที่ 9: Nested Resource Authorization
```
สร้าง CommentPolicy สำหรับ comments ที่เป็น nested resource ของ post
```

### ข้อที่ 10: API Authorization
```
เพิ่ม Pundit authorization ใน API controller
ส่ง JSON error response เมื่อไม่มีสิทธิ์
```

### ข้อที่ 11-20 (แบบสรุป)

**ข้อ 11:** สร้าง Dynamic Permissions system ด้วย database roles

**ข้อ 12:** เพิ่ม Permission checking ใน background job

**ข้อ 13:** สร้าง authorization audit log

**ข้อ 14:** Implement "own organization" scoping (multi-tenant)

**ข้อ 15:** สร้าง delegation system (user มอบสิทธิ์ให้ user อื่น)

**ข้อ 16:** ทดสอบ authorization ด้วย integration tests

**ข้อ 17:** เพิ่ม time-limited permissions (สิทธิ์ชั่วคราว)

**ข้อ 18:** สร้าง feature flags system

**ข้อ 19:** Implement row-level security ด้วย PostgreSQL

**ข้อ 20:** สร้าง permission inheritance (role hierarchy)

```ruby
# ข้อ 15: Permission Delegation
class PermissionDelegation < ApplicationRecord
  belongs_to :delegator, class_name: "User"
  belongs_to :delegate, class_name: "User"
  belongs_to :resource, polymorphic: true
  
  validates :action, presence: true
  validates :expires_at, presence: true
  validate :delegator_has_permission
  
  scope :active, -> { where("expires_at > ?", Time.current) }
  scope :for_user, ->(user) { where(delegate: user) }
  
  private
  
  def delegator_has_permission
    policy = policy_for_resource
    unless policy.send("#{action}?")
      errors.add(:base, "Delegator ไม่มีสิทธิ์นี้")
    end
  end
end
```

---

## สรุป: Authorization

| Approach | ข้อดี | ข้อเสีย |
|----------|-------|--------|
| Manual (before_action) | ง่าย เข้าใจง่าย | ไม่ structured, repeat code |
| Pundit | Policy objects, testable | Setup ค่อนข้างมาก |
| CanCanCan | Ability class, ง่าย | ยากเมื่อซับซ้อน |
| Role-based | ชัดเจน | ไม่ flexible |
| Dynamic permissions | Flexible มาก | ซับซ้อน |

**Key Takeaways:**
1. Authorization ต่างจาก Authentication - ต้องทำหลัง authenticate แล้ว
2. Fail Secure - deny by default
3. ใช้ Policy Objects (Pundit) สำหรับ application ขนาดกลาง-ใหญ่
4. เขียน tests ให้ครอบคลุม authorization policies
5. ใช้ Scopes เพื่อกรอง data ตาม permission
6. Log authorization decisions สำหรับ audit trail
