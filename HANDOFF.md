# SiteOne Item Enrichment POC — Agent Handoff

**Last updated:** Apr 19, 2026 — **STEP 32-3a COMPLETE.** New top-level variable `Init - Google API Calls Total` (Integer, init 0, var name `google_api_calls_total`) added alongside existing cost tracker variables. Foundation for 32-3 cost tracker wiring of the Gemini lifestyle image generation path. Previous: STEP 32-2 COMPLETE (end-to-end save working, Brilliance validated). Lifestyle image generation via Gemini Nano Banana 2 (gemini-3.1-flash-image-preview) fully functional end-to-end: Filter array → Compose ref URL → Condition → HTTP download → HTTP Gemini → Parse JSON → OneDrive save all working. Y14 RESOLVED: Gemini returns a single-element `parts` array (image at `[0]`, not `[1]` as originally assumed). Fix = `coalesce(parts[0].inlineData.data, parts[1].inlineData.data)` in base64ToBinary expression. `Compose - Gemini Debug` still in flow, slated for removal before production. Remaining Step 32 work: 32-3b Increment action, 32-3c Compose - Cost Metadata update, 32-3d HTML template cost breakdown update, 32-4 HTML preview lifestyle section, 32-5 Intermatic validation. Y12 image model swap to GPT-4.1 done. Step 31 email body redesign shipped.
**User:** Tony — Governance & Data Quality, SiteOne Landscape Supply

---

## 1. Project in one paragraph

