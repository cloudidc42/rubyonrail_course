# Part 69: Advanced Testing

## Steps 1496-1520

---

## Step 1496: Test-Driven Development (TDD) Workflow

TDD คือ methodology ที่เขียน test ก่อน แล้วค่อยเขียน code ให้ test ผ่าน

### Red-Green-Refactor Cycle

```
🔴 RED:   เขียน test ที่ fail
🟢 GREEN: เขียน code ให้ test pass
🔵 REFACTOR: ปรับปรุง code โดยไม่ให้ test fail
```

### ตัวอย่าง TDD กับ Cart Feature

```ruby
# 1. เขียน test ก่อน (Red)
# spec/models/cart_spec.rb
RSpec.describe Cart, type: :model do
  describe '#add_item' do
    let(:cart) { Cart.new }
    let(:product) { build(:product, price: 100) }
    
    it 'adds a product to the cart' do
      cart.add_item(product, quantity: 1)
      expect(cart.items.count).to eq(1)
    end
    
    it 'increases quantity if product already in cart' do
      cart.add_item(product, quantity: 1)
      cart.add_item(product, quantity: 2)
      
      item = cart.items.first
      expect(item.quantity).to eq(3)
    end
    
    it 'calculates total correctly' do
      cart.add_item(product, quantity: 3)
      expect(cart.total).to eq(300)
    end
    
    it 'raises error for out of stock products' do
      out_of_stock = build(:product, stock: 0)
      
      expect { cart.add_item(out_of_stock, quantity: 1) }
        .to raise_error(Cart::OutOfStockError)
    end
  end
  
  describe '#remove_item' do
    let(:cart) { Cart.new }
    let(:product) { build(:product) }
    
    before { cart.add_item(product, quantity: 2) }
    
    it 'removes item from cart' do
      cart.remove_item(product)
      expect(cart.items).to be_empty
    end
    
    it 'returns nil for product not in cart' do
      other_product = build(:product)
      expect(cart.remove_item(other_product)).to be_nil
    end
  end
  
  describe '#apply_coupon' do
    let(:cart) { Cart.new }
    let(:product) { build(:product, price: 1000) }
    
    before { cart.add_item(product, quantity: 1) }
    
    it 'applies percentage discount' do
      coupon = build(:coupon, discount_type: 'percentage', discount_value: 10)
      cart.apply_coupon(coupon)
      
      expect(cart.discount).to eq(100)
      expect(cart.total_after_discount).to eq(900)
    end
    
    it 'applies fixed discount' do
      coupon = build(:coupon, discount_type: 'fixed', discount_value: 200)
      cart.apply_coupon(coupon)
      
      expect(cart.discount).to eq(200)
    end
    
    it 'does not apply invalid coupon' do
      expired_coupon = build(:coupon, expires_at: 1.day.ago)
      
      expect { cart.apply_coupon(expired_coupon) }
        .to raise_error(Cart::InvalidCouponError)
    end
  end
end

# 2. เขียน Cart model (Green)
# app/models/cart.rb
class Cart
  class OutOfStockError < StandardError; end
  class InvalidCouponError < StandardError; end
  
  CartItem = Struct.new(:product, :quantity, :unit_price) do
    def total
      quantity * unit_price
    end
  end
  
  attr_reader :items, :coupon
  
  def initialize
    @items = []
    @coupon = nil
  end
  
  def add_item(product, quantity: 1)
    raise OutOfStockError, "#{product.name} หมดสต็อก" if product.stock.zero?
    
    existing_item = items.find { |i| i.product == product }
    
    if existing_item
      existing_item.quantity += quantity
    else
      @items << CartItem.new(product, quantity, product.price)
    end
  end
  
  def remove_item(product)
    item = items.find { |i| i.product == product }
    return nil unless item
    
    @items.delete(item)
  end
  
  def apply_coupon(coupon)
    raise InvalidCouponError, 'Coupon หมดอายุ' if coupon.expired?
    raise InvalidCouponError, 'Coupon ใช้งานไม่ได้' unless coupon.active?
    
    @coupon = coupon
  end
  
  def subtotal
    items.sum(&:total)
  end
  
  def discount
    return 0 unless coupon
    
    case coupon.discount_type
    when 'percentage'
      (subtotal * coupon.discount_value / 100).round(2)
    when 'fixed'
      [coupon.discount_value, subtotal].min
    else
      0
    end
  end
  
  def total
    subtotal
  end
  
  def total_after_discount
    subtotal - discount
  end
end
```

---

## Step 1497: BDD กับ RSpec

BDD เน้น behavior ของระบบจากมุมมองของ stakeholders

### RSpec Setup

```ruby
# Gemfile (development, test group)
group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'shoulda-matchers'
  gem 'database_cleaner-active_record'
end

# Terminal
rails generate rspec:install

# spec/rails_helper.rb
require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require_relative '../config/environment'

require 'rspec/rails'
require 'shoulda/matchers'

Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }

RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
  config.include Devise::Test::IntegrationHelpers, type: :request
  
  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end
  
  config.around(:each) do |example|
    DatabaseCleaner.cleaning { example.run }
  end
  
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!
end

Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

### Factory Bot Factories

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { Faker::Name.name }
    email { Faker::Internet.unique.email }
    password { 'password123' }
    role { 'user' }
    active { true }
    
    trait :admin do
      role { 'admin' }
    end
    
    trait :inactive do
      active { false }
    end
    
    trait :with_orders do
      after(:create) do |user|
        create_list(:order, 3, user: user)
      end
    end
    
    trait :with_profile do
      after(:create) do |user|
        create(:profile, user: user)
      end
    end
  end
end

# spec/factories/products.rb
FactoryBot.define do
  factory :product do
    name { Faker::Commerce.product_name }
    description { Faker::Lorem.paragraph }
    price { Faker::Commerce.price }
    stock { Faker::Number.between(from: 0, to: 100) }
    association :category
    
    trait :out_of_stock do
      stock { 0 }
    end
    
    trait :featured do
      featured { true }
    end
  end
end

# spec/factories/orders.rb
FactoryBot.define do
  factory :order do
    association :user
    status { 'pending' }
    total { Faker::Commerce.price(range: 100..10000) }
    
    trait :paid do
      status { 'paid' }
      paid_at { Time.current }
    end
    
    trait :with_items do
      after(:create) do |order|
        create_list(:order_item, 3, order: order)
      end
    end
  end
end
```

