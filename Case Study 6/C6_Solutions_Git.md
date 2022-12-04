### 1. How many users are there?

```sql
SELECT 
	COUNT(DISTINCT user_id)
FROM clique_bait.users;
```

### 2. How many cookies does each user have on average?

```sql
WITH cte AS (
	SELECT
		user_id,
		COUNT(DISTINCT cookie_id) cookie_count
	FROM clique_bait.users
	GROUP BY user_id
)

SELECT 
	ROUND(AVG(cookie_count), 2)
FROM cte;
```

### 3. What is the unique number of visits by all users per month?

```sql
SELECT
	DATE_PART('month', event_time),
	COUNT(DISTINCT visit_id)
FROM clique_bait.events
GROUP BY 1;
```

### 4. What is the number of events for each event type?

```sql
SELECT
	event_type,
	COUNT(visit_id)
FROM clique_bait.events
GROUP BY 1;
```

### 5. What is the percentage of visits which have a purchase event?

```sql
SELECT
	ROUND(SUM
		  (CASE
			WHEN event_type = 3 THEN 1
			ELSE 0
		   END) * 100::DECIMAL / COUNT(DISTINCT visit_id), 2) AS purchase_percentage
FROM clique_bait.events;
```

### 6. What is the percentage of visits which view the checkout page but do not have a purchase event?

```sql
SELECT
	ROUND(100 * (1 - SUM(CASE
						WHEN event_type = 3 THEN 1
						ELSE 0
					 END)::DECIMAL / (SELECT
										SUM(CASE
											WHEN page_id = 12 THEN 1
											ELSE 0
									  END)
								  	 )
				), 2) AS checkout_no_purchase_percent
FROM clique_bait.events
```

### 7. What are the top 3 pages by number of views?

```sql
SELECT
	page_id,
	COUNT(*)
FROM clique_bait.events AS e
JOIN clique_bait.page_hierarchy AS ph
	USING(page_id)
WHERE event_type = 1
GROUP BY page_id
ORDER BY 2 DESC
LIMIT 3;
```

### 8. What is the number of views and cart adds for each product category?

```sql
SELECT
	product_category,
	SUM(CASE
	   		WHEN event_type = 2 THEN 1
			ELSE 0
		END) AS add_cart_count,
	SUM(CASE
	   		WHEN event_type = 1 THEN 1
			ELSE 0
		END) AS page_view_count
FROM clique_bait.events AS e
JOIN clique_bait.page_hierarchy AS ph
	USING(page_id)
WHERE product_category IS NOT NULL
GROUP BY product_category;
```

### 9. What are the top 3 products by purchases?

```sql
WITH add_to_cart_cte AS (
	SELECT
		*
	FROM clique_bait.events AS e
	JOIN clique_bait.page_hierarchy AS ph
		USING(page_id)
	WHERE event_type = 2
),

checkout_cte AS (
	SELECT
		*
	FROM clique_bait.events
	WHERE event_type = 3
)

SELECT
	page_name,
	COUNT(*) AS num_purchased
FROM add_to_cart_cte AS atc
JOIN checkout_cte AS chk
	USING(visit_id)
GROUP BY page_name
ORDER BY 2 DESC
LIMIT 3;
```
