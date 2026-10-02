# Part 84: DevOps สำหรับ Rails

## บทนำ

DevOps เป็นแนวทางที่รวม Development และ Operations เข้าด้วยกัน เพื่อให้การ deploy และ maintain แอปพลิเคชันเป็นไปอย่างราบรื่น ในบทนี้เราจะเรียนรู้เครื่องมือและแนวปฏิบัติที่สำคัญ

---

## 1. Infrastructure as Code ด้วย Terraform

### 1.1 พื้นฐาน Terraform

```hcl
# infrastructure/main.tf
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

provider "aws" {
  region = var.aws_region
}

# Variables
variable "aws_region" {
  default = "ap-southeast-1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

variable "app_name" {
  default = "myapp"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "${var.app_name}-${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Subnets
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.app_name}-public-${count.index + 1}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "${var.app_name}-private-${count.index + 1}"
    Type = "private"
  }
}

# RDS Database
resource "aws_db_subnet_group" "main" {
  name       = "${var.app_name}-${var.environment}"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_db_instance" "postgres" {
  identifier        = "${var.app_name}-${var.environment}-db"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.t3.medium"
  allocated_storage = 20
  
  db_name  = var.app_name
  username = "postgres"
  password = var.db_password
  
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"
  
  multi_az               = var.environment == "production"
  skip_final_snapshot    = var.environment != "production"
  deletion_protection    = var.environment == "production"
  
  performance_insights_enabled = true
  
  tags = {
    Name        = "${var.app_name}-${var.environment}-db"
    Environment = var.environment
  }
}

# ElastiCache Redis
resource "aws_elasticache_subnet_group" "main" {
  name       = "${var.app_name}-${var.environment}-cache"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "${var.app_name}-${var.environment}"
  description                = "Redis for ${var.app_name} ${var.environment}"
  node_type                  = "cache.t3.micro"
  num_cache_clusters         = var.environment == "production" ? 2 : 1
  automatic_failover_enabled = var.environment == "production"
  
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "${var.app_name}-${var.environment}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  enable_deletion_protection = var.environment == "production"
  
  access_logs {
    bucket  = aws_s3_bucket.logs.id
    prefix  = "alb"
    enabled = true
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "${var.app_name}-${var.environment}"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# ECS Task Definition
resource "aws_ecs_task_definition" "web" {
  family                   = "${var.app_name}-web"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([
    {
      name  = "web"
      image = "${aws_ecr_repository.app.repository_url}:${var.image_tag}"
      
      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]
      
      environment = [
        { name = "RAILS_ENV", value = var.environment },
        { name = "RAILS_LOG_TO_STDOUT", value = "1" },
        { name = "DATABASE_URL", value = "postgresql://${aws_db_instance.postgres.username}:${var.db_password}@${aws_db_instance.postgres.endpoint}/${aws_db_instance.postgres.db_name}" },
        { name = "REDIS_URL", value = "rediss://${aws_elasticache_replication_group.redis.primary_endpoint_address}:6379" }
      ]
      
      secrets = [
        {
          name      = "SECRET_KEY_BASE"
          valueFrom = aws_ssm_parameter.secret_key_base.arn
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "web"
        }
      }
      
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
}

# Outputs
output "alb_dns_name" {
  value = aws_lb.main.dns_name
}

output "rds_endpoint" {
  value     = aws_db_instance.postgres.endpoint
  sensitive = true
}
```

### 1.2 Terraform Modules

```hcl
# infrastructure/modules/rails_app/main.tf
variable "app_name" {}
variable "environment" {}
variable "image_tag" {}
variable "desired_count" { default = 2 }

locals {
  common_tags = {
    Application = var.app_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = var.environment == "production" ? 2 : 1
  resource_id        = "service/${var.cluster_name}/${aws_ecs_service.web.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${var.app_name}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

---

## 2. Ansible Playbook สำหรับ Rails Deployment

### 2.1 โครงสร้าง Ansible Project

```yaml
# ansible/
# ├── inventory/
# │   ├── production
# │   └── staging
# ├── group_vars/
# │   ├── all.yml
# │   ├── production.yml
# │   └── staging.yml
# ├── roles/
# │   ├── common/
# │   ├── ruby/
# │   ├── rails/
# │   └── nginx/
# └── playbooks/
#     ├── deploy.yml
#     ├── setup.yml
#     └── rollback.yml

# inventory/production
[web_servers]
web1.example.com
web2.example.com

[db_servers]
db1.example.com

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_user=deploy
```

### 2.2 Setup Playbook

```yaml
# ansible/playbooks/setup.yml
---
- name: Setup Rails Application Server
  hosts: web_servers
  become: yes
  vars:
    ruby_version: "3.2.2"
    node_version: "20.x"
    app_name: myapp
    deploy_user: deploy
    app_dir: "/var/www/{{ app_name }}"
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
    
    - name: Create deploy user
      user:
        name: "{{ deploy_user }}"
        shell: /bin/bash
        groups: sudo
        append: yes
        create_home: yes
  
  roles:
    - common
    - ruby
    - nodejs
    - nginx
    - postgresql_client
    - redis_client
    - rails

# ansible/roles/ruby/tasks/main.yml
---
- name: Install rbenv dependencies
  apt:
    name:
      - git
      - curl
      - libssl-dev
      - libreadline-dev
      - zlib1g-dev
      - autoconf
      - bison
      - build-essential
      - libyaml-dev
      - libreadline6-dev
      - libncurses5-dev
      - libffi-dev
      - libgdbm-dev
    state: present

- name: Install rbenv
  become_user: "{{ deploy_user }}"
  git:
    repo: https://github.com/rbenv/rbenv.git
    dest: "~/.rbenv"
    version: master

- name: Add rbenv to PATH
  become_user: "{{ deploy_user }}"
  lineinfile:
    path: "~/.bashrc"
    line: "{{ item }}"
  loop:
    - 'export PATH="$HOME/.rbenv/bin:$PATH"'
    - 'eval "$(rbenv init -)"'

