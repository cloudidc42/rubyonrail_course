# ตอนที่ 35: Views และ ERB Templates (Steps 756-780)

## บทนำ

Views เป็นส่วนที่แสดงผลให้ user เห็น ใน Rails Views ใช้ ERB (Embedded Ruby) เป็น template engine หลัก ซึ่งช่วยให้เราฝัง Ruby code ลงใน HTML ได้ นอกจากนี้ยังมี helper methods มากมายที่ช่วยสร้าง HTML elements ต่างๆ

---

## Step 756: ERB Syntax

### Syntax พื้นฐาน

```erb
<%# นี่คือ comment - ไม่แสดงผล %>

<% # Ruby code - ไม่แสดงผล แต่รัน %>
<% if user.admin? %>

<%= # Ruby expression - แสดงผล %>
<%= user.name %>

<%- # เหมือน <% แต่ลบ whitespace %>
<%- if user.admin? %>

<%= # แสดงผลพร้อม HTML escape (ป้องกัน XSS) %>
<%= "<script>alert('xss')</script>" %>
<!-- แสดง: &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt; -->

<%== # แสดงผลโดย raw (ไม่ escape) - ระวัง XSS! %>
<%== content.html_content %>
```

### ตัวอย่างการใช้งาน

```erb
<!-- app/views/posts/index.html.erb -->

<%# แสดงรายการ posts %>
<% @posts.each do |post| %>
  <article>
    <h2><%= post.title %></h2>
    <p><%= post.content.truncate(200) %></p>
    <small>โพสต์โดย <%= post.user.name %> เมื่อ <%= time_ago_in_words(post.created_at) %> ที่แล้ว</small>
    <%= link_to "อ่านต่อ", post_path(post), class: "btn btn-primary" %>
  </article>
<% end %>

<%# แสดงถ้าไม่มี posts %>
<% if @posts.empty? %>
  <p>ยังไม่มีบทความ</p>
<% end %>

<%# Conditional rendering %>
<%= "ยืนยันแล้ว" if user.verified? %>

<%# Ternary operator %>
<%= user.admin? ? "ผู้ดูแล" : "สมาชิก" %>

<%# Safe navigation operator %>
<%= current_user&.name || "ผู้เยี่ยมชม" %>
```

### HTML Escaping

```erb
<!-- Auto escaped (ปลอดภัย) -->
<%= @post.content %>

<!-- Raw HTML (ไม่ escaped - อันตราย) -->
<%= raw @post.content %>
<%= @post.content.html_safe %>

<!-- เช็คว่า string เป็น html_safe หรือเปล่า -->
<% @post.content.html_safe? %>

<!-- ใช้ sanitize helper (แนะนำ) -->
<%= sanitize @post.content, tags: %w[p b i a], attributes: %w[href] %>
```

---

## Step 757: Layouts

Layouts คือ template หลักที่ wrap รอบๆ content ของทุก page

### Application Layout

```erb
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html lang="th">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <title><%= content_for?(:title) ? yield(:title) : "MyApp" %></title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    
    <!-- Styles -->
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= yield :head %>
  </head>

  <body class="<%= body_class %>">
    <!-- Navigation -->
    <%= render "shared/navbar" %>
    
    <!-- Flash messages -->
    <% flash.each do |type, msg| %>
      <div class="alert alert-<%= flash_class(type) %>">
        <%= msg %>
      </div>
    <% end %>
    
    <!-- Main content -->
    <main>
      <%= yield %>
    </main>
    
    <!-- Footer -->
    <%= render "shared/footer" %>
    
    <!-- JavaScript -->
    <%= javascript_importmap_tags %>
    <%= yield :scripts %>
  </body>
</html>
```

### ใช้ Layout หลาย layout

```ruby
# Controller - กำหนด layout สำหรับทั้ง controller
class AdminController < ApplicationController
  layout "admin"
end

class ApplicationController < ActionController::Base
  layout :determine_layout
  
  private
  
  def determine_layout
    if current_user&.admin?
      "admin"
    else
      "application"
    end
  end
end

# Per-action layout
class PagesController < ApplicationController
  def landing
    render layout: "landing"
  end
  
  def home
    render layout: "application"
  end
end
```

```erb
<!-- app/views/layouts/admin.html.erb -->
<!DOCTYPE html>
<html>
  <head>
    <title>Admin - <%= content_for?(:title) ? yield(:title) : "MyApp" %></title>
    <%= stylesheet_link_tag "admin" %>
  </head>
  <body class="admin-layout">
    <%= render "admin/sidebar" %>
    <main class="admin-content">
      <%= yield %>
    </main>
  </body>
</html>
```

---

## Step 758: yield

### Main Content Yield

```erb
<!-- Layout -->
<main>
  <%= yield %>
  <!-- เนื้อหาจาก view จะมาอยู่ตรงนี้ -->
</main>
```

```erb
<!-- View -->
<h1>หน้าหลัก</h1>
<p>เนื้อหาของ view นี้</p>
<!-- ทั้งหมดนี้จะถูก yield ใน layout -->
```

---

## Step 759: content_for / yield :section

