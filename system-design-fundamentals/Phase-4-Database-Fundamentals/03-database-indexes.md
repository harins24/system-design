# Database Indexes

**Phase 4 — Database Fundamentals | Topic 3 of 9**

---

## What is a Database Index?

An index is a separate data structure that allows the database to find rows quickly without scanning every row in a table.

```
Without index:
SELECT * FROM orders WHERE user_id = 'user_123';

Database scans every row in orders table:
Row 1:  user_id = 'user_456' → no
Row 2:  user_id = 'user_789' → no
Row 3:  user_id = 'user_123' → yes ✅
Row 4:  user_id = 'user_321' → no
...
Row 10,000,000: done

Full table scan: O(n) — scans 10 million rows to find 5

With index on user_id:
Database looks up 'user_123' in index → gets row locations
Fetches exactly those 5 rows → done

Index lookup: O(log n) → milliseconds instead of seconds
```

An index is like the index at the back of a book. Instead of reading every page to find "consistency," you look it up in the index, get page 247, and go directly there.

---

## How Indexes Work Internally — B-Tree

The most common index structure is the **B-Tree (Balanced Tree)**.

```
B-Tree for index on user_id:

                    [500]
                   /     \
           [250]           [750]
          /     \         /     \
      [125]   [375]   [625]   [875]
      / \     / \     / \     / \
   [..][..][..][..][..][..][..][..]
   (leaf nodes contain actual row pointers)

Properties:
  - Always balanced → consistent depth
  - Each lookup: O(log n) comparisons
  - Leaf nodes linked → efficient range scans
  - Self-balancing on insert/delete

Finding user_id = 'user_123':
  Root: 123 < 500 → go left
  Node: 123 < 250 → go left
  Node: 123 < 125 → go left
  Leaf: found! → row pointer → fetch row
  3-4 comparisons to find among millions of rows
```

B-Trees work well for:
- Exact match: `WHERE id = 123`
- Range queries: `WHERE age BETWEEN 25 AND 35`
- Sorting: `ORDER BY created_at`
- Prefix match: `WHERE name LIKE 'Har%'`

---

## Types of Indexes

### Primary Index (Clustered Index)

The table itself is stored in the order of the primary key. There's only one per table.

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,  -- clustered index in MySQL InnoDB
    ...
);
```

```
In MySQL InnoDB:
  Table rows physically stored sorted by id
  PRIMARY KEY lookup = direct disk location fetch
  Extremely fast for PK lookups
  Range scans on PK = sequential disk reads (fast)

In PostgreSQL:
  Heap storage (rows in insertion order)
  Primary key creates a B-tree index pointing to heap
  No physical ordering by default
  (can use CLUSTER command to reorder but not maintained)
```

---

### Secondary Index (Non-Clustered)

Additional indexes on non-primary-key columns. A table can have many.

```sql
-- Add index on user_id for fast order lookups
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Add index on created_at for time-based queries
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Add index on (user_id, status) for filtered queries
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

```
Secondary index structure:
B-Tree of indexed values → pointer to primary key (PostgreSQL)
                        → pointer to row location (MySQL)

Lookup flow (PostgreSQL):
  Find user_id='user_123' in secondary index
  → get primary key (order_id)
  → look up order_id in primary index
  → fetch actual row
  (two index lookups = index scan + heap fetch)
```

---

### Composite Index

Index on multiple columns. **Column order matters critically.**

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

This index **helps**:
```sql
WHERE user_id = 'user_123'                        ✅ leftmost prefix
WHERE user_id = 'user_123' AND status = 'PENDING' ✅ full composite
```

This index does **NOT** help:
```sql
WHERE status = 'PENDING'                          ❌ not leftmost prefix
```

**The leftmost prefix rule:** A composite index on (A, B, C) can be used for queries on A, (A,B), or (A,B,C) — but NOT B alone or C alone.

Think of it like a phone book sorted by (last_name, first_name) — useful for finding "Kumar" or "Kumar, Hari" but useless for finding everyone named "Hari" across all last names.

---

### Unique Index

Enforces uniqueness as a constraint while providing index performance.

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
-- or
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE(email);

Both:
  ✅ Fast lookup by email
  ✅ Database rejects duplicate emails
  Use for: email, username, SSN, order reference numbers
