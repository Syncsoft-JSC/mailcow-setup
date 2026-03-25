# CLAUDE.md

## Project

Mailcow mail server setup and management on AWS. This repo contains design specs and implementation plans — no application code.

## Key Info

- **AWS Profile:** `default`
- **Region:** `ap-southeast-2` (Sydney)
- **EC2:** `i-0fec63c97b3fa9362`, IP `15.135.14.30`
- **SSH:** `ssh -i ~/.ssh/antony.syncsoft-key.pem ubuntu@15.135.14.30`
- **Hosted Zone (portal-syncsoft.com):** `Z09305612IEOUCEGO7H9X`
- **S3 Backup Bucket:** `portal-syncsoft-mail-backup`
- **SNS Topic:** `arn:aws:sns:ap-southeast-2:650008108447:mailcow-alerts`
- **Git remote host alias:** `syncsoft.github.com` (not `github.com`)

## Domain Migration

Migrating from `portal-syncsoft.com` to `syncsoft.ai`. See migration plan in `docs/superpowers/plans/2026-03-25-migrate-to-syncsoft-ai.md`.

## Constraints

- **DO NOT** touch `syncsoftvn.com` DNS — it has Google Workspace + SES running
- Mail server is single EC2 instance, not HA
- Mailcow admin UI is at `/admin`, webmail (SOGo) is at `/SOGo`
- Mailcow admin login uses `/admin` path, not root `/`

## Conventions

- Plans go in `docs/superpowers/plans/`
- Specs go in `docs/specs/`
- All AWS commands use `--profile default --region ap-southeast-2`
