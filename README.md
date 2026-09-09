# Grafana + Traefik + Let's Encrypt on Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/grafana-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/grafana-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Contents

- [Why this stack?](#why-this-stack)
- [Prerequisites](#prerequisites)
- [Getting started](#getting-started)
- [Features](#features)
- [Updating](#updating)
  - [Typical use cases](#typical-use-cases)
- [Email alerts (SMTP)](#email-alerts-smtp)
- [Supply chain trust](#supply-chain-trust)
- [Production checklist](#production-checklist)
- [Backups](#backups)
- [Testing](#testing)
- [Security Notes](#security-notes)
- [About the maintainer](#about-the-maintainer)

This repository deploys Grafana behind Traefik with automatic Let's Encrypt TLS, backed by PostgreSQL (instead of the default SQLite, real backups, real concurrency), with a scheduled backup container and companion restore scripts. One `docker compose up` away from production-shaped dashboards at `https://your-domain`.

📙 Full narrative installation guide on the blog: [heyvaldemar.com/install-grafana-using-docker-compose/](https://www.heyvaldemar.com/install-grafana-using-docker-compose/).

## Why this stack?

| Need | This stack | Manual install | Kubernetes | Other compose examples |
|------|-----------|----------------|------------|------------------------|
| Ready to deploy in <10 min | ✅ | ❌ | ✅ if K8s is already running | Often |
| TLS via Let's Encrypt, auto-renewed | ✅ Traefik ACME built-in | Manual certbot | Via cert-manager | Rare |
| PostgreSQL backend (not SQLite) | ✅ | Manual config | ✅ | Rare |
| Zabbix datasource plugin preinstalled | ✅ | Manual install | Init containers | Rare |
| Scheduled DB + data backups + pruning | ✅ | Manual cron | External | Rare |
| Upstream images pinned by `sha256` digest | ✅ | N/A | Depends | Rare |
| Weekly pin-freshness check in CI | ✅ | N/A | Depends | Rare |
| CI-verified deployment on every push | ✅ `database: ok` | N/A | Varies | Rare |
| Credentials via env (never committed) | ✅ | N/A | K8s Secrets | Often committed plaintext |

Four moving parts (Traefik + Grafana + Postgres + backups). No Kubernetes prerequisites, no manual certificate management.

## Prerequisites

Before you start, you need:

- **A Linux server** with a public IP. Tested on Ubuntu 22.04 LTS+ and Debian 12+. Local Mac/Windows works for dev; production is Linux.
- **Docker Engine 24+ and Docker Compose 2.20+.** Quick check: `docker version` and `docker compose version`.
- **A domain you control,** with two `A` records pointing at your server's public IP: one for Grafana (e.g. `grafana.example.com`), one for the Traefik dashboard (e.g. `traefik.grafana.example.com`). DNS must propagate before deploy or the Let's Encrypt TLS-ALPN challenge will fail.
- **Ports 80 and 443 open** on the server's firewall and not bound by another service.
- **~1 GB free RAM and 1 free CPU** for the running stack.

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/grafana-traefik-letsencrypt-docker-compose
cd grafana-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create grafana-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: GRAFANA_DB_PASSWORD, GRAFANA_ADMIN_PASSWORD,
#   GRAFANA_HOSTNAME, GRAFANA_URL, TRAEFIK_HOSTNAME, TRAEFIK_ACME_EMAIL,
#   TRAEFIK_BASIC_AUTH. See .env.example for generation commands.

# 4. Deploy
docker compose -f grafana-traefik-letsencrypt-docker-compose.yml -p grafana up -d
```

Within a minute `https://${GRAFANA_HOSTNAME}` serves the Grafana login with a fresh Let's Encrypt certificate. Log in with `GRAFANA_ADMIN_USERNAME` / `GRAFANA_ADMIN_PASSWORD`.

### What success looks like

```bash
# All services healthy:
docker compose -f grafana-traefik-letsencrypt-docker-compose.yml -p grafana ps

# Health endpoint reports the database is fine:
curl -fsS "https://${GRAFANA_HOSTNAME}/api/health"
# Expected: { "database": "ok", "version": "13.2.1", ... }

# Traefik issued a certificate:
docker compose -p grafana logs traefik | grep -i "adding certificate"

# First backup lands after BACKUP_INIT_SLEEP (default 30m):
docker compose -p grafana logs backups | tail -3
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet. Confirm with `dig +short ${GRAFANA_HOSTNAME}` and `curl -I http://${GRAFANA_HOSTNAME}` from outside the server.
- **`docker compose up` fails with `set in .env`.** A required variable is empty; the error names it.
- **`network grafana-network not found`.** Step 2 was skipped.
- **Login loop or CSRF errors.** `GRAFANA_URL` must exactly match the public URL (protocol + hostname).

### Apply `.env` or compose-file changes

```bash
docker compose -f grafana-traefik-letsencrypt-docker-compose.yml -p grafana up -d --force-recreate
```

## Features

- **Grafana** latest stable (13.2.1) with a PostgreSQL backend: consistent backups and no SQLite locking.
- **Zabbix datasource plugin** (`alexanderzobnin-zabbix-app`) preinstalled by default; add more via `GRAFANA_PLUGINS_INSTALL`.
- **Traefik v3** reverse proxy with automatic HTTP→HTTPS redirect and Let's Encrypt TLS-ALPN certificate issuance.
- **Basic-auth protected Traefik dashboard** on a separate hostname.
- **Sign-up and anonymous access disabled by default**; SMTP off by default (opt-in for alert emails).
- **Scheduled backups** of the database and Grafana data with retention pruning, plus restore scripts.
- **Credentials required at deploy time**: compose fails fast if `.env` is incomplete.

### Typical use cases

- **Dashboards for a Zabbix installation**: pairs with the [Zabbix template](https://github.com/heyvaldemar/zabbix-traefik-letsencrypt-docker-compose); the datasource plugin ships preinstalled.
- **Central observability UI**: Prometheus, Loki, InfluxDB, and dozens of other datasources.
- **Team metrics portal**: org/team permissions on a proper database backend.
- **Alerting hub**: Grafana Alerting with email (enable SMTP), Slack, Telegram, or webhooks.

## Email alerts (SMTP)

SMTP is disabled by default. To enable alert emails, set in `.env`:

```bash
GRAFANA_SMTP_ENABLED=true
GRAFANA_SMTP_ADDRESS=smtp.example.com
GRAFANA_SMTP_PORT=587
GRAFANA_SMTP_USER_NAME=grafana@example.com
GRAFANA_SMTP_PASSWORD=your_smtp_password
GRAFANA_EMAIL_FROM=grafana@example.com
```

then `docker compose up -d --force-recreate`.

## Updating

`./update.sh` moves this checkout to the latest release tag — a combination this repository's CI has booted, upgraded from the previous release on the same volumes, and smoke-tested — and then runs `docker compose up -d`. It refuses to cross a major version unattended, refuses to run over local changes, and names any variable that became required since your version before anything has moved. `./update.sh --dry-run` says what would happen. Every release cut by fleet triage also carries what upstream changed, read from its release notes against this compose file.

## Supply chain trust

This repository is a deployment template, not a custom Docker image. It orchestrates three upstream images:

- [`traefik`](https://hub.docker.com/_/traefik): reverse proxy, Docker Hub official image
- [`grafana/grafana`](https://hub.docker.com/r/grafana/grafana): Grafana upstream
- [`postgres`](https://hub.docker.com/_/postgres): PostgreSQL (alpine), Docker Hub official image

All three are pinned to `tag@sha256:<digest>` as interpolation defaults in the compose file's `x-images` block. Compose pulls by digest, not by tag, and `git pull` alone delivers the version combination this repository has tested. Setting an `*_IMAGE_TAG` variable in `.env` overrides the default when you deliberately want a different version.

Two override levels exist per image. `<PREFIX>_IMAGE_VERSION` in `.env` swaps only the version of that image (Compose then pulls the tag, without a digest) and leaves every other pin as tested; `<PREFIX>_IMAGE_TAG` replaces the whole reference, digest included. The variable names are listed in `.env.example`. Nested defaults need Docker Compose v2.5 or newer (2022); v2.0 to v2.4 leave the inner `${...}` unexpanded and `docker compose up` fails with an invalid reference instead of deploying something unexpected.

The daily `check-pin-freshness` CI job re-resolves each pinned tag against its registry and compares the pinned Grafana and Traefik versions against the latest upstream releases. Any drift fails the run and notifies the maintainer. CI's Deployment Verification workflow runs on every push, pull request, and every day at 06:00 UTC. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

Before exposing this to real users, check every box:

- [ ] **Strong secrets.** `GRAFANA_DB_PASSWORD` and `GRAFANA_ADMIN_PASSWORD` at 24+ random characters; regenerate the Traefik dashboard BCrypt hash per deployment.
- [ ] **Keep sign-ups disabled** (`GRAFANA_USERS_ALLOW_SIGN_UP=false`, the default) unless you mean it.
- [ ] **Host-mount the backup volumes** for disaster recovery: bind the backup paths to host directories covered by your off-host backup solution.
- [ ] **Verify Let's Encrypt cert issuance** in the Traefik logs on first start.
- [ ] **Back up before major upgrades.** Grafana migrates its schema forward automatically (this template moved 12 → 13); the way back is a restore.
- [ ] **Know the restore procedure.** Run both restore scripts against a test environment before you need them in production.

## Backups

The `backups` container performs a dump → archive → prune → sleep loop: `pg_dump | gzip` of the Grafana database, `tar.gz` of the Grafana data directory (dashboards live in the DB; the data dir carries plugins and images), pruning by retention windows, then sleeping `BACKUP_INTERVAL` (default 24h).

Each cycle logs `Database backup OK: <file> (<bytes> bytes)` or `Database backup FAILED` (the same for the data archive where there is one). A failed dump is kept as `<file>.failed` for diagnosis and never overwrites a good backup. Grep the log for `FAILED` from your monitoring.

**Verify backups are running:**

```bash
docker compose -p grafana logs backups | tail -5
docker compose -p grafana exec backups sh -c 'ls -la /srv/grafana-postgres/backups/ /srv/grafana-application-data/backups/'
```

**Restore** with the interactive scripts (`chmod +x *.sh` once): `./grafana-restore-database.sh`, then `./grafana-restore-application-data.sh` if needed.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults: the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`, e.g. `TRAEFIK_MEMORY_LIMIT=512m`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> --format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Container hardening

Every service runs with `security_opt: no-new-privileges:true`, so a process cannot gain privileges through setuid binaries even if it escapes its initial capability set. Infrastructure containers (the reverse proxy, databases, caches, backups) run with `cap_drop: [ALL]` and add back only what their entrypoints need: `NET_BIND_SERVICE` for Traefik to bind :80/:443, `CHOWN`/`SETUID`/`SETGID` (and friends) for database images to own their data directory and drop to their service user. Application containers keep the default capability set on purpose: upstream images assume it, and a wrong guess there is a boot loop in production rather than a hardening win. CI boots the stack under exactly these settings on every push, so what ships is what was tested.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/grafana-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC:

1. **Lint**: shellcheck on both restore scripts, actionlint on the workflow.
2. **Trivy scans** of all three pinned images (CRITICAL/HIGH, SARIF to the Security tab).
3. **Pin freshness** (daily/manual): digest drift plus release-lag checks for Grafana and Traefik.
4. **Deploy-and-test**: boots the full stack with ephemeral credentials and requires `/api/health` to report `database: ok` through Traefik plus a 200 login page. The shipped configuration must produce a working Grafana on its Postgres backend, not just started containers.

A green run is the authoritative proof that the template deploys end-to-end and that its backups restore.

### Backup and restore, proven

`tests/e2e-backup-restore.sh` runs against the live stack and is what CI executes after the HTTPS smoke. The scenario that matters most is the restore roundtrip: insert a marker row, restore the earliest backup, assert the marker is gone. A backup that cannot be restored fails the build. Run it yourself against a running deployment with short intervals in `.env` (`BACKUP_INIT_SLEEP=15s`, `BACKUP_INTERVAL=60s`):

```bash
chmod +x tests/e2e-backup-restore.sh
./tests/e2e-backup-restore.sh
```

It stops the database container briefly to prove failure detection. Run it on a staging copy, not on production.

## Security notes

- Credentials are read from `.env` at deploy time; `.env` is gitignored and compose fails fast on missing required variables.
- **Pre-rotation advisory.** Releases before v1.0.0 (2026-08-31) shipped a tracked `.env` with generated-looking database, admin, and SMTP passwords. Rotate them if your deployment reused them.
- Anonymous access and sign-ups are disabled by default; SMTP is off by default.
- Upstream image digests are pinned; the daily freshness job flags drift loudly.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** · Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
