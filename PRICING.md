# PRICING.md: competitor benchmark and Pay-Per-Event (PPE) configuration

> **Snapshot date: 2026-11-07** (pulled 02:4x UTC from the public Apify Store API).
> Prices, user counts and run counts below are a **point-in-time snapshot**. Competitors change prices, and the Store search ranking shifts daily. Re-pull before any pricing decision:
> `curl -s "https://api.apify.com/v2/store?search=building%20permits&limit=50"`

---

## 1. How the benchmark was pulled

- **Searches** (Store API `GET https://api.apify.com/v2/store?search=<term>&limit=50`): `building permits`, `permit`, `contractor license`, `contractor`, `license lookup`. Each search returns at most 50 items. The searches are fuzzy: the server reported `total` = 829, 2,464, 865, 7,154 and 2,650 respectively. Across the five result pages there were **149 unique Actors**.
- **Kept:** Actors whose title is about US building permits (including BuildZoom permit/contractor data) or US contractor license lookup/verification. **Dropped:** Houzz, HomeAdvisor and Google-Maps-style "contractor email" lead scrapers, attorney/real-estate/insurance-agent/CPA/NMLS licenses, short-term-rental permits, non-US (Canada, Dubai, Singapore, Czech), oil-well permits, TTB permits, business-entity search, and "directory" listings. That leaves **50 permit Actors and 51 contractor-license Actors**.
- **Where prices live in the JSON:** `items[].currentPricingInfo.pricingModel`, and for PPE `items[].currentPricingInfo.pricingPerEvent.actorChargeEvents.<event-name>` with either
  - `eventPriceUsd` (flat price per event), or
  - `eventTieredPricingUsd.{FREE,BRONZE,SILVER,GOLD,PLATINUM,DIAMOND}.tieredEventPriceUsd`, a price that depends on the **customer's** Apify plan tier.
  Each event also carries `eventTitle`, `eventDescription`, `isPrimaryEvent` (the headline price on the Store card) and `isOneTimeEvent`.
- **Usage stats:** `items[].stats.totalUsers`, `totalUsers30Days`, `totalRuns` (also `totalUsers7Days`, `totalUsers90Days`, `publicActorRunStats30Days`).
- **Actor detail** (`GET https://api.apify.com/v2/acts/<username>~<name>`) returns `pricingInfos`, an **array holding the full pricing history** with `startedAt`, `reasonForChange`, and `isPPEPlatformUsagePaidByUser`, instead of only the current price.
- **Normalization:** "Primary $/1k" is the primary event's flat price × 1,000, or the FREE-tier price × 1,000 when the price is tiered. If no event is flagged primary, the most expensive per-result event is used. Per-run events (Actor start, per-search, per-scan, rollup) are shown as "$X each". Apify's default synthetic `apify-actor-start` at $0.00005 and the $0.00001 placeholder dataset-item events are omitted as noise.

### API structure surprises worth knowing

1. **Every one of the 149 Actors returned is `PAY_PER_EVENT`.** No rental (flat monthly) and no legacy pay-per-result pricing appeared in these niches. PPE is the norm, not a differentiator.
2. Many competitors don't define custom events. They price Apify's **synthetic `apify-default-dataset-item`** event, which Apify charges automatically for each dataset item. Our Actors charge **custom** events via `Actor.charge()`. We must therefore **not** also put a price on `apify-default-dataset-item`, or users would be double-charged.
3. **Tiered (per customer plan) prices** are common: FREE → DIAMOND, typically 0–30% cheaper at higher tiers, and up to about 75% cheaper in a few cases. They live under `eventTieredPricingUsd`, not `eventPriceUsd`, so naive scrapers of the API miss them.
4. `apifyMarginPercentage` is **0.2 on every Actor**, which confirms Apify's 20% commission.
5. `isPPEPlatformUsagePaidByUser` exists. It is `true` on 9/149 Actors (the user pays compute on top of event prices), but the Store search payload omits the key for most Actors. For `intelscrape~building-permit-scraper`, the detail endpoint shows `true` on the older pricing entries and no flag on the current one. Treat "absent" as "developer pays platform usage".
6. Pricing history is public (`pricingInfos[].reasonForChange`). Competitors and users can see every price change you make and your stated reason, which is one more reason to set prices correctly before the first publish.

---

## 2. Market snapshot (relevant Actors only)

| Segment | # Actors | Sum of 30-day users* | Sum of all-time users* | Primary $/1k: median | min | max |
|---|---|---|---|---|---|---|
| Building permits | 50 | 126 | 696 | **$4** | $0.40 | $100 |
| Contractor license lookup | 51 | 280 | 1,592 | **$4** | $0.04 | $80 |

\*These are sums of per-Actor counts. The same user can be counted in several Actors, so they are not unique users.

**Key readings**

