---
name: agent-browser
description: "How to use the agent-browser CLI (vercel-labs/agent-browser) reliably from inside Phase 1 skills. Command reference, the --cdp flag, the snapshot→ref→action pattern, and a self-healing recovery decision tree for when clicks miss, selectors vanish, pages don't load, or tabs go stale. Triggers: 'agent-browser', 'browser fallback', 'browser self heal', 'cdp', 'browser stuck', 'click not working', 'snapshot ref'."
---

# Using agent-browser inside Phase 1

Source of truth: https://github.com/vercel-labs/agent-browser (`README.md` is the only documented surface; anything not explicitly cited there is marked "not documented" below).

---

## ABSOLUTE RULES

1. **Single CDP, sequential only.** Phase 1 connects to **one** Chrome instance via CDP. **Never** issue parallel `agent-browser` calls. **Never** spawn parallel subagents that touch the browser. One command, await result, next command.
2. **`--cdp` on EVERY call. No exceptions.** Never use `agent-browser connect` to establish a persistent session and then drop the flag on subsequent commands. That mode silently breaks against the VortexIQ proxy — see "Why per-call --cdp is mandatory" below. Every single `agent-browser` invocation in every Phase 1 skill MUST be of the form `agent-browser --cdp "$CDP" <subcommand>`. If you see a snippet that omits `--cdp`, it is wrong and must be fixed before running.
3. **Read `cdp_ws` fresh from `config.json` at the start of each phase.** Cache it in a local `$CDP` variable for that phase. Re-derive it (via the procedure in Recovery A) only when a call returns a WebSocket/auth error — re-deriving on every command is wasteful. The URL is stable within a session; it rotates after a CDP service restart or after an `agent-browser close` against the remote.
4. **Single canonical config key.** The WebSocket URL lives at `cdp_ws` (top-level) in `config.json`. The HTTP discovery URL lives at `cdp_http`. Ignore any other key (e.g. `agent_browser.cdp`) — it's a legacy duplicate.
5. **Snapshot before acting.** Before any `click`, `type`, `fill`, etc., run `snapshot -i` to get accessibility refs (e.g. `@e1`). Acting on stale or guessed selectors is the #1 cause of failed runs.
6. **Wait, don't sleep.** Use `wait --load networkidle` or `wait <selector>` instead of fixed `sleep` between actions.
7. **`close` is FORBIDDEN against the shared remote browser.** `agent-browser close` against the VortexIQ Chrome detaches the local session AND can leave the remote Chrome in an inconsistent tab state. Use `tab close <n>` to remove individual tabs instead. The only place `close` is acceptable is the very end of a phase when the orchestrator explicitly wants a fresh remote session — and even then, prefer `tab close` per-tab.

---

## Standard invocation pattern

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
agent-browser --cdp "$CDP" <subcommand> [args]
```

`--cdp` accepts either a bare port (`9222`, resolved against `http://localhost:9222/json/version`) or a full `ws://` / `wss://` URL. For Phase 1 we always pass the `wss://chrome-cdp.vortexiq.ai/...` form derived in the bootstrap phase.

### Why per-call `--cdp` is mandatory (and `agent-browser connect` is forbidden)

The upstream `vercel-labs/agent-browser` README presents two modes:

```bash
# Mode A — per-call (what Phase 1 uses)
agent-browser --cdp "$WS_URL" snapshot

# Mode B — persistent connect (WHAT PHASE 1 FORBIDS)
agent-browser connect "$WS_URL"
agent-browser snapshot         # uses the saved session
```

**Mode B breaks against the VortexIQ Chrome proxy** because the auth token is carried in the WebSocket URL's query string (`wss://chrome-cdp.vortexiq.ai/devtools/browser/<id>?token=...`). `connect` persists the URL into a local session file, but the auth context does not survive cleanly across separate `agent-browser` processes invoked from bash. The next call reads the saved socket reference, opens a new WebSocket, and the proxy rejects it with:

```
✗ Navigation failed: net::ERR_INVALID_AUTH_CREDENTIALS
```

Passing `--cdp "$CDP"` on every call carries the full URL (token included) every time, so the proxy authenticates each request correctly. This is the *only* reliable mode against this proxy.

**Forbidden patterns** — none of these are allowed inside Phase 1 skills:

```bash
agent-browser connect "$WS_URL"   # FORBIDDEN — persistent mode
agent-browser snapshot            # FORBIDDEN — implicit-session mode (no --cdp)
agent-browser tab                 # FORBIDDEN — implicit-session mode
```

If a Phase 1 skill needs a "verify the browser is reachable" smoke test, do it with an explicit `--cdp` call like:

```bash
agent-browser --cdp "$CDP" get url || { echo "CDP unreachable"; exit 1; }
```

Do NOT use `agent-browser doctor` — see the next section.

---

## Command surface (grouped)

| Group | Commands | Purpose |
|---|---|---|
| **Lifecycle** | `open <url>`, ~~`close`~~, ~~`connect <port>`~~ | `open` only. `close` is forbidden (see Rule 7). `connect` is forbidden (see "Why per-call --cdp is mandatory"). |
| **Tabs/windows/frames** | `tab [new\|close <n>\|<tN>]`, `window new`, `frame <sel\|main>` | Manage tabs and iframes |
| **Capture** | `snapshot` (a11y tree, flags `-i -c -d -s -u`), `screenshot [path] [--full --annotate]`, `pdf <path>` | Read state of page |
| **Interaction** | `click`, `dblclick`, `focus`, `type <sel> <text>`, `fill <sel> <text>`, `press <key>`, `keyboard`, `hover`, `select`, `check`, `uncheck`, `scroll`, `scrollintoview`, `drag <src> <tgt>`, `upload <sel> <files>` | Drive the page |
| **Semantic finders** | `find role\|text\|label\|placeholder\|alt\|title\|testid\|first\|last\|nth <q> <action>` | Locate elements without raw selectors |
| **Wait** | `wait <sel\|ms>` with `--text`, `--url`, `--load <load\|domcontentloaded\|networkidle>`, `--fn`, `--state hidden` | Synchronise before next action |
| **Get info** | `get text\|html\|value\|attr\|title\|url\|cdp-url\|count\|box\|styles` | Extract data |
| **State checks** | `is visible\|enabled\|checked` | Conditional logic |
| **Batch / JS** | `batch <cmd1> <cmd2> ...`, `eval <js>` | Multiple ops in one round trip; arbitrary JS |
| **Cookies / storage** | `cookies [set\|clear]`, `storage local\|session [set\|clear]` | Session data |
| **Network** | `network route <url> [--abort --body --resource-type]`, `network unroute`, `network requests`, `network har start\|stop` | Mock / inspect requests |
| **Dialogs** | `dialog accept\|dismiss\|status` | Handle alerts/prompts |
| **Diff** | `diff snapshot\|screenshot\|url` | Detect changes |
| **Debug** | `trace`, `profiler`, `console`, `errors`, `highlight`, `inspect` | Diagnose problems |
| ~~Doctor~~ | ~~`doctor`~~ | **Not available in v0.23.x.** Earlier docs reference it; the installed Phase 1 binary returns `Unknown command: doctor`. Do not call it. Use `get url` against the live `$CDP` as the reachability test instead. |

The `batch` command runs multiple subcommands in one invocation — useful for atomic flows (snapshot + click + wait), but **does not** make them parallel; they execute in sequence within one process.

---

## The snapshot→ref→action pattern (mandatory)

Never do `agent-browser click "button.foo"` blindly. Do:

```bash
agent-browser --cdp "$CDP" snapshot -i           # returns refs like @e7, @e12
# parse the snapshot to find the right ref for the target element
agent-browser --cdp "$CDP" click "@e12"
```

`-i` includes interactive element refs. `-c` includes children, `-d` increases depth, `-s` filters by selector, `-u` includes URL.

---

## Self-healing decision tree

When `agent-browser` returns a non-zero exit, parse the error and apply the matching recovery in order. Stop at the first one that succeeds.

### A. "Failed to connect to CDP" / `ERR_INVALID_AUTH_CREDENTIALS` / WebSocket error

First, **rule out the most common cause**: a missing `--cdp` flag on the failing command. If the command being run is bare `agent-browser <cmd>` without `--cdp "$CDP"`, that's the bug — fix the call site (this skill mandates `--cdp` on every call, see Rule 2). `ERR_INVALID_AUTH_CREDENTIALS` against `wss://chrome-cdp.vortexiq.ai/...` is almost always this.

If `--cdp` is present and the call still fails:

