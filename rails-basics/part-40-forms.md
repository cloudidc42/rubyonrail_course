# ตอนที่ 40: Forms (Steps 881-905)

## บทนำ

Forms เป็นส่วนสำคัญของ web application ที่ให้ผู้ใช้กรอกข้อมูล ใน Rails มี helper `form_with` ที่ช่วยสร้าง HTML forms อย่างปลอดภัยและง่ายดาย Rails 7 มาพร้อม Turbo ที่ทำให้ forms ทำงานได้แบบ Single Page Application โดยไม่ต้อง reload หน้า

---

## ขั้นตอนที่ 881: form_with พื้นฐาน

### form_with สำหรับ Model

```erb
<%# สร้าง form สำหรับ @post model %>
<%= form_with(model: @post) do |form| %>
  <%# Rails จะ detect อัตโนมัติ: %>
  <%# - ถ้า @post เป็น new record → action="/posts" method="POST" %>
  <%# - ถ้า @post เป็น existing record → action="/posts/1" method="POST" + hidden _method="PATCH" %>

  <div class="mb-3">
    <%= form.label :title, "หัวข้อ" %>
    <%= form.text_field :title, class: "form-control",
        placeholder: "กรอกหัวข้อบทความ" %>
    <% if @post.errors[:title].any? %>
      <div class="invalid-feedback d-block">
        <%= @post.errors[:title].first %>
      </div>
    <% end %>
  </div>

  <div class="mb-3">
    <%= form.label :body, "เนื้อหา" %>
    <%= form.text_area :body, class: "form-control", rows: 8 %>
  </div>

  <div class="mb-3 form-check">
    <%= form.check_box :published, class: "form-check-input" %>
    <%= form.label :published, "เผยแพร่บทความ", class: "form-check-label" %>
  </div>

  <div class="d-flex gap-2">
    <%= form.submit @post.new_record? ? "สร้างบทความ" : "อัปเดตบทความ",
        class: "btn btn-primary" %>
    <%= link_to "ยกเลิก", posts_path, class: "btn btn-secondary" %>
  </div>
<% end %>
```

### form_with สำหรับ URL (ไม่ใช้ Model)

```erb
<%# Search form %>
<%= form_with(url: search_path, method: :get) do |form| %>
  <div class="input-group">
    <%= form.text_field :q,
        value: params[:q],
        class: "form-control",
        placeholder: "ค้นหา..." %>
    <%= form.submit "ค้นหา", class: "btn btn-primary" %>
  </div>
<% end %>

<%# Custom endpoint %>
<%= form_with(url: "/api/submit", method: :post) do |form| %>
  <%= form.text_field :data %>
  <%= form.submit "ส่ง" %>
<% end %>
```

### form_with Options

```erb
<%# id สำหรับ CSS/JS %>
<%= form_with(model: @user, id: "user-form") do |f| %>

<%# class สำหรับ styling %>
<%= form_with(model: @post, class: "post-form needs-validation") do |f| %>

<%# html attributes %>
<%= form_with(model: @post,
    html: { "data-controller": "form", novalidate: true }) do |f| %>

<%# data attributes (Turbo) %>
<%= form_with(model: @post,
    data: { turbo: false }) do |f| %>
<%# ปิด Turbo สำหรับ form นี้ %>

<%# multipart สำหรับ file upload %>
<%= form_with(model: @post, multipart: true) do |f| %>
<%# Rails ตั้งค่า enctype="multipart/form-data" อัตโนมัติเมื่อมี file_field %>
```

---

## ขั้นตอนที่ 882: Input Types ทั้งหมด

### Text Inputs

```erb
<%= form_with(model: @user) do |f| %>
  <%# text_field - input type="text" %>
  <%= f.text_field :name, class: "form-control" %>
  <%= f.text_field :name, size: 30, maxlength: 100 %>

  <%# email_field - input type="email" %>
  <%= f.email_field :email, autocomplete: "email" %>

  <%# password_field - input type="password" (ไม่แสดงค่า) %>
  <%= f.password_field :password, autocomplete: "new-password" %>

  <%# text_area - textarea element %>
  <%= f.text_area :bio, rows: 4, cols: 50, class: "form-control" %>
  <%= f.text_area :description, size: "60x10" %>

  <%# hidden_field - input type="hidden" (ไม่แสดงใน UI) %>
  <%= f.hidden_field :user_id, value: current_user.id %>
  <%= f.hidden_field :token %>

  <%# number_field - input type="number" %>
  <%= f.number_field :age, min: 0, max: 150 %>
  <%= f.number_field :price, step: 0.01, min: 0 %>
  <%= f.number_field :quantity, value: 1 %>

  <%# telephone_field - input type="tel" %>
  <%= f.telephone_field :phone, pattern: "[0-9]{10}" %>
  <%# หรือ tel_field %>
  <%= f.tel_field :phone %>

  <%# url_field - input type="url" %>
  <%= f.url_field :website, placeholder: "https://example.com" %>

  <%# search_field - input type="search" %>
  <%= f.search_field :query, placeholder: "ค้นหา..." %>
<% end %>
```

