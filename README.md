# 🛒 Analyzing E-Commerce Business Performance with SQL

An end-to-end e-commerce business performance analysis using PostgreSQL and SQL queries to evaluate annual customer growth, product category performance, and payment method trends.

---

## 📌 Project Overview

This project analyzes the multi-year business performance of a Brazilian e-commerce platform using relational SQL database techniques. By examining 99,441 order transactions spanning from 2016 to 2018, the analysis identifies critical patterns in customer acquisition and retention, product revenue performance, order cancellation drivers, and shifting payment preferences.

The primary objective of this project is to convert raw e-commerce operational data into actionable business intelligence that helps company stakeholders optimize marketing strategies, streamline supply chain fulfillment, and improve customer retention.

---

## 🎯 Business Problem

To measure and monitor e-commerce business performance effectively, management needs clarity on customer behavior, product profitability, and transaction methods over time. 

This project answers the following core business questions:
- **Customer Activity & Retention:** How is monthly active user (MAU) volume growing year-over-year, and are newly acquired customers making repeat purchases?
- **Product Category Quality:** Which product categories generate the highest total revenue each year, and which categories experience the highest rate of order cancellations?
- **Payment Preferences:** What payment methods do customers prefer, and how have payment preferences shifted between 2016 and 2018?
- **Business Strategy:** What strategic decisions can management implement to improve customer retention, boost revenue, and reduce cancellations?

---

## 📊 Dataset

The project utilizes the **Brazilian E-Commerce Public Dataset by Olist** (provided via Rakamin Academy), containing **99,441 order records** from 2016 to 2018 stored across 8 relational tables.

### Table Schema & Descriptions

| Table Name | Description | Key Fields |
| :--- | :--- | :--- |
| `customers_dataset` | Customer demographics and location | `customer_id`, `customer_unique_id`, `customer_city`, `customer_state` |
| `orders_dataset` | Order lifecycle and status tracking | `order_id`, `customer_id`, `order_status`, `order_purchase_timestamp` |
| `order_items_dataset` | Itemized order pricing and logistics costs | `order_id`, `order_item_id`, `product_id`, `seller_id`, `price`, `fright_value` |
| `product_dataset` | Product category and physical specifications | `product_id`, `product_category_name`, `product_weight_g`, `product_length_cm` |
| `order_payments_dataset` | Payment transaction types and values | `order_id`, `payment_type`, `payment_installments`, `payment_value` |
| `sellers_dataset` | Seller location data | `seller_id`, `seller_city`, `seller_state` |
| `geolocation_dataset` | Zip code geographical coordinates | `geolocation_zip_code_prefix`, `geolocation_lat`, `geolocation_lng` |
| `order_reviews_dataset` | Customer review ratings and comments | `review_id`, `order_id`, `review_score`, `review_creation_date` |

---

## 🛠️ Tools & Technologies

| Category | Tools / Technologies |
| :--- | :--- |
| **Database** | PostgreSQL |
| **Querying & Analysis** | SQL (Data Definition Language, Aggregations, CTEs, Window Functions) |
| **Database Management Tool** | pgAdmin 4 |
| **Data Visualization & Charting** | Microsoft Excel |
| **Development Environment** | VS Code |
| **Version Control** | Git / GitHub |

---

## 🔄 Project Workflow

```
Raw CSV Datasets
       ↓
Database & Schema Design (PostgreSQL / pgAdmin)
       ↓
Table Creation & Primary/Foreign Key Constraints (DDL)
       ↓
Data Import & Referential Integrity Validation
       ↓
Exploratory SQL Queries & Metric Extraction
 ┌─────┴─────────────────────┬──────────────────────────┐
 ↓                           ↓                          ↓
Annual Customer Activity   Annual Product Category    Annual Payment Type
 Growth Analysis            Quality Analysis           Usage Analysis
 └─────┬─────────────────────┴──────────────────────────┘
       ↓
Data Visualization & Charting (Excel)
       ↓
Business Insights & Strategic Recommendations
```