- **Permit records are a commodity at $2–10 per 1k.** The most-used permit Actor (`pink_comic`, 23 users in 30 days) charges **$2/1k**. The cluster sits at $3–4/1k for single-city "No Login" Actors (`velvety_bedbug`, `fortuitous_pirate` $3.5) and $8–10/1k for multi-city Actors. Products that claim to be lead-grade sell at $15–40/1k: `orkanaltinsoy` qualified-lead $15, `permitdata` scored-lead $20, `4l3c` $30, `intimate_pacu` $30, `marielise.dev` permit-lead $40.
- **Contact enrichment carries a premium.** `intelscrape` charges `contractor-enrichment` at **$50/1k** (skip-trace phone/email) on top of $5/1k permits. `memo23` charges `contact-email` at $52/1k. `zapticx` charges `personnel-enrichment` at $15/1k.
- **Contractor-license lookups are dominated by one publisher.** `scrapebench` has **27 of the 51 license Actors and 181 of the 280 summed 30-day users**, almost all at a flat **$4/1k** `dataset-item`. That includes `scrapebench~washington-contractor-license-lookup` (WA L&I, 60 users all-time / 7 in 30 days, $4/1k), the same state source as our Actor #2. Their `contractor-license-expiration-monitor` charges $4/1k `license-checked` and covers **license** expiry only (Notion delivery). We found **no Actor with insurance-policy or surety-bond expiration alerts.**
- **Per-search pricing exists and is tolerated:** `scrapebench` SC/NM charge `search` at $0.02 plus `license-match` at $0.01. `glowing_glove` charges $0.025 per processed permit search. `cblu` charges `monitor-scan` at $0.50 and `license-verified` at $20/1k.
- **High prices do not get usage.** Actors at $25–100/1k (`ramsford` $100, `tagadanar` $80, `marielise.dev` $40, `parseforge` CSLB $25) have 0–2 users in the last 30 days. The ones with traffic are at $2–5/1k for raw rows.

---

## 3. Competitor tables (snapshot 2026-11-07, sorted by 30-day users)

"Users" means all-time users / users in the last 30 days. Prices are the FREE-tier (non-subscriber) price. Where a tier range exists it's shown as "to $X top tier". "+ platform usage paid by user" means the customer also pays compute.

### 3a. Building-permit Actors (50)

