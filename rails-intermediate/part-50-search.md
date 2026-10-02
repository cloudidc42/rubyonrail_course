# ตอนที่ 50: Search (Steps 1101-1120)

## บทนำ

การค้นหาข้อมูลเป็นฟีเจอร์สำคัญของ web application แทบทุกตัว Rails รองรับการค้นหาได้หลายระดับตั้งแต่ LIKE query ง่ายๆ จนถึง full-text search ด้วย Elasticsearch

---

## Step 1101: Simple LIKE Search

### Basic LIKE Query

```ruby
# app/controllers/articles_controller.rb
def index
  @articles = Article.all
  
  if params[:search].present?
    search_term = "%#{params[:search].strip}%"
    @articles = @articles.where(
      "title LIKE ? OR body LIKE ?",
      search_term, search_term
    )
  end
  
  @articles = @articles.order(created_at: :desc).page(params[:page])
end
```

### Case-Insensitive Search (PostgreSQL)

```ruby
# PostgreSQL ใช้ ILIKE แทน LIKE (case-insensitive)
@articles = Article.where("title ILIKE ? OR body ILIKE ?", "%#{query}%", "%#{query}%")

# หรือใช้ lower()
@articles = Article.where(
  "lower(title) LIKE ? OR lower(body) LIKE ?",
  query.downcase.prepend("%").concat("%"),
  query.downcase.prepend("%").concat("%")
)
```

### Search Scope ใน Model

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  scope :search, ->(query) {
    return all if query.blank?
    
    safe_query = query.strip.gsub(/[%_]/, '\\\\\0')  # Escape wildcards
    where("title ILIKE :q OR body ILIKE :q", q: "%#{safe_query}%")
  }
  
  scope :by_author_name, ->(name) {
    return all if name.blank?
    joins(:user).where("users.name ILIKE ?", "%#{name}%")
  }
  
  scope :with_tags, ->(tags) {
    return all if tags.blank?
    joins(:tags).where(tags: { name: tags.split(',').map(&:strip) })
  }
end
```

---

## Step 1102: Ransack Gem

### Setup

```ruby
# Gemfile
gem 'ransack'
```

```bash
bundle install
```

### Basic Ransack Search

```ruby
# app/controllers/articles_controller.rb
def index
  @q = Article.ransack(params[:q])
  @articles = @q.result(distinct: true).includes(:user).order(created_at: :desc)
  @pagy, @articles = pagy(@articles)
end
```

### Search Form

```erb
<%# app/views/articles/index.html.erb %>
<%= search_form_for @q do |f| %>
  <%# Simple search %>
  <div class="search-field">
    <%= f.label :title_cont, "Title contains:" %>
    <%= f.search_field :title_cont, class: "form-control", placeholder: "Search articles..." %>
  </div>
  
  <%# Author name search %>
  <div class="search-field">
    <%= f.label :user_name_cont, "Author:" %>
    <%= f.search_field :user_name_cont, class: "form-control" %>
  </div>
  
  <%# Date range %>
  <div class="search-field">
    <%= f.label :created_at_gteq, "Published after:" %>
    <%= f.date_field :created_at_gteq, class: "form-control" %>
  </div>
  
  <div class="search-field">
    <%= f.label :created_at_lteq, "Published before:" %>
    <%= f.date_field :created_at_lteq, class: "form-control" %>
  </div>
  
  <%# Published status %>
  <div class="search-field">
    <%= f.label :published_eq, "Status:" %>
    <%= f.select :published_eq, [["All", ""], ["Published", true], ["Draft", false]], 
                 { include_blank: false }, class: "form-control" %>
  </div>
  
  <%# Sort %>
  <div class="search-field">
    <%= sort_link(@q, :created_at, "Date") %>
    <%= sort_link(@q, :title, "Title") %>
    <%= sort_link(@q, :views_count, "Views") %>
  </div>
  
  <%= f.submit "Search", class: "btn btn-primary" %>
  <%= link_to "Clear", articles_path, class: "btn btn-secondary" %>
