---
name: init-keyword-research
description: "SEO Domination Engine Step 4: Keyword Research. Validates Steps 1-3b are complete, checks CDP connection, reads Step-4 Steps.md, generates batch seed keywords from previous deliverables, runs Google Keyword Planner batches via agent-browser, exports CSVs, loops until keyword target is met, consolidates all keywords into master CSV, shortlists best keywords, and writes summary.md. Must be run from inside Pillar-Content-Architecture/Content-Creation-Workflow/. Triggers: 'init keyword research', 'run keyword research', 'keyword research', 'step 4', 'keyword planner', 'run kp batches', 'start keyword research'."
---

# Init Keyword Research (Step 4)

## Prerequisites

Must be run from inside `Pillar-Content-Architecture/Content-Creation-Workflow/`.

---

## AUTOPILOT RULES — READ FIRST

- **Never stop or pause after Phase 0.** No questions to the user mid-run.
- **Never skip a phase.** Complete every phase and sub-step in order.
- **Loop batches in iterations.** Each iteration consists of 6–10 batches (minimum 6, maximum 10 per iteration — never exceed 10 in a single iteration). After completing an iteration, check if `total_keywords_to_research` is met. If not, start a new iteration of 6–10 batches with fresh seed keywords. Repeat iterations until the target is met. There is no cap on the number of iterations — only the per-iteration batch count is capped at 10.
- **Self-heal on failure.** If a tool call fails, retry once silently. If it fails again, log inline and continue.
- **agent-browser is the only hard stop.** If CDP fails at any point, tell the user: "agent-browser connection failed — cannot continue. Please check the CDP endpoint and try again." Then stop.
- **Do not summarise phases.** Just do the work and mark todos complete.
- **Snapshot first, screenshot second.** To understand page structure, content, or where to click — always use `agent-browser snapshot` (DOM snapshot) as the primary method. Only fall back to reading a screenshot if the snapshot does not provide enough information.

---

## PHASE -1 — Prerequisite Check

Check that all four deliverables from Steps 1–3b exist and are non-empty:

```bash
test -s Step-1-Brand-Discovery/brand-discovery.md && echo "Step 1 OK" || echo "MISSING: brand-discovery.md"
test -s Step-2-Competitor-Discovery/competitor-discovery.md && echo "Step 2 OK" || echo "MISSING: competitor-discovery.md"
test -s Step-3-Content-Gap-Analysis/content-gap-analysis.md && echo "Step 3 OK" || echo "MISSING: content-gap-analysis.md"
test -s Step-3b-Content-Scope-Estimation/scope-estimation.md && echo "Step 3b OK" || echo "MISSING: scope-estimation.md"
```

If any file is missing → tell the user exactly which step is incomplete and stop. Do not proceed.

If all four exist → continue.

---

## PHASE 0 — CDP Check & Cleanup

### Step 1 — Derive WebSocket URL

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP_HTTP=$(jq -r '.cdp_http' config.json)
CDP_HOST=$(echo "$CDP_HTTP" | sed 's|https://||')
WS_URL=$(curl -s "$CDP_HTTP/json/version" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data['webSocketDebuggerUrl'].replace('ws://localhost', 'wss://$CDP_HOST'))
")
echo "WebSocket URL: $WS_URL"
```

### Step 2 — Write cdp_ws to config.json

```bash
jq --arg ws "$WS_URL" '.cdp_ws = $ws' config.json > config.tmp && mv config.tmp config.json
```

All subsequent agent-browser calls use:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
agent-browser --cdp "$CDP" <command>
```

Always read `cdp_ws` fresh from config.json — never hardcode the WebSocket URL.

### Step 3 — Close all tabs except one

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)

# List all open tabs
agent-browser --cdp "$CDP" tab

# Close every tab above index 1 in sequence
# agent-browser --cdp "$CDP" tab close 2
# agent-browser --cdp "$CDP" tab close 3
# (repeat until only 1 tab remains)
```

Run `tab` first to see the count, then close each one above index 1.

### Step 4 — Verify connection AND Google Ads access

Read the Keyword Planner URL from config.json and open it:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
KP_URL=$(jq -r '.agent_browser.keyword_planner_url' config.json)

# Open Keyword Planner home to verify both browser access and Ads account access
agent-browser --cdp "$CDP" open "$KP_URL"
```

Take a snapshot to check the page loaded correctly (snapshot first):
```bash
agent-browser --cdp "$CDP" snapshot
```

