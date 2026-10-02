# ตอนที่ 54: Internationalization/i18n (Steps 1181-1200)

## บทนำ

i18n (Internationalization) คือกระบวนการทำให้ application รองรับหลายภาษาและ locales Rails มี built-in i18n support ที่แข็งแกร่งด้วย `I18n` module ทำให้แปลข้อความ, วันที่, ตัวเลข และ currencies ได้ง่าย

---

## Step 1181: I18n คืออะไร?

### Internationalization vs Localization

- **i18n (Internationalization)** - กระบวนการออกแบบ app ให้รองรับหลายภาษา
- **l10n (Localization)** - การปรับ app สำหรับ locale เฉพาะ (ภาษา, รูปแบบ, วัฒนธรรม)

```
App (English) → i18n → App (English, Thai, Japanese, ...)
                l10n → "Hello" → "สวัสดี" (Thai)
                        "$1,234.56" → "฿1,234.56" (Thai currency)
                        "Jan 1, 2024" → "1 ม.ค. 2567" (Thai calendar)
```

---

## Step 1182: Setup และ config/locales/

### Initial Setup

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # ภาษา default
    config.i18n.default_locale = :th
    
    # Load path สำหรับ locale files
    config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
    
    # Locales ที่รองรับ
    config.i18n.available_locales = [:th, :en]
    
    # Fallback locale ถ้าไม่มี translation
    config.i18n.fallbacks = true
  end
end
```

### โครงสร้าง Locale Files

```
config/locales/
├── th.yml                    # Thai translations
├── en.yml                    # English translations
├── models/
│   ├── th.yml                # Model names (Thai)
│   └── en.yml                # Model names (English)
├── views/
│   ├── th.yml                # View translations (Thai)
│   └── en.yml                # View translations (English)
├── activerecord/
│   ├── th.yml                # AR error messages (Thai)
│   └── en.yml                # AR error messages (English)
└── devise/
    ├── th.yml                # Devise messages (Thai)
    └── en.yml
```

---

## Step 1183: YAML Translation Files

### Thai Translation (th.yml)

```yaml
# config/locales/th.yml
th:
  # App Name
  app_name: "My Application"
  
  # Navigation
  nav:
    home: "หน้าแรก"
    articles: "บทความ"
    about: "เกี่ยวกับ"
    contact: "ติดต่อ"
    login: "เข้าสู่ระบบ"
    logout: "ออกจากระบบ"
    register: "สมัครสมาชิก"
    profile: "โปรไฟล์"
    settings: "การตั้งค่า"
  
  # Common actions
  actions:
    create: "สร้าง"
    edit: "แก้ไข"
    update: "อัปเดต"
    delete: "ลบ"
    save: "บันทึก"
    cancel: "ยกเลิก"
    back: "กลับ"
    submit: "ส่ง"
    search: "ค้นหา"
    confirm: "ยืนยัน"
    yes: "ใช่"
    no: "ไม่"
    
  # Common messages
  messages:
    success: "ดำเนินการสำเร็จ"
    error: "เกิดข้อผิดพลาด"
    not_found: "ไม่พบข้อมูลที่ต้องการ"
    unauthorized: "คุณไม่มีสิทธิ์เข้าถึง"
    
  # Flash messages
  flash:
    article:
      created: "สร้างบทความสำเร็จ"
      updated: "อัปเดตบทความสำเร็จ"
      deleted: "ลบบทความสำเร็จ"
    user:
      created: "สมัครสมาชิกสำเร็จ"
      updated: "อัปเดตข้อมูลสำเร็จ"
      logged_in: "เข้าสู่ระบบสำเร็จ"
      logged_out: "ออกจากระบบสำเร็จ"
  
  # Pagination
  pagination:
    previous: "← ก่อนหน้า"
    next: "ถัดไป →"
    first: "หน้าแรก"
    last: "หน้าสุดท้าย"
    page: "หน้าที่"
    of: "จาก"
    total: "รวมทั้งหมด %{count} รายการ"
  
  # Time
  time:
    formats:
      short: "%d %b %y %H:%M น."
      long: "%d %B %Y เวลา %H:%M น."
      date_only: "%d %B %Y"
      time_only: "%H:%M น."
  
  # Pluralization
  items:
    zero: "ไม่มีรายการ"
    one: "%{count} รายการ"
    other: "%{count} รายการ"
