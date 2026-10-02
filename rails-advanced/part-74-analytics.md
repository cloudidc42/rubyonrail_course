# Part 74: Analytics and Reporting

## Steps 1601-1620

---

## Step 1601: Tracking กับ Ahoy

Ahoy เป็น gem สำหรับ track visits, events, และ experiments

```ruby
# Gemfile
gem 'ahoy_matey'
gem 'chartkick'  # สำหรับ charts
gem 'groupdate'  # สำหรับ group by date

# Terminal
bundle install
rails generate ahoy:install
rails db:migrate

# config/initializers/ahoy.rb
class Ahoy::Store < Ahoy::DatabaseStore
end

Ahoy.api = true
Ahoy.geocode = false  # ปิด geocoding ถ้าไม่ต้องการ

# ใน ApplicationController
class ApplicationController < ActionController::Base
  before_action :track_visit
  
  private
  
  def track_visit
    ahoy.track_visit if ahoy.new_visit?
  end
end

# Track events ใน Controller
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])
    
    # Track product view
    ahoy.track 'Viewed Product', {
      product_id: @product.id,
      product_name: @product.name,
      category: @product.category.name,
      price: @product.price
    }
  end
end

class OrdersController < ApplicationController
  def create
    # หลังจาก order สำเร็จ
    ahoy.track 'Placed Order', {
      order_id: @order.id,
      total: @order.total,
      items_count: @order.items.count,
      coupon_used: @order.coupon_code.present?
    }
  end
end

# Model associations
class User < ApplicationRecord
  has_many :ahoy_visits, class_name: 'Ahoy::Visit', foreign_key: :user_id
  has_many :ahoy_events, class_name: 'Ahoy::Event', foreign_key: :user_id
end
```

### Query Ahoy Data

```ruby
# ยอดชมสินค้าที่มากที่สุด
top_products = Ahoy::Event
  .where(name: 'Viewed Product')
  .where('time >= ?', 30.days.ago)
  .group("properties->>'product_id'")
  .order('COUNT(*) DESC')
  .limit(10)
  .count

# Conversion funnel
viewed_products = Ahoy::Event.where(name: 'Viewed Product').distinct.count(:visit_id)
added_to_cart = Ahoy::Event.where(name: 'Added to Cart').distinct.count(:visit_id)
placed_order = Ahoy::Event.where(name: 'Placed Order').distinct.count(:visit_id)

conversion_rate = (placed_order.to_f / viewed_products * 100).round(2)

# Average session duration
avg_duration = Ahoy::Visit
  .where('started_at >= ?', 30.days.ago)
  .average("EXTRACT(EPOCH FROM (ended_at - started_at))")
```

---

## Step 1602: Dashboard กับ Chartkick

```ruby
# Gemfile
gem 'chartkick'
gem 'groupdate'

# app/views/admin/analytics/index.html.erb
<h1>Analytics Dashboard</h1>

<div class="charts-row">
  <!-- Revenue Chart -->
  <div class="chart-card">
    <h3>รายได้รายวัน (30 วันล่าสุด)</h3>
    <%= line_chart Order.paid
      .where('paid_at >= ?', 30.days.ago)
      .group_by_day(:paid_at)
      .sum(:total),
      prefix: '฿',
      thousands: ',',
      colors: ['#4CAF50'],
      curve: false %>
  </div>
  
  <!-- Orders by Status -->
  <div class="chart-card">
    <h3>Orders ตาม Status</h3>
    <%= pie_chart Order.group(:status).count,
      colors: {
        pending: '#FFC107',
        processing: '#2196F3',
        shipped: '#9C27B0',
        delivered: '#4CAF50',
        cancelled: '#F44336'
      } %>
  </div>
</div>

<div class="charts-row">
  <!-- New Users -->
  <div class="chart-card">
    <h3>สมาชิกใหม่รายสัปดาห์</h3>
    <%= bar_chart User
      .where('created_at >= ?', 12.weeks.ago)
      .group_by_week(:created_at)
      .count,
      colors: ['#2196F3'] %>
  </div>
  
  <!-- Top Products -->
  <div class="chart-card">
    <h3>สินค้าขายดี Top 10</h3>
    <%= bar_chart OrderItem
      .joins(:product)
      .group('products.name')
      .order('SUM(order_items.quantity) DESC')
      .limit(10)
      .sum(:quantity),
      suffix: ' ชิ้น' %>
  </div>
</div>

<!-- Revenue by Category -->
<div class="chart-card full-width">
  <h3>รายได้ตาม Category (เดือนนี้)</h3>
  <%= column_chart OrderItem
    .joins(product: :category)
    .where('order_items.created_at >= ?', Date.today.beginning_of_month)
    .group('categories.name')
    .sum('order_items.quantity * order_items.unit_price'),
    prefix: '฿' %>
</div>
```

