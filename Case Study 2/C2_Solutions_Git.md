### Clean Customer Orders Table

```sql
DROP TABLE IF EXISTS customer_orders_t;
CREATE TABLE customer_orders_t AS

SELECT 
	order_id,
	customer_id,
	pizza_id,
	CASE
		WHEN exclusions ISNULL THEN ''
		WHEN exclusions = 'null' THEN ''
		ELSE exclusions
	END AS exclusions,
	CASE
		WHEN extras ISNULL THEN ''
		WHEN extras = 'null' THEN ''
		WHEN extras = 'NaN' THEN ''
		ELSE extras
	END AS extras,
	order_time
FROM pizza_runner.customer_orders;
```

### Clean Runner Orders Table

```sql
DROP TABLE IF EXISTS runner_orders_t;
CREATE TABLE runner_orders_t AS

SELECT 
	order_id,
	runner_id,
	CASE
		WHEN pickup_time = 'null' THEN ''
		ELSE pickup_time
	END AS pickup_time,
	CASE
		WHEN distance ISNULL THEN ''
		WHEN distance = 'null' THEN ''
		WHEN distance LIKE '%km' THEN REPLACE(distance, 'km', '')
		ELSE distance
	END AS distance,
	CASE
		WHEN duration ISNULL THEN ''
		WHEN duration = 'null' THEN ''
		WHEN duration = 'NaN' THEN ''
		WHEN duration LIKE '%minutes%' THEN REPLACE(duration, 'minutes', '')
		WHEN duration LIKE '%minute%' THEN REPLACE(duration, 'minute', '')
		WHEN duration LIKE '%min%' THEN REPLACE(duration, 'mins', '')
		ELSE duration
	END AS duration,
	CASE
		WHEN cancellation ISNULL THEN ''
		WHEN cancellation = 'NaN' THEN ''
		WHEN cancellation = 'null' THEN ''
		ELSE cancellation
	END AS cancellation
FROM pizza_runner.runner_orders;
	
```
## Part A

### 1. How many pizzas were ordered?

