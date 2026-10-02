# ตอนที่ 40: Forms (Steps 881-905)

## บทนำ

Forms เป็นส่วนสำคัญในการรับข้อมูลจาก user ใน Rails มี Form Helpers ที่ช่วยสร้าง HTML forms ที่ integrate กับ Active Record models และมี CSRF protection อัตโนมัติ

---

## Step 881: form_with (Rails 7 way)

`form_with` เป็น form helper หลักใน Rails 7 ที่รวมความสามารถของ `form_for` และ `form_tag` เข้าด้วยกัน

```erb
<!-- Form สำหรับ model (create/update) -->
<%= form_with model: @post do |form| %>
  <%= form.text_field :title %>
  <%= form.submit %>
<% end %>

<!-- HTML ที่สร้าง -->
<form action="/posts" method="post">
  <input type="hidden" name="authenticity_token" value="...">
  <input type="text" name="post[title]" id="post_title">
  <input type="submit" value="Create Post">
</form>

<!-- สำหรับ edit (auto detect จาก model) -->
<% @post = Post.find(1) %>
<%= form_with model: @post do |form| %>
  <!-- action="/posts/1" method="post" (hidden input _method=patch) -->
<% end %>
```

### form_with options

```erb
<!-- URL แบบ explicit -->
<%= form_with url: posts_path do |form| %>
<%= form_with url: "/posts" do |form| %>

<!-- Method -->
<%= form_with model: @post, method: :post do |form| %>
<%= form_with url: posts_path, method: :delete do |form| %>

<!-- HTML attributes -->
<%= form_with model: @post, 
    class: "post-form needs-validation",
    id: "new-post-form",
    data: { controller: "form" } do |form| %>

<!-- Scope - เปลี่ยน prefix ของ params -->
<%= form_with model: @user, scope: :registration do |form| %>
<!-- สร้าง params[:registration][:name] แทน params[:user][:name] -->

<!-- Local: true - ไม่ใช้ Turbo (Rails 7) -->
<%= form_with model: @post, local: true do |form| %>

<!-- Multipart (file upload) -->
<%= form_with model: @post, multipart: true do |form| %>
```

---

## Step 882: text_field, email_field, password_field, number_field, url_field

### text_field

```erb
<%= form_with model: @user do |form| %>
  <!-- Basic text field -->
  <%= form.text_field :name %>
  <!-- => <input type="text" name="user[name]" id="user_name"> -->
  
  <!-- ด้วย options -->
  <%= form.text_field :name, 
      placeholder: "กรอกชื่อ",
      class: "form-control",
      id: "user_name_field",
      maxlength: 100,
      autofocus: true,
      autocomplete: "name",
      required: true %>
  
  <!-- ด้วย value -->
  <%= form.text_field :name, value: @user.name %>
  
  <!-- Disabled -->
  <%= form.text_field :name, disabled: true %>
  
  <!-- Readonly -->
  <%= form.text_field :username, readonly: true %>
<% end %>
```

### email_field

```erb
<%= form.email_field :email,
    placeholder: "your@email.com",
    class: "form-control",
    autocomplete: "email",
    required: true %>
<!-- => <input type="email" name="user[email]"> -->
```

### password_field

```erb
<%= form.password_field :password,
    placeholder: "รหัสผ่าน (อย่างน้อย 8 ตัวอักษร)",
    class: "form-control",
    minlength: 8,
    autocomplete: "new-password" %>
<!-- => <input type="password" name="user[password]"> -->

<%= form.password_field :password_confirmation,
    placeholder: "ยืนยันรหัสผ่าน" %>
```

### number_field

```erb
<%= form.number_field :age,
    min: 0,
    max: 150,
    step: 1,
    placeholder: "อายุ",
    class: "form-control" %>
<!-- => <input type="number" min="0" max="150" step="1"> -->

<%= form.number_field :price,
    min: 0,
    max: 9999999,
    step: 0.01,
    placeholder: "0.00" %>
```

### url_field

```erb
<%= form.url_field :website,
    placeholder: "https://example.com",
    class: "form-control" %>
<!-- => <input type="url" name="user[website]"> -->
```

### อื่นๆ

```erb
<!-- Tel -->
<%= form.telephone_field :phone, placeholder: "0812345678" %>

<!-- Search -->
<%= form.search_field :q, placeholder: "ค้นหา..." %>

<!-- Color picker -->
<%= form.color_field :theme_color %>
<!-- => <input type="color"> -->

<!-- Range slider -->
<%= form.range_field :priority, min: 1, max: 10, step: 1 %>

<!-- Date/Time -->
<%= form.date_field :birthday %>
<%= form.time_field :start_time %>
<%= form.datetime_local_field :published_at %>
<%= form.month_field :birth_month %>
<%= form.week_field :target_week %>
```

---

## Step 883: text_area

```erb
<%= form_with model: @post do |form| %>
  <!-- Basic textarea -->
  <%= form.text_area :content %>
  <!-- => <textarea name="post[content]">...</textarea> -->
  
  <!-- ด้วย options -->
  <%= form.text_area :content,
      rows: 10,
      cols: 80,
      placeholder: "เขียนเนื้อหาที่นี่...",
      class: "form-control",
      maxlength: 10000,
      required: true %>
  
  <!-- ด้วย size shorthand -->
  <%= form.text_area :bio, size: "50x15" %>
  <!-- rows=15, cols=50 -->
<% end %>
```

---

## Step 884: check_box, radio_button

### check_box

