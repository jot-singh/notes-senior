# Database Normalization

## What You'll Learn

- Normalization forms (1NF through 5NF, BCNF)
- Functional dependencies and their role in normalization
- When to normalize and when to denormalize
- Trade-offs between normalization and performance
- Anomalies eliminated by normalization
- Real-world database design decisions
- Denormalization strategies for read-heavy workloads

## Why This Matters

Normalization is a systematic approach to organizing data in a database to reduce redundancy and improve data integrity. Poor normalization leads to update anomalies, data inconsistencies, and wasted storage space. However, over-normalization can harm query performance through excessive joins. Understanding normalization theory allows you to make informed design decisions balancing data integrity with performance requirements. Senior engineers must recognize when to apply strict normalization (for transactional systems) and when to strategically denormalize (for analytical workloads or read-heavy applications). This knowledge is crucial for designing maintainable, scalable database schemas that serve business needs efficiently.

## Functional Dependencies

A **functional dependency** (X → Y) means that for any two tuples with the same value of X, they must have the same value of Y.

### Understanding Functional Dependencies

```sql
-- Example table
CREATE TABLE employees (
    emp_id INT,
    emp_name VARCHAR(100),
    dept_id INT,
    dept_name VARCHAR(100),
    salary DECIMAL(10, 2)
);

-- Functional dependencies:
-- emp_id → emp_name, dept_id, salary (emp_id determines all other attributes)
-- emp_id → dept_id (emp_id determines department)
-- dept_id → dept_name (department ID determines department name)
```

**Types of Functional Dependencies:**

1. **Trivial FD**: X → Y where Y ⊆ X
   - Example: (emp_id, emp_name) → emp_id

2. **Non-trivial FD**: X → Y where Y ⊄ X
   - Example: emp_id → emp_name

3. **Completely Non-trivial FD**: X → Y where X ∩ Y = ∅
   - Example: emp_id → salary

### Armstrong's Axioms

Rules for deriving functional dependencies:

```python
def derive_functional_dependencies(given_fds, attributes):
    """
    Derive all functional dependencies using Armstrong's Axioms
    """
    derived_fds = set(given_fds)
    
    while True:
        new_fds = set()
        
        for fd in derived_fds:
            # Reflexivity: If Y ⊆ X, then X → Y
            for subset in get_subsets(fd.lhs):
                new_fds.add(FD(fd.lhs, subset))
            
            # Augmentation: If X → Y, then XZ → YZ
            for attr in attributes:
                new_lhs = fd.lhs | {attr}
                new_rhs = fd.rhs | {attr}
                new_fds.add(FD(new_lhs, new_rhs))
            
            # Transitivity: If X → Y and Y → Z, then X → Z
            for fd2 in derived_fds:
                if fd.rhs == fd2.lhs:
                    new_fds.add(FD(fd.lhs, fd2.rhs))
        
        # Check if we found new FDs
        if new_fds.issubset(derived_fds):
            break
        
        derived_fds.update(new_fds)
    
    return derived_fds
```

## Database Anomalies

### Insertion Anomaly

Cannot insert data without having complete information.

```sql
-- Unnormalized table
CREATE TABLE employee_project (
    emp_id INT,
    emp_name VARCHAR(100),
    dept_id INT,
    dept_name VARCHAR(100),
    project_id INT,
    project_name VARCHAR(100)
);

-- ❌ Problem: Cannot add a new department without an employee
INSERT INTO employee_project (dept_id, dept_name)
VALUES (50, 'Research');
-- ERROR: emp_id and project_id cannot be NULL

-- Must insert dummy data or leave nulls
INSERT INTO employee_project (emp_id, emp_name, dept_id, dept_name, project_id, project_name)
VALUES (NULL, NULL, 50, 'Research', NULL, NULL);
```

### Update Anomaly

Need to update multiple rows for a single logical change.

