# ตอนที่ 52: Email กับ Action Mailer (Steps 1141-1160)

## บทนำ

Action Mailer คือ Rails framework สำหรับส่ง email มี API คล้ายกับ controllers มาก รองรับทั้ง HTML และ plain text emails รวมถึง attachments, inline images และ multipart emails

---

## Step 1141: Action Mailer Setup

### Configuration

```ruby
# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener  # เปิด email ใน browser
config.action_mailer.perform_deliveries = true
config.action_mailer.raise_delivery_errors = true

# URL ใน emails
config.action_mailer.default_url_options = { 
  host: 'localhost', 
  port: 3000 
}

# Preview
config.action_mailer.preview_path = "#{Rails.root}/spec/mailers/previews"
```

```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.perform_deliveries = true
config.action_mailer.raise_delivery_errors = false  # ไม่ raise ใน production

config.action_mailer.default_url_options = {
  host: ENV['APP_HOST'] || 'myapp.com',
  protocol: 'https'
}

config.action_mailer.smtp_settings = {
  address: ENV['SMTP_HOST'],
  port: ENV['SMTP_PORT'].to_i,
  domain: ENV['SMTP_DOMAIN'],
  user_name: ENV['SMTP_USERNAME'],
  password: ENV['SMTP_PASSWORD'],
  authentication: :plain,
  enable_starttls_auto: true,
  open_timeout: 5,
  read_timeout: 5
}
```

---

## Step 1142: Generating Mailers

### Rails Generator

```bash
# Generate mailers
rails generate mailer UserMailer
rails generate mailer OrderMailer
rails generate mailer NotificationMailer
```

```
สร้างไฟล์:
app/mailers/user_mailer.rb
app/views/user_mailer/
spec/mailers/user_mailer_spec.rb (ถ้า RSpec)
spec/mailers/previews/user_mailer_preview.rb
```

---

## Step 1143: Mailer Class

### ApplicationMailer

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: ENV.fetch('DEFAULT_FROM_EMAIL') { 'noreply@myapp.com' }
  
  layout 'mailer'  # ใช้ layouts/mailer.html.erb
  
  before_action :set_user_locale
  
  private
  
  def set_user_locale
    # ตั้ง locale ตาม recipient preferences
    I18n.locale = @user&.locale || I18n.default_locale
  end
end
```

### User Mailer

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  before_action :set_user
  
  # Welcome email
  def welcome_email
    @token = @user.generate_confirmation_token!
    @confirmation_url = confirm_email_url(token: @token, host: default_url_options[:host])
    
    mail(
      to: @user.email,
      subject: "ยินดีต้อนรับสู่ MyApp! ยืนยันอีเมลของคุณ"
    )
  end
  
  # Email confirmation
  def email_confirmation
    @token = @user.generate_confirmation_token!
    @confirmation_url = confirm_email_url(token: @token)
    
    mail(to: @user.email, subject: "ยืนยันอีเมลของคุณ")
  end
  
  # Password reset
  def reset_password
    @token = @user.generate_reset_password_token!
    @reset_url = edit_password_reset_url(token: @token)
    @expires_at = 2.hours.from_now
    
    mail(
      to: @user.email,
      subject: "Reset รหัสผ่านของคุณ - MyApp"
    )
  end
  
  # Account locked notification
  def account_locked
    @unlock_url = unlock_account_url(token: @user.unlock_token)
    @locked_at = Time.current
    
    mail(
      to: @user.email,
      subject: "บัญชีของคุณถูกล็อค - MyApp",
      importance: "high"
    )
  end
  
  # Weekly digest
  def weekly_digest
    @articles = @user.recommended_articles.limit(5)
    @stats = @user.weekly_stats
    
    mail(
      to: @user.email,
      subject: "สรุปประจำสัปดาห์ของคุณ - #{Date.today.strftime('%d %b %Y')}"
    )
  end
  
  private
  
  def set_user
    @user = params[:user]
  end
end
```

---

## Step 1144: Mailer Views

### HTML Layout

