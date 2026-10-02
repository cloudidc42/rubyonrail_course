# คู่มือติดตั้ง Ruby และ Rails ฉบับสมบูรณ์

> สำหรับ Linux (Ubuntu/Debian), macOS, และ Windows (WSL2)

---

## 1. ติดตั้งบน Linux (Ubuntu/Debian)

### 1.1 ติดตั้ง Dependencies

```bash
sudo apt update
sudo apt install -y \
  git curl libssl-dev libreadline-dev zlib1g-dev \
  autoconf bison build-essential libyaml-dev \
  libreadline-dev libncurses5-dev libffi-dev libgdbm-dev \
  libpq-dev libsqlite3-dev
```

### 1.2 ติดตั้ง rbenv (แนะนำ)

```bash
# ติดตั้ง rbenv
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc

# ติดตั้ง ruby-build plugin
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build

# ตรวจสอบ
rbenv --version
```

### 1.3 ติดตั้ง Ruby

```bash
# ดู version ที่มีให้ติดตั้ง
rbenv install --list

# ติดตั้ง Ruby 3.3.0 (ล่าสุด)
rbenv install 3.3.0

# ตั้งเป็น default
rbenv global 3.3.0

# ตรวจสอบ
ruby --version
# ruby 3.3.0 (2023-12-25 revision ...) [x86_64-linux]
```

### 1.4 ติดตั้ง Rails

```bash
# อัปเดต gem ก่อน
gem update --system
gem install bundler

# ติดตั้ง Rails
gem install rails

# ตรวจสอบ
rails --version
# Rails 7.1.x
```

---

## 2. ติดตั้งบน macOS

### 2.1 ติดตั้ง Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2.2 ติดตั้ง Dependencies

```bash
brew install rbenv ruby-build openssl readline libyaml gmp libpq
```

### 2.3 ติดตั้ง rbenv และ Ruby

```bash
# ติดตั้ง rbenv
brew install rbenv

# เพิ่มใน shell config
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
source ~/.zshrc

# ติดตั้ง Ruby
rbenv install 3.3.0
rbenv global 3.3.0

# ตรวจสอบ
ruby --version
```

### 2.4 ติดตั้ง Rails

```bash
gem install rails
rails --version
```

---

## 3. ติดตั้งบน Windows (WSL2)

### 3.1 ติดตั้ง WSL2

```powershell
# ใน PowerShell (Admin)
wsl --install
# รีสตาร์ทเครื่อง

# ติดตั้ง Ubuntu
wsl --install -d Ubuntu
```

### 3.2 ใน Ubuntu (WSL2)

```bash
# ทำเหมือน Linux (Ubuntu) ด้านบน
sudo apt update && sudo apt upgrade -y
# ... ทำตามขั้นตอน Linux
```

---

## 4. ติดตั้ง Database

### 4.1 PostgreSQL (แนะนำสำหรับ Production)

```bash
# Ubuntu/Debian
sudo apt install postgresql postgresql-contrib libpq-dev

# เริ่ม service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# สร้าง user สำหรับ Rails
sudo -u postgres createuser --superuser $USER

# macOS
brew install postgresql@16
brew services start postgresql@16
```

### 4.2 SQLite (สำหรับ Development/Learning)

```bash
# Ubuntu
sudo apt install sqlite3 libsqlite3-dev

# macOS
brew install sqlite
```

### 4.3 Redis (สำหรับ Sidekiq/Cache)

```bash
# Ubuntu
sudo apt install redis-server
sudo systemctl start redis

# macOS
brew install redis
brew services start redis
```

---

## 5. สร้าง Rails App แรก

### 5.1 Rails new

```bash
# สร้าง app ด้วย SQLite (ง่ายสุด)
rails new myapp
cd myapp

# สร้าง app ด้วย PostgreSQL
rails new myapp --database=postgresql
cd myapp

# สร้าง API app
rails new myapi --api --database=postgresql

# สร้าง app พร้อม Tailwind CSS
rails new myapp --css=tailwind

# สร้าง app แบบ minimal
rails new myapp --skip-test --skip-action-mailer --skip-active-job
```

### 5.2 Setup Database

```bash
# แก้ config/database.yml (PostgreSQL)
# development:
#   adapter: postgresql
#   database: myapp_development
#   username: your_username
#   password:

# สร้าง database
rails db:create

# รัน migrations
rails db:migrate
```

### 5.3 เริ่ม Server

```bash
rails server
# หรือ
rails s

# เปิดที่ http://localhost:3000
```

---

## 6. VS Code Setup

### 6.1 Extensions ที่แนะนำ

```
ruby-lsp                    # Ruby Language Server
Rails         (by Hridoy)   # Rails helpers
endwise                     # Auto end
erb             (by erb)    # ERB highlighting
GitLens                     # Git integration
Prettier                    # Code formatter
```

### 6.2 settings.json

```json
{
  "[ruby]": {
    "editor.defaultFormatter": "Shopify.ruby-lsp",
    "editor.formatOnSave": true,
    "editor.tabSize": 2
  },
  "[erb]": {
    "editor.tabSize": 2
  },
  "editor.tabSize": 2,
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true
}
```

---

## 7. เครื่องมือที่แนะนำ

### 7.1 Gems ที่ควรรู้จัก

```ruby
# Gemfile ทั่วไปสำหรับ Development

group :development do
  gem 'pry-rails'         # Better console
  gem 'better_errors'     # Better error pages
  gem 'binding_of_caller' # Works with better_errors
  gem 'letter_opener'     # Preview emails in browser
  gem 'bullet'            # Detect N+1 queries
  gem 'annotate'          # Add schema comments to models
  gem 'rubocop'           # Code linter
  gem 'rubocop-rails'     # Rails specific rules
end

group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'pry-byebug'        # Debugger
end

group :test do
  gem 'shoulda-matchers'
  gem 'capybara'
  gem 'selenium-webdriver'
  gem 'simplecov'
end
```

### 7.2 Database GUI Tools

- **TablePlus** — ดีที่สุด (Mac/Windows/Linux)
- **DBeaver** — ฟรี, รองรับหลาย database
- **pgAdmin** — สำหรับ PostgreSQL โดยเฉพาะ
- **DB Browser for SQLite** — สำหรับ SQLite

### 7.3 API Testing

- **Insomnia** — แนะนำ (ฟรี)
- **Postman** — ยอดนิยม
- **httpie** — Command line

---

## 8. Troubleshooting ที่พบบ่อย

### Error: `Could not find gem 'rails'`

```bash
# ตรวจสอบ Ruby version
ruby --version

# ติดตั้ง bundler ใหม่
gem install bundler
bundle install
```

### Error: `PG::ConnectionBad`

```bash
# ตรวจสอบว่า PostgreSQL รันอยู่
sudo systemctl status postgresql

# สร้าง user ใหม่
sudo -u postgres createuser --superuser $(whoami)
```

### Error: `Could not find a JavaScript runtime`

```bash
# ติดตั้ง Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs

# หรือใช้ execjs gem
gem install execjs
```

### Port 3000 already in use

```bash
# หา process ที่ใช้ port 3000
lsof -i :3000
kill -9 <PID>

# หรือรัน Rails บน port อื่น
rails s -p 3001
```

---

## 9. Version Reference

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Ruby | 3.0+ | 3.3.x |
| Rails | 7.0+ | 7.1.x |
| PostgreSQL | 12+ | 16.x |
| Node.js | 14+ | 20.x |
| Redis | 6+ | 7.x |

---

*อัปเดต: ตุลาคม 2026*
