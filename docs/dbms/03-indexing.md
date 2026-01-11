# Indexing and Database Performance

## What You'll Learn

- Index structures and their internal workings (B-Tree, B+ Tree, Hash, Bitmap)
- When and how to create effective indexes
- Index selection strategies and query optimization
- Trade-offs between query performance and write overhead
- Covering indexes and composite indexes
- Full-text search indexes
- Spatial indexes for geographic data
- Index maintenance and monitoring

## Why This Matters

Indexes are the single most important factor in database query performance. A properly indexed database can execute queries in milliseconds that would otherwise take minutes or hours. However, indexes come with trade-offs: they consume storage space, slow down writes, and require maintenance. Understanding indexing is critical for senior engineers designing high-performance systems. Poor indexing decisions can cripple application performance, leading to timeout errors, degraded user experience, and infrastructure scaling costs that are 10-100x higher than necessary. A thorough understanding of indexing allows you to make informed trade-offs between read and write performance based on your specific workload patterns.

## Index Fundamentals

An **index** is a data structure that improves the speed of data retrieval operations on a database table at the cost of additional writes and storage space.

### How Indexes Work

Without an index, the database must scan every row (full table scan) to find matching records.

```sql
-- Without index: O(n) - Full table scan
SELECT * FROM employees WHERE emp_id = 12345;
-- Database scans all 1,000,000 rows

-- With index on emp_id: O(log n) - Index seek
CREATE INDEX idx_emp_id ON employees(emp_id);
SELECT * FROM employees WHERE emp_id = 12345;
-- Database uses index: finds record in ~20 comparisons (log₂ 1,000,000)
```

### Index Performance Comparison

| Operation | Without Index | With B-Tree Index | With Hash Index |
|-----------|--------------|-------------------|-----------------|
| Point Query (=) | O(n) | O(log n) | O(1) |
| Range Query (<, >, BETWEEN) | O(n) | O(log n + k) | O(n) |
| Sorting (ORDER BY) | O(n log n) | O(k) | O(n log n) |
| Full Table Scan | O(n) | O(n) | O(n) |

*k = number of matching rows*

## B-Tree and B+ Tree Indexes

The most common index structure in relational databases.

### B+ Tree Structure

```mermaid
graph TB
    subgraph "B+ Tree Index Structure"
        Root[Root Node<br/>50 | 100]
        
        L1A[Internal Node<br/>20 | 35]
        L1B[Internal Node<br/>70 | 85]
        L1C[Internal Node<br/>120 | 150]
        
        Leaf1[Leaf Node<br/>10,15,18]
        Leaf2[Leaf Node<br/>22,28,32]
        Leaf3[Leaf Node<br/>36,42,48]
        Leaf4[Leaf Node<br/>55,62,68]
        Leaf5[Leaf Node<br/>72,78,82]
        Leaf6[Leaf Node<br/>88,92,95]
        Leaf7[Leaf Node<br/>105,112,115]
        Leaf8[Leaf Node<br/>125,135,145]
        Leaf9[Leaf Node<br/>155,165,175]
        
        Root --> L1A
        Root --> L1B
        Root --> L1C
        
        L1A --> Leaf1
        L1A --> Leaf2
        L1A --> Leaf3
        
        L1B --> Leaf4
        L1B --> Leaf5
        L1B --> Leaf6
        
        L1C --> Leaf7
        L1C --> Leaf8
        L1C --> Leaf9
        
        Leaf1 -.->|Next| Leaf2
        Leaf2 -.->|Next| Leaf3
        Leaf3 -.->|Next| Leaf4
        Leaf4 -.->|Next| Leaf5
        Leaf5 -.->|Next| Leaf6
        Leaf6 -.->|Next| Leaf7
        Leaf7 -.->|Next| Leaf8
        Leaf8 -.->|Next| Leaf9
    end
```

**Key Properties:**

