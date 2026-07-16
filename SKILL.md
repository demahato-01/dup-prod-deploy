---
name: bq-prod-deploy
description: End-to-end INFRA prod deployment skill for Pipeline Composer SOX tickets. Given a Jira ticket ID (GDP-XXXX), reads the ticket, creates BigQuery tables in prod from stable DDL, generates the backfill SQL, reconciles row counts, and updates Jira. Use for all [INFRA] Create tables & copy history sub-tasks under prod deployment parent tickets.
---

# bq-prod-deploy — BQ Prod Deployment Skill

## Usage
```
/bq-prod-deploy <TICKET_ID>
```
Example: `/bq-prod-deploy GDP-8738`

Parse `$ARGUMENTS` to extract `TICKET_ID` (e.g. `GDP-8738`).

---

## Environment

- Jira token: `$JIRA_TOKEN` (already in env)
- Jira base URL: `https://groupondev.atlassian.net`
- Jira user: `demahato@groupon.com`
- Stable BQ project: `prj-grp-curation-sox-stable`
- Prod BQ project: `prj-grp-curation-sox-prod`
- BQ location flag: `--location=us-central1`
- GCS bucket swap: `grpn-dnd-stable-` → `grpn-dnd-prod-`

---

## Jira Comment — Format & Helper

**Post a structured comment to Jira after every step.** This creates a full audit trail in the ticket even if work stops mid-way.

### ADF comment poster (reuse this pattern for every comment)

```bash
post_jira_comment() {
  local TICKET_ID="$1"
  local JSON_BODY="$2"
  curl -s -u "demahato@groupon.com:$JIRA_TOKEN" \
    -H "Content-Type: application/json" \
    -X POST \
    "https://groupondev.atlassian.net/rest/api/3/issue/${TICKET_ID}/comment" \
    -d "{\"body\": ${JSON_BODY}}" \
    | python3 -c "import json,sys; r=json.load(sys.stdin); print('✅ Comment posted:', r.get('id','unknown'))"
}
```

### ADF building blocks

Use these JSON fragments to assemble each comment body. Keep comments clean — heading, then bullets or a short paragraph. No walls of text.

**Heading (level 3):**
```json
{"type":"heading","attrs":{"level":3},"content":[{"type":"text","text":"YOUR HEADING"}]}
```

**Paragraph with bold label + value:**
```json
{"type":"paragraph","content":[
  {"type":"text","text":"Label: ","marks":[{"type":"strong"}]},
  {"type":"text","text":"value here"}
]}
```

**Bullet list:**
```json
{"type":"bulletList","content":[
  {"type":"listItem","content":[{"type":"paragraph","content":[{"type":"text","text":"item"}]}]},
  {"type":"listItem","content":[{"type":"paragraph","content":[{"type":"text","text":"item"}]}]}
]}
```

**Horizontal rule:**
```json
{"type":"rule"}
```

**Code block:**
```json
{"type":"codeBlock","attrs":{"language":"sql"},"content":[{"type":"text","text":"SELECT ..."}]}
```

**Status emoji convention used throughout:**
- ✅ success / complete
- ⚠️ warning / needs attention
- ❌ error / blocker
- 🔄 in progress
- ⏭️ skipped

---

## Step 0 — Fetch & Parse the Ticket

```bash
curl -s -u "demahato@groupon.com:$JIRA_TOKEN" \
  -H "Accept: application/json" \
  "https://groupondev.atlassian.net/rest/api/3/issue/<TICKET_ID>?fields=summary,description,status,parent,assignee,reporter,comment,timetracking" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
f = data['fields']
print('STATUS:', f['status']['name'])
print('PARENT:', f.get('parent', {}).get('key',''), '-', f.get('parent', {}).get('fields',{}).get('summary',''))
print('SUMMARY:', f['summary'])
print('REPORTER:', f.get('reporter', {}).get('displayName',''), '|', f.get('reporter', {}).get('emailAddress',''))
tt = f.get('timetracking') or {}
print('ESTIMATE:', tt.get('originalEstimate', 'NONE'))

def walk(node):
    t = node.get('type','')
    if t == 'text': return [node.get('text','')]
    if t == 'codeBlock':
        code = []
        for c in node.get('content',[]): code.extend(walk(c))
        return ['CODE_BLOCK_START'] + code + ['CODE_BLOCK_END']
    parts = []
    for c in node.get('content',[]): parts.extend(walk(c))
    return parts

desc = f.get('description') or {}
print('DESC:', ' '.join(walk(desc)))
"
```

