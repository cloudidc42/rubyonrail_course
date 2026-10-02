# Part 65: Progressive Web Apps (PWA) ด้วย Ruby on Rails

## ขั้นตอนที่ 1411-1430: สร้าง PWA กับ Rails

---

## ขั้นตอนที่ 1411: PWA คืออะไร?

Progressive Web App (PWA) คือ web app ที่มีความสามารถเหมือน native app:

- **Installable** - ติดตั้งบน home screen ได้
- **Offline capable** - ทำงานได้แม้ไม่มีเน็ต
- **Fast** - โหลดเร็วด้วย Service Worker caching
- **Engaging** - Push notifications
- **Responsive** - ทำงานได้ทุก device

### เกณฑ์ PWA

1. ใช้ HTTPS
2. มี Web App Manifest
3. มี Service Worker ที่ทำงาน

---

## ขั้นตอนที่ 1412: Service Workers

Service Worker คือ JavaScript ที่รันใน background (แยกจาก main thread)

```javascript
// public/service-worker.js
const CACHE_NAME = "myapp-v1"
const STATIC_ASSETS = [
  "/",
  "/offline",
  "/manifest.json",
  "/icons/icon-192x192.png",
  "/icons/icon-512x512.png"
]

// Install: cache static assets
self.addEventListener("install", (event) => {
  console.log("Service Worker installing...")
  
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      console.log("Caching static assets")
      return cache.addAll(STATIC_ASSETS)
    })
  )
  
  self.skipWaiting()
})

// Activate: clean old caches
self.addEventListener("activate", (event) => {
  console.log("Service Worker activating...")
  
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => {
            console.log("Deleting old cache:", name)
            return caches.delete(name)
          })
      )
    })
  )
  
  self.clients.claim()
})

// Fetch: serve from cache or network
self.addEventListener("fetch", (event) => {
  const url = new URL(event.request.url)
  
  // ไม่ cache Chrome extension requests
  if (url.protocol === "chrome-extension:") return
  
  // ไม่ cache API requests
  if (url.pathname.startsWith("/api/")) {
    event.respondWith(networkFirst(event.request))
    return
  }
  
  // Static assets: Cache First
  if (isStaticAsset(url)) {
    event.respondWith(cacheFirst(event.request))
    return
  }
  
  // HTML pages: Network First
  if (event.request.headers.get("accept")?.includes("text/html")) {
    event.respondWith(networkFirstWithFallback(event.request))
    return
  }
  
  // อื่นๆ: Stale While Revalidate
  event.respondWith(staleWhileRevalidate(event.request))
})

// === Strategies ===

// Cache First: เหมาะกับ static assets
async function cacheFirst(request) {
  const cached = await caches.match(request)
  return cached || fetchAndCache(request)
}

// Network First: เหมาะกับ API, dynamic content
async function networkFirst(request) {
  try {
    const response = await fetch(request)
    if (response.ok) {
      const cache = await caches.open(CACHE_NAME)
      cache.put(request, response.clone())
    }
    return response
  } catch (error) {
    const cached = await caches.match(request)
    return cached || new Response("Network error", { status: 503 })
  }
}

// Network First with offline fallback
async function networkFirstWithFallback(request) {
  try {
    return await fetch(request)
  } catch (error) {
    const cached = await caches.match(request)
    return cached || caches.match("/offline")
  }
}

// Stale While Revalidate
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME)
  const cached = await cache.match(request)
  
  const fetchPromise = fetch(request).then((response) => {
    if (response.ok) cache.put(request, response.clone())
    return response
  })
  
  return cached || fetchPromise
}

async function fetchAndCache(request) {
  const response = await fetch(request)
  if (response.ok) {
    const cache = await caches.open(CACHE_NAME)
    cache.put(request, response.clone())
  }
  return response
}

function isStaticAsset(url) {
  return url.pathname.match(/\.(js|css|png|jpg|gif|svg|ico|woff|woff2)$/)
}
```

---

## ขั้นตอนที่ 1413: Web App Manifest

