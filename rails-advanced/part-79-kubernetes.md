# Part 79: Kubernetes and Production Operations

## ขั้นตอนที่ 1701-1720

---

## ขั้นตอนที่ 1701: Docker Basics สำหรับ Rails

### Dockerfile สำหรับ Rails Application

```dockerfile
# Dockerfile
ARG RUBY_VERSION=3.3.0
FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base

# ติดตั้ง dependencies พื้นฐาน
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      curl \
      libvips \
      postgresql-client && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Rails environment
ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development test"

# Build stage - ติดตั้ง gems และ precompile assets
FROM base AS build

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential \
      git \
      libpq-dev \
      node-gyp \
      pkg-config \
      python-is-python3 && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Node.js สำหรับ asset pipeline
ARG NODE_VERSION=20.11.1
ARG YARN_VERSION=1.22.21
ENV PATH=/usr/local/node/bin:$PATH

RUN curl -sL https://github.com/nodenv/node-build/archive/master.tar.gz | tar xz -C /tmp/ && \
    /tmp/node-build-master/bin/node-build "${NODE_VERSION}" /usr/local/node && \
    npm install -g yarn@$YARN_VERSION && \
    rm -rf /tmp/node-build-master

WORKDIR /rails

# Install gems
COPY Gemfile Gemfile.lock ./
RUN bundle install && \
    rm -rf ~/.bundle/ "${BUNDLE_PATH}"/ruby/*/cache "${BUNDLE_PATH}"/ruby/*/bundler/gems/*/.git && \
    bundle exec bootsnap precompile --gemfile

# Install Node.js packages
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# Copy application code
COPY . .

# Precompile bootsnap
RUN bundle exec bootsnap precompile app/ lib/

# Precompile assets
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile

# Final stage - สร้าง production image ที่เล็กที่สุด
FROM base

WORKDIR /rails

# Copy built artifacts
COPY --from=build "${BUNDLE_PATH}" "${BUNDLE_PATH}"
COPY --from=build /rails /rails

# สร้าง user ที่ไม่ใช่ root
RUN useradd rails --create-home --shell /bin/bash && \
    chown -R rails:rails db log storage tmp

USER rails:rails

# Entrypoint
ENTRYPOINT ["/rails/bin/docker-entrypoint"]

EXPOSE 3000

CMD ["./bin/rails", "server"]
```

### Docker Entrypoint

```bash
#!/bin/bash -e
# bin/docker-entrypoint

# ถ้า command คือ rails server ให้รัน migrations ก่อน
if [ "${1}" == "./bin/rails" ] && [ "${2}" == "server" ]; then
  ./bin/rails db:prepare
fi

exec "${@}"
```

```bash
chmod +x bin/docker-entrypoint
```

### .dockerignore

```
# .dockerignore
.git
.gitignore
.dockerignore
Dockerfile
docker-compose.yml
docker-compose.*.yml

# Test files
spec/
test/

# Development files
.ruby-version
.rubocop.yml
.env*
!.env.example

# Build artifacts
tmp/
log/
node_modules/
coverage/
public/assets/

# Editor
.vscode/
.idea/
*.swp
*.swo
```

---

## ขั้นตอนที่ 1702: docker-compose สำหรับ Development

```yaml
# docker-compose.yml
version: '3.9'

services:
  rails:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bash -c "bundle exec rails db:prepare && bundle exec rails server -b 0.0.0.0"
    environment:
      - RAILS_ENV=development
      - DATABASE_URL=postgresql://postgres:password@postgres/myapp_development
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY_BASE=development_secret_key_base_not_for_production
    ports:
      - "3000:3000"
    volumes:
      - .:/rails
      - bundle_cache:/usr/local/bundle
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  sidekiq:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bundle exec sidekiq
    environment:
      - RAILS_ENV=development
      - DATABASE_URL=postgresql://postgres:password@postgres/myapp_development
      - REDIS_URL=redis://redis:6379/0
    volumes:
      - .:/rails
      - bundle_cache:/usr/local/bundle
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_USER:     postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "8025:8025"   # Web UI
      - "1025:1025"   # SMTP

volumes:
  postgres_data:
  redis_data:
  bundle_cache:
```

### Dockerfile.dev (สำหรับ Development)

```dockerfile
# Dockerfile.dev
FROM ruby:3.3.0-slim

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential \
      git \
      curl \
      libpq-dev \
      libvips \
      nodejs \
      postgresql-client \
      yarn && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

WORKDIR /rails

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

CMD ["bash"]
```

