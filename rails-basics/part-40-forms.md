# Part 40: Forms (ขั้นตอนที่ 881-905)

## บทนำ

Forms เป็นส่วนสำคัญของ web applications Rails มี form helpers ที่ช่วยสร้าง forms ที่ผูกกับ models อัตโนมัติ รองรับ CSRF protection, nested forms, file uploads และ Turbo (Rails 7)

---

## ขั้นตอนที่ 881: form_with พื้นฐาน

```ruby
# form_with เป็น helper หลักใน Rails 5.1+
# รวม form_for และ form_tag เข้าด้วยกัน

# Form กับ model:
<%= form_with model: @article do |f| %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>
# สร้าง:
# <form action="/articles" method="post" data-remote="true">
#   <input type="hidden" name="_method" value="post">
#   <input type="hidden" name="authenticity_token" value="...">
#   <input type="text" name="article[title]" id="article_title">
#   <input type="submit" value="Create Article">
# </form>

# Form กับ URL:
<%= form_with url: search_path, method: :get do |f| %>
  <%= f.text_field :query, placeholder: "ค้นหา..." %>
  <%= f.submit "ค้นหา" %>
<% end %>

# Form สำหรับ new record:
@article = Article.new
<%= form_with model: @article do |f| %>
  <%# action: /articles, method: POST %>
<% end %>

# Form สำหรับ existing record:
@article = Article.find(1)
<%= form_with model: @article do |f| %>
  <%# action: /articles/1, method: PATCH %>
<% end %>

# Nested resources:
<%= form_with model: [@user, @article] do |f| %>
  <%# action: /users/1/articles, method: POST %>
<% end %>
```

---

## ขั้นตอนที่ 882: Text Fields

```ruby
<%= form_with model: @user do |f| %>
  <!-- Text field -->
  <%= f.text_field :name,
    placeholder: "ชื่อ-นามสกุล",
    class: "form-control",
    maxlength: 100 %>

  <!-- Email field -->
  <%= f.email_field :email,
    placeholder: "email@example.com",
    autocomplete: "email" %>

  <!-- Password field -->
  <%= f.password_field :password,
    placeholder: "รหัสผ่านอย่างน้อย 8 ตัว",
    minlength: 8 %>

  <!-- Search field -->
  <%= f.search_field :query, placeholder: "ค้นหา..." %>

  <!-- Tel field -->
  <%= f.telephone_field :phone, placeholder: "0812345678" %>

  <!-- URL field -->
  <%= f.url_field :website, placeholder: "https://example.com" %>

  <!-- Number field -->
  <%= f.number_field :age, min: 18, max: 99, step: 1 %>

  <!-- Range field -->
  <%= f.range_field :rating, min: 1, max: 5, step: 0.5 %>

  <!-- Hidden field -->
  <%= f.hidden_field :status, value: "pending" %>

  <!-- Textarea -->
  <%= f.text_area :bio,
    rows: 5,
    placeholder: "เล่าเกี่ยวกับตัวเอง...",
    class: "form-control" %>

  <%= f.submit "บันทึก", class: "btn btn-primary" %>
<% end %>
```

---

## ขั้นตอนที่ 883: Checkboxes และ Radio Buttons

```ruby
<%= form_with model: @article do |f| %>
  <!-- Single checkbox -->
  <%= f.check_box :featured %>
  <%= f.label :featured, "บทความแนะนำ" %>

  <!-- Checkbox กับ custom values -->
  <%= f.check_box :agree, { checked: false }, "yes", "no" %>

  <!-- Radio buttons -->
  <%= f.radio_button :status, "draft" %>
  <%= f.label :status_draft, "Draft" %>

  <%= f.radio_button :status, "published" %>
  <%= f.label :status_published, "Published" %>

  <%= f.radio_button :status, "archived" %>
  <%= f.label :status_archived, "Archived" %>
<% end %>

<!-- Collection of checkboxes -->
<%= form_with model: @article do |f| %>
  <%= f.collection_check_boxes :tag_ids, Tag.all, :id, :name do |b| %>
    <div class="flex items-center gap-2">
      <%= b.check_box(class: "form-checkbox") %>
      <%= b.label(class: "cursor-pointer") %>
    </div>
  <% end %>
<% end %>

<!-- Collection of radio buttons -->
<%= form_with model: @article do |f| %>
  <%= f.collection_radio_buttons :category_id, Category.all, :id, :name do |b| %>
    <div class="flex items-center gap-2">
      <%= b.radio_button(class: "form-radio") %>
      <%= b.label(class: "cursor-pointer") %>
    </div>
  <% end %>
<% end %>
```

