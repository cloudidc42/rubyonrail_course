# Part 50: Search ใน Rails

## ขั้นตอนที่ 1101-1120: การค้นหาข้อมูลด้วยวิธีต่างๆ

---

## ขั้นตอนที่ 1101: Basic Search ด้วย LIKE

```ruby
# Simple LIKE search
class PostsController < ApplicationController
  def index
    @posts = Post.all
    
    if params[:query].present?
      query = "%#{params[:query]}%"
      @posts = @posts.where("title LIKE ? OR content LIKE ?", query, query)
    end
    
    @posts = @posts.order(created_at: :desc).page(params[:page])
  end
end
```

```erb
<%# app/views/posts/index.html.erb %>
<%= form_with url: posts_path, method: :get do |f| %>
  <div class="search-box">
    <%= f.text_field :query, value: params[:query], placeholder: "ค้นหา..." %>
    <%= f.submit "ค้นหา" %>
  </div>
<% end %>

<% @posts.each do |post| %>
  <%= render post %>
<% end %>
```

## ขั้นตอนที่ 1102: ILIKE สำหรับ PostgreSQL

```ruby
# PostgreSQL ILIKE - case insensitive
class Post < ApplicationRecord
  scope :search_by_title, ->(query) {
    where("title ILIKE ?", "%#{query}%") if query.present?
  }
  
  scope :search_by_content, ->(query) {
    where("content ILIKE ?", "%#{query}%") if query.present?
  }
  
  scope :full_text_search, ->(query) {
    return all if query.blank?
    
    where(
      "title ILIKE :q OR content ILIKE :q OR tags_string ILIKE :q",
      q: "%#{query}%"
    )
  }
end
```

```ruby
# PostgreSQL Full-Text Search (built-in)
class Post < ApplicationRecord
  scope :pg_search, ->(query) {
    where(
      "to_tsvector('english', title || ' ' || content) @@ plainto_tsquery(?)",
      query
    )
  }
  
  # สร้าง index
  # add_index :posts, "to_tsvector('english', title || ' ' || content)", 
  #           using: :gin, name: 'index_posts_on_fts'
end
```

## ขั้นตอนที่ 1103: pg_search Gem

```ruby
# Gemfile
gem 'pg_search'

# bundle install
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include PgSearch::Model
  
  # Multisearch (ค้นหาข้ามหลาย model)
  multisearchable against: [:title, :content, :tags_string]
  
  # Single model search
  pg_search_scope :search_by_title,
    against: :title,
    using: {
      tsearch: {
        prefix: true,
        highlight: {
          StartSel: '<mark>',
          StopSel: '</mark>'
        }
      }
    }
  
  pg_search_scope :full_search,
    against: {
      title: 'A',      # น้ำหนักสูงสุด
      content: 'B',
      summary: 'C',
      tags_string: 'D' # น้ำหนักต่ำสุด
    },
    using: {
      tsearch: {
        prefix: true,
        dictionary: "english"
      },
      trigram: {
        threshold: 0.3
      }
    }
end
```

```ruby
# ใน controller
def index
  if params[:query].present?
    @posts = Post.full_search(params[:query])
  else
    @posts = Post.published.order(created_at: :desc)
  end
  
  @posts = @posts.page(params[:page]).per(20)
end
```

```ruby
# Global search ข้ามหลาย model
class SearchController < ApplicationController
  def index
    @query = params[:query]
    @results = PgSearch.multisearch(@query) if @query.present?
  end
end

# ต้อง run: rails pg_search:create_multisearch_tables
# app/models/post.rb - multisearchable against: [:title, :content]
# app/models/user.rb - multisearchable against: [:name, :bio]
# app/models/tag.rb  - multisearchable against: :name
```

## ขั้นตอนที่ 1104: Ransack Gem

```ruby
# Gemfile
gem 'ransack'

# bundle install
```

```ruby
# app/controllers/posts_controller.rb
def index
  @q = Post.ransack(params[:q])
  @posts = @q.result(distinct: true)
             .includes(:user, :tags)
             .page(params[:page])
             .per(20)
end
```