```

---

### Partial Index

Index only a subset of rows matching a condition. Smaller, faster.

```sql
-- Only index pending orders (not the millions of completed ones)
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'PENDING';

-- Only index active users
CREATE INDEX idx_users_active_email ON users(email)
WHERE is_active = true;
```

```
Benefits:
  Much smaller index → fits in memory → faster
  Writes to non-indexed rows don't update index
  Perfect when you query a specific subset heavily
```

---

### Covering Index (Index Only Scan)

An index that contains all columns needed by a query — database never needs to touch the actual table.

```sql
-- Query: Get user's pending order IDs and totals
SELECT id, total FROM orders
WHERE user_id = 'user_123' AND status = 'PENDING';

-- Covering index includes all needed columns
CREATE INDEX idx_orders_covering
ON orders(user_id, status, id, total);
--                          ↑   ↑
--                     included columns (not filter, just returned)

-- PostgreSQL: index-only scan
-- All needed data is IN the index → never reads actual table rows
-- Fastest possible query execution
```

---

### Hash Index

Uses a hash table instead of B-Tree. Only useful for exact equality.

```sql
-- PostgreSQL hash index
CREATE INDEX idx_sessions_token ON sessions USING HASH(token);

Perfect for: WHERE token = 'abc123'  (exact match only)
Useless for: WHERE token LIKE 'abc%' (no range support)
```

---

### Full-Text Index

Specialized index for searching within text content.

```sql
-- PostgreSQL full-text search
CREATE INDEX idx_products_search
ON products USING GIN(to_tsvector('english', name || ' ' || description));

-- Query
SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || description)
      @@ to_tsquery('english', 'laptop & stand');
```

Better option for production: **Elasticsearch** — dedicated full-text search engine with relevance scoring, fuzzy matching, facets. Sync from PostgreSQL via CDC or Kafka.

---

## The Cost of Indexes

Indexes aren't free. Understanding the cost prevents over-indexing.

```
Read performance:  indexes make reads faster ✅
Write performance: indexes make writes slower ❌

Every INSERT:
  Write row to table
  Update EVERY index on the table
  4 indexes = 5 write operations instead of 1

Every UPDATE to indexed column:
  Update row
  Remove old entry from index
  Add new entry to index

Every DELETE:
  Remove row
  Remove entry from EVERY index

Storage:
  Each index takes disk space
  Large tables with many indexes: index storage > table storage
  Index on varchar(255) email for 100M users → gigabytes

Rule of thumb:
  Indexes on read-heavy tables → add freely (few writes)
  Indexes on write-heavy tables → add carefully (each hurts write throughput)
  Indexes on high-cardinality columns → effective
  Indexes on low-cardinality columns → often useless
```

---

## Index Cardinality

Cardinality = number of distinct values. Critically affects index usefulness.

```
High cardinality (good for indexing):
  user_id:     10 million distinct values → highly selective
  email:       10 million distinct values → highly selective
  order_id:    uniquely identifies each row → perfect

Low cardinality (poor for indexing):
  status:      3 values (PENDING, SHIPPED, DELIVERED) → poor
  is_active:   2 values (true/false) → terrible
  country:     195 values → maybe useful if distribution is skewed

Why low cardinality is bad:
  Index on status='PENDING': 33% of rows match
  Database thinks: "faster to just scan the table"
  Query planner may IGNORE your index
  You added write overhead for no read benefit

Exception: Partial index on low-cardinality column
  WHERE status = 'PENDING' and only 0.1% are pending
  → Very selective for that specific value → useful
```

---

## Query Planner and EXPLAIN

The database's query planner decides whether to use an index or not. Use `EXPLAIN` to see its decision:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 'user_123' AND status = 'PENDING';

Output:
Index Scan using idx_orders_user_status on orders
  (cost=0.43..8.45 rows=3 width=200)
  (actual time=0.082..0.091 rows=3 loops=1)
  Index Cond: ((user_id = 'user_123') AND (status = 'PENDING'))

vs without index:
Seq Scan on orders  ← full table scan
  (cost=0.00..285432.00 rows=1000000 width=200)
  (actual time=0.052..3241.823 rows=3 loops=1)

3241ms vs 0.091ms = 35,000x difference
```