```json
// public/manifest.json
{
  "name": "My Rails App",
  "short_name": "MyApp",
  "description": "แอปพลิเคชั่น Rails ที่ยอดเยี่ยม",
  "start_url": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#ffffff",
  "theme_color": "#6366f1",
  "lang": "th",
  "categories": ["productivity", "utilities"],
  
  "icons": [
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  
  "shortcuts": [
    {
      "name": "เขียนบทความใหม่",
      "url": "/posts/new",
      "icons": [{"src": "/icons/new-post.png", "sizes": "192x192"}]
    },
    {
      "name": "การแจ้งเตือน",
      "url": "/notifications",
      "icons": [{"src": "/icons/notifications.png", "sizes": "192x192"}]
    }
  ],
  
  "screenshots": [
    {
      "src": "/screenshots/home.png",
      "sizes": "390x844",
      "type": "image/png",
      "label": "หน้าหลัก"
    }
  ]
}
```

### เพิ่ม Manifest ใน Layout

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  
  <%# PWA Meta Tags %>
  <meta name="mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="default">
  <meta name="apple-mobile-web-app-title" content="MyApp">
  <meta name="application-name" content="MyApp">
  <meta name="theme-color" content="#6366f1">
  
  <%# Manifest %>
  <link rel="manifest" href="/manifest.json">
  
  <%# Apple Touch Icons %>
  <link rel="apple-touch-icon" href="/icons/icon-152x152.png">
  <link rel="apple-touch-icon" sizes="180x180" href="/icons/icon-192x192.png">
  
  <%# Favicon %>
  <link rel="icon" type="image/png" sizes="32x32" href="/icons/favicon-32x32.png">
  <link rel="icon" type="image/png" sizes="16x16" href="/icons/favicon-16x16.png">
  
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= stylesheet_link_tag "application", media: "all" %>
</head>
```

---

## ขั้นตอนที่ 1414: Register Service Worker

```javascript
// app/javascript/pwa/service_worker_registration.js
class ServiceWorkerManager {
  static async register() {
    if (!('serviceWorker' in navigator)) {
      console.log("Service Workers ไม่รองรับ browser นี้")
      return
    }
    
    try {
      const registration = await navigator.serviceWorker.register(
        '/service-worker.js',
        { scope: '/' }
      )
      
      console.log("Service Worker registered:", registration.scope)
      
      registration.addEventListener('updatefound', () => {
        const newWorker = registration.installing
        
        newWorker.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            ServiceWorkerManager.showUpdatePrompt(registration)
          }
        })
      })
    } catch (error) {
      console.error("Service Worker registration failed:", error)
    }
  }
  
  static showUpdatePrompt(registration) {
    const prompt = document.createElement("div")
    prompt.className = "update-prompt"
    prompt.innerHTML = `
      <p>มีอัปเดตใหม่!</p>
      <button id="update-btn">อัปเดต</button>
      <button id="dismiss-btn">ภายหลัง</button>
    `
    
    document.body.appendChild(prompt)
    
    document.getElementById("update-btn").addEventListener("click", () => {
      if (registration.waiting) {
        registration.waiting.postMessage({ type: "SKIP_WAITING" })
        window.location.reload()
      }
    })
    
    document.getElementById("dismiss-btn").addEventListener("click", () => {
      prompt.remove()
    })
  }
}

