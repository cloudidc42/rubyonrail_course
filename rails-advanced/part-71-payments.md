# Part 71: Payment Integration

## Steps 1541-1560

---

## Step 1541: Stripe Integration (Full Setup)

Stripe เป็น payment provider ที่ได้รับความนิยมสูงสุดสำหรับ web applications

### การติดตั้ง

```ruby
# Gemfile
gem 'stripe'

# config/initializers/stripe.rb
Stripe.api_key = ENV.fetch('STRIPE_SECRET_KEY')
Stripe.api_version = '2023-10-16'

# config/credentials.yml.enc (แก้ผ่าน rails credentials:edit)
stripe:
  publishable_key: pk_test_xxx
  secret_key: sk_test_xxx
  webhook_secret: whsec_xxx

# ENV setup
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
```

### Database Schema

```ruby
# migration
class AddStripeToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :stripe_customer_id, :string, index: true
  end
end

class CreatePayments < ActiveRecord::Migration[7.0]
  def change
    create_table :payments do |t|
      t.references :user, null: false, foreign_key: true
      t.references :order, null: false, foreign_key: true
      t.string :stripe_payment_intent_id, index: { unique: true }
      t.string :stripe_charge_id
      t.decimal :amount, precision: 10, scale: 2
      t.string :currency, default: 'thb'
      t.string :status, default: 'pending'
      t.string :payment_method_type  # card, promptpay, etc
      t.string :failure_code
      t.string :failure_message
      t.datetime :processed_at
      t.jsonb :stripe_metadata, default: {}
      t.timestamps
    end
  end
end

class CreateStripeEvents < ActiveRecord::Migration[7.0]
  def change
    create_table :stripe_events do |t|
      t.string :stripe_event_id, null: false, index: { unique: true }
      t.string :event_type
      t.jsonb :payload
      t.string :status, default: 'pending'
      t.datetime :processed_at
      t.timestamps
    end
  end
end
```

### User Model สำหรับ Stripe

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def stripe_customer
    if stripe_customer_id.present?
      Stripe::Customer.retrieve(stripe_customer_id)
    else
      create_stripe_customer
    end
  end
  
  def create_stripe_customer
    customer = Stripe::Customer.create(
      email: email,
      name: name,
      phone: phone,
      metadata: { user_id: id, environment: Rails.env }
    )
    
    update!(stripe_customer_id: customer.id)
    customer
  end
  
  def stripe_payment_methods
    return [] unless stripe_customer_id
    
    Stripe::PaymentMethod.list(
      customer: stripe_customer_id,
      type: 'card'
    ).data
  end
  
  def add_payment_method(payment_method_id)
    Stripe::PaymentMethod.attach(
      payment_method_id,
      customer: stripe_customer.id
    )
    
    # ตั้งเป็น default
    Stripe::Customer.update(
      stripe_customer_id,
      invoice_settings: { default_payment_method: payment_method_id }
    )
  end
end
```

---

## Step 1542: Payment Intents API

```ruby
# app/services/stripe_payment_service.rb
class StripePaymentService
  def self.create_payment_intent(order:, user:, payment_method_id: nil)
    new(order: order, user: user).create_payment_intent(payment_method_id)
  end
  
  def self.confirm_payment_intent(payment_intent_id:, payment_method_id:)
    new.confirm_payment_intent(payment_intent_id, payment_method_id)
  end
  
  def initialize(order: nil, user: nil)
    @order = order
    @user = user
  end
  
  def create_payment_intent(payment_method_id = nil)
    intent_params = {
      amount: (@order.total * 100).to_i,  # Stripe ใช้สตางค์
      currency: 'thb',
      customer: @user.stripe_customer.id,
      description: "Order #{@order.number}",
      metadata: {
        order_id: @order.id,
        user_id: @user.id,
        environment: Rails.env
      },
      automatic_payment_methods: {
        enabled: true
      }
    }
    
    if payment_method_id
      intent_params[:payment_method] = payment_method_id
      intent_params[:confirm] = true
    end
    
    Stripe::PaymentIntent.create(intent_params)
  rescue Stripe::InvalidRequestError => e
    raise PaymentError, e.message
  end
  
  def confirm_payment_intent(payment_intent_id, payment_method_id)
    Stripe::PaymentIntent.confirm(
      payment_intent_id,
      payment_method: payment_method_id
    )
  rescue Stripe::CardError => e
    raise CardDeclinedError, e.message
  end
end

