# ตอนที่ 23: Testing with RSpec (ขั้นตอนที่ 491-520)

การทดสอบซอฟต์แวร์เป็นส่วนสำคัญของการพัฒนา Ruby และ Rails การเขียน Test ที่ดีช่วยให้มั่นใจว่าโค้ดทำงานถูกต้อง และช่วยในการ Refactor โดยไม่ทำให้ฟีเจอร์เดิมพัง RSpec เป็น Testing Framework ที่นิยมมากที่สุดใน Ruby Community

---

## ขั้นตอนที่ 491: Testing Philosophy - TDD และ BDD

### Test-Driven Development (TDD)

TDD เป็นแนวทางที่เขียน Test ก่อนเขียนโค้ด มี Cycle 3 ขั้นตอน:

1. **Red** - เขียน Test ที่ Fail
2. **Green** - เขียนโค้ดน้อยที่สุดเพื่อให้ Test ผ่าน
3. **Refactor** - ปรับปรุงโค้ดโดยยังให้ Test ผ่าน

```
Red → Green → Refactor → Red → Green → Refactor → ...
```

### Behavior-Driven Development (BDD)

BDD เป็นการพัฒนาต่อยอดจาก TDD โดยเน้นที่ Behavior ของระบบ ใช้ภาษาที่เข้าใจง่ายสำหรับทุกคนในทีม

```gherkin
Feature: ผู้ใช้สามารถ Login ได้
  Scenario: Login สำเร็จ
    Given มีผู้ใช้ที่ลงทะเบียนแล้ว
    When ผู้ใช้กรอก Email และ Password ที่ถูกต้อง
    Then ระบบ Redirect ไปยังหน้า Dashboard
    And แสดงข้อความต้อนรับ
```

### ประโยชน์ของการเขียน Test

- มั่นใจว่าโค้ดทำงานถูกต้อง
- ช่วยออกแบบ API ที่ดีขึ้น (เพราะต้องคิดถึงการใช้งานก่อน)
- เป็น Documentation ที่มีชีวิต
- ทำให้ Refactor ได้อย่างมั่นใจ
- ลดค่าใช้จ่ายในการหา Bug

---

## ขั้นตอนที่ 492: การติดตั้ง RSpec

### ติดตั้ง RSpec

```bash
# ติดตั้ง gem
gem install rspec

# ดูเวอร์ชัน
rspec --version

# สำหรับ Project
mkdir my_project && cd my_project
rspec --init
```

```ruby
# Gemfile
source 'https://rubygems.org'

group :development, :test do
  gem 'rspec', '~> 3.12'
  gem 'factory_bot', '~> 6.2'
  gem 'faker',       '~> 3.0'
  gem 'simplecov',   require: false
end
```

### โครงสร้าง Directory

```
my_project/
├── lib/
│   ├── user.rb
│   └── calculator.rb
├── spec/
│   ├── spec_helper.rb
│   ├── lib/
│   │   ├── user_spec.rb
│   │   └── calculator_spec.rb
│   └── support/
│       └── shared_examples.rb
├── Gemfile
└── .rspec
```

### .rspec Configuration

```
--require spec_helper
--format documentation
--color
```

### spec_helper.rb

```ruby
# spec/spec_helper.rb
require 'simplecov'
SimpleCov.start do
  add_filter '/spec/'
end

RSpec.configure do |config|
  # ใช้ expect syntax เท่านั้น (ไม่ใช้ should)
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end
  
  # ใช้ Mock Framework ของ RSpec
  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end
  
  config.shared_context_metadata_behavior = :apply_to_host_groups
  
  # Random order เพื่อหา Test ที่ Depend กัน
  config.order = :random
  Kernel.srand config.seed
  
  # Filter Run
  config.filter_run_when_matching :focus
  config.run_all_when_everything_filtered = true
  
  # Profile
  config.profile_examples = 10
end
```

---

## ขั้นตอนที่ 493: describe, context, it Blocks

### โครงสร้างพื้นฐาน

```ruby
# spec/lib/calculator_spec.rb
require 'spec_helper'
require_relative '../../lib/calculator'

# describe - บอกว่า Test อะไร
RSpec.describe Calculator do
  
  # describe สำหรับ Method
  describe '#add' do
    
    # context - บอก Scenario
    context 'เมื่อตัวเลขเป็นบวก' do
      
      # it - Test Case เดี่ยว
      it 'คืนผลบวกที่ถูกต้อง' do
        calc = Calculator.new
        expect(calc.add(2, 3)).to eq(5)
      end
      
      it 'คืนผลบวกสำหรับหลายตัวเลข' do
        calc = Calculator.new
        expect(calc.add(1, 2, 3, 4, 5)).to eq(15)
      end
    end
    
    context 'เมื่อมีตัวเลขติดลบ' do
      it 'คืนผลบวกที่ถูกต้อง' do
        calc = Calculator.new
        expect(calc.add(-1, -2)).to eq(-3)
      end
      
      it 'จัดการ Mixed Numbers ได้' do
        calc = Calculator.new
        expect(calc.add(-5, 10)).to eq(5)
      end
    end
    
    context 'เมื่อตัวเลขเป็นศูนย์' do
      it 'คืนค่าตัวเลขอีกตัว' do
        calc = Calculator.new
        expect(calc.add(0, 5)).to eq(5)
        expect(calc.add(5, 0)).to eq(5)
        expect(calc.add(0, 0)).to eq(0)
      end
    end
  end
  
  describe '#divide' do
    context 'เมื่อหารด้วยตัวเลขที่ไม่ใช่ศูนย์' do
      it 'คืนผลหาร' do
        calc = Calculator.new
        expect(calc.divide(10, 2)).to eq(5)
      end
    end
    
    context 'เมื่อหารด้วยศูนย์' do
      it 'raise ZeroDivisionError' do
        calc = Calculator.new
        expect { calc.divide(10, 0) }.to raise_error(ZeroDivisionError)
      end
    end
  end
  
  # Pending Test
  describe '#sqrt' do
    it 'คืนรากที่สอง' do
      pending 'ยังไม่ได้ Implement'
    end
    
    xit 'รองรับ Complex Numbers' do
      # Test ที่ skip
    end
  end
end
```

