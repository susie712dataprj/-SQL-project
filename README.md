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


