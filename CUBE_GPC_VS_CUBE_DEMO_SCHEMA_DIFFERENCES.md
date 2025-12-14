# Cube-GPC vs Cube-Demo Schema Differences

**Date**: December 14, 2025
**Purpose**: Document critical schema differences between production (cube-gpc) and development (cube-demo) environments

---

## Executive Summary

The `cube-gpc` and `cube-demo` repositories contain identical cube definitions but query **different BigQuery datasets with OPPOSITE schema characteristics**. This document explains why fixes applied to one repository would **BREAK** the other.

**Critical Finding**: DO NOT blindly sync fixes between repositories - schema differences require different cube configurations.

---

## Repository Overview

| Repository | Dataset | Environment | Status | Cube Count |
|------------|---------|-------------|--------|------------|
| **cube-demo** | `magical-desktop:demo.*` | Development/Testing | ✅ Tested, 39 bugs fixed | 27 |
| **cube-gpc** | `magical-desktop:gpc.*` | Production | ✅ Production-ready | 27 |

---

## Schema Differences: The Critical Issue

### Date Fields

| Aspect | demo Dataset | gpc Dataset |
|--------|-------------|-------------|
| **Type** | STRING | DATE |
| **Format** | 'DD/MM/YYYY' (e.g., '25/01/2024') | Native DATE type |
| **Parsing Required** | ✅ YES - `PARSE_DATE('%d/%m/%Y', field)` | ❌ NO - Use directly |
| **TIMESTAMP Conversion** | `TIMESTAMP(PARSE_DATE('%d/%m/%Y', field))` | `TIMESTAMP(field)` |
| **Example Field** | `transactions.trandate` | `transactions.trandate` |

**Impact**: All date parsing logic must be OPPOSITE between the two datasets.

### Boolean Fields

| Aspect | demo Dataset | gpc Dataset |
|--------|-------------|-------------|
| **Type** | BOOLEAN | STRING |
| **Values** | TRUE/FALSE | 'T'/'F' |
| **Comparisons** | `WHERE posting = TRUE` | `WHERE posting = 'T'` |
| **Example Fields** | `posting`, `voided`, `mainline`, `taxline`, `iscogs` | `posting`, `voided`, `mainline`, `taxline`, `iscogs` |

**Impact**: All boolean comparisons must use OPPOSITE syntax between the two datasets.

---

## Example: Why Fixes Don't Translate

### Example 1: Date Parsing in transactions.yml

**cube-demo (CORRECT for demo dataset)**:
```yaml
sql: >
  SELECT
    TIMESTAMP(PARSE_DATE('%d/%m/%Y', trandate)) as trandate
  FROM demo.transactions
```

**cube-gpc (CORRECT for gpc dataset)**:
```yaml
sql: >
  SELECT
    TIMESTAMP(trandate) as trandate  # trandate is already DATE type
  FROM gpc.transactions
```

**If you apply cube-demo fix to cube-gpc**: ❌ **BREAKS**
```yaml
# This would FAIL in gpc:
TIMESTAMP(PARSE_DATE('%d/%m/%Y', trandate))
# Error: PARSE_DATE expects STRING, gets DATE
```

### Example 2: Boolean Comparisons in transaction_lines.yml

**cube-demo (CORRECT for demo dataset)**:
```yaml
WHERE tl.mainline = FALSE
  AND tl.taxline = FALSE
  AND tl.iscogs = FALSE
```

**cube-gpc (CORRECT for gpc dataset)**:
```yaml
WHERE tl.mainline = 'F'
  AND tl.taxline = 'F'
  AND tl.iscogs = 'F'
```

**If you apply cube-demo fix to cube-gpc**: ❌ **BREAKS**
```yaml
# This would FAIL in gpc:
WHERE tl.mainline = FALSE
# Error: No matching signature for operator = for argument types: STRING, BOOL
```

---

## Bug Pattern Comparison

### Bugs in cube-demo (FIXED)

| Pattern | Count | Fix Applied |
|---------|-------|-------------|
| DATE parsing missing | 7 | Added `PARSE_DATE('%d/%m/%Y', ...)` |
| BOOLEAN type mismatch | 24 | Changed 'T'/'F' to TRUE/FALSE |
| Double PARSE_DATE | 5 | Removed redundant PARSE_DATE |
| Cross-cube date reference | 1 | Fixed reference to converted TIMESTAMP |
| Incomplete DATE parsing | 1 | Changed CAST to PARSE_DATE |
| **TOTAL** | **39** | **All fixed for demo dataset** |

### Expected Behavior in cube-gpc (PRODUCTION)

| Pattern | Expected Behavior | Status |
|---------|-------------------|--------|
| DATE parsing | NOT needed (native DATE) | ✅ Correct as-is |
| BOOLEAN comparisons | STRING 'T'/'F' is CORRECT | ✅ Correct as-is |
| Double PARSE_DATE | Would occur if demo fixes applied | ⚠️ Avoid |
| Cross-cube references | May need different handling | 🔍 Review if issues arise |
| DATE calculations | Use DATE type directly | ✅ Correct as-is |

---

## Schema Verification Commands

### Check demo Dataset Schema

```bash
# Check if date field is STRING or DATE
bq show --schema magical-desktop:demo.transactions | grep trandate
# Output: {"name":"trandate","type":"STRING",...}

# Check if boolean field is BOOLEAN or STRING
bq show --schema magical-desktop:demo.transactions | grep posting
# Output: {"name":"posting","type":"BOOLEAN",...}
```

### Check gpc Dataset Schema

