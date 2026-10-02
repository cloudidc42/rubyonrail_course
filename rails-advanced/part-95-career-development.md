# Part 95: Career Development สำหรับ Rails Developer

## บทนำ

การเป็น Rails developer ที่ประสบความสำเร็จไม่ได้หยุดแค่การเขียนโค้ดเก่ง บทนี้จะครอบคลุมเส้นทางอาชีพ, การสร้าง portfolio, การมีส่วนร่วม open source, การเตรียมตัวสัมภาษณ์, system design, การทำงาน remote และการวางแผนเงินเดือน

## 1. Career Path สำหรับ Rails Developer

### 1.1 Levels และ Responsibilities

```
Junior Developer (0-2 ปี)
├── สร้าง features ตาม spec ที่ชัดเจน
├── Fix bugs ที่ระบุชัดเจน
├── เขียน tests ตาม pattern ที่มีอยู่
├── Code review เบื้องต้น
└── ต้องการ guidance ในการ design

Mid-level Developer (2-5 ปี)
├── สร้าง features จาก requirements ที่ไม่ชัด
├── Design ระบบขนาดกลาง
├── Mentor juniors
├── Lead code reviews
└── Propose technical solutions

Senior Developer (5+ ปี)
├── Architect ระบบขนาดใหญ่
├── Cross-team technical leadership
├── Define coding standards
├── Drive technical strategy
└── Mentor multiple developers

Staff/Principal Engineer (8+ ปี)
├── Organization-wide technical impact
├── Set technology direction
├── Partner with business stakeholders
└── Represent engineering externally
```

### 1.2 Technical Skills ที่ต้องมีแต่ละ Level

```ruby
# Junior: รู้ Rails basics
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user
    else
      render :new
    end
  end
end

# Mid-level: รู้ patterns, performance, testing
class UsersController < ApplicationController
  before_action :authenticate_user!
  
  def create
    result = UserCreationService.call(user_params, current_user: current_user)
    
    if result.success?
      redirect_to result.user, notice: "Welcome!"
    else
      @errors = result.errors
      render :new, status: :unprocessable_entity
    end
  end
end

# Senior: รู้ architecture, scalability, tradeoffs
# - เมื่อไรควรใช้ background jobs
# - เมื่อไรควร denormalize database
# - เมื่อไรควร introduce caching layer
# - เมื่อไรควร move to microservices (และเมื่อไรไม่ควร)
# - เมื่อไรควร accept technical debt
```

## 2. Portfolio Projects

### 2.1 Project Ideas ที่โชว์ Skills

```
1. SaaS Application
   - Multi-tenancy (subdomain-based)
   - Stripe subscriptions + billing
   - Feature flags ด้วย Flipper
   - Background jobs ด้วย Sidekiq
   - Real-time features ด้วย Action Cable
   
2. API Platform
   - RESTful + GraphQL endpoints
   - JWT Authentication
   - Rate limiting
   - Versioning strategy
   - Comprehensive documentation (OpenAPI)
   
3. E-commerce Platform
   - Product catalog + search (Elasticsearch/pg_search)
   - Shopping cart (Redis/session)
   - Payment processing (Stripe)
   - Order management + inventory
   - Admin dashboard ด้วย metrics
   
4. Content Management System
   - Rich text editing (Action Text)
   - Media library (ActiveStorage + S3)
   - Publishing workflow (states)
   - Full-text search
   - REST API + Webhooks
   
5. Developer Tool / CLI
   - Solve real problems
   - สร้างเป็น gem
   - Open source ด้วย tests
   - Good documentation
```

### 2.2 Portfolio README Template

