# Part 52: Email ด้วย Action Mailer

## ขั้นตอนที่ 1141-1160: ส่งอีเมลใน Rails

---

## ขั้นตอนที่ 1141: Action Mailer Setup

```ruby
# สร้าง Mailer
rails generate mailer UserMailer welcome_email
# สร้าง: app/mailers/user_mailer.rb
# สร้าง: app/views/user_mailer/welcome_email.html.erb
# สร้าง: app/views/user_mailer/welcome_email.text.erb
```

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: ENV.fetch('DEFAULT_FROM_EMAIL', 'noreply@myapp.com')
  layout 'mailer'
end
```

```ruby
# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
config.action_mailer.perform_deliveries = true
config.action_mailer.raise_delivery_errors = true

# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.default_url_options = { host: ENV['APP_HOST'] }
config.action_mailer.perform_deliveries = true
config.action_mailer.raise_delivery_errors = true
```

## ขั้นตอนที่ 1142: Basic Mailer

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  # ส่ง welcome email
  def welcome_email(user)
    @user = user
    @login_url = login_url
    @app_name = "MyApp"
    
    mail(
      to: @user.email,
      subject: "ยินดีต้อนรับสู่ #{@app_name}!"
    )
  end
  
  # ส่ง notification email
  def post_published(user, post)
    @user = user
    @post = post
    @post_url = post_url(@post)
    
    mail(
      to: @user.email,
      subject: "โพสต์ \"#{@post.title}\" ได้รับการเผยแพร่แล้ว"
    )
  end
  
  # Email ไปยังหลายคน
  def weekly_digest(user)
    @user = user
    @posts = Post.published.where("created_at > ?", 1.week.ago).order(:created_at)
    
    mail(
      to: @user.email,
      subject: "สรุปข่าวสารประจำสัปดาห์"
    )
  end
end
```

## ขั้นตอนที่ 1143: Email Views

```erb
<%# app/views/user_mailer/welcome_email.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <style>
      body { font-family: Arial, sans-serif; color: #333; }
      .container { max-width: 600px; margin: 0 auto; padding: 20px; }
      .header { background: #007bff; color: white; padding: 20px; text-align: center; }
      .content { padding: 20px; }
      .button { 
        display: inline-block; 
        padding: 12px 24px; 
        background: #007bff; 
        color: white; 
        text-decoration: none;
        border-radius: 4px;
      }
      .footer { color: #999; font-size: 12px; text-align: center; }
    </style>
  </head>
  <body>
    <div class="container">
      <div class="header">
        <h1>ยินดีต้อนรับสู่ <%= @app_name %>!</h1>
      </div>
      
      <div class="content">
        <p>สวัสดีคุณ <%= @user.name %>,</p>
        
        <p>ขอบคุณที่ลงทะเบียนกับเรา! บัญชีของคุณได้รับการสร้างเรียบร้อยแล้ว</p>
        
        <p>คุณสามารถเริ่มใช้งานได้ทันทีโดยคลิกที่ปุ่มด้านล่าง:</p>
        
        <p style="text-align: center;">
          <%= link_to "เริ่มใช้งาน", @login_url, class: "button" %>
        </p>
        
        <p>หากคุณมีข้อสงสัย กรุณาติดต่อเราที่ <a href="mailto:support@myapp.com">support@myapp.com</a></p>
      </div>
      
      <div class="footer">
        <p>© <%= Date.current.year %> <%= @app_name %>. All rights reserved.</p>
        <p>คุณได้รับอีเมลนี้เพราะได้ลงทะเบียนที่ <%= @app_name %></p>
      </div>
    </div>
  </body>
</html>
```

```text
# app/views/user_mailer/welcome_email.text.erb
สวัสดีคุณ <%= @user.name %>,

ยินดีต้อนรับสู่ <%= @app_name %>!

ขอบคุณที่ลงทะเบียนกับเรา! บัญชีของคุณได้รับการสร้างเรียบร้อยแล้ว

เริ่มใช้งานได้ที่: <%= @login_url %>

หากคุณมีข้อสงสัย กรุณาติดต่อเราที่ support@myapp.com

ขอบคุณ,
ทีมงาน <%= @app_name %>
```

