# ตอนที่ 45: Testing ใน Rails (Steps 991-1010)

## บทนำ

Testing เป็นส่วนสำคัญของการพัฒนา software ที่ดี Rails มีการสนับสนุน testing มาตั้งแต่ต้น ทำให้เขียน tests ได้ง่าย ในตอนนี้เราจะเรียนรู้ทั้ง Minitest (built-in) และ RSpec (third-party) รวมถึงเครื่องมือต่างๆ เช่น Factory Bot, Capybara, WebMock

---

## ขั้นตอนที่ 991: Rails Testing Overview

### โครงสร้าง Test ใน Rails

```
test/ (Minitest)
├── models/           ← Model tests
├── controllers/      ← Controller tests
├── helpers/          ← Helper tests
├── mailers/          ← Mailer tests
├── integration/      ← Integration tests (หลาย controllers)
├── system/           ← System tests (browser)
├── channels/         ← ActionCable tests
├── jobs/             ← Background job tests
├── fixtures/         ← Test data (YAML)
└── test_helper.rb    ← Test configuration

spec/ (RSpec)
├── models/
├── requests/         ← Request specs (controller)
├── views/            ← View specs
├── helpers/
├── mailers/
├── system/           ← System specs (browser)
├── factories/        ← Factory Bot factories
├── support/          ← Shared examples, helpers
└── rails_helper.rb   ← RSpec configuration
```

### Minitest vs RSpec

```ruby
# Minitest (built-in, Rails default)
class PostTest < ActiveSupport::TestCase
  test "should not save post without title" do
    post = Post.new
    assert_not post.save, "บันทึก post โดยไม่มี title"
  end
end

# RSpec (gem, popular in community)
RSpec.describe Post, type: :model do
  it "should not save post without title" do
    post = Post.new
    expect(post).not_to be_valid
    expect(post.errors[:title]).to include("can't be blank")
  end
end
```

### ติดตั้ง RSpec

```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails", "~> 6.0"
  gem "factory_bot_rails"
  gem "faker"
  gem "shoulda-matchers"
end

group :test do
  gem "capybara"
  gem "selenium-webdriver"
  gem "webmock"
  gem "simplecov"
end

# ติดตั้ง
bundle install
rails generate rspec:install
# สร้าง: .rspec, spec/spec_helper.rb, spec/rails_helper.rb
```

### .rspec Configuration

```
# .rspec
--require spec_helper
--format documentation
--color
--order random
```

### spec/rails_helper.rb

```ruby
require "spec_helper"
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rspec/rails"
require "support/factory_bot"
require "support/database_cleaner"
require "support/shoulda_matchers"

RSpec.configure do |config|
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
  config.use_transactional_fixtures = true
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!

  # Shoulda Matchers
  config.include FactoryBot::Syntax::Methods
end
```

---

## ขั้นตอนที่ 992: Model Specs

### Model Spec พื้นฐาน