1. Re-derive `cdp_ws` from scratch (the WS URL rotates after a CDP service restart or an `agent-browser close`):
   ```bash
   CDP_HTTP=$(jq -r '.cdp_http' config.json)
   CDP_HOST=$(echo "$CDP_HTTP" | sed 's|https://||')
   WS_URL=$(curl -s "$CDP_HTTP/json/version" | python3 -c "
   import sys, json; d = json.load(sys.stdin)
   print(d['webSocketDebuggerUrl'].replace('ws://localhost', 'wss://$CDP_HOST'))")
   jq --arg ws "$WS_URL" '.cdp_ws = $ws' config.json > /tmp/c.tmp && mv /tmp/c.tmp config.json
   CDP="$WS_URL"
   ```
2. Reachability test (DO NOT use `doctor` — see command surface table):
   ```bash
   agent-browser --cdp "$CDP" get url
   ```
   If this prints a URL, CDP is healthy. If it fails, the proxy or remote Chrome is genuinely down.
3. Re-issue the originally failing command with the refreshed `$CDP`.
4. If still failing → **hard stop** with the standard "agent-browser connection failed" message.

### B. Click / type / fill failed (selector or ref)

1. Re-snapshot fresh — refs change when DOM updates:
   ```bash
   agent-browser --cdp "$CDP" snapshot -i -c
   ```
2. Re-locate the element by **semantic finder** instead of raw selector:
   - Tried `click @e7` → try `find role button "Get results" click`
   - Tried `find text "Submit"` → try `find label "Submit"` or `find role button "Submit"`
3. If element not in viewport, scroll then retry:
   ```bash
   agent-browser --cdp "$CDP" scrollintoview "<sel>"
   agent-browser --cdp "$CDP" click "<sel>"
   ```
4. If still failing, check visibility/state:
   ```bash
   agent-browser --cdp "$CDP" is visible "<sel>"
   agent-browser --cdp "$CDP" is enabled "<sel>"
   ```
5. Last resort — `eval` to dispatch a click via JS:
   ```bash
   agent-browser --cdp "$CDP" eval "document.querySelector('<sel>').click()"
   ```

### C. Page didn't load / blank snapshot

1. `wait --load networkidle` (timeout 30s).
2. `get url` — confirm the navigation actually happened. If wrong URL → re-`open`.
3. `errors` and `console` — read browser-side error log.
4. Re-`open` the URL.
5. If still blank → likely auth issue; check the original URL is correct in `config.json`.

### D. Tab / window confusion (more tabs open than expected)

1. `tab` — list all tabs.
2. Close everything above tab 1 in descending order:
   ```bash
   TAB_COUNT=$(agent-browser --cdp "$CDP" tab | grep -cE '^\s*\[?[0-9]+')
   if [ "$TAB_COUNT" -gt 1 ]; then
     for i in $(seq "$TAB_COUNT" -1 2); do
       agent-browser --cdp "$CDP" tab close "$i" || true
     done
   fi
   agent-browser --cdp "$CDP" tab 1
   ```
3. Confirm with `tab` again before continuing.

### E. Dialog blocking the page

1. `dialog status` — see if a native dialog is open.
2. `dialog accept` (or `dismiss`) to clear it.
3. Re-snapshot and continue.

### F. Download didn't appear in `~/Downloads`

1. `wait 2000` — give the download a moment.
2. `ls -t ~/Downloads/<pattern>* 2>/dev/null | head -n 1` — pick newest.
3. If still nothing, the export click probably missed — go back to recovery B.
4. Always clear `~/Downloads/<pattern>*` **before** triggering the next download to keep filename selection deterministic.

### G. Stale element / DOM rebuilt mid-action

1. Re-snapshot.
2. Use a `find role/text` semantic finder rather than a previously captured `@eN` ref.

### H. Google CAPTCHA / `/sorry/index` / bot challenge

Symptom: a navigation to `google.com/search?...` or `ads.google.com` redirects to `https://www.google.com/sorry/index?...` and the page shows "Our systems have detected unusual traffic from your computer network." This will hit Stage C (Keyword Planner / SERP research) more than anywhere else.

Recovery — in order, stop at first success:

1. **Confirm it's the CAPTCHA wall** (not just a slow page):
   ```bash
   URL_NOW=$(agent-browser --cdp "$CDP" get url)
   echo "$URL_NOW" | grep -qE '/sorry/|/recaptcha/' && echo "CAPTCHA HIT" || echo "Not CAPTCHA"
   ```
