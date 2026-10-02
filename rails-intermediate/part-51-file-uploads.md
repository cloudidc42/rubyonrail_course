# Part 51: File Uploads ด้วย Active Storage

## ขั้นตอนที่ 1121-1140: จัดการไฟล์อัพโหลด

---

## ขั้นตอนที่ 1121: Active Storage Setup

```bash
# ติดตั้ง Active Storage
rails active_storage:install
rails db:migrate
# สร้าง active_storage_blobs, active_storage_attachments, active_storage_variant_records tables
```

```ruby
# config/storage.yml
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: ap-southeast-1
  bucket: <%= ENV['AWS_S3_BUCKET'] %>

google:
  service: GCS
  project: <%= ENV['GCS_PROJECT'] %>
  credentials: <%= ENV['GCS_CREDENTIALS'] %>
  bucket: <%= ENV['GCS_BUCKET'] %>

azure:
  service: AzureStorage
  storage_account_name: <%= ENV['AZURE_STORAGE_ACCOUNT'] %>
  storage_access_key: <%= ENV['AZURE_STORAGE_KEY'] %>
  container: <%= ENV['AZURE_STORAGE_CONTAINER'] %>
```

```ruby
# config/environments/development.rb
config.active_storage.service = :local

# config/environments/production.rb
config.active_storage.service = :amazon
```

## ขั้นตอนที่ 1122: has_one_attached

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
  has_one_attached :resume
  
  validates :avatar, content_type: ['image/png', 'image/jpg', 'image/jpeg'],
                     size: { less_than: 5.megabytes }
end
```

```ruby
# Controller
class UsersController < ApplicationController
  def update
    @user = current_user
    if @user.update(user_params)
      redirect_to profile_path, notice: "อัพเดทโปรไฟล์แล้ว"
    else
      render :edit
    end
  end
  
  private
  
  def user_params
    params.require(:user).permit(:name, :email, :avatar, :resume)
  end
end
```

```erb
<%# View %>
<%= form_with model: @user do |f| %>
  <div class="field">
    <%= f.label :avatar, "รูปโปรไฟล์" %>
    
    <% if @user.avatar.attached? %>
      <%= image_tag @user.avatar.variant(resize_to_fill: [150, 150]) %>
      <%= link_to "ลบรูป", purge_avatar_user_path(@user), method: :delete,
                  data: { confirm: "ต้องการลบรูปใช่หรือไม่?" } %>
    <% end %>
    
    <%= f.file_field :avatar, accept: "image/*" %>
  </div>
  
  <%= f.submit "บันทึก" %>
<% end %>
```

## ขั้นตอนที่ 1123: has_many_attached

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many_attached :images
  has_many_attached :documents
  
  validates :images, content_type: { in: ['image/png', 'image/jpg', 'image/jpeg', 'image/gif'],
                                     message: "ต้องเป็นไฟล์รูปภาพ" },
                     size: { less_than: 10.megabytes,
                             message: "ไฟล์ต้องมีขนาดไม่เกิน 10MB" }
  
  validates :documents, content_type: ['application/pdf', 'application/msword'],
                        size: { less_than: 20.megabytes }
end
```

```ruby
# Controller
def create
  @post = current_user.posts.build(post_params)
  if @post.save
    redirect_to @post
  else
    render :new
  end
end

def update
  if @post.update(post_params)
    redirect_to @post
  else
    render :edit
  end
end

private

def post_params
  params.require(:post).permit(:title, :content, images: [], documents: [])
end
```

```erb
<%# Multiple file upload %>
<%= form_with model: @post do |f| %>
  <div class="field">
    <%= f.label :images, "รูปภาพ (เลือกได้หลายรูป)" %>
    <%= f.file_field :images, multiple: true, accept: "image/*",
                    data: { controller: "file-preview" } %>
  </div>
  
  <%# แสดงรูปที่อัพโหลดแล้ว %>
  <div class="existing-images">
    <% @post.images.each do |image| %>
      <div class="image-item">
        <%= image_tag image.variant(resize_to_fill: [200, 200]) %>
        <%= link_to "ลบ", remove_image_post_path(@post, image_id: image.id),
                    method: :delete, data: { confirm: "ลบรูปนี้?" } %>
      </div>
    <% end %>
  </div>
  
  <%= f.submit "บันทึก" %>
<% end %>
```

