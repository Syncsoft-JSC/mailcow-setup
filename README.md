# Mailcow Mail Server Setup on AWS

Self-hosted mail server using [Mailcow Dockerized](https://github.com/mailcow/mailcow-dockerized) on AWS EC2 for internal company email.

## Architecture

- **Server:** EC2 t3.medium (2 vCPU, 4GB RAM), Ubuntu 22.04 LTS
- **Mail Server:** Mailcow Dockerized (Postfix, Dovecot, SOGo, Rspamd, ClamAV, MariaDB, Redis, Nginx)
- **IP:** Elastic IP with rDNS configured
- **DNS:** Route 53 (MX, SPF, DKIM, DMARC, autoconfig, autodiscover)
- **Backup:** S3 with Glacier lifecycle (30d Standard, delete after 90d)
- **Monitoring:** CloudWatch agent + SNS alerts

## Domain

- **Current:** `portal-syncsoft.com` (migrating to `syncsoft.ai`)
- **Webmail:** `https://mail.<domain>/SOGo`
- **Admin UI:** `https://mail.<domain>/admin`

## Docs

| File | Description |
|------|-------------|
| [Design Spec](docs/specs/2026-03-24-mailcow-aws-setup-design.md) | Architecture, DNS, security group, backup strategy |
| [Setup Plan](docs/superpowers/plans/2026-03-24-mailcow-aws-setup.md) | Step-by-step implementation plan for initial setup |
| [Migration Plan](docs/superpowers/plans/2026-03-25-migrate-to-syncsoft-ai.md) | Plan to migrate from portal-syncsoft.com to syncsoft.ai |

## AWS Resources

| Resource | Detail |
|----------|--------|
| EC2 | `i-0fec63c97b3fa9362` (ap-southeast-2) |
| Elastic IP | `15.135.14.30` |
| Security Group | `sg-0321bbc8523508e7d` |
| IAM Role | `mailcow-ec2-role` |
| S3 Backup | `portal-syncsoft-mail-backup` |
| SNS Topic | `mailcow-alerts` |

## Mail Client Config

| Setting | Value |
|---------|-------|
| IMAP | `mail.<domain>:993` (SSL/TLS) |
| SMTP | `mail.<domain>:587` (STARTTLS) |
| Webmail | `https://mail.<domain>` |

## Backup

- Daily at 2:00 AM (crontab)
- Syncs to S3, local cleanup after 7 days
- S3 lifecycle: Standard 30d → Glacier → delete after 90d

## Constraints

- **SES on `syncsoftvn.com`**: Separate system, do not modify
- **Google Workspace on `syncsoftvn.com`**: Separate, do not modify
- **Single server**: No HA/failover
