# ตอนที่ 39: Validations (Steps 861-880)

## บทนำ

Validations ใน Rails ช่วยให้แน่ใจว่าข้อมูลที่บันทึกลง database มีความถูกต้องและสมบูรณ์ Rails มี built-in validators มากมายและยังสามารถสร้าง custom validators ได้

---

## Step 861: validates :presence

```ruby
class User < ApplicationRecord
  # ต้องมีค่า (ไม่เป็น nil หรือ empty string)
  validates :name, presence: true
  validates :email, presence: true
  
  # หลาย attributes พร้อมกัน
  validates :name, :email, :password, presence: true
end

# ใช้งาน
user = User.new
user.valid?    # => false
user.errors[:name]   # => ["can't be blank"]
user.errors.full_messages  # => ["Name can't be blank", "Email can't be blank"]

user = User.new(name: "สมชาย", email: "somchai@example.com")
user.valid?    # => true

# Custom message
validates :name, presence: { message: "กรุณากรอกชื่อ" }
validates :email, presence: { message: "กรุณากรอกอีเมล" }
```

---

## Step 862: validates :length

```ruby
class User < ApplicationRecord
  # ความยาว minimum
  validates :name, length: { minimum: 2 }
  
  # ความยาว maximum
  validates :bio, length: { maximum: 500 }
  
  # ความยาว range
  validates :username, length: { in: 3..20 }
  validates :password, length: { in: 8..128 }
  
  # ความยาว exact
  validates :postal_code, length: { is: 5 }
  validates :pin, length: { is: 6 }
  
  # Custom messages
  validates :name, length: {
    minimum: 2,
    maximum: 100,
    too_short: "ชื่อสั้นเกินไป (ขั้นต่ำ %{count} ตัวอักษร)",
    too_long: "ชื่อยาวเกินไป (สูงสุด %{count} ตัวอักษร)"
  }
  
  # นับด้วย token แทน char
  validates :content, length: { maximum: 1000, tokenizer: ->(str) { str.split } }
  # นับ words ไม่ใช่ characters
end
```

---

## Step 863: validates :numericality

```ruby
class Product < ApplicationRecord
  # ต้องเป็นตัวเลข
  validates :price, numericality: true
  
  # ต้องเป็น integer
  validates :quantity, numericality: { only_integer: true }
  
  # ค่า minimum
  validates :price, numericality: { greater_than: 0 }
  validates :price, numericality: { greater_than_or_equal_to: 0 }
  
  # ค่า maximum
  validates :discount, numericality: { less_than: 100 }
  validates :discount, numericality: { less_than_or_equal_to: 100 }
  
  # ค่าเฉพาะ
  validates :rating, numericality: { equal_to: 5 }
  validates :score, numericality: { other_than: 0 }
  
  # รวมหลาย conditions
  validates :price, numericality: {
    greater_than_or_equal_to: 0,
    less_than: 1_000_000,
    message: "ราคาต้องอยู่ระหว่าง 0 ถึง 999,999"
  }
  
  # อนุญาต odd/even numbers
  validates :even_number, numericality: { even: true }
  validates :odd_number, numericality: { odd: true }
end
```

---

## Step 864: validates :format

```ruby
class User < ApplicationRecord
  # ตรวจสอบ format ด้วย regex
  validates :email, format: { with: /\A[^@\s]+@[^@\s]+\.[^@\s]+\z/ }
  
  # Custom message
  validates :email, format: {
    with: /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i,
    message: "รูปแบบอีเมลไม่ถูกต้อง"
  }
  
  # ไม่ match กับ regex
  validates :username, format: { without: /\s/, message: "ชื่อผู้ใช้ไม่สามารถมีช่องว่าง" }
  
  # Phone number (Thai)
  validates :phone, format: {
    with: /\A0[0-9]{8,9}\z/,
    message: "เบอร์โทรศัพท์ไม่ถูกต้อง",
    allow_blank: true
  }
  
  # URL
  validates :website, format: {
    with: /\Ahttps?:\/\/[^\s]+\z/,
    message: "URL ไม่ถูกต้อง",
    allow_blank: true
  }
  
  # Password strength
  validates :password, format: {
    with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
    message: "รหัสผ่านต้องมีตัวพิมพ์เล็ก พิมพ์ใหญ่ และตัวเลข"
  }
  
  # Username (alphanumeric + underscore)
  validates :username, format: {
    with: /\A[a-zA-Z0-9_]+\z/,
    message: "ชื่อผู้ใช้ใช้ได้เฉพาะตัวอักษรและตัวเลขและ _"
  }
end
```

