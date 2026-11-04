<p align="center"><a href="https://apify.com/DevilScrapes/opencorporates-alternative-scraper"><img src="assets/banner.svg" alt="OpenCorporates Alternative — US Business Registry Scraper — Python example" width="100%"></a></p>

# OpenCorporates Alternative API in Python: US state registry lookups into pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/opencorporates-alternative-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/opencorporates-alternative-api-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/opencorporates-alternative-api-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository is a compact Python client for the **[OpenCorporates Alternative Scraper](https://apify.com/DevilScrapes/opencorporates-alternative-scraper)** on Apify. You hand it a list of company names, and it hands back what the New York, Colorado, Connecticut and Oregon registries have on file for each one: the legal name the state recorded, the filing type, the standing, when the entity was formed, and who its registered agent is. The lookups run on Apify's servers; your side is a short script and a JSON file. It suits analysts who keep vendor or prospect lists in CSV and want a check they can rerun next quarter.

> **Actor:** [OpenCorporates Alternative — US Business Registry Scraper](https://apify.com/DevilScrapes/opencorporates-alternative-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [opencorporates-alternative-api-javascript](https://github.com/devilscrapes/opencorporates-alternative-api-javascript)

## 🔥 What you get

- **Records from the states themselves**: names are matched against the free open-data datasets that New York, Colorado, Connecticut and Oregon publish, not against a third-party aggregator or a paid search page.
- **One schema for four registries**: each state names its columns differently; every row comes back with the same keys, so a single parser covers all of them.
- **Comparable status bucket**: alongside the state's own wording, `status_normalized` puts each entity into `active`, `inactive` or `unknown`, which you can also filter on before rows are written.
- **Connecticut officers on request**: with `fetchOfficers` on, matched Connecticut rows carry their principals. Home addresses from that dataset are left out on purpose; only business addresses are kept.
- **A shaky state does not stop the batch**: temporary errors are retried with backoff, and a state dataset that keeps failing is skipped with a warning while the remaining names and states resolve.
- **Traceable rows**: each record links to the raw state record and to the dataset's landing page, so an auditor can check the source in one click.

## 🎯 Use cases

### Screen a vendor list before procurement signs anything

Export supplier names from your ERP, feed them through `input.json` and read the output into a DataFrame. Names with no registry hit, or with only inactive hits, become your follow-up list. The rest get a formation date and an entity type next to them for the file.

### Clean a prospect spreadsheet ahead of a campaign

Join the rows back to your lead sheet on the `query` column. Companies that the state marks as dissolved or otherwise inactive can be dropped before anyone spends an email on them, and the registered agent column helps spot shell-like filings that share one agent.

### Run a quarterly re-check from cron

Point a cron entry or an Airflow task at `python main.py`, write each run into a dated folder with `--out`, and compare standings between quarters. A company that moves from active to inactive is exactly the change a credit or compliance team wants surfaced.

### Support an investigation with citable records

Reporters and researchers can collect formation dates and agent names for a list of entities, then cite `source_record_url` for every claim. The link points at the state's own record, which is what an editor or a lawyer will ask for.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/opencorporates-alternative-api-python.git
cd opencorporates-alternative-api-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [OpenCorporates Alternative — US Business Registry Scraper](https://apify.com/DevilScrapes/opencorporates-alternative-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Strip away the CLI handling and this is what remains. The official `apify-client` package starts a run, blocks until it completes, and iterates over the dataset rows for you.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/opencorporates-alternative-scraper").call(run_input={'searchQueries': ['Blue Sky Consulting LLC'], 'jurisdictions': ['NY', 'CO'], 'fetchOfficers': True, 'maxResultsPerQuery': 5, 'statusFilter': 'any', 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` forwards `input.json` to the Actor as written. The sample looks up one common consulting-firm name in the New York and Colorado registries and keeps at most five matches, the same input the Actor's own release check runs, so a first run returns rows. Swap in your own list; `searchQueries` takes up to 500 names in a single run.

```json
{
  "searchQueries": [
    "Blue Sky Consulting LLC"
  ],
  "jurisdictions": [
    "NY",
    "CO"
  ],
  "fetchOfficers": true,
  "maxResultsPerQuery": 5,
  "statusFilter": "any",
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchQueries` | array | yes | `["Blue Sky Consulting LLC"]` | — |
| `jurisdictions` | array | no | `["NY", "CO", "CT", "OR"]` | — |
| `fetchOfficers` | boolean | no | `true` | — |
| `maxResultsPerQuery` | integer | no | `5` | — |
| `statusFilter` | string | no | `"any"` | `any`, `active`, `inactive` |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

Each row is one entity that one state returned for one of your names, so a single input name can yield several rows across states. Dates arrive as ISO `YYYY-MM-DD` strings, and nullable columns stay `None` when a state does not publish that value rather than being filled with guesses.

| Field | Type | Always present |
|---|---|---|
| `query` | string | yes |
| `entity_id` | string | yes |
| `entity_name` | string | yes |
| `jurisdiction` | string | yes |
| `entity_type` | string or null | no |
| `status` | string or null | no |
| `status_normalized` | string | yes |
| `formation_date` | string or null | no |
| `dissolution_date` | string or null | no |
| `principal_address` | string or null | no |
| `registered_agent_name` | string or null | no |
| `registered_agent_address` | string or null | no |
| `officers` | array | yes |
| `source_record_url` | string | yes |
| `registry_url` | string | yes |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Result row emitted | $0.005 |

Prices as configured on 2026-11-03. The [Actor page](https://apify.com/DevilScrapes/opencorporates-alternative-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

The snippet below, which needs pandas (`pip install pandas`), rolls the rows up to one line per input name: how many entities matched, in which states, whether any of them is active, and the earliest formation date. It also lists the names from `input.json` that found nothing at all.

```python
import json

import pandas as pd

names = json.load(open("input.json", encoding="utf-8"))["searchQueries"]
rows = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))

rows["formed"] = pd.to_datetime(rows["formation_date"], errors="coerce")
rows["is_active"] = rows["status_normalized"].eq("active")

summary = (
    rows.groupby("query")
    .agg(
        matches=("entity_id", "size"),
        states=("jurisdiction", lambda s: ",".join(sorted(set(s)))),
        any_active=("is_active", "any"),
        earliest_formed=("formed", "min"),
    )
    .reset_index()
)
summary.to_csv("verification_summary.csv", index=False)
print(summary.to_string(index=False))

not_found = sorted(set(names) - set(rows["query"]))
print("No registry match:", ", ".join(not_found) or "none")
```

## ❓ FAQ

### Does this use OpenCorporates data or need an OpenCorporates key?

Neither. The Actor reads the four states' own open-data datasets, so the only credential you need is an Apify API token. It is an alternative source for the same kind of answer, not a wrapper around OpenCorporates.

### Which states are covered?

New York, Colorado, Connecticut and Oregon. Narrow the list with `jurisdictions` when you know where a company is registered; fewer states means fewer requests and a quicker run.

### Why is `status` empty for some New York or Oregon rows?

Those two states do not include a standing column in their open datasets. The Actor keeps `status` as `None` instead of inventing one, and its `status_normalized` value for those states is a default, not a verified standing. Treat an `active` filter on NY and OR rows as a weak signal and confirm important cases with the state.

### Why is the `officers` list empty?

Only Connecticut publishes a principals dataset that the Actor reads, so rows from the other three states always carry an empty list. For Connecticut, keep `fetchOfficers` set to `true`.

### How fresh is the data?

As fresh as each state's published snapshot. New York describes its dataset as current to the last business day of the month, so recent filings can lag there. None of the four sources is a live transactional lookup.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/opencorporates-alternative-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [OpenCorporates Alternative — US Business Registry Scraper on Apify Store](https://apify.com/DevilScrapes/opencorporates-alternative-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/opencorporates-alternative-api-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [New Business Filings — US State Registry Leads Feed](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) — a dated feed of newly formed companies from the same four state registries.
- [UCC Lien Filing Scraper — CT + CO State Portals](https://apify.com/DevilScrapes/ucc-lien-filing-scraper) — lien filings against debtors in Connecticut and Colorado.
- [GLEIF LEI Scraper — Legal Entity Identifier Records](https://apify.com/DevilScrapes/gleif-lei-records-scraper) — Legal Entity Identifier records when the counterparty sits outside these states.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/opencorporates-alternative-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with OpenCorporates or any US state Secretary of State office.
