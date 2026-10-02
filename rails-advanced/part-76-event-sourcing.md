# Part 76: Event Sourcing and CQRS ใน Ruby on Rails

## ขั้นตอนที่ 1641-1660

---

## ขั้นตอนที่ 1641: แนวคิด Event Sourcing คืออะไร?

Event Sourcing เป็นรูปแบบการออกแบบซอฟต์แวร์ที่แทนที่การเก็บ "state ปัจจุบัน" ด้วยการเก็บ "ลำดับของ events ที่เกิดขึ้น" State ปัจจุบันจะถูกคำนวณโดยการ replay events ทั้งหมด

### ปัญหาของ Traditional CRUD

```
# Traditional CRUD - เราเห็นแค่ state ล่าสุด
UPDATE accounts SET balance = 500 WHERE id = 1;
-- ไม่รู้ว่า balance เปลี่ยนจาก 1000 เป็น 500 ได้อย่างไร
-- ไม่รู้ว่าใครทำ, เมื่อไหร่, ทำไม
```

### แนวคิด Event Sourcing

```ruby
# แทนที่จะ update balance โดยตรง เราเก็บ events
events = [
  { type: 'AccountOpened',    amount: 1000, at: '2024-01-01' },
  { type: 'MoneyDeposited',   amount: 500,  at: '2024-01-15' },
  { type: 'MoneyWithdrawn',   amount: 300,  at: '2024-01-20' },
  { type: 'MoneyWithdrawn',   amount: 700,  at: '2024-01-25' },
]

# State ปัจจุบัน = replay events ทั้งหมด
balance = events.reduce(0) do |acc, event|
  case event[:type]
  when 'AccountOpened'  then acc + event[:amount]
  when 'MoneyDeposited' then acc + event[:amount]
  when 'MoneyWithdrawn' then acc - event[:amount]
  else acc
  end
end
# => 500
```

### ประโยชน์ของ Event Sourcing

1. **Audit Trail**: รู้ทุกอย่างที่เกิดขึ้นในระบบ
2. **Time Travel**: ดูสถานะของระบบ ณ เวลาใดก็ได้
3. **Event Replay**: rebuild state ใหม่ได้เสมอ
4. **Debugging**: ง่ายต่อการ debug เพราะเห็น history ทั้งหมด
5. **Analytics**: วิเคราะห์ business events ได้ง่าย

---

## ขั้นตอนที่ 1642: CQRS คืออะไร?

CQRS (Command Query Responsibility Segregation) คือการแยก "การเขียน" (Commands) ออกจาก "การอ่าน" (Queries)

### แนวคิดพื้นฐาน

```ruby
# ❌ Traditional - ทั้ง read และ write อยู่ด้วยกัน
class AccountController < ApplicationController
  def show
    @account = Account.find(params[:id])
    # อ่านและแสดงข้อมูล
  end

  def deposit
    account = Account.find(params[:id])
    account.update!(balance: account.balance + params[:amount].to_f)
    # เขียนและ return ผล
  end
end

# ✅ CQRS - แยก read model และ write model
class AccountCommandService
  def deposit(account_id:, amount:)
    # เขียน: สร้าง event
    EventStore.append(
      stream: "account-#{account_id}",
      event: MoneyDeposited.new(account_id: account_id, amount: amount)
    )
  end
end

class AccountQueryService
  def find(account_id)
    # อ่าน: จาก read model (projection)
    AccountReadModel.find(account_id)
  end
end
```

### Command vs Query

| Command | Query |
|---------|-------|
| เปลี่ยน state | อ่าน state |
| ไม่ return data | return data |
| สร้าง events | อ่านจาก projections |
| ผ่าน validation | เร็วกว่า |

---

## ขั้นตอนที่ 1643: ติดตั้ง RailsEventStore Gem

RailsEventStore (RES) เป็น gem ที่ได้รับความนิยมสูงสุดสำหรับ Event Sourcing ใน Rails

### การติดตั้ง

```ruby
# Gemfile
gem 'rails_event_store'
gem 'aggregate_root'   # สำหรับ Aggregate pattern
```

```bash
bundle install
```

### สร้าง Migration

```bash
rails generate rails_event_store_active_record:migration
rails db:migrate
```

Migration ที่สร้างจะมี table `event_store_events` และ `event_store_events_in_streams`

### ตั้งค่าใน Rails

```ruby
# config/initializers/rails_event_store.rb
require 'rails_event_store'
require 'aggregate_root'

Rails.configuration.to_prepare do
  Rails.configuration.event_store = RailsEventStore::Client.new

  Rails.configuration.event_store.tap do |store|
    # Subscribe handlers ที่นี่
    store.subscribe(
      SendWelcomeEmail,
      to: [UserRegistered]
    )

    store.subscribe(
      NotifyAdminOnLargeDeposit,
      to: [MoneyDeposited]
    )
  end
end
```

---

## ขั้นตอนที่ 1644: การสร้าง Events

Events คือ records ที่บอกว่า "อะไรเกิดขึ้น" ในระบบ

### สร้าง Event Classes

```ruby
# app/events/user_registered.rb
class UserRegistered < RailsEventStore::Event
  # ไม่ต้องกำหนด attributes เพิ่ม เพราะ Event มี data hash อยู่แล้ว
end

# app/events/money_deposited.rb
class MoneyDeposited < RailsEventStore::Event
  # validate ข้อมูลใน event
  def validate
    raise ArgumentError, "amount must be positive" unless data[:amount].positive?
    raise ArgumentError, "account_id is required" unless data[:account_id]
  end
end

# app/events/order_placed.rb
class OrderPlaced < RailsEventStore::Event
end

# app/events/order_shipped.rb
class OrderShipped < RailsEventStore::Event
end

# app/events/order_cancelled.rb
class OrderCancelled < RailsEventStore::Event
end
```

### Publish Events