---

## Step 865: validates :inclusion / :exclusion

### inclusion

```ruby
class Post < ApplicationRecord
  # ต้องอยู่ใน list
  validates :status, inclusion: { in: %w[draft published archived] }
  validates :status, inclusion: { in: ["draft", "published", "archived"],
                                    message: "สถานะ '%{value}' ไม่ถูกต้อง" }
  
  # ด้วย range
  validates :rating, inclusion: { in: 1..5 }
  
  # ด้วย lambda
  validates :role, inclusion: { in: -> (user) { User::ROLES } }
  
  # Case insensitive
  validates :country_code, inclusion: {
    in: -> (rec) { ISO3166::Country.codes.map(&:downcase) }
  }
end

# Enum จัดการ inclusion อัตโนมัติ
class Post < ApplicationRecord
  enum status: { draft: 0, published: 1, archived: 2 }
  # validates status เป็น enum values อัตโนมัติ
end
```

### exclusion

```ruby
class User < ApplicationRecord
  # ต้องไม่อยู่ใน list
  validates :username, exclusion: {
    in: %w[admin superuser root system moderator],
    message: "ชื่อนี้ไม่อนุญาตให้ใช้"
  }
  
  validates :role, exclusion: {
    in: ["super_admin"],
    message: "ไม่สามารถกำหนด role นี้ได้"
  }
end
```

---

## Step 866: validates :uniqueness

```ruby
class User < ApplicationRecord
  # ต้องไม่ซ้ำกับ records อื่น
  validates :email, uniqueness: true
  
  # Case insensitive
  validates :username, uniqueness: { case_sensitive: false }
  
  # Scoped uniqueness
  validates :name, uniqueness: { scope: :company_id }
  # ชื่อ unique ภายใน company เดียวกัน
  
  # หลาย scopes
  validates :email, uniqueness: {
    scope: [:organization_id, :role],
    message: "อีเมลนี้มีใน organization แล้ว"
  }
  
  # Custom message
  validates :email, uniqueness: { message: "อีเมลนี้ถูกใช้งานแล้ว" }
end

# ข้อควรระวัง: uniqueness validation ไม่ปลอดภัย 100%
# เพิ่ม database index unique เพื่อป้องกัน race condition
class AddUniqueIndexToUsers < ActiveRecord::Migration[7.0]
  def change
    add_index :users, :email, unique: true
    add_index :users, :username, unique: true
  end
end
```

---

## Step 867: validates :confirmation

```ruby
class User < ApplicationRecord
  validates :password, confirmation: true
  # ต้องการ attribute :password_confirmation ใน form
  
  validates :email, confirmation: {
    case_sensitive: false,
    message: "อีเมลยืนยันไม่ตรงกัน"
  }
end

# ใน form ต้องมี field confirmation
# <%= form.password_field :password %>
# <%= form.password_field :password_confirmation %>
```

---

## Step 868: validates :acceptance

```ruby
class Registration < ApplicationRecord
  # ต้องยอมรับ (เหมาะสำหรับ checkbox)
  validates :terms_of_service, acceptance: true
  validates :privacy_policy, acceptance: true
  
  # กำหนดค่าที่ accept ได้
  validates :terms_of_service, acceptance: { accept: ["1", "yes", true] }
  
  # Custom message
  validates :terms_of_service, acceptance: {
    message: "กรุณายอมรับข้อตกลงการใช้งาน"
  }
end
```

---

## Step 869: Custom Validation Methods

```ruby
class Post < ApplicationRecord
  validate :publish_date_cannot_be_in_the_past
  validate :title_must_be_unique_for_user
  validate :content_word_count
  
  private
  
  def publish_date_cannot_be_in_the_past
    if publish_date.present? && publish_date < Date.today
      errors.add(:publish_date, "ไม่สามารถย้อนหลังได้")
    end
  end
  
  def title_must_be_unique_for_user
    if user && Post.where(user: user, title: title).where.not(id: id).exists?
      errors.add(:title, "คุณมีบทความชื่อนี้อยู่แล้ว")
    end
  end
  
  def content_word_count
    word_count = content.to_s.split.size
    if word_count < 100
      errors.add(:content, "เนื้อหาต้องมีอย่างน้อย 100 คำ (ปัจจุบัน #{word_count} คำ)")
    end
    if word_count > 10000
      errors.add(:content, "เนื้อหาต้องไม่เกิน 10,000 คำ")
    end
  end
end
```

