# Part 85: Scaling Rails Applications

## บทนำ

เมื่อแอปพลิเคชันเติบโต ความสามารถในการรองรับ load ที่สูงขึ้นเป็นสิ่งจำเป็น ในบทนี้เราจะเรียนรู้กลยุทธ์การ scale Rails application ในหลายระดับ

---

## 1. Horizontal Scaling Strategies

### 1.1 Stateless Application Design

Rails application ต้องเป็น stateless เพื่อรองรับ horizontal scaling

```ruby
# config/environments/production.rb
Rails.application.configure do
  # Session store ใน Redis (ไม่ใช่ cookie/memory)
  config.session_store :redis_session_store, {
    key: '_myapp_session',
    redis: {
      expire_after: 120.minutes,
      key_prefix: 'myapp:session:',
      url: ENV['REDIS_URL']
    },
    secure: true,
    httponly: true
  }
  
  # Cache ใน Redis
  config.cache_store = :redis_cache_store, {
    url: ENV['REDIS_URL'],
    pool_size: ENV.fetch('RAILS_MAX_THREADS') { 5 }.to_i,
    pool_timeout: 5,
    connect_timeout: 0.5,
    error_handler: ->(method:, returning:, exception:) {
      Sentry.capture_exception(exception, level: 'warning',
        tags: { method: method, returning: returning })
    }
  }
  
  # Job backend ที่ shared ระหว่าง instances
  config.active_job.queue_adapter = :sidekiq
  
  # Cable ใน Redis
  # config/cable.yml กำหนดให้ใช้ Redis adapter
end
```

```ruby
# config/cable.yml
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: myapp_production
```

### 1.2 Load Balancing Configuration

```nginx
# nginx.conf - Upstream configuration
upstream rails_app {
  # least_conn หรือ ip_hash สำหรับ sticky sessions
  least_conn;
  
  server web1.internal:3000 weight=5 max_fails=3 fail_timeout=30s;
  server web2.internal:3000 weight=5 max_fails=3 fail_timeout=30s;
  server web3.internal:3000 weight=3 max_fails=3 fail_timeout=30s;
  
  keepalive 64;  # Keep connections alive
}

server {
  listen 443 ssl http2;
  server_name app.example.com;
  
  location / {
    proxy_pass http://rails_app;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    
    # Timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
  }
}
```

### 1.3 Puma Configuration สำหรับ Production

```ruby
# config/puma.rb
max_threads_count = ENV.fetch('RAILS_MAX_THREADS') { 5 }
min_threads_count = ENV.fetch('RAILS_MIN_THREADS') { max_threads_count }

threads min_threads_count, max_threads_count

port ENV.fetch('PORT') { 3000 }
environment ENV.fetch('RAILS_ENV') { 'development' }

workers ENV.fetch('WEB_CONCURRENCY') { 2 }

# Phased restart สำหรับ zero-downtime deployments
phased_restart

on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
  Rails.cache.reconnect if Rails.cache.respond_to?(:reconnect)
end

on_worker_fork do
  # Reset connections ใน forked process
  ActiveRecord::Base.connection_pool.disconnect!
end

before_fork do
  ActiveRecord::Base.connection_pool.disconnect!
end

# Preload app สำหรับ faster boot time (ใช้กับ Copy-on-Write)
preload_app!

lowlevel_error_handler do |e|
  [500, {}, ["An error occurred: #{e.message}"]]
end

# Plugin สำหรับ metrics
plugin :tmp_restart
```

### 1.4 Rack Middleware Optimization

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # ลด middleware ที่ไม่จำเป็น
    config.middleware.delete ActionDispatch::Session::CookieStore
    config.middleware.delete ActionDispatch::Flash
    config.middleware.delete ActionDispatch::Cookies
    
    # สำหรับ API-only app
    # config.api_only = true
    
    # Enable Gzip compression
    config.middleware.use Rack::Deflater
    
    # Request timeout
    config.middleware.insert_before Rack::Runtime, Rack::Timeout, service_timeout: 30
    
    # Rate limiting
    config.middleware.use Rack::Attack
  end
end
```

---

## 2. Database Read Replicas ใน Rails

### 2.1 Configure Multiple Databases

```yaml
# config/database.yml
production:
  primary:
    adapter: postgresql
    database: myapp_production
    host: primary.db.example.com
    pool: <%= ENV.fetch("DB_POOL") { 10 } %>
    
  primary_replica:
    adapter: postgresql
    database: myapp_production
    host: replica1.db.example.com
    pool: <%= ENV.fetch("DB_POOL") { 10 } %>
    replica: true
    
  primary_replica_2:
    adapter: postgresql
    database: myapp_production
    host: replica2.db.example.com
    pool: 5
    replica: true
```

### 2.2 Application-level Read/Write Split

```ruby
# config/initializers/database_connections.rb
# Rails 6+ built-in multi-database support

# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  
  connects_to database: {
    writing: :primary,
    reading: :primary_replica
  }
end

# ใช้ connected_to สำหรับ explicit control
User.connected_to(role: :reading) do
  User.where(active: true).count
end

User.connected_to(role: :writing) do
  User.create!(name: 'New User')
end
```

### 2.3 Automatic Read Replica Routing

```ruby
# config/application.rb
config.active_record.database_selector = { delay: 2.seconds }
config.active_record.database_resolver = 
  ActiveRecord::Middleware::DatabaseSelector::Resolver

config.active_record.database_resolver_context = 
  ActiveRecord::Middleware::DatabaseSelector::Resolver::Session