## ขั้นตอนที่ 1144: Sending Emails

```ruby
# ส่งทันที (synchronous)
UserMailer.welcome_email(@user).deliver_now

# ส่งใน background (asynchronous - แนะนำ)
UserMailer.welcome_email(@user).deliver_later

# ส่งในเวลาที่กำหนด
UserMailer.weekly_digest(@user).deliver_later(wait: 1.hour)
UserMailer.post_published(@user, @post).deliver_later(wait_until: Friday.noon)
```

```ruby
# ใน controller
class RegistrationsController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      UserMailer.welcome_email(@user).deliver_later
      redirect_to root_path, notice: "ลงทะเบียนสำเร็จ! กรุณาตรวจสอบอีเมลของคุณ"
    else
      render :new
    end
  end
end
```

## ขั้นตอนที่ 1145: Email Attachments

```ruby
# app/mailers/invoice_mailer.rb
class InvoiceMailer < ApplicationMailer
  def send_invoice(user, invoice)
    @user = user
    @invoice = invoice
    
    # แนบไฟล์ PDF
    pdf = InvoicePdfService.generate(@invoice)
    attachments["invoice_#{@invoice.number}.pdf"] = {
      mime_type: 'application/pdf',
      content: pdf
    }
    
    # แนบรูปภาพ
    attachments.inline['logo.png'] = File.read(Rails.root.join('app/assets/images/logo.png'))
    
    mail(
      to: @user.email,
      subject: "ใบแจ้งหนี้ ##{@invoice.number}"
    )
  end
end
```

```erb
<%# ใช้ inline image %>
<%= image_tag attachments['logo.png'].url, alt: "Logo" %>
```

## ขั้นตอนที่ 1146: Email Layout

```erb
<%# app/views/layouts/mailer.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><%= yield :title %></title>
    <style>
      /* Reset styles */
      * { margin: 0; padding: 0; box-sizing: border-box; }
      body { font-family: -apple-system, Arial, sans-serif; background: #f5f5f5; }
      
      /* Container */
      .email-wrapper { padding: 20px; }
      .email-container {
        max-width: 600px;
        margin: 0 auto;
        background: white;
        border-radius: 8px;
        overflow: hidden;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
      }
      
      /* Header */
      .email-header {
        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        color: white;
        padding: 30px 40px;
        text-align: center;
      }
      .email-header h1 { font-size: 24px; }
      
      /* Body */
      .email-body { padding: 40px; }
      .email-body p { line-height: 1.6; margin-bottom: 16px; }
      
      /* Button */
      .btn {
        display: inline-block;
        padding: 14px 28px;
        background: #667eea;
        color: white !important;
        text-decoration: none;
        border-radius: 6px;
        font-weight: bold;
      }
      
      /* Footer */
      .email-footer {
        background: #f9f9f9;
        padding: 20px 40px;
        text-align: center;
        color: #999;
        font-size: 13px;
        border-top: 1px solid #eee;
      }
    </style>
  </head>
  <body>
    <div class="email-wrapper">
      <div class="email-container">
        <div class="email-header">
          <h1>MyApp</h1>
        </div>
        <div class="email-body">
          <%= yield %>
        </div>
        <div class="email-footer">
          <p>© <%= Date.current.year %> MyApp. All rights reserved.</p>
          <p><%== t('mailer.unsubscribe_html', url: unsubscribe_url) %></p>
        </div>
      </div>
    </div>
  </body>
</html>
```

## ขั้นตอนที่ 1147: Letter Opener (Development)

```ruby
# Gemfile
group :development do
  gem 'letter_opener'
  gem 'letter_opener_web', '~> 2.0'  # Web UI
end
```

```ruby
# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.perform_deliveries = true

# หรือ letter_opener_web
# config.action_mailer.delivery_method = :letter_opener_web
```

```ruby
# routes.rb (letter_opener_web)
if Rails.env.development?
  mount LetterOpenerWeb::Engine, at: "/letter_opener"
end
```

## ขั้นตอนที่ 1148: SMTP Configuration