1. **Balanced Tree**: All leaf nodes are at the same level
2. **High Fan-out**: Each node contains many keys (100-1000+)
3. **Sorted Keys**: Keys are maintained in sorted order
4. **Linked Leaves**: Leaf nodes form a linked list for range scans

### B+ Tree Operations

```python
class BPlusTreeNode:
    """B+ Tree node implementation"""
    def __init__(self, is_leaf=False, order=4):
        self.is_leaf = is_leaf
        self.keys = []  # Sorted keys
        self.children = []  # Pointers to children or data records
        self.next = None  # Link to next leaf node (for range scans)
        self.order = order  # Maximum number of children
    
    def search(self, key):
        """
        Search for a key in B+ tree
        Time complexity: O(log n)
        """
        # Find position in current node
        i = 0
        while i < len(self.keys) and key > self.keys[i]:
            i += 1
        
        if self.is_leaf:
            # Leaf node: check if key exists
            if i < len(self.keys) and key == self.keys[i]:
                return self.children[i]  # Return data pointer
            return None
        else:
            # Internal node: descend to appropriate child
            return self.children[i].search(key)
    
    def range_search(self, start_key, end_key):
        """
        Range search in B+ tree
        Efficient due to sorted leaf nodes linked list
        Time complexity: O(log n + k) where k = number of results
        """
        # Find starting leaf node
        node = self._find_leaf_node(start_key)
        results = []
        
        # Scan leaf nodes using linked list
        while node is not None:
            for i, key in enumerate(node.keys):
                if start_key <= key <= end_key:
                    results.append((key, node.children[i]))
                elif key > end_key:
                    return results  # Done scanning
            
            # Move to next leaf node
            node = node.next
        
        return results
    
    def insert(self, key, value):
        """
        Insert key-value pair
        May cause node splits to maintain balance
        """
        if self.is_leaf:
            # Insert in sorted position
            i = 0
            while i < len(self.keys) and key > self.keys[i]:
                i += 1
            
            self.keys.insert(i, key)
            self.children.insert(i, value)
            
            # Check if node is overfull
            if len(self.keys) > self.order - 1:
                return self._split_leaf()
        else:
            # Find child to insert into
            i = 0
            while i < len(self.keys) and key > self.keys[i]:
                i += 1
            
            split_result = self.children[i].insert(key, value)
            
            if split_result:
                # Child was split, insert middle key into current node
                middle_key, new_child = split_result
                self.keys.insert(i, middle_key)
                self.children.insert(i + 1, new_child)
                
                # Check if current node is overfull
                if len(self.keys) > self.order - 1:
                    return self._split_internal()
        
        return None
```

### Real-World Example: Employee Database

```sql
-- Create table
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    department VARCHAR(50),
    salary DECIMAL(10, 2),
    hire_date DATE,
    manager_id INT
);

-- Insert sample data
INSERT INTO employees VALUES
(1001, 'John', 'Smith', 'john.smith@company.com', 'Engineering', 85000, '2020-01-15', NULL),
(1002, 'Jane', 'Doe', 'jane.doe@company.com', 'Sales', 75000, '2020-03-20', 1001),
(1003, 'Bob', 'Johnson', 'bob.j@company.com', 'Engineering', 90000, '2019-11-10', 1001);

-- Without index: Full table scan
EXPLAIN ANALYZE
SELECT * FROM employees WHERE salary > 80000;
/*
Seq Scan on employees (cost=0.00..180.00 rows=1000 width=100) (actual time=0.012..2.345 rows=250)
  Filter: (salary > 80000)
  Rows Removed by Filter: 750
Planning Time: 0.125 ms
Execution Time: 2.456 ms
*/

-- Create index on salary
CREATE INDEX idx_salary ON employees(salary);

-- With index: Index scan
EXPLAIN ANALYZE
SELECT * FROM employees WHERE salary > 80000;
/*
Index Scan using idx_salary on employees (cost=0.29..45.80 rows=250 width=100) (actual time=0.010..0.125 rows=250)
  Index Cond: (salary > 80000)
Planning Time: 0.095 ms
Execution Time: 0.145 ms
*/
-- 17x faster with index!
```

