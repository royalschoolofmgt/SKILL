---
name: Phase1
description: "Master orchestrator for SEO Domination Engine Phase 1 (content analysis). Runs the full pipeline in one autonomous stretch: workspace init (Stage A) → brand/competitor/gap/scope analysis Steps 1-3b (Stage B) → keyword research Step 4 (Stage C) → final JSON deliverable assembly (Stage D). Asks the user 3 questions once, then runs unattended with micro-step checkpointing in pipeline-state.json. Resumes from the last recorded micro-step on re-invocation. Single CDP browser, all browser ops sequential. Triggers: 'run phase 1', 'phase 1', 'master orchestrator', 'run content analysis pipeline', 'full content analysis', 'end-to-end seo', 'start phase 1', 'kick off seo phase 1', 'resume phase 1'."
---

# Phase 1 — Master Orchestrator (Content Analysis, end-to-end)

**Folder layout (this directory `Phase1/`):**

```
Phase1/
├── Phase1.md                              ← this file (orchestrator)
├── agent-browser.md                       ← browser usage + self-heal reference
├── init-pillar-content-architecture.md    ← Stage A child skill
├── init-pillar-content-analysis.md        ← Stage B child skill
└── init-keyword-research.md               ← Stage C child skill
```

**Workspace root (created by Stage A):** `Pillar-Content-Architecture/` — this is where `config.json`, every `Step-N-*` folder, the deliverables, and the state file live. There is **no** `Content-Creation-Workflow/` subfolder; the cloned template's root *is* the workflow root.

---

## ABSOLUTE RULES (read before every invocation)

1. **Single CDP, sequential only.** One Chrome instance via CDP. Never issue parallel `agent-browser` calls. Never spawn parallel subagents that touch the browser. Every browser operation across every stage is serialised.
2. **State JSON is truth.** `pipeline-state.json` (in workspace root) is updated after every micro-step. On resume, read it first and continue from `last_step`. Never trust in-memory state across runs.
3. **One-shot question phase.** Ask the user the 3 inputs (Section 1) once. After that, never ask anything until Phase 1 finishes or hard-stops.
4. **Resume-first.** On every invocation, read `pipeline-state.json` and the gate files. Skip any stage already `completed`. For any stage `in_progress`, resume from `last_step`.
5. **Self-heal via `agent-browser` skill.** When a child reports a recoverable browser issue, consult the `agent-browser` skill's recovery decision tree before re-delegating.
6. **Hard stops only on:** (a) agent-browser CDP failure that recovery can't fix, (b) same micro-step failing twice in a row, (c) missing user input in Section 1.
7. **Two completion artefacts.** Phase 1 is only "done" when **both** exist: `.phase-1-done` AND `Pillar-Content-Architecture/phase-1-deliverable.json` (Section 8).

---

## CHILD STAGES

| Stage | Child skill | Working directory | Gate file (proof) |
|---|---|---|---|
| **A. Architecture** | `init-pillar-content-architecture` | current dir (parent of `Pillar-Content-Architecture/`) | `Pillar-Content-Architecture/.architecture-done` |
| **B. Analysis (Steps 1–3b)** | `init-pillar-content-analysis` | `Pillar-Content-Architecture/` | `Pillar-Content-Architecture/.analysis-done` |
| **C. Keyword Research (Step 4)** | `init-keyword-research` | `Pillar-Content-Architecture/` | `Pillar-Content-Architecture/.keyword-research-done` |
| **D. Deliverable JSON assembly** | (orchestrator-internal, Section 8) | `Pillar-Content-Architecture/` | `Pillar-Content-Architecture/phase-1-deliverable.json` |
| **E. Content map JSON assembly** | (orchestrator-internal, Section 8b) | `Pillar-Content-Architecture/` | `Pillar-Content-Architecture/content_map.json` + `.phase-1-done` |

---

## SECTION 1 — Collect inputs (only on first run)