```markdown
# Project Name

> One sentence description of what this project does

## 🚀 Live Demo
[demo.projectname.com](https://demo.projectname.com) | 
[Video Walkthrough](https://youtube.com/...)

**Test Account:**
- Email: demo@example.com  
- Password: Demo123!

## ✨ Features
- **Multi-tenancy**: Subdomain-based tenant isolation
- **Stripe Billing**: Plans, trials, webhooks, customer portal
- **Real-time**: Action Cable notifications, live updates
- **Background Jobs**: Sidekiq + Redis for async processing
- **File Uploads**: ActiveStorage + S3 + image variants
- **Full-text Search**: PostgreSQL GIN index

## 🛠 Tech Stack
- **Backend**: Ruby 3.2, Rails 7.1
- **Database**: PostgreSQL 15
- **Cache/Jobs**: Redis, Sidekiq
- **Storage**: AWS S3 + CloudFront
- **Auth**: Devise + JWT
- **Payments**: Stripe
- **Deployment**: Kamal + Docker

## 🏗 Architecture Decisions

### Why multi-tenancy via subdomain?
Row-level tenancy was chosen over schema-per-tenant for simplicity
at this scale. Would consider schema-per-tenant beyond 10K tenants.

### Why Sidekiq over Solid Queue?
Redis was already required for cache, so Sidekiq added minimal
overhead. Would choose Solid Queue for simpler deployments.

## 🔧 Local Setup
```bash
git clone https://github.com/you/project
cd project
bundle install
cp .env.example .env
bin/rails db:setup
bin/dev
```

## 📊 Performance
- Page load: < 200ms (p95) on Heroku Standard-1X
- API response: < 50ms (p95) for cached endpoints
- Background jobs: ~1000/min throughput
```

### 2.3 GitHub Profile ที่ดี

```markdown
<!-- README.md สำหรับ GitHub Profile -->
# Hi, I'm [Name] 👋

Senior Rails Developer | 7 years building web applications

## About Me
- 🏗 Building scalable Rails applications
- 🔷 Open source contributor (rails/rails, activerecord-pg_enum)
- 📝 Writing about Ruby performance at [blog.example.com]
- 🌍 Available for remote positions

## Tech Stack
Ruby · Rails · PostgreSQL · Redis · Sidekiq · Docker · AWS

## Featured Projects
| Project | Description | Stack |
|---------|-------------|-------|
| [SaaSKit](https://github.com/you/saaskit) | Rails SaaS starter | Rails, Stripe, Flipper |
| [RubyMetrics](https://github.com/you/ruby-metrics) | Performance gem | Ruby, Prometheus |
| [APIBuilder](https://github.com/you/apibuilder) | Rails API generator | Rails, OpenAPI |

## Open Source Contributions
- [rails/rails #12345](https://github.com/rails/rails/pull/12345) - Fixed N+1 in belongs_to
- [heartcombo/devise #6789](https://github.com/heartcombo/devise/pull/6789) - JWT improvements

## Latest Blog Posts
- [Why We Migrated from Sidekiq to Solid Queue](https://blog.example.com/solid-queue)
- [Rails 8 Features You Should Know](https://blog.example.com/rails-8)
- [PostgreSQL Full-text Search in Rails](https://blog.example.com/pg-search)
```

## 3. Open Source Contribution

### 3.1 วิธีเริ่มต้น Contribute

```bash
# 1. Fork และ clone project ที่สนใจ
git clone https://github.com/your-username/rails.git
cd rails

# 2. Setup development environment
bundle install

# 3. หา good first issue
# - https://github.com/rails/rails/labels/good%20first%20issue
# - https://github.com/heartcombo/devise/labels/beginner%20friendly

# 4. สร้าง branch สำหรับ fix
git checkout -b fix/n-plus-one-in-belongs-to

# 5. เขียน failing test ก่อน (TDD)
# 6. Implement fix
# 7. รัน tests ให้ผ่าน
bundle exec rake test

# 8. Push และสร้าง Pull Request
git push origin fix/n-plus-one-in-belongs-to
```

### 3.2 สร้าง Ruby Gem

```bash
# สร้าง gem ใหม่
bundle gem my_awesome_gem

# Structure:
# my_awesome_gem/
# ├── lib/
# │   ├── my_awesome_gem.rb
# │   └── my_awesome_gem/
# │       └── version.rb
# ├── spec/
# │   └── my_awesome_gem_spec.rb
# ├── Gemfile
# ├── my_awesome_gem.gemspec
# └── README.md
```

