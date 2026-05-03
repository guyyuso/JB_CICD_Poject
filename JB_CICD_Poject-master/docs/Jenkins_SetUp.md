# Jenkins Setup Guide for Reviewers

This guide is specifically for **professors/reviewers** who want to run this student project on their own Jenkins instance.

---

## 🎯 Overview

You will need to:
1. Configure 3 credentials in Jenkins
2. Create a Pipeline job
3. Adjust one parameter (Docker username)
4. Run the pipeline

**Total setup time**: ~10 minutes  
**Total build time**: ~5 minutes

---

## 📋 Prerequisites Check

Before starting, verify your Jenkins server has:

```bash
# Check Docker
docker --version
# Expected: Docker version 20.x or higher

# Check Terraform
terraform --version
# Expected: Terraform v1.5.0 or higher

# Check AWS CLI (optional, for verification)
aws --version
# Expected: aws-cli/2.x or higher

# Check Python
python3 --version
# Expected: Python 3.9 or higher
```

If any are missing, install them on your Jenkins server.

---

## 🔐 Step 1: Configure Jenkins Credentials

### Why These Are Needed:
- **Docker Hub**: To pull the student's pre-built Flask image
- **AWS**: To provision EC2 infrastructure in your account

### How to Add Credentials:

#### Navigate to Credentials Page
```
Jenkins Dashboard 
  → Manage Jenkins 
  → Manage Credentials 
  → System 
  → Global credentials (unrestricted)
  → Add Credentials
```

---

### Credential 1: Docker Hub Login

**Purpose**: Allow Jenkins to pull/push images from Docker Hub

```
Click "Add Credentials"

Field Values:
- Kind: Username with password
- Scope: Global (Jenkins, nodes, items, all child items, etc)
- Username: <your-docker-hub-username>
- Password: <your-docker-hub-password>
- ID: dockerhub-credentials
       ^^^^^^^^^^^^^^^^^^^^^ MUST BE EXACTLY THIS
- Description: Docker Hub Access for Student Project

Click "Create"
```

**Important**: The ID **must** be `dockerhub-credentials` because the Jenkinsfile references this exact ID.

---

### Credential 2: AWS Access Key ID

**Purpose**: Authenticate with AWS for Terraform

```
Click "Add Credentials"

Field Values:
- Kind: Secret text
- Scope: Global
- Secret: <paste-your-aws-access-key-id-here>
         Example: AKIAIOSFODNN7EXAMPLE
- ID: aws-access-key-id
      ^^^^^^^^^^^^^^^^^^^ MUST BE EXACTLY THIS
- Description: AWS Access Key for Terraform

Click "Create"
```

---

### Credential 3: AWS Secret Access Key

**Purpose**: Complete AWS authentication

```
Click "Add Credentials"

Field Values:
- Kind: Secret text
- Scope: Global
- Secret: <paste-your-aws-secret-access-key-here>
         Example: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
- ID: aws-secret-access-key
      ^^^^^^^^^^^^^^^^^^^^^^^^ MUST BE EXACTLY THIS
- Description: AWS Secret Key for Terraform

Click "Create"
```

---

### Verify All Credentials Are Added

Your credentials page should now show:

```
Name                        Kind             ID
─────────────────────────────────────────────────────────────
Docker Hub Access...        Username/Pass    dockerhub-credentials
AWS Access Key...           Secret text      aws-access-key-id
AWS Secret Key...           Secret text      aws-secret-access-key
```

---

## 🏗️ Step 2: Create Jenkins Pipeline Job

### Create New Item
```
1. From Jenkins Dashboard, click "New Item" (top left)

2. Enter item name: Student-Flask-AWS-Project
   (you can name it anything you like)

3. Select: "Pipeline"

4. Click "OK"
```

---

### Configure the Pipeline

In the configuration page:

#### General Section
```
☐ Discard old builds (optional, but recommended)
  - Strategy: Log Rotation
  - Max # of builds to keep: 5
```

#### Build Triggers Section
```
☐ Skip this - we'll trigger manually
```

#### Advanced Project Options
```
☐ Skip this
```

#### Pipeline Section

**This is the important part!**

```
Definition: Pipeline script from SCM

SCM: Git

Repositories:
  Repository URL: <git-url-provided-by-student>
  Example: https://github.com/guyyusupov/flask-aws-monitor.git
  
  Credentials: - none -
  (unless the repo is private)

Branches to build:
  Branch Specifier: */main
  (or */master, depending on repo)

Script Path: Jenkinsfile
             ^^^^^^^^^^ This tells Jenkins where to find the pipeline code
```