### Custom Validation ที่ซับซ้อน

```ruby
class Order < ApplicationRecord
  validate :products_in_stock
  validate :total_amount_matches
  validate :shipping_address_valid
  
  private
  
  def products_in_stock
    order_items.each do |item|
      unless item.product.sufficient_stock?(item.quantity)
        errors.add(:base, "#{item.product.name} สต็อกไม่พอ (มี #{item.product.stock} ชิ้น)")
      end
    end
  end
  
  def total_amount_matches
    calculated = order_items.sum { |item| item.unit_price * item.quantity }
    unless (total - calculated).abs < 0.01  # floating point tolerance
      errors.add(:total, "ยอดรวมไม่ถูกต้อง")
    end
  end
  
  def shipping_address_valid
    address = shipping_address
    return if address.blank?
    
    unless address[:postal_code].to_s.match?(/\A\d{5}\z/)
      errors.add(:base, "รหัสไปรษณีย์ไม่ถูกต้อง")
    end
    
    unless address[:province].present?
      errors.add(:base, "กรุณาระบุจังหวัด")
    end
  end
end
```

---

## Step 870: Custom Validator Classes

```ruby
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless value =~ /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
      record.errors.add(attribute, options[:message] || "รูปแบบอีเมลไม่ถูกต้อง")
    end
  end
end

# ใช้งาน
class User < ApplicationRecord
  validates :email, email: true
  validates :backup_email, email: { message: "อีเมลสำรองไม่ถูกต้อง" }
end
```

```ruby
# app/validators/phone_validator.rb
class PhoneValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank? && options[:allow_blank]
    
    cleaned = value.to_s.gsub(/[\s\-]/, "")
    unless cleaned.match?(/\A0[0-9]{8,9}\z/)
      record.errors.add(attribute, "เบอร์โทรศัพท์ไม่ถูกต้อง")
    end
  end
end

# app/validators/url_validator.rb
class UrlValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank?
    
    begin
      uri = URI.parse(value)
      unless uri.is_a?(URI::HTTP) || uri.is_a?(URI::HTTPS)
        record.errors.add(attribute, "ต้องเป็น URL ที่ขึ้นต้นด้วย http หรือ https")
      end
    rescue URI::InvalidURIError
      record.errors.add(attribute, "URL ไม่ถูกต้อง")
    end
  end
end

# app/validators/strong_password_validator.rb
class StrongPasswordValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank?
    
    errors = []
    errors << "ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว" unless value.match?(/[a-z]/)
    errors << "ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว" unless value.match?(/[A-Z]/)
    errors << "ต้องมีตัวเลขอย่างน้อย 1 ตัว" unless value.match?(/\d/)
    errors << "ต้องมีอย่างน้อย 8 ตัวอักษร" if value.length < 8
    
    errors.each do |error|
      record.errors.add(attribute, error)
    end
  end
end
```

### ActiveModel::Validator class

```ruby
# สำหรับ validate หลาย attributes พร้อมกัน
class PersonValidator < ActiveModel::Validator
  def validate(record)
    if record.first_name.blank? && record.last_name.blank?
      record.errors.add(:base, "ต้องมีชื่อหรือนามสกุลอย่างน้อยหนึ่งอย่าง")
    end
  end
end

class Person < ApplicationRecord
  validates_with PersonValidator
end
```

---

## Step 871: Conditional Validations

```ruby
class Post < ApplicationRecord
  # if: method name
  validates :published_at, presence: true, if: :published?
  
  # unless: method name
  validates :draft_title, presence: true, unless: :published?
  
  # if: lambda/proc
  validates :price, numericality: { greater_than: 0 },
            if: -> { product_type == "paid" }
  
  validates :reason, presence: true,
            if: -> (post) { post.archived? && post.age_days > 365 }
  
  # unless: lambda
  validates :bio, presence: true,
            unless: -> { guest_user? }
  
  # on: context
  validates :terms_accepted, acceptance: true, on: :create
  validates :current_password, presence: true, on: :update_password
  
  # Combined conditions
  validates :price, numericality: { greater_than: 0 },
            if: :paid?,
            unless: :free_trial?
  
  private
  
  def published?
    status == "published"
  end
  
  def paid?
    pricing_model == "paid"
  end
  
  def free_trial?
    trial_period?
  end
end
```

