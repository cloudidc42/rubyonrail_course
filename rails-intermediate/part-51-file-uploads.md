# ตอนที่ 51: File Uploads กับ ActiveStorage (Steps 1121-1140)

## บทนำ

ActiveStorage คือ Rails framework สำหรับจัดการ file attachments มาตั้งแต่ Rails 5.2 รองรับ local storage, Amazon S3, Google Cloud Storage และ Microsoft Azure Blob Storage

---

## Step 1121: ActiveStorage Setup

### Installation

```bash
# ใน Rails project ที่มีอยู่แล้ว
rails active_storage:install
rails db:migrate

# สร้างไฟล์ 3 tables:
# active_storage_blobs      - metadata ของ file
# active_storage_attachments - link ระหว่าง blob กับ model
# active_storage_variant_records - cached variants
```

### Configuration

```ruby
# config/environments/development.rb
config.active_storage.service = :local

# config/environments/production.rb
config.active_storage.service = :amazon  # หรือ :google, :azure
```

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: ap-southeast-1
  bucket: <%= ENV['AWS_BUCKET_NAME'] %>

google:
  service: GCS
  project: <%= ENV['GCP_PROJECT'] %>
  credentials: <%= Rails.root.join("config/google-credentials.json") %>
  bucket: <%= ENV['GCP_BUCKET'] %>

azure:
  service: AzureStorage
  storage_account_name: <%= ENV['AZURE_STORAGE_ACCOUNT_NAME'] %>
  storage_access_key: <%= ENV['AZURE_STORAGE_ACCESS_KEY'] %>
  container: <%= ENV['AZURE_STORAGE_CONTAINER'] %>

# Test storage
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>
```

---

## Step 1122: has_one_attached และ has_many_attached

### Single File Attachment

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
  has_one_attached :cover_photo
  has_one_attached :resume, dependent: :purge_later  # ลบ file เมื่อ user ถูกลบ
  
  validates :avatar, 
    content_type: ['image/png', 'image/jpg', 'image/jpeg', 'image/gif'],
    size: { less_than: 5.megabytes }
end

# การใช้งาน
user = User.first
user.avatar.attached?  # => true/false
user.avatar.filename   # => "photo.jpg"
user.avatar.content_type # => "image/jpeg"
user.avatar.byte_size  # => 102400
url_for(user.avatar)   # => URL ของ file
```

### Multiple Files Attachment

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  has_one_attached :featured_image
  has_many_attached :photos
  has_many_attached :documents
  
  validate :acceptable_featured_image
  validate :acceptable_photos
  
  private
  
  def acceptable_featured_image
    return unless featured_image.attached?
    
    unless featured_image.content_type.in?(%w[image/jpeg image/png image/gif image/webp])
      errors.add(:featured_image, "must be a JPEG, PNG, GIF, or WebP image")
    end
    
    if featured_image.byte_size > 10.megabytes
      errors.add(:featured_image, "is too large (max 10MB)")
    end
  end
  
  def acceptable_photos
    return unless photos.attached?
    
    photos.each do |photo|
      unless photo.content_type.in?(%w[image/jpeg image/png image/gif image/webp])
        errors.add(:photos, "must be JPEG, PNG, GIF, or WebP")
        break
      end
      
      if photo.byte_size > 5.megabytes
        errors.add(:photos, "each photo must be less than 5MB")
        break
      end
    end
  end