```sql
SELECT 
    COUNT(order_id) AS num_orders
FROM customer_orders_t;
```
![image](https://user-images.githubusercontent.com/12231066/203700415-664fcd8a-bcb7-4550-a040-3f50459f4caf.png)

### 2. How many unique customer orders were made?

```sql
SELECT 
    COUNT(DISTINCT order_id) AS num_orders
FROM customer_orders_t;
```
![image](https://user-images.githubusercontent.com/12231066/203700425-b200ee78-0916-468c-87ca-f4ca779dd346.png)

### 3. How many successful orders were delivered by each runner?

```sql
SELECT 
    runner_id, 
    COUNT(order_id)
FROM runner_orders_t
WHERE pickup_time <> ' '
GROUP BY runner_id;
```
![image](https://user-images.githubusercontent.com/12231066/203700436-26bc3770-b194-4752-b8f2-dce30ee372de.png)

### 4. How many of each type of pizza was delivered?

```sql
SELECT 
	p.pizza_name, 
	COUNT(c.pizza_id) AS num_pizzas
FROM customer_orders_t AS c
JOIN runner_orders_t AS r ON c.order_id = r.order_id
JOIN pizza_runner.pizza_names AS p ON c.pizza_id = p.pizza_id
GROUP BY p.pizza_name;
```
![image](https://user-images.githubusercontent.com/12231066/203700445-7ffd60b2-fa7b-4802-bfa2-d772fdf0dbc6.png)

### 5. How many Vegetarian and Meatlovers were ordered by each customer?

```sql
SELECT 
	c.customer_id, 
	p.pizza_name, 
	COUNT(c.pizza_id) AS num_pizzas
FROM customer_orders_t AS c
JOIN runner_orders_t AS r ON c.order_id = r.order_id
JOIN pizza_runner.pizza_names AS p ON c.pizza_id = p.pizza_id
GROUP BY c.customer_id, p.pizza_name
ORDER BY c.customer_id;
```
![image](https://user-images.githubusercontent.com/12231066/203700449-6415df17-aa5b-4a43-9d26-cc89b21415c8.png)

### 6. What was the maximum number of pizzas delivered in a single order?

```sql
WITH cte AS (
	SELECT 
		c.order_id, 
	COUNT(c.pizza_id) AS num_pizza
	FROM customer_orders_t AS c
	JOIN runner_orders_t AS r ON c.order_id = r.order_id
	WHERE pickup_time <> ' '
	GROUP BY c.order_id
)

SELECT 
	MAX(num_pizza) AS num_pizza_ordered
FROM cte
```
![image](https://user-images.githubusercontent.com/12231066/203700456-26470b87-fd58-405c-bf66-51d603309831.png)

### 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

```sql
WITH cte AS (
	SELECT 
		*,
		CASE
			WHEN exclusions = '' AND extras = '' THEN 'N'
			ELSE 'Y'
		END AS changes
	FROM customer_orders_t
)

SELECT 
	c.changes, 
	COUNT(c.changes) AS cnt_of_chng
FROM cte AS c
JOIN runner_orders_t AS r ON c.order_id = r.order_id
WHERE r.pickup_time <> ' '
GROUP BY c.changes;
```
![image](https://user-images.githubusercontent.com/12231066/203700462-c6b57db8-a7b6-400f-8f99-c828ad8f1e5d.png)

### 8. How many pizzas were delivered that had both exclusions and extras?

```sql
WITH cte AS (
	SELECT 
		*,
		CASE
			WHEN exclusions <> '' AND extras <> '' THEN 'Y'
			ELSE 'N'
		END AS both_changes
	FROM customer_orders_t
)

SELECT 
	c.customer_id, 
	c.both_changes, 
	COUNT(c.both_changes) AS cnt_of_chng
FROM cte AS c
JOIN runner_orders_t AS r ON c.order_id = r.order_id
WHERE r.pickup_time <> ' ' AND c.both_changes = 'Y'
GROUP BY c.customer_id, c.both_changes
ORDER BY 1;
```
![image](https://user-images.githubusercontent.com/12231066/203700467-d9d5aa0f-a936-4ebc-85ea-0ea3b729e694.png)

### 9. What was the total volume of pizzas ordered for each hour of the day?

```sql
SELECT 
	EXTRACT(hour FROM order_time) AS hour_ordered,
	COUNT(order_id) AS num_ordered
FROM customer_orders_t
GROUP BY hour_ordered;
```
![image](https://user-images.githubusercontent.com/12231066/203700473-60df83ce-271e-449a-a660-487c6dacc2ce.png)

### 10. What was the volume of orders for each day of the week?

```sql
WITH cte AS (
	SELECT
		*,
		EXTRACT(dow FROM order_time) AS dow_ordered
	FROM customer_orders_t
)

SELECT
	CASE
		WHEN dow_ordered = 0 THEN 'Sunday'
		WHEN dow_ordered = 1 THEN 'Monday'
		WHEN dow_ordered = 2 THEN 'Tuesday'
		WHEN dow_ordered = 3 THEN 'Wednesday'
		WHEN dow_ordered = 4 THEN 'Thursday'
		WHEN dow_ordered = 5 THEN 'Friday'
		ElSE 'Sunday'
	END AS day_of_week,
COUNT(order_id) AS num_ordered
FROM cte
GROUP BY 1;
```
![image](https://user-images.githubusercontent.com/12231066/203700477-4e065170-08f7-41df-9997-73663979deff.png)

## Part B

### 1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)

```sql
WITH cte AS (
	SELECT
		*,
		EXTRACT(week FROM registration_date) AS run_reg_date_wk
	FROM pizza_runner.runners
)

SELECT 
	run_reg_date_wk, 
	COUNT(runner_id) AS num_registered
FROM cte
GROUP BY run_reg_date_wk;
```
![image](https://user-images.githubusercontent.com/12231066/203700489-ec56a677-1979-4997-be93-4516021c6b5f.png)

### 2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

```sql
WITH cte AS(
	SELECT
		c.order_id,
		runner_id,
		order_time AS cus_order_time,
		run_pickup_time
	FROM customer_orders_t AS c
	JOIN (
		SELECT
		 	order_id,
		  	runner_id,
		  	to_timestamp(pickup_time, 'YYYY/MM/DD HH24:MI:SS') AS run_pickup_time
		FROM runner_orders_t 
		WHERE pickup_time <> ''
		  ) AS r ON c.order_id = r.order_id
)

SELECT 
	runner_id,
	AVG(time_to_pickup) AS avg_time_to_pickup
FROM (
	SELECT 
		order_id,
		runner_id,
		cus_order_time,
		run_pickup_time,
		AGE(run_pickup_time, cus_order_time) AS time_to_pickup
	FROM cte
	) AS subq
GROUP BY runner_id;
```
![image](https://user-images.githubusercontent.com/12231066/203700497-3fc61ae3-691c-419b-9440-d6b1ae318abd.png)

### 3. Is there any relationship between the number of pizzas and how long the order takes to prepare?

```sql
WITH cte AS(
	SELECT
		c.order_id,
		runner_id,
		order_time AS cus_order_time,
		run_pickup_time
	FROM customer_orders_t AS c
	JOIN (
		SELECT
		 	order_id,
		  	runner_id,
		  	to_timestamp(pickup_time, 'YYYY/MM/DD HH24:MI:SS') AS run_pickup_time
		FROM runner_orders_t 
		WHERE pickup_time <> ''
		  ) AS r ON c.order_id = r.order_id
)

SELECT 
	order_id,
	COUNT(order_id) AS num_pizzas,
	AVG(time_to_pickup) AS avg_time_to_pickup
FROM (
	SELECT 
		order_id,
		runner_id,
		cus_order_time,
		run_pickup_time,
		AGE(run_pickup_time, cus_order_time) AS time_to_pickup
	FROM cte
	) AS subq
GROUP BY 1
ORDER BY 1;
```
![image](https://user-images.githubusercontent.com/12231066/203700500-bad61a8b-eec1-4141-b191-41cba9844490.png)

### 4. What was the average distance travelled for each customer?

```sql
SELECT 
	c.customer_id,
	AVG(CAST(r.distance AS DOUBLE PRECISION)) AS avg_distance_travelled
FROM customer_orders_t AS c
JOIN runner_orders_t AS r ON c.order_id = r.order_id
WHERE r.distance <> ''
GROUP BY 1;
```
![image](https://user-images.githubusercontent.com/12231066/203700505-9fee8a3f-6411-43c9-8c47-badeb4774b93.png)

### 5. What was the difference between the longest and shortest delivery times for all orders?

```sql
SELECT (MAX(CAST(duration AS int)) - MIN(CAST(duration AS int))) AS difference
FROM runner_orders_t
WHERE pickup_time <> '';
```
![image](https://user-images.githubusercontent.com/12231066/203700512-b7877987-5524-4e08-bf2a-1ba1bf1026df.png)

### 6. What was the average speed for each runner for each delivery and do you notice any trend for these values?

```sql
SELECT 
	order_id AS delivery,
	(CAST(distance AS double precision) / CAST(duration AS double precision)) * 60 AS speed
FROM runner_orders_t
WHERE distance <> '' OR duration <> '';
```
![image](https://user-images.githubusercontent.com/12231066/203700519-4dfb9f91-edaa-4a31-82f6-864b062f5850.png)

### 7. What is the successful delivery percentage for each runner?

```sql
SELECT
	runner_id,
	100 * SUM(
		CASE
			WHEN pickup_time = '' THEN 0
			ELSE 1
		END) / COUNT(*) AS del_percent
FROM runner_orders_t
GROUP BY runner_id;
```
![image](https://user-images.githubusercontent.com/12231066/203700525-9f026852-8e12-420c-83c0-c92f9bfaa101.png)
