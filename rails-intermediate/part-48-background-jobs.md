# Part 48: Background Jobs ด้วย Sidekiq

## ขั้นตอนที่ 1056-1080: การทำงานเบื้องหลัง

---

## ขั้นตอนที่ 1056: ทำไมต้องใช้ Background Jobs?

Tasks ที่ควรทำเบื้องหลัง:
- ส่งอีเมล (ใช้เวลา)
- Process ไฟล์ขนาดใหญ่
- เรียก External API
- Generate reports
- Image processing
- Sending notifications
- Data import/export

```ruby
# ❌ Bad: ทำใน request cycle (ทำให้ user รอ)
class UsersController < ApplicationController
  def create
    @user = User.create!(user_params)
    
    # ส่ง email โดยตรง - ทำให้ response ช้า
    UserMailer.welcome(@user).deliver_now
    
    # Resize avatar - ใช้เวลานาน
    ImageProcessor.resize(@user.avatar, [200, 200])
    
    redirect_to @user
  end
end

# ✅ Good: ทำเบื้องหลัง
class UsersController < ApplicationController
  def create
    @user = User.create!(user_params)
    
    # ส่ง email เบื้องหลัง
    UserMailer.welcome(@user).deliver_later
    
    # Process ภาพเบื้องหลัง
    ProcessAvatarJob.perform_later(@user.id)
    
    redirect_to @user  # Response ทันที
  end
end
```

## ขั้นตอนที่ 1057: Active Job Abstraction

Rails มี Active Job เป็น abstraction layer สำหรับ background jobs

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Retry on failure
  retry_on StandardError, wait: 5.seconds, attempts: 3
  
  # Discard job ถ้า record ไม่มีอยู่แล้ว
  discard_on ActiveRecord::RecordNotFound
  
  # Global error handling
  rescue_from(StandardError) do |exception|
    Rails.logger.error "Job failed: #{exception.message}"
    Sentry.capture_exception(exception)
    raise exception
  end
end
```

```ruby
# สร้าง job
rails generate job email_notification

# app/jobs/email_notification_job.rb
class EmailNotificationJob < ApplicationJob
  queue_as :default
  
  def perform(user_id, notification_type, data = {})
    user = User.find(user_id)
    
    case notification_type
    when 'welcome'
      UserMailer.welcome(user).deliver_now
    when 'password_reset'
      UserMailer.password_reset(user).deliver_now
    when 'new_follower'
      follower = User.find(data['follower_id'])
      UserMailer.new_follower(user, follower).deliver_now
    end
  end
end

# การ enqueue job
EmailNotificationJob.perform_later(user.id, 'welcome')

# ส่งทันที (synchronous)
EmailNotificationJob.perform_now(user.id, 'welcome')

# Schedule
EmailNotificationJob.set(wait: 5.minutes).perform_later(user.id, 'welcome')
EmailNotificationJob.set(wait_until: 3.days.from_now).perform_later(user.id, 'reminder')

# กำหนด queue
EmailNotificationJob.set(queue: :mailers).perform_later(user.id, 'welcome')
```

## ขั้นตอนที่ 1058: การติดตั้ง Sidekiq

```ruby
# Gemfile
gem 'sidekiq'
gem 'sidekiq-cron'       # สำหรับ scheduled jobs
gem 'redis'              # Redis client
```

```bash
bundle install
```

```ruby
# config/application.rb
class Application < Rails::Application
  # กำหนด Active Job backend
  config.active_job.queue_adapter = :sidekiq
end
```

```yaml
# config/sidekiq.yml
:concurrency: 10
:timeout: 25
:max_retries: 3

:queues:
  - critical
  - default
  - mailers
  - low

:scheduler:
  :rufus_scheduler_options:
    :max_work_threads: 1
```

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV['REDIS_URL'] || 'redis://localhost:6379/0' }
  
  # ตั้งค่า retry ขั้นสูง
  config.death_handlers << ->(job, _ex) {
    Rails.logger.error "Job died: #{job}"
    Sentry.capture_message("Sidekiq job died", extra: { job: job })
  }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV['REDIS_URL'] || 'redis://localhost:6379/0' }
end

# Logger
Sidekiq.logger = Rails.logger
```

## ขั้นตอนที่ 1059: สร้าง Sidekiq Jobs

