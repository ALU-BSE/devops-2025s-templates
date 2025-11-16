# Summative Project: Complete DevOps Pipeline

## Table of Contents
1. [Overview](#overview)
2. [Learning Objectives](#learning-objectives)
3. [Infrastructure Components Explained](#infrastructure-components-explained)
4. [Core Requirements](#core-requirements)
5. [Pipeline Architecture](#pipeline-architecture)
6. [Technical Specifications](#technical-specifications)
7. [AI & Academic Integrity](#ai--academic-integrity)
8. [Deliverables](#deliverables)
9. [Common Pitfalls](#common-pitfalls)
10. [Submission Guidelines](#submission-guidelines)
11. [Success Checklist](#success-checklist)

---

## Overview

This is your final, comprehensive project where you will build, secure, and deploy a complete end-to-end DevOps pipeline. You will integrate your work from **Formative 1** (Project Planning) and **Formative 2** (Containerization & CI) to demonstrate a fully automated **"Git-to-Production"** workflow.

**What this means:** A code change merged to the `main` branch should automatically and safely deploy to your live cloud environment without any manual intervention.

### What "Working" Looks Like
By the end, you should be able to:
1. Run `terraform apply` and see all infrastructure created in your cloud account
2. Make a code change, create a PR, and see CI run security scans automatically
3. Merge the PR and watch CD automatically deploy to production
4. Visit your public URL and see the change live within 2-3 minutes
5. Run `terraform destroy` and cleanly remove all resources

**You must test this complete flow at least twice before submitting.**

---

## Learning Objectives

By completing this project, you will demonstrate your ability to:
- Design and provision cloud infrastructure using Infrastructure as Code (Terraform)
- Implement secure network architecture with public and private subnets
- Automate server configuration using Configuration Management (Ansible)
- Integrate security scanning into CI/CD pipelines (DevSecOps)
- Build fully automated deployment pipelines using GitHub Actions
- Document and present complex technical systems clearly

---

## Infrastructure Components Explained

Your cloud infrastructure will include six key components:

| Component | Purpose | Example |
|-----------|---------|---------|
| **VPC/VNet** | Isolated network for your resources | Your private cloud network |
| **Private Subnet VM** | Hosts your application, not directly internet-accessible | Application server (t2.micro/B1s) |
| **Bastion Host** | Secure gateway for SSH access to private resources | Jump server in public subnet |
| **Managed Database** | Persistent data storage with automatic backups | RDS PostgreSQL, Azure MySQL |
| **Security Groups/NSGs** | Firewall rules controlling inbound/outbound traffic | Allow port 80/443, SSH from bastion only |
| **Private Registry** | Secure storage for your Docker images | AWS ECR or Azure ACR |

**Security Note:** Your application VM should NEVER be directly accessible from the internet. All SSH access must go through the Bastion Host.

---

## Core Requirements

### 1. Infrastructure as Code (IaC)

**Tool:** Terraform  
**Location:** `terraform/` directory in your repository

**Required Resources:**
- Private network (AWS VPC or Azure VNet)
- Public subnet (for Bastion Host)
- Private subnet (for Application VM and Database)
- Bastion Host/Jumpbox (t2.micro/B1s in public subnet)
- Application VM (t2.micro/B1s in private subnet)
- Managed database (RDS/Azure Database)
- Security Groups/NSGs with proper rules:
  - Allow HTTP/HTTPS from internet to Application VM
  - Allow SSH from Bastion to Application VM only
  - Allow database connections from Application VM only
  - Allow SSH to Bastion from your IP (or anywhere temporarily)
- Private container registry (AWS ECR or Azure ACR)

**Code Structure Requirements:**
```
terraform/
├── main.tf           # Main configuration
├── variables.tf      # Input variables
├── outputs.tf        # Output values (VM IPs, registry URL)
├── terraform.tfvars  # Variable values (git-ignored if contains secrets)
└── README.md         # Terraform-specific documentation
```

**Best Practices:**
- Use variables for all configurable values (region, VM size, etc.)
- Use meaningful resource names (e.g., `devops-app-vm`, not `vm1`)
- Add comments explaining non-obvious configurations
- Use `terraform fmt` to format your code
- Output important values (Bastion IP, App VM private IP, Registry URL)

---

### 2. Configuration Management (Ansible)

**Tool:** Ansible  
**Location:** `ansible/` directory in your repository

**Required Playbook Tasks:**
Your playbook must configure the Application VM to:
1. Install Docker and Docker Compose
2. Install required system packages (git, curl, etc.)
3. **Authenticate to your private container registry** (ECR/ACR)
4. **Pull the latest Docker image** from your registry
5. **Deploy/restart the application** using `docker-compose up -d`

**Playbook Structure:**
```yaml
# ansible/deploy.yml
---
- name: Deploy Application to Production
  hosts: app_server
  become: yes
  
  tasks:
    - name: Install Docker
      # your tasks here
    
    - name: Login to private registry
      # your tasks here
    
    - name: Pull latest image
      # your tasks here
    
    - name: Deploy with docker-compose
      # your tasks here
```

**Important Configuration:**
- Your Ansible inventory must use the Bastion Host as a jump/proxy host
- Use `ansible_ssh_common_args: '-o ProxyCommand="ssh -W %h:%p -q ubuntu@<bastion-ip>"'`
- Test your playbook multiple times before integrating into CD pipeline

**Example Inventory:**
```ini
# ansible/inventory.ini
[app_server]
app-vm ansible_host=<private-ip>

[app_server:vars]
ansible_user=ubuntu
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -q ubuntu@<bastion-ip>"'
```

---

### 3. DevSecOps Integration

**Goal:** Shift security left by scanning early in the development process.

**Required Security Scans in CI Pipeline:**

#### A. Container Image Scanning
- **Tool Options:** Trivy (recommended), Snyk, or Anchore
- **What to scan:** Your built Docker image
- **Action:** Fail the build if HIGH or CRITICAL vulnerabilities found

**Example (Trivy):**
```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'your-image:latest'
    format: 'sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'  # Fail the build
```

#### B. Infrastructure as Code Scanning
- **Tool Options:** tfsec (recommended), Checkov, or Terrascan
- **What to scan:** Your Terraform files in `terraform/` directory
- **Action:** Fail the build if security issues detected

**Example (tfsec):**
```yaml
- name: Run tfsec security scanner
  uses: aquasecurity/tfsec-action@v1.0.0
  with:
    working_directory: terraform/
    soft_fail: false  # Fail the build on issues
```

**Integration Requirements:**
- Both scans must run automatically on every Pull Request
- CI pipeline must be configured as a required status check
- PRs cannot be merged if security scans fail
- Failed scans should show clear error messages in the GitHub Actions log

---

### 4. Continuous Deployment (CD) Pipeline

**Tool:** GitHub Actions  
**Workflow File:** `.github/workflows/cd.yml`  
**Trigger:** Only on merge to `main` branch (NOT on PRs)

**Required Steps:**
```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      # 1. Run all CI checks
      - Checkout code
      - Run linting
      - Run tests
      - Run security scans
      
      # 2. Build and push to private registry
      - Build Docker image
      - Authenticate to ECR/ACR
      - Tag image with git SHA
      - Push to private registry
      
      # 3. Deploy via Ansible
      - Install Ansible
      - Configure SSH keys
      - Run Ansible playbook against Application VM
```

**Critical CD Requirements:**
- Must push to your private ECR/ACR (NOT Docker Hub)
- Must use GitHub Secrets for credentials (never hardcode)
- Must tag images with commit SHA for traceability
- Must run Ansible playbook to deploy (not manual SSH)
- Must include notification on success/failure (optional but recommended)

**Secrets to Configure:**
- `AWS_ACCESS_KEY_ID` / `AZURE_CREDENTIALS`
- `SSH_PRIVATE_KEY` (for Ansible to access VMs)
- Database credentials (if needed by your app)

---

## Pipeline Architecture

### CI Pipeline (`.github/workflows/ci.yml`)
**Triggers:** On Pull Requests to `main`  
**Purpose:** Quality gates before code can be merged

```
Pull Request Created
        ↓
    Checkout Code
        ↓
    Run Linter
        ↓
    Run Tests
        ↓
    Build Docker Image
        ↓
    Scan Image (Trivy)
        ↓
    Scan IaC (tfsec)
        ↓
    All Pass → Allow Merge
    Any Fail → Block Merge
```

### CD Pipeline (`.github/workflows/cd.yml`)
**Triggers:** On merge to `main` only  
**Purpose:** Automated deployment to production

```
Merge to Main
      ↓
  Run CI Checks
      ↓
  Build Image
      ↓
  Push to ECR/ACR
      ↓
  Run Ansible Playbook
      ↓
  Application Updated
      ↓
  Live on Public URL
```

**Important:** CD should NEVER run on PRs, only on successful merges to `main`.

---

## Technical Specifications

### Cloud Provider
- **Options:** AWS or Azure (choose one, be consistent throughout)
- **Region:** Choose one close to East Africa (e.g., `eu-west-1`, `southafricanorth`)

### Compute Resources
- **Bastion Host:** t2.micro (AWS) or B1s (Azure)
- **Application VM:** t2.micro (AWS) or B1s (Azure)
- **Database:** db.t3.micro RDS (AWS) or B-series Azure Database

### Container Registry
- **AWS:** Elastic Container Registry (ECR)
- **Azure:** Azure Container Registry (ACR)
- **Not Allowed:** Docker Hub (for final deployment)

### Application Requirements
- Must be containerized (built in Formative 2)
- Must connect to the managed database
- Must be accessible via HTTP/HTTPS
- Must demonstrate a visible change in your video demo

### Version Control
- **Single repository** containing all code
- Proper `.gitignore` (no secrets, no `terraform.tfstate`)
- Meaningful commit messages
- Branch protection rules enabled on `main`

---

## AI & Academic Integrity

### ALLOWED: AI Assistance for Application Code
You may use AI tools (GitHub Copilot, ChatGPT, Claude) to help write:
- Your application logic (Python, Node.js, Go, etc.)
- Database queries and models
- Frontend components and styling
- Unit tests for your application

### NOT ALLOWED: AI-Generated DevOps Configuration

**You must write the following files yourself:**
- `Dockerfile`
- `docker-compose.yml`
- GitHub Actions workflows (`.github/workflows/*.yml`)
- Terraform files (`terraform/*.tf`)
- Ansible playbooks (`ansible/*.yml`)

**Why this policy?**  
The purpose of this course is to learn DevOps skills. Using AI to generate configuration files prevents you from understanding these critical concepts. Submitting AI-generated DevOps config constitutes **academic misconduct** and will result in a zero for the project.

**How we verify:**  
We will review your git history, ask questions about your configuration choices, and may conduct verbal assessments of your understanding.

**If you're stuck:**  
- Review course materials and recorded lectures
- Attend office hours
- Post questions in the class discussion forum (without sharing complete solutions)
- Collaborate with your team members

---

## Deliverables

### 1. GitHub Repository

**Required Structure:**
```
your-repo/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # CI pipeline
│       └── cd.yml                    # CD pipeline
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
├── ansible/
│   ├── deploy.yml                    # Main playbook
│   ├── inventory.ini
│   └── README.md
├── src/                              # Your application code
├── docker-compose.yml
├── Dockerfile
├── README.md                         # Main documentation
└── .gitignore
```

### 2. Live Application
- Publicly accessible via HTTP/HTTPS
- Must show your application running (not a default web server page)
- Must demonstrate database connectivity
- Should load within 3 seconds

### 3. Video Demonstration (10-15 minutes)

**Required Content:**

**Introduction (1 min):**
- Team members and roles
- Brief overview of your application
- Architecture at a glance

**Live Demo (6-8 mins):**
1. **Starting State:** Show your live application currently running
2. **Code Change:** Make a small, visible change in your code
   - Examples: Change button text, modify heading, add a feature
3. **Git Workflow:** 
   - Commit and push to a feature branch
   - Create a Pull Request on GitHub
4. **CI Pipeline:** 
   - Show GitHub Actions running
   - Point out the security scans executing
   - Show that PR cannot merge until checks pass
5. **Merge:** 
   - Approve and merge the PR
   - Show merge commit in `main` branch
6. **CD Pipeline:** 
   - Show the deploy pipeline triggered automatically
   - Point out the image being pushed to ECR/ACR
   - Show Ansible playbook execution
7. **Verification:** 
   - Refresh your live application URL
   - Show the change is now deployed
   - Demonstrate that it happened automatically

**Wrap-up (1 min):**
- Briefly show your architecture diagram
- Mention any challenges overcome
- Summary of key learnings

**Video Format:**
- Upload to YouTube (unlisted) or Google Drive (view permissions)
- Include audio narration
- Use screen recording (Loom, OBS, QuickTime, etc.)
- Show your face in a small window (optional but encouraged)

### 4. README.md Documentation

Your repository's README.md is the "operations manual" for your project.

**Required Sections:**

```markdown
# Project Name

## Team Members
- Name 1 (Role: Terraform/IaC)
- Name 2 (Role: Ansible/CD)
- Name 3 (Role: CI/Security)

## Live Application
[Access Live App](http://your-public-url)

## Architecture Overview

### Architecture Diagram
[Include a clear diagram showing all components]

### Component Description
- Brief explanation of each infrastructure component
- How they connect and communicate
- Security controls in place

## Technology Stack
- Cloud Provider: AWS/Azure
- Application: Python Flask / Node.js Express / etc.
- Database: PostgreSQL / MySQL / etc.
- Container Registry: ECR / ACR
- IaC: Terraform
- Config Management: Ansible
- CI/CD: GitHub Actions

## Repository Structure
[Explain the directory layout]

## Setup Instructions

### Prerequisites
- AWS/Azure account with appropriate permissions
- Terraform installed
- Ansible installed
- GitHub account

### Deployment Steps
1. Clone the repository
2. Configure Terraform variables
3. Initialize and apply Terraform
4. [Detailed step-by-step instructions]

### Tearing Down
How to cleanly destroy all resources

## CI/CD Pipeline

### CI Pipeline
- Triggers on: Pull Requests
- Steps: [List the steps]
- Security scans: [Which tools and what they check]

### CD Pipeline
- Triggers on: Merge to main
- Deployment process: [Explain the flow]

## Security Measures
- List all security scans implemented
- Explain network security configuration
- Secret management approach

## Challenges & Solutions
[Brief discussion of major challenges and how you solved them]

## Video Demo
[Watch Demo Video](video-link)

## License
[If applicable]
```

**Architecture Diagram Requirements:**
- Must be a visual diagram (not just text)
- Should show: Internet → Load Balancer/Bastion → Private VM → Database
- Include all security groups/firewall rules
- Show the CI/CD flow
- Tools: draw.io, Lucidchart, or similar
- Export as image and embed in README

---

## Common Pitfalls to Avoid

### Security Issues
❌ Hardcoding secrets in code (API keys, passwords)  
✅ Use GitHub Secrets and environment variables

❌ Making database publicly accessible  
✅ Keep database in private subnet, only accessible from app VM

❌ SSH keys committed to repository  
✅ Add to .gitignore, use GitHub Secrets

❌ Overly permissive security groups (0.0.0.0/0 for all ports)  
✅ Restrict to only necessary ports and sources

### Pipeline Issues
❌ Running CD pipeline on every PR  
✅ CD should only trigger on merge to `main`

❌ Using Docker Hub instead of private registry  
✅ Must use ECR (AWS) or ACR (Azure)

❌ Not failing builds on security vulnerabilities  
✅ Configure scans with `exit-code: 1`

❌ Manual deployment steps  
✅ Everything automated via GitHub Actions and Ansible

### Infrastructure Issues
❌ Hardcoding IPs and values in Terraform  
✅ Use variables for all configurable values

❌ Not using Bastion as jump host for Ansible  
✅ Configure `ansible_ssh_common_args` properly

❌ Forgetting to test `terraform destroy`  
✅ Test destroy/recreate cycle before submission

❌ Not outputting important values (IPs, URLs)  
✅ Use `outputs.tf` to expose necessary information

### Documentation Issues
❌ No architecture diagram or poor quality diagram  
✅ Create clear, professional diagram using proper tools

❌ README doesn't explain how to deploy  
✅ Include step-by-step setup instructions

❌ Video doesn't show complete workflow  
✅ Demonstrate full Git-to-Production flow

❌ No explanation of team member contributions  
✅ Document who did what in README

---

## Submission Guidelines

### Canvas Submission (One per Team)

**Part 1: GitHub Repository URL**
```
https://github.com/your-team/your-repo-name
```

**Part 2: Text Entry Box**
```
Team Name: DevOps Masters
Team Members: 
- Alice Mutesi (Terraform & Infrastructure)
- Bob Kamau (Ansible & Configuration Management)  
- Carol Ochieng (CI/CD & Security Scanning)

Live Application URL: http://54.123.456.78

Video Demonstration: https://youtu.be/your-video-id
(or Google Drive link with view permissions)

Special Notes: [Any important information for graders]
```

### Pre-Submission Testing

Before submitting, complete this test sequence:

**Test 1: Fresh Infrastructure Build**
1. Run `terraform destroy` to remove everything
2. Run `terraform apply` to rebuild from scratch
3. Verify all resources created successfully
4. Verify application accessible

**Test 2: Complete Git-to-Production Flow**
1. Make a small code change
2. Create PR and verify CI runs
3. Merge PR and verify CD deploys automatically
4. Confirm change visible on live URL
5. Time the process (should be < 5 minutes)

**Test 3: Clean Teardown**
1. Run `terraform destroy`
2. Verify all resources removed from cloud console
3. Check for any orphaned resources

---

## Success Checklist

### Infrastructure 
- [ ] `terraform/` directory contains modular Terraform code
- [ ] All 6 infrastructure components provisioned
- [ ] `terraform apply` runs without errors
- [ ] `terraform destroy` cleanly removes all resources
- [ ] Outputs provide necessary IPs and URLs

### Configuration Management
- [ ] `ansible/` directory contains deployment playbook
- [ ] Playbook configures VM completely (Docker, packages, deploy)
- [ ] Ansible uses Bastion Host as jump server
- [ ] Playbook can be run multiple times (idempotent)

### CI Pipeline
- [ ] CI workflow triggers on Pull Requests
- [ ] Runs linting and tests
- [ ] Scans Docker images for vulnerabilities
- [ ] Scans Terraform code for security issues
- [ ] Fails build on HIGH/CRITICAL issues
- [ ] Set as required status check (blocks merges)

### CD Pipeline
- [ ] CD workflow triggers ONLY on merge to `main`
- [ ] Pushes images to private ECR/ACR (not Docker Hub)
- [ ] Uses GitHub Secrets for all credentials
- [ ] Automatically runs Ansible playbook
- [ ] Successfully deploys without manual intervention

### Security
- [ ] No secrets hardcoded in repository
- [ ] Database in private subnet
- [ ] Application VM in private subnet
- [ ] Security groups properly configured
- [ ] SSH access only through Bastion
- [ ] `.gitignore` includes terraform.tfstate and secrets

### Documentation
- [ ] README.md includes architecture diagram
- [ ] README.md has clear setup instructions
- [ ] README.md documents team contributions
- [ ] README.md includes live URL
- [ ] Repository is well-organized
- [ ] Code includes helpful comments

### Demo
- [ ] Live application URL works and shows your app
- [ ] Video demonstrates complete Git-to-Production flow
- [ ] Video is 5-10 minutes long
- [ ] Video includes clear narration
- [ ] Video uploaded with proper permissions
- [ ] Change visible in demo is meaningful

---

## Getting Help

### Resources
- **Course Materials:** Review Session recordings & Guided Learning Activites
- **Office Hours:** [Insert schedule]
- **Discussion Forum:** Post questions (don't share complete solutions)
- **Official Documentation:**
  - [Terraform Docs](https://www.terraform.io/docs)
  - [Ansible Docs](https://docs.ansible.com)
  - [GitHub Actions Docs](https://docs.github.com/actions)

### Common Questions
**Q: Can we use existing Terraform modules from the internet?**  
A: You can reference them for learning, but must write your own code.

**Q: Our CI pipeline takes too long, can we skip some checks?**  
A: No, all security scans are required. Optimize where possible.

**Q: Can we deploy to Heroku/Vercel instead?**  
A: No, you must provision your own infrastructure with Terraform.

**Q: What if our cloud costs exceed free tier?**  
A: Use t2.micro/B1s instances and delete resources promptly after testing.

---

## Academic Integrity Reminder

This is a team project, but each team member should contribute meaningfully to all aspects. Your git commit history and individual understanding will be assessed. Be prepared to:
- Explain any configuration file in your repository
- Demonstrate how to modify infrastructure or pipelines
- Discuss design decisions you made as a team

**Remember:** Using AI for DevOps configuration files is academic misconduct. Write your own Terraform, Ansible, and GitHub Actions configurations.

---

**Good luck! This project represents the culmination of everything you've learned. Take pride in building a professional-grade DevOps pipeline.**