```ruby
# spec/models/user_spec.rb
require "rails_helper"

RSpec.describe User, type: :model do
  # Association matchers (Shoulda)
  describe "associations" do
    it { should have_many(:posts).dependent(:destroy) }
    it { should have_one(:profile) }
    it { should belong_to(:organization).optional }
  end

  # Validation matchers (Shoulda)
  describe "validations" do
    it { should validate_presence_of(:name) }
    it { should validate_presence_of(:email) }
    it { should validate_uniqueness_of(:email).case_insensitive }
    it { should validate_length_of(:name).is_at_least(2).is_at_most(100) }
    it { should validate_length_of(:password).is_at_least(8) }
    it { should allow_value("test@example.com").for(:email) }
    it { should_not allow_value("invalid").for(:email) }
  end

  # Custom validation tests
  describe "#email format" do
    it "accepts valid email" do
      user = build(:user, email: "valid@example.com")
      expect(user).to be_valid
    end

    it "rejects email without @" do
      user = build(:user, email: "invalidemail.com")
      expect(user).not_to be_valid
      expect(user.errors[:email]).to be_present
    end

    it "rejects email without domain" do
      user = build(:user, email: "user@")
      expect(user).not_to be_valid
    end
  end

  # Instance methods
  describe "#full_name" do
    it "combines first and last name" do
      user = build(:user, first_name: "สมชาย", last_name: "ใจดี")
      expect(user.full_name).to eq("สมชาย ใจดี")
    end

    it "handles missing last name" do
      user = build(:user, first_name: "สมชาย", last_name: nil)
      expect(user.full_name).to eq("สมชาย")
    end
  end

  # Class methods
  describe ".active" do
    let!(:active_users) { create_list(:user, 3, active: true) }
    let!(:inactive_users) { create_list(:user, 2, active: false) }

    it "returns only active users" do
      expect(User.active.count).to eq(3)
      expect(User.active).to match_array(active_users)
    end
  end

  # Callbacks
  describe "callbacks" do
    it "generates slug before create" do
      user = create(:user, name: "John Doe")
      expect(user.slug).to eq("john-doe")
    end

    it "sends welcome email after create" do
      expect {
        create(:user)
      }.to have_enqueued_mail(UserMailer, :welcome_email)
    end
  end

  # Scopes
  describe "scopes" do
    describe ".recent" do
      it "orders by created_at desc" do
        old_user = create(:user, created_at: 1.week.ago)
        new_user = create(:user, created_at: Time.now)
        expect(User.recent.first).to eq(new_user)
      end
    end
  end

  # Enum
  describe "enum status" do
    it { should define_enum_for(:status).with_values(draft: 0, active: 1, suspended: 2) }

    it "defaults to active" do
      user = create(:user)
      expect(user).to be_active
    end
  end
end
```

### Shoulda Matchers Setup

```ruby
# spec/support/shoulda_matchers.rb
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

---

## ขั้นตอนที่ 993: Request Specs (Controller Specs)

### Request Spec พื้นฐาน

```ruby
# spec/requests/posts_spec.rb
require "rails_helper"

RSpec.describe "Posts", type: :request do
  let(:user) { create(:user) }

  describe "GET /posts" do
    let!(:posts) { create_list(:post, 3, published: true, user: user) }

    it "returns success" do
      get posts_path
      expect(response).to have_http_status(:success)
    end

    it "renders the index template" do
      get posts_path
      expect(response).to render_template(:index)
    end

    it "displays posts" do
      get posts_path
      posts.each do |post|
        expect(response.body).to include(post.title)
      end
    end
  end

  describe "GET /posts/:id" do
    let(:post) { create(:post, user: user) }

    context "when post exists" do
      it "returns success" do
        get post_path(post)
        expect(response).to have_http_status(:success)
      end
    end

    context "when post doesn't exist" do
      it "returns 404" do
        get post_path(id: 999999)
        expect(response).to have_http_status(:not_found)
      end
    end
  end

  describe "POST /posts" do
    context "when not authenticated" do
      it "redirects to login" do
        post posts_path, params: { post: { title: "Test" } }
        expect(response).to redirect_to(login_path)
      end
    end

    context "when authenticated" do
      before { sign_in user }

      context "with valid params" do
        let(:valid_params) do
          { post: { title: "New Post", body: "Content here", published: true } }
        end

        it "creates a post" do
          expect {
            post posts_path, params: valid_params
          }.to change(Post, :count).by(1)
        end

        it "redirects to the created post" do
          post posts_path, params: valid_params
          expect(response).to redirect_to(post_path(Post.last))
        end

        it "sets flash notice" do
          post posts_path, params: valid_params
          expect(flash[:notice]).to be_present
        end
      end

      context "with invalid params" do
        let(:invalid_params) { { post: { title: "", body: "" } } }

        it "does not create a post" do
          expect {
            post posts_path, params: invalid_params
          }.not_to change(Post, :count)
        end

        it "renders new template" do
          post posts_path, params: invalid_params
          expect(response).to render_template(:new)
        end

        it "returns unprocessable_entity" do
          post posts_path, params: invalid_params
          expect(response).to have_http_status(:unprocessable_entity)
        end
      end
    end
  end

  describe "PATCH /posts/:id" do
    let(:post) { create(:post, user: user, title: "Original") }

    before { sign_in user }

    it "updates the post" do
      patch post_path(post), params: { post: { title: "Updated" } }
      expect(post.reload.title).to eq("Updated")
    end

    it "cannot update other user's post" do
      other_post = create(:post)
      patch post_path(other_post), params: { post: { title: "Hacked" } }
      expect(response).to have_http_status(:forbidden).or redirect_to(root_path)
    end
  end

  describe "DELETE /posts/:id" do
    let!(:post) { create(:post, user: user) }

    before { sign_in user }

    it "deletes the post" do
      expect {
        delete post_path(post)
      }.to change(Post, :count).by(-1)
    end

    it "redirects to posts index" do
      delete post_path(post)
      expect(response).to redirect_to(posts_path)
    end
  end
