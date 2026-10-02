# Part 31: Introduction to Ruby on Rails (ขั้นตอนที่ 666-685)

## บทนำ

Ruby on Rails เป็น web framework ที่ทรงพลังที่สุดตัวหนึ่งในโลก เขียนด้วยภาษา Ruby และถูกออกแบบมาเพื่อให้นักพัฒนาสามารถสร้าง web application ได้อย่างรวดเร็วและมีประสิทธิภาพ

---

## ขั้นตอนที่ 666: What is Ruby on Rails?

### ประวัติของ Rails

Ruby on Rails (หรือที่เรียกสั้นๆ ว่า Rails) ถูกสร้างขึ้นโดย **David Heinemeier Hansson (DHH)** ในปี 2003 ขณะที่เขากำลังพัฒนา Basecamp ซึ่งเป็น project management tool สำหรับ 37signals (ปัจจุบันคือ Basecamp)

**เหตุการณ์สำคัญในประวัติศาสตร์ Rails:**

- **2003**: DHH สร้าง Rails จาก Basecamp codebase
- **2004**: Rails ถูก extract และ release เป็น open source
- **2005**: Rails 1.0 ถูก release อย่างเป็นทางการ
- **2007**: Rails 2.0 - เพิ่ม RESTful routing
- **2010**: Rails 3.0 - merge กับ Merb framework
- **2011**: Rails 3.1 - เพิ่ม Asset Pipeline
- **2013**: Rails 4.0 - Strong Parameters, Turbolinks
- **2016**: Rails 5.0 - Action Cable (WebSockets), API mode
- **2018**: Rails 5.2 - Active Storage, Credentials
- **2019**: Rails 6.0 - Action Mailbox, Action Text, Multiple Databases
- **2021**: Rails 7.0 - Hotwire (Turbo + Stimulus), Import Maps
- **2023**: Rails 7.1 - Solid Cache, Solid Queue
- **2024**: Rails 8.0 - Solid Trifecta, Propshaft

### Rails คืออะไร?

Rails เป็น **full-stack web framework** ที่ประกอบด้วย:
- **MVC Architecture** - Model, View, Controller
- **ORM (Object-Relational Mapping)** - Active Record
- **Routing** - RESTful URL patterns
- **View Layer** - ERB templates
- **Testing Framework** - Minitest / RSpec

```ruby
# ตัวอย่าง Rails application แบบง่าย
# config/routes.rb
Rails.application.routes.draw do
  resources :articles
  root "articles#index"
end

# app/models/article.rb
class Article < ApplicationRecord
  validates :title, presence: true
  validates :body, presence: true, length: { minimum: 10 }
end

# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  def index
    @articles = Article.all
  end

  def show
    @article = Article.find(params[:id])
  end

  def new
    @article = Article.new
  end

  def create
    @article = Article.new(article_params)
    if @article.save
      redirect_to @article, notice: "Article was successfully created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def article_params
    params.require(:article).permit(:title, :body)
  end
end
```

---

## ขั้นตอนที่ 667: Convention over Configuration (CoC)

หนึ่งในหลักการสำคัญที่สุดของ Rails คือ **Convention over Configuration** ซึ่งหมายความว่า Rails มี "conventions" (ข้อตกลงร่วม) ที่กำหนดไว้แล้ว ถ้าคุณทำตาม conventions เหล่านี้ คุณไม่จำเป็นต้อง configure อะไรมาก

### ตัวอย่าง Convention ใน Rails

**1. ชื่อ Model และ Table:**
```ruby
# Model ชื่อ Article (singular, CamelCase)
# Rails จะหา table ชื่อ articles (plural, snake_case) โดยอัตโนมัติ
class Article < ApplicationRecord
  # ไม่ต้องระบุ table_name เพราะ Rails รู้จาก convention
end

# Model ชื่อ PersonProfile
# Rails จะหา table ชื่อ person_profiles
class PersonProfile < ApplicationRecord
end

# ถ้าต้องการ override convention:
class LegacyData < ApplicationRecord
  self.table_name = "old_table_name"  # ระบุเองถ้าชื่อไม่ตรง convention
end
```

**2. Controller และ View:**
```ruby
# Controller ชื่อ ArticlesController
# Rails จะหา views ใน app/views/articles/
class ArticlesController < ApplicationController
  def index
    # Rails จะ render app/views/articles/index.html.erb โดยอัตโนมัติ
    @articles = Article.all
  end

  def show
    # Rails จะ render app/views/articles/show.html.erb โดยอัตโนมัติ
    @article = Article.find(params[:id])
  end
end
```

**3. Primary Key:**
```ruby
# Rails assume ว่าทุก table มี column ชื่อ id เป็น primary key
# ไม่ต้องระบุเพิ่มเติม

# ถ้าต้องการ override:
class Article < ApplicationRecord
  self.primary_key = "article_id"
end
```

**4. Foreign Keys:**
```ruby
# ถ้ามี belongs_to :user
# Rails จะ assume ว่ามี column ชื่อ user_id ใน table
class Article < ApplicationRecord
  belongs_to :user
  # Rails จะ join กับ users table ผ่าน user_id column
end
```

**5. Timestamps:**
```ruby
# Rails จะจัดการ created_at และ updated_at โดยอัตโนมัติ
# ถ้า table มี columns เหล่านี้

# Migration:
create_table :articles do |t|
  t.string :title
  t.text :body
  t.timestamps  # สร้าง created_at และ updated_at อัตโนมัติ
end
```

### ประโยชน์ของ Convention over Configuration

```ruby
# โปรเจกต์ที่ไม่ใช้ convention (เช่น PHP แบบเก่า)
# ต้อง configure ทุกอย่างเอง:
$db_table = "my_articles_table";
$primary_key = "article_id";
$created_column = "creation_date";
// ... และอีกมากมาย

# โปรเจกต์ Rails ที่ใช้ convention
# ทุกอย่างทำงานได้เลยโดยไม่ต้อง configure:
class Article < ApplicationRecord
  # นั่นแหละ! แค่นี้พอ
end
```

---

## ขั้นตอนที่ 668: MVC Architecture ใน Rails

### MVC คืออะไร?

**MVC (Model-View-Controller)** เป็น architectural pattern ที่แบ่งการทำงานของ application ออกเป็น 3 ส่วน:

```
Request → Router → Controller → Model → Database
                              ↓
                            View
                              ↓
                           Response
```

### Model

Model คือ business logic และ data layer ของ application:

```ruby
# app/models/article.rb
class Article < ApplicationRecord
  # Associations
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :tags, through: :article_tags

  # Validations
  validates :title, presence: true, length: { maximum: 255 }
  validates :body, presence: true, length: { minimum: 10 }
  validates :status, inclusion: { in: %w[draft published archived] }

  # Callbacks
  before_save :generate_slug
  after_create :send_notification

  # Scopes
  scope :published, -> { where(status: "published") }
  scope :recent, -> { order(created_at: :desc).limit(10) }
  scope :by_user, ->(user) { where(user: user) }

  # Class methods
  def self.search(query)
    where("title LIKE ? OR body LIKE ?", "%#{query}%", "%#{query}%")
  end

  # Instance methods
  def published?
    status == "published"
  end

  def word_count
    body.split.size
  end

  def reading_time
    (word_count / 200.0).ceil
  end

  private

  def generate_slug
    self.slug = title.parameterize if title_changed?
  end

  def send_notification
    ArticleMailer.new_article_notification(self).deliver_later
  end
end
```

### View

View คือ presentation layer - สิ่งที่ user เห็น:

```erb
<!-- app/views/articles/index.html.erb -->
<h1>บทความทั้งหมด</h1>

<% if @articles.any? %>
  <div class="articles-grid">
    <% @articles.each do |article| %>
      <div class="article-card">
        <h2><%= link_to article.title, article_path(article) %></h2>
        <p class="meta">
          โดย <%= article.user.name %> |
          <%= article.created_at.strftime("%d/%m/%Y") %> |
          อ่าน <%= article.reading_time %> นาที
        </p>
        <p class="excerpt">
          <%= truncate(article.body, length: 150) %>
        </p>
        <div class="actions">
          <%= link_to "อ่านเพิ่มเติม", article_path(article), class: "btn btn-primary" %>
          <% if current_user == article.user %>
            <%= link_to "แก้ไข", edit_article_path(article), class: "btn btn-secondary" %>
            <%= button_to "ลบ", article_path(article), method: :delete,
                data: { confirm: "คุณแน่ใจหรือไม่?" }, class: "btn btn-danger" %>
          <% end %>
        </div>
      </div>
    <% end %>
  </div>
<% else %>
  <p>ยังไม่มีบทความ</p>
<% end %>

<%= link_to "เขียนบทความใหม่", new_article_path, class: "btn btn-success" %>
```

### Controller

Controller รับ request และประสานงานระหว่าง Model และ View:

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_article, only: [:show, :edit, :update, :destroy]
  before_action :authorize_user!, only: [:edit, :update, :destroy]

  def index
    @articles = Article.published
                       .includes(:user, :tags)
                       .order(created_at: :desc)
                       .page(params[:page])
                       .per(12)

    # ถ้ามี search query
    if params[:q].present?
      @articles = @articles.search(params[:q])
    end
  end

  def show
    @comments = @article.comments.includes(:user).order(created_at: :asc)
    @new_comment = Comment.new
  end

  def new
    @article = current_user.articles.build
  end

  def create
    @article = current_user.articles.build(article_params)

    if @article.save
      redirect_to @article, notice: "บทความถูกสร้างเรียบร้อยแล้ว"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @article.update(article_params)
      redirect_to @article, notice: "บทความถูกอัปเดตเรียบร้อยแล้ว"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @article.destroy
    redirect_to articles_path, notice: "บทความถูกลบเรียบร้อยแล้ว"
  end

  private

  def set_article
    @article = Article.find(params[:id])
  end

  def article_params
    params.require(:article).permit(:title, :body, :status, tag_ids: [])
  end

  def authorize_user!
    redirect_to root_path, alert: "ไม่มีสิทธิ์เข้าถึง" unless @article.user == current_user
  end
end
```

### Request-Response Cycle ใน Rails

```
1. Browser ส่ง HTTP Request → GET /articles
2. Web Server (Puma) รับ request
3. Rails Router วิเคราะห์ URL
   config/routes.rb: resources :articles
   → ส่งไปยัง ArticlesController#index
4. Controller รัน before_actions
5. Controller เรียก Model: Article.all
6. Model ติดต่อ Database
7. Database return ข้อมูล
8. Controller ส่ง data ไปยัง View
9. View render HTML
10. Controller ส่ง HTTP Response กลับ Browser
```

---

## ขั้นตอนที่ 669: Rails Philosophy - DRY

### DRY (Don't Repeat Yourself)

**DRY** หมายความว่าทุก piece of knowledge ควรมีการแสดงออกเดียวใน system

```ruby
# BAD - Repeating yourself (ไม่ DRY)
class ArticlesController < ApplicationController
  def index
    @articles = Article.where(status: "published")
                       .order(created_at: :desc)
  end

  def show
    @article = Article.where(status: "published")
                      .find(params[:id])
  end
end

# GOOD - DRY approach ด้วย scope
class Article < ApplicationRecord
  scope :published, -> { where(status: "published") }
end

class ArticlesController < ApplicationController
  def index
    @articles = Article.published.order(created_at: :desc)
  end

  def show
    @article = Article.published.find(params[:id])
  end
end
```

**DRY ใน Views:**

```erb
<!-- BAD - Repeating navigation -->
<!-- app/views/articles/index.html.erb -->
<nav>
  <ul>
    <li><a href="/">หน้าแรก</a></li>
    <li><a href="/articles">บทความ</a></li>
    <li><a href="/about">เกี่ยวกับ</a></li>
  </ul>
</nav>
<!-- content here -->

<!-- app/views/articles/show.html.erb -->
<nav>
  <ul>
    <li><a href="/">หน้าแรก</a></li>
    <li><a href="/articles">บทความ</a></li>
    <li><a href="/about">เกี่ยวกับ</a></li>
  </ul>
</nav>
<!-- content here -->

<!-- GOOD - ใช้ Partial -->
<!-- app/views/shared/_navigation.html.erb -->
<nav>
  <ul>
    <li><%= link_to "หน้าแรก", root_path %></li>
    <li><%= link_to "บทความ", articles_path %></li>
    <li><%= link_to "เกี่ยวกับ", about_path %></li>
  </ul>
</nav>

<!-- app/views/articles/index.html.erb -->
<%= render "shared/navigation" %>
<!-- content here -->

<!-- app/views/articles/show.html.erb -->
<%= render "shared/navigation" %>
<!-- content here -->
```

**DRY ด้วย Concerns:**

```ruby
# app/models/concerns/trackable.rb
module Trackable
  extend ActiveSupport::Concern

  included do
    belongs_to :created_by, class_name: "User", optional: true
    belongs_to :updated_by, class_name: "User", optional: true

    before_create :set_created_by
    before_save :set_updated_by
  end

  private

  def set_created_by
    self.created_by = Current.user
  end

  def set_updated_by
    self.updated_by = Current.user
  end
end

# app/models/article.rb
class Article < ApplicationRecord
  include Trackable
  # ได้รับ trackable behavior โดยไม่ต้อง repeat code
end

# app/models/comment.rb
class Comment < ApplicationRecord
  include Trackable
  # เช่นเดียวกัน
