# electronic_stores_analytics
-- Create tables for customers, sales, products, stores and exchange rate to upload data into PostgreSQL

-- Link to dataset: https://www.kaggle.com/datasets/bhavikjikadara/global-electronics-retailers

-- Tableau - Electronic Stores Analytics Visualization: https://public.tableau.com/app/profile/huong6399/viz/GlobalElectronicStores/Overall

---- PostgreSQL Queries:

 -- Creat Customers Table

    CREATE TABLE customers (customer_key integer, gender text,
					   name text, city text, state_code text,
					   state text, zip_code varchar, country text,
					   continent text, birthday date);

    SELECT * FROM customers;

 -- Create Sales Table

    CREATE TABLE sales (order_number integer, line_item integer, 
				   order_date date, delivery_date date, 
				   customer_key integer, store_key integer, 
				   product_key integer, quantity integer,
				   currency_code text);

    SELECT * FROM sales;
    
 -- Create Products Table

    CREATE TABLE products (product_key integer, product_name varchar,
					  brand text, color text, unit_cost_USD money,
					  unit_price_USD money, subcategory_key integer,
					  subcategory varchar, category_key integer,
					  category text);

    SELECT * FROM products;
    
 -- Create Sales Table

    CREATE TABLE stores (store_key integer, country text, state text,
					square_meters integer, open_date date);

    SELECT * FROM stores;
    
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

 -- Average square meter by country

    SELECT country, 
	      ROUND(AVG(square_meters),3) AS average_square_meters
    FROM stores
    GROUP BY country
    ORDER BY AVG(square_meters) DESC;

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

 -- Profit by brand

    SELECT products.brand,
	      SUM((products.unit_price_usd - products.unit_cost_usd) * sales.quantity) AS profit
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.brand
    ORDER BY profit DESC;

 -- Brand sales rank	

    SELECT products.brand, RANK()OVER(ORDER BY SUM(products.unit_price_usd * sales.quantity) DESC)
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.brand

 -- Profit by subcategory

    SELECT products.subcategory,
	      SUM((products.unit_price_usd - products.unit_cost_usd) * sales.quantity) AS profit
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.subcategory
    ORDER BY profit DESC;

 -- Profit by category

    SELECT products.category,
	      SUM((products.unit_price_usd - products.unit_cost_usd) * sales.quantity) AS profit
    FROM products
    LEFT JOIN sales
    ON products.product_key = sales.product_key
    GROUP BY products.category
    ORDER BY profit DESC;

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

 -- Average of transaction that buy more than 2 products

    WITH quantity_table AS (
	      SELECT  order_number, SUM(quantity) AS total_quantity
	      FROM sales
	      GROUP BY order_number
	  )
    SELECT ROUND(100.0* COUNT(*)/ (SELECT COUNT(*) FROM quantity_table),3) AS percentage
    FROM quantity_table
    WHERE total_quantity > 1;

 -- Aggregated sales

    SELECT DISTINCT sales.order_date, 
	    SUM(sales.quantity*products.unit_price_usd) OVER (ORDER BY sales.order_date) AS sales
    FROM sales
    INNER JOIN products
    ON sales.product_key = products.product_key;

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

 -- Average time to deliver

    SELECT ROUND(AVG(delivery_date - order_date)::DECIMAL,3) AS average_day
    FROM sales
    WHERE delivery_date IS NOT NULL;

 -- Average quantities of products bought per order

    WITH quantity_table AS (
	      SELECT order_number, SUM(quantity) AS total_quantity
	      FROM sales
	      GROUP BY order_number
     )
    SELECT FLOOR(AVG(total_quantity)) AS average_quantity_per_order
    FROM quantity_table;

 -- Popular sales colors

    SELECT DISTINCT products.color, SUM(sales.quantity) OVER (PARTITION BY products.color) AS quantity
    FROM sales
    RIGHT JOIN products
    ON sales.product_key = products.product_key
    ORDER BY quantity DESC;

 -- Number of products offered by brand

    SELECT DISTINCT brand, COUNT(product_key) AS number_of_products
    FROM products
    GROUP BY brand
    ORDER BY number_of_products DESC;

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

 -- Males Vs Females

    SELECT ROUND(100.0* COUNT(gender) FILTER(WHERE LOWER(gender) = 'male')/COUNT(*),3) AS male_percentage,
	       ROUND(100.0* COUNT(gender) FILTER(WHERE LOWER(gender) = 'female')/COUNT(*),3) AS female_percentage
    FROM customers;

 -- Customers by country

    SELECT country, COUNT(customer_key) AS number_of_customers
    FROM customers
    GROUP BY country
    ORDER BY number_of_customers DESC;

 -- Average Age

    SELECT ROUND(AVG(EXTRACT(year FROM NOW()) - EXTRACT(year FROM birthday))) AS average_age
    FROM customers;

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

 -- Aggregated sales by country

    SELECT DISTINCT sales.order_date, stores.country, 
        SUM(sales.quantity * products.unit_price_usd)OVER(ORDER BY sales.order_date)
    FROM stores
    INNER JOIN sales
    ON stores.store_key = sales.store_key
    INNER JOIN products
    ON sales.product_key = products.product_key
    ORDER BY stores.country;

 -- Average time to deliver by year

    SELECT DISTINCT EXTRACT (year from order_date) AS year,
			     ROUND(AVG(delivery_date - order_date) 
			     OVER(PARTITION BY EXTRACT(year from order_date)),3) AS average_delivery_days
    FROM sales
    WHERE delivery_date IS NOT NULL
    ORDER BY year;

 -- Volumn order over years

    SELECT DISTINCT order_date, SUM(quantity) OVER (ORDER BY order_date) AS quantity
    FROM sales
    ORDER BY order_date;

 -- Volumn order over days by year

    SELECT DISTINCT order_date, SUM(quantity)OVER(PARTITION BY EXTRACT(year from order_date)
						  ORDER BY order_date) AS quantity
    FROM sales
    ORDER BY order_date;


					

					
