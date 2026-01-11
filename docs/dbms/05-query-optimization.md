# Query Optimization

## What You'll Learn

- Query execution plans and how to read them
- Cost-based vs rule-based optimization
- Query rewriting techniques
- Join algorithms and selection strategies
- Index selection for optimal query performance
- Statistics and cardinality estimation
- Query hints and forcing execution plans
- Common query performance problems

## Why This Matters

Query optimization is the difference between a query that runs in milliseconds versus one that times out after minutes. Understanding how the query optimizer works allows you to write efficient SQL, create appropriate indexes, and troubleshoot performance issues. In production systems handling millions of queries per day, even small optimizations can save significant infrastructure costs and dramatically improve user experience. Senior engineers must be able to analyze query execution plans, identify bottlenecks, and apply optimization techniques to ensure applications scale efficiently.

## Query Execution Plans

An execution plan shows how the database will execute a query.

### Reading Execution Plans

```sql
-- PostgreSQL
EXPLAIN ANALYZE
SELECT e.emp_name, d.dept_name, e.salary
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 80000
ORDER BY e.salary DESC
LIMIT 10;

/*
Limit  (cost=1234.56..1234.61 rows=10 width=45) (actual time=12.345..12.456 rows=10 loops=1)
  ->  Sort  (cost=1234.56..1245.78 rows=4500 width=45) (actual time=12.340..12.450 rows=10 loops=1)
        Sort Key: e.salary DESC
        Sort Method: top-N heapsort  Memory: 26kB
        ->  Hash Join  (cost=234.00..890.50 rows=4500 width=45) (actual time=2.345..10.234 rows=4500 loops=1)
              Hash Cond: (e.dept_id = d.dept_id)
              ->  Seq Scan on employees e  (cost=0.00..567.00 rows=4500 width=38) (actual time=0.012..5.234 rows=4500 loops=1)
                    Filter: (salary > 80000)
                    Rows Removed by Filter: 15500
              ->  Hash  (cost=120.00..120.00 rows=20 width=15) (actual time=0.234..0.234 rows=20 loops=1)
                    Buckets: 1024  Batches: 1  Memory Usage: 9kB
                    ->  Seq Scan on departments d  (cost=0.00..120.00 rows=20 width=15) (actual time=0.001..0.120 rows=20 loops=1)
Planning Time: 0.345 ms
Execution Time: 12.567 ms
*/
```

**Key Metrics:**

- **cost**: Estimated cost (arbitrary units)
- **rows**: Estimated number of rows
- **width**: Average row width in bytes
- **actual time**: Real execution time
- **loops**: Number of times the node was executed

### Common Execution Nodes

```sql
-- Sequential Scan: Full table scan
Seq Scan on table_name (cost=0.00..1234.56 rows=50000)

-- Index Scan: Using index
Index Scan using idx_name on table_name (cost=0.29..45.67 rows=100)

-- Index Only Scan: Covering index
Index Only Scan using idx_covering on table_name (cost=0.29..12.34 rows=50)

-- Bitmap Index Scan: Multiple index conditions
Bitmap Index Scan on idx_status (cost=0.00..123.45 rows=5000)

-- Nested Loop Join: For small result sets
Nested Loop (cost=1.00..234.56 rows=100)

-- Hash Join: For larger result sets
Hash Join (cost=567.00..1234.56 rows=5000)

-- Merge Join: For pre-sorted data
Merge Join (cost=234.56..890.12 rows=10000)

-- Sort: Explicit sorting
Sort (cost=456.78..567.89 rows=10000)
  Sort Key: column_name DESC
  
-- Aggregate: GROUP BY operations
HashAggregate (cost=890.12..901.23 rows=1000)
  Group Key: category_id
```

## Cost-Based Optimization

The optimizer estimates the cost of different execution plans and chooses the cheapest.

