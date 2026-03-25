# Mailcow Mail Server on AWS — Design Spec

**Date:** 2026-03-24
**Domain:** mail.portal-syncsoft.com
**Hosted Zone:** Z09305612IEOUCEGO7H9X
**Status:** Approved

---

## 1. Overview

Self-hosted mail server for internal employees using Mailcow Dockerized on AWS EC2.

| Criteria | Detail |
|----------|--------|
| Purpose | Internal email for employees (send/receive, attachments) |
| Team size | 20-50 people, slight growth expected |
| Volume | < 1,000 emails/day |
| Budget | $50-200/month (estimated ~$55/month) |
| Ops capability | Developers with Docker/Linux knowledge, no dedicated ops |
| Mail clients | Mixed: Outlook, Thunderbird, mobile, webmail |
| Availability | Accept short downtime during maintenance |
| LDAP | Not needed — manage users directly in Mailcow UI |
| Existing SES | Separate system on `syncsoftvn.com` — DO NOT TOUCH |

## 2. Architecture

```
                    ┌──────────────────────────────┐
                    │         Route 53              │
                    │   mail.portal-syncsoft.com         │
                    │   MX, SPF, DKIM, DMARC, PTR  │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │      Elastic IP               │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │     EC2 - t3.medium           │
                    │     Ubuntu 22.04 LTS          │
                    │     2 vCPU, 4GB RAM           │
                    │                               │
                    │  ┌─────────────────────────┐  │
                    │  │   Mailcow Dockerized    │  │
                    │  │                         │  │
                    │  │  Postfix (MTA)          │  │
                    │  │  Dovecot (IMAP)          │  │
                    │  │  SOGo (Webmail)         │  │
                    │  │  Rspamd (Anti-spam)     │  │
                    │  │  ClamAV (Anti-virus)    │  │
                    │  │  MariaDB + Redis        │  │
                    │  │  Nginx + Let's Encrypt  │  │
                    │  └─────────────────────────┘  │
                    │                               │
                    │  EBS gp3 - 100GB              │
                    └───────────────────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │      S3 Backup               │
                    │  30d Standard → Glacier       │
                    │  (permanent, never delete)    │
                    └──────────────────────────────┘
```

## 3. AWS Resources & Cost

| Resource | Config | Cost/month |
|----------|--------|------------|
| EC2 (t3.medium) | 2 vCPU, 4GB RAM, Ubuntu 22.04 LTS | ~$30 (Reserved 1yr) |
| EBS (gp3) | 100GB | ~$8 |
| Elastic IP | 1 static IP | ~$3.65 |
| Route 53 | Hosted zone + queries | ~$1 |
| S3 + Glacier | Backup (permanent) | ~$3 |
| Data Transfer | < 100GB outbound | ~$9 |
| **Total** | | **~$55/month** |

## 4. DNS Records (Route 53)

Domain: `portal-syncsoft.com`, IP: `52.xx.xx.xx` (Elastic IP)

| Record | Name | Value | TTL |
|--------|------|-------|-----|
| A | `mail.portal-syncsoft.com` | `52.xx.xx.xx` | 300 |
| MX | `portal-syncsoft.com` | `10 mail.portal-syncsoft.com` | 3600 |
| TXT (SPF) | `portal-syncsoft.com` | `v=spf1 a mx ip4:52.xx.xx.xx ~all` | 3600 |
| TXT (DKIM) | `dkim._domainkey.portal-syncsoft.com` | (Mailcow auto-generates) | 3600 |
| TXT (DMARC) | `_dmarc.portal-syncsoft.com` | `v=DMARC1; p=quarantine; rua=mailto:admin@portal-syncsoft.com` | 3600 |
| PTR (rDNS) | `52.xx.xx.xx` | `mail.portal-syncsoft.com` | (via AWS Console) |
| A | `autoconfig.portal-syncsoft.com` | `52.xx.xx.xx` | 300 |
| A | `autodiscover.portal-syncsoft.com` | `52.xx.xx.xx` | 300 |

> **DMARC ramp-up plan:** Start with `p=quarantine` → after 1 month move to `p=reject`.

## 5. Security Group

