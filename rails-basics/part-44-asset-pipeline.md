# Part 44: Asset Pipeline ใน Rails

## ขั้นตอนที่ 971-990: การจัดการ Assets ใน Rails 7

---

## ขั้นตอนที่ 971: Asset Pipeline คืออะไร?

Asset Pipeline คือกระบวนการที่ Rails ใช้จัดการ JavaScript, CSS, และ images เพื่อ:
- **Concatenation:** รวมไฟล์หลายๆ ไฟล์เป็นไฟล์เดียว (ลด HTTP requests)
- **Minification:** ลดขนาดไฟล์
- **Fingerprinting:** เพิ่ม hash ใน filename เพื่อ cache busting
- **Preprocessing:** แปลง Sass → CSS, TypeScript → JavaScript

```
app/assets/
├── config/
│   └── manifest.js
├── images/
│   ├── logo.png
│   └── icons/
├── javascripts/ (Rails < 7)
└── stylesheets/
    └── application.css
```

## ขั้นตอนที่ 972: Sprockets vs Propshaft vs esbuild

### Sprockets (ดั้งเดิม)
```ruby
# Gemfile (Rails 6 และก่อนหน้า)
gem 'sprockets-rails'
gem 'sass-rails'
```

### Propshaft (Rails 7 ใหม่)
```ruby
# Gemfile
gem 'propshaft'  # เร็วกว่า Sprockets แต่ features น้อยกว่า
```

### esbuild (สำหรับ JavaScript ซับซ้อน)
```bash
rails new myapp -j esbuild
# หรือ
rails new myapp --javascript esbuild
```

### การเปรียบเทียบ
| Feature | Sprockets | Propshaft | esbuild |
|---------|-----------|-----------|---------|
| Speed | ช้า | เร็ว | เร็วมาก |
| JS Bundling | จำกัด | จำกัด | เต็มรูปแบบ |
| Sass | ✓ | ต้องแยก | ต้องแยก |
| Tree shaking | ✗ | ✗ | ✓ |

## ขั้นตอนที่ 973: JavaScript ใน Rails 7 - Importmap

Rails 7 ใช้ Importmap เป็น default สำหรับ JavaScript

```bash
# สร้าง app ใหม่ด้วย importmap (default)
rails new myapp

# หรือระบุชัดเจน
rails new myapp -j importmap
```

```ruby
# config/importmap.rb
pin "application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js", preload: true
pin_all_from "app/javascript/controllers", under: "controllers"

# เพิ่ม library จาก CDN
pin "lodash", to: "https://cdn.jsdelivr.net/npm/lodash@4.17.21/lodash.min.js"
pin "chart.js", to: "https://cdn.jsdelivr.net/npm/chart.js"
```

```html
<%# config/layouts/application.html.erb %>
<%= javascript_importmap_tags %>
```

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "controllers"

// import library
import _ from "lodash"
import Chart from "chart.js"
```

```bash
# จัดการ packages ด้วย importmap
./bin/importmap pin lodash
./bin/importmap pin chart.js
./bin/importmap pin react react-dom
./bin/importmap unpin lodash
./bin/importmap audit  # ตรวจสอบ security vulnerabilities
```

## ขั้นตอนที่ 974: JavaScript ด้วย esbuild

สำหรับ application ที่ต้องการ TypeScript หรือ NPM packages

```bash
rails new myapp -j esbuild
```

```ruby
# Gemfile
gem "jsbundling-rails"
```

```json
// package.json
{
  "name": "myapp",
  "private": "true",
  "dependencies": {
    "@hotwired/stimulus": "^3.2.1",
    "@hotwired/turbo-rails": "^7.3.0",
    "esbuild": "^0.17.11"
  },
  "scripts": {
    "build": "esbuild app/javascript/*.* --bundle --sourcemap --outdir=app/assets/builds",
    "build:css": "sass ./app/assets/stylesheets/application.sass.scss:./app/assets/builds/application.css --no-source-map --load-path=node_modules"
  }
}
```

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "controllers"

// TypeScript
import { format } from 'date-fns'
import React from 'react'

// CommonJS module
const dayjs = require('dayjs')
```

```bash
# รัน build
yarn build
yarn build --watch  # สำหรับ development

# หรือใช้ foreman
gem 'foreman'
```

