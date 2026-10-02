# ตอนที่ 23: Testing ด้วย RSpec (Steps 491-520)

## บทนำ

**Testing** เป็นส่วนสำคัญของการพัฒนา software ที่ดี Ruby มี testing framework หลายตัว แต่ **RSpec** เป็นที่นิยมที่สุด เนื่องจากอ่านเข้าใจง่ายและ expressive มาก

RSpec ใช้แนวทาง **BDD (Behavior-Driven Development)** ที่เน้นการอธิบาย behavior ของ code ในภาษาที่คล้าย natural language

---

## Step 491: Testing Philosophy - TDD vs BDD

### TDD (Test-Driven Development)
```
1. เขียน test ที่ fail
2. เขียน code ที่น้อยที่สุดให้ test ผ่าน
3. Refactor
4. วนซ้ำ
```

### BDD (Behavior-Driven Development)
```
1. อธิบาย behavior ที่ต้องการ (ภาษาธรรมชาติ)
2. เขียน spec ตาม behavior
3. Implement
4. ตรวจสอบว่า spec ผ่าน
```

```ruby
# TDD style (Minitest)
def test_user_can_login
  user = User.new("alice", "password")
  assert user.authenticate("password")
  refute user.authenticate("wrong")
end

# BDD style (RSpec)
describe User do
  describe "#authenticate" do
    context "with correct password" do
      it "returns true" do
        user = User.new("alice", "password")
        expect(user.authenticate("password")).to be true
      end
    end

    context "with wrong password" do
      it "returns false" do
        user = User.new("alice", "password")
        expect(user.authenticate("wrong")).to be false
      end
    end
  end
end
```

---

## Step 492: RSpec Installation และ Setup

### Installation

```bash
# เพิ่ม rspec ใน Gemfile
# gem 'rspec', group: :development

# หรือ install โดยตรง
gem install rspec

# เริ่มต้น project
rspec --init
# สร้าง:
# .rspec       - default options
# spec/spec_helper.rb - configuration
```

### .rspec file

```
--require spec_helper
--format documentation
--color
```

### spec/spec_helper.rb

```ruby
RSpec.configure do |config|
  # Use expect syntax (ไม่ใช้ should)
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  # Mock framework
  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  # Shared context metadata
  config.shared_context_metadata_behavior = :apply_to_host_groups

  # Filter
  config.filter_run_when_matching :focus

  # Random order
  config.order = :random
  Kernel.srand config.seed

  # Output
  config.default_formatter = "doc" if config.files_to_run.one?
end
```

### โครงสร้าง Project

```
my_project/
├── lib/
│   ├── calculator.rb
│   └── user.rb
├── spec/
│   ├── spec_helper.rb
│   ├── calculator_spec.rb
│   └── user_spec.rb
├── Gemfile
└── .rspec
```

---

## Step 493: describe, context, it

```ruby
# lib/calculator.rb
class Calculator
  def add(a, b)
    a + b
  end

  def subtract(a, b)
    a - b
  end

  def multiply(a, b)
    a * b
  end

  def divide(a, b)
    raise ArgumentError, "Cannot divide by zero" if b == 0
    a.to_f / b
  end
end
```

```ruby
# spec/calculator_spec.rb
require 'calculator'

RSpec.describe Calculator do
  # describe คือ "สิ่งที่กำลัง test"
  # สร้าง Calculator instance ใช้ร่วมกัน
  let(:calculator) { Calculator.new }

  # describe nested
  describe "#add" do
    it "adds two positive numbers" do
      expect(calculator.add(2, 3)).to eq(5)
    end

    it "adds negative numbers" do
      expect(calculator.add(-2, -3)).to eq(-5)
    end

    it "adds positive and negative" do
      expect(calculator.add(5, -3)).to eq(2)
    end
  end

  describe "#subtract" do
    it "subtracts second from first" do
      expect(calculator.subtract(10, 3)).to eq(7)
    end

    it "handles negative result" do
      expect(calculator.subtract(3, 10)).to eq(-7)
    end
  end

  describe "#multiply" do
    context "when both numbers are positive" do
      it "returns positive result" do
        expect(calculator.multiply(3, 4)).to eq(12)
      end
    end

    context "when one number is negative" do
      it "returns negative result" do
        expect(calculator.multiply(3, -4)).to eq(-12)
      end
    end

    context "when both numbers are negative" do
      it "returns positive result" do
        expect(calculator.multiply(-3, -4)).to eq(12)
      end
    end

    context "when one number is zero" do
      it "returns zero" do
        expect(calculator.multiply(5, 0)).to eq(0)
      end
    end
  end

  describe "#divide" do
    context "when dividing by non-zero number" do
      it "returns float result" do
        result = calculator.divide(10, 4)
        expect(result).to eq(2.5)
      end

      it "returns exact division" do
        expect(calculator.divide(10, 2)).to eq(5.0)
      end
    end

    context "when dividing by zero" do
      it "raises ArgumentError" do
        expect { calculator.divide(10, 0) }.to raise_error(ArgumentError)
      end

      it "raises error with message" do
        expect { calculator.divide(10, 0) }
          .to raise_error(ArgumentError, "Cannot divide by zero")
      end
    end
  end
end
```

### รัน RSpec

```bash
# รัน tests ทั้งหมด
rspec

# รัน file เดียว
rspec spec/calculator_spec.rb

# รัน specific example
rspec spec/calculator_spec.rb:25

# รัน กับ format ต่างๆ
rspec --format documentation
rspec --format progress  # dots

# รัน เฉพาะ failing tests
rspec --only-failures
```

---

## Step 494: expect().to Matchers

