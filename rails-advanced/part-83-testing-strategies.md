# Part 83: Testing Strategies ขั้นสูงใน Ruby on Rails

## บทนำ

การทดสอบซอฟต์แวร์เป็นทักษะสำคัญที่แยกแยะนักพัฒนาระดับกลางกับระดับสูง ในบทนี้เราจะเรียนรู้กลยุทธ์การทดสอบขั้นสูงที่ช่วยให้ codebase มีความน่าเชื่อถือสูง

---

## 1. Testing Pyramid

Testing Pyramid เป็นแนวคิดที่บอกว่าควรมีการทดสอบในแต่ละระดับเท่าไหร่

```
         /\
        /E2E\         <- น้อยที่สุด (5-10%)
       /------\
      / Integr \      <- ปานกลาง (20-30%)
     /----------\
    /  Unit Tests \   <- มากที่สุด (60-70%)
   /--------------\
```

### 1.1 Unit Tests

```ruby
# app/models/user.rb
class User < ApplicationRecord
  validates :email, presence: true, 
                    format: { with: URI::MailTo::EMAIL_REGEXP },
                    uniqueness: { case_sensitive: false }
  validates :name, presence: true, length: { minimum: 2, maximum: 100 }
  validates :password, length: { minimum: 8 }, if: :password_required?
  
  has_secure_password
  
  before_save :normalize_email
  
  def active?
    !deactivated_at?
  end
  
  def deactivate!
    update!(deactivated_at: Time.current)
  end
  
  def display_name
    name.presence || email.split('@').first
  end
  
  private
  
  def normalize_email
    self.email = email.downcase.strip
  end
  
  def password_required?
    new_record? || password.present?
  end
end

# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  subject(:user) { build(:user) }
  
  # Validations
  describe 'validations' do
    it { is_expected.to validate_presence_of(:email) }
    it { is_expected.to validate_presence_of(:name) }
    it { is_expected.to validate_uniqueness_of(:email).case_insensitive }
    it { is_expected.to validate_length_of(:name).is_at_least(2).is_at_most(100) }
    
    describe 'email format' do
      it 'valid email ผ่าน validation' do
        user.email = 'valid@example.com'
        expect(user).to be_valid
      end
      
      it 'invalid email ไม่ผ่าน validation' do
        ['not-email', '@.com', 'missing@', 'a@b'].each do |invalid_email|
          user.email = invalid_email
          expect(user).not_to be_valid, "Expected #{invalid_email} to be invalid"
        end
      end
    end
  end
  
  # Associations
  describe 'associations' do
    it { is_expected.to have_many(:posts).dependent(:destroy) }
    it { is_expected.to have_many(:comments).dependent(:destroy) }
    it { is_expected.to belong_to(:organization).optional }
  end
  
  # Instance methods
  describe '#active?' do
    it 'คืน true เมื่อ deactivated_at เป็น nil' do
      user.deactivated_at = nil
      expect(user.active?).to be true
    end
    
    it 'คืน false เมื่อมี deactivated_at' do
      user.deactivated_at = 1.day.ago
      expect(user.active?).to be false
    end
  end
  
  describe '#deactivate!' do
    let(:user) { create(:user) }
    
    it 'ตั้งค่า deactivated_at' do
      freeze_time do
        user.deactivate!
        expect(user.reload.deactivated_at).to eq(Time.current)
      end
    end
    
    it 'ทำให้ user inactive' do
      user.deactivate!
      expect(user.active?).to be false
    end
  end
  
  describe '#display_name' do
    it 'คืน name เมื่อมี name' do
      user.name = 'John Doe'
      expect(user.display_name).to eq('John Doe')
    end
    
    it 'คืน email prefix เมื่อไม่มี name' do
      user.name = nil
      user.email = 'john@example.com'
      expect(user.display_name).to eq('john')
    end
  end
  
  # Callbacks
  describe 'callbacks' do
    describe '#normalize_email (before_save)' do
      it 'แปลง email เป็น lowercase' do
        user = create(:user, email: 'JOHN@EXAMPLE.COM')
        expect(user.reload.email).to eq('john@example.com')
      end
      
      it 'ตัด whitespace ออก' do
        user = create(:user, email: '  john@example.com  ')
        expect(user.reload.email).to eq('john@example.com')
      end
    end
  end
end
```

### 1.2 Integration Tests

```ruby
# spec/services/user_registration_service_spec.rb
RSpec.describe UserRegistrationService, type: :service do
  subject(:service) { described_class.new(params) }
  
  let(:params) do
    {
      name: 'John Doe',
      email: 'john@example.com',
      password: 'password123',
      plan: 'basic'
    }
  end
  
  describe '#call' do
    context 'ด้วย valid params' do
      it 'สร้าง user' do
        expect { service.call }.to change(User, :count).by(1)
      end
      
      it 'สร้าง subscription' do
        service.call
        expect(Subscription.last.plan).to eq('basic')
      end
      
      it 'ส่ง welcome email' do
        expect { service.call }
          .to have_enqueued_mail(UserMailer, :welcome)
      end
      
      it 'คืน success result' do
        result = service.call
        expect(result.success?).to be true
        expect(result.user).to be_a(User)
      end
    end
    
    context 'เมื่อ email ซ้ำ' do
      before { create(:user, email: 'john@example.com') }
      
      it 'ไม่สร้าง user' do
        expect { service.call }.not_to change(User, :count)
      end
      
      it 'คืน failure result พร้อม errors' do
        result = service.call
        expect(result.success?).to be false
        expect(result.errors[:email]).to include('has already been taken')
      end
    end
    
    context 'เมื่อ database error' do
      before do
        allow(User).to receive(:create!).and_raise(ActiveRecord::StatementInvalid, 'DB Error')
      end
      
      it 'rollback ทั้ง transaction' do
        expect { service.call }.not_to change(Subscription, :count)
      end
    end
  end
end
```

### 1.3 End-to-End Tests ด้วย Capybara

```ruby
# spec/features/user_registration_spec.rb
require 'rails_helper'

RSpec.describe 'User Registration', type: :feature, js: true do
  before do
    visit new_user_registration_path
  end
  
  scenario 'ลงทะเบียนสำเร็จ' do
    fill_in 'Name', with: 'John Doe'
    fill_in 'Email', with: 'john@example.com'
    fill_in 'Password', with: 'password123'
    fill_in 'Password confirmation', with: 'password123'
    click_button 'Register'
    
    expect(page).to have_current_path(dashboard_path)
    expect(page).to have_text('Welcome, John Doe!')
    expect(page).to have_text('Registration successful')
  end
  
  scenario 'แสดง errors เมื่อ validation fails' do
    click_button 'Register'
    
    expect(page).to have_css('.error-message')
    expect(page).to have_text("Name can't be blank")
    expect(page).to have_text("Email can't be blank")
  end
  
  scenario 'ไม่อนุญาต duplicate email' do
    create(:user, email: 'existing@example.com')
    
    fill_in 'Email', with: 'existing@example.com'
    fill_in 'Name', with: 'John'
    fill_in 'Password', with: 'password123'
    fill_in 'Password confirmation', with: 'password123'
    click_button 'Register'
    
    expect(page).to have_text('Email has already been taken')
  end
end
```

---

## 2. Property-based Testing ด้วย Rantly

Property-based testing หาข้อมูล input ที่ทำให้ test ล้มเหลวโดยอัตโนมัติ

### 2.1 การติดตั้ง Rantly

```ruby
# Gemfile
group :test do
  gem 'rantly'
end
```

### 2.2 พื้นฐาน Property-based Testing

```ruby
# spec/models/calculator_spec.rb
require 'rantly/rspec_extensions'

RSpec.describe Calculator do
  # Property: การบวกเป็น commutative
  it 'addition is commutative' do
    property_of {
      a = integer
      b = integer
      [a, b]
    }.check(100) { |a, b|
      expect(Calculator.add(a, b)).to eq(Calculator.add(b, a))
    }
  end
  
  # Property: การบวก 0 ไม่เปลี่ยนค่า
  it 'adding zero returns the same number' do
    property_of { integer }.check(100) { |n|
      expect(Calculator.add(n, 0)).to eq(n)
    }
  end
  
  # Property: การคูณด้วย 1 ไม่เปลี่ยนค่า
  it 'multiplying by 1 returns the same number' do
    property_of { integer }.check(100) { |n|
      expect(Calculator.multiply(n, 1)).to eq(n)
    }
  end
  
  # Property: reverse ของ reverse คือ original
  it 'reverse of reverse is identity' do
    property_of { 
      array(20) { integer(100) }
    }.check(100) { |arr|
      expect(arr.reverse.reverse).to eq(arr)
    }
  end
end
```

### 2.3 Custom Generators