```sql
-- Update department name
-- ❌ Problem: Must update ALL rows with that department
UPDATE employee_project 
SET dept_name = 'Engineering & Development'
WHERE dept_id = 10;
-- Must update potentially hundreds of rows

-- Risk: Partial update leaves data inconsistent
UPDATE employee_project 
SET dept_name = 'Engineering & Development'
WHERE dept_id = 10 AND emp_id < 1000;
-- Now some rows have old name, some have new name!
```

### Deletion Anomaly

Deleting data unintentionally loses other information.

```sql
-- ❌ Problem: Deleting last employee in department loses department info
DELETE FROM employee_project 
WHERE emp_id = 2001;
-- If employee 2001 was the only employee in "Research" department,
-- we've now lost all information about the Research department!
```

## Normal Forms

### First Normal Form (1NF)

**Rules:**
1. Each column contains atomic (indivisible) values
2. Each column contains values of a single type
3. Each column has a unique name
4. Order doesn't matter

```sql
-- ❌ Violates 1NF: Multi-valued attribute
CREATE TABLE employees_bad (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    phone_numbers VARCHAR(500),  -- "555-1234, 555-5678, 555-9012"
    skills VARCHAR(500)           -- "Java, Python, SQL, Docker"
);

-- ✅ 1NF: Atomic values
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100)
);

CREATE TABLE employee_phones (
    emp_id INT,
    phone_number VARCHAR(20),
    phone_type VARCHAR(20),  -- 'mobile', 'home', 'work'
    PRIMARY KEY (emp_id, phone_number),
    FOREIGN KEY (emp_id) REFERENCES employees(emp_id)
);

CREATE TABLE employee_skills (
    emp_id INT,
    skill_name VARCHAR(50),
    proficiency_level VARCHAR(20),
    PRIMARY KEY (emp_id, skill_name),
    FOREIGN KEY (emp_id) REFERENCES employees(emp_id)
);
```

### Second Normal Form (2NF)

**Rules:**
1. Must be in 1NF
2. No partial dependencies (all non-key attributes fully depend on the entire primary key)

```sql
-- ❌ Violates 2NF: Partial dependencies
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100),    -- Depends only on product_id, not on (order_id, product_id)
    product_price DECIMAL(10, 2), -- Depends only on product_id
    quantity INT,                 -- Depends on full key
    discount DECIMAL(5, 2),       -- Depends on full key
    PRIMARY KEY (order_id, product_id)
);

-- Functional dependencies:
-- (order_id, product_id) → quantity, discount  (OK - full dependency)
-- product_id → product_name, product_price     (BAD - partial dependency)

-- ✅ 2NF: Remove partial dependencies
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    product_price DECIMAL(10, 2)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    discount DECIMAL(5, 2),
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### Third Normal Form (3NF)

**Rules:**
1. Must be in 2NF
2. No transitive dependencies (non-key attributes depend only on primary key, not on other non-key attributes)

```sql
-- ❌ Violates 3NF: Transitive dependencies
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    dept_id INT,
    dept_name VARCHAR(100),    -- Transitively depends on emp_id through dept_id
    dept_location VARCHAR(100) -- Transitively depends on emp_id through dept_id
);

-- Functional dependencies:
-- emp_id → dept_id         (Direct)
-- dept_id → dept_name      (Direct)
-- emp_id → dept_name       (Transitive: emp_id → dept_id → dept_name)

-- ✅ 3NF: Remove transitive dependencies
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    dept_id INT,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);

CREATE TABLE departments (
    dept_id INT PRIMARY KEY,
    dept_name VARCHAR(100),
    dept_location VARCHAR(100)
);
```

### Boyce-Codd Normal Form (BCNF)

**Rules:**
1. Must be in 3NF
2. For every functional dependency X → Y, X must be a superkey

BCNF is stricter than 3NF and handles certain edge cases.

```sql
-- ❌ Violates BCNF (but satisfies 3NF)
CREATE TABLE course_instructor (
    student_id INT,
    course_id INT,
    instructor_id INT,
    PRIMARY KEY (student_id, course_id)
);