```yaml
# Procfile.dev
web: bin/rails server -p 3000
js: yarn build --watch
css: yarn build:css --watch
```

## ขั้นตอนที่ 975: JavaScript ด้วย Vite

```bash
# ใช้ vite_rails gem
bundle add vite_rails
bundle exec vite install
```

```ruby
# config/vite.json
{
  "all": {
    "sourceCodeDir": "app/frontend",
    "watchAdditionalPaths": []
  },
  "development": {
    "autoBuild": true,
    "publicOutputDir": "vite-dev",
    "port": 3036
  },
  "test": {
    "autoBuild": true,
    "publicOutputDir": "vite-test"
  }
}
```

```erb
<%# layouts/application.html.erb %>
<%= vite_client_tag %>
<%= vite_javascript_tag 'application' %>
<%= vite_stylesheet_tag 'application' %>
```

## ขั้นตอนที่ 976: CSS ใน Rails - Sass

```bash
# ติดตั้ง Sass
rails new myapp --css sass

# หรือเพิ่มภายหลัง
bundle add dartsass-rails
./bin/rails dartsass:install
```

```scss
// app/assets/stylesheets/application.scss
@use "variables";
@use "mixins";
@use "base";
@use "components/buttons";
@use "components/forms";
@use "layouts/header";
@use "layouts/footer";
@use "pages/home";
@use "pages/products";
```

```scss
// app/assets/stylesheets/_variables.scss
// สี
$primary: #007bff;
$secondary: #6c757d;
$success: #28a745;
$danger: #dc3545;
$warning: #ffc107;
$info: #17a2b8;

// Typography
$font-family-base: 'Sarabun', sans-serif;
$font-size-base: 16px;
$line-height-base: 1.5;

// Breakpoints
$breakpoint-sm: 576px;
$breakpoint-md: 768px;
$breakpoint-lg: 992px;
$breakpoint-xl: 1200px;

// Spacing
$spacer: 1rem;
$spacers: (
  0: 0,
  1: $spacer * 0.25,
  2: $spacer * 0.5,
  3: $spacer,
  4: $spacer * 1.5,
  5: $spacer * 3
);
```

```scss
// app/assets/stylesheets/_mixins.scss
@mixin respond-to($breakpoint) {
  @if $breakpoint == sm {
    @media (min-width: $breakpoint-sm) { @content; }
  } @else if $breakpoint == md {
    @media (min-width: $breakpoint-md) { @content; }
  } @else if $breakpoint == lg {
    @media (min-width: $breakpoint-lg) { @content; }
  } @else if $breakpoint == xl {
    @media (min-width: $breakpoint-xl) { @content; }
  }
}

@mixin flex-center {
  display: flex;
  align-items: center;
  justify-content: center;
}

@mixin truncate($width: auto) {
  max-width: $width;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

@mixin card-shadow {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: box-shadow 0.3s ease;
  
  &:hover {
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  }
}
```

## ขั้นตอนที่ 977: Tailwind CSS ใน Rails

```bash
# สร้าง app ใหม่ด้วย Tailwind
rails new myapp --css tailwind

# หรือเพิ่มภายหลัง
bundle add tailwindcss-rails
./bin/rails tailwindcss:install
```

```javascript
// tailwind.config.js
module.exports = {
  content: [
    './app/views/**/*.{html,html.erb,erb}',
    './app/helpers/**/*.rb',
    './app/javascript/**/*.js',
    './app/components/**/*.rb',
    './app/components/**/*.html.erb'
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
        }
      },
      fontFamily: {
        sans: ['Sarabun', 'sans-serif'],
        thai: ['Kanit', 'sans-serif']
      }
    }
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
    require('@tailwindcss/aspect-ratio')
  ]
}
```