| # | Actor (username~name) | Title | Users (all-time / 30d) | Runs | PPE events (FREE-tier price) | Primary $/1k |
|---|---|---|---|---|---|---|
| 1 | `pink_comic~building-permits-construction-leads` | Building Permits Scraper - Contractor & Construction Leads | 162 / 23 | 2259 | `apify-default-dataset-item` $2/1k | $2 |
| 2 | `inexhaustible_glass~us-building-permits-scraper` | US Building Permits Scraper — Construction Leads | 53 / 12 | 264 | `apify-default-dataset-item` $8/1k; `apify-actor-start` $0.02 each | $8 |
| 3 | `handstands.io~us-building-permit-scraper` | US Building Permit Scraper — Construction Lead Gen | 47 / 12 | 423 | `apify-default-dataset-item` $10/1k | $10 |
| 4 | `hanamira~building-permits-search` | Building Permits Search | 66 / 8 | 868 | `apify-default-dataset-item` $4/1k | $4 |
| 5 | `4l3c~building-permits-scraper` | Building Permits Scraper — Construction Leads | 25 / 7 | 239 | `result-item` $30/1k | $30 |
| 6 | `jungle_synthesizer~building-permits-national-aggregator` | Building Permits Scraper - National Permit Aggregator | 16 / 3 | 381 | `apify-actor-start` $0.1 each; `data-record` $1/1k | $1 |
| 7 | `egra_van~accela-building-permits` | US Building Permits Scraper (Accela) | 6 / 3 | 62 | `permit` $10/1k (to $8.5/1k top tier); `permit-details` $10/1k | $10 |
| 8 | `scrapesage~buildzoom-scraper` | BuildZoom Scraper - Contractors, Licenses, Permits & Leads | 5 / 3 | 162 | `contractor` $4/1k (to $2.01/1k top tier); `contractorDetail` $7/1k (to $3.52/1k top tier); `review` $1/1k (to $0.51/1k top tier) | $4 |
| 9 | `automation_craft~us-building-permits-scraper` | US Building Permits Scraper - 38 City Sources | 4 / 3 | 121 | `permit-record` $5/1k (to $4/1k top tier) | $5 |
| 10 | `intelscrape~building-permit-scraper` | Building Permits Scraper — Contractor & Construction Leads | 113 / 2 | 1566 | `apify-default-dataset-item` $5/1k; `apify-actor-start` $0.05 each; `contractor-enrichment` $50/1k; `contractor-rollup` $0.01 each | $5 |
| 11 | `scrapesage~us-building-permits-scraper` | US Building Permits Scraper - Construction & Contractor Leads | 7 / 2 | 220 | `permit` $3/1k (to $0.75/1k top tier) | $3 |
| 12 | `automation-lab~hillsborough-county-building-permits-scraper` | Hillsborough County Building Permits Scraper | 4 / 2 | 151 | `start` $0.015 each; `item` $2.76/1k (to $0.672/1k top tier) | $2.76 |
| 13 | `fortuitous_pirate~nashville-building-permits` | Nashville Building Permits Scraper — No Login, $10/1k | 4 / 2 | 283 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $10/1k | $10 |
| 14 | `flpublicdata~florida-building-permits` | Florida Building Permits – Roof, Pool, A/C, Solar Leads | 3 / 2 | 23 | `permit-record` $3/1k | $3 |
| 15 | `getascraper~construction-permit-lead-monitor` | Building Permit Lead Monitor: Chicago, NYC, Austin | 3 / 2 | 105 | `apify-default-dataset-item` $0.88/1k (to $0.66/1k top tier) | $0.88 |
| 16 | `filingradar~us-building-permits` | US Building & Construction Permit Leads | 3 / 2 | 140 | `apify-default-dataset-item` $4/1k | $4 |
| 17 | `great_pistachio~us-building-permits` | US Building Permits (Daily Feed) | 3 / 2 | 100 | `apify-default-dataset-item` $8/1k | $8 |
| 18 | `formal_sadness_xvx~us-building-permits` | US Building Permits - 19 sources, one schema | 3 / 2 | 104 | `apify-actor-start` $0.001 each; `permit-record` $2/1k | $2 |
| 19 | `orkanaltinsoy~permit-contractor-lead-enricher` | Building Permit Lead Finder & Scorer | 3 / 2 | 145 | `qualified-lead` $15/1k (+ platform usage paid by user) | $15 |
| 20 | `orkanaltinsoy~us-building-permits-scraper` | US Building Permits Scraper (Multi-City) | 3 / 2 | 145 | `permit-record` $3/1k (+ platform usage paid by user) | $3 |
| 21 | `carranza-tech~us-building-permit-leads` | US Building Permit Leads — Multi-City | 3 / 2 | 103 | `apify-default-dataset-item` $4/1k | $4 |
| 22 | `velvety_bedbug~miami-building-permits` | Miami Building Permits — No Login, $3.75/1k | 3 / 2 | 178 | `apify-actor-start` $0.001 each; `apify-default-dataset-item` $4.687/1k | $4.687 |
| 23 | `velvety_bedbug~montgomery-al-building-permits` | Montgomery AL Building Permits — No Login, $3/1k | 3 / 2 | 160 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $3/1k | $3 |
| 24 | `scrapemint~building-permit-leads` | Building Permit Leads Scraper: New Permits by City & Trade | 3 / 2 | 122 | `permit_row` $10/1k | $10 |
| 25 | `velvety_bedbug~cape-coral-building-permits` | Cape Coral Building Permits — No Login, $3/1k | 3 / 2 | 158 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $3/1k | $3 |
| 26 | `velvety_bedbug~murfreesboro-tn-building-permits` | Murfreesboro TN Building Permits — No Login, $3/1k | 3 / 2 | 155 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $3/1k | $3 |
| 27 | `velvety_bedbug~fort-lauderdale-building-permits` | Fort Lauderdale Building Permits — No Login, $3/1k | 3 / 2 | 159 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $3/1k | $3 |
| 28 | `intimate_pacu~building-permit-contractor-leads-api` | Building Permit Contractor Leads API | 3 / 2 | 99 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $30/1k | $30 |
| 29 | `moving_beacon-owner1~nyc-dob-permits-scraper` | NYC DOB Permits Scraper | 3 / 2 | 106 | `apify-default-dataset-item` $9.99/1k | $9.99 |
| 30 | `fortuitous_pirate~boston-permits-scraper` | Boston Permits Scraper — No Login, $3.5/1k | 3 / 2 | 253 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $3.5/1k | $3.5 |
| 31 | `glowing_glove~local-permit-business-license-lead-finder` | Local Permit & Business License Lead Finder | 3 / 2 | 128 | `processedPermitSearch` $0.025 each | $25 |
| 32 | `fortuitous_pirate~dallas-building-permits` | Dallas Building Permits Scraper — No Login, $10/1k | 19 / 1 | 405 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $10/1k | $10 |
| 33 | `parseforge~nyc-building-permits-scraper` | NYC Building Permits Scraper | 11 / 1 | 297 | `apify-default-dataset-item` $3/1k (to $2.136/1k top tier); `apify-actor-start` $0.01 each (to $0.0089 top tier) | $3 |
| 34 | `fortuitous_pirate~seattle-building-permits` | Seattle Building Permits Scraper — No Login, $10/1k | 11 / 1 | 411 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $10/1k | $10 |
| 35 | `finaldynamics~building-permit-lookup-scraper` | Building Permit Scraper: Contractor Leads & Status (Accela) | 5 / 1 | 128 | `actor-start` $0.01 each; `permit-record` $6/1k; `contractor-detail-page` $4/1k | $6 |
| 36 | `bujhmml~building-permits-scraper` | Building Permits Scraper - Socrata Open Data (Multi-City) | 4 / 1 | 144 | `permit` $0.4/1k | $0.4 |
| 37 | `lulzasaur~building-permit-scraper` | Building Permit Search - Multi-City | 3 / 1 | 380 | `apify-default-dataset-item` $10/1k | $10 |
| 38 | `permitdata~us-building-permits-scraper` | US Building Permits Scraper - Contractor Leads, Verified Cities | 3 / 1 | 286 | `permit` $3/1k; `scored-lead` $20/1k | $3 |
| 39 | `velvety_bedbug~howard-county-building-permits` | Howard County Building Permits — No Login, $3/1k | 3 / 1 | 179 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $3/1k | $3 |
| 40 | `marielise.dev~us-construction-permit-intelligence` | US Construction Permit Intelligence | 22 / 0 | 288 | `permit-lead` $40/1k; `ai-permit-review` $15/1k; `ai-market-insight` $0.03 each | $40 |
| 41 | `jungle_synthesizer~buildzoom-contractor-permit-aggregator-scraper` | BuildZoom Contractor & Permit Scraper | 9 / 0 | 263 | `apify-actor-start` $0.1 each; `data-record` $2/1k | $2 |
| 42 | `fortuitous_pirate~building-permits-scraper` | Building Permits Scraper — No Login, $3.5/1k | 8 / 0 | 271 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $3.5/1k | $3.5 |
| 43 | `motivational_nickel~my-actor-1` | NYC Building Permits & Contractor Data Extractor (DOB) | 8 / 0 | 372 | `apify-default-dataset-item` $0.5/1k | $0.5 |
| 44 | `ramsford~building-permit-tracker` | Building Permit & Project Intelligence Tracker | 5 / 0 | 178 | `apify-default-dataset-item` $100/1k | $100 |
| 45 | `fortuitous_pirate~sandiego-building-permits` | SAN Diego Building Permits Scraper — No Login, $10/1k | 4 / 0 | 274 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $10/1k | $10 |
| 46 | `jungle_synthesizer~nyc-dob-hpd-violations-permits-property-scraper` | NYC DOB & HPD Violations, Permits & Property Data Scraper | 4 / 0 | 171 | `apify-actor-start` $0.1 each; `data-record` $2/1k | $2 |
| 47 | `fortuitous_pirate~cambridge-building-permits` | Cambridge Massachusetts Building Permits Scraper — $10/1k | 3 / 0 | 275 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $10/1k | $10 |
| 48 | `fortuitous_pirate~austin-building-permits` | Austin Building Permits Scraper — No Login, $10/1k | 3 / 0 | 324 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $10/1k | $10 |
| 49 | `velvety_bedbug~hillsborough-county-building-permits` | Hillsborough County Building Permits — No Login, $3/1k | 3 / 0 | 151 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $3/1k | $3 |
| 50 | `quarterly_jingo~nyc-building-permits` | NYC Building Permits — No Login, $10/1k | 2 / 0 | 304 | `apify-actor-start` $0.001 each; `apify-default-dataset-item` $10/1k | $10 |


