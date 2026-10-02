# ตอนที่ 48: Background Jobs กับ Sidekiq (Steps 1056-1080)

## บทนำ

Background Jobs คือ tasks ที่ทำงานในพื้นหลัง โดยไม่บล็อก HTTP request เช่น การส่ง email, การ process ไฟล์ขนาดใหญ่, การ sync ข้อมูล หรือการส่ง notifications

Sidekiq เป็น background job processor ที่นิยมมากที่สุดใน Rails ecosystem ใช้ Redis เป็น message queue และสามารถประมวลผล jobs พร้อมกันหลาย threads

---

## Step 1056: ทำไมต้องใช้ Background Jobs?

### ปัญหาที่ Background Jobs แก้ได้

```
HTTP Request → Controller → View → Response (ต้องเสร็จภายใน ~30 วินาที)

ถ้า task ใช้เวลานาน:
HTTP Request → Controller → [task ที่ใช้เวลา 2 นาที] → Timeout! 😱

ใช้ Background Job:
HTTP Request → Controller → Queue Job → Response ✓
                                ↓
                          [Job ทำงานใน background] → Done ✓
```

### Use Cases ทั่วไป

1. **Email sending** - ส่ง email หลังจาก user register
2. **Image processing** - resize, crop, convert รูปภาพ
3. **Report generation** - สร้าง PDF หรือ CSV ขนาดใหญ่
4. **Third-party API calls** - sync ข้อมูลกับ external services
5. **Notifications** - ส่ง push notifications, SMS
6. **Data import/export** - import CSV หรือ database migrations
7. **Cache warming** - pre-build complex queries
8. **Cleanup tasks** - ลบข้อมูลเก่า

---

## Step 1057: Active Job Abstraction

### Active Job คืออะไร

Active Job เป็น framework ที่ Rails ใช้เป็น abstraction layer สำหรับ background job systems ทำให้สามารถสลับระหว่าง Sidekiq, Delayed Job, Resque ได้โดยไม่ต้องเปลี่ยน code มาก

```
Application Code → Active Job → [Sidekiq / Resque / Delayed Job / GoodJob]
```

### Built-in Adapters

```ruby
# config/application.rb
config.active_job.queue_adapter = :sidekiq        # Production recommended
config.active_job.queue_adapter = :delayed_job    # ใช้ database
config.active_job.queue_adapter = :resque         # Redis-based
config.active_job.queue_adapter = :good_job       # PostgreSQL-based
config.active_job.queue_adapter = :inline         # Execute immediately (test)
config.active_job.queue_adapter = :test           # For testing
```

---

## Step 1058: Sidekiq Setup

### Installation

```ruby
# Gemfile
gem 'sidekiq', '~> 7.0'
gem 'redis', '~> 5.0'

# Optional
gem 'sidekiq-scheduler'     # cron jobs
gem 'sidekiq-unique-jobs'   # prevent duplicate jobs
gem 'sidekiq-failures'      # track failed jobs
```

```bash
bundle install
```

### Redis Installation

```bash
# macOS
brew install redis
brew services start redis

# Ubuntu/Debian
sudo apt-get install redis-server
sudo systemctl start redis

# Docker
docker run -d -p 6379:6379 redis:alpine
```

### Configuration

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch('REDIS_URL') { 'redis://localhost:6379/0' } }
  
  # Optional: configure queues
  config.queues = %w[critical default low mailers]
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch('REDIS_URL') { 'redis://localhost:6379/0' } }
end
```

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :sidekiq
  end
end
```

```yaml
# config/sidekiq.yml
:concurrency: 5
:queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]
  - [mailers, 2]
:max_retries: 3
:timeout: 25
```

### Procfile สำหรับ Development

```yaml
# Procfile.dev
web: bin/rails server -p 3000
worker: bundle exec sidekiq -C config/sidekiq.yml
redis: redis-server
```