```

```ruby
# lib/middleware/replica_selector.rb
class ReplicaSelector
  READ_ONLY_PATHS = %r{
    \A/api/v\d+/(products|categories|users/\d+)\z
  }x.freeze
  
  def initialize(app)
    @app = app
  end
  
  def call(env)
    request = Rack::Request.new(env)
    
    if should_use_replica?(request)
      ActiveRecord::Base.connected_to(role: :reading) do
        @app.call(env)
      end
    else
      @app.call(env)
    end
  end
  
  private
  
  def should_use_replica?(request)
    request.get? && 
    READ_ONLY_PATHS.match?(request.path) &&
    !recently_wrote?(request)
  end
  
  def recently_wrote?(request)
    session = request.env['rack.session']
    last_write = session&.dig('last_db_write')
    
    return false unless last_write
    
    Time.at(last_write) > 2.seconds.ago
  end
end
```

### 2.4 Query Routing Service

```ruby
# app/services/database_router.rb
class DatabaseRouter
  READ_METHODS = %i[find find_by where select pluck count sum all exists?].freeze
  
  # Wrap service methods ให้ใช้ replica อัตโนมัติ
  def self.for_reads(&block)
    ActiveRecord::Base.connected_to(role: :reading, &block)
  end
  
  def self.for_writes(&block)
    ActiveRecord::Base.connected_to(role: :writing, &block)
  end
  
  # Concern สำหรับ automatic routing
  module AutoRouting
    extend ActiveSupport::Concern
    
    included do
      around_action :route_database_connections
    end
    
    private
    
    def route_database_connections
      if request.get? || request.head?
        ActiveRecord::Base.connected_to(role: :reading) { yield }
      else
        ActiveRecord::Base.connected_to(role: :writing) { yield }
      end
    end
  end
end

# app/controllers/api/v1/products_controller.rb
module Api
  module V1
    class ProductsController < ApplicationController
      include DatabaseRouter::AutoRouting
      
      def index
        # Automatically ใช้ replica สำหรับ GET requests
        products = Product.active.includes(:category).page(params[:page])
        render json: products
      end
      
      def create
        # Automatically ใช้ primary สำหรับ POST requests
        product = Product.new(product_params)
        if product.save
          render json: product, status: :created
        else
          render json: { errors: product.errors }, status: :unprocessable_entity
        end
      end
    end
  end
end
```

---

## 3. Database Sharding Basics

### 3.1 Horizontal Partitioning

```ruby
# config/database.yml - Sharded setup
production:
  primary:
    adapter: postgresql
    database: myapp_primary
    host: db-primary.example.com
    
  shard_1:
    adapter: postgresql
    database: myapp_shard_1
    host: db-shard-1.example.com
    
  shard_2:
    adapter: postgresql
    database: myapp_shard_2
    host: db-shard-2.example.com
```

```ruby
# app/models/concerns/shardable.rb
module Shardable
  extend ActiveSupport::Concern
  
  SHARD_COUNT = 4
  
  included do
    def self.shard_for(key)
      shard_index = shard_key(key) % SHARD_COUNT + 1
      :"shard_#{shard_index}"
    end
    
    def self.shard_key(key)
      Digest::MD5.hexdigest(key.to_s).to_i(16)
    end
    
    def self.with_shard(key, &block)
      shard = shard_for(key)
      ActiveRecord::Base.connected_to(shard: shard, &block)
    end
    
    def self.across_all_shards(&block)
      results = []
      (1..SHARD_COUNT).each do |i|
        ActiveRecord::Base.connected_to(shard: :"shard_#{i}") do
          results.concat(block.call)
        end
      end
      results
    end
  end
end

# app/models/user.rb
class User < ApplicationRecord
  include Shardable
  
  connects_to shards: {
    shard_1: { writing: :shard_1 },
    shard_2: { writing: :shard_2 }
  }
end

# การใช้งาน
User.with_shard(organization_id) do
  User.where(organization_id: organization_id).all
end
```

### 3.2 Sharding by Tenant

```ruby
# app/models/tenant.rb
class Tenant < ApplicationRecord
  SHARD_MAP = {
    'asia' => :shard_1,
    'europe' => :shard_2,
    'americas' => :shard_3
  }.freeze
  
  def shard
    SHARD_MAP[region] || :shard_1
  end
  
  def with_shard(&block)
    ActiveRecord::Base.connected_to(shard: shard, &block)
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_tenant_shard
  
  private
  
  def set_tenant_shard
    return unless current_tenant
    
    # สำคัญ: set shard สำหรับทุก request
    @shard = current_tenant.shard
    
    # ใช้ around_action หรือ set explicitly
    ActiveRecord::Base.connected_to(shard: @shard) do
      # นี่จะไม่ work แบบนี้ - ต้องใช้ around_action
    end
  end
  
  def current_tenant
    @current_tenant ||= Tenant.find_by(subdomain: request.subdomain)
  end
end
```

---

## 4. Message Queues: Sidekiq Pro, RabbitMQ, Kafka

### 4.1 Sidekiq Pro Configuration

```ruby
# Gemfile
gem 'sidekiq-pro', source: 'https://gems.contribsys.com/'
gem 'sidekiq-ent'  # Enterprise features

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = {
    url: ENV['REDIS_URL'],
    network_timeout: 5,
    pool_timeout: 5
  }
  
  # Sidekiq Pro: Batches
  # Sidekiq Pro: Unique jobs
  # Sidekiq Pro: Rate limiting
  
  config.on(:startup) do
    # Custom startup logic
  end
  
  config.death_handlers << ->(job, ex) do
    # Handle dead jobs
    Sentry.capture_exception(ex, extra: { job: job })
    DeadJobMailer.notify(job: job, error: ex.message).deliver_now
  end
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV['REDIS_URL'] }
end

# config/sidekiq.yml
:concurrency: 10
:queues:
  - [critical, 3]
  - [default, 2]
  - [mailers, 2]
  - [low, 1]

