# SQL Project: Data Analysis for Zomato - A Food Delivery Company

## Overview

This project demonstrates my SQL problem-solving skills through the analysis of data for Zomato, a popular food delivery company in India. The project involves setting up the database, importing data, handling null values, and solving a variety of business problems using complex SQL queries.

![ERD](https://github.com/Jay18034/Zomato_sql_project/blob/main/erd.png)

## Project Structure

- **Database Setup:** Creation of the `zomato_db` database and the required tables.
- **Data Import:** Inserting sample data into the tables.
- **Data Cleaning:** Handling null values and ensuring data integrity.
- **Business Problems:** Solving 17 specific business problems using SQL queries.


## Database Setup
```sql
CREATE DATABASE zomato_db;
```

### 1. Dropping Existing Tables
```sql
DROP TABLE IF EXISTS deliveries;
DROP TABLE IF EXISTS Orders;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS restaurants;
DROP TABLE IF EXISTS riders;

-- 2. Creating Tables
CREATE TABLE restaurants (
    restaurant_id SERIAL PRIMARY KEY,
    restaurant_name VARCHAR(100) NOT NULL,
    city VARCHAR(50),
    opening_hours VARCHAR(50)
);

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    reg_date DATE
);

CREATE TABLE riders (
    rider_id SERIAL PRIMARY KEY,
    rider_name VARCHAR(100) NOT NULL,
    sign_up DATE
);

CREATE TABLE Orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    restaurant_id INT,
    order_item VARCHAR(255),
    order_date DATE NOT NULL,
    order_time TIME NOT NULL,
    order_status VARCHAR(20) DEFAULT 'Pending',
    total_amount DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id)
);

CREATE TABLE deliveries (
    delivery_id SERIAL PRIMARY KEY,
    order_id INT,
    delivery_status VARCHAR(20) DEFAULT 'Pending',
    delivery_time TIME,
    rider_id INT,
    FOREIGN KEY (order_id) REFERENCES Orders(order_id),
    FOREIGN KEY (rider_id) REFERENCES riders(rider_id)
);
```

## Data Import

## Data Cleaning and Handling Null Values

Before performing analysis, I ensured that the data was clean and free from null values where necessary. For instance:

```sql
UPDATE orders
SET total_amount = COALESCE(total_amount, 0);
```

## Business Problems Solved

### 1. List the top 5 most frequently ordered dishes by Arjun Mehta for each year, based on the total number of orders placed.

```sql
SELECT 
    customer_name,
    order_year,
    dish_name,
    total_orders
FROM (
    SELECT 
        customer_name,
        order_year,
        dish_name,
        total_orders,
        DENSE_RANK() OVER (
            PARTITION BY order_year
            ORDER BY total_orders DESC
        ) AS rnk
    FROM (
        SELECT 
            c.customer_name,
            YEAR(o.order_date) AS order_year,
            o.order_item       AS dish_name,
            COUNT(*)           AS total_orders
        FROM orders o
        JOIN customers c ON c.customer_id = o.customer_id
        WHERE c.customer_name = 'Arjun Mehta'
        GROUP BY c.customer_name, YEAR(o.order_date), o.order_item
    ) AS yearly_counts
) AS ranked
WHERE rnk <= 5
ORDER BY order_year, total_orders DESC;
```

### 2. Popular Time Slots
-- Question: Identify the time slots during which the most orders are placed. based on 2-hour intervals.

```sql
SELECT 
    FLOOR(HOUR(order_time)/2)*2 AS start_time,
    FLOOR(HOUR(order_time)/2)*2 + 2 AS end_time,
    COUNT(*) AS total_orders
FROM orders
GROUP BY 1, 2
ORDER BY total_orders DESC;
```

### 3. Order Value Analysis
-- Question: Find the average order value per customer who has placed more than 750 orders.
-- Return customer_name, and aov(average order value)

```sql
SELECT 
    c.customer_name,
    AVG(o.total_amount) AS aov
FROM orders AS o
JOIN customers AS c ON c.customer_id = o.customer_id
GROUP BY c.customer_name
HAVING COUNT(o.order_id) > 750;
```

### 4. High-Value Customers
-- Question: List the customers who have spent more than 100K in total on food orders.
-- return customer_name, and customer_id!

```sql
-- ✅ Customers who spent more than 100K in total
SELECT 
    c.customer_id,
    c.customer_name,
    SUM(o.total_amount) AS total_spent
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING total_spent > 100000;
```

### 5. Orders Without Delivery
-- Question: Write a query to find orders that were placed but not delivered. 
-- Return each restuarant name, city and number of not delivered orders 

```sql

SELECT 
    r.restaurant_name,
    r.city,
    COUNT(o.order_id) AS not_delivered
FROM orders o
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
LEFT JOIN deliveries d ON o.order_id = d.order_id
WHERE d.delivery_id IS NULL OR d.delivery_status != 'Delivered'
GROUP BY r.restaurant_name, r.city
ORDER BY not_delivered DESC;

```


### 6. Restaurant Revenue Ranking: 
-- Rank restaurants by their total revenue from the last year(based on the latest order date). Return each restaurant's name, city, total revenue, and rank within their city

```sql
WITH latest_date AS (
    SELECT MAX(order_date) AS max_date FROM orders
),
ranking_table AS (
    SELECT 
        r.city,
        r.restaurant_name,
        SUM(o.total_amount) AS revenue,
        RANK() OVER (PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) AS rank
    FROM orders o
    JOIN restaurants r ON r.restaurant_id = o.restaurant_id
    JOIN latest_date ld ON 1=1
    WHERE o.order_date >= ld.max_date - INTERVAL 1 YEAR
    GROUP BY r.city, r.restaurant_name
)
SELECT *
FROM ranking_table
WHERE rank = 1;

```

### 7. Most Popular Dish by City: 
-- Identify the most popular dish in each city based on the number of orders.

```sql
WITH most_orders AS
(
SELECT 
	r.city,
    o.order_item as dish,
    COUNT(o.order_id) AS total_orders,
    DENSE_RANK() OVER (PARTITION BY r.city ORDER BY COUNT(o.order_id) DESC ) AS rnk
FROM restaurants r
JOIN orders o ON r.restaurant_id=o.restaurant_id
GROUP BY 1,2
)
SELECT 
	city,
    dish,
    total_orders
FROM most_orders
WHERE rnk = 1;

```

### 8. Customer Churn: 
-- Find customers who haven’t placed an order in 2024 but did in 2023.

```sql
SELECT DISTINCT customer_id 
FROM orders
WHERE YEAR(order_date) = 2023
  AND customer_id NOT IN (
		SELECT DISTINCT customer_id 
		FROM orders
		WHERE YEAR(order_date) = 2024
);
```

### 9. Cancellation Rate Comparison: 
-- Calculate and compare the order cancellation rate for each restaurant between the 
-- current year and the previous year.

```sql
WITH cancel_ratio_23 AS (
    SELECT 
        o.restaurant_id,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL THEN 1 END) AS not_delivered
    FROM orders o
    LEFT JOIN deliveries d ON o.order_id = d.order_id
    WHERE YEAR(o.order_date) = 2023
    GROUP BY o.restaurant_id
),
cancel_ratio_24 AS (
    SELECT 
        o.restaurant_id,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL THEN 1 END) AS not_delivered
    FROM orders o
    LEFT JOIN deliveries d ON o.order_id = d.order_id
    WHERE YEAR(o.order_date) = 2024
    GROUP BY o.restaurant_id
),
last_year_data AS (
    SELECT 
        restaurant_id,
        total_orders,
        not_delivered,
        ROUND((not_delivered / total_orders) * 100, 2) AS cancel_ratio
    FROM cancel_ratio_23
),
current_year_data AS (
    SELECT 
        restaurant_id,
        total_orders,
        not_delivered,
        ROUND((not_delivered / total_orders) * 100, 2) AS cancel_ratio
    FROM cancel_ratio_24
)

SELECT 
    c.restaurant_id,
    c.cancel_ratio AS current_year_cancel_ratio,
    l.cancel_ratio AS last_year_cancel_ratio
FROM current_year_data c
JOIN last_year_data l ON c.restaurant_id = l.restaurant_id;
```

### 10. Rider Average Delivery Time: 
-- Determine each rider's average delivery time.

```sql
SELECT 
    d.rider_id,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, o.order_time, d.delivery_time)), 2) AS avg_delivery_time_mins
FROM orders o
JOIN deliveries d ON o.order_id = d.order_id
WHERE d.delivery_status = 'Delivered'
GROUP BY d.rider_id;
```

### 11. Monthly Restaurant Growth Ratio: 
-- Calculate each restaurant's growth ratio based on the total number of delivered orders since its joining

```sql
WITH growth_ratio AS (
    SELECT 
        o.restaurant_id,
        YEAR(o.order_date) AS year,
        MONTH(o.order_date) AS month,
        COUNT(o.order_id) AS cr_month_orders,
        LAG(COUNT(o.order_id), 1) OVER (
            PARTITION BY o.restaurant_id 
            ORDER BY YEAR(o.order_date), MONTH(o.order_date)
        ) AS prev_month_orders
    FROM orders o
    JOIN deliveries d ON o.order_id = d.order_id
    WHERE d.delivery_status = 'Delivered'
    GROUP BY o.restaurant_id, YEAR(o.order_date), MONTH(o.order_date)
)

SELECT
    restaurant_id,
    month,
    prev_month_orders,
    cr_month_orders,
    ROUND(
        (cr_month_orders - prev_month_orders) / prev_month_orders * 100, 2
    ) AS growth_ratio
FROM growth_ratio
WHERE prev_month_orders IS NOT NULL;
```

### 12. Customer Segmentation: 
-- Customer Segmentation: Segment customers into 'Gold' or 'Silver' groups based on their total spending 
-- compared to the average order value (AOV). If a customer's total spending exceeds the AOV, 
-- label them as 'Gold'; otherwise, label them as 'Silver'. Write an SQL query to determine each segment's 
-- total number of orders and total revenue

```sql
SELECT 
    cx_category,
    SUM(total_orders) AS total_orders,
    SUM(total_spent) AS total_revenue
FROM (
    SELECT 
        customer_id,
        SUM(total_amount) AS total_spent,
        COUNT(order_id) AS total_orders,
        CASE 
            WHEN SUM(total_amount) > (
                SELECT AVG(total_amount) FROM orders
            ) THEN 'Gold'
            ELSE 'Silver'
        END AS cx_category
    FROM orders
    GROUP BY customer_id
) AS t1
GROUP BY cx_category;
```

### 13. Rider Monthly Earnings: 
-- Calculate each rider's total monthly earnings, assuming they earn 8% of the order amount.

```sql
SELECT 
    d.rider_id,
    DATE_FORMAT(o.order_date, '%m-%y') AS month,
    SUM(o.total_amount) AS revenue,
    ROUND(SUM(o.total_amount) * 0.08, 2) AS riders_earning
FROM orders o
JOIN deliveries d ON o.order_id = d.order_id
GROUP BY d.rider_id, month
ORDER BY d.rider_id, month;
```

### 14. Q.15 Order Frequency by Day: 
-- Analyze order frequency per day of the week and identify the peak day for each restaurant.

```sql
SELECT *
FROM (
    SELECT 
        r.restaurant_name,
        DAYNAME(o.order_date) AS day,
        COUNT(o.order_id) AS total_orders,
        RANK() OVER (PARTITION BY r.restaurant_name ORDER BY COUNT(o.order_id) DESC) AS rank
    FROM orders o
    JOIN restaurants r ON o.restaurant_id = r.restaurant_id
    GROUP BY r.restaurant_name, DAYNAME(o.order_date)
) AS t
WHERE rank = 1;
```

### 15. Customer Lifetime Value (CLV): 
-- Calculate the total revenue generated by each customer over all their orders.

```sql
SELECT 
    o.customer_id,
    c.customer_name,
    SUM(o.total_amount) AS CLV
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY o.customer_id, c.customer_name;
```

### 16. Monthly Sales Trends: 
-- Identify sales trends by comparing each month's total sales to the previous month.

```sql
WITH monthly_sales AS (
    SELECT 
        YEAR(order_date) AS year,
        MONTH(order_date) AS month,
        SUM(total_amount) AS total_sale
    FROM orders
    GROUP BY YEAR(order_date), MONTH(order_date)
)
SELECT 
    year,
    month,
    total_sale,
    LAG(total_sale) OVER(ORDER BY year, month) AS prev_month_sale
FROM monthly_sales;
```

### 17. Rank each city based on the total revenue for last year 2023
```sql
SELECT 
    r.city,
    SUM(o.total_amount) AS total_revenue,
    RANK() OVER (ORDER BY SUM(o.total_amount) DESC) AS city_rank
FROM orders o
JOIN restaurants r
    ON o.restaurant_id = r.restaurant_id
WHERE YEAR(o.order_date) = 2023
GROUP BY r.city;
```

## Conclusion

This project highlights my ability to handle complex SQL queries and provides solutions to real-world business problems in the context of a food delivery service like Zomato. The approach taken here demonstrates a structured problem-solving methodology, data manipulation skills, and the ability to derive actionable insights from data.

## Notice 
All customer names and data used in this project are computer-generated using AI and random functions. They do not represent real data associated with Zomato or any other entity. This project is solely for learning and educational purposes, and any resemblance to actual persons, businesses, or events is purely coincidental.
