# Enrich any contractor list with license status, insurance expiry, and 12-month permit volume

Clay HTTP API column, plus curl / Python / JavaScript / n8n / Make equivalents and an MCP setup for Claude Desktop and Cursor.

Actor: **Building Permit History API — Contractor & Address Lookup (MCP-ready)**, Apify Store slug `permit-history-lookup-api`.

| Endpoint | URL |
|---|---|
| Lookup (GET or POST) | `https://RickyZ1X--permit-history-lookup-api.apify.actor/lookup` |
| MCP (Streamable HTTP, JSON-RPC) | `https://RickyZ1X--permit-history-lookup-api.apify.actor/mcp` |
| Auth header (both) | `Authorization: Bearer YOUR_APIFY_API_TOKEN` |

Get your token in Apify Console -> **Settings -> API & Integrations**. Calls are billed to *your* Apify account. The Apify free plan's monthly credit covers a few hundred test lookups at the recommended price; check the Store page for the current price.

Data sources: official US open-data permit feeds (Austin TX, San Diego County CA, NYC DOB, Seattle WA, Los Angeles CA, Chicago IL) and Washington State L&I contractor license, insurance, and bond records. License, insurance, and bond fields are populated for WA contractors today; more states are added monthly. Outside WA, `license` may be `null` while permit history is still returned.

---

## 1. Request and response contract

### Request

Send **any combination** of these fields. Send at least one of the first four.

| Field | Example | Notes |
|---|---|---|
| `licenseNumber` | `EXAMPCL123AB` | The most precise match. |
| `contractorName` | `Example Solar LLC` | Normalized name match. Legal suffixes and punctuation are ignored. |
| `address` | `100 Example St, Austin, TX 78701` | Job-site address. Returns permits pulled at that property. |
| `phone` | `+15125550142` | Any US format is accepted and normalized to E.164. |
| `monthsBack` | `12` | Default is 12. |

Batch (non-HTTP) runs of the Actor also accept `queries: [{...}, {...}]`, `sources` (default: all six), and `maxPermitsPerLookup` (default: 25).

### Response (synthetic example; all values are fictional)

```json
{
  "query": { "contractorName": "Example Solar LLC" },
  "window": { "since": "2025-11-07", "until": "2026-11-07", "months_back": 12 },
  "matched": true,
  "charged": true,
  "license": {
    "license_number": "EXAMPCL123AB",
    "license_status": "ACTIVE",
    "license_expiration_date": "2027-03-14",
    "insurance_expiration_date": "2026-12-01",
    "bond_cancel_date": null,
    "phone_e164": "+12065550147"
  },
  "permits_summary": {
    "total": 41,
    "sampled": 41,
    "last_12_months": 37,
    "by_category": { "solar": 29, "battery_storage": 11, "ev_charger": 3 },
    "by_source": { "seattle_wa": 30, "austin_tx": 7 },
    "first_issued": "2025-11-20",
    "last_issued": "2026-11-03"
  },
  "recent_permits": [
    {
      "source": { "source_id": "seattle_wa", "jurisdiction": "Seattle", "state": "WA", "source_url": "https://data.seattle.gov/..." },
      "permit": { "permit_number": "6000000-EX", "issued_date": "2026-11-03", "description": "Install 8.4 kW roof-mounted PV + 1 battery" },
      "property": { "address": "100 Example St", "city": "Seattle", "state": "WA", "zip": "98101" },
      "contractor": { "company_name": "EXAMPLE SOLAR LLC", "phone_e164": "+12065550147" },
      "specs": { "categories": ["solar", "battery_storage"], "solar_kw": 8.4, "battery_units": 1 }
    }
  ]
}
```

`recent_permits` items use the full v1 permit schema. They are trimmed above for readability.

---

## 2. Clay HTTP API column (copy-paste config)

Add a column, choose **HTTP API**, and fill in the following. POST with a JSON body is recommended because Clay then handles quoting and you avoid URL-encoding problems with names such as `A&B Roofing`.

| Setting | Value |
|---|---|
| **Method** | `POST` |
| **Endpoint** | `https://RickyZ1X--permit-history-lookup-api.apify.actor/lookup` |
| **Headers** | `Authorization`: `Bearer YOUR_APIFY_API_TOKEN`<br>`Content-Type`: `application/json` |
| **Body** | see below |
| **Only run if** | `{{License Number}}` is not empty **OR** `{{Company Name}}` is not empty. This avoids paying for empty lookups. |
| **Rate limit** | Start with about **5 requests/second**. Clay's HTTP column has a per-column rate-limit setting. |
| **Retry** | Enable retries on 429 / 5xx if your Clay plan exposes it. |
| **Auto-update** | Off at first. Run on 10 rows, check the output, then run the whole table. |

