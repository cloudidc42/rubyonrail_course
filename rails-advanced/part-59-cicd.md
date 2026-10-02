# Part 59: CI/CD Pipeline สำหรับ Ruby on Rails

## ขั้นตอนที่ 1291-1310: การตั้งค่า Continuous Integration และ Deployment

---

## ขั้นตอนที่ 1291: CI/CD Concepts

### CI/CD คืออะไร?

**Continuous Integration (CI):**
- ทุกครั้งที่ push code ไปยัง repository จะ trigger การรัน tests อัตโนมัติ
- ตรวจสอบว่าโค้ดผ่าน tests ก่อน merge

**Continuous Deployment (CD):**
- หลังจาก CI ผ่าน จะ deploy อัตโนมัติไปยัง server
- ลด manual work และ human error

```
Developer → Push Code → GitHub → CI Pipeline → Tests → Deploy
                                    ↓
                              Run Tests
                              Lint Check
                              Security Scan
                              Build Docker Image
                                    ↓ (if pass)
                              Deploy to Staging
                                    ↓ (if approve)
                              Deploy to Production
```

### เครื่องมือ CI/CD ที่นิยม

- **GitHub Actions** - Built-in ใน GitHub, ฟรีสำหรับ public repos
- **GitLab CI** - Built-in ใน GitLab
- **CircleCI** - Cloud-based CI
- **Jenkins** - Self-hosted
- **Travis CI** - Cloud-based

---

## ขั้นตอนที่ 1292: GitHub Actions พื้นฐาน

### โครงสร้าง GitHub Actions

```
.github/
└── workflows/
    ├── ci.yml          # Run tests
    ├── security.yml    # Security checks
    └── deploy.yml      # Deployment
```

### Workflow พื้นฐาน

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
      
      - name: Run tests
        run: bundle exec rspec
```

---

## ขั้นตอนที่ 1293: รัน Tests ใน CI

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  RAILS_ENV: test
  DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test
  REDIS_URL: redis://localhost:6379/0

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
          POSTGRES_DB: myapp_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2.2'
          bundler-cache: true
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'yarn'
      
      - name: Install JS dependencies
        run: yarn install --frozen-lockfile
      
      - name: Build assets
        run: yarn build && yarn build:css
      
      - name: Setup database
        run: |
          bundle exec rails db:create
          bundle exec rails db:schema:load
      
      - name: Run RSpec
        run: bundle exec rspec --format progress --format RspecJunitFormatter --out tmp/rspec_results.xml
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: rspec-results
          path: tmp/rspec_results.xml
      
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: coverage/
```

---

## ขั้นตอนที่ 1294: Linting และ Security Checks

```yaml
# .github/workflows/lint.yml
name: Lint and Security

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  rubocop:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2.2'
          bundler-cache: true
      
      - name: Run RuboCop
        run: bundle exec rubocop --format github
  
  brakeman:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2.2'
          bundler-cache: true
      
      - name: Run Brakeman Security Scanner
        run: |
          bundle exec brakeman \
            --format json \
            --output tmp/brakeman_results.json \
            --no-pager \
            --exit-on-warn
      
      - name: Upload Brakeman results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: brakeman-results
          path: tmp/brakeman_results.json
  
  bundle-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2.2'
          bundler-cache: true
      
      - name: Audit Gems for Vulnerabilities
        run: |
          gem install bundler-audit
          bundle-audit check --update
```

### RuboCop Configuration

```yaml
# .rubocop.yml
require:
  - rubocop-rails
  - rubocop-rspec
  - rubocop-performance

AllCops:
  TargetRubyVersion: 3.2
  NewCops: enable
  Exclude:
    - 'db/**/*'
    - 'bin/**/*'
    - 'config/**/*'
    - 'vendor/**/*'
    - 'node_modules/**/*'

Style/StringLiterals:
  EnforcedStyle: double_quotes

Rails/FilePath:
  EnforcedStyle: slashes

Metrics/BlockLength:
  Exclude:
    - 'spec/**/*'
    - 'config/routes.rb'

Metrics/MethodLength:
  Max: 20

RSpec/ExampleLength:
  Max: 30
```

---

## ขั้นตอนที่ 1295: Database Setup ใน CI

```yaml
# .github/workflows/ci.yml (database section)
- name: Setup database
  env:
    DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test
  run: |
    # Wait for postgres to be ready
    until pg_isready -h localhost -p 5432 -U postgres; do
      echo "Waiting for postgres..."
      sleep 2
    done
    
    bundle exec rails db:create RAILS_ENV=test
    bundle exec rails db:schema:load RAILS_ENV=test
    
    # Seed test data if needed
    # bundle exec rails db:seed RAILS_ENV=test

- name: Verify database
  run: |
    bundle exec rails runner "puts 'Database connection OK'"
```

