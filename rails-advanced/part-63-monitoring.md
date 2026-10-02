# Part 63: Monitoring และ Logging ใน Ruby on Rails

## ขั้นตอนที่ 1371-1390: ระบบ Monitoring และ Log Management

---

## ขั้นตอนที่ 1371: Rails Logging

### Log Levels

```ruby
# log levels (ลำดับจากน้อยไปมาก)
Rails.logger.debug "Debug message"   # level 0
Rails.logger.info  "Info message"    # level 1
Rails.logger.warn  "Warning message" # level 2
Rails.logger.error "Error message"   # level 3
Rails.logger.fatal "Fatal message"   # level 4

# ตั้งค่า level ใน environment
# config/environments/production.rb
config.log_level = :info  # จะ log ตั้งแต่ info ขึ้นไป
```

### Logging ใน Rails

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    
    Rails.logger.info "Creating post for user #{current_user.id}"
    
    if @post.save
      Rails.logger.info "Post #{@post.id} created successfully"
      redirect_to @post
    else
      Rails.logger.warn "Failed to create post: #{@post.errors.full_messages}"
      render :new
    end
  rescue => e
    Rails.logger.error "Unexpected error creating post: #{e.message}"
    Rails.logger.error e.backtrace.join("\n")
    raise
  end
end
```

### Tagged Logging

```ruby
# เพิ่ม tags ให้ log entries
Rails.logger = ActiveSupport::TaggedLogging.new(Rails.logger)

# ใน controller
Rails.logger.tagged("User:#{current_user.id}", "Request:#{request.uuid}") do
  Rails.logger.info "Processing request"
  # logs: [User:123] [Request:abc] Processing request
end

# ใน middleware
class RequestLoggingMiddleware
  def call(env)
    request = ActionDispatch::Request.new(env)
    
    Rails.logger.tagged(request.request_id) do
      Rails.logger.info "#{request.method} #{request.path}"
      status, headers, body = @app.call(env)
      Rails.logger.info "Response: #{status}"
      [status, headers, body]
    end
  end
end
```

---

## ขั้นตอนที่ 1372: Structured Logging

### JSON Logging

```ruby
# config/environments/production.rb
config.log_formatter = proc do |severity, time, progname, msg|
  {
    level: severity,
    time: time.iso8601,
    program: progname,
    message: msg
  }.to_json + "\n"
end
```

### Custom Logger

```ruby
# app/services/structured_logger.rb
class StructuredLogger
  def self.log(event:, **attributes)
    log_entry = {
      timestamp: Time.current.iso8601,
      event: event,
      environment: Rails.env,
      app_version: ENV['APP_VERSION'],
      request_id: Current.request_id
    }.merge(attributes)
    
    Rails.logger.info log_entry.to_json
  end
  
  def self.error(message:, exception: nil, **attributes)
    log_entry = {
      timestamp: Time.current.iso8601,
      level: "error",
      message: message,
      exception: exception&.class&.name,
      exception_message: exception&.message,
      backtrace: exception&.backtrace&.first(10)
    }.merge(attributes)
    
    Rails.logger.error log_entry.to_json
  end
end

# ใช้งาน
StructuredLogger.log(
  event: "user.logged_in",
  user_id: user.id,
  ip: request.remote_ip,
  browser: request.user_agent
)
```

---

## ขั้นตอนที่ 1373: Lograge Gem

```ruby
# Gemfile
gem 'lograge'
gem 'logstash-event'
```

```ruby
# config/environments/production.rb
config.lograge.enabled = true
config.lograge.formatter = Lograge::Formatters::Json.new

config.lograge.custom_options = lambda do |event|
  {
    host: event.payload[:host] || request&.host,
    user_id: event.payload[:user_id],
    params: event.payload[:params]&.except('controller', 'action', 'format', '_method', 'authenticity_token'),
    exception: event.payload[:exception]&.first,
    exception_backtrace: event.payload[:exception_object]&.backtrace&.first(5)
  }.compact
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_log_tags
  
  def append_info_to_payload(payload)
    super
    payload[:user_id] = current_user&.id
    payload[:host] = request.host
    payload[:request_id] = request.request_id
  end
  
  private
  
  def set_log_tags
    Rails.logger.push_tags(
      "user=#{current_user&.id || 'anonymous'}",
      "request=#{request.request_id}"
    )
  end
end
```

---

## ขั้นตอนที่ 1374: Error Tracking (Sentry)

```ruby
# Gemfile
gem 'sentry-ruby'
gem 'sentry-rails'
gem 'sentry-sidekiq'
```

```ruby
# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV['SENTRY_DSN']
  config.breadcrumbs_logger = [:active_support_logger, :http_logger]
  
  # Environment
  config.environment = Rails.env
  config.enabled_environments = %w[production staging]
  
  # Sampling
  config.traces_sample_rate = Rails.env.production? ? 0.1 : 1.0
  config.profiles_sample_rate = 0.1
  
  # Filter sensitive data
  config.before_send = lambda do |event, hint|
    # ลบ sensitive data
    if event.request
      event.request.data&.delete('password')
      event.request.data&.delete('credit_card')
    end
    event
  end
