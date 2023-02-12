# 🍕 Case Study #2 - Pizza Runner

### C. Ingredient Optimisation

***

### 1. What are the standard ingredients for each pizza?

````sql
WITH cte AS(
    SELECT
      pizza_id,
      [value] AS topping
    FROM pizza_rep
    CROSS APPLY string_split(REPLACE(toppings,' ',''),',')
),
cte2 AS(
    SELECT
        pizza_id,  
        CONVERT(VARCHAR(20),topping_name) AS name
    FROM cte
    INNER JOIN pizza_toppings
    ON cte.topping = pizza_toppings.topping_id
)
SELECT
    cte2.pizza_id,
    STRING_AGG(cte2.name,', ') as toppings
FROM cte2
GROUP BY pizza_id
````

- This question took me lots of time, but it was interesting while working on it.
- I’m using **CROSS APPLY STRING_SPLIT** to extract each topping id from the toppings.
- Because **CROSS APPLY STRING_SPLIT** does not apply for ````TEXT```` type, 
I must create a temp table name to alter the table data type. 
- After taking all the topping id for each pizza, **INNER JOIN** with ````pizza_toppings````
to get the name then use **STRING_AGG** to combine it into a single row.

| pizza_id | toppings                                                              |
|----------|-----------------------------------------------------------------------|
| 1        | Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
| 2        | Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce            |

### 2. What was the most commonly added extra?

````sql
WITH cte AS(
    SELECT
        order_id,
        pizza_id,
        CONVERT(VARCHAR(50),extras) AS extras
    FROM customer_orders AS co
    WHERE
        extras IS NOT NULL AND
        extras NOT IN ('','null')
),
cte2 AS(
    SELECT
        order_id,
        pizza_id,
        [value] AS extras
    FROM cte
    CROSS APPLY STRING_SPLIT(REPLACE(extras,' ',''),',')
)
SELECT
    CONVERT(VARCHAR(20),topping_name) AS topping_name,
    COUNT(CONVERT(VARCHAR(20),topping_name)) AS frequency
FROM cte2
INNER JOIN pizza_toppings AS pt
ON cte2.extras = pt.topping_id
GROUP BY CONVERT(VARCHAR(20),topping_name)
ORDER BY 2 DESC
````

| topping_name | frequency |
|--------------|-----------|
| Bacon        | 4         |
| Cheese       | 1         |
| Chicken      | 1         |

- Bacon was the most commonly added extra.

### 3. What was the most common exclusion?

- Just change all the extras to exclusion and you can get the solution.

````sql
WITH cte AS(
    SELECT
        order_id,
        pizza_id,
        CONVERT(VARCHAR(50),exclusions) AS exclusions
    FROM customer_orders AS co
    WHERE
        exclusions IS NOT NULL AND
        exclusions NOT IN ('','null')
),
cte2 AS(
    SELECT
        order_id,
        pizza_id,
        [value] AS exclusions
    FROM cte
    CROSS APPLY STRING_SPLIT(REPLACE(exclusions,' ',''),',')
)
SELECT
    CONVERT(VARCHAR(20),topping_name) AS topping_name,
    COUNT(CONVERT(VARCHAR(20),topping_name)) AS frequency
FROM cte2
INNER JOIN pizza_toppings AS pt
ON cte2.exclusions = pt.topping_id
GROUP BY CONVERT(VARCHAR(20),topping_name)
ORDER BY 2 DESC
````

| topping_name | frequency |
|--------------|-----------|
| Cheese       | 4         |
| Mushrooms    | 1         |
| BBQ Sauce    | 1         |

- Cheese was the most commonly exclusion.

### 4. Generate an order item for each record in the customers_orders table in the format of one of the following:

- Meat Lovers
- Meat Lovers - Exclude Beef
- Meat Lovers - Extra Bacon
- Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

***