```ruby
# my_awesome_gem.gemspec
Gem::Specification.new do |spec|
  spec.name          = "my_awesome_gem"
  spec.version       = MyAwesomeGem::VERSION
  spec.authors       = ["Your Name"]
  spec.email         = ["you@example.com"]
  
  spec.summary       = "A short summary"
  spec.description   = "A longer description"
  spec.homepage      = "https://github.com/you/my_awesome_gem"
  spec.license       = "MIT"
  
  spec.required_ruby_version = ">= 3.0"
  
  spec.metadata["allowed_push_host"] = "https://rubygems.org"
  spec.metadata["homepage_uri"] = spec.homepage
  spec.metadata["source_code_uri"] = spec.homepage
  spec.metadata["changelog_uri"] = "#{spec.homepage}/blob/main/CHANGELOG.md"
  
  spec.files = Dir.chdir(File.expand_path('..', __FILE__)) do
    `git ls-files -z`.split("\x0").reject { |f| f.match(%r{^(test|spec|features)/}) }
  end
  
  spec.bindir        = "exe"
  spec.executables   = spec.files.grep(%r{^exe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]
  
  # Dependencies
  spec.add_dependency "activesupport", ">= 6.0"
  
  spec.add_development_dependency "rspec"
  spec.add_development_dependency "rubocop"
end
```

```ruby
# lib/my_awesome_gem.rb
require "my_awesome_gem/version"
require "my_awesome_gem/configuration"
require "my_awesome_gem/client"

module MyAwesomeGem
  class Error < StandardError; end
  
  class << self
    attr_accessor :configuration
    
    def configure
      self.configuration ||= Configuration.new
      yield(configuration)
    end
  end
end

# Publish ไปที่ RubyGems
# bundle exec rake release
```

## 4. Interview Preparation

### 4.1 Common Technical Questions

```ruby
# Q: อธิบาย N+1 problem และวิธีแก้

# ❌ N+1 problem
users = User.all
users.each do |user|
  puts user.posts.count  # Query ทุก user -> N+1 queries!
end

# ✅ Solution 1: includes (eager loading)
users = User.includes(:posts)
users.each do |user|
  puts user.posts.size  # ใช้ loaded data - 2 queries total
end

# ✅ Solution 2: counter_cache
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
end

class User < ApplicationRecord
  has_many :posts
end

users = User.all
users.each do |user|
  puts user.posts_count  # Read from column - 1 query
end

# ✅ Solution 3: SQL เดียว
User.joins(:posts).select("users.*, COUNT(posts.id) AS posts_count").group("users.id")
```

```ruby
# Q: อธิบาย Ruby memory management

# Object allocation
a = "hello"  # Ruby allocates String object on heap
b = a        # b และ a point to same object
b << " world"  # mutate in place - a ก็เปลี่ยนด้วย

# Garbage collection
GC.start  # Force GC (ปกติไม่ทำ)

# Weak references - ไม่ prevent GC
require 'weakref'
obj = Object.new
weak = WeakRef.new(obj)
obj = nil
GC.start
begin
  weak.something  # WeakRef::RefError - object was collected
rescue WeakRef::RefError
  puts "Object was garbage collected"
end

# Copy-on-write
str1 = "hello"
str2 = str1.dup  # Creates copy
str2 << " world"  # str1 unchanged
```

```ruby
# Q: อธิบาย Rails concerns vs inheritance

# Inheritance: สำหรับ IS-A relationships
class Vehicle
  def start_engine; end
end

class Car < Vehicle  # Car IS-A Vehicle
  def open_trunk; end
end

# Concerns: สำหรับ mixable behaviors (HAS-A)
module Trackable
  def track_location; end
end

module Sellable
  def list_for_sale; end
end

class Car < Vehicle
  include Trackable  # Car CAN BE tracked
  include Sellable   # Car CAN BE sold
end

class Motorcycle < Vehicle
  include Trackable  # Motorcycle CAN ALSO be tracked
  # But NOT Sellable - different business rules
end
```

### 4.2 System Design Questions

```
Q: Design a Twitter-like system

ต้องถามก่อน:
1. Scale: กี่ users? กี่ tweets/second?
2. Features: What specifically?
3. Latency requirements?
4. Consistency requirements?

สมมติ: 100M users, 10K tweets/sec, < 200ms timeline load

Components:
┌─────────────────────────────────────┐
│  Load Balancer (nginx/ELB)          │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│  Rails API Servers (multiple)        │
│  - Stateless                         │
│  - Read replicas for DB              │
└───┬──────────────────┬──────────────┘
    │                  │
┌───┴───┐         ┌───┴──────────┐
│ Write │         │ Redis Cache  │
│ DB    │         │ - Timeline   │
│ (PG)  │         │ - Sessions   │
└───────┘         └──────────────┘
                       │
                  ┌────┴─────────┐
                  │ Sidekiq      │
                  │ - Fan-out    │
                  │ - Notifications│
                  └──────────────┘

Timeline approach:
- Push model (fanout on write) สำหรับ < 10K followers
- Pull model สำหรับ celebrities (> 10K followers)
- Redis sorted sets สำหรับ timeline
```