```bash
# Commands ที่ใช้บ่อย
docker-compose up -d
docker-compose down
docker-compose logs -f rails
docker-compose exec rails bash
docker-compose exec rails rails db:migrate
docker-compose exec rails rails c
docker-compose exec rails rspec
docker-compose build --no-cache
```

---

## ขั้นตอนที่ 1703: Kubernetes Basics

### Kubernetes Architecture

```
Master Node
├── API Server      - รับ requests
├── etcd            - เก็บ cluster state
├── Scheduler       - assign pods to nodes
└── Controller Mgr  - ดูแล desired state

Worker Nodes (1..N)
├── kubelet         - ดูแล pods ใน node
├── kube-proxy      - network routing
└── Container Runtime (Docker/containerd)
```

### kubectl Commands พื้นฐาน

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes
kubectl get namespaces

# Pods
kubectl get pods
kubectl get pods -n kube-system
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> -f          # follow logs
kubectl logs <pod-name> -c <container>

# Exec into pod
kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -- rails console

# Apply/Delete resources
kubectl apply -f deployment.yml
kubectl delete -f deployment.yml
kubectl delete pod <pod-name>

# Scale
kubectl scale deployment myapp --replicas=5

# Port forwarding (สำหรับ debugging)
kubectl port-forward pod/<pod-name> 3000:3000
kubectl port-forward service/myapp 3000:3000

# Watch resources
kubectl get pods --watch
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## ขั้นตอนที่ 1704: Rails Deployment ใน Kubernetes

### Deployment Manifest

```yaml
# k8s/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-app
  namespace: production
  labels:
    app: rails-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rails-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge:       1  # เพิ่ม pod ได้อีก 1 ระหว่าง update
      maxUnavailable: 0  # ไม่ให้ pod ลดลงระหว่าง update
  template:
    metadata:
      labels:
        app: rails-app
    spec:
      # ต้องรัน migration ก่อน pods อื่น
      initContainers:
      - name: db-migrate
        image: ghcr.io/myorg/myapp:latest
        command: ["./bin/rails", "db:migrate"]
        env:
        - name: RAILS_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key: database-url

      containers:
      - name: rails
        image: ghcr.io/myorg/myapp:latest
        ports:
        - containerPort: 3000
        env:
        - name: RAILS_ENV
          value: "production"
        - name: RAILS_LOG_TO_STDOUT
          value: "true"
        - name: SECRET_KEY_BASE
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key: secret-key-base
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key: redis-url
        envFrom:
        - configMapRef:
            name: rails-config
        resources:
          requests:
            memory: "256Mi"
            cpu:    "250m"
          limits:
            memory: "512Mi"
            cpu:    "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds:       10
          timeoutSeconds:      5
          failureThreshold:    3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds:       5
          timeoutSeconds:      3
          failureThreshold:    3
        lifecycle:
          preStop:
            exec:
              # รอให้ requests ที่กำลัง process เสร็จก่อน shutdown
              command: ["/bin/sleep", "15"]
      terminationGracePeriodSeconds: 30
```

---

## ขั้นตอนที่ 1705: Kubernetes Service

```yaml
# k8s/service.yml
apiVersion: v1
kind: Service
metadata:
  name: rails-app
  namespace: production
spec:
  selector:
    app: rails-app
  ports:
  - name:       http
    port:       80
    targetPort: 3000
    protocol:   TCP
  type: ClusterIP  # ภายใน cluster

---
# Ingress สำหรับ external traffic
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rails-app
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect:   "true"
    cert-manager.io/cluster-issuer:             "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path:     /
        pathType: Prefix
        backend:
          service:
            name: rails-app
            port:
              number: 80
```

---

## ขั้นตอนที่ 1706: ConfigMaps และ Secrets

### ConfigMap

```yaml
# k8s/configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rails-config
  namespace: production
data:
  RAILS_ENV:             "production"
  RAILS_LOG_TO_STDOUT:   "true"
  RAILS_SERVE_STATIC_FILES: "true"
  WEB_CONCURRENCY:       "2"
  RAILS_MAX_THREADS:     "5"
  SMTP_HOST:             "smtp.sendgrid.net"
  SMTP_PORT:             "587"
  APP_HOST:              "myapp.example.com"
  TIME_ZONE:             "Bangkok"
```

