# Page Change Monitor

[![Python Version](https://img.shields.io/badge/python-3.10%2B-blue.svg?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg?style=for-the-badge&logo=github-actions&logoColor=white)](#)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)
[![Code Style: Black](https://img.shields.io/badge/code%20style-black-000000.svg?style=for-the-badge)](https://github.com/psf/black)

An automated background monitoring service designed to track targeted web page DOM updates. Page Change Monitor periodically fetches web content, extracts specific elements using CSS selectors, filters out rendering noise, computes semantic diffs between snapshots, and dispatches real-time notifications via webhooks and REST endpoints.

---

## Problem Statement & Target Audience

### The Problem
Modern web applications frequently update critical content—such as pricing tiers, Terms of Service, stock availability, and regulatory disclosures—without providing public APIs or RSS feeds.
* **Inadequacy of Uptime Checkers**: Traditional uptime tools monitor HTTP response status codes ($200\text{ OK}$) rather than localized content changes inside the DOM.
* **Inefficiency of Manual Audits**: Manual page checks are error-prone, unscalable, and delay response time to critical updates.
* **Dynamic Noise**: Basic HTML diffing produces continuous false positives caused by changing timestamps, session tokens, dynamic scripts, and shifting layout wrappers.

### Target Audience
* **Competitive Intelligence Teams**: Monitor pricing pages, feature matrix shifts, and promotional updates.
* **Compliance & Legal Officers**: Track regulatory changes, privacy policy updates, and terms of service revisions.
* **Procurement & Inventory Managers**: Track critical vendor stock updates and pricing threshold breaches.
* **DevOps & QA Engineers**: Audit production DOM state stability and detect unintended UI deployments.

---

## Key Features

* **Targeted Extraction & Selector Mapping**: Extract specific DOM subtrees using standard CSS selectors via BeautifulSoup4 to isolate precise elements.
* **Anti-Flapping & Whitespace Normalization**: Intelligent cleaning rules strip dynamic attributes, dynamic whitespace, and dynamic noise to eliminate false-positive alerts.
* **Semantic Diff Engine**: Line-by-line and character-level unified diff generation highlighting added, deleted, and modified content across snapshots.
* **Flexible Execution Core**: Run background periodic checks via `APScheduler`, expose REST control mechanisms through `FastAPI`, or operate directly from an interactive `Rich` terminal UI.
* **Structured Webhook Alerts**: Automatic HTTP POST payloads sent to specified endpoints whenever verified content changes exceed configurable tolerance thresholds.

---

## Architecture & System Design

The architecture is divided into a background execution engine, isolated parsing and diffing utilities, lightweight SQLite persistent storage, and interface adapters (FastAPI and Rich CLI).

```
                            ┌────────────────────────┐
                            │    Target Websites     │
                            └───────────┬────────────┘
                                        │ (HTTP GET)
                                        ▼
 ┌────────────────┐         ┌────────────────────────┐
 │ APScheduler    ├────────►│     src/core.py        │
 │ Background     │ Trigger │ - Scraper & Parser     │
 │ Engine         │         │ - Noise Filter         │
 └────────────────┘         │ - Diff Engine (difflib)│
                            └───────────┬────────────┘
                                        │
                         ┌──────────────┴──────────────┐
                         ▼                             ▼
              ┌─────────────────────┐       ┌────────────────────┐
              │ SQLite Storage      │       │ Webhook Dispatcher │
              │ (Snapshots & Diffs) │       └────────────────────┘
              └──────────┬──────────┘
                         │
        ┌────────────────┴────────────────┐
        ▼                                 ▼
┌───────────────┐                 ┌───────────────┐
│  src/api.py   │                 │  src/cli.py   │
│ (FastAPI REST)│                 │  (Rich TUI)   │
└───────────────┘                 └───────────────┘
```

### Module Breakdown

| Module | Purpose |
| :--- | :--- |
| `src/models.py` | Pydantic domain schemas and request/response models for monitoring jobs, snapshots, diffs, and webhooks. |
| `src/core.py` | Core execution engine: HTTP fetching, DOM parsing with BeautifulSoup4, noise normalization, `difflib` calculation, SQLite persistence, and webhook delivery. |
| `src/api.py` | FastAPI application defining REST endpoints for managing jobs, manually triggering runs, and retrieving historical diffs. |
| `src/cli.py` | Rich terminal user interface for job lifecycle management, real-time diff rendering, and health monitoring. |
| `src/main.py` | System orchestration entrypoint that wires up APScheduler, launches the FastAPI server, and routes CLI commands. |

---

## Technology Stack

* **Language**: Python 3.10+
* **Web Framework & Data Validation**: FastAPI, Pydantic v2
* **HTML Parsing**: BeautifulSoup4 (`lxml` parser)
* **Scheduling**: APScheduler (Advanced Python Scheduler)
* **HTTP Client**: Requests
* **CLI & Formatting**: Rich
* **Persistence**: SQLite3
* **Testing Framework**: pytest

---

## Installation & Virtual Environment Setup

### Prerequisites
* Python 3.10 or higher
* `pip` package manager
* `git`

### Installation Steps

1. **Clone the Repository**
   ```bash
   git clone https://github.com/your-org/page-change-monitor.git
   cd page-change-monitor
   ```

2. **Create and Activate Virtual Environment**
   ```bash
   # On macOS/Linux
   python3 -m venv .venv
   source .venv/bin/activate

   # On Windows
   python -m venv .venv
   .venv\Scripts\activate
   ```

3. **Install Dependencies**
   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

---

## Environment Variables Configuration

Create a `.env` file in the root project directory based on `.env.example`:

```bash
cp .env.example .env
```

### Configuration Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `DATABASE_URL` | string | `sqlite:///./monitor.db` | SQLite connection string for persistent storage. |
| `DEFAULT_POLL_INTERVAL` | integer | `3600` | Default polling interval in seconds for newly created jobs. |
| `REQUEST_TIMEOUT` | integer | `10` | HTTP request timeout in seconds when scraping target pages. |
| `WEBHOOK_RETRY_COUNT` | integer | `3` | Number of retry attempts for failed webhook alert deliveries. |
| `LOG_LEVEL` | string | `INFO` | Logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`). |

---

## Usage Guide

### 1. Interactive Terminal Interface (CLI)

The CLI provides an interface built with `Rich` for viewing status tables and inspecting diffs directly in your terminal.

```bash
# List all active monitoring jobs
python -m src.main cli list-jobs

# Register a new monitoring job
python -m src.main cli create-job \
  --name "Pricing Tier Monitor" \
  --url "https://example.com/pricing" \
  --selector ".pricing-table" \
  --interval 1800 \
  --webhook "https://api.site.com/hooks/pricing"

# Manually trigger an immediate check
python -m src.main cli run-job --id "JOB_ID_HERE"

# View diff history for a specific job
python -m src.main cli view-diff --id "JOB_ID_HERE"
```

### 2. REST API Endpoints

Launch the REST server:

```bash
python -m src.main api --host 0.0.0.0 --port 8000
```

Access the automated OpenAPI documentation at `http://localhost:8000/docs`.

#### API Reference

##### Create Job
* **POST** `/api/v1/jobs`
```json
{
  "name": "Terms of Service",
  "url": "https://example.com/terms",
  "css_selector": "article.legal-content",
  "interval_seconds": 86400,
  "webhook_url": "https://hooks.slack.com/services/XXX/YYY/ZZZ"
}
```

##### List All Jobs
* **GET** `/api/v1/jobs`

##### Trigger Immediate Execution
* **POST** `/api/v1/jobs/{job_id}/run`

##### Get Latest Snapshot Diff
* **GET** `/api/v1/jobs/{job_id}/diffs/latest`

---

## Automated Testing Instructions

The test suite covers DOM parsing, whitespace normalization, diff generation accuracy, API route validation, and persistent storage operations.

### Running Tests

```bash
# Execute all tests
pytest

# Execute core logic tests only
pytest tests/test_core.py

# Execute API route tests only
pytest tests/test_api.py

# Run with verbose output and coverage metrics
pytest -vv --cov=src tests/
```

---

## Project Structure

```
page-change-monitor/
├── .env.example          # Sample environment variables configuration
├── .gitignore             # Git ignore rules for virtualenvs, DBs, and caches
├── requirements.txt       # Production and development dependencies
├── src/
│   ├── __init__.py        # Package initialization marker
│   ├── api.py             # FastAPI REST endpoints and route handlers
│   ├── cli.py             # Rich interactive command-line interface
│   ├── core.py            # Scraping, BeautifulSoup parsing, diffing, DB & webhooks
│   ├── main.py            # System launcher, scheduler bootstrapper & CLI entry
│   └── models.py          # Pydantic domain models and API schemas
└── tests/
    ├── __init__.py        # Test package marker
    ├── test_api.py        # Integration tests for FastAPI endpoints
    └── test_core.py       # Unit tests for core parsing, diffing, and storage
```

---

## Roadmap & Future Enhancements

- [ ] **Headless Browser Execution**: Integrate Playwright/Selenium for rendering dynamic, JavaScript-heavy Single Page Applications (SPAs).
- [ ] **LLM Dynamic Summarization**: Integrate OpenAI/Anthropic APIs to generate human-readable summaries of complex DOM diffs.
- [ ] **Visual Regression Support**: Capture side-by-side PNG screenshots and highlight visual modifications using canvas diffing.
- [ ] **Multi-Tenant Authorization**: Add JWT authentication and tenant isolation for multi-team SaaS deployments.
- [ ] **Distributed Queue Integration**: Support Celery and Redis as alternative backends to APScheduler for enterprise scalability.

---

## License

Distributed under the MIT License. See `LICENSE` for more information.