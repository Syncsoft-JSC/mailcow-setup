# Migrate Mailcow from portal-syncsoft.com to syncsoft.ai

**Goal:** Replace `portal-syncsoft.com` with `syncsoft.ai` as the mail domain on existing Mailcow server.

**Server:** EC2 `i-0fec63c97b3fa9362`, IP `15.135.14.30`
**Old domain:** `portal-syncsoft.com` (to be removed)
**New domain:** `syncsoft.ai`
**Old Hosted Zone ID:** `Z09305612IEOUCEGO7H9X` (portal-syncsoft.com)
**New Hosted Zone ID:** `Z075035939H0BYDEY4QBK` (syncsoft.ai)

---

## Phase 1: Move syncsoft.ai DNS to Route 53 — DONE

NS already transferred. Hosted Zone `Z075035939H0BYDEY4QBK` active with AWS nameservers.

---

## Phase 2: Configure Mail DNS Records

### Task 2: Create DNS Records for Mail

- [ ] **Step 1: Restore google-site-verification TXT record (lost during NS transfer)**

- [ ] **Step 2: Create all mail DNS records**

```bash
EIP="15.135.14.30"

aws route53 change-resource-record-sets \
  --hosted-zone-id Z075035939H0BYDEY4QBK \
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

- [ ] **Step 3: Verify DNS propagation**

```bash
dig mail.syncsoft.ai A +short
dig syncsoft.ai MX +short
dig syncsoft.ai TXT +short
dig _dmarc.syncsoft.ai TXT +short
dig autoconfig.syncsoft.ai A +short
dig autodiscover.syncsoft.ai A +short
```

---

## Phase 3: Configure Mailcow

### Task 3: Update Mailcow Hostname & Config

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

- [ ] **Step 3: Update ADDITIONAL_SAN if present**

```bash
# Check if ADDITIONAL_SAN exists and add new domain
grep ADDITIONAL_SAN /opt/mailcow-dockerized/mailcow.conf
# If it exists, add mail.syncsoft.ai to the list
```

- [ ] **Step 4: Restart Mailcow**

```bash
cd /opt/mailcow-dockerized
sudo docker compose down
sudo docker compose up -d
```

Wait for all containers to be running:
```bash
sudo docker compose ps
```

> **Note:** SSL cert will auto-renew via Let's Encrypt for the new hostname. May take a few minutes.

### Task 4: Add Domain & DKIM in Mailcow (Manual — UI)

- [ ] **Step 1: Login to Mailcow Admin**

URL: `https://mail.syncsoft.ai/admin` or `https://15.135.14.30/admin`

- [ ] **Step 2: Add domain `syncsoft.ai`**

Go to: E-Mail → Configuration → Domains → Add domain
- Domain: `syncsoft.ai`
- Max mailboxes: 100
- Max aliases: 200
- Default mailbox quota: 2048 MB
- DKIM key length: 2048

- [ ] **Step 3: Copy DKIM key**

Go to: E-Mail → Configuration → click **DNS** button on `syncsoft.ai`
Copy the DKIM TXT record value → paste to Claude.

- [ ] **Step 4: Add DKIM record to Route 53**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z075035939H0BYDEY4QBK \
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

### Task 5: Create Mailboxes (Manual — UI)

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

### Task 6: Update rDNS (PTR record)

- [ ] **Step 1: Update rDNS to point to new hostname**

```bash
aws ec2 modify-address-attribute \
  --allocation-id eipalloc-0da63ebf65930bac5 \
  --domain-name mail.syncsoft.ai \
  --profile default \
  --region ap-southeast-2
```

- [ ] **Step 2: Verify rDNS update**

```bash
aws ec2 describe-addresses-attribute \
  --allocation-ids eipalloc-0da63ebf65930bac5 \
  --attribute domain-name \
  --profile default \
  --region ap-southeast-2
```

Expected: `mail.syncsoft.ai` with status PENDING or completed.

- [ ] **Step 3: Verify PTR record**

```bash
dig -x 15.135.14.30 +short
```

Expected: `mail.syncsoft.ai.`

### Task 7: Remove Old Domain