Build a Power Automate workflow that takes an incoming "new item setup" email (brand + MFG part #) and produces a fully enriched PIM-ready record. Pipeline: discover product URLs via search → scrape manufacturer + 2 distributors → extract structured specs → find similar SiteOne items for grounding → generate descriptions/bullets → validate → output CSV for PIM upload. Accuracy is paramount. Low-confidence items flagged for human review. Volume: **10–50 items/day**.

---

## 2. Hard constraints

- **Tools available:** Power Automate + AI Builder ONLY. No Azure OpenAI, no Claude API, no custom code hosting.
- **External API in use:** Serper.dev (Google SERP results) — called via HTTP action.
- **Human review model:** Confidence-based flagging (not full review, not no review).
- **Bing Search API is DEAD** (retired Aug 2025). Don't suggest it.
- **Microsoft's "Grounding with Bing" replacement requires Azure AI Foundry** — Tony doesn't have access.

---

## 3. Full 8-stage architecture (target end state + current progress)

1. **Intake** — email trigger → AI Builder extracts brand/part#/name  ← ✅ **DONE Step 30**
2. **Source discovery** — Serper search → AI picks best URLs  ← ✅ **DONE (Confidence-first Step 27a)**
3. **Harvest** — Firecrawl for static + JS-rendered pages, conditional image agent for base64-embedded sites  ← ✅ **DONE Steps 27-27g**
4. **Extraction** — AI Builder prompt: markdown → structured JSON with per-field confidence + source citations  ← ✅ **DONE Step 27e**
5. **Catalog grounding** — Dataverse lookup for similar SiteOne items, used as AI prompt grounding  ← ❌ NOT STARTED (deferred — no Dataverse setup yet)
6. **Content generation** — long description + feature bullets, grounded in extracted specs  ← ✅ **DONE Step 22**
7. **Validation gate** — rule-based checks + LLM-critic second pass + confidence threshold gating  ← 🟡 PARTIAL (per-field confidence captured, confidence-first URL selection gates, but no explicit "validation gate" step or LLM critic yet)
8. **PIM file creation** — CSV/Excel for upload + HTML preview for review  ← ✅ **DONE Steps 23, 26, 28**

**Bonus infrastructure completed:**
- Cost tracker + 3-KPI HTML banner (Confidence / Duration / Cost) — Step 28
- Per-run correlation_id folder isolation — Step 29
- Email-triggered workflow end-to-end — Step 30

**Five accuracy techniques that matter most:**
1. Always ground with source text (never let GPT recall from memory) ✅ enforced in Extract prompt
2. Force structured JSON output with explicit schema ✅ all 5 prompts use strict JSON + Parse JSON schemas
3. Per-field confidence scores from model itself ✅ in Extract + Image agent + URL selection
4. LLM-critic second pass (one extracts, another verifies) ❌ NOT BUILT — future work
5. Store source HTML/URL per field for audit ✅ source_url, source_domain, scope_reasoning captured

---

## 4. POC scope (CURRENT STATE)

**POC is now functional end-to-end.** Email-triggered, fully autonomous enrichment with HTML preview delivered back to requester.

**Validated working:**
- Intermatic DT200LT — manufacturer path (FALSE image agent branch, text extraction finds 7 images natively)
- Brilliance BRI-WIFI-SMART-SOCKET-3 — distributor fallback path (TRUE image agent branch, 3 real images @ 98% from Shopify base64-embedded HTML)
- End-to-end email flow: Email → AI extract → enrichment pipeline → HTML reply with attachment

**Known failure modes (all documented):**
- Serper non-determinism (Y4) — same product can pick different distributors run-to-run
- Brilliance run went through Sprinkler Supply Store (TRUE branch) in one run, Lighting Warehouse (FALSE branch, natively worked) in another
- This is expected and production-correct — flow adapts to whichever URL Serper surfaces

---

## 5. Current flow state — `POC - Product Page Harvester`

**Trigger:** `When a new email arrives (V3)` (Office 365 Outlook) — monitors a dedicated Outlook folder populated by an inbox rule (routes emails with "onboard" in subject).

**Settings on trigger:**
- Split On: **ON** (critical — V3 returns array; Split On iterates one email per run)
- Array field: `@triggerOutputs()?['body/value']`
- Concurrency Control: OFF

**Outlook setup (manual, outside flow):**
- Folder: `Item Setup Requests` (sub-folder under Inbox)
- Rule: subject contains `onboard` → move to `Item Setup Requests` folder
- Future: expand rule to match `item setup`, `enrich`, `new item`, `setup request`

**High-level flow structure (action count: 50+):**

```
1. Trigger: When email arrives (V3)
2. AI - Extract from Email          ← NEW Step 30
3. Parse JSON - Email Extraction    ← NEW Step 30
4. Check - Valid Email Extraction (Condition)  ← NEW Step 30
   ├── FALSE: Send rejection email → Terminate
   └── TRUE: (empty; flow falls through — see #47 in backlog)
5. Init - Correlation ID (guid())
6. Init - Run Timestamp (utcNow())
7. Init - Brand  (reads from Parse JSON Email Extraction)
8. Init - Part Number  (reads from Parse JSON Email Extraction)
9. Init - Search Query  (concat quotes + part + brand)
10. Init - Item Folder Path (includes correlation_id subfolder — Step 29)
11. Init - Image Counter (top level per PA rules)
12. Init - AI Builder Credits Total (cost tracker Step 28)
13. Init - Firecrawl Credits Total
14. Init - Serper Searches Total
14b. Init - Google API Calls Total (Step 32-3a — Gemini lifestyle cost line)
15. HTTP (Serper)
16. Increment - Serper Searches (+1)
17. Parse JSON (Serper response)
18. AI - Select URLs  (confidence-first URL selection, Step 27a)
19. Increment - AI Credits (Select URLs)
20. Parse JSON 1
21. Check - URL Qualifies (Condition, Step 27c)
    ├── FALSE: Terminate - Flag for Human Review
    └── TRUE:
        22. Firecrawl - Scrape Product Page (waitFor: 5000)
        23. Increment - Firecrawl Credits (Primary)
        24. AI - Extract Product Data
        25. Increment - AI Credits (Extract Data)
        26. Parse JSON - Extraction Output
        27. Check - Image Coverage (Condition, Step 27g-2 — fires when images < 2)
            ├── TRUE (image agent fires):
            │   28. Firecrawl - Scrape for Images (excludeTags: [script,style,noscript,iframe,link,meta,head], onlyMainContent: false)
            │   29. Increment - Firecrawl Credits (Images)
            │   30. AI - Extract Product Images (GPT-5 currently, should be GPT-4.1 — Y12)
            │   31. Increment - AI Credits (Image Agent)
            │   32. Parse JSON - Image Agent Output
            │   33. Compose - Merged Extraction (setProperty splices agent images in)
            └── FALSE (pass through):
                34. Compose - Passthrough
        35. AI - Generate Content (B2B SiteOne voice rewriting)
        36. Increment - AI Credits (Generate Content)
        37. Parse JSON - Generated Content
        38. AI - Flatten Row (flat JSON → CSV-ready)
        39. Increment - AI Credits (Flatten Row)
        40. Parse JSON - Flat Row
        41. Create CSV Table
        42. OneDrive - Save CSV
        43. Compose - Images to Download (coalesce merged vs passthrough)
        44. Loop - Download Images (concurrency 1)
            44a. Increment - Image Counter
            44b. HTTP - Download Image
            44c. OneDrive - Save Image
        45. Compose - Cost Metadata (Step 28)
        46. Compose - Extraction Metadata
        47. Compose - HTML Preview (template v4 with 3-KPI banner)
        48. OneDrive - Save HTML Preview
        49. Send Email - Reply with HTML  ← NEW Step 30-6
```

**See RECOVERY.md for the exact expressions, schemas, prompts, and configuration of every action.**

---

## 6. Hardening roadmap — status tracker

**Legend:** ✅ done · 🟡 in progress · ❌ not started · ⏸ deferred

### CRITICAL (must do before production)
- ✅ #1 — Hardcoded brand/part# → dynamic trigger inputs
- ✅ #2 — Hardcoded Serper query → variable-driven
- ❌ #3 — Error handling with Scope blocks (Try/Catch pattern)
- ❌ #4 — Retry policy + timeout on HTTP action
- ❌ #5 — API key → environment variable (currently plaintext in action)
- ❌ #6 — Filter Serper JSON before AI (token savings ~60%)

### IMPORTANT (address before scaling)
- ✅ #7 — Correlation ID
- ❌ #8 — Logging to SharePoint/Dataverse (governance-critical for Tony)
- ❌ #9 — Externalize exclusion list (currently in prompt text)
- ❌ #10 — URL liveness check (HEAD request) before scraping
- 🟡 #11 — B2B fallback in prompt (partial fix drafted, not applied — see Step 6 Part F proposal)
- ⏸ #12 — Modular child flows (defer until POC end-to-end works)

### NICE-TO-HAVE
- ❌ #13 — Cost tracking per item
- ❌ #14 — Deduplication check
- ❌ #15 — Smarter Serper query (exclude domains in query)
- ❌ #16 — Confidence threshold gating branch
- ❌ #17 — Prompt versioning / source control export
- ❌ #18 — **Fallback search strategy using product_name** (Tony's addition) — confirmed needed by Step 10 Brilliance test. **Design decisions captured:** trigger = mfg null AND zero distributors; query = additive `"part#" brand product_name`; empty product_name = terminate + flag; combine = AI Compare picks better. **Compare criteria deferred as future enhancement.** Deferred entirely to get to actual extraction first.
- ✅ #19 — **siteone.com excluded** (Step 11). Verified: Brilliance re-test now kicks out siteone.com with correct reason citation.
- ❌ #20 — **"Better result" criteria for AI Compare prompt** (new) — needs formal design before building Compare prompt.
- ❌ #21 — **Pass Firecrawl metadata to extraction prompt** as separate input so ogImage/ogTitle fields populate `images` array properly.
- ❌ #22 — **Multi-source field fill-in** — merge logic for when distributor scrapes have UPC/SKU that manufacturer page lacks.
- ✅ #23 — **Confidence-first URL selection (DONE Step 27a-c)** — Tony's revised design: manufacturer-first ALWAYS. Only fall through to distributor if no manufacturer URL meets 95% threshold. Single URL scraped per product (no multi-source merge for POC). AI prompt now returns `selected_url`, `confidence`, `source_type` (manufacturer|distributor|none), `source_domain`, `reasoning`, and full `candidates_considered` audit trail. Validated: Intermatic → manufacturer path, Brilliance → distributor fallback at Sprinkler Supply Store.
- ✅ #24 — **Null URL guard (DONE Step 27c)** — Condition action `Check - URL Qualifies` checks `body('Parse_JSON_1')?['selected_url']` is not null. True branch contains all 14 downstream actions. False branch has Terminate action (`NoQualifyingURL` code, Failed status) with human-readable error message including brand, part#, AI reasoning, and correlation_id.
- ❌ #25 — **Upgrade CSV output to true XLSX via Office Scripts** — Current POC uses CSV (opens in Excel, fine for POC). For production polish, build an Office Script in a blank workbook that accepts a JSON payload and writes dynamic columns/rows, then invoke via "Run script" action. **Full implementation guide now in RECOVERY.md Appendix X** — script code, flow actions, and common pitfalls documented.
- ❌ #26 — **HTML preview generation** — Create an HTML template that renders extracted data as a product page preview. Save to OneDrive alongside CSV. Per Tony's requirement Apr 17.
- ❌ #27 — **Image download + OneDrive storage** — HTTP GET each image URL from extraction → save to `<item_folder_path>/images/`. Needs Apply to each loop over `extracted_data.images` array. Per Tony's requirement Apr 17.
- ❌ #28 — **Lifestyle image generation via Runware** — Use extracted product images as input to Runware img2img API → generate lifestyle/contextual images. Requires Runware API key + new HTTP action. Per Tony's requirement Apr 17.
- ❌ #29 — **Email trigger replacement** — Swap manual trigger for Outlook "When a new email arrives" + AI Builder email parsing to extract brand/part#/product_name. Per Tony's requirement Apr 17.
- ❌ #30 — **Migrate flow ownership to service account** — Currently runs under Tony's credentials (OneDrive, AI Builder, API keys all tied to Tony). For production/shared use, provision a SiteOne service account (e.g., `itemsetup@siteone.com`), make it flow owner, reauthenticate all connections as that account. Critical before scaling to team use. Needs IT involvement to provision account.
- 🟡 #31 — **Complete image capture for JS-rendered / base64 galleries (IN PROGRESS — conditional image agent design locked Apr 18)** —
  - **Root cause diagnosed on Brilliance (Apr 18):** Sprinkler Supply Store uses Shopify's theme pattern that embeds main product images as inline base64 data URIs. Firecrawl strips these by design (`<Base64-Image-Removed>` placeholder in markdown). The markdown-only extraction path NEVER sees real image URLs for these sites.
  - **Earlier diagnosis wrong:** Initially attributed to lazy-load. Raw HTML inspection confirmed base64 embedding is the primary issue on Shopify; `cdn.shopify.com` CDN URLs exist in the HTML for related-products but primary images are base64.
  - **Attempted solutions and why rejected:**
    - Option 1 (flip `onlyMainContent: false`) — would flood extract prompt with noise and regress text quality
    - Option 2 (dual-input to existing extract prompt — markdown + html) — token blowup; Firecrawl HTML is 200-500KB on Shopify sites, exceeded GPT-4.1 context when tested in Prompt Builder
    - Path 1 (separate image prompt, always-on) — still hits HTML size limits; doubles Firecrawl cost on every run even when not needed
    - Parallel branch — wasteful on sites where text extraction already works
  - **LOCKED design (Path C — conditional fallback):** Dedicated image subsystem fires ONLY when text extraction returns fewer than 2 images. Isolation principle. Zero cost overhead on sites that work naturally (Intermatic, Rain Bird, likely all manufacturer domains). Fires on Sprinkler Supply Store / base64-embedding sites.
  - **v1 scope locked:**
    - Coverage: HTML parsing + attribute preference (src → data-src → srcset → data-zoom → etc.) + scope rules (primary product only, no related products)
    - Expected coverage: 80-85% of sites (Shopify, Magento, WooCommerce)
    - Out of scope v1: Firecrawl `actions` for JS-injected galleries, JSON-LD structured data parsing, cross-source fallback (if selected URL has bad images, try another). These are v2.
    - Quality bar: governance-grade output — URLs + alt + classification + dedup + resolution ranking + scope + per-image confidence + audit trail
    - Input: same `selected_url` as text path (no independent URL selection)
  - **Build plan:** Steps 27g-1 through 27g-9 (conditional architecture, new Firecrawl call with `excludeTags` to bound HTML size, new prompt, merge/passthrough Compose actions, test on Intermatic FALSE path + Brilliance TRUE path)
  - **Priority:** IMPORTANT — blocks production-quality output for distributor-fallback cases.
- ❌ #43 — **Image agent v2 enhancements (future after v1 ships)** — once v1 is in production and we have failure data:
  - Firecrawl `actions: [{type: "scroll"}, {type: "wait"}]` for JavaScript-injected galleries beyond static HTML
  - JSON-LD structured data parsing for SEO-heavy sites
  - Cross-source fallback — if selected URL yields thin images, automatically re-run image agent against next-highest-confidence candidate from `candidates_considered`
  - Per-distributor config / few-shot examples based on observed patterns
- 🟡 #44 — **Production-grade state management (MINIMUM-VIABLE FIX DONE Step 29 Apr 18; future hardening deferred)** — Tony's stated pain points (having to manually delete folders between runs, 409 eTag collisions on same-name writes) addressed via single change:
  - **Step 29-1 DONE:** `item_folder_path` variable updated to include `correlation_id` subfolder. Target structure: `/Item Setup POC/<Brand>_<Part>/<correlation_id>/`. Every run writes to unique folder, no collisions.
  - **Step 29-2 SKIPPED:** OneDrive "If file exists: Replace" parameter doesn't exist in Tony's connector version. Belt-and-suspenders unnecessary since 29-1 isolates runs by correlation_id — no file-name collisions possible within a single run.
  - **Remaining (deferred, not blocking):**
    - B) Atomic write pattern (scratch subfolder → promote on success)
    - D) Delete-before-write fallback (for connectors that support it, or via separate Delete File action)
    - E) Idempotency check (skip if already enriched)
    - F) Cleanup on failure (delete partial artifacts)
    - OneDrive folder cleanup policy (runs accumulate over time at ~1,500/month at 50 items/day × 30 days)
  - **Priority of remaining:** LOW for POC/validation; become relevant at production scale (months of operation).
