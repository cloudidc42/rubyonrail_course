# Part 58: Docker และ Containerization สำหรับ Ruby on Rails

## ขั้นตอนที่ 1271-1290: การใช้ Docker กับ Rails

---

## ขั้นตอนที่ 1271: Docker พื้นฐานสำหรับ Ruby Developers

### Docker คืออะไร?

Docker คือ Platform สำหรับ build, ship และ run applications ใน containers

```
┌─────────────────────────────────────────┐
│              Host OS                     │
│  ┌──────────────────────────────────┐   │
│  │         Docker Engine            │   │
│  │  ┌─────────┐  ┌─────────────┐   │   │
│  │  │Container│  │  Container  │   │   │
│  │  │  Rails  │  │ PostgreSQL  │   │   │
│  │  │  App    │  │             │   │   │
│  │  └─────────┘  └─────────────┘   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### Concepts หลัก

**Image** - Template ที่ใช้สร้าง Container (เหมือน class ใน Ruby)
**Container** - Instance ของ Image ที่กำลังรัน (เหมือน object ใน Ruby)
**Dockerfile** - ไฟล์คำสั่งสร้าง Image
**docker-compose** - เครื่องมือจัดการหลาย containers พร้อมกัน
**Registry** - ที่เก็บ Images (เช่น Docker Hub, GitHub Container Registry)

### ติดตั้ง Docker

```bash
# macOS
brew install --cask docker

# Ubuntu
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# ตรวจสอบ version
docker --version
docker compose version
```

### คำสั่ง Docker พื้นฐาน

```bash
# Images
docker pull ruby:3.2-alpine        # ดึง image
docker images                      # รายการ images
docker rmi image_name              # ลบ image

# Containers
docker run -it ruby:3.2 bash       # รัน container แบบ interactive
docker run -d -p 3000:3000 my-app  # รัน container ใน background
docker ps                          # รายการ containers ที่กำลังรัน
docker ps -a                       # รายการทุก containers
docker stop container_name         # หยุด container
docker rm container_name           # ลบ container
docker logs container_name         # ดู logs
docker exec -it container bash     # เข้าไปใน container

# Build
docker build -t my-rails-app .     # build image
docker build -t my-app:v1.0 .      # พร้อม tag

# Network
docker network create app-network  # สร้าง network
docker network ls                  # รายการ networks
```

---

## ขั้นตอนที่ 1272: Dockerfile สำหรับ Rails

### Dockerfile พื้นฐาน

```dockerfile
# Dockerfile
FROM ruby:3.2.2

# ติดตั้ง dependencies
RUN apt-get update -qq && apt-get install -y \
    nodejs \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# ตั้ง working directory
WORKDIR /app

# Copy Gemfile ก่อน (เพื่อ cache optimization)
COPY Gemfile Gemfile.lock ./
RUN bundle install

# Copy โค้ดทั้งหมด
COPY . .

# Precompile assets
RUN bundle exec rails assets:precompile

# Expose port
EXPOSE 3000

# คำสั่งเริ่มต้น
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```

### Dockerfile Production-ready

```dockerfile
# Dockerfile
FROM ruby:3.2.2-slim AS base

# ติดตั้ง system dependencies
RUN apt-get update -qq && apt-get install -y \
    build-essential \
    git \
    libpq-dev \
    libvips \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# ติดตั้ง Node.js และ Yarn
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g yarn

WORKDIR /rails

# ติดตั้ง bundler
ENV BUNDLE_VERSION=2.4.0
RUN gem install bundler:$BUNDLE_VERSION

# Development stage
FROM base AS development

COPY Gemfile Gemfile.lock ./
RUN bundle install --with development test

COPY . .

CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]

# Production stage  
FROM base AS production

# Environment variables
ENV RAILS_ENV=production \
    NODE_ENV=production \
    BUNDLE_WITHOUT="development:test" \
    BUNDLE_DEPLOYMENT=true

# Copy gems config
COPY Gemfile Gemfile.lock ./
RUN bundle install --jobs 4 --retry 3