```bash
# If state file exists with inputs, this is a resume — skip the question
if [ -f "Pillar-Content-Architecture/pipeline-state.json" ] \
   && jq -e '.inputs.brand_name' "Pillar-Content-Architecture/pipeline-state.json" > /dev/null 2>&1; then
  echo "Resuming — inputs already captured."
  RESUME=1
else
  RESUME=0
fi
```

If `RESUME=0`, ask the user in **one** message:

1. Client / brand name?
2. Client website URL?
3. Target market(s)? (comma-separated, e.g. UK, Ireland, USA)

Wait for all three. Do not proceed otherwise.

Then create the workspace folder shell (Stage A will populate it) and write the initial state:

```bash
mkdir -p Pillar-Content-Architecture
jq -n \
  --arg brand "$BRAND" \
  --arg url "$URL" \
  --arg markets "$MARKETS" \
  '{
    inputs: { brand_name: $brand, website_url: $url, target_markets: $markets },
    stages: {
      architecture:     { status: "pending", last_step: null, attempts: 0 },
      analysis:         { status: "pending", last_step: null, attempts: 0 },
      keyword_research: { status: "pending", last_step: null, attempts: 0, batches_done: 0, calibration: [] },
      deliverable:      { status: "pending", last_step: null, attempts: 0 },
      content_map:      { status: "pending", last_step: null, attempts: 0 }
    },
    started_at: (now | todate),
    updated_at: (now | todate)
  }' > Pillar-Content-Architecture/pipeline-state.json
```

---

## SECTION 2 — State JSON schema (canonical)

`pipeline-state.json` lives at `Pillar-Content-Architecture/pipeline-state.json`. Every child skill and the orchestrator update it after every micro-step.

```json
{
  "inputs": {
    "brand_name": "string",
    "website_url": "string",
    "target_markets": "comma-separated string"
  },
  "stages": {
    "architecture":     { "status": "pending|in_progress|completed|failed", "last_step": "string|null", "attempts": 0, "last_error": "string?" },
    "analysis":         { "status": "...", "last_step": "0.3 | 1.2 | 3b.4 | ...", "attempts": 0, "last_error": "string?" },
    "keyword_research": { "status": "...", "last_step": "...", "attempts": 0, "batches_done": 0, "calibration": [{"after_batch": 0, "scores": {}}], "last_error": "string?" },
    "deliverable":      { "status": "...", "last_step": "...", "attempts": 0, "last_error": "string?" },
    "content_map":      { "status": "...", "last_step": "...", "attempts": 0, "last_error": "string?" }
  },
  "started_at": "ISO-8601",
  "updated_at": "ISO-8601",
  "completed_at": "ISO-8601?"
}
```

**Update pattern (used everywhere):**

```bash
jq --arg s "in_progress" --arg step "<phase>.<substep>" \
   '.stages.<stage>.status = $s | .stages.<stage>.last_step = $step | .updated_at = (now | todate)' \
   Pillar-Content-Architecture/pipeline-state.json > /tmp/ps.tmp \
   && mv /tmp/ps.tmp Pillar-Content-Architecture/pipeline-state.json
```

**On any failure:**

```bash
jq --arg err "<error message>" \
   '.stages.<stage>.status = "failed" | .stages.<stage>.last_error = $err | .stages.<stage>.attempts += 1 | .updated_at = (now | todate)' \
   Pillar-Content-Architecture/pipeline-state.json > /tmp/ps.tmp \
   && mv /tmp/ps.tmp Pillar-Content-Architecture/pipeline-state.json
```

---

## SECTION 3 — Resume detection

Before delegating each stage, check both the state file and the gate file:

```bash
STATE=Pillar-Content-Architecture/pipeline-state.json
A_STATUS=$(jq -r '.stages.architecture.status' "$STATE")
B_STATUS=$(jq -r '.stages.analysis.status' "$STATE")
C_STATUS=$(jq -r '.stages.keyword_research.status' "$STATE")
D_STATUS=$(jq -r '.stages.deliverable.status' "$STATE")

[ "$A_STATUS" = "completed" ] && [ -f Pillar-Content-Architecture/.architecture-done ] && SKIP_A=1
[ "$B_STATUS" = "completed" ] && [ -f Pillar-Content-Architecture/.analysis-done ] && SKIP_B=1
[ "$C_STATUS" = "completed" ] && [ -f Pillar-Content-Architecture/.keyword-research-done ] && SKIP_C=1
[ "$D_STATUS" = "completed" ] && [ -f Pillar-Content-Architecture/phase-1-deliverable.json ] && SKIP_D=1
E_STATUS=$(jq -r '.stages.content_map.status' "$STATE")
[ "$E_STATUS" = "completed" ] && [ -f Pillar-Content-Architecture/content_map.json ] && SKIP_E=1
```

If status is `in_progress` or `failed`, **do not skip** — re-delegate the stage, telling the child its `last_step` so it can resume from there.

---

## SECTION 4 — Stage A: Architecture

**Working dir:** current directory (parent of `Pillar-Content-Architecture/`).

If `SKIP_A` → mark complete in todos, move on.

Else:
1. Set state: `architecture.status = in_progress`, `last_step = "starting"`.
2. Delegate to a fresh subagent:

   > Execute the `init-pillar-content-architecture` skill end-to-end from the current working directory. It will clone the template into `Pillar-Content-Architecture/`. Do not ask the user any questions. Update `pipeline-state.json` after each step. On success, confirm `Pillar-Content-Architecture/.architecture-done` exists and reply `STAGE_A_DONE`. On failure reply `STAGE_A_FAILED <reason>`.

3. **Gate check:** confirm both files exist —
   - `Pillar-Content-Architecture/.architecture-done`
   - `Pillar-Content-Architecture/config.json`
4. If gate passes → set `architecture.status = completed`. Move to Stage B.
5. If gate fails and `attempts < 2` → re-delegate.
6. If `attempts == 2` → hard stop with the failure message in `last_error`.

---

## SECTION 5 — Stage B: Analysis (Steps 1–3b)

**Working dir:** `Pillar-Content-Architecture/`

If `SKIP_B` → skip.

Pre-populate `config.json` so the child skill does **not** ask the user the 3 input questions:

```bash
cd Pillar-Content-Architecture
BRAND=$(jq -r '.inputs.brand_name' pipeline-state.json)
URL=$(jq -r '.inputs.website_url' pipeline-state.json)
MARKETS=$(jq -r '.inputs.target_markets' pipeline-state.json)
MARKETS_JSON=$(echo "$MARKETS" | jq -R 'split(",") | map(gsub("^\\s+|\\s+$";""))')

jq --arg b "$BRAND" --arg u "$URL" --argjson m "$MARKETS_JSON" \
   '.brand_name = $b | .website_url = $u | .target_markets = $m' \
   config.json > /tmp/c.tmp && mv /tmp/c.tmp config.json
cd ..
```

Set state: `analysis.status = in_progress`, `last_step = "0.1_skipped"`.

Delegate to a fresh subagent:

> Execute the `init-pillar-content-analysis` skill end-to-end from `Pillar-Content-Architecture/`. The 3 user inputs are already in `config.json` — **skip Phase 0.1** and start at Phase 0.2. If `pipeline-state.json` shows a `stages.analysis.last_step`, resume from that step. Update `pipeline-state.json` after every micro-step. Browser ops are serialised (single CDP). On any browser failure, consult the `agent-browser` skill's recovery decision tree before retrying. On success confirm `.analysis-done` exists and reply `STAGE_B_DONE`. On failure reply `STAGE_B_FAILED <last_step>: <reason>`.

