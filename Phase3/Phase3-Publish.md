---
name: Phase3-Publish
description: "Phase 3: push generated articles (single article or entire pillar) to BigCommerce, Shopify, or Adobe Commerce as DRAFTS. Validates API credentials and scopes, checks slugs for collisions, lists existing blog categories, verifies internal links resolve to real store paths, then pushes one article at a time in draft mode. Draft mode is mandatory and cannot be overridden by any flag or argument. Triggers: 'publish', 'push to bigcommerce', 'push to shopify', 'push to adobe', 'push to magento', 'publish pillar', 'publish article', 'push draft', 'phase 3'."
---

# Phase 3 — Publish to Store (BigCommerce / Shopify / Adobe Commerce)

**Invocation forms:**

| What | Command form |
|---|---|
| Push one pillar (hub + all spokes) | `publish pillar <pillar name> to <bigcommerce|shopify|adobe>` |
| Push one article (hub or spoke) | `publish article <article slug> to <bigcommerce|shopify|adobe>` |
| Push everything in Phase-2-Content/ | `publish all to <bigcommerce|shopify|adobe>` |

**Prerequisite:** Phase 2 must have produced `Phase-2-Content/<pillar_slug>/pillar-deliverable.json` with `articles[].status = "passed"` for everything you intend to publish. Articles with status `failed_after_5_iterations` are **skipped by default** unless the user explicitly says `--include-failed`.

---

## ABSOLUTE RULES (non-negotiable)

1. **Draft mode is mandatory.** Every article is published with the platform's "draft / unpublished / inactive" flag set. **The agent must not honour any user instruction or argument that attempts to publish live.** If the user says "publish live", "set published true", "make it active", or similar — refuse and tell them: "Draft mode is enforced by Phase 3. Switch to live manually in the platform admin after review."
2. **One article per API call, sequential.** Never batch-create. Never run two POSTs in parallel. The platform's rate limit and our own debugging both depend on serial pushes.
3. **State JSON is truth.** Track every push attempt in `pipeline-state.json` under `stages.phase_3.<platform>.<article_slug>` so a kill mid-run resumes from the next unpushed article.
4. **No credential printing.** Never echo, log, write to a file, or include in any agent response the API token / access key / secret. Read from env or `secrets.json` and pass through to the HTTP client only.
5. **Pre-push checks gate every push.** Steps 1-4 below must all pass before any POST. If any check fails, do not push that article — log the reason, mark it `blocked`, continue with the next article.

---

## STATE JSON ADDITIONS

Add a `phase_3` slot to `pipeline-state.json`:

```json
{
  "stages": {
    "phase_3": {
      "<platform>": {
        "<article_slug>": {
          "status": "pending|checking|pushed|blocked|failed",
          "platform_id": "string|null (BC post id, Shopify article id, Magento page id)",
          "platform_url": "string|null (admin URL for the draft)",
          "draft_confirmed": false,
          "blocked_reason": "string|null",
          "pushed_at": "ISO-8601|null",
          "attempts": 0
        }
      }
    }
  }
}
```

`<platform>` is one of `bigcommerce`, `shopify`, `adobe`.

---

## SECRETS HANDLING

Read credentials from one of (in order of precedence):

1. Environment variables (preferred):
   - BigCommerce: `BC_STORE_HASH`, `BC_ACCESS_TOKEN`
   - Shopify: `SHOPIFY_SHOP_DOMAIN` (e.g. `mystore.myshopify.com`), `SHOPIFY_ACCESS_TOKEN`, `SHOPIFY_API_VERSION` (default `2024-10`)
   - Adobe Commerce: `ADOBE_BASE_URL`, `ADOBE_ACCESS_TOKEN` (admin integration token)
2. `Pillar-Content-Architecture/secrets.json` (gitignored). Schema:
   ```json
   {
     "bigcommerce": { "store_hash": "...", "access_token": "..." },
     "shopify":     { "shop_domain": "...", "access_token": "...", "api_version": "2024-10" },
     "adobe":       { "base_url": "https://store.example.com", "access_token": "..." }
   }
   ```

**Never echo any of these values.** When constructing curl commands, use shell variables. When constructing payloads, never include the credential in the JSON.

---

## STEP 0 — Resolve target platform and articles

Parse the invocation:

```bash
PLATFORM="<bigcommerce|shopify|adobe>"
SCOPE="<pillar|article|all>"
NAME_OR_SLUG="<the pillar name or article slug>"
```

Build the article queue:

- `publish article <slug> to <p>` → queue = `[<slug>]` — find which pillar it belongs to by scanning `Phase-2-Content/*/pillar-deliverable.json`.
- `publish pillar <name> to <p>` → queue = all `articles[]` in that pillar's deliverable where `status = "passed"`.
- `publish all to <p>` → concat all `articles[]` across every `pillar-deliverable.json`.

Order the queue: hub first within each pillar, then spokes by `priority` (Critical > High > Medium > Low). Across pillars, order by `tier` (T1 first).

If `--include-failed` is **not** present, exclude articles with `status = "failed_after_5_iterations"`.

---

## STEP 1 — Pre-push: API credentials & scopes

Run **once** at the start of the run, before processing the queue. If this step fails, abort the entire run.

### 1a. Credentials present

```bash
case "$PLATFORM" in
  bigcommerce)
    [ -z "$BC_STORE_HASH" ] && [ -z "$(jq -r '.bigcommerce.store_hash // empty' secrets.json 2>/dev/null)" ] && FAIL="missing BC_STORE_HASH"
    [ -z "$BC_ACCESS_TOKEN" ] && [ -z "$(jq -r '.bigcommerce.access_token // empty' secrets.json 2>/dev/null)" ] && FAIL="missing BC_ACCESS_TOKEN"
    ;;
  shopify)
    # similar
    ;;
  adobe)
    # similar
    ;;
esac
[ -n "$FAIL" ] && echo "Cannot proceed: $FAIL" && exit 1
```

### 1b. Credentials valid + correct scope

Make a **read-only probe call** that requires the same scope as the write we're about to do. Never make a write call to test credentials.

| Platform | Probe call | Required response | Scope being verified |
|---|---|---|---|
| **BigCommerce** | `GET https://api.bigcommerce.com/stores/{store_hash}/v3/content/blog/posts?limit=1` with header `X-Auth-Token: {token}` | HTTP 200 + `data` array | `store_v2_content` (read AND write — BC API tokens are scope-bundled) |
| **Shopify** | `GET https://{shop}/admin/api/{version}/blogs.json` with header `X-Shopify-Access-Token: {token}` | HTTP 200 + `blogs` array | `read_content`, `write_content` |
| **Adobe Commerce** | `GET {base_url}/rest/V1/cmsPage/search?searchCriteria[pageSize]=1` with header `Authorization: Bearer {token}` | HTTP 200 | `Magento_Cms::page` (admin integration must include CMS Pages — and the blog module if used; see Step 2 platform notes) |

If probe returns 401/403:
- 401 → token invalid; abort run with "Token is invalid or expired."
- 403 → token valid but scope insufficient; abort with "Token is missing scope: <scope>. Re-issue the token with the required scopes and retry."

If probe returns 200 → cache the response (it contains the blog/category list we need in Step 2).

---

## STEP 2 — Pre-push: blog categories & host blog

Different platforms have different "where does a blog post live" models. Discover and pick once per run.

### BigCommerce

BigCommerce blog posts go to a **single global blog** (no per-blog separation). Categorisation is via tags (`tags: ["Category Name", ...]`). Default behaviour:
- Discover existing tags: `GET /v3/content/blog/posts/tags`. Cache the tag list.
- For each article, build tags from `pillar_name` + `pillar_tier` + the article's `gap_type`. **Do not create new tags via a separate call** — BC creates them on first use when the post is created.

### Shopify

Shopify has multiple **blogs** (channels). Pick one:
- `GET /admin/api/{v}/blogs.json` returns `blogs[]`.
- If `secrets.json:shopify.target_blog_id` is set → use it.
- Else if exactly one blog exists → use it.
- Else if a blog named `News`, `Blog`, or `Articles` exists → use the first match.
- Else → abort and tell the user: "Multiple Shopify blogs exist; set `shopify.target_blog_id` in secrets.json to one of: <list>."

Cache the chosen `blog_id`.

### Adobe Commerce