### Calculator Implementation

```ruby
# lib/calculator.rb
class Calculator
  def add(*numbers)
    numbers.sum
  end
  
  def subtract(a, b)
    a - b
  end
  
  def multiply(a, b)
    a * b
  end
  
  def divide(a, b)
    raise ZeroDivisionError, "ไม่สามารถหารด้วยศูนย์ได้" if b == 0
    a.to_f / b
  end
  
  def sqrt(n)
    raise ArgumentError, "ต้องเป็นจำนวนบวก" if n < 0
    Math.sqrt(n)
  end
end
```

---

## ขั้นตอนที่ 494: expect().to Matchers

### Equality Matchers

```ruby
RSpec.describe 'Equality Matchers' do
  # eq - Equal (ค่าเท่ากัน)
  it 'eq' do
    expect(1 + 1).to eq(2)
    expect("สวัสดี").to eq("สวัสดี")
    expect([1, 2, 3]).to eq([1, 2, 3])
  end
  
  # eql - Strict Equal (เท่ากันและ type เดียวกัน)
  it 'eql' do
    expect(1).to eql(1)
    expect(1).not_to eql(1.0)  # Integer != Float
  end
  
  # equal/be - Object Identity (เป็น Object เดียวกัน)
  it 'equal' do
    a = "hello"
    b = a
    c = "hello"
    
    expect(b).to equal(a)  # b คือ a (same object)
    expect(c).not_to equal(a)  # c ไม่ใช่ a (different object)
    expect(c).to eq(a)  # แต่ค่าเท่ากัน
  end
end
```

### Comparison Matchers

```ruby
RSpec.describe 'Comparison Matchers' do
  it 'be_greater_than / be_>>' do
    expect(10).to be > 5
    expect(10).to be >= 10
    expect(5).to be < 10
    expect(5).to be <= 5
  end
  
  it 'be_between' do
    expect(5).to be_between(1, 10).inclusive
    expect(5).to be_between(1, 10).exclusive
    expect(1).to be_between(1, 10).inclusive
    expect(1).not_to be_between(1, 10).exclusive
  end
  
  it 'be_within' do
    expect(3.14).to be_within(0.01).of(Math::PI)
    expect(100).to be_within(5).of(100)
    expect(97).to be_within(5).of(100)
    expect(95).not_to be_within(4).of(100)
  end
end
```

### Truthiness Matchers

```ruby
RSpec.describe 'Truthiness Matchers' do
  it 'be_truthy / be_falsy' do
    expect(true).to be_truthy
    expect(1).to be_truthy
    expect("string").to be_truthy
    expect([]).to be_truthy  # Array ว่างยังเป็น truthy!
    
    expect(nil).to be_falsy
    expect(false).to be_falsy
  end
  
  it 'be_nil' do
    expect(nil).to be_nil
    expect(false).not_to be_nil
    expect(0).not_to be_nil
  end
  
  it 'be true / be false' do
    expect(true).to be(true)
    expect(false).to be(false)
    expect(1 == 1).to be(true)
  end
end
```

### Collection Matchers

```ruby
RSpec.describe 'Collection Matchers' do
  let(:numbers) { [1, 2, 3, 4, 5] }
  let(:names)   { ['สมชาย', 'สมหญิง', 'มานี'] }
  
  it 'include' do
    expect(numbers).to include(3)
    expect(numbers).to include(1, 3, 5)
    expect(names).to include('มานี')
    expect("สวัสดีโลก").to include("โลก")
  end
  
  it 'contain_exactly' do
    expect(numbers).to contain_exactly(5, 4, 3, 2, 1)  # ลำดับไม่สำคัญ
    expect(names).to contain_exactly('มานี', 'สมชาย', 'สมหญิง')
  end
  
  it 'match_array' do
    expect(numbers).to match_array([3, 1, 4, 5, 2])
  end
  
  it 'start_with / end_with' do
    expect(numbers).to start_with(1)
    expect(numbers).to start_with(1, 2)
    expect(numbers).to end_with(5)
    expect("สวัสดี").to start_with("ส")
    expect("สวัสดี").to end_with("ดี")
  end
  
  it 'have_attributes' do
    user = double(name: 'สมชาย', age: 30, email: 'test@example.com')
    expect(user).to have_attributes(name: 'สมชาย', age: 30)
  end
  
  it 'all / any' do
    expect(numbers).to all(be > 0)
    expect(numbers).to all(be_a(Integer))
    expect([2, 4, 6]).to all(be_even)
    
    # any ไม่ใช่ built-in แต่ใช้ include ได้
    expect(numbers.any? { |n| n > 3 }).to be true
  end
end
```

### Error Matchers

```ruby
RSpec.describe 'Error Matchers' do
  it 'raise_error' do
    expect { 1 / 0 }.to raise_error(ZeroDivisionError)
    expect { Integer("abc") }.to raise_error(ArgumentError)
    expect { Integer("abc") }.to raise_error(ArgumentError, /invalid value/)
  end
  
  it 'raise_error with message' do
    expect { raise "ข้อผิดพลาด" }.to raise_error("ข้อผิดพลาด")
    expect { raise RuntimeError, "ข้อผิดพลาดร้ายแรง" }.to raise_error(RuntimeError, "ข้อผิดพลาดร้ายแรง")
  end
  
  it 'ตรวจสอบ Exception ที่ catch ได้' do
    expect {
      begin
        raise StandardError, "error"
      rescue StandardError => e
        raise RuntimeError, e.message
      end
    }.to raise_error(RuntimeError, "error")
  end
end
```