### Validation Context (on:)

```ruby
class User < ApplicationRecord
  validates :name, presence: true
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  
  # เฉพาะ create
  validates :password, presence: true, on: :create
  validates :terms_accepted, acceptance: true, on: :create
  
  # เฉพาะ update
  validates :current_password, presence: true, on: :update_password
  
  # Custom context
  validates :credit_card, presence: true, on: :payment
end

# ใช้ custom context
user.valid?(:payment)        # รัน validations สำหรับ :payment context
user.save(context: :payment) # บันทึกพร้อม validation context
```

---

## Step 872: valid? / invalid? / errors

```ruby
user = User.new

# ตรวจสอบ validity
user.valid?    # => false (รัน validations)
user.invalid?  # => true

# หลัง valid? errors จะถูก populate
user.errors                   # => ActiveModel::Errors object
user.errors.any?              # => true
user.errors.empty?            # => false
user.errors.count             # => 3

# ดึง errors
user.errors[:name]            # => ["can't be blank"]
user.errors[:email]           # => ["can't be blank", "is invalid"]
user.errors.full_messages     # => ["Name can't be blank", "Email is invalid"]
user.errors.messages          # => {name: ["can't be blank"], email: ["..."]}

# เพิ่ม error
user.errors.add(:name, "is too short")
user.errors.add(:base, "ข้อผิดพลาดทั่วไป")

# ลบ error
user.errors.delete(:name)

# ดู errors ทุก attributes
user.errors.each do |error|
  puts "#{error.attribute}: #{error.message}"
end

# ตรวจสอบ error เฉพาะ attribute
user.errors.include?(:name)  # => true

# Where (Rails 6.1+)
user.errors.where(:email).each { |e| puts e.message }
user.errors.where(:email, :invalid)  # เฉพาะ :invalid error
```

---

## Step 873: Error Messages Customization

### I18n Translations

```yaml
# config/locales/th.yml
th:
  activerecord:
    errors:
      messages:
        blank: "ไม่สามารถเว้นว่างได้"
        taken: "ถูกใช้งานแล้ว"
        invalid: "ไม่ถูกต้อง"
        too_short:
          one: "สั้นเกินไป (ขั้นต่ำ 1 ตัวอักษร)"
          other: "สั้นเกินไป (ขั้นต่ำ %{count} ตัวอักษร)"
        too_long:
          one: "ยาวเกินไป (สูงสุด 1 ตัวอักษร)"
          other: "ยาวเกินไป (สูงสุด %{count} ตัวอักษร)"
        not_a_number: "ต้องเป็นตัวเลข"
        not_an_integer: "ต้องเป็นจำนวนเต็ม"
        greater_than: "ต้องมากกว่า %{count}"
        less_than: "ต้องน้อยกว่า %{count}"
        greater_than_or_equal_to: "ต้องมากกว่าหรือเท่ากับ %{count}"
        less_than_or_equal_to: "ต้องน้อยกว่าหรือเท่ากับ %{count}"
        inclusion: "ไม่อยู่ในรายการที่อนุญาต"
        exclusion: "ไม่อนุญาตให้ใช้ค่านี้"
        confirmation: "ยืนยันไม่ตรงกัน"
        accepted: "ต้องยอมรับ"
        
      models:
        user:
          attributes:
            name:
              blank: "กรุณากรอกชื่อ"
              too_short: "ชื่อสั้นเกินไป"
            email:
              blank: "กรุณากรอกอีเมล"
              taken: "อีเมลนี้ถูกใช้งานแล้ว"
              invalid: "รูปแบบอีเมลไม่ถูกต้อง"
            password:
              blank: "กรุณากรอกรหัสผ่าน"
              too_short: "รหัสผ่านสั้นเกินไป"
```

### ใช้ i18n ใน Validation

```ruby
class User < ApplicationRecord
  validates :name, presence: { message: :blank }
  # ใช้ key จาก i18n
  
  validates :email, uniqueness: { message: :taken }
  
  # หรือ interpolation
  validates :name, length: {
    minimum: 2,
    too_short: :too_short  # ใช้ key จาก i18n
  }
end
```

