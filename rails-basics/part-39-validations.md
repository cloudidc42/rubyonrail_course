# Part 39: Validations (ขั้นตอนที่ 861-880)

## บทนำ

Validations คือกลไกที่ใช้ตรวจสอบข้อมูลก่อนที่จะบันทึกลงฐานข้อมูล Rails มี validation helpers ที่ใช้งานง่าย พร้อมกับระบบ custom validations ที่ยืดหยุ่น

---

## ขั้นตอนที่ 861: Validation Helpers พื้นฐาน

### presence

```ruby
class Article < ApplicationRecord
  validates :title, presence: true
  validates :body, presence: true
  validates :user, presence: true  # ตรวจสอบ association ด้วย

  # หลาย fields พร้อมกัน
  validates :title, :body, :status, presence: true
end

article = Article.new
article.valid?   # => false
article.errors[:title]  # => ["can't be blank"]
article.errors.full_messages  # => ["Title can't be blank", "Body can't be blank"]
```

### length

```ruby
class User < ApplicationRecord
  validates :username, length: { minimum: 3 }
  validates :username, length: { maximum: 50 }
  validates :username, length: { in: 3..50 }
  validates :username, length: { is: 10 }  # ต้องเท่ากับ 10

  validates :bio, length: {
    maximum: 500,
    too_long: "ไม่เกิน %{count} ตัวอักษร"
  }

  validates :pin, length: {
    minimum: 4,
    maximum: 6,
    too_short: "PIN ต้องมีอย่างน้อย %{count} หลัก",
    too_long: "PIN ต้องไม่เกิน %{count} หลัก"
  }
end
```

### numericality

```ruby
class Product < ApplicationRecord
  validates :price, numericality: true                     # ตัวเลขเท่านั้น
  validates :price, numericality: { greater_than: 0 }      # > 0
  validates :price, numericality: { greater_than_or_equal_to: 0 }  # >= 0
  validates :price, numericality: { less_than: 10000 }     # < 10000
  validates :price, numericality: { less_than_or_equal_to: 9999 }  # <= 9999
  validates :quantity, numericality: { only_integer: true } # ต้องเป็น integer
  validates :discount, numericality: {
    greater_than_or_equal_to: 0,
    less_than_or_equal_to: 100,
    message: "ต้องอยู่ระหว่าง 0-100"
  }
end
```

### format

```ruby
class User < ApplicationRecord
  validates :email, format: { with: /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i,
                               message: "รูปแบบอีเมลไม่ถูกต้อง" }

  validates :username, format: { with: /\A[a-zA-Z0-9_]+\z/,
                                  message: "ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _" }

  validates :phone, format: { with: /\A0[0-9]{8,9}\z/,
                               message: "เบอร์โทรไม่ถูกต้อง" }

  validates :url, format: { with: URI::DEFAULT_PARSER.make_regexp(%w[http https]),
                             allow_blank: true,
                             message: "URL ไม่ถูกต้อง" }
end
```

### inclusion / exclusion

```ruby
class Article < ApplicationRecord
  validates :status, inclusion: {
    in: %w[draft pending published archived],
    message: "สถานะ %{value} ไม่ถูกต้อง"
  }

  validates :category, exclusion: {
    in: %w[spam inappropriate],
    message: "ไม่สามารถใช้หมวดหมู่นี้"
  }
end

class Size < ApplicationRecord
  validates :value, inclusion: { in: ["small", "medium", "large", "extra_large"] }
end
```

### uniqueness

```ruby
class User < ApplicationRecord
  validates :email, uniqueness: true
  validates :email, uniqueness: { case_sensitive: false }  # ไม่สนใจ case

  # scope: unique ภายใน scope
  validates :username, uniqueness: {
    scope: :organization_id,
    message: "ชื่อ username นี้ถูกใช้ในองค์กรแล้ว"
  }

  validates :title, uniqueness: {
    scope: [:user_id, :year],
    message: "คุณมีบทความชื่อนี้ในปีนี้แล้ว"
  }
end

# หมายเหตุ: uniqueness validation ใน Rails ไม่ thread-safe 100%
# ควรเพิ่ม unique index ใน database ด้วย:
# add_index :users, :email, unique: true
# add_index :users, [:username, :organization_id], unique: true
```

### acceptance

```ruby
class Registration < ApplicationRecord
  validates :terms_of_service, acceptance: true
  validates :privacy_policy, acceptance: true
  validates :age_confirmation, acceptance: { accept: ["yes", "true"] }
end
```

### confirmation

```ruby
class User < ApplicationRecord
  validates :email, confirmation: true
  validates :password, confirmation: {
    case_sensitive: false,
    message: "รหัสผ่านไม่ตรงกัน"
  }
end

# ใน view:
# <%= form.text_field :password_confirmation %>
```