- First I have to create a temporary table named ````customer_order_cleaned```` where the null value is replaced with a blank space. 
- This question was long, but it was not hard when you know how to use **STRING_SPLIT** and **STRING_AGG**.

````sql
WITH pizza_number AS(
    SELECT *,
        ROW_NUMBER() OVER(PARTITION BY order_id ORDER BY order_id) AS pizza_num
    FROM customer_order_cleaned
),
exclusion_table AS(
    SELECT
        order_id, customer_id, pizza_id,
        exclusions,
        CASE
            WHEN extras IS NOT NULL THEN NULL
            ELSE extras END AS extras,
        order_time, pizza_num
    FROM pizza_number
    WHERE exclusions NOT IN ('')
),
exclu_name AS(
    SELECT
        order_id, customer_id, pizza_id,
        pt.topping_name AS exclusions,
        extras,order_time, pizza_num
    FROM exclusion_table
    CROSS APPLY STRING_SPLIT(exclusions,',')
    INNER JOIN pizza_toppings AS pt
    ON pt.topping_id = [value]
),
extras_table AS(
    SELECT
        order_id, customer_id, pizza_id,
        CASE
            WHEN exclusions IS NOT NULL THEN NULL
            ELSE exclusions END AS exclusions,
        extras,
        order_time, pizza_num
    FROM pizza_number
    WHERE extras NOT IN ('')
),
extra_name AS(
    SELECT
        order_id, customer_id, pizza_id, exclusions,
        pt.topping_name AS extras,
        order_time, pizza_num
    FROM extras_table
    CROSS APPLY STRING_SPLIT(extras,',')
    INNER JOIN pizza_toppings AS pt
    ON pt.topping_id = [value]
),
exclu_extra_table AS(
    SELECT *
    FROM extra_name
    UNION ALL
    SELECT *
    FROM exclu_name
    UNION ALL
    SELECT *
    FROM pizza_number AS coc
    WHERE coc.extras IN ('') AND coc.exclusions IN ('')
),
ex_table AS(
SELECT
    order_id, customer_id, pn.pizza_id,pizza_name,
    STRING_AGG(CONVERT(VARCHAR(30),exclusions),', ') AS exclusions,
    STRING_AGG(CONVERT(VARCHAR(30),extras),', ') AS extras,
    order_time, pizza_num
FROM exclu_extra_table AS eet
INNER JOIN pizza_names AS pn
ON pn.pizza_id = eet.pizza_id
GROUP BY order_id, customer_id, pn.pizza_id, order_time, pizza_num,pizza_name
)
SELECT
    order_id, customer_id,
    CASE
        WHEN exclusions IN ('') AND extras IN ('') THEN pizza_name
        WHEN exclusions IS NOT NULL AND extras IS NULL THEN pizza_name + ' - Exclude ' + exclusions
        WHEN exclusions IS NULL AND extras IS NOT NULL THEN pizza_name + ' - Extra ' + extras
        ELSE pizza_name + ' - Exclude ' + exclusions + ' - Extra ' + extras
        END AS pizza_ordered
FROM ex_table
````

| order_id | customer_id | pizza_ordered                                                  |
|----------|-------------|----------------------------------------------------------------|
| 4        | 103         | Meatlovers - Exclude Cheese                                    |
| 9        | 103         | Meatlovers - Exclude Cheese - Extra Bacon, Chicken              |
| 10       | 104         | Meatlovers - Exclude BBQ Sauce, Mushrooms - Extra Bacon, Cheese |

- I only show the result of 3 rows over 14 rows. But it's how it works.

### 5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
- For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"

