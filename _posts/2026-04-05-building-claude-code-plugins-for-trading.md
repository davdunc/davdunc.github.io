---
layout: post
title: "Building Claude Code Plugins for Trading: From Bad Data to Real Market Analysis"
date: 2026-04-05
author: davdunc
---

Last Thursday, Globalstar (GSAT) spiked 26% afterhours on rumors that Amazon was in talks to acquire the company. I wanted to build a trading game plan around it. What started as a simple exercise turned into a lesson in why data quality matters and ended with me publishing a set of [Claude Code](https://claude.ai/claude-code) plugins for working with market data.

The plugins are open source: [davdunc/davdunc-plugins](https://github.com/davdunc/davdunc-plugins).

### The Problem: Web Search Data Is Not Trading Data

My first attempt was embarrassing. I asked Claude Code to research the GSAT move and build a game plan with entry/exit prices for four scenarios. It searched the web, pulled numbers from news articles, and produced something that looked reasonable at first glance.

It wasn't. The web sources said the premarket high was around $78. The actual postmarket high on April 1 was **$87.75**, with a double tweezer top on the 15-minute chart at 18:30-18:45. The stock had already faded nearly $9 from that high to close the afterhours at $79.06 before premarket even opened the next morning.

That changes everything about how you plan the trade. A game plan built around "$78 resistance" when the real rejection was at $87.75 is worse than no plan at all. It gives you false confidence in wrong levels.

### Getting Real Data: Massive.com Flat Files

I have a [Massive.com](https://massive.com) (formerly Polygon.io) subscription that includes S3 access to their flat files: per-minute OHLCV bars for every US-listed stock, including extended hours. The data is delivered as gzipped CSV files organized by date, one file per day covering the entire market.

The files are big (~27MB compressed per day for minute aggregates), but extracting a single ticker is straightforward:

```bash
aws s3 cp s3://flatfiles/us_stocks_sip/minute_aggs_v1/2026/04/2026-04-01.csv.gz \
  /tmp/2026-04-01.csv.gz --endpoint-url https://files.massive.com

zcat /tmp/2026-04-01.csv.gz | head -1 > GSAT_2026-04-01_minute.csv
zcat /tmp/2026-04-01.csv.gz | grep "^GSAT," >> GSAT_2026-04-01_minute.csv
```

That gives you every minute bar from premarket through postmarket. With this data, I could see exactly what happened:

- **18:30 ET**: GSAT exploded from $69.41 to $87.75 in a single minute candle on 50K+ shares
- **18:45 ET**: Second candle hit $87.76 (matching high), forming the double tweezer top
- **19:45 ET**: Stock had faded to $79.06, giving back nearly half the spike
- **04:00 AM next day**: Premarket opened at $76.01, a $3 gap down from the afterhours close
- **07:45 AM**: Brief rally to $81, immediately rejected
- **10:45 AM**: Session low of $72.50
- **15:45 PM**: Closed at $77.70 after an afternoon recovery rally

The correct game plan, built on this data, identified $72.50-$73.00 as the key dip-buy entry (right at the April 1 regular session high of $72.98). That entry, with a stop at $71, yielded a 1:3 risk/reward to the close. The web-sourced plan had none of this.

### Building the MCP Server

I already had a Python MCP server for daily watchlist generation ([equities-watchlist](https://github.com/davdunc/davdunc-plugins/tree/main/equities-watchlist)) that uses the same flat files. It screens for candidates using Finviz and Polygon data, calculates Camarilla pivot points, detects significant intraday moves, and generates structured morning trading briefs.

The core of it is a `flatfiles.py` module that handles S3 downloads with local caching:

```python
def _download_and_cache(data_type: str, d: date) -> Path:
    cache = _cache_path(data_type, d)
    if cache.exists():
        return cache

    s3 = _s3_client()
    key = _s3_key(data_type, d)
    resp = s3.get_object(Bucket=BUCKET, Key=key)
    compressed = resp["Body"].read()
    decompressed = gzip.decompress(compressed).decode("utf-8")
    cache.write_text(decompressed, newline="")
    return cache
```

The MCP server exposes tools like `get_watchlist`, `get_trading_plan`, `get_previous_day_analysis`, and `get_calendar_events`. Claude Code can call these directly during a conversation to pull live data.

### Building the Skill

The MCP server is great for automated daily workflows, but I also wanted a simple way to say "get me GSAT data for last Thursday" during ad-hoc analysis. That's where Claude Code skills come in.

A skill is a markdown file with YAML frontmatter that tells Claude Code how to perform a task. Mine lives at `~/.claude/skills/market-data/SKILL.md` and handles:

1. Parsing the ticker and date range from arguments
2. Resolving S3 credentials (env vars, then `.env` file, with clear error messages if missing)
3. Downloading the full-market flat file from S3
4. Extracting only the requested ticker's rows
5. Caching locally so repeated requests are instant
6. Reporting a session-by-session summary

Usage is simple:

```
/market-data GSAT 2026-04-01 2026-04-02
```

Claude downloads the data, stores it under `~/market_data/GSAT/`, and reports what it found: premarket range, regular session OHLCV, postmarket activity.

### Publishing as a Plugin

Claude Code has a plugin system that bundles skills with metadata for distribution. The structure is:

```
market-data/
  .claude-plugin/
    plugin.json         # name, version, author
  skills/
    market-data/
      SKILL.md          # the skill definition
  README.md
```

For portability, I made the skill handle multiple credential formats (people set env vars differently), configurable storage directories, and automatic EST/EDT detection. The credential resolution checks `MASSIVE_S3_ACCESS_KEY` and `MASSIVE_ACCESS_KEY` naming conventions, looks in `./.env` then `~/.env`, and gives clear setup instructions if nothing is found.

Both plugins live in a single repository: [davdunc/davdunc-plugins](https://github.com/davdunc/davdunc-plugins).

### What I Learned

**Don't trust web-scraped financial data for trading decisions.** News articles round numbers, mix up sessions, and lack the granularity you need for entries and exits. Always use source data.

**Extended hours data changes the entire thesis.** The GSAT tweezer top at $87.75 was the most important signal of the session, and it happened at 6:30 PM the night before. If you only look at regular session data, you're trading blind.

**Claude Code is a good environment for this kind of work.** The combination of MCP servers for programmatic access, skills for ad-hoc tasks, and the ability to iterate on analysis in conversation made the whole workflow faster than writing scripts from scratch. The Massive.com MCP server gave real-time API access, and the flat files filled in the historical depth.

**Build tools that cache aggressively.** The full-market minute files are ~27MB each. Downloading them once and extracting tickers locally means subsequent analysis is instant. Both the MCP server and the skill check for cached data before hitting S3.

The plugins are MIT licensed. If you trade US equities and have a Massive.com subscription, they might save you some time: [davdunc/davdunc-plugins](https://github.com/davdunc/davdunc-plugins).
