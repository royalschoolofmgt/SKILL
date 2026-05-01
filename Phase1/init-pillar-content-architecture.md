---
name: init-pillar-content-architecture
description: "Initialise the Pillar Content Architecture workflow for a new client. Clones the SEO Domination Engine template from GitHub (royalschoolofmgt/pillar-content-creation-skill) into a new folder in the current working directory, strips git history, and prompts to fill in config.json. Triggers: 'init pillar content architecture', 'initialise pillar content', 'new SEO project', 'start pillar content', 'setup content workflow', 'create pillar architecture folder', 'create content architecture'."
---

# Init Pillar Content Architecture

## What this skill does

Clones the SEO Domination Engine template from `royalschoolofmgt/pillar-content-creation-skill` into a new folder in the current working directory, removes the `.git` history so it starts fresh, and guides you to fill in `config.json` for the new client.

---

## Steps

**1. Idempotency check**

If `Pillar-Content-Architecture/Content-Creation-Workflow/Master-Matrix.md` already exists, skip cloning. Tell the user: "Workspace already initialised — skipping clone." Then write the completion marker (step 4) and exit.

```bash
if [ -f "Pillar-Content-Architecture/Content-Creation-Workflow/Master-Matrix.md" ]; then
  mkdir -p "Pillar-Content-Architecture/Content-Creation-Workflow"
  touch "Pillar-Content-Architecture/Content-Creation-Workflow/.architecture-done"
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

**4. Write completion marker**

```bash
touch "Pillar-Content-Architecture/Content-Creation-Workflow/.architecture-done"
```

**5. Done**

Tell the user: "Initialisation successful."

Do not output anything else.