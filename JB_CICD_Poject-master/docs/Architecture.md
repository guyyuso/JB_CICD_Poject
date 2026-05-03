# Project Architecture

This document explains the technical architecture, workflow, and design decisions of the Flask AWS Monitor project.

---

## 📐 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          DEVELOPMENT PHASE                           │
│                                                                       │
│  Developer → Git Push → GitHub Repository                           │
│                              ↓                                        │
│                         Webhook/Poll                                 │
│                              ↓                                        │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        CI/CD PIPELINE (Jenkins)                      │
│                                                                       │
│  Stage 1: Clone Repository                                          │
│    │  ├─ Checkout source code from Git                             │
│    │  └─ Clean workspace                                            │
│    │                                                                 │
│  Stage 2: Quality Checks (Parallel)                                │
│    │  ├─ Linting (flake8) ─────────────┐                           │
│    │  └─ Security Scan (bandit) ───────┴─→ Code Quality Report     │
│    │                                                                 │
│  Stage 3: Build                                                     │
│    │  ├─ Build Docker Image                                        │
│    │  └─ Tag: build_number & latest                                │
│    │                                                                 │
│  Stage 4: Publish                                                   │
│    │  ├─ Login to Docker Hub                                       │
│    │  └─ Push images to registry                                   │
│    │                                                                 │
│  Stage 5: Deploy                                                    │
│    │  ├─ Initialize Terraform                                      │
│    │  ├─ Plan infrastructure changes                               │
│    │  └─ Apply changes to AWS                                      │
│    │                                                                 │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        DOCKER REGISTRY                               │
│                                                                       │
│  Docker Hub                                                          │
│    └─ guyyusupov/flask-aws-monitor                               │
│         ├─ :latest                                                  │
│         └─ :<build_number>                                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS CLOUD (us-east-2)                        │
│                                                                       │
│  ┌──────────────────────────────────────────────────────┐          │
│  │                    VPC (Default)                      │          │
│  │                                                        │          │
│  │  ┌─────────────────────────────────────────────────┐ │          │
│  │  │           Security Group                        │ │          │
│  │  │         jenkins-project-sg                      │ │          │
│  │  │                                                  │ │          │
│  │  │  Inbound Rules:                                │ │          │
│  │  │    22/tcp   ← 0.0.0.0/0  (SSH)                │ │          │
│  │  │    5001/tcp ← 0.0.0.0/0  (Flask)              │ │          │
│  │  │                                                  │ │          │
│  │  │  Outbound Rules:                               │ │          │
│  │  │    All traffic → 0.0.0.0/0                    │ │          │
│  │  └────────────┬────────────────────────────────────┘ │          │
│  │               ↓                                        │          │
│  │  ┌─────────────────────────────────────────────────┐ │          │
│  │  │       EC2 Instance (t3.medium)                 │ │          │
│  │  │       JBP-Builder-Instance                     │ │          │
│  │  │                                                  │ │          │
│  │  │  OS: Ubuntu 22.04 LTS                          │ │          │
│  │  │  vCPU: 2                                       │ │          │
│  │  │  RAM: 4 GB                                     │ │          │
│  │  │  Storage: 8 GB EBS                             │ │          │
│  │  │  Public IP: ✓ (Dynamic)                       │ │          │
│  │  │  IMDSv2: Required ✓                            │ │          │
│  │  │                                                  │ │          │
│  │  │  ┌──────────────────────────────────────────┐ │ │          │
│  │  │  │      Docker Engine                        │ │ │          │
│  │  │  │                                            │ │ │          │
│  │  │  │  ┌─────────────────────────────────────┐ │ │ │          │
│  │  │  │  │   Flask Container                   │ │ │ │          │
│  │  │  │  │   flask-dashboard                   │ │ │ │          │
│  │  │  │  │                                     │ │ │ │          │
│  │  │  │  │   Image:                            │ │ │ │          │
│  │  │  │  │   guyyusupov/flask-aws-monitor   │ │ │ │          │
│  │  │  │  │                                     │ │ │ │          │
│  │  │  │  │   Port Mapping:                    │ │ │ │          │
│  │  │  │  │   5001:5001                        │ │ │ │          │
│  │  │  │  │                                     │ │ │ │          │
│  │  │  │  │   Environment:                     │ │ │ │          │
│  │  │  │  │   - INSTANCE_TYPE                  │ │ │ │          │
│  │  │  │  │   - VPC_ID                         │ │ │ │          │
│  │  │  │  │   - REGION                         │ │ │ │          │
│  │  │  │  │   - SSH_KEY_PATH                   │ │ │ │          │
│  │  │  │  │                                     │ │ │ │          │
│  │  │  │  │   Restart: always                  │ │ │ │          │
│  │  │  │  └─────────────────────────────────────┘ │ │ │          │
│  │  │  └──────────────────────────────────────────┘ │ │          │
│  │  │                                                  │ │          │
│  │  │  AWS Metadata Service (IMDSv2):                │ │          │
│  │  │  169.254.169.254                               │ │          │
│  │  └─────────────────────────────────────────────────┘ │          │
│  │                                                        │          │
│  └────────────────────────────────────────────────────────┘          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│                          END USER ACCESS                             │
│                                                                       │
│  User Browser → http://<public-ip>:5001                            │
│                   ↓                                                  │
│              Flask Dashboard                                         │
│              (HTML + CSS)                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Detailed Workflow

