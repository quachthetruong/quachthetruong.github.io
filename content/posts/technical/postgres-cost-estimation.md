---
title: "How PostgreSQL Evaluates Execution Plans: Cost Estimation Deep Dive"
date: 2025-10-01T10:00:00+07:00
tags: ["computer science", "database"]
---

When diving deep into the `EXPLAIN` command, you already know what index scans, sequential scans, and hash joins are. But have you ever wondered how exactly the cost numbers are calculated? In this deep dive, we'll explore the formulas behind PostgreSQL's cost estimation for the three main scanning approaches: sequential scan, index scan, and bitmap heap scan.

## 1. Cost-based vs Rule-based {#postgresql-cost-constants}

Consider this query:

```sql
SELECT * FROM table_a ta
JOIN table_b tb ON ta.id = tb.foreign_id
WHERE ta.status = 'active' AND tb.created_at > '2020-12-01';
```

How would you approach this query? The intuitive flow would be: filter each table first, then join. Make tables smaller first, then start joining. Quite intuitive, right?

This intuitive approach represents **rule-based optimization** - applying fixed rules regardless of data characteristics. But what if `table_b` has only `100` rows while `table_a` has `10` million? The join could eliminate `99.99%` of the data before the expensive filter operation. In the early days of databases, query optimizers used rule-based approaches that couldn't handle such scenarios effectively.

This is where **cost-based optimization** shines - the solution PostgreSQL adopted. PostgreSQL imagines **all possible execution scenarios** - considering table statistics, cardinality, storage characteristics (SSD vs HDD), and different access methods (sequential scan, index scan, nested loop join, merge join). It then calculates costs for each approach and picks the cheapest one.

Here's how PostgreSQL shows these costs in practice:

```sql
EXPLAIN SELECT * FROM users;
                        QUERY PLAN
--------------------------------------------------------
 Seq Scan on users  (cost=0.00..450.00 rows=10000 width=64)
```

This shows four key numbers:

-   **startup_cost**: `0.00` (sequential scan starts immediately)
-   **total_cost**: `450.00` (cost to scan entire table)
-   **rows**: `10000` (estimated number of rows)
-   **width**: `64` (average row size in bytes)

Here are the key constants PostgreSQL uses in cost estimation:

```python
# Cost model components:
startup_cost                          # Time before first tuple (includes finding in index file, sorting)
run_cost                              # Time to fetch data and perform operations (filtering, joins, sorting, etc.)
total_cost                            # startup_cost + run_cost (total execution cost)

# PostgreSQL cost constants (from src/backend/optimizer/path/costsize.c):
seq_page_cost                         # Cost of a sequential page fetch (default: 1.0)
random_page_cost                      # Cost of a non-sequential page fetch (default: 4.0)
cpu_tuple_cost                        # Cost of typical CPU time to process a tuple (default: 0.01)
cpu_index_tuple_cost                  # Cost of typical CPU time to process an index tuple (default: 0.005)
cpu_operator_cost                     # Cost of CPU time to execute an operator or function (default: 0.0025)

# Index cost multipliers (from src/backend/utils/adt/selfuncs.c):
DEFAULT_PAGE_CPU_MULTIPLIER           # Multiplier for CPU cost per B-tree page descended (default: 50x cpu_operator_cost)

# Query variables:
N_tuple                               # Total number of tuples in table
N_page                                # Total number of pages in table
N_index_tuple                         # Total number of tuples in index
N_index_page                          # Total number of pages in index
Selectivity                           # Proportion of search range satisfying WHERE clause (0 to 1)
H_index                               # Height of internal tree levels (not including leaf level)
```

These are dimensionless units for relative comparison, not absolute performance.

## 2. Sequential Scan

For sequential scans, the startup cost is always `0` since there's no index traversal needed - PostgreSQL can read the first tuple immediately from the data file. Let's estimate the cost for this query:

```sql
SELECT * FROM tbl WHERE id <= 8000;
```

The run cost is calculated using:

```python
run_cost = cpu_cost + io_cost
run_cost = (cpu_tuple_cost + cpu_operator_cost) * N_tuple + seq_page_cost * N_page
```

Retrieve table statistics:

```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 'tbl';
 relpages | reltuples
----------+-----------
       45 |     10000
(1 row)
```

-   `N_tuple` = `10000`
-   `N_page` = `45`

Therefore:

