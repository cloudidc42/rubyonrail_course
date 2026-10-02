# Part 45: Testing ใน Rails

## ขั้นตอนที่ 991-1010: การทดสอบ Rails Application

---

## ขั้นตอนที่ 991: ทำไมต้องเขียน Tests?

Tests ช่วยให้:
- **Confidence:** มั่นใจว่า code ทำงานถูกต้อง
- **Refactoring:** แก้ไข code ได้อย่างมั่นใจ
- **Documentation:** Tests เป็นเอกสารที่ executable
- **Design:** เขียน tests ก่อนช่วยออกแบบ API ที่ดีกว่า
- **Regression:** จับ bugs ที่เกิดขึ้นซ้ำได้

```ruby
# โครงสร้าง test ใน Rails
test/
├── controllers/
│   ├── posts_controller_test.rb
│   └── users_controller_test.rb
├── fixtures/
│   ├── posts.yml
│   └── users.yml
├── helpers/
├── integration/
│   └── user_flows_test.rb
├── mailers/
│   └── user_mailer_test.rb
├── models/
│   ├── post_test.rb
│   └── user_test.rb
├── system/
│   └── user_sign_up_test.rb
└── test_helper.rb
```

## ขั้นตอนที่ 992: Minitest (Default Rails Testing)

Rails ใช้ Minitest เป็น default testing framework

```ruby
# test/test_helper.rb
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rails/test_help"

class ActiveSupport::TestCase
  # Run tests in parallel
  parallelize(workers: :number_of_processors)
  
  # Setup all fixtures
  fixtures :all
  
  # Helper methods สำหรับ tests
  def log_in_as(user, password: 'password', remember_me: '0')
    post login_path, params: { 
      session: { 
        email: user.email,
        password: password, 
        remember_me: remember_me 
      }
    }
  end
end
```

```ruby
# test/models/user_test.rb
require "test_helper"

class UserTest < ActiveSupport::TestCase
  def setup
    @user = User.new(
      name: "ทดสอบ ผู้ใช้",
      email: "test@example.com",
      password: "password123",
      password_confirmation: "password123"
    )
  end
  
  test "ควรถูกต้อง" do
    assert @user.valid?
  end
  
  test "ชื่อต้องมีค่า" do
    @user.name = ""
    assert_not @user.valid?
    assert_includes @user.errors[:name], "ไม่สามารถเว้นว่างได้"
  end
  
  test "อีเมลต้องมีค่า" do
    @user.email = "     "
    assert_not @user.valid?
  end
  
  test "ชื่อต้องไม่เกิน 50 ตัวอักษร" do
    @user.name = "ก" * 51
    assert_not @user.valid?
  end
  
  test "อีเมลต้องมีรูปแบบที่ถูกต้อง" do
    valid_emails = %w[user@example.com USER@foo.COM A_US-ER@foo.bar.org
                      first.last@foo.jp alice+bob@baz.cn]
    valid_emails.each do |valid_email|
      @user.email = valid_email
      assert @user.valid?, "#{valid_email} ควรถูกต้อง"
    end
    
    invalid_emails = %w[user@example,com user_at_foo.org user.name@example.
                         foo@bar_baz.com foo@bar+baz.com]
    invalid_emails.each do |invalid_email|
      @user.email = invalid_email
      assert_not @user.valid?, "#{invalid_email} ควรไม่ถูกต้อง"
    end
  end
  
  test "อีเมลต้องไม่ซ้ำกัน" do
    duplicate_user = @user.dup
    @user.save
    assert_not duplicate_user.valid?
  end
  
  test "อีเมลถูก downcase ก่อนบันทึก" do
    mixed_case_email = "Foo@ExAMPle.CoM"
    @user.email = mixed_case_email
    @user.save
    assert_equal mixed_case_email.downcase, @user.reload.email
  end
  
  test "รหัสผ่านต้องมีค่า" do
    @user.password = @user.password_confirmation = " " * 8
    assert_not @user.valid?
  end
  
  test "รหัสผ่านต้องมีความยาวอย่างน้อย 8 ตัวอักษร" do
    @user.password = @user.password_confirmation = "a" * 7
    assert_not @user.valid?
  end
end
```

## ขั้นตอนที่ 993: RSpec - การติดตั้ง

RSpec เป็น testing framework ที่นิยมมากกว่า Minitest