### RSpec Model Specs

```ruby
# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  # Shoulda-Matchers validations
  describe 'validations' do
    subject { build(:user) }
    
    it { is_expected.to validate_presence_of(:name) }
    it { is_expected.to validate_presence_of(:email) }
    it { is_expected.to validate_uniqueness_of(:email).case_insensitive }
    it { is_expected.to validate_length_of(:name).is_at_most(100) }
    it { is_expected.to allow_value('user@example.com').for(:email) }
    it { is_expected.not_to allow_value('invalid-email').for(:email) }
  end
  
  describe 'associations' do
    it { is_expected.to have_many(:orders).dependent(:destroy) }
    it { is_expected.to have_one(:profile).dependent(:destroy) }
    it { is_expected.to have_many(:api_keys) }
    it { is_expected.to belong_to(:organization).optional }
  end
  
  describe 'scopes' do
    describe '.active' do
      let!(:active_user)   { create(:user) }
      let!(:inactive_user) { create(:user, :inactive) }
      
      it 'returns only active users' do
        expect(User.active).to include(active_user)
        expect(User.active).not_to include(inactive_user)
      end
    end
    
    describe '.admin' do
      let!(:admin)    { create(:user, :admin) }
      let!(:regular)  { create(:user) }
      
      it 'returns only admin users' do
        expect(User.admin).to contain_exactly(admin)
      end
    end
  end
  
  describe 'instance methods' do
    describe '#full_name' do
      let(:user) { build(:user, first_name: 'สมชาย', last_name: 'ใจดี') }
      
      it 'returns combined first and last name' do
        expect(user.full_name).to eq('สมชาย ใจดี')
      end
    end
    
    describe '#admin?' do
      it 'returns true for admin users' do
        admin = build(:user, :admin)
        expect(admin.admin?).to be true
      end
      
      it 'returns false for regular users' do
        user = build(:user)
        expect(user.admin?).to be false
      end
    end
    
    describe '#can_purchase?' do
      let(:user) { create(:user) }
      let(:product) { create(:product, stock: 5) }
      
      it 'returns true when product is in stock' do
        expect(user.can_purchase?(product, quantity: 3)).to be true
      end
      
      it 'returns false when quantity exceeds stock' do
        expect(user.can_purchase?(product, quantity: 10)).to be false
      end
    end
  end
end
```

---

## Step 1498: Integration Testing ด้วย Capybara

```ruby
# Gemfile
group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
  gem 'cuprite'  # Headless Chrome
end

# spec/support/capybara.rb
require 'capybara/rspec'
require 'capybara/cuprite'

Capybara.register_driver :cuprite do |app|
  Capybara::Cuprite::Driver.new(
    app,
    window_size: [1200, 800],
    browser_options: {
      'no-sandbox': nil,
      'disable-dev-shm-usage': nil,
      'disable-gpu': nil
    },
    headless: ENV.fetch('HEADLESS', 'true') == 'true',
    slowmo: ENV['SLOWMO']&.to_f,
    timeout: 15
  )
end

Capybara.default_driver = :rack_test
Capybara.javascript_driver = :cuprite
Capybara.default_max_wait_time = 5

# ตัวอย่าง Feature Spec
# spec/features/user_authentication_spec.rb
RSpec.describe 'User Authentication', type: :feature do
  describe 'Sign Up' do
    it 'allows user to register' do
      visit new_user_registration_path
      
      fill_in 'ชื่อ', with: 'สมชาย ใจดี'
      fill_in 'อีเมล', with: 'somchai@example.com'
      fill_in 'รหัสผ่าน', with: 'password123'
      fill_in 'ยืนยันรหัสผ่าน', with: 'password123'
      check 'ยอมรับเงื่อนไขการใช้งาน'
      
      click_button 'สมัครสมาชิก'
      
      expect(page).to have_text('สมัครสมาชิกสำเร็จ')
      expect(page).to have_current_path(dashboard_path)
      expect(page).to have_text('สมชาย ใจดี')
    end
    
    it 'shows errors for invalid data' do
      visit new_user_registration_path
      
      fill_in 'อีเมล', with: 'invalid-email'
      click_button 'สมัครสมาชิก'
      
      expect(page).to have_css('.error', text: /อีเมลไม่ถูกต้อง/)
    end
    
    it 'shows error for existing email' do
      create(:user, email: 'existing@example.com')
      
      visit new_user_registration_path
      fill_in 'ชื่อ', with: 'ทดสอบ'
      fill_in 'อีเมล', with: 'existing@example.com'
      fill_in 'รหัสผ่าน', with: 'password123'
      fill_in 'ยืนยันรหัสผ่าน', with: 'password123'
      
      click_button 'สมัครสมาชิก'
      
      expect(page).to have_text('อีเมลนี้ถูกใช้งานแล้ว')
    end
  end
  
  describe 'Sign In' do
    let!(:user) { create(:user, email: 'user@example.com', password: 'password123') }
    
    it 'allows user to login' do
      visit new_user_session_path
      
      fill_in 'อีเมล', with: 'user@example.com'
      fill_in 'รหัสผ่าน', with: 'password123'
      click_button 'เข้าสู่ระบบ'
      
      expect(page).to have_current_path(dashboard_path)
      expect(page).to have_text('ยินดีต้อนรับกลับ')
    end
    
    it 'shows error for wrong password' do
      visit new_user_session_path
      
      fill_in 'อีเมล', with: 'user@example.com'
      fill_in 'รหัสผ่าน', with: 'wrongpassword'
      click_button 'เข้าสู่ระบบ'
      
      expect(page).to have_text('อีเมลหรือรหัสผ่านไม่ถูกต้อง')
      expect(page).to have_current_path(new_user_session_path)
    end
  end
end
```

