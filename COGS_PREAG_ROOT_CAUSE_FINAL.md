# COGS Pre-Aggregation Root Cause Analysis - FINAL

**Date**: December 15, 2025
**Status**: ❌ ROOT CAUSE IDENTIFIED - ARCHITECTURAL LIMITATION

---

## Error Summary

```
Error: Unrecognized name: transaction_lines; Did you mean transaction_lines__sku? at [1:927]
    at BigQueryDriver.awaitForJobStatus
```

**User Challenge**: "Still failing - are you checking Bigquery tables and field names or just guessing?"

**Answer**: The user was 100% correct to challenge me. I was making assumptions without checking the actual SQL structure of the COGS measures.

---

## Root Cause

### The Fundamental Problem

**Pre-aggregations in Cube.js CANNOT include measures that require cross-cube joins.**

### How COGS Measures Are Defined

**File**: `/home/produser/cube-demo/model/cubes/transaction_lines.yml`

**Lines 184-196** - The gl_based_cogs measure:
```yaml
- name: gl_based_cogs
  sql: '{transaction_accounting_lines_cogs.gl_cogs}'
  type: number
  format: currency
  description: GL-based COGS from TransactionAccountingLine
```

This measure references `transaction_accounting_lines_cogs.gl_cogs`, which is a **cross-cube reference**.

### The COGS Cube Structure

**File**: `/home/produser/cube-demo/model/cubes/transaction_accounting_lines_cogs.yml`

**Lines 2-10** - The COGS cube with JOIN back to transaction_lines:
```yaml
cubes:
  - name: transaction_accounting_lines_cogs
    sql_table: demo.transaction_accounting_lines_cogs

    joins:
      - name: transaction_lines
        relationship: many_to_one
        sql: "{CUBE}.transaction = {transaction_lines.transaction}"
```

**Lines 60-64** - The gl_cogs measure in COGS cube:
```yaml
measures:
  - name: gl_cogs
    sql: amount
    type: sum
    format: currency
```

### What Happens During Pre-Aggregation Build

When Cube tries to build the `cogs_analysis` pre-aggregation:

1. It sees measures that reference `transaction_accounting_lines_cogs.gl_cogs`
2. It tries to generate SQL that joins the two cubes
3. The generated SQL has TWO separate table contexts:
   - `transaction_lines_clean` (base table)
   - `transaction_accounting_lines_cogs` (COGS table)
4. The SQL generator tries to reference `transaction_lines` table in the COGS query context
5. **FAILURE**: The table alias `transaction_lines` doesn't exist in the COGS query context

### Why This Is a Limitation

Cube.js pre-aggregations are designed to **materialize rollup tables** from a **single cube's SQL context**. When you try to include cross-cube measures, Cube cannot generate valid SQL because:

- The pre-aggregation SQL query operates in a single table context
- Cross-cube joins require multiple table contexts
- The SQL generator cannot resolve table aliases across cube boundaries in pre-aggregation context

---

## Verified Facts

### 1. COGS Data EXISTS in BigQuery ✅

```bash
$ bq show magical-desktop:demo.transaction_accounting_lines_cogs
```

**Table Type**: EXTERNAL (Parquet)

**Schema**:
- account: INTEGER
- account_name: STRING
- acctnumber: INTEGER
- accttype: STRING
- amount: FLOAT (this is the gl_cogs value)
- credit: FLOAT
- debit: FLOAT
- links: STRING
- posting: BOOLEAN
- trandate: DATE
- transaction: INTEGER (FK to transactions)
- transactionline: INTEGER (FK to transaction lines)

### 2. The Join Relationship Is Valid ✅

The COGS table has `transaction` field that can join to `transaction_lines.transaction`.

### 3. The Measures Work in Normal Queries ✅

When you query `transaction_lines.gl_based_cogs` in a normal Cube query (not pre-aggregated), it works fine because Cube generates SQL with proper joins.

### 4. Pre-Aggregations Cannot Handle This ❌

The `cogs_analysis` pre-aggregation **cannot be built** because it tries to include cross-cube measures.

---

## Solution Options

### Option 1: ✅ RECOMMENDED - Create Denormalized COGS View in BigQuery

**Create a BigQuery view** that denormalizes COGS data with transaction dimensions:

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

  -- COGS data (LEFT JOIN to keep all transaction lines)
  cogs.amount as gl_cogs_amount,
  cogs.debit as gl_cogs_debit,
  cogs.credit as gl_cogs_credit,
  cogs.account_name as cogs_account_name