```

### English Translation (en.yml)

```yaml
# config/locales/en.yml
en:
  app_name: "My Application"
  
  nav:
    home: "Home"
    articles: "Articles"
    about: "About"
    contact: "Contact"
    login: "Login"
    logout: "Logout"
    register: "Register"
    profile: "Profile"
    settings: "Settings"
  
  actions:
    create: "Create"
    edit: "Edit"
    update: "Update"
    delete: "Delete"
    save: "Save"
    cancel: "Cancel"
    back: "Back"
    submit: "Submit"
    search: "Search"
    confirm: "Confirm"
    yes: "Yes"
    no: "No"
  
  flash:
    article:
      created: "Article was successfully created."
      updated: "Article was successfully updated."
      deleted: "Article was successfully deleted."
  
  items:
    zero: "no items"
    one: "%{count} item"
    other: "%{count} items"
```

### Model Translations

```yaml
# config/locales/models/th.yml
th:
  activerecord:
    models:
      user:
        one: "ผู้ใช้"
        other: "ผู้ใช้"
      article:
        one: "บทความ"
        other: "บทความ"
      comment:
        one: "ความคิดเห็น"
        other: "ความคิดเห็น"
    
    attributes:
      user:
        name: "ชื่อ"
        email: "อีเมล"
        password: "รหัสผ่าน"
        password_confirmation: "ยืนยันรหัสผ่าน"
        role: "บทบาท"
        created_at: "วันที่สมัคร"
      
      article:
        title: "หัวข้อ"
        body: "เนื้อหา"
        published: "เผยแพร่"
        views_count: "ยอดวิว"
        created_at: "วันที่สร้าง"
        updated_at: "วันที่แก้ไขล่าสุด"
      
      comment:
        body: "ความคิดเห็น"
        created_at: "วันที่แสดงความคิดเห็น"
    
    errors:
      models:
        user:
          attributes:
            email:
              taken: "อีเมลนี้ถูกใช้แล้ว"
              invalid: "รูปแบบอีเมลไม่ถูกต้อง"
            password:
              too_short: "รหัสผ่านต้องมีอย่างน้อย %{count} ตัวอักษร"
```

---

## Step 1184: t() Helper ใน Views และ Controllers

### ใน Views

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html lang="<%= I18n.locale %>">
<head>
  <title><%= t('app_name') %></title>
</head>
<body>
  <nav>
    <%= link_to t('nav.home'), root_path %>
    <%= link_to t('nav.articles'), articles_path %>
    
    <% if user_signed_in? %>
      <%= link_to t('nav.profile'), profile_path %>
      <%= link_to t('nav.logout'), logout_path, method: :delete %>
    <% else %>
      <%= link_to t('nav.login'), login_path %>
      <%= link_to t('nav.register'), register_path %>
    <% end %>
  </nav>
  
  <main>
    <% flash.each do |type, message| %>
      <div class="flash flash-<%= type %>">
        <%= message %>
      </div>
    <% end %>
    
    <%= yield %>
  </main>
</body>
</html>
```

```erb
<%# app/views/articles/index.html.erb %>
<h1><%= t('nav.articles') %></h1>
<p><%= t('items', count: @articles.total_count) %></p>

<% @articles.each do |article| %>
  <article>
    <h2><%= article.title %></h2>
    <p><%= t('activerecord.attributes.article.created_at') %>:
       <%= l(article.created_at, format: :long) %></p>
  </article>
<% end %>
```

### Lazy Lookup (relative paths)

```erb
<%# app/views/articles/show.html.erb %>
<%# Rails จะมองหา th.articles.show.title %>
<h1><%= t('.title') %></h1>
<p><%= t('.published_by', name: @article.user.name) %></p>
```