# app/controllers/api/v1/payments_controller.rb
class Api::V1::PaymentsController < ApplicationController
  # สร้าง Payment Intent
  def create_intent
    order = current_user.orders.find(params[:order_id])
    
    payment_intent = StripePaymentService.create_payment_intent(
      order: order,
      user: current_user
    )
    
    # บันทึก payment record
    payment = Payment.create!(
      order: order,
      user: current_user,
      stripe_payment_intent_id: payment_intent.id,
      amount: order.total,
      status: 'pending'
    )
    
    render json: {
      client_secret: payment_intent.client_secret,
      payment_id: payment.id,
      publishable_key: ENV['STRIPE_PUBLISHABLE_KEY']
    }
  rescue PaymentError => e
    render json: { error: e.message }, status: :unprocessable_entity
  end
  
  # ยืนยันการชำระเงิน (client-side complete → notify server)
  def confirm
    payment = current_user.payments.find(params[:payment_id])
    payment_intent = Stripe::PaymentIntent.retrieve(payment.stripe_payment_intent_id)
    
    case payment_intent.status
    when 'succeeded'
      payment.update!(status: 'completed', processed_at: Time.current)
      payment.order.update!(status: 'paid', paid_at: Time.current)
      
      OrderMailer.confirmation(payment.order).deliver_later
      
      render json: { 
        status: 'success',
        order: OrderSerializer.new(payment.order).as_json
      }
    when 'requires_action'
      render json: {
        status: 'requires_action',
        client_secret: payment_intent.client_secret
      }
    else
      render json: { status: 'failed', error: payment_intent.last_payment_error&.message }, 
             status: :payment_required
    end
  end
end
```

---

## Step 1543: Stripe Checkout (Hosted)

```ruby
# app/services/stripe_checkout_service.rb
class StripeCheckoutService
  def self.create_session(order:, user:, success_url:, cancel_url:)
    line_items = order.items.map do |item|
      {
        price_data: {
          currency: 'thb',
          product_data: {
            name: item.product.name,
            description: item.product.description&.truncate(500),
            images: [item.product.main_image_url].compact,
            metadata: { product_id: item.product.id }
          },
          unit_amount: (item.unit_price * 100).to_i
        },
        quantity: item.quantity
      }
    end
    
    # เพิ่ม shipping cost ถ้ามี
    if order.shipping_cost > 0
      line_items << {
        price_data: {
          currency: 'thb',
          product_data: { name: 'ค่าจัดส่ง' },
          unit_amount: (order.shipping_cost * 100).to_i
        },
        quantity: 1
      }
    end
    
    session = Stripe::Checkout::Session.create(
      customer: user.stripe_customer.id,
      payment_method_types: ['card', 'promptpay'],
      line_items: line_items,
      mode: 'payment',
      success_url: "#{success_url}?session_id={CHECKOUT_SESSION_ID}",
      cancel_url: cancel_url,
      client_reference_id: order.id.to_s,
      metadata: {
        order_id: order.id,
        user_id: user.id
      },
      payment_intent_data: {
        description: "Order #{order.number}",
        metadata: {
          order_id: order.id,
          order_number: order.number
        }
      },
      expires_at: 30.minutes.from_now.to_i,
      locale: 'th'
    )
    
    session
  end
end

# Controller
class CheckoutsController < ApplicationController
  def create
    @order = current_user.orders.find(params[:order_id])
    
    session = StripeCheckoutService.create_session(
      order: @order,
      user: current_user,
      success_url: checkout_success_url,
      cancel_url: checkout_cancel_url
    )
    
    # บันทึก session ID
    @order.update!(stripe_checkout_session_id: session.id)
    
    redirect_to session.url, allow_other_host: true
  end
  
  def success
    session = Stripe::Checkout::Session.retrieve(params[:session_id])
    
    if session.payment_status == 'paid'
      order = Order.find(session.client_reference_id)
      order.update!(status: 'paid', paid_at: Time.current)
      
      redirect_to order_path(order), notice: 'ชำระเงินสำเร็จ!'
    else
      redirect_to cart_path, alert: 'การชำระเงินไม่สำเร็จ'
    end
  end
  
  def cancel
    redirect_to cart_path, alert: 'ยกเลิกการชำระเงิน'
  end
