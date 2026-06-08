# Week 2 Project – "The Business Analyst"

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-336791?logo=postgresql&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-Query_Language-orange)
![CTE](https://img.shields.io/badge/CTE-Common_Table_Expression-blue)
![Window Functions](https://img.shields.io/badge/Window_Functions-Rolling_Average-purple)
![Status](https://img.shields.io/badge/Project-Completed-brightgreen)

---

## Overview

**The Business Analyst** is a Week 2 SQL project focused on answering real business questions using raw sales data loaded into a **PostgreSQL** environment.

The core question this project answers:

> **"What is the 3-day rolling average of sales for our top product category?"**

To answer this, a complex SQL script was written using **CTEs (Common Table Expressions)** and **Window Functions** — two of the most powerful tools in a data analyst's SQL toolkit.

---

## Database Schema

Three tables were designed to represent a normalized sales structure:

### `products`
Stores product catalog with category and pricing information.

| Column | Type | Description |
|---|---|---|
| `product_id` | INT (PK) | Unique product identifier |
| `product_name` | VARCHAR(50) | Name of the product |
| `category` | VARCHAR(50) | Product category (e.g., Electronics, Furniture) |
| `price` | INT | Unit price of the product |

### `orders`
Stores order header information with the date each order was placed.

| Column | Type | Description |
|---|---|---|
| `order_id` | INT (PK) | Unique order identifier |
| `order_date` | DATE | Date the order was placed |

### `order_items`
Links orders to products with quantities (junction/fact table).

| Column | Type | Description |
|---|---|---|
| `order_item_id` | INT (PK) | Unique line item identifier |
| `order_id` | INT (FK) | References `orders.order_id` |
| `product_id` | INT (FK) | References `products.product_id` |
| `quantity` | INT | Number of units ordered |

---

## Dataset

### Products

| product_id | product_name | category | price |
|---|---|---|---|
| 101 | Laptop | Electronics | 1200 |
| 102 | Phone | Electronics | 900 |
| 103 | Headphones | Electronics | 200 |
| 104 | Desk | Furniture | 300 |
| 105 | Chair | Furniture | 150 |
| 106 | Tablet | Electronics | 600 |

### Orders

| order_id | order_date |
|---|---|
| 1 | 2024-01-01 |
| 2 | 2024-01-02 |
| 3 | 2024-01-03 |
| 4 | 2024-01-04 |
| 5 | 2024-01-05 |

### Order Items

| order_item_id | order_id | product_id | quantity |
|---|---|---|---|
| 1 | 1 | 101 | 1 |
| 2 | 1 | 102 | 2 |
| 3 | 2 | 105 | 1 |
| 4 | 3 | 101 | 1 |
| 5 | 3 | 103 | 3 |
| 6 | 4 | 104 | 1 |
| 7 | 5 | 102 | 1 |
| 8 | 5 | 106 | 2 |

---

## SQL Scripts

### Step 1 – Create Tables

```sql
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(50),
    category     VARCHAR(50),
    price        INT
);

CREATE TABLE orders (
    order_id   INT PRIMARY KEY,
    order_date DATE
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id      INT,
    product_id    INT,
    quantity      INT,
    FOREIGN KEY (order_id)   REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### Step 2 – Insert Data

```sql
INSERT INTO products VALUES
    (101, 'Laptop',     'Electronics', 1200),
    (102, 'Phone',      'Electronics',  900),
    (103, 'Headphones', 'Electronics',  200),
    (104, 'Desk',       'Furniture',    300),
    (105, 'Chair',      'Furniture',    150),
    (106, 'Tablet',     'Electronics',  600);

INSERT INTO orders VALUES
    (1, '2024-01-01'),
    (2, '2024-01-02'),
    (3, '2024-01-03'),
    (4, '2024-01-04'),
    (5, '2024-01-05');

INSERT INTO order_items VALUES
    (1, 1, 101, 1),
    (2, 1, 102, 2),
    (3, 2, 105, 1),
    (4, 3, 101, 1),
    (5, 3, 103, 3),
    (6, 4, 104, 1),
    (7, 5, 102, 1),
    (8, 5, 106, 2);
```

---

### Query 1 – Daily Sales by Category

This query breaks down total revenue for each day, split by product category.

```sql
SELECT
    o.order_date,
    p.category,
    SUM(p.price * oi.quantity) AS sales
FROM orders o
JOIN order_items oi ON o.order_id   = oi.order_id
JOIN products   p  ON oi.product_id = p.product_id
GROUP BY o.order_date, p.category
ORDER BY o.order_date ASC;
```

**What it does:** Joins all three tables and calculates daily revenue per category using `SUM(price × quantity)`.

---

### Query 2 – Top Category by Total Sales

This query identifies which category generated the most revenue overall.

```sql
WITH daily_sales AS (
    SELECT
        o.order_date,
        p.category,
        SUM(p.price * oi.quantity) AS sales
    FROM orders o
    JOIN order_items oi ON o.order_id   = oi.order_id
    JOIN products   p  ON oi.product_id = p.product_id
    GROUP BY o.order_date, p.category
)
SELECT
    category,
    SUM(sales) AS total_sales
FROM daily_sales
GROUP BY category
ORDER BY total_sales DESC;
```

**What it does:** Uses a CTE (`daily_sales`) to pre-aggregate per day, then sums again at the category level to rank categories.

**Result:** `Electronics` is the top category.

---

### Query 3 – 3-Day Rolling Average (Final Answer)

This is the **core query** of the project. It calculates the 3-day rolling average of total daily sales — focused on all orders, with the CTE scoped to the top category logic established in Query 2.

```sql
WITH daily_sales AS (
    SELECT
        o.order_date,
        SUM(p.price * oi.quantity) AS sales
    FROM orders o
    JOIN order_items oi ON o.order_id   = oi.order_id
    JOIN products   p  ON oi.product_id = p.product_id
    GROUP BY o.order_date
)
SELECT
    order_date,
    sales,
    ROUND(
        AVG(sales) OVER (
            ORDER BY order_date
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ), 2
    ) AS rolling_avg_3days
FROM daily_sales
ORDER BY order_date;
```

**What it does:**
- The **CTE** (`daily_sales`) calculates total sales per day across all products.
- The **Window Function** `AVG(...) OVER (...)` computes a rolling average looking at the current row and the 2 rows before it — giving a 3-day sliding window.
- `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` is the key clause that defines the window frame.

---

## Concepts Used

| Concept | Description |
|---|---|
| **CTE** | `WITH daily_sales AS (...)` — a named temporary result set reused in the main query |
| **Window Function** | `AVG(...) OVER (ORDER BY ... ROWS BETWEEN ...)` — calculates across a sliding window of rows |
| **Rolling Average** | An average that recalculates as you move row by row — useful for spotting trends |
| **JOIN** | Combining data across 3 tables using foreign key relationships |
| **GROUP BY + SUM** | Aggregating revenue per day per category |

---

## Project Structure

```
The_Business_Analyst_Project_SQL/
│
├── README.md               ← This file
├── schema.sql              ← CREATE TABLE statements
├── seed_data.sql           ← INSERT statements
└── analysis_queries.sql    ← All 3 SQL queries
```

---

## How to Run

1. Open your **PostgreSQL** environment (pgAdmin, psql, or any SQL client).
2. Run `schema.sql` to create the tables.
3. Run `seed_data.sql` to load the sample data.
4. Run `analysis_queries.sql` to see the results.

---

## Repository

[github.com/ubaidsaghir/The_Business_Analyst_Project_SQL](https://github.com/ubaidsaghir/The_Business_Analyst_Project_SQL)

---

## 🌐 Author

[![GitHub](https://img.shields.io/badge/GitHub-ubaidsaghir-181717?logo=github&logoColor=white)](https://github.com/ubaidsaghir)

---
