# Part 94: Rails SaaS Patterns

## บทนำ

SaaS (Software as a Service) applications มี patterns และ requirements พิเศษ เช่น multi-tenancy, subscription billing, usage metering, feature flags และ onboarding flows บทนี้จะครอบคลุม patterns ที่ใช้จริงในการสร้าง SaaS ด้วย Rails

## 1. Multi-tenancy Patterns

### 1.1 Subdomain-based Multi-tenancy

```ruby
# config/routes.rb
Rails.application.routes.draw do
  constraints(TenantConstraint) do
    scope module: 'tenanted' do
      root to: 'dashboard#index'
      resources :projects
      resources :users
    end
  end
  
  scope module: 'public' do
    root to: 'landing#index'
    resources :accounts, only: [:new, :create]
    get 'pricing', to: 'pages#pricing'
  end
end

# lib/tenant_constraint.rb
class TenantConstraint
  def self.matches?(request)
    subdomain = request.subdomain
    return false if subdomain.blank? || subdomain == 'www'
    
    Account.find_by(subdomain: subdomain).present?
  end
end
```

### 1.2 Row-level Multi-tenancy ด้วย Acts As Tenant

```ruby
# Gemfile
gem 'acts_as_tenant'

# app/models/account.rb
class Account < ApplicationRecord
  has_many :users
  has_many :projects
  has_many :subscriptions
  
  validates :name, presence: true
  validates :subdomain, presence: true, uniqueness: true,
            format: { with: /\A[a-z0-9\-]+\z/, message: "only lowercase letters, numbers, and hyphens" }
  
  def self.current
    ActsAsTenant.current_tenant
  end
end

# app/models/project.rb
class Project < ApplicationRecord
  acts_as_tenant :account
  
  belongs_to :account
  belongs_to :owner, class_name: 'User'
  
  validates :name, presence: true
  # acts_as_tenant จะ scope queries ทั้งหมดด้วย account_id อัตโนมัติ
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  set_current_tenant_through_filter
  before_action :find_tenant
  
  private
  
  def find_tenant
    if request.subdomain.present? && request.subdomain != 'www'
      account = Account.find_by(subdomain: request.subdomain)
      
      if account
        set_current_tenant(account)
      else
        redirect_to root_url(subdomain: false), alert: 'Account not found'
      end
    end
  end
end
```

### 1.3 Current Attributes Pattern

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :user, :account, :request_id, :user_agent, :ip_address
  
  delegate :user_agent, to: :request, allow_nil: true
  
  def account
    super || user&.account
  end
  
  def admin?
    user&.admin?
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_current_attributes
  
  private
  
  def set_current_attributes
    Current.request_id = request.uuid
    Current.user_agent = request.user_agent
    Current.ip_address = request.ip
    Current.user = authenticate_user
    Current.account = find_current_account
  end
end
```

## 2. Stripe Subscriptions

### 2.1 Subscription Models

```ruby
# Gemfile
gem 'stripe'
gem 'pay'  # Payments library สำหรับ Rails

# app/models/account.rb
class Account < ApplicationRecord
  pay_customer  # จาก pay gem
  
  has_many :subscriptions, class_name: 'Pay::Subscription'
  
  def active_subscription
    subscriptions.active.order(:created_at).last
  end
  
  def subscribed?
    active_subscription.present?
  end
  
  def on_plan?(plan_name)
    active_subscription&.name == plan_name
  end
  
  def subscription_status
    active_subscription&.status || 'inactive'
  end
  
  def trial_ends_at
    subscriptions.trial.first&.trial_ends_at
  end
  
  def on_trial?
    trial_ends_at.present? && trial_ends_at > Time.current
  end
end

# app/models/plan.rb
class Plan < ApplicationRecord
  validates :name, presence: true, uniqueness: true
  validates :stripe_price_id, presence: true
  validates :price_cents, presence: true, numericality: { greater_than_or_equal_to: 0 }
  validates :interval, inclusion: { in: %w[month year] }
  
  FEATURES = {
    'starter' => {
      max_users: 5,
      max_projects: 10,
      storage_gb: 5,
      api_calls_per_month: 10_000
    },
    'pro' => {
      max_users: 25,
      max_projects: 100,
      storage_gb: 50,
      api_calls_per_month: 100_000
    },
    'enterprise' => {
      max_users: Float::INFINITY,
      max_projects: Float::INFINITY,
      storage_gb: 1000,
      api_calls_per_month: 1_000_000
    }
  }.freeze
  
  def features
    FEATURES[name.downcase] || FEATURES['starter']
  end
  
  def allows?(feature)
    features[feature] != 0
  end
  
  def price_dollars
    price_cents / 100.0
  end
  
  def formatted_price
    "$#{price_dollars.to_i}/#{interval}"
  end
end
```

### 2.2 Subscription Controller

```ruby
# app/controllers/subscriptions_controller.rb
class SubscriptionsController < ApplicationController
  before_action :authenticate_user!
  before_action :require_owner!
  
  def index
    @subscription = current_account.active_subscription
    @plans = Plan.active.order(:price_cents)
    @usage = UsageService.new(current_account).current_month_usage
  end
  
  def create
    plan = Plan.find(params[:plan_id])
    
    result = SubscriptionService.new(current_account).subscribe!(
      plan: plan,
      payment_method: params[:payment_method_id]
    )
    
    if result[:success]
      redirect_to subscription_path, notice: "Successfully subscribed to #{plan.name}!"
    else
      render :index, alert: result[:error]
    end
  rescue Stripe::CardError => e
    render :index, alert: e.user_message
  end
  
  def update
    new_plan = Plan.find(params[:plan_id])
    
    result = SubscriptionService.new(current_account).change_plan!(
      new_plan: new_plan,
      proration: params[:proration] != 'false'
    )
    
    if result[:success]
      redirect_to subscription_path, notice: "Plan updated to #{new_plan.name}!"
    else
      render :index, alert: result[:error]
    end
  end
  
  def destroy
    SubscriptionService.new(current_account).cancel!(
      at_period_end: params[:at_period_end] != 'false'
    )
    
    redirect_to subscription_path, notice: "Subscription cancelled."
  end
  
  def resume
    SubscriptionService.new(current_account).resume!
    redirect_to subscription_path, notice: "Subscription resumed!"
  end
  
  private
  
  def require_owner!
    unless current_user.owner?
      redirect_to dashboard_path, alert: "Only account owners can manage billing."
    end
  end
end
```

### 2.3 Subscription Service

```ruby
# app/services/subscription_service.rb
class SubscriptionService
  class SubscriptionError < StandardError; end
  
  def initialize(account)
    @account = account
  end
  
  def subscribe!(plan:, payment_method:, trial_days: nil)
    stripe_customer = ensure_stripe_customer!
    
    subscription_params = {
      customer: stripe_customer.id,
      items: [{ price: plan.stripe_price_id }],
      payment_behavior: 'default_incomplete',
      expand: ['latest_invoice.payment_intent']
    }
    
    subscription_params[:trial_period_days] = trial_days if trial_days
    
    if payment_method.present?
      attach_payment_method!(stripe_customer, payment_method)
      subscription_params[:default_payment_method] = payment_method
    end
    
    stripe_sub = Stripe::Subscription.create(subscription_params)
    
    # Save locally
    subscription = @account.subscriptions.create!(
      stripe_subscription_id: stripe_sub.id,
      stripe_price_id: plan.stripe_price_id,
      plan: plan,
      status: stripe_sub.status,
      current_period_start: Time.at(stripe_sub.current_period_start),
      current_period_end: Time.at(stripe_sub.current_period_end),
      trial_ends_at: stripe_sub.trial_end ? Time.at(stripe_sub.trial_end) : nil
    )
    
    { success: true, subscription: subscription, stripe_subscription: stripe_sub }
  rescue Stripe::StripeError => e
    { success: false, error: e.message }
  end
  
  def change_plan!(new_plan:, proration: true)
    subscription = @account.active_subscription
    raise SubscriptionError, "No active subscription" unless subscription
    
    stripe_sub = Stripe::Subscription.retrieve(subscription.stripe_subscription_id)
    
    Stripe::Subscription.update(
      stripe_sub.id,
      items: [{
        id: stripe_sub.items.data[0].id,
        price: new_plan.stripe_price_id
      }],
      proration_behavior: proration ? 'create_prorations' : 'none'
    )
    
    subscription.update!(
      plan: new_plan,
      stripe_price_id: new_plan.stripe_price_id
    )
    
    { success: true }
  rescue Stripe::StripeError => e
    { success: false, error: e.message }
  end
  
  def cancel!(at_period_end: true)
    subscription = @account.active_subscription
    raise SubscriptionError, "No active subscription" unless subscription
    
    if at_period_end
      Stripe::Subscription.update(
        subscription.stripe_subscription_id,
        cancel_at_period_end: true
      )
      subscription.update!(cancel_at_period_end: true)
    else
      Stripe::Subscription.cancel(subscription.stripe_subscription_id)
      subscription.update!(status: 'cancelled', cancelled_at: Time.current)
    end
  end
  
  def resume!
    subscription = @account.subscriptions.where(cancel_at_period_end: true).last
    raise SubscriptionError, "No cancelling subscription" unless subscription
    
    Stripe::Subscription.update(
      subscription.stripe_subscription_id,
      cancel_at_period_end: false
    )
    subscription.update!(cancel_at_period_end: false)
  end
  
  private
  
  def ensure_stripe_customer!
    if @account.stripe_customer_id.present?
      Stripe::Customer.retrieve(@account.stripe_customer_id)
    else
      customer = Stripe::Customer.create(
        email: @account.owner.email,
        name: @account.name,
        metadata: { account_id: @account.id }
      )
      @account.update!(stripe_customer_id: customer.id)
      customer
    end
  end
  
  def attach_payment_method!(customer, payment_method_id)
    Stripe::PaymentMethod.attach(payment_method_id, customer: customer.id)
    Stripe::Customer.update(customer.id, invoice_settings: {
      default_payment_method: payment_method_id
    })
  end