```ruby
# ใน Controller หรือ Service
class OrdersController < ApplicationController
  def create
    order_id = SecureRandom.uuid

    event_store = Rails.configuration.event_store
    event_store.publish(
      OrderPlaced.new(data: {
        order_id: order_id,
        customer_id: current_user.id,
        items: params[:items],
        total: calculate_total(params[:items]),
        placed_at: Time.current
      }),
      stream_name: "order-#{order_id}"
    )

    render json: { order_id: order_id }, status: :created
  end
end
```

---

## ขั้นตอนที่ 1645: Event Handlers

Event Handlers คือ code ที่ตอบสนองต่อ events

### สร้าง Event Handler

```ruby
# app/event_handlers/send_welcome_email.rb
class SendWelcomeEmail
  def call(event)
    user_id = event.data[:user_id]
    email   = event.data[:email]
    name    = event.data[:name]

    UserMailer.welcome(
      user_id: user_id,
      email:   email,
      name:    name
    ).deliver_later
  end
end

# app/event_handlers/update_account_balance.rb
class UpdateAccountBalance
  def call(event)
    account_id = event.data[:account_id]
    amount     = event.data[:amount]

    # อัพเดท read model
    AccountReadModel.find_or_create_by(account_id: account_id) do |model|
      model.balance = 0
    end.tap do |model|
      case event.class.name
      when 'MoneyDeposited'
        model.increment!(:balance, amount)
      when 'MoneyWithdrawn'
        model.decrement!(:balance, amount)
      end
    end
  end
end

# app/event_handlers/notify_admin_on_large_deposit.rb
class NotifyAdminOnLargeDeposit
  LARGE_AMOUNT_THRESHOLD = 100_000

  def call(event)
    return unless event.data[:amount] >= LARGE_AMOUNT_THRESHOLD

    AdminMailer.large_deposit_notification(
      account_id: event.data[:account_id],
      amount:     event.data[:amount]
    ).deliver_later
  end
end
```

### ลงทะเบียน Handlers

```ruby
# config/initializers/rails_event_store.rb
Rails.configuration.event_store.tap do |store|
  # Subscribe เดี่ยว
  store.subscribe(SendWelcomeEmail.new, to: [UserRegistered])

  # Subscribe หลาย events
  store.subscribe(
    UpdateAccountBalance.new,
    to: [MoneyDeposited, MoneyWithdrawn, AccountOpened]
  )

  # Async handler
  store.subscribe(
    NotifyAdminOnLargeDeposit,  # class (ไม่ใช่ instance) = async
    to: [MoneyDeposited]
  )
end
```

### Async Event Handler (ActiveJob)

```ruby
# app/event_handlers/notify_admin_on_large_deposit.rb
class NotifyAdminOnLargeDeposit < ActiveJob::Base
  prepend RailsEventStore::AsyncHandler

  queue_as :notifications

  def perform(event)
    return unless event.data[:amount] >= 100_000

    AdminMailer.large_deposit_notification(
      account_id: event.data[:account_id],
      amount:     event.data[:amount]
    ).deliver_now
  end
end
```

---

## ขั้นตอนที่ 1646: Projections

Projections คือการ rebuild state จาก events เพื่อ query อย่างมีประสิทธิภาพ

### สร้าง Projection

```ruby
# app/projections/account_balance_projection.rb
class AccountBalanceProjection
  def initialize(event_store)
    @event_store = event_store
  end

  def balance_for(account_id)
    events = @event_store.read
                         .stream("account-#{account_id}")
                         .of_type([AccountOpened, MoneyDeposited, MoneyWithdrawn])
                         .to_a

    events.reduce(0) do |balance, event|
      case event
      when AccountOpened   then balance + event.data[:initial_deposit].to_f
      when MoneyDeposited  then balance + event.data[:amount].to_f
      when MoneyWithdrawn  then balance - event.data[:amount].to_f
      else balance
      end
    end
  end

  def transaction_history(account_id, limit: 10)
    @event_store.read
                .stream("account-#{account_id}")
                .of_type([MoneyDeposited, MoneyWithdrawn])
                .last(limit)
                .map do |event|
      {
        type:   event.class.name,
        amount: event.data[:amount],
        at:     event.metadata[:timestamp]
      }
    end
  end
end
```

### Read Model (Materialized View)

```ruby
# app/models/account_read_model.rb
class AccountReadModel < ApplicationRecord
  self.table_name = 'account_read_models'
  # columns: account_id, balance, owner_name, created_at, updated_at

  validates :account_id, presence: true
  validates :balance, numericality: { greater_than_or_equal_to: 0 }
end

# app/projections/account_read_model_builder.rb
class AccountReadModelBuilder
  def initialize(event_store)
    @event_store = event_store
  end

  # สร้าง read model จาก scratch
  def build_for(account_id)
    events = @event_store.read
                         .stream("account-#{account_id}")
                         .to_a

    record = AccountReadModel.find_or_initialize_by(account_id: account_id)

    events.each do |event|
      apply(record, event)
    end

    record.save!
    record
  end

  private

  def apply(record, event)
    case event
    when AccountOpened
      record.balance    = event.data[:initial_deposit].to_f
      record.owner_name = event.data[:owner_name]
    when MoneyDeposited
      record.balance += event.data[:amount].to_f
    when MoneyWithdrawn
      record.balance -= event.data[:amount].to_f
    end
  end
end
```

---

## ขั้นตอนที่ 1647: Aggregate Pattern

Aggregate คือ cluster ของ domain objects ที่ถูก treat as a unit เพื่อ data changes

### สร้าง Aggregate

