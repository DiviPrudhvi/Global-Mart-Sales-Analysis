# Retail-Order-Analysis

### Project Statement: Retail Order Analysis of Global Mart Sales Data

**Objective:**
The goal of my project was to clean, analyze, and extract actionable insights from Global Mart's sales data. By examining sales trends, product performance, and regional variations, I aimed to identify key drivers of revenue and growth. Additionally, I conducted year-over-year comparisons to understand seasonal impacts and pinpoint areas for potential improvement.

**Scope of Work:**

1. **Data Collection and Preparation:**
   - **Data Import:** I used the Kaggle API to download the dataset and extracted the necessary files.
     ```python
     !kaggle datasets download ankitbansal06/retail-orders -f orders.csv
     import zipfile
     zip_ref = zipfile.ZipFile('orders.csv.zip')
     zip_ref.extractall()
     zip_ref.close()
     ```
   - **Data Loading:** Loaded the dataset into a Pandas DataFrame, handling null values and renaming columns for consistency.
     ```python
     import pandas as pd
     df = pd.read_csv('orders.csv', na_values=['Not Available', 'unknown'])
     df.columns = df.columns.str.lower().str.replace(' ', '_')
     ```

2. **Data Cleaning:**
   - **Date Conversion:** Converted the `order_date` column to datetime format.
     ```python
     df['order_date'] = pd.to_datetime(df['order_date'], format="%Y-%m-%d")
     ```
   - **Feature Engineering:** Derived new columns for `discount_price`, `sale_price`, and `profit`.
     ```python
     df['discount_price'] = df['list_price'] * df['discount_percent'] * 0.01
     df['sale_price'] = df['list_price'] - df['discount_price']
     df['profit'] = df['sale_price'] - df['cost_price']
     ```
   - **Column Removal:** Removed unnecessary columns to streamline the dataset.
     ```python
     df.drop(columns=['cost_price', 'list_price', 'discount_percent'], inplace=True)
     ```


3. **Data Export:**
   - **SQL Database Integration:** Loaded the cleaned dataset into an SQL Server database.
     ```python
     import sqlalchemy as sal
     engine = sal.create_engine('mssql://Lighting_Speed\\SQLEXPRESS/master?driver=ODBC+DRIVER+17+FOR+SQL+SERVER')
     conn = engine.connect()
     df.to_sql('df_orders', con=conn, index=False, if_exists='append')
     ```


4. **Data Analysis:**

   - **Top 10 Highest Revenue Generating Products:**
     ```sql
     -- Find top 10 highest revenue generating products
     SELECT TOP 10 product_id, SUM(sale_price) AS sales 
     FROM df_orders
     GROUP BY product_id
     ORDER BY sales DESC;
     ```

   - **Top 5 Highest Selling Products in Each Region:**
     ```sql
     -- Find top 5 highest selling products in each region
     WITH cte AS (
         SELECT region, product_id, SUM(sale_price) AS sales
         FROM df_orders
         GROUP BY region, product_id
     )
     SELECT * FROM (
         SELECT *, ROW_NUMBER() OVER (PARTITION BY region ORDER BY sales DESC) AS rn
         FROM cte
     ) A
     WHERE rn <= 5;
     ```

   - **Month-over-Month Sales Growth Comparison for 2022 and 2023:**
     ```sql
     -- Find month-over-month growth comparison for 2022 and 2023 sales
     WITH cte AS (
         SELECT YEAR(order_date) AS order_year, MONTH(order_date) AS order_month,
         SUM(sale_price) AS sales
         FROM df_orders
         GROUP BY YEAR(order_date), MONTH(order_date)
     )
     SELECT order_month,
     SUM(CASE WHEN order_year = 2022 THEN sales ELSE 0 END) AS sales_2022,
     SUM(CASE WHEN order_year = 2023 THEN sales ELSE 0 END) AS sales_2023
     FROM cte
     GROUP BY order_month;
     ```

   - **Monthly Highest Sales for Each Category:**
     ```sql
     -- For each category, which month had the highest sales
     WITH cte AS (
         SELECT category, FORMAT(order_date, 'yyyy-MM') AS order_year_month,
         SUM(sale_price) AS sales
         FROM df_orders 
         GROUP BY category, FORMAT(order_date, 'yyyy-MM')
     )
     SELECT * FROM (
         SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn 
         FROM cte
     ) a
     WHERE rn = 1;
     ```

   - **Sub-Category with Highest Profit Growth in 2023 Compared to 2022:**
     ```sql
     -- Which sub-category had the highest growth by profit in 2023 compared to 2022
     WITH cte AS (
         SELECT sub_category, YEAR(order_date) AS order_year,
         SUM(sale_price) AS sales
         FROM df_orders
         GROUP BY sub_category, YEAR(order_date)
     ),
     cte2 AS (
         SELECT sub_category,
         SUM(CASE WHEN order_year = 2022 THEN sales ELSE 0 END) AS sales_2022,
         SUM(CASE WHEN order_year = 2023 THEN sales ELSE 0 END) AS sales_2023
         FROM cte
         GROUP BY sub_category
     )
     SELECT TOP 1 *,
     (sales_2023 - sales_2022) * 100 / sales_2022 AS growth_percentage
     FROM cte2
     ORDER BY (sales_2023 - sales_2022) * 100 / sales_2022 DESC;
     ```

**Tools and Technologies:**
- Data Collection: Kaggle API
- Data Cleaning and Analysis: Python (Pandas), SQL
- Database: SQL Server

**Outcome:**
This project provided valuable insights for Global Mart stakeholders, enabling them to make informed decisions to enhance sales and profitability. My work highlighted critical areas for growth and optimization, offering a strategic advantage in the competitive retail market.
