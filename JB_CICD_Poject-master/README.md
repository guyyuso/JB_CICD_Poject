# Flask AWS Monitor - Jenkins CI/CD Project

## 📋 Project Overview

This project demonstrates a complete **CI/CD pipeline** that automatically deploys a Flask web application to AWS EC2 using:
- **Jenkins** for continuous integration and deployment
- **Docker** for containerization
- **Terraform** for infrastructure as code
- **AWS EC2** for hosting

### What It Does
When code is pushed to the repository, Jenkins automatically:
1. Runs code quality checks (linting & security scanning)
2. Builds a Docker image
3. Pushes the image to Docker Hub
4. Provisions AWS infrastructure using Terraform
5. Deploys the containerized Flask app to EC2

The Flask app displays a dashboard showing:
- EC2 Public IP Address
- SSH Key Information
- Security Group ID
- Instance Type
- AWS Region
- VPC ID

---

## 🏗️ Architecture

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│   GitHub    │────▶│   Jenkins    │─────▶│  Docker Hub │
│ Repository  │      │   Pipeline   │      │   Registry  │
└─────────────┘      └──────────────┘      └─────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │  Terraform   │
                     │  (AWS IaC)   │
                     └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   AWS EC2    │
                     │ Ubuntu 22.04 │
                     │   + Docker   │
                     └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │ Flask App    │
                     │ Port 5001    │
                     └──────────────┘
```

---

## 🚀 Quick Start for Professor/Reviewer

### Prerequisites

Before running this project, ensure you have:

1. **Jenkins Server** with the following installed:
   - Docker
   - Terraform (v1.5.0+)
   - Git
   - Python 3.9+

2. **AWS Account** with:
   - Programmatic access (Access Key ID & Secret Access Key)
   - Permissions to create EC2, Security Groups, Key Pairs

3. **Docker Hub Account** (free tier is sufficient)

4. **Git Repository** access to this code

---

## 🔧 Setup Instructions

### Step 1: Configure Jenkins Credentials

In your Jenkins dashboard, add the following credentials:

#### a) Docker Hub Credentials
```
1. Navigate to: Jenkins → Manage Jenkins → Credentials → System → Global credentials
2. Click "Add Credentials"
   - Kind: Username with password
   - Scope: Global
   - Username: <your-docker-hub-username>
   - Password: <your-docker-hub-password>
   - ID: dockerhub-credentials
   - Description: Docker Hub Access
3. Click "Save"
```

#### b) AWS Access Key ID
```
1. Add new credential:
   - Kind: Secret text
   - Scope: Global
   - Secret: <your-aws-access-key-id>
   - ID: aws-access-key-id
   - Description: AWS Access Key
2. Click "Save"
```

#### c) AWS Secret Access Key
```
1. Add new credential:
   - Kind: Secret text
   - Scope: Global
   - Secret: <your-aws-secret-access-key>
   - ID: aws-secret-access-key
   - Description: AWS Secret Key
2. Click "Save"
```

### Step 2: Configure Project Variables

Edit `terraform/variables.tf` and update:

```hcl
variable "docker_username" {
  description = "Your Docker Hub username"
  type        = string
  default     = "guyyusupov"  # ← Change this to YOUR username
}

variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-2"  # ← Change if you prefer a different region
}
```

***Important***
-  **If you're reviewing this project** you can leave `docker_username` as `"guyyusupov"` to use the author's pre-built Docker image.
- **Build with Parameters** field in Jenkins takes priority over the variables.tf default value.

### Step 3: Create Jenkins Pipeline Job

```
1. In Jenkins Dashboard, click "New Item"
2. Enter job name: "Flask-AWS-Monitor-Pipeline"
3. Select: "Pipeline"
4. Click "OK"
5. In the Pipeline section:
   - Definition: Pipeline script from SCM
   - SCM: Git
   - Repository URL: <your-git-repo-url>
   - Script Path: Jenkinsfile
