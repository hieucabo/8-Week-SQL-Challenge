# 🏦 Case Study #4 - Data Bank  

### A. Customer Nodes Exploration

***

### 1. How many unique nodes are there on the Data Bank system?

````sql
WITH cte AS(
  SELECT
    region_id,
    COUNT(DISTINCT node_id) AS unique_nodes
  FROM customer_nodes
  GROUP BY region_id
)
SELECT
  SUM(unique_nodes) AS total_nodes
FROM cte
````

| total_nodes |
|:-----------:|
|      25     |

- Because customers are randomly distributed across the nodes according to their region, each node and region_id combined is a unique.

### 2. What is the number of nodes per region?

````sql
SELECT
  r.region_name,
  COUNT(DISTINCT node_id) AS num_nodes
FROM customer_nodes AS c
LEFT JOIN regions AS r
ON c.region_id = r.region_id
GROUP BY r.region_name
````

| region_name | num_nodes |
|:-----------:|-----------|
|    Africa   | 5         |
| America     | 5         |
| Asia        | 5         |
| Australia   | 5         |
| Europe      | 5         |

### 3. How many customers are allocated to each region?

````sql
SELECT
  r.region_name,
  COUNT(DISTINCT customer_id) AS num_customer
FROM customer_nodes AS c
LEFT JOIN regions AS r
ON c.region_id = r.region_id
GROUP BY r.region_name
````

| region_name | num_customer |
|:-----------:|--------------|
| Africa      | 102          |
| America     | 105          |
| Asia        | 95           |
| Australia   | 110          |
| Europe      | 88           |

### 4. How many days on average are customers reallocated to a different node?

````sql
WITH cte AS(
  SELECT *,
    LEAD(node_id) OVER(PARTITION BY customer_id, region_id ORDER BY start_date) AS next_node
  FROM customer_nodes
),
cte_2 AS(
  SELECT customer_id, region_id, node_id,
    CASE WHEN next_node IS NULL THEN 0
    ELSE DATEDIFF(day,start_date,end_date)+1 END AS num_day
  FROM cte
  WHERE next_node IS NOT NULL
),
cte_3 AS(
  SELECT
  customer_id, region_id, node_id, SUM(num_day) AS num_day
  FROM cte_2
  GROUP BY customer_id, region_id, node_id
)
  SELECT
    AVG(num_day) AS avg_day
  FROM cte_3
````

| avg_day |
|:-------:|
| 25      |

### 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

````sql
WITH cte AS(
  SELECT *,
    LEAD(node_id) OVER(PARTITION BY customer_id, region_id ORDER BY start_date) AS next_node
  FROM customer_nodes
),
cte_2 AS(
  SELECT customer_id, region_id, node_id,
    CASE WHEN next_node IS NULL THEN 0
    ELSE DATEDIFF(day,start_date,end_date)+1 END AS num_day
  FROM cte
  WHERE next_node IS NOT NULL
),
cte_3 AS(
  SELECT
  customer_id, region_id, node_id, SUM(num_day) AS num_day
  FROM cte_2
  GROUP BY customer_id, region_id, node_id
),
cte_4 AS(
  SELECT
    region_id, num_day, COUNT(*) AS count
  FROM cte_3
  GROUP BY region_id, num_day
),
cte_5 AS(
  SELECT
    region_id,
    ROUND(PERCENTILE_CONT(0.5)
        WITHIN GROUP (ORDER BY num_day)
        OVER (PARTITION BY region_id),0)
        AS median,
    ROUND(PERCENTILE_CONT(0.8)
        WITHIN GROUP (ORDER BY num_day)
        OVER (PARTITION BY region_id),0)
        AS percentile_80,
    ROUND(PERCENTILE_CONT(0.95)
        WITHIN GROUP (ORDER BY num_day)
        OVER (PARTITION BY region_id),0)
        AS percentile_95  
  FROM cte_4
)
  SELECT
    region_id,
    MAX(median) AS median,
    MAX(percentile_80) AS percentile_80,
    MAX(percentile_95) AS percentile_95
  FROM cte_5
  GROUP BY region_id
````

- Using **PERCENTILE_CONT** function to calculate the 80th and 95th percentile

| region_id | median | percentile_80 | percentile_90 |
|:---------:|--------|---------------|---------------|
| 1         | 34     | 54            | 73            |
| 2         | 36     | 57            | 81            |
| 3         | 35     | 56            | 71            |
| 4         | 35     | 55            | 73            |
| 5         | 33     | 52            | 65            |