### Type/Class Matchers

```ruby
RSpec.describe 'Type Matchers' do
  it 'be_a / be_an / be_an_instance_of' do
    expect("hello").to be_a(String)
    expect(42).to be_an(Integer)
    expect([1, 2]).to be_a(Array)
    expect({}).to be_a(Hash)
    
    # be_an_instance_of (ตรงๆ ไม่รวม Subclass)
    expect("hello").to be_an_instance_of(String)
    
    # be_kind_of (รวม Subclass)
    expect(1).to be_kind_of(Numeric)
    expect(1.0).to be_kind_of(Numeric)
  end
  
  it 'respond_to' do
    expect("hello").to respond_to(:length)
    expect("hello").to respond_to(:upcase, :downcase)
    expect([1, 2, 3]).to respond_to(:each, :map, :select)
  end
end
```

---

## ขั้นตอนที่ 495: let, let!, before, after, around

### let และ let!

```ruby
RSpec.describe User do
  # let - Lazy evaluation (สร้างเมื่อถูกเรียกครั้งแรก)
  let(:user) { User.new(name: 'สมชาย', email: 'somchai@example.com') }
  let(:admin) { User.new(name: 'Admin', email: 'admin@example.com', role: :admin) }
  
  # let! - Eager evaluation (สร้างทันทีก่อน Each Test)
  let!(:existing_record) { User.create!(name: 'มีอยู่แล้ว', email: 'exists@example.com') }
  
  it 'มีชื่อที่ถูกต้อง' do
    expect(user.name).to eq('สมชาย')
  end
  
  it 'เป็น Admin' do
    expect(admin.admin?).to be true
  end
  
  it 'มี existing record ใน Database' do
    # existing_record ถูกสร้างแล้ว ไม่ต้องรอเรียก
    expect(User.count).to eq(1)
  end
  
  # Nested let
  describe '#full_name' do
    let(:user) { User.new(first_name: 'สม', last_name: 'ชาย') }
    
    it 'รวมชื่อและนามสกุล' do
      expect(user.full_name).to eq('สม ชาย')
    end
  end
  
  # let ที่ depend กัน
  let(:order) { Order.new(user: user, items: items) }
  let(:items) { [Item.new(price: 100), Item.new(price: 200)] }
  
  it 'คำนวณยอดรวม' do
    expect(order.total).to eq(300)
  end
end
```

### before, after, around

```ruby
RSpec.describe DatabaseOperation do
  before(:all) do
    # ทำงานครั้งเดียวก่อนทุก Test ใน Group นี้
    @db_connection = DatabaseConnection.connect
    puts "เชื่อมต่อ Database แล้ว"
  end
  
  after(:all) do
    # ทำงานครั้งเดียวหลังทุก Test ใน Group นี้
    @db_connection.disconnect
    puts "ปิดการเชื่อมต่อ Database แล้ว"
  end
  
  before(:each) do  # หรือ before(:example) หรือ before
    # ทำงานก่อน Each Test
    @transaction = @db_connection.begin_transaction
    puts "เริ่ม Transaction"
  end
  
  after(:each) do
    # ทำงานหลัง Each Test
    @transaction.rollback  # Roll back การเปลี่ยนแปลงหลัง Test
    puts "Rollback Transaction"
  end
  
  around(:each) do |example|
    # ห่อ Test ทั้งหมด
    puts "ก่อน Test: #{example.description}"
    example.run  # รัน Test
    puts "หลัง Test: #{example.description}"
  end
  
  it 'สร้าง Record ได้' do
    User.create!(name: 'Test User')
    expect(User.count).to eq(1)
  end
  
  it 'ลบ Record ได้' do
    user = User.create!(name: 'To Delete')
    user.destroy
    expect(User.count).to eq(0)
  end
end
```

### RSpec::Core::SharedContext

```ruby
# spec/support/database_helpers.rb
RSpec.shared_context 'with database' do
  before(:all) do
    DatabaseCleaner.clean_with(:truncation)
  end
  
  before(:each) do
    DatabaseCleaner.start
  end
  
  after(:each) do
    DatabaseCleaner.clean
  end
end

# ใช้งาน
RSpec.describe UserRepository do
  include_context 'with database'
  
  it 'บันทึกข้อมูลได้' do
    UserRepository.save(User.new(name: 'Test'))
    expect(UserRepository.count).to eq(1)
  end
end
```

---

## ขั้นตอนที่ 496: Shared Examples

### Shared Examples สำหรับ Interface Testing

```ruby
# spec/support/shared_examples.rb

# Shared Example สำหรับ Sortable
RSpec.shared_examples 'a sortable collection' do
  it 'มี sort method' do
    expect(subject).to respond_to(:sort)
  end
  
  it 'คืนข้อมูลที่เรียงลำดับแล้ว' do
    sorted = subject.sort
    expect(sorted).to eq(sorted.sort)
  end
end

# Shared Example สำหรับ Printable
RSpec.shared_examples 'a printable object' do |expected_fields|
  expected_fields.each do |field|
    it "มี #{field} ใน output" do
      expect(subject.to_s).to include(field.to_s)
    end
  end
end

# Shared Example สำหรับ Authentication
RSpec.shared_examples 'an authenticated endpoint' do
  context 'เมื่อไม่มี Token' do
    before { headers.delete('Authorization') }
    
    it 'คืน 401 Unauthorized' do
      expect(response.status).to eq(401)
    end
  end
  
  context 'เมื่อมี Token ไม่ถูกต้อง' do
    before { headers['Authorization'] = 'Bearer invalid_token' }
    
    it 'คืน 401 Unauthorized' do
      expect(response.status).to eq(401)
    end
  end
end

# การใช้งาน
RSpec.describe ProductList do
  subject { ProductList.new([3, 1, 4, 1, 5, 9, 2, 6]) }
  
  it_behaves_like 'a sortable collection'
end

RSpec.describe User do
  subject { User.new(name: 'สมชาย', email: 'test@example.com') }
  
  it_behaves_like 'a printable object', [:name, :email]
end
```