```python
run_cost = (0.01 + 0.0025) * 10000 + 1.0 * 45  # = 170
```

Finally:

```python
total_cost = startup_cost + run_cost  # = 0 + 170 = 170
```

For confirmation, the result of the `EXPLAIN` command of the above query is shown below:

```sql
EXPLAIN SELECT * FROM tbl WHERE id <= 8000;
                       QUERY PLAN
--------------------------------------------------------
 Seq Scan on tbl  (cost=0.00..170.00 rows=8000 width=8)
   Filter: (id <= 8000)
(2 rows)
```

In the output above, we can see that the startup and total costs are 0.00 and 170.00, respectively, which matches our calculation perfectly. PostgreSQL estimates that 8000 rows will be selected by scanning all rows.

Even though the condition `id <= 8000` matches only `8000` rows, sequential scan still reads the entire table (`10000` rows) and applies the filter during tuple processing.

> **Important**: As understood from the run-cost estimation, PostgreSQL assumes that all pages will be read from storage. In other words, PostgreSQL does not consider whether the scanned page is in the shared buffers or not.

## 3. Index Scan

Let's explore how to estimate the index scan cost for the following query:

```sql
SELECT id, data FROM tbl WHERE data <= 240;
```

Before estimating the cost, the numbers of the index pages and index tuples are shown below:

```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 'tbl_data_idx';
 relpages | reltuples
----------+-----------
       30 |     10000
(1 row)
```

-   `N_index_tuple` = `10000`
-   `N_index_page` = `30`

### 3.1. Startup Cost

The startup cost represents the cost of reading index pages to access the first tuple. It has two components:

```python
startup_cost = scan_cost + nav_cost
startup_cost = ceil(log2(N_index_tuple)) * cpu_operator_cost + (H_index + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost
```

-   **Key comparisons**: `ceil(log2(N_index_tuple))` - CPU cost of binary search comparisons to locate key
-   **Page processing**: `(H_index + 1) * DEFAULT_PAGE_CPU_MULTIPLIER` - CPU cost of processing pages while descending tree levels, including leaf level

To find the actual tree height:

```sql
CREATE EXTENSION pageinspect;
SELECT btpo_level FROM bt_page_stats('tbl_data_idx', bt_metap('tbl_data_idx')->root);
 btpo_level
------------
          1
(1 row)
```

Therefore:

```python
startup_cost = ceil(log2(10000)) * 0.0025 + (1 + 1) * 50 * 0.0025  # = 0.285
```

### 3.2. Run Cost

The run cost of the index scan is the sum of the CPU costs and the I/O costs of both the table and the index:

```python
run_cost = (index_cpu_cost + table_cpu_cost) + (index_io_cost + table_io_cost)
```

> **Info**: If Index-Only Scans can be applied, `table_cpu_cost` and `table_io_cost` are not estimated.

The first three costs (i.e., index CPU cost, table CPU cost, and index I/O cost) are shown below:

```python
index_cpu_cost = Selectivity * N_index_tuple * (cpu_index_tuple_cost + cpu_operator_cost)
table_cpu_cost = Selectivity * N_tuple * cpu_tuple_cost
index_io_cost  = ceil(Selectivity * N_index_page) * random_page_cost
```

<div style="
  background-color: var(--tertiary);
  border-left: 4px solid var(--primary);
  padding: 1rem;
  margin: 1rem 0;
  border-radius: 0.375rem;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
">

**📝 Background: Selectivity Calculation**

The selectivity of query predicates is estimated using either the `histogram_bounds` or the MCV (Most Common Value), both of which are stored in the statistics information in the `pg_stats`.

#### MCV-based Selectivity

The MCV of each column is stored in the `pg_stats` view as a pair of columns:

```sql
SELECT most_common_vals, most_common_freqs FROM pg_stats
WHERE tablename = 'countries' AND attname='continent';
------------------+-------------------------------------------------------------
most_common_vals  | {"Africa","Europe","Asia","North America","Oceania","South America"}
most_common_freqs | {0.274611,0.243523,0.227979,0.119171,0.0725389,0.0621762}
(1 row)
```

For a query like `continent = 'Asia'`, the selectivity is the frequency corresponding to 'Asia' in `most_common_freqs`, which is `0.227979`.

#### Histogram-based Selectivity

If the MCV cannot be used, e.g., the target column type is integer or double precision, then the value of the `histogram_bounds` of the target column is used to estimate the cost.

