# ตอนที่ 35: Views และ ERB (Steps 756-780)

## บทนำ

Views เป็นชั้น "V" ใน MVC pattern ที่รับผิดชอบการแสดงผลให้กับผู้ใช้ ใน Rails เราใช้ ERB (Embedded Ruby) เป็น template engine หลัก ซึ่งทำให้เราสามารถฝัง Ruby code เข้าไปใน HTML ได้ นอกจาก ERB ยังมี Haml และ Slim แต่ ERB คือ default ของ Rails

---

## ขั้นตอนที่ 756: ERB Syntax ทุกรูปแบบ

### ERB คืออะไร?

ERB ย่อมาจาก Embedded Ruby เป็น template language ที่ให้เราใส่ Ruby code ลงใน HTML ไฟล์ที่มีนามสกุล `.html.erb`

### รูปแบบ ERB Tags

```erb
<%# 1. Execute tag - รัน Ruby code แต่ไม่แสดงผล %>
<% ruby_code_here %>

<%# 2. Expression tag - รัน Ruby code และแสดงผล (escaped) %>
<%= ruby_expression %>

<%# 3. Comment tag - comment ที่ไม่แสดงใน HTML output %>
<%# this is a comment %>

<%# 4. Unescaped output - แสดงผล HTML โดยไม่ escape (ระวัง XSS!) %>
<%== raw_html_string %>

<%# หรือใช้ raw() helper %>
<%= raw(html_string) %>
```

### ตัวอย่างการใช้งาน ERB

```erb
<%# app/views/posts/show.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <title><%= @post.title %></title>
</head>
<body>
  <h1><%= @post.title %></h1>

  <%# แสดงวันที่สร้าง %>
  <p>สร้างเมื่อ: <%= @post.created_at.strftime("%d/%m/%Y") %></p>

  <%# แสดงเนื้อหา (safe HTML) %>
  <div class="content">
    <%= @post.body %>
  </div>

  <%# Conditional display %>
  <% if @post.published? %>
    <span class="badge bg-success">เผยแพร่แล้ว</span>
  <% else %>
    <span class="badge bg-warning">ร่าง</span>
  <% end %>

  <%# Loop through comments %>
  <% @post.comments.each do |comment| %>
    <div class="comment">
      <strong><%= comment.author %></strong>
      <p><%= comment.body %></p>
    </div>
  <% end %>

  <%# Inline condition %>
  <%= "มี #{@post.comments.count} ความคิดเห็น" if @post.comments.any? %>
</body>
</html>
```

### HTML Escaping

ERB จะ escape HTML characters อัตโนมัติเพื่อป้องกัน XSS:

```erb
<%# @name = "<script>alert('XSS')</script>" %>

<%# Escaped (safe) - default behavior %>
<%= @name %>
<%# Output: &lt;script&gt;alert('XSS')&lt;/script&gt; %>

<%# Unescaped (dangerous!) %>
<%= raw(@name) %>
<%# หรือ %>
<%== @name %>
<%# Output: <script>alert('XSS')</script> - อันตราย! %>

<%# ใช้ html_safe เมื่อมั่นใจว่า content ปลอดภัย %>
<%= "<b>Bold</b>".html_safe %>
```

---

## ขั้นตอนที่ 757: Layouts และการทำงาน

### Layouts คืออะไร?

Layout คือ template หลักที่ครอบ view ทั้งหมด ทำให้ไม่ต้องเขียน HTML skeleton ซ้ำในทุก view

### โครงสร้าง Default Layout

```
app/
  views/
    layouts/
      application.html.erb    <- default layout
      admin.html.erb          <- layout สำหรับ admin
      mailer.html.erb         <- layout สำหรับ email
      mailer.text.erb
```

### application.html.erb

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for?(:title) ? yield(:title) : "MyApp" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <meta name="description" content="<%= yield(:description) %>">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%# Stylesheets %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>

    <%# Extra head content from views %>
    <%= yield :head %>
  </head>

  <body class="<%= body_class %>">
    <%# Navigation partial %>
    <%= render "shared/navbar" %>

    <%# Flash messages %>
    <%= render "shared/flash_messages" %>

    <%# Main content area %>
    <main class="container my-4">
      <%= yield %>
    </main>

    <%# Footer %>
    <%= render "shared/footer" %>

    <%# JavaScript %>
    <%= javascript_importmap_tags %>

    <%# Extra scripts from views %>
    <%= yield :scripts %>
  </body>
</html>
```

### กำหนด Layout สำหรับ Controller

```ruby
# app/controllers/admin/base_controller.rb
class Admin::BaseController < ApplicationController
  # ใช้ layout admin สำหรับ controller นี้
  layout "admin"

  # หรือกำหนดแบบ dynamic
  layout :resolve_layout

  private

  def resolve_layout
    if current_user&.admin?
      "admin"
    else
      "application"
    end
  end