end
```

---

## Step 1544: Stripe Webhooks

```ruby
# app/controllers/webhooks/stripe_controller.rb
class Webhooks::StripeController < ApplicationController
  skip_before_action :authenticate_user!
  skip_before_action :verify_authenticity_token
  
  HANDLED_EVENTS = %w[
    payment_intent.succeeded
    payment_intent.payment_failed
    charge.dispute.created
    customer.subscription.created
    customer.subscription.updated
    customer.subscription.deleted
    invoice.payment_succeeded
    invoice.payment_failed
  ].freeze
  
  def receive
    payload = request.body.read
    sig_header = request.headers['Stripe-Signature']
    webhook_secret = ENV.fetch('STRIPE_WEBHOOK_SECRET')
    
    begin
      event = Stripe::Webhook.construct_event(
        payload,
        sig_header,
        webhook_secret
      )
    rescue JSON::ParserError
      return render json: { error: 'Invalid payload' }, status: :bad_request
    rescue Stripe::SignatureVerificationError
      return render json: { error: 'Invalid signature' }, status: :bad_request
    end
    
    # idempotency: บันทึก event และตรวจสอบว่าประมวลผลแล้วหรือยัง
    stripe_event = StripeEvent.find_or_initialize_by(stripe_event_id: event.id)
    
    if stripe_event.persisted? && stripe_event.processed?
      return render json: { message: 'Already processed' }, status: :ok
    end
    
    stripe_event.update!(
      event_type: event.type,
      payload: event.as_json,
      status: 'processing'
    )
    
    if HANDLED_EVENTS.include?(event.type)
      StripeWebhookProcessor.perform_later(stripe_event.id)
    end
    
    render json: { received: true }, status: :ok
  end
end

# app/jobs/stripe_webhook_processor.rb
class StripeWebhookProcessor < ApplicationJob
  queue_as :webhooks
  
  retry_on Stripe::StripeError, wait: :exponentially_longer, attempts: 5
  
  def perform(stripe_event_id)
    stripe_event = StripeEvent.find(stripe_event_id)
    event_data = stripe_event.payload.with_indifferent_access
    
    handler = "handle_#{event_data[:type].tr('.', '_')}"
    
    if respond_to?(handler, true)
      send(handler, event_data[:data][:object])
      stripe_event.update!(status: 'processed', processed_at: Time.current)
    else
      stripe_event.update!(status: 'skipped')
    end
  rescue => e
    stripe_event.update!(status: 'failed')
    raise e
  end
  
  private
  
  def handle_payment_intent_succeeded(payment_intent)
    payment = Payment.find_by(stripe_payment_intent_id: payment_intent[:id])
    return unless payment
    
    payment.update!(
      status: 'completed',
      stripe_charge_id: payment_intent[:latest_charge],
      processed_at: Time.current
    )
    
    order = payment.order
    order.update!(status: 'paid', paid_at: Time.current)
    
    OrderMailer.confirmation(order).deliver_later
    
    # อัพเดท inventory
    order.items.each do |item|
      item.product.decrement!(:stock, item.quantity)
    end
    
    Rails.logger.info "Payment succeeded for Order ##{order.number}"
  end
  
  def handle_payment_intent_payment_failed(payment_intent)
    payment = Payment.find_by(stripe_payment_intent_id: payment_intent[:id])
    return unless payment
    
    error = payment_intent[:last_payment_error]
    
    payment.update!(
      status: 'failed',
      failure_code: error&.dig(:code),
      failure_message: error&.dig(:message)
    )
    
    OrderMailer.payment_failed(payment.order).deliver_later
    
    Rails.logger.warn "Payment failed for Order ##{payment.order.number}: #{error&.dig(:message)}"
  end
  
  def handle_charge_dispute_created(dispute)
    charge = Stripe::Charge.retrieve(dispute[:charge])
    payment = Payment.find_by(stripe_charge_id: charge.id)
    return unless payment
    
    payment.order.update!(status: 'disputed')
    
    AdminMailer.dispute_created(payment, dispute).deliver_later
    
    Rails.logger.warn "Dispute created for charge #{charge.id}: #{dispute[:reason]}"
  end
  
  def handle_customer_subscription_created(subscription)
    user = User.find_by(stripe_customer_id: subscription[:customer])
    return unless user
    
    user.subscription.update!(
      stripe_subscription_id: subscription[:id],
      status: 'active',
      current_period_start: Time.at(subscription[:current_period_start]),
      current_period_end: Time.at(subscription[:current_period_end])
    )
  end
  
  def handle_invoice_payment_succeeded(invoice)
    subscription = Stripe::Subscription.retrieve(invoice[:subscription])
    user = User.find_by(stripe_customer_id: invoice[:customer])
    return unless user
    
    user.subscription.update!(
      status: 'active',
      current_period_end: Time.at(subscription[:current_period_end])
    )
    
    SubscriptionMailer.renewal_succeeded(user).deliver_later
  end
  
  def handle_invoice_payment_failed(invoice)
    user = User.find_by(stripe_customer_id: invoice[:customer])
    return unless user
    
    user.subscription.update!(status: 'past_due')
    
    SubscriptionMailer.renewal_failed(user).deliver_later
  end