```erb
<%# app/views/posts/index.html.erb %>
<%= search_form_for @q do |f| %>
  <div class="search-form">
    <%# Text search %>
    <div>
      <%= f.label :title_cont, "หัวข้อประกอบด้วย" %>
      <%= f.search_field :title_cont %>
    </div>
    
    <%# Select dropdown %>
    <div>
      <%= f.label :category_id_eq, "หมวดหมู่" %>
      <%= f.collection_select :category_id_eq, Category.all, :id, :name, include_blank: "ทั้งหมด" %>
    </div>
    
    <%# Date range %>
    <div>
      <%= f.label :created_at_gteq, "ตั้งแต่วันที่" %>
      <%= f.date_field :created_at_gteq %>
    </div>
    <div>
      <%= f.label :created_at_lteq, "ถึงวันที่" %>
      <%= f.date_field :created_at_lteq %>
    </div>
    
    <%# Checkbox %>
    <div>
      <%= f.label :published_eq, "เผยแพร่แล้ว" %>
      <%= f.check_box :published_eq %>
    </div>
    
    <%# Sort %>
    <%= f.sort_link :created_at, "วันที่สร้าง" %>
    <%= f.sort_link :title, "หัวข้อ" %>
    
    <%= f.submit "ค้นหา" %>
  </div>
<% end %>
```

```ruby
# Ransack predicates
# _eq           - equal
# _not_eq       - not equal
# _lt           - less than
# _lteq         - less than or equal
# _gt           - greater than
# _gteq         - greater than or equal
# _cont         - contains (LIKE %value%)
# _not_cont     - not contains
# _start        - starts with
# _end          - ends with
# _in           - in array
# _not_in       - not in array
# _null         - is null
# _not_null     - is not null
# _true         - is true
# _false        - is false

# Combine multiple predicates
# _and          - AND condition (default)
# _or           - OR condition
# title_or_content_cont - OR across fields
```

```ruby
# Advanced Ransack
# app/models/post.rb
class Post < ApplicationRecord
  # Allow searching associations
  ransacker :author_name do
    Arel.sql("users.name")
  end
  
  # Custom ransack scope
  ransacker :created_year do
    Arel.sql("EXTRACT(YEAR FROM created_at)")
  end
end

# ค้นหาด้วย custom ransacker
@q = Post.joins(:user).ransack(params[:q])
# params[:q] = { author_name_cont: "สมชาย", created_year_eq: 2024 }
```

## ขั้นตอนที่ 1105: Elasticsearch กับ Searchkick

```ruby
# Gemfile
gem 'searchkick'
gem 'elasticsearch-model'  # optional

# bundle install
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  searchkick text_start: [:title],  # prefix matching
             word_start: [:content],
             suggest: [:title],
             index_prefix: "myapp"
  
  # Define what gets indexed
  def search_data
    {
      title: title,
      content: content,
      category: category&.name,
      tags: tags.map(&:name),
      author: user.name,
      published_at: published_at,
      views_count: views_count
    }
  end
  
  # Index conditions
  scope :search_import, -> { includes(:user, :category, :tags).published }
end
```

```ruby
# Index data
Post.reindex  # index ทั้งหมด

# ค้นหา
results = Post.search("rails tutorial",
  fields: [:title, :content],
  highlight: { fields: [:title, :content] },
  suggest: true,
  limit: 20,
  page: 1
)

results.each do |post|
  puts post.title
  puts post.search_highlights[:title]  # highlighted text
end

results.suggestions  # spell suggestions
results.total_count  # total results
```

```ruby
# Advanced search
results = Post.search("rails",
  # Filtering
  where: {
    category: "เทคโนโลยี",
    published_at: { gte: 1.month.ago },
    views_count: { gte: 100 }
  },
  
  # Aggregations
  aggs: [:category, :tags],
  
  # Sorting
  order: { published_at: :desc },
  
  # Boost
  boost_by: [:views_count],
  boost_where: { featured: true }
)

# Facets/Aggregations
results.aggs[:category]["buckets"].each do |bucket|
  puts "#{bucket['key']}: #{bucket['doc_count']}"
end
```

```ruby
# Controller
class SearchController < ApplicationController
  def index
    @query = params[:q]
    @results = Post.search(
      @query.presence || "*",
      where: search_filters,
      aggs: [:category, :tags],
      page: params[:page],
      per_page: 20,
      highlight: { tag: "<mark>" }
    )
    
    @categories = @results.aggs[:category]["buckets"]
    @tag_list = @results.aggs[:tags]["buckets"]
  end
  
  def autocomplete
    render json: Post.search(
      params[:q],
      fields: ["title^5", "content"],
      match: :text_start,
      limit: 5,
      load: false  # ไม่โหลด ActiveRecord objects
    ).map(&:title)
  end
  
  private
  
  def search_filters
    filters = {}
    filters[:category] = params[:category] if params[:category].present?
    filters[:tags] = { all: params[:tags].split(",") } if params[:tags].present?
    filters
  end
end
```

## ขั้นตอนที่ 1106: Meilisearch

