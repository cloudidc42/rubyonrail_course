# Project 3: SaaS Application (Full Stack)

## ระบบ Multi-tenant SaaS สมบูรณ์

---

## ภาพรวม

เราจะสร้าง **ProjectHub** - ระบบ Project Management แบบ SaaS ที่มี:
- Multi-tenant (หลาย Organizations)
- Team members และ Roles
- Subscription billing ด้วย Stripe
- Feature flags ตาม Plan
- Admin Dashboard
- REST API พร้อม JWT Authentication
- Onboarding Flow

---

## โครงสร้าง Project

```
projecthub/
├── app/
│   ├── controllers/
│   │   ├── api/
│   │   │   └── v1/
│   │   ├── admin/
│   │   └── onboarding/
│   ├── models/
│   ├── services/
│   ├── jobs/
│   ├── mailers/
│   └── views/
├── config/
├── db/
└── spec/
```

---

## ขั้นตอนที่ 1: Setup Project

```bash
rails new projecthub \
  --database=postgresql \
  --css=tailwind \
  --javascript=importmap \
  --skip-test

cd projecthub

# เพิ่ม gems
bundle add devise
bundle add pundit
bundle add stripe
bundle add pay   # Stripe billing
bundle add flipper
bundle add flipper-active_record
bundle add kaminari
bundle add jwt
bundle add rack-cors
bundle add sidekiq
bundle add redis
bundle add pagy
```

---

## ขั้นตอนที่ 2: Database Schema

```ruby
# db/schema.rb สมบูรณ์

# Organizations (Tenants)
create_table :organizations do |t|
  t.string  :name,       null: false
  t.string  :slug,       null: false, index: { unique: true }
  t.string  :plan,       null: false, default: 'free'
  t.string  :status,     null: false, default: 'active'
  t.string  :logo_url
  t.jsonb   :settings,   default: {}
  t.timestamps
end

# Users
create_table :users do |t|
  t.string  :email,               null: false, index: { unique: true }
  t.string  :encrypted_password,  null: false
  t.string  :first_name,          null: false
  t.string  :last_name,           null: false
  t.string  :avatar_url
  t.string  :time_zone,           default: 'Bangkok'
  t.string  :locale,              default: 'th'
  t.datetime :last_sign_in_at
  t.string   :reset_password_token, index: { unique: true }
  t.datetime :reset_password_sent_at
  t.timestamps
end

# Organization Memberships
create_table :memberships do |t|
  t.references :organization, null: false, foreign_key: true
  t.references :user,         null: false, foreign_key: true
  t.string     :role,         null: false, default: 'member'
  t.string     :status,       null: false, default: 'active'
  t.datetime   :invited_at
  t.datetime   :accepted_at
  t.string     :invitation_token, index: { unique: true }
  t.timestamps

  add_index :memberships, [:organization_id, :user_id], unique: true
end

# Projects
create_table :projects do |t|
  t.references :organization, null: false, foreign_key: true
  t.references :created_by,   null: false, foreign_key: { to_table: :users }
  t.string  :name,            null: false
  t.text    :description
  t.string  :status,          null: false, default: 'active'
  t.date    :start_date
  t.date    :end_date
  t.jsonb   :settings,        default: {}
  t.timestamps

  add_index :projects, [:organization_id, :name]
end

# Tasks
create_table :tasks do |t|
  t.references :project,     null: false, foreign_key: true
  t.references :assigned_to, null: true,  foreign_key: { to_table: :users }
  t.references :created_by,  null: false, foreign_key: { to_table: :users }
  t.string  :title,          null: false
  t.text    :description
  t.string  :status,         null: false, default: 'todo'
  t.string  :priority,       null: false, default: 'medium'
  t.date    :due_date
  t.integer :position,       null: false, default: 0
  t.timestamps
end

# Subscriptions (ด้วย pay gem)
create_table :pay_customers do |t|
  t.belongs_to :owner, polymorphic: true, index: false
  t.string     :processor,            null: false
  t.string     :processor_id
  t.boolean    :default
  t.public_send(:jsonb, :data)
  t.datetime   :deleted_at
  t.timestamps null: false
end

create_table :pay_subscriptions do |t|
  t.belongs_to :customer, null: false
  t.string     :name,             null: false
  t.string     :processor,        null: false
  t.string     :processor_id,     null: false, index: { unique: true }
  t.string     :processor_plan,   null: false
  t.integer    :quantity,         null: false, default: 1
  t.string     :status,           null: false
  t.datetime   :current_period_start
  t.datetime   :current_period_end
  t.boolean    :metered
  t.string     :pause_behavior
  t.datetime   :pause_starts_at
  t.datetime   :pause_resumes_at
  t.decimal    :application_fee_percent, precision: 8, scale: 2
  t.datetime   :trial_ends_at
  t.datetime   :ends_at
  t.public_send(:jsonb, :data)
  t.timestamps null: false
end
```