### Date และ Time Inputs

```erb
<%= form_with(model: @event) do |f| %>
  <%# date_field - input type="date" %>
  <%= f.date_field :start_date, min: Date.today %>

  <%# time_field - input type="time" %>
  <%= f.time_field :event_time, min: "09:00", max: "18:00" %>

  <%# datetime_local_field - input type="datetime-local" %>
  <%= f.datetime_local_field :scheduled_at %>

  <%# month_field - input type="month" %>
  <%= f.month_field :birth_month %>

  <%# week_field - input type="week" %>
  <%= f.week_field :target_week %>
<% end %>
```

### Other Inputs

```erb
<%= form_with(model: @product) do |f| %>
  <%# range_field - input type="range" %>
  <%= f.range_field :rating, min: 1, max: 10, step: 1 %>

  <%# color_field - input type="color" %>
  <%= f.color_field :theme_color, value: "#3498db" %>

  <%# file_field - input type="file" %>
  <%= f.file_field :image %>
  <%= f.file_field :documents, multiple: true, accept: ".pdf,.doc" %>

  <%# check_box - checkbox %>
  <%= f.check_box :agree_terms %>
  <%= f.check_box :features, { multiple: true }, "feature1", false %>

  <%# radio_button %>
  <%= f.radio_button :gender, "male" %> ชาย
  <%= f.radio_button :gender, "female" %> หญิง
  <%= f.radio_button :gender, "other" %> อื่นๆ

  <%# submit %>
  <%= f.submit "บันทึก", class: "btn btn-primary" %>
  <%= f.submit "บันทึกร่าง", name: "commit", value: "draft" %>

  <%# button (submit alternative) %>
  <%= f.button "บันทึก", type: "submit", class: "btn btn-primary" %>
  <%= f.button do %>
    <i class="fas fa-save"></i> บันทึก
  <% end %>
<% end %>
```

---

## ขั้นตอนที่ 883: Select Helpers

### select พื้นฐาน

```erb
<%= form_with(model: @post) do |f| %>
  <%# select จาก array %>
  <%= f.select :status, ["draft", "published", "archived"] %>

  <%# select พร้อม labels %>
  <%= f.select :status, [["ร่าง", "draft"], ["เผยแพร่", "published"], ["เก็บถาวร", "archived"]] %>

  <%# select กับ selected value %>
  <%= f.select :status, [["ร่าง", "draft"], ["เผยแพร่", "published"]],
      { selected: "draft" } %>

  <%# include_blank - เพิ่ม empty option %>
  <%= f.select :category_id, Category.all.map { |c| [c.name, c.id] },
      { include_blank: "-- เลือกหมวดหมู่ --" } %>

  <%# prompt - เพิ่ม prompt option %>
  <%= f.select :country, Country.all.map { |c| [c.name, c.code] },
      { prompt: "เลือกประเทศ" } %>

  <%# selected กับ multiple %>
  <%= f.select :tag_ids, Tag.all.map { |t| [t.name, t.id] },
      {}, { multiple: true, size: 5 } %>
<% end %>
```

### collection_select

```erb
<%# collection_select - select จาก ActiveRecord collection %>
<%= f.collection_select :category_id,
    Category.order(:name),    # collection
    :id,                       # value method
    :name,                     # text method
    { include_blank: "เลือกหมวดหมู่", prompt: false },
    { class: "form-select" } %>

<%# grouped_collection_select %>
<%= f.grouped_collection_select :city_id,
    Country.order(:name),  # groups collection
    :cities,               # group method
    :name,                 # group label
    :id,                   # option value
    :name,                 # option text
    { include_blank: "เลือกเมือง" } %>
```

### select_tag (standalone, ไม่ใช้ form builder)