```ruby
# spec/support/generators/user_generator.rb
module UserGenerator
  def user_attributes
    {
      name: sized(rand(2..50)) { string(:alpha) }.join,
      email: "#{sized(5) { string(:alpha) }.join}@#{sized(4) { string(:alpha) }.join}.com",
      age: integer(18..120)
    }
  end
  
  def valid_password
    sized(rand(8..64)) { string }.join
  end
end

# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  include UserGenerator
  
  describe 'email normalization' do
    it 'email เสมอ lowercase หลัง save' do
      property_of {
        # สร้าง random email ที่มีทั้ง upper และ lower case
        "#{sized(5) { string(:alpha) }.join.upcase}@example.com"
      }.check(50) { |email|
        user = create(:user, email: email)
        expect(user.reload.email).to eq(email.downcase)
      }
    end
  end
  
  describe 'name length validation' do
    it 'ยอมรับ name ที่มีความยาว 2-100 ตัวอักษร' do
      property_of {
        length = integer(2..100)
        sized(length) { string(:alpha) }.join
      }.check(50) { |name|
        user = build(:user, name: name)
        expect(user).to be_valid, "Expected name '#{name}' (length: #{name.length}) to be valid"
      }
    end
    
    it 'ปฏิเสธ name ที่สั้นหรือยาวเกิน' do
      property_of {
        branch(
          1 => sized(rand(0..1)) { string(:alpha) }.join,    # Too short
          1 => sized(rand(101..200)) { string(:alpha) }.join  # Too long
        )
      }.check(50) { |name|
        user = build(:user, name: name)
        expect(user).not_to be_valid
      }
    end
  end
end
```

### 2.4 Property Testing สำหรับ Service Objects

```ruby
# spec/services/price_calculator_spec.rb
RSpec.describe PriceCalculator do
  describe 'pricing invariants' do
    # Price เสมอบวก
    it 'price is always positive' do
      property_of {
        quantity = integer(1..1000)
        unit_price = float * 10000
        discount_rate = float(0..1)
        [quantity, unit_price.abs, discount_rate]
      }.check(100) { |quantity, unit_price, discount_rate|
        price = PriceCalculator.calculate(
          quantity: quantity,
          unit_price: unit_price,
          discount_rate: discount_rate
        )
        expect(price).to be >= 0
      }
    end
    
    # Discount ลด price
    it 'discount always reduces price' do
      property_of {
        quantity = integer(1..100)
        unit_price = float(1..1000)
        discount = float(0.01..0.99)  # 1% ถึง 99%
        [quantity, unit_price, discount]
      }.check(100) { |quantity, unit_price, discount|
        no_discount = PriceCalculator.calculate(
          quantity: quantity, 
          unit_price: unit_price, 
          discount_rate: 0
        )
        with_discount = PriceCalculator.calculate(
          quantity: quantity,
          unit_price: unit_price,
          discount_rate: discount
        )
        
        expect(with_discount).to be <= no_discount
      }
    end
  end
end
```

---

## 3. Mutation Testing ด้วย Mutant

Mutation testing ตรวจสอบว่า tests ของเราดีพอที่จะจับ bugs ได้หรือไม่

### 3.1 การติดตั้ง Mutant

```ruby
# Gemfile
group :test do
  gem 'mutant-rspec'
end
```

```bash
# รัน mutation testing
bundle exec mutant run --include lib --require my_app --use rspec "MyApp::Calculator"

# รัน สำหรับ specific method
bundle exec mutant run --include app --require rails --use rspec "User#valid_email?"
```

### 3.2 ทำความเข้าใจ Mutation Results

```ruby
# app/models/user.rb
class User
  def valid_age?(age)
    age >= 18 && age <= 120  # Mutation target
  end
end

# Mutant จะสร้าง mutations เช่น:
# Mutation 1: age > 18  (>= เปลี่ยนเป็น >)
# Mutation 2: age >= 19 (18 เปลี่ยนเป็น 19)
# Mutation 3: age <= 120 (เปลี่ยนเป็น age < 120)
# Mutation 4: age >= 18 || age <= 120 (เปลี่ยน && เป็น ||)

# Tests ที่ดีต้องจับ mutations เหล่านี้ได้
# spec/models/user_spec.rb
RSpec.describe User do
  describe '#valid_age?' do
    # Test boundary cases เพื่อจับ mutations
    it 'อนุญาต age 18' do
      expect(subject.valid_age?(18)).to be true
    end
    
    it 'ไม่อนุญาต age 17' do
      expect(subject.valid_age?(17)).to be false  # จับ mutation >=18 vs >18
    end
    
    it 'อนุญาต age 120' do
      expect(subject.valid_age?(120)).to be true
    end
    
    it 'ไม่อนุญาต age 121' do
      expect(subject.valid_age?(121)).to be false  # จับ mutation <=120 vs <120
    end
    
    it 'อนุญาต age 65' do
      expect(subject.valid_age?(65)).to be true
    end
    
    it 'ไม่อนุญาต age 0' do
      expect(subject.valid_age?(0)).to be false  # จับ mutation && vs ||
    end
  end
end
```

### 3.3 Mutation Testing สำหรับ Complex Logic

```ruby
# app/services/discount_calculator.rb
class DiscountCalculator
  TIERS = {
    vip: { min_spend: 10_000, rate: 0.20 },
    gold: { min_spend: 5_000, rate: 0.15 },
    silver: { min_spend: 2_000, rate: 0.10 },
    basic: { min_spend: 0, rate: 0.05 }
  }.freeze
  
  def self.calculate(user, total_spend)
    tier = determine_tier(total_spend)
    apply_discount(total_spend, tier[:rate])
  end
  
  def self.determine_tier(spend)
    TIERS.values.sort_by { |t| -t[:min_spend] }.find do |tier|
      spend >= tier[:min_spend]
    end
  end
  
  def self.apply_discount(amount, rate)
    amount * (1 - rate)
  end
end

# spec/services/discount_calculator_spec.rb
# Tests ที่ครอบคลุม mutations
RSpec.describe DiscountCalculator do
  describe '.determine_tier' do
    # Exact boundary testing - สำคัญมากสำหรับ mutation testing
    [
      [9_999, :gold, 0.15],   # ต่ำกว่า VIP threshold
      [10_000, :vip, 0.20],   # ถึง VIP threshold พอดี
      [10_001, :vip, 0.20],   # เหนือ VIP threshold
      [4_999, :silver, 0.10], # ต่ำกว่า gold threshold
      [5_000, :gold, 0.15],   # ถึง gold threshold พอดี
      [1_999, :basic, 0.05],  # ต่ำกว่า silver threshold
      [2_000, :silver, 0.10], # ถึง silver threshold พอดี
      [0, :basic, 0.05],      # ขั้นต่ำ
      [1, :basic, 0.05]       # ต่ำกว่า threshold ทุก tier
    ].each do |spend, expected_tier, expected_rate|
      it "spend #{spend} อยู่ใน #{expected_tier} tier (rate: #{expected_rate})" do
        tier = DiscountCalculator.determine_tier(spend)
        expect(tier[:rate]).to eq(expected_rate)
      end
    end
  end
  
  describe '.apply_discount' do
    it '20% discount ลด 20%' do
      result = DiscountCalculator.apply_discount(1000, 0.20)
      expect(result).to eq(800)
    end
    
    it '0% discount ไม่ลด' do
      result = DiscountCalculator.apply_discount(1000, 0)
      expect(result).to eq(1000)
    end
    
    it '100% discount ลดเหลือ 0' do
      result = DiscountCalculator.apply_discount(1000, 1.0)
      expect(result).to eq(0)
    end
  end
end
```

### 3.4 Mutant Configuration

```ruby
# .mutant.yml
integration: rspec
require:
  - ./config/environment

subjects:
  - User
  - UserRegistrationService
  - PriceCalculator
  - DiscountCalculator

includes:
  - app/models
  - app/services

ignore_subjects:
  - "User#to_s"
  - "User.human_attribute_name"

fail_fast: false

jobs: 4  # parallel jobs
```

---

## 4. Load Testing ด้วย k6

k6 เป็น tool สำหรับ load testing ที่เขียนด้วย JavaScript

### 4.1 การติดตั้ง k6

```bash
# macOS
brew install k6

# Ubuntu
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

### 4.2 Basic Load Test Script

```javascript
// tests/load/api_load_test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Counter, Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorCount = new Counter('errors');
const loginRate = new Rate('login_success_rate');
const apiLatency = new Trend('api_latency');