## Hash Indexes

Hash indexes use a hash function to map keys to bucket locations.

```python
class HashIndex:
    """
    Hash index implementation
    Excellent for equality searches, poor for range queries
    """
    def __init__(self, num_buckets=1000):
        self.num_buckets = num_buckets
        self.buckets = [[] for _ in range(num_buckets)]
    
    def _hash(self, key):
        """Hash function to determine bucket"""
        return hash(key) % self.num_buckets
    
    def insert(self, key, value):
        """
        Insert key-value pair
        Time complexity: O(1) average case
        """
        bucket_index = self._hash(key)
        bucket = self.buckets[bucket_index]
        
        # Check if key already exists
        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket[i] = (key, value)  # Update
                return
        
        # Insert new entry
        bucket.append((key, value))
    
    def search(self, key):
        """
        Search for a key
        Time complexity: O(1) average case
        """
        bucket_index = self._hash(key)
        bucket = self.buckets[bucket_index]
        
        for k, v in bucket:
            if k == key:
                return v
        
        return None
    
    def range_search(self, start_key, end_key):
        """
        Range search - inefficient in hash index
        Must scan all buckets
        Time complexity: O(n)
        """
        results = []
        
        # No choice but to scan all buckets
        for bucket in self.buckets:
            for key, value in bucket:
                if start_key <= key <= end_key:
                    results.append((key, value))
        
        return results
```

### Hash Index Usage

```sql
-- PostgreSQL: Hash index for equality searches
CREATE INDEX idx_email_hash ON employees USING HASH (email);

-- Good: Equality search (O(1))
SELECT * FROM employees WHERE email = 'john.smith@company.com';

-- Poor: Range search (no benefit from hash index)
SELECT * FROM employees WHERE email > 'a' AND email < 'z';

-- Poor: Pattern matching (no benefit)
SELECT * FROM employees WHERE email LIKE 'john%';
```

## Composite Indexes

Indexes on multiple columns, useful for queries filtering on multiple fields.

```sql
-- Create composite index
CREATE INDEX idx_dept_salary ON employees(department, salary);

-- Index structure: (department, salary)
-- Can be visualized as nested sorting:
--   Engineering -> {50000, 55000, 60000, ..., 120000}
--   Sales -> {45000, 50000, 55000, ..., 100000}
--   Marketing -> {48000, 52000, 58000, ..., 95000}

-- ✅ Uses index efficiently (leftmost prefix)
SELECT * FROM employees 
WHERE department = 'Engineering' AND salary > 80000;

-- ✅ Uses index (leftmost column)
SELECT * FROM employees WHERE department = 'Engineering';

-- ❌ Cannot use index (missing leftmost column)
SELECT * FROM employees WHERE salary > 80000;
```

### Leftmost Prefix Rule

Composite indexes can be used for queries that filter on a prefix of the indexed columns.

```sql
-- Index: (col_a, col_b, col_c)
CREATE INDEX idx_composite ON table_name(col_a, col_b, col_c);

-- ✅ Uses full index
SELECT * FROM table_name 
WHERE col_a = 1 AND col_b = 2 AND col_c = 3;

-- ✅ Uses partial index (col_a, col_b)
SELECT * FROM table_name 
WHERE col_a = 1 AND col_b = 2;

-- ✅ Uses partial index (col_a only)
SELECT * FROM table_name 
WHERE col_a = 1;

-- ❌ Cannot use index (skips col_a)
SELECT * FROM table_name 
WHERE col_b = 2 AND col_c = 3;

-- ❌ Cannot use index (skips col_a)
SELECT * FROM table_name 
WHERE col_b = 2;

-- ✅ Uses index for col_a, scans for col_c
SELECT * FROM table_name 
WHERE col_a = 1 AND col_c = 3;
```