<% end %>
```

### Ransack Predicates

```
_eq        → =  (equal)
_not_eq    → !=
_lt        → < (less than)
_lteq      → <=
_gt        → > (greater than)
_gteq      → >=
_cont      → LIKE %value%
_not_cont  → NOT LIKE
_start     → LIKE value%
_end       → LIKE %value
_in        → IN (array)
_not_in    → NOT IN
_null      → IS NULL
_not_null  → IS NOT NULL
_present   → not null, not blank
_blank     → null or blank
_true      → IS TRUE
_false     → IS FALSE
_matches   → LIKE (SQL wildcards)
```

### Ransack กับ Custom Ransackable

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  def self.ransackable_attributes(auth_object = nil)
    # อนุญาต fields ที่ค้นหาได้
    %w[title body created_at updated_at published views_count]
  end
  
  def self.ransackable_associations(auth_object = nil)
    %w[user comments tags]
  end
  
  # Custom ransack scope
  ransack_scope :popular, ->(value) {
    where("views_count > ?", value.to_i) if value.present?
  }
end
```

---

## Step 1103: pg_search (PostgreSQL Full-Text Search)

### Setup

```ruby
# Gemfile
gem 'pg_search'
```

```bash
bundle install
rails generate pg_search:migration:multisearch
rails db:migrate
```

### Single Model Search

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  include PgSearch::Model
  
  # Full-text search
  pg_search_scope :search_by_full_text,
    against: {
      title: 'A',     # Weight A (most important)
      body: 'B',      # Weight B
      excerpt: 'C'    # Weight C (least important)
    },
    using: {
      tsearch: {
        language: "english",
        tsvector_column: 'searchable',  # pre-built tsvector column
        prefix: true                     # match partial words
      },
      trigram: {  # fuzzy matching
        only: [:title],
        threshold: 0.1
      }
    }
  
  # Simple search
  pg_search_scope :search_title,
    against: :title,
    using: { tsearch: { prefix: true } }
  
  # Multi-table search
  pg_search_scope :search_with_author,
    against: :title,
    associated_against: { user: [:name, :email] },
    using: :tsearch
end
```

### Multi-Search (ค้นหาข้ามหลาย models)

```ruby
# app/models/user.rb
class User < ApplicationRecord
  include PgSearch::Model
  
  multisearchable against: [:name, :email],
                  using: { tsearch: { prefix: true } }
end

# app/models/article.rb
class Article < ApplicationRecord
  include PgSearch::Model
  
  multisearchable against: [:title, :body],
                  if: :published?,
                  additional_attributes: -> (article) {
                    { 
                      status: article.published? ? 'published' : 'draft',
                      author: article.user.name
                    }
                  }
end

# ค้นหาทุก models พร้อมกัน
results = PgSearch.multisearch("ruby on rails")
results.each do |result|
  case result.searchable_type
  when 'Article'
    puts "Article: #{result.searchable.title}"
  when 'User'
    puts "User: #{result.searchable.name}"
  end
end
```

### Tsvector Column (Pre-built index)

```ruby
# migration
class AddSearchableToArticles < ActiveRecord::Migration[7.0]
  def change
    add_column :articles, :searchable, :tsvector
    
    execute <<-SQL
      CREATE INDEX articles_searchable_idx ON articles USING GIN (searchable);
    SQL
    
    execute <<-SQL
      CREATE TRIGGER articles_searchable_update
      BEFORE INSERT OR UPDATE ON articles
      FOR EACH ROW EXECUTE FUNCTION
      tsvector_update_trigger(searchable, 'pg_catalog.english', title, body);
    SQL
  end