```css
/* app/assets/stylesheets/application.tailwind.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom base styles */
@layer base {
  body {
    @apply font-sans text-gray-900 bg-white;
  }
  
  h1, h2, h3, h4, h5, h6 {
    @apply font-thai font-semibold;
  }
  
  a {
    @apply text-primary-600 hover:text-primary-800 transition-colors;
  }
}

/* Custom components */
@layer components {
  .btn {
    @apply inline-flex items-center px-4 py-2 rounded-md font-medium transition-colors;
  }
  
  .btn-primary {
    @apply btn bg-primary-600 text-white hover:bg-primary-700 focus:ring-2 focus:ring-primary-500;
  }
  
  .btn-secondary {
    @apply btn bg-gray-100 text-gray-700 hover:bg-gray-200;
  }
  
  .btn-danger {
    @apply btn bg-red-600 text-white hover:bg-red-700;
  }
  
  .form-input {
    @apply block w-full rounded-md border-gray-300 shadow-sm 
           focus:border-primary-500 focus:ring-primary-500;
  }
  
  .card {
    @apply bg-white rounded-lg shadow-md overflow-hidden;
  }
  
  .card-header {
    @apply px-6 py-4 bg-gray-50 border-b border-gray-200;
  }
  
  .card-body {
    @apply px-6 py-4;
  }
}

/* Custom utilities */
@layer utilities {
  .text-thai {
    font-family: 'Sarabun', sans-serif;
  }
  
  .line-clamp-2 {
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }
}
```

## ขั้นตอนที่ 978: Bootstrap ใน Rails

```bash
# Bootstrap ผ่าน CDN (importmap)
./bin/importmap pin bootstrap --download
./bin/importmap pin @popperjs/core --download
```

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "controllers"
import * as bootstrap from "bootstrap"

// Make bootstrap available globally
window.bootstrap = bootstrap
```

```scss
// app/assets/stylesheets/application.scss
// Customize Bootstrap variables
$primary: #0d6efd;
$font-family-base: 'Sarabun', sans-serif;

// Import Bootstrap
@import "bootstrap";
```

```erb
<%# หรือใช้ CDN ใน layout %>
<%# app/views/layouts/application.html.erb %>
<head>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" 
        rel="stylesheet">
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</head>
```

## ขั้นตอนที่ 979: Turbo (Hotwire)

Turbo คือ default JavaScript framework ของ Rails 7

```ruby
# Gemfile (มาพร้อม Rails 7 แล้ว)
gem 'turbo-rails'
gem 'stimulus-rails'
```

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
```

### Turbo Drive
```erb
<%# Turbo Drive - navigate โดยไม่ reload page %>
<%= link_to "ไปหน้าอื่น", other_page_path %>

<%# Disable Turbo สำหรับ link นั้น %>
<%= link_to "หน้านี้ reload ปกติ", other_path, 
    data: { turbo: false } %>
```

### Turbo Frames
```erb
<%# app/views/posts/index.html.erb %>
<turbo-frame id="posts">
  <%= render @posts %>
  <%= paginate @posts %>
</turbo-frame>

<%# link ที่อยู่ใน frame จะ navigate เฉพาะ frame นั้น %>
<turbo-frame id="post-<%= post.id %>">
  <h2><%= post.title %></h2>
  <%= link_to "แก้ไข", edit_post_path(post) %>
</turbo-frame>
```

```ruby
# app/controllers/posts_controller.rb
def edit
  @post = Post.find(params[:id])
  
  # Respond ด้วย turbo frame
  render partial: 'form', locals: { post: @post }
end
```

### Turbo Streams
```ruby
# app/controllers/posts_controller.rb
def create
  @post = Post.new(post_params)
  
  respond_to do |format|
    if @post.save
      format.turbo_stream
      format.html { redirect_to @post }
    else
      format.turbo_stream { 
        render turbo_stream: turbo_stream.replace(
          "post_form",
          partial: "posts/form",
          locals: { post: @post }
        )
      }
      format.html { render :new }
    end
  end
end
```

```erb
<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.append "posts" do %>
  <%= render @post %>
<% end %>

<%= turbo_stream.replace "new_post_form" do %>
  <%= render "form", post: Post.new %>
<% end %>

<%= turbo_stream.update "post_count" do %>
  <%= @total_posts %>
<% end %>
```

## ขั้นตอนที่ 980: Stimulus JavaScript

Stimulus เป็น JavaScript framework ที่ทำงานร่วมกับ Turbo

```javascript
// app/javascript/controllers/hello_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["name", "output"]
  static values = {
    greeting: { type: String, default: "สวัสดี" }
  }
  
  connect() {
    console.log("Hello controller connected!")
  }
  
  disconnect() {
    console.log("Hello controller disconnected!")
  }
  
  greet() {
    this.outputTarget.textContent = 
      `${this.greetingValue}, ${this.nameTarget.value}!`
  }
}
```

