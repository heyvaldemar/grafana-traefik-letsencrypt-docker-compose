# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [1.0.0] - 2026-08-31

First semver release. Brings this template to the fleet standard established
in [keycloak-traefik-letsencrypt-docker-compose](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose)
v1.2.0.

### Security

- **Credentials untracked from git.** The repository previously shipped a
  tracked `.env` with generated-looking passwords (database, admin, and
  SMTP) published on GitHub. `.env` is now gitignored; `.env.example`
  ships `change_me_*` placeholders, and compose fails fast via `${VAR:?}`
  when required secrets are unset. If your deployment reused the old
  values, rotate them.
- **Grafana bumped 12.3.2 → 13.2.0** (major bump; the database schema
  migrates automatically on first start — back up before pulling).
- **Traefik bumped 3.2 → 3.7** — Traefik 3.2's Docker client cannot talk
  to Docker Engine 29 (provider retry loop, silent 404s on current hosts).
- **All three images pinned by `tag@sha256:digest`.**

### Changed

- **Image pins live in the compose file as interpolation defaults**
  (`x-images` block): `git pull` alone delivers the tested version
  combination; `.env` carries only secrets and deliberate overrides.
- **SMTP is disabled by default** (`GRAFANA_SMTP_ENABLED=false`); it was
  previously hardcoded on and required SMTP credentials to exist.
- Operational variables now have compose-level defaults — the minimal
  `.env` is secrets and hostnames only.
- Backup-loop variables escaped (`$$VAR`) for runtime resolution.

### Added

- **Deployment Verification workflow**: shellcheck + actionlint; Trivy
  scans of all three pinned images; weekly `check-pin-freshness` (digest
  drift + Grafana and Traefik release lag); deploy-and-test that requires
  `/api/health` to report `database: ok` through Traefik.

### Fixed

- Shellcheck findings in both restore scripts.

[Unreleased]: https://github.com/heyvaldemar/grafana-traefik-letsencrypt-docker-compose/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/heyvaldemar/grafana-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