Body (replace the `{{...}}` tokens with your own Clay column names using the `/` column picker):

```json
{
  "licenseNumber": "{{License Number}}",
  "contractorName": "{{Company Name}}",
  "address": "{{Address}}",
  "phone": "{{Phone}}",
  "monthsBack": 12
}
```

Empty strings are ignored by the API, so leave unused tokens in place or delete them.

**GET alternative** (for tools that only build URLs; URL-encode each value):

```
https://RickyZ1X--permit-history-lookup-api.apify.actor/lookup?licenseNumber={{License Number}}&contractorName={{Company Name}}&address={{Address}}&monthsBack=12
```

### Fields to map into columns

After the first run, click the HTTP cell -> **Add as column** for each path below:

| New Clay column | Response path |
|---|---|
| License # | `license.license_number` |
| License status | `license.license_status` |
| License expires | `license.license_expiration_date` |
| Insurance expires | `license.insurance_expiration_date` |
| Bond cancel date | `license.bond_cancel_date` |
| Business phone (state record) | `license.phone_e164` |
| Permits (total) | `permits_summary.total` |
| Permits (last 12 mo) | `permits_summary.last_12_months` |
| Solar permits | `permits_summary.by_category.solar` |
| Battery permits | `permits_summary.by_category.battery_storage` |
| Roofing permits | `permits_summary.by_category.roofing` |
| HVAC permits | `permits_summary.by_category.hvac` |
| First permit | `permits_summary.first_issued` |
| Last permit | `permits_summary.last_issued` |
| Latest job | `recent_permits[0].permit.description` |
| Latest job city | `recent_permits[0].property.city` |
| Matched? (true = billed; false = free no-match) | `matched` |
| Top contractor name on matched permits | `permits_summary.top_contractors[0].company_name` |

Useful Clay formulas:

- Active installer: `{{Permits (last 12 mo)}} >= 10`
- Renewal window: `DATEDIF(TODAY(), {{Insurance expires}}, "D") <= 45`

### Cost and limits

- **Each call to `/lookup` is billed as one `lookup` Pay-Per-Event event** to the Apify account that owns the token. See the Store page for the current price. At a $0.015 starting price, enriching 2,000 rows costs about $30.
- Clay's own HTTP column credits, if any apply on your plan, are separate from Apify's charge.
- Calls go to a standby Actor that's kept warm, so typical latency is a few hundred ms to a few seconds. The first call after an idle period can be slower.
- Use **Only run if** and Clay's dedupe so you don't pay twice for the same contractor.
- HTTP `401`/`403` means the token is bad or missing. A `4xx` with an error message usually means no lookup field was provided. `429` means you should slow down by lowering the column rate limit.

---

## 3. Equivalents outside Clay

### curl

```bash
# GET
curl -s -G "https://RickyZ1X--permit-history-lookup-api.apify.actor/lookup" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  --data-urlencode "contractorName=Example Solar LLC" \
  --data-urlencode "monthsBack=12"

# POST
curl -s -X POST "https://RickyZ1X--permit-history-lookup-api.apify.actor/lookup" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"licenseNumber":"EXAMPCL123AB","monthsBack":12}'

# Batch mode (no standby): run the Actor and get dataset items
curl -s -X POST "https://api.apify.com/v2/acts/RickyZ1X~permit-history-lookup-api/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"queries":[{"contractorName":"Example Solar LLC"},{"address":"100 Example St, Austin, TX 78701"}],"monthsBack":12}'
```

### Python (requests)

```python
import os
import requests

BASE = "https://RickyZ1X--permit-history-lookup-api.apify.actor"
HEADERS = {"Authorization": f"Bearer {os.environ['APIFY_TOKEN']}"}

def permit_history(**fields):
    """fields: licenseNumber, contractorName, address, phone, monthsBack"""
    r = requests.post(f"{BASE}/lookup", json=fields, headers=HEADERS, timeout=60)
    r.raise_for_status()
    return r.json()

res = permit_history(contractorName="Example Solar LLC", monthsBack=12)
lic = res.get("license") or {}
print(lic.get("license_status"), lic.get("insurance_expiration_date"),
      res["permits_summary"]["last_12_months"])
```

### JavaScript (fetch; Node 18+ or browser)

```js
const BASE = "https://RickyZ1X--permit-history-lookup-api.apify.actor";

async function permitHistory(fields) {
  const res = await fetch(`${BASE}/lookup`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.APIFY_TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(fields),
  });
  if (!res.ok) throw new Error(`lookup failed: ${res.status}`);
  return res.json();
}

const r = await permitHistory({ licenseNumber: "EXAMPCL123AB", monthsBack: 12 });
console.log(r.license?.license_status, r.permits_summary.last_12_months);
```