```ruby
# app/aggregates/bank_account.rb
class BankAccount
  include AggregateRoot

  attr_reader :id, :balance, :owner_name, :status

  def initialize(id = SecureRandom.uuid)
    @id      = id
    @balance = 0
    @status  = :new
  end

  # Commands
  def open(owner_name:, initial_deposit:)
    raise "Account already opened" unless @status == :new
    raise "Initial deposit must be positive" unless initial_deposit > 0

    apply(AccountOpened.new(data: {
      account_id:      @id,
      owner_name:      owner_name,
      initial_deposit: initial_deposit
    }))
  end

  def deposit(amount:)
    raise "Account not open" unless @status == :open
    raise "Amount must be positive" unless amount > 0

    apply(MoneyDeposited.new(data: {
      account_id: @id,
      amount:     amount
    }))
  end

  def withdraw(amount:)
    raise "Account not open" unless @status == :open
    raise "Amount must be positive" unless amount > 0
    raise "Insufficient funds" if @balance < amount

    apply(MoneyWithdrawn.new(data: {
      account_id: @id,
      amount:     amount
    }))
  end

  def close
    raise "Account not open" unless @status == :open
    raise "Balance must be zero to close" unless @balance.zero?

    apply(AccountClosed.new(data: { account_id: @id }))
  end

  private

  # Event handlers (apply methods)
  on AccountOpened do |event|
    @status     = :open
    @balance    = event.data[:initial_deposit].to_f
    @owner_name = event.data[:owner_name]
  end

  on MoneyDeposited do |event|
    @balance += event.data[:amount].to_f
  end

  on MoneyWithdrawn do |event|
    @balance -= event.data[:amount].to_f
  end

  on AccountClosed do |event|
    @status = :closed
  end
end
```

### ใช้ Aggregate Repository

```ruby
# app/repositories/bank_account_repository.rb
class BankAccountRepository
  def initialize(event_store)
    @event_store = event_store
    @repository  = AggregateRoot::Repository.new(event_store)
  end

  def load(account_id)
    @repository.load(
      BankAccount.new(account_id),
      "account-#{account_id}"
    )
  end

  def save(account)
    @repository.store(account, "account-#{account.id}")
  end

  def with_account(account_id)
    account = load(account_id)
    yield account
    save(account)
  end
end

# การใช้งาน
class AccountService
  def initialize(event_store)
    @repo = BankAccountRepository.new(event_store)
  end

  def open_account(owner_name:, initial_deposit:)
    account = BankAccount.new
    account.open(owner_name: owner_name, initial_deposit: initial_deposit)
    @repo.save(account)
    account.id
  end

  def deposit(account_id:, amount:)
    @repo.with_account(account_id) do |account|
      account.deposit(amount: amount)
    end
  end

  def withdraw(account_id:, amount:)
    @repo.with_account(account_id) do |account|
      account.withdraw(amount: amount)
    end
  end
end
```

---

## ขั้นตอนที่ 1648: Sagas (Process Managers)

Sagas คือ long-running transactions ที่ coordinate หลาย aggregates

### ตัวอย่าง Order Saga

```ruby
# app/sagas/order_fulfillment_saga.rb
class OrderFulfillmentSaga
  include RailsEventStore::AsyncHandler  # หรือ sync

  # State ของ saga
  attr_reader :state

  def initialize
    @state = :new
  end

  # Saga ตอบสนองต่อ events
  def call(event)
    case event
    when OrderPlaced
      handle_order_placed(event)
    when PaymentConfirmed
      handle_payment_confirmed(event)
    when InventoryReserved
      handle_inventory_reserved(event)
    when ShipmentCreated
      handle_shipment_created(event)
    when PaymentFailed
      handle_payment_failed(event)
    when InventoryUnavailable
      handle_inventory_unavailable(event)
    end
  end

  private

  def handle_order_placed(event)
    @order_id = event.data[:order_id]
    @state    = :awaiting_payment

    # สั่งให้ payment service ทำงาน
    PaymentService.new.process_payment(
      order_id: @order_id,
      amount:   event.data[:total]
    )
  end

  def handle_payment_confirmed(event)
    return unless event.data[:order_id] == @order_id

    @state = :awaiting_inventory

    # สั่งให้ inventory service reserve items
    InventoryService.new.reserve_items(
      order_id: @order_id,
      items:    fetch_order_items(@order_id)
    )
  end

  def handle_inventory_reserved(event)
    return unless event.data[:order_id] == @order_id

    @state = :awaiting_shipment

    # สั่งให้ shipping service สร้าง shipment
    ShippingService.new.create_shipment(
      order_id: @order_id,
      address:  fetch_delivery_address(@order_id)
    )
  end

  def handle_shipment_created(event)
    return unless event.data[:order_id] == @order_id

    @state = :completed

    # แจ้ง customer
    CustomerNotificationService.new.notify_shipped(order_id: @order_id)
  end

  def handle_payment_failed(event)
    return unless event.data[:order_id] == @order_id

    @state = :failed
    # Cancel order, notify customer
    OrderService.new.cancel(order_id: @order_id, reason: 'payment_failed')
    CustomerNotificationService.new.notify_payment_failed(order_id: @order_id)
  end

  def handle_inventory_unavailable(event)
    return unless event.data[:order_id] == @order_id

    @state = :failed
    # Refund payment, cancel order
    PaymentService.new.refund(order_id: @order_id)
    OrderService.new.cancel(order_id: @order_id, reason: 'inventory_unavailable')
  end
end
```

### Persistent Saga

```ruby
# app/sagas/persistent_order_saga.rb
class PersistentOrderSaga < ApplicationRecord
  self.table_name = 'order_sagas'

  serialize :state_data, JSON

  STATES = %w[new awaiting_payment awaiting_inventory awaiting_shipment completed failed].freeze

  validates :order_id, presence: true
  validates :current_state, inclusion: { in: STATES }

  def self.handle(event)
    saga = case event
           when OrderPlaced
             create!(
               order_id:      event.data[:order_id],
               current_state: 'new',
               state_data:    {}
             )
           else
             find_by!(order_id: event.data[:order_id])
           end

    saga.process(event)
  end

  def process(event)
    case event
    when OrderPlaced          then transition_to_awaiting_payment(event)
    when PaymentConfirmed     then transition_to_awaiting_inventory(event)
    when InventoryReserved    then transition_to_awaiting_shipment(event)
    when ShipmentCreated      then transition_to_completed(event)
    when PaymentFailed        then compensate_payment_failure(event)
    when InventoryUnavailable then compensate_inventory_unavailable(event)
    end

    save!
  end

  private

  def transition_to_awaiting_payment(event)
    update!(
      current_state: 'awaiting_payment',
      state_data:    state_data.merge('total' => event.data[:total])
    )
    PaymentJob.perform_later(order_id: order_id, amount: event.data[:total])
  end

  # ... other transitions
end
```