---

## ขั้นตอนที่ 884: Select Fields

```ruby
<%= form_with model: @article do |f| %>
  <!-- Select จาก array -->
  <%= f.select :status,
    [["Draft", "draft"], ["Published", "published"], ["Archived", "archived"]],
    { prompt: "-- เลือกสถานะ --" },
    { class: "form-select" } %>

  <!-- Select จาก hash -->
  <%= f.select :country,
    { "Thailand" => "TH", "USA" => "US", "Japan" => "JP" },
    { include_blank: "เลือกประเทศ" } %>

  <!-- Select จาก range -->
  <%= f.select :year,
    (Date.today.year - 5)..(Date.today.year + 5),
    { selected: Date.today.year } %>

  <!-- collection_select -->
  <%= f.collection_select :category_id,
    Category.all,
    :id,            # value
    :name,          # display text
    { prompt: "เลือกหมวดหมู่" },
    { class: "form-select" } %>

  <!-- grouped_collection_select -->
  <%= f.grouped_collection_select :subcategory_id,
    Category.root.includes(:children),  # groups
    :children,     # method สำหรับ group items
    :name,         # group label method
    :id,           # option value method
    :name,         # option text method
    { prompt: "เลือกหมวดหมู่ย่อย" } %>

  <!-- time_zone_select -->
  <%= f.time_zone_select :timezone,
    ActiveSupport::TimeZone.all,
    { default: "Bangkok" },
    { class: "form-select" } %>

  <!-- select_tag (not model-bound) -->
  <%= select_tag :sort_by,
    options_for_select([
      ["ล่าสุด", "newest"],
      ["เก่าสุด", "oldest"],
      ["ยอดนิยม", "popular"]
    ], params[:sort_by]),
    class: "form-select" %>
<% end %>
```

---

## ขั้นตอนที่ 885: Date และ Time Fields

```ruby
<%= form_with model: @event do |f| %>
  <!-- Date field (HTML5) -->
  <%= f.date_field :event_date %>

  <!-- Time field (HTML5) -->
  <%= f.time_field :start_time %>

  <!-- Datetime local field -->
  <%= f.datetime_local_field :starts_at %>

  <!-- Month field -->
  <%= f.month_field :birth_month %>

  <!-- Week field -->
  <%= f.week_field :week %>

  <!-- date_select - dropdown selects -->
  <%= f.date_select :birthday,
    order: [:day, :month, :year],
    start_year: 1950,
    end_year: Date.today.year,
    include_blank: true %>

  <!-- time_select -->
  <%= f.time_select :meeting_time,
    minute_step: 15,
    include_blank: true %>

  <!-- datetime_select -->
  <%= f.datetime_select :published_at,
    include_blank: true,
    minute_step: 30 %>
<% end %>
```

---

## ขั้นตอนที่ 886: File Upload

```ruby
# Model:
class Article < ApplicationRecord
  has_one_attached :featured_image
  has_many_attached :attachments

  validates :featured_image,
    content_type: { in: ["image/png", "image/jpg", "image/jpeg", "image/gif"],
                    message: "ต้องเป็นไฟล์รูปภาพ" },
    size: { less_than: 5.megabytes, message: "ต้องไม่เกิน 5MB" }
end

# Form:
<%= form_with model: @article, html: { enctype: "multipart/form-data" } do |f| %>
  <!-- Single file -->
  <%= f.file_field :featured_image,
    accept: "image/*",
    class: "form-file" %>

  <!-- Multiple files -->
  <%= f.file_field :attachments,
    multiple: true,
    accept: ".pdf,.doc,.docx" %>

  <!-- File กับ preview (JavaScript) -->
  <%= f.file_field :featured_image,
    accept: "image/*",
    data: { controller: "image-preview" } %>
<% end %>

# Controller:
def create
  @article = Article.new(article_params)
  if @article.save
    redirect_to @article
  else
    render :new, status: :unprocessable_entity
  end
end

private

def article_params
  params.require(:article).permit(:title, :body, :featured_image,
                                    attachments: [])
end

# View แสดงไฟล์:
<% if @article.featured_image.attached? %>
  <%= image_tag @article.featured_image, class: "w-full h-64 object-cover" %>
  <%= link_to "ลบรูปภาพ",
    rails_storage_proxy_path(@article.featured_image),
    method: :delete,
    data: { confirm: "ต้องการลบรูปภาพ?" } %>
<% end %>

# Image variant:
<%= image_tag @article.featured_image.variant(resize_to_fill: [800, 400]) %>
<%= image_tag @article.featured_image.variant(resize_to_limit: [400, 400]) %>
```

