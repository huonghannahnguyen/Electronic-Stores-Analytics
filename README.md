# Electronic_stores_analytics
-- Create tables for customers, sales, products, stores and exchange rate to upload data into PostgreSQL

-- Link to dataset: https://www.kaggle.com/datasets/bhavikjikadara/global-electronics-retailers

-- Tableau - Electronic Stores Analytics Visualization: https://public.tableau.com/app/profile/huong6399/viz/GlobalElectronicStores/Overall

![image](https://github.com/user-attachments/assets/fffac77b-1b1a-499e-92d3-e9e4e3d44ea0)

---- PostgreSQL Queries:

 -- Creat Customers Table

    CREATE TABLE customers (customer_key integer, gender text,
					   name text, city text, state_code text,
					   state text, zip_code varchar, country text,
					   continent text, birthday date);

    SELECT * FROM customers;

![image](https://github.com/user-attachments/assets/b9627e5e-acde-4f12-a9b0-53c945206a03)

 -- Create Sales Table

    CREATE TABLE sales (order_number integer, line_item integer, 
				   order_date date, delivery_date date, 
				   customer_key integer, store_key integer, 
				   product_key integer, quantity integer,
				   currency_code text);

    SELECT * FROM sales;

![image](https://github.com/user-attachments/assets/8aeede57-648a-4e9c-85e3-b90e1d2e6d28)

 -- Create Products Table

    CREATE TABLE products (product_key integer, product_name varchar,
					  brand text, color text, unit_cost_USD money,
					  unit_price_USD money, subcategory_key integer,
					  subcategory varchar, category_key integer,
					  category text);

    SELECT * FROM products;

![image](https://github.com/user-attachments/assets/60604372-2961-4d49-8bb6-bb1cd1ed7564)

 -- Create Stores Table

    CREATE TABLE stores (store_key integer, country text, state text,
					square_meters integer, open_date date);

    SELECT * FROM stores;

![image](https://github.com/user-attachments/assets/4e51400c-e43f-49fa-b988-500385897238)

 -- Create Exchange_rates Table

    CREATE TABLE exchange_rates (date date, currency text,
							exchange decimal);

    SELECT * FROM exchange_rates;

---- SQL queries for stores, sales and products analytics:

 -- Number of stores by country

    SELECT country, COUNT(*) AS number_of_stores
    FROM stores
    GROUP BY country
    ORDER BY COUNT(*) DESC;

![image](https://github.com/user-attachments/assets/37681a43-6a55-4e3d-98a1-f3cb0d31d85a)

 -- Average square meter by country

    SELECT country, 
	      ROUND(AVG(square_meters),3) AS average_square_meters
    FROM stores
    GROUP BY country
    ORDER BY AVG(square_meters) DESC;

![image](https://github.com/user-attachments/assets/73366374-e01f-409c-be0d-dc1323772ca9)

 -- Profit by product

    SELECT products.product_key, products.product_name, 
	      SUM(sales.quantity * products.unit_price_USD) AS total_sales,
	      SUM(sales.quantity * (products.unit_price_USD - products.unit_cost_USD)) AS total_profit,
	      products.brand, products.subcategory, products.category
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.product_key, products.product_name, products.brand, products.subcategory,
        products.category
    ORDER BY total_profit DESC;

![image](https://github.com/user-attachments/assets/19392b34-7ca2-4461-b2b5-6186dd53e4b6)

 -- Profit by store

    WITH product_sale_table AS (
	    SELECT sales.order_number, sales.customer_key, sales.store_key, sales.quantity,
		      sales.order_date, sales.delivery_date, sales.line_item,
		      products.product_key, products.product_name, products.brand,
		      products.unit_cost_USD, products.unit_price_USD, 
		      products.subcategory, products.category
	    FROM sales
	    INNER JOIN products
	    ON sales.product_key = products.product_key
    ),

    store_profit_table AS (
	    SELECT store_key, SUM(quantity*unit_price_usd) AS total_sales, 
	  	    SUM(quantity*unit_cost_usd) AS total_cost, 
		      SUM(quantity*(unit_price_usd - unit_cost_usd)) AS total_profit
	    FROM product_sale_table
	    GROUP BY store_key
	    ORDER BY total_profit DESC
    )

    SELECT stores.country, stores.state, stores.store_key,
	      store_profit_table.total_sales, store_profit_table.total_cost, 
	      store_profit_table.total_profit
    FROM store_profit_table
    RIGHT JOIN stores
    ON stores.store_key = store_profit_table.store_key
    ORDER BY store_profit_table.total_profit DESC;

![image](https://github.com/user-attachments/assets/579bf805-2fbc-4ef1-b7c2-b0c92f163089)

 -- Profit by brand

    SELECT products.brand,
	      SUM((products.unit_price_usd - products.unit_cost_usd) * sales.quantity) AS profit
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.brand
    ORDER BY profit DESC;

![image](https://github.com/user-attachments/assets/d7aced22-e3f5-4a9c-9edf-625fe2ab1d6a)

 -- Brand sales rank	

    SELECT products.brand, RANK()OVER(ORDER BY SUM(products.unit_price_usd * sales.quantity) DESC)
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.brand

![image](https://github.com/user-attachments/assets/3d346421-e448-4e0d-94d8-e4dd9d25f099)

 -- Profit by subcategory

    SELECT products.subcategory,
	      SUM((products.unit_price_usd - products.unit_cost_usd) * sales.quantity) AS profit
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.subcategory
    ORDER BY profit DESC;

![image](https://github.com/user-attachments/assets/e4c57685-9a00-4b39-b598-941059bf3c9f)

 -- Profit by category

    SELECT products.category,
	      SUM((products.unit_price_usd - products.unit_cost_usd) * sales.quantity) AS profit
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.category
    ORDER BY profit DESC;

![image](https://github.com/user-attachments/assets/118432d7-8c87-435f-b26c-a1b84f4465ad)

 -- Sales by country

    SELECT customers.country, SUM(products.unit_price_usd * sales.quantity) AS sales,
	      ROUND( 100.0 * (SUM(products.unit_price_usd * sales.quantity)/ 
	      (SELECT SUM(products.unit_price_usd * sales.quantity) 
	  FROM products 
	  INNER JOIN sales
	  ON products.product_key = sales.product_key))::DECIMAL,3) AS percentage 
    FROM products
    INNER JOIN sales
    ON products.product_key = sales.product_key
    INNER JOIN customers
    ON sales.customer_key = customers.customer_key
    GROUP BY customers.country
    ORDER BY sales DESC;

![image](https://github.com/user-attachments/assets/c7537f7c-9d94-4da2-8408-b8e8b5dcf976)

 -- Average of transaction that buy more than 2 products

    WITH quantity_table AS (
	      SELECT  order_number, SUM(quantity) AS total_quantity
	      FROM sales
	      GROUP BY order_number
	  )
    SELECT ROUND(100.0* COUNT(*)/ (SELECT COUNT(*) FROM quantity_table),3) AS percentage
    FROM quantity_table
    WHERE total_quantity > 1;

![image](https://github.com/user-attachments/assets/11253dca-f6f5-41d7-81c9-43e1048faf89)

 -- Aggregated sales

    SELECT DISTINCT sales.order_date, 
	    SUM(sales.quantity*products.unit_price_usd) OVER (ORDER BY sales.order_date) AS sales
    FROM sales
    INNER JOIN products
    ON sales.product_key = products.product_key;

![image](https://github.com/user-attachments/assets/5889e4d0-9062-4f0e-a882-97d23e5cf81b)

 -- Top 100 customers who spent the most

    SELECT customers.name, customers.country, SUM(sales.quantity) AS quantity_bought, 
	      SUM(sales.quantity * products.unit_price_usd) AS total_amount
    FROM customers
    INNER JOIN sales
    ON customers.customer_key = sales.customer_key
    INNER JOIN products
    ON sales.product_key = products.product_key
    GROUP BY customers.name, customers.country
    ORDER BY total_amount DESC
    LIMIT 100;

![image](https://github.com/user-attachments/assets/344d738c-35ec-4573-b1dd-a99d1a620674)

 -- Average time to deliver

    SELECT ROUND(AVG(delivery_date - order_date)::DECIMAL,3) AS average_day
    FROM sales
    WHERE delivery_date IS NOT NULL;

![image](https://github.com/user-attachments/assets/9f7e1565-4789-4ff0-b718-85755e3db000)

 -- Average quantities of products bought per order

    WITH quantity_table AS (
	      SELECT order_number, SUM(quantity) AS total_quantity
	      FROM sales
	      GROUP BY order_number
     )
    SELECT FLOOR(AVG(total_quantity)) AS average_quantity_per_order
    FROM quantity_table;

![image](https://github.com/user-attachments/assets/956f6b45-cdbe-468e-81b4-bfcf23066840)

 -- Popular sales colors

    SELECT DISTINCT products.color, SUM(sales.quantity) OVER (PARTITION BY products.color) AS quantity
    FROM sales
    RIGHT JOIN products
    ON sales.product_key = products.product_key
    ORDER BY quantity DESC;

![image](https://github.com/user-attachments/assets/dd36a45b-b7ea-40b2-a1f4-8d4e47dd3fae)

 -- Number of products offered by brand

    SELECT DISTINCT brand, COUNT(product_key) AS number_of_products
    FROM products
    GROUP BY brand
    ORDER BY number_of_products DESC;

![image](https://github.com/user-attachments/assets/c380928d-2394-4f0f-8e71-4aeb6a663d41)

 -- Returned customers

    WITH customers_return AS(
	      SELECT customers.name, sales.order_date, customers.country,
		        ROW_NUMBER()OVER(PARTITION BY customers.name ORDER BY sales.order_date)
	      FROM customers
	      INNER JOIN sales
	      ON customers.customer_key = sales.customer_key
	      GROUP BY customers.name, sales.order_date, customers.country
	  )
    SELECT ROUND(100.0* COUNT(DISTINCT name) FILTER (WHERE row_number >1)::DECIMAL/ COUNT (DISTINCT name),3)
	  AS return_customers_percentage
    FROM customers_return;

![image](https://github.com/user-attachments/assets/1b8a58f2-98c2-49ba-a8f4-e6d6763537aa)

 -- Males Vs Females

    SELECT ROUND(100.0* COUNT(gender) FILTER(WHERE LOWER(gender) = 'male')/COUNT(*),3) AS male_percentage,
	       ROUND(100.0* COUNT(gender) FILTER(WHERE LOWER(gender) = 'female')/COUNT(*),3) AS female_percentage
    FROM customers;

![image](https://github.com/user-attachments/assets/e298f7f7-5002-48f2-90aa-2a251667b95a)

 -- Customers by country

    SELECT country, COUNT(customer_key) AS number_of_customers
    FROM customers
    GROUP BY country
    ORDER BY number_of_customers DESC;

![image](https://github.com/user-attachments/assets/6393cfd2-486f-49a9-b9b3-a27e8d9adc31)

 -- Average Age

    SELECT ROUND(AVG(EXTRACT(year FROM NOW()) - EXTRACT(year FROM birthday))) AS average_age
    FROM customers;

![image](https://github.com/user-attachments/assets/d29aafa5-4040-466a-a7b4-5cb50e7ee4d2)

 -- Age proportion

    WITH age_table AS(
	      SELECT EXTRACT(year FROM NOW()) - EXTRACT (year FROM birthday) AS age
	      FROM customers
    )
    SELECT ROUND(100.0*COUNT(age) FILTER (WHERE age > 1 AND age <= 20)::DECIMAL/ COUNT(*)) AS "1-20",
	    ROUND(100.0*COUNT(age) FILTER (WHERE age > 20 AND age <= 40)::DECIMAL/ COUNT(*)) AS "20-40",
	    ROUND(100.0*COUNT(age) FILTER (WHERE age > 40 AND age <= 60)::DECIMAL/ COUNT(*)) AS "40-60",
	    ROUND(100.0*COUNT(age) FILTER (WHERE age > 60 AND age <= 80)::DECIMAL/ COUNT(*)) AS "60-80",
	    ROUND(100.0*COUNT(age) FILTER (WHERE age > 80)::DECIMAL/ COUNT(*)) AS "80-100"
    FROM age_table;

![image](https://github.com/user-attachments/assets/0547652f-b4a9-40a0-9474-9c708a6056fd)

 -- Age proportion

    WITH age_table AS(
	      SELECT name, EXTRACT(year FROM NOW()) - EXTRACT (year FROM birthday) AS age
	      FROM customers
    ),
    age_range_table AS(
	      SELECT name, 
	    	CASE 
			    WHEN age > 1 AND age <= 20 THEN '1-20'
			    WHEN age > 20 AND age <= 40 THEN '20-40'
			    WHEN age > 40 AND age <= 60 THEN '40-60'
			    WHEN age > 60 AND age <= 80 THEN '60-80'
			    ELSE '80+'
		      END AS age_range
	      FROM age_table
    ),
    count_range_table AS(
	      SELECT DISTINCT age_range, COUNT(age_range)OVER(PARTITION BY age_range) 
	      FROM age_range_table
    )
    SELECT age_range, ROUND(100.0* count/(SELECT SUM(count) FROM count_range_table)) AS percentage
    FROM count_range_table;

![image](https://github.com/user-attachments/assets/03a931fa-856f-40b4-87a1-0b08156f86c9)

 -- Aggregated sales by country

    SELECT DISTINCT sales.order_date, stores.country, 
        SUM(sales.quantity * products.unit_price_usd)OVER(ORDER BY sales.order_date)
    FROM stores
    INNER JOIN sales
    ON stores.store_key = sales.store_key
    INNER JOIN products
    ON sales.product_key = products.product_key
    ORDER BY stores.country;

![image](https://github.com/user-attachments/assets/de5bc995-52c5-413e-9613-825c8fe36d16)

 -- Average time to deliver by year

    SELECT DISTINCT EXTRACT (year from order_date) AS year,
			     ROUND(AVG(delivery_date - order_date) 
			     OVER(PARTITION BY EXTRACT(year from order_date)),3) AS average_delivery_days
    FROM sales
    WHERE delivery_date IS NOT NULL
    ORDER BY year;

![image](https://github.com/user-attachments/assets/ba26bd3f-d8be-4535-bd98-b56ee1be7f4f)

 -- Volumn order over years

    SELECT DISTINCT order_date, SUM(quantity) OVER (ORDER BY order_date) AS quantity
    FROM sales
    ORDER BY order_date;

![image](https://github.com/user-attachments/assets/a3ab9a31-765d-4ab0-99b2-1034d54513fa)

 -- Volumn order over days by year

    SELECT DISTINCT order_date, SUM(quantity)OVER(PARTITION BY EXTRACT(year from order_date)
						  ORDER BY order_date) AS quantity
    FROM sales
    ORDER BY order_date;

![image](https://github.com/user-attachments/assets/824d379c-5903-4273-a953-03c35a265782)
					

					