```bash
# ติดตั้ง foreman
gem install foreman

# รัน
foreman start -f Procfile.dev
```

---

## Step 1059: Creating Jobs

### Rails Generator

```bash
# Generate job
rails generate job SendWelcomeEmail
rails generate job ProcessImage
rails generate job GenerateReport
rails generate job CleanupOldData
```

### Job File Structure

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Retry จาก error สูงสุด 3 ครั้ง
  retry_on StandardError, wait: :exponentially_longer, attempts: 3
  
  # ไม่ retry สำหรับ errors บางประเภท
  discard_on ActiveJob::DeserializationError
  
  before_perform :log_job_start
  after_perform :log_job_end
  
  private
  
  def log_job_start
    Rails.logger.info "[Job] Starting #{self.class.name} with args: #{arguments.inspect}"
  end
  
  def log_job_end
    Rails.logger.info "[Job] Completed #{self.class.name}"
  end
end
```

### Send Welcome Email Job

```ruby
# app/jobs/send_welcome_email_job.rb
class SendWelcomeEmailJob < ApplicationJob
  queue_as :mailers
  
  # Retry 5 ครั้ง โดย wait เพิ่มขึ้นเรื่อยๆ
  retry_on Net::SMTPServerBusy, wait: 5.minutes, attempts: 5
  retry_on Net::SMTPFatalError, attempts: 3
  
  # ไม่ retry ถ้า user ถูกลบไปแล้ว
  discard_on ActiveRecord::RecordNotFound do |job, error|
    Rails.logger.warn "User #{job.arguments.first} not found, discarding SendWelcomeEmailJob"
  end
  
  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome_email(user).deliver_now
    
    # Update ว่าส่ง email แล้ว
    user.update!(welcome_email_sent_at: Time.current)
    
    Rails.logger.info "Welcome email sent to #{user.email}"
  end
end
```

### Process Image Job

```ruby
# app/jobs/process_image_job.rb
class ProcessImageJob < ApplicationJob
  queue_as :default
  
  retry_on ImageProcessingError, attempts: 3, wait: 1.minute
  
  def perform(attachment_id)
    attachment = ActiveStorage::Attachment.find(attachment_id)
    
    # สร้าง variants
    attachment.blob.open do |file|
      image = MiniMagick::Image.open(file.path)
      
      # Resize thumbnail
      image.resize '300x300>'
      thumbnail_path = Rails.root.join('tmp', "thumbnail_#{attachment_id}.jpg")
      image.write thumbnail_path
      
      # Attach thumbnail
      record = attachment.record
      record.thumbnail.attach(
        io: File.open(thumbnail_path),
        filename: "thumbnail_#{attachment.blob.filename}",
        content_type: 'image/jpeg'
      )
      
      # Cleanup
      File.delete(thumbnail_path) if File.exist?(thumbnail_path)
    end
    
    Rails.logger.info "Processed image for attachment #{attachment_id}"
  end
end
```

### Generate Report Job

```ruby
# app/jobs/generate_report_job.rb
class GenerateReportJob < ApplicationJob
  queue_as :low
  
  def perform(report_id, user_id)
    report = Report.find(report_id)
    user = User.find(user_id)
    
    # Update status
    report.update!(status: 'processing', started_at: Time.current)
    
    begin
      # Generate report data
      data = generate_report_data(report)
      
      # Create CSV
      csv_content = generate_csv(data)
      
      # Attach file
      report.file.attach(
        io: StringIO.new(csv_content),
        filename: "report_#{report.id}_#{Date.today}.csv",
        content_type: 'text/csv'
      )
      
      report.update!(status: 'completed', completed_at: Time.current)
      
      # Notify user
      UserMailer.report_ready(user, report).deliver_later
      
    rescue => e
      report.update!(status: 'failed', error_message: e.message)
      raise e  # Re-raise เพื่อให้ Sidekiq retry
    end
  end
  
  private
  
  def generate_report_data(report)
    case report.type
    when 'sales'
      Order.where(created_at: report.date_range)
           .includes(:user, :items)
           .map { |order| format_order(order) }
    when 'users'
      User.where(created_at: report.date_range)
          .map { |user| format_user(user) }
    end
  end
  
  def generate_csv(data)
    require 'csv'
    CSV.generate(headers: true) do |csv|
      csv << data.first.keys
      data.each { |row| csv << row.values }
    end
  end