```ruby
# ============================================
# Equality Matchers
# ============================================
expect(1 + 1).to eq(2)          # ==
expect("hello").to eq("hello")   # ==
expect(1).not_to eq(2)          # !=

expect(1).to eql(1)             # eql? (same value and type)
expect(1).not_to eql(1.0)       # 1 eql? 1.0 is false!

expect(arr).to equal(arr)       # object_id (same object)
expect("a").not_to equal("a")   # คนละ object

# ============================================
# Truthiness Matchers
# ============================================
expect(true).to be_truthy       # truthy value
expect(false).to be_falsy       # falsy value
expect(nil).to be_falsy

expect(nil).to be_nil           # exactly nil
expect(false).not_to be_nil     # false is NOT nil

expect(true).to be true         # exactly true
expect(false).to be false       # exactly false

# ============================================
# Numeric Matchers
# ============================================
expect(5).to be > 3
expect(5).to be >= 5
expect(5).to be < 10
expect(5).to be <= 5

expect(5).to be_between(1, 10).inclusive
expect(5).to be_between(1, 10).exclusive  # fails at boundary

# Float comparison
expect(0.1 + 0.2).to be_within(0.001).of(0.3)

# ============================================
# String Matchers
# ============================================
expect("hello world").to include("world")
expect("hello world").to start_with("hello")
expect("hello world").to end_with("world")
expect("hello").to match(/ell/)
expect("HELLO").to match(/hello/i)

# ============================================
# Array Matchers
# ============================================
arr = [1, 2, 3, 4, 5]
expect(arr).to include(3)
expect(arr).to include(1, 3, 5)
expect(arr).to start_with(1, 2)
expect(arr).to end_with(4, 5)
expect(arr).to contain_exactly(5, 4, 3, 2, 1)  # same elements, any order
expect(arr).to match_array([5, 3, 1, 4, 2])     # same elements, any order

expect(arr.length).to eq(5)
expect(arr).to have_attributes(length: 5, first: 1)

# ============================================
# Hash Matchers
# ============================================
hash = { name: "Alice", age: 30 }
expect(hash).to include(name: "Alice")
expect(hash).to include(:name, :age)
expect(hash).to have_key(:name)
expect(hash).to have_value("Alice")

# ============================================
# Type Matchers
# ============================================
expect("hello").to be_a(String)
expect("hello").to be_an_instance_of(String)
expect(42).to be_a(Integer)
expect(42).to be_a(Numeric)  # Integer inherits Numeric

# be_kind_of checks inheritance
expect(42).to be_kind_of(Numeric)   # true
expect(42).to be_instance_of(Integer)  # true
expect(42).to be_instance_of(Numeric)  # false! (exact class)

# ============================================
# Predicate Matchers
# ============================================
expect([]).to be_empty
expect([1]).not_to be_empty
expect(nil).to be_nil
expect(0).to be_zero
expect(2).to be_even
expect(3).to be_odd

# Custom predicate (calls method with ?)
class User
  def admin?
    @role == :admin
  end
end

user = User.new
# RSpec เรียก user.admin? อัตโนมัติ
expect(user).to be_admin  # calls user.admin?
```

---

## Step 495: let และ let!

```ruby
RSpec.describe User do
  # let - lazy evaluation (สร้างเมื่อถูกเรียกครั้งแรก)
  let(:user) { User.new(name: "Alice", email: "alice@example.com") }
  let(:password) { "secure_password_123" }

  describe "#authenticate" do
    context "with correct password" do
      it "returns true" do
        # user สร้างเมื่อถูกเรียกครั้งแรก
        expect(user.authenticate(password)).to be true
      end
    end
  end

  # let! - eager evaluation (สร้างทันทีก่อน test)
  let!(:saved_user) do
    u = User.create(name: "Bob", email: "bob@example.com")
    u
  end

  it "exists in database" do
    # saved_user สร้างก่อน test นี้รัน
    expect(User.find(saved_user.id)).to eq(saved_user)
  end

  # let vs let!
  # ใช้ let เมื่อ: ไม่แน่ใจว่าทุก test จะใช้
  # ใช้ let! เมื่อ: ต้องการให้ side effects (เช่น create record) เกิดก่อน test

  # ตัวอย่างที่ชัดเจนขึ้น
  describe "counter example" do
    let(:counter) { Counter.new }  # สร้างใหม่ทุก example

    it "starts at zero" do
      expect(counter.value).to eq(0)
    end

    it "increments" do
      counter.increment
      expect(counter.value).to eq(1)
    end
    # แต่ละ test ได้ counter ใหม่ - ไม่ interference กัน
  end
end
```

---

## Step 496: before, after, around hooks

```ruby
RSpec.describe DatabaseTest do
  # before(:each) - รันก่อนทุก example
  before(:each) do
    @db = Database.connect
    @db.begin_transaction
    puts "Before each: transaction started"
  end

  # after(:each) - รันหลังทุก example
  after(:each) do
    @db.rollback
    @db.disconnect
    puts "After each: transaction rolled back"
  end

  # before(:all) / before(:context) - รันครั้งเดียวก่อน context
  before(:all) do
    puts "Before all: setting up test database"
    TestDatabase.create_schema
  end

  after(:all) do
    puts "After all: dropping test database"
    TestDatabase.drop
  end

  # around hook - wrap example
  around(:each) do |example|
    puts "Around: before"
    example.run
    puts "Around: after"
  end

  it "runs a test" do
    # before each, around before
    result = @db.query("SELECT 1")
    expect(result).not_to be_nil
    # around after, after each
  end

  # ตัวอย่างจริง: File cleanup
  describe "File operations" do
    let(:test_file) { "/tmp/test_#{SecureRandom.hex(8)}.txt" }

    after(:each) do
      File.delete(test_file) if File.exist?(test_file)
    end

    it "creates a file" do
      File.write(test_file, "content")
      expect(File.exist?(test_file)).to be true
    end

    it "reads file content" do
      File.write(test_file, "hello")
      expect(File.read(test_file)).to eq("hello")
    end
  end
end
```

---

## Step 497: subject

```ruby
RSpec.describe Array do
  # subject - object ที่กำลัง test (default: described class instance)
  subject { [1, 2, 3] }

  it "has a length of 3" do
    expect(subject.length).to eq(3)
  end

  # Shorthand - is_expected
  it { is_expected.to include(2) }
  it { is_expected.not_to be_empty }

  # Named subject
  describe Calculator do
    subject(:calc) { Calculator.new }

    it "can add" do
      expect(calc.add(1, 2)).to eq(3)
    end

    it "can multiply" do
      expect(calc.multiply(3, 4)).to eq(12)
    end
  end

  # subject กับ describe class
  describe String do
    # subject = String.new โดยอัตโนมัติ
    it { is_expected.to be_empty }
    it { is_expected.to be_a(String) }
  end

  # subject กับ one-liner expectations
  describe "even number" do
    subject { 4 }
    it { is_expected.to be_even }
    it { is_expected.to be > 0 }
    it { is_expected.not_to be_odd }
  end
end
```

---

## Step 498: Shared Examples