---

## ขั้นตอนที่ 3: Models

### Organization Model

```ruby
# app/models/organization.rb
class Organization < ApplicationRecord
  include Pay::Billable

  PLANS = {
    'free' => {
      name:          'Free',
      price:         0,
      max_members:   5,
      max_projects:  3,
      max_tasks:     100,
      features:      []
    },
    'starter' => {
      name:          'Starter',
      price:         999,   # บาท/เดือน
      max_members:   10,
      max_projects:  10,
      max_tasks:     1000,
      features:      ['api_access', 'advanced_reports']
    },
    'professional' => {
      name:          'Professional',
      price:         2999,
      max_members:   50,
      max_projects:  -1,   # unlimited
      max_tasks:     -1,
      features:      ['api_access', 'advanced_reports', 'custom_domain', 'sso']
    },
    'enterprise' => {
      name:          'Enterprise',
      price:         nil,  # ติดต่อ
      max_members:   -1,
      max_projects:  -1,
      max_tasks:     -1,
      features:      ['api_access', 'advanced_reports', 'custom_domain', 'sso',
                      'dedicated_support', 'audit_logs', 'saml']
    }
  }.freeze

  has_many :memberships,  dependent: :destroy
  has_many :members, through: :memberships, source: :user
  has_many :projects, dependent: :destroy
  has_many :invitations, dependent: :destroy

  validates :name, presence: true, length: { minimum: 2, maximum: 100 }
  validates :slug, presence: true, uniqueness: true,
                   format: { with: /\A[a-z0-9-]+\z/ }
  validates :plan, inclusion: { in: PLANS.keys }

  before_validation :generate_slug, on: :create

  def plan_config
    PLANS[plan] || PLANS['free']
  end

  def feature_enabled?(feature)
    plan_config[:features].include?(feature.to_s)
  end

  def can_add_member?
    max = plan_config[:max_members]
    max == -1 || memberships.active.count < max
  end

  def can_create_project?
    max = plan_config[:max_projects]
    max == -1 || projects.active.count < max
  end

  def upgrade_plan!(new_plan)
    raise ArgumentError, "Invalid plan: #{new_plan}" unless PLANS.key?(new_plan)

    update!(plan: new_plan)
    OrganizationMailer.plan_upgraded(self, new_plan).deliver_later
  end

  def owner
    memberships.find_by(role: 'owner')&.user
  end

  def member?(user)
    memberships.active.exists?(user: user)
  end

  def admin?(user)
    memberships.active.exists?(user: user, role: ['owner', 'admin'])
  end

  private

  def generate_slug
    return if slug.present?
    base_slug = name.to_s.parameterize
    self.slug = base_slug
    counter = 1
    while Organization.exists?(slug: self.slug)
      self.slug = "#{base_slug}-#{counter}"
      counter += 1
    end
  end
end
```

### User Model

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :trackable

  has_many :memberships,    dependent: :destroy
  has_many :organizations,  through: :memberships
  has_many :assigned_tasks, class_name: 'Task', foreign_key: :assigned_to_id
  has_many :created_tasks,  class_name: 'Task', foreign_key: :created_by_id

  validates :first_name, presence: true, length: { maximum: 50 }
  validates :last_name,  presence: true, length: { maximum: 50 }

  has_one_attached :avatar

  def full_name
    "#{first_name} #{last_name}"
  end

  def initials
    "#{first_name[0]}#{last_name[0]}".upcase
  end

  def current_organization
    @current_organization ||= organizations.first
  end

  def role_in(organization)
    memberships.find_by(organization: organization)&.role
  end

  def owner_of?(organization)
    role_in(organization) == 'owner'
  end

  def admin_of?(organization)
    role_in(organization).in?(%w[owner admin])
  end

  def generate_jwt
    payload = {
      user_id:    id,
      email:      email,
      exp:        24.hours.from_now.to_i,
      iat:        Time.current.to_i
    }

    JWT.encode(payload, Rails.application.credentials.secret_key_base, 'HS256')
  end

  def self.from_jwt(token)
    payload = JWT.decode(
      token,
      Rails.application.credentials.secret_key_base,
      true,
      { algorithm: 'HS256' }
    ).first

    find(payload['user_id'])
  rescue JWT::DecodeError, ActiveRecord::RecordNotFound
    nil
  end
