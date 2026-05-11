# n8n Task 3: Campaign Metrics -  Insight Summary Stored in PostgreSQL


## Task chosen

**Task 3 – Campaign Metrics - Insight Summary Stored in PostgreSQL**

---

## Why this task

Longship Marketing is a data-driven growth marketing agency focused on performance across US and EU markets. Campaign analytics sits at the core of that work - understanding which campaigns are working, which are anomalies, and being able to communicate that clearly to clients.

Task 2 was tempting as a way to stand out - two AI agents, structured diffs, line-by-line patching. But honestly, pulling that off well in a 2-hour window would rely heavily on AI assistance to generate the patching logic and prompt structure. That felt like demonstrating how well I can prompt an AI, not how well I understand marketing automation.

For a role at a marketing agency, I think what matters more is fluency with the tools and data that drive real campaign decisions, not just the ability to build clever AI pipelines in isolation. Task 3 maps directly onto that reality: ingest raw campaign data from the sources a team actually uses (Airtable, CSV exports), calculate meaningful metrics, generate a narrative summary a marketing director can act on, and store everything for historical reference. It felt like the most honest demonstration of day-to-day automation value for an agency like Longship.

---

## Approximate time spent

~2 hours (including setup of Supabase, Airtable, and OpenRouter)

---

## Workflow design

### Main steps and data flow

```
Click Trigger (with optional filters)
        ↓
Generate Run ID
        ↓
Read CSV (GitHub) ──────┐
                        ├──→ Normalize → Merge → Check Dataset → Calculate Metrics → Prepare LLM Input → LLM Summary → Save Insight Summary
Read Airtable ──────────┘                                              ↓
                                                               Save Campaign Metrics
```

1. **Click Trigger** — manual trigger with three optional input fields: `date_from`, `date_to`, `campaign_filter`. If left empty, all campaigns are processed.

2. **Generate Run ID** — generates a UUID for the current run. Used to link `campaign_metrics` and `insight_summaries` rows, and enables idempotency checks.

3. **Read CSV / Read Airtable** — two parallel data sources. CSV is loaded via HTTP GET from a GitHub raw URL. Airtable is read via the native n8n node using a Personal Access Token.

4. **Normalize CSV / Normalize Airtable** — separate Code nodes map each source's column names into a unified schema (`campaign_name`, `date`, `impressions`, `clicks`, `conversions`, `spend`, `source`). This makes the downstream pipeline source-agnostic and easy to extend.

5. **Merge** — combines both streams into a single array of campaigns using Append mode.

6. **Check Dataset** — validates the merged data. Throws a descriptive error if the dataset is empty. Also applies optional filters passed from the trigger (date range, campaign name substring).

7. **Calculate Metrics** — computes per-campaign CTR, CPC, and conversion rate. Calculates averages across all campaigns, identifies the top performer by conversion rate, and flags anomalies (campaigns whose CTR is >50% above or below the average).

8. **Prepare LLM Input** — aggregates all 10 campaign rows into a single structured prompt string. Formats the data as a readable table with anomaly flags, and includes key stats (total spend, average CTR/CVR, top performer).

9. **LLM Summary** — sends the prompt to `google/gemini-2.0-flash-001` via OpenRouter using n8n's Basic LLM Chain node. Returns a 3-paragraph executive summary covering overall performance, top performer analysis, and anomaly recommendations.

10. **Save Campaign Metrics** — writes one row per campaign to the `campaign_metrics` PostgreSQL table, including all computed metrics, anomaly flags, and the `run_id`.

11. **Save Insight Summary** — writes the LLM-generated summary and aggregate stats to the `insight_summaries` table, linked by the same `run_id`.

---

### Key decisions and trade-offs

**Dual data source with normalization layer**
Rather than requiring a specific column naming convention, each source has its own normalization node that maps known variants (e.g. `Impressions`, `impr`, `impressions`) to a standard schema. This makes the workflow resilient to real-world inconsistencies. In production, this could be extended to a config-driven mapping or an LLM-assisted auto-mapper for unknown schemas.

**Aggregation before LLM call**
All campaign data is collapsed into a single prompt instead of making one LLM call per campaign. This is both cheaper and produces a better summary — the model can compare campaigns and reason across the full dataset in one pass.