```erb
<!-- Layout -->
<!DOCTYPE html>
<html>
  <head>
    <title><%= yield :title %></title>
    <%= yield :meta %>
    <%= yield :styles %>
  </head>
  <body>
    <%= yield %>
    <%= yield :scripts %>
  </body>
</html>
```

```erb
<!-- View -->
<% content_for :title do %>
  หน้าบทความ - MyApp
<% end %>

<% content_for :meta do %>
  <meta name="description" content="<%= @post.excerpt %>">
  <meta property="og:title" content="<%= @post.title %>">
<% end %>

<% content_for :styles do %>
  <%= stylesheet_link_tag "posts" %>
<% end %>

<% content_for :scripts do %>
  <script>
    // JavaScript เฉพาะ page นี้
    initPostEditor();
  </script>
<% end %>

<!-- Main content -->
<h1><%= @post.title %></h1>
<div class="content">
  <%= @post.content %>
</div>
```

### content_for? - ตรวจสอบว่ามี content หรือเปล่า

```erb
<!-- Layout -->
<% if content_for?(:sidebar) %>
  <div class="layout-with-sidebar">
    <main><%= yield %></main>
    <aside><%= yield :sidebar %></aside>
  </div>
<% else %>
  <main class="full-width"><%= yield %></main>
<% end %>
```

---

## Step 760: Partials

Partials คือ template ย่อยๆ ที่ reuse ได้ ชื่อไฟล์ขึ้นต้นด้วย `_`

### สร้าง Partial

```erb
<!-- app/views/posts/_post.html.erb -->
<article class="post" id="<%= dom_id(post) %>">
  <h2><%= link_to post.title, post_path(post) %></h2>
  <p class="meta">
    โดย <%= post.user.name %> | 
    <%= time_ago_in_words(post.created_at) %> ที่แล้ว |
    <%= post.comments.count %> ความคิดเห็น
  </p>
  <p><%= post.content.truncate(150) %></p>
  <%= link_to "อ่านต่อ", post_path(post) %>
</article>
```

### render partial

```erb
<!-- app/views/posts/index.html.erb -->
<% @posts.each do |post| %>
  <%= render partial: "post", locals: { post: post } %>
  <!-- หรือ shorthand -->
  <%= render "post", post: post %>
  <!-- หรือ สั้นกว่า เมื่อ variable name เหมือนกัน -->
  <%= render post %>
<% end %>
```

---

## Step 761: render collection

```erb
<!-- Render collection - เร็วกว่า loop -->
<%= render partial: "post", collection: @posts %>

<!-- Shorthand -->
<%= render @posts %>
<!-- Rails จะหา partial ชื่อ _post.html.erb โดยอัตโนมัติ -->

<!-- ด้วย spacer template -->
<%= render partial: "post", collection: @posts, spacer_template: "post_divider" %>
<!-- จะ render _post_divider.html.erb ระหว่าง posts แต่ละอัน -->

<!-- ด้วย as option -->
<%= render partial: "product", collection: @items, as: :item %>
<!-- ใน partial จะใช้ item แทน product -->
```

```erb
<!-- app/views/posts/_post.html.erb -->
<!-- ตัวแปร post_counter ใช้ได้อัตโนมัติ -->
<div class="post post-<%= post_counter + 1 %>">
  <h3><%= post.title %></h3>
</div>
```

---

## Step 762: locals ใน Partials

```erb
<!-- render ด้วย locals -->
<%= render "form", post: @post, submit_text: "บันทึก" %>

<!-- app/views/posts/_form.html.erb -->
<% submit_text ||= "บันทึก" %>  <!-- default value -->

<%= form_with model: post do |form| %>
  <%= form.text_field :title %>
  <%= form.submit submit_text %>
<% end %>
```

### local_assigns

```erb
<!-- ตรวจสอบว่า local variable ถูกส่งมาหรือเปล่า -->
<% if local_assigns[:show_author] %>
  <p>โดย <%= post.user.name %></p>
<% end %>

<% unless local_assigns.key?(:show_footer) %>
  <!-- show footer by default -->
<% end %>
```

---

## Step 763: Helper Methods Overview

### link_to

```erb
<!-- Basic link -->
<%= link_to "หน้าหลัก", root_path %>
<!-- => <a href="/">หน้าหลัก</a> -->

<!-- Link ด้วย model -->
<%= link_to "ดูบทความ", @post %>
<!-- => <a href="/posts/1">ดูบทความ</a> -->

<!-- Link ด้วย options -->
<%= link_to "Google", "https://google.com", target: "_blank", rel: "noopener" %>

<!-- Link ด้วย block -->
<%= link_to post_path(@post) do %>
  <img src="<%= @post.image_url %>" alt="<%= @post.title %>">
  <span><%= @post.title %></span>
<% end %>

<!-- Delete link -->
<%= link_to "ลบ", post_path(@post), 
    data: { turbo_method: :delete, turbo_confirm: "แน่ใจ?" },
    class: "btn btn-danger" %>

<!-- Link ด้วย class และ id -->
<%= link_to "คลิก", root_path, class: "btn btn-primary", id: "main-btn" %>

<!-- Link disabled -->
<%= link_to "ปิดการใช้งาน", root_path, class: "disabled" %>
```