end

# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  # ใช้ layout ต่างกันตาม action
  layout "wide", only: [:index]
  layout "narrow", only: [:show, :edit]

  # ไม่ใช้ layout เลย
  def print
    render layout: false
  end
end
```

### Custom Layout สำหรับแต่ละ Render

```ruby
# ใน action
def show
  @post = Post.find(params[:id])
  render layout: "print"
end

# หรือไม่ใช้ layout
def export
  render layout: false
end
```

---

## ขั้นตอนที่ 758: yield และ content_for

### yield พื้นฐาน

`yield` ใน layout จะแทนที่ด้วย content ของ view

```erb
<%# Layout %>
<div class="container">
  <%= yield %>  <%# แทนที่ด้วย view content %>
</div>
```

### Named yields ด้วย content_for

```erb
<%# ใน view - กำหนด content สำหรับ named yield %>
<% content_for :title do %>
  บทความ: <%= @post.title %>
<% end %>

<% content_for :head do %>
  <meta name="description" content="<%= @post.excerpt %>">
  <link rel="canonical" href="<%= post_url(@post) %>">
<% end %>

<% content_for :scripts do %>
  <script>
    console.log("Page loaded: <%= @post.id %>");
  </script>
<% end %>

<% content_for :sidebar do %>
  <div class="sidebar">
    <h3>หมวดหมู่</h3>
    <%= render "categories/list" %>
  </div>
<% end %>

<%# เนื้อหาหลัก (ไม่อยู่ใน content_for block = ส่วน yield หลัก) %>
<article>
  <h1><%= @post.title %></h1>
  <p><%= @post.body %></p>
</article>
```

```erb
<%# ใน layout - ใช้ content ที่กำหนดไว้ %>
<!DOCTYPE html>
<html>
<head>
  <title><%= yield(:title) || "Default Title" %></title>
  <%= yield :head %>
</head>
<body>
  <div class="row">
    <div class="col-md-9">
      <%= yield %>           <%# main content %>
    </div>
    <div class="col-md-3">
      <%= yield :sidebar %>  <%# sidebar content %>
    </div>
  </div>
  <%= yield :scripts %>
</body>
</html>
```

### content_for? - ตรวจสอบว่ามี content หรือไม่

```erb
<%# ใน layout %>
<% if content_for?(:sidebar) %>
  <div class="col-md-3">
    <%= yield :sidebar %>
  </div>
<% end %>

<%# หรือกำหนด default content %>
<% unless content_for?(:title) %>
  <% content_for :title, "My Application" %>
<% end %>
```

---

## ขั้นตอนที่ 759: Partials (render, locals, collection)

### Partial คืออะไร?

Partial คือ view ย่อยที่แยกออกมาเพื่อ reuse ชื่อไฟล์ขึ้นต้นด้วย underscore `_`

### โครงสร้าง Partials

```
app/views/
  posts/
    _post.html.erb       <- partial สำหรับ post
    _form.html.erb       <- partial สำหรับ form
    index.html.erb
    show.html.erb
  shared/
    _navbar.html.erb     <- shared partial
    _flash_messages.html.erb
    _footer.html.erb
  comments/
    _comment.html.erb
```

### render Partial พื้นฐาน

```erb
<%# render partial file (ชื่อ partial ไม่มี underscore) %>
<%= render "form" %>

<%# render partial จาก path อื่น %>
<%= render "shared/navbar" %>

<%# render partial พร้อม specify ชัดเจน %>
<%= render partial: "post" %>
```

### Passing Variables ด้วย locals

```erb
<%# ส่ง local variables ไปยัง partial %>
<%= render "post", post: @post %>

<%# ส่งหลาย variables %>
<%= render "card", title: "สวัสดี", body: "เนื้อหา", color: "blue" %>

<%# ใน partial _card.html.erb - ใช้ local variables %>
<div class="card" style="border-color: <%= color %>">
  <h3><%= title %></h3>
  <p><%= body %></p>
</div>
```

### render Collection

```erb
<%# แทนที่การวน loop %>
<%# แบบเดิม (ไม่ดี) %>
<% @posts.each do |post| %>
  <%= render "post", post: post %>
<% end %>

<%# แบบที่ดีกว่า - render collection %>
<%= render @posts %>
<%# Rails จะหา partial _post.html.erb อัตโนมัติ %>
<%# และส่ง post: post ให้อัตโนมัติ %>

<%# กำหนด partial ชัดเจน %>
<%= render partial: "post", collection: @posts %>