end
```

## 3. Stripe Webhooks

```ruby
# app/controllers/webhooks/stripe_controller.rb
module Webhooks
  class StripeController < ApplicationController
    skip_before_action :verify_authenticity_token
    skip_before_action :authenticate_user!
    before_action :verify_stripe_signature
    
    def receive
      event_handler = StripeEventHandler.new(event)
      event_handler.handle!
      
      head :ok
    rescue StripeEventHandler::UnhandledEventError
      head :ok  # ยังคง return 200 เพื่อไม่ให้ Stripe retry
    rescue => e
      Rails.logger.error "Stripe webhook error: #{e.message}"
      head :internal_server_error
    end
    
    private
    
    def verify_stripe_signature
      payload = request.body.read
      sig_header = request.env['HTTP_STRIPE_SIGNATURE']
      
      @event = Stripe::Webhook.construct_event(
        payload, sig_header, ENV['STRIPE_WEBHOOK_SECRET']
      )
    rescue Stripe::SignatureVerificationError
      render json: { error: 'Invalid signature' }, status: :bad_request
    end
    
    def event
      @event
    end
  end
end

# app/services/stripe_event_handler.rb
class StripeEventHandler
  class UnhandledEventError < StandardError; end
  
  HANDLED_EVENTS = %w[
    customer.subscription.created
    customer.subscription.updated
    customer.subscription.deleted
    customer.subscription.trial_will_end
    invoice.payment_succeeded
    invoice.payment_failed
    invoice.upcoming
    customer.updated
  ].freeze
  
  def initialize(event)
    @event = event
  end
  
  def handle!
    raise UnhandledEventError unless HANDLED_EVENTS.include?(@event.type)
    
    case @event.type
    when 'customer.subscription.created'
      handle_subscription_created
    when 'customer.subscription.updated'
      handle_subscription_updated
    when 'customer.subscription.deleted'
      handle_subscription_deleted
    when 'customer.subscription.trial_will_end'
      handle_trial_will_end
    when 'invoice.payment_succeeded'
      handle_payment_succeeded
    when 'invoice.payment_failed'
      handle_payment_failed
    when 'invoice.upcoming'
      handle_invoice_upcoming
    end
  end
  
  private
  
  def handle_subscription_created
    stripe_sub = @event.data.object
    account = Account.find_by(stripe_customer_id: stripe_sub.customer)
    return unless account
    
    subscription = account.subscriptions.find_or_initialize_by(
      stripe_subscription_id: stripe_sub.id
    )
    subscription.update!(
      status: stripe_sub.status,
      current_period_start: Time.at(stripe_sub.current_period_start),
      current_period_end: Time.at(stripe_sub.current_period_end)
    )
    
    SubscriptionMailer.welcome(account).deliver_later if stripe_sub.status == 'active'
  end
  
  def handle_subscription_updated
    stripe_sub = @event.data.object
    account = Account.find_by(stripe_customer_id: stripe_sub.customer)
    return unless account
    
    subscription = account.subscriptions.find_by(stripe_subscription_id: stripe_sub.id)
    return unless subscription
    
    subscription.update!(
      status: stripe_sub.status,
      cancel_at_period_end: stripe_sub.cancel_at_period_end,
      current_period_end: Time.at(stripe_sub.current_period_end)
    )
    
    # Notify ถ้า plan เปลี่ยน
    if subscription.saved_change_to_stripe_price_id?
      SubscriptionMailer.plan_changed(account, subscription).deliver_later
    end
  end
  
  def handle_subscription_deleted
    stripe_sub = @event.data.object
    account = Account.find_by(stripe_customer_id: stripe_sub.customer)
    return unless account
    
    subscription = account.subscriptions.find_by(stripe_subscription_id: stripe_sub.id)
    subscription&.update!(status: 'cancelled', cancelled_at: Time.current)
    
    SubscriptionMailer.cancelled(account).deliver_later
  end
  
  def handle_trial_will_end
    stripe_sub = @event.data.object
    account = Account.find_by(stripe_customer_id: stripe_sub.customer)
    return unless account
    
    trial_ends_at = Time.at(stripe_sub.trial_end)
    days_remaining = ((trial_ends_at - Time.current) / 1.day).ceil
    
    SubscriptionMailer.trial_ending(account, days_remaining: days_remaining).deliver_later
  end
  
  def handle_payment_succeeded
    invoice = @event.data.object
    account = Account.find_by(stripe_customer_id: invoice.customer)
    return unless account
    
    # Record invoice
    account.invoices.create!(
      stripe_invoice_id: invoice.id,
      amount_cents: invoice.amount_paid,
      status: 'paid',
      period_start: Time.at(invoice.period_start),
      period_end: Time.at(invoice.period_end),
      invoice_url: invoice.hosted_invoice_url,
      invoice_pdf: invoice.invoice_pdf
    )
    
    SubscriptionMailer.payment_receipt(account, invoice: invoice).deliver_later
  end
  
  def handle_payment_failed
    invoice = @event.data.object
    account = Account.find_by(stripe_customer_id: invoice.customer)
    return unless account
    
    attempt_count = invoice.attempt_count
    
    SubscriptionMailer.payment_failed(
      account,
      attempt_count: attempt_count,
      next_attempt: invoice.next_payment_attempt ? Time.at(invoice.next_payment_attempt) : nil
    ).deliver_later
    
    # Suspend account after too many failures
    if attempt_count >= 3
      account.update!(suspended: true)
      AccountSuspensionJob.perform_later(account.id)
    end
  end
  
  def handle_invoice_upcoming
    invoice = @event.data.object
    account = Account.find_by(stripe_customer_id: invoice.customer)
    return unless account
    
    SubscriptionMailer.upcoming_invoice(account, invoice: invoice).deliver_later
  end
end
```

## 4. Usage-based Billing

### 4.1 Usage Tracking

```ruby
# app/models/usage_record.rb
class UsageRecord < ApplicationRecord
  belongs_to :account
  
  METRICS = %w[api_calls storage_bytes emails_sent active_users].freeze
  
  validates :metric, inclusion: { in: METRICS }
  validates :quantity, numericality: { greater_than_or_equal_to: 0 }
  
  scope :for_month, ->(year, month) {
    where(
      recorded_at: Time.new(year, month, 1, 0, 0, 0)..Time.new(year, month, 1, 0, 0, 0).end_of_month
    )
  }
  
  scope :current_month, -> { for_month(Time.current.year, Time.current.month) }
end

# app/services/usage_service.rb
class UsageService
  def initialize(account)
    @account = account
  end
  
  def track!(metric, quantity: 1, metadata: {})
    @account.usage_records.create!(
      metric: metric,
      quantity: quantity,
      metadata: metadata,
      recorded_at: Time.current
    )
    
    # Check if nearing limit
    check_limit_warning(metric)
  end
  
  def current_month_usage
    records = @account.usage_records.current_month.group(:metric).sum(:quantity)
    
    limits = plan_limits
    
    records.each_with_object({}) do |(metric, used), result|
      limit = limits[metric]
      
      result[metric] = {
        used: used,
        limit: limit,
        percentage: limit ? ((used.to_f / limit) * 100).round(1) : 0,
        over_limit: limit ? used > limit : false
      }
    end
  end
  
  def api_calls_this_month
    @account.usage_records.current_month.where(metric: 'api_calls').sum(:quantity)
  end
  
  def storage_used_bytes
    @account.usage_records.where(metric: 'storage_bytes').sum(:quantity)
  end
  
  def storage_used_gb
    (storage_used_bytes / 1.gigabyte.to_f).round(2)
  end
  
  def report_to_stripe!(stripe_subscription_item_id:, quantity:, timestamp: nil)
    Stripe::SubscriptionItem.create_usage_record(
      stripe_subscription_item_id,
      {
        quantity: quantity,
        timestamp: timestamp&.to_i || Time.current.to_i,
        action: 'increment'
      }
    )
  end
  
  private
  
  def plan_limits
    plan = @account.active_subscription&.plan
    return {} unless plan
    
    plan.features
  end
  
  def check_limit_warning(metric)
    usage = current_month_usage[metric]
    return unless usage
    
    if usage[:percentage] >= 90 && !usage[:over_limit]
      # Send warning once per period
      cache_key = "usage_warning:#{@account.id}:#{metric}:#{Time.current.strftime('%Y%m')}"
      
      unless Rails.cache.exist?(cache_key)
        Rails.cache.write(cache_key, true, expires_in: 30.days)
        UsageMailer.limit_warning(@account, metric: metric, usage: usage).deliver_later
      end
    elsif usage[:over_limit]
      UsageMailer.limit_exceeded(@account, metric: metric, usage: usage).deliver_later
    end
  end
end

# app/controllers/concerns/usage_trackable.rb
module UsageTrackable
  extend ActiveSupport::Concern
  
  def track_api_usage
    UsageService.new(Current.account).track!('api_calls')
  end
end
```

## 5. Feature Flags ด้วย Flipper

### 5.1 Flipper Setup

```ruby
# Gemfile
gem 'flipper'
gem 'flipper-active_record'
gem 'flipper-ui'

# config/initializers/flipper.rb
require 'flipper'
require 'flipper/adapters/active_record'

Flipper.configure do |config|
  config.default do
    adapter = Flipper::Adapters::ActiveRecord.new
    Flipper.new(adapter)
  end
end

# Define all features
FEATURES = {
  'new_dashboard' => 'New dashboard UI redesign',
  'ai_suggestions' => 'AI-powered content suggestions',
  'bulk_export' => 'Bulk data export feature',
  'advanced_analytics' => 'Advanced analytics dashboard',
  'team_workflows' => 'Team approval workflows',
  'custom_domains' => 'Custom domain support'
}.freeze
```

### 5.2 Feature Flag Usage

```ruby
# app/models/concerns/feature_flaggable.rb
module FeatureFlaggable
  extend ActiveSupport::Concern
  
  included do
    # Account เป็น Flipper actor
    def flipper_id
      "Account:#{id}"
    end
  end
  
  def feature_enabled?(feature_name)
    Flipper.enabled?(feature_name, self)
  end
  
  def enable_feature!(feature_name)
    Flipper.enable_actor(feature_name, self)
  end
  
  def disable_feature!(feature_name)
    Flipper.disable_actor(feature_name, self)
  end
  
  def enabled_features
    FEATURES.keys.select { |f| feature_enabled?(f) }
  end
end

# app/controllers/concerns/feature_flaggable.rb
module FeatureFlaggable
  extend ActiveSupport::Concern
  
  def require_feature!(feature_name)
    unless current_account.feature_enabled?(feature_name)
      respond_to do |format|
        format.html { redirect_to upgrade_path, alert: "This feature requires upgrading your plan" }
        format.json { render json: { error: 'Feature not available', feature: feature_name }, status: :payment_required }
      end
    end
  end
