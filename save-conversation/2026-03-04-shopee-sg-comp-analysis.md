# 2026-03-04 — Shopee SG Comp Analysis Page

## Session Info
- **Date**: 2026-03-04
- **Branch**: `feature/shopee-sg-comp-analysis` (cherry-picked from `kelly-sg-page-comp-analysis`)
- **Target**: `test`
- **PR**: Not yet created — branch pushed, create PR manually via GitHub web UI
- **PR URL**: https://github.com/it-awesomeree/awesomeree-web-app/pull/new/feature/shopee-sg-comp-analysis

## What Was Done

### Added Shopee SG Comp Analysis Page
- New page at `/analytics/table/shopee-sg` — identical functionality to VVIP but filters by `region='SG'`
- Reuses the VVIP repository (`shopee-vvip-products-repository.ts`) with an optional `region` parameter — zero code duplication
- VVIP routes updated to explicitly pass `region = "MY"` for data isolation
- SG routes pass `region = "SG"`
- Added "Shopee SG" tab in desktop + mobile navigation (between VVIP and TikTok)
- Added `"Shopee SG"` to `TabType` union in `lib/type.ts`
- Currently shows empty state (no SG data in DB yet) — will auto-populate when bot adds SG rows

### Data Isolation Design
- **Shopee MY**: Uses completely different repo file (`shopee-products-repository.ts`) — NOT touched at all
- **VVIP**: Now explicitly filters `region = 'MY'` in WHERE + JOIN — will never show SG data
- **Shopee SG**: Filters `region = 'SG'` in WHERE + JOIN — only shows SG data
- Cache key includes `region` to prevent cross-contamination between VVIP and SG

## Files Created (6)

| File | Purpose |
|------|---------|
| `app/analytics/table/shopee-sg/layout.tsx` | Simple pass-through layout (7 lines) |
| `app/analytics/table/shopee-sg/lib/api.ts` | Frontend API client, all endpoints → `/api/shopee-sg/products` |
| `app/analytics/table/shopee-sg/page.tsx` | Full page copied from VVIP, all refs changed to "Shopee SG" |
| `app/api/shopee-sg/products/route.ts` | Main API route with `REGION = "SG"` |
| `app/api/shopee-sg/products/counts/route.ts` | Tab counts with `REGION = "SG"` |
| `app/api/shopee-sg/products/details/route.ts` | Product details with `REGION = "SG"` |

## Files Modified (7)

| File | Change |
|------|--------|
| `lib/services/shopee-vvip-products-repository.ts` | Added optional `region?` param to `fetchVvipProducts`, `fetchVvipProductCounts`, `fetchVvipProductDetails`. Added `vvipJoinForRegion()` helper. Region added to cache key, `buildWhere()`, `buildMpOnlyWhere()`. |
| `components/analysis-sub-navigation.tsx` | Added `{ title: "Shopee SG", href: "/analytics/table/shopee-sg" }` after VVIP |
| `components/analysis-sub-navigation-mobile.tsx` | Same tab addition |
| `lib/type.ts` | Added `"Shopee SG"` to `TabType` union |
| `app/api/shopee-my-vvip/products/route.ts` | Added `const REGION = "MY"` and pass to `fetchVvipProducts(filters, REGION)` |
| `app/api/shopee-my-vvip/products/counts/route.ts` | Added `const REGION = "MY"` and pass to `fetchVvipProductCounts(REGION)` |
| `app/api/shopee-my-vvip/products/details/route.ts` | Added `const REGION = "MY"` and pass to `fetchVvipProductDetails(ids, REGION)` |

## How Region Filtering Works in SQL

**VVIP (region = 'MY'):**
```sql
SELECT ... FROM Shopee_My_Products mp
LEFT JOIN Shopee_Comp_Data cd ON cd.our_link = mp.our_link AND cd.region = 'MY'
WHERE mp.region = 'MY' AND mp.status = 'active'
```

**SG (region = 'SG'):**
```sql
SELECT ... FROM Shopee_My_Products mp
LEFT JOIN Shopee_Comp_Data cd ON cd.our_link = mp.our_link AND cd.region = 'SG'
WHERE mp.region = 'SG' AND mp.status = 'active'
```

## SG Page String Replacements (from VVIP copy)
- API import: `shopee-my-vvip/lib/api` → `shopee-sg/lib/api`
- TabType: `"Shopee MY"` → `"Shopee SG"`
- Display text: `Shopee MY` → `Shopee SG`
- Log labels: `[shopee-my-vvip]` → `[shopee-sg]`
- model_name: `shopee_products_vvip` → `shopee_products_sg`
- localStorage keys: `shopeeMY:*` → `shopeeSG:*` (column visibility, widths, density)

## Build Verification
- Zero new TS errors from our changes
- All TS errors in SG `page.tsx` are identical pre-existing ones from VVIP `page.tsx`
- Build failures are pre-existing (missing `@tanstack/react-virtual`, `cron-parser`) — unrelated

## Similarity Status
- All 10 similarity columns are currently `NULL AS` in the shared repository (`shopee-vvip-products-repository.ts` lines 168-177)
- This matches the current `test` branch state — similarity is NOT wired up yet
- The stash on `testing-on-the-comp2` had the full similarity + exclusions JOIN, but we built SG on `test`'s current state
- When similarity is ready: swap the 10 `NULL AS` lines to `mp.*` in the repository — applies to BOTH VVIP and SG automatically since they share the same repo file
- Exclusions JSON (`product_exclusions_json` / `variation_exclusions_json`) also returns NULL — can wire up later
- Bot needs to populate similarity scores in DB first — UI will auto-display once data exists

## Pending / Next Steps
- [ ] Create PR via GitHub web UI (branch already pushed)
- [ ] Get Agnes's approval on PR
- [ ] SG page currently shows empty — will show data when bot populates `region = 'SG'` rows
- [ ] Similarity: not wired up yet (NULL AS) — when ready, change in shared repo applies to both VVIP and SG
- [ ] Future: adding more regions (TH, PH) = just a new API route + page, no repo changes
