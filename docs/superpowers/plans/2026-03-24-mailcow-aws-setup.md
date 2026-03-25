# Mailcow Mail Server on AWS — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy a self-hosted Mailcow mail server on AWS for internal employee email at `mail.portal-syncsoft.com`.

**Architecture:** Single EC2 instance (t3.medium, Ubuntu 22.04) running Mailcow Dockerized with all components (Postfix, Dovecot, SOGo, Rspamd, ClamAV, MariaDB, Redis, Nginx). Backup to S3 with Glacier lifecycle (delete after 90 days). DNS managed via Route 53.

**Tech Stack:** AWS (EC2, EBS, Route 53, S3, CloudWatch), Docker, Mailcow Dockerized, Ubuntu 22.04 LTS

**Spec:** `docs/specs/2026-03-24-mailcow-aws-setup-design.md`

**AWS Profile:** `default`
**Region:** `ap-southeast-2`
**Hosted Zone ID:** `Z09305612IEOUCEGO7H9X`
**Domain:** `portal-syncsoft.com`

---

## Phase 1: AWS Pre-requisites (Day 1)

### Task 1: Request AWS Port 25 Unblock

AWS blocks outbound port 25 by default on all EC2 instances. This must be unblocked before the mail server can send mail. This typically takes 1-2 business days to approve.

- [ ] **Step 1: Submit port 25 removal request**

Go to AWS Console → Support → Create Case:
- **Type:** Service limit increase
- **Limit type:** EC2 Email Sending
- **Region:** ap-southeast-2
- **New limit value:** Desired (port 25 unblock)
- **Use case description:**

```
We are setting up a self-hosted mail server (Mailcow) on EC2 for internal
company email. Expected volume: <1,000 emails/day for 20-50 employees.
Domain: portal-syncsoft.com. We will configure SPF, DKIM, DMARC, and
reverse DNS properly.
```

- [ ] **Step 2: Verify request status**

Check AWS Support Console for approval. Expected: approved within 1-2 business days.

> **Note:** Continue with Tasks 2-5 while waiting for approval. Port 25 is only needed for sending mail to external servers.

---

### Task 2: Provision EC2 Instance

- [ ] **Step 1: Create Security Group**

```bash
aws ec2 create-security-group \
  --group-name mailcow-sg \
  --description "Security group for Mailcow mail server" \
  --vpc-id vpc-039cdb25993ce9cf5 \
  --profile default \
  --region ap-southeast-2
```

Save the `GroupId` from output (e.g., `sg-xxxxxxxxx`).

- [ ] **Step 2: Configure Security Group rules**

```bash
SG_ID="sg-xxxxxxxxx"  # Replace with actual GroupId

# SMTP inbound
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 25 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# SMTPS
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 465 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# Submission
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 587 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# IMAP
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 143 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# IMAPS
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 993 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# POP3
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 110 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# POP3S
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 995 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# HTTPS
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 443 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# HTTP (Let's Encrypt)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0 --profile default --region ap-southeast-2

# SSH (admin IP only - replace YOUR_IP)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr YOUR_IP/32 --profile default --region ap-southeast-2
```

- [ ] **Step 3: Find Ubuntu 22.04 LTS AMI**

```bash
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text \
  --profile default \
  --region ap-southeast-2
```

Save the AMI ID (e.g., `ami-xxxxxxxxx`).

- [ ] **Step 4: Create EC2 key pair (if not reusing existing)**

```bash
aws ec2 create-key-pair \
  --key-name mailcow-key \
  --query 'KeyMaterial' \
  --output text \
  --profile default \
  --region ap-southeast-2 > ~/.ssh/mailcow-key.pem

chmod 400 ~/.ssh/mailcow-key.pem
```

Or reuse existing key: `antony.syncsoft-key.pem`

- [ ] **Step 5: Create IAM Role for EC2**

