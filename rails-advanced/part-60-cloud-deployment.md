# Part 60: Cloud Deployment สำหรับ Ruby on Rails

## ขั้นตอนที่ 1311-1330: การ Deploy บน Cloud Platforms

---

## ขั้นตอนที่ 1311: Heroku Deployment (Full Guide)

### การเตรียม App

```ruby
# Gemfile
gem 'pg', group: :production
gem 'puma'

group :development do
  gem 'sqlite3'  # local development เท่านั้น
end
```

```ruby
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 2 }
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count
preload_app!
port ENV.fetch("PORT") { 3000 }
environment ENV.fetch("RAILS_ENV") { "development" }
```

```
# Procfile
web: bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -C config/sidekiq.yml
release: bundle exec rails db:migrate
```

### ขั้นตอน Deploy

```bash
# 1. Login Heroku
heroku login

# 2. สร้าง app
heroku create my-app-name
# หรือ connect repo ที่มีอยู่
heroku git:remote -a my-app-name

# 3. เพิ่ม Add-ons
heroku addons:create heroku-postgresql:essential-0 --app my-app-name
heroku addons:create heroku-redis:mini --app my-app-name
heroku addons:create sendgrid:starter --app my-app-name

# 4. ตั้งค่า Environment Variables
heroku config:set RAILS_ENV=production
heroku config:set RAILS_LOG_TO_STDOUT=true
heroku config:set RAILS_SERVE_STATIC_FILES=true
heroku config:set RAILS_MASTER_KEY=$(cat config/master.key)

# 5. Deploy
git push heroku main

# 6. Run migrations
heroku run rails db:migrate

# 7. Seed data (ถ้าต้องการ)
heroku run rails db:seed

# 8. ตรวจสอบ
heroku open
heroku logs --tail
```

### Heroku Dynos

```bash
# ดู dynos ที่รัน
heroku ps

# Scale web dynos
heroku ps:scale web=2
heroku ps:scale worker=1

# ดู usage
heroku ps:utilization

# Restart dynos
heroku restart
heroku dyno:restart web.1
```

### Heroku Config Vars

```bash
# ตั้งค่า
heroku config:set KEY=value
heroku config:set STRIPE_SECRET_KEY=sk_live_xxx AWS_REGION=ap-southeast-1

# ดูทั้งหมด
heroku config

# ดู value เดียว
heroku config:get DATABASE_URL

# ลบ
heroku config:unset KEY
```

### Custom Domain

```bash
# เพิ่ม domain
heroku domains:add www.myapp.com

# ดู DNS target
heroku domains

# ตั้งค่า DNS CNAME ของ domain ให้ชี้ไปที่ DNS target ที่ได้
```

---

## ขั้นตอนที่ 1312: Render.com Deployment

### render.yaml Configuration

```yaml
# render.yaml
services:
  - type: web
    name: my-rails-app
    env: ruby
    region: singapore
    plan: starter
    buildCommand: |
      bundle install
      yarn install
      bundle exec rake assets:precompile
      bundle exec rake assets:clean
    startCommand: bundle exec puma -C config/puma.rb
    healthCheckPath: /health
    
    envVars:
      - key: RAILS_ENV
        value: production
      - key: RAILS_LOG_TO_STDOUT
        value: true
      - key: RAILS_SERVE_STATIC_FILES
        value: true
      - key: DATABASE_URL
        fromDatabase:
          name: my-db
          property: connectionString
      - key: REDIS_URL
        fromService:
          type: redis
          name: my-redis
          property: connectionString
      - key: RAILS_MASTER_KEY
        sync: false  # ตั้งค่าใน Render dashboard
      - key: SECRET_KEY_BASE
        generateValue: true
  
  - type: worker
    name: my-sidekiq
    env: ruby
    buildCommand: bundle install
    startCommand: bundle exec sidekiq -C config/sidekiq.yml
    envVars:
      - key: RAILS_ENV
        value: production
      - key: DATABASE_URL
        fromDatabase:
          name: my-db
          property: connectionString
      - key: REDIS_URL
        fromService:
          type: redis
          name: my-redis
          property: connectionString

databases:
  - name: my-db
    databaseName: myapp_production
    user: myapp
    plan: free
    region: singapore

  - type: redis
    name: my-redis
    plan: free
    region: singapore
    maxmemoryPolicy: allkeys-lru
```

### ขั้นตอน Deploy บน Render