## ขั้นตอนที่ 1124: Image Processing

```ruby
# Gemfile
gem 'image_processing', '~> 1.2'

# bundle install
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb, resize_to_fill: [100, 100]
    attachable.variant :medium, resize_to_fill: [300, 300]
    attachable.variant :profile, resize_to_limit: [500, 500]
  end
end

# ใน view
image_tag current_user.avatar.variant(:thumb)
image_tag current_user.avatar.variant(:profile)
```

```ruby
# Variants ขั้นสูง
user.avatar.variant(
  resize_to_fill: [300, 300],
  format: :webp,
  quality: 85,
  auto_orient: true,
  strip_icc_profile: true  # ลบ metadata
)

# Resize ด้วย ImageMagick options
user.avatar.variant(
  resize_to_limit: [800, 600],
  format: 'jpg',
  saver: { quality: 90 }
)

# Convert format
user.avatar.variant(convert: "png")
```

## ขั้นตอนที่ 1125: Direct Upload

```ruby
# Gemfile
gem 'activestorage'  # ติดตั้งอยู่แล้วใน Rails

# config/environments/development.rb
config.active_storage.service = :local
```

```erb
<%# Enable direct upload %>
<%= javascript_include_tag 'activestorage' %>

<%# หรือใน importmap %>
<%# config/importmap.rb %>
<%# pin "@rails/activestorage", to: "activestorage.esm.js" %>
```

```javascript
// app/javascript/application.js
import * as ActiveStorage from "@rails/activestorage"
ActiveStorage.start()
```

```erb
<%# Direct upload in form %>
<%= form_with model: @post do |f| %>
  <%= f.file_field :images, multiple: true, 
                   direct_upload: true,
                   data: { 
                     controller: "upload-progress",
                     action: "direct-upload:progress->upload-progress#progress"
                   } %>
  
  <div class="upload-progress" data-upload-progress-target="bar" style="display:none">
    <div class="progress-bar" style="width: 0%"></div>
  </div>
<% end %>
```

```javascript
// app/javascript/controllers/upload_progress_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["bar"]
  
  progress({ detail: { progress } }) {
    const percentage = Math.round(progress * 100)
    
    this.barTarget.style.display = "block"
    this.barTarget.querySelector('.progress-bar').style.width = `${percentage}%`
    
    if (percentage === 100) {
      setTimeout(() => this.barTarget.style.display = "none", 1000)
    }
  }
}
```

## ขั้นตอนที่ 1126: S3 Configuration

```ruby
# Gemfile
gem 'aws-sdk-s3', require: false

# bundle install
```

```ruby
# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: <%= ENV['AWS_REGION'] || 'ap-southeast-1' %>
  bucket: <%= ENV['AWS_S3_BUCKET'] %>
  
  # Optional: Specify endpoint for compatible services
  # endpoint: "https://nyc3.digitaloceanspaces.com"
  
  # Upload settings
  upload:
    server_side_encryption: "AES256"
    multipart_threshold: 100.megabytes
```

```ruby
# config/environments/production.rb
config.active_storage.service = :amazon

# Expire signed URLs (สำหรับ private files)
config.active_storage.service_urls_expire_in = 1.hour
```

```ruby
# Public vs Private files
# Public - ทุกคนเข้าถึงได้
class Post < ApplicationRecord
  has_one_attached :cover_image  # สำหรับรูปที่เปิด public
end

# Private - ต้อง signed URL
class Document < ApplicationRecord
  has_one_attached :file  # ต้องผ่าน authentication
  
  def download_url
    Rails.application.routes.url_helpers.url_for(file)
    # จะสร้าง pre-signed URL อัตโนมัติ
  end
end
```

## ขั้นตอนที่ 1127: GCS Configuration

```ruby
# Gemfile
gem 'google-cloud-storage', '~> 1.11', require: false

# config/storage.yml
google:
  service: GCS
  project: <%= ENV['GCS_PROJECT'] %>
  credentials: <%= JSON.parse(ENV['GCS_CREDENTIALS']) %>
  bucket: <%= ENV['GCS_BUCKET'] %>
  cache_control: "public, max-age=3600"
```