### Complex Feature Spec

```ruby
# spec/features/shopping_cart_spec.rb
RSpec.describe 'Shopping Cart', type: :feature, js: true do
  let!(:user)     { create(:user) }
  let!(:product1) { create(:product, name: 'สินค้า A', price: 299, stock: 10) }
  let!(:product2) { create(:product, name: 'สินค้า B', price: 499, stock: 5) }
  
  before { sign_in user }
  
  describe 'Adding items to cart' do
    it 'adds product to cart and shows count' do
      visit product_path(product1)
      
      fill_in 'quantity', with: '2'
      click_button 'เพิ่มลงตะกร้า'
      
      expect(page).to have_text('เพิ่มลงตะกร้าสำเร็จ')
      
      # Cart counter updates via AJAX
      within('.cart-count') do
        expect(page).to have_text('2')
      end
    end
    
    it 'allows updating quantity in cart' do
      visit product_path(product1)
      click_button 'เพิ่มลงตะกร้า'
      
      visit cart_path
      
      within("[data-product-id='#{product1.id}']") do
        fill_in 'quantity', with: '3'
        click_button 'อัพเดท'
      end
      
      expect(page).to have_text('297')  # 99 * 3
    end
  end
  
  describe 'Checkout process' do
    before do
      visit product_path(product1)
      click_button 'เพิ่มลงตะกร้า'
    end
    
    it 'completes checkout successfully', :payment do
      visit checkout_path
      
      # กรอกที่อยู่จัดส่ง
      fill_in 'ชื่อ-นามสกุล', with: 'สมชาย ใจดี'
      fill_in 'ที่อยู่', with: '123 ถ.สุขุมวิท'
      fill_in 'จังหวัด', with: 'กรุงเทพมหานคร'
      fill_in 'รหัสไปรษณีย์', with: '10110'
      
      # กรอกบัตรเครดิต (Stripe test card)
      within_frame('stripe-card-frame') do
        fill_in 'cardnumber', with: '4242424242424242'
        fill_in 'exp-date', with: '12/25'
        fill_in 'cvc', with: '123'
      end
      
      click_button 'ชำระเงิน'
      
      expect(page).to have_text('ขอบคุณสำหรับการสั่งซื้อ')
      expect(page).to have_current_path(/orders\/\d+\/confirmation/)
    end
    
    it 'shows error for declined card' do
      visit checkout_path
      
      within_frame('stripe-card-frame') do
        fill_in 'cardnumber', with: '4000000000000002'  # Decline card
        fill_in 'exp-date', with: '12/25'
        fill_in 'cvc', with: '123'
      end
      
      click_button 'ชำระเงิน'
      
      expect(page).to have_text('บัตรถูกปฏิเสธ')
    end
  end
end
```

---

## Step 1499: JavaScript Testing ด้วย Selenium และ Cuprite

```ruby
# spec/support/selenium_helper.rb
# ตั้งค่า Selenium สำหรับ JS tests

RSpec.configure do |config|
  config.before(:each, type: :system) do
    driven_by :cuprite, using: :chrome, screen_size: [1400, 1400]
  end
  
  config.before(:each, js: true, type: :feature) do
    Capybara.current_driver = :cuprite
  end
  
  config.after(:each, js: true, type: :feature) do
    Capybara.use_default_driver
  end
end

# Helpers สำหรับ JS testing
module JavaScriptHelpers
  def wait_for_ajax
    Timeout.timeout(Capybara.default_max_wait_time) do
      loop until finished_all_ajax_requests?
    end
  end
  
  def finished_all_ajax_requests?
    page.evaluate_script('typeof jQuery === "undefined" || jQuery.active === 0')
  end
  
  def wait_for_turbo
    Timeout.timeout(5) do
      loop until page.evaluate_script('Turbo.navigator.currentVisit === null')
    end
  rescue Timeout::Error
    # Turbo ไม่ได้ใช้งาน
  end
  
  def scroll_to_element(element)
    page.execute_script(
      "arguments[0].scrollIntoView(true);",
      element.native
    )
  end
end

RSpec.configure do |config|
  config.include JavaScriptHelpers, type: :feature
end
```

---

## Step 1500: API Testing

