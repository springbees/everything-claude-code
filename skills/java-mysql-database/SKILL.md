---
name: java-mysql-database
description: MySQL database standards including table design rules, indexing guidelines, SQL best practices, and ORM mapping conventions. Use this when designing database schemas, writing SQL queries, optimizing performance, or working with MySQL databases.
---

# MySQL Database Standards

## Overview
Comprehensive MySQL database standards based on Alibaba Java Development Handbook, covering table design, indexing, SQL queries, and ORM best practices.

## When to Use
- Designing database schemas
- Writing SQL queries
- Optimizing database performance
- Setting up ORM mappings (MyBatis/Hibernate)
- Conducting database code reviews

## Table Design Rules

### Naming Conventions
```sql
-- Good: lowercase with underscores
CREATE TABLE user_order (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    order_no VARCHAR(64) NOT NULL,
    is_deleted TINYINT UNSIGNED DEFAULT 0,
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_order_no (order_no),
    KEY idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Bad: uppercase or mixed case
CREATE TABLE UserOrder (
    ID BIGINT,
    OrderNo VARCHAR(64)
);
```

### Field Types
```sql
-- Boolean: use tinyint unsigned
is_enabled TINYINT UNSIGNED DEFAULT 1  -- 1=true, 0=false

-- Decimal: use DECIMAL for money
amount DECIMAL(10, 2) UNSIGNED  -- NOT float or double!

-- Strings: use VARCHAR for variable, CHAR for fixed
phone VARCHAR(20)  -- variable length
country_code CHAR(2)  -- fixed length (ISO codes)

-- Text: use TEXT for long content (>5000 chars)
content TEXT

-- Timestamps
create_time DATETIME
-- OR for timezone awareness
create_time TIMESTAMP
```

### Required Fields
```sql
-- Every table MUST have:
id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
is_deleted TINYINT UNSIGNED NOT NULL DEFAULT 0  -- logical delete
```

### Type Selection Guide
| Age Range | Type | Bytes | Range |
|-----------|------|-------|-------|
| 0-255 | TINYINT UNSIGNED | 1 | 0 to 255 |
| 0-65535 | SMALLINT UNSIGNED | 2 | 0 to 65,535 |
| Large numbers | INT UNSIGNED | 4 | 0 to ~4.3 billion |
| IDs, timestamps | BIGINT UNSIGNED | 8 | 0 to ~10^19 |

## Indexing Guidelines

### Naming Convention
```sql
-- Primary key: pk_field
PRIMARY KEY (id)

-- Unique key: uk_field
UNIQUE KEY uk_email (email)

-- Normal index: idx_field
KEY idx_user_id_create_time (user_id, create_time)
```

### Indexing Rules

**1. Unique Indexes for Unique Business Fields**
```sql
-- Business unique fields MUST have unique index
UNIQUE KEY uk_order_no (order_no)
UNIQUE KEY uk_user_email (user_id, email)
```

**2. Composite Indexes**
```sql
-- Leftmost prefix rule
KEY idx_a_b_c (a, b, c)

-- These queries can use the index:
WHERE a = ?
WHERE a = ? AND b = ?
WHERE a = ? AND b = ? AND c = ?
ORDER BY a, b, c

-- These CANNOT use the index:
WHERE b = ?
WHERE b = ? AND c = ?
WHERE c = ?
```

**3. Index Length for VARCHAR**
```sql
-- For long strings, specify prefix length
KEY idx_content (content(20))

-- Calculate optimal length:
SELECT COUNT(DISTINCT LEFT(column_name, 20)) / COUNT(*) AS selectivity
FROM table_name;
-- Aim for >90% selectivity
```

**4. Order By Optimization**
```sql
-- Good: uses index for sorting
WHERE a = ? AND b = ? ORDER BY c  -- with index on (a,b,c)

-- Bad: filesort
WHERE a > ? ORDER BY b  -- cannot use index order
```

## SQL Best Practices

### COUNT Queries
```sql
-- Good: standard SQL92
SELECT COUNT(*) FROM table;

-- Bad: non-standard
SELECT COUNT(id) FROM table;
SELECT COUNT(1) FROM table;
```

### NULL Handling
```sql
-- NULL comparison
WHERE column IS NULL     -- Good
WHERE column IS NOT NULL  -- Good
WHERE column = NULL      -- Wrong! Always returns NULL

-- SUM with NULL
SELECT IFNULL(SUM(amount), 0) FROM table;  -- Good: returns 0 if all NULL
```

### Pagination
```sql
-- Good: check count first
int count = getCount();
if (count == 0) return Collections.emptyList();
return list(offset, limit);

-- Optimization for deep pagination
SELECT t1.*
FROM table1 AS t1
INNER JOIN (
    SELECT id FROM table1 WHERE condition LIMIT 100000, 20
) AS t2 ON t1.id = t2.id;
```