end
```

---

## Step 1545: Subscriptions

```ruby
# สร้าง Stripe Products และ Prices ใน Stripe Dashboard
# หรือผ่าน code:
#
# product = Stripe::Product.create(name: 'Pro Plan')
# price = Stripe::Price.create(
#   product: product.id,
#   unit_amount: 79900,  # ฿799 
#   currency: 'thb',
#   recurring: { interval: 'month' }
# )

# app/services/subscription_service.rb
class SubscriptionService
  PLANS = {
    'basic' => {
      stripe_price_id: ENV['STRIPE_BASIC_PRICE_ID'],
      name: 'Basic',
      price: 299,
      features: ['5 projects', '1GB storage', 'Email support']
    },
    'pro' => {
      stripe_price_id: ENV['STRIPE_PRO_PRICE_ID'],
      name: 'Pro',
      price: 799,
      features: ['Unlimited projects', '10GB storage', 'Priority support', 'API access']
    },
    'business' => {
      stripe_price_id: ENV['STRIPE_BUSINESS_PRICE_ID'],
      name: 'Business',
      price: 1999,
      features: ['Everything in Pro', '100GB storage', 'Dedicated support', 'SLA']
    }
  }.freeze
  
  def self.subscribe(user:, plan:, payment_method_id:)
    new(user: user).subscribe(plan, payment_method_id)
  end
  
  def self.cancel(user:, immediately: false)
    new(user: user).cancel(immediately)
  end
  
  def self.change_plan(user:, new_plan:)
    new(user: user).change_plan(new_plan)
  end
  
  def initialize(user:)
    @user = user
  end
  
  def subscribe(plan, payment_method_id)
    plan_config = PLANS[plan]
    raise ArgumentError, "Invalid plan: #{plan}" unless plan_config
    
    # เพิ่ม payment method เป็น default
    Stripe::PaymentMethod.attach(
      payment_method_id,
      customer: @user.stripe_customer.id
    )
    
    Stripe::Customer.update(
      @user.stripe_customer_id,
      invoice_settings: { default_payment_method: payment_method_id }
    )
    
    # สร้าง subscription
    stripe_sub = Stripe::Subscription.create(
      customer: @user.stripe_customer_id,
      items: [{ price: plan_config[:stripe_price_id] }],
      payment_behavior: 'default_incomplete',
      expand: ['latest_invoice.payment_intent'],
      trial_period_days: @user.has_used_trial? ? nil : 14
    )
    
    # บันทึกลง database
    subscription = @user.create_subscription!(
      stripe_subscription_id: stripe_sub.id,
      plan: plan,
      status: stripe_sub.status,
      current_period_start: Time.at(stripe_sub.current_period_start),
      current_period_end: Time.at(stripe_sub.current_period_end),
      trial_end: stripe_sub.trial_end ? Time.at(stripe_sub.trial_end) : nil
    )
    
    {
      subscription: subscription,
      client_secret: stripe_sub.latest_invoice.payment_intent&.client_secret,
      status: stripe_sub.status
    }
  end
  
  def cancel(immediately = false)
    return unless @user.subscription
    
    stripe_sub = Stripe::Subscription.retrieve(@user.subscription.stripe_subscription_id)
    
    if immediately
      stripe_sub.delete
      @user.subscription.update!(status: 'cancelled', cancelled_at: Time.current)
    else
      # Cancel at period end
      Stripe::Subscription.update(
        stripe_sub.id,
        cancel_at_period_end: true
      )
      @user.subscription.update!(cancel_at_period_end: true)
    end
    
    SubscriptionMailer.cancellation(@user).deliver_later
  end
  
  def change_plan(new_plan)
    plan_config = PLANS[new_plan]
    raise ArgumentError, "Invalid plan: #{new_plan}" unless plan_config
    
    stripe_sub = Stripe::Subscription.retrieve(@user.subscription.stripe_subscription_id)
    
    # Prorate immediately
    Stripe::Subscription.update(
      stripe_sub.id,
      items: [{
        id: stripe_sub.items.data[0].id,
        price: plan_config[:stripe_price_id]
      }],
      proration_behavior: 'create_prorations'
    )
    
    @user.subscription.update!(plan: new_plan)
    
    SubscriptionMailer.plan_changed(@user, new_plan).deliver_later
  end