---

## ขั้นตอนที่ 1649: Event Store Queries

```ruby
# อ่าน events ทั้งหมดจาก stream
events = event_store.read
                    .stream("account-123")
                    .to_a

# อ่าน events ตาม type
deposits = event_store.read
                      .stream("account-123")
                      .of_type([MoneyDeposited])
                      .to_a

# อ่าน events ใน time range
recent = event_store.read
                    .stream("account-123")
                    .newer_than(1.week.ago)
                    .to_a

# อ่าน events แบบ paginated
page1 = event_store.read
                   .stream("account-123")
                   .forward
                   .limit(10)
                   .to_a

page2 = event_store.read
                   .stream("account-123")
                   .forward
                   .from(page1.last.id)
                   .limit(10)
                   .to_a

# อ่าน events จากทุก streams (global stream)
all_events = event_store.read.to_a

# Query แบบ complex
event_store.read
           .of_type([OrderPlaced, OrderCancelled])
           .newer_than(24.hours.ago)
           .each do |event|
  puts "#{event.class}: #{event.data[:order_id]}"
end
```

---

## ขั้นตอนที่ 1650: Snapshots

Snapshots ช่วยเพิ่ม performance เมื่อ aggregate มี events จำนวนมาก

```ruby
# app/aggregates/bank_account_with_snapshot.rb
class BankAccountWithSnapshot
  include AggregateRoot

  SNAPSHOT_THRESHOLD = 50  # snapshot ทุก 50 events

  attr_reader :id, :balance, :version

  def initialize(id)
    @id      = id
    @balance = 0
    @version = 0
  end

  def self.build_from_snapshot(snapshot_data, events_since_snapshot)
    account = new(snapshot_data[:id])
    account.restore_from_snapshot(snapshot_data)

    events_since_snapshot.each do |event|
      account.send(:apply_event, event)
    end

    account
  end

  def to_snapshot
    {
      id:      @id,
      balance: @balance,
      version: @version,
      taken_at: Time.current
    }
  end

  def restore_from_snapshot(data)
    @balance = data[:balance]
    @version = data[:version]
  end

  def needs_snapshot?
    @version % SNAPSHOT_THRESHOLD == 0
  end

  private

  on MoneyDeposited do |event|
    @balance += event.data[:amount].to_f
    @version += 1
  end

  on MoneyWithdrawn do |event|
    @balance -= event.data[:amount].to_f
    @version += 1
  end
end

# Repository with snapshot support
class SnapshotRepository
  def load(account_id)
    snapshot = SnapshotStore.latest_for(account_id)

    if snapshot
      events_since = event_store.read
                                .stream("account-#{account_id}")
                                .after(snapshot[:event_id])
                                .to_a

      BankAccountWithSnapshot.build_from_snapshot(snapshot[:data], events_since)
    else
      # Load from scratch
      account = BankAccountWithSnapshot.new(account_id)
      events  = event_store.read.stream("account-#{account_id}").to_a
      events.each { |e| account.send(:apply_event, e) }
      account
    end
  end

  def save(account)
    # Save events
    repository.store(account, "account-#{account.id}")

    # Save snapshot if needed
    if account.needs_snapshot?
      last_event_id = event_store.read
                                  .stream("account-#{account.id}")
                                  .last
                                  &.id
      SnapshotStore.save(
        aggregate_id: account.id,
        event_id:     last_event_id,
        data:         account.to_snapshot
      )
    end
  end
end
```

---

## ขั้นตอนที่ 1651: Event Versioning

การจัดการ schema ของ events เมื่อ business requirements เปลี่ยน

```ruby
# Event version 1 (เก่า)
class MoneyDeposited_V1 < RailsEventStore::Event
  # data: { account_id:, amount: }
end

# Event version 2 (ใหม่ - เพิ่ม fields)
class MoneyDeposited < RailsEventStore::Event
  # data: { account_id:, amount:, currency:, reference: }
end

# Upcaster - แปลง V1 เป็น V2
class MoneyDepositedUpcaster
  def call(event)
    return event unless event.is_a?(MoneyDeposited_V1)

    MoneyDeposited.new(
      data: event.data.merge(
        currency:  'THB',  # default
        reference: nil
      ),
      metadata: event.metadata
    )
  end
end

# Register upcaster
event_store = RailsEventStore::Client.new(
  mapper: RubyEventStore::Mappers::Pipeline.new(
    transformations: [
      MoneyDepositedUpcaster.new
    ]
  )
)
```

---

## ขั้นตอนที่ 1652: Testing Event Sourcing