```ruby
# app/workers/email_worker.rb (Sidekiq native worker)
class EmailWorker
  include Sidekiq::Worker
  
  sidekiq_options(
    queue: :mailers,
    retry: 5,
    dead: false,  # ไม่เก็บใน dead queue
    backtrace: true
  )
  
  def perform(user_id, email_type, options = {})
    user = User.find(user_id)
    
    case email_type.to_sym
    when :welcome
      UserMailer.welcome(user).deliver_now
    when :newsletter
      NewsletterMailer.weekly_digest(user).deliver_now
    end
  end
end

# Enqueue
EmailWorker.perform_async(user.id, 'welcome')
EmailWorker.perform_in(5.minutes, user.id, 'welcome')
EmailWorker.perform_at(3.days.from_now, user.id, 'reminder')
```

```ruby
# ตัวอย่าง jobs ที่ใช้บ่อย

# app/jobs/send_welcome_email_job.rb
class SendWelcomeEmailJob < ApplicationJob
  queue_as :mailers
  
  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome(user).deliver_now
    
    # บันทึก event
    UserEvent.create!(user: user, event_type: 'welcome_email_sent')
  end
end

# app/jobs/process_image_job.rb
class ProcessImageJob < ApplicationJob
  queue_as :default
  retry_on ImageProcessingError, wait: :exponentially_longer, attempts: 5
  
  def perform(attachment_id, variant_params = {})
    attachment = ActiveStorage::Attachment.find(attachment_id)
    blob = attachment.blob
    
    # Process image variants
    variants = [
      { resize_to_fill: [800, 600], format: :webp, quality: 85 },
      { resize_to_fill: [400, 300], format: :webp, quality: 80 },
      { resize_to_fill: [100, 100], format: :webp, quality: 75 }
    ]
    
    variants.each do |variant_opts|
      blob.variant(**variant_opts).processed
    end
    
    # อัพเดท status
    attachment.record.update!(images_processed: true)
  end
end

# app/jobs/cleanup_old_sessions_job.rb
class CleanupOldSessionsJob < ApplicationJob
  queue_as :low
  
  def perform
    deleted = Session.where("expires_at < ?", 7.days.ago).delete_all
    Rails.logger.info "Deleted #{deleted} old sessions"
  end
end
```

## ขั้นตอนที่ 1060: Job Queues

```ruby
# กำหนด queues และ priorities
class CriticalJob < ApplicationJob
  queue_as :critical  # ประมวลผลก่อน
end

class DefaultJob < ApplicationJob
  queue_as :default
end

class LowPriorityJob < ApplicationJob
  queue_as :low  # ประมวลผลทีหลัง
end

# Sidekiq จะประมวลผล queues ตามลำดับใน config
# :queues: [critical, default, low]
```

```ruby
# Dynamic queue selection
class NotificationJob < ApplicationJob
  queue_as do
    if self.arguments.first[:urgent]
      :critical
    else
      :default
    end
  end
  
  def perform(notification_params)
    # process notification
  end
end
```

## ขั้นตอนที่ 1061: Scheduled Jobs ด้วย Sidekiq Cron

```ruby
# Gemfile
gem 'sidekiq-cron'
```

```ruby
# config/sidekiq.yml
:schedule:
  daily_cleanup:
    cron: '0 2 * * *'  # ทุกวัน 02:00
    class: CleanupOldDataJob
    queue: low
  
  weekly_newsletter:
    cron: '0 9 * * 1'  # ทุกวันจันทร์ 09:00
    class: SendWeeklyNewsletterJob
    queue: mailers
  
  hourly_stats:
    cron: '0 * * * *'  # ทุกชั่วโมง
    class: UpdateStatsJob
    queue: default
  
  every_5_minutes:
    cron: '*/5 * * * *'
    class: CheckPendingOrdersJob
    queue: default
```

```ruby
# config/initializers/sidekiq_cron.rb
Sidekiq::Cron::Job.load_from_hash(Rails.application.config.sidekiq_cron_jobs)
```