```ruby
# spec/support/shared_examples.rb

# Shared examples สำหรับ persistence
RSpec.shared_examples "a persistable model" do
  it "can be saved" do
    expect { subject.save }.not_to raise_error
  end

  it "has an id after saving" do
    subject.save
    expect(subject.id).not_to be_nil
  end

  it "can be found by id" do
    subject.save
    found = subject.class.find(subject.id)
    expect(found).to eq(subject)
  end

  it "can be destroyed" do
    subject.save
    id = subject.id
    subject.destroy
    expect { subject.class.find(id) }.to raise_error(RecordNotFound)
  end
end

# Shared examples สำหรับ validation
RSpec.shared_examples "validates presence of name" do
  it "is invalid without a name" do
    subject.name = nil
    expect(subject).not_to be_valid
  end

  it "is invalid with empty name" do
    subject.name = ""
    expect(subject).not_to be_valid
  end

  it "is valid with a name" do
    subject.name = "Test Name"
    expect(subject).to be_valid
  end
end

# Shared examples กับ parameters
RSpec.shared_examples "a collection" do |items|
  it "responds to each" do
    expect(subject).to respond_to(:each)
  end

  it "responds to map" do
    expect(subject).to respond_to(:map)
  end

  it "responds to select" do
    expect(subject).to respond_to(:select)
  end

  it "has the expected items" do
    expect(subject.to_a).to match_array(items) if items
  end
end

# ใช้ shared examples
RSpec.describe User do
  subject { User.new(name: "Alice", email: "alice@example.com") }
  it_behaves_like "a persistable model"
  it_behaves_like "validates presence of name"
end

RSpec.describe Product do
  subject { Product.new(name: "Widget", price: 9.99) }
  it_behaves_like "validates presence of name"
end

# include_context - shared setup
RSpec.shared_context "with logged in user" do
  let(:user) { User.create(name: "Test User", role: :admin) }
  let(:token) { user.generate_auth_token }
  let(:headers) { { "Authorization" => "Bearer #{token}" } }

  before { user.save }
end

RSpec.describe AdminController do
  include_context "with logged in user"

  it "allows admin access" do
    get "/admin", headers: headers
    expect(response.status).to eq(200)
  end
end
```

---

## Step 499: Doubles, Mocks, Stubs

```ruby
RSpec.describe PaymentService do
  # double - สร้าง test double
  let(:payment_gateway) { double("PaymentGateway") }

  describe "#charge" do
    context "when payment is successful" do
      before do
        # stub method บน double
        allow(payment_gateway).to receive(:charge).and_return({ status: "success", id: "txn_123" })
      end

      it "returns success" do
        service = PaymentService.new(payment_gateway)
        result = service.charge(amount: 100, card: "4111111111111111")
        expect(result[:status]).to eq("success")
      end
    end

    context "when payment fails" do
      before do
        allow(payment_gateway).to receive(:charge).and_raise(PaymentError, "Card declined")
      end

      it "raises PaymentError" do
        service = PaymentService.new(payment_gateway)
        expect { service.charge(amount: 100, card: "bad_card") }
          .to raise_error(PaymentError, "Card declined")
      end
    end
  end

  # instance_double - stricter double (checks method signatures)
  let(:gateway) { instance_double(PaymentGateway) }

  # allow - stub any method
  before do
    allow(gateway).to receive(:charge).with(any_args).and_return({ status: "ok" })
  end

  # spy - double ที่บันทึก calls
  let(:mailer) { spy("Mailer") }

  it "sends notification after charge" do
    service = PaymentService.new(gateway, mailer)
    service.charge(amount: 50, card: "4111111111111111")
    expect(mailer).to have_received(:send_receipt).once
  end

  # stub chain
  allow(User).to receive_message_chain(:find, :orders, :recent).and_return([])

  # stub with specific arguments
  allow(calculator).to receive(:add).with(1, 2).and_return(3)
  allow(calculator).to receive(:add).with(anything, 0).and_return(anything)
end
```

---

## Step 500: Message Expectations

```ruby
RSpec.describe OrderProcessor do
  let(:inventory) { double("Inventory") }
  let(:mailer)    { double("Mailer") }
  let(:logger)    { double("Logger") }
  let(:processor) { OrderProcessor.new(inventory, mailer, logger) }

  describe "#process" do
    let(:order) { { id: 1, items: [{ sku: "ABC", qty: 2 }], customer_email: "test@test.com" } }

    before do
      allow(inventory).to receive(:reserve).and_return(true)
      allow(mailer).to receive(:send_confirmation)
      allow(logger).to receive(:info)
    end

    it "reserves inventory" do
      expect(inventory).to receive(:reserve).with("ABC", 2)
      processor.process(order)
    end

    it "sends confirmation email" do
      expect(mailer).to receive(:send_confirmation).with("test@test.com", anything)
      processor.process(order)
    end

    it "logs the processing" do
      expect(logger).to receive(:info).with(/Processing order #1/)
      processor.process(order)
    end

    it "reserves inventory before sending email" do
      expect(inventory).to receive(:reserve).ordered
      expect(mailer).to receive(:send_confirmation).ordered
      processor.process(order)
    end

    # expect received N times
    it "logs each item" do
      expect(logger).to receive(:info).at_least(:twice)
      processor.process(order)
    end

    it "sends exactly one email" do
      expect(mailer).to receive(:send_confirmation).exactly(:once)
      processor.process(order)
    end

    # expect NOT to receive
    context "when inventory is unavailable" do
      before do
        allow(inventory).to receive(:reserve).and_return(false)
      end

      it "does not send confirmation" do
        expect(mailer).not_to receive(:send_confirmation)
        processor.process(order)
      end
    end
  end
end
```

---

## Step 501: Testing Exceptions

```ruby
RSpec.describe UserService do
  describe "#create_user" do
    context "when email is invalid" do
      it "raises InvalidEmailError" do
        expect { UserService.create_user(email: "invalid") }
          .to raise_error(InvalidEmailError)
      end

      it "raises with specific message" do
        expect { UserService.create_user(email: "invalid") }
          .to raise_error(InvalidEmailError, "Email format is invalid")
      end

      it "raises with message matching pattern" do
        expect { UserService.create_user(email: "invalid") }
          .to raise_error(InvalidEmailError, /invalid/)
      end

      it "raises specific error class" do
        expect { UserService.create_user(email: "invalid") }
          .to raise_error(InvalidEmailError)
          .with_message("Email format is invalid")
      end
    end

    context "when user already exists" do
      before { User.create(email: "alice@example.com") }

      it "raises DuplicateEmailError" do
        expect { UserService.create_user(email: "alice@example.com") }
          .to raise_error(DuplicateEmailError)
      end
    end

    context "when name is missing" do
      it "raises ArgumentError" do
        expect { UserService.create_user(email: "test@test.com") }
          .to raise_error(ArgumentError)
      end
    end
  end

  describe "#delete_user" do
    it "raises UserNotFoundError for unknown id" do
      expect { UserService.delete_user(99999) }
        .to raise_error(UserNotFoundError)
    end

    # ตรวจสอบว่า TIDAK raise_error
    it "succeeds for existing user" do
      user = User.create(email: "test@test.com", name: "Test")
      expect { UserService.delete_user(user.id) }
        .not_to raise_error
    end
  end
end
```

