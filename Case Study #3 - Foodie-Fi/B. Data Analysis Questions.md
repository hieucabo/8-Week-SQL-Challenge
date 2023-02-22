# 🍌 Case Study #3 - Foodie-Fi 

### B. Data Analysis Questions

***

### 1. How many customers has Foodie-Fi ever had?

````sql
SELECT
    COUNT(DISTINCT customer_id) AS customer_num
FROM subscriptions
````

| customer_num |
|--------------|
| 1000         |


### 2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value

````sql
SELECT
    MONTH(start_date) AS month,
    COUNT(*) AS trial_num
FROM subscriptions
WHERE plan_id IN (
    SELECT plan_id
    FROM plans
    WHERE plan_name = 'Trial'
)
GROUP BY MONTH(start_date)
````

| month | trial_num |
|-------|-----------|
| 1     | 88        |
| 2     | 68        |
| 3     | 94        |
| 4     | 81        |
| 5     | 88        |
| 6     | 79        |
| 7     | 89        |
| 8     | 88        |
| 9     | 87        |
| 10    | 79        |
| 11    | 75        |
| 12    | 84        |

### 3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name

````sql
SELECT
    p.plan_name,
    COUNT(*) AS num_plan
FROM subscriptions AS s
INNER JOIN plans AS p
ON s.plan_id = p.plan_id
WHERE YEAR(s.start_date) > 2020
GROUP BY p.plan_name
````

| plan_name     | num_plan |
|---------------|----------|
| basic monthly | 8        |
| churn         | 71       |
| pro annual    | 63       |
| pro monthly   | 60       |


### 4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

````sql
SELECT
    SUM(
        CASE
            WHEN plan_id = 4 THEN 1
            ELSE 0 END
    ) AS num_customer_churned,
    ROUND(100*
        SUM(
        CASE
            WHEN plan_id = 4 THEN 1
            ELSE 0 END
        ) / COUNT(DISTINCT customer_id),1
    ) AS percentage_customer_churned
FROM subscriptions
````

| num_customer_churned | percentage_customer_churned |
|----------------------|-----------------------------|
| 307                  | 30                          |

### 5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?

````sql
WITH plan_num AS(
    SELECT
        customer_id,
        plan_id,
        RANK() OVER(PARTITION BY customer_id ORDER BY start_date) AS count_plan
    FROM subscriptions
)
SELECT
    SUM(CASE  
        WHEN plan_id = 4 AND count_plan = 2 THEN 1
        ELSE 0 END
    ) AS num_churned_straight,
    ROUND(100*SUM(
            CASE  
            WHEN plan_id = 4 AND count_plan = 2 THEN 1
            ELSE 0 END)/COUNT(DISTINCT customer_id),2
    ) AS percentage_churned_straight
FROM plan_num
````

| num_churned_straight | percentage_churned_straight |
|----------------------|-----------------------------|
| 92                   | 9                           |

### 6. What is the number and percentage of customer plans after their initial free trial?

````sql
WITH plan_num AS(
SELECT
    customer_id,
    plan_id,
    RANK() OVER(PARTITION BY customer_id ORDER BY start_date) AS count_plan
FROM subscriptions
),
percentage_table AS(
    SELECT
        plan_id,
        COUNT(customer_id) AS count_num,
        ROUND(100*CONVERT(float,COUNT(customer_id))/
        (SELECT
            COUNT(DISTINCT customer_id)
        FROM subscriptions),1) AS percentage
    FROM plan_num
    WHERE count_plan = 2
    GROUP BY plan_id
)
SELECT
    plan_name,
    count_num,
    percentage
FROM percentage_table AS p
INNER JOIN plans
````

| plan_name     | count_num | percentage |
|---------------|-----------|------------|
| basic monthly | 546       | 54.6       |
| pro monthly   | 325       | 32.5       |
| pro annual    | 37        | 3.7        |
| churn         | 92        | 9.2        |

### 7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?

````sql
WITH rank_table AS(
    SELECT
        *,
        RANK() OVER(PARTITION BY customer_id ORDER BY start_date DESC) AS rank
    FROM subscriptions
    WHERE start_date <= '2020-12-31'
) ,
plan_table AS(
    SELECT
        plan_id,
        COUNT(customer_id) AS plan_num,
        100*CONVERT(float,COUNT(customer_id))
        /(
        SELECT
            COUNT(DISTINCT customer_id)
        FROM subscriptions) AS percentage
    FROM rank_table
    WHERE rank = 1
    GROUP BY plan_id
)
SELECT
    plan_name,
    plan_num,
    percentage
FROM plan_table
INNER JOIN plans
ON plan_table.plan_id = plans.plan_id
````