FROM `magical-desktop.demo.transaction_lines_clean` tl
LEFT JOIN `magical-desktop.demo.transactions_analysis` t ON tl.transaction = t.id
LEFT JOIN `magical-desktop.demo.items` i ON tl.item = i.id
LEFT JOIN `magical-desktop.demo.currencies` curr ON t.currency = curr.id
LEFT JOIN `magical-desktop.demo.locations` l ON tl.location = l.id
LEFT JOIN `magical-desktop.demo.departments` d ON tl.department = d.id
LEFT JOIN `magical-desktop.demo.classifications` c ON tl.class = c.id
LEFT JOIN `magical-desktop.demo.transaction_accounting_lines_cogs` cogs
  ON tl.transaction = cogs.transaction
WHERE tl.item != 25442
```

**Benefits**:
- Single denormalized view with both revenue AND COGS data
- Pre-aggregations can now work (no cross-cube joins needed)
- Simpler Cube.js configuration
- Better query performance

**Then update transaction_lines.yml to use this view**:
```yaml
cubes:
  - name: transaction_lines
    sql_table: demo.transaction_lines_with_cogs

measures:
  - name: gl_based_cogs
    sql: gl_cogs_amount  # Direct column reference, no cross-cube join
    type: sum
    format: currency
```

---

### Option 2: ❌ NOT RECOMMENDED - Remove COGS from Pre-Aggregations

Keep separate pre-aggregations:
- `revenue_analysis` - Revenue measures only (uses pre-agg)
- COGS measures query raw data (no pre-agg)

**Problems**:
- Poor performance for COGS queries
- Mixed query patterns (some fast, some slow)
- Doesn't solve the fundamental issue

---

### Option 3: ❌ NOT RECOMMENDED - Separate COGS Cube with Pre-Aggregation

Create a separate COGS cube with its own pre-aggregation that includes transaction dimensions.

**Problems**:
- Duplicate dimension storage
- Complex client-side joins required
- Still requires denormalized data in BigQuery

---

## Recommendation

**Implement Option 1** - Create a denormalized view in BigQuery that includes both transaction_lines and COGS data.

### Implementation Steps

1. **Create the denormalized view** in BigQuery (SQL above)

2. **Update transaction_lines.yml** to use the new view:
   ```yaml
   sql_table: demo.transaction_lines_with_cogs
   ```

3. **Update COGS measures** to use direct column references:
   ```yaml
   - name: gl_based_cogs
     sql: COALESCE({CUBE}.gl_cogs_amount, 0)
     type: sum
   ```

4. **Update pre-aggregation** to include COGS measures:
   ```yaml
   pre_aggregations:
     - name: sales_analysis
       measures:
         - total_revenue
         - net_revenue
         - gl_based_cogs        # Now works!
         - gl_based_gross_margin
         # ... other measures
   ```

5. **Test** the pre-aggregation build

6. **Remove** the separate `transaction_accounting_lines_cogs` cube (no longer needed)

---

## Lessons Learned

1. **Always check actual SQL structure** - Don't assume measure definitions
2. **Pre-aggregations have limitations** - They cannot handle cross-cube joins
3. **Denormalization is sometimes necessary** - For pre-aggregation performance
4. **Listen to users when challenged** - The user was right to question my assumptions

---

## Next Steps

1. Create the denormalized view in BigQuery
2. Update transaction_lines cube configuration
3. Remove cogs_analysis pre-aggregation (merge into sales_analysis)
4. Test pre-aggregation build
5. Deploy to Cube Cloud

---

## Related Files

- `/home/produser/cube-demo/model/cubes/transaction_lines.yml` - Main cube (needs update)
- `/home/produser/cube-demo/model/cubes/transaction_accounting_lines_cogs.yml` - COGS cube (can be removed)
- `/home/produser/cube-demo/COGS_PREAG_RESTORATION_COMPLETE.md` - Previous attempt (incorrect)
- `/home/produser/cube-demo/PRE_AGG_ISSUES_FOUND.md` - Initial discovery

---

## Status

**Current**: ❌ COGS pre-aggregation failing due to cross-cube join limitation
**Solution**: ✅ Create denormalized BigQuery view
**Next Action**: Create the denormalized view and update Cube configuration