**run_id for idempotency**
Each workflow execution generates a UUID that is attached to every row written to PostgreSQL. This means runs can be traced, compared over time, and duplicate runs can be detected by checking for matching `run_id` values. It also makes it easy to build a historical dashboard on top of the data.

**Anomaly definition**
Anomaly detection uses a simple rule: CTR >50% above or below the average across all campaigns. This is a pragmatic starting point — easy to understand, easy to explain to a client. In production, a more sophisticated approach (e.g. z-scores, rolling averages, or per-channel benchmarks) would be more meaningful.

**CSV source as GitHub raw URL**
The CSV is hosted as a raw file on GitHub. This is a reasonable proxy for a real file source in a demo context. In production this would be replaced with an S3 bucket, Google Drive, or SFTP depending on the client's infrastructure.

---

### Error handling and idempotency

- **Empty dataset**: the `Check Dataset` node throws a descriptive error if no campaigns are returned after merging and filtering. The workflow stops cleanly rather than proceeding with broken calculations.
- **Filter mismatch**: if filters are provided but no campaigns match, a specific error message is thrown indicating which filters were applied.
- **run_id**: every execution produces a unique ID that links all written rows, enabling deduplication and audit trails.
- **LLM failures**: not explicitly handled in this prototype. In production, a retry node or error branch would catch API failures and log them before stopping.

---

## What I would improve or extend with more time

- **Idempotency check**: before writing to PostgreSQL, query for existing rows with the same `run_id` and skip or update rather than inserting duplicates.
- **Schedule trigger**: replace or supplement the manual trigger with a Schedule node (e.g. every Monday morning) so the workflow runs automatically.
- **Per-channel anomaly benchmarks**: instead of comparing all campaigns against a single average, group by channel type (paid social, email, retargeting) and apply channel-specific thresholds.
- **Email/Slack alert on anomalies**: if anomalies are detected, send the summary to a Slack channel or email automatically, not just store it.
- **LLM error handling**: add a retry branch on the LLM node and log failures to a separate `workflow_errors` table.
- **Dashboard**: the PostgreSQL schema is designed to support a simple Metabase or Grafana dashboard showing trends across runs over time.

---

## Setup notes

### Credentials required

| Credential | Where to get it |
|---|---|
| `AIRTABLE_API_TOKEN` | airtable.com → Developer Hub → Personal Access Tokens |
| `OPENROUTER_API_KEY` | openrouter.ai → Keys |
| PostgreSQL connection | Any PostgreSQL instance (tested on Supabase free tier) |

### Airtable schema assumed

Table: `Campaigns`

| Field | Type |
|---|---|
| `campaign_name` (Name) | Text |
| `date` | Date |
| `Impressions` | Number |
| `clicks` | Number |
| `conversions` | Number |
| `spend` | Number |

### PostgreSQL schema

```sql
CREATE TABLE campaign_metrics (
  id SERIAL PRIMARY KEY,
  run_id UUID NOT NULL,
  campaign_name TEXT,
  report_date DATE,
  impressions INTEGER,
  clicks INTEGER,
  conversions INTEGER,
  spend NUMERIC(10,2),
  ctr NUMERIC(6,4),
  cpc NUMERIC(8,2),
  conversion_rate NUMERIC(6,4),
  is_anomaly BOOLEAN DEFAULT FALSE,
  anomaly_reason TEXT,
  source TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE insight_summaries (
  id SERIAL PRIMARY KEY,
  run_id UUID NOT NULL,
  summary_text TEXT,
  top_performer TEXT,
  avg_ctr NUMERIC(6,4),
  avg_conversion_rate NUMERIC(6,4),
  total_spend NUMERIC(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);
```

### What is real vs mocked

| Component | Status |
|---|---|
| Airtable read | ✅ Real integration |
| PostgreSQL write | ✅ Real integration (Supabase) |
| LLM summary | ✅ Real API call (OpenRouter / Gemini 2.0 Flash) |
| CSV source | ⚠️ Mocked — static file on GitHub. In production: S3, Google Drive, or SFTP |