```bash
# Create trust policy
cat > /tmp/ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
  --role-name mailcow-ec2-role \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json \
  --profile default

# Attach S3 backup policy
aws iam put-role-policy \
  --role-name mailcow-ec2-role \
  --policy-name mailcow-s3-backup \
  --profile default \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:PutObject", "s3:GetObject", "s3:ListBucket", "s3:DeleteObject"],
        "Resource": [
          "arn:aws:s3:::portal-syncsoft-mail-backup",
          "arn:aws:s3:::portal-syncsoft-mail-backup/*"
        ]
      }
    ]
  }'

# Attach CloudWatch agent policy
aws iam attach-role-policy \
  --role-name mailcow-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --profile default

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name mailcow-ec2-profile \
  --profile default

aws iam add-role-to-instance-profile \
  --instance-profile-name mailcow-ec2-profile \
  --role-name mailcow-ec2-role \
  --profile default
```

- [ ] **Step 6: Launch EC2 instance**

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxxx \
  --instance-type t3.medium \
  --key-name antony.syncsoft-key \
  --security-group-ids sg-xxxxxxxxx \
  --iam-instance-profile Name=mailcow-ec2-profile \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":100,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mailcow-server}]' \
  --profile default \
  --region ap-southeast-2
```

Save the `InstanceId` from output.

- [ ] **Step 7: Wait for instance to be running**

```bash
aws ec2 wait instance-running --instance-ids i-xxxxxxxxx --profile default --region ap-southeast-2
```

- [ ] **Step 8: Verify instance**

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxxx \
  --query 'Reservations[0].Instances[0].{State:State.Name,IP:PublicIpAddress,Type:InstanceType}' \
  --output table \
  --profile default \
  --region ap-southeast-2
```

Expected: State=running, Type=t3.medium

---

### Task 3: Allocate Elastic IP

- [ ] **Step 1: Allocate Elastic IP**

```bash
aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=mailcow-eip}]' \
  --profile default \
  --region ap-southeast-2
```

Save the `AllocationId` and `PublicIp` from output.

- [ ] **Step 2: Associate Elastic IP with EC2**

```bash
aws ec2 associate-address \
  --instance-id i-xxxxxxxxx \
  --allocation-id eipalloc-xxxxxxxxx \
  --profile default \
  --region ap-southeast-2
```

- [ ] **Step 3: Configure Reverse DNS (PTR record)**

```bash
aws ec2 modify-address-attribute \
  --allocation-id eipalloc-xxxxxxxxx \
  --domain-name mail.portal-syncsoft.com \
  --profile default \
  --region ap-southeast-2
```

> **Note:** This may require AWS approval. Check status with:
> ```bash
> aws ec2 describe-addresses-attribute --allocation-ids eipalloc-xxxxxxxxx --attribute domain-name --profile default --region ap-southeast-2
> ```

---

### Task 4: Configure Route 53 DNS

Replace `52.xx.xx.xx` with the actual Elastic IP from Task 3.

- [ ] **Step 1: Create DNS records**

```bash
EIP="52.xx.xx.xx"  # Replace with actual Elastic IP

aws route53 change-resource-record-sets \
  --hosted-zone-id Z09305612IEOUCEGO7H9X \
  --profile default \
  --change-batch "{
    \"Changes\": [
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"mail.portal-syncsoft.com\",
          \"Type\": \"A\",
          \"TTL\": 300,
          \"ResourceRecords\": [{\"Value\": \"$EIP\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"portal-syncsoft.com\",
          \"Type\": \"MX\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"10 mail.portal-syncsoft.com\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"portal-syncsoft.com\",
          \"Type\": \"TXT\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"\\\"v=spf1 a mx ip4:$EIP ~all\\\"\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"_dmarc.portal-syncsoft.com\",
          \"Type\": \"TXT\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"\\\"v=DMARC1; p=quarantine; rua=mailto:admin@portal-syncsoft.com\\\"\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"autoconfig.portal-syncsoft.com\",
          \"Type\": \"A\",
          \"TTL\": 300,
          \"ResourceRecords\": [{\"Value\": \"$EIP\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"autodiscover.portal-syncsoft.com\",
          \"Type\": \"A\",
          \"TTL\": 300,
          \"ResourceRecords\": [{\"Value\": \"$EIP\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"_autodiscover._tcp.portal-syncsoft.com\",
          \"Type\": \"SRV\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"0 1 443 mail.portal-syncsoft.com\"}]
        }
      }
    ]
  }"
```