```erb
<%= form_with model: @post do |form| %>
  <!-- Basic checkbox -->
  <%= form.check_box :published %>
  <!-- => hidden field + checkbox -->
  
  <!-- ด้วย label -->
  <div class="form-check">
    <%= form.check_box :published, class: "form-check-input", id: "post_published" %>
    <%= form.label :published, "เผยแพร่", class: "form-check-label" %>
  </div>
  
  <!-- Checked by default -->
  <%= form.check_box :newsletter, checked: true %>
  
  <!-- Custom on/off values -->
  <%= form.check_box :active, { checked: @user.active? }, "1", "0" %>
  <!-- value=1 เมื่อ checked, value=0 เมื่อ unchecked -->
  
  <!-- Multiple checkboxes -->
  <% @categories.each do |category| %>
    <div class="form-check">
      <%= check_box_tag "post[category_ids][]", category.id,
          @post.category_ids.include?(category.id),
          class: "form-check-input",
          id: "category_#{category.id}" %>
      <%= label_tag "category_#{category.id}", category.name, class: "form-check-label" %>
    </div>
  <% end %>
<% end %>
```

### radio_button

```erb
<%= form_with model: @post do |form| %>
  <!-- Radio buttons -->
  <% %w[draft published archived].each do |status| %>
    <div class="form-check">
      <%= form.radio_button :status, status, class: "form-check-input" %>
      <%= form.label :"status_#{status}", status.humanize, class: "form-check-label" %>
    </div>
  <% end %>
  
  <!-- Radio ด้วย checked state -->
  <%= form.radio_button :role, "admin", checked: @user.admin? %>
  <%= form.radio_button :role, "member", checked: @user.member? %>
<% end %>
```

---

## Step 885: select, collection_select, grouped_collection_select

### select

```erb
<%= form_with model: @post do |form| %>
  <!-- select ด้วย array of strings -->
  <%= form.select :status, %w[draft published archived] %>
  
  <!-- select ด้วย array of [label, value] pairs -->
  <%= form.select :status, 
      [["ร่าง", "draft"], ["เผยแพร่", "published"], ["เก็บถาวร", "archived"]] %>
  
  <!-- ด้วย prompt -->
  <%= form.select :category_id, @categories.map { |c| [c.name, c.id] },
      { prompt: "-- เลือกหมวดหมู่ --" },
      class: "form-select" %>
  
  <!-- ด้วย include_blank -->
  <%= form.select :status, options_for_select([...]), include_blank: "เลือก..." %>
  
  <!-- Multiple select -->
  <%= form.select :tag_ids, @tags.map { |t| [t.name, t.id] },
      {},
      multiple: true, size: 5 %>
  
  <!-- Grouped options -->
  <%= form.select :city,
      grouped_options_for_select([
        ["ภาคกลาง", [["กรุงเทพฯ", "bkk"], ["นนทบุรี", "nbi"]]],
        ["ภาคเหนือ", [["เชียงใหม่", "cnx"], ["เชียงราย", "cri"]]]
      ]),
      prompt: "เลือกจังหวัด" %>
  
  <!-- ด้วย selected value -->
  <%= form.select :status, 
      options_for_select(Post::STATUSES.map { |s| [s.humanize, s] }, @post.status) %>
<% end %>
```

### collection_select

```erb
<%= form_with model: @post do |form| %>
  <!-- collection_select(method, collection, value_method, text_method, options, html_options) -->
  <%= form.collection_select :category_id, 
      Category.all, 
      :id, :name,
      { prompt: "เลือกหมวดหมู่" },
      { class: "form-select" } %>
  
  <!-- Multiple selection -->
  <%= form.collection_select :tag_ids,
      Tag.all,
      :id, :name,
      {},
      { multiple: true, class: "form-select" } %>
  
  <!-- ด้วย scope -->
  <%= form.collection_select :category_id,
      Category.active.order(:name),
      :id, :name,
      { include_blank: "-- ไม่ระบุ --" } %>
<% end %>
```

### grouped_collection_select

```erb
<%= form.grouped_collection_select :city_id,
    Country.all, :cities, :name,  # collection, group_method, group_label_method
    :id, :name,                   # option_value_method, option_key_method
    { prompt: "เลือกเมือง" } %>
```

---

## Step 886: file_field

```erb
<%= form_with model: @user, multipart: true do |form| %>
  <!-- Basic file upload -->
  <%= form.file_field :avatar %>
  
  <!-- ด้วย accept types -->
  <%= form.file_field :avatar,
      accept: "image/*",
      class: "form-control" %>
  
  <!-- เฉพาะ images ที่กำหนด -->
  <%= form.file_field :document,
      accept: ".pdf,.doc,.docx",
      class: "form-control" %>
  
  <!-- Multiple files -->
  <%= form.file_field :photos,
      multiple: true,
      accept: "image/*" %>
  
  <!-- ด้วย Active Storage -->
  <%= form.file_field :cover_image,
      accept: "image/jpeg,image/png,image/gif",
      data: { controller: "file-preview" } %>
<% end %>
```

```ruby
# Controller
def create
  @user = User.new(user_params)
  if @user.save
    redirect_to @user
  else
    render :new, status: :unprocessable_entity
  end
end

private

def user_params
  params.require(:user).permit(:name, :email, :avatar)  # avatar เป็น Active Storage
end
```

---

## Step 887: hidden_field

```erb
<%= form_with model: @comment do |form| %>
  <!-- Hidden field สำหรับ post_id -->
  <%= form.hidden_field :post_id, value: @post.id %>
  
  <!-- Hidden field สำหรับ return URL -->
  <%= form.hidden_field :return_to, value: request.fullpath %>
  
  <!-- Hidden field สำหรับ session token -->
  <%= form.hidden_field :token, value: current_user.session_token %>
  
  <%= form.text_area :content, required: true %>
  <%= form.submit "ส่งความคิดเห็น" %>
<% end %>

<!-- ใช้ hidden_field_tag สำหรับ non-model -->
<%= form_tag search_path do %>
  <%= hidden_field_tag :sort, "created_at" %>
  <%= hidden_field_tag :direction, "desc" %>
  <%= text_field_tag :q, params[:q] %>
  <%= submit_tag "ค้นหา" %>
<% end %>
```

---

## Step 888: submit, button

### submit