### image_tag

```erb
<%= image_tag "logo.png" %>
<!-- => <img src="/assets/logo.png"> -->

<%= image_tag "logo.png", alt: "Logo", width: 100, height: 50 %>
<%= image_tag "logo.png", class: "img-fluid", id: "logo" %>

<!-- Active Storage -->
<%= image_tag @user.avatar %>
<%= image_tag @post.image.variant(resize_to_limit: [300, 300]) %>

<!-- ถ้าไม่มีรูป ใช้ default -->
<%= image_tag @user.avatar.attached? ? @user.avatar : "default-avatar.png" %>
```

### button_to

```erb
<!-- สร้าง form ที่มีปุ่มเดียว -->
<%= button_to "Publish", publish_post_path(@post), method: :post %>
<!-- => <form action="/posts/1/publish" method="post"><button>Publish</button></form> -->

<%= button_to "Delete", post_path(@post), 
    method: :delete,
    data: { confirm: "แน่ใจ?" },
    class: "btn btn-danger" %>
```

---

## Step 764: Number Helpers

```ruby
# ต้อง include ใน helper module หรือใช้ใน view โดยตรง
include ActionView::Helpers::NumberHelper
```

```erb
<!-- number_to_currency -->
<%= number_to_currency(1234.56) %>
<!-- => $1,234.56 -->

<%= number_to_currency(1234.56, unit: "฿", format: "%u%n") %>
<!-- => ฿1,234.56 -->

<%= number_to_currency(1234.56, unit: "บาท", separator: ".", delimiter: ",") %>
<!-- => 1,234.56บาท -->

<!-- number_with_delimiter -->
<%= number_with_delimiter(1234567) %>
<!-- => 1,234,567 -->

<%= number_with_delimiter(1234567.89) %>
<!-- => 1,234,567.89 -->

<!-- number_with_precision -->
<%= number_with_precision(3.14159, precision: 2) %>
<!-- => 3.14 -->

<%= number_with_precision(1234.5678, precision: 2, delimiter: ",") %>
<!-- => 1,234.57 -->

<!-- number_to_percentage -->
<%= number_to_percentage(75.3) %>
<!-- => 75.300% -->

<%= number_to_percentage(75.3, precision: 1) %>
<!-- => 75.3% -->

<!-- number_to_phone -->
<%= number_to_phone(0812345678) %>
<!-- => 081-234-5678 -->

<%= number_to_phone("0812345678", area_code: true) %>

<!-- number_to_human -->
<%= number_to_human(1234567) %>
<!-- => 1.23 Million -->

<%= number_to_human(1234567, locale: :th) %>

<!-- number_to_human_size (file size) -->
<%= number_to_human_size(1234567) %>
<!-- => 1.18 MB -->
```

---

## Step 765: Date Helpers

```erb
<!-- time_ago_in_words -->
<%= time_ago_in_words(@post.created_at) %>
<!-- => "about 2 hours" -->

<%= time_ago_in_words(3.days.ago) %>
<!-- => "3 days" -->

<!-- distance_of_time_in_words -->
<%= distance_of_time_in_words(Time.now, 1.hour.from_now) %>
<!-- => "about 1 hour" -->

<%= distance_of_time_in_words(Time.now, 2.weeks.from_now) %>
<!-- => "14 days" -->

<!-- strftime - format date/time -->
<%= @post.created_at.strftime("%d/%m/%Y") %>
<!-- => "15/01/2024" -->

<%= @post.created_at.strftime("%d %B %Y เวลา %H:%M น.") %>
<!-- => "15 มกราคม 2024 เวลา 10:30 น." (ต้องตั้ง locale) -->

<!-- l helper (localize) -->
<%= l @post.created_at %>
<%= l @post.created_at, format: :long %>
<%= l @post.created_at, format: :short %>
<%= l @post.created_at.to_date %>
```

### Configure Localization

```yaml
# config/locales/th.yml
th:
  date:
    formats:
      default: "%d/%m/%Y"
      short: "%d %b"
      long: "%d %B %Y"
  time:
    formats:
      default: "%d/%m/%Y %H:%M"
      short: "%d %b %H:%M"
      long: "%d %B %Y เวลา %H:%M น."
```

---

## Step 766: Text Helpers

```erb
<!-- truncate -->
<%= truncate("ข้อความยาวมากๆ ที่ต้องการตัด", length: 30) %>
<!-- => "ข้อความยาวมากๆ ที่ต้องการตั..." -->

<%= truncate("Long text here...", length: 50, omission: " [...]") %>
<!-- => "Long text here... [...]" -->

<!-- pluralize -->
<%= pluralize(1, "comment") %>
<!-- => "1 comment" -->

<%= pluralize(5, "comment") %>
<!-- => "5 comments" -->

<%= pluralize(0, "ความคิดเห็น", plural: "ความคิดเห็น") %>
<!-- => "0 ความคิดเห็น" -->

<!-- simple_format - แปลง \n เป็น <p> tags -->
<%= simple_format(@post.content) %>
<!-- "First para\n\nSecond para" => "<p>First para</p>\n\n<p>Second para</p>" -->

<%= simple_format(@post.content, {}, wrapper_tag: "div") %>

<!-- highlight -->
<%= highlight("Ruby on Rails is great", "Rails") %>
<!-- => "Ruby on <mark>Rails</mark> is great" -->

<%= highlight(@post.content, params[:q]) %>

<!-- word_wrap -->
<%= word_wrap("Long text...", line_width: 80) %>

<!-- strip_tags -->
<%= strip_tags("<p>Hello <b>World</b></p>") %>
<!-- => "Hello World" -->

<!-- strip_links -->
<%= strip_links('<a href="http://example.com">Click here</a>') %>
<!-- => "Click here" -->

<!-- sanitize -->
<%= sanitize(@post.content, tags: %w[p b i em strong a], 
             attributes: %w[href class]) %>

<!-- excerpt -->
<%= excerpt("Hello World, how are you?", "World", radius: 5) %>
<!-- => "...World, ho..." -->
```

