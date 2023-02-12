# 🍕 Case Study #2 - Pizza Runner

### B. Runner and Customer Experience

***

### 1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)

````sql
SET DATEFIRST 5
SELECT  
    DATEPART(WEEK, registration_date) AS week_period,
    COUNT(runner_id) AS runner_signed_up
FROM runners  
GROUP BY DATEPART(WEEK, registration_date)	
````
- Because 2021-01-01 is Friday,  I have to use **SET DATEFIRST 5** to start the week on Friday. 

| week_period | runner_signed_up |
|-------------|------------------|
| 1           | 2                |
| 2           | 1                |
| 3           | 1                |

### 2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

- Because duration is ````VARCHAR```` and have text including. 
I have to transform the ````runner_orders```` into ````temp_runner_orders```` by using **SELECT .. INTO .. FROM**. 
- After this question, I recognized how important to clean the data before any analysis.
- However, duration is still ````VARCHAR````. So I had to use **ALTER COLUMN**.
````sql
SELECT
    order_id,
    runner_id,
    CASE
        WHEN pickup_time = 'null' THEN ''
        ELSE pickup_time
        END AS pickup_time,
    CASE
        WHEN distance LIKE ('%km%') THEN REPLACE(distance,'km','')
        WHEN distance = 'null' THEN ''
        ELSE distance
        END AS distance,
    CASE
        WHEN duration LIKE ('%minutes%') THEN REPLACE(duration,'minutes','')
        WHEN duration LIKE ('%minute%') THEN REPLACE(duration,'minute','')
        WHEN duration LIKE ('%mins%') THEN REPLACE(duration,'mins','')
        WHEN duration = 'null' THEN ''
    ELSE duration END AS duration,
    CASE
        WHEN cancellation IS NULL OR cancellation = 'null' THEN ''
    ELSE cancellation END AS cancellation
INTO temp_runner_orders
FROM runner_orders

ALTER TABLE temp_runner_orders
ALTER COLUMN duration INT
````

- Now the data is ready for analyse.

````sql
WITH cte AS(
    SELECT DISTINCT
        t1.order_id,
        t1.runner_id,
        t2.order_time,
        t1.pickup_time,
        DATEDIFF(minute,t2.order_time,t1.pickup_time) AS minutes_needed
    FROM temp_runner_orders AS t1
    INNER JOIN customer_orders AS t2
    ON t1.order_id = t2.order_id
    WHERE t1.pickup_time <> ''
)
SELECT
    runner_id,
    AVG(minutes_needed) AS avg_time_by_runner
FROM cte
GROUP BY runner_id
````

- Only select where the order which ````pickup_time```` had a value.

| runner_id | avg_time_by_runner |
|-----------|--------------------|
| 1         | 14                 |
| 2         | 20                 |
| 3         | 10                 |

### 3. Is there any relationship between the number of pizzas and how long the order takes to prepare?

````sql
WITH cte AS(
    SELECT
        t1.order_id,
        t2.pizza_id,
        t2.order_time,
        t1.pickup_time,
        DATEDIFF(minute,t2.order_time,t1.pickup_time) AS minutes_needed
    FROM temp_runner_orders AS t1
    INNER JOIN customer_orders AS t2
    ON t1.order_id = t2.order_id
    WHERE t1.pickup_time <> ''
),
cte2 AS(
    SELECT
        order_id,
        COUNT(pizza_id) AS num_pizza,
        AVG(minutes_needed) AS time_to_prepare
    FROM cte
    GROUP BY order_id
)
SELECT
    num_pizza,
    AVG(time_to_prepare) AS avg_time_need
FROM cte2
GROUP BY num_pizza
ORDER BY num_pizza
````

| num_pizza | avg_time_need |
|-----------|---------------|
| 1         | 12            |
| 2         | 18            |
| 3         | 30            |

- The higher the number of pizzas increases, the longer the order takes to prepare.
- It takes 12 minutes to make a pizza and only 6 more minutes to make 2 pizzas.
- In contrast, to make 3 pizzas we need 12 more minutes compared to 2 pizzas.
- **Insight**: Maybe Danny's house only had 2 pizza ovens so he can only make 2 at the same time. If there were a third, he had
to start over.

### 4. What was the difference between the longest and shortest delivery times for all orders?

````sql
SELECT
    MAX(duration) - MIN(duration) AS the_difference
FROM temp_runner_orders
WHERE duration <> ''
````

| the_difference |
|----------------|
| 30             |

### 5. What was the average speed for each runner for each delivery and do you notice any trend for these values?

````sql
SELECT
    t1.order_id,
    t1.runner_id,
      COUNT(t2.pizza_id) AS number_of_pizzas,
    ROUND(t1.distance /t1.duration * 60,2) AS avg_speed
FROM temp_runner_orders AS t1
INNER JOIN customer_orders AS t2
ON t1.order_id = t2.order_id
WHERE distance <> ''
GROUP BY t1.order_id, t1.runner_id, ROUND(t1.distance /t1.duration * 60,2)
ORDER BY t1.runner_id, COUNT(t2.pizza_id)
````
| order_id | runner_id | number_of_pizzas | avg_speed |
|----------|-----------|------------------|-----------|
| 1        | 1         | 1                | 37.5      |
| 2        | 1         | 1                | 44.44     |
| 3        | 1         | 2                | 40.2      |
| 10       | 1         | 2                | 60        |
| 7        | 2         | 1                | 60        |
| 8        | 2         | 1                | 93.6      |
| 4        | 2         | 3                | 35.1      |
| 5        | 3         | 1                | 40        |

- There is actually no relationship between the number of pizzas by order and runner speed. 
- The average speed of the runner with id 1 is 37.5 to 40.2 km/h 
- The average speed of the runner with id 2 is 35.1 to 93.6 km/h
- The average speed of the runner with id 3 is 40 km/h.
- The average speed of the runner with id 2 is abnormal as the highest 
average speed is 3 times higher than the lowest. Danny should have a look at this runner.

### 6. What is the successful delivery percentage for each runner?

````sql
SELECT
    runner_id,
    ROUND(
        100*SUM(
        CASE WHEN duration = 0 THEN 0
        ELSE 1 END) / COUNT(order_id),0) AS delivery_percentage
FROM temp_runner_orders
GROUP BY runner_id
````
| runner_id | delivery_percentage |
|-----------|---------------------|
| 1         | 100                 |
| 2         | 75                  |
| 3         | 50                  |

- The runner with id 1 had a perfect delivery_percentage, 100%. 
- The reason that runner with id 3 had such a low successful delivery percentage because he only
had 2 ordered and 1 of them was cancelled by the restaurant.