<%# ใน _post.html.erb - ใช้ตัวแปร post อัตโนมัติ %>
<article>
  <h2><%= post.title %></h2>
  <p><%= post.excerpt %></p>
</article>
```

### Collection Counter

```erb
<%# Rails ส่ง counter ให้อัตโนมัติ: [partial_name]_counter %>
<%# ใน _post.html.erb %>
<article>
  <small><%# post_counter เริ่มจาก 1 %><%= post_counter %>.</small>
  <h2><%= post.title %></h2>
</article>

<%# ใน view %>
<%= render partial: "post", collection: @posts %>
```

### Spacer Template

```erb
<%# เพิ่ม separator ระหว่าง collection items %>
<%= render partial: "post",
           collection: @posts,
           spacer_template: "post_divider" %>

<%# _post_divider.html.erb %>
<hr class="my-3">
```

### as option - เปลี่ยนชื่อตัวแปรใน partial

```erb
<%# ใน view %>
<%= render partial: "post", collection: @articles, as: :post %>

<%# ใน _post.html.erb - ยังใช้ post ได้ถึงแม้จะวน @articles %>
<h2><%= post.title %></h2>
```

### locals ใน Collection

```erb
<%# ส่ง local variables เพิ่มเติมใน collection %>
<%= render partial: "post",
           collection: @posts,
           locals: { show_author: true, highlight: "ruby" } %>

<%# ใน _post.html.erb %>
<article>
  <h2><%= post.title %></h2>
  <% if show_author %>
    <p>โดย: <%= post.author %></p>
  <% end %>
</article>
```

### Render กับ Object

```erb
<%# Rails ค้นหา partial ตาม class ของ object %>
<%= render @post %>
<%# ค้นหา posts/_post.html.erb %>

<%# Collection ของ mixed types %>
<%= render [post, comment, photo] %>
<%# Rails จะ render partial ที่เหมาะสมสำหรับแต่ละ type %>
```

---

## ขั้นตอนที่ 760: Helpers พื้นฐาน

### link_to

```erb
<%# Basic link %>
<%= link_to "หน้าแรก", root_path %>

<%# Link ไปยัง resource %>
<%= link_to "ดูบทความ", post %>
<%= link_to "ดูบทความ", post_path(@post) %>

<%# Link พร้อม class และ attributes %>
<%= link_to "สร้างบทความ", new_post_path, class: "btn btn-primary" %>

<%# Link พร้อม data attributes (สำหรับ Turbo) %>
<%= link_to "ลบ", post_path(@post),
    data: { turbo_method: :delete, turbo_confirm: "แน่ใจหรือไม่?" },
    class: "btn btn-danger" %>

<%# Link แบบ block (เนื้อหาซับซ้อน) %>
<%= link_to post_path(@post) do %>
  <div class="card">
    <h3><%= @post.title %></h3>
    <p><%= @post.excerpt %></p>
  </div>
<% end %>

<%# Link เปิด tab ใหม่ %>
<%= link_to "เปิด", @post.url, target: "_blank", rel: "noopener" %>

<%# Link ไปยัง anchor %>
<%= link_to "ดูความคิดเห็น", post_path(@post, anchor: "comments") %>

<%# Link พร้อม query params %>
<%= link_to "กรอง", posts_path(category: "ruby", sort: "recent") %>
```

### image_tag

```erb
<%# Basic image %>
<%= image_tag "logo.png" %>

<%# Image พร้อม attributes %>
<%= image_tag "avatar.jpg",
    alt: "รูปโปรไฟล์",
    width: 100,
    height: 100,
    class: "rounded-circle" %>

<%# Image จาก URL %>
<%= image_tag "https://example.com/image.jpg", alt: "External image" %>

<%# Lazy loading %>
<%= image_tag "photo.jpg", loading: "lazy", alt: "Photo" %>

<%# Responsive image %>
<%= image_tag "hero.jpg",
    class: "img-fluid",
    srcset: "hero@2x.jpg 2x" %>

<%# Image จาก Active Storage %>
<%= image_tag @user.avatar, class: "avatar" if @user.avatar.attached? %>
```

### tag helper

```erb
<%# สร้าง HTML tag %>
<%= tag.div "เนื้อหา", class: "container" %>

<%# สร้าง tag แบบ block %>
<%= tag.div class: "card" do %>
  <p>เนื้อหาใน card</p>
<% end %>

<%# span พร้อม class %>
<%= tag.span "ใหม่", class: "badge bg-primary" %>

<%# br tag %>
<%= tag.br %>

<%# hr tag %>
<%= tag.hr class: "my-3" %>