end

# ใน controller
class AnalyticsController < ApplicationController
  include FeatureFlaggable
  
  before_action -> { require_feature!('advanced_analytics') }
  
  def index
    # Only accessible if feature flag is enabled
    @analytics = AnalyticsService.new(current_account).generate_report
  end
end
```

### 5.3 Gradual Rollout

```ruby
# lib/tasks/feature_flags.rake
namespace :features do
  desc "List all features and their status"
  task status: :environment do
    FEATURES.each do |name, description|
      feature = Flipper[name]
      puts "#{name}: #{feature.enabled? ? '✓ enabled globally' : '○ not enabled globally'}"
      puts "  Description: #{description}"
      puts "  Enabled for #{Flipper::Adapters::ActiveRecord::Gate.where(feature_key: name, key: 'actors').count} specific accounts"
      puts
    end
  end
  
  desc "Enable feature for percentage of accounts"
  task :rollout, [:feature, :percentage] => :environment do |_, args|
    feature = args[:feature]
    percentage = args[:percentage].to_i
    
    raise "Unknown feature: #{feature}" unless FEATURES.key?(feature)
    raise "Percentage must be 0-100" unless (0..100).include?(percentage)
    
    Flipper.enable_percentage_of_actors(feature, percentage)
    puts "Enabled #{feature} for #{percentage}% of accounts"
  end
  
  desc "Enable feature globally"
  task :enable, [:feature] => :environment do |_, args|
    Flipper.enable(args[:feature])
    puts "Enabled #{args[:feature]} globally"
  end
  
  desc "Disable feature globally"
  task :disable, [:feature] => :environment do |_, args|
    Flipper.disable(args[:feature])
    puts "Disabled #{args[:feature]} globally"
  end
  
  desc "Enable feature for specific plan"
  task :enable_for_plan, [:feature, :plan_name] => :environment do |_, args|
    plan = Plan.find_by!(name: args[:plan_name])
    
    accounts = Account.joins(:subscriptions)
      .where(subscriptions: { plan: plan, status: 'active' })
    
    accounts.find_each do |account|
      Flipper.enable_actor(args[:feature], account)
    end
    
    puts "Enabled #{args[:feature]} for #{accounts.count} #{args[:plan_name]} accounts"
  end
end
```

## 6. Onboarding Flow

### 6.1 Onboarding Steps

```ruby
# app/models/onboarding.rb
class Onboarding < ApplicationRecord
  belongs_to :account
  
  STEPS = %w[
    profile_setup
    invite_team
    connect_integration
    create_first_project
    completed
  ].freeze
  
  def self.for_account(account)
    find_or_create_by(account: account)
  end
  
  def current_step
    STEPS.find { |step| !completed_steps.include?(step) } || 'completed'
  end
  
  def complete_step!(step)
    return if completed_steps.include?(step)
    
    update!(
      completed_steps: completed_steps + [step],
      last_completed_at: Time.current
    )
    
    check_if_fully_complete
  end
  
  def completed?
    completed_steps.length >= STEPS.length - 1  # Exclude 'completed'
  end
  
  def progress_percentage
    (completed_steps.count.to_f / (STEPS.count - 1) * 100).round
  end
  
  def step_completed?(step)
    completed_steps.include?(step)
  end
  
  private
  
  def check_if_fully_complete
    if completed?
      update!(completed_at: Time.current)
      OnboardingCompleteJob.perform_later(account_id)
    end
  end
end

# app/controllers/onboarding_controller.rb
class OnboardingController < ApplicationController
  before_action :authenticate_user!
  before_action :set_onboarding
  
  def show
    redirect_to onboarding_step_path(@onboarding.current_step)
  end
  
  def profile_setup
    @account = current_account
    
    if request.patch?
      if current_account.update(profile_params)
        @onboarding.complete_step!('profile_setup')
        redirect_to onboarding_step_path('invite_team'),
                    notice: "Profile updated!"
      else
        render :profile_setup
      end
    end
  end
  
  def invite_team
    if request.post?
      emails = params[:emails].to_s.split(',').map(&:strip).compact
      
      emails.each do |email|
        InvitationService.new(current_account).invite!(
          email: email,
          role: 'member',
          invited_by: current_user
        )
      end
      
      @onboarding.complete_step!('invite_team')
      redirect_to onboarding_step_path('connect_integration'),
                  notice: "Invitations sent!"
    end
  end
  
  def skip_step
    step = params[:step]
    return head :bad_request unless Onboarding::STEPS.include?(step)
    
    @onboarding.complete_step!(step)
    redirect_to onboarding_step_path(@onboarding.current_step)
  end
  
  private
  
  def set_onboarding
    @onboarding = Onboarding.for_account(current_account)
    
    if @onboarding.completed?
      redirect_to dashboard_path unless request.path.start_with?('/onboarding')
    end
  end
  
  def profile_params
    params.require(:account).permit(:name, :logo, :website, :industry)
  end
end
```

### 6.2 Onboarding Email Sequences

```ruby
# app/jobs/onboarding_email_sequence_job.rb
class OnboardingEmailSequenceJob < ApplicationJob
  queue_as :onboarding
  
  SEQUENCE = [
    { delay: 0,     email: :welcome },
    { delay: 1.day, email: :getting_started },
    { delay: 3.days, email: :tips_and_tricks },
    { delay: 7.days, email: :check_in },
    { delay: 14.days, email: :upgrade_prompt }
  ].freeze
  
  def perform(account_id, sequence_index = 0)
    account = Account.find_by(id: account_id)
    return unless account
    return if account.subscription_cancelled?
    
    step = SEQUENCE[sequence_index]
    return unless step
    
    OnboardingMailer.send(step[:email], account).deliver_now
    
    next_index = sequence_index + 1
    return unless SEQUENCE[next_index]
    
    next_step = SEQUENCE[next_index]
    self.class.set(wait: next_step[:delay]).perform_later(account_id, next_index)
  end
end
```

## 7. SaaS Metrics Dashboard

### 7.1 Key Metrics

```ruby
# app/services/saas_metrics_service.rb
class SaasMetricsService
  def initialize(date_range: 30.days.ago..Time.current)
    @date_range = date_range
  end
  
  # Monthly Recurring Revenue
  def mrr
    Account.joins(subscriptions: :plan)
      .where(subscriptions: { status: 'active' })
      .where(plans: { interval: 'month' })
      .sum('plans.price_cents') / 100.0
  end
  
  # Annual Recurring Revenue
  def arr
    mrr * 12
  end
  
  # Average Revenue Per User
  def arpu
    active_accounts = Account.with_active_subscription.count
    return 0 if active_accounts.zero?
    
    mrr / active_accounts
  end
  
  # Customer Lifetime Value (simplified)
  def ltv
    avg_churn = churn_rate
    return 0 if avg_churn.zero?
    
    arpu / (avg_churn / 100.0)
  end
  
  # New MRR (from new subscriptions)
  def new_mrr
    Account.joins(subscriptions: :plan)
      .where(subscriptions: {
        status: 'active',
        created_at: @date_range
      })
      .where(plans: { interval: 'month' })
      .sum('plans.price_cents') / 100.0
  end
  
  # Expansion MRR (upgrades)
  def expansion_mrr
    # Logic to calculate MRR from plan upgrades
    Subscription
      .where(created_at: @date_range)
      .where('previous_plan_price_cents < plan_price_cents')
      .sum('plan_price_cents - previous_plan_price_cents') / 100.0
  end
  
  # Churned MRR
  def churned_mrr
    Subscription
      .where(status: 'cancelled', cancelled_at: @date_range)
      .joins(:plan)
      .where(plans: { interval: 'month' })
      .sum('plans.price_cents') / 100.0
  end
  
  # Net MRR Change
  def net_mrr_change
    new_mrr + expansion_mrr - churned_mrr
  end
  
  # Churn Rate (monthly)
  def churn_rate
    start_count = Account.with_active_subscription.where('created_at < ?', @date_range.begin).count
    churned_count = Subscription.where(status: 'cancelled', cancelled_at: @date_range).count
    
    return 0 if start_count.zero?
    
    (churned_count.to_f / start_count * 100).round(2)
  end
  
  # Customer Acquisition Cost
  def cac
    # Simplified: total marketing + sales cost / new customers
    total_cost = MarketingCost.where(date: @date_range).sum(:amount)
    new_customers = Account.where(created_at: @date_range).count
    
    return 0 if new_customers.zero?
    
    total_cost / new_customers
  end
  
  # Activation Rate (accounts that complete onboarding)
  def activation_rate
    total = Account.where(created_at: @date_range).count
    activated = Account.joins(:onboarding)
      .where(created_at: @date_range)
      .where(onboardings: { completed_at: @date_range }).count
    
    return 0 if total.zero?
    
    (activated.to_f / total * 100).round(2)
  end
  
  # Trial Conversion Rate
  def trial_conversion_rate
    trials = Subscription.where(status: 'trialing', created_at: @date_range).count
    converted = Subscription.where(
      created_at: @date_range,
      status: 'active'
    ).where.not(trial_start: nil).count
    
    return 0 if trials.zero?
    
    (converted.to_f / trials * 100).round(2)
  end
  
  def summary
    {
      mrr: mrr,
      arr: arr,
      arpu: arpu,
      ltv: ltv,
      churn_rate: churn_rate,
      trial_conversion_rate: trial_conversion_rate,
      activation_rate: activation_rate,
      new_mrr: new_mrr,
      expansion_mrr: expansion_mrr,
      churned_mrr: churned_mrr,
      net_mrr_change: net_mrr_change
    }
  end
end
```

### 7.2 Metrics API Endpoint

```ruby
# app/controllers/admin/metrics_controller.rb
module Admin
  class MetricsController < AdminController
    def index
      @metrics = SaasMetricsService.new.summary
      @cohort = CohortAnalysisService.new.cohort_data(months: 6)
      @growth_chart = GrowthChartService.new.monthly_data(months: 12)
      
      render json: {
        metrics: @metrics,
        cohort: @cohort,
        growth: @growth_chart,
        generated_at: Time.current
      }
    end
    
    def cohort
      months = params[:months].to_i.clamp(3, 24)
      
      render json: CohortAnalysisService.new.cohort_data(months: months)
    end
  end