### Composite Index Strategy

```python
def design_composite_index(query_patterns):
    """
    Design composite index based on query patterns
    """
    # Analyze query patterns
    filters = analyze_where_clauses(query_patterns)
    sorts = analyze_order_by_clauses(query_patterns)
    
    # Ordering strategy:
    # 1. Equality filters first (=)
    # 2. Range filters next (>, <, BETWEEN)
    # 3. Sort columns last (ORDER BY)
    
    equality_columns = [col for col, op in filters if op == '=']
    range_columns = [col for col, op in filters if op in ('>', '<', '>=', '<=', 'BETWEEN')]
    sort_columns = sorts
    
    # Build composite index column order
    index_columns = equality_columns + range_columns + sort_columns
    
    return index_columns

# Example
query_patterns = [
    "SELECT * FROM orders WHERE customer_id = ? AND order_date > ? ORDER BY total_amount",
    "SELECT * FROM orders WHERE customer_id = ? AND status = ? ORDER BY order_date"
]

# Result: CREATE INDEX idx_orders ON orders(customer_id, status, order_date, total_amount)
```

## Covering Indexes

A **covering index** includes all columns needed by a query, eliminating the need to access the table.

```sql
-- Query that needs emp_id, department, salary
SELECT emp_id, department, salary 
FROM employees 
WHERE department = 'Engineering' AND salary > 80000;

-- Non-covering index: Requires table lookup
CREATE INDEX idx_dept_salary ON employees(department, salary);
/*
1. Use index to find matching rows
2. For each match, lookup full row in table (expensive!)
*/

-- Covering index: No table lookup needed
CREATE INDEX idx_covering ON employees(department, salary) 
INCLUDE (emp_id);
-- Or in MySQL:
CREATE INDEX idx_covering ON employees(department, salary, emp_id);
/*
1. Use index to find matching rows
2. Index contains all needed columns - done!
*/
```

### Performance Comparison

```sql
-- Setup
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10, 2),
    status VARCHAR(20)
);

-- Index 1: Non-covering
CREATE INDEX idx_customer ON orders(customer_id);

-- Query with non-covering index
EXPLAIN ANALYZE
SELECT order_id, order_date, total_amount 
FROM orders 
WHERE customer_id = 12345;
/*
Index Scan using idx_customer on orders (cost=0.29..500.45 rows=100)
  Index Cond: (customer_id = 12345)
  Buffers: shared hit=150 (index) + shared hit=100 (table)
Total buffers: 250
*/

-- Index 2: Covering
CREATE INDEX idx_customer_covering ON orders(customer_id, order_date, total_amount, order_id);

-- Query with covering index
EXPLAIN ANALYZE
SELECT order_id, order_date, total_amount 
FROM orders 
WHERE customer_id = 12345;
/*
Index Only Scan using idx_customer_covering on orders (cost=0.29..50.45 rows=100)
  Index Cond: (customer_id = 12345)
  Buffers: shared hit=15 (index only)
Total buffers: 15
*/
-- 10x fewer disk accesses!
```

## Bitmap Indexes

Efficient for columns with low cardinality (few distinct values).

```sql
-- Bitmap index example (Oracle, PostgreSQL with extension)
CREATE BITMAP INDEX idx_status ON orders(status);

-- status column has only 5 distinct values:
-- 'PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'

-- Bitmap representation:
/*
Row | PENDING | CONFIRMED | SHIPPED | DELIVERED | CANCELLED
----|---------|-----------|---------|-----------|----------
1   |    1    |     0     |    0    |     0     |     0
2   |    0    |     1     |    0    |     0     |     0
3   |    1    |     0     |    0    |     0     |     0
4   |    0    |     0     |    1    |     0     |     0
5   |    0    |     1     |    0    |     0     |     0
*/

-- Efficient for queries with multiple conditions
SELECT * FROM orders 
WHERE status IN ('PENDING', 'CONFIRMED');
-- Uses bitmap operations: PENDING OR CONFIRMED (bitwise OR)

-- Combining multiple bitmap indexes
SELECT * FROM orders 
WHERE status = 'PENDING' AND priority = 'HIGH';
-- Uses bitmap operations: status_bitmap AND priority_bitmap (bitwise AND)
```