:max_retries: 5
:logfile: log/sidekiq.log
```

### 4.2 Sidekiq Pro Batches

```ruby
# app/jobs/bulk_email_job.rb
class BulkEmailJob < ApplicationJob
  queue_as :default
  
  def perform(campaign_id)
    campaign = Campaign.find(campaign_id)
    users = User.active.where(subscribed: true)
    
    batch = Sidekiq::Batch.new
    batch.description = "Email campaign #{campaign.name}"
    batch.on(:complete, BulkEmailCompletionJob, 'campaign_id' => campaign_id)
    batch.on(:success, BulkEmailSuccessJob, 'campaign_id' => campaign_id)
    
    batch.jobs do
      users.find_in_batches(batch_size: 100) do |user_batch|
        user_batch.each do |user|
          SendCampaignEmailJob.perform_async(user.id, campaign_id)
        end
      end
    end
    
    campaign.update!(batch_id: batch.bid, status: 'sending')
  end
end

class BulkEmailCompletionJob
  include Sidekiq::Worker
  
  def on_complete(status, options)
    campaign = Campaign.find(options['campaign_id'])
    campaign.update!(status: 'sent', completed_at: Time.current)
    
    report = {
      total: status.total,
      succeeded: status.successes,
      failed: status.failures
    }
    
    CampaignReportMailer.completed(campaign, report).deliver_now
  end
end
```

### 4.3 RabbitMQ Integration

```ruby
# Gemfile
gem 'bunny'
gem 'sneakers'  # สำหรับ Rails-style workers

# config/initializers/rabbitmq.rb
RABBITMQ_CONNECTION = Bunny.new(
  host: ENV['RABBITMQ_HOST'] || 'localhost',
  port: ENV['RABBITMQ_PORT']&.to_i || 5672,
  vhost: ENV['RABBITMQ_VHOST'] || '/',
  user: ENV['RABBITMQ_USER'] || 'guest',
  password: ENV['RABBITMQ_PASSWORD'] || 'guest',
  recovery_attempts: 5,
  network_recovery_interval: 10,
  automatically_recover: true
)

RABBITMQ_CONNECTION.start

# app/services/message_publisher.rb
class MessagePublisher
  def self.publish(exchange_name, routing_key, message, options = {})
    channel = RABBITMQ_CONNECTION.create_channel
    exchange = channel.topic(exchange_name, durable: true)
    
    exchange.publish(
      message.to_json,
      routing_key: routing_key,
      persistent: true,
      content_type: 'application/json',
      headers: {
        'x-message-id' => SecureRandom.uuid,
        'x-timestamp' => Time.current.to_i,
        'x-app-name' => 'myapp'
      }.merge(options[:headers] || {})
    )
    
    Rails.logger.info("Published to #{exchange_name}/#{routing_key}: #{message.to_json[0..100]}")
  rescue Bunny::Exception => e
    Rails.logger.error("RabbitMQ publish failed: #{e.message}")
    raise
  ensure
    channel&.close
  end
end

# app/workers/order_event_worker.rb
class OrderEventWorker
  include Sneakers::Worker
  
  from_queue 'order.events',
             exchange: 'order_exchange',
             exchange_type: :topic,
             routing_key: 'order.#',
             durable: true,
             ack: true
  
  def work(message)
    payload = JSON.parse(message)
    event_type = payload['event_type']
    
    case event_type
    when 'order.created'
      handle_order_created(payload)
    when 'order.paid'
      handle_order_paid(payload)
    when 'order.shipped'
      handle_order_shipped(payload)
    when 'order.cancelled'
      handle_order_cancelled(payload)
    else
      Rails.logger.warn("Unknown event type: #{event_type}")
    end
    
    ack!
  rescue => e
    Rails.logger.error("Error processing order event: #{e.message}")
    reject!  # หรือ requeue!
  end
  
  private
  
  def handle_order_created(payload)
    order = Order.find(payload['order_id'])
    OrderNotificationService.new(order).notify_created
    InventoryService.reserve_items(order)
  end
  
  def handle_order_paid(payload)
    order = Order.find(payload['order_id'])
    OrderFulfillmentJob.perform_later(order.id)
  end
end
```

### 4.4 Kafka Integration

```ruby
# Gemfile
gem 'ruby-kafka'
gem 'karafka'  # Kafka framework สำหรับ Rails

# config/karafka.rb
class KarafkaApp < Karafka::App
  setup do |config|
    config.kafka = {
      'bootstrap.servers': ENV['KAFKA_BROKERS'] || 'localhost:9092',
      'security.protocol': 'ssl',
      'ssl.ca.location': '/etc/ssl/certs/ca-certificates.crt'
    }
    
    config.client_id = 'myapp'
    config.consumer_persistence = true
    
    config.logger = Logger.new(Rails.root.join('log', 'karafka.log'))
  end
  
  routes.draw do
    topic 'user.events' do
      consumer UserEventsConsumer
      deserializer Karafka::Deserializers::Json.new
    end
    
    topic 'order.events' do
      consumer OrderEventsConsumer
      batch_consuming true
      batch_size 100
    end
    
    topic 'inventory.updates' do
      consumer InventoryConsumer
    end
  end
end

# app/consumers/order_events_consumer.rb
class OrderEventsConsumer < ApplicationConsumer
  def consume
    messages.each do |message|
      payload = message.payload
      
      case payload['event_type']
      when 'order.created'
        process_order_created(payload)
      when 'order.updated'
        process_order_updated(payload)
      end
      
      mark_as_consumed!(message)
    end
  end
  
  private
  
  def process_order_created(payload)
    Order.transaction do
      order = Order.find(payload['order_id'])
      order.update!(processed_at: Time.current)
      OrderAnalytics.track_creation(order)
    end
  rescue ActiveRecord::RecordNotFound => e
    Rails.logger.error("Order not found: #{payload['order_id']}")
  end