- [ ] **Step 1: Delete old mailboxes** (Manual — UI)

In Mailcow UI → E-Mail → Configuration → Mailboxes
- Remove `admin@portal-syncsoft.com`
- Remove `noreply@portal-syncsoft.com`

- [ ] **Step 2: Delete old domain from Mailcow** (Manual — UI)

In Mailcow UI → E-Mail → Configuration → Domains
- Remove `portal-syncsoft.com`

- [ ] **Step 3: Clean up ALL old mail DNS records on Route 53**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z09305612IEOUCEGO7H9X \
  --profile default \
  --change-batch '{
    "Changes": [
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "mail.portal-syncsoft.com.", "Type": "A", "TTL": 300, "ResourceRecords": [{"Value": "15.135.14.30"}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "portal-syncsoft.com.", "Type": "MX", "TTL": 3600, "ResourceRecords": [{"Value": "10 mail.portal-syncsoft.com"}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "portal-syncsoft.com.", "Type": "TXT", "TTL": 3600, "ResourceRecords": [{"Value": "\"v=spf1 a mx ip4:15.135.14.30 ~all\""}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "_dmarc.portal-syncsoft.com.", "Type": "TXT", "TTL": 3600, "ResourceRecords": [{"Value": "\"v=DMARC1; p=quarantine; rua=mailto:admin@portal-syncsoft.com\""}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "dkim._domainkey.portal-syncsoft.com.", "Type": "TXT", "TTL": 3600, "ResourceRecords": [{"Value": "\"v=DKIM1;k=rsa;t=s;s=email;p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAprOKyRBOwnHDDJ6NvizROoch5OM0Oo/5Yp5Cw4hRPvhfuvBTToC7eNJHf6iQXik1uVbNMokz0qakrZ3OJlJSFtmSyN6zkBHsUSB9d0hjw7QxC6haMHFxHoZSb3nU4+lVQpVXXaI5yjscFlUfgDinoea8X50h0ZVuPMWRAtA6wZU32qhgR9cxM\" \"MSvWrmpD4ReEEq7ygbmS5s12yeEa6yEFDmNw/MOYkv2jgJiM02x9r8ipGXtGYswUfjTmAsRxVQJx50Z2E28zVupiylLaGUeFOtWBHa1Em7R0EF12eKChWK0DZTLXwLQTbSBlPExBgM3ONys04xSS1BW+sQWhFURxQIDAQAB\""}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "_autodiscover._tcp.portal-syncsoft.com.", "Type": "SRV", "TTL": 3600, "ResourceRecords": [{"Value": "0 1 443 mail.portal-syncsoft.com"}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "autoconfig.portal-syncsoft.com.", "Type": "A", "TTL": 300, "ResourceRecords": [{"Value": "15.135.14.30"}]}},
      {"Action": "DELETE", "ResourceRecordSet": {"Name": "autodiscover.portal-syncsoft.com.", "Type": "A", "TTL": 300, "ResourceRecords": [{"Value": "15.135.14.30"}]}}
    ]
  }'
```

> **Note:** Only mail-related records are deleted. Other records (app, api, stg, etc.) are NOT touched.

---

## Phase 4: Verify & Test

### Task 8: Test Email Delivery

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

| Phase | Tasks | What | Who |
|-------|-------|------|-----|
| Phase 1 | Task 1 | Move DNS to Route 53 | **DONE** |
| Phase 2 | Task 2 | Create mail DNS records | Claude (CLI) |
| Phase 3 | Task 3 | Update hostname + Mailcow config | Claude (SSH) |
| Phase 3 | Task 4 | Add domain + DKIM | **User (Mailcow UI)** + Claude (DNS) |
| Phase 3 | Task 5 | Create mailboxes | **User (Mailcow UI)** |
| Phase 3 | Task 6 | Update rDNS (PTR) | Claude (CLI) |
| Phase 3 | Task 7 | Remove old domain + cleanup DNS | **User (Mailcow UI)** + Claude (CLI) |
| Phase 4 | Task 8 | Test email delivery | **User (browser)** + Claude (CLI) |

**Total estimated time:** ~1 hour (DNS already propagated)
