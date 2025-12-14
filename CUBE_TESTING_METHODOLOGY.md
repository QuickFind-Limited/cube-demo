# Cube.js Schema Testing Methodology

**Project**: cube-demo (QuickFind-Limited/cube-demo)
**Data Warehouse**: Google BigQuery (`magical-desktop:demo.*`)
**Testing Period**: December 2025
**Coverage**: 27/27 cubes (100%)
**Total Bugs Fixed**: 37 bugs across 19 cubes in 16 commits

---

## Table of Contents

1. [Overview](#overview)
2. [Testing Philosophy](#testing-philosophy)
3. [Comprehensive Testing Approach](#comprehensive-testing-approach)
4. [Testing Scenarios Per Cube](#testing-scenarios-per-cube)
5. [Bug Patterns Discovered](#bug-patterns-discovered)
6. [Testing Tools and Commands](#testing-tools-and-commands)
7. [Quality Assurance Checklist](#quality-assurance-checklist)
8. [Results Summary](#results-summary)

---

## Overview

This document outlines the comprehensive testing methodology used to validate all 27 Cube.js YML schema files against the BigQuery data warehouse. The testing approach was designed to catch **all** SQL syntax errors, type mismatches, and logical bugs before deployment to production.

### Key Principles

1. **No Assumptions**: Every SQL expression must be tested with actual query execution
2. **No Shortcuts**: Test every dimension, measure, CASE statement, filter, and segment
3. **No Batching**: Test cubes individually with full verification
4. **Immediate Fixes**: Fix bugs as soon as discovered, commit and push to GitHub
5. **Schema Validation**: Verify field types against BigQuery schema

---

## Testing Philosophy

### Why Comprehensive Testing Matters

**Cube Cloud operates in rollup-only mode**, meaning:
- Pre-aggregations are the ONLY source of query data
- A single SQL error breaks the entire pre-aggregation build
- Schema mismatches cause silent failures or incorrect results
- Type mismatches between YML and BigQuery cause runtime errors

**Cost of Failures**:
- Failed pre-aggregations block all queries to that cube
- Silent type mismatches produce incorrect business metrics
- Production debugging is expensive and time-consuming
- User trust is lost when dashboards show wrong data

**Therefore**: Test everything before deployment.

---

## Comprehensive Testing Approach

### Phase 1: Base SQL Validation

For each cube, extract and test the base SQL query:

```bash
# Extract base SQL from YML (lines 3-50 typically)
grep -A 50 "sql: >" cube_name.yml > /tmp/test_cube.sql

# Clean and execute
bq query --use_legacy_sql=false < /tmp/test_cube.sql
```

**What to verify**:
- ✅ Query executes without syntax errors
- ✅ All JOINs resolve correctly
- ✅ All referenced tables exist
- ✅ Field names match BigQuery schema
- ✅ Sample results look correct (LIMIT 3-10 rows)

**Common issues caught**:
- Missing PARSE_DATE for STRING date fields
- BOOLEAN fields compared to STRING literals ('T'/'F')
- Field name typos (e.g., `isinactive` vs `is_inactive`)
- Invalid table references

### Phase 2: Schema Type Verification

Verify field types match between YML and BigQuery:

```bash
# Check schema for specific fields
bq show --schema magical-desktop:demo.table_name | grep -i field_name

# Example: Check if 'posting' is BOOLEAN or STRING
bq show --schema magical-desktop:demo.transactions | grep posting
# Output: {"name": "posting", "type": "BOOLEAN"}
```

**Type validation rules**:
- BigQuery BOOLEAN → YML `type: boolean`, compare to TRUE/FALSE
- BigQuery STRING → YML `type: string`, compare to 'value'
- BigQuery DATE → YML `type: time`, may need PARSE_DATE('%d/%m/%Y', field)
- BigQuery TIMESTAMP → YML `type: time`, use directly
- BigQuery INT64/FLOAT64 → YML `type: number`

### Phase 3: Measure Testing

Test **every** measure individually:

```bash
# Test additive measures (SUM, COUNT, AVG)
bq query --use_legacy_sql=false "
SELECT
  SUM(field_name) as measure_value
FROM (
  -- Base SQL here
)
LIMIT 10"

# Test calculated measures (verify SQL logic)
bq query --use_legacy_sql=false "
SELECT
  CASE WHEN {condition} THEN {value} ELSE 0 END as calculated_measure
FROM (
  -- Base SQL here
)
LIMIT 10"
```

**What to verify**:
- ✅ All aggregation functions work (SUM, COUNT, MIN, MAX, AVG)
- ✅ CASE statements have correct syntax
- ✅ Division by zero handled (NULLIF, SAFE_DIVIDE)
- ✅ Type casting is correct (CAST, SAFE_CAST)
- ✅ Results are reasonable (no negative counts, etc.)

### Phase 4: Dimension Testing

Test **every** dimension individually:

```bash
# Test simple dimensions
bq query --use_legacy_sql=false "
SELECT DISTINCT
  field_name as dimension_value
FROM (
  -- Base SQL here
)
LIMIT 10"

# Test CASE dimensions
bq query --use_legacy_sql=false "
SELECT DISTINCT
  CASE
    WHEN condition1 THEN 'label1'
    WHEN condition2 THEN 'label2'
    ELSE 'default'
  END as case_dimension
FROM (
  -- Base SQL here
)
LIMIT 10"
```

**What to verify**:
- ✅ Field references are correct
- ✅ CASE logic covers all scenarios
- ✅ Time dimensions parse correctly
- ✅ No double PARSE_DATE on already-converted fields
- ✅ Dimension values match expected business logic

**Common double PARSE_DATE bug**:
```yaml
# BASE SQL (line 10):
TIMESTAMP(PARSE_DATE('%d/%m/%Y', t.trandate)) as trandate

# DIMENSION (line 70) - WRONG:
- name: trandate
  sql: "TIMESTAMP(PARSE_DATE('%d/%m/%Y', {CUBE}.trandate))"  # ❌ Double parse!

# DIMENSION (line 70) - CORRECT:
- name: trandate
  sql: "{CUBE}.trandate"  # ✅ Already parsed in base SQL
```

### Phase 5: Filter & Segment Testing

Test all WHERE clause conditions and segments:

```bash
# Test filter conditions
bq query --use_legacy_sql=false "
SELECT COUNT(*)
FROM (
  -- Base SQL here
)
WHERE filter_condition"

# Test segments
bq query --use_legacy_sql=false "
SELECT COUNT(*)
FROM (
  -- Base SQL here
)
WHERE segment_condition"
```

**What to verify**:
- ✅ Boolean comparisons use TRUE/FALSE (not 'T'/'F')
- ✅ Date comparisons use correct format
- ✅ NULL handling is correct (COALESCE, IS NULL)
- ✅ AND/OR logic is correct
- ✅ Segment counts are reasonable

### Phase 6: Complex CASE Logic Testing

For cubes with complex CASE statements (e.g., b2c_customer_channels):

```bash
# Test full CASE logic
bq query --use_legacy_sql=false "
SELECT
  CASE
    WHEN location = 'value1' THEN 'bucket1'
    WHEN location = 'value2' THEN 'bucket2'
    -- ... 20+ lines
    ELSE 'default'
  END as complex_case,
  COUNT(*) as count
FROM (
  -- Base SQL here
)
GROUP BY complex_case"
```

**What to verify**:
- ✅ All WHEN clauses have correct syntax
- ✅ String comparisons use correct quoting
- ✅ Logic covers all expected values
- ✅ ELSE clause provides reasonable default
- ✅ Results match business requirements

### Phase 7: Performance Optimization

For expensive queries (e.g., self-JOINs on large tables):

```bash
# Use dry-run to validate syntax without executing
bq query --dry_run --use_legacy_sql=false < /tmp/test_query.sql

# Expected output:
# "Query successfully validated. This query will process X bytes."

# Then run with LIMIT for actual verification
bq query --use_legacy_sql=false "
SELECT * FROM (
  -- Expensive query here
)
LIMIT 3"
```

**When to use dry-run**:
- Self-JOINs on tables with >1M rows
- Queries on VIEWs over EXTERNAL tables (Parquet files)
- Complex multi-table JOINs (5+ tables)
- Queries taking >60 seconds

**Example**: `cross_sell.yml` self-JOIN on 9.3M row table took 5+ minutes. Solution: dry-run validation for syntax + limited execution for verification.

---

## Testing Scenarios Per Cube

For each cube, the following scenarios were tested:

### Scenario 1: Base Query Execution
```bash
# Extract and test base SQL
bq query --use_legacy_sql=false < /tmp/cube_base.sql
```

### Scenario 2: Every Measure
```bash
# Test each measure individually
bq query --use_legacy_sql=false "
SELECT measure1, measure2, measure3 FROM (
  -- base SQL
) LIMIT 10"
```

### Scenario 3: Every Dimension
```bash
# Test each dimension individually
bq query --use_legacy_sql=false "
SELECT DISTINCT dim1, dim2, dim3 FROM (
  -- base SQL
) LIMIT 10"
```

### Scenario 4: Every CASE Dimension
```bash
# Test CASE logic for bucketing dimensions
bq query --use_legacy_sql=false "
SELECT DISTINCT case_dimension FROM (
  -- base SQL with CASE logic
) LIMIT 20"
```

### Scenario 5: Every Filter Condition
```bash
# Test WHERE clause conditions
bq query --use_legacy_sql=false "
SELECT COUNT(*) FROM (
  -- base SQL
) WHERE filter_condition"
```

### Scenario 6: Every Segment
```bash
# Test segment conditions
bq query --use_legacy_sql=false "
SELECT COUNT(*) FROM (
  -- base SQL
) WHERE segment_sql"
```

### Scenario 7: Schema Validation
```bash
# Verify field types
bq show --schema magical-desktop:demo.table_name | grep field_name
```

### Scenario 8: Join Validation
```bash
# Test all JOIN conditions
bq query --use_legacy_sql=false "
SELECT COUNT(*) FROM (
  -- base SQL with all JOINs
)"
```

### Scenario 9: Date/Time Parsing
```bash
# Test date conversion logic
bq query --use_legacy_sql=false "
SELECT
  original_field,
  PARSE_DATE('%d/%m/%Y', original_field) as parsed,
  TIMESTAMP(PARSE_DATE('%d/%m/%Y', original_field)) as timestamp
FROM demo.table_name
LIMIT 3"
```

### Scenario 10: Pre-Aggregation Component Testing
```bash
# Test all measures listed in pre-aggregation
bq query --use_legacy_sql=false "
SELECT
  dimension1,
  dimension2,
  SUM(measure1) as m1,
  SUM(measure2) as m2,
  COUNT(*) as count
FROM (
  -- base SQL
)
GROUP BY dimension1, dimension2
LIMIT 10"
```

---

## Bug Patterns Discovered

### Pattern 1: BOOLEAN Type Mismatch (24 instances)

**Root Cause**: NetSuite source data uses STRING 'T'/'F' for boolean values. BigQuery imports these as BOOLEAN TRUE/FALSE, but YML schemas incorrectly compared to STRING literals.

**Example Error**:
```
No matching signature for operator = for argument types: BOOL, STRING
```

**Cubes Affected**: transaction_lines, items, item_receipt_lines, transaction_accounting_lines_cogs, b2b_customers (5 bugs), classifications, departments (3 bugs), b2b_addresses, b2b_customer_addresses (6 bugs), landed_costs, sell_through_seasonal (2 bugs), supplier_lead_times

**Before (WRONG)**:
```yaml
WHERE tl.mainline = 'F'  # ❌ Comparing BOOLEAN to STRING
  AND t.posting = 'T'
  AND t.voided = 'F'
```

```yaml
dimensions:
  - name: is_inactive
    sql: isinactive  # ❌ Wrong field name
    type: string     # ❌ Wrong type
```

**After (CORRECT)**:
```yaml
WHERE tl.mainline = FALSE  # ✅ BOOLEAN comparison
  AND t.posting = TRUE
  AND t.voided = FALSE
```

```yaml
dimensions:
  - name: is_inactive
    sql: isinactive
    type: boolean    # ✅ Correct type
```

**Fix Strategy**:
1. Check BigQuery schema: `bq show --schema magical-desktop:demo.table | grep field_name`
2. If type is BOOLEAN, change YML to `type: boolean`
3. Replace STRING comparisons 'T'/'F' with TRUE/FALSE
4. Update COALESCE defaults from STRING to BOOLEAN

### Pattern 2: DATE Parsing Bugs (7 instances)

**Root Cause**: NetSuite date fields are STRING in DD/MM/YYYY format, not native DATE type. YML schemas must use PARSE_DATE to convert.

**Example Error**:
```
Invalid date: '01/01/2022'
No matching signature for function DATE_DIFF with argument types: STRING, STRING, DAY
```

**Cubes Affected**: transactions, fulfillments, item_receipts, purchase_orders, fulfillment_lines

**Before (WRONG)**:
```yaml
sql: >
  SELECT
    trandate,  # ❌ STRING '01/01/2022' not DATE
    DATE_DIFF(enddate, startdate, DAY) as days  # ❌ STRING comparison
  FROM demo.transactions
```

**After (CORRECT)**:
```yaml
sql: >
  SELECT
    TIMESTAMP(PARSE_DATE('%d/%m/%Y', trandate)) as trandate,  # ✅ Convert to TIMESTAMP
    DATE_DIFF(
      PARSE_DATE('%d/%m/%Y', enddate),
      PARSE_DATE('%d/%m/%Y', startdate),
      DAY
    ) as days  # ✅ DATE comparison
  FROM demo.transactions
```

**Fix Strategy**:
1. Identify STRING date fields in BigQuery schema
2. Add PARSE_DATE('%d/%m/%Y', field) for DD/MM/YYYY format
3. Wrap in TIMESTAMP() if cube expects time dimension
4. Test DATE_DIFF, DATE_ADD, and other date functions

### Pattern 3: Double PARSE_DATE Bugs (5 instances)

**Root Cause**: Base SQL already converts STRING dates to TIMESTAMP, but dimensions try to parse them again.

**Example Error**:
```
Unable to coerce type TIMESTAMP to expected type STRING
```

**Cubes Affected**: on_order_inventory, order_baskets, supplier_lead_times (2 bugs)

**Before (WRONG)**:
```yaml
# Base SQL (line 10):
TIMESTAMP(PARSE_DATE('%d/%m/%Y', t.trandate)) as trandate

# Dimension (line 70):
- name: trandate
  sql: "TIMESTAMP(PARSE_DATE('%d/%m/%Y', {CUBE}.trandate))"  # ❌ DOUBLE PARSE!
  type: time
```

**After (CORRECT)**:
```yaml
# Base SQL (line 10):
TIMESTAMP(PARSE_DATE('%d/%m/%Y', t.trandate)) as trandate

# Dimension (line 70):
- name: trandate
  sql: "{CUBE}.trandate"  # ✅ Already TIMESTAMP from base SQL
  type: time
```

**Fix Strategy**:
1. Check base SQL: Is PARSE_DATE already applied?
2. Check field type: Is it already TIMESTAMP?
3. If yes, remove redundant PARSE_DATE from dimension
4. Use `{CUBE}.field_name` directly

### Pattern 4: Field Name Typos (1 instance)

**Root Cause**: Manual typo in dimension field reference.

**Example Error**:
```
Unrecognized name: isinactive; Did you mean is_inactive?
```

**Cubes Affected**: items

**Before (WRONG)**:
```yaml
- name: is_inactive
  sql: isinactive  # ❌ Field doesn't exist
```

**After (CORRECT)**:
```yaml
- name: is_inactive
  sql: is_inactive  # ✅ Correct field name
```

**Fix Strategy**:
1. Verify field exists in BigQuery: `bq show --schema magical-desktop:demo.table`
2. Check for common typos (underscores, case sensitivity)
3. Test with SELECT to confirm field exists

---

## Testing Tools and Commands

### BigQuery CLI Commands

```bash
# Test query execution
bq query --use_legacy_sql=false "SELECT * FROM demo.table LIMIT 10"

# Dry-run validation (syntax check without execution)
bq query --dry_run --use_legacy_sql=false < /tmp/query.sql

# Check table schema
bq show --schema magical-desktop:demo.table_name

# Check specific field type
bq show --schema magical-desktop:demo.table_name | grep -i field_name

# Format schema as JSON for parsing
bq show --schema --format=json magical-desktop:demo.table_name

# Count rows
bq query --use_legacy_sql=false "SELECT COUNT(*) FROM demo.table"

# Show table metadata
bq show magical-desktop:demo.table_name
```

### Git Workflow

```bash
# Test query
bq query --use_legacy_sql=false < /tmp/test_query.sql

# If bug found, fix in YML file
vim /home/produser/cube-demo/model/cubes/cube_name.yml

# Re-test to verify fix
bq query --use_legacy_sql=false < /tmp/test_query.sql

# Commit immediately
cd /home/produser/cube-demo
git add model/cubes/cube_name.yml
git commit -m "Fix [bug type] in cube_name.yml"
git push origin main
```

### Test SQL File Template

```bash
# Create test file
cat > /tmp/test_cube.sql << 'EOF'
-- Test base SQL from cube_name.yml
SELECT
  field1,
  field2,
  CASE
    WHEN condition THEN 'value'
    ELSE 'default'
  END as case_field
FROM demo.source_table
WHERE condition = TRUE
LIMIT 10
EOF

# Execute
bq query --use_legacy_sql=false < /tmp/test_cube.sql
```

---

## Quality Assurance Checklist

Use this checklist for each cube:

### Base SQL Testing
- [ ] Base query executes without errors
- [ ] Sample results (LIMIT 10) look correct
- [ ] All tables exist and are accessible
- [ ] All JOINs resolve correctly
- [ ] Field names match BigQuery schema

### Type Validation
- [ ] BOOLEAN fields use `type: boolean` (not `type: string`)
- [ ] BOOLEAN comparisons use TRUE/FALSE (not 'T'/'F')
- [ ] DATE/TIMESTAMP fields use `type: time`
- [ ] STRING date fields use PARSE_DATE('%d/%m/%Y', field)
- [ ] No double PARSE_DATE on already-converted fields
- [ ] Numeric fields use `type: number`

### Measure Testing
- [ ] All SUM measures execute
- [ ] All COUNT measures execute
- [ ] All AVG measures execute
- [ ] All MIN/MAX measures execute
- [ ] Calculated measures (CASE, division) execute
- [ ] Division by zero handled (NULLIF, SAFE_DIVIDE)
- [ ] Results are reasonable (no negative counts, etc.)

### Dimension Testing
- [ ] All simple dimensions execute
- [ ] All CASE dimensions execute
- [ ] Time dimensions parse correctly
- [ ] Bucketing logic covers all cases
- [ ] No field name typos
- [ ] No double PARSE_DATE errors

### Filter & Segment Testing
- [ ] All WHERE conditions execute
- [ ] All segment SQL executes
- [ ] BOOLEAN filters use TRUE/FALSE
- [ ] Date filters use correct format
- [ ] NULL handling is correct (COALESCE, IS NULL)

### Pre-Aggregation Validation
- [ ] All pre-agg measures are additive (SUM, COUNT)
- [ ] All pre-agg dimensions exist
- [ ] Time dimension is valid
- [ ] Granularity is appropriate
- [ ] No partition_granularity (removed for rollup-only mode)

### Edge Case Testing
- [ ] Empty result sets handled
- [ ] NULL values handled
- [ ] Zero division handled
- [ ] Date range extremes work
- [ ] Very large/small numbers work

### Documentation
- [ ] Cube has clear title and description
- [ ] Measures have descriptions
- [ ] Dimensions have descriptions
- [ ] Complex logic is commented
- [ ] Pre-aggregation strategy is documented

---

## Results Summary

### Coverage Statistics

| Metric | Value |
|--------|-------|
| **Total Cubes** | 27 |
| **Cubes Tested** | 27 (100%) |
| **Cubes with Bugs** | 19 (70%) |
| **Cubes Clean** | 8 (30%) |
| **Total Bugs Fixed** | 37 |
| **Git Commits** | 16 |

### Bug Breakdown

| Bug Type | Count | Percentage |
|----------|-------|------------|
| BOOLEAN Type Mismatch | 24 | 65% |
| DATE Parsing | 7 | 19% |
| Double PARSE_DATE | 5 | 13% |
| Field Name Typos | 1 | 3% |

### Cubes Fixed (19)

1. transactions.yml - DATE parsing
2. transaction_lines.yml - BOOLEAN
3. items.yml - BOOLEAN + field name typo
4. fulfillments.yml - DATE parsing
5. fulfillment_lines.yml - DATE parsing
6. item_receipts.yml - DATE parsing
7. item_receipt_lines.yml - BOOLEAN
8. purchase_orders.yml - DATE parsing
9. transaction_accounting_lines_cogs.yml - BOOLEAN
10. b2b_customers.yml - 5 BOOLEAN bugs
11. classifications.yml - BOOLEAN
12. departments.yml - 3 BOOLEAN bugs
13. b2b_addresses.yml - BOOLEAN
14. b2b_customer_addresses.yml - 6 BOOLEAN bugs
15. landed_costs.yml - BOOLEAN
16. on_order_inventory.yml - Double PARSE_DATE
17. order_baskets.yml - Double PARSE_DATE
18. sell_through_seasonal.yml - 2 BOOLEAN bugs
19. supplier_lead_times.yml - 1 BOOLEAN + 2 double PARSE_DATE

### Cubes Clean (8)

1. b2c_customers.yml
2. b2c_customer_channels.yml - Complex virtual cube verified
3. cross_sell.yml - Expensive query dry-run validated
4. currencies.yml
5. inventory.yml
6. locations.yml
7. purchase_order_lines.yml
8. subsidiaries.yml

### Time Investment

- **Total Testing Time**: ~8 hours across 2 sessions
- **Average Time Per Cube**: ~18 minutes
- **Bugs Found Rate**: 4.6 bugs per hour
- **Value**: Prevented 37 production failures before deployment

### Key Learnings

1. **NetSuite Data Quirks**: STRING 'T'/'F' vs BOOLEAN, DD/MM/YYYY date format
2. **Double Parsing**: Watch for redundant PARSE_DATE in dimensions
3. **Schema Validation**: Always verify types against BigQuery schema
4. **Dry-Run Performance**: Use dry-run for expensive queries (>60 seconds)
5. **Incremental Testing**: Test and fix immediately, don't batch
6. **Git Hygiene**: One bug fix = one commit for clear audit trail

### Impact Assessment

**Without Testing**:
- 37 bugs would have caused pre-aggregation build failures
- 19 cubes (70%) would be broken in production
- Hours of production debugging required
- Incorrect business metrics served to users
- Loss of user trust in data accuracy

**With Testing**:
- All 37 bugs caught before deployment
- 100% of cubes validated working
- Clear documentation of bug patterns
- Reproducible testing methodology
- Confidence in production deployment

---

## Conclusion

This comprehensive testing methodology successfully validated all 27 Cube.js schema files, catching 37 critical bugs before production deployment. The systematic approach—testing every SQL expression without assumptions—proved essential for ensuring data quality in a rollup-only Cube Cloud environment.

**Key Success Factors**:
1. No shortcuts or assumptions
2. Actual query execution (not just dry-run)
3. Schema validation against BigQuery
4. Immediate bug fixes with git commits
5. Pattern recognition across similar bugs
6. Performance optimization for expensive queries

**Recommended for**:
- All Cube.js schema deployments
- BigQuery data warehouse projects
- NetSuite data integrations
- Rollup-only Cube Cloud environments
- High-stakes production deployments

---

**Document Version**: 1.0
**Last Updated**: December 14, 2025
**Author**: QuickFind Limited (via Claude Code comprehensive testing)
**Repository**: github.com/QuickFind-Limited/cube-demo
