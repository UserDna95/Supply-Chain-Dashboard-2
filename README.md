#Supply-Chain-Dashboard 2

Problem Statement: 
Use Kaggle set to create a supply chain dashboard. 

https://www.kaggle.com/datasets/harshsingh2209/supply-chain-analysis

Step 1) 
Clean data in Excel:

1) Handle null values with zero

2) Create necessary indexes for supply_id, order_id, etc. 

3) Change data types like defect rates to floating numbers, cost and revenue to whole numbers

![Screen Shot 2025-03-05 at 9 09 38 PM](https://github.com/UserDna95/Supply-Chain-Dashboard-2/blob/main/2025-03-05%20(12).png)

Step 2)
Upload and clean data in SQL

1) Attempted to insert data from the csv file into an empty table structure, but even after changing the directory and location of the file, the “secure-priv-authorization” did not accept it. 

Then attempted to load the LOCAL file using a query, but a connection between local system and MySQL was not established.

Ultimately, what worked was to upload the csv file into a table of its own using Data Import Wizard and then use the INSERT INTO statement to insert data from the csv file into the empty table structure. 

SQL statement creating a sales table and inserting date:

  ```
Create Sales Table
CREATE TABLE Sales (
    sales_id INT PRIMARY KEY,
    product_id INT,
    customer_id INT,
    products_sold INT,
    revenue INT,
    FOREIGN KEY (product_id) REFERENCES Products(product_id),
    FOREIGN KEY (customer_id) REFERENCES Customers(customer_id)
);


INSERT INTO sales(sales_id, product_id, customer_id, products_sold, revenue) 
SELECT sales_id, product_id, customer_id, products_sold, revenue FROM supplychaindata;
  ```

2) Though this is a mock data set, some numbers like; product_lead_time, order_quantities, revenue, and other columns did not make sense and were updated to reflect real-time data

  ```
SELECT * FROM supplychain.suppliers;

-- Update product_lead_time to a random number between 15 and 30
UPDATE Suppliers
SET product_lead_time = FLOOR(15 + RAND() * (30 - 15 + 1))
WHERE product_lead_time BETWEEN 1 AND 14 AND product_id IS NOT NULL;

SELECT * FROM supplychain.orders;

-- Update order_quantities to a number above 11
UPDATE Orders
Set order_quantities = FLOOR (11 + RAND() * (11 + 1))
Where order_quantities BETWEEN 1 AND 10 AND product_id IS NOT NULL; 
  ```

3) Do XYZ and ABC analysis using SQL and create the Product Analysis table

  ```
ABC/XYZ analysis uses CTE

-- Step 1: Calculate Annual Consumption Value
WITH AnnualConsumption AS (
    SELECT 
        p.product_id,
        p.product_price AS item_cost,
        s.products_sold AS annual_demand,
        (p.product_price * s.products_sold) AS annual_consumption_value
    FROM 
        products p
    JOIN 
        sales s ON p.product_id = s.product_id
),
-- Step 2: Perform ABC Analysis
RankedItems AS (
    SELECT product_id, annual_consumption_value,
           NTILE(100) OVER (ORDER BY annual_consumption_value DESC) AS percentile
    FROM AnnualConsumption
),
ABC_Categories AS (
    SELECT product_id, annual_consumption_value,
           CASE 
               WHEN percentile <= 20 THEN 'A'
               WHEN percentile <= 70 THEN 'B'
               ELSE 'C'
           END AS abc_category
    FROM RankedItems
),
-- Step 3: Perform XYZ Analysis
DemandStats AS (
    SELECT 
        p.product_id,
        s.products_sold AS annual_demand,
        CASE 
            WHEN s.products_sold < 50 THEN 'Z'  -- Low demand
            WHEN s.products_sold BETWEEN 50 AND 200 THEN 'Y'  -- Medium demand
            ELSE 'X'  -- High demand
        END AS xyz_category
    FROM 
        products p
    JOIN 
        sales s ON p.product_id = s.product_id
),
-- Step 4: Combine ABC and XYZ Analysis Results
CombinedData AS (
    SELECT a.product_id, a.annual_consumption_value, a.abc_category, d.annual_demand, d.xyz_category
    FROM ABC_Categories a
    JOIN DemandStats d ON a.product_id = d.product_id
),
-- Step 5: Rank Products by Annual Consumption Value
RankedProducts AS (
    SELECT product_id, annual_consumption_value, abc_category, xyz_category,
           RANK() OVER (ORDER BY annual_consumption_value DESC) AS sales_rank
    FROM CombinedData
)
-- Step 6: Retrieve Top 10 and Bottom 10 Products
SELECT 'Top 10 Products' AS category, product_id, annual_consumption_value, abc_category, xyz_category, sales_rank
FROM RankedProducts
WHERE sales_rank <= 10
UNION
SELECT 'Bottom 10 Products' AS category, product_id, annual_consumption_value, abc_category, xyz_category, sales_rank
FROM RankedProducts
WHERE sales_rank > (SELECT COUNT(*) FROM RankedProducts) - 10;

--Step 7:
CREATE TABLE product_analysis ( 
product_id INT PRIMARY KEY, 
annual_consumption_value INT, 
abc_category CHAR(1), 
annual_demand INT, 
xyz_category CHAR(1), 
sales_rank INT, 
category VARCHAR(20), 
FOREIGN KEY (product_id) REFERENCES products(product_id) 
);

INSERT INTO product_analysis (category, product_id, annual_consumption_value, abc_category, xyz_category, sales_rank)
VALUES ('Top 10 Products', 6, 87360, 'A', 'X', 1);
  ```