end

# app/producers/order_producer.rb
class OrderProducer
  TOPIC = 'order.events'.freeze
  
  def self.publish_created(order)
    publish('order.created', {
      order_id: order.id,
      user_id: order.user_id,
      total: order.total,
      items_count: order.order_items.count,
      created_at: order.created_at.iso8601
    })
  end
  
  def self.publish_paid(order)
    publish('order.paid', {
      order_id: order.id,
      payment_id: order.payment.id,
      amount: order.total,
      paid_at: Time.current.iso8601
    })
  end
  
  private
  
  def self.publish(event_type, data)
    WaterDrop::Producer.call(
      topic: TOPIC,
      payload: { event_type: event_type, data: data }.to_json,
      key: data[:order_id].to_s
    )
  end
end
```

---

## 5. CQRS Patterns ใน Rails

### 5.1 Command Query Responsibility Segregation

```ruby
# lib/cqrs/
# ├── commands/
# │   ├── base_command.rb
# │   ├── create_order_command.rb
# │   └── update_user_command.rb
# ├── queries/
# │   ├── base_query.rb
# │   ├── get_orders_query.rb
# │   └── search_products_query.rb
# └── handlers/
#     ├── command_handler.rb
#     └── query_handler.rb

# lib/cqrs/commands/base_command.rb
class BaseCommand
  include ActiveModel::Model
  include ActiveModel::Attributes
  
  def valid?
    super.tap do |result|
      raise CommandValidationError, errors.full_messages unless result
    end
  end
end

# lib/cqrs/commands/create_order_command.rb
class CreateOrderCommand < BaseCommand
  attribute :user_id, :integer
  attribute :items, array: true, default: []
  attribute :shipping_address, :string
  attribute :payment_method, :string
  
  validates :user_id, presence: true
  validates :items, presence: true
  validates :shipping_address, presence: true
  validates :payment_method, inclusion: { in: %w[credit_card bank_transfer] }
  
  def items=(value)
    super(value.map { |i| OrderItemData.new(i) })
  end
end

# lib/cqrs/handlers/create_order_handler.rb
class CreateOrderHandler
  def call(command)
    command.valid?
    
    user = User.find(command.user_id)
    
    ActiveRecord::Base.transaction do
      order = Order.create!(
        user: user,
        shipping_address: command.shipping_address,
        payment_method: command.payment_method,
        status: 'pending'
      )
      
      command.items.each do |item|
        product = Product.lock.find(item.product_id)
        
        raise InsufficientStockError, product.name if product.stock < item.quantity
        
        order.order_items.create!(
          product: product,
          quantity: item.quantity,
          unit_price: product.price
        )
        
        product.decrement!(:stock, item.quantity)
      end
      
      order.calculate_total!
      
      # Publish event
      OrderProducer.publish_created(order)
      
      order
    end
  end
end

# lib/cqrs/queries/get_orders_query.rb
class GetOrdersQuery
  def initialize(user_id:, status: nil, page: 1, per_page: 25, sort: :created_at)
    @user_id = user_id
    @status = status
    @page = page
    @per_page = per_page
    @sort = sort
  end
  
  def call
    query = Order.includes(:order_items, :payment)
                 .where(user_id: @user_id)
    
    query = query.where(status: @status) if @status.present?
    
    query.order(@sort => :desc).page(@page).per(@per_page)
  end
end

# app/controllers/api/v1/orders_controller.rb
module Api
  module V1
    class OrdersController < ApplicationController
      def index
        # Query side
        query = GetOrdersQuery.new(
          user_id: current_user.id,
          status: params[:status],
          page: params[:page],
          per_page: params[:per_page]
        )
        
        orders = query.call
        render json: OrderSerializer.new(orders).serializable_hash
      end
      
      def create
        # Command side
        command = CreateOrderCommand.new(order_params)
        order = CreateOrderHandler.new.call(command)
        
        render json: OrderSerializer.new(order).serializable_hash,
               status: :created
      rescue CommandValidationError => e
        render json: { errors: e.messages }, status: :unprocessable_entity
      rescue InsufficientStockError => e
        render json: { error: "Insufficient stock for #{e.message}" }, 
               status: :unprocessable_entity
      end
      
      private
      
      def order_params
        params.require(:order).permit(
          :shipping_address, :payment_method,
          items: [:product_id, :quantity]
        )
      end
    end
  end
end
```

### 5.2 Read Model / Projection

```ruby
# app/models/order_summary.rb
class OrderSummary < ApplicationRecord
  # Denormalized read model สำหรับ query ที่เร็ว
  # ไม่ต้อง join หลาย tables
  
  belongs_to :user
  
  def self.rebuild_for(user)
    summary_data = user.orders
      .includes(:order_items, :payment)
      .joins(:order_items)
      .group('orders.id')
      .select(
        'orders.id',
        'orders.status',
        'orders.created_at',
        'COUNT(order_items.id) as items_count',
        'SUM(order_items.quantity * order_items.unit_price) as total'
      )
      .map do |o|
        {
          user_id: user.id,
          order_id: o.id,
          status: o.status,
          items_count: o.items_count,
          total: o.total,
          created_at: o.created_at
        }
      end
    
    transaction do
      where(user_id: user.id).delete_all
      insert_all(summary_data)
    end
  end
end

# Background job สำหรับ update projections
class UpdateOrderProjectionJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    
    OrderSummary.upsert(
      {
        user_id: order.user_id,
        order_id: order.id,
        status: order.status,
        items_count: order.order_items.count,
        total: order.total,
        updated_at: Time.current
      },
      unique_by: :order_id
    )
  end
