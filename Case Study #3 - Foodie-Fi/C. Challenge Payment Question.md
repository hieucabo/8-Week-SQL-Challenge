# 🍌 Case Study #3 - Foodie-Fi 

### C. Challenge Payment Question


***

The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:
- monthly payments always occur on the same day of month as the original start_date of any monthly paid plan
- upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately
- upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period
- once a customer churns they will no longer make payments

````sql
WITH date_plan_table AS(
    SELECT
        customer_id,
        s.plan_id,
        p.price,
        CASE
            WHEN s.plan_id = 0 THEN 0
            WHEN s.plan_id = 1 OR s.plan_id = 2 THEN 1
            WHEN s.plan_id = 3 THEN 12
            ELSE 0 END AS month_interval,
        start_date,
        LEAD(s.plan_id) OVER(PARTITION BY customer_id ORDER BY s.plan_id) AS next_plan,
        LEAD(start_date) OVER(PARTITION BY customer_id ORDER BY s.plan_id) AS next_plan_day
    FROM subscriptions AS s
    INNER JOIN plans AS p
    ON s.plan_id = p.plan_id
    WHERE s.plan_id > 0 AND YEAR(s.start_date) = 2020
),
last_plan AS(
    SELECT
        *,
        CONVERT(float,DATEDIFF(month,start_date,'2020-12-31') + 1) AS month_count
    FROM date_plan_table
    WHERE next_plan_day IS NULL
),
last_plan_cost AS(
    SELECT
        customer_id,plan_id,
        CASE
            WHEN plan_id <> 4 THEN ROUND(CEILING(month_count/month_interval)*price,2)
            ELSE 0 END AS money
    FROM last_plan
),
mid_plan AS(
    SELECT
        customer_id, plan_id, price,
        month_interval, start_date,
        next_plan_day,
        CASE
            WHEN DAY(start_date) > DAY(next_plan_day) THEN DATEDIFF(month,start_date,next_plan_day)
            ELSE DATEDIFF(month,start_date,next_plan_day) + 1 END AS month_regis
    FROM date_plan_table
    WHERE next_plan IS NOT NULL AND plan_id = 1
),
mid_plan_spend AS(
    SELECT
        customer_id,
        plan_id,
        ROUND(month_regis*price,2) AS money
    FROM mid_plan
),
plan1_day_left AS(
    SELECT
        customer_id,
        plan_id,
        price,
        next_plan_day,
        DATEDIFF(day,next_plan_day,DATEADD(month,month_regis,start_date)) AS day_left
    FROM mid_plan
),
plan1_money_left AS(
    SELECT
        customer_id,
        plan_id,
        -ROUND(CONVERT(float,price*day_left/DATEDIFF(day,next_plan_day,DATEADD(month,1,next_plan_day))),2) AS money
    FROM plan1_day_left
),
plan1_total AS(
    SELECT * FROM mid_plan_spend
    UNION ALL
    SELECT * FROM plan1_money_left
),
plan1_cost AS(
    SELECT
        customer_id, plan_id,
        ROUND(SUM(money),2) AS money
    FROM plan1_total
    GROUP BY customer_id, plan_id
),
plan2_plan3_cost  AS(
    SELECT
        customer_id, plan_id,
        ROUND(price * DATEDIFF(month,start_date,next_plan_day),2) AS money
    FROM date_plan_table
    WHERE plan_id = 2 AND next_plan = 3
),
plan2_plan4 AS(
    SELECT
        *,
    CASE
        WHEN DAY(start_date) > DAY(next_plan_day) THEN DATEDIFF(month,start_date,next_plan_day)
        ELSE DATEDIFF(month,start_date,next_plan_day) + 1  END AS month_regis
    FROM date_plan_table
    WHERE plan_id = 2 AND next_plan = 4
),
plan2_plan4_cost AS(
    SELECT
        customer_id,
        plan_id,
        ROUND(month_regis * price,2) AS money
    FROM plan2_plan4
),
first_plan_cost AS(
    SELECT
        customer_id,
        plan_id,
        0 AS money
    FROM subscriptions
    WHERE plan_id = 0 AND YEAR(start_date) = 2020
),
total_cost AS(
    SELECT * FROM plan1_cost
    UNION ALL
    SELECT * FROM last_plan_cost
    UNION ALL
    SELECT * FROM plan2_plan3_cost
    UNION ALL
    SELECT * FROM plan2_plan4_cost
    UNION ALL
    SELECT * FROM first_plan_cost
)
SELECT
    customer_id,
    ROUND(SUM(money),2) AS total_spent
FROM total_cost
GROUP BY customer_id
ORDER BY customer_id
````

This is such a long and interesting question, so I will guide you step by step through my work. I will divide the plan into three parts to calculate the cost:

- The first plan, which only has a next plan but no last plan.
- The mid plan, which has both a next and a last plan.
- The last plan, which only has a last plan.

### **First Step**: 

- Create a ````data_plan_table```` to have a month interval for each plan, monthly is 1 and annually is 12. Including ````next_plan```` and ````next_plan_date```` using **LEAD()**. Select only those after 2020 and not a trial plan.

### **Second Step**: Calculate the last plan cost

- Create a ````last_plan```` for selecting only those does not have the next plan. This is for calculating the cost of the final plan in 2020.
- Create a ````last_plan_cost```` for calculating the money they had spent, if the plan is churn then 0. 

### **Third Step**: Calculate the mid plan cost

In this I will have 3 phases: 

**Phase 1**: Calculate the price of those who use plan 1 (basic monthly), if the customer did not use it through a full month, the money for the next plan is reduced, but we can consider it as they received a refund and then they charge full price for next plan.
- Create a ````mid_plan````and select only those who were in plan 1. 
- Create a ````mid_plan_spend```` to calculate the money the full month spent in plan 1.
- Create a ````plan1_day_left```` to get the day left of the last plan. To calculate the total money. 
- Create a ````plan1_money_left```` to calculate the money left. 
- Create a ````plan1_total```` using **UNION** of 2 table ````plan1_money_left```` and ````mid_plan_spend````.
- Create a ````plan1_cost```` to **SUM** all the money for plan 1, if the customer's next-day charge is higher than the end date, the money will be negative. 
**Phase 2**: Start with plan 2 and the next plan is plan 3 or plan 4, those who already used plan 2 will have to wait until the plan is finished for the next plan.
- Create a ````plan2_plan3_cost```` table to get the money for plan 2. 
- Create a ````plan2_plan4```` to get the number of months for plan 2.
- Create a ````plan2_plan4_cost```` to calculate the money. 

### **Fourth Step**: Calculate the total cost

- Create a ````first_plan_cost```` for those who had the first plan as plan 0 so their cost is 0.
- Create a ````total_cost```` using **UNION** for 5 cost table.
- Then **SUM** all the money spent. 

| customer_id | total_spent |
|:-----------:|:-----------:|
|      1      |     49.5    |
|      2      |     199     |
|      3      |    118.8    |
|      4      |    28.71    |
|      5      |     49.5    |
|      6      |     9.9     |
|      7      |    192.09   |
|      8      |    114.51   |
|      …      |      …      |
|     996     |     6.39    |
|     997     |    67.05    |
|     998     |     59.7    |
|     999     |     39.8    |
|     1000    |     59.7    |