```erb
<%# ไม่ต้องใช้กับ form builder %>
<%= select_tag :color, options_for_select(["แดง", "เขียว", "น้ำเงิน"]) %>

<%= select_tag :status,
    options_for_select([["ใช้งาน", 1], ["ปิดใช้", 0]], 1),
    include_blank: "ทั้งหมด" %>
```

### check_box_group และ radio_button_group

```erb
<%# collection_check_boxes %>
<%= f.collection_check_boxes :tag_ids,
    Tag.all,          # collection
    :id,              # value method
    :name do |b|      # text method + block %>
  <div class="form-check">
    <%= b.check_box class: "form-check-input" %>
    <%= b.label class: "form-check-label" %>
  </div>
<% end %>

<%# collection_radio_buttons %>
<%= f.collection_radio_buttons :category_id,
    Category.all,
    :id,
    :name do |b| %>
  <div class="form-check">
    <%= b.radio_button class: "form-check-input" %>
    <%= b.label class: "form-check-label" %>
  </div>
<% end %>
```

---

## ขั้นตอนที่ 884: File Uploads

### File Upload พื้นฐาน

```erb
<%# ต้องใช้ multipart: true สำหรับ file upload %>
<%= form_with(model: @user, multipart: true) do |f| %>
  <div class="mb-3">
    <%= f.label :avatar, "รูปโปรไฟล์" %>
    <%= f.file_field :avatar, accept: "image/*", class: "form-control" %>
    <%# แสดงรูปปัจจุบัน %>
    <% if @user.avatar.attached? %>
      <div class="mt-2">
        <%= image_tag @user.avatar, width: 100, class: "rounded-circle" %>
        <label>
          <%= f.check_box :remove_avatar %>
          ลบรูปภาพ
        </label>
      </div>
    <% end %>
  </div>

  <%# Multiple files %>
  <div class="mb-3">
    <%= f.label :attachments, "ไฟล์แนบ" %>
    <%= f.file_field :attachments, multiple: true,
        accept: ".pdf,.doc,.docx,.jpg,.png" %>
  </div>

  <%= f.submit "บันทึก" %>
<% end %>
```

### Active Storage Setup

```ruby
# Gemfile
gem "image_processing", "~> 1.2"  # สำหรับ image variants

# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :documents

  # Validation
  validate :avatar_type_and_size

  private

  def avatar_type_and_size
    return unless avatar.attached?

    unless avatar.content_type.in?(%w[image/jpeg image/png image/gif])
      errors.add(:avatar, "ต้องเป็น JPEG, PNG หรือ GIF เท่านั้น")
    end

    if avatar.byte_size > 5.megabytes
      errors.add(:avatar, "ขนาดไม่เกิน 5MB")
    end
  end
end

# app/controllers/users_controller.rb
def user_params
  params.require(:user).permit(:name, :email, :avatar,
                                documents: [])  # array for multiple
end
```

### แสดง Uploaded Files

```erb
<%# แสดง image %>
<% if @user.avatar.attached? %>
  <%= image_tag @user.avatar %>
  <%= image_tag @user.avatar.variant(resize_to_fill: [200, 200]) %>
<% end %>

<%# Download link %>
<% @post.attachments.each do |attachment| %>
  <div>
    <%= link_to attachment.filename, rails_blob_path(attachment, disposition: "attachment") %>
    (<%= number_to_human_size(attachment.byte_size) %>)
  </div>
<% end %>
```

---

## ขั้นตอนที่ 885: Nested Forms

### accepts_nested_attributes_for

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :tags, dependent: :destroy
  accepts_nested_attributes_for :tags,
    allow_destroy: true,
    reject_if: :all_blank
end

# app/controllers/posts_controller.rb
def post_params
  params.require(:post).permit(
    :title, :body, :published,
    tags_attributes: [:id, :name, :_destroy]
  )
end
```

```erb
<%# app/views/posts/_form.html.erb %>
<%= form_with(model: @post) do |form| %>
  <%= form.text_field :title %>
  <%= form.text_area :body %>

  <%# Nested form สำหรับ tags %>
  <h4>แท็ก</h4>
  <%= form.fields_for :tags do |tag_form| %>
    <div class="tag-fields">
      <%= tag_form.text_field :name, class: "form-control" %>
      <label>
        <%= tag_form.check_box :_destroy %>
        ลบ
      </label>
    </div>
  <% end %>

  <button type="button" id="add-tag">เพิ่มแท็ก</button>

  <%= form.submit "บันทึก" %>