`histogram_bounds` is a list of values that divide the column's values into groups of approximately equal population.

A specific example is shown. This is the value of the `histogram_bounds` of the column 'data' in the table 'tbl':

```sql
SELECT histogram_bounds FROM pg_stats WHERE tablename = 'tbl' AND attname = 'data';
                                   histogram_bounds
---------------------------------------------------------------------------------------------------
 {1,100,200,300,400,500,600,700,800,900,1000,1100,1200,1300,1400,1500,1600,1700,1800,1900,2000,2100,
2200,2300,2400,2500,2600,2700,2800,2900,3000,3100,3200,3300,3400,3500,3600,3700,3800,3900,4000,4100,
4200,4300,4400,4500,4600,4700,4800,4900,5000,5100,5200,5300,5400,5500,5600,5700,5800,5900,6000,6100,
6200,6300,6400,6500,6600,6700,6800,6900,7000,7100,7200,7300,7400,7500,7600,7700,7800,7900,8000,8100,
8200,8300,8400,8500,8600,8700,8800,8900,9000,9100,9200,9300,9400,9500,9600,9700,9800,9900,10000}
(1 row)
```

By default, the histogram_bounds is divided into `100` buckets. Buckets are numbered starting from `0`, and every bucket stores approximately the same number of tuples. The values of histogram_bounds are the bounds of the corresponding buckets:

```text
histogram_bounds | hb(0) | hb(1) | hb(2) | hb(3) | ... | hb(99) | hb(100)
-----------------+-------+-------+-------+-------+-----+--------+---------
                 |     1 |   100 |   200 |   300 | ... |   9900 |   10000
```

For our query `data <= 240`, the value `240` falls in bucket 2 (between `hb[2] = 200` and `hb[3] = 300`). Using linear interpolation:

```python
position_in_bucket = (target_value - bucket_min) / (bucket_max - bucket_min)
Selectivity        = (bucket_index + position_in_bucket) / total_buckets
Selectivity        = (2 + (240 - 200) / (300 - 200)) / 100   # = 0.024
```

</div>

Using our selectivity value of `0.024`:

```python
index_cpu_cost = 0.024 * 10000 * (0.005 + 0.0025)  # = 1.8
table_cpu_cost = 0.024 * 10000 * 0.01              # = 2.4
index_io_cost  = ceil(0.024 * 30) * 4.0            # = 4.0
```

The `table_io_cost` is defined by:

```python
table_io_cost = max_io_cost - pow(index_correlation, 2) * (max_io_cost - min_io_cost)
```

-   **max_io_cost** is the worst case I/O cost (randomly scanning all table pages):

```python
max_io_cost = N_page * random_page_cost
max_io_cost = 45 * 4.0  # = 180
```

-   **min_io_cost** is the best case I/O cost (sequentially scanning selected pages):

```python
min_io_cost = 1 * random_page_cost + (ceil(Selectivity * N_page) - 1) * seq_page_cost
min_io_cost = 1 * 4.0 + (ceil(0.024 * 45) - 1) * 1.0    # = 5
```

<div style="
  background-color: var(--tertiary);
  border-left: 4px solid var(--primary);
  padding: 1rem;
  margin: 1rem 0;
  border-radius: 0.375rem;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
">

**📝 Background: Index Correlation**

Index correlation is a statistical correlation between the physical row ordering and the logical ordering of the column values. This ranges from `-1` to `+1`.

```sql
SELECT col,col_asc,col_desc,col_rand
testdb-#                         FROM tbl_corr;
   col    | col_asc | col_desc | col_rand
----------+---------+----------+----------
 Tuple_1  |       1 |       12 |        3
 Tuple_2  |       2 |       11 |        8
 Tuple_3  |       3 |       10 |        5
 Tuple_4  |       4 |        9 |        9
 Tuple_5  |       5 |        8 |        7
 Tuple_6  |       6 |        7 |        2
 Tuple_7  |       7 |        6 |       10
 Tuple_8  |       8 |        5 |       11
 Tuple_9  |       9 |        4 |        4
 Tuple_10 |      10 |        3 |        1
 Tuple_11 |      11 |        2 |       12
 Tuple_12 |      12 |        1 |        6
(12 rows)
```