end
```

---

## Step 1546: Refunds

```ruby
# app/services/process_refund.rb
class ProcessRefund
  attr_reader :refund, :errors
  
  REFUND_REASONS = %w[
    duplicate
    fraudulent
    customer_request
    product_not_received
    product_unacceptable
    other
  ].freeze
  
  def self.call(**args)
    service = new(**args)
    service.call
    service
  end
  
  def initialize(order:, amount: nil, reason: 'customer_request', notes: nil, performed_by:)
    @order = order
    @amount = amount  # nil = full refund
    @reason = reason
    @notes = notes
    @performed_by = performed_by
    @errors = []
  end
  
  def call
    validate!
    return self if @errors.any?
    
    ActiveRecord::Base.transaction do
      @refund = process_stripe_refund
      create_refund_record
      update_order_status
      notify_customer
      log_refund
    end
    
    self
  rescue Stripe::InvalidRequestError => e
    @errors << "Stripe error: #{e.message}"
    self
  end
  
  def success?
    @errors.empty? && @refund.present?
  end
  
  private
  
  def validate!
    unless @order.paid?
      @errors << 'สามารถ refund ได้เฉพาะ orders ที่ชำระเงินแล้ว'
    end
    
    unless REFUND_REASONS.include?(@reason)
      @errors << "เหตุผลไม่ถูกต้อง: #{@reason}"
    end
    
    if @amount && @amount > @order.refundable_amount
      @errors << "จำนวนเงิน refund เกินกว่าที่สามารถ refund ได้ (#{@order.refundable_amount})"
    end
    
    if @order.refunded_at.present?
      @errors << 'Order นี้ถูก refund แล้ว'
    end
  end
  
  def process_stripe_refund
    refund_params = {
      payment_intent: @order.payment.stripe_payment_intent_id,
      reason: stripe_reason,
      metadata: {
        order_id: @order.id,
        performed_by: @performed_by.id,
        notes: @notes
      }
    }
    
    refund_params[:amount] = (@amount * 100).to_i if @amount
    
    Stripe::Refund.create(refund_params)
  end
  
  def stripe_reason
    case @reason
    when 'duplicate'       then 'duplicate'
    when 'fraudulent'      then 'fraudulent'
    else                        'requested_by_customer'
    end
  end
  
  def create_refund_record
    Refund.create!(
      order: @order,
      user: @order.user,
      performed_by: @performed_by,
      stripe_refund_id: @refund.id,
      amount: @amount || @order.total,
      reason: @reason,
      notes: @notes,
      status: @refund.status
    )
  end
  
  def update_order_status
    refunded_amount = @amount || @order.total
    
    if refunded_amount >= @order.total
      @order.update!(status: 'refunded', refunded_at: Time.current)
    else
      @order.update!(
        status: 'partially_refunded',
        refunded_amount: (@order.refunded_amount || 0) + refunded_amount
      )
    end
    
    # คืน stock
    @order.items.each do |item|
      item.product.increment!(:stock, item.quantity)
    end
  end
  
  def notify_customer
    OrderMailer.refund_processed(@order, @amount).deliver_later
  end
  
  def log_refund
    AuditLog.create!(
      action: 'refund_processed',
      user: @performed_by,
      resource: @order,
      metadata: {
        refund_amount: @amount || @order.total,
        reason: @reason,
        stripe_refund_id: @refund.id
      }
    )
  end
end
```

---

## Step 1547: Pay Gem (Unified Payments)

```ruby
# Gemfile
gem 'pay'
gem 'stripe'
# gem 'braintree'  # ถ้าต้องการ
# gem 'paddle_pay' # Paddle

# Terminal
rails pay:install:migrations
rails db:migrate

# config/initializers/pay.rb
Pay.setup do |config|
  config.business_name = 'MyApp'
  config.business_address = 'Bangkok, Thailand'
  config.application_name = 'MyApp'
  config.support_email = 'support@myapp.com'
  
  config.default_product_name = 'MyApp Subscription'
  config.default_plan_name = 'default'
  
  # Processors
  config.enabled_processors = [:stripe]
end

# app/models/user.rb
class User < ApplicationRecord
  pay_customer default_payment_processor: :stripe
  
  # Pay gem สร้าง methods:
  # user.payment_processor      => Pay::Stripe instance
  # user.payment_processor.subscribe(name: 'pro')
  # user.payment_processor.charge(1000)  # ฿10 in smallest currency unit
end

# การใช้งาน
user = User.find(1)

# สมัคร subscription
user.payment_processor.subscribe(name: 'pro', plan: 'price_xxx')

# charge ครั้งเดียว
user.payment_processor.charge(50000, description: 'One-time purchase')

# ยกเลิก subscription
user.payment_processor.subscription.cancel

# Reactivate subscription
user.payment_processor.subscription.resume