### Secrets

```bash
# สร้าง secrets
kubectl create secret generic rails-secrets \
  --namespace=production \
  --from-literal=secret-key-base="$(openssl rand -hex 64)" \
  --from-literal=database-url="postgresql://user:pass@postgres-svc/myapp_prod" \
  --from-literal=redis-url="redis://redis-svc:6379/0" \
  --from-literal=smtp-password="your-smtp-password" \
  --from-literal=stripe-secret-key="sk_live_..."
```

```yaml
# k8s/secrets.yml (ไม่ควร commit ลง git ตรงๆ - ใช้ sealed secrets แทน)
apiVersion: v1
kind: Secret
metadata:
  name: rails-secrets
  namespace: production
type: Opaque
data:
  # base64 encoded values
  secret-key-base: <base64_encoded_value>
  database-url:    <base64_encoded_value>
  redis-url:       <base64_encoded_value>
```

### Sealed Secrets (ปลอดภัยสำหรับ GitOps)

```bash
# ติดตั้ง kubeseal
brew install kubeseal

# สร้าง Sealed Secret (encrypt ด้วย cluster's public key)
kubectl create secret generic rails-secrets \
  --from-literal=secret-key-base="..." \
  --dry-run=client \
  -o yaml | kubeseal \
  --controller-name=sealed-secrets-controller \
  --format yaml > k8s/sealed-secrets.yml

# คนอื่นอ่าน sealed-secrets.yml ไม่ได้ถอดรหัส
# แต่ cluster ถอดรหัสได้เอง
kubectl apply -f k8s/sealed-secrets.yml
```

---

## ขั้นตอนที่ 1707: Health Checks

### Liveness และ Readiness Probes

```ruby
# app/controllers/health_controller.rb
class HealthController < ActionController::API
  # Liveness: app ยังทำงานอยู่ไหม (ถ้า fail -> restart)
  def live
    render json: { status: 'ok', time: Time.current }
  end

  # Readiness: app พร้อมรับ traffic ไหม (ถ้า fail -> remove from load balancer)
  def ready
    checks = {
      database: check_database,
      redis:    check_redis,
      jobs:     check_jobs
    }

    all_healthy = checks.values.all? { |v| v[:status] == 'ok' }
    status_code = all_healthy ? :ok : :service_unavailable

    render json: {
      status:   all_healthy ? 'ready' : 'not_ready',
      checks:   checks,
      time:     Time.current
    }, status: status_code
  end

  # Startup: app เริ่มต้นเสร็จแล้วไหม
  def startup
    # check migrations ran
    if ActiveRecord::Base.connection.migration_context.needs_migration?
      render json: { status: 'starting', reason: 'migrations_pending' },
             status: :service_unavailable
    else
      render json: { status: 'started' }
    end
  end

  private

  def check_database
    ActiveRecord::Base.connection.execute("SELECT 1")
    { status: 'ok', latency_ms: measure_latency { ActiveRecord::Base.connection.execute("SELECT 1") } }
  rescue => e
    { status: 'error', error: e.message }
  end

  def check_redis
    start = Time.current
    Redis.current.ping
    { status: 'ok', latency_ms: ((Time.current - start) * 1000).round(2) }
  rescue => e
    { status: 'error', error: e.message }
  end

  def check_jobs
    # ตรวจสอบ Sidekiq
    info = Sidekiq::Stats.new
    if info.dead_size > 100
      { status: 'warning', dead_jobs: info.dead_size }
    else
      { status: 'ok', queued: info.enqueued, failed: info.failed }
    end
  rescue => e
    { status: 'error', error: e.message }
  end

  def measure_latency
    start  = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
    ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start) * 1000).round(2)
  end
end
```

```ruby
# config/routes.rb
namespace :health do
  get :live
  get :ready
  get :startup
end
```

---

## ขั้นตอนที่ 1708: Horizontal Pod Autoscaler (HPA)

```yaml
# k8s/hpa.yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rails-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind:       Deployment
    name:       rails-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # Scale ตาม CPU usage
  - type: Resource
    resource:
      name: cpu
      target:
        type:               Utilization
        averageUtilization: 70  # scale up เมื่อ CPU > 70%

  # Scale ตาม Memory usage
  - type: Resource
    resource:
      name: memory
      target:
        type:               Utilization
        averageUtilization: 80

  # Custom metric (จาก Prometheus)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type:         AverageValue
        averageValue: 100  # scale up เมื่อ > 100 rps per pod

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60  # รอ 60 วิก่อน scale up อีก
      policies:
      - type:          Pods
        value:         2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อน scale down
      policies:
      - type:          Pods
        value:         1
        periodSeconds: 60
```

