### ECOMMERCE PERFORMANCE
## Data Cleaning

```RUBY
UPDATE dbo.[transaction]
SET quantity=1
WHERE quantity IS NULL OR quantity=0;

DELETE dbo.[transaction]
WHERE item_id IS NULL;
```
## 
# Which item has the highest GMV?
GMV = item_price * quantity_sold (excluding canceled & returned order)

```RUBY
SELECT Top 1
    item_name,
    SUM(price * quantity) as Total_GMV
FROM dbo.[item]
    INNER JOIN dbo.[transaction]
    ON dbo.[item].id=dbo.[transaction].item_id
WHERE status_id='2' or status_id='3'
GROUP BY item_name
ORDER BY Total_GMV DESC;
```

# Which weekday has the highest number
```RUBY
SELECT datename(dw, timestamp) as weekday, 
    count(*) as number_of_orders
FROM dbo.[transaction]
GROUP BY datename(dw, timestamp)
ORDER BY number_of_orders DESC;
```

# How much the most frequent customer spend? In how many orders?
```RUBY
SELECT Top 1
    customer_id,
    sum(price * quantity) as Spendings,
    count(order_no) AS Number_of_orders,
    sum(quantity) AS quantity_sold
FROM dbo.[item]
INNER JOIN dbo.[transaction]
ON dbo.[item].id=dbo.[transaction].item_id
GROUP BY customer_id
ORDER by count(order_no) DESC;
```

# For the most frequent customer, what did he/she buy after an order in Beauty category?
```RUBY
ALTER TABLE dbo.[transaction]
ADD category_id TINYINT

UPDATE dbo.[transaction]
SET category_id = left(item_id,2)

WITH FrequentCustomers AS (
  SELECT TOP 1
  customer_id, COUNT(*) AS order_count
  FROM dbo.[transaction]
  GROUP BY customer_id
  ORDER BY order_count DESC -- Get the customer with the highest order count (most frequent)
)
SELECT FrequentCustomers.customer_id,
        dbo.category.category,
       LEAD(dbo.category.category,1) OVER (PARTITION BY FrequentCustomers.customer_id ORDER BY timestamp) AS next_purchase,
       dbo.[transaction].timestamp
FROM dbo.[transaction] 
INNER JOIN FrequentCustomers ON FrequentCustomers.customer_id = dbo.[transaction].customer_id
INNER JOIN dbo.category ON dbo.category.id=dbo.[transaction].category_id  
ORDER BY FrequentCustomers.customer_id
```

# Breakdown of sales by category
# Total Sales in each category -> treemap
```RUBY
SELECT category, item_name,
    sum(price * quantity) as revenue
FROM dbo.[transaction]
INNER JOIN dbo.[item] 
    ON dbo.[transaction].item_id=dbo.[item].id
INNER JOIN dbo.[category]
    ON dbo.[category].id=dbo.[transaction].category_id
GROUP BY dbo.[category].category, item_name
ORDER BY category desc
```
# Quantites over months
```RUBY
SELECT category,
    [1] AS Jan,
    [2] AS Feb,
    [3] AS Mar,
    [4] AS Apr,
    [5] AS May
FROM (
SELECT category, 
    DATEPART(MONTH,timestamp) as month,
    sum(quantity) as Quantity
FROM dbo.[transaction]
LEFT JOIN dbo.[category]
ON dbo.[transaction].category_id=dbo.[category].id 
GROUP BY category, DATEPART(MONTH,timestamp)) as source
PIVOT (
    Sum(Quantity)
    FOR month
    IN ([1], [2], [3], [4], [5], [6])
) AS PivotMonth

UNION

SELECT 'Total',
    [1] AS Jan,
    [2] AS Feb,
    [3] AS Mar,
    [4] AS Apr,
    [5] AS May
FROM (
SELECT 
    DATEPART(MONTH,timestamp) as month,
    sum(quantity) as Quantity
FROM dbo.[transaction]
LEFT JOIN dbo.[category]
ON dbo.[transaction].category_id=dbo.[category].id 
GROUP BY DATEPART(MONTH,timestamp)) as source
PIVOT (
    Sum(Quantity)
    FOR month
    IN ([1], [2], [3], [4], [5], [6])
) AS PivotMonth
```

# Sales Contribution by Categories
Create Spendings column
```RUbY
ALTER TABLE dbo.[transaction]
ADD spendings int

UPDATE dbo.[transaction]
SET spendings = quantity * dbo.item.price
FROM dbo.[transaction]
INNER JOIN dbo.item
ON dbo.[transaction].item_id=dbo.item.id
```
# % of Grand Total in each Category
```RUbY
WITH cate as (
        SELECT 
        dbo.category.category as category,
        sum(dbo.[transaction].spendings) as Cate_GMV
        FROM dbo.[transaction]
        left join dbo.category
        on dbo.[transaction].category_id=dbo.category.id
        group by dbo.category.category
),

GMV_Total as (
        select
        sum(cast(dbo.[transaction].spendings as bigint)) as Total_GMV FROM dbo.[transaction]
)

SELECT
    cate.category,
    -- cate.Cate_GMV,
    -- GMV_Total.Total_GMV,
    round(cast(Cate.Cate_GMV as float)*100/cast(GMV_Total.Total_GMV as float),2) as '%'
FROM Cate 
CROSS JOIN GMV_Total
```

# % of Grand Total in each Category through months
```RUbY
WITH cate as (
        SELECT 
        month(dbo.[transaction].[timestamp]) as month,
        dbo.category.category as category,
        sum(dbo.[transaction].spendings) as Cate_GMV
        FROM dbo.[transaction]
        left join dbo.category
        on dbo.[transaction].category_id=dbo.category.id
        group by month(dbo.[transaction].[timestamp]), dbo.category.category
)
SELECT
    category,
    cast([1] as decimal(10,2)) as Jan,
    cast([2] as decimal(10,2)) as Feb,
    cast([3] as decimal(10,2)) as Mar,
    cast([4] as decimal(10,2)) as Apr,
    cast([5] as decimal(10,2)) as May,
    cast([6] as decimal(10,2)) as Jun
From (
    Select
    category,
    [1], [2], [3], [4], [5], [6]
    FROM (
        SELECT
            category,
            month,
            ROUND(SUM(cate_gmv) * 100.0 / total_gmv_per_month, 2) AS contribution
        FROM (
            SELECT
                category,
                month,
                cate_gmv,
                SUM(cate_gmv) OVER (PARTITION BY month) AS total_gmv_per_month
            FROM cate
        ) AS agg
        GROUP BY category, month, total_gmv_per_month
    ) AS source
    PIVOT (
        sum(contribution)
        FOR month IN ([1], [2], [3], [4], [5], [6])
    ) AS pivottable
) AS casted_results;
```