```ruby
# การ implement timeline cache
class TimelineService
  TIMELINE_SIZE = 800
  
  def timeline_for(user_id, limit: 20, before_id: nil)
    cache_key = "timeline:#{user_id}"
    
    # Try cache first
    cached = Redis.current.lrange(cache_key, 0, limit - 1)
    
    if cached.length >= limit
      tweet_ids = cached.map(&:to_i)
      Tweet.where(id: tweet_ids).includes(:user)
    else
      # Cache miss: rebuild from DB
      rebuild_timeline(user_id, limit: TIMELINE_SIZE)
      tweets_from_db(user_id, limit: limit)
    end
  end
  
  def fanout_tweet(tweet_id, author_id)
    # Get followers (paginated for large follow counts)
    User.joins(:follows).where(follows: { following_id: author_id })
        .in_batches(of: 1000) do |batch|
      batch.each do |follower|
        Redis.current.lpush("timeline:#{follower.id}", tweet_id)
        Redis.current.ltrim("timeline:#{follower.id}", 0, TIMELINE_SIZE - 1)
      end
    end
  end
  
  private
  
  def rebuild_timeline(user_id, limit:)
    following_ids = Follow.where(follower_id: user_id).pluck(:following_id)
    
    tweets = Tweet.where(user_id: [user_id] + following_ids)
      .order(created_at: :desc)
      .limit(limit)
      .pluck(:id)
    
    cache_key = "timeline:#{user_id}"
    Redis.current.del(cache_key)
    tweets.each { |id| Redis.current.rpush(cache_key, id) }
    Redis.current.expire(cache_key, 7.days)
  end
end
```

### 4.3 Behavioral Questions (STAR Method)

```
STAR = Situation, Task, Action, Result

Q: "Tell me about a time you improved performance significantly"

S: "At my previous job, our Rails app had slow page loads (8s average) 
   during peak hours"

T: "I was tasked with reducing load time to under 2 seconds"

A: "I:
   1. Profiled with rack-mini-profiler to identify bottlenecks
   2. Found 47 N+1 queries on the main dashboard
   3. Added includes() and counter_caches where appropriate
   4. Added Redis caching for expensive computations
   5. Optimized database indexes for slow queries"

R: "Reduced average load time from 8s to 1.2s (85% improvement)
   Reduced database load by 60%
   Improved user retention by 15% (measured via analytics)"

Key points:
- Specifics: ตัวเลขจริง
- Process: approach systematic
- Impact: business impact ไม่ใช่แค่ technical
```

## 5. Rails Performance Interview

### 5.1 Database Optimization

```ruby
# อธิบายการ optimize query นี้:
# SLOW:
Article.all.each do |article|
  puts article.comments.where(approved: true).count
end

# FAST: Single query
Article.left_joins(:comments)
  .where(comments: { approved: true })
  .group("articles.id")
  .count

# หรือ counter cache ถ้า query นี้ run บ่อย
class Comment < ApplicationRecord
  belongs_to :article, counter_cache: :approved_comments_count
  
  after_save :update_article_cache
  
  def update_article_cache
    if saved_change_to_approved?
      if approved?
        article.increment!(:approved_comments_count)
      else
        article.decrement!(:approved_comments_count)
      end
    end
  end
end
```

### 5.2 Caching Strategies

```ruby
# มี 4 caching strategies หลักใน Rails:

# 1. HTTP Caching (fastest - ไม่ต้อง hit server)
def show
  @article = Article.find(params[:id])
  fresh_when(etag: @article, last_modified: @article.updated_at, public: true)
end

# 2. Page Caching (deprecated แต่ยังใช้ใน static sites)
# caches_page :index, :show

# 3. Action Caching (via rack-cache)
class ArticlesController < ApplicationController
  caches_action :index, expires_in: 1.hour
end

# 4. Fragment/Russian Doll Caching (most common)
# view: <%= cache article do %>
#   <%= render article %>
# <% end %>

# model: touch: true ใน associations
class Comment < ApplicationRecord
  belongs_to :article, touch: true  # invalidates article cache when comment changes
end
```

