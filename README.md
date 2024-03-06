# AtliQ-Mart-Promotional-Insights
This is SQL-Power BI based project for promotional insights for AtliQ Mart in FMCG domain.

## SQL Queries for Ad-hoc requests:

### Request 1: List of Products with Base Price > 500 and Featured in BOGOF Promo

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