```ruby
# app/jobs/daily_digest_job.rb
class DailyDigestJob < ApplicationJob
  queue_as :mailers
  
  def perform
    # ส่ง daily digest ให้ users ที่ subscribe
    User.subscribed_to_digest.find_each do |user|
      # ใช้ batching เพื่อไม่ให้โหลดเยอะ
      DigestMailer.daily_digest(user).deliver_later
    end
    
    Rails.logger.info "Daily digest scheduled for #{User.subscribed_to_digest.count} users"
  end
end

# app/jobs/cleanup_expired_tokens_job.rb
class CleanupExpiredTokensJob < ApplicationJob
  queue_as :low
  
  def perform
    ActiveRecord::Base.transaction do
      expired_count = RevokedToken.where("expires_at < ?", Time.current).delete_all
      old_sessions = Session.where("expires_at < ?", 30.days.ago).delete_all
      
      Rails.logger.info "Cleaned up: #{expired_count} tokens, #{old_sessions} sessions"
    end
  end
end
```

## ขั้นตอนที่ 1062: Sidekiq Web UI

```ruby
# config/routes.rb
require 'sidekiq/web'

Rails.application.routes.draw do
  # Mount Sidekiq web UI
  mount Sidekiq::Web => '/sidekiq'
  
  # Secure with authentication
  authenticate :user, ->(u) { u.admin? } do
    mount Sidekiq::Web => '/admin/sidekiq'
  end
end
```

```ruby
# หรือใช้ HTTP Basic Auth
require 'sidekiq/web'

Sidekiq::Web.use(Rack::Auth::Basic) do |username, password|
  ActiveSupport::SecurityUtils.secure_compare(
    ::Digest::SHA256.hexdigest(username),
    ::Digest::SHA256.hexdigest(ENV['SIDEKIQ_USERNAME'])
  ) & ActiveSupport::SecurityUtils.secure_compare(
    ::Digest::SHA256.hexdigest(password),
    ::Digest::SHA256.hexdigest(ENV['SIDEKIQ_PASSWORD'])
  )
end

Rails.application.routes.draw do
  mount Sidekiq::Web => '/sidekiq'
end
```

## ขั้นตอนที่ 1063: Error Handling ใน Jobs

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Retry configuration
  retry_on StandardError, 
    wait: :exponentially_longer,  # 5s, 10s, 20s, 40s...
    attempts: 5
  
  retry_on Net::TimeoutError, wait: 30.seconds, attempts: 3
  
  # Discard specific errors
  discard_on ActiveRecord::RecordNotFound
  discard_on ActiveJob::DeserializationError
  
  # Custom error handling
  rescue_from StandardError do |exception|
    # Log the error
    Rails.logger.error "#{self.class.name} failed: #{exception.message}"
    Rails.logger.error exception.backtrace.first(5).join("\n")
    
    # Report to Sentry/Bugsnag
    Sentry.capture_exception(exception, extra: { job: self.class.name, args: arguments })
    
    # Re-raise to trigger retry
    raise exception
  end
  
  # Callbacks
  before_perform :log_start
  after_perform :log_completion
  around_perform :measure_performance
  
  private
  
  def log_start
    Rails.logger.info "[#{self.class.name}] Starting at #{Time.current}"
  end
  
  def log_completion
    Rails.logger.info "[#{self.class.name}] Completed at #{Time.current}"
  end
  
  def measure_performance
    start = Time.current
    yield
    duration = Time.current - start
    Rails.logger.info "[#{self.class.name}] Duration: #{duration.round(3)}s"
  end
end
```

```ruby
# Job ที่มี error handling ครบ
class ImportDataJob < ApplicationJob
  queue_as :default
  retry_on StandardError, wait: 30.seconds, attempts: 3
  
  def perform(file_path, user_id)
    user = User.find(user_id)
    
    begin
      # Track progress
      update_progress(0)
      
      records = parse_file(file_path)
      total = records.count
      
      imported = 0
      failed = 0
      errors = []
      
      records.each_with_index do |record, index|
        begin
          import_record(record)
          imported += 1
        rescue ActiveRecord::RecordInvalid => e
          failed += 1
          errors << { row: index + 1, error: e.message }
        end
        
        # Update progress every 10%
        update_progress((index + 1.0 / total * 100).round) if (index % (total / 10)).zero?
      end
      
      # ส่งรายงาน
      ImportMailer.completion_report(user, imported, failed, errors).deliver_later
      
    rescue Errno::ENOENT => e
      # ไฟล์ไม่พบ
      ImportMailer.import_failed(user, "ไม่พบไฟล์: #{e.message}").deliver_later
      raise e
    ensure
      # ล้างไฟล์ temp
      File.delete(file_path) if File.exist?(file_path)
    end
  end
  
  private
  
  def parse_file(path)
    case File.extname(path)
    when '.csv'
      CSV.read(path, headers: true).map(&:to_h)
    when '.json'
      JSON.parse(File.read(path))
    else
      raise "ไม่รองรับประเภทไฟล์: #{File.extname(path)}"
    end
  end
  
  def import_record(record)
    User.create!(
      name: record['name'],
      email: record['email'],
      password: SecureRandom.hex(16)
    )
  end
  
  def update_progress(percentage)
    Rails.cache.write("import_progress_#{job_id}", percentage, expires_in: 1.hour)
  end