- [ ] **Step 2: Verify DNS propagation**

```bash
dig mail.portal-syncsoft.com A +short
dig portal-syncsoft.com MX +short
dig portal-syncsoft.com TXT +short
dig _dmarc.portal-syncsoft.com TXT +short
dig autoconfig.portal-syncsoft.com A +short
dig autodiscover.portal-syncsoft.com A +short
```

Expected: All records return correct values. May take 5-10 minutes to propagate.

---

### Task 5: Create S3 Backup Bucket

- [ ] **Step 1: Create S3 bucket**

```bash
aws s3 mb s3://portal-syncsoft-mail-backup \
  --profile default \
  --region ap-southeast-2
```

- [ ] **Step 2: Enable versioning**

```bash
aws s3api put-bucket-versioning \
  --bucket portal-syncsoft-mail-backup \
  --versioning-configuration Status=Enabled \
  --profile default \
  --region ap-southeast-2
```

- [ ] **Step 3: Configure lifecycle policy (Standard 30d → Glacier, delete after 90d)**

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket portal-syncsoft-mail-backup \
  --profile default \
  --region ap-southeast-2 \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "mail-backup-lifecycle",
        "Status": "Enabled",
        "Filter": {"Prefix": ""},
        "Transitions": [
          {
            "Days": 30,
            "StorageClass": "GLACIER"
          }
        ],
        "Expiration": {
          "Days": 90
        }
      }
    ]
  }'
```

- [ ] **Step 4: Verify**

```bash
aws s3api get-bucket-versioning --bucket portal-syncsoft-mail-backup --profile default --region ap-southeast-2
aws s3api get-bucket-lifecycle-configuration --bucket portal-syncsoft-mail-backup --profile default --region ap-southeast-2
```

Expected: Versioning=Enabled, Lifecycle with Glacier transition at 30 days, expiration at 90 days.

---

## Phase 2: Server Setup (Day 1-2)

### Task 6: SSH & Base Server Configuration

- [ ] **Step 1: SSH into the server**

```bash
ssh -i ~/.ssh/antony.syncsoft-key.pem ubuntu@52.xx.xx.xx
```

- [ ] **Step 2: Set hostname**

```bash
sudo hostnamectl set-hostname mail.portal-syncsoft.com
```

- [ ] **Step 3: Update OS**

```bash
sudo apt update && sudo apt upgrade -y
```

- [ ] **Step 4: Configure 4GB swap**

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

- [ ] **Step 5: Verify swap**

```bash
free -h
```

Expected: Swap total = 4.0G

- [ ] **Step 6: Enable unattended-upgrades**

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

Select "Yes" when prompted.

- [ ] **Step 7: Install AWS CLI v2**

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install -y unzip
unzip awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip
```

- [ ] **Step 8: Verify AWS credentials (via IAM Role)**

```bash
aws sts get-caller-identity
aws s3 ls s3://portal-syncsoft-mail-backup/
```

Expected: Returns the `mailcow-ec2-role` identity. No `aws configure` needed — credentials come from the IAM Instance Profile attached in Task 2.

---

### Task 7: Install Docker & Mailcow

- [ ] **Step 1: Install Docker**

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

- [ ] **Step 2: Log out and back in** (for docker group to take effect)

```bash
exit
ssh -i ~/.ssh/antony.syncsoft-key.pem ubuntu@52.xx.xx.xx
```

