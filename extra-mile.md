# Lab M8.08 — Extra Mile Tutorial

Three optional security hardening exercises that extend the baseline established in the main lab.

---

## Option A: Block Public Access at the Account Level

### Why it matters

Bucket-level public access settings can be misconfigured by any developer. Account-level block public access acts as a guardrail — it prevents anyone in the account from ever making a bucket or object public, regardless of bucket policy or ACL. This is a mandatory control in most enterprise AWS environments.

### Prerequisites

- AWS account with `s3:PutAccountPublicAccessBlock` permission
- The `ACCOUNT_ID` variable set: `ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)`

### Steps

**1. Enable account-level block public access:**

```bash
aws s3control put-public-access-block \
  --account-id $ACCOUNT_ID \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**2. Verify the setting is active:**

```bash
aws s3control get-public-access-block --account-id $ACCOUNT_ID
```

Expected output:
```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
```

**3. Test that it works — try creating a public bucket:**

```bash
TEST_BUCKET="public-access-test-$ACCOUNT_ID"

aws s3api create-bucket --bucket $TEST_BUCKET --region us-east-1

aws s3api put-bucket-acl \
  --bucket $TEST_BUCKET \
  --acl public-read
```

Expected: the `put-bucket-acl` call returns an `AccessDenied` or `InvalidBucketAclWithBlockPublicAccessError`. This confirms the control is working.

**4. Cleanup the test bucket:**

```bash
aws s3api delete-bucket --bucket $TEST_BUCKET
```

### Why this matters in a team environment

Without this control, a single developer mistake — accidentally setting `--acl public-read` on an `aws s3 cp` command — can expose sensitive data publicly. Account-level block public access eliminates this entire class of misconfiguration regardless of who runs what command.

---

## Option B: CloudTrail Log File Validation

### Why it matters

CloudTrail delivers log files to S3, but by default nothing detects if those files are deleted or tampered with after delivery. Log file validation creates a signed digest file every hour that cryptographically chains all log files together. If an attacker deletes or modifies a log file, the validation check will fail — making this essential for SOC 2 and PCI-DSS compliance.

### Prerequisites

- The trail `lab-m8-08-security-trail` must be active
- The variables `ACCOUNT_ID`, `REGION`, and `TRAIL_BUCKET` must be set

### Steps

**1. Enable log file validation on the existing trail:**

```bash
aws cloudtrail update-trail \
  --name "lab-m8-08-security-trail" \
  --enable-log-file-validation
```

Expected output includes `"LogFileValidationEnabled": true`.

**2. Wait at least one hour** for CloudTrail to deliver a digest file. Digests are created hourly.

**3. Find the latest digest file:**

```bash
DIGEST_PREFIX="AWSLogs/$ACCOUNT_ID/CloudTrail-Digest/$REGION"

LATEST_DIGEST=$(aws s3 ls s3://$TRAIL_BUCKET/$DIGEST_PREFIX/ \
  --recursive \
  | sort | tail -1 | awk '{print $4}')

echo "Latest digest: $LATEST_DIGEST"
```

**4. Validate the logs:**

```bash
aws cloudtrail validate-logs \
  --trail-arn "arn:aws:cloudtrail:$REGION:$ACCOUNT_ID:trail/lab-m8-08-security-trail" \
  --start-time "$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null \
    || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)"
```

Expected output:
```
Validating log files for trail arn:aws:cloudtrail:...
No invalid log files found
```

If a log file had been tampered with or deleted, this command would report it with the specific file name and what failed.

### Why tamper-evident logging matters for compliance

- **SOC 2**: Requires evidence that audit logs cannot be altered by insiders. Digest validation provides a cryptographic proof chain.
- **PCI-DSS Requirement 10.5**: Log data must be protected from modifications. CloudTrail digest files stored in S3 with bucket versioning and MFA delete satisfy this requirement.

---

## Option C: GuardDuty Threat Detection

### Why it matters

GuardDuty is AWS's managed threat detection service. It continuously analyzes CloudTrail, VPC Flow Logs, and DNS logs using machine learning and threat intelligence feeds to detect things like: compromised credentials, crypto mining, port scanning, and data exfiltration — without any agents to install.

### Prerequisites

- AWS account with `guardduty:CreateDetector` permission
- No existing GuardDuty detector in the region (check with `aws guardduty list-detectors`)

### Steps

**1. Enable GuardDuty:**

```bash
DETECTOR_ID=$(aws guardduty create-detector \
  --enable \
  --query "DetectorId" \
  --output text)
echo "GuardDuty detector: $DETECTOR_ID"
```

**2. Check for initial findings** (usually empty on a new account):

```bash
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --query "FindingIds" \
  --output text
```

**3. Generate a simulated finding to test it:**

```bash
aws guardduty create-sample-findings \
  --detector-id $DETECTOR_ID \
  --finding-types "Recon:EC2/PortProbeUnprotectedPort"
```

**4. Wait one minute, then list the findings:**

```bash
sleep 60
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --output table
```

**5. Get details on the simulated finding:**

```bash
FINDING_ID=$(aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --query "FindingIds[0]" \
  --output text)

aws guardduty get-findings \
  --detector-id $DETECTOR_ID \
  --finding-ids $FINDING_ID \
  --query "Findings[0].{Title:Title,Severity:Severity,Type:Type,Description:Description}"
```

### Understanding `Recon:EC2/PortProbeUnprotectedPort`

This finding means an external IP was probing ports on an EC2 instance that has no security group blocking the traffic. The "Recon" prefix indicates it's a reconnaissance activity — an attacker scanning for open ports before launching an attack.

**Remediation steps you would take in production:**
1. Identify the target instance from the finding details
2. Review the instance's security group — ensure no ports are open to `0.0.0.0/0` unless explicitly required
3. Check VPC Flow Logs for the source IP to understand the scope of the probe
4. Add the source IP to a WAF IP block list or Network ACL deny rule if the scanning persists
5. If the instance runs a public service (e.g., HTTPS), use AWS Shield for DDoS protection

### Cleanup

```bash
aws guardduty delete-detector --detector-id $DETECTOR_ID
echo "GuardDuty detector deleted."
```

> **Note:** GuardDuty has a 30-day free trial per account per region. After the trial, it charges based on the volume of CloudTrail events and VPC Flow Log data analyzed. Always delete the detector after testing if you don't intend to use it in production.