# Copy app
COPY . .

# Precompile assets
RUN bundle exec rake assets:precompile

# Add non-root user
RUN useradd -m -s /bin/bash app && chown -R app:app /rails
USER app

EXPOSE 3000

CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

---

## ขั้นตอนที่ 1273: docker-compose สำหรับ Development

### docker-compose.yml พื้นฐาน

```yaml
# docker-compose.yml
version: '3.9'

services:
  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-rails}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
      POSTGRES_DB: ${POSTGRES_DB:-myapp_development}
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-rails}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    build:
      context: .
      target: development
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails server -b '0.0.0.0'"
    volumes:
      - .:/rails
      - bundle_cache:/usr/local/bundle
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - RAILS_ENV=development
      - DATABASE_URL=postgresql://rails:password@db:5432/myapp_development
      - REDIS_URL=redis://redis:6379/0
    stdin_open: true
    tty: true

  sidekiq:
    build:
      context: .
      target: development
    command: bundle exec sidekiq
    volumes:
      - .:/rails
      - bundle_cache:/usr/local/bundle
    depends_on:
      - redis
      - db
    environment:
      - RAILS_ENV=development
      - DATABASE_URL=postgresql://rails:password@db:5432/myapp_development
      - REDIS_URL=redis://redis:6379/0

volumes:
  postgres_data:
  redis_data:
  bundle_cache:
```

### ใช้งาน docker-compose

```bash
# เริ่มต้น services ทั้งหมด
docker compose up

# เริ่มใน background
docker compose up -d

# หยุดทุก services
docker compose down

# หยุดและลบ volumes ด้วย
docker compose down -v

# ดู logs
docker compose logs -f web
docker compose logs -f

# รันคำสั่งใน container
docker compose exec web rails console
docker compose exec web rails db:migrate
docker compose exec web bundle exec rspec

# Rebuild image
docker compose build
docker compose up --build

# รัน service เดียว
docker compose run --rm web rails g migration AddPhoneToUsers phone:string
```

---

## ขั้นตอนที่ 1274: Multi-stage Builds

Multi-stage builds ช่วยลดขนาด production image

```dockerfile
# Dockerfile
# === Stage 1: Build Assets ===
FROM node:18-alpine AS assets

WORKDIR /app

COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

COPY app/assets ./app/assets
COPY app/javascript ./app/javascript
COPY config/webpack* ./config/

RUN yarn build && yarn build:css

# === Stage 2: Bundle Gems ===
FROM ruby:3.2.2-slim AS bundle

RUN apt-get update -qq && apt-get install -y \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY Gemfile Gemfile.lock ./
RUN bundle config set --local without 'development test' \
    && bundle install --jobs 4

# === Stage 3: Final Production Image ===
FROM ruby:3.2.2-slim AS production

# Runtime dependencies เท่านั้น
RUN apt-get update -qq && apt-get install -y \
    libpq5 \
    libvips \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy gems จาก bundle stage
COPY --from=bundle /usr/local/bundle /usr/local/bundle

# Copy compiled assets จาก assets stage
COPY --from=assets /app/public/assets ./public/assets
COPY --from=assets /app/public/packs ./public/packs

# Copy app code
COPY . .

# Security: รันด้วย non-root user
RUN useradd --create-home --shell /bin/bash app \
    && chown -R app:app /app
USER app

ENV RAILS_ENV=production \
    RAILS_LOG_TO_STDOUT=true \
    RAILS_SERVE_STATIC_FILES=true

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

---

## ขั้นตอนที่ 1275: Environment Variables

### .env Files

```bash
# .env (สำหรับ local development เท่านั้น!)
POSTGRES_USER=rails
POSTGRES_PASSWORD=password
POSTGRES_DB=myapp_development
DATABASE_URL=postgresql://rails:password@db:5432/myapp_development
REDIS_URL=redis://redis:6379/0
SECRET_KEY_BASE=development_secret_key_base_here
RAILS_ENV=development

# AWS
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
AWS_REGION=ap-southeast-1
S3_BUCKET=myapp-development