- [ ] **Step 3: Verify Docker**

```bash
docker --version
docker compose version
```

Expected: Docker 27.x+ and Docker Compose v2.x+

- [ ] **Step 4: Clone Mailcow**

```bash
cd /opt
sudo git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized
```

- [ ] **Step 5: Generate Mailcow config**

```bash
sudo ./generate_config.sh
```

When prompted:
- **Hostname:** `mail.portal-syncsoft.com`
- **Timezone:** `Asia/Ho_Chi_Minh`

- [ ] **Step 6: Pull and start Mailcow**

```bash
sudo docker compose pull
sudo docker compose up -d
```

This will take 5-10 minutes to download all images and start.

- [ ] **Step 7: Verify all containers are running**

```bash
sudo docker compose ps
```

Expected: All containers should show "Up" status (~15 containers).

- [ ] **Step 8: Check memory usage**

```bash
free -h
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}"
```

Expected: Total memory usage < 3.5GB. If above 3.5GB, ClamAV may cause issues.

> **ClamAV fallback:** If memory is exhausted, disable ClamAV:
> ```bash
> cd /opt/mailcow-dockerized
> echo "SKIP_CLAMD=y" >> mailcow.conf
> sudo docker compose up -d
> ```
> Rspamd still provides spam filtering without ClamAV.

- [ ] **Step 9: Verify web access**

Open browser: `https://mail.portal-syncsoft.com`

Expected: Mailcow login page. Default credentials: `admin` / `moohoo`

**IMPORTANT: Change the admin password immediately!**

---

## Phase 3: Mailcow Configuration (Day 2)

### Task 8: Configure Domain & DKIM

All steps via Mailcow Admin UI at `https://mail.portal-syncsoft.com`

- [ ] **Step 1: Login to Mailcow Admin**

URL: `https://mail.portal-syncsoft.com`
User: `admin`
Password: `moohoo` (change immediately!)

- [ ] **Step 2: Change admin password**

Go to: Access → Password → change to a strong password. Store securely.

- [ ] **Step 3: Add domain**

Go to: Configuration → Mail Setup → Domains → Add domain
- Domain: `portal-syncsoft.com`
- Max mailboxes: 100
- Max aliases: 200
- Default mailbox quota: 2048 MB (2GB per user, adjust as needed)

- [ ] **Step 4: Copy DKIM key**

Go to: Configuration → ARC/DKIM Keys
- Select domain: `portal-syncsoft.com`
- DKIM key length: 2048
- Click "Generate"
- Copy the DKIM TXT record value

- [ ] **Step 5: Add DKIM record to Route 53**

Back on your local machine:

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z09305612IEOUCEGO7H9X \
  --profile default \
  --change-batch "{
    \"Changes\": [
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"dkim._domainkey.portal-syncsoft.com\",
          \"Type\": \"TXT\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"\\\"PASTE_DKIM_KEY_HERE\\\"\"}]
        }
      }
    ]
  }"
