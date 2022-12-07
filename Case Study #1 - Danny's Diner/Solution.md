# 🍜 Case Study #1: Danny's Diner 

***


### 1. What is the total amount each customer spent at the restaurant?

````sql
SELECT sales.customer_id, SUM(menu.price) AS total_spent
FROM sales
INNER JOIN menu
ON sales.product_id = menu.product_id
GROUP BY sales.customer_id;
````

- Use **INNER JOIN** to join 2 tables, ```customer_id``` from ```sales```, ```price``` from ```menu```.
- Use **SUM** and **GROUP BY** to calculate the total ```total_spent``` for each ```customer_id```.

| customer_id | total_spent |
|-------------|-------------|
| A           | 76          |
| B           | 74          |
| C           | 36          |

- Customer A spent $76.
- Customer B spent $74.
- Customer C spent $36.

### 2. How many days has each customer visited the restaurant?

````sql
SELECT customer_id, COUNT(DISTINCT(order_date)) AS days_visited
FROM sales
GROUP BY customer_id;
````

- Use **COUNT** and **DISTINCT** to calculate the ````day_visited```` of each ````customer_id````.

| customer_id | days_visited |
|-------------|--------------|
| A           | 4            |
| B           | 6            |
| C           | 2            |

- Customer A visited 4 times.
- Customer B visited 6 times.
- Customer C visited 2 times.

### 3. What was the first item from the menu purchased by each customer?

````sql
WITH rank_table AS(
    SELECT order_date, customer_id, DENSE_RANK() OVER
    (ORDER BY order_date) AS rank
    FROM sales
    )
SELECT DISTINCT sales.customer_id,
menu.product_name,
sales.order_date
FROM sales
INNER JOIN  menu
ON sales.product_id = menu.product_id
WHERE sales.order_date IN
    (SELECT order_date
    FROM rank_table
    WHERE rank = 1);
````

- Create a temp table ````rank_table```` using **WITH AS** function.
- Use **DENSE_RANK** and **OVER** to list the first item purchased by the customer, do not use **RANK** here because there might be a chance that 
the customer purchased 2 item in the same day.
- **INNER JOIN** ```sales``` which contains ````order_date```` and ````customer_id```` with ```menu``` which contains ````product_name```` .
- Use a sub-querry to get only the ````order_date```` from ````rank_table```` where ````rank = 1````

| customer_id | product_name | order_date |
|-------------|--------------|------------|
| A           | curry        | 2021-01-01 |
| A           | sushi        | 2021-01-01 |
| B           | curry        | 2021-01-01 |
| C           | ramen        | 2021-01-01 |

- Customer A's first orders are curry and sushi on 2021-01-01.
- Customer B's first order is curry on 2021-01-01.
- Customer C's first order is ramen on 2021-01-01.

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

````sql
SELECT TOP 1 menu.product_name, COUNT(*) as count_times
FROM sales
INNER JOIN menu
ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY count_times DESC;
````
- **INNER JOIN** ````sales```` and ````menu````.
- Use **COUNT** and **GROUP BY** to find ````count_times```` of each ````product_name````.
- **ORDER BY** ````count_times```` **DESC** to list the highest at the top.
- Use **TOP 1** to get the highest ````count_times````. 

| product_name | count_times |
|--------------|-------------|
| ramen        | 8           |

- The most popular product is ramen which was ordered 8 times.

### 5. Which item was the most popular for each customer?

````sql
WITH rank_table AS(
SELECT
    sales.customer_id,
    menu.product_name,
    COUNT(sales.product_id) as num_purchased,
    DENSE_RANK()
    OVER (PARTITION BY customer_id ORDER BY COUNT(sales.product_id) DESC) AS rank
FROM sales
INNER JOIN menu
ON menu.product_id = sales.product_id
GROUP BY sales.customer_id, menu.product_name)
    SELECT
    customer_id,
    product_name,
    num_purchased
    FROM rank_table
    WHERE rank = 1;
````

- Create a temp table ````rank_table```` using **WITH AS** function.
- **INNER JOIN** ````sales```` and ````menu````.
- Use **COUNT** and **GROUP BY** ````customer_id```` and ````product_name```` to know how many times each product was purchased by each customer.
- Use **DENSE_RANK**, **OVER** and **PARTITION BY** to rank the product with the most purchased first by each customer.
- **SELECT** only the ````customer_id```` and ````product_name```` **WHERE** ````rank```` = 1.