end
```

### Authentication Helper สำหรับ Specs

```ruby
# spec/support/authentication_helpers.rb
module AuthenticationHelpers
  def sign_in(user)
    post login_path, params: { email: user.email, password: "password" }
    # หรือสำหรับ Devise:
    # sign_in user (ถ้าใช้ Devise test helpers)
  end

  def sign_out
    delete logout_path
  end
end

RSpec.configure do |config|
  config.include AuthenticationHelpers, type: :request
  config.include AuthenticationHelpers, type: :system
end
```

---

## ขั้นตอนที่ 994: System Specs ด้วย Capybara

### System Spec Setup

```ruby
# Gemfile
gem "capybara"
gem "selenium-webdriver"
gem "webdrivers"  # จัดการ driver downloads

# spec/support/capybara.rb
RSpec.configure do |config|
  config.before(:each, type: :system) do
    driven_by :rack_test  # สำหรับ tests ที่ไม่ต้องใช้ JS
  end

  config.before(:each, type: :system, js: true) do
    driven_by :selenium_chrome_headless
  end
end

# หรือ config ใน spec_helper.rb
Capybara.default_max_wait_time = 5
Capybara.server = :puma, { Silent: true }
```

### System Spec Examples

```ruby
# spec/system/posts_spec.rb
require "rails_helper"

RSpec.describe "Posts", type: :system do
  let(:user) { create(:user) }

  before do
    sign_in user
  end

  describe "viewing posts" do
    let!(:post) { create(:post, title: "Test Post", user: user) }

    it "displays list of posts" do
      visit posts_path
      expect(page).to have_content("Test Post")
    end

    it "can view post details" do
      visit posts_path
      click_link "Test Post"
      expect(current_path).to eq(post_path(post))
      expect(page).to have_content(post.body)
    end
  end

  describe "creating a post" do
    it "can create a post" do
      visit new_post_path

      fill_in "หัวข้อ", with: "My New Post"
      fill_in "เนื้อหา", with: "This is the content of my post"
      select "Ruby", from: "หมวดหมู่"
      check "เผยแพร่บทความ"

      click_button "สร้างบทความ"

      expect(page).to have_content("บทความถูกสร้างแล้ว")
      expect(page).to have_content("My New Post")
    end

    it "shows errors for invalid input" do
      visit new_post_path

      fill_in "หัวข้อ", with: ""  # empty title
      click_button "สร้างบทความ"

      expect(page).to have_content("กรุณากรอกหัวข้อ")
      expect(current_path).to eq(posts_path)
    end
  end

  describe "editing a post", js: true do
    let!(:post) { create(:post, user: user) }

    it "can edit and update a post" do
      visit post_path(post)
      click_link "แก้ไข"

      fill_in "หัวข้อ", with: "Updated Title"
      click_button "อัปเดตบทความ"

      expect(page).to have_content("บทความถูกอัปเดตแล้ว")
      expect(page).to have_content("Updated Title")
    end
  end

  describe "deleting a post", js: true do
    let!(:post) { create(:post, user: user, title: "Delete Me") }

    it "can delete a post" do
      visit posts_path
      expect(page).to have_content("Delete Me")

      accept_confirm do
        click_link "ลบ", href: post_path(post)
      end

      expect(page).not_to have_content("Delete Me")
      expect(page).to have_content("บทความถูกลบแล้ว")
    end
  end