```bash
# 1. Push ไป GitHub
git push origin main

# 2. ใน Render Dashboard:
# - New > Web Service
# - Connect GitHub repository
# - Configure settings
# - Deploy

# หรือใช้ render.yaml (Blueprint)
# New > Blueprint
# Connect repository ที่มี render.yaml
```

---

## ขั้นตอนที่ 1313: Fly.io Deployment

### ติดตั้งและ Setup

```bash
# ติดตั้ง flyctl
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# Initialize app
fly launch
# ตอบคำถาม:
# - App name: my-rails-app
# - Region: sin (Singapore)
# - Database: yes (PostgreSQL)
# - Redis: yes
```

### fly.toml Configuration

```toml
# fly.toml
app = "my-rails-app"
primary_region = "sin"

[build]
  dockerfile = "Dockerfile"
  [build.args]
    RAILS_ENV = "production"

[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"
  RAILS_SERVE_STATIC_FILES = "true"
  PORT = "3000"

[deploy]
  strategy = "rolling"
  release_command = "bundle exec rails db:migrate"

[[services]]
  protocol = "tcp"
  internal_port = 3000
  
  [[services.ports]]
    port = 80
    handlers = ["http"]
    force_https = true
  
  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]
  
  [services.concurrency]
    type = "connections"
    hard_limit = 25
    soft_limit = 20
  
  [[services.http_checks]]
    interval = 10000
    timeout = 2000
    grace_period = "30s"
    method = "get"
    path = "/health"

[mounts]
  source = "myapp_storage"
  destination = "/rails/storage"
```

### Deploy Commands

```bash
# Deploy
fly deploy

# ดู status
fly status

# ดู logs
fly logs

# SSH เข้า container
fly ssh console

# Run Rails console
fly ssh console -C "bin/rails console"

# Scale
fly scale count web=2
fly scale vm shared-cpu-1x

# Secrets
fly secrets set SECRET_KEY_BASE=xxx RAILS_MASTER_KEY=xxx

# Database
fly postgres connect -a my-postgres-db
fly postgres list
```

---

## ขั้นตอนที่ 1314: AWS EC2 + RDS Setup

### AWS Architecture

```
Internet → Route 53 (DNS) → CloudFront (CDN) → Load Balancer
                                                        ↓
                                               EC2 Auto Scaling Group
                                               (Rails App Servers)
                                                        ↓
                                               RDS PostgreSQL (Multi-AZ)
                                               ElastiCache Redis
                                               S3 (File Storage)
```

### EC2 Setup Script

```bash
#!/bin/bash
# scripts/setup_server.sh

# อัปเดต system
sudo apt-get update && sudo apt-get upgrade -y

# ติดตั้ง dependencies
sudo apt-get install -y \
  build-essential \
  libpq-dev \
  libssl-dev \
  libreadline-dev \
  git \
  curl \
  nginx \
  certbot \
  python3-certbot-nginx

# ติดตั้ง Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# ติดตั้ง Yarn
sudo npm install -g yarn

# สร้าง deploy user
sudo adduser deploy
sudo usermod -aG sudo deploy
sudo su - deploy

# ติดตั้ง rbenv
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build

# ติดตั้ง Ruby
rbenv install 3.2.2
rbenv global 3.2.2

# ติดตั้ง Bundler
gem install bundler

echo "Server setup complete!"
```

### RDS Connection

```yaml
# config/database.yml
production:
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  url: <%= ENV['DATABASE_URL'] %>
  # หรือแยก fields
  host: <%= ENV['DB_HOST'] %>
  port: 5432
  database: <%= ENV['DB_NAME'] %>
  username: <%= ENV['DB_USER'] %>
  password: <%= ENV['DB_PASSWORD'] %>
  sslmode: require
```

### S3 Configuration

```ruby
# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: <%= ENV['AWS_REGION'] %>
  bucket: <%= ENV['S3_BUCKET'] %>
  
  # CDN เพื่อ performance
  upload:
    cache_control: "max-age=31536000"
    content_disposition: "inline"
```

```ruby
# config/environments/production.rb
config.active_storage.service = :amazon
```

---

## ขั้นตอนที่ 1315: Environment Variables Management

### Rails Credentials

```bash
# แก้ไข credentials
EDITOR=vim rails credentials:edit

# ดู credentials ทั้งหมด
rails credentials:show

# Environment-specific credentials
rails credentials:edit --environment production
```

```yaml
# config/credentials.yml.enc (ตัวอย่าง)
secret_key_base: your_secret_key

aws:
  access_key_id: your_access_key_id
  secret_access_key: your_secret_access_key

stripe:
  publishable_key: pk_live_xxx
  secret_key: sk_live_xxx

sendgrid:
  api_key: SG.xxx
```

