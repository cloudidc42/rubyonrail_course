# Part 88: Distributed Systems กับ Ruby on Rails

## บทนำ

Distributed systems คือระบบที่ประกอบด้วย components หลายตัวที่ทำงานบน network ต่างกัน การออกแบบระบบแบบนี้มีความท้าทายเฉพาะตัว เช่น network failures, partial failures, consistency, และ latency บทนี้จะครอบคลุม patterns สำคัญสำหรับสร้าง distributed systems ที่ reliable ด้วย Ruby on Rails

---

## 1. Distributed Systems Fundamentals

### 1.1 CAP Theorem

```ruby
# CAP Theorem: ระบบ distributed ไม่สามารถมีครบทุกอย่างพร้อมกัน:
# - Consistency (C): ทุก node เห็นข้อมูลเหมือนกัน
# - Availability (A): ทุก request ได้รับ response
# - Partition Tolerance (P): ระบบทำงานได้แม้ network partition เกิดขึ้น

# ในความเป็นจริง: ต้องเลือกระหว่าง CP หรือ AP เมื่อเกิด partition

# CP Systems: PostgreSQL, HBase, Zookeeper
# - เลือก Consistency มากกว่า Availability
# - บาง nodes อาจไม่ตอบ request เมื่อเกิด partition

# AP Systems: Cassandra, DynamoDB, CouchDB
# - เลือก Availability มากกว่า Consistency
# - อาจคืน stale data แต่จะ available เสมอ

# Rails example: ออกแบบให้รองรับ AP
class UserPreferencesController < ApplicationController
  def show
    # ลอง read จาก cache ก่อน (AP: อาจ stale แต่ available)
    preferences = Rails.cache.fetch("user_#{params[:id]}_prefs", expires_in: 5.minutes) do
      # Fallback to database (CP)
      User.find(params[:id]).preferences
    end
    
    render json: preferences
  rescue ActiveRecord::RecordNotFound
    # Graceful degradation: return default preferences
    render json: UserPreferences.defaults
  end
end
```

### 1.2 PACELC Theorem

```ruby
# PACELC ขยาย CAP:
# - ถ้าเกิด Partition (P): เลือกระหว่าง Availability (A) กับ Consistency (C)
# - Else (E) ไม่มี partition: เลือกระหว่าง Latency (L) กับ Consistency (C)

# ตัวอย่าง trade-off ใน Rails:
class OrderService
  # Low latency path: อ่านจาก read replica (อาจ stale นิดหน่อย)
  def self.find_recent_orders(user_id)
    ApplicationRecord.connected_to(role: :reading) do
      Order.where(user_id: user_id)
           .recent
           .limit(20)
    end
  end
  
  # Consistency path: อ่านจาก primary ตรงๆ (latency สูงกว่า)
  def self.find_order_for_payment(order_id)
    ApplicationRecord.connected_to(role: :writing) do
      Order.find(order_id)
    end
  end
end
```

### 1.3 Fallacies of Distributed Computing

```ruby
# 8 ความเข้าใจผิดเกี่ยวกับ distributed computing:
# 1. Network is reliable
# 2. Latency is zero
# 3. Bandwidth is infinite
# 4. Network is secure
# 5. Topology doesn't change
# 6. There is one administrator
# 7. Transport cost is zero
# 8. Network is homogeneous

# การ handle network unreliability:
class ExternalApiService
  include HTTParty
  
  MAX_RETRIES = 3
  RETRY_DELAY = [1, 2, 4]  # exponential backoff (seconds)
  
  def self.fetch_data(endpoint)
    attempts = 0
    
    begin
      attempts += 1
      response = get(endpoint, timeout: 5)
      
      raise ServiceUnavailableError unless response.success?
      response.parsed_response
      
    rescue Net::OpenTimeout, Net::ReadTimeout, ServiceUnavailableError => e
      if attempts < MAX_RETRIES
        delay = RETRY_DELAY[attempts - 1] + rand(0.1..0.5)  # jitter
        sleep delay
        retry
      else
        Rails.logger.error "External API failed after #{MAX_RETRIES} attempts: #{e.message}"
        raise
      end
    end
  end
end
```

---

## 2. Circuit Breaker Pattern ด้วย Faraday

### 2.1 Circuit Breaker คืออะไร

```ruby
# Circuit Breaker ป้องกัน cascading failures
# States:
# - CLOSED: ทำงานปกติ
# - OPEN: ปฏิเสธ request ทันที (service ล้มเหลวมากเกินไป)
# - HALF-OPEN: ทดสอบว่า service กลับมาแล้วหรือยัง

# ติดตั้ง gems:
# gem 'faraday'
# gem 'faraday-retry'
# gem 'circuitbox'

# config/initializers/circuit_breaker.rb
require 'circuitbox'

Circuitbox.configure do |config|
  config.default_circuit_store = Moneta.new(:Redis, url: ENV['REDIS_URL'])
  
  # Default settings
  config.default_notifier = Circuitbox::Notifier::Log.new(Rails.logger)
end
```

### 2.2 Faraday กับ Circuit Breaker

```ruby
# app/services/payment_gateway_client.rb
class PaymentGatewayClient
  CIRCUIT_NAME = 'payment_gateway'
  
  def initialize
    @connection = Faraday.new(url: ENV['PAYMENT_GATEWAY_URL']) do |faraday|
      faraday.request :json
      faraday.response :json
      faraday.response :logger, Rails.logger, headers: false, bodies: false
      
      faraday.options.timeout = 5
      faraday.options.open_timeout = 2
      
      faraday.adapter Faraday.default_adapter
    end
    
    @circuit = Circuitbox.circuit(CIRCUIT_NAME, {
      exceptions: [Faraday::Error, Timeout::Error],
      volume_threshold: 5,      # จำนวน requests ต่ำสุดก่อน trip
      sleep_window: 30,         # วินาทีที่ circuit อยู่ใน OPEN state
      error_threshold: 50,      # % ของ failures ก่อน trip
      time_window: 60           # วินาทีสำหรับ rate calculation
    })
  end
  
  def charge(amount:, currency:, token:)
    @circuit.run do
      response = @connection.post('/charges', {
        amount: amount,
        currency: currency,
        source: token
      })
      
      if response.success?
        OpenStruct.new(
          success: true,
          charge_id: response.body['id'],
          status: response.body['status']
        )
      else
        raise PaymentError, response.body['error']['message']
      end
    end
  rescue Circuitbox::OpenCircuitError
    # Circuit ยัง OPEN อยู่ - fail fast
    Rails.logger.warn "Payment gateway circuit is OPEN"
    OpenStruct.new(
      success: false,
      error: 'payment_service_unavailable',
      retry_after: 30
    )
  rescue PaymentError => e
    Rails.logger.error "Payment failed: #{e.message}"
    OpenStruct.new(success: false, error: e.message)
  end
  
  def refund(charge_id:, amount: nil)
    @circuit.run do
      response = @connection.post("/charges/#{charge_id}/refunds", {
        amount: amount
      }.compact)
      
      raise PaymentError, response.body['error']['message'] unless response.success?
      
      OpenStruct.new(
        success: true,
        refund_id: response.body['id']
      )
    end
  rescue Circuitbox::OpenCircuitError
    OpenStruct.new(
      success: false,
      error: 'payment_service_unavailable',
      message: 'Cannot process refund at this time. Please try again later.'
    )
  end
end

# app/jobs/process_payment_job.rb
class ProcessPaymentJob < ApplicationJob
  queue_as :payments
  
  def perform(order_id)
    order = Order.find(order_id)
    client = PaymentGatewayClient.new
    
    result = client.charge(
      amount: order.total_cents,
      currency: order.currency,
      token: order.payment_token
    )
    
    if result.success
      order.update!(
        payment_status: 'paid',
        payment_id: result.charge_id
      )
      OrderMailer.payment_success(order).deliver_later
    else
      order.update!(payment_status: 'failed')
      
      if result.error == 'payment_service_unavailable'
        # Retry later when circuit may close
        self.class.set(wait: result.retry_after.seconds).perform_later(order_id)
      else
        OrderMailer.payment_failed(order, result.error).deliver_later
      end
    end
  end
end
```