-- Assumptions:
-- - Each student takes a course
-- - Each course is taught by one instructor
-- - Each instructor teaches only one course
--
-- Functional dependencies:
-- (student_id, course_id) → instructor_id  (Primary key determines instructor)
-- instructor_id → course_id                (Non-superkey determines part of key!)

-- ✅ BCNF: Decompose to remove anomaly
CREATE TABLE student_courses (
    student_id INT,
    course_id INT,
    PRIMARY KEY (student_id, course_id)
);

CREATE TABLE course_instructors (
    course_id INT PRIMARY KEY,
    instructor_id INT,
    FOREIGN KEY (instructor_id) REFERENCES instructors(instructor_id)
);

CREATE TABLE instructors (
    instructor_id INT PRIMARY KEY,
    instructor_name VARCHAR(100)
);
```

### Fourth Normal Form (4NF)

**Rules:**
1. Must be in BCNF
2. No multi-valued dependencies

```sql
-- ❌ Violates 4NF: Multi-valued dependencies
CREATE TABLE employee_skills_certifications (
    emp_id INT,
    skill VARCHAR(50),
    certification VARCHAR(50),
    PRIMARY KEY (emp_id, skill, certification)
);

-- Problem: Skills and certifications are independent
-- If employee has 3 skills and 2 certifications, we need 6 rows (3 × 2)
INSERT INTO employee_skills_certifications VALUES
(1001, 'Java', 'AWS Certified'),
(1001, 'Java', 'Azure Certified'),
(1001, 'Python', 'AWS Certified'),
(1001, 'Python', 'Azure Certified'),
(1001, 'SQL', 'AWS Certified'),
(1001, 'SQL', 'Azure Certified');
-- Lots of redundancy!

-- ✅ 4NF: Separate independent multi-valued facts
CREATE TABLE employee_skills (
    emp_id INT,
    skill VARCHAR(50),
    PRIMARY KEY (emp_id, skill)
);

CREATE TABLE employee_certifications (
    emp_id INT,
    certification VARCHAR(50),
    PRIMARY KEY (emp_id, certification)
);

-- Now only 5 rows total instead of 6
INSERT INTO employee_skills VALUES
(1001, 'Java'),
(1001, 'Python'),
(1001, 'SQL');

INSERT INTO employee_certifications VALUES
(1001, 'AWS Certified'),
(1001, 'Azure Certified');
```

### Fifth Normal Form (5NF)

**Rules:**
1. Must be in 4NF
2. No join dependencies

5NF deals with very specific cases where information can be reconstructed by joining smaller tables.

```sql
-- ❌ Violates 5NF: Join dependency
CREATE TABLE supplier_part_project (
    supplier_id INT,
    part_id INT,
    project_id INT,
    PRIMARY KEY (supplier_id, part_id, project_id)
);

-- Constraints:
-- - If supplier S supplies part P, and project J uses part P,
--   and supplier S supplies to project J, then S supplies P to J

-- ✅ 5NF: Decompose to eliminate join dependency
CREATE TABLE supplier_parts (
    supplier_id INT,
    part_id INT,
    PRIMARY KEY (supplier_id, part_id)
);

CREATE TABLE project_parts (
    project_id INT,
    part_id INT,
    PRIMARY KEY (project_id, part_id)
);

CREATE TABLE supplier_projects (
    supplier_id INT,
    project_id INT,
    PRIMARY KEY (supplier_id, project_id)
);

-- Valid combination can be reconstructed:
SELECT sp.supplier_id, sp.part_id, pp.project_id
FROM supplier_parts sp
JOIN project_parts pp ON sp.part_id = pp.part_id
JOIN supplier_projects spr ON sp.supplier_id = spr.supplier_id 
    AND pp.project_id = spr.project_id;
