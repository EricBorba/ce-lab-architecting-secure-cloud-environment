# Lab M8.08 — Architecting a Secure Cloud Environment

![AWS EC2](https://img.shields.io/badge/AWS-EC2%20t3.micro-FF9900?logo=amazonec2&logoColor=white)
![AWS IAM](https://img.shields.io/badge/AWS-IAM-DD344C?logo=amazonaws&logoColor=white)
![Amazon CloudWatch](https://img.shields.io/badge/AWS-CloudWatch-FF4F8B?logo=amazonaws&logoColor=white)
![Amazon S3](https://img.shields.io/badge/AWS-S3-FF9900?logo=amazons3&logoColor=white)
![AWS CloudTrail](https://img.shields.io/badge/AWS-CloudTrail-FF9900?logo=amazonaws&logoColor=white)
![Amazon VPC](https://img.shields.io/badge/AWS-VPC%20Flow%20Logs-8C4FFF?logo=amazonaws&logoColor=white)
![Amazon EBS](https://img.shields.io/badge/AWS-EBS%20Encryption-FF9900?logo=amazonaws&logoColor=white)

## Overview

Security baseline implementation on AWS using CLI only. Configures four foundational security controls before any workloads are deployed: least-privilege security groups, VPC Flow Logs, EBS default encryption, and CloudTrail.

## Controls Implemented

| Control | Description |
|---|---|
| Least-privilege Security Group | Deny-all inbound except SSH from engineer's IP only (/32) |
| VPC Flow Logs | All network traffic captured to CloudWatch Logs (30-day retention) |
| EBS Default Encryption | Account-level encryption for all new EBS volumes |
| CloudTrail | Multi-region trail logging all API calls to S3 |

## Repository Structure

```
.
├── README.md
├── security-baseline.md            ← compliance evidence document
├── vpc-flow-logs-trust.json        ← IAM trust policy for VPC Flow Logs role
├── vpc-flow-logs-permissions.json  ← IAM permissions policy for VPC Flow Logs role
└── screenshots/
```

## Verification Results

- Security group: SSH port 22 open from engineer IP only (0 other inbound rules)
- VPC Flow Logs: `FlowLogStatus = ACTIVE`, delivering to `/aws/vpc/flow-logs`
- EBS encryption: `EbsEncryptionByDefault = true`
- CloudTrail: `IsLogging = true`, multi-region, S3 delivery confirmed
- SSH connection to EC2 test instance: confirmed
- Flow log shows `ACCEPT` entry for port 22 from engineer IP
- CloudTrail shows `RunInstances` event with correct username

## Screenshots

| # | File | Description |
|---|---|---|
| 1 | [01-security-group-created-ssh-rule-verified.png](screenshots/01-security-group-created-ssh-rule-verified.png) | Security group created with SSH rule scoped to engineer IP only (/32) — verified via `describe-security-groups` |
| 2 | [02-cloudwatch-log-group-created-retention-set.png](screenshots/02-cloudwatch-log-group-created-retention-set.png) | CloudWatch log group `/aws/vpc/flow-logs` created with 30-day retention policy |
| 3 | [03-vpc-flow-logs-iam-policy-files-created.png](screenshots/03-vpc-flow-logs-iam-policy-files-created.png) | IAM trust and permissions policy JSON files created for the VPC Flow Logs delivery role |
| 4 | [04-iam-role-created-for-vpc-flow-logs.png](screenshots/04-iam-role-created-for-vpc-flow-logs.png) | IAM role `lab-m8-08-vpc-flow-logs-role` created with the trust policy |
| 5 | [05-iam-inline-policy-attached-to-flow-logs-role.png](screenshots/05-iam-inline-policy-attached-to-flow-logs-role.png) | Inline CloudWatch Logs permissions policy attached to the flow logs IAM role |
| 6 | [06-vpc-flow-logs-active.png](screenshots/06-vpc-flow-logs-active.png) | VPC Flow Logs enabled on the default VPC — status `ACTIVE`, destination confirmed |
| 7 | [07-ebs-encryption-by-default-enabled.png](screenshots/07-ebs-encryption-by-default-enabled.png) | EBS default encryption enabled and verified: `EbsEncryptionByDefault = true` |
| 8 | [08-cloudtrail-s3-bucket-trail-created-logging.png](screenshots/08-cloudtrail-s3-bucket-trail-created-logging.png) | S3 bucket created, bucket policy attached, CloudTrail trail created and logging started (`IsLogging = true`) |
| 9 | [09-ami-lookup-keypair-created.png](screenshots/09-ami-lookup-keypair-created.png) | Latest Amazon Linux 2 AMI resolved and SSH key pair created |
| 10 | [10-ec2-instance-launched.png](screenshots/10-ec2-instance-launched.png) | EC2 instance launched in the correct subnet with the baseline security group |
| 11 | [11-ec2-instance-running-public-ip.png](screenshots/11-ec2-instance-running-public-ip.png) | Instance in running state with public IP assigned |
| 12 | [12-ssh-connection-successful.png](screenshots/12-ssh-connection-successful.png) | SSH connection to EC2 instance confirmed — hostname, uptime, and outbound internet verified |
| 13 | [13-flow-logs-ssh-accept-cloudtrail-runinstances.png](screenshots/13-flow-logs-ssh-accept-cloudtrail-runinstances.png) | VPC Flow Logs showing SSH `ACCEPT` event from engineer IP; CloudTrail `RunInstances` event captured with correct username |