### 3b. Contractor-license Actors (51)

| # | Actor (username~name) | Title | Users (all-time / 30d) | Runs | PPE events (FREE-tier price) | Primary $/1k |
|---|---|---|---|---|---|---|
| 1 | `pink_comic~cslb-contractor-license-search` | CSLB Contractor License Search - CA License KYB & Leads | 70 / 20 | 4962 | `apify-default-dataset-item` $2/1k | $2 |
| 2 | `haketa~az-roc-contractor-license-scraper` | Arizona ROC Contractor License Scraper | 78 / 18 | 1852 | `apify-default-dataset-item` $3.5/1k (to $2.5/1k top tier) | $3.5 |
| 3 | `scrapebench~california-contractor-license-lookup` | California Contractor License Lookup & Verify (CSLB) | 69 / 11 | 958 | `dataset-item` $4/1k | $4 |
| 4 | `scrapebench~michigan-contractor-license-lookup` | Michigan Contractor License Lookup & Verify (LARA) | 61 / 11 | 1406 | `dataset-item` $4/1k | $4 |
| 5 | `scrapebench~nevada-contractor-license-lookup` | Nevada Contractor License Lookup & Verify (NSCB) | 62 / 10 | 611 | `dataset-item` $4/1k | $4 |
| 6 | `scrapebench~florida-contractor-license-lookup` | Florida Contractor License Lookup & Verify (DBPR) | 75 / 9 | 1109 | `dataset-item` $4/1k | $4 |
| 7 | `scrapebench~minnesota-contractor-license-lookup` | Minnesota Contractor License Lookup & Leads Scraper (DLI) | 57 / 9 | 432 | `dataset-item` $4/1k | $4 |
| 8 | `getascraper~multistate-contractor-license-verifier` | Contractor License Verification: Multi-State | 10 / 9 | 187 | `apify-default-dataset-item` $3.52/1k (to $2.64/1k top tier) | $3.52 |
| 9 | `lulzasaur~oh-contractor-license-scraper` | Ohio Contractor License Search (OCILB) | 9 / 9 | 556 | `apify-default-dataset-item` $5/1k | $5 |
| 10 | `scrapebench~tennessee-contractor-license-lookup` | Tennessee Contractor License Lookup & Verify (TN DOCI) | 64 / 8 | 489 | `dataset-item` $4/1k | $4 |
| 11 | `scrapebench~texas-contractor-license-lookup` | Texas Contractor & Trade License Lookup & Export (TDLR) | 63 / 8 | 553 | `dataset-item` $4/1k | $4 |
| 12 | `scrapebench~alabama-contractor-license-lookup` | Alabama Contractor License Lookup & Verify (LBGC) | 59 / 8 | 505 | `dataset-item` $4/1k | $4 |
| 13 | `scrapebench~north-carolina-contractor-license-lookup` | North Carolina Contractor License Lookup & Verify (NCLBGC) | 56 / 8 | 456 | `dataset-item` $4/1k | $4 |
| 14 | `scrapebench~us-contractor-license-lookup` | US Contractor License Lookup & List Scraper (State Boards) | 13 / 8 | 279 | `license-verified` $4/1k | $4 |
| 15 | `scrapebench~south-carolina-contractor-license-lookup` | South Carolina Contractor License Lookup & Verify (SC LLR) | 67 / 7 | 507 | `license-match` $10/1k; `search` $0.02 each | $10 |
| 16 | `scrapebench~virginia-contractor-license-lookup` | Virginia Contractor License Lookup (DPOR Bulk) | 64 / 7 | 433 | `dataset-item` $4/1k | $4 |
| 17 | `scrapebench~washington-contractor-license-lookup` | Washington Contractor License Lookup & Verify (L&I) | 60 / 7 | 575 | `dataset-item` $4/1k | $4 |
| 18 | `scrapebench~oregon-contractor-license-lookup` | Oregon Contractor License Lookup & Verify (CCB) | 58 / 7 | 479 | `dataset-item` $4/1k | $4 |
| 19 | `scrapebench~louisiana-contractor-license-lookup` | Louisiana Contractor License Lookup & Leads Scraper (LSLBC) | 60 / 6 | 511 | `dataset-item` $4/1k | $4 |
| 20 | `scrapebench~ohio-contractor-license-lookup` | Ohio Contractor License Lookup & Leads Scraper (OCILB) | 6 / 6 | 54 | `dataset-item` $4/1k | $4 |
| 21 | `scrapebench~arkansas-contractor-license-lookup` | Arkansas Contractor License Lookup & Verify (ACLB) | 56 / 5 | 479 | `dataset-item` $4/1k | $4 |
| 22 | `cblu~florida-license-records-scraper` | Florida Contractor License Lookup (DBPR) — Bulk Verification | 19 / 5 | 196 | `license-record` $4/1k; `monitor-scan` $0.5 each; `monitored-license` $0.5/1k; `license-verified` $20/1k | $4 |
| 23 | `scrapebench~contractor-license-expiration-monitor` | Contractor License Expiration Monitor & Alerts (Notion) | 10 / 5 | 221 | `license-checked` $4/1k | $4 |
| 24 | `scrapebench~arizona-contractor-license-lookup` | Arizona Contractor License Lookup & Leads Scraper (ROC) | 6 / 5 | 56 | `dataset-item` $4/1k | $4 |
| 25 | `scrapebench~district-of-columbia-contractor-license-lookup` | DC Contractor License Lookup & List Scraper (DLCP) | 6 / 5 | 54 | `dataset-item` $4/1k | $4 |
| 26 | `scrapebench~delaware-contractor-license-lookup` | Delaware Contractor License Lookup & List Scraper (DOR + DPR) | 6 / 5 | 56 | `dataset-item` $4/1k | $4 |
| 27 | `scrapebench~new-jersey-contractor-license-lookup` | New Jersey Contractor License Lookup & List Scraper (DCA) | 5 / 5 | 62 | `dataset-item` $4/1k | $4 |
| 28 | `scrapebench~west-virginia-contractor-license-lookup` | West Virginia Contractor License Lookup & Leads Scraper (WVCLB) | 5 / 5 | 54 | `dataset-item` $4/1k | $4 |
| 29 | `scrapebench~multistate-contractor-license-lookup` | Multi-State Contractor & Trade License Lookup | 60 / 4 | 585 | `dataset-item` $4/1k | $4 |
| 30 | `scrapebench~new-mexico-contractor-license-lookup` | New Mexico Contractor License Lookup & List Scraper (NM CID) | 56 / 4 | 384 | `license-match` $10/1k; `search` $0.02 each | $10 |
| 31 | `automation-lab~arizona-roc-contractor-license-scraper` | Arizona ROC Contractor License Scraper | 9 / 4 | 410 | `start` $0.005 each; `contractor` $4.1447/1k (to $1.0091/1k top tier) | $4.1447 |
| 32 | `automation-lab~california-contractor-license-lookup-v2` | California Contractor Licenses Scraper | 9 / 4 | 143 | `start` $0.005 each; `item` $17.48/1k (to $4.256/1k top tier) | $17.48 |
| 33 | `scrapebench~maryland-contractor-license-lookup` | Maryland Contractor License Lookup & List Scraper (MHIC) | 4 / 4 | 51 | `dataset-item` $4/1k | $4 |
| 34 | `scrapebench~mississippi-contractor-license-lookup` | Mississippi Contractor License Lookup & Leads Scraper (MSBOC) | 4 / 4 | 56 | `dataset-item` $4/1k | $4 |
| 35 | `stackrelay~cslb-california-scraper` | CSLB California Contractor License Scraper | 30 / 3 | 6937 | `apify-default-dataset-item` $19/1k (+ platform usage paid by user) | $19 |
| 36 | `stackrelay~tdlr-texas-contractor-license-scraper` | TDLR Texas Contractor & Trade License Scraper | 20 / 3 | 815 | `apify-default-dataset-item` $10/1k (to $8.5/1k top tier); `actor_start` $0.004 each | $10 |
| 37 | `stackrelay~az-roc-license-scraper` | Arizona ROC Contractor License Scraper | 8 / 3 | 279 | `apify-default-dataset-item` $6.5/1k (to $5.5/1k top tier) | $6.5 |
| 38 | `automation-lab~north-carolina-contractor-license-lookup-scraper` | North Carolina Contractor License Lookup Scraper | 8 / 3 | 289 | `start` $0.005 each; `item` $0.041071/1k (to $0.01/1k top tier) | $0.041071 |
| 39 | `zapticx~california-cslb-contractor-scraper` | California CSLB Contractor Scraper — Licenses & Leads | 5 / 3 | 113 | `contractor-result` $2/1k; `personnel-enrichment` $15/1k | $2 |
| 40 | `parseforge~cslb-california-scraper` | California CSLB Contractor Scraper | 105 / 2 | 1498 | `result-item` $25/1k (to $21/1k top tier); `apify-actor-start` $0.005 each (to $0.00445 top tier) | $25 |
| 41 | `pink_comic~tdlr-texas-professional-license-search` | TDLR Texas License Search - Contractor & Pro License KYB | 14 / 2 | 1248 | `apify-default-dataset-item` $2/1k | $2 |
| 42 | `maximedupre~arizona-roc` | Arizona ROC Contractor License Scraper | 7 / 2 | 178 | `license-record` $2.7/1k | $2.7 |
| 43 | `avant_technologies~contractor-license-scraper` | US Contractor License Scraper — Multi-State Trade Registry | 5 / 2 | 148 | `apify-default-dataset-item` $2/1k | $2 |
| 44 | `scrapersdelight~florida-dbpr-license-scraper` | Florida Contractor License Scraper — DBPR Records | 4 / 2 | 166 | `license-scraped` $4/1k | $4 |
| 45 | `wellbuilt_zythem~us-contractor-license-lookup` | US Contractor License Scraper & Lookup - 20 States | 4 / 2 | 138 | `record` $5/1k (to $4.5/1k top tier); `detail` $12/1k (to $10.8/1k top tier) | $5 |
| 46 | `lulzasaur~contractor-license-scraper` | Contractor License Verification - CA, TX, FL, NY | 16 / 1 | 45848 | `apify-default-dataset-item` $20/1k | $20 |
| 47 | `tagadanar~contractor-license-leads` | New Contractor License Leads (CA, FL, TX) | 4 / 1 | 156 | `lead-found` $80/1k (to $56/1k top tier); `actor-start` $0.01 each | $80 |
| 48 | `lulzasaur~md-contractor-license-scraper` | Maryland Contractor License Search (MHIC) | 3 / 1 | 161 | `apify-default-dataset-item` $10/1k | $10 |
| 49 | `moving_beacon-owner1~contractors-state-license-board-cslb-scraper` | CSLB (Contractors State License Board) Scraper | 24 / 0 | 564 | `apify-default-dataset-item` $10/1k | $10 |
| 50 | `memo23~us-contractor-license-leads-scraper` | US Contractor License Leads Scraper (Multi-State) | 13 / 0 | 84 | `apify-actor-start` $0.005 each; `apify-default-dataset-item` $4/1k; `additional-data` $4/1k; `contact-email` $52/1k | $4 |
| 51 | `inexhaustible_glass~us-contractor-licenses` | US Contractor License Scraper — Multi-State Leads | 6 / 0 | 159 | `apify-default-dataset-item` $5/1k; `apify-actor-start` $0.02 each | $5 |

