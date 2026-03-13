# Error Log

Errors encountered during development and how they were resolved. Prevents repeat debugging.

**Format**: Each entry follows the standard structure below. Add newest entries at the top.

**Session ID Convention**: Use `MMDD-N` format (e.g., `0219-1`) matching WORKLOG sessions.

---
### Session 0312-1 (2026-03-12)

- **Error**: `apply_patch` repeatedly failed with exit code 1 and no output while patching SG/VVIP files
- **Discovered by**: Claude Code during implementation
- **Root cause**: Tool-level patch application issue in this environment for these file edits
- **Resolution**: Switched to targeted PowerShell in-place edits and validated with `git diff` + build/lint
- **Prevention**: If `apply_patch` fails twice with no diagnostics, switch early to deterministic scripted replacements and re-verify exact line-level diff
- **Status**: RESOLVED

---

- **Error**: GitHub MCP `create_pull_request` failed with `Authentication Failed: Bad credentials`
- **Discovered by**: Claude Code when creating PR for `codex/vvip-sg-from-test -> main`
- **Root cause**: Missing/invalid GitHub MCP token for this session
- **Resolution**: Provided manual compare/PR URL and proceeded with branch push only
- **Prevention**: Refresh MCP GitHub token (`GITHUB_MCP_TOKEN`) before PR creation steps
- **Status**: UNRESOLVED (token still needs update)

---

- **Error**: Agent 1 CI run (`anthropics/claude-code-action`) exited with code 1
- **Discovered by**: User from GitHub Actions log screenshot
- **Root cause**: Likely CLI auth/context issue in Agent 1 step while invoking `gh pr ...` from Claude action (workflow does not explicitly provide `GH_TOKEN` to that step)
- **Resolution**: Investigated workflow and provided recommended patch (add `env: GH_TOKEN/GITHUB_TOKEN` and `github_token` input). Not applied yet per user direction.
- **Prevention**: Ensure Claude action steps that execute `gh` commands always receive explicit GitHub auth env/input
- **Status**: UNRESOLVED (workflow unchanged)

---


### Session 0313-1 (2026-03-13)

- **Error**: PATCH `/api/parts-shortage-ordering/79` returning 500 Internal Server Error when editing a case with existing Date Order
- **Discovered by**: Kelly (console screenshot showing 500 on PATCH)
- **Root cause**: API GET returns dates as ISO strings (e.g. `"2026-03-16T16:00:00.000Z"`). Edit form stores this in React state. When user saves without re-picking the date, raw ISO string is sent to PATCH endpoint. MySQL `date_order` column is `DATE` type and `STRICT_TRANS_TABLES` mode rejects the ISO format — causing SQL error.
- **Impact**: Users cannot save changes on any Parts Shortage case that already has a Date Order set, unless they re-pick the date
- **Resolution**: Added `normalizeDate()` in edit form `onSubmit` — strips ISO strings to `YYYY-MM-DD` via `.split("T")[0]`. PR #594 → test (merged).
- **Prevention**: When forms submit data loaded from an API, always normalize date fields to `YYYY-MM-DD` before sending to MySQL. JSON serialization turns MySQL DATE into ISO strings with timezone offset.
- **Status**: RESOLVED

---