1. **Database Setup & Modeling:** Created the `ecommerce_miniproject` PostgreSQL database, defined table schemas, imported raw datasets, and assigned Primary Keys and Foreign Keys to establish relational integrity.
2. **Data Cleaning & Filtering:** Filtered for valid order statuses (`delivered`, `canceled`) and excluded anomalous historical entries (e.g., partial 2020 records).
3. **Exploratory SQL Analysis:** Executed SQL scripts using aggregate functions, subqueries, CTEs, and window functions to compute customer growth metrics, top category revenues, cancellation counts, and payment breakdowns.
4. **Insight Generation & Reporting:** Synthesized query results into key business insights and actionable strategic recommendations.

---

## 🧹 Data Cleaning & Preparation

The following data preparation and validation steps were executed prior to analysis:
- **Schema & Type Standardization:** Defined precise data types for dates (`timestamp`), financial figures (`decimal`/`numeric`), metrics (`integer`), and identifiers (`varchar`).
- **Relational Integrity Enforcement:** Applied Primary Keys on `customer_id`, `seller_id`, `product_id`, and `order_id`, and configured Foreign Key references between orders, items, payments, reviews, sellers, and products.
- **Order Status Filtering:** Filtered revenue calculations to include only orders with status `delivered` to ensure accurate financial reporting.
- **Anomaly Cleanup:** Identified and deleted incomplete data entries for year 2020 in product category tables to avoid skewing multi-year historical trends.
- **Null & Category Validation:** Grouped missing or undefined payment values under `not_defined` to maintain aggregation accuracy.

---

## 🔍 Data Analysis

The analytical execution was structured into three main modules using PostgreSQL.

### 1. Annual Customer Activity Growth Analysis
Used Common Table Expressions (CTEs) and conditional logic to evaluate customer acquisition, active user volume, and purchase frequency.

- **Monthly Active Users (MAU):** Computed average monthly active unique customers (`customer_unique_id`) per year.
- **New Customer Acquisition:** Identified the initial purchase year for each customer to track brand adoption.
- **Repeat Purchase Behavior:** Aggregated customer order counts per year using `HAVING COUNT(order_id) > 1` to measure repeat orders.
- **Purchase Frequency:** Derived the average annual order count per unique customer.

```sql
WITH cte_mau AS (
    SELECT year, FLOOR(AVG(customer_total)) AS avg_mau
    FROM (
        SELECT 
            date_part('year', od.order_purchase_timestamp) AS year,
            date_part('month', od.order_purchase_timestamp) AS month,
            COUNT(DISTINCT cd.customer_unique_id) AS customer_total
        FROM orders_dataset AS od
        JOIN customers_dataset AS cd ON cd.customer_id = od.customer_id
        GROUP BY 1, 2
    ) AS sub
    GROUP BY 1
),
cte_new_cust AS (
    SELECT year, COUNT(customer_unique_id) AS total_new_customer
    FROM (
        SELECT MIN(date_part('year', od.order_purchase_timestamp)) AS year, cd.customer_unique_id
        FROM orders_dataset AS od
        JOIN customers_dataset AS cd ON cd.customer_id = od.customer_id
        GROUP BY 2
    ) AS sub
    GROUP BY 1
)
SELECT mau.year, avg_mau, total_new_customer, total_customer_repeat, avg_frequency
FROM cte_mau AS mau
JOIN cte_new_cust AS nc ON mau.year = nc.year
JOIN cte_repeat_order AS ro ON nc.year = ro.year
JOIN cte_frequency AS f ON ro.year = f.year;
```

### 2. Annual Product Category Quality Analysis
Leveraged SQL Window Functions (`RANK() OVER`) and multi-table `JOIN` operations to identify top revenue-generating categories and categories with high cancellation volumes.

- **Total Revenue Calculation:** Summarized product price plus freight charges (`price + fright_value`) for delivered orders.
- **Category Ranking:** Applied `RANK()` partitioned by shipping year to dynamically isolate top revenue categories and most canceled categories per year.