end
```

---

## Step 1123: Attaching Files in Forms

### Single File Upload Form

```erb
<%# app/views/users/_form.html.erb %>
<%= form_with model: @user, multipart: true do |form| %>
  <div class="field">
    <%= form.label :name, "ชื่อ" %>
    <%= form.text_field :name, class: "form-control" %>
  </div>
  
  <div class="field">
    <%= form.label :avatar, "รูปโปรไฟล์" %>
    
    <%# แสดงรูปปัจจุบัน %>
    <% if @user.avatar.attached? %>
      <%= image_tag @user.avatar.variant(resize_to_limit: [100, 100]), class: "preview-image" %>
      <label>
        <%= check_box_tag :remove_avatar %>
        ลบรูปโปรไฟล์
      </label>
    <% end %>
    
    <%= form.file_field :avatar, 
                        accept: "image/*",
                        class: "form-control",
                        data: { 
                          controller: "preview",
                          preview_target: "input"
                        } %>
    
    <%# Preview ก่อน upload %>
    <img data-preview-target="output" style="max-width: 200px; display: none;">
  </div>
  
  <%= form.submit "บันทึก", class: "btn btn-primary" %>
<% end %>
```

### Multiple Files Upload Form

```erb
<%# app/views/articles/_form.html.erb %>
<%= form_with model: @article, multipart: true do |form| %>
  <div class="field">
    <%= form.label :featured_image, "รูปหน้าปก" %>
    <%= form.file_field :featured_image, accept: "image/*" %>
  </div>
  
  <div class="field">
    <%= form.label :photos, "รูปภาพประกอบ (เลือกได้หลายรูป)" %>
    <%= form.file_field :photos, 
                        multiple: true,
                        accept: "image/*",
                        class: "form-control" %>
    
    <%# แสดงรูปที่มีอยู่แล้ว %>
    <% if @article.photos.attached? %>
      <div class="existing-photos">
        <% @article.photos.each do |photo| %>
          <div class="photo-item">
            <%= image_tag photo.variant(resize_to_limit: [200, 200]), class: "thumbnail" %>
            <%= link_to "ลบ", remove_photo_article_path(@article, photo_id: photo.id), 
                        method: :delete, 
                        data: { confirm: "ต้องการลบรูปนี้?" } %>
          </div>
        <% end %>
      </div>
    <% end %>
  </div>
  
  <div class="field">
    <%= form.label :documents, "เอกสารแนบ (PDF, Word)" %>
    <%= form.file_field :documents, 
                        multiple: true, 
                        accept: ".pdf,.doc,.docx" %>
  </div>
  
  <%= form.submit %>
<% end %>
```

### Controller

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  def update
    if @user.update(user_params)
      # Handle avatar removal
      @user.avatar.purge if params[:remove_avatar] == "1"
      
      redirect_to @user, notice: "อัปเดตสำเร็จ"
    else
      render :edit
    end
  end
  
  private
  
  def user_params
    params.require(:user).permit(:name, :email, :bio, :avatar, :cover_photo)
  end
end

# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  def update
    if @article.update(article_params)
      redirect_to @article, notice: "อัปเดตสำเร็จ"
    else
      render :edit
    end
  end
  
  # DELETE /articles/:id/remove_photo
  def remove_photo
    photo = @article.photos.find_by_blob_id!(params[:photo_id]) rescue nil
    
    if photo
      photo.purge
      redirect_to edit_article_path(@article), notice: "ลบรูปสำเร็จ"
    else
      redirect_to edit_article_path(@article), alert: "ไม่พบรูปที่ต้องการลบ"
    end
  end
  
  private
  
  def article_params
    params.require(:article).permit(:title, :body, :published, :featured_image, photos: [], documents: [])
  end
end
```

---

## Step 1124: Image Processing ด้วย image_processing

### Setup

```ruby
# Gemfile
gem 'image_processing', '~> 1.2'

# ต้องติดตั้ง ImageMagick หรือ libvips
# macOS: brew install imagemagick libvips
# Ubuntu: apt-get install imagemagick libvips
```

### Image Variants

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
  
  def avatar_thumbnail
    avatar.variant(
      resize_to_fill: [50, 50],  # crop ให้พอดี 50x50
      format: :webp,
      quality: 80
    )
  end
  
  def avatar_medium
    avatar.variant(
      resize_to_limit: [300, 300],  # ย่อให้ไม่เกิน 300x300
      format: :jpeg,
      quality: 85
    )
  end
  
  def avatar_large
    avatar.variant(
      resize_to_limit: [600, 600],
      format: :jpeg,
      quality: 90
    )
  end
