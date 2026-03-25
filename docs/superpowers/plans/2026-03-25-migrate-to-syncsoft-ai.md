# Migrate Mailcow from portal-syncsoft.com to syncsoft.ai

**Goal:** Replace `portal-syncsoft.com` with `syncsoft.ai` as the mail domain on existing Mailcow server.

**Server:** EC2 `i-0fec63c97b3fa9362`, IP `15.135.14.30`
**Old domain:** `portal-syncsoft.com` (to be removed)
**New domain:** `syncsoft.ai`

**Pre-requisite:** Move `syncsoft.ai` DNS to Route 53 (NS transfer from GoDaddy)

---

## Phase 1: Move syncsoft.ai DNS to Route 53 (Day 1)

### Task 1: Create Hosted Zone in Route 53

- [ ] **Step 1: Create hosted zone**

```bash
aws route53 create-hosted-zone \
  --name syncsoft.ai \
  --caller-reference "syncsoft-ai-$(date +%s)" \
  --profile default \
  --query 'HostedZone.Id' \
  --output text
```

Save the Hosted Zone ID.

- [ ] **Step 2: Get NS records from Route 53**

```bash
aws route53 get-hosted-zone \
  --id HOSTED_ZONE_ID \
  --profile default \
  --query 'DelegationSet.NameServers'
```

Save the 4 NS records (e.g., `ns-xxx.awsdns-xx.xxx`).

- [ ] **Step 3: Copy existing DNS records from GoDaddy**

Before changing NS, note down all existing records on GoDaddy (A record `216.198.79.1`, TXT google-site-verification, etc.) and recreate them in Route 53.

- [ ] **Step 4: Update NS on GoDaddy**

In GoDaddy DNS management for `syncsoft.ai`:
- Change nameservers to the 4 AWS NS records from Step 2
- Remove old GoDaddy nameservers (`ns45.domaincontrol.com`, `ns46.domaincontrol.com`)

> **Note:** NS propagation takes 24-48 hours. Wait for propagation before proceeding.

- [ ] **Step 5: Verify NS propagation**

```bash
dig syncsoft.ai NS +short
```

Expected: Shows AWS nameservers.

---

## Phase 2: Configure Mail DNS Records (Day 1-2)

### Task 2: Create DNS Records for Mail

- [ ] **Step 1: Create all mail DNS records**

```bash
ZONE_ID="HOSTED_ZONE_ID"  # Replace with actual Zone ID
EIP="15.135.14.30"

aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --profile default \
  --change-batch "{
    \"Changes\": [
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"mail.syncsoft.ai\",
          \"Type\": \"A\",
          \"TTL\": 300,
          \"ResourceRecords\": [{\"Value\": \"$EIP\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"syncsoft.ai\",
          \"Type\": \"MX\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"10 mail.syncsoft.ai\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"syncsoft.ai\",
          \"Type\": \"TXT\",
          \"TTL\": 3600,
          \"ResourceRecords\": [
            {\"Value\": \"\\\"v=spf1 a mx ip4:$EIP ~all\\\"\"},
            {\"Value\": \"\\\"google-site-verification=nLZW3kJ_22i53y3uCGuaDUTLVZs8gP3bdyvo_zzytes\\\"\"}
          ]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"_dmarc.syncsoft.ai\",
          \"Type\": \"TXT\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"\\\"v=DMARC1; p=quarantine; rua=mailto:admin@syncsoft.ai\\\"\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"autoconfig.syncsoft.ai\",
          \"Type\": \"A\",
          \"TTL\": 300,
          \"ResourceRecords\": [{\"Value\": \"$EIP\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"autodiscover.syncsoft.ai\",
          \"Type\": \"A\",
          \"TTL\": 300,
          \"ResourceRecords\": [{\"Value\": \"$EIP\"}]
        }
      },
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"_autodiscover._tcp.syncsoft.ai\",
          \"Type\": \"SRV\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"0 1 443 mail.syncsoft.ai\"}]
        }
      }
    ]
  }"
```

> **Note:** SPF TXT record includes both SPF and existing google-site-verification in the same record set.

- [ ] **Step 2: Verify DNS propagation**

```bash
dig mail.syncsoft.ai A +short
dig syncsoft.ai MX +short
dig syncsoft.ai TXT +short
dig _dmarc.syncsoft.ai TXT +short
```

---

## Phase 3: Configure Mailcow (Day 2)

### Task 3: Update Mailcow Hostname

- [ ] **Step 1: Update hostname on server**

```bash
ssh -i ~/.ssh/antony.syncsoft-key.pem ubuntu@15.135.14.30
sudo hostnamectl set-hostname mail.syncsoft.ai
```

- [ ] **Step 2: Update Mailcow config**

```bash
cd /opt/mailcow-dockerized
sudo sed -i 's/MAILCOW_HOSTNAME=mail.portal-syncsoft.com/MAILCOW_HOSTNAME=mail.syncsoft.ai/' mailcow.conf
```

- [ ] **Step 3: Restart Mailcow**

