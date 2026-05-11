---
name: UI-UX-Audit
description: "Perform a UI/UX audit on any URL by walking through a user-supplied flow at a specified screen resolution. Drives agent-browser sequentially (no CDP), captures an above-the-fold snapshot + screenshot at every step, evaluates each screen against Nielsen's 10 heuristics + mobile-first + visual hierarchy + accessibility quick-checks, and produces a self-contained HTML report (with literal screenshot paths that always resolve) plus a JSON deliverable of recommended actions. Triggers: 'ui ux audit', 'ui audit', 'ux audit', 'audit checkout flow', 'audit mobile site', 'audit homepage', 'usability audit', 'heuristic audit', 'perform audit on', 'audit <url>', 'audit <site> mobile', 'audit <site> desktop'."
---

# UI/UX Audit Skill

Run a structured, screenshot-backed UI/UX audit of any URL. The user supplies:

1. **A target URL** (e.g. `https://www.richersounds.com`).
2. **An audit prompt** — the flow to walk through in plain English (e.g. "audit the checkout flow", "audit homepage → search → product → add to cart").
3. **A screen resolution / device profile** — `mobile`, `tablet`, `desktop`, or explicit `WxH` (e.g. `390x844`).

Output is a self-contained audit folder with screenshots, a Markdown report, and a JSON deliverable of recommended actions.

---

## ABSOLUTE RULES

1. **No CDP.** This skill uses the `agent-browser` CLI in its **standalone mode** — it launches and manages its own persistent browser. **Never** pass `--cdp` from inside this skill, and do not read or write `cdp_ws` / `cdp_http` from any config.
2. **Sequential only.** One `agent-browser` call at a time. Never spawn parallel subagents that share the browser session.
3. **Snapshot before every action.** Use `agent-browser snapshot -i` to get refs (`@e1`, `@e7`…) before any click/type. Never click on guessed selectors.
4. **Screenshot every step.** Every step (navigate, click, type, scroll, wait) ends with an **above-the-fold** screenshot (visible viewport only, no `--full`) saved into `screenshots/`. No exceptions.
5. **Device / viewport emulation is MANDATORY and VERIFIED.** When the user specifies a device or resolution (mobile, tablet, desktop, `WxH`, or a named device like "iPhone 14"), the emulation MUST be applied **before the first navigation** AND verified by reading back `window.innerWidth / innerHeight / devicePixelRatio` and `navigator.userAgent` via `agent-browser eval`. If the read-back values do not match the requested profile within 1px tolerance for width, **hard stop** — do not continue with a wrong-size browser. The folder slug, the `audit-state.json`, and the report MUST all reflect the *verified* device profile, never the *requested* one if they diverge.
6. **Always wait for networkidle BEFORE every screenshot.** No exceptions. If `wait --load networkidle` times out, log it as a step finding ("page never reached networkidle within 30s") **then** retry once after a 2s pause — and only then take the screenshot. Never substitute a flat `wait 1500` for the pre-screenshot wait.
7. **Always above-the-fold screenshots.** Every screenshot is the visible viewport only — **never** pass `--full`. The audit captures what a real user sees on first paint at the requested device size. If a step needs to inspect content below the fold, scroll first, wait `networkidle`, then take another above-the-fold screenshot named `step-<N>-scroll-<n>.png`. Do not stitch full-page screenshots.
8. **Set viewport BEFORE the first navigation.** The whole point of resolution-based auditing is broken if the first paint happens at the wrong size.
9. **Do not change the page beyond what the audit prompt asks.** No checkout completions, no form submissions to live endpoints, no signup with real emails. Stop one step before any irreversible action and document it in the report.
10. **Hard stops:** (a) `agent-browser doctor` fails, (b) device/viewport read-back does not match the requested profile, (c) two consecutive commands fail without a clear cause. Every other failure is logged inline and the audit continues.
11. **Background execution is mandatory.** The skill is invoked from a chat where the user is waiting. Browser audits take minutes — never block the main agent. The flow is: the main agent does STEP 0–STEP 3 (planning + folder + state + step list), then **spawns ONE background subagent** that owns STEP 1's doctor check, STEP 2's emulation, STEP 4's step loop, STEP 5/5b's report, and STEP 6's finalisation **sequentially**, then returns control to the user immediately with the canned acknowledgement (see STEP 6.a). Progress lives on disk so the user can monitor it without the main agent staying alive.
12. **One executor subagent — never parallel.** The background subagent runs the entire step loop **sequentially in a single process**. Do not split steps across multiple subagents (the browser session is single-threaded and not safely shareable). Within the executor, per-step heuristic *evaluation* (text-only, no browser) may run inline; do not fork it to additional subagents.

---

## INPUTS

The user invocation typically looks like:

> "Perform UI/UX audit on richersounds.com mobile site. Audit the checkout flow."

Parse it into three fields:

| Field | How to derive |
|---|---|
| `target_url` | The first URL or domain mentioned. If only a bare domain, prepend `https://`. |
| `device_profile` | Look for `mobile`, `tablet`, `desktop`, an explicit `WxH`, or a named device ("iPhone 14", "iPhone 15 Pro", "Pixel 7", "iPad Pro", "Galaxy S24"). Default = `mobile`. **Phrases like "mobile site", "on mobile", "mobile view", "from mobile" all map to `mobile`.** |
| `flow_prompt` | The remainder — the action verbs and pages to audit. |

If any field is genuinely ambiguous (e.g. no URL at all), ask **one** clarifying question and wait for an answer. Otherwise proceed without questions.

### Device profiles