end
```

### Variants ใน Views

```erb
<%# app/views/users/show.html.erb %>

<%# Thumbnail 50x50 %>
<%= image_tag @user.avatar.variant(resize_to_fill: [50, 50]), 
              class: "avatar-sm",
              alt: @user.name %>

<%# Medium 300x300 %>
<%= image_tag @user.avatar.variant(resize_to_limit: [300, 300]),
              class: "avatar-md" %>

<%# ด้วย lazy loading %>
<%= image_tag @user.avatar.variant(resize_to_limit: [200, 200]),
              loading: "lazy",
              class: "profile-photo" %>

<%# WebP format %>
<picture>
  <source srcset="<%= url_for(@user.avatar.variant(format: :webp, resize_to_limit: [300, 300])) %>" type="image/webp">
  <img src="<%= url_for(@user.avatar.variant(format: :jpeg, resize_to_limit: [300, 300])) %>" alt="<%= @user.name %>">
</picture>
```

### Advanced Transformations

```ruby
# Crop เฉพาะส่วน
variant = avatar.variant(
  crop: [100, 100, 200, 200],  # [x, y, width, height]
  format: :jpeg
)

# Rotate
variant = photo.variant(
  rotate: 90,
  format: :jpeg
)

# Convert format
pdf_preview = document.variant(
  format: :png,
  saver: { quality: 80 }
)

# Grayscale
thumbnail = photo.variant(
  resize_to_fill: [100, 100],
  colourspace: :grey16
)

# Multiple transforms
avatar.variant(
  resize_and_pad: [200, 200, background: :white],
  format: :jpeg,
  quality: 85
)
```

---

## Step 1125: Direct Uploads to S3

### Setup Direct Upload

```ruby
# app/javascript/application.js
import * as ActiveStorage from "@rails/activestorage"
ActiveStorage.start()
```

```erb
<%# ใน form %>
<%= form.file_field :photos, 
                    multiple: true,
                    direct_upload: true,
                    class: "form-control",
                    data: { 
                      controller: "upload-progress"
                    } %>

<div class="upload-progress" data-upload-progress-target="progress" hidden>
  <div class="progress-bar">
    <div class="progress-fill" data-upload-progress-target="fill"></div>
  </div>
  <span data-upload-progress-target="text">0%</span>
</div>
```

```javascript
// app/javascript/controllers/upload_progress_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["progress", "fill", "text"]
  
  connect() {
    this.element.addEventListener("direct-upload:initialize", this.onInit.bind(this))
    this.element.addEventListener("direct-upload:start", this.onStart.bind(this))
    this.element.addEventListener("direct-upload:progress", this.onProgress.bind(this))
    this.element.addEventListener("direct-upload:error", this.onError.bind(this))
    this.element.addEventListener("direct-upload:end", this.onEnd.bind(this))
  }
  
  onInit(event) {
    this.progressTarget.hidden = false
  }
  
  onStart(event) {
    this.fillTarget.style.width = "0%"
  }
  
  onProgress(event) {
    const progress = event.detail.progress
    this.fillTarget.style.width = `${progress}%`
    this.textTarget.textContent = `${Math.round(progress)}%`
  }
  
  onError(event) {
    event.preventDefault()
    this.textTarget.textContent = "Upload failed!"
  }
  
  onEnd(event) {
    this.textTarget.textContent = "Upload complete!"
    setTimeout(() => { this.progressTarget.hidden = true }, 2000)
  }
}
```

### CORS Configuration สำหรับ S3

```xml
<!-- S3 Bucket CORS Configuration -->
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>https://yourdomain.com</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedMethod>PUT</AllowedMethod>
    <AllowedMethod>POST</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
    <ExposeHeader>ETag</ExposeHeader>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
  </CORSRule>
</CORSConfiguration>
```

---

## Step 1126: Storage Services

### Local Storage

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
  
# URL ที่ได้: http://localhost:3000/rails/active_storage/blobs/...
```