**Gate check:**
- `Pillar-Content-Architecture/.analysis-done` exists
- `Pillar-Content-Architecture/Step-1-Brand-Discovery/brand-discovery.md` non-empty
- `Pillar-Content-Architecture/Step-2-Competitor-Discovery/competitor-discovery.md` non-empty
- `Pillar-Content-Architecture/Step-3-Content-Gap-Analysis/content-gap-analysis.md` non-empty
- `Pillar-Content-Architecture/Step-3b-Content-Scope-Estimation/scope-estimation.md` non-empty
- `jq -e '.scope.total_keywords_to_research' Pillar-Content-Architecture/config.json` returns a number > 0

Same retry rule.

---

## SECTION 6 — Stage C: Keyword Research

**Working dir:** `Pillar-Content-Architecture/`

If `SKIP_C` → skip.

Set state: `keyword_research.status = in_progress`.

Delegate to a fresh subagent:

> Execute the `init-keyword-research` skill end-to-end from `Pillar-Content-Architecture/`. Prereqs are met. If `pipeline-state.json` shows `stages.keyword_research.last_step` and `batches_done`, resume from that point — do not re-run completed batches (skip any batch whose CSV already exists in `Step-4-Keyword-Research/batches/`). Browser ops are serialised. On any browser failure, consult `agent-browser` skill recovery before retrying. On success confirm `.keyword-research-done` exists and reply `STAGE_C_DONE`. On failure reply `STAGE_C_FAILED <last_step>: <reason>`.

**Gate check:**
- `Pillar-Content-Architecture/.keyword-research-done`
- `Pillar-Content-Architecture/Step-4-Keyword-Research/master-keywords.csv` non-empty
- `Pillar-Content-Architecture/Step-4-Keyword-Research/shortlisted-keywords.csv` non-empty
- `Pillar-Content-Architecture/Step-4-Keyword-Research/summary.md` non-empty

Same retry rule.

---

## SECTION 7 — Stage D: Deliverable JSON assembly

**Working dir:** `Pillar-Content-Architecture/`. Done by the orchestrator itself, no subagent needed.

If `SKIP_D` → skip.

Set state: `deliverable.status = in_progress`, `last_step = "starting"`.

Read these inputs:
- `config.json` (has `brand_name`, `website_url`, `target_markets`, `scope.*`, `template_vars`)
- `Step-1-Brand-Discovery/brand-discovery.md`
- `Step-3b-Content-Scope-Estimation/scope-estimation.md`
- `Step-4-Keyword-Research/master-keywords.csv`
- `Step-4-Keyword-Research/shortlisted-keywords.csv`
- `Step-4-Keyword-Research/summary.md`

Compute aggregates from the keyword CSVs:
- Total keywords (line count of master, minus header)
- Sum of `avg_monthly_searches` across master = total volume/month
- Estimated total words = `scope.total_articles * avg_words_per_article` (default `avg_words_per_article` = 2000 if not in config)
- Priority distribution: count rows in master grouped by `priority` (Critical / High / Medium / Low — note `init-keyword-research` Phase 4 currently produces only High/Medium/Low; treat Critical = 0 unless populated by Stage B)

Build pillars list:
- Read `scope.pillars` count from config
- For each pillar entry in `scope.pillar_definitions[]` (if present in config) capture: number, name, tier, spokes count, keywords count, vol/mo total, avg KD, mapped_blog_url
- If `scope.pillar_definitions[]` is missing → emit `pillars: []` and add a warning to the deliverable's `notes` field

Build deployment tiers:
- T1 Quick Wins / T2 Foundation / T3 Authority — counts come from `scope.deployment_tiers.{t1,t2,t3}` in config; if missing default to `null` and add to `notes`.

Write the JSON (Section 8 schema) atomically:

```bash
jq -n \
  --slurpfile cfg config.json \
  --arg brand "$(jq -r '.brand_name' config.json)" \
  --argjson pillars "<computed>" \
  --argjson priority "<computed>" \
  --argjson tiers "<computed>" \
  '...' > phase-1-deliverable.json.tmp
mv phase-1-deliverable.json.tmp phase-1-deliverable.json
```

Then:

```bash
jq '.stages.deliverable.status = "completed" | .stages.deliverable.last_step = "json_written" | .updated_at = (now | todate)' \
   pipeline-state.json > /tmp/ps.tmp && mv /tmp/ps.tmp pipeline-state.json
```