- ❌ #45 — **Multi-item email handling (DEFERRED from Step 30 v1 — Tony flagged as eventual requirement)** — v1 Step 30 handles one item per email. Production will see emails with multiple items: *"Please onboard Intermatic DT200LT, Rain Bird 5004PC, and Hunter PGP."* Build requirements for v2:
  - AI parsing prompt extracts ARRAY of brand/part# pairs (not single item)
  - Parent flow fans out to N parallel child-flow invocations
  - Response aggregation — collect all child flow outputs, build unified reply with multiple HTML attachments OR multi-item summary
  - Partial failure handling — one item fails extraction, others still succeed
  - Correlation ID strategy — parent ID + child sub-IDs for audit trail
  - Email threading — do we reply once with all results, or N replies (one per item)?
  - **Priority:** IMPORTANT post-v1. Production users WILL send multi-item requests; single-item-only will feel broken.
- ❌ #46 — **Attachment parsing for email requests (DEFERRED from Step 30 v1 — Tony flagged as eventual requirement)** — Many item-setup requests arrive as forwarded vendor emails with datasheets, spec sheets, or price sheets attached. v1 reads email body only; attachments ignored. Build requirements for v2:
  - Detect email attachments via Outlook trigger metadata
  - Route by attachment type:
    - PDF datasheets → PDF text extraction (AI Builder has a "Extract from PDF" prompt primitive) → feed into existing parsing prompt
    - Excel/CSV product lists → read cells, extract brand/part rows
    - Images/screenshots of vendor portals → OCR → text extraction → parsing
    - Word docs → similar text extraction
  - Merge attachment content with email body text for unified AI parsing input
  - Respect attachment size limits (Outlook message size caps, AI prompt context limits)
  - Handle encrypted/password-protected PDFs gracefully (skip, note in reply)
  - **Priority:** IMPORTANT post-v1. Many vendor workflows send specs ONLY as attachments, not inline. Without this, users have to manually retype brand/part# from the attachment into the email body.
