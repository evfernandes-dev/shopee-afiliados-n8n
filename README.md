# Shopee Affiliates — n8n Automation

n8n workflow that finds electronics accessory deals through the Shopee Affiliate GraphQL API and publishes selected offers to a Telegram channel every 30 minutes. A separate error chain sends Telegram alerts when the workflow fails.

> Personal automation project, published as a sanitized workflow export for documentation and portfolio. It is not a reusable product.

## What it does

1. Every 30 minutes, builds a GraphQL request with 35 electronics accessory keywords (cases, screen protectors, chargers, cables, power banks, headphones, mice, keyboards and similar).
2. Signs the request with SHA256 using `appId + timestamp + payload + appSecret`.
3. Calls the Shopee Affiliate API (`productOfferV2`).
4. Flattens the GraphQL response, interleaves keyword groups, escapes HTML and builds a Telegram caption.
5. Filters only electronics accessories with a word allowlist and a blocklist for clothes, toys, decoration and unrelated items.
6. Deduplicates offers: at most 3 per run and 2 times per day per item, prioritizing items not sent today.
7. Publishes each selected offer to Telegram with `sendPhoto`.
8. On failure, the error chain sends the workflow name, failing node and error details.

## Workflow

Publishing chain:

```text
Every 30 minutes
  -> Build Shopee request (Code: keywords, GraphQL query, signed payload)
  -> Generate signature (Crypto: SHA256)
  -> productOfferV2 (HTTP Request: Shopee GraphQL API)
  -> Extract offers (Code: flatten, interleave, escape HTML)
  -> Filter electronics (Code: allowlist/blocklist)
  -> Deduplicate (Code: 3 per run, 2 per day per item)
  -> Publish offer (Telegram: sendPhoto)
```

Error-alert chain (set as the workflow's **Error Workflow**):

```text
Error Trigger -> Build alert (timestamp, workflow, node, error) -> Send alert (Telegram sendMessage)
```

## Stack

n8n, Shopee Affiliate GraphQL API, Telegram Bot API, Docker/Docker Compose, Oracle Cloud (OCI) and Git/GitHub.

The production infrastructure files (Docker Compose, reverse proxy and TLS) are not included in this repository.

## Repository layout

```text
├── README.md
├── .env.example
├── .gitignore
└── workflows/
    └── shopee-afiliados-producao.json
```

The exported workflow is sanitized: credentials are placeholders (`SEU_APP_ID_AQUI`, `SEU_APP_SECRET_AQUI`, `SEU_CHAT_ID_AQUI`).

## Setup

1. Import `workflows/shopee-afiliados-producao.json` into n8n.
2. Replace the placeholders in the `Build Shopee request`, `Publish offer` and `Send alert` nodes.
3. Create a Telegram API credential with your bot token and select it in both Telegram nodes.
4. In the workflow settings, set **Error Workflow** to this same workflow.
5. Test with a private chat/channel before activating.

`.env.example` documents the expected variable names; n8n does not load it automatically. Fill the node placeholders or use n8n credentials. Never commit `.env` with real values.

## Security

- The current export uses placeholders only.
- Older revisions may still contain real values. Do not reuse credentials from them, and rotate anything that was ever committed.
- `.gitignore` excludes `.env`, private keys, certificates, databases, logs, n8n volumes and raw exports.
- Server addresses and domains are omitted.
- Before publishing a new export, search for `appId`, `appSecret`, `chatId`, `bot<id>:<token>` and other secrets. If a secret was committed, rotate it and rewrite the history.

### Known content issue

The offer caption still ends with `#Suplementos`, a legacy hashtag from an earlier version. Update it in the `Extract offers` Code node to match the current electronics category.

## Backup and restore

- Workflows, credentials and execution data live in the n8n database/volume.
- Store the n8n encryption key outside Git; without it, stored credentials cannot be decrypted.
- Export workflows through the UI and keep raw exports outside the repository until sanitized.
- Never restore a backup over production without taking a current snapshot first.

## Limitations

- No automated tests for the workflow.
- Deduplication state lives in n8n static data, not in this repository.
- Publishes to a single Telegram channel.
- Keywords and filters need periodic review as the Shopee catalog changes.
- This is a documentation snapshot; production may be newer.

## License and usage

Personal automation project kept as backup, documentation and portfolio. Contact the maintainer before reusing the workflow.