### 2.3 Custom Circuit Breaker Implementation

```ruby
# app/lib/circuit_breaker.rb
class CircuitBreaker
  STATES = %i[closed open half_open].freeze
  
  class OpenCircuitError < StandardError; end
  
  attr_reader :name, :state, :failure_count, :last_failure_time
  
  def initialize(name, options = {})
    @name = name
    @state = :closed
    @failure_count = 0
    @success_count = 0
    @last_failure_time = nil
    
    @failure_threshold = options[:failure_threshold] || 5
    @success_threshold = options[:success_threshold] || 2
    @timeout = options[:timeout] || 30  # seconds
    @exceptions = options[:exceptions] || [StandardError]
    
    @mutex = Mutex.new
  end
  
  def call
    @mutex.synchronize do
      case @state
      when :open
        if Time.now - @last_failure_time >= @timeout
          transition_to(:half_open)
        else
          raise OpenCircuitError, "Circuit '#{@name}' is OPEN"
        end
      when :closed, :half_open
        # proceed
      end
    end
    
    begin
      result = yield
      on_success
      result
    rescue *@exceptions => e
      on_failure(e)
      raise
    end
  end
  
  def closed?; @state == :closed; end
  def open?; @state == :open; end
  def half_open?; @state == :half_open; end
  
  private
  
  def on_success
    @mutex.synchronize do
      @failure_count = 0
      
      if @state == :half_open
        @success_count += 1
        if @success_count >= @success_threshold
          transition_to(:closed)
        end
      end
    end
  end
  
  def on_failure(exception)
    @mutex.synchronize do
      @failure_count += 1
      @last_failure_time = Time.now
      @success_count = 0
      
      if @state == :half_open || (@state == :closed && @failure_count >= @failure_threshold)
        transition_to(:open)
      end
    end
  end
  
  def transition_to(new_state)
    Rails.logger.warn "Circuit '#{@name}' transitioning from #{@state} to #{new_state}"
    @state = new_state
    @success_count = 0
    @failure_count = 0 if new_state == :closed
  end
end

# การใช้งาน
INVENTORY_CIRCUIT = CircuitBreaker.new(
  'inventory_service',
  failure_threshold: 3,
  timeout: 20,
  exceptions: [Faraday::Error, Timeout::Error]
)

class InventoryService
  def self.check_stock(product_id)
    INVENTORY_CIRCUIT.call do
      # เรียก external inventory service
      response = Faraday.get("#{ENV['INVENTORY_URL']}/products/#{product_id}/stock")
      JSON.parse(response.body)
    end
  rescue CircuitBreaker::OpenCircuitError
    # Fallback: ใช้ cached data
    Rails.cache.read("inventory_#{product_id}") || { available: true, quantity: nil }
  end
end
```

---

## 3. Distributed Tracing ด้วย OpenTelemetry

### 3.1 Setup OpenTelemetry

```ruby
# Gemfile
gem 'opentelemetry-sdk'
gem 'opentelemetry-exporter-otlp'
gem 'opentelemetry-instrumentation-rails'
gem 'opentelemetry-instrumentation-active_record'
gem 'opentelemetry-instrumentation-sidekiq'
gem 'opentelemetry-instrumentation-faraday'
gem 'opentelemetry-instrumentation-redis'

# config/initializers/opentelemetry.rb
require 'opentelemetry/sdk'
require 'opentelemetry/exporter/otlp'
require 'opentelemetry/instrumentation/rails'
require 'opentelemetry/instrumentation/active_record'

OpenTelemetry::SDK.configure do |c|
  c.service_name = ENV.fetch('OTEL_SERVICE_NAME', 'my-rails-app')
  c.service_version = ENV.fetch('APP_VERSION', '1.0.0')
  
  # Export to Jaeger หรือ Honeycomb หรือ Datadog
  c.add_span_processor(
    OpenTelemetry::SDK::Trace::Export::BatchSpanProcessor.new(
      OpenTelemetry::Exporter::OTLP::Exporter.new(
        endpoint: ENV.fetch('OTLP_ENDPOINT', 'http://localhost:4318'),
        headers: {
          'Authorization' => "Bearer #{ENV['OTEL_API_KEY']}"
        }
      )
    )
  )
  
  # Auto-instrument
  c.use 'OpenTelemetry::Instrumentation::Rails'
  c.use 'OpenTelemetry::Instrumentation::ActiveRecord'
  c.use 'OpenTelemetry::Instrumentation::Sidekiq'
  c.use 'OpenTelemetry::Instrumentation::Faraday'
  c.use 'OpenTelemetry::Instrumentation::Redis'
end
```

### 3.2 Manual Instrumentation

```ruby
# app/services/order_processing_service.rb
class OrderProcessingService
  TRACER = OpenTelemetry.tracer_provider.tracer('order-service', '1.0')
  
  def self.process(order_id)
    TRACER.in_span('order.process', attributes: { 'order.id' => order_id }) do |span|
      order = find_order(order_id, span)
      validate_inventory(order, span)
      charge_payment(order, span)
      fulfill_order(order, span)
    end
  end
  
  private
  
  def self.find_order(order_id, parent_span)
    TRACER.in_span('order.find', attributes: { 'order.id' => order_id }) do |span|
      order = Order.includes(:items, :user).find(order_id)
      
      span.set_attribute('order.items.count', order.items.count)
      span.set_attribute('order.total_cents', order.total_cents)
      span.set_attribute('user.id', order.user_id)
      
      order
    end
  end
  
  def self.validate_inventory(order, parent_span)
    TRACER.in_span('inventory.validate') do |span|
      insufficient_items = []
      
      order.items.each do |item|
        available = InventoryService.check_stock(item.product_id)
        
        if available['quantity'] && available['quantity'] < item.quantity
          insufficient_items << item.product_id
        end
      end
      
      if insufficient_items.any?
        span.add_event('inventory.insufficient', attributes: {
          'product_ids' => insufficient_items.join(',')
        })
        span.status = OpenTelemetry::Trace::Status.error('Insufficient inventory')
        
        raise InsufficientInventoryError, "Products unavailable: #{insufficient_items}"
      end
      
      span.add_event('inventory.validated', attributes: {
        'items.count' => order.items.count
      })
    end
  end
  
  def self.charge_payment(order, parent_span)
    TRACER.in_span('payment.charge', attributes: {
      'payment.amount' => order.total_cents,
      'payment.currency' => order.currency
    }) do |span|
      client = PaymentGatewayClient.new
      result = client.charge(
        amount: order.total_cents,
        currency: order.currency,
        token: order.payment_token
      )
      
      if result.success
        span.set_attribute('payment.charge_id', result.charge_id)
        span.set_attribute('payment.status', result.status)
        
        order.update!(
          payment_status: 'paid',
          payment_id: result.charge_id
        )
      else
        span.status = OpenTelemetry::Trace::Status.error(result.error)
        span.record_exception(PaymentError.new(result.error))
        raise PaymentError, result.error
      end
    end
  end
  
  def self.fulfill_order(order, parent_span)
    TRACER.in_span('order.fulfill') do |span|
      order.update!(status: 'processing')
      
      FulfillmentJob.perform_later(order.id)
      
      span.set_attribute('fulfillment.job_id', order.id)
      span.add_event('order.fulfilled')
    end
  end
end
```