```erb
<%# app/views/layouts/mailer.html.erb %>
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="x-apple-disable-message-reformatting">
  <title></title>
  
  <style>
    body {
      font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
      font-size: 16px;
      line-height: 1.6;
      color: #333;
      background-color: #f4f4f4;
      margin: 0;
      padding: 0;
    }
    
    .email-container {
      max-width: 600px;
      margin: 0 auto;
      background-color: #ffffff;
      padding: 30px;
    }
    
    .email-header {
      background-color: #4a90e2;
      color: white;
      padding: 20px;
      text-align: center;
      border-radius: 4px 4px 0 0;
    }
    
    .email-body {
      padding: 30px 20px;
    }
    
    .email-footer {
      background-color: #f8f8f8;
      padding: 20px;
      text-align: center;
      font-size: 12px;
      color: #999;
      border-top: 1px solid #ddd;
    }
    
    .btn {
      display: inline-block;
      padding: 12px 24px;
      background-color: #4a90e2;
      color: #ffffff !important;
      text-decoration: none;
      border-radius: 4px;
      font-weight: bold;
    }
    
    .btn-danger {
      background-color: #e74c3c;
    }
    
    a { color: #4a90e2; }
  </style>
</head>
<body>
  <div class="email-container">
    <div class="email-header">
      <%= link_to "MyApp", root_url, style: "color: white; text-decoration: none;" %>
    </div>
    
    <div class="email-body">
      <%= yield %>
    </div>
    
    <div class="email-footer">
      <p>คุณได้รับอีเมลนี้เพราะคุณลงทะเบียนใช้งาน MyApp</p>
      <p>
        <%= link_to "Unsubscribe", unsubscribe_url(token: @user&.unsubscribe_token) %> | 
        <%= link_to "การตั้งค่า", settings_url %>
      </p>
      <p>© <%= Time.current.year %> MyApp. All rights reserved.</p>
    </div>
  </div>
</body>
</html>
```

### HTML Email View

```erb
<%# app/views/user_mailer/welcome_email.html.erb %>
<h1>ยินดีต้อนรับสู่ MyApp, <%= @user.name %>! 🎉</h1>

<p>ขอบคุณที่ลงทะเบียนกับเรา เราตื่นเต้นที่จะมีคุณอยู่ด้วยกัน</p>

<p>กรุณายืนยันอีเมลของคุณโดยคลิกปุ่มด้านล่าง:</p>

<p style="text-align: center; margin: 30px 0;">
  <%= link_to "ยืนยันอีเมล", @confirmation_url, class: "btn" %>
</p>

<p>หรือคัดลอก URL นี้ไปวางในเบราว์เซอร์:</p>
<p><code><%= @confirmation_url %></code></p>

<p><strong>ลิงก์นี้จะหมดอายุใน 48 ชั่วโมง</strong></p>

<hr style="border: none; border-top: 1px solid #eee; margin: 30px 0;">

<p>สิ่งที่คุณสามารถทำได้:</p>
<ul>
  <li>เขียนบทความแรกของคุณ</li>
  <li>ติดตามนักเขียนที่คุณชอบ</li>
  <li>ค้นพบเนื้อหาที่น่าสนใจ</li>
</ul>

<p>หากคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้</p>
```

### Plain Text View

```text
# app/views/user_mailer/welcome_email.text.erb
ยินดีต้อนรับสู่ MyApp, <%= @user.name %>!

ขอบคุณที่ลงทะเบียนกับเรา

กรุณายืนยันอีเมลของคุณที่:
<%= @confirmation_url %>

ลิงก์นี้จะหมดอายุใน 48 ชั่วโมง

หากคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้

ขอบคุณ,
ทีมงาน MyApp
```

### Password Reset Email

```erb
<%# app/views/user_mailer/reset_password.html.erb %>
<h2>Reset รหัสผ่าน</h2>

<p>สวัสดี <%= @user.name %>,</p>

<p>เราได้รับคำขอ reset รหัสผ่านสำหรับบัญชีของคุณ</p>

<p style="text-align: center; margin: 30px 0;">
  <%= link_to "Reset รหัสผ่าน", @reset_url, class: "btn btn-danger" %>
</p>

<p>ลิงก์นี้จะหมดอายุเวลา <%= @expires_at.strftime("%H:%M น. %d %B %Y") %></p>

<div style="background: #fff3cd; padding: 15px; border-radius: 4px; margin: 20px 0;">
  <strong>⚠️ หากคุณไม่ได้ขอ reset รหัสผ่าน</strong>
  <p>กรุณาเพิกเฉยต่ออีเมลนี้ รหัสผ่านของคุณจะไม่มีการเปลี่ยนแปลง</p>
  <p>หากคุณต้องการรายงานกิจกรรมที่น่าสงสัย กรุณาติดต่อ <a href="mailto:security@myapp.com">security@myapp.com</a></p>
</div>
```