### Bitmap Index Implementation Concept

```python
class BitmapIndex:
    """
    Bitmap index for low-cardinality columns
    Excellent for analytical queries, poor for OLTP
    """
    def __init__(self):
        self.bitmaps = {}  # value -> bitmap
        self.row_count = 0
    
    def add_value(self, row_id, value):
        """Add a value for a specific row"""
        if value not in self.bitmaps:
            # Create new bitmap for this value
            self.bitmaps[value] = bitarray(self.row_count + 1)
            self.bitmaps[value].setall(0)
        
        # Ensure all bitmaps are same length
        if row_id >= self.row_count:
            for bitmap in self.bitmaps.values():
                bitmap.extend([0] * (row_id - self.row_count + 1))
            self.row_count = row_id + 1
        
        # Set bit for this row
        self.bitmaps[value][row_id] = 1
    
    def search(self, value):
        """Find all rows with given value - O(1) lookup"""
        if value not in self.bitmaps:
            return bitarray(self.row_count)  # Empty bitmap
        return self.bitmaps[value]
    
    def search_or(self, values):
        """Find rows matching any of the values"""
        result = bitarray(self.row_count)
        result.setall(0)
        
        for value in values:
            if value in self.bitmaps:
                result |= self.bitmaps[value]  # Bitwise OR
        
        return result
    
    def search_and(self, conditions):
        """Find rows matching all conditions"""
        result = bitarray(self.row_count)
        result.setall(1)
        
        for index, value in conditions:
            result &= index.search(value)  # Bitwise AND
        
        return result

# Usage example
status_index = BitmapIndex()
priority_index = BitmapIndex()

# Add data
status_index.add_value(0, 'PENDING')
priority_index.add_value(0, 'HIGH')

status_index.add_value(1, 'CONFIRMED')
priority_index.add_value(1, 'LOW')

status_index.add_value(2, 'PENDING')
priority_index.add_value(2, 'HIGH')

# Query: status = 'PENDING' AND priority = 'HIGH'
result = status_index.search('PENDING') & priority_index.search('HIGH')
# Returns: [True, False, True] -> rows 0 and 2
```

## Full-Text Search Indexes

Specialized indexes for text search and natural language queries.

```sql
-- PostgreSQL: Full-text search index
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    ts_vector tsvector GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''))
    ) STORED
);

-- Create GIN index for full-text search
CREATE INDEX idx_documents_fts ON documents USING GIN(ts_vector);

-- Full-text search queries
-- Simple search
SELECT doc_id, title, ts_rank(ts_vector, query) AS rank
FROM documents, to_tsquery('english', 'database & performance') AS query
WHERE ts_vector @@ query
ORDER BY rank DESC;

-- Search with stemming and synonyms
SELECT doc_id, title 
FROM documents
WHERE ts_vector @@ to_tsquery('english', 'optimizing | optimize | optimization');

-- Phrase search
SELECT doc_id, title 
FROM documents
WHERE ts_vector @@ phraseto_tsquery('english', 'database performance tuning');
```

### Full-Text Index in Application