end
```

## ขั้นตอนที่ 1064: Job Callbacks

```ruby
class NotificationJob < ApplicationJob
  queue_as :default
  
  # ก่อนทำงาน
  before_perform do |job|
    Rails.logger.info "Starting job #{job.job_id}"
    Statsd.increment('jobs.started')
  end
  
  # หลังทำงานสำเร็จ
  after_perform do |job|
    Rails.logger.info "Completed job #{job.job_id}"
    Statsd.increment('jobs.completed')
  end
  
  # รอบ perform
  around_perform do |job, block|
    start = Time.current
    begin
      block.call
    rescue => e
      Statsd.increment('jobs.failed')
      raise e
    ensure
      duration = Time.current - start
      Statsd.timing('jobs.duration', duration * 1000)
    end
  end
  
  # Enqueue callbacks
  before_enqueue do |job|
    Rails.logger.info "Enqueueing #{job.class.name}"
  end
  
  after_enqueue do |job|
    # อาจจะบันทึกใน database
    JobRecord.create!(job_id: job.job_id, class_name: job.class.name)
  end
  
  def perform(notification_id)
    notification = Notification.find(notification_id)
    notification.deliver!
  end
end
```

## ขั้นตอนที่ 1065: Batch Jobs

```ruby
# ประมวลผล records เป็น batches
class ProcessUsersJob < ApplicationJob
  queue_as :default
  
  BATCH_SIZE = 100
  
  def perform(user_ids = nil)
    users = user_ids ? User.where(id: user_ids) : User.all
    
    users.find_in_batches(batch_size: BATCH_SIZE) do |batch|
      batch.each do |user|
        ProcessSingleUserJob.perform_later(user.id)
      end
    end
  end
end

class ProcessSingleUserJob < ApplicationJob
  queue_as :default
  
  def perform(user_id)
    user = User.find(user_id)
    # Process individual user
  end
end
```

```ruby
# ใช้ sidekiq-batch (Pro feature) หรือ custom
class BulkEmailJob < ApplicationJob
  def perform(user_ids, email_template)
    User.where(id: user_ids).find_each(batch_size: 50) do |user|
      # ส่ง email แต่ละคน
      SingleEmailJob.perform_later(user.id, email_template)
    end
  end
end
```

## ขั้นตอนที่ 1066: Job Uniqueness

```ruby
# ป้องกัน duplicate jobs ด้วย sidekiq-unique-jobs
# gem 'sidekiq-unique-jobs'

class UpdateStatsJob < ApplicationJob
  include SidekiqUniqueJobs::Worker
  
  sidekiq_options(
    unique: :until_executed,
    unique_args: ->(args) { [args.first] }  # unique by first arg
  )
  
  def perform(user_id)
    user = User.find(user_id)
    user.update_stats!
  end
end
```

```ruby
# Custom uniqueness ไม่ใช้ gem
class SyncDataJob < ApplicationJob
  queue_as :default
  
  def perform(resource_id, resource_type)
    lock_key = "sync_#{resource_type}_#{resource_id}"
    
    # ใช้ Redis lock
    return if already_running?(lock_key)
    
    with_lock(lock_key, 10.minutes) do
      # ทำงาน
      klass = resource_type.constantize
      record = klass.find(resource_id)
      record.sync_to_external_service!
    end
  end
  
  private
  
  def already_running?(key)
    $redis.exists?(key)
  end
  
  def with_lock(key, ttl)
    $redis.set(key, 1, ex: ttl.to_i, nx: true)
    
    begin
      yield
    ensure
      $redis.del(key)
    end
  end