```ruby
# spec/requests/api/v1/products_spec.rb
RSpec.describe 'API::V1::Products', type: :request do
  let!(:user) { create(:user) }
  let!(:admin) { create(:user, :admin) }
  let(:auth_headers) { 
    { 'Authorization' => "Bearer #{JsonWebToken.encode(user_id: user.id, type: 'access')}" }
  }
  let(:admin_headers) {
    { 'Authorization' => "Bearer #{JsonWebToken.encode(user_id: admin.id, type: 'access')}" }
  }
  
  describe 'GET /api/v1/products' do
    let!(:products) { create_list(:product, 5) }
    
    it 'returns list of products' do
      get '/api/v1/products', headers: auth_headers
      
      expect(response).to have_http_status(:ok)
      expect(json_body['products'].length).to eq(5)
    end
    
    it 'returns paginated results' do
      create_list(:product, 20)
      
      get '/api/v1/products', 
          params: { page: 1, per_page: 10 },
          headers: auth_headers
      
      expect(json_body['products'].length).to eq(10)
      expect(json_body['meta']['total_pages']).to eq(3)
    end
    
    it 'filters by category' do
      category = create(:category)
      category_products = create_list(:product, 3, category: category)
      
      get '/api/v1/products', 
          params: { category_id: category.id },
          headers: auth_headers
      
      returned_ids = json_body['products'].map { |p| p['id'] }
      expect(returned_ids).to match_array(category_products.map(&:id))
    end
    
    it 'searches by name' do
      create(:product, name: 'iPhone 15 Case')
      create(:product, name: 'Samsung Galaxy Cover')
      
      get '/api/v1/products',
          params: { q: 'iPhone' },
          headers: auth_headers
      
      expect(json_body['products'].length).to eq(1)
      expect(json_body['products'].first['name']).to include('iPhone')
    end
    
    it 'requires authentication' do
      get '/api/v1/products'
      expect(response).to have_http_status(:unauthorized)
    end
  end
  
  describe 'POST /api/v1/products' do
    let(:valid_params) {
      {
        product: {
          name: 'New Product',
          description: 'Test product',
          price: 299.99,
          stock: 100,
          category_id: create(:category).id
        }
      }
    }
    
    context 'as admin' do
      it 'creates a new product' do
        expect {
          post '/api/v1/products',
               params: valid_params,
               headers: admin_headers
        }.to change(Product, :count).by(1)
        
        expect(response).to have_http_status(:created)
        expect(json_body['product']['name']).to eq('New Product')
      end
    end
    
    context 'as regular user' do
      it 'returns forbidden' do
        post '/api/v1/products',
             params: valid_params,
             headers: auth_headers
        
        expect(response).to have_http_status(:forbidden)
      end
    end
    
    context 'with invalid params' do
      it 'returns validation errors' do
        post '/api/v1/products',
             params: { product: { name: '' } },
             headers: admin_headers
        
        expect(response).to have_http_status(:unprocessable_entity)
        expect(json_body).to have_key('errors')
      end
    end
  end
  
  describe 'PUT /api/v1/products/:id' do
    let!(:product) { create(:product) }
    
    it 'updates product' do
      put "/api/v1/products/#{product.id}",
          params: { product: { price: 199.99 } },
          headers: admin_headers
      
      expect(response).to have_http_status(:ok)
      expect(json_body['product']['price']).to eq('199.99')
      expect(product.reload.price).to eq(199.99)
    end
    
    it 'returns 404 for non-existent product' do
      put '/api/v1/products/99999',
          params: { product: { price: 100 } },
          headers: admin_headers
      
      expect(response).to have_http_status(:not_found)
    end
  end
  
  def json_body
    JSON.parse(response.body)
  end
end
```

---

## Step 1501: Performance Testing

```ruby
# spec/performance/products_performance_spec.rb
require 'rails_helper'
require 'benchmark'

RSpec.describe 'Products Performance', type: :request do
  let!(:products) { create_list(:product, 1000, :with_category) }
  let(:auth_headers) { ... }
  
  describe 'GET /api/v1/products' do
    it 'responds within acceptable time' do
      elapsed = Benchmark.realtime do
        get '/api/v1/products', headers: auth_headers
      end
      
      expect(elapsed).to be < 0.5  # ต้องเร็วกว่า 500ms
    end
    
    it 'does not have N+1 queries' do
      expect {
        get '/api/v1/products', headers: auth_headers
      }.to make_database_queries(count: 1..5)
      # ใช้ gem 'db-query-matchers'
    end
  end
end

# Benchmark Service Objects
RSpec.describe UserRegistration do
  it 'completes within 1 second' do
    elapsed = Benchmark.realtime do
      UserRegistration.call(
        name: 'Test',
        email: Faker::Internet.unique.email,
        password: 'password123',
        password_confirmation: 'password123'
      )
    end
    
    expect(elapsed).to be < 1.0
  end
end
```

### ใช้ RSpec Profiler

```ruby
# spec/support/profiling.rb
if ENV['PROFILE']
  RSpec.configure do |config|
    config.before(:suite) do
      puts "\nProfiling enabled"
    end
    
    config.around(:each) do |example|
      start = Time.current
      example.run
      elapsed = Time.current - start
      
      if elapsed > 0.5
        puts "\n⚠️  SLOW TEST (#{elapsed.round(3)}s): #{example.full_description}"
      end
    end
  end
end
```

---

## Step 1502: Mock External Services กับ WebMock

