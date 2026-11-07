# US Building Permits & Contractor License Automation Workflows (n8n, Make, Clay, MCP)

Free, ready-to-import **n8n workflows**, **Make.com blueprints**, **Clay HTTP API column configs**, and **Claude/Cursor MCP configurations** for automating B2B construction, solar, battery storage, roofing, HVAC, and commercial insurance lead workflows from official US municipal open-data portals.

---

## Live Apify Store APIs Powering These Templates

All workflows in this repository connect to three normalized, schema-validated, zero-PII data feeds published on the Apify Store:

| # | Apify Actor | Best For | Store Link |
|---|---|---|---|
| **1** | **Building Permit Leads + Contractor Phone (Solar, Roofing, HVAC)** | Daily new municipal building permits across 6 US jurisdictions (Austin TX, San Diego County CA, NYC DOB, Seattle WA, Los Angeles CA, Chicago IL) with extracted job specs (`solar_kw`, `battery_kwh`, `roof_squares`, `hvac_tons`) and verified E.164 contractor phones. | [**Open on Apify Store →**](https://apify.com/rickyz1x/building-permits-contractor-leads) |
| **2** | **Contractor License, Insurance & Bond Expiration Radar** | Active contractors whose general liability insurance policy, surety bond, or state contractor license expires in the next 30/60/90 days (WA L&I official data), including carrier, policy number, surety, and phone. | [**Open on Apify Store →**](https://apify.com/rickyz1x/contractor-license-insurance-bond-radar) |
| **3** | **Contractor & Address Permit History API (MCP-ready)** | On-demand contractor KYB verification and 12-month permit volume lookup by license number, company name, phone, or job-site address. Supports Batch, HTTP Standby (`/lookup`), and Model Context Protocol (`/mcp`). | [**Open on Apify Store →**](https://apify.com/rickyz1x/permit-history-lookup-api) |

---

## Included Free Templates

### 1. n8n Workflows (`templates/n8n/`)
- **[`solar-battery-permit-leads.json`](./templates/n8n/solar-battery-permit-leads.json)** — Runs every morning at 07:00, pulls new solar PV & battery storage permits with a verified contractor phone (`onlyNew: true`), appends flattened rows to Google Sheets, and posts a daily Slack digest sorted by system size (`solar_kw`).
- **[`contractor-insurance-expiry-radar.json`](./templates/n8n/contractor-insurance-expiry-radar.json)** — Runs every Monday at 07:00, fetches active contractors whose liability insurance policy, surety bond, or license expires within 30 days, logs them to Google Sheets, and sends an urgent renewal outreach digest to Slack.
- **[`contractor-permit-history-enrichment.json`](./templates/n8n/contractor-permit-history-enrichment.json)** — Reads a list of contractors from Google Sheets and enriches each row with 12-month permit counts, category breakdown, project valuation total, and state license/insurance status.

### 2. Make.com Scenario Blueprint (`templates/make/`)
- **[`insurance-expiry-to-crm.blueprint.json`](./templates/make/insurance-expiry-to-crm.blueprint.json)** & **[Setup Guide](./templates/make/README.md)** — 7-module Make scenario that pulls expiring contractor insurance policies and surety bonds weekly and creates renewal outreach tasks in **HubSpot** or **Pipedrive**.

### 3. Clay HTTP Column & Claude/Cursor MCP Config (`templates/clay/`)
- **[`templates/clay/README.md`](./templates/clay/README.md)** — Copy-paste Clay HTTP API column configuration, JSONPath response mappings, and scoring formulas.
- **[`templates/clay/mcp_config.json`](./templates/clay/mcp_config.json)** — Ready-to-use `claude_desktop_config.json` / `.cursor/mcp.json` snippet for querying live contractor permit history from Claude Desktop or Cursor.

### 4. Measured Coverage & Unified JSON Schemas
- **[`COVERAGE.md`](./COVERAGE.md)** — Live measured 30-day permit volumes, contractor phone fill rates, license fill rates, and golden-set spec extraction precision across all sources.
- **[`schema/permit_v1.schema.json`](./schema/permit_v1.schema.json)** & **[`schema/contractor_v1.schema.json`](./schema/contractor_v1.schema.json)** — Formal JSON Schemas for the `v1` permit and contractor records.

---

## Compliance & Privacy Notice
B2B use only. All records are sourced from official public municipal permit portals and state contractor licensing boards and contain **no homeowner names or consumer contact data**. Users are responsible for complying with the TCPA, the National Do-Not-Call Registry, state telemarketing laws, and CAN-SPAM in any outreach.