end
```

**DRY ด้วย Helper Methods:**

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def format_date(date)
    return "ไม่ระบุ" if date.nil?
    date.strftime("%d %B %Y")
  end

  def format_currency(amount)
    number_to_currency(amount, unit: "฿", format: "%u%n", precision: 2)
  end

  def status_badge(status)
    css_class = case status
                when "published" then "badge-success"
                when "draft" then "badge-warning"
                when "archived" then "badge-secondary"
                end
    content_tag(:span, status.capitalize, class: "badge #{css_class}")
  end
end

# ใช้ใน views:
# <%= format_date(article.created_at) %>
# <%= format_currency(product.price) %>
# <%= status_badge(article.status) %>
```

---

## ขั้นตอนที่ 670: Rails vs Other Frameworks

### เปรียบเทียบ Rails กับ Frameworks อื่น

**Rails vs Django (Python):**

```
Rails (Ruby):
+ Convention over Configuration แข็งแกร่งกว่า
+ Scaffolding ดีกว่า
+ Active Record เขียนง่าย
- Ruby ช้ากว่า Python เล็กน้อย
- Deployment ซับซ้อนกว่า

Django (Python):
+ Python ใช้กันแพร่หลายมากกว่า
+ Admin interface built-in ดีมาก
+ Data science / ML ecosystem ดีกว่า
- Configuration มากกว่า
- ORM ซับซ้อนกว่า
```

**Rails vs Laravel (PHP):**

```
Rails (Ruby):
+ Syntax สะอาด อ่านง่าย
+ Convention แข็งแกร่ง
+ Testing culture ดีกว่า
- Ruby developer หาได้ยากกว่า PHP
- Hosting ราคาแพงกว่า

Laravel (PHP):
+ PHP hosting ถูกและหาง่าย
+ PHP developer มากกว่า
+ Ecosystem ใหญ่กว่า
- Convention น้อยกว่า Rails
- Verbosity มากกว่า
```

**Rails vs Express/Node.js:**

```
Rails (Ruby):
+ Full-featured framework (batteries included)
+ Convention ชัดเจน
+ เหมาะกับ rapid prototyping
- Performance ต่ำกว่า Node.js
- Concurrent connections น้อยกว่า

Express/Node.js:
+ Performance สูงมาก (Non-blocking I/O)
+ JavaScript ทั้ง frontend และ backend
+ Ecosystem ใหญ่มาก (npm)
- ต้อง configure ทุกอย่างเอง
- Structure ขึ้นอยู่กับ developer
```

**Rails vs Spring Boot (Java):**

```
Rails (Ruby):
+ Development speed เร็วกว่ามาก
+ Code น้อยกว่ามาก
+ Convention ชัดเจน
- Type safety น้อยกว่า
- Performance ต่ำกว่า

Spring Boot (Java):
+ Enterprise-ready
+ Type safety (Java)
+ Performance สูง
- Verbose มาก
- Learning curve สูง
- Configuration ซับซ้อน
```

### เมื่อไหร่ควรใช้ Rails?

```
✅ เหมาะกับ Rails:
- Startup / MVP development
- Content management systems (CMS)
- E-commerce platforms
- Social networks
- API backends
- B2B SaaS applications
- Rapid prototyping

❌ ไม่เหมาะกับ Rails:
- Real-time high-frequency trading
- Game servers (low latency critical)
- CPU-intensive processing
- Massive concurrent connections (millions)
- Embedded systems
```

---

## ขั้นตอนที่ 671: ติดตั้ง Rails

### ความต้องการของระบบ

```bash
# ตรวจสอบ Ruby version
ruby --version
# ควรได้ ruby 3.2.x หรือสูงกว่า

# ตรวจสอบ Bundler
bundler --version

# ตรวจสอบ Node.js
node --version
# ควรได้ v18.x หรือสูงกว่า

# ตรวจสอบ npm/yarn
npm --version
yarn --version
```

### ติดตั้ง rbenv (Ruby Version Manager)

```bash
# macOS ด้วย Homebrew
brew install rbenv ruby-build

# เพิ่มใน ~/.zshrc หรือ ~/.bashrc
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(rbenv init -)"' >> ~/.zshrc
source ~/.zshrc

# Ubuntu/Debian
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash

# ติดตั้ง Ruby
rbenv install 3.3.0
rbenv global 3.3.0

# ตรวจสอบ
ruby --version
```

### ติดตั้ง Rails

```bash
# ติดตั้ง Rails gem
gem install rails

# หรือระบุ version
gem install rails -v 7.1.3

# ตรวจสอบ
rails --version
# Rails 7.1.3
```

### ติดตั้ง Database

```bash
# SQLite (เหมาะสำหรับ development - ติดตั้งอยู่แล้วบน macOS)
sqlite3 --version

# PostgreSQL
# macOS:
brew install postgresql@14
brew services start postgresql@14

# Ubuntu:
sudo apt-get install postgresql postgresql-contrib libpq-dev

# MySQL
# macOS:
brew install mysql
brew services start mysql

# Ubuntu:
sudo apt-get install mysql-server libmysqlclient-dev
```

### ติดตั้ง Node.js และ Yarn

```bash
# macOS
brew install node
npm install -g yarn

# Ubuntu
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
npm install -g yarn
```

---

## ขั้นตอนที่ 672: Rails New Command

### สร้าง Rails Application ใหม่

```bash
# สร้าง app แบบ default (SQLite, Hotwire)
rails new myapp

# สร้าง app ด้วย PostgreSQL
rails new myapp --database=postgresql

# สร้าง API-only app
rails new myapp --api

# สร้าง app โดยไม่มี Action Cable
rails new myapp --skip-action-cable

# สร้าง app โดยไม่มี JavaScript framework
rails new myapp --skip-javascript

# สร้าง app โดยไม่มี CSS framework
rails new myapp --skip-asset-pipeline

# สร้าง app ด้วย MySQL
rails new myapp --database=mysql

# สร้าง app โดยไม่มี test framework
rails new myapp --skip-test

# สร้าง app ด้วย RSpec
rails new myapp --skip-test  # แล้วเพิ่ม rspec-rails ใน Gemfile

# สร้าง app แบบ minimal
rails new myapp --minimal

# ดู options ทั้งหมด
rails new --help
```

### ตัวอย่าง options ที่ใช้บ่อย

```bash
# สร้าง production-ready app
rails new myapp \
  --database=postgresql \
  --skip-test \
  --css=tailwind \
  --javascript=importmap

# หรือสั้นกว่า
rails new myapp -d postgresql --css tailwind

# ตัวเลือก --database (-d):
# sqlite3, mysql, postgresql, oracle, frontbase, ibm_db, sqlserver, jdbcmysql, jdbcsqlite3, jdbcpostgresql, jdbc

# ตัวเลือก --css:
# tailwind, bootstrap, bulma, postcss, sass

# ตัวเลือก --javascript:
# importmap, bun, webpack, esbuild, rollup
```

---

## ขั้นตอนที่ 673: โครงสร้าง Rails Project

### ภาพรวม Directory Structure