---

## 4. Recommended PPE configuration (set before the first publish)

**Charging semantics assumed below.** Events stack: every delivered record fires its base event, and a qualifying record also fires the premium event. For example, a permit with a contractor phone costs `permit-record` + `contact-enriched-record`. **Verified against the code:** `permit_intel_core.apify_compat.push_and_charge` charges the base event per delivered record and the add-on on top; Actor #3 charges `lookup` only when `matched` is true (`actors/permit-history-lookup/src/main.py`). If the implementation instead charges one or the other, set the premium price to the all-in price ($15/1k for #1, $22/1k for #2).

All Actors must respect the user's **max cost per run** (`ACTOR_MAX_TOTAL_CHARGE_USD`). `Actor.charge()` reports when the limit is reached, and the Actor must stop producing billable output at that point.

### Actor #1: Building Permit Leads + Contractor Phone (`building-permits-contractor-leads`)

| Event name | Title | Description (paste into Console) | Price | Primary |
|---|---|---|---|---|
| `permit-record` | Permit record | One normalized building permit (v1 schema, extracted job specs, delta flags) written to the dataset. Permits you already received are skipped when "Only new permits" is on, so you never pay twice for them. | **$0.003** ($3 / 1,000) | yes |
| `contact-enriched-record` | Contractor contact add-on | Charged in addition to Permit record, only when the record includes a valid contractor phone (E.164). Records with a license but no phone are not charged this add-on. | **$0.012** ($12 / 1,000) | no |
| `apify-actor-start` (synthetic) | Actor start | Apify's built-in start event (one per GB of memory, minimum one). | **$0.005** (per GB; default memory 1 GB) | no |

