---
name: init-pillar-content-analysis
description: "SEO Domination Engine: Run the full content analysis workflow (Steps 1 to 3b) for a new client. Asks for client name, URL, and target market, updates config.json, checks CDP browser connection, builds a granular TodoList, then executes Steps 1, 2, 3, and 3b autonomously without stopping. Produces 4 deliverable MD files and updates seo-domination-report.html. Must be run from inside Pillar-Content-Architecture/. Triggers: 'init pillar content analysis', 'run pillar content analysis', 'start content analysis', 'run steps 1 to 3b', 'begin seo analysis', 'start seo workflow'."
---

# Init Pillar Content Analysis

## Prerequisites

This skill MUST be run from inside `Pillar-Content-Architecture/`. The `init-pillar-content-architecture` skill must have been run first to create this workspace.

---

## AUTOPILOT RULES — READ FIRST

- **Sequential browser, single CDP.** Only one Chrome instance is connected via CDP. **Never** issue parallel `agent-browser` calls. Never spawn parallel subagents that touch the browser. One operation at a time, await result, then next. This applies to every phase below.
- **Never stop or pause after Phase 0.** Do not ask the user any questions mid-run.
- **Never skip a phase.** Complete every phase and every sub-step in order.
- **Validate before proceeding.** Before moving to the next step, confirm the deliverable file exists and is non-empty.
- **State JSON is truth.** After every micro-step (each numbered sub-section), update `pipeline-state.json` with `stages.analysis.last_step = "<phase>.<substep>"`. The orchestrator reads this on resume. See "State JSON updates" below.
- **Self-heal on failure.** If a tool call fails, retry once silently. If it fails again, consult the `agent-browser` skill for fallback recovery commands. If the recovery still fails and the failure is browser-related → hard stop. Otherwise log inline and continue.
- **agent-browser is the only hard stop.** If the CDP connection fails or agent-browser cannot connect, tell the user: "agent-browser connection failed — cannot continue. Please check the CDP endpoint at https://chrome-cdp.vortexiq.ai and try again." Then stop.
- **Do not summarise each phase.** Just do the work and mark todos complete.
- **Snapshot first, screenshot second.** Always use `agent-browser snapshot` (DOM snapshot) as the primary method to understand page structure. Fall back to a screenshot only if the snapshot is insufficient.

## State JSON updates (after every micro-step)

After completing each numbered sub-step, run:

```bash
jq --arg step "<phase>.<substep>" '.stages.analysis.last_step = $step | .stages.analysis.status = "in_progress" | .updated_at = (now | todate)' \
   pipeline-state.json > /tmp/ps.tmp && mv /tmp/ps.tmp pipeline-state.json
```

Replace `<phase>.<substep>` with e.g. `0.3`, `1.2`, `3.4b`. On Phase 5 completion, set `status = "completed"`.

---

## PHASE 0 — Bootstrap

### 0.1 Collect client info

Ask the user exactly these three questions in one message:
1. Client / brand name?
2. Client website URL?
3. Target market(s)? (e.g. UK, Ireland, USA)

Wait for the response. Do not proceed until all three are provided.

### 0.2 Update config.json

Read `config.json`. Update these fields with the values provided:
- `brand_name` → client name
- `website_url` → client URL
- `target_markets` → array of market strings

Write the updated config.json back to disk.

### 0.3 Check CDP connection

**Step 1 — Curl the CDP HTTP endpoint from config.json**

Read `cdp_http` from `config.json` (e.g. `https://chrome-cdp.vortexiq.ai`). Curl its `/json/version` endpoint:

```bash
curl -s "$(jq -r '.cdp_http' config.json)/json/version"
```

This returns a JSON object. Extract the `webSocketDebuggerUrl` field — it will look like `ws://localhost/devtools/browser/<id>`. Replace `ws://localhost` with `wss://<cdp_host>` (the hostname from `cdp_http`) to get the secure WebSocket URL:

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

**Step 2 — Update config.json with the live WebSocket URL**

Write the derived `WS_URL` back into `config.json` under a `cdp_ws` field:

```bash
jq --arg ws "$WS_URL" '.cdp_ws = $ws' config.json > config.tmp && mv config.tmp config.json
```

This is the value that every subsequent agent-browser command will use. The pattern for all agent-browser calls throughout this skill is:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
agent-browser --cdp "$CDP" <command>
```

Always read `cdp_ws` fresh from `config.json` at the start of each phase — never hardcode the WebSocket URL inline.

**Step 3 — Close all browser tabs except one**

List all open tabs and close all but the first using agent-browser:

```bash
# Count open tabs, then close every tab above index 1 in descending order
TAB_COUNT=$(agent-browser --cdp "$WS_URL" tab | grep -cE '^\s*\[?[0-9]+')
if [ "$TAB_COUNT" -gt 1 ]; then
  for i in $(seq "$TAB_COUNT" -1 2); do
    agent-browser --cdp "$WS_URL" tab close "$i" || true
  done
