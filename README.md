In database design, column partitioning and row partitioning are two distinct strategies for organizing and managing data. Here’s a breakdown of the differences between them:

## Row Partitioning

Row partitioning, often simply referred to as partitioning, involves splitting a table into smaller tables (partitions) based on the values of one or more columns. Each partition contains a subset of the rows from the main table, organized based on the partitioning criteria.

### Key Characteristics

- **Data Distribution**: Rows are distributed across partitions.
- **Partitions**: Each partition is a subset of the rows in the table.
- **Queries**: Querying the main table transparently queries the appropriate partitions.
- **Use Cases**: Effective for time-series data, range-based data, or categorical data where queries often target specific subsets.

### Example

Partition a `sales` table by year.

```sql
-- Main partitioned table
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    sale_date DATE NOT NULL,
    region TEXT NOT NULL,
    amount NUMERIC
) PARTITION BY RANGE (sale_date);

-- Partitions by year
CREATE TABLE sales_2019 PARTITION OF sales
    FOR VALUES FROM ('2019-01-01') TO ('2019-12-31');

CREATE TABLE sales_2020 PARTITION OF sales
    FOR VALUES FROM ('2020-01-01') TO ('2020-12-31');

CREATE TABLE sales_2021 PARTITION OF sales
    FOR VALUES FROM ('2021-01-01') TO ('2021-12-31');

CREATE TABLE sales_2022 PARTITION OF sales
    FOR VALUES FROM ('2022-01-01') TO ('2022-12-31');

```


#### Insert Demo Data

```sql
INSERT INTO sales VALUES (1, '2019-05-15', 'North', 100.00);
INSERT INTO sales VALUES (2, '2020-07-21', 'South', 200.00);
INSERT INTO sales VALUES (3, '2021-08-10', 'East', 150.00);
INSERT INTO sales VALUES (4, '2022-09-05', 'West', 300.00);
```

#### Retrieve Data with Extended Conditional Logic

```sql
DO $$
BEGIN
    IF EXTRACT(YEAR FROM CURRENT_DATE) = 2019 THEN
        RAISE NOTICE 'Retrieving sales for 2019';
        SELECT * FROM sales_2019;
    ELSIF EXTRACT(YEAR FROM CURRENT_DATE) = 2020 THEN
        RAISE NOTICE 'Retrieving sales for 2020';
        SELECT * FROM sales_2020;
    ELSIF EXTRACT(YEAR FROM CURRENT_DATE) = 2021 THEN
        RAISE NOTICE 'Retrieving sales for 2021';
        SELECT * FROM sales_2021;
    ELSIF EXTRACT(YEAR FROM CURRENT_DATE) = 2022 THEN
        RAISE NOTICE 'Retrieving sales for 2022';
        SELECT * FROM sales_2022;
    ELSE
        RAISE NOTICE 'Retrieving all sales';
        SELECT * FROM sales;
    END IF;
END $$;
```

### Benefits

- Improved query performance by scanning only relevant partitions.
- Easier maintenance by operating on smaller subsets of data.

## Column Partitioning

Column partitioning, also known as vertical partitioning or sharding, involves splitting a table by its columns into multiple tables. Each partition contains a subset of the columns from the main table, often to optimize storage or improve query performance for specific access patterns.

### Key Characteristics

- **Data Distribution**: Columns are distributed across partitions.
- **Partitions**: Each partition is a subset of the columns in the table.
- **Queries**: Queries may need to join the partitions to retrieve the full set of data.
- **Use Cases**: Useful when different columns are accessed with different frequencies, or when certain columns are large and not always needed.

### Example

Partition a `customers` table into two tables: one for frequently accessed columns and one for infrequently accessed columns.

```sql
-- Main table
CREATE TABLE customers_base (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    region TEXT NOT NULL
);

-- Extended table for infrequent columns
CREATE TABLE customers_extended (
    id SERIAL PRIMARY KEY,
    address TEXT,
    phone TEXT,
    email TEXT,
    registration_date DATE,
    FOREIGN KEY (id) REFERENCES customers_base(id)
);
```

#### Insert Demo Data

```sql
INSERT INTO customers_base VALUES (1, 'John Doe', 'North');
INSERT INTO customers_base VALUES (2, 'Jane Smith', 'South');

INSERT INTO customers_extended VALUES (1, '123 North St', '123-456-7890', 'john@example.com', '2019-01-01');
INSERT INTO customers_extended VALUES (2, '456 South St', '987-654-3210', 'jane@example.com', '2020-01-01');
```

#### Retrieve Data with Extended Conditional Logic