**If ESTIMATE is NONE**, automatically set it to 2h before proceeding:

```bash
curl -s -u "demahato@groupon.com:$JIRA_TOKEN" \
  -H "Content-Type: application/json" \
  -X PUT \
  "https://groupondev.atlassian.net/rest/api/3/issue/<TICKET_ID>" \
  -d '{"fields": {"timetracking": {"originalEstimate": "2h"}}}'
# Confirm with: ✅ Estimate set to 2h on <TICKET_ID>
```

From the output, extract and display to the user in a clean summary:

```
Pipeline : dly_paypal_stl_region_varrundate
Ticket   : GDP-8738  (Status: To Do)
Parent   : GDP-8735

OBJECTS TO CREATE:
  # | Dataset.Table                          | Role     | Backfill?
  1 | staging.paypal_stl_raw                 | external | No
  2 | staging.paypal_stl                     | staging  | No
  3 | prod_groupondw.paypal_stl              | final    | Yes — day_rw, region=US
  4 | user_groupondw.paypal_stl              | view     | No

BACKFILL:
  Target   : prod_groupondw.paypal_stl
  Source   : Teradata prod_groupondw.paypal_STL  (fallback: stable)
  Filter   : day_rw (DATE) + region = 'US'
  Range    : <confirm with Payments Engineering>

UPSTREAM DEPS: None (self-contained)

WATCH-OUTS:
  ⚠️  PK grain discrepancy — confirm 3-col vs 4-col key before backfill
```

**Stop if status is already Done.** Otherwise ask: **"Does this look right? Proceed?"**

### On user confirmation — before posting Jira comment

1. **Transition ticket to In Progress:**

```bash
# Get transitions
curl -s -u "demahato@groupon.com:$JIRA_TOKEN" \
  "https://groupondev.atlassian.net/rest/api/3/issue/<TICKET_ID>/transitions" \
  | python3 -c "import json,sys; [print(t['id'],'—',t['name']) for t in json.load(sys.stdin)['transitions']]"

# Apply "Start progress" (id varies — pick the one named "Start progress" or "In Progress")
curl -s -u "demahato@groupon.com:$JIRA_TOKEN" \
  -H "Content-Type: application/json" -X POST \
  "https://groupondev.atlassian.net/rest/api/3/issue/<TICKET_ID>/transitions" \
  -d '{"transition":{"id":"<IN_PROGRESS_ID>"}}'
```

### Jira comment — Step 0

Post immediately after transitioning to In Progress, to mark work as started:

```
Heading:  🔄 INFRA Deployment Started — <PIPELINE_NAME>

Bullets:
  • Ticket: <TICKET_ID>  |  Parent: <PARENT_KEY>
  • Operator: demahato@groupon.com
  • Objects to create: <N> (external: X, staging: Y, final: Z, views: W)
  • Backfill target: <dataset>.<final_table>  |  Filter: <filter_col>
  • Backfill source: Teradata prod_groupondw.<TD_TABLE>  (fallback: stable)
  • Watch-outs: <any flagged issues, or "None">
```

---

## Step 1 — Extract DDL from Stable

**If inline DDL is already in the ticket:** display it and skip the bq query below.

**If only an INFORMATION_SCHEMA query is provided:** run it:

```bash
bq query --use_legacy_sql=false --location=us-central1 \
  --project_id=prj-grp-curation-sox-stable \
  --format=json \
  '<INFORMATION_SCHEMA_QUERY_FROM_TICKET>'
```

For views not returned by the tables query, fetch separately:

```bash
bq query --use_legacy_sql=false --location=us-central1 \
  --project_id=prj-grp-curation-sox-stable \
  --format=json \
  "SELECT ddl FROM \`prj-grp-curation-sox-stable.<dataset>\`.INFORMATION_SCHEMA.VIEWS
   WHERE table_name = '<view_name>'"
```

Display each DDL statement to the user. Ask: **"DDL looks correct? Proceed to create tables in prod?"**

### Jira comment — Step 1