end
```

### Capybara DSL

```ruby
# Navigation
visit root_path
visit "https://example.com/posts"
go_back
go_forward
refresh

# Interaction
click_link "ลิงก์"
click_link "Link Text"
click_button "บันทึก"
click_on "อะไรก็ได้"  # link หรือ button

# Fill in forms
fill_in "Label Name", with: "value"
fill_in "Name", with: "John"
choose "Radio Button Label"
check "Checkbox Label"
uncheck "Checkbox Label"
select "Option", from: "Select Label"
attach_file "File Field", "/path/to/file.pdf"

# Assertions (matchers)
expect(page).to have_content("text")
expect(page).to have_text("text")
expect(page).to have_css("h1", text: "Title")
expect(page).to have_css(".alert-success")
expect(page).to have_selector("table tr", count: 5)
expect(page).to have_link("Click Here")
expect(page).to have_button("Submit")
expect(page).to have_field("Email", with: "test@example.com")
expect(page).to have_checked_field("Remember Me")
expect(page).not_to have_content("Error")

# Scoped queries
within("#post-list") do
  expect(page).to have_css(".post", count: 3)
end

within("table") do
  click_link "Delete"
end

# Screenshots (useful for debugging)
save_screenshot "debug.png"
save_page "debug.html"
```

---

## ขั้นตอนที่ 995: Factory Bot

### Setup Factory Bot

```ruby
# Gemfile
gem "factory_bot_rails"
gem "faker"

# spec/support/factory_bot.rb
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end
```

### สร้าง Factories

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    name { Faker::Name.full_name }
    password { "password123" }
    password_confirmation { "password123" }
    role { "member" }
    active { true }

    # Trait - variants
    trait :admin do
      role { "admin" }
      email { "admin@example.com" }
    end

    trait :inactive do
      active { false }
    end

    trait :with_posts do
      after(:create) do |user|
        create_list(:post, 3, user: user)
      end
    end

    trait :with_profile do
      after(:create) do |user|
        create(:profile, user: user)
      end
    end
  end
end

# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    sequence(:title) { |n| "Post #{n}" }
    body { Faker::Lorem.paragraphs(number: 3).join("\n\n") }
    published { false }
    views_count { 0 }
    association :user

    trait :published do
      published { true }
      published_at { 1.day.ago }
    end

    trait :draft do
      published { false }
    end

    trait :with_comments do
      after(:create) do |post|
        create_list(:comment, 5, post: post)
      end
    end

    trait :with_tags do
      after(:create) do |post|
        tags = create_list(:tag, 3)
        post.tags << tags
      end
    end

    factory :published_post, traits: [:published]
  end
end

# spec/factories/comments.rb
FactoryBot.define do
  factory :comment do
    body { Faker::Lorem.sentence }
    approved { false }
    association :post
    association :user

    trait :approved do
      approved { true }
    end
  end
end

# spec/factories/profiles.rb
FactoryBot.define do
  factory :profile do
    bio { Faker::Lorem.paragraph }
    website { Faker::Internet.url }
    location { Faker::Address.city }
    association :user
  end
end
```

### ใช้ Factory Bot ใน Specs