### 3.3 Context Propagation

```ruby
# Propagate trace context ผ่าน HTTP headers
class TracedHttpClient
  TRACER = OpenTelemetry.tracer_provider.tracer('http-client')
  
  def self.get(url, headers: {})
    TRACER.in_span("HTTP GET #{url}") do |span|
      # Inject trace context เข้า headers
      propagator_headers = {}
      OpenTelemetry.propagation.inject(propagator_headers)
      
      merged_headers = headers.merge(propagator_headers)
      
      span.set_attribute('http.method', 'GET')
      span.set_attribute('http.url', url)
      
      begin
        response = Faraday.get(url, nil, merged_headers)
        
        span.set_attribute('http.status_code', response.status)
        span.status = response.success? ? 
          OpenTelemetry::Trace::Status.ok :
          OpenTelemetry::Trace::Status.error("HTTP #{response.status}")
        
        response
      rescue Faraday::Error => e
        span.record_exception(e)
        span.status = OpenTelemetry::Trace::Status.error(e.message)
        raise
      end
    end
  end
end

# Propagate ผ่าน Sidekiq jobs
class TracedApplicationJob < ApplicationJob
  TRACER = OpenTelemetry.tracer_provider.tracer('sidekiq')
  
  before_perform do |job|
    # Extract trace context จาก job arguments
    if job.arguments.last.is_a?(Hash) && job.arguments.last[:trace_context]
      @trace_ctx = OpenTelemetry.propagation.extract(
        job.arguments.last[:trace_context]
      )
    end
  end
  
  def self.perform_later(*args)
    # Inject trace context ใน job arguments
    trace_context = {}
    OpenTelemetry.propagation.inject(trace_context)
    
    super(*args, trace_context: trace_context)
  end
end
```

---

## 4. Saga Pattern สำหรับ Microservices Transactions

### 4.1 Saga Pattern คืออะไร

```ruby
# Saga แก้ปัญหา distributed transactions โดยใช้ sequence ของ local transactions
# แต่ละ transaction มี compensating transaction เมื่อต้องการ rollback

# 2 แบบของ Saga:
# 1. Choreography: services สื่อสารกันผ่าน events (decoupled)
# 2. Orchestration: มี orchestrator ควบคุม flow (centralized)

# เราจะสร้าง Orchestration-based Saga สำหรับ order processing
```

### 4.2 Saga Orchestrator

```ruby
# app/sagas/create_order_saga.rb
class CreateOrderSaga
  include ActiveModel::Model
  
  STEPS = [
    :reserve_inventory,
    :charge_payment,
    :create_shipment,
    :send_confirmation
  ].freeze
  
  attr_accessor :order_id, :current_step, :status, :error_message
  
  def initialize(order_id)
    @order_id = order_id
    @current_step = nil
    @status = :pending
    @compensations = []
  end
  
  def execute
    ActiveRecord::Base.transaction do
      saga_record = SagaExecution.create!(
        saga_type: self.class.name,
        order_id: @order_id,
        status: 'running',
        steps_completed: []
      )
      
      begin
        STEPS.each do |step|
          saga_record.update!(current_step: step.to_s)
          
          execute_step(step)
          
          saga_record.update!(
            steps_completed: saga_record.steps_completed + [step.to_s]
          )
        end
        
        saga_record.update!(status: 'completed')
        @status = :completed
        
      rescue => error
        saga_record.update!(
          status: 'compensating',
          error_message: error.message
        )
        
        compensate(saga_record)
        
        saga_record.update!(status: 'compensated')
        @status = :failed
        @error_message = error.message
        
        raise ActiveRecord::Rollback
      end
    end
    
    @status == :completed
  end
  
  private
  
  def execute_step(step)
    case step
    when :reserve_inventory
      result = InventoryService.reserve(order_id: @order_id)
      raise SagaError, result.error unless result.success?
      
      @compensations.push({
        step: :release_inventory,
        action: -> { InventoryService.release(order_id: @order_id) }
      })
      
    when :charge_payment
      order = Order.find(@order_id)
      client = PaymentGatewayClient.new
      result = client.charge(
        amount: order.total_cents,
        currency: order.currency,
        token: order.payment_token
      )
      
      raise SagaError, result.error unless result.success?
      
      order.update!(payment_id: result.charge_id)
      
      @compensations.push({
        step: :refund_payment,
        action: -> { PaymentGatewayClient.new.refund(charge_id: result.charge_id) }
      })
      
    when :create_shipment
      result = ShippingService.create_shipment(order_id: @order_id)
      raise SagaError, result.error unless result.success?
      
      Order.find(@order_id).update!(shipment_id: result.shipment_id)
      
      @compensations.push({
        step: :cancel_shipment,
        action: -> { ShippingService.cancel_shipment(shipment_id: result.shipment_id) }
      })
      
    when :send_confirmation
      order = Order.includes(:user).find(@order_id)
      OrderMailer.confirmation(order).deliver_later
    end
  end
  
  def compensate(saga_record)
    # Execute compensations ย้อนกลับ (LIFO order)
    @compensations.reverse.each do |compensation|
      begin
        compensation[:action].call
        saga_record.update!(
          compensations_completed: (saga_record.compensations_completed || []) + [compensation[:step].to_s]
        )
      rescue => e
        Rails.logger.error "Compensation failed for #{compensation[:step]}: #{e.message}"
        # Log แต่ continue กับ compensations อื่น
      end
    end
  end
end

# db/migrate/xxx_create_saga_executions.rb
class CreateSagaExecutions < ActiveRecord::Migration[7.1]
  def change
    create_table :saga_executions do |t|
      t.string :saga_type, null: false
      t.integer :order_id
      t.string :status, null: false, default: 'running'
      t.string :current_step
      t.jsonb :steps_completed, default: []
      t.jsonb :compensations_completed, default: []
      t.text :error_message
      t.timestamps
    end
    
    add_index :saga_executions, :order_id
    add_index :saga_executions, :status
  end
end
```

### 4.3 Choreography-based Saga