end
```

---

## Step 1060: Queues

### Queue Priority

```ruby
# config/sidekiq.yml
:queues:
  - [critical, 3]    # Check 3 times more often
  - [default, 2]     # Check 2 times more often
  - [low, 1]         # Check 1 time
  - [mailers, 2]     # Mail queue

# หมายเหตุ: ตัวเลขหลัง queue name คือ weight (ไม่ใช่ priority)
# Sidekiq จะเลือก queue โดย random weighted
```

### กำหนด Queue ใน Job

```ruby
class CriticalPaymentJob < ApplicationJob
  queue_as :critical  # สำคัญมาก
end

class SendNewsletterJob < ApplicationJob
  queue_as :low  # ไม่เร่งด่วน
end

class SendWelcomeEmailJob < ApplicationJob
  queue_as :mailers
end

# กำหนด queue แบบ dynamic
class ImportDataJob < ApplicationJob
  queue_as do
    # Queue ขึ้นกับ argument
    if arguments.first == 'priority'
      :critical
    else
      :low
    end
  end
end
```

---

## Step 1061: Scheduling ด้วย sidekiq-scheduler

### Setup

```ruby
# Gemfile
gem 'sidekiq-scheduler'
```

```yaml
# config/sidekiq.yml
:schedule:
  cleanup_old_data:
    cron: '0 2 * * *'   # ทุกคืน 2:00 AM
    class: CleanupOldDataJob
    queue: low
    
  generate_daily_report:
    cron: '0 8 * * 1-5'  # จันทร์-ศุกร์ 8:00 AM
    class: GenerateDailyReportJob
    args:
      - daily
    queue: default
    
  check_subscriptions:
    cron: '*/30 * * * *'  # ทุก 30 นาที
    class: CheckSubscriptionsJob
    queue: default
    
  send_reminders:
    cron: '0 9 * * *'    # ทุกวัน 9:00 AM
    class: SendRemindersJob
    queue: mailers
```

```ruby
# app/jobs/cleanup_old_data_job.rb
class CleanupOldDataJob < ApplicationJob
  queue_as :low
  
  def perform
    # ลบข้อมูลที่เก่ากว่า 90 วัน
    count = ActivityLog.where("created_at < ?", 90.days.ago).delete_all
    Rails.logger.info "Deleted #{count} old activity logs"
    
    # ลบ unverified users
    expired_users = User.unverified.where("created_at < ?", 7.days.ago)
    expired_count = expired_users.count
    expired_users.destroy_all
    Rails.logger.info "Deleted #{expired_count} unverified users"
    
    # ลบ temp files
    Dir.glob(Rails.root.join('tmp', 'uploads', '*')).each do |file|
      File.delete(file) if File.mtime(file) < 1.day.ago
    end
  end
end
```

---

## Step 1062: Sidekiq Web UI

### Setup

```ruby
# config/routes.rb
require 'sidekiq/web'

Rails.application.routes.draw do
  # Protect with authentication
  authenticate :user, ->(user) { user.admin? } do
    mount Sidekiq::Web => '/sidekiq'
  end
  
  # หรือใช้ HTTP Basic Auth
  Sidekiq::Web.use(Rack::Auth::Basic) do |username, password|
    ActiveSupport::SecurityUtils.secure_compare(
      ::Digest::SHA256.hexdigest(username),
      ::Digest::SHA256.hexdigest(ENV.fetch("SIDEKIQ_USERNAME", "admin"))
    ) & ActiveSupport::SecurityUtils.secure_compare(
      ::Digest::SHA256.hexdigest(password),
      ::Digest::SHA256.hexdigest(ENV.fetch("SIDEKIQ_PASSWORD", "password"))
    )
  end
  mount Sidekiq::Web => '/sidekiq'