```ruby
# Gemfile
group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'shoulda-matchers'
  gem 'database_cleaner-active_record'
end

group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
  gem 'webdrivers'
end
```

```bash
bundle install
rails generate rspec:install
```

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end
  
  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end
  
  config.shared_context_metadata_behavior = :apply_to_host_groups
  config.filter_run_when_matching :focus
  config.disable_monkey_patching!
  config.order = :random
  Kernel.srand config.seed
end
```

```ruby
# spec/rails_helper.rb
require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require_relative '../config/environment'
require 'rspec/rails'
require 'capybara/rails'

Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }

ActiveRecord::Migration.maintain_test_schema!

RSpec.configure do |config|
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
  config.use_transactional_fixtures = true
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!
  
  config.include FactoryBot::Syntax::Methods
  config.include Devise::Test::IntegrationHelpers, type: :request
  config.include Devise::Test::ControllerHelpers, type: :controller
end
```

## ขั้นตอนที่ 994: Model Tests ด้วย RSpec

```ruby
# spec/models/post_spec.rb
require 'rails_helper'

RSpec.describe Post, type: :model do
  # Shoulda Matchers
  describe "associations" do
    it { should belong_to(:user) }
    it { should have_many(:comments).dependent(:destroy) }
    it { should have_many(:tags).through(:taggings) }
    it { should have_one_attached(:image) }
  end
  
  describe "validations" do
    it { should validate_presence_of(:title) }
    it { should validate_presence_of(:content) }
    it { should validate_length_of(:title).is_at_most(200) }
    it { should validate_uniqueness_of(:slug).case_insensitive }
  end
  
  # Custom tests
  describe "scopes" do
    let!(:published_posts) { create_list(:post, 3, published: true) }
    let!(:draft_posts) { create_list(:post, 2, published: false) }
    
    describe ".published" do
      it "returns only published posts" do
        expect(Post.published).to match_array(published_posts)
      end
      
      it "does not include draft posts" do
        expect(Post.published).not_to include(*draft_posts)
      end
    end
    
    describe ".recent" do
      it "orders by created_at descending" do
        expect(Post.recent.first).to eq(Post.order(created_at: :desc).first)
      end
    end
    
    describe ".featured" do
      let!(:featured) { create_list(:post, 2, :featured) }
      
      it "returns only featured posts" do
        expect(Post.featured).to match_array(featured)
      end
    end
  end
  
  describe "instance methods" do
    let(:post) { build(:post, title: "Hello World") }
    
    describe "#generate_slug" do
      it "generates slug from title" do
        post.save
        expect(post.slug).to eq("hello-world")
      end
      
      it "handles Thai characters" do
        post.title = "สวัสดี World"
        post.save
        expect(post.slug).to be_present
      end
      
      it "makes slug unique" do
        post.save
        duplicate = create(:post, title: "Hello World")
        expect(duplicate.slug).not_to eq(post.slug)
      end
    end
    
    describe "#reading_time" do
      it "calculates reading time correctly" do
        post.content = "word " * 300  # 300 words
        expect(post.reading_time).to eq(2)  # ~2 minutes
      end
    end
    
    describe "#publish!" do
      it "sets published to true" do
        expect { post.publish! }.to change { post.published }.to(true)
      end
      
      it "sets published_at" do
        expect { post.publish! }.to change { post.published_at }.from(nil)
      end
    end
  end
  
  describe "callbacks" do
    describe "before_save" do
      let(:post) { build(:post) }
      
      it "strips whitespace from title" do
        post.title = "  Hello World  "
        post.save
        expect(post.title).to eq("Hello World")
      end
    end
    
    describe "after_create" do
      it "creates initial version" do
        expect { create(:post) }.to change(PostVersion, :count).by(1)
      end
    end
  end
