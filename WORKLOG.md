# Work Log

Kelly's activity log for the AWESOMEREE Web App. Entries are organized by work session, starting from 2026-02-16.

**Session ID Convention**: Use `MMDD-N` format (e.g., `0219-1`) where MMDD is the date and N is the session number for that day.

---

### Session 0306-3 (2026-03-06)

**Feat: Shopee SG Analytics Table Page — Standalone AllBots Repository**

- **Context**: Kelly wanted a Shopee SG analytics table page that fetches from `AllBots.Shopee_My_Products` + `AllBots.Shopee_Comp_Data` (region = 'SG'). Must not affect Shopee MY or VVIP pages.
- **New files created (7)**:
  - `lib/services/shopee-sg-products-repository.ts` — standalone SG data access layer (AllBots DB), copied from VVIP repo with table refs changed from `webapp_test.*` to AllBots, function names renamed from `Vvip` to `Sg`, similarity columns changed from NULL to real `mp.*` columns
  - `app/api/shopee-sg/products/route.ts` — main API (GET/POST/PUT/PATCH/DELETE)
  - `app/api/shopee-sg/products/counts/route.ts` — tab counts
  - `app/api/shopee-sg/products/details/route.ts` — product details (images/descriptions)
  - `app/analytics/table/shopee-sg/page.tsx` — full analytics table page (copied from test, cleared MY shop options)
  - `app/analytics/table/shopee-sg/lib/api.ts` — client-side API helpers
  - `app/analytics/table/shopee-sg/layout.tsx` — layout wrapper
- **Modified files**: navigation (added "Shopee SG" tab), `lib/type.ts` (added "Shopee SG" to TabType)
- **Fixes from CI review**:
  - Added `"Shopee SG"` to `TabType` union type in `lib/type.ts`
  - Disabled similarity exclusion on SG page (prevents writing to MY tables)
