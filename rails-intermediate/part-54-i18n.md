# Part 54: Internationalization (I18n) ใน Rails

## ขั้นตอนที่ 1181-1200: รองรับหลายภาษา

---

## ขั้นตอนที่ 1181: I18n Overview

```ruby
# Rails รองรับ I18n ตั้งแต่ต้น
I18n.locale         # => :en (default)
I18n.default_locale # => :en

I18n.t("hello")     # translate
I18n.l(Date.today)  # localize date/time
```

## ขั้นตอนที่ 1182: Setup

```ruby
# config/application.rb
config.i18n.default_locale = :th        # ภาษาเริ่มต้น
config.i18n.available_locales = [:th, :en]
config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.yml').to_s]
config.i18n.fallbacks = [I18n.default_locale]
```

```yaml
# config/locales/th.yml
th:
  hello: "สวัสดี"
  goodbye: "ลาก่อน"
  
  # Nested keys
  users:
    title: "รายการผู้ใช้"
    new: "เพิ่มผู้ใช้ใหม่"
    edit: "แก้ไขผู้ใช้"
    
  posts:
    title: "บทความ"
    published: "เผยแพร่แล้ว"
    draft: "แบบร่าง"
    
  # Interpolation
  greetings:
    welcome: "ยินดีต้อนรับ %{name}!"
    count: "มีบทความทั้งหมด %{count} บทความ"
    
  # Pluralization
  messages:
    one: "1 ข้อความ"
    other: "%{count} ข้อความ"
```

```yaml
# config/locales/en.yml
en:
  hello: "Hello"
  goodbye: "Goodbye"
  
  users:
    title: "Users"
    new: "New User"
    edit: "Edit User"
    
  posts:
    title: "Posts"
    published: "Published"
    draft: "Draft"
    
  greetings:
    welcome: "Welcome, %{name}!"
    count: "Total %{count} posts"
    
  messages:
    one: "1 message"
    other: "%{count} messages"
```

## ขั้นตอนที่ 1183: Translation ใน Views

```erb
<%# ใช้ t() helper %>
<h1><%= t('posts.title') %></h1>

<%# Short form (ใช้ relative key) %>
<%# ใน app/views/posts/index.html.erb %>
<h1><%= t('.title') %></h1>
<%# จะ translate 'posts.index.title' %>

<%# Interpolation %>
<p><%= t('greetings.welcome', name: current_user.name) %></p>

<%# Pluralization %>
<p><%= t('messages', count: @user.messages.count) %></p>

<%# HTML safe translation %>
<%# Translation key ที่ลงท้ายด้วย _html จะไม่ escape %>
<p><%= t('disclaimer_html') %></p>

<%# Default value %>
<p><%= t('unknown_key', default: 'ไม่พบข้อความ') %></p>

<%# Localize date/time %>
<p><%= l(@post.created_at, format: :short) %></p>
<p><%= l(Date.today, format: :long) %></p>
```

## ขั้นตอนที่ 1184: Locale Switching

```ruby
# routes.rb - locale ใน URL
scope "/:locale" do
  resources :posts
  resources :pages
end

# หรือ optional locale
scope "(:locale)", locale: /th|en/ do
  resources :posts
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_locale
  
  private
  
  def set_locale
    I18n.locale = extract_locale || I18n.default_locale
  end
  
  def extract_locale
    # 1. จาก URL params
    if params[:locale] && I18n.available_locales.include?(params[:locale].to_sym)
      return params[:locale].to_sym
    end
    
    # 2. จาก user preference
    if user_signed_in? && current_user.locale.present?
      return current_user.locale.to_sym
    end
    
    # 3. จาก session
    if session[:locale]
      return session[:locale].to_sym
    end
    
    # 4. จาก Accept-Language header
    http_accept_language = request.env['HTTP_ACCEPT_LANGUAGE']
    if http_accept_language
      locale = http_accept_language.scan(/[a-z]{2}/).first
      return locale.to_sym if I18n.available_locales.include?(locale.to_sym)
    end
    
    nil
  end
  
  def default_url_options
    { locale: I18n.locale }
  end
end
```

```ruby
# Locale controller
class LocalesController < ApplicationController
  def update
    locale = params[:locale].to_sym
    
    if I18n.available_locales.include?(locale)
      session[:locale] = locale
      current_user&.update(locale: locale)
    end
    
    redirect_back fallback_location: root_path
  end
end
```