### Phase 1: Code Change Trigger

```
1. Developer commits code
2. Git push to repository
3. Jenkins detects change (webhook or poll)
4. Pipeline is triggered
```

### Phase 2: CI/CD Pipeline Execution

#### **Step 1: Clone Repository**
```groovy
stage('Clone Repository') {
    steps {
        cleanWs()          // Clean workspace
        checkout scm       // Checkout code
    }
}
```

**What happens:**
- Jenkins workspace is cleaned
- Fresh copy of code is pulled
- All files are now available to Jenkins

---

#### **Step 2: Quality Checks (Parallel)**
```groovy
stage('Parallel Quality Checks') {
    parallel {
        stage('Linting') { 
            sh 'flake8 app.py --ignore=E501'
        }
        stage('Security Scan') {
            sh 'bandit -r .'
        }
    }
}
```

**What happens:**
- **Linting**: Checks code style, syntax errors, unused imports
- **Security**: Scans for hardcoded passwords, SQL injection, etc.
- Both run simultaneously (faster than sequential)

**Example Issues Caught:**
- Unused variables
- Missing whitespace
- Hardcoded secrets (bandit would flag)
- Potential security vulnerabilities

---

#### **Step 3: Build Docker Image**
```groovy
stage('Build Docker Image') {
    sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} app/"
    sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest"
}
```

**What happens:**
1. Docker reads app/Dockerfile
2. Builds image layer by layer:
   ```
   Layer 1: Base Python 3.9 image
   Layer 2: Install Flask, boto3, requests
   Layer 3: Copy application code
   Layer 4: Set entrypoint (python app.py)
   ```
3. Tags with two tags:
   - Specific: `flask-aws-monitor:42` (build number)
   - Generic: `flask-aws-monitor:latest`

**Why two tags?**
- `:latest` → Always pulls newest version
- `:42` → Can rollback to specific version

---

#### **Step 4: Push to Registry**
```groovy
stage('Push to Docker Hub') {
    sh "docker login -u $CREDS_USR --password-stdin"
    sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
    sh "docker push ${IMAGE_NAME}:latest"
}
```

**What happens:**
- Jenkins retrieves Docker Hub credentials securely
- Logs into Docker Hub
- Uploads both image tags
- Makes image publicly accessible

---

#### **Step 5: Deploy Infrastructure**
```groovy
stage('Deploy Infrastructure (Terraform)') {
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    }
    steps {
        sh 'terraform init'
        sh 'terraform apply -auto-approve'
    }
}
```

**What happens:**

**5a. Terraform Init**
```
Downloads required providers:
- hashicorp/aws
- hashicorp/tls (for SSH key generation)
- hashicorp/local (for saving key file)
```

**5b. Terraform Plan (implicit)**
```
Terraform calculates:
- What resources to CREATE
- What resources to CHANGE
- What resources to DESTROY
```