### Shared Examples with Parameters

```ruby
RSpec.shared_examples 'a payment processor' do |processor_class, options|
  let(:processor) { processor_class.new(options) }
  
  describe '#process' do
    it 'ประมวลผลการชำระเงิน' do
      result = processor.process(amount: 100, currency: 'THB')
      expect(result[:success]).to be true
      expect(result[:transaction_id]).not_to be_nil
    end
    
    it 'คืน Error เมื่อจำนวนเงินเป็น 0' do
      expect { processor.process(amount: 0, currency: 'THB') }
        .to raise_error(ArgumentError)
    end
  end
  
  describe '#refund' do
    it 'คืนเงินได้' do
      # Mock transaction
      result = processor.refund(transaction_id: 'test_123', amount: 50)
      expect(result[:success]).to be true
    end
  end
end

# การใช้งาน
RSpec.describe StripeProcessor do
  it_behaves_like 'a payment processor', StripeProcessor, { api_key: 'test_key' }
end

RSpec.describe PayPalProcessor do
  it_behaves_like 'a payment processor', PayPalProcessor, { 
    client_id: 'test_client',
    client_secret: 'test_secret'
  }
end
```

---

## ขั้นตอนที่ 497: Subject

### Implicit Subject

```ruby
RSpec.describe Array do
  # subject โดยปริยายคือ Array.new
  it { is_expected.to respond_to(:push) }
  it { is_expected.to respond_to(:pop) }
  it { is_expected.to be_empty }
  it { is_expected.to be_a(Array) }
end

RSpec.describe String do
  # subject คือ String.new (String ว่างๆ)
  it { is_expected.to be_empty }
  it { is_expected.to respond_to(:upcase) }
end
```

### Explicit Subject

```ruby
RSpec.describe User do
  subject(:user) { User.new(name: 'สมชาย', age: 25) }
  
  it 'มีชื่อ' do
    expect(user.name).to eq('สมชาย')
  end
  
  it 'ยังไม่บรรลุนิติภาวะ' do
    expect(user).not_to be_adult
  end
  
  # ใช้ is_expected กับ Named Subject ได้
  it { is_expected.to respond_to(:name) }
  it { is_expected.to respond_to(:age) }
end
```

### One-liner Syntax

```ruby
RSpec.describe Calculator do
  subject(:calc) { Calculator.new }
  
  describe '#add' do
    subject { calc.add(2, 3) }
    it { is_expected.to eq(5) }
    it { is_expected.to be_a(Numeric) }
    it { is_expected.to be > 0 }
  end
  
  describe '#negative' do
    subject { calc.add(-1, -2) }
    it { is_expected.to be < 0 }
    it { is_expected.to eq(-3) }
  end
end
```

---

## ขั้นตอนที่ 498: Doubles, Mocks, Stubs

### Test Doubles

```ruby
RSpec.describe OrderService do
  # Double - Object ที่ Simulate Object จริง
  let(:mailer)  { double('Mailer') }
  let(:payment) { double('PaymentGateway') }
  
  before do
    # Stub method calls
    allow(mailer).to receive(:send_email).and_return(true)
    allow(payment).to receive(:charge).and_return({ success: true, id: 'txn_123' })
  end
  
  subject(:service) { OrderService.new(mailer: mailer, payment: payment) }
  
  describe '#create_order' do
    it 'ส่ง Email ยืนยันหลังสร้าง Order' do
      service.create_order(amount: 100, email: 'test@example.com')
      
      # ตรวจสอบว่า mailer ถูกเรียก
      expect(mailer).to have_received(:send_email)
        .with('test@example.com', anything)
    end
    
    it 'เรียก Payment Gateway' do
      service.create_order(amount: 100, email: 'test@example.com')
      
      expect(payment).to have_received(:charge)
        .with(hash_including(amount: 100))
    end
  end
end
```

### Mocks

```ruby
RSpec.describe UserRegistration do
  describe '#register' do
    it 'ส่ง Welcome Email' do
      email_service = double('EmailService')
      
      # Mock - ตั้งค่า Expectation ก่อนเรียก
      expect(email_service).to receive(:send_welcome_email)
        .with('user@example.com')
        .once
        .and_return(true)
      
      registration = UserRegistration.new(email_service: email_service)
      registration.register(email: 'user@example.com', password: 'password123')
    end
    
    it 'ส่ง Email 2 ครั้ง (welcome + confirmation)' do
      email_service = double('EmailService')
      
      expect(email_service).to receive(:send_welcome_email).once
      expect(email_service).to receive(:send_confirmation_email).once
      
      registration = UserRegistration.new(email_service: email_service)
      registration.register(email: 'user@example.com', password: 'password123')
    end
  end
end
```

### Stubs