---

## ขั้นตอนที่ 862: Conditional Validations

```ruby
class Article < ApplicationRecord
  # if: symbol
  validates :published_at, presence: true, if: :published?
  validates :review_note, presence: true, if: :pending_review?

  # if: Proc
  validates :discount_reason, presence: true,
    if: -> { discount.present? && discount > 0 }

  # unless
  validates :draft_note, presence: true, unless: :published?

  # Multiple conditions
  validates :approval_note,
    presence: true,
    if: :requires_approval?,
    unless: :admin?

  # on: ระบุว่า validate เมื่อไหร่
  validates :password, presence: true, on: :create
  validates :reason, presence: true, on: :update
  validates :terms, acceptance: true, on: :create

  # on: custom context
  validates :credit_card, presence: true, on: :payment

  def published?
    status == "published"
  end

  def pending_review?
    status == "pending"
  end

  def requires_approval?
    category&.requires_approval?
  end
end

# ใช้งาน custom context:
article.valid?(:payment)
article.save(context: :payment)
```

---

## ขั้นตอนที่ 863: Custom Validations

### validate Method

```ruby
class Article < ApplicationRecord
  validate :title_must_be_unique_within_30_days
  validate :body_cannot_contain_banned_words
  validate :publication_date_cannot_be_in_past, on: :create

  private

  def title_must_be_unique_within_30_days
    if Article.where(title: title)
              .where("created_at > ?", 30.days.ago)
              .where.not(id: id)
              .exists?
      errors.add(:title, "มีบทความชื่อนี้ในช่วง 30 วันที่ผ่านมาแล้ว")
    end
  end

  def body_cannot_contain_banned_words
    banned_words = %w[spam inappropriate offensive]
    banned_words.each do |word|
      if body&.downcase&.include?(word)
        errors.add(:body, "มีคำที่ไม่เหมาะสม: #{word}")
      end
    end
  end

  def publication_date_cannot_be_in_past
    if published_at.present? && published_at < Time.current
      errors.add(:published_at, "ไม่สามารถตั้งเวลาย้อนหลังได้")
    end
  end
end
```

### Custom Validator Class

```ruby
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i

  def validate_each(record, attribute, value)
    unless value =~ EMAIL_REGEX
      record.errors.add(attribute, options[:message] || "ไม่ใช่รูปแบบอีเมลที่ถูกต้อง")
    end
  end
end

# ใช้งาน:
class User < ApplicationRecord
  validates :email, email: true
  validates :work_email, email: { message: "รูปแบบอีเมลงานไม่ถูกต้อง" }
end

# app/validators/url_validator.rb
class UrlValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank? && options[:allow_blank]

    begin
      uri = URI.parse(value)
      valid = uri.is_a?(URI::HTTP) || uri.is_a?(URI::HTTPS)
      valid &&= uri.host.present?
    rescue URI::InvalidURIError
      valid = false
    end

    record.errors.add(attribute, "ไม่ใช่ URL ที่ถูกต้อง") unless valid
  end
end

# app/validators/thai_id_validator.rb
class ThaiIdValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank?

    unless valid_thai_id?(value)
      record.errors.add(attribute, "เลขบัตรประชาชนไม่ถูกต้อง")
    end
  end

  private

  def valid_thai_id?(id)
    return false unless id =~ /\A\d{13}\z/

    # Luhn algorithm สำหรับเลขบัตรประชาชนไทย
    sum = (0..11).sum { |i| id[i].to_i * (13 - i) }
    check_digit = (11 - (sum % 11)) % 10
    check_digit == id[12].to_i
  end
end

# ใช้งาน:
class Person < ApplicationRecord
  validates :national_id, thai_id: true
end
```

---

## ขั้นตอนที่ 864: Validation Errors

### จัดการ Errors

```ruby
article = Article.new(title: "")
article.valid?  # => false

# ดู errors:
article.errors                     # ActiveModel::Errors object
article.errors[:title]             # => ["can't be blank"]
article.errors[:body]              # => ["can't be blank"]
article.errors.full_messages        # => ["Title can't be blank", "Body can't be blank"]
article.errors.full_messages.join(", ")

article.errors.messages            # => {title: ["can't be blank"], body: ["can't be blank"]}
article.errors.details             # => {title: [{error: :blank}]}
article.errors.count               # => 2
article.errors.any?                # => true
article.errors.empty?              # => false

# เพิ่ม error:
article.errors.add(:title, "ต้องมีความยาวอย่างน้อย 5 ตัวอักษร")
article.errors.add(:base, "เกิดข้อผิดพลาดทั่วไป")  # ไม่ระบุ field

# ลบ error:
article.errors.delete(:title)

# ใน Controller:
def create
  @article = Article.new(article_params)
  if @article.save
    redirect_to @article, notice: "สร้างบทความสำเร็จ"
  else
    render :new, status: :unprocessable_entity
  end
end

# ใน View:
<% if @article.errors.any? %>
  <div class="alert alert-danger">
    <h4><%= pluralize(@article.errors.count, "error") %> ต้องแก้ไข:</h4>
    <ul>
      <% @article.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  </div>
<% end %>
```