6. Click "Save"
```

### Step 4: Run the Pipeline

```
1. Click "Build with Parameters"
2. Click "Build"
```

### Step 5: Access the Dashboard

After the pipeline completes (approximately 3-5 minutes):

1. Check Jenkins console output for the dashboard URL
2. Look for the output that says:
   ```
   web_dashboard_url = "http://<PUBLIC_IP>:5001"
   ```
3. Open this URL in your browser

**Example**: `http://3.145.112.89:5001`

---

## 📁 Project Structure

```
.
├── app/
│   ├── app.py              # Flask application code
│   ├── Dockerfile          # Container build instructions
│   └── requirements.txt    # Python dependencies
├── terraform/
│   ├── main.tf            # Main infrastructure definitions
│   ├── providers.tf       # Terraform provider configuration
│   ├── variables.tf       # Input variables
│   ├── outputs.tf         # Output values after deployment
│   └── user_data.sh       # EC2 initialization script
├── Jenkinsfile            # CI/CD pipeline definition
├── .gitignore            # Git ignore rules (excludes keys!)
└── README.md             # This file
```

---

## 🔐 Security Features

### 1. IMDSv2 (Instance Metadata Service v2)
The application uses AWS IMDSv2 to securely fetch instance metadata, preventing SSRF attacks:

```python
# Step 1: Get session token
token = requests.put("http://169.254.169.254/latest/api/token",
                     headers={"X-aws-ec2-metadata-token-ttl-seconds": "21600"})

# Step 2: Use token to access metadata
public_ip = requests.get("http://169.254.169.254/latest/meta-data/public-ipv4",
                         headers={"X-aws-ec2-metadata-token": token})
```

### 2. No Secrets in Code
- All credentials stored in Jenkins Credentials Store
- SSH keys excluded via `.gitignore`
- AWS credentials passed as environment variables

### 3. Security Scanning
- **Bandit**: Scans Python code for security vulnerabilities
- **Flake8**: Ensures code quality and identifies potential issues

---

## 🛠️ Pipeline Stages Explained

### Stage 1: Clone Repository
```groovy
checkout scm
```
Downloads the latest code from the Git repository.

### Stage 2: Parallel Quality Checks
```groovy
parallel {
    stage('Linting') { ... }      // Code style checks
    stage('Security Scan') { ... } // Vulnerability scanning
}
```
Runs two checks simultaneously for faster execution.

### Stage 3: Build Docker Image
```groovy
docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} app/
```
Creates a Docker container image with the Flask application.

### Stage 4: Push to Docker Hub
```groovy
docker push ${IMAGE_NAME}:latest
```
Uploads the image to Docker Hub registry for later deployment.

### Stage 5: Deploy Infrastructure
```groovy
terraform apply -auto-approve
```
Provisions AWS resources:
- EC2 Instance (Ubuntu 22.04, t3.medium)
- Security Group (SSH port 22, Flask port 5001)
- SSH Key Pair (4096-bit RSA)

---

## 🧪 Testing the Application

### Manual Testing Checklist

1. **Access Dashboard**: Open `http://<PUBLIC_IP>:5001`
2. **Verify Data Display**:
   - ✅ Public IP matches EC2 instance IP
   - ✅ Instance Type shows "t3.medium"
   - ✅ Region matches your terraform variable
   - ✅ Security Group ID is displayed
   - ✅ VPC ID is displayed

3. **SSH Access** (Optional):
   ```bash
   # The private key is saved locally by Terraform
   chmod 400 terraform/builder_key.pem
   ssh -i terraform/builder_key.pem ubuntu@<PUBLIC_IP>
   
   # Once connected, verify Docker container is running:
   docker ps
   ```

### Expected Output
You should see a beautifully styled dashboard with:
- Purple gradient background
- White card container
- AWS cloud emoji ☁️
- Green "✅ Running in Docker" status badge
- All metadata displayed in organized cards

---

## 🐛 Troubleshooting

### Problem: "Metadata Service Unavailable"

**Cause**: IMDSv2 token request failed

**Solution**:
1. SSH into EC2 instance
2. Check if Docker container is running: `docker ps`
3. View container logs: `docker logs flask-dashboard`
4. Verify IMDSv2 is enabled in EC2 instance settings