end
```

## ขั้นตอนที่ 1067: Good Job (Alternative)

```ruby
# Gemfile
gem 'good_job'
```

```ruby
# config/application.rb
config.active_job.queue_adapter = :good_job
```

```ruby
# config/initializers/good_job.rb
GoodJob.configure do |config|
  config.max_threads = 5
  config.queues = '*'  # ทุก queues
  config.poll_interval = 30.seconds
  config.cleanup_preserved_jobs_before_seconds_ago = 86_400  # 1 วัน
end
```

```ruby
# Mount Good Job dashboard
# config/routes.rb
Rails.application.routes.draw do
  authenticate :user, ->(user) { user.admin? } do
    mount GoodJob::Engine => 'good_job'
  end
end
```

## ขั้นตอนที่ 1068: Delayed::Job (Lightweight Alternative)

```ruby
# Gemfile
gem 'delayed_job_active_record'
```

```bash
rails generate delayed_job:active_record
rails db:migrate
```

```ruby
# config/application.rb
config.active_job.queue_adapter = :delayed_job
```

```ruby
# ใช้งาน Delayed::Job
class ReportJob < ApplicationJob
  def perform(user_id, month)
    user = User.find(user_id)
    report = Report.generate(user, month)
    ReportMailer.monthly_report(user, report).deliver_now
  end
end

# Enqueue
ReportJob.perform_later(user.id, Date.current.beginning_of_month)

# ดู queue
Delayed::Job.all.map { |job| job.handler }

# Process jobs
bundle exec rake jobs:work
bundle exec rake jobs:clear  # ล้าง queue
```

## ขั้นตอนที่ 1069: Testing Jobs

```ruby
# spec/jobs/send_welcome_email_job_spec.rb
require 'rails_helper'

RSpec.describe SendWelcomeEmailJob, type: :job do
  include ActiveJob::TestHelper
  
  let(:user) { create(:user) }
  
  describe "#perform" do
    it "sends welcome email" do
      expect {
        described_class.perform_now(user.id)
      }.to change { ActionMailer::Base.deliveries.count }.by(1)
    end
    
    it "sends to user email" do
      described_class.perform_now(user.id)
      expect(ActionMailer::Base.deliveries.last.to).to include(user.email)
    end
    
    it "raises error for non-existent user" do
      expect {
        described_class.perform_now(0)
      }.to raise_error(ActiveRecord::RecordNotFound)
    end
  end
  
  describe "enqueuing" do
    it "enqueues job" do
      expect {
        described_class.perform_later(user.id)
      }.to have_enqueued_job(described_class).with(user.id)
    end
    
    it "uses mailers queue" do
      expect {
        described_class.perform_later(user.id)
      }.to have_enqueued_job.on_queue('mailers')
    end
    
    it "scheduled for future" do
      expect {
        described_class.set(wait: 5.minutes).perform_later(user.id)
      }.to have_enqueued_job(described_class)
        .at(5.minutes.from_now)
    end
  end
  
  describe "retry behavior" do
    it "retries on failure" do
      allow(UserMailer).to receive(:welcome).and_raise(Net::SMTPError)
      
      perform_enqueued_jobs do
        expect {
          described_class.perform_later(user.id)
        }.to have_enqueued_job(described_class).at_least(:once)
      end
    end
  end
end
```

## ขั้นตอนที่ 1070: Job Monitoring และ Metrics

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.server_middleware do |chain|
    chain.add JobMonitoringMiddleware
  end
end

# lib/job_monitoring_middleware.rb
class JobMonitoringMiddleware
  def call(worker, job, queue)
    start = Time.current
    
    begin
      yield
      
      # Success metrics
      StatsD.increment("sidekiq.#{job['class']}.success")
      StatsD.timing("sidekiq.#{job['class']}.duration", 
                    (Time.current - start) * 1000)
    rescue => e
      # Failure metrics
      StatsD.increment("sidekiq.#{job['class']}.failure")
      raise e
    end
  end
end
```

```ruby
# Job health check
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  def check
    health = {
      database: database_healthy?,
      redis: redis_healthy?,
      sidekiq: sidekiq_healthy?,
      queue_size: Sidekiq::Queue.new.size,
      failed_jobs: Sidekiq::DeadSet.new.size
    }
    
    status = health.values.all? ? :ok : :service_unavailable
    render json: health, status: status
  end
  
  private
  
  def sidekiq_healthy?
    Sidekiq.redis(&:ping) == "PONG"
  rescue
    false
  end
end
```

## ขั้นตอนที่ 1071: Rate Limiting Jobs

