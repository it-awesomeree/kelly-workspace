# Error Log

Errors encountered during development and how they were resolved. Prevents repeat debugging.

**Format**: Each entry follows the standard structure below. Add newest entries at the top.

**Session ID Convention**: Use `MMDD-N` format (e.g., `0219-1`) matching WORKLOG sessions.

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