// Register เมื่อ page โหลด
if (document.readyState === "loading") {
  document.addEventListener("DOMContentLoaded", () => ServiceWorkerManager.register())
} else {
  ServiceWorkerManager.register()
}
```

---

## ขั้นตอนที่ 1415: Offline Capabilities

### Offline Page

```erb
<%# app/views/pwa/offline.html.erb %>
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>ไม่มีการเชื่อมต่อ</title>
  <style>
    body { 
      font-family: sans-serif; 
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      background: #f9fafb;
      text-align: center;
      padding: 20px;
    }
    
    .offline-icon { font-size: 64px; }
    h1 { font-size: 24px; color: #374151; }
    p { color: #6b7280; }
    
    button {
      background: #6366f1;
      color: white;
      border: none;
      padding: 12px 24px;
      border-radius: 8px;
      cursor: pointer;
      font-size: 16px;
    }
  </style>
</head>
<body>
  <div>
    <div class="offline-icon">📱</div>
    <h1>ไม่มีการเชื่อมต่ออินเทอร์เน็ต</h1>
    <p>กรุณาตรวจสอบการเชื่อมต่อและลองใหม่อีกครั้ง</p>
    <button onclick="window.location.reload()">ลองใหม่</button>
  </div>
</body>
</html>
```

```ruby
# config/routes.rb
get "/offline", to: "pwa#offline"
```

```ruby
# app/controllers/pwa_controller.rb
class PwaController < ApplicationController
  skip_before_action :authenticate_user!
  
  def offline
    render layout: false
  end
  
  def service_worker
    render "pwa/service_worker", content_type: "application/javascript"
  end
  
  def manifest
    render "pwa/manifest", content_type: "application/manifest+json"
  end
end
```

---

## ขั้นตอนที่ 1416: Push Notifications

### Backend Setup

```ruby
# Gemfile
gem 'web-push'
gem 'serviceworker-rails'
```

```ruby
# config/initializers/web_push.rb
VAPID_KEYS = {
  public_key: ENV['VAPID_PUBLIC_KEY'],
  private_key: ENV['VAPID_PRIVATE_KEY'],
  subject: "mailto:#{ENV['CONTACT_EMAIL']}"
}
```

```bash
# Generate VAPID keys
rails runner "puts WebPush.generate_key.to_s"
```

```ruby
# app/models/push_subscription.rb
class PushSubscription < ApplicationRecord
  belongs_to :user
  
  validates :endpoint, presence: true, uniqueness: true
  
  def as_push_subscription
    {
      endpoint: endpoint,
      keys: {
        p256dh: p256dh_key,
        auth: auth_key
      }
    }
  end
end

# Migration
# add_column :push_subscriptions, :endpoint, :string, null: false
# add_column :push_subscriptions, :p256dh_key, :string
# add_column :push_subscriptions, :auth_key, :string
```

```ruby
# app/controllers/push_subscriptions_controller.rb
class PushSubscriptionsController < ApplicationController
  before_action :authenticate_user!
  
  def create
    subscription = current_user.push_subscriptions.find_or_create_by(
      endpoint: push_params[:endpoint]
    ) do |sub|
      sub.p256dh_key = push_params[:keys][:p256dh]
      sub.auth_key = push_params[:keys][:auth]
    end
    
    render json: { status: "subscribed", id: subscription.id }
  end
  
  def destroy
    subscription = current_user.push_subscriptions.find_by(endpoint: push_params[:endpoint])
    subscription&.destroy
    
    render json: { status: "unsubscribed" }
  end
  
  private
  
  def push_params
    params.require(:subscription).permit(:endpoint, keys: [:p256dh, :auth])
  end
end
```

```ruby
# app/services/push_notification_service.rb
class PushNotificationService
  def self.send_to_user(user:, title:, body:, url: nil, icon: nil)
    user.push_subscriptions.each do |subscription|
      send_notification(subscription, title: title, body: body, url: url, icon: icon)
    end
  end
  
  def self.send_notification(subscription, title:, body:, url: nil, icon: nil)
    payload = {
      title: title,
      body: body,
      url: url || "/",
      icon: icon || "/icons/icon-192x192.png",
      badge: "/icons/badge-72x72.png",
      timestamp: Time.current.to_i
    }.to_json
    
    WebPush.payload_send(
      message: payload,
      endpoint: subscription.endpoint,
      p256dh: subscription.p256dh_key,
      auth: subscription.auth_key,
      vapid: VAPID_KEYS
    )
  rescue WebPush::ExpiredSubscription
    subscription.destroy
  rescue WebPush::Error => e
    Rails.logger.error "Push notification failed: #{e.message}"
  end
end
```

### Frontend Push Notifications

```javascript
// app/javascript/pwa/push_notifications.js
class PushNotificationManager {
  constructor(vapidPublicKey) {
    this.vapidPublicKey = vapidPublicKey
  }
  
  async init() {
    if (!('Notification' in window) || !('PushManager' in window)) {
      console.log("Push notifications ไม่รองรับ browser นี้")
      return
    }
    
    this.registration = await navigator.serviceWorker.ready
  }
  
  async requestPermission() {
    const permission = await Notification.requestPermission()
    return permission === "granted"
  }
  
  async subscribe() {
    const granted = await this.requestPermission()
    if (!granted) return null
    
    try {
      const subscription = await this.registration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: this.urlBase64ToUint8Array(this.vapidPublicKey)
      })
      
      await this.saveSubscription(subscription)
      return subscription
    } catch (error) {
      console.error("Push subscription failed:", error)
      return null
    }
  }
  
  async saveSubscription(subscription) {
    const data = subscription.toJSON()
    
    await fetch('/push_subscriptions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]').content
      },
      body: JSON.stringify({ subscription: data })
    })
  }
  
  async unsubscribe() {
    const subscription = await this.registration.pushManager.getSubscription()
    if (subscription) {
      await subscription.unsubscribe()
      
      await fetch('/push_subscriptions', {
        method: 'DELETE',
        headers: {
          'Content-Type': 'application/json',
          'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]').content
        },
        body: JSON.stringify({ subscription: { endpoint: subscription.endpoint } })
      })
    }
  }
  
  urlBase64ToUint8Array(base64String) {
    const padding = '='.repeat((4 - base64String.length % 4) % 4)
    const base64 = (base64String + padding)
      .replace(/-/g, '+')
      .replace(/_/g, '/')
    
    const rawData = window.atob(base64)
    const outputArray = new Uint8Array(rawData.length)
    
    for (let i = 0; i < rawData.length; ++i) {
      outputArray[i] = rawData.charCodeAt(i)
    }
    return outputArray
  }
}

