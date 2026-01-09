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

---

### 2. Write a SQL Query to Find the top customers by total spending

```sql

EXPLAIN ANALYZE SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o
ON o.customer_id = c.customer_id
GROUP BY
    c.customer_id,
    c.first_name,
    c.last_name
ORDER BY
    total_spent DESC;
```

#### Query Execution Plan Before Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Sort  (cost=2601563.16..2614062.75 rows=4999836 width=65) (actual time=9877.241..10142.455 rows=2475000 loops=1)
		  Sort Key: (sum(o.total_amount)) DESC
		  Sort Method: external merge  Disk: 120080kB
		  ->  HashAggregate  (cost=1417913.85..1635101.17 rows=4999836 width=65) (actual time=6650.605..8847.050 rows=2475000 loops=1)
				Group Key: c.customer_id
				Planned Partitions: 256  Batches: 257  Memory Usage: 8209kB  Disk Usage: 769224kB
				->  Hash Join  (cost=218125.31..551653.35 rows=9900120 width=38) (actual time=970.940..4336.534 rows=9900000 loops=1)
					  Hash Cond: (o.customer_id = c.customer_id)
					  ->  Seq Scan on orders o  (cost=0.00..171796.20 rows=9900120 width=13) (actual time=43.545..553.536 rows=9900000 loops=1)
					  ->  Hash  (cost=116565.36..116565.36 rows=4999836 width=33) (actual time=926.543..926.544 rows=5000000 loops=1)
							Buckets: 131072  Batches: 64  Memory Usage: 6103kB
							->  Seq Scan on customers c  (cost=0.00..116565.36 rows=4999836 width=33) (actual time=0.098..326.173 rows=5000000 loops=1)
		Planning Time: 0.212 ms
		JIT:
		  Functions: 17
		  Options: Inlining true, Optimization true, Expressions true, Deforming true
		  Timing: Generation 0.866 ms, Inlining 9.450 ms, Optimization 34.816 ms, Emission 25.613 ms, Total 70.745 ms
		Execution Time: 10258.384 ms
</details>

#### What to Optimize?

Bottleneck: JOIN on orders.customer_id and SUM aggregation.

1. Index on orders.customer_id

	CREATE INDEX idx_orders_customer_id ON orders(customer_id);

2. Index on orders(total_amount) if filtering is needed

3. Optional: Precompute totals in a materialized view if table is huge:
	
	CREATE MATERIALIZED VIEW mv_customer_spending AS
	SELECT customer_id, SUM(total_amount) AS total_spent
	FROM orders
	GROUP BY customer_id;

#### Query Execution Plan After Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Sort  (cost=1908258.20..1920757.79 rows=4999836 width=65) (actual time=5293.470..5555.028 rows=2475000 loops=1)
		  Sort Key: (sum(o.total_amount)) DESC
		  Sort Method: external merge  Disk: 120072kB
		  ->  GroupAggregate  (cost=44.89..941796.21 rows=4999836 width=65) (actual time=45.454..4393.561 rows=2475000 loops=1)
				Group Key: c.customer_id
				->  Merge Join  (cost=44.89..829798.26 rows=9900000 width=38) (actual time=45.423..3059.280 rows=9900000 loops=1)
					  Merge Cond: (c.customer_id = o.customer_id)
					  ->  Index Scan using customers_pkey on customers c  (cost=0.43..196415.97 rows=4999836 width=33) (actual time=0.012..494.243 rows=4998991 loops=1)
					  ->  Index Scan using idx_orders_customer_id on orders o  (cost=0.43..497173.65 rows=9900000 width=13) (actual time=0.025..1657.436 rows=9900000 loops=1)
		Planning Time: 0.275 ms
		JIT:
		  Functions: 10
		  Options: Inlining true, Optimization true, Expressions true, Deforming true
		  Timing: Generation 0.359 ms, Inlining 3.794 ms, Optimization 22.500 ms, Emission 19.080 ms, Total 45.734 ms
		Execution Time: 5607.273 ms
</details>

---

### 3. Write a SQL Query to Retrieve the most recent 1000 orders with customer information

```sql

EXPLAIN ANALYZE SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    o.order_id,
    o.order_date
FROM customers c
JOIN orders o
ON c.customer_id = o.customer_id
ORDER BY o.order_date DESC
LIMIT 1000;

```