### Chartkick กับ Custom Data

```ruby
# app/controllers/admin/analytics_controller.rb
class Admin::AnalyticsController < Admin::BaseController
  def index
    @revenue_by_day = Order.paid
      .where('paid_at >= ?', 30.days.ago)
      .group_by_day(:paid_at, range: 30.days.ago..Time.current)
      .sum(:total)
    
    @orders_by_hour = Order
      .where('created_at >= ?', 7.days.ago)
      .group_by_hour_of_day(:created_at)
      .count
    
    @revenue_comparison = {
      'เดือนนี้' => Order.paid.where('paid_at >= ?', Date.today.beginning_of_month).sum(:total),
      'เดือนที่แล้ว' => Order.paid.where(
        paid_at: 1.month.ago.beginning_of_month..1.month.ago.end_of_month
      ).sum(:total)
    }
    
    @top_customers = User
      .joins(:orders)
      .where(orders: { status: 'paid' })
      .group('users.id', 'users.name')
      .order('SUM(orders.total) DESC')
      .limit(10)
      .select('users.id, users.name, SUM(orders.total) as total_spent')
  end
end
```

---

## Step 1603: Custom Analytics Service

```ruby
# app/services/analytics_service.rb
class AnalyticsService
  def initialize(tenant: nil, period: 30.days)
    @tenant = tenant
    @period = period
    @start_date = period.ago
  end
  
  def dashboard_data
    {
      kpis: calculate_kpis,
      revenue_trend: revenue_trend,
      category_breakdown: category_breakdown,
      customer_segments: customer_segments,
      product_performance: product_performance
    }
  end
  
  private
  
  def calculate_kpis
    {
      total_revenue: {
        value: revenue_in_period,
        change: percentage_change(:revenue),
        formatted: format_currency(revenue_in_period)
      },
      total_orders: {
        value: orders_in_period,
        change: percentage_change(:orders)
      },
      new_customers: {
        value: new_customers_in_period,
        change: percentage_change(:customers)
      },
      average_order_value: {
        value: avg_order_value,
        change: percentage_change(:aov),
        formatted: format_currency(avg_order_value)
      },
      conversion_rate: {
        value: conversion_rate,
        formatted: "#{conversion_rate}%"
      },
      cart_abandonment_rate: {
        value: cart_abandonment_rate,
        formatted: "#{cart_abandonment_rate}%"
      }
    }
  end
  
  def revenue_in_period
    base_orders.sum(:total)
  end
  
  def orders_in_period
    base_orders.count
  end
  
  def new_customers_in_period
    User.where(created_at: @start_date..)
        .where(role: 'customer')
        .count
  end
  
  def avg_order_value
    base_orders.average(:total)&.round(2) || 0
  end
  
  def conversion_rate
    total_sessions = Ahoy::Visit.where('started_at >= ?', @start_date).count
    return 0 if total_sessions.zero?
    
    converting_sessions = base_orders.distinct.count(:ahoy_visit_id)
    (converting_sessions.to_f / total_sessions * 100).round(2)
  end
  
  def cart_abandonment_rate
    carts_created = Cart.where('created_at >= ?', @start_date).count
    carts_converted = base_orders.count
    return 0 if carts_created.zero?
    
    ((carts_created - carts_converted).to_f / carts_created * 100).round(2)
  end
  
  def percentage_change(metric)
    current = case metric
    when :revenue then revenue_in_period
    when :orders then orders_in_period
    when :customers then new_customers_in_period
    when :aov then avg_order_value
    end
    
    previous = case metric
    when :revenue then prev_period_orders.sum(:total)
    when :orders then prev_period_orders.count
    when :customers
      User.where(
        created_at: (2 * @period).ago..@start_date,
        role: 'customer'
      ).count
    when :aov then prev_period_orders.average(:total)&.to_f || 0
    end
    
    return nil if previous.zero?
    ((current.to_f - previous) / previous * 100).round(1)
  end
  
  def revenue_trend
    base_orders
      .group_by_day(:paid_at, range: @start_date..Time.current)
      .sum(:total)
  end
  
  def category_breakdown
    OrderItem
      .joins(product: :category, order: [])
      .merge(base_orders)
      .group('categories.name')
      .select(
        'categories.name',
        'SUM(order_items.quantity * order_items.unit_price) as revenue',
        'SUM(order_items.quantity) as quantity'
      )
      .order('revenue DESC')
  end
  
  def customer_segments
    segments = {
      new: User.where('created_at >= ?', @start_date).count,
      returning: User.joins(:orders)
                    .where(orders: { status: 'paid' })
                    .where('orders.paid_at >= ?', @start_date)
                    .having('COUNT(orders.id) > 1')
                    .count,
      at_risk: User.where(
        'last_order_at < ? AND last_order_at >= ?',
        90.days.ago, 180.days.ago
      ).count,
      lost: User.where('last_order_at < ?', 180.days.ago).count
    }
    segments
  end
  
  def product_performance
    Product
      .joins(:order_items)
      .merge(OrderItem.joins(:order).merge(base_orders))
      .group(:id, :name)
      .select(
        'products.id, products.name',
        'SUM(order_items.quantity) as total_sold',
        'SUM(order_items.quantity * order_items.unit_price) as revenue'
      )
      .order('revenue DESC')
      .limit(20)
  end
  
  def base_orders
    Order.paid.where('paid_at >= ?', @start_date)
  end
  
  def prev_period_orders
    Order.paid.where(paid_at: (2 * @period).ago..@start_date)
  end
  
  def format_currency(amount)
    "฿#{format('%.2f', amount).reverse.gsub(/(\d{3})(?=\d)/, '\\1,').reverse}"
  end
end
```