````sql
WITH toppings_num AS (
    SELECT
        *,
        ROW_NUMBER() OVER(PARTITION BY order_id ORDER BY order_id) AS pizza_num
    FROM customer_order_cleaned
),
exclusions_num AS (
    SELECT
        order_id, customer_id, pizza_id,
        exclusions,
        null AS extras,
        order_time,
        pizza_num
    FROM toppings_num
    WHERE exclusions NOT IN ('')
),
exclusions_split AS(
    SELECT
        order_id, customer_id, pizza_id,
        [value] AS exclusions,
        extras, order_time, pizza_num
    FROM exclusions_num
    CROSS APPLY STRING_SPLIT(exclusions,',')
),
extras_num AS (
    SELECT
        order_id, customer_id, pizza_id,
        null AS exclusions,
        extras,
        order_time,
        pizza_num
    FROM toppings_num
    WHERE extras NOT IN ('')
),
extras_split AS(
    SELECT
        order_id, customer_id, pizza_id,
        exclusions,
        [value] AS extras, order_time, pizza_num
    FROM extras_num
    CROSS APPLY STRING_SPLIT(extras,',')
),
exclusions_extras AS (
    SELECT *
    FROM extras_split
    UNION ALL
    SELECT *
    FROM exclusions_split
    UNION ALL
    SELECT
        order_id, customer_id, pizza_id,
        null AS exclusions, null AS extras, order_time, pizza_num
    FROM toppings_num
    WHERE exclusions IN ('') AND extras IN ('')
),
ingredients_table AS(
    SELECT
        pizza_id,
        REPLACE(CONVERT(VARCHAR(50),toppings),' ','') AS ingredients
    FROM pizza_recipes
),
ingredients_split AS(
    SELECT
        pizza_id,
        [value] AS ingredients
    FROM ingredients_table
    CROSS APPLY string_split(ingredients,',')
),
ingredients_order AS(
    SELECT DISTINCT
        order_id, customer_id, ee.pizza_id,
        ingredients, order_time, pizza_num
    FROM exclusions_extras AS ee
    INNER JOIN ingredients_split AS ins
    ON ee.pizza_id = ins.pizza_id
),
ingredients_toppings AS(
    SELECT
        t1.order_id, t1.customer_id, t1.pizza_id,
        t1.ingredients,
        (1 - (CASE WHEN exclusions IS NULL THEN 0 ELSE 1 END)
        + (CASE WHEN extras IS NULL THEN 0 ELSE 1 END)) AS count, t1.order_time, t1.pizza_num
    FROM ingredients_order AS t1
    LEFT JOIN exclusions_extras AS t2
    ON
        (t1.ingredients = t2.exclusions OR t1.ingredients = t2.extras)
        AND t1.order_id = t2.order_id AND t1.pizza_num = t2.pizza_num
),
ingredients_split_full AS(
    SELECT t1.order_id, t1.customer_id, t1.pizza_id,
    t2.topping_name, count,
    CASE  
        WHEN count = 1 THEN t2.topping_name
        WHEN count = 0 THEN null
        ELSE CONCAT(t1.count,'x',t2.topping_name) END AS ingredient_list,
    order_time,pizza_num
    FROM ingredients_toppings AS t1
    INNER JOIN pizza_toppings AS t2
    ON t1.ingredients = t2.topping_id
)
    SELECT
        order_id, customer_id, pizza_id,
        STRING_AGG(CONVERT(VARCHAR(100),ingredient_list),',') WITHIN GROUP (ORDER BY CONVERT(VARCHAR(100),topping_name)) AS ingredient_list,
        order_time, pizza_num
    FROM ingredients_split_full
    GROUP BY order_id, customer_id, pizza_id, order_time, pizza_num
    ORDER BY order_id, pizza_num
````

- The result for this question is very long. So I'll put a picture here, hope you guys don't mind.

<p align="center">
  <img src="https://user-images.githubusercontent.com/115451301/218319219-08620e27-b5c8-4fb0-a4af-b08ce003f1b6.png">
</p>

### 6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

