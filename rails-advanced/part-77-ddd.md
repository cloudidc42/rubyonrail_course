# Part 77: Domain-Driven Design (DDD) ใน Ruby on Rails

## ขั้นตอนที่ 1661-1680

---

## ขั้นตอนที่ 1661: DDD คืออะไร?

Domain-Driven Design (DDD) คือแนวทางการออกแบบซอฟต์แวร์ที่เน้น **business domain** เป็นศูนย์กลาง โดย Eric Evans เสนอในหนังสือ "Domain-Driven Design" ปี 2003

### Strategic Design vs Tactical Design

**Strategic Design** = การวางแผนระดับสูง
- Bounded Contexts
- Context Mapping
- Ubiquitous Language

**Tactical Design** = การ implement รายละเอียด
- Entities
- Value Objects
- Aggregates
- Domain Events
- Repositories
- Domain Services
- Application Services

### ปัญหาที่ DDD แก้ไข

```ruby
# ❌ Anemic Domain Model - ทุกอย่างเป็นแค่ data container
class Order < ApplicationRecord
  belongs_to :customer
  has_many :line_items

  # ไม่มี business logic ใน model
  # logic กระจายอยู่ใน controllers, services
end

class OrderService
  def calculate_total(order)
    order.line_items.sum { |item| item.price * item.quantity }
  end

  def apply_discount(order, code)
    # ... logic ที่ควรอยู่ใน Order model
  end

  def can_ship?(order)
    # ... logic ที่ควรอยู่ใน Order model
  end
end

# ✅ Rich Domain Model - business logic อยู่ใน domain
class Order
  def total
    line_items.sum(&:subtotal)
  end

  def apply_discount(coupon)
    raise OrderError, "Already discounted" if discounted?
    raise OrderError, "Invalid coupon" unless coupon.valid_for?(self)

    @discount = coupon.discount_for(self)
  end

  def ready_to_ship?
    paid? && items_in_stock? && address_valid?
  end
end
```

---

## ขั้นตอนที่ 1662: Ubiquitous Language

Ubiquitous Language คือภาษากลางที่ developers และ domain experts ใช้ร่วมกัน

### หลักการ

```ruby
# ❌ Technical language ไม่ตรงกับ business
class UserRecord
  def deactivate_flag_to_true
    update!(is_active: false)
  end
end

# ✅ Ubiquitous language
class Customer
  def suspend(reason:)
    raise "Customer already suspended" if suspended?

    @status         = :suspended
    @suspended_at   = Time.current
    @suspension_reason = reason

    domain_events << CustomerSuspended.new(
      customer_id: id,
      reason:      reason,
      suspended_at: Time.current
    )
  end

  def reinstate
    raise "Customer not suspended" unless suspended?

    @status = :active
    domain_events << CustomerReinstated.new(customer_id: id)
  end
end
```

### Glossary

```ruby
# lib/domain/glossary.rb
# เก็บ terminology ของ domain

module BankingDomain
  # Account: บัญชีธนาคารของลูกค้า
  # Balance: ยอดเงินคงเหลือในบัญชี ณ เวลาปัจจุบัน
  # Deposit: การฝากเงินเข้าบัญชี (เพิ่ม balance)
  # Withdrawal: การถอนเงินออกจากบัญชี (ลด balance)
  # Overdraft: การถอนเงินเกินกว่า balance (บางบัญชีอนุญาต)
  # Freeze: การระงับบัญชีชั่วคราว (ไม่สามารถทำ transactions)
  # Close: การปิดบัญชีถาวร (ต้อง balance = 0)
end
```

---

## ขั้นตอนที่ 1663: Bounded Contexts

Bounded Context คือ boundary ที่ชัดเจนว่า domain model ใดใช้ได้ที่ไหน

### ตัวอย่าง E-commerce

```
E-commerce System
├── Sales Context
│   - Order, OrderItem, Customer, Price, Discount
│
├── Inventory Context
│   - Product, Stock, Warehouse, ReorderPoint
│
├── Shipping Context
│   - Shipment, Package, Address, Carrier, TrackingNumber
│
├── Payment Context
│   - Payment, Invoice, Refund, PaymentMethod
│
└── Customer Context
    - Customer, Account, Preferences, LoyaltyPoints
```

### ข้อสังเกต: "Customer" ในแต่ละ context แตกต่างกัน

```ruby
# Sales::Customer - สิ่งที่ Sales care
module Sales
  class Customer
    attr_reader :id, :name, :email, :credit_limit, :loyalty_tier

    def can_place_order?(order_total)
      within_credit_limit?(order_total) && account_active?
    end
  end
end

# Shipping::Customer - สิ่งที่ Shipping care
module Shipping
  class Customer
    attr_reader :id, :name, :default_address, :delivery_preferences

    def preferred_carrier
      # logic เกี่ยวกับการจัดส่ง
    end
  end
end

# Payment::Customer - สิ่งที่ Payment care
module Payment
  class Customer
    attr_reader :id, :payment_methods, :billing_address, :tax_info

    def default_payment_method
      payment_methods.find(&:default?)
    end
  end
end
```

### Context Mapping