```
myapp/
├── app/                    # Application code หลัก
│   ├── assets/             # Static files (CSS, JS, Images)
│   ├── channels/           # Action Cable channels
│   ├── controllers/        # Controllers
│   ├── helpers/            # View helpers
│   ├── javascript/         # JavaScript files (Hotwire)
│   ├── jobs/               # Background jobs
│   ├── mailers/            # Email mailers
│   ├── models/             # Models
│   └── views/              # Views
├── bin/                    # Executable scripts
│   ├── bundle
│   ├── rails
│   ├── rake
│   └── setup
├── config/                 # Configuration
│   ├── environments/       # Environment configs
│   ├── initializers/       # Initializer files
│   ├── locales/            # i18n translation files
│   ├── application.rb      # Application config
│   ├── boot.rb             # Boot configuration
│   ├── database.yml        # Database config
│   ├── environment.rb      # Environment loading
│   ├── puma.rb             # Puma web server config
│   ├── routes.rb           # URL routing
│   └── storage.yml         # Active Storage config
├── db/                     # Database files
│   ├── migrate/            # Migration files
│   ├── schema.rb           # Database schema
│   └── seeds.rb            # Seed data
├── lib/                    # Library code
│   ├── assets/             # Library assets
│   └── tasks/              # Rake tasks
├── log/                    # Application logs
├── public/                 # Public web files
│   ├── 404.html
│   ├── 422.html
│   └── 500.html
├── storage/                # Active Storage files
├── test/                   # Test files
│   ├── controllers/
│   ├── fixtures/
│   ├── helpers/
│   ├── integration/
│   ├── mailers/
│   ├── models/
│   └── test_helper.rb
├── tmp/                    # Temporary files
├── vendor/                 # Third-party code
├── .gitignore
├── .ruby-version
├── Gemfile                 # Ruby dependencies
├── Gemfile.lock            # Locked dependency versions
├── README.md
└── Rakefile
```

---

## ขั้นตอนที่ 674: app/ Directory

### app/controllers/

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # ทุก controller จะ inherit จาก ApplicationController
  protect_from_forgery with: :exception

  before_action :set_locale
  before_action :authenticate_user!  # ถ้าใช้ Devise

  private

  def set_locale
    I18n.locale = params[:locale] || :th
  end
end

# app/controllers/concerns/
# เก็บ shared controller behavior

# ตัวอย่าง: app/controllers/concerns/pagination.rb
module Pagination
  extend ActiveSupport::Concern

  private

  def paginate(scope)
    scope.page(params[:page]).per(params[:per_page] || 20)
  end
end
```

### app/models/

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  # ทุก model จะ inherit จาก ApplicationRecord
  primary_abstract_class
end

# app/models/concerns/
# เก็บ shared model behavior

# ตัวอย่าง: app/models/concerns/sluggable.rb
module Sluggable
  extend ActiveSupport::Concern

  included do
    before_save :generate_slug
  end

  private

  def generate_slug
    self.slug = title.parameterize if respond_to?(:title) && title_changed?
  end
end
```

### app/views/

```
app/views/
├── articles/               # Views สำหรับ ArticlesController
│   ├── index.html.erb     # app/views/articles/index.html.erb
│   ├── show.html.erb
│   ├── new.html.erb
│   ├── edit.html.erb
│   └── _form.html.erb     # Partial (ขึ้นต้นด้วย _)
├── layouts/               # Layout templates
│   ├── application.html.erb  # Default layout
│   ├── mailer.html.erb       # Email layout
│   └── mailer.text.erb       # Plain text email layout
└── shared/                # Shared partials
    ├── _navigation.html.erb
    ├── _footer.html.erb
    └── _flash_messages.html.erb
```

### app/assets/

```
app/assets/
├── config/                # Asset manifest
│   └── manifest.js
├── images/                # Image files
└── stylesheets/           # CSS files
    └── application.css
```

### app/javascript/

```
app/javascript/
├── application.js          # Entry point
├── controllers/            # Stimulus controllers
│   ├── index.js
│   └── hello_controller.js
└── channels/              # Action Cable channels
```

### app/helpers/

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  # Helper methods ที่ใช้ได้ในทุก view
  def page_title(title = nil)
    base_title = "MyApp"
    title.present? ? "#{title} | #{base_title}" : base_title
  end
end

# app/helpers/articles_helper.rb
module ArticlesHelper
  # Helper methods เฉพาะสำหรับ articles views
  def article_status_label(article)
    color = article.published? ? "green" : "gray"
    content_tag(:span, article.status, class: "label label-#{color}")
  end
end
```

### app/mailers/

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: "noreply@myapp.com"
  layout "mailer"
end

# app/mailers/article_mailer.rb
class ArticleMailer < ApplicationMailer
  def new_article_notification(article)
    @article = article
    @user = article.user

    mail(
      to: User.admins.pluck(:email),
      subject: "บทความใหม่: #{article.title}"
    )
  end

  def article_published(article)
    @article = article
    @user = article.user

    mail(
      to: @user.email,
      subject: "บทความของคุณถูกเผยแพร่แล้ว"
    )
  end
end
```

### app/jobs/

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  retry_on StandardError, wait: :polynomially_longer, attempts: 5
  discard_on ActiveJob::DeserializationError
end

# app/jobs/send_weekly_digest_job.rb
class SendWeeklyDigestJob < ApplicationJob
  queue_as :mailers

  def perform
    User.subscribed_to_digest.find_each do |user|
      DigestMailer.weekly_digest(user).deliver_now
    end
  end
end
```

---

## ขั้นตอนที่ 675: config/ Directory

### config/routes.rb

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Root route
  root "home#index"

  # RESTful resources
  resources :articles do
    resources :comments, shallow: true
    collection do
      get :search
      get :popular
    end
    member do
      post :publish
      post :unpublish
    end
  end

  # Authentication
  devise_for :users, controllers: {
    sessions: "users/sessions",
    registrations: "users/registrations"
  }

  # API
  namespace :api do
    namespace :v1 do
      resources :articles, only: [:index, :show, :create, :update, :destroy]
    end
  end

  # Health check
  get "/health", to: "health#show"
end
```

### config/database.yml

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: myapp_development
  username: <%= ENV['DB_USERNAME'] %>
  password: <%= ENV['DB_PASSWORD'] %>
  host: localhost

test:
  <<: *default
  database: myapp_test

production:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

### config/application.rb

```ruby
# config/application.rb
require_relative "boot"
require "rails/all"

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

module Myapp
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 7.1

    # Configuration for the application, engines, and railties goes here.
    config.time_zone = "Bangkok"
    config.i18n.default_locale = :th
    config.i18n.available_locales = [:th, :en]

    # Autoload paths
    config.autoload_paths += %W[#{config.root}/lib]

    # Generator defaults
    config.generators do |g|
      g.orm :active_record
      g.test_framework :rspec
      g.stylesheet_engine :scss
      g.javascript_engine :js
      g.helper false
      g.jbuilder false
    end

    # API configuration
    # config.api_only = true  # Uncomment for API-only mode
  end
end
```