```bash
# Check if date field is STRING or DATE
bq show --schema magical-desktop:gpc.transactions | grep trandate
# Output: {"name":"trandate","type":"DATE",...}

# Check if boolean field is BOOLEAN or STRING
bq show --schema magical-desktop:gpc.transactions | grep posting
# Output: {"name":"posting","type":"STRING",...}
```

---

## Testing Approach by Environment

### cube-demo (Development Environment)

**Status**: ✅ Comprehensive testing complete
- All 27 cubes tested
- 39 bugs found and fixed
- Schema-specific fixes applied for STRING dates and BOOLEAN booleans
- Safe to deploy to any environment using `demo` dataset

**Testing Methodology**: See `CUBE_TESTING_METHODOLOGY.md`

### cube-gpc (Production Environment)

**Status**: ⚠️ SKIP comprehensive testing - different schema
- Production cubes already deployed and functional
- Schema differences mean demo bugs DON'T exist in gpc
- Applying demo fixes would BREAK production

**Recommended Approach**:
1. ✅ Monitor production for actual errors
2. ❌ Do NOT apply demo fixes blindly
3. ✅ Only fix reported production issues
4. ✅ Test fixes against gpc schema specifically

---

## Infrastructure Differences

### demo Dataset

**Type**: Standard BigQuery tables
**Source**: Parquet files imported with specific schema
**Access**: Direct table queries work
**Date Format**: STRING in DD/MM/YYYY format
**Boolean Format**: Native BOOLEAN type

### gpc Dataset

**Type**: External tables over Parquet files
**Source**: Parquet files in GCS (`gs://gym-plus-coffee-bucket-dev/parquet/`)
**Access**: Some tables have Parquet schema mismatches (e.g., `transactions.postingperiod`)
**Workaround**: Cubes query VIEWS (`transactions_analysis`), not raw tables
**Date Format**: Native DATE type
**Boolean Format**: STRING 'T'/'F'

### Parquet Schema Issues (gpc only)

**Issue**: Base table `gpc.transactions` has Parquet type mismatch
```
Error: Parquet column 'postingperiod' has type BYTE_ARRAY which does not match
the target cpp_type INT64
```

**Workaround**: Production cubes query `gpc.transactions_analysis` VIEW, not raw table
**Impact**: None - views work correctly, production is functional

---

## Deployment Guidelines

### When Deploying to demo Environment

1. ✅ Use cube-demo repository
2. ✅ Apply all 39 bug fixes
3. ✅ Verify cubes use `demo.*` dataset references
4. ✅ Test with STRING dates (DD/MM/YYYY format)
5. ✅ Test with BOOLEAN TRUE/FALSE

### When Deploying to gpc Environment

1. ✅ Use cube-gpc repository AS-IS
2. ❌ DO NOT apply cube-demo bug fixes
3. ✅ Verify cubes use `gpc.*` dataset references
4. ✅ Test with native DATE types
5. ✅ Test with STRING 'T'/'F' booleans
6. ✅ Ensure queries use VIEWS for problematic tables

---

## Migration Strategy (If Needed)

### Scenario 1: Sync cube-demo Fixes to cube-gpc

**NOT RECOMMENDED** - Schema differences are too significant

If absolutely required:
1. Review EVERY date field reference
2. Remove PARSE_DATE where not needed
3. Review EVERY boolean comparison
4. Change TRUE/FALSE back to 'T'/'F'
5. Test extensively against gpc dataset

### Scenario 2: Standardize Schemas

**Option A**: Convert gpc to match demo schema
- Re-import Parquet files with STRING dates and BOOLEAN booleans
- Apply cube-demo fixes
- Benefit: Single set of cube definitions

**Option B**: Convert demo to match gpc schema
- Re-import with DATE types and STRING booleans
- Remove all PARSE_DATE from cube-demo
- Change all TRUE/FALSE to 'T'/'F'
- Benefit: Matches production

**Recommended**: Option B (match production schema) for consistency

---

## Quick Reference

### Date Field Handling

```yaml
# demo dataset (STRING dates):
TIMESTAMP(PARSE_DATE('%d/%m/%Y', trandate)) as trandate
DATE_DIFF(PARSE_DATE('%d/%m/%Y', date1), PARSE_DATE('%d/%m/%Y', date2), DAY)

# gpc dataset (DATE dates):
TIMESTAMP(trandate) as trandate
DATE_DIFF(date1, date2, DAY)
```

### Boolean Field Handling

```yaml
# demo dataset (BOOLEAN):
WHERE posting = TRUE
  AND voided = FALSE
  AND mainline = FALSE

# gpc dataset (STRING):
WHERE posting = 'T'
  AND voided = 'F'
  AND mainline = 'F'
```

---

## Conclusion

**Key Takeaways**:

1. ✅ **cube-demo**: Tested and production-ready for environments using `demo` dataset
2. ✅ **cube-gpc**: Already production-ready for `gpc` dataset environment
3. ❌ **DO NOT** cross-apply fixes between repositories without schema verification
4. ✅ **ALWAYS** verify schema types before writing cube SQL
5. ⚠️ **REMEMBER**: What's a bug in one environment is CORRECT in the other

**Recommendation**: Keep repositories separate with environment-specific configurations. Only sync cube logic/structure changes, not schema-specific fixes.

---

**Document Version**: 1.0
**Last Updated**: December 14, 2025
**Author**: QuickFind Limited (via Claude Code comprehensive analysis)
**Related Files**:
- `/home/produser/cube-demo/CUBE_TESTING_METHODOLOGY.md`
- `/home/produser/cube-demo/COMPREHENSIVE_CUBE_TESTING_SUMMARY.md`