---

## Step 502: Testing Output

```ruby
RSpec.describe Reporter do
  describe "#print_report" do
    it "outputs report header" do
      reporter = Reporter.new
      expect { reporter.print_report([]) }
        .to output(/=== Report ===/).to_stdout
    end

    it "outputs each item" do
      reporter = Reporter.new
      items = ["item1", "item2", "item3"]
      expect { reporter.print_report(items) }
        .to output("item1\nitem2\nitem3\n").to_stdout
    end

    it "outputs nothing when empty" do
      reporter = Reporter.new
      expect { reporter.print_report([]) }
        .to output("").to_stdout
    end

    # stderr
    it "outputs errors to stderr" do
      expect { STDERR.puts "error message" }
        .to output("error message\n").to_stderr
    end

    # output matching pattern
    it "outputs valid JSON" do
      data = { name: "test" }
      expect { puts data.to_json }
        .to output(/"name":"test"/).to_stdout
    end
  end
end
```

---

## Step 503: Test Coverage กับ SimpleCov

```ruby
# Gemfile
# group :development, :test do
#   gem 'simplecov', require: false
# end

# spec/spec_helper.rb (ต้องอยู่บนสุดก่อน require อื่นๆ)
require 'simplecov'
SimpleCov.start do
  add_filter '/spec/'
  add_filter '/config/'

  add_group "Models", "lib/models"
  add_group "Services", "lib/services"

  minimum_coverage 90
end

# รัน rspec จะสร้าง coverage/index.html
# เปิดไฟล์นั้นดู visual report

# config advanced
SimpleCov.start do
  enable_coverage :branch  # branch coverage

  # Refuse รัน ถ้า coverage ต่ำกว่า threshold
  minimum_coverage line: 90, branch: 80

  # Format
  formatter SimpleCov::Formatter::MultiFormatter.new([
    SimpleCov::Formatter::HTMLFormatter,
    SimpleCov::Formatter::SimpleFormatter
  ])
end

# ดู coverage ใน output
# Coverage report generated for RSpec to coverage. 1505 / 1625 LOC (92.62%) covered.
```

---

## Step 504: Factory Bot Integration

```ruby
# Gemfile
# gem 'factory_bot_rspec', group: :test
# gem 'faker', group: :test

# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name  { Faker::Name.full_name }
    email { Faker::Internet.email }
    age   { rand(18..80) }
    role  { :user }
    active { true }

    # Traits
    trait :admin do
      role { :admin }
    end

    trait :inactive do
      active { false }
    end

    trait :young do
      age { rand(18..25) }
    end

    # Nested factory
    factory :admin_user, traits: [:admin]
    factory :inactive_user, traits: [:inactive]
  end

  factory :order do
    user
    total   { rand(10.0..1000.0).round(2) }
    status  { :pending }
    items_count { rand(1..10) }

    trait :completed do
      status { :completed }
    end

    trait :cancelled do
      status { :cancelled }
    end
  end
end

# spec/spec_helper.rb
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end

# ใช้ใน specs
RSpec.describe UserService do
  describe "#activate" do
    let(:inactive_user) { create(:inactive_user) }

    it "activates the user" do
      UserService.activate(inactive_user)
      expect(inactive_user.reload.active).to be true
    end
  end

  describe "#find_admins" do
    let!(:admin1) { create(:admin_user) }
    let!(:admin2) { create(:admin_user) }
    let!(:regular_user) { create(:user) }

    it "returns only admins" do
      admins = UserService.find_admins
      expect(admins).to include(admin1, admin2)
      expect(admins).not_to include(regular_user)
    end
  end

  # Build (ไม่บันทึก)
  let(:unsaved_user) { build(:user) }

  # Build stubbed (ไม่ call DB)
  let(:stubbed_user) { build_stubbed(:user) }

  # Attributes only
  let(:user_attrs) { attributes_for(:user) }
  it "creates valid user" do
    expect(User.new(user_attrs)).to be_valid
  end

  # Create list
  let!(:users) { create_list(:user, 5) }
  it "has 5 users" do
    expect(User.count).to eq(5)
  end
end
```

---

## Step 505: Testing Private Methods

```ruby
# ไม่แนะนำ! แต่บางครั้งจำเป็น
class PasswordValidator
  def valid?(password)
    check_length(password) &&
      check_complexity(password)
  end

  private

  def check_length(password)
    password.length >= 8
  end

  def check_complexity(password)
    password.match?(/[A-Z]/) &&
      password.match?(/[0-9]/) &&
      password.match?(/[^A-Za-z0-9]/)
  end
end

RSpec.describe PasswordValidator do
  let(:validator) { PasswordValidator.new }

  # วิธีที่ 1: ใช้ send
  describe "#check_length (private)" do
    it "returns true for long password" do
      expect(validator.send(:check_length, "longpassword")).to be true
    end

    it "returns false for short password" do
      expect(validator.send(:check_length, "abc")).to be false
    end
  end

  # วิธีที่ดีกว่า: test ผ่าน public interface
  describe "#valid?" do
    it "rejects short passwords" do
      expect(validator.valid?("Short1!")).to be false
    end

    it "rejects passwords without numbers" do
      expect(validator.valid?("LongPassword!")).to be false
    end

    it "accepts strong passwords" do
      expect(validator.valid?("StrongPass1!")).to be true
    end
  end

  # instance_eval สำหรับ access private
  it "checks length" do
    result = validator.instance_eval { check_length("longpass") }
    expect(result).to be true
  end
end
```

---

## Step 506: Custom Matchers

