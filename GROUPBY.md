# Advanced GROUP BY Guide in SQL

## Table of Contents
1. [Advanced Grouping Operations](#1-advanced-grouping-operations)
2. [Window Functions with GROUP BY](#2-window-functions-with-group-by)
3. [GROUPING SETS](#3-grouping-sets)
4. [CUBE and ROLLUP](#4-cube-and-rollup)
5. [Complex Grouping Conditions](#5-complex-grouping-conditions)
6. [Pivot and Unpivot](#6-pivot-and-unpivot)
7. [Performance Optimization](#7-performance-optimization)

## 1. Advanced Grouping Operations

### A) Grouping with Calculated Expressions
```sql
SELECT 
    EXTRACT(YEAR FROM order_date) as year,
    EXTRACT(MONTH FROM order_date) as month,
    SUM(amount) as total_amount,
    COUNT(DISTINCT customer_id) as unique_customers
FROM orders
GROUP BY 
    EXTRACT(YEAR FROM order_date),
    EXTRACT(MONTH FROM order_date)
HAVING COUNT(DISTINCT customer_id) > 100
ORDER BY year, month;
```
This query demonstrates grouping by extracted date parts and filtering groups based on unique customer count.

### B) Grouping with CASE Statements
```sql
SELECT 
    CASE 
        WHEN annual_revenue < 100000 THEN 'Small'
        WHEN annual_revenue BETWEEN 100000 AND 1000000 THEN 'Medium'
        ELSE 'Large'
    END as business_size,
    industry,
    COUNT(*) as company_count,
    AVG(employee_count) as avg_employees,
    SUM(annual_revenue) as total_revenue
FROM companies
GROUP BY 
    CASE 
        WHEN annual_revenue < 100000 THEN 'Small'
        WHEN annual_revenue BETWEEN 100000 AND 1000000 THEN 'Medium'
        ELSE 'Large'
    END,
    industry;
```

## 2. Window Functions with GROUP BY

### A) Ranking Within Groups
```sql
SELECT 
    department,
    employee_name,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as salary_rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dense_rank,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as row_num,
    AVG(salary) OVER (PARTITION BY department) as dept_avg_salary
FROM employees
GROUP BY department, employee_name, salary;
```

### B) Advanced Running Totals
```sql
SELECT 
    category,
    product_name,
    sales_amount,
    SUM(sales_amount) OVER (PARTITION BY category ORDER BY sales_amount) as running_total,
    sales_amount / SUM(sales_amount) OVER (PARTITION BY category) * 100 as percentage_of_category,
    LAG(sales_amount) OVER (PARTITION BY category ORDER BY sales_amount) as prev_amount,
    LEAD(sales_amount) OVER (PARTITION BY category ORDER BY sales_amount) as next_amount
FROM sales
GROUP BY category, product_name, sales_amount;
```

## 3. GROUPING SETS
GROUPING SETS allow multiple levels of grouping in a single query:

```sql
SELECT 
    COALESCE(region, 'All Regions') as region,
    COALESCE(product_category, 'All Products') as category,
    COALESCE(CAST(EXTRACT(YEAR FROM sale_date) AS VARCHAR), 'All Years') as year,
    SUM(sales_amount) as total_sales,
    GROUPING(region) as is_region_grouped,
    GROUPING(product_category) as is_category_grouped,
    GROUPING(EXTRACT(YEAR FROM sale_date)) as is_year_grouped
FROM sales
GROUP BY GROUPING SETS (
    (region, product_category, EXTRACT(YEAR FROM sale_date)),
    (region, product_category),
    (region),
    (product_category),
    ()
);
```

## 4. CUBE and ROLLUP

### A) CUBE Operation
```sql
SELECT 
    COALESCE(region, 'All Regions') as region,
    COALESCE(product_category, 'All Products') as category,
    COALESCE(CAST(EXTRACT(YEAR FROM sale_date) AS VARCHAR), 'All Years') as year,
    SUM(sales_amount) as total_sales,
    COUNT(*) as transaction_count,
    AVG(sales_amount) as avg_sale
FROM sales
GROUP BY CUBE(region, product_category, EXTRACT(YEAR FROM sale_date));
```

### B) ROLLUP Operation with Multiple Hierarchies
```sql
SELECT 
    COALESCE(country, 'All Countries') as country,
    COALESCE(state, 'All States') as state,
    COALESCE(city, 'All Cities') as city,
    SUM(revenue) as total_revenue,
    COUNT(DISTINCT customer_id) as customer_count
FROM sales
GROUP BY ROLLUP(country, state, city);
```

## 5. Complex Grouping Conditions

### A) HAVING with Subqueries and Window Functions
```sql
WITH sales_stats AS (
    SELECT 
        category,
        SUM(amount) as total_amount,
        AVG(amount) as avg_amount,
        COUNT(*) as transaction_count,
        STDDEV(amount) as amount_stddev
    FROM sales
    GROUP BY category
)
SELECT *
FROM sales_stats
WHERE total_amount > (
    SELECT AVG(total_amount) * 1.5
    FROM sales_stats
)
AND transaction_count >= 100
AND amount_stddev / avg_amount < 0.5;
```

### B) Multiple HAVING Conditions
```sql
SELECT 
    customer_id,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MAX(amount) as max_amount,
    MIN(amount) as min_amount
FROM orders
GROUP BY customer_id
HAVING COUNT(*) >= 5 
    AND SUM(amount) > 1000
    AND MAX(amount) / MIN(amount) >= 2
    AND AVG(amount) > (
        SELECT AVG(amount) 
        FROM orders
    );
```

## 6. Pivot and Unpivot

### A) Static Pivot
```sql
SELECT *
FROM (
    SELECT 
        product_category,
        EXTRACT(QUARTER FROM sale_date) as quarter,
        sales_amount
    FROM sales
) AS SourceTable
PIVOT (
    SUM(sales_amount)
    FOR quarter IN (1, 2, 3, 4)
) AS PivotTable;
```

### B) Dynamic Pivot
```sql
DECLARE @columns NVARCHAR(MAX);
DECLARE @sql NVARCHAR(MAX);

SET @columns = STUFF((
    SELECT DISTINCT ',' + QUOTENAME(quarter)
    FROM (
        SELECT EXTRACT(QUARTER FROM sale_date) as quarter
        FROM sales
    ) q
    FOR XML PATH(''), TYPE
).value('.', 'NVARCHAR(MAX)'), 1, 1, '');

SET @sql = N'
SELECT *
FROM (
    SELECT 
        product_category,
        EXTRACT(QUARTER FROM sale_date) as quarter,
        sales_amount
    FROM sales
) AS SourceTable
PIVOT (
    SUM(sales_amount)
    FOR quarter IN (' + @columns + ')
) AS PivotTable;
';

EXEC sp_executesql @sql;
```

## 7. Performance Optimization

### Best Practices
1. **Indexing**
   - Create indexes on frequently grouped columns
   - Consider composite indexes for multiple grouping columns
   - Include commonly used aggregate columns in indexes

2. **Memory Considerations**
   - Use CTEs instead of temporary tables when possible
   - Consider partitioning for large tables
   - Monitor memory usage for complex grouping operations

3. **Query Optimization**
   - Use GROUPING SETS instead of multiple UNION ALL queries
   - Pre-aggregate data in materialized views for common groupings
   - Consider approximate COUNT DISTINCT for large datasets

### Example of Optimized Complex Grouping
```sql
WITH sales_summary AS (
    SELECT 
        date_trunc('month', sale_date) as sale_month,
        product_category,
        region,
        SUM(amount) as total_amount,
        COUNT(*) as transaction_count
    FROM sales
    WHERE sale_date >= current_date - interval '12 months'
    GROUP BY 
        date_trunc('month', sale_date),
        product_category,
        region
)
SELECT 
    sale_month,
    product_category,
    region,
    total_amount,
    transaction_count,
    total_amount / NULLIF(LAG(total_amount) OVER (
        PARTITION BY product_category, region 
        ORDER BY sale_month
    ), 0) - 1 as month_over_month_growth
FROM sales_summary
WHERE total_amount > 0;
```