```ruby
# build - สร้าง object แต่ไม่ save ลง DB
user = build(:user)
user = build(:user, name: "สมชาย")
user = build(:user, :admin)

# create - สร้างและ save ลง DB
user = create(:user)
user = create(:user, :admin)
user = create(:user, email: "custom@example.com")

# build_list / create_list
users = build_list(:user, 5)
posts = create_list(:post, 10, user: user)
admins = create_list(:user, 3, :admin)

# build_stubbed - สร้าง stub (ไม่มีจริงใน DB)
user = build_stubbed(:user)

# attributes_for - แค่ attributes hash
attrs = attributes_for(:user)
# => { name: "...", email: "...", ... }

# Traits
admin = create(:user, :admin)
post = create(:post, :published, :with_comments)
user = create(:user, :with_posts, :with_profile)
```

---

## ขั้นตอนที่ 996: Testing กับ Time

### Freezing Time

```ruby
# ใช้ ActiveSupport::Testing::TimeHelpers
RSpec.describe "time-sensitive logic" do
  include ActiveSupport::Testing::TimeHelpers

  it "creates post with correct published_at" do
    freeze_time do
      post = create(:post, :published)
      expect(post.published_at).to eq(Time.current)
    end
  end

  it "sends reminder after 7 days" do
    user = create(:user, created_at: Time.current)

    travel_to(8.days.from_now) do
      expect(user.should_send_reminder?).to be true
    end
  end

  it "expires token after 24 hours" do
    token = create(:password_reset_token)
    expect(token).to be_valid

    travel_to(25.hours.from_now) do
      expect(token).not_to be_valid
    end
  end
end

# travel (เคลื่อนเวลาไปข้างหน้า)
travel 1.week

# travel_to (ไปยังเวลาที่กำหนด)
travel_to Time.zone.local(2024, 1, 15, 10, 0, 0)

# freeze_time (หยุดเวลา)
freeze_time

# back to current time
travel_back
```

---

## ขั้นตอนที่ 997: WebMock สำหรับ API Mocking

### Setup WebMock

```ruby
# Gemfile
gem "webmock", group: :test

# spec/support/webmock.rb
require "webmock/rspec"

# ปิด real HTTP requests ในทุก specs (ยกเว้น localhost)
WebMock.disable_net_connect!(allow_localhost: true)
```

### ใช้งาน WebMock

```ruby
# spec/services/weather_service_spec.rb
require "rails_helper"

RSpec.describe WeatherService do
  describe "#current_weather" do
    before do
      # Mock HTTP request
      stub_request(:get, "https://api.weather.com/current")
        .with(
          query: { city: "Bangkok", units: "metric" },
          headers: { "Authorization" => "Bearer #{ENV["WEATHER_API_KEY"]}" }
        )
        .to_return(
          status: 200,
          body: {
            temperature: 32,
            humidity: 80,
            description: "Sunny"
          }.to_json,
          headers: { "Content-Type" => "application/json" }
        )
    end

    it "returns weather data" do
      result = WeatherService.new.current_weather("Bangkok")
      expect(result[:temperature]).to eq(32)
      expect(result[:description]).to eq("Sunny")
    end

    it "makes the correct API call" do
      WeatherService.new.current_weather("Bangkok")

      expect(WebMock).to have_requested(:get, "https://api.weather.com/current")
        .with(query: { city: "Bangkok", units: "metric" })
    end
  end

  describe "when API returns error" do
    before do
      stub_request(:get, /api.weather.com/)
        .to_return(status: 500, body: "Server Error")
    end

    it "raises WeatherService::ApiError" do
      expect {
        WeatherService.new.current_weather("Bangkok")
      }.to raise_error(WeatherService::ApiError)
    end
  end

  describe "when API times out" do
    before do
      stub_request(:get, /api.weather.com/)
        .to_timeout
    end

    it "handles timeout gracefully" do
      result = WeatherService.new.current_weather("Bangkok")
      expect(result).to be_nil
    end
  end
end
```

