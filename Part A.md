# Questions

1. What is the total amount spent by each customer at the restaurant
```sql
WITH CTE AS(
SELECT s.customer_id, s.product_id, COUNT(s.customer_id) * m.price AS total_spend, m.price
FROM sales s
INNER JOIN menu m
ON s.product_id = m.product_id
GROUP BY s.product_id, s.customer_id, m.price
ORDER BY s.customer_id)

SELECT customer_id, SUM(total_spend) AS total_spend
FROM CTE
GROUP BY customer_id;
```
**Output**
customer_id | total_spend
:----------:|:------------:
A | 76
B | 74
C | 36

---
2. How many days has each customer visited the restaurant?
```sql
SELECT customer_id, COUNT (DISTINCT order_date) AS total_days
FROM sales
GROUP BY 1;
```
**Output**
customer_id | total_days
:-------:|:----------:
A|4
B|6
C|2

---
3. What was the first item from the menu purchased by each customer?
```sql
SELECT sub.customer_id, sub.product_name, sub.order_date
FROM ( 
		SELECT s.customer_id, s.product_id, m.product_name, s.order_date,
				RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS date_rank
		FROM sales s 
		JOIN menu m ON s.product_id = m.product_id
	 ) AS sub
WHERE date_rank = 1
ORDER BY 1;
```
**Output**
customer_id | product_name |order_date
:----:|:-------:|:--------:
A|curry|2021-01-01
A|sushi|2021-01-01
B|curry|2021-01-01
C|ramen|2021-01-01
C|ramen|2021-01-01

---
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT s.customer_id, m.product_name, s.product_id, 
		COUNT(s.product_id) AS num_purchases_by_customer
FROM sales s
INNER JOIN menu m
ON s.product_id = m.product_id
INNER JOIN (
    SELECT product_id
    FROM sales
    GROUP BY product_id
    ORDER BY COUNT(product_id) DESC
    LIMIT 1
) most_purchased
ON s.product_id = most_purchased.product_id
GROUP BY s.customer_id, m.product_name, s.product_id
ORDER BY num_purchases_by_customer DESC;
```
**Output**
customer_id|product_name|product_id|num_purchases_by_customer
:--------:|:-----------:|:----------:|:-----------:
A|ramen|3|3
C|ramen|3|3
B|ramen|3|2

To get only the total purchases not grouped by customers
```sql
SELECT m.product_name, s.product_id, COUNT(s.product_id) AS num_purchases
FROM sales s
INNER JOIN menu m
ON s.product_id = m.product_id
GROUP BY 1, 2
ORDER BY num_purchases DESC
LIMIT 1;
```
**Output**
product_name|product_id|num_purchases
:---------:|:---------:|:-----------:
ramen|3|8

---
5. Which item was the most popular for each customer?
```sql
WITH CTE AS(
    SELECT 
        customer_id,
        product_name,
      RANK() OVER (PARTITION BY customer_id ORDER BY COUNT(*) DESC) AS rank
    FROM sales s
  	INNER JOIN menu m 
	ON s.product_id = m.product_id
    GROUP BY customer_id, product_name) 
SELECT customer_id, product_name
FROM CTE
WHERE rank = 1;
```
**Output**
customer_id|product_name
:------:|:-----------:
A|ramen
B|sushi
B|curry
B|ramen
C|ramen

---
6. Which item was purchased first by the customer after they became a member?
```sql
WITH CTE AS (
    SELECT 
        sales.customer_id,
        menu.product_name,
        RANK() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date) AS purchase_rank
    FROM sales
    INNER JOIN menu 
	ON sales.product_id = menu.product_id
	INNER JOIN members
	ON sales.customer_id = members.customer_id
	WHERE sales.order_date >= members.join_date
)
SELECT customer_id, product_name
FROM CTE
WHERE purchase_rank = 1;
```
**Output**
customer_id|product_name
:------:|:---------:
A|curry
B|sushi

---
7. Which item was purchased just before the customer became a member?
```sql
WITH CTE AS (
    SELECT 
        sales.customer_id,
        menu.product_name,
        RANK() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date DESC) AS purchase_rank
    FROM sales
    INNER JOIN menu 
	ON sales.product_id = menu.product_id
	INNER JOIN members
	ON sales.customer_id = members.customer_id
	WHERE sales.order_date < members.join_date
)
SELECT customer_id, product_name
FROM CTE
WHERE purchase_rank = 1;
```
**Output**
customer_id|product_name
:------:|:---------:
A|sushi
A|curry
B|sushi

---
8. What is the total items and amount spent for each member before they became a member?
```sql
WITH CTE AS (
    SELECT 
        sales.customer_id,
        COUNT(*) AS total_items,
        SUM(menu.price) AS amt_spent
    FROM sales
    INNER JOIN menu ON sales.product_id = menu.product_id
    INNER JOIN members ON sales.customer_id = members.customer_id
    WHERE sales.order_date < members.join_date
    GROUP BY sales.customer_id
)
SELECT customer_id, total_items, amt_spent
FROM CTE
ORDER BY customer_id;
```
**Output**
customer_id|total_items|amt_spent
:----------:|:---------:|:-------:
A|2|25
B|3|40

---
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
WITH CTE AS (
    SELECT 
        sales.customer_id,
        COUNT(*) AS total_items,
        SUM(menu.price) AS total_amount_spent,
        SUM(CASE WHEN menu.product_name = 'sushi' THEN menu.price * 2 
			ELSE menu.price END) AS total_amount_with_points,
        sales.order_date
    FROM sales
    INNER JOIN menu 
	ON sales.product_id = menu.product_id
    GROUP BY sales.customer_id, sales.order_date
)
SELECT 
    customer_id,
    SUM(total_amount_with_points * 10) AS total_points
FROM CTE
GROUP BY customer_id
ORDER BY customer_id;
```
**Output**
customer_id|total_points
:------:|:-------:
A|860
B|940
C|360

---
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH CTE AS(
SELECT sales.customer_id, 
	members.join_date, 
	menu.product_id, 
	menu.product_name, 
	menu.price,
	sales.order_date
FROM sales
INNER JOIN members
ON sales.customer_id = members.customer_id
INNER JOIN menu
ON menu.product_id = sales.product_id
WHERE sales.order_date < '02-01-2021'
ORDER BY sales.customer_id
)
SELECT customer_id,
SUM(CASE 
     WHEN order_date BETWEEN join_date AND (join_date + INTERVAL '1 week') THEN price * 20 -- 2x points for all items in the first week
     WHEN product_name = 'sushi' THEN price * 20 -- 2x points for sushi outside the first week
     ELSE price * 10 -- Default 10 points per USD for other purchases outside the first week
        END
    ) AS total_points
FROM CTE
GROUP BY customer_id;
```
**Output**
customer_id|total_points
:------:|:-------:
A|1370
B|940