### แสดง Error ใน Views

```erb
<!-- app/views/shared/_errors.html.erb -->
<% if @object.errors.any? %>
  <div class="alert alert-danger">
    <h4>พบ <%= pluralize(@object.errors.count, "error") %></h4>
    <ul>
      <% @object.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<!-- ใน form field -->
<div class="form-group <%= 'has-error' if @user.errors[:email].any? %>">
  <%= form.label :email %>
  <%= form.email_field :email, class: "form-control" %>
  <% if @user.errors[:email].any? %>
    <p class="help-block text-danger">
      <%= @user.errors[:email].join(", ") %>
    </p>
  <% end %>
</div>
```

---

## ตัวอย่าง Model ที่มี Validations สมบูรณ์

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # Constants
  ROLES = %w[member moderator admin].freeze
  
  # Attributes
  attr_accessor :password, :password_confirmation, :current_password
  
  # Associations
  has_many :posts, dependent: :destroy
  has_one :profile, dependent: :destroy
  
  # Validations
  
  # Name
  validates :name,
    presence: { message: "กรุณากรอกชื่อ" },
    length: { minimum: 2, maximum: 100,
              too_short: "ชื่อสั้นเกินไป",
              too_long: "ชื่อยาวเกินไป" },
    format: { with: /\A[\p{L}\p{M}\s]+\z/u,
              message: "ชื่อต้องเป็นตัวอักษรเท่านั้น" }
  
  # Email
  validates :email,
    presence: { message: "กรุณากรอกอีเมล" },
    format: { with: URI::MailTo::EMAIL_REGEXP,
              message: "รูปแบบอีเมลไม่ถูกต้อง" },
    uniqueness: { case_sensitive: false,
                  message: "อีเมลนี้ถูกใช้งานแล้ว" }
  
  # Password
  validates :password,
    presence: { message: "กรุณากรอกรหัสผ่าน", on: :create },
    length: { minimum: 8, too_short: "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร",
              allow_nil: true },
    confirmation: { message: "รหัสผ่านยืนยันไม่ตรงกัน" }
  
  # Phone
  validates :phone,
    format: { with: /\A0[0-9]{8,9}\z/,
              message: "เบอร์โทรศัพท์ไม่ถูกต้อง" },
    allow_blank: true
  
  # Role
  validates :role,
    inclusion: { in: ROLES, message: "role '%{value}' ไม่ถูกต้อง" }
  
  # Age
  validates :age,
    numericality: { greater_than_or_equal_to: 0,
                    less_than: 150,
                    only_integer: true,
                    message: "อายุไม่ถูกต้อง" },
    allow_nil: true
  
  # Terms
  validates :terms_accepted, acceptance: true, on: :create
  
  # Custom validations
  validate :birthdate_cannot_be_future
  validate :phone_format_check, if: :phone_changed?
  
  private
  
  def birthdate_cannot_be_future
    if birthdate.present? && birthdate > Date.today
      errors.add(:birthdate, "วันเกิดไม่สามารถเป็นอนาคตได้")
    end
  end
  
  def phone_format_check
    return if phone.blank?
    cleaned = phone.gsub(/[\s\-]/, "")
    unless cleaned.match?(/\A0[0-9]{8,9}\z/)
      errors.add(:phone, "รูปแบบเบอร์โทรศัพท์ไม่ถูกต้อง")
    end
  end
end
```

---

## แบบฝึกหัด (Steps 874-880)

### แบบฝึกหัดที่ 1
เพิ่ม validation presence สำหรับ title และ content ใน Post model

**เฉลย:**
```ruby
class Post < ApplicationRecord
  validates :title, presence: true
  validates :content, presence: true
  # หรือ
  validates :title, :content, presence: true