4) Similar to Product Analysis, do the Supplier Analysis 

  ```
-- Normalize supplier metrics
CREATE TEMPORARY TABLE normalized_suppliers AS 
SELECT 
    suppliers_id,
    product_id,
    (defect_rates - (SELECT MIN(defect_rates) FROM suppliers)) / 
    (SELECT MAX(defect_rates) - MIN(defect_rates) FROM suppliers) AS normalized_defect_rates,
    (suppliers_lead_time - (SELECT MIN(suppliers_lead_time) FROM suppliers)) / 
    (SELECT MAX(suppliers_lead_time) - MIN(suppliers_lead_time) FROM suppliers) AS normalized_lead_time,
    (cost_of_product - (SELECT MIN(cost_of_product) FROM suppliers)) / 
    (SELECT MAX(cost_of_product) - MIN(cost_of_product) FROM suppliers) AS normalized_cost_of_product,
    (shipping_cost - (SELECT MIN(shipping_cost) FROM suppliers)) / 
    (SELECT MAX(shipping_cost) - MIN(shipping_cost) FROM suppliers) AS normalized_shipping_cost
FROM 
    suppliers;

-- Calculate composite scores for suppliers
CREATE TEMPORARY TABLE supplier_scores AS
SELECT
    suppliers_id,
    product_id,
    normalized_defect_rates * 0.4 + 
    normalized_lead_time * 0.3 + 
    normalized_cost_of_product * 0.2 + 
    normalized_shipping_cost * 0.1 AS composite_score
FROM 
    normalized_suppliers;

-- Combine top and bottom suppliers into one table and rank them
CREATE TEMPORARY TABLE combined_top_bottom_suppliers AS
SELECT 
    suppliers_id, product_id, composite_score, 
    ROW_NUMBER() OVER (ORDER BY composite_score DESC) AS ranking
FROM 
    supplier_scores;

-- Retrieve top 5 suppliers
SELECT 
    suppliers_id, product_id, composite_score, ranking
FROM 
    combined_top_bottom_suppliers 
WHERE 
    ranking <= 5;

-- Retrieve bottom 5 suppliers
SELECT 
    suppliers_id, product_id, composite_score, ranking
FROM 
    combined_top_bottom_suppliers 
ORDER BY 
    composite_score ASC 
LIMIT 5;


CREATE TABLE suppliers_analysis ( 
product_id INT PRIMARY KEY, 
suppliers_id INT, 
composite_score INT, 
ranking INT, 
FOREIGN KEY (product_id) REFERENCES products(product_id) 
);

ALTER TABLE suppliers_analysis
ADD category VARCHAR(255);

INSERT INTO suppliers_analysis (product_id, suppliers_id, composite_score, ranking, category)
VALUES ('7', '3', '0.8761000', '1', 'Top 5 Suppliers');
  ```