### config/environments/

```ruby
# config/environments/development.rb
Rails.application.configure do
  config.cache_classes = false
  config.eager_load = false

  # Show full error reports
  config.consider_all_requests_local = true

  # Enable/disable caching
  if Rails.root.join("tmp/caching-dev.txt").exist?
    config.action_controller.perform_caching = true
    config.cache_store = :memory_store
    config.public_file_server.headers = {
      "Cache-Control" => "public, max-age=#{2.days.to_i}"
    }
  else
    config.action_controller.perform_caching = false
    config.cache_store = :null_store
  end

  # Debug logging
  config.log_level = :debug

  # Bullet gem (N+1 detection)
  config.after_initialize do
    Bullet.enable = true
    Bullet.alert = true
    Bullet.rails_logger = true
  end
end

# config/environments/production.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load = true

  config.consider_all_requests_local = false
  config.action_controller.perform_caching = true

  # Logging
  config.log_level = :info
  config.log_tags = [:request_id]

  # Force SSL
  config.force_ssl = true

  # Asset delivery
  config.assets.js_compressor = :terser

  # Email
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.smtp_settings = {
    address: "smtp.sendgrid.net",
    port: 587,
    user_name: ENV["SENDGRID_USERNAME"],
    password: ENV["SENDGRID_PASSWORD"],
    authentication: :plain,
    enable_starttls_auto: true
  }
end
```

---

## ขั้นตอนที่ 676: db/ Directory

### db/schema.rb

```ruby
# db/schema.rb (auto-generated โดย Rails - ห้ามแก้ตรงๆ)
ActiveRecord::Schema[7.1].define(version: 2024_01_15_123456) do
  # These are extensions that must be enabled in order to support this database
  enable_extension "plpgsql"

  create_table "articles", force: :cascade do |t|
    t.string "title", null: false
    t.text "body", null: false
    t.string "slug"
    t.string "status", default: "draft"
    t.bigint "user_id", null: false
    t.datetime "published_at"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["slug"], name: "index_articles_on_slug", unique: true
    t.index ["user_id"], name: "index_articles_on_user_id"
    t.index ["status"], name: "index_articles_on_status"
  end

  create_table "users", force: :cascade do |t|
    t.string "email", null: false
    t.string "name"
    t.string "encrypted_password", null: false
    t.string "reset_password_token"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["email"], name: "index_users_on_email", unique: true
  end

  add_foreign_key "articles", "users"
end
```

### db/seeds.rb

```ruby
# db/seeds.rb
# ข้อมูล seed สำหรับ development

# สร้าง admin user
admin = User.find_or_create_by!(email: "admin@example.com") do |user|
  user.name = "Admin User"
  user.password = "password123"
  user.role = "admin"
end
puts "Created admin: #{admin.email}"

# สร้าง regular users
5.times do |i|
  user = User.find_or_create_by!(email: "user#{i+1}@example.com") do |u|
    u.name = "User #{i+1}"
    u.password = "password123"
    u.role = "user"
  end
  puts "Created user: #{user.email}"
end

# สร้าง articles
User.all.each do |user|
  rand(3..8).times do |i|
    Article.find_or_create_by!(
      title: "บทความที่ #{i+1} โดย #{user.name}",
      user: user
    ) do |article|
      article.body = Faker::Lorem.paragraphs(number: 5).join("\n\n")
      article.status = ["draft", "published", "published", "published"].sample
      article.published_at = article.status == "published" ? rand(1..30).days.ago : nil
    end
  end
end

puts "Seeds completed!"
puts "Users: #{User.count}"
puts "Articles: #{Article.count}"
```

### db/migrate/

```ruby
# db/migrate/20240115123456_create_articles.rb
class CreateArticles < ActiveRecord::Migration[7.1]
  def change
    create_table :articles do |t|
      t.string :title, null: false
      t.text :body, null: false
      t.string :slug
      t.string :status, default: "draft"
      t.references :user, null: false, foreign_key: true
      t.datetime :published_at

      t.timestamps
    end

    add_index :articles, :slug, unique: true
    add_index :articles, :status
  end
end
```

---

## ขั้นตอนที่ 677: Gemfile และ Bundler

### Gemfile

```ruby
# Gemfile
source "https://rubygems.org"
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby "3.3.0"

# Rails framework
gem "rails", "~> 7.1.3"

# Database
gem "pg", "~> 1.1"  # PostgreSQL
# gem "sqlite3", "~> 1.4"  # SQLite

# Web server
gem "puma", ">= 5.0"

# JSON
gem "jbuilder"

# CSS/JS
gem "importmap-rails"
gem "turbo-rails"
gem "stimulus-rails"
gem "tailwindcss-rails"  # ถ้าใช้ Tailwind CSS

# Credentials
gem "dotenv-rails", groups: [:development, :test]

# Authentication
gem "devise"

# Authorization
gem "pundit"
# หรือ
gem "cancancan"

# Pagination
gem "kaminari"
# หรือ
gem "pagy"

# File Upload
gem "active_storage_validations"
gem "image_processing", ">= 1.2"

# Background Jobs
gem "sidekiq"

# Search
gem "ransack"

# Internationalization
gem "rails-i18n"

# Development tools
group :development, :test do
  gem "debug", platforms: %i[mri windows]
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
  gem "pry-rails"
end

group :development do
  gem "web-console"
  gem "rack-mini-profiler"
  gem "better_errors"
  gem "binding_of_caller"
  gem "bullet"  # N+1 detection
  gem "annotate"  # Annotate models
end

group :test do
  gem "capybara"
  gem "selenium-webdriver"
  gem "shoulda-matchers"
  gem "database_cleaner-active_record"
  gem "webmock"
  gem "vcr"
  gem "simplecov"
end
```

### Bundle Commands

```bash
# ติดตั้ง gems ทั้งหมด
bundle install

# อัพเดท gem เฉพาะตัว
bundle update rails

# อัพเดท gems ทั้งหมด
bundle update

# ดู gems ที่ติดตั้ง
bundle list

# ดู gem dependency
bundle info devise

# รัน command ในบริบท bundle
bundle exec rails server

# สร้าง binstubs
bundle binstubs rspec-core

# ตรวจสอบ outdated gems
bundle outdated
```

---

## ขั้นตอนที่ 678: Rails Generators

### ประเภทของ Generators

