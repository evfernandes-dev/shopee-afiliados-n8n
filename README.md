# Shopee Affiliates — n8n Automation

Automation that publishes Shopee deals to a Telegram channel, running continuously on
a self-hosted **n8n** instance. This repository holds the **versioned and sanitized**
production workflow, plus the documentation needed to reproduce it in another
environment.

> **This repository contains no credentials.** Every secret was replaced with a
> placeholder (`SEU_APP_ID_AQUI`, `SEU_APP_SECRET_AQUI`, `SEU_CHAT_ID_AQUI` — Portuguese
> for `YOUR_APP_ID_HERE`, `YOUR_APP_SECRET_HERE`, `YOUR_CHAT_ID_HERE`).
> See [Data and credential protection](#data-and-credential-protection).

---

## Table of contents

- [Overview](#overview)
- [Workflow](#workflow)
- [Technologies](#technologies)
- [Hosting architecture](#hosting-architecture)
- [Repository layout](#repository-layout)
- [Requirements](#requirements)
- [Configuration](#configuration)
- [Backup and restore](#backup-and-restore)
- [Running locally](#running-locally)
- [Data and credential protection](#data-and-credential-protection)
- [Publishing to GitHub](#publishing-to-github)

---

## Overview

The automation polls the **Shopee Affiliate API** for sports supplements and gym
accessories deals and automatically posts the promotions it finds to a Telegram channel,
applying deduplication rules so the same product is not repeated excessively. If any step
fails, an error alert is sent to the same channel.

What it does:

1. Queries the Shopee affiliate API with fitness and supplement keywords.
2. Filters the results, ensuring rigorous authenticity checks for sports supplements and verifiable promotions for gym accessories.
3. Discards offers that were already published recently.
4. Publishes up to 3 offers per run (every 30 minutes) with image, price, discount,
   shop and link.
5. Sends a Telegram alert if the workflow errors out.

---

## Workflow

The workflow published in this repository has **11 nodes** and two chains: the
publishing chain and the error-alert chain.

### Schedule

| Trigger | Configuration |
| --- | --- |
| `A cada 30 minutos` (Schedule Trigger) | Cron expression `0 */30 * * * *` — runs every 30 minutes |

### Publishing chain

```
A cada 30 minutos                       (Schedule Trigger)
        │
        ▼
PREP 1 — Montar request Shopee          (Code)
        │  builds the keyword list, the productOfferV2
        │  GraphQL query and the string to be signed
        ▼
Crypto                                  (Crypto — SHA256)
        │  computes the request signature
        ▼
PREP 3 — productOfferV2 (suplementos)    (HTTP Request)
        │  POST to the Shopee affiliate GraphQL API
        ▼
PREP 4 — Extrair ofertas                (Code)
        │  flattens the response (GraphQL aliases k0, k1, ...),
        │  interleaves the groups (round-robin) and escapes HTML for Telegram
        ▼
PREP 5 — Filtrar suplementos            (Code)
        │  authenticity & brand checks for sports supplements (whey, creatina,
        │  bcaa, pre-treino) + gym accessories filter + blocklist
        ▼
PREP 6 — Deduplicar (itemId)            (Code)
        │  uses workflow static data to avoid repeats:
        │  max 3 per run and max 2x per day per item
        ▼
PREP 7 — Publicar oferta (Telegram)     (Telegram — sendPhoto)
           sends image + caption with name, price, discount,
           shop, link and hashtags
```

### Error-alert chain

```
Error Trigger
        │
        ▼
Montar alerta                           (Code)
        │  builds the message with the timestamp (America/Sao_Paulo),
        │  workflow name, failing node and error details
        ▼
Enviar alerta (Telegram)                (Telegram — sendMessage)
```

> The alert chain is fired by the **Error Workflow** field in the workflow's own
> settings (self-reference).

> Node names are kept in Portuguese because that is how they are defined in the
> production workflow and in the exported JSON.

### Key business rules

- **Request signing:** the API requires an `Authorization` header in the format
  `SHA256 Credential=<appId>, Timestamp=<timestamp>, Signature=<signature>`, where the
  signature is the SHA256 hash of `appId + timestamp + payload + appSecret`.
- **API limit:** 50 results per keyword; the PREP 1 node uses GraphQL *aliases*
  (`k0`, `k1`, ...) to query several niches in a single call.
- **Category filter:** only sports supplements (whey protein, creatine, BCAA, pre-workout, glutamine, multivitamin) and gym accessories (straps, knee sleeves, shaker bottles, gym bags, resistance bands) pass through, with rigorous authenticity checks and official/authorized vendor prioritization.
- **Never invents prices:** published values are only the ones returned by the API.
- **Deduplication state** lives in the workflow's *static data* inside n8n, not in
  this repository.

---

## Technologies

| Technology | Role in the project |
| --- | --- |
| **n8n** | Automation engine that runs the workflow |
| **n8n nodes** | `Schedule Trigger`, `Code`, `Crypto`, `HTTP Request`, `Telegram`, `Error Trigger` |
| **Shopee Affiliate API** | Public GraphQL offers endpoint (`productOfferV2`) |
| **Telegram Bot API** | Publishes offers and sends alerts (via n8n's built-in credential) |
| **Docker / Docker Compose** | Runs n8n on the Oracle Cloud VM |
| **Oracle Cloud (OCI)** | VM infrastructure hosting n8n |

---

## Hosting architecture

```
┌─────────────────────────────────────────────┐
│  Oracle Cloud VM                            │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │  Docker Compose                       │  │
│  │                                       │  │
│  │  ┌─────────┐      ┌────────────────┐  │  │
│  │  │  n8n    │◄────►│  Data volume   │  │  │
│  │  │         │      │  (SQLite +     │  │  │
│  │  └────┬────┘      │  config)       │  │  │
│  │       │           └────────────────┘  │  │
│  └───────┼───────────────────────────────┘  │
│          │ HTTPS                            │
└──────────┼──────────────────────────────────┘
           │
    ┌──────┴───────┐        ┌──────────────────┐
    │  Reverse     │        │  External       │
    │  proxy +     │        │  services:      │
    │  dynamic     │        │  • Shopee API   │
    │  domain      │        │  • Telegram API │
    └──────────────┘        └──────────────────┘
```

- **n8n** runs in a container (Docker/Docker Compose) inside an Oracle Cloud VM.
- State (workflows, credentials, executions, deduplication *static data*) lives in a
  **persistent volume** pointed at by Docker Compose — without it, the automation loses
  its history and credentials.
- The n8n UI and API are reached over **HTTPS** behind a reverse proxy with a dynamic
  domain, exposed only with TLS.
- Credentials (Telegram, etc.) are stored **encrypted by n8n itself**, using an
  encryption key that never leaves the VM.

> Addresses, IPs, domains and identifiers of the production infrastructure are
> deliberately omitted from this repository.

---

## Repository layout

```
shopee-afiliados-n8n/
├── README.md                                  # this document
├── .gitignore                                 # prevents secrets from being committed
├── .env.example                               # environment variable template
└── workflows/
    └── shopee-afiliados-producao.json         # production workflow (sanitized)
```

- `workflows/shopee-afiliados-producao.json` is a **faithful export** of the production
  workflow with these security adjustments:
  - `appId` → `SEU_APP_ID_AQUI`
  - `appSecret` → `SEU_APP_SECRET_AQUI`
  - Telegram `chatId` → `SEU_CHAT_ID_AQUI`
  - removed: sharing metadata, e-mails, internal instance identifiers, the
    deduplication history and internal version data.

---

## Requirements

To reproduce the project elsewhere:

| Requirement | Details |
| --- | --- |
| n8n | Self-hosted instance, recent version supporting `Code` (v2), `HTTP Request` (v4.5), `Schedule Trigger` (v1.2) and `Telegram` (v1.2) nodes |
| Docker + Docker Compose | To run n8n following the production architecture |
| Shopee affiliate account | With `appId` and `appSecret` generated in the affiliate dashboard |
| Telegram bot | Created via `@BotFather`, with a token and a chat/channel where the bot is an admin |
| Domain + TLS | To expose the n8n UI over HTTPS |

---

## Configuration

### 1. Environment variables

Copy the template and fill in **your** values (`.env` is ignored by Git):

```bash
cp .env.example .env
```

| Variable | Description | Example value |
| --- | --- | --- |
| `SHOPEE_APP_ID` | Application ID in the affiliate dashboard | `SEU_APP_ID_AQUI` |
| `SHOPEE_APP_SECRET` | Application secret (used in the SHA256 signature) | `SEU_APP_SECRET_AQUI` |
| `TELEGRAM_CHAT_ID` | Target chat/channel ID | `SEU_CHAT_ID_AQUI` |

> The published workflow uses **literal placeholders** instead of these values, so the
> file will not work as-is when imported. After importing, replace the placeholders with
> the real values **directly in the n8n UI** — never commit the filled-in values.

### 2. Credentials inside n8n

Prefer n8n's credential store over any file:

1. In n8n: **Credentials → Add credential → Telegram API**.
2. Paste the **bot token** and save. Name it something like `Telegram account`.
3. In the `PREP 7 - Publicar oferta (Telegram)` and `Enviar alerta (Telegram)` nodes,
   select that credential under **Credential for Telegram API**.

Recommendations:

- The **bot token must never** appear in `Code` nodes, in versioned environment
  variables or in workflow exports.
- For the Shopee `appSecret`, the safest path is moving it into a secure n8n mechanism
  (for example a *credential* of a type suitable for your environment) instead of
  keeping it as text inside the `Code` node. Until that is done, the real value should
  exist only on the production instance.

### 3. After importing

After importing `workflows/shopee-afiliados-producao.json`:

1. Fill in the placeholders (`SEU_APP_ID_AQUI`, `SEU_APP_SECRET_AQUI`, `SEU_CHAT_ID_AQUI`).
2. Select the Telegram credential in both send nodes.
3. Re-point the **Error Workflow** field in the workflow settings to the workflow
   itself (the exported value points to the original production instance's ID).
4. Confirm the timezone and execution *timeout* in the settings if needed.
5. Activate the workflow manually and monitor the first run.

---

## Backup and restore

### Backup

What must be preserved:

| Item | Where it lives | How to protect it |
| --- | --- | --- |
| Workflows | n8n database | Export via the UI (sidebar) or the API |
| Credentials | n8n database (encrypted) | Volume backup **plus** the n8n **encryption key** |
| Encryption key | n8n config file | Store outside Git, in a secrets vault |
| Docker volume | Oracle Cloud VM | Volume snapshot / backup of the directory referenced in Compose |

Recommended procedure:

1. **Snapshot the volume** on the VM before any maintenance.
2. **Export the workflows** through the n8n UI (do not version the raw file — it
   contains placeholders only in this repository; the production export contains real
   values).
3. Store the export and the encryption key **outside** the repository (vault/secret
   manager).

### Restore

1. Bring the instance up with Docker Compose, pointing the volume at the backup.
2. Restore the config file with the **same encryption key** — without it, stored
   credentials cannot be decrypted.
3. Import the workflow exports (or restore the whole database).
4. Verify triggers, credentials and **Error Workflow** before activating.
5. Run a manual execution and check a test channel before releasing to production.

> Never restore a backup over the production instance without first confirming the
> current snapshot has been saved.

---

## Running locally

To run a test copy without touching production:

```bash
docker run -d \
  --name n8n-local \
  -p 5678:5678 \
  -e N8N_HOST=localhost \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=http \
  -e GENERIC_TIMEZONE=America/Sao_Paulo \
  -e TZ=America/Sao_Paulo \
  -v n8n_local_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

1. Open `http://localhost:5678` and create the owner account.
2. Import `workflows/shopee-afiliados-producao.json`.
3. Fill in the placeholders and register the Telegram credential.
4. Use a **test** chat (not the production channel) to validate the messages.
5. Activate the workflow only once you are happy with the results.

---

## Data and credential protection

Measures adopted in this repository:

- **No secrets are versioned:** `appSecret`, bot token, chat IDs and API keys are kept
  out of Git. The published workflow uses clearly marked placeholders.
- **Robust `.gitignore`:** ignores `.env*` (except `.env.example`), private keys,
  certificates, databases, logs, n8n volumes and raw exports (`exports/brutos/`) that
  have not been reviewed yet.
- **No operational data:** internal instance identifiers, e-mails, the deduplication
  history and infrastructure domains/IPs were removed from the export.
- **Public repository by explicit decision of the maintainer** (visibility chosen for
  portfolio/recruiting purposes). Since the content is visible to everyone, no secret
  may ever be added here — sanitization must happen **before** every commit. If the
  visibility goes back to private, that is a policy change that should be recorded.
- **Production untouched:** this repository is a documented mirror; nothing here is
  automatically applied to the production instance.

Recommended practices for the maintainer:

1. Never commit an export taken straight from production without sanitizing it first
   (look for `appSecret`, `appId`, `chatId`, `bot<id>:<token>`).
2. Prefer storing secrets in n8n credentials over text inside `Code` nodes.
3. Run a secret scanner frequently (e.g. `gitleaks`, `trufflehog`) before every push.
4. If a secret is ever committed, **removing it in the next commit is not enough** —
   the content stays in the history. In that case, revoke/rotate the key and rewrite
   the history (e.g. `git filter-repo`) before any push.
5. Keep the n8n API token and the encryption key in a secrets vault, never in the
   repository or in a shared terminal.

---

## Publishing to GitHub

The repository is published in **public mode** (explicit decision by the maintainer,
for portfolio display):

- Name: `shopee-afiliados-n8n`
- Visibility: **public**
- Default branch: `main`

> Since it is public, every commit here is permanent and visible to everyone. Review
> the files before each push.

Procedure used (the token must never be typed into scripts, terminal history or this
file):

```bash
# 1. Install the GitHub CLI (once)
winget install --id GitHub.cli

# 2. Authenticate (interactive flow, run in your own terminal)
gh auth login

# 3. Confirm the authenticated account
gh auth status

# 4. Publish (from the project folder)
cd shopee-afiliados-n8n
gh repo create shopee-afiliados-n8n --private --source=. --remote=origin --push

# 5. If the decision is to keep it public:
gh repo edit <user>/shopee-afiliados-n8n --visibility public \
  --accept-visibility-change-consequences
```

Before any `push`, run a secret scan:

```bash
git ls-files | xargs grep -nE "bot[0-9]{6,}:|[A-Z0-9]{25,}|-----BEGIN|sk-[A-Za-z0-9]{20,}|gh[pousr]_[A-Za-z0-9]{20,}"
```

Recommendation for a public repository: enable **Secret scanning** and **Push
protection** under *Settings → Code security and analysis* (free for public
repositories) to automatically block credential uploads.

Rules:

- Public visibility demands full sanitization discipline. Never push a raw production
  export: look for `appSecret`, `appId`, `chatId` and `bot<id>:<token>` before
  committing.
- If a secret was already committed, removing it in the next commit **is not enough** —
  the content stays in the history (and on a public repo it may already have been
  indexed). Revoke/rotate the key immediately and rewrite the history (e.g.
  `git filter-repo`) before any further push.

---

## License and usage

Personal automation project, maintained as backup, documentation and portfolio.
Contact the maintainer before reusing the code.