| Port | Protocol | Purpose | Source |
|------|----------|---------|--------|
| 25 | TCP | SMTP inbound | 0.0.0.0/0 |
| 465 | TCP | SMTPS | 0.0.0.0/0 |
| 587 | TCP | Submission (client + internal apps) | 0.0.0.0/0 |
| 143 | TCP | IMAP | 0.0.0.0/0 |
| 993 | TCP | IMAPS | 0.0.0.0/0 |
| 110 | TCP | POP3 (optional) | 0.0.0.0/0 |
| 995 | TCP | POP3S (optional) | 0.0.0.0/0 |
| 443 | TCP | HTTPS (Webmail + Admin) | 0.0.0.0/0 |
| 80 | TCP | HTTP (Let's Encrypt) | 0.0.0.0/0 |
| 22 | TCP | SSH | Admin IP only |

## 6. Internal App Mail Relay

Internal apps send mail via authenticated SMTP:

- **Server:** `mail.portal-syncsoft.com:587` (STARTTLS)
- **Account:** `app@portal-syncsoft.com` or `noreply@portal-syncsoft.com`
- **Auth:** Username + password (generated in Mailcow)
- Rate limit configurable via Mailcow UI

## 7. Backup Strategy

**Crontab:**
```bash
0 2 * * * /opt/mailcow-dockerized/helper-scripts/backup_and_restore.sh backup all
0 3 * * * aws s3 sync /opt/mailcow-backup/ s3://portal-syncsoft-mail-backup/
```

**S3 Lifecycle:**
- 0-30 days: S3 Standard
- After 30 days: Glacier (~$0.004/GB/month)
- **Delete after 90 days**

## 8. Mail Client Config

| Client | IMAP | SMTP |
|--------|------|------|
| Outlook | `mail.portal-syncsoft.com:993` (SSL) | `mail.portal-syncsoft.com:587` (STARTTLS) |
| Thunderbird | Autodiscover | Autodiscover |
| iOS/Android | `mail.portal-syncsoft.com:993` (SSL) | `mail.portal-syncsoft.com:587` (STARTTLS) |
| Webmail | `https://mail.portal-syncsoft.com` | — |

## 9. Monitoring

- **CloudWatch:** CPU, RAM, Disk — alarm when disk > 80%
- **Mailcow UI:** Queue, spam stats, mail traffic
- **Monthly:** Check IP blacklist (mxtoolbox.com), review spam stats, verify SSL cert

## 10. Server Hardening

- **Swap:** Configure 4GB swap to handle ClamAV memory spikes
- **fail2ban:** Mailcow includes built-in Netfilter/fail2ban — verify enabled for SMTP, IMAP, webmail
- **TLS:** Enforce TLS 1.2+ on Postfix and Dovecot (Mailcow default)
- **OS updates:** Enable unattended-upgrades for security patches
- **Fallback:** If RAM is insufficient, disable ClamAV first (Rspamd still provides spam filtering)
- **Upgrade path:** If team grows beyond 50, upgrade to t3.large (8GB)

## 11. Pre-deployment Checklist

1. [ ] Request AWS to remove port 25 outbound restriction (1-2 business days)
2. [ ] Provision EC2 + EBS + Elastic IP
3. [ ] Configure 4GB swap on EC2
4. [ ] Configure Route 53 DNS records (including autoconfig/autodiscover)
5. [ ] Configure rDNS (PTR) via AWS Console
6. [ ] Verify rDNS resolves correctly (`dig -x 52.xx.xx.xx`)
7. [ ] Install Docker + Mailcow
8. [ ] Add domain + DKIM in Mailcow UI → update DNS
9. [ ] Verify fail2ban is enabled
10. [ ] Enable unattended-upgrades
11. [ ] Create employee mailboxes
12. [ ] Create app relay account (`app@portal-syncsoft.com`)
13. [ ] Enable 2FA for all users
14. [ ] Setup S3 backup bucket + lifecycle policy (never delete)
15. [ ] Enable S3 bucket versioning
16. [ ] Setup crontab for daily backup
17. [ ] Test backup restore on a fresh instance
18. [ ] Setup CloudWatch alarms
19. [ ] Test send/receive with external providers (Gmail, Outlook)
20. [ ] Test autodiscover with Thunderbird/Outlook
21. [ ] IP warm-up: send low volume first, increase gradually over 1-2 weeks
22. [ ] Distribute mail client config to employees
23. [ ] DMARC ramp-up: `p=none` → `p=quarantine` (after 2 weeks) → `p=reject` (after 1 month)

## 12. Disaster Recovery

If the EC2 instance fails:
1. Launch new EC2 (same spec) + attach new EBS + Elastic IP
2. Install Docker + Mailcow
3. Download latest backup from S3: `aws s3 sync s3://portal-syncsoft-mail-backup/ /opt/mailcow-backup/`
4. Restore: `/opt/mailcow-dockerized/helper-scripts/backup_and_restore.sh restore`
5. Verify DNS still points to correct IP

**Estimated recovery time:** 1-2 hours (manual process)

## 13. Constraints

- **SES system on `syncsoftvn.com`**: Completely separate, do not modify
- **No LDAP**: User management directly in Mailcow UI
- **Single server**: No HA/failover — accept short maintenance downtime
