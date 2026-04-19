# SiteOne Item Enrichment POC — Technical Recovery Doc

**Purpose:** Complete configuration of every action, expression, schema, and prompt so the entire flow can be rebuilt from scratch if destroyed or corrupted.
**Companion to:** `HANDOFF.md` (project context and status)
**Last updated:** Apr 19, 2026 — Step 32-3b complete. `Increment - Google API Calls` action added directly after `HTTP - Gemini Lifestyle 1` in the TRUE branch of `Check - Has Reference Image`; increments `google_api_calls_total` by 1 per Gemini call. See Section V3 Increment actions table. Previous: Step 32-3a complete (new top-level variable `Init - Google API Calls Total`, Integer, init 0). Step 32-2 complete end-to-end. Lifestyle image generation via Gemini Nano Banana 2 working (Brilliance validated Apr 19). Y14 RESOLVED: Gemini returns 1 part (image at `[0]`) — not 2 (text + image at `[1]`) as initial build assumed. Fix = coalesce pattern on parts index. Diagnostic `Compose - Gemini Debug` added between Parse JSON and Save (remove before prod). Step 31 email body redesign complete. Remaining Step 32 work: 32-3b Increment, 32-3c Compose - Cost Metadata update, 32-3d HTML template cost line, 32-4 HTML lifestyle section, 32-5 Intermatic validation. See Section Z for full Step 32 action configs.

---

## A. Flow identity

- **Flow name:** `POC - Product Page Harvester`
- **Type:** Instant cloud flow
- **Environment:** SiteOne tenant (confirm with Tony)

---

## B. Trigger — When a new email arrives (V3)

**Trigger type:** Office 365 Outlook → `When a new email arrives (V3)` (as of Step 30)

**Historical note:** Originally used a "Manually trigger a flow" with 6 text inputs (brand, part_number, product_name, provided_url, num_results, correlation_id_override). Replaced in Step 30 Apr 18 when email automation was implemented.

**Parameters:**