```ruby
# lib/context_map.rb
# Sales Context -> Payment Context: Conformist
# Sales Context -> Inventory Context: Partnership
# Shipping Context -> Sales Context: Customer/Supplier
# Payment Context เป็น separate bounded context

# Anti-Corruption Layer ระหว่าง contexts
module Sales
  class PaymentService
    def initialize(payment_gateway)
      @gateway = payment_gateway
    end

    def process_payment_for_order(order)
      # แปลง Sales::Order เป็น Payment::PaymentRequest
      payment_request = to_payment_request(order)
      result          = @gateway.process(payment_request)

      # แปลง Payment::Result กลับมาเป็น Sales terms
      from_payment_result(result)
    end

    private

    def to_payment_request(order)
      Payment::PaymentRequest.new(
        amount:     order.total_amount_cents,
        currency:   order.currency,
        customer_id: order.customer.id,
        reference:  "order-#{order.id}"
      )
    end

    def from_payment_result(result)
      case result.status
      when :authorized then :payment_received
      when :declined   then :payment_failed
      when :error      then :payment_error
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1664: Entities

Entity คือ object ที่มี **identity** ที่ไม่เปลี่ยนแปลง แม้ attributes จะเปลี่ยน

```ruby
# app/domain/entities/customer.rb
class Customer
  attr_reader :id, :name, :email, :status

  def initialize(id:, name:, email:, status: :active)
    @id     = id
    @name   = name
    @email  = email
    @status = status

    validate!
  end

  def change_email(new_email)
    raise DomainError, "Invalid email" unless valid_email?(new_email)
    raise DomainError, "Email already taken" if email_taken?(new_email)

    @email = new_email
    record_event(CustomerEmailChanged.new(customer_id: @id, new_email: new_email))
  end

  def suspend(reason:)
    raise DomainError, "Already suspended" if suspended?

    @status = :suspended
    record_event(CustomerSuspended.new(customer_id: @id, reason: reason))
  end

  def active?
    @status == :active
  end

  def suspended?
    @status == :suspended
  end

  # Equality ใช้ identity (id)
  def ==(other)
    return false unless other.is_a?(Customer)
    id == other.id
  end

  alias_method :eql?, :==

  def hash
    id.hash
  end

  private

  def validate!
    raise DomainError, "ID is required"    unless @id
    raise DomainError, "Name is required"  unless @name.present?
    raise DomainError, "Email is required" unless @email.present?
    raise DomainError, "Invalid email"     unless valid_email?(@email)
  end

  def valid_email?(email)
    email =~ URI::MailTo::EMAIL_REGEXP
  end

  def email_taken?(email)
    CustomerRepository.email_exists?(email, exclude_id: @id)
  end

  def record_event(event)
    domain_events << event
  end

  def domain_events
    @domain_events ||= []
  end
end
```

---

## ขั้นตอนที่ 1665: Value Objects

Value Object คือ object ที่ไม่มี identity - equality ขึ้นอยู่กับ values ทั้งหมด

```ruby
# app/domain/value_objects/money.rb
class Money
  attr_reader :amount, :currency

  def initialize(amount, currency)
    @amount   = BigDecimal(amount.to_s)
    @currency = currency.to_s.upcase

    raise ArgumentError, "Amount cannot be negative" if @amount < 0
    raise ArgumentError, "Invalid currency"          unless valid_currency?(@currency)

    freeze  # Value objects ควร immutable
  end

  def +(other)
    raise TypeError, "Cannot add different currencies" unless same_currency?(other)
    Money.new(@amount + other.amount, @currency)
  end

  def -(other)
    raise TypeError, "Cannot subtract different currencies" unless same_currency?(other)
    result = @amount - other.amount
    raise ArgumentError, "Result would be negative" if result < 0
    Money.new(result, @currency)
  end

  def *(multiplier)
    Money.new(@amount * multiplier, @currency)
  end

  def /(divisor)
    raise ArgumentError, "Cannot divide by zero" if divisor.zero?
    Money.new(@amount / divisor, @currency)
  end

  def zero?
    @amount.zero?
  end

  def positive?
    @amount.positive?
  end

  def ==(other)
    return false unless other.is_a?(Money)
    @amount == other.amount && @currency == other.currency
  end

  alias_method :eql?, :==

  def hash
    [@amount, @currency].hash
  end

  def to_s
    "#{@currency} #{@amount.to_f.round(2)}"
  end

  def inspect
    "#<Money #{to_s}>"
  end

  private

  def same_currency?(other)
    @currency == other.currency
  end

  def valid_currency?(currency)
    Money::Currency::TABLE.keys.map(&:to_s).include?(currency)
  end
end

# ตัวอย่างใช้งาน
price    = Money.new(100, 'THB')
tax      = price * 0.07
total    = price + tax
# => Money(107 THB)

# app/domain/value_objects/email_address.rb
class EmailAddress
  attr_reader :value

  def initialize(value)
    @value = value.to_s.downcase.strip
    raise ArgumentError, "Invalid email: #{value}" unless valid?
    freeze
  end

  def domain
    @value.split('@').last
  end

  def local_part
    @value.split('@').first
  end

  def ==(other)
    return false unless other.is_a?(EmailAddress)
    @value == other.value
  end

  alias_method :eql?, :==
  def hash = @value.hash

  def to_s = @value
  def inspect = "#<EmailAddress #{@value}>"

  private

  def valid?
    @value =~ URI::MailTo::EMAIL_REGEXP
  end
end

# app/domain/value_objects/address.rb
class Address
  attr_reader :street, :city, :state, :postal_code, :country

  def initialize(street:, city:, state:, postal_code:, country:)
    @street      = street.to_s.strip
    @city        = city.to_s.strip
    @state       = state.to_s.strip
    @postal_code = postal_code.to_s.strip
    @country     = country.to_s.upcase.strip

    validate!
    freeze
  end

  def full_address
    "#{@street}, #{@city}, #{@state} #{@postal_code}, #{@country}"
  end

  def domestic?
    @country == 'TH'
  end

  def ==(other)
    return false unless other.is_a?(Address)
    street == other.street &&
      city == other.city &&
      postal_code == other.postal_code &&
      country == other.country
  end

  alias_method :eql?, :==
  def hash = [@street, @city, @postal_code, @country].hash

  private

  def validate!
    raise ArgumentError, "Street is required"      unless @street.present?
    raise ArgumentError, "City is required"        unless @city.present?
    raise ArgumentError, "Postal code is required" unless @postal_code.present?
    raise ArgumentError, "Country is required"     unless @country.present?
  end