end
```

### Membership Model

```ruby
# app/models/membership.rb
class Membership < ApplicationRecord
  ROLES = %w[owner admin member viewer].freeze

  belongs_to :organization
  belongs_to :user

  validates :role, inclusion: { in: ROLES }
  validates :user_id, uniqueness: { scope: :organization_id }

  scope :active,     -> { where(status: 'active') }
  scope :pending,    -> { where(status: 'pending') }
  scope :owners,     -> { where(role: 'owner') }
  scope :admins,     -> { where(role: ['owner', 'admin']) }

  def accept!(user)
    raise "Wrong user" unless self.user_id == user.id || invitation_token.present?

    update!(
      user:             user,
      status:           'active',
      accepted_at:      Time.current,
      invitation_token: nil
    )
  end

  def permissions
    case role
    when 'owner'  then Permission::OWNER_PERMISSIONS
    when 'admin'  then Permission::ADMIN_PERMISSIONS
    when 'member' then Permission::MEMBER_PERMISSIONS
    when 'viewer' then Permission::VIEWER_PERMISSIONS
    else []
    end
  end
end
```

### Project Model

```ruby
# app/models/project.rb
class Project < ApplicationRecord
  belongs_to :organization
  belongs_to :created_by, class_name: 'User'
  has_many   :tasks, dependent: :destroy
  has_many   :project_members, dependent: :destroy
  has_many   :members, through: :project_members, source: :user

  STATUSES   = %w[active archived completed].freeze
  validates :name,   presence: true, length: { minimum: 2, maximum: 100 }
  validates :status, inclusion: { in: STATUSES }

  scope :active,   -> { where(status: 'active') }
  scope :recent,   -> { order(updated_at: :desc) }

  def completion_percentage
    return 0 if tasks.count.zero?
    (tasks.completed.count.to_f / tasks.count * 100).round
  end

  def overdue_tasks
    tasks.where('due_date < ? AND status != ?', Date.current, 'done')
  end
end
```

---

## ขั้นตอนที่ 4: Stripe Billing

### ตั้งค่า Pay Gem

```ruby
# config/initializers/pay.rb
Pay.setup do |config|
  config.application_name = "ProjectHub"
  config.business_name    = "ProjectHub Inc."
  config.business_address = "123 Main St, Bangkok 10100, Thailand"
  config.support_email    = Mail::Address.new("support@projecthub.com")

  config.default_product_name = "ProjectHub"
  config.default_plan_name    = "free"

  config.send_emails = true
  config.automount_routes = true
  config.routes_path = "/billing"

  config.currency = "thb"
  config.supported_currencies = %w[thb usd]
end

# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
StripeEvent.signing_secret = Rails.application.credentials.dig(:stripe, :webhook_secret)
```

### Subscription Controller

```ruby
# app/controllers/subscriptions_controller.rb
class SubscriptionsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_organization

  def index
    @current_subscription = @organization.subscription
    @plans = Organization::PLANS
  end

  def create
    plan = params[:plan]
    raise ArgumentError, "Invalid plan" unless Organization::PLANS.key?(plan)

    unless @organization.can_upgrade_to?(plan)
      redirect_to billing_path, alert: "Cannot upgrade to this plan"
      return
    end

    # Stripe Checkout Session
    session = Stripe::Checkout::Session.create({
      customer_email: current_user.email,
      line_items: [{
        price:    stripe_price_id_for(plan),
        quantity: 1
      }],
      mode:                  'subscription',
      success_url:           billing_url(@organization) + '?session_id={CHECKOUT_SESSION_ID}',
      cancel_url:            billing_url(@organization),
      metadata: {
        organization_id: @organization.id,
        plan:            plan
      },
      subscription_data: {
        metadata: {
          organization_id: @organization.id,
          plan:            plan
        }
      }
    })

    redirect_to session.url, allow_other_host: true
  end

  def cancel
    subscription = @organization.subscription

    if subscription
      subscription.cancel
      redirect_to billing_path(@organization),
                  notice: "Subscription cancelled. You'll retain access until #{subscription.ends_at.strftime('%B %d, %Y')}."
    else
      redirect_to billing_path(@organization), alert: "No active subscription found."
    end
  end

  private

  def set_organization
    @organization = current_user.organizations.find_by!(slug: params[:organization_slug])
    authorize @organization, :billing?
  end

  def stripe_price_id_for(plan)
    Rails.application.credentials.dig(:stripe, :prices, plan.to_sym)
  end