end
```

### 5.3 Event Sourcing พื้นฐาน

```ruby
# app/models/domain_event.rb
class DomainEvent < ApplicationRecord
  belongs_to :aggregate, polymorphic: true
  
  validates :event_type, presence: true
  validates :event_data, presence: true
  validates :occurred_at, presence: true
  
  before_create :set_occurred_at
  
  scope :for_aggregate, ->(agg) { where(aggregate: agg).order(:occurred_at) }
  scope :since, ->(time) { where('occurred_at > ?', time) }
  
  def data
    @data ||= JSON.parse(event_data, symbolize_names: true)
  end
  
  private
  
  def set_occurred_at
    self.occurred_at ||= Time.current
  end
end

# app/models/concerns/event_sourceable.rb
module EventSourceable
  extend ActiveSupport::Concern
  
  included do
    has_many :domain_events, as: :aggregate
    
    after_save :publish_pending_events
  end
  
  def apply_event(event_type, data)
    @pending_events ||= []
    @pending_events << {
      event_type: event_type,
      event_data: data.to_json,
      occurred_at: Time.current
    }
    
    # Apply to current state
    handle_event(event_type, data)
  end
  
  def rebuild_from_events
    # Rebuild state จาก events
    domain_events.each do |event|
      handle_event(event.event_type, event.data)
    end
    self
  end
  
  private
  
  def publish_pending_events
    return unless @pending_events&.any?
    
    @pending_events.each do |event|
      domain_events.create!(event)
      
      # Publish to message bus
      EventBus.publish(
        event_type: event[:event_type],
        aggregate_id: id,
        aggregate_type: self.class.name,
        data: event[:event_data]
      )
    end
    
    @pending_events = []
  end
end
```

---

## 6. Load Balancing Considerations

### 6.1 Database Connection Pool Management

```ruby
# config/initializers/connection_pool.rb
ActiveSupport.on_load(:active_record) do
  # ตั้งค่า pool size ให้เหมาะกับจำนวน threads และ workers
  web_workers = ENV.fetch('WEB_CONCURRENCY') { 2 }.to_i
  threads_per_worker = ENV.fetch('RAILS_MAX_THREADS') { 5 }.to_i
  
  # Total connections = workers * threads + buffer
  optimal_pool_size = (web_workers * threads_per_worker) + 2
  
  ActiveRecord::Base.connection_pool.disconnect!
  
  db_config = ActiveRecord::Base.configurations.find_db_config(Rails.env)
  new_config = db_config.configuration_hash.merge(pool: optimal_pool_size)
  
  Rails.logger.info("Setting DB pool size to #{optimal_pool_size}")
end

# Monitoring connection pool
Rails.application.config.after_initialize do
  Thread.new do
    loop do
      pool = ActiveRecord::Base.connection_pool
      stats = {
        size: pool.size,
        checked_out: pool.stat[:busy],
        waiting: pool.stat[:waiting],
        dead: pool.stat[:dead]
      }
      
      StatsD.gauge('db.pool.size', stats[:size])
      StatsD.gauge('db.pool.checked_out', stats[:checked_out])
      StatsD.gauge('db.pool.waiting', stats[:waiting])
      
      if stats[:waiting] > 0
        Rails.logger.warn("DB connection pool: #{stats[:waiting]} threads waiting!")
      end
      
      sleep 30
    end
  end
end
```

### 6.2 HTTP Client Pooling

```ruby
# config/initializers/faraday.rb
# Persistent connections สำหรับ external services
PAYMENT_GATEWAY_CONNECTION = Faraday.new(
  url: ENV['PAYMENT_GATEWAY_URL'],
  headers: { 'Content-Type' => 'application/json' }
) do |f|
  f.use :json
  f.response :logger if Rails.env.development?
  
  # Connection pooling
  f.adapter :net_http_persistent, pool_size: 5 do |http|
    http.idle_timeout = 100
    http.read_timeout = 10
    http.open_timeout = 5
  end
  
  # Retry on failure
  f.request :retry, {
    max: 3,
    interval: 0.5,
    interval_randomness: 0.5,
    backoff_factor: 2,
    exceptions: [Faraday::ConnectionFailed, Faraday::TimeoutError]
  }
end
```

### 6.3 Caching Strategy

```ruby
# app/services/caching_service.rb
class CachingService
  # Multi-level caching
  
  # L1: In-process memory cache (fastest)
  L1_CACHE = ActiveSupport::Cache::MemoryStore.new(size: 64.megabytes)
  
  # L2: Redis cache (shared across instances)
  L2_CACHE = Rails.cache
  
  def self.fetch(key, expires_in: 5.minutes, &block)
    # ลอง L1 ก่อน
    l1_result = L1_CACHE.fetch(key, expires_in: [expires_in, 1.minute].min)
    return l1_result if l1_result
    
    # ลอง L2
    l2_result = L2_CACHE.fetch(key, expires_in: expires_in) do
      # ถ้าไม่มีใน L2 ด้วย ไปดึงจาก source
      block.call
    end
    
    # Store ใน L1
    L1_CACHE.write(key, l2_result, expires_in: [expires_in / 5, 1.minute].min)
    
    l2_result
  end
  
  def self.invalidate(key)
    L1_CACHE.delete(key)
    L2_CACHE.delete(key)
    
    # Broadcast invalidation ไปยัง servers อื่น
    ActionCable.server.broadcast('cache_invalidation', { key: key })
  end
  
  def self.invalidate_pattern(pattern)
    # ลบ L1 cache ที่ match pattern
    L1_CACHE.instance_variable_get(:@data)&.keys&.select { |k| k.to_s.match?(pattern) }
           &.each { |k| L1_CACHE.delete(k) }
    
    # ลบ L2 cache ด้วย scan
    if L2_CACHE.respond_to?(:redis)
      L2_CACHE.redis.with do |redis|
        keys = redis.scan_each(match: "#{L2_CACHE.key_prefix}#{pattern}*").to_a
        redis.del(*keys) if keys.any?
      end
    end
  end