---

## ขั้นตอนที่ 887: Nested Forms

```ruby
# Model:
class Order < ApplicationRecord
  has_many :order_items, dependent: :destroy
  accepts_nested_attributes_for :order_items,
    allow_destroy: true,
    reject_if: :all_blank

  validates :customer_name, presence: true
end

class OrderItem < ApplicationRecord
  belongs_to :order
  belongs_to :product
  validates :quantity, numericality: { greater_than: 0 }
  validates :unit_price, numericality: { greater_than: 0 }
end

# View:
<%= form_with model: @order do |f| %>
  <%= f.text_field :customer_name, placeholder: "ชื่อลูกค้า" %>

  <div id="order-items">
    <%= f.fields_for :order_items do |item_form| %>
      <%= render "order_item_fields", f: item_form %>
    <% end %>
  </div>

  <button type="button" id="add-item">เพิ่มรายการ</button>

  <%= f.submit "สั่งซื้อ" %>
<% end %>

# app/views/orders/_order_item_fields.html.erb
<div class="order-item-fields">
  <%= f.hidden_field :_destroy, class: "destroy-field" %>

  <%= f.collection_select :product_id, Product.all, :id, :name %>
  <%= f.number_field :quantity, min: 1, value: 1 %>
  <%= f.number_field :unit_price, min: 0, step: "0.01" %>

  <button type="button" class="remove-item">ลบ</button>
</div>

# Dynamic nested forms ด้วย Stimulus (Rails 7):
# app/javascript/controllers/nested_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["template", "container"]

  addItem() {
    const content = this.templateTarget.innerHTML.replace(
      /NEW_RECORD/g,
      new Date().getTime()
    )
    this.containerTarget.insertAdjacentHTML("beforeend", content)
  }

  removeItem(event) {
    const item = event.target.closest(".nested-fields")
    const destroyField = item.querySelector("[data-destroy]")
    if (destroyField) {
      destroyField.value = "1"
      item.style.display = "none"
    } else {
      item.remove()
    }
  }
}

# Controller:
def order_params
  params.require(:order).permit(
    :customer_name, :notes,
    order_items_attributes: [:id, :product_id, :quantity, :unit_price, :_destroy]
  )
end
```

---

## ขั้นตอนที่ 888: CSRF Protection

```ruby
# Rails รวม CSRF protection อัตโนมัติ
# form_with สร้าง authenticity_token ให้เสมอ

# Application Controller:
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception  # default: raise InvalidAuthenticityToken

  # หรือ:
  protect_from_forgery with: :null_session  # สำหรับ API
  protect_from_forgery with: :reset_session

  # ข้าม CSRF สำหรับ specific actions:
  skip_forgery_protection only: [:webhook_callback]
  skip_before_action :verify_authenticity_token, only: [:webhook_callback]
end

# สำหรับ AJAX requests:
# Rails ส่ง X-CSRF-Token header อัตโนมัติถ้าใช้ Rails UJS
# สำหรับ Fetch API ต้องส่ง manually:

// JavaScript:
const token = document.querySelector('meta[name="csrf-token"]').content
fetch("/articles", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-CSRF-Token": token
  },
  body: JSON.stringify({ article: { title: "Test" } })
})

# Meta tag ใน layout:
# application.html.erb มี:
# <%= csrf_meta_tags %>
# สร้าง: <meta name="csrf-param" content="authenticity_token">
#         <meta name="csrf-token" content="xyz...">
```

---

## ขั้นตอนที่ 889: Form Error Handling

```ruby
# Model:
class Article < ApplicationRecord
  validates :title, presence: true, length: { minimum: 5 }
  validates :body, presence: true
end

# Controller:
class ArticlesController < ApplicationController
  def new
    @article = Article.new
  end

  def create
    @article = Article.new(article_params)
    if @article.save
      redirect_to @article, notice: "สร้างบทความสำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def article_params
    params.require(:article).permit(:title, :body, :status)
  end
end

# View - แสดง field-specific errors:
<%= form_with model: @article do |f| %>
  <div class="field <%= "field_with_errors" if @article.errors[:title].any? %>">
    <%= f.label :title, "หัวข้อ" %>
    <%= f.text_field :title, class: "form-control #{"is-invalid" if @article.errors[:title].any?}" %>
    <% @article.errors[:title].each do |error| %>
      <div class="invalid-feedback"><%= error %></div>
    <% end %>
  </div>

  <div class="field">
    <%= f.label :body, "เนื้อหา" %>
    <%= f.text_area :body, class: "form-control #{"is-invalid" if @article.errors[:body].any?}" %>
    <% @article.errors[:body].each do |error| %>
      <div class="invalid-feedback"><%= error %></div>
    <% end %>
  </div>

  <%= f.submit %>
<% end %>

# Reusable error helper:
# app/helpers/form_helper.rb
module FormHelper
  def field_with_error(form, field, &block)
    content_tag(:div, class: "form-field #{form.object.errors[field].any? ? "has-error" : ""}") do
      concat(capture(&block))
      if form.object.errors[field].any?
        concat(content_tag(:ul, class: "errors") {
          form.object.errors[field].map { |e|
            content_tag(:li, e)
          }.join.html_safe
        })
      end
    end
  end
end
```