end
```

## ขั้นตอนที่ 995: Factory Bot

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { Faker::Name.name }
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "Password123!" }
    password_confirmation { "Password123!" }
    
    # Traits
    trait :admin do
      role { :admin }
    end
    
    trait :moderator do
      role { :moderator }
    end
    
    trait :confirmed do
      confirmed_at { Time.current }
    end
    
    trait :unconfirmed do
      confirmed_at { nil }
    end
    
    trait :locked do
      locked_at { Time.current }
      failed_attempts { 5 }
    end
    
    trait :with_avatar do
      after(:create) do |user|
        file = Rails.root.join("spec/fixtures/files/avatar.jpg")
        user.avatar.attach(
          io: File.open(file),
          filename: "avatar.jpg",
          content_type: "image/jpeg"
        )
      end
    end
    
    trait :with_posts do
      after(:create) do |user|
        create_list(:post, 3, :published, user: user)
      end
    end
    
    # กำหนดค่า default จาก trait
    factory :admin_user, traits: [:admin, :confirmed]
    factory :confirmed_user, traits: [:confirmed]
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { Faker::Lorem.sentence(word_count: 5) }
    content { Faker::Lorem.paragraphs(number: 3).join("\n\n") }
    published { false }
    association :user, factory: :confirmed_user
    
    trait :published do
      published { true }
      published_at { 1.day.ago }
    end
    
    trait :draft do
      published { false }
    end
    
    trait :featured do
      featured { true }
      published { true }
    end
    
    trait :with_image do
      after(:create) do |post|
        file = Rails.root.join("spec/fixtures/files/test_image.jpg")
        post.image.attach(
          io: File.open(file),
          filename: "test_image.jpg",
          content_type: "image/jpeg"
        )
      end
    end
    
    trait :with_comments do
      after(:create) do |post|
        create_list(:comment, 5, :approved, post: post)
      end
    end
  end
end
```

## ขั้นตอนที่ 996: Controller Tests

```ruby
# spec/controllers/posts_controller_spec.rb
require 'rails_helper'

RSpec.describe PostsController, type: :controller do
  let(:user) { create(:confirmed_user) }
  let(:admin) { create(:admin_user) }
  let(:post_record) { create(:post, :published, user: user) }
  
  describe "GET #index" do
    it "returns http success" do
      get :index
      expect(response).to have_http_status(:success)
    end
    
    it "assigns published posts" do
      published = create_list(:post, 3, :published)
      draft = create_list(:post, 2, :draft)
      
      get :index
      
      expect(assigns(:posts)).to match_array(published)
      expect(assigns(:posts)).not_to include(*draft)
    end
    
    it "paginates results" do
      create_list(:post, 30, :published)
      get :index
      expect(assigns(:posts).count).to be <= 20
    end
  end
  
  describe "GET #show" do
    context "published post" do
      it "returns http success" do
        get :show, params: { id: post_record }
        expect(response).to have_http_status(:success)
      end
      
      it "assigns post" do
        get :show, params: { id: post_record }
        expect(assigns(:post)).to eq(post_record)
      end
    end
    
    context "draft post" do
      let(:draft_post) { create(:post, :draft, user: user) }
      
      context "as guest" do
        it "redirects to posts" do
          get :show, params: { id: draft_post }
          expect(response).to redirect_to(posts_path)
        end
      end
      
      context "as owner" do
        before { sign_in user }
        
        it "returns success" do
          get :show, params: { id: draft_post }
          expect(response).to have_http_status(:success)
        end
      end
    end
  end
  
  describe "POST #create" do
    context "as guest" do
      it "redirects to login" do
        post :create, params: { post: attributes_for(:post) }
        expect(response).to redirect_to(new_user_session_path)
      end
    end
    
    context "as logged in user" do
      before { sign_in user }
      
      context "with valid params" do
        it "creates a new post" do
          expect {
            post :create, params: { post: attributes_for(:post) }
          }.to change(Post, :count).by(1)
        end
        
        it "redirects to the created post" do
          post :create, params: { post: attributes_for(:post) }
          expect(response).to redirect_to(Post.last)
        end
        
        it "sets the user" do
          post :create, params: { post: attributes_for(:post) }
          expect(Post.last.user).to eq(user)
        end
      end
      
      context "with invalid params" do
        it "does not create a post" do
          expect {
            post :create, params: { post: attributes_for(:post, title: "") }
          }.not_to change(Post, :count)
        end
        
        it "returns unprocessable entity" do
          post :create, params: { post: attributes_for(:post, title: "") }
          expect(response).to have_http_status(:unprocessable_entity)
        end
      end
    end
  end
  
  describe "DELETE #destroy" do
    let!(:post_to_delete) { create(:post, user: user) }
    
    context "as owner" do
      before { sign_in user }
      
      it "destroys the post" do
        expect {
          delete :destroy, params: { id: post_to_delete }
        }.to change(Post, :count).by(-1)
      end
      
      it "redirects to posts" do
        delete :destroy, params: { id: post_to_delete }
        expect(response).to redirect_to(posts_path)
      end
    end
    
    context "as other user" do
      before { sign_in create(:confirmed_user) }
      
      it "does not destroy the post" do
        expect {
          delete :destroy, params: { id: post_to_delete }
        }.not_to change(Post, :count)
      end
    end
  end
end
```