end
```

---

## ขั้นตอนที่ 1666: Aggregates

Aggregate คือ cluster ของ Entities และ Value Objects ที่มี single root

```ruby
# app/domain/aggregates/order.rb
class Order  # Aggregate Root
  attr_reader :id, :customer_id, :status, :items, :shipping_address

  def initialize(
    id:,
    customer_id:,
    shipping_address:,
    status: :draft
  )
    @id               = id
    @customer_id      = customer_id
    @shipping_address = shipping_address
    @status           = status
    @items            = []
    @domain_events    = []
  end

  # Commands
  def add_item(product_id:, name:, price:, quantity:)
    raise OrderError, "Cannot modify #{@status} order" unless draft?
    raise ArgumentError, "Quantity must be positive"   unless quantity.positive?
    raise ArgumentError, "Price must be positive"      unless price.positive?

    existing = @items.find { |i| i.product_id == product_id }

    if existing
      existing.increase_quantity(quantity)
    else
      @items << OrderItem.new(
        id:         SecureRandom.uuid,
        product_id: product_id,
        name:       name,
        price:      Money.new(price, 'THB'),
        quantity:   quantity
      )
    end
  end

  def remove_item(product_id:)
    raise OrderError, "Cannot modify #{@status} order" unless draft?

    item = @items.find { |i| i.product_id == product_id }
    raise OrderError, "Item not found" unless item

    @items.delete(item)
  end

  def submit
    raise OrderError, "Order is empty"        if @items.empty?
    raise OrderError, "Order already submitted" unless draft?

    @status = :submitted
    @submitted_at = Time.current

    @domain_events << OrderSubmitted.new(
      order_id:    @id,
      customer_id: @customer_id,
      total:       total.amount,
      items_count: @items.count
    )
  end

  def confirm
    raise OrderError, "Order not submitted" unless submitted?

    @status = :confirmed
    @confirmed_at = Time.current
    @domain_events << OrderConfirmed.new(order_id: @id)
  end

  def ship(tracking_number:, carrier:)
    raise OrderError, "Order not confirmed" unless confirmed?

    @status          = :shipped
    @tracking_number = tracking_number
    @carrier         = carrier
    @shipped_at      = Time.current

    @domain_events << OrderShipped.new(
      order_id:        @id,
      tracking_number: tracking_number,
      carrier:         carrier
    )
  end

  def cancel(reason:)
    raise OrderError, "Cannot cancel #{@status} order" if shipped? || delivered?

    @status       = :cancelled
    @cancelled_at = Time.current
    @cancel_reason = reason

    @domain_events << OrderCancelled.new(
      order_id: @id,
      reason:   reason
    )
  end

  # Queries
  def total
    @items.reduce(Money.new(0, 'THB')) do |sum, item|
      sum + item.subtotal
    end
  end

  def item_count
    @items.sum(&:quantity)
  end

  def draft?      = @status == :draft
  def submitted?  = @status == :submitted
  def confirmed?  = @status == :confirmed
  def shipped?    = @status == :shipped
  def delivered?  = @status == :delivered
  def cancelled?  = @status == :cancelled

  def pop_domain_events
    events         = @domain_events.dup
    @domain_events = []
    events
  end
end

# OrderItem เป็น Entity ภายใน Order aggregate
class OrderItem
  attr_reader :id, :product_id, :name, :price, :quantity

  def initialize(id:, product_id:, name:, price:, quantity:)
    @id         = id
    @product_id = product_id
    @name       = name
    @price      = price  # Money value object
    @quantity   = quantity
  end

  def subtotal
    @price * @quantity
  end

  def increase_quantity(amount)
    raise ArgumentError, "Amount must be positive" unless amount.positive?
    @quantity += amount
  end

  def decrease_quantity(amount)
    raise ArgumentError, "Amount must be positive" unless amount.positive?
    raise ArgumentError, "Cannot go below zero"    if @quantity - amount < 0
    @quantity -= amount
  end
end
```

---

## ขั้นตอนที่ 1667: Domain Events

Domain Events แสดงว่า "สิ่งสำคัญเกิดขึ้นใน domain"

```ruby
# app/domain/events/base_domain_event.rb
class BaseDomainEvent
  attr_reader :event_id, :occurred_at, :data

  def initialize(data = {})
    @event_id    = SecureRandom.uuid
    @occurred_at = Time.current
    @data        = data.freeze
  end

  def event_type
    self.class.name
  end

  def to_h
    {
      event_id:    @event_id,
      event_type:  event_type,
      occurred_at: @occurred_at,
      data:        @data
    }
  end
end

# app/domain/events/order_events.rb
class OrderSubmitted < BaseDomainEvent
  def initialize(order_id:, customer_id:, total:, items_count:)
    super(
      order_id:    order_id,
      customer_id: customer_id,
      total:       total,
      items_count: items_count
    )
  end

  def order_id    = data[:order_id]
  def customer_id = data[:customer_id]
  def total       = data[:total]
end

class OrderShipped < BaseDomainEvent
  def initialize(order_id:, tracking_number:, carrier:)
    super(
      order_id:        order_id,
      tracking_number: tracking_number,
      carrier:         carrier
    )
  end
end

class OrderCancelled < BaseDomainEvent
  def initialize(order_id:, reason:)
    super(order_id: order_id, reason: reason)
  end
end

# Domain Event Dispatcher
class DomainEventDispatcher
  def initialize
    @handlers = Hash.new { |h, k| h[k] = [] }
  end

  def register(event_class, handler)
    @handlers[event_class] << handler
  end

  def dispatch(events)
    Array(events).each do |event|
      @handlers[event.class].each do |handler|
        handler.call(event)
      rescue StandardError => e
        Rails.logger.error "Handler #{handler} failed for #{event.class}: #{e.message}"
      end
    end
  end
end

# config/initializers/domain_events.rb
DOMAIN_EVENT_DISPATCHER = DomainEventDispatcher.new.tap do |dispatcher|
  dispatcher.register(OrderSubmitted,  SendOrderConfirmationEmail.new)
  dispatcher.register(OrderSubmitted,  ReserveInventory.new)
  dispatcher.register(OrderShipped,    SendShipmentNotification.new)
  dispatcher.register(OrderCancelled,  ReleaseInventory.new)
  dispatcher.register(OrderCancelled,  ProcessRefund.new)
end
```

---

## ขั้นตอนที่ 1668: Repositories

Repository pattern ซ่อน persistence details จาก domain

```ruby
# app/domain/repositories/order_repository.rb
module Domain
  module Repositories
    class OrderRepository
      # Interface / Contract
      # Implement ใน infrastructure layer

      def find(id)
        raise NotImplementedError
      end

      def find_by_customer(customer_id)
        raise NotImplementedError
      end

      def save(order)
        raise NotImplementedError
      end

      def delete(id)
        raise NotImplementedError
      end

      def next_id
        SecureRandom.uuid
      end
    end
  end
end

