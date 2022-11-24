### 1.

```sql
SELECT 
	COUNT(DISTINCT customer_id)
FROM foodie_fi.subscriptions;
```

### 2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value?

```sql
SELECT
	EXTRACT(MONTH FROM start_date),
	COUNT(EXTRACT(MONTH FROM start_date)) AS month_cnt
FROM foodie_fi.subscriptions
WHERE plan_id = 0
GROUP BY 1
ORDER BY 1;
```

### 3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name

```sql
SELECT
	plan_id,
	COUNT(plan_id) AS cnt_plan_id
FROM foodie_fi.subscriptions
WHERE EXTRACT(YEAR FROM start_date) > 2020
GROUP BY plan_id
ORDER BY plan_id;
```

### 4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

```sql
SELECT
	SUM(CASE
			WHEN plan_id = 4 THEN 1
			ELSE 0
		END) AS num_churned,
	ROUND(SUM(CASE
			WHEN plan_id = 4 THEN 1
			ELSE 0
		END) * 100 / COUNT(DISTINCT customer_id), 1) AS percentage_churned
FROM foodie_fi.subscriptions;
```

### 5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?

```sql
SELECT
	SUM(CASE
	   		WHEN churned_start_date - start_date = 7 THEN 1
	   		ELSE 0
	   	END) AS churned_customers,
	SUM(CASE
	   		WHEN churned_start_date - start_date = 7 THEN 1
	   		ELSE 0
	   	END) * 100 / (SELECT COUNT(DISTINCT customer_id)
						FROM foodie_fi.subscriptions) AS churned_percentage
FROM (SELECT 
	  	customer_id,
	  	start_date
	    FROM foodie_fi.subscriptions
	   WHERE plan_id = 0) AS s1
JOIN (SELECT 
		customer_id AS churned_cust_id,
		start_date AS churned_start_date
	 	FROM foodie_fi.subscriptions
	   WHERE plan_id = 4) AS s2
ON s1.customer_id = s2.churned_cust_id;
```

### 6. What is the number and percentage of customer plans after their initial free trial?

```sql
WITH cte AS (
	SELECT
		customer_id,
		plan_id,
		LEAD(plan_id, 1) OVER(PARTITION BY customer_id ORDER BY plan_id) AS next_plan
	FROM foodie_fi.subscriptions
)

SELECT 
	next_plan,
	COUNT(*) AS cnt_next_plan,
	COUNT(*) * 100 / (SELECT COUNT(DISTINCT customer_id)
			   		FROM foodie_fi.subscriptions) AS percentage_next_plan
FROM cte AS c
WHERE c.plan_id = 0
GROUP BY next_plan
ORDER BY next_plan;
```

### 7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?

```sql
WITH cte AS (
SELECT *
	FROM foodie_fi.subscriptions
WHERE start_date <= '2020-12-31'
)

SELECT 
	plan_id,
	COUNT(DISTINCT customer_id) AS cnt_customers,
	COUNT(DISTINCT customer_id) * 100 / (SELECT COUNT(customer_id)
									   		FROM foodie_fi.subscriptions) AS percentage_customers
FROM cte
GROUP BY 1
ORDER by 1;
```

### 8. How many customers have upgraded to an annual plan in 2020?

```sql
SELECT
	COUNT(DISTINCT customer_id)
FROM foodie_fi.subscriptions
WHERE plan_id = 3 AND start_date < '2021-01-01';
```

### 9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?

```sql
WITH cte1 AS (
	SELECT
		customer_id,
		plan_id,
		start_date,
		DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY plan_id) AS first_start_date
	FROM foodie_fi.subscriptions
),

cte2 AS (
	SELECT 
		customer_id,
		start_date AS annual_join_date
		FROM foodie_fi.subscriptions
	WHERE plan_id = 3
)
	
SELECT
	ROUND(AVG(annual_join_date - start_date), 0) AS avg_num_days_to_annual
FROM cte1 AS c1
JOIN cte2 AS c2 ON c1.customer_id = c2.customer_id
WHERE first_start_date = 1;
```

### 10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)

```sql

```

### 11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

```sql
WITH cte AS (
	SELECT 
		customer_id, 
		plan_id, 
		start_date,
		LEAD(plan_id, 1) OVER(PARTITION BY customer_id ORDER BY plan_id) as next_plan
	FROM foodie_fi.subscriptions
)

SELECT
	COUNT(customer_id)
FROM cte
WHERE start_date < '2021-01-01' 
  AND plan_id = 2
  AND plan_id = 1;
```