```ruby
# config/environments/production.rb

# Gmail
config.action_mailer.smtp_settings = {
  address: 'smtp.gmail.com',
  port: 587,
  user_name: ENV['GMAIL_USERNAME'],
  password: ENV['GMAIL_PASSWORD'],
  authentication: :plain,
  enable_starttls_auto: true
}

# SendGrid
config.action_mailer.smtp_settings = {
  address: 'smtp.sendgrid.net',
  port: 587,
  user_name: 'apikey',
  password: ENV['SENDGRID_API_KEY'],
  authentication: :plain,
  enable_starttls_auto: true
}

# Mailgun
config.action_mailer.smtp_settings = {
  address: 'smtp.mailgun.org',
  port: 587,
  user_name: ENV['MAILGUN_SMTP_LOGIN'],
  password: ENV['MAILGUN_SMTP_PASSWORD'],
  authentication: :plain,
  enable_starttls_auto: true
}
```

## ขั้นตอนที่ 1149: SendGrid API

```ruby
# Gemfile
gem 'sendgrid-actionmailer'

# config/environments/production.rb
config.action_mailer.delivery_method = :sendgrid_actionmailer
config.action_mailer.sendgrid_actionmailer_settings = {
  api_key: ENV['SENDGRID_API_KEY'],
  raise_delivery_errors: true
}
```

## ขั้นตอนที่ 1150: Email Previews

```ruby
# test/mailers/previews/user_mailer_preview.rb
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    user = User.first || User.new(name: "สมชาย", email: "test@example.com")
    UserMailer.welcome_email(user)
  end
  
  def post_published
    user = User.first
    post = Post.first
    UserMailer.post_published(user, post)
  end
  
  def weekly_digest
    user = User.first
    UserMailer.weekly_digest(user)
  end
end

# เข้าดูได้ที่: http://localhost:3000/rails/mailers
```

## ขั้นตอนที่ 1151: Email Testing

```ruby
# spec/mailers/user_mailer_spec.rb
RSpec.describe UserMailer do
  describe "#welcome_email" do
    let(:user) { create(:user, name: "สมชาย", email: "test@example.com") }
    let(:mail) { described_class.welcome_email(user) }
    
    it "renders the headers" do
      expect(mail.subject).to eq("ยินดีต้อนรับสู่ MyApp!")
      expect(mail.to).to eq(["test@example.com"])
      expect(mail.from).to eq(["noreply@myapp.com"])
    end
    
    it "renders the body" do
      expect(mail.body.encoded).to include("สมชาย")
      expect(mail.body.encoded).to include("ยินดีต้อนรับ")
    end
    
    it "includes login link" do
      expect(mail.body.encoded).to include(login_url)
    end
  end
end
```

```ruby
# spec/models/user_spec.rb - test email delivery
RSpec.describe User do
  describe "after create" do
    it "sends welcome email" do
      expect {
        create(:user)
      }.to have_enqueued_mail(UserMailer, :welcome_email)
    end
  end
end
```

## ขั้นตอนที่ 1152: Notification Preferences

```ruby
# migration
add_column :users, :email_preferences, :jsonb, default: {}

# model
class User < ApplicationRecord
  NOTIFICATION_TYPES = %w[
    welcome_email
    post_published
    comment_received
    weekly_digest
    promotional
  ].freeze
  
  def notify?(type)
    email_preferences.fetch(type.to_s, true)
  end
  
  def update_notification_preference(type, enabled)
    update(email_preferences: email_preferences.merge(type.to_s => enabled))
  end
end
```

```ruby
# ตรวจสอบก่อนส่ง
def send_notification(user, type, *args)
  return unless user.notify?(type)
  
  mailer_method = "#{type}_email"
  UserMailer.public_send(mailer_method, user, *args).deliver_later
end
```

## ขั้นตอนที่ 1153: Unsubscribe Links

```ruby
# Generate unique token
class User < ApplicationRecord
  def unsubscribe_token
    Rails.application.message_verifier(:unsubscribe).generate(id, expires_in: 30.days)
  end
end

# routes.rb
get '/unsubscribe/:token', to: 'subscriptions#destroy', as: :unsubscribe

# ใน mailer
def weekly_digest(user)
  @user = user
  @unsubscribe_url = unsubscribe_url(user.unsubscribe_token)
  mail(to: user.email, subject: "สรุปข่าวสาร")
end
```

---