# Get current subscription
subscription = user.payment_processor.subscription
subscription.active?
subscription.on_trial?
subscription.cancelled?
subscription.ends_at
```

---

## Step 1548: Testing Payments

```ruby
# spec/services/stripe_payment_service_spec.rb
require 'rails_helper'

RSpec.describe StripePaymentService do
  let(:user) { create(:user) }
  let(:order) { create(:order, user: user, total: 1000) }
  
  before do
    # สร้าง Stripe customer ปลอม
    user.update!(stripe_customer_id: 'cus_test123')
  end
  
  describe '#create_payment_intent' do
    context 'successful' do
      before do
        stub_request(:post, 'https://api.stripe.com/v1/payment_intents')
          .to_return(
            status: 200,
            body: {
              id: 'pi_test123',
              client_secret: 'pi_test123_secret',
              amount: 100000,
              currency: 'thb',
              status: 'requires_payment_method'
            }.to_json,
            headers: { 'Content-Type' => 'application/json' }
          )
      end
      
      it 'creates payment intent' do
        result = StripePaymentService.create_payment_intent(
          order: order,
          user: user
        )
        
        expect(result.id).to eq('pi_test123')
        expect(result.amount).to eq(100000)
      end
    end
    
    context 'stripe error' do
      before do
        stub_request(:post, 'https://api.stripe.com/v1/payment_intents')
          .to_return(
            status: 400,
            body: {
              error: {
                type: 'invalid_request_error',
                message: 'No such customer'
              }
            }.to_json
          )
      end
      
      it 'raises PaymentError' do
        expect {
          StripePaymentService.create_payment_intent(
            order: order,
            user: user
          )
        }.to raise_error(PaymentError)
      end
    end
  end
end

# spec/jobs/stripe_webhook_processor_spec.rb
RSpec.describe StripeWebhookProcessor do
  let(:stripe_event) { create(:stripe_event, event_type: 'payment_intent.succeeded') }
  
  describe 'payment_intent.succeeded' do
    let(:payment) { create(:payment, stripe_payment_intent_id: 'pi_test123') }
    
    before do
      stripe_event.update!(
        payload: {
          type: 'payment_intent.succeeded',
          data: {
            object: {
              id: 'pi_test123',
              latest_charge: 'ch_test123',
              status: 'succeeded'
            }
          }
        }
      )
    end
    
    it 'updates payment status' do
      StripeWebhookProcessor.perform_now(stripe_event.id)
      
      expect(payment.reload.status).to eq('completed')
      expect(payment.stripe_charge_id).to eq('ch_test123')
    end
    
    it 'updates order status' do
      StripeWebhookProcessor.perform_now(stripe_event.id)
      
      expect(payment.order.reload.status).to eq('paid')
    end
    
    it 'sends confirmation email' do
      expect {
        StripeWebhookProcessor.perform_now(stripe_event.id)
      }.to change { ActionMailer::Base.deliveries.count }.by(1)
    end
    
    it 'marks event as processed' do
      StripeWebhookProcessor.perform_now(stripe_event.id)
      
      expect(stripe_event.reload.status).to eq('processed')
    end
  end
end

# การทดสอบ Webhook Endpoint
RSpec.describe 'Stripe Webhooks', type: :request do
  describe 'POST /webhooks/stripe' do
    let(:payload) do
      {
        id: 'evt_test123',
        type: 'payment_intent.succeeded',
        data: {
          object: { id: 'pi_test123' }
        }
      }.to_json
    end
    
    let(:signature) do
      timestamp = Time.current.to_i.to_s
      signed_payload = "#{timestamp}.#{payload}"
      secret = ENV['STRIPE_WEBHOOK_SECRET']
      
      hmac = OpenSSL::HMAC.hexdigest('SHA256', secret, signed_payload)
      "t=#{timestamp},v1=#{hmac}"
    end
    
    it 'returns 200 for valid webhook' do
      post '/webhooks/stripe',
        params: payload,
        headers: {
          'Content-Type' => 'application/json',
          'Stripe-Signature' => signature
        }
      
      expect(response).to have_http_status(:ok)
    end
    
    it 'returns 400 for invalid signature' do
      post '/webhooks/stripe',
        params: payload,
        headers: {
          'Content-Type' => 'application/json',
          'Stripe-Signature' => 'invalid_sig'
        }
      
      expect(response).to have_http_status(:bad_request)
    end
  end
end
```

---

## Step 1549: PromptPay Integration (Thailand)

```ruby
# PromptPay ผ่าน Stripe (Payment Methods)

# ใน frontend: สร้าง PaymentMethod สำหรับ PromptPay
# stripe.createPaymentMethod({ type: 'promptpay' })