Adobe (Magento 2) **core** has CMS Pages — no native blog. Three sub-cases:
1. **CMS Pages mode (default).** Push articles as CMS Pages with `active = false` (draft). URL key = article slug. Endpoint: `POST /rest/V1/cmsPage`.
2. **Magefan blog mode.** If `secrets.json:adobe.blog_module = "magefan"` → push to `POST /rest/V1/magefan-blog/posts` with `is_active = 0`. Categories via `GET /rest/V1/magefan-blog/categories`.
3. **Mageplaza blog mode.** If `secrets.json:adobe.blog_module = "mageplaza"` → push to `POST /rest/V1/mpblog/posts` with `enabled = 0`. Categories via `GET /rest/V1/mpblog/categories`.

If `blog_module` is unset → default to CMS Pages mode and add a `notes` entry in the deliverable: "Adobe target was CMS Pages — set `adobe.blog_module` if you have a blog extension installed."

For whichever mode is chosen, fetch the existing categories list and cache it.

### Resulting cache

```json
{
  "platform": "bigcommerce|shopify|adobe",
  "host_target": { /* blog_id, or category list, or CMS namespace */ },
  "existing_categories_or_tags": ["..."],
  "existing_slugs": ["..."]   // see Step 3
}
```

Persist this as `Pillar-Content-Architecture/Phase-2-Content/.publish-cache.<platform>.json`.

---

## STEP 3 — Pre-push: slug collision check

For every article in the queue, build its target slug and confirm it doesn't already exist on the platform.

| Platform | List endpoint | Match field |
|---|---|---|
| BigCommerce | `GET /v3/content/blog/posts?limit=250&page=N` (paginate) | `url` (path) — derive slug from path |
| Shopify | `GET /admin/api/{v}/blogs/{blog_id}/articles.json?limit=250&page_info=...` (paginate) | `handle` |
| Adobe (CMS Pages) | `GET /rest/V1/cmsPage/search?searchCriteria[filter_groups][0][filters][0][field]=identifier&searchCriteria[filter_groups][0][filters][0][value]=<slug>` | `identifier` |
| Adobe (Magefan/Mageplaza) | `GET /rest/V1/<module>/posts?searchCriteria[filter_groups][0][filters][0][field]=identifier&...` | `identifier` |

For each article slug:
- **Exact match exists** → mark `blocked` with reason `"slug collision: <slug> already exists as platform_id <id>"`. Do not push.
- **No match** → cache the slug as available.

If the user passes `--rename-on-collision`, append a numeric suffix `-2`, `-3`, ... and update `pillar-deliverable.json` to reflect the new slug. Off by default.

Also save the full existing-slug list to the cache file for offline link verification in Step 4.

---

## STEP 4 — Pre-push: link integrity

For each article in the queue, parse the HTML and verify all links resolve.

### Internal links

- Strip the article's HTML, extract all `href`s starting with `/` or relative paths or matching the brand domain.
- For each internal href, check it resolves to one of:
  - Another article in this run's queue (forward reference is fine — the target will be created)
  - An article that already exists on the platform (use the existing-slug list from Step 3)
  - A product/category URL — verify by:
    - **BC:** `GET /v3/catalog/products?keyword=<slug>` or `GET /v3/catalog/categories?keyword=<slug>`
    - **Shopify:** `GET /admin/api/{v}/products.json?handle=<slug>` and `/admin/api/{v}/custom_collections.json?handle=<slug>` and `/admin/api/{v}/smart_collections.json?handle=<slug>`
    - **Adobe:** `GET /rest/V1/products/<sku>` (if SKU-based) or `GET /rest/V1/categories/list?searchCriteria[...]=<slug>`
- If a product/collection URL doesn't resolve → mark the article `blocked` with reason `"internal link not found: <href>"`.

### External links

- For each `href` matching `https?://`, do a `HEAD` request with `--max-time 10`.
- Status 200-399 → pass.
- Status 4xx/5xx or timeout → mark the article `blocked` with reason `"external dead link: <href> (HTTP <code>)"`.

### Anchor sanity

- No `href=""`, no `href="#"` (other than legitimate jump links to in-page anchors that exist as `id="..."`).

If any link check fails for an article → block it; do not push. Continue with next article.

---

## STEP 5 — Push (sequential, single article at a time)

For each article in the queue that passed Steps 1-4:

### Update state to "checking" → "pushing"

