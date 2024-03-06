# AtliQ-Mart-Promotional-Insights
This is SQL-Power BI based project for promotional insights for AtliQ Mart in FMCG domain.

## SQL Queries for Ad-hoc requests:

### Request 1: Provide a list of products with a base price greater than 500 and that are featured in promo type of BOGOF (Buy One Get One Free).

```sql
SELECT
    p.product_code,
    p.product_name,
    p.category,
    SUM(e.`quantity_sold(before_promo)`) AS total_sold_quantity_before_promo,
    SUM(e.`quantity_sold(after_promo)`) AS total_sold_quantity_after_promo
FROM
    dim_products p
JOIN
    fact_events e
USING (product_code)
WHERE
    base_price > 500 AND promo_type = 'BOGOF'
GROUP BY
    p.product_code;
```

### Request 2: Generate a report that provides an overview of the number of stores in each city.

#### The report includes two essential fields:
- city
- store_count

```sql
SELECT
    city,
    COUNT(store_id) AS store_count
FROM
    dim_stores
GROUP BY
    city
ORDER BY
    store_count DESC;
```

### Request 3: Generate a report that displays each campaign along with the total revenue generated before and after the campaign.

#### The report includes three key fields:
- campaign_name
- total_revenue(before_promo)
- total_revenue(after_promo)

```sql
SELECT 
    c.campaign_name,
    CONCAT(ROUND(SUM(r.revenue_before_promo / 1000000),2), ' M') AS total_revenue_before_promo,
    CONCAT(ROUND(SUM(r.revenue_after_promo / 1000000),2), ' M') AS total_revenue_after_promo
FROM
    revenue r
JOIN
    dim_campaigns c
USING (campaign_id)
GROUP BY
    c.campaign_name;
```

### Request 4: Produce a report that calculates the Incremental Sold Quantity (ISU%) for each category during Diwali campaign. Additionally, provide rankings for the categories based on their ISU%.

#### The report will include three key fields:
- category
- ISU%
- rank_order


```sql
WITH CTE1 AS
(
    SELECT 
        r.category,
        SUM(r.`quantity_sold(before_promo)`) AS sold_quantity_before_promo,
        SUM(r.`quantity_sold(after_promo)`) AS sold_quantity_after_promo
    FROM 
        revenue r
    JOIN 
        dim_campaigns c USING (campaign_id)
    WHERE 
        c.campaign_name = "Diwali"
    GROUP BY 
        r.category
),
CTE2 AS
(
    SELECT *,
        ROUND((sold_quantity_after_promo - sold_quantity_before_promo) / sold_quantity_before_promo * 100, 2) AS incremental_sold_quantity_pct
    FROM 
        CTE1
)
SELECT 
    *,
    RANK() OVER(ORDER BY incremental_sold_quantity_pct DESC) AS rank_order
FROM 
    CTE2;
```

### Request 5: Create a report featuring the Top 5 products, ranked by Incremental Revenue Percentage (IR%), across all campaigns.

#### The report will include three key fields:
- product_name
- category
- IR%

```sql
WITH CTE1 AS
(
    SELECT
        product_code,
        product_name,
        category,
        CONCAT(ROUND(SUM(revenue_before_promo)/1000000,2), ' M') AS total_revenue_before_promo,
        CONCAT(ROUND(SUM(revenue_after_promo)/1000000,2), ' M') AS total_revenue_after_promo,
        CONCAT(ROUND(SUM(`IR`)/1000000,2), ' M') AS total_incremental_revenue
    FROM
        revenue
    GROUP BY
        product_code
),
CTE2 AS
(
    SELECT *,
        ROUND((total_incremental_revenue/total_revenue_before_promo)*100, 2) AS incremental_revenue_pct
    FROM
        CTE1
),
CTE3 AS
(
    SELECT *,
        RANK() OVER(ORDER BY incremental_revenue_pct DESC) AS rank_order
    FROM
        CTE2
)

SELECT * FROM CTE3 WHERE rank_order <= 5;
```

## Database View: Revenue

#### The revenue view aggregates data from the fact_events table and calculates revenue metrics before and after promotions.

```sql
CREATE VIEW revenue_view AS
WITH CTE1 AS
(
    SELECT *,
        CASE
            WHEN promo_type = '50% OFF' THEN base_price * 0.5
            WHEN promo_type = '25% OFF' THEN base_price * 0.75
            WHEN promo_type = '33% OFF' THEN base_price * 0.67
            WHEN promo_type = 'BOGOF' THEN base_price
            WHEN promo_type = '500 Cashback' THEN base_price - 500
        END AS base_price_after_promo
    FROM
        fact_events
),
CTE2 AS
(
    SELECT 
        e.event_id,
        s.store_id,
        c.campaign_id,
        s.city,
        p.product_code,
        p.product_name,
        p.category,
        e.base_price,
        a.base_price_after_promo,
        e.promo_type,
        e.`quantity_sold(before_promo)`,
        e.`quantity_sold(after_promo)`,
        c.campaign_name,
        ROUND(e.base_price * e.`quantity_sold(before_promo)`, 2) AS revenue_before_promo,
        ROUND(a.base_price_after_promo * e.`quantity_sold(after_promo)`, 2) AS revenue_after_promo
    FROM
        fact_events e
    JOIN
        dim_campaigns c USING (campaign_id)
    JOIN
        dim_products p USING (product_code)
    JOIN
        dim_stores s USING (store_id)
    JOIN
        CTE1 a USING (event_id)
)
SELECT *,
    (revenue_after_promo - revenue_before_promo) AS IR,
    (`quantity_sold(after_promo)` - `quantity_sold(before_promo)`) AS ISU
FROM
    CTE2;
```