---

## ขั้นตอนที่ 865: save, save!, valid?, invalid?

```ruby
article = Article.new(title: "")

# valid? และ invalid?
article.valid?   # => false (runs validations)
article.invalid? # => true

# save
article.save     # => false (ถ้า validation fail)
article.save!    # => raises ActiveRecord::RecordInvalid

# create
Article.create(title: "")   # => Article object กับ errors
Article.create!(title: "")  # => raises ActiveRecord::RecordInvalid

# update
article = Article.find(1)
article.update(title: "")   # => false
article.update!(title: "")  # => raises ActiveRecord::RecordInvalid

# skip validations (ระวัง! ใช้เมื่อจำเป็น)
article.save(validate: false)
article.update_column(:title, "")   # ไม่ run validations, ไม่ update timestamps
article.update_columns(title: "")   # ไม่ run validations, ไม่ update timestamps
article.update_attribute(:title, "") # ข้าม validations บาง ส่วน

# rescue error:
begin
  article.save!
rescue ActiveRecord::RecordInvalid => e
  Rails.logger.error "Validation failed: #{e.record.errors.full_messages}"
  raise
end
```

---

## ขั้นตอนที่ 866: accepts_nested_attributes_for

```ruby
class Article < ApplicationRecord
  has_one :seo_meta, dependent: :destroy
  has_many :tags, through: :article_tags

  accepts_nested_attributes_for :seo_meta,
    allow_destroy: true,
    reject_if: :all_blank

  accepts_nested_attributes_for :article_tags,
    allow_destroy: true,
    reject_if: proc { |attrs| attrs["tag_id"].blank? }

  # หรือ custom reject_if:
  accepts_nested_attributes_for :comments,
    reject_if: ->(attrs) { attrs["body"].blank? },
    allow_destroy: true,
    limit: 5  # จำกัดจำนวน records
end

class SeoMeta < ApplicationRecord
  belongs_to :article
  validates :meta_title, length: { maximum: 60 }
  validates :meta_description, length: { maximum: 160 }
end

# Controller:
def article_params
  params.require(:article).permit(
    :title, :body, :status,
    seo_meta_attributes: [:id, :meta_title, :meta_description, :_destroy],
    article_tags_attributes: [:id, :tag_id, :_destroy]
  )
end

# View:
<%= form_with model: @article do |f| %>
  <%= f.text_field :title %>

  <%= f.fields_for :seo_meta do |seo_form| %>
    <%= seo_form.text_field :meta_title %>
    <%= seo_form.text_area :meta_description %>
    <%= seo_form.check_box :_destroy %>
    <%= seo_form.label :_destroy, "ลบ SEO Meta" %>
  <% end %>

  <%= f.submit %>
<% end %>
```

---

## ขั้นตอนที่ 867: Client-side Validation Hints

```ruby
# Rails form helpers สร้าง HTML validation attributes อัตโนมัติ
# โดย reflect บน model validations

# ตัวอย่าง:
class User < ApplicationRecord
  validates :username, presence: true, length: { minimum: 3, maximum: 50 }
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
end

# Form:
<%= form_with model: @user do |f| %>
  <%= f.text_field :username %>
  <%# สร้าง: <input type="text" name="user[username]" required minlength="3" maxlength="50"> %>

  <%= f.email_field :email %>
  <%# สร้าง: <input type="email" name="user[email]" required> %>
<% end %>

# Custom HTML5 validation attributes:
<%= f.text_field :phone, pattern: "0[0-9]{8,9}", title: "กรอกเบอร์โทร 9-10 หลัก" %>

# Disable HTML5 validation (ใช้เฉพาะ server-side):
<%= form_with model: @user, novalidate: true do |f| %>
```

---