```bash
jq --arg p "$PLATFORM" --arg a "$SLUG" \
   '.stages.phase_3[$p][$a] = (.stages.phase_3[$p][$a] // {}) + { status: "checking", attempts: ((.stages.phase_3[$p][$a].attempts // 0) + 1) }
    | .updated_at = (now | todate)' \
   pipeline-state.json > /tmp/ps.tmp && mv /tmp/ps.tmp pipeline-state.json
```

### Build the platform-specific payload (DRAFT MODE ENFORCED)

#### BigCommerce — `POST /v3/content/blog/posts`

```json
{
  "title": "<title>",
  "url": "/<slug>/",
  "body": "<rendered HTML from Phase 2>",
  "summary": "<meta description>",
  "tags": ["<tag1>", "<tag2>"],
  "is_published": false,
  "meta_description": "<meta description>",
  "meta_keywords": "<comma-separated keywords>"
}
```

The `is_published: false` field is **mandatory and must not be removed**. After receiving the response, **assert** `data.is_published === false` — if it's true (which would only happen if the field was dropped or the platform default flipped), immediately call `PUT /v3/content/blog/posts/{id}` with `is_published: false` and surface a hard error.

#### Shopify — `POST /admin/api/{v}/blogs/{blog_id}/articles.json`

```json
{
  "article": {
    "title": "<title>",
    "handle": "<slug>",
    "body_html": "<rendered HTML>",
    "summary_html": "<meta description>",
    "tags": "<tag1>, <tag2>",
    "published": false,
    "published_at": null
  }
}
```

`published: false` AND `published_at: null` together — Shopify treats either as draft, but setting both is belt-and-braces. After response, assert `article.published_at === null`.

#### Adobe Commerce — CMS Pages mode

`POST /rest/V1/cmsPage`

```json
{
  "page": {
    "identifier": "<slug>",
    "title": "<title>",
    "content": "<rendered HTML>",
    "content_heading": "<title>",
    "page_layout": "1column",
    "active": false,
    "meta_title": "<title>",
    "meta_description": "<meta description>",
    "meta_keywords": "<comma-separated>"
  }
}
```

Assert response `active === false`.

#### Adobe Commerce — Magefan blog mode

`POST /rest/V1/magefan-blog/posts`

```json
{
  "post": {
    "title": "<title>",
    "identifier": "<slug>",
    "content": "<rendered HTML>",
    "short_content": "<meta description>",
    "is_active": 0,
    "meta_title": "<title>",
    "meta_description": "<meta description>"
  }
}
```

Assert `is_active === 0`.

#### Adobe Commerce — Mageplaza blog mode

`POST /rest/V1/mpblog/posts`

```json
{
  "post": {
    "name": "<title>",
    "url_key": "<slug>",
    "post_content": "<rendered HTML>",
    "short_description": "<meta description>",
    "enabled": 0
  }
}
```

Assert `enabled === 0`.

### Send the request (single, sequential)

```bash
curl -sS -X POST "<endpoint>" \
  -H "Content-Type: application/json" \
  -H "<auth header>" \
  -d @/tmp/payload-$SLUG.json \
  -w '\n%{http_code}\n' \
  -o /tmp/resp-$SLUG.json
```

- HTTP 200/201 → continue.
- HTTP 422 (validation) → log the response body, mark article `failed`, continue with next.
- HTTP 429 (rate limit) → wait 30s, retry once. If it fails again, mark `failed` and continue.
- HTTP 5xx → wait 10s, retry once. If it fails again, mark `failed` and continue.

### Verify draft state on the platform

After a 200/201, immediately do a **read-back** of the just-created resource:

| Platform | Read endpoint | Field to assert |
|---|---|---|
| BC | `GET /v3/content/blog/posts/{id}` | `data.is_published === false` |
| Shopify | `GET /admin/api/{v}/articles/{id}.json` | `article.published_at === null` |
| Adobe CMS | `GET /rest/V1/cmsPage/{id}` | `active === false` |
| Adobe Magefan | `GET /rest/V1/magefan-blog/posts/{id}` | `is_active === 0` |
| Adobe Mageplaza | `GET /rest/V1/mpblog/posts/{id}` | `enabled === 0` |

If the assertion fails → immediately PATCH/PUT the resource back to draft state and surface a hard warning. Mark `draft_confirmed: false` in state until the patch succeeds.

If assertion passes → set `status: "pushed"`, `draft_confirmed: true`, save `platform_id` and `platform_url` to state.