---

## Step 1145: ส่ง Mail จาก Controller

### deliver_now vs deliver_later

```ruby
# app/controllers/registrations_controller.rb
def create
  @user = User.new(user_params)
  
  if @user.save
    # deliver_now - ส่งทันทีใน request (ช้าลง ~100-500ms)
    UserMailer.with(user: @user).welcome_email.deliver_now
    
    # deliver_later - ส่ง ใน background job (แนะนำ)
    UserMailer.with(user: @user).welcome_email.deliver_later
    
    # deliver_later พร้อม options
    UserMailer.with(user: @user).weekly_digest.deliver_later(
      wait: 1.hour,               # รอ 1 ชั่วโมง
      priority: 0,                 # priority
      queue: :mailers             # queue name
    )
    
    # deliver_later ตาม timestamp
    UserMailer.with(user: @user).welcome_email.deliver_later(
      wait_until: Time.zone.now.tomorrow.beginning_of_day
    )
    
    redirect_to root_path, notice: "สมัครสมาชิกสำเร็จ!"
  else
    render :new
  end
end
```

---

## Step 1146: Mail Preview ใน Development

### Mailer Preview Class

```ruby
# spec/mailers/previews/user_mailer_preview.rb (หรือ test/mailers/previews/)
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    user = User.first || FactoryBot.build_stubbed(:user)
    UserMailer.with(user: user).welcome_email
  end
  
  def reset_password
    user = User.first
    UserMailer.with(user: user).reset_password
  end
  
  def weekly_digest
    user = User.first
    UserMailer.with(user: user).weekly_digest
  end
  
  def order_confirmation
    order = Order.includes(:items, :user).last
    OrderMailer.with(order: order).confirmation
  end
end
```

```
เข้าถึงได้ที่: http://localhost:3000/rails/mailers/user_mailer/welcome_email
```

---

## Step 1147: Letter Opener Gem

### Setup

```ruby
# Gemfile
group :development do
  gem 'letter_opener'
  gem 'letter_opener_web'  # web interface
end
```

```ruby
# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.perform_deliveries = true
```

```ruby
# config/routes.rb (optional web interface)
if Rails.env.development?
  mount LetterOpenerWeb::Engine, at: "/letter_opener"
end
```

---

## Step 1148: Attachments

### File Attachments

```ruby
# app/mailers/order_mailer.rb
class OrderMailer < ApplicationMailer
  def confirmation_with_invoice(order)
    @order = order
    
    # Attach PDF invoice
    pdf_content = InvoicePdf.new(order).render
    attachments["invoice_#{order.id}.pdf"] = {
      mime_type: 'application/pdf',
      content: pdf_content
    }
    
    # Attach terms of service
    terms_file = Rails.root.join('public', 'terms.pdf')
    attachments['terms.pdf'] = File.read(terms_file)
    
    mail(
      to: order.user.email,
      subject: "ยืนยันคำสั่งซื้อ ##{order.id}"
    )
  end
  
  # Inline attachment (รูปใน email body)
  def promotional_email(user)
    @user = user
    
    # Inline image
    attachments.inline['banner.jpg'] = File.read(Rails.root.join('app', 'assets', 'images', 'email-banner.jpg'))
    
    mail(to: user.email, subject: "โปรโมชั่นพิเศษสำหรับคุณ!")
  end
end
```

### Inline Image ใน View

```erb
<%# app/views/order_mailer/promotional_email.html.erb %>
<%= image_tag attachments['banner.jpg'].url, alt: "Banner" %>
```

---

## Step 1149: Multipart Emails

Rails ส่ง multipart email อัตโนมัติเมื่อมีทั้ง .html.erb และ .text.erb

