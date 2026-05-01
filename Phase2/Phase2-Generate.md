---
name: Phase2-Generate
description: "Phase 2: generate full pillar + spoke content for ONE pillar at a time. Triggered by 'Generate <pillar name>'. Reads the pillar's allocation from content_map.json (spokes, keywords, KD, volume, word budgets), the brand and competitor research, and the keyword density targets, then writes the pillar hub article and every spoke article with internal + external linking per pillar-content-clusters strategy. Runs editorial checks (no em-dashes, no emojis, word count, keyword density, keyword stuffing, dead-link check), rewrites until all checks pass, then writes a per-pillar deliverable JSON. Triggers: 'generate', 'generate pillar', 'generate <name>', 'phase 2', 'content generation', 'write pillar', 'write spokes'."
---

# Phase 2 — Content Generation (one pillar at a time)

**Invocation:** `Generate <pillar name>` — exactly one pillar per invocation. Run again with the next pillar name to continue. Never batch pillars in one run; the editorial loop is too long to fit two pillars in one context.

**Prerequisite:** Phase 1 must be complete. The following files MUST exist in `Pillar-Content-Architecture/`:
- `phase-1-deliverable.json`
- `content_map.json`
- `Step-1-Brand-Discovery/brand-discovery.md`
- `Step-2-Competitor-Discovery/competitor-discovery.md`
- `Step-4-Keyword-Research/master-keywords.csv`
- `Step-4-Keyword-Research/shortlisted-keywords.csv`

If any are missing → stop and tell the user "Phase 1 not complete — cannot generate content."

---

## ABSOLUTE RULES

1. **One pillar per invocation.** Do not start a second pillar in the same run.
2. **No em-dashes (`—`, `–`).** Use a regular hyphen `-` or rephrase. Editorial check enforces this.
3. **No emojis anywhere.** Plain text + HTML only.
4. **Editorial loop is mandatory.** Generate → check → rewrite if any check fails → re-check. Repeat until **every** check passes for **every** article. No exceptions.
5. **State JSON is truth.** Update `pipeline-state.json` (in `Pillar-Content-Architecture/`) after every micro-step under `stages.phase_2.<pillar_slug>.last_step`. Resume reads it.
6. **Sequential only for any external calls.** If verifying external links via `agent-browser`, single CDP, no parallelism — see `Phase1/agent-browser.md`.
7. **Strategy reference is `pillar-content-clusters` skill.** Every article (hub + spokes) must follow the structure defined there: pillar = 3000-5000 word hub with summaries linking out to spokes; spoke = 1500-3000 word deep-dive linking back to pillar 2-3 times + cross-linking to other spokes + product/collection links.

---

## STATE JSON ADDITIONS

Add a `phase_2` slot to `pipeline-state.json` (created on first invocation if absent):

```json
{
  "stages": {
    "phase_2": {
      "<pillar_slug>": {
        "status": "pending|in_progress|completed|failed",
        "last_step": "string",
        "attempts": 0,
        "articles": {
          "<article_slug>": {
            "status": "drafting|editorial_check|rewriting|passed",
            "iterations": 0,
            "last_failed_check": "string|null"
          }
        },
        "started_at": "ISO-8601",
        "completed_at": "ISO-8601?"
      }
    }
  }
}
```

Update pattern after every micro-step:

```bash
jq --arg p "<pillar_slug>" --arg step "<step_id>" \
   '.stages.phase_2[$p].status = "in_progress"
    | .stages.phase_2[$p].last_step = $step
    | .updated_at = (now | todate)' \
   pipeline-state.json > /tmp/ps.tmp && mv /tmp/ps.tmp pipeline-state.json
```

---

## STEP 0 — Resolve the pillar

Parse the pillar name from the invocation. Match it (case-insensitive, whitespace-tolerant) against `content_map.json:pillars[].pillar_name`. Pick the first exact match, then the first prefix match, then fail with the list of available pillar names if no match.

```bash
PILLAR_NAME="<arg>"
PILLAR_SLUG=$(echo "$PILLAR_NAME" | tr '[:upper:] ' '[:lower:]-' | tr -cd 'a-z0-9-')

PILLAR_JSON=$(jq --arg n "$PILLAR_NAME" '
  .pillars
  | map(select(.pillar_name | ascii_downcase == ($n | ascii_downcase)))
  | first
  // (.pillars | map(select(.pillar_name | ascii_downcase | startswith($n | ascii_downcase))) | first)
' content_map.json)

if [ "$PILLAR_JSON" = "null" ] || [ -z "$PILLAR_JSON" ]; then
  echo "Pillar not found. Available pillars:"
  jq -r '.pillars[].pillar_name' content_map.json
  exit 1
fi
```