end
```

---

## Step 1104: Search Form ที่สมบูรณ์

### Search Form Helper

```ruby
# app/helpers/search_helper.rb
module SearchHelper
  def search_highlighted(text, query)
    return text if query.blank?
    
    highlighted = text.gsub(
      /(#{Regexp.escape(query)})/i,
      '<mark>\1</mark>'
    )
    highlighted.html_safe
  end
  
  def active_search_filter?(key, value = nil)
    if value
      params.dig(:q, key) == value.to_s
    else
      params.dig(:q, key).present?
    end
  end
end
```

### Advanced Search Controller

```ruby
# app/controllers/search_controller.rb
class SearchController < ApplicationController
  def index
    @query = params[:q].to_s.strip
    @type = params[:type] || 'all'
    @results = []
    @total = 0
    
    return if @query.blank?
    
    case @type
    when 'articles'
      search_articles
    when 'users'
      search_users
    else
      search_all
    end
    
    @pagy, @results = pagy_array(@results, items: 20)
  end
  
  private
  
  def search_articles
    @results = Article.search_by_full_text(@query)
                      .published
                      .includes(:user, :tags)
                      .with_pg_search_rank
                      .order("pg_search_rank DESC")
    @total = @results.count
  end
  
  def search_users
    @results = User.where(
      "name ILIKE ? OR email ILIKE ?",
      "%#{@query}%", "%#{@query}%"
    )
    @total = @results.count
  end
  
  def search_all
    article_results = Article.search_by_full_text(@query).published.includes(:user).limit(5).to_a
    user_results = User.where("name ILIKE ?", "%#{@query}%").limit(5).to_a
    
    @results = (article_results + user_results).sort_by { |r| r.respond_to?(:views_count) ? -r.views_count : 0 }
    @total = article_results.length + user_results.length
  end
end
```

---

## Step 1105: Elasticsearch กับ Searchkick

### Setup

```ruby
# Gemfile
gem 'searchkick'
gem 'elasticsearch'  # หรือ 'opensearch-ruby' สำหรับ OpenSearch
```

```bash
# ติดตั้ง Elasticsearch
# macOS
brew install elastic/tap/elasticsearch-full
brew services start elasticsearch-full

# Docker
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  elasticsearch:8.0.0
```

### Setup Model

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :user
  has_many :comments
  has_many :taggings
  has_many :tags, through: :taggings
  
  searchkick(
    # Fields ที่ต้อง index
    text_fields: [:title, :body, :excerpt],
    word_start: [:title],  # prefix search
    
    # Highlight
    highlight: [:title, :body],
    
    # Callbacks
    callbacks: :async,  # reindex ใน background
    
    # Language
    language: "thai"  # หรือ "english"
  )
  
  # Custom search data
  def search_data
    {
      title: title,
      body: body,
      excerpt: body&.truncate(500),
      author: user.name,
      tags: tags.map(&:name),
      published: published,
      published_at: published_at,
      views_count: views_count,
      category: category&.name
    }
  end
  
  # Reindex conditions
  def should_index?
    published?
  end
end
```

### Index Management

```bash
# สร้าง index
Article.reindex

# Reindex เฉพาะบาง records
Article.where(published: true).reindex

# Async reindex (ด้วย background job)
Article.reindex_async
```

### ค้นหาด้วย Searchkick

```ruby
# app/controllers/articles_controller.rb
def index
  @query = params[:q].to_s.strip
  
  if @query.present?
    @articles = Article.search(
      @query,
      
      # Fields ที่ค้นหา
      fields: ["title^3", "body", "author"],  # title มี weight 3x
      
      # Misspelling tolerance
      misspellings: { below: 5 },  # อนุญาต typo ถ้า result < 5
      
      # Filters
      where: {
        published: true,
        created_at: { gte: 1.year.ago }
      },
      
      # Aggregations (facets)
      aggs: [:tags, :category],
      
      # Highlighting
      highlight: { fields: [:title, :body], tag: "<mark>" },
      
      # Sorting
      order: { views_count: :desc },
      
      # Pagination
      page: params[:page],
      per_page: 20,
      
      # Includes (avoid N+1)
      includes: [:user, :tags]
    )
    
    @highlights = @articles.with_details[:hits].map { |h| h[:_highlight] }
    @aggregations = @articles.aggs
  else
    @articles = Article.published.recent.includes(:user)
    @pagy, @articles = pagy(@articles)
  end
end
```

### Faceted Search

```ruby
def index
  @search = Article.search(
    params[:q] || "*",
    where: build_filters,
    aggs: {
      tags: { limit: 20 },
      category: { limit: 10 },
      published_year: {
        date_histogram: {
          field: :published_at,
          calendar_interval: :year
        }
      }
    },
    page: params[:page],
    per_page: 20
  )
  
  @articles = @search.results
  @facets = @search.aggs
end

private

def build_filters
  filters = { published: true }
  filters[:tags] = params[:tags].split(',') if params[:tags].present?
  filters[:category] = params[:category] if params[:category].present?
  filters
end
```

---

## Step 1106: Meilisearch

### Setup

```ruby
# Gemfile
gem 'meilisearch-rails'
```

```bash
# ติดตั้ง Meilisearch
# macOS
brew install meilisearch
meilisearch

# Docker
docker run -it --rm \
  -p 7700:7700 \
  getmeili/meilisearch:v1.0
```

```ruby
# config/initializers/meilisearch.rb
MeiliSearch::Rails.configuration = {
  meilisearch_url: ENV.fetch("MEILISEARCH_URL", "http://localhost:7700"),
  meilisearch_api_key: ENV.fetch("MEILISEARCH_API_KEY", ""),
  timeout: 5,
  max_retries: 2
}
```

### Model Setup

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  include MeiliSearch::Rails
  
  meilisearch do
    attribute :title, :body, :excerpt
    attribute :author do
      user.name
    end
    attribute :tags do
      tags.map(&:name)
    end
    
    searchable_attributes [:title, :body, :author, :tags]
    
    filterable_attributes [:published, :created_at, :tags]
    
    sortable_attributes [:created_at, :views_count, :title]
    
    ranking_rules [
      'words',
      'typo',
      'proximity',
      'attribute',
      'sort',
      'exactness',
      'views_count:desc'
    ]
    
    pagination max_total_hits: 1000
    
    # Reindex condition
    add_index 'articles' do
      attribute :id
      attribute :title
      attribute :body
      
      searchable_attributes [:title, :body]
    end
  end
end
```

### ค้นหาด้วย Meilisearch

```ruby
# Basic search
results = Article.search("ruby on rails")

# Search with filters
results = Article.search("rails", filter: ["published = true"])

# Search with sort
results = Article.search("rails", sort: ["created_at:desc"])

# Pagination
results = Article.search("rails", page: 1, hits_per_page: 20)
results.hits      # Array of results
results.total_hits
results.processing_time_ms

# Faceted search
results = Article.search("rails",
  facets: ["tags", "category"],
  filter: ["published = true"]
)
results.facet_distribution  # { "tags" => {"ruby" => 5, "rails" => 3} }
```

---

## Step 1107: Autocomplete

### Turbo Autocomplete

```erb
<%# app/views/shared/_search.html.erb %>
<div class="search-autocomplete" data-controller="autocomplete">
  <%= form_tag search_path, method: :get, data: { turbo: false } do %>
    <input type="text" 
           name="q" 
           class="search-input"
           data-action="input->autocomplete#search"
           data-autocomplete-target="input"
           autocomplete="off"
           placeholder="ค้นหา...">
           
    <div class="autocomplete-dropdown" data-autocomplete-target="dropdown" hidden>
      <%# Suggestions จะถูก inject ที่นี่ %>
    </div>
    
    <%= submit_tag "ค้นหา", class: "search-btn" %>
  <% end %>
</div>
```

```javascript
// app/javascript/controllers/autocomplete_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "dropdown"]
  
  connect() {
    this.debounceTimer = null
  }
  
  search(event) {
    clearTimeout(this.debounceTimer)
    
    this.debounceTimer = setTimeout(() => {
      const query = event.target.value.trim()
      
      if (query.length < 2) {
        this.hideDropdown()
        return
      }
      
      this.fetchSuggestions(query)
    }, 300)  // Debounce 300ms
  }
  
  async fetchSuggestions(query) {
    try {
      const response = await fetch(`/search/suggestions?q=${encodeURIComponent(query)}`, {
        headers: { 'Accept': 'application/json' }
      })
      
      const data = await response.json()
      this.renderSuggestions(data.suggestions)
    } catch (error) {
      console.error('Autocomplete error:', error)
    }
  }
  
  renderSuggestions(suggestions) {
    if (suggestions.length === 0) {
      this.hideDropdown()
      return
    }
    
    this.dropdownTarget.innerHTML = suggestions.map(s => `
      <div class="suggestion-item" data-action="click->autocomplete#select" data-value="${s.text}">
        <span class="suggestion-icon">${s.type === 'article' ? '📄' : '👤'}</span>
        <span class="suggestion-text">${s.text}</span>
        <span class="suggestion-type">${s.type}</span>
      </div>
    `).join('')
    
    this.showDropdown()
  }
  
  select(event) {
    const value = event.currentTarget.dataset.value
    this.inputTarget.value = value
    this.hideDropdown()
    this.element.querySelector('form').submit()
  }
  
  showDropdown() {
    this.dropdownTarget.hidden = false
  }
  
  hideDropdown() {
    this.dropdownTarget.hidden = true
  }
}
```

```ruby
# app/controllers/search_controller.rb
def suggestions
  query = params[:q].to_s.strip
  suggestions = []
  
  if query.length >= 2
    # Article suggestions
    articles = Article.published
                      .where("title ILIKE ?", "#{query}%")
                      .order(:views_count)
                      .limit(5)
    
    suggestions += articles.map { |a| { text: a.title, type: 'article', id: a.id } }
    
    # User suggestions
    users = User.where("name ILIKE ?", "#{query}%").limit(3)
    suggestions += users.map { |u| { text: u.name, type: 'user', id: u.id } }
  end
  
  render json: { suggestions: suggestions.first(8) }