#### Query Execution Plan Before Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Limit  (cost=546374.92..546491.60 rows=1000 width=49) (actual time=3297.895..3370.839 rows=1000 loops=1)
		  ->  Gather Merge  (cost=546374.92..1508942.13 rows=8250000 width=49) (actual time=3263.542..3336.447 rows=1000 loops=1)
				Workers Planned: 2
				Workers Launched: 2
				->  Sort  (cost=545374.90..555687.40 rows=4125000 width=49) (actual time=3232.239..3232.304 rows=897 loops=3)
					  Sort Key: o.order_date DESC
					  Sort Method: top-N heapsort  Memory: 163kB
					  Worker 0:  Sort Method: top-N heapsort  Memory: 164kB
					  Worker 1:  Sort Method: top-N heapsort  Memory: 166kB
					  ->  Parallel Hash Join  (cost=129716.46..319205.60 rows=4125000 width=49) (actual time=2034.289..2780.491 rows=3300000 loops=3)
							Hash Cond: (o.customer_id = c.customer_id)
							->  Parallel Seq Scan on orders o  (cost=0.00..114045.00 rows=4125000 width=24) (actual time=0.390..488.608 rows=3300000 loops=3)
							->  Parallel Hash  (cost=87399.65..87399.65 rows=2083265 width=33) (actual time=1012.769..1012.770 rows=1666667 loops=3)
								  Buckets: 131072  Batches: 64  Memory Usage: 6592kB
								  ->  Parallel Seq Scan on customers c  (cost=0.00..87399.65 rows=2083265 width=33) (actual time=43.977..694.812 rows=1666667 loops=3)
		Planning Time: 5.455 ms
		JIT:
		  Functions: 31
		  Options: Inlining true, Optimization true, Expressions true, Deforming true
		  Timing: Generation 2.104 ms, Inlining 80.234 ms, Optimization 46.535 ms, Emission 37.991 ms, Total 166.864 ms
		Execution Time: 3371.878 ms
</details>

#### What to Optimize?

Bottleneck: ORDER BY o.order_date DESC,
	If orders is huge, PostgreSQL sorts all rows before applying LIMIT.

1. Index on orders(order_date DESC)

	CREATE INDEX idx_orders_order_date_desc ON orders(order_date DESC);

Supports index-only scan + fast LIMIT.

2. Optional composite index for JOIN

	CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date DESC);
	
Speeds up JOIN + ORDER BY if needed.

#### Query Execution Plan After Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Limit  (cost=0.87..509.04 rows=1000 width=49) (actual time=0.018..1.494 rows=1000 loops=1)
		  ->  Nested Loop  (cost=0.87..5030926.34 rows=9900000 width=49) (actual time=0.017..1.433 rows=1000 loops=1)
				->  Index Scan using idx_orders_order_date_desc on orders o  (cost=0.43..254810.43 rows=9900000 width=24) (actual time=0.010..0.160 rows=1000 loops=1)
				->  Index Scan using customers_pkey on customers c  (cost=0.43..0.48 rows=1 width=33) (actual time=0.001..0.001 rows=1 loops=1000)
					  Index Cond: (customer_id = o.customer_id)
		Planning Time: 0.289 ms
		Execution Time: 1.542 ms
</details>

---

### 4. Write a SQL Query to List products with stock quantity of less than 10

```sql

EXPLAIN ANALYZE SELECT
    product_id,
    name,
    stock_quantity
FROM products
WHERE stock_quantity < 10
ORDER BY product_id;

```

#### Query Execution Plan Before Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Gather Merge  (cost=86538.94..95074.17 rows=73154 width=27) (actual time=126.612..137.561 rows=90000 loops=1)
		  Workers Planned: 2
		  Workers Launched: 2
		  ->  Sort  (cost=85538.91..85630.36 rows=36577 width=27) (actual time=115.323..116.182 rows=30000 loops=3)
				Sort Key: product_id
				Sort Method: quicksort  Memory: 3383kB
				Worker 0:  Sort Method: quicksort  Memory: 2523kB
				Worker 1:  Sort Method: quicksort  Memory: 1782kB
				->  Parallel Seq Scan on products  (cost=0.00..82766.62 rows=36577 width=27) (actual time=0.056..111.370 rows=30000 loops=3)
					  Filter: (stock_quantity < 10)
					  Rows Removed by Filter: 1635000
		Planning Time: 0.054 ms
		Execution Time: 139.342 ms
</details>

#### What to Optimize?

Bottleneck: Full table scan on products,
	Simple numeric filter, ideal for index.

1. -- General index

	CREATE INDEX idx_products_stock_qty ON products(stock_quantity);