## แบบฝึกหัด: Email (20 ข้อ)

### ข้อที่ 1: Generate Mailer
```bash
rails generate mailer OrderMailer order_confirmation order_shipped
```

### ข้อที่ 2: Welcome Email
```
เขียน welcome_email mailer พร้อม HTML และ text views
```

### ข้อที่ 3: Setup Letter Opener
```ruby
gem 'letter_opener'
config.action_mailer.delivery_method = :letter_opener
```

### ข้อที่ 4: Email Attachments
```
ส่ง PDF invoice แนบไปกับ email
```

### ข้อที่ 5: deliver_later
```
ส่ง email ใน background queue
```

### ข้อที่ 6-20 (แบบสรุป)

**ข้อ 6:** Email layout ที่สวยงาม
**ข้อ 7:** SendGrid SMTP setup
**ข้อ 8:** Email preview
**ข้อ 9:** Test email content ด้วย RSpec
**ข้อ 10:** Email preferences (opt-in/opt-out)
**ข้อ 11:** Unsubscribe link พร้อม token
**ข้อ 12:** Scheduled weekly digest
**ข้อ 13:** Multi-language emails
**ข้อ 14:** Email tracking (open/click)
**ข้อ 15:** Bulk email ด้วย background jobs
**ข้อ 16:** Email templates ใน database
**ข้อ 17:** CC/BCC
**ข้อ 18:** Reply-to header
**ข้อ 19:** Email queue monitoring
**ข้อ 20:** Resend failed emails

---

## สรุป: Action Mailer

| Feature | Method |
|---------|--------|
| สร้าง mailer | `rails g mailer` |
| ส่งทันที | `.deliver_now` |
| ส่ง background | `.deliver_later` |
| ดู dev email | Letter Opener |
| Preview | `/rails/mailers` |
| แนบไฟล์ | `attachments[]` |

**Key Takeaways:**
1. ใช้ `deliver_later` เสมอเพื่อ UX ที่ดี
2. มีทั้ง HTML และ text version
3. ทดสอบด้วย Letter Opener ใน development
4. ให้ unsubscribe link ในทุก email
5. Track delivery rates ใน production

---

## Step 1153: Advanced Action Mailer

### Email Interceptors

```ruby
# app/mailers/development_mail_interceptor.rb
class DevelopmentMailInterceptor
  def self.delivering_email(message)
    message.subject = "[TEST] #{message.subject}"
    message.to = [ENV.fetch('TEST_EMAIL', 'dev@example.com')]
    message.cc = nil
    message.bcc = nil
  end
end

# config/initializers/mail.rb
if Rails.env.development?
  ActionMailer::Base.register_interceptor(DevelopmentMailInterceptor)
end
```

### Email Observers

```ruby
# app/mailers/email_delivery_observer.rb
class EmailDeliveryObserver
  def self.delivered_email(message)
    EmailLog.create!(
      to: message.to.join(', '),
      from: message.from.join(', '),
      subject: message.subject,
      delivered_at: Time.current,
      message_id: message.message_id
    )
  end
end

# config/initializers/mail.rb
ActionMailer::Base.register_observer(EmailDeliveryObserver)
```

---

## Step 1154: Email Templates with MJML

```bash
# ติดตั้ง MJML (email templating language)
npm install -g mjml
```

```erb
<%# app/views/user_mailer/welcome_email.mjml %>
<mjml>
  <mj-head>
    <mj-title>ยินดีต้อนรับ</mj-title>
    <mj-attributes>
      <mj-all font-family="Arial, sans-serif" />
      <mj-text font-size="16px" line-height="24px" />
    </mj-attributes>
  </mj-head>
  <mj-body>
    <mj-section background-color="#4a90e2">
      <mj-column>
        <mj-text color="#ffffff" font-size="24px" font-weight="bold" align="center">
          ยินดีต้อนรับสู่ MyApp!
        </mj-text>
      </mj-column>
    </mj-section>
    
    <mj-section>
      <mj-column>
        <mj-text>
          สวัสดี <%= @user.name %>,
        </mj-text>
        <mj-text>
          ขอบคุณที่ลงทะเบียนกับเรา
        </mj-text>
        <mj-button background-color="#4a90e2" href="<%= @confirmation_url %>">
          ยืนยันอีเมล
        </mj-button>
      </mj-column>
    </mj-section>
  </mj-body>
</mjml>
```