```erb
<%= form_with model: @post do |form| %>
  <!-- Auto label: "Create Post" หรือ "Update Post" -->
  <%= form.submit %>
  
  <!-- Custom label -->
  <%= form.submit "บันทึก" %>
  
  <!-- ด้วย options -->
  <%= form.submit "บันทึก",
      class: "btn btn-primary",
      id: "submit-btn",
      data: { loading_text: "กำลังบันทึก..." },
      disabled: !@post.valid? %>
<% end %>
```

### button

```erb
<%= form_with model: @post do |form| %>
  <!-- Button ด้วย icon -->
  <%= form.button class: "btn btn-primary" do %>
    <i class="icon-save"></i> บันทึก
  <% end %>
  
  <!-- Button ด้วย type -->
  <%= form.button "ยกเลิก", type: :button, 
      onclick: "history.back()", 
      class: "btn btn-secondary" %>
  
  <!-- Reset button -->
  <%= form.button "รีเซ็ต", type: :reset, class: "btn btn-outline-secondary" %>
<% end %>
```

---

## Step 889: Displaying Validation Errors in Form

```erb
<!-- app/views/posts/new.html.erb -->
<h1>สร้างบทความใหม่</h1>

<%= form_with model: @post do |form| %>
  <!-- แสดง errors รวม -->
  <% if @post.errors.any? %>
    <div class="alert alert-danger">
      <h5>พบ <%= pluralize(@post.errors.count, "ข้อผิดพลาด") %>:</h5>
      <ul class="mb-0">
        <% @post.errors.full_messages.each do |message| %>
          <li><%= message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <!-- Title field ด้วย error display -->
  <div class="mb-3">
    <%= form.label :title, "หัวข้อ", class: "form-label" %>
    <%= form.text_field :title, 
        class: "form-control #{'is-invalid' if @post.errors[:title].any?}",
        placeholder: "กรอกหัวข้อบทความ" %>
    <% if @post.errors[:title].any? %>
      <div class="invalid-feedback">
        <%= @post.errors[:title].join(", ") %>
      </div>
    <% end %>
  </div>

  <!-- Content field -->
  <div class="mb-3">
    <%= form.label :content, "เนื้อหา", class: "form-label" %>
    <%= form.text_area :content,
        rows: 10,
        class: "form-control #{'is-invalid' if @post.errors[:content].any?}" %>
    <% if @post.errors[:content].any? %>
      <div class="invalid-feedback">
        <%= @post.errors[:content].join(", ") %>
      </div>
    <% end %>
  </div>

  <%= form.submit "บันทึก", class: "btn btn-primary" %>
<% end %>
```

### Helper สำหรับ field error

```ruby
# app/helpers/form_helper.rb
module FormHelper
  def field_with_error(form, attribute, label_text, &block)
    has_error = form.object.errors[attribute].any?
    
    content_tag :div, class: "mb-3" do
      concat form.label(attribute, label_text, class: "form-label")
      concat capture(&block)
      if has_error
        concat content_tag(:div, form.object.errors[attribute].join(", "), 
                           class: "invalid-feedback d-block")
      end
    end
  end
end
```

---

## Step 890: form_with without model (search forms)

```erb
<!-- Search form - ไม่ใช้ model -->
<%= form_with url: search_path, method: :get do |form| %>
  <div class="input-group">
    <%= form.search_field :q,
        placeholder: "ค้นหา...",
        value: params[:q],
        class: "form-control" %>
    <%= form.submit "ค้นหา", class: "btn btn-primary" %>
  </div>
<% end %>

<!-- Filter form -->
<%= form_with url: posts_path, method: :get, class: "filter-form" do |form| %>
  <!-- Select status -->
  <div class="mb-2">
    <%= form.label :status, "สถานะ" %>
    <%= form.select :status, 
        [["ทั้งหมด", ""], ["ร่าง", "draft"], ["เผยแพร่", "published"]],
        { selected: params[:status] },
        class: "form-select" %>
  </div>
  
  <!-- Date range -->
  <div class="row">
    <div class="col">
      <%= form.date_field :from_date, value: params[:from_date], class: "form-control" %>
    </div>
    <div class="col">
      <%= form.date_field :to_date, value: params[:to_date], class: "form-control" %>
    </div>
  </div>
  
  <%= form.submit "กรอง", class: "btn btn-secondary" %>
  <%= link_to "ล้างตัวกรอง", posts_path, class: "btn btn-outline-secondary" %>
<% end %>
```

---

## Step 891: Nested Forms

### accepts_nested_attributes_for

```ruby
# Model
class User < ApplicationRecord
  has_one :profile
  has_many :addresses
  
  accepts_nested_attributes_for :profile
  accepts_nested_attributes_for :addresses,
    allow_destroy: true,
    reject_if: :all_blank
end
```

### Nested form ใน view

```erb
<!-- app/views/users/new.html.erb -->
<%= form_with model: @user do |form| %>
  <!-- User fields -->
  <div class="mb-3">
    <%= form.label :name %>
    <%= form.text_field :name, class: "form-control" %>
  </div>
  
  <!-- Profile nested -->
  <%= form.fields_for :profile do |profile_form| %>
    <div class="mb-3">
      <%= profile_form.label :bio, "ประวัติ" %>
      <%= profile_form.text_area :bio, rows: 4, class: "form-control" %>
    </div>
    <div class="mb-3">
      <%= profile_form.label :website, "เว็บไซต์" %>
      <%= profile_form.url_field :website, class: "form-control" %>
    </div>
  <% end %>
  
  <!-- Addresses nested (has_many) -->
  <h4>ที่อยู่</h4>
  
  <%= form.fields_for :addresses do |address_form| %>
    <div class="address-fields border p-3 mb-2" id="<%= dom_id(address_form.object) %>">
      <!-- Hidden id for update -->
      <%= address_form.hidden_field :id %>
      
      <div class="mb-2">
        <%= address_form.label :street, "ที่อยู่" %>
        <%= address_form.text_field :street, class: "form-control" %>
      </div>
      
      <div class="row">
        <div class="col">
          <%= address_form.label :city, "เมือง" %>
          <%= address_form.text_field :city, class: "form-control" %>
        </div>
        <div class="col">
          <%= address_form.label :postal_code, "รหัสไปรษณีย์" %>
          <%= address_form.text_field :postal_code, class: "form-control" %>
        </div>
      </div>
      
      <!-- Destroy checkbox -->
      <div class="form-check">
        <%= address_form.check_box :_destroy, class: "form-check-input" %>
        <%= address_form.label :_destroy, "ลบที่อยู่นี้", class: "form-check-label text-danger" %>
      </div>
    </div>
  <% end %>
  
  <%= form.submit "บันทึก", class: "btn btn-primary" %>
<% end %>
```