(Phase 1 is **not** done yet — Stage E builds the second deliverable next.)

---

## SECTION 8 — Final deliverable JSON schema

Path: `Pillar-Content-Architecture/phase-1-deliverable.json`

This is the canonical output of Phase 1 — it powers the `VortexIQ App.html` dashboard. Every key is required; use `null` (not omission) for unknowns and add an entry to `notes[]` explaining why.

```json
{
  "schema_version": "1.0",
  "generated_at": "ISO-8601",
  "brand": {
    "name": "string",
    "website_url": "string",
    "target_markets": ["string"]
  },

  "content_planning": {
    "pillars": 0,
    "spokes": 0,
    "keywords": 0,
    "volume_per_month_total": 0,
    "total_words_estimated": 0
  },

  "executive_summary": "string (2-4 sentences from scope-estimation.md + summary.md)",
  "strategic_value": "string (1-2 sentences — why this matters for the brand)",

  "research_sources_integrated": {
    "brand_intelligence": true,
    "competitor_analysis": true,
    "live_serp_difficulty": false,
    "google_keyword_planner": true,
    "google_trends": false
  },

  "current_state_assessment": {
    "total_blog_posts": null,
    "pillar_or_hub_pages": null,
    "articles_drafted": null,
    "moz_domain_authority": null,
    "referring_domains": null,
    "indexed_pages": null
  },

  "critical_findings_and_opportunities": [
    "string (5–10 bullet points)"
  ],

  "pillar_architecture_overview": [
    {
      "pillar_number": 1,
      "pillar_name": "string",
      "pillar_tier": "T1|T2|T3",
      "spokes_count": 0,
      "keywords_count": 0,
      "volume_per_month": 0,
      "avg_kd": 0,
      "maps_to_blog_url": "string|null"
    }
  ],

  "priority_distribution": {
    "critical": 0,
    "high": 0,
    "medium": 0,
    "low": 0
  },

  "deployment_tiers": {
    "t1_quick_wins":  { "count": 0, "pillar_numbers": [] },
    "t2_foundation":  { "count": 0, "pillar_numbers": [] },
    "t3_authority":   { "count": 0, "pillar_numbers": [] }
  },

  "source_artifacts": {
    "brand_discovery_md":     "Step-1-Brand-Discovery/brand-discovery.md",
    "competitor_discovery_md":"Step-2-Competitor-Discovery/competitor-discovery.md",
    "content_gap_md":         "Step-3-Content-Gap-Analysis/content-gap-analysis.md",
    "scope_estimation_md":    "Step-3b-Content-Scope-Estimation/scope-estimation.md",
    "master_keywords_csv":    "Step-4-Keyword-Research/master-keywords.csv",
    "shortlisted_keywords_csv":"Step-4-Keyword-Research/shortlisted-keywords.csv",
    "keyword_summary_md":     "Step-4-Keyword-Research/summary.md"
  },

  "notes": [
    "string (one entry per null/default field, explaining why)"
  ]
}
```

**Field-source mapping (where each value comes from):**

| Field | Source |
|---|---|
| `brand.*` | `config.json` (already populated by Section 1) |
| `content_planning.pillars` | `config.json:scope.pillars` |
| `content_planning.spokes` | `config.json:scope.total_spokes` |
| `content_planning.keywords` | `wc -l Step-4-Keyword-Research/master-keywords.csv` minus 1 |
| `content_planning.volume_per_month_total` | `awk -F, 'NR>1{s+=$3}END{print s}' Step-4-Keyword-Research/master-keywords.csv` |
| `content_planning.total_words_estimated` | `scope.total_articles * (scope.avg_words_per_article // 2000)` |
| `executive_summary` | first paragraph of `Step-3b/scope-estimation.md` + first paragraph of `Step-4/summary.md` |
| `strategic_value` | derived from `Step-1/brand-discovery.md` "positioning" section |
| `research_sources_integrated.*` | hard-coded based on what Phase 1 actually does (see defaults above; flip to true only if a step actually ran) |
| `current_state_assessment.*` | from `Step-1/brand-discovery.md` if captured; else null + note |
| `critical_findings_and_opportunities` | top 5–10 bullets from `Step-3/content-gap-analysis.md` "opportunities" section |
| `pillar_architecture_overview[]` | `config.json:scope.pillar_definitions[]` if present; else empty + note |
| `priority_distribution.*` | grouped count of `priority` column in `master-keywords.csv` |
| `deployment_tiers.*` | `config.json:scope.deployment_tiers` if present; else zeros + note |