---

## Step 767: Asset Helpers

```erb
<!-- stylesheet_link_tag -->
<%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
<!-- => <link rel="stylesheet" href="/assets/application.css"> -->

<%= stylesheet_link_tag "admin", media: "all" %>
<%= stylesheet_link_tag "print", media: "print" %>

<!-- javascript_include_tag -->
<%= javascript_include_tag "application", "data-turbo-track": "reload", defer: true %>
<!-- => <script src="/assets/application.js" defer="defer"></script> -->

<!-- javascript_importmap_tags (Rails 7) -->
<%= javascript_importmap_tags %>

<!-- image_tag -->
<%= image_tag "logo.png", alt: "Logo" %>
<%= image_tag "icons/user.svg", width: 24, height: 24 %>

<!-- image_path / image_url -->
<% src = image_path("logo.png") %>
<% src = image_url("logo.png") %>  <!-- full URL -->

<!-- asset_path / asset_url -->
<% path = asset_path("manifest.json") %>
<% url = asset_url("manifest.json") %>  <!-- full URL -->

<!-- favicon_link_tag -->
<%= favicon_link_tag "favicon.ico" %>
<%= favicon_link_tag "icon.png", rel: "apple-touch-icon", type: "image/png" %>

<!-- auto_discovery_link_tag (RSS/Atom) -->
<%= auto_discovery_link_tag :rss, posts_path(format: :rss), 
    title: "My Blog RSS Feed" %>
```

---

## Step 768: DOM Helpers

```erb
<!-- tag helper (Rails 7) -->
<%= tag.div "Hello", class: "greeting" %>
<!-- => <div class="greeting">Hello</div> -->

<%= tag.p %>
<!-- => <p></p> -->

<%= tag.p "ข้อความ", class: "lead" %>
<!-- => <p class="lead">ข้อความ</p> -->

<!-- ด้วย block -->
<%= tag.div class: "container" do %>
  <p>เนื้อหา</p>
<% end %>

<!-- content_tag (เก่ากว่า แต่ยังใช้งานได้) -->
<%= content_tag :div, "Hello", class: "greeting" %>

<!-- dom_id -->
<%= dom_id(@post) %>
<!-- => "post_1" -->

<div id="<%= dom_id(@post) %>">...</div>
<!-- => <div id="post_1">...</div> -->

<!-- dom_class -->
<div class="<%= dom_class(@post) %>">...</div>
<!-- => <div class="post">...</div> -->

<!-- tag สำหรับ custom elements -->
<%= tag.turbo_frame id: "posts-list" do %>
  <%= render @posts %>
<% end %>
```

---

## Step 769: Custom Helpers

### สร้าง Helper

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  # Format date ภาษาไทย
  def thai_date(date)
    return "" unless date
    date.strftime("%d %B %Y")
  end
  
  # Flash message CSS class
  def flash_class(type)
    {
      notice: "success",
      success: "success",
      alert: "danger",
      error: "danger",
      warning: "warning",
      info: "info"
    }[type.to_sym] || "secondary"
  end
  
  # Page title
  def page_title(title = nil)
    if title
      content_for(:title) { "#{title} | MyApp" }
    else
      content_for?(:title) ? content_for(:title) : "MyApp"
    end
  end
  
  # Active link
  def active_link_to(text, path, options = {})
    options[:class] = "#{options[:class]} active" if current_page?(path)
    link_to text, path, options
  end
  
  # Bootstrap badge
  def status_badge(status)
    classes = {
      "active" => "badge bg-success",
      "inactive" => "badge bg-secondary",
      "pending" => "badge bg-warning",
      "banned" => "badge bg-danger"
    }
    content_tag :span, status.humanize, class: classes[status] || "badge bg-secondary"
  end
end

# app/helpers/posts_helper.rb
module PostsHelper
  def reading_time(content)
    words_per_minute = 200
    words = content.split.size
    minutes = (words / words_per_minute.to_f).ceil
    "#{minutes} นาที"
  end
  
  def post_status_color(post)
    case post.status
    when "published" then "text-success"
    when "draft"     then "text-secondary"
    when "archived"  then "text-muted"
    else "text-dark"
    end
  end
  
  def post_card(post, &block)
    content_tag :div, class: "card post-card" do
      concat content_tag(:div, class: "card-body") {
        concat content_tag(:h5, post.title, class: "card-title")
        concat content_tag(:p, post.content.truncate(100), class: "card-text")
        concat capture(&block) if block_given?
      }
    end
  end
