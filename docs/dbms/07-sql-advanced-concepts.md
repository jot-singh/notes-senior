# SQL Advanced Concepts

## What You'll Learn

- Window functions and analytical queries
- Common Table Expressions (CTEs) and recursive queries
- Advanced JOIN techniques and query patterns
- Pivoting and unpivoting data
- JSON and array operations in SQL
- Stored procedures and functions
- Triggers and their use cases
- Dynamic SQL and query generation
- Performance optimization for complex queries

## Why This Matters

Advanced SQL techniques enable you to solve complex data problems efficiently within the database, leveraging the query optimizer and reducing data transfer. Mastering window functions, CTEs, and analytical queries allows you to write maintainable, performant code for reporting, analytics, and complex business logic. These skills are essential for senior engineers working with large datasets, building data pipelines, or optimizing application performance. Understanding when to use stored procedures vs application logic, and how to leverage database-specific features, can significantly impact system performance and maintainability.

## Window Functions

Window functions perform calculations across a set of rows related to the current row without collapsing the result set.

### Basic Window Functions

```sql
-- Ranking functions
SELECT 
    emp_name,
    department,
    salary,
    -- Rank within department (gaps for ties)
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank,
    -- Dense rank (no gaps)
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dense_rank,
    -- Row number (unique sequential)
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as row_num,
    -- Percentile rank
    PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as percentile
FROM employees;

/*
Results:
emp_name    | department | salary  | rank | dense_rank | row_num | percentile
------------|------------|---------|------|------------|---------|------------
Alice       | Engineering| 120000  | 1    | 1          | 1       | 0.0
Bob         | Engineering| 120000  | 1    | 1          | 2       | 0.0
Charlie     | Engineering| 110000  | 3    | 2          | 3       | 0.5
David       | Engineering| 100000  | 4    | 3          | 4       | 0.75
*/
```

### Aggregate Window Functions

```sql
-- Running totals and moving averages
SELECT 
    order_date,
    order_id,
    amount,
    -- Running total (cumulative sum)
    SUM(amount) OVER (ORDER BY order_date) as running_total,
    -- Moving average (last 7 days)
    AVG(amount) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7days,
    -- Cumulative average
    AVG(amount) OVER (ORDER BY order_date) as cumulative_avg,
    -- Count of orders so far
    COUNT(*) OVER (ORDER BY order_date) as orders_so_far
FROM orders
ORDER BY order_date;
```

### LAG and LEAD Functions

```sql
-- Compare current row with previous/next rows
SELECT 
    order_date,
    customer_id,
    amount,
    -- Previous order amount
    LAG(amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_amount,
    -- Next order amount
    LEAD(amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) as next_amount,
    -- Difference from previous order
    amount - LAG(amount, 1, 0) OVER (
        PARTITION BY customer_id ORDER BY order_date
    ) as amount_change,
    -- Days since last order
    order_date - LAG(order_date) OVER (
        PARTITION BY customer_id ORDER BY order_date
    ) as days_since_last_order
FROM orders
ORDER BY customer_id, order_date;
```

### FIRST_VALUE and LAST_VALUE

```sql
-- Compare with first and last values in window
SELECT 
    emp_name,
    department,
    hire_date,
    salary,
    -- First employee hired in department
    FIRST_VALUE(emp_name) OVER (
        PARTITION BY department 
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as first_hired,
    -- Last employee hired in department  
    LAST_VALUE(emp_name) OVER (
        PARTITION BY department 
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as last_hired,
    -- Difference from department average
    salary - AVG(salary) OVER (PARTITION BY department) as diff_from_avg
FROM employees;
```

### Advanced Window Frame Specifications

```sql
-- Different frame specifications
SELECT 
    order_date,
    amount,
    -- ROWS: Physical rows
    SUM(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
    ) as sum_5_rows,
    
    -- RANGE: Logical range (same values treated as peers)
    SUM(amount) OVER (
        ORDER BY order_date
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
    ) as sum_last_7_days,
    
    -- GROUPS: Groups of peers
    SUM(amount) OVER (
        ORDER BY order_date
        GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) as sum_3_groups,
    
    -- Unbounded windows
    SUM(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as cumulative_sum
FROM orders;
```

## Common Table Expressions (CTEs)