### Test Database Configuration

```yaml
# config/database.yml
test:
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  url: <%= ENV['DATABASE_URL'] || 'postgresql://localhost/myapp_test' %>
  
  # ใน CI อาจต้องการ
  timeout: 5000
  connect_timeout: 10
```

---

## ขั้นตอนที่ 1296: Deployment Automation

### Deploy ไป Heroku

```yaml
# .github/workflows/deploy-heroku.yml
name: Deploy to Heroku

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: [test, lint]  # รอให้ tests ผ่านก่อน
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Deploy to Heroku
        uses: akhileshns/heroku-deploy@v3.13.15
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: ${{ secrets.HEROKU_APP_NAME }}
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
          
      - name: Run migrations on Heroku
        run: |
          heroku run bundle exec rails db:migrate \
            --app ${{ secrets.HEROKU_APP_NAME }}
        env:
          HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
```

### Deploy ไป Render

```yaml
# .github/workflows/deploy-render.yml
name: Deploy to Render

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - name: Trigger Render Deploy
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.RENDER_API_KEY }}" \
            -H "Content-Type: application/json" \
            "https://api.render.com/v1/services/${{ secrets.RENDER_SERVICE_ID }}/deploys" \
            -d '{"clearCache": false}'
      
      - name: Wait for deployment
        run: |
          echo "Waiting for deployment to complete..."
          sleep 60
          
          # Check deployment status
          STATUS=$(curl -s \
            -H "Authorization: Bearer ${{ secrets.RENDER_API_KEY }}" \
            "https://api.render.com/v1/services/${{ secrets.RENDER_SERVICE_ID }}/deploys?limit=1" \
            | jq -r '.[0].status')
          
          echo "Deploy status: $STATUS"
          if [ "$STATUS" != "live" ]; then
            echo "Deployment may still be in progress"
          fi
```

---

## ขั้นตอนที่ 1297: Heroku Deployment

### การติดตั้งครั้งแรก

```bash
# ติดตั้ง Heroku CLI
curl https://cli-assets.heroku.com/install.sh | sh

# Login
heroku login

# สร้าง app
heroku create my-rails-app

# เพิ่ม PostgreSQL
heroku addons:create heroku-postgresql:essential-0

# เพิ่ม Redis
heroku addons:create heroku-redis:mini

# ตั้งค่า Environment Variables
heroku config:set RAILS_ENV=production
heroku config:set RAILS_MASTER_KEY=$(cat config/master.key)
heroku config:set RAILS_LOG_TO_STDOUT=true
heroku config:set RAILS_SERVE_STATIC_FILES=true

# Deploy
git push heroku main

# Run migrations
heroku run rails db:migrate

# ดู logs
heroku logs --tail
```

### Procfile

```
# Procfile
web: bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -C config/sidekiq.yml
release: bundle exec rails db:migrate
```

### Heroku-specific Configuration

```ruby
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 2 }
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

preload_app!

port ENV.fetch("PORT") { 3000 }
environment ENV.fetch("RAILS_ENV") { "development" }

on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end
```

---

## ขั้นตอนที่ 1298: Render Deployment

```yaml
# render.yaml
services:
  - type: web
    name: my-rails-app
    env: ruby
    buildCommand: |
      bundle install
      bundle exec rake assets:precompile
    startCommand: bundle exec puma -C config/puma.rb
    envVars:
      - key: RAILS_ENV
        value: production
      - key: RAILS_LOG_TO_STDOUT
        value: true
      - key: RAILS_SERVE_STATIC_FILES
        value: true
      - key: DATABASE_URL
        fromDatabase:
          name: my-rails-db
          property: connectionString
      - key: REDIS_URL
        fromService:
          name: my-redis
          type: redis
          property: connectionString
    
  - type: worker
    name: my-rails-worker
    env: ruby
    buildCommand: bundle install
    startCommand: bundle exec sidekiq -C config/sidekiq.yml
    envVars:
      - key: RAILS_ENV
        value: production

databases:
  - name: my-rails-db
    databaseName: myapp_production
    user: myapp

  - type: redis
    name: my-redis
    maxmemoryPolicy: allkeys-lru
```

---

## ขั้นตอนที่ 1299: Capistrano

Capistrano ใช้ deploy ไปยัง VPS/Server โดยตรง