# app/infrastructure/repositories/active_record_order_repository.rb
module Infrastructure
  module Repositories
    class ActiveRecordOrderRepository < Domain::Repositories::OrderRepository

      def find(id)
        record = OrderRecord.includes(:line_item_records).find(id)
        to_domain(record)
      rescue ActiveRecord::RecordNotFound
        nil
      end

      def find_by_customer(customer_id)
        OrderRecord.where(customer_id: customer_id)
                   .includes(:line_item_records)
                   .map { |record| to_domain(record) }
      end

      def save(order)
        record = OrderRecord.find_or_initialize_by(id: order.id)
        update_record(record, order)
        record.save!

        # Dispatch domain events
        events = order.pop_domain_events
        DOMAIN_EVENT_DISPATCHER.dispatch(events)

        order
      end

      def delete(id)
        OrderRecord.find(id).destroy
      end

      private

      def to_domain(record)
        Order.new(
          id:               record.id,
          customer_id:      record.customer_id,
          shipping_address: to_address(record),
          status:           record.status.to_sym,
          items:            record.line_item_records.map { |lr| to_item(lr) }
        )
      end

      def to_address(record)
        Address.new(
          street:      record.shipping_street,
          city:        record.shipping_city,
          state:       record.shipping_state,
          postal_code: record.shipping_postal_code,
          country:     record.shipping_country
        )
      end

      def to_item(record)
        OrderItem.new(
          id:         record.id,
          product_id: record.product_id,
          name:       record.name,
          price:      Money.new(record.price_cents / 100.0, record.currency),
          quantity:   record.quantity
        )
      end

      def update_record(record, order)
        record.assign_attributes(
          customer_id:      order.customer_id,
          status:           order.status,
          shipping_street:  order.shipping_address.street,
          shipping_city:    order.shipping_address.city,
          shipping_state:   order.shipping_address.state,
          shipping_postal_code: order.shipping_address.postal_code,
          shipping_country: order.shipping_address.country
        )

        existing_ids = record.line_item_records.map(&:id)
        new_ids      = order.items.map(&:id)
        to_delete    = existing_ids - new_ids

        record.line_item_records.where(id: to_delete).destroy_all

        order.items.each do |item|
          lr = record.line_item_records.find_or_initialize_by(id: item.id)
          lr.assign_attributes(
            product_id:  item.product_id,
            name:        item.name,
            price_cents: (item.price.amount * 100).to_i,
            currency:    item.price.currency,
            quantity:    item.quantity
          )
        end
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1669: Application Services

Application Services orchestrate domain objects และ handle use cases

```ruby
# app/application/services/order_service.rb
module Application
  module Services
    class OrderService
      def initialize(
        order_repository:,
        customer_repository:,
        product_catalog:,
        event_dispatcher: DOMAIN_EVENT_DISPATCHER
      )
        @orders     = order_repository
        @customers  = customer_repository
        @catalog    = product_catalog
        @dispatcher = event_dispatcher
      end

      # Use case: Place an order
      def place_order(customer_id:, items:, shipping_address:)
        customer = @customers.find(customer_id)
        raise ApplicationError, "Customer not found" unless customer
        raise ApplicationError, "Customer suspended" unless customer.active?

        address = Address.new(**shipping_address)
        order   = Order.new(
          id:               @orders.next_id,
          customer_id:      customer_id,
          shipping_address: address
        )

        items.each do |item_params|
          product = @catalog.find(item_params[:product_id])
          raise ApplicationError, "Product #{item_params[:product_id]} not found" unless product
          raise ApplicationError, "#{product.name} is out of stock" unless product.in_stock?

          order.add_item(
            product_id: product.id,
            name:       product.name,
            price:      product.price.amount,
            quantity:   item_params[:quantity]
          )
        end

        order.submit

        @orders.save(order)

        { order_id: order.id, total: order.total.to_s, status: order.status }
      end

      # Use case: Cancel an order
      def cancel_order(order_id:, reason:, cancelled_by:)
        order = @orders.find(order_id)
        raise ApplicationError, "Order not found" unless order

        order.cancel(reason: reason)
        @orders.save(order)

        AuditLog.create!(
          action:      'order_cancelled',
          resource_id: order_id,
          user_id:     cancelled_by,
          details:     { reason: reason }
        )

        true
      end

      # Use case: Confirm an order (admin action)
      def confirm_order(order_id:, confirmed_by:)
        order = @orders.find(order_id)
        raise ApplicationError, "Order not found" unless order

        order.confirm
        @orders.save(order)

        true
      end
    end
  end
end

# การใช้งานใน Controller
class OrdersController < ApplicationController
  def create
    service = Application::Services::OrderService.new(
      order_repository:    Infrastructure::Repositories::ActiveRecordOrderRepository.new,
      customer_repository: Infrastructure::Repositories::ActiveRecordCustomerRepository.new,
      product_catalog:     Infrastructure::Catalog::ActiveRecordProductCatalog.new
    )

    result = service.place_order(
      customer_id:      current_user.id,
      items:            order_params[:items],
      shipping_address: order_params[:shipping_address]
    )

    render json: result, status: :created
  rescue ApplicationError => e
    render json: { error: e.message }, status: :unprocessable_entity
  rescue DomainError => e
    render json: { error: e.message }, status: :unprocessable_entity
  end
end
```

---

## ขั้นตอนที่ 1670: Domain Services

Domain Service คือ operations ที่ไม่ fit ใน Entity หรือ Value Object ใด

```ruby
# app/domain/services/price_calculator.rb
module Domain
  module Services
    class PriceCalculator
      def initialize(tax_rates:, discount_rules:)
        @tax_rates       = tax_rates
        @discount_rules  = discount_rules
      end

      def calculate(order)
        subtotal  = order.items.sum(&:subtotal)
        discounts = apply_discounts(order, subtotal)
        tax       = calculate_tax(subtotal - discounts, order.shipping_address)

        {
          subtotal:   subtotal,
          discounts:  discounts,
          tax:        tax,
          total:      subtotal - discounts + tax
        }
      end

      private

      def apply_discounts(order, subtotal)
        applicable_rules = @discount_rules.select { |rule| rule.applicable?(order) }
        applicable_rules.sum { |rule| rule.discount_amount(subtotal) }
      end

      def calculate_tax(amount, address)
        rate = @tax_rates.for_region(address.state, address.country)
        amount * rate
      end
    end
  end
end

# app/domain/services/inventory_allocator.rb
module Domain
  module Services
    class InventoryAllocator
      def initialize(inventory_repository)
        @inventory = inventory_repository
      end

      # Domain service ที่ต้องการ access หลาย aggregates
      def allocate_for_order(order)
        allocations = []

        order.items.each do |item|
          stock = @inventory.find_by_product(item.product_id)

          raise InsufficientStockError, "#{item.name} is out of stock" unless stock
          raise InsufficientStockError, "Not enough #{item.name}" unless stock.available >= item.quantity

          allocation = stock.allocate(item.quantity, order_id: order.id)
          allocations << allocation
        end

        allocations
      rescue InsufficientStockError => e
        # Rollback ถ้า fail
        allocations.each { |a| a.deallocate }
        raise
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1671: Dry-rb สำหรับ DDD

Dry-rb เป็น collection ของ gems ที่ช่วย implement DDD patterns ใน Ruby

### ติดตั้ง Dry gems

```ruby
# Gemfile
gem 'dry-validation'
gem 'dry-struct'
gem 'dry-types'
gem 'dry-monads'
gem 'dry-events'
```

### Dry::Struct สำหรับ Value Objects

```ruby
# app/domain/value_objects/money_dry.rb
require 'dry-struct'
require 'dry-types'