**Resume check:**
```bash
EXISTING=$(jq -r --arg p "$PILLAR_SLUG" '.stages.phase_2[$p].status // "absent"' pipeline-state.json)
[ "$EXISTING" = "completed" ] && echo "$PILLAR_NAME already generated. Re-invoke with a different pillar." && exit 0
```

If `status = in_progress` or `failed`, resume from `last_step` rather than restart.

---

## STEP 1 — Read all inputs for this pillar

Build a single in-memory context bundle for the pillar — the model reads each file once:

| File | What to extract |
|---|---|
| `content_map.json` (this pillar entry) | `hub`, `spokes[]`, `tier`, `why_this_pillar`, `cross_pillar_bridges` referencing this pillar, `linking_rules`, `pillar_totals` |
| `Step-1-Brand-Discovery/brand-discovery.md` | Brand voice, positioning, products, USP, audience, terminology to use, terminology to avoid |
| `Step-2-Competitor-Discovery/competitor-discovery.md` | Competitor URLs (for benchmark + avoid copying), gaps competitors are missing |
| `Step-3-Content-Gap-Analysis/content-gap-analysis.md` | Specific opportunities + intent angles for this pillar's keywords |
| `Step-4-Keyword-Research/master-keywords.csv` | Keyword universe — used to find related keywords + secondary keywords for each spoke |
| `Step-4-Keyword-Research/shortlisted-keywords.csv` | Curated set — primary + secondary keyword assignments |
| `pillar-content-clusters` skill (read it now) | Structure rules: hub structure, spoke structure, linking strategy, anchor text variation, FAQ schema |

**Keyword density target derivation:** for each article, primary keyword density target = 0.8%-1.5% of word count (rule from `pillar-content-clusters`). Secondary keywords each 0.3%-0.8%. These are the editorial targets used in Step 6.

Persist the bundle to `Pillar-Content-Architecture/Phase-2-Content/<pillar_slug>/_context.json` so a resume can re-read it without re-parsing every source file.

---

## STEP 2 — Generate the pillar hub article

Working dir: `Pillar-Content-Architecture/Phase-2-Content/<pillar_slug>/`. Create if missing.

For the hub article (`hub` field in the pillar entry):

1. **Title** — already set in `content_map.json:pillars[].hub.title`. Use verbatim.
2. **Word budget** — `hub.word_budget` (typically 3000-5000).
3. **Primary keyword** — `hub.target_keyword`.
4. **Secondary keywords** — pick 4-6 from `master-keywords.csv` whose `product_category` matches this pillar AND that are NOT used as primary keywords by any spoke (avoids cannibalisation).
5. **Structure (per `pillar-content-clusters`):**
   - H1 (contains primary keyword)
   - 150-250 word intro
   - Table of contents with jump links
   - One H2 section per spoke topic, each 200-400 words, ending with `<a href="<spoke_slug>/">Read our full guide: <spoke title></a>`
   - Comparison or summary table
   - FAQ (5-8 questions with FAQPage JSON-LD)
   - Conclusion + CTA
   - Related Articles section listing every spoke
6. **Internal links from hub:**
   - One link to every spoke (mandatory per `linking_rules.every_pillar_links_out_to_all_spokes = true`).
   - Optional links to category/product pages.
7. **External links:** 2-4 authoritative references (e.g. industry bodies, government sites, original research). Avoid linking to direct competitors.
8. **Anchor text:** vary per the matrix in `pillar-content-clusters`. Never use identical anchor text twice.