## ขั้นตอนที่ 997: Request Specs (Integration Tests)

```ruby
# spec/requests/posts_spec.rb
require 'rails_helper'

RSpec.describe "Posts", type: :request do
  let(:user) { create(:confirmed_user) }
  let(:admin) { create(:admin_user) }
  
  describe "GET /posts" do
    let!(:posts) { create_list(:post, 5, :published) }
    
    it "returns successful response" do
      get posts_path
      expect(response).to have_http_status(:ok)
    end
    
    it "includes all published posts" do
      get posts_path
      posts.each do |post|
        expect(response.body).to include(post.title)
      end
    end
    
    context "with search query" do
      let!(:target_post) { create(:post, :published, title: "Ruby on Rails Guide") }
      
      it "filters by search query" do
        get posts_path, params: { q: "Ruby" }
        expect(response.body).to include(target_post.title)
      end
    end
  end
  
  describe "GET /posts/:id" do
    let(:post) { create(:post, :published) }
    
    it "returns the post" do
      get post_path(post)
      expect(response).to have_http_status(:ok)
      expect(response.body).to include(post.title)
    end
    
    it "records view count" do
      expect { get post_path(post) }.to change { post.reload.views_count }.by(1)
    end
  end
  
  describe "POST /posts" do
    let(:valid_params) do
      { post: { title: "New Post", content: "Content here", category_id: create(:category).id } }
    end
    
    context "authenticated" do
      before { sign_in user }
      
      it "creates a post" do
        expect { post posts_path, params: valid_params }.to change(Post, :count).by(1)
        expect(response).to redirect_to(Post.last)
      end
      
      it "returns errors for invalid post" do
        post posts_path, params: { post: { title: "" } }
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
    
    context "unauthenticated" do
      it "redirects to login" do
        post posts_path, params: valid_params
        expect(response).to redirect_to(new_user_session_path)
      end
    end
  end
end
```

## ขั้นตอนที่ 998: System Tests (Capybara)

```ruby
# test/system/user_sign_up_test.rb (Minitest style)
require "application_system_test_case"

class UserSignUpTest < ApplicationSystemTestCase
  test "ลงทะเบียนผู้ใช้ใหม่สำเร็จ" do
    visit new_user_registration_path
    
    fill_in "ชื่อ", with: "สมชาย ใจดี"
    fill_in "อีเมล", with: "somchai@example.com"
    fill_in "รหัสผ่าน", with: "Password123!"
    fill_in "ยืนยันรหัสผ่าน", with: "Password123!"
    
    click_button "สร้างบัญชี"
    
    assert_text "ลงทะเบียนสำเร็จ"
    assert_current_path dashboard_path
  end
  
  test "แสดง error เมื่อกรอกข้อมูลไม่ถูกต้อง" do
    visit new_user_registration_path
    
    fill_in "อีเมล", with: "invalid-email"
    click_button "สร้างบัญชี"
    
    assert_text "รูปแบบอีเมลไม่ถูกต้อง"
  end
end
```

```ruby
# spec/system/posts_spec.rb (RSpec style)
require 'rails_helper'

RSpec.describe "Posts", type: :system do
  let(:user) { create(:confirmed_user) }
  
  before do
    driven_by(:selenium, using: :headless_chrome, screen_size: [1400, 900])
  end
  
  describe "สร้างบทความใหม่" do
    before { sign_in user }
    
    it "สร้างได้สำเร็จ" do
      visit new_post_path
      
      fill_in "หัวข้อ", with: "บทความทดสอบ"
      find('[data-testid="content-editor"]').set("เนื้อหาบทความ")
      select "ทั่วไป", from: "หมวดหมู่"
      
      click_button "บันทึก"
      
      expect(page).to have_text("สร้างบทความสำเร็จ")
      expect(page).to have_text("บทความทดสอบ")
    end
    
    it "แสดง errors เมื่อข้อมูลไม่ครบ" do
      visit new_post_path
      click_button "บันทึก"
      
      expect(page).to have_text("กรุณากรอกหัวข้อ")
    end
  end
  
  describe "ค้นหาบทความ" do
    let!(:ruby_post) { create(:post, :published, title: "Ruby on Rails Guide") }
    let!(:python_post) { create(:post, :published, title: "Python Tutorial") }
    
    it "แสดงผลการค้นหาที่ถูกต้อง" do
      visit posts_path
      
      fill_in "ค้นหา", with: "Ruby"
      click_button "ค้นหา"
      
      expect(page).to have_text("Ruby on Rails Guide")
      expect(page).not_to have_text("Python Tutorial")
    end
  end
  
  describe "Turbo Frame interaction" do
    it "แก้ไขบทความโดยไม่ reload หน้า" do
      post = create(:post, :published, user: user)
      sign_in user
      
      visit post_path(post)
      
      click_link "แก้ไข"
      
      # รอ Turbo Frame โหลด
      within("turbo-frame#post-form") do
        fill_in "หัวข้อ", with: "หัวข้อที่แก้ไข"
        click_button "บันทึก"
      end
      
      expect(page).to have_text("หัวข้อที่แก้ไข")
      expect(page).to have_text("อัพเดทบทความแล้ว")
    end
  end
end
```