```yaml
# config/locales/th.yml
th:
  articles:
    show:
      title: "รายละเอียดบทความ"
      published_by: "เขียนโดย %{name}"
    index:
      title: "บทความทั้งหมด"
      new_article: "เขียนบทความใหม่"
    form:
      title_label: "หัวข้อ"
      body_label: "เนื้อหา"
      submit: "บันทึกบทความ"
```

### ใน Controllers

```ruby
# app/controllers/articles_controller.rb
def create
  @article = current_user.articles.build(article_params)
  
  if @article.save
    redirect_to @article, notice: t('flash.article.created')
  else
    render :new, status: :unprocessable_entity
  end
end

def destroy
  @article.destroy
  redirect_to articles_path, notice: t('flash.article.deleted')
end
```

### t() กับ Interpolation

```yaml
th:
  welcome: "ยินดีต้อนรับ, %{name}!"
  articles_count: "คุณมี %{count} บทความ"
  last_login: "เข้าสู่ระบบครั้งล่าสุด: %{time}"
```

```erb
<%= t('welcome', name: current_user.name) %>
<%= t('articles_count', count: current_user.articles.count) %>
<%= t('last_login', time: l(current_user.last_sign_in_at, format: :short)) %>
```

---

## Step 1185: Locale Switching

### URL-based Switching

```ruby
# config/routes.rb
scope "/:locale", locale: /en|th/ do
  resources :articles
  root 'home#index'
end

# หรือ prefix ทุก routes
Rails.application.routes.draw do
  scope '/:locale', locale: /th|en/, defaults: { locale: 'th' } do
    resources :articles
    resources :users
    root 'home#index'
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_locale
  
  def set_locale
    I18n.locale = extract_locale || I18n.default_locale
  end
  
  def extract_locale
    # 1. จาก URL params (สูงสุด)
    parsed_locale = params[:locale]
    return parsed_locale if I18n.available_locales.map(&:to_s).include?(parsed_locale)
    
    # 2. จาก user preferences (ถ้า login)
    return current_user.locale if user_signed_in? && current_user.locale.present?
    
    # 3. จาก cookie
    return cookies[:locale] if cookies[:locale].present?
    
    # 4. จาก Accept-Language header
    extract_locale_from_accept_language_header
  end
  
  def extract_locale_from_accept_language_header
    request.env['HTTP_ACCEPT_LANGUAGE']&.scan(/^[a-z]{2}/)&.first
  end
  
  # Helper method สำหรับ URL helpers
  def default_url_options
    { locale: I18n.locale }
  end
end
```

### Language Switcher ใน View

```erb
<%# app/views/shared/_language_switcher.html.erb %>
<div class="language-switcher">
  <% I18n.available_locales.each do |locale| %>
    <%= link_to locale_display_name(locale), 
                url_for(locale: locale),
                class: "lang-btn #{locale == I18n.locale ? 'active' : ''}" %>
  <% end %>
</div>
```

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def locale_display_name(locale)
    case locale.to_sym
    when :th then "ภาษาไทย 🇹🇭"
    when :en then "English 🇬🇧"
    when :ja then "日本語 🇯🇵"
    else locale.to_s.upcase
    end
  end
end
```

### Subdomain-based Switching

```ruby
# config/routes.rb
constraints subdomain: 'en' do
  scope locale: :en do
    resources :articles
  end
end

constraints subdomain: 'th' do
  scope locale: :th do
    resources :articles
  end
