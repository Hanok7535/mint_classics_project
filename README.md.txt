mint classics inventory analysis  

project overview 
mint classics is closing one of its storage facilities. our goal is to analyze inventory, sales, and payments to make data-driven recommendations on:  
which warehouse to close
how to reorganize inventory efficiently 
ensuring timely customer service 

step 1: set up & explore database  
before writing queries, we first understand the data structure.  
1. use the mint classics database 
```sql
use mintclassics;

2. show available tables
displays an overview of all tables in the database.
show tables;

3. describe table structures
to understand the columns and data types in each table.
desc customers;
desc employees;
desc offices;
desc orderdetails;
desc orders;
desc payments;
desc productlines;
desc products;
desc warehouses;

4. show indexes (primary & foreign keys)
identifies relationships between tables.
show indexes from customers;
show indexes from orders;


step 2: 
once the database structure is clear, the next step is to identify the key data points that will help in analyzing inventory, warehouse efficiency, and sales performance. this ensures that we work with the correct tables and columns when writing sql queries.

irrelavant tables
these tables do not contribute directly to warehouse efficiency, inventory management, or sales analysis, so they will not be part of our queries.
1. customers – customer details do not impact warehouse decision-making.
2. employees – employee information is not relevant to inventory analysis.
3. offices – office locations do not influence storage facility closures.

relevant tables
these are the tables that contain the necessary information for our analysis:
1. products – contains details about each product, including inventory levels.
2. warehouses – stores information about different storage locations.
3. orders – contains customer orders and their shipment details.
4. orderdetails – links products with orders, showing quantities sold.
5. payments – tracks customer payments and revenue flow.

important tables & columns
this section highlights the exact columns we will use in our analysis.
1. products (inventory management)
productcode – unique identifier for each product.
productname – name of the product.
quantityinstock – number of units available in stock.
productline – category of the product.

2. warehouses (storage & distribution)
warehousecode – unique identifier for each warehouse.
warehousename – name of the warehouse.
location – geographic location of the warehouse.

3. orders (order tracking & fulfillment)
ordernumber – unique identifier for each order.
orderdate – when the order was placed.
shippeddate – when the order was shipped.
status – current state of the order (shipped, pending, canceled).

4. orderdetails (product sales & movement)
ordernumber – links to the orders table.
productcode – links to the products table.
quantityordered – number of units ordered.
priceeach – price per unit of the product.

5. payments (financial analysis)
customernumber – links payments to specific customers.
paymentdate – when the payment was made.
amount – total amount paid.


step 3: defining analysis questions
once the key data points are clear, it's time to ask the right questions

warehouse analysis
before making any decision, first, we need to understand the warehouses.
how many unique warehouses exist? (COUNT(warehousecode))
which warehouse has the highest stock? (SUM(quantityinstock) GROUP BY warehousecode)
which warehouse has the least orders? (COUNT(ordernumber) GROUP BY warehousecode)
which warehouse is storing products that have not been sold? (LEFT JOIN products and orderdetails to find unsold stock)
this will help decide which warehouse should be closed and where inventory should be moved.

product analysis
after understanding the warehouses, the next step is to look at the products.
what products exist in inventory, and how much stock is available? (SELECT productname, quantityinstock FROM products)
which products have the highest sales volume? (SUM(quantityordered) GROUP BY productcode)
which products have high stock but low sales? (compare quantityinstock vs. SUM(quantityordered))
which products have zero sales in the last 6 months? (check orders within last 6 months and compare to productcode with no sales)
this helps in deciding which products to restock, discontinue, or redistribute.

order analysis
products move because of orders, so we need to check order patterns.
how many orders have been placed? (COUNT(ordernumber))
what is the monthly order trend? (GROUP BY MONTH(orderdate))
which products have the most orders? (COUNT(ordernumber) GROUP BY productcode ORDER BY COUNT DESC)
which orders have been delayed? (compare orderdate vs. shippeddate)
this gives clarity on demand patterns and warehouse efficiency.

payment analysis
finally, payments tell us about actual revenue flow.
what is the total revenue generated? (SUM(amount))
how many pending/unpaid orders exist? (filter status = 'pending')
which customers have made the largest payments? (SUM(amount) GROUP BY customernumber ORDER BY SUM(amount) DESC)
what is the monthly revenue trend? (GROUP BY MONTH(paymentdate))
this helps in identifying risks and ensuring that we don’t shut down a warehouse that actually brings in steady cash flow.


step 4: warehouse analysis

before making a decision on warehouse closure, we need to analyze stock distribution and sales performance across warehouses.
1️ total number of warehouses
first, we check how many warehouses exist.
SELECT COUNT(*) AS TOTAL_WAREHOUSES FROM WAREHOUSES;
insight: we found that there are 4 warehouses in total.

2 total_stock,total_sold, percentage of stock sold per warehouse
to get a clearer insight, we calculate the percentage of stock sold per warehouse.
SELECT P.WAREHOUSECODE,
SUM(P.QUANTITYINSTOCK) AS TOTAL_STOCK, 
SUM(OD.QUANTITYORDERED) AS TOTAL_SOLD, 
ROUND((SUM(OD.QUANTITYORDERED)/SUM(P.QUANTITYINSTOCK))*100,2) AS PERCENT_SOLD 
FROM ORDERDETAILS OD JOIN PRODUCTS P ON OD.PRODUCTCODE = P.PRODUCTCODE 
GROUP BY P.WAREHOUSECODE 
ORDER BY TOTAL_SOLD DESC;
insight: warehouse d has sold only 1% of its stock, making it a strong candidate for closure. warehouse b is also underperforming.

step 5: product analysis

analyzing product data is not just about checking what products exist—it’s about ensuring efficient stock distribution across warehouses.
the main focus here is on demand vs. supply mismatch to identify shortages and redistribution needs.
product analysis should help us evaluate warehouse efficiency, not just manage product movement. the goal is to determine which warehouse is inefficient based on stock and sales trends.

1️ understanding stock distribution per warehouse
first, we check which products are stored in which warehouses and their stock levels.
SELECT PRODUCTCODE, PRODUCTNAME, WAREHOUSECODE, QUANTITYINSTOCK
FROM PRODUCTS
ORDER BY WAREHOUSECODE, QUANTITYINSTOCK DESC;
insight:helps us see stock concentration in each warehouse.

2️ identifying slow-moving warehouses (high stock, low sales)
to find warehouses that are holding too much stock but selling less, we check stock vs. sales per warehouse.
SELECT 
    P.WAREHOUSECODE, 
    SUM(P.QUANTITYINSTOCK) AS TOTAL_STOCK, 
    SUM(OD.QUANTITYORDERED) AS TOTAL_SOLD, 
    ROUND((SUM(OD.QUANTITYORDERED) / SUM(P.QUANTITYINSTOCK)) * 100, 2) AS PERCENTAGE_SOLD
FROM PRODUCTS P
LEFT JOIN ORDERDETAILS OD ON P.PRODUCTCODE = OD.PRODUCTCODE
GROUP BY P.WAREHOUSECODE
ORDER BY PERCENTAGE_SOLD ASC;
insight:warehouses with low percentage sold → inefficient and potential candidates for closure.

3️ finding slow-moving products per warehouse
some warehouses might be holding products that don’t sell.
SELECT 
    P.WAREHOUSECODE, 
    P.PRODUCTCODE, 
    P.PRODUCTNAME, 
    P.QUANTITYINSTOCK, 
    COALESCE(SUM(OD.QUANTITYORDERED), 0) AS TOTAL_SOLD
FROM PRODUCTS P
LEFT JOIN ORDERDETAILS OD ON P.PRODUCTCODE = OD.PRODUCTCODE
GROUP BY P.WAREHOUSECODE, P.PRODUCTCODE, P.PRODUCTNAME
ORDER BY TOTAL_SOLD ASC;
insight:products with low sales but high stock → contributing to warehouse inefficiency.

4️ checking warehouses storing high-demand, low-supply products
to ensure we aren’t closing a warehouse with valuable products, we check which warehouses hold high-demand but low-stock products.
SELECT 
    P.WAREHOUSECODE, 
    P.PRODUCTCODE, 
    P.PRODUCTNAME, 
    SUM(OD.QUANTITYORDERED) AS TOTAL_DEMAND, 
    P.QUANTITYINSTOCK AS TOTAL_SUPPLY
FROM PRODUCTS P
LEFT JOIN ORDERDETAILS OD ON P.PRODUCTCODE = OD.PRODUCTCODE
GROUP BY P.WAREHOUSECODE, P.PRODUCTCODE, P.PRODUCTNAME
HAVING TOTAL_SUPPLY < TOTAL_DEMAND
ORDER BY TOTAL_DEMAND DESC;
insight:warehouses storing high-demand, low-stock products should NOT be closed.

5️ determining the final warehouse to close
after all analysis, we check which warehouse still holds too much unused stock.
SELECT 
    WAREHOUSECODE, 
    SUM(QUANTITYINSTOCK) AS REMAINING_STOCK
FROM PRODUCTS
GROUP BY WAREHOUSECODE
ORDER BY REMAINING_STOCK ASC;
insight:warehouse b with high remaining stock and low sales is the best candidate for closure.

step 6: order analysis

order analysis helps evaluate warehouse efficiency in handling orders. the goal is to determine which warehouses have low order volume, high delays, or poor order management.
1️ total number of orders per warehouse
to check how many orders each warehouse processes.
SELECT 
    P.WAREHOUSECODE, 
    COUNT(DISTINCT O.ORDERNUMBER) AS TOTAL_ORDERS
FROM PRODUCTS P
JOIN ORDERDETAILS OD ON P.PRODUCTCODE = OD.PRODUCTCODE
JOIN ORDERS O ON OD.ORDERNUMBER = O.ORDERNUMBER
GROUP BY P.WAREHOUSECODE
ORDER BY TOTAL_ORDERS ASC;
insight: warehouse A with low order volume may be inefficient or underutilized.

2️ delayed shipments per warehouse
to check which warehouses have the most late shipments.
SELECT 
    P.WAREHOUSECODE, 
    COUNT(O.ORDERNUMBER) AS DELAYED_ORDERS
FROM PRODUCTS P
JOIN ORDERDETAILS OD ON P.PRODUCTCODE = OD.PRODUCTCODE
JOIN ORDERS O ON OD.ORDERNUMBER = O.ORDERNUMBER
WHERE O.STATUS = 'Shipped' AND O.SHIPPEDDATE > O.ORDERDATE + INTERVAL 1 DAY
GROUP BY P.WAREHOUSECODE
ORDER BY DELAYED_ORDERS DESC;
insight: warehouse B with high delay counts could be causing logistical inefficiencies.


step 7: pricing analysis
pricing analysis helps determine which warehouses contribute the most to revenue and whether some warehouses only sell low-value products.
1️ average selling price per warehouse
to check if some warehouses are selling low-value products.
SELECT 
    P.WAREHOUSECODE, 
    ROUND(AVG(OD.PRICEEACH), 2) AS AVG_SELLING_PRICE
FROM PRODUCTS P
JOIN ORDERDETAILS OD ON P.PRODUCTCODE = OD.PRODUCTCODE
GROUP BY P.WAREHOUSECODE
ORDER BY AVG_SELLING_PRICE DESC;
insight: warehouse C selling cheaper products may be less profitable.

2️ total revenue contribution per warehouse
to check which warehouse generates the most revenue.
SELECT 
    P.WAREHOUSECODE, 
    ROUND(SUM(OD.QUANTITYORDERED * OD.PRICEEACH), 2) AS TOTAL_REVENUE
FROM PRODUCTS P
JOIN ORDERDETAILS OD ON P.PRODUCTCODE = OD.PRODUCTCODE
GROUP BY P.WAREHOUSECODE
ORDER BY TOTAL_REVENUE ASC;
insight: warehouse C with low total revenue may not justify operational costs.

Summary:
This warehouse analysis identifies underperforming warehouses using four key factors: 
warehouse analysis, product analysis, order analysis, and price analysis, offering insights for potential closures.
Solution:
Warehouse B, with high stock and low sales, is recommended for closure due to inefficiency in stock movement. 
Warehouse A, despite lower revenue, has high stock turnover and should remain operational. 
Redistributing stock from underperforming warehouses to efficient ones will ensure better use of space and higher overall warehouse efficiency. 
Regular performance monitoring is essential to maintain optimized operations.