---

## Step 1604: Google Analytics Integration

```ruby
# ใส่ Google Analytics ผ่าน Turbo/Hotwire
# app/views/layouts/application.html.erb

<%# Google Analytics 4 %>
<% if Rails.env.production? && ENV['GA_MEASUREMENT_ID'].present? %>
  <!-- Google tag (gtag.js) -->
  <script async src="https://www.googletagmanager.com/gtag/js?id=<%= ENV['GA_MEASUREMENT_ID'] %>"></script>
  <script>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', '<%= ENV['GA_MEASUREMENT_ID'] %>', {
      user_id: '<%= current_user&.id %>',
      user_properties: {
        role: '<%= current_user&.role %>',
        plan: '<%= current_user&.subscription&.plan %>'
      }
    });
  </script>
<% end %>

<%# Track Turbo page views %>
<script>
  document.addEventListener('turbo:load', function() {
    if (typeof gtag === 'function') {
      gtag('event', 'page_view', {
        page_title: document.title,
        page_path: location.pathname + location.search
      });
    }
  });
</script>

# Helper สำหรับ track events
module GoogleAnalyticsHelper
  def gtag_event(event_name, params = {})
    return '' unless Rails.env.production?
    
    content_tag(:script) do
      "gtag('event', '#{event_name}', #{params.to_json});".html_safe
    end
  end
end

# ใน views
<% if @order.just_placed? %>
  <%= gtag_event 'purchase', {
    transaction_id: @order.number,
    value: @order.total.to_f,
    currency: 'THB',
    items: @order.items.map { |i| {
      item_id: i.product.sku,
      item_name: i.product.name,
      price: i.unit_price.to_f,
      quantity: i.quantity
    }}
  } %>
<% end %>
```