```ruby
class UserMailer < ApplicationMailer
  def newsletter(user)
    @user = user
    @articles = Article.published.recent.limit(5)
    
    # Rails จะ render ทั้ง HTML และ text versions อัตโนมัติ
    # ถ้ามีทั้ง newsletter.html.erb และ newsletter.text.erb
    
    mail(
      to: user.email,
      subject: "Newsletter ประจำสัปดาห์"
    ) do |format|
      format.html  # renders newsletter.html.erb
      format.text  # renders newsletter.text.erb
    end
  end
end
```

---

## Step 1150: Testing Mailers

### RSpec

```ruby
# spec/mailers/user_mailer_spec.rb
require 'rails_helper'

RSpec.describe UserMailer, type: :mailer do
  let(:user) { create(:user) }
  
  describe "#welcome_email" do
    let(:mail) { described_class.with(user: user).welcome_email }
    
    it "renders the headers" do
      expect(mail.subject).to eq("ยินดีต้อนรับสู่ MyApp! ยืนยันอีเมลของคุณ")
      expect(mail.to).to eq([user.email])
      expect(mail.from).to eq(["noreply@myapp.com"])
    end
    
    it "renders the body" do
      expect(mail.body.encoded).to include(user.name)
      expect(mail.body.encoded).to include("ยืนยันอีเมล")
    end
    
    it "contains confirmation link" do
      expect(mail.body.encoded).to match(/confirm_email/)
    end
    
    it "is multipart" do
      expect(mail.parts.count).to eq(2)
      expect(mail.parts.map(&:content_type)).to include(
        match("text/html"),
        match("text/plain")
      )
    end
  end
  
  describe "#reset_password" do
    let(:mail) { described_class.with(user: user).reset_password }
    
    it "renders reset password link" do
      expect(mail.body.encoded).to include("password_reset")
    end
    
    it "expires in 2 hours" do
      expect(mail.body.encoded).to include("2 ชั่วโมง")
    end
  end
end
```

### ทดสอบการส่ง Mail ใน Feature Specs

```ruby
# spec/requests/registrations_spec.rb
require 'rails_helper'

RSpec.describe "Registrations", type: :request do
  include ActiveJob::TestHelper
  
  describe "POST /register" do
    let(:params) do
      {
        user: {
          name: "Test User",
          email: "test@example.com",
          password: "password123"
        }
      }
    end
    
    it "sends welcome email" do
      expect {
        post "/register", params: params
      }.to change(ActionMailer::Base.deliveries, :count).by(1)
    end
    
    it "enqueues welcome email job" do
      perform_enqueued_jobs do
        post "/register", params: params
      end
      
      email = ActionMailer::Base.deliveries.last
      expect(email.to).to include("test@example.com")
      expect(email.subject).to include("ยินดีต้อนรับ")
    end
  end
end
```

---

## Step 1151: SMTP Configuration

### SendGrid

```ruby
# config/environments/production.rb
config.action_mailer.smtp_settings = {
  user_name: 'apikey',  # ใช้ 'apikey' เสมอ
  password: ENV['SENDGRID_API_KEY'],
  domain: 'myapp.com',
  address: 'smtp.sendgrid.net',
  port: 587,
  authentication: :plain,
  enable_starttls_auto: true
}
```

### Mailgun

```ruby
config.action_mailer.smtp_settings = {
  port: 587,
  address: 'smtp.mailgun.org',
  user_name: ENV['MAILGUN_SMTP_LOGIN'],
  password: ENV['MAILGUN_SMTP_PASSWORD'],
  domain: 'myapp.com',
  authentication: :plain,
  enable_starttls_auto: true
}
```

### Amazon SES

```ruby
# Gemfile
gem 'aws-sdk-sesv2'

# config
config.action_mailer.delivery_method = :ses
config.action_mailer.ses_settings = {
  region: ENV['AWS_REGION'],
  access_key_id: ENV['AWS_ACCESS_KEY_ID'],
  secret_access_key: ENV['AWS_SECRET_ACCESS_KEY']
}

# หรือใช้ SMTP
config.action_mailer.smtp_settings = {
  address: "email-smtp.#{ENV['AWS_REGION']}.amazonaws.com",
  port: 587,
  domain: 'myapp.com',
  user_name: ENV['AWS_SES_SMTP_USERNAME'],
  password: ENV['AWS_SES_SMTP_PASSWORD'],
  authentication: :plain,
  enable_starttls_auto: true
}
```