### Controller สำหรับ nested forms

```ruby
class UsersController < ApplicationController
  def new
    @user = User.new
    @user.build_profile
    3.times { @user.addresses.build }
  end
  
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  private
  
  def user_params
    params.require(:user).permit(
      :name, :email, :password,
      profile_attributes: [:id, :bio, :website],
      addresses_attributes: [:id, :street, :city, :postal_code, :_destroy]
    )
  end
end
```

---

## Step 892: CSRF Protection

### CSRF Token

```erb
<!-- Layout auto includes this -->
<%= csrf_meta_tags %>
<!-- => <meta name="csrf-param" content="authenticity_token">
        <meta name="csrf-token" content="xxxx"> -->

<!-- form_with auto includes hidden field -->
<%= form_with model: @post do |form| %>
  <!-- hidden field: <input type="hidden" name="authenticity_token" value="xxxx"> -->
<% end %>
```

### CSRF ใน API/AJAX

```ruby
# สำหรับ API controllers
class Api::ApplicationController < ActionController::API
  # ActionController::API ไม่มี CSRF protection โดย default
end

# สำหรับ ActionController::Base ที่ต้องการ skip
class WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token
  
  def receive
    # Process webhook
  end
end
```

```javascript
// JavaScript - ส่ง CSRF token ใน AJAX request
const csrfToken = document.querySelector('meta[name="csrf-token"]').getAttribute('content');

fetch('/posts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': csrfToken
  },
  body: JSON.stringify({ post: { title: 'New Post' } })
});
```

---

## Step 893: Turbo และ Forms (Rails 7)

Rails 7 ใช้ Hotwire Turbo ซึ่งเปลี่ยนวิธีที่ forms ทำงาน

```erb
<!-- Form ที่ใช้ Turbo Stream response -->
<%= form_with model: @comment do |form| %>
  <%= form.text_area :content, class: "form-control" %>
  <%= form.submit "ส่ง", class: "btn btn-primary" %>
<% end %>

<!-- Frame สำหรับ partial updates -->
<%= turbo_frame_tag "comment-form" do %>
  <%= form_with model: @comment do |form| %>
    <%= form.text_area :content %>
    <%= form.submit %>
  <% end %>
<% end %>

<!-- Turbo Stream ใน response -->
<!-- app/views/comments/create.turbo_stream.erb -->
<%= turbo_stream.append "comments-list" do %>
  <%= render @comment %>
<% end %>

<%= turbo_stream.replace "comment-form" do %>
  <%= render "form", comment: Comment.new %>
<% end %>
```

### Controller สำหรับ Turbo

```ruby
class CommentsController < ApplicationController
  def create
    @post = Post.find(params[:post_id])
    @comment = @post.comments.new(comment_params)
    @comment.user = current_user
    
    respond_to do |format|
      if @comment.save
        format.turbo_stream
        format.html { redirect_to @post }
      else
        format.turbo_stream do
          render turbo_stream: turbo_stream.replace("comment-form",
            partial: "comments/form",
            locals: { comment: @comment }
          )
        end
        format.html { render "posts/show", status: :unprocessable_entity }
      end
    end
  end
end
```

---

## Step 894: Form Helpers ใน Views

### link_to

```erb
<!-- Basic link -->
<%= link_to "ดูบทความ", post_path(@post) %>

<!-- Link ด้วย class -->
<%= link_to "แก้ไข", edit_post_path(@post), class: "btn btn-sm btn-warning" %>

<!-- Delete link -->
<%= link_to "ลบ", post_path(@post),
    data: { turbo_method: :delete, turbo_confirm: "ต้องการลบ?" },
    class: "btn btn-sm btn-danger" %>

<!-- Link ด้วย block -->
<%= link_to post_path(@post) do %>
  <img src="<%= @post.thumbnail_url %>" alt="<%= @post.title %>">
  <span><%= @post.title %></span>
<% end %>

<!-- Link ที่เปิด tab ใหม่ -->
<%= link_to "เปิดในแท็บใหม่", @post, target: "_blank", rel: "noopener noreferrer" %>
```

### button_to

```erb
<!-- สร้าง mini form ที่มีปุ่มเดียว -->
<%= button_to "Like", like_post_path(@post), method: :post %>

<%= button_to "Follow", follow_user_path(@user),
    method: :post,
    class: "btn btn-sm btn-primary",
    data: { turbo_confirm: "Follow #{@user.name}?" } %>

<!-- DELETE button -->
<%= button_to "Unfollow", follow_user_path(@follow),
    method: :delete,
    class: "btn btn-sm btn-outline-danger" %>

<!-- ด้วย block -->
<%= button_to post_path(@post), method: :delete do %>
  <i class="icon-trash"></i> ลบ
<% end %>
```

---

## ตัวอย่าง Form ที่สมบูรณ์

### Registration Form