## ขั้นตอนที่ 999: Fixtures

```yaml
# test/fixtures/users.yml
admin:
  name: ผู้ดูแลระบบ
  email: admin@example.com
  password_digest: <%= BCrypt::Password.create("password123") %>
  role: admin
  confirmed_at: <%= Time.current %>

regular_user:
  name: ผู้ใช้ทั่วไป
  email: user@example.com
  password_digest: <%= BCrypt::Password.create("password123") %>
  role: user
  confirmed_at: <%= Time.current %>
```

```yaml
# test/fixtures/posts.yml
published_post:
  title: บทความที่เผยแพร่แล้ว
  content: เนื้อหาของบทความ...
  published: true
  published_at: <%= 1.day.ago %>
  user: regular_user

draft_post:
  title: บทความร่าง
  content: เนื้อหาที่ยังไม่เผยแพร่
  published: false
  user: regular_user

admin_post:
  title: บทความของ Admin
  content: เนื้อหาจาก Admin
  published: true
  published_at: <%= 2.days.ago %>
  user: admin
```

## ขั้นตอนที่ 1000: Mailer Tests

```ruby
# spec/mailers/user_mailer_spec.rb
require "rails_helper"

RSpec.describe UserMailer, type: :mailer do
  let(:user) { create(:user) }
  
  describe "welcome" do
    let(:mail) { UserMailer.welcome(user) }
    
    it "renders the headers" do
      expect(mail.subject).to eq("ยินดีต้อนรับสู่ MyApp!")
      expect(mail.to).to eq([user.email])
      expect(mail.from).to eq(["noreply@myapp.com"])
    end
    
    it "renders the body" do
      expect(mail.body.encoded).to include(user.name)
      expect(mail.body.encoded).to include("ยินดีต้อนรับ")
    end
  end
  
  describe "password_reset" do
    let(:mail) { UserMailer.password_reset(user) }
    
    before { user.create_reset_digest }
    
    it "sends to user email" do
      expect(mail.to).to eq([user.email])
    end
    
    it "includes reset link" do
      expect(mail.body.encoded).to include("reset_password_token")
    end
    
    it "includes expiry information" do
      expect(mail.body.encoded).to include("6 ชั่วโมง")
    end
  end
  
  describe "delivery" do
    it "actually sends email" do
      expect {
        UserMailer.welcome(user).deliver_now
      }.to change { ActionMailer::Base.deliveries.count }.by(1)
    end
    
    it "sends email asynchronously" do
      expect {
        UserMailer.welcome(user).deliver_later
      }.to have_enqueued_mail(UserMailer, :welcome)
    end
  end
end
```

## ขั้นตอนที่ 1001: WebMock and VCR

```ruby
# Gemfile
group :test do
  gem 'webmock'
  gem 'vcr'
end
```

```ruby
# spec/support/vcr.rb
VCR.configure do |config|
  config.cassette_library_dir = "spec/vcr_cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!
  config.filter_sensitive_data('<API_KEY>') { ENV['EXTERNAL_API_KEY'] }
end
```

