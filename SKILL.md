---
name: buyingunits
description: "Automated buying-unit mapping for enterprise software sales. Discovers subsidiaries, brands, divisions, acquisitions, and other buying units for any company and persists to Neon Postgres. Use when user wants to find buying units, map org structure, or analyze enterprise account opportunity."
---

# Buying Units Mapper

Map the full buying-unit landscape for any enterprise company — subsidiaries, brands, business units, geographies, acquisitions, joint ventures, divisions, and regional entities — to understand total addressable opportunity.

## Triggers

Activate this skill when the user says any of:
- "buyingunits" followed by one or more company names
- "find buying units for X"
- "how many buying units does X have"
- "map X's org structure"
- "buying unit analysis for X"
- Questions about previously saved buying unit data (triggers Query/Dashboard mode)

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FMP_API_KEY` | Optional | Financial Modeling Prep API key for M&A data |

## Neon Database

- **Project:** `buyingunits` (Neon project ID: `little-art-52445507`)
- **Database:** `neondb` (default)
- Always use the Neon MCP tools (`mcp__Neon__run_sql`, `mcp__Neon__run_sql_transaction`) with `project_id: "little-art-52445507"` for all database operations.

### Schema

Tables: `companies`, `buying_units`, `review_actions`, `research_runs`

## Mode Detection

Determine which mode to run based on user input:

- **Research mode**: User provides company name(s) to analyze → discover and save buying units
- **Query/Dashboard mode**: User asks about previously saved data → query DB and generate reports

---

## Research Mode Workflow

### Input
One or more company names (e.g., "buyingunits Nike, Unilever, P&G")

### Step 1 — Confirm Company Identity

For **each** company name provided:

1. **WebSearch** for "{company name} company" to find official info
2. Present to user via **AskUserQuestion**:
   - Legal name
   - Ticker symbol (if public)
   - HQ location
   - Industry
   - Ask: "Is this the right company?"
3. **Confirm ALL companies** before starting any research

### Step 2 — Check Database

For each confirmed company, check if it already exists:

```sql
SELECT c.id, c.name, c.analyzed_at,
  (SELECT COUNT(*) FROM buying_units WHERE company_id = c.id) as bu_count
FROM companies c WHERE LOWER(c.name) = LOWER($1);
```

If found, use **AskUserQuestion**: "I found {name} with {count} buying units (last analyzed {date}). Would you like to refresh the data or view existing?"
- **Refresh**: Re-research and merge new findings with existing data
- **View existing**: Skip to Step 5 (present existing data)

### Step 3 — Parallel Research

Research **all companies simultaneously** using the **Agent tool** to launch parallel research agents. For each company, query these sources in order of reliability:

#### 3a. SEC EDGAR (Confidence: high)
- **WebFetch** `https://efts.sec.gov/LATEST/search-index?q=%22{company_name}%22&dateRange=custom&startdt={last_year}&enddt={today}&forms=10-K`
- Find the latest 10-K filing
- **WebFetch** the filing index page to locate Exhibit 21 (Subsidiaries of the Registrant)
- Parse subsidiary names and jurisdictions of incorporation
- If no 10-K found (private company), skip gracefully

#### 3b. GLEIF LEI (Confidence: high)
- **WebFetch** `https://api.gleif.org/api/v1/lei-records?filter[entity.legalName]={company_legal_name}&page[size]=1`
- Extract the LEI from the response
- **WebFetch** `https://api.gleif.org/api/v1/lei-records/{LEI}/ultimate-parent-relationship`
- **WebFetch** `https://api.gleif.org/api/v1/lei-records?filter[entity.legalAddress.country]={country}&filter[relationship.managingLOU]={LEI}&page[size]=100`
- Build parent-child hierarchy from relationship data

#### 3c. FMP — Financial Modeling Prep (Confidence: medium)
Only if `FMP_API_KEY` environment variable is set:
- **WebFetch** `https://financialmodelingprep.com/api/v3/mergers-acquisitions/search?name={company_name}&apikey={FMP_API_KEY}`
- Extract acquisition targets, dates, and deal values
- Each acquisition becomes a buying unit with type `acquisition`

#### 3d. Web Search (Confidence: low-medium)
Run multiple **WebSearch** queries:
- "{company} subsidiaries list"
- "{company} brands portfolio"
- "{company} organizational structure divisions"
- "{company} regional offices locations"
- "{company} acquisitions history"
- "{company} business units segments"

#### 3e. WebFetch Follow-up (Confidence: varies)
From search results, **WebFetch** the most promising URLs:
- Investor relations / annual report pages
- Wikipedia organization/subsidiary sections
- Company "about us" / "our brands" / "our businesses" pages
- Press releases about acquisitions or restructuring

### Step 4 — Build Hierarchy

After all sources are queried:

1. **Deduplicate** across sources — match by name similarity (case-insensitive, trim suffixes like "Inc.", "LLC", "Ltd.")
2. **Build parent-child tree** — use GLEIF relationships as primary hierarchy, supplement with SEC/web data
3. **Classify each entity** with:

| Field | How to determine |
|-------|-----------------|
| `type` | `subsidiary` (SEC/GLEIF), `brand` (consumer-facing name), `business_unit` (internal segment), `geography` (regional entity), `acquisition` (FMP/news), `joint_venture` (ownership <100%), `division` (org chart), `regional_entity` (geographic office) |
| `confidence` | `high` (SEC EDGAR, GLEIF), `medium` (FMP, official website), `low` (web search, inferred) |
| `tax_types` | Infer from jurisdiction: US → SUT, EU → VAT, India → GST, etc. |
| `tax_complexity_notes` | E.g., "VAT registered in 12 EU countries", "Nexus in 38 US states" |
| `revenue_estimate` | From annual reports, press releases, or estimates |
| `employee_count` | From company pages, LinkedIn data in search results |
| `hq_location` | From GLEIF address, company website, or search results |
| `industry_vertical` | From company description or parent industry |
| `source_url` | URL where this entity was discovered |
| `source_name` | 'SEC EDGAR', 'GLEIF', 'FMP', 'Web Search', 'Company Website' |

### Step 5 — Present and Review

Present results for **one company at a time**, sequentially:

#### 5a. Hierarchical Tree View
Display an indented text tree showing parent-child relationships:

```
Nike, Inc. (NYSE: NKE)
├── Nike Brand
│   ├── Nike North America (geography) [high confidence]
│   ├── Nike EMEA (geography) [high confidence]
│   │   ├── Nike European Operations Netherlands BV (subsidiary) [high - SEC]
│   │   └── Nike Retail BV (subsidiary) [high - SEC]
│   └── Nike APLA (geography) [high confidence]
├── Converse (brand) [high - SEC]
│   └── Converse Inc. (subsidiary) [high - SEC]
└── Jordan Brand (brand) [medium - web]
```

#### 5b. Summary Table
Show counts by type and tax complexity highlights in a formatted table.

#### 5c. User Review
Use **AskUserQuestion** to ask the user to review:
- "Review the buying units above. What would you like to do?"
- Options:
  - "Confirm all — looks good"
  - "Remove some — I'll specify which"
  - "Add missing ones — I'll provide details"
  - "Edit classifications — I'll specify changes"
  - "Tag customer status — I'll mark which are customers"

Process each review action:
- **Confirmed**: Set confidence to `confirmed` for all
- **Remove**: Ask which to remove (by number/name), delete from results, log to `review_actions`
- **Add**: Ask for name, type, parent; add to results, log to `review_actions`
- **Edit**: Ask what to change, update, log to `review_actions`
- **Tag status**: Present list, ask user to mark as `customer`, `prospect`, or `not_applicable`

Log every action:
```sql
INSERT INTO review_actions (buying_unit_id, action, previous_values)
VALUES ($1, $2, $3);
```

### Step 6 — Persist to Database

After user confirms:

1. Insert/update `companies` record
2. Insert all `buying_units` with full enrichment fields
3. Record research run:
```sql
INSERT INTO research_runs (company_id, sources_queried, buying_units_found)
VALUES ($1, $2, $3);
```

### Step 7 — Cross-Company Summary

After all companies are reviewed and saved, show an aggregate summary:
- Total buying units across all companies
- Breakdown by type
- Tax complexity overview

---

## Query/Dashboard Mode

Triggered when the user asks questions about previously saved data. Examples:
- "which prospects have the most buying units?"
- "show me all buying units with VAT exposure"
- "coverage gap analysis"

### Capabilities

1. **Rank prospects** by buying unit count:
```sql
SELECT c.name, COUNT(bu.id) as bu_count
FROM companies c JOIN buying_units bu ON bu.company_id = c.id
GROUP BY c.name ORDER BY bu_count DESC;
```

2. **Filter by tax regime**:
```sql
SELECT c.name, bu.name, bu.tax_types, bu.tax_complexity_notes
FROM buying_units bu JOIN companies c ON bu.company_id = c.id
WHERE bu.tax_types ILIKE '%VAT%';
```

3. **Coverage gap analysis** (non-customer buying units):
```sql
SELECT c.name as company, bu.name as buying_unit, bu.type, bu.customer_status
FROM buying_units bu JOIN companies c ON bu.company_id = c.id
WHERE bu.customer_status != 'customer'
ORDER BY c.name, bu.name;
```

4. **Custom natural language queries** — Translate the user's question into SQL using the schema above

### Output Formats

Always provide a **chat summary** first with the key findings.

Then ask via **AskUserQuestion**: "Would you like an export?"
- **CSV export**: Generate a CSV file and save to a location the user specifies (default: `~/Downloads/buyingunits_export.csv`)
- **HTML report**: Generate a styled HTML report with tables and save to `~/Downloads/buyingunits_report.html`

#### HTML Report Template
When generating HTML reports, use clean styling with:
- A header with the report title and generation date
- Styled tables with alternating row colors
- Summary statistics at the top
- Responsive layout

#### CSV Export
Use standard CSV format with headers. Include all relevant columns for the query results.

---

## Important Guidelines

1. **Always use AskUserQuestion** for user interactions — never pose questions as plain text
2. **Always use Neon MCP tools** for database operations with `project_id: "little-art-52445507"`
3. **Parallelize research** — Use Agent tool to research multiple companies simultaneously
4. **Be transparent about confidence** — Always show the confidence level and source for each buying unit
5. **Handle errors gracefully** — If a data source is unavailable, skip it and note which sources were used
6. **Respect rate limits** — Space out WebFetch calls; don't hammer APIs
7. **Deduplicate carefully** — The same entity may appear in multiple sources with different names
8. **Preserve user edits** — When refreshing data, merge new findings but never overwrite user-confirmed data
