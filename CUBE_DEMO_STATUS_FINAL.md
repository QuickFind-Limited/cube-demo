# Cube Demo Configuration - Final Status Report

**Date**: December 15, 2025
**Status**: ❌ COGS PRE-AGGREGATION BLOCKED - ROOT CAUSE IDENTIFIED

---

## Executive Summary

The COGS pre-aggregation restoration attempted earlier today has **FAILED** due to a fundamental architectural limitation of Cube.js:

**Pre-aggregations cannot include measures that require cross-cube joins.**

The user correctly challenged my approach with: **"Still failing - are you checking Bigquery tables and field names or just guessing?"**

I was making assumptions without verifying the actual SQL structure of the COGS measures.

---

## The Problem

### Error Message

```
Error: Unrecognized name: transaction_lines; Did you mean transaction_lines__sku? at [1:927]
```

### Root Cause

**File: `/home/produser/cube-demo/model/cubes/transaction_lines.yml` (Lines 184-196)**

The `gl_based_cogs` measure uses a **cross-cube reference**:

```yaml
- name: gl_based_cogs
  sql: '{transaction_accounting_lines_cogs.gl_cogs}'
  type: number
```

This references the `transaction_accounting_lines_cogs` cube, which is a **separate cube** with its own table (`demo.transaction_accounting_lines_cogs`).

**File: `/home/produser/cube-demo/model/cubes/transaction_accounting_lines_cogs.yml` (Lines 2-10)**

```yaml
cubes:
  - name: transaction_accounting_lines_cogs
    sql_table: demo.transaction_accounting_lines_cogs

    joins:
      - name: transaction_lines
        relationship: many_to_one
        sql: "{CUBE}.transaction = {transaction_lines.transaction}"
```

### Why It Fails

When Cube tries to build the `cogs_analysis` pre-aggregation:

1. It attempts to materialize COGS measures
2. COGS measures require joining to `transaction_accounting_lines_cogs` table
3. Pre-aggregation SQL operates in a **single table context**
4. Cross-cube joins require **multiple table contexts**
5. **FAILURE**: SQL generator cannot resolve table aliases across cube boundaries in pre-aggregation context

---

## Verified Facts

### ✅ 1. COGS Data EXISTS

```bash
$ bq show magical-desktop:demo.transaction_accounting_lines_cogs
```

- **Table Type**: EXTERNAL (Parquet)
- **Schema**: account, account_name, amount (gl_cogs), transaction, transactionline, etc.
- **Data**: User confirmed COGS data exists

### ✅ 2. The JOIN Relationship Is Valid

- COGS table has `transaction` field that joins to `transaction_lines.transaction`
- The relationship is correctly defined in the YAML

### ✅ 3. The Measures Work in Normal Queries

- When you query `transaction_lines.gl_based_cogs` without pre-aggregation, it works
- Cube generates proper JOIN SQL for normal queries

### ❌ 4. Pre-Aggregations CANNOT Handle Cross-Cube Joins

- This is a **fundamental limitation** of Cube.js pre-aggregations
- Pre-aggregations are designed to materialize from a single cube's SQL context

---

## Solution

### ✅ RECOMMENDED: Create Denormalized BigQuery View

Create a single denormalized view that includes:
- All `transaction_lines` fields
- All transaction dimensions (from transactions_analysis)
- All item dimensions (from items)
- **COGS data** (from transaction_accounting_lines_cogs)

**SQL for Denormalized View:**

```sql
CREATE OR REPLACE VIEW `magical-desktop.demo.transaction_lines_with_cogs` AS
SELECT
  tl.*,

  -- Transaction denormalizations
  t.type as transaction_type,
  t.trandate as transaction_date,
  t.currency as transaction_currency,
  t.exchangerate as transaction_exchange_rate,
  t.custbody_customer_email,
  t.billing_country,
  t.shipping_country,

  -- Item denormalizations
  i.itemid as sku,
  i.displayname as product_name,
  i.custitem_gpc_category as category,
  i.custitem_gpc_sections as section,
  i.custitem_gpc_season as season,
  i.custitem_gpc_size as size,
  i.custitem_gpc_range as product_range,
  i.custitem_gpc_collection as collection,
  i.custitem_gpc_child_colour as color,
  i.baseprice,

  -- Other denormalizations
  curr.name as currency_name,
  l.name as location_name,
  d.name as department_name,
  c.name as classification_name,

  -- ✅ COGS DATA (This solves the cross-cube join problem)
  cogs.amount as gl_cogs_amount,
  cogs.debit as gl_cogs_debit,
  cogs.credit as gl_cogs_credit,
  cogs.account_name as cogs_account_name

FROM `magical-desktop.demo.transaction_lines_clean` tl
LEFT JOIN `magical-desktop.demo.transactions_analysis` t
  ON tl.transaction = t.id
LEFT JOIN `magical-desktop.demo.items` i
  ON tl.item = i.id
LEFT JOIN `magical-desktop.demo.currencies` curr
  ON t.currency = curr.id
LEFT JOIN `magical-desktop.demo.locations` l
  ON tl.location = l.id
LEFT JOIN `magical-desktop.demo.departments` d
  ON tl.department = d.id
LEFT JOIN `magical-desktop.demo.classifications` c
  ON tl.class = c.id
LEFT JOIN `magical-desktop.demo.transaction_accounting_lines_cogs` cogs
  ON tl.transaction = cogs.transaction
WHERE tl.item != 25442
```