```ruby
RSpec.describe WeatherService do
  let(:http_client) { double('HTTPClient') }
  
  before do
    # Stub HTTP Call
    allow(http_client).to receive(:get)
      .with('https://api.weather.com/current?city=Bangkok')
      .and_return({
        temperature: 35,
        humidity: 80,
        condition: 'sunny'
      })
  end
  
  subject(:service) { WeatherService.new(http_client: http_client) }
  
  describe '#current_weather' do
    it 'คืนข้อมูลอากาศปัจจุบัน' do
      weather = service.current_weather('Bangkok')
      
      expect(weather[:temperature]).to eq(35)
      expect(weather[:condition]).to eq('sunny')
    end
  end
  
  # Stub ที่มีหลาย Return Values
  it 'เรียก API หลายครั้ง' do
    allow(http_client).to receive(:get)
      .and_return(
        { temperature: 30 },
        { temperature: 32 },
        { temperature: 28 }
      )
    
    expect(service.current_weather('Bangkok')[:temperature]).to eq(30)
    expect(service.current_weather('Bangkok')[:temperature]).to eq(32)
    expect(service.current_weather('Bangkok')[:temperature]).to eq(28)
  end
end
```

### Instance Double และ Class Double

```ruby
# instance_double ตรวจสอบว่า Method มีอยู่จริงใน Class
RSpec.describe UserService do
  let(:user) { instance_double(User, name: 'สมชาย', email: 'somchai@example.com') }
  
  before do
    allow(user).to receive(:save!).and_return(true)
    # allow(user).to receive(:nonexistent_method)  # จะ Error!
  end
  
  it 'อัพเดทข้อมูล User ได้' do
    service = UserService.new(user)
    result  = service.update_profile(name: 'ชาย สม')
    expect(result).to be_truthy
  end
end

# class_double สำหรับ Class Methods
RSpec.describe BatchProcessor do
  let(:user_class) { class_double(User) }
  
  before do
    allow(user_class).to receive(:find_each).and_yield(
      instance_double(User, id: 1, email: 'user1@example.com'),
      instance_double(User, id: 2, email: 'user2@example.com')
    )
  end
  
  it 'ประมวลผล User ทุกคน' do
    processor = BatchProcessor.new(user_class)
    expect { processor.process_all }.not_to raise_error
  end
end
```

### Spy

```ruby
# Spy - Double ที่บันทึกทุก Interaction
RSpec.describe EventTracker do
  let(:analytics) { spy('Analytics') }
  
  subject(:tracker) { EventTracker.new(analytics: analytics) }
  
  it 'บันทึก Page View' do
    tracker.track_page_view('/home')
    tracker.track_page_view('/about')
    
    expect(analytics).to have_received(:track)
      .with('page_view', hash_including(path: '/home'))
    expect(analytics).to have_received(:track)
      .with('page_view', hash_including(path: '/about'))
    expect(analytics).to have_received(:track).exactly(2).times
  end
end
```

---

## ขั้นตอนที่ 499: Testing Errors

```ruby
RSpec.describe BankAccount do
  let(:account) { BankAccount.new(balance: 1000) }
  
  describe '#withdraw' do
    context 'เมื่อยอดเงินพอ' do
      it 'ถอนเงินสำเร็จ' do
        expect { account.withdraw(500) }.not_to raise_error
        expect(account.balance).to eq(500)
      end
    end
    
    context 'เมื่อยอดเงินไม่พอ' do
      it 'raise InsufficientFundsError' do
        expect { account.withdraw(1500) }
          .to raise_error(InsufficientFundsError)
      end
      
      it 'Error message ถูกต้อง' do
        expect { account.withdraw(1500) }
          .to raise_error(InsufficientFundsError, /ยอดเงินไม่เพียงพอ/)
      end
      
      it 'ยอดเงินไม่เปลี่ยน' do
        begin
          account.withdraw(1500)
        rescue InsufficientFundsError
          # ignored
        end
        expect(account.balance).to eq(1000)
      end
    end
    
    context 'เมื่อถอนจำนวนลบ' do
      it 'raise ArgumentError' do
        expect { account.withdraw(-100) }
          .to raise_error(ArgumentError, 'จำนวนเงินต้องมากกว่า 0')
      end
    end
  end
  
  describe '#transfer' do
    let(:target_account) { BankAccount.new(balance: 500) }
    
    it 'โอนเงินสำเร็จ' do
      expect {
        account.transfer(target_account, 200)
      }.to change { account.balance }.by(-200)
       .and change { target_account.balance }.by(200)
    end
    
    it 'raise Error เมื่อโอนไปยัง Account เดียวกัน' do
      expect { account.transfer(account, 100) }
        .to raise_error(ArgumentError, 'ไม่สามารถโอนเงินให้ตัวเองได้')
    end
  end
end
```

---

## ขั้นตอนที่ 500: change Matcher

```ruby
RSpec.describe ShoppingCart do
  let(:cart) { ShoppingCart.new }
  let(:item) { Product.new(id: 1, name: 'สินค้า', price: 100) }
  
  describe '#add_item' do
    it 'เพิ่มจำนวน Item' do
      expect { cart.add_item(item) }
        .to change { cart.item_count }.by(1)
    end
    
    it 'เพิ่มยอดรวม' do
      expect { cart.add_item(item) }
        .to change { cart.total }.from(0).to(100)
    end
    
    it 'เปลี่ยนสถานะ Empty' do
      expect { cart.add_item(item) }
        .to change { cart.empty? }.from(true).to(false)
    end
    
    # ตรวจสอบหลาย change พร้อมกัน
    it 'เพิ่มทั้ง count และ total' do
      expect { cart.add_item(item, quantity: 2) }
        .to change { cart.item_count }.by(2)
        .and change { cart.total }.by(200)
    end
  end
  
  describe '#remove_item' do
    before { cart.add_item(item, quantity: 3) }
    
    it 'ลดจำนวน Item' do
      expect { cart.remove_item(item) }
        .to change { cart.item_count }.by(-1)
    end
    
    it 'ไม่เปลี่ยน Item Count เมื่อ Item ไม่มีอยู่' do
      other_item = Product.new(id: 99, name: 'ไม่มี')
      
      expect { cart.remove_item(other_item) }
        .not_to change { cart.item_count }
    end
  end
end
```