Why these prices:

- **$3/1k base.** This is below the $4 segment median and the $8–10 multi-city cluster, and above the $2 volume leader. Our rows carry extracted specs (kW, kWh, squares, tons) and delta tracking that the $2–4 Actors don't have, so we don't need to win on price. The Store card's headline then reads "$3.00 / 1,000 permit records".
- **+$12/1k for the contact add-on.** An enriched lead costs $15/1k all-in, which is under every lead-grade competitor ($15–40/1k) and far under skip-trace enrichment ($50/1k). Our phones come from the permit filing itself, not from skip-tracing, so our marginal cost is near zero and we can undercut while still earning about 5× the base price.
- **$0.005 start fee.** This covers the compute of empty `onlyNew` runs, since daily scheduled templates often return 0–10 rows. It's in line with competitors ($0.005–0.05). It's too small to matter on a real run.

### Actor #2: Contractor License, Insurance & Bond Expiration Radar (`contractor-license-insurance-bond-radar`)

| Event name | Title | Description | Price | Primary |
|---|---|---|---|---|
| `contractor-record` | Contractor record | One contractor written to the dataset with license status, insurance policy, and bond details (WA L&I in V1). | **$0.004** ($4 / 1,000), lowered from the $5 starting point | yes |
| `expiring-alert-record` | Expiration alert add-on | Charged in addition to Contractor record, only when the record has at least one alert (insurance expiring, bond cancelling, or license expiring) inside your window. | **$0.018** ($18 / 1,000) | no |
| `apify-actor-start` (synthetic) | Actor start | Apify's built-in start event. | **$0.005** | no |