```ruby
# spec/support/matchers/be_valid_email.rb
RSpec::Matchers.define :be_valid_email do
  match do |actual|
    actual.to_s.match?(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
  end

  failure_message do |actual|
    "expected #{actual.inspect} to be a valid email address"
  end

  failure_message_when_negated do |actual|
    "expected #{actual.inspect} not to be a valid email address"
  end

  description do
    "be a valid email address"
  end
end

# ใช้งาน
expect("alice@example.com").to be_valid_email
expect("invalid").not_to be_valid_email

# Custom matcher กับ parameters
RSpec::Matchers.define :have_error_on do |field|
  match do |model|
    model.errors[field].any?
  end

  failure_message do |model|
    "expected #{model.class} to have error on #{field}, but errors were: #{model.errors.full_messages}"
  end
end

expect(user).to have_error_on(:email)
expect(user).not_to have_error_on(:name)

# Compound matchers
RSpec::Matchers.define :be_a_positive_integer do
  match do |actual|
    actual.is_a?(Integer) && actual > 0
  end
end

expect(5).to be_a_positive_integer
expect(-1).not_to be_a_positive_integer
expect(5.0).not_to be_a_positive_integer

# Custom matcher ที่ซับซ้อนขึ้น
RSpec::Matchers.define :have_sent_email do
  chain :to do |email|
    @to = email
  end

  chain :with_subject do |subject|
    @subject = subject
  end

  match do |_|
    ActionMailer::Base.deliveries.any? do |mail|
      (@to.nil? || mail.to.include?(@to)) &&
        (@subject.nil? || mail.subject == @subject)
    end
  end

  failure_message do
    "expected email to be sent" +
      (@to ? " to #{@to}" : "") +
      (@subject ? " with subject '#{@subject}'" : "")
  end
end

# ใช้งาน
expect(ActionMailer::Base.deliveries).to have_sent_email.to("alice@test.com").with_subject("Welcome!")
```

---

## Step 507-510: Complete Test Suite ตัวอย่าง

```ruby
# lib/bank_account.rb
class BankAccount
  class InsufficientFundsError < StandardError; end
  class InvalidAmountError < StandardError; end
  class AccountFrozenError < StandardError; end

  attr_reader :balance, :owner, :account_number, :transactions

  def initialize(owner, initial_balance = 0)
    @owner          = owner
    @balance        = initial_balance.to_f
    @account_number = generate_account_number
    @transactions   = []
    @frozen         = false
  end

  def deposit(amount)
    validate_amount!(amount)
    raise AccountFrozenError, "Account is frozen" if frozen?

    @balance += amount
    @transactions << { type: :deposit, amount: amount, balance: @balance, timestamp: Time.now }
    amount
  end

  def withdraw(amount)
    validate_amount!(amount)
    raise AccountFrozenError, "Account is frozen" if frozen?
    raise InsufficientFundsError, "Insufficient funds" if amount > @balance

    @balance -= amount
    @transactions << { type: :withdrawal, amount: amount, balance: @balance, timestamp: Time.now }
    amount
  end

  def transfer(amount, target_account)
    withdraw(amount)
    target_account.deposit(amount)
    amount
  end

  def freeze!
    @frozen = true
  end

  def unfreeze!
    @frozen = false
  end

  def frozen?
    @frozen
  end

  def statement
    lines = ["Account: #{@account_number}", "Owner: #{@owner}", "-" * 40]
    @transactions.each do |t|
      type = t[:type].to_s.capitalize
      lines << "#{t[:timestamp].strftime('%Y-%m-%d %H:%M')} | #{type.ljust(10)} | #{format('%.2f', t[:amount]).rjust(10)} | Balance: #{format('%.2f', t[:balance])}"
    end
    lines << "-" * 40
    lines << "Current Balance: #{format('%.2f', @balance)}"
    lines.join("\n")
  end

  private

  def validate_amount!(amount)
    raise InvalidAmountError, "Amount must be positive" unless amount.is_a?(Numeric) && amount > 0
  end

  def generate_account_number
    "ACC#{SecureRandom.hex(4).upcase}"
  end
end

# spec/bank_account_spec.rb
require 'bank_account'

RSpec.describe BankAccount do
  let(:account)     { BankAccount.new("Alice", 1000) }
  let(:other_account) { BankAccount.new("Bob", 500) }

  describe "initialization" do
    it "sets the owner" do
      expect(account.owner).to eq("Alice")
    end

    it "sets initial balance" do
      expect(account.balance).to eq(1000.0)
    end

    it "generates account number" do
      expect(account.account_number).to match(/\AACC[A-F0-9]{8}\z/)
    end

    it "starts with empty transactions" do
      expect(account.transactions).to be_empty
    end

    it "starts unfrozen" do
      expect(account).not_to be_frozen
    end

    context "with zero initial balance" do
      let(:zero_account) { BankAccount.new("Empty") }
      it "has zero balance" do
        expect(zero_account.balance).to eq(0.0)
      end
    end
  end

  describe "#deposit" do
    context "with valid amount" do
      it "increases balance" do
        expect { account.deposit(500) }
          .to change { account.balance }.by(500)
      end

      it "returns the deposited amount" do
        expect(account.deposit(200)).to eq(200)
      end

      it "records transaction" do
        account.deposit(300)
        last_txn = account.transactions.last
        expect(last_txn[:type]).to eq(:deposit)
        expect(last_txn[:amount]).to eq(300)
        expect(last_txn[:balance]).to eq(1300)
      end
    end

    context "with invalid amount" do
      it "raises InvalidAmountError for zero" do
        expect { account.deposit(0) }.to raise_error(BankAccount::InvalidAmountError)
      end

      it "raises InvalidAmountError for negative" do
        expect { account.deposit(-100) }.to raise_error(BankAccount::InvalidAmountError)
      end

      it "raises InvalidAmountError for string" do
        expect { account.deposit("100") }.to raise_error(BankAccount::InvalidAmountError)
      end
    end

    context "when account is frozen" do
      before { account.freeze! }

      it "raises AccountFrozenError" do
        expect { account.deposit(100) }
          .to raise_error(BankAccount::AccountFrozenError, "Account is frozen")
      end

      it "does not change balance" do
        expect { account.deposit(100) rescue nil }
          .not_to change { account.balance }
      end
    end
  end

  describe "#withdraw" do
    context "with sufficient funds" do
      it "decreases balance" do
        expect { account.withdraw(300) }
          .to change { account.balance }.by(-300)
      end

      it "returns withdrawn amount" do
        expect(account.withdraw(200)).to eq(200)
      end

      it "records transaction" do
        account.withdraw(400)
        last_txn = account.transactions.last
        expect(last_txn[:type]).to eq(:withdrawal)
        expect(last_txn[:amount]).to eq(400)
        expect(last_txn[:balance]).to eq(600)
      end
    end

    context "with insufficient funds" do
      it "raises InsufficientFundsError" do
        expect { account.withdraw(2000) }
          .to raise_error(BankAccount::InsufficientFundsError, "Insufficient funds")
      end

      it "does not change balance" do
        expect { account.withdraw(2000) rescue nil }
          .not_to change { account.balance }
      end
    end

    context "with exact balance" do
      it "allows withdrawal" do
        expect { account.withdraw(1000) }
          .not_to raise_error
        expect(account.balance).to eq(0)
      end
    end
  end

  describe "#transfer" do
    it "moves money between accounts" do
      expect { account.transfer(200, other_account) }
        .to change { account.balance }.by(-200)
        .and change { other_account.balance }.by(200)
    end

    it "raises InsufficientFundsError if not enough balance" do
      expect { account.transfer(5000, other_account) }
        .to raise_error(BankAccount::InsufficientFundsError)
    end

    it "returns transferred amount" do
      expect(account.transfer(100, other_account)).to eq(100)
    end
  end

  describe "#freeze! and #unfreeze!" do
    it "freezes the account" do
      account.freeze!
      expect(account).to be_frozen
    end

    it "unfreezes the account" do
      account.freeze!
      account.unfreeze!
      expect(account).not_to be_frozen
    end

    it "allows deposits after unfreeze" do
      account.freeze!
      account.unfreeze!
      expect { account.deposit(100) }.not_to raise_error
    end
  end

  describe "#statement" do
    before do
      account.deposit(500)
      account.withdraw(200)
      account.deposit(100)
    end

    it "includes account number" do
      expect(account.statement).to include(account.account_number)
    end

    it "includes owner name" do
      expect(account.statement).to include("Alice")
    end

    it "shows current balance" do
      expect(account.statement).to include("1400.00")
    end

    it "includes all transactions" do
      expect(account.statement).to include("Deposit")
      expect(account.statement).to include("Withdrawal")
    end
  end
end
```