```ruby
# ใช้ events แทน direct calls

# app/sagas/order_saga_choreography.rb

# Step 1: Order Service publishes event
class OrdersController < ApplicationController
  def create
    order = Order.create!(order_params)
    
    # Publish event สำหรับ saga
    OrderCreatedEvent.publish(order_id: order.id, items: order.items_data)
    
    render json: { order_id: order.id, status: 'pending' }
  end
end

# app/events/order_created_event.rb
class OrderCreatedEvent < ApplicationEvent
  def self.publish(payload)
    ApplicationEventBus.publish('orders.created', payload)
  end
end

# Inventory Service subscribes และ processes
class InventoryReservationHandler
  def self.call(event_data)
    order_id = event_data['order_id']
    items = event_data['items']
    
    # Try to reserve
    reserved = items.all? do |item|
      Inventory.reserve(product_id: item['product_id'], quantity: item['quantity'])
    end
    
    if reserved
      InventoryReservedEvent.publish(order_id: order_id)
    else
      InventoryReservationFailedEvent.publish(
        order_id: order_id,
        reason: 'insufficient_stock'
      )
    end
  end
end

# Payment Service subscribes to InventoryReserved
class PaymentProcessingHandler
  def self.call(event_data)
    order_id = event_data['order_id']
    order = Order.find(order_id)
    
    result = PaymentGatewayClient.new.charge(
      amount: order.total_cents,
      currency: order.currency,
      token: order.payment_token
    )
    
    if result.success
      PaymentCompletedEvent.publish(
        order_id: order_id,
        payment_id: result.charge_id
      )
    else
      PaymentFailedEvent.publish(
        order_id: order_id,
        reason: result.error
      )
    end
  end
end

# Event Bus
class ApplicationEventBus
  def self.publish(topic, payload)
    # Publish to message broker (Redis/Kafka/RabbitMQ)
    encoded = { topic: topic, payload: payload, timestamp: Time.now.iso8601 }.to_json
    
    redis = Redis.new(url: ENV['REDIS_URL'])
    redis.xadd("events:#{topic}", '*', 'data', encoded)
    
    # Also store for audit trail
    DomainEvent.create!(
      topic: topic,
      payload: payload,
      published_at: Time.now
    )
  end
  
  def self.subscribe(topic, &handler)
    Thread.new do
      redis = Redis.new(url: ENV['REDIS_URL'])
      last_id = '$'
      
      loop do
        events = redis.xread({ "events:#{topic}" => last_id }, count: 10, block: 1000)
        
        events&.each do |stream, entries|
          entries.each do |id, data|
            begin
              payload = JSON.parse(data['data'])
              handler.call(payload['payload'])
              last_id = id
            rescue => e
              Rails.logger.error "Error processing event #{id}: #{e.message}"
            end
          end
        end
      end
    end
  end
end
```

---

## 5. Outbox Pattern สำหรับ Reliable Messaging

### 5.1 ปัญหาที่ Outbox แก้

```ruby
# ปัญหา: ถ้า database commit สำเร็จแต่ publish event ล้มเหลว
# ข้อมูลจะ inconsistent

# ไม่ดี: สองอย่างแยกกัน อาจล้มเหลวระหว่างกลาง
class BadOrderService
  def create_order(params)
    order = Order.create!(params)
    
    # ถ้า event bus ล้มเหลวที่นี่
    # order ถูกสร้างแล้ว แต่ downstream services ไม่รู้
    EventBus.publish('order.created', order_id: order.id)
    
    order
  end
end
```

### 5.2 Outbox Implementation

```ruby
# db/migrate/xxx_create_outbox_events.rb
class CreateOutboxEvents < ActiveRecord::Migration[7.1]
  def change
    create_table :outbox_events do |t|
      t.string :aggregate_type, null: false  # 'Order', 'User', etc.
      t.integer :aggregate_id, null: false
      t.string :event_type, null: false      # 'order.created', etc.
      t.jsonb :payload, null: false, default: {}
      t.string :status, null: false, default: 'pending'
      t.integer :attempts, null: false, default: 0
      t.datetime :last_attempt_at
      t.text :last_error
      t.datetime :published_at
      t.timestamps
    end
    
    add_index :outbox_events, :status
    add_index :outbox_events, [:status, :created_at]
    add_index :outbox_events, :aggregate_type
  end
end

# app/models/outbox_event.rb
class OutboxEvent < ApplicationRecord
  enum status: {
    pending: 'pending',
    processing: 'processing',
    published: 'published',
    failed: 'failed'
  }
  
  scope :publishable, -> {
    where(status: [:pending, :failed])
      .where('attempts < ?', 5)
      .where('last_attempt_at IS NULL OR last_attempt_at < ?', 5.minutes.ago)
      .order(:created_at)
  }
  
  def mark_published!
    update!(
      status: 'published',
      published_at: Time.now
    )
  end
  
  def mark_failed!(error)
    update!(
      status: attempts >= 4 ? 'failed' : 'pending',
      attempts: attempts + 1,
      last_attempt_at: Time.now,
      last_error: error.message
    )
  end
end

# app/services/order_service.rb
class OrderService
  def create_order(params)
    # ทั้งสองเกิดใน single transaction
    ActiveRecord::Base.transaction do
      order = Order.create!(params)
      
      # Store event ใน outbox (same transaction)
      OutboxEvent.create!(
        aggregate_type: 'Order',
        aggregate_id: order.id,
        event_type: 'order.created',
        payload: {
          order_id: order.id,
          user_id: order.user_id,
          total_cents: order.total_cents,
          items: order.items.map { |i| { product_id: i.product_id, quantity: i.quantity } }
        }
      )
      
      order
    end
  end
  
  def cancel_order(order_id, reason: nil)
    ActiveRecord::Base.transaction do
      order = Order.find(order_id)
      order.update!(status: 'cancelled')
      
      OutboxEvent.create!(
        aggregate_type: 'Order',
        aggregate_id: order.id,
        event_type: 'order.cancelled',
        payload: {
          order_id: order.id,
          reason: reason,
          cancelled_at: Time.now.iso8601
        }
      )
      
      order
    end
  end
end

# app/jobs/outbox_publisher_job.rb
class OutboxPublisherJob < ApplicationJob
  queue_as :outbox
  
  # Run ทุก 5 วินาที
  def perform
    OutboxEvent.publishable.limit(100).each do |event|
      publish_event(event)
    end
    
    # Schedule ตัวเองใหม่
    self.class.set(wait: 5.seconds).perform_later
  end
  
  private
  
  def publish_event(event)
    # Optimistic locking: try to claim the event
    rows_updated = OutboxEvent.where(id: event.id, status: ['pending', 'failed'])
                              .update_all(status: 'processing')
    
    return unless rows_updated == 1
    
    begin
      EventPublisher.publish(event.event_type, event.payload)
      event.mark_published!
    rescue => e
      Rails.logger.error "Failed to publish event #{event.id}: #{e.message}"
      event.mark_failed!(e)
    end
  end
end

# app/lib/event_publisher.rb
class EventPublisher
  def self.publish(event_type, payload)
    # Publish to message broker
    encoded_payload = payload.to_json
    
    case Rails.env
    when 'production', 'staging'
      publish_to_kafka(event_type, encoded_payload)
    else
      publish_to_redis(event_type, encoded_payload)
    end
  end
  
  private
  
  def self.publish_to_kafka(topic, payload)
    producer = Karafka.producer
    producer.produce_sync(
      topic: topic,
      payload: payload,
      key: JSON.parse(payload)['order_id']&.to_s
    )
  end
  
  def self.publish_to_redis(channel, payload)
    redis = Redis.new(url: ENV['REDIS_URL'])
    redis.xadd("events:#{channel}", '*', 'data', payload)
  end
end
```

---

## 6. Eventually Consistent Patterns

### 6.1 Read-Your-Writes Consistency