<%# tag แบบ content_tag (เก่า - ยังใช้ได้) %>
<%= content_tag :div, "เนื้อหา", class: "container" %>
```

---

## ขั้นตอนที่ 761: Number Helpers

```ruby
# config/locales/th.yml
th:
  number:
    currency:
      format:
        unit: "฿"
        separator: "."
        delimiter: ","
        precision: 2
```

```erb
<%# number_with_delimiter - เพิ่ม separator %>
<%= number_with_delimiter(1234567) %>
<%# => "1,234,567" %>

<%= number_with_delimiter(1234567.89, delimiter: ",", separator: ".") %>
<%# => "1,234,567.89" %>

<%# number_with_precision - กำหนดทศนิยม %>
<%= number_with_precision(3.14159, precision: 2) %>
<%# => "3.14" %>

<%= number_with_precision(1234567.891, precision: 2, delimiter: ",") %>
<%# => "1,234,567.89" %>

<%# number_to_currency - แปลงเป็นสกุลเงิน %>
<%= number_to_currency(1234.50) %>
<%# => "$1,234.50" (default USD) %>

<%= number_to_currency(1234.50, unit: "฿", format: "%u%n") %>
<%# => "฿1,234.50" %>

<%= number_to_currency(1234.50, unit: "บาท", separator: ".", delimiter: ",") %>
<%# => "$1,234.50บาท" %>

<%# number_to_percentage - แปลงเป็น % %>
<%= number_to_percentage(80) %>
<%# => "80.000%" %>

<%= number_to_percentage(80.5, precision: 1) %>
<%# => "80.5%" %>

<%# number_to_human_size - ขนาดไฟล์ %>
<%= number_to_human_size(1234567) %>
<%# => "1.18 MB" %>

<%= number_to_human_size(1234567890) %>
<%# => "1.15 GB" %>

<%# number_to_human - ตัวเลขอ่านง่าย %>
<%= number_to_human(1234567) %>
<%# => "1.23 Million" %>

<%= number_to_human(1234567, locale: :th) %>

<%# number_to_phone - รูปแบบเบอร์โทร %>
<%= number_to_phone(0812345678) %>
<%# => "081-234-5678" %>

<%= number_to_phone(0812345678, delimiter: "-", area_code: false) %>
```

---

## ขั้นตอนที่ 762: Date และ Time Helpers

```erb
<%# time_ago_in_words - เวลาที่ผ่านไป %>
<%= time_ago_in_words(@post.created_at) %>
<%# => "about 2 hours ago" %>

<%# distance_of_time_in_words %>
<%= distance_of_time_in_words(@post.created_at, Time.now) %>
<%# => "about 2 hours" %>

<%# distance_of_time_in_words_to_now (alias) %>
<%= distance_of_time_in_words_to_now(@post.created_at) %>

<%# time_tag - HTML5 time element %>
<%= time_tag @post.created_at %>
<%# => <time datetime="2024-01-15T10:30:00+07:00">Mon, 15 Jan 2024 10:30:00 +0700</time> %>

<%= time_tag @post.created_at, class: "timestamp" do %>
  <%= @post.created_at.strftime("%d %B %Y") %>
<% end %>

<%# strftime สำหรับ format เอง %>
<%= @post.created_at.strftime("%d/%m/%Y %H:%M") %>
<%# => "15/01/2024 10:30" %>

<%# Thai date formatting %>
<%= l @post.created_at, format: :long %>
<%# ต้องตั้งค่า locale ใน config/locales/th.yml %>

<%# ใช้ strftime สำหรับภาษาไทย %>
<% thai_months = %w[ม.ค. ก.พ. มี.ค. เม.ย. พ.ค. มิ.ย. ก.ค. ส.ค. ก.ย. ต.ค. พ.ย. ธ.ค.] %>
<%= "#{@post.created_at.day} #{thai_months[@post.created_at.month - 1]} #{@post.created_at.year + 543}" %>
```

---

## ขั้นตอนที่ 763: Text Helpers

```erb
<%# truncate - ตัดข้อความ %>
<%= truncate(@post.body, length: 100) %>
<%# => "เนื้อหาบทความ..." %>

<%= truncate(@post.body, length: 100, omission: " [อ่านต่อ]") %>

<%= truncate(@post.body, length: 100, separator: " ") %>
<%# ตัดที่คำ ไม่ตัดกลางคำ %>

<%# excerpt - ดึงข้อความรอบ keyword %>
<%= excerpt(@post.body, "Ruby", radius: 100) %>
<%# => "...เรียนรู้ Ruby on Rails..." %>

<%# pluralize - single/plural %>
<%= pluralize(@post.comments.count, "comment") %>
<%# => "0 comments", "1 comment", "2 comments" %>

<%= pluralize(3, "บทความ") %>
<%# => "3 บทความ" %>

