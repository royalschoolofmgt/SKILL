---
name: Phase1
description: "Master orchestrator for SEO Domination Engine Phase 1 (content analysis). Executes the full pipeline in one autonomous stretch: workspace init (architecture) → brand/competitor/gap/scope analysis (Steps 1–3b) → keyword research (Step 4). Asks the user three questions once, then runs unattended with checkpoint recovery via .pipeline-state.json. Resumes from the last incomplete stage on re-invocation. Triggers: 'run phase 1', 'phase 1', 'master orchestrator', 'run content analysis pipeline', 'full content analysis', 'end-to-end seo', 'start phase 1', 'kick off seo phase 1'."
---

# Phase 1 — Master Orchestrator (Content Analysis, end-to-end)

**Child skills (in this same `Phase1/` folder):**
- `init-pillar-content-architecture.md` — Stage A
- `init-pillar-content-analysis.md` — Stage B
- `init-keyword-research.md` — Stage C

This is an **orchestrator** skill. It does not duplicate the work of the three child skills — it sequences them, validates gates between them, and persists checkpoint state so a long run can resume after interruption or context compaction.

---

## CHILD SKILLS (do not inline their logic)

| Stage | Child skill | Working directory | Gate file (proof of completion) |
|---|---|---|---|
| **A. Architecture** | `init-pillar-content-architecture` | current dir | `Pillar-Content-Architecture/Content-Creation-Workflow/.architecture-done` |
| **B. Analysis (Steps 1–3b)** | `init-pillar-content-analysis` | `Pillar-Content-Architecture/Content-Creation-Workflow/` | `.analysis-done` (in that dir) |
| **C. Keyword Research (Step 4)** | `init-keyword-research` | `Pillar-Content-Architecture/Content-Creation-Workflow/` | `.keyword-research-done` (in that dir) |

---

## AUTOPILOT RULES — READ FIRST

- **One-shot question phase.** Ask the user the three inputs (Section 1) **once**. After that, never ask anything else until the pipeline finishes or hard-stops.
- **No phase summaries.** Do the work, mark the todo complete, move on.
- **Resume-first.** On every invocation, read `.pipeline-state.json` (if present) and the three gate files. Skip any stage whose gate file exists. Restart from the first incomplete stage.
- **Hard stops only on:**
  1. agent-browser CDP failure inside a child skill (child surfaces the error; orchestrator stops).
  2. Same stage failing **twice in a row** when delegated.
  3. Missing input from the user in Section 1.
- **Soft failures self-heal.** If a child reports a recoverable issue, the orchestrator records it in `.pipeline-state.json` and re-delegates the same stage **once** before escalating to a hard stop.
- **Inputs persist across resumes.** The three answers are written to `.pipeline-state.json` immediately. If the file already has them, do not re-ask.

---

## SECTION 1 — Collect Inputs (only on first run)

Check first:

```bash
if [ -f ".pipeline-state.json" ] && jq -e '.inputs.brand_name' .pipeline-state.json > /dev/null 2>&1; then
  echo "Resuming — inputs already captured."
else
  echo "First run — need inputs."
fi
```

If first run, ask the user exactly these three questions in **one** message:

1. Client / brand name?
2. Client website URL?
3. Target market(s)? (comma-separated, e.g. UK, Ireland, USA)

Wait for the response. Do not proceed without all three.

Then write `.pipeline-state.json`:

```bash
jq -n \
  --arg brand "$BRAND" \
  --arg url "$URL" \
  --arg markets "$MARKETS" \
  '{
    inputs: { brand_name: $brand, website_url: $url, target_markets: $markets },
    stages: {
      architecture: { status: "pending", attempts: 0 },
      analysis:     { status: "pending", attempts: 0 },
      keyword_research: { status: "pending", attempts: 0 }
    },
    started_at: now | todate
  }' > .pipeline-state.json
```

---

## SECTION 2 — Build the Master TodoList

Use TaskCreate to create these tasks (in order, with `addBlockedBy` chaining each on the previous):

1. **Stage A — Initialise architecture workspace**
2. **Stage B — Run content analysis (Steps 1–3b)**
3. **Stage C — Run keyword research (Step 4)**
4. **Stage D — Final verification + handoff to Phase 2**

Mark Stage A as `in_progress` when starting.

---

## SECTION 3 — Resume Detection

Before delegating each stage, check its gate file. If present, mark the task completed and skip:

```bash
# Stage A gate
[ -f "Pillar-Content-Architecture/Content-Creation-Workflow/.architecture-done" ] && SKIP_A=1

# Stage B gate (path only valid after Stage A)
[ -f "Pillar-Content-Architecture/Content-Creation-Workflow/.analysis-done" ] && SKIP_B=1

# Stage C gate
[ -f "Pillar-Content-Architecture/Content-Creation-Workflow/.keyword-research-done" ] && SKIP_C=1
```

Update `.pipeline-state.json` to reflect any skipped stages as `completed`.

---

## SECTION 4 — Stage A: Architecture

Mark stage A `in_progress` in `.pipeline-state.json`.

Delegate to a fresh subagent (Agent tool, `subagent_type: general-purpose`) with this prompt:

> Execute the `init-pillar-content-architecture` skill end-to-end from the current working directory. Do not ask the user any questions. When done, confirm `Pillar-Content-Architecture/Content-Creation-Workflow/.architecture-done` exists. Reply with exactly `STAGE_A_DONE` on success or `STAGE_A_FAILED <reason>` on failure.

**Gate check:** confirm `Pillar-Content-Architecture/Content-Creation-Workflow/.architecture-done` exists AND `Master-Matrix.md` exists in that folder.