```ruby
# หลัง write, user ควรเห็น write ของตัวเอง
# แม้ใช้ read replicas

# app/models/concerns/consistency_aware.rb
module ConsistencyAware
  extend ActiveSupport::Concern
  
  included do
    after_save :record_write_timestamp
    after_destroy :record_write_timestamp
  end
  
  private
  
  def record_write_timestamp
    # เก็บ timestamp ล่าสุดที่ user นี้ write
    if respond_to?(:user_id) && user_id
      ConsistencyTracker.record_write(user_id: user_id)
    end
  end
end

# app/lib/consistency_tracker.rb
class ConsistencyTracker
  WRITE_TTL = 30  # seconds
  
  def self.record_write(user_id:)
    redis = Redis.new(url: ENV['REDIS_URL'])
    redis.setex("user_wrote:#{user_id}", WRITE_TTL, Time.now.to_f.to_s)
  end
  
  def self.should_use_primary?(user_id)
    redis = Redis.new(url: ENV['REDIS_URL'])
    timestamp = redis.get("user_wrote:#{user_id}")
    
    return false unless timestamp
    
    # ถ้า user เพิ่ง write ภายใน WRITE_TTL วินาที ใช้ primary
    Time.now.to_f - timestamp.to_f < WRITE_TTL
  end
end

# app/controllers/concerns/consistency_routing.rb
module ConsistencyRouting
  extend ActiveSupport::Concern
  
  included do
    around_action :route_based_on_consistency
  end
  
  private
  
  def route_based_on_consistency(&block)
    if current_user && ConsistencyTracker.should_use_primary?(current_user.id)
      # Route ไป primary database
      ApplicationRecord.connected_to(role: :writing, &block)
    else
      # Route ไป read replica
      ApplicationRecord.connected_to(role: :reading, &block)
    end
  end
end

class ArticlesController < ApplicationController
  include ConsistencyRouting
  
  def index
    @articles = Article.published.recent.page(params[:page])
  end
  
  def create
    @article = Article.new(article_params)
    @article.user = current_user
    
    if @article.save
      redirect_to @article, notice: 'Article created!'
    else
      render :new
    end
  end
end
```

### 6.2 Eventual Consistency ด้วย Event Sourcing

```ruby
# app/models/account_projection.rb
class AccountProjection
  attr_reader :id, :balance, :version, :events

  def initialize(id)
    @id = id
    @balance = 0
    @version = 0
    @events = []
  end

  def apply(event)
    case event[:type]
    when 'account.opened'
      @balance = event[:data][:initial_balance]
    when 'amount.deposited'
      @balance += event[:data][:amount]
    when 'amount.withdrawn'
      @balance -= event[:data][:amount]
    when 'account.closed'
      @closed = true
    end

    @version += 1
    self
  end

  def deposit(amount)
    raise "Account closed" if @closed
    raise ArgumentError, "Amount must be positive" unless amount > 0

    event = {
      type: 'amount.deposited',
      aggregate_id: @id,
      data: { amount: amount },
      timestamp: Time.now.iso8601,
      version: @version + 1
    }

    apply(event)
    @events << event
    self
  end

  def withdraw(amount)
    raise "Account closed" if @closed
    raise ArgumentError, "Insufficient funds" if amount > @balance

    event = {
      type: 'amount.withdrawn',
      aggregate_id: @id,
      data: { amount: amount },
      timestamp: Time.now.iso8601,
      version: @version + 1
    }

    apply(event)
    @events << event
    self
  end

  def self.rebuild_from_events(account_id)
    new(account_id).tap do |account|
      AccountEvent.where(aggregate_id: account_id)
                  .order(:version)
                  .each do |stored_event|
        account.apply(stored_event.data.symbolize_keys)
      end
    end
  end
end

# app/services/account_command_handler.rb
class AccountCommandHandler
  def deposit(account_id:, amount:)
    ActiveRecord::Base.transaction do
      account = AccountProjection.rebuild_from_events(account_id)
      account.deposit(amount)

      save_events(account)
      update_read_model(account)
    end
  end

  def withdraw(account_id:, amount:)
    ActiveRecord::Base.transaction do
      account = AccountProjection.rebuild_from_events(account_id)
      account.withdraw(amount)

      save_events(account)
      update_read_model(account)
    end
  end

  private

  def save_events(account)
    account.events.each do |event|
      AccountEvent.create!(
        event_type: event[:type],
        aggregate_id: event[:aggregate_id],
        data: event,
        version: event[:version]
      )
    end
  end

  def update_read_model(account)
    AccountSummary.upsert({
      account_id: account.id,
      balance: account.balance,
      version: account.version,
      updated_at: Time.now
    }, unique_by: :account_id)
  end
end
```

### 6.3 Idempotency Keys

```ruby
# app/controllers/concerns/idempotency.rb
module Idempotency
  extend ActiveSupport::Concern

  included do
    before_action :check_idempotency_key, only: [:create, :update]
  end

  private

  def check_idempotency_key
    key = request.headers['Idempotency-Key']
    return unless key

    cached = IdempotencyCache.get(key)

    if cached
      render json: cached[:response], status: cached[:status]
      return false
    end

    @idempotency_key = key
  end

  def store_idempotency_response(response_data, status_code)
    return unless @idempotency_key

    IdempotencyCache.set(@idempotency_key, {
      response: response_data,
      status: status_code,
      created_at: Time.now.iso8601
    })
  end
end

# app/lib/idempotency_cache.rb
class IdempotencyCache
  TTL = 24.hours.to_i

  def self.set(key, data)
    redis = Redis.new(url: ENV['REDIS_URL'])
    redis.setex("idempotency:#{key}", TTL, data.to_json)
  end

  def self.get(key)
    redis = Redis.new(url: ENV['REDIS_URL'])
    raw = redis.get("idempotency:#{key}")
    return nil unless raw

    JSON.parse(raw, symbolize_names: true)
  end
end

# app/controllers/payments_controller.rb
class PaymentsController < ApplicationController
  include Idempotency

  def create
    result = PaymentService.charge(
      user_id: current_user.id,
      amount: params[:amount],
      currency: params[:currency]
    )

    response_data = {
      payment_id: result.payment_id,
      status: result.status,
      amount: result.amount
    }

    store_idempotency_response(response_data, :created)

    render json: response_data, status: :created
  end
end
```

---

## 7. Distributed Locking

### 7.1 Redis Distributed Lock

```ruby
# app/lib/distributed_lock.rb
class DistributedLock
  class LockError < StandardError; end
  class LockNotOwnedError < LockError; end

  DEFAULT_TTL = 30        # seconds
  DEFAULT_RETRY_COUNT = 3
  DEFAULT_RETRY_DELAY = 0.2  # seconds

  def initialize(key, ttl: DEFAULT_TTL, redis: nil)
    @key = "lock:#{key}"
    @ttl = ttl
    @token = SecureRandom.uuid
    @redis = redis || Redis.new(url: ENV['REDIS_URL'])
  end

  def acquire!
    retry_count = 0

    loop do
      acquired = @redis.set(@key, @token, nx: true, ex: @ttl)

      return self if acquired

      retry_count += 1
      raise LockError, "Failed to acquire lock for #{@key}" if retry_count >= DEFAULT_RETRY_COUNT

      sleep DEFAULT_RETRY_DELAY * retry_count
    end
  end

  def release!
    # Lua script ensures atomic check-and-delete
    # เพื่อป้องกัน race condition
    script = <<~LUA
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    LUA

    result = @redis.eval(script, keys: [@key], argv: [@token])
    raise LockNotOwnedError, "Lock not owned by this process" if result == 0
    true
  end

  def self.with_lock(key, ttl: DEFAULT_TTL, &block)
    lock = new(key, ttl: ttl)
    lock.acquire!

    begin
      yield
    ensure
      lock.release! rescue LockNotOwnedError
    end
  end
end

# การใช้งาน
class InventoryService
  def self.update_stock(product_id, quantity_delta)
    DistributedLock.with_lock("inventory:product:#{product_id}", ttl: 10) do
      product = Product.find(product_id)

      new_quantity = product.stock_quantity + quantity_delta
      raise "Insufficient stock" if new_quantity < 0

      product.update!(stock_quantity: new_quantity)
    end
  rescue DistributedLock::LockError => e
    Rails.logger.warn "Could not acquire inventory lock: #{e.message}"
    raise
  end
end
```