## ขั้นตอนที่ 1128: File Validations

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many_attached :attachments
  
  # Custom validation
  validate :attachments_size_and_type
  
  private
  
  def attachments_size_and_type
    attachments.each do |attachment|
      unless attachment.content_type.in?(%w[
        image/png image/jpg image/jpeg image/gif
        application/pdf application/msword
      ])
        errors.add(:attachments, "ประเภทไฟล์ #{attachment.content_type} ไม่รองรับ")
      end
      
      if attachment.blob.byte_size > 20.megabytes
        errors.add(:attachments, "ไฟล์ #{attachment.filename} ขนาดใหญ่เกิน 20MB")
      end
    end
    
    if attachments.count > 10
      errors.add(:attachments, "อัพโหลดได้ไม่เกิน 10 ไฟล์")
    end
  end
end
```

```ruby
# ใช้ active_storage_validations gem
# Gemfile
gem 'active_storage_validations'

class User < ApplicationRecord
  has_one_attached :avatar
  
  validates :avatar, 
    attached: true,
    content_type: ['image/png', 'image/jpg', 'image/jpeg'],
    size: { less_than: 5.megabytes },
    dimension: { 
      width: { min: 100, max: 2000 },
      height: { min: 100, max: 2000 },
      in: 100..2000
    }
end
```

## ขั้นตอนที่ 1129: Serving Files

```ruby
# routes.rb
Rails.application.routes.draw do
  resources :documents do
    member do
      get :download
    end
  end
end
```

```ruby
# app/controllers/documents_controller.rb
class DocumentsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_document
  
  def show
    # Inline display
    redirect_to url_for(@document.file)
  end
  
  def download
    # Force download
    if @document.file.attached?
      send_data @document.file.download,
                filename: @document.file.filename.to_s,
                type: @document.file.content_type,
                disposition: 'attachment'
    else
      redirect_to @document, alert: "ไม่พบไฟล์"
    end
  end
  
  private
  
  def set_document
    @document = current_user.documents.find(params[:id])
  end
end
```

## ขั้นตอนที่ 1130: Background Processing

```ruby
# Process images ใน background
class ImageProcessingJob < ApplicationJob
  queue_as :default
  
  def perform(attachment_id)
    attachment = ActiveStorage::Attachment.find(attachment_id)
    blob = attachment.blob
    
    return unless blob.image?
    
    # Generate variants
    blob.open do |file|
      processed = ImageProcessing::Vips
        .source(file)
        .resize_to_fill(800, 600)
        .convert("webp")
        .call
      
      # Store processed version
      attachment.record.optimized_image.attach(
        io: processed,
        filename: "#{blob.filename.base}.webp",
        content_type: "image/webp"
      )
    end
  end
end

# Trigger after upload
class Post < ApplicationRecord
  has_one_attached :image
  has_one_attached :optimized_image
  
  after_create_commit :process_image
  
  private
  
  def process_image
    ImageProcessingJob.perform_later(image.id) if image.attached?
  end
end
```

## ขั้นตอนที่ 1131: Purge / Delete Files

```ruby
# ลบไฟล์เดี่ยว
user.avatar.purge       # ลบทันที
user.avatar.purge_later # ลบใน background job

# ลบหลายไฟล์
post.images.purge
post.images.purge_later

# ลบไฟล์เฉพาะ
post.images.find { |img| img.filename == "old.jpg" }.purge

# ใน controller
def remove_image
  @post = current_user.posts.find(params[:id])
  image = @post.images.find_by_id(params[:image_id])
  
  if image
    image.purge
    redirect_to edit_post_path(@post), notice: "ลบรูปภาพแล้ว"
  else
    redirect_to edit_post_path(@post), alert: "ไม่พบรูปภาพ"
  end
end
```

## ขั้นตอนที่ 1132: Metadata Extraction

```ruby
# ดู metadata ของไฟล์
blob = user.avatar.blob

blob.filename      # => "photo.jpg"
blob.content_type  # => "image/jpeg"
blob.byte_size     # => 1234567
blob.checksum      # => "abc123..."
blob.metadata      # => { "identified" => true, "width" => 800, "height" => 600 }
blob.created_at    # => 2024-01-01 12:00:00