---

## Step 1605: Export Reports (CSV, PDF)

```ruby
# app/controllers/reports_controller.rb
class ReportsController < ApplicationController
  def index
    @report_types = [
      { id: 'sales', name: 'รายงานยอดขาย' },
      { id: 'customers', name: 'รายงานลูกค้า' },
      { id: 'inventory', name: 'รายงานสินค้าคงคลัง' },
      { id: 'revenue', name: 'รายงานรายได้' }
    ]
  end
  
  def generate
    report_service = ReportGenerator.call(
      report_type: params[:report_type],
      user: current_user,
      start_date: params[:start_date],
      end_date: params[:end_date],
      format: params[:format]
    )
    
    if report_service.success?
      respond_to do |format|
        format.csv do
          send_data report_service.to_csv,
            filename: "#{params[:report_type]}_#{Date.today}.csv",
            type: 'text/csv; charset=utf-8',
            disposition: 'attachment'
        end
        
        format.pdf do
          send_data generate_pdf(report_service.report_data),
            filename: "#{params[:report_type]}_#{Date.today}.pdf",
            type: 'application/pdf',
            disposition: 'attachment'
        end
        
        format.xlsx do
          xlsx = generate_xlsx(report_service.report_data)
          send_data xlsx.to_stream.read,
            filename: "#{params[:report_type]}_#{Date.today}.xlsx",
            type: 'application/xlsx'
        end
        
        format.json do
          render json: report_service.report_data
        end
      end
    else
      redirect_to reports_path, alert: report_service.errors.join(', ')
    end
  end
  
  private
  
  def generate_pdf(data)
    Prawn::Document.new do |pdf|
      pdf.font_families.update('THSarabun' => {
        normal: Rails.root.join('app/assets/fonts/THSarabunNew.ttf'),
        bold: Rails.root.join('app/assets/fonts/THSarabunNew Bold.ttf')
      })
      
      pdf.font 'THSarabun'
      
      # Header
      pdf.text data[:title], size: 20, style: :bold
      pdf.text "ช่วงเวลา: #{data[:period]}", size: 12
      pdf.text "สร้างเมื่อ: #{data[:generated_at].strftime('%d/%m/%Y %H:%M')}", size: 10
      pdf.move_down 20
      
      # Summary
      pdf.text 'สรุป', size: 14, style: :bold
      data[:summary].each do |key, value|
        pdf.text "#{key.to_s.humanize}: #{value}"
      end
      pdf.move_down 20
      
      # Table
      if data[:records].any?
        headers = data[:records].first.keys.map { |k| k.to_s.humanize }
        rows = data[:records].map(&:values)
        
        pdf.table([headers] + rows, header: true, row_colors: ['FFFFFF', 'F5F5F5']) do
          row(0).background_color = '1E88E5'
          row(0).text_color = 'FFFFFF'
          row(0).font_style = :bold
        end
      end
    end.render
  end
  
  def generate_xlsx(data)
    Axlsx::Package.new do |p|
      wb = p.workbook
      
      styles = wb.styles
      header_style = styles.add_style(
        b: true,
        bg_color: '1E88E5',
        fg_color: 'FFFFFF',
        border: Axlsx::STYLE_THIN_BORDER
      )
      
      wb.add_worksheet(name: data[:title]) do |sheet|
        # Title row
        sheet.add_row [data[:title]], style: [header_style]
        sheet.add_row ["ช่วงเวลา: #{data[:period]}"]
        sheet.add_row []
        
        # Headers
        if data[:records].any?
          headers = data[:records].first.keys.map { |k| k.to_s.humanize }
          sheet.add_row headers, style: Array.new(headers.length, header_style)
          
          # Data
          data[:records].each do |record|
            sheet.add_row record.values
          end
        end
        
        # Auto-width
        sheet.column_widths(*Array.new(data[:records].first&.keys&.length || 0, 20))
      end
    end
  end
end
```