end
```

### Stripe Webhook Handler

```ruby
# app/controllers/stripe_webhooks_controller.rb
class StripeWebhooksController < ActionController::Base
  skip_before_action :verify_authenticity_token

  def create
    payload    = request.body.read
    sig_header = request.env['HTTP_STRIPE_SIGNATURE']

    begin
      event = Stripe::Webhook.construct_event(
        payload,
        sig_header,
        Rails.application.credentials.dig(:stripe, :webhook_secret)
      )
    rescue JSON::ParserError => e
      render json: { error: e.message }, status: :bad_request
      return
    rescue Stripe::SignatureVerificationError => e
      render json: { error: e.message }, status: :unauthorized
      return
    end

    handle_event(event)
    render json: { received: true }
  end

  private

  def handle_event(event)
    case event['type']
    when 'checkout.session.completed'
      handle_checkout_completed(event['data']['object'])
    when 'customer.subscription.updated'
      handle_subscription_updated(event['data']['object'])
    when 'customer.subscription.deleted'
      handle_subscription_cancelled(event['data']['object'])
    when 'invoice.payment_failed'
      handle_payment_failed(event['data']['object'])
    when 'invoice.payment_succeeded'
      handle_payment_succeeded(event['data']['object'])
    end
  end

  def handle_checkout_completed(session)
    org_id = session.dig('metadata', 'organization_id')
    plan   = session.dig('metadata', 'plan')

    organization = Organization.find(org_id)
    organization.update!(
      plan:                  plan,
      stripe_customer_id:    session['customer'],
      stripe_subscription_id: session['subscription']
    )

    OrganizationMailer.subscription_activated(organization, plan).deliver_later
  end

  def handle_subscription_cancelled(subscription)
    org = Organization.find_by(stripe_subscription_id: subscription['id'])
    return unless org

    org.update!(plan: 'free', stripe_subscription_id: nil)
    OrganizationMailer.subscription_cancelled(org).deliver_later
  end

  def handle_payment_failed(invoice)
    org = Organization.find_by(stripe_customer_id: invoice['customer'])
    return unless org

    OrganizationMailer.payment_failed(org, invoice['amount_due']).deliver_later
  end
end
```

---

## ขั้นตอนที่ 5: Feature Flags

```ruby
# config/initializers/flipper.rb
require 'flipper'
require 'flipper/adapters/active_record'

Flipper.configure do |config|
  config.adapter { Flipper::Adapters::ActiveRecord.new }
end

# ตั้งค่า features ตาม plan
Flipper[:api_access].enable_group(:starter_and_above)
Flipper[:advanced_reports].enable_group(:starter_and_above)
Flipper[:custom_domain].enable_group(:professional_and_above)
Flipper[:audit_logs].enable_group(:enterprise)

# Define groups
Flipper.register(:starter_and_above) do |actor|
  actor.respond_to?(:plan) &&
    actor.plan.in?(%w[starter professional enterprise])
end

Flipper.register(:professional_and_above) do |actor|
  actor.respond_to?(:plan) &&
    actor.plan.in?(%w[professional enterprise])
end

Flipper.register(:enterprise) do |actor|
  actor.respond_to?(:plan) && actor.plan == 'enterprise'
end

# app/helpers/feature_helper.rb
module FeatureHelper
  def feature_enabled?(feature, organization = current_organization)
    Flipper.enabled?(feature, organization)
  end
end

# ใน Controller
class ReportsController < ApplicationController
  before_action :check_reports_feature

  def advanced
    raise Pundit::NotAuthorizedError unless Flipper.enabled?(:advanced_reports, current_organization)
    @data = AdvancedReportService.new(current_organization).generate
  end

  private

  def check_reports_feature
    unless Flipper.enabled?(:advanced_reports, current_organization)
      redirect_to upgrade_path, alert: "Advanced reports require Starter plan or above"
    end
  end
end

# ใน View
# <% if feature_enabled?(:api_access) %>
#   <%= link_to "API Settings", api_settings_path %>
# <% else %>
#   <%= link_to "Upgrade to access API", upgrade_path, class: "upgrade-link" %>
# <% end %>
```

---

## ขั้นตอนที่ 6: Admin Dashboard

```ruby
# app/controllers/admin/base_controller.rb
module Admin
  class BaseController < ApplicationController
    before_action :authenticate_admin!

    layout 'admin'

    private

    def authenticate_admin!
      unless current_user&.admin?
        redirect_to root_path, alert: "Access denied"
      end
    end
  end
end

# app/controllers/admin/dashboard_controller.rb
module Admin
  class DashboardController < BaseController
    def index
      @stats = {
        total_organizations: Organization.count,
        total_users:         User.count,
        total_projects:      Project.count,
        monthly_revenue:     calculate_monthly_revenue,
        new_signups_today:   User.where(created_at: Time.current.beginning_of_day..).count,
        active_subscriptions: Organization.where.not(plan: 'free').count
      }

      @plan_distribution = Organization.group(:plan).count
      @recent_signups    = User.order(created_at: :desc).limit(10).includes(:memberships)
      @recent_orgs       = Organization.order(created_at: :desc).limit(10)
    end

    private

    def calculate_monthly_revenue
      # ดึงจาก Stripe
      Stripe::Invoice.list(
        created: {
          gte: Time.current.beginning_of_month.to_i
        },
        status: 'paid'
      ).data.sum { |inv| inv.amount_paid } / 100.0
    rescue Stripe::StripeError
      0
    end
  end