```ruby
# Gemfile
group :development do
  gem 'capistrano', '~> 3.18'
  gem 'capistrano-rails', '~> 1.6'
  gem 'capistrano-bundler', '~> 2.1'
  gem 'capistrano-rbenv', '~> 2.2'
  gem 'capistrano3-puma', '~> 5.2'
  gem 'capistrano-sidekiq', '~> 2.3'
end
```

```bash
bundle exec cap install
```

```ruby
# Capfile
require 'capistrano/setup'
require 'capistrano/deploy'
require 'capistrano/rbenv'
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
require 'capistrano/puma'
require 'capistrano/sidekiq'

# Include custom tasks
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

```ruby
# config/deploy.rb
lock "~> 3.18.0"

set :application, "myapp"
set :repo_url, "git@github.com:username/myapp.git"
set :branch, :main
set :deploy_to, "/var/www/myapp"

set :rbenv_type, :user
set :rbenv_ruby, '3.2.2'
set :rbenv_path, "~/.rbenv"

set :bundle_without, %w[development test].join(' ')
set :bundle_binstubs, nil

set :linked_files, %w[config/master.key .env]
set :linked_dirs, %w[log tmp/pids tmp/cache tmp/sockets public/system storage]

set :puma_threads, [4, 16]
set :puma_workers, 2
set :puma_bind, "unix://#{shared_path}/tmp/sockets/puma.sock"
set :puma_state, "#{shared_path}/tmp/pids/puma.state"
set :puma_pid, "#{shared_path}/tmp/pids/puma.pid"
set :puma_access_log, "#{release_path}/log/puma_access.log"
set :puma_error_log, "#{release_path}/log/puma_error.log"

namespace :deploy do
  desc "ตรวจสอบ requirements ก่อน deploy"
  task :check_requirements do
    on roles(:app), in: :sequence do
      execute :ruby, '--version'
      execute :bundle, '--version'
    end
  end
  
  after :finishing, :cleanup_assets do
    on roles(:web) do
      execute :rake, "assets:clean"
    end
  end
end
```

```ruby
# config/deploy/production.rb
server "YOUR_SERVER_IP", user: "deploy", roles: %w[app db web]

set :ssh_options, {
  keys: %w[~/.ssh/id_rsa],
  forward_agent: true,
  auth_methods: %w[publickey]
}
```

```bash
# Deploy
bundle exec cap production deploy