- name: Install ruby-build
  become_user: "{{ deploy_user }}"
  git:
    repo: https://github.com/rbenv/ruby-build.git
    dest: "~/.rbenv/plugins/ruby-build"
    version: master

- name: Install Ruby {{ ruby_version }}
  become_user: "{{ deploy_user }}"
  shell: |
    export PATH="$HOME/.rbenv/bin:$PATH"
    eval "$(rbenv init -)"
    rbenv install {{ ruby_version }} --skip-existing
    rbenv global {{ ruby_version }}
  args:
    creates: "{{ deploy_user_home }}/.rbenv/versions/{{ ruby_version }}"

- name: Install Bundler
  become_user: "{{ deploy_user }}"
  shell: |
    export PATH="$HOME/.rbenv/bin:$PATH"
    eval "$(rbenv init -)"
    gem install bundler --no-document
  args:
    executable: /bin/bash
```

### 2.3 Deploy Playbook

```yaml
# ansible/playbooks/deploy.yml
---
- name: Deploy Rails Application
  hosts: web_servers
  serial: 1  # Rolling deployment
  vars:
    app_name: myapp
    deploy_user: deploy
    app_dir: "/var/www/{{ app_name }}"
    shared_dir: "{{ app_dir }}/shared"
    releases_dir: "{{ app_dir }}/releases"
    current_dir: "{{ app_dir }}/current"
    release_timestamp: "{{ ansible_date_time.epoch }}"
    release_dir: "{{ releases_dir }}/{{ release_timestamp }}"
    keep_releases: 5
    git_repo: "git@github.com:company/myapp.git"
    git_branch: main
  
  tasks:
    - name: Create release directory
      file:
        path: "{{ release_dir }}"
        state: directory
        owner: "{{ deploy_user }}"
        mode: '0755'
    
    - name: Clone/Update repository
      become_user: "{{ deploy_user }}"
      git:
        repo: "{{ git_repo }}"
        dest: "{{ release_dir }}"
        version: "{{ git_branch }}"
        depth: 1
    
    - name: Link shared files
      become_user: "{{ deploy_user }}"
      file:
        src: "{{ shared_dir }}/{{ item.src }}"
        dest: "{{ release_dir }}/{{ item.dest }}"
        state: link
        force: yes
      loop:
        - { src: "config/database.yml", dest: "config/database.yml" }
        - { src: "config/credentials/production.key", dest: "config/credentials/production.key" }
        - { src: "public/uploads", dest: "public/uploads" }
        - { src: "log", dest: "log" }
        - { src: "tmp/pids", dest: "tmp/pids" }
    
    - name: Install gems
      become_user: "{{ deploy_user }}"
      bundler:
        state: present
        chdir: "{{ release_dir }}"
        deployment: yes
        without: development:test
        path: "{{ shared_dir }}/bundle"
    
    - name: Precompile assets
      become_user: "{{ deploy_user }}"
      shell: |
        cd {{ release_dir }}
        RAILS_ENV=production bundle exec rails assets:precompile
      environment:
        RAILS_ENV: production
    
    - name: Run database migrations
      become_user: "{{ deploy_user }}"
      shell: |
        cd {{ release_dir }}
        RAILS_ENV=production bundle exec rails db:migrate
      run_once: yes  # รันแค่ครั้งเดียวบน first server
      environment:
        RAILS_ENV: production
    
    - name: Warm up application
      become_user: "{{ deploy_user }}"
      shell: |
        cd {{ release_dir }}
        RAILS_ENV=production bundle exec rails runner "puts 'Warmup complete'"
      environment:
        RAILS_ENV: production
    
    - name: Update current symlink
      become_user: "{{ deploy_user }}"
      file:
        src: "{{ release_dir }}"
        dest: "{{ current_dir }}"
        state: link
        force: yes
    
    - name: Restart application
      shell: |
        cd {{ current_dir }}
        bundle exec pumactl phased-restart
      become_user: "{{ deploy_user }}"
      ignore_errors: yes
      register: puma_restart
    
    - name: Start application if not running
      shell: |
        cd {{ current_dir }}
        bundle exec puma -C config/puma.rb -e production -d
      become_user: "{{ deploy_user }}"
      when: puma_restart.failed
    
    - name: Cleanup old releases
      become_user: "{{ deploy_user }}"
      shell: |
        ls -dt {{ releases_dir }}/* | tail -n +{{ keep_releases + 1 }} | xargs rm -rf
    
    - name: Verify deployment
      uri:
        url: "http://localhost:3000/health"
        method: GET
        status_code: 200
      retries: 5
      delay: 10

# Rollback playbook
# ansible/playbooks/rollback.yml
---
- name: Rollback deployment
  hosts: web_servers
  vars:
    app_dir: "/var/www/{{ app_name }}"
    
  tasks:
    - name: Get list of releases
      find:
        paths: "{{ app_dir }}/releases"
        file_type: directory
      register: releases
    
    - name: Find previous release
      set_fact:
        previous_release: "{{ (releases.files | sort(attribute='mtime') | list)[-2].path }}"
    
    - name: Symlink to previous release
      file:
        src: "{{ previous_release }}"
        dest: "{{ app_dir }}/current"
        state: link
        force: yes
    
    - name: Restart application
      shell: bundle exec pumactl phased-restart
      args:
        chdir: "{{ app_dir }}/current"
    
    - name: Verify rollback
      uri:
        url: "http://localhost:3000/health"
        status_code: 200
```

---

## 3. Blue-Green Deployment Strategy

### 3.1 Blue-Green ด้วย AWS ECS

```hcl
# infrastructure/blue_green.tf

# Target Groups
resource "aws_lb_target_group" "blue" {
  name     = "${var.app_name}-blue"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  target_type = "ip"
  
  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 10
    matcher             = "200"
  }
}

resource "aws_lb_target_group" "green" {
  name     = "${var.app_name}-green"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  target_type = "ip"
  
  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 10
    matcher             = "200"
  }
}

# Load Balancer Listener Rules
resource "aws_lb_listener" "main" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.blue.arn
  }
}

# ECS Deployment with CodeDeploy
resource "aws_codedeploy_app" "main" {
  compute_platform = "ECS"
  name             = var.app_name
}

resource "aws_codedeploy_deployment_group" "main" {
  app_name               = aws_codedeploy_app.main.name
  deployment_group_name  = "${var.app_name}-${var.environment}"
  deployment_config_name = "CodeDeployDefault.ECSAllAtOnce"
  service_role_arn       = aws_iam_role.codedeploy.arn
  
  ecs_service {
    cluster_name = aws_ecs_cluster.main.name
    service_name = aws_ecs_service.web.name
  }
  
  load_balancer_info {
    target_group_pair_info {
      prod_traffic_route {
        listener_arns = [aws_lb_listener.main.arn]
      }
      
      target_group {
        name = aws_lb_target_group.blue.name
      }
      
      target_group {
        name = aws_lb_target_group.green.name
      }
      
      test_traffic_route {
        listener_arns = [aws_lb_listener.test.arn]  # Port 8080
      }
    }
  }
  
  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"
    deployment_type   = "BLUE_GREEN"
  }
  
  blue_green_deployment_config {
    deployment_ready_option {
      action_on_timeout    = "STOP_DEPLOYMENT"
      wait_time_in_minutes = 5
    }
    
    terminate_blue_instances_on_deployment_success {
      action                           = "TERMINATE"
      termination_wait_time_in_minutes = 5
    }
  }
  
  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }
}
```

### 3.2 Blue-Green Script สำหรับ VPS

```bash
#!/bin/bash
# scripts/blue_green_deploy.sh

set -euo pipefail

APP_NAME="myapp"
BLUE_PORT=3000
GREEN_PORT=3001
NGINX_CONFIG="/etc/nginx/sites-available/${APP_NAME}"
HEALTH_ENDPOINT="/health"

get_active_color() {
  if nginx -t 2>/dev/null && grep -q "proxy_pass.*:${BLUE_PORT}" "${NGINX_CONFIG}"; then
    echo "blue"
  else
    echo "green"
  fi
}

get_inactive_color() {
  [ "$(get_active_color)" == "blue" ] && echo "green" || echo "blue"
}

get_inactive_port() {
  [ "$(get_active_color)" == "blue" ] && echo "${GREEN_PORT}" || echo "${BLUE_PORT}"
}

deploy_to_inactive() {
  local color=$(get_inactive_color)
  local port=$(get_inactive_port)
  local deploy_dir="/var/www/${APP_NAME}-${color}"
  
  echo "Deploying to ${color} (port ${port})..."
  
  # Clone latest code
  git -C "${deploy_dir}" pull origin main
  
  # Install dependencies
  cd "${deploy_dir}"
  bundle install --without development test
  
  # Precompile assets
  RAILS_ENV=production bundle exec rails assets:precompile
  
  # Run migrations
  RAILS_ENV=production bundle exec rails db:migrate
  
  # Start new server
  RAILS_ENV=production RAILS_SERVE_STATIC_FILES=true \
    bundle exec puma -p "${port}" -e production \
    -C config/puma.rb --pidfile "tmp/pids/puma-${color}.pid" -d
  
  echo "Waiting for ${color} to start..."
  sleep 10
  
  # Health check
  for i in {1..10}; do
    if curl -s -o /dev/null -w "%{http_code}" "http://localhost:${port}${HEALTH_ENDPOINT}" | grep -q "200"; then
      echo "${color} is healthy!"
      return 0
    fi
    echo "Health check attempt ${i}/10..."
    sleep 5
  done
  
  echo "Health check failed for ${color}!"
  return 1
}

switch_traffic() {
  local color=$(get_inactive_color)
  local port=$(get_inactive_port)
  
  echo "Switching traffic to ${color}..."
  
  # Update nginx config
  sed -i "s/proxy_pass http:\/\/localhost:[0-9]*/proxy_pass http:\/\/localhost:${port}/" "${NGINX_CONFIG}"
  
  # Reload nginx (zero downtime)
  nginx -s reload
  
  echo "Traffic switched to ${color}!"
}

