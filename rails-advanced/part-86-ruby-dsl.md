# Part 86: Ruby DSL (Domain-Specific Language)

## บทนำ

DSL (Domain-Specific Language) เป็นหนึ่งในความสามารถที่ทรงพลังที่สุดของ Ruby ทำให้สามารถสร้างภาษาหรือ API ที่อ่านง่ายและแสดงความตั้งใจได้ชัดเจน ในบทนี้เราจะเรียนรู้การสร้าง Internal DSL ในหลายรูปแบบ

---

## 1. DSL Design Patterns ใน Ruby

### 1.1 ความเข้าใจ DSL

DSL มี 2 ประเภทหลัก:

- **Internal DSL**: เขียนด้วย Ruby syntax ปกติ (Rake, RSpec, Rails routes)
- **External DSL**: ภาษาใหม่ที่ต้องมี parser (Haml, ERB, Markdown)

```ruby
# ตัวอย่าง Internal DSL ที่เราใช้ทุกวัน
# Rake
task :deploy do
  sh 'bundle exec cap production deploy'
end

# RSpec
describe User do
  it 'validates email' do
    expect(user.valid?).to be true
  end
end

# Rails Routes
resources :users do
  member do
    post :activate
  end
end

# ActiveRecord
class User < ApplicationRecord
  validates :name, presence: true
  belongs_to :organization
  scope :active, -> { where(active: true) }
end
```

### 1.2 การออกแบบ DSL ที่ดี

```ruby
# หลักการออกแบบ DSL:
# 1. อ่านเข้าใจง่าย (Readable)
# 2. แสดง intent ไม่ใช่ implementation
# 3. ป้องกัน errors ด้วย validation

# DSL ที่ไม่ดี (เหมือนโค้ดทั่วไป):
config = Configuration.new
config.set_host('localhost')
config.set_port(3000)
config.enable_ssl(true)
config.set_timeout(30)

# DSL ที่ดี (อ่านเข้าใจง่าย):
configure do
  host 'localhost'
  port 3000
  enable :ssl
  timeout 30.seconds
end
```

---

## 2. Internal DSL ด้วย instance_eval/module_eval

### 2.1 instance_eval พื้นฐาน

```ruby
# instance_eval ทำให้ block ทำงานใน context ของ object
# self ใน block คือ object ที่ call instance_eval

class Configuration
  attr_reader :settings
  
  def initialize
    @settings = {}
  end
  
  def configure(&block)
    instance_eval(&block)
    self
  end
  
  def method_missing(name, *args, **kwargs)
    if name.to_s.end_with?('=')
      @settings[name.to_s.chomp('=')] = args.first
    elsif args.any?
      @settings[name.to_s] = args.first
    else
      @settings[name.to_s]
    end
  end
  
  def respond_to_missing?(name, include_private = false)
    true
  end
end

# การใช้งาน
config = Configuration.new.configure do
  host 'localhost'
  port 3000
  database 'myapp'
  ssl true
end

config.settings  # => {"host"=>"localhost", "port"=>3000, ...}
```

### 2.2 DSL Builder Pattern

```ruby
# Config DSL แบบสมบูรณ์
class AppConfig
  class Server
    attr_reader :options
    
    def initialize
      @options = {
        host: 'localhost',
        port: 3000,
        workers: 2,
        threads: 5
      }
    end
    
    def host(value = nil)
      value ? @options[:host] = value : @options[:host]
    end
    
    def port(value = nil)
      value ? @options[:port] = value.to_i : @options[:port]
    end
    
    def workers(value = nil)
      value ? @options[:workers] = value.to_i : @options[:workers]
    end
    
    def threads(value = nil)
      value ? @options[:threads] = value.to_i : @options[:threads]
    end
    
    def ssl?
      @options[:ssl] || false
    end
    
    def ssl(value = nil)
      value.nil? ? ssl? : @options[:ssl] = value
    end
    
    def enable(feature)
      @options[feature] = true
    end
    
    def disable(feature)
      @options[feature] = false
    end
  end
  
  class Database
    attr_reader :options
    
    def initialize
      @options = {
        adapter: 'postgresql',
        pool: 5,
        timeout: 5000
      }
    end
    
    def adapter(value = nil)
      value ? @options[:adapter] = value : @options[:adapter]
    end
    
    def url(value = nil)
      value ? @options[:url] = value : @options[:url]
    end
    
    def pool(value = nil)
      value ? @options[:pool] = value.to_i : @options[:pool]
    end
    
    def replica(host, options = {})
      @options[:replicas] ||= []
      @options[:replicas] << { host: host }.merge(options)
    end
  end
  
  class Mailer
    attr_reader :options
    
    def initialize
      @options = { delivery_method: :smtp }
    end
    
    def smtp(&block)
      smtp_config = SmtpConfig.new
      smtp_config.instance_eval(&block) if block_given?
      @options[:smtp] = smtp_config.options
    end
    
    def from(address)
      @options[:from] = address
    end
    
    def reply_to(address)
      @options[:reply_to] = address
    end
  end
  
  class SmtpConfig
    attr_reader :options
    
    def initialize
      @options = {}
    end
    
    def host(value); @options[:host] = value; end
    def port(value); @options[:port] = value.to_i; end
    def username(value); @options[:username] = value; end
    def password(value); @options[:password] = value; end
    def authentication(value); @options[:authentication] = value; end
    def enable_starttls_auto(value = true); @options[:enable_starttls_auto] = value; end
  end
  
  attr_reader :server, :database, :mailer
  
  def initialize
    @server = Server.new
    @database = Database.new
    @mailer = Mailer.new
  end
  
  def self.define(&block)
    config = new
    config.instance_eval(&block)
    config
  end
  
  def server(&block)
    if block_given?
      @server.instance_eval(&block)
    else
      @server
    end
  end
  
  def database(&block)
    if block_given?
      @database.instance_eval(&block)
    else
      @database
    end
  end
  
  def mailer(&block)
    if block_given?
      @mailer.instance_eval(&block)
    else
      @mailer
    end
  end
  
  def to_h
    {
      server: server.options,
      database: database.options,
      mailer: mailer.options
    }
  end
end

# การใช้งาน
config = AppConfig.define do
  server do
    host 'app.example.com'
    port 443
    workers 4
    threads 8
    enable :ssl
    enable :force_ssl
  end
  
  database do
    adapter 'postgresql'
    url ENV['DATABASE_URL']
    pool 10
    replica 'db-replica-1.example.com', priority: 1
    replica 'db-replica-2.example.com', priority: 2
  end
  
  mailer do
    from 'noreply@example.com'
    reply_to 'support@example.com'
    
    smtp do
      host 'smtp.sendgrid.net'
      port 587
      username ENV['SMTP_USERNAME']
      password ENV['SMTP_PASSWORD']
      authentication :plain
      enable_starttls_auto true
    end
  end
end

puts config.server.options
# => {:host=>"app.example.com", :port=>443, :workers=>4, ...}
```

