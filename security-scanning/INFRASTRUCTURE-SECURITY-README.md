# 🛡️ Pattern Infrastructure Security Scanning Guide

## 🎯 What is This?

Automated security scanning for all infrastructure container images and Kubernetes configurations using [Trivy](https://trivy.dev/). Protects your EKS cluster, worker nodes, and infrastructure components from security vulnerabilities.

**What Gets Scanned:**
- ✅ **EKS Infrastructure**: CoreDNS, pause containers, worker node images
- ✅ **AWS Services**: Amazon Linux, Neuron/Inferentia images
- ✅ **Test/Utility Images**: nginx, busybox, aws-cli
- ✅ **Terraform Configs**: Infrastructure-as-code security
- ✅ **Karpenter Configs**: Node provisioning security

## 👨‍💻 For Infrastructure Teams: What You Need to Know

### **🚀 The Good News**
- **100% Automated** - Runs on every infrastructure change
- **Non-blocking** - Your deployments continue normally
- **Zero Setup** - No tools to install, no credentials to manage

### **🛡️ Will This Break Anything?**
**Absolutely NOT!** The security scanning is designed to be completely safe:
- ✅ **Never fails deployments** - Uses `exit-code: '0'` to report only
- ✅ **Never stops infrastructure** - Runs parallel to your normal workflow
- ✅ **Never modifies configs** - Read-only scanning and reporting

**Think of it like infrastructure monitoring** - it watches and reports, but never interferes.

### **🏢 Current Security Team Status**
**Important Note:** We currently don't have a dedicated security team. This means:
- ✅ **Fixable vulnerabilities** - We'll update images when fixes are available
- ⚠️ **No-fix vulnerabilities** - Will remain in Security tab until vendor releases fix
- ⚠️ **Complex security issues** - May require external security consultation

**This is normal for startups** - having visibility into infrastructure security is better than having none.

### **🖥️ Platform Support**
**Automatic Scanning:** Works on all platforms (runs in GitHub Actions)

**Local Testing:**
- **Linux/Mac:** `trivy image nginx:latest`
- **Windows:** Use WSL or install Trivy manually

## 📊 How It Works

**Triggers:**
- Every push to `main` or `develop` branches
- Every pull request to `main`
- Daily at 2 AM UTC (catches new vulnerabilities)

**When you see a security alert:**
1. **Go to GitHub Security tab** → See vulnerability details
2. **Update the image version** → Change image tags in YAML files
3. **Push the change** → Security scan runs automatically
4. **Done** → Alert disappears when fixed

### **⚠️ Important: Trivy Only Scans, Doesn't Fix**
**What Trivy does:**
- ✅ **Scans and reports** vulnerabilities in infrastructure images
- ✅ **Tells you what to fix** (image versions, configurations)

**What Trivy does NOT do:**
- ❌ **Auto-update infrastructure** - You manually update YAML files
- ❌ **Auto-rebuild clusters** - You manually apply changes
- ❌ **Auto-deploy fixes** - You control when infrastructure changes

**Example workflow:**
```
Day 1: Trivy finds CVE-2023-12345 in nginx:latest
Day 1: You get GitHub Security alert
Day 2: You update: nginx:latest → nginx:1.25.3-alpine
Day 2: You apply: kubectl apply -f updated-deployment.yaml
Day 3: Trivy scans again → CVE fixed ✅
```

## 🛠️ Infrastructure Images We Monitor

### **EKS Core Components**
```yaml
# CoreDNS
image: 602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/coredns:v1.11.4-eksbuild.2

# Pause containers
image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
image: public.ecr.aws/eks-distro/kubernetes/pause:3.10
```

### **AWS Infrastructure**
```yaml
# Amazon Linux base
image: public.ecr.aws/amazonlinux/amazonlinux:2023

# Neuron/Inferentia ML
image: public.ecr.aws/neuron/pytorch-inference-neuron:1.13.1-neuron-py310-sdk2.20.2-ubuntu20.04
```

### **Test & Utility Images**
```yaml
# Web servers
image: nginx:latest

# Debugging tools  
image: busybox:1.28
image: amazon/aws-cli:latest

# GPU testing
image: nvidia/cuda:11.6.2-base-ubuntu20.04
```

## 📊 Understanding Security Results

| Severity | What it means | Action needed |
|----------|---------------|---------------|
| **CRITICAL** | Immediate infrastructure risk | Fix immediately |
| **HIGH** | Significant cluster risk | Fix within 1 week |
| **MEDIUM** | Moderate infrastructure risk | Fix in next sprint |
| **LOW** | Minor security issue | Fix when convenient |

**Note**: LOW priority alerts are excluded from automatic scans to reduce noise.

## 🔧 Common Infrastructure Fixes

**Update Container Images:**
```yaml
# Before (vulnerable)
image: nginx:latest

# After (you manually change this)
image: nginx:1.25.3-alpine
```

**Update EKS Components:**
```yaml
# Before
image: 602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/coredns:v1.11.4-eksbuild.2

# After (when AWS releases update)
image: 602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/coredns:v1.11.5-eksbuild.1
```

**Terraform Security:**
```hcl
# Add security configurations
resource "aws_security_group" "example" {
  # Remove overly permissive rules
  # ingress {
  #   from_port = 0
  #   to_port   = 65535
  #   protocol  = "tcp"
  #   cidr_blocks = ["0.0.0.0/0"]  # ← Remove this
  # }
}
```

## 🧪 Local Testing (Optional)

```bash
# Install Trivy
# Linux: curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
# Mac: brew install trivy

# Test infrastructure images
trivy image nginx:latest
trivy image busybox:1.28
trivy image public.ecr.aws/amazonlinux/amazonlinux:2023

# Test Terraform configs
trivy config terraform/
```

## 🚨 Troubleshooting

**"Critical vulnerabilities found":**
1. Check GitHub Security tab for details
2. Look for "Fixed Version" in the alert
3. Update the image tag in your YAML files
4. Apply changes: `kubectl apply -f your-file.yaml`

**"Cannot scan private ECR images":**
- Private AWS ECR images require authentication
- Focus on public images first
- Use AWS CLI for private image scanning locally

## 🎯 Quick Reference

**Daily Infrastructure Workflow:**
```bash
# Normal infrastructure work (security scanning is automatic)
git add terraform/
git commit -m "Update EKS node groups"
git push origin main

# Check results: GitHub → Security → Code scanning
```

**Infrastructure Security Response:**
```bash
# When alerts appear
1. Check GitHub Security tab
2. Update image tags in YAML/Terraform files
3. Test changes in dev environment
4. Apply to production
5. Verify fix in next security scan
```

---

**Questions?** Check the GitHub Security tab or test images locally with `trivy image <image-name>`.