end
```

---

## Step 1108: Indexing Strategies

### Database Indexes สำหรับ Search

```ruby
# Migration
class AddSearchIndexes < ActiveRecord::Migration[7.0]
  def change
    # Full-text search index (PostgreSQL)
    execute <<-SQL
      CREATE INDEX articles_title_search ON articles USING GIN (to_tsvector('english', title));
      CREATE INDEX articles_body_search ON articles USING GIN (to_tsvector('english', body));
    SQL
    
    # Composite index สำหรับ common queries
    add_index :articles, [:published, :created_at]
    add_index :articles, [:user_id, :published, :created_at]
    add_index :articles, :views_count
    
    # GiST index สำหรับ similarity search
    execute "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
    execute <<-SQL
      CREATE INDEX articles_title_trgm ON articles USING GIN (title gin_trgm_ops);
    SQL
  end
end
```

### Elasticsearch Index Templates

```ruby
# app/models/article.rb - Custom Elasticsearch settings
searchkick(
  settings: {
    index: {
      number_of_shards: 1,
      number_of_replicas: 0
    },
    analysis: {
      analyzer: {
        thai_analyzer: {
          type: 'custom',
          tokenizer: 'thai',
          filter: ['lowercase']
        }
      }
    }
  },
  mappings: {
    properties: {
      title: { 
        type: 'text', 
        analyzer: 'thai_analyzer',
        fields: {
          keyword: { type: 'keyword' }
        }
      },
      body: { 
        type: 'text', 
        analyzer: 'thai_analyzer' 
      }
    }
  }
)
```

---

## แบบฝึกหัด (20 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** เขียน simple search ที่ค้นหา articles ด้วย title และ body
```ruby
# เฉลย
def index
  @articles = if params[:search].present?
    q = "%#{params[:search]}%"
    Article.where("title ILIKE ? OR body ILIKE ?", q, q)
  else
    Article.all
  end