### VCR gem (Record and Replay HTTP)

```ruby
# Gemfile
gem "vcr", group: :test

# spec/support/vcr.rb
VCR.configure do |config|
  config.cassette_library_dir = "spec/vcr_cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!
  config.filter_sensitive_data("<API_KEY>") { ENV["WEATHER_API_KEY"] }
end

# ใช้ใน spec
RSpec.describe WeatherService, vcr: { cassette_name: "weather/bangkok" } do
  it "fetches weather" do
    result = WeatherService.new.current_weather("Bangkok")
    expect(result).to be_present
  end
end
```

---

## ขั้นตอนที่ 998: Code Coverage ด้วย SimpleCov

### Setup SimpleCov

```ruby
# Gemfile
gem "simplecov", require: false, group: :test

# spec/spec_helper.rb (ต้องอยู่ด้านบนสุด!)
require "simplecov"
SimpleCov.start "rails" do
  add_filter "/bin/"
  add_filter "/db/"
  add_filter "/spec/"
  add_filter "/config/"

  add_group "Models", "app/models"
  add_group "Controllers", "app/controllers"
  add_group "Services", "app/services"
  add_group "Helpers", "app/helpers"
  add_group "Mailers", "app/mailers"
  add_group "Jobs", "app/jobs"

  minimum_coverage 80  # ต้องมี coverage อย่างน้อย 80%
  maximum_coverage_drop 5  # ลดได้ไม่เกิน 5% จาก build ล่าสุด
end
```

### ดู Coverage Report

```bash
# รัน specs
bundle exec rspec

# ดู report ใน browser
open coverage/index.html
```

### Coverage Badge

```
# สร้าง badge สำหรับ README
SimpleCov.formatter = SimpleCov::Formatter::HTMLFormatter
# หรือ
require "simplecov-badge"
SimpleCov.formatter = SimpleCov::Formatter::BadgeFormatter
```

---

## ขั้นตอนที่ 999: Database Cleaner

### Setup Database Cleaner

```ruby
# Gemfile
gem "database_cleaner-active_record", group: :test

# spec/support/database_cleaner.rb
RSpec.configure do |config|
  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end

  # System specs ต้องใช้ truncation เพราะ browser กับ server คนละ thread
  config.before(:each, type: :system) do
    DatabaseCleaner.strategy = :truncation
  end

  config.after(:each, type: :system) do
    DatabaseCleaner.strategy = :transaction
  end
end
```

---

## ขั้นตอนที่ 1000: CI Setup (GitHub Actions)

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Rails Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: myapp_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost/myapp_test
      REDIS_URL: redis://localhost:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.2"
          bundler-cache: true

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"

      - name: Install dependencies
        run: |
          bundle install --jobs 4 --retry 3
          yarn install --frozen-lockfile

      - name: Setup database
        run: |
          bundle exec rails db:create
          bundle exec rails db:schema:load

      - name: Run RSpec
        run: bundle exec rspec --format documentation --format RspecJunitFormatter --out tmp/rspec.xml

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: tmp/rspec.xml

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage
          path: coverage/

  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.2"
          bundler-cache: true

      - name: Run RuboCop
        run: bundle exec rubocop --format progress
```

### Parallel Testing

```ruby
# Gemfile
gem "parallel_tests", group: :test

# รัน specs แบบ parallel
bundle exec parallel_rspec spec/

# config/database.yml (สำหรับ parallel testing)
test:
  database: myapp_test<%= ENV["TEST_ENV_NUMBER"] %>
```

---

## ขั้นตอนที่ 1001: Testing Best Practices

### Test Organization

```ruby
# ✅ ดี: Describe + Context + It
RSpec.describe User, type: :model do
  describe "#full_name" do
    context "when both names present" do
      it "returns combined name" do
        user = build(:user, first_name: "John", last_name: "Doe")
        expect(user.full_name).to eq("John Doe")
      end
    end

    context "when last name is missing" do
      it "returns only first name" do
        user = build(:user, first_name: "John", last_name: nil)
        expect(user.full_name).to eq("John")
      end
    end
  end
