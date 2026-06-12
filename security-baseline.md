# AWS Security Baseline — Verification Report

**Date:** 2026-06-12T08:45:12Z
**Engineer:** Eric Borba
**Account ID:** 871939031886
**Region:** us-east-1

---

## Controls Implemented

| Control | Description | Status | Command to Verify |
|---|---|---|---|
| Least-privilege SG | SSH from 91.11.68.2/32 only | ✅ Active | `aws ec2 describe-security-groups --group-ids sg-0b8109cf45839bcc0` |
| VPC Flow Logs | ALL traffic captured to CloudWatch | ✅ Active | `aws ec2 describe-flow-logs` |
| EBS encryption by default | New volumes auto-encrypted | ✅ Enabled | `aws ec2 get-ebs-encryption-by-default` |
| CloudTrail | All API calls logged to S3 | ✅ Logging | `aws cloudtrail get-trail-status --name lab-m8-08-security-trail` |

---

## Resources Created

- Security Group: sg-0b8109cf45839bcc0
- VPC Flow Logs: /aws/vpc/flow-logs (CloudWatch)
- CloudTrail: lab-m8-08-security-trail → s3://cloudtrail-logs-871939031886-1781252774
- EC2 Instance: i-07570fdf08516d946 (54.147.110.55)

---

## Test Results

- SSH connection to EC2 instance: ✅ Successful
- SSH ACCEPT event visible in VPC Flow Logs: ✅ Confirmed
- RunInstances event visible in CloudTrail: ✅ Confirmed
- EBS volume on instance is encrypted: ✅ (default encryption enabled before launch)

---

## Security Design Decisions

| Decision | Rationale |
|---|---|
| SSH restricted to /32 | Least-privilege: no wider than necessary |
| ALL traffic type in Flow Logs | Capture both accepted and rejected traffic for full visibility |
| Multi-region trail | CloudTrail covers all regions, not just the one you're working in |
| 30-day log retention | Balance between cost and forensic investigation windows |
