### 1. How many unique nodes are there on the Data Bank system?

```sql
SELECT COUNT(DISTINCT node_id)
FROM data_bank.customer_nodes;
```
![image](https://user-images.githubusercontent.com/12231066/203700732-cabaa1b1-7d3e-4abe-8d41-ab8cd16bb268.png)

### 2. What is the number of nodes per region?

```sql
SELECT 
	region_name, 
	COUNT(node_id)
FROM data_bank.customer_nodes AS c
JOIN data_bank.regions AS r ON c.region_id = r.region_id
GROUP BY region_name;
```
![image](https://user-images.githubusercontent.com/12231066/203700740-21e02096-811a-441f-9d25-3536e42a66f3.png)

### 3. How many customers are allocated to each region?

```sql
SELECT 
	region_id, 
	COUNT(DISTINCT customer_id)
FROM data_bank.customer_nodes
GROUP BY region_id;
```
![image](https://user-images.githubusercontent.com/12231066/203700743-e4b9ed71-2b1d-4372-8ada-3e9f94aef755.png)

### 4. How many days on average are customers reallocated to a different node?

```sql
WITH cte AS (
	SELECT 
		customer_id,
		region_id,
		node_id,
		start_date,
		end_date,
		end_date - start_date AS age_diff
	FROM data_bank.customer_nodes
	WHERE end_date <> '9999-12-31'
),
cte2 AS (
	SELECT
		customer_id,
		node_id,
		SUM(age_diff) AS sum_diff
	FROM cte
	GROUP BY customer_id, node_id
)
	
SELECT
	ROUND(AVG(sum_diff), 2) AS average_realloc_age
FROM cte2;
```
![image](https://user-images.githubusercontent.com/12231066/203700758-af6dec72-e55b-4ba6-97f4-29818973d90a.png)

### 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

```sql
WITH cte AS (
	SELECT 
		customer_id,
		region_id,
		node_id,
		start_date,
		end_date,
		end_date - start_date AS age_diff
	FROM data_bank.customer_nodes
	WHERE end_date <> '9999-12-31'
)
	
SELECT
	region_id,
	PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY age_diff) AS median,
	PERCENTILE_DISC(0.8) WITHIN GROUP (ORDER BY age_diff) AS eighty_percentile,
	PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY age_diff) AS ninetyfive_percentile
FROM cte
GROUP BY region_id;
```
![image](https://user-images.githubusercontent.com/12231066/203700782-62addd65-cb20-4879-ac64-103b87bad612.png)

### 1. What is the unique count and total amount for each transaction type?

```sql
SELECT
	txn_type,
	COUNT(DISTINCT customer_id),
	SUM(txn_amount) AS txn_sum
FROM data_bank.customer_transactions
GROUP BY txn_type;
```
![image](https://user-images.githubusercontent.com/12231066/203700813-67b2ce63-7470-4b32-8137-f549a1a002a8.png)

### 2. What is the average total historical deposit counts and amounts for all customers?

```sql
WITH cte AS (
	SELECT
		customer_id,
		COUNT(customer_id) AS count_cust,
		SUM(txn_amount) AS txn_sum
	FROM data_bank.customer_transactions
	WHERE txn_type = 'deposit'
	GROUP BY 1
)

SELECT
	ROUND(AVG(count_cust), 0) AS avg_deposit_txn,
	ROUND(AVG(txn_sum), 2) AS avg_deposit_amt
FROM cte;
```
![image](https://user-images.githubusercontent.com/12231066/203700824-377cae57-4e0c-4876-88ef-b9a87b24a652.png)

### 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

```sql
WITH cte AS (
	SELECT
		customer_id,
		DATE_PART('month', txn_date) AS txn_month,
		SUM(CASE
				WHEN txn_type = 'deposit' THEN 1
				ELSE 0
				END) AS dep_count,
		SUM(CASE
				WHEN txn_type = 'purchase' THEN 1
				ELSE 0
				END) AS purch_count,
		SUM(CASE
				WHEN txn_type = 'withdrawal' THEN 1
				ELSE 0
				END) AS withd_count
	FROM data_bank.customer_transactions
	GROUP BY 1, 2
	ORDER BY 1
)

SELECT
	txn_month,
	COUNT(DISTINCT customer_id)
FROM cte
WHERE 
	dep_count > 1 AND (purch_count = 1 OR withd_count = 1)
GROUP BY txn_month
ORDER BY txn_month;
```
![image](https://user-images.githubusercontent.com/12231066/203700833-061a6b86-8efc-4afa-8283-b7c6d8d6b14c.png)

### 4. What is the closing balance for each customer at the end of the month?

```sql
WITH cte AS (
	SELECT
		customer_id,
		DATE_PART('month', txn_date) AS txn_month,
		CASE
			WHEN txn_type = 'deposit' THEN txn_amount
			ELSE txn_amount * -1
			END AS upd_txn_amount
	FROM data_bank.customer_transactions
	GROUP BY 1, 2, 3
	ORDER BY 1
)

SELECT
	customer_id,
	txn_month,
	SUM(upd_txn_amount) AS closing_bal
FROM cte
GROUP BY 1, 2
ORDER BY 1, 2;
```
![image](https://user-images.githubusercontent.com/12231066/203700870-591efde7-f2f3-4645-b766-bf388bb677ca.png)

### 5. What is the percentage of customers who increase their closing balance by more than 5%?

```sql

```