```erb
<%# ใช้ Stimulus controller ใน view %>
<div data-controller="hello">
  <input data-hello-target="name" type="text" placeholder="ชื่อของคุณ">
  <button data-action="click->hello#greet">ทักทาย</button>
  <p data-hello-target="output"></p>
</div>
```

```javascript
// app/javascript/controllers/dropdown_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu"]
  static classes = ["open"]
  
  toggle() {
    this.menuTarget.classList.toggle(this.openClass)
  }
  
  hide(event) {
    if (!this.element.contains(event.target)) {
      this.menuTarget.classList.remove(this.openClass)
    }
  }
}
```

```javascript
// app/javascript/controllers/form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["submit"]
  static values = { url: String }
  
  connect() {
    this.boundHandleChange = this.handleChange.bind(this)
    this.element.addEventListener("input", this.boundHandleChange)
  }
  
  disconnect() {
    this.element.removeEventListener("input", this.boundHandleChange)
  }
  
  handleChange(event) {
    this.validate()
  }
  
  async validate() {
    const form = this.element
    const formData = new FormData(form)
    
    const response = await fetch(this.urlValue, {
      method: 'POST',
      body: formData,
      headers: {
        'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]').content
      }
    })
    
    const { valid, errors } = await response.json()
    
    this.submitTarget.disabled = !valid
    this.showErrors(errors)
  }
  
  showErrors(errors) {
    // แสดง errors
    this.element.querySelectorAll('.error-message').forEach(el => el.remove())
    
    Object.entries(errors || {}).forEach(([field, messages]) => {
      const input = this.element.querySelector(`[name*="[${field}]"]`)
      if (input) {
        const errorDiv = document.createElement('div')
        errorDiv.className = 'error-message text-red-500 text-sm mt-1'
        errorDiv.textContent = messages.join(', ')
        input.parentNode.appendChild(errorDiv)
      }
    })
  }
}
```

## ขั้นตอนที่ 981: Stimulus ขั้นสูง

```javascript
// app/javascript/controllers/sortable_controller.js
import { Controller } from "@hotwired/stimulus"
import Sortable from "sortablejs"

export default class extends Controller {
  static values = {
    url: String,
    handle: { type: String, default: ".handle" }
  }
  
  connect() {
    this.sortable = Sortable.create(this.element, {
      handle: this.handleValue,
      animation: 150,
      onEnd: this.onEnd.bind(this)
    })
  }
  
  disconnect() {
    this.sortable.destroy()
  }
  
  async onEnd(event) {
    const ids = [...this.element.children].map(el => el.dataset.id)
    
    await fetch(this.urlValue, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]').content
      },
      body: JSON.stringify({ ids })
    })
  }
}
```

```javascript
// app/javascript/controllers/clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source", "button"]
  
  copy() {
    navigator.clipboard.writeText(this.sourceTarget.value)
      .then(() => {
        const originalText = this.buttonTarget.textContent
        this.buttonTarget.textContent = "คัดลอกแล้ว!"
        this.buttonTarget.disabled = true
        
        setTimeout(() => {
          this.buttonTarget.textContent = originalText
          this.buttonTarget.disabled = false
        }, 2000)
      })
      .catch(err => console.error('Failed to copy:', err))
  }
}
```

## ขั้นตอนที่ 982: Asset Fingerprinting และ Caching

```ruby
# config/environments/production.rb
config.assets.digest = true  # เพิ่ม fingerprint
config.assets.compress = true  # Minify
config.assets.gzip = true  # Gzip compression

# กำหนด assets ที่ต้องการ precompile
config.assets.precompile += ['admin.css', 'admin.js']
```

```bash
# Precompile assets
RAILS_ENV=production rails assets:precompile

# ลบ assets เก่า
RAILS_ENV=production rails assets:clean

# ลบทุกอย่าง
RAILS_ENV=production rails assets:clobber
```

```ruby
# ใช้ asset helpers
image_tag "logo.png"              # => /assets/logo-abc123.png
stylesheet_link_tag "application" # => stylesheet ที่ fingerprint แล้ว
javascript_include_tag "app"      # => javascript ที่ fingerprint แล้ว
asset_path "image.png"            # => path เต็ม
asset_url "image.png"             # => URL เต็ม
```