### Amazon S3

```ruby
# Gemfile
gem 'aws-sdk-s3', require: false
```

```yaml
amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: ap-southeast-1
  bucket: mybucket-production
  
  # IAM Role (แนะนำสำหรับ EC2/ECS)
  # ไม่ต้องระบุ access_key_id และ secret_access_key
  
  # Public access
  public: false  # หรือ true สำหรับ public files
  
  # Server-side encryption
  upload: { server_side_encryption: "AES256" }
```

### IAM Policy สำหรับ S3

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::mybucket-production/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::mybucket-production"
    }
  ]
}
```

### Google Cloud Storage

```ruby
# Gemfile
gem 'google-cloud-storage', '~> 1.11', require: false
```

```yaml
google:
  service: GCS
  project: my-gcp-project
  credentials: <%= Rails.root.join("config/gcs-credentials.json") %>
  bucket: mybucket-production
```

---

## Step 1127: Displaying Files in Views

### Images

```erb
<%# Basic image display %>
<%= image_tag @user.avatar if @user.avatar.attached? %>

<%# With variant %>
<%= image_tag @user.avatar.variant(resize_to_fill: [100, 100]) if @user.avatar.attached? %>

<%# With fallback %>
<% if @user.avatar.attached? %>
  <%= image_tag @user.avatar.variant(resize_to_limit: [200, 200]), 
                alt: @user.name,
                loading: "lazy" %>
<% else %>
  <%= image_tag "default-avatar.png", alt: "Default avatar" %>
<% end %>

<%# Direct URL %>
<img src="<%= url_for(@user.avatar) %>" alt="<%= @user.name %>">
```

### Documents

```erb
<%# Download link %>
<% @article.documents.each do |doc| %>
  <div class="document">
    <%= link_to doc.filename.to_s, url_for(doc), 
                target: "_blank",
                data: { turbo: false } %>
    <span class="file-size"><%= number_to_human_size(doc.byte_size) %></span>
    <span class="file-type"><%= doc.content_type %></span>
  </div>
<% end %>
```

### Video/Audio

```erb
<%# Video player %>
<% if @product.demo_video.attached? %>
  <video controls width="640" height="360">
    <source src="<%= url_for(@product.demo_video) %>" 
            type="<%= @product.demo_video.content_type %>">
    Browser ของคุณไม่รองรับ video
  </video>
<% end %>
```

---

## Step 1128: Validation of Attachments

### Custom Validation

```ruby
# app/validators/file_size_validator.rb
class FileSizeValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return unless value.attached?
    
    max_size = options[:max] || 10.megabytes
    
    attachments = value.respond_to?(:each) ? value : [value]
    
    attachments.each do |attachment|
      if attachment.byte_size > max_size
        record.errors.add(attribute, 
          "is too large (max #{ActiveSupport::NumberHelper.number_to_human_size(max_size)})")
      end
    end
  end
end

# app/validators/content_type_validator.rb
class ContentTypeValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return unless value.attached?
    
    allowed_types = options[:in] || []
    
    attachments = value.respond_to?(:each) ? value : [value]
    
    attachments.each do |attachment|
      unless allowed_types.include?(attachment.content_type)
        record.errors.add(attribute, 
          "must be #{allowed_types.join(' or ')} (got #{attachment.content_type})")
      end
    end
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
  
  validates :avatar, 
    file_size: { max: 5.megabytes },
    content_type: { in: ['image/jpeg', 'image/png', 'image/gif', 'image/webp'] },
    if: -> { avatar.attached? }
end

# app/models/article.rb
class Article < ApplicationRecord
  has_many_attached :photos
  has_many_attached :documents
  
  validates :photos,
    file_size: { max: 10.megabytes },
    content_type: { in: ['image/jpeg', 'image/png', 'image/webp'] }
  
  validates :documents,
    file_size: { max: 25.megabytes },
    content_type: { in: ['application/pdf', 'application/msword', 
                         'application/vnd.openxmlformats-officedocument.wordprocessingml.document'] }
