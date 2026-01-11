# Pagination, Filtering, Sorting

## What You'll Learn
How to design scalable list endpoints that handle large datasets efficiently. You'll understand different pagination strategies, their trade-offs, and how to implement stable sorting and flexible filtering.

## Why This Matters
Unpaginated lists are a performance disaster: they consume excessive memory, slow down databases with large scans, and create terrible user experience with giant payloads. Good pagination enables instant responses, predictable resource usage, and smooth infinite scroll. As datasets grow, pagination strategy affects both cost and reliability. Wrong choices (like deep offset pagination) can make endpoints unusable at scale.

## Pagination Strategies Deep Dive

### 1. Offset Pagination (Page-Based)

**How It Works**:
```
GET /orders?page=1&pageSize=20    // Items 1-20
GET /orders?page=2&pageSize=20    // Items 21-40
GET /orders?page=3&pageSize=20    // Items 41-60

SQL: SELECT * FROM orders LIMIT 20 OFFSET 40
```

**Advantages**:
- Simple to understand and implement
- Users can jump to arbitrary pages
- Total pages calculation straightforward
- Good for UIs with page numbers

**Disadvantages**:
- **Performance degrades** with deep pagination: `OFFSET 1000000` scans and skips 1M rows
- **Inconsistent results** if data changes between requests:
  ```
  Page 1: [A, B, C]
  (New item X inserted at top)
  Page 2: [C, D, E]  // C appears twice!
  ```
- Database load increases linearly with page number
- Not suitable for real-time feeds or infinite scroll

**When to Use**:
- Admin dashboards with few pages
- Reports with < 10,000 total items
- When users need page numbers
- Internal tools with small datasets

**Implementation**:
```json
GET /orders?page=2&pageSize=20

{
  "data": [...],
  "pagination": {
    "page": 2,
    "pageSize": 20,
    "total": 1543,
    "totalPages": 78,
    "hasNext": true,
    "hasPrev": true
  }
}
```

### 2. Cursor Pagination (Keyset)

**How It Works**: Use last item's values as "bookmark" for next page.

```
GET /orders?limit=20
→ Returns items + cursor (encoded last item reference)

GET /orders?limit=20&cursor=eyJpZCI6MTIzLCJjcmVhdGVkQXQiOiIuLi4ifQ
→ Returns next 20 items after that cursor

SQL: SELECT * FROM orders 
     WHERE (created_at, id) > ('2026-01-10T10:00:00Z', 123)
     ORDER BY created_at, id
     LIMIT 20
```

**Advantages**:
- **Consistent results** even if data changes (insertions/deletions don't affect pagination)
- **O(1) performance** regardless of page depth (always uses index)
- Perfect for infinite scroll / "Load More" UIs
- Works with real-time data (feeds, timelines)
- Scales to millions of records

**Disadvantages**:
- Can't jump to arbitrary pages
- No total count (expensive to calculate)
- Requires stable sort order
- Cursor format must be consistent

**When to Use**:
- Mobile apps with infinite scroll
- Activity feeds / timelines
- Large datasets (> 100K items)
- Real-time data streams
- Production-grade public APIs

**Implementation**:
```json
GET /orders?limit=20

{
  "data": [
    {"id": "ord-1", "createdAt": "2026-01-11T10:00:00Z", ...},
    {"id": "ord-2", "createdAt": "2026-01-11T09:58:00Z", ...},
    ...
  ],
  "pagination": {
    "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI2LTAxLTEwVDEwOjAwOjAwWiIsImlkIjoib3JkLTIwIn0=",
    "hasNext": true,
    "limit": 20
  }
}
```

**Cursor Format**:
```javascript
// Encode
const cursor = Buffer.from(JSON.stringify({
  createdAt: '2026-01-11T09:45:00Z',
  id: 'ord-20'
})).toString('base64url');

// Decode
const decoded = JSON.parse(Buffer.from(cursor, 'base64url').toString());

// SQL
WHERE (created_at < ? OR (created_at = ? AND id < ?))
ORDER BY created_at DESC, id DESC
```

### 3. Time-Based Pagination

**How It Works**: Use timestamps as boundaries.

```
GET /events?since=2026-01-11T00:00:00Z&until=2026-01-11T23:59:59Z&limit=100
```

**When to Use**:
- Audit logs
- Event streams
- Time-series data
- Data synchronization

**Advantages**:
- Natural for time-ordered data
- Easy to parallelize (fetch multiple time windows)
- Good for incremental sync

**Disadvantages**:
- Need to handle ties (multiple items same timestamp)
- Gaps if no data in time range

### Comparison Table

| Aspect | Offset | Cursor | Time-Based |
|--------|--------|--------|------------|
| Performance at scale | Poor | Excellent | Good |
| Arbitrary page jump | Yes | No | Partial |
| Stability under changes | Poor | Excellent | Good |
| Total count | Easy | Expensive | Expensive |
| Complexity | Low | Medium | Medium |
| Use case | Small datasets | Large, real-time | Time-series |

## Filtering & Sorting

### Filter Design Patterns

**Simple Equality**:
```
GET /orders?status=pending&customerId=cust-123

SQL: WHERE status = 'pending' AND customer_id = 'cust-123'
```

**Comparison Operators**:
```
GET /orders?amount[gte]=100&amount[lt]=1000&createdAt[gte]=2026-01-01

Means: amount >= 100 AND amount < 1000 AND createdAt >= '2026-01-01'
```

**IN Operator** (multiple values):
```
GET /orders?status=pending,processing,shipped

SQL: WHERE status IN ('pending', 'processing', 'shipped')
```

**Negation**:
```
GET /orders?status[ne]=cancelled

SQL: WHERE status != 'cancelled'
```

**Pattern Matching**:
```
GET /customers?email[like]=@example.com

SQL: WHERE email LIKE '%@example.com%'
```

### Multi-Field Sorting

**Format**: `sort=field1:direction,field2:direction`

```
GET /orders?sort=createdAt:desc,total:asc

SQL: ORDER BY created_at DESC, total ASC
```

**Default Sort**: Always define a default for consistency.
```
Default: createdAt:desc,id:desc
```

**Why Include ID**: Ensure deterministic order (createdAt may have duplicates).

### Canonical Sort for Stability

**Problem**: Without stable sort, pagination is unpredictable.

```
// If only sorting by amount (not unique):
Page 1: Order A ($100), Order B ($100), Order C ($99)
Page 2: Order B ($100), Order D ($98)  // Order B repeated!
```

**Solution**: Always include unique field as final sort:
```
Sort: amount:desc,id:asc  // ID guarantees uniqueness
```