### 2.3 module_eval สำหรับ Dynamic Methods

```ruby
# module_eval ใช้สำหรับ define methods บน class/module
module Attributable
  def self.included(base)
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def attributes(*names)
      names.each do |name|
        # สร้าง getter และ setter สำหรับแต่ละ attribute
        module_eval do
          define_method(name) do
            instance_variable_get("@#{name}")
          end
          
          define_method("#{name}=") do |value|
            instance_variable_set("@#{name}", value)
          end
          
          define_method("#{name}?") do
            !!instance_variable_get("@#{name}")
          end
        end
      end
      
      # เก็บ list ของ attributes
      @_attributes ||= []
      @_attributes.concat(names)
    end
    
    def attribute_names
      @_attributes || []
    end
  end
  
  def to_h
    self.class.attribute_names.each_with_object({}) do |attr, hash|
      hash[attr] = send(attr)
    end
  end
  
  def initialize(attrs = {})
    attrs.each do |key, value|
      send("#{key}=", value) if respond_to?("#{key}=")
    end
  end
end

# การใช้งาน
class Product
  include Attributable
  
  attributes :name, :price, :stock, :active
  
  def initialize(attrs = {})
    super
    @active = true if @active.nil?
  end
  
  def expensive?
    price.to_f > 1000
  end
end

product = Product.new(name: 'Laptop', price: 25000, stock: 10)
puts product.to_h
# => {:name=>"Laptop", :price=>25000, :stock=>10, :active=>true}
puts product.expensive?  # => true
puts product.active?     # => true
```

---

## 3. External DSL Basics

### 3.1 Simple Parser

```ruby
# สร้าง simple DSL parser สำหรับ query language
class QueryParser
  # DSL ที่เราต้องการ parse:
  # SELECT name, email FROM users WHERE role = 'admin' AND active = true LIMIT 10
  
  KEYWORDS = %w[SELECT FROM WHERE AND OR LIMIT ORDER BY ASC DESC].freeze
  
  def parse(query_string)
    tokens = tokenize(query_string)
    build_ast(tokens)
  end
  
  def tokenize(string)
    string.scan(/\w+|'[^']*'|=|>|<|>=|<=|!=|,|\*/).map do |token|
      if KEYWORDS.include?(token.upcase)
        { type: :keyword, value: token.upcase }
      elsif token.start_with?("'") && token.end_with?("'")
        { type: :string, value: token[1..-2] }
      elsif token.match?(/^\d+$/)
        { type: :integer, value: token.to_i }
      elsif token.match?(/^\d+\.\d+$/)
        { type: :float, value: token.to_f }
      elsif token == ','
        { type: :comma, value: ',' }
      elsif ['=', '>', '<', '>=', '<=', '!='].include?(token)
        { type: :operator, value: token }
      else
        { type: :identifier, value: token }
      end
    end
  end
  
  def build_ast(tokens)
    ast = {}
    i = 0
    
    while i < tokens.length
      token = tokens[i]
      
      case token[:value]
      when 'SELECT'
        i += 1
        fields = []
        while i < tokens.length && tokens[i][:type] != :keyword
          fields << tokens[i][:value] unless tokens[i][:type] == :comma
          i += 1
        end
        ast[:select] = fields
        next
        
      when 'FROM'
        i += 1
        ast[:from] = tokens[i][:value]
        
      when 'WHERE'
        i += 1
        conditions = []
        
        while i < tokens.length && !['LIMIT', 'ORDER'].include?(tokens[i][:value])
          if tokens[i][:type] == :identifier
            field = tokens[i][:value]
            i += 1
            operator = tokens[i][:value]
            i += 1
            value = tokens[i][:value]
            conditions << { field: field, operator: operator, value: value }
          end
          i += 1
          next if tokens[i]&.dig(:value) == 'AND' || tokens[i]&.dig(:value) == 'OR'
        end
        
        ast[:where] = conditions
        next
        
      when 'LIMIT'
        i += 1
        ast[:limit] = tokens[i][:value]
      end
      
      i += 1
    end
    
    ast
  end
  
  def to_active_record(model, ast)
    scope = model.all
    
    # Fields
    if ast[:select] && ast[:select] != ['*']
      scope = scope.select(ast[:select].join(', '))
    end
    
    # Conditions
    if ast[:where]
      ast[:where].each do |condition|
        scope = case condition[:operator]
                when '=' then scope.where("#{condition[:field]} = ?", condition[:value])
                when '!=' then scope.where("#{condition[:field]} != ?", condition[:value])
                when '>' then scope.where("#{condition[:field]} > ?", condition[:value])
                when '<' then scope.where("#{condition[:field]} < ?", condition[:value])
                end
      end
    end
    
    # Limit
    scope = scope.limit(ast[:limit]) if ast[:limit]
    
    scope
  end
end

# การใช้งาน
parser = QueryParser.new
ast = parser.parse("SELECT name, email FROM users WHERE role = 'admin' LIMIT 10")
puts ast.inspect
# => {:select=>["name", "email"], :from=>"users", :where=>[{:field=>"role", :operator=>"=", :value=>"admin"}], :limit=>10}

scope = parser.to_active_record(User, ast)
puts scope.to_sql
```

### 3.2 Template DSL

```ruby
# Simple template DSL
class TemplateDSL
  def initialize
    @parts = []
    @variables = {}
  end
  
  def self.define(&block)
    template = new
    template.instance_eval(&block)
    template
  end
  
  def text(content)
    @parts << [:text, content]
  end
  
  def var(name)
    @parts << [:variable, name]
  end
  
  def if_var(name, &block)
    @parts << [:conditional, name, block]
  end
  
  def loop_var(name, &block)
    @parts << [:loop, name, block]
  end
  
  def render(variables = {})
    @variables = variables
    
    @parts.map do |part|
      case part[0]
      when :text
        part[1]
      when :variable
        @variables[part[1]].to_s
      when :conditional
        if @variables[part[1]]
          inner = TemplateDSL.new
          inner.instance_eval(&part[2])
          inner.render(@variables)
        else
          ''
        end
      when :loop
        collection = @variables[part[1]] || []
        collection.map do |item|
          inner = TemplateDSL.new
          inner.instance_eval { @variables = { item: item } }
          inner.instance_eval(&part[2])
          inner.render(item: item)
        end.join
      end
    end.join
  end
end

# การใช้งาน
template = TemplateDSL.define do
  text "Hello, "
  var :name
  text "!\n"
  
  if_var(:premium) do
    text "You are a premium member.\n"
  end
  
  text "Your orders:\n"
  loop_var(:orders) do
    text "  - "
    var :item
    text "\n"
  end
end

puts template.render(
  name: 'John',
  premium: true,
  orders: ['Order #001', 'Order #002', 'Order #003']
)
```