end
```

### ใช้ Custom Helpers ใน Views

```erb
<!-- ใน views (helpers load อัตโนมัติ) -->
<p>วันที่: <%= thai_date(@post.created_at) %></p>

<h1><%= page_title(@post.title) %></h1>

<%= active_link_to "หน้าหลัก", root_path, class: "nav-link" %>

<%= status_badge(@post.status) %>

<p>เวลาอ่าน: <%= reading_time(@post.content) %></p>
```

---

## Step 770: View Testing

### Minitest View Tests

```ruby
# test/helpers/posts_helper_test.rb
require "test_helper"

class PostsHelperTest < ActionView::TestCase
  test "reading_time returns correct estimate" do
    content = "word " * 400  # 400 words
    result = reading_time(content)
    assert_equal "2 นาที", result
  end
  
  test "status_badge returns correct classes" do
    result = status_badge("active")
    assert_includes result, "bg-success"
    assert_includes result, "active"
  end
end

# test/views/posts/index_test.rb (View tests)
require "test_helper"

class PostsIndexViewTest < ActionView::TestCase
  def setup
    @posts = [
      Post.new(id: 1, title: "First Post", content: "Content 1"),
      Post.new(id: 2, title: "Second Post", content: "Content 2")
    ]
  end
  
  test "renders all posts" do
    assign(:posts, @posts)
    render template: "posts/index"
    
    assert_select "h2", text: "First Post"
    assert_select "h2", text: "Second Post"
  end
  
  test "shows post count" do
    assign(:posts, @posts)
    render template: "posts/index"
    
    assert_select ".post-count", text: /2/
  end
end
```

### RSpec View Tests

```ruby
# spec/views/posts/index.html.erb_spec.rb
require "rails_helper"

RSpec.describe "posts/index", type: :view do
  before do
    @posts = [
      assign(:post1, create(:post, title: "First Post")),
      assign(:post2, create(:post, title: "Second Post"))
    ]
    assign(:posts, @posts)
  end
  
  it "renders a list of posts" do
    render
    expect(rendered).to match(/First Post/)
    expect(rendered).to match(/Second Post/)
  end
  
  it "renders links to each post" do
    render
    @posts.each do |post|
      expect(rendered).to have_selector("a[href='#{post_path(post)}']")
    end
  end
end
```

---

## ตัวอย่าง View ที่สมบูรณ์

### Blog Post Show Page

```erb
<!-- app/views/posts/show.html.erb -->

<% content_for :title, @post.title %>

<% content_for :meta do %>
  <meta name="description" content="<%= @post.excerpt %>">
  <meta property="og:title" content="<%= @post.title %>">
  <meta property="og:description" content="<%= @post.excerpt %>">
  <% if @post.cover_image.attached? %>
    <meta property="og:image" content="<%= url_for(@post.cover_image) %>">
  <% end %>
<% end %>

<article class="post">
  <!-- Header -->
  <header class="post-header">
    <% if @post.cover_image.attached? %>
      <%= image_tag @post.cover_image, class: "cover-image", alt: @post.title %>
    <% end %>
    
    <h1 class="post-title"><%= @post.title %></h1>
    
    <div class="post-meta">
      <%= image_tag @post.user.avatar, class: "avatar", width: 32 if @post.user.avatar.attached? %>
      <span>โดย <strong><%= @post.user.name %></strong></span>
      <time datetime="<%= @post.published_at.iso8601 %>">
        <%= l @post.published_at, format: :long %>
      </time>
      <span><%= reading_time(@post.content) %></span>
    </div>
    
    <% if @post.tags.any? %>
      <div class="tags">
        <% @post.tags.each do |tag| %>
          <%= link_to tag.name, tag_path(tag), class: "badge bg-secondary" %>
        <% end %>
      </div>
    <% end %>
  </header>
  
  <!-- Content -->
  <div class="post-content">
    <%= simple_format(@post.content) %>
  </div>
  
  <!-- Footer -->
  <footer class="post-footer">
    <!-- Author info -->
    <div class="author-card">
      <h4>เกี่ยวกับผู้เขียน</h4>
      <p><%= @post.user.bio %></p>
    </div>
    
    <!-- Actions (admin only) -->
    <% if current_user&.admin? || @post.user == current_user %>
      <div class="post-actions">
        <%= link_to "แก้ไข", edit_post_path(@post), class: "btn btn-outline-primary" %>
        <%= link_to "ลบ", post_path(@post), 
            data: { turbo_method: :delete, turbo_confirm: "ต้องการลบบทความนี้?" },
            class: "btn btn-outline-danger" %>
      </div>
    <% end %>
  </footer>
</article>