end
```

## 8. Billing Portal

### 8.1 Stripe Customer Portal

```ruby
# app/controllers/billing_controller.rb
class BillingController < ApplicationController
  before_action :authenticate_user!
  
  def portal
    session = Stripe::BillingPortal::Session.create(
      customer: current_account.stripe_customer_id,
      return_url: billing_url
    )
    
    redirect_to session.url, allow_other_host: true
  rescue Stripe::InvalidRequestError => e
    redirect_to billing_path, alert: e.message
  end
  
  def index
    @subscription = current_account.active_subscription
    @invoices = Invoice.where(account: current_account).order(created_at: :desc).limit(12)
    @payment_methods = fetch_payment_methods
    @usage = UsageService.new(current_account).current_month_usage
    @plans = Plan.active.order(:price_cents)
  end
  
  private
  
  def fetch_payment_methods
    return [] unless current_account.stripe_customer_id
    
    Stripe::PaymentMethod.list(
      customer: current_account.stripe_customer_id,
      type: 'card'
    ).data
  rescue Stripe::StripeError
    []
  end
end
```

## 9. Team Management

```ruby
# app/models/membership.rb
class Membership < ApplicationRecord
  belongs_to :account
  belongs_to :user
  
  ROLES = %w[owner admin editor viewer].freeze
  
  validates :role, inclusion: { in: ROLES }
  validates :user_id, uniqueness: { scope: :account_id }
  
  def owner?  = role == 'owner'
  def admin?  = role.in?(%w[owner admin])
  def editor? = role.in?(%w[owner admin editor])
  
  def can_manage_billing? = owner?
  def can_invite_members? = admin?
  def can_manage_settings? = admin?
end

# app/models/invitation.rb
class Invitation < ApplicationRecord
  belongs_to :account
  belongs_to :invited_by, class_name: 'User'
  
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :role, inclusion: { in: Membership::ROLES }
  validates :token, presence: true, uniqueness: true
  
  before_create :generate_token
  
  scope :pending, -> { where(accepted_at: nil).where('expires_at > ?', Time.current) }
  
  def accept!(user)
    return false if expired? || accepted?
    
    account.memberships.create!(user: user, role: role)
    update!(accepted_at: Time.current, accepted_by: user)
    
    true
  end
  
  def expired?
    expires_at < Time.current
  end
  
  def accepted?
    accepted_at.present?
  end
  
  private
  
  def generate_token
    self.token = SecureRandom.urlsafe_base64(32)
    self.expires_at = 7.days.from_now
  end
end

# app/controllers/invitations_controller.rb
class InvitationsController < ApplicationController
  before_action :authenticate_user!, except: [:accept]
  
  def create
    authorize :invitation, :create?
    
    result = InvitationService.new(current_account).invite!(
      email: params[:email],
      role: params[:role],
      invited_by: current_user
    )
    
    if result[:success]
      redirect_to team_path, notice: "Invitation sent to #{params[:email]}"
    else
      redirect_to team_path, alert: result[:error]
    end
  end
  
  def accept
    @invitation = Invitation.pending.find_by(token: params[:token])
    
    if @invitation.nil?
      return redirect_to root_path, alert: "Invalid or expired invitation"
    end
    
    if current_user
      @invitation.accept!(current_user)
      redirect_to dashboard_path(subdomain: @invitation.account.subdomain),
                  notice: "You've joined #{@invitation.account.name}!"
    else
      session[:invitation_token] = params[:token]
      redirect_to new_user_registration_path
    end
  end
  
  def destroy
    @invitation = current_account.invitations.find(params[:id])
    authorize @invitation
    
    @invitation.destroy!
    redirect_to team_path, notice: "Invitation cancelled"
  end
end
```

## 10. Subscription Limits Enforcement

```ruby
# app/models/concerns/plan_limited.rb
module PlanLimited
  extend ActiveSupport::Concern
  
  included do
    before_create :check_plan_limits
  end
  
  class_methods do
    def plan_limit(feature_name)
      @plan_feature = feature_name
    end
  end
  
  private
  
  def check_plan_limits
    return unless self.class.instance_variable_get(:@plan_feature)
    
    feature = self.class.instance_variable_get(:@plan_feature)
    limit = account.active_subscription&.plan&.features&.dig(feature)
    return if limit.nil? || limit == Float::INFINITY
    
    current_count = account.send(self.class.table_name).count
    
    if current_count >= limit
      raise PlanLimitExceededError.new(
        feature: feature,
        limit: limit,
        current: current_count
      )
    end
  end
end

# app/models/project.rb
class Project < ApplicationRecord
  include PlanLimited
  
  plan_limit :max_projects
  
  belongs_to :account
end

# Custom error
class PlanLimitExceededError < StandardError
  attr_reader :feature, :limit, :current
  
  def initialize(feature:, limit:, current:)
    @feature = feature
    @limit = limit
    @current = current
    super("Plan limit exceeded for #{feature}: #{current}/#{limit}")
  end
end

# Handle ใน controllers
class ApplicationController < ActionController::Base
  rescue_from PlanLimitExceededError do |error|
    respond_to do |format|
      format.html { redirect_to upgrade_path, alert: "You've reached your #{error.feature.humanize} limit. Please upgrade your plan." }
      format.json { render json: { error: 'plan_limit_exceeded', feature: error.feature, limit: error.limit }, status: :payment_required }
    end
  end
end
```

---

## แบบฝึกหัดบทที่ 94

### แบบฝึกหัดที่ 1: Multi-tenant Data Isolation
**โจทย์:** สร้าง system ที่ isolate data ระหว่าง tenants อย่างสมบูรณ์

**เฉลย:**
```ruby
# app/models/concerns/tenant_scoped.rb
module TenantScoped
  extend ActiveSupport::Concern
  
  included do
    belongs_to :account
    
    before_validation :set_current_account
    
    default_scope { where(account_id: Current.account&.id) if Current.account }
    
    validates :account_id, presence: true
  end
  
  class_methods do
    def unscoped_by_tenant
      unscoped
    end
    
    def for_account(account)
      unscoped.where(account: account)
    end
  end
  
  private
  
  def set_current_account
    self.account ||= Current.account if Current.account
  end
end

# Tenant isolation middleware
class TenantIsolationMiddleware
  def initialize(app)
    @app = app
  end
  
  def call(env)
    request = ActionDispatch::Request.new(env)
    
    # Find tenant from subdomain or header
    tenant = resolve_tenant(request)
    
    if tenant
      ActsAsTenant.with_tenant(tenant) do
        @app.call(env)
      end
    else
      @app.call(env)
    end
  end
  
  private
  
  def resolve_tenant(request)
    # From subdomain
    subdomain = request.subdomain
    return Account.find_by(subdomain: subdomain) if subdomain.present? && subdomain != 'www'
    
    # From API key
    api_key = request.headers['X-API-Key']
    return ApiKey.active.find_by(token: api_key)&.account if api_key.present?
    
    nil
  end
end
```

### แบบฝึกหัดที่ 2: Stripe Trial ด้วย Credit Card Required
**โจทย์:** Implement trial ที่ require credit card แต่ไม่ charge จนกว่าจะหมด trial

**เฉลย:**
```ruby
# app/services/trial_subscription_service.rb
class TrialSubscriptionService
  TRIAL_DAYS = 14
  
  def initialize(account)
    @account = account
  end
  
  def start_trial!(payment_method_id:)
    customer = create_or_retrieve_customer!
    
    # Attach payment method แต่ยังไม่ charge
    attach_payment_method!(customer, payment_method_id)
    
    plan = Plan.find_by!(name: 'Pro')  # Default trial plan
    
    stripe_sub = Stripe::Subscription.create(
      customer: customer.id,
      items: [{ price: plan.stripe_price_id }],
      trial_period_days: TRIAL_DAYS,
      default_payment_method: payment_method_id,
      expand: ['latest_invoice.payment_intent'],
      metadata: { account_id: @account.id, type: 'trial' }
    )
    
    subscription = @account.subscriptions.create!(
      stripe_subscription_id: stripe_sub.id,
      plan: plan,
      status: 'trialing',
      trial_ends_at: TRIAL_DAYS.days.from_now,
      current_period_end: Time.at(stripe_sub.current_period_end)
    )
    
    # Schedule reminder emails
    TrialReminderJob.set(wait: (TRIAL_DAYS - 3).days).perform_later(@account.id)
    TrialReminderJob.set(wait: (TRIAL_DAYS - 1).days).perform_later(@account.id)
    
    { success: true, subscription: subscription, trial_ends_at: subscription.trial_ends_at }
  rescue Stripe::CardError => e
    { success: false, error: e.user_message }
  rescue Stripe::StripeError => e
    { success: false, error: e.message }
  end
  
  def convert_to_paid!(plan_id: nil)
    subscription = @account.active_subscription
    raise "No trial subscription" unless subscription&.status == 'trialing'
    
    if plan_id
      plan = Plan.find(plan_id)
      stripe_sub = Stripe::Subscription.retrieve(subscription.stripe_subscription_id)
      
      Stripe::Subscription.update(
        stripe_sub.id,
        items: [{ id: stripe_sub.items.data[0].id, price: plan.stripe_price_id }],
        trial_end: 'now'
      )
      
      subscription.update!(status: 'active', plan: plan)
    else
      # Just end trial and charge with existing plan
      stripe_sub = Stripe::Subscription.update(
        subscription.stripe_subscription_id,
        trial_end: 'now'
      )
      subscription.update!(status: 'active')
    end
    
    { success: true }
  rescue Stripe::StripeError => e
    { success: false, error: e.message }
  end
  
  private
  
  def create_or_retrieve_customer!
    if @account.stripe_customer_id.present?
      Stripe::Customer.retrieve(@account.stripe_customer_id)
    else
      customer = Stripe::Customer.create(
        email: @account.owner.email,
        name: @account.name,
        metadata: { account_id: @account.id }
      )
      @account.update!(stripe_customer_id: customer.id)
      customer
    end
  end
  
  def attach_payment_method!(customer, payment_method_id)
    Stripe::PaymentMethod.attach(payment_method_id, { customer: customer.id })
    Stripe::Customer.update(customer.id, {
      invoice_settings: { default_payment_method: payment_method_id }
    })
  end
