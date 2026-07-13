# TPC-H Query Optimization & Choke Point Analysis

Performance analysis and optimization of all 22 TPC-H benchmark queries on SQLite (SF1, ~1GB, 8 tables). Identified structural bottlenecks causing 500x+ runtime spreads and rewrote the three worst-performing queries for up to a **12,632x speedup**.

Co-authored with Kobe Richards for CS 542 (Database Management Systems).

## Summary

TPC-H is a standard decision-support benchmark modeling a supplier/customer/order business (customers, orders, suppliers, parts, regions, 8 tables, 22 analytical queries). Because SQLite has no automatic subquery decorrelation, it executes correlated subqueries exactly as written, making it a clean environment to observe the *real* cost of query structure without an optimizer masking it.

## What I did

- Ran all 22 TPC-H queries against a 1GB SQLite database (LINEITEM: 6M rows, ORDERS: 1.5M rows, CUSTOMER: 150K rows, and 5 other tables), measuring cold-cache execution time (median of 3 runs)
- Classified all 22 queries into 6 structural categories (join-centric, aggregation-heavy, selection-focused, correlated-subquery, conditional-logic, distribution-analysis) based on `EXPLAIN QUERY PLAN` output
- Identified the 3 worst-performing queries (Q17, Q20, Q22) and manually rewrote them using CTEs, join conversion, and predicate reordering to eliminate correlated subqueries
- Verified each rewrite against the original result set for correctness before comparing runtimes

## Results

| Query | Original Runtime | Optimized Runtime | Speedup |
|---|---|---|---|
| Q17 | 2,334 sec | 36.8 sec | **62.5x** |
| Q20 | 8,304 sec | 36.9 sec | **232x** |
| Q22 | 2,413 sec | 0.19 sec | **12,632x** |

Q22 went from a 40-minute query to under 0.2 seconds by computing a repeated average once instead of ~150,000 times — cutting 22.5 billion row comparisons down to 300,000 (a 75,000x reduction in computational work).

## Key finding

Query **structure** predicts runtime far better than data size. Every extreme outlier (Q17, Q20, Q22) shared the same root cause: a correlated subquery being re-evaluated once per outer row, turning an O(n) problem into O(n²) or worse. The fix in each case followed the same principle, compute once, reuse many times, via CTEs and derived tables that materialize an aggregate before joining, instead of recalculating it inside a nested loop.

This is the difference between "the database is slow" and "the query is slow", and it's the first thing worth checking before assuming a system needs more hardware.

## Files

- [`optimized_queries.sql`](./optimized_queries.sql) - All 22 queries with original vs. rewritten versions and measured runtimes inline
- [`Project_Report.docx`](./Project_Report.docx) - Full writeup: methodology, related work, structural classification, and choke-point analysis
- 
## Reproducing this

The TPC-H database (~1GB at SF1) isn't included in this repo since it's a synthetic, regeneratable benchmark rather than original data. To reproduce:

1. Download the TPC-H tools (`dbgen`/`qgen`) from the [official TPC-H page](https://www.tpc.org/tpch/)
2. Generate data at scale factor 1: `./dbgen -s 1`
3. Load the generated `.tbl` files into SQLite using the standard TPC-H schema (8 tables: LINEITEM, ORDERS, CUSTOMER, PART, PARTSUPP, SUPPLIER, NATION, REGION)
4. Run the queries in [`optimized_queries.sql`](./optimized_queries.sql) against the resulting database using the SQLite CLI with `.timer on`

## Tools

SQLite 3.x, SQL (CTEs, window functions, query plan analysis), TPC-H `dbgen`/`qgen`

## References

Builds on choke-point analysis from Boncz, Neumann & Erling (*TPC-H Analyzed*, TPCTC 2014) and Dreseler et al. (*Quantifying TPC-H Choke Points*, VLDB 2020), full citations in the report.