export const options = {
  stages: [
    { duration: '1m', target: 10 },   // Ramp up
    { duration: '3m', target: 50 },   // Stay at 50 users
    { duration: '1m', target: 100 },  // Peak load
    { duration: '2m', target: 100 },  // Stay at peak
    { duration: '1m', target: 0 },    // Ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],  // 95% < 500ms
    'http_req_failed': ['rate<0.01'],                   // < 1% errors
    'login_success_rate': ['rate>0.99'],                // > 99% success
    'errors': ['count<10'],                             // < 10 total errors
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export function setup() {
  // สร้าง test data
  const res = http.post(`${BASE_URL}/api/v1/auth/login`, JSON.stringify({
    email: 'admin@example.com',
    password: 'password123'
  }), { headers: { 'Content-Type': 'application/json' } });
  
  const token = res.json('data.token');
  return { token };
}

export default function(data) {
  const { token } = data;
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  };
  
  group('Users API', function() {
    // List users
    const listRes = http.get(`${BASE_URL}/api/v1/users`, { headers });
    
    check(listRes, {
      'list users: status 200': (r) => r.status === 200,
      'list users: has data': (r) => r.json('data') !== null,
      'list users: response time OK': (r) => r.timings.duration < 300
    });
    
    apiLatency.add(listRes.timings.duration, { endpoint: 'list_users' });
    
    if (listRes.status !== 200) {
      errorCount.add(1);
      console.error(`List users failed: ${listRes.status} ${listRes.body}`);
    }
    
    sleep(0.5);
    
    // Get single user
    const userId = Math.floor(Math.random() * 100) + 1;
    const getRes = http.get(`${BASE_URL}/api/v1/users/${userId}`, { headers });
    
    check(getRes, {
      'get user: status 200 or 404': (r) => [200, 404].includes(r.status),
    });
    
    sleep(0.3);
  });
  
  group('Products API', function() {
    const searchRes = http.get(
      `${BASE_URL}/api/v1/products?q=test&page=1&per_page=20`,
      { headers }
    );
    
    check(searchRes, {
      'search products: status 200': (r) => r.status === 200,
      'search products: has results': (r) => r.json('data') !== null,
      'search products: < 500ms': (r) => r.timings.duration < 500
    });
    
    apiLatency.add(searchRes.timings.duration, { endpoint: 'search_products' });
    
    sleep(1);
  });
  
  group('Auth', function() {
    const loginRes = http.post(
      `${BASE_URL}/api/v1/auth/login`,
      JSON.stringify({
        email: `user${Math.floor(Math.random() * 1000)}@example.com`,
        password: 'password123'
      }),
      { headers: { 'Content-Type': 'application/json' } }
    );
    
    const loginSuccess = loginRes.status === 200;
    loginRate.add(loginSuccess);
    
    if (!loginSuccess && loginRes.status !== 401) {
      errorCount.add(1);
    }
    
    sleep(1);
  });
}

export function teardown(data) {
  console.log('Load test completed');
}
```

### 4.3 Stress Test และ Spike Test

```javascript
// tests/load/stress_test.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    // Stress test - เพิ่ม load จนระบบเริ่มล้มเหลว
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 300 },  // จุดที่ระบบเริ่มล้มเหลว
    { duration: '5m', target: 300 },
    { duration: '2m', target: 0 },
  ],
};

// tests/load/spike_test.js
export const options = {
  stages: [
    // Spike test - ทดสอบ sudden traffic surge
    { duration: '1m', target: 10 },   // Baseline
    { duration: '30s', target: 500 }, // Spike!
    { duration: '1m', target: 10 },   // Recovery
    { duration: '30s', target: 1000 }, // Bigger spike!
    { duration: '1m', target: 10 },   // Recovery
  ],
};
```

### 4.4 Rails Performance Test Setup

```ruby
# config/environments/load_test.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load = true
  config.consider_all_requests_local = false
  
  # ใช้ Redis สำหรับ cache
  config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }
  
  # Log performance metrics
  config.log_level = :warn
  
  # Database connection pool
  config.active_record_connection_pool = 25
end

# lib/tasks/load_test.rake
namespace :load_test do
  desc 'Seed data for load testing'
  task seed: :environment do
    puts 'Creating load test data...'
    
    # สร้าง users
    User.find_or_create_by(email: 'admin@example.com') do |u|
      u.name = 'Admin User'
      u.password = 'password123'
      u.role = 'admin'
    end
    
    # สร้าง test users
    100.times do |i|
      User.find_or_create_by(email: "user#{i}@example.com") do |u|
        u.name = "Test User #{i}"
        u.password = 'password123'
        u.role = 'user'
      end
    end
    
    puts 'Done!'
  end
  
  desc 'Run k6 load test'
  task run: :environment do
    system('k6 run tests/load/api_load_test.js --env BASE_URL=http://localhost:3000')
  end
end
```

---

## 5. Consumer-driven Contract Testing

### 5.1 Pact Consumer Test

```ruby
# spec/pacts/order_service_consumer_spec.rb
require 'pact/consumer/rspec'

Pact.service_consumer 'OrderService' do
  has_pact_with 'PaymentService' do
    mock_service :payment_service do
      port 1235
      pact_specification_version '3'
    end
  end
end

RSpec.describe 'OrderService - PaymentService Contract', pact: true do
  subject(:payment_client) { PaymentServiceClient.new('http://localhost:1235') }
  
  describe 'charging a payment' do
    before do
      payment_service
        .given('a valid payment method exists for customer 123')
        .upon_receiving('a charge request')
        .with(
          method: :post,
          path: '/api/v1/charges',
          headers: {
            'Content-Type' => 'application/json',
            'Authorization' => Pact.term(
              matcher: /^Bearer .+$/,
              generate: 'Bearer valid_token'
            )
          },
          body: {
            customer_id: Pact.like(123),
            amount: Pact.like(1500),
            currency: 'THB',
            description: Pact.like('Order #001'),
            idempotency_key: Pact.like('uuid-123')
          }
        )
        .will_respond_with(
          status: 201,
          headers: { 'Content-Type' => 'application/json' },
          body: {
            charge_id: Pact.like('ch_123'),
            status: Pact.term(matcher: /^(succeeded|pending|failed)$/, generate: 'succeeded'),
            amount: Pact.like(1500),
            currency: 'THB'
          }
        )
    end
    
    it 'charges the customer successfully' do
      result = payment_client.charge(
        customer_id: 123,
        amount: 1500,
        currency: 'THB',
        description: 'Order #001',
        idempotency_key: 'uuid-123'
      )
      
      expect(result[:status]).to eq('succeeded')
      expect(result[:charge_id]).to be_present
    end
  end
  
  describe 'handling insufficient funds' do
    before do
      payment_service
        .given('customer 456 has insufficient funds')
        .upon_receiving('a charge request that will fail')
        .with(
          method: :post,
          path: '/api/v1/charges',
          body: {
            customer_id: 456,
            amount: Pact.like(5000),
            currency: 'THB'
          }
        )
        .will_respond_with(
          status: 402,
          body: {
            error: {
              code: 'insufficient_funds',
              message: Pact.like('Your card has insufficient funds')
            }
          }
        )
    end
    
    it 'raises InsufficientFundsError' do
      expect {
        payment_client.charge(customer_id: 456, amount: 5000, currency: 'THB')
      }.to raise_error(PaymentServiceClient::InsufficientFundsError)
    end
  end
end
```

### 5.2 Pact Provider Test

```ruby
# spec/pacts/payment_service_provider_spec.rb
require 'rails_helper'
require 'pact/provider/rspec'

Pact.service_provider 'PaymentService' do
  honours_pact_with 'OrderService' do
    pact_uri ENV['PACT_URL'] || 
             './spec/pacts/orderservice-paymentservice.json'
  end
end

Pact.provider_states_for 'OrderService' do
  provider_state 'a valid payment method exists for customer 123' do
    set_up do
      customer = Customer.find_or_create_by(external_id: 123) do |c|
        c.name = 'Test Customer'
        c.email = 'test@example.com'
      end
      
      PaymentMethod.find_or_create_by(customer: customer) do |pm|
        pm.type = 'credit_card'
        pm.token = 'tok_visa'
        pm.is_default = true
      end
    end
  end
  
  provider_state 'customer 456 has insufficient funds' do
    set_up do
      customer = Customer.find_or_create_by(external_id: 456) do |c|
        c.name = 'Broke Customer'
        c.email = 'broke@example.com'
      end
      
      # Mock payment gateway to return insufficient funds
      allow(PaymentGateway).to receive(:charge).and_raise(
        PaymentGateway::InsufficientFundsError, 'Your card has insufficient funds'
      )
    end
  end
end

# เพิ่ม request header สำหรับ auth
module Rack
  module Test
    class Session
      def process_request(uri, env)
        # เพิ่ม auth token สำหรับ pact verification
        env['HTTP_AUTHORIZATION'] ||= "Bearer #{PaymentServiceToken.generate}"
        super
      end
    end
  end
end
```

---

## 6. Approval Testing

Approval testing (หรือ Snapshot testing) บันทึก output ของโค้ดและเปรียบเทียบกับครั้งต่อไป

### 6.1 การติดตั้ง Approvals

```ruby
# Gemfile
group :test do
  gem 'approvals'
end
```

### 6.2 Basic Approval Tests

```ruby
# spec/services/report_generator_spec.rb
require 'approvals/rspec'

RSpec.describe ReportGenerator do
  describe '#generate_monthly_report' do
    let(:year) { 2024 }
    let(:month) { 6 }
    let(:data) do
      {
        total_orders: 150,
        total_revenue: 75_000.00,
        top_products: [
          { name: 'Product A', sales: 45 },
          { name: 'Product B', sales: 30 },
          { name: 'Product C', sales: 25 }
        ],
        daily_breakdown: generate_daily_data(year, month)
      }
    end
    
    it 'generates correct HTML report' do
      report = ReportGenerator.new(data).generate_html
      
      verify(format: :html) { report }
    end
    
    it 'generates correct JSON report' do
      report = ReportGenerator.new(data).generate_json
      
      verify(format: :json) { report }
    end
    
    it 'generates correct CSV export' do
      csv = ReportGenerator.new(data).generate_csv
      
      verify(format: :txt) { csv }
    end
    
    private
    
    def generate_daily_data(year, month)
      days_in_month = Date.new(year, month, -1).day
      (1..days_in_month).map do |day|
        { date: "#{year}-#{month.to_s.rjust(2, '0')}-#{day.to_s.rjust(2, '0')}",
          orders: rand(1..20),
          revenue: rand(100..5000) }
      end
    end
  end