## 6. Remote Work Best Practices

### 6.1 Setup และ Tools

```
Essential Setup:
├── Hardware
│   ├── Ergonomic chair (Herman Miller, Steelcase)
│   ├── External monitor (ultrawide preferred)
│   ├── Mechanical keyboard
│   └── Quality headset with noise cancellation
│
├── Internet
│   ├── Fiber connection (100+ Mbps)
│   └── Backup hotspot
│
└── Software
    ├── Slack/Discord (async communication)
    ├── Linear/Jira (project management)
    ├── Notion/Confluence (documentation)
    ├── Loom (async video updates)
    └── Zoom/Meet (video calls)

Productivity:
├── Time zones: Use World Time Buddy
├── Focus time: Block calendar for deep work
├── Over-communicate: Write everything down
└── Documentation: Document decisions
```

### 6.2 Async Communication

```markdown
# Good async update example:

## Weekly Update - Week 42

### Done
- ✅ Implemented Stripe webhook handling (PR #234)
- ✅ Fixed N+1 in dashboard (PR #235) - reduced load time 40%
- ✅ Added RSpec tests for PaymentService (coverage: 94%)

### In Progress
- 🔄 Subscription cancellation flow (PR #236 - WIP)
  - Retention offer logic done
  - Still need: email templates, analytics events
  - Blocked: need designer to review UX (tagged @sarah)

### Blocked
- ⛔ Email sequence setup - waiting for marketing to provide copy

### Next Week
- Finish cancellation flow (target: Mon)
- Start multi-currency support
- Review @john's PR for SSO integration

### Questions/Decisions needed
1. Should we handle SEPA payment in v1 or defer to v2?
2. Stripe Radar rules - do we implement custom rules?
```

## 7. Salary Negotiation

### 7.1 ทำ Research

```
Sources สำหรับ salary data:
- levels.fyi (tech companies, especially FAANG)
- glassdoor.com (broader, self-reported)
- linkedin.com/salary
- remote.com/blog (remote salaries)
- stackoverflow.com/jobs/salary
- hired.com/state-of-software-engineers (annual report)

ปัจจัยที่ affect salary:
1. Location (SF vs Chicago vs Remote)
2. Company size (startup vs enterprise)
3. Industry (fintech pays more than education)
4. Years of experience
5. Specialization (ML, security pay premium)
6. Open source notoriety
7. Conference speaking
```

### 7.2 Negotiation Script

```
Initial offer: $150,000

Research: Market rate $165,000-180,000

Response:
"Thank you for the offer! I'm very excited about the role.
Based on my research of market rates for senior Rails engineers 
with my background in [specific skills], and considering [specific value 
you bring], I was expecting something closer to $175,000.

Is there flexibility in the base salary?"

If they say no:
"I understand. Could we revisit the equity component or 
discuss a performance review at 6 months?"

Other levers to negotiate:
- Signing bonus
- Equity/RSUs
- Home office stipend
- Learning budget ($2,000-5,000/year)
- Conference budget
- Flexible working hours
- Extra PTO
- Remote work flexibility
```

### 7.3 Total Compensation Calculation

```ruby
# ตัวอย่าง TC calculation
class CompensationCalculator
  def total_annual(
    base_salary:,
    bonus_percentage: 0,
    equity_value: 0,
    vesting_years: 4,
    benefits_value: 0
  )
    bonus = base_salary * (bonus_percentage / 100.0)
    annual_equity = equity_value / vesting_years.to_f
    
    base_salary + bonus + annual_equity + benefits_value
  end
  
  def compare_offers(offers)
    offers.map do |offer|
      tc = total_annual(**offer.slice(:base_salary, :bonus_percentage, :equity_value, :vesting_years, :benefits_value))
      
      {
        **offer,
        total_compensation: tc,
        monthly_net: estimate_monthly_net(tc, state: offer[:state])
      }
    end.sort_by { |o| -o[:total_compensation] }
  end
  
  private
  
  def estimate_monthly_net(tc, state:)
    # Simplified - actual depends on filing status, deductions
    federal_rate = 0.32  # Approx for high earners
    state_rates = { 'CA' => 0.093, 'TX' => 0.0, 'NY' => 0.0685, 'WA' => 0.0 }
    state_rate = state_rates[state] || 0.05
    
    net = tc * (1 - federal_rate - state_rate - 0.0765)  # 7.65% FICA
    (net / 12).round
  end
end

calc = CompensationCalculator.new
puts calc.total_annual(
  base_salary: 175_000,
  bonus_percentage: 15,
  equity_value: 200_000,
  vesting_years: 4,
  benefits_value: 20_000
)
# => $225,000 total annual compensation
```