```
Heading:  ✅ Step 1 — DDL Extracted from Stable

Bullets:
  • Source project: prj-grp-curation-sox-stable
  • Objects extracted:
      - staging.<table> (external) — <N> chars DDL
      - staging.<table> — <N> chars DDL
      - prod_groupondw.<table> — PARTITION BY <col>, CLUSTER BY (<cols>)
      - user_groupondw.<view> — CREATE OR REPLACE VIEW
  • DDL source: INFORMATION_SCHEMA  |  inline from ticket
```

---

## Step 2 — Create Prod Datasets (if missing)

**NEVER drop or delete an existing dataset.** Only create datasets that are missing. If a dataset already exists, leave it exactly as-is and move on.

Check each required dataset first:

```bash
bq show prj-grp-curation-sox-prod:<dataset> 2>&1
```

If it does NOT exist, create it:

```bash
bq --location=us-central1 --project_id=prj-grp-curation-sox-prod \
  mk --dataset --description "bq-prod-deploy <TICKET_ID>" \
  prj-grp-curation-sox-prod:<dataset>
```

Check for: `staging`, `prod_groupondw`, `user_groupondw` (only if a view exists).
Print ✅ created or ✅ already exists for each. Never print anything about deleting.

### Jira comment — Step 2

```
Heading:  ✅ Step 2 — Prod Datasets Verified

Bullets:
  • staging: ✅ created | ✅ already existed (no changes made)
  • prod_groupondw: ✅ created | ✅ already existed (no changes made)
  • user_groupondw: ✅ created | ✅ already existed (no changes made) | ⏭️ not needed
```

---

## Step 3 — Apply DDL to Prod

Apply two substitutions to every DDL statement:
1. `prj-grp-curation-sox-stable` → `prj-grp-curation-sox-prod`
2. GCS buckets: `grpn-dnd-stable-` → `grpn-dnd-prod-` (external tables only)

**Show the final substituted DDL to the user before running anything.**

**Creation order (strict):**
1. External staging tables
2. Typed staging tables
3. Final tables
4. Views (last — they reference the finals)

For each object, follow this sequence **individually** — check, create, post DDL, repeat:

> ❗ **MANDATORY — DO NOT SKIP — DO NOT BATCH**
> The DDL Jira comment is REQUIRED after EVERY object creation, no exceptions.
> You MUST NOT move to the next object until the DDL comment for the current one is posted.
> This is non-negotiable regardless of how many objects are in the ticket.

1. Check if the object already exists (see HARD STOP below).
2. Run the DDL:
   ```bash
   bq query --use_legacy_sql=false --location=us-central1 \
     --project_id=prj-grp-curation-sox-prod \
     '<DDL_STATEMENT>'
   ```
3. **Before moving to the next object**, fetch the DDL from prod and post it to Jira:
   - Tables: `SELECT ddl FROM \`prj-grp-curation-sox-prod.<dataset>\`.INFORMATION_SCHEMA.TABLES WHERE table_name = '<table>'`
   - Views: `SELECT view_definition FROM \`prj-grp-curation-sox-prod.<dataset>\`.INFORMATION_SCHEMA.VIEWS WHERE table_name = '<view>'`
   - External tables: BQ REST API `GET /bigquery/v2/projects/prj-grp-curation-sox-prod/datasets/<dataset>/tables/<table>`

   **Jira comment format per object:**
   ```
   Heading (level 4): ✅ DDL Applied — <dataset>.<table> (<TYPE>)
   Code block (sql): <full DDL fetched from INFORMATION_SCHEMA or REST API>
   Note: any fixes applied (e.g. SELECT * alias fix, REST API instead of SQL DDL)
   ```

4. Only after the Jira comment is confirmed posted, proceed to the next object.

**Checklist before moving on from any object:**
- [ ] Table created in BQ prod ✅
- [ ] DDL fetched from INFORMATION_SCHEMA ✅
- [ ] DDL comment posted to Jira ✅

---

### ⚠️ Table Already Exists — HARD STOP

Before running the DDL for any object, check if it already exists:

```bash
bq show --format=prettyjson prj-grp-curation-sox-prod:<dataset>.<table> 2>&1
```

**If the table exists, STOP. Do NOT drop it. Follow this sequence:**

#### 3a — Capture existing DDL (backup)

Pull the current DDL from prod and post it to Jira immediately as a backup before touching anything:

```bash
bq query --use_legacy_sql=false --location=us-central1 \
  --project_id=prj-grp-curation-sox-prod \
  --format=json \
  "SELECT ddl FROM \`prj-grp-curation-sox-prod.<dataset>\`.INFORMATION_SCHEMA.TABLES
   WHERE table_name = '<table_name>'
   UNION ALL
   SELECT ddl FROM \`prj-grp-curation-sox-prod.<dataset>\`.INFORMATION_SCHEMA.VIEWS
   WHERE table_name = '<table_name>'"
```

Post this Jira comment immediately (before asking the user anything):

```
Heading:  📋 Existing DDL Backup — <dataset>.<table>

Paragraph:
  Existing table found in prj-grp-curation-sox-prod. DDL captured below as backup
  before any changes are applied.

  Type         : TABLE | EXTERNAL_TABLE | VIEW
  Row count    : <numRows>
  Partition    : <timePartitioning or "none">
  Clustering   : <clustering fields or "none">
  Last modified: <lastModifiedTime>

Code block (language: sql):
  <FULL EXISTING DDL FROM PROD>
```

#### 3b — Show comparison and ask

Display to the user:

```
⚠️  Table already exists: prj-grp-curation-sox-prod.<dataset>.<table>

  Existing spec (prod):
    Type        : TABLE | EXTERNAL_TABLE | VIEW
    Partition   : <timePartitioning or "none">
    Clustering  : <clustering fields or "none">
    Row count   : <numRows>
    Last modified: <lastModifiedTime>

  Stable spec (to apply):
    Partition   : <from stable DDL>
    Clustering  : <from stable DDL>

  Spec match: ✅ matches stable | ⚠️ DIFFERS — <what differs>

  Existing DDL backed up to Jira ✅

What would you like to do?
  1. Replace — apply stable DDL using CREATE OR REPLACE (no data loss for tables with data;
               schema will be updated in place)
  2. Skip    — keep the existing object as-is and continue
  3. Abort   — stop the deployment entirely
```

**Wait for explicit user input. Do not proceed until a choice is made.**

#### 3c — Apply the chosen action

- **Replace:** Use `CREATE OR REPLACE TABLE` / `CREATE OR REPLACE EXTERNAL TABLE` / `CREATE OR REPLACE VIEW` — never `bq rm`. Modify the DDL from Step 1 to use `CREATE OR REPLACE` instead of `CREATE TABLE IF NOT EXISTS` or plain `CREATE TABLE`, then run it.

  ```bash
  bq query --use_legacy_sql=false --location=us-central1 \
    --project_id=prj-grp-curation-sox-prod \
    'CREATE OR REPLACE TABLE `prj-grp-curation-sox-prod.<dataset>.<table>` ...'
  ```

- **Skip:** Mark as ⏭️ skipped, note the reason, continue to the next object.

- **Abort:** Post a Jira comment documenting what was completed before the stop, then halt.

**NEVER use `bq rm` or any DROP command on any object at any point.**

**Apply this check individually for each object — never batch or auto-proceed.**

---

### Jira comment — Step 3

Post after all objects are processed:

```
Heading:  ✅ Step 3 — Objects Applied in Prod  (or ⚠️ if any were skipped)

Table per object:
  | Object                              | Action              | Partition           | Cluster            |
  | staging.paypal_stl_raw              | ✅ Created          | —                   | —                  |
  | staging.paypal_stl                  | ✅ Created          | —                   | —                  |
  | prod_groupondw.paypal_stl           | ✅ Created          | PARTITION BY day_rw | account_id, txn_id |
  | user_groupondw.paypal_stl           | ✅ Created          | —                   | —                  |

Possible action values per row:
  ✅ Created          — did not exist, created fresh
  🔄 Replaced         — existed, DDL backed up to Jira, applied CREATE OR REPLACE from stable
  ⏭️  Skipped          — existed, user chose to keep as-is
  ❌ Failed           — error applying DDL (detail in notes below)

If any object was replaced, add a note:
  • 🔄  prod_groupondw.paypal_stl — replaced via CREATE OR REPLACE (spec mismatch: clustering
       differed from stable; existing DDL backed up in comment above)

If any object was skipped, add a note:
  • ⏭️  staging.paypal_stl — skipped (already existed, spec matched stable, user chose to keep)
```