```

## Normalization Process Example

### Step-by-Step Normalization

**Original Unnormalized Table:**

```sql
CREATE TABLE student_courses_raw (
    student_id INT,
    student_name VARCHAR(100),
    student_email VARCHAR(100),
    course_list VARCHAR(500),     -- "CS101, CS102, MATH201"
    instructor_list VARCHAR(500),  -- "Dr. Smith, Dr. Jones, Dr. Brown"
    grades VARCHAR(100)            -- "A, B, A"
);
```

**Step 1: Convert to 1NF**

```sql
-- Remove multi-valued attributes
CREATE TABLE student_courses_1nf (
    student_id INT,
    student_name VARCHAR(100),
    student_email VARCHAR(100),
    course_id VARCHAR(10),
    instructor_name VARCHAR(100),
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id)
);

-- Sample data
INSERT INTO student_courses_1nf VALUES
(1001, 'John Doe', 'john@email.com', 'CS101', 'Dr. Smith', 'A'),
(1001, 'John Doe', 'john@email.com', 'CS102', 'Dr. Jones', 'B'),
(1001, 'John Doe', 'john@email.com', 'MATH201', 'Dr. Brown', 'A');
```

**Step 2: Convert to 2NF**

```sql
-- Remove partial dependencies
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100),
    student_email VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INT,
    course_id VARCHAR(10),
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(student_id)
);

-- instructor_name still needs a home - depends on course_id
CREATE TABLE courses_2nf (
    course_id VARCHAR(10) PRIMARY KEY,
    instructor_name VARCHAR(100)
);
```

**Step 3: Convert to 3NF**

```sql
-- Remove transitive dependencies
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100),
    student_email VARCHAR(100)
);

CREATE TABLE courses (
    course_id VARCHAR(10) PRIMARY KEY,
    course_name VARCHAR(100),
    instructor_id INT,
    FOREIGN KEY (instructor_id) REFERENCES instructors(instructor_id)
);

CREATE TABLE instructors (
    instructor_id INT PRIMARY KEY,
    instructor_name VARCHAR(100),
    department VARCHAR(50)
);