```erb
<%# Locale switcher in view %>
<div class="locale-switcher">
  <%= link_to "🇹🇭 ไทย", url_for(locale: :th), 
              class: I18n.locale == :th ? "active" : "" %>
  <%= link_to "🇬🇧 English", url_for(locale: :en),
              class: I18n.locale == :en ? "active" : "" %>
</div>
```

## ขั้นตอนที่ 1185: ActiveRecord Error Translations

```yaml
# config/locales/th.yml
th:
  activerecord:
    models:
      user: "ผู้ใช้"
      post: "บทความ"
      comment: "ความเห็น"
    
    attributes:
      user:
        name: "ชื่อ"
        email: "อีเมล"
        password: "รหัสผ่าน"
        phone: "เบอร์โทร"
      post:
        title: "หัวข้อ"
        content: "เนื้อหา"
        category: "หมวดหมู่"
    
    errors:
      models:
        user:
          attributes:
            email:
              blank: "กรุณากรอกอีเมล"
              taken: "อีเมลนี้มีผู้ใช้แล้ว"
              invalid: "รูปแบบอีเมลไม่ถูกต้อง"
            password:
              too_short: "รหัสผ่านต้องมีอย่างน้อย %{count} ตัวอักษร"
  
  errors:
    messages:
      blank: "ไม่สามารถเว้นว่างได้"
      taken: "มีคนใช้แล้ว"
      invalid: "ไม่ถูกต้อง"
      too_short: "สั้นเกินไป (ต้องมีอย่างน้อย %{count} ตัวอักษร)"
      too_long: "ยาวเกินไป (ไม่เกิน %{count} ตัวอักษร)"
      not_a_number: "ต้องเป็นตัวเลข"
      greater_than: "ต้องมากกว่า %{count}"
      less_than: "ต้องน้อยกว่า %{count}"
      inclusion: "ไม่อยู่ในรายการที่กำหนด"
      confirmation: "ไม่ตรงกัน"
```

## ขั้นตอนที่ 1186: Date and Number Localization

```yaml
# config/locales/th.yml
th:
  date:
    formats:
      default: "%d/%m/%Y"
      short: "%d %b"
      long: "%d %B %Y"
    
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
  
  time:
    formats:
      default: "%a %d %b %Y เวลา %H:%M น."
      short: "%d %b %H:%M น."
      long: "%A %d %B %Y เวลา %H:%M:%S น."
  
  number:
    format:
      separator: "."
      delimiter: ","
      precision: 2
    
    currency:
      format:
        unit: "฿"
        precision: 2
        separator: "."
        delimiter: ","
        format: "%u%n"
    
    percentage:
      format:
        delimiter: ""
    
    human:
      storage_units:
        units:
          byte: ไบต์
          kb: KB
          mb: MB
          gb: GB
          tb: TB
```

## ขั้นตอนที่ 1187: Thai Buddhist Era (พ.ศ.)

```ruby
# config/initializers/thai_date.rb
module ThaiDate
  def to_thai_date(format = :default)
    year = strftime("%Y").to_i + 543
    
    case format
    when :short
      strftime("%-d %b ") + year.to_s[-2..]
    when :long
      strftime("%-d %B ") + year.to_s + " (" + strftime("%H:%M น.") + ")"
    else
      strftime("%-d/%-m/") + year.to_s
    end
  end
end

class Date
  include ThaiDate
end

class Time
  include ThaiDate
end

class DateTime
  include ThaiDate
end
```

```erb
<%# ใน view %>
<p><%= @post.created_at.to_thai_date(:long) %></p>
<%# => "1 มกราคม 2567 (12:00 น.)" %>
```

## ขั้นตอนที่ 1188: Devise I18n

```ruby
# Gemfile
gem 'devise-i18n'

# bundle install
# rails generate devise:i18n:views -l th
```