```sql
CREATE TABLE top_product_category AS
SELECT year, top_category, product_revenue
FROM (
    SELECT
        date_part('year', shipping_limit_date) AS year,
        pd.product_category_name AS top_category,
        SUM(oid.price + oid.fright_value) AS product_revenue,
        RANK() OVER (PARTITION BY date_part('year', shipping_limit_date)
                     ORDER BY SUM(oid.price + oid.fright_value) DESC) AS ranking
    FROM orders_dataset AS od 
    JOIN order_items_dataset AS oid ON od.order_id = oid.order_id
    JOIN product_dataset AS pd ON oid.product_id = pd.product_id
    WHERE od.order_status LIKE 'delivered'
    GROUP BY 1, 2
) AS sub
WHERE ranking = 1;
```

### 3. Annual Payment Type Usage Analysis
Utilized pivot aggregation queries (`CASE WHEN`) to track shifts in payment method preferences across 2016, 2017, and 2018.

```sql
SELECT
    payment_type,
    SUM(CASE WHEN year = 2016 THEN total ELSE 0 END) AS "2016",
    SUM(CASE WHEN year = 2017 THEN total ELSE 0 END) AS "2017",
    SUM(CASE WHEN year = 2018 THEN total ELSE 0 END) AS "2018",
    SUM(total) AS sum_payment_type_usage
FROM (
    SELECT 
        date_part('year', od.order_purchase_timestamp) AS year,
        opd.payment_type,
        COUNT(opd.payment_type) AS total
    FROM orders_dataset AS od
    JOIN order_payments_dataset AS opd ON od.order_id = opd.order_id
    GROUP BY 1, 2
) AS sub
GROUP BY 1
ORDER BY 2 DESC;
```

---

## 📈 Key KPIs

| KPI Metric | 2016 | 2017 | 2018 | Overall / Total |
| :--- | :---: | :---: | :---: | :---: |
| **Total Revenue (BRL)** | 46,653.74 | 6,921,535.24 | 8,451,584.77 | **15,419,773.75** |
| **Average Monthly Active Users (MAU)** | 108 | 3,694 | 5,338 | **—** |
| **New Customer Acquisition** | 326 | 43,708 | 52,062 | **96,096** |
| **Repeat Customers** | 3 | 1,256 | 1,167 | **2,426** |
| **Average Order Frequency (Orders/Cust)** | 1.009 | 1.032 | 1.024 | **1.022** |
| **Top Revenue Product Category** | `furniture_decor` | `bed_bath_table` | `health_beauty` | **—** |
| **Top Category Revenue (BRL)** | 6,899.35 | 569,964.78 | 877,065.73 | **1,453,929.86** |
| **Total Order Cancellations** | 26 | 265 | 334 | **625** |
| **Most Canceled Product Category** | `toys` | `sports_leisure` | `health_beauty` | **—** |
| **Dominant Payment Method** | Credit Card (258) | Credit Card (34,568) | Credit Card (41,969) | **Credit Card (76,795)** |

---

## 💡 Key Insights

### Insight 1 — Strong Top-Line Revenue Growth Driven by Customer Acquisition
- **Finding:** Total e-commerce revenue expanded rapidly from 2016 to 2018.
- **Evidence:** Annual revenue rose from 46,653.74 BRL in 2016 to 6,921,535.24 BRL in 2017, reaching 8,451,584.77 BRL in 2018 (+22.1% YoY growth in 2018).
- **Business Meaning:** Overall market demand for the platform grew significantly, validating overall commercial expansion.

### Insight 2 — Low Customer Retention and High One-Time Purchase Rates
- **Finding:** Despite massive new customer acquisition, customer retention remains extremely low.
- **Evidence:** New customer sign-ups grew from 43,708 in 2017 to 52,062 in 2018. However, repeat customers dropped from 1,256 (2.87% of new customers) in 2017 to 1,167 (2.24%) in 2018. The average annual order frequency remained static at ~1.02 orders per customer.
- **Business Meaning:** Over 97% of buyers treat the platform as a one-time transaction channel. The platform relies heavily on acquisition paid media rather than organic retention.