end
```

### 6.3 Approval Testing สำหรับ API Responses

```ruby
# spec/requests/api/snapshot_spec.rb
require 'approvals/rspec'

RSpec.describe 'API Response Snapshots', type: :request do
  let(:user) { create(:user, id: 1, name: 'Snapshot User', email: 'snapshot@example.com') }
  let(:token) { JwtService.encode(user_id: user.id) }
  let(:headers) { { 'Authorization' => "Bearer #{token}" } }
  
  before do
    # Seed deterministic data
    travel_to Time.zone.parse('2024-01-15 10:00:00')
    create(:product, id: 1, name: 'Product One', price: 100)
    create(:product, id: 2, name: 'Product Two', price: 200)
  end
  
  after { travel_back }
  
  it 'products index response matches snapshot' do
    get '/api/v1/products', headers: headers
    
    verify(format: :json) { response.body }
  end
  
  it 'single product response matches snapshot' do
    get '/api/v1/products/1', headers: headers
    
    verify(format: :json) { response.body }
  end
end
```

### 6.4 Custom Approval Format

```ruby
# spec/support/approvals_setup.rb
require 'approvals'

Approvals.configure do |c|
  c.approvals_path = 'spec/fixtures/approvals'
end

# Custom namer สำหรับ better file naming
RSpec.configure do |config|
  config.before(:each, :approval) do |example|
    Approvals.namer = Approvals::Namers::RSpecNamer.new(example)
  end
end

# Helper method
module ApprovalHelpers
  def verify_json_response(response, &block)
    parsed = JSON.parse(response.body)
    
    # Remove dynamic fields
    sanitized = deep_remove_keys(parsed, ['created_at', 'updated_at', 'token'])
    
    verify(format: :json) { sanitized.to_json }
  end
  
  def deep_remove_keys(hash, keys_to_remove)
    case hash
    when Hash
      hash.reject { |k, _| keys_to_remove.include?(k.to_s) }
          .transform_values { |v| deep_remove_keys(v, keys_to_remove) }
    when Array
      hash.map { |v| deep_remove_keys(v, keys_to_remove) }
    else
      hash
    end
  end
end
```

---

## 7. Advanced Testing Patterns

### 7.1 Test Data Builder Pattern

```ruby
# spec/support/builders/order_builder.rb
class OrderBuilder
  def initialize
    @attributes = {
      status: 'pending',
      user: nil,
      items: [],
      coupon: nil,
      shipping_address: default_address
    }
  end
  
  def for_user(user)
    @attributes[:user] = user
    self
  end
  
  def with_status(status)
    @attributes[:status] = status
    self
  end
  
  def with_items(items)
    @attributes[:items] = items
    self
  end
  
  def with_item(product, quantity: 1)
    @attributes[:items] << { product: product, quantity: quantity }
    self
  end
  
  def with_coupon(code)
    @attributes[:coupon] = Coupon.find_by(code: code) || create(:coupon, code: code)
    self
  end
  
  def shipped_to(address)
    @attributes[:shipping_address] = address
    self
  end
  
  def build
    user = @attributes[:user] || create(:user)
    
    order = Order.new(
      user: user,
      status: @attributes[:status],
      shipping_address: @attributes[:shipping_address],
      coupon: @attributes[:coupon]
    )
    
    @attributes[:items].each do |item|
      order.order_items.build(
        product: item[:product],
        quantity: item[:quantity],
        unit_price: item[:product].price
      )
    end
    
    order.save!
    order
  end
  
  def create
    build
  end
  
  private
  
  def default_address
    {
      street: '123 Test St',
      city: 'Bangkok',
      zip: '10100',
      country: 'TH'
    }
  end
end

# การใช้งาน
RSpec.describe OrderProcessingService do
  let(:user) { create(:user) }
  let(:product) { create(:product, price: 500) }
  
  let(:order) do
    OrderBuilder.new
      .for_user(user)
      .with_item(product, quantity: 2)
      .with_coupon('SAVE10')
      .build
  end
  
  it 'processes order correctly' do
    result = OrderProcessingService.process(order)
    expect(result).to be_success
  end
end
```

### 7.2 Object Mother Pattern

```ruby
# spec/support/mothers/order_mother.rb
module OrderMother
  def self.simple_order(user: nil)
    user ||= UserMother.regular_user
    product = ProductMother.in_stock_product
    
    create(:order,
           user: user,
           status: 'confirmed',
           order_items: [
             build(:order_item, product: product, quantity: 1)
           ])
  end
  
  def self.paid_order(user: nil)
    order = simple_order(user: user)
    order.update!(status: 'paid', paid_at: Time.current)
    create(:payment, order: order, status: 'succeeded')
    order
  end
  
  def self.large_order(user: nil, item_count: 10)
    user ||= UserMother.premium_user
    
    items = item_count.times.map do
      product = create(:product, price: rand(100..1000))
      build(:order_item, product: product, quantity: rand(1..5))
    end
    
    create(:order, user: user, status: 'confirmed', order_items: items)
  end
  
  def self.international_order(user: nil, country: 'US')
    user ||= UserMother.regular_user
    order = simple_order(user: user)
    order.update!(
      shipping_address: {
        street: '123 Broadway',
        city: 'New York',
        state: 'NY',
        zip: '10001',
        country: country
      }
    )
    order
  end
end

module UserMother
  def self.regular_user
    create(:user, role: 'user', plan: 'free')
  end
  
  def self.premium_user
    create(:user, role: 'user', plan: 'premium')
  end
  
  def self.admin_user
    create(:user, role: 'admin')
  end
end
```

### 7.3 Golden Master Testing

```ruby
# spec/support/golden_master_testing.rb
module GoldenMasterTesting
  GOLDEN_MASTER_DIR = Rails.root.join('spec', 'fixtures', 'golden_master')
  
  def save_golden_master(name, content)
    FileUtils.mkdir_p(GOLDEN_MASTER_DIR)
    File.write(GOLDEN_MASTER_DIR.join("#{name}.txt"), content)
  end
  
  def verify_golden_master(name, content)
    master_file = GOLDEN_MASTER_DIR.join("#{name}.txt")
    
    if master_file.exist?
      expected = File.read(master_file)
      expect(content).to eq(expected), 
        "Golden master mismatch for '#{name}'.\n" \
        "Run with UPDATE_GOLDEN_MASTERS=1 to update."
    elsif ENV['UPDATE_GOLDEN_MASTERS'] == '1'
      save_golden_master(name, content)
      puts "Created golden master: #{name}"
    else
      raise "Golden master '#{name}' not found. Run with UPDATE_GOLDEN_MASTERS=1 to create."
    end
  end
end

# การใช้งาน
RSpec.describe LegacyReportService do
  include GoldenMasterTesting
  
  it 'generates report matching golden master' do
    report = LegacyReportService.generate(
      start_date: Date.new(2024, 1, 1),
      end_date: Date.new(2024, 1, 31)
    )
    
    verify_golden_master('january_2024_report', report)
  end
end
```

### 7.4 Test Coverage Analysis

```ruby
# .simplecov
require 'simplecov'
require 'simplecov-html'
require 'simplecov_json_formatter'

SimpleCov.configure do
  formatter SimpleCov::Formatter::MultiFormatter.new([
    SimpleCov::Formatter::HTMLFormatter,
    SimpleCov::Formatter::JSONFormatter
  ])
  
  add_group 'Models', 'app/models'
  add_group 'Controllers', 'app/controllers'
  add_group 'Services', 'app/services'
  add_group 'Jobs', 'app/jobs'
  add_group 'Mailers', 'app/mailers'
  add_group 'Serializers', 'app/serializers'
  add_group 'Lib', 'lib'
  
  add_filter '/spec/'
  add_filter '/config/'
  add_filter '/db/'
  add_filter '/vendor/'
  
  minimum_coverage 85  # ต้องมี coverage อย่างน้อย 85%
  minimum_coverage_by_file 70  # แต่ละ file ต้องมี coverage 70%
  
  refuse_coverage_drop 0.5  # ไม่อนุญาตให้ coverage ลดเกิน 0.5%
end

