# Vanta Compliance Fixes for ubinfra

> **Note:** This knowledgebase is local reference documentation only — it is not tracked in git.

## Overview

Vanta flags AWS infrastructure compliance issues. This document tracks findings and their fixes in ubinfra.

## Status Summary (2026-02-04)

| Issue | Status |
|-------|--------|
| MFA on infrastructure provider | **FIXED** - IAM user deleted, using OIDC + SSO |
| Point-in-time recovery (DynamoDB) | **FIXED** - Terraform automation in place |
| Server logs retained 365 days | **FIXED** - Terraform automation in place |
| Password policy configured | **FIXED** - Set via AWS Console |
| SQS queue age monitored | **FIXED** - CloudWatch alarms on runs-on SQS queues |
| VPC flow logs enabled | **FIXED** - Flow logs on runs-on VPCs |
| DynamoDB read/write capacity monitored | **FIXED** - CloudWatch alarms on runs-on tables |
| Security group allows public SSH | **FIXED** - Replaced 0.0.0.0/0 with VPC CIDR on port 22 |

---

## Issues and Resolutions

### 1. MFA on infrastructure provider (IAM User)

**Finding:** IAM user `packer-builder-ci` has no MFA.

**Problem:** MFA cannot be used with programmatic access in CI/CD pipelines (no human to enter codes). Shared MFA for a service account defeats the purpose.

**Solution:**
- **CI:** Replace IAM user with OIDC authentication
- **Local:** Use AWS IAM Identity Center (SSO) - each developer has own MFA
- **Delete IAM user:** Eliminates the compliance issue entirely

**Implementation:**
- File: `deploy/sst/stacks/Idp.ts` - OIDC role `github-action-packer-build`
- Workflow: `.github/workflows/rebuild-ami.yml` uses `role-to-assume`
- Local: AWS SSO configured in `~/.aws/config`

**Completed:** IAM user `packer-builder-ci` deleted. SSO is now default for local access.

---

### 2. Point-in-time recovery on DynamoDB (2 tables)

**Finding:** DynamoDB tables `runs-on-workflow-jobs` and `runs-on-locks` don't have PITR enabled.

**Problem:** These tables are created by runs-on's CloudFormation, not our IaC.

**Solution:** Terraform module that enables PITR without taking ownership of the tables.
- Uses `null_resource` with `local-exec` to run AWS CLI
- Doesn't import tables into Terraform state (avoids CloudFormation conflict)
- Can be re-run if runs-on updates their stack

**Implementation:**
- Module: `deploy/terraform/runs-on-compliance/`
- Workflow: `.github/workflows/runs-on-compliance.yml` (manual trigger)
- Permissions in OIDC role: `dynamodb:UpdateContinuousBackups`, `dynamodb:DescribeContinuousBackups`

**Completed:** PITR enabled on both tables.

---

### 3. Server logs retained for 365 days

**Finding:** CloudWatch Log Groups don't retain logs for 365 days.

**Problem:** runs-on creates log groups with short/no retention:
- `/aws/apprunner/RunsOnService-.../application` - was: no retention
- `/aws/apprunner/RunsOnService-.../service` - was: no retention
- `runs-on-EC2InstanceLogGroup-...` - was: 7 days

**Solution:** Same Terraform module sets 365-day retention on all runs-on log groups.

**Implementation:**
- Module: `deploy/terraform/runs-on-compliance/`
- Permissions in OIDC role: `logs:PutRetentionPolicy`, `logs:DescribeLogGroups`

**Completed:** All log groups now have 365-day retention.

---

### 5. Messaging queue message age monitored (SQS)

**Finding:** Vanta flagged missing CloudWatch alarms for the `ApproximateAgeOfOldestMessage` metric on runs-on SQS queues.

**Problem:** runs-on creates SQS queues outside our IaC, so alarms were not configured.

**Solution:** Terraform module that discovers runs-on queues by prefix and creates CloudWatch alarms without taking ownership of the queues.

**Implementation:**
- Module: `deploy/terraform/runs-on-sqs-alarms/`
- Workflow: `.github/workflows/runs-on-sqs-alarms.yml` (manual trigger)
- Defaults: threshold `900s`, period `60s`, evaluation `5`, datapoints `3`, missing data `notBreaching`

**Completed:** Alarms created for all runs-on queues in `eu-central-1` (as of 2026-02-06).

---

### 6. VPC Flow Logs enabled

**Finding:** Vanta flagged VPCs without Flow Logs enabled.

**Problem:** runs-on creates VPCs outside our IaC, so Flow Logs were not configured.

**Solution:** Terraform module that discovers runs-on VPCs by Name prefix and explicit IDs, then creates Flow Logs to CloudWatch Logs.

**Implementation:**
- Module: `deploy/terraform/runs-on-vpc-flow-logs/`
- Workflow: `.github/workflows/runs-on-vpc-flow-logs.yml` (manual trigger)
- Defaults: traffic `ALL`, log group `/aws/vpc-flow-logs/runs-on`, retention disabled