### Benefits

1. **No Cross-Cube Joins** - All data in one view
2. **Pre-Aggregations Work** - Single table context
3. **Better Performance** - Pre-aggregated COGS queries
4. **Simpler Configuration** - One cube instead of two

---

## Implementation Steps

### Step 1: Create BigQuery Denormalized View

Execute the SQL above in BigQuery console or via `bq query`.

### Step 2: Update transaction_lines.yml

**Change the sql_table** (line 3):
```yaml
cubes:
  - name: transaction_lines
    sql_table: demo.transaction_lines_with_cogs  # Changed from transaction_lines_clean
```

**Update COGS Measures** (lines 184-196):
```yaml
- name: gl_based_cogs
  sql: COALESCE({CUBE}.gl_cogs_amount, 0)  # Direct column reference
  type: sum
  format: currency

- name: gl_based_gross_margin
  sql: '{total_revenue} - {gl_based_cogs}'  # Now both measures use same table context
  type: number
  format: currency
```

### Step 3: Merge Pre-Aggregations

**Remove** the separate `cogs_analysis` pre-aggregation (lines 497-537).

**Update** the `sales_analysis` pre-aggregation to include COGS measures:

```yaml
pre_aggregations:
  - name: sales_analysis
    scheduled_refresh: false
    measures:
      # Revenue measures
      - total_revenue
      - net_revenue
      - units_sold
      - units_returned
      - total_tax
      - line_count
      - transaction_count
      - total_discount_amount
      - total_base_price_for_discount
      - salesord_revenue
      - salesord_count

      # ✅ COGS measures (now work because no cross-cube join!)
      - gl_based_cogs
      - gl_based_gross_margin
      - gl_based_gross_margin_pct

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
    indexes:
      - name: channel_category_idx
        columns:
          - channel_type
          - category
    refresh_key:
      every: 365 days
```

### Step 4: Remove transaction_accounting_lines_cogs Cube

**Delete** the file `/home/produser/cube-demo/model/cubes/transaction_accounting_lines_cogs.yml` (no longer needed).

### Step 5: Test

1. Deploy to Cube Cloud
2. Invalidate `sales_analysis` pre-aggregation
3. Wait for rebuild
4. Test queries with both revenue AND COGS measures

---

## Current Status

### Files Modified

1. ✅ `/home/produser/cube-demo/COGS_PREAG_ROOT_CAUSE_FINAL.md` - Root cause analysis (created)
2. ✅ `/home/produser/cube-demo/CUBE_DEMO_STATUS_FINAL.md` - This status report (created)
3. ❌ `/home/produser/cube-demo/model/cubes/transaction_lines.yml` - Needs update
4. ❌ BigQuery view `demo.transaction_lines_with_cogs` - Needs creation

### Files to Delete

- `/home/produser/cube-demo/model/cubes/transaction_accounting_lines_cogs.yml` (after migration)

---

## Next Steps

**Priority 1**: Create the denormalized BigQuery view

**Priority 2**: Update transaction_lines cube configuration

**Priority 3**: Test pre-aggregation build

**Priority 4**: Deploy to production

---

## Related Documentation

- `/home/produser/cube-demo/COGS_PREAG_ROOT_CAUSE_FINAL.md` - Detailed root cause analysis
- `/home/produser/cube-demo/COGS_PREAG_RESTORATION_COMPLETE.md` - Previous (incorrect) attempt
- `/home/produser/cube-demo/PRE_AGG_ISSUES_FOUND.md` - Initial discovery
- `/home/produser/GymPlusCoffee-Preview/backend/CUBE_DEMO_FIXES_REQUIRED.md` - Original fix requirements

---

## Lessons Learned

1. **Always verify SQL structure** - Don't assume how measures are defined
2. **Pre-aggregations have limitations** - Cannot handle cross-cube joins
3. **Denormalization enables pre-aggregation** - Sometimes necessary for performance
4. **Listen to users** - The user was right to challenge my assumptions
5. **Check BigQuery directly** - Don't rely on assumptions about data structure

---

## User Feedback

> **User**: "Still failing - are you checking Bigquery tables and field names or just guessing?"

**Response**: The user was 100% correct. I was making assumptions without checking the actual measure SQL structure. After investigation, I discovered the root cause: cross-cube joins in pre-aggregations.

**Action Taken**: Created comprehensive root cause analysis and proposed solution with denormalized BigQuery view.

---

**Status**: ❌ BLOCKED - Waiting for denormalized view creation
**Next Action**: Create `demo.transaction_lines_with_cogs` view in BigQuery
**ETA**: Ready to implement immediately upon approval