```ruby
# spec/services/weather_service_spec.rb
require 'rails_helper'

RSpec.describe WeatherService do
  describe "#current_weather" do
    context "สภาพอากาศปกติ" do
      it "returns weather data", :vcr do
        service = WeatherService.new("Bangkok")
        result = service.current_weather
        
        expect(result[:temperature]).to be_a(Numeric)
        expect(result[:description]).to be_a(String)
      end
    end
    
    context "เมื่อ API ล้มเหลว" do
      before do
        stub_request(:get, /api.openweathermap.org/)
          .to_return(status: 503, body: "Service Unavailable")
      end
      
      it "raises ServiceError" do
        service = WeatherService.new("Bangkok")
        expect { service.current_weather }.to raise_error(WeatherService::ServiceError)
      end
    end
    
    context "เมื่อ network timeout" do
      before do
        stub_request(:get, /api.openweathermap.org/)
          .to_timeout
      end
      
      it "handles timeout gracefully" do
        service = WeatherService.new("Bangkok")
        result = service.current_weather
        expect(result).to be_nil
      end
    end
  end
end
```

## ขั้นตอนที่ 1002: Code Coverage ด้วย SimpleCov

```ruby
# Gemfile
group :test do
  gem 'simplecov'
  gem 'simplecov-console'
end
```

```ruby
# spec/spec_helper.rb (ต้องอยู่บนสุด)
require 'simplecov'
require 'simplecov-console'

SimpleCov.start 'rails' do
  add_filter '/bin/'
  add_filter '/db/'
  add_filter '/spec/'
  add_filter '/test/'
  add_filter '/config/'
  add_filter '/vendor/'
  
  add_group 'Controllers', 'app/controllers'
  add_group 'Models', 'app/models'
  add_group 'Services', 'app/services'
  add_group 'Helpers', 'app/helpers'
  add_group 'Mailers', 'app/mailers'
  add_group 'Jobs', 'app/jobs'
  
  minimum_coverage 90
  maximum_coverage_drop 5
end

SimpleCov.formatters = [
  SimpleCov::Formatter::HTMLFormatter,
  SimpleCov::Formatter::Console
]
```

## ขั้นตอนที่ 1003: Testing with Database Cleaner

```ruby
# spec/support/database_cleaner.rb
RSpec.configure do |config|
  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end
  
  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
  end
  
  config.before(:each, :js => true) do
    DatabaseCleaner.strategy = :truncation
  end
  
  config.before(:each) do
    DatabaseCleaner.start
  end
  
  config.after(:each) do
    DatabaseCleaner.clean
  end
end
```

## ขั้นตอนที่ 1004: Shared Examples

```ruby
# spec/support/shared_examples/authenticatable.rb
RSpec.shared_examples "requires authentication" do
  context "ไม่ได้ login" do
    it "redirects to login page" do
      subject
      expect(response).to redirect_to(new_user_session_path)
    end
  end
end

RSpec.shared_examples "requires authorization" do |action|
  context "ไม่มีสิทธิ์" do
    before { sign_in create(:confirmed_user) }
    
    it "redirects with alert" do
      subject
      expect(flash[:alert]).to be_present
      expect(response).to redirect_to(root_path)
    end
  end
end
```

```ruby
# ใช้งาน shared examples
RSpec.describe Admin::PostsController do
  describe "GET #index" do
    subject { get :index }
    
    it_behaves_like "requires authentication"
    it_behaves_like "requires authorization"
  end
end
```

## ขั้นตอนที่ 1005: Custom Matchers

```ruby
# spec/support/matchers/custom_matchers.rb
RSpec::Matchers.define :be_valid_email do
  match do |actual|
    actual.match?(/\A[^@\s]+@[^@\s]+\.[^@\s]+\z/)
  end
  
  failure_message do |actual|
    "ควรเป็นอีเมลที่ถูกต้อง แต่ได้ '#{actual}'"
  end
end

RSpec::Matchers.define :have_flash_message do |type, message|
  match do |page|
    page.has_css?(".alert-#{type}", text: message)
  end
  
  failure_message do
    "ควรมี flash message ประเภท #{type} ว่า '#{message}'"
  end
end
```

```ruby
# ใช้งาน custom matchers
it "validates email format" do
  user.email = "invalid"
  expect(user.email).not_to be_valid_email
end

it "shows success message" do
  visit new_post_path
  fill_in_and_submit_valid_form
  expect(page).to have_flash_message(:success, "บันทึกสำเร็จ")
end
```

## ขั้นตอนที่ 1006: Test Helpers