# Rollback
bundle exec cap production deploy:rollback
```

---

## ขั้นตอนที่ 1300: Zero-Downtime Deployment

### Blue-Green Deployment

```yaml
# .github/workflows/deploy-blue-green.yml
name: Blue-Green Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker Image
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker tag myapp:${{ github.sha }} registry.example.com/myapp:${{ github.sha }}
          docker push registry.example.com/myapp:${{ github.sha }}
      
      - name: Deploy Green Environment
        run: |
          # Deploy new version to "green" environment
          kubectl set image deployment/myapp-green \
            web=registry.example.com/myapp:${{ github.sha }}
          
          # Wait for rollout
          kubectl rollout status deployment/myapp-green
      
      - name: Health Check Green
        run: |
          # ตรวจสอบว่า green environment ทำงานได้
          response=$(curl -s -o /dev/null -w "%{http_code}" https://green.myapp.com/health)
          if [ $response -ne 200 ]; then
            echo "Health check failed"
            exit 1
          fi
      
      - name: Switch Traffic to Green
        run: |
          # เปลี่ยน traffic ไปที่ green
          kubectl patch service myapp-service \
            -p '{"spec":{"selector":{"version":"green"}}}'
      
      - name: Cleanup Blue
        run: |
          # ลบ blue environment เก่า (หลังจาก stable)
          sleep 60  # รอให้แน่ใจว่า green stable
          kubectl scale deployment/myapp-blue --replicas=0
```

### Rolling Updates

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # เพิ่ม pod ได้สูงสุด 1 ใหม่ขณะ update
      maxUnavailable: 0   # ต้องมี pod ให้บริการตลอดเวลา
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: web
          image: myapp:latest
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
```

---

## ขั้นตอนที่ 1301-1305: Complete CI/CD Pipeline

```yaml
# .github/workflows/complete-pipeline.yml
name: Complete CI/CD Pipeline

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

env:
  RAILS_ENV: test
  NODE_ENV: test

jobs:
  # Job 1: Lint
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2.2'
          bundler-cache: true
      - run: bundle exec rubocop --format github
      - run: bundle exec standardrb --no-fix
  
  # Job 2: Security
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2.2'
          bundler-cache: true
      - name: Security scan
        run: |
          bundle exec brakeman -q -w1 --no-pager
          bundle-audit check --update
  
  # Job 3: Tests
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: myapp_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ["5432:5432"]
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ["6379:6379"]
    
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2.2'
          bundler-cache: true
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'yarn'
      - run: yarn install
      - run: yarn build
      - run: bundle exec rails db:create db:schema:load
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test
      - run: bundle exec rspec
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test
          REDIS_URL: redis://localhost:6379/0
      - uses: codecov/codecov-action@v4
        with:
          file: coverage/lcov.info
  
  # Job 4: Build Docker Image
  build:
    needs: [lint, security, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          target: production
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  # Job 5: Deploy to Production
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy to Heroku
        uses: akhileshns/heroku-deploy@v3.13.15
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: ${{ secrets.HEROKU_APP_NAME }}
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
      
      - name: Run DB migrations
        run: |
          heroku run rails db:migrate --app ${{ secrets.HEROKU_APP_NAME }}
        env:
          HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
      
      - name: Notify deployment
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "Deployment to production succeeded! :rocket:",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Deployment* :white_check_mark:\n*Commit:* ${{ github.sha }}\n*By:* ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## ขั้นตอนที่ 1306-1310: Secrets Management

### GitHub Secrets

```yaml
# .github/workflows/deploy.yml
steps:
  - name: Use secrets
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
      SECRET_KEY_BASE: ${{ secrets.SECRET_KEY_BASE }}
      STRIPE_SECRET_KEY: ${{ secrets.STRIPE_SECRET_KEY }}
    run: bundle exec rails db:migrate
```

### การ Setup Secrets ใน GitHub

```
GitHub Repository → Settings → Secrets and variables → Actions
→ New repository secret

ตัวอย่าง secrets:
- HEROKU_API_KEY
- HEROKU_APP_NAME
- HEROKU_EMAIL
- SLACK_WEBHOOK_URL
- DATABASE_URL
- SECRET_KEY_BASE
```

---

## แบบฝึกหัดที่ 1-20

### แบบฝึกหัดที่ 1
สร้าง Basic CI Workflow ด้วย GitHub Actions

### แบบฝึกหัดที่ 2
เพิ่ม PostgreSQL service ใน CI workflow

### แบบฝึกหัดที่ 3
เพิ่ม RuboCop linting ใน pipeline

### แบบฝึกหัดที่ 4
เพิ่ม Brakeman security scan

### แบบฝึกหัดที่ 5
ตั้งค่า test coverage reporting

### แบบฝึกหัดที่ 6
Deploy app ไป Heroku อัตโนมัติหลัง push to main

### แบบฝึกหัดที่ 7
เพิ่ม staging environment ก่อน production

### แบบฝึกหัดที่ 8
สร้าง Notification เมื่อ deploy สำเร็จหรือล้มเหลว

### แบบฝึกหัดที่ 9
ตั้งค่า Branch Protection Rules

### แบบฝึกหัดที่ 10
สร้าง Environment-specific deployments

### แบบฝึกหัดที่ 11
เพิ่ม bundle-audit ใน security checks

### แบบฝึกหัดที่ 12
Deploy ด้วย Docker image ไปยัง Render

### แบบฝึกหัดที่ 13
ตั้งค่า Capistrano สำหรับ VPS deployment

### แบบฝึกหัดที่ 14
Implement Database migration ใน CI/CD

### แบบฝึกหัดที่ 15
ตั้งค่า GitHub Environments ที่ต้องการ approval

### แบบฝึกหัดที่ 16
สร้าง Rollback workflow

### แบบฝึกหัดที่ 17
เพิ่ม Performance testing ใน pipeline

### แบบฝึกหัดที่ 18
ตั้งค่า Caching สำหรับ Bundle install และ Node modules

### แบบฝึกหัดที่ 19
สร้าง Matrix testing สำหรับหลาย Ruby versions

```yaml
strategy:
  matrix:
    ruby-version: ['3.1', '3.2', '3.3']
```

### แบบฝึกหัดที่ 20
สร้าง Complete CI/CD Pipeline สำหรับ Rails app ที่รวม:
- Tests (RSpec)
- Linting (RuboCop)
- Security (Brakeman)
- Docker build
- Deploy to production
- Slack notifications

---

## สรุป Part 59

เราได้เรียนรู้:
1. CI/CD concepts และประโยชน์
2. GitHub Actions workflow syntax
3. รัน tests ใน CI พร้อม database services
4. Linting และ security checks
5. Heroku และ Render deployment
6. Capistrano สำหรับ VPS
7. Zero-downtime deployment
8. Complete pipeline

CI/CD ช่วยให้ทีม deliver software ได้เร็วและ confident มากขึ้น