---

## 4. Method Missing สำหรับ Fluent Interfaces

### 4.1 Fluent Interface / Method Chaining

```ruby
# Query Builder DSL
class QueryBuilder
  def initialize(model)
    @model = model
    @conditions = []
    @order_clauses = []
    @limit_value = nil
    @offset_value = nil
    @includes_list = []
    @select_fields = ['*']
  end
  
  def select(*fields)
    @select_fields = fields
    self
  end
  
  def where(condition = nil, **kwargs, &block)
    if condition
      @conditions << condition
    elsif kwargs.any?
      kwargs.each do |key, value|
        if value.is_a?(Array)
          @conditions << ["#{key} IN (?)", value]
        else
          @conditions << { key => value }
        end
      end
    elsif block_given?
      @conditions << block
    end
    self
  end
  
  def and(condition)
    where(condition)
  end
  
  def or_where(condition)
    @conditions << [:or, condition]
    self
  end
  
  def order(clause)
    @order_clauses << clause
    self
  end
  
  def limit(n)
    @limit_value = n
    self
  end
  
  def offset(n)
    @offset_value = n
    self
  end
  
  def includes(*associations)
    @includes_list.concat(associations)
    self
  end
  
  def page(n, per_page: 25)
    @limit_value = per_page
    @offset_value = (n.to_i - 1) * per_page
    self
  end
  
  def build
    scope = @model.select(@select_fields.join(', '))
    
    @conditions.each do |condition|
      case condition
      when Hash
        scope = scope.where(condition)
      when Array
        if condition[0] == :or
          scope = scope.or(scope.where(condition[1]))
        else
          scope = scope.where(*condition)
        end
      when String
        scope = scope.where(condition)
      end
    end
    
    scope = scope.order(@order_clauses.join(', ')) if @order_clauses.any?
    scope = scope.limit(@limit_value) if @limit_value
    scope = scope.offset(@offset_value) if @offset_value
    scope = scope.includes(@includes_list) if @includes_list.any?
    
    scope
  end
  
  def to_a
    build.to_a
  end
  
  def each(&block)
    to_a.each(&block)
  end
  
  # Delegate missing methods to built scope
  def method_missing(name, *args, &block)
    result = build
    if result.respond_to?(name)
      result.send(name, *args, &block)
    else
      super
    end
  end
  
  def respond_to_missing?(name, include_private = false)
    build.respond_to?(name, include_private) || super
  end
end

# Module extension สำหรับ models
module Queryable
  def self.included(base)
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def query
      QueryBuilder.new(self)
    end
  end
end

class User < ApplicationRecord
  include Queryable
end

# Fluent interface ที่สวยงาม
users = User.query
  .select('id', 'name', 'email')
  .where(active: true)
  .where('created_at > ?', 30.days.ago)
  .order('name ASC')
  .includes(:posts, :profile)
  .page(2, per_page: 20)
  .to_a
```

### 4.2 Method Missing สำหรับ Dynamic Finders

```ruby
# สร้าง find_by_xxx dynamic methods
class DynamicFinder
  def self.included(base)
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def method_missing(method_name, *args, &block)
      # find_by_name, find_by_email, find_by_name_and_email
      if method_name.to_s.start_with?('find_by_')
        attributes = method_name.to_s.sub('find_by_', '').split('_and_')
        
        if attributes.all? { |a| column_names.include?(a) }
          conditions = attributes.zip(args).to_h
          find_by(conditions)
        else
          super
        end
      elsif method_name.to_s.start_with?('find_all_by_')
        attributes = method_name.to_s.sub('find_all_by_', '').split('_and_')
        conditions = attributes.zip(args).to_h
        where(conditions)
      else
        super
      end
    end
    
    def respond_to_missing?(method_name, include_private = false)
      if method_name.to_s.start_with?('find_by_') || 
         method_name.to_s.start_with?('find_all_by_')
        attributes = method_name.to_s
          .sub(/find_(all_)?by_/, '')
          .split('_and_')
        attributes.all? { |a| column_names.include?(a) }
      else
        super
      end
    end
  end
end
```

### 4.3 Proxy Object Pattern

```ruby
# Proxy สำหรับ lazy loading และ transformation
class LazyProxy
  def initialize(&loader)
    @loader = loader
    @loaded = false
    @value = nil
  end
  
  def method_missing(name, *args, &block)
    load_value! unless @loaded
    @value.send(name, *args, &block)
  end
  
  def respond_to_missing?(name, include_private = false)
    load_value! unless @loaded
    @value.respond_to?(name, include_private)
  end
  
  def loaded?
    @loaded
  end
  
  private
  
  def load_value!
    @value = @loader.call
    @loaded = true
  end
end

# การใช้งาน
class UserProfile
  def current_user_profile
    @profile ||= LazyProxy.new do
      UserProfile.find_by(user: current_user)  # ดึง DB เฉพาะเมื่อใช้งาน
    end
  end
end
```

---

## 5. สร้าง DSL จริง

### 5.1 Config DSL

```ruby
# lib/config_dsl.rb
class ConfigDSL
  class ValidationError < StandardError; end
  
  attr_reader :data
  
  def initialize
    @data = {}
    @validations = {}
    @required_keys = []
    @defaults = {}
  end
  
  def self.define(&block)
    config = new
    config.instance_eval(&block)
    config.validate!
    config
  end
  
  # DSL Methods
  def set(key, value)
    @data[key.to_sym] = value
    self
  end
  
  def get(key)
    @data[key.to_sym] || @defaults[key.to_sym]
  end
  
  def required(*keys)
    @required_keys.concat(keys.map(&:to_sym))
  end
  
  def default(key, value)
    @defaults[key.to_sym] = value
  end
  
  def validate(key, &validator)
    @validations[key.to_sym] = validator
  end
  
  # Validation
  def validate!
    errors = []
    
    @required_keys.each do |key|
      errors << "#{key} is required" unless @data.key?(key)
    end
    
    @validations.each do |key, validator|
      value = get(key)
      next unless value
      
      unless validator.call(value)
        errors << "#{key} failed validation"
      end
    end
    
    raise ValidationError, errors.join(', ') if errors.any?
    
    self
  end
  
  def method_missing(name, value = nil)
    if name.to_s.end_with?('=')
      set(name.to_s.chomp('='), value)
    elsif @data.key?(name.to_sym) || @defaults.key?(name.to_sym)
      get(name)
    elsif value
      set(name, value)
    else
      get(name)
    end
  end
  
  def to_h
    @defaults.merge(@data)
  end
end

# Database Configuration DSL
class DatabaseConfig < ConfigDSL
  def initialize
    super
    default :adapter, 'postgresql'
    default :port, 5432
    default :pool, 5
    default :timeout, 5000
    
    required :database
    required :username
    
    validate(:port) { |v| v.is_a?(Integer) && v > 0 && v < 65536 }
    validate(:pool) { |v| v.is_a?(Integer) && v > 0 }
  end
  
  def self.configure(&block)
    define(&block)
  end
end

# การใช้งาน
db_config = DatabaseConfig.configure do
  adapter 'postgresql'
  host 'db.example.com'
  port 5432
  database 'myapp_production'
  username 'app_user'
  password ENV['DB_PASSWORD']
  pool 10
end

puts db_config.to_h
```