---

### Problem: Jenkins Build Fails at "Push to Docker Hub"

**Cause**: Docker Hub credentials not configured correctly

**Solution**:
1. Verify credential ID is exactly: `dockerhub-credentials`
2. Test Docker Hub login manually:
   ```bash
   docker login -u <username>
   ```
3. Re-add the credential in Jenkins

---

### Problem: Terraform Apply Fails with "Invalid AMI ID"

**Cause**: AMI ID is region-specific

**Solution**:
The default AMI `ami-09040d770ff222417` is for `us-east-2`. If using a different region:
1. Find Ubuntu 22.04 AMI for your region: https://cloud-images.ubuntu.com/locator/ec2/
2. Update `terraform/variables.tf`:
   ```hcl
   variable "ami_id" {
     default = "ami-xxxxxxxxx"  # Your region's AMI
   }
   ```

---

### Problem: Cannot Access Dashboard on Port 5001

**Cause**: Security Group not allowing traffic

**Solution**:
1. Check AWS Console → EC2 → Security Groups
2. Verify `jenkins-project-sg` has inbound rule:
   - Type: Custom TCP
   - Port: 5001
   - Source: 0.0.0.0/0

---

### Problem: Docker Container Exits Immediately

**Cause**: Application error or port conflict

**Solution**:
SSH into instance and check:
```bash
docker logs flask-dashboard
```
Look for Python errors or port binding issues.

---

## 💰 Cost Estimate

Running this project costs approximately:

| Resource | Cost (per hour) | Cost (per month)* |
|----------|----------------|-------------------|
| EC2 t3.medium | $0.0416 | ~$30 |
| Data Transfer | Minimal | <$1 |
| **Total** | **$0.04** | **~$31** |

*Based on running 24/7. For testing, stop instance when not in use.

**Cost Saving Tip**:
```bash
# Stop instance to avoid charges:
aws ec2 stop-instances --instance-ids <instance-id>

# Start again when needed:
aws ec2 start-instances --instance-ids <instance-id>
```

---

## 🧹 Cleanup

To destroy all resources and avoid charges:

```bash
cd terraform
terraform destroy -auto-approve
```

This will delete:
- EC2 Instance
- Security Group
- SSH Key Pair
- All associated resources

**Important**: Docker images remain in Docker Hub and must be deleted manually if desired.

---

## 📝 Technologies Used

| Technology | Version | Purpose |
|-----------|---------|---------|
| Python | 3.9 | Application runtime |
| Flask | Latest | Web framework |
| Docker | Latest | Containerization |
| Terraform | ≥1.5.0 | Infrastructure as Code |
| Jenkins | Latest | CI/CD automation |
| AWS EC2 | Ubuntu 22.04 | Hosting platform |
| boto3 | Latest | AWS SDK for Python |

---

## 👨‍🎓 Learning Outcomes

This project demonstrates proficiency in:

1. **Continuous Integration/Continuous Deployment (CI/CD)**
   - Automated build and deployment pipeline
   - Parallel execution of quality checks
   - Automated testing and deployment

2. **Infrastructure as Code (IaC)**
   - Terraform resource management
   - Dynamic infrastructure provisioning
   - Output management for dependent resources

3. **Containerization**
   - Docker image creation
   - Multi-stage container deployment
   - Container registry management

4. **Cloud Computing (AWS)**
   - EC2 instance management
   - Security group configuration
   - Metadata service usage (IMDSv2)
   - SSH key management

5. **Security Best Practices**
   - Credential management
   - Secrets handling
   - Security scanning integration
   - Secure metadata access

---

## 📞 Support

If you encounter issues:

1. Check the **Troubleshooting** section above
2. Review Jenkins console output for error messages
3. Check AWS CloudWatch logs for EC2 instance issues
4. Verify all credentials are correctly configured

---

## 📄 License

This is an educational project created for academic purposes.

---

## 👤 Author

**Student**: Guy Yusupov  
**Docker Hub**: guyyusupov  
**Project**: Flask AWS Monitor with Jenkins CI/CD

---

**Last Updated**: February 2026