````sql
WITH toppings_num AS (
    SELECT
        *,
        ROW_NUMBER() OVER(PARTITION BY order_id ORDER BY order_id) AS pizza_num
    FROM customer_order_cleaned
),
exclusions_num AS (
    SELECT
        order_id, customer_id, pizza_id,
        exclusions,
        null AS extras,
        order_time,
        pizza_num
    FROM toppings_num
    WHERE exclusions NOT IN ('')
),
exclusions_split AS(
    SELECT
        order_id, customer_id, pizza_id,
        [value] AS exclusions,
        extras, order_time, pizza_num
    FROM exclusions_num
    CROSS APPLY STRING_SPLIT(exclusions,',')
),
extras_num AS (
    SELECT
        order_id, customer_id, pizza_id,
        null AS exclusions,
        extras,
        order_time,
        pizza_num
    FROM toppings_num
    WHERE extras NOT IN ('')
),
extras_split AS(
    SELECT
        order_id, customer_id, pizza_id,
        exclusions,
        [value] AS extras, order_time, pizza_num
    FROM extras_num
    CROSS APPLY STRING_SPLIT(extras,',')
),
exclusions_extras AS (
    SELECT *
    FROM extras_split
    UNION ALL
    SELECT *
    FROM exclusions_split
    UNION ALL
    SELECT
        order_id, customer_id, pizza_id,
        null AS exclusions, null AS extras, order_time, pizza_num
    FROM toppings_num
    WHERE exclusions IN ('') AND extras IN ('')
),
ingredients_table AS(
    SELECT
        pizza_id,
        REPLACE(CONVERT(VARCHAR(50),toppings),' ','') AS ingredients
    FROM pizza_recipes
),
ingredients_split AS(
    SELECT
        pizza_id,
        [value] AS ingredients
    FROM ingredients_table
    CROSS APPLY string_split(ingredients,',')
),
ingredients_order AS(
    SELECT DISTINCT
        order_id, customer_id, ee.pizza_id,
        ingredients, order_time, pizza_num
    FROM exclusions_extras AS ee
    INNER JOIN ingredients_split AS ins
    ON ee.pizza_id = ins.pizza_id
),
ingredients_toppings AS(
    SELECT
        t1.order_id, t1.customer_id, t1.pizza_id,
        t1.ingredients,
        (1 - (CASE WHEN exclusions IS NULL THEN 0 ELSE 1 END)
        + (CASE WHEN extras IS NULL THEN 0 ELSE 1 END)) AS count, t1.order_time, t1.pizza_num
    FROM ingredients_order AS t1
    LEFT JOIN exclusions_extras AS t2
    ON
        (t1.ingredients = t2.exclusions OR t1.ingredients = t2.extras)
        AND t1.order_id = t2.order_id AND t1.pizza_num = t2.pizza_num
),
ingredients_split_full AS(
    SELECT t1.order_id, t1.customer_id, t1.pizza_id,
    t2.topping_name, count,
    CASE  
        WHEN count = 1 THEN t2.topping_name
        WHEN count = 0 THEN null
        ELSE CONCAT(t1.count,'x',t2.topping_name) END AS ingredient_list,
    order_time,pizza_num
    FROM ingredients_toppings AS t1
    INNER JOIN pizza_toppings AS t2
    ON t1.ingredients = t2.topping_id
)
    SELECT
        CONVERT(VARCHAR(20),topping_name) AS ingredient,
        SUM(CONVERT(INT,count)) AS quantity
    FROM ingredients_split_full
    GROUP BY CONVERT(VARCHAR(20),topping_name)
    ORDER BY SUM(CONVERT(INT,count)) DESC
````

| ingredient   | quantity |
|--------------|----------|
| Bacon        | 13       |
| Mushrooms    | 13       |
| Cheese       | 11       |
| Chicken      | 11       |
| Pepperoni    | 10       |
| Salami       | 10       |
| Beef         | 10       |
| BBQ Sauce    | 9        |
| Peppers      | 4        |
| Onions       | 4        |
| Tomato Sauce | 4        |
| Tomatoes     | 4        |