```sql
SELECT tablename,attname, correlation FROM pg_stats WHERE tablename = 'tbl_corr';
 tablename | attname  | correlation
-----------+----------+-------------
 tbl_corr  | col_asc  |           1
 tbl_corr  | col_desc |          -1
 tbl_corr  | col_rand |    0.125874
(3 rows)
```

Perfect correlation (`1.0`) means rows are stored in index order, enabling sequential access. Poor correlation (`0.0`) forces random page access.

**Summary**: Higher correlation (closer to `1.0`) → Lower I/O costs due to sequential access. Lower correlation (closer to `0.0`) → Higher I/O costs due to random access.

</div>

In this example, `index_correlation = 1.0`:

```python
table_io_cost = max_io_cost - pow(index_correlation, 2) * (max_io_cost - min_io_cost)
table_io_cost = 180 - pow(1.0, 2) * (180 - 5)   # = 5
```

Finally, we can calculate the total run cost:

```python
run_cost = (index_cpu_cost + table_cpu_cost) + (index_io_cost + table_io_cost)
run_cost = (1.8 + 2.4) + (4.0 + 5)  # = 13.2
```

### 3.3. Total Cost

According to the previous calculations:

```python
total_cost = startup_cost + run_cost
total_cost = 0.285 + 13.2   # = 13.485
```

For confirmation, the result of the EXPLAIN command shows:

```sql
EXPLAIN SELECT id, data FROM tbl WHERE data <= 240;
                           QUERY PLAN
-----------------------------------------------------------------------
 Index Scan using tbl_data_idx on tbl (cost=0.29..13.49 rows=240 width=8)
   Index Cond: (data <= 240)
(2 rows)
```

Our calculated cost (`13.485`) closely matches PostgreSQL's estimate (`13.49`), validating the formula accuracy.

