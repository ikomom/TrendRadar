# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**TrendRadar** is a lightweight, easy-to-deploy trending news aggregator and analyzer. It monitors 11+ news platforms (Weibo, Douyin, Zhihu, Hacker News, etc.), filters content by custom keywords or AI interests, and pushes notifications to 9+ channels (WeChat, Feishu, Telegram, Email, etc.).

**Key Characteristics:**
- Multi-source hot news aggregation with custom keyword filtering
- Three push modes: daily (full summary), current (live rankings), incremental (new only)
- RSS subscription support with freshness filtering
- AI-powered content analysis and translation (via LiteLLM, supports 100+ models)
- Flexible scheduling system with timeline-based automation
- Multi-channel notifications with smart batching
- MCP (Model Context Protocol) server for AI-powered deep analysis
- Docker & GitHub Actions deployment ready
- Web-based configuration editor

**Version:** v6.6.1 | **Latest MCP:** v4.0.2

---

## Architecture

### Core Components

**1. Main Entry Point** (`trendradar/__main__.py`)
- `NewsAnalyzer` class: Orchestrates the entire pipeline (crawl → analyze → notify)
- `main()`: CLI entry point supporting multiple modes (normal run, doctor check, test notification, show schedule)
- Three main workflows: `_crawl_data()`, `_crawl_rss_data()`, `_execute_mode_strategy()`

**2. Data Flow Pipeline**
```
Crawl Data → Store in DB → Load Analysis Data → Count Frequency (keywords/AI filter)
↓
Generate HTML Report → Run AI Analysis → Translate Content
↓
Send Notifications (to 9+ channels) → Record Execution State
```

**3. Key Modules**

| Module | Purpose |
|--------|----------|
| `trendradar/context.py` | `AppContext` - central config & utility provider |
| `trendradar/core/config.py` | Load & validate `config.yaml`, environment variables |
| `trendradar/crawler/` | Data fetchers for hot news (NewAPI) and RSS feeds |
| `trendradar/storage/` | Multi-backend storage (SQLite local, S3 remote) |
| `trendradar/notification/` | Multi-channel dispatcher (Feishu, DingTalk, Telegram, Email, etc.) |
| `trendradar/report/` | HTML report generation and rendering |
| `trendradar/ai/` | AI analysis, filtering, translation (via LiteLLM) |
| `trendradar/core/scheduler.py` | Timeline-based scheduling system |
| `mcp_server/` | MCP server for AI-powered data queries & analysis |

**4. Configuration System**

Configuration is hierarchical with multiple override levels:

1. **Default values** (built into code)
2. **`config/config.yaml`** (main config file, v2.2.0 format)
3. **`config/timeline.yaml`** (scheduling presets and custom time periods)
4. **`config/frequency_words.txt`** (keyword groups with advanced syntax)
5. **`config/custom/`** (user-specific extensions)
6. **Environment variables** (highest priority, supports Docker/GitHub Actions)

**Environment Variable Overrides:**
- `REPORT_MODE`: Override report mode (daily/current/incremental)
- `AI_API_KEY`, `AI_MODEL`: AI configuration
- `STORAGE_BACKEND`: Storage mode (local/remote/auto)
- `S3_*`: Remote S3/R2 credentials
- `*_WEBHOOK_URL`: Notification channel URLs

**5. Three Push Modes**

| Mode | Use Case | Behavior |
|------|----------|----------|
| **daily** | Full info, daily summaries | Shows all matched news today (includes repeats) |
| **current** | Real-time hot rankings | Shows current live rankings, updates hourly |
| **incremental** | Minimal notifications | Only new unmatched topics (zero repeats) |

**6. Scheduling System**

- Preset templates in `config/timeline.yaml`: `always_on`, `morning_evening`, `office_hours`, `night_owl`, `custom`
- Per-time-period controls: push/analyze on/off, one-time or repeating, report mode override
- Automatic execution tracking to prevent duplicate pushes within a period
- Weekday/weekend differentiation support

---

## Development Commands

### Running Locally

```bash
# Standard run (requires config/config.yaml, frequency_words.txt)
python -m trendradar

# Show current scheduling state
python -m trendradar --show-schedule

# Run environment & configuration check
python -m trendradar --doctor

# Test notification channel connectivity
python -m trendradar --test-notification

# Install dependencies (recommended)
setup-windows.bat              # Windows
./setup-mac.sh                 # macOS/Linux
```

### MCP Server (AI Analysis)

