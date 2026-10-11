# buyingunits

Automated buying-unit mapping for enterprise software sales. A [Claude Code](https://claude.com/claude-code) skill that discovers subsidiaries, brands, divisions, acquisitions, and other buying units for any company and persists results to Neon Postgres.

## What It Does

Enterprise software companies sell to large accounts that appear as a single "logo" (e.g., Nike) but are actually made up of many independent **buying units** — subsidiaries, brands, business units, geographies — each potentially a separate purchaser.

This skill automates mapping the full buying-unit landscape:

1. **Research** — For any target company, queries SEC EDGAR, GLEIF, Financial Modeling Prep, and web sources to discover all buying units
2. **Hierarchy building** — Deduplicates, classifies, and builds parent-child relationships
3. **Interactive review** — Presents findings for user confirmation, editing, and customer-status tagging
4. **Query/dashboard mode** — Analyze saved data, generate CSV exports and HTML reports

## Data Sources

| Source | Cost | What It Provides |
|--------|------|-----------------|
| SEC EDGAR | Free | Legal subsidiary lists from Exhibit 21 (10-K filings) |
| GLEIF LEI | Free | Official corporate parent/child hierarchy |
| FMP API | Key required | M&A deal history with dates and values |
| Web Search | Free | Brands, divisions, org structure, M&A news |
| Company websites | Free | Brand portfolio, investor relations data |

## Installation

### Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI installed
- Neon MCP server configured in Claude Code (the `buyingunits` Neon project is pre-provisioned)

### Install the skill

```bash
# Clone the repo
git clone https://github.com/RyLoki/buyingunits.git

# Symlink into Claude Code skills directory
ln -s "$(pwd)/buyingunits" ~/.claude/skills/buyingunits
```

Or if you already cloned it elsewhere:

```bash
ln -s /path/to/buyingunits ~/.claude/skills/buyingunits
```

### Optional: FMP API Key

For M&A data from Financial Modeling Prep, set the environment variable:

```bash
export FMP_API_KEY="your_api_key_here"
```

## Usage

### Research Mode

```
buyingunits Nike
buyingunits Nike, Unilever, P&G
find buying units for Nike
map Nike's org structure
```

### Query Mode

```
which prospects have the most buying units?
show me all buying units with VAT exposure
coverage gap analysis
```

## Database

Uses Neon Serverless Postgres with the following tables:

- `companies` — Target/prospect companies
- `buying_units` — Discovered buying units with enrichment
- `review_actions` — Audit trail of user review decisions
- `research_runs` — Research run metadata

## License

MIT