```ruby
# spec/aggregates/bank_account_spec.rb
RSpec.describe BankAccount do
  let(:account_id) { SecureRandom.uuid }
  let(:account)    { BankAccount.new(account_id) }

  describe '#open' do
    it 'applies AccountOpened event' do
      account.open(owner_name: 'John Doe', initial_deposit: 1000)

      expect(account.unpublished_events).to include(
        an_object_having_attributes(
          class: AccountOpened,
          data:  hash_including(
            owner_name:      'John Doe',
            initial_deposit: 1000
          )
        )
      )
    end

    it 'sets balance to initial deposit' do
      account.open(owner_name: 'John Doe', initial_deposit: 1000)
      expect(account.balance).to eq(1000)
    end

    it 'raises error when already opened' do
      account.open(owner_name: 'John Doe', initial_deposit: 1000)

      expect {
        account.open(owner_name: 'Jane Doe', initial_deposit: 500)
      }.to raise_error("Account already opened")
    end
  end

  describe '#deposit' do
    before { account.open(owner_name: 'John', initial_deposit: 1000) }

    it 'increases balance' do
      account.deposit(amount: 500)
      expect(account.balance).to eq(1500)
    end

    it 'raises error for non-positive amount' do
      expect { account.deposit(amount: 0) }.to raise_error("Amount must be positive")
      expect { account.deposit(amount: -100) }.to raise_error("Amount must be positive")
    end
  end

  describe '#withdraw' do
    before do
      account.open(owner_name: 'John', initial_deposit: 1000)
    end

    it 'decreases balance' do
      account.withdraw(amount: 300)
      expect(account.balance).to eq(700)
    end

    it 'raises error for insufficient funds' do
      expect {
        account.withdraw(amount: 1001)
      }.to raise_error("Insufficient funds")
    end
  end
end

# spec/event_handlers/update_account_balance_spec.rb
RSpec.describe UpdateAccountBalance do
  let(:handler) { described_class.new }

  it 'increases balance on MoneyDeposited' do
    event = MoneyDeposited.new(data: { account_id: 'acc-1', amount: 500 })

    expect {
      handler.call(event)
    }.to change {
      AccountReadModel.find_by(account_id: 'acc-1')&.balance
    }.from(nil).to(500)
  end
end
```

---

## ขั้นตอนที่ 1653: CQRS Command Bus

```ruby
# app/commands/deposit_money_command.rb
DepositMoney = Struct.new(:account_id, :amount, keyword_init: true) do
  def valid?
    account_id.present? && amount.to_f.positive?
  end
end

# app/command_handlers/deposit_money_handler.rb
class DepositMoneyHandler
  def initialize(event_store)
    @repository = BankAccountRepository.new(event_store)
  end

  def call(command)
    raise ArgumentError, "Invalid command" unless command.valid?

    @repository.with_account(command.account_id) do |account|
      account.deposit(amount: command.amount)
    end
  end
end

# app/command_bus.rb
class CommandBus
  def initialize
    @handlers = {}
  end

  def register(command_class, handler)
    @handlers[command_class] = handler
  end

  def dispatch(command)
    handler = @handlers[command.class]
    raise "No handler for #{command.class}" unless handler

    handler.call(command)
  end
end

# config/initializers/command_bus.rb
Rails.configuration.command_bus = CommandBus.new.tap do |bus|
  event_store = Rails.configuration.event_store

  bus.register(DepositMoney,  DepositMoneyHandler.new(event_store))
  bus.register(WithdrawMoney, WithdrawMoneyHandler.new(event_store))
  bus.register(OpenAccount,   OpenAccountHandler.new(event_store))
  bus.register(CloseAccount,  CloseAccountHandler.new(event_store))
end

# การใช้งานใน Controller
class AccountsController < ApplicationController
  def deposit
    command = DepositMoney.new(
      account_id: params[:account_id],
      amount:     params[:amount].to_f
    )

    Rails.configuration.command_bus.dispatch(command)

    render json: { status: 'success' }
  rescue ArgumentError => e
    render json: { error: e.message }, status: :unprocessable_entity
  end
end
```

---

## ขั้นตอนที่ 1654: Read Model Rebuilding

```ruby
# app/tasks/rebuild_read_models.rake
namespace :event_store do
  desc "Rebuild all read models from event store"
  task rebuild_read_models: :environment do
    puts "Starting read model rebuild..."

    AccountReadModel.delete_all
    puts "Cleared account read models"

    event_store = Rails.configuration.event_store

    # หา streams ทั้งหมด
    streams = event_store.read
                         .streams_of([AccountOpened])
                         .map(&:name)

    streams.each do |stream_name|
      account_id = stream_name.gsub('account-', '')
      builder    = AccountReadModelBuilder.new(event_store)
      builder.build_for(account_id)
      print '.'
    end

    puts "\nDone! Rebuilt #{streams.count} account read models."
  end

  desc "Rebuild read model for specific account"
  task :rebuild_account, [:account_id] => :environment do |_, args|
    account_id = args[:account_id]
    raise "account_id required" unless account_id

    AccountReadModel.where(account_id: account_id).delete_all

    builder = AccountReadModelBuilder.new(Rails.configuration.event_store)
    model   = builder.build_for(account_id)

    puts "Rebuilt read model for account #{account_id}: balance=#{model.balance}"
  end
end
```

---

## ขั้นตอนที่ 1655: Event Store Schema และ Migration

```ruby
# db/migrate/20240101000001_create_event_store_tables.rb
class CreateEventStoreTables < ActiveRecord::Migration[7.1]
  def change
    create_table :event_store_events, id: :uuid do |t|
      t.string  :event_type,    null: false
      t.binary  :metadata,      null: true
      t.binary  :data,          null: false
      t.datetime :created_at,   null: false
      t.datetime :valid_at,     null: true
    end

    add_index :event_store_events, :event_type
    add_index :event_store_events, :created_at
    add_index :event_store_events, :valid_at

    create_table :event_store_events_in_streams, id: :bigint do |t|
      t.string  :stream,      null: false
      t.integer :position,    null: true
      t.references :event, null: false, type: :uuid,
                   foreign_key: { to_table: :event_store_events },
                   index: false
      t.datetime :created_at, null: false
    end

    add_index :event_store_events_in_streams,
              [:stream, :position],
              unique: true,
              name:   'index_es_streams_on_stream_and_position'

    add_index :event_store_events_in_streams,
              [:stream, :event_id],
              unique: true,
              name:   'index_es_streams_on_stream_and_event_id'
  end
end

# db/migrate/20240101000002_create_account_read_models.rb
class CreateAccountReadModels < ActiveRecord::Migration[7.1]
  def change
    create_table :account_read_models do |t|
      t.string  :account_id,  null: false, index: { unique: true }
      t.decimal :balance,     null: false, default: 0, precision: 15, scale: 2
      t.string  :owner_name,  null: false
      t.string  :status,      null: false, default: 'open'
      t.timestamps
    end
  end
end
```