| Parameter | Value |
|---|---|
| **Folder** | `Item Setup Requests` (sub-folder under Inbox — created manually in Outlook) |
| **To** | blank |
| **From** | blank |
| **Importance** | Any |
| **Only with Attachments** | No |
| **Include Attachments** | No (v1 — attachments deferred to backlog #46) |
| **Subject Filter** | blank (inbox rule does the filtering) |

**Settings tab:**

| Setting | Value |
|---|---|
| **Split On** | **ON** (CRITICAL — V3 trigger returns array; Split On iterates one email per flow run instead of wrapping everything in For Each) |
| **Array** | `@triggerOutputs()?['body/value']` (select from dropdown) |
| **Split-on tracking ID** | blank |
| **Concurrency Control** | OFF |
| **Trigger Conditions** | none |
| **Retry Policy** | Default |

**Gotcha from Step 30 build:** If Split On is OFF, Power Automate auto-wraps downstream actions in a For Each container, and all trigger dynamic content resolves incorrectly. Fix: enable Split On, delete any unwanted For Each wrappers, rebind actions.

**Outlook side setup (manual, outside the flow):**
1. Outlook → right-click Inbox → Create new subfolder → name: `Item Setup Requests`
2. Outlook → Settings → Mail → Rules → Add new rule:
   - Name: `Route Item Setup Requests to Automation Folder`
   - Condition: Subject includes `onboard` (lowercase matches case-insensitively)
   - Action: Move to → select `Item Setup Requests` folder
   - Stop processing more rules: ON
3. Future: expand the rule with additional keywords: `item setup`, `enrich`, `new item`, `setup request`

---

## B2. Post-trigger email parsing actions

Actions 2-4 in the flow (immediately after the email trigger) are part of the email workflow. Added in Step 30.

### B2-1. `AI - Extract from Email` (AI Builder Run a Prompt)

- **Prompt:** `Extract Enrichment Request from Email` (see Section P4 for full prompt text)
- **Model in prompt config:** GPT-4.1 mini
- **Inputs:**
  | Input | Source | Value |
  |---|---|---|
  | `email_subject` | Dynamic content | Subject (from trigger) |
  | `email_body` | Expression | `triggerBody()?['Body']` |

**Gotcha from build:** Originally named the prompt inputs as `[email_subject]` and `[email_body]` (with brackets) — caused `_5b` URL-encoding in request and `InvalidPredictionInput` errors. Fix: rename inputs to plain `email_subject` and `email_body` (no brackets). The bracket notation `[variable]` is only used inside the prompt text to reference variables, NOT in the input names.

### B2-2. `Parse JSON - Email Extraction`

- **Content (Expression):**
  ```
  replace(replace(replace(body('AI_-_Extract_from_Email')?['responsev2']?['predictionOutput']?['text'], '```json', ''), '```', ''), decodeUriComponent('%0A'), '')
  ```
- **Schema:**
  ```json
  {
    "type": "object",
    "additionalProperties": true,
    "properties": {
      "brand": { "type": ["string", "null"] },
      "part_number": { "type": ["string", "null"] },
      "product_name": { "type": ["string", "null"] },
      "provided_url": { "type": ["string", "null"] },
      "requester_notes": { "type": ["string", "null"] },
      "confidence": { "type": ["integer", "number"] },
      "multi_item_detected": { "type": "boolean" },
      "attachment_likely": { "type": "boolean" },
      "parse_notes": { "type": "string" }
    }
  }
  ```

### B2-3. `Check - Valid Email Extraction` (Condition)

Checks that both brand and part_number are non-null. Two rows with AND logic:

- **Row 1:**
  - Left (Expression): `body('Parse_JSON_-_Email_Extraction')?['brand']`
  - Operator: is not equal to
  - Right (Expression): `null`
- **Row 2:**
  - Left (Expression): `body('Parse_JSON_-_Email_Extraction')?['part_number']`
  - Operator: is not equal to
  - Right (Expression): `null`

**TRUE branch:** Empty (flow falls through to Init variables — see backlog #47 for hardening)
**FALSE branch:** Two actions (see B2-4 and B2-5)

### B2-4. `Send Email - Rejection` (inside FALSE branch of Check - Valid Email Extraction)

- **To:** Dynamic content → From (trigger)
- **Subject (Expression):** `concat('Re: ', triggerBody()?['Subject'])`
- **Body:** HTML content explaining brand/part# couldn't be parsed, asking user to resend with clear labels. Shows the original parse attempt for debugging. Full template in backup file.

### B2-5. `Terminate - Invalid Email` (inside FALSE branch, after rejection email)

- **Status:** Failed
- **Code:** `InvalidEmailRequest`
- **Message (Expression):** `concat('Email from ', triggerBody()?['From'], ' could not be parsed for brand/part. Parse notes: ', coalesce(body('Parse_JSON_-_Email_Extraction')?['parse_notes'], 'none'))`

---

## C. Variables (Initialize Variable actions, in order)

### C1. `Init - Correlation ID`
- **Name:** `correlation_id`
- **Type:** `String`
- **Value (Expression):**
  ```
  guid()
  ```
- **Purpose:** Unique identifier per run. Used in folder path (per Step 29-1), HTML preview footer, all logging.
- **History:** Originally read from optional trigger input `correlation_id_override` with expression `if(empty(triggerBody()?['text_5']), guid(), triggerBody()?['text_5'])`. Simplified to always-auto when Step 30 replaced the manual trigger with email trigger (email has no override concept).

### C2. `Init - Run Timestamp`
- **Name:** `run_timestamp`
- **Type:** `String`
- **Value (Expression):** `utcNow()`

### C3. `Init - Brand`
- **Name:** `brand`
- **Type:** `String`
- **Value (Expression — updated Step 30-4):**
  ```
  body('Parse_JSON_-_Email_Extraction')?['brand']
  ```
- **Purpose:** Brand extracted from email by AI parsing prompt. All downstream actions reference `variables('brand')`.
- **History:** Originally read from trigger dynamic content (manual trigger input). Rewired to read from email extraction when Step 30 replaced manual trigger.

### C4. `Init - Part Number`
- **Name:** `part_number`
- **Type:** `String`
- **Value (Expression — updated Step 30-4):**
  ```
  body('Parse_JSON_-_Email_Extraction')?['part_number']
  ```
- **Purpose:** Part number extracted from email. All downstream actions reference `variables('part_number')`.

### C5. `Init - Search Query`
- **Name:** `search_query`
- **Type:** `String`
- **Value (Expression):**
  ```
  concat('"', variables('part_number'), '" ', variables('brand'))
  ```

### C6. `Init - Item Folder Path`
- **Name:** `item_folder_path`
- **Type:** `String`
- **Value (Expression — updated Step 29-1 for per-run isolation):**
  ```
  concat('/Item Setup POC/', variables('brand'), '_', variables('part_number'), '/', variables('correlation_id'))
  ```
- **Purpose:** Used as destination path for saving CSV, HTML preview, and downloaded images to OneDrive.
- **Target structure at runtime:** `/Item Setup POC/<Brand>_<Part>/<correlation_id>/` — every run gets its own unique subfolder, preventing 409 eTag collisions and allowing back-to-back runs without manual folder deletion.
- **Prior version (pre Step 29-1):** `concat('/Item Setup POC/', variables('brand'), '_', variables('part_number'))` — collided with itself on re-runs.

### C7. `Init - Image Counter` *(moved to top-level in Step 27c)*
- **Name:** `image_counter`
- **Type:** `Integer`
- **Value:** `0`
- **Position:** Must be at the top level of the flow (between `Init - Item Folder Path` and `HTTP` Serper action). Power Automate enforces the rule that Initialize Variable actions cannot be nested inside Conditions, Loops, or Scopes — and since the image download loop lives inside the `Check - URL Qualifies` True branch (Step 27c), the counter had to move up.
- **Side effect:** Counter now initializes on EVERY run, even ones that hit the Terminate branch. No functional impact — unused variable costs nothing.
- **Purpose:** Sequential counter used inside `Loop - Download Images` to name downloaded images (`image_1_xxx.jpg`, `image_2_xxx.jpg`, etc.). Required because `iterationIndexes()` function is unreliable in newer Power Automate versions.

---

## D. Action — `HTTP` (Serper search)

**Parameters:**
- **Method:** `POST`
- **URI:** `https://google.serper.dev/search`
- **Headers:**
  | Key | Value |
  |---|---|
  | `X-API-KEY` | *Tony's Serper API key — stored in plaintext currently (Tech debt #5 to fix)* |
  | `Content-Type` | `application/json` |
- **Body (as expression within literal JSON):**
  ```
  {"q": "@{variables('search_query')}", "num": 10, "gl": "us", "hl": "en"}
  ```
  *(The `@{variables('search_query')}` renders as a chip in the UI, not literal text.)*

**Settings:**
- **Secure inputs:** ON (hides API key in logs)
- **Secure outputs:** ON (hides search results in logs)
- **Retry Policy:** Default *(⚠️ Tech debt #4 — should be Exponential, 4 retries, PT10S/PT5S/PT1H)*
- **Action timeout:** blank / default *(⚠️ Tech debt #4 — should be `PT30S`)*
- **Authentication:** None
- **Automatic decompression:** ON
- **Asynchronous pattern:** ON
- **Allow chunking:** ON

---

## E. Action — `Parse JSON` (first one, parses Serper response)

**Inputs:**
- **Content:** Dynamic content → `Body` from `HTTP` action
- **Schema:** Auto-generated from sample (the full Serper response with `searchParameters`, `organic`, etc.). Not hardened.

*(Tech debt: should add `additionalProperties: true` at top level for future-proofing.)*

---

## F. Action — `AI - Select URLs` (Run a prompt)

**Prompt referenced:** `Select Product URLs from Search Results` (see Section J for full prompt text)

**Inputs:**
- **brand:** Dynamic content → `brand` variable (from Init - Brand)
- **part_number:** Dynamic content → `part_number` variable (from Init - Part Number)
- **search_results (Expression):**
  ```
  string(body('Parse_JSON')?['organic'])
  ```
  *(Note: `Parse_JSON` is the action's internal name — spaces become underscores in expressions.)*

---

## G. Action — `Parse JSON 1` (second Parse JSON, parses AI URL selection output)

**Inputs:**
- **Content (Expression — defensive fence-stripping added Step 27):**
  ```
  replace(replace(replace(body('AI_-_Select_URLs')?['responsev2']?['predictionOutput']?['text'], '```json', ''), '```', ''), decodeUriComponent('%0A'), '')
  ```
  *(This strips ```` ```json ```` and ```` ``` ```` markdown code fences plus newlines before parsing. Safety net for occasional LLM fence-wrapping even with strict prompt directives.)*
- **Schema (HARDENED, REBUILT Step 27b for new single-URL output shape):**
  ```json
  {
    "type": "object",
    "additionalProperties": true,
    "properties": {
      "selected_url": {
        "type": ["string", "null"]
      },
      "confidence": {
        "type": ["integer", "number"]
      },
      "source_type": {
        "type": "string"
      },
      "source_domain": {
        "type": ["string", "null"]
      },
      "reasoning": {
        "type": "string"
      },
      "candidates_considered": {
        "type": "array",
        "minItems": 0,
        "items": {
          "type": "object",
          "additionalProperties": true
        }
      }
    }
  }
  ```

**Output shape note:** The OLD output (`manufacturer_url`, `distributor_urls`, `distributor_confidences`, `manufacturer_confidence`, `excluded_reasons`, `overall_reasoning`) is GONE as of Step 27b. Anything downstream referencing those old field names must be updated to the new shape (`selected_url`, `source_type`, `source_domain`, `candidates_considered[]`, etc.).

---

## G2. Action — `Check - URL Qualifies` (Condition) *(NEW in Step 27c)*

**Purpose:** Null guard. If the AI couldn't find a URL meeting the 95% confidence threshold, `selected_url` is null. This Condition short-circuits the flow into a graceful Terminate rather than letting Firecrawl crash on a null URL.

**Position:** Directly after `Parse JSON 1`, before `Firecrawl - Scrape Product Page`.

**Condition configuration:**
- **Left operand (Expression):** `body('Parse_JSON_1')?['selected_url']`
- **Operator:** `is not equal to`
- **Right operand (Expression):** `null`

**True branch contents (in order — ALL 14 downstream actions moved here in Step 27c):**
1. `Firecrawl - Scrape Product Page`
2. `AI - Extract Product Data`
3. `Parse JSON - Extraction Output`
4. `AI - Generate Content`
5. `Parse JSON - Generated Content`
6. `AI - Flatten Row`
7. `Parse JSON - Flat Row`
8. `Create CSV Table`
9. `OneDrive - Save CSV`
10. `Loop - Download Images` (Apply to each with 3 child actions inside)
11. `Compose - Extraction Metadata` *(run-after: ✓ success + ✓ failed)*
12. `Compose - HTML Preview` *(run-after: ✓ success + ✓ failed)*
13. `OneDrive - Save HTML Preview` *(run-after: ✓ success + ✓ failed)*

*(14 counting the Apply-to-each as one; 13 actions plus the Loop container.)*

**⚠️ Important:** `Init - Image Counter` CANNOT live inside the True branch — Power Automate rejects nested Initialize Variable actions with error `InvalidVariableInitialization`. It was moved to the top-level variables section (C7 above).

**False branch contents:**
- Single action: `Terminate - Flag for Human Review` (see G3 below)

---

## G3. Action — `Terminate - Flag for Human Review` *(NEW in Step 27c)*

**Position:** Inside the False branch of `Check - URL Qualifies`.

**Action type:** Control → Terminate

**Parameters:**
- **Status:** `Failed`
- **Code:** `NoQualifyingURL`
- **Message (Expression):**
  ```
  concat('No URL met 95% confidence threshold for ', variables('brand'), ' ', variables('part_number'), '. Reasoning: ', coalesce(body('Parse_JSON_1')?['reasoning'], 'no reasoning provided'), '. Correlation ID: ', variables('correlation_id'))
  ```

**Behavior:** When fired, the Terminate action halts the entire flow with Failed status. Run history shows the full message for operator debugging. Correlation ID is captured for cross-referencing with future logging (#8).

---

## H. Action — `Firecrawl - Scrape Product Page` *(renamed from "Scrape Manufacturer Page" in Step 27c)*

**Position:** Inside the True branch of `Check - URL Qualifies` (first action in True branch).

**Purpose:** Fetch product page content as clean markdown AND HTML (for richer image extraction). Handles JavaScript-rendered SPAs with `waitFor` delay. Replaced direct HTTP GET which failed on React SPAs like intermatic.com. Renamed from "Manufacturer Page" because post Step 27 the URL can be manufacturer OR distributor depending on confidence-first selection.

**Parameters:**
- **Method:** `POST`
- **URI:** `https://api.firecrawl.dev/v2/scrape`
- **Headers:**
  | Key | Value |
  |---|---|
  | `Authorization` | `Bearer fc-YOUR_FIRECRAWL_KEY` *(plaintext — tech debt, move to env var)* |
  | `Content-Type` | `application/json` |
  | `Accept` | `application/json` |
- **Body (paste as raw JSON text, Power Automate resolves `@{}` at runtime):**
  ```json
  {
    "url": "@{body('Parse_JSON_1')?['selected_url']}",
    "formats": ["markdown", "html"],
    "onlyMainContent": true,
    "waitFor": 5000
  }
  ```
  *(URL reference updated in Step 27c from `['manufacturer_url']` to `['selected_url']`.)*

**Key parameter notes:**
- `formats: ["markdown", "html"]` — markdown is for extraction prompt; html gives us `<img>` tags for gallery images
- `onlyMainContent: true` — strips headers/nav/footer/sidebars; keeps main product content only
- `waitFor: 5000` — waits 5 seconds for JavaScript to finish rendering before scraping. Critical for React SPAs where product galleries lazy-load. ⚠️ May not be enough for all sites — image gallery capture is STILL incomplete (see Section W backlog #31).

**Settings:**
- **Retry Policy:** Exponential Interval, Count 4, Interval `PT10S`, Min `PT5S`, Max `PT1H`
- **Action timeout:** `PT180S` (3 minutes — bumped from PT120S due to waitFor)
- **Secure inputs:** ON (hides API key) — turn OFF temporarily for debug
- **Secure outputs:** OFF during dev (turn ON for prod)

**Response format (success):**
```json
{
  "success": true,
  "data": {
    "markdown": "...clean product content in markdown...",
    "html": "<!DOCTYPE html>...fully-rendered HTML with <img> tags...",
    "metadata": {
      "creditsUsed": 1,
      "sourceURL": "...",
      "ogImage": "...",
      "cacheState": "miss|hit"
    }
  }
}
```

**Credits:** 1 per scrape (basic). Free tier = 500 credits/month.

**Known issues:**
- `cacheState: "hit"` can serve stale/wrong content. Firecrawl caches aggressively — same URL returns cached result. No clean cache-bust parameter exposed in v2/scrape.
- Some React SPAs don't finish rendering within `waitFor: 5000`. Product galleries may still be missing. Use Firecrawl `actions` (scroll/click/wait-for-selector) for deeper JS interaction — noted as backlog #31.
- Serper URL results vary per run — can return Mexico subdomain (`mx.intermatic.com`) instead of US (`www.intermatic.com`), with much sparser content. Backlog #32.

---

## I. Action — `AI - Extract Product Data`

**Purpose:** Kitchen-sink extraction of structured product data from Firecrawl markdown using the `Extract Product Data from Markdown` prompt.

**Action type:** AI Builder → Run a prompt

**Prompt referenced:** `Extract Product Data from Markdown` (see Section L for full prompt text)

**Inputs (as configured in the flow action):**
| Input | Source | Value/Expression |
|---|---|---|
| `brand` | Dynamic content → Variables | `variables('brand')` |
| `part_number` | Dynamic content → Variables | `variables('part_number')` |
| `source_url` | Expression tab | `body('Parse_JSON_1')?['selected_url']` *(updated Step 27c — was `['manufacturer_url']`)* |
| `markdown` | Expression tab | `body('Firecrawl_-_Scrape_Product_Page')?['data']?['markdown']` *(internal name updated to match 27c rename)* |

**Settings:**
- **Secure inputs:** OFF (set to ON for production)
- **Secure outputs:** OFF (for debugging)
- **Asynchronous Pattern:** ON (MANDATORY per Microsoft docs — required for AI Builder responses to return properly)

**Typical run stats (Intermatic DT200LT test):**
- Model used: `gpt-5-chat-2025-07-14`
- Prompt tokens: ~4,873
- Completion tokens: ~2,478
- Total tokens: ~7,351
- **AI Builder credits: ~247**
- Runtime: 20-60 seconds depending on markdown size
- Output: `responsev2.predictionOutput.text` field contains the JSON string

---

## J. Action — `Parse JSON - Extraction Output`

**Purpose:** Convert the `Text` output from `AI - Extract Product Data` into a queryable object so downstream actions can reference specific fields (e.g., `body('Parse_JSON_-_Extraction_Output')?['product_identity']?['product_name']?['value']`).

**Inputs:**
- **Content (Expression — defensive fence-stripping added Step 27):**
  ```
  replace(replace(replace(body('AI_-_Extract_Product_Data')?['responsev2']?['predictionOutput']?['text'], '```json', ''), '```', ''), decodeUriComponent('%0A'), '')
  ```
- **Schema (HARDENED — keep this exact):**

```json
{
  "type": "object",
  "additionalProperties": true,
  "properties": {
    "product_identity": {
      "type": "object",
      "additionalProperties": true,
      "properties": {
        "product_name": { "type": "object", "additionalProperties": true },
        "brand": { "type": "object", "additionalProperties": true },
        "manufacturer_part_number": { "type": "object", "additionalProperties": true },
        "model_number": { "type": "object", "additionalProperties": true },
        "upc": { "type": "object", "additionalProperties": true },
        "sku": { "type": "object", "additionalProperties": true }
      }
    },
    "description": {
      "type": "object",
      "additionalProperties": true
    },
    "features": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "object", "additionalProperties": true }
    },
    "applications": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "object", "additionalProperties": true }
    },
    "specifications": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "object", "additionalProperties": true }
    },
    "dimensions": {
      "type": "object",
      "additionalProperties": true
    },
    "packaging": {
      "type": "object",
      "additionalProperties": true
    },
    "compliance_and_certifications": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "object", "additionalProperties": true }
    },
    "country_of_origin": { "type": "object", "additionalProperties": true },
    "warranty": { "type": "object", "additionalProperties": true },
    "images": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "object", "additionalProperties": true }
    },
    "documents": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "object", "additionalProperties": true }
    },
    "videos": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "object", "additionalProperties": true }
    },
    "price_observed": { "type": "object", "additionalProperties": true },
    "additional_attributes": { "type": "object", "additionalProperties": true },
    "extraction_metadata": { "type": "object", "additionalProperties": true }
  }
}
```

**Schema design rationale:**
- `additionalProperties: true` everywhere → tolerates new fields the AI might add in future extractions
- Array items typed as generic `object` with `additionalProperties: true` → we don't over-constrain per-item shape
- `minItems: 0` on all arrays → empty arrays are valid (e.g., a product with no documents)
- No `required` field list at top level → schema won't fail validation if AI skips entire sections for sparse data

---

---

## K. Action — `AI - Generate Content`

**Purpose:** Generate fresh SiteOne-voice marketing content (descriptions, feature bullets, SEO keywords) grounded in the extracted specs. Avoids republishing manufacturer copy.

**Action type:** AI Builder → Run a prompt

**Prompt referenced:** `Generate Product Content` (see Section R for full prompt text)

**Inputs:**
| Input | Source | Value/Expression |
|---|---|---|
| `extracted_data` | Expression tab | `body('AI_-_Extract_Product_Data')?['responsev2']?['predictionOutput']?['text']` |

**Settings:**
- **Asynchronous Pattern:** ON
- **Secure inputs/outputs:** OFF for dev

---

## L. Action — `Parse JSON - Generated Content`

**Inputs:**
- **Content (Expression — defensive fence-stripping added Step 27):**
  ```
  replace(replace(replace(body('AI_-_Generate_Content')?['responsev2']?['predictionOutput']?['text'], '```json', ''), '```', ''), decodeUriComponent('%0A'), '')
  ```
- **Schema:**

```json
{
  "type": "object",
  "additionalProperties": true,
  "properties": {
    "short_description": { "type": "string" },
    "long_description": { "type": "string" },
    "feature_bullets": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "string" }
    },
    "seo_keywords": {
      "type": "array",
      "minItems": 0,
      "items": { "type": "string" }
    },
    "generation_metadata": { "type": "object", "additionalProperties": true }
  }
}
```

---

## M. Action — `AI - Flatten Row`

**Purpose:** Transform nested extraction JSON + generated content JSON into a single flat JSON object with dynamic keys. Output feeds Create CSV Table directly.

**Action type:** AI Builder → Run a prompt

**Prompt referenced:** `Flatten Product Data to CSV Row` (see Section S)

**Inputs:**
| Input | Source | Value/Expression |
|---|---|---|
| `extracted_data` | Expression tab | `body('AI_-_Extract_Product_Data')?['responsev2']?['predictionOutput']?['text']` |
| `generated_content` | Expression tab | `string(body('Parse_JSON_-_Generated_Content'))` |

**Settings:**
- Asynchronous Pattern: ON

**Typical stats:** ~5-10 sec runtime, GPT-4.1 model, lower credit usage than extraction prompt.

---

## N. Action — `Parse JSON - Flat Row`

**Inputs:**
- **Content (Expression — defensive fence-stripping added Step 27):**
  ```
  replace(replace(replace(body('AI_-_Flatten_Row')?['responsev2']?['predictionOutput']?['text'], '```json', ''), '```', ''), decodeUriComponent('%0A'), '')
  ```
- **Schema (permissive — handles dynamic keys):**

```json
{
  "type": "object",
  "additionalProperties": true
}
```

---

## O. Action — `Create CSV Table`

**Parameters:**
- **From (Expression):**
  ```
  createArray(body('Parse_JSON_-_Flat_Row'))
  ```
  *(Wrapping single flat object into a 1-element array — Create CSV Table expects an array input.)*
- **Columns:** `Automatic`

---

## O2. Action — `OneDrive - Save CSV`

**Connection:** OneDrive for Business (SiteOne work account)

**Parameters:**
- **Folder Path:** Dynamic content → `item_folder_path` variable (must be a purple chip, NOT typed text — if typed as expression, OneDrive treats it as literal path)
- **File Name (Expression tab):**
  ```
  concat(variables('part_number'), '.csv')
  ```
- **File Content:** Dynamic content → Output from `Create CSV Table`

**Runtime behavior:**
- If folder `/Item Setup POC/<Brand>_<Part>/` doesn't exist, OneDrive creates it automatically
- If file already exists with the same name, overwrites it
- Returns Path field showing full OneDrive path (use this to verify resolution at runtime)

**Known pitfalls:**
- If `item_folder_path` variable's VALUE contains literal text like `concat('/Item Setup POC/'...)` instead of a resolved string, OneDrive creates a folder with that bizarre name. Fix: init the variable using the Expression (fx) picker, not typing raw text. Should appear as a purple chip in the variable's Value field.
- Filename field must show purple chip too after pasting expression

---

## P1. Action — `Init - Image Counter` *(MOVED in Step 27c)*

**Status:** This action was relocated to the top-level Variables section (see **C7** above) during Step 27c. Power Automate does not permit Initialize Variable actions inside Condition/Scope/Loop containers, and since the image download loop now lives inside the `Check - URL Qualifies` True branch, the counter had to move up.

**If rebuilding from scratch:** Add the counter at Variables initialization time (C7 in this doc), not here. This section is kept for historical reference only.

---

## P2. Action — `Loop - Download Images` (Apply to each)

**Purpose:** Iterate through the images array from extraction output, downloading each to OneDrive.

**Position:** Inside True branch of `Check - URL Qualifies`, after `OneDrive - Save CSV`.

**Parameters:**
- **Select an output from previous steps (Expression — updated Step 27 to filter null URLs):**
  ```
  filter(body('Parse_JSON_-_Extraction_Output')?['images'], item()?['url'] != null)
  ```
  *(Previously was a Dynamic content reference to `images` direct. Some lazy-loaded sites yield image entries with null URLs even after 27e's prompt update; this filter skips them before the HTTP download fires.)*

**Settings:**
- **Concurrency Control:** ON
- **Degree of Parallelism:** `1` (required because `image_counter` variable would race-condition under parallel execution)

**Child actions (in order inside the loop):**

### P2a. `Increment - Image Counter`
- **Name:** `image_counter`
- **Value:** `1`

### P2b. `HTTP - Download Image`
- **Method:** `GET`
- **URI (Expression):** `item()?['url']`
- **Headers/Queries/Body:** empty
- **Settings → Retry Policy:** Exponential, Count 3, Interval PT10S

### P2c. `OneDrive - Save Image`
- **Connection:** OneDrive for Business (same as Save CSV)
- **Folder Path:** Dynamic content → `item_folder_path` variable (purple chip)
- **File Name (Expression):**
  ```
  concat('image_', string(variables('image_counter')), '_', coalesce(item()?['type'], 'unknown'), '.jpg')
  ```
- **File Content:** Dynamic content → Body (from `HTTP - Download Image`)

**Known issues:**
- **409 eTag mismatch** if the target folder has leftover files from prior failed runs. Fix: manually delete `/Item Setup POC/<Brand>_<Part>/` folder before rerun to start fresh.
- **.jpg extension hardcoded** — production fix = detect Content-Type response header, map to extension. Most landscape product images are JPEG so it's functionally fine.
- **Image capture incomplete** — currently gets 4/14 product images; `waitFor: 5000` helps but gallery lazy-load continues beyond 5-second wait. Backlog #31.

---

## Q1. Action — `Compose - Extraction Metadata`

**Position:** After `Loop - Download Images` (outside/after the loop)

**Purpose:** Build the metadata object consumed by the HTML preview template. Captures inputs, timestamps, and extraction metadata for display in the preview's header and footer.

**Run After:** Configured with ✓ `is successful` AND ✓ `has failed` on the preceding `Loop - Download Images`. This lets the HTML preview generate even if one or more image downloads fail (409 eTag, timeout, etc.).

**Inputs:**
```json
{
  "correlation_id": "@{variables('correlation_id')}",
  "run_timestamp": "@{variables('run_timestamp')}",
  "input_brand": "@{variables('brand')}",
  "input_part_number": "@{variables('part_number')}",
  "input_provided_url": null,
  "ai_builder_credits_used": null,
  "firecrawl_credits_used": null,
  "serper_searches": 1,
  "run_duration_seconds": null,
  "extraction_metadata": @{body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']},
  "confidence_by_field": {},
  "excluded_urls": []
}
```

**Note on null fields:** `ai_builder_credits_used`, `firecrawl_credits_used`, `run_duration_seconds` are placeholders until cost/time tracking is implemented (backlog #39). Preview renders "—" for null values.

---

## Q2. Action — `Compose - HTML Preview`

**Position:** After `Compose - Extraction Metadata`

**Run After:** ✓ `is successful` AND ✓ `has failed` on preceding action.

**Purpose:** Assemble the complete HTML preview file. The full template content (~24KB of HTML/CSS/JavaScript) is pasted into the Compose Inputs field. The template uses two base64-encoded data injections:
- Flat JSON (from Parse JSON - Flat Row) → injected as `@{base64(string(body('Parse_JSON_-_Flat_Row')))}`
- Extraction metadata (from Q1 above) → injected as `@{base64(outputs('Compose_-_Extraction_Metadata'))}`

**Why base64:** The flat JSON contains fields like long descriptions with real newlines and quotes. Directly embedding JSON in a `<script>` tag breaks when these characters appear. Base64 converts the payload to safe A-Z/0-9 characters; JavaScript decodes at page load with `atob()`.

**Runtime behavior of the template JavaScript:**
1. Reads both base64 payloads from `<script type="text/plain">` tags
2. Strips whitespace (Power Automate may inject line breaks into long base64 strings)
3. Decodes using `decodeURIComponent(escape(atob(...)))` (UTF-8 safe for special characters like °, ®)
4. Parses JSON
5. Scans keys for patterns: `feature_bullet_N`, `seo_keyword_N`, `image_N_url/alt/type`, `document_N_url/title/type`, `video_N_url/title`
6. All remaining keys (not in the fixed-field list and not matching indexed patterns) are treated as dynamic specifications
7. Renders each section adaptively — any count of bullets, specs, images, documents works

**Full template file:** `compose_template_v3.html` (included in project artifacts). Paste its contents verbatim into Compose Inputs.

**Template sections rendered:**
- Banner (product name, brand, part#, overall confidence)
- Enrichment Request (inputs, source URL)
- Product Identity (name/brand/MFG#/UPC/SKU with per-field confidence badges; missing criticals show ⚠ warning)
- Content Comparison (Extracted source vs Generated SiteOne-voice, side-by-side, short + long)
- Feature Bullets (numbered list, any count)
- Specifications (dynamic table, ALL spec attributes)
- Certifications (comma-separated, displayed as accent badges)
- Image Gallery (adaptive grid with type labels)
- Documents & Assets (clickable cards with type indicators)
- Videos (thumbnail cards)
- SEO Keywords (monospace tag cloud)
- Audit — Sources Excluded (reason for each rejected URL)
- Extraction Notes (from extraction_metadata)
- Footer (correlation ID, timestamp, duration, credit breakdown, confidence, field count)

---

## Q3. Action — `OneDrive - Save HTML Preview`

**Position:** After `Compose - HTML Preview` (last action in flow)

**Run After:** ✓ `is successful` AND ✓ `has failed` on preceding action.

**Connection:** OneDrive for Business (same as other saves)

**Parameters:**
- **Folder Path:** Dynamic content → `item_folder_path` variable (purple chip)
- **File Name (Expression):** `concat(variables('part_number'), '_preview.html')`
- **File Content:** Dynamic content → Outputs from `Compose - HTML Preview`

**Result:** Opens in any browser. Fully self-contained HTML (inline CSS and JavaScript). No external dependencies except Google Fonts (IBM Plex Sans + Mono) which require internet but gracefully fall back.

---

## Q4. Action — `Send Email - Reply with HTML` *(Step 30-6; body REDESIGNED Step 31 Apr 19)*

**Position:** LAST action in the flow, after `OneDrive - Save HTML Preview`.

**Run After:** ✓ `is successful` AND ✓ `has failed` on preceding action. (We want to reply even if OneDrive save had issues — HTML content is already generated in the Compose action and attached directly, no dependency on OneDrive save succeeding.)

**Connection:** Office 365 Outlook (same connection as trigger and rejection email)

**Parameters:**

| Parameter | Value |
|---|---|
| **To** | Dynamic content → `From` (from trigger — the original sender) |
| **Subject (Expression)** | `concat('Enrichment complete: ', variables('brand'), ' ', variables('part_number'))` |
| **Body** | HTML (see full body below — Step 31 visual KPI redesign) |
| **Importance** | Normal |
| **From (Send as)** | blank (uses authenticated connection identity) |

**Body HTML (v2 — Step 31):**

The body uses table-based layout with inline CSS for cross-client compatibility (Outlook desktop, Outlook web, Gmail, Apple Mail). Contains:
- **Green success banner** with ✅ emoji + "Enrichment complete" headline + `brand · part_number` subtitle (banner is always green — flow only reaches this action on successful runs; Terminate fires earlier on extraction failure)
- **3-KPI strip** with confidence-tier-based color coding on Confidence card only (green ≥90, amber 70-89, red <70), Duration with image-agent-fired subtitle, Est. Cost with accent styling
- **Existing summary bullets** preserved below banner (Brand, Part #, Confidence, Cost, Duration, Image Agent)
- **Footer** with Correlation ID + attribution

```html
<table width="100%" cellpadding="0" cellspacing="0" border="0" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Arial, sans-serif; color: #1f2937;">
  <tr>
    <td>

      <!-- SUCCESS BANNER -->
      <table width="100%" cellpadding="0" cellspacing="0" border="0" style="background: #ecfdf5; border-left: 4px solid #059669; border-radius: 4px;">
        <tr>
          <td style="padding: 18px 20px;">
            <table cellpadding="0" cellspacing="0" border="0">
              <tr>
                <td style="font-size: 32px; padding-right: 14px; vertical-align: middle; line-height: 1;">&#9989;</td>
                <td style="vertical-align: middle;">
                  <div style="font-size: 18px; font-weight: 600; color: #064e3b; line-height: 1.2;">Enrichment complete</div>
                  <div style="font-size: 13px; color: #047857; margin-top: 3px;">@{variables('brand')} &middot; @{variables('part_number')}</div>
                </td>
              </tr>
            </table>
          </td>
        </tr>
      </table>

      <!-- KPI STRIP -->
      <table width="100%" cellpadding="0" cellspacing="0" border="0" style="margin-top: 16px;">
        <tr>
          <td width="33%" valign="top" style="padding-right: 6px;">
            <table width="100%" cellpadding="0" cellspacing="0" border="0" style="background: #f0fdfa; border: 1px solid #99f6e4; border-radius: 6px;">
              <tr>
                <td style="padding: 14px 12px; text-align: center;">
                  <div style="font-size: 10px; color: #64748b; text-transform: uppercase; letter-spacing: 0.1em; font-weight: 600;">Confidence</div>
                  <div style="font-size: 30px; font-weight: 700; color: @{if(greaterOrEquals(int(body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']?['overall_confidence']), 90), '#059669', if(greaterOrEquals(int(body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']?['overall_confidence']), 70), '#b45309', '#b91c1c'))}; margin-top: 6px; line-height: 1;">@{body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']?['overall_confidence']}<span style="font-size: 16px;">%</span></div>
                  <div style="font-size: 11px; color: @{if(greaterOrEquals(int(body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']?['overall_confidence']), 90), '#047857', if(greaterOrEquals(int(body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']?['overall_confidence']), 70), '#92400e', '#991b1b'))}; margin-top: 6px;">@{if(greaterOrEquals(int(body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']?['overall_confidence']), 90), 'Ready for PIM', if(greaterOrEquals(int(body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']?['overall_confidence']), 70), 'Review recommended', 'Human review required'))}</div>
                </td>
              </tr>
            </table>
          </td>
          <td width="34%" valign="top" style="padding: 0 4px;">
            <table width="100%" cellpadding="0" cellspacing="0" border="0" style="background: #f8fafc; border: 1px solid #e2e8f0; border-radius: 6px;">
              <tr>
                <td style="padding: 14px 12px; text-align: center;">
                  <div style="font-size: 10px; color: #64748b; text-transform: uppercase; letter-spacing: 0.1em; font-weight: 600;">Duration</div>
                  <div style="font-size: 30px; font-weight: 700; color: #1e293b; margin-top: 6px; line-height: 1;">@{div(sub(ticks(utcNow()), ticks(variables('run_timestamp'))), 10000000)}<span style="font-size: 16px;">s</span></div>
                  <div style="font-size: 11px; color: #64748b; margin-top: 6px;">@{if(outputs('Compose_-_Cost_Metadata')['image_agent_fired'], 'Image agent fired', 'Standard path')}</div>
                </td>
              </tr>
            </table>
          </td>
          <td width="33%" valign="top" style="padding-left: 6px;">
            <table width="100%" cellpadding="0" cellspacing="0" border="0" style="background: #f0fdfa; border: 1px solid #99f6e4; border-radius: 6px;">
              <tr>
                <td style="padding: 14px 12px; text-align: center;">
                  <div style="font-size: 10px; color: #64748b; text-transform: uppercase; letter-spacing: 0.1em; font-weight: 600;">Est. Cost</div>
                  <div style="font-size: 30px; font-weight: 700; color: #0f766e; margin-top: 6px; line-height: 1;">$@{formatNumber(outputs('Compose_-_Cost_Metadata')['estimated_total_usd'], '0.00')}</div>
                  <div style="font-size: 11px; color: #0f766e; margin-top: 6px;">Per this run</div>
                </td>
              </tr>
            </table>
          </td>
        </tr>
      </table>

      <!-- EXISTING SUMMARY TEXT (preserved per Tony) -->
      <div style="margin-top: 24px; font-size: 14px; color: #1f2937; line-height: 1.5;">
        <p style="margin: 0 0 12px 0;">Hi,</p>
        <p style="margin: 0 0 12px 0;">Your item setup request has been processed.</p>
        <p style="margin: 0 0 6px 0;"><strong>Summary:</strong></p>
        <ul style="margin: 0 0 12px 0; padding-left: 20px;">
          <li>Brand: @{variables('brand')}</li>
          <li>Part Number: @{variables('part_number')}</li>
          <li>Overall Confidence: @{body('Parse_JSON_-_Extraction_Output')?['extraction_metadata']?['overall_confidence']}%</li>
          <li>Total Cost (estimated): $@{formatNumber(outputs('Compose_-_Cost_Metadata')['estimated_total_usd'], '0.00')}</li>
          <li>Run Duration: @{div(sub(ticks(utcNow()), ticks(variables('run_timestamp'))), 10000000)} seconds</li>
          <li>Image Agent: @{if(outputs('Compose_-_Cost_Metadata')['image_agent_fired'], 'fired (distributor fallback path)', 'not needed')}</li>
        </ul>
        <p style="margin: 0 0 12px 0;">See the attached HTML preview for full details &mdash; product identity, content comparison (extracted vs. generated), specifications, images, documents, and audit trail.</p>
        <p style="margin: 0 0 12px 0;">Open the attached file in a browser to view the full enrichment report.</p>
      </div>

      <!-- FOOTER -->
      <div style="margin-top: 20px; padding-top: 14px; border-top: 1px solid #e5e7eb; font-size: 12px; color: #6b7280;">
        <div><strong style="color: #374151;">Correlation ID:</strong> @{variables('correlation_id')}</div>
        <div style="margin-top: 6px;">SiteOne Item Setup Automation</div>
      </div>

    </td>
  </tr>
</table>
```

**Attachments section (expand under Advanced options):**

| Parameter | Value |
|---|---|
| **Attachments Name - 1 (Expression)** | `concat(variables('part_number'), '_preview.html')` |
| **Attachments Content - 1 (Expression)** | `outputs('Compose_-_HTML_Preview')` |

**Design notes (Step 31):**
- Email-safe HTML: tables for layout (flexbox/grid don't work in Outlook desktop), inline CSS only (`<style>` blocks stripped by some clients), native emoji (no SVG, no image hosting), system-font stack (no web fonts).
- Banner stays green regardless of confidence tier — pipeline only reaches this action on successful runs. Tier signaling happens in the Confidence KPI card (number color + tier label: "Ready for PIM" / "Review recommended" / "Human review required").
- Confidence color-coding thresholds match template v4 HTML preview: ≥90 green (#059669), 70-89 amber (#b45309), <70 red (#b91c1c).
- Tier label subtitle color uses darker stop of same hue (green #047857, amber #92400e, red #991b1b) for contrast on light background.
- Existing summary bullet list preserved per Tony's explicit preference — acts as plain-text fallback if any KPI chip renders oddly.
- Each `@{}` expression evaluates independently — the confidence expression is duplicated 4 times (number, number color, subtitle color, tier label). Unavoidable in PA without adding a variable. Fine because expressions are cheap.
- Attaches the HTML preview directly from the Compose action's output, not re-read from OneDrive (simpler, no extra API call, avoids dependency on OneDrive save success).
- Only one attachment — no CSV. Per Tony's explicit decision in Step 30 design review.
- Reply goes to sender only (no CC to Tony). Per Tony's decision.

**Edit gotcha (future agents):** The rich-text editor in the `Send Email` action's Body field escapes HTML by default. To paste/edit raw HTML you MUST click the `</>` (Code View) icon in the toolbar first, paste, then click out. If the email arrives showing literal `<table>` tags as text instead of rendering, the HTML was pasted in rich-text mode, not code view.

---

## P. AI Builder Prompt — `Select Product URLs from Search Results` *(REWRITTEN Step 27a)*

**Location:** make.powerautomate.com → AI hub → Prompts

**Inputs:**
| Name | Type | Sample value (for testing) |
|---|---|---|
| `brand` | Text | `Intermatic` |
| `part_number` | Text | `DT200LT` |
| `search_results` | Text | *(paste the organic array from a recent Serper response)* |

**Model settings:**
- **Model:** GPT-4.1 or GPT-5 (highest available, NOT mini)
- **Temperature:** `0`
- **Output format:** JSON (if available) or plain text (prompt forces JSON)

**Design rationale (Step 27a):** Single URL output, not a ranked list. Manufacturer site ALWAYS wins if one exists at ≥95% — no multi-source merging for POC. Only falls through to distributor when Google doesn't surface a manufacturer domain in the results.

**Full prompt text:**
```
You are a product data analyst helping select the single best URL to scrape for product enrichment.

INPUTS:
- Brand: [brand]
- Manufacturer Part Number: [part_number]
- Search Results (JSON): [search_results]

YOUR TASK:
Select EXACTLY ONE URL to scrape. Return null if no URL meets the 95% confidence threshold.

SELECTION LOGIC (strict priority order):

1. PREFER MANUFACTURER SITE:
   - Scan search results for URLs on the brand's official manufacturer domain (e.g., for Intermatic, intermatic.com or its regional subdomains like www.intermatic.com preferred over mx.intermatic.com)
   - PREFER the primary US English domain over regional/localized subdomains
   - If a manufacturer URL exists AND the page is for THIS SPECIFIC part number AND confidence ≥ 95% → SELECT IT. STOP. Return it.

2. FALLBACK TO DISTRIBUTOR:
   - Only if no manufacturer URL meets the criteria above
   - Search for distributor URLs that clearly sell THIS SPECIFIC product
   - Valid distributors include: Grainger, Zoro, Ferguson, SupplyHouse, specialized B2B distributors, and major retailers (Lowes, Home Depot, Menards) for product data
   - Select the distributor URL with highest confidence, provided it meets ≥ 95%

3. NO ACCEPTABLE URL:
   - If neither a manufacturer nor distributor meets 95% confidence → return null URL
   - This triggers human review downstream

HARD EXCLUSIONS (never select, any match):
- YouTube, Vimeo, other video sites
- Facebook, Instagram, Pinterest, Twitter/X, Reddit
- Forums, Q&A sites, community boards
- Review-only sites (TrustPilot, ResellerRatings, SiteJabber)
- Amazon product pages (inconsistent data, review noise)
- eBay, Etsy, Craigslist (user listings, unreliable)
- PDF-only direct links
- Blog posts, news articles
- Category/listing pages (must be a specific product detail page)
- siteone.com or any *.siteone.com subdomain (our own catalog)

CONFIDENCE SCORING (0-100):
- 95-100: URL path or title contains EXACT part number AND domain/brand is clearly correct AND URL is a product detail page
- 85-94: Strong match but one weak signal (e.g., title has part number but domain is a reseller we're unsure about)
- 70-84: Plausible but uncertain (partial part number match, ambiguous page type)
- Below 70: Do not select

OUTPUT FORMAT (JSON only, no preamble, no code fences):
{
  "selected_url": "https://... or null",
  "confidence": 0-100,
  "source_type": "manufacturer" | "distributor" | "none",
  "source_domain": "example.com",
  "reasoning": "2-3 sentence explanation of selection (or why nothing qualified)",
  "candidates_considered": [
    {
      "url": "https://...",
      "domain": "...",
      "source_type": "manufacturer|distributor|other",
      "confidence": 0-100,
      "decision": "selected|rejected",
      "reason": "brief reason"
    }
  ]
}

Return ONLY the raw JSON object. Do NOT wrap it in code fences (```json or ```). Do NOT add any preamble, explanation, or text before or after. Your entire response must be parseable by JSON.parse() starting at character 1.
```

*(The bracket-style `[brand]`, `[part_number]`, `[search_results]` are dynamic variable placeholders inserted via the `/` shortcut or "Insert variable" button in Prompt Builder — NOT literal text.)*

**Step 27 validated results:**
- Intermatic DT200LT → picks `www.intermatic.com/Product/DT200LT` at 97-100% confidence (manufacturer path)
- Brilliance BRI-WIFI-SMART-SOCKET-3 → falls through to distributor; picks Sprinkler Supply Store at 98% OR Ewing Outdoor Supply at 97% (varies per Serper run — no manufacturer indexed)
- siteone.com correctly rejected with "Explicitly excluded per hard rules"


## Q. AI Builder Prompt — `Extract Product Data from Markdown`

**Location:** make.powerautomate.com → AI hub → Prompts

**Purpose:** Kitchen-sink extraction of all structured product data from scraped markdown. Returns JSON with per-field confidence, source citations, and honest nulls for missing data.

**Inputs:**
| Name | Type | Sample value |
|---|---|---|
| `brand` | Text | `Intermatic` |
| `part_number` | Text | `DT200LT` |
| `source_url` | Text | `https://www.intermatic.com/Product/DT200LT` |
| `markdown` | Text | *(full markdown output from Firecrawl scrape)* |

**Model settings:**
- **Model:** GPT-5 (`gpt-5-chat-2025-07-14`) or GPT-4.1 (highest available, NOT mini)
- **Temperature:** `0`
- **Output format:** JSON if available

**Full prompt text (verbatim — in-prompt variable references like `[brand]` are inserted as dynamic chips via the `/` key or Insert Variable button):**

```
You are a product data extraction specialist. Your job is to extract every piece of structured product data from scraped webpage markdown and return it as clean JSON.

INPUTS:
- Brand: [brand]
- Manufacturer Part Number: [part_number]
- Source URL: [source_url]
- Scraped Markdown Content: [markdown]

EXTRACTION RULES:

1. EXTRACT EVERYTHING. Capture every spec, attribute, feature, application, certification, document link, image URL, and description you can find in the markdown. Better to over-capture than miss data.

IMAGE URL DETECTION (important for lazy-loaded sites) — *added Step 27e*:
When extracting image URLs, many modern e-commerce sites use lazy-loading patterns where the `src` attribute is empty or contains a placeholder, and the REAL image URL is stored in one of these alternate attributes:
- `data-src="..."` (most common lazy-load pattern)
- `data-srcset="..."` or `srcset="..."` (responsive images — take the first URL or highest-resolution one)
- `data-original="..."`
- `data-lazy-src="..."`
- `data-image="..."`

When scanning HTML for `<img>` tags, check ALL of these attributes in order of preference. If `src` is empty, null, a data URI (starts with `data:image/`), or a tiny placeholder (1x1 pixel, "blank.gif", etc.) → use the alternate attribute value instead.

If you find no valid image URL after checking all patterns → skip that image entirely rather than returning null. We do not want image entries with null URLs.

2. GROUND IN THE SOURCE. Only extract what appears in the markdown. Do NOT fill in values from your training knowledge. If a field is not present, omit it or set to null — NEVER guess.

3. PRESERVE ORIGINAL WORDING for descriptions, features, and applications. Do not summarize or rewrite.

4. NORMALIZE UNITS where obvious (e.g., "3.307 in" stays as "3.307 in"; do not convert). Keep units as they appear.

5. DEDUPE. The scraped markdown may contain repeated content blocks. Output each unique fact only once.

6. CONFIDENCE SCORE PER FIELD. For every extracted value, provide a confidence score 0-100:
   - 95-100: Exact value present in a clearly labeled spec table or field
   - 80-94: Value present in structured form but with minor ambiguity
   - 60-79: Inferred from surrounding context
   - Below 60: Uncertain — flag for review

7. SOURCE CITATION. For each extracted field, note where in the markdown it was found (e.g., "spec_table", "description", "features_list", "documents_section").

OUTPUT FORMAT (JSON only, no preamble, no code fences, no markdown):

{
  "product_identity": {
    "product_name": {"value": "string", "confidence": 0-100, "source": "string"},
    "brand": {"value": "string", "confidence": 0-100, "source": "string"},
    "manufacturer_part_number": {"value": "string", "confidence": 0-100, "source": "string"},
    "model_number": {"value": "string or null", "confidence": 0-100, "source": "string"},
    "upc": {"value": "string or null", "confidence": 0-100, "source": "string"},
    "sku": {"value": "string or null", "confidence": 0-100, "source": "string"}
  },
  "description": {
    "short_description": {"value": "string", "confidence": 0-100, "source": "string"},
    "long_description": {"value": "string", "confidence": 0-100, "source": "string"}
  },
  "features": [
    {"value": "feature text", "confidence": 0-100}
  ],
  "applications": [
    {"value": "application text", "confidence": 0-100}
  ],
  "specifications": [
    {"attribute": "string", "value": "string", "unit": "string or null", "confidence": 0-100}
  ],
  "dimensions": {
    "length": {"value": "string or null", "unit": "string or null", "confidence": 0-100},
    "width": {"value": "string or null", "unit": "string or null", "confidence": 0-100},
    "height": {"value": "string or null", "unit": "string or null", "confidence": 0-100},
    "depth": {"value": "string or null", "unit": "string or null", "confidence": 0-100},
    "weight": {"value": "string or null", "unit": "string or null", "confidence": 0-100}
  },
  "packaging": {
    "carton_length": {"value": "string or null", "unit": "string or null", "confidence": 0-100},
    "carton_width": {"value": "string or null", "unit": "string or null", "confidence": 0-100},
    "carton_height": {"value": "string or null", "unit": "string or null", "confidence": 0-100},
    "carton_weight": {"value": "string or null", "unit": "string or null", "confidence": 0-100},
    "units_per_pallet": {"value": "string or null", "confidence": 0-100},
    "units_per_case": {"value": "string or null", "confidence": 0-100}
  },
  "compliance_and_certifications": [
    {"value": "certification name", "confidence": 0-100}
  ],
  "country_of_origin": {"value": "string or null", "confidence": 0-100, "source": "string"},
  "warranty": {"value": "string or null", "confidence": 0-100, "source": "string"},
  "images": [
    {"url": "https://...", "alt_text": "string or null", "type": "product|lifestyle|infographic|certification|other"}
  ],
  "documents": [
    {"url": "https://...", "title": "string", "type": "datasheet|manual|instructions|sell_sheet|specification|other"}
  ],
  "videos": [
    {"url": "https://...", "title": "string or null"}
  ],
  "price_observed": {"value": "string or null", "currency": "string or null", "confidence": 0-100, "note": "string or null"},
  "additional_attributes": {
    "any_field_not_covered_above": {"value": "string", "confidence": 0-100}
  },
  "extraction_metadata": {
    "source_url": "[source_url]",
    "total_fields_extracted": 0,
    "overall_confidence": 0-100,
    "extraction_notes": "1-2 sentence summary of quality and any gaps",
    "fields_not_found": ["list", "of", "common", "fields", "absent"]
  }
}

FLEXIBILITY:
- Arrays (features, applications, specifications, certifications, images, documents) should contain ALL items found. No max limit.
- The additional_attributes object is your catch-all for anything that doesn't fit a defined field — use freely with descriptive keys.
- Keep JSON valid. No trailing commas, no comments, no ellipsis.

Return ONLY the raw JSON object. Do NOT wrap it in code fences (```json or ```). Do NOT add any preamble, explanation, or text before or after. Your entire response must be parseable by JSON.parse() starting at character 1.
```

**Validated test result (Intermatic DT200LT):**
- 75 fields extracted (65 in Prompt Builder test, 75 when fed live Firecrawl markdown)
- 95% overall confidence
- Zero hallucinations
- All 5 document PDFs, 3 certifications, 30 spec rows, 5 features, 3 applications, dimensions + packaging captured
- Null for UPC/SKU/weight (honestly reported in `fields_not_found`)

---

## R. AI Builder Prompt — `Generate Product Content`

**Location:** make.powerautomate.com → AI hub → Prompts

**Purpose:** Generate original SiteOne-voice marketing content (descriptions, feature bullets, SEO keywords) grounded in extracted specs. AVOIDS republishing manufacturer copy (copyright protection). Written for B2B trade audience.

**Inputs:**
| Name | Type | Sample value |
|---|---|---|
| `extracted_data` | Text | *(paste the full `text` output from `AI - Extract Product Data` — the nested JSON with product_identity, specifications, etc.)* |

**Model settings:**
- **Model:** GPT-5 or GPT-4.1 (highest available)
- **Temperature:** `0.4` (slight creativity encourages rewording vs. robotic paraphrasing)
- **Output format:** JSON if available

**Full prompt text:**

```
You are a content writer for SiteOne Landscape Supply, a B2B distributor serving landscape, irrigation, hardscape, and lighting contractors across North America. SiteOne sells to trade professionals — not homeowners.

Your job is to write fresh, original marketing content for a new product based on its extracted technical data. The content will be published on SiteOne.com and used in product listings.

INPUTS:
- Extracted product data (JSON): [extracted_data]

STRICT RULES:

1. ORIGINAL CONTENT ONLY. Do not copy or closely paraphrase text from the extracted_data. Rewrite entirely in your own words. The source description and features are manufacturer copy — we cannot republish them verbatim.

2. GROUND EVERY CLAIM. Only state facts supported by the extracted specs, features, applications, or certifications. Do NOT invent capabilities, benchmarks, or claims.

3. B2B TRADE VOICE. Write for landscape contractors, irrigation pros, and commercial buyers. Practical, direct, informative. Avoid:
   - Consumer marketing language ("amazing", "you'll love", "perfect for your home")
   - Superlatives without backing ("best-in-class", "industry-leading")
   - Second-person ("you") — use third-person or imperative
   - Exclamation points

4. NO COMPETITOR REFERENCES. Don't mention other brands, comparisons, or "better than X."

5. USE ACTUAL VALUES. When mentioning specs (voltage, dimensions, load ratings, etc.), use the exact values from the extracted_data.

6. PRIORITIZE APPLICATIONS. Emphasize where/how contractors would use this product.

LENGTH AND FORMAT:

- `short_description`: 1 sentence, 80-150 characters. Used in catalog listings.
- `long_description`: 2-3 short paragraphs, 150-250 words total. Plain text, no markdown headers.
- `feature_bullets`: Exactly 5-7 bullets. Each 8-15 words. Benefit-focused. Start with action verbs or key nouns, not "Feature includes..." or "This product has...".
- `seo_keywords`: 5-10 terms. Mix of product category, use cases, and technical attributes. Lowercase, comma-separated as array items.

OUTPUT FORMAT (JSON only, no preamble, no code fences, no markdown):

{
  "short_description": "string",
  "long_description": "string",
  "feature_bullets": ["string", "string", "string", "string", "string"],
  "seo_keywords": ["string", "string", "string"],
  "generation_metadata": {
    "word_count_long_description": 0,
    "bullet_count": 0,
    "generation_notes": "1 sentence on content quality and what was prioritized"
  }
}

Return ONLY the raw JSON object. Do NOT wrap it in code fences (```json or ```). Do NOT add any preamble, explanation, or text before or after. Your entire response must be parseable by JSON.parse() starting at character 1.
```

**Validated test result (Rain Bird 5000 Series rotor):**
- Long description 224 words, 7 feature bullets, 10 SEO keywords
- B2B voice, third-person, no "you", no exclamations, no superlatives
- All claims traceable to extracted specs
- Zero hallucinations

---

## S. AI Builder Prompt — `Flatten Product Data to CSV Row`

**Location:** make.powerautomate.com → AI hub → Prompts

**Purpose:** Transform the nested extraction JSON + generated content JSON into a single FLAT JSON object where every value is a top-level key. Output feeds directly into `Create CSV Table` action.

**Why this exists:** Products have variable-length arrays (specifications, feature bullets, images, documents). A fixed CSV column schema can't represent this cleanly. The flatten prompt creates dynamic column keys per product: each spec attribute becomes its own column, `feature_bullet_N` per bullet, `image_N_url/alt/type` per image, etc.

**Inputs:**
| Name | Type | Sample value |
|---|---|---|
| `extracted_data` | Text | *(paste full `text` output from `AI - Extract Product Data`)* |
| `generated_content` | Text | *(paste full output from `Parse JSON - Generated Content`)* |

**Model settings:**
- **Model:** GPT-4.1 (transformation task, no reasoning needed — GPT-5 would be overkill)
- **Temperature:** `0` (deterministic)
- **Output format:** JSON if available

**Full prompt text:**

```
You are a data transformation specialist. Your job is to flatten two nested JSON objects about a product into a single flat JSON object, where every piece of data becomes a top-level key suitable for a CSV column.

INPUTS:
- Extracted Product Data (JSON): [extracted_data]
- Generated Marketing Content (JSON): [generated_content]

TRANSFORMATION RULES:

1. OUTPUT A SINGLE FLAT JSON OBJECT. No nesting, no arrays. Every value must be a simple string, number, or null at the top level.

2. FIXED KEYS (always include, even if null):
   - product_name, brand, manufacturer_part_number, model_number, upc, sku
   - short_description (from generated_content)
   - long_description (from generated_content)
   - extracted_short_description (from extracted_data.description.short_description.value)
   - extracted_long_description (from extracted_data.description.long_description.value)
   - overall_confidence (from extracted_data.extraction_metadata.overall_confidence)
   - source_url (from extracted_data.extraction_metadata.source_url)

3. FEATURE BULLETS: Convert generated_content.feature_bullets array to flat keys feature_bullet_1, feature_bullet_2, feature_bullet_3... based on array length. No padding — include only as many as exist.

4. SEO KEYWORDS: Convert generated_content.seo_keywords array to seo_keyword_1, seo_keyword_2... No padding.

5. SPECIFICATIONS: For each item in extracted_data.specifications array, create ONE column using the attribute name as the key and the value as the value. If a unit exists, append it to the value (e.g., "3.307 in" not just "3.307"). Attribute names should be kept verbatim from the source (e.g., "Input Voltage Range(s)", "Product Width (in)", "Country of Origin"). Do NOT rename, normalize, or group them.

6. DIMENSIONS: If extracted_data.dimensions has values that are NOT already present as entries in specifications, include them too (prefix with "dimension_"). If they duplicate spec table entries, skip to avoid duplication.

7. PACKAGING: Same rule — include packaging fields only if they don't already appear in specifications. Prefix with "packaging_".

8. IMAGES: For each item in extracted_data.images, create three flat keys per image:
   - image_1_url, image_1_alt, image_1_type, image_1_source_domain
   - image_2_url, image_2_alt, image_2_type, image_2_source_domain
   - ...etc. No padding.

   For source_domain: extract the hostname from the image URL (e.g., from "https://www.intermatic.com/UserFiles/img/cULus.jpg", source_domain = "intermatic.com"; from "https://djuly1j3idynn.cloudfront.net/...", source_domain = "djuly1j3idynn.cloudfront.net"). Strip "www." prefix but keep the rest of the domain.

9. DOCUMENTS: For each item in extracted_data.documents:
   - document_1_url, document_1_title, document_1_type
   - document_2_url, document_2_title, document_2_type
   - ...etc. No padding.

10. VIDEOS: For each in extracted_data.videos:
    - video_1_url, video_1_title
    - video_2_url, video_2_title
    - ...etc.

11. CERTIFICATIONS: Collapse extracted_data.compliance_and_certifications array to a single comma-separated string in key "certifications" (e.g., "cULus, Green Energy, Title 20").

12. PRICE: price_observed_value, price_observed_currency (if present).

13. COUNTRY / WARRANTY: country_of_origin, warranty (if present, from their respective fields in extracted_data).

14. ADDITIONAL ATTRIBUTES: Any keys in extracted_data.additional_attributes become top-level columns directly.

15. DO NOT INCLUDE confidence scores, source citations, or metadata wrappers. Only clean values.

16. NULL HANDLING: If a fixed key's source value is null or missing, include it with null value. For dynamic keys (specs, bullets, images), just omit.

OUTPUT FORMAT: Single flat JSON object, no preamble, no code fences, no markdown.

Example output structure (abbreviated):
{
  "product_name": "Digital Astronomic Landscape Timer",
  "brand": "Intermatic",
  "manufacturer_part_number": "DT200LT",
  "short_description": "...",
  "long_description": "...",
  "feature_bullet_1": "Program ON at dusk...",
  "feature_bullet_2": "...",
  "seo_keyword_1": "...",
  "Input Voltage Range(s)": "120 VAC, 60 Hz",
  "Product Width (in)": "3.307 in",
  "Color": "Black",
  "Country of Origin": "MEXICO",
  "image_1_url": "https://...",
  "image_1_alt": "cULus",
  "image_1_type": "certification",
  "image_1_source_domain": "intermatic.com",
  "document_1_url": "https://...",
  "document_1_title": "DT200LT Technical Data Sheet",
  "document_1_type": "datasheet",
  "certifications": "cULus, Green Energy, Title 20",
  "overall_confidence": 95,
  "source_url": "https://..."
}

Return ONLY the raw JSON object. Do NOT wrap it in code fences (```json or ```). Do NOT add any preamble, explanation, or text before or after. Your entire response must be parseable by JSON.parse() starting at character 1.
```

**Status:** Validated via Intermatic + Brilliance runs in Step 27e flow tests. `image_N_source_domain` populated correctly in CSV and HTML preview output (when images are extracted at all — distributor lazy-load issue separately).

---

## P4. AI Builder Prompt — `Extract Enrichment Request from Email` *(NEW Step 30-1)*

**Location:** make.powerautomate.com → AI hub → Prompts

**Model:** GPT-4.1 mini (downgraded from initial GPT-4.1 recommendation; validated behavior on email parsing doesn't need full 4.1)

**Temperature:** 0

**Inputs:**

| Name | Type | Sample value (for testing) |
|---|---|---|
| `email_subject` | Text | `Onboard Intermatic DT200LT` |
| `email_body` | Text | `Please set up this product. Thanks.` |

**⚠️ GOTCHA:** Input names must NOT contain square brackets. The bracket notation `[email_subject]` is used inside the prompt text to REFERENCE variables — but the input NAMES themselves are plain `email_subject` and `email_body`. Getting this wrong causes `_5b` URL-encoding and `InvalidPredictionInput` errors.

**Full prompt text:**
```
You are an item-setup request parser. You receive an email from a SiteOne Landscape Supply employee who wants a product enriched. Extract structured information the enrichment pipeline needs.

INPUTS:
- Email Subject: [email_subject]
- Email Body: [email_body]

WHAT TO EXTRACT:

REQUIRED FIELDS:

1. BRAND — the manufacturer or vendor brand name. Examples: "Intermatic", "Rain Bird", "Hunter", "Brilliance", "FX Luminaire", "Toro", "Ewing".

2. PART_NUMBER — the manufacturer part number / SKU / model identifier. Examples: "DT200LT", "5004PC", "PGP-ADJ", "BRI-WIFI-SMART-SOCKET-3".

OPTIONAL FIELDS (extract if present, null if not):

3. PRODUCT_NAME — short descriptive product name if the sender mentions one. Examples: "Digital Astronomic Landscape Timer", "5000 Series Rotor Sprinkler", "Wi-Fi Smart Socket". NOT the same as brand+part# — this is the human-readable name. Only populate if the sender explicitly states it. Do not invent.

4. PROVIDED_URL — a direct URL to the product page if the sender pasted one. Examples: "https://www.intermatic.com/Product/DT200LT". Include only URLs that appear to point to a specific product page, not category pages, company homepages, or unrelated links. If the sender included multiple URLs, pick the one that best matches a product detail page.

5. REQUESTER_NOTES — any substantive context the requester included that downstream reviewers should see. Examples: "vendor said this is replacing DT100", "need this by Friday for a commercial quote", "customer has 500 units on hand", "different pricing than our last order". 
   - INCLUDE: requester's own substantive remarks, vendor notes they relayed, pricing/quantity context, urgency, special instructions
   - EXCLUDE: email signatures, quoted reply chains ("On [date], X wrote:"), auto-disclaimers, "thanks" or "please" without further context, greetings, typical email pleasantries
   - If no substantive notes, set to null (do NOT fabricate content)

EDGE CASES / FLAGS:

- MULTI-ITEM: If the email contains MORE THAN ONE clear brand+part# pair (e.g., "Please onboard Intermatic DT200LT AND Rain Bird 5004PC"), set `multi_item_detected` to true. Extract the FIRST pair only for brand/part_number. Current system processes one item per email.

- ATTACHMENTS: If the email references attachments ("see attached spec sheet", "datasheet attached", "PDF below") but brand/part# is NOT in the body text, set `attachment_likely` to true. The current system can't parse attachments yet.

- NOT A REQUEST: If the email is clearly NOT an item-setup request (auto-reply, meeting invite, general question, spam, out-of-office), set brand and part_number to null and explain in `parse_notes`.

- AMBIGUITY: If either brand OR part# is unclear/missing, set that specific field to null. Do not guess.

HOW TO PARSE:

- Requests can be terse: "Onboard Intermatic DT200LT"
- Or conversational: "Hey, can we get this one set up? It's an Intermatic DT200LT."
- Or forwarded vendor emails with noise, quoted text, signatures, disclaimers
- Or structured: "Brand: Intermatic, Part: DT200LT, Product: Digital Timer"
- Fields can appear in any order, on any line, in subject or body
- If the user uses labels like "Brand:", "Part:", "SKU:", "MFG:", "Model:", "MPN:", "URL:", "Name:" — trust those labels strongly

CONFIDENCE SCORING:

Score 0-100 based on overall extraction clarity for required fields:
- 95-100: brand + part# both clearly stated, unambiguous
- 80-94: Both present but required light interpretation
- 60-79: One field clear, the other inferred with some uncertainty
- Below 60: Significant ambiguity — brand or part# null, or highly uncertain extraction

OUTPUT FORMAT:

{
  "brand": "string or null",
  "part_number": "string or null",
  "product_name": "string or null",
  "provided_url": "string or null",
  "requester_notes": "string or null",
  "confidence": 0-100,
  "multi_item_detected": true or false,
  "attachment_likely": true or false,
  "parse_notes": "1-2 sentence explanation of extraction quality, flags raised, or why parsing failed"
}

Return ONLY the raw JSON object. Do NOT wrap it in code fences (```json or ```). Do NOT add any preamble, explanation, or text before or after. Your entire response must be parseable by JSON.parse() starting at character 1.
```

**Validation:** Tested Apr 18 via 4 scenarios (terse, conversational with context, structured with URL, non-request email). Validated by running live email test with Brilliance BRI-WIFI-SMART-SOCKET-3 → extracted at 98% confidence, triggered full pipeline successfully.

**Cost note:** Currently costs 68 credits per call at GPT-4.1 mini (seen in validation run). Moderate cost — fires on every email in monitored folder. At 50 emails/day, that's 3,400 credits/day for email parsing alone.

---

## P5. Design notes — Why these 5 fields (not more, not less)

The email prompt extracts only fields that map to existing flow capabilities. Fields NOT included and why:

- `priority` — flow doesn't route on priority today; dead data until we build routing logic
- `requester_name` — already captured automatically via Outlook trigger's From field
- `existing_sku` — would need new schema fields and HTML template changes; scope creep

The 5 fields chosen (`brand`, `part_number`, `product_name`, `provided_url`, `requester_notes`) map directly to either existing variables (brand, part_number) or production-valuable future fields (product_name for URL fallback, provided_url to bypass Serper, requester_notes for audit trail). See backlog items #45 (multi-item) and #46 (attachments) for planned v2 expansions.

---

## T. External Services / API Keys

### T1. Serper.dev
- **Endpoint:** `https://google.serper.dev/search`
- **Auth:** `X-API-KEY` header
- **Key location:** Currently inline in HTTP action headers (plaintext). ⚠️ Tech debt #5: move to environment variable.
- **Rate limit:** 25 requests/sec (visible in response header `x-ratelimit-limit`)
- **Free tier:** 2,500 searches

### T2. AI Builder
- **Connection:** Default AI Builder connection in the environment
- **Credit consumption:** GPT-4.1 ~16 credits per 1000 tokens; costs scale with payload size.
- **Tech debt #6:** Token-optimize by trimming Serper results before passing to AI.

### T3. Firecrawl
- **Endpoint:** `https://api.firecrawl.dev/v2/scrape`
- **Auth:** `Authorization: Bearer fc-...` header
- **Key location:** Currently inline in HTTP action headers. ⚠️ Tech debt #5: move to env var.
- **Purpose:** Handles JavaScript-rendered SPAs (like intermatic.com). Returns clean markdown instead of raw HTML.
- **Cost:** 1 credit per scrape (basic markdown). Free tier = 500 credits/month.
- **Rate limit:** 10 scrapes/min on free tier.

---

## U. Action name → internal reference name mapping

Power Automate converts action display names to internal names by replacing spaces with underscores. Use internal names in expressions. Current mapping:

| Display name | Internal name (for expressions) |
|---|---|
| `HTTP` | `HTTP` |
| `Parse JSON` | `Parse_JSON` |
| `AI - Select URLs` | `AI_-_Select_URLs` |
| `Parse JSON 1` | `Parse_JSON_1` |
| `Check - URL Qualifies` *(Condition, new Step 27c)* | `Check_-_URL_Qualifies` |
| `Terminate - Flag for Human Review` *(inside False branch)* | `Terminate_-_Flag_for_Human_Review` |
| `Firecrawl - Scrape Product Page` *(renamed Step 27c from "Scrape Manufacturer Page")* | `Firecrawl_-_Scrape_Product_Page` |
| `AI - Extract Product Data` | `AI_-_Extract_Product_Data` |
| `Parse JSON - Extraction Output` | `Parse_JSON_-_Extraction_Output` |
| `AI - Generate Content` | `AI_-_Generate_Content` |
| `Parse JSON - Generated Content` | `Parse_JSON_-_Generated_Content` |
| `AI - Flatten Row` | `AI_-_Flatten_Row` |
| `Parse JSON - Flat Row` | `Parse_JSON_-_Flat_Row` |
| `Create CSV Table` | `Create_CSV_table` |
| `OneDrive - Save CSV` | `OneDrive_-_Save_CSV` |
| `Loop - Download Images` (Apply to each) | `Apply_to_each` or `Apply_to_each_2` (check Code view) |
| `Increment - Image Counter` | `Increment_-_Image_Counter` |
| `HTTP - Download Image` | `HTTP_-_Download_Image` |
| `OneDrive - Save Image` | `OneDrive_-_Save_Image` |
| `Compose - Extraction Metadata` | `Compose_-_Extraction_Metadata` |
| `Compose - HTML Preview` | `Compose_-_HTML_Preview` |
| `OneDrive - Save HTML Preview` | `OneDrive_-_Save_HTML_Preview` |

**⚠️ Historical note:** `Init - Image Counter` used to live after Save CSV. In Step 27c it was moved to the top-level Variables block because Power Automate does not allow Initialize Variable inside a Condition. Internal name `Init_-_Image_Counter` is unchanged by the move.

**⚠️ Rename caveat:** Renaming the display name does NOT change the internal name. Check **Code view** of any action if expressions fail — that shows the true internal reference.

---

## V. Quick rebuild checklist (in order)

If the entire flow is lost, rebuild in this order:

1. Create all 4 AI Builder prompts FIRST (Sections P, Q, R, S) — prompts are reusable across flows. Use the Step 27e versions with strict "Return ONLY the raw JSON..." final directive.
2. Sign up for Serper.dev + Firecrawl, get API keys (Section T)
3. Create new Instant cloud flow, add 6 trigger inputs (Section B)
4. Add 7 Initialize Variable actions (Sections C1–C7) — INCLUDING `Init - Image Counter` at the TOP LEVEL (not nested)
5. Add HTTP (Serper) action (Section D) — paste Serper API key
6. Add Parse JSON (Section E)
7. Add Run a Prompt — `AI - Select URLs` (Section F)
8. Add Parse JSON 1 with Step 27b hardened schema AND defensive fence-stripping expression (Section G)
9. **Add Condition `Check - URL Qualifies` (Section G2)** — left operand `body('Parse_JSON_1')?['selected_url']` is not equal to null
10. **In False branch: add Terminate `Flag for Human Review` (Section G3)** — status Failed, code `NoQualifyingURL`
11. **All remaining steps below go INSIDE the True branch of the Condition.**
12. Add Firecrawl - Scrape Product Page (Section H) — paste Firecrawl API key, URL points to `selected_url`
13. Add AI - Extract Product Data (Section I) — source_url references `selected_url`
14. Add Parse JSON - Extraction Output with hardened schema + fence-strip expression (Section J)
15. Add AI - Generate Content (Section K)
16. Add Parse JSON - Generated Content with fence-strip expression (Section L)
17. Add AI - Flatten Row (Section M)
18. Add Parse JSON - Flat Row with fence-strip expression (Section N)
19. Add Create CSV Table (Section O)
20. Add OneDrive - Save CSV (Section O2) — authenticate OneDrive for Business first
21. Add Loop - Download Images with Concurrency=1 (Section P2):
    - Loop input uses `filter(body('Parse_JSON_-_Extraction_Output')?['images'], item()?['url'] != null)` expression
    - Inside loop: Increment - Image Counter → HTTP - Download Image → OneDrive - Save Image
22. Add Compose - Extraction Metadata (Section Q1) — Run After ✓ success + ✓ failed on Loop
23. Add Compose - HTML Preview (Section Q2) — paste template_v3.html contents; Run After ✓ success + ✓ failed
24. Add OneDrive - Save HTML Preview (Section Q3) — Run After ✓ success + ✓ failed
25. Test end-to-end with Intermatic/DT200LT (manufacturer path), Brilliance/BRI-WIFI-SMART-SOCKET-3 (distributor fallback path), and FakeBrand/XYZ-DOESNOTEXIST-999 (clean Terminate path). Delete OneDrive folder before each test to avoid 409 eTag issues. Verify `<Part>_preview.html` opens in browser with full visual rendering.

---

## V2. Image Agent architecture (Step 27g — under construction)

**Status:** Design locked Apr 18, build in progress. Replaces the narrower Path 1 approach (dedicated image prompt always-on) with a conditional fallback pattern.

### Design decisions

| Decision | Choice | Rationale |
|---|---|---|
| When does image agent run? | Only when text extraction returns < 2 images | Zero overhead on sites that work naturally; focused cost on failure cases |
| Architecture | Sequential fallback inside main flow (not parallel, not child flow) | Simplest; reuses all existing flow infrastructure |
| Coverage breadth v1 | HTML parsing + attribute preference + scope rules | ~80-85% of sites; Shopify/Magento/WooCommerce base64 cases handled |
| Quality bar | Governance-grade output with per-image confidence + audit trail | Consistent with existing text extraction quality bar |
| URL input | Same `selected_url` as text path | No duplicate Serper calls or URL selection work |
| Firecrawl strategy | Dedicated second Firecrawl call (`excludeTags` to strip bulk, `onlyMainContent: false` to preserve gallery regions) | Text extraction unchanged and safe; bounded HTML size for image prompt |
| Merge logic | When agent fires, its output fully replaces the `images` array in extraction payload | Single authoritative source; agent is purpose-built; no dedup question |

### Why rejected: earlier designs

1. **Option 2 (dual-input markdown+html to existing extract prompt):** HTML on Shopify sites is 200-500KB, exceeded GPT-4.1 context window when tested. Would also carry risk of AI drift between inputs, and text extraction quality could silently regress.

2. **Path 1 (dedicated image prompt, always-on):** Still hits HTML size limits on large sites. Also wastes Firecrawl credits on every run regardless of need.

3. **Flip `onlyMainContent: false` on existing Firecrawl call:** Would flood extract prompt with related-products carousels, site chrome, promotional banners. Trades missing data for wrong data — unacceptable for PIM.

4. **Pre-filter HTML in Power Automate via `replace()`:** Power Automate's `replace()` is literal-only (no regex). Can't cleanly strip `<script>...</script>` blocks with dynamic content. Fragile.

### Flow structure when built

```
Firecrawl - Scrape Product Page          (existing — unchanged)
    ↓
AI - Extract Product Data                 (existing — unchanged)
    ↓
Parse JSON - Extraction Output            (existing — unchanged)
    ↓
Condition - Check Image Coverage          (NEW)
    ├── TRUE (images array < 2):
    │     Firecrawl - Scrape for Images   (NEW — dedicated call)
    │     AI - Extract Product Images     (NEW — uses existing prompt, refined)
    │     Parse JSON - Image Agent Output (NEW)
    │     Compose - Merged Extraction     (NEW — splices agent images into extraction payload)
    │
    └── FALSE (images array ≥ 2):
          Compose - Passthrough           (NEW — emits unchanged extraction payload)
    ↓
[Downstream reads from whichever Compose ran]
AI - Generate Content                     (existing — input unchanged)
AI - Flatten Row                          (existing — input CHANGED to read from Compose)
... rest of flow unchanged
Loop - Download Images                    (existing — input CHANGED to read from Compose)
```

### New Firecrawl call specs (Scrape for Images)

Key differences from the existing Scrape Product Page call:
- `onlyMainContent: false` — preserves gallery regions that might be outside Firecrawl's "main content" detection
- `excludeTags: ["script", "style", "noscript", "iframe", "link", "meta"]` — strips bulk server-side to fit within GPT-4.1 context window
- `formats: ["html"]` — markdown not needed
- `waitFor: 5000` — same lazy-load tolerance as existing call
- Uses same `selected_url` as primary scrape

### Merge logic (Compose - Merged Extraction)

When agent fires (TRUE branch), the compose takes the full extraction JSON and replaces ONLY the `images` array with the agent's output. All text fields (product_identity, description, features, specifications, documents, videos, etc.) are preserved from text extraction as-is.

### Credit cost analysis

| Site pattern | Before agent | After agent |
|---|---|---|
| Works naturally (Intermatic, Rain Bird, most manufacturers) | 1 Firecrawl + ~247 AI | 1 Firecrawl + ~247 AI + 0 (FALSE branch skips agent) |
| Base64 / lazy-load (Sprinkler Supply Store, etc.) | 1 Firecrawl + ~247 AI (empty images) | 2 Firecrawl + ~247 AI + ~150 AI (agent extraction) |

Expected production mix once observed: if 80% of items hit FALSE branch, net cost increase is ~20% of runs × 1 extra Firecrawl credit + ~150 extra AI credits = modest.

### v1 prompt — VALIDATED Apr 18 on Brilliance BRI-WIFI-SMART-SOCKET-3

**Status:** Built in Step 27g-1, tested end-to-end in Step 27g-9 via Brilliance run that hit TRUE branch.

**Test result:**
- Input: Firecrawl HTML from `sprinklersupplystore.com/products/brilliance-bri-wifi-smart-socket-3-...` (100K prompt tokens after excludeTags filtering)
- Output: 3 real product images, all from `sprinklersupplystore.com/cdn/shop/files/Brilliance-BRI-WIFI-SMART-SOCKET-3-*_1000x1000.png`, all classified type=product, per-image confidence 97-98
- Scope judgment: 17 images correctly excluded (site logos, promotional icons, UI images), overall_confidence 98, zero scope contamination
- Cost: 2,052 AI Builder credits per run (101,428 total tokens, 583 completion)
- Model used: GPT-5 (`gpt-5-chat-2025-07-14`) — **Note:** original design intent was GPT-4.1 but prompt was saved with GPT-5 during 27g-1. Should be switched to GPT-4.1 for ~5x cost reduction; scope judgment task doesn't require GPT-5 reasoning depth.

**Prompt text:** See `Extract Product Images from HTML` in AI Builder (Step 27g-1 paste). Key design choices:
- 10-attribute preference order for URL extraction (src → data-src → data-srcset → srcset → data-original → data-lazy-src → data-image → data-zoom/data-zoom-image → data-large_image/data-large → data-bgset)
- Resolution ranking for srcset (prefer highest width descriptor or density)
- Explicit dedup rule (strip query strings for comparison, keep highest-res variant)
- Scope rules: INCLUDE primary-product containers (`product-gallery`, `product-image`, `product-main`, etc.), EXCLUDE recommendations/chrome/social/newsletter/loading indicators
- URL pattern indicators for exclusion: `?lssrc=related`, `&lssrc=related`, `lshst=product` (Shopify tracking params for related products)
- Per-image confidence 0-100 with 70 as minimum threshold (below 70 = excluded entirely)
- Classification: product/lifestyle/infographic/certification/packaging/installation/other
- Source domain extraction from URL
- Full extraction_metadata block with total counts, exclusion_summary, overall_confidence, notes

**Known refinement needed post-validation:**
- Switch model dropdown from GPT-5 to GPT-4.1 in AI Builder prompt settings
- Token cost at ~100K input per call is higher than expected (Shopify pages are structurally large even after script/style stripping). Future optimization could use Firecrawl `includeTags` to whitelist product-region selectors instead of blacklisting via excludeTags — but site-specific.

---

## V3. Cost Tracker architecture (Step 28 — COMPLETE)

**Status:** Built and validated on Intermatic DT200LT Apr 18. All instrumentation live in production flow.

### Variables (added to top-level variables block)

| Variable | Type | Initial | Purpose |
|---|---|---|---|
| `ai_builder_credits_total` | Integer | 0 | Accumulates `costAsAiBuilderCredits` from all AI Builder prompt responses |
| `firecrawl_credits_total` | Integer | 0 | Accumulates `creditsUsed` from Firecrawl responses |
| `serper_searches_total` | Integer | 0 | Count of Serper API calls (always 1 currently) |
| `google_api_calls_total` | Integer | 0 | Count of Gemini Nano Banana 2 calls (Step 32-3a). Used to compute `estimated_google_usd` at $0.045/call. |

### Increment actions (8 total)

Each placed directly AFTER the service call it tracks:

| Placement | Action name | Expression |
|---|---|---|
| After `HTTP` (Serper) | `Increment - Serper Searches` | `serper_searches_total` += `1` |
| After `Firecrawl - Scrape Product Page` | `Increment - Firecrawl Credits (Primary)` | `firecrawl_credits_total` += `coalesce(int(body('Firecrawl_-_Scrape_Product_Page')?['data']?['metadata']?['creditsUsed']), 1)` |
| After `AI - Select URLs` | `Increment - AI Credits (Select URLs)` | `ai_builder_credits_total` += `coalesce(int(body('AI_-_Select_URLs')?['responsev2']?['predictionOutput']?['costAsAiBuilderCredits']), 0)` |
| After `AI - Extract Product Data` | `Increment - AI Credits (Extract Data)` | `ai_builder_credits_total` += `coalesce(int(body('AI_-_Extract_Product_Data')?['responsev2']?['predictionOutput']?['costAsAiBuilderCredits']), 0)` |
| After `Firecrawl - Scrape for Images` (TRUE branch only) | `Increment - Firecrawl Credits (Images)` | `firecrawl_credits_total` += `coalesce(int(body('Firecrawl_-_Scrape_for_Images')?['data']?['metadata']?['creditsUsed']), 1)` |
| After `AI - Extract Product Images` (TRUE branch only) | `Increment - AI Credits (Image Agent)` | `ai_builder_credits_total` += `coalesce(int(body('AI_-_Extract_Product_Images')?['responsev2']?['predictionOutput']?['costAsAiBuilderCredits']), 0)` |
| After `AI - Generate Content` | `Increment - AI Credits (Generate Content)` | `ai_builder_credits_total` += `coalesce(int(body('AI_-_Generate_Content')?['responsev2']?['predictionOutput']?['costAsAiBuilderCredits']), 0)` |
| After `AI - Flatten Row` | `Increment - AI Credits (Flatten Row)` | `ai_builder_credits_total` += `coalesce(int(body('AI_-_Flatten_Row')?['responsev2']?['predictionOutput']?['costAsAiBuilderCredits']), 0)` |
| After `HTTP - Gemini Lifestyle 1` (TRUE branch of `Check - Has Reference Image`) | `Increment - Google API Calls` *(Step 32-3b)* | `google_api_calls_total` += `1` (plain integer — Gemini doesn't return a credit/cost field in response body, so we count calls and multiply by $0.045/call in Compose - Cost Metadata) |

### Compose - Cost Metadata

Placed directly BEFORE `Compose - Extraction Metadata`. Builds the unified object consumed by the HTML template.

**Inputs (plain-text with inline @{} interpolation, NOT fx expression):**

```
{
  "ai_builder_credits": @{variables('ai_builder_credits_total')},
  "firecrawl_credits": @{variables('firecrawl_credits_total')},
  "serper_searches": @{variables('serper_searches_total')},
  "estimated_ai_builder_usd": @{mul(variables('ai_builder_credits_total'), 0.001)},
  "estimated_firecrawl_usd": @{mul(variables('firecrawl_credits_total'), 0.003)},
  "estimated_serper_usd": @{mul(variables('serper_searches_total'), 0.0003)},
  "estimated_total_usd": @{add(add(mul(variables('ai_builder_credits_total'), 0.001), mul(variables('firecrawl_credits_total'), 0.003)), mul(variables('serper_searches_total'), 0.0003))},
  "run_duration_seconds": @{div(sub(ticks(utcNow()), ticks(variables('run_timestamp'))), 10000000)},
  "run_start_iso": "@{variables('run_timestamp')}",
  "run_end_iso": "@{utcNow()}",
  "image_agent_fired": @{if(less(length(coalesce(body('Parse_JSON_-_Extraction_Output')?['images'], createArray())), 2), true, false)},
  "conversion_rates_disclaimer": "Estimated USD based on approximate rates: AI Builder $0.001/credit, Firecrawl $0.003/credit, Serper $0.0003/search. Actual costs depend on your specific plans."
}
```

### Conversion rates (verified Apr 18, 2026)

| Service | Rate | Source |
|---|---|---|
| AI Builder credits | $0.001/credit | Microsoft licensing ballpark (varies by plan) |
| Firecrawl credits | $0.003/credit | Verified Firecrawl 2026 Standard plan pricing (100K credits/$83/mo) |
| Serper searches | $0.0003/search | Verified serper.dev 2026 base pricing ($0.30/1000 queries) |

**Hardcoded in the Compose expression.** To update rates: edit the Compose action inputs directly. Could be promoted to environment variables if they change frequently.

### HTML template changes (compose_template_v4.html)

**Data injection:** New `<script type="text/plain" id="cost-data-b64">` tag paired with existing flat-data and extraction-data. Points to `@{base64(string(outputs('Compose_-_Cost_Metadata')))}`.

**Banner restructured:** Removed the right-side confidence-only display. Banner now product identity only.

**Three-KPI strip added** immediately after banner (CSS `.kpi-strip` class). Three columns:
1. **Confidence %** (color-coded high/mid/low) + subtitle describing tier readiness
2. **Run Duration** (formatted as Ns or Nm Ns) + start timestamp subtitle
3. **Cost to Execute** (estimated USD, accent colored) + image-agent-fired subtitle

**Cost Breakdown section** added before footer (CSS `.cost-breakdown`). Three itemized cards:
- AI Builder: credits + estimated USD
- Firecrawl: credits + estimated USD
- Serper Search: count + estimated USD
- Disclaimer strip below ("Estimated USD based on approximate rates...")

**Footer expanded:** Added Run Start / Run End / all 3 credit totals / image_agent status / fields extracted / overall confidence / total cost (accent-colored).

**Mobile media query updated:** KPI strip + cost breakdown collapse to single column under 720px.

### Validated run (Intermatic DT200LT)

- AI Builder credits: 456
- Firecrawl credits: 1
- Serper searches: 1
- Estimated AI Builder: $0.46
- Estimated Firecrawl: <$0.01
- Estimated Serper: <$0.01
- **Total estimated: $0.46**
- Image agent: `not needed` (FALSE branch, text extraction found ≥2 images)

### Expected Brilliance run (TRUE branch, image agent fires)

With GPT-5 currently on image prompt (Y12 — should be flipped to GPT-4.1):
- AI Builder: ~2,500 credits ($2.50) — Extract Product Images alone is ~2,052
- Firecrawl: 2 credits ($0.006) — primary + image scrape
- Serper: 1 search (<$0.01)
- **Total estimated: ~$2.51**

After Y12 fix (GPT-4.1 on image prompt):
- AI Builder: ~850 credits ($0.85)
- Firecrawl: 2 credits ($0.006)
- Serper: 1 search (<$0.01)
- **Total estimated: ~$0.86**

---



## W. Not yet built (as of this doc)

Reference `HANDOFF.md` section 6 for full roadmap. Actions not yet in flow:
- Lifestyle image generation (Runware API integration) — Step 28-alt pending
- Email trigger (replaces manual trigger) — Step 29 pending
- State management fixes (#44) — NEXT PRIORITY per Tony Apr 18
- Distributor scrape loop (multi-source corroboration) — deferred; current design scrapes single best URL
- All error handling Scope blocks (#3)
- All logging actions (SharePoint) (#8)
- Retry policy + timeout hardening on all HTTP actions (#4)
- Environment variables for API keys (#5)
- Serper result trim / Select action (#6)
- Fallback search with product_name (#18)
- AI Compare prompt for dual-search results (#20)
- Firecrawl `actions` for JS-rendered galleries (#31) — partially addressed via image agent (#27g)
- Mexico-subdomain detection / URL normalization (#32) — partially addressed in Step 27a prompt (prefers US English domains)
- Firecrawl cache busting (#33)
- Image Content-Type → extension mapping (#34)
- Folder pre-cleanup before run / 409 eTag handling (#35) — consolidating into #44
- Image source blacklist (#41) — `image_N_source_domain` field ready; blacklist logic deferred

**Completed in Step 27 (no longer pending):**
- ✅ #23 Confidence-first URL selection — Sections F, G (new output shape)
- ✅ #24 Null URL guard — Sections G2, G3 (Condition + Terminate)
- ✅ #40 Defensive JSON fence-stripping — all 4 Parse JSON sections + all 4 prompt final directives
- ✅ #42 Null-URL filter in image loop — Section P2

**Completed in Step 27g (image agent):**
- ✅ #31 (partial — v1 scope) — Conditional image agent via `Check - Image Coverage`, dedicated `Firecrawl - Scrape for Images`, `AI - Extract Product Images` prompt, `Compose - Merged Extraction`. See Section V2.

**Completed in Step 28 (cost tracker):**
- ✅ #39 Cost & time metrics tracking — full implementation in Section V3 (3 variables, 8 Increments, Compose - Cost Metadata, template v4 with 3-KPI banner + Cost Breakdown + expanded footer). Validated Intermatic at $0.46/run.

---

## Y. Known Issues & Gotchas (important for continued work)

### Y1. OneDrive 409 eTag mismatch *(RESOLVED Step 29-1 via correlation_id subfolders)*
**Symptom (historical):** `OneDrive - Save Image` or `OneDrive - Save CSV` returned 409 status: "The resource has changed since the caller last read it"  
**Cause:** A file with the same name existed from a prior failed/successful run in the same folder. Compounded by OneDrive connector's stale eTag header within the image download loop.  
**Resolution:** Step 29-1 updated `item_folder_path` to include `correlation_id` subfolder. Every run writes to a unique path. No manual folder delete needed between runs. Historical runs preserved as natural audit trail.  
**Deferred:** OneDrive cleanup policy for accumulated subfolders (at 50 items/day, ~1,500 subfolders/month). Not blocking for POC/validation phase.

### Y2. Expression-as-text variable init
**Symptom:** OneDrive creates folders with literal names like `concat('/Item Setup POC/'...)`  
**Cause:** Variable was initialized by typing the expression as plain text in Value field, not using the fx expression picker  
**Fix:** Open `Init - Item Folder Path` → Value → click fx icon → paste expression → click OK → purple chip appears. Same applies to filename fields everywhere.

### Y3. iterationIndexes() unreliable
**Symptom:** "Unable to process template language expressions" when using `iterationIndexes('Apply_to_each')`  
**Cause:** Function is flaky in newer Power Automate versions  
**Workaround:** Use a counter variable (Init → Increment inside loop). Requires Concurrency=1.

### Y4. Serper URL variance per run
**Symptom:** Different runs return different top results; sometimes Mexico subdomain (`mx.intermatic.com`) instead of US  
**Cause:** Google SERP is non-deterministic  
**Fix:** Confidence-first URL selection (#23) would help by making the AI selector prefer known-good US domains. Short-term: `provided_url` trigger input allows manual override.

### Y5. Firecrawl cache hits
**Symptom:** Response includes `"cacheState": "hit"` — returning cached (potentially stale or wrong-subdomain) content  
**Cause:** Firecrawl caches aggressively on same URL  
**Fix:** No clean cache-bust param in v2/scrape API. Workaround: pass `provided_url` with tracking param `?setcontextlanguagecode=en&t=<timestamp>` to force fresh scrape. Backlog #33.

### Y6. LLM markdown code-fence wrapping *(fixed Step 27)*
**Symptom:** `Parse JSON` action fails with `Unexpected character encountered while parsing value: ` at position 0.  
**Cause:** AI Builder prompt response occasionally wrapped in ```` ```json ... ``` ```` markdown code fences despite prompt saying "no code fences". LLM output is not 100% deterministic — even temperature 0 can drift.  
**Fix applied in Step 27:**
1. Updated all 4 prompts with strict final directive: "Return ONLY the raw JSON object. Do NOT wrap it in code fences..." (see Sections P, Q, R, S)
2. Defensive wrapping on all 4 Parse JSON Content fields using nested `replace()` expressions to strip fences + newlines pre-parse. Example (from Parse JSON 1):
   ```
   replace(replace(replace(body('AI_-_Select_URLs')?['responsev2']?['predictionOutput']?['text'], '```json', ''), '```', ''), decodeUriComponent('%0A'), '')
   ```

### Y7. Default run-after cascades failures downstream
**Symptom:** A single failure (e.g., one image 409) causes ALL downstream actions to skip, including unrelated ones like the HTML preview generation.  
**Cause:** Power Automate's default Run After behavior is `is successful` only. When a preceding action fails, the current action is silently skipped.  
**Fix:** On actions that should run regardless of upstream failures, configure three-dots menu → Run After → check BOTH `is successful` AND `has failed`. Applied to Compose - Extraction Metadata, Compose - HTML Preview, OneDrive - Save HTML Preview.

### Y8. Incomplete image capture — manufacturer React SPAs (partial fix via waitFor)
**Current state:** With `waitFor: 5000`, Intermatic DT200LT yields 4 product images (different angles). **But the page actually has 14 images.** Gallery lazy-loading continues after the 5-second wait window, so ~10 images per product are still missed.  
**Cause:** React SPA gallery loads images progressively as user scrolls/interacts. Firecrawl's static scrape catches only what's in DOM at capture time.  
**Priority:** IMPORTANT for production quality. Tony flagged this as a must-fix: "I would want all 14 images not just the 4."  
**Fix options (see HANDOFF.md backlog #31):** increase waitFor, Firecrawl `actions` with scroll, wait-for-selector, or two-pass scrape strategy. Per-site tuning likely required.

### Y9. Init Variable inside Condition/Loop fails
**Symptom:** Flow save fails with `InvalidVariableInitialization` error: "The variable action 'Init_-_...' of type 'InitializeVariable' cannot be nested in an action of type 'Check_-_URL_Qualifies'."  
**Cause:** Power Automate design rule — Initialize Variable actions must live at the top level of the flow. They cannot be inside Conditions, Loops, Scopes, or any other container.  
**Fix:** Move the Init Variable action to the top-level variables section of the flow. Hit during Step 27c when moving actions into the True branch of the Condition — `Init - Image Counter` had to relocate from its former position to section C7 (top-level Variables).

### Y10. Distributor lazy-load / base64-embedded images *(RESOLVED Apr 18 via Step 27g image agent)*
**Symptom:** Brilliance BRI-WIFI-SMART-SOCKET-3 run through Sprinkler Supply Store distributor returned image entries with `url: null` — extraction was dropping them because Shopify embeds main product images as inline base64 data URIs, which Firecrawl strips by design (`<Base64-Image-Removed>` placeholder in markdown).  
**Root cause:** Not lazy-load as initially suspected. Shopify Dawn-family themes embed the primary gallery images as inline base64 for instant-zoom. Firecrawl strips data URIs to keep response sizes manageable. The markdown-only extraction path never sees real URLs on these sites.  
**Resolution:** Step 27g — conditional image agent. When text extraction returns < 2 images, a dedicated second Firecrawl call (with `excludeTags` + `onlyMainContent: false`) fetches HTML, and a specialized image-extraction prompt (`Extract Product Images from HTML`, GPT-4.1) parses `<img>` tags with attribute preference ordering, resolution ranking, dedup, and scope rules. Agent output replaces extraction's empty images array via `Compose - Merged Extraction` using `setProperty()`.

### Y11. `filter()` function NOT supported in Power Automate expressions *(discovered Step 27g-8)*
**Symptom:** `InvalidTemplate. Unable to process template language expressions... The template function 'filter' is not defined or not valid.`  
**Cause:** Unlike Azure Logic Apps (which DOES support `filter(array, condition)`), Power Automate workflows do not expose `filter()` in the expression library. This is not documented clearly anywhere — most tutorials mix Logic Apps and Power Automate syntax.  
**What doesn't work:**
```
filter(array, item()?['url'] != null)
```
**Workarounds:**
1. **Trust source data** — have prompts/upstream ensure no null URLs arrive downstream. Simplest; used in Step 27g-8 since image prompts explicitly instruct "skip images with no valid URL."
2. **Use Select action** — a standalone action (not an expression function) that maps an array. Supports conditional logic via `if()`.
3. **Use Apply to each with nested Condition** — loops over array, runs child actions only if condition met. Most verbose but most flexible.
4. **Use Power Fx formulas** — newer Power Automate flows can use Power Fx (different syntax) where filter-like operations work differently. Not broadly adopted yet.

**Functions that DO work in Power Automate expressions (partial list):** `coalesce()`, `createArray()`, `length()`, `first()`, `last()`, `take()`, `skip()`, `union()`, `intersection()`, `split()`, `join()`, `contains()`, `empty()`, `equals()`, `if()`, `setProperty()`, `addProperty()`, `removeProperty()`, `replace()`, `substring()`, `indexOf()`, `string()`, `json()`, `base64()`, `decodeUriComponent()`, `array()`.

### Y12. Image agent prompt model set to GPT-5 instead of GPT-4.1 *(RESOLVED Apr 19 by Tony)*
**Symptom:** Cost of image agent runs is 2,052 AI Builder credits per call — roughly 8x the cost of the main text extraction prompt (which uses GPT-5 for 75-field extraction at 247 credits). The image extraction is a much smaller task.  
**Cause:** When Tony saved the `Extract Product Images from HTML` prompt in Step 27g-1, the model was set to GPT-5 (`gpt-5-chat-2025-07-14`) instead of GPT-4.1. This was not the original design intent — the task (HTML `<img>` tag parsing + scope judgment) doesn't require GPT-5 reasoning depth.  
**Fix applied Apr 19:** Tony flipped the model dropdown in AI Builder → Prompts → `Extract Product Images from HTML` to GPT-4.1. Zero prompt text changed.  
**Expected impact:** ~5x reduction in AI Builder credits on TRUE-branch runs. Brilliance run cost should drop from ~$2.50 to ~$0.86. Not re-validated end-to-end yet; re-verify during Step 32 validation.

### Y13. PA expression validator rejects `$content` syntax for binary HTTP body *(discovered during Step 32-2e build)*
**Symptom:** Pasting `body('HTTP_-_Download_Reference_Image')$content` into a Body field triggers `The input parameter(s) of operation '...' contains invalid expression(s). Fix invalid expression(s)...` validation error that blocks save.  
**Cause:** `$content` is legacy Azure Logic Apps syntax for accessing the base64-encoded content of a binary HTTP response. Power Automate's expression validator no longer accepts it (was accepted in older versions; flow-definition compatibility broken sometime pre-2026).  
**Fix:** Use `base64(body('HTTP_-_Download_Reference_Image'))` instead. The `base64()` function takes the binary response body and emits a base64 string inline. Functionally equivalent output; validator accepts it.  
**Applied in:** Step 32-2e `HTTP - Gemini Lifestyle 1` body, `inline_data.data` field.  
**Broader lesson:** When copying HTTP body syntax from Logic Apps documentation or StackOverflow, audit for `$content`, `$content-type`, and similar `$`-prefixed property accessors — they may not work.

### Y14. Gemini response `parts` array has 1 element (image at `[0]`), not 2 elements (text + image at `[1]`) *(RESOLVED Apr 19 via coalesce expression)*
**Symptom:** `OneDrive - Save Lifestyle Image 1` failed with: `InvalidTemplate. Unable to process template language expressions... The template language expression 'base64ToBinary(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates'][0]?['content']?['parts'][1]?['inlineData']?['data'])' cannot be evaluated because array index '1' is outside bounds (0, 0) of array.`

**Initial hypotheses (from build docs — all wrong except #2):**
1. SAFETY block (`finishReason: "SAFETY"` or `"IMAGE_SAFETY"`)
2. ✅ **ACTUAL CAUSE — Different parts shape** (parts[0] instead of parts[1])
3. `promptFeedback.blockReason` at top level
4. Schema mismatch

**Diagnostic method (reusable pattern — documented here because response body was 1.8MB and couldn't be pasted to Claude or easily read in PA run history):**

Added an interstitial `Compose - Gemini Debug` action BETWEEN `Parse JSON - Gemini Lifestyle 1` and `OneDrive - Save Lifestyle Image 1`. Inputs:

```
{
  "candidate_count": @{length(coalesce(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates'], createArray()))},
  "finish_reason": "@{coalesce(string(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['finishReason']), 'NULL')}",
  "parts_length": @{length(coalesce(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['content']?['parts'], createArray()))},
  "prompt_feedback": "@{coalesce(string(body('Parse_JSON_-_Gemini_Lifestyle_1')?['promptFeedback']), 'NULL')}",
  "candidate_safety_ratings": "@{coalesce(string(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['safetyRatings']), 'NULL')}"
}
```

Output from Brilliance test run Apr 19:
```json
{
  "candidate_count": 1,
  "finish_reason": "STOP",
  "parts_length": 1,
  "prompt_feedback": "",
  "candidate_safety_ratings": ""
}
```

`finish_reason: "STOP"` rules out safety block. `parts_length: 1` reveals the actual cause: only one element in the array, and our expression was indexing `[1]`.

**Root cause:** Gemini's `generateContent` response shape varies even with `responseModalities: ["TEXT", "IMAGE"]` configured. Sometimes the model returns BOTH a text part and an image part (2-element array, image at `[1]`); other times it skips the text narration and returns just the image (1-element array, image at `[0]`). Undocumented behavior. Cannot be forced via config.

**Fix applied:** Rewrite the File Content expression in `OneDrive - Save Lifestyle Image 1` to use `coalesce()` across both possible indices:

```
base64ToBinary(coalesce(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['content']?['parts']?[0]?['inlineData']?['data'], body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['content']?['parts']?[1]?['inlineData']?['data']))
```

Behavior: coalesce tries `parts[0].inlineData.data` first; if that is null (because parts[0] is a text part), falls through to `parts[1].inlineData.data`. Works regardless of which shape Gemini returns.

**Validated Apr 19 on Brilliance run:** PNG saved to `/lifestyle/BRI-WIFI-SMART-SOCKET-3_lifestyle_1.png`, image quality confirmed.

**Diagnostic Compose retained in flow** (`Compose - Gemini Debug`) — useful if Gemini introduces a third response shape variant. Remove before production deployment.

**Lesson — the diagnostic pattern is reusable:** Any time a large external API response can't be inspected directly (too big for run history render, too big for LLM context), an interstitial Compose with targeted `coalesce()` + `length()` + `string()` extractions turns the opaque blob into a 5-field decision tree.

### Y15. Gemini image generation is stochastic; same prompt yields different images *(documented during Step 32-1 design)*
**Symptom:** Two HTTP calls with identical prompt + identical reference image produce two different generated images.  
**Cause:** By design — Gemini uses sampling during inference. Normal and desired for variety.  
**Implication:** If we ever build 2-image variant, we can simply call Gemini twice with the same prompt. Don't need to engineer prompt variations. (v1 Step 32 scope is 1 image only, per Tony Apr 19.)

---


## X. APPENDIX — Upgrade path: CSV to proper XLSX via Office Scripts

**Status:** Not implemented. Current POC saves as CSV (opens in Excel, fully functional). This appendix documents how to upgrade when/if true XLSX deliverable quality is needed.

### Why consider the upgrade
- Real .xlsx file instead of .csv
- Bold headers, auto-fitted columns, proper formatting
- Multiple sheets possible (Main, Specs, Images, Documents)
- More professional deliverable for leadership
- Tradeoff: ~30 min extra one-time setup

### Prerequisites
1. Flatten prompt (Section S) must be working — Office Script expects flat JSON input
2. OneDrive for Business connector already configured (we already use this for CSV)
3. Excel Online (Business) connector available in Power Automate

### One-time setup steps

**Step 1: Create the template file**
1. Open OneDrive → navigate to `/Item Setup POC/` folder
2. Create new blank Excel workbook → name it `Template.xlsx`
3. Leave it blank (the Office Script will write everything)
4. Close file

**Step 2: Create the Office Script**
1. Open Excel Online (`excel.office.com`)
2. Open `Template.xlsx`
3. Click **Automate** tab → **New Script**
4. Name: `WriteFlatProductRow`
5. Paste this script:

```typescript
function main(workbook: ExcelScript.Workbook, jsonData: string): void {
    const data = JSON.parse(jsonData);
    const keys = Object.keys(data);
    const values: string[] = keys.map(k => {
        const v = data[k];
        if (v === null || v === undefined) return '';
        if (typeof v === 'object') return JSON.stringify(v);
        return String(v);
    });
    
    const sheet = workbook.getActiveWorksheet();
    
    // Clear any existing content
    const used = sheet.getUsedRange();
    if (used) { used.clear(); }
    
    // Write headers (row 1)
    const headerRange = sheet.getRangeByIndexes(0, 0, 1, keys.length);
    headerRange.setValues([keys]);
    headerRange.getFormat().getFont().setBold(true);
    headerRange.getFormat().getFill().setColor("#D9E2F3");
    
    // Write data (row 2)
    const dataRange = sheet.getRangeByIndexes(1, 0, 1, values.length);
    dataRange.setValues([values]);
    
    // Auto-fit columns
    sheet.getUsedRange().getFormat().autofitColumns();
    
    // Freeze header row
    sheet.getFreezePanes().freezeRows(1);
}
```

6. Click **Save script**
7. Test with sample JSON: click **Run** → provide a test JSON string in the `jsonData` parameter

### Flow modifications needed

Replace the current "Create CSV Table → OneDrive Save CSV" sequence with:

**Action 1: Copy Template → OneDrive for Business "Copy file"**
- Source File: `/Item Setup POC/Template.xlsx`
- Destination File Path (Expression): `concat(variables('item_folder_path'), '/', variables('part_number'), '.xlsx')`

**Action 2: Run the script → Excel Online (Business) "Run script"**
- Location: OneDrive for Business
- Document Library: OneDrive
- File (Expression): `concat(variables('item_folder_path'), '/', variables('part_number'), '.xlsx')`
- Script: `WriteFlatProductRow`
- jsonData (Expression): `body('AI_-_Flatten_Row')?['responsev2']?['predictionOutput']?['text']`
  *(Or whatever the internal name of the flatten prompt action is)*

### Things that can go wrong
- File locking: If the template is open in another user's Excel, the copy fails. Ensure template is always closed.
- Permissions: Office Scripts require user-level Excel Online permissions — depends on SiteOne M365 tenant config.
- Script not visible in Power Automate: After creating the script, wait 2-3 minutes for it to sync; may need to refresh the connector.
- Column count limit: Excel supports 16,384 columns per sheet. Highly unlikely to hit, but if a product has more unique spec attributes than that, script would need to overflow to additional sheets.

### Multi-sheet extension (future)
If you later want separate sheets (Main, Specifications, Images, Documents) instead of one wide row:
- Modify script to accept multiple JSON inputs (main, specs, images, docs)
- Script writes each to its own named sheet via `workbook.addWorksheet("Specs")` etc.
- Flow needs separate flatten prompts per sheet OR one prompt that returns nested structure

### If the upgrade ever happens
1. Build + test Office Script in Excel Online
2. Copy template to a fresh flow test
3. Verify per-product .xlsx files appear in OneDrive with correct formatting
4. Only then remove the CSV actions from the flow

---

## Z. Step 32 — Gemini Nano Banana 2 lifestyle image generation *(32-2 COMPLETE Apr 19; 32-3/4/5 PENDING)*

**Status:** Actions 32-2a through 32-2g all built, saved, and validated end-to-end on Brilliance run Apr 19. PNG saved to OneDrive. Y14 resolved via coalesce expression. Resume build at 32-3 (cost tracker wiring).

**Model:** `gemini-3.1-flash-image-preview` (Nano Banana 2, Google's latest image model as of Feb 26, 2026). Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-image-preview:generateContent`. Auth via `?key=API_KEY` query string. No Vertex AI / GCP project required — AI Studio API key works.

**Cost estimate:** ~$0.045 per image call. v1 scope = 1 image per product = ~$0.045 added per run. At 50 items/day → ~$67.50/month.

**Design choices:** See HANDOFF Section 8 decisions log for the rationale on Google-over-Runware, static prompt template, separate subfolder / separate HTML section / not in CSV, and 1-image-v1 scope.

### Flow position

Actions placed AFTER the existing `Loop - Download Images` completes. Lifestyle generation is a "side-branch subsystem" that reads from the unified images array but doesn't block the main pipeline. Everything lives inside the TRUE branch of a Condition that null-guards on "has product-type reference image."

```
...existing actions through Loop - Download Images complete...
    ↓
Z-1. Filter array - Product Images Only       (new)
Z-2. Compose - Reference Image URL            (new)
Z-3. Check - Has Reference Image (Condition)  (new)
    ├── TRUE:
    │   Z-4. HTTP - Download Reference Image    (new)
    │   Z-5. HTTP - Gemini Lifestyle 1          (new)
    │   Z-6. Parse JSON - Gemini Lifestyle 1    (new)
    │   Z-6b. Compose - Gemini Debug            (new — diagnostic, remove before prod)
    │   Z-7. OneDrive - Save Lifestyle Image 1  (new, Y14-fixed)
    │   [future: Compose - Lifestyle Metadata for HTML preview — Step 32-3+]
    └── FALSE:
        (empty; no lifestyle generation if no product-type reference available)
    ↓
...existing Compose - Cost Metadata / HTML preview / email send...
```

### Z-1. `Filter array - Product Images Only`

**Action type:** Data Operations → Filter array

**Position:** Immediately after `Loop - Download Images` completes (at top level of flow, outside any Condition).

**Configuration:**

| Field | Value |
|---|---|
| **From** | Dynamic content → Outputs of `Compose - Images to Download` |
| **Filter query left operand (Expression)** | `item()?['type']` |
| **Operator** | `is equal to` |
| **Filter query right operand** | `product` (plain text, no expression) |

**Output:** Array of image objects where `type` field equals `"product"`. Excludes certification badges, infographics, packaging shots, install diagrams, lifestyle shots already in the catalog, or anything else the extract/image-agent prompts tagged with a different type. Array may be empty if no product-type images were extracted.

**Safety check:** Filter array is read-only — it does NOT modify the `Compose - Images to Download` output. The main pipeline (image download loop, HTML preview, CSV export) continues to see all images. Lifestyle branch is a read-only consumer.

### Z-2. `Compose - Reference Image URL`

**Action type:** Data Operations → Compose

**Position:** Immediately after Z-1.

**Configuration:**

| Field | Value |
|---|---|
| **Inputs (Expression)** | `first(body('Filter_array_-_Product_Images_Only'))?['url']` |

**Output:** Single URL string (the `url` field of the first product-type image), OR `null` if the filter returned empty. Null is specifically what Z-3 checks.

### Z-3. `Check - Has Reference Image` (Condition)

**Action type:** Control → Condition

**Position:** Immediately after Z-2.

**Configuration:**

| Field | Value |
|---|---|
| **Left operand (Expression)** | `outputs('Compose_-_Reference_Image_URL')` |
| **Operator** | `is not equal to` |
| **Right operand (Expression)** | `null` |

**TRUE branch:** contains Z-4 through Z-7 (and future Z-8+ for metadata/HTML).
**FALSE branch:** empty. Product enriched with no lifestyle imagery — still delivers normal flow output.

### Z-4. `HTTP - Download Reference Image`

**Action type:** HTTP (standard connector, same one used for Serper/Firecrawl)

**Position:** First action inside TRUE branch of Z-3.

**Configuration:**

| Parameter | Value |
|---|---|
| **Method** | `GET` |
| **URI (Expression)** | `outputs('Compose_-_Reference_Image_URL')` |
| **Headers** | blank |
| **Queries** | blank |
| **Body** | blank |

**Settings tab:**

| Setting | Value |
|---|---|
| **Retry Policy** | Exponential Interval (default) |
| **Action Timeout** | `PT60S` |
| **Secure Inputs** | OFF |
| **Secure Outputs** | OFF |

**Output:** Binary response body (the raw bytes of the source image). Used as input to Z-5 via `base64(body(...))` expression.

### Z-5. `HTTP - Gemini Lifestyle 1`

**Action type:** HTTP

**Position:** Immediately after Z-4, still in TRUE branch.

**Configuration:**

| Parameter | Value |
|---|---|
| **Method** | `POST` |
| **URI** | `https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-image-preview:generateContent?key=GEMINI_API_KEY` (plain text with literal key appended; Tech debt #5 — move to env var) |
| **Headers** | `Content-Type` : `application/json` |
| **Body** | See block below — paste as-is with `@{}` expressions embedded (PA resolves at runtime) |

**Body JSON:**

```json
{
  "contents": [
    {
      "parts": [
        {
          "text": "Professional product photograph showing this @{variables('brand')} @{body('Parse_JSON_-_Extraction_Output')?['product_identity']?['product_name']?['value']} installed and in use in a residential outdoor landscape setting. Natural ambient lighting, photorealistic, sharp focus on the product with softly blurred background. Preserve the product's exact appearance, shape, color, labeling, and materials from the reference image. No text overlays, no added logos, no watermarks, no humans in frame."
        },
        {
          "inline_data": {
            "mime_type": "image/jpeg",
            "data": "@{base64(body('HTTP_-_Download_Reference_Image'))}"
          }
        }
      ]
    }
  ],
  "generationConfig": {
    "responseModalities": ["TEXT", "IMAGE"]
  }
}
```

**Settings tab:**

| Setting | Value |
|---|---|
| **Retry Policy** | `None` (Gemini failures must surface fast, not retry and burn credits) |
| **Action Timeout** | `PT120S` (Nano Banana 2 took 15-30s in tests; generous headroom) |
| **Secure Inputs** | ON (hides API key in run history) |
| **Secure Outputs** | OFF for now (need to inspect response body during Y14 debugging) |

**Validated in test:** HTTP 200 success with 1.8MB response body and 15-16s duration on Brilliance test.

**Important notes:**
- `mime_type: "image/jpeg"` is hardcoded. Most product images are JPEG; Gemini usually sniffs actual content. If a product page serves WebP or PNG, may need dynamic detection — not yet a problem.
- `generationConfig.responseModalities: ["TEXT", "IMAGE"]` is REQUIRED for Gemini to return an image. Without it, you get only text.
- **Y13 gotcha:** The initial build attempted `body('HTTP_-_Download_Reference_Image')$content` — PA validator rejects `$content`. Fix: `base64(body(...))` as shown.

### Z-6. `Parse JSON - Gemini Lifestyle 1`

**Action type:** Data Operations → Parse JSON

**Position:** Immediately after Z-5.

**Configuration:**

| Field | Value |
|---|---|
| **Content** | Dynamic content → Body from `HTTP - Gemini Lifestyle 1` |
| **Schema** | See block below (hardened with `additionalProperties: true`) |

**Schema:**

```json
{
  "type": "object",
  "additionalProperties": true,
  "properties": {
    "candidates": {
      "type": "array",
      "minItems": 0,
      "items": {
        "type": "object",
        "additionalProperties": true,
        "properties": {
          "content": {
            "type": "object",
            "additionalProperties": true,
            "properties": {
              "parts": {
                "type": "array",
                "minItems": 0,
                "items": {
                  "type": "object",
                  "additionalProperties": true,
                  "properties": {
                    "text": { "type": ["string", "null"] },
                    "inlineData": {
                      "type": "object",
                      "additionalProperties": true,
                      "properties": {
                        "mimeType": { "type": "string" },
                        "data": { "type": "string" }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**Validated:** Parse succeeds. Schema is loose enough to tolerate response variations.

### Z-6b. `Compose - Gemini Debug` *(diagnostic; added Apr 19 during Y14 troubleshoot — remove before production)*

**Action type:** Data Operations → Compose

**Position:** Between Z-6 and Z-7 in the TRUE branch of `Check - Has Reference Image`.

**Purpose:** The Gemini HTTP response is ~1.8MB of base64 and impossible to inspect directly — PA run history struggles to render it, and it exceeds LLM context windows for outside diagnosis. This Compose distills the response to 5 decision-relevant signals so a failure can be diagnosed in one glance.

**Inputs (paste as-is; will render as mixed plain text + `@{}` chips after save):**

```
{
  "candidate_count": @{length(coalesce(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates'], createArray()))},
  "finish_reason": "@{coalesce(string(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['finishReason']), 'NULL')}",
  "parts_length": @{length(coalesce(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['content']?['parts'], createArray()))},
  "prompt_feedback": "@{coalesce(string(body('Parse_JSON_-_Gemini_Lifestyle_1')?['promptFeedback']), 'NULL')}",
  "candidate_safety_ratings": "@{coalesce(string(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['safetyRatings']), 'NULL')}"
}
```

**Signal interpretation:**

| Field | Meaning |
|---|---|
| `candidate_count` | 0 → response had no candidates (total failure). 1 → normal. |
| `finish_reason` | `STOP` → normal completion. `SAFETY` / `IMAGE_SAFETY` → content filter block. `MAX_TOKENS` → truncated. |
| `parts_length` | 1 → image at `[0]`. 2 → text + image at `[1]`. 0 → nothing generated. |
| `prompt_feedback` | Non-empty string → check for `blockReason` (prompt-level safety block). |
| `candidate_safety_ratings` | Non-empty → per-category safety scores; inspect if filtering suspected. |

**Historical use:** Brilliance run Apr 19 returned `{candidate_count: 1, finish_reason: "STOP", parts_length: 1, prompt_feedback: "", candidate_safety_ratings: ""}` — confirmed Gemini was succeeding and image was at `parts[0]`, driving the Y14 coalesce fix.

**Removal before production:** Delete this Compose when Step 32 is validated on enough runs that no new response shape variants are expected. At that point Z-7 can be simplified OR the debug can be left in for diagnostic continuity (trivial cost — Compose is free).

### Z-7. `OneDrive - Save Lifestyle Image 1` *(WORKING — Y14 fixed Apr 19)*

**Action type:** OneDrive for Business → Create file

**Position:** Immediately after Z-6b (`Compose - Gemini Debug`).

**Configuration:**

| Field | Value |
|---|---|
| **Folder Path (Expression)** | `concat(variables('item_folder_path'), '/lifestyle')` |
| **File Name (Expression)** | `concat(variables('part_number'), '_lifestyle_1.png')` |
| **File Content (Expression)** | `base64ToBinary(coalesce(body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['content']?['parts']?[0]?['inlineData']?['data'], body('Parse_JSON_-_Gemini_Lifestyle_1')?['candidates']?[0]?['content']?['parts']?[1]?['inlineData']?['data']))` |

**Output path at runtime:** `/Item Setup POC/<Brand>_<Part>/<correlation_id>/lifestyle/<Part>_lifestyle_1.png`

**Validated Apr 19 on Brilliance run:** PNG saved successfully. Image quality confirmed — Gemini preserved product appearance in residential outdoor landscape scene per prompt spec.

**Expression rationale (coalesce pattern):** Gemini's response can have either 1 or 2 elements in `parts`. When `responseModalities: ["TEXT", "IMAGE"]` is set, the model sometimes returns both a text narration (at `[0]`) AND the image (at `[1]`), and other times just the image (at `[0]`). Coalesce tries index 0 first; if that `inlineData.data` is null (meaning `[0]` is a text part), falls through to `[1]`. Works for both shapes.

See Y14 for full diagnostic history.

### Remaining Step 32 sub-steps (not yet built)

- **32-3: Cost tracker wiring** — new top-level variable `google_api_calls_total` (Integer, init 0). New `Increment - Google API Calls` action after Z-5 HTTP. Add `google_api_calls` and `estimated_google_usd` (`mul(variables('google_api_calls_total'), 0.045)`) to `Compose - Cost Metadata`. Update `estimated_total_usd` to include Google. Update HTML preview Cost Breakdown section to show Google API as 4th line.
- **32-4: HTML preview new section** — add "Generated Lifestyle Images" section to `compose_template_v4.html`, positioned after the existing "Image Gallery" section. Reads from new `lifestyle_images` field in extraction metadata payload. Needs new `Compose - Lifestyle Metadata` action that collects generated filenames, prompt text, generation timestamp, Gemini model ID. Kept SEPARATE from the `images` flat_row output per Tony's "not in CSV yet" decision.
- **32-5: End-to-end validation** — rerun Brilliance and Intermatic after Y14 unblock. Confirm `/lifestyle/` folder appears with PNG, HTML preview renders the new section, cost tracker shows Google line, email body KPI reflects lifestyle generation success.

### Internal action names (for expressions)

Add these to Section U mapping when Step 32 is fully complete:

| Display name | Internal name |
|---|---|
| `Filter array - Product Images Only` | `Filter_array_-_Product_Images_Only` |
| `Compose - Reference Image URL` | `Compose_-_Reference_Image_URL` |
| `Check - Has Reference Image` | `Check_-_Has_Reference_Image` |
| `HTTP - Download Reference Image` | `HTTP_-_Download_Reference_Image` |
| `HTTP - Gemini Lifestyle 1` | `HTTP_-_Gemini_Lifestyle_1` |
| `Parse JSON - Gemini Lifestyle 1` | `Parse_JSON_-_Gemini_Lifestyle_1` |
| `Compose - Gemini Debug` *(diagnostic, remove before prod)* | `Compose_-_Gemini_Debug` |
| `OneDrive - Save Lifestyle Image 1` | `OneDrive_-_Save_Lifestyle_Image_1` (or `Create_file` if Tony didn't rename) |

---

