---
name: agent-browser
description: "How to use the agent-browser CLI (vercel-labs/agent-browser) reliably from inside Phase 1 skills. Command reference, the --cdp flag, the snapshot→ref→action pattern, and a self-healing recovery decision tree for when clicks miss, selectors vanish, pages don't load, or tabs go stale. Triggers: 'agent-browser', 'browser fallback', 'browser self heal', 'cdp', 'browser stuck', 'click not working', 'snapshot ref'."
---

# Using agent-browser inside Phase 1

Source of truth: https://github.com/vercel-labs/agent-browser (`README.md` is the only documented surface; anything not explicitly cited there is marked "not documented" below).

---

## ABSOLUTE RULES

1. **Single CDP, sequential only.** Phase 1 connects to **one** Chrome instance via CDP. **Never** issue parallel `agent-browser` calls. **Never** spawn parallel subagents that touch the browser. One command, await result, next command.
2. **Read `cdp_ws` fresh from `config.json` every time.** Never hardcode the WebSocket URL — it changes when the CDP service restarts.
3. **Snapshot before acting.** Before any `click`, `type`, `fill`, etc., run `snapshot -i` to get accessibility refs (e.g. `@e1`). Acting on stale or guessed selectors is the #1 cause of failed runs.
4. **Wait, don't sleep.** Use `wait --load networkidle` or `wait <selector>` instead of fixed `sleep` between actions.

---

## Standard invocation pattern

```bash
export PATH="$PATH:/home/saasvortex/.npm-global/bin"
CDP=$(jq -r '.cdp_ws' config.json)
agent-browser --cdp "$CDP" <subcommand> [args]
```

`--cdp` accepts either a bare port (`9222`, resolved against `http://localhost:9222/json/version`) or a full `ws://` / `wss://` URL. For Phase 1 we always pass the `wss://chrome-cdp.vortexiq.ai/...` form derived in the bootstrap phase.

---

## Command surface (grouped)

| Group | Commands | Purpose |
|---|---|---|
| **Lifecycle** | `open <url>`, `close`, `connect <port>` | Navigate, attach to existing browser |
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
| **Doctor** | `doctor` | Health-check the CLI itself |

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

### A. "Failed to connect to CDP" / WebSocket error

1. Re-derive `cdp_ws` from scratch (the websocket URL rotates):
   ```bash
   CDP_HTTP=$(jq -r '.cdp_http' config.json)
   CDP_HOST=$(echo "$CDP_HTTP" | sed 's|https://||')
   WS_URL=$(curl -s "$CDP_HTTP/json/version" | python3 -c "
   import sys, json; d = json.load(sys.stdin)
   print(d['webSocketDebuggerUrl'].replace('ws://localhost', 'wss://$CDP_HOST'))")
   jq --arg ws "$WS_URL" '.cdp_ws = $ws' config.json > /tmp/c.tmp && mv /tmp/c.tmp config.json
   ```
2. Run `agent-browser --cdp "$WS_URL" doctor` to confirm the binary itself is healthy.
3. Re-issue the failing command with the new `$CDP`.
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

---

## Things NOT documented in the upstream README

- **Auth headers beyond URL query params** — if the CDP service needs custom headers, there's no documented flag. Embed the token in the `wss://...?token=...` URL.
- **Per-CDP serialisation guarantees** — the README does not promise that two concurrent calls against the same `--cdp` are safe. Treat it as **unsafe** and serialise from the caller (which Phase 1 already does).
- **Built-in retry / self-healing** — there is none. The recovery tree above is owned by *us*, not the CLI.

---

## When to give up (hard stop)

Hard stop and surface to the user only when **all** apply:
- A and at least one of B/C have been tried and failed.
- The same micro-step has failed twice in a row according to `pipeline-state.json`.
- Doctor (`agent-browser doctor`) reports an issue.

Use this exact message:

```
agent-browser connection failed — cannot continue. Please check the CDP endpoint and try again.
Last attempted step: <stages.<X>.last_step from pipeline-state.json>
```

Then stop. Do not retry further. Do not delete partial deliverables. The orchestrator will resume from the last recorded step on next invocation.