---

## Step 1155: Email Queue Monitoring

```ruby
# app/jobs/email_stats_job.rb
class EmailStatsJob < ApplicationJob
  queue_as :default
  
  def perform
    stats = {
      sent_today: EmailLog.where("delivered_at > ?", Time.current.beginning_of_day).count,
      failed_today: FailedEmail.where("failed_at > ?", Time.current.beginning_of_day).count,
      queue_size: Sidekiq::Queue.new('mailers').size
    }
    
    if stats[:failed_today] > 100
      AdminMailer.email_failure_alert(stats).deliver_now
    end
    
    StatsD.gauge('email.queue_size', stats[:queue_size])
    StatsD.count('email.sent_today', stats[:sent_today])
  end
end
```

---

## Step 1156: Bounce Handling

```ruby
# app/controllers/webhooks/mailgun_controller.rb
class Webhooks::MailgunController < ApplicationController
  skip_before_action :verify_authenticity_token
  before_action :verify_mailgun_signature
  
  def bounce
    email = params.dig('event-data', 'recipient')
    severity = params.dig('event-data', 'severity')  # 'permanent' or 'temporary'
    
    if severity == 'permanent'
      user = User.find_by(email: email)
      user&.update!(
        email_bounced: true,
        email_bounce_at: Time.current,
        email_bounce_reason: params.dig('event-data', 'delivery-status', 'message')
      )
      Rails.logger.warn "Permanent bounce for #{email}"
    end
    
    head :ok
  end
  
  def unsubscribe
    email = params.dig('event-data', 'recipient')
    User.find_by(email: email)&.update!(
      email_unsubscribed: true,
      email_unsubscribed_at: Time.current
    )
    head :ok
  end
  
  private
  
  def verify_mailgun_signature
    # Verify webhook signature for security
    api_key = ENV['MAILGUN_API_KEY']
    token = params[:token]
    timestamp = params[:timestamp]
    signature = params[:signature]
    
    expected = OpenSSL::HMAC.hexdigest(
      OpenSSL::Digest::SHA256.new,
      api_key,
      "#{timestamp}#{token}"
    )
    
    head :forbidden unless ActiveSupport::SecurityUtils.secure_compare(expected, signature)
  end
end
```

---

## Step 1157: Scheduled Emails

```ruby
# app/jobs/send_birthday_emails_job.rb
class SendBirthdayEmailsJob < ApplicationJob
  queue_as :mailers
  
  def perform
    today = Date.today
    
    User.where(
      "EXTRACT(MONTH FROM birthday) = ? AND EXTRACT(DAY FROM birthday) = ?",
      today.month, today.day
    ).find_each do |user|
      UserMailer.with(user: user).birthday_email.deliver_later
    end
  end
end

# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  def birthday_email
    @user = params[:user]
    @discount_code = generate_discount_code(@user)
    
    mail(
      to: @user.email,
      subject: "สุขสันต์วันเกิด #{@user.name}! 🎂 ของขวัญพิเศษรอคุณอยู่"
    )
  end
  
  private
  
  def generate_discount_code(user)
    code = "BDAY#{user.id}#{Date.today.year}"
    Discount.create_or_find_by!(
      code: code,
      user: user,
      discount_percent: 20,
      expires_at: 1.week.from_now
    )
    code
  end
end
```

---

## Step 1158: Email Unsubscribe System