// Service Worker Push Event Handler (ใน service-worker.js)
self.addEventListener("push", (event) => {
  const data = event.data?.json() || {}
  
  const options = {
    body: data.body,
    icon: data.icon || "/icons/icon-192x192.png",
    badge: data.badge || "/icons/badge-72x72.png",
    data: { url: data.url || "/" },
    actions: data.actions || [],
    requireInteraction: false,
    vibrate: [100, 50, 100]
  }
  
  event.waitUntil(
    self.registration.showNotification(data.title, options)
  )
})

self.addEventListener("notificationclick", (event) => {
  event.notification.close()
  
  const url = event.notification.data?.url || "/"
  
  event.waitUntil(
    clients.matchAll({ type: "window" }).then((windowClients) => {
      const existingClient = windowClients.find(client => 
        client.url === url && "focus" in client
      )
      
      if (existingClient) {
        return existingClient.focus()
      }
      
      return clients.openWindow(url)
    })
  )
})
```

---

## ขั้นตอนที่ 1417: App-like Experience

### Install Prompt

```javascript
// app/javascript/pwa/install_prompt.js
class InstallManager {
  constructor() {
    this.deferredPrompt = null
    this.installButton = null
  }
  
  init() {
    window.addEventListener("beforeinstallprompt", (event) => {
      event.preventDefault()
      this.deferredPrompt = event
      this.showInstallBanner()
    })
    
    window.addEventListener("appinstalled", () => {
      this.deferredPrompt = null
      this.hideInstallBanner()
      console.log("App ติดตั้งสำเร็จ!")
    })
  }
  
  showInstallBanner() {
    const banner = document.getElementById("install-banner")
    if (banner) {
      banner.style.display = "flex"
    }
  }
  
  hideInstallBanner() {
    const banner = document.getElementById("install-banner")
    if (banner) {
      banner.style.display = "none"
    }
  }
  