```ruby
# Gemfile
group :test do
  gem 'webmock'
  gem 'vcr'
end

# spec/support/webmock.rb
require 'webmock/rspec'

WebMock.disable_net_connect!(allow_localhost: true)

# Shared stubs
module ExternalServiceStubs
  def stub_stripe_charge_success(charge_id: 'ch_test123', amount: 1000)
    stub_request(:post, 'https://api.stripe.com/v1/charges')
      .to_return(
        status: 200,
        body: {
          id: charge_id,
          amount: amount,
          status: 'succeeded',
          object: 'charge'
        }.to_json,
        headers: { 'Content-Type' => 'application/json' }
      )
  end
  
  def stub_stripe_charge_failure(code: 'card_declined', message: 'Your card was declined')
    stub_request(:post, 'https://api.stripe.com/v1/charges')
      .to_return(
        status: 402,
        body: {
          error: {
            type: 'card_error',
            code: code,
            message: message
          }
        }.to_json,
        headers: { 'Content-Type' => 'application/json' }
      )
  end
  
  def stub_mailchimp_subscribe_success(email:)
    stub_request(:post, /api.mailchimp.com/)
      .to_return(status: 200, body: { email_address: email }.to_json)
  end
  
  def stub_google_geocoding(address:, lat:, lng:)
    stub_request(:get, /maps.googleapis.com\/maps\/api\/geocode/)
      .with(query: hash_including(address: address))
      .to_return(
        status: 200,
        body: {
          results: [
            {
              geometry: {
                location: { lat: lat, lng: lng }
              },
              formatted_address: address
            }
          ],
          status: 'OK'
        }.to_json
      )
  end
end

RSpec.configure do |config|
  config.include ExternalServiceStubs
end

# ใน spec
RSpec.describe PaymentProcessor do
  describe '#call' do
    let(:order) { create(:order, total: 1000) }
    
    context 'successful payment' do
      before { stub_stripe_charge_success(amount: 100000) }  # cents
      
      it 'processes payment' do
        result = PaymentProcessor.call(
          order: order,
          payment_method_id: 'pm_test',
          user: order.user
        )
        
        expect(result.success?).to be true
        expect(order.reload.status).to eq('paid')
      end
    end
    
    context 'failed payment' do
      before { stub_stripe_charge_failure }
      
      it 'returns failure' do
        result = PaymentProcessor.call(
          order: order,
          payment_method_id: 'pm_test',
          user: order.user
        )
        
        expect(result.success?).to be false
        expect(result.errors).to include(/card/)
      end
    end
  end
end
```

---

## Step 1503: VCR สำหรับ Record HTTP Interactions

```ruby
# spec/support/vcr.rb
require 'vcr'

VCR.configure do |config|
  config.cassette_library_dir = 'spec/fixtures/vcr_cassettes'
  config.hook_into :webmock
  config.configure_rspec_metadata!
  
  # ซ่อน sensitive data
  config.filter_sensitive_data('<STRIPE_SECRET_KEY>') { ENV['STRIPE_SECRET_KEY'] }
  config.filter_sensitive_data('<API_KEY>') { ENV['EXTERNAL_API_KEY'] }
  
  # Default settings
  config.default_cassette_options = {
    record: :new_episodes,  # บันทึก request ใหม่, เล่น request เก่า
    match_requests_on: [:method, :uri, :body]
  }
  
  # ignore localhost
  config.ignore_localhost = true
end

# ใช้งาน VCR ใน spec
RSpec.describe ExchangeRateService do
  describe '#fetch_rate' do
    it 'fetches USD to THB rate', :vcr do
      # VCR จะบันทึก HTTP request ครั้งแรก
      # แล้วเล่นซ้ำในครั้งต่อไปโดยไม่ต้องเรียก API จริง
      rate = ExchangeRateService.new.fetch_rate(from: 'USD', to: 'THB')
      expect(rate).to be_a(Float)
      expect(rate).to be > 30
    end
  end
  
  it 'handles API errors', vcr: { cassette_name: 'exchange_rate/error' } do
    expect {
      ExchangeRateService.new.fetch_rate(from: 'INVALID', to: 'THB')
    }.to raise_error(ExchangeRateService::InvalidCurrencyError)
  end
end
```

---

## Step 1504: Testing กับ Time ด้วย travel_to

```ruby
# ใช้ Rails built-in travel helpers
RSpec.describe SubscriptionExpiry do
  describe '#expired?' do
    it 'returns true for past subscription' do
      subscription = build(:subscription, expires_at: 1.day.ago)
      expect(subscription.expired?).to be true
    end
    
    it 'returns false for future subscription' do
      subscription = build(:subscription, expires_at: 1.day.from_now)
      expect(subscription.expired?).to be false
    end
  end
end

RSpec.describe 'Subscription renewal' do
  let(:user) { create(:user) }
  
  it 'auto-renews on expiry date' do
    subscription = create(:subscription, 
      user: user,
      expires_at: 30.days.from_now
    )
    
    # Jump to expiry date
    travel_to(31.days.from_now) do
      SubscriptionRenewalJob.perform_now(subscription.id)
      
      expect(subscription.reload.expires_at).to be_within(1.minute).of(61.days.from_now)
    end
  end
  
  it 'sends reminder 7 days before expiry' do
    expires_at = 7.days.from_now
    subscription = create(:subscription, user: user, expires_at: expires_at)
    
    # 8 days before: ยังไม่ส่ง
    travel_to(expires_at - 8.days) do
      expect {
        ExpiryReminderJob.perform_now
      }.not_to change { ActionMailer::Base.deliveries.count }
    end
    
    # 7 days before: ส่ง
    travel_to(expires_at - 7.days) do
      expect {
        ExpiryReminderJob.perform_now
      }.to change { ActionMailer::Base.deliveries.count }.by(1)
    end
  end
end

# ใช้ Timecop gem (รองรับ freeze_time)
require 'timecop'

RSpec.describe DailyReportJob do
  describe '#perform' do
    it 'generates report for yesterday' do
      Timecop.freeze(Date.new(2024, 6, 15)) do
        DailyReportJob.perform_now
        
        report = DailyReport.last
        expect(report.report_date).to eq(Date.new(2024, 6, 14))
      end
    end
    
    it 'processes midnight correctly' do
      Timecop.freeze(Time.new(2024, 6, 15, 0, 0, 0, '+07:00')) do
        expect(Time.current.yesterday.to_date).to eq(Date.new(2024, 6, 14))
      end
    end
  end
end
```

---

## Step 1505: Parallel Tests