Click "Save"

---

## ▶️ Step 3: Run the Pipeline

### First Build
```
1. From the project page, click "Build with Parameters"
   (if you don't see this, click "Build Now" instead)

2. You'll see a parameter field:
   
   DOCKER_USER: [guyyusupov    ]
                  ^^^^^^^^^^^
   This is the student's Docker Hub username
   
   IMPORTANT: Leave this as "guyyusupov" to use the student's 
   pre-built image. This way you don't need to build the image yourself.

3. Click "Build"
```

---

## 📊 Step 4: Monitor the Build

### Console Output
```
1. Click on the build number (e.g., "#1") in Build History

2. Click "Console Output"

3. Watch the stages execute:
   ✓ Clone Repository
   ✓ Parallel Quality Checks
     ├─ Linting
     └─ Security Scan
   ✓ Build Docker Image
   ✓ Push to Docker Hub
   ✓ Deploy Infrastructure (Terraform)
```

### Expected Timeline
```
Stage                          Duration
─────────────────────────────────────────
Clone Repository              10-20 sec
Parallel Quality Checks       30-45 sec
Build Docker Image            1-2 min
Push to Docker Hub            30-60 sec
Deploy Infrastructure         2-3 min
─────────────────────────────────────────
Total                         ~5 minutes
```

---

## 🎯 Step 5: Access the Dashboard

### Get the URL

At the end of the build, look for this output:

```
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

instance_id = "i-0abcd1234efgh5678"
instance_name = "JBP-Builder-Instance"
instance_public_ip = "3.145.112.89"
private_key_path = "./builder_key.pem"
security_group_id = "sg-0123456789abcdef"
web_dashboard_url = "http://3.145.112.89:5001"
                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
                     THIS IS YOUR DASHBOARD URL
```

### Open the Dashboard

1. Copy the `web_dashboard_url`
2. Paste it in your browser
3. You should see a beautiful purple-gradient dashboard showing:
   - Public IP Address
   - SSH Key Location
   - Security Group ID
   - Instance Type (t3.medium)
   - AWS Region (us-east-2)
   - VPC ID

---

## ✅ Verification Checklist

Use this to verify the project is working correctly:

```
Infrastructure Verification:
☐ EC2 instance is running in AWS Console
☐ Instance has public IP assigned
☐ Security group allows ports 22 and 5001
☐ Docker container is running on the instance

Dashboard Verification:
☐ Dashboard URL is accessible from browser
☐ Dashboard displays correct public IP
☐ All metadata fields are populated
☐ No "Metadata Service Unavailable" errors
☐ Dashboard has professional styling

Pipeline Verification:
☐ All 5 stages completed successfully
☐ No build errors in console output
☐ Docker image pushed to Docker Hub
☐ Terraform outputs displayed correctly
```

---

## 🐛 Common Issues & Solutions

### Issue 1: "Credential not found: dockerhub-credentials"

**Cause**: Credential ID doesn't match exactly

**Solution**:
1. Go to Jenkins → Manage Jenkins → Credentials
2. Click on "dockerhub-credentials" (or whatever you named it)
3. Click "Update"
4. Ensure ID field shows: `dockerhub-credentials`
5. Click "Save"

---

### Issue 2: "Error: UnauthorizedOperation - Not authorized to perform..."

**Cause**: AWS credentials lack necessary permissions

**Solution**:
Your AWS user needs these IAM permissions:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:RunInstances",
      "ec2:TerminateInstances",
      "ec2:DescribeInstances",
      "ec2:CreateSecurityGroup",
      "ec2:AuthorizeSecurityGroupIngress",
      "ec2:CreateKeyPair",
      "ec2:DescribeKeyPairs",
      "ec2:DescribeSecurityGroups",
      "ec2:CreateTags"
    ],
    "Resource": "*"
  }]
}
```

---

### Issue 3: "Error acquiring the state lock"

**Cause**: Another Terraform process is running

**Solution**:
Wait 5 minutes and try again, OR:
```bash
# SSH into Jenkins server
cd /var/lib/jenkins/workspace/Student-Flask-AWS-Project/terraform
terraform force-unlock <LOCK_ID>
# Lock ID will be shown in the error message
```

---

### Issue 4: Dashboard shows "Metadata Service Unavailable"

**Cause**: Container can't reach AWS metadata service

**Solution**:
1. SSH into EC2 instance:
   ```bash
   # Get private key from Terraform output directory
   chmod 400 ./builder_key.pem
   ssh -i ./builder_key.pem ubuntu@<public-ip>
   ```

2. Check Docker container:
   ```bash
   docker ps
   docker logs flask-dashboard
   ```

3. Verify IMDSv2 is working:
   ```bash
   TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
   curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
   ```

---

### Issue 5: "Cannot connect to the Docker daemon"

**Cause**: Docker not running on Jenkins server

**Solution**:
```bash
# SSH into Jenkins server
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## 🧹 Cleanup After Testing

