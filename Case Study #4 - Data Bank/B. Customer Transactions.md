# 🏦 Case Study #4 - Data Bank  

### B. Customer Transactions

***

### 1. What is the unique count and total amount for each transaction type?

````sql
SELECT
  txn_type,
  COUNT(*) AS count,
  SUM(txn_amount) AS total
FROM customer_transactions
GROUP BY txn_type
````

|  txn_type  | count | total   |
|:----------:|-------|---------|
| purchase   | 1617  | 806537  |
| withdrawal | 1580  | 793003  |
| deposit    | 2671  | 1359168 |

### 2. What is the average total historical deposit counts and amounts for all customers?

````sql
WITH cte AS(
  SELECT
    customer_id,
    COUNT(*) AS count,
    AVG(txn_amount) AS avg_amount_cus
  FROM customer_transactions
  WHERE txn_type = 'deposit'
  GROUP BY customer_id
)
SELECT
  ROUND(AVG(count),1) AS avg_count,
  ROUND(AVG(avg_amount_cus),2) AS avg_amount_total
FROM cte
````

| avg_count | avg_amount_total |
|:---------:|------------------|
| 5         | 508,61           |

### 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

````sql
WITH cte AS(
  SELECT
    customer_id,
    MONTH(txn_date) AS month,
    SUM(
      CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END
    ) AS deposit,
    SUM(
      CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END
    ) AS purchase,
    SUM(
      CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END
    ) AS withdrawal
  FROM customer_transactions
  GROUP BY customer_id, MONTH(txn_date)
)
SELECT
  month,
  COUNT(*) AS num_cus
FROM cte
WHERE deposit > 1 AND (purchase > 0 OR withdrawal >0)
GROUP BY month
````

| month | num_cus |
|:-----:|---------|
| 1     | 168     |
| 2     | 181     |
| 3     | 192     |
| 4     | 70      |

### 4. What is the closing balance for each customer at the end of the month?

````sql
WITH cte AS(
  SELECT
    customer_id,
    MONTH(txn_date) AS month,
    SUM(
      CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END
    ) AS deposit,
    SUM(
      CASE WHEN txn_type = 'purchase' THEN txn_amount ELSE 0 END
    ) AS purchase,
    SUM(
      CASE WHEN txn_type = 'withdrawal' THEN txn_amount ELSE 0 END
    ) AS withdrawal
  FROM customer_transactions
  GROUP BY customer_id, MONTH(txn_date)
), cte2 AS(
  SELECT
    customer_id,
    month,
    deposit,
    purchase,
    withdrawal,
    deposit - purchase - withdrawal AS transaction_dif
  FROM cte
), cte3 AS(
  SELECT
    customer_id, month, transaction_dif,
    LAG(transaction_dif) OVER(PARTITION BY customer_id ORDER BY month) AS last_month
  FROM cte2
)
  SELECT
    customer_id, month, transaction_dif,
    transaction_dif + COALESCE(last_month,0) AS closing_balance
  FROM cte3
````

| customer_id | month | transaction_dif | closing_balance |
|:-----------:|-------|-----------------|-----------------|
| 1           | 1     | 312             | 312             |
| 1           | 3     | -952            | -640            |
| 2           | 1     | 549             | 549             |
| 2           | 3     | 61              | 610             |
| 3           | 1     | 144             | 144             |
| 3           | 2     | -965            | -821            |
| 3           | 3     | -401            | -1366           |

### 5. What is the percentage of customers who increase their closing balance by more than 5%?	

````sql
WITH cte AS(
  SELECT
    customer_id,
    MONTH(txn_date) AS month,
    SUM(
      CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END
    ) AS deposit,
    SUM(
      CASE WHEN txn_type = 'purchase' THEN txn_amount ELSE 0 END
    ) AS purchase,
    SUM(
      CASE WHEN txn_type = 'withdrawal' THEN txn_amount ELSE 0 END
    ) AS withdrawal
  FROM customer_transactions
  GROUP BY customer_id, MONTH(txn_date)
), cte2 AS(
  SELECT
    customer_id,
    month,
    deposit,
    purchase,
    withdrawal,
    deposit - purchase - withdrawal AS transaction_dif
  FROM cte
), cte3 AS(
  SELECT
    customer_id, month, transaction_dif,
    LAG(transaction_dif) OVER(PARTITION BY customer_id ORDER BY month) AS last_month
  FROM cte2
), cte4 AS(
  SELECT
    customer_id, month, transaction_dif,
    transaction_dif + COALESCE(last_month,0) AS closing_balance
  FROM cte3
), cte5 AS(
  SELECT
    customer_id, month, closing_balance,
    LEAD(closing_balance) OVER(PARTITION BY customer_id ORDER BY month) AS next_balance
  FROM cte4
), cte6 AS(
  SELECT
    *,
    ROUND((COALESCE(next_balance,0) - closing_balance)/
      (CASE WHEN closing_balance = 0 THEN 1
      ELSE closing_balance END),1) AS dif_per
  FROM cte5
)
  SELECT
  100 * COUNT(DISTINCT customer_id)/
    (SELECT COUNT(DISTINCT customer_id)
    FROM cte) AS cus_per
  FROM cte6
  WHERE dif_per > 0.05
````

| cus_per |
|:-------:|
| 75      |