## ขั้นตอนที่ 983: CDN Integration

```ruby
# config/environments/production.rb
config.action_controller.asset_host = "https://cdn.myapp.com"

# หรือใช้ lambda สำหรับ multiple CDNs
config.action_controller.asset_host = lambda do |source|
  if source.end_with?('.jpg', '.png', '.gif')
    "https://images.cdn.com"
  else
    "https://assets.cdn.com"
  end
end
```

```ruby
# config/environments/production.rb
# CloudFront configuration
config.action_controller.asset_host = ENV['CDN_HOST']
```

```bash
# Upload assets to S3/CloudFront
# ใช้ rake task หรือ CI/CD pipeline
rails assets:precompile
aws s3 sync public/assets s3://myapp-assets/assets --cache-control "max-age=31536000"
```

## ขั้นตอนที่ 984: Image Optimization

```ruby
# Gemfile
gem 'image_processing'
gem 'mini_magick'
# หรือ
gem 'ruby-vips'  # เร็วกว่า
```

```ruby
# config/application.rb
config.active_storage.variant_processor = :vips
```

```erb
<%# สร้าง image variants %>
<%= image_tag @user.avatar.variant(
  resize_to_limit: [300, 300],
  format: :webp,
  quality: 80
) %>

<%# Responsive images %>
<%= image_tag @post.hero_image.variant(resize_to_fill: [1200, 600]),
    srcset: {
      @post.hero_image.variant(resize_to_fill: [600, 300]) => "600w",
      @post.hero_image.variant(resize_to_fill: [900, 450]) => "900w",
      @post.hero_image.variant(resize_to_fill: [1200, 600]) => "1200w"
    },
    sizes: "(max-width: 600px) 100vw, (max-width: 900px) 50vw, 1200px",
    loading: "lazy" %>
```

## ขั้นตอนที่ 985: Web Font Integration

```css
/* app/assets/stylesheets/fonts.css */
/* Google Fonts สำหรับภาษาไทย */
@import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&family=Kanit:wght@400;600&display=swap');

/* หรือ host เอง */
@font-face {
  font-family: 'Sarabun';
  src: url('/assets/fonts/sarabun-regular.woff2') format('woff2'),
       url('/assets/fonts/sarabun-regular.woff') format('woff');
  font-weight: 400;
  font-display: swap;
}

/* ใช้ฟอนต์ */
body {
  font-family: 'Sarabun', sans-serif;
}
```

```erb
<%# Preload fonts ใน head %>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap" 
      rel="stylesheet">
```

## ขั้นตอนที่ 986: JavaScript Testing

```javascript
// test/javascript/controllers/hello_controller.test.js
import { Application } from "@hotwired/stimulus"
import HelloController from "controllers/hello_controller"

describe("HelloController", () => {
  beforeEach(() => {
    document.body.innerHTML = `
      <div data-controller="hello">
        <input data-hello-target="name" value="สมชาย">
        <button data-action="click->hello#greet">ทักทาย</button>
        <p data-hello-target="output"></p>
      </div>
    `
    
    const app = Application.start()
    app.register("hello", HelloController)
  })
  
  it("greets with name", async () => {
    const button = document.querySelector("button")
    button.click()
    
    await new Promise(resolve => setTimeout(resolve, 100))
    
    const output = document.querySelector("[data-hello-target='output']")
    expect(output.textContent).toBe("สวัสดี, สมชาย!")
  })
})
```

## ขั้นตอนที่ 987: ViewComponent กับ Assets

```ruby
# Gemfile
gem 'view_component'
```

```ruby
# app/components/button_component.rb
class ButtonComponent < ViewComponent::Base
  def initialize(text:, variant: :primary, size: :md, **options)
    @text = text
    @variant = variant
    @size = size
    @options = options
  end
  
  def css_classes
    base = "btn"
    variant_class = "btn-#{@variant}"
    size_class = "btn-#{@size}" if @size != :md
    [base, variant_class, size_class, @options[:class]].compact.join(" ")
  end
end
```