CREATE TABLE enrollments (
    student_id INT,
    course_id VARCHAR(10),
    grade CHAR(2),
    semester VARCHAR(20),
    PRIMARY KEY (student_id, course_id, semester),
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

## Denormalization

### When to Denormalize

Denormalization intentionally introduces redundancy to improve read performance.

```sql
-- Normalized schema
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10, 2),
    PRIMARY KEY (order_id, product_id)
);

-- Query requires join to calculate order total
SELECT o.order_id, o.order_date, SUM(oi.quantity * oi.unit_price) as total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.order_date;

-- ✅ Denormalized: Add calculated column
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(12, 2);

-- Update total when items change
CREATE TRIGGER update_order_total
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW
BEGIN
    UPDATE orders
    SET total_amount = (
        SELECT SUM(quantity * unit_price)
        FROM order_items
        WHERE order_id = NEW.order_id
    )
    WHERE order_id = NEW.order_id;
END;

-- Now simple query without join
SELECT order_id, order_date, total_amount
FROM orders;
```

### Denormalization Strategies

**1. Calculated Columns**

```sql
-- Add frequently accessed calculated values
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(200),
    base_price DECIMAL(10, 2),
    tax_rate DECIMAL(5, 4),
    -- Denormalized: Store calculated price with tax
    price_with_tax DECIMAL(10, 2) AS (base_price * (1 + tax_rate)) STORED
);
```

**2. Summary Tables**

```sql
-- Daily aggregation for reporting
CREATE TABLE daily_sales_summary (
    sale_date DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(12, 2),
    avg_order_value DECIMAL(10, 2),
    unique_customers INT,
    last_updated TIMESTAMP
);

-- Refresh summary periodically
CREATE PROCEDURE refresh_daily_summary()
BEGIN
    INSERT INTO daily_sales_summary
    SELECT 
        DATE(order_date) as sale_date,
        COUNT(*) as total_orders,
        SUM(total_amount) as total_revenue,
        AVG(total_amount) as avg_order_value,
        COUNT(DISTINCT customer_id) as unique_customers,
        CURRENT_TIMESTAMP
    FROM orders
    WHERE DATE(order_date) = CURRENT_DATE - INTERVAL 1 DAY
    ON DUPLICATE KEY UPDATE
        total_orders = VALUES(total_orders),
        total_revenue = VALUES(total_revenue),
        avg_order_value = VALUES(avg_order_value),
        unique_customers = VALUES(unique_customers),
        last_updated = VALUES(last_updated);
END;
```

**3. Materialized Views**

```sql
-- PostgreSQL: Materialized view for complex aggregations
CREATE MATERIALIZED VIEW customer_lifetime_value AS
SELECT 
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) as total_orders,
    SUM(o.total_amount) as lifetime_value,
    AVG(o.total_amount) as avg_order_value,
    MAX(o.order_date) as last_order_date,
    MIN(o.order_date) as first_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;

-- Refresh periodically
REFRESH MATERIALIZED VIEW customer_lifetime_value;

-- Or create index for faster queries
CREATE INDEX idx_clv_lifetime_value ON customer_lifetime_value(lifetime_value DESC);
```

**4. Duplicate Data Across Tables**

```java
// Denormalize for read-heavy workloads
@Entity
@Table(name = "order_snapshots")
public class OrderSnapshot {
    @Id
    private Long orderId;
    
    // Customer info duplicated in order
    private Long customerId;
    private String customerName;
    private String customerEmail;
    private String shippingAddress;  // Snapshot at order time
    
    // Product info duplicated in order items
    @OneToMany
    private List<OrderItemSnapshot> items;
    
    // Prevents issues if customer changes address
    // or product name changes after order placed
}

@Entity
@Table(name = "order_item_snapshots")
public class OrderItemSnapshot {
    @Id
    private Long id;
    
    private Long orderId;
    private Long productId;
    private String productName;      // Snapshot
    private String productSku;       // Snapshot
    private BigDecimal unitPrice;    // Snapshot
    private Integer quantity;
    
    // Historical record - immutable
}
```

## Real-World Scenarios

### Scenario 1: E-commerce Order Management

```sql
-- Fully normalized design
CREATE TABLE customers (
    customer_id BIGINT PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE addresses (
    address_id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    address_type VARCHAR(20),  -- 'SHIPPING' or 'BILLING'
    street_address VARCHAR(200),
    city VARCHAR(100),
    state VARCHAR(50),
    zip_code VARCHAR(20),
    country VARCHAR(50),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    sku VARCHAR(50) UNIQUE,
    name VARCHAR(200),
    description TEXT,
    category_id INT,
    base_price DECIMAL(10, 2),
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    shipping_address_id BIGINT,
    billing_address_id BIGINT,
    order_date TIMESTAMP,
    status VARCHAR(20),
    -- Denormalized for performance
    total_amount DECIMAL(12, 2),
    tax_amount DECIMAL(10, 2),
    shipping_cost DECIMAL(10, 2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id),
    FOREIGN KEY (billing_address_id) REFERENCES addresses(address_id)
);

CREATE TABLE order_items (
    order_item_id BIGINT PRIMARY KEY,
    order_id BIGINT,
    product_id BIGINT,
    quantity INT,
    -- Denormalized: Snapshot price at order time
    unit_price DECIMAL(10, 2),
    discount_amount DECIMAL(10, 2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### Scenario 2: Social Media Platform

```sql
-- Normalized design
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(100) UNIQUE,
    profile_photo_url VARCHAR(500),
    bio TEXT,
    created_at TIMESTAMP
);

