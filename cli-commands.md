# Lab M8.08 — CLI Commands Log

All commands executed chronologically during the lab.

---

## Step 1: Create a Least-Privilege Security Group

```bash
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "Your IP: $MY_IP"

VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)
echo "Default VPC: $VPC_ID"

SG_ID=$(aws ec2 create-security-group \
  --group-name "lab-m8-08-baseline-sg" \
  --description "Security baseline lab - SSH from my IP only" \
  --vpc-id $VPC_ID \
  --query "GroupId" \
  --output text)
echo "Security Group: $SG_ID"

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr "$MY_IP/32"

aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --query "SecurityGroups[0].IpPermissions"
```

---

## Step 2: Enable VPC Flow Logs

```bash
aws logs create-log-group \
  --log-group-name "/aws/vpc/flow-logs"

aws logs put-retention-policy \
  --log-group-name "/aws/vpc/flow-logs" \
  --retention-in-days 30

cat > vpc-flow-logs-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "vpc-flow-logs.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
EOF

cat > vpc-flow-logs-permissions.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents",
      "logs:DescribeLogGroups",
      "logs:DescribeLogStreams"
    ],
    "Resource": "*"
  }]
}
EOF

FLOW_LOGS_ROLE_ARN=$(aws iam create-role \
  --role-name "lab-m8-08-vpc-flow-logs-role" \
  --assume-role-policy-document file://vpc-flow-logs-trust.json \
  --query "Role.Arn" \
  --output text)
echo "Role ARN: $FLOW_LOGS_ROLE_ARN"

aws iam put-role-policy \
  --role-name "lab-m8-08-vpc-flow-logs-role" \
  --policy-name "vpc-flow-logs-cloudwatch" \
  --policy-document file://vpc-flow-logs-permissions.json

LOG_GROUP_ARN=$(aws logs describe-log-groups \
  --log-group-name-prefix "/aws/vpc/flow-logs" \
  --query "logGroups[0].arn" \
  --output text)
echo "Log Group ARN: $LOG_GROUP_ARN"

aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-destination $LOG_GROUP_ARN \
  --deliver-logs-permission-arn $FLOW_LOGS_ROLE_ARN

aws ec2 describe-flow-logs \
  --filter "Name=resource-id,Values=$VPC_ID" \
  --query "FlowLogs[0].{FlowLogStatus:FlowLogStatus,LogDestination:LogDestination}"
```

---

## Step 3: Enable EBS Encryption by Default

```bash
aws ec2 enable-ebs-encryption-by-default

aws ec2 get-ebs-encryption-by-default
```

---

## Step 4: Enable CloudTrail

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TRAIL_BUCKET="cloudtrail-logs-$ACCOUNT_ID-$(date +%s)"
echo "Trail bucket: $TRAIL_BUCKET"

aws s3api create-bucket \
  --bucket $TRAIL_BUCKET \
  --region us-east-1

aws s3api put-bucket-policy \
  --bucket $TRAIL_BUCKET \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Sid\": \"AWSCloudTrailAclCheck\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"Service\": \"cloudtrail.amazonaws.com\"},
        \"Action\": \"s3:GetBucketAcl\",
        \"Resource\": \"arn:aws:s3:::$TRAIL_BUCKET\"
      },
      {
        \"Sid\": \"AWSCloudTrailWrite\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"Service\": \"cloudtrail.amazonaws.com\"},
        \"Action\": \"s3:PutObject\",
        \"Resource\": \"arn:aws:s3:::$TRAIL_BUCKET/AWSLogs/$ACCOUNT_ID/*\",
        \"Condition\": {
          \"StringEquals\": {
            \"s3:x-amz-acl\": \"bucket-owner-full-control\"
          }
        }
      }
    ]
  }"

aws cloudtrail create-trail \
  --name "lab-m8-08-security-trail" \
  --s3-bucket-name $TRAIL_BUCKET \
  --include-global-service-events \
  --is-multi-region-trail

aws cloudtrail start-logging \
  --name "lab-m8-08-security-trail"

aws cloudtrail get-trail-status \
  --name "lab-m8-08-security-trail" \
  --query "{IsLogging:IsLogging,LatestDeliveryTime:LatestDeliveryTime}"
```

---

## Step 5: Launch EC2 Instance to Test the Baseline

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)
echo "AMI: $AMI_ID"

aws ec2 create-key-pair \
  --key-name "lab-m8-08-key" \
  --query "KeyMaterial" \
  --output text > lab-m8-08-key.pem
chmod 400 lab-m8-08-key.pem

SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" \
  --output text)
echo "Subnet: $SUBNET_ID"

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name "lab-m8-08-key" \
  --security-group-ids $SG_ID \
  --subnet-id $SUBNET_ID \
  --associate-public-ip-address \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-m8-08-baseline}]' \
  --query "Instances[0].InstanceId" \
  --output text)
echo "Instance: $INSTANCE_ID"

aws ec2 wait instance-running --instance-ids $INSTANCE_ID

PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)
echo "Public IP: $PUBLIC_IP"

ssh -i lab-m8-08-key.pem ec2-user@$PUBLIC_IP
```

---

## Step 6: Verify Logging

```bash
aws logs describe-log-streams \
  --log-group-name "/aws/vpc/flow-logs" \
  --query "logStreams[*].logStreamName" \
  --output table

aws logs filter-log-events \
  --log-group-name "/aws/vpc/flow-logs" \
  --filter-pattern "91.11.68.2" \
  --limit 10 \
  --query "events[*].message" \
  --output text

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --max-results 5 \
  --query "Events[*].{Time:EventTime,Who:Username,Event:EventName}" \
  --output table
```

---

## Cleanup

```bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID

aws ec2 delete-key-pair --key-name "lab-m8-08-key"
rm -f lab-m8-08-key.pem

aws ec2 delete-security-group --group-id $SG_ID

aws cloudtrail stop-logging --name "lab-m8-08-security-trail"
aws cloudtrail delete-trail --name "lab-m8-08-security-trail"

aws s3 rm s3://$TRAIL_BUCKET --recursive
aws s3api delete-bucket --bucket $TRAIL_BUCKET

FLOW_LOG_ID=$(aws ec2 describe-flow-logs \
  --filter "Name=resource-id,Values=$VPC_ID" \
  --query "FlowLogs[0].FlowLogId" \
  --output text)
aws ec2 delete-flow-logs --flow-log-ids $FLOW_LOG_ID

aws logs delete-log-group --log-group-name "/aws/vpc/flow-logs"

aws iam delete-role-policy \
  --role-name "lab-m8-08-vpc-flow-logs-role" \
  --policy-name "vpc-flow-logs-cloudwatch"
aws iam delete-role --role-name "lab-m8-08-vpc-flow-logs-role"

echo "Cleanup complete."
```
