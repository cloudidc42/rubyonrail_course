# Part 93: Headless CMS ด้วย Rails

## บทนำ

Headless CMS คือ backend-only content management system ที่ provide content ผ่าน API โดยไม่มี frontend ในตัว Rails เป็นตัวเลือกที่ยอดเยี่ยมสำหรับ Headless CMS เพราะมี ActiveStorage, Action Text, และ powerful API tools

## 1. Headless CMS Architecture

### 1.1 Headless vs Traditional CMS

```
Traditional CMS:                    Headless CMS:
┌─────────────────────┐            ┌─────────────────────┐
│  CMS (WordPress)    │            │  Rails API Backend  │
│  ┌───────────────┐  │            │  ┌───────────────┐  │
│  │   Content DB  │  │            │  │   Content DB  │  │
│  │   + Admin UI  │  │            │  │   + Admin UI  │  │
│  │   + Templates │  │            │  └───────┬───────┘  │
│  └───────┬───────┘  │            │          │ API      │
│          │          │            └──────────┼──────────┘
│          ▼ HTML     │                       │ JSON/GraphQL
│  ┌───────────────┐  │            ┌──────────┼──────────┐
│  │   Browser     │  │            │  ┌───────┴───────┐  │
└──┴───────────────┴──┘            │  │  Any Frontend │  │
                                    │  │  React/Vue/   │  │
                                    │  │  Mobile/etc   │  │
                                    │  └───────────────┘  │
                                    └─────────────────────┘
```

### 1.2 สร้าง Rails API Application

```bash
# สร้าง Rails API app
rails new my_cms --api --database=postgresql

# เพิ่ม dependencies
cd my_cms
```

```ruby
# Gemfile
source 'https://rubygems.org'
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby '3.2.0'

gem 'rails', '~> 7.1.0'
gem 'pg', '~> 1.1'
gem 'puma', '>= 5.0'
gem 'bootsnap', require: false
gem 'tzinfo-data', platforms: %i[mingw mswin x64_mingw jruby]

# API
gem 'jsonapi-serializer'  # หรือ active_model_serializers
gem 'pagy'                # Pagination
gem 'rack-cors'           # CORS support

# Content
gem 'action_text'         # Rich text
gem 'image_processing'    # Image manipulation
gem 'aws-sdk-s3'          # S3 storage (production)

# Search
gem 'pg_search'           # Full-text search
gem 'ransack'             # Advanced searching/filtering

# Auth
gem 'devise'
gem 'devise-jwt'          # JWT authentication

# Versioning
gem 'paper_trail'         # Version tracking

# Misc
gem 'kaminari'            # Pagination
gem 'friendly_id'         # Friendly URLs

group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'bullet'            # N+1 detection
end

group :development do
  gem 'annotate'
end
```

## 2. Content Models

### 2.1 Database Schema

```ruby
# db/migrate/20240101000001_create_content_types.rb
class CreateContentTypes < ActiveRecord::Migration[7.1]
  def change
    create_table :content_types do |t|
      t.string :name, null: false
      t.string :slug, null: false
      t.text :description
      t.jsonb :fields_config, default: {}
      t.boolean :enabled, default: true
      t.integer :position, default: 0
      
      t.timestamps
    end
    
    add_index :content_types, :slug, unique: true
    add_index :content_types, :enabled
  end
end

# db/migrate/20240101000002_create_entries.rb
class CreateEntries < ActiveRecord::Migration[7.1]
  def change
    create_table :entries do |t|
      t.references :content_type, null: false, foreign_key: true
      t.references :author, null: false, foreign_key: { to_table: :users }
      t.string :title, null: false
      t.string :slug, null: false
      t.string :status, default: 'draft'
      t.datetime :published_at
      t.datetime :unpublished_at
      t.integer :version, default: 1
      t.jsonb :fields, default: {}
      t.jsonb :metadata, default: {}
      
      t.timestamps
    end
    
    add_index :entries, [:content_type_id, :slug], unique: true
    add_index :entries, :status
    add_index :entries, :published_at
    add_index :entries, :fields, using: :gin  # GIN index สำหรับ JSONB queries
  end
end

# db/migrate/20240101000003_create_assets.rb
class CreateAssets < ActiveRecord::Migration[7.1]
  def change
    create_table :assets do |t|
      t.references :user, null: false, foreign_key: true
      t.string :filename, null: false
      t.string :content_type
      t.bigint :byte_size
      t.string :title
      t.text :description
      t.string :alt_text
      t.jsonb :metadata, default: {}
      t.jsonb :tags, default: []
      
      t.timestamps
    end
    
    add_index :assets, :content_type
    add_index :assets, :tags, using: :gin
  end
end

# db/migrate/20240101000004_create_collections.rb
class CreateCollections < ActiveRecord::Migration[7.1]
  def change
    create_table :collections do |t|
      t.string :name, null: false
      t.string :slug, null: false
      t.text :description
      t.references :parent, foreign_key: { to_table: :collections }
      t.integer :position, default: 0
      
      t.timestamps
    end
    
    add_index :collections, :slug, unique: true
    
    create_table :collection_entries do |t|
      t.references :collection, null: false, foreign_key: true
      t.references :entry, null: false, foreign_key: true
      t.integer :position, default: 0
      
      t.timestamps
    end
    
    add_index :collection_entries, [:collection_id, :entry_id], unique: true
  end
end
```

### 2.2 Content Type Model

```ruby
# app/models/content_type.rb
class ContentType < ApplicationRecord
  include FriendlyId
  
  friendly_id :name, use: :slugged
  
  has_many :entries, dependent: :destroy
  
  validates :name, presence: true, uniqueness: true
  validates :slug, presence: true, uniqueness: true
  
  # Field types ที่ support
  FIELD_TYPES = %w[
    text
    rich_text
    number
    boolean
    date
    datetime
    media
    reference
    references
    json
    select
    tags
  ].freeze
  
  # fields_config format:
  # {
  #   "title" => { "type" => "text", "required" => true, "max_length" => 200 },
  #   "body"  => { "type" => "rich_text", "required" => true },
  #   "image" => { "type" => "media", "allowed_types" => ["image/*"] },
  #   "tags"  => { "type" => "tags" }
  # }
  
  def field_names
    fields_config.keys
  end
  
  def required_fields
    fields_config.select { |_, config| config["required"] }.keys
  end
  
  def validate_entry_fields(fields)
    errors = {}
    
    required_fields.each do |field_name|
      unless fields[field_name].present?
        errors[field_name] = "is required"
      end
    end
    
    fields_config.each do |field_name, config|
      value = fields[field_name]
      next if value.nil?
      
      case config["type"]
      when "text"
        if config["max_length"] && value.to_s.length > config["max_length"]
          errors[field_name] = "exceeds maximum length of #{config['max_length']}"
        end
      when "number"
        unless value.is_a?(Numeric)
          errors[field_name] = "must be a number"
        end
      end
    end
    
    errors
  end
end
```

### 2.3 Entry Model ด้วย ActionText

```ruby
# app/models/entry.rb
class Entry < ApplicationRecord
  include PaperTrail::Model
  include FriendlyId
  include PgSearch::Model
  
  friendly_id :title, use: :slugged
  
  belongs_to :content_type
  belongs_to :author, class_name: 'User'
  has_many :collection_entries, dependent: :destroy
  has_many :collections, through: :collection_entries
  
  has_rich_text :rich_body
  has_one_attached :featured_image
  has_many_attached :attachments
  
  has_paper_trail(
    only: [:title, :slug, :fields, :status, :published_at],
    meta: {
      author_id: :author_id,
      entry_title: :title
    }
  )
  
  # Status enum
  STATUSES = %w[draft review published scheduled unpublished archived].freeze
  
  validates :title, presence: true
  validates :slug, presence: true, uniqueness: { scope: :content_type_id }
  validates :status, inclusion: { in: STATUSES }
  validate :fields_valid_for_content_type
  validate :published_at_required_when_published
  
  # Scopes
  scope :published, -> {
    where(status: 'published')
    .where('published_at <= ?', Time.current)
    .where('unpublished_at IS NULL OR unpublished_at > ?', Time.current)
  }
  
  scope :scheduled, -> {
    where(status: 'scheduled')
    .where('published_at > ?', Time.current)
  }
  
  scope :by_content_type, ->(type_slug) {
    joins(:content_type).where(content_types: { slug: type_slug })
  }
  
  scope :recent, -> { order(published_at: :desc, created_at: :desc) }
  
  # Full-text search
  pg_search_scope :search_content,
    against: {
      title: 'A',
      fields: 'B'
    },
    using: {
      tsearch: { prefix: true, dictionary: 'english' }
    }
  
  # Lifecycle methods
  def publish!
    return false unless can_publish?
    update!(status: 'published', published_at: Time.current)
  end
  
  def unpublish!
    update!(status: 'unpublished', unpublished_at: Time.current)
  end
  
  def schedule!(publish_at)
    return false if publish_at <= Time.current
    update!(status: 'scheduled', published_at: publish_at)
  end
  
  def archive!
    update!(status: 'archived')
  end
  
  def published?
    status == 'published' &&
      published_at.present? &&
      published_at <= Time.current &&
      (unpublished_at.nil? || unpublished_at > Time.current)
  end
  
  def can_publish?
    field_errors = content_type.validate_entry_fields(fields)
    field_errors.empty?
  end
  
  private
  
  def fields_valid_for_content_type
    return unless content_type
    
    field_errors = content_type.validate_entry_fields(fields)
    field_errors.each do |field, message|
      errors.add("fields.#{field}", message)
    end
  end
  
  def published_at_required_when_published
    if status == 'published' && published_at.blank?
      errors.add(:published_at, "is required when status is published")
    end
  end
end
```

## 3. Action Text Integration

### 3.1 Rich Text Content

```ruby
# config/initializers/action_text.rb
# ตั้งค่า Action Text สำหรับ API

# app/models/article.rb
class Article < ApplicationRecord
  include PaperTrail::Model
  
  belongs_to :author, class_name: 'User'
  has_many :article_tags, dependent: :destroy
  has_many :tags, through: :article_tags
  has_one_attached :cover_image
  
  has_rich_text :body
  has_rich_text :excerpt_rich  # Optional rich excerpt
  
  has_paper_trail only: [:title, :body, :status]
  
  validates :title, presence: true, length: { maximum: 200 }
  validates :body, presence: true
  
  def plain_text_body
    body.to_plain_text
  end
  
  def body_word_count
    plain_text_body.split.count
  end
  
  def reading_time_minutes
    (body_word_count / 200.0).ceil
  end
  
  def excerpt(length: 200)
    plain_text_body.truncate(length)
  end
  
  # Render body as HTML สำหรับ API
  def body_html
    body.body.to_html
  end
  
  # Extract images from body
  def body_images
    body.embeds.attachments.select { |a| a.image? }
  end
end
```

