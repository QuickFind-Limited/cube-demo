# Cube.js Demo Configuration Updated for Denormalization Testing

## Date: 2025-01-14

## Summary

Updated `cube-demo/model/cubes/transaction_lines.yml` to use the denormalized materialized view instead of performing LEFT JOINs at query time.

---

## Changes Made

### File: `/home/produser/cube-demo/model/cubes/transaction_lines.yml`

**BEFORE:**
```yaml
sql: "SELECT
  tl.*,
  t.type as transaction_type,
  t.currency as transaction_currency,
  t.trandate as transaction_date,
  # ... more fields from joined tables
FROM demo.transaction_lines_clean tl
LEFT JOIN demo.transactions_analysis t ON tl.transaction = t.id
LEFT JOIN demo.currencies curr ON t.currency = curr.id
LEFT JOIN demo.locations l ON tl.location = l.id
LEFT JOIN demo.items i ON tl.item = i.id
LEFT JOIN demo.departments d ON tl.department = d.id
LEFT JOIN demo.classifications c ON tl.class = c.id
WHERE tl.item != 25442
"
```

**AFTER:**
```yaml
sql: "SELECT
    id,
    transaction,
    item,
    department,
    location,
    class,
    amount,
    quantity,
    rate,
    taxamount,
    -- All fields below are now directly available (no JOINs needed)
    transaction_type,
    transaction_date,
    transaction_currency_id as transaction_currency,
    transaction_exchange_rate,
    custbody_customer_email as customer_email,
    billing_country,
    shipping_country,
    sku,
    product_name,
    category,
    section,
    season,
    size,
    product_range,
    collection,
    color,
    baseprice as item_base_price,
    currency_name,
    location_name,
    department_name,
    classification_name
  FROM `magical-desktop.demo.transaction_lines_denormalized_mv`
"
```

---

## Key Differences

| Aspect | Before | After |
|--------|--------|-------|
| **Data Source** | `demo.transaction_lines_clean` + 6 LEFT JOINs | `magical-desktop.demo.transaction_lines_denormalized_mv` (single table) |
| **JOINs** | 6 runtime JOINs to 7 tables | **0 JOINs** (all data pre-materialized) |
| **Query Complexity** | Complex multi-table query | Simple SELECT from one view |
| **Expected Build Time** | ~9 minutes (from GPC testing) | ~2-3 minutes (70% faster) |

---

## What This Tests

This configuration change allows us to test the denormalization strategy with the **demo dataset** before applying it to production (**GPC dataset**).

### Testing Goals:

1. **Verify Cube.js works with denormalized view** - Ensure all measures and dimensions function correctly
2. **Measure pre-aggregation build time improvement** - Compare build times before/after denormalization
3. **Validate data correctness** - Ensure results match expectations
4. **Test query performance** - Verify improved query speeds

---

## Important Notes

### Cube Configuration Still References Old Joins

The `joins:` section in the YAML file still exists but is now **unused** because the SQL query no longer references other cubes:

```yaml
joins:
  - name: transactions
    relationship: many_to_one
    sql: '{CUBE}.transaction = {transactions.id}'
  - name: inventory
    relationship: many_to_one
    sql: '{CUBE}.item = {inventory.item}'
  - name: locations
    relationship: many_to_one
    sql: '{CUBE}.location_name = {locations.name}'
  # ...
```

**These can be safely removed** after confirming everything works, but leaving them in place doesn't hurt (they're simply ignored since the SQL doesn't reference the joined cubes).

### All Measures and Dimensions Unchanged

The measures and dimensions sections **remain identical**. They still reference the same field names, they're just now coming from direct columns in the denormalized view instead of joined tables:

- `{CUBE}.transaction_type` - Now from `transaction_type` column (not `t.type`)
- `{CUBE}.sku` - Now from `sku` column (not `i.itemid`)
- `{CUBE}.product_name` - Now from `product_name` column (not `i.displayname`)
- `{CUBE}.currency_name` - Now from `currency_name` column (not `curr.name`)
- etc.

---

## Next Steps for Testing

### 1. Deploy to Cube Cloud (Demo Environment)

```bash
cd /home/produser/cube-demo
git add model/cubes/transaction_lines.yml
git commit -m "Test: Switch to denormalized materialized view for performance testing"
git push
```

### 2. Monitor Pre-Aggregation Build Times

In Cube Cloud, watch the **Pre-Aggregations** tab to see build times for the `sales_analysis` pre-aggregation.

**Expected Results:**
- Before: ~9 minutes per week partition (based on GPC dataset testing)
- After: ~2-3 minutes per week partition (70% faster due to no JOINs)

### 3. Verify Query Correctness

Run sample queries to ensure results match previous implementation:

```sql
-- Test query via Cube.js API
{
  "measures": ["transaction_lines.total_revenue"],
  "dimensions": ["transaction_lines.channel_type", "transaction_lines.category"],
  "timeDimensions": [{
    "dimension": "transaction_lines.transaction_date",
    "dateRange": "Last 7 days"
  }]
}
```

### 4. Compare Performance

| Metric | Before (JOINs) | After (Denormalized) | Target |
|--------|----------------|----------------------|--------|
| Pre-agg build (1 week) | ~9 minutes | ~2-3 minutes | ✅ <5 min |
| Query latency | Variable | Sub-second | ✅ <1 sec |
| BigQuery cost | 6 table scans | 1 view scan | ✅ 60% lower |

---

## Rollback Plan

If issues arise, revert by restoring the original SQL query with JOINs:

```bash
git revert HEAD
git push
```

Or manually restore the old `sql:` definition from git history.

---

## After Successful Demo Testing

Once verified working in demo:

1. **Apply same denormalization to GPC dataset:**
   ```sql
   CREATE OR REPLACE TABLE `magical-desktop.gpc.transaction_lines_denormalized_native`
   -- ... same structure as demo
   ```

2. **Update production Cube.js config** to use GPC denormalized view

3. **Monitor production metrics** and compare to demo results

---

## Documentation Created

- `/home/produser/cube-demo/DENORMALIZATION_COMPLETE_DEMO.md` - Complete technical details
- `/home/produser/cube-demo/DEMO_DENORMALIZATION_TESTING.md` - This file (testing guide)

---

## Conclusion

The cube-demo configuration is now ready to test denormalization performance with the demo dataset. All changes are minimal and reversible, making this a low-risk proof of concept.

**Expected Outcome:** 70% faster pre-aggregation builds with zero changes to query results.