If the snapshot shows the Keyword Planner UI → access confirmed, continue.
If the snapshot shows a login page, error, or access denied → hard stop. Tell the user: "Cannot access Google Keyword Planner — check that the CDP browser is logged into the correct Google Ads account (authuser=1, customer ID 7937773813)." Then stop.

Take a screenshot for the record:
```bash
agent-browser --cdp "$CDP" screenshot --output Step-4-Keyword-Research/screenshots/cdp-kp-access-check.png
```

---

## PHASE 1 — Read Steps.md & Load Config

Read `Step-4-Keyword-Research/Steps.md` fully and internalize every phase and sub-step.

Then read these values from `config.json`:
- `scope.total_keywords_to_research` — the keyword target to hit
- `scope.kp_batches` — planned number of batches
- `brand_name` — for context
- `target_markets` — for geo targeting in Keyword Planner
- `website_url` — for context

Also read all four deliverables:
- `Step-1-Brand-Discovery/brand-discovery.md`
- `Step-2-Competitor-Discovery/competitor-discovery.md`
- `Step-3-Content-Gap-Analysis/content-gap-analysis.md`
- `Step-3b-Content-Scope-Estimation/scope-estimation.md`

---

## PHASE 2 — Create TodoList

Using the TodoWrite tool, create a granular todo list:

- PHASE -1: Prerequisite check
- PHASE 0: CDP check, tab cleanup, connection verify
- PHASE 1: Read Steps.md, load config, read deliverables
- PHASE 2: Generate batch seeds
- PHASE 3: For each batch — open KP, enter seeds, export CSV, save file
- PHASE 3b: Quality check after each batch — count, relevance, volume
- PHASE 3c: Generate new batch if target not met, repeat
- PHASE 4: Consolidate all batch CSVs into master-keywords.csv
- PHASE 4b: Shortlist best keywords into shortlisted-keywords.csv
- PHASE 5: Write summary.md, update Master-Matrix.md, update HTML

---

## PHASE 2 — Generate Batch Seeds

Using the four deliverables as input, derive seed keyword batches:

**Rules:**
- 2–3 keywords per batch (hard limit — never more than 3)
- Each keyword must be directly relevant to the client's products or content pillars
- Keywords should cover different pillars/product categories across batches
- Assign a `product_category` label to each batch based on which client product area the seeds target
- Plan enough batches to realistically hit `scope.total_keywords_to_research`

**Output:** A numbered batch plan, e.g.:
```
Batch 001 — product_category: [category] — seeds: [kw1, kw2]
Batch 002 — product_category: [category] — seeds: [kw1, kw2, kw3]
...
```

Create `Step-4-Keyword-Research/batches/` directory if it doesn't exist.

---

## PHASE 3 — Run Batches in Google Keyword Planner

At the start of every batch, always read `cdp_ws` fresh from config.json — never hardcode the WebSocket URL:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
```

Use `$CDP` in every agent-browser call throughout this phase.

### 3.0 Tab cleanup before each batch

Run this at the start of **every** batch, not just the first:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)

# List all open tabs
agent-browser --cdp "$CDP" tab

# Close every tab above index 1 in sequence
agent-browser --cdp "$CDP" tab close 2
# agent-browser --cdp "$CDP" tab close 3
# (repeat until only tab 1 remains)

# Switch to tab 1 to confirm it is active
agent-browser --cdp "$CDP" tab 1
```

### 3.1 Navigate to Keyword Planner