---

## Step 1152: Order Mailer Complete Example

```ruby
# app/mailers/order_mailer.rb
class OrderMailer < ApplicationMailer
  before_action { @order = params[:order] }
  before_action { @user = @order.user }
  
  def confirmation
    @items = @order.items.includes(:product)
    @total = @order.total_amount
    @estimated_delivery = 3.business_days.from_now
    
    mail(
      to: @user.email,
      subject: "ยืนยันคำสั่งซื้อ ##{@order.id} - #{@order.total_amount_formatted}",
      message_id: "<order-#{@order.id}@myapp.com>"
    )
  end
  
  def shipped
    @tracking_number = @order.tracking_number
    @carrier = @order.carrier
    @tracking_url = @order.tracking_url
    @estimated_delivery = @order.estimated_delivery
    
    mail(
      to: @user.email,
      subject: "คำสั่งซื้อ ##{@order.id} ถูกจัดส่งแล้ว!"
    )
  end
  
  def delivered
    @order_date = @order.created_at.strftime("%d %B %Y")
    
    mail(
      to: @user.email,
      subject: "คำสั่งซื้อ ##{@order.id} ส่งถึงแล้ว - บอกเราว่าคุณรู้สึกอย่างไร"
    )
  end
  
  def refund_processed
    @refund_amount = @order.refund_amount
    @refund_reason = @order.refund_reason
    @refund_date = @order.refunded_at.strftime("%d %B %Y")
    
    mail(
      to: @user.email,
      subject: "การคืนเงินสำหรับคำสั่งซื้อ ##{@order.id} ดำเนินการแล้ว"
    )
  end
end
```

```erb
<%# app/views/order_mailer/confirmation.html.erb %>
<h2>ขอบคุณสำหรับคำสั่งซื้อของคุณ!</h2>

<p>สวัสดี <%= @user.name %>,</p>
<p>เราได้รับคำสั่งซื้อของคุณแล้ว</p>

<table style="width: 100%; border-collapse: collapse; margin: 20px 0;">
  <thead>
    <tr style="background-color: #f4f4f4;">
      <th style="padding: 10px; text-align: left; border: 1px solid #ddd;">สินค้า</th>
      <th style="padding: 10px; text-align: center; border: 1px solid #ddd;">จำนวน</th>
      <th style="padding: 10px; text-align: right; border: 1px solid #ddd;">ราคา</th>
    </tr>
  </thead>
  <tbody>
    <% @items.each do |item| %>
      <tr>
        <td style="padding: 10px; border: 1px solid #ddd;">
          <%= item.product.name %>
        </td>
        <td style="padding: 10px; text-align: center; border: 1px solid #ddd;">
          <%= item.quantity %>
        </td>
        <td style="padding: 10px; text-align: right; border: 1px solid #ddd;">
          <%= number_to_currency(item.subtotal, unit: "฿") %>
        </td>
      </tr>
    <% end %>
  </tbody>
  <tfoot>
    <tr>
      <td colspan="2" style="padding: 10px; text-align: right; font-weight: bold;">รวมทั้งหมด:</td>
      <td style="padding: 10px; text-align: right; font-weight: bold; color: #e74c3c;">
        <%= number_to_currency(@total, unit: "฿") %>
      </td>
    </tr>
  </tfoot>
</table>

<p>กำหนดจัดส่ง: <strong><%= @estimated_delivery.strftime("%d %B %Y") %></strong></p>

<p style="text-align: center;">
  <%= link_to "ดูรายละเอียดคำสั่งซื้อ", order_url(@order), class: "btn" %>
</p>
```

---

## แบบฝึกหัด (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** สร้าง UserMailer ด้วย generator
```bash
# เฉลย
rails generate mailer UserMailer welcome_email password_reset
```

**ข้อ 2:** เขียน welcome_email ที่ส่งชื่อและ confirmation link
```ruby
# เฉลย
def welcome_email
  @user = params[:user]
  @url = confirm_email_url(token: @user.confirmation_token)
  mail(to: @user.email, subject: "ยินดีต้อนรับ!")
end
```