---

## ขั้นตอนที่ 1656: ตัวอย่าง E-commerce Order System

```ruby
# Events
class OrderCreated      < RailsEventStore::Event; end
class ItemAddedToOrder  < RailsEventStore::Event; end
class ItemRemovedFromOrder < RailsEventStore::Event; end
class OrderConfirmed    < RailsEventStore::Event; end
class PaymentReceived   < RailsEventStore::Event; end
class OrderShipped      < RailsEventStore::Event; end
class OrderDelivered    < RailsEventStore::Event; end
class OrderCancelled    < RailsEventStore::Event; end
class RefundIssued      < RailsEventStore::Event; end

# Aggregate
class Order
  include AggregateRoot

  attr_reader :id, :items, :status, :total, :customer_id

  def initialize(id)
    @id        = id
    @items     = []
    @status    = :new
    @total     = 0
    @customer_id = nil
  end

  def create(customer_id:)
    apply(OrderCreated.new(data: {
      order_id:    @id,
      customer_id: customer_id,
      created_at:  Time.current
    }))
  end

  def add_item(product_id:, name:, price:, quantity:)
    raise "Cannot modify confirmed order" if [:confirmed, :shipped].include?(@status)

    apply(ItemAddedToOrder.new(data: {
      order_id:   @id,
      product_id: product_id,
      name:       name,
      price:      price.to_f,
      quantity:   quantity.to_i
    }))
  end

  def confirm
    raise "Order is empty" if @items.empty?
    raise "Order already confirmed" if @status != :created

    apply(OrderConfirmed.new(data: {
      order_id: @id,
      total:    @total
    }))
  end

  def receive_payment(amount:, payment_id:)
    raise "Order not confirmed" unless @status == :confirmed
    raise "Wrong payment amount" unless amount == @total

    apply(PaymentReceived.new(data: {
      order_id:   @id,
      amount:     amount,
      payment_id: payment_id
    }))
  end

  def ship(tracking_number:, carrier:)
    raise "Payment not received" unless @status == :paid

    apply(OrderShipped.new(data: {
      order_id:        @id,
      tracking_number: tracking_number,
      carrier:         carrier,
      shipped_at:      Time.current
    }))
  end

  def cancel(reason:)
    raise "Cannot cancel shipped order" if @status == :shipped

    apply(OrderCancelled.new(data: {
      order_id:     @id,
      reason:       reason,
      cancelled_at: Time.current
    }))
  end

  private

  on OrderCreated do |event|
    @status      = :created
    @customer_id = event.data[:customer_id]
  end

  on ItemAddedToOrder do |event|
    item = { product_id: event.data[:product_id],
             name:       event.data[:name],
             price:      event.data[:price],
             quantity:   event.data[:quantity] }
    existing = @items.find { |i| i[:product_id] == item[:product_id] }

    if existing
      existing[:quantity] += item[:quantity]
    else
      @items << item
    end

    recalculate_total
  end

  on OrderConfirmed do |event|
    @status = :confirmed
  end

  on PaymentReceived do |event|
    @status = :paid
  end

  on OrderShipped do |event|
    @status = :shipped
  end

  on OrderCancelled do |event|
    @status = :cancelled
  end

  def recalculate_total
    @total = @items.sum { |i| i[:price] * i[:quantity] }
  end
end
```

---

## ขั้นตอนที่ 1657: การทำ Event Sourcing กับ Rails Concerns

```ruby
# app/concerns/event_sourceable.rb
module EventSourceable
  extend ActiveSupport::Concern

  included do
    after_create  :publish_created_event
    after_update  :publish_updated_event
    after_destroy :publish_destroyed_event
  end

  private

  def publish_created_event
    event_class_name = "#{self.class.name}Created"
    return unless Object.const_defined?(event_class_name)

    event_class = Object.const_get(event_class_name)
    Rails.configuration.event_store.publish(
      event_class.new(data: event_data),
      stream_name: "#{self.class.name.downcase}-#{id}"
    )
  end

  def publish_updated_event
    event_class_name = "#{self.class.name}Updated"
    return unless Object.const_defined?(event_class_name)

    event_class = Object.const_get(event_class_name)
    Rails.configuration.event_store.publish(
      event_class.new(data: event_data.merge(changes: previous_changes)),
      stream_name: "#{self.class.name.downcase}-#{id}"
    )
  end

  def publish_destroyed_event
    event_class_name = "#{self.class.name}Destroyed"
    return unless Object.const_defined?(event_class_name)

    event_class = Object.const_get(event_class_name)
    Rails.configuration.event_store.publish(
      event_class.new(data: { id: id }),
      stream_name: "#{self.class.name.downcase}-#{id}"
    )
  end

  def event_data
    attributes.symbolize_keys
  end
end

# ใช้ใน model
class Product < ApplicationRecord
  include EventSourceable
  # สร้าง ProductCreated, ProductUpdated, ProductDestroyed events อัตโนมัติ
end
```

---

## ขั้นตอนที่ 1658: Event Sourcing กับ Multi-tenancy

```ruby
# Multi-tenant event store
class TenantAwareEventStore
  def initialize(event_store)
    @event_store = event_store
  end

  def publish(event, stream_name:)
    tenant_id = Current.tenant_id
    raise "No tenant context" unless tenant_id

    @event_store.publish(
      event,
      stream_name: "#{tenant_id}-#{stream_name}"
    )
  end

  def read_for_tenant(tenant_id: Current.tenant_id)
    @event_store.read
                .tap { |r| r.stream_prefix("#{tenant_id}-") }
  end
end

# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :tenant_id
  attribute :user

  def tenant
    @tenant ||= Tenant.find(tenant_id)
  end
end

# Middleware ที่ set tenant
class TenantMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    request = ActionDispatch::Request.new(env)
    tenant  = Tenant.find_by(subdomain: request.subdomain)

    if tenant
      Current.tenant_id = tenant.id
      @app.call(env)
    else
      [404, {}, ['Tenant not found']]
    end
  ensure
    Current.tenant_id = nil
  end
end
```