```ruby
# spec/support/helpers/authentication_helpers.rb
module AuthenticationHelpers
  def sign_in_as(user, password: "Password123!")
    visit new_user_session_path
    fill_in "อีเมล", with: user.email
    fill_in "รหัสผ่าน", with: password
    click_button "เข้าสู่ระบบ"
  end
  
  def sign_out
    click_link "ออกจากระบบ"
  end
end

# spec/support/helpers/form_helpers.rb
module FormHelpers
  def fill_in_post_form(title: "Test Post", content: "Test content")
    fill_in "หัวข้อ", with: title
    fill_in "เนื้อหา", with: content
  end
  
  def submit_form(button_text = "บันทึก")
    click_button button_text
  end
end
```

## ขั้นตอนที่ 1007: CI Testing (GitHub Actions)

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
        image: postgres:14
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
      DATABASE_URL: postgresql://postgres:postgres@localhost/myapp_test
      REDIS_URL: redis://localhost:6379/0
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
      
      - name: Set up Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'yarn'
      
      - name: Install dependencies
        run: |
          bundle install
          yarn install
      
      - name: Setup database
        run: |
          bundle exec rails db:create
          bundle exec rails db:schema:load
      
      - name: Precompile assets
        run: bundle exec rails assets:precompile
      
      - name: Run tests
        run: bundle exec rspec --format progress --format RspecJunitFormatter --out tmp/rspec_results.xml
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: tmp/rspec_results.xml
      
      - name: Upload coverage report
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/coverage.xml
```

## ขั้นตอนที่ 1008: Performance Testing

```ruby
# spec/performance/post_spec.rb
require 'rails_helper'

RSpec.describe "Post Performance", type: :performance do
  let!(:posts) { create_list(:post, 100, :published, :with_comments) }
  
  it "loads posts list quickly" do
    # ใช้ benchmark
    time = Benchmark.realtime do
      get posts_path
    end
    
    expect(time).to be < 0.5  # ต้องโหลดภายใน 500ms
  end
  
  it "does not cause N+1 queries" do
    queries_count = 0
    ActiveSupport::Notifications.subscribe("sql.active_record") do
      queries_count += 1
    end
    
    get posts_path
    
    expect(queries_count).to be < 5  # ไม่เกิน 5 queries
  end
end
```

## ขั้นตอนที่ 1009: Testing Background Jobs

```ruby
# spec/jobs/email_notification_job_spec.rb
require 'rails_helper'

RSpec.describe EmailNotificationJob, type: :job do
  include ActiveJob::TestHelper
  
  let(:user) { create(:user) }
  let(:post) { create(:post, :published) }
  
  it "เพิ่มเข้า queue" do
    expect {
      EmailNotificationJob.perform_later(user.id, post.id)
    }.to have_enqueued_job(EmailNotificationJob)
      .with(user.id, post.id)
      .on_queue('default')
  end
  
  it "ส่ง email เมื่อ execute" do
    expect {
      perform_enqueued_jobs do
        EmailNotificationJob.perform_later(user.id, post.id)
      end
    }.to change { ActionMailer::Base.deliveries.count }.by(1)
  end
  
  it "retry เมื่อเกิด error" do
    allow(UserMailer).to receive(:notification).and_raise(StandardError)
    
    expect {
      perform_enqueued_jobs do
        EmailNotificationJob.perform_later(user.id, post.id)
      end
    }.to raise_error(StandardError)
  end
end
```

## ขั้นตอนที่ 1010: Testing Tips และ Best Practices

```ruby
# 1. ทดสอบ behavior ไม่ใช่ implementation
# BAD: ทดสอบว่าเรียก method ชื่ออะไร
it "calls send_email method" do
  expect(user).to receive(:send_email)
  user.signup!
end

# GOOD: ทดสอบว่า email ถูกส่ง
it "sends welcome email" do
  expect { user.signup! }.to change { ActionMailer::Base.deliveries.count }.by(1)
end

# 2. One assertion per test (generally)
it "creates user with correct attributes" do
  user = create(:user, name: "สมชาย")
  expect(user.name).to eq("สมชาย")
end

# 3. ใช้ let ไม่ใช่ before each สำหรับ data setup
let(:user) { create(:user) }  # ดีกว่า @user = ...

# 4. ใช้ subject สำหรับ object ที่กำลังทดสอบ
subject { described_class.new(email: "test@example.com") }

# 5. ใช้ described_class ไม่ใช่ class name
RSpec.describe User do
  subject { described_class.new }  # ดีกว่า User.new
end