module Types
  include Dry.Types()

  PositiveDecimal = Types::Decimal.constrained(gt: 0)
  Currency        = Types::String.enum('THB', 'USD', 'EUR', 'GBP')
end

class MoneyDry < Dry::Struct
  attribute :amount,   Types::Decimal.constrained(gteq: 0)
  attribute :currency, Types::Currency

  def +(other)
    raise TypeError, "Currency mismatch" unless currency == other.currency
    self.class.new(amount: amount + other.amount, currency: currency)
  end

  def *(multiplier)
    self.class.new(amount: amount * multiplier, currency: currency)
  end

  def zero?
    amount.zero?
  end

  def to_s
    "#{currency} #{amount.to_f.round(2)}"
  end
end

# ตัวอย่าง Dry::Struct สำหรับ Entity
class ProductDry < Dry::Struct
  attribute :id,          Types::UUID
  attribute :name,        Types::String.constrained(min_size: 1)
  attribute :price,       MoneyDry
  attribute :category,    Types::String
  attribute :in_stock,    Types::Bool

  def discounted_price(percentage)
    new_amount = price.amount * (1 - percentage / 100.0)
    MoneyDry.new(amount: new_amount, currency: price.currency)
  end
end
```

### Dry::Validation สำหรับ Commands

```ruby
# app/application/contracts/place_order_contract.rb
require 'dry-validation'

class PlaceOrderContract < Dry::Validation::Contract
  params do
    required(:customer_id).filled(:string)
    required(:items).array(:hash) do
      required(:product_id).filled(:string)
      required(:quantity).filled(:integer)
    end
    required(:shipping_address).hash do
      required(:street).filled(:string)
      required(:city).filled(:string)
      required(:postal_code).filled(:string)
      required(:country).filled(:string)
    end
  end

  rule(:items) do
    unless value.all? { |item| item[:quantity].positive? }
      key.failure("all quantities must be positive")
    end
  end

  rule(:shipping_address) do
    unless %w[TH US GB EU].include?(value[:country])
      key(:country).failure("is not a supported country")
    end
  end
end

# การใช้งาน
contract = PlaceOrderContract.new
result   = contract.call(params)

if result.success?
  service.place_order(**result.to_h)
else
  render json: { errors: result.errors.to_h }, status: :unprocessable_entity
end
```

### Dry::Monads สำหรับ Error Handling

```ruby
# app/application/services/order_service_monadic.rb
require 'dry-monads'

class OrderServiceMonadic
  include Dry::Monads[:result, :do]

  def place_order(params)
    # Validate
    validation = yield validate_params(params)

    # Find customer
    customer   = yield find_customer(validation[:customer_id])

    # Check customer status
    yield check_customer_active(customer)

    # Build order
    order      = yield build_order(customer, validation)

    # Save
    yield save_order(order)

    Success(order.id)
  end

  private

  def validate_params(params)
    result = PlaceOrderContract.new.call(params)
    result.success? ? Success(result.to_h) : Failure(result.errors.to_h)
  end

  def find_customer(customer_id)
    customer = CustomerRepository.find(customer_id)
    customer ? Success(customer) : Failure("Customer not found")
  end

  def check_customer_active(customer)
    customer.active? ? Success(customer) : Failure("Customer is suspended")
  end

  def build_order(customer, params)
    Success(Order.new(
      id:          SecureRandom.uuid,
      customer_id: customer.id,
      # ...
    ))
  rescue DomainError => e
    Failure(e.message)
  end

  def save_order(order)
    OrderRepository.save(order)
    Success(order)
  rescue ActiveRecord::RecordInvalid => e
    Failure(e.message)
  end
end

# การใช้งาน
result = OrderServiceMonadic.new.place_order(params)

case result
in Success[order_id]
  render json: { order_id: order_id }
in Failure[errors]
  render json: { errors: errors }, status: :unprocessable_entity
end
```

---

## ขั้นตอนที่ 1672: Folder Structure สำหรับ DDD

```
app/
├── application/                    # Application layer
│   ├── commands/
│   │   ├── place_order_command.rb
│   │   └── cancel_order_command.rb
│   ├── contracts/
│   │   ├── place_order_contract.rb
│   │   └── register_customer_contract.rb
│   └── services/
│       ├── order_service.rb
│       └── customer_service.rb
│
├── domain/                         # Domain layer (core business)
│   ├── aggregates/
│   │   ├── order.rb
│   │   └── customer.rb
│   ├── entities/
│   │   ├── order_item.rb
│   │   └── product.rb
│   ├── value_objects/
│   │   ├── money.rb
│   │   ├── address.rb
│   │   └── email_address.rb
│   ├── events/
│   │   ├── order_submitted.rb
│   │   └── order_cancelled.rb
│   ├── repositories/
│   │   └── order_repository.rb     # interface
│   ├── services/
│   │   ├── price_calculator.rb
│   │   └── inventory_allocator.rb
│   └── errors/
│       ├── domain_error.rb
│       └── order_error.rb
│
└── infrastructure/                 # Infrastructure layer
    ├── repositories/
    │   ├── active_record_order_repository.rb
    │   └── active_record_customer_repository.rb
    ├── persistence/
    │   ├── order_record.rb         # ActiveRecord model
    │   └── customer_record.rb
    └── external/
        ├── payment_gateway.rb
        └── email_service.rb
```

---

## ขั้นตอนที่ 1673: Hanami Framework สำหรับ DDD

Hanami เป็น Ruby framework ที่ออกแบบมาเพื่อ DDD

```ruby
# Gemfile
gem 'hanami', '~> 2.0'
gem 'hanami-db'