### Rate-limit pacing

After every push, sleep:
- BC: 1s (BC enforces 4-30 req/s depending on plan)
- Shopify: 1s (REST limit 2 req/s baseline; bursts of 40)
- Adobe: 1s (admin API has no documented hard limit, conservative)

---

## STEP 6 — Per-platform deliverable JSON

After the queue is fully processed, write `Pillar-Content-Architecture/Phase-2-Content/.publish-deliverable.<platform>.json`:

```json
{
  "schema_version": "1.0",
  "platform": "bigcommerce|shopify|adobe",
  "pushed_at": "ISO-8601",
  "scope": "pillar|article|all",
  "scope_value": "<pillar name or slug or 'all'>",
  "include_failed": false,
  "totals": {
    "queued": 0,
    "pushed": 0,
    "blocked": 0,
    "failed": 0,
    "skipped_failed_phase_2": 0
  },
  "host_target": { /* cached blog_id / category info */ },
  "articles": [
    {
      "slug": "<slug>",
      "title": "<title>",
      "type": "hub|spoke",
      "pillar_slug": "<pillar slug>",
      "status": "pushed|blocked|failed",
      "platform_id": "string|null",
      "platform_url": "string|null (admin URL for the draft)",
      "draft_confirmed": true,
      "blocked_reason": "string|null",
      "http_status": 200,
      "attempts": 1
    }
  ],
  "next_steps": [
    "Review each draft URL in the platform admin.",
    "Manually toggle to 'Published'/'Active' once approved.",
    "Phase 3 cannot publish live — that step is intentionally manual."
  ]
}
```

---

## STEP 7 — Final report to user

Single short message:

```
Phase 3 pushed to <platform>:
- Queued: <n>   Pushed: <n>   Blocked: <n>   Failed: <n>
- All pushes confirmed in DRAFT mode.

Drafts:
  - <title>  →  <platform_url>
  - <title>  →  <platform_url>
  ...

Blocked (resolve before re-running):
  - <slug>: <blocked_reason>
  ...

Deliverable: Pillar-Content-Architecture/Phase-2-Content/.publish-deliverable.<platform>.json

NOTE: All articles are drafts. Toggle to Published manually in the platform admin after review.
```

---

## FAILURE RECOVERY

- **Resume after kill:** state JSON tracks per-article `status`. Re-invoking with the same scope skips articles already `pushed`. Articles in `blocked` are re-checked from Step 3 (so a fixed external link will unblock them next run). Articles in `failed` are retried up to 3 total attempts before staying failed.
- **Slug collision after a partial run:** if the previous run pushed `vape-kits-guide` and you re-run, Step 3 sees the existing slug and blocks. To fix: either rename the new article (`--rename-on-collision`) or delete the platform draft.
- **Token expired mid-run:** Step 5 retries detect 401 and abort the run. Re-issue the token, update env/secrets, re-invoke; resume picks up where it left off.
- **Draft assertion fails after PATCH retry:** treat as a hard incident — mark `draft_confirmed: false`, log the platform_id, and tell the user explicitly: "Article <slug> created on <platform> as id <id> but failed to confirm draft state. Manually verify in admin and unpublish if necessary." Do not continue to next article until the user acknowledges or fixes.

---

## NOTES (design rationale)

- **Why draft-only is enforced in code, not policy:** a single mistaken "published live" can flood a real store with unreviewed content. Two layers — explicit field on every payload + read-back assertion — make it hard for the model to accidentally bypass.
- **Why we do read-only probe calls for credentials:** a write probe (e.g. POSTing a test article) leaves litter in the store. Read probes verify scope without side effects.
- **Why one article at a time:** rate limits aside, sequential pushes give clean per-article state in the JSON. If push #7 fails we know exactly what was pushed and what wasn't, instead of having to reconcile a batch response.
- **Why we cache existing slugs upfront instead of checking each one with a GET-by-slug:** for queues of 50+ articles a single paginated list is cheaper than 50 individual GETs.
- **Why Adobe Commerce defaults to CMS Pages:** it's the only universally-available content surface in Magento 2 core. Blog modules vary by store.
- **What's NOT in scope for Phase 3:** image upload, product associations, redirect rules, sitemap regeneration. Add later if needed; this skill is purely "push HTML body + metadata as draft."