Key things to look for in EXPLAIN:
```
Seq Scan       → full table scan → probably missing index
Index Scan     → using index → good
Index Only Scan → covering index → best
Bitmap Scan    → multiple indexes combined → complex query
Nested Loop    → JOIN algorithm → often needs index on join column
Hash Join      → JOIN algorithm → check for missing indexes
```

---

## Index Design Process

In interviews, walk through this process:

```
Step 1: Identify your most frequent, most expensive queries
  "What are the top 5 queries this system runs?"

Step 2: Look at WHERE clauses
  → index candidates

Step 3: Look at JOIN conditions
  → index both sides of joins

Step 4: Look at ORDER BY
  → index matches sort order = avoid filesort

Step 5: Consider composite indexes
  → combine WHERE + ORDER BY columns
  → put equality columns first, range columns last

Step 6: Check cardinality
  → high cardinality columns make effective indexes

Step 7: Measure, don't assume
  → EXPLAIN ANALYZE before and after
  → monitor slow query log in production
```

---

## Index Anti-Patterns

```
Anti-pattern 1: Indexing everything
  CREATE INDEX on every column
  → Write performance collapses
  → Index maintenance overhead exceeds query benefit

Anti-pattern 2: Wrong column order in composite
  CREATE INDEX ON orders(status, user_id)  ← wrong
  WHERE user_id = 'x' AND status = 'PENDING' ← user_id first in query
  → Index not used or suboptimal
  → Should be: CREATE INDEX ON orders(user_id, status)

Anti-pattern 3: Function on indexed column
  WHERE UPPER(email) = 'HARI@EXAMPLE.COM'  ← index not used!
  → B-Tree stores raw value, not function result
  → Fix: store lowercase in DB, or create functional index:
  CREATE INDEX ON users(LOWER(email));

Anti-pattern 4: Implicit type conversion
  WHERE user_id = 123  ← user_id is VARCHAR, 123 is INT
  → Database converts every row → index not used
  → Fix: match types: WHERE user_id = '123'

Anti-pattern 5: Leading wildcard
  WHERE name LIKE '%hari%'  ← index not used
  → B-Tree can't seek to middle of string
  → Fix: full-text search, or LIKE 'hari%' (prefix only)
```

---

## Indexes in Spring Boot

```java
// Spring Data JPA — define indexes at entity level
@Entity
@Table(name = "orders",
    indexes = {
        @Index(name = "idx_orders_user_id",
               columnList = "user_id"),
        @Index(name = "idx_orders_user_status",
               columnList = "user_id, status"),
        @Index(name = "idx_orders_created_at",
               columnList = "created_at"),
    })
public class Order {
    @Id
    private UUID id;

    @Column(name = "user_id")
    private String userId;

    private String status;

    @Column(name = "created_at")
    private Instant createdAt;
}

// Liquibase migration (production-grade index management)
// <createIndex indexName="idx_orders_user_status"
//              tableName="orders">
//     <column name="user_id"/>
//     <column name="status"/>
// </createIndex>
```

---

## Key Takeaways

```
Index: Separate data structure for fast row lookup
       O(n) table scan → O(log n) index lookup

B-Tree: Default index type
        Works for: equality, range, sort, prefix
        Balanced → consistent O(log n) performance

Index types:
  Primary (clustered): one per table, rows ordered by PK
  Secondary:           additional indexes, many per table
  Composite:           multiple columns, leftmost prefix rule
  Unique:              constraint + performance
  Partial:             subset of rows, smaller + faster
  Covering:            all query columns in index, no table read
  Hash:                equality only, no range support
  Full-text:           text search (better: Elasticsearch)

Cost of indexes:
  Reads faster ✅
  Writes slower ❌ (update every index on write)
  Storage overhead ❌

Index design rules:
  High cardinality columns → good index candidates
  Low cardinality → often useless (exception: partial index)
  Composite: equality columns first, range columns last
  Avoid functions on indexed columns
  Avoid leading wildcards (LIKE '%x')
  Use EXPLAIN ANALYZE to verify index is actually used

Interview answer when asked about slow queries:
  1. Run EXPLAIN ANALYZE
  2. Look for Seq Scan on large tables
  3. Check WHERE, JOIN, ORDER BY columns
  4. Add composite index matching query pattern
  5. Verify with EXPLAIN ANALYZE again
```