# Backend: สร้าง PaymentIntent สำหรับ PromptPay
class PromptPayService
  def self.create_payment_intent(order:, user:)
    intent = Stripe::PaymentIntent.create(
      amount: (order.total * 100).to_i,
      currency: 'thb',
      payment_method_types: ['promptpay'],
      customer: user.stripe_customer.id,
      metadata: {
        order_id: order.id,
        user_id: user.id
      }
    )
    
    # Confirm immediately (PromptPay ไม่ต้อง card details)
    Stripe::PaymentIntent.confirm(
      intent.id,
      payment_method_data: { type: 'promptpay' }
    )
  end
end

# PromptPay จะ return QR code ใน next_action
# { type: 'promptpay_display_qr_code', promptpay_display_qr_code: { hosted_instructions_url, image_url_png } }
```

---

## Step 1550-1560: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: Complete Checkout Flow

```ruby
# สร้าง checkout flow ครบถ้วน

# app/controllers/checkouts_controller.rb
class CheckoutsController < ApplicationController
  before_action :authenticate_user!
  
  # GET /checkout
  def show
    @cart = current_user.cart
    @order = build_order_from_cart
    @payment_methods = current_user.stripe_payment_methods
  end
  
  # POST /checkout
  def create
    @order = current_user.orders.create!(order_params)
    
    case params[:payment_type]
    when 'stripe'
      handle_stripe_checkout
    when 'promptpay'
      handle_promptpay_checkout
    when 'cod'
      handle_cod_checkout
    end
  rescue ActiveRecord::RecordInvalid => e
    redirect_to checkout_path, alert: e.message
  end
  
  private
  
  def handle_stripe_checkout
    session = StripeCheckoutService.create_session(
      order: @order,
      user: current_user,
      success_url: checkout_success_url(order_id: @order.id),
      cancel_url: checkout_cancel_url
    )
    
    redirect_to session.url, allow_other_host: true
  end
  
  def handle_promptpay_checkout
    intent = PromptPayService.create_payment_intent(
      order: @order,
      user: current_user
    )
    
    @qr_code_url = intent.next_action.promptpay_display_qr_code.image_url_png
    @payment_intent_id = intent.id
    
    render :promptpay
  end
  
  def handle_cod_checkout
    @order.update!(payment_method: 'cod', status: 'processing')
    OrderMailer.cod_confirmation(@order).deliver_later
    redirect_to order_path(@order), notice: 'สั่งซื้อสำเร็จ! ชำระเงินเมื่อรับสินค้า'
  end
  
  def build_order_from_cart
    cart = current_user.cart
    Order.new(
      user: current_user,
      items: cart.items,
      subtotal: cart.subtotal,
      discount: 0,
      total: cart.subtotal
    )
  end
  
  def order_params
    params.require(:order).permit(
      :shipping_address_id, :coupon_code, :notes
    )
  end
end
```

### แบบฝึกหัดที่ 2: Subscription Management UI

```ruby
# app/controllers/subscriptions_controller.rb
class SubscriptionsController < ApplicationController
  def index
    @subscription = current_user.subscription
    @plans = SubscriptionService::PLANS
    @invoices = fetch_invoices
  end
  
  def create
    result = SubscriptionService.subscribe(
      user: current_user,
      plan: params[:plan],
      payment_method_id: params[:payment_method_id]
    )
    
    if result[:status] == 'active' || result[:status] == 'trialing'
      redirect_to subscriptions_path, notice: 'สมัคร subscription สำเร็จ!'
    else
      # ต้องการ 3D Secure authentication
      render json: {
        requires_action: true,
        client_secret: result[:client_secret]
      }
    end
  rescue => e
    redirect_to new_subscription_path, alert: e.message
  end
  
  def update
    SubscriptionService.change_plan(
      user: current_user,
      new_plan: params[:plan]
    )
    
    redirect_to subscriptions_path, notice: "เปลี่ยนเป็น #{params[:plan]} plan สำเร็จ"
  end
  
  def destroy
    SubscriptionService.cancel(
      user: current_user,
      immediately: params[:immediately] == 'true'
    )
    
    message = if params[:immediately] == 'true'
      'ยกเลิก subscription ทันที'
    else
      'ยกเลิก subscription เมื่อสิ้นสุดรอบบิล'
    end
    
    redirect_to subscriptions_path, notice: message
  end
  
  private
  
  def fetch_invoices
    return [] unless current_user.stripe_customer_id
    
    Stripe::Invoice.list(
      customer: current_user.stripe_customer_id,
      limit: 10
    ).data
  rescue Stripe::StripeError
    []
  end