```sql
DO $$
DECLARE
    v_customer_id INTEGER := 1;  -- example customer ID
BEGIN
    IF (SELECT region FROM customers_base WHERE id = v_customer_id) = 'North' THEN
        RAISE NOTICE 'Retrieving customer from North region';
        SELECT cb.*, ce.address, ce.phone, ce.email, ce.registration_date
        FROM customers_base cb
        LEFT JOIN customers_extended ce ON cb.id = ce.id
        WHERE cb.id = v_customer_id;
    ELSIF (SELECT region FROM customers_base WHERE id = v_customer_id) = 'South' THEN
        RAISE NOTICE 'Retrieving customer from South region';
        SELECT cb.*, ce.address, ce.phone, ce.email, ce.registration_date
        FROM customers_base cb
        LEFT JOIN customers_extended ce ON cb.id = ce.id
        WHERE cb.id = v_customer_id;
    ELSE
        RAISE NOTICE 'Retrieving customer from other region';
        SELECT cb.*
        FROM customers_base cb
        WHERE cb.id = v_customer_id;
    END IF;
END $$;
```

### General Conditional Logic with Extended `CASE` Statements

You can also use extended `CASE` statements directly in `SELECT` queries to handle more complex conditions.

#### Example with Extended `CASE` in SELECT

```sql
-- Retrieve sales with extended conditional logic in SELECT statement
SELECT
    id,
    sale_date,
    region,
    amount,
    CASE
        WHEN EXTRACT(YEAR FROM sale_date) = 2019 THEN 'Year 2019'
        WHEN EXTRACT(YEAR FROM sale_date) = 2020 THEN 'Year 2020'
        WHEN EXTRACT(YEAR FROM sale_date) = 2021 THEN 'Year 2021'
        WHEN EXTRACT(YEAR FROM sale_date) = 2022 THEN 'Year 2022'
        ELSE 'Other Year'
    END AS year_category,
    CASE
        WHEN region = 'North' THEN 'North Region'
        WHEN region = 'South' THEN 'South Region'
        WHEN region = 'East' THEN 'East Region'
        WHEN region = 'West' THEN 'West Region'
        ELSE 'Other Region'
    END AS region_category
FROM sales;
```

#### Example with Extended `CASE` in WHERE Clause

```sql
-- Retrieve sales for specific years with extended conditional logic in WHERE clause
SELECT *
FROM sales
WHERE CASE
    WHEN EXTRACT(YEAR FROM CURRENT_DATE) = 2019 THEN sale_date BETWEEN '2019-01-01' AND '2019-12-31'
    WHEN EXTRACT(YEAR FROM CURRENT_DATE) = 2020 THEN sale_date BETWEEN '2020-01-01' AND '2020-12-31'
    WHEN EXTRACT(YEAR FROM CURRENT_DATE) = 2021 THEN sale_date BETWEEN '2021-01-01' AND '2021-12-31'
    WHEN EXTRACT(YEAR FROM CURRENT_DATE) = 2022 THEN sale_date BETWEEN '2022-01-01' AND '2022-12-31'
    ELSE TRUE
END
AND region = 'North';
```


### Benefits

- Improved performance for queries that access only a subset of columns.
- Reduced I/O by not loading infrequently accessed columns.

## Differences

| Aspect                   | Row Partitioning                                  | Column Partitioning                                |
|--------------------------|---------------------------------------------------|----------------------------------------------------|
| **Distribution Basis**   | Rows (horizontal slicing)                         | Columns (vertical slicing)                         |
| **Partitions**           | Sub-tables with a subset of rows                  | Sub-tables with a subset of columns                |
| **Querying**             | Transparently accesses relevant partitions        | May require joining multiple partitions            |
| **Use Cases**            | Time-series, range-based, categorical data        | Different column access patterns, large columns    |
| **Maintenance**          | Partition-level maintenance for subsets of rows   | Partition-level maintenance for subsets of columns |
| **Storage Optimization** | Better for large datasets with frequent range queries | Better for large tables with varying column access patterns |


### Summary

- **Row Partitioning**: Expanded to include multiple years and regions with complex retrieval conditions.
- **Column Partitioning**: Expanded to include more columns and regions with detailed retrieval conditions.
- **Conditional Logic**: Demonstrated using both PL/pgSQL blocks and `CASE` statements for comprehensive conditions.

## Conclusion

Both row and column partitioning offer strategies to optimize database performance and manageability, but they serve different purposes and are suitable for different scenarios. Understanding the differences and appropriate use cases for each type can help design more efficient and effective database schemas.