end
```

### แบบฝึกหัดที่ 2
Validate ว่า username มีความยาว 3-20 ตัวอักษร

**เฉลย:**
```ruby
validates :username, length: { in: 3..20,
  too_short: "Username ต้องมีอย่างน้อย %{count} ตัวอักษร",
  too_long: "Username ต้องไม่เกิน %{count} ตัวอักษร"
}
```

### แบบฝึกหัดที่ 3
Validate ว่า price เป็นตัวเลขที่มากกว่า 0

**เฉลย:**
```ruby
validates :price, numericality: {
  greater_than: 0,
  message: "ราคาต้องมากกว่า 0"
}
```

### แบบฝึกหัดที่ 4
Validate format ของ email ด้วย regex

**เฉลย:**
```ruby
validates :email, format: {
  with: /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i,
  message: "รูปแบบอีเมลไม่ถูกต้อง"
}
```

### แบบฝึกหัดที่ 5
Validate ว่า status อยู่ใน allowed values

**เฉลย:**
```ruby
validates :status, inclusion: {
  in: %w[draft published archived],
  message: "สถานะ '%{value}' ไม่ถูกต้อง"
}
```

### แบบฝึกหัดที่ 6
Validate uniqueness ของ email (case insensitive)

**เฉลย:**
```ruby
validates :email, uniqueness: { case_sensitive: false, message: "อีเมลนี้ถูกใช้งานแล้ว" }
```

### แบบฝึกหัดที่ 7
Validate ว่า password และ password_confirmation ตรงกัน

**เฉลย:**
```ruby
validates :password, confirmation: { message: "รหัสผ่านยืนยันไม่ตรงกัน" }
# ในฟอร์มต้องมี password_confirmation field
```

### แบบฝึกหัดที่ 8
สร้าง custom validation method ที่ตรวจสอบว่า end_date มาหลัง start_date

**เฉลย:**
```ruby
validate :end_date_after_start_date

private

def end_date_after_start_date
  return unless start_date && end_date
  
  if end_date <= start_date
    errors.add(:end_date, "ต้องมาหลังวันเริ่มต้น")
  end
