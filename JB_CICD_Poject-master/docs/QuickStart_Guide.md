# Quick Start Guide for Professor

**Goal**: Get this project running in under 15 minutes.

---

## ⚡ Prerequisites Checklist

```
☐ Jenkins server running
☐ Docker installed on Jenkins server
☐ Terraform v1.5.0+ installed
☐ AWS account with access keys
☐ Docker Hub account
```

If missing any, see detailed **JENKINS_SETUP.md** for installation steps.

---

## 🚀 5-Minute Setup

### Step 1: Add Jenkins Credentials (3 minutes)

Navigate to: **Jenkins → Manage Jenkins → Credentials → System → Global credentials**

**Add these 3 credentials:**

| Kind | ID | Secret Value |
|------|----|----|
| Username/Password | `dockerhub-credentials` | Docker Hub username + password |
| Secret text | `aws-access-key-id` | Your AWS access key |
| Secret text | `aws-secret-access-key` | Your AWS secret key |

⚠️ **CRITICAL**: The IDs must match EXACTLY as shown above.

---

### Step 2: Create Pipeline Job (2 minutes)

```
1. Jenkins Dashboard → "New Item"
2. Name: Flask-AWS-Monitor
3. Type: "Pipeline"
4. Pipeline section:
   - Definition: "Pipeline script from SCM"
   - SCM: Git
   - Repository URL: <student's-repo-url>
   - Script Path: Jenkinsfile
5. Save
```

---

### Step 3: Run Pipeline (1 click)

```
1. Click "Build with Parameters"
2. DOCKER_USER: guyyusupov  (← Leave this unchanged to use student's image)
3. Click "Build"
```

**Wait ~5 minutes** while pipeline executes.

---

### Step 4: Access Dashboard

Look for this in console output:
```
web_dashboard_url = "http://3.145.112.89:5001"
                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Open this URL in your browser. Done! ✅

---

## 🎯 What You Should See

### Jenkins Pipeline (5 stages, all green):
```
✓ Clone Repository          (~20 sec)
✓ Parallel Quality Checks   (~45 sec)
✓ Build Docker Image        (~90 sec)
✓ Push to Docker Hub        (~60 sec)
✓ Deploy Infrastructure     (~180 sec)
```

### Flask Dashboard:
- Purple gradient background
- White card with AWS logo ☁️
- Green "Running in Docker" badge
- 6 information cards showing:
  - Public IP
  - SSH Key Path
  - Security Group ID
  - Instance Type (t3.medium)
  - Region (us-east-2)
  - VPC ID

---

## ❗ Common Issues (Quick Fixes)

### "Credential not found"
→ Check credential ID matches exactly: `dockerhub-credentials`

### "terraform: command not found"
```bash
# Install Terraform on Jenkins server:
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

### "UnauthorizedOperation" from AWS
→ Add `AmazonEC2FullAccess` policy to your AWS IAM user

### "Connection refused" on port 5001
→ Wait 1-2 minutes for Docker container to start, then refresh

### Dashboard shows "Metadata Service Unavailable"
→ This is OK on first load. Wait 30 seconds and refresh.

---

## 🧹 Cleanup After Testing

**To avoid AWS charges:**

```bash
# Option 1: Via Jenkins
cd /var/lib/jenkins/workspace/Flask-AWS-Monitor/terraform
terraform destroy -auto-approve

# Option 2: Via AWS Console
Go to EC2 → Instances → Select "JBP-Builder-Instance" → Terminate
```

Cost if you forget: ~$1/day

---

## 📋 Evaluation Checklist

**Infrastructure (Terraform):**
```
☐ Creates EC2 instance successfully
☐ Generates SSH key pair
☐ Configures security group correctly
☐ Uses variables properly
☐ Implements IMDSv2 security
```

**CI/CD Pipeline (Jenkins):**
```
☐ All 5 stages complete successfully
☐ Parallel execution works
☐ Credentials managed securely
☐ No secrets in code
☐ Proper error handling
```

**Application (Flask):**
```
☐ Dashboard accessible from internet
☐ Displays correct metadata
☐ Professional styling
☐ Uses IMDSv2 for metadata
☐ No errors on page load
```

**Code Quality:**
```
☐ Clean code structure
☐ Proper documentation
☐ Security scanning integrated
☐ .gitignore excludes sensitive files
```

---

## 🎓 Student Demonstrated

This project shows competency in:

✅ **DevOps Tools**: Jenkins, Docker, Terraform  
✅ **Cloud Computing**: AWS EC2, Security Groups, IAM  
✅ **Automation**: Full CI/CD pipeline  
✅ **Security**: IMDSv2, credential management, scanning  
✅ **Best Practices**: IaC, containerization, version control  

---

## 💡 Questions to Ask

1. "Walk me through what happens when you push code"
   - Expected: Explains full CI/CD flow

2. "What's the difference between IMDSv1 and IMDSv2?"
   - Expected: Security benefits, SSRF protection

3. "Why use Docker?"
   - Expected: Portability, consistency, ease of deployment

4. "What would you change for production?"
   - Expected: Load balancer, auto-scaling, monitoring

5. "Where are your AWS credentials stored?"
   - Expected: Jenkins credentials store, never in code

---

## 📞 Need Help?

See detailed guides:
- **JENKINS_SETUP.md** - Complete Jenkins configuration
- **AWS_PERMISSIONS.md** - IAM setup and permissions
- **TROUBLESHOOTING.md** - Common errors and solutions
- **ARCHITECTURE.md** - Technical deep dive

---

**Total Time to Working Dashboard: ~10 minutes** ⚡

**Expected Build Time: ~5 minutes** ⏱️