---

---

## SECTION 7b — Stage E: Content Map JSON assembly

**Working dir:** `Pillar-Content-Architecture/`. Orchestrator-internal, no subagent.

If `SKIP_E` → skip.

Set state: `content_map.status = in_progress`, `last_step = "starting"`.

**Purpose:** produce `content_map.json` — the second deliverable that powers `Content_Map.html`. It contains the full pillar/spoke architecture with **planned content titles**, KD, search volume, and word-count budget per spoke. **No body content is generated** — titles and metrics only.

### Inputs

- `phase-1-deliverable.json` (just written by Stage D — has pillars[], priority distribution, totals)
- `Step-3b-Content-Scope-Estimation/scope-estimation.md` (avg_spokes_per_pillar, tier mapping, rationale)
- `Step-4-Keyword-Research/master-keywords.csv` (kd, avg_monthly_searches, priority, product_category)
- `Step-4-Keyword-Research/shortlisted-keywords.csv` (the curated subset)
- `config.json` (`scope.pillar_definitions[]`, `scope.deployment_tiers`, `scope.avg_words_per_article`)

### Allocation algorithm (deterministic)

1. **Pillar list:** start from `phase-1-deliverable.json:pillar_architecture_overview[]`. Each pillar has number, name, tier, target spokes_count, mapped_blog_url.
2. **Bucket the shortlisted keywords by `product_category`** (which maps 1:1 to pillar in Step 4).
3. **Pillar hub keyword:** the highest-volume keyword in the pillar's bucket becomes the hub keyword. The hub itself gets a single "pillar page" entry — title format: `"<Pillar Name>: The Complete Guide ([Year])"`. Use `config.json:scope.pillar_target_words` (default 4000).
4. **Spoke selection:** within the pillar's bucket, sort remaining keywords by `priority` (Critical > High > Medium > Low) then by `avg_monthly_searches` desc. Take the top `spokes_count` keywords as spoke seeds.
5. **Spoke title generation:** apply title templates based on intent inferred from the keyword shape:
   - starts with `how to` → `"How to <kw remainder>: A Step-by-Step Guide"`
   - starts with `best` / `top` → `"<KW Title-Cased>: Tested & Reviewed for [Year]"`
   - contains `vs` → `"<KW>: Which Is Right for You?"`
   - ends in `?` or starts with `what` / `why` → `"<KW Title-Cased>"` (verbatim, capitalised)
   - default → `"<KW Title-Cased>: The Complete Guide"`
6. **Per-spoke word budget:** read `config.json:scope.avg_words_per_article` (default 2000). Optionally scale: priority Critical/High → ×1.2, Low → ×0.8, rounded to nearest 100.
7. **KD per spoke:** copy `kd` from the source row in `master-keywords.csv`. If blank (Phase 4 currently leaves it blank), set `null` and add a `notes[]` entry.
8. **Cross-pillar bridges:** for every pair of pillars, find shortlisted keywords whose tokens overlap across both pillars' product_categories. If overlap exists, add a bridge entry. (Cap at 2 bridges per pillar pair to avoid noise.)

### Write the file atomically