---

## ขั้นตอนที่ 1659: Monitoring และ Observability

```ruby
# app/event_handlers/event_logger.rb
class EventLogger
  def call(event)
    Rails.logger.info({
      event_type:  event.class.name,
      event_id:    event.event_id,
      stream:      event.metadata[:stream],
      timestamp:   event.metadata[:timestamp],
      data:        event.data.except(:sensitive_field)
    }.to_json)
  end
end

# app/event_handlers/event_metrics.rb
class EventMetrics
  def call(event)
    # Prometheus metrics
    EventCounter.increment(
      labels: { event_type: event.class.name }
    )

    # StatsD
    StatsD.increment("events.#{event.class.name.underscore}")
  end
end

# Subscribe ทุก events
event_store.subscribe_to_all_events(EventLogger.new)
event_store.subscribe_to_all_events(EventMetrics.new)

# Dashboard สำหรับ events
class EventsDashboard
  def self.summary(since: 24.hours.ago)
    events = Rails.configuration.event_store
                  .read
                  .newer_than(since)
                  .to_a

    {
      total:         events.count,
      by_type:       events.group_by(&:class).transform_values(&:count),
      recent:        events.last(10).map { |e| {
                       type: e.class.name,
                       at:   e.metadata[:timestamp]
                     }}
    }
  end
end
```

---

## ขั้นตอนที่ 1660: แนวปฏิบัติที่ดีสำหรับ Event Sourcing

### DOs ✅

```ruby
# 1. Events ควรเป็น past tense (สิ่งที่เกิดขึ้นแล้ว)
class UserRegistered < RailsEventStore::Event; end  # ✅
class RegisterUser   < RailsEventStore::Event; end  # ❌ (นี่คือ Command)

# 2. Events ควร immutable
class MoneyDeposited < RailsEventStore::Event
  def initialize(...)
    super
    freeze  # ✅ ป้องกันการเปลี่ยนแปลง
  end
end

# 3. Events ควรมีข้อมูลครบ (self-contained)
MoneyDeposited.new(data: {
  account_id:   'acc-123',
  amount:       1000,
  currency:     'THB',
  deposited_at: Time.current,
  reference:    'TXN-456'  # ✅ มีทุกอย่างที่จำเป็น
})

# 4. ใช้ UUID สำหรับ aggregate IDs
account_id = SecureRandom.uuid  # ✅ ไม่ต้องรอ database

# 5. แยก read model จาก event store
class AccountBalance < ApplicationRecord
  # Read model - optimized for queries
  scope :high_balance, -> { where('balance > ?', 100_000) }
  scope :overdrawn,    -> { where('balance < 0') }
end
```

### DON'Ts ❌

```ruby
# 1. อย่าเปลี่ยน events ที่ publish แล้ว
# Events คือ history ที่เปลี่ยนไม่ได้

# 2. อย่าใส่ logic ใน event handlers มากเกินไป
class BadHandler
  def call(event)
    # ❌ too much logic here
    user    = User.find(event.data[:user_id])
    profile = user.build_profile(...)
    profile.save!
    MailChimp.subscribe(user.email)
    Slack.notify("#signups", "New user: #{user.email}")
    Analytics.track('user_registered', user.id)
    # ...too long
  end
end

# ✅ แยก handlers
store.subscribe(CreateUserProfile, to: [UserRegistered])
store.subscribe(SubscribeToMailchimp, to: [UserRegistered])
store.subscribe(NotifySlack, to: [UserRegistered])

# 3. อย่า query event store สำหรับ read operations ที่ frequent
# ใช้ read models แทน
def account_balance(id)
  # ❌ ช้า ถ้า events มีเยอะ
  event_store.read.stream("account-#{id}").to_a.reduce(0) { ... }

  # ✅ เร็ว
  AccountReadModel.find_by!(account_id: id).balance
end
```

---

## แบบฝึกหัด: Event Sourcing และ CQRS

### ข้อที่ 1: Event Design
ออกแบบ events สำหรับระบบ Library (ยืม/คืนหนังสือ) อย่างน้อย 6 events พร้อม data fields

```ruby
# ตัวอย่างคำตอบ
class BookRegistered    < RailsEventStore::Event; end
# data: { book_id:, title:, isbn:, author:, copies: }

class BookBorrowed      < RailsEventStore::Event; end
# data: { book_id:, member_id:, due_date:, borrowed_at: }

class BookReturned      < RailsEventStore::Event; end
# data: { book_id:, member_id:, returned_at:, condition: }

class BookDamaged       < RailsEventStore::Event; end
# data: { book_id:, description:, fine_amount: }

class MemberRegistered  < RailsEventStore::Event; end
# data: { member_id:, name:, email:, joined_at: }

class FineIssued        < RailsEventStore::Event; end
# data: { member_id:, book_id:, amount:, reason:, issued_at: }
```

### ข้อที่ 2: สร้าง Aggregate สำหรับ Book

```ruby
# แบบฝึกหัด: implement Book aggregate
class Book
  include AggregateRoot

  attr_reader :id, :title, :available_copies, :status

  def initialize(id)
    @id               = id
    @available_copies = 0
    @status           = :new
  end

  def register(title:, isbn:, author:, copies:)
    raise "Already registered" unless @status == :new
    raise "Must have at least 1 copy" unless copies > 0

    apply(BookRegistered.new(data: {
      book_id: @id, title: title, isbn: isbn,
      author: author, copies: copies
    }))
  end

  def borrow(member_id:)
    raise "No copies available" unless @available_copies > 0

    apply(BookBorrowed.new(data: {
      book_id:   @id,
      member_id: member_id,
      due_date:  2.weeks.from_now,
      borrowed_at: Time.current
    }))
  end

  def return_book(member_id:)
    apply(BookReturned.new(data: {
      book_id:     @id,
      member_id:   member_id,
      returned_at: Time.current,
      condition:   'good'
    }))
  end

  private

  on BookRegistered do |event|
    @status           = :available
    @title            = event.data[:title]
    @available_copies = event.data[:copies]
  end

  on BookBorrowed do |event|
    @available_copies -= 1
  end

  on BookReturned do |event|
    @available_copies += 1
  end
end
```