CREATE TABLE posts (
    post_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    content TEXT,
    created_at TIMESTAMP,
    -- Denormalized for performance
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0,
    shares_count INT DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE post_likes (
    user_id BIGINT,
    post_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (post_id) REFERENCES posts(post_id)
);

CREATE TABLE comments (
    comment_id BIGINT PRIMARY KEY,
    post_id BIGINT,
    user_id BIGINT,
    parent_comment_id BIGINT,  -- For nested comments
    content TEXT,
    created_at TIMESTAMP,
    -- Denormalized
    likes_count INT DEFAULT 0,
    FOREIGN KEY (post_id) REFERENCES posts(post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_comment_id) REFERENCES comments(comment_id)
);

-- Triggers to maintain denormalized counts
CREATE TRIGGER update_post_likes_count
AFTER INSERT ON post_likes
FOR EACH ROW
BEGIN
    UPDATE posts 
    SET likes_count = likes_count + 1
    WHERE post_id = NEW.post_id;
END;

CREATE TRIGGER update_post_comments_count
AFTER INSERT ON comments
FOR EACH ROW
BEGIN
    UPDATE posts 
    SET comments_count = comments_count + 1
    WHERE post_id = NEW.post_id;
END;
```

## Best Practices

### ✅ Normalize to 3NF by Default

Most applications benefit from normalizing to 3NF.

```python
def evaluate_normalization_level(table_schema, access_patterns):
    """
    Decide appropriate normalization level
    """
    # Factors to consider
    factors = {
        'data_integrity_critical': check_data_integrity_requirements(),
        'write_heavy': analyze_write_frequency(access_patterns),
        'complex_reporting': analyze_query_complexity(access_patterns),
        'storage_constraints': check_storage_limits(),
        'real_time_requirements': check_latency_requirements()
    }
    
    # Decision logic
    if factors['data_integrity_critical']:
        # Financial, medical, legal systems
        return 'BCNF or 3NF'
    
    elif factors['write_heavy']:
        # OLTP systems
        return '3NF'
    
    elif factors['complex_reporting'] and not factors['real_time_requirements']:
        # Analytical systems
        return 'Denormalized / Star Schema'
    
    else:
        # General purpose
        return '3NF with selective denormalization'
```

### ✅ Denormalize for Read-Heavy Workloads

Strategic denormalization can significantly improve query performance.

```sql
-- Read-heavy scenario: Product catalog browsing
-- Users frequently view products with category and reviews

-- Normalized (requires joins)
SELECT 
    p.product_id,
    p.name,
    p.price,
    c.category_name,
    AVG(r.rating) as avg_rating,
    COUNT(r.review_id) as review_count
FROM products p
JOIN categories c ON p.category_id = c.category_id
LEFT JOIN reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.name, p.price, c.category_name;

-- Denormalized (optimized for reads)
ALTER TABLE products ADD COLUMN category_name VARCHAR(100);
ALTER TABLE products ADD COLUMN avg_rating DECIMAL(3, 2);
ALTER TABLE products ADD COLUMN review_count INT DEFAULT 0;

-- Maintain with triggers or batch jobs
CREATE TRIGGER maintain_product_denorm
AFTER INSERT OR UPDATE OR DELETE ON reviews
FOR EACH ROW
BEGIN
    UPDATE products
    SET 
        avg_rating = (SELECT AVG(rating) FROM reviews WHERE product_id = NEW.product_id),
        review_count = (SELECT COUNT(*) FROM reviews WHERE product_id = NEW.product_id)
    WHERE product_id = NEW.product_id;
END;

-- Now simple query
SELECT product_id, name, price, category_name, avg_rating, review_count
FROM products
WHERE category_name = 'Electronics';
```

### ✅ Use Surrogate Keys

Prefer surrogate keys (auto-increment IDs) over natural keys.

```sql
-- ❌ Natural key can change
CREATE TABLE employees (
    email VARCHAR(100) PRIMARY KEY,  -- What if employee changes email?
    emp_name VARCHAR(100),
    dept_id INT
);

-- ✅ Surrogate key is stable
CREATE TABLE employees (
    emp_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(100) UNIQUE NOT NULL,
    emp_name VARCHAR(100),
    dept_id INT
);
```

### ✅ Document Denormalization Decisions

```sql
-- Document why denormalization was chosen
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    
    -- DENORMALIZED COLUMNS (for performance)
    -- Updated by triggers when order_items change
    -- Trade-off: Faster ORDER BY total_amount queries
    --            vs. additional write overhead
    total_amount DECIMAL(12, 2),
    item_count INT,
    
    order_date TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

COMMENT ON COLUMN orders.total_amount IS 
'Denormalized: Sum of order_items. Updated by trigger. Improves query performance for ORDER BY total_amount.';
```

## Anti-Patterns

### ❌ Over-Normalization

Going beyond 3NF/BCNF without clear benefit.

```sql
-- ❌ Over-normalized: Separate table for every attribute
CREATE TABLE person_names (
    person_id INT PRIMARY KEY,
    name_value VARCHAR(100)
);

CREATE TABLE person_emails (
    person_id INT PRIMARY KEY,
    email_value VARCHAR(100)
);

CREATE TABLE person_phones (
    person_id INT PRIMARY KEY,
    phone_value VARCHAR(20)
);

-- Requires 3 joins for basic person info!
SELECT n.name_value, e.email_value, p.phone_value
FROM person_names n
JOIN person_emails e ON n.person_id = e.person_id
JOIN person_phones p ON n.person_id = p.person_id
WHERE n.person_id = 1001;

-- ✅ Better: Reasonable grouping
CREATE TABLE persons (
    person_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    phone VARCHAR(20)
);
```

### ❌ Premature Denormalization

Denormalizing before understanding access patterns.

```python
# ❌ Anti-pattern: Denormalize everything upfront
class Order(models.Model):
    order_id = models.AutoField(primary_key=True)
    customer_id = models.IntegerField()
    # Denormalized customer info (may not be needed)
    customer_name = models.CharField(max_length=100)
    customer_email = models.EmailField()
    customer_phone = models.CharField(max_length=20)
    customer_address = models.TextField()
    # ... more denormalized fields

# ✅ Better: Start normalized, denormalize based on profiling
class Order(models.Model):
    order_id = models.AutoField(primary_key=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    # Denormalize only after identifying bottlenecks
```

### ❌ Ignoring Update Anomalies

Not maintaining consistency in denormalized data.

```sql
-- ❌ Anti-pattern: Denormalized without update logic
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_email VARCHAR(100)  -- Denormalized from customers table
);

-- Customer changes email
UPDATE customers SET email = 'newemail@example.com' WHERE customer_id = 1001;

-- ❌ Old email still in orders table!
SELECT * FROM orders WHERE customer_id = 1001;
-- Shows outdated email

-- ✅ Better: Use snapshot pattern when appropriate
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_email_at_order_time VARCHAR(100)  -- Historical snapshot, intentionally not updated
);

-- Or use triggers to maintain consistency
CREATE TRIGGER sync_customer_email
AFTER UPDATE ON customers
FOR EACH ROW
BEGIN
    UPDATE orders
    SET customer_email = NEW.email
    WHERE customer_id = NEW.customer_id;
END;
```

### ❌ Using EAV (Entity-Attribute-Value) Anti-Pattern

```sql
-- ❌ Anti-pattern: EAV model (pseudo-normalization)
CREATE TABLE entities (
    entity_id INT PRIMARY KEY,
    entity_type VARCHAR(50)
);

CREATE TABLE attributes (
    attribute_id INT PRIMARY KEY,
    attribute_name VARCHAR(50)
);

CREATE TABLE entity_values (
    entity_id INT,
    attribute_id INT,
    value TEXT,
    PRIMARY KEY (entity_id, attribute_id)
);

-- Problems:
-- 1. Cannot enforce data types
-- 2. No foreign key constraints on values
-- 3. Extremely complex queries
-- 4. Poor performance

-- ✅ Better: Proper schema design
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10, 2),
    category_id INT
);
```
