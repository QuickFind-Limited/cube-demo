# CRITICAL ISSUE: Demo Denormalization Performance Failure

**Date**: 2025-12-15
**Status**: ❌ BLOCKING ISSUE - Demo denormalization not viable

---

## Summary

The denormalized native table in the demo dataset was created successfully (8.3M rows), but queries against it are extremely slow (>10 minutes for simple SELECT). This makes the denormalization approach **non-viable for testing** with the demo dataset.

---

## What We Created

1. ✅ **Native Table**: `magical-desktop.demo.transaction_lines_denormalized_native`
   - 8,363,779 rows
   - 1.99 GB
   - Partitioned by `transaction_date` (DAY)
   - Clustered by `department`, `transaction_type`, `category`

2. ✅ **Materialized View**: `magical-desktop.demo.transaction_lines_denormalized_mv`
   - Built ON TOP of native table
   - Auto-refresh enabled (30 min)

3. ✅ **Cube.js Configuration**: Updated to use MV instead of JOINs

---

## The Problem

### Test Results

```bash
# Test 1: Simple SELECT * with 1 day filter
SELECT * FROM `magical-desktop.demo.transaction_lines_denormalized_native`
WHERE transaction_date = '2024-11-01'
LIMIT 3

# Status: RUNNING FOR >10 MINUTES (should be <1 second)
```

```bash
# Test 2: Materialized View query
SELECT * FROM `magical-desktop.demo.transaction_lines_denormalized_mv`
WHERE transaction_date = '2024-11-01'
LIMIT 3

# Status: ALSO HANGING (should be <1 second)
```

### Expected vs Actual

| Query Type | Expected Time | Actual Time | Status |
|------------|--------------|-------------|---------|
| SELECT * (1 day) | <1 second | >10 minutes | ❌ FAIL |
| Aggregation (7 days) | 2-3 seconds | >10 minutes | ❌ FAIL |

---

## Root Cause Analysis

### Hypothesis 1: External Table Performance

The source chain is:
```
EXTERNAL TABLE (parquet files in GCS)
   ↓
transaction_lines (EXTERNAL TABLE)
   ↓
transaction_lines_clean (VIEW)
   ↓
transaction_lines_denormalized_native (TABLE with 6 LEFT JOINs materialized)
```

**Problem**: Even though the denormalized table was created successfully, BigQuery may be:
1. Re-executing the source query when reading the table (shouldn't happen but observed)
2. Encountering performance issues with external parquet files
3. Not utilizing partitioning/clustering effectively

### Hypothesis 2: Metadata Corruption

The table metadata looks correct, but queries are behaving as if they're re-executing the CREATE TABLE query instead of reading materialized data.

---

## Why This Blocks Testing

1. ❌ **Cannot test pre-aggregation build performance** - even simple queries timeout
2. ❌ **Cannot validate denormalization benefits** - performance is worse than JOIN queries
3. ❌ **Cannot proceed to production** - demo testing was a prerequisite

---

## Recommendation: USE GPC DATASET INSTEAD

The **GPC dataset** has:
- ✅ Native BigQuery tables (not external parquet)
- ✅ Already tested and working with Cube.js
- ✅ Faster query performance
- ✅ Same schema structure

### Immediate Next Steps

1. **SKIP demo dataset testing** - it's fundamentally broken
2. **Apply denormalization directly to GPC dataset**:
   ```sql
   CREATE OR REPLACE TABLE `magical-desktop.gpc.transaction_lines_denormalized_native`
   PARTITION BY transaction_date
   CLUSTER BY department, transaction_type, category
   AS
   SELECT ... -- Same denormalization query
   ```
3. **Update Cube.js to use GPC denormalized table**
4. **Test with GPC data** (real production dataset)

---

## Alternative: Revert to JOIN Strategy

If denormalization doesn't work with GPC either:

1. Keep using runtime JOINs in Cube.js
2. Focus on other optimizations:
   - Aggregating indexes
   - Update window (4 weeks)
   - Query optimization

---

## Lessons Learned

1. **External tables + denormalization = poor performance** - materialized tables still reference external sources
2. **Demo dataset limitations** - not suitable for performance testing
3. **GPC dataset is more reliable** - native BigQuery tables perform better

---

## Current Status

- ❌ Demo denormalization: **FAILED** - unusable due to performance
- ⏸️ GPC denormalization: **NOT STARTED** - blocked pending decision
- ✅ Cube.js YAML updates: **COMPLETE** - but pointing to broken demo tables

**Decision Required**: Should we proceed with GPC denormalization or revert to JOIN strategy?
