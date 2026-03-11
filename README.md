### 1. Customer revenue summary 
```sql
WITH base_cte AS (
    SELECT 
        c.CustomerName,
        c.CustomerID,
        OrderNumber,
        o.OrderID,
        o.OrderDate,
        Quantity,
        UnitPrice
    FROM orders.orders o
    LEFT JOIN orders.OrderItems od
        ON od.OrderID = o.OrderID
    LEFT JOIN Customer.Customers c
        ON c.CustomerID = o.CustomerID
    LEFT JOIN sales.Payments p
        ON p.OrderID = o.OrderID
),
cte_aggregation AS (
    SELECT
        CustomerName,
        CustomerID,
        MAX(OrderDate) AS last_order,
        MIN(OrderDate) AS first_order,
        COUNT(OrderNumber) AS total_orders,
        SUM(Quantity) AS total_quantity,
        SUM(Quantity * UnitPrice) AS total_sales
    FROM base_cte
    GROUP BY
        CustomerName,
        CustomerID
)
SELECT TOP 10
    CustomerName,
    total_quantity,
    total_orders,
    total_sales,
    ROUND((total_sales / total_orders), 2) AS avg_order_value,
    total_sales / DATEDIFF(month, first_order, last_order) AS avg_monthly_spend
FROM cte_aggregation
ORDER BY total_sales DESC;

```


<img width="838" height="352" alt="image" src="https://github.com/user-attachments/assets/5357703b-6392-4585-8ffc-0afcbdc56bbc" />

### 2. Find customers whose total revenue is greater than the average total revenue of all customers. 
```sql
SELECT 
	CustomerName,
	total_sales
FROM(
	SELECT 
		c.CustomerName,
		SUM(quantity * unitprice) AS total_sales,
		AVG(SUM(quantity * unitprice)) OVER() AS avg_sales
	FROM orders.orders o
	LEFT JOIN orders.OrderItems od
		ON od.OrderID = o.OrderID
	LEFT JOIN Customer.Customers c
		ON c.CustomerID = o.CustomerID
	GROUP BY CustomerName
	) AS t
WHERE total_sales > avg_sales
```

<img width="670" height="333" alt="image" src="https://github.com/user-attachments/assets/dc30e1ef-6fde-4590-ae3b-0917e217b7e7" />

### 3. Product sales performance by category  
```sql
SELECT 
    pc.CategoryName AS category_name,
    COALESCE(SUM(o.Quantity), 0) AS total_quantity,
    COALESCE(SUM(o.Quantity * p.UnitCost), 0) AS cost_of_sales,
    CAST(ROUND(COALESCE(SUM((o.Quantity * o.UnitPrice) * (1 - o.DiscountPct)), 0), 1) AS DECIMAL(18,1)) AS total_sales,
    CAST(
        COALESCE(SUM((o.Quantity * o.UnitPrice) * (1 - o.DiscountPct)), 0)
        / NULLIF(SUM(o.Quantity), 0)
        AS DECIMAL(18,1)
    ) AS avg_selling_price,
    COUNT(DISTINCT p.ProductID) AS number_of_products
FROM Product.Products p
LEFT JOIN Product.Categories pc
    ON p.CategoryID = pc.CategoryID
LEFT JOIN Orders.OrderItems o
    ON o.ProductID = p.ProductID
GROUP BY pc.CategoryName
ORDER BY total_sales DESC;
```
<img width="901" height="283" alt="image" src="https://github.com/user-attachments/assets/00e3f9a4-a1f2-41ed-963f-dc701d40a469" />

### 3. Top 3 products per category  
```sql
WITH cte_base AS (
    SELECT
        pc.CategoryName,
        p.ProductName,
        o.Quantity,
        o.UnitPrice,
        o.DiscountPct
    FROM Product.Products p
    LEFT JOIN Product.Categories pc
        ON p.CategoryID = pc.CategoryID
    LEFT JOIN Orders.OrderItems o
        ON o.ProductID = p.ProductID
),
cte_aggregation AS (
    SELECT
        CategoryName,
        ProductName,
        SUM((Quantity * UnitPrice) * (1 - DiscountPct)) AS total_revenue
    FROM cte_base
    GROUP BY
        CategoryName,
        ProductName
),
cte_rank AS (
    SELECT
        CategoryName,
        ProductName,
        CAST(
            SUM(total_revenue) OVER (
                PARTITION BY CategoryName
                ORDER BY total_revenue DESC
            ) AS DECIMAL(10, 2)
        ) AS product_revenue,
        ROW_NUMBER() OVER (
            PARTITION BY CategoryName
            ORDER BY total_revenue ASC
        ) AS Rank_number
    FROM cte_aggregation
)
SELECT
    CategoryName,
    ProductName,
    product_revenue,
    Rank_number
FROM cte_rank
WHERE Rank_number <= 3;
```
<img width="501" height="307" alt="Screenshot 2026-03-11 091839" src="https://github.com/user-attachments/assets/be14f977-c644-44a3-85cc-69585c44322f" />

