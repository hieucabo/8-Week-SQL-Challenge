# 🍕 Case Study #2 - Pizza Runner

### D. Pricing and Ratings

***

### 1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?

````sql
SELECT
    SUM(
        CASE
            WHEN pizza_id = 1 THEN 12
            ELSE 10 END
    ) AS money
FROM customer_order_cleaned
WHERE order_id IN (
    SELECT order_id
    FROM runner_orders
    WHERE duration NOT IN ('null')
)
````

| money |
|-------|
| 138   |

### 2. What if there was an additional $1 charge for any pizza extras?

````sql
WITH extras_split AS(
    SELECT
        order_id, pizza_id,
        CASE
            WHEN [value] = '0' THEN 0
            WHEN [value] <> '0' THEN 1 END
            AS extras, pizza_num
    FROM  
        (SELECT
            order_id,
            pizza_id,
            CASE
                WHEN extras IN ('') THEN '0'
                ELSE extras END AS extras ,
            ROW_NUMBER() OVER(PARTITION BY order_id ORDER BY pizza_id) AS pizza_num
        FROM customer_order_cleaned) AS a
    CROSS APPLY STRING_SPLIT(extras,',')
)
SELECT
    SUM(CASE
        WHEN pizza_id = 1 THEN 12 + extras_num
        ELSE 10 + extras_num END) AS money
FROM(
    SELECT
        order_id,
        pizza_id,
        SUM(extras) AS extras_num
    FROM extras_split
    GROUP BY order_id, pizza_id, pizza_num
) AS t
````

| money |
|-------|
| 166   |

### 3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.

- At first, I try to use **RAND()** to generate a number between 0 and 1, 
but the result is the same for all the rows. Then I use **NEWID()** to generate random 
values and **CHECKSUM()** to return it to INT type. Because it can be negative so **ABS()** 
is needed. Because I want a value rating between 0 and 5, so I mod the result by 5 and plus 1. 

````sql
SELECT
    order_id,
    ABS(CHECKSUM(NEWID()))%5+1 AS rating
INTO runner_rating
FROM temp_runner_orders
WHERE distance <> 0;
SELECT * FROM runner_rating
````

| order_id | rating |
|----------|--------|
| 1        | 2      |
| 2        | 1      |
| 3        | 4      |
| 4        | 3      |
| 5        | 3      |
| 7        | 5      |
| 8        | 5      |
| 10       | 4      |

### 4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?

- customer_id
- order_id
- runner_id
- rating
- order_time
- pickup_time
- Time between order and pickup
- Delivery duration
- Average speed
- Total number of pizzas

````sql
WITH order_pizza AS(
    SELECT
        order_id,customer_id,order_time,
        COUNT(pizza_id) AS pizza_num
    FROM customer_order_cleaned
    GROUP BY order_id,customer_id,order_time
)
SELECT
    customer_id,
    tro.order_id,
    runner_id,
    rating,
    order_time,
    pickup_time,
    DATEDIFF(minute,order_time,pickup_time) AS cooking_time_minutes,
    duration AS delivery_time_minutes,
    ROUND(distance * 60 / duration,2) AS avg_speed_km_per_h,
    op.pizza_num
FROM temp_runner_orders AS tro
INNER JOIN runner_rating AS rt
ON tro.order_id = rt.order_id
INNER JOIN order_pizza AS op
ON op.order_id = rt.order_id
ORDER BY customer_id,order_id
````

### 5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

````sql
WITH revenue_table AS(
    SELECT  
        order_id,
        SUM(CASE
            WHEN pizza_id = 1 THEN 12
            ELSE 10 END) AS revenue
    FROM customer_order_cleaned
    WHERE order_id IN (
        SELECT order_id
        FROM temp_runner_orders
        WHERE distance <> 0)
    GROUP BY order_id
),
expense_table AS(
    SELECT
        order_id,
        (0.3 * distance) AS cost
    FROM temp_runner_orders
    WHERE distance <> 0
)
SELECT
    ROUND(SUM(revenue-cost),2) AS money_left
FROM revenue_table AS r
INNER JOIN expense_table AS e
ON r.order_id = e.order_id
````

| money_left |
|------------|
| 94.44      |