end
```

```ruby
# ใช้งาน
begin
  risky_operation
rescue => e
  Sentry.capture_exception(e, extra: { user_id: current_user&.id })
  raise
end

# Capture message
Sentry.capture_message("Unusual activity detected", 
  level: :warning,
  extra: { user_id: user.id }
)

# User context
Sentry.configure_scope do |scope|
  scope.set_user(
    id: current_user.id,
    email: current_user.email,
    name: current_user.name
  )
end
```

---

## ขั้นตอนที่ 1375: Performance Monitoring (New Relic)

```ruby
# Gemfile
gem 'newrelic_rpm'
```

```yaml
# config/newrelic.yml
production:
  license_key: <%= ENV['NEW_RELIC_LICENSE_KEY'] %>
  app_name: My Rails App (Production)
  monitor_mode: true
  log_level: info
  transaction_tracer:
    enabled: true
    transaction_threshold: 0.001
    record_sql: obfuscated
  error_collector:
    enabled: true
  browser_monitoring:
    auto_instrument: true
```

### Custom Instrumentation

```ruby
# ใน Service Objects
class PaymentService
  include ::NewRelic::Agent::MethodTracer
  
  add_method_tracer :process_payment, "Custom/PaymentService/process_payment"
  
  def process_payment(order)
    # ... payment logic
  end
end

# Manual tracing
::NewRelic::Agent::Transaction.start_web_transaction("Custom/BulkImport") do
  BulkImportJob.perform(file_path)
end

# Record custom metrics
::NewRelic::Agent.record_metric("Custom/Cart/ItemsAdded", 1)
::NewRelic::Agent.record_metric("Custom/Revenue/Total", order.total)
```

---

## ขั้นตอนที่ 1376: Scout APM

```ruby
# Gemfile
gem 'scout_apm'
```

```yaml
# config/scout_apm.yml
common: &defaults
  name: My Rails App
  
production:
  <<: *defaults
  key: <%= ENV['SCOUT_APM_KEY'] %>
  monitor: true
  
development:
  <<: *defaults
  monitor: false
```

### Custom Scout Instrumentation

```ruby
# ใน controllers/services
include ScoutApm::Tracing

# Trace custom method
def complex_calculation
  req = ScoutApm::Tracer.instrument("Custom", "complex_calculation") do
    # ... heavy calculation
  end
end
```

---

## ขั้นตอนที่ 1377: Health Checks

```ruby
# Gemfile
gem 'health_check'
# หรือ
gem 'ok_computer'
```

### Custom Health Check

```ruby
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  skip_before_action :authenticate_user!
  
  def index
    checks = perform_checks
    status = checks.values.all? { |v| v[:status] == "ok" } ? :ok : :service_unavailable
    
    render json: {
      status: status == :ok ? "healthy" : "unhealthy",
      checks: checks,
      version: ENV.fetch("APP_VERSION", "unknown"),
      deployed_at: ENV.fetch("DEPLOYED_AT", "unknown"),
      timestamp: Time.current.iso8601
    }, status: status
  end
  
  def live
    render plain: "OK", status: :ok
  end
  
  def ready
    if ActiveRecord::Base.connection.execute("SELECT 1")
      render plain: "READY", status: :ok
    else
      render plain: "NOT READY", status: :service_unavailable
    end
  rescue => e
    render plain: "NOT READY: #{e.message}", status: :service_unavailable
  end
  
  private
  
  def perform_checks
    {
      database: check_database,
      redis: check_redis,
      sidekiq: check_sidekiq,
      disk_space: check_disk_space
    }
  end
  
  def check_database
    start = Time.current
    ActiveRecord::Base.connection.execute("SELECT 1")
    { status: "ok", latency_ms: ((Time.current - start) * 1000).round(2) }
  rescue => e
    { status: "error", message: e.message }
  end
  
  def check_redis
    start = Time.current
    Redis.new(url: ENV['REDIS_URL']).ping
    { status: "ok", latency_ms: ((Time.current - start) * 1000).round(2) }
  rescue => e
    { status: "error", message: e.message }
  end
  
  def check_sidekiq
    stats = Sidekiq::Stats.new
    dead = stats.dead_size
    
    if dead > 100
      { status: "warning", dead_jobs: dead }
    else
      { status: "ok", processed: stats.processed, failed: stats.failed }
    end
  rescue => e
    { status: "error", message: e.message }
  end
  
  def check_disk_space
    stat = Sys::Filesystem.stat("/")
    free_percent = (stat.blocks_available.to_f / stat.blocks) * 100
    
    status = free_percent < 10 ? "error" : free_percent < 20 ? "warning" : "ok"
    
    { status: status, free_percent: free_percent.round(2) }
  rescue
    { status: "unknown" }
  end