end
```

---

## Step 1186: Active Record Error Messages

```yaml
# config/locales/th.yml
th:
  errors:
    format: "%{attribute} %{message}"
    messages:
      blank: "ต้องไม่เว้นว่าง"
      empty: "ต้องไม่ว่างเปล่า"
      invalid: "ไม่ถูกต้อง"
      taken: "ถูกใช้ไปแล้ว"
      too_short:
        one: "สั้นเกินไป (ต้องมีอย่างน้อย %{count} ตัวอักษร)"
        other: "สั้นเกินไป (ต้องมีอย่างน้อย %{count} ตัวอักษร)"
      too_long:
        one: "ยาวเกินไป (ต้องไม่เกิน %{count} ตัวอักษร)"
        other: "ยาวเกินไป (ต้องไม่เกิน %{count} ตัวอักษร)"
      not_a_number: "ต้องเป็นตัวเลข"
      greater_than: "ต้องมากกว่า %{count}"
      greater_than_or_equal_to: "ต้องมากกว่าหรือเท่ากับ %{count}"
      less_than: "ต้องน้อยกว่า %{count}"
      less_than_or_equal_to: "ต้องน้อยกว่าหรือเท่ากับ %{count}"
      inclusion: "ไม่อยู่ในรายการที่อนุญาต"
      exclusion: "ถูก reserved ไว้"
      confirmation: "ไม่ตรงกัน"
      accepted: "ต้องยอมรับ"
      present: "ต้องไม่มีค่า"
  
  activerecord:
    errors:
      messages:
        record_invalid: "Validation failed: %{errors}"
        restrict_dependent_destroy:
          has_one: "ไม่สามารถลบ record นี้ได้เนื่องจากมี %{record} ที่เกี่ยวข้อง"
          has_many: "ไม่สามารถลบ record นี้ได้เนื่องจากมี %{record} ที่เกี่ยวข้อง"
```

### แสดง Error Messages

```erb
<%# app/views/shared/_error_messages.html.erb %>
<% if object.errors.any? %>
  <div class="error-messages">
    <h3>
      <%= t('errors.title', 
            count: object.errors.count,
            model: object.class.model_name.human) %>
    </h3>
    <ul>
      <% object.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>
```

```yaml
th:
  errors:
    title:
      one: "เกิดข้อผิดพลาด 1 รายการที่ทำให้ไม่สามารถบันทึก %{model} ได้"
      other: "เกิดข้อผิดพลาด %{count} รายการที่ทำให้ไม่สามารถบันทึก %{model} ได้"
```

---

## Step 1187: Date และ Time Localization

### l() Helper

```erb
<%# l() สำหรับ localize dates/times %>
<%= l(article.created_at)                        # default format %>
<%= l(article.created_at, format: :short)        # short format %>
<%= l(article.created_at, format: :long)         # long format %>
<%= l(Date.today, format: :date_only)            # date only %>
```

### Thai Date Format

```yaml
# config/locales/th.yml
th:
  date:
    formats:
      default: "%d/%m/%Y"
      short: "%d %b"
      long: "%d %B %Y"
      thai_long: "วันที่ %d เดือน %B พ.ศ. %Y"
    
    day_names:
      - อาทิตย์
      - จันทร์
      - อังคาร
      - พุธ
      - พฤหัสบดี
      - ศุกร์
      - เสาร์
    
    abbr_day_names:
      - อา.
      - จ.
      - อ.
      - พ.
      - พฤ.
      - ศ.
      - ส.
    
    month_names:
      - ~
      - มกราคม
      - กุมภาพันธ์
      - มีนาคม
      - เมษายน
      - พฤษภาคม
      - มิถุนายน
      - กรกฎาคม
      - สิงหาคม
      - กันยายน
      - ตุลาคม
      - พฤศจิกายน
      - ธันวาคม
    
    abbr_month_names:
      - ~
      - ม.ค.
      - ก.พ.
      - มี.ค.
      - เม.ย.
      - พ.ค.
      - มิ.ย.
      - ก.ค.
      - ส.ค.
      - ก.ย.
      - ต.ค.
      - พ.ย.
      - ธ.ค.
    
    order:
      - :day
      - :month
      - :year
  
  time:
    formats:
      default: "%a, %d %b %Y %H:%M:%S %z"
      short: "%d %b %H:%M น."
      long: "%d %B %Y %H:%M น."
    
    am: "AM"
    pm: "PM"
```

---

## Step 1188: Number และ Currency Localization

```yaml
# config/locales/th.yml
th:
  number:
    format:
      separator: "."
      delimiter: ","
      precision: 2
      significant: false
      strip_insignificant_zeros: false
    
    currency:
      format:
        format: "%u%n"
        unit: "฿"
        separator: "."
        delimiter: ","
        precision: 2
    
    percentage:
      format:
        delimiter: ""
        format: "%n%"
    
    human:
      format:
        delimiter: ""
        precision: 3
        significant: true
      
      storage_units:
        format: "%n %u"
        units:
          byte:
            one: "ไบต์"
            other: "ไบต์"
          kb: "KB"
          mb: "MB"
          gb: "GB"
          tb: "TB"