### Vertical Pod Autoscaler (VPA)

```yaml
# k8s/vpa.yml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: rails-app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       rails-app
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Recreate, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: rails
      minAllowed:
        cpu:    100m
        memory: 128Mi
      maxAllowed:
        cpu:    2000m
        memory: 2Gi
```

---

## ขั้นตอนที่ 1709: Sidekiq ใน Kubernetes

```yaml
# k8s/sidekiq-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sidekiq
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sidekiq
  template:
    metadata:
      labels:
        app: sidekiq
    spec:
      containers:
      - name: sidekiq
        image: ghcr.io/myorg/myapp:latest
        command: ["bundle", "exec", "sidekiq", "-C", "config/sidekiq.yml"]
        env:
        - name: RAILS_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key:  database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key:  redis-url
        resources:
          requests:
            memory: "512Mi"
            cpu:    "250m"
          limits:
            memory: "1Gi"
            cpu:    "500m"
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - bundle exec sidekiqmon check
          initialDelaySeconds: 30
          periodSeconds:       30

---
# HPA สำหรับ Sidekiq (scale ตาม queue depth)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sidekiq-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind:       Deployment
    name:       sidekiq
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: External
    external:
      metric:
        name: sidekiq_queue_size
      target:
        type:  AverageValue
        averageValue: 50  # scale up เมื่อ queue > 50 jobs per worker
```

---

## ขั้นตอนที่ 1710: Database ใน Kubernetes

```yaml
# PostgreSQL ด้วย StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "myapp_production"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key:  username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key:  password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            memory: "512Mi"
            cpu:    "500m"
          limits:
            memory: "2Gi"
            cpu:    "2000m"
        volumeMounts:
        - name:      postgres-storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"
          initialDelaySeconds: 30
          periodSeconds:       10
          failureThreshold:    3
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
      storageClassName: standard

---
# Service สำหรับ PostgreSQL
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  selector:
    app: postgres
  ports:
  - port:       5432
    targetPort: 5432
  clusterIP: None  # Headless service สำหรับ StatefulSet
```

---

## ขั้นตอนที่ 1711: Namespace และ Resource Quotas

```yaml
# k8s/namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production

---
# Resource Quota สำหรับ namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu:    "4"
    requests.memory: "8Gi"
    limits.cpu:      "8"
    limits.memory:   "16Gi"
    pods:            "20"
    services:        "10"
    persistentvolumeclaims: "5"

---
# LimitRange - default limits สำหรับ containers
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - default:
      cpu:    "500m"
      memory: "512Mi"
    defaultRequest:
      cpu:    "100m"
      memory: "128Mi"
    type: Container
```

---

## ขั้นตอนที่ 1712: Rolling Deployments และ Zero Downtime

```yaml
# k8s/deployment-zero-downtime.yml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge:       1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: rails
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        # ทำให้ Rails รับ SIGTERM อย่างถูกต้อง
        terminationGracePeriodSeconds: 30
```

### Puma Configuration สำหรับ Graceful Shutdown

```ruby
# config/puma.rb
# Graceful shutdown - รับ requests ที่กำลัง process ก่อน shutdown
on_restart do
  Rails.application.config.after_initialize_callbacks.clear
end

lowlevel_error_handler do |e, env|
  # Handle errors ก่อน shutdown
  body = { error: "Internal Server Error" }.to_json
  [500, { 'Content-Type' => 'application/json' }, [body]]
end

# SIGTERM -> graceful shutdown
Signal.trap 'SIGTERM' do
  @launcher&.stop
end
```

### Database Migration Strategy

```bash
# เรียกใช้ migration ด้วย Kubernetes Job
kubectl apply -f k8s/migration-job.yml
kubectl wait --for=condition=complete job/db-migrate --timeout=120s
kubectl apply -f k8s/deployment.yml
```

```yaml
# k8s/migration-job.yml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  namespace: production
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: db-migrate
        image: ghcr.io/myorg/myapp:latest
        command: ["./bin/rails", "db:migrate"]
        env:
        - name: RAILS_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key:  database-url
```

---

## ขั้นตอนที่ 1713: GitOps ด้วย ArgoCD