---

## ขั้นตอนที่ 501: Test Coverage ด้วย SimpleCov

```ruby
# spec/spec_helper.rb
require 'simplecov'

SimpleCov.start do
  # ตั้งค่า Coverage Threshold
  minimum_coverage 80
  
  # Group
  add_group 'Controllers', 'app/controllers'
  add_group 'Models',      'app/models'
  add_group 'Services',    'app/services'
  add_group 'Helpers',     'app/helpers'
  
  # Exclude
  add_filter '/spec/'
  add_filter '/config/'
  add_filter '/vendor/'
  
  # Format
  formatter SimpleCov::Formatter::HTMLFormatter
end
```

```bash
# รัน RSpec พร้อม Coverage
bundle exec rspec

# ดูผลลัพธ์ Coverage
open coverage/index.html
```

### Coverage Report ที่ดี

```
Coverage report generated to /path/to/coverage/index.html

COVERAGE:  95.43% -- 2341 / 2452 lines in 45 files

+----------+----------+----------+--------+
| Group    | Coverage | Lines    | Missed |
+----------+----------+----------+--------+
| Models   |   98.2%  |  854     |  15    |
| Services |   96.5%  |  632     |  22    |
| Helpers  |   88.3%  |  120     |  14    |
+----------+----------+----------+--------+
```

---

## ขั้นตอนที่ 502: Factory Bot

```ruby
# Gemfile
# gem 'factory_bot'
# gem 'faker'

# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name  { Faker::Name.name }
    email { Faker::Internet.email }
    password { 'password123' }
    role { :member }
    active { true }
    
    created_at { Time.current }
    
    # Trait - สำหรับ Variation ต่างๆ
    trait :admin do
      role { :admin }
      email { "admin_#{SecureRandom.hex(4)}@example.com" }
    end
    
    trait :premium do
      role { :premium }
      subscription_expires_at { 1.year.from_now }
    end
    
    trait :inactive do
      active { false }
    end
    
    trait :with_orders do
      after(:create) do |user|
        create_list(:order, 3, user: user)
      end
    end
    
    # Factory ที่ Inherit
    factory :admin_user, traits: [:admin]
    factory :premium_user, traits: [:premium]
  end
end

# spec/factories/orders.rb
FactoryBot.define do
  factory :order do
    association :user
    status { :pending }
    total  { Faker::Commerce.price(range: 100..10000) }
    
    trait :completed do
      status { :completed }
      completed_at { Time.current }
    end
    
    trait :with_items do
      after(:create) do |order|
        create_list(:order_item, rand(1..5), order: order)
      end
    end
  end
end

# การใช้งานใน Specs
RSpec.describe User do
  # build - สร้าง Object แต่ไม่ Save
  let(:user) { build(:user) }
  
  # create - สร้างและ Save ใน Database
  let(:saved_user) { create(:user) }
  
  # build_stubbed - สร้าง Object ที่ Stub Database Calls
  let(:stubbed_user) { build_stubbed(:user) }
  
  # with attributes
  let(:admin) { create(:user, :admin, name: 'ผู้ดูแล') }
  
  # create_list - สร้างหลายตัว
  let(:users) { create_list(:user, 5) }
  
  # create_pair - สร้าง 2 ตัว
  let(:user_pair) { create_pair(:user) }
  
  it 'สร้าง User ได้' do
    expect(user).to be_valid
  end
  
  it 'Admin มี role admin' do
    expect(admin.admin?).to be true
  end
  
  it 'สร้าง User หลายคน' do
    expect(users.size).to eq(5)
    expect(users.map(&:email).uniq.size).to eq(5)
  end
end
```

---

## ขั้นตอนที่ 503: Testing Private Methods

```ruby
class CreditCardValidator
  def validate(number)
    cleaned = clean_number(number)
    
    return false unless valid_length?(cleaned)
    return false unless valid_format?(cleaned)
    return false unless luhn_check?(cleaned)
    
    true
  end
  
  private
  
  def clean_number(number)
    number.to_s.gsub(/\D/, '')
  end
  
  def valid_length?(number)
    [13, 15, 16].include?(number.length)
  end
  
  def valid_format?(number)
    number.match?(/\A\d+\z/)
  end
  
  def luhn_check?(number)
    digits = number.chars.map(&:to_i)
    sum = digits.reverse.each_with_index.sum do |digit, index|
      if index.odd?
        doubled = digit * 2
        doubled > 9 ? doubled - 9 : doubled
      else
        digit
      end
    end
    sum % 10 == 0
  end
end

# spec/lib/credit_card_validator_spec.rb
RSpec.describe CreditCardValidator do
  subject(:validator) { CreditCardValidator.new }
  
  # Test Public Method (ดีกว่า)
  describe '#validate' do
    context 'Visa Card' do
      it 'ยอมรับ Visa Card ที่ถูกต้อง' do
        expect(validator.validate('4532015112830366')).to be true
      end
      
      it 'ปฏิเสธ Card ที่ผิด' do
        expect(validator.validate('4532015112830367')).to be false
      end
    end
  end
  
  # Test Private Method (ทำได้แต่ไม่แนะนำ)
  describe '#clean_number' do
    it 'ลบ Non-digit Characters' do
      # วิธีที่ 1: send
      result = validator.send(:clean_number, '4532-0151-1283-0366')
      expect(result).to eq('4532015112830366')
    end
  end
  
  describe '#luhn_check?' do
    it 'ตรวจสอบ Luhn Algorithm' do
      # ทดสอบโดยอ้อมผ่าน validate
      # หรือใช้ send สำหรับ Edge Cases
      valid_number   = '4532015112830366'
      invalid_number = '4532015112830367'
      
      expect(validator.send(:luhn_check?, valid_number)).to be true
      expect(validator.send(:luhn_check?, invalid_number)).to be false
    end
  end
end
```

