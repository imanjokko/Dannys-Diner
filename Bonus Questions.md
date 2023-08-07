1. Recreate the following table output using the available data:
![](https://github.com/imanjokko/Dannys-Diner/blob/main/images/bonus%201.png)
```sql
SELECT sales.customer_id, sales.order_date, menu.product_name, menu.price,
CASE WHEN sales.customer_id = members.customer_id AND sales.order_date >= members.join_date THEN 'Y' ELSE 'N' END AS member
FROM sales
INNER JOIN menu
ON sales.product_id = menu.product_id
LEFT JOIN members
ON members.customer_id = sales.customer_id
ORDER BY customer_id, order_date;
```
**Output**
customer_id|order_date|product_name|price|member
:-----:|:-------:|:--------:|:-----:|:-------:
A|2021-01-01|sushi|10|N
A|2021-01-01|curry|15|N
A|2021-01-07|curry|15|Y
A|2021-01-10|ramen|12|Y
A|2021-01-11|ramen|12|Y
A|2021-01-11|ramen|12|Y
B|2021-01-01|curry|15|N
B|2021-01-02|curry|15|N
B|2021-01-04|sushi|10|N
B|2021-01-11|sushi|10|Y
B|2021-01-16|ramen|12|Y
B|2021-02-01|ramen|12|Y
C|2021-01-01|ramen|12|N
C|2021-01-01|ramen|12|N
C|2021-01-07|ramen|12|N

---
2. Rank All The Things
Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.
![](https://github.com/imanjokko/Dannys-Diner/blob/main/images/bonus2.png)
```sql
WITH CTE AS(
SELECT sales.customer_id, 
        sales.order_date, 
        menu.product_name, 
        menu.price,
        CASE 
            WHEN sales.customer_id = members.customer_id AND sales.order_date >= members.join_date THEN 'Y' 
            ELSE 'N' 
        END AS member
    FROM sales
    INNER JOIN menu ON sales.product_id = menu.product_id
    LEFT JOIN members ON members.customer_id = sales.customer_id
	ORDER BY customer_id, order_date
)
SELECT 
    customer_id, 
    order_date, 
    product_name, 
    price, 
    member,
    CASE 
        WHEN member = 'Y' THEN DENSE_RANK() OVER (PARTITION BY customer_id, member ORDER BY order_date) 
        ELSE NULL
    END AS ranking
FROM CTE
ORDER BY customer_id, order_date;
```
**Output**
customer_id|order_date|product_name|price|member|ranking
:-----:|:-----:|:-----:|:------:|:-------:|:--------:
A|2021-01-01|sushi|10|N|NULL
A|2021-01-01|curry|15|N|NULL
A|2021-01-07|curry|15|Y|1
A|2021-01-10|ramen|12|Y|2
A|2021-01-11|ramen|12|Y|3
A|2021-01-11|ramen|12|Y|3
B|2021-01-01|curry|15|N|NULL
B|2021-01-02|curry|15|N|NULL
B|2021-01-04|sushi|10|N|NULL
B|2021-01-11|sushi|10|Y|1
B|2021-01-16|ramen|12|Y|2
B|2021-02-01|ramen|12|Y|3
C|2021-01-01|ramen|12|N|NULL
C|2021-01-01|ramen|12|N|NULL
C|2021-01-07|ramen|12|N|NULL