<%# cycle - สลับค่า %>
<% @posts.each do |post| %>
  <tr class="<%= cycle("odd", "even") %>">
    <td><%= post.title %></td>
  </tr>
<% end %>

<%# highlight - เน้น keyword %>
<%= highlight(@post.body, params[:q]) %>
<%# => ห่อ keyword ด้วย <mark> %>

<%= highlight(@post.body, "Ruby",
    highlighter: '<span class="highlight">\1</span>') %>

<%# simple_format - แปลง newlines เป็น <p> %>
<%= simple_format(@post.body) %>
<%# แปลง "\n\n" เป็น </p><p> %>

<%# word_wrap - ตัดบรรทัดตามความกว้าง %>
<%= word_wrap(@long_text, line_width: 80) %>

<%# strip_tags - ลบ HTML tags %>
<%= strip_tags(@post.body_html) %>

<%# strip_links - ลบ links แต่คง text %>
<%= strip_links(@post.body_html) %>

<%# sanitize - อนุญาต tags บางอย่าง %>
<%= sanitize(@post.body_html, tags: %w[b i em strong a]) %>

<%# auto_link - แปลง URL เป็น link อัตโนมัติ %>
<%# ต้องใช้ gem rails_autolink %>
```

---

## ขั้นตอนที่ 764: Asset Helpers

```erb
<%# stylesheet_link_tag %>
<%= stylesheet_link_tag "application", media: "all",
    "data-turbo-track": "reload" %>

<%= stylesheet_link_tag "print", media: "print" %>

<%# javascript_include_tag (ไม่ใช้ใน Rails 7 importmap) %>
<%= javascript_include_tag "application", defer: true %>

<%# javascript_importmap_tags (Rails 7+) %>
<%= javascript_importmap_tags %>

<%# image_path - path ไปยัง image asset %>
<%= image_path("logo.png") %>
<%# => "/assets/logo-abc123.png" %>

<%# asset_path %>
<%= asset_path("document.pdf") %>

<%# stylesheet_path %>
<%= stylesheet_path("custom") %>

<%# javascript_path %>
<%= javascript_path("app") %>

<%# audio_tag %>
<%= audio_tag "podcast.mp3", controls: true %>

<%# video_tag %>
<%= video_tag "intro.mp4",
    controls: true,
    width: "100%",
    poster: image_path("thumbnail.jpg") %>

<%# favicon_link_tag %>
<%= favicon_link_tag "favicon.ico" %>
<%= favicon_link_tag "icon.png", rel: "apple-touch-icon", type: "image/png" %>
```

---

## ขั้นตอนที่ 765: Form Helpers Overview

### form_with Helper

```erb
<%# Form สำหรับ Model (model-backed) %>
<%= form_with(model: @post) do |form| %>
  <div class="mb-3">
    <%= form.label :title, "หัวข้อ" %>
    <%= form.text_field :title, class: "form-control" %>
  </div>

  <div class="mb-3">
    <%= form.label :body, "เนื้อหา" %>
    <%= form.text_area :body, class: "form-control", rows: 5 %>
  </div>

  <div class="mb-3">
    <%= form.label :published, "เผยแพร่" %>
    <%= form.check_box :published %>
  </div>

  <%= form.submit "บันทึก", class: "btn btn-primary" %>
<% end %>

<%# Form ที่ไม่ผูกกับ Model %>
<%= form_with(url: search_path, method: :get) do |form| %>
  <%= form.text_field :q, placeholder: "ค้นหา..." %>
  <%= form.submit "ค้นหา" %>
<% end %>
```

### Input Types ทั้งหมด

```erb
<%= form_with(model: @user) do |f| %>
  <%# Text inputs %>
  <%= f.text_field :name %>
  <%= f.email_field :email %>
  <%= f.password_field :password %>
  <%= f.number_field :age %>
  <%= f.tel_field :phone %>
  <%= f.url_field :website %>
  <%= f.search_field :query %>

  <%# Date/Time inputs %>
  <%= f.date_field :birth_date %>
  <%= f.time_field :event_time %>
  <%= f.datetime_local_field :scheduled_at %>
  <%= f.month_field :month %>
  <%= f.week_field :week %>

  <%# Other inputs %>
  <%= f.range_field :rating, min: 1, max: 10 %>
  <%= f.color_field :theme_color %>
  <%= f.hidden_field :role, value: "user" %>
  <%= f.file_field :avatar %>

  <%# Multi-line %>
  <%= f.text_area :bio, rows: 4 %>

  <%# Boolean %>
  <%= f.check_box :subscribe_newsletter %>
  <%= f.radio_button :gender, "male" %>
  <%= f.radio_button :gender, "female" %>

  <%# Select %>
  <%= f.select :country, [["ไทย", "TH"], ["USA", "US"]] %>