---

## แบบฝึกหัด (20 ข้อพร้อมเฉลย)

### ข้อ 1: CAP Theorem Analysis
**คำถาม:** วิเคราะห์ว่า Rails ที่ใช้ PostgreSQL primary + read replicas อยู่ใน category ใดของ CAP theorem และเพราะอะไร

**เฉลย:**
```
Rails + PostgreSQL (Primary/Replica) อยู่ใน CP ระบบ:

- Consistency (C): ✅ Primary ให้ strong consistency
  read replicas อาจมี replication lag แต่ primary เสมอ consistent

- Availability (A): ❌ บางส่วน เมื่อ primary ล้มเหลว
  readonly operations ยังทำได้ แต่ writes จะ fail จนกว่า failover จะเสร็จ

- Partition Tolerance (P): ✅ ทำงานได้แม้ network partition
  แต่ consistency ลดลงในช่วง partition

สรุป: CP system ที่ tradeoff availability เพื่อ consistency
การออกแบบ: ใช้ connected_to สำหรับ read replicas
แต่ critical writes ต้องไปที่ primary เสมอ
```

### ข้อ 2: Circuit Breaker States
**คำถาม:** สร้าง test สำหรับ CircuitBreaker ที่ verify state transitions ทั้งหมด

**เฉลย:**
```ruby
RSpec.describe CircuitBreaker do
  let(:circuit) do
    CircuitBreaker.new('test',
      failure_threshold: 2,
      success_threshold: 1,
      timeout: 0.1,  # 100ms สำหรับ test
      exceptions: [RuntimeError]
    )
  end

  describe 'CLOSED -> OPEN transition' do
    it 'opens after failure threshold' do
      expect(circuit.closed?).to be true

      2.times do
        expect {
          circuit.call { raise RuntimeError, "service error" }
        }.to raise_error(RuntimeError)
      end

      expect(circuit.open?).to be true
    end
  end

  describe 'OPEN -> HALF_OPEN transition' do
    before do
      2.times { circuit.call { raise RuntimeError } rescue nil }
    end

    it 'transitions to half-open after timeout' do
      expect(circuit.open?).to be true
      sleep 0.15  # wait for timeout

      # next call attempts half-open
      circuit.call { "success" } rescue nil

      expect(circuit.half_open?).to be true
    end
  end

  describe 'HALF_OPEN -> CLOSED on success' do
    before do
      2.times { circuit.call { raise RuntimeError } rescue nil }
      sleep 0.15
    end

    it 'closes on successful call' do
      circuit.call { "success" }
      expect(circuit.closed?).to be true
    end
  end
end
```

### ข้อ 3: OpenTelemetry Tracing
**คำถาม:** เพิ่ม distributed tracing สำหรับ method ที่เรียก 3 external services

**เฉลย:**
```ruby
class DataAggregationService
  TRACER = OpenTelemetry.tracer_provider.tracer('data-aggregation')

  def self.aggregate_user_data(user_id)
    TRACER.in_span('data.aggregate', attributes: { 'user.id' => user_id }) do |span|
      results = {}

      TRACER.in_span('fetch.profile') do
        results[:profile] = UserProfileService.get(user_id)
      end

      TRACER.in_span('fetch.orders') do |s|
        results[:orders] = OrderService.get_recent(user_id)
        s.set_attribute('orders.count', results[:orders].count)
      end

      TRACER.in_span('fetch.preferences') do
        results[:preferences] = PreferenceService.get(user_id)
      end

      span.set_attribute('data.sections', results.keys.join(','))
      results
    end
  end
end
```

### ข้อ 4: Saga Orchestration
**คำถาม:** สร้าง saga สำหรับ user registration ที่ต้องสร้าง account ใน 3 systems

**เฉลย:**
```ruby
class UserRegistrationSaga
  STEPS = [:create_auth_account, :create_profile, :setup_billing, :send_welcome].freeze

  def execute(registration_params)
    results = {}
    compensations = []

    begin
      # Step 1: Auth
      auth_result = AuthService.create_user(registration_params.slice(:email, :password))
      raise "Auth failed: #{auth_result.error}" unless auth_result.success?
      results[:auth_id] = auth_result.user_id
      compensations << -> { AuthService.delete_user(auth_result.user_id) }

      # Step 2: Profile
      profile_result = ProfileService.create(
        auth_id: results[:auth_id],
        name: registration_params[:name]
      )
      raise "Profile failed: #{profile_result.error}" unless profile_result.success?
      results[:profile_id] = profile_result.profile_id
      compensations << -> { ProfileService.delete(profile_result.profile_id) }

      # Step 3: Billing
      billing_result = BillingService.setup_customer(email: registration_params[:email])
      raise "Billing failed: #{billing_result.error}" unless billing_result.success?
      results[:customer_id] = billing_result.customer_id
      compensations << -> { BillingService.delete_customer(billing_result.customer_id) }

      # Step 4: Welcome email (no compensation needed)
      WelcomeMailer.welcome(email: registration_params[:email]).deliver_later

      { success: true, user_data: results }

    rescue => error
      compensations.reverse.each do |comp|
        comp.call rescue nil
      end
      { success: false, error: error.message }
    end
  end
end
```

### ข้อ 5: Outbox Pattern Testing
**คำถาม:** เขียน test สำหรับ OutboxEvent lifecycle ตั้งแต่ create จนถึง publish

**เฉลย:**
```ruby
RSpec.describe OutboxEvent do
  describe 'lifecycle' do
    let(:event) { create(:outbox_event, event_type: 'order.created') }

    it 'starts as pending' do
      expect(event.pending?).to be true
      expect(event.attempts).to eq(0)
    end

    context 'when publishing succeeds' do
      it 'marks as published' do
        event.mark_published!
        expect(event.published?).to be true
        expect(event.published_at).to be_present
      end
    end

    context 'when publishing fails' do
      let(:error) { RuntimeError.new("connection refused") }

      it 'increments attempts and stays pending (< 5)' do
        4.times { event.mark_failed!(error) }
        expect(event.pending?).to be true
        expect(event.attempts).to eq(4)
      end

      it 'marks as failed after 5 attempts' do
        5.times { event.mark_failed!(error) }
        expect(event.failed?).to be true
      end
    end

    describe '.publishable scope' do
      it 'returns pending events with attempts < 5' do
        pending_event = create(:outbox_event, status: 'pending', attempts: 0)
        failed_event = create(:outbox_event, status: 'failed', attempts: 5)
        published_event = create(:outbox_event, status: 'published')

        publishable = OutboxEvent.publishable
        expect(publishable).to include(pending_event)
        expect(publishable).not_to include(failed_event, published_event)
      end
    end
  end
end
```

### ข้อ 6: Idempotency Middleware
**คำถาม:** สร้าง Rack middleware สำหรับ handle Idempotency-Key header