```

### ใช้งาน Number Helpers

```erb
<%= number_to_currency(1234.56) %>           # ฿1,234.56
<%= number_to_percentage(85.5) %>            # 85.5%
<%= number_to_human(1_234_567) %>            # 1.23 ล้าน (ถ้าตั้งค่า)
<%= number_to_human_size(1.megabyte) %>      # 1 MB
<%= number_with_delimiter(1234567) %>        # 1,234,567
<%= number_with_precision(3.14159, precision: 2) %>  # 3.14
```

---

## Step 1189: Plural Rules

### Thai Plural (ไม่มี plural)

```yaml
# config/locales/th.yml
th:
  # Thai ไม่มี plural form แต่ Rails ยังต้องมี key
  comments:
    zero: "ยังไม่มีความคิดเห็น"
    one: "1 ความคิดเห็น"
    other: "%{count} ความคิดเห็น"
  
  articles_by:
    zero: "ยังไม่มีบทความโดย %{name}"
    one: "1 บทความโดย %{name}"
    other: "%{count} บทความโดย %{name}"
```

```erb
<%= t('comments', count: @article.comments.count) %>
<%= t('articles_by', count: @user.articles.count, name: @user.name) %>
```

---

## Step 1190: Thai Language Complete Setup

### Thai Locale File ที่สมบูรณ์

```yaml
# config/locales/th.yml
th:
  app_name: "แอปพลิเคชัน"
  
  # Devise messages
  devise:
    sessions:
      signed_in: "เข้าสู่ระบบสำเร็จ"
      signed_out: "ออกจากระบบสำเร็จ"
      already_signed_in: "คุณเข้าสู่ระบบอยู่แล้ว"
    
    registrations:
      signed_up: "สมัครสมาชิกสำเร็จ โปรดยืนยันอีเมลของคุณ"
      signed_up_but_unconfirmed: "สมัครสมาชิกสำเร็จ โปรดตรวจสอบอีเมลของคุณ"
      update_needs_confirmation: "อัปเดตข้อมูลสำเร็จ โปรดยืนยันอีเมลใหม่ของคุณ"
      destroyed: "ลบบัญชีสำเร็จ"
    
    passwords:
      send_instructions: "คุณจะได้รับอีเมลสำหรับ reset รหัสผ่านในไม่ช้า"
      send_paranoid_instructions: "หากอีเมลของคุณมีอยู่ในระบบ คุณจะได้รับอีเมลในไม่ช้า"
      updated: "เปลี่ยนรหัสผ่านสำเร็จ"
      updated_not_active: "เปลี่ยนรหัสผ่านสำเร็จ"
    
    confirmations:
      send_instructions: "คุณจะได้รับอีเมลสำหรับยืนยันในไม่ช้า"
      send_paranoid_instructions: "หากอีเมลของคุณมีอยู่ในระบบและยังไม่ได้ยืนยัน คุณจะได้รับอีเมลในไม่ช้า"
      confirmed: "ยืนยันอีเมลสำเร็จ"
      
    mailer:
      confirmation_instructions:
        subject: "ยืนยันอีเมลของคุณ"
      reset_password_instructions:
        subject: "Reset รหัสผ่าน"
      unlock_instructions:
        subject: "ปลดล็อกบัญชีของคุณ"
      email_changed:
        subject: "อีเมลของคุณเปลี่ยนแปลงแล้ว"
      password_change:
        subject: "รหัสผ่านของคุณเปลี่ยนแปลงแล้ว"
  
  # Kaminari pagination
  views:
    pagination:
      first: "&laquo; หน้าแรก"
      last: "หน้าสุดท้าย &raquo;"
      previous: "&lsaquo; ก่อนหน้า"
      next: "ถัดไป &rsaquo;"
      truncate: "..."