```ruby
# ควบคุม rate ของ jobs
class SendEmailJob < ApplicationJob
  queue_as :mailers
  
  # ส่งได้สูงสุด 100 emails ต่อนาที
  throttle threshold: 100, period: 1.minute
  
  def perform(user_id, email_type)
    user = User.find(user_id)
    UserMailer.send(email_type, user).deliver_now
  end
end
```

```ruby
# custom rate limiting ด้วย Redis
class RateLimitedJob < ApplicationJob
  def perform(resource_id)
    rate_limit("process_#{resource_id}", 10, 1.minute) do
      # ทำงาน
      process_resource(resource_id)
    end
  end
  
  private
  
  def rate_limit(key, max_calls, period)
    redis_key = "rate_limit:#{key}"
    
    current = $redis.get(redis_key).to_i
    
    if current < max_calls
      $redis.multi do
        $redis.incr(redis_key)
        $redis.expire(redis_key, period.to_i) if current == 0
      end
      yield
    else
      # Reschedule
      self.class.set(wait: period).perform_later(resource_id)
    end
  end
end
```

## ขั้นตอนที่ 1072: Job Chain

```ruby
# ทำงาน jobs ต่อเนื่อง
class ProcessOrderJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    
    # Chain ไป job ถัดไปหลังจาก success
    order.transaction do
      order.process!
      
      # Enqueue next job
      SendOrderConfirmationJob.perform_later(order.id)
      UpdateInventoryJob.perform_later(order.items.map(&:id))
      NotifyFulfillmentJob.perform_later(order.id)
    end
  end
end
```

## ขั้นตอนที่ 1073: Sidekiq Middleware

```ruby
# Custom middleware
class ContextMiddleware
  def call(worker, job, queue, redis_pool)
    # Set context สำหรับ request tracking
    RequestContext.set(job_id: job['jid'], worker: worker.class.name)
    
    begin
      yield
    ensure
      RequestContext.clear
    end
  end
end

Sidekiq.configure_server do |config|
  config.server_middleware do |chain|
    chain.add ContextMiddleware
  end
end
```

## ขั้นตอนที่ 1074: Production Deployment

```bash
# Procfile (Heroku)
web: bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -C config/sidekiq.yml

# Docker Compose
services:
  web:
    ...
  worker:
    build: .
    command: bundle exec sidekiq -C config/sidekiq.yml
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis
  redis:
    image: redis:7-alpine
```

## ขั้นตอนที่ 1075: Advanced Patterns

```ruby
# Saga pattern สำหรับ distributed transactions
class PlaceOrderSaga
  include Sidekiq::Worker
  
  def perform(order_id)
    order = Order.find(order_id)
    
    begin
      # Step 1: Reserve inventory
      InventoryService.reserve!(order.items)
      
      # Step 2: Process payment
      payment = PaymentService.charge!(order.total, order.payment_method)
      
      # Step 3: Create shipment
      shipment = ShippingService.create!(order)
      
      order.update!(
        status: :confirmed,
        payment_id: payment.id,
        shipment_id: shipment.id
      )
      
      # Confirm all services
      InventoryService.confirm!(order.items)
      
    rescue InventoryError => e
      # Compensating transaction
      order.update!(status: :failed, failure_reason: "Inventory: #{e.message}")
      
    rescue PaymentError => e
      # Release inventory
      InventoryService.release!(order.items)
      order.update!(status: :failed, failure_reason: "Payment: #{e.message}")
      
    rescue ShippingError => e
      # Refund payment
      PaymentService.refund!(payment.id)
      InventoryService.release!(order.items)
      order.update!(status: :failed, failure_reason: "Shipping: #{e.message}")
    end
  end
end
```

## ขั้นตอนที่ 1076-1080: Complex Job Patterns