stop_old_server() {
  local old_color=$(get_active_color)  # Active before switch
  
  echo "Stopping old ${old_color} server..."
  
  local pid_file="/var/www/${APP_NAME}-${old_color}/tmp/pids/puma-${old_color}.pid"
  
  if [ -f "${pid_file}" ]; then
    kill -TERM $(cat "${pid_file}") 2>/dev/null || true
    rm -f "${pid_file}"
  fi
  
  echo "Old ${old_color} server stopped."
}

# Main deployment flow
echo "=== Blue-Green Deployment Started ==="
echo "Current active: $(get_active_color)"
echo "Deploying to: $(get_inactive_color)"

deploy_to_inactive || {
  echo "Deployment failed! Keeping current ${get_active_color} active."
  exit 1
}

switch_traffic
echo "Waiting for connections to drain..."
sleep 30
stop_old_server

echo "=== Deployment Complete ==="
echo "Active: $(get_active_color)"
```

---

## 4. Feature Flags ด้วย Flipper Gem

### 4.1 การติดตั้ง Flipper

```ruby
# Gemfile
gem 'flipper'
gem 'flipper-active_record'
gem 'flipper-ui'

# config/initializers/flipper.rb
require 'flipper'
require 'flipper/adapters/active_record'

Flipper.configure do |config|
  config.default do
    adapter = Flipper::Adapters::ActiveRecord.new
    Flipper.new(adapter)
  end
end