2. -- Better: Partial index for low stock

	CREATE INDEX idx_products_low_stock ON products(product_id, stock_quantity)

WHERE stock_quantity < 10;


#### Query Execution Plan After Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Limit  (cost=0.87..509.04 rows=1000 width=49) (actual time=0.024..0.881 rows=1000 loops=1)
		  ->  Nested Loop  (cost=0.87..5030926.34 rows=9900000 width=49) (actual time=0.023..0.842 rows=1000 loops=1)
				->  Index Scan using idx_orders_order_date_desc on orders o  (cost=0.43..254810.43 rows=9900000 width=24) (actual time=0.009..0.088 rows=1000 loops=1)
				->  Index Scan using customers_pkey on customers c  (cost=0.43..0.48 rows=1 width=33) (actual time=0.001..0.001 rows=1 loops=1000)
					  Index Cond: (customer_id = o.customer_id)
		Planning Time: 0.218 ms
		Execution Time: 0.954 ms
</details>

--- 

### 5. Write a SQL Query to Calculate the revenue generated from each category.

```sql

EXPLAIN ANALYZE SELECT
    c.category_id,
    c.category_name,
    SUM(od.quantity * od.unit_price) AS total_revenue
FROM categories c
JOIN products p
ON p.category_id = c.category_id
JOIN order_details od
ON od.product_id = p.product_id
GROUP BY
    c.category_id,
    c.category_name
ORDER BY
    total_revenue DESC;

```

#### Query Execution Plan Before Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Sort  (cost=1640805.32..1641055.32 rows=100000 width=54) (actual time=15159.092..15274.609 rows=49450 loops=1)
		  Sort Key: (sum(((od.quantity)::numeric * od.unit_price))) DESC
		  Sort Method: quicksort  Memory: 3934kB
		  ->  Finalize GroupAggregate  (cost=1602996.03..1629081.00 rows=100000 width=54) (actual time=15090.285..15257.488 rows=49450 loops=1)
				Group Key: c.category_id
				->  Gather Merge  (cost=1602996.03..1626331.00 rows=200000 width=54) (actual time=15090.237..15221.451 rows=148350 loops=1)
					  Workers Planned: 2
					  Workers Launched: 2
					  ->  Sort  (cost=1601996.01..1602246.01 rows=100000 width=54) (actual time=15047.505..15051.430 rows=49450 loops=3)
							Sort Key: c.category_id
							Sort Method: external merge  Disk: 4736kB
							Worker 0:  Sort Method: external merge  Disk: 4736kB
							Worker 1:  Sort Method: external merge  Disk: 4736kB
							->  Partial HashAggregate  (cost=1453675.83..1590271.69 rows=100000 width=54) (actual time=13814.645..15024.219 rows=49450 loops=3)
								  Group Key: c.category_id
								  Planned Partitions: 8  Batches: 9  Memory Usage: 8273kB  Disk Usage: 256528kB
								  Worker 0:  Batches: 9  Memory Usage: 8273kB  Disk Usage: 255608kB
								  Worker 1:  Batches: 9  Memory Usage: 8273kB  Disk Usage: 251432kB
								  ->  Hash Join  (cost=116629.12..615304.89 rows=9899583 width=31) (actual time=6315.839..10324.506 rows=7958605 loops=3)
										Hash Cond: (p.category_id = c.category_id)
										->  Parallel Hash Join  (cost=113742.12..586430.39 rows=9899583 width=17) (actual time=6192.732..8072.680 rows=7958605 loops=3)
											  Hash Cond: (od.product_id = p.product_id)
											  ->  Parallel Seq Scan on order_details od  (cost=0.00..320526.83 rows=9899583 width=17) (actual time=0.401..3814.909 rows=7958605 loops=3)
											  ->  Parallel Hash  (cost=77563.50..77563.50 rows=2081250 width=16) (actual time=902.616..902.617 rows=1665000 loops=3)
													Buckets: 262144  Batches: 64  Memory Usage: 5760kB
													->  Parallel Seq Scan on products p  (cost=0.00..77563.50 rows=2081250 width=16) (actual time=0.467..633.234 rows=1665000 loops=3)
										->  Hash  (cost=1637.00..1637.00 rows=100000 width=22) (actual time=122.583..122.584 rows=100000 loops=3)
											  Buckets: 131072  Batches: 1  Memory Usage: 6385kB
											  ->  Seq Scan on categories c  (cost=0.00..1637.00 rows=100000 width=22) (actual time=0.191..7.420 rows=100000 loops=3)
		Planning Time: 9.006 ms
		JIT:
		  Functions: 75
		  Options: Inlining true, Optimization true, Expressions true, Deforming true
		  Timing: Generation 10.081 ms, Inlining 171.305 ms, Optimization 140.484 ms, Emission 110.970 ms, Total 432.840 ms
		Execution Time: 15301.469 ms