```yaml
# config/locales/devise.th.yml
th:
  devise:
    sessions:
      signed_in: "เข้าสู่ระบบสำเร็จ"
      signed_out: "ออกจากระบบสำเร็จ"
    
    registrations:
      signed_up: "สมัครสมาชิกสำเร็จ"
      destroyed: "ลบบัญชีสำเร็จ"
      updated: "อัพเดทข้อมูลสำเร็จ"
    
    passwords:
      send_instructions: "ส่งคำแนะนำรีเซ็ตรหัสผ่านไปยังอีเมลของคุณแล้ว"
      updated: "เปลี่ยนรหัสผ่านสำเร็จ"
    
    confirmations:
      send_instructions: "ส่งคำแนะนำยืนยันไปยังอีเมลของคุณแล้ว"
      confirmed: "ยืนยันบัญชีสำเร็จ"
    
    failures:
      already_authenticated: "คุณเข้าสู่ระบบแล้ว"
      unauthenticated: "กรุณาเข้าสู่ระบบก่อน"
      unconfirmed: "กรุณายืนยันอีเมลก่อน"
      locked: "บัญชีถูกล็อก"
      invalid: "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
      invalid_token: "token ไม่ถูกต้อง"
      timeout: "session หมดอายุ กรุณาเข้าสู่ระบบใหม่"
      not_found_in_database: "ไม่พบอีเมลหรือรหัสผ่านนี้ในระบบ"
```

## ขั้นตอนที่ 1189: Translating Enum Values

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  enum status: {
    draft: 0,
    published: 1,
    archived: 2
  }
  
  def status_label
    I18n.t("post.statuses.#{status}")
  end
end
```

```yaml
# config/locales/th.yml
th:
  post:
    statuses:
      draft: "แบบร่าง"
      published: "เผยแพร่"
      archived: "เก็บถาวร"
```

```erb
<p>สถานะ: <%= @post.status_label %></p>

<%# หรือใน select %>
<%= f.select :status, 
    Post.statuses.keys.map { |s| [I18n.t("post.statuses.#{s}"), s] } %>
```

## ขั้นตอนที่ 1190: i18n-tasks Gem

```ruby
# Gemfile
group :development do
  gem 'i18n-tasks'
end
```

```bash
# ตรวจสอบ missing translations
rails i18n:missing_keys

# ตรวจสอบ unused translations
rails i18n:unused_keys

# Copy keys จาก en ไป th (ใช้ machine translation)
rails i18n:add_missing

# Normalize (sort + format) YAML files
rails i18n:normalize
```

---

## แบบฝึกหัด: I18n (20 ข้อ)

### ข้อที่ 1: Setup I18n
```ruby
config.i18n.default_locale = :th
config.i18n.available_locales = [:th, :en]
```

### ข้อที่ 2: YAML Translation Files
```
สร้าง th.yml และ en.yml สำหรับ navigation menu
```

### ข้อที่ 3: Locale Switcher
```
สร้าง UI ให้ user เปลี่ยนภาษาได้
```

### ข้อที่ 4: ActiveRecord Translations
```
Translate model name และ attribute names เป็นภาษาไทย
```

### ข้อที่ 5: Date Localization
```
แสดงวันที่เป็นภาษาไทย เช่น "1 มกราคม 2567"
```

### ข้อที่ 6-20 (แบบสรุป)

**ข้อ 6:** Locale ใน URL path (`/th/posts`, `/en/posts`)
**ข้อ 7:** Translate error messages
**ข้อ 8:** Pluralization rules
**ข้อ 9:** Number/currency formatting
**ข้อ 10:** Buddhist Era year conversion
**ข้อ 11:** Devise I18n setup
**ข้อ 12:** Enum value translations
**ข้อ 13:** Email templates ในหลายภาษา
**ข้อ 14:** Test translations
**ข้อ 15:** i18n-tasks missing keys check
**ข้อ 16:** User locale preference
**ข้อ 17:** Accept-Language header detection
**ข้อ 18:** Time zone + locale
**ข้อ 19:** Flash messages ในภาษาไทย
**ข้อ 20:** SEO - hreflang tags

---

## สรุป: I18n

| Function | Usage |
|----------|-------|
| `t("key")` | Translate string |
| `l(date)` | Localize date/time |
| `I18n.locale` | Get current locale |
| `I18n.locale=` | Set locale |
| `t(".key")` | Relative key (ใน view) |
| `t("key", var: val)` | Interpolation |
| `t("key", count: n)` | Pluralization |

**Key Takeaways:**
1. ตั้งค่า `default_locale` และ `available_locales` ใน application.rb
2. แยก YAML ไฟล์ตาม scope (models, views, etc.)
3. ใช้ relative keys ใน views เพื่อความสะดวก
4. Translate error messages ของ ActiveRecord
5. ใช้ i18n-tasks ตรวจสอบ missing/unused keys

---

## Step 1191: Advanced Locale Switching

```ruby
# app/controllers/concerns/locale_setter.rb
module LocaleSetter
  extend ActiveSupport::Concern
  
  included do
    before_action :set_locale
  end
  
  private
  
  def set_locale
    I18n.locale = extract_locale || I18n.default_locale
  end
  
  def extract_locale
    locale_from_params || locale_from_subdomain ||
      locale_from_cookie || locale_from_header ||
      locale_from_user
  end
  
  def locale_from_params
    requested = params[:locale]&.to_sym
    I18n.available_locales.include?(requested) ? requested : nil
  end
  
  def locale_from_subdomain
    subdomain = request.subdomain
    case subdomain
    when 'th', 'www' then :th
    when 'en' then :en
    when 'ja' then :ja
    else nil
    end
  end
  
  def locale_from_cookie
    requested = cookies[:locale]&.to_sym
    I18n.available_locales.include?(requested) ? requested : nil
  end
  
  def locale_from_header
    request.env['HTTP_ACCEPT_LANGUAGE']
      &.scan(/[a-z]{2}(?=[-;,])/)
      &.map(&:to_sym)
      &.find { |l| I18n.available_locales.include?(l) }
  end
  
  def locale_from_user
    return nil unless respond_to?(:current_user, true)
    current_user&.preferred_locale&.to_sym
  end
  
  def default_url_options
    { locale: I18n.locale }
  end