### Insight 3 — Shift in Category Demand Toward Health & Beauty
- **Finding:** Leading product categories shifted annually, with `health_beauty` becoming the primary revenue driver in 2018.
- **Evidence:** Top product categories by year were `furniture_decor` (2016), `bed_bath_table` (2017: 569,964.78 BRL), and `health_beauty` (2018: 877,065.73 BRL).
- **Business Meaning:** High-margin personal care and beauty products expanded rapidly, presenting a strong category growth candidate for targeted promotional campaigns.

### Insight 4 — High Cancellation Volume in Fast-Selling Categories
- **Finding:** Categories generating high order volumes also suffered the highest absolute order cancellations.
- **Evidence:** Total order cancellations increased from 265 in 2017 to 334 in 2018. In 2018, `health_beauty` was simultaneously the highest revenue generator (877,065.73 BRL) and the most canceled product category (27 cancellations).
- **Business Meaning:** High demand for fast-moving items exposes logistics, inventory stock-out, or delivery time delays among sellers.

### Insight 5 — Credit Card Dominance and Rapid Debit Card Growth
- **Finding:** Credit card remains the preferred payment option, while debit card usage experienced a significant upward spike.
- **Evidence:** Credit card accounted for 76,795 total payments (78.3%), followed by Boleto with 19,784 payments (20.2%). Debit card usage grew by +161.8% from 422 payments in 2017 to 1,105 payments in 2018. Voucher usage declined from 3,027 in 2017 to 2,725 in 2018 (-10.0%).
- **Business Meaning:** Customers favor credit cards for installment flexibility, while promotion of debit card options successfully converted cash/voucher buyers into instant digital payment users.

---

## 🚀 Business Recommendations

### Recommendation 1 — Launch Automated Customer Retention & Re-Engagement Programs
**Action:** Implement automated email/SMS campaigns, post-purchase coupon incentives, and a customer loyalty points program for second-time buyers.  
**Reason:** Customer repeat rate dropped to 2.24% in 2018, with average order frequency stuck at ~1.02 orders/year.  
**Expected Impact:** Raising repeat purchase rates by 3–5% could generate substantial additional revenue without increasing customer acquisition costs (CAC).

### Recommendation 2 — Optimize Inventory & Seller Fulfillment for High-Demand Categories
**Action:** Establish inventory stock reservation protocols and seller SLA monitoring for top-performing categories like `health_beauty` and `bed_bath_table`.  
**Reason:** `health_beauty` suffered the highest number of order cancellations (27) in 2018 due to stock shortages or shipping delays during peak demand.  
**Expected Impact:** Could reduce overall order cancellations by 15–20% and protect merchant revenue.

### Recommendation 3 — Capitalize on Debit Card Growth & Refine Voucher Strategies
**Action:** Partner with digital banking providers to offer instant cashback for debit transactions, while re-evaluating voucher discount structures.  
**Reason:** Debit card usage expanded +161.8% in 2018, whereas voucher transactions declined by 10.0%.  
**Expected Impact:** Could accelerate conversion rates among non-credit-card users and improve payment processing speed.

---

## 🖼️ Dashboard & Visualization

Analytical query outputs and trend charts were created in Excel to visualize historical business performance.

| Visualization Description | File / Asset Path |
| :--- | :--- |
| **Entity Relationship Diagram (ERD)** | ![ERD Schema](asset/gambar_1_ERD.png) |
| **Monthly Active Users (MAU) & Customer Growth** | ![MAU Trend](asset/activity.png) |
| **Annual Product Category Quality & Revenue** | ![Product Category Revenue](asset/produk.png) |
| **Annual Payment Method Distribution** | ![Payment Method Usage](asset/payment.png) |

---

## 📁 Project Structure