SimpleCov.start 'rails'
```

---

## แบบฝึกหัด (20 ข้อพร้อมเฉลย)

### ข้อ 1: Unit Test สำหรับ Model
**คำถาม:** เขียน unit tests ครบถ้วนสำหรับ Product model ที่มี validates presence, numericality, และ custom validation ตรวจสอบว่า price > cost

**เฉลย:**
```ruby
# app/models/product.rb
class Product < ApplicationRecord
  validates :name, presence: true, length: { minimum: 2, maximum: 200 }
  validates :price, presence: true, numericality: { greater_than: 0 }
  validates :cost, numericality: { greater_than_or_equal_to: 0 }, allow_nil: true
  validate :price_must_be_greater_than_cost
  
  def margin
    return nil unless cost.present? && cost > 0
    ((price - cost) / price * 100).round(2)
  end
  
  private
  
  def price_must_be_greater_than_cost
    return unless price.present? && cost.present?
    errors.add(:price, 'must be greater than cost') unless price > cost
  end
end

# spec/models/product_spec.rb
RSpec.describe Product, type: :model do
  subject(:product) { build(:product) }
  
  describe 'validations' do
    it { is_expected.to validate_presence_of(:name) }
    it { is_expected.to validate_length_of(:name).is_at_least(2).is_at_most(200) }
    it { is_expected.to validate_presence_of(:price) }
    it { is_expected.to validate_numericality_of(:price).is_greater_than(0) }
    
    describe 'price vs cost validation' do
      it 'valid เมื่อ price > cost' do
        product.price = 100
        product.cost = 50
        expect(product).to be_valid
      end
      
      it 'invalid เมื่อ price == cost' do
        product.price = 100
        product.cost = 100
        expect(product).not_to be_valid
        expect(product.errors[:price]).to include('must be greater than cost')
      end
      
      it 'invalid เมื่อ price < cost' do
        product.price = 50
        product.cost = 100
        expect(product).not_to be_valid
      end
      
      it 'valid เมื่อ cost เป็น nil' do
        product.price = 100
        product.cost = nil
        expect(product).to be_valid
      end
    end
  end
  
  describe '#margin' do
    it 'คำนวณ margin ถูกต้อง' do
      product.price = 100
      product.cost = 60
      expect(product.margin).to eq(40.0)
    end
    
    it 'คืน nil เมื่อไม่มี cost' do
      product.cost = nil
      expect(product.margin).to be_nil
    end
    
    it 'คืน nil เมื่อ cost เป็น 0' do
      product.cost = 0
      expect(product.margin).to be_nil
    end
    
    it 'handle รายการที่มี margin สูง' do
      product.price = 1000
      product.cost = 100
      expect(product.margin).to eq(90.0)
    end
  end
end
```

### ข้อ 2: Property-based Test
**คำถาม:** เขียน property-based test สำหรับ function ที่แปลง string เป็น slug (downcase, replace spaces with -)

**เฉลย:**
```ruby
# app/helpers/slug_helper.rb
module SlugHelper
  def slugify(string)
    string.to_s
          .downcase
          .strip
          .gsub(/[^\w\s-]/, '')
          .gsub(/[\s_]+/, '-')
          .gsub(/^-+|-+$/, '')
  end
end

# spec/helpers/slug_helper_spec.rb
require 'rantly/rspec_extensions'

RSpec.describe SlugHelper do
  include SlugHelper
  
  describe '#slugify' do
    # Property: slug เสมอ lowercase
    it 'result is always lowercase' do
      property_of {
        sized(rand(1..50)) { string(:alpha) }.join
      }.check(100) { |str|
        expect(slugify(str)).to eq(slugify(str).downcase)
      }
    end
    
    # Property: slug ไม่มี leading/trailing dashes
    it 'never has leading or trailing dashes' do
      property_of {
        sized(rand(1..50)) { string }.join
      }.check(100) { |str|
        result = slugify(str)
        expect(result).not_to start_with('-') unless result.empty?
        expect(result).not_to end_with('-') unless result.empty?
      }
    end
    
    # Property: idempotent - slugify ซ้ำได้ผลเหมือนเดิม
    it 'is idempotent' do
      property_of {
        sized(rand(1..50)) { string(:alpha) }.join
      }.check(100) { |str|
        once = slugify(str)
        twice = slugify(once)
        expect(once).to eq(twice)
      }
    end
    
    # Property: ไม่มี spaces
    it 'never contains spaces' do
      property_of {
        sized(rand(1..50)) { string }.join
      }.check(100) { |str|
        expect(slugify(str)).not_to include(' ')
      }
    end
  end
end
```

### ข้อ 3: Mutation Testing
**คำถาม:** เขียน tests ที่ครอบคลุม mutations ทั้งหมดสำหรับ method ที่ check eligibility สำหรับ discount

**เฉลย:**
```ruby
# app/services/discount_eligibility_checker.rb
class DiscountEligibilityChecker
  MIN_ORDER_COUNT = 3
  MIN_TOTAL_SPEND = 1000
  MAX_DAYS_SINCE_LAST_ORDER = 90
  
  def eligible?(user)
    user.orders.count >= MIN_ORDER_COUNT &&
      user.total_spend >= MIN_TOTAL_SPEND &&
      user.last_order_date >= MAX_DAYS_SINCE_LAST_ORDER.days.ago &&
      !user.blacklisted?
  end
end

# spec/services/discount_eligibility_checker_spec.rb
RSpec.describe DiscountEligibilityChecker do
  subject(:checker) { described_class.new }
  
  let(:user) { instance_double(User) }
  let(:orders_relation) { double('orders') }
  
  before do
    allow(user).to receive(:orders).and_return(orders_relation)
    allow(orders_relation).to receive(:count).and_return(5)
    allow(user).to receive(:total_spend).and_return(2000)
    allow(user).to receive(:last_order_date).and_return(30.days.ago)
    allow(user).to receive(:blacklisted?).and_return(false)
  end
  
  context 'eligible user' do
    it 'คืน true' do
      expect(checker.eligible?(user)).to be true
    end
  end
  
  # Test boundary: MIN_ORDER_COUNT (จับ mutations >=3 vs >3, 3 vs 4)
  context 'order count boundaries' do
    it 'eligible เมื่อมี 3 orders (minimum)' do
      allow(orders_relation).to receive(:count).and_return(3)
      expect(checker.eligible?(user)).to be true
    end
    
    it 'not eligible เมื่อมี 2 orders' do
      allow(orders_relation).to receive(:count).and_return(2)
      expect(checker.eligible?(user)).to be false
    end
    
    it 'eligible เมื่อมี 100 orders' do
      allow(orders_relation).to receive(:count).and_return(100)
      expect(checker.eligible?(user)).to be true
    end
  end
  
  # Test boundary: MIN_TOTAL_SPEND
  context 'total spend boundaries' do
    it 'eligible เมื่อ spend 1000 (minimum)' do
      allow(user).to receive(:total_spend).and_return(1000)
      expect(checker.eligible?(user)).to be true
    end
    
    it 'not eligible เมื่อ spend 999' do
      allow(user).to receive(:total_spend).and_return(999)
      expect(checker.eligible?(user)).to be false
    end
  end
  
  # Test boundary: MAX_DAYS_SINCE_LAST_ORDER
  context 'last order date boundaries' do
    it 'eligible เมื่อ last order 90 days ago' do
      allow(user).to receive(:last_order_date).and_return(90.days.ago)
      expect(checker.eligible?(user)).to be true
    end
    
    it 'not eligible เมื่อ last order 91 days ago' do
      allow(user).to receive(:last_order_date).and_return(91.days.ago)
      expect(checker.eligible?(user)).to be false
    end
  end
  
  # Test: blacklist condition (จับ mutation ! vs เอา ! ออก)
  context 'blacklisted user' do
    it 'not eligible เมื่อ user is blacklisted' do
      allow(user).to receive(:blacklisted?).and_return(true)
      expect(checker.eligible?(user)).to be false
    end
    
    it 'eligible เมื่อ user is not blacklisted' do
      allow(user).to receive(:blacklisted?).and_return(false)
      expect(checker.eligible?(user)).to be true
    end
  end
  
  # Test: AND conditions (จับ mutation && vs ||)
  context 'failing multiple conditions' do
    it 'not eligible เมื่อ fail ทุก condition' do
      allow(orders_relation).to receive(:count).and_return(0)
      allow(user).to receive(:total_spend).and_return(0)
      allow(user).to receive(:last_order_date).and_return(200.days.ago)
      allow(user).to receive(:blacklisted?).and_return(true)
      
      expect(checker.eligible?(user)).to be false
    end
    
    it 'not eligible เมื่อ fail เพียง 1 condition' do
      # ทุก condition pass ยกเว้น blacklisted
      allow(user).to receive(:blacklisted?).and_return(true)
      expect(checker.eligible?(user)).to be false
    end
  end
