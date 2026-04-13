# Bug Report — Inventory Management

Top 15 user-facing bugs, ranked by user impact. Paths are relative to the repo root. Line numbers correspond to the current `alex_features` branch.

Prerequisites for reproducing: backend on `http://localhost:8001`, frontend on `http://localhost:3000`, fresh browser session.

---

## 1. Category spending percentages sum to >100%
**Severity:** High
**Files:** `server/data/spending.json` → rendered at `client/src/views/Spending.vue:121`

**Symptom:** On the Spending page, the "Category Spending Breakdown" list shows percentages that sum to ~123.6% (Raw Materials 42.5% + Components 38.8% + Equipment 19.0% + Consumables 23.3%), which is mathematically impossible for "% of total".

**Repro:**
1. Navigate to `/spending`.
2. Scroll to "Category Spending Breakdown".
3. Add the `%` values — you get 123.6%, not 100%.

**Root cause:** Percentage values in the `category_spending` array of `server/data/spending.json` are hardcoded and no longer agree with the dollar amounts or the total.

---

## 2. "Create PO" button shown for backlog items that already have a PO
**Severity:** High
**File:** `client/src/views/Dashboard.vue:213`

**Symptom:** In the Dashboard "Inventory Shortages" table, every row shows a "Create PO" button on initial load, even when that backlog item already has a purchase order. The button should read "View PO".

**Repro:**
1. Load the Dashboard.
2. Observe the Inventory Shortages table — every row has "Create PO".
3. Inspect network response for `/api/backlog` — entries include `has_purchase_order: true` for several items.

**Root cause:** Template uses `v-if="!item.purchase_order_id"`, but the API returns the boolean `has_purchase_order`. The field name on the item never matches, so the condition is always truthy until a new PO is created in-session.

---

## 3. Inventory & Orders prices don't convert when switching to JPY
**Severity:** High
**Files:** `client/src/views/Inventory.vue:63-64`, `client/src/views/Orders.vue:59`, `client/src/views/Orders.vue:71`

**Symptom:** Switching locale to Japanese changes the currency symbol to ¥ but the number stays in USD (e.g., "¥24.99" instead of the expected ¥3,749 at 150× conversion).

**Repro:**
1. Open the locale/language switcher and select Japanese.
2. Go to `/inventory` — unit prices still show USD numbers with a ¥ prefix.
3. Go to `/orders` — unit price and order total have the same bug.
4. Compare with the Dashboard, which uses the shared `formatCurrency` helper and renders the converted value correctly.

**Root cause:** These two views concatenate `currencySymbol + value.toLocaleString()` inline instead of calling `formatCurrency`, which is the helper that applies the 150× JPY multiplier.

---

## 4. Spending summary totals don't match monthly breakdown
**Severity:** High
**Files:** `server/data/spending.json`, exposed via `/api/spending/summary` and `/api/spending/monthly` (`server/main.py`)

**Symptom:** On the Spending page, the "Total Procurement" and "Total Labor" KPIs disagree with the sum of the monthly chart. Monthly sums to ≈ $7.415M procurement / $5.838M labor; summary reads $7.37M / $4.959M. Change-% badges are computed against one source, totals against the other, so the whole page looks inconsistent.

**Repro:**
1. Navigate to `/spending`.
2. Sum the monthly procurement bars (or check the monthly endpoint directly).
3. Compare to the headline "Total Procurement" KPI — they differ.

**Root cause:** Summary values in `spending.json` were not regenerated when the monthly breakdown changed — the two sections drifted.

---

## 5. Inventory "Location" column mistranslated as a warehouse city
**Severity:** Medium
**File:** `client/src/views/Inventory.vue:65`

**Symptom:** In Japanese locale, inventory rows show "倉庫A-12" — a kanji word for "warehouse" followed by an English bay code. The column header says "Location" but the values are shelf labels like "Warehouse B-09", not cities.

**Repro:**
1. Go to `/inventory`.
2. Switch to Japanese.
3. Observe the Location column — produces mixed-script mojibake-like output.

**Root cause:** Template calls `translateWarehouse(item.location)` on a shelf label. The helper is for city names (e.g., "Tokyo"); it substring-replaces "Warehouse" and breaks shelf codes.

---

## 6. Fill Rate progress bar overflows when value > goal [FIXED]
**Severity:** Medium
**File:** `client/src/views/Dashboard.vue:43`
**Status:** Fixed — width now clamped via `Math.min(fillRate / 95 * 100, 100)`.

**Symptom:** When Order Fill Rate is 96.8% against a 95% goal, the progress bar visually overflows its container (width becomes 101.9%).

**Repro:**
1. Load the Dashboard with a dataset where fill rate > 95%.
2. Inspect the Fill Rate progress bar — its inner fill extends beyond the track.

**Root cause:** The width is computed as `(actual / goal) * 100` with no `Math.min(..., 100)` clamp. The sibling revenue KPI clamps correctly; this one doesn't.

---

## 7. Orders table renders "Invalid Date"
**Severity:** Medium
**File:** `client/src/views/Orders.vue:149`

**Symptom:** If an order has a missing or malformed date, the cell renders the literal string "Invalid Date" instead of a dash.

**Repro:**
1. In `server/data/orders.json`, temporarily null out or corrupt an `order_date` field.
2. Restart backend; visit `/orders`.
3. The row shows "Invalid Date".

**Root cause:** The formatter calls `new Date(dateString).toLocaleDateString(...)` without validating the input. Dashboard's equivalent formatter has `if (!dateString) return '-'`; Orders does not.

---