# Image metadata
if blob.image?
  puts blob.metadata["width"]
  puts blob.metadata["height"]
end

# Video metadata
if blob.video?
  puts blob.metadata["duration"]
end
```

---

## แบบฝึกหัด: File Uploads (20 ข้อ)

### ข้อที่ 1: Setup Active Storage
```bash
rails active_storage:install && rails db:migrate
```

### ข้อที่ 2: User Avatar
```
เพิ่ม avatar ให้ User model และ form สำหรับอัพโหลด
```

**เฉลย:**
```ruby
# model
has_one_attached :avatar

# view
<%= f.file_field :avatar, accept: "image/*" %>
<%= image_tag user.avatar.variant(resize_to_fill: [100, 100]) if user.avatar.attached? %>
```

### ข้อที่ 3: Multiple Images
```
Post ที่มี has_many_attached :images พร้อม validation
```

### ข้อที่ 4: Image Variants
```
สร้าง variants: thumb (100x100), medium (400x400), large (800x600)
```

### ข้อที่ 5: S3 Configuration
```
ตั้งค่า production ให้ใช้ Amazon S3
```

### ข้อที่ 6-20 (แบบสรุป)

**ข้อ 6:** Direct upload พร้อม progress bar
**ข้อ 7:** File validation (type, size, dimension)
**ข้อ 8:** Force download endpoint
**ข้อ 9:** Purge attachment
**ข้อ 10:** Image processing ใน background job
**ข้อ 11:** GCS setup
**ข้อ 12:** Display file metadata
**ข้อ 13:** Generate pre-signed URL
**ข้อ 14:** Drag-and-drop upload ด้วย Stimulus
**ข้อ 15:** Multiple file types (image + PDF)
**ข้อ 16:** Optimize images ก่อน save
**ข้อ 17:** Test file uploads ด้วย RSpec
**ข้อ 18:** CDN URL configuration
**ข้อ 19:** Migrate files ระหว่าง storage services
**ข้อ 20:** Track upload statistics

---

## สรุป: Active Storage

| Feature | Method |
|---------|--------|
| Single file | `has_one_attached` |
| Multiple files | `has_many_attached` |
| Image resize | `.variant(resize_to_fill: [w, h])` |
| Check attached | `.attached?` |
| Download | `.download` |
| Delete | `.purge` / `.purge_later` |
| Direct upload | `direct_upload: true` |

**Key Takeaways:**
1. ใช้ `purge_later` แทน `purge` เพื่อไม่บล็อก request
2. ตรวจสอบ content_type และ size เสมอ
3. ใช้ Direct Upload สำหรับไฟล์ขนาดใหญ่
4. สร้าง variants ไว้ล่วงหน้าเพื่อประสิทธิภาพ
5. ใช้ Cloud Storage (S3/GCS) ใน production

---

## Step 1131: Image Processing Pipeline

### Background Processing

```ruby
# app/jobs/process_avatar_job.rb
class ProcessAvatarJob < ApplicationJob
  queue_as :default
  
  def perform(user_id)
    user = User.find(user_id)
    return unless user.avatar.attached?
    
    # Pre-generate commonly used variants
    variants_to_process = [
      { resize_to_fill: [50, 50], format: :webp, quality: 80 },
      { resize_to_fill: [100, 100], format: :webp, quality: 85 },
      { resize_to_limit: [300, 300], format: :webp, quality: 90 },
      { resize_to_limit: [600, 600], format: :jpeg, quality: 90 }
    ]
    
    variants_to_process.each do |opts|
      user.avatar.variant(opts).processed  # Pre-process and cache
    end
    
    Rails.logger.info "Processed avatar variants for user #{user_id}"
  rescue => e
    Rails.logger.error "Failed to process avatar for user #{user_id}: #{e.message}"
  end
end
```

### Batch Image Processing

```ruby
# app/jobs/batch_process_images_job.rb
class BatchProcessImagesJob < ApplicationJob
  queue_as :low
  
  def perform(article_id)
    article = Article.find(article_id)
    return unless article.photos.attached?
    
    article.photos.each_with_index do |photo, index|
      next unless photo.content_type.start_with?('image/')
      
      begin
        # Generate multiple sizes
        [
          { resize_to_limit: [800, 600] },
          { resize_to_fill: [400, 300] },
          { resize_to_fill: [200, 150] }
        ].each { |opts| photo.variant(opts).processed }
        
        Rails.logger.info "Processed photo #{index + 1} of #{article.photos.count}"
      rescue => e
        Rails.logger.warn "Could not process photo #{index + 1}: #{e.message}"
      end
    end
  end