```java
// Elasticsearch: Full-text search
@Document(indexName = "products")
public class Product {
    @Id
    private String id;
    
    @Field(type = FieldType.Text, analyzer = "standard")
    private String name;
    
    @Field(type = FieldType.Text, analyzer = "standard")
    private String description;
    
    @Field(type = FieldType.Keyword)
    private String category;
    
    @Field(type = FieldType.Double)
    private Double price;
}

// Search query
@Service
public class ProductSearchService {
    
    @Autowired
    private ElasticsearchOperations operations;
    
    public List<Product> searchProducts(String searchTerm) {
        // Multi-field search with boosting
        Query query = new NativeSearchQueryBuilder()
            .withQuery(
                QueryBuilders.multiMatchQuery(searchTerm)
                    .field("name", 3.0f)      // Boost name matches
                    .field("description", 1.0f)
                    .type(MultiMatchQueryBuilder.Type.BEST_FIELDS)
                    .fuzziness(Fuzziness.AUTO)  // Handle typos
            )
            .build();
        
        SearchHits<Product> hits = operations.search(query, Product.class);
        return hits.stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }
}
```

## Spatial Indexes

Indexes for geographic and geometric data.

```sql
-- PostgreSQL with PostGIS extension
CREATE EXTENSION postgis;

CREATE TABLE locations (
    location_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    location GEOMETRY(Point, 4326)  -- WGS84 coordinate system
);

-- Create spatial index
CREATE INDEX idx_locations_geom ON locations USING GIST(location);

-- Insert sample data
INSERT INTO locations (name, location) VALUES
('Store A', ST_SetSRID(ST_MakePoint(-122.4194, 37.7749), 4326)),  -- San Francisco
('Store B', ST_SetSRID(ST_MakePoint(-118.2437, 34.0522), 4326)),  -- Los Angeles
('Store C', ST_SetSRID(ST_MakePoint(-73.9352, 40.7306), 4326));   -- New York

-- Find locations within 50km of point
SELECT name, ST_Distance(location, ST_SetSRID(ST_MakePoint(-122.4, 37.8), 4326)) as distance_meters
FROM locations
WHERE ST_DWithin(
    location,
    ST_SetSRID(ST_MakePoint(-122.4, 37.8), 4326),
    50000  -- 50km in meters
)
ORDER BY distance_meters;

-- Find locations in bounding box
SELECT name
FROM locations
WHERE ST_Within(
    location,
    ST_MakeEnvelope(-123, 37, -122, 38, 4326)  -- Bounding box
);
```

## Index Maintenance

### Monitoring Index Usage

```sql
-- PostgreSQL: Check index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Find unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- MySQL: Check index usage
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    INDEX_NAME,
    INDEX_TYPE,
    CARDINALITY
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_NAME, SEQ_IN_INDEX;
```

### Index Fragmentation

```sql
-- Check index fragmentation (PostgreSQL)
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    round(100 * pg_relation_size(indexrelid) / pg_relation_size(tablename::regclass)) as percent_of_table
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild fragmented index
REINDEX INDEX idx_name;
REINDEX TABLE table_name;

-- MySQL: Optimize table (rebuilds indexes)
OPTIMIZE TABLE table_name;
```

## Real-World Scenarios

### Scenario 1: E-commerce Product Search

```sql
-- Product table
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    name VARCHAR(200),
    description TEXT,
    category_id INT,
    brand_id INT,
    price DECIMAL(10, 2),
    stock_quantity INT,
    rating DECIMAL(3, 2),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Index strategy for different query patterns

-- 1. Category browsing with price filter
CREATE INDEX idx_category_price ON products(category_id, price);
-- Query: SELECT * FROM products WHERE category_id = ? AND price BETWEEN ? AND ?

-- 2. Brand browsing sorted by rating
CREATE INDEX idx_brand_rating ON products(brand_id, rating DESC);
-- Query: SELECT * FROM products WHERE brand_id = ? ORDER BY rating DESC

-- 3. Search by name (full-text)
CREATE INDEX idx_product_name_fts ON products USING GIN(to_tsvector('english', name));
-- Query: SELECT * FROM products WHERE to_tsvector('english', name) @@ to_tsquery('search term')

-- 4. Recent products (time-based queries)
CREATE INDEX idx_created_at ON products(created_at DESC);
-- Query: SELECT * FROM products WHERE created_at > ? ORDER BY created_at DESC

-- 5. Covering index for product list view
CREATE INDEX idx_category_covering ON products(category_id, created_at DESC)
INCLUDE (product_id, name, price, rating, stock_quantity);
-- Query: SELECT product_id, name, price, rating, stock_quantity 
--        FROM products WHERE category_id = ? ORDER BY created_at DESC
```