```python
class QueryOptimizer:
    """
    Simplified cost-based query optimizer
    """
    def __init__(self, statistics):
        self.statistics = statistics
    
    def optimize_query(self, query):
        """Generate and compare execution plans"""
        plans = []
        
        # Generate alternative plans
        plans.append(self.plan_nested_loop_join(query))
        plans.append(self.plan_hash_join(query))
        plans.append(self.plan_merge_join(query))
        
        # Estimate cost for each plan
        costs = [self.estimate_cost(plan) for plan in plans]
        
        # Choose plan with minimum cost
        best_index = costs.index(min(costs))
        return plans[best_index]
    
    def estimate_cost(self, plan):
        """Estimate execution cost"""
        if plan.type == 'SEQUENTIAL_SCAN':
            # Cost = (# of pages) * (cost per page)
            table_pages = self.statistics.get_table_pages(plan.table)
            return table_pages * self.SEQUENTIAL_SCAN_COST
        
        elif plan.type == 'INDEX_SCAN':
            # Cost = (height of index) + (# of matching pages)
            index_height = self.statistics.get_index_height(plan.index)
            selectivity = self.estimate_selectivity(plan.condition)
            table_pages = self.statistics.get_table_pages(plan.table)
            matching_pages = table_pages * selectivity
            return index_height + matching_pages
        
        elif plan.type == 'HASH_JOIN':
            # Cost = (read left table) + (build hash) + (read right table) + (probe hash)
            left_cost = self.estimate_cost(plan.left_input)
            right_cost = self.estimate_cost(plan.right_input)
            hash_build_cost = self.estimate_hash_build_cost(plan.left_input)
            hash_probe_cost = self.estimate_hash_probe_cost(plan.right_input)
            return left_cost + right_cost + hash_build_cost + hash_probe_cost
        
        elif plan.type == 'NESTED_LOOP':
            # Cost = (read outer table) + (rows in outer) * (cost of inner scan)
            outer_cost = self.estimate_cost(plan.outer_input)
            outer_rows = self.estimate_cardinality(plan.outer_input)
            inner_cost = self.estimate_cost(plan.inner_input)
            return outer_cost + (outer_rows * inner_cost)
    
    def estimate_selectivity(self, condition):
        """Estimate fraction of rows matching condition"""
        if condition.operator == '=':
            # Equality: 1 / distinct_values
            distinct_values = self.statistics.get_distinct_values(condition.column)
            return 1.0 / distinct_values
        
        elif condition.operator in ('>', '<', '>=', '<='):
            # Range: Use histogram
            return self.statistics.estimate_range_selectivity(condition)
        
        elif condition.operator == 'BETWEEN':
            # Range query
            low = condition.low_value
            high = condition.high_value
            return self.statistics.estimate_range_selectivity(condition.column, low, high)
```

## Join Algorithms

### Nested Loop Join

Best for small result sets or when inner relation has an index.

```python
def nested_loop_join(outer_table, inner_table, join_condition):
    """
    Nested loop join implementation
    Time Complexity: O(n * m) where n = outer rows, m = inner rows
    """
    result = []
    
    for outer_row in outer_table:  # O(n)
        for inner_row in inner_table:  # O(m)
            if join_condition(outer_row, inner_row):
                result.append(combine(outer_row, inner_row))
    
    return result

# With index on inner table
def indexed_nested_loop_join(outer_table, inner_table_index, join_condition):
    """
    Indexed nested loop join
    Time Complexity: O(n * log m)
    """
    result = []
    
    for outer_row in outer_table:  # O(n)
        # Use index to find matching inner rows
        matching_inner_rows = inner_table_index.find(outer_row.join_key)  # O(log m)
        for inner_row in matching_inner_rows:
            if join_condition(outer_row, inner_row):
                result.append(combine(outer_row, inner_row))
    
    return result
```

```sql
-- When optimizer chooses nested loop
EXPLAIN
SELECT e.emp_name, d.dept_name
FROM departments d  -- Small table (outer)
JOIN employees e ON d.dept_id = e.dept_id  -- Large table with index (inner)
WHERE d.dept_id = 10;

/*
Nested Loop  (cost=0.29..123.45 rows=50)
  ->  Seq Scan on departments d  (cost=0.00..1.25 rows=1)
        Filter: (dept_id = 10)
  ->  Index Scan using idx_emp_dept on employees e  (cost=0.29..122.20 rows=50)
        Index Cond: (dept_id = d.dept_id)
*/
```

### Hash Join

Best for large result sets when no indexes available.