# Email
SENDGRID_API_KEY=your_sendgrid_key
MAILER_FROM=noreply@myapp.com

# Stripe
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_SECRET_KEY=sk_test_xxx
```

```bash
# .env.example (commit ไว้ใน git)
POSTGRES_USER=
POSTGRES_PASSWORD=
POSTGRES_DB=
DATABASE_URL=
REDIS_URL=
SECRET_KEY_BASE=
RAILS_ENV=development
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
```

```bash
# .gitignore
.env
.env.local
.env.*.local
```

### docker-compose Override

```yaml
# docker-compose.override.yml (สำหรับ local, ไม่ commit)
version: '3.9'

services:
  web:
    environment:
      - STRIPE_SECRET_KEY=sk_test_your_local_key
      - DEBUG=true
    ports:
      - "3001:3000"  # override port
```

---

## ขั้นตอนที่ 1276: Volume Mounts

Volumes ช่วยให้ข้อมูลอยู่รอดแม้ container ถูกลบ

```yaml
# docker-compose.yml
services:
  web:
    volumes:
      # Bind mount: sync code ระหว่าง host และ container
      - .:/rails
      
      # Named volume: cache gems
      - bundle_cache:/usr/local/bundle
      
      # Named volume: cache node_modules
      - node_modules:/rails/node_modules
      
      # Tmp files
      - rails_tmp:/rails/tmp
      
  db:
    volumes:
      # Named volume: ข้อมูล database
      - postgres_data:/var/lib/postgresql/data
      
      # Init scripts
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  postgres_data:
  redis_data:
  bundle_cache:
  node_modules:
  rails_tmp:
```

### Performance Optimization สำหรับ macOS

```yaml
# docker-compose.yml (macOS)
services:
  web:
    volumes:
      - type: bind
        source: .
        target: /rails
        consistency: cached  # macOS performance boost
```

---

## ขั้นตอนที่ 1277: Docker Networking

```yaml
# docker-compose.yml
version: '3.9'

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # ไม่สามารถเข้าถึงจากภายนอกได้

services:
  nginx:
    image: nginx:alpine
    networks:
      - frontend
    ports:
      - "80:80"
      - "443:443"
    
  web:
    build: .
    networks:
      - frontend  # รับ request จาก nginx
      - backend   # คุยกับ database
    
  db:
    image: postgres:15
    networks:
      - backend  # เข้าถึงได้จาก backend เท่านั้น
    
  redis:
    image: redis:7
    networks:
      - backend
```

### Nginx Configuration

```nginx
# docker/nginx/nginx.conf
upstream rails_app {
  server web:3000;
}

server {
  listen 80;
  server_name localhost;

  root /rails/public;
  
  # Static files
  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  # Proxy to Rails
  location / {
    try_files $uri $uri/index.html $uri.html @rails;
  }

  location @rails {
    proxy_pass http://rails_app;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_redirect off;
  }
}
```

---

## ขั้นตอนที่ 1278: PostgreSQL และ Redis ใน Docker

### PostgreSQL Setup

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      # Performance tuning
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --locale=th_TH.UTF-8"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./docker/postgres/init:/docker-entrypoint-initdb.d
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5
```

```sql
-- docker/postgres/init/01_init.sql
-- สร้าง databases สำหรับทุก environments
CREATE DATABASE myapp_test;
CREATE DATABASE myapp_development;

-- Extensions
\c myapp_development
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";
CREATE EXTENSION IF NOT EXISTS "unaccent";
```

### Redis Setup

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
      - ./docker/redis/redis.conf:/usr/local/etc/redis/redis.conf
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Database Configuration

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  url: <%= ENV['DATABASE_URL'] %>

development:
  <<: *default
  database: myapp_development

test:
  <<: *default
  database: myapp_test
  url: <%= ENV['TEST_DATABASE_URL'] || ENV['DATABASE_URL']&.gsub('development', 'test') %>

production:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 10 } %>
```

---

## ขั้นตอนที่ 1279: Development Workflow

### Makefile สำหรับ Common Commands

```makefile
# Makefile
.PHONY: build up down restart logs shell console db-migrate db-seed test