---

---

## Step 3b — Verify Upstream Tables (if any)

If the ticket lists upstream read-only tables the pipeline reads but doesn't write, verify they exist and are populated in prod:

```bash
bq show prj-grp-curation-sox-prod:<dataset>.<table> 2>&1
bq query --use_legacy_sql=false --location=us-central1 \
  --project_id=prj-grp-curation-sox-prod \
  "SELECT COUNT(*) as row_count FROM \`prj-grp-curation-sox-prod.<dataset>.<table>\`"
```

Warn if any upstream table is missing or empty — the pipeline will silently misbehave (e.g. fact rows routed to rejects instead of finals).

### Jira comment — Step 3b (only if upstream deps exist)

```
Heading:  ✅ Step 3b — Upstream Tables Verified  (or ⚠️ if issues found)

Bullets:
  • prod_groupondw.dim_giftcard_brand_retailer — ✅ exists, 12,345 rows
  • prod_groupondw.dim_giftcard_promos — ✅ exists, 890 rows
  (or)
  • prod_groupondw.dim_giftcard_promos — ⚠️ EXISTS but 0 rows — pipeline will route to rejects at runtime
```

---

## Step 4 — Backfill Date Range

Ask the user:

```
Step 4: Backfill date range needed.

  Target table : prj-grp-curation-sox-prod.<dataset>.<final_table>
  Filter column: <file_date | day_rw | settlement_date_key>
  Source       : Teradata prod_groupondw.<TD_TABLE>
                 (fallback: prj-grp-curation-sox-stable.<dataset>.<final_table>)

  Please provide:
    Start date   (e.g. 2024-01-01) :
    Cutover date (e.g. 2026-07-14) :

  Note: confirm these dates with Payments Engineering before proceeding.
  For settlement_date_key — use INT format YYYYMMDD (e.g. 20240101).
```

**Do not proceed until dates are provided.**

### Jira comment — Step 4

```
Heading:  ✅ Step 4 — Backfill Date Range Confirmed

Bullets:
  • Target table : <dataset>.<final_table>
  • Source       : Teradata prod_groupondw.<TD_TABLE>  |  stable fallback
  • Filter column: <filter_col>
  • Start date   : <start_date>
  • Cutover date : <cutover_date>
  • Confirmed by : demahato@groupon.com
```

---

## Step 5 — Generate & Execute Backfill

Determine the load pattern from the ticket description:

### Pattern A — INSERT with date filter (most common: `file_date`, `day_rw` with date partition)
```sql
INSERT INTO `prj-grp-curation-sox-prod.<dataset>.<final_table>`
SELECT * FROM `prj-grp-curation-sox-stable.<dataset>.<final_table>`
WHERE <filter_col> BETWEEN DATE '<start_date>' AND DATE '<cutover_date>'
  AND <region_filter_if_applicable>
```

### Pattern B — MERGE upsert on natural key (no date partition, e.g. `paypal_winloss`)
```sql
MERGE `prj-grp-curation-sox-prod.<dataset>.<final_table>` T
USING (
  SELECT * FROM `prj-grp-curation-sox-stable.<dataset>.<final_table>`
) S
ON T.<key1> = S.<key1> AND T.<key2> = S.<key2> AND T.<key3> = S.<key3>
WHEN MATCHED THEN UPDATE SET <all non-key cols>
WHEN NOT MATCHED THEN INSERT VALUES (<all cols>)
```

### Pattern C — INT date key (`settlement_date_key` YYYYMMDD)
```sql
INSERT INTO `prj-grp-curation-sox-prod.<dataset>.<final_table>`
SELECT * FROM `prj-grp-curation-sox-stable.<dataset>.<final_table>`
WHERE <filter_col> BETWEEN <start_int> AND <cutover_int>
```

**Show the generated SQL to the user.** Ask: **"Run this backfill? (yes/no)"**

On confirmation:

```bash
bq query --use_legacy_sql=false --location=us-central1 \
  --project_id=prj-grp-curation-sox-prod \
  --nouse_cache \
  '<BACKFILL_SQL>'
```

Capture and display: rows inserted, bytes processed, job ID, elapsed time.

**Staging tables: never backfill.** They are truncated and reloaded on every pipeline run.

### Jira comment — Step 5