end
```

### Web UI Features

```
/sidekiq → Dashboard แสดง:
- จำนวน processed jobs
- จำนวน failed jobs
- จำนวน jobs ใน queues
- จำนวน busy workers

/sidekiq/queues → Queue list
/sidekiq/retries → Jobs ที่กำลัง retry
/sidekiq/dead → Jobs ที่ failed ทั้งหมด
/sidekiq/scheduled → Scheduled jobs
/sidekiq/busy → Jobs ที่กำลังทำงาน
```

---

## Step 1063: Error Handling และ Retries

### Retry Configuration

```ruby
# app/jobs/send_notification_job.rb
class SendNotificationJob < ApplicationJob
  queue_as :default
  
  # Retry strategies
  
  # 1. Simple retry
  retry_on Net::TimeoutError, attempts: 3
  
  # 2. Retry with wait
  retry_on Faraday::TimeoutError, wait: 30.seconds, attempts: 5
  
  # 3. Exponential backoff
  retry_on ThirdPartyApiError, wait: :exponentially_longer, attempts: 10
  # Wait times: 3s, 18s, 81s, 4m, 18m, 1h, 4h, 17h, 3d, 12d
  
  # 4. Custom wait
  retry_on RateLimitError, wait: ->(executions) { executions * 30 }, attempts: 5
  
  # 5. Discard without retry
  discard_on UserBlockedError
  discard_on ActiveJob::DeserializationError
  
  # 6. Custom discard logic
  discard_on NotificationError do |job, error|
    user_id = job.arguments.first
    Rails.logger.error "Failed to send notification to user #{user_id}: #{error.message}"
    User.find_by(id: user_id)&.update(notifications_enabled: false)
  end
  
  def perform(user_id, message)
    user = User.find(user_id)
    
    raise UserBlockedError if user.blocked?
    
    NotificationService.send(user, message)
  end
end
```

### Dead Job Queue

```ruby
# Jobs ที่ retry ครบแล้วและยังไม่สำเร็จจะไปอยู่ใน "dead" queue
# สามารถ retry manually ผ่าน Web UI หรือ programmatically

# Retry dead jobs programmatically
ds = Sidekiq::DeadSet.new
ds.each do |job|
  if job['class'] == 'SendNotificationJob'
    job.retry  # Retry job
    # หรือ
    job.delete  # ลบออก
  end
end

# ลบ dead jobs เก่ากว่า 30 วัน
Sidekiq::DeadSet.new.each do |job|
  job.delete if Time.parse(job['failed_at']) < 30.days.ago
end
```

---

## Step 1064: Real-World Examples

### User Registration Flow

```ruby
# app/controllers/api/v1/auth_controller.rb
def register
  user = User.new(register_params)
  
  if user.save
    # Queue background jobs
    SendWelcomeEmailJob.perform_later(user.id)
    SetupUserDefaultsJob.perform_later(user.id)
    TrackSignupEventJob.perform_later(user.id, request.remote_ip)
    
    render json: { success: true, user: UserSerializer.new(user).as_json }, 
           status: :created
  else
    render json: { errors: user.errors.full_messages }, 
           status: :unprocessable_entity
  end
end
```

```ruby
# app/jobs/setup_user_defaults_job.rb
class SetupUserDefaultsJob < ApplicationJob
  queue_as :default
  
  def perform(user_id)
    user = User.find(user_id)
    
    # สร้าง default profile
    user.create_profile!(
      bio: "",
      avatar_url: Gravatar.url(user.email),
      preferences: { 
        notifications_email: true,
        notifications_push: false,
        theme: 'light'
      }
    )
    
    # Subscribe to default newsletter
    NewsletterService.subscribe(user.email)
    
    # Create tutorial checklist
    TutorialChecklist.create_for(user)
    
    Rails.logger.info "Setup defaults for user #{user.id}"
  end