```ruby
# Gemfile
gem 'meilisearch-rails'

# bundle install
```

```ruby
# config/initializers/meilisearch.rb
MeiliSearch::Rails.configuration = {
  meilisearch_url: ENV['MEILISEARCH_URL'] || 'http://localhost:7700',
  meilisearch_api_key: ENV['MEILISEARCH_API_KEY']
}
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include MeiliSearch::Rails
  
  meilisearch do
    attribute :title, :content, :category_name, :tag_names, :published_at
    
    searchable_attributes [:title, :content]
    displayed_attributes [:id, :title, :content, :category_name]
    sortable_attributes [:published_at, :views_count]
    filterable_attributes [:category_name, :tag_names]
    
    ranking_rules [
      'words',
      'typo',
      'proximity',
      'attribute',
      'sort',
      'exactness'
    ]
  end
  
  def category_name
    category&.name
  end
  
  def tag_names
    tags.map(&:name)
  end
end
```

```ruby
# Search
results = Post.search(
  "ruby on rails",
  filter: "category_name = 'เทคโนโลยี'",
  sort: ["published_at:desc"],
  limit: 20,
  offset: 0,
  attributes_to_highlight: ["title", "content"],
  highlight_pre_tag: "<mark>",
  highlight_post_tag: "</mark>"
)
```

## ขั้นตอนที่ 1107: Autocomplete / Typeahead

```ruby
# app/controllers/search_controller.rb
def autocomplete
  @suggestions = if params[:q].length >= 2
    Post.search(params[:q],
      fields: ["title^3", "tags.name"],
      match: :word_start,
      limit: 8,
      load: false,
      misspellings: false
    ).map { |r| { id: r.id, title: r.title } }
  else
    []
  end
  
  render json: @suggestions
end
```

```javascript
// app/javascript/controllers/search_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "results"]
  static values = { url: String }
  
  connect() {
    this.debounceTimer = null
  }
  
  search() {
    clearTimeout(this.debounceTimer)
    this.debounceTimer = setTimeout(() => {
      this.fetchSuggestions()
    }, 300)
  }
  
  async fetchSuggestions() {
    const query = this.inputTarget.value
    if (query.length < 2) {
      this.resultsTarget.innerHTML = ""
      return
    }
    
    const response = await fetch(
      `${this.urlValue}?q=${encodeURIComponent(query)}`,
      { headers: { "Accept": "application/json" } }
    )
    const suggestions = await response.json()
    
    this.resultsTarget.innerHTML = suggestions
      .map(s => `<li><a href="/posts/${s.id}">${s.title}</a></li>`)
      .join("")
  }
}
```

```erb
<%# view %>
<div data-controller="search" data-search-url-value="<%= autocomplete_search_path %>">
  <input type="text" 
         data-search-target="input"
         data-action="input->search#search"
         placeholder="ค้นหา...">
  <ul data-search-target="results"></ul>
</div>
```

## ขั้นตอนที่ 1108: Search Indexing Strategies

```ruby
# Async indexing (ใช้กับ Sidekiq)
class Post < ApplicationRecord
  searchkick callbacks: :async  # index ใน background
end

# Queue-based indexing
class PostIndexJob < ApplicationJob
  queue_as :search
  
  def perform(post_id)
    post = Post.find_by(id: post_id)
    post&.reindex
  end
end

# Bulk indexing
Post.reindex          # reindex ทั้งหมด
Post.search_index.refresh  # refresh Elasticsearch index
```

```ruby
# Index management
# ไม่ index บางกรณี
class Post < ApplicationRecord
  searchkick
  
  def should_index?
    published? && !deleted?
  end
end
```

## ขั้นตอนที่ 1109: Advanced Search Filters

```ruby
# app/services/post_search_service.rb
class PostSearchService
  def initialize(params, current_user: nil)
    @params = params
    @current_user = current_user
  end
  
  def call
    scope = base_scope
    scope = apply_text_search(scope)
    scope = apply_filters(scope)
    scope = apply_sorting(scope)
    scope
  end
  
  private
  
  def base_scope
    Post.includes(:user, :category, :tags).published
  end
  
  def apply_text_search(scope)
    return scope if @params[:query].blank?
    
    scope.search(@params[:query])
  end
  
  def apply_filters(scope)
    scope = scope.where(category_id: @params[:category_id]) if @params[:category_id].present?
    scope = scope.tagged_with(@params[:tag]) if @params[:tag].present?
    scope = filter_by_date(scope)
    scope
  end
  
  def filter_by_date(scope)
    if @params[:date_from].present?
      scope = scope.where("created_at >= ?", @params[:date_from].to_date.beginning_of_day)
    end
    if @params[:date_to].present?
      scope = scope.where("created_at <= ?", @params[:date_to].to_date.end_of_day)
    end
    scope
  end
  
  def apply_sorting(scope)
    case @params[:sort]
    when "newest"   then scope.order(created_at: :desc)
    when "oldest"   then scope.order(created_at: :asc)
    when "popular"  then scope.order(views_count: :desc)
    when "az"       then scope.order(title: :asc)
    else                 scope.order(created_at: :desc)
    end
  end
end
```