end
```

### 6.4 Auto-scaling Triggers

```ruby
# app/services/auto_scaling_service.rb
class AutoScalingService
  METRICS_WINDOW = 5.minutes
  
  SCALE_OUT_RULES = [
    { metric: :cpu_utilization, threshold: 70, action: :scale_out },
    { metric: :memory_utilization, threshold: 80, action: :scale_out },
    { metric: :request_queue_depth, threshold: 100, action: :scale_out },
    { metric: :avg_response_time_ms, threshold: 500, action: :scale_out }
  ].freeze
  
  SCALE_IN_RULES = [
    { metric: :cpu_utilization, threshold: 30, action: :scale_in },
    { metric: :request_queue_depth, threshold: 10, action: :scale_in }
  ].freeze
  
  def self.evaluate!
    metrics = collect_metrics
    
    should_scale_out = SCALE_OUT_RULES.any? do |rule|
      metrics[rule[:metric]] > rule[:threshold]
    end
    
    should_scale_in = !should_scale_out && SCALE_IN_RULES.all? do |rule|
      metrics[rule[:metric]] < rule[:threshold]
    end
    
    if should_scale_out
      Rails.logger.info("Scaling OUT based on metrics: #{metrics}")
      scale_out(metrics)
    elsif should_scale_in
      Rails.logger.info("Scaling IN based on metrics: #{metrics}")
      scale_in(metrics)
    end
  end
  
  def self.collect_metrics
    {
      cpu_utilization: fetch_cpu_utilization,
      memory_utilization: fetch_memory_utilization,
      request_queue_depth: Sidekiq::Stats.new.queued,
      avg_response_time_ms: fetch_avg_response_time
    }
  end
  
  private
  
  def self.scale_out(metrics)
    # AWS Auto Scaling
    client = Aws::AutoScaling::Client.new
    group = client.describe_auto_scaling_groups(
      auto_scaling_group_names: [ENV['ASG_NAME']]
    ).auto_scaling_groups.first
    
    new_capacity = [group.desired_capacity + 1, group.max_size].min
    
    client.set_desired_capacity(
      auto_scaling_group_name: ENV['ASG_NAME'],
      desired_capacity: new_capacity
    )
    
    Rails.logger.info("Scaled out to #{new_capacity} instances")
  end
  
  def self.fetch_avg_response_time
    Rails.cache.fetch('metrics:avg_response_time', expires_in: 1.minute) do
      # Fetch จาก metrics store
      50.0  # Default
    end
  end
end
```

---

## แบบฝึกหัด (20 ข้อพร้อมเฉลย)

### ข้อ 1: Stateless Session
**คำถาม:** แปลง session-based authentication เป็น token-based เพื่อรองรับ horizontal scaling

**เฉลย:**
```ruby
# app/services/jwt_service.rb
class JwtService
  SECRET = Rails.application.credentials.secret_key_base
  ALGORITHM = 'HS256'.freeze
  DEFAULT_EXPIRY = 24.hours
  
  def self.encode(payload, expiry: DEFAULT_EXPIRY)
    payload = payload.merge(
      iat: Time.current.to_i,
      exp: expiry.from_now.to_i,
      jti: SecureRandom.uuid
    )
    JWT.encode(payload, SECRET, ALGORITHM)
  end
  
  def self.decode(token)
    decoded = JWT.decode(token, SECRET, true, algorithm: ALGORITHM)
    decoded.first.with_indifferent_access
  rescue JWT::ExpiredSignature
    raise TokenExpiredError
  rescue JWT::DecodeError
    raise InvalidTokenError
  end
  
  # Token blacklist สำหรับ logout
  def self.blacklist!(jti, expiry:)
    Rails.cache.write("jwt:blacklist:#{jti}", true, expires_in: expiry)
  end
  
  def self.blacklisted?(jti)
    Rails.cache.exist?("jwt:blacklist:#{jti}")
  end
end
```

### ข้อ 2: Read Replica Configuration
**คำถาม:** configure Rails ให้ใช้ read replica สำหรับ analytics queries โดยอัตโนมัติ

**เฉลย:**
```ruby
# app/models/concerns/analytics_query.rb
module AnalyticsQuery
  extend ActiveSupport::Concern
  
  class_methods do
    def analytics(&block)
      connected_to(role: :reading) do
        block.call
      end
    end
    
    def analytics_count
      analytics { count }
    end
    
    def analytics_sum(column)
      analytics { sum(column) }
    end
    
    def analytics_group(column, &block)
      analytics { group(column).instance_eval(&block) }
    end
  end
end

# app/models/order.rb
class Order < ApplicationRecord
  include AnalyticsQuery
  
  # ใช้งาน: Order.analytics { where(created_at: last_month).sum(:total) }
end

# app/services/analytics_service.rb
class AnalyticsService
  def monthly_revenue(year:, month:)
    Order.analytics do
      where(
        status: 'completed',
        created_at: Date.new(year, month, 1).beginning_of_month..Date.new(year, month, 1).end_of_month
      ).sum(:total)
    end
  end