CTEs create temporary named result sets that exist for the duration of a query.

### Basic CTEs

```sql
-- Simple CTE
WITH high_earners AS (
    SELECT emp_id, emp_name, salary, department
    FROM employees
    WHERE salary > 100000
),
dept_stats AS (
    SELECT 
        department,
        COUNT(*) as emp_count,
        AVG(salary) as avg_salary
    FROM employees
    GROUP BY department
)
SELECT 
    he.emp_name,
    he.salary,
    ds.avg_salary,
    he.salary - ds.avg_salary as diff_from_avg
FROM high_earners he
JOIN dept_stats ds ON he.department = ds.department
ORDER BY diff_from_avg DESC;
```

### Multiple CTEs

```sql
-- Chain multiple CTEs
WITH monthly_sales AS (
    SELECT 
        DATE_TRUNC('month', order_date) as month,
        SUM(amount) as total_sales,
        COUNT(*) as order_count
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
),
sales_with_growth AS (
    SELECT 
        month,
        total_sales,
        order_count,
        LAG(total_sales) OVER (ORDER BY month) as prev_month_sales,
        total_sales - LAG(total_sales) OVER (ORDER BY month) as sales_growth
    FROM monthly_sales
),
categorized_months AS (
    SELECT 
        month,
        total_sales,
        sales_growth,
        CASE 
            WHEN sales_growth > 0 THEN 'Growth'
            WHEN sales_growth < 0 THEN 'Decline'
            ELSE 'Flat'
        END as trend
    FROM sales_with_growth
)
SELECT * FROM categorized_months
WHERE trend = 'Growth'
ORDER BY sales_growth DESC;
```

### Recursive CTEs

```sql
-- Employee hierarchy (organization chart)
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: Top-level employees (no manager)
    SELECT 
        emp_id,
        emp_name,
        manager_id,
        1 as level,
        emp_name as path,
        ARRAY[emp_id] as emp_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: Employees with managers
    SELECT 
        e.emp_id,
        e.emp_name,
        e.manager_id,
        eh.level + 1,
        eh.path || ' > ' || e.emp_name,
        eh.emp_path || e.emp_id
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.emp_id
)
SELECT 
    emp_id,
    REPEAT('  ', level - 1) || emp_name as indented_name,
    level,
    path
FROM employee_hierarchy
ORDER BY emp_path;

/*
Results:
emp_id | indented_name      | level | path
-------|-------------------|-------|---------------------------
1      | CEO               | 1     | CEO
2      |   VP Engineering  | 2     | CEO > VP Engineering
5      |     Senior Dev    | 3     | CEO > VP Engineering > Senior Dev
6      |     Junior Dev    | 3     | CEO > VP Engineering > Junior Dev
3      |   VP Sales        | 2     | CEO > VP Sales
7      |     Account Exec  | 3     | CEO > VP Sales > Account Exec
*/

-- Find all subordinates of a manager
WITH RECURSIVE subordinates AS (
    SELECT emp_id, emp_name, manager_id
    FROM employees
    WHERE emp_id = 2  -- VP Engineering
    
    UNION ALL
    
    SELECT e.emp_id, e.emp_name, e.manager_id
    FROM employees e
    INNER JOIN subordinates s ON e.manager_id = s.emp_id
)
SELECT * FROM subordinates;

-- Category hierarchy with depth
WITH RECURSIVE category_tree AS (
    SELECT 
        category_id,
        category_name,
        parent_category_id,
        0 as depth,
        ARRAY[category_id] as path
    FROM categories
    WHERE parent_category_id IS NULL
    
    UNION ALL
    
    SELECT 
        c.category_id,
        c.category_name,
        c.parent_category_id,
        ct.depth + 1,
        ct.path || c.category_id
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_category_id = ct.category_id
    WHERE NOT c.category_id = ANY(ct.path)  -- Prevent cycles
)
SELECT 
    category_id,
    REPEAT('  ', depth) || category_name as indented_name,
    depth,
    path
FROM category_tree
ORDER BY path;
```

### Generate Series with CTEs