<% end %>
```

---

## ขั้นตอนที่ 766: Custom Helpers

### สร้าง Helper Module

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  # แปลง boolean เป็นสัญลักษณ์
  def yes_no(value)
    value ? "✓ ใช่" : "✗ ไม่"
  end

  # สร้าง badge
  def status_badge(status)
    colors = {
      active: "success",
      inactive: "secondary",
      pending: "warning",
      blocked: "danger"
    }
    color = colors[status.to_sym] || "light"
    content_tag(:span, status.capitalize,
                class: "badge bg-#{color}")
  end

  # Title helper
  def page_title(title = nil)
    base_title = "MyApp"
    if title.present?
      "#{title} | #{base_title}"
    else
      base_title
    end
  end

  # Active nav link
  def nav_link(text, path, **options)
    active = current_page?(path)
    options[:class] = [options[:class], "active"].compact.join(" ") if active
    link_to text, path, **options
  end

  # Format currency in Thai Baht
  def baht(amount)
    number_to_currency(amount, unit: "฿", format: "%u%n",
                       separator: ".", delimiter: ",",
                       precision: 2)
  end

  # Thai date format
  def thai_date(date)
    return "-" if date.nil?
    thai_months = %w[มกราคม กุมภาพันธ์ มีนาคม เมษายน พฤษภาคม มิถุนายน
                     กรกฎาคม สิงหาคม กันยายน ตุลาคม พฤศจิกายน ธันวาคม]
    "#{date.day} #{thai_months[date.month - 1]} #{date.year + 543}"
  end

  # Truncate with link to full content
  def truncate_with_link(text, path, length: 150)
    if text.length > length
      "#{truncate(text, length: length)} #{link_to('อ่านต่อ...', path)}".html_safe
    else
      text
    end
  end

  # Flash message class
  def flash_class(type)
    {
      notice: "alert-success",
      alert: "alert-danger",
      warning: "alert-warning",
      info: "alert-info"
    }[type.to_sym] || "alert-info"
  end
end
```

### ใช้ Helper ใน View

```erb
<%# ใช้ custom helpers %>
<p>สถานะ: <%= yes_no(@user.active?) %></p>
<p>สถานะบัญชี: <%= status_badge(@user.status) %></p>
<title><%= page_title(@post&.title) %></title>
<p>ราคา: <%= baht(@product.price) %></p>
<p>วันที่: <%= thai_date(@post.created_at) %></p>
```

### Helper สำหรับ Specific Controller

```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  # Helper เฉพาะ posts controller
  def post_meta(post)
    parts = []
    parts << "โดย #{post.author.name}"
    parts << "#{post.comments.count} ความคิดเห็น"
    parts << time_ago_in_words(post.created_at) + " ที่แล้ว"
    parts.join(" · ")
  end

  def category_options
    Category.all.map { |c| [c.name, c.id] }
  end
end
```

---

## ขั้นตอนที่ 767: Testing Views

### View Specs ด้วย RSpec

```ruby
# spec/views/posts/show.html.erb_spec.rb
require "rails_helper"

RSpec.describe "posts/show", type: :view do
  let(:post) do
    create(:post,
           title: "Test Post",
           body: "Test body content",
           published: true)
  end

  before do
    assign(:post, post)
    render
  end

  it "displays the post title" do
    expect(rendered).to include("Test Post")
  end

  it "displays the post body" do
    expect(rendered).to include("Test body content")
  end

  it "shows published badge" do
    expect(rendered).to have_css(".badge", text: "เผยแพร่แล้ว")
  end

  it "does not show draft badge" do
    expect(rendered).not_to have_css(".badge-warning")
  end

  context "when post is not published" do
    let(:post) { create(:post, published: false) }

    it "shows draft badge" do
      expect(rendered).to have_css(".badge", text: "ร่าง")
    end
  end
end
```

### Request Specs (Controller + View integration)

```ruby
# spec/requests/posts_spec.rb
require "rails_helper"

RSpec.describe "Posts", type: :request do
  describe "GET /posts/:id" do
    let!(:post) { create(:post, title: "My Post") }

    it "returns success" do
      get post_path(post)
      expect(response).to have_http_status(:success)
    end

    it "displays post title" do
      get post_path(post)
      expect(response.body).to include("My Post")
    end

    it "renders the show template" do
      get post_path(post)
      expect(response).to render_template(:show)
    end
  end
end
```

### System Specs ด้วย Capybara