end
```

---

## Step 1192: Locale Files Structure

```yaml
# config/locales/th.yml
th:
  # Common
  common:
    yes: "ใช่"
    no: "ไม่"
    save: "บันทึก"
    cancel: "ยกเลิก"
    delete: "ลบ"
    edit: "แก้ไข"
    back: "กลับ"
    search: "ค้นหา"
    loading: "กำลังโหลด..."
    required: "(จำเป็น)"
    
  # Navigation
  nav:
    home: "หน้าหลัก"
    about: "เกี่ยวกับเรา"
    products: "สินค้า"
    contact: "ติดต่อเรา"
    login: "เข้าสู่ระบบ"
    logout: "ออกจากระบบ"
    register: "ลงทะเบียน"
    profile: "โปรไฟล์"
    
  # Flash messages
  flash:
    success: "สำเร็จ!"
    error: "เกิดข้อผิดพลาด"
    notice: "แจ้งเตือน"
    
  # Time
  time:
    formats:
      default: "%d %B %Y %H:%M"
      short: "%d/%m/%Y"
      long: "%A, %d %B %Y"
      
  # Active Record
  activerecord:
    errors:
      messages:
        blank: "ต้องไม่ว่างเปล่า"
        taken: "ถูกใช้ไปแล้ว"
        too_short: "สั้นเกินไป (ต้องมีอย่างน้อย %{count} ตัวอักษร)"
        too_long: "ยาวเกินไป (ต้องไม่เกิน %{count} ตัวอักษร)"
        invalid: "ไม่ถูกต้อง"
        not_a_number: "ต้องเป็นตัวเลข"
        greater_than: "ต้องมากกว่า %{count}"
```

```yaml
# config/locales/en.yml
en:
  common:
    yes: "Yes"
    no: "No"
    save: "Save"
    cancel: "Cancel"
    delete: "Delete"
    edit: "Edit"
    back: "Back"
    search: "Search"
    loading: "Loading..."
    required: "(required)"
    
  nav:
    home: "Home"
    about: "About"
    products: "Products"
    contact: "Contact"
    login: "Login"
    logout: "Logout"
    register: "Register"
    profile: "Profile"
```

---

## Step 1193: Model Translations

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  # สร้าง translated attributes
  translates :name, :description
  
  # หรือใช้ Globalize gem
  # หรือ store translations ใน JSONB column
end
```

```ruby
# ใช้ JSONB สำหรับ translations (ไม่ต้องใช้ gem)
# db/migrate/xxx_add_translations_to_products.rb
class AddTranslationsToProducts < ActiveRecord::Migration[7.0]
  def change
    add_column :products, :name_translations, :jsonb, default: {}
    add_column :products, :description_translations, :jsonb, default: {}
    
    add_index :products, :name_translations, using: :gin
  end
end

# app/models/concerns/translatable.rb
module Translatable
  extend ActiveSupport::Concern
  
  included do
    def self.translates(*attrs)
      attrs.each do |attr|
        define_method("#{attr}_in") do |locale = I18n.locale|
          translations = send("#{attr}_translations") || {}
          translations[locale.to_s] || translations[I18n.default_locale.to_s] || ''
        end
        
        define_method("#{attr}_in=") do |value, locale = I18n.locale|
          translations = send("#{attr}_translations") || {}
          send("#{attr}_translations=", translations.merge(locale.to_s => value))
        end
        
        # alias สำหรับ current locale
        define_method(attr) do
          send("#{attr}_in")
        end
      end
    end
  end
end
```