### ติดตั้ง ArgoCD

```bash
# ติดตั้ง ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# รอให้พร้อม
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Port forward เพื่อ access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ดู initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### สร้าง ArgoCD Application

```yaml
# argocd/application.yml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name:      myapp-production
  namespace: argocd
spec:
  project: default

  source:
    repoURL:        https://github.com/myorg/myapp-k8s-configs
    targetRevision: HEAD
    path:           environments/production

  destination:
    server:    https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune:    true   # ลบ resources ที่ไม่มีใน git
      selfHeal: true   # auto-fix manual changes
      allowEmpty: false
    syncOptions:
    - Validate=true
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration:    5s
        factor:      2
        maxDuration: 3m

  # Health checks
  ignoreDifferences:
  - group: apps
    kind:  Deployment
    jsonPointers:
    - /spec/replicas  # ignore HPA changes
```

### Kustomize สำหรับ Multi-Environment

```
k8s/
├── base/
│   ├── deployment.yml
│   ├── service.yml
│   ├── configmap.yml
│   └── kustomization.yml
└── overlays/
    ├── staging/
    │   ├── kustomization.yml
    │   └── patches/
    │       └── deployment-patch.yml
    └── production/
        ├── kustomization.yml
        └── patches/
            └── deployment-patch.yml
```

```yaml
# k8s/base/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yml
- service.yml
- configmap.yml

# k8s/overlays/production/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../../base

patches:
- path: patches/deployment-patch.yml

images:
- name:    ghcr.io/myorg/myapp
  newTag:  "1.2.3"  # update image tag ตาม release

# k8s/overlays/production/patches/deployment-patch.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-app
spec:
  replicas: 3  # production ต้องการ 3 replicas
  template:
    spec:
      containers:
      - name: rails
        resources:
          requests:
            memory: "512Mi"
            cpu:    "500m"
          limits:
            memory: "1Gi"
            cpu:    "1000m"
```

---

## ขั้นตอนที่ 1714: Helm Charts

```bash
# ติดตั้ง Helm
brew install helm

# สร้าง Helm Chart
helm create myapp-chart

# โครงสร้าง Chart
myapp-chart/
├── Chart.yaml
├── values.yaml
├── values-production.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl
└── charts/
    └── postgresql/  # dependency
```

```yaml
# myapp-chart/values.yaml
replicaCount: 2

image:
  repository: ghcr.io/myorg/myapp
  pullPolicy: IfNotPresent
  tag:        "latest"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
  - host: myapp.example.com
    paths:
    - path: /
      pathType: Prefix

resources:
  requests:
    memory: "256Mi"
    cpu:    "250m"
  limits:
    memory: "512Mi"
    cpu:    "500m"

autoscaling:
  enabled:        true
  minReplicas:    2
  maxReplicas:    10
  targetCPUPercentage: 70

rails:
  env:         production
  logToStdout: true
  maxThreads:  5

postgresql:
  enabled: true
  auth:
    database: myapp_production
    username: myapp
    existingSecret: postgres-secrets

redis:
  enabled: true
```

```bash
# Deploy ด้วย Helm
helm install myapp myapp-chart/ \
  --namespace production \
  --create-namespace \
  -f values-production.yaml

# Update
helm upgrade myapp myapp-chart/ \
  --namespace production \
  -f values-production.yaml \
  --set image.tag="1.2.3"

# Rollback
helm rollback myapp 1 --namespace production

# Uninstall
helm uninstall myapp --namespace production
```

---

## ขั้นตอนที่ 1715: Logging ใน Kubernetes

```ruby
# config/environments/production.rb
# Log เป็น JSON format สำหรับ log aggregation
config.log_formatter = proc do |severity, datetime, progname, msg|
  {
    severity:  severity,
    timestamp: datetime.iso8601,
    pid:       Process.pid,
    message:   msg.is_a?(String) ? msg : msg.inspect
  }.to_json + "\n"
end
```

### Structured Logging

```ruby
# app/middleware/request_logger.rb
class RequestLogger
  def initialize(app)
    @app = app
  end

  def call(env)
    request    = ActionDispatch::Request.new(env)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    status, headers, body = @app.call(env)

    duration_ms = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time) * 1000).round(2)

    log_data = {
      timestamp:      Time.current.iso8601,
      method:         request.method,
      path:           request.path,
      query_string:   request.query_string,
      status:         status,
      duration_ms:    duration_ms,
      ip:             request.remote_ip,
      user_agent:     request.user_agent,
      request_id:     request.request_id,
      pod_name:       ENV['HOSTNAME'],
      node_name:      ENV['NODE_NAME']
    }

    Rails.logger.info(log_data.to_json)

    [status, headers, body]
  end