end

# app/controllers/admin/organizations_controller.rb
module Admin
  class OrganizationsController < BaseController
    def index
      @organizations = Organization.includes(:memberships, :projects)
                                   .order(created_at: :desc)
                                   .page(params[:page]).per(25)

      if params[:q].present?
        @organizations = @organizations.where(
          "name ILIKE ? OR slug ILIKE ?",
          "%#{params[:q]}%",
          "%#{params[:q]}%"
        )
      end

      if params[:plan].present?
        @organizations = @organizations.where(plan: params[:plan])
      end
    end

    def show
      @organization = Organization.find(params[:id])
      @members      = @organization.memberships.includes(:user).order(created_at: :asc)
      @projects     = @organization.projects.order(created_at: :desc)
      @billing_info = fetch_billing_info(@organization)
    end

    def update_plan
      @organization = Organization.find(params[:id])
      new_plan = params[:plan]

      if @organization.update(plan: new_plan)
        AdminMailer.plan_changed_by_admin(@organization, new_plan, current_user).deliver_later
        redirect_to admin_organization_path(@organization),
                    notice: "Plan updated to #{new_plan}"
      else
        render :show, alert: "Failed to update plan"
      end
    end

    def suspend
      @organization = Organization.find(params[:id])
      @organization.update!(status: 'suspended')

      OrganizationMailer.account_suspended(@organization).deliver_later

      redirect_to admin_organizations_path,
                  notice: "Organization #{@organization.name} suspended"
    end

    private

    def fetch_billing_info(organization)
      return nil unless organization.stripe_customer_id.present?

      {
        customer:     Stripe::Customer.retrieve(organization.stripe_customer_id),
        subscription: organization.stripe_subscription_id.present? ?
                        Stripe::Subscription.retrieve(organization.stripe_subscription_id) : nil,
        invoices:     Stripe::Invoice.list(customer: organization.stripe_customer_id, limit: 5)
      }
    rescue Stripe::StripeError => e
      Rails.logger.error "Stripe error for org #{organization.id}: #{e.message}"
      nil
    end
  end
end
```

---

## ขั้นตอนที่ 7: JWT API

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      include ActionController::HttpAuthentication::Token::ControllerMethods

      before_action :authenticate_api_user!

      rescue_from ActiveRecord::RecordNotFound,    with: :not_found
      rescue_from Pundit::NotAuthorizedError,      with: :forbidden
      rescue_from ActiveRecord::RecordInvalid,     with: :unprocessable_entity
      rescue_from ActionController::ParameterMissing, with: :bad_request

      private

      def authenticate_api_user!
        token = extract_token_from_request
        @current_user = User.from_jwt(token)

        unless @current_user
          render json: { error: 'Unauthorized. Invalid or expired token.' },
                 status: :unauthorized
        end
      end

      def current_user
        @current_user
      end

      def current_organization
        @current_organization ||= begin
          org_id = request.headers['X-Organization-ID'] || params[:organization_id]
          org    = Organization.find_by(slug: org_id) ||
                   Organization.find_by(id: org_id)

          unless org && current_user.organizations.include?(org)
            render json: { error: 'Organization not found or access denied' },
                   status: :not_found
            return
          end

          org
        end
      end

      def extract_token_from_request
        # Authorization: Bearer <token>
        auth_header = request.headers['Authorization']
        return nil unless auth_header&.start_with?('Bearer ')

        auth_header.split(' ', 2).last
      end

      def not_found(e)
        render json: { error: e.message }, status: :not_found
      end

      def forbidden(e)
        render json: { error: 'Access denied' }, status: :forbidden
      end

      def unprocessable_entity(e)
        render json: {
          error:  'Validation failed',
          errors: e.record.errors.full_messages
        }, status: :unprocessable_entity
      end

      def bad_request(e)
        render json: { error: e.message }, status: :bad_request
      end
    end
  end
end

# app/controllers/api/v1/auth_controller.rb
module Api
  module V1
    class AuthController < ActionController::API
      def login
        user = User.find_by(email: params[:email])

        if user&.valid_password?(params[:password])
          render json: {
            token:      user.generate_jwt,
            expires_at: 24.hours.from_now.iso8601,
            user:       UserSerializer.new(user).as_json
          }
        else
          render json: { error: 'Invalid email or password' },
                 status: :unauthorized
        end
      end

      def refresh
        old_token = extract_token_from_request
        user      = User.from_jwt(old_token)

        if user
          render json: {
            token:      user.generate_jwt,
            expires_at: 24.hours.from_now.iso8601
          }
        else
          render json: { error: 'Invalid token' }, status: :unauthorized
        end
      end

      private

      def extract_token_from_request
        auth_header = request.headers['Authorization']
        return nil unless auth_header&.start_with?('Bearer ')
        auth_header.split(' ', 2).last
      end
    end
  end
end

# app/controllers/api/v1/projects_controller.rb
module Api
  module V1
    class ProjectsController < BaseController
      before_action :set_project, only: [:show, :update, :destroy]

      def index
        @projects = current_organization.projects
                                        .order(updated_at: :desc)
                                        .page(params[:page]).per(params[:per_page] || 20)

        render json: {
          projects:   @projects.map { |p| ProjectSerializer.new(p).as_json },
          pagination: pagination_meta(@projects)
        }
      end

      def show
        render json: { project: ProjectSerializer.new(@project, include_tasks: true).as_json }
      end

      def create
        unless current_organization.can_create_project?
          render json: {
            error: "Project limit reached for your plan. Upgrade to create more projects."
          }, status: :forbidden
          return
        end

        @project = current_organization.projects.build(project_params)
        @project.created_by = current_user

        @project.save!
        render json: { project: ProjectSerializer.new(@project).as_json },
               status: :created
      end

      def update
        @project.update!(project_params)
        render json: { project: ProjectSerializer.new(@project).as_json }
      end

      def destroy
        @project.destroy!
        head :no_content
      end

      private

      def set_project
        @project = current_organization.projects.find(params[:id])
      end

      def project_params
        params.require(:project).permit(:name, :description, :status, :start_date, :end_date)
      end

      def pagination_meta(collection)
        {
          current_page:  collection.current_page,
          total_pages:   collection.total_pages,
          total_count:   collection.total_count,
          per_page:      collection.limit_value
        }
      end
    end
  end
end
```

