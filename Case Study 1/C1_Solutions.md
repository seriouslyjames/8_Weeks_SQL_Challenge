### 1. What is the total amount each customer spent at the restaurant?

- Since customer sales data is in a separate table from menu prices, we'll use a `JOIN` to work out the amount spent.
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

- Use  `DISTINCT` to count each unique visit and avoid double counting multiple visits on the same day
```sql
SELECT 
    customer_id AS customer, 
    COUNT(DISTINCT order_date) AS times_visited
FROM sales
GROUP BY customer_id;
```
![image](https://user-images.githubusercontent.com/12231066/202123519-48d5579c-395a-442f-813c-027f5fa87d9d.png)

### 3. What was the first item from the menu purchased by each customer?

- Use a CTE to create a temp table. 
- Within the CTE, use a Window Function, specifically `DENSE_RANK` to order the menu items per customer
- In the query, in order to select only the first item(s), we'll use `WHERE` to filter for all '1' rank menu items per our CTE
```sql
WITH earliest_date_cte AS (
	SELECT customer_id,
		   product_name,
		   DENSE_RANK() OVER(PARTITION BY s.customer_id 
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

- Use `COUNT` with `ORDER BY` and `DESC` to work out the most frequently purchased item
- `TOP` 1 will return only the most purchased item
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

- Use a CTE to create a temp table
- Within the CTE, return the `COUNT` of each product along with a Window Function, `DENSE_RANK`, to return each poroducts popularity rank - with rank 1 being most popular
- `GROUP` by customer
- In the query, use `WHERE` to return all rows with rank = 1
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

- Use a CTE to create a temp table
- Within the CTE, use a subquery to return a combined table from sales and members 
- This subquery table will return the date differential between the member's join date and their order date using `DATEDIFF`
- Outside of the subquery, and within the CTE, we'll use a Window Function, `DENSE_RANK`, to order the date differential returned by our subquery in `ASC` order
- In the event the customer became a member after their first order, we'll use `WHERE` to filter for only date differentials equal or higher than 0
- In the query, return the results using `WHERE` rank = 1 for the first product purchased after becoming a member
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

- Logic is similar to question 6, instead of the Window Function being ordered by `ASC`, we'll use `DESC` instead 
- Combined with using `WHERE` to return only date differentials of less than 0, this will give us list of items purchased before the membership start date 
- In the query, return the results using `WHERE` rank = 1 for the last product purchased before they became a member
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

- Create a temp table with a CTE
- Within the CTE, use `WHERE` to filter for data with 'order_date' being before 'join_date'
- In the query, use `COUNT` to return the number of items they've ordered and `SUM` to return the amount spent per customer
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

- Create a temp table with a CTE
- WIthin the CTE, use `CASE` to create a new column and assign the points value of each menu item to this column
- Since Sushi is the only item with a different point value, we only need the `WHEN` clause for Sushi with the `ELSE` clause accounting for the remaining items on the menu
- In the query, use `SUM` to return the total amount spent and total points
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

- Create a temp table with a CTE
- Within the CTE, return a table using `WHERE` to filter the specified date range
- To get our specified date range (1st week after becoming a member), we use `DATEADD` with the join date and '7' to return the 7th day after becoming a member
- In the query, we can now use `SUM` to return the total amount spent for the first week
- Unlike Question 9 where only Sushi was worth 20 points, all items are now worth 20 points which allows us to simply multiply the price by 20 to get the total points
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

Scenario: Recreate the following table output using the available data

![image](https://user-images.githubusercontent.com/12231066/202135979-c9372405-87a7-4860-b0de-0b1211d42e17.png)

- It appears the table we're being asked to recreate is a combination of three tables - 'sales, 'menu', and 'members'
- The requested table also specifies each customer order and whether there were a member or not at the time of the order

- Use `CASE` to create a new column with 'Y' or 'N' results based on whether the customer's membership join date is before their order date 
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

### Bonus Question 2. Rank All The Things

Scenario: Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.

- Create a temp table with a CTE
- Within the CTE, use `CASE` to create a new column with 'Y' or 'N' results based on whether the customer's membership join date is before their order date 
- The CTE will require 2 `JOIN` clauses to combine the 'sales', 'menu', and 'members' table
- In the query, use `CASE` combined with a Window Function, `DENSE_RANK`, to order all members denoted 'Y'
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


