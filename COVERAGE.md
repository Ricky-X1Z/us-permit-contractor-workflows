# COVERAGE — measured fill rates

Generated 2026-11-07T03:38:47+00:00 by `scripts/coverage_report.py` (live Socrata queries, 31 requests).

## Permit sources

Measured live on 2026-11-07 over permits issued 2026-10-08 … 2026-11-07 (volume = exact count; fill rates = the 300 most recent permits per source run through the full pipeline).

| Source | Permits (30 d) | Contractor name | Phone | License # | ≥1 job category | Phone comes from |
|---|---:|---:|---:|---:|---:|---|
| Austin, TX | 4,975 | 87% | 87% | 0% | 26% | permit feed |
| San Diego County (unincorporated), CA | 913 | 80% | 80% | 80% | 57% | permit feed + CSLB sidecar |
| New York City, NY | 15,026 | 100% | 92% | 100% | 5% | BIS feed + license join |
| Seattle, WA | 526 | 4% | 0% | 0% | 7% | WA L&I name match |
| Los Angeles, CA | 4,756 | 0% | 0% | 0% | 33% | — (CSLB join in V1.1) |
| Chicago, IL | 2,683 | 97% | 0% | 0% | 12% | — (name only) |

Volume-weighted contractor-phone fill across all sources: **65%**. Spec extractor precision on the golden set: **100.0%** (208 labeled real descriptions).

WA L&I (Actor #2): 76,335 active contractors (100% with phone); next 30 days: 3,790 insurance policies expire, 157 bonds cancel, 1,175 active licenses expire.

## Job categories in the sample

| Source | solar | battery_storage | roofing | hvac | ev_charger | adu | pool |
|---|---:|---:|---:|---:|---:|---:|---:|
| `austin_tx` | 2 | 37 | 2 | 28 | 5 | 0 | 5 |
| `san_diego_ca` | 47 | 45 | 31 | 51 | 15 | 12 | 16 |
| `nyc_dob` | 6 | 0 | 2 | 5 | 0 | 2 | 0 |
| `seattle_wa` | 0 | 0 | 7 | 0 | 0 | 11 | 2 |
| `los_angeles_ca` | 2 | 0 | 52 | 1 | 1 | 33 | 9 |
| `chicago_il` | 4 | 4 | 16 | 11 | 1 | 2 | 1 |

## Spec extractor — golden set

208 labeled descriptions (`tests/golden/labels.csv`): precision **100.0%**, recall **98.9%**.

| Category | TP | FP | FN |
|---|---:|---:|---:|
| solar | 39 | 0 | 0 |
| battery_storage | 25 | 0 | 0 |
| roofing | 23 | 0 | 1 |
| hvac | 27 | 0 | 1 |
| ev_charger | 23 | 0 | 0 |
| adu | 20 | 0 | 0 |
| pool | 21 | 0 | 0 |

## Day-2 go/no-go gate (plan Section 3)

| Check | Measured | Target | Result |
|---|---:|---:|---|
| Sources green | 6/6 | ≥ 4/6 | PASS |
| Permit rows, last 30 days | 28,879 | ≥ 10,000 | PASS |
| Phone fill (volume-weighted) | 65% | ≥ 40% | PASS |
| Spec precision | 100.0% | ≥ 95% | PASS |

## Sample rows (phones masked, individuals withheld)

| Issued | City | Job | Specs | Contractor | Phone | License |
|---|---|---|---|---|---|---|
| 2026-11-05 | Austin, TX | roofing | — | Caden Roofing, Sheetmetal and General Contracting | +1 512-364-xxxx | — |
| 2026-11-05 | Austin, TX | hvac | 5.0 ton | Norrell HVAC, LLC | +1 979-480-xxxx | — |
| 2026-11-05 | San Diego County (unincorporated), CA | ev_charger | — | SMITH ELECTRICAL CONTRACTORS INC | +1 619-758-xxxx | 871xxx |
| 2026-11-05 | San Diego County (unincorporated), CA | hvac | — | AIR PRO HOME SERVICES INC | +1 619-273-xxxx | 109xxxx |
| 2026-11-05 | New York City, NY | solar | 41.34 kW | NYSS LLC | +1 516-418-xxxx | 622xxx |
| 2026-11-05 | New York City, NY | hvac | — | EDITA MC CORP | +1 646-719-xxxx | 613xxx |
| 2026-11-05 | Seattle, WA | adu | — | — | — | — |
| 2026-11-04 | Seattle, WA | adu | — | — | — | — |
| 2026-10-31 | Los Angeles, CA | roofing | 22.0 sq | — | — | — |
| 2026-10-31 | Los Angeles, CA | roofing | 30.0 sq | — | — | — |
| 2026-11-05 | Chicago, IL | roofing | 17.0 sq | MATTHEWS ROOFING COMPANY INC | — | — |
| 2026-11-05 | Chicago, IL | roofing | 13.17 sq | APPLE ROOFING LLC | — | — |

Compliance: contractor contact data is business information from official open-data portals. No homeowner data is collected or published. Buyers are responsible for TCPA, DNC and state telemarketing rules.