## 8. Building Your Brand

### 8.1 Technical Blog

```ruby
# Topics ที่ควรเขียน:
# 1. Problem + Solution posts (most valuable)
# 2. Deep dives into Rails internals
# 3. Performance optimization case studies
# 4. Architecture decision records
# 5. Tutorial series

# Good blog post structure:
# Title: "How We Reduced Rails App Load Time by 80%"
# 
# 1. Problem statement (what was broken, impact)
# 2. Investigation (tools used, findings)
# 3. Solution (what you did, alternatives considered)
# 4. Results (metrics before/after)
# 5. Lessons learned
# 6. Code examples

# SEO tips:
# - Include "Ruby" or "Rails" + specific feature in title
# - Write > 1,500 words for complex topics
# - Include code snippets (people search for solutions)
# - Cross-post to dev.to and Medium
```

### 8.2 Conference Speaking

```
Getting into conferences:
1. Start with local meetups (Ruby meetup, RailsBridge)
2. Submit to regional conferences first
3. Then RailsConf, RubyConf, Brighton Ruby
4. Keep CFP open in calendar

Good talk topics:
- Performance optimization (always popular)
- Architecture patterns
- Upgrading major Rails versions
- Security in Rails apps
- Testing strategies

CFP Writing tips:
- Clear, specific title
- Problem + solution structure
- Who is the audience?
- What will they learn?
- Real-world case study > theory

Example CFP abstract:
"Debugging Memory Leaks in Production Rails Apps

Memory leaks are the silent killers of Rails applications.
In this talk, I'll share how we diagnosed and fixed a memory leak
that was causing our app to consume 2GB+ before crashing.

Attendees will learn:
- How to use memory_profiler and ObjectSpace
- Common sources of memory leaks in Rails
- GC configuration for production
- Monitoring strategies to catch leaks early

Based on a real incident at [company], this talk includes
the full debugging process with actual graphs and code."
```

## 9. Staying Current

### 9.1 Learning Resources

```
Daily:
- Ruby Weekly (newsletter)
- RubyFlow (aggregator)
- Twitter/X: @dhh, @tenderlove, @eileencodes, @_blind_sam

Weekly:
- Ruby on Rails Podcast
- Remote Ruby Podcast
- Rails changelog (github.com/rails/rails/releases)

Monthly:
- RubyGems statistics
- Ruby benchmarks
- New gem releases

Books:
- The Rails Way (Fernandez)
- Confident Ruby (Avdi Grimm)
- Practical Object-Oriented Design in Ruby (Sandy Metz)
- Ruby Under a Microscope (Pat Shaughnessy)
- Designing Data-Intensive Applications (Kleppmann) - not Ruby specific

Courses:
- GoRails (Chris Oliver - rails)
- thoughtbot (Upcase)
- Destroy All Software (Gary Bernhardt)
```

### 9.2 Keeping Skills Sharp

```ruby
# สิ่งที่ทำสม่ำเสมอเพื่อเติบโต:

# 1. Read source code ของ gems ที่ใช้
# https://github.com/rails/rails
# https://github.com/sidekiq/sidekiq

# 2. Contribute to open source (แม้ documentation)
# 3. Code kata / algorithm practice (Exercism.io ดีมากสำหรับ Ruby)
# 4. Review RFCs และ GitHub issues ใน Rails

# 5. Experiment ใน personal projects
# Try ของใหม่ที่ยังไม่กล้าใช้ใน production:
# - Hotwire Turbo
# - Solid Queue
# - Kamal 2.0
# - TruffleRuby

# 6. Teach others
# - Blog posts
# - Meetup talks
# - Pair programming
# - Code reviews with explanations
```

## 10. Career Paths นอก Rails

### 10.1 Specializations