| Profile | Width | Height | DPR | UA / device name |
|---|---|---|---|---|
| `mobile` | 390 | 844 | 3 | `iPhone 14` |
| `tablet` | 820 | 1180 | 2 | `iPad (gen 7)` |
| `desktop` | 1440 | 900 | 1 | macOS Chrome UA (no named device) |
| `WxH` | parsed | parsed | 1 | desktop Chrome UA |
| named device (e.g. `iPhone 14`) | per device | per device | per device | passed verbatim to `agent-browser set device` |

**Use `set device` whenever a named device is given or implied** (mobile/tablet defaults map to named devices). Use `set viewport` only when a bare `WxH` is given or for the desktop default.

---

## STEP 0 — Workspace setup

**All audit folders MUST live under `/home/app/audit/`.** Never write to the current working directory, never to `./audits/`, never to `$HOME/audits/`. The root is fixed so every audit run on this host is discoverable in one place and so downstream tooling (viewers, archival, the post-processor) can rely on a single canonical location.

```bash
AUDIT_ROOT="/home/app/audit"
mkdir -p "$AUDIT_ROOT"

SLUG=$(echo "$TARGET_URL" | sed -E 's|https?://||; s|/.*||; s/[^a-zA-Z0-9]/-/g' | tr '[:upper:]' '[:lower:]')
TS=$(date +%Y%m%d-%H%M%S)
AUDIT_DIR="${AUDIT_ROOT}/${SLUG}-${DEVICE_PROFILE}-${TS}"
mkdir -p "$AUDIT_DIR/screenshots"
```

If `/home/app/audit/` is not writable (permission error, missing parent, read-only filesystem) → **hard stop** with: `Cannot create audit folder under /home/app/audit/ — check permissions on this path before re-running.` Do not silently fall back to `~/audits` or the cwd; that's how prior runs ended up scattered across the filesystem.

Create `$AUDIT_DIR/audit-state.json` with:

```json
{
  "target_url": "<url>",
  "device_profile": "<profile>",
  "viewport": {"width": 0, "height": 0, "dpr": 1},
  "flow_prompt": "<verbatim>",
  "status": "planned",
  "started_at": "<ISO>",
  "updated_at": "<ISO>",
  "completed_at": null,
  "current_step": 0,
  "steps": []
}
```

The `status` field is the single source of truth for the user's "is it done yet?" question. Allowed values:

- `planned` — STEP 0–STEP 3 finished, executor not yet started.
- `running` — executor subagent is alive and processing steps.
- `completed` — audit finished, `REPORT.html` written.
- `failed` — hard stop hit; `last_error` field set with the reason.

Also seed `$AUDIT_DIR/progress.log` (append-only human-readable log the executor writes to):

```bash
echo "[$(date -u +%FT%TZ)] planned: $TARGET_URL ($DEVICE_PROFILE) — $FLOW_PROMPT" > "$AUDIT_DIR/progress.log"
```

Update `audit-state.json` after every step (atomic write via temp file). Append one line to `progress.log` before and after each step so the user tailing the file sees real-time motion.

---

## STEP 1 — Verify agent-browser is healthy

No CDP, no `--cdp` flag. Just confirm the binary works and the persistent browser session is reachable:

```bash
agent-browser --version
agent-browser doctor || { echo "agent-browser doctor failed — cannot continue."; exit 1; }
```

If `doctor` fails:
1. Try once more after `agent-browser close` (resets the persistent session).
2. If still failing → hard stop with the message: `agent-browser unavailable — cannot continue. Run 'agent-browser doctor' manually to diagnose.`

Close any leftover tabs from a previous run so the audit starts clean:

```bash
TAB_COUNT=$(agent-browser tab | grep -cE '^\s*\[?[0-9]+')
if [ "$TAB_COUNT" -gt 1 ]; then
  for i in $(seq "$TAB_COUNT" -1 2); do
    agent-browser tab close "$i" || true
  done
fi
agent-browser tab 1 2>/dev/null || true
```

---

## STEP 2 — Apply device emulation (BEFORE first navigation)

`agent-browser` exposes the following browser-settings commands — these are the only correct way to configure the session:

```
agent-browser set viewport <w> <h> [scale]   # Set viewport size; scale = DPR (e.g. 2 or 3 for retina)
agent-browser set device <name>              # Emulate a named device — sets viewport, DPR, UA, touch
agent-browser set geo <lat> <lng>            # Geolocation
agent-browser set offline [on|off]           # Offline mode
agent-browser set headers <json>             # Extra HTTP headers
agent-browser set credentials <u> <p>        # HTTP basic auth
agent-browser set media [dark|light]         # Color scheme
```

**Prefer `set device` over `set viewport` whenever a named device fits** — it's the only path that also sets the mobile UA, DPR, and touch capability together. Use `set viewport` only for explicit `WxH` requests on desktop.

### 2.1 Resolve the profile

```bash
case "$DEVICE_PROFILE" in
  mobile)  EMULATION_KIND=device; DEVICE_NAME="iPhone 14";   EXPECT_W=390;  EXPECT_H=844;  EXPECT_DPR=3 ;;
  tablet)  EMULATION_KIND=device; DEVICE_NAME="iPad (gen 7)"; EXPECT_W=810; EXPECT_H=1080; EXPECT_DPR=2 ;;
  desktop) EMULATION_KIND=viewport; EXPECT_W=1440; EXPECT_H=900; EXPECT_DPR=1 ;;
  *x*)     EMULATION_KIND=viewport; EXPECT_W=$(echo "$DEVICE_PROFILE" | cut -dx -f1); EXPECT_H=$(echo "$DEVICE_PROFILE" | cut -dx -f2); EXPECT_DPR=1 ;;
  *)       EMULATION_KIND=device; DEVICE_NAME="$DEVICE_PROFILE"; EXPECT_W=0; EXPECT_H=0; EXPECT_DPR=0 ;;  # named device — verify via read-back
esac
```