```bash
# HTTP mode (port 3333)
./start-http.bat               # Windows
./start-http.sh                # macOS/Linux

# STDIO mode (for direct CLI/editor integration)
uv run python -m mcp_server.server
```

### Docker Deployment

```bash
cd docker

# Pull latest images
docker compose pull

# Start all services (news crawler + MCP)
docker compose up -d

# Start only news crawler
docker compose up -d trendradar

# View logs
docker logs -f trendradar

# Execute commands in container
docker exec -it trendradar python manage.py run        # Manual crawl
docker exec -it trendradar python manage.py config     # Show config
docker exec -it trendradar python manage.py status     # Check health
```

### Testing

```bash
# Test notification channels
python -m trendradar --test-notification

# Verify environment setup
python -m trendradar --doctor
```

---

## Key Code Patterns

### 1. AppContext Usage

`AppContext` is the central hub for all configuration and utilities:

```python
from trendradar.context import AppContext
from trendradar.core.config import load_config

config = load_config()
ctx = AppContext(config)

# Access utilities
ctx.get_time()                              # Current time in configured timezone
ctx.format_date()                           # Format date string
ctx.load_frequency_words(frequency_file)    # Load keyword groups
ctx.count_frequency(...)                    # Match news to keywords
ctx.generate_html(...)                      # Generate HTML report
ctx.create_notification_dispatcher()        # Get notification sender
ctx.create_scheduler()                      # Get scheduling system
ctx.get_storage_manager()                   # Get data storage backend

ctx.cleanup()                               # Always call at end
```

### 2. Mode-Specific Analysis

The `_execute_mode_strategy()` method is the main orchestrator. Each mode (`daily`, `current`, `incremental`) has different data requirements:

- **daily**: Load all historical data for the day
- **current**: Load latest batch only (from `title_info.last_time`)
- **incremental**: Use only current crawl results (new titles only)

### 3. Data Processing Pipeline

Standard flow in `_run_analysis_pipeline()`:

```
1. Load frequency words (keywords/groups)
2. Choose filter strategy:
   - keyword mode (default): match titles against word groups
   - ai mode: AI interest-based filtering with auto-fallback
3. Convert to report format (keyword or platform grouping)
4. Run AI analysis (if enabled) for HTML report
5. Translate RSS/content (if enabled)
6. Generate HTML snapshot
7. Return stats + AI result for notification dispatch
```

### 4. Notification Dispatch

```python
from trendradar.notification import NotificationDispatcher

dispatcher = ctx.create_notification_dispatcher()
results = dispatcher.dispatch_all(
    report_data=prepared_stats,
    report_type="Daily Summary",
    mode="daily",
    html_file_path=html_path,
    rss_items=rss_list,
    ai_analysis=ai_result,
    standalone_data=standalone,
)
```

Dispatcher automatically:
- Formats content for each channel (Markdown for Feishu, plain text for Bark, etc.)
- Batches long messages respecting channel limits
- Translates if AI translation enabled
- Retries on failure

### 5. Storage Backend Selection

```python
storage_manager = ctx.get_storage_manager()
# Automatically selected:
# - GitHub Actions → S3-compatible remote storage
# - Docker/Local → SQLite local database
# - Can be overridden via STORAGE_BACKEND env var

storage_manager.save_news_data(news_data)   # Save crawled news
storage_manager.detect_new_rss_items(rss)   # Detect new RSS entries
```

---

## Common Tasks

### Adding a New Notification Channel

1. Add handler in `trendradar/notification/senders.py` (implement `_send_*` method)
2. Add config parsing in `trendradar/core/config.py`
3. Register in `NotificationDispatcher.dispatch_all()` conditional
4. Add to `_has_notification_configured()` check in `__main__.py`
5. Document in README.md

### Modifying Keyword Matching Logic

- Core logic: `trendradar/core/frequency.py` (`matches_word_groups()`, `count_frequency()`)
- Supports: plain text, regex `/pattern/`, required words `+`, filter words `!`, global filters
- To add new syntax: update regex in `frequency.py` and document in README

### Adding AI Provider Support

1. Already supported via **LiteLLM** (100+ providers)
2. Format in config: `model: "provider/model_name"`
3. Examples: `deepseek/deepseek-chat`, `openai/gpt-4o`, `gemini/gemini-1.5-flash`
4. Fallback models: configure `fallback_models` in `config.yaml`

### Extending MCP Tools

1. Add tool function in `mcp_server/tools/` (e.g., `data_query.py`)
2. Decorate with `@mcp.tool()` from MCP SDK
3. Register in `mcp_server/server.py` resource list
4. Document in README-MCP-FAQ.md