## 8. "First Order" column is earliest-in-filter, not lifetime
**Severity:** Medium
**File:** `client/src/views/Dashboard.vue:494`

**Symptom:** When a month filter is applied, the "First Order" date in the Top Products table shows the earliest date *within* the filtered window, not the SKU's actual first-ever order. Users read the column as lifetime history.

**Repro:**
1. On the Dashboard, change the Time Period filter to a single month (e.g., "May").
2. Note the "First Order" dates for products — all fall inside May.
3. Switch back to "All Months" — dates jump back in time.

**Root cause:** `topProducts` iterates `allOrders`, which is already month-filtered, and picks the min date from that set. Column label should be "First Order in Range" or computation should use an unfiltered dataset.

---

## 9. Reports "vs previous month" breaks across gaps
**Severity:** Medium
**File:** `client/src/views/Reports.vue:87,93`

**Symptom:** If a month has zero orders, the backend omits it entirely. The UI then computes "change vs previous month" against the month before the gap, but still labels it "vs previous month". Growth is over- or understated with no warning.

**Repro:**
1. In `server/data/orders.json`, remove all orders for a single month (e.g., February).
2. Restart backend; visit `/reports`.
3. March's "Change" column compares against January but is labeled as month-over-month.

**Root cause:** `getMonthlyTrends` emits only months with orders. UI uses `monthlyData[index-1]` without verifying that the previous entry is truly the previous calendar month.

---

## 10. Reports currency formatting inconsistent with the rest of the app
**Severity:** Medium
**File:** `client/src/views/Reports.vue:214-240`

**Symptom:** Reports page renders `$1,234,567.00` with forced `.00`, and can produce `-$0.00` for tiny negative change values. Dashboard/Spending use `toLocaleString` with zero fraction digits, so numbers look different across pages.

**Repro:**
1. Compare a total revenue figure on the Dashboard ("$1,234,567") to the same figure on Reports ("$1,234,567.00").
2. Trigger a near-zero negative change (e.g., one month shrinks by $0.10) — the badge reads `-$0.00`.

**Root cause:** Home-rolled `formatNumber` helper always appends two decimals and has no sign-zero guard.

---

## 11. Quarterly revenue columns lack scaling and wrap on wide values
**Severity:** Low
**Files:** `client/src/views/Reports.vue:31-32`, backend: `server/main.py:268`

**Symptom:** "Total Revenue" and "Avg Order Value" columns render raw dollar amounts (e.g., `$2,456,789.00`) with no K/M suffix. Combined with the verbose change values (`-$1,234,567.00`) from bug #10, columns wrap at normal widths.

**Repro:**
1. Visit `/reports`.
2. Narrow the browser window to ~1200px.
3. Revenue cells wrap onto two lines.

**Root cause:** No abbreviation helper; Dashboard uses a shortening helper for its KPI cards but Reports does not.

---

## 12. Spending transaction IDs padded as if numeric; no sort
**Severity:** Low
**File:** `client/src/views/Spending.vue:153`

**Symptom:** Transaction IDs like `TXN-2025-0901` are run through `.toString().padStart(3, '0')` — a no-op on strings that are already 10+ chars. Transactions also render in the raw JSON order with no tie-breaking, so the table looks scrambled by date/amount.

**Repro:**
1. Visit `/spending`.
2. Scroll to the Transactions table — IDs render fine (no visible effect) but sort is inconsistent.

**Root cause:** `padStart` call is meaningless for non-numeric IDs; no sort function applied to the list.

---

## 13. Backlog page not localized
**Severity:** Medium
**File:** `client/src/views/Backlog.vue:13,17,21,71-73`

**Symptom:** With Japanese selected, the Backlog page still renders English stat labels ("Total Backlog", "High Priority", etc.) and raw priority values ("high", "medium", "low") in priority badges.

**Repro:**
1. Switch locale to Japanese.
2. Navigate to `/backlog`.
3. Stat card labels and priority badges remain English.

**Root cause:** Backlog.vue was never wired to `useI18n` — it has no `t(...)` calls and uses `item.priority` directly as both class and label.

---

## 14. Reports uses `:key="index"` in v-for
**Severity:** Low
**File:** `client/src/views/Reports.vue:28,51,82`

**Symptom:** When filters change the order or length of the list, Vue reuses DOM nodes incorrectly — transient glitches like the wrong badge color staying attached to a row for one frame.

**Repro:**
1. On `/reports`, change the filter set (e.g., switch warehouse) multiple times quickly.
2. Watch badge colors — occasional one-frame mismatches.

**Root cause:** Keys should be stable identifiers (`q.quarter`, `month.month`) instead of array index.

---

## 15. Inventory Shortages hidden by warehouse filter
**Severity:** Medium
**Files:** `client/src/views/Dashboard.vue:551-559`, `client/src/views/Backlog.vue:101-109`

**Symptom:** Selecting a warehouse filter (e.g., "Tokyo") hides backlog items whose SKU happens to be stocked in a different warehouse, even though they are still real shortages. User believes there are no shortages when there are.

**Repro:**
1. On the Dashboard, set Warehouse filter to "Tokyo".
2. Note Inventory Shortages table shrinks or empties.
3. Clear the filter — shortages reappear.

**Root cause:** Backlog is filtered by intersection with currently visible inventory SKUs. Backlog is a global demand-vs-supply problem and shouldn't be scoped by warehouse visibility.

---

## Notes on items excluded from the top 15

- `useFilters` preserves status filter when navigating to pages that don't support it (e.g., Inventory). Minor; not a visible break.
- Transaction-row click opens a browser `alert()`. UX decision, not a bug.
