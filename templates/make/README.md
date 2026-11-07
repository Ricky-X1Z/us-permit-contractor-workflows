# Make.com template: contractors whose insurance or bond expires within 30 days -> HubSpot / Pipedrive tasks

Blueprint: [`insurance-expiry-to-crm.blueprint.json`](./insurance-expiry-to-crm.blueprint.json)

Actor: **Contractor License, Insurance & Bond Expiration Radar** (`RickyZ1X~contractor-license-insurance-bond-radar`)

Coverage: **Washington State (WA L&I) now, more states added monthly.** This is not all-US coverage.

## What the scenario does

```
[1] HTTP: POST Apify run-sync-get-dataset-items  (state=WA, expiringWithinDays=30, alertTypes=[insurance, bond])
      |
[2] Iterator (builtin:BasicFeeder) over the returned contractor records
      |
[3] Router
      |-- filter "is_expiring_alert = true" --> [4] HubSpot: create/update company --> [5] HubSpot: create task (due = first alert date)
      |-- filter "is_expiring_alert = true" --> [6] Pipedrive: create organization --> [7] Pipedrive: create activity (due = first alert date)
```

Insurance agents, surety brokers, and suppliers who want to be first in line before a contractor's policy or bond lapses get a weekly call list in their CRM. Each task body contains the carrier, policy number, expiration date, days remaining, bond surety and cancel date, and the contractor's business phone.

The HTTP module calls the Apify REST API directly rather than using the Apify app, so the blueprint imports into any Make account without an extra app connection.

## Import

1. In Make, open **Scenarios -> Create a new scenario**.
2. Click the **... (More)** menu at the bottom of the editor, choose **Import Blueprint**, and select `insurance-expiry-to-crm.blueprint.json`.
3. **Module 1 (HTTP):**
   - In the URL, replace `RickyZ1X` with the Actor owner's username from the Store page.
   - In the `Authorization` header, replace `YOUR_APIFY_API_TOKEN` with your token. You'll find it in Apify Console -> **Settings -> API & Integrations**. For anything shared, store the token in a Make **custom variable** or **keychain** rather than pasting it in plain text.
   - Change the request body if you need to. The supported fields are `state`, `expiringWithinDays`, `alertTypes` (`insurance`, `bond`, `license`), `includeNonExpiring`, `licenseNumbers`, `businessNameContains`, `city`, `specialtyContains`, and `maxRecords`.
4. Right-click module 1 and choose **Run this module only** once. Make needs this to learn the output structure. After that, the `2.contractor.*`, `2.insurance.*`, `2.bond.*`, and `2.alerts[]` mappings resolve in the editor.
5. **Pick one CRM.** Delete the router route you don't use, then select your connection in each remaining CRM module. The blueprint ships with an empty connection (`__IMTCONN__: null`).
   - **HubSpot:** consider creating a custom company property `wa_license_number` and mapping `{{2.contractor.license_number}}` into it. You can then dedupe on it instead of on company name.
   - **Pipedrive:** set the activity `type` to one that exists in your account (`call` is the default).
6. If Make reports a module as unknown, the app version has probably been renamed. Re-add the equivalent module, for example "HubSpot CRM -> Create/Update a Company" or "Pipedrive -> Create an Activity", then copy the mappings from the blueprint JSON.
7. **Schedule:** click the clock on the trigger and set **Every week, Monday, 07:00**. The blueprint carries a `scheduling` block, but depending on how you import, Make may ignore it.
8. Click **Run once**, check the created tasks, then switch the scenario **ON**.

## Output fields used

| Make mapping | Meaning |
|---|---|
| `2.contractor.company_name`, `2.contractor.license_number`, `2.contractor.license_status`, `2.contractor.ubi` | WA L&I contractor identity |
| `2.contractor.phone_e164`, `2.contractor.address.line1/city/state/zip` | Business phone (E.164) and business address |
| `2.insurance.company`, `2.insurance.policy_no`, `2.insurance.expiration_date`, `2.insurance.days_to_expiration`, `2.insurance.agency_name` | Liability insurance on file |
| `2.bond.surety`, `2.bond.amount_usd`, `2.bond.cancel_date`, `2.bond.days_to_cancel` | Contractor bond on file |
| `2.alerts[]` (`type`, `date`, `days_until`) | `insurance_expiring`, `bond_cancelling`, `license_expiring`. Make arrays are 1-indexed, so `2.alerts[1]` is the first alert |
| `2.is_expiring_alert` | `true` if at least one alert falls inside the window; the route filters use it |

## Cost

Each run is billed to **your** Apify account under Pay-Per-Event pricing:

- `contractor-record` for each contractor row returned, and
- `expiring-alert-record` for each row that carries an alert.

With `includeNonExpiring: false` (the default here), every row returned is an alert row. `maxRecords` caps what a run can cost. See the Actor's Store page for current prices. The Apify free plan's monthly credit is enough to try the scenario a few times at no cost.

Make operations: each contractor uses roughly 1 iterator bundle plus 2 CRM operations. A weekly run with 200 alerts uses about 600 Make operations.

## Troubleshooting

- **HTTP 408 or timeout:** `run-sync-get-dataset-items` waits at most about 300 seconds. Lower `maxRecords`, narrow the scope with `city` or `specialtyContains`, or switch to the async pattern: `POST /v2/acts/.../runs`, then *Sleep*, then `GET /v2/datasets/{defaultDatasetId}/items`.
- **401:** the token is wrong or missing the `Bearer ` prefix.
- **Empty iterator:** no contractors in WA have an insurance or bond event inside the window. Try `expiringWithinDays: 60`.

## Compliance

- **B2B use only.** The data comes from Washington State Department of Labor & Industries public contractor licensing records (business licensing, insurance, and bond filings). It contains **no homeowner or consumer data**.
- Phone numbers are the contractor's business numbers as filed with the state.
- **You, the user, are responsible** for complying with the TCPA, the National Do-Not-Call Registry, state telemarketing and "mini-TCPA" laws, CAN-SPAM, and any insurance-solicitation rules that apply to you before calling, texting, or emailing anyone.
- Treat expiration dates as a prompt to verify, not as proof that coverage has lapsed. Always confirm with the state record or the carrier.