| customer_id | product_name | num_purchased |
|-------------|--------------|---------------|
| A           | ramen        | 3             |
| B           | sushi        | 2             |
| B           | curry        | 2             |
| B           | ramen        | 2             |
| C           | ramen        | 3             |

- Ramen is the most purchased by customer A.
- Customer B bought 3 products at the same time. 
- Customer C favourite product is ramen which was purchased 3 times.

### 6. Which item was purchased first by the customer after they became a member?

````sql
WITH rank_table AS(
    SELECT
        sales.customer_id,
        sales.order_date,
        members.join_date,
        DENSE_RANK() OVER(
            PARTITION BY sales.customer_id
            ORDER BY sales.order_date
            ) AS rank
    FROM sales
    INNER JOIN members
    ON sales.customer_id = members.customer_id
    WHERE sales.order_date >= members.join_date
)
        SELECT DISTINCT
            sales.customer_id,
            menu.product_name
        FROM sales
        INNER JOIN menu
        ON menu.product_id = sales.product_id
        INNER JOIN rank_table
        ON sales.customer_id = rank_table.customer_id
        AND sales.order_date = rank_table.order_date
        WHERE rank_table.rank = 1;
````

- Create a temp table ````rank_table```` using **WITH AS** function.
- **INNER JOIN** ````sales```` and ````members````.
- Use **WHERE** to only filtered ````order_date```` after or equal to ````join_date````.
- Use **DENSE_RANK**, **OVER** and **PARTITION BY** to rank the least recently purchased by each customer after they became a member.
- **SELECT** only the ````customer_id```` and ````product_name```` **WHERE** ````rank```` = 1.

| customer_id | product_name |
|-------------|--------------|
| A           | curry        |
| B           | sushi        |

- Customer A and B first purchased product is curry and sushi, respectively. 
- Customer C did not become a member so there is no record for customer C.

### 7. Which item was purchased just before the customer became a member?

````sql
WITH rank_table AS(
    SELECT
        sales.customer_id,
        sales.order_date,
        DENSE_RANK() OVER(
            PARTITION BY sales.customer_id
            ORDER BY order_date DESC
        ) AS rank
    FROM sales
    INNER JOIN members
    ON members.customer_id = sales.customer_id
    WHERE sales.order_date < members.join_date
)
    SELECT DISTINCT
        r1.customer_id,
        r2.product_name,
        r1.order_date
    FROM sales AS r1
    INNER JOIN menu AS r2
    ON r1.product_id = r2.product_id
    INNER JOIN rank_table AS r3
    ON r1.customer_id = r3.customer_id
    AND r1.order_date = r3.order_date
    WHERE r3.rank = 1;
````

- The solution is similar to question 6, except only filtering ````order_date```` before ````join_date````.
- I also listed out the date of the order for a clearer solution.

| customer_id | product_name | order_date |
|-------------|--------------|------------|
| A           | curry        | 2021-01-01 |
| A           | sushi        | 2021-01-01 |
| B           | sushi        | 2021-01-04 |

- Before becoming a member, customer A had purchased curry and sushi on the same day.
- Customer B had purchased sushi before becoming a member.

### 8. What is the total items and amount spent for each member before they became a member?

````sql
WITH cte AS(
    SELECT
        sales.customer_id,
        sales.order_date,
        sales.product_id,
        menu.price,
        members.join_date
    FROM sales
    INNER JOIN members
    ON sales.customer_id = members.customer_id
    INNER JOIN menu
    ON sales.product_id = menu.product_id
    WHERE sales.order_date < members.join_date
)
    SELECT
        customer_id,
        COUNT(*) AS total_items,
        SUM(price) AS total_spent
    FROM cte
    GROUP BY customer_id;
````
- Create a temp table ````cte```` using **WITH AS** function.
- **INNER JOIN** ````sales```` and ````menu````.
- Filter only the ````product_id```` which has ````order_date```` before ````join_date```` using **WHERE**.
- Use **COUNT**, **SUM** and **GROUP BY** to find the ````price```` to find the total items and amount spent for each ````customer_id````.

| customer_id | total_items | total_spent |
|-------------|-------------|-------------|
| A           | 2           | 25          |
| B           | 3           | 40          |

- Customer A bought 2 items and spent a total of 25$.
- Customer B bought 3 items and spent a total of 40$.
- Customer C did not become a member so there is no record.


### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