```erb
<%# app/components/button_component.html.erb %>
<button class="<%= css_classes %>" <%= @options.except(:class).map { |k, v| "#{k}=\"#{v}\"" }.join(" ") %>>
  <%= @text %>
</button>
```

```erb
<%# การใช้งาน %>
<%= render ButtonComponent.new(text: "บันทึก", variant: :primary) %>
<%= render ButtonComponent.new(text: "ยกเลิก", variant: :secondary, size: :lg) %>
```

## ขั้นตอนที่ 988: Performance สำหรับ Assets

```ruby
# config/environments/production.rb
# HTTP/2 Push
config.action_controller.asset_host = "https://cdn.example.com"

# Preload assets
config.assets.precompile += %w[
  admin.js admin.css
  charts.js
  mobile.css
]
```

```erb
<%# Preload critical assets %>
<% content_for :head do %>
  <link rel="preload" href="<%= font_url('sarabun-regular.woff2') %>" 
        as="font" type="font/woff2" crossorigin>
  <link rel="preload" href="<%= stylesheet_path('critical') %>" as="style">
<% end %>

<%# Lazy load non-critical CSS %>
<link rel="stylesheet" href="<%= stylesheet_path('non-critical') %>"
      media="print" onload="this.media='all'">
<noscript>
  <link rel="stylesheet" href="<%= stylesheet_path('non-critical') %>">
</noscript>
```

```erb
<%# Lazy load images %>
<%= image_tag post.thumbnail, loading: "lazy", decoding: "async" %>
```

## ขั้นตอนที่ 989: Asset Pipeline ใน Development

```ruby
# config/environments/development.rb
# Reload assets on change
config.assets.debug = true

# ไม่ compress ใน development
config.assets.compress = false

# ไม่ fingerprint ใน development
config.assets.digest = false

# Live reload
config.reload_classes_only_on_change = false
```

```ruby
# ใช้ Guard สำหรับ auto-reload
# Gemfile (development)
gem 'guard'
gem 'guard-livereload'
```

```ruby
# Guardfile
guard 'livereload' do
  watch(%r{app/views/.+\.(erb|haml|slim)$})
  watch(%r{app/helpers/.+\.rb})
  watch(%r{public/.+\.(css|js|html)})
  watch(%r{config/locales/.+\.yml})
  watch(%r{(app|vendor)(/assets/\w+/(.+\.(css|js|html|png|jpg))).*}) { |m| "/assets/#{m[3]}" }
end
```

## ขั้นตอนที่ 990: Progressive Web App (PWA)

```ruby
# Gemfile
gem 'serviceworker-rails'
```

```javascript
// app/javascript/service_worker.js
const CACHE_NAME = 'myapp-v1'
const urlsToCache = [
  '/',
  '/offline',
  '/assets/application.css',
  '/assets/application.js'
]

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(urlsToCache))
  )
})

self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request).then(response => {
      if (response) return response
      
      return fetch(event.request).catch(() => {
        return caches.match('/offline')
      })
    })
  )
})
```

```json
// app/views/manifest.json.erb
{
  "name": "My Rails App",
  "short_name": "MyApp",
  "description": "ภาษาไทย Rails Application",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#007bff",
  "icons": [
    {
      "src": "<%= asset_path('icons/icon-72x72.png') %>",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "<%= asset_path('icons/icon-192x192.png') %>",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "<%= asset_path('icons/icon-512x512.png') %>",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

---

## แบบฝึกหัด: Asset Pipeline (20 ข้อ)

### ข้อที่ 1: สร้าง Rails App ด้วย Tailwind
```bash
rails new myapp --css tailwind
rails generate controller Home index
```

### ข้อที่ 2: เพิ่ม Bootstrap ผ่าน Importmap
```bash
./bin/importmap pin bootstrap --download
./bin/importmap pin @popperjs/core --download
```

### ข้อที่ 3: สร้าง Sass Variables และ Mixins
สร้าง `_variables.scss` และ `_mixins.scss` พร้อมใช้งานใน application.scss

### ข้อที่ 4: สร้าง Stimulus Controller
สร้าง controller สำหรับ toggle dark/light mode

**เฉลย:**
```javascript
// app/javascript/controllers/theme_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  toggle() {
    document.documentElement.classList.toggle('dark')
    const isDark = document.documentElement.classList.contains('dark')
    localStorage.setItem('theme', isDark ? 'dark' : 'light')
  }
  
  connect() {
    const saved = localStorage.getItem('theme')
    if (saved === 'dark') {
      document.documentElement.classList.add('dark')
    }
  }
}
```

### ข้อที่ 5: Turbo Frame สำหรับ Infinite Scroll
```erb
<turbo-frame id="posts" data-controller="infinite-scroll"
             data-infinite-scroll-url-value="<%= next_page_url %>">
  <%= render @posts %>
  <%= link_to "โหลดเพิ่ม", next_page_path, 
      data: { turbo_frame: "posts" } %>