end
```

### ActiveStorage Validations Gem

```ruby
# Gemfile (alternative)
gem 'active_storage_validations'

# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
  
  validates :avatar,
    attached: true,  # ต้องมีไฟล์
    content_type: {
      in: ['image/jpeg', 'image/png', 'image/gif'],
      message: "ต้องเป็นรูปภาพ"
    },
    size: {
      less_than: 5.megabytes,
      message: "ขนาดไม่เกิน 5MB"
    },
    dimension: {
      width: { min: 100, max: 2000 },
      height: { min: 100, max: 2000 },
      message: "ขนาดรูปต้องระหว่าง 100x100 ถึง 2000x2000 pixels"
    }
end
```

---

## Step 1129: Deleting Attachments

### Purge Attachment

```ruby
# ลบ file ทันที
user.avatar.purge

# ลบ file ใน background job (แนะนำ)
user.avatar.purge_later

# ลบ specific photo จาก has_many_attached
photo_to_delete = @article.photos.find_by_blob_id(blob_id)
photo_to_delete.purge

# ลบทุก photos
@article.photos.purge
```

### Controller สำหรับ Delete

```ruby
# app/controllers/attachments_controller.rb
class AttachmentsController < ApplicationController
  before_action :authenticate_user!
  
  def destroy
    attachment = ActiveStorage::Attachment.find(params[:id])
    
    # ตรวจสอบ ownership
    unless can_delete?(attachment)
      return render json: { error: "ไม่มีสิทธิ์ลบไฟล์นี้" }, status: :forbidden
    end
    
    attachment.purge_later
    
    respond_to do |format|
      format.json { render json: { success: true } }
      format.html { redirect_back fallback_location: root_path, notice: "ลบไฟล์สำเร็จ" }
    end
  end
  
  private
  
  def can_delete?(attachment)
    record = attachment.record
    case record
    when User
      record == current_user || current_user.admin?
    when Article
      record.user == current_user || current_user.admin?
    else
      current_user.admin?
    end
  end
end
```

---

## Step 1130: Security Considerations

### Secure File URLs

```ruby
# config/environments/production.rb

# URL ที่ expire หลังจาก 30 นาที (สำหรับ private files)
config.active_storage.service_urls_expire_in = 30.minutes

# ป้องกัน public access โดยตรง
config.active_storage.draw_routes = true
```

```ruby
# app/controllers/downloads_controller.rb
class DownloadsController < ApplicationController
  before_action :authenticate_user!
  before_action :authorize_download!
  
  def show
    blob = ActiveStorage::Blob.find_signed!(params[:signed_id])
    
    # Redirect ไปยัง signed URL (expire ใน 1 ชั่วโมง)
    redirect_to blob.url(expires_in: 1.hour, disposition: :attachment)
  end
  
  private
  
  def authorize_download!
    # ตรวจสอบว่าผู้ใช้มีสิทธิ์ download
    unless current_user.can_download?(params[:signed_id])
      render json: { error: "Unauthorized" }, status: :unauthorized
    end
  end
end
```

### ป้องกัน File Type Spoofing

```ruby
# ตรวจสอบ actual file type ไม่ใช่แค่ extension
require 'open3'

class FileTypeValidator
  MAGIC_NUMBERS = {
    "\x89PNG\r\n\x1a\n" => 'image/png',
    "\xFF\xD8\xFF" => 'image/jpeg',
    "GIF87a" => 'image/gif',
    "GIF89a" => 'image/gif',
    "%PDF" => 'application/pdf'
  }
  
  def self.valid_type?(file, expected_type)
    header = File.read(file.path, 8)
    detected_type = MAGIC_NUMBERS.find { |magic, _| header.start_with?(magic) }&.last
    detected_type == expected_type
  end