### n8n HTTP Request node (v4.2)

Paste this into the n8n canvas with Ctrl/Cmd+V. Create a **Header Auth** credential with Name `Authorization` and Value `Bearer YOUR_APIFY_API_TOKEN`.

```json
{
  "nodes": [
    {
      "parameters": {
        "method": "POST",
        "url": "https://RickyZ1X--permit-history-lookup-api.apify.actor/lookup",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ licenseNumber: $json.licenseNumber || '', contractorName: $json.companyName || '', address: $json.address || '', monthsBack: 12 }) }}",
        "options": { "batching": { "batch": { "batchSize": 5, "batchInterval": 1000 } } }
      },
      "id": "7b1c2d30-0001-4a00-9000-00000000c1a1",
      "name": "Permit history lookup",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [0, 0]
    }
  ],
  "connections": {}
}
```

Downstream, reference `{{ $json.license.license_status }}`, `{{ $json.permits_summary.last_12_months }}`, and so on.

### Make HTTP module ("HTTP -> Make a request")

| Field | Value |
|---|---|
| URL | `https://RickyZ1X--permit-history-lookup-api.apify.actor/lookup` |
| Method | `POST` |
| Headers | `Authorization` = `Bearer YOUR_APIFY_API_TOKEN` |
| Body type | Raw |
| Content type | JSON (application/json) |
| Request content | `{"contractorName":"{{1.company_name}}","licenseNumber":"{{1.license_number}}","monthsBack":12}` |
| Parse response | Yes |

Map the outputs as `{{2.data.license.license_status}}`, `{{2.data.permits_summary.last_12_months}}`, and so on, where module 2 is this HTTP module.

---

## 4. MCP setup (Claude Desktop, Cursor, and other MCP clients)

The Actor exposes one MCP tool, **`permit_history_lookup`**, with the arguments `licenseNumber`, `contractorName`, `address`, `phone`, and `monthsBack`. Each tool call is billed as one `lookup` event.

Ready-made config: [`mcp_config.json`](./mcp_config.json). It contains two alternative entries. Copy only the one your client needs.

### Claude Desktop

1. Install Node.js 18+ (needed for `npx`).
2. Open **Settings -> Developer -> Edit Config**. This opens `claude_desktop_config.json`:
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
3. Merge in the `permit-history` entry. It uses the `mcp-remote` bridge, which turns stdio into Streamable HTTP:

```json
{
  "mcpServers": {
    "permit-history": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://RickyZ1X--permit-history-lookup-api.apify.actor/mcp",
        "--header", "Authorization: Bearer ${APIFY_TOKEN}"
      ],
      "env": { "APIFY_TOKEN": "YOUR_APIFY_API_TOKEN" }
    }
  }
}
```

4. Fully quit and restart Claude Desktop. Then ask, for example: *"Use permit_history_lookup to show permits pulled by license EXAMPCL123AB in the last 12 months and tell me when their insurance expires."*

**Windows tip:** some clients mangle spaces inside `args`. If auth fails, use `"--header", "Authorization:${AUTH_HEADER}"` with `"env": {"AUTH_HEADER": "Bearer YOUR_APIFY_API_TOKEN"}`.

### Cursor

Cursor supports remote Streamable HTTP servers natively. Add the `permit-history-http` entry to `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (project):

```json
{
  "mcpServers": {
    "permit-history-http": {
      "url": "https://RickyZ1X--permit-history-lookup-api.apify.actor/mcp",
      "headers": { "Authorization": "Bearer YOUR_APIFY_API_TOKEN" }
    }
  }
}
```

If your Cursor version doesn't support `headers`, use the `npx mcp-remote` entry instead. It works in every client that can launch a stdio server.

The `"type": "http"` key in `mcp_config.json` is what Claude Code and VS Code expect for native HTTP servers. Clients that don't use it can drop it. Never commit a real token. Prefer environment variables or your client's secret storage.

### Quick protocol check (curl)

```bash
curl -s -X POST "https://RickyZ1X--permit-history-lookup-api.apify.actor/mcp" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

---

## Compliance

- **B2B use only.** Responses contain contractor business data from public permit filings and state licensing records: company name, license, insurance and bond dates, and business phone. They contain **no homeowner names or homeowner contact data**. Job-site addresses are public permit locations, and you should not use them to target residents.
- **You, the user, are responsible** for complying with the TCPA, the National Do-Not-Call Registry, state telemarketing and mini-TCPA laws, CAN-SPAM, and any other outreach rules before calling, texting, or emailing anyone found through this API.
- Insurance, bond, and license dates reflect the state record at fetch time. Verify them before acting on them.