<% end %>
```

### Dynamic Nested Forms ด้วย Stimulus

```javascript
// app/javascript/controllers/nested_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["template", "container"]

  addItem() {
    const timestamp = new Date().getTime()
    const content = this.templateTarget.innerHTML.replace(/NEW_RECORD/g, timestamp)
    this.containerTarget.insertAdjacentHTML("beforeend", content)
  }

  removeItem(event) {
    const item = event.target.closest(".nested-item")
    const destroyField = item.querySelector("input[name*='_destroy']")

    if (destroyField) {
      destroyField.value = "1"
      item.style.display = "none"
    } else {
      item.remove()
    }
  }
}
```

```erb
<%# View ด้วย Stimulus %>
<div data-controller="nested-form">
  <div data-nested-form-target="container">
    <%= form.fields_for :tags do |tag_form| %>
      <div class="nested-item">
        <%= tag_form.text_field :name %>
        <button type="button" data-action="nested-form#removeItem">ลบ</button>
        <%= tag_form.hidden_field :_destroy %>
      </div>
    <% end %>
  </div>

  <%# Template สำหรับ new items %>
  <template data-nested-form-target="template">
    <div class="nested-item">
      <input type="text" name="post[tags_attributes][NEW_RECORD][name]">
      <button type="button" data-action="nested-form#removeItem">ลบ</button>
    </div>
  </template>

  <button type="button" data-action="nested-form#addItem">เพิ่มแท็ก</button>
</div>
```

### Nested Forms สำหรับ Line Items

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  has_many :line_items, dependent: :destroy
  accepts_nested_attributes_for :line_items,
    allow_destroy: true,
    reject_if: proc { |attrs| attrs[:product_id].blank? }

  def total_price
    line_items.sum { |li| li.quantity * li.unit_price }
  end
end

# app/controllers/orders_controller.rb
def new
  @order = Order.new
  3.times { @order.line_items.build }  # สร้าง 3 empty line items
end

def order_params
  params.require(:order).permit(
    :customer_name, :shipping_address,
    line_items_attributes: [
      :id, :product_id, :quantity, :unit_price, :_destroy
    ]
  )
end
```

---

## ขั้นตอนที่ 886: CSRF และ Security

### CSRF Protection

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # เปิดใช้ CSRF protection (default)
  protect_from_forgery with: :exception
  # หรือ
  protect_from_forgery with: :reset_session
  # หรือ (สำหรับ API)
  protect_from_forgery with: :null_session
end
```

```erb
<%# csrf_meta_tags ใน layout %>
<head>
  <%= csrf_meta_tags %>
  <%# สร้าง: %>
  <%# <meta name="csrf-param" content="authenticity_token"> %>
  <%# <meta name="csrf-token" content="...token..."> %>
</head>

<%# form_with เพิ่ม CSRF token อัตโนมัติ %>
<%= form_with(model: @post) do |f| %>
  <%# Rails เพิ่ม hidden field: %>
  <%# <input type="hidden" name="authenticity_token" value="..."> %>
<% end %>
```

### ปิด CSRF สำหรับ API

```ruby
class Api::BaseController < ApplicationController
  skip_before_action :verify_authenticity_token

  # หรือสำหรับ specific actions
  skip_before_action :verify_authenticity_token, only: [:webhook]
end
```

### Strong Parameters

```ruby
# app/controllers/posts_controller.rb
def create
  @post = Post.new(post_params)
  # ...
end

private

def post_params
  params.require(:post).permit(
    :title,
    :body,
    :published,
    :category_id,
    tag_ids: [],
    images: [],
    metadata: [:key, :value]
  )
end
```

---

## ขั้นตอนที่ 887: Turbo และ Forms

### Turbo Form Submission

```erb
<%# ส่ง form ด้วย Turbo (default ใน Rails 7) %>
<%= form_with(model: @post) do |f| %>
  <%# ส่ง AJAX + update page บางส่วน %>
<% end %>

<%# ปิด Turbo สำหรับ form นี้ %>
<%= form_with(model: @post, data: { turbo: false }) do |f| %>
<% end %>
```

### Turbo Stream Response

```ruby
# app/controllers/posts_controller.rb
def create
  @post = Post.new(post_params)

  if @post.save
    respond_to do |format|
      format.html { redirect_to @post, notice: "บันทึกสำเร็จ" }
      format.turbo_stream  # ค้นหา create.turbo_stream.erb
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

```erb
<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.prepend "posts" do %>
  <%= render @post %>
<% end %>

<%= turbo_stream.update "flash" do %>
  <div class="alert alert-success">บทความถูกสร้างแล้ว</div>
<% end %>
```