```bash
# (Build $CONTENT_MAP_JSON via jq from the inputs above, then:)
echo "$CONTENT_MAP_JSON" > content_map.json.tmp
mv content_map.json.tmp content_map.json

touch .phase-1-done
jq '.stages.content_map.status = "completed" | .stages.content_map.last_step = "json_written" | .completed_at = (now | todate) | .updated_at = (now | todate)' \
   pipeline-state.json > /tmp/ps.tmp && mv /tmp/ps.tmp pipeline-state.json
```

---

## SECTION 8b — content_map.json schema

Path: `Pillar-Content-Architecture/content_map.json`

This is the **second** Phase 1 deliverable. It powers `Content_Map.html`. **No article body content is in this file** — only planning metadata: titles, KD, volume, word budget, link plan.

```json
{
  "schema_version": "1.0",
  "generated_at": "ISO-8601",
  "brand": {
    "name": "string",
    "website_url": "string",
    "target_markets": ["string"]
  },

  "totals": {
    "pillars": 0,
    "spokes": 0,
    "keywords_used": 0,
    "volume_per_month_total": 0,
    "estimated_total_words": 0
  },

  "deployment_tiers": {
    "t1_quick_wins":  { "label": "Quick Wins", "pillar_numbers": [] },
    "t2_foundation": { "label": "Foundation",  "pillar_numbers": [] },
    "t3_authority":  { "label": "Authority",   "pillar_numbers": [] }
  },

  "pillars": [
    {
      "pillar_number": 1,
      "pillar_name": "string",
      "tier": "T1|T2|T3",
      "status": "draft",
      "why_this_pillar": "string (1–2 sentence strategic rationale)",
      "maps_to_blog_url": "string|null",

      "hub": {
        "type": "pillar_page",
        "title": "string (e.g. 'Vape Kits: The Complete Guide (2026)')",
        "slug": "string (lowercase-hyphenated)",
        "target_keyword": "string",
        "search_volume": 0,
        "kd": 0,
        "word_budget": 4000,
        "intent": "informational|commercial|transactional|navigational",
        "internal_links": {
          "links_out_to_spokes": 0,
          "links_out_to_collections": 0,
          "links_out_to_products": 0
        }
      },

      "spokes": [
        {
          "spoke_number": 1,
          "title": "string (planned title — content not yet written)",
          "slug": "string",
          "target_keyword": "string",
          "secondary_keywords": ["string"],
          "search_volume": 0,
          "kd": 0,
          "word_budget": 2000,
          "priority": "Critical|High|Medium|Low",
          "priority_score": 0,
          "intent": "informational|commercial|transactional|navigational",
          "competition_label": "Low|Medium|High",
          "trend_growth": "positive|flat|negative",
          "internal_links": {
            "links_back_to_pillar": 2,
            "links_to_other_spokes": [],
            "links_to_collections": [],
            "links_to_products": []
          },
          "selected_because": "string (1 sentence — SERP gap, low KD + good volume, etc.)",
          "content_status": "planned"
        }
      ],

      "pillar_totals": {
        "spokes_count": 0,
        "keywords_count": 0,
        "volume_per_month": 0,
        "avg_kd": 0,
        "estimated_words": 0
      }
    }
  ],

  "cross_pillar_bridges": [
    {
      "from_pillar": 1,
      "to_pillar": 3,
      "rationale": "string (e.g. 'Kits ↔ Beginners — buyers researching kits often need beginner content')",
      "bridge_keyword": "string",
      "suggested_anchor_text": "string"
    }
  ],

  "linking_rules": {
    "every_spoke_links_back_to_pillar": true,
    "every_pillar_links_out_to_all_spokes": true,
    "min_spoke_to_pillar_links": 1,
    "min_pillar_to_spoke_links": 1,
    "cross_spoke_links_allowed": true,
    "anchor_text_variation_required": true
  },

  "source_artifacts": {
    "phase_1_deliverable_json": "phase-1-deliverable.json",
    "scope_estimation_md":      "Step-3b-Content-Scope-Estimation/scope-estimation.md",
    "master_keywords_csv":      "Step-4-Keyword-Research/master-keywords.csv",
    "shortlisted_keywords_csv": "Step-4-Keyword-Research/shortlisted-keywords.csv"
  },

  "notes": [
    "string (one entry per null/default field, explaining why)"
  ]
}
```