build:
	docker compose build

up:
	docker compose up -d

up-logs:
	docker compose up

down:
	docker compose down

restart:
	docker compose restart web

logs:
	docker compose logs -f

shell:
	docker compose exec web bash

console:
	docker compose exec web bundle exec rails console

db-create:
	docker compose exec web bundle exec rails db:create

db-migrate:
	docker compose exec web bundle exec rails db:migrate

db-rollback:
	docker compose exec web bundle exec rails db:rollback

db-seed:
	docker compose exec web bundle exec rails db:seed

db-reset:
	docker compose exec web bundle exec rails db:drop db:create db:migrate db:seed

test:
	docker compose exec web bundle exec rspec

test-file:
	docker compose exec web bundle exec rspec $(FILE)

lint:
	docker compose exec web bundle exec rubocop

generate:
	docker compose exec web bundle exec rails generate $(ARGS)

routes:
	docker compose exec web bundle exec rails routes

bundle-install:
	docker compose exec web bundle install

bundle-update:
	docker compose exec web bundle update

clean:
	docker compose down -v
	docker system prune -f
```

```bash
# ใช้งาน
make up           # เริ่ม services
make console      # เข้า Rails console
make db-migrate   # migrate database
make test         # รัน tests
make logs         # ดู logs
```

### entrypoint.sh

```bash
#!/bin/bash
# docker/entrypoint.sh

set -e

# Remove stale server.pid
if [ -f tmp/pids/server.pid ]; then
  rm tmp/pids/server.pid
fi

# Wait for database
until pg_isready -h "$DB_HOST" -p "${DB_PORT:-5432}" -U "$DB_USER"; do
  echo "Waiting for PostgreSQL..."
  sleep 2
done

# Run migrations
bundle exec rails db:create 2>/dev/null || true
bundle exec rails db:migrate

exec "$@"
```

```dockerfile
# Dockerfile
COPY docker/entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```

---

## ขั้นตอนที่ 1280: Health Checks

```ruby
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  def show
    checks = {
      database: database_healthy?,
      redis: redis_healthy?,
      sidekiq: sidekiq_healthy?
    }
    
    all_healthy = checks.values.all?
    
    render json: {
      status: all_healthy ? "ok" : "error",
      checks: checks,
      timestamp: Time.current.iso8601
    }, status: all_healthy ? :ok : :service_unavailable
  end
  
  private
  
  def database_healthy?
    ActiveRecord::Base.connection.execute("SELECT 1")
    true
  rescue
    false
  end
  
  def redis_healthy?
    Redis.new.ping == "PONG"
  rescue
    false
  end
  
  def sidekiq_healthy?
    Sidekiq::ProcessSet.new.size > 0
  rescue
    false
  end
end
```

```ruby
# config/routes.rb
get "/health", to: "health#show"
get "/health/live", to: proc { [200, {}, ["OK"]] }
get "/health/ready", to: "health#show"
```

---

## ขั้นตอนที่ 1281-1285: Production Docker Setup

### docker-compose.production.yml

```yaml
# docker-compose.production.yml
version: '3.9'

services:
  web:
    image: ${DOCKER_IMAGE:-myapp}:${VERSION:-latest}
    restart: unless-stopped
    environment:
      - RAILS_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - SECRET_KEY_BASE=${SECRET_KEY_BASE}
      - RAILS_LOG_TO_STDOUT=true
      - RAILS_SERVE_STATIC_FILES=true
    depends_on:
      - db
      - redis
    networks:
      - app_network
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - web
    networks:
      - app_network

  sidekiq:
    image: ${DOCKER_IMAGE:-myapp}:${VERSION:-latest}
    command: bundle exec sidekiq -C config/sidekiq.yml
    restart: unless-stopped
    environment:
      - RAILS_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    depends_on:
      - redis
      - db
    networks:
      - app_network

networks:
  app_network:
    driver: overlay