```ruby
# Fan-out pattern
class SendNewsletterJob < ApplicationJob
  queue_as :mailers
  
  def perform(newsletter_id)
    newsletter = Newsletter.find(newsletter_id)
    user_count = User.subscribed.count
    
    # แบ่งส่ง batch ละ 100 คน
    User.subscribed.find_in_batches(batch_size: 100) do |users|
      SendNewsletterBatchJob.perform_later(
        newsletter.id,
        users.map(&:id)
      )
    end
    
    newsletter.update!(
      status: :sending,
      total_recipients: user_count,
      sending_started_at: Time.current
    )
  end
end

class SendNewsletterBatchJob < ApplicationJob
  queue_as :mailers
  
  def perform(newsletter_id, user_ids)
    newsletter = Newsletter.find(newsletter_id)
    users = User.where(id: user_ids)
    
    sent_count = 0
    
    users.each do |user|
      begin
        NewsletterMailer.send_to(user, newsletter).deliver_now
        sent_count += 1
      rescue => e
        Rails.logger.error "Failed to send to #{user.email}: #{e.message}"
      end
    end
    
    # Update counter (thread-safe)
    Newsletter.where(id: newsletter_id)
              .update_all("sent_count = sent_count + #{sent_count}")
  end
end
```

---

## แบบฝึกหัด: Background Jobs (25 ข้อ)

### ข้อที่ 1: ติดตั้ง Sidekiq
```bash
gem 'sidekiq'
bundle install
# ตั้งค่า config/application.rb และ config/initializers/sidekiq.rb
```

### ข้อที่ 2: สร้าง Welcome Email Job
```ruby
class WelcomeEmailJob < ApplicationJob
  queue_as :mailers
  
  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome(user).deliver_now
  end
end
```

### ข้อที่ 3: Scheduled Cleanup Job
```
สร้าง job ที่รันทุกวันตอนตี 2 เพื่อล้าง expired sessions
```

### ข้อที่ 4: Image Processing Job
```
สร้าง job ที่ process uploaded images ทีหลัง
```

### ข้อที่ 5: Sidekiq Web UI
```
Mount Sidekiq Web UI ที่ /admin/sidekiq ป้องกันเฉพาะ admin
```

### ข้อที่ 6: Retry with Backoff
```
ตั้งค่า job ให้ retry 5 ครั้ง ด้วย exponential backoff
```

### ข้อที่ 7: Job Testing
```
เขียน tests ครอบคลุม:
- Job ทำงาน
- Error handling
- Queue assignment
```

### ข้อที่ 8: Batch Processing
```
สร้าง job ที่ประมวลผล 1000 users เป็น batches ละ 100
```

### ข้อที่ 9: Progress Tracking
```
Track progress ของ import job ด้วย Redis
แสดงความคืบหน้าใน UI
```

### ข้อที่ 10-25 (แบบสรุป)

**ข้อ 10:** Good Job alternative สำหรับ Postgres-only
**ข้อ 11:** Report generation job
**ข้อ 12:** Notification fan-out job
**ข้อ 13:** Data sync job ที่มี uniqueness constraint
**ข้อ 14:** Newsletter sending job (batch)
**ข้อ 15:** Database cleanup job
**ข้อ 16:** External API sync job
**ข้อ 17:** Monitoring job failures
**ข้อ 18:** Job rate limiting
**ข้อ 19:** Job dependencies chain
**ข้อ 20:** Dead letter queue handling
**ข้อ 21:** Job metrics สำหรับ Prometheus
**ข้อ 22:** Priority queues
**ข้อ 23:** Job cancellation
**ข้อ 24:** Idempotent jobs
**ข้อ 25:** Distributed locking

```ruby
# ข้อ 24: Idempotent job
class ProcessPaymentJob < ApplicationJob
  def perform(payment_id)
    payment = Payment.find(payment_id)
    
    # ตรวจสอบว่าทำไปแล้วหรือยัง
    return if payment.processed?
    
    # ทำงาน
    Payment.transaction do
      payment.lock!  # database lock
      return if payment.processed?  # double check after lock
      
      result = PaymentGateway.charge!(payment)
      payment.update!(
        status: :completed,
        gateway_id: result.id,
        processed_at: Time.current
      )
    end
  end
end
```

---

## สรุป: Background Jobs

| Tool | Database | ข้อดี |
|------|----------|-------|
| Sidekiq | Redis | เร็วมาก, mature |
| Good Job | PostgreSQL | ไม่ต้องการ Redis |
| Delayed::Job | ActiveRecord | ง่าย, รองรับทุก DB |
| Resque | Redis | Ruby background jobs |

**Key Takeaways:**
1. ใช้ background jobs สำหรับ tasks ที่ใช้เวลา > 100ms
2. ทำ jobs ให้ idempotent เสมอ
3. Handle errors และ retries อย่างระมัดระวัง
4. Monitor queue sizes และ failure rates
5. ใช้ appropriate queues สำหรับ different priorities