end
```

### Order Processing Pipeline

```ruby
# app/jobs/process_order_job.rb
class ProcessOrderJob < ApplicationJob
  queue_as :critical
  
  def perform(order_id)
    order = Order.find(order_id)
    
    ActiveRecord::Base.transaction do
      # Verify inventory
      order.items.each do |item|
        product = item.product.lock!  # Pessimistic lock
        raise InsufficientStockError, product.name if product.stock_quantity < item.quantity
        product.decrement!(:stock_quantity, item.quantity)
      end
      
      # Process payment
      payment_result = PaymentGateway.charge(
        amount: order.total_amount,
        currency: 'THB',
        source: order.payment_method_token,
        description: "Order ##{order.id}"
      )
      
      order.update!(
        status: 'paid',
        payment_id: payment_result[:id],
        paid_at: Time.current
      )
    end
    
    # Queue follow-up jobs
    SendOrderConfirmationJob.perform_later(order.id)
    NotifyWarehouseJob.perform_later(order.id)
    UpdateInventoryReportJob.perform_later
    
  rescue InsufficientStockError => e
    order.update!(status: 'failed', failure_reason: "Out of stock: #{e.message}")
    SendOrderFailureNotificationJob.perform_later(order.id, e.message)
    raise e
    
  rescue PaymentGateway::CardDeclinedError => e
    order.update!(status: 'payment_failed', failure_reason: e.message)
    SendPaymentFailureNotificationJob.perform_later(order.id)
    discard_job  # ไม่ retry สำหรับ card declined
    
  rescue PaymentGateway::NetworkError => e
    order.update!(status: 'pending')
    raise e  # Retry
  end
end
```

---

## Step 1065: Testing กับ RSpec

### Setup

```ruby
# Gemfile (test group)
gem 'rspec-rails'
gem 'shoulda-matchers'
```

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  # ใช้ test adapter สำหรับ ActiveJob
  config.before(:each) do
    ActiveJob::Base.queue_adapter = :test
  end
  
  # Include job matchers
  config.include ActiveJob::TestHelper
end
```

### Job Tests

```ruby
# spec/jobs/send_welcome_email_job_spec.rb
require 'rails_helper'

RSpec.describe SendWelcomeEmailJob, type: :job do
  let(:user) { create(:user) }
  
  describe '#perform' do
    it 'sends welcome email to user' do
      expect {
        SendWelcomeEmailJob.perform_now(user.id)
      }.to change(ActionMailer::Base.deliveries, :count).by(1)
    end
    
    it 'sends email to correct address' do
      SendWelcomeEmailJob.perform_now(user.id)
      email = ActionMailer::Base.deliveries.last
      expect(email.to).to include(user.email)
    end
    
    it 'updates welcome_email_sent_at' do
      expect {
        SendWelcomeEmailJob.perform_now(user.id)
      }.to change { user.reload.welcome_email_sent_at }.from(nil)
    end
    
    context 'when user is not found' do
      it 'discards the job without raising' do
        expect {
          SendWelcomeEmailJob.perform_now(99999)
        }.not_to raise_error
      end
    end
  end
  
  describe 'queue' do
    it 'uses the mailers queue' do
      expect(SendWelcomeEmailJob.new.queue_name).to eq('mailers')
    end
  end
end
```