### 5.2 Query DSL

```ruby
# lib/query_dsl.rb - Expressibe Query Builder

class QueryDSL
  class << self
    def from(model)
      new(model)
    end
  end
  
  def initialize(model)
    @model = model
    @filters = []
    @sorts = []
    @joins = []
    @includes = []
    @pagination = nil
    @aggregations = {}
  end
  
  # Filtering
  def filter_by(attribute, operator = :eq, value)
    @filters << { attribute: attribute, operator: operator, value: value }
    self
  end
  
  # Shorthand methods
  def equals(attribute, value)
    filter_by(attribute, :eq, value)
  end
  
  def not_equals(attribute, value)
    filter_by(attribute, :neq, value)
  end
  
  def greater_than(attribute, value)
    filter_by(attribute, :gt, value)
  end
  
  def less_than(attribute, value)
    filter_by(attribute, :lt, value)
  end
  
  def between(attribute, min, max)
    filter_by(attribute, :between, [min, max])
  end
  
  def contains(attribute, value)
    filter_by(attribute, :like, "%#{value}%")
  end
  
  def starts_with(attribute, value)
    filter_by(attribute, :like, "#{value}%")
  end
  
  def in_list(attribute, values)
    filter_by(attribute, :in, values)
  end
  
  def null(attribute)
    filter_by(attribute, :null, nil)
  end
  
  def not_null(attribute)
    filter_by(attribute, :not_null, nil)
  end
  
  # Sorting
  def sort_by(attribute, direction = :asc)
    @sorts << { attribute: attribute, direction: direction }
    self
  end
  
  def sort_asc(attribute)
    sort_by(attribute, :asc)
  end
  
  def sort_desc(attribute)
    sort_by(attribute, :desc)
  end
  
  # Pagination
  def paginate(page: 1, per_page: 25)
    @pagination = { page: page.to_i, per_page: per_page.to_i }
    self
  end
  
  # Joins
  def join(*tables)
    @joins.concat(tables)
    self
  end
  
  def eager_load(*associations)
    @includes.concat(associations)
    self
  end
  
  # Aggregations
  def count_by(attribute)
    @aggregations[:count] = attribute
    self
  end
  
  def sum_by(attribute)
    @aggregations[:sum] = attribute
    self
  end
  
  def average(attribute)
    @aggregations[:average] = attribute
    self
  end
  
  # Build and execute
  def build
    scope = @model.all
    
    @filters.each do |filter|
      scope = apply_filter(scope, filter)
    end
    
    @sorts.each do |sort|
      scope = scope.order(sort[:attribute] => sort[:direction])
    end
    
    scope = scope.joins(@joins) if @joins.any?
    scope = scope.includes(@includes) if @includes.any?
    
    if @pagination
      scope = scope
        .limit(@pagination[:per_page])
        .offset((@pagination[:page] - 1) * @pagination[:per_page])
    end
    
    scope
  end
  
  def to_sql
    build.to_sql
  end
  
  def execute
    scope = build
    
    if @aggregations.any?
      apply_aggregations(scope)
    else
      scope.to_a
    end
  end
  
  alias :results :execute
  alias :all :execute
  
  def first
    build.first
  end
  
  def count
    build.count
  end
  
  def each(&block)
    execute.each(&block)
  end
  
  private
  
  def apply_filter(scope, filter)
    column = "#{@model.table_name}.#{filter[:attribute]}"
    
    case filter[:operator]
    when :eq
      scope.where("#{column} = ?", filter[:value])
    when :neq
      scope.where("#{column} != ?", filter[:value])
    when :gt
      scope.where("#{column} > ?", filter[:value])
    when :lt
      scope.where("#{column} < ?", filter[:value])
    when :gte
      scope.where("#{column} >= ?", filter[:value])
    when :lte
      scope.where("#{column} <= ?", filter[:value])
    when :between
      scope.where("#{column} BETWEEN ? AND ?", *filter[:value])
    when :like
      scope.where("#{column} LIKE ?", filter[:value])
    when :in
      scope.where("#{column} IN (?)", filter[:value])
    when :null
      scope.where("#{column} IS NULL")
    when :not_null
      scope.where("#{column} IS NOT NULL")
    else
      scope
    end
  end
  
  def apply_aggregations(scope)
    result = {}
    @aggregations.each do |agg, attribute|
      result[agg] = scope.send(agg, attribute)
    end
    result
  end
end

# การใช้งาน
users = QueryDSL.from(User)
  .equals(:active, true)
  .not_null(:email)
  .greater_than(:age, 18)
  .between(:created_at, 1.year.ago, Time.current)
  .contains(:name, 'John')
  .sort_desc(:created_at)
  .sort_asc(:name)
  .eager_load(:profile, :orders)
  .paginate(page: 1, per_page: 20)
  .results

# Query analytics
stats = QueryDSL.from(Order)
  .equals(:status, 'completed')
  .greater_than(:total, 1000)
  .sum_by(:total)
  .execute
```

### 5.3 Test DSL