- If gate passes → mark stage A `completed`, move to Stage B.
- If gate fails and attempts < 2 → increment attempts, re-delegate.
- If attempts == 2 → hard stop with `STAGE_A_FAILED` reason.

---

## SECTION 5 — Stage B: Analysis (Steps 1–3b)

Change into the workflow directory:

```bash
cd Pillar-Content-Architecture/Content-Creation-Workflow
```

Mark stage B `in_progress`.

Before delegating, write the three user inputs into `config.json` so the child skill does **not** re-prompt:

```bash
BRAND=$(jq -r '.inputs.brand_name' ../../.pipeline-state.json)
URL=$(jq -r '.inputs.website_url' ../../.pipeline-state.json)
MARKETS=$(jq -r '.inputs.target_markets' ../../.pipeline-state.json)

# Convert comma-separated markets into JSON array
MARKETS_JSON=$(echo "$MARKETS" | jq -R 'split(",") | map(gsub("^\\s+|\\s+$";""))')

jq --arg b "$BRAND" --arg u "$URL" --argjson m "$MARKETS_JSON" \
   '.brand_name = $b | .website_url = $u | .target_markets = $m' \
   config.json > config.tmp && mv config.tmp config.json
```

Delegate to a fresh subagent:

> Execute the `init-pillar-content-analysis` skill end-to-end from this directory (`Pillar-Content-Architecture/Content-Creation-Workflow`). The user inputs (`brand_name`, `website_url`, `target_markets`) are **already populated in config.json** — skip the Phase 0.1 question step entirely and proceed straight to 0.2 (CDP check). Run all phases (0 through 5) without stopping or asking questions. On success, confirm `.analysis-done` exists and reply with exactly `STAGE_B_DONE`. On failure reply `STAGE_B_FAILED <reason>`.

**Gate check:** confirm all four deliverables exist and are non-empty:
- `Step-1-Brand-Discovery/brand-discovery.md`
- `Step-2-Competitor-Discovery/competitor-discovery.md`
- `Step-3-Content-Gap-Analysis/content-gap-analysis.md`
- `Step-3b-Content-Scope-Estimation/scope-estimation.md`

AND `.analysis-done` exists AND `scope.total_keywords_to_research` is set in `config.json`.

Same retry rule: one silent retry, then hard stop.

---

## SECTION 6 — Stage C: Keyword Research

Mark stage C `in_progress`.

Delegate to a fresh subagent:

> Execute the `init-keyword-research` skill end-to-end from this directory (`Pillar-Content-Architecture/Content-Creation-Workflow`). All prerequisites are met (Steps 1–3b deliverables exist, config is populated). Run all phases (-1 through 5) without stopping or asking questions. Loop batches per the iteration rules until `scope.total_keywords_to_research` is met. On success, confirm `.keyword-research-done` exists and reply with exactly `STAGE_C_DONE`. On failure reply `STAGE_C_FAILED <reason>`.

**Gate check:** confirm all of these exist and are non-empty:
- `Step-4-Keyword-Research/master-keywords.csv`
- `Step-4-Keyword-Research/shortlisted-keywords.csv`
- `Step-4-Keyword-Research/summary.md`
- `.keyword-research-done`

Same retry rule.

---

## SECTION 7 — Stage D: Verification & Handoff

After all three stages complete, run a final verification:

```bash
echo "=== PHASE 1 COMPLETE ==="
echo "Brand: $(jq -r '.brand_name' config.json)"
echo "Markets: $(jq -r '.target_markets | join(", ")' config.json)"
echo "Pillars planned: $(jq -r '.scope.pillars' config.json)"
echo "Total keywords researched: $(wc -l < Step-4-Keyword-Research/master-keywords.csv)"
echo "Shortlisted keywords: $(wc -l < Step-4-Keyword-Research/shortlisted-keywords.csv)"
```

Update `.pipeline-state.json`:

```bash
jq '.completed_at = (now | todate) | .stages.architecture.status = "completed" | .stages.analysis.status = "completed" | .stages.keyword_research.status = "completed"' \
   ../../.pipeline-state.json > ../../.pipeline-state.tmp && mv ../../.pipeline-state.tmp ../../.pipeline-state.json
```

Output to the user (one short message):

```
Phase 1 complete.
- Brand: <brand>
- Pillars: <n>
- Master keywords: <n>
- Shortlisted: <n>

Ready for Phase 2 (content writing). Invoke `pillar-content-clusters` per pillar.
```

---

## SECTION 8 — Failure Recovery

If any stage hard-stops:

1. Leave `.pipeline-state.json` in place with `status: "failed"` and the reason.
2. Tell the user **only** the stage name, the failure reason, and the resume command:
   ```
   Stage <X> failed: <reason>.
   Re-invoke `run-content-analysis-pipeline` once the issue is resolved — it will resume from the failed stage.
   ```
3. Do not delete any partial deliverables. Do not rerun other stages.

---

## NOTES

- **Why subagents, not nested skill calls:** skills cannot invoke other skills directly. Each stage runs in its own subagent context, which keeps the orchestrator's context window clean across what may be a multi-hour run.
- **Why gate files instead of TodoList state:** TodoList is in-conversation only; it does not survive a fresh process. Gate files on disk + `.pipeline-state.json` are the durable source of truth.
- **Why fresh inputs are written before Stage B:** the analysis skill normally asks the user for them in Phase 0.1. Pre-populating `config.json` lets the orchestrator suppress that question without modifying the child skill.
- **Phase 2** (content writing using `pillar-content-clusters`) is intentionally out of scope. This skill ends at the keyword shortlist.