---

## Step 1194: Currency และ Number Formatting

```yaml
# config/locales/th.yml
th:
  number:
    format:
      separator: "."
      delimiter: ","
      precision: 2
    currency:
      format:
        unit: "฿"
        format: "%u%n"
        separator: "."
        delimiter: ","
        precision: 2
    percentage:
      format:
        delimiter: ","
        format: "%n%"
    human:
      storage_units:
        format: "%n %u"
        units:
          byte:
            one: "ไบต์"
            other: "ไบต์"
          kb: "กิโลไบต์"
          mb: "เมกะไบต์"
          gb: "กิกะไบต์"
          tb: "เทราไบต์"
```

```ruby
# ตัวอย่างการใช้งาน
I18n.locale = :th

number_to_currency(1234567.89)
# => "฿1,234,567.89"

number_to_percentage(95.5)
# => "95.5%"

number_to_human_size(1024 * 1024)
# => "1 เมกะไบต์"

number_with_delimiter(1000000)
# => "1,000,000"
```

---

## Step 1195: Date/Time Localization

```yaml
# config/locales/th.yml
th:
  date:
    formats:
      default: "%d %B %Y"
      short: "%d/%m/%Y"
      long: "%A ที่ %d %B พ.ศ. %Y"
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
```

```ruby
# Helper สำหรับปีพุทธศักราช
module ThaiDateHelper
  def thai_year(date = Date.today)
    date.year + 543
  end
  
  def thai_date(date)
    month_names = %w[~ มกราคม กุมภาพันธ์ มีนาคม เมษายน พฤษภาคม
                     มิถุนายน กรกฎาคม สิงหาคม กันยายน ตุลาคม พฤศจิกายน ธันวาคม]
    
    "#{date.day} #{month_names[date.month]} พ.ศ. #{thai_year(date)}"
  end
end

# config/initializers/thai_date.rb
class Date
  def to_thai
    month_names = %w[~ มกราคม กุมภาพันธ์ มีนาคม เมษายน พฤษภาคม
                     มิถุนายน กรกฎาคม สิงหาคม กันยายน ตุลาคม พฤศจิกายน ธันวาคม]
    "#{day} #{month_names[month]} พ.ศ. #{year + 543}"
  end
end
```

---

## Step 1196: Pluralization Rules

```ruby
# config/initializers/i18n.rb
I18n::Backend::Simple.include(I18n::Backend::Pluralization)

# config/locales/plurals.rb
{
  th: {
    i18n: {
      plural: {
        # ภาษาไทยไม่มี plural form
        keys: [:other],
        rule: ->(n) { :other }
      }
    }
  },
  en: {
    i18n: {
      plural: {
        keys: [:one, :other],
        rule: ->(n) { n == 1 ? :one : :other }
      }
    }
  }
}
```

```yaml
# config/locales/th.yml
th:
  items:
    other: "%{count} รายการ"
    
# config/locales/en.yml
en:
  items:
    one: "%{count} item"
    other: "%{count} items"
```

---

## Step 1197: Translation Management

```ruby
# Gemfile
gem 'i18n-tasks'  # ตรวจสอบ translations ที่หายไป

# ตรวจสอบ missing translations
# bundle exec i18n-tasks missing

# ตรวจสอบ unused translations
# bundle exec i18n-tasks unused

# normalize format
# bundle exec i18n-tasks normalize

# เพิ่ม missing keys อัตโนมัติ
# bundle exec i18n-tasks add-missing
```

```ruby
# config/i18n-tasks.yml
base_locale: th
locales: [th, en]

data:
  adapter: I18n::Tasks::Data::FileSystem
  read: config/locales/%{locale}/**/*.yml
  write: config/locales/%{locale}.yml

ignore_missing:
  - '{devise,kaminari,will_paginate}.*'

ignore_unused:
  - 'activerecord.attributes.*'
```

---

## Step 1198: Testing i18n