<!-- Comments Section -->
<section class="comments-section" id="comments">
  <h3>ความคิดเห็น (<%= @post.comments.count %>)</h3>
  
  <% if @comments.any? %>
    <%= render @comments %>
  <% else %>
    <p class="text-muted">ยังไม่มีความคิดเห็น เป็นคนแรกที่แสดงความคิดเห็น!</p>
  <% end %>
  
  <!-- Comment form -->
  <% if user_signed_in? %>
    <div class="comment-form-section">
      <h4>แสดงความคิดเห็น</h4>
      <%= render "comments/form", post: @post, comment: Comment.new %>
    </div>
  <% else %>
    <p>
      <%= link_to "เข้าสู่ระบบ", login_path %> เพื่อแสดงความคิดเห็น
    </p>
  <% end %>
</section>

<!-- Related Posts -->
<% if @related_posts.any? %>
  <section class="related-posts">
    <h3>บทความที่เกี่ยวข้อง</h3>
    <div class="row">
      <% @related_posts.each do |related| %>
        <div class="col-md-4">
          <%= render "posts/card", post: related %>
        </div>
      <% end %>
    </div>
  </section>
<% end %>
```

---

## แบบฝึกหัด (Steps 771-780)

### แบบฝึกหัดที่ 1
เขียน ERB code แสดงชื่อ user ถ้า login อยู่ ถ้าไม่ login แสดง "ผู้เยี่ยมชม"

**เฉลย:**
```erb
<%= current_user&.name || "ผู้เยี่ยมชม" %>
```

### แบบฝึกหัดที่ 2
สร้าง layout ที่มี navigation, flash messages, main content และ footer

**เฉลย:**
```erb
<!DOCTYPE html>
<html lang="th">
<head>
  <title><%= yield :title || "MyApp" %></title>
  <%= stylesheet_link_tag "application" %>
</head>
<body>
  <%= render "shared/navbar" %>
  
  <% flash.each do |type, msg| %>
    <div class="alert alert-<%= flash_class(type) %>"><%= msg %></div>
  <% end %>
  
  <main class="container">
    <%= yield %>
  </main>
  
  <%= render "shared/footer" %>
  <%= javascript_importmap_tags %>
</body>
</html>
```

### แบบฝึกหัดที่ 3
สร้าง partial `_post_card.html.erb` ที่แสดง post title, excerpt, และ link

**เฉลย:**
```erb
<!-- app/views/posts/_post_card.html.erb -->
<div class="card">
  <div class="card-body">
    <h5 class="card-title"><%= post.title %></h5>
    <p class="card-text"><%= post.content.truncate(100) %></p>
    <%= link_to "อ่านต่อ", post_path(post), class: "btn btn-primary" %>
  </div>
</div>
```

### แบบฝึกหัดที่ 4
ใช้ render collection แสดง posts ทั้งหมดโดยใช้ partial

**เฉลย:**
```erb
<%= render partial: "post_card", collection: @posts, as: :post %>
<!-- หรือ shorthand -->
<%= render @posts %>
```

### แบบฝึกหัดที่ 5
ใช้ content_for เพิ่ม page-specific JavaScript ใน view

**เฉลย:**
```erb
<% content_for :scripts do %>
  <script>
    document.addEventListener("DOMContentLoaded", function() {
      initMap();
    });
  </script>
<% end %>
```

### แบบฝึกหัดที่ 6
สร้าง helper method `thai_baht(amount)` ที่แสดงตัวเลขในรูปแบบสกุลเงินบาท

**เฉลย:**
```ruby
def thai_baht(amount)
  number_to_currency(amount, unit: "฿", format: "%u%n", 
                    delimiter: ",", separator: ".")
end
```

### แบบฝึกหัดที่ 7
ใช้ time_ago_in_words แสดงเวลาที่โพสต์

**เฉลย:**
```erb
<small>โพสต์เมื่อ <%= time_ago_in_words(@post.created_at) %> ที่แล้ว</small>
```

### แบบฝึกหัดที่ 8
สร้าง helper `active_nav_link` ที่เพิ่ม class "active" เมื่อ current page

**เฉลย:**
```ruby
def active_nav_link(text, path, options = {})
  options[:class] = [options[:class], ("active" if current_page?(path))].compact.join(" ")
  link_to text, path, options
end
```

### แบบฝึกหัดที่ 9
แสดงรูปภาพ post ถ้ามี ถ้าไม่มีแสดง placeholder

**เฉลย:**
```erb
<% if @post.image.attached? %>
  <%= image_tag @post.image.variant(resize_to_limit: [800, 400]), 
      class: "post-image", alt: @post.title %>
<% else %>
  <%= image_tag "post-placeholder.jpg", class: "post-image", alt: "No image" %>
<% end %>
```

### แบบฝึกหัดที่ 10
สร้าง partial `_pagination.html.erb` ที่แสดง pagination controls

**เฉลย:**
```erb
<!-- app/views/shared/_pagination.html.erb -->
<% if total_pages > 1 %>
  <nav aria-label="Page navigation">
    <ul class="pagination">
      <% if current_page > 1 %>
        <li class="page-item">
          <%= link_to "ก่อนหน้า", url_for(page: current_page - 1), 
              class: "page-link" %>
        </li>
      <% end %>
      
      <% (1..total_pages).each do |p| %>
        <li class="page-item <%= 'active' if p == current_page %>">
          <%= link_to p, url_for(page: p), class: "page-link" %>
        </li>
      <% end %>
      
      <% if current_page < total_pages %>
        <li class="page-item">
          <%= link_to "ถัดไป", url_for(page: current_page + 1), 
              class: "page-link" %>
        </li>
      <% end %>
    </ul>
  </nav>