---

## ขั้นตอนที่ 8: Onboarding Flow

```ruby
# app/controllers/onboarding_controller.rb
class OnboardingController < ApplicationController
  before_action :authenticate_user!
  before_action :redirect_if_onboarded

  def step1_profile
    @user = current_user
  end

  def step1_profile_update
    if current_user.update(profile_params)
      redirect_to onboarding_step2_path
    else
      render :step1_profile
    end
  end

  def step2_organization
  end

  def step2_organization_create
    @organization = Organization.new(org_params)
    @organization.plan = 'free'

    if @organization.save
      Membership.create!(
        organization: @organization,
        user:         current_user,
        role:         'owner',
        status:       'active',
        accepted_at:  Time.current
      )

      session[:current_organization_id] = @organization.id
      redirect_to onboarding_step3_path
    else
      render :step2_organization
    end
  end

  def step3_invite_team
    @organization = current_organization
  end

  def step3_invite_team_send
    emails = params[:emails].to_s.split(/[\s,]+/).map(&:strip).uniq
    emails.select { |e| e.match?(URI::MailTo::EMAIL_REGEXP) }.each do |email|
      InvitationService.new(
        organization: current_organization,
        invited_by:   current_user,
        email:        email,
        role:         'member'
      ).invite!
    end

    redirect_to onboarding_step4_path
  end

  def step4_first_project
  end

  def step4_first_project_create
    @project = current_organization.projects.create!(
      name:       params[:project_name],
      created_by: current_user
    )

    current_user.update!(onboarded_at: Time.current)

    redirect_to organization_project_path(current_organization, @project),
                notice: "Welcome to ProjectHub! 🎉"
  end

  private

  def redirect_if_onboarded
    if current_user.onboarded_at.present?
      redirect_to dashboard_path
    end
  end

  def profile_params
    params.require(:user).permit(:first_name, :last_name, :avatar)
  end

  def org_params
    params.require(:organization).permit(:name)
  end
end

# app/services/invitation_service.rb
class InvitationService
  def initialize(organization:, invited_by:, email:, role: 'member')
    @organization = organization
    @invited_by   = invited_by
    @email        = email
    @role         = role
  end

  def invite!
    return :already_member if already_member?

    existing_user = User.find_by(email: @email)

    if existing_user
      invite_existing_user(existing_user)
    else
      invite_new_user
    end
  end

  private

  def already_member?
    @organization.members.exists?(email: @email)
  end

  def invite_existing_user(user)
    membership = Membership.find_or_initialize_by(
      organization: @organization,
      user:         user
    )

    if membership.new_record? || membership.status == 'pending'
      membership.update!(
        role:         @role,
        status:       'pending',
        invited_at:   Time.current,
        invitation_token: SecureRandom.urlsafe_base64(32)
      )

      OrganizationMailer.invite_existing_user(
        membership:   membership,
        invited_by:   @invited_by
      ).deliver_later
    end

    membership
  end

  def invite_new_user
    token = SecureRandom.urlsafe_base64(32)

    invitation = Invitation.create!(
      organization:     @organization,
      invited_by:       @invited_by,
      email:            @email,
      role:             @role,
      token:            token,
      expires_at:       7.days.from_now
    )

    OrganizationMailer.invite_new_user(
      invitation: invitation,
      invited_by: @invited_by
    ).deliver_later

    invitation
  end
end
```