```

> **Note:** DKIM keys are long. If the key exceeds 255 characters, split into multiple strings: `"part1" "part2"`

- [ ] **Step 6: Verify DKIM**

```bash
dig dkim._domainkey.portal-syncsoft.com TXT +short
```

Expected: DKIM public key value.

---

### Task 9: Create Mailboxes

Via Mailcow Admin UI:

- [ ] **Step 1: Create admin mailbox**

Go to: Configuration → Mail Setup → Mailboxes → Add mailbox
- Username: `admin`
- Domain: `portal-syncsoft.com`
- Password: (strong password)
- Quota: 4096 MB

- [ ] **Step 2: Create app relay account**

Add mailbox:
- Username: `noreply`
- Domain: `portal-syncsoft.com`
- Password: (generate strong password — save this for app config)
- Quota: 1024 MB

- [ ] **Step 3: Create employee mailboxes**

For each employee, add mailbox:
- Username: (employee name, e.g., `nguyen.van.a`)
- Domain: `portal-syncsoft.com`
- Password: (temporary, employee changes on first login)
- Quota: 2048 MB

- [ ] **Step 4: Enable 2FA for admin**

Go to: Access → Two-Factor Authentication → Enable TOTP
Scan QR code with authenticator app (Google Authenticator, Authy, etc.)

> **Note:** Ask all employees to enable 2FA after first login.

---

### Task 10: Verify fail2ban

- [ ] **Step 1: SSH into server and check fail2ban status**

```bash
sudo docker compose exec netfilter-mailcow fail2ban-client status
```

Expected: Shows jail list including `dovecot`, `postfix`, `sogo`

- [ ] **Step 2: Check ban settings**

```bash
sudo docker compose exec netfilter-mailcow fail2ban-client get dovecot bantime
sudo docker compose exec netfilter-mailcow fail2ban-client get dovecot maxretry
```

Expected: Reasonable ban time (e.g., 3600s) and max retries (e.g., 5)

---

## Phase 4: Backup & Monitoring (Day 2-3)

### Task 11: Configure Automated Backup

- [ ] **Step 1: SSH into server**

```bash
ssh -i ~/.ssh/antony.syncsoft-key.pem ubuntu@52.xx.xx.xx
```

- [ ] **Step 2: Create backup directory**

```bash
sudo mkdir -p /opt/mailcow-backup
```

- [ ] **Step 3: Create backup script**

```bash
sudo tee /opt/mailcow-backup.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

LOG="/var/log/mailcow-backup.log"
echo "$(date): Starting Mailcow backup..." >> $LOG

# Run Mailcow backup
cd /opt/mailcow-dockerized
MAILCOW_BACKUP_LOCATION=/opt/mailcow-backup ./helper-scripts/backup_and_restore.sh backup all >> $LOG 2>&1

echo "$(date): Mailcow backup completed. Syncing to S3..." >> $LOG

# Sync to S3
aws s3 sync /opt/mailcow-backup/ s3://portal-syncsoft-mail-backup/ --delete >> $LOG 2>&1

echo "$(date): S3 sync completed. Cleaning up old local backups..." >> $LOG

# Clean up local backups older than 7 days to prevent disk filling up
find /opt/mailcow-backup/ -type f -mtime +7 -delete >> $LOG 2>&1
find /opt/mailcow-backup/ -type d -empty -delete >> $LOG 2>&1

echo "$(date): Backup complete." >> $LOG
SCRIPT

sudo chmod +x /opt/mailcow-backup.sh
```

- [ ] **Step 4: Setup crontab**

```bash
(sudo crontab -l 2>/dev/null; echo "0 2 * * * /opt/mailcow-backup.sh") | sudo crontab -
```

Verify:
```bash
sudo crontab -l
```

Expected: Shows `0 2 * * * /opt/mailcow-backup.sh`

- [ ] **Step 5: Test backup manually**

```bash
sudo /opt/mailcow-backup.sh
```

Expected: No errors. Check S3:

```bash
aws s3 ls s3://portal-syncsoft-mail-backup/
```

Expected: Backup files present.

- [ ] **Step 6: Test backup restore**

> **WARNING:** Only do this BEFORE go-live or on a separate test instance. This will cause downtime.

```bash
cd /opt/mailcow-dockerized
sudo docker compose down
MAILCOW_BACKUP_LOCATION=/opt/mailcow-backup ./helper-scripts/backup_and_restore.sh restore
sudo docker compose up -d
```

Expected: Mailcow starts successfully with restored data. Verify by logging into webmail.

---

### Task 12: Configure CloudWatch Monitoring

Run from local machine:

- [ ] **Step 1: Install CloudWatch agent on EC2**

SSH into server:

```bash
sudo apt install -y amazon-cloudwatch-agent
```

- [ ] **Step 2: Create CloudWatch agent config**

```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'CONFIG'
{
  "metrics": {
    "namespace": "Mailcow",
    "metrics_collected": {
      "disk": {
        "measurement": ["used_percent"],
        "resources": ["/"],
        "metrics_collection_interval": 300
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 300
      },
      "cpu": {
        "measurement": ["cpu_usage_active"],
        "metrics_collection_interval": 300
      }
    }
  }
}
CONFIG
```

- [ ] **Step 3: Start CloudWatch agent**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s
```

