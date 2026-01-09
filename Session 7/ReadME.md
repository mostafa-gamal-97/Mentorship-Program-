# E-Commerce Database Query Optimization

This ReadMe contains the analysis for session 7 task-related queries.
This includes reading the execution plans for original queries, identifying bottlenecks in performance,
and performing different kinds of optimizations to reach query optimal performance.

---

## Query Optimizations

### 1. Write a SQL Query to Retrieve the total number of products in each category

```sql

EXPLAIN ANALYZE SELECT
    c.category_id,
    c.category_name,
    COUNT(p.product_id) AS products_count_per_category
FROM categories c
LEFT JOIN products p
    ON p.category_id = c.category_id
GROUP BY
    c.category_id,
    c.category_name
ORDER BY
    c.category_id;
```

#### Query Execution Plan Before Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Sort  (cost=600728.91..600978.91 rows=100000 width=30) (actual time=3594.082..3600.691 rows=100000 loops=1)
		  Sort Key: c.category_id
		  Sort Method: external merge  Disk: 4120kB
		  ->  HashAggregate  (cost=520739.12..590030.09 rows=100000 width=30) (actual time=3215.808..3561.663 rows=100000 loops=1)
				Group Key: c.category_id
				Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 122960kB
				->  Hash Right Join  (cost=2887.00..122700.38 rows=4994996 width=30) (actual time=183.699..1903.493 rows=4995100 loops=1)
					  Hash Cond: (p.category_id = c.category_id)
					  ->  Seq Scan on products p  (cost=0.00..106700.96 rows=4994996 width=16) (actual time=0.045..641.153 rows=4995000 loops=1)
					  ->  Hash  (cost=1637.00..1637.00 rows=100000 width=22) (actual time=182.798..182.799 rows=100000 loops=1)
							Buckets: 131072  Batches: 1  Memory Usage: 6385kB
							->  Seq Scan on categories c  (cost=0.00..1637.00 rows=100000 width=22) (actual time=0.033..16.195 rows=100000 loops=1)
		Planning Time: 0.732 ms
		JIT:
		  Functions: 17
		  Options: Inlining true, Optimization true, Expressions true, Deforming true
		  Timing: Generation 3.015 ms, Inlining 15.567 ms, Optimization 92.466 ms, Emission 60.009 ms, Total 171.056 ms
		Execution Time: 3619.211 ms
</details>

#### What to Optimize?

1. Add an index on products.category_id
	
	CREATE INDEX idx_products_category_id ON products(category_id);
	
   This speeds up the JOIN since PostgreSQL can quickly locate products for each category.

2. Optional: Use COUNT(*) inside LEFT JOIN instead of COUNT(p.product_id)

	In PostgreSQL, COUNT(p.product_id) and COUNT(*) after LEFT JOIN are similar.
	This is mostly semantic; indexing matters more.

#### Query Execution Plan After Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		GroupAggregate  (cost=0.72..411231.77 rows=100000 width=30) (actual time=4.205..2217.296 rows=100000 loops=1)
		  Group Key: c.category_id
		  ->  Merge Left Join  (cost=0.72..385256.77 rows=4995000 width=30) (actual time=4.083..1919.396 rows=4995100 loops=1)
				Merge Cond: (c.category_id = p.category_id)
				->  Index Scan using categories_pkey on categories c  (cost=0.29..3244.29 rows=100000 width=22) (actual time=0.016..15.687 rows=100000 loops=1)
				->  Index Scan using idx_products_category_id on products p  (cost=0.43..319324.97 rows=4995000 width=16) (actual time=0.029..1349.004 rows=4995000 loops=1)
		Planning Time: 0.207 ms
		JIT:
		  Functions: 10
		  Options: Inlining false, Optimization false, Expressions true, Deforming true
		  Timing: Generation 0.378 ms, Inlining 0.000 ms, Optimization 0.237 ms, Emission 3.794 ms, Total 4.409 ms
		Execution Time: 2222.021 ms
</details>