end
```

### แบบฝึกหัดที่ 9
สร้าง EmailValidator class ที่ reuse ได้

**เฉลย:**
```ruby
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless value.to_s.match?(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
      record.errors.add(attribute, options[:message] || "ไม่ใช่อีเมลที่ถูกต้อง")
    end
  end
end

# ใช้งาน
class User < ApplicationRecord
  validates :email, email: true
end
```

### แบบฝึกหัดที่ 10
ใช้ conditional validation ที่รันเฉพาะตอน create

**เฉลย:**
```ruby
validates :password, presence: true, on: :create
validates :terms_accepted, acceptance: true, on: :create
```

### แบบฝึกหัดที่ 11
แสดง validation errors ใน view

**เฉลย:**
```erb
<% if @user.errors.any? %>
  <div class="alert alert-danger">
    <ul>
      <% @user.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  </div>
<% end %>
```

### แบบฝึกหัดที่ 12
ใช้ allow_blank option กับ validation

**เฉลย:**
```ruby
validates :website, format: {
  with: /\Ahttps?:\/\//,
  message: "ต้องขึ้นต้นด้วย http:// หรือ https://"
}, allow_blank: true

validates :phone, format: { with: /\A0[0-9]{8,9}\z/ }, allow_nil: true
```

### แบบฝึกหัดที่ 13
เพิ่ม i18n translations สำหรับ validation errors ในภาษาไทย

**เฉลย:**
```yaml
# config/locales/th.yml
th:
  activerecord:
    errors:
      messages:
        blank: "ไม่สามารถเว้นว่างได้"
        taken: "ถูกใช้งานแล้ว"
        too_short: "สั้นเกินไป (ขั้นต่ำ %{count} ตัวอักษร)"
        invalid: "ไม่ถูกต้อง"
```

### แบบฝึกหัดที่ 14
สร้าง uniqueness validation ที่ scoped

**เฉลย:**
```ruby
# Username unique ภายใน team
validates :username, uniqueness: {
  scope: :team_id,
  message: "ชื่อนี้มีใน team แล้ว"
}
```

### แบบฝึกหัดที่ 15
validate ว่า age เป็น integer ระหว่าง 0-150

**เฉลย:**
```ruby
validates :age, numericality: {
  only_integer: true,
  greater_than_or_equal_to: 0,
  less_than_or_equal_to: 150,
  message: "อายุต้องอยู่ระหว่าง 0-150 ปี"
}, allow_nil: true
```

### แบบฝึกหัดที่ 16
สร้าง custom validator สำหรับ Thai phone number

**เฉลย:**
```ruby
# app/validators/thai_phone_validator.rb
class ThaiPhoneValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank? && options[:allow_blank]
    
    cleaned = value.to_s.gsub(/[\s\-]/, "")
    unless cleaned.match?(/\A0[0-9]{8,9}\z/)
      record.errors.add(attribute, options[:message] || "เบอร์โทรศัพท์ไม่ถูกต้อง")
    end
  end
end

# ใช้งาน
class User < ApplicationRecord
  validates :phone, thai_phone: { allow_blank: true }
end
```

### แบบฝึกหัดที่ 17
ตรวจสอบ valid? และ invalid? พร้อมแสดง errors

**เฉลย:**
```ruby
user = User.new(email: "invalid-email")
user.valid?    # => false

user.errors.any?   # => true
user.errors.full_messages  # => ["Name can't be blank", "Email is invalid"]
user.errors[:email]  # => ["is invalid"]

# ดี errors ทั้งหมด
user.errors.each do |error|
  puts "#{error.attribute.to_s.humanize}: #{error.message}"
end
```

### แบบฝึกหัดที่ 18
ใช้ validation บน custom context

**เฉลย:**
```ruby
class Checkout < ApplicationRecord
  validates :payment_method, presence: true, on: :payment
  validates :shipping_address, presence: true, on: :shipping
  
  validates :items, presence: true
end

# รัน validation เฉพาะ context
checkout.valid?(:payment)
checkout.valid?(:shipping)

# Save พร้อม context
checkout.save(context: :payment)
```

### แบบฝึกหัดที่ 19
เขียน test สำหรับ User validations

**เฉลย:**
```ruby
# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  describe "validations" do
    it { is_expected.to validate_presence_of(:name) }
    it { is_expected.to validate_presence_of(:email) }
    it { is_expected.to validate_uniqueness_of(:email).case_insensitive }
    it { is_expected.to validate_length_of(:password).is_at_least(8) }
    
    describe "email format" do
      it "rejects invalid email" do
        user = build(:user, email: "not-an-email")
        expect(user).not_to be_valid
        expect(user.errors[:email]).to include("รูปแบบอีเมลไม่ถูกต้อง")
      end
      
      it "accepts valid email" do
        user = build(:user, email: "valid@example.com")
        expect(user).to be_valid
      end
    end
  end
end
```

### แบบฝึกหัดที่ 20
สร้าง model พร้อม validations ครบถ้วน

**เฉลย:**
```ruby
class Article < ApplicationRecord
  enum status: { draft: 0, published: 1, archived: 2 }, _default: :draft
  
  validates :title,
    presence: { message: "กรุณากรอกหัวข้อ" },
    length: { minimum: 5, maximum: 200 },
    uniqueness: { scope: :author_id }
  
  validates :content,
    presence: { message: "กรุณากรอกเนื้อหา" },
    length: { minimum: 100 }
  
  validates :status,
    inclusion: { in: statuses.keys }
  
  validates :published_at,
    presence: { message: "กรุณาระบุวันที่เผยแพร่", if: :published? }
  
  validate :published_at_in_valid_range, if: :published_at_changed?
  
  private
  
  def published_at_in_valid_range
    return unless published_at
    
    if published_at < 1.year.ago
      errors.add(:published_at, "ไม่สามารถย้อนหลังเกิน 1 ปีได้")
    end
    
    if published_at > 1.year.from_now
      errors.add(:published_at, "ไม่สามารถกำหนดเกิน 1 ปีข้างหน้าได้")
    end
  end
end
```

---

## สรุป

ใน Validations เราได้เรียนรู้:

1. **presence** - ต้องมีค่า
2. **length** - ความยาว min/max/in/is
3. **numericality** - ตรวจสอบตัวเลข
4. **format** - ตรวจสอบ format ด้วย regex
5. **inclusion/exclusion** - อยู่ใน/ไม่อยู่ใน list
6. **uniqueness** - ต้องไม่ซ้ำ (รวม scope)
7. **confirmation** - ยืนยัน (password, email)
8. **acceptance** - checkbox ต้องถูก check
9. **Custom validate methods** - logic เฉพาะ
10. **Custom validator classes** - reusable validators
11. **Conditional validations** - if/unless/on
12. **valid?/invalid?/errors** - ตรวจสอบ validity
13. **Error messages** - customization ด้วย i18n
14. **Testing validations** - RSpec matchers