```bash
# ดู generators ทั้งหมด
rails generate --help
rails g --help  # short form

# Generate model
rails generate model Article title:string body:text user:references
rails g model Article title:string body:text user:references

# Generate controller
rails generate controller Articles index show new create edit update destroy
rails g controller Articles index show

# Generate scaffold (model + controller + views + tests)
rails generate scaffold Article title:string body:text status:string user:references
rails g scaffold Article title:string body:text

# Generate migration
rails generate migration AddSlugToArticles slug:string:uniq
rails g migration CreateJoinTableArticlesTags articles tags

# Generate job
rails g job SendWeeklyDigest

# Generate mailer
rails g mailer Article new_article_notification

# Generate channel
rails g channel Chat

# Generate concern
rails g concern Sluggable

# Generate helper
rails g helper Articles

# Generate system test
rails g system_test Articles
```

### ตัวอย่าง Output ของ Generators

```bash
# rails g scaffold Article title:string body:text
# สร้างไฟล์เหล่านี้:
invoke  active_record
create    db/migrate/20240115_create_articles.rb
create    app/models/article.rb
invoke    test_unit
create      test/models/article_test.rb
create      test/fixtures/articles.yml
invoke  resource_route
 route    resources :articles
invoke  scaffold_controller
create    app/controllers/articles_controller.rb
invoke    erb
create      app/views/articles
create      app/views/articles/index.html.erb
create      app/views/articles/edit.html.erb
create      app/views/articles/show.html.erb
create      app/views/articles/new.html.erb
create      app/views/articles/_form.html.erb
create      app/views/articles/_article.html.erb
invoke    resource_route
invoke    test_unit
create      test/controllers/articles_controller_test.rb
invoke    helper
create      app/helpers/articles_helper.rb
invoke    jbuilder
create      app/views/articles/index.json.jbuilder
create      app/views/articles/show.json.jbuilder
```

---

## ขั้นตอนที่ 679: Rails Server และ Console

### Rails Server

```bash
# เริ่ม Rails server (default: port 3000)
rails server
rails s  # short form

# เริ่มที่ port อื่น
rails s -p 3001

# เริ่มใน environment อื่น
rails s -e production

# เริ่มที่ IP address อื่น
rails s -b 0.0.0.0

# Bind ทั้ง port และ IP
rails s -p 4000 -b 0.0.0.0
```

### Rails Console

```bash
# เปิด Rails console (interactive Ruby shell)
rails console
rails c  # short form

# เปิดใน environment อื่น
rails c -e production

# Sandbox mode (rollback changes หลัง exit)
rails c --sandbox
```

```ruby
# ตัวอย่างการใช้ Rails console

# CRUD operations
article = Article.create!(title: "Test", body: "Test body content here")
Article.find(1)
Article.where(status: "published").count
Article.last

# ทดสอบ validations
a = Article.new
a.valid?
a.errors.full_messages

# ทดสอบ methods
user = User.first
user.articles.published.count
user.articles.recent.pluck(:title)

# ทดสอบ routes
app.articles_path
app.article_url(1)

# ทดสอบ helpers
helper.number_to_currency(1000)
helper.time_ago_in_words(1.hour.ago)
```

### Rails Routes

```bash
# ดู routes ทั้งหมด
rails routes

# ดู routes ของ controller เฉพาะ
rails routes -c articles

# ดู routes ที่ตรงกับ URL
rails routes -g /articles

# Format เป็น expanded view
rails routes --expanded
```

---

## ขั้นตอนที่ 680: Rails Rake Tasks

### Rake Tasks ที่ใช้บ่อย

```bash
# Database
rails db:create          # สร้าง database
rails db:drop            # ลบ database
rails db:migrate         # รัน migrations
rails db:rollback        # ย้อน migration ล่าสุด
rails db:rollback STEP=3 # ย้อน 3 migrations
rails db:seed            # รัน seed data
rails db:reset           # drop + create + migrate + seed
rails db:schema:load     # โหลด schema.rb โดยตรง
rails db:version         # ดู current migration version
rails db:migrate:status  # ดูสถานะ migrations

# Testing
rails test               # รัน tests ทั้งหมด
rails test:models        # รัน model tests
rails test:controllers   # รัน controller tests
rails test:system        # รัน system tests

# Assets
rails assets:precompile  # Compile assets สำหรับ production
rails assets:clean       # ลบ compiled assets เก่า

# Logging
rails log:clear          # ลบ log files

# สร้าง Rake task เอง
# lib/tasks/maintenance.rake
namespace :maintenance do
  desc "ลบ articles เก่าที่ draft นานกว่า 30 วัน"
  task cleanup_drafts: :environment do
    count = Article.where(status: "draft")
                   .where("created_at < ?", 30.days.ago)
                   .destroy_all
                   .count
    puts "ลบ #{count} draft articles"
  end

  desc "ส่ง weekly digest email"
  task send_weekly_digest: :environment do
    SendWeeklyDigestJob.perform_now
    puts "Weekly digest sent!"
  end
end

# รัน:
# rails maintenance:cleanup_drafts
# rails maintenance:send_weekly_digest
```

---

## ขั้นตอนที่ 681-685: Rails Best Practices และ Tips

### Best Practices

```ruby
# 1. ใช้ Service Objects สำหรับ complex business logic
# app/services/article_publisher.rb
class ArticlePublisher
  def initialize(article, publisher)
    @article = article
    @publisher = publisher
  end

  def call
    return false unless can_publish?

    ActiveRecord::Base.transaction do
      @article.update!(status: "published", published_at: Time.current)
      @article.tags.each(&:increment_usage_count!)
      notify_subscribers
      log_publication
    end

    true
  rescue => e
    Rails.logger.error("Failed to publish article #{@article.id}: #{e.message}")
    false
  end

  private

  def can_publish?
    @publisher.can_publish? && @article.ready_to_publish?
  end

  def notify_subscribers
    @article.user.followers.each do |follower|
      ArticleMailer.new_article_notification(@article, follower).deliver_later
    end
  end

  def log_publication
    AuditLog.create!(
      action: "article_published",
      user: @publisher,
      subject: @article
    )
  end
end

# ใช้ใน controller:
# publisher = ArticlePublisher.new(@article, current_user)
# if publisher.call
#   redirect_to @article, notice: "Published!"
# else
#   flash[:alert] = "Could not publish"
#   render :edit
# end
```

```ruby
# 2. ใช้ Query Objects สำหรับ complex queries
# app/queries/featured_articles_query.rb
class FeaturedArticlesQuery
  def initialize(relation = Article.all)
    @relation = relation
  end

  def call(limit: 10, category: nil)
    result = @relation
      .published
      .includes(:user, :tags)
      .order(views_count: :desc, published_at: :desc)

    result = result.where(category: category) if category.present?
    result.limit(limit)
  end
end

# ใช้:
# FeaturedArticlesQuery.new.call(limit: 5, category: "technology")
```

```ruby
# 3. ใช้ Form Objects สำหรับ complex forms
# app/forms/user_registration_form.rb
class UserRegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :email, :string
  attribute :password, :string
  attribute :name, :string
  attribute :terms_accepted, :boolean

  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :name, presence: true
  validates :terms_accepted, acceptance: true

  def save
    return false unless valid?

    User.create!(
      email: email,
      password: password,
      name: name
    )
    true
  rescue ActiveRecord::RecordInvalid => e
    errors.merge!(e.record.errors)
    false
  end
end
```