end
```

### แบบฝึกหัดที่ 3: Usage Metering และ Overage Billing
**โจทย์:** Implement usage-based billing ที่คิดค่า overage

**เฉลย:**
```ruby
# app/services/overage_billing_service.rb
class OverageBillingService
  OVERAGE_RATES = {
    'api_calls' => 0.001,      # $0.001 per extra API call
    'storage_bytes' => 0.00005 # $0.05 per extra GB
  }.freeze
  
  def initialize(account)
    @account = account
  end
  
  def calculate_overages_for_month(year:, month:)
    plan = @account.active_subscription&.plan
    return {} unless plan
    
    overages = {}
    
    OVERAGE_RATES.each do |metric, rate|
      limit = plan.features[metric]
      next unless limit && limit != Float::INFINITY
      
      used = @account.usage_records
        .for_month(year, month)
        .where(metric: metric)
        .sum(:quantity)
      
      overage_amount = [used - limit, 0].max
      
      if overage_amount > 0
        unit = metric == 'storage_bytes' ? (overage_amount / 1.gigabyte.to_f) : overage_amount
        overages[metric] = {
          used: used,
          limit: limit,
          overage_units: unit,
          rate: rate,
          amount_cents: (unit * rate * 100).ceil
        }
      end
    end
    
    overages
  end
  
  def bill_overages!(year:, month:)
    overages = calculate_overages_for_month(year: year, month: month)
    return 0 if overages.empty?
    
    total_cents = overages.sum { |_, data| data[:amount_cents] }
    return 0 if total_cents.zero?
    
    # Create invoice item ใน Stripe
    overages.each do |metric, data|
      Stripe::InvoiceItem.create(
        customer: @account.stripe_customer_id,
        amount: data[:amount_cents],
        currency: 'usd',
        description: "#{metric.humanize} overage for #{Date.new(year, month).strftime('%B %Y')}: #{data[:overage_units].round(2)} units",
        metadata: {
          metric: metric,
          year: year,
          month: month,
          account_id: @account.id
        }
      )
    end
    
    Rails.logger.info "Billed #{total_cents} cents in overages for account #{@account.id}"
    total_cents
  rescue Stripe::StripeError => e
    Rails.logger.error "Failed to bill overages: #{e.message}"
    raise
  end
end

# Job สำหรับรัน monthly overage billing
class MonthlyOverageBillingJob < ApplicationJob
  queue_as :billing
  
  def perform
    last_month = 1.month.ago
    
    Account.with_active_subscription.find_each do |account|
      OverageBillingService.new(account).bill_overages!(
        year: last_month.year,
        month: last_month.month
      )
    rescue => e
      Rails.logger.error "Failed to bill overages for account #{account.id}: #{e.message}"
    end
  end
end
```

### แบบฝึกหัดที่ 4: Feature Flag Targeting Rules
**โจทย์:** สร้าง advanced feature flag ด้วย targeting rules

**เฉลย:**
```ruby
# app/services/feature_flag_service.rb
class FeatureFlagService
  class Rule
    attr_reader :type, :value, :operator
    
    def initialize(type:, value:, operator: :equals)
      @type = type
      @value = value
      @operator = operator
    end
    
    def matches?(account)
      case type
      when :plan
        matches_value?(account.active_subscription&.plan&.name)
      when :account_age_days
        account_age = (Time.current - account.created_at).to_i / 1.day
        compare_numeric(account_age)
      when :user_count
        compare_numeric(account.users.count)
      when :mrr
        subscription = account.active_subscription
        compare_numeric(subscription&.plan&.price_cents.to_i / 100.0)
      when :country
        matches_value?(account.country)
      when :custom_attribute
        matches_value?(account.metadata[value[:key]])
      end
    end
    
    private
    
    def matches_value?(actual)
      case operator
      when :equals then actual == value
      when :not_equals then actual != value
      when :in then Array(value).include?(actual)
      when :not_in then !Array(value).include?(actual)
      end
    end
    
    def compare_numeric(actual)
      return false unless actual.is_a?(Numeric)
      
      case operator
      when :greater_than then actual > value
      when :less_than then actual < value
      when :greater_than_or_equal then actual >= value
      when :less_than_or_equal then actual <= value
      when :equals then actual == value
      end
    end
  end
  
  def initialize(feature_name)
    @feature_name = feature_name
    @flag = FeatureFlag.find_by(name: feature_name)
  end
  
  def enabled_for?(account)
    return false unless @flag&.active?
    
    # Global override
    return true if @flag.enabled_globally?
    return false if @flag.disabled_globally?
    
    # Check targeting rules
    return true if matches_targeting_rules?(account)
    
    # Check percentage rollout
    return true if in_percentage_rollout?(account)
    
    # Check explicit account list
    @flag.enabled_account_ids.include?(account.id)
  end
  
  private
  
  def matches_targeting_rules?(account)
    return false if @flag.targeting_rules.empty?
    
    @flag.targeting_rules.all? do |rule_data|
      rule = Rule.new(
        type: rule_data['type'].to_sym,
        value: rule_data['value'],
        operator: rule_data.fetch('operator', 'equals').to_sym
      )
      rule.matches?(account)
    end
  end
  
  def in_percentage_rollout?(account)
    return false if @flag.rollout_percentage.zero?
    return true if @flag.rollout_percentage == 100
    
    # Consistent hash-based rollout
    hash = Digest::MD5.hexdigest("#{@feature_name}:#{account.id}").to_i(16)
    (hash % 100) < @flag.rollout_percentage
  end
end
```

### แบบฝึกหัดที่ 5: Customer Success Automation
**โจทย์:** สร้าง automated customer success workflows

**เฉลย:**
```ruby
# app/services/customer_health_service.rb
class CustomerHealthService
  HEALTH_SCORES = {
    engaged: 80..100,
    healthy: 60..79,
    at_risk: 40..59,
    churning: 0..39
  }.freeze
  
  def initialize(account)
    @account = account
  end
  
  def health_score
    scores = []
    
    # Login frequency (max 25 points)
    scores << login_score
    
    # Feature adoption (max 25 points)
    scores << feature_adoption_score
    
    # Data usage (max 25 points)
    scores << data_usage_score
    
    # Support interactions (max 25 points)
    scores << support_score
    
    scores.sum.clamp(0, 100)
  end
  
  def health_status
    score = health_score
    
    HEALTH_SCORES.find { |_, range| range.include?(score) }&.first || :churning
  end
  
  def at_risk?
    health_status.in?(%i[at_risk churning])
  end
  
  def recommendations
    recs = []
    
    if login_score < 10
      recs << { type: 'engagement', message: "Account hasn't been active in #{days_since_last_login} days", action: 'send_reengagement_email' }
    end
    
    if feature_adoption_score < 10
      recs << { type: 'adoption', message: "Only using #{used_features_count} of #{total_features_count} features", action: 'schedule_feature_walkthrough' }
    end
    
    recs
  end
  
  private
  
  def login_score
    last_login = @account.users.maximum(:last_sign_in_at)
    return 0 unless last_login
    
    days_ago = (Time.current - last_login).to_i / 1.day
    
    case days_ago
    when 0..1 then 25
    when 2..7 then 20
    when 8..14 then 15
    when 15..30 then 5
    else 0
    end
  end
  
  def feature_adoption_score
    total = FEATURES.count
    return 0 if total.zero?
    
    used = @account.usage_records
      .where('recorded_at > ?', 30.days.ago)
      .distinct.pluck(:metric).count
    
    ((used.to_f / total) * 25).round
  end
  
  def data_usage_score
    usage = UsageService.new(@account).current_month_usage
    
    api_percentage = usage.dig('api_calls', :percentage) || 0
    
    case api_percentage
    when 50..Float::INFINITY then 25
    when 25..49 then 15
    when 10..24 then 10
    when 1..9 then 5
    else 0
    end
  end
  
  def support_score
    # Penalize for many support tickets
    open_tickets = SupportTicket.where(account: @account, status: 'open').count
    
    case open_tickets
    when 0 then 25
    when 1..2 then 15
    when 3..5 then 5
    else 0
    end
  end
  
  def days_since_last_login
    last_login = @account.users.maximum(:last_sign_in_at)
    return Float::INFINITY unless last_login
    
    ((Time.current - last_login) / 1.day).to_i
  end
  
  def used_features_count
    @account.usage_records.where('recorded_at > ?', 30.days.ago).distinct.pluck(:metric).count
  end
  
  def total_features_count
    FEATURES.count
  end
end

# app/jobs/customer_health_check_job.rb
class CustomerHealthCheckJob < ApplicationJob
  queue_as :monitoring
  
  def perform
    Account.with_active_subscription.find_each do |account|
      service = CustomerHealthService.new(account)
      
      # Update health score
      account.update_column(:health_score, service.health_score)
      
      # Trigger interventions for at-risk accounts
      if service.at_risk?
        CustomerSuccessMailer.at_risk_alert(account, service.recommendations).deliver_later
      end
    end
  end
end
```

### แบบฝึกหัดที่ 6: Referral System
**โจทย์:** สร้าง referral program สำหรับ SaaS

**เฉลย:**
```ruby
# app/models/referral.rb
class Referral < ApplicationRecord
  belongs_to :referrer, class_name: 'Account'
  belongs_to :referred_account, class_name: 'Account', optional: true
  
  REWARD_MONTHS = 1
  REFERRAL_CODE_LENGTH = 8
  
  validates :code, presence: true, uniqueness: true, length: { is: REFERRAL_CODE_LENGTH }
  
  before_create :generate_code
  
  scope :pending, -> { where(status: 'pending') }
  scope :completed, -> { where(status: 'completed') }
  
  def reward_applied?
    rewarded_at.present?
  end
  
  def apply_reward!
    return if reward_applied?
    
    # Give referrer one month free
    SubscriptionService.new(referrer).extend_subscription!(months: REWARD_MONTHS)
    
    # Give referred account discount on first month
    ReferralDiscountService.new(referred_account).apply_first_month_discount!
    
    update!(status: 'completed', rewarded_at: Time.current)
    
    ReferralMailer.reward_applied(referrer, referred_account).deliver_later
  end
  
  private
  
  def generate_code
    loop do
      self.code = SecureRandom.alphanumeric(REFERRAL_CODE_LENGTH).upcase
      break unless Referral.exists?(code: code)
    end
  end
end