**เฉลย:**
```ruby
class IdempotencyMiddleware
  TTL = 86400  # 24 hours

  def initialize(app)
    @app = app
    @redis = Redis.new(url: ENV['REDIS_URL'])
  end

  def call(env)
    request = Rack::Request.new(env)
    key = request.get_header('HTTP_IDEMPOTENCY_KEY')

    if key && mutating_method?(request.request_method)
      cached = @redis.get("idempotency:#{key}")

      if cached
        data = JSON.parse(cached)
        return [data['status'], data['headers'], [data['body']]]
      end

      status, headers, body = @app.call(env)

      body_str = body.map(&:to_s).join
      @redis.setex("idempotency:#{key}", TTL, {
        status: status,
        headers: headers,
        body: body_str
      }.to_json)

      [status, headers, [body_str]]
    else
      @app.call(env)
    end
  end

  private

  def mutating_method?(method)
    %w[POST PUT PATCH DELETE].include?(method)
  end
end
```

### ข้อ 7: Distributed Lock ด้วย Lua Script
**คำถาม:** อธิบายว่าทำไม Lua script ถึงจำเป็นสำหรับ release distributed lock

**เฉลย:**
```ruby
# ปัญหาโดยไม่ใช้ Lua script (race condition):
# 1. Process A check: key exists, value = A's token ✅
# 2. Context switch - Process A's lock expires
# 3. Process B acquires lock, sets token = B's token
# 4. Process A resumes, deletes key (ลบ lock ของ B โดยผิดพลาด!)

# Lua script แก้ปัญหา:
lua_release = <<~LUA
  if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
  else
    return 0
  end
LUA

# Lua script execute แบบ atomic ใน Redis
# check และ delete เกิดในขั้นตอนเดียว ไม่มี race condition

# Test:
RSpec.describe DistributedLock do
  it 'prevents another process from releasing the lock' do
    lock1 = DistributedLock.new('resource:1', ttl: 30)
    lock2 = DistributedLock.new('resource:1', ttl: 30)

    lock1.acquire!
    lock2_acquired = lock2.acquire! rescue false
    expect(lock2_acquired).to be false

    expect { lock2.release! }.to raise_error(DistributedLock::LockNotOwnedError)

    # lock1 ยังคง valid
    lock1.release!
  end
end
```

### ข้อ 8: Event Sourcing Projection
**คำถาม:** สร้าง projection สำหรับ shopping cart ที่ rebuild ได้จาก events

**เฉลย:**
```ruby
class CartProjection
  attr_reader :cart_id, :items, :version

  def initialize(cart_id)
    @cart_id = cart_id
    @items = {}
    @version = 0
  end

  def apply(event)
    case event[:type]
    when 'cart.item_added'
      product_id = event[:data][:product_id]
      @items[product_id] = (@items[product_id] || 0) + event[:data][:quantity]
    when 'cart.item_removed'
      product_id = event[:data][:product_id]
      @items.delete(product_id)
    when 'cart.item_quantity_updated'
      product_id = event[:data][:product_id]
      @items[product_id] = event[:data][:quantity]
    when 'cart.cleared'
      @items = {}
    when 'cart.checked_out'
      @checked_out = true
    end
    @version += 1
    self
  end

  def total_items
    @items.values.sum
  end

  def checked_out?
    @checked_out == true
  end

  def self.rebuild(cart_id)
    new(cart_id).tap do |cart|
      CartEvent.where(cart_id: cart_id).order(:version).each do |e|
        cart.apply(e.data.symbolize_keys)
      end
    end
  end
end
```

### ข้อ 9: Compensation Transaction
**คำถาม:** สร้าง compensation transaction system ที่ retry ด้วย exponential backoff

**เฉลย:**
```ruby
class CompensationExecutor
  MAX_RETRIES = 5
  BASE_DELAY = 1.0

  def self.execute_with_retry(name:, &block)
    retries = 0

    begin
      block.call
      Rails.logger.info "Compensation #{name} succeeded"
    rescue => e
      retries += 1
      if retries <= MAX_RETRIES
        delay = BASE_DELAY * (2 ** (retries - 1)) + rand(0.0..0.5)
        Rails.logger.warn "Compensation #{name} failed (attempt #{retries}), retrying in #{delay}s: #{e.message}"
        sleep delay
        retry
      else
        Rails.logger.error "Compensation #{name} permanently failed after #{MAX_RETRIES} attempts: #{e.message}"
        CompensationFailure.create!(
          name: name,
          error: e.message,
          attempts: retries
        )
        raise
      end
    end
  end
end
```

### ข้อ 10: Consistent Hashing
**คำถาม:** implement consistent hashing สำหรับ distribute keys ไปยัง multiple shards

**เฉลย:**
```ruby
class ConsistentHashRing
  VIRTUAL_NODES = 150

  def initialize(nodes = [])
    @ring = {}
    @sorted_keys = []

    nodes.each { |node| add_node(node) }
  end

  def add_node(node)
    VIRTUAL_NODES.times do |i|
      key = hash_key("#{node}:#{i}")
      @ring[key] = node
      @sorted_keys << key
    end
    @sorted_keys.sort!
  end

  def remove_node(node)
    VIRTUAL_NODES.times do |i|
      key = hash_key("#{node}:#{i}")
      @ring.delete(key)
      @sorted_keys.delete(key)
    end
  end

  def get_node(key)
    return nil if @ring.empty?

    hash = hash_key(key)
    target = @sorted_keys.bsearch { |k| k >= hash }
    target ||= @sorted_keys.first

    @ring[target]
  end

  private

  def hash_key(key)
    Digest::MD5.hexdigest(key).to_i(16)
  end
end

ring = ConsistentHashRing.new(['db1', 'db2', 'db3'])
puts ring.get_node('user:1234')  # => "db2" (consistent)
puts ring.get_node('user:5678')  # => "db1"
```

### ข้อ 11: Health Check Endpoint

```ruby
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  skip_before_action :authenticate_user!

  def liveness
    render json: { status: 'ok', timestamp: Time.now.iso8601 }
  end

  def readiness
    checks = {
      database: check_database,
      redis: check_redis,
      sidekiq: check_sidekiq
    }

    all_healthy = checks.values.all? { |c| c[:status] == 'ok' }
    status_code = all_healthy ? :ok : :service_unavailable

    render json: {
      status: all_healthy ? 'ready' : 'not_ready',
      checks: checks,
      timestamp: Time.now.iso8601
    }, status: status_code
  end

  private

  def check_database
    ActiveRecord::Base.connection.execute('SELECT 1')
    { status: 'ok' }
  rescue => e
    { status: 'error', message: e.message }
  end

  def check_redis
    Redis.new(url: ENV['REDIS_URL']).ping == 'PONG' ?
      { status: 'ok' } :
      { status: 'error', message: 'Redis ping failed' }
  rescue => e
    { status: 'error', message: e.message }
  end

  def check_sidekiq
    stats = Sidekiq::Stats.new
    { status: 'ok', queued: stats.enqueued, failed: stats.failed }
  rescue => e
    { status: 'error', message: e.message }
  end
end
```

### ข้อ 12: Message Deduplication

```ruby
class MessageDeduplicator
  TTL = 24.hours.to_i

  def self.process(message_id, &block)
    redis = Redis.new(url: ENV['REDIS_URL'])
    key = "msg_processed:#{message_id}"

    # SETNX: set if not exists (atomic)
    was_set = redis.setnx(key, Time.now.iso8601)

    if was_set
      redis.expire(key, TTL)
      block.call
      true
    else
      Rails.logger.info "Duplicate message #{message_id} skipped"
      false
    end
  end
end

# Usage in event handler
class OrderCreatedHandler
  def self.call(event)
    MessageDeduplicator.process(event['id']) do
      order_id = event['payload']['order_id']
      InventoryService.reserve(order_id: order_id)
    end
  end
end
```