```ruby
# lib/test_dsl.rb - Simple testing DSL

class TestDSL
  class TestFailure < StandardError; end
  
  attr_reader :name, :failures, :passed_count
  
  def initialize(name, &block)
    @name = name
    @failures = []
    @passed_count = 0
    @tests = []
    instance_eval(&block) if block_given?
  end
  
  def self.describe(name, &block)
    new(name, &block).run
  end
  
  def it(description, &block)
    @tests << { description: description, block: block }
  end
  
  def before_each(&block)
    @before_each = block
  end
  
  def after_each(&block)
    @after_each = block
  end
  
  def run
    puts "\n#{@name}"
    
    @tests.each do |test|
      run_test(test)
    end
    
    print_summary
    self
  end
  
  def passed?
    @failures.empty?
  end
  
  private
  
  def run_test(test)
    begin
      @before_each&.call
      test[:block].call
      @passed_count += 1
      print '.'
    rescue TestFailure => e
      @failures << { description: test[:description], error: e.message }
      print 'F'
    rescue => e
      @failures << { description: test[:description], error: "Unexpected error: #{e.message}" }
      print 'E'
    ensure
      @after_each&.call
    end
  end
  
  def print_summary
    puts "\n\n#{@passed_count} passed, #{@failures.count} failed"
    
    @failures.each_with_index do |failure, i|
      puts "\n#{i + 1}) #{failure[:description]}"
      puts "   #{failure[:error]}"
    end
  end
end

module Assertions
  def assert(condition, message = 'Assertion failed')
    raise TestDSL::TestFailure, message unless condition
  end
  
  def assert_equal(expected, actual, message = nil)
    msg = message || "Expected #{expected.inspect}, got #{actual.inspect}"
    raise TestDSL::TestFailure, msg unless expected == actual
  end
  
  def assert_nil(value, message = nil)
    msg = message || "Expected nil, got #{value.inspect}"
    raise TestDSL::TestFailure, msg unless value.nil?
  end
  
  def assert_raises(exception_class, &block)
    block.call
    raise TestDSL::TestFailure, "Expected #{exception_class} to be raised"
  rescue exception_class
    # Test passed
  end
end

# การใช้งาน
include Assertions

TestDSL.describe 'Calculator' do
  before_each do
    @calc = Calculator.new
  end
  
  it 'adds two numbers' do
    result = @calc.add(2, 3)
    assert_equal 5, result
  end
  
  it 'multiplies correctly' do
    result = @calc.multiply(4, 5)
    assert_equal 20, result
  end
  
  it 'raises error on division by zero' do
    assert_raises(ZeroDivisionError) do
      @calc.divide(10, 0)
    end
  end
end
```

---

## 6. Practical DSL Examples

### 6.1 Rake DSL ที่สมบูรณ์

```ruby
# Rakefile
require 'rake'

namespace :app do
  desc 'Setup development environment'
  task :setup do
    Rake::Task['db:setup'].invoke
    Rake::Task['assets:precompile'].invoke
    Rake::Task['cache:clear'].invoke
    puts 'Setup complete!'
  end
  
  namespace :deploy do
    desc 'Deploy to staging'
    task :staging do
      sh 'kamal deploy --config-file config/deploy.staging.yml'
    end
    
    desc 'Deploy to production'
    task :production => [:test, :check_clean_git] do
      if ENV['CONFIRM_PRODUCTION'] == 'yes'
        sh 'kamal deploy --config-file config/deploy.production.yml'
      else
        puts 'Set CONFIRM_PRODUCTION=yes to deploy to production'
        exit 1
      end
    end
  end
  
  task :check_clean_git do
    status = `git status --porcelain`
    unless status.empty?
      puts 'Git working directory is not clean!'
      puts status
      exit 1
    end
  end
  
  task :test do
    sh 'bundle exec rspec'
    sh 'bundle exec rubocop'
  end
end

# Custom Rake task ที่อ่านง่ายด้วย DSL
task :daily_report do
  puts "Generating daily report for #{Date.today}..."
  
  stats = {
    new_users: User.where('created_at >= ?', 1.day.ago).count,
    new_orders: Order.where('created_at >= ?', 1.day.ago).count,
    revenue: Order.completed.where('created_at >= ?', 1.day.ago).sum(:total)
  }
  
  DailyReportMailer.send_report(stats, to: 'management@example.com').deliver_now
  puts "Report sent!"
end
```

### 6.2 Routes DSL ของ Rails

```ruby
# config/routes.rb - Advanced routing DSL
Rails.application.routes.draw do
  # Namespace grouping
  namespace :api do
    namespace :v1 do
      # Standard REST resources
      resources :users do
        member do
          post :activate
          post :deactivate
          get :activity
        end
        
        collection do
          get :search
          post :bulk_create
          delete :bulk_destroy
        end
        
        # Nested resources
        resources :posts, shallow: true do
          resources :comments, only: [:index, :create]
        end
      end
      
      # Custom routes
      scope :auth do
        post :login, to: 'sessions#create'
        delete :logout, to: 'sessions#destroy'
        post :refresh, to: 'sessions#refresh'
        post :register, to: 'registrations#create'
        post :forgot_password, to: 'passwords#create'
        put :reset_password, to: 'passwords#update'
      end
    end
  end
  
  # Constraints
  constraints(lambda { |req| req.env['HTTP_X_API_VERSION'] == 'v2' }) do
    namespace :api do
      namespace :v2 do
        resources :users
      end
    end
  end
  
  # Redirect
  get '/old-path', to: redirect('/new-path', status: 301)
  
  # Health check
  get '/health', to: proc { [200, {}, ['OK']] }
  
  root 'home#index'
end
```

### 6.3 RSpec-like DSL

