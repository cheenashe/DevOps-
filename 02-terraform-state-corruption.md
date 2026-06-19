# Terraform State Corruption — How to Recover Without Losing Infrastructure

> A practical recovery guide for when `terraform.tfstate` goes wrong in production.

---

## Table of Contents

- [What is Terraform State and Why It Breaks](#what-is-terraform-state-and-why-it-breaks)
- [Recognizing State Corruption](#recognizing-state-corruption)
- [Step 1 — Stop Everything First](#step-1--stop-everything-first)
- [Step 2 — Backup Current State](#step-2--backup-current-state)
- [Step 3 — Diagnose the Corruption](#step-3--diagnose-the-corruption)
- [Recovery Scenarios](#recovery-scenarios)
- [Preventing State Corruption](#preventing-state-corruption)
- [Remote State Best Practices](#remote-state-best-practices)

---

## What is Terraform State and Why It Breaks

Terraform's state file maps your real cloud resources to your configuration. When it breaks, Terraform either tries to recreate resources that exist, or ignores resources it should manage.

**Common causes of corruption:**

| Cause | What Happens |
|-------|-------------|
| Two engineers run `terraform apply` simultaneously | Concurrent writes corrupt the state |
| Manual resource deletion in cloud console | State references a resource that no longer exists |
| Failed mid-apply | Partial state written, inconsistent with reality |
| `terraform force-unlock` used incorrectly | Lock removed while another apply was running |
| State file edited manually with errors | JSON structure broken |
| S3 versioning disabled | Previous good state permanently lost |

---

## Recognizing State Corruption

```bash
# These errors signal state issues:

# Error 1 — resource exists in state but not in cloud
Error: error reading EC2 Instance (i-0abc123def): InvalidInstanceID.NotFound

# Error 2 — duplicate resource
Error: Provider configuration not present
A provider configuration block is required

# Error 3 — state lock stuck
Error: Error locking state: Error acquiring the state lock
Lock Info:
  ID: 12345678-abcd-efgh-ijkl-123456789012
  Operation: OperationTypeApply
  Who: user@hostname

# Error 4 — JSON parse error (manual edit gone wrong)
Error: Failed to read state: state file was created by a newer version of Terraform
```

---

## Step 1 — Stop Everything First

**Before touching anything:**

```bash
# 1. Tell your team — no one runs terraform apply until this is resolved
# 2. Check if a lock exists
terraform force-unlock --help  # Do NOT run this yet

# 3. Check who holds the lock (AWS DynamoDB backend)
aws dynamodb get-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "your-state-path/terraform.tfstate"}}' \
  --region us-east-1

# If the lock owner's process is dead, then it's safe to force-unlock
terraform force-unlock <LOCK_ID>
```

> ⚠️ **Never force-unlock if you're not sure the locking process is dead.** Two concurrent applies = guaranteed corruption.

---

## Step 2 — Backup Current State

```bash
# Always backup BEFORE any recovery attempt

# If using S3 backend — download current state
aws s3 cp s3://your-terraform-state-bucket/path/terraform.tfstate \
  ./terraform.tfstate.backup-$(date +%Y%m%d-%H%M%S)

# If using local state
cp terraform.tfstate terraform.tfstate.backup-$(date +%Y%m%d-%H%M%S)

# If S3 versioning is enabled — list versions (your lifeline)
aws s3api list-object-versions \
  --bucket your-terraform-state-bucket \
  --prefix path/terraform.tfstate \
  --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified}' \
  --output table
```

---

## Step 3 — Diagnose the Corruption

```bash
# Validate the state file is valid JSON
cat terraform.tfstate | python3 -m json.tool > /dev/null && echo "Valid JSON" || echo "Invalid JSON"

# Check what terraform thinks it manages
terraform state list

# Check a specific resource
terraform state show aws_instance.my_server

# Compare state vs reality (dry run — no changes made)
terraform plan -detailed-exitcode
# Exit code 0 = no changes
# Exit code 1 = error
# Exit code 2 = changes needed (state drift)
```

---

## Recovery Scenarios

### Scenario A — State lock stuck (no actual corruption)

```bash
# Get the lock ID from the error message
terraform force-unlock <LOCK-ID>

# Verify it worked
terraform plan
```

---

### Scenario B — Resource exists in cloud but missing from state

This happens after manual resource creation or a partial state loss.

```bash
# Import the existing resource into state
# Syntax: terraform import <resource_type>.<resource_name> <cloud_resource_id>

# Example — import an existing EC2 instance
terraform import aws_instance.web_server i-0abc123def456

# Example — import an existing S3 bucket
terraform import aws_s3_bucket.assets my-existing-bucket-name

# Example — import an existing RDS instance
terraform import aws_db_instance.database mydb-identifier

# After import, run plan to verify no unintended changes
terraform plan
```

---

### Scenario C — Resource in state but deleted from cloud

```bash
# Remove the orphaned resource from state
terraform state rm aws_instance.deleted_server

# Verify it's gone
terraform state list | grep deleted_server

# Run plan — Terraform will now plan to recreate it
# Decide: do you want it recreated, or was deletion intentional?
terraform plan
```

---

### Scenario D — Corrupted state file (invalid JSON)

```bash
# Step 1 — Restore from S3 versioning (best option)
aws s3api get-object \
  --bucket your-terraform-state-bucket \
  --key path/terraform.tfstate \
  --version-id <PREVIOUS_VERSION_ID> \
  terraform.tfstate.restored

# Verify it's valid
cat terraform.tfstate.restored | python3 -m json.tool > /dev/null && echo "Valid"

# Upload restored state
aws s3 cp terraform.tfstate.restored \
  s3://your-terraform-state-bucket/path/terraform.tfstate

# Step 2 — If no backup exists, rebuild from scratch
terraform plan -generate-config-out=generated.tf  
# Terraform 1.5+ can generate config from existing resources
```

---

### Scenario E — Partial apply left infrastructure in unknown state

```bash
# 1. Run plan to see what Terraform thinks needs to happen
terraform plan -out=recovery.tfplan

# 2. Review carefully — look for unexpected destroys
terraform show recovery.tfplan | grep -A 5 "will be destroyed"

# 3. If destroys look wrong, use targeted apply to fix specific resources
terraform apply -target=aws_instance.web_server
terraform apply -target=aws_security_group.web_sg

# 4. Never use -target as a regular workflow — only for recovery
```

---

### Scenario F — Two state files diverged (team conflict)

```bash
# Pull both states and compare
terraform state pull > state-current.json

# Download the other engineer's state (from backup or their local)
# Compare resources in both
cat state-current.json | jq '.resources[].name' | sort > resources-current.txt
cat state-other.json | jq '.resources[].name' | sort > resources-other.txt

diff resources-current.txt resources-other.txt

# Merge carefully using terraform state mv and terraform import
# There is no automated merge — this requires manual reconciliation
```

---

## Preventing State Corruption

### 1. Always use remote state with locking

```hcl
# backend.tf — S3 + DynamoDB for locking
terraform {
  backend "s3" {
    bucket         = "your-org-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # ← enables locking
  }
}
```

```bash
# Create the DynamoDB lock table (one-time setup)
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

---

### 2. Enable S3 versioning (your time machine)

```bash
aws s3api put-bucket-versioning \
  --bucket your-org-terraform-state \
  --versioning-configuration Status=Enabled

# Verify
aws s3api get-bucket-versioning --bucket your-org-terraform-state
```

---

### 3. Never allow direct console changes to Terraform-managed resources

Add an SCP or IAM policy tag condition:

```json
{
  "Effect": "Deny",
  "Action": ["ec2:TerminateInstances", "ec2:StopInstances"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/ManagedBy": "terraform"
    },
    "StringNotEquals": {
      "aws:CalledVia": ["cloudformation.amazonaws.com"]
    }
  }
}
```

---

### 4. Run terraform plan in CI before every apply

```yaml
# .github/workflows/terraform.yml
name: Terraform Plan

on:
  pull_request:
    branches: [main]

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -no-color -out=tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const plan = require('fs').readFileSync('tfplan_output.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `\`\`\`hcl\n${plan}\n\`\`\``
            });
```

---

## Remote State Best Practices

```
✅ Use S3 + DynamoDB (AWS) or GCS with locking (GCP) or Terraform Cloud
✅ Enable S3 versioning — keep at least 30 versions
✅ Encrypt state at rest (contains secrets!)
✅ Never commit tfstate files to git
✅ Use separate state files per environment (dev/staging/prod)
✅ Use workspaces or separate backends per environment
✅ Add .tfstate and .tfstate.backup to .gitignore
✅ Rotate state access credentials regularly
✅ Use CI/CD for all applies — no local terraform apply in production
```

```bash
# .gitignore additions for Terraform
*.tfstate
*.tfstate.*
*.tfvars
.terraform/
tfplan
crash.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

---

## When State Management Becomes a Team Problem

Managing Terraform state across multiple engineers, environments, and cloud accounts requires more than a shared S3 bucket.

> **Sygitech's [DevOps & Automation Services](https://www.sygitech.com/devops-and-automation-services.html)** help engineering teams implement production-grade IaC pipelines — with proper state management, CI/CD guardrails, and drift detection built in from day one.
>
> 👉 [Talk to our DevOps engineers](https://www.sygitech.com/devops-and-automation-services.html)

---

*Maintained by [Sygitech](https://www.sygitech.com) — Managed Cloud Services for Engineering Teams*  
*Have a recovery scenario not covered here? Open a Discussion — we'll add it.*