---

## ขั้นตอนที่ 890: Turbo และ Forms (Rails 7)

```ruby
# Rails 7 ใช้ Turbo แทน Rails UJS
# form_with ทำงานร่วมกับ Turbo อัตโนมัติ

# Turbo Drive - navigate without full page reload
<%= form_with model: @article do |f| %>
  <%# Turbo จัดการ form submission อัตโนมัติ %>
<% end %>

# Controller สำหรับ Turbo:
def create
  @article = Article.new(article_params)
  if @article.save
    redirect_to @article  # Turbo จัดการ redirect
  else
    render :new, status: :unprocessable_entity  # สำคัญ! ต้องส่ง 422
  end
end

# Turbo Stream response:
def create
  @article = Article.new(article_params)
  respond_to do |format|
    if @article.save
      format.turbo_stream {
        render turbo_stream: [
          turbo_stream.prepend("articles", partial: "articles/article",
            locals: { article: @article }),
          turbo_stream.replace("article-form", partial: "articles/form",
            locals: { article: Article.new })
        ]
      }
      format.html { redirect_to articles_path }
    else
      format.turbo_stream {
        render turbo_stream: turbo_stream.replace(
          "article-form",
          partial: "articles/form",
          locals: { article: @article }
        )
      }
      format.html { render :new, status: :unprocessable_entity }
    end
  end
end

# View กับ Turbo Frame:
<%= turbo_frame_tag "article-form" do %>
  <%= form_with model: @article do |f| %>
    <%= f.text_field :title %>
    <%= f.text_area :body %>
    <%= f.submit %>
  <% end %>
<% end %>

<div id="articles">
  <%= render @articles %>
</div>

# Disable Turbo สำหรับ form:
<%= form_with model: @article, data: { turbo: false } do |f| %>
  <%# Full page reload %>
<% end %>
```

---

## ขั้นตอนที่ 891: Custom FormBuilder

```ruby
# app/helpers/form_builder_helper.rb
class ApplicationFormBuilder < ActionView::Helpers::FormBuilder
  def text_field_with_label(method, options = {})
    label_text = options.delete(:label) || method.to_s.humanize
    error_class = object.errors[method].any? ? "border-red-500" : ""
    
    @template.content_tag(:div, class: "form-group mb-4") do
      concat label(method, label_text, class: "block text-sm font-medium text-gray-700 mb-1")
      concat text_field(method, {
        class: "w-full px-3 py-2 border rounded-md #{error_class}",
        **options
      })
      
      if object.errors[method].any?
        concat @template.content_tag(:p, 
          object.errors[method].first,
          class: "text-red-500 text-sm mt-1"
        )
      end
    end
  end

  def submit_button(text = "บันทึก", options = {})
    @template.content_tag(:div, class: "form-actions mt-6") do
      submit(text, {
        class: "bg-blue-600 text-white px-6 py-2 rounded-md hover:bg-blue-700",
        **options
      })
    end
  end
end

# config/application.rb หรือ initializer:
ActionView::Base.default_form_builder = ApplicationFormBuilder

# ใช้งาน:
<%= form_with model: @user, builder: ApplicationFormBuilder do |f| %>
  <%= f.text_field_with_label :name, placeholder: "ชื่อของคุณ" %>
  <%= f.text_field_with_label :email, label: "อีเมล", type: "email" %>
  <%= f.submit_button "สร้างบัญชี" %>
<% end %>
```

---

## แบบฝึกหัด Part 40 (ขั้นตอนที่ 881-905)

**ข้อ 1:** สร้าง form สำหรับ Article พร้อม error handling