---

## ขั้นตอนที่ 9: Mailers

```ruby
# app/mailers/organization_mailer.rb
class OrganizationMailer < ApplicationMailer
  def invite_existing_user(membership:, invited_by:)
    @membership  = membership
    @invited_by  = invited_by
    @org         = membership.organization
    @accept_url  = accept_invitation_url(token: membership.invitation_token)

    mail(
      to:      membership.user.email,
      subject: "#{invited_by.full_name} invited you to #{@org.name} on ProjectHub"
    )
  end

  def subscription_activated(organization, plan)
    @organization = organization
    @plan         = plan
    @plan_config  = Organization::PLANS[plan]

    mail(
      to:      organization.owner.email,
      subject: "Your #{@plan_config[:name]} subscription is now active!"
    )
  end

  def subscription_cancelled(organization)
    @organization = organization
    mail(
      to:      organization.owner.email,
      subject: "Your ProjectHub subscription has been cancelled"
    )
  end

  def payment_failed(organization, amount_due)
    @organization = organization
    @amount_due   = amount_due / 100.0  # convert from cents

    mail(
      to:      organization.owner.email,
      subject: "⚠️ Payment failed for your ProjectHub subscription"
    )
  end

  def account_suspended(organization)
    @organization = organization
    mail(
      to:      organization.owner.email,
      subject: "Your ProjectHub account has been suspended"
    )
  end

  def plan_upgraded(organization, new_plan)
    @organization = organization
    @new_plan     = new_plan
    @plan_config  = Organization::PLANS[new_plan]

    mail(
      to:      organization.owner.email,
      subject: "Welcome to ProjectHub #{@plan_config[:name]}! 🚀"
    )
  end
end
```

---

## ขั้นตอนที่ 10: Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, controllers: {
    registrations: 'users/registrations',
    sessions:      'users/sessions',
    passwords:     'users/passwords'
  }

  # Onboarding
  namespace :onboarding do
    get  'step1',  to: 'main#step1_profile'
    post 'step1',  to: 'main#step1_profile_update'
    get  'step2',  to: 'main#step2_organization'
    post 'step2',  to: 'main#step2_organization_create'
    get  'step3',  to: 'main#step3_invite_team'
    post 'step3',  to: 'main#step3_invite_team_send'
    get  'step4',  to: 'main#step4_first_project'
    post 'step4',  to: 'main#step4_first_project_create'
  end

  # Organizations
  resources :organizations, param: :slug do
    resources :projects do
      resources :tasks
    end
    resources :memberships
    resources :invitations
    resource  :billing, controller: 'billing'
    resource  :settings, controller: 'organization_settings'
  end

  # Admin
  namespace :admin do
    root to: 'dashboard#index'
    resources :organizations do
      member do
        patch :update_plan
        patch :suspend
        patch :unsuspend
      end
    end
    resources :users
    resources :subscriptions
    resources :feature_flags
  end

  # API
  namespace :api do
    namespace :v1 do
      post 'auth/login',   to: 'auth#login'
      post 'auth/refresh', to: 'auth#refresh'

      resources :organizations, param: :slug do
        resources :projects do
          resources :tasks
        end
        resources :members
      end

      resource :profile
    end
  end

  # Stripe webhooks
  post '/webhooks/stripe', to: 'stripe_webhooks#create'

  # Health check
  get '/health', to: 'health#check'

  # Dashboard (root after login)
  authenticated :user do
    root to: 'dashboard#index', as: :authenticated_root
  end

  root to: 'landing#index'
end
```

---

## ขั้นตอนที่ 11: Background Jobs

```ruby
# app/jobs/trial_expiration_job.rb
class TrialExpirationJob < ApplicationJob
  queue_as :default

  def perform
    expiring_soon = Organization.where(plan: 'trial')
                                .where(trial_ends_at: 3.days.from_now..)
                                .where(trial_ends_at: ..Time.current + 4.days)

    expiring_soon.each do |org|
      OrganizationMailer.trial_expiring_soon(org).deliver_later
    end

    expired = Organization.where(plan: 'trial')
                          .where('trial_ends_at < ?', Time.current)

    expired.each do |org|
      org.update!(plan: 'free')
      OrganizationMailer.trial_expired(org).deliver_later
    end
  end