**Completed:** Flow Logs enabled for runs-on VPCs in `eu-central-1` (as of 2026-02-06).

---

### 7. NoSQL database read/write capacity monitored (DynamoDB)

**Finding:** Vanta flagged missing CloudWatch alarms for `ConsumedWriteCapacityUnits` and `ConsumedReadCapacityUnits` on runs-on DynamoDB tables.

**Problem:** runs-on creates tables outside our IaC, so alarms were not configured.

**Solution:** Terraform module that discovers runs-on tables by prefix and creates CloudWatch alarms for read and write capacity.

**Implementation:**
- Module: `deploy/terraform/runs-on-dynamodb-alarms/`
- Workflow: `.github/workflows/runs-on-dynamodb-alarms.yml` (manual trigger)
- Defaults: thresholds `1000`, period `60s`, evaluation `5`, datapoints `3`, missing data `notBreaching`

**Completed:** Alarms created for runs-on tables in `eu-central-1` (as of 2026-02-06).

---

### 8. Security group allows public SSH access

**Finding:** Security group `sg-09374b75c7ac9154a` (`runs-on-SecurityGroup-It1pcthzw26H`) allows inbound SSH (port 22) from `0.0.0.0/0`.

**Problem:** The security group is managed by runs-on's CloudFormation stack, which sets a public SSH rule by default.

**Solution:** Terraform module that revokes the public rule and replaces it with a VPC-scoped rule (`10.1.0.0/16`).
- Uses `null_resource` with `local-exec` to avoid CloudFormation state conflicts
- Revokes `0.0.0.0/0` on port 22, authorizes VPC CIDR on port 22
- Verification script checks no public rules remain on ports 22 or 3389

**Implementation:**
- Module: `deploy/terraform/runs-on-sg-hardening/`
- Workflow: `.github/workflows/runs-on-sg-hardening.yml` (manual trigger)
- Permissions in OIDC role: `ec2:RevokeSecurityGroupIngress` (added to `PackerEC2` statement)

**Completed:** Public SSH rule replaced with VPC-scoped rule.

---

### 4. Password policy configured

**Finding:** IAM password policy not configured.

**Note:** This is an account-level setting, not per-project.

**Solution:** Configured via AWS Console with:
- Minimum length: 14 characters
- Require uppercase, lowercase, numbers, symbols
- Password expiration: 90 days
- Prevent reuse: 24 passwords

**Completed:** Set via AWS Console.

---

## OIDC Role Permissions Reference

The `github-action-packer-build` role in `Idp.ts` has four permission blocks:

| Sid | Purpose | Resource Scope |
|-----|---------|----------------|
| `PackerEC2` | Packer AMI builds + SG hardening | `*` (Packer creates temp resources with unpredictable ARNs) |
| `TerraformS3State` | Terraform state files | `logs-tfstate-999229612575/ubinfra/*` |
| `DynamoDBPITR` | Enable PITR on runs-on tables | `runs-on-*` tables only |
| `CloudWatchLogsRetention` | Set log retention | `*` (log group names are dynamic) |

---

## Local Development Setup

### AWS SSO Configuration

SSO is configured in `~/.aws/config`:

```ini
[default]
sso_session = useblocks
sso_account_id = 595319716151
sso_role_name = AdministratorAccess
region = eu-central-1

[sso-session useblocks]
sso_start_url = https://d-99677a8644.awsapps.com/start/
sso_region = eu-central-1
sso_registration_scopes = sso:account:access
```

### Daily Workflow

```bash
# Login (opens browser, authenticate with MFA)
aws sso login

# Session lasts 8 hours (configurable in IAM Identity Center)
# Then use AWS CLI normally
aws s3 ls
aws ec2 describe-images --owners self
```

---

## Running Compliance Fixes

If runs-on updates their stack and resets settings, re-run the compliance workflow:

1. **Via GitHub Actions:**
   - Go to Actions → "Enforce runs-on Compliance" → Run workflow

2. **Locally (for testing):**
   ```bash
   cd deploy/terraform/runs-on-compliance
   # Temporarily change backend to "local" in main.tf
   docker run --rm -v $(pwd):/workspace -v ~/.aws:/root/.aws \
     -w /workspace --entrypoint "" amazon/aws-cli \
     sh -c "yum install -y unzip > /dev/null 2>&1 && \
            curl -s https://releases.hashicorp.com/terraform/1.7.4/terraform_1.7.4_linux_amd64.zip -o tf.zip && \
            unzip -o -q tf.zip && \
            ./terraform init && \
            ./terraform apply -auto-approve"
   ```

---

## Deployment Checklist

Before the OIDC role can be used in CI, deploy the IDP stack:

```bash
cd deploy/sst
yarn install
yarn sst deploy --stage common --region eu-central-1
```

This creates the `github-action-packer-build` IAM role that GitHub Actions will assume.