```
Rails → DevOps/Platform Engineering
- Kubernetes, Docker, CI/CD
- Kamal, Capistrano
- AWS/GCP/Azure
- Monitoring, alerting

Rails → Security Engineering
- OWASP Top 10
- Penetration testing
- Security audits
- Secure coding practices

Rails → Architecture/Staff Engineering
- System design
- Technical leadership
- Cross-team influence
- Technology strategy

Rails → Engineering Management
- People management
- Project management
- Hiring and growing teams
- Technical roadmap

Rails → Technical Founder/CTO
- Business knowledge
- Fundraising
- Hiring
- Product strategy
- Customer development
```

### 10.2 Entrepreneurship

```ruby
# SaaS Ideas ที่ Rails developer สามารถ build ได้คนเดียว:
# - Developer tools (linting, testing, monitoring)
# - API aggregators
# - Automation tools
# - B2B vertical SaaS (niche industries)
# - Marketplace platforms

# Indie Hacker path:
# 1. Build MVP ใน 2-4 สัปดาห์
# 2. Launch บน Product Hunt / HN
# 3. Get 10 paying customers (validation)
# 4. Grow to $1K MRR (ramen profitable)
# 5. Grow to $10K MRR (quit your job)
# 6. Scale or sell

# Resources:
# - Indie Hackers (indiehackers.com)
# - MicroConf
# - "The Lean Startup"
# - "Zero to Sold" (Arvid Kahl)

# Rails-specific SaaS starters:
# - Jumpstart Pro (jumpstartrails.com)
# - SaaSkit (saaskit.dev)
# - Bullet Train (bullettrain.co)
```

## 11. Technical Leadership

### 11.1 Architecture Decision Records

```markdown
# ADR 001: Use Sidekiq over Solid Queue

## Status: Accepted

## Context
We need a background job processor. Options considered:
1. Solid Queue (new, DB-backed, no Redis needed)
2. Sidekiq (Redis-backed, battle-tested)
3. Resque (older, Redis-backed)
4. Delayed Job (DB-backed, no external deps)

## Decision
We will use Sidekiq.

## Rationale
- We already use Redis for caching, so no new infrastructure
- Sidekiq has 10+ years of production use
- Better monitoring (Web UI, metrics)
- Faster than DB-backed queues for our scale
- Large ecosystem of add-ons

## Consequences
- Redis becomes a critical dependency
- Developers need to understand Redis concepts
- Need Redis cluster for HA in production
- Migration path to Solid Queue documented if needed

## Revisit
When: If Redis becomes too expensive or we move to Postgres-only infra
Owner: @backend-team
```

### 11.2 Technical Mentoring

```ruby
# Effective Code Review ที่ช่วย grow juniors:

# ❌ Unhelpful comment:
# "This is wrong"

# ✅ Helpful comment:
# "Consider using `find_by` instead of `where.first` here.
# `find_by` is more expressive and slightly more efficient.
# Also, note that `where.first` will always sort by primary key,
# which may not match your intent.
#
# See: https://api.rubyonrails.org/classes/ActiveRecord/FinderMethods.html#method-i-find_by
#
# Before: User.where(email: email).first
# After:  User.find_by(email: email)"

# Mentoring principles:
# 1. Explain WHY, not just WHAT
# 2. Point to resources
# 3. Ask questions instead of giving answers
#    "What would happen if this method received nil here?"
# 4. Praise good approaches explicitly
#    "Nice use of early return - keeps the happy path at the bottom"
# 5. Separate blocking vs nitpick comments
#    "[Blocking] ..." vs "[Nit] ..."
```

## 12. Work-Life Balance

### 12.1 Avoiding Burnout

```
Signs of burnout:
- Dreading Monday more than usual
- Finding tasks boring that used to be interesting
- Making more mistakes than normal
- Cynicism about company/work
- Physical symptoms (headaches, fatigue)

Prevention:
1. Set clear work hours (especially remote)
2. Take breaks (Pomodoro technique)
3. Exercise regularly
4. Invest in relationships outside work
5. Pursue hobbies unrelated to coding
6. Say no to unreasonable demands
7. Negotiate realistic deadlines

Recovery:
1. Talk to manager about workload
2. Take real vacation (no Slack/email)
3. Shorter work days temporarily
4. Work on lower-stress tasks
5. Consider role change if needed
```

### 12.2 Sustainable Pace

