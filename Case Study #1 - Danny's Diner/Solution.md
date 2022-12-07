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

### 7. Which item was purchased just before the customer became a member?

### 8. What is the total items and amount spent for each member before they became a member?

### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