```ruby
# ใช้งาน
Rails.application.credentials.aws.access_key_id
Rails.application.credentials.dig(:aws, :access_key_id)
Rails.application.credentials.stripe[:secret_key]
```

### dotenv-rails Gem

```ruby
# Gemfile
gem 'dotenv-rails', groups: [:development, :test]
```

```bash
# .env
DATABASE_URL=postgresql://localhost/myapp_development
REDIS_URL=redis://localhost:6379
STRIPE_SECRET_KEY=sk_test_xxx
```

### Vault (สำหรับ enterprise)

```ruby
# Gemfile
gem 'vault'

# config/initializers/vault.rb
Vault.address = ENV['VAULT_ADDR']
Vault.token = ENV['VAULT_TOKEN']

# ใช้งาน
secret = Vault.logical.read("secret/data/myapp/production")
database_url = secret.data[:data][:database_url]
```

---

## ขั้นตอนที่ 1316: Database Backup Strategies

### Automated Heroku Backups

```bash
# สร้าง backup ทันที
heroku pg:backups:capture --app my-app

# ดูรายการ backups
heroku pg:backups --app my-app

# Download backup
heroku pg:backups:download --app my-app

# Schedule automated backups
heroku pg:backups:schedule DATABASE_URL \
  --at '02:00 Asia/Bangkok' \
  --app my-app

# Restore backup
heroku pg:backups:restore b001 DATABASE_URL --app my-app
```

### Custom Backup Script

```ruby
# lib/tasks/backup.rake
namespace :db do
  namespace :backup do
    desc "Backup database to S3"
    task to_s3: :environment do
      timestamp = Time.current.strftime("%Y%m%d_%H%M%S")
      backup_file = "backup_#{Rails.env}_#{timestamp}.dump"
      
      # สร้าง backup
      system(
        "pg_dump", 
        "-Fc",
        ENV['DATABASE_URL'],
        "-f", "/tmp/#{backup_file}"
      )
      
      # Upload ไป S3
      s3 = Aws::S3::Resource.new
      bucket = s3.bucket(ENV['BACKUP_S3_BUCKET'])
      bucket.object("backups/#{backup_file}").upload_file("/tmp/#{backup_file}")
      
      # ลบไฟล์ temp
      File.delete("/tmp/#{backup_file}")
      
      # เก็บแค่ 30 วันล่าสุด
      cleanup_old_backups(bucket)
      
      puts "Backup สำเร็จ: #{backup_file}"
    end
    
    private
    
    def cleanup_old_backups(bucket)
      cutoff = 30.days.ago
      
      bucket.objects(prefix: "backups/").each do |obj|
        if obj.last_modified < cutoff
          obj.delete
          puts "ลบ backup เก่า: #{obj.key}"
        end
      end
    end
  end
end
```

---

## ขั้นตอนที่ 1317: Log Management

### Structured Logging

```ruby
# Gemfile
gem 'lograge'
gem 'logstash-event'
```

```ruby
# config/environments/production.rb
config.log_formatter = ::Logger::Formatter.new
config.log_level = :info

config.lograge.enabled = true
config.lograge.formatter = Lograge::Formatters::Json.new
config.lograge.custom_options = lambda do |event|
  {
    request_id: event.payload[:headers]['action_dispatch.request_id'],
    user_id: event.payload[:user_id],
    remote_ip: event.payload[:remote_ip],
    params: event.payload[:params].except('controller', 'action', 'format', 'id')
  }
end
```

### Log to External Service

```ruby
# config/initializers/logging.rb
if Rails.env.production?
  # ส่ง logs ไป Papertrail
  remote_syslog_logger = RemoteSyslogLogger.new(
    ENV['PAPERTRAIL_HOST'],
    ENV['PAPERTRAIL_PORT'].to_i,
    program: Rails.application.class.module_parent_name
  )
  
  Rails.logger = remote_syslog_logger
  ActiveRecord::Base.logger = remote_syslog_logger
end
```

---

## ขั้นตอนที่ 1318: SSL Certificates

### Let's Encrypt บน VPS

```bash
# ติดตั้ง Certbot
sudo apt install certbot python3-certbot-nginx

# สร้าง certificate
sudo certbot --nginx -d myapp.com -d www.myapp.com

# Auto-renew
sudo certbot renew --dry-run

# Crontab สำหรับ auto-renew
sudo crontab -e
# เพิ่ม:
# 0 12 * * * /usr/bin/certbot renew --quiet
```

