# 👜 Luxury Apparel Analytics using SQL



## Overview
This project delivers an end-to-end SQL-based analysis of a 5,000-product Luxury Apparel Catalog using PostgreSQL.
The goal is to uncover:

- Category trends
- Material usage patterns
- High-value product insights
- Brand dominance
- Data quality improvements
- Advanced text-based analytics

This mirrors real-world product intelligence work done in companies like Farfetch, Net-a-Porter, Moda Operandi, and SSENSE.
---

## Objectives
- Analyze product distribution across categories and subcategories
- Extract material patterns from descriptions
- Perform text mining on product names
- Identify potential duplicates and inconsistencies
- Build portfolio-grade SQL queries with window functions, full-text search, and rollups

---
## Dataset
**Dataset Name:** Luxury Apparel Dataset
**Source:** [Luxury Apparel Dataset](https://www.kaggle.com/datasets/chitwanmanchanda/luxury-apparel-data?select=Luxury_Products_Apparel_Data.csv)

This dataset includes luxury fashion items from categories like footwear, accessories, apparel, jewelry, activewear, etc.
---

## Database Schema
---sql
CREATE TABLE luxury_apparel (
    id INT,
    category TEXT,
    subcategory TEXT,
    product_name TEXT,
    description TEXT
);
---

--sql
--QUERY(1) - Category-wise Product Share Using Window Function
-- Category-wise product count + % share of total catalog
SELECT 
    category,
    COUNT(*) AS product_count,
    ROUND(
        100.0 * COUNT(*) / SUM(COUNT(*)) OVER (),
        2
    ) AS pct_of_catalog
FROM luxury_apparel
GROUP BY category
ORDER BY product_count DESC;
---

---sql
--QUERY(2) - Top Subcategories per Category (Ranking with Window Functions)
-- Rank subcategories inside each category by number of products
SELECT
    category,
    subcategory,
    COUNT(*) AS product_count,
    RANK() OVER (
        PARTITION BY category
        ORDER BY COUNT(*) DESC
    ) AS subcategory_rank
FROM luxury_apparel
GROUP BY category, subcategory
ORDER BY category, subcategory_rank;
---

---sql
--QUERY(3) - Material-Based Grouping from Description (CASE + ILIKE)
-- Classify products by key material mentioned in description
SELECT 
    material_group,
    COUNT(*) AS product_count
FROM (
    SELECT 
        CASE
            WHEN description ILIKE '%leather%'   THEN 'Leather'
            WHEN description ILIKE '%cashmere%'  THEN 'Cashmere'
            WHEN description ILIKE '%cotton%'    THEN 'Cotton'
            WHEN description ILIKE '%wool%'      THEN 'Wool'
            WHEN description ILIKE '%silk%'      THEN 'Silk'
            ELSE 'Other / Unknown'
        END AS material_group
    FROM luxury_apparel
) AS t
GROUP BY material_group
ORDER BY product_count DESC;
---

---sql
--QUERY(4) - Longest Product Descriptions (Content Depth)
-- See which products have the longest descriptions
SELECT 
    id,
    category,
    subcategory,
    product_name,
    LENGTH(description) AS description_length
FROM luxury_apparel
ORDER BY description_length DESC
LIMIT 20;
---

---sql
--QUERY(5) - Most Frequent Keywords in Product Names (Text + CTE + LATERAL)
-- Find most common words used in product names
WITH tokens AS (
    SELECT 
        LOWER(
            regexp_replace(word, '[^a-z0-9]', '', 'g')
        ) AS token
    FROM luxury_apparel,
    LATERAL regexp_split_to_table(product_name, '\s+') AS word
),
filtered AS (
    SELECT token
    FROM tokens
    WHERE 
        token <> '' 
        AND length(token) > 3
        AND token NOT IN ('with','from','this','that','your','into','over','made')
)
SELECT 
    token,
    COUNT(*) AS frequency
FROM filtered
GROUP BY token
HAVING COUNT(*) > 5
ORDER BY frequency DESC
LIMIT 30;
---

---sql
--QUERY(6) - Products Where Description Doesn’t Mention Subcategory
-- Spot products where subcategory isn't mentioned in the description
SELECT 
    id,
    category,
    subcategory,
    product_name,
    description
FROM luxury_apparel
WHERE 
    subcategory IS NOT NULL
    AND description NOT ILIKE '%' || subcategory || '%';
---

---sql
--QUERY(7) - Brand Hint Analysis Using First Word as Brand
-- Approximate "brand" as the first word of the product name
WITH brand_base AS (
    SELECT
        TRIM(SPLIT_PART(product_name, ' ', 1)) AS brand_hint
    FROM luxury_apparel
)
SELECT 
    brand_hint,
    COUNT(*) AS product_count
FROM brand_base
GROUP BY brand_hint
HAVING COUNT(*) > 20
ORDER BY product_count DESC;
---

---sql
--QUERY(8) - Full-Text Search on ProductName + Description
-- Find products relevant for "cashmere scarf" using full text search
SELECT 
    id,
    category,
    subcategory,
    product_name
FROM luxury_apparel
WHERE 
    to_tsvector(
        'english',
        COALESCE(product_name, '') || ' ' || COALESCE(description, '')
    ) @@ plainto_tsquery('english', 'cashmere scarf')
LIMIT 30;
---

---sql
--QUERY(9) - Detect Exact Duplicate Product Names
-- Check if any product names are duplicated
SELECT 
    LOWER(product_name) AS normalized_name,
    COUNT(*) AS occurrences
FROM luxury_apparel
GROUP BY LOWER(product_name)
HAVING COUNT(*) > 1
ORDER BY occurrences DESC;
---

---sql
-- QUERY(10) - Multi-level Category & Subcategory Summary using ROLLUP
-- Gives product counts at:
-- 1) Category + Subcategory level
-- 2) Category total level
-- 3) Overall catalog total

SELECT
    COALESCE(category, 'ALL CATEGORIES')    AS category,
    COALESCE(subcategory, 'ALL SUBCATEGORIES') AS subcategory,
    COUNT(*) AS product_count
FROM luxury_apparel
GROUP BY ROLLUP (category, subcategory)
ORDER BY 
    CASE WHEN category IS NULL THEN 2 
         WHEN subcategory IS NULL THEN 1 
         ELSE 0 
    END,
    category,
    subcategory;
---