</turbo-frame>
```

### ข้อที่ 6: Turbo Stream สำหรับ Real-time Updates
สร้าง chat message ที่ append ด้วย Turbo Stream

### ข้อที่ 7: Asset Precompile
เพิ่ม admin.js และ admin.css ในรายการ precompile

**เฉลย:**
```ruby
config.assets.precompile += ['admin.js', 'admin.css']
```

### ข้อที่ 8: Image Lazy Loading
เพิ่ม lazy loading ให้กับ images ทั้งหมด

**เฉลย:**
```erb
<%= image_tag post.image, loading: "lazy", decoding: "async" %>
```

### ข้อที่ 9: Custom Font สำหรับภาษาไทย
เพิ่ม Sarabun font จาก Google Fonts

### ข้อที่ 10: ViewComponent
สร้าง AlertComponent ที่รับ type และ message

**เฉลย:**
```ruby
class AlertComponent < ViewComponent::Base
  TYPES = %w[success danger warning info].freeze
  
  def initialize(message:, type: :success, dismissible: false)
    @message = message
    @type = type.to_s
    @dismissible = dismissible
  end
end
```

### ข้อที่ 11-20 (แบบสรุป)

**ข้อ 11:** สร้าง Stimulus form validation controller

**ข้อ 12:** เพิ่ม CDN configuration ใน production

**ข้อ 13:** สร้าง responsive image ด้วย srcset

**ข้อ 14:** ทดสอบ Stimulus controller

**ข้อ 15:** สร้าง Turbo Frame สำหรับ modal dialog

**ข้อ 16:** เพิ่ม Service Worker สำหรับ offline support

**ข้อ 17:** สร้าง PWA manifest

**ข้อ 18:** Optimize Sass structure

**ข้อ 19:** สร้าง JavaScript bundle ด้วย esbuild

**ข้อ 20:** ทดสอบ asset pipeline ใน production mode

```javascript
// ข้อ 11: Form Validation
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["field", "error", "submit"]
  
  validate() {
    const errors = []
    
    this.fieldTargets.forEach(field => {
      if (field.required && !field.value) {
        errors.push(`${field.name} จำเป็นต้องกรอก`)
      }
      
      if (field.type === 'email' && field.value) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
        if (!emailRegex.test(field.value)) {
          errors.push("รูปแบบอีเมลไม่ถูกต้อง")
        }
      }
    })
    
    this.showErrors(errors)
    this.submitTarget.disabled = errors.length > 0
  }
  
  showErrors(errors) {
    this.errorTarget.innerHTML = errors
      .map(e => `<p class="text-red-500">${e}</p>`)
      .join('')
  }
}
```

---

## สรุป: Asset Pipeline

| Tool | ใช้เมื่อ |
|------|---------|
| Importmap | JavaScript ง่ายๆ ไม่ต้อง bundling |
| esbuild | TypeScript, NPM packages ซับซ้อน |
| Vite | Modern build tool, Hot reload ดี |
| Sass | CSS preprocessing |
| Tailwind | Utility-first CSS |
| Bootstrap | Component library |
| Turbo | Navigation โดยไม่ reload |
| Stimulus | JavaScript behavior |

**Key Takeaways:**
1. Rails 7 ใช้ Importmap เป็น default ซึ่งเรียบง่ายแต่จำกัด
2. ใช้ esbuild เมื่อต้องการ NPM packages หรือ TypeScript
3. Turbo + Stimulus = Hotwire ซึ่งเป็น modern Rails approach
4. Tailwind CSS เหมาะกับ Rails 7 มาก
5. Precompile assets ใน production เสมอ