### Nginx SSL Configuration

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name myapp.com www.myapp.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.com www.myapp.com;
    
    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    
    root /var/www/myapp/current/public;
    
    location ^~ /assets/ {
        gzip_static on;
        expires max;
        add_header Cache-Control public;
    }
    
    location / {
        try_files $uri $uri/ @app;
    }
    
    location @app {
        proxy_pass http://unix:/var/www/myapp/shared/tmp/sockets/puma.sock;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## ขั้นตอนที่ 1319: Domain Configuration

### DNS Records

```
# DNS Records สำหรับ myapp.com
Type    Name    Value                   TTL
A       @       45.32.100.1             300
A       www     45.32.100.1             300
CNAME   api     api-server.example.com  300

# สำหรับ Heroku
CNAME   www     my-app.herokuapp.com    300

# สำหรับ Email (SendGrid)
MX      @       mx.sendgrid.net         300
TXT     @       v=spf1 include:sendgrid.net ~all
```

### Heroku Custom Domain

```bash
# เพิ่ม custom domain
heroku domains:add myapp.com
heroku domains:add www.myapp.com

# ดู DNS target
heroku domains
# output: www.myapp.com → abcde.herokudns.com

# SSL
heroku certs:auto:enable
```

---

## ขั้นตอนที่ 1320: Scaling

### Horizontal Scaling (เพิ่ม servers)

```bash
# Heroku - เพิ่ม dynos
heroku ps:scale web=3

# Fly.io - scale
fly scale count web=3

# Docker Swarm
docker service scale myapp_web=3

# Kubernetes
kubectl scale deployment myapp --replicas=3
```

### Vertical Scaling (เพิ่ม resources)

```bash
# Heroku - upgrade dyno type
heroku ps:resize web=standard-2x

# Fly.io - upgrade VM size
fly scale vm performance-2x

# AWS - change instance type
aws ec2 modify-instance-attribute \
  --instance-id i-xxx \
  --instance-type c5.2xlarge
```

### Auto-scaling

```yaml
# kubernetes HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## แบบฝึกหัดที่ 1-20

### แบบฝึกหัดที่ 1
Deploy Rails app ไป Heroku ครั้งแรก

### แบบฝึกหัดที่ 2
ตั้งค่า PostgreSQL add-on ใน Heroku

### แบบฝึกหัดที่ 3
Deploy ไป Render.com โดยใช้ render.yaml

### แบบฝึกหัดที่ 4
Deploy ไป Fly.io

### แบบฝึกหัดที่ 5
ตั้งค่า Custom Domain พร้อม SSL

### แบบฝึกหัดที่ 6
ตั้งค่า Environment Variables ใน Production

### แบบฝึกหัดที่ 7
ตั้งค่า Database Backups อัตโนมัติ

### แบบฝึกหัดที่ 8
ตั้งค่า S3 สำหรับ File Storage

### แบบฝึกหัดที่ 9
ตั้งค่า CDN ด้วย CloudFront

### แบบฝึกหัดที่ 10
ตั้งค่า Log Management ด้วย Papertrail

### แบบฝึกหัดที่ 11
Scale app ไปยัง multiple instances

### แบบฝึกหัดที่ 12
ตั้งค่า Load Balancer

### แบบฝึกหัดที่ 13
Monitor app ด้วย Heroku metrics

### แบบฝึกหัดที่ 14
ตั้งค่า Scheduled Jobs (Heroku Scheduler)

### แบบฝึกหัดที่ 15
Implement Database Migration Strategy

### แบบฝึกหัดที่ 16
ตั้งค่า Redis cache ใน production

### แบบฝึกหัดที่ 17
ทดสอบ Disaster Recovery procedures

### แบบฝึกหัดที่ 18
ตั้งค่า Staging Environment

### แบบฝึกหัดที่ 19
สร้าง Backup and Restore procedures

### แบบฝึกหัดที่ 20
Deploy complete Rails app พร้อม:
- Custom domain + SSL
- PostgreSQL + Redis
- S3 file storage
- Email service
- Error tracking
- Performance monitoring
- Auto-backups

---

## สรุป Part 60

เราได้เรียนรู้:
1. Heroku deployment ครบขั้นตอน
2. Render.com deployment ด้วย render.yaml
3. Fly.io deployment
4. AWS EC2 + RDS setup
5. Environment variables management
6. Database backup strategies
7. Log management
8. SSL certificates
9. Domain configuration
10. Scaling strategies

การ Deploy ที่ดีต้องคิดถึง reliability, security, scalability และ observability ไปพร้อมกัน