```erb
<%# คำตอบ: app/views/articles/_form.html.erb %>
<%= form_with model: article, class: "space-y-4" do |f| %>
  <% if article.errors.any? %>
    <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded">
      <ul>
        <% article.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= f.label :title, "หัวข้อ", class: "block font-medium" %>
    <%= f.text_field :title, class: "w-full border rounded px-3 py-2" %>
  </div>

  <div>
    <%= f.label :body, "เนื้อหา", class: "block font-medium" %>
    <%= f.text_area :body, rows: 8, class: "w-full border rounded px-3 py-2" %>
  </div>

  <div>
    <%= f.label :status, "สถานะ", class: "block font-medium" %>
    <%= f.select :status,
      [["Draft", "draft"], ["Published", "published"]],
      { prompt: "เลือกสถานะ" },
      { class: "w-full border rounded px-3 py-2" } %>
  </div>

  <%= f.submit article.new_record? ? "สร้างบทความ" : "บันทึกการแก้ไข",
    class: "bg-blue-600 text-white px-6 py-2 rounded hover:bg-blue-700" %>
<% end %>
```

**ข้อ 2:** สร้าง file upload form

```erb
<%# คำตอบ %>
<%= form_with model: @product do |f| %>
  <%= f.text_field :name %>

  <div>
    <%= f.label :images, "รูปภาพ (เลือกได้หลายรูป)" %>
    <%= f.file_field :images, multiple: true, accept: "image/*" %>
  </div>

  <%= f.submit %>
<% end %>
```

**ข้อ 3:** Turbo Stream form response

```ruby
# คำตอบ:
# controller:
def create
  @comment = @article.comments.new(comment_params.merge(user: current_user))
  if @comment.save
    respond_to do |format|
      format.turbo_stream {
        render turbo_stream: [
          turbo_stream.prepend("comments", partial: "comments/comment",
            locals: { comment: @comment }),
          turbo_stream.replace("comment-form",
            partial: "comments/form",
            locals: { article: @article, comment: Comment.new })
        ]
      }
      format.html { redirect_to @article }
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

**ข้อ 4:** สร้าง nested form สำหรับ Survey และ Questions

```erb
<%# คำตอบ: app/views/surveys/_form.html.erb %>
<%= form_with model: @survey do |f| %>
  <%= f.text_field :title, placeholder: "ชื่อแบบสอบถาม" %>

  <div id="questions-container">
    <%= f.fields_for :questions do |qf| %>
      <div class="question-fields border p-4 rounded mb-2">
        <%= qf.hidden_field :_destroy, class: "destroy-hidden" %>
        <%= qf.text_field :text, placeholder: "คำถาม" %>
        <%= qf.select :question_type,
          [["ข้อความ", "text"], ["เลือก", "choice"], ["คะแนน", "rating"]] %>
        <button type="button" onclick="removeQuestion(this)">ลบ</button>
      </div>
    <% end %>
  </div>

  <button type="button" id="add-question">+ เพิ่มคำถาม</button>

  <%= f.submit "บันทึกแบบสอบถาม" %>
<% end %>
```

**ข้อ 5:** CSRF ใน AJAX request

```javascript
// คำตอบ:
// ใช้ใน Stimulus controller หรือ vanilla JS
const csrfToken = document.querySelector('meta[name="csrf-token"]').content

async function createArticle(data) {
  const response = await fetch("/articles", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-CSRF-Token": csrfToken,
      "Accept": "application/json"
    },
    body: JSON.stringify({ article: data })
  })
  return response.json()
}
```

---

## สรุป Part 40

ในบทนี้เราได้เรียนรู้:

1. **form_with** - helper หลักสำหรับสร้าง forms ใน Rails 7
2. **Text Fields** - text, email, password, search, tel, url, number, range, hidden, textarea
3. **Checkboxes และ Radio Buttons** - check_box, radio_button, collection_check_boxes, collection_radio_buttons
4. **Select Fields** - select, collection_select, grouped_collection_select
5. **Date/Time Fields** - date_field, time_field, datetime_local_field, date_select, time_select
6. **File Upload** - file_field กับ Active Storage
7. **Nested Forms** - fields_for กับ accepts_nested_attributes_for
8. **CSRF Protection** - authenticity_token, skip_forgery_protection
9. **Error Handling** - แสดง errors ใน form
10. **Turbo Integration** - form กับ Turbo Drive, Turbo Streams (Rails 7)
11. **Custom FormBuilder** - สร้าง reusable form builder

Forms เป็นส่วนสำคัญที่เชื่อม user interface กับ server-side logic การเข้าใจ form helpers อย่างลึกซึ้งจะช่วยให้พัฒนา web applications ได้รวดเร็วและมีประสิทธิภาพ

---

*จบ Rails Basics (Part 31-40)*