end
```

### ข้อ 4: Integration Test สำหรับ Service
**คำถาม:** เขียน integration test สำหรับ OrderCreationService ที่ test ว่ามีการสร้าง records ที่เกี่ยวข้องทั้งหมด

**เฉลย:**
```ruby
# spec/services/order_creation_service_spec.rb
RSpec.describe OrderCreationService, type: :service do
  let(:user) { create(:user) }
  let(:products) { create_list(:product, 3) }
  
  let(:params) do
    {
      user: user,
      items: products.map { |p| { product_id: p.id, quantity: 2 } },
      shipping_address: {
        street: '123 Test St',
        city: 'Bangkok',
        zip: '10100'
      },
      payment_method: 'credit_card',
      coupon_code: nil
    }
  end
  
  describe '#call' do
    context 'ด้วย valid params' do
      subject(:result) { described_class.new(params).call }
      
      it 'สร้าง order' do
        expect { result }.to change(Order, :count).by(1)
      end
      
      it 'สร้าง order items' do
        expect { result }.to change(OrderItem, :count).by(3)
      end
      
      it 'สร้าง payment record' do
        expect { result }.to change(Payment, :count).by(1)
      end
      
      it 'คำนวณ total ถูกต้อง' do
        result
        expected_total = products.sum { |p| p.price * 2 }
        expect(Order.last.total).to eq(expected_total)
      end
      
      it 'ส่ง order confirmation email' do
        expect { result }
          .to have_enqueued_mail(OrderMailer, :confirmation)
      end
      
      it 'ลด stock ของ products' do
        initial_stocks = products.map { |p| [p.id, p.stock] }.to_h
        result
        
        products.each do |product|
          expect(product.reload.stock).to eq(initial_stocks[product.id] - 2)
        end
      end
      
      it 'คืน success result' do
        expect(result.success?).to be true
        expect(result.order).to be_a(Order)
        expect(result.order).to be_persisted
      end
    end
    
    context 'เมื่อ product out of stock' do
      before { products.first.update!(stock: 0) }
      
      it 'ไม่สร้าง order' do
        expect { described_class.new(params).call }.not_to change(Order, :count)
      end
      
      it 'คืน error result' do
        result = described_class.new(params).call
        expect(result.success?).to be false
        expect(result.errors).to include(/out of stock/i)
      end
      
      it 'rollback ทั้งหมด' do
        described_class.new(params).call
        expect(OrderItem.count).to eq(0)
        expect(Payment.count).to eq(0)
      end
    end
    
    context 'ด้วย coupon code' do
      let(:coupon) { create(:coupon, code: 'SAVE20', discount_rate: 0.20) }
      let(:params_with_coupon) { params.merge(coupon_code: coupon.code) }
      
      it 'apply discount' do
        result = described_class.new(params_with_coupon).call
        original_total = products.sum { |p| p.price * 2 }
        expect(result.order.total).to eq(original_total * 0.80)
      end
    end
  end
end
```

### ข้อ 5: Load Test Script
**คำถาม:** เขียน k6 script สำหรับ test checkout flow ที่ประกอบด้วย add to cart, view cart, และ checkout

**เฉลย:**
```javascript
// tests/load/checkout_flow_test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const checkoutSuccessRate = new Rate('checkout_success');
const cartDuration = new Trend('cart_duration');
const checkoutDuration = new Trend('checkout_duration');