# สร้าง migrations
# rails generate flipper:active_record
# rails db:migrate
```

### 4.2 การใช้งาน Feature Flags

```ruby
# config/initializers/flipper_features.rb
# กำหนด features ทั้งหมด
FEATURES = %i[
  new_checkout_flow
  ai_recommendations
  dark_mode
  beta_dashboard
  experimental_search
].freeze

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  helper_method :feature_enabled?
  
  def feature_enabled?(feature, actor = current_user)
    Flipper.enabled?(feature, actor)
  end
  
  def require_feature(feature)
    unless feature_enabled?(feature)
      respond_to do |format|
        format.html { redirect_to root_path, alert: 'This feature is not available.' }
        format.json { render json: { error: 'Feature not available' }, status: :not_found }
      end
    end
  end
end

# app/controllers/checkout_controller.rb
class CheckoutController < ApplicationController
  before_action -> { require_feature(:new_checkout_flow) }, only: [:new_flow]
  
  def show
    if feature_enabled?(:new_checkout_flow)
      render 'checkout/new_flow'
    else
      render 'checkout/classic'
    end
  end
  
  def new_flow
    # New checkout implementation
  end
end
```

### 4.3 Feature Flags สำหรับ Groups

```ruby
# config/initializers/flipper_groups.rb
Flipper.register(:beta_users) do |actor|
  actor.respond_to?(:beta?) && actor.beta?
end

Flipper.register(:premium_users) do |actor|
  actor.respond_to?(:plan) && actor.plan == 'premium'
end

Flipper.register(:internal_team) do |actor|
  actor.respond_to?(:email) && actor.email.end_with?('@company.com')
end

Flipper.register(:high_value_customers) do |actor|
  actor.respond_to?(:total_spend) && actor.total_spend > 50_000
end

# การ enable สำหรับ groups
# Flipper.enable_group(:new_checkout_flow, :beta_users)
# Flipper.enable_group(:ai_recommendations, :premium_users)
# Flipper.enable(:dark_mode, :internal_team)

# Percentage rollout
# Flipper.enable_percentage_of_actors(:experimental_search, 10)  # 10% of users
# Flipper.enable_percentage_of_time(:load_testing_mode, 5)       # 5% of requests
```

### 4.4 Feature Flag Management API

```ruby
# app/controllers/admin/feature_flags_controller.rb
module Admin
  class FeatureFlagsController < ApplicationController
    before_action :require_admin
    
    def index
      @features = FEATURES.map do |feature|
        {
          name: feature,
          enabled: Flipper.enabled?(feature),
          enabled_for_everyone: Flipper[feature].enabled?,
          gates: Flipper[feature].gates.select(&:enabled?).map(&:name)
        }
      end
      
      render json: @features
    end
    
    def enable
      feature = params[:feature].to_sym
      
      case params[:target]
      when 'everyone'
        Flipper.enable(feature)
      when 'percentage'
        percentage = params[:percentage].to_i
        Flipper.enable_percentage_of_actors(feature, percentage)
      when 'group'
        Flipper.enable_group(feature, params[:group].to_sym)
      when 'user'
        user = User.find(params[:user_id])
        Flipper.enable(feature, user)
      end
      
      render json: { feature: feature, status: 'enabled' }
    end
    
    def disable
      feature = params[:feature].to_sym
      Flipper.disable(feature)
      render json: { feature: feature, status: 'disabled' }
    end
  end
end
```

### 4.5 Feature Flags ใน Background Jobs

```ruby
# app/jobs/email_campaign_job.rb
class EmailCampaignJob < ApplicationJob
  def perform(campaign_id)
    campaign = Campaign.find(campaign_id)
    
    users = if Flipper.enabled?(:new_email_engine)
      User.where(email_verified: true)
    else
      User.active
    end
    
    users.in_batches do |batch|
      batch.each do |user|
        # Feature check ต่อ user
        if Flipper.enabled?(:personalized_campaigns, user)
          PersonalizedCampaignMailer.send_campaign(user, campaign).deliver_later
        else
          StandardCampaignMailer.send_campaign(user, campaign).deliver_later
        end
      end
    end
  end
end
```

---

## 5. Zero-downtime Deployments ด้วย Kamal

### 5.1 การติดตั้ง Kamal

```bash
gem install kamal
# หรือ
bundle add kamal
```

### 5.2 Kamal Configuration

```yaml
# config/deploy.yml
service: myapp
image: company/myapp

servers:
  web:
    hosts:
      - web1.example.com
      - web2.example.com
    labels:
      traefik.http.routers.myapp.rule: Host(`app.example.com`)
      traefik.http.routers.myapp.tls: true
      traefik.http.routers.myapp.tls.certresolver: letsencrypt
    options:
      network: private
  
  workers:
    hosts:
      - worker1.example.com
    cmd: bundle exec sidekiq -C config/sidekiq.yml
    options:
      network: private

proxy:
  ssl: true
  host: app.example.com
  app_port: 3000
  healthcheck:
    path: /up
    interval: 3
    threshold: 5

registry:
  server: registry.digitalocean.com
  username: 
    - KAMAL_REGISTRY_USERNAME
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    RAILS_ENV: production
    RAILS_LOG_TO_STDOUT: "1"
    RAILS_SERVE_STATIC_FILES: "1"
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
    - REDIS_URL
    - SECRET_KEY_BASE

volumes:
  - data:/rails/storage

asset_path: /rails/public/assets

builder:
  multiarch: false
  args:
    RUBY_VERSION: "3.2.2"

accessories:
  postgres:
    image: postgres:15
    host: db.example.com
    port: 5432
    env:
      clear:
        POSTGRES_DB: myapp_production
      secret:
        - POSTGRES_PASSWORD
    volumes:
      - /var/lib/postgresql/data:/var/lib/postgresql/data
  
  redis:
    image: redis:7
    host: cache.example.com
    port: 6379
    volumes:
      - /var/lib/redis:/data