2. **Back off and retry once** — the proxy IP may rotate or the rate-limit window may pass:
   ```bash
   agent-browser --cdp "$CDP" wait 15000
   agent-browser --cdp "$CDP" open "$ORIGINAL_URL"
   agent-browser --cdp "$CDP" wait --load networkidle
   agent-browser --cdp "$CDP" get url   # check we're off /sorry/
   ```
3. **Clear cookies & storage scoped to google.com, then retry**:
   ```bash
   agent-browser --cdp "$CDP" cookies clear google.com
   agent-browser --cdp "$CDP" storage local clear
   agent-browser --cdp "$CDP" open "$ORIGINAL_URL"
   ```
4. **Switch search target if the audit allows it.** For brand/competitor research only (not for Keyword Planner — KP has no substitute):
   - Bing: `https://www.bing.com/search?q=...`
   - DuckDuckGo HTML: `https://duckduckgo.com/html/?q=...`
   - Brave Search: `https://search.brave.com/search?q=...`
5. **If the failing surface is Google Ads / Keyword Planner specifically** (Stage C):
   - DO NOT switch to a different search engine — KP data is the deliverable, not substitutable.
   - Take a screenshot of the challenge page for the run log: `screenshot --output Step-4-Keyword-Research/screenshots/captcha-batch-NNN.png`.
   - Update `pipeline-state.json`: `stages.keyword_research.status = "blocked"`, `last_step = "captcha_at_batch_NNN"`.
   - **Hard stop** with this exact message — do not retry further, the user must resolve the challenge in their VortexIQ Chrome session manually before resuming:
     ```
     Google Keyword Planner blocked by CAPTCHA at batch NNN.
     Open the VortexIQ Chrome session (https://chrome-cdp.vortexiq.ai), solve the challenge interactively, then re-run Phase 1 — it will resume from this batch.
     Screenshot: Step-4-Keyword-Research/screenshots/captcha-batch-NNN.png
     ```
6. The CAPTCHA recovery counter (`stages.keyword_research.captcha_retries`) increments by 1 each time Recovery H runs. If it reaches 3 within one phase, treat it as a permanent block and hard-stop with the message above regardless of which step triggered it.

---

## Things NOT documented in the upstream README

- **Auth headers beyond URL query params** — if the CDP service needs custom headers, there's no documented flag. Embed the token in the `wss://...?token=...` URL.
- **Per-CDP serialisation guarantees** — the README does not promise that two concurrent calls against the same `--cdp` are safe. Treat it as **unsafe** and serialise from the caller (which Phase 1 already does).
- **Built-in retry / self-healing** — there is none. The recovery tree above is owned by *us*, not the CLI.

---

## When to give up (hard stop)

There are three hard-stop scenarios. Each has its own message — pick the one that matches.

### Hard stop 1 — CDP connection genuinely down

Trigger: Recovery A has been tried fully (including the WS URL re-derivation and `get url` reachability test) and the connection is still failing.

```
agent-browser connection failed — cannot continue. Please check the CDP endpoint at https://chrome-cdp.vortexiq.ai and try again.
Last attempted step: <stages.<X>.last_step from pipeline-state.json>
```

### Hard stop 2 — Google CAPTCHA on Keyword Planner

Trigger: Recovery H reached step 5 (KP-specific block) or step 6 (3+ CAPTCHAs in one phase).

```
Google Keyword Planner blocked by CAPTCHA at batch NNN.
Open the VortexIQ Chrome session (https://chrome-cdp.vortexiq.ai), solve the challenge interactively, then re-run Phase 1 — it will resume from this batch.
Screenshot: Step-4-Keyword-Research/screenshots/captcha-batch-NNN.png
```

### Hard stop 3 — Same micro-step failed twice with non-browser cause

Trigger: a step (not browser-related) has failed twice in a row according to `pipeline-state.json`, and the failure is not recoverable via the tree above.

```
Phase 1 micro-step <X.Y> failed twice in a row. Manual intervention required.
Reason: <last error captured>
```

In all three cases:

- Stop. Do not retry further.
- Do not delete partial deliverables (screenshots, CSVs, MDs).
- The orchestrator will resume from the last recorded step on next invocation.
