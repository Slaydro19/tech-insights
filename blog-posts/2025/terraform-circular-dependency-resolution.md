---
title: "Resolving Terraform Circular Dependencies: A Real-World AWS EKS Case Study"
date: 2025-01-08
author: "Jalen Salgado"
tags: [Terraform, AWS, EKS, Infrastructure, DevOps, Circular Dependencies]
category: Infrastructure Engineering
word_count: ~1,100
reading_time: 4 minutes
description: "A practical guide to identifying and resolving Terraform circular dependencies in AWS EKS infrastructure, with real-world examples and multiple solution approaches."
---

# Resolving Terraform Circular Dependencies: A Real-World AWS EKS Case Study

## The Problem: When Infrastructure Gets Stuck

During a recent AWS EKS infrastructure deployment, we encountered a classic Terraform issue that many infrastructure engineers face: **circular dependencies**. While `terraform plan` and `terraform apply` worked perfectly, `terraform destroy` would consistently fail after 15+ minutes with timeout errors.

## Understanding Circular Dependencies

### What Is a Circular Dependency?

A circular dependency occurs when two resources need each other to exist, but neither can be created first. Think of it like this:

- **Person A**: "I'll open the door after you go through"
- **Person B**: "I'll go through after you open the door"
- **Result**: Nobody moves = Deadlock

### Our Specific Case

The issue manifested in our Terraform configuration across two key files:

**File 1: `terraform/vpc.tf` (Line 122)**
```hcl
security_groups = [module.eks.cluster_primary_security_group_id]
```
*Translation: "VPC security group needs EKS security group to exist first"*

**File 2: `terraform/eks.tf` (Lines 20-21)**
```hcl
vpc_id     = module.vpc.vpc_id
subnet_ids = module.vpc.private_subnets
```
*Translation: "EKS cluster needs VPC to exist first"*

### Why Apply Works But Destroy Fails

- **terraform apply**: Terraform automatically figures out the correct build order
- **terraform destroy**: Terraform gets stuck in the circular reference and can't determine deletion order

The specific error we encountered:
```
Error: deleting Security Group (sg-0d113cc0a7eb09cdd): operation error EC2: DeleteSecurityGroup, 
https response error StatusCode: 400, RequestID: 44b5e36d-e07f-4e49-8e7e-823c59df365f, 
api error DependencyViolation: resource sg-0d113cc0a7eb09cdd has a dependent object
```

## Solution Approaches

We evaluated four potential solutions, each with distinct trade-offs:

### Solution 1: CIDR Blocks Instead of Security Group References

**Change:**
```hcl
# Before
security_groups = [module.eks.cluster_primary_security_group_id]

# After  
cidr_blocks = [var.vpc_cidr]
```

**Trade-offs:**
- ✅ Completely eliminates circular dependency
- ✅ Simple one-line change
- ✅ Guaranteed to fix destroy issues
- ❌ Less granular security (allows access from entire VPC instead of just EKS nodes)

### Solution 2: Separate Security Group Rules (Our Choice)

**Approach:**
1. Create the security group without EKS references
2. Add EKS-specific rules separately using explicit dependencies
3. Use `depends_on` to control resource creation order

**Trade-offs:**
- ✅ Maintains granular security (security group to security group)
- ✅ Eliminates circular dependency through explicit ordering
- ✅ Best practice for complex Terraform dependencies
- ❌ More complex code structure (multiple resources instead of one)

### Solution 3: Move VPC Endpoints to Separate Module

**Approach:** Extract all VPC endpoint resources into their own module with proper dependency management.

**Trade-offs:**
- ✅ Clean separation of concerns
- ✅ Easier dependency management
- ❌ More files to maintain
- ❌ Additional complexity

### Solution 4: Use Data Sources

**Change:**
```hcl
data "aws_security_group" "eks_cluster" {
  name = module.eks.cluster_primary_security_group_name
}

resource "aws_vpc_endpoint" "example" {
  security_group_ids = [data.aws_security_group.eks_cluster.id]
}
```

**Trade-offs:**
- ✅ Breaks the circular dependency
- ❌ Requires the security group to already exist
- ❌ Fragile if naming changes

## Implementation: Separate Security Group Rules

We chose **Solution 2** for its balance of security and maintainability. Here's how we implemented it:

```hcl
# Create security group without EKS references
resource "aws_security_group" "vpc_endpoint" {
  name_prefix = "${var.cluster_name}-vpc-endpoint-"
  vpc_id      = aws_vpc.this.id
  
  tags = merge(local.tags, {
    Name = "${var.cluster_name}-vpc-endpoint-sg"
  })
}

# Add rules separately with explicit dependencies
resource "aws_security_group_rule" "vpc_endpoint_https_ingress" {
  type                     = "ingress"
  from_port               = 443
  to_port                 = 443
  protocol                = "tcp"
  source_security_group_id = module.eks.cluster_primary_security_group_id
  security_group_id       = aws_security_group.vpc_endpoint.id
  
  depends_on = [module.eks]
}
```

## Key Lessons Learned

### 1. Context Matters for AI Assistance

During troubleshooting, we discovered that when we moved to a new Claude AI chat session and provided the same information, it told us the issue was already fixed. This highlighted the critical importance of **context engineering** - providing AI tools with complete background and history for accurate analysis.

### 2. Prevention is Better Than Cure

**Best Practices to Avoid Circular Dependencies:**
- Use explicit `depends_on` statements for complex dependencies
- Separate resource creation from rule/policy attachment
- Consider using data sources for existing resources
- Plan your module dependencies before implementation

### 3. Testing Destroy Operations

Always test `terraform destroy` in development environments. Many circular dependency issues only surface during destruction, not creation.

## Verification and Results

After implementing the separate security group rules approach:

1. **✅ terraform plan**: Clean execution
2. **✅ terraform apply**: All resources created successfully  
3. **✅ terraform destroy**: Complete destruction without timeouts
4. **✅ Security posture**: Maintained granular access controls

## Conclusion

Circular dependencies in Terraform are common but solvable. The key is understanding the dependency chain and choosing the right approach for your specific use case. In our case, separating security group rules provided the best balance of security, maintainability, and operational reliability.

For infrastructure teams facing similar issues, remember that the goal isn't just to make the code work—it's to create maintainable, secure, and operationally sound infrastructure that can be reliably managed throughout its lifecycle.

---

*This case study demonstrates the importance of thorough testing and the value of understanding Terraform's dependency resolution mechanisms in complex AWS environments.*