Write to `<pillar_slug>/hub.html` and `<pillar_slug>/hub.meta.json` (the meta has title, slug, primary keyword, secondary keywords, word target, link targets — populated by Step 6's checker).

Update state: `last_step = "2.hub_drafted"`.

---

## STEP 3 — Generate each spoke article

Iterate spokes **sequentially** (never parallel). For each spoke in `pillars[].spokes[]`:

1. **Title** — already in `spokes[N].title`. Use verbatim.
2. **Word budget** — `spokes[N].word_budget` (typically 1500-3000).
3. **Primary keyword** — `spokes[N].target_keyword`.
4. **Secondary keywords** — `spokes[N].secondary_keywords` (already set in Phase 1).
5. **Structure (per `pillar-content-clusters`):**
   - H1 (contains primary keyword)
   - Breadcrumbs (Home > Blog > [title])
   - Article meta line (no real author needed — use brand name + estimated read time from word count / 200 wpm)
   - Intro (150-250 words) with one link to the hub within the first 2 paragraphs
   - Table of contents
   - Core content sections (H2/H3)
   - Comparison table if applicable to the keyword
   - FAQ section (5-8 questions) with FAQPage JSON-LD
   - Verdict / summary
   - "More From Our Resource Library" section linking to 2-4 other spokes in the same pillar (use `spokes[].title` + `slug` from content_map)
   - Disclaimer if compliance-relevant
6. **Internal links from spoke (per `linking_rules`):**
   - 2-3 links back to the pillar hub (intro / mid / conclusion area), with **varied anchor text**.
   - 2-4 links to other spokes in this pillar (cross-spoke).
   - For any cross-pillar bridge listed in `content_map.cross_pillar_bridges` that involves this spoke's topic, include 1 link using the suggested anchor text.
   - 3-6 product links and 1-3 collection links from the brand (extract product/collection URLs from `brand-discovery.md`).
   - Total internal links per spoke: **8-16** (per `pillar-content-clusters`).
7. **External links:** 1-3 authoritative references. No competitors.
8. **Anchor text variation:** mandatory. Never reuse the same phrase across two links pointing to the same destination.

Write to `<pillar_slug>/<spoke_slug>.html` and `<pillar_slug>/<spoke_slug>.meta.json` for each spoke.

Update state after each spoke: `last_step = "3.spoke.<n>.drafted"`, `articles[<spoke_slug>].status = "drafting"` → `editorial_check` after Step 4.

---

## STEP 4 — Editorial checks (per article)

Run on every article (hub + every spoke) sequentially. Each check produces a pass/fail. **Any fail triggers Step 5 (rewrite) for that article.**

### 4.1 No em-dashes, no en-dashes, no emojis

```bash
ARTICLE=<pillar_slug>/<article_slug>.html
EM=$(grep -cE '[\xE2\x80\x94\xE2\x80\x93]' "$ARTICLE")
EMOJI=$(python3 -c "
import sys, re
text = open('$ARTICLE', encoding='utf-8').read()
emoji_pattern = re.compile('[\U0001F300-\U0001FAFF\U00002600-\U000027BF\U0001F000-\U0001F02F\U0001F100-\U0001F2FF]')
print(len(emoji_pattern.findall(text)))
")
[ "$EM" -gt 0 ] && echo "FAIL em-dash: $EM"
[ "$EMOJI" -gt 0 ] && echo "FAIL emoji: $EMOJI"
```

Fix: rewrite the offending lines, replacing `—` with `-` (or restructuring), removing all emoji characters.

### 4.2 Word count

Strip HTML tags, count words. Pass criteria:
- Hub: within ±10% of `hub.word_budget`.
- Spoke: within ±10% of `spokes[N].word_budget`.

```bash
WORDS=$(python3 -c "
import re, sys
text = open('$ARTICLE', encoding='utf-8').read()
plain = re.sub(r'<[^>]+>', ' ', text)
print(len(plain.split()))
")
```

Fail outside ±10% → rewrite to expand or trim.

### 4.3 Keyword presence

Primary keyword must appear:
- In H1 (mandatory)
- In meta title (in `.meta.json`)
- In meta description (in `.meta.json`)
- In first paragraph (within first 100 words)
- In conclusion / last paragraph

Each secondary keyword must appear at least once in the body.

```bash
PRIMARY=$(jq -r '.primary_keyword' "$ARTICLE.meta.json")
grep -iq "$PRIMARY" "$ARTICLE" || echo "FAIL: primary keyword missing entirely"
# (additional positional checks via python script)
```

### 4.4 Keyword density (per `pillar-content-clusters` targets)

For each article:
- Primary keyword density: target 0.8%-1.5% (count of occurrences ÷ word count).
- Each secondary keyword density: target 0.3%-0.8%.

```bash
density() {
  local kw="$1" total="$2"
  local hits=$(grep -ioE "\\b$kw\\b" "$ARTICLE" | wc -l)
  awk -v h="$hits" -v t="$total" 'BEGIN { if (t==0) print 0; else printf "%.4f", h/t }'
}
```

Below target → rewrite to weave the keyword in more naturally. Above target → see 4.5.

### 4.5 Keyword stuffing detection

Stuffing thresholds (any one fails the article):
- Primary density > 2.5%
- Any secondary density > 1.5%
- Any keyword appears more than 3 times in the same paragraph
- Any 5-word window contains the primary keyword twice

Fail → rewrite the offending sections to thin out occurrences.

### 4.6 Internal link count + dead links

```bash
# Count internal links (relative URLs or same-domain)
INTERNAL=$(grep -oE 'href="(/[^"]+|\.\./[^"]+|[a-z0-9-]+/[^"]+)"' "$ARTICLE" | wc -l)
# Count external links
EXTERNAL=$(grep -oE 'href="https?://[^"]+"' "$ARTICLE" | wc -l)
```

Pass criteria:
- Hub: internal links ≥ (number of spokes in pillar) AND external 2-4
- Spoke: internal 8-16 AND external 1-3

**Dead-link check:** for every external link, HEAD-request it. Use `agent-browser` only if a normal `curl` fails (single CDP rule still applies):

```bash
for url in $(grep -oE 'https?://[^"]+' "$ARTICLE"); do
  CODE=$(curl -s -o /dev/null -L -w "%{http_code}" --max-time 10 "$url")
  [ "$CODE" -ge 400 ] && echo "DEAD: $url ($CODE)"
done
```

For internal links, verify the target file exists in `<pillar_slug>/` or in the brand's site (compare against URLs harvested from `brand-discovery.md`).

Any dead link → fail; rewrite to swap in a working alternative.

### 4.7 Anchor text variation

Build a map of `(href, anchor_text)` pairs. If two pairs share the same href AND identical anchor text → fail. Rewrite one of them.

### 4.8 Cross-link compliance

- Hub must link to **every** spoke at least once. Compute `set(spoke_slugs) - set(hrefs in hub)` — must be empty.
- Each spoke must link back to the hub at least 2-3 times.
- Each spoke should link to 2-4 other spokes in the pillar (not mandatory but recommended; flag as warning if 0).

---

## STEP 5 — Rewrite loop

For each article that fails any check in Step 4:

1. Set state: `articles[<slug>].status = "rewriting"`, `articles[<slug>].last_failed_check = "<check_id>"`, `articles[<slug>].iterations += 1`.
2. Rewrite **only the failing sections**, not the entire article. Pass the failing checks list as constraints to the rewrite.
3. Re-run **all** Step 4 checks (not just the failed one — fixing one can break another).
4. Repeat until all checks pass.
5. **Hard cap: 5 iterations per article.** If still failing after 5, set `articles[<slug>].status = "failed"`, log `last_failed_check`, continue to next article. Do not block the whole pillar on one stubborn article.

When all articles in the pillar pass (or hit the 5-iteration cap), proceed to Step 6.

---

## STEP 6 — Per-pillar deliverable JSON

Write `Pillar-Content-Architecture/Phase-2-Content/<pillar_slug>/pillar-deliverable.json`.

This is the canonical Phase 2 output for this pillar. One entry per article (1 hub + N spokes).

```json
{
  "schema_version": "1.0",
  "generated_at": "ISO-8601",
  "pillar_number": 1,
  "pillar_name": "string",
  "pillar_slug": "string",
  "pillar_tier": "T1|T2|T3",
  "articles": [
    {
      "type": "hub",
      "title": "string",
      "slug": "string",
      "primary_keyword": "string",
      "secondary_keywords": ["string"],
      "combined_search_volume": 0,
      "kd": 0,
      "gap_type": "string (from content-gap-analysis.md, e.g. 'missing comparison content')",
      "product": "string|null (which brand product/category this maps to)",
      "priority": "Critical|High|Medium|Low",
      "tier": "T1|T2|T3",
      "status": "passed|failed_after_5_iterations",
      "word_count": 0,
      "internal_link_count": 0,
      "interlinks": [
        { "href": "string", "anchor_text": "string", "to_type": "spoke|product|collection|hub|external" }
      ],
      "external_link_count": 0,
      "external_links": [
        { "href": "string", "anchor_text": "string", "domain": "string" }
      ],
      "primary_kw_density": 0.0,
      "all_kw_densities": {
        "<keyword>": 0.0
      },
      "suggested_post_date": "YYYY-MM-DD"
    },
    {
      "type": "spoke",
      "title": "string",
      "slug": "string",
      "primary_keyword": "string",
      "secondary_keywords": ["string"],
      "combined_search_volume": 0,
      "kd": 0,
      "gap_type": "string",
      "product": "string|null",
      "priority": "Critical|High|Medium|Low",
      "tier": "T1|T2|T3",
      "status": "passed|failed_after_5_iterations",
      "word_count": 0,
      "internal_link_count": 0,
      "interlinks": [{ "href": "...", "anchor_text": "...", "to_type": "..." }],
      "external_link_count": 0,
      "external_links": [{ "href": "...", "anchor_text": "...", "domain": "..." }],
      "primary_kw_density": 0.0,
      "all_kw_densities": { "<keyword>": 0.0 },
      "suggested_post_date": "YYYY-MM-DD"
    }
  ],
  "totals": {
    "articles_planned": 0,
    "articles_passed": 0,
    "articles_failed": 0,
    "total_word_count": 0,
    "total_internal_links": 0,
    "total_external_links": 0
  },
  "editorial_summary": {
    "total_iterations": 0,
    "articles_needing_rewrite": 0,
    "common_failure_modes": ["string"]
  }
}
```

### Field-source mapping

| Field | Source |
|---|---|
| `type` | `"hub"` for the pillar page, `"spoke"` for cluster articles |
| `title`, `slug` | `content_map.json` hub/spoke entry |
| `primary_keyword` | `content_map.json:hub.target_keyword` or `spokes[N].target_keyword` |
| `secondary_keywords` | hub: derived in Step 2; spoke: `content_map.json:spokes[N].secondary_keywords` |
| `combined_search_volume` | sum of `avg_monthly_searches` for primary + all secondaries from `master-keywords.csv` |
| `kd` | from `content_map.json` (may be `null`; preserve as-is) |
| `gap_type` | extracted from `content-gap-analysis.md` for this keyword's category |
| `product` | mapped from `brand-discovery.md` product list — pick the closest match to the keyword's `product_category` |
| `priority`, `tier` | `content_map.json:spokes[N].priority`, pillar `tier` |
| `status` | `articles[<slug>].status` from state JSON |
| `word_count` | actual count from final HTML (Step 4.2) |
| `internal_link_count`, `interlinks[]` | parsed from final HTML (Step 4.6) |
| `external_link_count`, `external_links[]` | parsed from final HTML (Step 4.6) |
| `primary_kw_density`, `all_kw_densities` | computed in Step 4.4 |
| `suggested_post_date` | start from today + offset based on tier (T1: +0 days, T2: +14 days, T3: +30 days), then space subsequent articles in the pillar 7 days apart in priority order |

---

## STEP 7 — Completion

```bash
jq --arg p "$PILLAR_SLUG" \
   '.stages.phase_2[$p].status = "completed"
    | .stages.phase_2[$p].last_step = "6.deliverable_written"
    | .stages.phase_2[$p].completed_at = (now | todate)
    | .updated_at = (now | todate)' \
   pipeline-state.json > /tmp/ps.tmp && mv /tmp/ps.tmp pipeline-state.json

touch "Phase-2-Content/$PILLAR_SLUG/.pillar-done"
```

Tell the user (single short message):

```
Pillar generated: <pillar name>
- Articles: 1 hub + N spokes (M passed, K failed after 5 iterations)
- Total words: <n>
- Total internal links: <n>   external: <n>
- Avg primary KW density: <%>
- Deliverable: Pillar-Content-Architecture/Phase-2-Content/<slug>/pillar-deliverable.json

Next pillar: invoke `Generate <next pillar name>`. Available remaining:
  - <pillar name 2>
  - <pillar name 3>
  ...
```

---

## FAILURE RECOVERY

- **Single article stuck:** capped at 5 iterations; logs as `failed`, pillar continues.
- **External dead link cannot be replaced:** drop the link, regenerate that paragraph without it, log a `notes[]` entry in the deliverable.
- **Brand site URLs unknown for product/collection links:** read `brand-discovery.md` more carefully; if still empty, use placeholders `https://<brand>.com/products/<slug>` and add a `notes[]` entry warning that URLs must be confirmed before publishing.
- **Resume after kill:** state JSON tracks per-article status; on re-invocation with the same pillar name, skip articles already `passed`, resume from the article with `drafting`/`rewriting`/`editorial_check` status.

---

## NOTES (design rationale)

- **Why one pillar per invocation:** a pillar is typically 1 hub + 8-15 spokes. Each article is 1500-5000 words. With editorial loops, that is hours of generation. Doing two pillars in one run thrashes the context and increases failure-mode coupling.
- **Why per-article 5-iteration cap:** without a cap, a single stubborn article (e.g. one that can't hit a keyword density target without sounding stuffed) would block the entire pillar indefinitely. Cap + log + continue is more useful than infinite loop.
- **Why suggested_post_date staggers by tier:** T1 quick-wins should publish immediately; T2 foundation can wait 2 weeks; T3 authority pieces are larger and benefit from being spaced out so internal-link signals are not all dumped at once.
- **Why we re-run ALL checks after a rewrite (not just the failing one):** fixing a keyword density issue can introduce new dead links, push word count out of range, or create duplicate anchor text. The cheapest correctness guarantee is re-running the full battery.
- **Why we don't validate internal links via HTTP:** at generation time the spokes are not yet deployed to the brand's domain. Internal-link validation is structural (file exists in `<pillar_slug>/`) rather than network.