end
```

---

## แบบฝึกหัด (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** Setup ActiveStorage ใน Rails project ใหม่
```bash
# เฉลย
rails active_storage:install
rails db:migrate
```

**ข้อ 2:** เพิ่ม avatar attachment ใน User model
```ruby
# เฉลย
class User < ApplicationRecord
  has_one_attached :avatar
end
```

**ข้อ 3:** สร้าง form สำหรับ upload avatar
```erb
<%# เฉลย %>
<%= form_with model: @user, multipart: true do |f| %>
  <%= f.file_field :avatar, accept: "image/*" %>
  <%= f.submit %>
<% end %>
```

**ข้อ 4:** แสดงรูป avatar ใน view พร้อม fallback
```erb
<%# เฉลย %>
<% if @user.avatar.attached? %>
  <%= image_tag @user.avatar.variant(resize_to_fill: [100, 100]) %>
<% else %>
  <%= image_tag "default-avatar.png" %>
<% end %>
```

**ข้อ 5:** เพิ่ม validation ตรวจสอบ file type และ size
```ruby
# เฉลย
validates :avatar,
  content_type: { in: ['image/jpeg', 'image/png', 'image/gif'] },
  size: { less_than: 5.megabytes },
  if: -> { avatar.attached? }
```

### ระดับกลาง

**ข้อ 6:** ตั้งค่า S3 storage สำหรับ production
```yaml
# เฉลย - config/storage.yml
amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: ap-southeast-1
  bucket: <%= ENV['AWS_BUCKET'] %>
```

**ข้อ 7:** implement direct upload ไปยัง S3
```erb
<%# เฉลย %>
<%= form.file_field :photos, 
                    multiple: true,
                    direct_upload: true %>
```

**ข้อ 8:** สร้าง image variant สำหรับ thumbnail, medium, large
```ruby
# เฉลย
def thumbnail
  avatar.variant(resize_to_fill: [50, 50], format: :webp)
end

def medium
  avatar.variant(resize_to_limit: [300, 300], format: :jpeg, quality: 85)
end
```

**ข้อ 9:** เพิ่ม delete attachment functionality
```ruby
# เฉลย
def remove_photo
  photo = @article.photos.find_by_blob_id!(params[:photo_id])
  photo.purge_later
  redirect_to @article
end
```

**ข้อ 10:** เพิ่ม upload progress bar ด้วย Stimulus
```javascript
// เฉลย - implement ในข้อ 10
addEventListener("direct-upload:progress", event => {
  const { progress } = event.detail
  progressBar.style.width = `${progress}%`
})
```

### ระดับสูง

**ข้อ 11-20:** (แบบฝึกหัดเพิ่มเติม)

```ruby
# เฉลย ข้อ 11 - Batch image processing
class ProcessImagesJob < ApplicationJob
  def perform(article_id)
    article = Article.find(article_id)
    article.photos.each do |photo|
      # Pre-generate variants
      photo.variant(resize_to_limit: [800, 800]).processed
      photo.variant(resize_to_fill: [400, 400]).processed
      photo.variant(resize_to_fill: [100, 100]).processed
    end
  end
end
```

```ruby
# เฉลย ข้อ 14 - Custom Storage Service
class SecureFileService
  def self.url_for(attachment, user:)
    unless user.can_access?(attachment.record)
      raise "Unauthorized"
    end
    
    attachment.blob.url(expires_in: 1.hour, disposition: :attachment)
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ActiveStorage Setup** - installation และ configuration
2. **Attachments** - has_one_attached และ has_many_attached
3. **Forms** - file upload forms
4. **Image Processing** - variants, resize, crop, convert
5. **Direct Upload** - upload ตรงไปยัง S3
6. **Storage Services** - Local, S3, GCS
7. **Validation** - ตรวจสอบ type, size, dimensions
8. **Security** - secure URLs, type spoofing prevention

ActiveStorage เป็นส่วนสำคัญของ modern Rails apps โดยเฉพาะที่ต้องจัดการกับ user-uploaded content