### 3.2 Rich Text ใน API Response

```ruby
# app/serializers/article_serializer.rb
class ArticleSerializer
  include JSONAPI::Serializer
  
  attributes :id, :title, :slug, :status, :published_at, :reading_time_minutes
  
  attribute :body_html do |article|
    article.body_html
  end
  
  attribute :excerpt do |article|
    article.excerpt(length: 250)
  end
  
  attribute :cover_image_url do |article|
    if article.cover_image.attached?
      Rails.application.routes.url_helpers.rails_blob_url(
        article.cover_image,
        only_path: true
      )
    end
  end
  
  attribute :cover_image_variants do |article|
    next {} unless article.cover_image.attached?
    
    {
      thumbnail: article.cover_image.variant(resize_to_fill: [300, 200]).processed.url,
      medium: article.cover_image.variant(resize_to_fill: [800, 600]).processed.url,
      large: article.cover_image.variant(resize_to_fill: [1200, 800]).processed.url
    }
  end
  
  has_one :author, serializer: UserSerializer
  has_many :tags, serializer: TagSerializer
end
```

## 4. ActiveStorage + CDN

### 4.1 Storage Configuration

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: <%= ENV['AWS_REGION'] || 'us-east-1' %>
  bucket: <%= ENV['AWS_BUCKET'] %>
  upload:
    server_side_encryption: "AES256"

cloudfront:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: <%= ENV['AWS_REGION'] %>
  bucket: <%= ENV['AWS_BUCKET'] %>
  http_open_timeout: 0
  http_read_timeout: 0
  retry_limit: 0
```

```ruby
# config/environments/production.rb
config.active_storage.service = :amazon

# CDN configuration
config.active_storage.url_options = {
  host: ENV['CDN_HOST'] || ENV['APP_HOST'],
  protocol: 'https'
}
```

### 4.2 Asset Model ด้วย ActiveStorage

```ruby
# app/models/asset.rb
class Asset < ApplicationRecord
  belongs_to :user
  
  has_one_attached :file
  
  validates :title, length: { maximum: 200 }
  validates :alt_text, length: { maximum: 500 }
  validate :acceptable_file
  
  ACCEPTED_TYPES = %w[
    image/jpeg image/png image/gif image/webp image/svg+xml
    video/mp4 video/webm
    application/pdf
    text/plain application/json
  ].freeze
  
  MAX_FILE_SIZE = 50.megabytes
  
  after_create :generate_variants
  
  scope :images, -> { joins(file_attachment: :blob).where("active_storage_blobs.content_type LIKE 'image/%'") }
  scope :videos, -> { joins(file_attachment: :blob).where("active_storage_blobs.content_type LIKE 'video/%'") }
  scope :documents, -> { joins(file_attachment: :blob).where("active_storage_blobs.content_type LIKE 'application/%'") }
  
  def image?
    file.blob&.content_type&.start_with?("image/")
  end
  
  def video?
    file.blob&.content_type&.start_with?("video/")
  end
  
  def file_url
    return nil unless file.attached?
    
    if Rails.env.production?
      # CloudFront URL
      file_cdn_url
    else
      Rails.application.routes.url_helpers.rails_blob_url(file, only_path: true)
    end
  end
  
  def file_cdn_url
    return nil unless file.attached?
    
    cdn_host = ENV['CDN_HOST']
    return file.url unless cdn_host
    
    blob_key = file.blob.key
    "https://#{cdn_host}/#{blob_key}"
  end
  
  def image_variants
    return {} unless image?
    
    {
      thumbnail: variant_url(:thumbnail),
      small: variant_url(:small),
      medium: variant_url(:medium),
      large: variant_url(:large),
      original: file_url
    }
  end
  
  def variant_url(size)
    dimensions = image_dimensions[size]
    return nil unless dimensions
    
    variant = file.variant(resize_to_fill: dimensions).processed
    Rails.application.routes.url_helpers.rails_representation_url(
      variant,
      only_path: true
    )
  rescue => e
    Rails.logger.error "Failed to generate variant #{size}: #{e.message}"
    nil
  end
  
  private
  
  def acceptable_file
    return unless file.attached?
    
    unless ACCEPTED_TYPES.include?(file.blob.content_type)
      errors.add(:file, "is not an acceptable file type")
    end
    
    if file.blob.byte_size > MAX_FILE_SIZE
      errors.add(:file, "is too large (max #{MAX_FILE_SIZE / 1.megabyte}MB)")
    end
  end
  
  def generate_variants
    return unless image?
    
    GenerateImageVariantsJob.perform_later(id)
  end
  
  def image_dimensions
    {
      thumbnail: [150, 150],
      small: [400, 300],
      medium: [800, 600],
      large: [1600, 1200]
    }
  end
end

# app/jobs/generate_image_variants_job.rb
class GenerateImageVariantsJob < ApplicationJob
  queue_as :default
  
  def perform(asset_id)
    asset = Asset.find(asset_id)
    return unless asset.image?
    
    [
      [150, 150],   # thumbnail
      [400, 300],   # small
      [800, 600],   # medium
      [1600, 1200]  # large
    ].each do |dimensions|
      asset.file.variant(resize_to_fill: dimensions).processed
    end
    
    Rails.logger.info "Generated variants for asset #{asset_id}"
  rescue => e
    Rails.logger.error "Failed to generate variants for asset #{asset_id}: #{e.message}"
    raise
  end
end
```

## 5. PaperTrail Versioning

### 5.1 Setup และ Configuration

```ruby
# config/initializers/paper_trail.rb
PaperTrail.config.enabled = true
PaperTrail.config.version_limit = 50  # เก็บ 50 versions ล่าสุด
PaperTrail.config.track_associations = true

# app/models/concerns/versionable.rb
module Versionable
  extend ActiveSupport::Concern
  
  included do
    has_paper_trail(
      meta: {
        author_id: :current_user_id,
        comment: :version_comment
      }
    )
    
    attr_accessor :version_comment
  end
  
  def current_user_id
    Current.user&.id
  end
  
  def version_history
    versions.map do |version|
      {
        id: version.id,
        event: version.event,
        created_at: version.created_at,
        author: version.whodunnit ? User.find_by(id: version.whodunnit) : nil,
        comment: version.comment,
        changes: version.changeset
      }
    end
  end
  
  def restore_to_version!(version_id)
    version = versions.find(version_id)
    raise "Version not found" unless version
    
    reified = version.reify
    
    transaction do
      update!(
        reified.attributes.except(
          'id', 'created_at', 'updated_at'
        )
      )
    end
  end
  
  def diff_with_version(version_id)
    version = versions.find(version_id)
    raise "Version not found" unless version
    
    reified = version.reify
    
    changes = {}
    reified.attributes.each do |key, old_value|
      current_value = send(key)
      if current_value != old_value
        changes[key] = { from: old_value, to: current_value }
      end
    end
    
    changes
  end
end
```

### 5.2 Version API Endpoints

```ruby
# app/controllers/api/v1/versions_controller.rb
module Api
  module V1
    class VersionsController < ApplicationController
      before_action :authenticate_user!
      before_action :set_entry
      
      def index
        @versions = @entry.versions
          .order(created_at: :desc)
          .page(params[:page])
          .per(20)
        
        render json: {
          data: @versions.map { |v| serialize_version(v) },
          meta: pagination_meta(@versions)
        }
      end
      
      def show
        @version = @entry.versions.find(params[:id])
        
        render json: {
          data: serialize_version(@version),
          reified: @version.reify&.attributes
        }
      end
      
      def restore
        @version = @entry.versions.find(params[:id])
        
        @entry.restore_to_version!(@version.id)
        
        render json: {
          message: "Restored to version #{@version.id}",
          data: EntrySerializer.new(@entry).serializable_hash
        }
      rescue => e
        render json: { error: e.message }, status: :unprocessable_entity
      end
      
      def diff
        @version = @entry.versions.find(params[:id])
        
        changes = @entry.diff_with_version(@version.id)
        
        render json: {
          version_id: @version.id,
          version_date: @version.created_at,
          current_date: @entry.updated_at,
          changes: changes
        }
      end
      
      private
      
      def set_entry
        @entry = Entry.find(params[:entry_id])
        authorize @entry, :manage?
      end
      
      def serialize_version(version)
        {
          id: version.id,
          event: version.event,
          created_at: version.created_at,
          author: version.whodunnit ? UserSerializer.new(User.find_by(id: version.whodunnit)) : nil,
          comment: version.comment,
          changes_count: version.changeset.keys.count
        }
      end
    end
  end
end
```

## 6. Draft/Publishing System

### 6.1 Publication Workflow

```ruby
# app/models/concerns/publishable.rb
module Publishable
  extend ActiveSupport::Concern
  
  included do
    before_save :set_version_on_publish
    after_commit :notify_webhooks, on: [:create, :update]
  end
  
  STATUSES = %w[draft review published scheduled unpublished archived].freeze
  
  def draft?    = status == 'draft'
  def in_review? = status == 'review'
  def published? = status == 'published' && published_at.present? && published_at <= Time.current
  def scheduled? = status == 'scheduled' && published_at.present? && published_at > Time.current
  def unpublished? = status == 'unpublished'
  def archived? = status == 'archived'
  
  def publish!(by: nil)
    return false unless can_publish?
    
    update!(
      status: 'published',
      published_at: Time.current,
      version: version + 1
    )
    
    Entry::PublishEvent.broadcast(self, by: by)
    true
  rescue ActiveRecord::RecordInvalid
    false
  end
  
  def unpublish!(by: nil)
    return false unless published? || scheduled?
    
    update!(
      status: 'unpublished',
      unpublished_at: Time.current
    )
    
    Entry::UnpublishEvent.broadcast(self, by: by)
    true
  end
  
  def schedule!(publish_at:, by: nil)
    return false if publish_at <= Time.current
    return false unless can_publish?
    
    update!(
      status: 'scheduled',
      published_at: publish_at
    )
    
    ScheduledPublishJob.set(wait_until: publish_at).perform_later(id)
    true
  end
  
  def submit_for_review!(by: nil)
    return false unless draft?
    
    update!(status: 'review')
    
    # Notify reviewers
    ReviewMailer.new_submission(self, submitter: by).deliver_later if by
    true
  end
  
  def approve!(by: nil)
    return false unless in_review?
    
    publish!(by: by)
  end
  
  def reject!(reason: nil, by: nil)
    return false unless in_review?
    
    update!(status: 'draft')
    
    # Notify author
    ReviewMailer.rejected(self, reason: reason, reviewer: by).deliver_later if by
    true
  end
  
  def archive!
    update!(status: 'archived')
  end
  
  def can_publish?
    # Override in models
    true
  end
  
  private
  
  def set_version_on_publish
    if status_changed? && status == 'published'
      self.version = (version || 0) + 1
    end
  end
  
  def notify_webhooks
    return unless saved_changes.key?('status')
    
    WebhookDeliveryJob.perform_later(
      event_type: "entry.#{status}",
      payload: {
        id: id,
        title: title,
        status: status,
        content_type: content_type.slug,
        updated_at: updated_at
      }
    )
  end