```sql
-- Generate date range
WITH RECURSIVE date_range AS (
    SELECT DATE '2024-01-01' as date
    UNION ALL
    SELECT date + INTERVAL '1 day'
    FROM date_range
    WHERE date < DATE '2024-12-31'
)
SELECT 
    dr.date,
    COALESCE(SUM(o.amount), 0) as daily_sales
FROM date_range dr
LEFT JOIN orders o ON DATE(o.order_date) = dr.date
GROUP BY dr.date
ORDER BY dr.date;

-- Or use generate_series (PostgreSQL)
SELECT 
    date::DATE,
    COALESCE(SUM(o.amount), 0) as daily_sales
FROM generate_series(
    '2024-01-01'::DATE,
    '2024-12-31'::DATE,
    '1 day'::INTERVAL
) date
LEFT JOIN orders o ON DATE(o.order_date) = date
GROUP BY date
ORDER BY date;
```

## Advanced JOINs

### LATERAL JOIN

LATERAL allows a subquery to reference columns from preceding FROM items.

```sql
-- Get top 3 products per category
SELECT 
    c.category_name,
    p.product_name,
    p.price,
    p.sales_rank
FROM categories c
CROSS JOIN LATERAL (
    SELECT 
        product_name,
        price,
        ROW_NUMBER() OVER (ORDER BY sales_count DESC) as sales_rank
    FROM products
    WHERE category_id = c.category_id
    ORDER BY sales_count DESC
    LIMIT 3
) p
ORDER BY c.category_name, p.sales_rank;

-- Get latest order for each customer
SELECT 
    c.customer_id,
    c.customer_name,
    latest.order_id,
    latest.order_date,
    latest.amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_id, order_date, amount
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC
    LIMIT 1
) latest;
```

### SELF JOIN

```sql
-- Find employees earning more than their manager
SELECT 
    e.emp_name as employee,
    e.salary as emp_salary,
    m.emp_name as manager,
    m.salary as mgr_salary,
    e.salary - m.salary as salary_diff
FROM employees e
INNER JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;

-- Find gaps in sequential IDs
SELECT 
    t1.id + 1 as gap_start,
    MIN(t2.id) - 1 as gap_end
FROM table_with_ids t1
LEFT JOIN table_with_ids t2 ON t1.id + 1 = t2.id
WHERE t2.id IS NULL
GROUP BY t1.id
HAVING MIN(t2.id) IS NOT NULL;
```

### FULL OUTER JOIN with Aggregation

```sql
-- Compare data from two time periods
WITH current_month AS (
    SELECT product_id, SUM(quantity) as qty
    FROM sales
    WHERE order_date >= DATE_TRUNC('month', CURRENT_DATE)
    GROUP BY product_id
),
last_month AS (
    SELECT product_id, SUM(quantity) as qty
    FROM sales
    WHERE order_date >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
      AND order_date < DATE_TRUNC('month', CURRENT_DATE)
    GROUP BY product_id
)
SELECT 
    COALESCE(cm.product_id, lm.product_id) as product_id,
    COALESCE(cm.qty, 0) as current_month_qty,
    COALESCE(lm.qty, 0) as last_month_qty,
    COALESCE(cm.qty, 0) - COALESCE(lm.qty, 0) as change,
    CASE 
        WHEN lm.qty IS NULL THEN 'New Product'
        WHEN cm.qty IS NULL THEN 'No Sales'
        WHEN cm.qty > lm.qty THEN 'Increased'
        WHEN cm.qty < lm.qty THEN 'Decreased'
        ELSE 'Stable'
    END as trend
FROM current_month cm
FULL OUTER JOIN last_month lm ON cm.product_id = lm.product_id;
```

## Pivoting and Unpivoting

### PIVOT (Creating Columns from Rows)

```sql
-- PostgreSQL: Using CASE statements for pivoting
SELECT 
    product_id,
    product_name,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 1 THEN amount ELSE 0 END) as jan_sales,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 2 THEN amount ELSE 0 END) as feb_sales,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 3 THEN amount ELSE 0 END) as mar_sales,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 4 THEN amount ELSE 0 END) as apr_sales
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE EXTRACT(YEAR FROM order_date) = 2024
GROUP BY product_id, product_name;

-- Using crosstab (PostgreSQL with tablefunc extension)
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT *
FROM crosstab(
    'SELECT product_id, 
            TO_CHAR(order_date, ''Mon'') as month,
            SUM(amount)
     FROM orders
     WHERE EXTRACT(YEAR FROM order_date) = 2024
     GROUP BY product_id, TO_CHAR(order_date, ''Mon'')
     ORDER BY 1, 2',
    'SELECT DISTINCT TO_CHAR(order_date, ''Mon'')
     FROM orders
     WHERE EXTRACT(YEAR FROM order_date) = 2024
     ORDER BY 1'
) AS ct(product_id INT, Jan NUMERIC, Feb NUMERIC, Mar NUMERIC, Apr NUMERIC);
```