---

## Step 1606: Scheduled Reports กับ Active Job

```ruby
# app/jobs/scheduled_report_job.rb
class ScheduledReportJob < ApplicationJob
  queue_as :reports
  
  def perform(report_config_id)
    config = ReportSchedule.find(report_config_id)
    return unless config.active?
    
    # สร้าง report
    report_service = ReportGenerator.call(
      report_type: config.report_type,
      user: config.created_by,
      start_date: config.report_start_date,
      end_date: config.report_end_date,
      format: config.format
    )
    
    if report_service.success?
      # แนบไฟล์ใน email
      ReportMailer.scheduled_report(
        config.recipients,
        report_service.to_file,
        config
      ).deliver_later
      
      # บันทึก history
      config.report_runs.create!(
        status: 'success',
        records_count: report_service.report_data[:records]&.count || 0,
        ran_at: Time.current
      )
    else
      config.report_runs.create!(
        status: 'failed',
        error_message: report_service.errors.join(', '),
        ran_at: Time.current
      )
    end
  end
end

# config/initializers/scheduled_jobs.rb
# กำหนด schedule ด้วย sidekiq-scheduler
Sidekiq::Scheduler.enabled = true

# config/schedule.yml
daily_reports:
  cron: '0 8 * * *'  # ทุกวันเวลา 08:00
  class: 'DailyReportJob'
  queue: reports

weekly_summary:
  cron: '0 9 * * 1'  # ทุกวันจันทร์ 09:00
  class: 'WeeklyReportJob'
  queue: reports
  
monthly_revenue:
  cron: '0 10 1 * *'  # วันที่ 1 ของทุกเดือน 10:00
  class: 'MonthlyRevenueReportJob'
  queue: reports
```

---

## Step 1607-1620: แบบฝึกหัด 20 ข้อ

### แบบฝึกหัดที่ 1: Real-time Analytics Dashboard

```ruby
# app/channels/analytics_channel.rb
class AnalyticsChannel < ApplicationCable::Channel
  def subscribed
    stream_from "analytics_#{current_user.id}" if current_user.admin?
  end
  
  def request_stats
    stats = {
      live_visitors: Ahoy::Visit.where('started_at > ?', 5.minutes.ago).count,
      orders_today: Order.paid.where('paid_at >= ?', Date.today).count,
      revenue_today: Order.paid.where('paid_at >= ?', Date.today).sum(:total)
    }
    
    transmit stats
  end
end

# app/jobs/broadcast_analytics_job.rb
class BroadcastAnalyticsJob < ApplicationJob
  queue_as :default
  
  def perform
    stats = {
      live_visitors: Ahoy::Visit.where('started_at > ?', 5.minutes.ago).count,
      orders_last_hour: Order.paid.where('paid_at >= ?', 1.hour.ago).count,
      revenue_today: Order.paid.where('paid_at >= ?', Date.today).sum(:total)
    }
    
    AdminUser.all.each do |admin|
      ActionCable.server.broadcast("analytics_#{admin.id}", stats)
    end
  end
end
```

### แบบฝึกหัดที่ 2: Cohort Analysis