```ruby
# lib/mini_spec.rb - Mini RSpec-like DSL
module MiniSpec
  class Example
    attr_reader :description, :result, :error
    
    def initialize(description, &block)
      @description = description
      @block = block
    end
    
    def run(context)
      context.instance_eval(&@block)
      @result = :passed
    rescue ExpectationError => e
      @result = :failed
      @error = e.message
    rescue => e
      @result = :error
      @error = "#{e.class}: #{e.message}"
    end
  end
  
  class ExpectationError < StandardError; end
  
  class Expectation
    def initialize(value)
      @value = value
    end
    
    def to(matcher)
      unless matcher.matches?(@value)
        raise ExpectationError, matcher.failure_message(@value)
      end
    end
    
    def not_to(matcher)
      if matcher.matches?(@value)
        raise ExpectationError, matcher.failure_message_for_should_not(@value)
      end
    end
  end
  
  module Matchers
    class Equal
      def initialize(expected)
        @expected = expected
      end
      
      def matches?(actual)
        @expected == actual
      end
      
      def failure_message(actual)
        "Expected #{@expected.inspect} but got #{actual.inspect}"
      end
      
      def failure_message_for_should_not(actual)
        "Expected not #{@expected.inspect}"
      end
    end
    
    class BeTrue
      def matches?(actual)
        actual == true
      end
      
      def failure_message(actual)
        "Expected true but got #{actual.inspect}"
      end
      
      def failure_message_for_should_not(_actual)
        "Expected not to be true"
      end
    end
    
    def eq(expected)
      Equal.new(expected)
    end
    
    def be_true
      BeTrue.new
    end
  end
  
  class ExampleGroup
    include Matchers
    
    attr_reader :examples, :before_hooks, :after_hooks
    
    def initialize(description)
      @description = description
      @examples = []
      @before_hooks = []
      @after_hooks = []
      @nested_groups = []
    end
    
    def it(description, &block)
      @examples << Example.new(description, &block)
    end
    
    def before(&block)
      @before_hooks << block
    end
    
    def after(&block)
      @after_hooks << block
    end
    
    def context(description, &block)
      group = ExampleGroup.new(description)
      group.instance_eval(&block)
      @nested_groups << group
    end
    
    def expect(value)
      Expectation.new(value)
    end
    
    def run
      results = { passed: 0, failed: 0, errors: 0, failures: [] }
      
      puts "\n#{@description}"
      
      @examples.each do |example|
        context = dup  # Fresh context สำหรับแต่ละ test
        @before_hooks.each { |hook| context.instance_eval(&hook) }
        
        example.run(context)
        
        case example.result
        when :passed
          results[:passed] += 1
          print '.'
        when :failed
          results[:failed] += 1
          results[:failures] << example
          print 'F'
        when :error
          results[:errors] += 1
          results[:failures] << example
          print 'E'
        end
        
        @after_hooks.each { |hook| context.instance_eval(&hook) }
      end
      
      @nested_groups.each do |group|
        nested_results = group.run
        results[:passed] += nested_results[:passed]
        results[:failed] += nested_results[:failed]
        results[:failures].concat(nested_results[:failures])
      end
      
      results
    end
  end
  
  def self.describe(description, &block)
    group = ExampleGroup.new(description)
    group.instance_eval(&block)
    group.run
  end
end

# การใช้งาน
results = MiniSpec.describe 'User' do
  before do
    @user = OpenStruct.new(name: 'John', email: 'john@example.com', active: true)
  end
  
  it 'has a name' do
    expect(@user.name).to eq('John')
  end
  
  it 'has an email' do
    expect(@user.email).to eq('john@example.com')
  end
  
  it 'is active' do
    expect(@user.active).to be_true
  end
  
  context 'when deactivated' do
    before do
      @user.active = false
    end
    
    it 'is not active' do
      expect(@user.active).not_to be_true
    end
  end
end
```

---

## แบบฝึกหัด (20 ข้อพร้อมเฉลย)

### ข้อ 1: Config DSL
**คำถาม:** สร้าง DSL สำหรับ configure email service ที่รองรับ multiple providers

**เฉลย:**
```ruby
class EmailServiceConfig
  attr_reader :provider, :settings
  
  def initialize
    @provider = :smtp
    @settings = {}
  end
  
  def self.configure(&block)
    config = new
    config.instance_eval(&block)
    config
  end
  
  def use_sendgrid(api_key:)
    @provider = :sendgrid
    @settings = { api_key: api_key }
  end
  
  def use_ses(region:, access_key:, secret_key:)
    @provider = :ses
    @settings = { region: region, access_key: access_key, secret_key: secret_key }
  end
  
  def use_smtp(&block)
    @provider = :smtp
    smtp_config = OpenStruct.new
    smtp_config.instance_eval(&block) if block
    @settings = smtp_config.to_h
  end
  
  def from_email(email)
    @settings[:from] = email
  end
  
  def reply_to(email)
    @settings[:reply_to] = email
  end
end

email_config = EmailServiceConfig.configure do
  use_sendgrid api_key: ENV['SENDGRID_API_KEY']
  from_email 'noreply@example.com'
  reply_to 'support@example.com'
end
```

### ข้อ 2: Validation DSL
**คำถาม:** สร้าง custom validation DSL สำหรับ validate data objects

**เฉลย:**
```ruby
module ValidatableDSL
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@validations, [])
  end
  
  module ClassMethods
    def validates(attribute, **rules)
      @validations << { attribute: attribute, rules: rules }
    end
    
    def validations
      @validations
    end
  end
  
  def valid?
    @errors = []
    
    self.class.validations.each do |validation|
      value = send(validation[:attribute])
      
      if validation[:rules][:presence] && value.blank?
        @errors << "#{validation[:attribute]} can't be blank"
      end
      
      if validation[:rules][:length]
        len_rules = validation[:rules][:length]
        if len_rules[:minimum] && value.to_s.length < len_rules[:minimum]
          @errors << "#{validation[:attribute]} is too short"
        end
        if len_rules[:maximum] && value.to_s.length > len_rules[:maximum]
          @errors << "#{validation[:attribute]} is too long"
        end
      end
      
      if validation[:rules][:format]
        unless value.to_s.match?(validation[:rules][:format][:with])
          @errors << validation[:rules][:format][:message] || "#{validation[:attribute]} is invalid"
        end
      end
      
      if validation[:rules][:inclusion]
        unless validation[:rules][:inclusion][:in].include?(value)
          @errors << "#{validation[:attribute]} is not included in list"
        end
      end
    end
    
    @errors.empty?
  end
  
  def errors
    @errors || []
  end
end

class UserData
  include ValidatableDSL
  
  attr_accessor :name, :email, :role, :age
  
  validates :name, presence: true, length: { minimum: 2, maximum: 100 }
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP, message: 'is invalid' }
  validates :role, inclusion: { in: %w[admin user guest] }
end
```

### ข้อ 3: Pipeline DSL
**คำถาม:** สร้าง data processing pipeline DSL

**เฉลย:**
```ruby
class Pipeline
  def initialize
    @steps = []
  end
  
  def self.define(&block)
    pipeline = new
    pipeline.instance_eval(&block)
    pipeline
  end
  
  def step(name, &processor)
    @steps << { name: name, processor: processor }
    self
  end
  
  def filter(&condition)
    @steps << { name: :filter, processor: condition, type: :filter }
    self
  end
  
  def transform(&transformer)
    @steps << { name: :transform, processor: transformer }
    self
  end
  
  def process(data)
    @steps.reduce(data) do |current, step|
      if step[:type] == :filter
        current.select(&step[:processor])
      else
        current.map { |item| step[:processor].call(item) }
      end
    end
  end
end

pipeline = Pipeline.define do
  step(:parse) { |record| record.merge(parsed: true) }
  filter { |record| record[:age].to_i >= 18 }
  transform { |record| record.slice(:name, :email) }
  step(:normalize) { |record| record.transform_values(&:to_s) }
end

result = pipeline.process(user_data)
```

### ข้อ 4: Router DSL
**คำถาม:** สร้าง simple URL router DSL

