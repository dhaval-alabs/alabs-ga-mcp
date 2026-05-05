# Brief: Fix & Re-enable `lookup_gclid` Tool
**Repo:** `dhaval-alabs/alabs-ga-mcp`  
**Priority:** High  

---

## Background

The `lookup_gclid` tool is currently throwing the following Google Ads API error:

```
clickViewError: EXPECTED_FILTER_ON_A_SINGLE_DAY
"Queries including ClickView must have a filter limiting the results to one day."
```

Additionally, the tool may have been disabled/commented out during a prior debugging session and needs to be confirmed active.

---

## Action 1 — Fix the GAQL Query (Day-by-Day Loop)

**Root cause:** The tool currently queries `click_view` across a date range in a single GAQL call. The Google Ads API requires ClickView queries to be scoped to **exactly one day** per request.

**Fix:** Replace the range query with a loop that iterates one day at a time, from today going back up to the `days` lookback window, stopping as soon as the GCLID is found.

**GAQL pattern per iteration:**
```sql
SELECT
  click_view.gclid,
  campaign.name,
  ad_group.name,
  segments.date
FROM click_view
WHERE segments.date BETWEEN 'YYYY-MM-DD' AND 'YYYY-MM-DD'
  AND click_view.gclid = '{gclid}'
LIMIT 1
```

> Note: Start and end date must be the **same date** for each call. Loop backwards day by day until a result is found or the lookback window is exhausted.

---

## Action 2 — Confirm Tool is Active & Exposed

1. Check that `lookup_gclid` is **not commented out** in the MCP server route/handler.
2. Confirm it appears in the tool manifest returned by the MCP server.
3. Run a quick end-to-end test with a known GCLID before closing this out.

---

## Acceptance Criteria

- [ ] `lookup_gclid` tool is active and visible in the MCP tool list
- [ ] A valid GCLID resolves correctly with campaign/ad group details
- [ ] No `EXPECTED_FILTER_ON_A_SINGLE_DAY` error is thrown