```
Heading:  ✅ Step 5 — Backfill Completed

Bullets:
  • Table    : prod_groupondw.<final_table>
  • Pattern  : A (INSERT with date filter) | B (MERGE upsert) | C (INT key)
  • Source   : prj-grp-curation-sox-stable  |  Teradata (via export)
  • Filter   : <filter_col> BETWEEN '<start>' AND '<cutover>'  [+ region = 'US']
  • Rows written  : X,XXX,XXX
  • Bytes processed: X.X GB
  • BQ Job ID: <bqjob_...>
  • Duration : Xm Xs

Code block (the SQL that was run):
  INSERT INTO `prj-grp-curation-sox-prod.<dataset>.<final_table>` ...
```

---

## Step 6 — Reconcile Row Counts

Run counts on both sides for the backfilled date range:

```bash
# Prod
bq query --use_legacy_sql=false --location=us-central1 \
  --project_id=prj-grp-curation-sox-prod \
  "SELECT COUNT(*) as cnt FROM \`prj-grp-curation-sox-prod.<dataset>.<table>\`
   WHERE <filter_col> BETWEEN <start> AND <cutover>"

# Stable (reference)
bq query --use_legacy_sql=false --location=us-central1 \
  --project_id=prj-grp-curation-sox-stable \
  "SELECT COUNT(*) as cnt FROM \`prj-grp-curation-sox-stable.<dataset>.<table>\`
   WHERE <filter_col> BETWEEN <start> AND <cutover>"
```

Display to the user:

```
Reconciliation — prod_groupondw.<table>
  Date range   : <start> → <cutover>
  Prod rows    : X,XXX,XXX
  Stable rows  : Y,YYY,YYY
  Delta        : Z  (0.00%)
  Status       : ✅ MATCH  |  ⚠️ MISMATCH — do not proceed to Done
```

**Flag any delta > 0.01% as a blocker. Do NOT post Done or transition the ticket until resolved.**

### Jira comment — Step 6

```
Heading:  ✅ Step 6 — Reconciliation Complete  (or ⚠️ MISMATCH — Blocked)

Table (one row per final table reconciled):
  | Table                          | Date Range              | Prod Rows  | Stable Rows | Delta | Status     |
  | prod_groupondw.paypal_stl      | 2024-01-01 → 2026-07-14 | 12,345,678 | 12,345,678  | 0     | ✅ MATCH   |

If mismatch:
  ⚠️  Delta exceeds 0.01% threshold.
  Next step: investigate missing rows before proceeding to handoff.
  Blocker: ticket NOT transitioned to Done.
```

---

## Step 7 — Final Summary & Jira Wrap-up

Post a final consolidated summary comment and offer to transition the ticket to Done.

### Final Jira comment — full summary

```
Heading:  ✅ INFRA Complete — <PIPELINE_NAME>

Section: Summary
  • Ticket     : <TICKET_ID>  |  Parent: <PARENT_KEY>
  • Pipeline   : <pipeline_name>
  • Requestor  : @mention the Jira reporter using ADF mention node (requires accountId lookup):
                 1. GET /rest/api/3/user/search?query=<reporter_email> → extract accountId
                 2. In ADF use: {"type":"mention","attrs":{"id":"<accountId>","text":"@<displayName>","accessLevel":""}}
                 This tags them in Jira so they receive a notification.
  • Operator   : demahato@groupon.com
  • Completed  : <date>

Section: Objects Created in Prod
Bullet list:
  ✅ staging.<external_table>  (external, GCS: grpn-dnd-prod-*)
  ✅ staging.<staging_table>   (truncate/reload per run)
  ✅ prod_groupondw.<final>    (PARTITION BY <col>, CLUSTER BY (<cols>))
  ✅ user_groupondw.<view>     (CREATE OR REPLACE VIEW)

  Note any skipped/recreated tables:
  ⏭️  <table> — skipped (existed, spec matched stable)
  🔄  <table> — replaced via CREATE OR REPLACE (spec mismatch, existing DDL backed up to Jira, user confirmed)

Section: Backfill
  • Table  : prod_groupondw.<final>
  • Source : prj-grp-curation-sox-stable  |  Teradata
  • Range  : <start> → <cutover>  on <filter_col>
  • Rows   : X,XXX,XXX

Section: Reconciliation
Table:
  | Table                     | Prod Rows  | Stable Rows | Delta | Status   |
  | prod_groupondw.<final>    | X,XXX,XXX  | X,XXX,XXX   | 0     | ✅ MATCH |

Section: Handoff
  Ready for PRE sub-task: <next_ticket_key if known, e.g. GDP-8704>
  Upstream tables confirmed: <N/A | ✅ all populated>
```

