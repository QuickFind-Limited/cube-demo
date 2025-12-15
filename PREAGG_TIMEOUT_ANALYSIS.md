# Pre-Aggregation Timeout Analysis

## Problem Statement
Single wide pre-aggregation with 20 dimensions times out after 10 minutes in Cube Cloud.

## Performance Test Results (1 Week of Data: Nov 1-7, 2024)

### Query Performance
- **Execution Time:** 9m 14s
- **Cube.js Timeout Limit:** 10 minutes
- **Status:** ❌ FAILURE (too close to timeout, will fail for some weeks)

### Query Complexity
- **Dimensions:** 20
- **JOINs:** 6 tables (items, transactions_analysis, currencies, locations, departments, classifications)
- **Measures:** 16 aggregated measures
- **Date Range:** 1 week (7 days)

### Extrapolated Performance
- 1 week = 9m 14s
- 1 month (4 weeks) = ~37 minutes (exceeds 20-minute Cube Cloud hard limit)

## Root Cause Analysis

### 1. High Cardinality Dimensions
- `sku` (itemid): ~5,000+ unique products
- `product_name` (displayname): ~5,000+ unique names
- `customer_email`: ~500K unique values (from old customer_geography preagg)

### 2. JOIN Overhead
Each LEFT JOIN adds significant overhead:
1. `transactions_analysis` (transaction metadata)
2. `items` (product details)
3. `currencies` (currency names)
4. `locations` (location names)
5. `departments` (department names)
6. `classifications` (classification names)

### 3. NULL Value Problem
Many dimensions have NULL values, reducing GROUP BY efficiency:
- `category`: NULL
- `sku`: NULL
- `product_name`: NULL
- `size`: NULL
- `color`: NULL
- `location_name`: NULL

## Recommended Solution: Split into Multiple Focused Pre-Aggregations

### Strategy: Query Pattern-Based Pre-Aggregations

Instead of 1 wide pre-aggregation, create 4-5 focused pre-aggregations based on actual query patterns:

#### 1. **core_sales** (General Analytics)
**Purpose:** High-level business metrics
**Dimensions (8):**
- channel_type
- transaction_type
- billing_country
- shipping_country
- department_name
- classification_name
- transaction_currency
- customer_type

**Performance:** ~2-3 minutes per week

#### 2. **product_analysis** (Product Performance)
**Purpose:** Product-level analytics
**Dimensions (10):**
- channel_type
- category
- section
- season
- product_range
- collection
- pricing_type
- size
- color
- sku (optional - high cardinality)

**Performance:** ~4-5 minutes per week

#### 3. **geography_analysis** (Location Analytics)
**Purpose:** Store/location performance
**Dimensions (6):**
- channel_type
- location_name
- billing_country
- shipping_country
- classification_name
- department_name

**Performance:** ~2-3 minutes per week

#### 4. **channel_product** (Channel + Product Mix)
**Purpose:** Channel-specific product analysis
**Dimensions (8):**
- channel_type
- category
- section
- season
- pricing_type
- size
- color
- customer_type

**Performance:** ~3-4 minutes per week

### Benefits
1. ✅ Each pre-agg builds in <6 minutes (well under 10-minute timeout)
2. ✅ Cube.js automatically selects the best pre-agg for each query
3. ✅ Reduced NULL values (only relevant dimensions per pre-agg)
4. ✅ Fewer JOINs per pre-agg (only needed tables)
5. ✅ Better partition performance (less data per partition)

### Trade-offs
- ⚠️ More pre-aggregations to maintain (4 vs 1)
- ⚠️ Slightly more storage (but still less than original 16)
- ✅ But: Better query performance and reliability

## Implementation Plan

1. Revert consolidation to pre-consolidation state (16 pre-aggs)
2. Analyze original 16 pre-aggs to identify actual query patterns
3. Design 4-5 focused pre-aggs based on patterns
4. Implement with weekly partitioning
5. Test each pre-agg build time (<6 minutes target)
6. Deploy and monitor

## Conclusion

**Single wide pre-aggregation is NOT viable** for this dataset with 20 dimensions and 6 JOINs. The solution is to split into 4-5 focused pre-aggregations based on actual query patterns, ensuring each builds well under the 10-minute timeout limit.