## ขั้นตอนที่ 1110: Search Analytics

```ruby
# app/models/search_query.rb
class SearchQuery < ApplicationRecord
  # columns: query, results_count, user_id, ip_address, created_at
  
  validates :query, presence: true
  
  scope :popular, -> { group(:query).order('count_all DESC').count }
  scope :no_results, -> { where(results_count: 0) }
  scope :recent, -> { where("created_at > ?", 7.days.ago) }
end
```

```ruby
# Track searches
class SearchController < ApplicationController
  def index
    @query = params[:q]&.strip
    @results = perform_search
    
    track_search if @query.present?
  end
  
  private
  
  def track_search
    SearchQuery.create!(
      query: @query,
      results_count: @results.total_count,
      user_id: current_user&.id,
      ip_address: request.remote_ip
    )
  end
end
```

---

## แบบฝึกหัด: Search (20 ข้อ)

### ข้อที่ 1: Basic LIKE Search
**เขียน scope ค้นหา Post ด้วย title และ content:**

**เฉลย:**
```ruby
scope :search, ->(query) {
  where("title ILIKE :q OR content ILIKE :q", q: "%#{query}%") if query.present?
}
```

### ข้อที่ 2: pg_search
```
ติดตั้ง pg_search และสร้าง search scope พร้อม weighted fields
```

### ข้อที่ 3: Ransack Form
```
สร้าง search form ด้วย Ransack ที่มี text, date range, และ dropdown
```

### ข้อที่ 4: Elasticsearch Reindex
```
สร้าง Rake task ที่ reindex Posts ทั้งหมดและแสดง progress
```

### ข้อที่ 5: Autocomplete API
```
สร้าง endpoint สำหรับ autocomplete ที่ return JSON array ของ suggestions
```

### ข้อที่ 6-20 (แบบสรุป)

**ข้อ 6:** Meilisearch setup และ configuration
**ข้อ 7:** Search with filters (category, date, status)
**ข้อ 8:** Search analytics - track queries และ results count
**ข้อ 9:** Highlight search terms ใน results
**ข้อ 10:** Paginate search results
**ข้อ 11:** Multi-model search ด้วย PgSearch multisearch
**ข้อ 12:** Searchkick with aggregations (faceted search)
**ข้อ 13:** Typeahead controller ด้วย Stimulus
**ข้อ 14:** Search ด้วย synonyms
**ข้อ 15:** Fuzzy search (ค้นหาแม้สะกดผิดเล็กน้อย)
**ข้อ 16:** Search scopes ด้วย Ransack custom ransackers
**ข้อ 17:** Search indexing ใน background job
**ข้อ 18:** Cache search results
**ข้อ 19:** Test search functionality
**ข้อ 20:** Search API endpoint พร้อม JSON response

---

## สรุป: Search Strategies

| วิธีการ | เหมาะกับ | ข้อดี | ข้อเสีย |
|--------|---------|-------|--------|
| LIKE/ILIKE | ข้อมูลน้อย | ง่าย ไม่ต้องติดตั้งเพิ่ม | ช้า, ไม่ flexible |
| pg_search | PostgreSQL, ข้อมูลปานกลาง | ใช้ DB เดิม, full-text | ไม่ flexible เท่า Elasticsearch |
| Ransack | Advanced filtering | Form helper, หลาย filter | ไม่เหมาะ full-text search |
| Elasticsearch | ข้อมูลมาก, complex search | เร็ว, flexible มาก | ต้องติดตั้ง service เพิ่ม |
| Meilisearch | UX-focused | เร็ว, typo-tolerant | เปรียบเทียบกับ ES |

**Key Takeaways:**
1. เริ่มด้วย LIKE/pg_search ก่อน
2. ใช้ Ransack สำหรับ filter forms
3. ใช้ Elasticsearch/Meilisearch เมื่อ traffic สูงหรือ search ซับซ้อน
4. Track searches เพื่อ improve UX
5. Cache search results เมื่อเหมาะสม