```ruby
# Gemfile
gem 'parallel_tests', group: :test

# Rakefile - เพิ่ม task
require 'parallel_tests/tasks'

# config/database.yml
test:
  database: myapp_test<%= ENV['TEST_ENV_NUMBER'] %>

# สร้าง test databases สำหรับ parallel testing
bundle exec rake parallel:create
bundle exec rake parallel:migrate
bundle exec rake parallel:prepare

# รัน tests แบบ parallel
bundle exec parallel_rspec spec/

# รัน tests 4 processes
bundle exec parallel_rspec spec/ -n 4

# รัน specific folder
bundle exec parallel_rspec spec/models/ spec/services/

# spec/support/parallel_tests.rb
# ทำให้ sequence generator unique สำหรับแต่ละ process
if defined?(FactoryBot)
  FactoryBot.define do
    sequence(:unique_email) { |n| "user_#{n}_#{ENV['TEST_ENV_NUMBER']}@example.com" }
  end
end

# ใช้ TEST_ENV_NUMBER ใน seeds หรือ setup
ActiveRecord::Base.connection.execute(
  "SET search_path TO test_#{ENV.fetch('TEST_ENV_NUMBER', 1)}"
) if ENV['PARALLEL_TESTS']
```

---

## Step 1506: Test Coverage Analysis

```ruby
# Gemfile
group :test do
  gem 'simplecov'
  gem 'simplecov-html'
end

# spec/spec_helper.rb
require 'simplecov'

SimpleCov.start 'rails' do
  # Minimum coverage
  minimum_coverage 80
  
  # Groups
  add_group 'Models', 'app/models'
  add_group 'Controllers', 'app/controllers'
  add_group 'Services', 'app/services'
  add_group 'Jobs', 'app/jobs'
  add_group 'Mailers', 'app/mailers'
  
  # Exclude
  add_filter '/spec/'
  add_filter '/config/'
  add_filter 'app/channels/application_cable/'
  
  # Custom formatter
  formatter SimpleCov::Formatter::HTMLFormatter
end

# CI/CD: Fail if coverage drops
SimpleCov.at_exit do
  SimpleCov.result.format!
  
  if SimpleCov.result.covered_percent < 80
    puts "Coverage #{SimpleCov.result.covered_percent.round(2)}% is below 80%!"
    exit 1
  end
end

# ดู coverage report
open coverage/index.html
```

---

## Step 1507: Advanced RSpec Patterns

### Shared Examples

```ruby
# spec/support/shared_examples/api_authentication.rb
RSpec.shared_examples 'requires authentication' do
  it 'returns 401 without token' do
    send(http_method, path)
    expect(response).to have_http_status(:unauthorized)
  end
  
  it 'returns 401 with expired token' do
    expired_token = JsonWebToken.encode(
      { user_id: 1 },
      expiry: 1.hour.ago
    )
    
    send(http_method, path, 
      headers: { 'Authorization' => "Bearer #{expired_token}" }
    )
    
    expect(response).to have_http_status(:unauthorized)
  end
end

RSpec.shared_examples 'requires admin role' do
  context 'as regular user' do
    let(:user) { create(:user) }
    
    it 'returns forbidden' do
      send(http_method, path, headers: auth_headers(user))
      expect(response).to have_http_status(:forbidden)
    end
  end
end

# ใช้งาน
RSpec.describe 'Admin Products API' do
  describe 'DELETE /api/v1/admin/products/:id' do
    let(:http_method) { :delete }
    let(:path) { "/api/v1/admin/products/#{create(:product).id}" }
    
    it_behaves_like 'requires authentication'
    it_behaves_like 'requires admin role'
  end
end
```

### Custom Matchers

```ruby
# spec/support/matchers/api_matchers.rb
RSpec::Matchers.define :have_api_error do |expected_message|
  match do |response|
    body = JSON.parse(response.body)
    body['error'] == expected_message || 
    Array(body['errors']).include?(expected_message)
  end
  
  failure_message do |response|
    "Expected response to have error '#{expected_message}', " \
    "but got: #{response.body}"
  end
end

RSpec::Matchers.define :be_paginated do
  match do |response|
    body = JSON.parse(response.body)
    body.key?('meta') && 
    body['meta'].key?('current_page') &&
    body['meta'].key?('total_pages')
  end
  
  failure_message do |response|
    "Expected response to have pagination meta, but got: #{response.body}"
  end
end

RSpec::Matchers.define :match_schema do |schema|
  match do |response|
    body = JSON.parse(response.body)
    JSON::Validator.validate(schema_path(schema), body)
  end
  
  def schema_path(schema)
    Rails.root.join("spec/fixtures/schemas/#{schema}.json")
  end
end

# ใช้งาน
it 'returns paginated products' do
  get '/api/v1/products', headers: auth_headers
  
  expect(response).to be_paginated
  expect(response).to match_schema('product_list')
end
```

### Contexts และ Let Helpers

```ruby
# spec/support/helpers/authentication_helpers.rb
module AuthenticationHelpers
  def auth_headers(user = nil)
    user ||= create(:user)
    token = JsonWebToken.encode(user_id: user.id, type: 'access')
    { 'Authorization' => "Bearer #{token}", 'Content-Type' => 'application/json' }
  end
  
  def admin_headers
    auth_headers(create(:user, :admin))
  end
  
  def api_key_headers(user = nil)
    user ||= create(:user)
    api_key = create(:api_key, user: user)
    { 'X-API-Key' => api_key.raw_key }
  end
end

RSpec.configure do |config|
  config.include AuthenticationHelpers, type: :request
end
```

---

## Step 1508-1520: แบบฝึกหัด 25 ข้อ

### แบบฝึกหัดที่ 1: TDD สำหรับ UserRegistration