```

---

## แบบฝึกหัด (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** ตั้งค่า i18n ให้ default locale เป็น Thai
```ruby
# เฉลย
# config/application.rb
config.i18n.default_locale = :th
config.i18n.available_locales = [:th, :en]
```

**ข้อ 2:** สร้าง th.yml ที่มี navigation translations
```yaml
# เฉลย
th:
  nav:
    home: "หน้าแรก"
    articles: "บทความ"
    login: "เข้าสู่ระบบ"
```

**ข้อ 3:** ใช้ t() helper ใน view
```erb
<%# เฉลย %>
<h1><%= t('nav.home') %></h1>
<%= link_to t('actions.create'), new_article_path %>
```

**ข้อ 4:** ใช้ l() helper แสดงวันที่เป็นภาษาไทย
```erb
<%# เฉลย %>
<%= l(@article.created_at, format: :long) %>
```

**ข้อ 5:** เพิ่ม locale switcher ใน navigation
```erb
<%# เฉลย %>
<%= link_to "ภาษาไทย", url_for(locale: :th) %>
<%= link_to "English", url_for(locale: :en) %>
```

### ระดับกลาง

**ข้อ 6:** ตั้งค่า URL-based locale switching
```ruby
# เฉลย
scope "/:locale", locale: /en|th/ do
  resources :articles
end

before_action :set_locale

def set_locale
  I18n.locale = params[:locale] || I18n.default_locale
end
```

**ข้อ 7:** แปล Active Record error messages เป็นภาษาไทย
```yaml
# เฉลย
th:
  errors:
    messages:
      blank: "ต้องไม่เว้นว่าง"
      too_short:
        other: "สั้นเกินไป (ต้องมีอย่างน้อย %{count} ตัวอักษร)"
```

**ข้อ 8:** เพิ่ม Thai date format
```yaml
# เฉลย
th:
  date:
    formats:
      long: "%d %B %Y"
    month_names:
      - ~
      - มกราคม
      - กุมภาพันธ์
      # ...ต่อไปจนถึงธันวาคม
```

**ข้อ 9:** ใช้ number_to_currency กับ Thai Baht
```yaml
# เฉลย
th:
  number:
    currency:
      format:
        unit: "฿"
        format: "%u%n"
```

**ข้อ 10:** เพิ่ม plural rules สำหรับ articles count
```yaml
# เฉลย
th:
  articles_count:
    zero: "ยังไม่มีบทความ"
    one: "1 บทความ"
    other: "%{count} บทความ"
```

### ระดับสูง

**ข้อ 11-20:**

```ruby
# เฉลย ข้อ 11 - User locale preference
class User < ApplicationRecord
  validates :locale, inclusion: { in: %w[th en] }, allow_nil: true
end

# ApplicationController
def set_locale
  I18n.locale = if user_signed_in? && current_user.locale.present?
    current_user.locale
  else
    params[:locale] || cookies[:locale] || I18n.default_locale
  end
end
```

```ruby
# เฉลย ข้อ 14 - Accept-Language header
def extract_locale_from_headers
  accept_language = request.env['HTTP_ACCEPT_LANGUAGE']
  return nil unless accept_language
  
  locales = accept_language.scan(/[a-z]{2}/).map(&:to_sym)
  locales.find { |l| I18n.available_locales.include?(l) }
end
```

```ruby
# เฉลย ข้อ 17 - Translation helper ใน tests
RSpec.configure do |config|
  config.include AbstractController::Translation
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **I18n Setup** - configuration สำหรับ multi-language
2. **YAML Files** - โครงสร้าง translation files
3. **t() Helper** - แปลข้อความใน views และ controllers
4. **l() Helper** - localize dates, times, numbers
5. **Locale Switching** - URL, subdomain, cookie, header
6. **AR Error Messages** - แปล validation messages
7. **Thai Date/Time** - Thai month names และ formats
8. **Number/Currency** - Thai Baht formatting
9. **Plural Rules** - handle singular/plural ภาษาไทย

i18n เป็นสิ่งสำคัญมากสำหรับ applications ที่มี users จากหลายประเทศ ควรวางแผน i18n ตั้งแต่ต้นเพราะการเพิ่มทีหลังจะยากกว่า