Why these prices:

- **$4 instead of $5 for the base.** The direct same-source competitor (`scrapebench~washington-contractor-license-lookup`) and about 27 sister Actors sit at exactly **$4/1k**. Matching $4 means our Store card isn't the most expensive in a list where the cheapest option dominates, and our row is a superset (insurance plus bond plus days-to-expiry).
- **+$18/1k for the alert add-on.** An actionable renewal alert costs $22/1k all-in, which is 2.2¢ per alert. Nobody else sells insurance or bond expiry. The closest analogues are `cblu` license-verified at $20/1k plus $0.50 per scan, and `permitdata` scored-lead at $20/1k. For an insurance agent or surety broker, one renewal is worth hundreds of dollars, so the price is not the constraint. Coverage (WA only) is, and that is why we don't go higher in V1.
- With the default `includeNonExpiring: false`, almost every delivered row is an alert row. Users effectively pay $22/1k, and `maxRecords` (default 1,000) caps a default run at about $22.

### Actor #3: Contractor & Address Permit History API (`permit-history-lookup-api`)

| Event name | Title | Description | Price | Primary |
|---|---|---|---|---|
| `lookup` | Lookup | One lookup that matched at least one permit or license, whether a batch query, a standby `GET`/`POST /lookup` call, or an MCP `permit_history_lookup` tool call. It returns the license block, the permit summary, and up to `maxPermitsPerLookup` recent permits. No-match and invalid lookups are free. | **$0.015** | yes |
| `apify-actor-start` (synthetic) | Actor start | Apify's built-in start event. | leave at Apify default ($0.00005) | no |

Why these prices:

- **$0.015 per lookup.** That is between `scrapebench`'s `search` ($0.02) plus `license-match` ($0.01), which totals $0.03 per successful search, and the plain $0.01 per search Actors. It's cheap enough for 1,000–5,000-row Clay tables ($15–75).
- **Implemented:** only lookups that matched something (a license, at least one permit, or both) are charged. Charging every request, including empty or no-match ones, would churn Clay users enriching messy lists. A response with `matched: false` is free and carries `charged: false`; the README says so.
- **Leave the start fee at the default** for #3. In standby mode one container serves many requests, and per-start fees make no sense to API users.

**Standby compute risk for #3.** Under PPE the **developer pays platform usage**. A standby container that stays warm consumes compute units whether or not anyone calls it. Keep memory low (256–512 MB) and the idle timeout short. After week 1, compare *compute cost per day* against *lookups per day × $0.012 net*. As a rough estimate (verify the CU price for your plan), 512 MB running 24/7 is about 12 CU/day. That would need several hundred lookups per day to break even, which a short idle timeout avoids.

### Tiered (per customer plan) pricing

Several competitors give BRONZE+ customers 10–30% off. **Recommendation: launch with flat prices** (simpler copy, simpler revenue math). Revisit after 30 days of data. Any later change is subject to Apify's price-change rules (see §7), so if you want tiers, decide **before** first publish.

---

## 5. Worked cost examples (what the customer pays)

Austin rows use the **measured** Austin phone fill from `COVERAGE.md` (87% on 2026-11-07); the other phone fill rates are illustrative.

| Scenario | Calculation | Customer pays |
|---|---|---|
| **1,000 Austin permits, 87% phone fill (measured)** (`requirePhone: false`) | 1,000 × $0.003 + 870 × $0.012 + $0.005 start | **≈ $13.45** |
| Same pull with `requirePhone: true` (only the 870 rows that have a phone) | 870 × ($0.003 + $0.012) + $0.005 | **≈ $13.06** |
| n8n template, daily: about 40 new solar/battery permits per day across 3 sources, 70% phone | per day 40 × $0.003 + 28 × $0.012 + $0.005 = $0.461; × 30 days | **≈ $13.83 / month** |
| 1,000 permits, no phones at all (e.g. a source without contractor phone) | 1,000 × $0.003 + $0.005 | **≈ $3.01** |
| Actor #2, weekly WA run returning 300 alert rows | 300 × ($0.004 + $0.018) + $0.005 = $6.605; × 4.33 weeks | **≈ $6.61 / run, ≈ $28.60 / month** |
| Actor #2, full WA roster snapshot, 1,000 contractors (`includeNonExpiring: true`), of which 150 have alerts | 1,000 × $0.004 + 150 × $0.018 + $0.005 | **≈ $6.71** |
| Actor #3, Clay table of 2,000 contractors, 80% match | 1,600 × $0.015 (no-match rows free, per the recommendation) | **≈ $24.00** |
| Actor #3, 100 MCP questions from Claude/Cursor | 100 × $0.015 | **≈ $1.50** |

README-ready line for Actor #1: *"1,000 Austin permits ≈ $13.45 including the contractor-phone add-on at Austin's measured 87% phone fill, or ≈ $3 for a source without phones."*

---