```python
def hash_join(left_table, right_table, join_key):
    """
    Hash join implementation
    Time Complexity: O(n + m)
    Space Complexity: O(min(n, m)) for hash table
    """
    # Phase 1: Build hash table from smaller relation
    hash_table = {}
    smaller_table = left_table if len(left_table) < len(right_table) else right_table
    larger_table = right_table if smaller_table is left_table else left_table
    
    for row in smaller_table:
        key = row[join_key]
        if key not in hash_table:
            hash_table[key] = []
        hash_table[key].append(row)
    
    # Phase 2: Probe hash table with larger relation
    result = []
    for row in larger_table:
        key = row[join_key]
        if key in hash_table:
            for matching_row in hash_table[key]:
                result.append(combine(row, matching_row))
    
    return result
```

```sql
-- Hash join example
EXPLAIN ANALYZE
SELECT e.emp_name, d.dept_name
FROM employees e  -- Large table
JOIN departments d ON e.dept_id = d.dept_id;  -- No useful index

/*
Hash Join  (cost=125.00..2345.67 rows=10000)
  Hash Cond: (e.dept_id = d.dept_id)
  ->  Seq Scan on employees e  (cost=0.00..1890.00 rows=100000)
  ->  Hash  (cost=100.00..100.00 rows=1000)
        Buckets: 1024  Batches: 1  Memory Usage: 48kB
        ->  Seq Scan on departments d  (cost=0.00..100.00 rows=1000)
*/
```

### Merge Join

Best when both inputs are already sorted on join key.

```python
def merge_join(left_table_sorted, right_table_sorted, join_key):
    """
    Merge join implementation (assumes sorted inputs)
    Time Complexity: O(n + m)
    Space Complexity: O(1)
    """
    result = []
    left_index = 0
    right_index = 0
    
    while left_index < len(left_table_sorted) and right_index < len(right_table_sorted):
        left_key = left_table_sorted[left_index][join_key]
        right_key = right_table_sorted[right_index][join_key]
        
        if left_key == right_key:
            # Found match - handle duplicates
            right_start = right_index
            while right_index < len(right_table_sorted) and \
                  right_table_sorted[right_index][join_key] == left_key:
                result.append(combine(left_table_sorted[left_index], 
                                    right_table_sorted[right_index]))
                right_index += 1
            
            left_index += 1
            right_index = right_start
        
        elif left_key < right_key:
            left_index += 1
        else:
            right_index += 1
    
    return result
```

```sql
-- Merge join example
EXPLAIN ANALYZE
SELECT o.order_id, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
ORDER BY o.customer_id;  -- Already need sorted output

/*
Merge Join  (cost=567.89..2345.67 rows=10000)
  Merge Cond: (o.customer_id = c.customer_id)
  ->  Index Scan using idx_orders_customer on orders o  (cost=0.29..1234.56 rows=10000)
  ->  Index Scan using pk_customers on customers c  (cost=0.29..890.12 rows=5000)
*/
```

## Query Rewriting

### Predicate Pushdown

Move filters as early as possible in execution.

```sql
-- ❌ Inefficient: Filter after join
SELECT e.emp_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Engineering';

-- Optimizer rewrites to (conceptually):
SELECT e.emp_name, d.dept_name
FROM employees e
JOIN (
    SELECT dept_id, dept_name 
    FROM departments 
    WHERE dept_name = 'Engineering'  -- Filter pushed down
) d ON e.dept_id = d.dept_id;
```

### Subquery Flattening

Convert correlated subqueries to joins.

```sql
-- ❌ Inefficient: Correlated subquery
SELECT emp_name, salary
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.dept_id = e.dept_id  -- Correlated!
);

-- ✅ Optimizer rewrites to:
SELECT e.emp_name, e.salary
FROM employees e
JOIN (
    SELECT dept_id, AVG(salary) as avg_salary
    FROM employees
    GROUP BY dept_id
) dept_avg ON e.dept_id = dept_avg.dept_id
WHERE e.salary > dept_avg.avg_salary;
```

### Common Table Expression (CTE) Optimization

```sql
-- CTE materialization vs inlining
WITH high_salaries AS (
    SELECT emp_id, emp_name, salary
    FROM employees
    WHERE salary > 100000
)
SELECT hs.emp_name, d.dept_name
FROM high_salaries hs
JOIN departments d ON hs.dept_id = d.dept_id;

-- PostgreSQL 12+: Can inline CTE for better optimization
WITH high_salaries AS MATERIALIZED (  -- Force materialization
    -- Expensive computation worth caching
    SELECT emp_id, emp_name, salary
    FROM employees
    WHERE salary > 100000
)
SELECT * FROM high_salaries
UNION ALL
SELECT * FROM high_salaries WHERE salary > 150000;

-- Or prevent materialization
WITH high_salaries AS NOT MATERIALIZED (
    SELECT emp_id, emp_name, salary
    FROM employees
    WHERE salary > 100000
)
SELECT * FROM high_salaries WHERE emp_id = 1001;
```