export const options = {
  stages: [
    { duration: '1m', target: 20 },
    { duration: '3m', target: 50 },
    { duration: '1m', target: 0 }
  ],
  thresholds: {
    'checkout_success': ['rate>0.95'],
    'cart_duration': ['p(95)<500'],
    'checkout_duration': ['p(95)<2000']
  }
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export function setup() {
  // Login
  const loginRes = http.post(`${BASE_URL}/api/v1/auth/login`, JSON.stringify({
    email: 'test@example.com',
    password: 'password123'
  }), { headers: { 'Content-Type': 'application/json' } });
  
  return { token: loginRes.json('data.token') };
}

export default function(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`
  };
  
  let cartId;
  
  group('Add to Cart', function() {
    const productId = Math.floor(Math.random() * 50) + 1;
    const startTime = Date.now();
    
    const addRes = http.post(
      `${BASE_URL}/api/v1/cart/items`,
      JSON.stringify({ product_id: productId, quantity: 1 }),
      { headers }
    );
    
    cartDuration.add(Date.now() - startTime);
    
    check(addRes, {
      'add to cart: status 200 or 201': (r) => [200, 201].includes(r.status)
    });
    
    cartId = addRes.json('data.cart_id');
    sleep(1);
  });
  
  group('View Cart', function() {
    const cartRes = http.get(`${BASE_URL}/api/v1/cart`, { headers });
    
    check(cartRes, {
      'view cart: status 200': (r) => r.status === 200,
      'view cart: has items': (r) => r.json('data.items.length') > 0
    });
    
    sleep(0.5);
  });
  
  group('Checkout', function() {
    const startTime = Date.now();
    
    const checkoutRes = http.post(
      `${BASE_URL}/api/v1/orders`,
      JSON.stringify({
        order: {
          delivery_address: { street: '123 Test', city: 'Bangkok', zip: '10100' },
          payment_method: 'credit_card',
          card_token: 'tok_visa'
        }
      }),
      { headers }
    );
    
    checkoutDuration.add(Date.now() - startTime);
    
    const success = checkoutRes.status === 201;
    checkoutSuccessRate.add(success);
    
    check(checkoutRes, {
      'checkout: status 201': (r) => r.status === 201,
      'checkout: has order_id': (r) => r.json('data.id') !== null
    });
    
    sleep(2);
  });
}
```

### ข้อ 6: Contract Test
**คำถาม:** เขียน Pact contract test สำหรับ inventory service ที่ check stock levels

**เฉลย:**
```ruby
# spec/pacts/inventory_service_consumer_spec.rb
require 'pact/consumer/rspec'

Pact.service_consumer 'OrderService' do
  has_pact_with 'InventoryService' do
    mock_service :inventory_service do
      port 1236
    end
  end
end

RSpec.describe 'Inventory Service Contract', pact: true do
  subject(:client) { InventoryServiceClient.new('http://localhost:1236') }
  
  describe 'checking stock' do
    before do
      inventory_service
        .given('product 123 has 10 units in stock')
        .upon_receiving('a stock check request')
        .with(
          method: :get,
          path: '/api/v1/products/123/stock',
          headers: { 'Accept' => 'application/json' }
        )
        .will_respond_with(
          status: 200,
          body: {
            product_id: Pact.like(123),
            available: Pact.like(10),
            reserved: Pact.like(0),
            location: Pact.like('warehouse-A')
          }
        )
    end
    
    it 'returns stock information' do
      stock = client.check_stock(123)
      expect(stock[:available]).to be_a(Integer)
      expect(stock[:available]).to be >= 0
    end
  end
  
  describe 'reserving stock' do
    before do
      inventory_service
        .given('product 123 has sufficient stock')
        .upon_receiving('a stock reservation request')
        .with(
          method: :post,
          path: '/api/v1/products/123/reservations',
          body: { quantity: Pact.like(5), order_id: Pact.like('order-001') }
        )
        .will_respond_with(
          status: 201,
          body: {
            reservation_id: Pact.like('res-001'),
            product_id: Pact.like(123),
            quantity: Pact.like(5),
            expires_at: Pact.like('2024-01-15T10:00:00Z')
          }
        )
    end
    
    it 'returns reservation confirmation' do
      reservation = client.reserve_stock(123, quantity: 5, order_id: 'order-001')
      expect(reservation[:reservation_id]).to be_present
    end
  end
  
  describe 'handling out of stock' do
    before do
      inventory_service
        .given('product 999 is out of stock')
        .upon_receiving('a reservation request for out of stock product')
        .with(
          method: :post,
          path: '/api/v1/products/999/reservations',
          body: { quantity: Pact.like(1) }
        )
        .will_respond_with(
          status: 409,
          body: {
            error: {
              code: 'insufficient_stock',
              available: 0,
              requested: Pact.like(1)
            }
          }
        )
    end
    
    it 'raises InsufficientStockError' do
      expect {
        client.reserve_stock(999, quantity: 1)
      }.to raise_error(InventoryServiceClient::InsufficientStockError)
    end
  end
end
```

### ข้อ 7: Approval Test
**คำถาม:** เขียน approval test สำหรับ email template generator

**เฉลย:**
```ruby
# spec/mailers/order_mailer_spec.rb
require 'approvals/rspec'

RSpec.describe OrderMailer do
  let(:user) { create(:user, name: 'Test User', email: 'test@example.com') }
  let(:order) do
    create(:order,
           user: user,
           id: 12345,
           total: 1500.00,
           status: 'confirmed',
           created_at: Time.zone.parse('2024-01-15 10:00:00'))
  end
  
  before do
    create(:order_item,
           order: order,
           product: create(:product, name: 'Test Product', price: 500),
           quantity: 3,
           unit_price: 500)
  end
  
  describe '#confirmation' do
    let(:mail) { OrderMailer.confirmation(order) }
    
    it 'email subject matches snapshot' do
      verify { mail.subject }
    end
    
    it 'email text body matches snapshot' do
      # Remove dynamic timestamps
      body = mail.text_part&.body&.to_s || mail.body.to_s
      sanitized = body.gsub(/\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/, '[TIMESTAMP]')
      
      verify(format: :txt) { sanitized }
    end
    
    it 'email HTML body matches snapshot' do
      body = mail.html_part&.body&.to_s || mail.body.to_s
      sanitized = body.gsub(/\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/, '[TIMESTAMP]')
      
      verify(format: :html) { sanitized }
    end
    
    it 'คืน email พื้นฐาน' do
      expect(mail.to).to eq(['test@example.com'])
      expect(mail.from).to eq([Rails.application.credentials.mailer_from])
    end
  end
end
```

### ข้อ 8: Shared Examples
**คำถาม:** สร้าง shared examples สำหรับ CRUD operations ที่ใช้ได้กับหลาย resources

**เฉลย:**
```ruby
# spec/support/shared_examples/crud_operations.rb
RSpec.shared_examples 'CRUD operations' do |resource_name, factory_name|
  let(:resource) { create(factory_name) }
  let(:admin) { create(:user, role: 'admin') }
  let(:token) { JwtService.encode(user_id: admin.id) }
  let(:auth_headers) do
    { 'Authorization' => "Bearer #{token}", 'Content-Type' => 'application/json' }
  end
  
  describe "GET /api/v1/#{resource_name.pluralize}" do
    before { create_list(factory_name, 3) }
    
    it 'คืน list' do
      get "/api/v1/#{resource_name.pluralize}", headers: auth_headers
      expect(response).to have_http_status(:ok)
      expect(JSON.parse(response.body)['data']).to be_an(Array)
    end
    
    it 'รองรับ pagination' do
      get "/api/v1/#{resource_name.pluralize}",
          params: { page: 1, per_page: 2 },
          headers: auth_headers
      
      json = JSON.parse(response.body)
      expect(json['meta']['per_page']).to eq(2)
    end
  end
  
  describe "GET /api/v1/#{resource_name.pluralize}/:id" do
    it 'คืน resource' do
      get "/api/v1/#{resource_name.pluralize}/#{resource.id}", headers: auth_headers
      expect(response).to have_http_status(:ok)
    end
    
    it 'คืน 404 เมื่อไม่พบ' do
      get "/api/v1/#{resource_name.pluralize}/999999", headers: auth_headers
      expect(response).to have_http_status(:not_found)
    end
  end
  
  describe "DELETE /api/v1/#{resource_name.pluralize}/:id" do
    it 'ลบ resource' do
      id = resource.id
      delete "/api/v1/#{resource_name.pluralize}/#{id}", headers: auth_headers
      expect(response).to have_http_status(:no_content)
      expect(resource.class.find_by(id: id)).to be_nil
    end
  end
end

# การใช้งาน
RSpec.describe 'Products', type: :request do
  include_examples 'CRUD operations', 'product', :product
end

RSpec.describe 'Categories', type: :request do
  include_examples 'CRUD operations', 'category', :category
end
```

### ข้อ 9: Test Doubles
**คำถาม:** เขียน test ที่ใช้ mocks และ stubs อย่างถูกต้องสำหรับ external service calls

**เฉลย:**
```ruby
# spec/services/sms_notification_service_spec.rb
RSpec.describe SmsNotificationService do
  let(:user) { create(:user, phone: '+66891234567') }
  let(:sms_gateway) { instance_double(TwilioClient) }
  
  before do
    allow(TwilioClient).to receive(:new).and_return(sms_gateway)
  end
  
  describe '#send_otp' do
    context 'ส่ง SMS สำเร็จ' do
      before do
        allow(sms_gateway).to receive(:send_message).and_return(
          OpenStruct.new(sid: 'SM123', status: 'queued', error_code: nil)
        )
      end
      
      it 'เรียก Twilio API' do
        expect(sms_gateway).to receive(:send_message).with(
          to: user.phone,
          from: Rails.application.credentials.twilio.phone_number,
          body: /^\d{6}$/  # OTP format
        )
        
        described_class.new(user).send_otp
      end
      
      it 'บันทึก OTP ใน redis' do
        described_class.new(user).send_otp
        
        expect(Rails.cache.read("otp:#{user.id}")).to be_present
      end
      
      it 'OTP หมดอายุใน 10 นาที' do
        described_class.new(user).send_otp
        
        expect(Rails.cache.read("otp:#{user.id}", raw: true)).to include(
          a_hash_including(expires_in: 600)
        )
      end
    end
    
    context 'เมื่อ Twilio error' do
      before do
        allow(sms_gateway).to receive(:send_message).and_raise(
          Twilio::REST::RestError.new('Unable to create record', nil, nil)
        )
      end
      
      it 'ไม่ raise error ออกไป' do
        expect { described_class.new(user).send_otp }.not_to raise_error
      end
      
      it 'log error' do
        expect(Rails.logger).to receive(:error).with(/Twilio error/)
        described_class.new(user).send_otp
      end
      
      it 'คืน false' do
        result = described_class.new(user).send_otp
        expect(result).to be false
      end
    end
    
    context 'เมื่อไม่มี phone number' do
      let(:user) { create(:user, phone: nil) }
      
      it 'ไม่เรียก Twilio API' do
        expect(sms_gateway).not_to receive(:send_message)
        described_class.new(user).send_otp
      end
    end
  end
end
```

### ข้อ 10: Database Testing
**คำถาม:** เขียน test ที่ verify database constraints และ indices ทำงานถูกต้อง

**เฉลย:**
```ruby
# spec/models/concerns/database_constraints_spec.rb
RSpec.describe 'Database Constraints', type: :model do
  describe 'User' do
    it 'enforces unique email at database level' do
      create(:user, email: 'test@example.com')
      
      expect {
        # Bypass model validation ด้วย direct SQL
        User.connection.execute(
          "INSERT INTO users (name, email, created_at, updated_at) VALUES ('Test', 'test@example.com', NOW(), NOW())"
        )
      }.to raise_error(ActiveRecord::RecordNotUnique)
    end
    
    it 'has index on email' do
      expect(ActiveRecord::Base.connection.indexes('users').map(&:columns))
        .to include(include('email'))
    end
    
    it 'NOT NULL constraint บน name' do
      expect {
        User.connection.execute(
          "INSERT INTO users (email, created_at, updated_at) VALUES ('null@example.com', NOW(), NOW())"
        )
      }.to raise_error(ActiveRecord::NotNullViolation)
    end
  end
  
  describe 'Order' do
    let(:user) { create(:user) }
    
    it 'foreign key constraint บน user_id' do
      expect {
        Order.connection.execute(
          "INSERT INTO orders (user_id, status, total, created_at, updated_at) VALUES (999999, 'pending', 0, NOW(), NOW())"
        )
      }.to raise_error(ActiveRecord::InvalidForeignKey)
    end
    
    it 'total ต้องไม่ติดลบ (check constraint)' do
      expect {
        Order.connection.execute(
          "INSERT INTO orders (user_id, status, total, created_at, updated_at) VALUES (#{user.id}, 'pending', -1, NOW(), NOW())"
        )
      }.to raise_error(ActiveRecord::StatementInvalid)
    end
  end
end
```

### ข้อ 11-20 ย่อรวม:

### ข้อ 11: Feature Test with JavaScript
```ruby
# spec/features/dynamic_search_spec.rb
RSpec.describe 'Dynamic Search', type: :feature, js: true do
  before do
    create_list(:product, 20)
    sign_in create(:user)
    visit products_path
  end
  
  scenario 'แสดงผลแบบ real-time' do
    find('#search-input').fill_in with: 'laptop'
    
    expect(page).to have_css('.loading-indicator')
    expect(page).not_to have_css('.loading-indicator', wait: 3)
    expect(page).to have_css('.product-card')
  end
  
  scenario 'clear search รีเซ็ตผลลัพธ์' do
    fill_in 'Search', with: 'laptop'
    sleep 1
    
    find('#clear-search').click
    expect(page).to have_css('.product-card', count: 20, wait: 3)
  end
end
```

### ข้อ 12: Custom Matcher
```ruby
# spec/support/matchers/json_matchers.rb
RSpec::Matchers.define :be_valid_json_api_response do
  match do |response|
    return false unless response.is_a?(ActionDispatch::TestResponse)
    return false unless response.headers['Content-Type'].include?('application/json')
    
    body = JSON.parse(response.body)
    body.key?('data') || body.key?('errors')
  rescue JSON::ParserError
    false
  end
  
  failure_message do |response|
    "Expected response to be valid JSON:API but got: #{response.body[0..200]}"
  end
end

# การใช้งาน
it 'returns valid JSON:API response' do
  get '/api/v1/users', headers: auth_headers
  expect(response).to be_valid_json_api_response
end
```

### ข้อ 13: Time-dependent Tests
```ruby
# spec/models/subscription_spec.rb
RSpec.describe Subscription do
  describe '#expired?' do
    it 'คืน false เมื่อยังไม่หมดอายุ' do
      travel_to Time.zone.parse('2024-01-15') do
        sub = create(:subscription, expires_at: Time.zone.parse('2024-12-31'))
        expect(sub.expired?).to be false
      end
    end
    
    it 'คืน true เมื่อหมดอายุแล้ว' do
      travel_to Time.zone.parse('2025-01-01') do
        sub = create(:subscription, expires_at: Time.zone.parse('2024-12-31'))
        expect(sub.expired?).to be true
      end
    end
    
    it 'expired? เปลี่ยนตามเวลา' do
      sub = create(:subscription, expires_at: 1.second.from_now)
      expect(sub.expired?).to be false
      
      travel 2.seconds do
        expect(sub.expired?).to be true
      end
    end
  end
end
```

### ข้อ 14: Background Job Testing
```ruby
# spec/jobs/report_generation_job_spec.rb
RSpec.describe ReportGenerationJob, type: :job do
  let(:user) { create(:user) }
  
  it 'enqueued ใน correct queue' do
    expect(described_class.queue_name).to eq('reports')
  end
  
  it 'perform generates report' do
    expect(ReportService).to receive(:generate).with(
      user_id: user.id,
      type: 'monthly'
    ).and_return({ success: true })
    
    described_class.perform_now(user.id, 'monthly')
  end
  
  it 'retries on failure' do
    allow(ReportService).to receive(:generate).and_raise(StandardError, 'Service unavailable')
    
    expect {
      described_class.perform_now(user.id, 'monthly')
    }.to have_enqueued_job(described_class).on_queue('reports')
  end
end
```

### ข้อ 15: File Upload Testing
```ruby
# spec/requests/api/v1/uploads_spec.rb
RSpec.describe 'File Uploads', type: :request do
  let(:user) { create(:user) }
  let(:token) { JwtService.encode(user_id: user.id) }
  
  describe 'POST /api/v1/uploads' do
    let(:file) { fixture_file_upload('spec/fixtures/files/test_image.jpg', 'image/jpeg') }
    
    context 'valid file' do
      it 'อัพโหลดสำเร็จ' do
        post '/api/v1/uploads',
             params: { file: file },
             headers: { 'Authorization' => "Bearer #{token}" }
        
        expect(response).to have_http_status(:created)
        expect(JSON.parse(response.body)['data']['url']).to be_present
      end
    end
    
    context 'file too large' do
      let(:large_file) { create_large_file(11.megabytes) }
      
      it 'คืน 422' do
        post '/api/v1/uploads',
             params: { file: large_file },
             headers: { 'Authorization' => "Bearer #{token}" }
        
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
    
    context 'invalid file type' do
      let(:exe_file) { fixture_file_upload('spec/fixtures/files/malware.exe', 'application/x-msdownload') }
      
      it 'คืน 422' do
        post '/api/v1/uploads',
             params: { file: exe_file },
             headers: { 'Authorization' => "Bearer #{token}" }
        
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end
  
  private
  
  def create_large_file(size)
    Tempfile.new(['test', '.jpg']).tap do |f|
      f.write('x' * size)
      f.rewind
      ActionDispatch::Http::UploadedFile.new(tempfile: f, filename: 'large.jpg', type: 'image/jpeg')
    end
  end
end
```

### ข้อ 16: WebSocket Testing
```ruby
# spec/channels/order_channel_spec.rb
RSpec.describe OrderChannel, type: :channel do
  let(:user) { create(:user) }
  let(:order) { create(:order, user: user) }
  
  before do
    stub_connection current_user: user
  end
  
  it 'subscribes successfully' do
    subscribe order_id: order.id
    expect(subscription).to be_confirmed
  end
  
  it 'ปฏิเสธ subscription สำหรับ order ที่ไม่ใช่ของ user' do
    other_order = create(:order)
    subscribe order_id: other_order.id
    expect(subscription).to be_rejected
  end
  
  it 'broadcasts order status updates' do
    subscribe order_id: order.id
    
    expect {
      order.update!(status: 'shipped')
      ActionCable.server.broadcast(
        "order_#{order.id}",
        { status: 'shipped', updated_at: Time.current.iso8601 }
      )
    }.to have_broadcasted_to("order_#{order.id}").with(
      hash_including(status: 'shipped')
    )
  end
end
```

### ข้อ 17: Multi-tenancy Testing
```ruby
# spec/support/tenant_helpers.rb
module TenantHelpers
  def with_tenant(tenant)
    Apartment::Tenant.switch(tenant.subdomain) do
      yield
    end
  end
end

# spec/models/user_spec.rb (multi-tenant)
RSpec.describe User, type: :model do
  include TenantHelpers
  
  let(:tenant_a) { create(:tenant, subdomain: 'tenant_a') }
  let(:tenant_b) { create(:tenant, subdomain: 'tenant_b') }
  
  it 'users ของ tenant A ไม่เห็น users ของ tenant B' do
    with_tenant(tenant_a) { create(:user, email: 'user@tenant_a.com') }
    with_tenant(tenant_b) { create(:user, email: 'user@tenant_b.com') }
    
    with_tenant(tenant_a) do
      expect(User.all.map(&:email)).to include('user@tenant_a.com')
      expect(User.all.map(&:email)).not_to include('user@tenant_b.com')
    end
  end
end
```

### ข้อ 18: Performance Test
```ruby
# spec/performance/n_plus_one_spec.rb
require 'n_plus_one_control/rspec'

RSpec.describe 'N+1 Query Prevention', type: :request do
  let(:user) { create(:user) }
  let(:token) { JwtService.encode(user_id: user.id) }
  
  describe 'GET /api/v1/orders' do
    before { create_list(:order, 10, user: user) }
    
    it 'ไม่มี N+1 queries' do
      expect {
        get '/api/v1/orders', 
            headers: { 'Authorization' => "Bearer #{token}" }
      }.to perform_constant_number_of_queries
    end
  end
end
```

### ข้อ 19: Security Testing
```ruby
# spec/requests/security_spec.rb
RSpec.describe 'Security Tests', type: :request do
  describe 'SQL Injection Prevention' do
    let(:user) { create(:user) }
    let(:token) { JwtService.encode(user_id: user.id) }
    
    it 'ป้องกัน SQL injection ใน search' do
      sql_injection = "'; DROP TABLE users; --"
      
      get '/api/v1/users', 
          params: { search: sql_injection },
          headers: { 'Authorization' => "Bearer #{token}" }
      
      expect(response).to have_http_status(:ok)
      expect(User.count).to be > 0  # Users table ยังอยู่
    end
  end
  
  describe 'XSS Prevention' do
    let(:admin) { create(:user, role: 'admin') }
    let(:token) { JwtService.encode(user_id: admin.id) }
    
    it 'sanitize HTML content' do
      xss_payload = '<script>alert("XSS")</script>Innocent Text'
      
      post '/api/v1/posts',
           params: { post: { title: xss_payload, content: 'Test' } }.to_json,
           headers: { 
             'Authorization' => "Bearer #{token}",
             'Content-Type' => 'application/json'
           }
      
      if response.status == 201
        body = JSON.parse(response.body)
        title = body.dig('data', 'attributes', 'title')
        expect(title).not_to include('<script>')
      end
    end
  end
  
  describe 'Mass Assignment Protection' do
    let(:user) { create(:user, role: 'user') }
    let(:token) { JwtService.encode(user_id: user.id) }
    
    it 'ไม่อนุญาตให้ update role ผ่าน mass assignment' do
      patch "/api/v1/users/#{user.id}",
            params: { user: { name: 'Hacker', role: 'admin' } }.to_json,
            headers: { 
              'Authorization' => "Bearer #{token}",
              'Content-Type' => 'application/json'
            }
      
      expect(user.reload.role).to eq('user')
    end
  end
end
```

### ข้อ 20: Complete Test Suite Setup
```ruby
# spec/rails_helper.rb (complete configuration)
require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require_relative '../config/environment'
require 'rspec/rails'
require 'capybara/rspec'
require 'database_cleaner/active_record'
require 'factory_bot_rails'
require 'shoulda-matchers'
require 'faker'

Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }

ActiveRecord::Migration.maintain_test_schema!

RSpec.configure do |config|
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
  config.use_transactional_fixtures = false
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!
  
  config.include FactoryBot::Syntax::Methods
  config.include Devise::Test::IntegrationHelpers, type: :request
  config.include Warden::Test::Helpers, type: :feature
  config.include ApiHelpers, type: :request
  
  # Database cleaner
  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end
  
  config.around(:each) do |example|
    strategy = example.metadata[:js] ? :truncation : :transaction
    DatabaseCleaner.strategy = strategy
    DatabaseCleaner.cleaning { example.run }
  end
  
  # VCR for external HTTP calls
  config.around(:each, :vcr) do |example|
    VCR.use_cassette(example.metadata[:vcr_cassette] || example.description) do
      example.run
    end
  end
  
  # Shoulda matchers
  Shoulda::Matchers.configure do |conf|
    conf.integrate do |with|
      with.test_framework :rspec
      with.library :rails
    end
  end
  
  # Capybara
  Capybara.default_max_wait_time = 5
  Capybara.server = :puma, { Silent: true }
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Testing Pyramid** - Unit (60-70%), Integration (20-30%), E2E (5-10%)
2. **Property-based Testing** - Rantly สำหรับ generate random inputs อัตโนมัติ
3. **Mutation Testing** - Mutant gem เพื่อ verify quality ของ tests
4. **Load Testing** - k6 สำหรับ performance และ stress testing
5. **Contract Testing** - Pact สำหรับ consumer-driven contracts
6. **Approval Testing** - Snapshot testing สำหรับ complex outputs

กุญแจสำคัญ: tests ที่ดีไม่ใช่แค่ coverage สูง แต่ต้องจับ mutations ได้และมี confidence สูงในการ refactor
