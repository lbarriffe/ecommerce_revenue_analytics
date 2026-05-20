# Olist Revenue Case Study (SQL Script)

Tools used:
- PostgreSQL
- Tableau

Data source: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

Write-Up: [Analysis of Revenue Risk, Customer Value, and Delivery Performance](analysis.md)

## Table of Contents
1.[Import](README.md#Data-Import)   
2.[Data Exploration](README.md#Data-Exploration)   
3.[Data Cleaning and Normalization](README.md#Data-Cleaning-and-Normalization)   
4.[Feature Engineering](README.md#Feature-Engineering)   
5.[Dashboard](README.md#Dashboard)   

---

## Olist Revenue Analysis
Purpose:
This analysis explores revenue performance, customer value, and operational efficiency in the Olist Brazilian E-commerce Dataset.
### Methods
- SQL - Data cleaning and feature engineering
- Tableau - Dashboard

## Data Import
```sql
DROP SCHEMA IF EXISTS staging CASCADE;
CREATE SCHEMA IF NOT EXISTS staging;

CREATE TABLE staging.customers(
customer_id TEXT,
customer_unique_id TEXT,
customer_zip_code_prefix TEXT,
customer_city TEXT,
customer_state TEXT
);

CREATE TABLE staging.geolocation(
geolocation_zip_code_prefix TEXT,
geolocation_lat TEXT,
geolocation_lng TEXT,
geolocation_city TEXT,
geolocation_state TEXT
);

CREATE TABLE staging.order_items(
order_id TEXT,
order_item_id TEXT,
product_id TEXT,
seller_id TEXT,
shipping_limit_date TEXT,
price TEXT,
freight_value TEXT
);

CREATE TABLE staging.order_payments(
order_id TEXT,
payment_sequential TEXT,
payment_type TEXT,
payment_installments TEXT,
payment_value TEXT
);

CREATE TABLE staging.order_reviews(
review_id TEXT,
order_id TEXT,
review_score TEXT,
review_comment_title TEXT,
review_comment_message TEXT,
review_creation_date TEXT,
review_answer_timestamp TEXT
);

CREATE TABLE staging.orders(
order_id TEXT,
customer_id	TEXT,
order_status TEXT,
order_purchase_timestamp TEXT,
order_approved_at TEXT,
order_delivered_carrier_date TEXT,
order_delivered_customer_date TEXT,
order_estimated_delivery_date TEXT
);

CREATE TABLE staging.products(
product_id TEXT,
product_category_name TEXT,
product_name_length	TEXT,
product_description_length TEXT,
product_photos_qty TEXT,
product_weight_g TEXT,
product_length_cm TEXT,
product_height_cm TEXT,
product_width_cm TEXT
);

CREATE TABLE staging.sellers(
seller_id TEXT,
seller_zip_code_prefix TEXT,
seller_city TEXT,
seller_state TEXT
);

CREATE TABLE staging.product_category_name_translation(
product_category_name TEXT,
product_category_name_english TEXT
);

COPY staging.customers
FROM 'C:\SQL_DATA\Raw_Data\Olist\olist_customers_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY staging.geolocation
FROM 'C:\SQL_DATA\Raw_Data\Olist\olist_geolocation_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY staging.order_items
FROM 'C:\SQL_DATA\Raw_Data\Olist\olist_order_items_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY staging.order_payments
FROM 'C:\SQL_DATA\Raw_Data\Olist\olist_order_payments_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY staging.order_reviews
FROM 'C:\SQL_DATA\Raw_Data\Olist\olist_order_reviews_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY staging.orders
FROM 'C:\SQL_DATA\Raw_Data\Olist\olist_orders_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY staging.products
FROM 'C:\SQL_DATA\Raw_Data\Olist\olist_products_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY staging.sellers
FROM 'C:\SQL_DATA\Raw_Data\Olist\olist_sellers_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY staging.product_category_name_translation
FROM 'C:\SQL_DATA\Raw_Data\Olist\product_category_name_translation.csv'
DELIMITER ','
CSV HEADER;
```
## Data Exploration
```sql
----------------------------------------staging.customers---------------------------------------------------|
SELECT
	COUNT(*) FILTER (WHERE btrim(customer_id) = '')             AS customer_id_blanks,
	COUNT(*) FILTER (WHERE btrim(customer_unique_id) = '')      AS unique_customer_blanks
FROM staging.customers
;
SELECT
	COUNT(*) FILTER (WHERE customer_id IS NULL)                 AS customerid_nulls,
	COUNT(*) FILTER (WHERE customer_unique_id IS NULL)          AS customer_unique_nulls
FROM staging.customers 
;
SELECT
	customer_unique_id,
	COUNT(*) AS unique_id_row_count
FROM staging.customers
GROUP BY customer_unique_id
HAVING COUNT(*) > 1
;
SELECT
	customer_id,
	COUNT(*) AS customer_id_row_count
FROM staging.customers
GROUP BY customer_id
HAVING COUNT(*) > 1
;

----------------------------------------staging.geolocation-------------------------------------------------|
SELECT
	COUNT(*) FILTER (WHERE btrim(geolocation_zip_code_prefix) = '')     AS zip_blanks,
	COUNT(*) FILTER (WHERE btrim(geolocation_state) = '')       		AS state_blanks,
	COUNT(*) FILTER (WHERE btrim(geolocation_city) = '')                AS city_blanks
FROM staging.geolocation
;
SELECT
	COUNT(*) FILTER (WHERE geolocation_zip_code_prefix IS NULL)       AS geolocation_nulls,
	COUNT(*) FILTER (WHERE geolocation_city IS NULL)                  AS customerid_nulls
FROM staging.geolocation 
;
SELECT DISTINCT geolocation_city
FROM staging.geolocation
LIMIT 30
;
SELECT
	COUNT(DISTINCT geolocation_city)
FROM staging.geolocation
;
SELECT *
FROM staging.geolocation
WHERE geolocation_zip_code_prefix   !~ '^\d+$'
   OR geolocation_lat               !~ '^[+-]?\d+(\.\d+)?$'
   OR geolocation_lng               !~ '^[+-]?\d+(\.\d+)?$'
;
SELECT
	geolocation_zip_code_prefix::numeric
FROM staging.geolocation
WHERE LENGTH(geolocation_zip_code_prefix) <> 5
;

-- Findings:
-- 	City names include variations in formatting
-- 	such as '4º Centenário' vs '4º Centenário'.
-- Likely Reasons:
-- 	These represent the same location but appear as different strings.
-- Decision:
-- 	Normalize city names by: removing accents, removing (...) and (*), standardizing ordinals ('º' → 'o'),
--	converting to lowercase.
--  Use zip_code_prefix as primary key

----------------------------------------------staging.orders-------------------------------------------------| 
SELECT * FROM staging.orders LIMIT 10;
SELECT
	COUNT(*) FILTER (WHERE btrim(order_id) = '')                         AS order_id_blanks,
	COUNT(*) FILTER (WHERE btrim(customer_id) = '')                      AS customer_id_blanks,
	COUNT(*) FILTER (WHERE btrim(order_status) = '')                     AS order_status_blanks,
	COUNT(*) FILTER (WHERE btrim(order_delivered_carrier_date) = '')     AS carrier_delivered_blanks,
	COUNT(*) FILTER (WHERE btrim(order_delivered_customer_date) = '')    AS customer_delivered_blanks,
	COUNT(*) FILTER (WHERE btrim(order_purchase_timestamp) = '')         AS order_purchase_timestamp_blanks
FROM staging.orders
;
SELECT
	COUNT(*) FILTER (WHERE order_id IS NULL)                             AS order_id_nulls,
	COUNT(*) FILTER (WHERE customer_id IS NULL)                          AS customerid_nulls,
	COUNT(*) FILTER (WHERE order_status IS NULL)                         AS order_status,
	COUNT(*) FILTER (WHERE order_delivered_carrier_date IS NULL)         AS carrier_delivered_nulls,
	COUNT(*) FILTER (WHERE order_delivered_customer_date IS NULL)        AS customer_delivered_nulls,
	COUNT(*) FILTER (WHERE order_purchase_timestamp IS NULL)			 AS order_purchase_timestamp_nulls
FROM staging.orders 
;
SELECT *
FROM staging.orders
WHERE order_delivered_customer_date::timestamp <= order_purchase_timestamp::timestamp AND 
	  order_delivered_carrier_date::timestamp >= order_delivered_customer_date::timestamp
;
SELECT
MIN(order_purchase_timestamp::TIMESTAMP) AS min_purchase_time,
MAX(order_purchase_timestamp::TIMESTAMP) AS max_purchase_time,
MIN(order_delivered_customer_date::TIMESTAMP) AS min_delivered_time,
MAX(order_delivered_customer_date::TIMESTAMP) AS max_delivered_time
FROM staging.orders
;
SELECT *
FROM staging.orders
WHERE order_purchase_timestamp IS NULL
OR order_purchase_timestamp !~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01]) ([01]\d|2[0-3]):[0-5]\d:[0-5]\d$'
;
SELECT
	order_id,
	COUNT(*) AS order_id_row_count
FROM staging.orders
GROUP BY order_id
HAVING COUNT(*) > 1
;
SELECT DISTINCT order_status
FROM staging.orders
;
SELECT *
FROM staging.orders
WHERE order_delivered_carrier_date IS NULL OR order_delivered_customer_date IS NULL
LIMIT 20
;
SELECT *
FROM staging.orders
WHERE order_delivered_carrier_date IS NULL AND order_delivered_customer_date IS NULL AND order_status = 'delivered'
;
SELECT *
FROM staging.orders
WHERE order_status <> 'delivered' AND order_delivered_carrier_date IS NOT NULL
;
SELECT *
FROM staging.orders
WHERE order_status <> 'delivered' AND order_delivered_customer_date IS NOT NULL AND order_status <> 'shipped' AND order_status <> 'canceled'
;

-- Findings:
-- 	NULL values in order_delivered_carrier_date and 
-- 	order_delivered_customer_date where order_status = 'delivered'.
-- 	Some 'delivered' orders have missing carrier or customer delivery dates
-- 	Some orders with non-delivered statuses have delivery timestamps.
-- Likely Causes:
-- 	Nulls in carrier and customer delivered without 'delivered' have not been delivered.
-- 	Records with 'delivered', but no carrier or customer delivered date could be missing scan events
-- 	Records without 'delivered' and carrier or customer delivery dates could have
-- 	integration gaps between an order management and delivery tracking systems.
-- Decision:
-- 	Retain records without 'delivered' and nulls in the carrier and customer delivered date
-- 	Treat order_status as the source of truth for delivery status
-- 	Only calculate delivery metrics where order status = 'delivered' and customer delivery
-- 	date is not null.

----------------------------------------staging.orders_items-----------------------------------------------|
SELECT * FROM staging.order_items LIMIT 10;
SELECT
	COUNT(*) FILTER (WHERE btrim(order_id) = '')                    AS order_id_blanks,
	COUNT(*) FILTER (WHERE btrim(product_id) = '')                  AS product_id_blanks,
	COUNT(*) FILTER (WHERE btrim(seller_id) = '')                   AS seller_id_blanks,
	COUNT(*) FILTER (WHERE btrim(price) = '')                       AS price_blanks,
	COUNT(*) FILTER (WHERE btrim(freight_value) = '')               AS freight_value_blanks,
	COUNT(*) FILTER (WHERE btrim(shipping_limit_date) = '')         AS shipping_limit_blanks
FROM staging.order_items
;
SELECT
	COUNT(*) FILTER (WHERE order_id IS NULL)                        AS order_id_nulls,
	COUNT(*) FILTER (WHERE product_id IS NULL)                      AS product_id_nulls,
	COUNT(*) FILTER (WHERE price IS NULL)                           AS price_nulls,
	COUNT(*) FILTER (WHERE freight_value IS NULL)                   AS freight_value_nulls,
	COUNT(*) FILTER (WHERE shipping_limit_date IS NULL)             AS shipping_limit_nulls
FROM staging.order_items
;

----------------------------------------staging.orders_payments--------------------------------------------|
SELECT * FROM staging.order_payments LIMIT 10;
SELECT
	COUNT(*) FILTER (WHERE btrim(order_id) = '')                    AS order_id_blanks,
	COUNT(*) FILTER (WHERE btrim(payment_type) = '')                AS payment_type_blanks,
	COUNT(*) FILTER (WHERE btrim(payment_value) = '')               AS payment_value_blanks
FROM staging.order_payments
;
SELECT
	COUNT(*) FILTER (WHERE order_id IS NULL)                        AS order_id_nulls,
	COUNT(*) FILTER (WHERE payment_type IS NULL)                    AS payment_type_nulls,
	COUNT(*) FILTER (WHERE payment_value IS NULL)                   AS payment_value_nulls
FROM staging.order_payments
;
SELECT
	MIN(payment_installments)  AS min_installments,
	MAX(payment_installments)  AS max_installments
FROM staging.order_payments
;
SELECT *
FROM staging.order_payments
WHERE payment_installments = '0'
LIMIT 20
;
SELECT DISTINCT payment_type
FROM staging.order_payments
;
SELECT
	order_id,
	COUNT(*) AS order_id_row_count
FROM staging.order_payments
GROUP BY order_id
HAVING COUNT(*) > 1
;
SELECT *
FROM staging.order_payments
WHERE order_id = '53f5a7f622d498ff3eeb334b8efa7ae7'
;
SELECT
  p.order_id,
  p.amount_paid,
  i.amount_charged
FROM (
  SELECT 
  	order_id, 
  	SUM(payment_value::numeric) AS amount_paid
  FROM staging.order_payments
  GROUP BY order_id
) p
LEFT JOIN (
  SELECT
    order_id,
    SUM(COALESCE(price::numeric,0)) + SUM(COALESCE(freight_value::numeric,0)) AS amount_charged
  FROM staging.order_items
  GROUP BY order_id
) i ON i.order_id = p.order_id
WHERE amount_paid != amount_charged
LIMIT 20
;
SELECT*
FROM staging.orders
WHERE order_id = '00789ce015e7e5791c7914f32bb4fad4'
;
SELECT COUNT(DISTINCT order_id) FROM staging.order_payments;
SELECT COUNT(DISTINCT order_id) FROM staging.order_items;

SELECT 
  LENGTH(order_id),
  COUNT(*)
FROM staging.order_payments
GROUP BY LENGTH(order_id)
;
SELECT 
  LENGTH(order_id),
  COUNT(*)
FROM staging.order_items
GROUP BY LENGTH(order_id)
;
SELECT
	COUNT(*) 
FROM staging.order_payments p
LEFT JOIN staging.order_items i
ON p.order_id = i.order_id
WHERE i.order_id IS NULL
;
-- Findings:
-- 	830 payments do not have matching order items.
-- For these, `amount_charged` is NULL.
-- Decisions:
-- 	When joining flag records with either no matching order_item or
-- 	order payments.
-- 	Then, exclude depending on the analytical question

----------------------------------------staging.orders_products--------------------------------------------|
SELECT * FROM staging.products LIMIT 10;
SELECT
	COUNT(*) FILTER (WHERE btrim(product_id) = '')                    AS product_id_blanks,
	COUNT(*) FILTER(WHERE btrim(product_category_name) = '')               AS product_category_blanks
FROM staging.products
;
SELECT
	COUNT(*) FILTER (WHERE product_id IS NULL)                        AS product_id_nulls,
	COUNT(*) FILTER(WHERE product_category_name IS NULL)              AS product_category_nulls
FROM staging.products
;
SELECT *
FROM staging.products
WHERE product_category_name IS NULL
LIMIT 10
;
SELECT
	product_id,
	COUNT(*) AS product_id_row_count
FROM staging.products
GROUP BY product_id
HAVING COUNT(*) > 1
;
-- Findings:
-- 	Some products do not have category names
-- Decisions:
-- 	Exclude records from any product category analysis

----------------------------------------staging.orders_reviews--------------------------------------------|

SELECT * FROM staging.order_reviews LIMIT 10;
SELECT
	COUNT(*) FILTER (WHERE btrim(review_id) = '')             AS review_id_blanks,
	COUNT(*) FILTER(WHERE btrim(order_id) = '')               AS order_id_blanks,
	COUNT(*) FILTER(WHERE btrim(review_score) = '')           AS review_score_blanks
FROM staging.order_reviews
;
SELECT
	COUNT(*) FILTER (WHERE review_id IS NULL)           AS review_id_nulls,
	COUNT(*) FILTER(WHERE order_id IS NULL)             AS order_id_nulls,
	COUNT(*) FILTER(WHERE review_score IS NULL)         AS review_score_nulls
FROM staging.order_reviews
;
SELECT*
FROM staging.order_reviews
WHERE review_score !~ '^[0-5]$'
LIMIT 10
;
SELECT
    review_id,
    COUNT(DISTINCT order_id) AS order_count
FROM staging.order_reviews
GROUP BY review_id
HAVING COUNT(DISTINCT order_id) > 1
;
SELECT *
FROM staging.order_reviews
WHERE review_id = '0837bc8fa2cb75353513858b3bb5be1d'
;
SELECT
    review_id,
    order_id,
    review_score,
    review_comment_title,
    review_comment_message,
	review_creation_date
FROM staging.order_reviews
WHERE review_id IN (
    SELECT review_id
    FROM staging.order_reviews
    GROUP BY review_id
    HAVING COUNT(DISTINCT order_id) > 1
)
ORDER BY review_id
;
SELECT
    review_id,
    COUNT(DISTINCT order_id) AS order_count,
    COUNT(DISTINCT review_score) AS score_variation
FROM staging.order_reviews
GROUP BY review_id
HAVING COUNT(DISTINCT review_score) > 1
;
SELECT
    review_id,
    COUNT(DISTINCT review_comment_message) AS message_variation,
    COUNT(DISTINCT review_comment_title) AS title_variation
FROM staging.order_reviews
GROUP BY review_id
HAVING COUNT(DISTINCT order_id) > 1
;
SELECT
    order_id,
    COUNT(DISTINCT review_id) AS review_count
FROM staging.order_reviews
GROUP BY order_id
HAVING COUNT(DISTINCT review_id) > 1
;
SELECT *
FROM staging.order_reviews
WHERE order_id = '0544030711e50ec2cb6c15764d22891a'
;
SELECT
    review_id,
    order_id,
    review_score,
    review_comment_title,
    review_comment_message,
	review_creation_date
FROM staging.order_reviews
WHERE order_id IN (
    SELECT order_id
    FROM staging.order_reviews
    GROUP BY order_id
    HAVING COUNT(DISTINCT review_id) > 1
)
ORDER BY order_id
;
SELECT
    order_id,
    COUNT(DISTINCT review_id) AS review_count,
    COUNT(DISTINCT review_score) AS score_variation
FROM staging.order_reviews
GROUP BY order_id
HAVING COUNT(DISTINCT review_score) > 1
;
SELECT
    order_id,
    COUNT(DISTINCT review_comment_message) AS message_variation,
    COUNT(DISTINCT review_comment_title) AS title_variation
FROM staging.order_reviews
GROUP BY order_id
HAVING COUNT(DISTINCT review_id) > 1
;
-- Findings:
-- Some records are exact duplicates
-- There are records where all columns except either review_id or order_id are identical.
-- Likely Reasons:
-- 	There are duplicate records
-- 	Two or more orders were reviewed in the same post
-- 	Two or more reviews were provided for the same order because of
-- 	customers updating reviews.
-- Decisions:
-- 	Remove exact duplicates
-- 	When only review_id varies, keep one of the records
-- 	When only order_id varies, keep only one record for any order level analysis.
```
## Data Cleaning and Normalization
```sql
CREATE SCHEMA IF NOT EXISTS int;

DROP TABLE IF EXISTS int.orders;

CREATE TABLE IF NOT EXISTS int.orders AS
SELECT
	o.order_id::TEXT,
	o.customer_id::TEXT,
	c.customer_unique_id::TEXT,
	o.order_status::TEXT,
	o.order_purchase_timestamp::TIMESTAMP,
	o.order_delivered_carrier_date::TIMESTAMP,
	o.order_delivered_customer_date::TIMESTAMP
FROM staging.orders o
LEFT JOIN staging.customers c
	ON o.customer_id = c.customer_id	
;
ALTER TABLE int.orders
ADD COLUMN is_valid_delivery BOOLEAN
;
UPDATE int.orders
SET is_valid_delivery =
	CASE
		WHEN order_status = 'delivered'
		AND order_delivered_customer_date IS NOT NULL
		THEN TRUE
		ELSE FALSE
	END
;
DROP TABLE IF EXISTS int.order_payments;

CREATE TABLE IF NOT EXISTS int.order_payments AS
SELECT
	order_id::TEXT,
	payment_sequential::INT,
	payment_type::TEXT,
	payment_installments::INT,
	payment_value::NUMERIC(10,2)
FROM staging.order_payments
;
DROP VIEW IF EXISTS int.order_payments_agg;

CREATE VIEW IF NOT EXISTS int.order_payments_agg AS
SELECT
	order_id,
	SUM(payment_value) AS total_payment
FROM int.order_payments
GROUP BY order_id
;
DROP TABLE IF EXISTS int.order_items;

CREATE TABLE IF NOT EXISTS int.order_items AS
SELECT
	order_id::TEXT,
	order_item_id::INT,
	product_id::TEXT,
	seller_id::TEXT,
	shipping_limit_date::TIMESTAMP,
	price::NUMERIC(10,2),
	freight_value::NUMERIC(10,2)
FROM staging.order_items
;
DROP VIEW IF EXISTS int.order_items_agg;

CREATE VIEW int.order_items_agg AS
SELECT
	order_id,
	COUNT(*) AS item_count,
	SUM(PRICE) AS total_price,
	SUM(freight_value) AS total_freight_value
FROM int.order_items
GROUP BY order_id
;
DROP VIEW IF EXISTS int.order_items_check;

CREATE VIEW int.order_items_check AS
SELECT
	COALESCE(p.order_id, i.order_id) AS order_id,
	CASE
		WHEN p.order_id IS NULL AND i.order_id IS NOT NULL THEN 'item_no_payment'
		WHEN p.order_id IS NOT NULL AND i.order_id IS NULL THEN 'payment_no_items'
		ELSE 'valid'
	END AS item_status
FROM int.order_payments_agg p
FULL OUTER JOIN int.order_items_agg i
	ON p.order_id = i.order_id
;
DROP VIEW IF EXISTS int.reviews_dedupe;

CREATE VIEW IF NOT EXISTS int.reviews_dedupe AS
SELECT DISTINCT *
FROM staging.order_reviews
;
DROP VIEW IF EXISTS int.order_reviews;
CREATE VIEW IF NOT EXISTS int.order_reviews AS
SELECT *
FROM( 
	SELECT *,
		   ROW_NUMBER() OVER(
				PARTITION BY order_id
				ORDER BY review_creation_date::timestamp DESC
		   ) AS rn
	FROM int.reviews_dedupe
)t
WHERE rn = 1
;
DROP TABLE IF EXISTS int.products;

CREATE TABLE IF NOT EXISTS int.products AS
SELECT
	product_id::TEXT,
	product_category_name::TEXT,
	product_name_length::INT,
	product_description_length::INT,
	product_photos_qty::INT,
	product_weight_g::INT,
	product_length_cm::INT,
	product_height_cm::INT,
	product_width_cm::INT
FROM staging.products
;
ALTER TABLE int.products
ADD COLUMN has_category BOOLEAN
;
UPDATE int.products
SET has_category =
	CASE
		WHEN product_category_name IS NULL THEN FALSE
		ELSE TRUE
	END
;
CREATE EXTENSION IF NOT EXISTS unaccent;

DROP TABLE IF EXISTS int.geolocation;

CREATE TABLE IF NOT EXISTS int.geolocation AS
SELECT
  geolocation_zip_code_prefix::int,
  lower(
    regexp_replace(
      unaccent(geolocation_city),
      '[^a-zA-Z0-9]',
      '',
      'g'
    )
  ) AS normalized_city,
  geolocation_state::text,
  geolocation_lat::numeric,
  geolocation_lng::numeric
FROM staging.geolocation
;
```
## Feature Engineering
```sql
CREATE SCHEMA IF NOT EXISTS analytics;

DROP TABLE IF EXISTS analytics.fact_orders;
CREATE TABLE IF NOT EXISTS analytics.fact_orders AS
SELECT
	o.order_id,
	o.customer_id,
	o.customer_unique_id,
	o.order_status,
	o.order_purchase_timestamp,
	p.total_payment,
	i.item_count,
	i.total_price,
	i.total_freight_value,
	r.review_score,
	CASE
		WHEN o.is_valid_delivery THEN
			EXTRACT(DAY FROM
				(o.order_delivered_customer_date - o.order_purchase_timestamp)
			)
		ELSE NULL
	END AS delivery_days,
	CASE
		WHEN o.order_status = 'canceled' THEN 'lost'
		WHEN o.order_status = 'delivered' THEN 'realized'
		ELSE 'at risk'
	END AS revenue_status,
	c.item_status
FROM int.orders o
LEFT JOIN int.order_payments_agg p
	ON o.order_id = p.order_id
LEFT JOIN int.order_items_agg i
	ON o.order_id = i.order_id
LEFT JOIN int.order_reviews r
	ON o.order_id = r.order_id
LEFT JOIN int.order_items_check c
	ON o.order_id = c.order_id
;
```
## Dashboard
[Tableau Dashboard](https://public.tableau.com/views/Olist_Ecommerce_Dashboard_17785230046370/Dashboard1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)