## 6. Revenue math (what we keep)

Apify keeps **20%** of PPE revenue (`apifyMarginPercentage: 0.2`). Under PPE the developer also pays the **platform usage** (compute, storage, proxy) of users' runs, unless "platform usage paid by user" is enabled, which we don't recommend because it makes the price unpredictable.

```
net = 0.80 × (sum of event charges) − platform usage cost of those runs
```

Platform usage per run (estimate only; read the real figure from run stats after the first week): a Socrata-only Python run at 1 GB for 1–3 minutes is roughly 0.02–0.05 CU, about $0.01–0.02 at typical CU prices. There's no browser and no proxy, so this stays small relative to event revenue.

| Scenario | Gross charges / month | After 20% commission | Est. platform cost | **Net / month** |
|---|---|---|---|---|
| 1 user running the n8n template daily (row above) | $13.83 | $11.06 | 30 runs × ~$0.015 = $0.45 | **≈ $10.6** |
| 10 such users | $138.30 | $110.64 | ≈ $4.50 | **≈ $106** |
| 5 insurance agents running #2 weekly (300 alerts each) | 5 × $28.60 = $143.00 | $114.40 | 5 × 4.33 × $0.015 ≈ $0.33 | **≈ $114** |
| 3 Clay users enriching 2,000 matched rows/month on #3 | 3 × 2,000 × $0.015 = $90.00 | $72.00 | standby compute (see risk note in §4), budget $5–20 | **≈ $52–67** |
| **Combined example** | **$371** | **$297** | ≈ $10–25 | **≈ $272–287** |

These are scenarios, not forecasts. The Store data in §2 shows the whole niche had about 400 summed 30-day users across about 100 Actors, and most individual Actors have fewer than 10 per month.

**Break-even per run:** a run must charge more than about $0.02 to cover its own compute after commission. That is roughly 7 permit records, which is why the $0.005 start fee plus `onlyNew` empty-run economics matter for daily schedules.

---

## 7. Console click path: set PPE before the first publish

> UI labels reflect the Apify Console at the time of writing and may differ slightly. The order matters more than the exact labels.

1. **Monetization prerequisites (once per account):** Apify Console → **Settings** → **Monetization / Payouts**. Accept the developer/monetization terms, fill in payout and tax details (W-8BEN or W-9), and complete any identity verification Apify requests.
2. Push the Actor (`apify push`) and do **not** publish yet.
3. Console → **Actors** → **My Actors** (Development) → select the Actor → **Publication** tab.
4. Find the **Monetization** section → **Set up monetization** (or **Edit**) → choose **Pay per event**.
5. For each row in §4, click **Add event** and enter:
   - **Event name**: exactly the machine name the code passes to `Actor.charge()` (`permit-record`, `contact-enriched-record`, `contractor-record`, `expiring-alert-record`, `lookup`). A typo here means the charge silently fails or errors.
   - **Title** and **Description**: copy from the tables in §4.
   - **Price (USD)**: per single event, e.g. `0.003`, not `3`.
   - **Primary event**: on for `permit-record` / `contractor-record` / `lookup`.
   - **One-time event**: leave off for all custom events. They fire many times per run.
6. Set the synthetic **`apify-actor-start`** price (`0.005` for #1/#2; leave the default for #3).
7. Make sure **no price** is set on `apify-default-dataset-item`, since we charge custom events instead.
8. Leave "platform usage paid by user" **off**.
9. **Save**. Re-open the pricing preview on the Actor's Store-page preview and check that the headline reads "$3.00 / 1,000 permit records" (or the equivalent).
10. Do 2–3 test runs from the Console **as a different (test) account**, or check **Run → Charged events** on your own runs. Confirm the event counts match the dataset (e.g. 1,000 `permit-record` + N `contact-enriched-record` where N = rows with a phone).
11. Only now: **Publication → Display information** (title, SEO description, categories) → **Publish to Store**.

**Why "before first publish" matters:** Apify limits how often an Actor's pricing can change, and price changes, especially increases, are **announced to existing users in advance** before they take effect. The Store API exposes this through `notifiedAboutFutureChangeAt`, `notifiedAboutChangeAt` and a public `reasonForChange` in the pricing history. Check the current rules at <https://docs.apify.com/platform/actors/publishing/monetize> before publishing. Launching too cheap and raising prices later is slow and visible. Launching too high costs the first-week users who drive Store ranking. The prices above are deliberately set at or under the competitor medians for the base event, with the premium on the add-ons nobody else sells.

---

## 8. Re-pulling this benchmark

```bash
for t in "building%20permits" "permit" "contractor%20license" "contractor" "license%20lookup"; do
  curl -s "https://api.apify.com/v2/store?search=$t&limit=50" > "store_${t//%20/_}.json"
done
# prices:  .data.items[].currentPricingInfo.pricingPerEvent.actorChargeEvents.<event>.eventPriceUsd
#      or  ...<event>.eventTieredPricingUsd.FREE.tieredEventPriceUsd
# usage:   .data.items[].stats.{totalUsers,totalUsers30Days,totalRuns}
# history: curl -s "https://api.apify.com/v2/acts/<username>~<name>" | jq '.data.pricingInfos'
```

Re-run monthly. Watch for (a) any competitor launching insurance or bond expiry alerts, (b) the $2/1k permit leader adding contractor phones, and (c) `scrapebench` dropping below $4/1k.