```erb
<!-- app/views/users/new.html.erb -->
<% content_for :title, "สมัครสมาชิก" %>

<div class="container">
  <div class="row justify-content-center">
    <div class="col-md-6">
      <h1 class="mb-4">สมัครสมาชิก</h1>
      
      <%= form_with model: @user, class: "needs-validation" do |form| %>
        <!-- Errors -->
        <% if @user.errors.any? %>
          <div class="alert alert-danger">
            <% @user.errors.full_messages.each do |msg| %>
              <p class="mb-0"><%= msg %></p>
            <% end %>
          </div>
        <% end %>
        
        <!-- Name -->
        <div class="mb-3">
          <%= form.label :name, "ชื่อ-นามสกุล", class: "form-label" %>
          <%= form.text_field :name,
              class: "form-control #{'is-invalid' if @user.errors[:name].any?}",
              placeholder: "กรอกชื่อ-นามสกุล",
              autocomplete: "name",
              required: true %>
          <% if @user.errors[:name].any? %>
            <div class="invalid-feedback"><%= @user.errors[:name].first %></div>
          <% end %>
        </div>
        
        <!-- Email -->
        <div class="mb-3">
          <%= form.label :email, "อีเมล", class: "form-label" %>
          <%= form.email_field :email,
              class: "form-control #{'is-invalid' if @user.errors[:email].any?}",
              placeholder: "your@email.com",
              autocomplete: "email",
              required: true %>
          <% if @user.errors[:email].any? %>
            <div class="invalid-feedback"><%= @user.errors[:email].first %></div>
          <% end %>
        </div>
        
        <!-- Password -->
        <div class="mb-3">
          <%= form.label :password, "รหัสผ่าน", class: "form-label" %>
          <%= form.password_field :password,
              class: "form-control #{'is-invalid' if @user.errors[:password].any?}",
              placeholder: "อย่างน้อย 8 ตัวอักษร",
              minlength: 8,
              autocomplete: "new-password",
              required: true %>
          <% if @user.errors[:password].any? %>
            <div class="invalid-feedback"><%= @user.errors[:password].first %></div>
          <% else %>
            <div class="form-text">ต้องมีอย่างน้อย 8 ตัวอักษร</div>
          <% end %>
        </div>
        
        <!-- Password Confirmation -->
        <div class="mb-3">
          <%= form.label :password_confirmation, "ยืนยันรหัสผ่าน", class: "form-label" %>
          <%= form.password_field :password_confirmation,
              class: "form-control",
              placeholder: "กรอกรหัสผ่านอีกครั้ง",
              autocomplete: "new-password",
              required: true %>
        </div>
        
        <!-- Role -->
        <div class="mb-3">
          <%= form.label :role, "บทบาท", class: "form-label" %>
          <%= form.select :role,
              [["สมาชิก", "member"], ["ผู้ดูแล", "moderator"]],
              { selected: "member" },
              class: "form-select" %>
        </div>
        
        <!-- Phone -->
        <div class="mb-3">
          <%= form.label :phone, "เบอร์โทรศัพท์ (ไม่บังคับ)", class: "form-label" %>
          <%= form.telephone_field :phone,
              class: "form-control",
              placeholder: "0812345678" %>
        </div>
        
        <!-- Avatar -->
        <div class="mb-3">
          <%= form.label :avatar, "รูปโปรไฟล์ (ไม่บังคับ)", class: "form-label" %>
          <%= form.file_field :avatar,
              accept: "image/jpeg,image/png,image/gif",
              class: "form-control" %>
        </div>
        
        <!-- Terms -->
        <div class="mb-3 form-check">
          <%= form.check_box :terms_accepted,
              class: "form-check-input #{'is-invalid' if @user.errors[:terms_accepted].any?}",
              required: true %>
          <%= form.label :terms_accepted, class: "form-check-label" do %>
            ฉันยอมรับ <%= link_to "ข้อตกลงการใช้งาน", terms_path, target: "_blank" %>
            และ <%= link_to "นโยบายความเป็นส่วนตัว", privacy_path, target: "_blank" %>
          <% end %>
          <% if @user.errors[:terms_accepted].any? %>
            <div class="invalid-feedback"><%= @user.errors[:terms_accepted].first %></div>
          <% end %>
        </div>
        
        <!-- Submit -->
        <div class="d-grid">
          <%= form.submit "สมัครสมาชิก", class: "btn btn-primary btn-lg" %>
        </div>
        
      <% end %>
      
      <p class="text-center mt-3">
        มีบัญชีแล้ว? <%= link_to "เข้าสู่ระบบ", login_path %>
      </p>
    </div>
  </div>
</div>
```

---

## แบบฝึกหัด (Steps 895-905)

### แบบฝึกหัดที่ 1
สร้าง basic form สำหรับ Post model ด้วย form_with

**เฉลย:**
```erb
<%= form_with model: @post do |form| %>
  <%= form.label :title %>
  <%= form.text_field :title, class: "form-control" %>
  
  <%= form.label :content %>
  <%= form.text_area :content, rows: 5, class: "form-control" %>
  
  <%= form.submit class: "btn btn-primary" %>
<% end %>
```

### แบบฝึกหัดที่ 2
สร้าง search form ที่ไม่ใช้ model

**เฉลย:**
```erb
<%= form_with url: search_path, method: :get do |form| %>
  <%= form.text_field :q, placeholder: "ค้นหา...", value: params[:q] %>
  <%= form.submit "ค้นหา" %>
<% end %>
```

### แบบฝึกหัดที่ 3
เพิ่ม validation error display ใน form field

**เฉลย:**
```erb
<div class="mb-3">
  <%= form.label :email %>
  <%= form.email_field :email,
      class: "form-control #{'is-invalid' if @user.errors[:email].any?}" %>
  <% if @user.errors[:email].any? %>
    <div class="invalid-feedback">
      <%= @user.errors[:email].first %>
    </div>
  <% end %>
</div>
```

### แบบฝึกหัดที่ 4
สร้าง file upload form สำหรับ avatar