```ruby
# 4. ใช้ Presenters/Decorators สำหรับ view logic
# app/presenters/article_presenter.rb
class ArticlePresenter
  delegate_missing_to :@article

  def initialize(article, view_context)
    @article = article
    @view = view_context
  end

  def formatted_date
    @article.published_at&.strftime("%d %B %Y") || "ยังไม่เผยแพร่"
  end

  def status_badge
    color = @article.published? ? "green" : "yellow"
    @view.content_tag(:span, @article.status,
      class: "px-2 py-1 rounded text-#{color}-800 bg-#{color}-100 text-sm")
  end

  def reading_time_text
    mins = @article.reading_time
    "#{mins} นาทีในการอ่าน"
  end

  def truncated_body(length: 150)
    @view.truncate(@article.body, length: length)
  end
end

# ใช้ใน controller:
# @article_presenter = ArticlePresenter.new(@article, view_context)

# ใช้ใน view:
# <%= @article_presenter.formatted_date %>
# <%= @article_presenter.status_badge %>
```

---

## แบบฝึกหัด Part 31 (ขั้นตอนที่ 666-685)

### แบบฝึกหัดที่ 1-5: พื้นฐาน Rails

**ข้อ 1:** สร้าง Rails application ใหม่ชื่อ `blog_app` ด้วย PostgreSQL

```bash
# คำตอบ:
rails new blog_app --database=postgresql
cd blog_app
rails db:create
```

**ข้อ 2:** อธิบาย Convention over Configuration โดยยกตัวอย่าง 3 ตัวอย่าง

```ruby
# คำตอบ:

# Convention 1: Model ชื่อ User → table ชื่อ users
class User < ApplicationRecord
  # ไม่ต้อง self.table_name = "users" เพราะ Rails รู้เอง
end

# Convention 2: Controller ชื่อ UsersController → views ใน app/views/users/
class UsersController < ApplicationController
  def index
    # Rails จะ render app/views/users/index.html.erb อัตโนมัติ
  end
end

# Convention 3: belongs_to :user → มี column user_id
class Post < ApplicationRecord
  belongs_to :user  # Rails รู้ว่าต้อง join กับ users table ผ่าน user_id
end
```

**ข้อ 3:** สร้าง Rails application แบบ API-only

```bash
# คำตอบ:
rails new api_app --api --database=postgresql
```

**ข้อ 4:** อธิบาย MVC ใน Rails และบทบาทของแต่ละส่วน

```
คำตอบ:
- Model: จัดการ business logic และ database operations
  ตัวอย่าง: Article.rb จัดการ validations, queries, callbacks

- View: แสดงผลข้อมูลให้ user เห็น
  ตัวอย่าง: articles/index.html.erb แสดงรายการบทความ

- Controller: รับ request, เรียก model, ส่งผลลัพธ์ไปที่ view
  ตัวอย่าง: ArticlesController#index เรียก Article.all แล้วส่งไป view
```

**ข้อ 5:** อธิบาย DRY principle และยกตัวอย่างในบริบท Rails

```ruby
# คำตอบ:
# DRY = Don't Repeat Yourself
# ตัวอย่างการ violate DRY:
class ArticlesController < ApplicationController
  def index
    @user = User.find(session[:user_id])  # repeated!
  end

  def show
    @user = User.find(session[:user_id])  # repeated!
  end
end

# DRY solution:
class ArticlesController < ApplicationController
  before_action :set_current_user

  def index
    # @current_user พร้อมใช้งาน
  end

  def show
    # @current_user พร้อมใช้งาน
  end

  private

  def set_current_user
    @current_user = User.find(session[:user_id])
  end
end
```

### แบบฝึกหัดที่ 6-10: Directory Structure

**ข้อ 6:** ระบุว่าไฟล์เหล่านี้อยู่ที่ไหนใน Rails project structure:
- Application controller
- Database configuration
- Route definitions
- Seed data
- Database schema

```
คำตอบ:
- Application controller: app/controllers/application_controller.rb
- Database configuration: config/database.yml
- Route definitions: config/routes.rb
- Seed data: db/seeds.rb
- Database schema: db/schema.rb
```

**ข้อ 7:** สร้าง scaffold สำหรับ Post model ที่มี title, content, published columns

```bash
# คำตอบ:
rails generate scaffold Post title:string content:text published:boolean
rails db:migrate
```

**ข้อ 8:** ดู routes ทั้งหมดที่ถูกสร้างจาก scaffold ในข้อ 7

```bash
# คำตอบ:
rails routes -c posts
# หรือ
rails routes | grep post
```

**ข้อ 9:** เขียน Rake task ที่ลบ posts เก่ากว่า 1 ปี

```ruby
# คำตอบ:
# lib/tasks/cleanup.rake
namespace :cleanup do
  desc "ลบ posts เก่ากว่า 1 ปี"
  task old_posts: :environment do
    deleted_count = Post.where("created_at < ?", 1.year.ago).destroy_all.count
    puts "ลบ #{deleted_count} posts เก่า"
  end
end
```

**ข้อ 10:** อธิบาย Gemfile.lock และทำไมต้องเก็บใน version control

```
คำตอบ:
Gemfile.lock เก็บ exact versions ของ gems ทั้งหมดรวมถึง dependencies
ทำไมต้องเก็บใน version control:
1. ทุกคนในทีมใช้ gems version เดียวกัน
2. Production environment ใช้ versions เดียวกับ development
3. ป้องกัน "works on my machine" problems
4. Reproducible builds - สามารถ deploy เวอร์ชันเดิมได้เสมอ
```

### แบบฝึกหัดที่ 11-15: Generators และ Configuration

**ข้อ 11:** Generate model สำหรับ Comment ที่มี body:text และ references ไปยัง User และ Article

```bash
# คำตอบ:
rails g model Comment body:text user:references article:references
rails db:migrate
```

**ข้อ 12:** Generate controller สำหรับ API ที่ return JSON (without views)

```bash
# คำตอบ:
rails g controller Api::V1::Articles index show create update destroy --no-helper --no-assets --no-template-engine
```

**ข้อ 13:** เขียน config/application.rb เพื่อตั้งค่า timezone เป็น Bangkok และ locale เป็น Thai

```ruby
# คำตอบ:
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.load_defaults 7.1
    config.time_zone = "Bangkok"
    config.i18n.default_locale = :th
    config.i18n.available_locales = [:th, :en]
  end
end
```

**ข้อ 14:** สร้าง Concern ที่เพิ่ม soft delete functionality ให้กับ model