# app/actions/orders/create.rb (Hanami Action)
module Orders
  class Create < Hanami::Action
    include Deps[
      'application.order_service',
      'contracts.place_order_contract'
    ]

    def handle(request, response)
      contract_result = place_order_contract.call(request.params)

      if contract_result.failure?
        response.status  = 422
        response[:body]  = { errors: contract_result.errors.to_h }
        return
      end

      result = order_service.place_order(**contract_result.to_h)

      response.status = 201
      response[:body] = { order_id: result }
    end
  end
end

# app/repos/order_repo.rb (Hanami Repo)
module Repos
  class OrderRepo < Hanami::DB::Repo
    struct_namespace Domain

    def find_by_customer(customer_id)
      orders.where(customer_id: customer_id).map_to(Domain::Order).to_a
    end

    def save(order)
      if order.persisted?
        orders.where(id: order.id).update(to_record(order))
      else
        orders.insert(to_record(order)).returning(:id).first
      end
    end

    private

    def orders
      root
    end

    def to_record(order)
      {
        id:          order.id,
        customer_id: order.customer_id,
        status:      order.status.to_s,
        total_cents: (order.total.amount * 100).to_i
      }
    end
  end
end
```

---

## ขั้นตอนที่ 1674: Testing DDD Components

```ruby
# spec/domain/aggregates/order_spec.rb
RSpec.describe Order do
  let(:customer_id)      { SecureRandom.uuid }
  let(:shipping_address) {
    Address.new(
      street: '123 Main St', city: 'Bangkok',
      state: 'BKK', postal_code: '10100', country: 'TH'
    )
  }
  let(:order) { Order.new(id: SecureRandom.uuid, customer_id: customer_id, shipping_address: shipping_address) }

  describe '#add_item' do
    it 'adds item to order' do
      order.add_item(product_id: 'p1', name: 'Widget', price: 100, quantity: 2)

      expect(order.items.count).to eq(1)
      expect(order.items.first.quantity).to eq(2)
    end

    it 'increases quantity for duplicate product' do
      order.add_item(product_id: 'p1', name: 'Widget', price: 100, quantity: 2)
      order.add_item(product_id: 'p1', name: 'Widget', price: 100, quantity: 3)

      expect(order.items.count).to eq(1)
      expect(order.items.first.quantity).to eq(5)
    end

    it 'raises error for non-draft orders' do
      order.add_item(product_id: 'p1', name: 'Widget', price: 100, quantity: 1)
      order.submit

      expect { order.add_item(product_id: 'p2', name: 'Gadget', price: 50, quantity: 1) }
        .to raise_error(OrderError, /Cannot modify submitted order/)
    end
  end

  describe '#total' do
    it 'calculates correct total' do
      order.add_item(product_id: 'p1', name: 'Widget', price: 100, quantity: 2)
      order.add_item(product_id: 'p2', name: 'Gadget', price: 50,  quantity: 3)

      expect(order.total).to eq(Money.new(350, 'THB'))
    end
  end

  describe '#submit' do
    it 'submits order with items' do
      order.add_item(product_id: 'p1', name: 'Widget', price: 100, quantity: 1)
      order.submit

      expect(order.submitted?).to be true
    end

    it 'publishes OrderSubmitted event' do
      order.add_item(product_id: 'p1', name: 'Widget', price: 100, quantity: 1)
      order.submit

      events = order.pop_domain_events
      expect(events).to include(an_instance_of(OrderSubmitted))
    end

    it 'raises error for empty order' do
      expect { order.submit }.to raise_error(OrderError, "Order is empty")
    end
  end
end

# spec/domain/value_objects/money_spec.rb
RSpec.describe Money do
  describe 'arithmetic' do
    it 'adds same currency' do
      a = Money.new(100, 'THB')
      b = Money.new(50, 'THB')

      expect(a + b).to eq(Money.new(150, 'THB'))
    end

    it 'raises error for different currencies' do
      a = Money.new(100, 'THB')
      b = Money.new(3, 'USD')

      expect { a + b }.to raise_error(TypeError)
    end
  end

  describe 'equality' do
    it 'is equal when amount and currency match' do
      expect(Money.new(100, 'THB')).to eq(Money.new(100, 'THB'))
    end

    it 'is not equal when amount differs' do
      expect(Money.new(100, 'THB')).not_to eq(Money.new(101, 'THB'))
    end
  end
end
```

---

## ขั้นตอนที่ 1675: Integration กับ Rails

```ruby
# app/models/order_record.rb (ActiveRecord - ชื่อต่างจาก Domain::Order)
class OrderRecord < ApplicationRecord
  self.table_name = 'orders'

  has_many :line_item_records, foreign_key: :order_id, class_name: 'LineItemRecord'

  enum status: {
    draft:     'draft',
    submitted: 'submitted',
    confirmed: 'confirmed',
    shipped:   'shipped',
    delivered: 'delivered',
    cancelled: 'cancelled'
  }

  validates :customer_id, presence: true
end

# app/models/line_item_record.rb
class LineItemRecord < ApplicationRecord
  self.table_name = 'line_items'
  belongs_to :order_record, foreign_key: :order_id

  validates :product_id, :name, :quantity, presence: true
  validates :quantity, numericality: { greater_than: 0 }
end

# config/application.rb - configure DDD dependencies
module MyApp
  class Application < Rails::Application
    config.to_prepare do
      # Wire up dependencies
      order_repo    = Infrastructure::Repositories::ActiveRecordOrderRepository.new
      customer_repo = Infrastructure::Repositories::ActiveRecordCustomerRepository.new
      product_cat   = Infrastructure::Catalog::ActiveRecordProductCatalog.new

      Rails.configuration.order_service = Application::Services::OrderService.new(
        order_repository:    order_repo,
        customer_repository: customer_repo,
        product_catalog:     product_cat
      )
    end
  end