- [ ] **Step 4: Verify CloudWatch agent is running**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

Expected: Status shows `running`.

- [ ] **Step 5: Create SNS topic for alarm notifications**

From local machine:

```bash
aws sns create-topic \
  --name mailcow-alerts \
  --profile default \
  --region ap-southeast-2
```

Save the `TopicArn` from output.

- [ ] **Step 6: Subscribe email to SNS topic**

> **Note:** Use your personal/existing email first (not `admin@portal-syncsoft.com` — that mailbox doesn't exist yet). You can add the Mailcow admin email as a second subscriber after Task 9.

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-southeast-2:650008108447:mailcow-alerts \
  --protocol email \
  --notification-endpoint YOUR_EXISTING_EMAIL@gmail.com \
  --profile default \
  --region ap-southeast-2
```

Check inbox and confirm the subscription email from AWS.

- [ ] **Step 7: Create disk usage alarm**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "mailcow-disk-usage-high" \
  --alarm-description "Disk usage above 80% on mailcow server" \
  --metric-name "disk_used_percent" \
  --namespace "Mailcow" \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=path,Value=/ \
  --alarm-actions arn:aws:sns:ap-southeast-2:650008108447:mailcow-alerts \
  --profile default \
  --region ap-southeast-2
```

---

## Phase 5: Testing & Go-live (Day 3-4)

### Task 13: Test Email Delivery

- [ ] **Step 1: Test send from webmail to Gmail**

1. Login to `https://mail.portal-syncsoft.com` with admin account
2. Compose email to a personal Gmail address
3. Check Gmail inbox — mail should arrive (check spam folder too)

- [ ] **Step 2: Test receive from Gmail to Mailcow**

1. Reply from Gmail to `admin@portal-syncsoft.com`
2. Check Mailcow webmail — mail should arrive

- [ ] **Step 3: Test SPF/DKIM/DMARC**

From Gmail, open the received email → "Show original" → check:
- `SPF: PASS`
- `DKIM: PASS`
- `DMARC: PASS`

If any FAIL, review DNS records.

- [ ] **Step 4: Test with mail-tester.com**

1. Go to `https://www.mail-tester.com`
2. Copy the test email address shown
3. Send an email from Mailcow to that address
4. Check score — aim for 9/10 or 10/10

- [ ] **Step 5: Verify TLS 1.2+ enforcement**

```bash
# This should succeed (TLS 1.2)
openssl s_client -connect mail.portal-syncsoft.com:993 -tls1_2 </dev/null 2>&1 | head -5

# This should fail (TLS 1.1 - insecure)
openssl s_client -connect mail.portal-syncsoft.com:993 -tls1_1 </dev/null 2>&1 | head -5
```

Expected: TLS 1.2 connects successfully, TLS 1.1 fails with handshake error.

- [ ] **Step 6: Test Outlook client**

Configure Outlook with:
- IMAP: `mail.portal-syncsoft.com` port 993 (SSL/TLS)
- SMTP: `mail.portal-syncsoft.com` port 587 (STARTTLS)
- Username: `admin@portal-syncsoft.com`
- Password: (admin password)

Send and receive a test email.

- [ ] **Step 7: Test autodiscover with Thunderbird**

1. Open Thunderbird → Add account
2. Enter: `admin@portal-syncsoft.com` + password
3. Thunderbird should auto-detect server settings

Expected: Auto-detects IMAP 993 and SMTP 587.

- [ ] **Step 8: Test app relay**

From any machine with curl/swaks:

```bash
swaks --to test@gmail.com \
  --from noreply@portal-syncsoft.com \
  --server mail.portal-syncsoft.com \
  --port 587 \
  --tls \
  --auth-user noreply@portal-syncsoft.com \
  --auth-password "APP_PASSWORD_HERE"
```

Or if swaks not available, use Python:

```python
import smtplib
from email.mime.text import MIMEText

msg = MIMEText("Test email from internal app")
msg["Subject"] = "App Relay Test"
msg["From"] = "noreply@portal-syncsoft.com"
msg["To"] = "test@gmail.com"

with smtplib.SMTP("mail.portal-syncsoft.com", 587) as server:
    server.starttls()
    server.login("noreply@portal-syncsoft.com", "APP_PASSWORD_HERE")
    server.send_message(msg)
    print("Email sent successfully")
```

---

### Task 14: Verify rDNS & IP Reputation

- [ ] **Step 1: Verify reverse DNS**

```bash
dig -x 52.xx.xx.xx +short
```

Expected: `mail.portal-syncsoft.com.`

- [ ] **Step 2: Check IP blacklist**

Go to: `https://mxtoolbox.com/blacklists.aspx`
Enter: `52.xx.xx.xx`

Expected: Not listed on any blacklists. New AWS IPs are usually clean.

- [ ] **Step 3: Check MX toolbox full report**

Go to: `https://mxtoolbox.com/SuperTool.aspx`
Enter: `portal-syncsoft.com`
Run: MX Lookup, SPF Check, DKIM Check, DMARC Check

Expected: All checks pass.

---

### Task 15: IP Warm-up & Employee Onboarding

- [ ] **Step 1: IP warm-up plan (Week 1-2)**

| Day | Action |
|-----|--------|
| Day 1-3 | Send only internal emails (between employees) |
| Day 4-7 | Send 10-20 external emails/day (to known contacts) |
| Week 2 | Gradually increase to normal volume |

- [ ] **Step 2: Distribute mail client config to employees**

Send this info to all employees:

```
=== Mail Configuration ===

Webmail: https://mail.portal-syncsoft.com

For Outlook / Thunderbird / Mobile:
- Email: your.name@portal-syncsoft.com
- Password: (provided separately)
- IMAP Server: mail.portal-syncsoft.com (Port 993, SSL/TLS)
- SMTP Server: mail.portal-syncsoft.com (Port 587, STARTTLS)

Please enable Two-Factor Authentication (2FA) after first login:
  Login → Access → Two-Factor Authentication → Enable TOTP
```

- [ ] **Step 3: DMARC ramp-up schedule**

| When | Action | Command |
|------|--------|---------|
| Day 1 (done) | `p=quarantine` | Already configured |
| After 1 month | `p=reject` | Update DNS TXT `_dmarc.portal-syncsoft.com` |

DMARC update command (run when ready to enforce `reject`):

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z09305612IEOUCEGO7H9X \
  --profile default \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "_dmarc.portal-syncsoft.com",
        "Type": "TXT",
        "TTL": 3600,
        "ResourceRecords": [{"Value": "\"v=DMARC1; p=reject; rua=mailto:admin@portal-syncsoft.com\""}]
      }
    }]
  }'
```

---

## Summary

| Phase | Tasks | Timeline |
|-------|-------|----------|
| Phase 1: AWS Pre-requisites | Tasks 1-5 | Day 1 (parallel, wait for port 25 approval) |
| Phase 2: Server Setup | Tasks 6-7 | Day 1-2 |
| Phase 3: Mailcow Config | Tasks 8-10 | Day 2 |
| Phase 4: Backup & Monitoring | Tasks 11-12 | Day 2-3 |
| Phase 5: Testing & Go-live | Tasks 13-15 | Day 3-4 |

**Total estimated time:** 3-4 days (including port 25 approval wait)
