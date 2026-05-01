---
name: init-pillar-content-architecture
description: "Stage A of Phase 1. Clones the SEO Domination Engine template from GitHub (royalschoolofmgt/pillar-content-creation-skill) into Pillar-Content-Architecture/, strips git history, writes the architecture completion marker. Idempotent. Triggers: 'init pillar content architecture', 'initialise pillar content', 'new SEO project', 'start pillar content', 'setup content workflow', 'create pillar architecture folder', 'create content architecture'."
---

# Init Pillar Content Architecture (Stage A)

## What this skill does

Clones the SEO Domination Engine template from `royalschoolofmgt/pillar-content-creation-skill` into `Pillar-Content-Architecture/` (in the current working directory), removes the `.git` history, and writes a completion marker the orchestrator reads.

**Workspace root after this step:** `Pillar-Content-Architecture/` (NOT a `Content-Creation-Workflow/` subfolder — the cloned repo's root *is* the workflow root).

---

## Hard rules

- **Sequential only.** This skill executes its 5 steps in strict order. Never parallelise.
- **No questions.** No user prompts in any step.
- **Idempotent.** Re-running over an existing valid workspace is a no-op.

---

## Steps

**1. Idempotency check**

If `Pillar-Content-Architecture/config.json` already exists, the workspace is already initialised. Update the state JSON and exit.

```bash
if [ -f "Pillar-Content-Architecture/config.json" ]; then
  touch "Pillar-Content-Architecture/.architecture-done"
  if [ -f "Pillar-Content-Architecture/.pipeline-state.json" ]; then
    jq '.stages.architecture.status = "completed" | .stages.architecture.last_step = "idempotent_skip" | .updated_at = (now | todate)' \
       "Pillar-Content-Architecture/.pipeline-state.json" > /tmp/ps.tmp && mv /tmp/ps.tmp "Pillar-Content-Architecture/.pipeline-state.json"
  fi
  echo "Initialisation successful (already initialised)."
  exit 0
fi
```

**2. Clone the template**

```bash
gh repo clone royalschoolofmgt/pillar-content-creation-skill "Pillar-Content-Architecture" -- --depth=1
```

**3. Strip git history**

```bash
rm -rf "Pillar-Content-Architecture/.git"
```

**4. Verify expected layout**

Confirm the cloned repo contains the files later stages depend on. If any are missing, fail loudly.

```bash
for f in config.json seo-domination-report.html Master-Matrix.md \
         Step-1-Brand-Discovery/Steps.md \
         Step-2-Competitor-Discovery/Steps.md \
         Step-3-Content-Gap-Analysis/Steps.md \
         Step-3b-Content-Scope-Estimation/Steps.md \
         Step-4-Keyword-Research/Steps.md; do
  if [ ! -e "Pillar-Content-Architecture/$f" ]; then
    echo "ERROR: template missing $f"
    exit 1
  fi
done
```

**5. Write completion marker + state**

```bash
touch "Pillar-Content-Architecture/.architecture-done"
if [ -f "Pillar-Content-Architecture/.pipeline-state.json" ]; then
  jq '.stages.architecture.status = "completed" | .stages.architecture.last_step = "clone_verified" | .updated_at = (now | todate)' \
     "Pillar-Content-Architecture/.pipeline-state.json" > /tmp/ps.tmp && mv /tmp/ps.tmp "Pillar-Content-Architecture/.pipeline-state.json"
fi
echo "Initialisation successful."
```

Do not output anything else.