end
```

### ข้อ 3: Sidekiq Batch Processing
**คำถาม:** ใช้ Sidekiq batches สำหรับ process รูปภาพ 10,000 รูปพร้อมกัน พร้อม progress tracking

**เฉลย:**
```ruby
class ImageProcessingBatchJob < ApplicationJob
  def perform(image_ids)
    batch = Sidekiq::Batch.new
    batch.description = "Process #{image_ids.count} images"
    batch.on(:complete, self.class, 'count' => image_ids.count)
    
    batch.jobs do
      image_ids.each_slice(100) do |slice|
        slice.each do |id|
          ProcessImageJob.perform_async(id)
        end
      end
    end
    
    # Track progress
    Rails.cache.write("batch:#{batch.bid}:total", image_ids.count)
    Rails.cache.write("batch:#{batch.bid}:processed", 0)
    
    batch.bid
  end
  
  def on_complete(status, options)
    Rails.logger.info("Image batch complete: #{status.successes}/#{options['count']}")
    ActionCable.server.broadcast('admin', {
      type: 'batch_complete',
      successes: status.successes,
      failures: status.failures
    })
  end
end
```

### ข้อ 4: Kafka Consumer Group
**คำถาม:** setup Kafka consumer group สำหรับ process order events จากหลาย partitions

**เฉลย:**
```ruby
# config/karafka.rb - Consumer group configuration
class KarafkaApp < Karafka::App
  setup do |config|
    config.kafka = { 'bootstrap.servers': ENV['KAFKA_BROKERS'] }
    config.consumer_group = 'order-processors'
  end
  
  routes.draw do
    consumer_group :order_processors do
      topic 'order.events' do
        consumer OrderEventsConsumer
        # 6 partitions -> 6 consumers max
      end
    end
    
    # Separate consumer group สำหรับ analytics
    consumer_group :order_analytics do
      topic 'order.events' do
        consumer OrderAnalyticsConsumer
        start_from_beginning true  # Process ทุก messages
      end
    end
  end
end
```

### ข้อ 5: CQRS Command Validation
**คำถาม:** สร้าง command object สำหรับ update user profile ที่มี validation ครบถ้วน

**เฉลย:**
```ruby
class UpdateUserProfileCommand < BaseCommand
  attribute :user_id, :integer
  attribute :name, :string
  attribute :bio, :string
  attribute :website_url, :string
  attribute :location, :string
  attribute :avatar, # ActionDispatch::Http::UploadedFile
  
  validates :user_id, presence: true
  validates :name, length: { minimum: 2, maximum: 100 }, allow_blank: true
  validates :bio, length: { maximum: 500 }, allow_blank: true
  validates :website_url, url: true, allow_blank: true
  validate :validate_avatar
  
  private
  
  def validate_avatar
    return unless avatar.present?
    
    unless ['image/jpeg', 'image/png', 'image/gif'].include?(avatar.content_type)
      errors.add(:avatar, 'must be JPEG, PNG, or GIF')
    end
    
    if avatar.size > 5.megabytes
      errors.add(:avatar, 'must be smaller than 5MB')
    end
  end
end
```

### ข้อ 6-20 (สรุปเฉลย):

**ข้อ 6: Database Sharding Lookup**
```ruby
# Consistent hashing สำหรับ tenant-based sharding
class ShardLookup
  SHARDS = 8
  
  def self.for_tenant(tenant_id)
    shard_number = Zlib.crc32(tenant_id.to_s) % SHARDS
    :"shard_#{shard_number + 1}"
  end
end
```

**ข้อ 7: RabbitMQ Dead Letter Queue**
```ruby
# config/initializers/rabbitmq_setup.rb
channel = RABBITMQ_CONNECTION.create_channel
channel.queue('order.events.dlq', durable: true)  # Dead letter queue

channel.queue('order.events', 
  durable: true,
  arguments: {
    'x-dead-letter-exchange' => 'dlx',
    'x-dead-letter-routing-key' => 'order.events.dlq',
    'x-message-ttl' => 86400000  # 24 hours
  }
)
```

**ข้อ 8: Cache Warming Strategy**
```ruby
# lib/tasks/cache_warm.rake
namespace :cache do
  task warm: :environment do
    puts 'Warming up cache...'
    
    # Popular products
    Product.popular.limit(100).each do |product|
      Rails.cache.fetch("product:#{product.id}", expires_in: 1.hour) do
        ProductSerializer.new(product).serializable_hash
      end
    end
    
    # Common queries
    Rails.cache.fetch('products:featured', expires_in: 1.hour) do
      Product.featured.limit(20).as_json
    end
    
    puts 'Cache warming complete!'
  end
end
```

**ข้อ 9: Connection Pool Monitoring**
```ruby
class DatabasePoolMonitor
  def self.report
    pools = ActiveRecord::Base.connection_handler.connection_pools
    
    pools.map do |pool|
      {
        name: pool.db_config.name,
        size: pool.size,
        checked_out: pool.stat[:busy],
        available: pool.stat[:idle],
        waiting: pool.stat[:waiting]
      }
    end
  end
  
  def self.alert_if_exhausted!
    report.each do |pool|
      if pool[:waiting] > 5
        AlertService.warning("DB pool #{pool[:name]} has #{pool[:waiting]} threads waiting!")
      end
    end
  end
end
```

**ข้อ 10: Horizontal Pod Autoscaling**
```yaml
# kubernetes/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-web
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

**ข้อ 11: Event Sourcing Projection**
```ruby
class OrderProjection
  def self.build
    DomainEvent.where(aggregate_type: 'Order').order(:occurred_at).each do |event|
      case event.event_type
      when 'order_created'
        OrderSummary.create!(event.data.slice(:order_id, :user_id, :total))
      when 'order_paid'
        OrderSummary.find_by(order_id: event.data[:order_id])&.update!(status: 'paid')
      when 'order_cancelled'
        OrderSummary.find_by(order_id: event.data[:order_id])&.update!(status: 'cancelled')
      end
    end
  end
end
```

