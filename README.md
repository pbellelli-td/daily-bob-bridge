# daily-bob-bridge

Bridges the /daily-bob skill (runs inside Claude/Cowork) to Supabase, using GitHub as the
trusted middle step. Claude commits the day's JSON here; a GitHub Action pushes it into
Supabase. Claude never sees your Supabase key — it lives only as a GitHub secret.

## One-time setup (you do this, ~5 minutes)

### 1. Create the repo
1. Go to https://github.com/new
2. Repository name: `daily-bob-bridge` (or any name — just tell Claude what you picked)
3. Visibility: **Private** (recommended — this will hold account-level business data)
4. Do NOT initialize with a README (we're uploading one)
5. Click **Create repository**

### 2. Add the Supabase secrets
1. In the new repo, click **Settings** (top nav of the repo, not your account settings)
2. Left sidebar → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Add secret #1:
   - Name: `SUPABASE_URL`
   - Value: `https://yhvjxbjpbjfofpbingop.supabase.co`
   - Click **Add secret**
5. Click **New repository secret** again. Add secret #2:
   - Name: `SUPABASE_ANON_KEY`
   - Value: (open Supabase → your project → Settings → API → copy the `anon` `public` key)
   - Click **Add secret**
6. You should now see both `SUPABASE_URL` and `SUPABASE_ANON_KEY` listed (values hidden) under
   Actions secrets.

### 3. Tell Claude the repo name
Once created, tell Claude the repo URL (e.g. `github.com/pbellelli-td/daily-bob-bridge`) and
it will push the workflow file + push daily data there on each /daily-bob run.

## How the daily flow works
1. Cowork runs `/daily-bob` on schedule (weekday 08:30 CEST — set this up separately in Cowork).
2. The skill computes today's flagged accounts + CSAM stats as usual.
3. Instead of pasting SQL into Supabase manually, the skill writes two files to this repo:
   - `data/accounts.json` — array of row objects matching `daily_bob_accounts` columns
   - `data/csam_stats.json` — array of row objects matching `daily_bob_csam_stats` columns
4. The commit triggers `.github/workflows/push-to-supabase.yml` automatically.
5. The Action calls the Supabase REST API with `Prefer: resolution=merge-duplicates`, which
   upserts on the existing primary keys (`snapshot_date, account_id` / `snapshot_date, csam`) —
   matching the same upsert behavior as the SQL in `daily_bob_tables.sql`.
6. Check the **Actions** tab in the repo any morning to confirm the run succeeded (green check)
   or failed (red X, with logs).

## Data shape reference
`data/accounts.json` — one object per flagged account:
```json
[
  {
    "snapshot_date": "2026-07-10",
    "account_id": "54974934672",
    "name": "RBC ACCOUNTANTS LTD",
    "csam": "Piera Bellelli",
    "market": "UK",
    "rag": "critical",
    "arr": 1807.97,
    "health_score": "At-Risk",
    "days_since_last_activity": 12,
    "emails_logged_7d": 0,
    "expansion": null,
    "churn": "Slack #region_uk: threatening to leave",
    "hubspot_url": "https://app.hubspot.com/contacts/46271156/record/0-2/54974934672",
    "notes": "🆕 new today"
  }
]
```

`data/csam_stats.json` — one object per CSAM:
```json
[
  {
    "snapshot_date": "2026-07-10",
    "csam": "Julia Seitz",
    "market": "DACH",
    "critical_count": 3,
    "at_risk_count": 5,
    "watch_count": 8,
    "arr_at_risk": 24500,
    "avg_days_since_contact": 9.4,
    "emails_logged_7d": 22,
    "outreach_touches_7d": 30,
    "untouched_14d": 4,
    "responsiveness": "yellow"
  }
]
```