**เฉลย:**
```ruby
class Router
  Route = Struct.new(:method, :pattern, :handler)
  
  def initialize
    @routes = []
  end
  
  def self.define(&block)
    router = new
    router.instance_eval(&block)
    router
  end
  
  def get(path, &handler)
    add_route('GET', path, handler)
  end
  
  def post(path, &handler)
    add_route('POST', path, handler)
  end
  
  def put(path, &handler)
    add_route('PUT', path, handler)
  end
  
  def delete(path, &handler)
    add_route('DELETE', path, handler)
  end
  
  def match(method, path)
    @routes.find do |route|
      route.method == method.upcase &&
        path_matches?(route.pattern, path)
    end
  end
  
  private
  
  def add_route(method, path, handler)
    @routes << Route.new(method, path, handler)
  end
  
  def path_matches?(pattern, path)
    regex = pattern.gsub(/:(\w+)/, '(?<\1>[^/]+)')
    path.match?(/\A#{regex}\z/)
  end
end
```

### ข้อ 5: State Machine DSL
**คำถาม:** สร้าง state machine DSL สำหรับ Order lifecycle

**เฉลย:**
```ruby
module StateMachineDSL
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@states, {})
    base.instance_variable_set(:@transitions, [])
  end
  
  module ClassMethods
    def state(name, initial: false, &block)
      @states[name] = { initial: initial, callbacks: block }
      define_method("#{name}?") { current_state == name }
    end
    
    def transition(from:, to:, on:, &guard)
      @transitions << { from: [from].flatten, to: to, event: on, guard: guard }
      
      define_method(on) do
        fire_event(on)
      end
    end
    
    def states; @states; end
    def transitions; @transitions; end
    
    def initial_state
      @states.find { |_, v| v[:initial] }&.first
    end
  end
  
  def initialize(*)
    super
    @current_state = self.class.initial_state
  end
  
  def current_state
    @current_state
  end
  
  private
  
  def fire_event(event)
    transition = self.class.transitions.find do |t|
      t[:event] == event && t[:from].include?(@current_state)
    end
    
    raise "Invalid transition: #{event} from #{@current_state}" unless transition
    
    if transition[:guard] && !instance_eval(&transition[:guard])
      raise "Guard failed for #{event}"
    end
    
    @current_state = transition[:to]
  end
end

class Order
  include StateMachineDSL
  
  state :pending, initial: true
  state :confirmed
  state :paid
  state :shipped
  state :delivered
  state :cancelled
  
  transition from: :pending, to: :confirmed, on: :confirm
  transition from: :confirmed, to: :paid, on: :pay
  transition from: :paid, to: :shipped, on: :ship
  transition from: :shipped, to: :delivered, on: :deliver
  transition from: [:pending, :confirmed], to: :cancelled, on: :cancel
end
```

### ข้อ 6-20 (สรุปเฉลย):

**ข้อ 6: Form Builder DSL**
```ruby
class FormBuilder
  def initialize(model)
    @model = model
    @fields = []
  end
  
  def self.for(model, &block)
    builder = new(model)
    builder.instance_eval(&block)
    builder.render
  end
  
  def text_field(attribute, **options)
    @fields << { type: :text, attribute: attribute, options: options }
  end
  
  def select_field(attribute, choices, **options)
    @fields << { type: :select, attribute: attribute, choices: choices, options: options }
  end
  
  def render
    @fields.map { |f| render_field(f) }.join("\n")
  end
end
```

**ข้อ 7: Decorator DSL**
```ruby
module Decorator
  def self.decorate(object, &block)
    decorator = Module.new
    decorator.module_eval(&block)
    object.extend(decorator)
    object
  end
end

user = Decorator.decorate(User.new) do
  def greeting
    "Hello, #{name}!"
  end
  
  def formatted_email
    "<#{email}>"
  end
end
```

**ข้อ 8: Notification DSL**
```ruby
class NotificationConfig
  def self.define(&block)
    config = new
    config.instance_eval(&block)
    config
  end
  
  def on(event, &handler)
    @handlers ||= {}
    @handlers[event] = handler
  end
  
  def send_email(options = {})
    @email_config = options
  end
  
  def send_sms(options = {})
    @sms_config = options
  end
  
  def push_notification(options = {})
    @push_config = options
  end
end
```

**ข้อ 9: Serialization DSL**
```ruby
module SerializerDSL
  def self.included(base)
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def attribute(name, &transform)
      @attributes ||= {}
      @attributes[name] = transform || ->(obj) { obj.send(name) }
    end
    
    def serialize(object)
      @attributes.transform_values { |transform| transform.call(object) }
    end
  end
end
```

**ข้อ 10: Authorization DSL**
```ruby
class PolicyDSL
  def self.define(&block)
    new.tap { |p| p.instance_eval(&block) }
  end
  
  def can(action, resource, &condition)
    @rules ||= []
    @rules << { action: action, resource: resource, condition: condition }
  end
  
  def cannot(action, resource)
    @rules ||= []
    @rules << { action: action, resource: resource, condition: -> { false } }
  end
  
  def allowed?(user, action, resource)
    rule = @rules.find { |r| r[:action] == action && r[:resource] == resource.class }
    return false unless rule
    rule[:condition].call(user, resource)
  end
end
```

**ข้อ 11: Migration DSL**
```ruby
# Rails-like migration DSL
class SchemaDSL
  def create_table(name, **options, &block)
    table = TableBuilder.new(name, options)
    table.instance_eval(&block)
    execute_create(table)
  end
  
  def add_column(table, column, type, **options)
    execute_sql("ALTER TABLE #{table} ADD COLUMN #{column} #{type}")
  end
  
  def add_index(table, columns, **options)
    cols = [columns].flatten.join(', ')
    unique = options[:unique] ? 'UNIQUE' : ''
    execute_sql("CREATE #{unique} INDEX ON #{table}(#{cols})")
  end
end
```

**ข้อ 12: Assertion DSL**
```ruby
module AssertionDSL
  def assert_that(value)
    AssertionProxy.new(value)
  end
  
  class AssertionProxy
    def initialize(value)
      @value = value
    end
    
    def equals(expected)
      raise AssertionError unless @value == expected
      self
    end
    
    def is_greater_than(n)
      raise AssertionError unless @value > n
      self
    end
    
    def contains(item)
      raise AssertionError unless @value.include?(item)
      self
    end
  end
end
```

**ข้อ 13: Workflow DSL**
```ruby
class WorkflowDSL
  def self.define(name, &block)
    workflow = new(name)
    workflow.instance_eval(&block)
    workflow
  end
  
  def step(name, **options, &action)
    @steps ||= []
    @steps << { name: name, action: action, on_failure: options[:on_failure] }
  end
  
  def parallel(*step_names, &block)
    @steps << { type: :parallel, steps: step_names }
  end
  
  def run(context = {})
    @steps.each { |step| execute_step(step, context) }
  end
end
```