### UNPIVOT (Creating Rows from Columns)

```sql
-- Unpivot quarterly sales into rows
WITH quarterly_sales AS (
    SELECT 
        product_id,
        product_name,
        q1_sales,
        q2_sales,
        q3_sales,
        q4_sales
    FROM products
)
SELECT 
    product_id,
    product_name,
    quarter,
    sales
FROM quarterly_sales
CROSS JOIN LATERAL (
    VALUES 
        ('Q1', q1_sales),
        ('Q2', q2_sales),
        ('Q3', q3_sales),
        ('Q4', q4_sales)
) AS quarters(quarter, sales);

-- Or using UNION ALL
SELECT product_id, product_name, 'Q1' as quarter, q1_sales as sales FROM products
UNION ALL
SELECT product_id, product_name, 'Q2', q2_sales FROM products
UNION ALL
SELECT product_id, product_name, 'Q3', q3_sales FROM products
UNION ALL
SELECT product_id, product_name, 'Q4', q4_sales FROM products;
```

## JSON Operations

### Querying JSON Data

```sql
-- PostgreSQL JSON operations
CREATE TABLE events (
    event_id SERIAL PRIMARY KEY,
    event_name VARCHAR(200),
    event_data JSONB  -- JSONB for better performance and indexing
);

-- Insert JSON data
INSERT INTO events (event_name, event_data) VALUES
('user_signup', '{"user_id": 1001, "email": "user@example.com", "source": "web"}'),
('purchase', '{"user_id": 1001, "amount": 99.99, "items": ["item1", "item2"]}');

-- Query JSON fields
SELECT 
    event_name,
    event_data->>'user_id' as user_id,
    event_data->>'email' as email,
    event_data->'items' as items,
    jsonb_array_length(event_data->'items') as item_count
FROM events
WHERE event_data->>'user_id' = '1001';

-- Filter by JSON field
SELECT * FROM events
WHERE event_data @> '{"source": "web"}';

-- Extract nested JSON
SELECT 
    event_data->'user'->>'name' as user_name,
    event_data->'user'->>'email' as user_email,
    event_data->'metadata'->>'ip_address' as ip_address
FROM events;

-- Aggregate JSON arrays
SELECT 
    event_name,
    jsonb_agg(event_data->'user_id') as user_ids,
    jsonb_object_agg(
        event_data->>'user_id',
        event_data->>'email'
    ) as user_emails
FROM events
GROUP BY event_name;
```

### Creating JSON from Relational Data

```sql
-- Build JSON object from columns
SELECT 
    order_id,
    jsonb_build_object(
        'order_id', order_id,
        'customer', jsonb_build_object(
            'id', customer_id,
            'name', customer_name,
            'email', email
        ),
        'items', (
            SELECT jsonb_agg(
                jsonb_build_object(
                    'product_id', product_id,
                    'quantity', quantity,
                    'price', unit_price
                )
            )
            FROM order_items
            WHERE order_items.order_id = orders.order_id
        ),
        'total', total_amount
    ) as order_json
FROM orders
JOIN customers ON orders.customer_id = customers.customer_id;
```

## Stored Procedures and Functions

### Stored Functions