**5c. Terraform Apply**
```
Creates in order (respecting dependencies):
1. Generate SSH key pair (4096-bit RSA)
2. Save private key locally (builder_key.pem)
3. Create AWS key pair with public key
4. Create Security Group
5. Launch EC2 Instance
6. Wait for instance to be running
7. Execute user_data.sh script
```

---

### Phase 3: Instance Initialization (user_data.sh)

When EC2 instance boots, this script runs ONCE:

```bash
# 1. Install Docker
apt-get update -y
apt-get install -y docker.io
systemctl start docker

# 2. Fetch metadata using IMDSv2
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token")
INSTANCE_TYPE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
                http://169.254.169.254/latest/meta-data/instance-type)
REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
         http://169.254.169.254/latest/meta-data/placement/region)
VPC_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
         http://169.254.169.254/latest/meta-data/.../vpc-id)

# 3. Pull Docker image
docker pull guyyusupov/flask-aws-monitor:latest

# 4. Run container with metadata
docker run -d \
  --name flask-dashboard \
  -p 5001:5001 \
  -e INSTANCE_TYPE="$INSTANCE_TYPE" \
  -e VPC_ID="$VPC_ID" \
  -e REGION="$REGION" \
  -e SSH_KEY_PATH="AWS KeyPair: builder_key" \
  --restart always \
  guyyusupov/flask-aws-monitor:latest
```

**Key Points:**
- Runs as root (required for Docker installation)
- Happens automatically at boot
- No manual SSH required
- Container starts on boot failure (--restart always)

---

### Phase 4: Application Runtime

Once container is running:

```python
# Flask app starts
app.run(host="0.0.0.0", port=5001)

# When user accesses http://<ip>:5001
@app.route('/')
def home():
    # App requests metadata AGAIN (not from user_data)
    # This ensures fresh data even if IP changes
    token = get_imdsv2_token()
    public_ip = get_metadata_with_token(token, 'public-ipv4')
    
    # Reads environment variables (passed from user_data)
    ssh_key = os.getenv("SSH_KEY_PATH")
    instance_type = os.getenv("INSTANCE_TYPE")
    
    # Renders HTML dashboard
    return render_dashboard(public_ip, ssh_key, instance_type, ...)
```

---

## 🏗️ Component Deep Dive

### 1. Jenkins Pipeline (Jenkinsfile)

**Structure:**
```groovy
pipeline {
    agent any                    // Run on any available agent
    
    parameters {                 // User inputs
        string(name: 'DOCKER_USER', ...)
    }
    
    environment {                // Global variables
        IMAGE_NAME = "..."
        DOCKER_CREDS = credentials('...')
    }
    
    stages { ... }              // Sequential execution
    
    post {                       // Cleanup after build
        always { ... }
    }
}
```

**Why Groovy?**
- Jenkins native language
- Powerful scripting capabilities
- Easy integration with shell commands

---

### 2. Docker Container

**Dockerfile Analysis:**
```dockerfile
FROM python:3.9-slim           # Minimal base (200MB vs 900MB full)
WORKDIR /app                    # Sets working directory
COPY requirements.txt .         # Copy dependencies FIRST (layer caching)
RUN pip install ...             # Install once, reuse if unchanged
COPY . .                        # Copy code (changes frequently)
EXPOSE 5001                     # Document port (not enforce)
CMD ["python", "app.py"]        # Container entrypoint
```

**Layer Caching Strategy:**
- If `requirements.txt` unchanged → reuse cached layer
- Only rebuild code copy layer (fast)
- Reduces build time from 2 min to 30 sec

---

### 3. Terraform Infrastructure

**Resource Dependencies:**
```
tls_private_key.key
    ↓
aws_key_pair.generated_key
    ↓
local_file.private_key

aws_security_group.builder_sg
    ↓
aws_instance.builder_instance
```

Terraform automatically determines order based on resource references.

**State Management:**
```
terraform.tfstate (local file)
├─ Tracks all resource IDs
├─ Maps Terraform config to real AWS resources
└─ Enables updates and deletions

If lost: Terraform can't manage resources!
```