```ruby
# app/services/cohort_analysis.rb
class CohortAnalysis
  def initialize(months_back: 6)
    @months_back = months_back
  end
  
  def retention_matrix
    cohorts = {}
    
    @months_back.downto(0) do |months_ago|
      cohort_date = months_ago.months.ago.beginning_of_month
      
      # Users ที่สมัครใน cohort นี้
      cohort_users = User.where(
        created_at: cohort_date..cohort_date.end_of_month
      ).pluck(:id)
      
      next if cohort_users.empty?
      
      # Retention ในแต่ละเดือนถัดไป
      retention = {}
      0.upto(@months_back - months_ago) do |period|
        period_start = cohort_date + period.months
        period_end = period_start.end_of_month
        
        returning_users = Order.paid
          .where(user_id: cohort_users)
          .where(paid_at: period_start..period_end)
          .distinct.count(:user_id)
        
        retention[period] = {
          count: returning_users,
          percentage: (returning_users.to_f / cohort_users.count * 100).round(1)
        }
      end
      
      cohorts[cohort_date.strftime('%b %Y')] = {
        size: cohort_users.count,
        retention: retention
      }
    end
    
    cohorts
  end
end
```

### แบบฝึกหัดที่ 3: Funnel Analysis

```ruby
# app/services/funnel_analysis.rb
class FunnelAnalysis
  STEPS = [
    { name: 'เข้าเว็บ', event: nil, model: Ahoy::Visit },
    { name: 'ดูสินค้า', event: 'Viewed Product', model: Ahoy::Event },
    { name: 'เพิ่มลงตะกร้า', event: 'Added to Cart', model: Ahoy::Event },
    { name: 'เริ่ม Checkout', event: 'Started Checkout', model: Ahoy::Event },
    { name: 'ชำระเงินสำเร็จ', event: 'Placed Order', model: Ahoy::Event }
  ].freeze
  
  def initialize(start_date: 30.days.ago, end_date: Time.current)
    @start_date = start_date
    @end_date = end_date
  end
  
  def analyze
    results = STEPS.map do |step|
      count = if step[:event]
        Ahoy::Event
          .where(name: step[:event])
          .where(time: @start_date..@end_date)
          .distinct.count(:visit_id)
      else
        Ahoy::Visit
          .where(started_at: @start_date..@end_date)
          .count
      end
      
      { name: step[:name], count: count }
    end
    
    # คำนวณ drop-off rates
    results.each_with_index.map do |step, index|
      prev_count = index > 0 ? results[index - 1][:count] : step[:count]
      
      step.merge(
        conversion_from_prev: prev_count > 0 ? (step[:count].to_f / prev_count * 100).round(1) : 0,
        dropoff_rate: prev_count > 0 ? ((prev_count - step[:count]).to_f / prev_count * 100).round(1) : 0
      )
    end
  end
end
```

### แบบฝึกหัดที่ 4: Custom Report Builder

```ruby
# app/services/custom_report_builder.rb
class CustomReportBuilder
  AVAILABLE_METRICS = %w[revenue orders customers products].freeze
  AVAILABLE_DIMENSIONS = %w[date category product user].freeze
  
  def initialize(metrics:, dimensions:, filters: {}, date_range:)
    @metrics = metrics & AVAILABLE_METRICS
    @dimensions = dimensions & AVAILABLE_DIMENSIONS
    @filters = filters
    @date_range = date_range
  end
  
  def build
    base_query = Order.paid.where(paid_at: @date_range)
    
    # Apply filters
    base_query = apply_filters(base_query, @filters)
    
    # Group by dimensions
    grouped = group_by_dimensions(base_query, @dimensions)
    
    # Select metrics
    select_metrics(grouped, @metrics)
  end
  
  private
  
  def apply_filters(query, filters)
    query = query.where(user: User.where(role: filters[:role])) if filters[:role]
    query = query.where(total: filters[:min_amount]..) if filters[:min_amount]
    query = query.joins(items: { product: :category }).where(categories: { id: filters[:category_id] }) if filters[:category_id]
    query
  end
  
  def group_by_dimensions(query, dimensions)
    dimensions.reduce(query) do |q, dim|
      case dim
      when 'date'     then q.group_by_day(:paid_at)
      when 'category' then q.joins(items: { product: :category }).group('categories.name')
      when 'user'     then q.group(:user_id)
      else                 q
      end
    end
  end
  
  def select_metrics(query, metrics)
    result = {}
    
    metrics.each do |metric|
      result[metric] = case metric
      when 'revenue'   then query.sum(:total)
      when 'orders'    then query.count
      when 'customers' then query.distinct.count(:user_id)
      end
    end
    
    result
  end
end
```