### 2.2 Apply

```bash
if [ "$EMULATION_KIND" = "device" ]; then
  agent-browser set device "$DEVICE_NAME" || { echo "FAIL: set device '$DEVICE_NAME' rejected"; exit 1; }
else
  agent-browser set viewport "$EXPECT_W" "$EXPECT_H" "$EXPECT_DPR" || { echo "FAIL: set viewport ${EXPECT_W} ${EXPECT_H} ${EXPECT_DPR} rejected"; exit 1; }
fi
```

### 2.3 Verify (HARD STOP if the read-back disagrees)

```bash
READBACK=$(agent-browser eval "JSON.stringify({w:innerWidth,h:innerHeight,dpr:devicePixelRatio,ua:navigator.userAgent,touch:('ontouchstart' in window)})")
echo "$READBACK" > "$AUDIT_DIR/.viewport-readback.json"
ACT_W=$(echo "$READBACK" | jq -r '.w'); ACT_H=$(echo "$READBACK" | jq -r '.h'); ACT_DPR=$(echo "$READBACK" | jq -r '.dpr'); ACT_UA=$(echo "$READBACK" | jq -r '.ua')

# For named devices we don't know exact W/H ahead of time — accept whatever the read-back says,
# but assert mobile/tablet UA must contain "Mobile" or "iPad" / "Android" tokens.
if [ "$EMULATION_KIND" = "viewport" ]; then
  DIFF=$(( ACT_W > EXPECT_W ? ACT_W - EXPECT_W : EXPECT_W - ACT_W ))
  if [ "$DIFF" -gt 1 ]; then
    echo "FAIL: viewport read-back ${ACT_W}x${ACT_H} does not match requested ${EXPECT_W}x${EXPECT_H}"; exit 1
  fi
fi

case "$DEVICE_PROFILE" in
  mobile|tablet)
    echo "$ACT_UA" | grep -qiE 'Mobile|iPad|Android' || { echo "FAIL: requested $DEVICE_PROFILE but UA is desktop: $ACT_UA"; exit 1; }
  ;;
esac

VIEWPORT_APPLIED=true
```

If any of these checks fail → **hard stop**. Do not let the audit continue against a desktop browser when the user asked for mobile. The previous failure mode (folder named `*-desktop-*` while the user asked for mobile, screenshots clearly desktop) is exactly what this gate prevents.

### 2.4 Persist the verified profile

```bash
jq --argjson w "$ACT_W" --argjson h "$ACT_H" --argjson dpr "$ACT_DPR" --arg ua "$ACT_UA" --arg dev "${DEVICE_NAME:-}" \
   '.viewport = {width:$w, height:$h, dpr:$dpr, ua:$ua, device_name:$dev} | .updated_at = (now | todate)' \
   "$AUDIT_DIR/audit-state.json" > /tmp/as.tmp && mv /tmp/as.tmp "$AUDIT_DIR/audit-state.json"
```