```ruby
# spec/support/i18n_helpers.rb
module I18nHelpers
  def with_locale(locale)
    original = I18n.locale
    I18n.locale = locale
    yield
  ensure
    I18n.locale = original
  end
end

RSpec.configure do |config|
  config.include I18nHelpers
end

# spec/models/product_spec.rb
RSpec.describe Product, type: :model do
  describe "translations" do
    subject(:product) { create(:product) }
    
    it "returns name in current locale" do
      with_locale(:th) do
        product.name_translations = { 'th' => 'สินค้า', 'en' => 'Product' }
        expect(product.name).to eq('สินค้า')
      end
      
      with_locale(:en) do
        expect(product.name).to eq('Product')
      end
    end
    
    it "falls back to default locale" do
      product.name_translations = { 'th' => 'สินค้า' }
      
      with_locale(:ja) do
        expect(product.name).to eq('สินค้า')
      end
    end
  end
end

# spec/controllers/products_controller_spec.rb
RSpec.describe ProductsController, type: :controller do
  describe "locale switching" do
    it "sets locale from params" do
      get :index, params: { locale: 'en' }
      expect(I18n.locale).to eq(:en)
    end
    
    it "falls back to default locale for invalid locale" do
      get :index, params: { locale: 'xx' }
      expect(I18n.locale).to eq(I18n.default_locale)
    end
  end
end
```

---

## Step 1199: Language Switcher UI

```erb
<%# app/views/shared/_language_switcher.html.erb %>
<div class="language-switcher">
  <% I18n.available_locales.each do |locale| %>
    <%= link_to locale_name(locale),
        url_for(locale: locale),
        class: locale == I18n.locale ? 'active' : '' %>
  <% end %>
</div>
```

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def locale_name(locale)
    {
      th: "ภาษาไทย 🇹🇭",
      en: "English 🇬🇧",
      ja: "日本語 🇯🇵",
      zh: "中文 🇨🇳"
    }[locale.to_sym] || locale.to_s.upcase
  end
end
```

---

## Step 1200: Lazy Loading Translations

```yaml
# config/locales/th/products.yml
th:
  products:
    index:
      title: "รายการสินค้า"
      subtitle: "ค้นพบสินค้าทั้งหมดของเรา"
    show:
      add_to_cart: "เพิ่มในตะกร้า"
      out_of_stock: "สินค้าหมด"
    form:
      name: "ชื่อสินค้า"
      price: "ราคา"
      description: "รายละเอียด"
```

```erb
<%# ใช้ lazy loading ใน views %>
<%# app/views/products/index.html.erb %>
<h1><%= t('.title') %></h1>
<p><%= t('.subtitle') %></p>

<%# app/views/products/show.html.erb %>
<% if @product.in_stock? %>
  <%= button_to t('.add_to_cart'), cart_items_path %>
<% else %>
  <span class="badge"><%= t('.out_of_stock') %></span>
<% end %>
```

---

## แบบฝึกหัดเพิ่มเติม: i18n

### ข้อ 1: Dynamic Translation Loading

```ruby
# โหลด translations จาก database
class DatabaseTranslation < ApplicationRecord
  after_save { I18n.backend.reload! }
end

# config/initializers/i18n_backend.rb
I18n.backend = I18n::Backend::Chain.new(
  I18n::Backend::ActiveRecord.new,
  I18n::Backend::Simple.new
)
```

### ข้อ 2: Email Translations

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  def welcome_email(user)
    @user = user
    
    # ส่งอีเมลตาม locale ของ user
    I18n.with_locale(user.preferred_locale || :th) do
      mail(
        to: user.email,
        subject: t('mailer.welcome.subject', name: user.name)
      )
    end
  end
end
```

### ข้อ 3: API Locale

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      before_action :set_locale_from_header
      
      private
      
      def set_locale_from_header
        requested = request.headers['Accept-Language']&.split(',')&.first&.split('-')&.first
        I18n.locale = if requested && I18n.available_locales.include?(requested.to_sym)
                        requested.to_sym
                      else
                        I18n.default_locale
                      end
      end
    end
  end
end
```

---

## สรุป i18n

| Method | การใช้งาน |
|--------|----------|
| `t('key')` | Translate a key |
| `t('.key')` | Lazy lookup ใน current view |
| `l(date)` | Localize date/time |
| `number_to_currency` | Format currency |
| `I18n.locale=` | Set current locale |
| `I18n.with_locale` | Temporarily change locale |

**Best Practices:**
1. ใช้ Lazy lookup (`.key`) ใน views
2. แยก locale files ตาม feature/namespace
3. ตั้ง fallback locale เสมอ
4. ใช้ i18n-tasks ตรวจสอบ coverage
5. Test translations แบบ explicit locale
6. Store user locale preference ใน database
7. API: ใช้ Accept-Language header