end
```

---

## ขั้นตอนที่ 1676: Anti-Corruption Layer (ACL)

ACL ป้องกัน domain ของเราจาก external systems

```ruby
# app/infrastructure/external/stripe_payment_gateway.rb
module Infrastructure
  module External
    class StripePaymentGateway
      # ACL: แปลง domain objects เป็น Stripe API calls

      def initialize(api_key:)
        Stripe.api_key = api_key
      end

      # Domain interface: receive domain objects, return domain results
      def charge(payment_request)
        stripe_result = Stripe::PaymentIntent.create(
          amount:   to_cents(payment_request.amount),
          currency: payment_request.currency.downcase,
          metadata: { order_id: payment_request.reference }
        )

        from_stripe_result(stripe_result)
      rescue Stripe::CardError => e
        Domain::Payment::ChargeResult.failure(reason: e.message, code: e.code)
      rescue Stripe::StripeError => e
        Domain::Payment::ChargeResult.failure(reason: 'Payment service error')
      end

      private

      def to_cents(money)
        (money.amount * 100).to_i
      end

      def from_stripe_result(stripe_result)
        case stripe_result.status
        when 'succeeded'
          Domain::Payment::ChargeResult.success(
            payment_id: stripe_result.id,
            amount:     Money.new(stripe_result.amount / 100.0, stripe_result.currency.upcase)
          )
        when 'requires_payment_method'
          Domain::Payment::ChargeResult.failure(reason: 'Card declined')
        else
          Domain::Payment::ChargeResult.failure(reason: "Unknown status: #{stripe_result.status}")
        end
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1677: Specification Pattern

```ruby
# app/domain/specifications/order_specification.rb
module Domain
  module Specifications
    class OrderSpecification
      def satisfied_by?(order)
        raise NotImplementedError
      end

      def &(other)
        AndSpecification.new(self, other)
      end

      def |(other)
        OrSpecification.new(self, other)
      end

      def ~
        NotSpecification.new(self)
      end
    end

    class AndSpecification < OrderSpecification
      def initialize(left, right)
        @left  = left
        @right = right
      end

      def satisfied_by?(order)
        @left.satisfied_by?(order) && @right.satisfied_by?(order)
      end
    end

    class OrSpecification < OrderSpecification
      def initialize(left, right)
        @left  = left
        @right = right
      end

      def satisfied_by?(order)
        @left.satisfied_by?(order) || @right.satisfied_by?(order)
      end
    end

    class NotSpecification < OrderSpecification
      def initialize(spec)
        @spec = spec
      end

      def satisfied_by?(order)
        !@spec.satisfied_by?(order)
      end
    end

    # Concrete specifications
    class EligibleForFreeShipping < OrderSpecification
      MINIMUM_AMOUNT = Money.new(500, 'THB')

      def satisfied_by?(order)
        order.total >= MINIMUM_AMOUNT
      end
    end

    class HasVIPCustomer < OrderSpecification
      def initialize(customer_repository)
        @customers = customer_repository
      end

      def satisfied_by?(order)
        customer = @customers.find(order.customer_id)
        customer&.vip?
      end
    end

    class IsExpressOrder < OrderSpecification
      def satisfied_by?(order)
        order.shipping_method == :express
      end
    end
  end
end

# การใช้งาน
free_shipping_spec   = Specifications::EligibleForFreeShipping.new
vip_spec             = Specifications::HasVIPCustomer.new(customer_repo)
express_spec         = Specifications::IsExpressOrder.new

# Free shipping for orders > 500 THB OR VIP customers
free_shipping = free_shipping_spec | vip_spec
# Express orders without free shipping
priority_paid = express_spec & ~free_shipping

order.qualifies_for_free_shipping = free_shipping.satisfied_by?(order)
```

---

## ขั้นตอนที่ 1678: Domain Error Handling

```ruby
# app/domain/errors/domain_error.rb
class DomainError < StandardError
  attr_reader :code, :context

  def initialize(message, code: nil, context: {})
    super(message)
    @code    = code || self.class.name.underscore.to_sym
    @context = context
  end
end

# app/domain/errors/order_errors.rb
class OrderError           < DomainError; end
class InsufficientFunds    < DomainError; end
class InsufficientStock    < DomainError
  attr_reader :product_id, :requested, :available

  def initialize(product_id:, requested:, available:)
    @product_id = product_id
    @requested  = requested
    @available  = available
    super("Insufficient stock for product #{product_id}: requested #{requested}, available #{available}")
  end
end
class InvalidStateTransition < DomainError
  def initialize(from:, to:)
    super("Cannot transition from #{from} to #{to}")
  end
end

# Global error handling ใน ApplicationController
class ApplicationController < ActionController::API
  rescue_from DomainError, with: :handle_domain_error
  rescue_from ApplicationError, with: :handle_application_error

  private

  def handle_domain_error(error)
    render json: {
      error:   error.message,
      code:    error.code,
      context: error.context
    }, status: :unprocessable_entity
  end

  def handle_application_error(error)
    render json: { error: error.message }, status: :bad_request
  end
end
```

---

## ขั้นตอนที่ 1679: Context Integration Patterns

```ruby
# Shared Kernel - code ที่ใช้ร่วมระหว่าง contexts
module SharedKernel
  class UniqueId
    def self.generate
      SecureRandom.uuid
    end
  end

  class DomainEvent
    attr_reader :id, :occurred_at

    def initialize
      @id          = UniqueId.generate
      @occurred_at = Time.current
    end
  end
end

# Published Language - interface ระหว่าง contexts ผ่าน events
module Sales
  class OrderConfirmed < SharedKernel::DomainEvent
    attr_reader :order_id, :customer_id, :items, :total

    def initialize(order_id:, customer_id:, items:, total:)
      super()
      @order_id    = order_id
      @customer_id = customer_id
      @items       = items
      @total       = total
    end
  end
end

# Inventory context subscribe ต่อ Sales events
module Inventory
  class OrderConfirmedHandler
    def call(event)
      # แปลง Sales::OrderConfirmed event เป็น Inventory domain terms
      event.items.each do |item|
        inventory_item = InventoryItem.find_by!(sku: item[:product_id])
        inventory_item.reserve(quantity: item[:quantity], order_id: event.order_id)
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1680: แนวปฏิบัติที่ดี

```ruby
# 1. Keep domain pure - ไม่มี ActiveRecord ใน domain layer
class Order  # ❌ ไม่ควร inherit ApplicationRecord
  # ✅ Plain Ruby object
end

# 2. Invariants enforce ใน aggregate
class BankAccount
  def withdraw(amount:)
    # ✅ Business rules อยู่ใน aggregate
    raise InsufficientFunds.new if balance < amount
    raise AccountFrozen.new     if frozen?
    # ...
  end
end

# 3. Value Objects สำหรับ domain concepts สำคัญ
# ❌ ใช้ primitive ตรงๆ
def transfer(from_id, to_id, amount_float)
  # amount เป็น float? หน่วยอะไร?
end