**ข้อ 14: Report DSL**
```ruby
class ReportDSL
  def self.define(&block)
    new.tap { |r| r.instance_eval(&block) }
  end
  
  def title(text)
    @title = text
  end
  
  def section(name, &block)
    @sections ||= []
    section = SectionBuilder.new(name)
    section.instance_eval(&block)
    @sections << section
  end
  
  def chart(type, data:, **options)
    @charts ||= []
    @charts << { type: type, data: data, options: options }
  end
  
  def render_pdf
    # PDF generation
  end
end
```

**ข้อ 15: Event Handler DSL**
```ruby
class EventSystem
  def self.on(event, &handler)
    @handlers ||= Hash.new { |h, k| h[k] = [] }
    @handlers[event] << handler
  end
  
  def self.emit(event, payload = {})
    (@handlers[event] || []).each { |h| h.call(payload) }
  end
  
  def self.once(event, &handler)
    wrapper = nil
    wrapper = ->(payload) {
      handler.call(payload)
      @handlers[event].delete(wrapper)
    }
    on(event, &wrapper)
  end
end

EventSystem.on(:user_created) { |u| WelcomeEmailJob.perform_later(u[:id]) }
EventSystem.on(:user_created) { |u| Analytics.track('user_created', u) }
EventSystem.emit(:user_created, { id: 123, email: 'test@example.com' })
```

**ข้อ 16: API Client DSL**
```ruby
class ApiClientDSL
  def self.configure(&block)
    client = new
    client.instance_eval(&block)
    client
  end
  
  def base_url(url)
    @base_url = url
  end
  
  def header(name, value)
    @headers ||= {}
    @headers[name] = value
  end
  
  def timeout(seconds)
    @timeout = seconds
  end
  
  def retry_on(*exceptions, max: 3, delay: 1)
    @retry_config = { exceptions: exceptions, max: max, delay: delay }
  end
  
  def get(path, params = {})
    make_request(:get, path, params: params)
  end
  
  def post(path, body = {})
    make_request(:post, path, body: body)
  end
end
```

**ข้อ 17: Background Job DSL**
```ruby
module JobDSL
  def self.included(base)
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def retry_on(*exceptions, wait: 30.seconds, attempts: 3)
      @retry_config = { exceptions: exceptions, wait: wait, attempts: attempts }
    end
    
    def timeout_after(duration)
      @timeout = duration
    end
    
    def unique(scope: :queue, timeout: 1.hour)
      @unique_config = { scope: scope, timeout: timeout }
    end
    
    def throttle(limit, period:)
      @throttle = { limit: limit, period: period }
    end
  end
end
```

**ข้อ 18: Caching DSL**
```ruby
module CachingDSL
  def self.included(base)
    base.extend(ClassMethods)
  end
  
  module ClassMethods
    def cached(method_name, expires_in: 1.hour, key: nil)
      original = instance_method(method_name)
      define_method(method_name) do |*args|
        cache_key = key || "#{self.class.name}:#{method_name}:#{args.hash}"
        Rails.cache.fetch(cache_key, expires_in: expires_in) do
          original.bind(self).call(*args)
        end
      end
    end
  end
end
```

**ข้อ 19: Feature Toggle DSL**
```ruby
class FeatureDSL
  def self.for(user)
    new(user)
  end
  
  def initialize(user)
    @user = user
  end
  
  def enabled?(feature, &fallback)
    if Flipper.enabled?(feature, @user)
      yield if block_given?
      true
    else
      fallback&.call
      false
    end
  end
  
  def with_feature(feature)
    Flipper.enabled?(feature, @user) ? yield : nil
  end
  
  def unless_feature(feature)
    Flipper.enabled?(feature, @user) ? nil : yield
  end
end

feature = FeatureDSL.for(current_user)
feature.with_feature(:new_dashboard) { render 'new_dashboard' }
feature.unless_feature(:new_dashboard) { render 'old_dashboard' }
```

**ข้อ 20: Complete DSL Application**
```ruby
# สร้าง mini ActiveRecord-like ORM ด้วย DSL

module MiniORM
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@columns, {})
    base.instance_variable_set(:@table_name, base.name.downcase.pluralize)
  end
  
  module ClassMethods
    def table(name)
      @table_name = name
    end
    
    def column(name, type, **options)
      @columns[name] = { type: type, options: options }
      
      define_method(name) { @attributes[name] }
      define_method("#{name}=") { |v| @attributes[name] = v }
    end
    
    def find(id)
      row = DB.query("SELECT * FROM #{@table_name} WHERE id = ?", id).first
      from_row(row)
    end
    
    def where(**conditions)
      sql = conditions.map { |k, _| "#{k} = ?" }.join(' AND ')
      values = conditions.values
      rows = DB.query("SELECT * FROM #{@table_name} WHERE #{sql}", *values)
      rows.map { |r| from_row(r) }
    end
    
    def from_row(row)
      obj = new
      row.each { |k, v| obj.instance_variable_get(:@attributes)[k.to_sym] = v }
      obj
    end
  end
  
  def initialize
    @attributes = {}
    @new_record = true
  end
  
  def save
    if @new_record
      result = DB.insert(self.class.instance_variable_get(:@table_name), @attributes)
      @attributes[:id] = result.last_id
      @new_record = false
    else
      DB.update(self.class.instance_variable_get(:@table_name), @attributes, id: @attributes[:id])
    end
    true
  end
  
  def valid?
    self.class.instance_variable_get(:@columns).all? do |name, config|
      value = @attributes[name]
      !config[:options][:required] || value.present?
    end
  end
end

class Person
  include MiniORM
  
  table :people
  column :name, :string, required: true
  column :email, :string, required: true, unique: true
  column :age, :integer
  column :created_at, :datetime
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **DSL Design Patterns** - หลักการออกแบบ DSL ที่อ่านง่าย
2. **instance_eval/module_eval** - เทคนิคหลักสำหรับ Internal DSL
3. **External DSL** - การสร้าง parser สำหรับ custom language
4. **Method Missing** - Fluent interfaces และ dynamic methods
5. **สร้าง DSL จริง** - Config, Query, Test DSL ที่ใช้งานได้จริง
6. **Practical Examples** - Rake, Routes, RSpec-like DSL

กุญแจสำคัญ: DSL ที่ดีต้องอ่านเข้าใจง่ายโดยไม่ต้องรู้ implementation details Ruby's flexibility ทำให้สร้าง expressive DSL ได้โดยใช้ instance_eval, method_missing, และ block passing