```ruby
# คำตอบ:
# app/models/concerns/soft_deletable.rb
module SoftDeletable
  extend ActiveSupport::Concern

  included do
    scope :not_deleted, -> { where(deleted_at: nil) }
    scope :deleted, -> { where.not(deleted_at: nil) }
    default_scope { not_deleted }
  end

  def soft_delete
    update_column(:deleted_at, Time.current)
  end

  def restore
    update_column(:deleted_at, nil)
  end

  def deleted?
    deleted_at.present?
  end
end

# ใช้:
class Article < ApplicationRecord
  include SoftDeletable
end
```

**ข้อ 15:** ตั้งค่า generators ใน application.rb เพื่อใช้ RSpec แทน Minitest

```ruby
# คำตอบ:
# config/application.rb
config.generators do |g|
  g.test_framework :rspec,
    fixtures: true,
    view_specs: false,
    helper_specs: false,
    routing_specs: false,
    controller_specs: false,
    request_specs: true
  g.fixture_replacement :factory_bot, dir: "spec/factories"
end
```

### แบบฝึกหัดที่ 16-20: Rails Philosophy และ Concepts

**ข้อ 16:** สร้าง Service Object สำหรับการ register user ใหม่

```ruby
# คำตอบ:
# app/services/user_registration_service.rb
class UserRegistrationService
  def initialize(params)
    @params = params
  end

  def call
    @user = User.new(@params)

    if @user.save
      send_welcome_email
      create_default_profile
      { success: true, user: @user }
    else
      { success: false, errors: @user.errors.full_messages }
    end
  end

  private

  def send_welcome_email
    UserMailer.welcome_email(@user).deliver_later
  end

  def create_default_profile
    @user.create_profile!(display_name: @user.name)
  end
end
```

**ข้อ 17:** อธิบาย Rails asset pipeline และ วิธีการทำงาน

```
คำตอบ:
Asset Pipeline คือระบบที่จัดการ CSS, JavaScript, และ images:

1. Concatenation: รวม files หลายไฟล์เป็นไฟล์เดียว (ลด HTTP requests)
2. Minification: บีบอัด JS/CSS ให้เล็กลง
3. Fingerprinting: เพิ่ม hash ในชื่อไฟล์ (cache busting)
4. Preprocessing: SCSS → CSS, CoffeeScript → JS

Rails 7 ใช้ Propshaft หรือ Sprockets สำหรับ asset pipeline
```

**ข้อ 18:** อธิบายความแตกต่างระหว่าง development, test, และ production environments

```ruby
# คำตอบ:

# Development:
# - Code reloading ทุก request (ไม่ต้อง restart server)
# - Verbose error messages
# - ไม่ minify assets
# - Database logging
# - Debug mode เปิด

# Test:
# - ใช้ test database แยกต่างหาก
# - Fast feedback
# - Fixtures/Factories สำหรับ test data
# - ไม่ส่ง emails จริง

# Production:
# - Cache everything
# - Minify assets
# - Force SSL
# - Less verbose logging
# - Assets precompiled
# - ENV variables สำหรับ secrets
```

**ข้อ 19:** เขียน Gemfile ที่มี gems สำหรับ authentication, pagination, และ background jobs

```ruby
# คำตอบ:
source "https://rubygems.org"
ruby "3.3.0"

gem "rails", "~> 7.1.3"
gem "pg", "~> 1.1"
gem "puma", ">= 5.0"

# Authentication
gem "devise"

# Pagination
gem "pagy"

# Background Jobs
gem "sidekiq"
gem "sidekiq-cron"  # Scheduled jobs

group :development, :test do
  gem "debug"
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end
```

**ข้อ 20:** สร้าง full-featured Article model ที่มี validations, scopes, และ callbacks ครบถ้วน

```ruby
# คำตอบ:
# app/models/article.rb
class Article < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :article_tags, dependent: :destroy
  has_many :tags, through: :article_tags

  # Validations
  validates :title, presence: true, length: { minimum: 5, maximum: 255 }
  validates :body, presence: true, length: { minimum: 50 }
  validates :status, inclusion: { in: %w[draft published archived] }
  validates :slug, uniqueness: true, allow_nil: true

  # Scopes
  scope :published, -> { where(status: "published") }
  scope :drafts, -> { where(status: "draft") }
  scope :archived, -> { where(status: "archived") }
  scope :recent, -> { order(created_at: :desc) }
  scope :popular, -> { order(views_count: :desc) }
  scope :by_user, ->(user) { where(user: user) }
  scope :search, ->(q) { where("title ILIKE ? OR body ILIKE ?", "%#{q}%", "%#{q}%") }

  # Callbacks
  before_validation :generate_slug, if: :title_changed?
  before_save :set_published_at, if: :status_changed?
  after_create :notify_admin
  after_destroy :cleanup_tags

  # Instance methods
  def published?
    status == "published"
  end

  def draft?
    status == "draft"
  end

  def word_count
    body.to_s.split.size
  end

  def reading_time
    (word_count / 200.0).ceil
  end

  def publish!
    update!(status: "published", published_at: Time.current)
  end

  def archive!
    update!(status: "archived")
  end

  private

  def generate_slug
    self.slug = title.parameterize
    # ถ้า slug ซ้ำ ให้เพิ่ม suffix
    count = 1
    while Article.where(slug: slug).where.not(id: id).exists?
      self.slug = "#{title.parameterize}-#{count}"
      count += 1
    end
  end

  def set_published_at
    if status == "published" && published_at.nil?
      self.published_at = Time.current
    elsif status != "published"
      self.published_at = nil
    end
  end

  def notify_admin
    AdminMailer.new_article_notification(self).deliver_later
  end

  def cleanup_tags
    # ลบ tags ที่ไม่มีบทความแล้ว
    tags.each do |tag|
      tag.destroy if tag.articles.empty?
    end
  end
end
```

---

## สรุป Part 31

ในบทนี้เราได้เรียนรู้:

1. **ประวัติของ Rails** - จาก Basecamp สู่ framework ระดับโลก
2. **Convention over Configuration** - ทำตาม conventions = code น้อยลง
3. **MVC Architecture** - Model (data), View (presentation), Controller (logic)
4. **DRY Principle** - ไม่ repeat code โดยใช้ concerns, helpers, partials
5. **Rails vs Other Frameworks** - จุดแข็งและจุดอ่อน
6. **การติดตั้ง Rails** - rbenv, Ruby, Rails, databases
7. **rails new options** - API mode, database, CSS, JS
8. **Directory Structure** - ทุก folder และ file มีหน้าที่ชัดเจน
9. **Bundler** - จัดการ gem dependencies
10. **Generators** - สร้าง code ได้รวดเร็ว

Rails เป็น framework ที่มีปรัชญาชัดเจนและทรงพลัง เมื่อคุณเข้าใจ conventions แล้ว คุณจะสามารถพัฒนา application ได้อย่างรวดเร็วและมีคุณภาพสูง

---

*ต่อไป: Part 32 - Rails Setup and Structure (ขั้นตอนที่ 686-705)*