ssh:
  user: deploy
  proxy: "bastion.example.com"

healthcheck:
  path: /up
  port: 3000
  max_attempts: 10
  interval: 20s

hooks_path: .kamal/hooks
```

### 5.3 Kamal Hooks

```bash
#!/bin/bash
# .kamal/hooks/pre-deploy

echo "Running pre-deploy checks..."

# ตรวจสอบว่า migrations ไม่ destructive
if bundle exec rails db:migrate:status 2>&1 | grep -q "down"; then
  echo "WARNING: There are pending migrations"
fi

# Run tests
bundle exec rspec --format progress

# Check for security vulnerabilities
bundle exec bundler-audit check --update

echo "Pre-deploy checks complete!"
```

```bash
#!/bin/bash
# .kamal/hooks/post-deploy

echo "Running post-deploy tasks..."

# Clear application cache
kamal app exec --primary "bundle exec rails cache:clear"

# Warm up cache
kamal app exec --primary "bundle exec rails runner 'Rails.cache.write(:warmup, true)'"

# Send deployment notification
curl -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  --data "{\"text\":\"Deployed ${VERSION} to production successfully!\"}"

echo "Post-deploy tasks complete!"
```

### 5.4 Deploy Commands

```bash
# Setup สำหรับ servers ครั้งแรก
kamal setup

# Deploy
kamal deploy

# Deploy specific version
kamal deploy --version v1.2.3

# Rollback
kamal rollback

# Check status
kamal app status

# View logs
kamal app logs --follow

# Run console
kamal app exec --primary "bundle exec rails console"

# Run migration
kamal app exec --primary "bundle exec rails db:migrate"

# Update environment variables
kamal env push

# Scale
kamal scale web=5
```

---

## 6. Health Checks และ Readiness Probes

### 6.1 Health Check Controller

```ruby
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  skip_before_action :authenticate_user!
  
  # Liveness probe - แอปยังรันอยู่หรือไม่
  def liveness
    render json: { 
      status: 'ok',
      timestamp: Time.current.iso8601,
      version: ENV['APP_VERSION'] || 'unknown'
    }
  end
  
  # Readiness probe - แอปพร้อม serve traffic หรือไม่
  def readiness
    checks = {
      database: check_database,
      redis: check_redis,
      sidekiq: check_sidekiq,
      disk_space: check_disk_space
    }
    
    all_healthy = checks.values.all? { |c| c[:status] == 'ok' }
    
    render json: {
      status: all_healthy ? 'ok' : 'degraded',
      checks: checks,
      timestamp: Time.current.iso8601
    }, status: all_healthy ? :ok : :service_unavailable
  end
  
  # Detailed health status
  def status
    render json: {
      application: application_status,
      dependencies: dependency_status,
      metrics: application_metrics,
      timestamp: Time.current.iso8601
    }
  end
  
  private
  
  def check_database
    ActiveRecord::Base.connection.execute('SELECT 1')
    { status: 'ok', response_time_ms: measure_time { ActiveRecord::Base.connection.execute('SELECT 1') } }
  rescue => e
    { status: 'error', error: e.message }
  end
  
  def check_redis
    redis = Redis.new(url: ENV['REDIS_URL'])
    redis.ping
    { status: 'ok', response_time_ms: measure_time { redis.ping } }
  rescue => e
    { status: 'error', error: e.message }
  end
  
  def check_sidekiq
    stats = Sidekiq::Stats.new
    dead_count = stats.dead_size
    
    if dead_count > 100
      { status: 'warning', dead_jobs: dead_count }
    else
      { status: 'ok', queued: stats.queued, processed: stats.processed }
    end
  rescue => e
    { status: 'error', error: e.message }
  end
  
  def check_disk_space
    disk_info = `df -h / | tail -1`.split
    usage_percent = disk_info[4].to_i
    
    if usage_percent > 90
      { status: 'critical', usage_percent: usage_percent }
    elsif usage_percent > 75
      { status: 'warning', usage_percent: usage_percent }
    else
      { status: 'ok', usage_percent: usage_percent }
    end
  end
  
  def application_status
    {
      version: ENV['APP_VERSION'],
      environment: Rails.env,
      ruby_version: RUBY_VERSION,
      rails_version: Rails::VERSION::STRING,
      uptime_seconds: uptime
    }
  end
  
  def dependency_status
    {
      database: check_database,
      redis: check_redis,
      sidekiq: check_sidekiq,
      external_apis: check_external_apis
    }
  end
  
  def check_external_apis
    # ตรวจ payment gateway
    payment_status = begin
      response = HTTP.timeout(2).get("#{ENV['PAYMENT_GATEWAY_URL']}/health")
      response.code == 200 ? 'ok' : 'degraded'
    rescue => e
      'error'
    end
    
    { payment_gateway: payment_status }
  end
  
  def application_metrics
    {
      active_users: User.where('last_seen_at > ?', 5.minutes.ago).count,
      pending_orders: Order.pending.count,
      queue_depth: Sidekiq::Stats.new.queued,
      memory_usage_mb: (`ps -o rss= -p #{Process.pid}`.to_i / 1024.0).round(2)
    }
  end
  
  def measure_time
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
    ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start) * 1000).round(2)
  end
  
  def uptime
    Process.clock_gettime(Process::CLOCK_MONOTONIC).to_i
  end
end
```

### 6.2 Routes สำหรับ Health Checks

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Health endpoints (ไม่ต้องการ authentication)
  get '/health', to: 'health#liveness'
  get '/health/live', to: 'health#liveness'
  get '/health/ready', to: 'health#readiness'
  get '/health/status', to: 'health#status'
  
  # Kubernetes-style endpoints
  get '/up', to: proc { [200, {}, ['OK']] }  # Rails 7.1+
end
```

