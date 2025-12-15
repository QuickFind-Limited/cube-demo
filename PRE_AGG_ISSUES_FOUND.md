# Pre-Aggregation Issues Found

**Date**: December 15, 2025

## Critical Issues with New Separate Pre-Aggregations

### Issue #1: Missing Measures

**Original `sales_analysis` pre-aggregation** had 17 measures:
1. total_revenue ✅
2. net_revenue ✅
3. units_sold ✅
4. units_returned ✅
5. **gross_margin** ❌ MISSING
6. **total_cost** ❌ MISSING
7. **gl_based_cogs** ❌ MISSING
8. **gl_based_gross_margin** ❌ MISSING
9. total_tax ✅
10. line_count ✅
11. **min_transaction_date** ❌ MISSING
12. **max_transaction_date** ❌ MISSING
13. total_discount_amount ✅
14. total_base_price_for_discount ❌ MISSING (needed for discount % calculation)
15. transaction_count ✅
16. salesord_revenue ✅
17. salesord_count ✅

**New `revenue_analysis` pre-aggregation** has only 10 measures (missing 7)
**New `cogs_analysis` pre-aggregation** has 5 measures (correctly split)

### Issue #2: Missing Configuration

**Original pre-aggregation had:**
```yaml
indexes:
- name: channel_category_idx
  columns:
  - channel_type
  - category
refresh_key:
  every: 365 days
```

**New pre-aggregations are missing:**
- ❌ indexes
- ❌ refresh_key

### Issue #3: Partitioning May Not Work

The original pre-aggregation was working and building partitions. The new ones won't fetch partitions according to your report.

## Required Fixes

### Fix revenue_analysis pre-aggregation:
```yaml
- name: revenue_analysis
  scheduled_refresh: false
  measures:
    - total_revenue
    - net_revenue
    - units_sold
    - units_returned
    - total_tax
    - line_count
    - transaction_count
    - total_discount_amount
    - total_base_price_for_discount  # ADD THIS
    - salesord_revenue
    - salesord_count
    - min_transaction_date  # ADD THIS
    - max_transaction_date  # ADD THIS
  dimensions:
    - channel_type
    - category
    - section
    - season
    - product_range
    - collection
    - department_name
    - billing_country
    - shipping_country
    - classification_name
    - transaction_currency
    - currency_name
    - transaction_type
    - pricing_type
    - sku
    - product_name
    - size
    - color
    - location_name
    - customer_type
  time_dimension: transaction_date
  granularity: day
  partition_granularity: week
  build_range_end:
    sql: SELECT '2025-10-31'
  indexes:  # ADD THIS
    - name: channel_category_idx
      columns:
        - channel_type
        - category
  refresh_key:  # ADD THIS
    every: 365 days
```

### Fix cogs_analysis pre-aggregation:
```yaml
- name: cogs_analysis
  scheduled_refresh: false
  measures:
    - gl_based_cogs
    - gl_based_gross_margin
    - gl_based_gross_margin_pct
    - total_cost
    - gross_margin
  dimensions:
    - channel_type
    - category
    - section
    - season
    - product_range
    - collection
    - department_name
    - billing_country
    - shipping_country
    - classification_name
    - transaction_currency
    - currency_name
    - transaction_type
    - pricing_type
    - sku
    - product_name
    - size
    - color
    - location_name
    - customer_type
  time_dimension: transaction_date
  granularity: day
  partition_granularity: week
  build_range_end:
    sql: SELECT '2025-10-31'
  indexes:  # ADD THIS
    - name: channel_category_idx
      columns:
        - channel_type
        - category
  refresh_key:  # ADD THIS
    every: 365 days
```

## Summary

The new pre-aggregations are missing:
- **3 critical measures** in revenue_analysis (min_transaction_date, max_transaction_date, total_base_price_for_discount)
- **indexes** configuration (important for query performance)
- **refresh_key** configuration (controls when pre-agg rebuilds)

These omissions explain why partitions won't fetch - the configuration is incomplete.