end

# ✅ ดี: One assertion per test
it "creates a user" do
  expect { create(:user) }.to change(User, :count).by(1)
end

# ❌ ไม่ดี: หลาย assertions ที่ไม่เกี่ยวกัน
it "does everything" do
  user = create(:user)
  expect(user.name).to be_present
  expect(user.email).to include("@")
  expect(user).to be_active
  # ถ้า test แรก fail จะไม่รู้ว่าอื่นๆ pass หรือ fail
end
```

### Shared Examples

```ruby
# spec/support/shared_examples/timestampable.rb
RSpec.shared_examples "timestampable" do
  it "has created_at" do
    expect(subject).to respond_to(:created_at)
  end

  it "has updated_at" do
    expect(subject).to respond_to(:updated_at)
  end

  it "sets created_at on create" do
    expect(subject.created_at).to be_present
  end
end

# ใช้ใน spec
RSpec.describe Post, type: :model do
  subject { create(:post) }
  it_behaves_like "timestampable"
end

RSpec.describe User, type: :model do
  subject { create(:user) }
  it_behaves_like "timestampable"
end
```

### Custom Matchers

```ruby
# spec/support/matchers/be_a_valid_email.rb
RSpec::Matchers.define :be_a_valid_email do
  match do |actual|
    actual =~ /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  end

  failure_message do |actual|
    "expected #{actual} to be a valid email address"
  end
end

# ใช้
expect(user.email).to be_a_valid_email
```

---

## ขั้นตอนที่ 1002: Testing Email

### Mailer Specs

```ruby
# spec/mailers/user_mailer_spec.rb
require "rails_helper"

RSpec.describe UserMailer, type: :mailer do
  describe "#welcome_email" do
    let(:user) { create(:user, name: "สมชาย", email: "somchai@example.com") }
    let(:mail) { UserMailer.with(user: user).welcome_email }

    it "renders the headers" do
      expect(mail.subject).to eq("ยินดีต้อนรับ #{user.name}")
      expect(mail.to).to eq([user.email])
      expect(mail.from).to eq(["noreply@myapp.com"])
    end

    it "renders the body" do
      expect(mail.body.encoded).to include(user.name)
      expect(mail.body.encoded).to include("ยินดีต้อนรับ")
    end
  end
end

# Testing email delivery in feature/system specs
it "sends welcome email on registration" do
  expect {
    post registrations_path, params: {
      user: { email: "new@example.com", password: "password" }
    }
  }.to change { ActionMailer::Base.deliveries.count }.by(1)

  email = ActionMailer::Base.deliveries.last
  expect(email.to).to include("new@example.com")
end
```

---

## ขั้นตอนที่ 1003: Testing Background Jobs

### Job Specs

```ruby
# spec/jobs/send_digest_email_job_spec.rb
require "rails_helper"

RSpec.describe SendDigestEmailJob, type: :job do
  let(:user) { create(:user) }

  it "queues the job" do
    expect {
      SendDigestEmailJob.perform_later(user.id)
    }.to have_enqueued_job(SendDigestEmailJob).with(user.id)
  end

  it "performs the job" do
    expect(DigestMailer).to receive(:digest_email)
      .with(user: user)
      .and_return(double(deliver_later: nil))

    SendDigestEmailJob.perform_now(user.id)
  end

  it "enqueues to correct queue" do
    expect {
      SendDigestEmailJob.perform_later(user.id)
    }.to have_enqueued_job.on_queue("mailers")
  end