```ruby
# เขียน test ก่อน แล้วค่อยสร้าง service
RSpec.describe UserRegistration do
  describe '#call' do
    let(:valid_params) do
      {
        name: 'สมชาย ใจดี',
        email: 'somchai@example.com',
        password: 'password123',
        password_confirmation: 'password123'
      }
    end
    
    context 'with valid params' do
      it 'creates a new user' do
        expect { UserRegistration.call(**valid_params) }
          .to change(User, :count).by(1)
      end
      
      it 'returns success result' do
        result = UserRegistration.call(**valid_params)
        expect(result.success?).to be true
      end
      
      it 'sets the user on result' do
        result = UserRegistration.call(**valid_params)
        expect(result.user).to be_a(User)
        expect(result.user.email).to eq('somchai@example.com')
      end
      
      it 'sends welcome email' do
        expect {
          UserRegistration.call(**valid_params)
        }.to change { ActionMailer::Base.deliveries.count }.by(1)
      end
      
      it 'creates user profile' do
        result = UserRegistration.call(**valid_params)
        expect(result.user.profile).to be_present
      end
    end
    
    context 'with invalid email' do
      it 'returns failure' do
        result = UserRegistration.call(**valid_params.merge(email: 'invalid'))
        expect(result.failure?).to be true
        expect(result.errors).to include(match(/อีเมล/))
      end
      
      it 'does not create user' do
        expect {
          UserRegistration.call(**valid_params.merge(email: 'invalid'))
        }.not_to change(User, :count)
      end
    end
    
    context 'with existing email' do
      before { create(:user, email: 'somchai@example.com') }
      
      it 'returns failure with duplicate email error' do
        result = UserRegistration.call(**valid_params)
        expect(result.failure?).to be true
        expect(result.errors).to include(match(/ถูกใช้งานแล้ว/))
      end
    end
    
    context 'with password mismatch' do
      it 'returns failure' do
        result = UserRegistration.call(
          **valid_params.merge(password_confirmation: 'different')
        )
        expect(result.failure?).to be true
        expect(result.errors).to include(match(/ไม่ตรงกัน/))
      end
    end
    
    context 'when database error occurs' do
      before do
        allow(User).to receive(:create!).and_raise(ActiveRecord::RecordInvalid)
      end
      
      it 'returns failure' do
        result = UserRegistration.call(**valid_params)
        expect(result.failure?).to be true
      end
    end
  end
end
```

### แบบฝึกหัดที่ 2: WebMock สำหรับ External Service

```ruby
RSpec.describe WeatherService do
  let(:service) { WeatherService.new(api_key: 'test_key') }
  
  describe '#current_weather' do
    before do
      stub_request(:get, 'https://api.openweathermap.org/data/2.5/weather')
        .with(
          query: {
            q: 'Bangkok',
            appid: 'test_key',
            units: 'metric',
            lang: 'th'
          }
        )
        .to_return(
          status: 200,
          body: {
            name: 'Bangkok',
            main: { temp: 32.5, humidity: 70 },
            weather: [{ description: 'มีเมฆบางส่วน', icon: '02d' }],
            wind: { speed: 3.5 }
          }.to_json,
          headers: { 'Content-Type' => 'application/json' }
        )
    end
    
    it 'returns weather data' do
      weather = service.current_weather(city: 'Bangkok')
      
      expect(weather[:temperature]).to eq(32.5)
      expect(weather[:description]).to eq('มีเมฆบางส่วน')
      expect(weather[:humidity]).to eq(70)
    end
    
    it 'calls the API once' do
      service.current_weather(city: 'Bangkok')
      
      expect(a_request(:get, 'https://api.openweathermap.org/data/2.5/weather'))
        .to have_been_made.once
    end
  end
  
  describe 'error handling' do
    before do
      stub_request(:get, /api.openweathermap.org/)
        .to_return(status: 404, body: { message: 'city not found' }.to_json)
    end
    
    it 'raises CityNotFoundError' do
      expect { service.current_weather(city: 'FakeCity99') }
        .to raise_error(WeatherService::CityNotFoundError)
    end
  end
  
  describe 'timeout handling' do
    before do
      stub_request(:get, /api.openweathermap.org/)
        .to_timeout
    end
    
    it 'raises TimeoutError' do
      expect { service.current_weather(city: 'Bangkok') }
        .to raise_error(WeatherService::TimeoutError)
    end
  end
end
```

### แบบฝึกหัดที่ 3: Capybara Feature Test

```ruby
# spec/features/product_search_spec.rb
RSpec.describe 'Product Search', type: :feature, js: true do
  let!(:iphone_case)  { create(:product, name: 'iPhone 15 Case', price: 299) }
  let!(:samsung_case) { create(:product, name: 'Samsung Galaxy Case', price: 199) }
  let!(:laptop_bag)   { create(:product, name: 'Laptop Bag', price: 899) }
  
  before { sign_in create(:user) }
  
  it 'searches products by name' do
    visit products_path
    
    fill_in 'ค้นหาสินค้า', with: 'iPhone'
    click_button 'ค้นหา'
    
    expect(page).to have_text('iPhone 15 Case')
    expect(page).not_to have_text('Samsung')
    expect(page).not_to have_text('Laptop')
  end
  
  it 'filters by price range' do
    visit products_path
    
    fill_in 'ราคาต่ำสุด', with: '200'
    fill_in 'ราคาสูงสุด', with: '400'
    click_button 'กรอง'
    
    expect(page).to have_text('iPhone 15 Case')
    expect(page).not_to have_text('Samsung Galaxy Case')  # 199 < 200
    expect(page).not_to have_text('Laptop Bag')  # 899 > 400
  end
  
  it 'shows auto-suggestions while typing', js: true do
    visit products_path
    
    fill_in 'ค้นหาสินค้า', with: 'Case'
    
    # รอ auto-suggest dropdown ปรากฏ
    expect(page).to have_css('.autocomplete-results', wait: 2)
    
    within('.autocomplete-results') do
      expect(page).to have_text('iPhone 15 Case')
      expect(page).to have_text('Samsung Galaxy Case')
    end
    
    # Click suggestion
    click_on 'iPhone 15 Case'
    
    expect(page).to have_current_path(product_path(iphone_case))
  end
  
  it 'sorts by price' do
    visit products_path
    
    select 'ราคา: ต่ำ-สูง', from: 'เรียงตาม'
    
    products = all('.product-name').map(&:text)
    
    expect(products.index('Samsung Galaxy Case')).to be < products.index('iPhone 15 Case')
    expect(products.index('iPhone 15 Case')).to be < products.index('Laptop Bag')
  end
end
```