### ข้อ 13: Retry ด้วย Exponential Backoff

```ruby
module Retryable
  def with_retry(max_attempts: 3, base_delay: 1.0, exceptions: [StandardError])
    attempts = 0

    begin
      attempts += 1
      yield
    rescue *exceptions => e
      if attempts < max_attempts
        delay = base_delay * (2 ** (attempts - 1))
        jitter = rand(0.0..(delay * 0.1))
        Rails.logger.warn "Attempt #{attempts} failed: #{e.message}. Retrying in #{delay + jitter}s"
        sleep(delay + jitter)
        retry
      else
        raise
      end
    end
  end
end

class ExternalServiceClient
  include Retryable

  def fetch_data(id)
    with_retry(max_attempts: 3, base_delay: 0.5, exceptions: [Net::OpenTimeout, Faraday::Error]) do
      connection.get("/data/#{id}").body
    end
  end
end
```

### ข้อ 14-20 (สรุปเฉลย):

**ข้อ 14: Two-Phase Commit Alternative**
```ruby
# ใช้ Saga + Outbox แทน 2PC
# เพราะ 2PC มี single point of failure (coordinator)
# Saga + Outbox ให้ eventual consistency ที่ reliable กว่า

class TransferService
  def transfer(from_account_id, to_account_id, amount)
    ActiveRecord::Base.transaction do
      from = Account.lock.find(from_account_id)
      raise "Insufficient funds" if from.balance < amount
      from.decrement!(:balance, amount)

      OutboxEvent.create!(
        aggregate_type: 'Transfer',
        aggregate_id: from_account_id,
        event_type: 'transfer.initiated',
        payload: { from: from_account_id, to: to_account_id, amount: amount }
      )
    end
  end
end
```

**ข้อ 15: Service Mesh Pattern**
```ruby
class ServiceMeshClient
  def initialize(service_name)
    @service_name = service_name
    @base_url = discover_service(service_name)
    @circuit = CircuitBreaker.new(service_name, failure_threshold: 5)
  end

  def call(method, path, body: nil)
    @circuit.call do
      conn = Faraday.new(@base_url) do |f|
        f.request :json
        f.response :json
      end
      conn.send(method, path, body)
    end
  end

  private

  def discover_service(name)
    consul_url = ENV['CONSUL_URL']
    response = Faraday.get("#{consul_url}/v1/catalog/service/#{name}")
    services = JSON.parse(response.body)
    "http://#{services.first['ServiceAddress']}:#{services.first['ServicePort']}"
  end
end
```

**ข้อ 16: Bulkhead Pattern**
```ruby
# แยก thread pool สำหรับแต่ละ service เพื่อป้องกัน cascade
class BulkheadExecutor
  def initialize(pool_size: 10)
    @executor = Concurrent::ThreadPoolExecutor.new(
      min_threads: 2,
      max_threads: pool_size,
      max_queue: 50,
      fallback_policy: :caller_runs
    )
  end

  def execute(&block)
    future = Concurrent::Future.execute(executor: @executor, &block)
    future.value(5)  # 5 second timeout
  rescue Concurrent::TimeoutError
    raise ServiceTimeoutError
  end
end

PAYMENT_BULKHEAD = BulkheadExecutor.new(pool_size: 5)
INVENTORY_BULKHEAD = BulkheadExecutor.new(pool_size: 10)
```

**ข้อ 17: Dead Letter Queue**
```ruby
class DeadLetterQueue
  QUEUE_NAME = 'dead_letter_queue'

  def self.push(message, error:, original_queue:)
    Redis.new.lpush(QUEUE_NAME, {
      message: message,
      error: error.message,
      original_queue: original_queue,
      failed_at: Time.now.iso8601,
      retry_count: (message[:retry_count] || 0) + 1
    }.to_json)
  end

  def self.process_all
    redis = Redis.new
    while (item = redis.rpop(QUEUE_NAME))
      data = JSON.parse(item, symbolize_names: true)
      yield data
    end
  end
end
```

**ข้อ 18: Rate Limiter with Sliding Window**
```ruby
class SlidingWindowRateLimiter
  def initialize(limit:, window:)
    @limit = limit
    @window = window
    @redis = Redis.new(url: ENV['REDIS_URL'])
  end

  def allow?(key)
    now = Time.now.to_f
    window_start = now - @window
    redis_key = "rate_limit:#{key}"

    @redis.multi do |r|
      r.zremrangebyscore(redis_key, '-inf', window_start)
      r.zadd(redis_key, now, "#{now}:#{SecureRandom.uuid}")
      r.zcard(redis_key)
      r.expire(redis_key, @window.ceil)
    end.then { |results| results[2] <= @limit }
  end
end
```

**ข้อ 19: Chaos Engineering**
```ruby
# Inject faults สำหรับ testing resilience
module ChaosMonkey
  def self.maybe_fail!(probability: 0.01, exception: RuntimeError)
    raise exception, "Chaos monkey strikes!" if rand < probability
  end

  def self.maybe_delay!(max_ms: 1000, probability: 0.05)
    if rand < probability
      delay = rand(0..(max_ms / 1000.0))
      sleep delay
    end
  end
end

class ExternalService
  def call(params)
    if Rails.env.test? && ENV['CHAOS_ENABLED']
      ChaosMonkey.maybe_fail!(probability: 0.1)
      ChaosMonkey.maybe_delay!(max_ms: 500, probability: 0.2)
    end
    # actual implementation
  end
end
```

**ข้อ 20: Complete Distributed Transaction Example**
```ruby
class PurchaseOrchestrator
  include ActiveSupport::Callbacks
  define_callbacks :purchase

  def complete_purchase(cart_id:, user_id:, payment_token:)
    saga = CreateOrderSaga.new(
      cart_id: cart_id,
      user_id: user_id,
      payment_token: payment_token
    )

    result = saga.execute

    if result.success?
      PurchaseCompletedEvent.publish(
        order_id: result.order_id,
        user_id: user_id
      )
      { success: true, order_id: result.order_id }
    else
      { success: false, error: result.error, retryable: result.retryable? }
    end
  rescue CircuitBreaker::OpenCircuitError => e
    { success: false, error: 'service_unavailable', retry_after: 30 }
  rescue DistributedLock::LockError => e
    { success: false, error: 'concurrent_request', message: 'Please try again' }
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ distributed systems patterns สำคัญ:

1. **CAP Theorem** - ความเข้าใจ trade-offs ระหว่าง Consistency, Availability, Partition Tolerance
2. **Circuit Breaker** - ป้องกัน cascading failures ด้วย Faraday + Circuitbox
3. **OpenTelemetry** - Distributed tracing ข้าม services
4. **Saga Pattern** - จัดการ distributed transactions ด้วย Orchestration และ Choreography
5. **Outbox Pattern** - Reliable message publishing ด้วย database transaction
6. **Eventually Consistent** - Read-your-writes, Event Sourcing, Idempotency

Key principles:
- Design for failure (assume things will break)
- Use idempotency keys สำหรับ safe retries
- Implement circuit breakers เพื่อ fail fast
- Use outbox pattern สำหรับ reliable messaging
- Trace everything ด้วย OpenTelemetry