**เฉลย:**
```erb
<%= form_with model: @user, multipart: true do |form| %>
  <%= form.file_field :avatar, 
      accept: "image/*",
      class: "form-control" %>
  <%= form.submit "อัปโหลด" %>
<% end %>
```

### แบบฝึกหัดที่ 5
สร้าง select dropdown สำหรับ category

**เฉลย:**
```erb
<%= form.collection_select :category_id,
    Category.order(:name),
    :id, :name,
    { prompt: "เลือกหมวดหมู่" },
    class: "form-select" %>
```

### แบบฝึกหัดที่ 6
สร้าง multiple checkboxes สำหรับ tags

**เฉลย:**
```erb
<% Tag.all.each do |tag| %>
  <div class="form-check form-check-inline">
    <%= check_box_tag "post[tag_ids][]", tag.id,
        @post.tag_ids.include?(tag.id),
        class: "form-check-input",
        id: "tag_#{tag.id}" %>
    <%= label_tag "tag_#{tag.id}", tag.name, class: "form-check-label" %>
  </div>
<% end %>
```

### แบบฝึกหัดที่ 7
สร้าง radio buttons สำหรับ status

**เฉลย:**
```erb
<% %w[draft published archived].each do |status| %>
  <div class="form-check">
    <%= form.radio_button :status, status, class: "form-check-input" %>
    <%= form.label :"status_#{status}", status.humanize, class: "form-check-label" %>
  </div>
<% end %>
```

### แบบฝึกหัดที่ 8
implement nested form สำหรับ User และ Profile

**เฉลย:**
```ruby
# Model
class User < ApplicationRecord
  has_one :profile
  accepts_nested_attributes_for :profile
end

# Controller
def new
  @user = User.new
  @user.build_profile
end

def user_params
  params.require(:user).permit(:name, :email,
    profile_attributes: [:id, :bio, :website])
end
```

```erb
<!-- View -->
<%= form_with model: @user do |form| %>
  <%= form.text_field :name %>
  
  <%= form.fields_for :profile do |profile_form| %>
    <%= profile_form.text_area :bio %>
    <%= profile_form.url_field :website %>
  <% end %>
  
  <%= form.submit %>
<% end %>
```

### แบบฝึกหัดที่ 9
สร้าง form ที่มี password และ password_confirmation fields

**เฉลย:**
```erb
<div class="mb-3">
  <%= form.label :password, "รหัสผ่าน" %>
  <%= form.password_field :password, 
      class: "form-control",
      minlength: 8,
      autocomplete: "new-password" %>
</div>

<div class="mb-3">
  <%= form.label :password_confirmation, "ยืนยันรหัสผ่าน" %>
  <%= form.password_field :password_confirmation,
      class: "form-control",
      autocomplete: "new-password" %>
</div>
```

### แบบฝึกหัดที่ 10
ใช้ button_to สำหรับ delete action

**เฉลย:**
```erb
<%= button_to "ลบ", post_path(@post),
    method: :delete,
    data: { confirm: "ต้องการลบบทความนี้?" },
    class: "btn btn-danger" %>
```

### แบบฝึกหัดที่ 11
สร้าง form ที่ใช้ hidden_field สำหรับ parent id

**เฉลย:**
```erb
<%= form_with model: @comment, url: post_comments_path(@post) do |form| %>
  <%= form.hidden_field :post_id, value: @post.id %>
  <%= form.text_area :content, rows: 3, class: "form-control" %>
  <%= form.submit "ส่ง", class: "btn btn-primary" %>
<% end %>
```

### แบบฝึกหัดที่ 12
สร้าง date picker form field

**เฉลย:**
```erb
<div class="mb-3">
  <%= form.label :published_at, "วันที่เผยแพร่" %>
  <%= form.datetime_local_field :published_at,
      class: "form-control",
      value: @post.published_at&.strftime("%Y-%m-%dT%H:%M") %>
</div>
```

### แบบฝึกหัดที่ 13
Implement form ที่รับ array values

**เฉลย:**
```erb
<!-- Controller -->
def post_params
  params.require(:post).permit(:title, tag_ids: [], category_ids: [])
end

<!-- View -->
<% Tag.all.each do |tag| %>
  <%= check_box_tag "post[tag_ids][]", tag.id, @post.tag_ids.include?(tag.id) %>
  <%= label_tag "tag_#{tag.id}", tag.name %>
<% end %>
```

### แบบฝึกหัดที่ 14
สร้าง Turbo Stream form สำหรับ comment

**เฉลย:**
```erb
<!-- Form -->
<%= turbo_frame_tag "comment-form" do %>
  <%= form_with model: @comment, url: post_comments_path(@post) do |form| %>
    <%= form.text_area :content, rows: 3, class: "form-control" %>
    <%= form.submit "ส่ง", class: "btn btn-primary" %>
  <% end %>
<% end %>

<turbo-frame id="comments-list">
  <%= render @post.comments %>
</turbo-frame>
```

```ruby
# Controller
def create
  @comment = @post.comments.new(comment_params)
  
  respond_to do |format|
    if @comment.save
      format.turbo_stream do
        render turbo_stream: [
          turbo_stream.append("comments-list", partial: "comment", locals: { comment: @comment }),
          turbo_stream.replace("comment-form", partial: "form", locals: { comment: Comment.new, post: @post })
        ]
      end
      format.html { redirect_to @post }
    end
  end
end
```

### แบบฝึกหัดที่ 15
สร้าง grouped select สำหรับ country/city

**เฉลย:**
```erb
<%= form.select :city,
    grouped_options_for_select([
      ["ภาคกลาง", [["กรุงเทพฯ", "BKK"], ["นนทบุรี", "NBI"], ["ปทุมธานี", "PTN"]]],
      ["ภาคเหนือ", [["เชียงใหม่", "CNX"], ["เชียงราย", "CRI"]]],
      ["ภาคใต้", [["สงขลา", "SGK"], ["ภูเก็ต", "HKT"]]]
    ]),
    prompt: "เลือกจังหวัด",
    selected: params[:city] %>
```