## ขั้นตอนที่ 868: Complete Model Example

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # Constants
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  VALID_PHONE_REGEX = /\A0[0-9]{8,9}\z/
  ROLES = %w[user moderator admin].freeze

  # Validations
  validates :name, presence: true, length: { in: 2..100 }
  validates :username, presence: true,
    length: { in: 3..50 },
    format: { with: /\A[a-zA-Z0-9_]+\z/, message: "ใช้ได้เฉพาะ a-z, 0-9, _" },
    uniqueness: { case_sensitive: false }

  validates :email, presence: true,
    format: { with: VALID_EMAIL_REGEX, message: "รูปแบบอีเมลไม่ถูกต้อง" },
    uniqueness: { case_sensitive: false }

  validates :phone, format: { with: VALID_PHONE_REGEX, allow_blank: true }

  validates :role, inclusion: { in: ROLES }

  validates :password, presence: true,
    length: { minimum: 8 },
    format: {
      with: /(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      message: "ต้องมีตัวพิมพ์ใหญ่ ตัวพิมพ์เล็ก และตัวเลข"
    },
    on: :create
  validates :password_confirmation, presence: true, on: :create

  validates :bio, length: { maximum: 500 }, allow_blank: true
  validates :website, format: { with: URI::DEFAULT_PARSER.make_regexp(%w[http https]),
                                  allow_blank: true }

  validates :age, numericality: { only_integer: true, greater_than_or_equal_to: 13,
                                   less_than: 120 }, allow_nil: true

  validates :terms_accepted, acceptance: true, on: :create

  validate :email_not_in_blocked_list

  # Callbacks
  before_validation :normalize_email, :normalize_username

  private

  def normalize_email
    self.email = email.strip.downcase if email.present?
  end

  def normalize_username
    self.username = username.strip.downcase if username.present?
  end

  def email_not_in_blocked_list
    blocked_domains = %w[temp-mail.com throwaway.com fakeinbox.com]
    if email.present?
      domain = email.split("@").last
      if blocked_domains.include?(domain)
        errors.add(:email, "ไม่สามารถใช้อีเมลชั่วคราวได้")
      end
    end
  end
end
```

---

## แบบฝึกหัด Part 39 (ขั้นตอนที่ 861-880)

**ข้อ 1:** เพิ่ม validation สำหรับ Article model

```ruby
# คำตอบ:
class Article < ApplicationRecord
  validates :title, presence: true, length: { minimum: 5, maximum: 200 }
  validates :body, presence: true, length: { minimum: 50 }
  validates :status, inclusion: { in: %w[draft published archived] }
  validates :published_at, presence: true, if: :published?

  private
  def published?
    status == "published"
  end
end
```

**ข้อ 2:** สร้าง custom EmailValidator

```ruby
# คำตอบ:
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless value =~ /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
      record.errors.add(attribute, "ไม่ใช่รูปแบบอีเมลที่ถูกต้อง")
    end
  end
end

class User < ApplicationRecord
  validates :email, email: true
end
```

**ข้อ 3:** ใช้ conditional validation กับ on: context

```ruby
# คำตอบ:
class Order < ApplicationRecord
  validates :shipping_address, presence: true, on: :checkout
  validates :payment_token, presence: true, on: :payment
  validates :review_text, length: { minimum: 20 }, on: :review

  # ใช้งาน:
  # order.valid?(:checkout)
  # order.save(context: :payment)
end
```

**ข้อ 4:** จัดการ validation errors ใน view

```ruby
# คำตอบ:
# app/views/shared/_form_errors.html.erb
<% if object.errors.any? %>
  <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
    <h4 class="font-bold">พบ <%= pluralize(object.errors.count, "ข้อผิดพลาด") %>:</h4>
    <ul class="list-disc ml-4 mt-2">
      <% object.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  </div>
<% end %>

# ใช้งาน:
# <%= render "shared/form_errors", object: @article %>
```

**ข้อ 5:** ใช้ accepts_nested_attributes_for

```ruby
# คำตอบ:
class Survey < ApplicationRecord
  has_many :questions, dependent: :destroy
  accepts_nested_attributes_for :questions,
    allow_destroy: true,
    reject_if: ->(attrs) { attrs["text"].blank? }
  
  validates :title, presence: true
  validates :questions, length: { minimum: 1, message: "ต้องมีอย่างน้อย 1 คำถาม" }
end

class Question < ApplicationRecord
  belongs_to :survey
  validates :text, presence: true
  validates :question_type, inclusion: { in: %w[text choice rating] }
end

# Controller:
def survey_params
  params.require(:survey).permit(
    :title,
    questions_attributes: [:id, :text, :question_type, :required, :_destroy]
  )
end
```

---

## สรุป Part 39

ในบทนี้เราได้เรียนรู้:

1. **Validation Helpers** - presence, length, numericality, format, inclusion, exclusion, uniqueness, acceptance, confirmation
2. **Conditional Validations** - if, unless, on:
3. **Custom Validations** - validate method, custom validator classes
4. **Validation Errors** - errors object, full_messages, add, delete
5. **save vs save!** - และวิธีจัดการ
6. **accepts_nested_attributes_for** - validate nested models
7. **Client-side hints** - HTML5 attributes

Validations เป็นส่วนสำคัญในการรักษาความถูกต้องของข้อมูล และควรใช้ร่วมกับ database constraints เสมอ

---

*ต่อไป: Part 40 - Forms (ขั้นตอนที่ 881-905)*