# app/controllers/referrals_controller.rb
class ReferralsController < ApplicationController
  before_action :authenticate_user!
  
  def show
    @referral = Referral.find_or_create_by(referrer: current_account)
    @referred_accounts = @referral.referred_accounts.includes(:subscriptions)
    @total_earned = @referred_accounts.where(referrals: { status: 'completed' }).count * Referral::REWARD_MONTHS
    
    render :show
  end
  
  def track
    referral_code = params[:code]
    referral = Referral.find_by(code: referral_code)
    
    if referral
      session[:referral_code] = referral_code
      cookies[:referral_code] = { value: referral_code, expires: 30.days.from_now }
    end
    
    redirect_to new_account_path
  end
end
```

### แบบฝึกหัดที่ 7: Invoice Generation
**โจทย์:** สร้าง PDF invoice generation system

**เฉลย:**
```ruby
# Gemfile
gem 'prawn'
gem 'prawn-table'

# app/services/invoice_pdf_service.rb
class InvoicePdfService
  def initialize(invoice)
    @invoice = invoice
    @account = invoice.account
  end
  
  def generate
    Prawn::Document.new do |pdf|
      pdf.font_families.update(
        'Helvetica' => {
          normal: 'Helvetica',
          bold: 'Helvetica-Bold'
        }
      )
      
      pdf.font 'Helvetica'
      
      render_header(pdf)
      render_billing_info(pdf)
      render_line_items(pdf)
      render_totals(pdf)
      render_footer(pdf)
    end.render
  end
  
  private
  
  def render_header(pdf)
    pdf.bounding_box([0, pdf.cursor], width: pdf.bounds.width) do
      pdf.text "INVOICE", size: 28, style: :bold, color: "2563EB"
      pdf.move_down 5
      pdf.text "Invoice ##{@invoice.invoice_number}", size: 14
      pdf.text "Date: #{@invoice.created_at.strftime('%B %d, %Y')}", size: 12
      pdf.text "Due: #{@invoice.due_date.strftime('%B %d, %Y')}", size: 12, color: "EF4444"
      
      pdf.bounding_box([pdf.bounds.width - 200, pdf.cursor + 60], width: 200) do
        pdf.text "MyApp Inc.", size: 14, style: :bold
        pdf.text "123 Main Street", size: 11
        pdf.text "San Francisco, CA 94105", size: 11
        pdf.text "billing@myapp.com", size: 11
      end
    end
    
    pdf.move_down 20
    pdf.stroke_horizontal_rule
    pdf.move_down 20
  end
  
  def render_billing_info(pdf)
    pdf.text "BILL TO", size: 10, color: "6B7280"
    pdf.move_down 5
    pdf.text @account.name, size: 14, style: :bold
    pdf.text @account.billing_email || @account.owner.email, size: 12
    pdf.text @account.billing_address if @account.billing_address.present?
    pdf.move_down 20
  end
  
  def render_line_items(pdf)
    headers = ["Description", "Period", "Quantity", "Unit Price", "Amount"]
    
    rows = @invoice.line_items.map do |item|
      [
        item.description,
        item.period_label,
        item.quantity.to_s,
        "$#{(item.unit_price_cents / 100.0).round(2)}",
        "$#{(item.amount_cents / 100.0).round(2)}"
      ]
    end
    
    pdf.table([headers] + rows,
      width: pdf.bounds.width,
      header: true,
      row_colors: ["FFFFFF", "F9FAFB"],
      cell_style: { size: 11, padding: [8, 10] }
    ) do
      row(0).font_style = :bold
      row(0).background_color = "1E3A5F"
      row(0).text_color = "FFFFFF"
      columns(-1).align = :right
      columns(2..4).align = :right
    end
    
    pdf.move_down 20
  end
  
  def render_totals(pdf)
    subtotal = @invoice.subtotal_cents / 100.0
    tax = @invoice.tax_cents / 100.0
    total = @invoice.total_cents / 100.0
    
    totals = [
      ["Subtotal", "$#{subtotal.round(2)}"],
      ["Tax (#{@invoice.tax_rate}%)", "$#{tax.round(2)}"],
      ["TOTAL", "$#{total.round(2)}"]
    ]
    
    pdf.bounding_box([pdf.bounds.width - 250, pdf.cursor], width: 250) do
      totals.each do |(label, amount)|
        is_total = label == "TOTAL"
        
        pdf.bounding_box([0, pdf.cursor], width: 250) do
          pdf.text label, size: is_total ? 14 : 11, style: is_total ? :bold : :normal
          pdf.bounding_box([150, pdf.cursor + (is_total ? 14 : 11)], width: 100) do
            pdf.text amount, size: is_total ? 14 : 11,
                     style: is_total ? :bold : :normal, align: :right
          end
        end
        
        pdf.move_down is_total ? 5 : 5
      end
    end
  end
  
  def render_footer(pdf)
    pdf.bounding_box([0, 60], width: pdf.bounds.width) do
      pdf.stroke_horizontal_rule
      pdf.move_down 10
      pdf.text "Thank you for your business!", size: 12, align: :center
      pdf.text "Questions? Contact us at billing@myapp.com", size: 10,
               align: :center, color: "6B7280"
    end
  end
end
```

### แบบฝึกหัดที่ 8: Upgrade Flow
**โจทย์:** สร้าง smooth upgrade flow ด้วย proration

**เฉลย:**
```ruby
# app/controllers/upgrades_controller.rb
class UpgradesController < ApplicationController
  before_action :authenticate_user!
  before_action :require_owner!
  
  def index
    @current_plan = current_account.active_subscription&.plan
    @plans = Plan.active.order(:price_cents)
    @comparison = plan_comparison
  end
  
  def preview
    new_plan = Plan.find(params[:plan_id])
    current_sub = current_account.active_subscription
    
    proration = calculate_proration(current_sub, new_plan)
    
    render json: {
      current_plan: current_sub.plan.name,
      new_plan: new_plan.name,
      proration_amount: proration[:amount_cents] / 100.0,
      next_billing_date: proration[:next_billing_date],
      description: proration[:description]
    }
  end
  
  def confirm
    new_plan = Plan.find(params[:plan_id])
    
    result = SubscriptionService.new(current_account).change_plan!(
      new_plan: new_plan,
      proration: params[:with_proration] != 'false'
    )
    
    if result[:success]
      track_upgrade_event(new_plan)
      redirect_to billing_path, notice: "Upgraded to #{new_plan.name}!"
    else
      redirect_to upgrades_path, alert: result[:error]
    end
  end
  
  private
  
  def calculate_proration(subscription, new_plan)
    return { amount_cents: 0, description: "No change" } unless subscription
    
    current_plan = subscription.plan
    period_end = subscription.current_period_end
    
    days_remaining = ((period_end - Time.current) / 1.day).ceil
    total_days = ((period_end - subscription.current_period_start) / 1.day).ceil
    
    current_unused = (current_plan.price_cents * days_remaining.to_f / total_days).ceil
    new_charge = (new_plan.price_cents * days_remaining.to_f / total_days).ceil
    
    proration = new_charge - current_unused
    
    {
      amount_cents: proration,
      next_billing_date: period_end,
      description: proration > 0 ?
        "You'll be charged $#{proration / 100.0} now for the remainder of this billing period" :
        "Credit will be applied to your next invoice"
    }
  end
  
  def plan_comparison
    features = %w[max_users max_projects storage_gb api_calls_per_month]
    plans = Plan.active.order(:price_cents)
    
    plans.each_with_object({}) do |plan, result|
      result[plan.name] = {
        price: plan.formatted_price,
        features: features.each_with_object({}) do |feature, fdata|
          value = plan.features[feature]
          fdata[feature] = value == Float::INFINITY ? 'Unlimited' : value.to_s
        end
      }
    end
  end
  
  def track_upgrade_event(new_plan)
    Analytics.track(
      user_id: current_user.id,
      event: 'Plan Upgraded',
      properties: {
        previous_plan: current_account.active_subscription&.plan&.name,
        new_plan: new_plan.name,
        new_plan_price: new_plan.price_cents / 100.0
      }
    )
  end
end
```

### แบบฝึกหัดที่ 9: SaaS Admin Dashboard
**โจทย์:** สร้าง admin dashboard ด้วย metrics และ account management

**เฉลย:**
```ruby
# app/controllers/admin/dashboard_controller.rb
module Admin
  class DashboardController < AdminController
    def index
      @metrics = SaasMetricsService.new.summary
      @recent_signups = Account.includes(:owner).order(created_at: :desc).limit(10)
      @at_risk_accounts = accounts_at_risk
      @recent_churns = recent_churned_accounts
    end
    
    def accounts
      @accounts = Account.includes(:owner, :subscriptions)
        .order(created_at: :desc)
      
      @accounts = @accounts.where('name ILIKE ?', "%#{params[:search]}%") if params[:search]
      @accounts = @accounts.joins(:subscriptions).where(subscriptions: { status: params[:status] }) if params[:status]
      
      @pagy, @accounts = pagy(@accounts)
    end
    
    def account_detail
      @account = Account.find(params[:id])
      @subscription = @account.active_subscription
      @usage = UsageService.new(@account).current_month_usage
      @health = CustomerHealthService.new(@account)
      @invoices = @account.invoices.order(created_at: :desc).limit(12)
      @audit_logs = AuditLog.for_resource(@account).recent.limit(20)
    end
    
    def impersonate
      account = Account.find(params[:account_id])
      user = account.owner
      
      # Store original session
      session[:admin_user_id] = current_user.id
      
      sign_in(:user, user)
      redirect_to root_url(subdomain: account.subdomain),
                  notice: "Impersonating #{user.name} on #{account.name}"
    end
    
    def stop_impersonating
      original_admin = User.find(session.delete(:admin_user_id))
      sign_in(:user, original_admin)
      redirect_to admin_root_path, notice: "Stopped impersonating"
    end
    
    private
    
    def accounts_at_risk
      Account.with_active_subscription
        .where('health_score < ?', 40)
        .order(health_score: :asc)
        .limit(10)
    end
    
    def recent_churned_accounts
      Account.joins(:subscriptions)
        .where(subscriptions: { status: 'cancelled', cancelled_at: 30.days.ago.. })
        .order('subscriptions.cancelled_at DESC')
        .limit(10)
    end
  end