  async promptInstall() {
    if (!this.deferredPrompt) return
    
    this.deferredPrompt.prompt()
    
    const { outcome } = await this.deferredPrompt.userChoice
    
    if (outcome === "accepted") {
      console.log("User ยอมรับการติดตั้ง")
    } else {
      console.log("User ปฏิเสธการติดตั้ง")
    }
    
    this.deferredPrompt = null
  }
}
```

```erb
<%# app/views/layouts/_install_banner.html.erb %>
<div id="install-banner" class="install-banner" style="display: none">
  <div class="install-info">
    <img src="/icons/icon-48x48.png" alt="App icon" />
    <div>
      <strong>ติดตั้ง MyApp</strong>
      <p>เข้าถึงได้ง่ายจาก home screen</p>
    </div>
  </div>
  
  <div class="install-actions">
    <button onclick="installManager.promptInstall()" class="btn-install">
      ติดตั้ง
    </button>
    <button onclick="document.getElementById('install-banner').style.display='none'" 
            class="btn-dismiss">
      ✕
    </button>
  </div>
</div>
```

---

## ขั้นตอนที่ 1418-1420: Rails PWA Gem

```ruby
# Gemfile (Rails 7.2+)
# PWA มาพร้อม Rails 7.2 แล้ว!

# config/routes.rb
Rails.application.routes.draw do
  # Service worker routes
  get "/service-worker.js", to: "rails/pwa#service_worker", as: :pwa_service_worker
  get "/manifest.json", to: "rails/pwa#manifest", as: :pwa_manifest
end
```

```ruby
# app/views/pwa/service_worker.js.erb
const CACHE_NAME = "myapp-<%= Rails.version %>"
const OFFLINE_PAGE = "<%= pwa_offline_path %>"
```

---

## แบบฝึกหัดที่ 1-15

### แบบฝึกหัดที่ 1
สร้าง Web App Manifest สำหรับ Blog App

### แบบฝึกหัดที่ 2
สร้าง Service Worker พื้นฐานที่ cache static assets

### แบบฝึกหัดที่ 3
สร้าง Offline page และ fallback

### แบบฝึกหัดที่ 4
Implement Cache First strategy สำหรับ images

### แบบฝึกหัดที่ 5
Implement Network First strategy สำหรับ API calls

### แบบฝึกหัดที่ 6
สร้าง Install Prompt/Banner

### แบบฝึกหัดที่ 7
ตั้งค่า Push Notifications

### แบบฝึกหัดที่ 8
ส่ง Push Notification เมื่อมี Comment ใหม่

### แบบฝึกหัดที่ 9
ตั้งค่า Background Sync สำหรับ offline forms

### แบบฝึกหัดที่ 10
ทดสอบ PWA ด้วย Lighthouse

### แบบฝึกหัดที่ 11
Optimize icons สำหรับทุก sizes

### แบบฝึกหัดที่ 12
ตั้งค่า App Shortcuts ใน Manifest

### แบบฝึกหัดที่ 13
Implement Update Detection และ Prompt

### แบบฝึกหัดที่ 14
ตั้งค่า Periodic Background Sync

### แบบฝึกหัดที่ 15
สร้าง Complete PWA Blog App:
- Manifest
- Service Worker (offline capable)
- Push Notifications
- Install prompt
- Lighthouse score > 90

---

## Lighthouse PWA Audit

```bash
# ติดตั้ง Lighthouse CLI
npm install -g lighthouse

# รัน audit
lighthouse https://myapp.com --view

# รัน เฉพาะ PWA
lighthouse https://myapp.com --only-categories=pwa
```

**เกณฑ์ Lighthouse PWA:**
- ✅ Registers a service worker
- ✅ Responds with a 200 when offline  
- ✅ Has a `<meta name="viewport">` tag
- ✅ Contains some content when JavaScript is not available
- ✅ Provides a valid apple-touch-icon
- ✅ Manifest's `display` property is set to standalone, fullscreen, or minimal-ui
- ✅ Has a theme color for the address bar

---

## สรุป Part 65

เราได้เรียนรู้:
1. PWA concepts และ requirements
2. Service Workers และ Caching Strategies
3. Web App Manifest
4. Offline Capabilities
5. Push Notifications
6. App Installation Experience
7. Rails 7.2 PWA support

PWA ช่วยให้ Rails app มี User Experience ที่ดีขึ้นบน mobile โดยไม่ต้องสร้าง native app แยก