> **SSD Optimization**: The [default values](#postgresql-cost-constants) assume random scans are four times slower than sequential scans, reflecting traditional HDD performance. For SSDs, consider reducing [`random_page_cost`](#postgresql-cost-constants) to around `1.0` for better query plans.

## 4. Bitmap Scan

Bitmap scans combine the benefits of index scans and sequential scans. Let's set up a test table to examine bitmap scan cost estimation:

```sql
CREATE TABLE foo (typ int, bar int, id1 int);
CREATE INDEX ON foo(typ);
CREATE INDEX ON foo(bar);
CREATE INDEX ON foo(id1);

INSERT INTO foo (typ, bar, id1)
SELECT
    CAST(cos(2 * pi() * random()) * sqrt(-2 * ln(random())) * 100 AS integer),
    n % 97, n % 101
FROM generate_series(1, 1000000) n;

VACUUM ANALYZE foo;
```

Now let's estimate the cost for this Bitmap Scan query:

```sql
SELECT * FROM foo WHERE bar = 2;
```

First, let's get the table statistics:

```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 'foo';
 relpages | reltuples
----------+-----------
     5405 |   1000000
(1 row)
```

-   `N_tuple` = `1000000`
-   `N_page` = `5405`

For the index statistics:

```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 'foo_bar_idx';
 relpages | reltuples
----------+-----------
      871 |   1000000
(1 row)
```

-   `N_index_page` = `871`
-   `N_index_tuple` = `1000000`

The planner estimates the selectivity here at approximately:

```python
Selectivity = rows_returned / N_tuple
Selectivity = 10200 / 1000000   # = 0.0102
```

The total Bitmap Index Scan cost is calculated identically to the plain Index Scan cost, except for table scans:

```python
# Calculate selected pages and tuples from index
pages = N_index_page * Selectivity
pages = 871 * 0.0102  # = 8.8842

tuples = round(N_index_tuple * Selectivity)
tuples = round(1000000 * 0.0102)  # = 10200

# Bitmap Index Scan cost
bitmap_index_cost = round(random_page_cost * pages + cpu_index_tuple_cost * tuples + cpu_operator_cost * tuples)
bitmap_index_cost = round(4.0 * 8.8842 + 0.005 * 10200 + 0.0025 * 10200)  # = 112
```

The Bitmap Heap Scan I/O cost calculation differs from Index Scan. Pages are fetched in ascending order without repeats, but tuples are no longer sequentially located, increasing the number of pages to fetch.

<div style="
  background-color: var(--tertiary);
  border-left: 4px solid var(--primary);
  padding: 1rem;
  margin: 1rem 0;
  border-radius: 0.375rem;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
">

**📝 Background: Mackert-Lohman Formula**

The Mackert-Lohman formula estimates the number of pages fetched in a bitmap scan, based on the coupon collector problem. It calculates how many unique pages contain the selected tuples:

<div style="display: inline-block; background-color: #6A6767; width: 35rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?\color{white}\text{pages\_fetched} = \min\left(\frac{2 \cdot \text{N\_page} \cdot \text{tuples\_fetched}}{2 \cdot \text{N\_page} + \text{tuples\_fetched}}, \text{N\_page}\right)" title="Mackert-Lohman Formula" />
</div>

```python
# First calculate tuples fetched
tuples_fetched = N_tuple * Selectivity
tuples_fetched = 1000000 * 0.0102  # = 10200

# Apply Mackert-Lohman formula
pages_fetched = min((2 * N_page * tuples_fetched) / (2 * N_page + tuples_fetched), N_page)
pages_fetched = min((2 * 5405 * 10200) / (2 * 5405 + 10200), 5405)  # = 5248.54
```

The formula accounts for the probability of tuple distribution across pages, with the `min()` ensuring we never fetch more pages than the table actually has.

</div>

The fetch cost of a single page is estimated somewhere between seq_page_cost and random_page_cost, depending on the fraction of the total number of pages to be fetched.

```python
# Calculate cost per page (interpolated between seq and random)
cost_per_page = random_page_cost - (random_page_cost - seq_page_cost) * (pages_fetched / N_page) ** 0.5
cost_per_page = 4.0 - (4.0 - 1.0) * (5248.54 / 5405) ** 0.5  # = 1.0440
```

The Bitmap Heap Scan includes processing costs for scanned tuples and recheck operations. PostgreSQL adds bitmap manipulation overhead of `1.1 * cpu_operator_cost` per expected index tuple: `0.1` for building the bitmap (startup cost) and `1.0` for rechecking tuples (run cost).

```python
# Bitmap Heap Scan startup cost (includes bitmap building cost)
bitmap_ops_multiplier = 0.1  # Bitmap building cost: 0.1 * cpu_operator_cost per index tuple
startup_cost = round(bitmap_index_cost + bitmap_ops_multiplier * cpu_operator_cost * N_tuple * Selectivity)
startup_cost = round(112 + 0.1 * 0.0025 * 1000000 * 0.0102)  # = 115

# Bitmap Heap Scan run cost
run_cost = round(cost_per_page * pages_fetched + cpu_tuple_cost * tuples_fetched + cpu_operator_cost * tuples_fetched)
run_cost = round(1.0440 * 5248.54 + 0.01 * 10200 + 0.0025 * 10200)  # = 5607

# Total cost
total_cost = startup_cost + run_cost
total_cost = 115 + 5607  # = 5722
```

For confirmation, the result of the EXPLAIN command shows:

```sql
EXPLAIN SELECT * FROM foo WHERE bar = 2;
                                   QUERY PLAN
--------------------------------------------------------------------------------
 Bitmap Heap Scan on foo  (cost=115.47..5722.32 rows=10200 width=12)
   Recheck Cond: (bar = 2)
   ->  Bitmap Index Scan on foo_bar_idx  (cost=0.00..112.92 rows=10200 width=0)
         Index Cond: (bar = 2)
(4 rows)
```

Our calculated costs are very close to PostgreSQL's estimates: startup cost (`115` vs `115.47`) and total cost (`5722` vs `5722.32`). The small differences are due to rounding in our step-by-step calculations, additional internal optimizations in PostgreSQL's implementation, and floating-point precision differences. This close match validates that our cost estimation formulas capture the core logic correctly.

## Conclusion

I initially wanted to define simple selectivity thresholds: 5% use index scan, 5-10% use bitmap scan, above 15% use sequential scan. However, after diving deep into PostgreSQL's cost calculations, I realize: just let the optimizer do its job. The complexity of factors like index correlation, page distribution, and hardware characteristics makes manual rules impractical. Our job is simply to create valuable indexes and trust PostgreSQL's cost-based optimizer.

## References

-   https://www.interdb.jp/pg/pgsql03/02.html
-   https://www.postgresql.org/docs/current/using-explain.html
-   https://www.rockdata.net/tutorial/plan-bitmap-scan/
-   https://gist.github.com/SamsadSajid/40e14bbb9157f53f44cca08d1e9eba39
-   https://github.com/postgres/postgres/blob/master/src/backend/optimizer/path/costsize.c