end
```

### 6.2 Scheduled Publishing

```ruby
# app/jobs/scheduled_publish_job.rb
class ScheduledPublishJob < ApplicationJob
  queue_as :scheduled
  
  def perform(entry_id)
    entry = Entry.find_by(id: entry_id)
    return unless entry
    return unless entry.scheduled?
    return if entry.published_at > Time.current
    
    entry.publish!
    
    Rails.logger.info "Published scheduled entry: #{entry.id} - #{entry.title}"
  rescue => e
    Rails.logger.error "Failed to publish scheduled entry #{entry_id}: #{e.message}"
    raise
  end
end

# app/jobs/auto_unpublish_job.rb
class AutoUnpublishJob < ApplicationJob
  queue_as :scheduled
  
  def perform
    # หา entries ที่ถึงเวลา unpublish แล้ว
    Entry
      .where(status: 'published')
      .where('unpublished_at <= ?', Time.current)
      .find_each do |entry|
        entry.unpublish!
        Rails.logger.info "Auto-unpublished entry: #{entry.id}"
      end
  end
end

# config/initializers/sidekiq_cron.rb (หรือ Solid Queue)
Sidekiq::Cron::Job.load_from_hash(
  'auto_unpublish' => {
    'cron' => '* * * * *',  # Every minute
    'class' => 'AutoUnpublishJob'
  }
)
```

## 7. API Controllers

### 7.1 Entries API

```ruby
# app/controllers/api/v1/entries_controller.rb
module Api
  module V1
    class EntriesController < ApplicationController
      include Pagy::Backend
      
      before_action :authenticate_user!, except: [:index, :show]
      before_action :set_content_type, only: [:index, :create]
      before_action :set_entry, only: [:show, :update, :destroy, :publish, :unpublish]
      
      def index
        @entries = Entry
          .by_content_type(@content_type.slug)
          .published
          .includes(:author, :tags, :collections)
        
        # Filtering
        @entries = apply_filters(@entries, params)
        
        # Search
        if params[:q].present?
          @entries = @entries.search_content(params[:q])
        end
        
        # Sorting
        @entries = apply_sorting(@entries, params[:sort], params[:order])
        
        @pagy, @entries = pagy(@entries)
        
        render json: {
          data: EntrySerializer.new(@entries).serializable_hash,
          meta: pagy_metadata(@pagy)
        }
      end
      
      def show
        # Public: only published
        # Authenticated: any status if admin or author
        unless @entry.published?
          unless current_user && (current_user.admin? || @entry.author == current_user)
            return render json: { error: 'Not found' }, status: :not_found
          end
        end
        
        render json: EntrySerializer.new(@entry, include: [:author, :tags]).serializable_hash
      end
      
      def create
        @entry = @content_type.entries.build(entry_params)
        @entry.author = current_user
        
        if @entry.save
          render json: EntrySerializer.new(@entry).serializable_hash,
                 status: :created
        else
          render json: { errors: @entry.errors.full_messages },
                 status: :unprocessable_entity
        end
      end
      
      def update
        authorize @entry
        
        if @entry.update(entry_params)
          render json: EntrySerializer.new(@entry).serializable_hash
        else
          render json: { errors: @entry.errors.full_messages },
                 status: :unprocessable_entity
        end
      end
      
      def destroy
        authorize @entry
        @entry.archive!
        head :no_content
      end
      
      def publish
        authorize @entry, :publish?
        
        if @entry.publish!(by: current_user)
          render json: { message: 'Entry published', data: EntrySerializer.new(@entry).serializable_hash }
        else
          render json: { errors: @entry.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def unpublish
        authorize @entry, :unpublish?
        
        if @entry.unpublish!(by: current_user)
          render json: { message: 'Entry unpublished' }
        else
          render json: { error: 'Cannot unpublish' }, status: :unprocessable_entity
        end
      end
      
      def schedule
        authorize @entry, :publish?
        
        publish_at = Time.parse(params[:publish_at])
        
        if @entry.schedule!(publish_at: publish_at, by: current_user)
          render json: {
            message: "Entry scheduled for #{publish_at.iso8601}",
            data: EntrySerializer.new(@entry).serializable_hash
          }
        else
          render json: { error: 'Cannot schedule entry' }, status: :unprocessable_entity
        end
      rescue ArgumentError
        render json: { error: 'Invalid publish_at date' }, status: :bad_request
      end
      
      private
      
      def set_content_type
        @content_type = ContentType.friendly.find(params[:content_type_id])
      rescue ActiveRecord::RecordNotFound
        render json: { error: 'Content type not found' }, status: :not_found
      end
      
      def set_entry
        @entry = Entry.friendly.find(params[:id])
      rescue ActiveRecord::RecordNotFound
        render json: { error: 'Entry not found' }, status: :not_found
      end
      
      def entry_params
        params.require(:entry).permit(
          :title, :slug, :status, :published_at, :unpublished_at,
          fields: {},
          metadata: {},
          tag_ids: [],
          collection_ids: []
        )
      end
      
      def apply_filters(entries, params)
        entries = entries.where(status: params[:status]) if params[:status].present?
        entries = entries.where(author_id: params[:author_id]) if params[:author_id].present?
        
        if params[:published_after].present?
          entries = entries.where('published_at >= ?', Time.parse(params[:published_after]))
        end
        
        if params[:published_before].present?
          entries = entries.where('published_at <= ?', Time.parse(params[:published_before]))
        end
        
        entries
      end
      
      def apply_sorting(entries, sort_field, sort_order)
        return entries.recent if sort_field.blank?
        
        allowed_fields = %w[title published_at created_at updated_at]
        return entries.recent unless allowed_fields.include?(sort_field)
        
        direction = sort_order&.downcase == 'asc' ? :asc : :desc
        entries.order(sort_field => direction)
      end
    end
  end
end
```

### 7.2 Assets API

```ruby
# app/controllers/api/v1/assets_controller.rb
module Api
  module V1
    class AssetsController < ApplicationController
      before_action :authenticate_user!
      before_action :set_asset, only: [:show, :update, :destroy]
      
      def index
        @assets = Asset.includes(file_attachment: :blob)
        
        @assets = @assets.images if params[:type] == 'image'
        @assets = @assets.videos if params[:type] == 'video'
        @assets = @assets.documents if params[:type] == 'document'
        
        if params[:q].present?
          @assets = @assets.where(
            "title ILIKE ? OR filename ILIKE ?",
            "%#{params[:q]}%", "%#{params[:q]}%"
          )
        end
        
        if params[:tags].present?
          tag_list = params[:tags].split(',').map(&:strip)
          @assets = @assets.where("tags @> ?", tag_list.to_json)
        end
        
        @pagy, @assets = pagy(@assets.order(created_at: :desc))
        
        render json: {
          data: @assets.map { |a| serialize_asset(a) },
          meta: pagy_metadata(@pagy)
        }
      end
      
      def show
        render json: serialize_asset(@asset)
      end
      
      def create
        @asset = Asset.new(asset_params.except(:file))
        @asset.user = current_user
        @asset.file.attach(params[:file]) if params[:file]
        
        if @asset.save
          render json: serialize_asset(@asset), status: :created
        else
          render json: { errors: @asset.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def update
        authorize @asset
        
        if @asset.update(asset_params.except(:file))
          render json: serialize_asset(@asset)
        else
          render json: { errors: @asset.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def destroy
        authorize @asset
        @asset.file.purge_later
        @asset.destroy!
        head :no_content
      end
      
      # Bulk upload
      def bulk_upload
        files = params[:files]
        return render json: { error: 'No files provided' }, status: :bad_request if files.blank?
        
        assets = []
        errors = []
        
        files.each_with_index do |file, index|
          asset = Asset.new(user: current_user, filename: file.original_filename)
          asset.file.attach(file)
          
          if asset.save
            assets << serialize_asset(asset)
          else
            errors << { index: index, errors: asset.errors.full_messages }
          end
        end
        
        render json: {
          uploaded: assets,
          errors: errors,
          total: files.count,
          success_count: assets.count,
          error_count: errors.count
        }
      end
      
      private
      
      def set_asset
        @asset = Asset.find(params[:id])
      end
      
      def asset_params
        params.require(:asset).permit(:title, :description, :alt_text, tags: [])
      end
      
      def serialize_asset(asset)
        {
          id: asset.id,
          filename: asset.filename,
          title: asset.title,
          description: asset.description,
          alt_text: asset.alt_text,
          content_type: asset.file.blob&.content_type,
          byte_size: asset.file.blob&.byte_size,
          url: asset.file_url,
          variants: asset.image? ? asset.image_variants : {},
          tags: asset.tags,
          created_at: asset.created_at,
          updated_at: asset.updated_at
        }
      end
    end
  end
end
```

## 8. Full-text Search

### 8.1 PostgreSQL Full-text Search

```ruby
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern
  
  included do
    include PgSearch::Model
    
    pg_search_scope :full_text_search,
      against: {
        title: 'A',
        fields: 'B'
      },
      associated_against: {
        tags: [:name]
      },
      using: {
        tsearch: {
          prefix: true,
          dictionary: 'english',
          tsvector_column: 'search_vector'
        },
        trigram: {
          word_similarity: true,
          threshold: 0.3
        }
      },
      ignoring: :accents
  end
  
  class_methods do
    def search(query, options = {})
      return published if query.blank?
      
      results = full_text_search(query)
      results = results.published unless options[:include_unpublished]
      results = results.by_content_type(options[:content_type]) if options[:content_type]
      results
    end
  end
end

# Migration สำหรับ search_vector
class AddSearchVectorToEntries < ActiveRecord::Migration[7.1]
  def up
    add_column :entries, :search_vector, :tsvector
    
    # Create index
    add_index :entries, :search_vector, using: :gin
    
    # Trigger สำหรับ update search_vector อัตโนมัติ
    execute <<-SQL
      CREATE OR REPLACE FUNCTION entries_search_vector_update() RETURNS TRIGGER AS $$
      BEGIN
        NEW.search_vector := 
          setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
          setweight(to_tsvector('english', coalesce(NEW.fields::text, '')), 'B');
        RETURN NEW;
      END;
      $$ LANGUAGE plpgsql;
      
      CREATE TRIGGER entries_search_vector_update
      BEFORE INSERT OR UPDATE ON entries
      FOR EACH ROW EXECUTE PROCEDURE entries_search_vector_update();
    SQL
    
    # Update existing records
    execute "UPDATE entries SET search_vector = to_tsvector('english', coalesce(title, '') || ' ' || coalesce(fields::text, ''))"
  end
  
  def down
    remove_column :entries, :search_vector
    execute "DROP TRIGGER IF EXISTS entries_search_vector_update ON entries"
    execute "DROP FUNCTION IF EXISTS entries_search_vector_update"
  end
end
```

## 9. Webhooks System

### 9.1 Webhook Model และ Delivery

```ruby
# app/models/webhook.rb
class Webhook < ApplicationRecord
  belongs_to :user
  has_many :webhook_deliveries, dependent: :destroy
  
  validates :url, presence: true, format: { with: URI::regexp(%w[http https]) }
  validates :events, presence: true
  validates :secret, presence: true
  
  EVENTS = %w[
    entry.created entry.updated entry.published entry.unpublished entry.deleted
    asset.created asset.deleted
  ].freeze
  
  before_create :generate_secret
  
  def subscribed_to?(event)
    events.include?('*') || events.include?(event)
  end
  
  def sign_payload(payload)
    digest = OpenSSL::Digest.new('sha256')
    OpenSSL::HMAC.hexdigest(digest, secret, payload)
  end
  
  private
  
  def generate_secret
    self.secret ||= SecureRandom.hex(32)
  end
end

# app/jobs/webhook_delivery_job.rb
class WebhookDeliveryJob < ApplicationJob
  queue_as :webhooks
  
  retry_on StandardError, wait: :polynomially_longer, attempts: 5
  
  def perform(event_type:, payload:)
    webhooks = Webhook.where(enabled: true).select { |w| w.subscribed_to?(event_type) }
    
    webhooks.each do |webhook|
      deliver_to_webhook(webhook, event_type: event_type, payload: payload)
    end
  end
  
  private
  
  def deliver_to_webhook(webhook, event_type:, payload:)
    body = payload.merge(event: event_type, timestamp: Time.current.iso8601).to_json
    signature = webhook.sign_payload(body)
    
    response = Net::HTTP.start(webhook_uri(webhook).host, webhook_uri(webhook).port, use_ssl: true) do |http|
      request = Net::HTTP::Post.new(webhook_uri(webhook))
      request['Content-Type'] = 'application/json'
      request['X-Webhook-Signature'] = "sha256=#{signature}"
      request['X-Webhook-Event'] = event_type
      request.body = body
      
      http.request(request)
    end
    
    delivery = webhook.webhook_deliveries.create!(
      event_type: event_type,
      payload: payload,
      response_status: response.code.to_i,
      response_body: response.body,
      delivered_at: Time.current
    )
    
    unless response.is_a?(Net::HTTPSuccess)
      raise "Webhook delivery failed with status #{response.code}"
    end
    
    delivery
  rescue => e
    webhook.webhook_deliveries.create!(
      event_type: event_type,
      payload: payload,
      error_message: e.message,
      delivered_at: Time.current
    ) rescue nil
    
    raise
  end
  
  def webhook_uri(webhook)
    @webhook_uri ||= URI(webhook.url)
  end
end
```

## 10. Authentication

### 10.1 JWT Authentication ด้วย Devise

```ruby
# Gemfile
gem 'devise'
gem 'devise-jwt'

# app/models/user.rb
class User < ApplicationRecord
  include Devise::JWT::RevocationStrategies::JTIMatcher
  
  devise :database_authenticatable, :registerable,
         :recoverable, :validatable,
         :jwt_authenticatable, jwt_revocation_strategy: self
  
  ROLES = %w[super_admin admin editor contributor viewer].freeze
  
  validates :role, inclusion: { in: ROLES }
  
  def admin?     = role.in?(%w[super_admin admin])
  def editor?    = role.in?(%w[super_admin admin editor])
  def contributor? = role.in?(%w[super_admin admin editor contributor])
  
  def can_publish?
    role.in?(%w[super_admin admin editor])
  end
  
  def can_manage_users?
    role.in?(%w[super_admin admin])
  end
  
  def can_manage_settings?
    role == 'super_admin'
  end
end

# config/initializers/devise.rb
Devise.setup do |config|
  config.jwt do |jwt|
    jwt.secret = ENV['DEVISE_JWT_SECRET_KEY']
    jwt.dispatch_requests = [
      ['POST', %r{^/api/v1/users/sign_in$}]
    ]
    jwt.revocation_requests = [
      ['DELETE', %r{^/api/v1/users/sign_out$}]
    ]
    jwt.expiration_time = 1.day.to_i
  end
end
```

## 11. GraphQL API (Optional Layer)

### 11.1 GraphQL Setup

```ruby
# Gemfile
gem 'graphql'

# Install
bundle exec rails generate graphql:install

# app/graphql/types/entry_type.rb
module Types
  class EntryType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :slug, String, null: false
    field :status, String, null: false
    field :published_at, GraphQL::Types::ISO8601DateTime, null: true
    field :fields, GraphQL::Types::JSON, null: true
    field :reading_time, Integer, null: true
    field :excerpt, String, null: true do
      argument :length, Integer, required: false, default_value: 200
    end
    
    field :author, Types::UserType, null: false
    field :content_type, Types::ContentTypeType, null: false
    field :tags, [Types::TagType], null: false
    field :collections, [Types::CollectionType], null: false
    
    def excerpt(length:)
      object.plain_text_body.truncate(length)
    end
    
    def reading_time
      object.body_word_count / 200
    end
  end
end

# app/graphql/queries/entries_query.rb
module Queries
  class EntriesQuery < Queries::BaseQuery
    type [Types::EntryType], null: false
    
    argument :content_type, String, required: false
    argument :status, String, required: false
    argument :search, String, required: false
    argument :page, Integer, required: false, default_value: 1
    argument :per_page, Integer, required: false, default_value: 20
    
    def resolve(content_type: nil, status: 'published', search: nil, page: 1, per_page: 20)
      entries = Entry.published.includes(:author, :tags, :content_type)
      entries = entries.by_content_type(content_type) if content_type
      entries = entries.search_content(search) if search.present?
      
      entries.page(page).per(per_page)
    end
  end
end
```

---

## แบบฝึกหัดบทที่ 93

### แบบฝึกหัดที่ 1: Content Type Builder
**โจทย์:** สร้าง dynamic content type builder ที่ผู้ใช้กำหนด fields ได้

**เฉลย:**
```ruby
# app/services/content_type_builder.rb
class ContentTypeBuilder
  VALID_FIELD_TYPES = %w[text rich_text number boolean date datetime media reference tags select].freeze
  
  class InvalidFieldTypeError < StandardError; end
  class DuplicateFieldError < StandardError; end
  
  attr_reader :content_type, :errors
  
  def initialize(name:, description: nil)
    @name = name
    @description = description
    @fields = {}
    @errors = []
  end
  
  def add_field(name, type:, required: false, **options)
    raise InvalidFieldTypeError, "Unknown field type: #{type}" unless VALID_FIELD_TYPES.include?(type.to_s)
    raise DuplicateFieldError, "Field '#{name}' already exists" if @fields.key?(name.to_s)
    
    @fields[name.to_s] = {
      "type" => type.to_s,
      "required" => required
    }.merge(options.stringify_keys)
    
    self
  end
  
  def text_field(name, max_length: nil, min_length: nil, required: false)
    options = {}
    options["max_length"] = max_length if max_length
    options["min_length"] = min_length if min_length
    add_field(name, type: :text, required: required, **options)
  end
  
  def rich_text_field(name, required: false)
    add_field(name, type: :rich_text, required: required)
  end
  
  def number_field(name, min: nil, max: nil, required: false)
    options = {}
    options["min"] = min if min
    options["max"] = max if max
    add_field(name, type: :number, required: required, **options)
  end
  
  def media_field(name, allowed_types: nil, required: false)
    options = {}
    options["allowed_types"] = allowed_types if allowed_types
    add_field(name, type: :media, required: required, **options)
  end
  
  def select_field(name, options_list:, multiple: false, required: false)
    add_field(name, type: :select, required: required,
              options: options_list, multiple: multiple)
  end
  
  def build!
    validate!
    
    ContentType.create!(
      name: @name,
      description: @description,
      fields_config: @fields
    )
  rescue ActiveRecord::RecordInvalid => e
    @errors << e.message
    nil
  end
  
  def valid?
    @errors.clear
    validate!
    @errors.empty?
  end
  
  private
  
  def validate!
    @errors << "Name is required" if @name.blank?
    @errors << "At least one field is required" if @fields.empty?
    
    # Validate field names
    @fields.keys.each do |name|
      unless name.match?(/\A[a-z][a-z0-9_]*\z/)
        @errors << "Invalid field name '#{name}': must start with lowercase letter and contain only letters, numbers, and underscores"
      end
    end
  end
end

# ใช้งาน
builder = ContentTypeBuilder.new(name: "Blog Post", description: "Standard blog post content type")

builder
  .text_field(:title, max_length: 200, required: true)
  .rich_text_field(:body, required: true)
  .text_field(:excerpt, max_length: 500)
  .media_field(:cover_image, allowed_types: ["image/*"])
  .select_field(:category, options_list: ["Tech", "Design", "Business"], required: true)
  .add_field(:tags, type: :tags)

content_type = builder.build!
```

### แบบฝึกหัดที่ 2: Full-text Search API
**โจทย์:** Implement advanced search ด้วย filters และ facets

**เฉลย:**
```ruby
# app/services/content_search_service.rb
class ContentSearchService
  DEFAULT_PER_PAGE = 20
  MAX_PER_PAGE = 100
  
  def initialize(query:, filters: {}, options: {})
    @query = query
    @filters = filters
    @options = options
  end
  
  def search
    entries = Entry.published.includes(:content_type, :author, :tags)
    
    # Full-text search
    if @query.present?
      entries = entries.search_content(@query)
    else
      entries = entries.order(published_at: :desc)
    end
    
    # Apply filters
    entries = apply_content_type_filter(entries)
    entries = apply_date_filter(entries)
    entries = apply_tag_filter(entries)
    entries = apply_author_filter(entries)
    
    # Calculate facets
    facets = calculate_facets(entries)
    
    # Pagination
    per_page = [@options[:per_page].to_i || DEFAULT_PER_PAGE, MAX_PER_PAGE].min
    per_page = DEFAULT_PER_PAGE if per_page <= 0
    
    page = @options[:page].to_i || 1
    page = 1 if page <= 0
    
    paginated = entries.page(page).per(per_page)
    
    {
      results: paginated.to_a,
      total_count: entries.count,
      facets: facets,
      page: page,
      per_page: per_page,
      total_pages: (entries.count.to_f / per_page).ceil,
      query: @query,
      filters: @filters
    }
  end
  
  private
  
  def apply_content_type_filter(entries)
    return entries unless @filters[:content_type].present?
    
    entries.by_content_type(@filters[:content_type])
  end
  
  def apply_date_filter(entries)
    if @filters[:published_after].present?
      entries = entries.where('published_at >= ?', parse_date(@filters[:published_after]))
    end
    
    if @filters[:published_before].present?
      entries = entries.where('published_at <= ?', parse_date(@filters[:published_before]))
    end
    
    entries
  end
  
  def apply_tag_filter(entries)
    return entries unless @filters[:tags].present?
    
    tag_names = Array(@filters[:tags])
    entries.joins(:tags).where(tags: { name: tag_names })
  end
  
  def apply_author_filter(entries)
    return entries unless @filters[:author_id].present?
    
    entries.where(author_id: @filters[:author_id])
  end
  
  def calculate_facets(entries)
    {
      content_types: entries
        .joins(:content_type)
        .group('content_types.name')
        .count,
      
      tags: entries
        .joins(:tags)
        .group('tags.name')
        .count
        .first(20),
      
      authors: entries
        .joins(:author)
        .group('users.name')
        .count
        .first(10),
      
      dates: entries
        .group("date_trunc('month', published_at)")
        .count
        .transform_keys { |k| k&.strftime('%Y-%m') }
    }
  end
  
  def parse_date(date_str)
    Time.parse(date_str)
  rescue ArgumentError
    nil
  end
end

# app/controllers/api/v1/search_controller.rb
module Api
  module V1
    class SearchController < ApplicationController
      def index
        result = ContentSearchService.new(
          query: params[:q],
          filters: {
            content_type: params[:content_type],
            published_after: params[:published_after],
            published_before: params[:published_before],
            tags: params[:tags]&.split(','),
            author_id: params[:author_id]
          },
          options: {
            page: params[:page],
            per_page: params[:per_page]
          }
        ).search
        
        render json: {
          data: result[:results].map { |e| EntrySerializer.new(e).serializable_hash },
          meta: {
            total_count: result[:total_count],
            total_pages: result[:total_pages],
            page: result[:page],
            per_page: result[:per_page],
            query: result[:query],
            facets: result[:facets]
          }
        }
      end
    end
  end
end
```

### แบบฝึกหัดที่ 3: Image Processing Pipeline
**โจทย์:** สร้าง image processing pipeline ด้วย ActiveStorage

**เฉลย:**
```ruby
# app/services/image_processing_service.rb
class ImageProcessingService
  PRESETS = {
    thumbnail:  { resize_to_fill: [150, 150], format: :webp, quality: 80 },
    small:      { resize_to_fill: [400, 300], format: :webp, quality: 85 },
    medium:     { resize_to_fill: [800, 600], format: :webp, quality: 85 },
    large:      { resize_to_fill: [1200, 900], format: :webp, quality: 90 },
    hero:       { resize_to_fill: [1920, 1080], format: :webp, quality: 90 },
    square:     { resize_to_fill: [600, 600], format: :webp, quality: 85 },
    og_image:   { resize_to_fill: [1200, 630], format: :jpeg, quality: 90 }
  }.freeze
  
  def initialize(asset)
    @asset = asset
    raise ArgumentError, "Asset must have an image file" unless asset.image?
  end
  
  def process_all_presets
    results = {}
    
    PRESETS.each do |preset_name, preset_options|
      results[preset_name] = process_preset(preset_name, preset_options)
    end
    
    results
  end
  
  def process_preset(preset_name, options = nil)
    options ||= PRESETS[preset_name]
    raise ArgumentError, "Unknown preset: #{preset_name}" unless options
    
    variant = @asset.file.variant(
      **build_variant_options(options)
    ).processed
    
    {
      url: Rails.application.routes.url_helpers.rails_representation_url(variant, only_path: true),
      format: options[:format] || :jpeg,
      width: options.dig(:resize_to_fill, 0),
      height: options.dig(:resize_to_fill, 1)
    }
  rescue => e
    Rails.logger.error "Failed to process variant #{preset_name} for asset #{@asset.id}: #{e.message}"
    nil
  end
  
  def generate_blur_placeholder
    # Small base64-encoded blur
    tiny = @asset.file.variant(
      resize_to_fill: [20, 20],
      format: :webp,
      quality: 10
    ).processed
    
    # Convert to base64 for inline use
    blob_key = tiny.blob.key
    data = ActiveStorage::Blob.service.download(blob_key)
    "data:image/webp;base64,#{Base64.strict_encode64(data)}"
  rescue => e
    Rails.logger.error "Failed to generate blur placeholder: #{e.message}"
    nil
  end
  
  def extract_dominant_color
    # Using MiniMagick to extract dominant color
    require 'mini_magick'
    
    @asset.file.blob.open do |file|
      image = MiniMagick::Image.new(file.path)
      
      # Resize to 1x1 to get average color
      image.resize "1x1!"
      pixel = image.get_pixels.first.first
      
      "#%02x%02x%02x" % pixel
    end
  rescue => e
    Rails.logger.error "Failed to extract dominant color: #{e.message}"
    nil
  end
  
  private
  
  def build_variant_options(options)
    variant_opts = {}
    
    variant_opts[:resize_to_fill] = options[:resize_to_fill] if options[:resize_to_fill]
    variant_opts[:format] = options[:format] if options[:format]
    
    if options[:quality]
      variant_opts[:saver] = { quality: options[:quality] }
    end
    
    variant_opts
  end
end

# Job สำหรับ process ใน background
class ProcessImageVariantsJob < ApplicationJob
  queue_as :image_processing
  
  def perform(asset_id)
    asset = Asset.find(asset_id)
    return unless asset.image?
    
    service = ImageProcessingService.new(asset)
    variants = service.process_all_presets
    
    # Store variant info ใน metadata
    asset.update!(
      metadata: asset.metadata.merge(
        'variants' => variants,
        'processed_at' => Time.current.iso8601
      )
    )
  end
end
```

### แบบฝึกหัดที่ 4: Content Localization
**โจทย์:** เพิ่ม multi-language support สำหรับ entries

**เฉลย:**
```ruby
# db/migrate/add_locale_to_entries.rb
class AddLocaleToEntries < ActiveRecord::Migration[7.1]
  def change
    add_column :entries, :locale, :string, default: 'en', null: false
    add_reference :entries, :original_entry, foreign_key: { to_table: :entries }
    
    add_index :entries, [:content_type_id, :slug, :locale], unique: true
    remove_index :entries, [:content_type_id, :slug]
  end
end

# app/models/entry.rb (additions)
class Entry < ApplicationRecord
  belongs_to :original_entry, class_name: 'Entry', optional: true
  has_many :translations, class_name: 'Entry', foreign_key: :original_entry_id
  
  SUPPORTED_LOCALES = %w[en th ja zh ko fr de es].freeze
  
  validates :locale, inclusion: { in: SUPPORTED_LOCALES }
  
  scope :for_locale, ->(locale) { where(locale: locale) }
  scope :original, -> { where(original_entry_id: nil) }
  scope :translations, -> { where.not(original_entry_id: nil) }
  
  def translation_for(locale)
    return self if self.locale == locale.to_s
    
    if original_entry
      original_entry.translations.find_by(locale: locale) ||
        original_entry.translations.find_by(locale: 'en') ||
        original_entry
    else
      translations.find_by(locale: locale) || self
    end
  end
  
  def create_translation!(locale:, fields:, title:, author:)
    target_locale = locale.to_s
    
    raise ArgumentError, "Unsupported locale: #{target_locale}" unless SUPPORTED_LOCALES.include?(target_locale)
    raise ArgumentError, "Translation for #{target_locale} already exists" if translations.find_by(locale: target_locale)
    
    original = original_entry || self
    
    original.translations.create!(
      content_type: content_type,
      author: author,
      title: title,
      slug: "#{slug}-#{target_locale}",
      fields: fields,
      locale: target_locale,
      status: 'draft'
    )
  end
  
  def available_locales
    if original_entry
      [original_entry.locale] + original_entry.translations.pluck(:locale)
    else
      [locale] + translations.pluck(:locale)
    end
  end
  
  def missing_locales
    SUPPORTED_LOCALES - available_locales
  end
end

# app/controllers/api/v1/entries_controller.rb (with locale support)
def index
  locale = params[:locale] || I18n.default_locale.to_s
  
  @entries = Entry
    .by_content_type(@content_type.slug)
    .for_locale(locale)
    .original
    .published
    .includes(:author, :tags)
  
  render json: {
    data: EntrySerializer.new(@entries).serializable_hash,
    locale: locale
  }
end
```

### แบบฝึกหัดที่ 5: Webhook System
**โจทย์:** สร้าง complete webhook management system

**เฉลย:**
```ruby
# app/controllers/api/v1/webhooks_controller.rb
module Api
  module V1
    class WebhooksController < ApplicationController
      before_action :authenticate_user!
      before_action :require_admin!
      before_action :set_webhook, only: [:show, :update, :destroy, :deliveries, :test]
      
      def index
        @webhooks = current_user.webhooks.order(created_at: :desc)
        render json: @webhooks.map { |w| serialize_webhook(w) }
      end
      
      def show
        render json: serialize_webhook(@webhook)
      end
      
      def create
        @webhook = current_user.webhooks.build(webhook_params)
        
        if @webhook.save
          render json: serialize_webhook(@webhook), status: :created
        else
          render json: { errors: @webhook.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def update
        if @webhook.update(webhook_params)
          render json: serialize_webhook(@webhook)
        else
          render json: { errors: @webhook.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def destroy
        @webhook.destroy!
        head :no_content
      end
      
      def deliveries
        @deliveries = @webhook.webhook_deliveries
          .order(created_at: :desc)
          .page(params[:page])
          .per(20)
        
        render json: {
          data: @deliveries.map { |d| serialize_delivery(d) },
          meta: { total: @webhook.webhook_deliveries.count }
        }
      end
      
      def test
        # ส่ง test payload
        test_payload = {
          event: 'webhook.test',
          timestamp: Time.current.iso8601,
          message: 'This is a test webhook delivery'
        }
        
        WebhookDeliveryJob.perform_now(
          event_type: 'webhook.test',
          payload: test_payload,
          webhook_ids: [@webhook.id]
        )
        
        render json: { message: 'Test webhook sent' }
      end
      
      private
      
      def set_webhook
        @webhook = current_user.webhooks.find(params[:id])
      rescue ActiveRecord::RecordNotFound
        render json: { error: 'Webhook not found' }, status: :not_found
      end
      
      def webhook_params
        params.require(:webhook).permit(:url, :description, :enabled, events: [])
      end
      
      def serialize_webhook(webhook)
        {
          id: webhook.id,
          url: webhook.url,
          description: webhook.description,
          events: webhook.events,
          enabled: webhook.enabled,
          secret: webhook.secret,
          deliveries_count: webhook.webhook_deliveries.count,
          last_delivery_at: webhook.webhook_deliveries.maximum(:created_at),
          created_at: webhook.created_at
        }
      end
      
      def serialize_delivery(delivery)
        {
          id: delivery.id,
          event_type: delivery.event_type,
          response_status: delivery.response_status,
          success: delivery.response_status.to_i.between?(200, 299),
          error_message: delivery.error_message,
          delivered_at: delivery.delivered_at,
          duration_ms: delivery.duration_ms
        }
      end
    end
  end
end
```

### แบบฝึกหัดที่ 6: Content Collections API
**โจทย์:** สร้าง Collections API สำหรับ grouping content

**เฉลย:**
```ruby
# app/controllers/api/v1/collections_controller.rb
module Api
  module V1
    class CollectionsController < ApplicationController
      before_action :authenticate_user!, except: [:index, :show]
      before_action :set_collection, only: [:show, :update, :destroy, :entries, :add_entry, :remove_entry]
      
      def index
        @collections = Collection
          .includes(:parent, :children)
          .roots
          .order(position: :asc, name: :asc)
        
        render json: @collections.map { |c| serialize_collection(c, include_children: true) }
      end
      
      def show
        render json: serialize_collection(@collection, include_entries: true)
      end
      
      def create
        @collection = Collection.new(collection_params)
        
        if @collection.save
          render json: serialize_collection(@collection), status: :created
        else
          render json: { errors: @collection.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def update
        authorize @collection
        
        if @collection.update(collection_params)
          render json: serialize_collection(@collection)
        else
          render json: { errors: @collection.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def destroy
        authorize @collection
        @collection.destroy!
        head :no_content
      end
      
      def entries
        @entries = @collection.entries.published.includes(:author, :tags)
        @pagy, @entries = pagy(@entries.order(collection_entries: { position: :asc }))
        
        render json: {
          data: EntrySerializer.new(@entries).serializable_hash,
          meta: pagy_metadata(@pagy)
        }
      end
      
      def add_entry
        authorize @collection, :manage?
        
        entry = Entry.find(params[:entry_id])
        
        @collection.collection_entries.create!(
          entry: entry,
          position: @collection.entries.count
        )
        
        render json: serialize_collection(@collection, include_entries: true)
      rescue ActiveRecord::RecordNotFound
        render json: { error: 'Entry not found' }, status: :not_found
      rescue ActiveRecord::RecordInvalid => e
        render json: { error: e.message }, status: :unprocessable_entity
      end
      
      def remove_entry
        authorize @collection, :manage?
        
        collection_entry = @collection.collection_entries.find_by(entry_id: params[:entry_id])
        
        if collection_entry
          collection_entry.destroy!
          head :no_content
        else
          render json: { error: 'Entry not in collection' }, status: :not_found
        end
      end
      
      def reorder
        authorize @collection, :manage?
        
        entry_ids = params[:entry_ids]
        
        transaction do
          entry_ids.each_with_index do |entry_id, position|
            @collection.collection_entries
              .find_by(entry_id: entry_id)
              &.update!(position: position)
          end
        end
        
        render json: serialize_collection(@collection, include_entries: true)
      end
      
      private
      
      def set_collection
        @collection = Collection.friendly.find(params[:id])
      rescue ActiveRecord::RecordNotFound
        render json: { error: 'Collection not found' }, status: :not_found
      end
      
      def collection_params
        params.require(:collection).permit(:name, :slug, :description, :parent_id, :position)
      end
      
      def serialize_collection(collection, include_children: false, include_entries: false)
        data = {
          id: collection.id,
          name: collection.name,
          slug: collection.slug,
          description: collection.description,
          position: collection.position,
          parent_id: collection.parent_id,
          entries_count: collection.entries.count,
          created_at: collection.created_at
        }
        
        if include_children && collection.children.any?
          data[:children] = collection.children.map { |c| serialize_collection(c) }
        end
        
        if include_entries
          data[:entries] = collection.entries
            .published
            .order(collection_entries: { position: :asc })
            .map { |e| { id: e.id, title: e.title, slug: e.slug } }
        end
        
        data
      end
    end
  end
end
```

### แบบฝึกหัดที่ 7: Media Library Controller
**โจทย์:** สร้าง media library ด้วย folder organization

**เฉลย:**
```ruby
# app/models/media_folder.rb
class MediaFolder < ApplicationRecord
  belongs_to :parent, class_name: 'MediaFolder', optional: true
  has_many :children, class_name: 'MediaFolder', foreign_key: :parent_id, dependent: :destroy
  has_many :assets, dependent: :nullify
  
  validates :name, presence: true, uniqueness: { scope: :parent_id }
  
  def full_path
    if parent
      "#{parent.full_path}/#{name}"
    else
      "/#{name}"
    end
  end
  
  def descendants
    children.includes(:children).flat_map do |child|
      [child] + child.descendants
    end
  end
  
  def all_assets
    Asset.where(media_folder_id: [id] + descendants.map(&:id))
  end
end

# app/controllers/api/v1/media_controller.rb
module Api
  module V1
    class MediaController < ApplicationController
      before_action :authenticate_user!
      
      def folders
        @folders = MediaFolder.roots.includes(:children).order(:name)
        
        render json: @folders.map { |f| serialize_folder(f) }
      end
      
      def folder_contents
        @folder = params[:folder_id] ? MediaFolder.find(params[:folder_id]) : nil
        
        @assets = if @folder
          @folder.assets.includes(file_attachment: :blob)
        else
          Asset.where(media_folder_id: nil).includes(file_attachment: :blob)
        end
        
        @assets = @assets.where(user: current_user) unless current_user.admin?
        
        @pagy, @assets = pagy(@assets.order(created_at: :desc))
        
        @subfolders = @folder ? @folder.children.order(:name) : MediaFolder.roots.order(:name)
        
        render json: {
          current_folder: @folder ? serialize_folder(@folder) : nil,
          folders: @subfolders.map { |f| serialize_folder(f) },
          assets: @assets.map { |a| AssetSerializer.new(a).serializable_hash },
          meta: pagy_metadata(@pagy)
        }
      end
      
      def create_folder
        @folder = MediaFolder.new(
          name: params[:name],
          parent_id: params[:parent_id]
        )
        
        if @folder.save
          render json: serialize_folder(@folder), status: :created
        else
          render json: { errors: @folder.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def move_asset
        @asset = Asset.find(params[:asset_id])
        authorize @asset
        
        @asset.update!(media_folder_id: params[:folder_id])
        
        render json: AssetSerializer.new(@asset).serializable_hash
      end
      
      private
      
      def serialize_folder(folder)
        {
          id: folder.id,
          name: folder.name,
          path: folder.full_path,
          parent_id: folder.parent_id,
          children_count: folder.children.count,
          assets_count: folder.assets.count
        }
      end
    end
  end
end
```

### แบบฝึกหัดที่ 8: Scheduled Content Publishing
**โจทย์:** สร้าง complete scheduled publishing system

**เฉลย:**
```ruby
# app/models/publish_schedule.rb
class PublishSchedule < ApplicationRecord
  belongs_to :entry
  belongs_to :created_by, class_name: 'User'
  
  validates :action, inclusion: { in: %w[publish unpublish] }
  validates :scheduled_at, presence: true
  validate :scheduled_at_in_future, on: :create
  
  scope :pending, -> { where(status: 'pending').where('scheduled_at > ?', Time.current) }
  scope :due, -> { where(status: 'pending').where('scheduled_at <= ?', Time.current) }
  scope :completed, -> { where(status: 'completed') }
  scope :failed, -> { where(status: 'failed') }
  
  def execute!
    case action
    when 'publish'
      entry.publish!
    when 'unpublish'
      entry.unpublish!
    end
    
    update!(status: 'completed', executed_at: Time.current)
  rescue => e
    update!(status: 'failed', error_message: e.message)
    raise
  end
  
  private
  
  def scheduled_at_in_future
    if scheduled_at.present? && scheduled_at <= Time.current
      errors.add(:scheduled_at, "must be in the future")
    end
  end
end

# app/jobs/process_publish_schedules_job.rb
class ProcessPublishSchedulesJob < ApplicationJob
  queue_as :scheduled
  
  def perform
    PublishSchedule.due.each do |schedule|
      schedule.execute!
    rescue => e
      Rails.logger.error "Failed to execute schedule #{schedule.id}: #{e.message}"
    end
  end
end

# app/controllers/api/v1/publish_schedules_controller.rb
module Api
  module V1
    class PublishSchedulesController < ApplicationController
      before_action :authenticate_user!
      before_action :set_entry
      
      def index
        @schedules = @entry.publish_schedules.order(scheduled_at: :asc)
        
        render json: @schedules.map { |s| serialize_schedule(s) }
      end
      
      def create
        authorize @entry, :publish?
        
        @schedule = @entry.publish_schedules.build(
          action: params[:action_type],
          scheduled_at: params[:scheduled_at],
          created_by: current_user
        )
        
        if @schedule.save
          render json: serialize_schedule(@schedule), status: :created
        else
          render json: { errors: @schedule.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def destroy
        authorize @entry, :publish?
        
        @schedule = @entry.publish_schedules.find(params[:id])
        
        if @schedule.pending?
          @schedule.update!(status: 'cancelled')
          head :no_content
        else
          render json: { error: 'Cannot cancel completed or failed schedule' }, status: :unprocessable_entity
        end
      end
      
      private
      
      def set_entry
        @entry = Entry.find(params[:entry_id])
      end
      
      def serialize_schedule(schedule)
        {
          id: schedule.id,
          action: schedule.action,
          scheduled_at: schedule.scheduled_at,
          status: schedule.status,
          executed_at: schedule.executed_at,
          error_message: schedule.error_message,
          created_by: { id: schedule.created_by.id, name: schedule.created_by.name }
        }
      end
    end
  end
end
```

### แบบฝึกหัดที่ 9: Rate Limiting และ API Keys
**โจทย์:** Implement API key management และ rate limiting

**เฉลย:**
```ruby
# app/models/api_key.rb
class ApiKey < ApplicationRecord
  belongs_to :user
  
  has_secure_token :token
  
  validates :name, presence: true
  validates :token, presence: true, uniqueness: true
  
  scope :active, -> { where(revoked: false).where('expires_at IS NULL OR expires_at > ?', Time.current) }
  
  RATE_LIMITS = {
    'free' => { requests_per_hour: 100, requests_per_day: 1000 },
    'basic' => { requests_per_hour: 1000, requests_per_day: 10000 },
    'premium' => { requests_per_hour: 10000, requests_per_day: 100000 }
  }.freeze
  
  def revoke!
    update!(revoked: true, revoked_at: Time.current)
  end
  
  def active?
    !revoked && (expires_at.nil? || expires_at > Time.current)
  end
  
  def rate_limit
    RATE_LIMITS[user.subscription_tier] || RATE_LIMITS['free']
  end
  
  def track_usage!
    redis = Redis.current
    hour_key = "api_key:#{id}:hour:#{Time.current.strftime('%Y%m%d%H')}"
    day_key = "api_key:#{id}:day:#{Time.current.strftime('%Y%m%d')}"
    
    redis.multi do |multi|
      multi.incr(hour_key)
      multi.expire(hour_key, 3600)
      multi.incr(day_key)
      multi.expire(day_key, 86400)
    end
  end
  
  def rate_limit_exceeded?
    redis = Redis.current
    limit = rate_limit
    
    hour_key = "api_key:#{id}:hour:#{Time.current.strftime('%Y%m%d%H')}"
    day_key = "api_key:#{id}:day:#{Time.current.strftime('%Y%m%d')}"
    
    hour_count = redis.get(hour_key).to_i
    day_count = redis.get(day_key).to_i
    
    hour_count >= limit[:requests_per_hour] || day_count >= limit[:requests_per_day]
  end
end

# app/controllers/concerns/api_key_authenticatable.rb
module ApiKeyAuthenticatable
  extend ActiveSupport::Concern
  
  included do
    before_action :authenticate_api_key!
    before_action :check_rate_limit!
  end
  
  private
  
  def authenticate_api_key!
    token = extract_api_key
    
    unless token
      return render json: { error: 'API key required' }, status: :unauthorized
    end
    
    @api_key = ApiKey.active.find_by(token: token)
    
    unless @api_key
      return render json: { error: 'Invalid or expired API key' }, status: :unauthorized
    end
    
    Current.user = @api_key.user
    Current.api_key = @api_key
  end
  
  def check_rate_limit!
    return unless @api_key
    
    if @api_key.rate_limit_exceeded?
      limit = @api_key.rate_limit
      render json: {
        error: 'Rate limit exceeded',
        limit: limit[:requests_per_hour],
        window: '1 hour'
      }, status: :too_many_requests
    else
      @api_key.track_usage!
    end
  end
  
  def extract_api_key
    # Support Bearer token and X-API-Key header
    auth_header = request.headers['Authorization']
    
    if auth_header&.start_with?('Bearer ')
      auth_header.sub('Bearer ', '')
    else
      request.headers['X-API-Key']
    end
  end
end
```

### แบบฝึกหัดที่ 10: Content Preview System
**โจทย์:** สร้าง preview system สำหรับ draft content

**เฉลย:**
```ruby
# app/models/preview_token.rb
class PreviewToken < ApplicationRecord
  belongs_to :entry
  belongs_to :created_by, class_name: 'User'
  
  before_create :generate_token
  
  validates :token, presence: true, uniqueness: true
  
  scope :valid, -> { where('expires_at > ?', Time.current) }
  
  def self.create_for(entry, created_by:, expires_in: 24.hours)
    create!(
      entry: entry,
      created_by: created_by,
      expires_at: Time.current + expires_in
    )
  end
  
  def expired?
    expires_at <= Time.current
  end
  
  def preview_url(base_url)
    "#{base_url}/preview/#{token}"
  end
  
  private
  
  def generate_token
    self.token = SecureRandom.urlsafe_base64(32)
  end
end

# app/controllers/api/v1/preview_controller.rb
module Api
  module V1
    class PreviewController < ApplicationController
      # No authentication required - token provides access
      
      def create
        authenticate_user!
        
        entry = Entry.find(params[:entry_id])
        authorize entry, :preview?
        
        preview = PreviewToken.create_for(
          entry,
          created_by: current_user,
          expires_in: (params[:expires_in]&.to_i || 24).hours
        )
        
        render json: {
          token: preview.token,
          expires_at: preview.expires_at,
          preview_url: preview.preview_url(request.base_url)
        }, status: :created
      end
      
      def show
        preview = PreviewToken.valid.find_by(token: params[:token])
        
        unless preview
          return render json: { error: 'Invalid or expired preview token' }, status: :unauthorized
        end
        
        entry = preview.entry
        
        render json: EntrySerializer.new(entry, include: [:author, :tags]).serializable_hash.merge(
          meta: {
            preview: true,
            preview_expires_at: preview.expires_at,
            current_status: entry.status
          }
        )
      end
    end
  end
end
```

### แบบฝึกหัดที่ 11: Content Tagging System
**โจทย์:** สร้าง advanced tagging system ด้วย hierarchical tags

**เฉลย:**
```ruby
# app/models/tag.rb
class Tag < ApplicationRecord
  belongs_to :parent, class_name: 'Tag', optional: true
  has_many :children, class_name: 'Tag', foreign_key: :parent_id, dependent: :destroy
  has_many :entry_tags
  has_many :entries, through: :entry_tags
  
  validates :name, presence: true, uniqueness: { case_sensitive: false }
  validates :slug, presence: true, uniqueness: true
  
  before_validation :generate_slug
  
  scope :roots, -> { where(parent_id: nil) }
  scope :popular, -> { joins(:entry_tags).group(:id).order('COUNT(entry_tags.id) DESC') }
  scope :by_name, ->(name) { where('name ILIKE ?', "%#{name}%") }
  
  def full_path
    parent ? "#{parent.full_path}/#{slug}" : slug
  end
  
  def descendants
    result = []
    children.each do |child|
      result << child
      result.concat(child.descendants)
    end
    result
  end
  
  def all_entry_ids
    Entry.joins(:tags).where(tags: { id: [id] + descendants.map(&:id) }).pluck(:id)
  end
  
  def usage_count
    entry_tags.count
  end
  
  private
  
  def generate_slug
    self.slug ||= name&.parameterize
  end
end

# app/controllers/api/v1/tags_controller.rb
module Api
  module V1
    class TagsController < ApplicationController
      before_action :authenticate_user!, except: [:index, :show]
      
      def index
        @tags = Tag.includes(:children, :parent)
        
        @tags = @tags.roots if params[:root_only]
        @tags = @tags.by_name(params[:q]) if params[:q]
        @tags = @tags.popular.limit(params[:limit] || 50) if params[:popular]
        
        render json: @tags.map { |t| serialize_tag(t) }
      end
      
      def show
        @tag = Tag.find(params[:id])
        
        @entries = @tag.entries.published.page(params[:page]).per(20)
        
        render json: {
          tag: serialize_tag(@tag, include_hierarchy: true),
          entries: {
            data: EntrySerializer.new(@entries).serializable_hash,
            total: @tag.entries.published.count
          }
        }
      end
      
      def create
        authorize :tag, :create?
        
        @tag = Tag.new(tag_params)
        
        if @tag.save
          render json: serialize_tag(@tag), status: :created
        else
          render json: { errors: @tag.errors.full_messages }, status: :unprocessable_entity
        end
      end
      
      def merge
        authorize :tag, :manage?
        
        source = Tag.find(params[:source_id])
        target = Tag.find(params[:target_id])
        
        ActiveRecord::Base.transaction do
          EntryTag.where(tag_id: source.id).update_all(tag_id: target.id)
          source.children.update_all(parent_id: target.id)
          source.destroy!
        end
        
        render json: { message: "Merged #{source.name} into #{target.name}" }
      end
      
      private
      
      def tag_params
        params.require(:tag).permit(:name, :slug, :description, :parent_id)
      end
      
      def serialize_tag(tag, include_hierarchy: false)
        data = {
          id: tag.id,
          name: tag.name,
          slug: tag.slug,
          description: tag.description,
          parent_id: tag.parent_id,
          children_count: tag.children.count,
          usage_count: tag.usage_count,
          path: tag.full_path
        }
        
        if include_hierarchy
          data[:breadcrumbs] = build_breadcrumbs(tag)
          data[:children] = tag.children.map { |c| serialize_tag(c) }
        end
        
        data
      end
      
      def build_breadcrumbs(tag)
        crumbs = []
        current = tag
        
        while current
          crumbs.unshift({ id: current.id, name: current.name, slug: current.slug })
          current = current.parent
        end
        
        crumbs
      end
    end
  end
end
```

### แบบฝึกหัดที่ 12: Content Export/Import
**โจทย์:** สร้าง content export/import system

**เฉลย:**
```ruby
# app/services/content_exporter.rb
class ContentExporter
  def initialize(content_type_ids: nil, include_assets: false)
    @content_type_ids = content_type_ids
    @include_assets = include_assets
  end
  
  def export
    {
      version: "1.0",
      exported_at: Time.current.iso8601,
      content_types: export_content_types,
      entries: export_entries,
      tags: export_tags
    }
  end
  
  def export_to_file(path)
    data = export
    File.write(path, JSON.pretty_generate(data))
    path
  end
  
  private
  
  def export_content_types
    content_types.map do |ct|
      {
        id: ct.id,
        name: ct.name,
        slug: ct.slug,
        description: ct.description,
        fields_config: ct.fields_config
      }
    end
  end
  
  def export_entries
    entries = Entry.published.includes(:content_type, :author, :tags)
    entries = entries.where(content_type_id: @content_type_ids) if @content_type_ids
    
    entries.map do |entry|
      {
        id: entry.id,
        content_type_slug: entry.content_type.slug,
        title: entry.title,
        slug: entry.slug,
        status: entry.status,
        published_at: entry.published_at&.iso8601,
        fields: entry.fields,
        tags: entry.tags.pluck(:name),
        author_email: entry.author.email
      }
    end
  end
  
  def export_tags
    Tag.all.map { |t| { id: t.id, name: t.name, slug: t.slug, parent_slug: t.parent&.slug } }
  end
  
  def content_types
    if @content_type_ids
      ContentType.where(id: @content_type_ids)
    else
      ContentType.all
    end
  end
end

# app/services/content_importer.rb
class ContentImporter
  class ImportError < StandardError; end
  
  def initialize(data)
    @data = data
    @errors = []
    @imported_counts = Hash.new(0)
  end
  
  def import!
    validate_format!
    
    ActiveRecord::Base.transaction do
      import_tags
      import_content_types
      import_entries
    end
    
    { success: true, imported: @imported_counts, errors: @errors }
  rescue ImportError => e
    { success: false, error: e.message, errors: @errors }
  end
  
  private
  
  def validate_format!
    raise ImportError, "Invalid export format" unless @data.is_a?(Hash)
    raise ImportError, "Missing version field" unless @data['version']
    raise ImportError, "Unsupported version" unless @data['version'] == '1.0'
  end
  
  def import_tags
    Array(@data['tags']).each do |tag_data|
      tag = Tag.find_or_initialize_by(slug: tag_data['slug'])
      tag.name = tag_data['name']
      tag.save!
      @imported_counts[:tags] += 1
    rescue => e
      @errors << "Failed to import tag #{tag_data['name']}: #{e.message}"
    end
    
    # Set parent relationships
    Array(@data['tags']).select { |t| t['parent_slug'] }.each do |tag_data|
      tag = Tag.find_by(slug: tag_data['slug'])
      parent = Tag.find_by(slug: tag_data['parent_slug'])
      
      tag&.update!(parent: parent) if parent
    end
  end
  
  def import_content_types
    Array(@data['content_types']).each do |ct_data|
      ct = ContentType.find_or_initialize_by(slug: ct_data['slug'])
      ct.name = ct_data['name']
      ct.description = ct_data['description']
      ct.fields_config = ct_data['fields_config']
      ct.save!
      @imported_counts[:content_types] += 1
    rescue => e
      @errors << "Failed to import content type #{ct_data['name']}: #{e.message}"
    end
  end
  
  def import_entries
    Array(@data['entries']).each do |entry_data|
      content_type = ContentType.find_by(slug: entry_data['content_type_slug'])
      next @errors << "Content type not found: #{entry_data['content_type_slug']}" unless content_type
      
      author = User.find_by(email: entry_data['author_email']) || User.find_by(role: 'super_admin')
      next @errors << "Author not found: #{entry_data['author_email']}" unless author
      
      entry = Entry.find_or_initialize_by(
        content_type: content_type,
        slug: entry_data['slug']
      )
      
      entry.assign_attributes(
        title: entry_data['title'],
        fields: entry_data['fields'],
        status: 'draft',
        author: author
      )
      
      entry.save!
      
      # Import tags
      tag_names = Array(entry_data['tags'])
      tags = tag_names.map { |name| Tag.find_or_create_by!(name: name) { |t| t.slug = name.parameterize } }
      entry.tags = tags
      
      @imported_counts[:entries] += 1
    rescue => e
      @errors << "Failed to import entry #{entry_data['title']}: #{e.message}"
    end
  end
end
```

### แบบฝึกหัดที่ 13: API Rate Limiting ด้วย Rack::Attack
**โจทย์:** Configure Rack::Attack สำหรับ CMS API

**เฉลย:**
```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Cache store
  Rack::Attack.cache.store = ActiveSupport::Cache::RedisCacheStore.new(
    url: ENV['REDIS_URL']
  )
  
  # Throttle API requests by IP
  throttle('api/ip', limit: 300, period: 5.minutes) do |req|
    req.ip if req.path.start_with?('/api/')
  end
  
  # Throttle API requests by API key
  throttle('api/key', limit: 1000, period: 1.hour) do |req|
    if req.path.start_with?('/api/')
      token = req.get_header('HTTP_X_API_KEY') ||
              req.get_header('HTTP_AUTHORIZATION')&.sub('Bearer ', '')
      token if token.present?
    end
  end
  
  # Throttle login attempts
  throttle('login/ip', limit: 10, period: 1.hour) do |req|
    req.ip if req.path == '/api/v1/users/sign_in' && req.post?
  end
  
  # Throttle signup
  throttle('signup/ip', limit: 5, period: 1.hour) do |req|
    req.ip if req.path == '/api/v1/users' && req.post?
  end
  
  # Block suspicious requests
  blocklist('block/bad_user_agents') do |req|
    bad_agents = ['sqlmap', 'nikto', 'masscan', 'zgrab']
    user_agent = req.user_agent.to_s.downcase
    bad_agents.any? { |agent| user_agent.include?(agent) }
  end
  
  # Return proper response for throttled requests
  self.throttled_responder = lambda do |env|
    match_data = env['rack.attack.match_data']
    retry_after = (match_data[:period] - match_data[:epoch_time] % match_data[:period]).to_s
    
    [
      429,
      {
        'Content-Type' => 'application/json',
        'Retry-After' => retry_after,
        'X-RateLimit-Limit' => match_data[:limit].to_s,
        'X-RateLimit-Remaining' => '0',
        'X-RateLimit-Reset' => retry_after
      },
      [JSON.generate({ error: 'Rate limit exceeded', retry_after: retry_after.to_i })]
    ]
  end
end
```

### แบบฝึกหัดที่ 14: Audit Log System
**โจทย์:** สร้าง audit log สำหรับ CMS actions

**เฉลย:**
```ruby
# app/models/audit_log.rb
class AuditLog < ApplicationRecord
  belongs_to :user, optional: true
  belongs_to :resource, polymorphic: true, optional: true
  
  ACTIONS = %w[
    created updated deleted published unpublished
    uploaded downloaded previewed accessed
    login logout failed_login
    api_access
  ].freeze
  
  validates :action, inclusion: { in: ACTIONS }
  
  scope :recent, -> { order(created_at: :desc) }
  scope :for_resource, ->(resource) { where(resource: resource) }
  scope :by_user, ->(user) { where(user: user) }
  scope :by_action, ->(action) { where(action: action) }
  
  def self.log(action:, user: nil, resource: nil, metadata: {})
    create!(
      action: action,
      user: user,
      resource: resource,
      metadata: metadata.merge(
        ip_address: Current.ip_address,
        user_agent: Current.user_agent,
        request_id: Current.request_id
      )
    )
  end
end

# app/models/concerns/audited.rb
module Audited
  extend ActiveSupport::Concern
  
  included do
    after_create  { log_audit_event('created') }
    after_update  { log_audit_event('updated') if significant_change? }
    after_destroy { log_audit_event('deleted') }
  end
  
  private
  
  def log_audit_event(action)
    AuditLog.log(
      action: action,
      user: Current.user,
      resource: self,
      metadata: {
        changes: saved_changes.reject { |k, _| k.in?(%w[updated_at created_at]) }
      }
    )
  rescue => e
    Rails.logger.error "Failed to log audit event: #{e.message}"
  end
  
  def significant_change?
    ignored_columns = %w[updated_at]
    (saved_changes.keys - ignored_columns).any?
  end
end

# app/controllers/api/v1/audit_logs_controller.rb
module Api
  module V1
    class AuditLogsController < ApplicationController
      before_action :authenticate_user!
      before_action :require_admin!
      
      def index
        @logs = AuditLog
          .includes(:user)
          .recent
        
        @logs = @logs.by_user(params[:user_id]) if params[:user_id]
        @logs = @logs.by_action(params[:action]) if params[:action]
        
        if params[:resource_type] && params[:resource_id]
          @logs = @logs.where(resource_type: params[:resource_type], resource_id: params[:resource_id])
        end
        
        @pagy, @logs = pagy(@logs)
        
        render json: {
          data: @logs.map { |log|
            {
              id: log.id,
              action: log.action,
              user: log.user ? { id: log.user.id, name: log.user.name } : nil,
              resource_type: log.resource_type,
              resource_id: log.resource_id,
              metadata: log.metadata,
              created_at: log.created_at
            }
          },
          meta: pagy_metadata(@pagy)
        }
      end
    end
  end
end
```

### แบบฝึกหัดที่ 15: Complete CMS Setup
**โจทย์:** Setup complete Headless CMS ด้วย Rails

**เฉลย:**
```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Authentication
  devise_for :users,
    path: 'api/v1/auth',
    controllers: {
      sessions: 'api/v1/auth/sessions',
      registrations: 'api/v1/auth/registrations',
      passwords: 'api/v1/auth/passwords'
    }
  
  namespace :api do
    namespace :v1 do
      # Content
      resources :content_types do
        resources :entries do
          member do
            post :publish
            post :unpublish
            post :schedule
            post :submit_for_review
            post :approve
            post :reject
          end
          
          resources :versions, only: [:index, :show] do
            member do
              post :restore
              get :diff
            end
          end
          
          resources :preview_tokens, only: [:create]
        end
      end
      
      # Standalone entries search
      get 'entries/search', to: 'search#index'
      
      # Assets / Media Library
      resources :assets, only: [:index, :show, :create, :update, :destroy] do
        collection do
          post :bulk_upload
        end
      end
      
      resources :media_folders do
        member do
          get :contents
        end
        collection do
          post :create_folder
        end
      end
      
      # Collections
      resources :collections do
        resources :entries, only: [:index] do
          collection do
            post :add
            delete ':entry_id', to: 'collection_entries#destroy', as: :remove
            post :reorder
          end
        end
      end
      
      # Tags
      resources :tags do
        collection do
          post :merge
        end
      end
      
      # Webhooks
      resources :webhooks do
        member do
          get :deliveries
          post :test
        end
      end
      
      # Audit Logs
      resources :audit_logs, only: [:index]
      
      # Preview
      get 'preview/:token', to: 'preview#show', as: :content_preview
      
      # API Keys
      resources :api_keys do
        member do
          post :revoke
        end
      end
      
      # Admin
      namespace :admin do
        resources :users, only: [:index, :show, :update, :destroy]
        get :stats, to: 'stats#show'
      end
    end
  end
  
  # Direct upload (ActiveStorage)
  direct :rails_direct_uploads do
    "#{ENV['APP_URL']}/rails/active_storage/direct_uploads"
  end
end

# db/seeds.rb - Initial setup
puts "Creating admin user..."
admin = User.create_with(
  password: ENV['ADMIN_PASSWORD'] || 'changeme123',
  role: 'super_admin'
).find_or_create_by!(email: ENV['ADMIN_EMAIL'] || 'admin@cms.example.com')
puts "Admin user: #{admin.email}"

puts "Creating Blog Post content type..."
blog = ContentType.create_with(
  description: 'Standard blog post',
  fields_config: {
    'body' => { 'type' => 'rich_text', 'required' => true },
    'excerpt' => { 'type' => 'text', 'max_length' => 500 },
    'cover_image' => { 'type' => 'media', 'allowed_types' => ['image/*'] },
    'featured' => { 'type' => 'boolean' }
  }
).find_or_create_by!(name: 'Blog Post')
puts "Content type: #{blog.name}"

puts "Creating default tags..."
['Technology', 'Design', 'Business', 'Tutorial', 'News'].each do |tag_name|
  Tag.find_or_create_by!(name: tag_name) { |t| t.slug = tag_name.parameterize }
end

puts "CMS setup complete!"
```

---

## สรุปบทที่ 93

ในบทนี้เราได้เรียนรู้:

1. **Headless CMS Architecture** - การออกแบบ API-first CMS
2. **Content Models** - JSONB fields, dynamic schemas
3. **Action Text** - Rich text content ใน API context
4. **ActiveStorage + CDN** - File management และ image variants
5. **PaperTrail** - Version tracking และ content history
6. **Draft/Publishing Workflow** - Scheduled publishing และ review flows
7. **Full-text Search** - PostgreSQL GIN index และ pg_search
8. **Webhooks** - Event-driven notifications
9. **GraphQL** - Alternative API layer
10. **Production Concerns** - Rate limiting, auth, audit logs