# ✅ ใช้ Value Object
def transfer(from_account:, to_account:, amount:)
  # amount คือ Money value object - ชัดเจน
end

# 4. Repositories ซ่อน persistence
class OrderService
  def initialize(order_repository:)  # ✅ inject repository
    @orders = order_repository
  end
  # ไม่รู้ว่า database ใช้อะไร
end
```

---

## แบบฝึกหัด: Domain-Driven Design

### ข้อที่ 1: Modeling
ออกแบบ domain model สำหรับ Hospital Management System ระบุ:
- Bounded Contexts (อย่างน้อย 3)
- Entities หลักในแต่ละ context
- Value Objects สำคัญ
- Aggregates และ Aggregate Roots

**คำตอบ:**
```
Bounded Contexts:
1. Patient Care Context: Patient, Appointment, MedicalRecord, Doctor
2. Billing Context: Invoice, Payment, InsuranceClaim, Coverage
3. Pharmacy Context: Prescription, Medication, Inventory, Dispense

Value Objects: BloodType, PatientId, Dosage, DrugCode, Symptoms
Aggregates: Patient (root), Appointment, Prescription
```

### ข้อที่ 2: สร้าง Value Object

```ruby
# สร้าง BloodType value object
class BloodType
  VALID_TYPES = %w[A+ A- B+ B- AB+ AB- O+ O-].freeze

  attr_reader :type

  def initialize(type)
    @type = type.upcase.strip
    raise ArgumentError, "Invalid blood type: #{type}" unless valid?
    freeze
  end

  def compatible_donors
    case @type
    when 'O-' then %w[O-]
    when 'O+' then %w[O+ O-]
    when 'A-' then %w[A- O-]
    when 'A+' then %w[A+ A- O+ O-]
    when 'B-' then %w[B- O-]
    when 'B+' then %w[B+ B- O+ O-]
    when 'AB-' then %w[AB- A- B- O-]
    when 'AB+' then VALID_TYPES  # universal recipient
    end
  end

  def compatible_with?(donor_blood_type)
    compatible_donors.include?(donor_blood_type.type)
  end

  def ==(other)
    other.is_a?(BloodType) && @type == other.type
  end

  def to_s = @type

  private

  def valid?
    VALID_TYPES.include?(@type)
  end
end
```

### ข้อที่ 3: สร้าง Patient Aggregate

```ruby
class Patient
  attr_reader :id, :name, :blood_type, :status

  def initialize(id:, name:, blood_type:, date_of_birth:)
    @id            = id
    @name          = name
    @blood_type    = BloodType.new(blood_type)
    @date_of_birth = date_of_birth
    @status        = :active
    @appointments  = []
    @domain_events = []
  end

  def schedule_appointment(doctor_id:, datetime:, reason:)
    raise PatientError, "Patient is not active" unless active?
    raise PatientError, "Appointment time conflicts" if has_conflict?(datetime)

    appointment = Appointment.new(
      id:         SecureRandom.uuid,
      patient_id: @id,
      doctor_id:  doctor_id,
      datetime:   datetime,
      reason:     reason
    )

    @appointments << appointment
    @domain_events << AppointmentScheduled.new(
      patient_id:     @id,
      appointment_id: appointment.id,
      doctor_id:      doctor_id,
      datetime:       datetime
    )

    appointment
  end

  def admit
    raise PatientError, "Already admitted" if admitted?
    @status = :admitted
    @domain_events << PatientAdmitted.new(patient_id: @id, admitted_at: Time.current)
  end

  def discharge(notes:)
    raise PatientError, "Patient not admitted" unless admitted?
    @status = :active
    @domain_events << PatientDischarged.new(
      patient_id:    @id,
      discharged_at: Time.current,
      notes:         notes
    )
  end

  def age
    ((Time.current - @date_of_birth.to_time) / 1.year).floor
  end

  def active?   = @status == :active
  def admitted? = @status == :admitted

  def pop_domain_events
    events         = @domain_events.dup
    @domain_events = []
    events
  end

  private

  def has_conflict?(datetime)
    @appointments.any? { |a|
      (a.datetime - datetime).abs < 1.hour && a.active?
    }
  end
end
```

### ข้อที่ 4-20: แบบฝึกหัดเพิ่มเติม

**ข้อ 4:** สร้าง Repository interface และ ActiveRecord implementation สำหรับ Patient

**ข้อ 5:** สร้าง Application Service สำหรับ "Book Appointment" use case

**ข้อ 6:** Implement SpecificationPattern สำหรับ "AvailableAppointmentSlots"

**ข้อ 7:** เพิ่ม Domain Events: AppointmentCancelled, AppointmentRescheduled

**ข้อ 8:** สร้าง Anti-Corruption Layer กับ external HL7 medical system

**ข้อ 9:** ใช้ Dry::Validation สำหรับ ScheduleAppointment command validation

**ข้อ 10:** ใช้ Dry::Monads ใน Application Service (Result monad)

**ข้อ 11:** Design Billing Context กับ Invoice aggregate

**ข้อ 12:** สร้าง Context Map ระหว่าง Patient Care และ Billing contexts

**ข้อ 13:** เขียน unit tests สำหรับ Patient aggregate (100% coverage)

**ข้อ 14:** สร้าง Domain Service สำหรับ "TreatmentPlanner" ที่ coordinates หลาย aggregates

**ข้อ 15:** Implement Saga สำหรับ patient admission workflow

**ข้อ 16:** สร้าง folder structure สมบูรณ์สำหรับ Hospital system

**ข้อ 17:** ทำ Event Sourcing สำหรับ Patient aggregate

**ข้อ 18:** สร้าง GraphQL API ที่ expose domain objects ผ่าน presentation layer

**ข้อ 19:** เพิ่ม Audit Trail ให้ทุก domain events

**ข้อ 20:** Deploy Hospital system พร้อม monitoring ครบถ้วน

---

## สรุป

DDD เป็น philosophy ที่ช่วยให้ software reflect business domain ได้แม่นยำ:
- **Strategic Design**: แบ่ง system เป็น Bounded Contexts
- **Tactical Design**: implement domain ด้วย Entities, Value Objects, Aggregates
- **Clean Architecture**: แยก domain จาก infrastructure
- **Rich Domain Model**: business logic อยู่ใน domain, ไม่กระจัดกระจาย

ขั้นตอนต่อไป: Part 78 - Advanced Performance