```

---

## ขั้นตอนที่ 1286-1290: Docker Best Practices

### .dockerignore

```dockerignore
# .dockerignore
.git
.gitignore
.dockerignore
.env
.env.*

# Development files
README.md
Makefile
docker-compose.override.yml

# Logs
log/*
tmp/*

# Test files
spec/
test/
coverage/

# Assets (จะถูก precompile)
node_modules/
public/assets/
public/packs/

# IDE files
.idea/
.vscode/
*.swp
*.swo
```

### Security Best Practices

```dockerfile
# ใช้ specific version แทน latest
FROM ruby:3.2.2-slim

# ไม่รันด้วย root
RUN groupadd -r app && useradd -r -g app app
USER app

# ลดสิทธิ์ directory
RUN chown -R app:app /rails

# ไม่เก็บ secrets ใน Dockerfile
# BAD:
ENV SECRET_KEY=my_secret

# GOOD: ส่งผ่าน env vars ตอน runtime
```

### Resource Limits

```yaml
# docker-compose.yml
services:
  web:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

---

## แบบฝึกหัดที่ 1-20

### แบบฝึกหัดที่ 1
สร้าง Dockerfile สำหรับ Rails app พื้นฐาน

### แบบฝึกหัดที่ 2
สร้าง docker-compose.yml สำหรับ Rails + PostgreSQL + Redis

### แบบฝึกหัดที่ 3
ทดสอบการ Build และ Run container

```bash
docker compose build
docker compose up
# เข้า http://localhost:3000
```

### แบบฝึกหัดที่ 4
สร้าง Multi-stage Dockerfile สำหรับ production

### แบบฝึกหัดที่ 5
ตั้งค่า Environment Variables ด้วย .env files

### แบบฝึกหัดที่ 6
เพิ่ม Nginx ใน docker-compose เป็น Reverse Proxy

### แบบฝึกหัดที่ 7
สร้าง Makefile สำหรับ common Docker commands

### แบบฝึกหัดที่ 8
เพิ่ม Health Check ใน Dockerfile และ docker-compose

### แบบฝึกหัดที่ 9
ตั้งค่า Volume Mounts สำหรับ development (code sync)

### แบบฝึกหัดที่ 10
แก้ปัญหา Gem caching ใน Docker (bundle_cache volume)

### แบบฝึกหัดที่ 11
เพิ่ม Sidekiq service ใน docker-compose

### แบบฝึกหัดที่ 12
ตั้งค่า Docker Networking สำหรับ Security

### แบบฝึกหัดที่ 13
สร้าง entrypoint.sh ที่ handle startup tasks

### แบบฝึกหัดที่ 14
Deploy Docker image ไป Docker Hub หรือ GitHub Container Registry

### แบบฝึกหัดที่ 15
ตั้งค่า PostgreSQL ใน Docker ให้ persist data

### แบบฝึกหัดที่ 16
รัน Rails tests ใน Docker

```bash
docker compose run --rm web bundle exec rspec
```

### แบบฝึกหัดที่ 17
สร้าง docker-compose.test.yml สำหรับ CI

### แบบฝึกหัดที่ 18
Optimize Docker build time ด้วย layer caching

### แบบฝึกหัดที่ 19
ตั้งค่า Resource Limits สำหรับ containers

### แบบฝึกหัดที่ 20
Build Complete Development Environment:
- Rails app
- PostgreSQL
- Redis
- Sidekiq
- Nginx
- Mailhog (สำหรับ test emails)

---

## สรุป Part 58

ในส่วนนี้เราได้เรียนรู้:
1. Docker concepts และคำสั่งพื้นฐาน
2. การสร้าง Dockerfile สำหรับ Rails
3. docker-compose สำหรับ development
4. Multi-stage builds
5. Environment variables management
6. Volume mounts
7. Docker networking
8. PostgreSQL และ Redis ใน Docker
9. Development workflow
10. Best practices สำหรับ Security และ Performance

Docker ทำให้ development environment เหมือนกันในทุก machine และ production ทำให้ deployment ง่ายและ consistent ขึ้น