# 6. Four-phase test structure
it "updates post title" do
  # Setup
  post = create(:post, title: "Original")
  
  # Exercise
  post.update(title: "Updated")
  
  # Verify
  expect(post.title).to eq("Updated")
  
  # Teardown (usually automatic)
end
```

---

## แบบฝึกหัด: Testing in Rails (25 ข้อ)

### ข้อที่ 1: ติดตั้ง RSpec
```bash
bundle add rspec-rails --group development,test
rails generate rspec:install
```

### ข้อที่ 2: Model Test พื้นฐาน
```
เขียน tests สำหรับ User model:
- validates presence of name และ email
- validates uniqueness of email
- validates password length
```

**เฉลย:**
```ruby
RSpec.describe User, type: :model do
  it { should validate_presence_of(:name) }
  it { should validate_presence_of(:email) }
  it { should validate_uniqueness_of(:email) }
  it { should validate_length_of(:password).is_at_least(8) }
end
```

### ข้อที่ 3: FactoryBot Setup
```
สร้าง factory สำหรับ User ที่มี traits: :admin, :confirmed, :with_posts
```

### ข้อที่ 4: Controller Test
```
ทดสอบ PostsController#create:
- guest redirected ไป login
- valid params สร้าง post สำเร็จ
- invalid params แสดง errors
```

### ข้อที่ 5: Request Spec
```
ทดสอบ API endpoint GET /api/v1/posts
ที่ส่งคืน JSON ที่มี posts array
```

**เฉลย:**
```ruby
RSpec.describe "Api::V1::Posts", type: :request do
  let!(:posts) { create_list(:post, 3, :published) }
  
  it "returns posts as JSON" do
    get "/api/v1/posts",
        headers: { "Authorization" => "Bearer #{jwt_token}" }
    
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)["posts"].length).to eq(3)
  end
end
```

### ข้อที่ 6-25 (แบบสรุป)

**ข้อ 6:** System test สำหรับ user registration flow

**ข้อ 7:** Test mailer สำหรับ password reset

**ข้อ 8:** Test ด้วย WebMock สำหรับ external API

**ข้อ 9:** เพิ่ม SimpleCov ให้ถึง 90% coverage

**ข้อ 10:** Test background job

**ข้อ 11:** เขียน shared examples สำหรับ authentication

**ข้อ 12:** Test scope methods

**ข้อ 13:** Test service object

**ข้อ 14:** Test ด้วย VCR cassettes

**ข้อ 15:** Setup GitHub Actions CI

**ข้อ 16:** Test pagination

**ข้อ 17:** Custom matchers

**ข้อ 18:** Test file uploads

**ข้อ 19:** Test Turbo Streams

**ข้อ 20:** Test JavaScript behavior ด้วย Capybara

**ข้อ 21:** Test authorization policies

**ข้อ 22:** Performance test - ตรวจจับ N+1

**ข้อ 23:** Test internationalization

**ข้อ 24:** Test caching behavior

**ข้อ 25:** Test complete user journey

```ruby
# ข้อ 23: Test I18n
RSpec.describe Post, type: :model do
  describe "error messages" do
    context "ภาษาไทย" do
      around do |example|
        I18n.with_locale(:th) { example.run }
      end
      
      it "shows Thai error messages" do
        post = Post.new(title: "")
        post.valid?
        expect(post.errors[:title]).to include("ไม่สามารถเว้นว่างได้")
      end
    end
  end
end
```

---

## สรุป: Testing in Rails

| Type | Tool | ใช้เมื่อ |
|------|------|---------|
| Model tests | RSpec/Minitest | ทดสอบ validations, scopes, methods |
| Controller tests | RSpec | ทดสอบ HTTP responses |
| Request specs | RSpec | ทดสอบ full request/response cycle |
| System tests | Capybara | ทดสอบ browser interactions |
| Mailer tests | RSpec | ทดสอบ email content |
| Job tests | ActiveJob::TestHelper | ทดสอบ background jobs |
| API tests | RSpec | ทดสอบ JSON API |

**Key Takeaways:**
1. เขียน tests เป็น habit ไม่ใช่ afterthought
2. ใช้ Factory Bot แทน Fixtures สำหรับ complex data
3. System tests ช้ากว่าแต่ครอบคลุมกว่า
4. ตั้งเป้า code coverage อย่างน้อย 80-90%
5. รัน tests ใน CI ทุก pull request