```ruby
# app/models/concerns/unsubscribable.rb
module Unsubscribable
  extend ActiveSupport::Concern
  
  included do
    has_many :email_preferences, dependent: :destroy
    
    EMAIL_TYPES = %w[
      marketing newsletters product_updates
      security_alerts account_activity weekly_digest
    ].freeze
  end
  
  def unsubscribed_from?(email_type)
    email_preferences.exists?(email_type: email_type, subscribed: false)
  end
  
  def subscribe_to!(email_type)
    pref = email_preferences.find_or_initialize_by(email_type: email_type)
    pref.update!(subscribed: true)
  end
  
  def unsubscribe_from!(email_type)
    pref = email_preferences.find_or_initialize_by(email_type: email_type)
    pref.update!(subscribed: false)
  end
  
  def unsubscribe_all!
    EMAIL_TYPES.each { |type| unsubscribe_from!(type) }
    update!(global_unsubscribe: true)
  end
  
  def unsubscribe_token
    verifier = ActiveSupport::MessageVerifier.new(Rails.application.secret_key_base)
    verifier.generate({ user_id: id, created_at: Time.current.to_i })
  end
  
  def self.find_by_unsubscribe_token(token)
    verifier = ActiveSupport::MessageVerifier.new(Rails.application.secret_key_base)
    payload = verifier.verify(token)
    find(payload[:user_id])
  rescue ActiveSupport::MessageVerifier::InvalidSignature
    nil
  end
end
```

```ruby
# app/controllers/unsubscribes_controller.rb
class UnsubscribesController < ApplicationController
  skip_before_action :authenticate_user!
  
  def show
    @user = User.find_by_unsubscribe_token(params[:token])
    render :invalid_token unless @user
  end
  
  def update
    @user = User.find_by_unsubscribe_token(params[:token])
    return render :invalid_token unless @user
    
    if params[:email_type] == 'all'
      @user.unsubscribe_all!
      flash[:notice] = "ยกเลิกการรับอีเมลทั้งหมดแล้ว"
    else
      @user.unsubscribe_from!(params[:email_type])
      flash[:notice] = "ยกเลิกการรับอีเมลประเภทนี้แล้ว"
    end
    
    redirect_to unsubscribe_confirmation_path
  end
end
```

---

## Step 1159: Email Analytics

```ruby
# app/models/email_log.rb
class EmailLog < ApplicationRecord
  belongs_to :user, optional: true
  
  enum status: { sent: 0, delivered: 1, opened: 2, clicked: 3, bounced: 4, unsubscribed: 5 }
  
  scope :by_type, ->(type) { where(email_type: type) }
  scope :in_period, ->(start, finish) { where(created_at: start..finish) }
  
  def self.stats_for_period(start_date, end_date)
    emails = in_period(start_date, end_date)
    total = emails.count
    
    return {} if total.zero?
    
    {
      total_sent: total,
      delivered: emails.delivered.count,
      opened: emails.opened.count,
      clicked: emails.clicked.count,
      bounced: emails.bounced.count,
      delivery_rate: (emails.delivered.count.to_f / total * 100).round(2),
      open_rate: (emails.opened.count.to_f / total * 100).round(2),
      click_rate: (emails.clicked.count.to_f / total * 100).round(2),
      bounce_rate: (emails.bounced.count.to_f / total * 100).round(2)
    }
  end
end
```

---

## Step 1160: Transactional vs Marketing Emails

```ruby
# app/mailers/transactional_mailer.rb
# Transactional emails - ส่งทุกครั้ง ไม่สนใจ unsubscribe preference
class TransactionalMailer < ApplicationMailer
  def password_reset(user)
    @user = user
    mail(to: user.email, subject: "Reset รหัสผ่าน")
  end
  
  def order_confirmation(order)
    @order = order
    mail(to: order.user.email, subject: "ยืนยันคำสั่งซื้อ ##{order.id}")
  end
  
  def security_alert(user, details)
    @user = user
    @details = details
    mail(to: user.email, subject: "การแจ้งเตือนความปลอดภัย")
  end
end

# app/mailers/marketing_mailer.rb
# Marketing emails - ตรวจสอบ preferences ก่อนส่ง
class MarketingMailer < ApplicationMailer
  before_action :check_marketing_preferences!
  
  def weekly_newsletter(user)
    @user = user
    @articles = Article.trending.limit(5)
    mail(to: user.email, subject: "Newsletter ประจำสัปดาห์")
  end
  
  def promotional_offer(user, offer)
    @user = user
    @offer = offer
    mail(to: user.email, subject: "โปรโมชั่นพิเศษสำหรับคุณ!")
  end
  
  private
  
  def check_marketing_preferences!
    user = params[:user]
    if user.unsubscribed_from?('marketing') || user.global_unsubscribe?
      Rails.logger.info "Skipping marketing email to #{user.email} (unsubscribed)"
      throw :abort
    end
  end
end
```