Read the ideas URL from config.json — never hardcode it:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
KP_IDEAS_URL=$(jq -r '.agent_browser.keyword_planner_ideas_url' config.json)
agent-browser --cdp "$CDP" open "$KP_IDEAS_URL"
```

Snapshot first to confirm the page loaded and identify the input field:
```bash
agent-browser --cdp "$CDP" snapshot
```

Then screenshot for the record:
```bash
agent-browser --cdp "$CDP" screenshot --output Step-4-Keyword-Research/screenshots/batch-NNN-open.png
```

### 3.2 Enter seed keywords

- Click into the keyword input field
- Type the 2–3 seed keywords for this batch (one per line or comma-separated)
- Set location/geo targeting to match `target_markets` from config.json
- Click "Get results"

Screenshot after results load:
```bash
agent-browser --cdp "$CDP" screenshot --output Step-4-Keyword-Research/screenshots/batch-NNN-results.png
```

### 3.3 Export CSV

- Click the download/export button in Keyword Planner
- Save to `Step-4-Keyword-Research/batches/batch-NNN.csv`

If the browser saves to a default downloads location, move it:
```bash
mv ~/Downloads/keyword_ideas*.csv Step-4-Keyword-Research/batches/batch-NNN.csv
```

Final screenshot confirming export:
```bash
agent-browser --cdp "$CDP" screenshot --output Step-4-Keyword-Research/screenshots/batch-NNN-exported.png
```

---

## PHASE 3b — Quality Check After Each Batch

After each batch CSV is saved:

1. Count the keywords in the batch file
2. Sum the running total of keywords collected across all batches so far
3. Check relevance: are the returned keywords on-topic for the client's products/pillars?
4. Check volume: are avg_monthly_searches values meaningful (not all zeros)?

**Decision:**
- If `running_total >= total_keywords_to_research` AND quality is acceptable → proceed to Phase 4
- If `running_total < total_keywords_to_research` OR quality is poor → go to Phase 3c

---

## PHASE 3c — Generate New Batch If Needed

If the target is not met:
1. Generate a new batch of 2–3 seed keywords — use different seeds than previous batches, targeting unexplored pillars or product categories
2. Increment the batch number
3. Return to Phase 3 and run the new batch
4. Repeat Phase 3b quality check

Continue looping until `total_keywords_to_research` is met with satisfactory keywords.

---

## PHASE 4 — Consolidate Into Master CSV

Merge all `Step-4-Keyword-Research/batches/batch-NNN.csv` files into a single master CSV.

Output file: `Step-4-Keyword-Research/master-keywords.csv`

Columns (in this exact order):
```
id,keyword,avg_monthly_searches,volume,competition_label,competition_index,kd,kd_level,three_month_change,yoy_change,trend_growth,intent,priority,product_category,score,bid_low,bid_high,source,geo
```

**Column mapping from Keyword Planner CSV:**
- `id` → sequential integer starting at 1
- `keyword` → Keyword (by relevance)
- `avg_monthly_searches` → Avg. monthly searches
- `volume` → same as avg_monthly_searches
- `competition_label` → Competition (Low / Medium / High)
- `competition_index` → Competition (indexed value)
- `kd` → leave blank
- `kd_level` → leave blank
- `three_month_change` → Three month change
- `yoy_change` → YoY change
- `trend_growth` → derive from yoy_change (positive/flat/negative)
- `intent` → leave blank
- `priority` → derive: High if avg_monthly_searches > 1000 and competition_label = Low or Medium; Medium if searches > 200; Low otherwise
- `product_category` → from the batch this keyword came from
- `score` → leave blank
- `bid_low` → Top of page bid (low range)
- `bid_high` → Top of page bid (high range)
- `source` → "Google Keyword Planner"
- `geo` → target_markets from config.json (comma-separated if multiple)

Remove duplicate keywords (keep the entry with the highest avg_monthly_searches).

---

## PHASE 4b — Shortlist Keywords

From `master-keywords.csv`, select the best keywords based on criteria from `scope-estimation.md` and Step-3b findings.

Shortlisting criteria (apply in order):
1. Remove keywords with avg_monthly_searches = 0
2. Prefer Low and Medium competition over High
3. Prefer keywords with positive or flat trend_growth
4. Prioritise keywords that map directly to identified content gaps from Step 3
5. Aim for a spread across all product_categories — don't over-index on one category

Output file: `Step-4-Keyword-Research/shortlisted-keywords.csv`

Use the same column structure as master-keywords.csv.

---

## PHASE 5 — Wrap Up

### 5.1 Write summary.md

Write `Step-4-Keyword-Research/summary.md` covering:
- Total keywords collected across all batches
- Number of batches run
- Total keywords in master CSV (after deduplication)
- Total keywords in shortlisted CSV
- Top 5 keywords by avg_monthly_searches
- Competition breakdown (how many Low / Medium / High)
- Product category coverage
- Key observations and recommendations for Step 5 (Pillar Architecture)

### 5.2 Update Master-Matrix.md

Mark Step 4 as complete in the progress tracker.

### 5.3 Update seo-domination-report.html

Read `config.json`. Do a full `{{PLACEHOLDER}}` replacement pass on `seo-domination-report.html` using the `template_vars` mapping. Write the updated file back to disk.