end
```

### Fluentd/Fluentbit สำหรับ Log Aggregation

```yaml
# fluentbit-configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: kube-system
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Merge_Log           On
        Keep_Log            Off

    [OUTPUT]
        Name  elasticsearch
        Match *
        Host  ${FLUENT_ELASTICSEARCH_HOST}
        Port  ${FLUENT_ELASTICSEARCH_PORT}
        Index myapp-logs
```

---

## ขั้นตอนที่ 1716: Monitoring ด้วย Prometheus + Grafana

```yaml
# prometheus-values.yml (Helm chart)
prometheus:
  serviceMonitorSelector:
    matchLabels:
      release: prometheus

grafana:
  adminPassword: admin
  dashboardProviders:
    dashboardproviders.yaml:
      providers:
      - name: default
        folder: ''
        type: file
        options:
          path: /var/lib/grafana/dashboards
```

```yaml
# k8s/service-monitor.yml
# สำหรับ Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rails-app
  namespace: production
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: rails-app
  endpoints:
  - port:     http
    path:     /metrics
    interval: 30s
```

---

## ขั้นตอนที่ 1717: Network Policies

```yaml
# k8s/network-policy.yml
# ควบคุม traffic ระหว่าง pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rails-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: rails-app
  policyTypes:
  - Ingress
  - Egress

  ingress:
  # อนุญาตจาก Ingress Controller เท่านั้น
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port:     3000

  egress:
  # อนุญาตออก internet
  - ports:
    - port:     443
      protocol: TCP
    - port:     80
      protocol: TCP

  # อนุญาต database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port:     5432

  # อนุญาต Redis
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port:     6379

  # DNS
  - ports:
    - port:     53
      protocol: UDP
```

---

## ขั้นตอนที่ 1718: CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v4

    - name: Setup Ruby
      uses: ruby/setup-ruby@v1
      with:
        bundler-cache: true

    - name: Run tests
      env:
        DATABASE_URL: postgresql://postgres:password@localhost/test
        RAILS_ENV:    test
      run: bundle exec rspec

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
    - uses: actions/checkout@v4

    - name: Docker meta
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ghcr.io/${{ github.repository }}
        tags: |
          type=sha,prefix=,suffix=,format=short
          type=ref,event=branch
          latest

    - name: Login to GHCR
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push:    true
        tags:    ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to:   type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging

    steps:
    - uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3

    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBECONFIG_STAGING }}" | base64 -d > /tmp/kubeconfig
        echo "KUBECONFIG=/tmp/kubeconfig" >> $GITHUB_ENV

    - name: Deploy to staging
      run: |
        IMAGE_TAG=$(echo "${{ github.sha }}" | head -c7)
        kubectl set image deployment/rails-app \
          rails=ghcr.io/${{ github.repository }}:$IMAGE_TAG \
          -n staging

        kubectl rollout status deployment/rails-app -n staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # ต้องการ approval จาก reviewer

    steps:
    - name: Deploy to production
      run: |
        IMAGE_TAG=$(echo "${{ github.sha }}" | head -c7)
        kubectl set image deployment/rails-app \
          rails=ghcr.io/${{ github.repository }}:$IMAGE_TAG \
          -n production

        kubectl rollout status deployment/rails-app \
          --timeout=5m \
          -n production
```

---

## ขั้นตอนที่ 1719: Disaster Recovery

```bash
# Backup PostgreSQL
kubectl exec -it postgres-0 -n production -- \
  pg_dump -U myapp myapp_production | \
  gzip > backup-$(date +%Y%m%d).sql.gz

# Upload to S3
aws s3 cp backup-$(date +%Y%m%d).sql.gz \
  s3://myapp-backups/db/

# Restore
gzip -d backup-20240101.sql.gz
kubectl exec -i postgres-0 -n production -- \
  psql -U myapp myapp_production < backup-20240101.sql

# Kubernetes Resources Backup (Velero)
velero backup create myapp-backup \
  --include-namespaces production \
  --snapshot-volumes

# Restore
velero restore create --from-backup myapp-backup
```

---

## ขั้นตอนที่ 1720: แนวปฏิบัติ Production Operations

