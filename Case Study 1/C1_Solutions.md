### 1. What is the total amount each customer spent at the restaurant?

```sql
SELECT 
    s.customer_id AS customer, 
    SUM(price) AS total
FROM sales AS s
JOIN menu AS m ON s.product_id = m.product_id
GROUP BY s.customer_id;
```

### 2. How many days has each customer visited the restaurant?

```sql
SELECT 
    customer_id AS customer, 
    COUNT(DISTINCT order_date) AS times_visited
FROM sales
GROUP BY customer_id;
```

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

### Bonus Question 3. Join All The Things

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