- **Error**: Wrong fix applied first — Radix Dialog `onPointerDownOutside`/`onInteractOutside` handlers (PR #590)
- **Discovered by**: Kelly (tested on staging, save button still failed)
- **Root cause**: Misdiagnosed the issue as native date picker causing Radix Dialog to close. Didn't check the HTTP response (500 error) before assuming a UI framework issue.
- **Impact**: Wasted one Path B round
- **Resolution**: Reverted wrong fix, investigated actual HTTP error via console screenshot, found the real root cause (ISO date format)
- **Prevention**: When "button doesn't respond", check Network tab for HTTP errors BEFORE assuming UI framework bugs. "Saving..." stuck state means the request was sent — check if it returned an error.
- **Status**: RESOLVED (lesson captured in ai-ops)

---

### Session 0310-3 (2026-03-10)

- **Error**: Product 20525200467 showing "0 COMP" despite having 16 comp rows in DB
- **Discovered by**: Kelly (visual check on VVIP page)
- **Root cause**: Trailing slash mismatch — `Shopee_My_Products.our_link` = `.../20525200467` (no slash), `Shopee_Comp_Data.our_link` = `.../20525200467/` (trailing slash). SQL JOIN used exact match → no join → 0 comps. 1,102 rows in Shopee_Comp_Data had trailing slashes.
- **Resolution**: Changed SQL JOIN in `shopee-vvip-products-repository.ts` to `TRIM(TRAILING '/' FROM cd.our_link) = TRIM(TRAILING '/' FROM mp.our_link)`. Also added `normalizeLink()` helper in API route for consistent URL key matching.
- **Prevention**: When JOINing on URL columns from different scraper sources, always normalize (trim trailing slashes) since different bots may store URLs with or without trailing `/`.
- **Status**: RESOLVED

---

- **Error**: Products showing "0 COMP" after implementing Strategy A (bot-driven matching) when bot marked all variations as NO_MATCH
- **Discovered by**: Claude Code (testing product 23028416061 — 207 rows all NO_MATCH in Shopee_Variation_Match)
- **Root cause**: Initial Strategy A implementation only emitted rows for MATCH/UNSURE. When bot said NO_MATCH for every variation, Pass 1 found nothing and there was no Pass 2 (round-robin was removed). Result: 0 comp rows emitted → "0 COMP" badge.
- **Resolution**: Restructured logic — bot matching only controls which our_var a comp_var pairs with (Step 3). Steps 1/2/4 (top 3 selection, row collection, blank rows) stay unchanged and always emit comp products. Final version: removed round-robin entirely per Kelly's request — only bot-confirmed matches show under our variation expand.
- **Prevention**: When replacing matching logic, ensure it only affects the pairing decision, not whether comp products appear at all. Top 3 comp display should be independent of variation-level matching.
- **Status**: RESOLVED

---

- **Error**: GitHub MCP `create_pull_request` returns "Not Found" for private repo (recurring)
- **Context**: Attempted `mcp__github__create_pull_request` for `it-awesomeree/awesomeree-web-app` and `it-awesomeree/AWESOMEREE-WEB-APP`
- **Root cause**: Same recurring issue — MCP GitHub token lacks access to private org repos
- **Resolution**: Provided manual PR creation URL for Kelly
- **Prevention**: For `it-awesomeree` private repos, always use GitHub web UI for PR creation
- **Status**: RESOLVED (workaround — recurring)

---

### Session 0310-2 (2026-03-10)

- **Error**: CI test "caps tie expansion at 10 even when all have same sales" failing — expected 10, got 15
- **Discovered by**: GitHub Actions CI on PR to `test`
- **Root cause**: `getQualifiedCompRows` was rewritten to per-product ranking. Old logic had `MAX_QUALIFIED_COMP_ROWS = 10` cap. New logic keeps all tied products — 15 products with identical sales all tie, so all 15 returned.
- **Resolution**: Updated test to expect 15 and renamed to "includes all tied products when all have same sales" (commit `9ac808a`)
- **Prevention**: When rewriting business logic, always check existing tests BEFORE pushing. Run `npx jest` locally.
- **Status**: RESOLVED

---

- **Error**: Cherry-pick conflict on `app/api/shopee-my-vvip/products/route.ts` — "modify/delete"
- **Context**: Cherry-picking `a4dada6` from `fix/comp-show-all-variation-on-sg` onto `main`
- **Root cause**: VVIP route file exists on `test` but not on `main`. Cherry-pick modified a file that doesn't exist on target branch.
- **Resolution**: `git rm app/api/shopee-my-vvip/products/route.ts` — excluded from main PR since it's test-only
- **Prevention**: When cherry-picking from test-based branches to main, expect conflicts for test-only files. Plan to `git rm` them.
- **Status**: RESOLVED

---

- **Error**: GitHub MCP `create_pull_request` returns "Not Found" + `gh` CLI not installed
- **Context**: Tried both MCP and gh CLI to create PR for private repo
- **Root cause**: MCP token lacks private repo access (recurring). gh CLI not installed on this machine.
- **Resolution**: Provided manual PR creation URL for Kelly
- **Prevention**: For `it-awesomeree` private repos, always use GitHub web UI for PR creation
- **Status**: RESOLVED (workaround — recurring)

---

### Session 0310-1 (2026-03-10)

- **Error**: HTTP 500 on Shopee SG page after adding debug code to `route.ts`
- **Context**: Added `require("fs").writeFileSync(...)` and `console.log` debug statements to `app/api/shopee-sg/products/route.ts` to trace missing competitors
- **Root cause**: Debug code introduced module-level mutable state (`_debugLines` array) and `require("fs")` in a Next.js API route, which may not work in all contexts. Also added too much debug scaffolding that broke the route.
- **Resolution**: `git checkout -- app/api/shopee-sg/products/route.ts` to fully revert all debug changes
- **Prevention**: For server-side debugging in Next.js API routes, use simple `console.log` only (no file writes, no module-level state). Better yet, write standalone test scripts to reproduce logic bugs outside the server.
- **Status**: RESOLVED (reverted)

---

### Session 0306-5 (2026-03-06)

- **Error**: GitHub MCP `create_or_update_file` returns "Permission Denied: Resource not accessible by personal access token" for `it-awesomeree/comp-analysis` repo
- **Context**: Tried to push documentation files via `mcp__github__push_files`
- **Root cause**: Same recurring issue — MCP GitHub token lacks access to private org repos
- **Resolution**: Cloned repo via `git clone`, created files locally, pushed via git CLI
- **Prevention**: For `it-awesomeree` private repos, always use git CLI. MCP tools won't work.
- **Status**: RESOLVED (workaround)

---

- **Error**: `git push` rejected multiple times on comp-analysis repo
- **Context**: Remote had new commits from other contributors between push attempts
- **Root cause**: Multiple people (or CI) pushing to same branch concurrently
- **Resolution**: `git pull --rebase origin main` before each push
- **Prevention**: Always pull before pushing, especially on shared `main` branch
- **Status**: RESOLVED

---

- **Error**: Claude-in-Chrome browser extension kept disconnecting during screenshot attempts
- **Context**: Tried to take screenshots of the comp analysis page for documentation visuals
- **Root cause**: Browser extension instability — same recurring issue from previous sessions
- **Resolution**: Kelly said "nevermine, no need to put picture" — switched to ASCII art diagrams instead
- **Prevention**: For documentation visuals, prefer ASCII diagrams over browser screenshots when possible
- **Status**: RESOLVED (workaround)

---

### Session 0306-4 (2026-03-06)

- **Error**: RDP "Connection Closed — The network connection was closed unexpectedly" when connecting to VM TT
- **Discovered by**: Kelly (tried to RDP into VM TT)
- **Root cause**: TermService (Remote Desktop) was stopped on VM TT (Status: Stopped, StartType: Manual)
- **Impact**: Cannot RDP into VM TT to manage bot scripts or view desktop
- **Resolution**: Started TermService via VM Control Plane (`Start-Service -Name TermService`). Service confirmed running, but Kelly reported still unable to connect — may be a network/firewall issue beyond the service state.
- **Prevention**: Consider setting TermService StartType to Automatic on bot VMs so RDP is always available after reboot. Check firewall rules if service is running but connection still fails.
- **Status**: PARTIALLY RESOLVED (service started, connection issue may persist)

---

### Session 0306-3 (2026-03-06)

- **Error**: Merge conflicts when PR'ing `feature/shopee-sg-page` (from `main`) to `test`
- **Context**: Created branch from `main`, copied SG files. But `test` already had SG files importing from `shopee-vvip-products-repository` (old implementation).
- **Root cause**: `test` branch had existing SG code using different import paths (VVIP repo) than the new standalone repo approach. Cherry-picking/merging creates conflicts on every SG file.
- **Resolution**: Abandoned first branch. Created `feature/shopee-sg-page-v2` from `test` instead and modified existing files in-place to use the new standalone `shopee-sg-products-repository.ts`.
- **Prevention**: Before creating a branch from `main` for PR to `test`, check if `test` already has related files that would conflict. If so, branch from `test` directly.
- **Status**: RESOLVED

---

- **Error**: `next-env.d.ts` blocking git checkout
- **Context**: Trying to switch branches, got "error: Your local changes to the following files would be overwritten by checkout: next-env.d.ts"
- **Root cause**: Auto-generated TypeScript env file had local modifications from `npm run dev`
- **Resolution**: `git checkout -- next-env.d.ts` to discard local changes before switching
- **Prevention**: Add `next-env.d.ts` to `.gitignore` or always discard its changes before branch switches
- **Status**: RESOLVED

---

- **Error**: `git push` rejected on `feature/shopee-sg-page-main`
- **Context**: Push failed with "Updates were rejected because the remote contains work that you do not have locally"
- **Root cause**: CI agent or automated process had pushed new changes to the remote branch
- **Resolution**: `git pull --rebase origin feature/shopee-sg-page-main` then push again
- **Prevention**: Always pull before pushing, especially on branches that CI agents may modify
- **Status**: RESOLVED

---

- **Error**: GitHub MCP `create_pull_request` returns "Not Found" for private repo
- **Context**: Attempted `mcp__github__create_pull_request` for `it-awesomeree/AWESOMEREE-WEB-APP`
- **Root cause**: Same recurring issue — MCP GitHub token lacks access to private org repos
- **Resolution**: Provided Kelly with manual PR creation URLs for both `main` and `test` PRs
- **Prevention**: For `it-awesomeree` private repos, always use git CLI for push and GitHub web UI for PR creation
- **Status**: RESOLVED (workaround — recurring)

---

### Session 0306-2 (2026-03-06)

- **Error**: CI tests failing — "filters out rows with monthlySales = 0" expected length 1, got 2; "filters out rows with monthlySales = null or undefined" expected length 1, got 3
- **Discovered by**: GitHub Actions CI on PR #493
- **Root cause**: Code intentionally removed the `> 0` filter in `getQualifiedCompRows()` (commit `fd236fe`) so competitors with 0/null/undefined monthly sales are now included. Tests still expected old filtering behavior.
- **Resolution**: Updated 2 tests to expect new behavior — renamed test descriptions and changed expected lengths (commit `3ccfd47`)
- **Prevention**: When changing business logic that has test coverage, always update the corresponding tests in the same commit or follow-up commit before pushing.
- **Status**: RESOLVED

---

### Session 0304-2 (2026-03-04)

- **Error**: Cherry-pick from `test`-based branch onto `main`-based branch causes massive conflicts
- **Context**: Tried creating `feature/shopee-sg-comp-analysis-v2` from `main` and cherry-picking the SG commit — to follow Agnes's PR flow (branch from `main`)
- **Root cause**: VVIP files (`shopee-vvip-products-repository.ts`, `app/api/shopee-my-vvip/*`, nav tabs) only exist on `test`, not `main`. Cherry-pick reports "modify/delete" conflicts for all these files.
- **Impact**: Cannot create a `main`-based branch for SG feature — too many dependencies on `test`-only code
- **Resolution**: Aborted cherry-pick, kept `test`-based branch. Need to clarify with Agnes whether `test`-based branching is acceptable when feature depends on `test`-only files.
- **Prevention**: Before following branch-from-main workflow, check if modified files exist on `main`. If they don't, branch from `test` and discuss with lead.
- **Status**: RESOLVED (workaround — using `test`-based branch)

---

- **Error**: GitHub MCP tools returning "Not Found" for `it-awesomeree/awesomeree-web-app`
- **Context**: Tried `mcp__github__create_pull_request` to create PR for SG feature branch
- **Root cause**: Same recurring issue — MCP GitHub token lacks access to private org repo
- **Resolution**: Pushed branch via git CLI, PR to be created manually via GitHub web UI
- **Prevention**: For `it-awesomeree` private repos, always use git CLI for push and GitHub web UI for PR creation
- **Status**: RESOLVED (workaround)

---

### Session 0304-1 (2026-03-04)

- **Error 1**: HTTP 500 — "Column 'our_link' in where clause is ambiguous"
- **Context**: After changing `GROUP BY product_name` to `GROUP BY our_link`, the rows query's WHERE clause used `our_link IN (...)` without a table prefix
- **Root cause**: The rows query JOINs `Shopee_Comp sc` with exclusion tables (`pe`, `ve`) that also have an `our_link` column. MySQL couldn't determine which table's column to use.
- **Resolution**: Added `sc.` prefix → `sc.our_link IN (...)`. The old `product_name IN (...)` never had this issue because exclusion tables don't have a `product_name` column.
- **Prevention**: When changing column references in queries with JOINs, always check if the new column exists in joined tables. Use table alias prefix (`sc.column`) for safety.

- **Error 2**: HTTP 500 — "productNames is not defined"
- **Context**: After renaming variable `productNames` to `groupValues`, one reference was missed
- **Root cause**: Line 1463 still used `productNames.length` after the rename. Only triggered when shop filter was applied (specific code path).
- **Resolution**: Changed to `groupValues.length`
- **Prevention**: After renaming a variable, always search the entire file for all remaining references (`grep productNames`). Don't rely on just the lines you changed.

---

### Session 0303-2 (2026-03-03)

- **Error**: Shopee MY analytics table date filter hides product variations — 14 unique SKUs reduced to 10 when date filter applied
- **Discovered by**: Kelly (filtered by March 2 on product 20525200467, saw only 10 VVIP SKUs instead of 14)
- **Root cause**: `shopee-products-repository-mysql.ts` uses row-level date filter (`date_taken >= ? AND date_taken <= ?`). The scraper inserts variations as separate rows with different `date_taken` per scrape run. 4 SKUs (BW-RSP, INTX530GPH with sheet_name="NONE") were scraped on March 3 while VVIP data was scraped March 2. Row-level filter excludes cross-date variations.
- **Impact**: Users see incomplete variation data when any date filter is active. Also affects comp analysis analyze API (exact `date_taken` match hid competitor variations scraped on different dates).
- **Resolution**: Changed date filter to product-level subquery: `our_link IN (SELECT DISTINCT our_link FROM Shopee_Comp WHERE date_taken >= ? AND date_taken <= ?)`. Frontend `deduplicateCompetitorRows()` keeps latest data per variation. Also fixed comp analysis analyze API with `ROW_NUMBER()` dedup.
- **Prevention**: When filtering time-series data with per-row timestamps, use entity-level filters (match the product, then return all its rows) rather than row-level filters, especially when scrapers don't guarantee all variations are scraped simultaneously.
- **Status**: Code done locally, NOT yet deployed
- **Note**: DB dates stored in MYT but MCP tool displays UTC (subtract 8h). `2026-03-01T16:00:00Z` in MCP = `2026-03-02 00:00:00` in MySQL.

---

### Session 0303-1 (2026-03-03)

- **Error**: WhatsApp screenshot bots on VM TT failing with `TimeoutException` — cannot find WhatsApp search box
- **Discovered by**: Kelly (ran `TT-AcountHealthScreenshot.py` on VM TT, got TimeoutException at line 90)
- **Root cause**: WhatsApp Web DOM update changed the search box from `<p>` element (inside nested divs with obfuscated classes) to `<input role="textbox" data-tab="3">`. Old CSS selector `div.x1hx0egp.x6ikm8r.x1odjw0f.x6prxxf.x1k6rcq7.x1whj5v > p` no longer matches.
- **Impact**: All 3 screenshot scripts (TikTok, Shopee MY, Shopee SG) could not send reports to "Awesomeree Discussion" WhatsApp group
- **Resolution**: Updated selector to `input[data-tab='3']` in all 3 files via VM Control Plane `vm_execute`
- **Prevention**: Use stable attribute-based selectors (`data-tab`, `aria-label`, `role`) instead of obfuscated CSS class names. WhatsApp Web frequently changes class names but keeps functional attributes stable.
- **Status**: RESOLVED

---

- **Error**: GitHub MCP token cannot access `it-awesomeree/bot-scripts` private repo
- **Discovered by**: Claude Code (all MCP GitHub API calls return "Not Found")
- **Root cause**: Same recurring issue — MCP GitHub token not authorized for this private repo. Token update attempted but did not resolve.
- **Impact**: Cannot create PR or push code via MCP tools
- **Resolution**: Pending — need to either update token permissions or push manually via GitHub Desktop
- **Prevention**: For `it-awesomeree` private repos, default to GitHub Desktop or browser-based PR creation
- **Status**: UNRESOLVED

---

### Session 0302-1 (2026-03-02)

- **Error**: Bug 4 — BBM Kerusi Lipat Bamboo Chair showed only 4/8 comp variations in VVIP page
- **Discovered by**: Kelly (visual check — expanded competitor, only Brown variations visible, Wood missing)
- **Root cause**: `getQualifiedCompRows()` in shared `shopee-history-calculations.ts` applies per-variation top 3 filter. BBM Wood variations (52 sales) shared an our_var ("D.Wood - Grey Set") with NIVISON (988 sales). Per-variation top 3 kept NIVISON and dropped BBM Wood. BBM Brown survived on a different our_var with no NIVISON competition.
- **Resolution**: Added optional `backfillProducts` parameter to `getQualifiedCompRows()`. When enabled, if a comp product survives the top 3 on ANY variation, ALL its variations are back-filled from original data. Defaults to `false` (Shopee MY unchanged), VVIP passes `true`.
- **Prevention**: When applying per-variation filters, consider that a comp product may have variations spread across different our_vars with different competition levels. Product-level survival should include all variations.
- **Status**: RESOLVED

---

- **Error**: MySQL query `WHERE comp_product LIKE 'BBM Kerusi Lipat Bamboo%'` returned empty results
- **Discovered by**: Claude Code (DB query returned 0 rows when product clearly existed)
- **Root cause**: Product name in `Shopee_Comp_Data` contained invisible BOM characters: `BBM﻿ Kerusi Lipat Bamboo...` — the `﻿` after "BBM" is a Unicode BOM (U+FEFF)
- **Resolution**: Used wildcard pattern `LIKE '%Bamboo%Foldable%Chair%'` to bypass the BOM characters
- **Prevention**: When querying scraped data, use `%keyword%` patterns instead of prefix matching. Scraped text may contain invisible Unicode characters.
- **Status**: RESOLVED

---

- **Error**: GitHub MCP tools returning "Not Found" for `it-awesomeree/AWESOMEREE-WEB-APP` private repo
- **Discovered by**: Claude Code (`mcp__github__create_pull_request` failed with "Resource not found")
- **Root cause**: GitHub MCP server token does not have access to the private org repo (same issue as session 0226-1)
- **Resolution**: Pushed branch via git CLI, provided PR description for Kelly to create manually via GitHub web UI
- **Prevention**: For this org's private repos, use git CLI for push and GitHub web UI for PR creation. MCP tools won't work.
- **Status**: RESOLVED (workaround)

---

### Session 0227-1 (2026-02-27)

- **Error**: Parent category badge showing "VIP" instead of "VVIP" for products that only have VVIP + NONE in database
- **Discovered by**: Kelly (visual check on analytics table)
- **Root cause**: Code used `groupMetrics?.category ?? "VIP"` — server enrichment stores metrics in Map keyed by product name, lookup uses each product's own name. When names mismatch (or multiple product_ids share same name), lookup returns undefined, falls back to "VIP"
- **Resolution**: Changed to `getHighestPriorityCategory(groupRows)` — computes parent category live from actual variation data
- **Prevention**: Avoid relying on name-based Map lookups for data keyed by product_id. Compute values from source data where possible.
- **Status**: RESOLVED

---

- **Error**: Products with same product_id but different names (seller renamed) showing as separate parent groups
- **Discovered by**: Kelly (visual check — product 22026149091 appeared as two groups)
- **Root cause**: `grouped-rows.tsx` had its own LOCAL `groupProducts()` function that grouped by product **name** (`byName` Map). The shared version in `shopee-history-calculations.ts` groups by product_id extracted from URL. The component used the local (wrong) version.
- **Resolution**: Removed local `groupProducts` from `grouped-rows.tsx`, imported the shared version that groups by product_id. Also fixed duplicate React key error by using `group.productId ?? group.name` as key.
- **Prevention**: Don't duplicate shared functions locally in components. Always import from the shared module. Search codebase for duplicate function names when fixing bugs.
- **Status**: RESOLVED

---

- **Error**: Category sort mixing across pages — VVIP on page 1, then VVIP again on page 2 instead of continuous sort
- **Discovered by**: Kelly (visual check — changed to 100 rows per page, compared page 1 vs page 2)
- **Root cause**: Server paginates by `GROUP BY product_name` and sorts by `MAX(sheet_name priority)`. Client groups by `product_id`. Different product_ids sharing the same product_name get the same server sort values. Also, server uses date-filtered data for sorting while client sees all dates (Stage 2 is date-free). This mismatch means server and client compute different category priorities for some products.
- **Resolution**: PARTIALLY FIXED — added live category computation per product_id group for within-page sort accuracy. Cross-page consistency requires server-side change to paginate by product_id instead of product_name.
- **Prevention**: When changing client-side grouping strategy (name → product_id), ensure server-side pagination/sorting uses the same grouping key.
- **Status**: UNRESOLVED (cross-page mixing — needs server-side GROUP BY product_id)

---

### Session 0226-1 (2026-02-26)

- **Error**: Claude-in-Chrome browser extension repeatedly disconnecting during GRBT-155 review workflow
- **Discovered by**: Kelly + Claude Code (tool calls returning connection errors)
- **Root cause**: Browser extension instability — disconnects triggered by navigating between GitHub and Jira pages, especially on GitHub file views with large code content
- **Impact**: Multiple workflow steps interrupted — had to retry Jira comments, GitHub changelog creation, and ticket transitions. Some steps done manually by Kelly.
- **Resolution**: Workaround — Kelly toggled extension off/on and restarted Chrome multiple times. No permanent fix.
- **Prevention**: Save work frequently between steps. For long workflows, expect disconnects and plan manual fallbacks for GitHub operations.
- **Status**: UNRESOLVED (recurring issue, no fix available)

---

- **Error**: Content filter blocking JavaScript code extraction from GitHub file view
- **Discovered by**: Claude Code (response contained `[BLOCKED: Cookie/query string data]` placeholders)
- **Root cause**: Claude-in-Chrome content filter flagged code containing DB connection strings, environment variables, and SQL queries as sensitive data
- **Impact**: Could not extract full `stockID.py` source code via browser JavaScript for code review
- **Resolution**: Used Task agent (subagent) to analyze code structure from available excerpts; extracted only non-sensitive function signatures and flow logic via targeted JavaScript
- **Prevention**: For code review of files with DB credentials/env vars, use GitHub API or git clone instead of browser-based extraction
- **Status**: RESOLVED (workaround applied, review completed)

---

- **Error**: GitHub MCP tools returning "Not Found" for private repo `it-awesomeree/bot-scripts`
- **Discovered by**: Claude Code (`mcp__github__get_file_contents` and `mcp__github__get_pull_request_files` both failed)
- **Root cause**: GitHub MCP server token does not have access to the `it-awesomeree` organization's private repositories
- **Impact**: Could not use MCP tools for file reads or PR diff retrieval on bot-scripts repo
- **Resolution**: Used browser-based authentication instead (GitHub web UI via Claude-in-Chrome, JavaScript fetch calls with session cookies)
- **Prevention**: For private org repos, default to browser-based access. MCP GitHub tools only work for repos the token has access to.
- **Status**: RESOLVED (workaround applied)

---

- **Error**: GitHub main branch protected — cannot commit changelog directly
- **Discovered by**: Kelly (saw "You can't commit to main because it is a protected branch" in GitHub web UI)
- **Root cause**: `bot-scripts` repo has branch protection rules on `main` requiring PR for all changes
- **Impact**: Could not create `docs/changelog/GRBT-155.md` directly on main
- **Resolution**: Kelly created branch `KellyEe0111-patch-1`, proposed changes via PR, then merged through GitHub web UI
- **Prevention**: For bot-scripts repo, always create a branch first. Use PR workflow for all changes including documentation.
- **Status**: RESOLVED

---

<!-- Template for new entries:

### Session MMDD-N (YYYY-MM-DD)

- **Error**: [What went wrong — error message or symptom]
- **Discovered by**: [Who/what found it — e.g., Kelly (visual check), CI pipeline, user report]
- **Root cause**: [Why it happened — the actual technical reason]
- **Impact**: [What was affected — e.g., wrong data displayed, build broken, users blocked]
- **Resolution**: [How it was fixed — what was changed and where]
- **Prevention**: [How to avoid this in the future — process change, test, check]
- **Status**: [RESOLVED / UNRESOLVED — include commit hash if resolved]

-->