---

## Step 511-520: แบบฝึกหัด (30 ข้อ)

### ข้อที่ 1-10: Basic RSpec

**ข้อ 1:** เขียน spec สำหรับ FizzBuzz method

```ruby
# lib/fizzbuzz.rb
def fizzbuzz(n)
  if n % 15 == 0 then "FizzBuzz"
  elsif n % 3 == 0 then "Fizz"
  elsif n % 5 == 0 then "Buzz"
  else n.to_s
  end
end

# spec/fizzbuzz_spec.rb
# เฉลย
RSpec.describe "#fizzbuzz" do
  it "returns 'Fizz' for multiples of 3" do
    expect(fizzbuzz(3)).to eq("Fizz")
    expect(fizzbuzz(9)).to eq("Fizz")
  end

  it "returns 'Buzz' for multiples of 5" do
    expect(fizzbuzz(5)).to eq("Buzz")
    expect(fizzbuzz(10)).to eq("Buzz")
  end

  it "returns 'FizzBuzz' for multiples of 15" do
    expect(fizzbuzz(15)).to eq("FizzBuzz")
    expect(fizzbuzz(30)).to eq("FizzBuzz")
  end

  it "returns string number for other cases" do
    expect(fizzbuzz(1)).to eq("1")
    expect(fizzbuzz(7)).to eq("7")
  end
end
```

**ข้อ 2:** เขียน spec สำหรับ Stack class

```ruby
# เฉลย - spec/stack_spec.rb
class Stack
  def initialize
    @data = []
  end

  def push(item)
    @data.push(item)
    self
  end

  def pop
    raise "Stack underflow" if empty?
    @data.pop
  end

  def peek
    raise "Stack is empty" if empty?
    @data.last
  end

  def empty?
    @data.empty?
  end

  def size
    @data.size
  end
end

RSpec.describe Stack do
  subject(:stack) { Stack.new }

  it { is_expected.to be_empty }
  it { expect(stack.size).to eq(0) }

  describe "#push" do
    it "adds item to stack" do
      stack.push(1)
      expect(stack.size).to eq(1)
    end

    it "returns self for chaining" do
      expect(stack.push(1).push(2)).to eq(stack)
      expect(stack.size).to eq(2)
    end
  end

  describe "#pop" do
    context "with items" do
      before { stack.push(1).push(2).push(3) }

      it "removes top item" do
        stack.pop
        expect(stack.size).to eq(2)
      end

      it "returns top item" do
        expect(stack.pop).to eq(3)
      end

      it "follows LIFO order" do
        expect(stack.pop).to eq(3)
        expect(stack.pop).to eq(2)
        expect(stack.pop).to eq(1)
      end
    end

    context "when empty" do
      it "raises error" do
        expect { stack.pop }.to raise_error("Stack underflow")
      end
    end
  end

  describe "#peek" do
    it "returns top without removing" do
      stack.push("hello")
      expect(stack.peek).to eq("hello")
      expect(stack.size).to eq(1)
    end

    it "raises when empty" do
      expect { stack.peek }.to raise_error("Stack is empty")
    end
  end
end
```

**ข้อ 3:** Test สำหรับ String extension

```ruby
# เฉลย
class String
  def word_count
    split.size
  end

  def palindrome?
    cleaned = downcase.gsub(/[^a-z0-9]/, '')
    cleaned == cleaned.reverse
  end

  def truncate(max_length, omission: "...")
    return self if length <= max_length
    self[0, max_length - omission.length] + omission
  end
end

RSpec.describe String do
  describe "#word_count" do
    it "counts words in sentence" do
      expect("hello world ruby".word_count).to eq(3)
    end

    it "handles multiple spaces" do
      expect("hello  world".word_count).to eq(2)
    end

    it "returns 0 for empty string" do
      expect("".word_count).to eq(0)
    end
  end

  describe "#palindrome?" do
    it "recognizes simple palindromes" do
      expect("racecar").to be_palindrome
      expect("level").to be_palindrome
    end

    it "ignores case" do
      expect("Racecar").to be_palindrome
    end

    it "ignores punctuation" do
      expect("A man, a plan, a canal: Panama").to be_palindrome
    end

    it "returns false for non-palindromes" do
      expect("hello").not_to be_palindrome
    end
  end

  describe "#truncate" do
    it "truncates long strings" do
      expect("Hello, World!".truncate(8)).to eq("Hello...")
    end

    it "does not truncate short strings" do
      expect("Hi".truncate(10)).to eq("Hi")
    end

    it "uses custom omission" do
      expect("Hello, World!".truncate(8, omission: " [...]")).to eq("He [...]")
    end
  end
end
```