### 6.3 Kubernetes Health Check Configuration

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-web
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
        - name: web
          image: company/myapp:latest
          ports:
            - containerPort: 3000
          
          # Liveness Probe - restart container ถ้า unhealthy
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
          
          # Readiness Probe - remove from LB ถ้าไม่พร้อม
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 5
          
          # Startup Probe - ให้เวลา app start up
          startupProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 30  # 30 * 5 = 150 seconds to start
            
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          env:
            - name: RAILS_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: DATABASE_URL
```

### 6.4 Monitoring Middleware

```ruby
# lib/middleware/request_monitor.rb
class RequestMonitor
  def initialize(app)
    @app = app
  end
  
  def call(env)
    start_time = Time.current
    status, headers, body = @app.call(env)
    duration = (Time.current - start_time) * 1000
    
    # Log slow requests
    if duration > 500
      Rails.logger.warn("Slow request: #{env['REQUEST_METHOD']} #{env['PATH_INFO']} took #{duration.round(2)}ms")
    end
    
    # Prometheus metrics
    REQUEST_COUNT.increment(
      labels: {
        method: env['REQUEST_METHOD'],
        path: normalized_path(env['PATH_INFO']),
        status: status.to_s
      }
    )
    
    REQUEST_DURATION.observe(
      duration / 1000.0,
      labels: {
        method: env['REQUEST_METHOD'],
        path: normalized_path(env['PATH_INFO'])
      }
    )
    
    headers['X-Request-Id'] ||= SecureRandom.uuid
    headers['X-Response-Time'] = "#{duration.round(2)}ms"
    
    [status, headers, body]
  end
  
  private
  
  def normalized_path(path)
    path.gsub(%r{/\d+}, '/:id')
        .gsub(%r{/[0-9a-f-]{36}}, '/:uuid')
  end
end

# config/application.rb
config.middleware.use RequestMonitor
```

---

## แบบฝึกหัด (20 ข้อพร้อมเฉลย)

### ข้อ 1: Terraform VPC Module
**คำถาม:** สร้าง Terraform module สำหรับ VPC ที่มี public และ private subnets พร้อม NAT Gateway

**เฉลย:**
```hcl
# modules/vpc/main.tf
variable "cidr_block" { default = "10.0.0.0/16" }
variable "environment" {}
variable "app_name" {}
variable "az_count" { default = 2 }

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "${var.app_name}-${var.environment}"
    Environment = var.environment
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.app_name}-igw" }
}

resource "aws_subnet" "public" {
  count                   = var.az_count
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.app_name}-public-${count.index + 1}", Type = "public" }
}

resource "aws_subnet" "private" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "${var.app_name}-private-${count.index + 1}", Type = "private" }
}