5) Measure the COGS, Revenue Per Product, GMROI

  ```
COGS
ALTER TABLE suppliers
ADD COGS INT;
UPDATE suppliers
SET COGS = cost_of_product + shipping_cost;
ALTER TABLE suppliers
CHANGE COGS cogs INT;

REVENUE PER PRODUCT
ALTER TABLE sales
ADD revenue_per_product INT;
UPDATE sales
SET revenue_per_product = revenue/products_sold;

GMROI
ALTER TABLE sales
ADD COLUMN gross_profit INT;
UPDATE sales
JOIN suppliers ON sales.product_id = suppliers.product_id
SET sales.gross_profit = sales.revenue - suppliers.cogs
WHERE suppliers.cogs IS NOT NULL;
  ```

![Screen Shot 2025-03-05 at 9 09 38 PM](https://github.com/UserDna95/Supply-Chain-Dashboard-2/blob/main/supplychaindashboard1.png)

Step 3)
Model data in Power BI

1) Created new tables such as Date Table and SafetyStock Table to calculate Reorder Point and Safety Stock

  ```
date table = 
ADDCOLUMNS (
    CALENDAR (DATE(2022, 1, 1), DATE(2024, 12, 31)),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month Name", FORMAT([Date], "MMMM"),
    "Quarter", "Q" & FORMAT(QUARTER([Date]), "0"),
    "Day of Week", FORMAT([Date], "dddd"),
    "Week Number", WEEKNUM([Date])
)
  ```

  ```
 SafetyStockTable = 
ADDCOLUMNS (
    'supplychain orders',
    "Total lead time", RELATED('supplychain suppliers'[total_lead_time]),
    "Z-Score", 1.65,
    "Average Daily Usage", 
        VAR TotalProductsSold = SUMX('supplychain sales', 'supplychain sales'[products_sold])
        VAR NumberOfDays = DISTINCTCOUNT('date table'[Date])
        RETURN TotalProductsSold / NumberOfDays,
    "Average Lead Time Demand", 
        VAR ADU = 
            VAR TotalProductsSold = SUMX('supplychain sales', 'supplychain sales'[products_sold])
            VAR NumberOfDays = DISTINCTCOUNT('date table'[Date])
            RETURN TotalProductsSold / NumberOfDays
        VAR TotalLeadTime = RELATED('supplychain suppliers'[total_lead_time])
        RETURN ADU * TotalLeadTime,
    "Standard Deviation of Daily Usage", 
        STDEV.P('supplychain sales'[products_sold]),
    "Standard Deviation of Lead Time", STDEV.P('supplychain suppliers'[total_lead_time]),
    "Safety Stock", 
        VAR AvgLeadTimeDemand = 
            VAR ADU = 
                VAR TotalProductsSold = SUMX('supplychain sales', 'supplychain sales'[products_sold])
                VAR NumberOfDays = DISTINCTCOUNT('date table'[Date])
                RETURN TotalProductsSold / NumberOfDays
            VAR TotalLeadTime = RELATED('supplychain suppliers'[total_lead_time])
            RETURN ADU * TotalLeadTime
        VAR LeadTimeVar = STDEV.P('supplychain suppliers'[total_lead_time])
        VAR AvgDemandVar = STDEV.P('supplychain sales'[products_sold])
        VAR Z = 1.65
        RETURN Z * SQRT((AvgLeadTimeDemand * LeadTimeVar) + (AvgDemandVar * RELATED('supplychain suppliers'[total_lead_time])))
)
  ```

![Screen Shot 2025-03-05 at 9 09 38 PM](https://github.com/UserDna95/Supply-Chain-Dashboard-2/blob/main/2025-03-05%20(10).png)


Step 4) 
Dashboard 

Key Insights:
- Mumbai, Kolkata, and Chennai have the top suppliers
- For Top 10 products, the Reorder Point is incredibly high in comparison to the safety stock due to the high lead times and the high variance in daily demand
- All top-ranked products are high-value items with high predictability, which is a great indicator of a smooth operating supply chain 
- The Supplier and Customer flow use Sankey Charts to show the general flow of which product_type belongs to which category, and which supplier transportation mode and route falls into which category



![Screen Shot 2025-03-05 at 9 09 38 PM](https://github.com/UserDna95/Supply-Chain-Dashboard-2/blob/main/2025-03-05%20(13).png)
![Screen Shot 2025-03-05 at 9 09 38 PM](https://github.com/UserDna95/Supply-Chain-Dashboard-2/blob/main/2025-03-05%20(9).png)