### แบบฝึกหัดที่ 4: Travel Time Tests

```ruby
RSpec.describe 'Coupon Expiry', type: :model do
  describe 'Coupon validity over time' do
    let(:coupon) { create(:coupon, expires_at: 7.days.from_now) }
    
    it 'is valid before expiry' do
      travel_to(6.days.from_now) do
        expect(coupon.valid?).to be true
      end
    end
    
    it 'is expired after expiry date' do
      travel_to(8.days.from_now) do
        expect(coupon.expired?).to be true
      end
    end
    
    it 'is valid exactly at expiry time' do
      travel_to(coupon.expires_at - 1.second) do
        expect(coupon.valid?).to be true
      end
    end
    
    it 'is expired exactly after expiry time' do
      travel_to(coupon.expires_at + 1.second) do
        expect(coupon.expired?).to be true
      end
    end
  end
end

RSpec.describe DailyDigestMailer do
  describe '#daily_digest' do
    it 'includes only today orders' do
      today_order     = create(:order, :paid, created_at: Time.current)
      yesterday_order = create(:order, :paid, created_at: 1.day.ago)
      
      travel_to(Time.current.beginning_of_day + 8.hours) do
        mail = DailyDigestMailer.daily_digest(create(:user, :admin))
        
        expect(mail.body.encoded).to include(today_order.number)
        expect(mail.body.encoded).not_to include(yesterday_order.number)
      end
    end
  end
end
```

### แบบฝึกหัดที่ 5: Test Coverage สำหรับ Service Object

```ruby
# ตัวอย่าง complete test suite สำหรับ PaymentProcessor
RSpec.describe PaymentProcessor do
  let(:user)    { create(:user) }
  let(:order)   { create(:order, user: user, total: 1000, status: 'pending') }
  let(:service) { described_class.new(order: order, payment_method_id: 'pm_test', user: user) }
  
  describe '#call' do
    context 'successful payment' do
      before do
        stub_request(:post, 'https://api.stripe.com/v1/charges')
          .to_return(
            status: 200,
            body: { id: 'ch_success', status: 'succeeded' }.to_json
          )
      end
      
      it 'processes payment successfully' do
        result = service.call
        expect(result.success?).to be true
      end
      
      it 'updates order status to paid' do
        service.call
        expect(order.reload.status).to eq('paid')
      end
      
      it 'records payment' do
        expect { service.call }.to change(Payment, :count).by(1)
        expect(Payment.last.status).to eq('completed')
      end
      
      it 'sends receipt email' do
        expect { service.call }
          .to change { ActionMailer::Base.deliveries.count }.by(1)
      end
      
      it 'creates audit log' do
        expect { service.call }.to change(AuditLog, :count).by(1)
      end
    end
    
    context 'payment failure' do
      before do
        stub_request(:post, 'https://api.stripe.com/v1/charges')
          .to_return(
            status: 402,
            body: {
              error: {
                type: 'card_error',
                code: 'card_declined',
                message: 'Your card was declined.'
              }
            }.to_json
          )
      end
      
      it 'returns failure' do
        result = service.call
        expect(result.success?).to be false
      end
      
      it 'does not update order status' do
        service.call
        expect(order.reload.status).to eq('pending')
      end
      
      it 'records failed payment' do
        service.call
        expect(Payment.last&.status).to eq('failed')
      end
    end
    
    context 'with invalid order' do
      let(:order) { create(:order, status: 'paid') }
      
      it 'returns failure for already paid order' do
        result = service.call
        expect(result.success?).to be false
        expect(result.errors).to include(match(/ชำระเงินแล้ว/))
      end
    end
    
    context 'concurrent payment attempts' do
      it 'handles race conditions gracefully' do
        threads = 3.times.map do
          Thread.new do
            PaymentProcessor.call(
              order: order,
              payment_method_id: 'pm_test',
              user: user
            )
          end
        end
        
        results = threads.map(&:value)
        successful = results.count(&:success?)
        
        expect(successful).to eq(1)
        expect(order.reload.status).to eq('paid')
      end
    end
  end
end
```

### สรุป Advanced Testing Best Practices

```
TDD Tips:
1. เขียน test ที่ fail ก่อนเสมอ
2. เขียน code น้อยที่สุดเพื่อทำให้ test ผ่าน
3. Refactor หลังจาก test ผ่านแล้วเท่านั้น
4. ชื่อ test ควรบอกว่า "เมื่อไหร่" และ "ทำอะไร"

RSpec Tips:
1. ใช้ let และ let! อย่างเหมาะสม
2. describe/context ให้ชัดเจน
3. หลีกเลี่ยง before(:all) ใช้ let(:factory) แทน
4. ใช้ shared_examples สำหรับ behavior เดิมๆ

Integration Test Tips:
1. ทดสอบ happy path และ error path
2. ทดสอบ boundaries และ edge cases
3. ใช้ factory traits แทน manual setup
4. Mock external services เสมอ

Coverage Tips:
1. ตั้ง minimum coverage threshold
2. ดู branch coverage ไม่ใช่แค่ line coverage
3. Coverage 100% ไม่ได้หมายความว่า code ดี
4. Focus on critical paths
```

---

**จบ Part 69: Advanced Testing**

*ในส่วนถัดไป Part 70 เราจะเรียนรู้เกี่ยวกับ Admin Panels*