**ข้อ 4-10:** (ตัวอย่างย่อ)

```ruby
# ข้อ 4: Test Array extensions
RSpec.describe Array do
  describe "#second" do
    it "returns second element" do
      expect([1, 2, 3].second).to eq(2)
    end

    it "returns nil for single element" do
      expect([1].second).to be_nil
    end
  end
end

# ข้อ 5: Test Hash methods
RSpec.describe Hash do
  describe "#deep_merge" do
    it "merges nested hashes" do
      h1 = { a: { b: 1, c: 2 } }
      h2 = { a: { c: 3, d: 4 } }
      expect(h1.deep_merge(h2)).to eq({ a: { b: 1, c: 3, d: 4 } })
    end
  end
end

# ข้อ 6: Test with mocks
RSpec.describe WeatherService do
  let(:http_client) { double("HttpClient") }
  let(:service)     { WeatherService.new(http_client) }

  it "returns temperature" do
    allow(http_client).to receive(:get)
      .with("/weather?city=Bangkok")
      .and_return({ temp: 35, unit: "C" })

    expect(service.temperature("Bangkok")).to eq("35°C")
  end
end

# ข้อ 7: Shared examples
RSpec.shared_examples "a notification sender" do |channel|
  it "sends notification" do
    expect(subject).to respond_to(:send_notification)
  end

  it "returns true on success" do
    allow(subject).to receive(:deliver).and_return(true)
    expect(subject.send_notification("Hello")).to be true
  end
end

# ข้อ 8: Test exceptions with detail
RSpec.describe ApiClient do
  describe "#request" do
    it "raises NetworkError with status code" do
      allow(Net::HTTP).to receive(:get_response).and_raise(Net::TimeoutError)

      expect { ApiClient.new.get("/api/users") }
        .to raise_error(ApiClient::NetworkError) { |e|
          expect(e.message).to include("timeout")
        }
    end
  end
end

# ข้อ 9: Test output
RSpec.describe ProgressBar do
  describe "#render" do
    it "shows progress percentage" do
      bar = ProgressBar.new(100)
      bar.update(50)
      expect { bar.render }
        .to output(/50%/).to_stdout
    end
  end
end

# ข้อ 10: Change matchers
RSpec.describe UserRepository do
  let(:repo) { UserRepository.new }

  describe "#create" do
    it "increases user count" do
      expect { repo.create(name: "Alice") }
        .to change { repo.count }.by(1)
    end

    it "assigns an id" do
      user = repo.create(name: "Alice")
      expect(user.id).not_to be_nil
    end
  end

  describe "#delete" do
    let!(:user) { repo.create(name: "Bob") }

    it "decreases count" do
      expect { repo.delete(user.id) }
        .to change { repo.count }.by(-1)
    end

    it "changes multiple things" do
      expect { repo.delete(user.id) }
        .to change { repo.count }.by(-1)
        .and change { repo.exists?(user.id) }.from(true).to(false)
    end
  end
end
```

### ข้อที่ 11-20: Intermediate

```ruby
# ข้อ 11: Integration test สำหรับ Order system
RSpec.describe "Order Processing" do
  let(:user)    { create(:user) }
  let(:product) { create(:product, price: 100, stock: 5) }

  it "completes order flow" do
    order = Order.create(user: user)
    order.add_item(product, quantity: 2)

    expect(order.total).to eq(200)
    expect { order.checkout }.to change { product.reload.stock }.by(-2)
    expect(order.status).to eq(:completed)
  end
end

# ข้อ 12: Test JSON API
RSpec.describe ApiController do
  describe "GET /api/users" do
    let!(:users) { create_list(:user, 3) }

    it "returns users as JSON" do
      get "/api/users", headers: { "Accept" => "application/json" }
      json = JSON.parse(response.body)
      expect(json.length).to eq(3)
      expect(json.first).to include("name", "email")
    end

    it "returns 200 status" do
      get "/api/users"
      expect(response.status).to eq(200)
    end
  end
end

# ข้อ 13-20: Test สำหรับ specific patterns
# ข้อ 13: Stub time
RSpec.describe ScheduledJob do
  it "runs at midnight" do
    allow(Time).to receive(:now).and_return(Time.new(2024, 1, 15, 0, 0, 0))
    expect(ScheduledJob.should_run?).to be true
  end
end

# ข้อ 14: Test memoization
RSpec.describe ExpensiveCalculator do
  let(:calc) { ExpensiveCalculator.new }

  it "only calculates once" do
    expect(calc).to receive(:perform_calculation).once
    3.times { calc.result }
  end
end

# ข้อ 15: Test callbacks
RSpec.describe Document do
  it "calls after_save callback" do
    doc = Document.new(title: "Test")
    expect(doc).to receive(:notify_subscribers).once
    doc.save
  end
end

# ข้อ 16: Test with frozen time
require 'timecop'  # gem 'timecop'

RSpec.describe Session do
  describe "#expired?" do
    it "expires after 30 minutes" do
      session = Session.new
      Timecop.travel(31.minutes) do
        expect(session).to be_expired
      end
    end

    it "is not expired within 30 minutes" do
      session = Session.new
      Timecop.travel(29.minutes) do
        expect(session).not_to be_expired
      end
    end
  end
end

# ข้อ 17: Custom aggregate failures
RSpec.describe UserProfile do
  let(:profile) { UserProfile.new(age: 25, name: "Alice", bio: "Developer") }

  it "has all required attributes", aggregate_failures: true do
    expect(profile.name).to eq("Alice")
    expect(profile.age).to eq(25)
    expect(profile.bio).to include("Developer")
    expect(profile).to be_complete
  end
end

# ข้อ 18: Verify doubles
RSpec.describe EmailSender do
  let(:mailer) { instance_double(UserMailer) }

  it "calls deliver_now" do
    allow(mailer).to receive(:send_welcome).and_return(true)
    EmailSender.new(mailer).welcome("alice@test.com")
    expect(mailer).to have_received(:send_welcome).with("alice@test.com")
  end
end

# ข้อ 19: Test async behavior
RSpec.describe AsyncWorker do
  it "processes job" do
    job = create(:job, status: :pending)
    AsyncWorker.perform_now(job.id)
    expect(job.reload.status).to eq(:completed)
  end
end

# ข้อ 20: Complex matcher
RSpec::Matchers.define :have_pagination do
  match do |response|
    json = JSON.parse(response.body)
    json.key?("data") &&
      json.key?("meta") &&
      json["meta"].key?("total") &&
      json["meta"].key?("page")
  end
end

RSpec.describe ApiController do
  it "returns paginated response" do
    get "/api/users?page=1&per_page=10"
    expect(response).to have_pagination
  end
end
```