### Turbo Frame Forms

```erb
<%# กรอบ frame ที่ update เฉพาะส่วน %>
<%= turbo_frame_tag "post_form" do %>
  <%= form_with(model: @post) do |f| %>
    <%= f.text_field :title %>
    <%= f.submit %>
  <% end %>
<% end %>
```

---

## ขั้นตอนที่ 888: Displaying Errors

### Error Display Patterns

```erb
<%# Pattern 1: Alert block ก่อน form %>
<% if @post.errors.any? %>
  <div class="alert alert-danger" role="alert">
    <h4 class="alert-heading">
      พบ <%= pluralize(@post.errors.count, "ข้อผิดพลาด") %>:
    </h4>
    <ul class="mb-0">
      <% @post.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<%# Pattern 2: Inline errors ข้าง field %>
<div class="mb-3">
  <%= f.label :email %>
  <%= f.email_field :email,
      class: "form-control #{@user.errors[:email].any? ? "is-invalid" : ""}" %>
  <%= f.text_field :email,
      class: ["form-control", ("is-invalid" if @user.errors[:email].any?)] %>
  <div class="invalid-feedback">
    <%= @user.errors[:email].first %>
  </div>
</div>

<%# Pattern 3: Helper method %>
```

```ruby
# app/helpers/forms_helper.rb
module FormsHelper
  def form_group(form, field, options = {}, &block)
    has_error = form.object.errors[field].any?

    content_tag(:div, class: "mb-3") do
      concat form.label(field, options[:label], class: "form-label")
      concat capture(&block)
      if has_error
        concat content_tag(:div, form.object.errors[field].first,
                           class: "invalid-feedback d-block")
      end
    end
  end
end
```

```erb
<%# ใช้ helper %>
<%= form_group(f, :email, label: "Email") do %>
  <%= f.email_field :email,
      class: ["form-control", ("is-invalid" if @user.errors[:email].any?)] %>
<% end %>
```

---

## ขั้นตอนที่ 889: Form Object Pattern

### Form Object

```ruby
# app/forms/registration_form.rb
class RegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :name, :string
  attribute :email, :string
  attribute :password, :string
  attribute :password_confirmation, :string
  attribute :terms_accepted, :boolean, default: false

  validates :name, presence: true
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :password_confirmation, presence: true
  validate :passwords_match
  validates :terms_accepted, acceptance: true

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      user = User.create!(
        name: name,
        email: email,
        password: password
      )
      WelcomeMailer.with(user: user).welcome_email.deliver_later
      user
    end
  rescue ActiveRecord::RecordInvalid => e
    e.record.errors.each do |error|
      errors.add(error.attribute, error.message)
    end
    false
  end

  private

  def passwords_match
    if password != password_confirmation
      errors.add(:password_confirmation, "รหัสผ่านไม่ตรงกัน")
    end
  end
end
```

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  def new
    @form = RegistrationForm.new
  end

  def create
    @form = RegistrationForm.new(registration_params)

    if @form.save
      redirect_to root_path, notice: "สมัครสมาชิกสำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def registration_params
    params.require(:registration_form).permit(
      :name, :email, :password, :password_confirmation, :terms_accepted
    )
  end
end
```

```erb
<%# app/views/registrations/new.html.erb %>
<%= form_with(model: @form, url: registrations_path) do |f| %>
  <%= f.text_field :name %>
  <%= f.email_field :email %>
  <%= f.password_field :password %>
  <%= f.password_field :password_confirmation %>
  <%= f.check_box :terms_accepted %>
  <%= f.label :terms_accepted, "ยอมรับเงื่อนไขการใช้งาน" %>
  <%= f.submit "สมัครสมาชิก" %>
<% end %>
```

---

## ขั้นตอนที่ 890: Advanced Form Techniques

### Stimulus Controllers สำหรับ Form

```javascript
// app/javascript/controllers/character_counter_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "counter"]
  static values = { max: Number }

  connect() {
    this.update()
  }

  update() {
    const length = this.inputTarget.value.length
    const remaining = this.maxValue - length
    this.counterTarget.textContent = `${remaining} ตัวอักษรที่เหลือ`

    if (remaining < 0) {
      this.counterTarget.classList.add("text-danger")
    } else {
      this.counterTarget.classList.remove("text-danger")
    }
  }
}
```

```erb
<div data-controller="character-counter"
     data-character-counter-max-value="200">
  <%= f.text_area :bio,
      class: "form-control",
      maxlength: 200,
      "data-character-counter-target": "input",
      "data-action": "input->character-counter#update" %>
  <small data-character-counter-target="counter" class="text-muted"></small>