### Scenario 2: Social Media Timeline

```sql
-- Posts table
CREATE TABLE posts (
    post_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT,
    created_at TIMESTAMP NOT NULL,
    visibility VARCHAR(20),
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0
);

-- Index strategy

-- 1. User timeline (most common query)
CREATE INDEX idx_user_timeline ON posts(user_id, created_at DESC)
INCLUDE (post_id, content, likes_count, comments_count);
-- Query: SELECT post_id, content, likes_count, comments_count
--        FROM posts WHERE user_id = ? ORDER BY created_at DESC LIMIT 20

-- 2. Trending posts (high engagement)
CREATE INDEX idx_trending ON posts(created_at DESC, likes_count DESC)
WHERE created_at > CURRENT_TIMESTAMP - INTERVAL '24 hours';
-- Partial index for recent posts only
-- Query: SELECT * FROM posts 
--        WHERE created_at > NOW() - INTERVAL '24 hours'
--        ORDER BY likes_count DESC

-- 3. Followers feed (join optimization)
CREATE INDEX idx_posts_created ON posts(created_at DESC);
-- Used with: SELECT p.* FROM posts p
--            JOIN followers f ON p.user_id = f.following_id
--            WHERE f.follower_id = ? ORDER BY p.created_at DESC
```

## Best Practices

### ✅ Index Selective Columns

Index columns with high cardinality (many distinct values).

```sql
-- Good: High cardinality
CREATE INDEX idx_email ON users(email);  -- Each user has unique email
CREATE INDEX idx_ssn ON employees(ssn);  -- Social Security Number is unique

-- Poor: Low cardinality
-- ❌ Don't index boolean columns
CREATE INDEX idx_is_active ON users(is_active);  -- Only 2 values: TRUE/FALSE

-- ❌ Don't index gender
CREATE INDEX idx_gender ON users(gender);  -- Only 2-3 distinct values

-- Exception: Bitmap indexes work well for low cardinality
-- But only in analytical databases, not OLTP
```

### ✅ Use Composite Indexes Wisely

Order columns by selectivity (most selective first for equality) and query patterns.

```sql
-- Query pattern: WHERE country = 'USA' AND city = 'New York' AND zip_code = '10001'

-- Good: Most to least selective
CREATE INDEX idx_location ON addresses(zip_code, city, country);
-- zip_code is most selective (unique per area)
-- city is moderately selective
-- country is least selective

-- Query can use full index
SELECT * FROM addresses 
WHERE zip_code = '10001' AND city = 'New York' AND country = 'USA';
```

### ✅ Monitor and Remove Unused Indexes

```python
def audit_indexes(database_connection):
    """
    Audit indexes and identify candidates for removal
    """
    query = """
        SELECT 
            schemaname,
            tablename,
            indexname,
            idx_scan as usage_count,
            pg_size_pretty(pg_relation_size(indexrelid)) as size
        FROM pg_stat_user_indexes
        WHERE idx_scan < 100  -- Used less than 100 times
            AND pg_relation_size(indexrelid) > 1048576  -- Larger than 1MB
        ORDER BY pg_relation_size(indexrelid) DESC;
    """
    
    unused_indexes = database_connection.execute(query)
    
    recommendations = []
    for index in unused_indexes:
        # Calculate cost vs benefit
        storage_cost = index['size']
        usage = index['usage_count']
        
        if usage == 0:
            recommendations.append({
                'index': index['indexname'],
                'action': 'DROP',
                'reason': 'Never used',
                'savings': storage_cost
            })
        elif usage < 10:
            recommendations.append({
                'index': index['indexname'],
                'action': 'REVIEW',
                'reason': f'Rarely used ({usage} times)',
                'savings': storage_cost
            })
    
    return recommendations
```