end
```

**ข้อ 2:** เพิ่ม search scope ใน Article model
```ruby
# เฉลย
scope :search, ->(query) {
  where("title ILIKE :q OR body ILIKE :q", q: "%#{query}%") if query.present?
}
```

**ข้อ 3:** setup Ransack และสร้าง search form พื้นฐาน
```ruby
# เฉลย
# controller
def index
  @q = Article.ransack(params[:q])
  @articles = @q.result
end

# view
<%= search_form_for @q do |f| %>
  <%= f.search_field :title_cont %>
  <%= f.submit "Search" %>
<% end %>
```

**ข้อ 4:** เพิ่ม sorting ด้วย Ransack
```erb
<%# เฉลย %>
<%= sort_link(@q, :created_at, "วันที่") %>
<%= sort_link(@q, :title, "หัวข้อ") %>
<%= sort_link(@q, :views_count, "ยอดวิว") %>
```

**ข้อ 5:** setup pg_search สำหรับ Article model
```ruby
# เฉลย
class Article < ApplicationRecord
  include PgSearch::Model
  
  pg_search_scope :full_text_search,
    against: { title: 'A', body: 'B' },
    using: { tsearch: { prefix: true } }
end
```

### ระดับกลาง

**ข้อ 6:** เพิ่ม multi-search ที่ค้นหาได้ทั้ง articles และ users
```ruby
# เฉลย
class User < ApplicationRecord
  include PgSearch::Model
  multisearchable against: [:name, :email]
