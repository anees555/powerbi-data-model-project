# Data Dictionary

Full column-level reference for every table in the model. See the main [README](../README.md) for the ER diagram and relationship overview.

---

## Fact Tables

### fact_sales
Grain: one row per order line item

| Column | Data Type | Key | Description |
|---|---|---|---|
| order_id | string | | Order this line item belongs to |
| quantity | int64 | | Units sold on this line |
| price | double | | Unit selling price |
| cost | double | | Unit cost (used to derive margin) |
| discount | double | | Discount applied to the line |
| line_total | double | | Net revenue for the line (price × quantity, less discount) |
| order_date | date | FK → dim_date | Date the order was placed (active relationship) |
| customer_id | int64 | FK → dim_customer | Customer who placed the order |
| product_key | int64 | FK → dim_product | Product sold |
| flag_key | int64 | FK → dim_order_flag | Order channel/status/priority combination |
| ship_to_city_key | int64 | FK → dim_geo | Shipping destination city (active relationship) |
| bill_to_city_key | int64 | FK → dim_geo | Billing city (inactive relationship — use `USERELATIONSHIP()` to activate) |

### fact_order_process
Grain: one row per order (fulfillment cycle)

| Column | Data Type | Key | Description |
|---|---|---|---|
| order_id | string | | Order identifier |
| customer_id | int64 | FK → dim_customer | Customer who placed the order |
| order_date | date | FK → dim_date | Date order was placed (active relationship) |
| ship_date | date | FK → dim_date | Date order shipped (inactive relationship) |
| delivery_date | date | FK → dim_date | Date order was delivered (inactive relationship) |
| invoice_date | date | FK → dim_date | Date invoice was issued (inactive relationship) |
| pay_date | date | FK → dim_date | Date payment was received (inactive relationship) |
| order_to_pay | int64 | | Calculated: `DATEDIFF(order_date, pay_date, DAY)` — days from order to payment |

### fact_inventory
Grain: one row per product per month

| Column | Data Type | Key | Description |
|---|---|---|---|
| product_key | int64 | FK → dim_product | Product being stocked |
| month | date | FK → dim_date | Month of the stock snapshot (unpivoted from wide monthly source columns) |
| units | int64 | | Units in stock for that product/month |

### fact_promotion_coverage
Grain: one row per campaign–product pairing (factless fact / bridge table)

| Column | Data Type | Key | Description |
|---|---|---|---|
| campaign_key | int64 | FK → dim_campaign | Campaign covering this product |
| product_key | int64 | FK → dim_product | Product covered by the campaign |

No numeric measures — this table exists purely to record which products each campaign promoted (split out from a comma-separated SKU list in the source data).

### fact_campaign_spend
Grain: one row per campaign per day

| Column | Data Type | Key | Description |
|---|---|---|---|
| campaign_key | int64 | FK → dim_campaign | Campaign this spend belongs to |
| date | date | FK → dim_date (bi-directional) | Day of activity |
| impressions | int64 | | Ad impressions served |
| clicks | int64 | | Ad clicks recorded |
| spend | double | | Amount spent |

### fact_sales_target
Grain: one row per date

| Column | Data Type | Key | Description |
|---|---|---|---|
| date | date | FK → dim_date (bi-directional) | Date the target applies to |
| target_revenue | int64 | | Revenue target for that date |

---

## Dimension Tables

### dim_customer
| Column | Data Type | Key | Description |
|---|---|---|---|
| customer_id | int64 | PK | Unique customer identifier |
| customer_name | string | | Customer/company name |
| segment | string | | Customer segment/category |
| account_manager | string | | Assigned account manager |
| payment_terms | string | | Agreed payment terms |
| contact_name | string | | Primary contact person |
| contact_email | string | | Primary contact email |
| credit_limit | int64 | | Approved credit limit |
| phone | string | | Contact phone number |
| street | string | | Street address |
| city | string | | City |
| region | string | | Region — also drives row-level security |

### dim_product
| Column | Data Type | Key | Description |
|---|---|---|---|
| product_key | int64 | PK (surrogate) | Generated key, since source lacked a clean integer key |
| product_code | string | | Business/SKU code |
| product_name | string | | Product name |
| brand | string | | Brand |
| subcategory | string | | Product subcategory |
| category | string | | Product category (text-standardized to proper case) |
| price | double | | Unit price |
| supplier | string | | Primary supplier |

### dim_geo
| Column | Data Type | Key | Description |
|---|---|---|---|
| geo_key | int64 | PK (surrogate) | Generated key |
| city | string | | City name |
| region | string | | Region the city belongs to |

### dim_date
| Column | Data Type | Key | Description |
|---|---|---|---|
| Date | date | PK | Calendar date, generated via `CALENDARAUTO()` |
| year | int64 | | Calculated: `YEAR(Date)` |
| month | int64 | | Calculated: `MONTH(Date)` |

### dim_campaign
| Column | Data Type | Key | Description |
|---|---|---|---|
| campaign_key | int64 | PK (surrogate) | Generated key |
| campaign_name | string | | Campaign name |
| channel | string | | Marketing channel used |
| start_date | date | | Campaign start date |
| end_date | date | | Campaign end date |
| budget | int64 | | Allocated campaign budget |

### dim_order_flag
| Column | Data Type | Key | Description |
|---|---|---|---|
| flag_key | int64 | PK (surrogate) | Generated key representing a unique channel/status/priority combination |
| channel | string | | Order channel name |
| status | string | | Order status |
| priority | string | | Order priority level |
| channel_code | int64 | | Underlying channel code from source system |

---

## Support / Security Table

### security
| Column | Data Type | Key | Description |
|---|---|---|---|
| user_email | string | | Email of a Power BI user |
| region | string | | Region that user is restricted to via the 'Regional Access' RLS role |

---

## Measures Table (`_measures`)

A dedicated disconnected table holding all DAX measures, following the common Power BI best practice of separating measures from data tables for easier organization in the fields list.

| Measure | DAX | Description |
|---|---|---|
| total_sales | `SUM(fact_sales[line_total])` | Total revenue |
| total_orders | `DISTINCTCOUNT(fact_sales[order_id])` | Count of distinct orders |
| total_active_customers | `DISTINCTCOUNT(fact_sales[customer_id])` | Customers who have placed at least one order |
| base_total_customers | `COUNT(dim_customer[customer_id])` | All customers in the master list, regardless of order activity |
| avg_order_to_pay | `AVERAGE(fact_order_process[order_to_pay])` | Average days from order placement to payment |