```ruby
# spec/system/posts_spec.rb
require "rails_helper"

RSpec.describe "Posts", type: :system do
  describe "viewing a post" do
    let!(:post) { create(:post, title: "Test Post", body: "Test body") }

    it "displays post content" do
      visit post_path(post)
      expect(page).to have_content("Test Post")
      expect(page).to have_content("Test body")
    end

    it "has working navigation" do
      visit post_path(post)
      click_link "กลับ"
      expect(current_path).to eq(posts_path)
    end
  end
end
```

---

## ขั้นตอนที่ 768: Advanced ERB Techniques

### Capture Helper

```erb
<%# capture - เก็บ ERB output เป็น variable %>
<% @post_card = capture do %>
  <div class="card">
    <h3><%= @post.title %></h3>
    <p><%= @post.excerpt %></p>
  </div>
<% end %>

<%# ใช้ทีหลัง %>
<%= @post_card %>
```

### render กับ block

```erb
<%# Passing block to partial (Rails 7+) %>
<%= render("card", title: "My Card") do %>
  <p>นี่คือเนื้อหาใน card</p>
<% end %>

<%# ใน _card.html.erb %>
<div class="card">
  <h3><%= title %></h3>
  <div class="card-body">
    <%= yield %>
  </div>
</div>
```

### Inline Partials ด้วย render inline

```ruby
# ใน controller (ไม่แนะนำสำหรับ production)
def show
  render inline: "<p><%= @post.title %></p>"
end
```

---

## ขั้นตอนที่ 769: i18n ใน Views

### การใช้ I18n

```erb
<%# ใช้ t() helper สำหรับ translation %>
<h1><%= t("posts.index.title") %></h1>
<%= t("hello", name: @user.name) %>

<%# Lazy lookup (ไม่ต้องพิมพ์ path เต็ม) %>
<%# ใน views/posts/index.html.erb %>
<h1><%= t(".title") %></h1>
<%# ค้นหา: en.views.posts.index.title %>

<%# pluralization %>
<p><%= t("posts.count", count: @posts.count) %></p>
```

```yaml
# config/locales/th.yml
th:
  hello: "สวัสดี %{name}"
  posts:
    index:
      title: "บทความทั้งหมด"
    count:
      zero: "ไม่มีบทความ"
      one: "1 บทความ"
      other: "%{count} บทความ"
  activerecord:
    models:
      post: "บทความ"
    attributes:
      post:
        title: "หัวข้อ"
        body: "เนื้อหา"
```

---

## ขั้นตอนที่ 770: Turbo Frame Views

### Turbo Frame ใน Views

```erb
<%# app/views/posts/index.html.erb %>
<%= turbo_frame_tag "posts_list" do %>
  <% @posts.each do |post| %>
    <div class="post-item">
      <h3><%= link_to post.title, post %></h3>
      <p><%= post.excerpt %></p>
    </div>
  <% end %>
<% end %>

<%# แสดง pagination นอก frame %>
<%= paginate @posts %>

<%# View ที่ใช้ Turbo Stream update %>
<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.prepend "posts_list" do %>
  <%= render @post %>
<% end %>

<%= turbo_stream.update "flash" do %>
  <%= render "shared/flash_messages" %>
<% end %>
```

---

## ขั้นตอนที่ 771: View Components

### ViewComponent (gem)

```ruby
# Gemfile
gem "view_component"

# app/components/card_component.rb
class CardComponent < ViewComponent::Base
  def initialize(title:, color: "white")
    @title = title
    @color = color
  end
end
```

```erb
<%# app/components/card_component.html.erb %>
<div class="card" style="background: <%= @color %>">
  <div class="card-header">
    <h3><%= @title %></h3>
  </div>
  <div class="card-body">
    <%= content %>
  </div>
</div>
```

```erb
<%# ใช้ใน view %>
<%= render(CardComponent.new(title: "My Card", color: "#f0f0f0")) do %>
  <p>เนื้อหาของ card</p>
<% end %>
```

---

## ขั้นตอนที่ 772: Caching Views

### Fragment Caching

```erb
<%# cache block - cache HTML fragment %>
<% cache @post do %>
  <div class="post">
    <h2><%= @post.title %></h2>
    <p><%= @post.body %></p>
    <%= render @post.comments %>
  </div>
<% end %>

<%# Cache พร้อม expire time %>
<% cache @post, expires_in: 1.hour do %>
  <%= render "post_card", post: @post %>
<% end %>

<%# Cache Collection %>
<% cache @posts do %>
  <%= render @posts %>
<% end %>

<%# Russian Doll Caching - nested cache %>
<% cache @post do %>
  <h2><%= @post.title %></h2>
  <% @post.comments.each do |comment| %>
    <% cache comment do %>
      <p><%= comment.body %></p>
    <% end %>
  <% end %>
<% end %>
```

---