- **DB verification**: Confirmed `AllBots.Shopee_My_Products` (0 SG rows, bot hasn't run) and `AllBots.Shopee_Comp_Data` (474 SG rows) — identical column structures to `webapp_test.*`
- **Data flow verified**: All 80+ columns from SG repo SELECT correctly map to enrichment layer fields
- **Branches & PRs**:
  - `feature/shopee-sg-page-main` → PR to `main` (cherry-picked from original branch + fixes)
  - `feature/shopee-sg-page-v2` → PR to `test` (branched from test, updated SG routes to use new SG repo)
  - `feature/shopee-sg-page` — original branch (superseded by above two)
- **Key decisions**: Shop filter left empty (no SG shop data yet), category filter kept same as MY (VVIP/VIP/Links Input/New Item), status/bulk-status/permanent-delete routes skipped for now
- **Tools used**: Git, MySQL queries, code editing, GitHub MCP (failed on private repo — PRs created manually)
- **Status**: Both branches pushed, PRs pending manual creation/review

---

### Session 0306-2 (2026-03-06)

**Fix: CI Test Failures + PRs for SKU Dedup / 0-Sales / Latest Date Scraped / Category Variations**

- **Context**: Previous session created 4 commits across branches (SKU dedup, 0-sales comps, latest date scraped, category variations). CI failed on PR #493 — 2 tests in `shopee-history-calculations.test.ts` expected old behavior (filtering out 0-sales comps) but code intentionally removed that filter.
- **Fix — CI tests** (`lib/shared/shopee-history-calculations.test.ts`):
  - "filters out rows with monthlySales = 0" → renamed to "includes rows with monthlySales = 0", expects 2 results instead of 1
  - "filters out rows with monthlySales = null or undefined" → renamed to "includes rows with monthlySales = null or undefined", expects 3 results instead of 1
- **Branch management**:
  - `fix/dup-var-and-date-scraped-main` (from `main`) — added test fix commit `3ccfd47`, pushed to origin
  - `fix/dup-var-and-date-scraped` (from `test`) — deleted old branch, recreated from `test`, cherry-picked all 5 commits, force-pushed to origin. PR #493 updated.
- **Commits (5 per branch)**:
  1. `1ec6c30` / `6df9433` — SKU dedup (use SKU as variation key)
  2. `fd236fe` / `50c4cc9` — show 0-sales competitors in UI
  3. `afd24e3` / `85c4943` — show only latest scrape date competitors (LEFT JOIN subquery)
  4. `134050d` / `36da034` — keep all variations with same category
  5. `3ccfd47` / `01f2f02` — fix CI tests to match new behavior
- **PRs**:
  - `fix/dup-var-and-date-scraped` → PR to `test` (PR #493, updated via force-push)
  - `fix/dup-var-and-date-scraped-main` → PR to `main` (created manually by Kelly)
- **Tools used**: Git cherry-pick, force-push, code editing
- **Status**: Both PRs created, pending CI checks and Agnes's approval

---

### Session 0305-1 (2026-03-05)

**Fix: Date Filter Hiding Variations + Comp Variation Backfill**

- **Branch**: `DATE-KELLY` (3 commits: `89ee6dc`, `12dd905`, `8dadbd1`)
- **Context**: Two related issues:
  1. Date filter hid product variations scraped on different dates (e.g., Luxbin 41 variations → only 12 when filtering March 4, because NONE records from listing-to-db bot had different `date_taken`)
  2. Competitor product variations dropped by top-3 per-variation filter (e.g., MOF Exclusive showed 3/4 variations, Dustbin showed 12/13)
- **Fix 1 — Date filter** (`lib/services/shopee-products-repository.ts`):
  - Names query (pagination) keeps date filter → determines WHICH products to show
  - Rows query skips date filter via `whereSqlRowsExpanded` → returns ALL active variations
  - All other filters (shop, sheet_name, status, search) preserved in both queries
  - First version used subquery (`our_link IN (SELECT ...)`) — too slow on unindexed TEXT column. Second version splits WHERE clauses — fast.
- **Fix 2 — Comp backfill** (`lib/shared/shopee-history-calculations.ts`):
  - Added step 1.5 to `getQualifiedCompRows()`: if a competitor product survives top-3 on ANY our_variation, ALL its comp_variations are backfilled from original data
  - Fixes cases like MOF Pink (4th ranked on "Gingham-Pink" but survives on Green/Blue/Black), Dustbin "20L-Silver", Miss Jan "S,red"
- **VM3 bot fix** (`Shopee-mylinks-sales-data-merged.py`):
  - Added `c.sheet_name = '{NONE_SHEET_NAME}'` to `sheet_date_condition` so NONE rows get SiteGiant `our_sales_7d/14d/30d` updates daily
  - Previously NONE rows had stale sales data because the UPDATE's WHERE excluded them
- **Investigation**: Verified variation counts for multiple products (Knitted Vest item 25900367485, MOF Exclusive MFLC012, Dustbin, LANFY truck) comparing DB vs UI
- **Tools used**: MySQL queries, VM3 bot script edit, code editing, TypeScript
- **Branches & PRs**:
  - `fix/date-filter-all-variations-v2` (from `test`) → PR to `test` (staging validation)
  - `fix/date-filter-all-variations` (from `main`) → PR to `main` (production deploy)
  - Both have same 3 commits, cherry-picked from `DATE-KELLY`
  - AI checks running, waiting for Agnes's approval
- **Status**: Both PRs created, pending review

---

### Session 0304-2 (2026-03-04)

**Add Shopee SG Comp Analysis Page**

- **Context**: New Shopee SG page needed at `/analytics/table/shopee-sg`, identical functionality to VVIP but filtering by `region='SG'`. The `region` column already exists in both `Shopee_My_Products` and `Shopee_Comp_Data` (default 'MY'). No SG data exists yet.
- **Approach**: Reuse VVIP repository (`shopee-vvip-products-repository.ts`) with optional `region` parameter — zero code duplication. Tech lead recommendation: adding future regions (TH, PH) = just a new API route + page, no repo changes.
- **Files created (6)**:
  - `app/analytics/table/shopee-sg/layout.tsx` — pass-through layout
  - `app/analytics/table/shopee-sg/lib/api.ts` — frontend API client → `/api/shopee-sg/products`
  - `app/analytics/table/shopee-sg/page.tsx` — full page copied from VVIP, all refs → "Shopee SG"
  - `app/api/shopee-sg/products/route.ts` — main API with `REGION = "SG"`
  - `app/api/shopee-sg/products/counts/route.ts` — tab counts with `REGION = "SG"`
  - `app/api/shopee-sg/products/details/route.ts` — details with `REGION = "SG"`
- **Files modified (7)**:
  - `lib/services/shopee-vvip-products-repository.ts` — added optional `region?` param to `fetchVvipProducts`, `fetchVvipProductCounts`, `fetchVvipProductDetails`. Added `vvipJoinForRegion()` helper. Region added to cache key, `buildWhere()`, `buildMpOnlyWhere()`.
  - `components/analysis-sub-navigation.tsx` — added "Shopee SG" tab
  - `components/analysis-sub-navigation-mobile.tsx` — added "Shopee SG" tab
  - `lib/type.ts` — added `"Shopee SG"` to `TabType` union
  - `app/api/shopee-my-vvip/products/route.ts` — added `REGION = "MY"` for isolation
  - `app/api/shopee-my-vvip/products/counts/route.ts` — added `REGION = "MY"`
  - `app/api/shopee-my-vvip/products/details/route.ts` — added `REGION = "MY"`
- **Data isolation**: VVIP passes `region="MY"` (WHERE + JOIN), SG passes `region="SG"`. Shopee MY uses different repo — untouched. Zero cross-contamination.
- **Similarity**: All 10 similarity columns remain `NULL AS` in shared repo (matches `test` branch state). When similarity is ready, change in shared repo applies to both VVIP and SG automatically.
- **SG page string replacements from VVIP**: API endpoints, TabType `"Shopee MY"` → `"Shopee SG"`, display text, log labels, model_name, localStorage keys (`shopeeMY:*` → `shopeeSG:*`)
- **Branch**: `feature/shopee-sg-comp-analysis` (created from `test`, cherry-picked from `kelly-sg-page-comp-analysis`)
- **PR**: Not yet created — branch pushed to remote. Need to create via GitHub web UI → base: `test`
- **PR workflow note**: Branch was created from `test` (not `main`) because VVIP files only exist on `test`. Agnes's new PR flow requires branching from `main` — need to clarify with Agnes for this case.
- **Build verification**: Zero new TS errors. Pre-existing build failures unrelated (missing `@tanstack/react-virtual`, `cron-parser`).
- **Tools used**: Code editing, git cherry-pick, GitHub push, TypeScript compiler, sed for bulk string replacement
- **Status**: Code done, branch pushed, PR pending creation

---

### Session 0304-1 (2026-03-04)

**Fix: Uncategorized product among VVIPs + Cap COMP to top 3**

- **Context**: When sorting by Category with a date filter (March 2, 2026), an Uncategorized product ("Projector Stand Tripod") appeared on page 1 among VVIPs. Two different products shared the same `product_name` but had different `our_link` URLs and categories.
- **Root cause**: Server paginated by `GROUP BY product_name`, but client groups by `product_id` (from `our_link`). Two products with same name merged into one server group → sorted as VVIP → both placed on page 1. Client split them → one showed Uncategorized among VVIPs.
- **Fix — `lib/services/shopee-products-repository.ts`**:
  - Non-summary path: `GROUP BY our_link` instead of `GROUP BY product_name`
  - All non-summary `orderExpr` tiebreakers: `our_link` instead of `product_name`
  - Aggregate sort CTE (var_sales, totals, pct, metric): all use `our_link`
  - Count query: `COUNT(DISTINCT our_link)` instead of `COUNT(DISTINCT product_name)`
  - Extraction uses `groupCol` variable for summary vs non-summary paths
  - Added `sc.` prefix to WHERE clause to avoid "ambiguous column" error from exclusion JOINs
  - Fixed missed `productNames` → `groupValues` rename at line 1463
- **Fix — `components/Shopee-MY-History/grouped-rows.tsx`**:
  - Added `.slice(0, 3)` to cap visible COMP products to top 3 by monthly sales
- **Previous wrong fix**: Pagination badge fix (window function approach) — misdiagnosed as group split across pages, but real issue was two products merging into one group
- **HTTP 500 errors encountered and fixed**:
  1. "Column 'our_link' in where clause is ambiguous" — added `sc.` prefix
  2. "productNames is not defined" — missed variable rename
- **Branch**: `fix/group-by-ourlink-top3-comp-v2` (created from `main` per PR workflow rules)
- **PRs created**: PR #1 to `test`, PR #2 to `main` (same branch → both targets)
- **DB investigation**: Queried "Luxbin Dustbin" product — 17 variations VVIP on March 2, 35 variations NONE on March 3. Mixed sheet_names across scrape dates.
- **Tools used**: MySQL MCP queries, Chrome DevTools (network 500 debugging), code editing, GitHub push
- **Status**: PRs created, waiting for CI checks and Agnes's approval

---

### Session 0303-2 (2026-03-03)

**Date Filter Fix — Shopee MY Analytics Table & Comp Analysis**

- **Context**: Product `20525200467` (Swimming Pool Intex) shows 14 unique SKUs without date filter, but only 10 when filtering by March 2. Also investigated comp analysis `date_taken` matching issue.
- **Root cause**: The scraper inserts new rows with different `date_taken` timestamps each run. Some variations (BW-RSP, INTX530GPH, sheet_name="NONE") were scraped on March 3 while VVIP variations were scraped on March 2. The row-level date filter (`date_taken >= ? AND date_taken <= ?`) excluded cross-date variations.
- **DB investigation**: Queried `AllBots.Shopee_Comp` — confirmed 50 total rows across 4 dates for this product, 14 unique SKUs. With date filter March 2: only 16 rows / 10 unique SKUs returned (old code). With product-level subquery: 50 rows / 14 unique SKUs (fixed).
- **Fix 1 — Analytics table** (Shopee MY):
  - File: `lib/services/shopee-products-repository-mysql.ts` (lines 268-284)
  - Changed row-level date filter to product-level subquery: `our_link IN (SELECT DISTINCT our_link FROM Shopee_Comp WHERE date_taken >= ? AND date_taken <= ?)`
  - Frontend `deduplicateCompetitorRows()` (page.tsx line 403) already handles keeping latest data per variation
- **Fix 2 — Comp analysis analyze API**:
  - File: `app/api/comp-analysis/analyze/route.ts` (lines 216-284)
  - Removed `date_taken` exact match from WHERE clause
  - Added `ROW_NUMBER() OVER (PARTITION BY comp_variation ORDER BY date_taken DESC, id DESC)` for dedup
  - VVIP not affected (completely separate code)
- **Additional finding**: "Shopee MY 50" count badge shows raw DB count instead of deduped count — NOT yet fixed
- **DB note**: MCP tool shows UTC dates (subtract 8h from stored MYT values). `2026-03-01T16:00:00Z` = `2026-03-02 00:00:00 MYT`
- **Status**: Code done locally, NOT yet committed/pushed/PR'd
- **Tools used**: MySQL MCP queries, code exploration, plan mode

---

### Session 0303-1 (2026-03-03)

**VM TT — WhatsApp Screenshot Bot Selector Fix**

- **Context**: 3 Python scripts on VM TT (`Screenshots my sg tt/`) that screenshot account health reports and send them to WhatsApp group "Awesomeree Discussion" were all failing. Scripts: `TT-AcountHealthScreenshot.py` (TikTok), `Shopee.py` (Shopee MY), `SGShopee.py` (Shopee SG)
- **Issue diagnosed**: WhatsApp Web updated its DOM — the search box changed from a `<p>` element inside nested `<div>`s to an `<input>` element with `data-tab="3"` attribute
- **Old selector (broken)**: `div.x1hx0egp.x6ikm8r.x1odjw0f.x6prxxf.x1k6rcq7.x1whj5v > p`
- **New selector (fixed)**: `input[data-tab='3']`
- **Files changed on VM TT**:
  - `Screenshots my sg tt/TT-AcountHealthScreenshot.py` (line 89)
  - `Screenshots my sg tt/Shopee.py` (line 137)
  - `Screenshots my sg tt/SGShopee.py` (line 114)
- **PR to bot-scripts**: Prepared PR title + description, but could not push — GitHub MCP token lacks access to `it-awesomeree/bot-scripts`, gh CLI not installed, Chrome DevTools MCP connects to debug Chrome (not logged into GitHub). PR creation pending manual action.
- **Other finding**: `TT-AcountHealthScreenshot.py` has no `logging.basicConfig()` — all `logging.info()` calls are silently ignored. `Shopee.py` and `SGShopee.py` have logging configured properly.
- **Tools used**: VM Control Plane (vm_status, vm_read_file, vm_execute, vm_processes), Chrome DevTools MCP

---

### Session 0302-1 (2026-03-02)

**VVIP Comp Analysis — Bug Fixes, UI Simplification & Variation Count**

- **Context**: Shopee MY VVIP page (`/analytics/table/shopee-my-vvip`) had 3 bugs in competitor variation display + missing features flagged by Agnes. Branch: `fix/vvip-comp-analysis-improvements` → PR to `test`
- **Bug 1 — Unmatched our_variations dropped (FIXED)**:
  - Our variations with no matching competitor were silently omitted from the UI
  - Fix: API now emits blank competitor columns for all unmatched our_variations
  - File: `app/api/shopee-my-vvip/products/route.ts` (Step 4 in `limitAndDedupCompetitors()`)
- **Bug 2 — Wrong variation assignment due to generic words (FIXED)**:
  - "LED Light" (ours) matched to "Brown Set" (comp) because "set" overlapped
  - Fix: Added `variationsMatchScore()` with `MATCH_STOP_WORDS` (set, pcs, free) and `LOW_SIGNAL_WORDS` (led, mini, pro)
  - File: `app/api/shopee-my-vvip/products/route.ts`
- **Bug 4 — Partial comp variations shown (FIXED)**:
  - BBM Chair showed only 4 Brown variations but DB had 8 (4 Brown + 4 Wood)
  - Root cause: Per-variation top 3 filter dropped BBM Wood (52 sales) because NIVISON (988 sales) outranked it on the same our_var
  - Fix: Added optional `backfillProducts` param to `getQualifiedCompRows()` — if a comp product survives top 3 on any variation, ALL its variations are included
  - Isolation: param defaults to `false`, only VVIP passes `true`. **Shopee MY unaffected**.
  - Files: `lib/shared/shopee-history-calculations.ts`, `components/Shopee-MY-History/grouped-rows.tsx`, `app/analytics/table/shopee-my-vvip/page.tsx`
- **UI Simplification — Removed non-functional tabs**:
  - Agnes flagged 3 missing API routes (status, bulk-status, permanent-delete)
  - Decision: omit these features for now instead of implementing
  - Removed Pending/Fixed/Deleted tabs, BulkActionsSheet, DeleteConfirmation, PermanentDeleteConfirmation
  - Only "Shopee MY" tab remains with styled orange badge
- **Badge count — Changed to variation count**:
  - Before: badge showed product count (134)
  - After: badge shows total our_variations across all product groups (sum of variations)
  - Uses `groupProducts()` + `useMemo` to compute client-side
- **Shopee MY impact: ZERO** — verified `git diff main -- app/analytics/table/shopee-my/` returns empty
- **Branch**: `fix/vvip-comp-analysis-improvements` (cherry-picked from `fix/vvip-competitor-display`)
- **PR**: Created to `test` branch
- **Files changed**: `route.ts` (VVIP API), `page.tsx` (VVIP page), `shopee-history-calculations.ts` (shared, safe), `grouped-rows.tsx` (shared, safe)
- **Tools used**: MySQL direct query, TypeScript compiler, git cherry-pick, GitHub MCP

---

### Session 0227-1 (2026-02-27)

**Comp Analysis — Parent Category & Grouping Fixes (In Progress)**

- **Context**: Shopee MY analytics table (`/analytics/table/shopee-my`) had multiple display bugs. Branch: `kelly-fixing-comp-analysis`
- **Bug 1 — Parent category showing wrong value (FIXED)**:
  - Product 25938069995 (das.nature2) showed VIP at parent level, but DB only has VVIP + NONE
  - Root cause: code used `groupMetrics?.category ?? "VIP"` — pre-computed value from server enrichment that fell back to "VIP" when name-based lookup failed
  - Fix: changed to `getHighestPriorityCategory(groupRows)` — computes live from actual variation data
  - File: `components/Shopee-MY-History/grouped-rows.tsx` line 1156
- **Bug 2 — Products with same product_id split into separate groups (FIXED)**:
  - Product 22026149091 appeared as two parent groups with different names (seller renamed product)
  - Root cause: `grouped-rows.tsx` had its own LOCAL `groupProducts()` that grouped by **product name**, separate from the shared version in `shopee-history-calculations.ts` that groups by **product_id**
  - Fix: removed local `groupProducts` and imported the shared version that groups by product_id
  - Also fixed duplicate React key error by using `group.productId ?? group.name` as key
- **Bug 3 — Category sort mixing across pages (INVESTIGATING)**:
  - Server paginates by `product_name` (GROUP BY product_name), client groups by `product_id`
  - This mismatch causes categories to mix across pages (e.g., VVIP on page 1, then VVIP again on page 2)
  - Products sharing same name but different product_ids get same sort values from enrichment
  - Added live category computation for sort to fix within-page mixing
  - Cross-page consistency still needs investigation — server vs client grouping mismatch is the root cause
  - Discussion with Kelly: needs server-side change to GROUP BY product_id for true global sort
- **DB queries run**: Verified `sheet_name` values for products 25938069995 (VVIP+NONE), 24924470455 (LINKS_INPUT+VIP+NONE), 26719140723 (LINKS_INPUT+NONE)
- **Files changed**: `components/Shopee-MY-History/grouped-rows.tsx`, `lib/shared/shopee-history-calculations.ts` (debug log added then removed)
- **Tools used**: MySQL direct query, Chrome browser automation, TypeScript compiler
- **Status**: Bugs 1 & 2 fixed. Bug 3 (cross-page sort) needs further discussion — fundamental server/client grouping mismatch

---

### Session 0226-1 (2026-02-26)

**GRBT-155 — Stock Adjustment Flow (Review & Close)**

- **Context**: Stock adjustment bot (`stockID.py` on VM5) was 100% broken since Feb 11. Ticket was in TO REVIEW — Jana (Janarthan Nair) submitted fixes. Agnes/Claude Code reviewed.
- **Workflow**: `/workflow-webapp` — Path B (dev-submitted work, review only)
- **Work done**:
  - Read full Jira ticket (description, 5 comments, 3 attachments)
  - Dev compliance check: verified all 7 of Agnes's action items from Feb 17
  - Code review of PR #22 (bot-scripts repo) — 903-line `stockID.py` Selenium scraper
  - Security audit: PASS (no hardcoded creds, parameterized SQL, .env usage)
  - Critical code review ("angry senior dev" style) — identified 12 issues including no session validation, no idempotency, fragile CSS selectors, self-merged PR
  - Posted technical review comment on Jira (Step 8B) — compliance matrix, security review, root cause classification
  - Created GitHub changelog `docs/changelog/GRBT-155.md` in bot-scripts repo (Step 11) — committed via PR (main is protected)
  - Posted non-technical CS team comment on Jira (Step 12B) — plain language for Myrsya
  - Transitioned ticket to DONE, ranked to top of Done column
- **Root cause**: Configuration/Environment Issue + System/Code Defect — expired SiteGiant session, rotated DB password, retry bug in WHERE clause
- **Verdict**: REVIEW PASSED — all 7 action items complete, bot operational at 67% (remainder explained)
- **Out of scope**: Stock reservation automation (needs separate ticket)
- **Tools used**: Claude-in-Chrome (Jira + GitHub), Jira REST API (comments, transitions, ranking), GitHub web UI

**GRBT-194 — Stock Adjustment (Started, incomplete)**

- **Context**: Follow-up to GRBT-155 — same bot, new issue. SiteGiant cookie consent banner overlaying confirm button.
- **Status**: Step 1 (Study Jira Ticket) started — read ticket details, Jana's comment, PR #27 linked. Session ended before completing review.
- **Will resume next session**

---
## AW-357 — Add duplicate submission prevention on Replacement & Review form

**Date:** 2026-02-25 to 2026-02-27
**Status:** Deployed to Production
**Assignee:** Kelly Ee
**Reporter:** Agnes Foong
**Labels:** frontend, backend

### Problem
The Replacement & Review form (`/customer-service/replacement-review`) had no protection against duplicate submissions. When the system was slow, users could double-click the submit button and create identical records (see GRBT-202). Audit log showed two identical records created 1 second apart — classic double-submit.

### What was done

**1. Frontend — Disable submit button on click**
- Added `submitting` state on the Add New form — disables the Create button and shows "Submitting…" while request is in-flight
- Re-enables on error so the user can retry
- Handles 409 Conflict response and shows user-friendly duplicate warning

**2. Backend — Duplicate order check**
- Added `checkRecentDuplicate()` function — checks if a record with the same `order_id` was created within the last 60 seconds
- Returns 409 Conflict if duplicate detected, blocking repeated submissions
- Implemented atomic duplicate check to prevent TOCTOU race condition (handles concurrent requests)

**3. Logging gap — Firebase auth headers**
- Passed Firebase auth headers (`x-firebase-uid`, `x-user-email`, `x-user-name`) from the replacement review form
- Audit logs now capture who created each record in `history_logs` for both create and update actions

### Files changed
- `lib/replacement-review.ts` — added `checkRecentDuplicate()`
- `app/api/replacement-review1/route.ts` — added duplicate check before insert
- `components/replacement-review1/controls.tsx` — added `submitting` state to prevent double-click

### Commits
- `3576461` — fix-add-duplicate-submission-prevention-on-Replacement-&-Review-form-(AW-357)
- `edd9480` — fix: atomic duplicate check to prevent TOCTOU race condition (AW-357)

### PRs & Deployment
- **PR #369**: `kelly/AW-357-duplicate-prevention-and-date-fix` → merged to `test` (Feb 26, 4:34 PM)
- **PR #364**: `test` → `main` (merged Feb 27, 12:32 AM) — bundled with 25 commits from multiple devs
- **Deployment issue**: GitHub Actions billing was locked, blocking all deployments after Feb 26 ~4 PM
- **Resolution**: Billing fixed by Agnes → manually triggered Agent 5 (Build & Deploy Staging) from GitHub Actions
- **Agent 5** (Run #154): Build (3m 24s) → Deploy to Staging (16m 42s) → Smoke Tests (59s) — all passed
- **Agent 6** (Run #162): Production Canary Rollout (16m 34s) — 5% → 25% → 50% → 100% traffic — all passed
- **Live in production**: Feb 27, 2026 ~10:35 AM at https://employee.awesomeree.com.my


### Session 0220-1 (2026-02-20)

**AW-298 — Comp. Analysis MY Bug Fix (Done)**

- **Context**: COMP Product count was not aligned with per-variation competitor display in Shopee MY History. Competitors were being deduped incorrectly when listings had multiple variations.
- **Root cause**: Dedup logic used `|||` magic string separator to group competitor+variation pairs — fragile and error-prone. Count aggregation didn't match the per-variation display logic.
- **Fix implemented**:
  - Added shared `getCompMonthlySales()` function in `lib/shared/shopee-history-calculations.ts` — single source of truth for comp monthly sales calculation
  - Replaced `|||` string separator with proper two-level `Map` data structure
  - Added `MAX_QUALIFIED_COMP_ROWS = 10` safety cap (prevents unbounded rendering if data is unexpectedly large)
  - Moved `formatPriceValue()` to shared location (was duplicated across files)
  - Removed unnecessary `as any` casts (TypeScript strictness)
- **Files changed**: `components/Shopee-MY-History/grouped-rows.tsx`, `components/ca-test/Shopee-MY-History/grouped-rows.tsx`, `lib/shared/shopee-history-calculations.ts`, `lib/shared/shopee-history-calculations.test.ts`
- **Commit**: `bf13282` — fix: align COMP Product count with per-variation competitor display

### Session 0219-2 (2026-02-19)

**AW-324 — Costing: Frontend Data Fetching to Database**

- **Context**: The costing module needed to pull comp analysis data from the frontend into the database for the selling price calculation pipeline.
- **Work done**:
  - Built frontend-to-database data fetching in `lib/services/shopee-products-enrichment.ts`
  - Tested syncing of comp analysis data to the costing module via `app/api/costing-excel/selling-price/route.ts`
- **Files changed**: `lib/services/shopee-products-enrichment.ts`, `app/api/costing-excel/selling-price/route.ts`
- **Commits**:
  - `49c6f94` — AW-324 frontend data fetching to database
  - `9979144` — testing the syncing of the comp analysis → costing
- **Status**: To Do (data fetching pipeline set up, further integration pending)

### Session 0219-1 (2026-02-19)

**AW-317 — Chatbot Conversation Log Viewer: Round 1 Improvements (In Progress)**

- **Context**: Building a premium UI/UX chatbot conversation log viewer. This session focused on the first round of major UI improvements to the viewer page.
- **Features implemented**:
  - Wider chat detail panel — responsive scaling on `md`/`lg` screens for better readability
  - Stats time breakdown — Today/Week/Month conversation counts displayed in metrics cards
  - Export CSV button — added to filter bar, client-side CSV generation
  - Search by message content — `EXISTS` subquery on `chatbot_messages` table for full-text search
  - Full pagination bar — rows per page selector, go-to-page input, page number navigation
  - Error handling + retry — error banner with Retry button on API failures
  - URL filter persistence — all filter states synced to URL search params (shareable/bookmarkable)
  - Date separators in chat — visual dividers between messages on different days
  - Full CSV export — fetches all filtered results in batches (not just current page)
- **Files changed**: `app/api/chatbot/conversations/route.ts`, `app/chatbots/overview/page.tsx`, `components/chatbot/conversation-viewer.tsx`
- **Commits**:
  - `c197bc6` — AW-317: Add chatbot viewer improvements — wider panel, time stats, CSV export, message search, full pagination
  - `2492fa2` — make improvement on the chatbots (half way done)
  - `a031942` — add on the pagination inside the bottom
- **Remaining TODO**: Connection pooling (`dbPool.executeWithRetry()`), auto-scroll to latest message, column sorting, product context display, mark as resolved action, real-time updates / auto-refresh
