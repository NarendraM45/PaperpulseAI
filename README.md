# PaperPulse AI

**Automated ArXiv discovery and researcher-focused paper briefs, delivered to Telegram.**

PaperPulse is a self-hosted n8n workflow that searches selected ArXiv subject categories, filters already-seen papers, extracts citation metadata, and builds a concise evidence-grounded digest.

## Features

- Searches the latest submissions in configurable ArXiv categories.
- Normalizes versioned ArXiv IDs and deduplicates papers across categories.
- Extracts title, authors, submission date, primary/all subjects, DOI, and journal reference when available.
- Uses OpenRouter to summarize each abstract into contribution, method, reported evidence, limitations, and tags.
- Sends HTML-formatted Telegram messages with direct links to the papers.
- Splits large digests to stay below Telegram message limits.
- Runs locally with Docker Compose; n8n credentials remain in its local encrypted database.

## Architecture

```mermaid
flowchart LR
    A[Daily 7:00 AM trigger] --> B[Topic and result settings]
    B --> C[Build ArXiv queries]
    C --> D[Fetch Atom feeds]
    D --> E[Parse paper and citation metadata]
    E --> F[Filter previously seen IDs]
    F --> G[Summarize each abstract with OpenRouter]
    G --> H[Build researcher digest]
    H --> I[Deliver through Telegram]
```

## Example digest fields

Each paper entry includes its arXiv ID, submission date, matched category, primary subject, other subjects, authors, and DOI or journal reference when supplied by ArXiv. The AI brief contains:

- **Contribution** — the paper's claim, based on its abstract.
- **Method** — the described approach.
- **Evidence** — datasets and quantitative results reported in the abstract.
- **Limitations** — limitations stated in the abstract, or a note that none are stated.
- **Tags** — three to five research keywords.

The prompt tells the model to treat the abstract as source text, avoid inventing results, and disclose when the abstract lacks evidence or limitations. Readers are reminded to verify claims against the full paper before citing them.

## Requirements

- Docker Desktop with Docker Compose.
- An OpenRouter API key.
- A Telegram bot and the destination chat ID.

## Setup

1. Copy `.env.example` to `.env` and replace the placeholder with a unique random `N8N_ENCRYPTION_KEY` of at least 32 characters. Keep the same key for future restarts; do not commit `.env`.
2. Start n8n from this directory:

   ```powershell
   docker compose up -d
   ```

3. Open `http://localhost:5678` and create or sign in to the local n8n owner account.
4. Import `workflow.json` from the n8n Workflows menu.
5. Create an **HTTP Header Auth** credential with header name `Authorization` and value `Bearer <your OpenRouter API key>`. Select it in `Summarize Paper (OpenRouter)`.
6. Create a **Telegram API** credential with your bot token. Select it in `Send to Telegram` and replace `YOUR_TELEGRAM_CHAT_ID` with your own destination chat ID.
7. In `Config`, choose the ArXiv categories, number of papers per category, and model. Execute a one-paper manual test and confirm the Telegram output before publishing the workflow.

The default config searches `cs.AI`, `cs.LG`, `cs.CL`, and `stat.ML`, with up to five papers per category. The schedule is 7:00 AM in `Asia/Kolkata`.

## Model capacity and rate limits

The default model is [`xingchen-agi/xing4.0-29b-a4b:free`](https://openrouter.ai/xingchen-agi/xing4.0-29b-a4b:free). OpenRouter lists a 262,144-token context window. Paper requests are separate: the abstract is capped at 5,000 characters and each summary at 450 output tokens, so context size is not the practical limit for this workflow.

OpenRouter currently lists free-model limits of 20 requests per minute and 50 per day. The workflow sends one request per paper, one at a time, with a six-second interval (about 10 requests per minute). Four categories with five results each means up to 20 paper requests in one full digest, before duplicate filtering. Retries and manual runs add requests. Free provider capacity can still vary and return HTTP 429 errors; this workflow has seen intermittent 429s from the Xing free provider. Check the current [OpenRouter pricing and rate limits](https://openrouter.ai/pricing) before increasing volume. Limits checked October 27, 2026.

## Deduplication and operation

Deduplication uses n8n workflow static data and retains up to 2,000 ArXiv IDs. n8n persists this data after successful production executions; manual executions do not persist it. Summary calls use six-second batching and one delayed retry after 60 seconds.

The repository workflow export contains no API credentials. You must select your own credentials after import. The chat ID is a placeholder and must be configured for your Telegram destination.

Stop the local n8n container with:

```powershell
docker compose down
```

This keeps the `n8n_data` volume. `docker compose down -v` removes the workflow database, credentials, and execution state.

## Validation status

A one-paper end-to-end run has delivered the formatted digest to Telegram. Full batches have encountered intermittent rate limits from the free Xing provider, so large-batch reliability depends on provider availability.

## Resume description

> Built a Docker-hosted n8n research pipeline that discovers ArXiv papers by subject, normalizes and deduplicates citation metadata, generates evidence-grounded AI briefs, and delivers formatted daily digests through Telegram.