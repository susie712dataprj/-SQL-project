### ECOMMERCE PERFORMANCE
## Data Cleaning

```RUBY
UPDATE dbo.[transaction]
SET quantity=1
WHERE quantity IS NULL OR quantity=0;

DELETE dbo.[transaction]
WHERE item_id IS NULL;
```