```ruby
# spec/jobs/process_order_job_spec.rb
require 'rails_helper'

RSpec.describe ProcessOrderJob, type: :job do
  let(:user) { create(:user) }
  let(:product) { create(:product, stock_quantity: 10, price: 100) }
  let(:order) { create(:order, user: user, status: 'pending') }
  
  before do
    create(:order_item, order: order, product: product, quantity: 2)
  end
  
  describe '#perform' do
    context 'with sufficient stock' do
      it 'processes the order' do
        expect {
          ProcessOrderJob.perform_now(order.id)
        }.to change { order.reload.status }.from('pending').to('paid')
      end
      
      it 'reduces stock quantity' do
        expect {
          ProcessOrderJob.perform_now(order.id)
        }.to change { product.reload.stock_quantity }.by(-2)
      end
      
      it 'enqueues confirmation email' do
        expect {
          ProcessOrderJob.perform_now(order.id)
        }.to have_enqueued_job(SendOrderConfirmationJob).with(order.id)
      end
    end
    
    context 'with insufficient stock' do
      before { product.update!(stock_quantity: 1) }
      
      it 'marks order as failed' do
        expect {
          ProcessOrderJob.perform_now(order.id)
        }.to change { order.reload.status }.to('failed')
      end
    end
  end
end
```

### Testing Job Enqueueing

```ruby
# spec/requests/api/v1/auth_spec.rb
require 'rails_helper'

RSpec.describe "Auth", type: :request do
  describe "POST /api/v1/auth/register" do
    let(:user_params) do
      {
        user: {
          name: "Test User",
          email: "test@example.com",
          password: "password123",
          password_confirmation: "password123"
        }
      }
    end
    
    it 'enqueues welcome email job' do
      expect {
        post '/api/v1/auth/register', params: user_params
      }.to have_enqueued_job(SendWelcomeEmailJob)
    end
    
    it 'enqueues setup defaults job' do
      expect {
        post '/api/v1/auth/register', params: user_params
      }.to have_enqueued_job(SetupUserDefaultsJob)
    end
    
    it 'enqueues job with correct user id' do
      post '/api/v1/auth/register', params: user_params
      user_id = User.last.id
      
      expect(SendWelcomeEmailJob).to have_been_enqueued.with(user_id)
    end
  end
end
```

---

## Step 1066: Good Job (PostgreSQL-based alternative)

### เมื่อควรใช้ Good Job แทน Sidekiq

- ไม่มี Redis infrastructure
- ต้องการ ACID transactions
- ต้องการ database-level visibility
- โปรเจกต์ขนาดเล็กถึงกลาง

### Setup Good Job

```ruby
# Gemfile
gem 'good_job', '~> 3.0'
```

```bash
bundle install
rails generate good_job:install
rails db:migrate
```

```ruby
# config/application.rb
config.active_job.queue_adapter = :good_job
```

```ruby
# config/initializers/good_job.rb
GoodJob.configure do |config|
  config.max_threads = 5
  config.queues = 'critical:3;default:2;low:1'
  config.poll_interval = 5  # วินาที
  config.shutdown_timeout = 25
  
  # Cleanup completed jobs
  config.cleanup_preserved_jobs_before_seconds_ago = 1.day.to_i
  config.cleanup_interval_jobs = 1000
end
```

### Good Job Dashboard

```ruby
# config/routes.rb
require 'good_job/engine'

Rails.application.routes.draw do
  authenticate :user, ->(user) { user.admin? } do
    mount GoodJob::Engine => '/good_job'
  end
end
```

---

## Step 1067: Sidekiq Best Practices

### Idempotency

```ruby
# Job ต้องทำงานได้หลายครั้งโดยให้ผลเหมือนกัน
class SendConfirmationEmailJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    
    # ตรวจสอบว่าส่งแล้วหรือยัง (idempotency check)
    return if order.confirmation_email_sent?
    
    UserMailer.order_confirmation(order).deliver_now
    order.update!(confirmation_email_sent: true)
  end
end
```

### Small Payloads