```sql
-- PostgreSQL function
CREATE OR REPLACE FUNCTION calculate_order_total(p_order_id INT)
RETURNS DECIMAL(12, 2)
LANGUAGE plpgsql
AS $$
DECLARE
    v_total DECIMAL(12, 2);
BEGIN
    SELECT SUM(quantity * unit_price)
    INTO v_total
    FROM order_items
    WHERE order_id = p_order_id;
    
    RETURN COALESCE(v_total, 0);
END;
$$;

-- Use function
SELECT 
    order_id,
    calculate_order_total(order_id) as total
FROM orders;

-- Function with multiple return values
CREATE TYPE employee_info AS (
    emp_count INT,
    avg_salary DECIMAL(10, 2),
    max_salary DECIMAL(10, 2)
);

CREATE OR REPLACE FUNCTION get_department_stats(p_dept_id INT)
RETURNS employee_info
LANGUAGE plpgsql
AS $$
DECLARE
    result employee_info;
BEGIN
    SELECT 
        COUNT(*),
        AVG(salary),
        MAX(salary)
    INTO result
    FROM employees
    WHERE dept_id = p_dept_id;
    
    RETURN result;
END;
$$;

-- Use function returning composite type
SELECT (get_department_stats(10)).*;
```

### Stored Procedures

```sql
-- PostgreSQL procedure
CREATE OR REPLACE PROCEDURE process_monthly_billing()
LANGUAGE plpgsql
AS $$
DECLARE
    v_customer_id INT;
    v_amount DECIMAL(10, 2);
    cur CURSOR FOR 
        SELECT customer_id, monthly_subscription_amount
        FROM customers
        WHERE is_active = TRUE;
BEGIN
    OPEN cur;
    
    LOOP
        FETCH cur INTO v_customer_id, v_amount;
        EXIT WHEN NOT FOUND;
        
        -- Create invoice
        INSERT INTO invoices (customer_id, amount, invoice_date, status)
        VALUES (v_customer_id, v_amount, CURRENT_DATE, 'PENDING');
        
        -- Log billing
        INSERT INTO billing_log (customer_id, action, timestamp)
        VALUES (v_customer_id, 'INVOICE_CREATED', CURRENT_TIMESTAMP);
    END LOOP;
    
    CLOSE cur;
    
    -- Commit transaction
    COMMIT;
    
    RAISE NOTICE 'Monthly billing processed for all active customers';
END;
$$;

-- Execute procedure
CALL process_monthly_billing();
```

## Triggers

```sql
-- Audit trigger: Log all changes to employees table
CREATE TABLE employee_audit (
    audit_id SERIAL PRIMARY KEY,
    emp_id INT,
    operation VARCHAR(10),
    old_values JSONB,
    new_values JSONB,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION audit_employee_changes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF (TG_OP = 'INSERT') THEN
        INSERT INTO employee_audit (emp_id, operation, new_values, changed_by)
        VALUES (NEW.emp_id, 'INSERT', row_to_json(NEW), current_user);
        RETURN NEW;
    
    ELSIF (TG_OP = 'UPDATE') THEN
        INSERT INTO employee_audit (emp_id, operation, old_values, new_values, changed_by)
        VALUES (NEW.emp_id, 'UPDATE', row_to_json(OLD), row_to_json(NEW), current_user);
        RETURN NEW;
    
    ELSIF (TG_OP = 'DELETE') THEN
        INSERT INTO employee_audit (emp_id, operation, old_values, changed_by)
        VALUES (OLD.emp_id, 'DELETE', row_to_json(OLD), current_user);
        RETURN OLD;
    END IF;
END;
$$;

CREATE TRIGGER employee_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
EXECUTE FUNCTION audit_employee_changes();

-- Validation trigger: Prevent salary decrease
CREATE OR REPLACE FUNCTION prevent_salary_decrease()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.salary < OLD.salary THEN
        RAISE EXCEPTION 'Salary cannot be decreased. Old: %, New: %', OLD.salary, NEW.salary;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER salary_decrease_prevention
BEFORE UPDATE OF salary ON employees
FOR EACH ROW
EXECUTE FUNCTION prevent_salary_decrease();
```

## Dynamic SQL