end
```

---

## Step 1132: ActiveStorage กับ Shrine Migration

### Migration จาก Shrine ไป ActiveStorage

```ruby
# lib/tasks/migrate_attachments.rake
namespace :attachments do
  desc "Migrate attachments from Shrine to ActiveStorage"
  task migrate: :environment do
    User.find_each do |user|
      next unless user.avatar_data.present?
      
      begin
        shrine_data = JSON.parse(user.avatar_data)
        file_path = shrine_data.dig('id')
        
        user.avatar.attach(
          io: URI.open("https://old-storage.example.com/#{file_path}"),
          filename: File.basename(file_path),
          content_type: shrine_data.dig('metadata', 'mime_type')
        )
        
        puts "Migrated avatar for user #{user.id}"
      rescue => e
        puts "Failed for user #{user.id}: #{e.message}"
      end
    end
  end
end
```

---

## Step 1133: File Upload API Endpoint

```ruby
# app/controllers/api/v1/uploads_controller.rb
module Api
  module V1
    class UploadsController < BaseController
      def create
        uploader = FileUploaderService.new(
          file: upload_params[:file],
          user: current_user,
          context: upload_params[:context]
        )
        
        if uploader.valid?
          result = uploader.upload!
          render json: {
            success: true,
            data: {
              id: result[:attachment].id,
              url: url_for(result[:attachment]),
              filename: result[:attachment].filename.to_s,
              content_type: result[:attachment].content_type,
              byte_size: result[:attachment].byte_size
            }
          }, status: :created
        else
          render json: { 
            success: false, 
            errors: uploader.errors 
          }, status: :unprocessable_entity
        end
      end
      
      def destroy
        attachment = ActiveStorage::Attachment.find(params[:id])
        
        unless can_delete?(attachment)
          return render json: { error: "Forbidden" }, status: :forbidden
        end
        
        attachment.purge_later
        head :no_content
      end
      
      private
      
      def upload_params
        params.permit(:file, :context)
      end
      
      def can_delete?(attachment)
        case attachment.record
        when User then attachment.record == current_user
        when Article then attachment.record.user == current_user
        else false
        end
      end
    end
  end
end
```

---

## Step 1134: Virus Scanning

```ruby
# Gemfile
# gem 'clamby'  # ClamAV integration

# app/services/virus_scanner_service.rb
class VirusScannerService
  def self.scan(blob)
    return true unless Rails.env.production?
    
    blob.open do |file|
      result = `clamscan --no-summary #{file.path}`
      $?.exitstatus == 0  # 0 = clean, 1 = virus found, 2 = error
    end
  rescue => e
    Rails.logger.error "Virus scan error: #{e.message}"
    true  # Allow on error (fail open) - adjust based on security requirements
  end
end

# app/jobs/scan_attachment_job.rb
class ScanAttachmentJob < ApplicationJob
  queue_as :default
  
  def perform(blob_id)
    blob = ActiveStorage::Blob.find(blob_id)
    
    unless VirusScannerService.scan(blob)
      Rails.logger.warn "Virus detected in blob #{blob_id}, purging"
      blob.purge
      
      # Notify admin
      AdminMailer.virus_detected(blob_id).deliver_later
    end
  end
end
```

---

## Step 1135: Storage Cleanup Jobs

```ruby
# app/jobs/cleanup_unattached_blobs_job.rb
class CleanupUnattachedBlobsJob < ApplicationJob
  queue_as :low
  
  def perform
    # ลบ blobs ที่ไม่มี attachment ที่เก่ากว่า 2 วัน
    unattached_blobs = ActiveStorage::Blob.unattached.where("active_storage_blobs.created_at <= ?", 2.days.ago)
    count = unattached_blobs.count
    
    unattached_blobs.find_each(&:purge)
    
    Rails.logger.info "Purged #{count} unattached blobs"
  end