end

class Article < ApplicationRecord
  include PgSearch::Model
  multisearchable against: [:title, :body], if: :published?
end

# controller
@results = PgSearch.multisearch(params[:q])
```

**ข้อ 7:** setup Searchkick สำหรับ Article model พร้อม custom search data
```ruby
# เฉลย
class Article < ApplicationRecord
  searchkick text_fields: [:title, :body], word_start: [:title]
  
  def search_data
    {
      title: title,
      body: body,
      author: user.name,
      tags: tags.map(&:name),
      published: published
    }
  end
end
```

**ข้อ 8:** เพิ่ม autocomplete endpoint ที่ return suggestions เป็น JSON
```ruby
# เฉลย
def suggestions
  query = params[:q]
  articles = Article.where("title ILIKE ?", "#{query}%").limit(5)
  render json: { suggestions: articles.map { |a| { text: a.title, id: a.id } } }
end
```

**ข้อ 9:** เพิ่ม faceted search กับ category filters
```ruby
# เฉลย
results = Article.search(params[:q],
  aggs: [:category, :tags],
  where: build_facet_filters
)

def build_facet_filters
  filters = { published: true }
  filters[:category] = params[:category] if params[:category].present?
  filters
end
```

**ข้อ 10:** implement search result highlighting
```ruby
# เฉลย กับ pg_search
pg_search_scope :search,
  against: [:title, :body],
  using: {
    tsearch: {
      highlight: {
        StartSel: '<mark>',
        StopSel: '</mark>',
        MaxWords: 30,
        MinWords: 15
      }
    }
  }
```

### ระดับสูง

**ข้อ 11-20:**

```ruby
# เฉลย ข้อ 11 - Search Caching
def index
  @query = params[:q]
  cache_key = "search/#{Digest::MD5.hexdigest(@query)}/page/#{params[:page]}"
  
  @articles = Rails.cache.fetch(cache_key, expires_in: 5.minutes) do
    Article.search(@query, page: params[:page]).to_a
  end
end
```

```ruby
# เฉลย ข้อ 14 - Search Analytics
class SearchQuery < ApplicationRecord
  def self.track(query, user = nil)
    record = find_or_initialize_by(query: query.downcase.strip)
    record.count += 1
    record.last_searched_at = Time.current
    record.user_id = user&.id
    record.save!
  end
end

# ใน controller
SearchQuery.track(params[:q], current_user) if params[:q].present?
```

```ruby
# เฉลย ข้อ 17 - Typo Tolerance
Article.search(params[:q], misspellings: { below: 5, edit_distance: 1 })
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **LIKE Search** - วิธีพื้นฐาน ง่ายแต่ช้าสำหรับ large data
2. **Ransack** - search forms ที่สะดวก มี sort support
3. **pg_search** - PostgreSQL full-text search ที่ดีมาก
4. **Searchkick** - Elasticsearch integration ที่ง่าย
5. **Meilisearch** - fast search engine ที่ setup ง่าย
6. **Autocomplete** - suggestions ด้วย Stimulus
7. **Indexes** - database indexes สำคัญสำหรับ performance

**แนะนำ:**
- โปรเจกต์เล็ก → pg_search (ไม่ต้องการ extra infrastructure)
- โปรเจกต์ใหญ่ที่ต้องการ advanced features → Elasticsearch + Searchkick
- ต้องการ setup ง่าย → Meilisearch
