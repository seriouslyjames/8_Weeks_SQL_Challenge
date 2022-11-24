### 1. What day of the week is used for each `week_date` value?

```sql
SELECT
	DISTINCT TO_CHAR(week_date, 'Day')
FROM public.clean_weekly_sales
```

### 2. What range of week numbers are missing from the dataset?

```sql
WITH cte AS (
	SELECT *
	FROM GENERATE_SERIES(1, 52) AS week_series
)

SELECT
	week_series
FROM cte AS c1
LEFT JOIN public.clean_weekly_sales AS c2 ON c1.week_series = c2.week_number
WHERE week_number IS NULL;
```

### 3. How many total transactions were there for each year in the dataset?

```sql
SELECT
	calendar_year,
	SUM(transactions) AS total_txn
FROM public.clean_weekly_sales
GROUP BY 1
ORDER BY 1;
```

### 4. What is the total sales for each region for each month?

```sql
SELECT
	region,
	month_number,
	SUM(sales) AS total_sales
FROM public.clean_weekly_sales
GROUP BY 1, 2
ORDER BY 1, 2;
```

### 5. What is the total count of transactions for each platform?

```sql
SELECT
	platform,
	SUM(transactions) AS total_txn
FROM public.clean_weekly_sales
GROUP BY 1
ORDER BY 1;
```

### 6. What is the percentage of sales for Retail vs Shopify for each month?

```sql
-- The MAX clause in the query is neccessary as the platform column needs to be aggregated
-- as we only need one value from that column we can use MAX to select that value (even though we are not looking for the MAX value per se)

WITH cte AS (
	SELECT
		calendar_year,
		month_number,
		platform,
		SUM(sales) AS total_sales
	FROM public.clean_weekly_sales
	GROUP BY 1, 2, 3
	ORDER BY 1, 2, 3
)

SELECT
	calendar_year,
	month_number,
	ROUND(100 * MAX(CASE
					WHEN platform = 'Retail' THEN total_sales
				END) / SUM(total_sales), 2) AS retail_percentage,
	ROUND(100 * MAX(CASE
				WHEN platform = 'Shopify' THEN total_sales
			END) / SUM(total_sales), 2) AS shopify_percentage
FROM cte
GROUP BY 1, 2
ORDER BY 1, 2
```

### 7. What is the percentage of sales by demographic for each year in the dataset?

```sql
WITH cte AS (
	SELECT
		calendar_year,
		demographic,
		SUM(sales) AS total_sales
	FROM public.clean_weekly_sales
	GROUP BY 1, 2
	ORDER BY 1, 2
)

SELECT
	calendar_year,
	ROUND(100 * MAX(CASE
					WHEN demographic = 'Couples' THEN total_sales
				END) / SUM(total_sales), 2) AS couples_percentage,
	ROUND(100 * MAX(CASE
				WHEN demographic = 'Families' THEN total_sales
			END) / SUM(total_sales), 2) AS families_percentage,
	ROUND(100 * MAX(CASE
				WHEN demographic = 'unknown' THEN total_sales
			END) / SUM(total_sales), 2) AS unknown_percentage
FROM cte
GROUP BY 1
ORDER BY 1
```

### 8. Which `age_band` and `demographic` values contribute the most to Retail sales?

```sql
SELECT
	age_band,
	demographic,
	SUM(sales) AS total_sales
FROM public.clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY 1, 2
ORDER BY 3 DESC;
```

### 9. Can we use the `avg_transaction` column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead

```sql
SELECT
	calendar_year,
	platform,
	ROUND(AVG(avg_transaction), 2) AS avg_txn_row,
	ROUND(SUM(sales) / SUM(transactions), 2) AS avg_txn_table
FROM public.clean_weekly_sales
GROUP BY 1, 2
ORDER BY 1, 2
```

### 1. What is the total sales for the 4 weeks before and after `2020-06-15`? What is the growth or reduction rate in actual values and percentage of sales?

```sql
SELECT DISTINCT week_number
FROM public.clean_weekly_sales
WHERE week_date = '2020-06-15';
```

```sql
WITH cte AS (
	SELECT
		SUM(CASE
				WHEN week_number BETWEEN 21 AND 24 THEN sales
		END) AS total_sales_before,
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 28 THEN sales
		END) AS total_sales_after
	FROM public.clean_weekly_sales
	WHERE calendar_year = '2020'
)
	
SELECT
	total_sales_before,
	total_sales_after,
	total_sales_after - total_sales_before AS difference,
	ROUND(100 * (total_sales_after - total_sales_before)::decimal / total_sales_before, 2) AS percentage_difference
FROM cte;
```

### 3. What about the entire 12 weeks before and after?

```sql
WITH cte AS (
	SELECT
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN sales
		END) AS total_sales_before,
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 37 THEN sales
		END) AS total_sales_after
	FROM public.clean_weekly_sales
	WHERE calendar_year = '2020'
)
	
SELECT
	total_sales_before,
	total_sales_after,
	total_sales_after - total_sales_before AS difference,
	ROUND(100 * (total_sales_after - total_sales_before)::decimal / total_sales_before, 2) AS percentage_difference
FROM cte
```

### 3. How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?

```sql
WITH cte AS (
	SELECT
		calendar_year,
		SUM(CASE
				WHEN week_number BETWEEN 21 AND 24 THEN sales
		END) AS sales_before_4_weeks,
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 28 THEN sales
		END) AS sales_after_4_weeks,
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN sales
		END) AS sales_before_12_Weeks,
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 37 THEN sales
		END) AS sales_after_12_weeks
	FROM public.clean_weekly_sales
	GROUP BY calendar_year
)
	
SELECT
	calendar_year,
	sales_before_4_weeks,
	sales_after_4_weeks,
	sales_after_4_weeks - sales_before_4_weeks AS difference_4_weeks,
	ROUND(100 * (sales_after_4_weeks - sales_before_4_weeks)::decimal / sales_before_4_weeks, 2) AS percentage_difference_4_weeks,
	sales_before_12_Weeks,
	sales_after_12_weeks,
	sales_after_12_Weeks - sales_before_12_weeks AS difference_12_weeks,
	ROUND(100 * (sales_after_12_Weeks - sales_before_12_weeks)::decimal / sales_before_12_weeks, 2) AS percentage_difference_12_weeks
FROM cte
```