| plan_name     | count_num | percentage |
|---------------|-----------|------------|
| trial         | 19        | 1.9        |
| basic monthly | 224       | 22.4       |
| pro monthly   | 326       | 32.6       |
| pro annual    | 195       | 19.5       |
| churn         | 236       | 23.6       |

### 8. How many customers have upgraded to an annual plan in 2020?

````sql
SELECT
    COUNT(customer_id) AS num_cus
FROM subscriptions
WHERE YEAR(start_date) = '2020' AND plan_id = '3'
````

| num_cus |
|---------|
| 195     |

### 9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?

````sql
WITH cus_plan0 AS(
    SELECT
        customer_id,
        start_date AS plan_0
    FROM subscriptions
    WHERE
        plan_id = '0' AND
        customer_id IN (
            SELECT
                customer_id
            FROM subscriptions
            WHERE plan_id = '3'
            )
),
cus_plan3 AS(
    SELECT
        customer_id,
        start_date AS plan_3
    FROM subscriptions
    WHERE plan_id = '3'
)
SELECT
    ROUND(SUM(
            DATEDIFF(day,cp0.plan_0,cp3.plan_3)
            )/CONVERT(float,COUNT(cp0.customer_id)),0) AS avg_day
FROM cus_plan0 AS cp0
INNER JOIN cus_plan3 AS cp3
ON cp0.customer_id = cp3.customer_id
````

| avg_day |
|---------|
| 105     |

### 10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)

````sql
WITH cus_plan0 AS(
    SELECT
        customer_id,
        start_date AS plan_0
    FROM subscriptions
    WHERE
        plan_id = '0' AND
        customer_id IN (
            SELECT
                customer_id
            FROM subscriptions
            WHERE plan_id = '3'
            )
),
cus_plan3 AS(
    SELECT
        customer_id,
        start_date AS plan_3
    FROM subscriptions
    WHERE plan_id = '3'
),
date_dif_table AS(
    SELECT
        cp0.customer_id,
        CONVERT(float,DATEDIFF(day,cp0.plan_0,cp3.plan_3)) AS date_diff
    FROM cus_plan0 AS cp0
    INNER JOIN cus_plan3 AS cp3
    ON cp0.customer_id = cp3.customer_id
),
date_dif_group AS(
    SELECT
        customer_id,
        date_diff,
        CEILING(date_diff/30) AS diff_group
    FROM date_dif_table
),
date_dif_group_name AS(
    SELECT
        date_diff,
        diff_group,
        CASE
            WHEN diff_group = 1 THEN '0-30 days'
            ELSE CONCAT((diff_group-1)*30+1,'-',diff_group*30,' days')
        END AS group_name
    FROM date_dif_group
)
SELECT
    group_name,
    ROUND(AVG(date_diff),0) AS avg_days
FROM date_dif_group_name
GROUP BY group_name,diff_group
ORDER BY diff_group
````

- Create a ````cus_plan0```` temp table for selecting ````customer_id```` and their ````start_date````, only choose those who had upgraded to an annual plan.
- Create a ````cus_plan3```` temp table for storing the ````customer_id```` and ````start_date```` for those had plan 3.
- Create a ````date_dif_table```` to get the difference between the day they join and the day they upgrade to an annual plan.
- Create a ````date_dif_group```` to have the group of 30 days period, I'm using **CEILING** to round it up to the nearest integer.
- Create a ````date_dif_group_name```` to create a ````group_name```` using **CASE WHEN**.

| group_name   | avg_day |
|--------------|---------|
| 0-30 days    | 10      |
| 31-60 days   | 42      |
| 61-90 days   | 71      |
| 91-120 days  | 101     |
| 121-150 days | 133     |
| 151-180 days | 162     |
| 181-210 days | 191     |
| 211-240 days | 224     |
| 241-270 days | 257     |
| 271-300 days | 285     |
| 301-330 days | 327     |
| 331-360 days | 346     |

### 11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

````sql
WITH plan_table AS(
    SELECT
        customer_id,
        start_date,
        plan_id,
        LAG(plan_id) OVER(PARTITION BY customer_id ORDER BY plan_id) AS pre_plan_id
    FROM subscriptions
    WHERE customer_id IN (
        SELECT
            customer_id
        FROM subscriptions
        WHERE plan_id = '2' AND YEAR(start_date) = '2020'
    )
)
    SELECT
        COUNT(*) AS num_cus
    FROM plan_table
    WHERE plan_id = '2' AND pre_plan_id = '3'
````

- Use **LAG** to get the last plan id.
- Select only customers **WHERE** they had a basic monthly plan in 2020. 

| num_cus |
|---------|
| 0       |
