# Competition Monitoring Pipeline

A modular, cost-conscious **scrape → normalize → pre-filter → deduplicate → optionally classify → notify → persist** pipeline for competition and opportunity monitoring.

## Why this is not a Zapier workflow

Zapier is excellent for simple SaaS event chains, but this brief requires browser scraping, PostgreSQL state, candidate scoring, deduplication, retries, and optional LLM classification. Implementing those pieces in Zapier would require several paid steps and an external scraper/database, making the result less reliable and typically more expensive at recurring volume. This repository keeps deterministic work in Python and calls an LLM only for candidates that pass the cheap pre-filter.

## Quick start

```bash
cp .env.example .env
# Edit .env: add source URLs and notification settings.
docker compose up -d db
python -m venv .venv && source .venv/bin/activate
pip install -e '.[dev,browser]'
playwright install chromium
competition-monitor init-db
competition-monitor run
```

For a one-off dry run without notifications or database writes:

```bash
competition-monitor run --dry-run
```

## Configuration

`SOURCE_URLS` is a JSON array, for example:

```env
SOURCE_URLS=["https://example.org/opportunities", "https://example.org/feed.xml"]
```

The initial implementation extracts generic HTML cards, links, headings, deadline/prize text, and RSS/Atom entries. Site-specific selectors should be added in `ingestion/extractors.py` as sources are finalized.

## Production operation

Run `competition-monitor run` from cron, Prefect, or a process supervisor. The database enforces URL-hash uniqueness, while the pipeline also checks existing records before notification. Configure one or more of Slack, Discord, or SMTP. Keep secrets in the runtime environment, not in source control.

Recommended production controls:

- Use a persistent PostgreSQL instance and daily backups.
- Add source-specific extractors and respect robots.txt, terms, rate limits, and access controls.
- Keep `LLM_PROVIDER=none` until deterministic filtering is tuned; then enable `openai` only if needed.
- Use a low-frequency schedule such as every 4–12 hours unless a source provides a push feed.
- Monitor failed fetches and quarantined records; do not silently discard malformed payloads.

## Architecture

- `config/`: settings and model bindings
- `ingestion/`: HTTP/browser retrieval, RSS parsing, HTML normalization
- `filtering/`: keyword scoring, deduplication, optional LLM relevance
- `notifications/`: Slack, Discord, SMTP Markdown delivery
- `storage/`: SQLAlchemy models, repository, migrations
- `orchestration/`: CLI and scheduled entry point

## Cost profile

The pipeline itself has no per-step automation-platform fee. Costs are limited to hosting, PostgreSQL, bandwidth, and optional LLM calls. The default path performs keyword filtering before any LLM call and uses `gpt-4o-mini` only when explicitly enabled. A persistent host is required for unattended execution; the current sandbox is for development and testing, not durable background service operation.