</div>
```

### Auto-save Form

```javascript
// app/javascript/controllers/autosave_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 2000 } }

  connect() {
    this.timeout = null
  }

  save() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }

  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

```erb
<%= form_with(model: @draft, data: { controller: "autosave" }) do |f| %>
  <%= f.text_field :title,
      data: { action: "input->autosave#save" } %>
  <%= f.text_area :content,
      data: { action: "input->autosave#save" } %>
  <small class="text-muted">บันทึกอัตโนมัติ</small>
<% end %>
```

---

## แบบฝึกหัดตอนที่ 40 (25 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** สร้าง form สำหรับสร้าง/แก้ไข Post ด้วย `form_with(model: @post)` ที่มี: title, body, published checkbox, category select

**ข้อ 2:** สร้าง search form ที่ใช้ GET method พร้อม placeholder และ submit button

**ข้อ 3:** ใช้ `collection_select` เพื่อสร้าง dropdown จาก Category model

**ข้อ 4:** สร้าง radio buttons สำหรับ gender: ชาย, หญิง, ไม่ระบุ

**ข้อ 5:** สร้าง date field สำหรับ birth_date พร้อม max date เป็นวันนี้

**ข้อ 6:** สร้าง file upload field ที่ accept เฉพาะ image files

**ข้อ 7:** แสดง validation errors ด้วย Bootstrap `is-invalid` class

**ข้อ 8:** สร้าง hidden field สำหรับ referral_code จาก params

**ข้อ 9:** ใช้ `number_field` พร้อม min, max, step สำหรับ quantity input

**ข้อ 10:** สร้าง select สำหรับ tags ที่ multiple: true

### ระดับกลาง

**ข้อ 11:** Implement nested form สำหรับ Order พร้อม LineItems ที่ dynamic add/remove ด้วย Stimulus

**ข้อ 12:** สร้าง Form Object สำหรับ registration ที่มี email, password, confirm password, terms

**ข้อ 13:** Implement file upload พร้อม Active Storage และ validation ประเภทและขนาด

**ข้อ 14:** สร้าง character counter สำหรับ text area ด้วย Stimulus

**ข้อ 15:** สร้าง form ที่ respond_to HTML และ Turbo Stream

**ข้อ 16:** ใช้ `accepts_nested_attributes_for` กับ `reject_if: :all_blank`

**ข้อ 17:** สร้าง dependent dropdowns (country → state → city) ด้วย Stimulus และ Turbo Frame

**ข้อ 18:** Implement password strength indicator ด้วย Stimulus

**ข้อ 19:** สร้าง multi-step form (wizard) ด้วย state machine

**ข้อ 20:** ทดสอบ CSRF protection: verify token ใน non-GET requests

### ระดับสูง

**ข้อ 21:** สร้าง Auto-save form ที่ save draft ทุก 3 วินาที

**ข้อ 22:** Implement drag-and-drop file upload ด้วย Stimulus

**ข้อ 23:** สร้าง rich text editor integration (Trix ผ่าน Action Text)

**ข้อ 24:** สร้าง form ที่ validate real-time ด้วย Turbo Frame

**ข้อ 25:** เขียน system spec ครอบคลุม form validation, submission, และ error display

---

## สรุปตอนที่ 40

| หัวข้อ | สิ่งสำคัญ |
|--------|-----------|
| form_with | model-backed vs URL-based |
| Input Types | text, email, password, number, date, file, etc. |
| Select | select, collection_select, grouped_collection_select |
| File Upload | file_field + Active Storage |
| Nested Forms | accepts_nested_attributes_for + fields_for |
| CSRF | authenticity_token, protect_from_forgery |
| Turbo | form submission, turbo_stream response |
| Error Display | Bootstrap integration, inline errors |
| Form Object | สำหรับ complex forms |

**Key Points:**
1. `form_with` detect create/update อัตโนมัติจาก model
2. ใช้ Strong Parameters เสมอ
3. CSRF protection เปิดใช้ default
4. Turbo ทำให้ forms ทำงานแบบ AJAX โดยอัตโนมัติ
5. Form Object pattern สำหรับ business logic ที่ซับซ้อน

ตอนถัดไป: **ตอนที่ 41** - Sessions and Cookies