### แบบฝึกหัดที่ 16
implement form ที่ handle AJAX submission

**เฉลย:**
```erb
<!-- Rails 7 ใช้ Turbo โดย default (AJAX) -->
<%= form_with model: @post do |form| %>
  <%= form.text_field :title %>
  <%= form.submit %>
<% end %>

<!-- ปิด Turbo ถ้าต้องการ regular form -->
<%= form_with model: @post, data: { turbo: false } do |form| %>
  ...
<% end %>
```

### แบบฝึกหัดที่ 17
สร้าง form ที่แสดง inline errors ทุก field

**เฉลย:**
```erb
<%= form_with model: @user do |form| %>
  <% [:name, :email, :password].each do |attr| %>
    <div class="mb-3">
      <%= form.label attr %>
      <%= form.send("#{attr}_field", attr,
          class: "form-control #{'is-invalid' if @user.errors[attr].any?}") %>
      <% if @user.errors[attr].any? %>
        <div class="invalid-feedback">
          <%= @user.errors[attr].join(", ") %>
        </div>
      <% end %>
    </div>
  <% end %>
  
  <%= form.submit "บันทึก", class: "btn btn-primary" %>
<% end %>
```

### แบบฝึกหัดที่ 18
สร้าง filter form ที่ maintain state หลัง submit

**เฉลย:**
```erb
<%= form_with url: posts_path, method: :get, class: "row g-2 align-items-end" do |form| %>
  <div class="col-auto">
    <%= form.text_field :q, 
        value: params[:q], 
        placeholder: "ค้นหา...",
        class: "form-control" %>
  </div>
  
  <div class="col-auto">
    <%= form.select :status,
        [["ทั้งหมด", ""], ["ร่าง", "draft"], ["เผยแพร่", "published"]],
        { selected: params[:status] },
        class: "form-select" %>
  </div>
  
  <div class="col-auto">
    <%= form.select :sort,
        [["ใหม่สุด", "desc"], ["เก่าสุด", "asc"]],
        { selected: params[:sort] || "desc" },
        class: "form-select" %>
  </div>
  
  <div class="col-auto">
    <%= form.submit "กรอง", class: "btn btn-secondary" %>
    <%= link_to "ล้าง", posts_path, class: "btn btn-outline-secondary" %>
  </div>
<% end %>
```

### แบบฝึกหัดที่ 19
implement range slider สำหรับ price filter

**เฉลย:**
```erb
<%= form_with url: products_path, method: :get do |form| %>
  <div class="mb-3">
    <%= form.label :min_price, "ราคาขั้นต่ำ: " %>
    <span id="min-price-display"><%= params[:min_price] || 0 %></span> บาท
    <%= form.range_field :min_price,
        min: 0, max: 10000, step: 100,
        value: params[:min_price] || 0,
        class: "form-range",
        oninput: "document.getElementById('min-price-display').textContent = this.value" %>
  </div>
  
  <div class="mb-3">
    <%= form.label :max_price, "ราคาสูงสุด: " %>
    <span id="max-price-display"><%= params[:max_price] || 10000 %></span> บาท
    <%= form.range_field :max_price,
        min: 0, max: 10000, step: 100,
        value: params[:max_price] || 10000,
        class: "form-range",
        oninput: "document.getElementById('max-price-display').textContent = this.value" %>
  </div>
  
  <%= form.submit "กรอง", class: "btn btn-primary" %>
<% end %>
```

### แบบฝึกหัดที่ 20
สร้าง multi-step form ด้วย session

**เฉลย:**
```ruby
# Controller
class RegistrationsController < ApplicationController
  def step1
    @user = User.new(session[:registration_data])
  end
  
  def save_step1
    user_data = params.require(:user).permit(:name, :email)
    session[:registration_data] = user_data
    redirect_to registration_step2_path
  end
  
  def step2
    @user = User.new(session[:registration_data])
  end
  
  def create
    @user = User.new(session[:registration_data])
    additional = params.require(:user).permit(:password, :terms_accepted)
    @user.assign_attributes(additional)
    
    if @user.save
      session.delete(:registration_data)
      redirect_to root_path, notice: "สมัครสมาชิกสำเร็จ"
    else
      render :step2, status: :unprocessable_entity
    end
  end
end
```

### แบบฝึกหัดที่ 21
สร้าง accepts_nested_attributes_for สำหรับ has_many

**เฉลย:**
```ruby
class Invoice < ApplicationRecord
  has_many :line_items
  accepts_nested_attributes_for :line_items,
    allow_destroy: true,
    reject_if: :all_blank
end
```

```erb
<%= form_with model: @invoice do |form| %>
  <%= form.text_field :invoice_number %>
  
  <h4>รายการสินค้า</h4>
  <%= form.fields_for :line_items do |item_form| %>
    <div class="line-item">
      <%= item_form.hidden_field :id %>
      <%= item_form.text_field :product_name, placeholder: "ชื่อสินค้า" %>
      <%= item_form.number_field :quantity, min: 1 %>
      <%= item_form.number_field :unit_price, min: 0, step: 0.01 %>
      <div class="form-check">
        <%= item_form.check_box :_destroy %>
        <%= item_form.label :_destroy, "ลบ" %>
      </div>
    </div>
  <% end %>
  
  <%= form.submit "บันทึก" %>
<% end %>
```

### แบบฝึกหัดที่ 22
สร้าง form พร้อม Ajax validation

**เฉลย:**
```erb
<%= form_with model: @user, data: { controller: "form-validation" } do |form| %>
  <%= form.email_field :email,
      data: { 
        action: "blur->form-validation#validateEmail",
        form_validation_url_value: validate_email_path
      } %>
  <div id="email-validation-result"></div>
  <%= form.submit %>
<% end %>
```