fi
agent-browser --cdp "$WS_URL" tab 1
```

**Step 4 — Verify connection**

Take a test screenshot to confirm agent-browser is working:

```bash
agent-browser --cdp "$WS_URL" screenshot --output /tmp/cdp-test.png && echo "CDP OK" || echo "CDP FAIL"
```

If result is `CDP FAIL` → tell the user and stop (hard stop, see autopilot rules).
If result is `CDP OK` → clean up: `rm -f /tmp/cdp-test.png` and continue.

### 0.4 Read and understand the workflow

Read all of these files in order:
1. `Master-Matrix.md`
2. `Step-1-Brand-Discovery/Steps.md`
3. `Step-2-Competitor-Discovery/Steps.md`
4. `Step-3-Content-Gap-Analysis/Steps.md`
5. `Step-3b-Content-Scope-Estimation/Steps.md`

Internalize every phase, sub-phase, and task listed in each Steps.md. You will use these as the exact instructions for execution.

### 0.5 Create the TodoList

Using the TodoWrite tool, create a comprehensive todo list covering every single phase and sub-step from all four Steps.md files, plus the HTML update and summary. Be maximally granular — each individual task from each Steps.md becomes its own todo item. Structure as:

- PHASE 0: Bootstrap (mark completed items as done)
- PHASE 1: Brand Discovery — one todo per sub-step from Step-1 Steps.md
- PHASE 2: Competitor Discovery — one todo per sub-step from Step-2 Steps.md
- PHASE 3: Content Gap Analysis — one todo per sub-step from Step-3 Steps.md
- PHASE 3b: Content Scope Estimation — one todo per sub-step from Step-3b Steps.md
- PHASE 4: HTML Update
- PHASE 5: Summary

---

## PHASE 1 — Brand Discovery

Follow `Step-1-Brand-Discovery/Steps.md` exactly. Complete every phase and sub-step in that file.

For all browser-based research tasks, use agent-browser with the WebSocket URL from config.json:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
agent-browser --cdp "$CDP" open <url>
agent-browser --cdp "$CDP" screenshot --output Step-1-Brand-Discovery/screenshots/<name>.png
```

Save all screenshots to `Step-1-Brand-Discovery/screenshots/`.

Write all findings to `Step-1-Brand-Discovery/brand-discovery.md`.

**Validation gate:** Confirm `Step-1-Brand-Discovery/brand-discovery.md` exists and is non-empty before proceeding to Phase 2.

---

## PHASE 2 — Competitor Discovery

Follow `Step-2-Competitor-Discovery/Steps.md` exactly. Complete every phase and sub-step in that file.

For all browser-based research, use:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
agent-browser --cdp "$CDP" open <url>
agent-browser --cdp "$CDP" screenshot --output Step-2-Competitor-Discovery/screenshots/<name>.png
```

Write all findings to `Step-2-Competitor-Discovery/competitor-discovery.md`.

**Validation gate:** Confirm `Step-2-Competitor-Discovery/competitor-discovery.md` exists and is non-empty before proceeding to Phase 3.

---

## PHASE 3 — Content Gap Analysis

Follow `Step-3-Content-Gap-Analysis/Steps.md` exactly. Complete every phase and sub-step in that file.

Use findings from brand-discovery.md and competitor-discovery.md as inputs. For any live SERP or site research:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
agent-browser --cdp "$CDP" open <url>
agent-browser --cdp "$CDP" screenshot --output Step-3-Content-Gap-Analysis/screenshots/<name>.png
```

Write all findings to `Step-3-Content-Gap-Analysis/content-gap-analysis.md`.

**Validation gate:** Confirm `Step-3-Content-Gap-Analysis/content-gap-analysis.md` exists and is non-empty before proceeding to Phase 3b.

---

## PHASE 3b — Content Scope Estimation

Follow `Step-3b-Content-Scope-Estimation/Steps.md` exactly. Complete every phase and sub-step in that file.

Use findings from all three previous deliverables as inputs. For any browser research:

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
agent-browser --cdp "$CDP" open <url>
agent-browser --cdp "$CDP" screenshot --output Step-3b-Content-Scope-Estimation/screenshots/<name>.png
```

Update the `scope` object in `config.json` with the estimated values (pillars, avg_spokes_per_pillar, total_spokes, total_articles, avg_keywords_per_piece, total_keywords_to_research, kp_batches, tier, rationale).

Write the deliverable to `Step-3b-Content-Scope-Estimation/scope-estimation.md`.

**Validation gate:** Confirm `Step-3b-Content-Scope-Estimation/scope-estimation.md` exists and is non-empty before proceeding to Phase 4.

---

## PHASE 4 — Update seo-domination-report.html

Read `config.json` to get all current values.

Read `seo-domination-report.html`.

Replace every `{{PLACEHOLDER}}` variable in the HTML with the corresponding value from config.json, using the `template_vars` mapping in config.json as the reference. Do a full replacement pass — every variable, not just the ones changed in this run.

Write the updated HTML back to `seo-domination-report.html`.

---

## PHASE 4b — Completion Marker

After the HTML update succeeds, write a marker AND update state JSON:

```bash
touch .analysis-done
jq '.stages.analysis.status = "completed" | .stages.analysis.last_step = "4b" | .updated_at = (now | todate)' \
   pipeline-state.json > /tmp/ps.tmp && mv /tmp/ps.tmp pipeline-state.json
```

---

## PHASE 5 — Summary

Write a 5–6 sentence summary covering:
- What brand was analysed and what markets were targeted
- Key brand positioning findings from Step 1
- Top competitors identified and their main strengths/gaps from Step 2
- Most significant content opportunities found in Step 3
- Estimated content scope from Step 3b (pillars, spokes, keywords, tier)
- Overall readiness for keyword research (Step 4)

Output this summary to the user. This is the only output after Phase 0.