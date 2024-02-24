# TheLook e-commerce - Returned product analysis
A project analyzing an e-commerce dataset, with a focus on returned products, using SQL and Power BI.
## 1. Basic information
### Dataset
This project uses a public dataset hosted on the Kaggle platform, which encompasses a comprehensive array of information, including orders, web events, products, customers, and distribution centers, all pertinent to ecommerce operations.
<br><br> Here is the [link](https://www.kaggle.com/datasets/daichiuchigashima/thelook-ecommerce) to the dataset.
<br><br> Details of each table of the dataset is described below.
#### distribution_centers.csv
- id: Unique identifier for each distribution center.
- name: Name of the distribution center.
- latitude: Latitude coordinate of the distribution center.
- longitude: Longitude coordinate of the distribution center.
#### events.csv
- id: Unique identifier for each event.
- user_id: Identifier for the user associated with the event.
- sequence_number: Sequence number of the event.
- session_id: Identifier for the session during which the event occurred.
- created_at: Timestamp indicating when the event took place.
- ip_address: IP address from which the event originated.
- city: City where the event occurred.
- state: State where the event occurred.
- postal_code: Postal code of the event location.
- browser: Web browser used during the event.
- traffic_source: Source of the traffic leading to the event.
- uri: Uniform Resource Identifier associated with the event.
- event_type: Type of event recorded.
#### inventory_items.csv
- id: Unique identifier for each inventory item.
- product_id: Identifier for the associated product.
- created_at: Timestamp indicating when the inventory item was created.
- sold_at: Timestamp indicating when the item was sold.
- cost: Cost of the inventory item.
- product_category: Category of the associated product.
- product_name: Name of the associated product.
- product_brand: Brand of the associated product.
- product_retail_price: Retail price of the associated product.
- product_department: Department to which the product belongs.
- product_sku: Stock Keeping Unit (SKU) of the product.
- product_distribution_center_id: Identifier for the distribution center associated with the product.
#### order_items.csv
- id: Unique identifier for each order item.
- order_id: Identifier for the associated order.
- user_id: Identifier for the user who placed the order.
- product_id: Identifier for the associated product.
- inventory_item_id: Identifier for the associated inventory item.
- status: Status of the order item.
- created_at: Timestamp indicating when the order item was created.
- shipped_at: Timestamp indicating when the order item was shipped.
- delivered_at: Timestamp indicating when the order item was delivered.
- returned_at: Timestamp indicating when the order item was returned.
#### orders.csv
- order_id: Unique identifier for each order.
- user_id: Identifier for the user who placed the order.
- status: Status of the order.
- gender: Gender information of the user.
- created_at: Timestamp indicating when the order was created.
- returned_at: Timestamp indicating when the order was returned.
- shipped_at: Timestamp indicating when the order was shipped.
- delivered_at: Timestamp indicating when the order was delivered.
- num_of_item: Number of items in the order.
#### products.csv
- id: Unique identifier for each product.
- cost: Cost of the product.
- category: Category to which the product belongs.
- name: Name of the product.
- brand: Brand of the product.
- retail_price: Retail price of the product.
- department: Department to which the product belongs.
- sku: Stock Keeping Unit (SKU) of the product.
- distribution_center_id: Identifier for the distribution center associated with the product.
#### users.csv
- id: Unique identifier for each user.
- first_name: First name of the user.
- last_name: Last name of the user.
- email: Email address of the user.
- age: Age of the user.
- gender: Gender of the user.
- state: State where the user is located.
- street_address: Street address of the user.
- postal_code: Postal code of the user.
- city: City where the user is located.
- country: Country where the user is located.
- latitude: Latitude coordinate of the user.
- longitude: Longitude coordinate of the user.
- traffic_source: Source of the traffic leading to the user.
- created_at: Timestamp indicating when the user account was created.
### Objective
The primary focus of this project is to address the challenge of high product returns. Through comprehensive analysis, this project aims to uncover the underlying reasons behind this problem and propose actionable solutions to mitigate this issue.
## 2. Exploratory Data Analysis
### Keep the dataset relevant
To keep things clear and consistent, this project only looks at orders marked 'Complete' or 'Returned', focusing on understanding returned products. The 'users' table has information about users worldwide. However, since we'll be considering the distance between distribution centers and customers, we're starting by analyzing orders from users within the United States. This decision helps us stay organized and ensures that our analysis matches the location of all distribution centers, which are in the US. By focusing on orders from the US, we make it easier to study ecommerce returns effectively.
```sql
--Only include records of complete and returned orders in US
DELETE FROM users WHERE country <> 'United States';
DELETE FROM order_items WHERE status NOT IN ('Complete','Returned');
```
### Add in some useful data
To better understand the process of getting products to customers, we've added two new columns in the order_items table. These columns track how long it takes to prepare and ship items.
```sql
--Calculate preparation time and shipping time in minutes
ALTER TABLE order_items 
	ADD COLUMN prep_time int,
	ADD COLUMN ship_time int;
UPDATE order_items SET prep_time = EXTRACT(EPOCH FROM (shipped_at - created_at))/60;
UPDATE order_items SET ship_time = EXTRACT(EPOCH FROM (delivered_at - shipped_at))/60;
```
Understanding the impact of distance on customer satisfaction is also important in minimizing product returns. Therefore, additional columns will be created in the order_items table to include data related to distance. Before delving into this, let's outline the process of preparing and shipping products to customers. The dataset reveals that even if an order contains items from different distribution centers, they all share the same timestamp for being shipped and delivered. This suggests that they are consolidated at one distribution center before being shipped to the customer. For optimal efficiency, this consolidation occurs at the distribution center closest to the customer. 
<br><br>Please refer to the diagram below for a clearer visualization of this process.
<br><br><p align="center"><img width="459" alt="image" src="https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/99746bfc-b74e-4dde-8b36-e596d17def01">

<br>So we need to calculate 2 kind of distance:
- Distance from the start distribution center to the end distribution center (or s2e_distance)
- Distance from the end distribution center to the customer (or e2c_distance)
```sql
--Calculate e2c_distance - distance from the nearest distribution center to customer
CREATE EXTENSION cube;
CREATE EXTENSION earthdistance;

ALTER TABLE order_items ADD COLUMN e2c_distance float ;

UPDATE order_items oi1 SET e2c_distance =
(SELECT MIN(distance) AS distance
FROM 
(SELECT 
	oi.order_id,
	oi.product_id,	 
	(point(dc.longitude,dc.latitude)) <@> (point(u.longitude, u.latitude)) AS distance
FROM distribution_centers dc 
JOIN products p
ON dc.id = p.distribution_center_id 
JOIN order_items oi 
ON p.id = oi.product_id
JOIN users u 
ON oi.user_id = u.id) t1
WHERE oi1.order_id = t1.order_id
GROUP BY order_id);

--Find the ending distribution center (nearest to the customer)
ALTER TABLE order_items ADD COLUMN ending_distribution_center INT;

UPDATE order_items oi1 SET ending_distribution_center =
(SELECT
	DISTINCT distribution_center_id
FROM 
(SELECT 
	oi.*,
	p.distribution_center_id,
	(point(dc.longitude,dc.latitude)) <@> (point(u.longitude, u.latitude)) AS distance
FROM distribution_centers dc 
JOIN products p
ON dc.id = p.distribution_center_id 
JOIN order_items oi 
ON p.id = oi.product_id
JOIN users u 
ON oi.user_id = u.id
ORDER BY 1) t1
WHERE distance = e2c_distance AND oi1.order_id = t1.order_id)

--Calculate distance from starting to ending distribution center
ALTER TABLE order_items ADD COLUMN s2e_distance float;

UPDATE order_items oi SET s2e_distance =
(SELECT (point(starting_lon,starting_lat)) <@> (point(ending_lon, ending_lat))
FROM 
(SELECT 
	oi.order_id,
	oi.id,
	p.distribution_center_id AS starting_distribution_center,
	dc.latitude AS starting_lat,
	dc.longitude AS starting_lon,
	oi.ending_distribution_center,
	dc2.latitude AS ending_lat,
	dc2.longitude AS ending_lon
FROM distribution_centers dc 
JOIN products p
ON dc.id = p.distribution_center_id 
JOIN order_items oi 
ON p.id = oi.product_id
JOIN distribution_centers dc2 
ON oi.ending_distribution_center = dc2.id) d1
WHERE oi.id = d1.id)
```
### Validating the data
To ensure our data is fit for analysis, we conducted validation checks. These included searching for duplicates, null values, outliers, and any inappropriate data.
<br><br>The validation process revealed that there are no duplicates or nulls in the tables used for analysis. However, outliers were detected when examining the locations of customers within the United States. These outliers originate from customers in Alaska and Hawaii, with distances ranging from approximately 2,300 miles to 4,900 miles. This is significantly higher than the distances from other states, with a mean of around 3,500 miles compared to a mean of around 900 miles and a maximum of around 2,600 miles from the remaining states. As these outliers could skew our distance analysis and represent only 0.9% of total products sold, they will be removed from our dataset.
<br><br>Additionally, some anomalies were observed during descriptive statistics for the prep_time and ship_time columns. 
```sql
--Descriptive statistics for prep_time and ship_time
SELECT 
	'prep_time',
	AVG(oi.prep_time) AS mean,
	MIN(oi.prep_time) AS min,
	PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY oi.prep_time) AS percentile_25,
	PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY oi.prep_time) AS percentile_50,
	PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY oi.prep_time) AS percentile_75,
	MAX(oi.prep_time) AS max,
	stddev(oi.prep_time) AS std 
FROM order_items oi 
JOIN users u 
ON oi.user_id = u.id

UNION 

SELECT 
	'ship_time',
	AVG(oi.ship_time) AS mean,
	MIN(oi.ship_time) AS min,
	PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY oi.ship_time) AS percentile_25,
	PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY oi.ship_time) AS percentile_50,
	PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY oi.ship_time) AS percentile_75,
	MAX(oi.ship_time) AS max,
	stddev(oi.ship_time) AS std 
FROM order_items oi 
JOIN users u 
ON oi.user_id = u.id;
```
The prep_time column contains negative values, with even its 25th quartile being negative. Moreover, the ship_time column includes zero values, suggesting that some products were shipped before the order was created or that delivery took no time at all. These inconsistencies raise concerns about the reliability of the time data for analysis purposes. Therefore, we have decided not to conduct any analysis related to the time data.
<br><br> And now the data is fully ready for analysis! Let's dive in!
### Data Analysis
<br><p align="center">![image](https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/e09d8d4a-1dc8-40f3-a1a2-3d485cd07135)

<br>The donut chart highlights that 28% of products are returned, indicating that for every 4 products ordered, 1 is returned.
<br><br><p align="center"> ![image](https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/32fa288a-d2dc-45db-92aa-49e43362aa62)
<br><p align="center"> _Filter only complete orders_
<br><p align="center">![image](https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/180acb2c-f286-4530-a577-732fd14c2aa9)
<br><p align="center"> _Filter only returned orders_

<br> When examining traffic sources and age demographics, there isn't a significant difference in how complete and returned orders are distributed. This suggests that factors such as where customers come from and their age don't have a substantial impact on whether an order is completed or returned.

<br><p align="center"><img width="661" alt="image" src="https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/49710265-d627-43ae-b9d1-59d609aa9b50">

<br> However, when looking at product categories, we observe that the top 5 categories with the highest return rates are jumpsuits & rompers, underwear, plus, pants and shorts. This suggests that our marketing efforts should prioritize addressing issues within these categories first.

<br> While we currently lack specific information on why customers return products, further research such as customer surveys could provide valuable insights. Some potential actions to reduce returns include enhancing user understanding, improving clarity in size measurements, providing more sample pictures, and offering detailed measurement guidelines. These actions can help us better meet customer expectations and reduce return rates over time.

<br><p align="center">![image](https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/ce9c90a1-f6bd-42fe-89b2-5a854ca147fa)
<br> _Filter only complete orders_
<br><p align="center">![image](https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/1442b757-48c2-4ea9-a283-992746fa8757)
<br> _Filter only returned orders_

<br> The geo plot shows that the majority of orders are concentrated in the Eastern region. In contrast, for the Western region, Los Angeles emerges as the sole city receiving significant numbers of orders.  There is no discernible difference in how complete and returned orders are distributed across different US states.

<br><p align="center"> ![image](https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/7ea52495-c78b-47e2-9cd4-7d25ff67ae7f)

<br> The analysis of the distance between starting and ending distribution centers reveals notable efficiency in Memphis and Chicago. These centers, which also rank highly in terms of the number of product orders, exhibit a tight spread of distances. This suggests that they are strategically located close to other distribution centers with necessary products, resulting in minimized delivery costs.

<br> Conversely, Los Angeles, despite ranking fifth in product orders, has the highest median distance. Similarly, Houston, ranking fourth, also exhibits a high median and spread of distances, second only to Los Angeles. This indicates that the products available in the Los Angeles and Houston distribution centers may not fully cater to customer needs. Furthermore, the distance between these centers and others with required products is considerably far. To address these inefficiencies, it is recommended that additional distribution centers be established in proximity to Los Angeles and Houston. This would facilitate more efficient delivery of products to customers in these regions, ultimately enhancing customer satisfaction and reducing delivery costs.

<br><p align="center"><img width="585" alt="image" src="https://github.com/thtrang294/thelook_returnedproduct/assets/150567547/e6b55c5b-b7c3-4365-9b0a-151405715931">

<br> The analysis of the distance between ending distribution centers and customers reveals generally shorter distances compared to those between starting and ending distribution centers. This indicates that the current distribution centers are operating effectively in fulfilling customer needs.

<br> Interestingly, despite having the highest median distance from starting to ending distribution centers, Los Angeles boasts the lowest median distance from ending distribution centers to customers. This underscores the significance of Los Angeles as a pivotal center for the Western region, emphasizing the need for heightened focus and strategic initiatives in this area.
# 3. Conclusion