```
Analyzing-eCommerce-Business-Performance-with-SQL/
│
├── sql_query/
│   ├── Create Table.sql                      # DDL scripts for table schema, PKs, and FKs
│   ├── Annual Customer Activity Growth.sql   # SQL queries for MAU, new/repeat customers, and frequency
│   ├── Annual Product Category Quality.sql  # SQL queries for revenue, cancellations, and category ranking
│   └── Annual Payment Type Usage.sql         # SQL queries for payment method breakdowns
│
├── asset/
│   ├── gambar_1_ERD.png                      # Database Entity Relationship Diagram
│   ├── activity.png                          # Customer activity & growth query result table
│   ├── produk.png                            # Product category performance query result table
│   ├── payment.png                           # Payment method usage query result table
│   └── [chart_graphics].png                  # Performance visualization charts
│
└── README.md                                 # Project documentation
```

---

## ⚙️ How to Run / Reproduce the Project

1. **Clone the Repository:**
   ```bash
   git clone https://github.com/your-username/Analyzing-eCommerce-Business-Performance-with-SQL.git
   ```
2. **Database Setup (PostgreSQL & pgAdmin):**
   - Open pgAdmin 4 and create a new database named `ecommerce_miniproject`.
   - Open the Query Tool and execute `sql_query/Create Table.sql` to build tables and establish Primary/Foreign Key constraints.
3. **Import Data:**
   - Right-click each table in pgAdmin and import the corresponding CSV data files.
4. **Execute Analytical SQL Queries:**
   - Run `sql_query/Annual Customer Activity Growth.sql` to generate customer metrics.
   - Run `sql_query/Annual Product Category Quality.sql` to extract category revenue and cancellation rankings.
   - Run `sql_query/Annual Payment Type Usage.sql` to evaluate payment method distributions.

---

## 🛠️ Skills Demonstrated

### Technical Skills
- **SQL Data Definition (DDL):** `CREATE TABLE`, `ALTER TABLE`, Primary Key & Foreign Key constraints.
- **SQL Querying & Analysis:** Multi-table `JOIN`, `GROUP BY`, `HAVING`, `CASE WHEN` conditional aggregations.
- **Advanced SQL:** Window Functions (`RANK() OVER (PARTITION BY ... ORDER BY ...)`), Common Table Expressions (`WITH` CTEs), Nested Subqueries.
- **Data Modeling & Validation:** Relational schema design, entity relationship mapping, anomaly filtering, data type coercion.
- **Database Management:** PostgreSQL, pgAdmin 4.

### Analytical & Business Skills
- **E-Commerce Metrics Analysis:** MAU, Customer Acquisition Cost dynamics, Repeat Purchase Rate, Order Frequency.
- **Category & Product Performance:** Revenue contribution breakdown, stock-out & cancellation tracking.
- **Financial & Payment Analytics:** Payment method share, installment trend evaluation.
- **Data-Driven Decision Making:** Formulating evidence-based business recommendations.

---

## 🎓 Key Learnings

Completing this project provided several key practical insights into real-world data analytics:
- **Navigating Relational Schemas:** Designing and linking 8 relational tables reinforced the importance of enforcing primary and foreign key constraints to maintain data integrity across multi-table JOIN queries.
- **Isolating Metric Anomalies:** Real-world datasets often contain partial date ranges or missing records (such as incomplete 2020 shipping dates), requiring explicit filtering to prevent biased trend analysis.
- **Balancing Growth vs. Retention:** Discovered that rapid acquisition of new customers does not equal long-term business sustainability if customer retention is neglected (~97% single-order rate).
- **Translating Code into Business Strategy:** Learned to convert complex SQL query results into clear financial KPIs and practical recommendations that directly support decision-making for business stakeholders.

---

## 📌 Conclusion

This e-commerce business performance analysis demonstrated significant top-line expansion between 2016 and 2018, with total revenue reaching **15.42 million BRL** and annual Monthly Active Users (MAU) increasing to **5,338**. However, the analysis uncovered a critical operational vulnerability: **over 97% of acquired customers made only a single purchase**.

By optimizing product fulfillment in fast-growing categories like `health_beauty`, introducing automated retention campaigns, and leveraging payment preferences like credit cards and debit cards, the platform can transform rapid customer acquisition into sustainable, repeatable long-term profitability.