### ข้อที่ 3: Projection สำหรับ Library

```ruby
# สร้าง projection ที่แสดง:
# - หนังสือที่ถูกยืมมากที่สุด 10 เล่ม
# - สมาชิกที่ค้างคืน
# - จำนวนหนังสือที่ available

class LibraryProjection
  def initialize(event_store)
    @event_store = event_store
  end

  def most_borrowed_books(limit: 10)
    @event_store.read
                .of_type([BookBorrowed])
                .to_a
                .group_by { |e| e.data[:book_id] }
                .transform_values(&:count)
                .sort_by { |_, count| -count }
                .first(limit)
                .map { |book_id, count|
                  { book_id: book_id, borrow_count: count }
                }
  end

  def overdue_members
    borrowed_events = @event_store.read.of_type([BookBorrowed]).to_a
    returned_events = @event_store.read.of_type([BookReturned]).to_a

    borrowed_events.select { |e| e.data[:due_date] < Time.current }
                   .reject { |borrow|
                     returned_events.any? { |ret|
                       ret.data[:book_id]   == borrow.data[:book_id] &&
                       ret.data[:member_id] == borrow.data[:member_id] &&
                       ret.metadata[:timestamp] > borrow.metadata[:timestamp]
                     }
                   }
                   .map { |e| e.data[:member_id] }
                   .uniq
  end
end
```

### ข้อที่ 4: Command สำหรับ Library

```ruby
# สร้าง Commands และ Handlers
BorrowBook  = Struct.new(:book_id, :member_id, keyword_init: true)
ReturnBook  = Struct.new(:book_id, :member_id, keyword_init: true)
RegisterBook = Struct.new(:title, :isbn, :author, :copies, keyword_init: true)

class BorrowBookHandler
  def initialize(event_store)
    @repo = BookRepository.new(event_store)
  end

  def call(command)
    @repo.with_book(command.book_id) do |book|
      book.borrow(member_id: command.member_id)
    end
  end
end

class ReturnBookHandler
  def initialize(event_store)
    @repo = BookRepository.new(event_store)
  end

  def call(command)
    @repo.with_book(command.book_id) do |book|
      book.return_book(member_id: command.member_id)
    end
  end
end
```

### ข้อที่ 5-20: แบบฝึกหัดเพิ่มเติม

**ข้อ 5:** สร้าง Saga สำหรับ overdue book notification (ส่ง email เมื่อหนังสือค้างคืน)

**ข้อ 6:** เพิ่ม Snapshot สำหรับ Book aggregate หลังจาก 100 events

**ข้อ 7:** สร้าง Read Model `BookAvailabilityReadModel` ที่อัพเดทจาก events

**ข้อ 8:** เขียน RSpec tests สำหรับ Book aggregate ครอบคลุมทุก scenarios

**ข้อ 9:** ทำ Event Versioning สำหรับ `BookBorrowed` event (เพิ่ม `loan_period` field)

**ข้อ 10:** สร้าง Command Bus ที่มี middleware สำหรับ logging และ validation

**ข้อ 11:** Implement `MemberAggregate` พร้อม events: registered, suspended, banned, reactivated

**ข้อ 12:** สร้าง projection สำหรับ fine calculation (คำนวณค่าปรับจาก events)

**ข้อ 13:** เพิ่ม multi-tenancy ให้ library system (หลายสาขา)

**ข้อ 14:** สร้าง rake task สำหรับ rebuild read models ทั้งหมดของ library

**ข้อ 15:** เพิ่ม event metadata เช่น user_agent, ip_address, user_id ทุก events

**ข้อ 16:** สร้าง dashboard แสดง real-time events โดยใช้ Action Cable

**ข้อ 17:** ทำ event correlation (link events ที่เกี่ยวข้องกัน ด้วย correlation_id)

**ข้อ 18:** เพิ่ม idempotency key ป้องกัน duplicate commands

```ruby
class IdempotentCommandBus < CommandBus
  def dispatch(command, idempotency_key: nil)
    if idempotency_key
      return if ProcessedCommand.exists?(key: idempotency_key)

      result = super(command)
      ProcessedCommand.create!(key: idempotency_key)
      result
    else
      super(command)
    end
  end
end
```

**ข้อ 19:** เขียน integration test ที่ test flow ทั้งหมด: register book → borrow → return

**ข้อ 20:** Deploy event sourcing application พร้อม monitoring ด้วย Prometheus

```ruby
# EventMetrics สำหรับ Prometheus
class PrometheusEventMetrics
  EVENT_COUNTER = Prometheus::Client::Counter.new(
    :event_store_events_total,
    docstring: 'Total events published',
    labels:    [:event_type]
  )

  PROCESSING_TIME = Prometheus::Client::Histogram.new(
    :event_handler_duration_seconds,
    docstring: 'Event handler processing time',
    labels:    [:event_type, :handler]
  )

  def call(event)
    EVENT_COUNTER.increment(labels: { event_type: event.class.name })
  end
end
```

---

## สรุป

Event Sourcing และ CQRS เป็น patterns ที่ทรงพลังสำหรับ:
- Systems ที่ต้องการ audit trail สมบูรณ์
- Business domains ที่ซับซ้อน
- Systems ที่ต้องการ scale read/write แยกกัน
- Applications ที่ต้องการ time-travel debugging

**RailsEventStore** ทำให้ implementation ง่ายขึ้นมากสำหรับ Rails applications

ขั้นตอนต่อไป: Part 77 - Domain-Driven Design in Rails