<% end %>
```

### แบบฝึกหัดที่ 11
แสดง reading time ของ post โดยคำนวณจาก word count

**เฉลย:**
```ruby
# app/helpers/posts_helper.rb
def reading_time(content)
  words = content.to_s.split.size
  minutes = [(words / 200.0).ceil, 1].max
  "ใช้เวลาอ่านประมาณ #{minutes} นาที"
end
```

```erb
<p class="reading-time"><%= reading_time(@post.content) %></p>
```

### แบบฝึกหัดที่ 12
ใช้ sanitize helper ที่อนุญาตเฉพาะ p, b, i tags

**เฉลย:**
```erb
<div class="content">
  <%= sanitize @post.content, tags: %w[p b i em strong], 
      attributes: %w[class] %>
</div>
```

### แบบฝึกหัดที่ 13
สร้าง shared navbar partial ที่มี active state

**เฉลย:**
```erb
<!-- app/views/shared/_navbar.html.erb -->
<nav class="navbar">
  <ul class="nav">
    <li class="<%= 'active' if current_page?(root_path) %>">
      <%= link_to "หน้าหลัก", root_path %>
    </li>
    <li class="<%= 'active' if controller_name == 'posts' %>">
      <%= link_to "บทความ", posts_path %>
    </li>
    <% if user_signed_in? %>
      <li><%= link_to current_user.name, profile_path %></li>
      <li><%= link_to "ออกจากระบบ", logout_path, data: { turbo_method: :delete } %></li>
    <% else %>
      <li><%= link_to "เข้าสู่ระบบ", login_path %></li>
    <% end %>
  </ul>
</nav>
```

### แบบฝึกหัดที่ 14
ใช้ number_to_currency สำหรับ product price

**เฉลย:**
```erb
<p class="price">
  <% if @product.on_sale? %>
    <s class="original-price">
      <%= number_to_currency(@product.original_price, unit: "฿", format: "%u%n") %>
    </s>
    <strong class="sale-price text-danger">
      <%= number_to_currency(@product.sale_price, unit: "฿", format: "%u%n") %>
    </strong>
  <% else %>
    <%= number_to_currency(@product.price, unit: "฿", format: "%u%n") %>
  <% end %>
</p>
```

### แบบฝึกหัดที่ 15
สร้าง view ที่แสดงตาราง users พร้อม action links

**เฉลย:**
```erb
<!-- app/views/admin/users/index.html.erb -->
<h1>ผู้ใช้ทั้งหมด</h1>

<table class="table">
  <thead>
    <tr>
      <th>ID</th>
      <th>ชื่อ</th>
      <th>อีเมล</th>
      <th>วันที่สมัคร</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody>
    <% @users.each do |user| %>
      <tr id="<%= dom_id(user) %>">
        <td><%= user.id %></td>
        <td><%= user.name %></td>
        <td><%= user.email %></td>
        <td><%= l user.created_at, format: :short %></td>
        <td>
          <%= link_to "ดู", admin_user_path(user), class: "btn btn-sm btn-info" %>
          <%= link_to "แก้ไข", edit_admin_user_path(user), class: "btn btn-sm btn-warning" %>
          <%= link_to "ลบ", admin_user_path(user), 
              data: { turbo_method: :delete, turbo_confirm: "ลบผู้ใช้ #{user.name}?" },
              class: "btn btn-sm btn-danger" %>
        </td>
      </tr>
    <% end %>
  </tbody>
</table>
```

### แบบฝึกหัดที่ 16
สร้าง helper ที่ generate breadcrumb navigation

**เฉลย:**
```ruby
# app/helpers/application_helper.rb
def breadcrumbs(*items)
  content_tag :nav, aria: { label: "breadcrumb" } do
    content_tag :ol, class: "breadcrumb" do
      items.map.with_index do |(text, path), index|
        if index == items.length - 1
          content_tag :li, text, class: "breadcrumb-item active"
        else
          content_tag :li, class: "breadcrumb-item" do
            link_to text, path
          end
        end
      end.join.html_safe
    end
  end
end
```

```erb
<%= breadcrumbs ["หน้าหลัก", root_path], ["บทความ", posts_path], [@post.title] %>
```

### แบบฝึกหัดที่ 17
render partial ด้วย spacer template

**เฉลย:**
```erb
<!-- app/views/posts/index.html.erb -->
<%= render partial: "post", collection: @posts, spacer_template: "divider" %>

<!-- app/views/posts/_divider.html.erb -->
<hr class="post-divider">
```

### แบบฝึกหัดที่ 18
สร้าง view ที่ใช้ tag helper สร้าง custom elements

**เฉลย:**
```erb
<%= tag.section class: "posts-grid" do %>
  <% @posts.each do |post| %>
    <%= tag.article class: "post-card", id: dom_id(post) do %>
      <%= tag.h2 post.title %>
      <%= tag.p post.content.truncate(150) %>
      <%= link_to "อ่านต่อ", post_path(post) %>
    <% end %>
  <% end %>
