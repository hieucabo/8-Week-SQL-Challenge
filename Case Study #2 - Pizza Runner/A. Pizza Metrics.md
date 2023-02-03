# 🍕 Case Study #2 - Pizza Runner

### A. Pizza Metrics

***

### 1. How many pizzas were ordered?

````sql
SELECT
    COUNT(*) AS pizzas_ordered
FROM customer_orders;
````

| pizza_ordered |
|---------------|
| 14            |

### 2. How many unique customer orders were made?

````sql
SELECT
    COUNT(DISTINCT order_id) AS unique_orders
FROM customer_orders;
````
| unique_orders |
|---------------|
| 10            |

### 3. How many successful orders were delivered by each runner?

````sql
SELECT
    runner_id,
    COUNT(pickup_time) AS successful_order
FROM runner_orders
WHERE duration <> 'null'
GROUP BY runner_id;
````
- Exclude order where ````duration```` is null.

| runner_id | successful_order |
|-----------|------------------|
| 1         | 4                |
| 2         | 3                |
| 3         | 1                |

### 4. How many of each type of pizza was delivered?

````sql
SELECT
    t3.pizza_name,
   COUNT(t3.pizza_name) AS successful_order
FROM runner_orders AS t1
INNER JOIN customer_orders AS t2  
ON t1.order_id = t2.order_id
INNER JOIN pizza_names AS t3
ON t2.pizza_id = t3.pizza_id
WHERE t1.distance <> 'null'
GROUP BY t3.pizza_name
````

- First **INNER JOIN** ````runner_orders```` and ````customer_orders```` to get the information for each pizza in each order.
- The second **INNER JOIN** is only to have the name of the pizza. 
- Exclude the pizza where ````distance```` is null.

| pizza_name | successful_order |
|------------|------------------|
| Meatlovers | 9                |
| Vegetarian | 3                |

### 5. How many Vegetarian and Meatlovers were ordered by each customer?

````sql
SELECT
    t1.customer_id,
    t2.pizza_name,
    COUNT(t2.pizza_name) AS cus_piz
FROM customer_orders AS t1
INNER JOIN pizza_names AS t2
ON t1.pizza_id = t2.pizza_id
GROUP BY t1.customer_id,t2.pizza_name
ORDER BY 1,2;
````

| customer_id | pizza_name | cus_piz |
|-------------|------------|---------|
| 101         | Meatlovers | 2       |
| 101         | Vegetarian | 1       |
| 102         | Meatlovers | 2       |
| 102         | Vegetarian | 1       |
| 103         | Meatlovers | 3       |
| 103         | Vegetarian | 1       |
| 104         | Meatlovers | 3       |
| 105         | Vegetarian | 1       |

### 6. What was the maximum number of pizzas delivered in a single order?

````sql
WITH cte AS(
    SELECT
    t2.order_id,
    COUNT(t2.pickup_time) AS number_pizzas
    FROM customer_orders AS t1
    INNER JOIN runner_orders AS t2
    ON t1.order_id = t2.order_id
    WHERE t2.pickup_time <> 'null'
    GROUP BY t2.order_id
),
cte_2 AS(
    SELECT
        number_pizzas,
        DENSE_RANK() OVER(
            ORDER BY number_pizzas DESC
            ) AS rank
    FROM cte
)
SELECT
    number_pizzas
FROM cte_2
WHERE rank = 1;
````

- Create a temp table to select only orders which were delivered. 
- Use **COUNT** to count the number of pizza in that order.
- Create a second temp table to rank those orders by the number of pizza using **DENSE_RANK** for it may has 2 or more orders with the same number of pizza.
- Select ````number_pizzas```` **WHERE** ````rank```` = 1

| number_pizzas |
|---------------|
| 3             |

### 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

````sql
WITH cte AS(
    SELECT
        t1.order_id,
        t1.customer_id,
        SUM(CASE
            WHEN( t1.exclusions IS NULL OR t1.exclusions IN ('null',' '))
            AND (t1.extras IS NULL OR t1.extras IN ('null',' ')) THEN 0
        ELSE 1
        END) AS at_least_1_change,
        SUM(CASE
            WHEN (t1.exclusions IS NOT NULL AND t1.exclusions NOT IN ('null',' '))
            OR (t1.extras IS NOT NULL AND t1.extras NOT IN ('null',' ')) THEN 0
        ELSE 1
        END) AS no_change
    FROM customer_orders AS t1
    GROUP BY t1.order_id, t1.customer_id
)
SELECT
    customer_id,
    SUM(at_least_1_change) AS at_least_1_change,
    SUM(no_change) AS no_change
FROM cte
WHERE order_id IN (
    SELECT order_id
    FROM runner_orders
    WHERE distance <> 'null'
)
GROUP BY customer_id
````

- Combine **SUM** ,**CASE WHEN** and **GROUP BY** to count the number of ````exclusions```` and ````extras```` per order and customer.
- **GROUP BY** customer to find the total number of pizzas that had at least 1 change and how many had no changes.
- Only choose ````order_id```` which was delivered.

| customer_id | at_least_1_change | no_change |
|-------------|-------------------|-----------|
| 101         | 0                 | 2         |
| 102         | 0                 | 3         |
| 103         | 3                 | 0         |
| 104         | 2                 | 1         |
| 105         | 1                 | 0         |

### 8. How many pizzas were delivered that had both exclusions and extras?

````sql
WITH pizza_order AS(
SELECT
    c1.order_id,
    CASE WHEN (
        c1.exclusions NOT IN ('','null') AND c1.exclusions IS NOT NULL
        AND c1.extras NOT IN ('','null') AND c1.extras IS NOT NULL
    ) THEN 1
    ELSE 0 END AS count
FROM customer_orders AS c1
)
SELECT
    SUM(t1.count) AS pizzas_delivered_exclu_extra
FROM pizza_order AS t1
INNER JOIN runner_orders AS t2
ON t1.order_id = t2.order_id
WHERE t2.distance IS NOT NULL AND t2.distance NOT IN ('','null');
````

- Use **CASE WHEN** to number the pizza which has both ````exclusions```` and ````extras````.
- Select only pizza where its order was delivered. 

| pizzas_delivered_exclu_extra |
|------------------------------|
| 1                            |

### 9. What was the total volume of pizzas ordered for each hour of the day?

````sql
SELECT  
    DATEPART(HOUR,[order_time]) AS hour,
    COUNT(order_id) AS pizza_count
FROM customer_orders
GROUP BY DATEPART(HOUR,[order_time]);
````

- Use **DATEPART** to extract hour from ````DATETIME```` data.
- **COUNT** the number of pizza by hour.

| hour | pizza_count |
|------|-------------|
| 11   | 1           |
| 13   | 3           |
| 18   | 3           |
| 19   | 1           |
| 21   | 3           |
| 23   | 3           |

### 10. What was the volume of orders for each day of the week?

````sql
SET DATEFIRST 1
    SELECT  
        COUNT(order_time) AS orders,
        DATENAME(WEEKDAY, order_time) AS week_day
    FROM customer_orders  
    GROUP BY DATENAME(WEEKDAY, order_time)
````

- I have to use **SET DATEFIRST 1** to set the beginning of the week as Monday. 

| orders | week_day  |
|--------|-----------|
| 1      | Friday    |
| 5      | Saturday  |
| 3      | Thursday  |
| 5      | Wednesday |