end
```

---

## แบบฝึกหัดตอนที่ 45 (25 ข้อ)

### ระดับพื้นฐาน

**ข้อ 1:** ติดตั้ง RSpec, Factory Bot, Shoulda Matchers และ configure spec/rails_helper.rb

**ข้อ 2:** สร้าง factory สำหรับ User ที่มี traits: admin, inactive, with_posts

**ข้อ 3:** เขียน model spec สำหรับ User ที่ test: presence validations, email format, uniqueness

**ข้อ 4:** เขียน model spec สำหรับ Post ที่ test: associations, scopes, callbacks

**ข้อ 5:** ใช้ Shoulda Matchers ทดสอบ associations และ validations

**ข้อ 6:** เขียน request spec สำหรับ GET /posts ที่ test: status code, template, content

**ข้อ 7:** เขียน request spec สำหรับ POST /posts ที่ test: create success, create failure, redirect

**ข้อ 8:** เขียน request spec ที่ test authentication (redirect ถ้ายังไม่ login)

**ข้อ 9:** ใช้ `freeze_time` ทดสอบ logic ที่เกี่ยวกับเวลา

**ข้อ 10:** Setup Database Cleaner สำหรับ clean database ระหว่าง tests

### ระดับกลาง

**ข้อ 11:** เขียน system spec ด้วย Capybara สำหรับ post creation flow

**ข้อ 12:** เขียน system spec ที่ใช้ JS (js: true) สำหรับ dynamic feature

**ข้อ 13:** Setup WebMock และเขียน spec สำหรับ external API service

**ข้อ 14:** Setup SimpleCov และ achieve 80% code coverage

**ข้อ 15:** สร้าง shared examples สำหรับ timestampable behavior

**ข้อ 16:** เขียน custom Capybara matcher

**ข้อ 17:** เขียน mailer spec ที่ test headers และ body content

**ข้อ 18:** เขียน job spec ที่ test enqueuing และ performing

**ข้อ 19:** Setup GitHub Actions CI pipeline สำหรับ Rails app

**ข้อ 20:** ใช้ `travel_to` ทดสอบ token expiration logic

### ระดับสูง

**ข้อ 21:** Implement parallel testing ด้วย parallel_tests gem

**ข้อ 22:** สร้าง custom RSpec matcher สำหรับ business logic

**ข้อ 23:** เขียน performance spec ที่ test query count (N+1 detection)

**ข้อ 24:** Setup VCR gem เพื่อ record และ replay HTTP interactions

**ข้อ 25:** เขียน spec suite ที่ครอบคลุม authentication flow ตั้งแต่ register ถึง login ถึง logout

---

## สรุปตอนที่ 45

| เครื่องมือ | วัตถุประสงค์ |
|-----------|------------|
| RSpec | Testing framework |
| Factory Bot | Test data generation |
| Faker | Random data |
| Shoulda Matchers | Concise matchers |
| Capybara | Browser/system tests |
| Selenium | JavaScript support |
| WebMock | HTTP request mocking |
| VCR | Record/replay HTTP |
| SimpleCov | Code coverage |
| DatabaseCleaner | Clean test DB |
| parallel_tests | Faster test suite |

### Testing Pyramid

```
       /\
      /  \
     /    \          Unit Tests (Model Specs)
    /------\
   /        \        Integration Tests (Request Specs)
  /          \
 /------------\
/              \     End-to-End Tests (System Specs)
```

**Best Practices:**
1. เขียน tests ก่อน code (TDD/BDD)
2. ทุก feature ต้องมี test
3. Test ต้องรันเร็ว (<5 นาที สำหรับ suite เต็ม)
4. ไม่ test implementation detail แค่ behavior
5. เขียน tests ที่อ่านเข้าใจง่าย (describe/context/it)
6. ใช้ factory traits แทนการ create หลาย factories
7. Mock external services ด้วย WebMock
8. ตั้ง CI ให้รัน tests ทุก push

ตอนถัดไป: **ตอนที่ 46** - Deployment