end

# app/jobs/send_weekly_report_job.rb
class SendWeeklyReportJob < ApplicationJob
  queue_as :reports

  def perform
    Organization.active.each do |org|
      next unless org.feature_enabled?('weekly_reports')

      data = WeeklyReportService.new(org).generate
      OrganizationMailer.weekly_report(org, data).deliver_later
    end
  end
end

# config/sidekiq.yml
:concurrency: 10
:queues:
  - [critical, 3]
  - [default, 2]
  - [reports, 1]
  - [mailers, 2]

# config/initializers/sidekiq_cron.rb
schedule = [
  {
    name:   'Trial Expiration',
    cron:   '0 8 * * *',      # ทุกวัน 8am
    class:  'TrialExpirationJob'
  },
  {
    name:   'Weekly Report',
    cron:   '0 9 * * 1',      # ทุกวันจันทร์ 9am
    class:  'SendWeeklyReportJob'
  }
]

Sidekiq::Cron::Job.load_from_array(schedule)
```

---

## ขั้นตอนที่ 12: Authorization ด้วย Pundit

```ruby
# app/policies/project_policy.rb
class ProjectPolicy < ApplicationPolicy
  class Scope < Scope
    def resolve
      # เห็นเฉพาะ projects ของ organization ที่ user เป็นสมาชิก
      scope.where(organization: user.organizations)
    end
  end

  def index?   = member_of_organization?
  def show?    = member_of_organization?
  def create?  = admin_of_organization?
  def update?  = admin_or_project_lead?
  def destroy? = admin_of_organization?

  private

  def member_of_organization?
    record.organization.member?(user)
  end

  def admin_of_organization?
    record.organization.admin?(user)
  end

  def admin_or_project_lead?
    admin_of_organization? ||
      record.project_members.exists?(user: user, role: 'lead')
  end
end

# app/policies/organization_policy.rb
class OrganizationPolicy < ApplicationPolicy
  def show?    = member?
  def update?  = owner_or_admin?
  def billing? = owner?
  def destroy? = owner?

  private

  def member?       = record.member?(user)
  def owner?        = record.owner == user
  def owner_or_admin? = record.admin?(user)
end
```

---

## ขั้นตอนที่ 13: Testing

```ruby
# spec/models/organization_spec.rb
RSpec.describe Organization, type: :model do
  describe 'plan features' do
    it 'free plan has no features' do
      org = build(:organization, plan: 'free')
      expect(org.feature_enabled?('api_access')).to be false
    end

    it 'starter plan has api_access' do
      org = build(:organization, plan: 'starter')
      expect(org.feature_enabled?('api_access')).to be true
    end

    it 'can add member when under limit' do
      org = create(:organization, plan: 'free')
      create_list(:membership, 4, organization: org)
      expect(org.can_add_member?).to be true
    end

    it 'cannot add member when at limit' do
      org = create(:organization, plan: 'free')
      create_list(:membership, 5, organization: org)
      expect(org.can_add_member?).to be false
    end
  end
end

# spec/factories/organizations.rb
FactoryBot.define do
  factory :organization do
    name   { Faker::Company.name }
    plan   { 'free' }
    status { 'active' }

    after(:create) do |org, evaluator|
      create(:membership, organization: org, role: 'owner')
    end

    trait :starter  { plan { 'starter' } }
    trait :pro      { plan { 'professional' } }
    trait :enterprise { plan { 'enterprise' } }
  end
end
```

---

## แบบฝึกหัดท้ายบท

1. เพิ่ม OAuth login ด้วย Google
2. เพิ่ม real-time notifications ด้วย Action Cable
3. สร้าง API documentation ด้วย Swagger
4. เพิ่ม export data ในรูปแบบ CSV/PDF
5. สร้าง mobile-friendly PWA

---

## สรุป

ProjectHub SaaS Application ครอบคลุม:
- **Multi-tenancy**: Organizations, Memberships
- **Billing**: Stripe integration ด้วย Pay gem
- **Feature Flags**: Flipper สำหรับ plan-based features
- **Admin**: Complete dashboard สำหรับ monitoring
- **API**: JWT-based REST API
- **Onboarding**: Step-by-step user onboarding
- **Security**: Pundit authorization, HTTPS only

โปรเจคนี้เป็น foundation สำหรับ production-ready SaaS application