### แบบฝึกหัดที่ 5: PDF Report กับ Prawn

```ruby
# app/services/pdf_report_generator.rb
class PdfReportGenerator
  THAI_FONT_PATH = Rails.root.join('app/assets/fonts/THSarabunNew.ttf')
  
  def self.generate(report_data)
    new(report_data).generate
  end
  
  def initialize(report_data)
    @data = report_data
  end
  
  def generate
    Prawn::Document.new(page_size: 'A4', margin: [40, 40, 40, 40]) do |pdf|
      setup_fonts(pdf)
      add_header(pdf)
      add_summary_section(pdf)
      add_chart_placeholder(pdf)
      add_data_table(pdf)
      add_footer(pdf)
    end.render
  end
  
  private
  
  def setup_fonts(pdf)
    pdf.font_families.update('THSarabun' => {
      normal: THAI_FONT_PATH,
      bold: THAI_FONT_PATH  # use same if no bold variant
    })
    pdf.font 'THSarabun'
  end
  
  def add_header(pdf)
    pdf.bounding_box([0, pdf.cursor], width: pdf.bounds.width) do
      pdf.text @data[:title], size: 22, style: :bold, align: :center
      pdf.move_down 5
      pdf.text @data[:period], size: 12, align: :center, color: '666666'
      pdf.move_down 5
      pdf.text "สร้างเมื่อ: #{Time.current.strftime('%d/%m/%Y %H:%M')}", size: 10, align: :right
      pdf.stroke_horizontal_rule
    end
    pdf.move_down 20
  end
  
  def add_summary_section(pdf)
    return unless @data[:summary]
    
    pdf.text 'สรุปผล', size: 16, style: :bold
    pdf.move_down 10
    
    # Summary ใน grid
    col_width = pdf.bounds.width / 3
    
    @data[:summary].each_slice(3) do |group|
      pdf.bounding_box([0, pdf.cursor], width: pdf.bounds.width, height: 60) do
        group.each_with_index do |(key, value), i|
          pdf.bounding_box([i * col_width, pdf.cursor], width: col_width - 10) do
            pdf.fill_color '1E88E5'
            pdf.rectangle [0, pdf.cursor], col_width - 10, 50
            pdf.fill
            pdf.fill_color 'FFFFFF'
            pdf.text key.to_s.humanize, size: 10
            pdf.text value.to_s, size: 16, style: :bold
            pdf.fill_color '000000'
          end
        end
      end
      pdf.move_down 70
    end
  end
  
  def add_data_table(pdf)
    return if @data[:records].blank?
    
    pdf.text 'รายละเอียด', size: 16, style: :bold
    pdf.move_down 10
    
    headers = @data[:records].first.keys.map { |k| k.to_s.humanize }
    rows = @data[:records].map(&:values)
    
    pdf.table(
      [headers] + rows,
      header: true,
      width: pdf.bounds.width,
      row_colors: %w[FFFFFF F8F9FA]
    ) do
      row(0).background_color = '1E88E5'
      row(0).text_color = 'FFFFFF'
      row(0).font_style = :bold
      cells.border_width = 0.5
      cells.border_color = 'DDDDDD'
    end
  end
  
  def add_footer(pdf)
    pdf.repeat(:all) do
      pdf.draw_text "หน้า #{pdf.page_number}", at: [pdf.bounds.right - 50, -20], size: 9
      pdf.draw_text 'MyApp Report System', at: [0, -20], size: 9
    end
  end
end
```

---

**จบ Part 74: Analytics and Reporting**

*ในส่วนถัดไป Part 75 เราจะเรียนรู้เกี่ยวกับ Rails Engines*
