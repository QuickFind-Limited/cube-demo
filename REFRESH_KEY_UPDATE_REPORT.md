# Pre-Aggregation Refresh Key Update Report

**Date:** December 13, 2025  
**Task:** Disable automatic pre-aggregation refreshes by setting `refresh_key: every: 365 days`

## Summary

Successfully updated all pre-aggregations across 26 Cube YAML files to use `every: 365 days` refresh interval, effectively disabling automatic refreshes.

## Files Modified (26 total)

1. b2b_addresses.yml - 1 pre-aggregation updated
2. b2b_customer_addresses.yml - 1 pre-aggregation updated
3. b2b_customers.yml - 1 pre-aggregation updated
4. b2c_customer_channels.yml - 1 pre-aggregation updated
5. b2c_customers.yml - 1 pre-aggregation updated
6. classifications.yml - 1 pre-aggregation updated
7. cross_sell.yml - 1 pre-aggregation updated
8. currencies.yml - 1 pre-aggregation updated
9. departments.yml - 1 pre-aggregation updated
10. fulfillment_lines.yml - 3 pre-aggregations updated
11. fulfillments.yml - 1 pre-aggregation updated
12. inventory.yml - 2 pre-aggregations updated
13. item_receipt_lines.yml - 2 pre-aggregations updated
14. items.yml - 1 pre-aggregation updated
15. landed_costs.yml - 1 pre-aggregation updated
16. locations.yml - 1 pre-aggregation updated
17. on_order_inventory.yml - 2 pre-aggregations updated
18. order_baskets.yml - 1 pre-aggregation updated
19. purchase_order_lines.yml - 1 pre-aggregation updated
20. purchase_orders.yml - 1 pre-aggregation updated
21. sell_through.yml - 1 pre-aggregation updated
22. sell_through_seasonal.yml - 1 pre-aggregation updated
23. subsidiaries.yml - 1 pre-aggregation updated
24. supplier_lead_times.yml - 1 pre-aggregation updated
25. transaction_lines.yml - 17 pre-aggregations updated
26. transactions.yml - 1 pre-aggregation updated

## Total Pre-Aggregations Updated

**47 pre-aggregations** across 26 files

## Changes Made

### Before:
```yaml
refresh_key:
  every: 24 hour  # or "1 hour" or "1 day"
```

### After:
```yaml
refresh_key:
  every: 365 days
```

## Verification

All files verified to contain only `every: 365 days` refresh keys. No files with old refresh intervals remain.

## Issues Encountered

None - all updates completed successfully.

## Notes

- Files with multiple pre-aggregations were updated using `replace_all: true` for efficiency
- The 365-day interval effectively disables automatic refreshes while maintaining valid YAML syntax
- Pre-aggregations can still be manually refreshed on demand via Cube.js API or CLI
- Largest number of pre-aggregations in a single file: transaction_lines.yml (17 pre-aggs)

## Next Steps

Pre-aggregations will no longer refresh automatically. To refresh them:
1. Use Cube.js API: `POST /cubejs-api/v1/pre-aggregations/build`
2. Use Cube CLI: `npx cubejs-cli pre-aggregations build`
3. Trigger manual refresh through Cube Cloud UI (if applicable)