end
```

### แบบฝึกหัดที่ 10: Email Sequence Automation
**โจทย์:** สร้าง automated email sequence engine

**เฉลย:**
```ruby
# app/models/email_sequence.rb
class EmailSequence < ApplicationRecord
  has_many :email_sequence_steps, dependent: :destroy
  has_many :email_sequence_enrollments, dependent: :destroy
  
  validates :name, presence: true
  validates :trigger, presence: true
  
  TRIGGERS = %w[
    account_created
    trial_started
    trial_ending_3_days
    subscription_activated
    subscription_cancelled
    plan_upgraded
    no_activity_7_days
    feature_not_used
  ].freeze
  
  validates :trigger, inclusion: { in: TRIGGERS }
  
  def enroll!(account, context: {})
    return if already_enrolled?(account)
    
    enrollment = enrollments.create!(
      account: account,
      context: context,
      started_at: Time.current
    )
    
    schedule_next_step(enrollment)
  end
  
  def already_enrolled?(account)
    enrollments.active.where(account: account).exists?
  end
  
  private
  
  def schedule_next_step(enrollment)
    first_step = steps.order(position: :asc).first
    return unless first_step
    
    SendSequenceEmailJob.set(wait: first_step.delay).perform_later(
      enrollment.id, first_step.id
    )
  end
end

# app/jobs/send_sequence_email_job.rb
class SendSequenceEmailJob < ApplicationJob
  queue_as :email_sequences
  
  def perform(enrollment_id, step_id)
    enrollment = EmailSequenceEnrollment.find_by(id: enrollment_id)
    return unless enrollment&.active?
    
    step = EmailSequenceStep.find_by(id: step_id)
    return unless step
    
    # Check exit conditions
    if exit_condition_met?(enrollment, step)
      enrollment.complete!
      return
    end
    
    # Send email
    SequenceMailer.send_step(
      enrollment.account,
      step: step,
      context: enrollment.context
    ).deliver_now
    
    # Record delivery
    enrollment.deliveries.create!(step: step, delivered_at: Time.current)
    
    # Schedule next step
    next_step = step.sequence.steps
      .where('position > ?', step.position)
      .order(position: :asc)
      .first
    
    if next_step
      self.class.set(wait: next_step.delay).perform_later(enrollment.id, next_step.id)
    else
      enrollment.complete!
    end
  end
  
  private
  
  def exit_condition_met?(enrollment, step)
    account = enrollment.account
    
    case enrollment.sequence.trigger
    when 'trial_started'
      account.subscribed? && !account.on_trial?
    when 'no_activity_7_days'
      account.users.where('last_sign_in_at > ?', 2.days.ago).exists?
    else
      false
    end
  end
end
```

### แบบฝึกหัดที่ 11: API Usage Dashboard
**โจทย์:** สร้าง API usage visualization สำหรับ accounts

**เฉลย:**
```ruby
# app/controllers/api_usage_controller.rb
class ApiUsageController < ApplicationController
  before_action :authenticate_user!
  
  def index
    @usage_data = ApiUsageAnalyzer.new(current_account).analyze(
      period: params[:period] || '30d'
    )
    
    respond_to do |format|
      format.html
      format.json { render json: @usage_data }
    end
  end
end

# app/services/api_usage_analyzer.rb
class ApiUsageAnalyzer
  PERIODS = {
    '24h' => 24.hours,
    '7d' => 7.days,
    '30d' => 30.days,
    '90d' => 90.days
  }.freeze
  
  def initialize(account)
    @account = account
  end
  
  def analyze(period:)
    duration = PERIODS[period] || 30.days
    start_time = duration.ago
    
    records = @account.usage_records
      .where(metric: 'api_calls')
      .where('recorded_at >= ?', start_time)
    
    {
      summary: calculate_summary(records),
      time_series: generate_time_series(records, period),
      top_endpoints: top_endpoints(records),
      status_distribution: status_distribution(records),
      response_times: response_time_percentiles(records)
    }
  end
  
  private
  
  def calculate_summary(records)
    {
      total_calls: records.sum(:quantity),
      avg_per_day: (records.sum(:quantity).to_f / 30).round(1),
      success_rate: calculate_success_rate(records),
      current_limit: @account.active_subscription&.plan&.features&.dig('api_calls_per_month'),
      percentage_used: calculate_percentage_used
    }
  end
  
  def generate_time_series(records, period)
    case period
    when '24h'
      records.group_by_hour(:recorded_at).sum(:quantity)
    when '7d'
      records.group_by_day(:recorded_at).sum(:quantity)
    else
      records.group_by_week(:recorded_at).sum(:quantity)
    end.map { |time, count| { time: time.iso8601, count: count } }
  end
  
  def top_endpoints(records)
    records
      .where("metadata->>'endpoint' IS NOT NULL")
      .group("metadata->>'endpoint'")
      .order(Arel.sql("COUNT(*) DESC"))
      .limit(10)
      .count
      .map { |endpoint, count| { endpoint: endpoint, count: count } }
  end
  
  def status_distribution(records)
    records
      .group("metadata->>'status_code'")
      .count
      .each_with_object({}) do |(code, count), result|
        category = case code.to_i
                   when 200..299 then 'success'
                   when 400..499 then 'client_error'
                   when 500..599 then 'server_error'
                   else 'other'
                   end
        result[category] = (result[category] || 0) + count
      end
  end
  
  def response_time_percentiles(records)
    times = records
      .where("metadata->>'response_time_ms' IS NOT NULL")
      .pluck(Arel.sql("(metadata->>'response_time_ms')::float"))
      .sort
    
    return {} if times.empty?
    
    {
      p50: percentile(times, 50),
      p90: percentile(times, 90),
      p99: percentile(times, 99),
      avg: (times.sum / times.size).round(2)
    }
  end
  
  def percentile(sorted_array, p)
    return 0 if sorted_array.empty?
    
    index = (p / 100.0 * sorted_array.size).ceil - 1
    sorted_array[[index, 0].max].round(2)
  end
  
  def calculate_success_rate(records)
    total = records.count
    return 100.0 if total.zero?
    
    successful = records
      .where("(metadata->>'status_code')::int BETWEEN 200 AND 299")
      .count
    
    (successful.to_f / total * 100).round(2)
  end
  
  def calculate_percentage_used
    current_month_usage = @account.usage_records.current_month
      .where(metric: 'api_calls').sum(:quantity)
    
    limit = @account.active_subscription&.plan&.features&.dig('api_calls_per_month')
    return 0 unless limit && limit != Float::INFINITY
    
    (current_month_usage.to_f / limit * 100).round(2)
  end
end
```

### แบบฝึกหัดที่ 12: Cancellation Flow
**โจทย์:** สร้าง smart cancellation flow ที่พยายาม retain customers

**เฉลย:**
```ruby
# app/controllers/cancellations_controller.rb
class CancellationsController < ApplicationController
  before_action :authenticate_user!
  before_action :require_owner!
  
  def new
    @subscription = current_account.active_subscription
    @retention_offer = determine_retention_offer
    @cancellation_reasons = [
      "Too expensive",
      "Missing features",
      "Switching to competitor",
      "No longer needed",
      "Technical issues",
      "Poor customer support",
      "Other"
    ]
  end
  
  def create
    reason = params[:reason]
    feedback = params[:feedback]
    
    # Record reason
    current_account.cancellation_feedbacks.create!(
      reason: reason,
      feedback: feedback,
      subscription: current_account.active_subscription
    )
    
    # Handle retention offer acceptance
    if params[:accept_offer].present?
      apply_retention_offer!(params[:accept_offer])
      redirect_to billing_path, notice: "We've applied a special offer to your account!"
      return
    end
    
    # Process cancellation
    SubscriptionService.new(current_account).cancel!(at_period_end: true)
    
    # Send cancellation email
    SubscriptionMailer.cancellation_confirmation(
      current_account,
      reason: reason,
      ends_at: current_account.active_subscription.current_period_end
    ).deliver_later
    
    # Trigger offboarding sequence
    EmailSequence.find_by(trigger: 'subscription_cancelled')
      &.enroll!(current_account, context: { reason: reason })
    
    redirect_to billing_path, notice: "Your subscription has been cancelled and will end on #{current_account.active_subscription.current_period_end.strftime('%B %d, %Y')}."
  end
  
  private
  
  def determine_retention_offer
    subscription = current_account.active_subscription
    return nil unless subscription
    
    account_age_months = ((Time.current - current_account.created_at) / 30.days).to_i
    
    # Long-time customer: offer free month
    if account_age_months >= 12
      {
        type: 'free_month',
        description: 'One month free on us',
        code: 'STAY_#{account_age_months}M'
      }
    # Active account: offer discount
    elsif current_account.health_score >= 60
      {
        type: 'discount',
        description: '50% off for 3 months',
        code: 'RETAIN50',
        discount_percent: 50,
        months: 3
      }
    # Downgrade offer
    elsif subscription.plan.name != 'Starter'
      {
        type: 'downgrade',
        description: 'Downgrade to Starter plan instead',
        plan: Plan.find_by(name: 'Starter')
      }
    end
  end
  
  def apply_retention_offer!(offer_type)
    case offer_type
    when 'free_month'
      SubscriptionService.new(current_account).extend_subscription!(months: 1)
    when 'discount'
      Stripe::Coupon.create(
        percent_off: 50,
        duration: :repeating,
        duration_in_months: 3
      ).tap do |coupon|
        Stripe::Subscription.update(
          current_account.active_subscription.stripe_subscription_id,
          coupon: coupon.id
        )
      end
    when 'downgrade'
      starter_plan = Plan.find_by(name: 'Starter')
      SubscriptionService.new(current_account).change_plan!(new_plan: starter_plan)
    end
  end
end
```

### แบบฝึกหัดที่ 13: SSO Integration
**โจทย์:** Implement SSO (SAML/OAuth) สำหรับ Enterprise accounts

**เฉลย:**
```ruby
# Gemfile
gem 'omniauth'
gem 'omniauth-google-oauth2'
gem 'ruby-saml'