```ruby
# Controller
def validate_email
  exists = User.exists?(email: params[:email])
  render json: { valid: !exists, message: exists ? "อีเมลนี้ถูกใช้งานแล้ว" : "OK" }
end
```

### แบบฝึกหัดที่ 23
implement form ด้วย character counter

**เฉลย:**
```erb
<div class="mb-3">
  <%= form.label :bio, "ประวัติ" %>
  <%= form.text_area :bio,
      rows: 4,
      class: "form-control",
      maxlength: 500,
      data: { 
        controller: "char-counter",
        char_counter_max_value: 500
      } %>
  <div class="text-muted">
    <span data-char-counter-target="count">0</span>/500 ตัวอักษร
  </div>
</div>
```

### แบบฝึกหัดที่ 24
สร้าง autocomplete text field

**เฉลย:**
```erb
<div class="mb-3" data-controller="autocomplete" 
     data-autocomplete-url-value="<%= search_users_path %>">
  <%= form.text_field :recipient,
      class: "form-control",
      autocomplete: "off",
      data: { 
        autocomplete_target: "input",
        action: "input->autocomplete#search"
      } %>
  <ul class="dropdown-menu" data-autocomplete-target="results">
  </ul>
</div>
```

### แบบฝึกหัดที่ 25
สร้าง complete Post form ด้วย features ครบถ้วน

**เฉลย:**
```erb
<%= form_with model: @post, class: "post-form" do |form| %>
  <!-- Errors -->
  <%= render "shared/errors", object: @post %>
  
  <!-- Title -->
  <div class="mb-3">
    <%= form.label :title, "หัวข้อบทความ *", class: "form-label fw-bold" %>
    <%= form.text_field :title,
        class: "form-control form-control-lg #{'is-invalid' if @post.errors[:title].any?}",
        placeholder: "กรอกหัวข้อที่น่าสนใจ",
        autofocus: true, required: true %>
    <% if @post.errors[:title].any? %>
      <div class="invalid-feedback"><%= @post.errors[:title].first %></div>
    <% else %>
      <div class="form-text">หัวข้อควรสั้น กระชับ และน่าสนใจ</div>
    <% end %>
  </div>
  
  <!-- Category -->
  <div class="mb-3">
    <%= form.label :category_id, "หมวดหมู่", class: "form-label" %>
    <%= form.collection_select :category_id, Category.active, :id, :name,
        { prompt: "-- เลือกหมวดหมู่ --" },
        class: "form-select #{'is-invalid' if @post.errors[:category_id].any?}" %>
  </div>
  
  <!-- Content -->
  <div class="mb-3">
    <%= form.label :content, "เนื้อหา *", class: "form-label fw-bold" %>
    <%= form.text_area :content,
        rows: 15,
        class: "form-control #{'is-invalid' if @post.errors[:content].any?}",
        placeholder: "เขียนเนื้อหาที่นี่...",
        required: true %>
    <% if @post.errors[:content].any? %>
      <div class="invalid-feedback"><%= @post.errors[:content].first %></div>
    <% end %>
  </div>
  
  <!-- Tags -->
  <div class="mb-3">
    <%= form.label :tag_ids, "แท็ก", class: "form-label" %>
    <div class="d-flex flex-wrap gap-2">
      <% Tag.all.each do |tag| %>
        <div class="form-check form-check-inline">
          <%= check_box_tag "post[tag_ids][]", tag.id,
              @post.tag_ids.include?(tag.id),
              class: "btn-check",
              id: "tag_#{tag.id}" %>
          <%= label_tag "tag_#{tag.id}", tag.name, class: "btn btn-outline-secondary btn-sm" %>
        </div>
      <% end %>
    </div>
  </div>
  
  <!-- Cover Image -->
  <div class="mb-3">
    <%= form.label :cover_image, "ภาพปก", class: "form-label" %>
    <% if @post.cover_image.attached? %>
      <%= image_tag @post.cover_image.variant(resize_to_limit: [200, 100]), class: "mb-2 d-block" %>
    <% end %>
    <%= form.file_field :cover_image, accept: "image/*", class: "form-control" %>
  </div>
  
  <!-- Status -->
  <div class="mb-3">
    <%= form.label :status, "สถานะ", class: "form-label" %>
    <div>
      <% { "draft" => "บันทึกร่าง", "published" => "เผยแพร่" }.each do |value, label| %>
        <div class="form-check form-check-inline">
          <%= form.radio_button :status, value, class: "form-check-input" %>
          <%= form.label :"status_#{value}", label, class: "form-check-label" %>
        </div>
      <% end %>
    </div>
  </div>
  
  <!-- Published at (conditional) -->
  <div class="mb-3" id="published-at-field">
    <%= form.label :published_at, "วันที่เผยแพร่", class: "form-label" %>
    <%= form.datetime_local_field :published_at, class: "form-control" %>
  </div>
  
  <!-- Submit -->
  <div class="d-flex gap-2">
    <%= form.submit @post.new_record? ? "สร้างบทความ" : "อัปเดตบทความ",
        class: "btn btn-primary" %>
    <%= link_to "ยกเลิก",
        @post.new_record? ? posts_path : post_path(@post),
        class: "btn btn-outline-secondary" %>
  </div>
<% end %>
```

---

## สรุป

ใน Forms เราได้เรียนรู้:

1. **form_with** - Rails 7 form builder
2. **text_field, email_field, password_field** - input types ต่างๆ
3. **text_area** - multi-line input
4. **check_box, radio_button** - boolean selections
5. **select, collection_select** - dropdown menus
6. **file_field** - file uploads
7. **hidden_field** - hidden data
8. **submit, button** - form submission
9. **Validation errors** - แสดง errors ใน form
10. **search forms** - forms ที่ไม่ใช้ model
11. **Nested forms** - fields_for + accepts_nested_attributes_for
12. **CSRF protection** - security
13. **Turbo forms** - Rails 7 Hotwire integration
14. **Form helpers** - link_to, button_to