```ruby
# ไม่ดี - ส่ง object ทั้งก้อน
class ProcessUserJob < ApplicationJob
  def perform(user)  # ห้าม! object อาจ stale
    # ...
  end
end

# ดี - ส่ง ID แล้วดึงข้อมูลใหม่ใน job
class ProcessUserJob < ApplicationJob
  def perform(user_id)
    user = User.find(user_id)  # ดึงข้อมูล fresh
    # ...
  end
end
```

### Transaction Safety

```ruby
# ไม่ดี - Queue job ก่อน transaction เสร็จ
def create_order
  order = Order.create!(...)
  ProcessOrderJob.perform_later(order.id)  # อาจ run ก่อน commit!
end

# ดี - Queue หลัง transaction เสร็จ
def create_order
  order = nil
  ActiveRecord::Base.transaction do
    order = Order.create!(...)
  end
  # transaction เสร็จแล้ว ปลอดภัย
  ProcessOrderJob.perform_later(order.id)
end

# หรือใช้ after_commit callback
class Order < ApplicationRecord
  after_commit :queue_processing_job, on: :create
  
  private
  
  def queue_processing_job
    ProcessOrderJob.perform_later(id)
  end
end
```

---

## แบบฝึกหัด (25 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** สร้าง job ที่ส่ง welcome email หลังจาก user register
```ruby
# เฉลย
class SendWelcomeEmailJob < ApplicationJob
  queue_as :mailers
  
  def perform(user_id)
    user = User.find(user_id)
    UserMailer.welcome(user).deliver_now
  end
end

# ใน controller
SendWelcomeEmailJob.perform_later(user.id)
```

**ข้อ 2:** ตั้งค่า Sidekiq ให้ใช้ 3 queues: critical, default, low
```yaml
# เฉลย - config/sidekiq.yml
:concurrency: 5
:queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]
```

**ข้อ 3:** สร้าง job ที่ทำงานซ้ำได้ (idempotent) สำหรับ charge ค่าบริการ monthly
```ruby
# เฉลย
class ChargeMonthlyFeeJob < ApplicationJob
  def perform(subscription_id)
    sub = Subscription.find(subscription_id)
    return if sub.charged_this_month?
    
    PaymentService.charge(sub)
    sub.update!(last_charged_at: Time.current)
  end
end
```

**ข้อ 4:** เพิ่ม retry ใน job ที่ call external API
```ruby
# เฉลย
class SyncWithExternalApiJob < ApplicationJob
  retry_on Faraday::Error, wait: :exponentially_longer, attempts: 5
  discard_on ActiveRecord::RecordNotFound
  
  def perform(record_id)
    record = MyModel.find(record_id)
    ExternalApi.sync(record)
  end
end
```

**ข้อ 5:** Protected Sidekiq Web UI ด้วย admin authentication
```ruby
# เฉลย
authenticate :user, ->(u) { u.admin? } do
  mount Sidekiq::Web => '/sidekiq'
end
```

### ระดับกลาง

**ข้อ 6:** สร้าง scheduled job ที่ส่ง digest email ทุกวันจันทร์ 9:00 AM
```yaml
# เฉลย - config/sidekiq.yml
:schedule:
  weekly_digest:
    cron: '0 9 * * 1'
    class: SendWeeklyDigestJob
    queue: mailers
```

**ข้อ 7:** เขียน job test ที่ตรวจสอบว่า job ถูก enqueue พร้อม correct arguments
```ruby
# เฉลย
it 'enqueues job with user id' do
  user = create(:user)
  expect {
    post '/register', params: { user: user_params }
  }.to have_enqueued_job(SendWelcomeEmailJob).with(User.last.id)
end
```

**ข้อ 8:** สร้าง job pipeline ที่ chain หลาย jobs ด้วยกัน
```ruby
# เฉลย
class OrderProcessingPipelineJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    
    ValidateOrderJob.perform_now(order_id)
    ChargePaymentJob.perform_now(order_id)
    SendConfirmationJob.perform_later(order_id)
    NotifyWarehouseJob.perform_later(order_id)
  end
end
```

