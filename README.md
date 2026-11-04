# Shopee Affiliates — n8n Automation

An n8n workflow that finds electronics accessory deals through the Shopee Affiliate GraphQL API and publishes selected offers to a Telegram channel every 30 minutes. The workflow also has a separate error-alert chain.

> **Project status:** personal automation project, published as a sanitized workflow export for documentation and portfolio. It is not a reusable product.

---

## What it does

1. Every 30 minutes, builds a GraphQL request with 35 electronics accessory keywords (phone cases, screen protectors, chargers, cables, power banks, headphones, mice, keyboards and similar items).
2. Signs the request with SHA256 using `appId + timestamp + payload + appSecret`.
3. Calls the Shopee Affiliate API (`productOfferV2`).
4. Flattens the GraphQL response, interleaves the keyword groups, escapes HTML and builds a Telegram caption.
5. Filters only electronics accessories with a whole-word allowlist and a blocklist for clothes, toys, decoration and other unrelated items.
6. Deduplicates offers: at most 3 per run and at most 2 times per day for the same item, prioritizing items that have not been sent today.
7. Publishes each selected offer to Telegram with `sendPhoto`.
8. On failure, the error chain sends a Telegram alert with the workflow name, failing node and error details.

---

## Workflow

The exported workflow has **11 active nodes** plus one sticky note.

### Publishing chain

```text
Every 30 minutes
        │
        ▼
Build Shopee request          (Code — keyword list, GraphQL query and signed payload)
        │
        ▼
Generate signature            (Crypto — SHA256)
        │
        ▼
productOfferV2 (electronics)  (HTTP Request — Shopee GraphQL API)
        │
        ▼
Extract offers                (Code — flatten aliases, interleave groups, escape HTML)
        │
        ▼
Filter electronics            (Code — allowlist, blocklist and normalization)
        │
        ▼
Deduplicate                   (Code — static data, 3 per run, 2 per day per item)
        │
        ▼
Publish offer                 (Telegram — sendPhoto)
```

### Error-alert chain

```text
Error Trigger
        │
        ▼
Build alert                   (Code — timestamp, workflow, node and error)
        │
        ▼
Send alert                    (Telegram — sendMessage)
```

The error chain is connected through the workflow's **Error Workflow** setting.

---

## Stack

| Technology | Role |
| --- | --- |
| n8n | Automation engine |
| n8n nodes | Schedule Trigger, Code, Crypto, HTTP Request, Telegram, Error Trigger |
| Shopee Affiliate GraphQL API | Offer source (`productOfferV2`) |
| Telegram Bot API | Offer publishing and error alerts |
| Docker / Docker Compose | Runs the production n8n instance |
| Oracle Cloud (OCI) | VM that hosts n8n |
| Git / GitHub | Version control and documentation |

The production infrastructure files (Docker Compose, reverse proxy and TLS configuration) are not included in this repository.

---

## Repository layout

```text
shopee-afiliados-n8n/
├── README.md
├── .env.example
├── .gitignore
└── workflows/
    └── shopee-afiliados-producao.json
```

The workflow file is a sanitized export. Credentials were replaced with placeholders:

- `SEU_APP_ID_AQUI`
- `SEU_APP_SECRET_AQUI`
- `SEU_CHAT_ID_AQUI`

---

## Setup

1. Import `workflows/shopee-afiliados-producao.json` into n8n.
2. Replace the placeholders in the `Build Shopee request`, `Publish offer` and `Send alert` nodes.
3. In **Credentials → Telegram API**, create a credential with your bot token and select it in both Telegram nodes.
4. In the workflow settings, set the **Error Workflow** field to this same workflow.
5. Use a test chat/channel for the first execution.
6. Activate the workflow when the output looks correct.

The `.env.example` file documents the expected variable names. `.env` is ignored by Git and must never be committed with real values.

---

## Security

- The current workflow file contains no bot token, API key or app secret. Before treating the repository as sanitized, check the commit history and clean any secret that may have been committed.
- The Telegram chat ID is replaced with `SEU_CHAT_ID_AQUI` in the exported workflow.
- `.gitignore` excludes `.env` files, private keys, certificates, databases, logs, n8n volumes and raw exports.
- The production instance's addresses, domains and internal identifiers are omitted.
- Before publishing a new export, search for `appId`, `appSecret`, `chatId`, `bot<id>:<token>` and other secrets.
- If a real secret or private identifier is ever committed, rotate it and rewrite the history; removing it in a later commit is not enough.

### Known content issue

The offer caption still ends with `#Suplementos`, a legacy hashtag from an earlier version of the project. The current filter selects electronics accessories. Update the caption in the `Extract offers` Code node if you want the hashtag to match the current category.

---

## Backup and restore

- Workflows, credentials and execution data live in the n8n database/volume.
- The n8n encryption key must be stored outside Git; without it, stored credentials cannot be decrypted.
- Export workflows through the n8n UI and keep the raw export outside the repository until it is sanitized.
- Never restore a backup over the production instance without taking a current snapshot first.

---

## Limitations

- The n8n workflow has no automated tests.
- Deduplication state lives in n8n static data, not in this repository.
- The workflow publishes to a single Telegram channel.
- The keyword list and filters need periodic review as the Shopee catalog changes.
- The repository is a documentation snapshot; production may contain newer changes.

---

## License and usage

Personal automation project maintained as backup, documentation and portfolio. Contact the maintainer before reusing the workflow.