Then ask: **"Transition ticket to Done? (yes/no)"**

If yes:

```bash
# Get transitions
curl -s -u "demahato@groupon.com:$JIRA_TOKEN" \
  "https://groupondev.atlassian.net/rest/api/3/issue/<TICKET_ID>/transitions" \
  | python3 -c "
import json, sys
[print(t['id'], '—', t['name']) for t in json.load(sys.stdin)['transitions']]
"

# Apply Done
curl -s -u "demahato@groupon.com:$JIRA_TOKEN" \
  -H "Content-Type: application/json" -X POST \
  "https://groupondev.atlassian.net/rest/api/3/issue/<TICKET_ID>/transitions" \
  -d '{"transition":{"id":"<DONE_TRANSITION_ID>"}}'
```

Print: `✅ GDP-XXXX transitioned to Done.`

---

## Error Handling

| Situation | Action |
|---|---|
| Ticket not found / wrong project | Stop. Report the error and check ticket ID. |
| Ticket already Done | Report and stop — nothing to do. |
| Table already exists | **HARD STOP** — capture existing DDL and post to Jira as backup first. Then show spec comparison and ask: replace (CREATE OR REPLACE) / skip / abort. Never use `bq rm` or DROP. Never auto-proceed. |
| DDL extraction returns empty | Check stable project access; try `ddl/*.sql` in the repo as fallback. |
| Backfill row count mismatch > 0.01% | Flag as blocker — do NOT transition ticket. Post a ⚠️ Jira comment and ask user how to proceed. |
| View creation fails (table not found) | Ensure final table was created first — views must come last. |
| GCS external table URI wrong | Verify bucket: `gsutil ls gs://grpn-dnd-prod-*/` before creating. |
| Any bq command fails | Show full error output. Post a ❌ Jira comment documenting where it failed and the error. Stop. |

---

## Notes

- **Post a Jira comment after every step** — even on partial success. The ticket is the source of truth for what was done.
- **Never drop datasets** — only create missing ones. If a dataset exists, leave it untouched.
- **Never use `bq rm` or any DROP command** — not on tables, not on views, not on datasets, ever.
- **Table exists → backup first, then CREATE OR REPLACE** — always capture the existing prod DDL and post it to Jira before applying any change. Use `CREATE OR REPLACE TABLE` / `CREATE OR REPLACE EXTERNAL TABLE` / `CREATE OR REPLACE VIEW`, never drop-and-recreate.
- **Never backfill staging tables** — they are truncated each run; history wastes space and gets overwritten on first run.
- **Always match partition/cluster spec to stable exactly** — a mismatch causes DVT failures and silent dedup bugs.
- **External tables** require both the project ID swap AND the GCS bucket swap (`grpn-dnd-stable-` → `grpn-dnd-prod-`). Always create via BQ REST API (SQL DDL fails in prod for external tables with hive partitioning).
- **External table URI prefix** — the prod API requires CUSTOM mode with a **type-encoded** URI prefix: `gs://.../{p_region:STRING}/{dt:STRING}`. A plain URI (as seen in stable) is rejected by the current prod API. When the URI is type-encoded, `p_region`/`dt` must NOT be in the schema (BQ rejects them as duplicate partition keys). They remain fully queryable as partition columns regardless. Do NOT try to match stable's plain URI — it is a legacy artifact from before the API enforced encoding.
- **Post full DDL to Jira immediately after each object is created** — do not batch at end. Fetch from INFORMATION_SCHEMA (tables: `ddl` column, views: `view_definition` column) or BQ REST API for external tables.
- **Teradata is preferred** over stable for backfill — stable may only hold partial or test-scale data.
- **`--location=us-central1` is required** on every `bq` command.
- **Backfill date range must be confirmed with Payments Engineering** before running — start dates vary per pipeline.
- **Table-exists is a HARD STOP** — never silently skip or overwrite. Always ask explicitly, one object at a time.
