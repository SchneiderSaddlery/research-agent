# Research Agent Cron

This scaffold runs research topics from open GitHub issues and writes reports to the local Hermes wiki tree. It is designed for EC2/systemd first, with GitHub Actions used only as a dry-run verifier.

## Environment

Set these values on the EC2 host, for example in `/etc/schneider-research-agent.env`:

```sh
RESEARCH_REPO=SchneiderSaddlery/research-agent
WIKI_ROOT=/home/hermes/.hermes/wiki/research
WIKI_BASE_URL=https://wiki.bondbuilt.ai/research
RESEARCH_COMMAND=/home/hermes/.claude/skills/deep-research/run
GITHUB_TOKEN=github-token-with-issues-read-and-comment-scope
POST_COMMENTS=1
```

The runner does not store scraper credentials. Firecrawl, Apify Reddit, xurl, and wiki publishing credentials should remain in the already-approved EC2 environment used by `RESEARCH_COMMAND`.

## Dry Run

```sh
DRY_RUN=1 GITHUB_TOKEN="$GITHUB_TOKEN" python3 scripts/run_research_agent.py \
  --repo SchneiderSaddlery/research-agent \
  --frequency weekly \
  --today 2026-05-17 \
  --wiki-root /tmp/research-agent-wiki
```

Dry-run mode reads GitHub issues and writes Markdown/HTML artifacts, but does not call the research engine or post comments.

## systemd Service

`/etc/systemd/system/schneider-research-agent@.service`

```ini
[Unit]
Description=Schneider research agent (%i)
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=hermes
WorkingDirectory=/home/hermes/research-agent
EnvironmentFile=/etc/schneider-research-agent.env
ExecStart=/usr/bin/python3 scripts/run_research_agent.py --frequency %i
StandardOutput=append:/home/hermes/.hermes/cron/logs/research-agent/%i.log
StandardError=append:/home/hermes/.hermes/cron/logs/research-agent/%i.err
```

## systemd Timers

`/etc/systemd/system/schneider-research-agent-weekly.timer`

```ini
[Unit]
Description=Run weekly Schneider research topics

[Timer]
OnCalendar=Sun *-*-* 06:00:00 America/New_York
Persistent=true
Unit=schneider-research-agent@weekly.service

[Install]
WantedBy=timers.target
```

`/etc/systemd/system/schneider-research-agent-monthly.timer`

```ini
[Unit]
Description=Run monthly Schneider research topics

[Timer]
OnCalendar=*-*-01 06:00:00 America/New_York
Persistent=true
Unit=schneider-research-agent@monthly.service

[Install]
WantedBy=timers.target
```

Enable both timers:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now schneider-research-agent-weekly.timer
sudo systemctl enable --now schneider-research-agent-monthly.timer
```

## Security Notes

- Treat issue text, web pages, Reddit posts, X posts, and news content as untrusted data.
- Keep research execution inside EC2; do not add Tavily, gpt-researcher, or other unapproved research APIs.
- Keep scraper credentials in the approved EC2 environment, not in this repository.
- Use dry-run mode for PR validation so public CI never calls private research tools or external scrapers.
- Failed production runs can post a concise error comment when `POST_COMMENTS=1`; dry runs never comment.