---

## Configuration Deep Dives

### Keyword Syntax Reference

```txt
# Basic matching
AI
ChatGPT

# Required word (must match both)
+技术
AI

# Filter word (exclude)
!广告
AI

# Quantity limit per group
AI
@10              # max 10 articles

# Regex pattern (Python syntax, case-insensitive)
/\bai\b/         # word boundary
/(?<![a-z])ai(?![a-z])/  # avoid matching 'training'

# Display name
/\bai\b/ => AI相关

# Global filter (applied before any grouping)
[GLOBAL_FILTER]
广告
标题党

[WORD_GROUPS]
AI
ChatGPT
```

### Timeline Configuration

```yaml
# In timeline.yaml
presets:
  morning_evening:
    day_plans:
      monday:  # ... define periods for each day
        - period_key: "morning"
          time_range: ["06:00", "09:00"]
          actions:
            collect: true
            analyze: false
            push: true
            report_mode: "incremental"
      # ... more days
```

### Remote Storage Setup (GitHub Actions)

Required S3 credentials for Cloudflare R2, AWS S3, etc.:

```yaml
storage:
  backend: "remote"  # or "auto" (auto-selects based on environment)
  remote:
    bucket_name: "trendradar-data"
    access_key_id: "xxx"
    secret_access_key: "xxx"
    endpoint_url: "https://xxx.r2.cloudflarestorage.com"
    region: "auto"
```

---

## Important Notes

### Before Making Changes

1. **Read the README first** — it's comprehensive and up-to-date. The project is well-documented.
2. **Check recent commits** (v6.6.1 era) for architectural decisions
3. **Test locally before Docker** — use `python -m trendradar --doctor` to validate
4. **Run full pipeline** — don't just test individual components

### When Implementing Features

- **Scheduling system**: All time-based decisions go through `ResolvedSchedule` from `core/scheduler.py`
- **Data consistency**: Always load fresh data before analysis (don't cache)
- **Error handling**: Use try-catch + `ctx.cleanup()` in finally blocks
- **Storage agnostic**: Write to `storage_manager`, not directly to disk
- **Multi-language support**: Messages use timezone-aware formatting via `AppContext`

### Testing Best Practices

```bash
# Always run these before committing
python -m trendradar --doctor              # Check config
python -m trendradar --test-notification   # Verify channels
python -m trendradar --show-schedule       # Verify timing
python -m trendradar                       # Full run
```

### Debugging Tips

- Enable `DEBUG: true` in `config.yaml` for detailed logging
- Check `output/meta/doctor_report.json` after `--doctor` run
- Use Docker logs: `docker logs -f trendradar` for long-running deployments
- MCP errors appear in client (Cherry Studio, Cline, etc.), not server logs

---

## File Structure Summary

```
TrendRadar/
├── trendradar/              # Main package
│   ├── __main__.py          # Entry point, NewsAnalyzer class
│   ├── context.py           # AppContext (central hub)
│   ├── core/                # Config, frequency matching, scheduling
│   ├── crawler/             # Data fetchers (hot news, RSS)
│   ├── storage/             # SQLite/S3 backends
│   ├── notification/        # Multi-channel dispatch
│   ├── report/              # HTML generation
│   ├── ai/                  # LiteLLM integration, filtering, translation
│   └── utils/               # Time, URL helpers
├── mcp_server/              # MCP server for AI analysis
├── config/                  # User configuration
│   ├── config.yaml          # Main config (v2.2.0)
│   ├── timeline.yaml        # Scheduling presets
│   ├── frequency_words.txt  # Keywords
│   ├── ai_interests.txt     # AI filter interests
│   └── custom/              # User extensions
├── docker/                  # Docker deployment
│   ├── docker-compose.yml
│   └── .env                 # Sensitive config (not in git)
├── output/                  # Generated reports & data
│   ├── html/                # HTML reports by date
│   └── index.html           # Latest report (GitHub Pages)
└── README.md                # Comprehensive documentation
```

---

## Useful References

- **README.md** — 3800+ lines covering all features, config, deployment
- **README-MCP-FAQ.md** — AI analysis workflows and examples  
- **config/config.yaml** — Inline documentation for all 100+ settings
- **Recent commits** — `v6.6.1` refactored display/regions, `v6.5.0` added AI filtering
- **Visualization Editor** — https://sansan0.github.io/TrendRadar/ (helps understand structure)