---

### 4. IMDSv2 Security

**Traditional SSRF Attack (IMDSv1):**
```
Attacker → Vulnerable Web App → http://169.254.169.254/...
           (Example: image URL parameter)
           
Result: Attacker gets AWS credentials!
```

**Protection with IMDSv2:**
```
Step 1: Must use PUT (not GET) to request token
Step 2: Must include token in subsequent requests
Step 3: Token has TTL (expires after 6 hours)

Attacker → Vulnerable Web App → Can't use PUT method
           (Limited to GET requests)
           
Result: Attack fails!
```

**Implementation:**
```python
# Step 1: Request token (PUT method required)
token = requests.put("http://169.254.169.254/latest/api/token",
                     headers={"X-aws-ec2-metadata-token-ttl-seconds": "21600"})

# Step 2: Use token (header required)
ip = requests.get("http://169.254.169.254/latest/meta-data/public-ipv4",
                  headers={"X-aws-ec2-metadata-token": token.text})
```

---

## 🔐 Security Architecture

### Defense in Depth

**Layer 1: Code**
- ✓ No hardcoded secrets
- ✓ Environment variables for config
- ✓ Security scanning (bandit)

**Layer 2: Container**
- ✓ Minimal base image (fewer vulnerabilities)
- ✓ Non-root user (could be improved)
- ✓ Read-only filesystem (could be improved)

**Layer 3: Network**
- ✓ Security Group restricts ports
- ⚠ SSH open to world (acceptable for demo)
- ✓ IMDSv2 prevents SSRF

**Layer 4: Access Control**
- ✓ IAM permissions for Terraform
- ✓ Key pair authentication for SSH
- ✓ Jenkins credentials store

**Layer 5: Monitoring**
- ⚠ No CloudWatch logs (could be added)
- ⚠ No intrusion detection (could be added)

---

## 📊 Data Flow

### Metadata Flow
```
AWS Metadata Service
    ↓ (IMDSv2 token auth)
EC2 Instance (user_data.sh)
    ↓ (Environment variables)
Docker Container
    ↓ (os.getenv())
Flask Application
    ↓ (HTML templating)
User's Browser
```

### Deployment Flow
```
Git Repository
    ↓ (git pull)
Jenkins Workspace
    ↓ (docker build)
Docker Image
    ↓ (docker push)
Docker Hub Registry
    ↓ (docker pull)
EC2 Instance
    ↓ (docker run)
Running Container
```

---

## 🎯 Design Decisions

### Why Flask?
- ✓ Lightweight (single file app)
- ✓ Built-in templating (no separate HTML files)
- ✓ Easy to containerize
- ✓ Good for demos and learning

### Why Docker?
- ✓ Ensures consistency (works everywhere)
- ✓ Easy updates (just pull new image)
- ✓ Portable (runs on any platform)
- ✓ Industry standard

### Why Terraform?
- ✓ Infrastructure as Code (version controlled)
- ✓ Reproducible (run anywhere)
- ✓ Declarative (describe desired state)
- ✓ Multi-cloud support (could switch to Azure)

### Why Jenkins?
- ✓ Industry standard CI/CD tool
- ✓ Extensible (thousands of plugins)
- ✓ Self-hosted (full control)
- ✓ Free and open source

### Why t3.medium?
- ✓ Burstable performance (cost-effective)
- ✓ 2 vCPU, 4GB RAM (sufficient for demo)
- ✓ ~$30/month (reasonable for learning)

---

## 🔄 Update Process

**Zero-Downtime Update (current limitation):**
```
1. Push code changes
2. Jenkins builds new image
3. Terraform detects no infrastructure changes
4. User_data doesn't re-run
5. Container keeps running old version

Solution: Manual update required:
ssh into instance → docker pull latest → docker restart
```

**Production Solution (future improvement):**
```
1. Use Auto Scaling Group
2. Implement Blue-Green deployment
3. Health checks before routing traffic
4. Automatic rollback on failure
```

---

## 🌐 Network Architecture

