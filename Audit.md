# Workspace context for Claude

This workspace is a **BigCommerce Stencil theme**, cloned from a live store via `stencil download`. You are working inside the AskVIQ Stencil platform — a chat-driven theme editor with a live iframe preview.

## What you can assume

- **Stencil CLI is installed** at `~/.nvm/versions/node/v24.15.0/bin/stencil`. It's on your PATH.
- **Theme files live in this directory**: `templates/`, `assets/scss/`, `assets/js/`, `lang/`, `meta/`, plus `config.json`, `schema.json`, `package.json`, `config.stencil.json`, `secrets.stencil.json`.
- **Store credentials** are already configured in `secrets.stencil.json`. Do not re-prompt the user for them.
- **Channel** is preconfigured (default `1`). Pass `-c 1` only if you need to override.

## Running the dev server — IMPORTANT

The platform manages preview ports for you. There is one preconfigured port baked into `config.stencil.json` for this workspace, and the channel is preconfigured (usually `1`).

### Easiest path: ask the user to click Run

Tell the user to click **Run dev server** in the workspace overflow menu (◆ icon at the top-left → "Run dev server"). The platform handles the port + channel + background launch for you and refreshes the iframe preview automatically. This is the preferred path for almost every situation.

### If you must start it yourself from the terminal

You **must** pass the channel flag, otherwise the CLI prompts interactively and the process hangs (no TTY in this environment):

```bash
nohup stencil start -c 1 > /tmp/stencil.log 2>&1 < /dev/null &
disown
```

Replace `1` with whatever channel the workspace was set up with. Read it from `config.stencil.json` if unsure, or ask the user.

### Rules

✅ **DO** run `stencil start -c <channel>` (always pass `-c`).
✅ **DO** run it backgrounded with stdin closed (`< /dev/null`) — interactive prompts will hang the agent otherwise.
❌ **DO NOT** pass `--port` to `stencil start`. Overriding the port breaks the iframe preview proxy and the user sees a blank page.
❌ **DO NOT** start it without `-c <channel>`. With multi-storefront stores it prompts and hangs.
❌ **DO NOT** start multiple `stencil start` processes — they will fail to bind.

Before starting, check if it is already running:
```bash
PORT=$(node -e "console.log(JSON.parse(require('fs').readFileSync('config.stencil.json','utf8')).port)")
ss -tln | grep ":$PORT" && echo "Already running on $PORT" || echo "Not running"
```

To stop (do **not** use `pkill -f 'stencil start'` — it kills its own parent shell because the pattern matches the pkill command line itself):
```bash
PORT=$(node -e "console.log(JSON.parse(require('fs').readFileSync('config.stencil.json','utf8')).port)")
PIDS=$(ss -tlnpH "sport = :$PORT" 2>/dev/null | grep -oP 'pid=\K[0-9]+' | sort -u)
[ -n "$PIDS" ] && kill -TERM $PIDS
```

## Where the user will see your changes

- The iframe preview in the right pane loads `http://<host>/preview/<workspace-id>/`. This is a reverse proxy to whatever port `stencil start` bound to.
- BrowserSync hot reload is proxied through too — most CSS/JS edits show up without a manual refresh.
- Handlebars template changes (`.html` files in `templates/`) require a refresh; the proxied reload icon in the URL bar handles that.

## Editing conventions

- **Styles** live in `assets/scss/`. Sass is compiled by Stencil at runtime in dev. Don't write to `assets/css/` directly.
- **JavaScript** lives in `assets/js/` and uses page-specific modules extending `PageManager`. Server data flows to client via the `{{{jsContext}}}` helper in templates.
- **Templates** are Handlebars. Use existing helpers (`{{lang 'key'}}`, `{{> components/...}}`, `{{#each}}`) rather than ad-hoc string concatenation.
- **Theme variations** live in `config.json` under `variations`. The Theme Editor (Page Builder) UI is driven by `schema.json`.
- **Localization**: `lang/en.json` and friends. New strings go through `{{lang 'my.key'}}`.

## Pushing changes back to the store

The platform exposes these in the workspace overflow menu (user-triggered, not for you to call directly):

- **Bundle theme** → `stencil bundle` (creates the .zip)
- **Publish to live store** → `stencil push -a -d` (uploads + activates + deletes oldest)
- **Pull config from live** → `stencil pull` (one-way config sync)

If the user asks you to publish, prefer telling them which menu item to click. If they explicitly tell you to run the CLI yourself, you may, but warn that it overwrites the live store.

## Secrets

`secrets.stencil.json` contains the BigCommerce access token. Treat it as sensitive — don't paste its contents into chat messages, log files, or commits. It's in `.gitignore` by default; keep it that way.

## Common gotchas

- The workspace folder is owned by `ec2-user`. You can `cd` and edit freely, but global installs (`sudo npm i -g ...`) are not allowed and not needed.
- Node version is fixed by nvm at v24. Don't `nvm use` a different version.
- The legacy `.stencil` file is auto-migrated by the CLI to `config.stencil.json` + `secrets.stencil.json` on first run. Both formats may briefly coexist.
- If `stencil start` errors with "port already in use", kill the previous process first.

## Quick reference

| Task | Command |
|---|---|
| Start preview | `nohup stencil start -c 1 > /tmp/stencil.log 2>&1 < /dev/null & disown` |
| Stop preview | Kill by port (never use `pkill -f`; it self-matches): `ss -tlnpH "sport = :$PORT" \| grep -oP 'pid=\K[0-9]+' \| xargs -r kill -TERM` |
| View preview log | `tail -f /tmp/stencil.log` |
| Re-fetch live theme | `stencil download --overwrite -c 1` |
| Bundle for upload | `stencil bundle` |
| Push to live | `stencil push -a -d` |

When in doubt, the user wants the smallest, most targeted edit that solves their request — don't refactor surrounding code unless asked.