</details>

#### What to Optimize?

Bottleneck: JOINs across large tables (products + order_details),
	SUM multiplication is fine; main gain comes from indexes.

1. Index on products.category_id (already added in Query 5)

	CREATE INDEX idx_order_details_product_id ON order_details(product_id);

2. Index on order_details.product_id

3. Optional: Materialized view for frequent revenue reporting:
	
	CREATE MATERIALIZED VIEW mv_category_revenue AS
	SELECT
		p.category_id,
		SUM(od.quantity * od.unit_price) AS total_revenue
	FROM products p
	JOIN order_details od
	ON od.product_id = p.product_id
	GROUP BY p.category_id;

#### Query Execution Plan After Optimization
<details> 
	<summary> EXPLAIN ANALYZE output </summary>
		Sort  (cost=1646905.13..1647155.13 rows=100000 width=54) (actual time=11392.803..12314.987 rows=49450 loops=1)
		  Sort Key: (sum(((od.quantity)::numeric * od.unit_price))) DESC
		  Sort Method: quicksort  Memory: 3934kB
		  ->  Finalize GroupAggregate  (cost=1609095.85..1635180.81 rows=100000 width=54) (actual time=11318.689..12300.727 rows=49450 loops=1)
				Group Key: c.category_id
				->  Gather Merge  (cost=1609095.85..1632430.81 rows=200000 width=54) (actual time=11318.643..12261.597 rows=148350 loops=1)
					  Workers Planned: 2
					  Workers Launched: 2
					  ->  Sort  (cost=1608095.83..1608345.83 rows=100000 width=54) (actual time=11265.234..11269.227 rows=49450 loops=3)
							Sort Key: c.category_id
							Sort Method: external merge  Disk: 4736kB
							Worker 0:  Sort Method: external merge  Disk: 4736kB
							Worker 1:  Sort Method: external merge  Disk: 4736kB
							->  Partial HashAggregate  (cost=1459110.18..1596371.51 rows=100000 width=54) (actual time=9841.964..11244.219 rows=49450 loops=3)
								  Group Key: c.category_id
								  Planned Partitions: 8  Batches: 9  Memory Usage: 8273kB  Disk Usage: 252496kB
								  Worker 0:  Batches: 9  Memory Usage: 8273kB  Disk Usage: 255760kB
								  Worker 1:  Batches: 9  Memory Usage: 8273kB  Disk Usage: 257544kB
								  ->  Hash Join  (cost=116629.12..616617.16 rows=9948257 width=31) (actual time=2401.697..6508.422 rows=7958605 loops=3)
										Hash Cond: (p.category_id = c.category_id)
										->  Parallel Hash Join  (cost=113742.12..587614.89 rows=9948257 width=17) (actual time=2280.629..4207.141 rows=7958605 loops=3)
											  Hash Cond: (od.product_id = p.product_id)
											  ->  Parallel Seq Scan on order_details od  (cost=0.00..321013.57 rows=9948257 width=17) (actual time=0.417..591.913 rows=7958605 loops=3)
											  ->  Parallel Hash  (cost=77563.50..77563.50 rows=2081250 width=16) (actual time=395.125..395.127 rows=1665000 loops=3)
													Buckets: 262144  Batches: 64  Memory Usage: 5760kB
													->  Parallel Seq Scan on products p  (cost=0.00..77563.50 rows=2081250 width=16) (actual time=0.103..150.561 rows=1665000 loops=3)
										->  Hash  (cost=1637.00..1637.00 rows=100000 width=22) (actual time=120.728..120.729 rows=100000 loops=3)
											  Buckets: 131072  Batches: 1  Memory Usage: 6385kB
											  ->  Seq Scan on categories c  (cost=0.00..1637.00 rows=100000 width=22) (actual time=0.020..4.847 rows=100000 loops=3)
		Planning Time: 0.679 ms
		JIT:
		  Functions: 75
		  Options: Inlining true, Optimization true, Expressions true, Deforming true
		  Timing: Generation 3.414 ms, Inlining 152.894 ms, Optimization 172.536 ms, Emission 125.119 ms, Total 453.963 ms
		Execution Time: 12334.769 ms

</details>

