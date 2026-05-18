<!-- AI CONTEXT: This document is the single source of truth for Lixeo AB's infrastructure,
products, services, and architecture. When making changes to any part of the system (deployments,
DNS, database, edge functions, new apps, server config, etc.), consult this document first for
current state and update it after changes are applied. Keep all sections accurate and in sync. -->

# Lixeo AB - Infrastructure & Architecture

> **Company:** Lixeo AB | **Website:** [www.lixeo.se](https://www.lixeo.se)
>
> *Last updated: 2026-05-18 — Deployed ClinBrew n8n on KVM 2 (clinbrew-n8n.lixeo.cloud)*

---

## Table of Contents

- [Overview](#overview)
- [Products](#products)
  - [Production Apps](#production-apps)
  - [In Development](#in-development)
- [Infrastructure](#infrastructure)
  - [Hostinger VPS Servers](#hostinger-vps-servers)
  - [KVM 2 — Internal Tools & Services](#kvm-2--internal-tools--services)
  - [KVM 4 — Clinova + ClinBrew](#kvm-4--clinova--clinbrew)
  - [KVM 8 — ST Tracker Web](#kvm-8--st-tracker-web)
- [Cloud Services](#cloud-services)
  - [Supabase](#supabase)
  - [Firebase](#firebase)
  - [Cloudflare](#cloudflare)
- [Domains & URLs](#domains--urls)
- [Technology Stack](#technology-stack)
- [GitHub Repositories](#github-repositories)
- [Cursor MCP Servers](#cursor-mcp-servers)
- [ST Tracker Signing Architecture](#st-tracker-signing-architecture)
- [Network & Tunnel Configuration](#network--tunnel-configuration)
- [SSH Access](#ssh-access)
- [Architecture Diagram](#architecture-diagram)

---

## Overview

Lixeo AB is a Swedish company building healthcare and productivity applications. The infrastructure runs on **3 Hostinger VPS servers** managed with **Coolify** (self-hosted PaaS), with databases on **Supabase Cloud** for the mobile apps and **self-hosted PostgreSQL** for the clinical system.

---

## Products

### Production Apps

#### ST Tracker

Swedish ST-läkare (specialist trainee doctor) training app for tracking residency progress.

| Property      | Value                                              |
|---------------|----------------------------------------------------|
| **Built with** | FlutterFlow                                       |
| **Platforms**  | Web, Android, iOS                                 |
| **Web URL**    | [app.sttracker.se](https://app.sttracker.se)      |
| **Web hosted on** | KVM 8 (via Coolify, static build + nginx)      |
| **Database**   | Supabase Cloud (`rbffcuylplfcpjwkgcvd`, eu-central-1, Postgres 15) |
| **Push Notifications** | Firebase Cloud Messaging (only Firebase use) |
| **Digital Signing** | BankID (via Idkollen API) + PAdES PDF signing (VPS microservice) |
| **GitHub repo** | `Lixeo-AB/sttracker` (branch: `web` for web deploy) |

#### ClockTracker

Employee schedule tracking and clock-in/clock-out app.

| Property      | Value                                              |
|---------------|----------------------------------------------------|
| **Built with** | FlutterFlow                                       |
| **Platforms**  | Web, Android, iOS                                 |
| **Web URL**    | [clocktracker.se](https://clocktracker.se)         |
| **Web hosted** | Deployed directly from FlutterFlow (not on Coolify) |
| **Database**   | Supabase Cloud (`assyvrgtnnxvstnuysbw`, eu-north-1, Postgres 17) |

### In Development

#### Clinova (Dossiya)

Patient medical journal system designed for the Moroccan market. Full-stack clinical application with medical imaging support.

| Property      | Value                                                  |
|---------------|--------------------------------------------------------|
| **Built with** | Angular (frontend) + Node.js API + PostgreSQL + Orthanc |
| **Hosted on**  | KVM 4 (Docker Compose via Coolify)                    |
| **URL**        | [clinova.lixeo.cloud](https://clinova.lixeo.cloud)    |
| **GitHub repo** | `majbar-emir/Dossiya`                                |
| **Status**     | Running (in development)                               |

**Stack breakdown:**

| Container   | Image / Build        | Purpose                              |
|-------------|----------------------|--------------------------------------|
| `frontend`  | Dockerfile.frontend  | Angular app served by nginx          |
| `api`       | Dockerfile.api       | Node.js REST API (port 8080)         |
| `postgres`  | postgres:16-alpine   | Local PostgreSQL database (`clinova`)|
| `orthanc`   | Dockerfile.orthanc   | DICOM server for medical imaging     |

#### ClinBrew

Medical literature summary app for doctors. Fetches PubMed abstracts, generates AI summaries, and presents weekly digests.

| Property       | Value                                                    |
|----------------|----------------------------------------------------------|
| **Built with** | Angular 21 (frontend) + FastAPI (API) + Python workers   |
| **Hosted on**  | KVM 4 (app + API + Supabase) + KVM 2 (n8n)             |
| **URL**        | [clinbrew.lixeo.cloud](https://clinbrew.lixeo.cloud)    |
| **GitHub repo**| `Lixeo-AB/ClinBrew`                                     |
| **Status**     | In development                                           |

**Stack breakdown:**

| Container | Purpose |
|-----------|---------|
| `frontend` | Angular app served by nginx |
| `fastapi` | FastAPI REST API (port 8001) |
| `supabase-*` | Self-hosted Supabase (13 containers: db, auth, rest, kong, meta, studio, storage, imgproxy, analytics, realtime, functions, vector, supavisor) |
| `pubmed-worker` | PubMed article fetcher |
| `summary-worker` | AI summarization worker |
| `translation-worker` | Translation worker |

#### SimplyResearch

Data collection tool for researchers. Planned for open-source release.

| Property       | Value                          |
|----------------|--------------------------------|
| **Built with** | TypeScript                     |
| **GitHub repo**| `majbar-emir/SimplyResearch`   |
| **Status**     | Testing phase                  |
| **License**    | Planned open-source            |

---

## Infrastructure

### Hostinger VPS Servers

All 3 servers are hosted on **Hostinger** (data center 19), running **Ubuntu 24.04** with **Coolify** pre-installed. Each server runs **Traefik** as a reverse proxy with automatic **Let's Encrypt** SSL certificates.

| Server | Plan   | IP              | CPUs | RAM   | Disk   | Role                       |
|--------|--------|-----------------|------|-------|--------|----------------------------|
| KVM 2  | KVM 2  | `145.223.118.11`| 2    | 8 GB  | 100 GB | Internal tools & services  |
| KVM 4  | KVM 4  | `72.62.88.124`  | 4    | 16 GB | 200 GB | Clinova + ClinBrew (dev)   |
| KVM 8  | KVM 8  | `72.60.135.12`  | 8    | 32 GB | 400 GB | ST Tracker web             |

> **Note:** The two Hostinger MCP integrations (`user-hostinger-mcp` and `user-kvm4-hostinger`) connect to the same Hostinger account.

---

### KVM 2 — Internal Tools & Services

The smallest server hosts all internal/admin tooling and shared infrastructure.

**Coolify MCP:** `user-coolify-infra`

#### Applications

| App                    | URL                         | Type          | Source                            | Status  |
|------------------------|-----------------------------|---------------|-----------------------------------|---------|
| Admin Dashboard        | admin.lixeo.cloud           | Dockerfile    | `Lixeo-AB/lixeo-admin`           | Running |
| Lixeo Website          | www.lixeo.se                | Nixpacks      | `Lixeo-AB/lixeo_website`         | Running |
| Intyg Signer           | signer.lixeo.cloud          | Dockerfile    | `Lixeo-AB/intyg-signer`          | Running |
| STTracker Load Tester  | load.lixeo.cloud            | Dockerfile    | `Lixeo-AB/sttracker-load-testing`| Running |
| ClinBrew n8n           | clinbrew-n8n.lixeo.cloud    | Docker Compose | `n8nio/n8n`                      | Running |

> **Removed (2026-05-18):** n8n, Docmost, Dashy, Supabase Monitor (Prometheus + Grafana), postgresql-lixeo, and redis-docmost were stopped and removed to simplify the server.

---

### KVM 4 — Clinova + ClinBrew

Dedicated to the Clinova/Dossiya clinical system and ClinBrew medical literature app.

**Coolify MCP:** `user-coolify-kvm4`

| App      | URL                     | Type           | Source                 | Status  |
|----------|-------------------------|----------------|------------------------|---------|
| Dossiya  | clinova.lixeo.cloud     | Docker Compose | `majbar-emir/Dossiya`  | Running |
| ClinBrew | clinbrew.lixeo.cloud    | Docker Compose | `Lixeo-AB/ClinBrew`   | In dev  |

The Dossiya Docker Compose stack includes 4 containers: `frontend`, `api`, `postgres`, and `orthanc` (see [Clinova details](#clinova-dossiya) above).

The ClinBrew Docker Compose stack includes 16+ containers: `frontend`, `fastapi`, `pubmed-worker`, `summary-worker`, `translation-worker`, and 13 self-hosted Supabase containers (see [ClinBrew details](#clinbrew) above).

---

### KVM 8 — ST Tracker Web

The most powerful server, dedicated to serving the ST Tracker web application.

**Coolify MCP:** `user-coolify-app`

| App            | URL              | Type    | Source                        | Status  |
|----------------|------------------|---------|-------------------------------|---------|
| sttracker-web  | app.sttracker.se | Static  | `Lixeo-AB/sttracker` (web branch) | Running |

The web app is a static FlutterFlow build served by nginx with custom security headers (CSP, HSTS, X-Frame-Options) and caching rules.

---

## Cloud Services

### Supabase

Two Supabase Cloud projects (managed via Supabase MCP servers in Cursor):

| Project       | Ref                    | Region        | Postgres | Created    | Used by      |
|---------------|------------------------|---------------|----------|------------|--------------|
| sttracker     | `rbffcuylplfcpjwkgcvd` | eu-central-1  | 15       | 2023-09-26 | ST Tracker   |
| clocktracker  | `assyvrgtnnxvstnuysbw` | eu-north-1    | 17       | 2025-06-25 | ClockTracker |

**Cursor MCP servers:**
- `plugin-supabase-supabase` — General Supabase management (all projects)
- `user-supabase-sttracker` — Scoped to the ST Tracker project
- `user-supabase-clocktracker` — Scoped to the ClockTracker project

### Firebase

Used **only** in ST Tracker for push notifications (Firebase Cloud Messaging). No other Firebase services are in use.

**Cursor MCP:** `user-firebase`

### Cloudflare

Handles **DNS management** for all domains. Also runs a **Cloudflare Tunnel** on KVM 4.

**Cursor MCP:** `user-cloudflare-api`

**Cloudflare Zone:** `lixeo.cloud` (zone `dd8bc08bcc057f55903098aaa216e3a3`)

**KVM 4 Tunnel:** A Cloudflare Tunnel (`c83e721c-...`) routes traffic from `*.lixeo.cloud` subdomains to the server:
- `coolify-kvm4.lixeo.cloud` → Coolify dashboard (port 8000)
- `*.lixeo.cloud` wildcard → Traefik (port 80) for Coolify-managed apps

> **Note:** The tunnel runs in **local config mode** (`cloudflared` uses a config file on disk). Switching to remote config mode would allow managing tunnel routes via the Cloudflare API without SSH.

---

## Domains & URLs

| Domain / URL            | Points to                     | Purpose                          |
|-------------------------|-------------------------------|----------------------------------|
| `www.lixeo.se`          | KVM 2                         | Company website                  |
| `app.sttracker.se`      | KVM 8                         | ST Tracker web app               |
| `clocktracker.se`       | FlutterFlow hosting           | ClockTracker web app             |
| `clinova.lixeo.cloud`   | KVM 4                         | Clinova clinical system          |
| `admin.lixeo.cloud`     | KVM 2                         | Internal admin dashboard         |
| `signer.lixeo.cloud`    | KVM 2                         | Intyg Signer (PAdES PDF signing) |
| `load.lixeo.cloud`      | KVM 2                         | STTracker load testing tool      |
| `coolify-kvm4.lixeo.cloud` | KVM 4 (Cloudflare Tunnel)  | Coolify dashboard for KVM 4      |
| `clinbrew.lixeo.cloud`     | KVM 4                       | ClinBrew Angular frontend        |
| `clinbrew-api.lixeo.cloud` | KVM 4                       | ClinBrew FastAPI API             |
| `clinbrew-supabase.lixeo.cloud` | KVM 4                  | ClinBrew Supabase Kong gateway   |
| `clinbrew-studio.lixeo.cloud`   | KVM 4                  | ClinBrew Supabase Studio         |
| `clinbrew-n8n.lixeo.cloud` | KVM 2                       | ClinBrew n8n automation          |

---

## Technology Stack

| Layer            | Technology                                                  |
|------------------|-------------------------------------------------------------|
| **Mobile apps**  | FlutterFlow (Dart/Flutter)                                  |
| **Web apps**     | Angular, FlutterFlow Web                                    |
| **Backend APIs** | Node.js (Clinova), FastAPI (ClinBrew), Supabase Edge Functions |
| **Python**       | FastAPI, SQLAlchemy, Alembic, Python workers (ClinBrew)     |
| **Databases**    | Supabase Cloud (PostgreSQL), self-hosted PostgreSQL 16 (Clinova), self-hosted Supabase (ClinBrew) |
| **Medical Imaging** | Orthanc (DICOM server)                                  |
| **Hosting**      | Hostinger VPS (Ubuntu 24.04)                                |
| **PaaS**         | Coolify (self-hosted)                                       |
| **Reverse Proxy**| Traefik v3.6 (via Coolify)                                  |
| **SSL**          | Let's Encrypt (auto via Traefik)                            |
| **DNS**          | Cloudflare                                                  |
| **Push Notifications** | Firebase Cloud Messaging (ST Tracker only)            |
| **Digital Signing** | BankID (via Idkollen API), PAdES-B-B (self-signed CA)    |
| **CI/CD**        | GitHub → Coolify (auto-deploy from GitHub repos)            |
| **Source Control**| GitHub (Lixeo-AB org + personal repos)                     |

---

## GitHub Repositories

### Lixeo-AB Organization

| Repository                         | Language   | Project                          | Default Branch | Created    |
|------------------------------------|------------|----------------------------------|----------------|------------|
| `Lixeo-AB/sttracker`              | —          | ST Tracker (web deploy)          | `main`         | 2025-11-26 |
| `Lixeo-AB/ST-Tracker`             | PLpgSQL    | ST Tracker (DB migrations/SQL)   | `main`         | 2024-05-19 |
| `Lixeo-AB/lixeo_website`          | TypeScript | Company Website (lixeo.se)       | `main`         | 2026-03-13 |
| `Lixeo-AB/intyg-signer`           | JavaScript | Intyg Signer service             | `main`         | 2025-12-22 |
| `Lixeo-AB/sttracker-load-testing` | JavaScript | k6 load tests for ST Tracker     | `main`         | 2025-11-30 |
| `Lixeo-AB/lixeo-admin`           | HTML       | Admin dashboard (admin.lixeo.cloud)| `main`       | 2026-05-18 |
| `Lixeo-AB/ClinBrew`             | TypeScript | ClinBrew (medical literature app)  | `main`       | 2026-05-18 |

### Personal (majbar-emir)

| Repository                         | Language   | Project                                      | Default Branch  | Created    |
|------------------------------------|------------|----------------------------------------------|-----------------|------------|
| `majbar-emir/Dossiya`             | TypeScript | Clinova — offline-first clinical system       | `main`          | 2026-04-09 |
| `majbar-emir/STTRACKER`           | Dart       | ST Tracker (FlutterFlow source, original)     | `flutterflow`   | 2023-09-07 |
| `majbar-emir/SimplyResearch`      | TypeScript | SimplyResearch — data collection tool for researchers | `main`   | 2026-03-26 |

> All repositories are **private** except `Lixeo-AB/lixeo-admin` (public — static HTML dashboard, no secrets).

---

## Cursor MCP Servers

These MCP (Model Context Protocol) servers are configured in Cursor IDE for managing infrastructure directly from the editor.

### Infrastructure & Hosting

| MCP Server            | Purpose                                   | Status |
|-----------------------|-------------------------------------------|--------|
| `user-coolify-infra`  | Coolify on KVM 2 (internal tools)         | OK     |
| `user-coolify-kvm4`   | Coolify on KVM 4 (Clinova)                | OK     |
| `user-coolify-app`    | Coolify on KVM 8 (ST Tracker web)         | OK     |
| `user-hostinger-mcp`  | Hostinger account API                     | OK     |
| `user-kvm4-hostinger` | Hostinger account API (same account)      | OK     |
| `user-cloudflare-api` | Cloudflare DNS management                 | OK     |

### Databases

| MCP Server                  | Purpose                            | Status |
|-----------------------------|------------------------------------|--------|
| `plugin-supabase-supabase`  | Supabase management (all projects) | OK     |
| `user-supabase-sttracker`   | Supabase — ST Tracker project      | OK     |
| `user-supabase-clocktracker`| Supabase — ClockTracker project    | OK     |

### Development Frameworks

| MCP Server                   | Purpose                               | Status |
|------------------------------|---------------------------------------|--------|
| `user-angular-cli`           | Angular CLI tooling                   | OK     |
| `user-dart`                  | Dart & Flutter development tools      | OK     |
| `user-flutterflow-designer`  | FlutterFlow Designer integration      | OK     |
| `user-remotion-documentation`| Remotion (React video) documentation  | OK     |
| `user-firebase`              | Firebase management                   | OK     |

### DevOps & Utilities

| MCP Server            | Purpose                          | Status  |
|-----------------------|----------------------------------|---------|
| `cursor-ide-browser`  | In-IDE browser for testing       | OK      |
| `user-github`         | GitHub API integration           | OK      |
| `user-dockerhub`      | Docker Hub integration           | Error   |

> **Note:** The Docker Hub MCP server is currently errored. Check MCP status in Cursor Settings if needed.

---

## ST Tracker Signing Architecture

ST Tracker has a two-layer signing system for medical certificates (intyg):

### Layer 1: BankID Digital Signing (Identity Verification)

The handledare (supervisor) signs certificates using **Swedish BankID** via the Idkollen API. This provides legal identity verification.

**Flow:**
1. App triggers `signIntygDocument` Supabase Edge Function
2. Edge function fetches the document data and computes a SHA-256 digest
3. Sends a signing request to **Idkollen BankID API** (`/v3/bankid-se/sign`)
4. Returns QR codes + BankID deep-link to the app
5. Handledare scans QR / opens BankID and signs
6. Idkollen sends a callback to `signingCallback` Edge Function
7. Signature status is stored in `bankid_sessions` table

**Key edge functions:** `signIntygDocument`, `signingCallback`, `DocumentGroupSign`, `issueGroupCertificates`

### Layer 2: PAdES PDF Signing (Document Integrity)

A self-hosted microservice on KVM 2 applies **PAdES-B-B digital signatures** to the PDF files using a self-signed certificate chain.

**Flow (storage-mediated):**
1. `generateCertificatePdf` Edge Function generates an unsigned PDF and uploads to Supabase Storage
2. Calls `POST signer.lixeo.cloud/sign` with storage path (small JSON, no PDF bytes)
3. The signer downloads the PDF from Supabase Storage, applies PAdES signature
4. Uploads signed PDF back to Supabase Storage at `{recordId}_signed.pdf`
5. Returns metadata: SHA-256 hash, cert fingerprint, signing level

**Service:** `Lixeo-AB/intyg-signer` on KVM 2 (port 3001)
**Certificate:** Self-signed CA with 1-year validity, stored in `/certs/signer.p12`

> **Status (2026-05-18):** The Intyg Signer is **running** and healthy. Signer certificate valid until 2026-12-28, CA certificate valid until 2035-12-26.

### Certificate Types (Supabase Edge Functions)

ST Tracker has **40+ edge functions** for generating and managing different certificate types:

| Category | Functions | Examples |
|----------|-----------|---------|
| Certificate PDF generation | ~20 functions | `intygauskultation2015`, `intygklinisk2021`, `intygkurs2015`, etc. |
| No-logo variants | ~12 functions | `IntygAuskultation2015_NoLogo`, `IntygKlinisk2021_NoLogo`, etc. |
| Signing & callbacks | 4 functions | `signIntygDocument`, `signingCallback`, `DocumentGroupSign`, `issueGroupCertificates` |
| PDF processing | 2 functions | `generateCertificatePdf`, `processPdfQueue` |
| User management | 4 functions | `create_newuser`, `delete_user`, `bulk_onboard_users`, `verify-reset-password` |
| Push notifications | 1 function | `send-push-notification` |
| Guest evaluator | 3 functions | `setup-guest-evaluator`, `get-form-by-token`, `submit-form-by-token` |
| Invitations | 3 functions | `create-invite-and-send-email`, `send-invite-email`, `send-invite-email-v2` |
| Feedback | 1 function | `generate-feedback-pdf` |

---

## Network & Tunnel Configuration

### KVM 4 (Cloudflare Tunnel)

KVM 4 uses a **Cloudflare Tunnel** for ingress instead of direct IP exposure:

```
Internet → Cloudflare Edge → Tunnel (c83e721c-...) → KVM 4
```

| Subdomain                  | Target            | Purpose                    |
|----------------------------|--------------------|----------------------------|
| `coolify-kvm4.lixeo.cloud` | localhost:8000    | Coolify dashboard          |
| `*.lixeo.cloud`            | localhost:80      | Traefik (Coolify apps)     |

The tunnel daemon (`cloudflared`) runs in **local config mode** (config file on disk, not manageable via Cloudflare API).

### KVM 2 (Cloudflare Tunnel)

KVM 2 also uses a **Cloudflare Tunnel** (`0c93b295-...`) for its `*.lixeo.cloud` subdomains. The tunnel runs as a `cloudflared` systemd service in **local config mode** (config at `/etc/cloudflared/config.yml`).

| Subdomain                  | Target             | Purpose                    |
|----------------------------|--------------------|----------------------------|
| `admin.lixeo.cloud`       | localhost:3100     | Admin dashboard            |
| `signer.lixeo.cloud`      | localhost:3001     | Intyg Signer               |
| `coolify-infra.lixeo.cloud`| localhost:8000    | Coolify dashboard          |
| `clinbrew-n8n.lixeo.cloud` | localhost:5678    | ClinBrew n8n automation    |

> **Note:** `www.lixeo.se` and `load.lixeo.cloud` use direct A records (not the tunnel) since they are on the `lixeo.se` domain or use path-based routing.

### KVM 8 (Direct IP)

KVM 8 uses **direct IP** with Cloudflare DNS (A records) → Traefik → apps.

---

## SSH Access

Direct SSH access is configured for all 3 VPS servers from the local machine. The SSH key (`~/.ssh/id_ed25519`) is deployed to each server's `authorized_keys`.

**Local SSH config:** `~/.ssh/config`

| Alias  | Host             | User | Key                      |
|--------|------------------|------|--------------------------|
| `kvm2` | 145.223.118.11   | root | `~/.ssh/id_ed25519`      |
| `kvm4` | 72.62.88.124     | root | `~/.ssh/id_ed25519`      |
| `kvm8` | 72.60.135.12     | root | `~/.ssh/id_ed25519`      |

**Usage:**
```bash
ssh kvm2   # Connect to KVM 2
ssh kvm4   # Connect to KVM 4
ssh kvm8   # Connect to KVM 8
```

> **Note:** The SSH key has a passphrase stored in macOS Keychain (`UseKeychain yes` in SSH config). The key is automatically loaded by `ssh-agent`.

---

## Architecture Diagram

```
                        ┌──────────────────┐
                        │    Cloudflare    │
                        │  DNS + Tunnel    │
                        └────────┬─────────┘
                                 │
               ┌─────────────────┼─────────────────┐
               │ (A record)      │ (Tunnel)         │ (A record)
        ┌──────▼──────┐   ┌─────▼──────┐   ┌──────▼──────┐
        │   KVM 2     │   │   KVM 4    │   │   KVM 8     │
        │ 145.223.    │   │ 72.62.     │   │ 72.60.      │
        │ 118.11      │   │ 88.124     │   │ 135.12      │
        │ 2 CPU / 8GB │   │ 4 CPU/16GB │   │ 8 CPU/32GB  │
        ├─────────────┤   ├────────────┤   ├─────────────┤
        │  Coolify     │   │  Coolify   │   │  Coolify    │
        │  Traefik     │   │  Traefik   │   │  Traefik    │
        ├─────────────┤   ├────────────┤   ├─────────────┤
        │ Admin Dash  │   │ Clinova    │   │ sttracker   │
        │ lixeo.se    │   │  ├ Angular │   │  web app    │
        │ Load Tester │   │  ├ Node API│   │  (static)   │
        │ Signer  ✓   │   │  ├ Postgres│   │             │
        │ ClinBrew n8n│   │  └ Orthanc │   │             │
        │  ✓ running  │   │ ClinBrew   │   │             │
        │             │   │  ├ Angular │   │             │
        │             │   │  ├ FastAPI │   │             │
        │             │   │  ├ Workers │   │             │
        │             │   │  └Supabase │   │             │
        └──────┬──────┘   └────────────┘   └─────────────┘
               │
               │ PAdES PDF signing
               ▼
        ┌────────────────────────┐
        │    Supabase Cloud      │        ┌──────────────┐
        ├────────────────────────┤        │   Idkollen   │
        │ sttracker (eu-central) │◄──────►│  BankID API  │
        │  ├ 40+ Edge Functions  │        └──────────────┘
        │  ├ BankID signing      │
        │  └ PDF generation      │
        │ clocktracker (eu-north)│◄── ClockTracker app
        └────────────────────────┘

        ┌────────────────────────┐   ┌──────────────┐
        │  FlutterFlow Hosting   │   │   Firebase   │
        ├────────────────────────┤   ├──────────────┤
        │ clocktracker.se (web)  │   │ FCM (push)   │
        └────────────────────────┘   └──────────────┘
```