---

## ขั้นตอนที่ 504: Integration Testing

```ruby
# spec/integration/user_registration_spec.rb
RSpec.describe 'User Registration', type: :integration do
  let(:email)    { 'newuser@example.com' }
  let(:password) { 'SecurePass123!' }
  
  context 'Registration Flow ทั้งหมด' do
    it 'ลงทะเบียน Login และเข้าถึงหน้า Protected ได้' do
      # Step 1: ลงทะเบียน
      user = UserService.register(
        email: email,
        password: password,
        name: 'ผู้ใช้ใหม่'
      )
      
      expect(user).to be_persisted
      expect(user.email).to eq(email)
      expect(user.role).to eq('member')
      
      # Step 2: Verify Email
      verification = UserVerification.find_by(user: user)
      expect(verification).not_to be_nil
      
      UserVerification.confirm!(verification.token)
      user.reload
      expect(user.email_verified?).to be true
      
      # Step 3: Login
      session = AuthService.login(email: email, password: password)
      expect(session).not_to be_nil
      expect(session.valid?).to be true
      
      # Step 4: เข้าถึงข้อมูล
      profile = ProfileService.get_profile(session.user_id)
      expect(profile.name).to eq('ผู้ใช้ใหม่')
    end
    
    it 'ลงทะเบียนซ้ำ Email เดิมไม่ได้' do
      UserService.register(email: email, password: password, name: 'คนแรก')
      
      expect {
        UserService.register(email: email, password: password, name: 'คนที่สอง')
      }.to raise_error(DuplicateEmailError)
    end
  end
end
```

---

## ขั้นตอนที่ 505-510: Custom Matchers

```ruby
# spec/support/custom_matchers.rb

# Custom Matcher สำหรับ Email Format
RSpec::Matchers.define :be_valid_email do
  match do |email|
    email.match?(/\A[^@\s]+@[^@\s]+\z/)
  end
  
  failure_message do |email|
    "คาดว่า #{email.inspect} เป็น Valid Email"
  end
  
  failure_message_when_negated do |email|
    "คาดว่า #{email.inspect} ไม่เป็น Valid Email"
  end
  
  description do
    "เป็น valid email address"
  end
end

# Custom Matcher สำหรับ Thai Phone Number
RSpec::Matchers.define :be_valid_thai_phone do
  match do |phone|
    cleaned = phone.gsub(/[\s\-]/, '')
    cleaned.match?(/\A(0[689]\d{8}|0\d{8})\z/)
  end
  
  failure_message do |phone|
    "คาดว่า #{phone.inspect} เป็นเบอร์โทรศัพท์ไทยที่ถูกต้อง"
  end
end

# Custom Matcher สำหรับ Response
RSpec::Matchers.define :be_successful_response do
  match do |response|
    response[:success] == true && !response[:data].nil?
  end
  
  failure_message do |response|
    "คาดว่า Response เป็น Successful แต่ได้: #{response.inspect}"
  end
end

# การใช้งาน
RSpec.describe User do
  it 'มี Email ที่ถูกต้อง' do
    user = User.new(email: 'test@example.com')
    expect(user.email).to be_valid_email
  end
  
  it 'มี Phone ที่ถูกต้อง' do
    user = User.new(phone: '0812345678')
    expect(user.phone).to be_valid_thai_phone
  end
end
```

---

## แบบฝึกหัดบทที่ 23 (30 ข้อ)

### ระดับพื้นฐาน (ข้อ 1-10)

**ข้อ 1:** เขียน Test สำหรับ Stack Class ที่มี `push`, `pop`, `peek`, `empty?`, `size` โดยครอบคลุม Edge Cases

**ข้อ 2:** เขียน Test สำหรับ StringFormatter Class ที่มี `capitalize`, `snake_case`, `camel_case`

**ข้อ 3:** เขียน Test สำหรับ DateHelper Module ที่มี `days_between`, `working_days`, `format_thai`

**ข้อ 4:** ใช้ `change` matcher ทดสอบ Counter Class ที่มี `increment`, `decrement`, `reset`

**ข้อ 5:** เขียน Shared Example สำหรับ Comparable Objects ที่มี `>`, `<`, `==`, `between?`

**ข้อ 6:** ใช้ Factory Bot สร้าง Factories สำหรับ User, Post, Comment Models

**ข้อ 7:** เขียน Test สำหรับ Method ที่ raise หลาย Error Types

**ข้อ 8:** ใช้ `let` และ `let!` ทดสอบ Repository Pattern

**ข้อ 9:** เขียน Custom Matcher สำหรับ Valid URL และ Valid Credit Card Number

**ข้อ 10:** เขียน Test สำหรับ Fibonacci Sequence Generator

### ระดับกลาง (ข้อ 11-20)

**ข้อ 11:** ใช้ Mocks/Stubs ทดสอบ EmailService ที่ depend on SMTP Connection

**ข้อ 12:** ทดสอบ Observer Pattern โดย Mock Observers

**ข้อ 13:** เขียน Integration Test สำหรับ User Authentication Flow

**ข้อ 14:** ใช้ `subject` และ One-liner Syntax ทดสอบ Model Validations

**ข้อ 15:** ทดสอบ Decorator Pattern โดย Test แต่ละ Decorator แยกกัน

**ข้อ 16:** เขียน Shared Context สำหรับ Database Transactions

**ข้อ 17:** ทดสอบ Strategy Pattern โดยใช้ Test Doubles

**ข้อ 18:** เขียน Test สำหรับ Command Pattern พร้อม Undo/Redo

**ข้อ 19:** ทดสอบ Chain of Responsibility ด้วย Stubs