resource "aws_eip" "nat" {
  count  = var.az_count
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = var.az_count
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags = { Name = "${var.app_name}-nat-${count.index + 1}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table" "private" {
  count  = var.az_count
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
}

output "vpc_id" { value = aws_vpc.main.id }
output "public_subnet_ids" { value = aws_subnet.public[*].id }
output "private_subnet_ids" { value = aws_subnet.private[*].id }
```

### ข้อ 2: Ansible Role สำหรับ Nginx
**คำถาม:** สร้าง Ansible role สำหรับ configure Nginx เป็น reverse proxy สำหรับ Puma

**เฉลย:**
```yaml
# roles/nginx/tasks/main.yml
---
- name: Install Nginx
  apt:
    name: nginx
    state: present

- name: Remove default config
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent

- name: Configure app site
  template:
    src: rails_app.conf.j2
    dest: "/etc/nginx/sites-available/{{ app_name }}"
    mode: '0644'
  notify: Reload nginx

- name: Enable site
  file:
    src: "/etc/nginx/sites-available/{{ app_name }}"
    dest: "/etc/nginx/sites-enabled/{{ app_name }}"
    state: link
  notify: Reload nginx

- name: Test nginx config
  command: nginx -t
  changed_when: false

- name: Ensure nginx is started
  service:
    name: nginx
    state: started
    enabled: yes

# roles/nginx/templates/rails_app.conf.j2
upstream puma {
  server unix:{{ app_dir }}/current/tmp/sockets/puma.sock fail_timeout=0;
}

server {
  listen 80;
  server_name {{ server_name }};
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl http2;
  server_name {{ server_name }};
  
  ssl_certificate /etc/letsencrypt/live/{{ server_name }}/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/{{ server_name }}/privkey.pem;
  
  root {{ app_dir }}/current/public;
  
  location /assets {
    expires 1y;
    add_header Cache-Control public;
    gzip_static on;
  }
  
  location / {
    proxy_pass http://puma;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
  }
  
  location /health {
    proxy_pass http://puma;
    access_log off;
  }
  
  error_page 500 502 503 504 /500.html;
  client_max_body_size 50M;
}
```

### ข้อ 3: Blue-Green Health Check
**คำถาม:** เขียน script ที่ verify health ของ green environment ก่อน switch traffic

**เฉลย:**
```bash
#!/bin/bash
# scripts/verify_green.sh

GREEN_HOST="${1:-localhost}"
GREEN_PORT="${2:-3001}"
MAX_ATTEMPTS=20
SLEEP_INTERVAL=10
REQUIRED_CONSECUTIVE_SUCCESSES=3
SUCCESS_COUNT=0

check_health() {
  local response
  response=$(curl -s -o /dev/null -w "%{http_code}" \
    --max-time 5 \
    "http://${GREEN_HOST}:${GREEN_PORT}/health/ready" 2>/dev/null)
  
  [ "$response" == "200" ]
}

check_detailed_health() {
  local body
  body=$(curl -s --max-time 10 "http://${GREEN_HOST}:${GREEN_PORT}/health/status" 2>/dev/null)
  
  if echo "$body" | python3 -c "
import json, sys
data = json.load(sys.stdin)
checks = data.get('checks', {})
all_ok = all(v.get('status') in ['ok', 'warning'] for v in checks.values())
sys.exit(0 if all_ok else 1)
" 2>/dev/null; then
    echo "All health checks passed"
    return 0
  else
    echo "Some health checks failed"
    echo "Response: $body"
    return 1
  fi
}

echo "Verifying green environment at ${GREEN_HOST}:${GREEN_PORT}..."

for ((i=1; i<=MAX_ATTEMPTS; i++)); do
  echo "Attempt ${i}/${MAX_ATTEMPTS}..."
  
  if check_health; then
    SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    echo "Health check passed (${SUCCESS_COUNT}/${REQUIRED_CONSECUTIVE_SUCCESSES})"
    
    if [ $SUCCESS_COUNT -ge $REQUIRED_CONSECUTIVE_SUCCESSES ]; then
      echo "Running detailed health check..."
      if check_detailed_health; then
        echo "Green environment is HEALTHY. Safe to switch traffic."
        exit 0
      else
        SUCCESS_COUNT=0
      fi
    fi
  else
    SUCCESS_COUNT=0
    echo "Health check failed, resetting counter"
  fi
  
  if [ $i -lt $MAX_ATTEMPTS ]; then
    sleep $SLEEP_INTERVAL
  fi
done

echo "ERROR: Green environment failed health verification after ${MAX_ATTEMPTS} attempts"
exit 1
```

### ข้อ 4: Feature Flag สำหรับ A/B Testing
**คำถาม:** implement A/B testing โดยใช้ Flipper ที่ assign users อย่าง consistent

**เฉลย:**
```ruby
# app/services/ab_test_service.rb
class AbTestService
  EXPERIMENT_GROUPS = {
    checkout_ab_test: {
      variants: ['control', 'variant_a', 'variant_b'],
      weights: [50, 25, 25]
    },
    homepage_ab_test: {
      variants: ['original', 'new_design'],
      weights: [50, 50]
    }
  }.freeze
  
  def self.variant_for(experiment, user)
    config = EXPERIMENT_GROUPS[experiment.to_sym]
    return nil unless config
    
    # Consistent hashing สำหรับ user
    hash = Digest::MD5.hexdigest("#{experiment}:#{user.id}").to_i(16)
    bucket = hash % 100
    
    cumulative = 0
    config[:variants].zip(config[:weights]).each do |variant, weight|
      cumulative += weight
      return variant if bucket < cumulative
    end
    
    config[:variants].first
  end
  
  def self.track_conversion(experiment, user, variant, metric)
    Rails.logger.info({
      event: 'ab_test_conversion',
      experiment: experiment,
      user_id: user.id,
      variant: variant,
      metric: metric,
      timestamp: Time.current.iso8601
    }.to_json)
    
    # Store in analytics
    AbTestResult.create!(
      experiment: experiment,
      user: user,
      variant: variant,
      metric: metric
    )
  end
end

# app/controllers/checkout_controller.rb
class CheckoutController < ApplicationController
  before_action :set_ab_variant
  
  def show
    if @checkout_variant == 'variant_a'
      render 'checkout/streamlined'
    elsif @checkout_variant == 'variant_b'
      render 'checkout/one_page'
    else
      render 'checkout/classic'
    end
  end
  
  private
  
  def set_ab_variant
    if Flipper.enabled?(:checkout_ab_test)
      @checkout_variant = AbTestService.variant_for(:checkout_ab_test, current_user)
    else
      @checkout_variant = 'control'
    end
  end
end
```

### ข้อ 5: Kamal Multi-stage Deployment
**คำถาม:** configure Kamal สำหรับ deploy ไป staging และ production ด้วย config ต่างกัน

**เฉลย:**
```yaml
# config/deploy.yml (base)
service: myapp
image: company/myapp

# config/deploy.staging.yml
servers:
  web:
    hosts:
      - staging.example.com
    env:
      clear:
        RAILS_ENV: staging

# config/deploy.production.yml
servers:
  web:
    hosts:
      - web1.prod.example.com
      - web2.prod.example.com
    options:
      network: production
  
  workers:
    hosts:
      - worker1.prod.example.com
    cmd: bundle exec sidekiq

env:
  clear:
    RAILS_ENV: production
    RAILS_SERVE_STATIC_FILES: "1"
```

```bash
# Deploy to staging
kamal deploy --config-file config/deploy.staging.yml

# Deploy to production
kamal deploy --config-file config/deploy.production.yml

# Alias ใน .bashrc หรือ Makefile
# make deploy-staging
# make deploy-production
```

### ข้อ 6-20 (สรุปเฉลย):

**ข้อ 6: Database Migration Strategy**
```ruby
# Strong Migrations pattern - ป้องกัน locking
class AddIndexToUsersEmail < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!
  
  def change
    add_index :users, :email, algorithm: :concurrently, unique: true, if_not_exists: true
  end
end
```

**ข้อ 7: Rolling Deployment Ansible**
```yaml
# Serial deployment ด้วย Ansible
- hosts: web_servers
  serial: "25%"  # Update 25% of servers at a time
  max_fail_percentage: 0  # ถ้า fail ให้หยุดทันที
  tasks:
    - include_tasks: deploy_tasks.yml
```

**ข้อ 8: Feature Flag Gradual Rollout**
```ruby
# Gradual rollout 1% -> 10% -> 50% -> 100%
def rollout_feature(feature_name, percentage)
  Flipper.enable_percentage_of_actors(feature_name.to_sym, percentage)
  Rails.logger.info("#{feature_name} rolled out to #{percentage}% of users")
end
```

**ข้อ 9: Zero-downtime Database Migration**
```ruby
# lib/tasks/zero_downtime_migrate.rake
namespace :db do
  task :zero_downtime_migrate do
    ActiveRecord::Base.connection.execute("SET statement_timeout = '30s'")
    Rake::Task['db:migrate'].invoke
    ActiveRecord::Base.connection.execute("SET statement_timeout = '0'")
  end
end
```

**ข้อ 10: Health Check ด้วย Circuit Breaker**
```ruby
class HealthController < ApplicationController
  def readiness
    checks = {}
    
    CircuitBreaker.wrap(:database_check) do
      checks[:database] = { status: test_db ? 'ok' : 'error' }
    end
    
    render json: { status: all_ok?(checks) ? 'ok' : 'degraded', checks: checks },
           status: all_ok?(checks) ? :ok : :service_unavailable
  end
end
```

**ข้อ 11: Monitoring Alerts**
```ruby
# config/initializers/monitoring.rb
ActiveSupport::Notifications.subscribe('process_action.action_controller') do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  duration = event.duration
  
  if duration > 2000
    AlertService.critical("Extremely slow request: #{duration.round}ms")
  elsif duration > 1000
    AlertService.warning("Slow request: #{duration.round}ms")
  end
end
```

**ข้อ 12: Docker Compose สำหรับ Development**
```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    build: .
    command: bundle exec rails s -p 3000 -b 0.0.0.0
    volumes:
      - .:/app
      - bundle:/usr/local/bundle
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - redis
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres/myapp_development
      REDIS_URL: redis://redis:6379/0
  
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  bundle:
  postgres_data:
  redis_data:
```

**ข้อ 13: CI/CD Pipeline**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: bundle exec rspec
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy with Kamal
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
        run: kamal deploy
```

**ข้อ 14: Backup Strategy**
```bash
#!/bin/bash
# scripts/backup_database.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"
S3_BUCKET="myapp-backups"

pg_dump "$DATABASE_URL" | gzip > "${BACKUP_DIR}/db_${DATE}.sql.gz"
aws s3 cp "${BACKUP_DIR}/db_${DATE}.sql.gz" "s3://${S3_BUCKET}/postgres/${DATE}/"

# Delete local backups older than 7 days
find "${BACKUP_DIR}" -name "db_*.sql.gz" -mtime +7 -delete

echo "Backup completed: db_${DATE}.sql.gz"
```

**ข้อ 15: Log Aggregation**
```ruby
# config/initializers/logging.rb
Rails.application.configure do
  config.log_formatter = proc do |severity, time, progname, msg|
    {
      severity: severity,
      timestamp: time.iso8601,
      progname: progname,
      message: msg,
      app: ENV['APP_NAME'],
      environment: Rails.env,
      hostname: Socket.gethostname,
      pid: Process.pid
    }.to_json + "\n"
  end
end
```

**ข้อ 16: Database Connection Pooling**
```ruby
# config/database.yml
production:
  adapter: postgresql
  pool: <%= ENV.fetch("DB_POOL") { [ActiveRecord::Base.connection_pool.size, 25].min } %>
  timeout: 5000
  connect_timeout: 10
  checkout_timeout: 5
  reaping_frequency: 10
  idle_timeout: 60
```

**ข้อ 17: Secret Management**
```ruby
# lib/secret_manager.rb
class SecretManager
  def self.fetch(key)
    if Rails.env.production?
      # AWS Secrets Manager
      client = Aws::SecretsManager::Client.new
      response = client.get_secret_value(secret_id: key)
      JSON.parse(response.secret_string)
    else
      Rails.application.credentials.send(key)
    end
  end
end
```

**ข้อ 18: Deployment Notification**
```ruby
# lib/deployment_notifier.rb
class DeploymentNotifier
  def self.notify(version:, environment:, deployer:)
    SlackNotifier.post(
      channel: '#deployments',
      text: ":rocket: *#{environment}* deployment complete!",
      attachments: [{
        color: 'good',
        fields: [
          { title: 'Version', value: version, short: true },
          { title: 'Deployed by', value: deployer, short: true },
          { title: 'Time', value: Time.current.strftime('%Y-%m-%d %H:%M:%S UTC'), short: false }
        ]
      }]
    )
  end
end
```

**ข้อ 19: Load Balancer Health Check**
```ruby
# app/controllers/health_controller.rb
def liveness
  # รองรับ AWS ALB health check
  # ALB ส่ง request ทุก 10 วินาที
  if request.user_agent.include?('ELB-HealthChecker')
    render plain: 'OK', status: :ok
  else
    render json: { status: 'ok', timestamp: Time.current.iso8601 }
  end
end
```

**ข้อ 20: Complete Deployment Checklist**
```ruby
# lib/tasks/pre_deploy.rake
namespace :deploy do
  task :check do
    errors = []
    
    # Check pending migrations
    pending = ActiveRecord::Tasks::DatabaseTasks.pending_migration_versions
    errors << "Pending migrations: #{pending.join(', ')}" if pending.any?
    
    # Check for missing translations
    # I18n::Tasks::BaseTask.new.missing_keys.count
    
    # Check for deprecated gems
    # Bundler::Audit::Database.update!
    
    if errors.any?
      puts "Pre-deploy checks FAILED:"
      errors.each { |e| puts "  - #{e}" }
      exit 1
    else
      puts "All pre-deploy checks passed!"
    end
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Terraform** - Infrastructure as Code สำหรับ provision AWS resources
2. **Ansible** - Configuration management และ deployment automation
3. **Blue-Green Deployment** - Zero-downtime deployment strategy
4. **Feature Flags** - Flipper gem สำหรับ controlled rollout
5. **Kamal** - Modern zero-downtime deployment สำหรับ Rails
6. **Health Checks** - Liveness, Readiness, Startup probes

DevOps best practices:
- Automate everything ตั้งแต่ infrastructure ถึง deployment
- Test in staging ก่อน production เสมอ
- Feature flags ช่วย reduce deployment risk
- Health checks สำคัญมากสำหรับ automatic recovery
- Monitoring และ alerting ต้องมีตั้งแต่วันแรก