```sql
-- Dynamic query generation
CREATE OR REPLACE FUNCTION dynamic_search(
    p_table_name TEXT,
    p_search_column TEXT,
    p_search_value TEXT
)
RETURNS TABLE(result_json JSONB)
LANGUAGE plpgsql
AS $$
DECLARE
    v_query TEXT;
BEGIN
    -- Build dynamic query
    v_query := format(
        'SELECT row_to_json(t) FROM %I t WHERE %I = %L',
        p_table_name,
        p_search_column,
        p_search_value
    );
    
    -- Execute and return results
    RETURN QUERY EXECUTE v_query;
END;
$$;

-- Use dynamic function
SELECT * FROM dynamic_search('employees', 'department', 'Engineering');

-- Dynamic aggregation
CREATE OR REPLACE FUNCTION dynamic_aggregate(
    p_table TEXT,
    p_group_by TEXT,
    p_aggregate_col TEXT,
    p_aggregate_func TEXT
)
RETURNS TABLE(group_val TEXT, aggregate_val NUMERIC)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY EXECUTE format(
        'SELECT %I::TEXT, %s(%I) 
         FROM %I 
         GROUP BY %I',
        p_group_by,
        p_aggregate_func,
        p_aggregate_col,
        p_table,
        p_group_by
    );
END;
$$;

-- Use: Get average salary by department
SELECT * FROM dynamic_aggregate('employees', 'department', 'salary', 'AVG');
```

## Best Practices

### ✅ Use Window Functions for Analytical Queries

```sql
-- ✅ Efficient: Single pass with window function
SELECT 
    product_id,
    product_name,
    category,
    sales_count,
    RANK() OVER (PARTITION BY category ORDER BY sales_count DESC) as rank_in_category
FROM products
WHERE RANK() OVER (PARTITION BY category ORDER BY sales_count DESC) <= 5;

-- ❌ Inefficient: Multiple subqueries
SELECT p.*
FROM products p
WHERE (
    SELECT COUNT(*)
    FROM products p2
    WHERE p2.category = p.category
      AND p2.sales_count >= p.sales_count
) <= 5;
```

### ✅ Materialize CTEs When Reused

```sql
-- PostgreSQL 12+: Control CTE materialization
WITH expensive_calculation AS MATERIALIZED (
    -- Complex query that's used multiple times
    SELECT 
        customer_id,
        SUM(amount) as lifetime_value,
        COUNT(*) as order_count
    FROM orders
    GROUP BY customer_id
)
SELECT * FROM expensive_calculation WHERE lifetime_value > 10000
UNION ALL
SELECT * FROM expensive_calculation WHERE order_count > 50;
```

### ✅ Use Appropriate JSON Functions

```sql
-- ✅ Use containment operator for existence checks
SELECT * FROM events
WHERE event_data @> '{"status": "completed"}';

-- ❌ Slower: Extracting and comparing
SELECT * FROM events
WHERE event_data->>'status' = 'completed';

-- ✅ Create GIN index for JSON queries
CREATE INDEX idx_events_data ON events USING GIN (event_data);
```

## Anti-Patterns

### ❌ Overusing Triggers

```sql
-- ❌ Anti-pattern: Complex business logic in triggers
CREATE TRIGGER complex_business_logic
AFTER INSERT ON orders
FOR EACH ROW
EXECUTE FUNCTION process_order_complex();
-- Problems: Hard to debug, hidden logic, performance impact

-- ✅ Better: Handle in application layer or explicit stored procedure
CALL process_order(order_id);
```

### ❌ Recursive CTE Without Termination

```sql
-- ❌ Anti-pattern: No cycle detection
WITH RECURSIVE infinite_loop AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM infinite_loop  -- Never terminates!
)
SELECT * FROM infinite_loop;

-- ✅ Better: Add termination condition
WITH RECURSIVE controlled_recursion AS (
    SELECT 1 as n, 1 as depth
    UNION ALL
    SELECT n + 1, depth + 1 
    FROM controlled_recursion
    WHERE depth < 100  -- Limit recursion depth
)
SELECT * FROM controlled_recursion;
```

### ❌ Dynamic SQL Without Proper Escaping

```sql
-- ❌ SQL Injection vulnerability
CREATE FUNCTION unsafe_search(search_term TEXT)
RETURNS TABLE(result TEXT)
AS $$
BEGIN
    RETURN QUERY EXECUTE 
        'SELECT name FROM users WHERE name = ''' || search_term || '''';
    -- Vulnerable to: search_term = "'; DROP TABLE users; --"
END;
$$ LANGUAGE plpgsql;

-- ✅ Better: Use parameterized queries
CREATE FUNCTION safe_search(search_term TEXT)
RETURNS TABLE(result TEXT)
AS $$
BEGIN
    RETURN QUERY EXECUTE 
        'SELECT name FROM users WHERE name = $1'
    USING search_term;
END;
$$ LANGUAGE plpgsql;
```