end
```

### แบบฝึกหัดที่ 3: Admin Refund Interface

```ruby
# app/controllers/admin/refunds_controller.rb
class Admin::RefundsController < Admin::BaseController
  before_action :set_order
  
  def new
    @refund = Refund.new
    @max_refund = @order.refundable_amount
  end
  
  def create
    result = ProcessRefund.call(
      order: @order,
      amount: params[:amount].to_f > 0 ? params[:amount].to_f : nil,
      reason: params[:reason],
      notes: params[:notes],
      performed_by: current_admin_user
    )
    
    if result.success?
      redirect_to admin_order_path(@order), 
        notice: "Refund สำเร็จ: #{number_to_currency(result.refund.amount, unit: '฿')}"
    else
      @refund = Refund.new
      @max_refund = @order.refundable_amount
      flash.now[:alert] = result.errors.join(', ')
      render :new
    end
  end
  
  private
  
  def set_order
    @order = Order.find(params[:order_id])
  end
end
```

### แบบฝึกหัดที่ 4: Webhook Testing Helper

```ruby
# spec/support/stripe_webhooks_helper.rb
module StripeWebhooksHelper
  def stripe_webhook_headers(payload)
    timestamp = Time.current.to_i.to_s
    signed_payload = "#{timestamp}.#{payload}"
    secret = ENV['STRIPE_WEBHOOK_SECRET'] || 'test_secret'
    
    hmac = OpenSSL::HMAC.hexdigest('SHA256', secret, signed_payload)
    
    {
      'Content-Type' => 'application/json',
      'Stripe-Signature' => "t=#{timestamp},v1=#{hmac}"
    }
  end
  
  def send_stripe_webhook(event_type, object_data)
    payload = {
      id: "evt_#{SecureRandom.hex(8)}",
      type: event_type,
      data: { object: object_data }
    }.to_json
    
    post '/webhooks/stripe',
      params: payload,
      headers: stripe_webhook_headers(payload)
  end
end

RSpec.configure do |config|
  config.include StripeWebhooksHelper, type: :request
end

# ใช้งาน
RSpec.describe 'Stripe Webhooks' do
  it 'processes payment success' do
    payment = create(:payment, stripe_payment_intent_id: 'pi_test')
    
    send_stripe_webhook('payment_intent.succeeded', {
      id: 'pi_test',
      latest_charge: 'ch_test',
      status: 'succeeded'
    })
    
    expect(response).to have_http_status(:ok)
    expect(payment.reload.status).to eq('completed')
  end
end
```

### แบบฝึกหัดที่ 5: Payment Analytics

```ruby
# app/services/payment_analytics.rb
class PaymentAnalytics
  def initialize(period: 30.days)
    @period = period
    @start_date = period.ago
  end
  
  def summary
    {
      total_revenue: total_revenue,
      successful_payments: successful_payments,
      failed_payments: failed_payments,
      refunded_amount: refunded_amount,
      success_rate: success_rate,
      average_order_value: average_order_value,
      by_payment_method: by_payment_method,
      daily_revenue: daily_revenue
    }
  end
  
  private
  
  def total_revenue
    Payment.completed
      .where('processed_at >= ?', @start_date)
      .sum(:amount)
  end
  
  def successful_payments
    Payment.completed
      .where('processed_at >= ?', @start_date)
      .count
  end
  
  def failed_payments
    Payment.failed
      .where('created_at >= ?', @start_date)
      .count
  end
  
  def refunded_amount
    Refund.where('created_at >= ?', @start_date)
          .sum(:amount)
  end
  
  def success_rate
    total = successful_payments + failed_payments
    return 0 if total.zero?
    
    (successful_payments.to_f / total * 100).round(2)
  end
  
  def average_order_value
    Payment.completed
      .where('processed_at >= ?', @start_date)
      .average(:amount)
      &.round(2)
  end
  
  def by_payment_method
    Payment.completed
      .where('processed_at >= ?', @start_date)
      .group(:payment_method_type)
      .select(:payment_method_type, 'COUNT(*) as count', 'SUM(amount) as total')
      .map { |p| [p.payment_method_type, { count: p.count, total: p.total }] }
      .to_h
  end
  
  def daily_revenue
    Payment.completed
      .where('processed_at >= ?', @start_date)
      .group("DATE(processed_at)")
      .order("DATE(processed_at)")
      .sum(:amount)
  end
end
```

---

**จบ Part 71: Payment Integration**

*ในส่วนถัดไป Part 72 เราจะเรียนรู้เกี่ยวกับ Notifications System*