<% end %>
```

### แบบฝึกหัดที่ 19
เขียน view test ที่ตรวจสอบว่า post title แสดงใน template

**เฉลย:**
```ruby
# spec/views/posts/show.html.erb_spec.rb
require "rails_helper"

RSpec.describe "posts/show", type: :view do
  let(:post) { create(:post, title: "Test Post") }
  
  before do
    assign(:post, post)
    assign(:comments, [])
  end
  
  it "displays the post title" do
    render
    expect(rendered).to have_selector("h1", text: "Test Post")
  end
end
```

### แบบฝึกหัดที่ 20
สร้าง helper ที่ format phone number ไทย

**เฉลย:**
```ruby
def thai_phone(number)
  cleaned = number.to_s.gsub(/\D/, "")
  if cleaned.length == 10
    "#{cleaned[0..2]}-#{cleaned[3..5]}-#{cleaned[6..]}"
  else
    number
  end
end
```

### แบบฝึกหัดที่ 21
ใช้ pluralize แสดงจำนวน comments อย่างถูกต้อง

**เฉลย:**
```erb
<p><%= pluralize(@post.comments.count, "comment", locale: :en) %></p>
<!-- หรือภาษาไทย -->
<p>มี <%= @post.comments.count %> ความคิดเห็น</p>
```

### แบบฝึกหัดที่ 22
สร้าง alert partial ที่ support หลาย types

**เฉลย:**
```erb
<!-- app/views/shared/_flash_messages.html.erb -->
<% flash.each do |type, message| %>
  <% css_class = { notice: "success", alert: "danger", warning: "warning", info: "info" }[type.to_sym] || "secondary" %>
  <div class="alert alert-<%= css_class %> alert-dismissible fade show" role="alert">
    <%= message %>
    <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
  </div>
<% end %>
```

### แบบฝึกหัดที่ 23
เขียน layout ที่ condition-based ระหว่าง admin และ user

**เฉลย:**
```ruby
# app/controllers/application_controller.rb
layout :choose_layout

private

def choose_layout
  if controller_path.start_with?("admin/")
    "admin"
  elsif devise_controller?
    "auth"
  else
    "application"
  end
end
```

### แบบฝึกหัดที่ 24
ใช้ excerpt helper แสดง search result snippet

**เฉลย:**
```erb
<!-- Search results -->
<% @results.each do |post| %>
  <div class="search-result">
    <h3><%= link_to post.title, post_path(post) %></h3>
    <p>...<%= excerpt(post.content, params[:q], radius: 100) %>...</p>
    <p><%= highlight(post.content.truncate(200), params[:q]) %></p>
  </div>
<% end %>
```

### แบบฝึกหัดที่ 25
สร้าง complete posts/index view ที่มี search, filter, และ pagination

**เฉลย:**
```erb
<!-- app/views/posts/index.html.erb -->
<% content_for :title, "บทความทั้งหมด" %>

<div class="posts-header">
  <h1>บทความทั้งหมด</h1>
  
  <!-- Search form -->
  <%= form_with url: posts_path, method: :get, class: "search-form" do |f| %>
    <%= f.text_field :q, placeholder: "ค้นหา...", value: params[:q], class: "form-control" %>
    <%= f.submit "ค้นหา", class: "btn btn-primary" %>
  <% end %>
  
  <!-- Category filter -->
  <div class="category-filter">
    <%= link_to "ทั้งหมด", posts_path, class: "btn #{params[:category].blank? ? 'btn-primary' : 'btn-outline-primary'}" %>
    <% Category.all.each do |cat| %>
      <%= link_to cat.name, posts_path(category: cat.id),
          class: "btn #{params[:category] == cat.id.to_s ? 'btn-primary' : 'btn-outline-secondary'}" %>
    <% end %>
  </div>
</div>

<!-- Posts list -->
<% if @posts.any? %>
  <div class="posts-grid">
    <%= render @posts %>
  </div>
  
  <!-- Pagination -->
  <div class="pagination-wrapper">
    <%= paginate @posts %>
  </div>
<% else %>
  <div class="empty-state">
    <p>ไม่พบบทความ</p>
    <% if params[:q].present? %>
      <p>ลองค้นหาด้วยคำอื่น</p>
      <%= link_to "ดูบทความทั้งหมด", posts_path, class: "btn btn-secondary" %>
    <% end %>
  </div>
<% end %>
```

---

## สรุป

ใน Views และ ERB เราได้เรียนรู้:

1. **ERB Syntax** - `<%= %>`, `<% %>`, `<%# %>` และ HTML escaping
2. **Layouts** - application.html.erb, multiple layouts
3. **yield** - main content yield
4. **content_for** - page-specific content
5. **Partials** - reusable view fragments
6. **render collection** - efficient rendering
7. **locals** - passing data to partials
8. **Number helpers** - currency, percentage, file size
9. **Date helpers** - time_ago_in_words, l()
10. **Text helpers** - truncate, highlight, sanitize
11. **Asset helpers** - stylesheet, javascript, image tags
12. **DOM helpers** - tag, content_tag, dom_id
13. **Custom helpers** - สร้าง helper methods เอง
14. **View testing** - Minitest และ RSpec