### ✅ Consider Index-Only Scans

Use covering indexes for frequently accessed column combinations.

```sql
-- Frequently executed query
SELECT product_id, name, price 
FROM products 
WHERE category_id = ? 
ORDER BY price;

-- Create covering index
CREATE INDEX idx_category_price_covering 
ON products(category_id, price, product_id, name);

-- Result: Index-only scan (no table access needed)
```

### ✅ Use Partial Indexes for Filtered Queries

Index only the subset of data that's frequently queried.

```sql
-- Most queries only access active users
CREATE INDEX idx_active_users ON users(last_login)
WHERE is_active = TRUE AND deleted_at IS NULL;

-- Much smaller than indexing all users
-- Query: SELECT * FROM users 
--        WHERE is_active = TRUE AND deleted_at IS NULL
--        ORDER BY last_login DESC;
```

## Anti-Patterns

### ❌ Over-Indexing

Creating too many indexes slows down writes and wastes space.

```sql
-- ❌ Anti-pattern: Index every column
CREATE INDEX idx_col1 ON table_name(col1);
CREATE INDEX idx_col2 ON table_name(col2);
CREATE INDEX idx_col3 ON table_name(col3);
CREATE INDEX idx_col4 ON table_name(col4);
CREATE INDEX idx_col5 ON table_name(col5);

-- Problems:
-- 1. Every INSERT updates 6 structures (table + 5 indexes)
-- 2. High storage overhead
-- 3. Index maintenance cost

-- ✅ Better: Strategic indexes based on actual queries
CREATE INDEX idx_composite ON table_name(col1, col2);  -- For common query pattern
CREATE INDEX idx_col4 ON table_name(col4);  -- For another query pattern
```

### ❌ Wrong Column Order in Composite Index

```sql
-- Query: WHERE status = 'ACTIVE' AND created_at > '2024-01-01' ORDER BY user_id

-- ❌ Bad: Wrong order
CREATE INDEX idx_bad ON users(created_at, status, user_id);
-- Cannot efficiently use index (range on created_at comes before equality on status)

-- ✅ Better: Equality first, then range, then sort
CREATE INDEX idx_good ON users(status, created_at, user_id);
```

### ❌ Function-Based Queries Breaking Index

```sql
-- ❌ Anti-pattern: Function prevents index usage
CREATE INDEX idx_email ON users(email);

-- This query won't use the index!
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- ✅ Better: Function-based index
CREATE INDEX idx_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- ✅ Or: Store normalized data
UPDATE users SET email = LOWER(email);
SELECT * FROM users WHERE email = 'user@example.com';
```

### ❌ Ignoring Index Maintenance

```sql
-- ❌ Anti-pattern: Never rebuild fragmented indexes

-- Over time, indexes become fragmented
-- Especially with lots of updates/deletes

-- ✅ Better: Regular maintenance
-- PostgreSQL
REINDEX TABLE large_table;

-- Or use pg_repack for online reindexing
pg_repack -t large_table database_name

-- MySQL
OPTIMIZE TABLE large_table;

-- Monitor fragmentation
SELECT 
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### ❌ Using SELECT * with Covering Indexes

```sql
-- ❌ Anti-pattern: Defeats covering index purpose
CREATE INDEX idx_covering ON products(category_id, price, product_id, name);

-- This won't use index-only scan (needs all columns)
SELECT * FROM products WHERE category_id = 10;

-- ✅ Better: Select only needed columns
SELECT product_id, name, price 
FROM products 
WHERE category_id = 10;
-- Uses index-only scan
```