**ข้อ 12: Distributed Locking**
```ruby
# app/services/distributed_lock.rb
class DistributedLock
  def self.acquire(key, timeout: 30.seconds, &block)
    redis = Redis.new(url: ENV['REDIS_URL'])
    lock_key = "lock:#{key}"
    lock_value = SecureRandom.uuid
    
    acquired = redis.set(lock_key, lock_value, nx: true, ex: timeout.to_i)
    
    if acquired
      begin
        block.call
      ensure
        # Release lock (Lua script for atomic check-and-delete)
        redis.eval(
          "if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end",
          keys: [lock_key], argv: [lock_value]
        )
      end
    else
      raise LockAcquisitionError, "Could not acquire lock for #{key}"
    end
  end
end
```

**ข้อ 13: Queue Priority**
```ruby
# config/sidekiq.yml
:queues:
  - [critical, 10]  # ทำงานบ่อยที่สุด
  - [payments, 8]
  - [emails, 5]
  - [default, 3]
  - [reports, 1]    # ทำงานน้อยที่สุด

# Job ที่ตั้งค่า queue priority
class PaymentProcessingJob < ApplicationJob
  queue_as :payments
  sidekiq_options retry: 3, dead: false, unique: :until_executed
end
```

**ข้อ 14: API Response Caching**
```ruby
# app/controllers/api/v1/products_controller.rb
def index
  cache_key = "api/v1/products/#{request.query_string}"
  
  cached = Rails.cache.read(cache_key)
  if cached
    response.headers['X-Cache'] = 'HIT'
    render json: cached
  else
    response.headers['X-Cache'] = 'MISS'
    products = Product.active.page(params[:page])
    data = ProductSerializer.new(products).serializable_hash
    Rails.cache.write(cache_key, data, expires_in: 5.minutes)
    render json: data
  end
end
```

**ข้อ 15: Background Job Deduplication**
```ruby
class EmailNotificationJob < ApplicationJob
  sidekiq_options unique: :until_executed,
                  unique_args: ->(args) { [args.first] }
  
  def perform(user_id, notification_type)
    user = User.find(user_id)
    NotificationMailer.send(notification_type, user).deliver_now
  end
end
```

**ข้อ 16: Database Partitioning**
```ruby
# db/migrate/create_partitioned_events.rb
class CreatePartitionedEvents < ActiveRecord::Migration[7.1]
  def up
    execute <<~SQL
      CREATE TABLE events (
        id bigserial,
        user_id bigint NOT NULL,
        event_type varchar(100) NOT NULL,
        data jsonb,
        created_at timestamptz NOT NULL DEFAULT NOW()
      ) PARTITION BY RANGE (created_at);
      
      CREATE TABLE events_2024_01 PARTITION OF events
        FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
      
      CREATE TABLE events_2024_02 PARTITION OF events
        FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
    SQL
  end
end
```

**ข้อ 17: Service Mesh Configuration**
```yaml
# kubernetes/istio-virtual-service.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - match:
        - uri:
            prefix: /api/v2
      route:
        - destination:
            host: myapp-v2
            port:
              number: 3000
          weight: 10  # Canary: 10% to v2
        - destination:
            host: myapp-v1
            port:
              number: 3000
          weight: 90
```

**ข้อ 18: Multi-Region Deployment**
```ruby
# config/initializers/multi_region.rb
REGIONS = {
  'ap-southeast-1' => { db_host: 'db.ap.example.com', redis_host: 'redis.ap.example.com' },
  'eu-west-1' => { db_host: 'db.eu.example.com', redis_host: 'redis.eu.example.com' },
  'us-east-1' => { db_host: 'db.us.example.com', redis_host: 'redis.us.example.com' }
}.freeze

current_region = ENV['AWS_REGION'] || 'ap-southeast-1'
region_config = REGIONS[current_region]

ENV['DATABASE_HOST'] ||= region_config[:db_host]
ENV['REDIS_HOST'] ||= region_config[:redis_host]
```

**ข้อ 19: Circuit Breaker Pattern**
```ruby
# app/services/external_service.rb
class ExternalService
  CIRCUIT_BREAKER = CircuitBreaker.new(
    threshold: 5,          # 5 failures ต่อ 1 นาที
    timeout: 60,           # Open for 60 seconds
    monitor: StatsD,
    error_handler: Sentry
  )
  
  def self.call(method, *args)
    CIRCUIT_BREAKER.call do
      HTTP.timeout(5).public_send(method, *args)
    end
  rescue Faraday::Error, CircuitBreaker::Open => e
    { error: 'Service temporarily unavailable', fallback: true }
  end
end
```

**ข้อ 20: Complete Scaling Checklist**
```ruby
# lib/tasks/scaling_check.rake
namespace :scaling do
  task check: :environment do
    checks = {
      stateless_session: check_stateless_session,
      redis_caching: check_redis_caching,
      background_jobs: check_background_jobs,
      db_pool_size: check_db_pool_size,
      no_local_files: check_no_local_files,
      health_endpoint: check_health_endpoint
    }
    
    failed = checks.select { |_, v| !v[:passed] }
    
    if failed.any?
      puts "SCALING READINESS FAILED:"
      failed.each { |k, v| puts "  #{k}: #{v[:message]}" }
      exit 1
    else
      puts "Application is ready for horizontal scaling!"
    end
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Horizontal Scaling** - Stateless design, Puma configuration, load balancing
2. **Read Replicas** - Rails multi-database, automatic routing, query optimization
3. **Sharding** - Horizontal partitioning, tenant-based sharding
4. **Message Queues** - Sidekiq Pro, RabbitMQ, Kafka สำหรับ async processing
5. **CQRS** - Command/Query separation, Event Sourcing, Projections
6. **Auto-scaling** - Connection pooling, caching, scaling triggers

หลักสำคัญ: แอปต้องเป็น stateless ก่อนจึงจะ scale horizontally ได้ การใช้ Redis สำหรับ sessions, cache, และ job queue เป็นพื้นฐานที่จำเป็น
