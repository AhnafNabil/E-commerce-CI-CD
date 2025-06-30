# CI/CD Pipeline Implementation for E-commerce Microservices

This guide provides a complete plug-and-play CI/CD pipeline setup using GitHub Actions with self-hosted runners. The original codebase is in a public repository, so you'll create your own repository and set up automatic deployment with secure environment variable management.

## Overview

The CI/CD pipeline automatically deploys your microservices when changes are pushed to the main branch. It intelligently detects which services changed and deploys only those services, with secure credential management through GitHub Secrets.

```
Public Repo → Your Repo → Code Push → GitHub Actions → Self-Hosted Runner → Smart Deployment → Running Services
```

---

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Ubuntu/Linux server (minimum 4GB RAM)
- [ ] GitHub account
- [ ] Terminal access to your server
- [ ] Basic Git knowledge
- [ ] Mailtrap.io account (for email notifications)

---

## Step-by-Step Implementation

### **Step 1: Create Your Own Repository**

**1.1 Create a new GitHub repository:**

1. Go to [GitHub](https://github.com)
2. Click the **"+"** icon in the top right
3. Select **"New repository"**
4. Fill in the details:
   - **Repository name**: `my-ecommerce-microservices` (or any name you prefer)
   - **Description**: `E-commerce microservices with CI/CD pipeline`
   - **Visibility**: Public or Private (your choice)
   - **Initialize**: ❌ **DO NOT** check "Add a README file" (leave empty)
5. Click **"Create repository"**

**1.2 Prepare the codebase locally:**

```bash
# Clone the original public repository
git clone https://github.com/poridhioss/E-commerce-Microservices-with-Kafka.git my-ecommerce-repo

# Navigate into the cloned directory
cd my-ecommerce-repo

# Remove the original git history
rm -rf .git

# Initialize new git repository
git init
git branch -M main

# Add your repository as remote (replace with your actual repository URL)
git remote add origin https://github.com/YOUR-USERNAME/my-ecommerce-microservices.git
```

**Important:** Do NOT push yet! We'll first add the CI/CD workflows.

---

### **Step 2: Server Preparation**

**2.1 Connect to your server and update it:**
```bash
sudo apt update && sudo apt upgrade -y
```

**2.2 Install Docker and Docker Compose:**
```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installations
docker --version
docker-compose --version
```

**2.3 Create a dedicated user for GitHub Actions:**
```bash
# Create user
sudo adduser github-runner

# When prompted, enter:
# - Password: [choose a secure password]
# - Full name: GitHub Runner
# - Other fields: [press Enter to skip]

# Add user to required groups
sudo usermod -aG sudo github-runner
sudo usermod -aG docker github-runner

# Switch to the new user
su - github-runner
```

---

### **Step 3: GitHub Runner Setup**

**3.1 Get runner configuration from GitHub:**

1. Go to **your** GitHub repository (the empty one you just created)
2. Navigate to **Settings → Actions → Runners**
3. Click **New self-hosted runner**
4. Select **Linux**
5. **Keep this page open** - you'll need the commands shown

**3.2 Download and configure the runner:**

```bash
# Create runner directory
mkdir actions-runner && cd actions-runner

# Copy the download command from GitHub and run it
# Example (replace with actual command from GitHub):
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Extract the installer
tar xzf actions-runner-linux-x64-2.311.0.tar.gz

# Copy the config command from GitHub and run it
# IMPORTANT: Use YOUR repository URL and token from GitHub
./config.sh --url https://github.com/YOUR-USERNAME/YOUR-REPO-NAME --token YOUR-TOKEN
```

**3.3 Configure the runner when prompted:**
```bash
# Enter the name of the runner group: [Press Enter]
# Enter the name of runner: ecommerce-runner
# Enter any additional labels: deployment,docker
# Enter name of work folder: [Press Enter]
```

**3.4 Install as a service:**
```bash
# Install the service
sudo ./svc.sh install

# Start the service
sudo ./svc.sh start

# Verify it's running
sudo ./svc.sh status
```

**3.5 Verify in GitHub:**
Go back to **Settings → Actions → Runners** in **your** repository. You should see your runner showing as "Idle".

---

### **Step 4: Create CI/CD Scripts (Locally)**

**4.1 Go back to your local repository directory:**
```bash
# Navigate to your local repository
cd my-ecommerce-repo  # or whatever you named your directory
```

**4.2 Create the scripts directory:**
```bash
mkdir -p .github/scripts
```

**4.3 Create the change detection script:**

Create `.github/scripts/detect-changes.sh`:
```bash
cat > .github/scripts/detect-changes.sh << 'EOF'
#!/bin/bash

# detect-changes.sh - Detects which services have changed

set -e

# Service mapping - maps file paths to services
declare -A SERVICE_MAP=(
    ["product-service"]="product-service"
    ["inventory-service"]="inventory-service"
    ["order-service"]="order-service"
    ["user-service"]="user-service"
    ["notification-service"]="notification-service"
    ["nginx"]="nginx-gateway"
    ["docker-compose.yml"]="all"
    ["events"]="product-service inventory-service"
)

# Get changed files between current commit and previous commit
CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD 2>/dev/null || git ls-files)

echo "Changed files:"
echo "$CHANGED_FILES"
echo ""

SERVICES_TO_DEPLOY=""
DEPLOY_ALL=false

# Check each changed file
while IFS= read -r file; do
    if [ -z "$file" ]; then
        continue
    fi
    
    echo "Analyzing: $file"
    
    # Check if it's a service directory
    for service_dir in "${!SERVICE_MAP[@]}"; do
        if [[ "$file" == "$service_dir"* ]]; then
            services="${SERVICE_MAP[$service_dir]}"
            if [ "$services" = "all" ]; then
                DEPLOY_ALL=true
                break
            else
                for service in $services; do
                    if [[ ! " $SERVICES_TO_DEPLOY " =~ " $service " ]]; then
                        SERVICES_TO_DEPLOY="$SERVICES_TO_DEPLOY $service"
                    fi
                done
            fi
            break
        fi
    done
done <<< "$CHANGED_FILES"

# Output results
if [ "$DEPLOY_ALL" = true ]; then
    echo "DEPLOY_ALL=true" >> $GITHUB_OUTPUT
    echo "SERVICES=" >> $GITHUB_OUTPUT
    echo "Deploying all services due to infrastructure changes"
else
    SERVICES_TO_DEPLOY=$(echo "$SERVICES_TO_DEPLOY" | xargs)
    echo "DEPLOY_ALL=false" >> $GITHUB_OUTPUT
    echo "SERVICES=$SERVICES_TO_DEPLOY" >> $GITHUB_OUTPUT
    echo "Services to deploy: $SERVICES_TO_DEPLOY"
fi
EOF
```

**4.4 Create the deployment script:**

Create `.github/scripts/deploy.sh`:
```bash
cat > .github/scripts/deploy.sh << 'EOF'
#!/bin/bash

# deploy.sh - Main deployment script with environment variable injection

set -e

DEPLOY_ALL=${1:-false}
SERVICES=${2:-""}
APP_DIR="/opt/ecommerce-app"

echo "=== E-commerce Microservices Deployment ==="
echo "Deploy All: $DEPLOY_ALL"
echo "Services: $SERVICES"
echo "Working Directory: $APP_DIR"
echo ""

# Change to application directory
cd "$APP_DIR"

# Pull latest code
echo "Pulling latest code..."
git fetch origin
git reset --hard origin/main
echo "✓ Code updated"

# Create/Update environment files with secrets
create_env_files() {
    echo "Setting up environment variables..."
    
    # Update notification service .env file
    if [ -f "notification-service/.env" ]; then
        # Backup original
        cp notification-service/.env notification-service/.env.backup
        
        # Update SMTP credentials if provided
        if [ -n "$SMTP_USER" ]; then
            sed -i "s/^SMTP_USER=.*/SMTP_USER=${SMTP_USER}/" notification-service/.env
        fi
        if [ -n "$SMTP_PASSWORD" ]; then
            sed -i "s/^SMTP_PASSWORD=.*/SMTP_PASSWORD=${SMTP_PASSWORD}/" notification-service/.env
        fi
        if [ -n "$EMAIL_FROM" ]; then
            sed -i "s/^EMAIL_FROM=.*/EMAIL_FROM=${EMAIL_FROM}/" notification-service/.env
        fi
        
        echo "✓ Updated notification service environment variables"
    else
        echo "⚠️ Warning: notification-service/.env not found"
    fi
}

# Function to deploy all services
deploy_all_services() {
    echo "Deploying all services..."
    
    create_env_files
    
    echo "Stopping all services..."
    docker-compose down
    
    echo "Pulling latest images..."
    docker-compose pull 2>/dev/null || echo "Building images locally..."
    
    echo "Building and starting all services..."
    docker-compose up -d --build
    
    echo "✓ All services deployed"
}

# Function to deploy specific services
deploy_specific_services() {
    local services=($1)
    
    echo "Deploying specific services: ${services[*]}"
    
    # Update env files if notification-service is being deployed
    for service in "${services[@]}"; do
        if [ "$service" = "notification-service" ]; then
            create_env_files
            break
        fi
    done
    
    for service in "${services[@]}"; do
        if [ -z "$service" ]; then
            continue
        fi
        
        echo "Deploying service: $service"
        
        # Pull latest image for the service
        docker-compose pull "$service" 2>/dev/null || echo "No pre-built image for $service, will build locally"
        
        # Build and restart the service
        docker-compose up -d --build --no-deps "$service"
        
        echo "✓ Service $service deployed"
        
        # Wait a moment for service to start
        sleep 5
    done
}

# Execute deployment
if [ "$DEPLOY_ALL" = "true" ]; then
    deploy_all_services
else
    if [ -n "$SERVICES" ]; then
        deploy_specific_services "$SERVICES"
    else
        echo "No services to deploy"
    fi
fi

echo ""
echo "=== Deployment completed successfully ==="

# Show running services
echo ""
echo "Running services:"
docker-compose ps
EOF
```

---

### **Step 5: Create GitHub Actions Workflow (Locally)**

**5.1 Create the workflow directory:**
```bash
mkdir -p .github/workflows
```

**5.2 Create the main deployment workflow:**

Create `.github/workflows/deploy.yml`:
```bash
cat > .github/workflows/deploy.yml << 'EOF'
name: Deploy E-commerce Microservices

on:
  push:
    branches: [ main ]

jobs:
  detect-changes:
    runs-on: self-hosted
    outputs:
      deploy_all: ${{ steps.detect.outputs.DEPLOY_ALL }}
      services: ${{ steps.detect.outputs.SERVICES }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Detect changes
        id: detect
        run: |
          chmod +x .github/scripts/detect-changes.sh
          .github/scripts/detect-changes.sh

      - name: Display detection results
        run: |
          echo "Deploy All: ${{ steps.detect.outputs.DEPLOY_ALL }}"
          echo "Services: ${{ steps.detect.outputs.SERVICES }}"

  deploy:
    needs: detect-changes
    runs-on: self-hosted
    steps:
      - name: Setup application directory
        run: |
          sudo mkdir -p /opt/ecommerce-app
          sudo chown -R github-runner:github-runner /opt/ecommerce-app
          
      - name: Initialize or update repository
        run: |
          cd /opt/ecommerce-app
          
          if [ ! -d ".git" ]; then
            echo "🔄 First time setup - cloning repository..."
            git clone https://github.com/${{ github.repository }}.git .
          else
            echo "✅ Repository exists - updating..."
            git fetch origin
            git reset --hard origin/main
          fi
          
          echo "📁 Repository status:"
          git status --short

      - name: Deploy services
        env:
          SMTP_USER: ${{ secrets.SMTP_USER }}
          SMTP_PASSWORD: ${{ secrets.SMTP_PASSWORD }}
          EMAIL_FROM: ${{ secrets.EMAIL_FROM }}
        run: |
          cd /opt/ecommerce-app
          chmod +x .github/scripts/deploy.sh
          .github/scripts/deploy.sh "${{ needs.detect-changes.outputs.deploy_all }}" "${{ needs.detect-changes.outputs.services }}"

      - name: Deployment summary
        run: |
          echo "🚀 Deployment completed successfully!"
          echo "Deploy All: ${{ needs.detect-changes.outputs.deploy_all }}"
          echo "Services: ${{ needs.detect-changes.outputs.services }}"
          cd /opt/ecommerce-app
          echo ""
          echo "Running services:"
          docker-compose ps
EOF
```

---

### **Step 6: Configure GitHub Secrets**

**6.1 Get your Mailtrap credentials:**

1. Go to [Mailtrap.io](https://mailtrap.io) and sign in
2. Navigate to **Email Testing → Inboxes**
3. Select your inbox
4. Go to **SMTP Settings**
5. Copy the username and password

**6.2 Add secrets to your GitHub repository:**

1. Go to your GitHub repository
2. Navigate to **Settings → Secrets and variables → Actions**
3. Click **New repository secret**
4. Add these secrets one by one:

```
Name: SMTP_USER
Value: [your_mailtrap_username_from_step_6.1]

Name: SMTP_PASSWORD  
Value: [your_mailtrap_password_from_step_6.1]

Name: EMAIL_FROM
Value: notifications@yourstore.com
```

**6.3 Verify secrets are added:**
You should see three secrets listed in your repository secrets section.

---

### **Step 7: First Commit and Push**

**7.1 Add all files and make the initial commit:**
```bash
# Add all files including the original codebase and new CI/CD files
git add .

# Check what will be committed
git status

# Make the initial commit
git commit -m "Initial commit: Add e-commerce microservices with CI/CD pipeline"

# Push to your repository
git push -u origin main
```

---

### **Step 8: Verify Initial Deployment**

**8.1 Check the workflow execution:**

1. Go to **your** GitHub repository
2. Click on the **Actions** tab
3. You should see a workflow run called "Deploy E-commerce Microservices"
4. Click on it to see the details

**8.2 Monitor the deployment:**

The workflow will:
- **🔄 Clone repository** to server (first time)
- **📝 Detect changes** (all files are new, so deploy all)
- **🔧 Inject environment variables** for notification service
- **🚀 Deploy all services**
- **✅ Show final status**

**8.3 Verify services are running:**

On your server, check that services are running:
```bash
# Switch to github-runner user
su - github-runner

# Check services
cd /opt/ecommerce-app
docker-compose ps
```

You should see all your services in "Up" status.

**8.4 Test notification service:**

```bash
# Test the notification service with updated credentials
curl -X POST "http://localhost/api/v1/notifications/test" | jq .
```

You should receive a test email in your Mailtrap inbox.

---

### **Step 9: Test the Pipeline with Service Changes**

**9.1 Test single service deployment:**

On your local machine, make a change to test the pipeline:

```bash
# Make a small change to product service
echo "# Updated $(date)" >> product-service/README.md

# Commit and push
git add product-service/README.md
git commit -m "Test: Update product service documentation"
git push origin main
```

**9.2 Monitor the workflow:**

1. Go to **Actions** tab in your GitHub repository
2. Watch the new workflow run
3. Verify that only `product-service` is redeployed

**9.3 Verify selective deployment:**

On your server:
```bash
cd /opt/ecommerce-app
docker-compose ps

# Check the logs to see only product-service was restarted
docker-compose logs --tail=20 product-service
```

---

### **Step 10: Test Infrastructure Change**

**10.1 Test full stack deployment:**

```bash
# Make a change that triggers full deployment
echo "# Configuration updated $(date)" >> docker-compose.yml

# Commit and push
git add docker-compose.yml
git commit -m "Test: Update docker-compose configuration"
git push origin main
```

**10.2 Verify full deployment:**

1. Check GitHub Actions - should show "Deploy All: true"
2. On server, verify all services were restarted:
```bash
cd /opt/ecommerce-app
docker-compose ps
```

All services should show recent restart times.

---

### **Step 11: Test Notification Service Update**

**11.1 Test notification-specific deployment:**

```bash
# Change notification service to test env var injection
echo "# Updated notification logic $(date)" >> notification-service/README.md

# Commit and push
git add notification-service/README.md
git commit -m "Test: Update notification service"
git push origin main
```

**11.2 Verify environment variable injection:**

1. Check GitHub Actions logs - should show "✓ Updated notification service environment variables"
2. Test email functionality:
```bash
curl -X POST "http://localhost/api/v1/notifications/test" | jq .
```

**11.3 Verify in Mailtrap:**
Check your Mailtrap inbox for the test email with your custom credentials.

---

## Pipeline Testing Summary

Your testing should follow this sequence:

### **✅ Test 1: Initial Deployment** 
- **Action**: First push with all code + CI/CD workflows
- **Expected**: All services deployed, repository auto-cloned
- **Verify**: `docker-compose ps` shows all services running

### **✅ Test 2: Single Service Change** 
- **Action**: Modify only product-service 
- **Expected**: Only product-service redeployed
- **Verify**: GitHub Actions logs + docker logs

### **✅ Test 3: Infrastructure Change** 
- **Action**: Modify docker-compose.yml
- **Expected**: All services redeployed
- **Verify**: All containers restarted

### **✅ Test 4: Notification Service** 
- **Action**: Modify notification-service
- **Expected**: Only notification-service redeployed + env vars updated
- **Verify**: Service restarted, test email works with new credentials

### **✅ Test 5: Multiple Services** 
```bash
# Change multiple services
echo "# Updated $(date)" >> inventory-service/README.md
echo "# Updated $(date)" >> order-service/README.md

git add inventory-service/ order-service/
git commit -m "Test: Update multiple services"
git push origin main
```
- **Expected**: Only those two services redeployed
- **Verify**: Only affected services restarted

---

## Working with Your Repository

### **Development Workflow:**

```bash
# Clone your repository to local machine for development
git clone https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
cd YOUR-REPO-NAME

# Make changes to any service
echo "new feature" >> product-service/app/main.py

# Commit and push - automatic deployment happens!
git add .
git commit -m "Add new feature"
git push origin main
```

### **Getting Updates from Original Repository:**

```bash
# Add the original repository as a remote
git remote add upstream https://github.com/poridhioss/E-commerce-Microservices-with-Kafka.git

# Fetch updates from original repository
git fetch upstream

# Merge updates into your main branch
git merge upstream/main

# Push the updates to your repository (triggers deployment)
git push origin main
```

### **Updating Environment Variables:**

1. **Update GitHub Secrets:**
   - Go to Settings → Secrets and variables → Actions
   - Update `SMTP_USER`, `SMTP_PASSWORD`, or `EMAIL_FROM`

2. **Trigger Deployment:**
   ```bash
   # Make any change to notification service
   echo "# Trigger env update $(date)" >> notification-service/README.md
   git add notification-service/README.md
   git commit -m "Update notification service environment"
   git push origin main
   ```

---

## Verification Checklist

After completing all steps, verify everything works:

- [ ] ✅ Your GitHub repository contains all code plus CI/CD workflows
- [ ] ✅ GitHub runner shows as "Idle" in Settings → Actions → Runners
- [ ] ✅ GitHub Secrets are configured (SMTP_USER, SMTP_PASSWORD, EMAIL_FROM)
- [ ] ✅ Initial deployment worked (all services running)
- [ ] ✅ Single service change triggers selective deployment
- [ ] ✅ Infrastructure change triggers full deployment  
- [ ] ✅ Notification service gets updated environment variables
- [ ] ✅ Test email works with your Mailtrap credentials
- [ ] ✅ Multiple service changes deploy only affected services
- [ ] ✅ Docker containers are running: `docker-compose ps`
- [ ] ✅ Application is accessible at configured ports
- [ ] ✅ All workflow runs show successful completion

---

## Troubleshooting

### **Common Issues and Solutions:**

**1. Runner not appearing in GitHub:**
```bash
# Check runner service
sudo systemctl status actions.runner.*

# Restart if needed
sudo systemctl restart actions.runner.*
```

**2. Repository clone fails:**
```bash
# Fix permissions
sudo chown -R github-runner:github-runner /opt/ecommerce-app

# Check Git configuration
cd /opt/ecommerce-app
git config user.name "GitHub Runner"
git config user.email "runner@example.com"
```

**3. Services won't start:**
```bash
# Check Docker status
sudo systemctl status docker

# Check for port conflicts
netstat -tlnp | grep :80

# View service logs
cd /opt/ecommerce-app
docker-compose logs
```

**4. Environment variables not updating:**
```bash
# Check secrets are set in GitHub
# Verify workflow logs show environment variable injection
# Check notification service .env file:
cd /opt/ecommerce-app
cat notification-service/.env
```

**5. Email not working:**
```bash
# Test notification endpoint
curl -X POST "http://localhost/api/v1/notifications/test"

# Check notification service logs
docker-compose logs notification-service

# Verify Mailtrap credentials in GitHub Secrets
```

---

## Advanced Features (Optional)

### **Add Slack/Discord Notifications:**

```yaml
# Add to your workflow after deployment
- name: Notify deployment status
  if: always()
  run: |
    STATUS="${{ job.status }}"
    if [ "$STATUS" = "success" ]; then
      MESSAGE="✅ Deployment successful"
    else
      MESSAGE="❌ Deployment failed"
    fi
    
    curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"$MESSAGE for ${{ github.repository }}\"}" \
    ${{ secrets.SLACK_WEBHOOK_URL }}
```

### **Add Rollback Capability:**

Create `.github/workflows/rollback.yml`:
```yaml
name: Rollback Deployment

on:
  workflow_dispatch:
    inputs:
      commit_hash:
        description: 'Commit hash to rollback to'
        required: true

jobs:
  rollback:
    runs-on: self-hosted
    steps:
      - name: Rollback to specific commit
        run: |
          cd /opt/ecommerce-app
          git checkout ${{ github.event.inputs.commit_hash }}
          docker-compose down
          docker-compose up -d --build
```

### **Add Health Checks:**

```yaml
# Add after deployment
- name: Health check
  run: |
    sleep 30
    curl -f http://localhost/health || exit 1
    echo "✅ All services healthy"
```

---

## Security Best Practices

### **Repository Security:**
- ✅ Use GitHub Secrets for sensitive data
- ✅ Never commit credentials to code
- ✅ Regularly rotate secrets
- ✅ Use branch protection rules

### **Server Security:**
- ✅ Dedicated user for runner
- ✅ Firewall configuration
- ✅ Regular security updates
- ✅ Monitor access logs

### **Docker Security:**
- ✅ Use official base images
- ✅ Regular image updates
- ✅ Scan for vulnerabilities
- ✅ Limit container privileges

---

## What You've Accomplished

🎉 **Congratulations!** You now have a **production-ready, automated CI/CD pipeline** that:

✅ **Fully Automated Deployment** - Deploys on every push to main  
✅ **Smart Change Detection** - Only deploys affected services  
✅ **Secure Credential Management** - Environment variables via GitHub Secrets  
✅ **Self-Hosted Infrastructure** - Complete control over deployment environment  
✅ **Zero Manual Intervention** - Fully automated from code to production  
✅ **Scalable Architecture** - Easily add new services or modify existing ones  
✅ **Production Ready** - Handles failures, rollbacks, and service dependencies  

### **Your Development Workflow is Now:**

```
1. Write Code → 2. Git Push → 3. Automatic Deployment → 4. Live Application ✨
```

**No servers to manage, no manual deployments, no credential worries!**

Your e-commerce microservices platform is now ready for serious development and production use! 🚀

---

## Support and Maintenance

### **Regular Maintenance:**
- Monitor GitHub Actions for failed deployments
- Update runner software quarterly
- Rotate secrets annually
- Monitor server resources

### **Scaling Considerations:**
- Add more runners for parallel deployments
- Implement blue-green deployment for zero downtime
- Add monitoring and alerting
- Consider container orchestration (Kubernetes) for larger scale

**Happy coding and deploying!** 🎊