**ข้อ 3:** ตั้งค่า Letter Opener สำหรับ development
```ruby
# เฉลย
# Gemfile: gem 'letter_opener'
# config/environments/development.rb:
config.action_mailer.delivery_method = :letter_opener
```

**ข้อ 4:** สร้าง mailer preview
```ruby
# เฉลย
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    UserMailer.with(user: User.first).welcome_email
  end
end
```

**ข้อ 5:** ส่ง email ใน background job
```ruby
# เฉลย
UserMailer.with(user: @user).welcome_email.deliver_later
```

### ระดับกลาง

**ข้อ 6:** เพิ่ม PDF attachment ใน order confirmation email
```ruby
# เฉลย
def order_confirmation(order)
  @order = order
  pdf = Prawn::Document.new
  pdf.text "Order ##{order.id}"
  attachments["invoice.pdf"] = pdf.render
  mail(to: order.user.email, subject: "Order Confirmation")
end
```

**ข้อ 7:** ตั้งค่า SendGrid SMTP
```ruby
# เฉลย
config.action_mailer.smtp_settings = {
  user_name: 'apikey',
  password: ENV['SENDGRID_API_KEY'],
  address: 'smtp.sendgrid.net',
  port: 587,
  authentication: :plain,
  enable_starttls_auto: true
}
```

**ข้อ 8:** เขียน test สำหรับ welcome_email
```ruby
# เฉลย
it "sends to correct address" do
  mail = UserMailer.with(user: user).welcome_email
  expect(mail.to).to include(user.email)
  expect(mail.subject).to include("ยินดีต้อนรับ")
end
```

**ข้อ 9:** สร้าง weekly digest email ที่ส่งทุกวันจันทร์
```ruby
# เฉลย - config/sidekiq.yml
:schedule:
  weekly_digest:
    cron: '0 9 * * 1'
    class: SendWeeklyDigestJob

# app/jobs/send_weekly_digest_job.rb
class SendWeeklyDigestJob < ApplicationJob
  def perform
    User.active.each do |user|
      UserMailer.with(user: user).weekly_digest.deliver_later
    end
  end
end
```

**ข้อ 10:** เพิ่ม unsubscribe link ใน email
```erb
<%# เฉลย %>
<%= link_to "ยกเลิกการรับอีเมล", 
            unsubscribe_url(token: @user.unsubscribe_token) %>
```

### ระดับสูง

**ข้อ 11-20:** (แบบฝึกหัดเพิ่มเติม)

```ruby
# เฉลย ข้อ 11 - Track email opens
class TrackableMailer < ApplicationMailer
  def self.tracking_pixel_url(user, email_type)
    "#{Rails.application.routes.url_helpers.root_url}email_opens/track?user=#{user.id}&type=#{email_type}&t=#{Time.current.to_i}"
  end
end

# view
<img src="<%= TrackableMailer.tracking_pixel_url(@user, 'welcome') %>" width="1" height="1">
```

```ruby
# เฉลย ข้อ 14 - Email bounce handling
class EmailBounceController < ApplicationController
  skip_before_action :authenticate_user!
  
  def create
    bounce = params[:bounce]
    user = User.find_by(email: bounce[:email])
    
    if user && bounce[:type] == 'permanent'
      user.update!(email_bounced: true, email_bounce_at: Time.current)
    end
    
    head :ok
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Action Mailer Setup** - configuration สำหรับ development และ production
2. **Generating Mailers** - สร้าง mailer classes
3. **Views** - HTML และ text email templates
4. **deliver_now vs deliver_later** - เลือกวิธีส่งที่เหมาะสม
5. **Preview** - ดู email ใน browser ก่อนส่ง
6. **Letter Opener** - เปิด email ใน browser ระหว่าง development
7. **Attachments** - แนบ files และ inline images
8. **Testing** - test mailers ด้วย RSpec
9. **SMTP** - ตั้งค่า SendGrid, Mailgun, SES

Action Mailer เป็นส่วนสำคัญของ user engagement ใน web applications ควรใช้ `deliver_later` เสมอใน production เพื่อไม่ให้ส่งผลต่อ response time
