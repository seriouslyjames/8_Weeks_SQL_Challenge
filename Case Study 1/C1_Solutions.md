### 1. What is the total amount each customer spent at the restaurant?

Since customer sales data is in a separate table from menu prices, we'll use a `JOIN` to work out the amount spent.
```sql
SELECT 
    s.customer_id AS customer, 
    SUM(price) AS total
FROM sales AS s
JOIN menu AS m ON s.product_id = m.product_id
GROUP BY s.customer_id;
```

![image](https://user-images.githubusercontent.com/12231066/202123504-e8408ea3-3750-48f3-8f14-e387a48967ed.png)

### 2. How many days has each customer visited the restaurant?

```sql
SELECT 
    customer_id AS customer, 
    COUNT(DISTINCT order_date) AS times_visited
FROM sales
GROUP BY customer_id;
```
![image](https://user-images.githubusercontent.com/12231066/202123519-48d5579c-395a-442f-813c-027f5fa87d9d.png)

### 3. What was the first item from the menu purchased by each customer?

```sql
WITH earliest_date_cte AS (
	SELECT customer_id,
		   product_name,
		   RANK() OVER(PARTITION BY s.customer_id 
						ORDER BY s.order_date) AS date_rank
	FROM sales AS s
	JOIN menu m ON s.product_id = m.product_id
)

SELECT 
	customer_id, 
	product_name
FROM earliest_date_cte
WHERE date_rank = 1
GROUP BY customer_id, product_name
```
![image](https://user-images.githubusercontent.com/12231066/202123536-bbb0cb9d-c5f8-4644-ae2a-43ff73462558.png)

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql
SELECT 
	TOP 1 product_name, 
	COUNT(product_name) AS times_ordered
FROM sales AS s
JOIN menu AS m ON s.product_id = m.product_id
GROUP BY product_name
ORDER BY times_ordered DESC;
```
![image](https://user-images.githubusercontent.com/12231066/202123554-1de24c29-df80-475e-bbfd-590b66ab6331.png)

### 5. Which item was the most popular for each customer?

```sql
WITH cte AS (
	SELECT 
		s.customer_id, 
		product_name, 
		COUNT(product_name) AS order_count,
		DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY COUNT(s.customer_id) DESC) AS order_rank
	FROM sales AS s
	JOIN menu AS m ON s.product_id = m.product_id
	GROUP BY s.customer_id, product_name
)
	
SELECT 
	customer_id AS customer, 
	product_name AS menu_item, 
	order_count
FROM cte
WHERE order_rank = 1
```
![image](https://user-images.githubusercontent.com/12231066/202123571-cc8c5903-90f3-4794-8ef2-663c7084d775.png)

### 6. Which item was purchased first by the customer after they became a member?

```sql
WITH cte AS (
	SELECT *,
			DENSE_RANK() OVER(PARTITION BY customer ORDER BY date_diff ASC) AS date_rank
	FROM (SELECT s.customer_id AS customer, 
				order_date, 
				product_id, 
				mb.join_date AS members_join_date, 
				DATEDIFF(day, mb.join_date, order_date) AS date_diff
			FROM sales AS s
			LEFT JOIN members AS mb ON s.customer_id = mb.customer_id
		) as subq
	WHERE date_diff >= 0
)

SELECT 
	customer, 
	order_date, 
	product_name
FROM cte AS c
LEFT JOIN menu AS m ON c.product_id = m.product_id
WHERE date_rank = 1
```
![image](https://user-images.githubusercontent.com/12231066/202123590-ca62ac45-ba4b-4cc8-ad6b-136e86301aeb.png)

### 7. Which item was purchased just before the customer became a member?

```sql
WITH cte AS (
	SELECT *,
			DENSE_RANK() OVER(PARTITION BY customer ORDER BY date_diff DESC) AS date_rank
	FROM (SELECT s.customer_id AS customer, 
				order_date, 
				product_id, 
				mb.join_date AS members_join_date, 
				DATEDIFF(day, mb.join_date, order_date) AS date_diff
			FROM sales AS s
			LEFT JOIN members AS mb ON s.customer_id = mb.customer_id
		) as subq
	WHERE date_diff < 0
)

SELECT 
	customer, 
	order_date, 
	product_name
FROM cte AS c
LEFT JOIN menu AS m ON c.product_id = m.product_id
WHERE date_rank = 1
```
![image](https://user-images.githubusercontent.com/12231066/202123596-a6d513db-48b7-46b6-9d33-b79ef22cf356.png)

### 8. What is the total items and amount spent for each member before they became a member?

```sql
WITH cte AS (
	SELECT 
		s.customer_id, 
		order_date, 
		product_id
	FROM sales AS s
	JOIN members AS mb ON s.customer_id = mb.customer_id
	WHERE order_date < join_date
)

SELECT 
	c.customer_id, 
	COUNT(DISTINCT c.product_id) AS num_items, 
	SUM(price) AS total_spent
FROM cte AS c
JOIN menu AS m ON c.product_id = m.product_id
GROUP BY c.customer_id
```
![image](https://user-images.githubusercontent.com/12231066/202123601-fda139d4-d0ae-46b8-8322-4460b5d01466.png)

### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

```sql
WITH cte AS (
	SELECT product_id, 
		product_name, 
		price,
		CASE
			WHEN product_name = 'sushi' THEN price * 20
			ELSE price * 10
		END AS points
	FROM menu
)

SELECT 
	s.customer_id AS customer, 
	SUM(price) as total_spent, 
	SUM(points) AS total_points
FROM sales AS s
JOIN cte AS c ON s.product_id = c.product_id
GROUP BY s.customer_id;

```
![image](https://user-images.githubusercontent.com/12231066/202123620-bd533b5b-99f1-4f05-9543-b765b58acab0.png)

### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

```sql
WITH cte AS (
	SELECT 
		s.customer_id,
		s.product_id
	FROM sales AS s
	LEFT JOIN members AS mb ON s.customer_id = mb.customer_id
	WHERE order_date BETWEEN join_date AND DATEADD(day, 7, join_date)
)

SELECT 
	customer_id AS customer, 
	SUM(price) as total_spent, 
	SUM(price * 20) AS total_points
FROM cte AS c
LEFT JOIN menu AS m ON c.product_id = m.product_id
GROUP BY customer_id;
```
![image](https://user-images.githubusercontent.com/12231066/202123634-4add6ffb-3f0c-44b3-beee-94c567544a83.png)

### Bonus Question 1. Join All The Things

```sql
SELECT 
	s.customer_id,
	s.order_date,
	m.product_name,
	m.price,
	CASE
		WHEN mb.join_date <= s.order_date THEN 'Y'
		ELSE 'N'
	END AS member
FROM sales AS s
LEFT JOIN menu AS m ON s.product_id = m.product_id
LEFT JOIN members AS mb ON s.customer_id = mb.customer_id
```
![image](https://user-images.githubusercontent.com/12231066/202123669-6685a55f-1617-4b5e-bf43-620b51e87bf9.png)

### Bonus Question 2. Join All The Things

```sql
WITH cte AS (
	SELECT 
		s.customer_id,
		s.order_date,
		m.product_name,
		m.price,
		CASE
			WHEN mb.join_date <= s.order_date THEN 'Y'
			ELSE 'N'
		END AS member
	FROM sales AS s
	LEFT JOIN menu AS m ON s.product_id = m.product_id
	LEFT JOIN members AS mb ON s.customer_id = mb.customer_id
)

SELECT *,
	CASE
		WHEN member = 'Y' THEN DENSE_RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date)
		ELSE NULL
	END AS ranking
FROM cte
```
![image](https://user-images.githubusercontent.com/12231066/202123707-95ef1d0d-838c-4b75-8689-2bd923d938e2.png)