**Important field semantics:**

- `content_status` is always `"planned"` in this file. The actual article body is written in Phase 2 by the `pillar-content-clusters` skill — Phase 1 only plans titles + metrics.
- `priority_score` (0–100): scale `priority` to numeric — Critical=90, High=70, Medium=50, Low=30. Bump ±5 based on volume bucket.
- `internal_links.links_to_other_spokes`/`collections`/`products` are arrays of slugs/URLs. Cross-pillar bridges (top-level) are the source of truth for *cross-pillar* spoke-to-spoke links; the per-spoke arrays cover *intra-pillar* links + product/collection links inferred from the keyword.
- `intent` defaults: keywords containing "buy", "price", "cheap", "deal", "discount" → `transactional`; "best", "top", "vs", "review" → `commercial`; "how to", "what is", "why", "guide" → `informational`; everything else → `commercial`.
- If `kd` is `null` in source CSV (Phase 4 currently leaves it blank), copy as `null` and add a `notes[]` entry. Do not invent values.

---

## SECTION 9 — Verification & handoff

After Stage E writes `content_map.json`, verify both deliverables and tell the user:

```bash
echo "=== PHASE 1 COMPLETE ==="
jq '{
  brand: .brand.name,
  pillars: .content_planning.pillars,
  keywords: .content_planning.keywords,
  volume_per_month: .content_planning.volume_per_month_total,
  estimated_words: .content_planning.total_words_estimated
}' Pillar-Content-Architecture/phase-1-deliverable.json

jq '{
  pillars: .totals.pillars,
  spokes: .totals.spokes,
  bridges: (.cross_pillar_bridges | length),
  null_kds: ([.. | objects | select(.kd == null)] | length)
}' Pillar-Content-Architecture/content_map.json
```

Output to user (single short message):

```
Phase 1 complete.
Deliverables:
  - Pillar-Content-Architecture/phase-1-deliverable.json   (dashboard data)
  - Pillar-Content-Architecture/content_map.json           (full pillar/spoke plan)

- Brand: <brand>
- Pillars: <n>  Spokes: <n>  Keywords: <n>
- Volume/mo: <n>  Estimated total words: <n>
- KW batches run: <n> / 20 cap   Calibration cycles: <n>

Ready for Phase 2 (content writing). The content_map.json drives `pillar-content-clusters`
— each spoke entry has a planned title, target keyword, KD, volume, and word budget,
but no body content yet.
```

---

## SECTION 10 — Failure recovery

If any stage hard-stops:
1. State JSON already records `status: "failed"`, `last_error`, and `last_step`.
2. Tell the user:
   ```
   Phase 1 stopped at Stage <X> / step <last_step>: <last_error>.
   Re-invoke `Phase1` once the issue is resolved — it will resume from that step.
   ```
3. Do not delete partial deliverables. Do not roll back completed stages.

---

## NOTES (design rationale)

- **Why subagents per stage:** child skills cannot directly call other skills, and a single context can't survive a multi-hour browser run. Fresh subagent context per stage avoids context-window thrashing.
- **Why a state JSON in addition to gate files:** gate files only encode "stage done / not done" — they cannot encode "stage in_progress, resume from step 3.4". Micro-step resume requires structured state.
- **Why pre-populate config.json before Stage B:** the analysis skill normally asks the user 3 questions in Phase 0.1. Pre-filling lets the orchestrator suppress that without modifying the child.
- **Why the deliverable JSON is built by the orchestrator (Stage D) and not by Stage C:** the deliverable aggregates outputs from all three previous stages — only the orchestrator has visibility across all of them.
- **Why one CDP, sequential:** the deployment has exactly one Chrome instance behind `chrome-cdp.vortexiq.ai`. Parallel calls collide on tabs. Enforce serially everywhere.