To avoid AWS charges after reviewing:

### Option 1: Destroy via Jenkins

```
1. In Jenkins, go to the project
2. Click "Configure"
3. Add a new stage to Jenkinsfile temporarily:
   
   stage('Cleanup') {
       steps {
           script {
               dir('terraform') {
                   sh 'terraform destroy -auto-approve'
               }
           }
       }
   }
4. Save and run pipeline
```

### Option 2: Manual Cleanup

```bash
# SSH into Jenkins server
cd /var/lib/jenkins/workspace/<job-name>/terraform
terraform destroy -auto-approve

# Or from your local machine if you have Terraform:
cd terraform
terraform destroy -auto-approve
```

### Option 3: AWS Console

```
1. Go to AWS Console → EC2
2. Select the instance (Name: "JBP-Builder-Instance")
3. Actions → Instance State → Terminate
4. Also delete:
   - Security Group: jenkins-project-sg
   - Key Pair: builder_key
```

---

## 📊 Grading Rubric Suggestions

For professors evaluating this project:

### Infrastructure as Code (25 points)
- ✓ Terraform configuration is well-structured
- ✓ Resources are properly defined
- ✓ Variables and outputs are used correctly
- ✓ IMDSv2 security feature implemented

### CI/CD Pipeline (25 points)
- ✓ Jenkinsfile follows best practices
- ✓ Parallel stages for efficiency
- ✓ Proper credential management
- ✓ Automated deployment works end-to-end

### Containerization (20 points)
- ✓ Dockerfile is optimized (layer caching)
- ✓ Application runs successfully in container
- ✓ Docker image tagged properly
- ✓ Registry push/pull works correctly

### Application Code (15 points)
- ✓ Flask app is functional
- ✓ Metadata fetching works (IMDSv2)
- ✓ Dashboard displays correct information
- ✓ Code is clean and readable

### Security (10 points)
- ✓ No hardcoded secrets
- ✓ .gitignore excludes sensitive files
- ✓ Security scanning integrated
- ✓ IMDSv2 implemented correctly

### Documentation (5 points)
- ✓ README is comprehensive
- ✓ Setup instructions are clear
- ✓ Troubleshooting guide included

---

## 💡 What Makes This Project Good?

1. **Complete CI/CD Pipeline**: Not just code, but automated deployment
2. **Modern Tools**: Docker, Terraform, Jenkins - industry standard
3. **Security Awareness**: IMDSv2, credential management, security scanning
4. **Infrastructure as Code**: Reproducible infrastructure
5. **Clean Code**: Well-organized, properly documented
6. **Real-World Applicable**: This pattern is used in production environments

---

## 📞 Need Help?

If you encounter issues not covered in this guide:

1. Check Jenkins console output for detailed error messages
2. Review AWS CloudWatch logs for EC2 issues
3. Verify all credential IDs match exactly
4. Ensure your AWS account has sufficient permissions
5. Check that Jenkins plugins are up to date:
   - Docker Pipeline
   - AWS Credentials Plugin
   - Git Plugin

---

## 🎓 Questions to Ask the Student

During the review, consider asking:

1. "Why did you choose IMDSv2 over IMDSv1?"
   - Expected: Security benefits, prevents SSRF attacks

2. "What happens if the Docker image build fails?"
   - Expected: Pipeline stops, no infrastructure is deployed

3. "How would you scale this to handle 1000 concurrent users?"
   - Expected: Add load balancer, auto-scaling group, multiple instances

4. "Where are your secrets stored and how are they managed?"
   - Expected: Jenkins credentials store, never in code/git

5. "What AWS permissions does Terraform need?"
   - Expected: EC2, VPC, Security Group management

---

**Good luck with your review!** 🚀

This is a solid DevOps project demonstrating competency in modern deployment practices.