end
```

```yaml
# config/sidekiq.yml
:schedule:
  cleanup_blobs:
    cron: '0 3 * * *'  # 3:00 AM daily
    class: CleanupUnattachedBlobsJob
    queue: low
```

---

## Step 1136: Content Delivery Network (CDN)

```ruby
# config/environments/production.rb
config.asset_host = ENV['CDN_HOST']  # "https://cdn.example.com"

# ActiveStorage URL proxy through CDN
config.active_storage.service_urls_expire_in = 1.week

# Custom URL generation
# app/helpers/storage_helper.rb
module StorageHelper
  def cdn_url_for(attachment)
    return nil unless attachment.attached?
    
    if Rails.env.production? && ENV['CDN_HOST'].present?
      blob_url = Rails.application.routes.url_helpers.rails_blob_url(
        attachment,
        host: ENV['APP_HOST']
      )
      blob_url.sub(ENV['APP_HOST'], ENV['CDN_HOST'])
    else
      url_for(attachment)
    end
  end
  
  def responsive_image_tag(attachment, **options)
    return "" unless attachment.attached?
    
    sizes = options.delete(:sizes) || [300, 600, 900, 1200]
    
    srcset = sizes.map { |w|
      variant = attachment.variant(resize_to_limit: [w, nil])
      "#{url_for(variant)} #{w}w"
    }.join(", ")
    
    image_tag(
      url_for(attachment.variant(resize_to_limit: [600, nil])),
      srcset: srcset,
      sizes: options.delete(:responsive_sizes) || "(max-width: 600px) 300px, 600px",
      loading: "lazy",
      **options
    )
  end
end
```

---

## Step 1137: Complete Upload Form with Preview

```erb
<%# app/views/articles/_form.html.erb %>
<%= form_with model: @article, multipart: true, data: { controller: "file-upload" } do |form| %>
  
  <%# Featured Image with Preview %>
  <div class="upload-section">
    <label class="upload-label">รูปหน้าปก</label>
    
    <div class="upload-area" 
         data-file-upload-target="dropzone"
         data-action="dragover->file-upload#dragover dragenter->file-upload#dragenter dragleave->file-upload#dragleave drop->file-upload#drop">
      
      <% if @article.featured_image.attached? %>
        <div class="current-image">
          <%= image_tag @article.featured_image.variant(resize_to_limit: [400, 300]),
                        class: "preview", 
                        data: { "file-upload-target": "preview" } %>
          <button type="button" class="remove-btn" data-action="click->file-upload#removeImage">×</button>
        </div>
      <% else %>
        <div class="upload-placeholder" data-file-upload-target="placeholder">
          <div class="upload-icon">📸</div>
          <p>ลากรูปมาวาง หรือคลิกเพื่อเลือกรูป</p>
          <p class="hint">รองรับ JPEG, PNG, WebP ขนาดไม่เกิน 10MB</p>
        </div>
        <img data-file-upload-target="preview" style="display:none; max-width: 400px;">
      <% end %>
    </div>
    
    <%= form.file_field :featured_image,
                        accept: "image/jpeg,image/png,image/webp",
                        class: "file-input",
                        data: { 
                          "file-upload-target": "input",
                          action: "change->file-upload#previewImage"
                        } %>
  </div>
  
  <%# Multiple Photos %>
  <div class="upload-section">
    <label>รูปภาพประกอบ (สูงสุด 10 รูป)</label>
    
    <div class="photos-grid" data-file-upload-target="photosGrid">
      <% @article.photos.each do |photo| %>
        <div class="photo-item" data-blob-id="<%= photo.id %>">
          <%= image_tag photo.variant(resize_to_fill: [200, 150]), class: "photo-thumb" %>
          <button type="button" 
                  class="remove-photo"
                  data-action="click->file-upload#removePhoto"
                  data-blob-id="<%= photo.id %>">×</button>
        </div>
      <% end %>
    </div>
    
    <%= form.file_field :photos,
                        multiple: true,
                        accept: "image/*",
                        class: "file-input",
                        data: { action: "change->file-upload#previewPhotos" } %>
  </div>
  
  <%# Upload Progress %>
  <div class="upload-progress" data-file-upload-target="progress" hidden>
    <div class="progress-bar">
      <div class="progress-fill" data-file-upload-target="progressFill" style="width: 0%"></div>
    </div>
    <span data-file-upload-target="progressText">Uploading...</span>
  </div>
  
  <%= form.submit "บันทึก", class: "btn btn-primary" %>
