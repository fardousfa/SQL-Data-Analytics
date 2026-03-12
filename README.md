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

### 4. Top 3 products per category  
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

### 5.Monthly revenue with previous month comparison,running totals, 3 Months rolling average
```sql
WITH base_query AS(
SELECT 
	YEAR(s.orderdate) AS order_year,
	datetrunc(Month,s.OrderDate) AS order_month,
	CAST(SUM((UnitPrice * Quantity) * (1-DiscountPct)) AS DECIMAL (10,2)) AS total_revenue
FROM Orders.OrderItems o
	INNER JOIN orders.Orders s
ON o.OrderID = s.OrderID
GROUP BY datetrunc(Month,s.OrderDate),YEAR(s.orderdate)
)
SELECT
	order_year,
	order_month,
	total_revenue,
	CASE WHEN LAG(total_revenue) OVER (ORDER BY order_month) is NULL THEN '-' 
			ELSE 
			CONCAT(CAST(((total_revenue / LAG(total_revenue) OVER (ORDER BY order_month)) -1)*100 AS DECIMAL (10,2)),'%') 
		END AS pct_change,
	SUM(total_revenue) Over(order By order_month) as running_total,
	CAST(AVG(total_revenue) Over(order by order_month rows between 2 preceding and current row) as decimal(10,2)) as three_month_rolling_avg
FROM base_query
ORDER BY order_month
```
<img width="979" height="553" alt="image" src="https://github.com/user-attachments/assets/de673889-82da-4b50-ba4a-9b99c3fe73d0" />

### 6. Running total revenue for customers over time

```sql
WITH cte_cust AS(
SELECT 
	o.CustomerID,
	c.CustomerName,
	o.OrderID,
	OrderDate,
	SUM(Quantity * UnitPrice) as revenue
FROM Customer.Customers c
LEFT JOIN orders.Orders o
	ON c.CustomerID = o.CustomerID
INNER JOIN Orders.OrderItems OT
	ON ot.OrderID = o.OrderID
GROUP BY o.CustomerID,
		c.CustomerName,
		o.OrderID,
		OrderDate
)
SELECT
	*,
	SUM(revenue) Over (partition BY customerID order by revenue,orderdate
    rows between unbounded preceding and current row) as running_revenue
FROM cte_cust;
```
<img width="409" height="268" alt="Screenshot 2026-03-11 192822" src="https://github.com/user-attachments/assets/278d2759-a691-43cb-bed0-462656d3229c" />

### 7. Find products that were returned at least once
```sql
SELECT 
	ProductID,
	productName,
	total_quantity_sold,
	total_return,
	CONCAT(
		CAST(100* CAST(
		total_return as FLOAT) / NULLIF(total_quantity_sold, 0) as decimal(10,2))
	,'%') as pct_return
FROM(
	SELECT
		p.ProductID,
		p.ProductName,
		SUM(o.quantity) as total_quantity_sold,
		COALESCE(SUM(r.ReturnQty), 0) as total_return
	FROM Orders.OrderItems o
	LEFT JOIN sales.Returns r
	ON r.OrderItemID = o.OrderItemID
	LEFT JOIN Product.Products p
	ON o.ProductID = p.ProductID
	GROUP BY p.ProductID,
	p.ProductName)t
ORDER BY pct_return ASC;
```
<img width="831" height="483" alt="image" src="https://github.com/user-attachments/assets/c0620963-bd97-4ada-bf5f-ced787dfe709" />

### 8. Creating View: order_revenue_details

```sql
CREATE or ALTER VIEW order_summary AS
SELECT 
	o.OrderID,
	DATETRUNC(MONTH,o.OrderDate) as order_month,
	c.CustomerID,
	c.CustomerName,
	e.EmployeeID,
	e.FullName,
	r.RegionID,
	r.CityName,
	o.OrderStatus,
	SUM(od.Quantity * od.UnitPrice * (1 - od.DiscountPct))as total_revenue
FROM orders.Orders o
LEFT JOIN Customer.Customers c
	ON c.CustomerID = o.CustomerID 
LEFT JOIN Employees.Employees e
	ON e.EmployeeID = o.EmployeeID
LEFT JOIN Region.Regions r
	ON r.RegionID = o.RegionID
LEFT JOIN orders.OrderItems od
	ON od.OrderID = o.OrderID
GROUP BY 	o.OrderID,
		o.OrderDate,
		c.CustomerID,
		c.CustomerName,
		e.EmployeeID,
		e.FullName,
		r.RegionID,
		o.OrderStatus,
		r.CityName;
GO
```
<img width="1324" height="541" alt="image" src="https://github.com/user-attachments/assets/4dba1180-a094-487a-833f-861a0f41987d" />

### 8. Creating View: customer_performance_details

```sql

CREATE OR ALTER VIEW dbo.customer_performance_details AS
SELECT 
	customerid,
	customername,
	last_order_date,
	total_orders,
	total_revenue,
	CAST(total_revenue / total_orders as decimal (10,2)) as avg_order_revenue,
	DATEDIFF(Month,last_order_date,getdate()) as month_since_last_order,
	CASE when total_revenue <= 15000 THEN 'Low Value'
		WHEN total_revenue between 15000 and 35000 THEN 'Mid Value'
		ELSE 'High Value'
		END as customer_category
FROM(
SELECT 
	c.CustomerID,
	c.CustomerName,
	MAX(o.orderdate) as last_order_date,
	SUM(Quantity) as total_orders,
	CAST(SUM((Quantity *	UnitPrice)*(1-DiscountPct)) as decimal(10,2)) as total_revenue
FROM orders.Orders o
LEFT JOIN orders.OrderItems od
ON o.OrderID = od.OrderID
LEFT JOIN customer.Customers c
ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID,
		c.CustomerName)t
;

```

<img width="1114" height="720" alt="image" src="https://github.com/user-attachments/assets/8a5b911e-71b8-4bfa-8f81-237fd3a3daa4" />

### 9. For each sales territory, find the employee with the highest total revenue.
```sql
WITH cte_base AS(
SELECT 
	SalesTerritory,
	e.FullName,
	CONCAT('£',CAST(SUM((quantity * unitprice)*(1-od.discountpct)) as decimal(10,2))) AS total_rev
FROM Employees.Employees e
LEFT JOIN Region.Regions r
	ON r.RegionID = e.RegionID
LEFT JOIN orders.orders ord
	ON ord.EmployeeID = e.EmployeeID
LEFT JOIN orders.OrderItems od
	ON od.OrderID=ord.OrderID
GROUP BY SalesTerritory,
	e.FullName
),
rank_cte AS(
SELECT *,
	ROW_NUMBER() OVER(PARTITION BY salesterritory ORDER BY total_rev DESC) AS Ranks
FROM cte_base)

SELECT
	SalesTerritory,
	FullName,
	total_rev
FROM rank_cte
WHERE Ranks = 1
ORDER BY total_rev;

```
<img width="579" height="274" alt="image" src="https://github.com/user-attachments/assets/5019895a-38d1-488f-a298-17b56a179d9e" />