- ❌ #47 — **Move existing actions inside `Check - Valid Email Extraction` True branch (PRODUCTION HARDENING — deferred from Step 30-4)** — In Step 30-4, for speed, we kept the existing 35+ downstream actions in the flow's main sequence AFTER the Condition rather than moving them all into the True branch. This works functionally because the False branch has a Terminate action that halts the entire flow — so downstream actions don't run when extraction fails. **BUT:**
  - Architecturally messy. Reading the flow in the designer, it looks like downstream actions might run regardless of the Condition. Future maintainer has to trace the Terminate to understand the actual control flow.
  - Same pattern as `Check - URL Qualifies` (Step 27c) where we DID move everything into True branch — so this creates inconsistency in the flow architecture.
  - No functional difference today; production deployment should harden by moving actions into the True branch for clarity and to prevent future bugs when someone adds a new action assuming the Condition actually scopes execution.
  - **Fix approach:** Drag all actions from "below the Condition" into the Condition's True branch. Same tedious drag operation we did for Check - URL Qualifies. ~20-30 minutes.
  - **Priority:** LOW — pre-production cleanup. Doesn't affect functionality; affects code clarity / maintainability.
- ❌ #32 — **Mexico subdomain / URL normalization** — Serper non-determinism sometimes returns `mx.intermatic.com` instead of `www.intermatic.com`. Mexico site has sparser content. Fix in AI URL selection prompt: prefer US English domains, exclude non-US locale subdomains, or normalize URLs post-selection.
- ❌ #33 — **Firecrawl cache busting** — Firecrawl aggressively caches on URL; `cacheState: "hit"` can return stale content. No clean cache-bust param in v2/scrape. Workaround: append timestamp query param to URL before scraping.
- ❌ #34 — **Image file extension detection** — Currently hardcoded `.jpg` for all image downloads. Should detect Content-Type from HTTP response header, map to proper extension (.png, .webp, etc.).
- ❌ #35 — **Pre-run folder cleanup / 409 eTag handling** — OneDrive returns 409 eTag mismatch when a target file exists from a prior failed run. Current workaround: manually delete folder in OneDrive before test. Production fix: Add Delete File action before write, or use Update File with Overwrite behavior.
- ❌ #36 — **Excel CSV date auto-format** — When opening CSV in Excel, strings like `"5-15"` (plug configuration value) get auto-parsed as `15-May` date. Not a flow bug, just Excel display. Can be avoided by opening CSV via Power Query or importing into PIM. Minor cosmetic issue.
- ❌ #37 — **Image deduplication** — Some products expose the same image at multiple resolutions (e.g., `/preview/` and `/thumbnail/` folders). Currently captures both as separate rows. Could dedupe by filename stem.
- ❌ #38 — **Document asset download** — Currently HTML preview will show links to PDFs/datasheets/sell sheets/instructions. These should be downloaded to OneDrive (same pattern as image download loop). Deferred for now to focus on HTML preview first.
- ✅ #39 — **Cost & time metrics tracking (COMPLETE Step 28, validated on Intermatic Apr 18)** — Full instrumentation live:
  - 3 accumulator variables initialized at flow start: `ai_builder_credits_total`, `firecrawl_credits_total`, `serper_searches_total`
  - 8 Increment actions wired (Serper after HTTP, Firecrawl after primary scrape + conditional image scrape, AI Builder after all 4-5 prompt calls)
  - `Compose - Cost Metadata` builds unified object with credits, estimated $, duration, image_agent_fired flag, disclaimer
  - `compose_template_v4.html` deployed with:
    - Three-KPI banner at top: Confidence % / Run Duration / Cost to Execute (prominent, Tony's explicit requirement)
    - Cost Breakdown section before footer with itemized AI Builder / Firecrawl / Serper credits + $ + disclaimer
    - Expanded footer metadata grid with Run Start / Run End / credit totals / image_agent status / total cost
  - Conversion rates baked in (approximate, disclaimer labeled):
    - AI Builder: $0.001/credit
    - Firecrawl: $0.003/credit (verified 2026 Standard plan pricing)
    - Serper: $0.0003/search (verified 2026 pricing — was corrected from original $0.002 estimate)
  - Validated Intermatic run: 456 AI Builder credits + 1 Firecrawl + 1 Serper = $0.46 estimated total
  - **Remaining polish (deferred, optional):** Environment variables for conversion rates so they don't require template redeploy to update
- ✅ #40 — **Defensive JSON parse hardening (DONE Step 27 side-quest)** — All 4 AI Builder prompts hit a markdown code-fence issue: model occasionally wrapped JSON in ` ```json ... ``` `. Root-caused and fixed two ways: (1) Final directive in every prompt rewritten to `Return ONLY the raw JSON object. Do NOT wrap it in code fences...`. (2) Defensive `replace(replace(replace(..., '```json', ''), '```', ''), decodeUriComponent('%0A'), '')` expression on ALL 4 Parse JSON content inputs strips fences + newlines pre-parse. Belt-and-suspenders — applies to all 4 prompts.
- ❌ #41 — **Image source blacklist (deferred — design only)** — Tony's vision: certain distributor domains (Lowes, HD) give good specs but low-quality/watermarked images unsuitable for SiteOne catalog. Future logic: tag every image with `image_N_source_domain` (✅ DONE Step 27d), then maintain an exclusion list per domain. If the selected URL's domain is on the image-blacklist, trigger a secondary scrape of the manufacturer site (or another approved distributor) specifically for images. Not blocking POC; document pattern now, build logic later.
- ❌ #42 — **Null-URL filter in image loop (DONE reactively Step 27 trouble-shoot)** — Needed because some lazy-load sites yielded images with `url: null`. Loop input changed from raw images array to `filter(body('Parse_JSON_-_Extraction_Output')?['images'], item()?['url'] != null)`. Marking done but leaving as #42 since it was a reactive fix, not a planned hardening item.

---

## 7. Current state (Apr 18, end of session)

**POC is end-to-end functional.** Email-triggered, produces enriched HTML report with cost/confidence/duration KPIs, delivers back to sender as attachment. Validated Intermatic + Brilliance runs.

### Build arc summary

| Step | What was built | Status |
|---|---|---|
| Steps 1-26 | Original build: Serper → URL selection → Firecrawl → text extraction → content generation → flatten → CSV → image download → HTML preview template | ✅ complete (see prior handoff history) |
| Step 27a-c | **Confidence-first URL selection** — new prompt output shape (single selected_url + candidates_considered audit), Check - URL Qualifies Condition, Terminate on null URL | ✅ complete |
| Step 27d | Image source_domain tagging in Flatten prompt | ✅ complete |
| Step 27e | Extract prompt updated with data-src/srcset handling for lazy-load sites | ✅ complete (helpful but not sufficient for base64 sites) |
| Step 27g-1 through 27g-9 | **Image agent** — dedicated image-extraction subsystem. Condition → dedicated Firecrawl call with excludeTags → specialized prompt (`Extract Product Images from HTML`) → Compose merge. Fires only when text extraction returns <2 images. | ✅ complete. Known: prompt currently uses GPT-5, should be GPT-4.1 (Y12). |
| Step 28 (1-9) | **Cost tracker** — 3 accumulator variables, 8 Increment actions, `Compose - Cost Metadata`, HTML template v4 with 3-KPI banner (Confidence/Duration/Cost) + Cost Breakdown section | ✅ complete. Validated Intermatic at $0.46/run, Brilliance ~$2.50 with current GPT-5 image prompt. |
| Step 29 (1-2) | **State management MVP** — per-run correlation_id subfolder in `item_folder_path`. No more manual folder deletes. Step 29-2 (Replace conflict behavior) skipped — not available in Tony's OneDrive connector. | ✅ complete |
| Step 30 (1-8) | **Email trigger** — new `Extract Enrichment Request from Email` prompt (GPT-4.1 mini), Outlook V3 trigger on filtered folder, Parse JSON, Validation Condition with rejection email + Terminate in False branch, Init - Brand/Part Number rewired to read from email extraction, final Send Email action with HTML attachment. | ✅ complete and VALIDATED end-to-end. |
| Step 31 | **Reply email body visual redesign** — replaced plain-text bullets with visual KPI email body: green ✅ success banner + product identity, 3-KPI strip (Confidence with conditional green/amber/red by tier + tier label "Ready for PIM"/"Review recommended"/"Human review required", Duration with image-agent-fired subtitle, Est. Cost with accent styling), existing summary bullets preserved below banner. Email-safe HTML (tables, inline CSS, native emoji, no web fonts). | ✅ complete, validated Apr 19 ("worked perfectly"). |
| Y12 | **Image agent prompt: GPT-5 → GPT-4.1** swap | ✅ done Apr 19 by Tony directly (one dropdown). Expected ~5x cost reduction on TRUE-branch runs. Not re-validated on Brilliance but no regression expected (task is parsing/classification not reasoning). |
| Step 32 (in progress) | **Gemini Nano Banana 2 lifestyle image generation** — Built: `Filter array - Product Images Only` (filters merged images to type=product), `Compose - Reference Image URL` (first URL), `Check - Has Reference Image` Condition, TRUE branch: `HTTP - Download Reference Image` (GET source JPEG), `HTTP - Gemini Lifestyle 1` (POST to gemini-3.1-flash-image-preview:generateContent with prompt + base64 image), `Parse JSON - Gemini Lifestyle 1`, `Compose - Gemini Debug` (diagnostic, remove before prod), `OneDrive - Save Lifestyle Image 1` using coalesce parts[0]/parts[1] pattern. v1 scope: 1 lifestyle image per product. | ✅ 32-2 complete Apr 19 (Brilliance validated, PNG saved). Y14 resolved. Remaining: 32-3 cost tracker, 32-4 HTML preview section, 32-5 Intermatic validation. |

### Validated scenarios

- **Intermatic DT200LT** (manufacturer path): Text extraction finds 7 images natively → FALSE branch of image agent → cost ~$0.46/run, runs cleanly
- **Brilliance BRI-WIFI-SMART-SOCKET-3** (distributor fallback path via Sprinkler Supply Store): Text extraction returns 0 images (Shopify base64) → TRUE branch fires → image agent extracts 3 real product images @ 98% confidence → cost ~$2.50/run with GPT-5
- **Email-triggered end-to-end:** Sent "Onboard Brilliance BRI-WIFI-SMART-SOCKET-3" → flow ran to completion → reply email with HTML attachment arrived

### Known issues that surfaced during validation

1. **Y12 — Image prompt uses GPT-5 not GPT-4.1.** One-dropdown change in AI Builder, 5x cost reduction, minimal accuracy risk. Highest-ROI next task.
2. **Y4 — Serper non-determinism.** Same product can route to different distributors run-to-run. This is production-correct (flow adapts) but makes Brilliance-type testing inconsistent.
3. **#47 — Main flow actions are BELOW the `Check - Valid Email Extraction` Condition, not inside its TRUE branch.** Works functionally because the FALSE branch Terminates the flow, but architecturally messy. LOW priority cleanup.
4. **#44 state management minimum fix only** — future hardening (atomic writes, cleanup on failure, OneDrive subfolder accumulation policy) is backlogged.

### Next priorities (Tony's call — pick order)

1. **Resume Step 32 build** — 32-3 cost tracker wiring (new `google_api_calls_total` var + Increment + Compose - Cost Metadata update) → 32-4 HTML preview section for lifestyle images → 32-5 Intermatic validation → remove `Compose - Gemini Debug` before prod.
2. ~~Y12 model swap~~ ✅ DONE by Tony directly
3. ~~Step 32 save unblock~~ ✅ DONE Apr 19 (Y14 resolved via coalesce expression)
4. **#45 multi-item email handling** (production-critical for real rollout)
5. **#46 email attachment parsing** (PDF datasheets → brand/part# extraction)
6. **Backlog hardening** — Scope blocks, env variables for API keys, retry policies, SharePoint logging
7. **Stage 5 catalog grounding** — Dataverse lookup of similar SiteOne items (never started)
8. **Stage 7 validation gate** — LLM-critic second pass on extraction output (never started)

### Step 32 sub-step checklist (RESUME HERE)

- ✅ 32-1 Prompt template locked (static with {brand} + {product_name} placeholders, explicit "preserve product appearance" anchor, excludes humans/watermarks/text)
- ✅ 32-2a `Filter array - Product Images Only` (after `Loop - Download Images`)
- ✅ 32-2b `Compose - Reference Image URL` (`first(body('Filter_array_-_Product_Images_Only'))?['url']`)
- ✅ 32-2c `Check - Has Reference Image` (Condition on `outputs('Compose_-_Reference_Image_URL')` != null)
- ✅ 32-2d `HTTP - Download Reference Image` (GET, PT60S timeout, in TRUE branch)
- ✅ 32-2e `HTTP - Gemini Lifestyle 1` (POST to gemini-3.1-flash-image-preview, PT120S, retry None, secure inputs ON) — initial `$content` expression error fixed by switching to `base64(body(...))`
- ✅ 32-2f `Parse JSON - Gemini Lifestyle 1` (hardened schema, additionalProperties: true everywhere)
- ✅ 32-2g `OneDrive - Save Lifestyle Image 1` — WORKING after Y14 fix (coalesce parts[0]/parts[1] pattern). Validated on Brilliance Apr 19 — PNG saved successfully.
- 🟡 **Diagnostic Compose still in place** — `Compose - Gemini Debug` sits between Parse JSON and OneDrive save. Useful if we hit another response-shape variant. Remove before production.
- 🟡 32-3 Cost tracker wiring (in progress):
  - ✅ 32-3a `Init - Google API Calls Total` top-level variable added (Integer, `google_api_calls_total`, value 0)
  - ❌ 32-3b `Increment - Google API Calls` action after `HTTP - Gemini Lifestyle 1`
  - ❌ 32-3c Add `google_api_calls` + `estimated_google_usd` + rolled-up `estimated_total_usd` to `Compose - Cost Metadata`
  - ❌ 32-3d Update compose_template_v4.html Cost Breakdown section + footer to show Google API as 4th cost line
- ❌ 32-4 HTML preview section "Generated Lifestyle Images" in compose_template_v4.html (separate section per Tony's choice, not in CSV)
- ❌ 32-5 End-to-end validation on Intermatic run (Brilliance validated Apr 19; Intermatic still pending)

---

## 8. Key decisions log

| Decision | Rationale | When |
|---|---|---|
| Serper.dev over Brave/SerpAPI | Best free tier for volume, Google results quality | Day 1 |
| Single-URL scrape (not multi-source merge) | Complexity vs. accuracy tradeoff for POC; confidence-first logic picks best URL | Step 27 |
| Manufacturer URL ALWAYS wins if ≥95% confidence | Manufacturer data is authoritative; distributor only as fallback | Step 27a |
| AI picks URLs (not domain lookup table) | Flexibility for unknown brands; transparent reasoning via candidates_considered | Early build |
| Confidence-based flagging (not full review) | Scale without sacrificing quality | Day 1 |
| Hardened Parse JSON schemas from the start | Tony pushed back on POC shortcuts | Early build |
| Conditional image agent (not always-on) | Zero overhead on sites that work natively (Intermatic); agent fires only when needed (Brilliance) | Step 27g |
| Dedicated second Firecrawl call for images (not reuse existing) | Isolation — doesn't risk regressing text extraction path | Step 27g |
| GPT-4.1 for image agent (not GPT-5) | Task is parsing + classification, not deep reasoning; 5x cheaper. **Currently set to GPT-5 — needs flip (Y12)** | Step 27g-1 |
| GPT-4.1 mini for email extraction prompt | Smallest model that works; email parsing is simple extraction | Step 30-1 |
| Per-run correlation_id subfolder | Eliminates 409 collisions; natural audit trail | Step 29-1 |
| Email trigger via folder-filtered Outlook rule | Simpler than shared mailbox for POC; clear upgrade path to shared mailbox | Step 30 |
| Defensive JSON fence-stripping on all Parse JSON actions | LLM output isn't 100% deterministic even at temp 0; markdown code fences occasionally wrap JSON | Step 27 side-quest |
| Hardcoded USD conversion rates in Compose (not env var) | POC shortcut; labeled as estimated; upgrade to env var deferred | Step 28-8 |
| Cost tracker VISIBLE in HTML preview (prominent, not footer-only) | Tony's explicit requirement — leadership cost conversations | Step 28 |
| Email body banner stays green ✅ regardless of confidence tier | Flow only reaches email-send action on successful runs (Terminate fires earlier on extraction failure). Banner signals "pipeline completed"; confidence KPI color/tier label carries the confidence signal. | Step 31 |
| Conditional color-coding on Confidence KPI only (not Duration/Cost) | Confidence is the signal Tony cares about for human-review routing. Duration and Cost are informational — no action threshold. | Step 31 |
| Table-based email HTML with inline CSS | Outlook desktop strips `<style>` blocks and doesn't support flexbox/grid. Tables + inline CSS are the only format that renders reliably across Outlook/Gmail/Apple Mail. | Step 31 |
| Native emoji (✅) over inline base64 images | Renders everywhere modern without hosting/embed overhead. Tony explicitly chose this path. | Step 31 |
| Google Nano Banana 2 over Runware for lifestyle images | Nano Banana 2 is multimodal-native (takes reference image + prompt → preserves specific product in new scene). Runware is cheaper but relies on FLUX/SDXL variation which doesn't "understand" what the product is semantically. For true contextual scenes Tony wanted, Nano Banana 2 is the only Google model that works. ~$0.045/image vs. ~$0.0006 Runware, worth it for quality. | Step 32 |
| Filter array → Compose → Condition pattern for reference image selection | Only generate lifestyle if ≥1 product-type image exists. Avoids conditioning Gemini on a cert badge or infographic. `type == "product"` tag was already in extraction prompt output. | Step 32-2 |
| Static prompt template (not AI-generated per product) | Two placeholders only: brand + product_name. Explicitly excludes part numbers (Gemini tries to render them as literal text). "Residential outdoor landscape setting" covers bulk of SiteOne catalog. "Preserve product appearance from reference image" anchor is critical or model drifts. Per-category prompt engineering deferred. | Step 32-1 |
| Lifestyle images in separate `/lifestyle/` subfolder, separate HTML section, NOT in CSV yet | Tony's explicit choice. Keeps generated content distinct from extracted source content for catalog-review clarity. Upgrade to CSV later after PIM field design. | Step 32 |
| v1 scope = 1 lifestyle image per product (not 2) | Tony scaled down mid-build. "If scrape came back with 10 images, I don't want 20 lifestyle images." Conservative for cost + speed; trivial to add 2nd call later. | Step 32 |
| `base64(body(...))` not `$content` for binary base64 in HTTP body | PA rejects the legacy Logic Apps `$content` syntax in expression validator. `base64(body('HTTP_-_Download_Reference_Image'))` converts binary response to base64 string inline. | Step 32-2e |
| Coalesce-based parts index for Gemini image extraction (`parts[0]` OR `parts[1]`) | Gemini's `generateContent` response shape varies even with `responseModalities: ["TEXT", "IMAGE"]` — sometimes 1 part (image only), sometimes 2 parts (text + image). Hardcoding either index breaks the other case. `coalesce(parts[0].inlineData.data, parts[1].inlineData.data)` tries index 0 first, falls through to 1 if null. Future-proof against shape changes. | Step 32-2g (Y14 fix) |
| Diagnostic Compose pattern for large-response debugging | When Gemini returned 1.8MB response body, PA run history couldn't render it and Claude conversation couldn't ingest it. Fix: interstitial Compose extracting 5 key signals (`candidate_count`, `finish_reason`, `parts_length`, `prompt_feedback`, `safety_ratings`). Tiny output, diagnosable in one paste. Reusable pattern for any large external API response. | Step 32-2g |
| One step at a time | Per Tony's explicit preference | Ongoing |

---

## 9. User preferences (IMPORTANT for next agent)

- **Tone:** Direct, terse. No cheerleading. No fluff.
- **Format:** Step-by-step, ONE step per response. Never batch multiple steps into one message.
- **Clarifying questions:** Ask when unsure (he requested this explicitly).
- **Push-back expected:** Tony will catch shortcuts and narrow thinking. Welcome it — don't get defensive, acknowledge and adjust.
- **Prefers standalone HTML/ready-to-use deliverables** over partial guidance (general pattern, not applicable to this flow build).

---

## 10. Open questions / known unknowns

- [ ] Target PIM field list (needed to design extraction schema in Stage 4)
- [ ] What do real item-setup emails contain? URL attached? Datasheet? Tony said "don't know yet."
- [ ] Where will this flow run in production — Tony's personal env or a shared SiteOne env with environment variables/Dataverse?
- [ ] Is Power Automate Desktop (RPA) available for JS-rendered site fallback?
- [ ] SharePoint site for governance logs — does one exist or do we need to provision?

---

## 11. If you're picking up mid-stream

1. Read this doc.
2. Ask Tony which step we're on — he'll tell you.
3. Give ONE step's worth of instructions. Wait for his report. Update this doc.
4. When in doubt about scope, re-read section 4 (POC scope).
5. Keep this doc updated every 1–2 steps. Don't let it go stale.