```bash
cd /opt/mailcow-dockerized
sudo docker compose down
sudo docker compose up -d
```

Wait for all containers to be running:
```bash
sudo docker compose ps
```

### Task 4: Add Domain & DKIM in Mailcow

- [ ] **Step 1: Login to Mailcow Admin**

URL: `https://mail.syncsoft.ai/admin` (after SSL cert is issued)

> **Note:** SSL cert will auto-renew via Let's Encrypt for the new hostname. May take a few minutes. If cannot access via new hostname, use IP temporarily: `https://15.135.14.30/admin`

- [ ] **Step 2: Add domain `syncsoft.ai`**

Go to: E-Mail → Configuration → Domains → Add domain
- Domain: `syncsoft.ai`
- Max mailboxes: 100
- Max aliases: 200
- Default mailbox quota: 2048 MB
- DKIM key length: 2048

- [ ] **Step 3: Copy DKIM key**

Go to: E-Mail → Configuration → click DNS button on `syncsoft.ai`
Copy the DKIM TXT record value.

- [ ] **Step 4: Add DKIM record to Route 53**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id HOSTED_ZONE_ID \
  --profile default \
  --change-batch "{
    \"Changes\": [
      {
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"dkim._domainkey.syncsoft.ai\",
          \"Type\": \"TXT\",
          \"TTL\": 3600,
          \"ResourceRecords\": [{\"Value\": \"\\\"PASTE_DKIM_KEY_HERE\\\"\"}]
        }
      }
    ]
  }"
```

- [ ] **Step 5: Verify DKIM**

```bash
dig dkim._domainkey.syncsoft.ai TXT +short
```

### Task 5: Create Mailboxes

- [ ] **Step 1: Create admin mailbox**

In Mailcow UI → E-Mail → Configuration → Mailboxes → Add mailbox
- Username: `admin`
- Domain: `syncsoft.ai`
- Password: (strong password)
- Quota: 4096 MB

- [ ] **Step 2: Create noreply mailbox**

- Username: `noreply`
- Domain: `syncsoft.ai`
- Password: (strong password, save for app config)
- Quota: 1024 MB

- [ ] **Step 3: Create employee mailboxes as needed**

### Task 6: Remove Old Domain

- [ ] **Step 1: Delete old mailboxes on portal-syncsoft.com**

In Mailcow UI → E-Mail → Configuration → Mailboxes
- Remove `admin@portal-syncsoft.com`
- Remove `noreply@portal-syncsoft.com`

- [ ] **Step 2: Delete old domain from Mailcow**

In Mailcow UI → E-Mail → Configuration → Domains
- Remove `portal-syncsoft.com`

- [ ] **Step 3: Clean up old DNS records on Route 53**

```bash
# Delete mail-related records from portal-syncsoft.com hosted zone
aws route53 change-resource-record-sets \
  --hosted-zone-id Z09305612IEOUCEGO7H9X \
  --profile default \
  --change-batch '{
    "Changes": [
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "mail.portal-syncsoft.com", "Type": "A", "TTL": 300, "ResourceRecords": [{"Value": "15.135.14.30"}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "portal-syncsoft.com", "Type": "MX", "TTL": 3600, "ResourceRecords": [{"Value": "10 mail.portal-syncsoft.com"}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "autoconfig.portal-syncsoft.com", "Type": "A", "TTL": 300, "ResourceRecords": [{"Value": "15.135.14.30"}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "autodiscover.portal-syncsoft.com", "Type": "A", "TTL": 300, "ResourceRecords": [{"Value": "15.135.14.30"}]}}
    ]
  }'
```

---

## Phase 4: Verify & Test (Day 2)

### Task 7: Test Email

- [ ] **Step 1: Send test email from webmail**

Login to `https://mail.syncsoft.ai/SOGo` with `admin@syncsoft.ai`
Send email to personal Gmail.

- [ ] **Step 2: Check SPF/DKIM/DMARC**

In Gmail → Show original → verify all PASS.

- [ ] **Step 3: Test with mail-tester.com**

Aim for 9/10 or 10/10.

- [ ] **Step 4: Test receive**

Reply from Gmail to `admin@syncsoft.ai` — verify it arrives in Mailcow.

- [ ] **Step 5: Update SNS alert subscriber**

After mailboxes are working, add `admin@syncsoft.ai` to SNS topic:

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-southeast-2:650008108447:mailcow-alerts \
  --protocol email \
  --notification-endpoint admin@syncsoft.ai \
  --profile default \
  --region ap-southeast-2
```

---

## Summary

| Phase | Tasks | Timeline |
|-------|-------|----------|
| Phase 1: Move DNS to Route 53 | Task 1 | Day 1 (wait NS propagation) |
| Phase 2: Create mail DNS records | Task 2 | Day 1-2 |
| Phase 3: Configure Mailcow | Tasks 3-6 | Day 2 |
| Phase 4: Verify & Test | Task 7 | Day 2 |

**Total estimated time:** 1-2 days (including NS propagation wait)
**Downtime:** Minimal — old domain stops working when removed, new domain works immediately after DNS propagation.