# app/controllers/sso_controller.rb
class SsoController < ApplicationController
  skip_before_action :authenticate_user!
  
  # Google OAuth
  def google_oauth
    # Handled by OmniAuth middleware
  end
  
  def google_callback
    auth = request.env['omniauth.auth']
    
    user = User.from_omniauth(auth)
    
    if user
      sign_in_and_redirect user, event: :authentication
    else
      redirect_to root_path, alert: "Authentication failed"
    end
  end
  
  # SAML SSO
  def saml_sso
    account = Account.find_by(subdomain: params[:subdomain])
    unless account&.saml_enabled?
      return redirect_to login_path, alert: "SSO not configured for this account"
    end
    
    settings = saml_settings_for(account)
    request = OneLogin::RubySaml::Authrequest.new
    
    redirect_to request.create(settings), allow_other_host: true
  end
  
  def saml_acs
    account = Account.find(params[:account_id])
    settings = saml_settings_for(account)
    
    response = OneLogin::RubySaml::Response.new(
      params[:SAMLResponse],
      settings: settings
    )
    
    if response.is_valid?
      email = response.attributes['email']
      name = response.attributes['name'] || response.nameid
      
      user = User.find_or_create_from_saml(
        email: email,
        name: name,
        account: account
      )
      
      sign_in_and_redirect user
    else
      redirect_to login_path, alert: "SAML authentication failed: #{response.errors.join(', ')}"
    end
  end
  
  private
  
  def saml_settings_for(account)
    settings = OneLogin::RubySaml::Settings.new
    
    settings.assertion_consumer_service_url = saml_acs_url(account_id: account.id)
    settings.idp_sso_service_url = account.saml_idp_sso_url
    settings.idp_cert = account.saml_idp_cert
    settings.sp_entity_id = "https://#{account.subdomain}.myapp.com"
    settings.name_identifier_format = "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
    
    settings
  end
end
```

### แบบฝึกหัดที่ 14: Compliance และ Data Export
**โจทย์:** สร้าง GDPR-compliant data export system

**เฉลย:**
```ruby
# app/services/gdpr_export_service.rb
class GdprExportService
  EXPORT_TYPES = %w[personal_data activity_data all].freeze
  
  def initialize(account)
    @account = account
  end
  
  def generate_export(export_type: 'all')
    data = {}
    
    if export_type.in?(%w[personal_data all])
      data[:account] = export_account_data
      data[:users] = export_user_data
      data[:billing] = export_billing_data
    end
    
    if export_type.in?(%w[activity_data all])
      data[:activity_logs] = export_activity_logs
      data[:api_usage] = export_api_usage
    end
    
    data[:projects] = export_projects if export_type == 'all'
    
    {
      exported_at: Time.current.iso8601,
      account_id: @account.id,
      account_name: @account.name,
      data: data
    }
  end
  
  def create_export_package
    export_data = generate_export
    
    # Save as JSON file
    tmpfile = Tempfile.new(['gdpr_export', '.json'])
    tmpfile.write(JSON.pretty_generate(export_data))
    tmpfile.close
    
    # Zip the file
    zipfile = Tempfile.new(['gdpr_export', '.zip'])
    
    Zip::File.open(zipfile.path, Zip::File::CREATE) do |zip|
      zip.add('data_export.json', tmpfile.path)
    end
    
    zipfile
  ensure
    tmpfile&.unlink
  end
  
  def schedule_export(requested_by:)
    export_request = @account.gdpr_export_requests.create!(
      requested_by: requested_by,
      status: 'pending'
    )
    
    GdprExportJob.perform_later(@account.id, export_request.id)
    export_request
  end
  
  private
  
  def export_account_data
    {
      id: @account.id,
      name: @account.name,
      subdomain: @account.subdomain,
      created_at: @account.created_at.iso8601,
      plan: @account.active_subscription&.plan&.name
    }
  end
  
  def export_user_data
    @account.users.map do |user|
      {
        id: user.id,
        name: user.name,
        email: user.email,
        role: user.role,
        created_at: user.created_at.iso8601,
        last_sign_in_at: user.last_sign_in_at&.iso8601
      }
    end
  end
  
  def export_billing_data
    {
      invoices: @account.invoices.map { |inv|
        {
          id: inv.id,
          amount: inv.total_cents / 100.0,
          currency: 'USD',
          date: inv.created_at.iso8601,
          status: inv.status
        }
      },
      payment_methods: [] # Don't include full card data - just last4
    }
  end
  
  def export_activity_logs
    AuditLog.where(resource: @account)
      .order(created_at: :desc)
      .limit(10_000)
      .map do |log|
        {
          action: log.action,
          timestamp: log.created_at.iso8601,
          ip_address: log.metadata['ip_address']
        }
      end
  end
  
  def export_api_usage
    @account.usage_records
      .where('recorded_at > ?', 1.year.ago)
      .group_by_month(:recorded_at)
      .sum(:quantity)
      .map { |month, count| { month: month.strftime('%Y-%m'), api_calls: count } }
  end
  
  def export_projects
    @account.projects.map do |project|
      {
        id: project.id,
        name: project.name,
        created_at: project.created_at.iso8601,
        members_count: project.memberships.count
      }
    end
  end
end

# app/jobs/gdpr_export_job.rb
class GdprExportJob < ApplicationJob
  queue_as :gdpr
  
  def perform(account_id, export_request_id)
    account = Account.find(account_id)
    export_request = account.gdpr_export_requests.find(export_request_id)
    
    export_request.update!(status: 'processing')
    
    service = GdprExportService.new(account)
    package = service.create_export_package
    
    # Upload to S3
    s3_key = "gdpr_exports/#{account_id}/#{export_request_id}.zip"
    Aws::S3::Resource.new.bucket(ENV['AWS_BUCKET']).object(s3_key).upload_file(package.path)
    
    # Generate presigned URL (valid for 24 hours)
    download_url = Aws::S3::Presigner.new.presigned_url(
      :get_object,
      bucket: ENV['AWS_BUCKET'],
      key: s3_key,
      expires_in: 86400
    )
    
    export_request.update!(
      status: 'completed',
      download_url: download_url,
      completed_at: Time.current,
      expires_at: 24.hours.from_now
    )
    
    GdprMailer.export_ready(account, export_request).deliver_later
  rescue => e
    export_request.update!(status: 'failed', error_message: e.message)
    raise
  ensure
    package&.unlink
  end
end
```

### แบบฝึกหัดที่ 15: Complete SaaS Lifecycle
**โจทย์:** Setup complete SaaS lifecycle management

**เฉลย:**
```ruby
# app/services/account_lifecycle_service.rb
class AccountLifecycleService
  def initialize(account)
    @account = account
  end
  
  # Account creation + onboarding kickoff
  def on_account_created!
    # Start email sequence
    EmailSequence.find_by(trigger: 'account_created')
      &.enroll!(@account)
    
    # Initialize onboarding
    Onboarding.for_account(@account)
    
    # Setup initial feature flags
    setup_default_features!
    
    # Track in analytics
    Analytics.identify(
      user_id: @account.owner.id,
      traits: {
        name: @account.owner.name,
        email: @account.owner.email,
        company: @account.name,
        plan: 'free',
        created_at: @account.created_at
      }
    )
    
    Analytics.track(
      user_id: @account.owner.id,
      event: 'Account Created',
      properties: { account_id: @account.id, subdomain: @account.subdomain }
    )
  end
  
  # Trial started
  def on_trial_started!(subscription)
    EmailSequence.find_by(trigger: 'trial_started')
      &.enroll!(@account, context: { trial_ends_at: subscription.trial_ends_at })
    
    Analytics.track(
      user_id: @account.owner.id,
      event: 'Trial Started',
      properties: { plan: subscription.plan.name, trial_ends_at: subscription.trial_ends_at }
    )
    
    # Enable trial features
    %w[advanced_analytics ai_suggestions].each do |feature|
      @account.enable_feature!(feature)
    end
  end
  
  # Subscription activated (paid)
  def on_subscription_activated!(subscription)
    EmailSequence.find_by(trigger: 'subscription_activated')
      &.enroll!(@account)
    
    # Remove trial-specific features, enable plan features
    apply_plan_features!(subscription.plan)
    
    Analytics.track(
      user_id: @account.owner.id,
      event: 'Subscription Activated',
      properties: {
        plan: subscription.plan.name,
        mrr: subscription.plan.price_cents / 100.0,
        payment_method: 'card'
      }
    )
  end
  
  # Subscription cancelled
  def on_subscription_cancelled!(subscription)
    EmailSequence.find_by(trigger: 'subscription_cancelled')
      &.enroll!(@account, context: { ends_at: subscription.current_period_end })
    
    Analytics.track(
      user_id: @account.owner.id,
      event: 'Subscription Cancelled',
      properties: {
        plan: subscription.plan.name,
        reason: @account.cancellation_feedbacks.last&.reason
      }
    )
    
    # Downgrade features after period ends
    DowngradeFeaturesJob.set(wait_until: subscription.current_period_end)
      .perform_later(@account.id)
  end
  
  # Account deleted (GDPR)
  def on_account_deleted!(requested_by:)
    AnonymizeAccountDataJob.perform_later(@account.id, requested_by: requested_by.id)
    
    Analytics.track(
      user_id: requested_by.id,
      event: 'Account Deleted',
      properties: { account_id: @account.id }
    )
  end
  
  private
  
  def setup_default_features!
    # All accounts get these on start
    Flipper.enable_actor('new_dashboard', @account)
  end
  
  def apply_plan_features!(plan)
    # Clear existing feature flags
    FEATURES.keys.each { |f| Flipper.disable_actor(f, @account) }
    
    # Enable plan-specific features
    plan_features = {
      'starter' => %w[new_dashboard],
      'pro' => %w[new_dashboard advanced_analytics bulk_export],
      'enterprise' => FEATURES.keys  # All features
    }
    
    Array(plan_features[plan.name.downcase]).each do |feature|
      Flipper.enable_actor(feature, @account)
    end
  end
end
```

---

## สรุปบทที่ 94

ในบทนี้เราได้เรียนรู้:

1. **Multi-tenancy** - Row-level isolation ด้วย subdomain routing
2. **Stripe Subscriptions** - Plans, trials, webhooks, customer portal
3. **Usage-based Billing** - Metering, overage charges
4. **Feature Flags** - Flipper, gradual rollout, targeting rules
5. **Onboarding** - Step-by-step flows, email sequences
6. **SaaS Metrics** - MRR, ARR, churn, LTV, CAC
7. **Team Management** - Roles, invitations, SSO
8. **Compliance** - GDPR data export, audit logs
9. **Customer Success** - Health scoring, churn prevention, retention offers
10. **Complete Lifecycle** - Account creation ถึง deletion workflows