```bash
# Useful kubectl commands สำหรับ production

# ดู resource usage
kubectl top pods -n production
kubectl top nodes

# ดู events ที่เกิดขึ้นล่าสุด
kubectl get events -n production \
  --sort-by=.metadata.creationTimestamp \
  | tail -20

# Debug pod ที่ CrashLoopBackOff
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production --previous

# Force delete stuck pod
kubectl delete pod <pod-name> -n production --force --grace-period=0

# Scale down/up
kubectl scale deployment rails-app --replicas=0 -n production
kubectl scale deployment rails-app --replicas=3 -n production

# Drain node สำหรับ maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node-name>

# ดู logs หลาย pods พร้อมกัน
kubectl logs -l app=rails-app -n production --max-log-requests 10

# Rollback deployment
kubectl rollout undo deployment/rails-app -n production
kubectl rollout undo deployment/rails-app --to-revision=2 -n production

# ดู deployment history
kubectl rollout history deployment/rails-app -n production
```

---

## แบบฝึกหัด: Kubernetes and Production Operations

### ข้อที่ 1: สร้าง Dockerfile ที่ optimize

```dockerfile
# ปรับปรุง Dockerfile นี้ให้:
# 1. Image size เล็กลง
# 2. Build cache ดีขึ้น
# 3. Security ดีขึ้น (non-root user)

# Multi-stage ที่สมบูรณ์
FROM ruby:3.3.0-slim AS base

RUN apt-get update -qq && apt-get install -y libpq5 curl && rm -rf /var/lib/apt/lists

FROM base AS builder
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev && rm -rf /var/lib/apt/lists

WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local without 'development test' && bundle install

COPY . .

FROM base AS final
WORKDIR /app
COPY --from=builder /usr/local/bundle /usr/local/bundle
COPY --from=builder /app .

RUN useradd -m -s /bin/bash app && chown -R app:app /app
USER app

EXPOSE 3000
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```

### ข้อที่ 2: สร้าง docker-compose.yml สมบูรณ์
รวม Rails, Sidekiq, PostgreSQL, Redis, และ MailHog

### ข้อที่ 3: Health Check Implementation
สร้าง `/health/live` และ `/health/ready` endpoints ที่ comprehensive

### ข้อที่ 4: Kubernetes Manifests
สร้าง manifests ครบ: Deployment, Service, ConfigMap, HPA

### ข้อที่ 5: Zero Downtime Deployment
Test rolling update deployment และ verify ว่า no requests fail

### ข้อที่ 6: ArgoCD Setup
Setup ArgoCD ใน local Kubernetes cluster และ configure GitOps

### ข้อที่ 7: Helm Chart
สร้าง Helm Chart สำหรับ Rails application

### ข้อที่ 8-20: แบบฝึกหัดเพิ่มเติม

**ข้อ 8:** Configure resource limits และ requests อย่างเหมาะสม

**ข้อ 9:** สร้าง Network Policy ที่ปลอดภัยสำหรับ Rails pods

**ข้อ 10:** Setup HPA ที่ scale ตาม custom metrics จาก Prometheus

**ข้อ 11:** Implement database backup job ด้วย CronJob

**ข้อ 12:** สร้าง CI/CD pipeline ด้วย GitHub Actions

**ข้อ 13:** Setup Prometheus + Grafana monitoring

**ข้อ 14:** Configure structured logging ด้วย JSON format

**ข้อ 15:** Setup Fluentbit สำหรับ log aggregation

**ข้อ 16:** สร้าง Kubernetes RBAC roles สำหรับ team members

**ข้อ 17:** Implement canary deployment strategy

**ข้อ 18:** Setup Velero สำหรับ cluster backup

**ข้อ 19:** Test disaster recovery procedure

**ข้อ 20:** สร้าง runbook สำหรับ common production issues

---

## สรุป

Kubernetes สำหรับ Rails production:
- **Docker**: multi-stage builds, non-root user, optimize layers
- **Kubernetes Manifests**: Deployment, Service, ConfigMap, Secret
- **Health Probes**: liveness, readiness, startup checks
- **HPA**: auto-scaling ตาม metrics
- **GitOps**: ArgoCD สำหรับ declarative deployments
- **Observability**: Prometheus, Grafana, structured logging
- **Zero Downtime**: rolling updates, graceful shutdown

ขั้นตอนต่อไป: Part 80 - Contributing to Open Source Ruby/Rails