## แบบฝึกหัดตอนที่ 35 (25 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** สร้าง layout `app/views/layouts/blog.html.erb` ที่มี navigation bar, sidebar, และ footer โดยใช้ `yield` สำหรับ content หลัก และ `yield :sidebar` สำหรับ sidebar

**ข้อ 2:** สร้าง view `posts/show.html.erb` ที่:
- แสดง title, body, created_at
- ใช้ `content_for :title` เพื่อกำหนด title ของหน้า
- แสดง status badge (เผยแพร่/ร่าง)

**ข้อ 3:** สร้าง partial `_post_card.html.erb` ที่รับ `post` เป็น local variable แล้ว render collection ของ posts จาก index view

**ข้อ 4:** ใช้ `link_to` สร้าง:
- Link ไปหน้า show
- Link ลบ post ด้วย confirm dialog
- Link ที่เปิด tab ใหม่

**ข้อ 5:** แสดงราคาสินค้าด้วย `number_to_currency` ในรูปแบบ `฿1,234.50`

**ข้อ 6:** แสดงเวลาที่โพสต์ด้วย `time_ago_in_words` และ `time_tag`

**ข้อ 7:** ใช้ `truncate` ตัดเนื้อหา 200 ตัวอักษร พร้อม "อ่านต่อ..." link

**ข้อ 8:** สร้าง flash messages partial ที่แสดง success, error, warning messages ด้วยสีที่เหมาะสม

**ข้อ 9:** สร้าง navigation partial ที่ highlight active menu item

**ข้อ 10:** สร้าง custom helper `format_phone(number)` ที่แสดงเบอร์โทรรูปแบบ `081-234-5678`

### ระดับกลาง

**ข้อ 11:** สร้าง partial `_pagination.html.erb` ที่แสดง pagination links (previous, next, page numbers)

**ข้อ 12:** สร้าง view ที่ใช้ `cycle` สลับสี row ในตาราง posts

**ข้อ 13:** ใช้ `content_for :head` เพิ่ม Open Graph meta tags ใน post show view

**ข้อ 14:** สร้าง `status_badge` helper ที่รับ status (active/inactive/pending/blocked) แล้วแสดง badge ด้วยสีที่เหมาะสม

**ข้อ 15:** สร้าง view สำหรับแสดงรูปภาพ gallery โดยใช้ `image_tag` และ `link_to`

**ข้อ 16:** ใช้ `highlight` helper เพื่อ highlight คำที่ค้นหาใน search results

**ข้อ 17:** สร้าง `nav_link` helper ที่เพิ่ม class "active" เมื่ออยู่ที่หน้านั้น

**ข้อ 18:** สร้าง partial collection rendering สำหรับ comments พร้อม spacer template

**ข้อ 19:** เขียน view spec (RSpec) สำหรับ `posts/show.html.erb` ที่ test การแสดงผลต่างๆ

**ข้อ 20:** สร้าง `thai_date` helper และ `buddhist_year` helper

### ระดับสูง

**ข้อ 21:** Implement fragment caching สำหรับ post index page ด้วย Russian Doll pattern

**ข้อ 22:** สร้าง Layout Inheritance: `layouts/admin.html.erb` extends `layouts/application.html.erb`

**ข้อ 23:** สร้าง view component (ถ้าติดตั้ง gem) สำหรับ `AlertComponent` ที่รับ type และ message

**ข้อ 24:** สร้าง helper ที่ generate breadcrumb navigation จาก nested resources (categories > posts > comments)

**ข้อ 25:** เขียน system spec (Capybara) สำหรับ post creation flow: ไปหน้า new, กรอกข้อมูล, submit, verify ผลลัพธ์

---

## สรุปตอนที่ 35

ในตอนนี้เราเรียนรู้เกี่ยวกับ Views และ ERB อย่างละเอียด:

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| ERB Syntax | `<% %>`, `<%= %>`, `<%# %>`, `<%== %>` |
| Layouts | application.html.erb, named yields, layout selection |
| yield/content_for | Named slots สำหรับ layout |
| Partials | render, locals, collection, spacer |
| Helpers | link_to, image_tag, tag, number, date, text |
| Custom Helpers | ApplicationHelper, module methods |
| Testing | View specs, request specs, system specs |
| Advanced | i18n, caching, Turbo, ViewComponent |

Views เป็นชั้นที่ผู้ใช้มองเห็นโดยตรง การเขียน view ที่ดีควร:
- **แยก concern** ด้วย partials และ helpers
- **ไม่ใส่ business logic** ในขั้น view
- **ใช้ caching** เพื่อ performance
- **Test** ให้ครอบคลุมทุก scenario

ตอนถัดไป: **ตอนที่ 36** - Models และ Active Record