**ข้อ 20:** Setup SimpleCov และตั้งค่า Coverage 90%

### ระดับสูง (ข้อ 21-30)

**ข้อ 21:** เขียน Full Test Suite สำหรับ BankAccount System พร้อม Transaction History

**ข้อ 22:** ทดสอบ Concurrent Code โดยใช้ Thread ใน Tests

**ข้อ 23:** เขียน Test สำหรับ ActiveRecord Models ด้วย Factory Bot พร้อม Complex Associations

**ข้อ 24:** Implement Custom Matcher สำหรับ JSON Response Validation

**ข้อ 25:** ทดสอบ File I/O Operations โดยใช้ Temporary Files

**ข้อ 26:** เขียน Test สำหรับ Recursive Algorithms (Binary Search, Merge Sort)

**ข้อ 27:** ทดสอบ External API Client ด้วย WebMock หรือ VCR

**ข้อ 28:** เขียน Performance Test ด้วย RSpec Benchmark

**ข้อ 29:** ทดสอบ Background Jobs (Sidekiq/Active Job)

**ข้อ 30:** สร้าง Complete TDD Cycle สำหรับ Feature ใหม่ตั้งแต่ต้น

---

### เฉลยตัวอย่าง ข้อ 1: Stack Class Tests

```ruby
# lib/stack.rb
class Stack
  def initialize
    @data = []
  end
  
  def push(item)
    @data.push(item)
    self
  end
  
  def pop
    raise "Stack เป็นว่าง" if empty?
    @data.pop
  end
  
  def peek
    raise "Stack เป็นว่าง" if empty?
    @data.last
  end
  
  def empty?
    @data.empty?
  end
  
  def size
    @data.size
  end
  
  def to_a
    @data.dup
  end
end

# spec/lib/stack_spec.rb
require 'spec_helper'
require_relative '../../lib/stack'

RSpec.describe Stack do
  subject(:stack) { Stack.new }
  
  describe '#push' do
    it 'เพิ่ม Item ลงใน Stack' do
      stack.push(1)
      expect(stack.size).to eq(1)
    end
    
    it 'เพิ่มได้หลาย Item' do
      stack.push(1).push(2).push(3)
      expect(stack.size).to eq(3)
    end
    
    it 'คืน Stack เพื่อ Chaining' do
      expect(stack.push(1)).to eq(stack)
    end
    
    it 'รับ Item ได้ทุกประเภท' do
      stack.push("string").push(42).push(nil).push([1, 2])
      expect(stack.size).to eq(4)
    end
  end
  
  describe '#pop' do
    context 'เมื่อ Stack มีข้อมูล' do
      before { stack.push(1).push(2).push(3) }
      
      it 'คืนค่าล่าสุด' do
        expect(stack.pop).to eq(3)
      end
      
      it 'ลบค่าล่าสุดออก' do
        expect { stack.pop }.to change { stack.size }.by(-1)
      end
      
      it 'ทำงานตาม LIFO order' do
        expect(stack.pop).to eq(3)
        expect(stack.pop).to eq(2)
        expect(stack.pop).to eq(1)
      end
    end
    
    context 'เมื่อ Stack ว่าง' do
      it 'raise RuntimeError' do
        expect { stack.pop }.to raise_error(RuntimeError, 'Stack เป็นว่าง')
      end
    end
  end
  
  describe '#peek' do
    context 'เมื่อ Stack มีข้อมูล' do
      before { stack.push(1).push(2).push(3) }
      
      it 'คืนค่าล่าสุดโดยไม่ลบ' do
        expect(stack.peek).to eq(3)
        expect(stack.size).to eq(3)  # ขนาดไม่เปลี่ยน
      end
    end
    
    context 'เมื่อ Stack ว่าง' do
      it 'raise RuntimeError' do
        expect { stack.peek }.to raise_error(RuntimeError, 'Stack เป็นว่าง')
      end
    end
  end
  
  describe '#empty?' do
    it 'คืน true เมื่อ Stack ว่าง' do
      expect(stack).to be_empty
    end
    
    it 'คืน false หลัง Push' do
      stack.push(1)
      expect(stack).not_to be_empty
    end
    
    it 'คืน true หลัง Pop ทั้งหมด' do
      stack.push(1).push(2)
      stack.pop
      stack.pop
      expect(stack).to be_empty
    end
  end
  
  describe '#size' do
    it 'คืน 0 เมื่อว่าง' do
      expect(stack.size).to eq(0)
    end
    
    it 'เพิ่มขึ้นเมื่อ Push' do
      expect { stack.push(1) }.to change { stack.size }.from(0).to(1)
    end
    
    it 'ลดลงเมื่อ Pop' do
      stack.push(1).push(2)
      expect { stack.pop }.to change { stack.size }.from(2).to(1)
    end
  end
  
  describe 'Complex scenarios' do
    it 'ทำงานถูกต้องในสถานการณ์ซับซ้อน' do
      stack.push(1).push(2).push(3)
      
      popped = stack.pop
      expect(popped).to eq(3)
      expect(stack.peek).to eq(2)
      
      stack.push(10).push(20)
      expect(stack.size).to eq(4)
      expect(stack.to_a).to eq([1, 2, 10, 20])
    end
  end
end
```

---

*สรุปบทที่ 23: RSpec เป็น Testing Framework ที่ทรงพลังสำหรับ Ruby ที่ช่วยให้เขียน Test ที่อ่านเข้าใจง่าย การใช้ describe/context/it ช่วยจัดโครงสร้าง Test Matchers ที่หลากหลายช่วย Assert ได้ครบถ้วน Doubles/Mocks/Stubs ช่วย Isolate Dependencies และ Factory Bot ช่วยสร้าง Test Data ได้อย่างสะดวก*