### Multi-Table Joins
```sql
-- Good: limit to <=3 tables
-- Good: join fields have same type and indexes
SELECT t1.name, t2.amount
FROM orders AS t1
INNER JOIN order_items AS t2 ON t1.id = t2.order_id
WHERE t1.user_id = ?;

-- Bad: >3 tables
-- Bad: inconsistent field types
```

### IN Clause
```sql
-- Limit IN values to <1000
WHERE id IN (1, 2, 3, ..., 999)  -- OK
WHERE id IN (1, 2, 3, ..., 1001)  -- Too many!
```

## ORM Mapping (MyBatis)

### Result Mapping
```xml
<!-- Good: always define resultMap -->
<resultMap id="BaseResultMap" type="com.example.UserDO">
    <id column="id" property="id"/>
    <result column="user_name" property="userName"/>
    <result column="is_deleted" property="deleted"/>
</resultMap>

<!-- Bad: no resultMap -->
<select id="selectUser" resultType="com.example.UserDO">
    SELECT * FROM user
</select>
```

### Boolean Field Mapping
```java
// Database: is_enabled (tinyint)
// Java: enabled (Boolean) - NO "is" prefix!

// MyBatis mapping
<resultMap id="BaseResultMap" type="UserDO">
    <result column="is_enabled" property="enabled"/>
</resultMap>
```

### Parameter Binding
```xml
<!-- Good: #{} for parameters (prevents SQL injection)-->
<select id="findById" resultMap="BaseResultMap">
    SELECT id, name FROM user WHERE id = #{id}
</select>

<!-- Bad: ${} (SQL injection risk!)-->
<select id="findById" resultMap="BaseResultMap">
    SELECT id, name FROM user WHERE id = ${id}
</select>
```

### Column Selection
```xml
<!-- Good: specify columns -->
<select id="findById" resultMap="BaseResultMap">
    SELECT id, name, email FROM user WHERE id = #{id}
</select>

<!-- Bad: SELECT * -->
<select id="findById" resultMap="BaseResultMap">
    SELECT * FROM user WHERE id = #{id}
</select>
```

### Aliases
```xml
<!-- Good: use t1, t2, t3... with AS -->
<select id="findWithOrders" resultMap="ResultMap">
    SELECT
        t1.id,
        t1.name,
        t2.order_id,
        t2.amount
    FROM user AS t1
    INNER JOIN user_order AS t2 ON t1.id = t2.user_id
    WHERE t1.id = #{userId}
</select>
```

## Performance Optimization

### EXPLAIN Analysis
```sql
-- Target: const > ref > range > index > ALL
EXPLAIN SELECT * FROM user WHERE id = 1;

-- Good types:
-- const: primary key or unique index
-- ref: non-unique index
-- range: index range scan

-- Bad types:
-- index: full index scan
-- ALL: full table scan
```

### Partitioning Guidelines
- Consider partitioning when table >5M rows OR >2GB
- Don't partition prematurely (check 3-year projections)

### Transaction Usage
```java
// Don't overuse @Transactional
// Consider: cache rollback, search engine rollback, message compensation
```

## Common Pitfalls

### 1. Float for Money
```sql
-- Bad: precision loss
amount FLOAT(10, 2)

-- Good: exact precision
amount DECIMAL(10, 2)
```

### 2. No Index on Foreign Key
```sql
-- Bad: no index
user_id BIGINT UNSIGNED

-- Good: add index
KEY idx_user_id (user_id)
```

### 3. Left Wildcard Search
```sql
-- Bad: cannot use index
WHERE name LIKE '%keyword%'

-- Good: can use index
WHERE name LIKE 'keyword%'

-- Alternative: use Elasticsearch for full-text search
```

### 4. SELECT *
```sql
-- Bad: unnecessary data transfer
SELECT * FROM large_table;

-- Good: specific columns
SELECT id, name FROM large_table;
```

## Quick Reference

### Table Template
```sql
CREATE TABLE `{table_name}` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `{key_field}` VARCHAR(64) NOT NULL COMMENT '...',
    `is_deleted` TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '0=normal, 1=deleted',
    `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'create time',
    `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'update time',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_{key_field}` (`{key_field}`),
    KEY `idx_{query_field}` (`{query_field}`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='{table comment}';
```

### Review Checklist
- [ ] Table/field names are lowercase
- [ ] Boolean fields use is_xxx (DB) / xxx (POJO)
- [ ] Decimal used for money fields
- [ ] id, create_time, update_time present
- [ ] Indexes on foreign keys and query fields
- [ ] No SELECT * in production code
- [ ] ResultMap defined for all queries
- [ ] #{} used for parameters
- [ ] LIMIT used for large result sets

---

**Source:** Alibaba Java Development Handbook (Huangshan Edition 1.7.1)