### ข้อที่ 21-30: ขั้นสูง

```ruby
# ข้อ 21: Full BDD spec สำหรับ feature
RSpec.describe "User Registration", type: :feature do
  scenario "successful registration" do
    visit "/register"
    fill_in "Name", with: "Alice"
    fill_in "Email", with: "alice@example.com"
    fill_in "Password", with: "Password1!"
    click_button "Register"

    expect(page).to have_text("Welcome, Alice!")
    expect(current_path).to eq("/dashboard")
  end

  scenario "failed registration with invalid email" do
    visit "/register"
    fill_in "Email", with: "invalid"
    click_button "Register"

    expect(page).to have_text("Email is invalid")
    expect(current_path).to eq("/register")
  end
end

# ข้อ 22-30 (simplified)
# ข้อ 22: Performance spec
RSpec.describe "Large dataset processing" do
  it "processes 10000 records quickly" do
    data = create_list(:record, 10000)
    start = Time.now
    DataProcessor.process(data)
    elapsed = Time.now - start
    expect(elapsed).to be < 5  # ไม่เกิน 5 วินาที
  end
end

# ข้อ 23: Shared context กับ database
RSpec.shared_context "clean database" do
  before(:all)  { DatabaseCleaner.start }
  after(:all)   { DatabaseCleaner.clean }
  before(:each) { DatabaseCleaner.clean_with(:truncation) }
end

# ข้อ 24: Test observers/subscribers
RSpec.describe EventBus do
  it "notifies subscribers" do
    handler = spy("EventHandler")
    EventBus.subscribe(:user_created, handler)
    EventBus.publish(:user_created, { id: 1 })
    expect(handler).to have_received(:call).with(id: 1)
  end
end

# ข้อ 25: Stub external service
RSpec.describe WeatherWidget do
  before do
    stub_request(:get, "https://api.weather.com/current")
      .to_return(body: { temperature: 30, city: "Bangkok" }.to_json)
  end

  it "displays current temperature" do
    widget = WeatherWidget.new
    expect(widget.display).to include("30°C")
  end
end

# ข้อ 26: Test mailer
RSpec.describe UserMailer do
  describe "#welcome_email" do
    let(:user) { create(:user) }
    let(:mail) { UserMailer.welcome_email(user) }

    it "renders headers" do
      expect(mail.to).to eq([user.email])
      expect(mail.subject).to eq("Welcome to our app!")
    end

    it "renders body" do
      expect(mail.body.encoded).to include(user.name)
    end
  end
end

# ข้อ 27: Test file operations
RSpec.describe FileProcessor do
  around do |example|
    Dir.mktmpdir do |tmpdir|
      @tmpdir = tmpdir
      example.run
    end
  end

  it "processes CSV file" do
    csv_file = File.join(@tmpdir, "data.csv")
    File.write(csv_file, "name,age\nAlice,30\nBob,25")

    result = FileProcessor.process(csv_file)
    expect(result.count).to eq(2)
    expect(result.first[:name]).to eq("Alice")
  end
end

# ข้อ 28: Test retry logic
RSpec.describe RetryableService do
  let(:flaky_api) { double("FlakyApi") }

  it "retries on failure then succeeds" do
    call_count = 0
    allow(flaky_api).to receive(:call) do
      call_count += 1
      raise "Error" if call_count < 3
      "success"
    end

    service = RetryableService.new(flaky_api, max_retries: 3)
    expect(service.execute).to eq("success")
    expect(call_count).to eq(3)
  end
end

# ข้อ 29: Test background jobs
RSpec.describe EmailNotificationJob do
  it "enqueues email job" do
    expect { EmailNotificationJob.perform_later(user_id: 1) }
      .to have_enqueued_job(EmailNotificationJob)
      .with(user_id: 1)
  end
end

# ข้อ 30: Complete TDD cycle demo
# 1. เขียน failing test
RSpec.describe "Shopping Cart" do
  describe "#apply_discount" do
    let(:cart) { ShoppingCart.new }

    before do
      cart.add_item("item1", 100)
      cart.add_item("item2", 200)
    end

    context "with 10% discount code" do
      it "reduces total by 10%" do
        cart.apply_discount("SAVE10", :percentage, 10)
        expect(cart.total).to eq(270)
      end
    end

    context "with flat discount" do
      it "reduces total by fixed amount" do
        cart.apply_discount("SAVE50", :flat, 50)
        expect(cart.total).to eq(250)
      end
    end

    context "with invalid code" do
      it "raises InvalidDiscountError" do
        expect { cart.apply_discount("INVALID", :percentage, 10) }
          .to raise_error(ShoppingCart::InvalidDiscountError)
      end
    end
  end
end
```

---

## สรุป RSpec

### Structure
```
RSpec.describe ClassName do
  context "situation" do
    let(:object) { ClassName.new }
    before { setup }

    it "does something" do
      expect(object.method).to matcher
    end
  end
end
```

### Matchers สำคัญ
| Matcher | ใช้เพื่อ |
|---------|---------|
| `eq` | ค่าเท่ากัน |
| `be_truthy/be_falsy` | true/false |
| `be_nil` | nil |
| `include` | มี element |
| `raise_error` | exception |
| `change` | ค่าเปลี่ยน |
| `output` | stdout/stderr |
| `receive` | method call |
| `have_received` | spy verification |

### Best Practices
1. **One assertion per test** (หรือ aggregate_failures)
2. **Descriptive names** - อ่านแล้วเข้าใจ
3. **AAA pattern** - Arrange, Act, Assert
4. **Avoid test interdependence** - ทุก test ต้องรันได้อิสระ
5. **Mock ส่วนที่ slow/external** - database, API, file system
6. **Use factories** ไม่ใช่ fixtures
7. **Test behavior ไม่ใช่ implementation**

---

*ตอนถัดไป: ตอนที่ 24 - Gems และ Bundler*