end
```

---

## ขั้นตอนที่ 1378: Custom Metrics

```ruby
# Gemfile
gem 'statsd-ruby'

# config/initializers/statsd.rb
$statsd = Statsd.new(
  ENV.fetch('STATSD_HOST', 'localhost'),
  ENV.fetch('STATSD_PORT', 8125).to_i
)

# app/services/metrics.rb
class Metrics
  def self.increment(metric, tags = {})
    $statsd.increment(metric, tags: tags_string(tags))
  end
  
  def self.timing(metric, duration_ms, tags = {})
    $statsd.timing(metric, duration_ms, tags: tags_string(tags))
  end
  
  def self.gauge(metric, value, tags = {})
    $statsd.gauge(metric, value, tags: tags_string(tags))
  end
  
  def self.time(metric, tags = {}, &block)
    start = Time.current
    result = yield
    duration_ms = ((Time.current - start) * 1000).round
    timing(metric, duration_ms, tags)
    result
  end
  
  private
  
  def self.tags_string(tags)
    tags.map { |k, v| "#{k}:#{v}" }
  end
end

# ใช้งาน
Metrics.increment("orders.created", user_type: "premium")
Metrics.gauge("cart.items", cart.items.count)
Metrics.time("payment.processing") do
  PaymentService.charge(order)
end
```

---

## ขั้นตอนที่ 1379-1380: Alerting

### Slack Alerts

```ruby
# app/services/slack_notifier.rb
class SlackNotifier
  WEBHOOK_URL = ENV['SLACK_WEBHOOK_URL']
  
  def self.alert(message:, level: :info, details: {})
    color = case level
            when :error then "danger"
            when :warning then "warning"
            else "good"
            end
    
    payload = {
      attachments: [{
        color: color,
        title: message,
        fields: details.map { |k, v| { title: k.to_s, value: v.to_s, short: true } },
        footer: "Rails App | #{Rails.env}",
        ts: Time.current.to_i
      }]
    }
    
    HTTParty.post(WEBHOOK_URL, body: payload.to_json, 
                  headers: { 'Content-Type' => 'application/json' })
  end
end

# ใช้งาน
SlackNotifier.alert(
  message: "Payment processing error",
  level: :error,
  details: {
    order_id: order.id,
    error: e.message,
    environment: Rails.env
  }
)
```

---

## แบบฝึกหัดที่ 1-20

### แบบฝึกหัดที่ 1
ตั้งค่า Rails logging สำหรับ production

### แบบฝึกหัดที่ 2
เพิ่ม Structured JSON logging

### แบบฝึกหัดที่ 3
ติดตั้งและตั้งค่า Lograge

### แบบฝึกหัดที่ 4
ตั้งค่า Sentry สำหรับ Error Tracking

### แบบฝึกหัดที่ 5
เพิ่ม User context ใน error reports

### แบบฝึกหัดที่ 6
ตั้งค่า Performance Monitoring ด้วย Scout APM

### แบบฝึกหัดที่ 7
สร้าง Custom Health Check endpoint

### แบบฝึกหัดที่ 8
เพิ่ม Kubernetes liveness/readiness probes

### แบบฝึกหัดที่ 9
สร้าง Custom Metrics ด้วย StatsD

### แบบฝึกหัดที่ 10
ตั้งค่า Slack Alerts สำหรับ critical errors

### แบบฝึกหัดที่ 11
Monitor Sidekiq jobs และ alerting

### แบบฝึกหัดที่ 12
Log Database slow queries

### แบบฝึกหัดที่ 13
ตั้งค่า Log rotation

### แบบฝึกหัดที่ 14
สร้าง Dashboard สำหรับ application metrics

### แบบฝึกหัดที่ 15
Implement distributed tracing

### แบบฝึกหัดที่ 16
ตั้งค่า Uptime monitoring

### แบบฝึกหัดที่ 17
Monitor Memory usage

### แบบฝึกหัดที่ 18
ตั้งค่า Error rate alerting

### แบบฝึกหัดที่ 19
สร้าง Performance regression detection

### แบบฝึกหัดที่ 20
สร้าง Complete Monitoring Stack:
- Structured logs (Lograge + JSON)
- Error tracking (Sentry)
- Performance monitoring (Scout)
- Health checks
- Custom metrics
- Alerting (Slack)

---

## สรุป Part 63

เราได้เรียนรู้:
1. Rails logging และ log levels
2. Structured logging และ JSON format
3. Lograge สำหรับ cleaner request logs
4. Error tracking ด้วย Sentry
5. Performance monitoring ด้วย New Relic/Scout
6. Health checks สำหรับ Kubernetes
7. Custom metrics และ alerting

Monitoring ช่วยให้เราทราบปัญหาก่อนที่ users จะรายงาน และช่วยให้ debug ปัญหาได้เร็วขึ้น