```ruby
# Software development marathon ไม่ใช่ sprint
# "Sustainable pace" = pace ที่ maintain ได้ตลอดไป

# Practices:
# 1. เขียน tests เสมอ = faster in long run
# 2. Refactor ทันที เมื่อเห็น technical debt
# 3. Document ณ ขณะนั้น ไม่ใช่ "later"
# 4. Say no to feature creep
# 5. Estimate realistically (multiply by 2-3 for uncertainty)

# Time blocking:
# - Deep work: 9am-12pm (no meetings, no Slack)
# - Collaborative work: 1pm-3pm
# - Admin/communication: 3pm-5pm
# - Learning: 30 min/day

# Weekly rhythm:
# Monday: Plan the week, catch up on messages
# Tue-Thu: Deep work, implementation
# Friday: Code review, documentation, retrospective, learning
```

## 13. Next Steps

### 13.1 สร้าง 30-60-90 Day Plan

```
First 30 Days (New Job):
Week 1: Setup, introductions, read docs
Week 2: Small bug fixes to learn codebase
Week 3: First feature with pair programmer
Week 4: Solo feature, first code review as reviewer

Days 31-60:
- Understand business domain deeply
- Identify biggest technical pain points
- Build relationships with PM, design, data
- Contribute to architecture decisions

Days 61-90:
- Drive a significant improvement
- Start mentoring one junior
- Propose/lead a technical initiative
- Establish reputation for quality

First Year Goals:
- Be the go-to person for at least one system
- Drive measurable improvement (performance, reliability)
- Help hire one person
- Give one internal tech talk
```

### 13.2 Personal Roadmap Template

```markdown
## My Rails Career Roadmap - 2025

### Where I Am Now
- Level: Mid-level (~3 years experience)
- Skills: Rails 7, Postgres, Redis, Sidekiq, basic AWS
- Gaps: System design, distributed systems, leadership

### 6-Month Goals
1. Ship a SaaS side project to $1K MRR
2. Contribute to one major open source gem
3. Give one talk at local Ruby meetup
4. Get AWS Solutions Architect certification

### 1-Year Goals
1. Reach senior level at current company (or get senior role elsewhere)
2. Published 12 blog posts
3. 5 merged PRs to major open source projects
4. Speaking at a regional conference

### 3-Year Vision
- Staff engineer at product company
- Known in Ruby community for specialization
- Mentoring team of 3-5 developers
- Earning $200K+ TC

### Actions This Week
- [ ] Deploy side project MVP
- [ ] Submit talk proposal to Ruby meetup
- [ ] Read Rails source code for 1 hour
- [ ] Write blog post draft about N+1 optimization
```

---

## สรุปบทที่ 95

ในบทนี้เราได้เรียนรู้:

1. **Career Levels** - Junior ถึง Staff Engineer, ความรับผิดชอบแต่ละระดับ
2. **Portfolio** - Projects ที่โชว์ skills, README ที่ดี, GitHub profile
3. **Open Source** - วิธีเริ่ม contribute, สร้าง gem
4. **Interview Prep** - Technical questions, system design, behavioral
5. **Remote Work** - Setup, async communication, productivity
6. **Salary** - Research, negotiation, total comp calculation
7. **Personal Branding** - Blog, conference speaking, community
8. **Technical Leadership** - ADRs, code review, mentoring
9. **Work-Life Balance** - Preventing burnout, sustainable pace
10. **Career Planning** - 30-60-90 day plan, personal roadmap

### คำแนะนำสุดท้าย

การเป็น Rails developer ที่ยอดเยี่ยมต้องใช้เวลาและความพยายาม:

- **เรียนรู้ตลอดชีวิต**: Technology เปลี่ยนเร็ว แต่ fundamentals ไม่เปลี่ยน
- **สร้าง relationships**: คนในวงการ Ruby/Rails มักจะ helpful มาก
- **Share knowledge**: เมื่อเรียนรู้อะไรใหม่ สอนคนอื่นต่อ
- **Work on real problems**: Side projects ที่แก้ปัญหาจริงๆ ดีกว่า todo apps
- **Be patient**: Senior engineer ไม่ได้สร้างในวันเดียว

จบ Course Ruby on Rails ทั้ง 95 บทแล้ว! หวังว่าคุณจะได้รับ value จาก course นี้และนำไปใช้ใน career ของคุณได้จริง 🎉