<% end %>
```

---

## Step 1138: Testing File Uploads

```ruby
# spec/requests/api/v1/uploads_spec.rb
require 'rails_helper'

RSpec.describe "Uploads API", type: :request do
  let(:user) { create(:user) }
  let(:headers) { auth_headers(user) }
  
  describe "POST /api/v1/uploads" do
    let(:image_file) do
      fixture_file_upload(
        Rails.root.join('spec', 'fixtures', 'files', 'test_image.jpg'),
        'image/jpeg'
      )
    end
    
    it "uploads a file successfully" do
      post "/api/v1/uploads",
           params: { file: image_file, context: 'avatar' },
           headers: headers
      
      expect(response).to have_http_status(:created)
      expect(json_response[:success]).to be true
      expect(json_response[:data][:url]).to be_present
    end
    
    it "rejects files that are too large" do
      large_file = create_large_file(11.megabytes)
      
      post "/api/v1/uploads",
           params: { file: large_file },
           headers: headers
      
      expect(response).to have_http_status(:unprocessable_entity)
    end
    
    it "rejects non-image files" do
      pdf_file = fixture_file_upload(
        Rails.root.join('spec', 'fixtures', 'files', 'document.pdf'),
        'application/pdf'
      )
      
      post "/api/v1/uploads",
           params: { file: pdf_file, context: 'avatar' },
           headers: headers
      
      expect(response).to have_http_status(:unprocessable_entity)
    end
  end
end
```

---

## Step 1139: Storage Analytics

```ruby
# app/models/concerns/trackable_attachment.rb
module TrackableAttachment
  extend ActiveSupport::Concern
  
  included do
    after_create_commit :track_attachment_created
    before_destroy :track_attachment_deleted
  end
  
  private
  
  def track_attachment_created
    StorageMetric.create!(
      event: 'attachment_created',
      user_id: record.respond_to?(:user_id) ? record.user_id : nil,
      blob_key: blob.key,
      content_type: blob.content_type,
      byte_size: blob.byte_size,
      record_type: record_type,
      record_id: record_id
    )
  end
  
  def track_attachment_deleted
    StorageMetric.create!(
      event: 'attachment_deleted',
      blob_key: blob.key,
      byte_size: blob.byte_size
    )
  end
end

ActiveStorage::Attachment.include(TrackableAttachment)
```

---

## Step 1140: S3 Presigned URLs

```ruby
# app/services/presigned_url_service.rb
class PresignedUrlService
  def self.for_upload(filename:, content_type:, max_size: 10.megabytes)
    s3 = Aws::S3::Client.new(region: ENV['AWS_REGION'])
    
    presigner = Aws::S3::Presigner.new(client: s3)
    
    key = "uploads/#{SecureRandom.uuid}/#{filename}"
    
    url = presigner.presigned_url(
      :put_object,
      bucket: ENV['AWS_BUCKET'],
      key: key,
      expires_in: 3600,  # 1 hour
      content_type: content_type,
      content_length_range: 1..max_size
    )
    
    { url: url, key: key }
  end
  
  def self.for_download(key:, filename: nil)
    s3 = Aws::S3::Client.new(region: ENV['AWS_REGION'])
    presigner = Aws::S3::Presigner.new(client: s3)
    
    presigner.presigned_url(
      :get_object,
      bucket: ENV['AWS_BUCKET'],
      key: key,
      expires_in: 1800,
      response_content_disposition: filename ? "attachment; filename=\"#{filename}\"" : nil
    )
  end
end
```

```ruby
# app/controllers/api/v1/presigned_urls_controller.rb
module Api
  module V1
    class PresignedUrlsController < BaseController
      def create
        result = PresignedUrlService.for_upload(
          filename: params[:filename],
          content_type: params[:content_type],
          max_size: 50.megabytes
        )
        
        render json: {
          success: true,
          data: {
            upload_url: result[:url],
            storage_key: result[:key]
          }
        }
      end
    end
  end
end
```