**ข้อ 9:** implement job progress tracking ด้วย Redis
```ruby
# เฉลย
class ImportUsersJob < ApplicationJob
  def perform(file_path)
    rows = CSV.read(file_path)
    total = rows.count
    
    rows.each_with_index do |row, index|
      User.create!(email: row[0], name: row[1])
      
      # Update progress
      progress = ((index + 1).to_f / total * 100).round
      Rails.cache.write("import_progress_#{job_id}", progress, expires_in: 1.hour)
    end
  end
end

# ใน controller
def import_progress
  progress = Rails.cache.read("import_progress_#{params[:job_id]}") || 0
  render json: { progress: progress }
end
```

**ข้อ 10:** สร้าง job ที่ batch process records เป็น chunks
```ruby
# เฉลย
class ProcessAllUsersJob < ApplicationJob
  def perform(offset = 0, batch_size = 100)
    users = User.order(:id).offset(offset).limit(batch_size)
    
    return if users.empty?
    
    users.each { |user| process_user(user) }
    
    # Queue next batch
    self.class.perform_later(offset + batch_size, batch_size)
  end
  
  private
  
  def process_user(user)
    # process logic
  end
end
```

### ระดับสูง

**ข้อ 11-25:**

**ข้อ 11:** implement unique jobs ด้วย sidekiq-unique-jobs gem
**ข้อ 12:** สร้าง job ที่ handle distributed locks ป้องกัน race conditions
**ข้อ 13:** เขียน job ที่ track performance metrics (duration, memory)
**ข้อ 14:** สร้าง dead letter queue handler ที่ alert เมื่อมี failures
**ข้อ 15:** implement job middleware สำหรับ logging และ tracing

```ruby
# เฉลย ข้อ 12 - Distributed Lock
class ProcessPaymentJob < ApplicationJob
  def perform(order_id)
    lock_key = "process_payment_#{order_id}"
    
    # Redis lock
    lock_acquired = $redis.set(lock_key, 1, nx: true, ex: 300)
    
    unless lock_acquired
      Rails.logger.warn "Lock exists for order #{order_id}, skipping"
      return
    end
    
    begin
      # Process payment
      process_payment_for_order(order_id)
    ensure
      $redis.del(lock_key)
    end
  end
end
```

```ruby
# เฉลย ข้อ 15 - Job Middleware
class JobLoggingMiddleware
  def call(worker, job, queue, &block)
    start_time = Time.current
    
    Rails.logger.info({
      event: 'job_started',
      job_class: job['class'],
      job_id: job['jid'],
      queue: queue,
      args: job['args']
    }.to_json)
    
    yield
    
    duration = Time.current - start_time
    Rails.logger.info({
      event: 'job_completed',
      job_class: job['class'],
      job_id: job['jid'],
      duration_ms: (duration * 1000).round
    }.to_json)
    
  rescue => e
    Rails.logger.error({
      event: 'job_failed',
      job_class: job['class'],
      error: e.class.name,
      message: e.message
    }.to_json)
    raise
  end
end

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.server_middleware do |chain|
    chain.add JobLoggingMiddleware
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ Background Jobs** - แก้ปัญหา long-running tasks
2. **Active Job** - abstraction layer ของ Rails
3. **Sidekiq Setup** - การติดตั้งและ configure
4. **Creating Jobs** - สร้าง jobs ด้วย generator
5. **Queues** - Priority queues สำหรับ different tasks
6. **Scheduling** - Cron jobs ด้วย sidekiq-scheduler
7. **Error Handling** - retry_on และ discard_on
8. **Web UI** - Monitor jobs ผ่าน browser
9. **Testing** - Test jobs ด้วย RSpec
10. **Good Job** - PostgreSQL-based alternative

Background jobs เป็นส่วนสำคัญของ production Rails apps ทุกตัว ควรใช้เมื่อมี tasks ที่ใช้เวลานาน หรือต้องการ reliability ในการทำงาน