The folder `SLUG` was generated in STEP 0 from the *requested* profile. After verification, if the requested profile was a named device or `WxH`, leave the folder name as-is (it's already accurate). The hard-stop in 2.3 guarantees the folder name and the actual emulation never silently disagree.

Persist the resolved viewport to `audit-state.json`:

```bash
jq --argjson w $W --argjson h $H --argjson dpr $DPR \
   '.viewport = {width:$w, height:$h, dpr:$dpr} | .updated_at = (now | todate)' \
   "$AUDIT_DIR/audit-state.json" > /tmp/as.tmp && mv /tmp/as.tmp "$AUDIT_DIR/audit-state.json"
```

> Note: if `agent-browser viewport` is not exposed by the installed version (`agent-browser --version` < required), surface a one-line warning into the report's environment block but **do not abort**. The audit's heuristic checks still run on whatever the live viewport is.

---

## STEP 3 — Decompose the flow prompt into ordered steps

Read `flow_prompt`. Break it into a numbered, atomic `steps[]` plan **before** touching the browser. Each step is one of:

| Kind | Required fields |
|---|---|
| `navigate` | `url` |
| `click` | `target` (semantic description, resolved to a ref via snapshot) |
| `type` | `target`, `text` (NEVER real PII; use placeholder values like `Test User`, `test+audit@example.com`, `4111 1111 1111 1111`) |
| `scroll` | `direction`, `amount` |
| `wait` | `selector` or `ms` |
| `assert` | `condition` (e.g. "cart count = 1") |

Common flow expansions:

- **"audit checkout flow"** → home → category → product → add to cart → view cart → start checkout → guest details (stop **before** payment submission) → screenshot payment screen → end.
- **"audit homepage"** → home → scroll to fold 1 / fold 2 / fold 3 / footer → end.
- **"audit search"** → home → focus search → type query → results → first result → end.

Write the planned `steps[]` into `audit-state.json` before executing any of them. This is the resume contract: if the run dies mid-flow, the next invocation can pick up at `current_step`.

**This is where the main agent's responsibility ends.** STEP 1 (doctor), STEP 2 (emulation), STEP 4 (step loop), STEP 5/5b (report + inlining) and STEP 6 (finalisation) are all owned by the background executor spawned in STEP 3.5.

---

## STEP 3.5 — Hand off to background executor

Once `audit-state.json` has the full `steps[]` plan, dispatch a single background subagent that owns the rest of the run. The main agent does NOT execute STEP 4 inline — it hands off and returns control to the user.

Use the `Agent` tool with `run_in_background=true` and `subagent_type=general-purpose`. The dispatch prompt MUST contain:

- The absolute `$AUDIT_DIR` path.
- The verbatim `$TARGET_URL`, `$DEVICE_PROFILE`, `$FLOW_PROMPT`.
- An instruction to read `audit-state.json` for the planned steps.
- A note that the executor is the *sole* owner of the browser session for this audit.
- The exact set of files it must produce (`screenshots/step-*.png`, `REPORT.linked.html`, `REPORT.html`, `recommendations.json`, `.audit-done`).
- Status-update contract: set `status:"running"` first, append `progress.log` lines per step, set `status:"completed"` and write `.audit-done` on success, or `status:"failed"` + `last_error` on hard stop.

Dispatch template (Claude assembles this and calls `Agent` once):

```
description: UI/UX audit executor — sequential
subagent_type: general-purpose
run_in_background: true

prompt:
You are the background executor for a UI/UX audit. Run sequentially — never parallel.

Audit dir: <AUDIT_DIR>
Target URL: <TARGET_URL>
Device profile: <DEVICE_PROFILE>
Flow prompt: <FLOW_PROMPT>

1. Read <AUDIT_DIR>/audit-state.json. The planned `steps[]` array is your work queue.
2. Set status="running", updated_at=now, append "[ts] executor started" to progress.log.
3. Execute STEP 1 of UI-UX-Audit.md (agent-browser doctor + tab cleanup).
4. Execute STEP 2 (apply device emulation via `agent-browser set device` / `set viewport`, verify via eval read-back, hard stop on mismatch).
5. For each step in `steps[]` (in order):
   - Append "[ts] step <N> start: <description>" to progress.log.
   - Follow STEP 4.1 → 4.7 of UI-UX-Audit.md exactly. Always wait `networkidle` before screenshot. Always above-the-fold (no `--full`). Update `current_step` + `updated_at` after each step.
   - Append "[ts] step <N> done: <findings_count> findings" to progress.log.
6. Execute STEP 5 (write REPORT.linked.html) and STEP 5b (write inline_images.py and run it to produce REPORT.html).
7. Execute STEP 6 (set status="completed", completed_at, touch .audit-done, append "[ts] completed" to progress.log).
8. On any hard stop: set status="failed", last_error=<reason>, append "[ts] FAILED: <reason>" to progress.log, then exit. Do not delete any partial deliverable.

Read UI-UX-Audit.md before starting if you need the full command syntax. Do not invent shortcuts. Do not call any tool that would touch a second browser session.
```

After the `Agent` call returns its agent id, the main agent **immediately** proceeds to STEP 6.a (the canned user acknowledgement). Do **not** wait for the executor. Do **not** poll. Do **not** sleep. The user will check `$AUDIT_DIR` on their own time.

---

## STEP 4 — Execute steps (loop)

> **Owned by the background executor spawned in STEP 3.5, not by the main agent.** The instructions below are what the executor follows. The main agent must NOT run these inline.

For each step in order:

### 4.1 Pre-step snapshot

```bash
agent-browser snapshot -i -c > "$AUDIT_DIR/screenshots/step-${N}-pre.snapshot.txt"
```

### 4.2 Perform the action

Use the snapshot to resolve refs. Prefer **semantic finders** over raw selectors:

```bash
# Examples
agent-browser open "$URL"
agent-browser find role button "Add to basket" click
agent-browser find label "Email" type "test+audit@example.com"
agent-browser scroll down 800
agent-browser wait --load networkidle
```

### 4.3 Post-step wait — MANDATORY networkidle before screenshot

Always wait for `networkidle` before capturing. This is non-negotiable — short-circuiting it is the cause of "screenshot showed a half-loaded page" reports.

```bash
if ! agent-browser wait --load networkidle; then
  echo "WARN: networkidle timeout on step $N; pausing 2s and retrying once"
  agent-browser wait 2000
  agent-browser wait --load networkidle || NETWORKIDLE_TIMEOUT=true
  # Add a finding for this step:
  STEP_NETWORKIDLE_NOTE="page never reached networkidle within ~30s (retried once)"
fi
```

If `NETWORKIDLE_TIMEOUT=true`, take the screenshot anyway, but emit a `major` finding under the **Performance signals** heuristic group: "Page does not reach network-idle within 30s — long-tail third-party requests blocking quiescence."

### 4.4 Post-step capture

Always capture both. `--full` is mandatory — never take a viewport-only screenshot in place of a step screenshot.

```bash
# Re-verify viewport hasn't been silently reset by a navigation (some sites force a desktop layout via UA spoofing).
WB=$(agent-browser eval "JSON.stringify({w:innerWidth,ua:navigator.userAgent})")
WB_W=$(echo "$WB" | jq -r '.w')
case "$DEVICE_PROFILE" in
  mobile|tablet) [ "$WB_W" -gt 900 ] && echo "WARN: step $N viewport drifted to ${WB_W}px — re-applying emulation" && \
                 ([ "$EMULATION_KIND" = "device" ] && agent-browser set device "$DEVICE_NAME" || agent-browser set viewport "$EXPECT_W" "$EXPECT_H" "$EXPECT_DPR") && \
                 agent-browser wait --load networkidle ;;
esac

agent-browser screenshot --output "$AUDIT_DIR/screenshots/step-${N}.png"   # above-the-fold viewport only — DO NOT pass --full
agent-browser snapshot -i -c > "$AUDIT_DIR/screenshots/step-${N}.snapshot.txt"
URL_NOW=$(agent-browser get url)
TITLE_NOW=$(agent-browser get title)
```

### 4.5 Heuristic evaluation (per step)

Read the post-step snapshot and screenshot. Evaluate the screen against **all** of the heuristic groups below. For each finding, write a record into the step's `findings[]`. If a heuristic has no issue on this screen, omit it (do not pad the report).

#### Heuristic groups

1. **Nielsen's 10 usability heuristics** — Visibility of system status; Match between system and real world; User control & freedom; Consistency & standards; Error prevention; Recognition over recall; Flexibility & efficiency; Aesthetic & minimalist design; Help users recognise/recover errors; Help & documentation.
2. **Mobile-first checks** (only if `device_profile` is `mobile` or `tablet`) — Tap-target ≥ 44×44 px; thumb reach; sticky header/footer behaviour; viewport meta presence; horizontal scroll absent; font-size ≥ 16px on inputs (avoid iOS zoom); modals/dialogs respect viewport.
3. **Visual hierarchy** — Primary CTA prominence and contrast; F/Z scan path; whitespace; type scale consistency; image-to-copy ratio; above-the-fold value clarity.
4. **Accessibility quick-checks** — Visible focus ring on the focused element; alt text on images (read from snapshot); colour-contrast obvious failures (low-confidence visual estimate); landmarks (`<main>`, `<nav>`, `<header>`, `<footer>`); form labels associated.
5. **Performance signals (visual only, no Lighthouse)** — Layout shift on load (compare pre vs post snapshot if applicable); blocking modals/popups before content; lazy-load failures (broken image placeholders).
6. **Trust & conversion** (e-commerce flows) — Price clarity; stock/availability; delivery info; returns/security signals; cart visibility; guest checkout option; field count in forms.

#### Finding record schema

Each finding is one object in `findings[]`:

```json
{
  "id": "F-<step>-<n>",
  "heuristic_group": "Nielsen | Mobile | Hierarchy | A11y | Performance | Trust",
  "heuristic": "<specific name, e.g. 'Visibility of system status'>",
  "severity": "critical | major | minor | nit",
  "evidence": "<one sentence pointing at what's visible in the screenshot or snapshot>",
  "screenshot": "screenshots/step-<N>.png",
  "screenshot_region": {"x": 0, "y": 0, "w": 0, "h": 0} ,
  "recommendation": "<concrete, single-action fix>",
  "effort": "low | medium | high",
  "impact": "low | medium | high"
}
```

`screenshot_region` is optional; include it when a specific element is the subject of the finding (estimate from the snapshot's bounding box if available via `agent-browser get box <ref>`).

#### Severity rubric

- **critical** — blocks the flow or causes data loss / wrong charge / accessibility lockout.
- **major** — causes drop-off, confusion, or rework for a meaningful share of users.
- **minor** — friction or polish issue; not a blocker.
- **nit** — taste-level inconsistency.

### 4.6 Persist step

Append the step record into `audit-state.json`:

```json
{
  "n": 3,
  "kind": "click",
  "description": "Add to basket from PDP",
  "url_before": "...",
  "url_after": "...",
  "title": "...",
  "screenshot": "screenshots/step-3.png",
  "findings": [ ... ],
  "started_at": "...",
  "completed_at": "..."
}
```

Then update `current_step`.

### 4.7 Failure handling inside a step

agent-browser non-zero exit → apply this short recovery tree (no CDP variants):

1. **Click / type / fill failed (selector or ref)**
   - Re-run `snapshot -i -c` (refs change when the DOM updates).
   - Retry with a **semantic finder** instead of the captured ref: `find role button "<text>" click`, `find label "<text>" type "..."`.
   - If element not in viewport, `scrollintoview "<sel>"` then retry the action.
   - If still failing, check `is visible "<sel>"` / `is enabled "<sel>"`.
   - Last resort: `eval "document.querySelector('<sel>').click()"`.
2. **Page didn't load / blank snapshot**
   - `wait --load networkidle` (timeout ~30s).
   - `get url` — confirm navigation actually happened. If wrong URL, re-`open`.
   - Read `errors` and `console` for browser-side log lines.
3. **Tab confusion**
   - `tab` to list, then close everything above tab 1 in descending order, then `tab 1`.
4. **Dialog blocking the page**
   - `dialog status` → `dialog accept` (or `dismiss`) → re-snapshot.
5. **Stale element after DOM rebuild**
   - Re-snapshot and use a `find role/text` semantic finder rather than the previous `@eN` ref.

If the element is still not findable after recovery → log a `critical` finding ("flow blocker — could not locate `<target>`"), capture the screenshot anyway, **continue** to the next step **only if** the next step does not depend on this one. Otherwise abort the remaining steps and proceed to STEP 5 with what you have.

If two consecutive `agent-browser` calls fail without a clear cause and `agent-browser doctor` also fails, hard stop with: `agent-browser unavailable — cannot continue.`

---

## STEP 5 — Generate the report

**The human-readable deliverable is HTML, not Markdown.** Markdown rendering pipelines (cloud bucket viewers, GitHub previews, MD-to-PDF tools) routinely escape, rewrite, or strip relative image paths — that's exactly why the previous `step-7.png` came out broken. HTML embeds the screenshots with literal `<img src="...">` paths the browser resolves directly, and renders identically anywhere.

Write `$AUDIT_DIR/REPORT.linked.html` (this is the intermediate; STEP 5b produces the final `REPORT.html`). Image paths MUST be the **exact relative path** to the on-disk file (`screenshots/step-N.png`), since the screenshots live at `$AUDIT_DIR/screenshots/`. Do not URL-encode the path. Do not prefix with `./`. Do not inline as base64 — the post-processor in STEP 5b does that.

Use this exact structure (self-contained — no external CSS/JS, no CDN):

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>UI/UX Audit — <SITE> (<DEVICE_PROFILE>)</title>
<style>
  :root { --critical:#c0392b; --major:#e67e22; --minor:#f1c40f; --nit:#7f8c8d; --bg:#fafafa; --card:#fff; --border:#e1e4e8; --text:#24292e; --muted:#586069; }
  *{box-sizing:border-box}
  body{font:14px/1.5 -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif;color:var(--text);background:var(--bg);margin:0;padding:24px;max-width:1100px;margin:0 auto}
  h1{font-size:24px;margin:0 0 4px}
  h2{font-size:18px;margin:32px 0 12px;border-bottom:1px solid var(--border);padding-bottom:6px}
  h3{font-size:16px;margin:20px 0 8px}
  .meta{color:var(--muted);font-size:13px;margin-bottom:20px}
  .card{background:var(--card);border:1px solid var(--border);border-radius:6px;padding:16px;margin:12px 0}
  table{width:100%;border-collapse:collapse;font-size:13px}
  th,td{text-align:left;padding:6px 8px;border-bottom:1px solid var(--border)}
  th{background:#f6f8fa;font-weight:600}
  .sev{display:inline-block;padding:2px 8px;border-radius:3px;color:#fff;font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.4px}
  .sev.critical{background:var(--critical)}.sev.major{background:var(--major)}.sev.minor{background:var(--minor);color:#222}.sev.nit{background:var(--nit)}
  .step{margin:24px 0;padding-bottom:24px;border-bottom:1px solid var(--border)}
  .step img{display:block;max-width:100%;height:auto;border:1px solid var(--border);border-radius:4px;margin:8px 0}
  .finding{padding:10px 12px;border-left:3px solid var(--border);background:#f6f8fa;margin:8px 0;border-radius:0 4px 4px 0}
  .finding.critical{border-left-color:var(--critical)}.finding.major{border-left-color:var(--major)}.finding.minor{border-left-color:var(--minor)}.finding.nit{border-left-color:var(--nit)}
  .finding .id{font-family:ui-monospace,monospace;color:var(--muted);font-size:12px}
  .kvp{color:var(--muted);font-size:12px}
  code{font-family:ui-monospace,SFMono-Regular,Menlo,monospace;background:#f6f8fa;padding:1px 4px;border-radius:3px;font-size:12px}
</style>
</head>
<body>

<h1>UI/UX Audit — <SITE> (<DEVICE_PROFILE>)</h1>
<div class="meta">
  <strong>Target URL:</strong> <a href="<TARGET_URL>"><TARGET_URL></a> &middot;
  <strong>Device:</strong> <DEVICE_PROFILE> — <W>×<H> @ <DPR>x &middot;
  <strong>Flow:</strong> <FLOW_PROMPT> &middot;
  <strong>Started:</strong> <ISO_START> &middot;
  <strong>Completed:</strong> <ISO_END> &middot;
  <strong>Steps:</strong> <N_TOTAL> (<N_OK> ok, <N_PARTIAL> partial, <N_FAIL> failed)
</div>

<h2>Executive summary</h2>
<div class="card">
  <p><!-- 3–5 sentences: flow audited, biggest 2–3 themes, headline recommendation --></p>
</div>

<h3>Top 5 priority recommendations</h3>
<ol>
  <li><strong><!-- title --></strong> — <!-- one-line rationale -->. <span class="kvp">(Severity: <span class="sev critical">critical</span>, Effort: low, Impact: high)</span></li>
  <!-- repeat -->
</ol>

<h2>Flow summary</h2>
<table>
  <thead><tr><th>#</th><th>Step</th><th>URL</th><th>Findings</th><th>Worst severity</th></tr></thead>
  <tbody>
    <tr><td>1</td><td>Open homepage</td><td><code>/</code></td><td>4</td><td><span class="sev major">major</span></td></tr>
    <!-- repeat per step -->
  </tbody>
</table>

<h2>Step-by-step findings</h2>

<!-- One <section class="step"> block per step, in order: -->
<section class="step" id="step-1">
  <h3>Step 1 — <DESCRIPTION></h3>
  <div class="kvp"><strong>URL:</strong> <code><URL_AFTER></code></div>
  <img src="screenshots/step-1.png" alt="Step 1 screenshot">

  <div class="finding major">
    <div><span class="id">F-1-1</span> &middot; <strong>Visibility of system status</strong> &middot; <span class="sev major">major</span></div>
    <div><strong>Evidence:</strong> <!-- one sentence --></div>
    <div><strong>Recommendation:</strong> <!-- concrete fix --></div>
    <div class="kvp">Effort: low &middot; Impact: high</div>
  </div>
  <!-- repeat per finding -->
</section>
<!-- repeat per step -->

<h2>Cross-cutting themes</h2>
<div class="card">
  <!-- 2–4 paragraphs grouping findings that recurred across multiple steps -->
</div>

<h2>Environment &amp; limitations</h2>
<ul>
  <li><strong>agent-browser version:</strong> <VERSION></li>
  <li><strong>Browser session:</strong> standalone agent-browser (no CDP)</li>
  <li><strong>Viewport applied:</strong> <true|false — reason></li>
  <li><strong>Steps that could not be executed:</strong> <list with reasons></li>
  <li><strong>Out-of-scope by design:</strong> payment submission, account creation with real PII, anything destructive on the live site.</li>
</ul>

</body>
</html>
```

### Image-path rules (non-negotiable)

The HTML is generated by Claude with **relative `src` paths**, then a Python post-processor (STEP 5b) rewrites every `<img src>` into an inline `data:image/png;base64,…` URI so the report is a **single self-contained file** that works in any viewer (bucket previewers, email attachments, zipped archives) without sibling files.

While generating the HTML in this step:

- Image paths in `<img src>` are **literal relative paths** matching the on-disk layout: `screenshots/step-<N>.png`, `screenshots/step-<N>-scroll-<n>.png`.
- Do not URL-encode (`%20`, `%2F`).
- Do not use `file://` absolute paths.
- **Do NOT inline base64 yourself** — it would balloon Claude's context. STEP 5b handles inlining.
- Every image referenced in HTML MUST exist on disk. Before writing the HTML, list `$AUDIT_DIR/screenshots/` and only emit `<img>` tags for files that exist; for steps where the screenshot capture failed, render a placeholder card instead of a broken `<img>`:

```html
<div class="card" style="background:#fff5f5;border-color:#fecaca">
  <strong>Screenshot unavailable</strong> for step <N>: <reason>
</div>
```

The intermediate file Claude writes is `$AUDIT_DIR/REPORT.linked.html` (small, references screenshots by path). STEP 5b produces `$AUDIT_DIR/REPORT.html` (large, fully self-contained, the deliverable).

### Severity colour coding (already in the CSS)

- `critical` → red, `major` → orange, `minor` → yellow, `nit` → grey.
- Apply to both the inline `.sev` chip and the left-border of `.finding` cards.

Also write `$AUDIT_DIR/recommendations.json`:

```json
{
  "target_url": "<url>",
  "device_profile": "<profile>",
  "flow_prompt": "<verbatim>",
  "viewport": {"width": 0, "height": 0, "dpr": 1},
  "totals": {
    "steps": 0,
    "findings": {"critical": 0, "major": 0, "minor": 0, "nit": 0}
  },
  "top_recommendations": [
    {
      "title": "...",
      "rationale": "...",
      "severity": "critical|major|minor|nit",
      "effort": "low|medium|high",
      "impact": "low|medium|high",
      "linked_findings": ["F-1-2", "F-3-1"]
    }
  ],
  "findings": [ /* every finding from every step, flat list */ ],
  "steps": [ /* the same steps[] shape persisted in audit-state.json */ ],
  "generated_at": "<ISO>"
}
```

The `recommendations.json` is the machine-readable deliverable; `REPORT.html` is the human-readable one. Both must be in sync — generate them from the same in-memory model.

---

## STEP 5b — Inline screenshots (Python post-processor)

The relative-path version of the report (`REPORT.linked.html`) only renders correctly when the viewer also serves the sibling `screenshots/` folder. Many viewers don't (cloud bucket previewers, email clients, zipped previews) — they serve the HTML alone, and every `<img>` 404s.

Fix: a tiny Python script that reads `REPORT.linked.html`, replaces every `<img src="screenshots/...">` with a `data:image/png;base64,…` URI inlined from the on-disk PNG, and writes `REPORT.html`. Claude never has to read or emit base64 — Python does it deterministically.

### 5b.1 Write the post-processor

Write this **verbatim** to `$AUDIT_DIR/inline_images.py`:

```python
#!/usr/bin/env python3
"""Inline every <img src="screenshots/..."> in REPORT.linked.html as a base64 data URI.

Reads:  $AUDIT_DIR/REPORT.linked.html
Writes: $AUDIT_DIR/REPORT.html  (self-contained, single file)

Usage:  python3 inline_images.py [audit_dir]
        audit_dir defaults to the script's own directory.
"""
import base64
import mimetypes
import os
import re
import sys
from pathlib import Path

IMG_SRC_RE = re.compile(r'<img\b([^>]*?)\bsrc="([^"]+)"([^>]*)>', re.IGNORECASE)


def to_data_uri(path: Path) -> str | None:
    if not path.is_file():
        return None
    mime, _ = mimetypes.guess_type(str(path))
    if mime is None:
        # Fall back by extension
        ext = path.suffix.lower().lstrip(".")
        mime = f"image/{'jpeg' if ext in ('jpg', 'jpeg') else ext or 'png'}"
    b64 = base64.b64encode(path.read_bytes()).decode("ascii")
    return f"data:{mime};base64,{b64}"


def main() -> int:
    audit_dir = Path(sys.argv[1]) if len(sys.argv) > 1 else Path(__file__).resolve().parent
    src = audit_dir / "REPORT.linked.html"
    dst = audit_dir / "REPORT.html"

    if not src.is_file():
        print(f"ERROR: {src} not found", file=sys.stderr)
        return 1

    html = src.read_text(encoding="utf-8")
    replaced = 0
    missing: list[str] = []

    def repl(match: re.Match) -> str:
        nonlocal replaced
        pre, src_attr, post = match.group(1), match.group(2), match.group(3)
        # Skip already-inlined data URIs and absolute http(s) URLs.
        if src_attr.startswith(("data:", "http://", "https://")):
            return match.group(0)
        img_path = (audit_dir / src_attr).resolve()
        # Refuse to read anything outside audit_dir.
        try:
            img_path.relative_to(audit_dir.resolve())
        except ValueError:
            missing.append(src_attr)
            return match.group(0)
        data_uri = to_data_uri(img_path)
        if data_uri is None:
            missing.append(src_attr)
            return match.group(0)
        replaced += 1
        return f'<img{pre}src="{data_uri}"{post}>'

    new_html = IMG_SRC_RE.sub(repl, html)
    dst.write_text(new_html, encoding="utf-8")

    print(f"OK: inlined {replaced} image(s) → {dst}")
    if missing:
        print(f"WARN: {len(missing)} reference(s) could not be inlined:")
        for m in missing:
            print(f"  - {m}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 5b.2 Run it

```bash
python3 "$AUDIT_DIR/inline_images.py" "$AUDIT_DIR"
```

Expected output: `OK: inlined N image(s) → <audit_dir>/REPORT.html`. If any image is missing, the script prints a `WARN` line listing the unresolved `src` values; the HTML is still produced — broken `<img>` references are left untouched (they'll show as broken icons rather than crashing the report).

### 5b.3 Verify

After the script runs, the deliverable is `$AUDIT_DIR/REPORT.html`. A quick sanity check:

```bash
# Confirm REPORT.html contains data URIs and no remaining "screenshots/" path refs.
grep -c 'data:image' "$AUDIT_DIR/REPORT.html"               # should be > 0
grep -c 'src="screenshots/' "$AUDIT_DIR/REPORT.html"        # should be 0
ls -lh "$AUDIT_DIR/REPORT.html"                              # expect MB-scale, not KB
```

If `data:image` count is 0 → the regex didn't match anything (likely the linked HTML had no `<img>` tags or used different quoting). Open `REPORT.linked.html`, confirm the tag shape, and re-run.

### 5b.4 Why this works where relative paths failed

- **Single file, no companions needed.** The data URIs travel inside the HTML. Hand the file to anyone — bucket viewer, email, ZIP, Slack — the screenshots render.
- **Deterministic.** No path-rewriting heuristics in the viewer, no CORS, no asset proxy.
- **Cheap on Claude's context.** Claude writes plain `<img src="screenshots/step-N.png">` (a few dozen bytes per tag). The base64 (kilobytes per image) only ever lives on disk and inside the Python process.

### 5b.5 Trade-offs

- File size: a 12-step audit at iPhone 14 (~150 KB per PNG) yields a ~2.5 MB HTML — fine for any modern viewer, but slow to scroll on very low-end machines.
- If size becomes a concern, the script can be extended to recompress PNGs to JPEG at quality 85 before encoding. Not added by default — lossless is preferable for a UX audit where button-edge sharpness matters.

`REPORT.linked.html` is kept on disk as a smaller artifact alongside `REPORT.html` — useful for re-running the post-processor with different settings without regenerating the report.

---

## STEP 6.a — Main agent's response to the user (IMMEDIATELY after STEP 3.5 dispatch)

The instant the background `Agent` call has been dispatched, the main agent's job is done. Reply to the user with **exactly** this message — no embellishment, no preamble, no trailing summary:

```
Okay : Task is initiated. Please check after sometime. Look For folder for progress. Thank you.

Folder: <AUDIT_DIR>
Progress log: <AUDIT_DIR>/progress.log
State: <AUDIT_DIR>/audit-state.json   (status field: planned → running → completed | failed)
Final report (when done): <AUDIT_DIR>/REPORT.html
```

Substitute `<AUDIT_DIR>` with the actual absolute path. Do not add anything else. Do not describe what the executor is doing. Do not estimate completion time. Do not say "I'll let you know when it's done" — the main agent is not staying alive for that.

## STEP 6.b — Executor's finalisation (inside the background subagent)

When the executor finishes its step loop, the report, and the post-processor:

```bash
jq --arg ts "$(date -u +%FT%TZ)" \
   '.status = "completed" | .completed_at = $ts | .updated_at = $ts' \
   "$AUDIT_DIR/audit-state.json" > /tmp/as.tmp && mv /tmp/as.tmp "$AUDIT_DIR/audit-state.json"
touch "$AUDIT_DIR/.audit-done"
echo "[$(date -u +%FT%TZ)] completed: REPORT.html ready" >> "$AUDIT_DIR/progress.log"
```

On any hard stop the executor hits:

```bash
jq --arg ts "$(date -u +%FT%TZ)" --arg err "$ERROR_REASON" \
   '.status = "failed" | .last_error = $err | .updated_at = $ts' \
   "$AUDIT_DIR/audit-state.json" > /tmp/as.tmp && mv /tmp/as.tmp "$AUDIT_DIR/audit-state.json"
echo "[$(date -u +%FT%TZ)] FAILED: $ERROR_REASON" >> "$AUDIT_DIR/progress.log"
```

Partial deliverables (whatever screenshots and `REPORT.linked.html` already exist) are left on disk. The user can still open the partial report.

## STEP 6.c — How the user checks progress (informational; no agent action)

The user picks one of:

| Question | Where to look |
|---|---|
| "Is it done?" | `cat <AUDIT_DIR>/audit-state.json \| jq -r '.status'` — returns `running` / `completed` / `failed`. |
| "Where is it up to?" | `tail -n 20 <AUDIT_DIR>/progress.log` |
| "How many steps so far?" | `jq -r '.current_step, (.steps \| length)' <AUDIT_DIR>/audit-state.json` |
| "Show me the screenshots so far" | `ls <AUDIT_DIR>/screenshots/` |
| "Open the final report" | `open <AUDIT_DIR>/REPORT.html` (only when status = `completed`) |

---

## Resume contract

If `$AUDIT_DIR/audit-state.json` already exists and `.audit-done` does not, this is a resume:

1. Read `current_step` and `steps[]`.
2. Re-run STEP 1 (doctor) and STEP 2 (viewport) — viewport may have been lost when the previous session ended.
3. Re-`open` the last known URL (`steps[current_step-1].url_after` or `target_url`).
4. Continue from `current_step + 1`.
5. Do NOT re-run earlier steps — their findings/screenshots are still on disk.

---

## What this skill will NOT do

- Submit payments or any irreversible action.
- Create real accounts. Use placeholder PII (`test+audit@example.com`, `Test User`, test card `4111 1111 1111 1111`).
- Run Lighthouse / WebPageTest / axe-core. Heuristics are visual + snapshot-based only. If the user wants those, they'll ask for a separate skill.
- Audit behind-login flows unless the user has explicitly logged the agent-browser session in beforehand. If a step lands on a login wall or a bot challenge (Cloudflare, hCaptcha, reCAPTCHA), log a `critical` finding ("audit blocked by auth/challenge — out of scope") and stop the flow gracefully — still produce the HTML report with the screenshots collected so far.
- Make code changes. This skill produces a report, not a PR.