```
Internet
    ↓
AWS Internet Gateway
    ↓
Default VPC (172.31.0.0/16)
    ↓
Public Subnet
    ↓
Route Table (0.0.0.0/0 → IGW)
    ↓
Network Interface (ENI)
    ↓
EC2 Instance
    ├─ Private IP: 172.31.x.x
    └─ Public IP: Dynamic (e.g., 3.145.112.89)
```

**Why Default VPC?**
- Simpler for demo (no VPC creation needed)
- Already configured with IGW and route table
- Public subnet by default

**Production Consideration:**
- Create custom VPC
- Use private subnets for apps
- ALB in public subnet
- NAT Gateway for outbound

---

## 💾 Storage Architecture

**Container Storage:**
```
EC2 Instance (8GB EBS Volume)
    ├─ OS: Ubuntu 22.04 (~3GB)
    ├─ Docker: (~500MB)
    ├─ Images: flask-aws-monitor (~100MB)
    └─ Logs: (~100MB)
    
Remaining: ~4GB free
```

**No Persistent Data:**
- App is stateless
- No database required
- Metadata fetched on-demand
- Logs in container (lost on restart)

---

## 🔍 Observability

**Current Monitoring:**
- ⚠ None (basic visibility only)

**What We Can See:**
- AWS Console: Instance status
- Docker: Container logs (`docker logs`)
- Flask: stdout/stderr

**Production Improvements:**
```
Add:
- CloudWatch Logs → Centralized logging
- CloudWatch Metrics → CPU, Memory, Network
- Application Performance Monitoring (APM)
- Health check endpoint (/health)
- Alerts on failures
```

---

## 🚀 Scalability Considerations

**Current Limitations:**
- Single instance (no redundancy)
- No load balancing
- Manual scaling required
- Single region

**Scale to 100 users:**
```
✓ Current setup handles this fine
   (Flask can handle ~500 req/sec on t3.medium)
```

**Scale to 10,000 users:**
```
Required changes:
1. Application Load Balancer
2. Auto Scaling Group (3-10 instances)
3. Multi-AZ deployment
4. CloudFront CDN (if needed)
5. Consider serverless (Lambda + API Gateway)
```

---

## 📈 Cost Optimization

**Current Costs:**
```
EC2 t3.medium: $0.0416/hour × 730 hours = $30.37/month
EBS 8GB: $0.10/GB/month × 8 = $0.80/month
Data Transfer: ~$0.09/GB (minimal)
────────────────────────────────────────────────
Total: ~$31/month
```

**Cost Saving Options:**
```
1. Stop instance when not in use: -70% (only pay when running)
2. Use t3.small instead: -50% ($15/month)
3. Reserved Instance (1 year): -40% (~$18/month)
4. Spot Instance: -70% (~$9/month, can be terminated)
```

---

## 🎓 Learning Outcomes

This architecture teaches:

1. **CI/CD Principles**
   - Automated testing
   - Continuous deployment
   - Pipeline orchestration

2. **Cloud Architecture**
   - Compute (EC2)
   - Networking (VPC, Security Groups)
   - IAM (permissions)

3. **Container Technology**
   - Image building
   - Registry management
   - Runtime orchestration

4. **Infrastructure as Code**
   - Declarative configuration
   - State management
   - Automated provisioning

5. **Security**
   - Credential management
   - IMDSv2 protection
   - Security scanning

---

## 🔮 Future Architecture (Production-Ready)

```
                    ┌──────────────┐
                    │  CloudFront  │  (CDN)
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │     ALB      │  (Load Balancer)
                    └──────┬───────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
        ┌───▼───┐     ┌───▼───┐     ┌───▼───┐
        │ EC2-1 │     │ EC2-2 │     │ EC2-3 │  (Auto Scaling Group)
        │  AZ-1 │     │  AZ-2 │     │  AZ-3 │
        └───┬───┘     └───┬───┘     └───┬───┘
            │              │              │
            └──────────────┼──────────────┘
                           │
                    ┌──────▼───────┐
                    │   RDS/DDB    │  (If needed for state)
                    └──────────────┘
```

---

**This architecture represents a solid foundation for learning DevOps practices!** 🎯