## Optimization Techniques

### Use Covering Indexes

```sql
-- Query needs only specific columns
SELECT customer_id, order_date, total_amount
FROM orders
WHERE order_date > '2024-01-01'
ORDER BY order_date DESC;

-- Create covering index
CREATE INDEX idx_orders_covering 
ON orders(order_date DESC, customer_id, total_amount);

-- Result: Index-only scan (much faster)
```

### Avoid Function Calls on Indexed Columns

```sql
-- ❌ Function prevents index usage
SELECT * FROM employees 
WHERE YEAR(hire_date) = 2023;

-- ✅ Rewrite to use index
SELECT * FROM employees
WHERE hire_date >= '2023-01-01' 
  AND hire_date < '2024-01-01';
```

### Use EXISTS Instead of IN for Subqueries

```sql
-- ❌ IN with subquery (may not optimize well)
SELECT * FROM customers c
WHERE c.customer_id IN (
    SELECT DISTINCT customer_id 
    FROM orders 
    WHERE order_date > '2024-01-01'
);

-- ✅ EXISTS (often faster)
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 
    FROM orders o
    WHERE o.customer_id = c.customer_id 
      AND o.order_date > '2024-01-01'
);

-- ✅ Or use JOIN
SELECT DISTINCT c.*
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date > '2024-01-01';
```

### Partition Large Tables

```sql
-- Partition by range
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    total_amount DECIMAL(12, 2)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Query only accesses relevant partition
SELECT * FROM orders 
WHERE order_date BETWEEN '2024-06-01' AND '2024-06-30';
-- Optimizer uses partition pruning (scans only orders_2024)
```

## Query Hints

Force specific execution plans when optimizer makes poor choices.

```sql
-- PostgreSQL: Set parameters for specific query
SET LOCAL enable_seqscan = OFF;  -- Force index usage
SELECT * FROM large_table WHERE indexed_column = 'value';
RESET enable_seqscan;

-- MySQL: Index hints
SELECT * FROM employees 
USE INDEX (idx_dept_salary)
WHERE dept_id = 10 AND salary > 80000;

SELECT * FROM employees 
FORCE INDEX (idx_dept_salary)
WHERE dept_id = 10;

-- Oracle: Optimizer hints
SELECT /*+ INDEX(e idx_emp_dept) */ 
    e.emp_name, e.salary
FROM employees e
WHERE e.dept_id = 10;

SELECT /*+ FULL(e) */ * FROM employees e;  -- Force full table scan
```

## Best Practices

### ✅ Analyze Query Plans Regularly

```sql
-- Identify slow queries
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Analyze specific query
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders 
WHERE order_date > CURRENT_DATE - INTERVAL '30 days';
```

### ✅ Keep Statistics Up to Date

```sql
-- PostgreSQL: Update statistics
ANALYZE table_name;
ANALYZE;  -- All tables

-- Check statistics freshness
SELECT 
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY last_analyze NULLS FIRST;

-- Configure autovacuum
ALTER TABLE large_table SET (
    autovacuum_analyze_scale_factor = 0.05,  -- Analyze after 5% change
    autovacuum_analyze_threshold = 5000
);
```

## Anti-Patterns

### ❌ SELECT *

```sql
-- ❌ Fetches unnecessary data
SELECT * FROM orders;

-- ✅ Select only needed columns
SELECT order_id, customer_id, order_date, total_amount
FROM orders;
```

### ❌ N+1 Query Problem

```java
// ❌ N+1 queries
List<Order> orders = orderRepository.findAll();  // 1 query
for (Order order : orders) {
    Customer customer = customerRepository.findById(order.getCustomerId());  // N queries
    System.out.println(customer.getName());
}

// ✅ Use JOIN or batch fetch
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomers();
```

### ❌ Using OR with Different Columns

```sql
-- ❌ Cannot use single index efficiently
SELECT * FROM employees
WHERE dept_id = 10 OR salary > 100000;

-- ✅ Use UNION instead
SELECT * FROM employees WHERE dept_id = 10
UNION
SELECT * FROM employees WHERE salary > 100000;
```
