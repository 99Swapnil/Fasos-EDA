# Fasos-EDA
Exploratory Data Analysis on Fasos dummy data with advanced concepts of MySQL.

use fasos;

drop table if exists drivers;
create table drivers
		( driver_id integer not null unique,
         registration_date date not null);
insert into drivers(driver_id, registration_date)
values
	(1, '2021-01-01'),
	(2, '2021-01-03'),
    (3, '2021-01-08'),
    (4, '2021-01-15');

drop table if exists driver_order;
create table driver_order
	(order_id int not null unique,
    driver_id int,
    pickup_time datetime,
    distance varchar(7),
    duration varchar(10),
    cancellation varchar(50));
    
#Some alterations

alter table drivers change registration_date  regn_date date not null;

insert into driver_order(order_id, driver_id, pickup_time, distance, duration, cancellation)
values 
	(1,1,'2021-01-01 18:15:34','20km','32 minutes',''),
	(2,1,'2021-01-01 19:10:54','20km','27 minutes',''),
	(3,1,'2021-01-03 00:12:37','13.4km','20 mins','NaN'),
	(4,2,'2021-01-04 13:53:03','23.4','40','NaN'),
	(5,3,'2021-01-08 21:10:57','10','15','NaN'),
	(6,3, null,null,null,'Cancellation'),
	(7,2,'2020-01-08 21:30:45','25km','25mins',null),
	(8,2,'2020-10-01 00:15:02','23.4 km','15 minute',null),
	(9,2, null,null,null,'Customer Cancellation'),
	(10,1,'2020-11-01 18:50:20','10km','10minutes',null);

drop table if exists ingredients;
create table ingredients (ingredients_id int not null unique, ingredient_name varchar(50));
insert into ingredients (ingredients_id, ingredient_name)
values
	(1,'BBQ Chicken'),
    (2, 'Chilli Sauce'),
    (3, 'Chicken'),
    (4, 'Cheese'),
    (5, 'Kebab'),
    (6, 'Mushrooms'),
    (7, 'Onions'),
    (8, 'Egg'),
    (9, 'Peppers'),
    (10, 'Schezwan Sauce'),
    (11, 'Tomatoes'),
    (12, 'Tomato Sauce');
    
drop table if exists rolls;
create table rolls (roll_id int unique not null, roll_name varchar(10));
alter table rolls change roll_name roll_name varchar(20);
insert into rolls (roll_id, roll_name)
values (1, 'Non Veg Roll'),
	   (2, 'Veg Roll');

drop table if exists rolls_recipes;
create table rolls_recipes (roll_id int unique not null, ingredients varchar (24));
insert into rolls_recipes(roll_id, ingredients)
values (1, '1,2,3,4,5,6,8,10'),
	   (2, '4,6,7,9,11,12');

drop table if exists customer_orders;
create table customer_orders 
(order_id int not null, customer_id int not null, 
roll_id int, not_include_items varchar(10), extra_items_included varchar(20), order_date datetime);
insert into customer_orders(order_id, customer_id, roll_id, not_include_items, extra_items_included, order_date)
values 
(1,101,1,'','','2021-01-01  18:05:02'),
(2,101,1,'','','2021-01-01 19:00:52'),
(3,102,1,'','','2021-01-02 23:51:23'),
(3,102,2,'','NaN','2021-01-02 23:51:23'),
(4,103,1,'4','','2021-01-04 13:23:46'),
(4,103,1,'4','','2021-01-04 13:23:46'),
(4,103,2,'4','','2021-01-04 13:23:46'),
(5,104,1,null,'1','2021-01-08 21:00:29'),
(6,101,2,null,null,'2021-01-08 21:03:13'),
(7,105,2,null,'1','2021-01-08 21:20:29'),
(8,102,1,null,null,'2021-01-09 23:54:33'),
(9,103,1,'4','1,5','2021-01-10 11:22:59'),
(10,104,1,null,null,'2021-01-11 18:34:49'),
(10,104,1,'2,6','1,4','2021-01-11 18:34:49');

#Analysis

# A. Rolls Matrices
#1. How many rolls were ordered?

select roll_id, count(roll_id) cnt 
from customer_orders
group by roll_id;

#2. How many unique customer orders were made?

select customer_id, count(customer_id) from customer_orders group by customer_id;

#alternative to 2

select count(distinct customer_id) from customer_orders;

#3. How many successful orders were delivered by each driver?

select driver_id, count(pickup_time) from driver_order
where cancellation not in ('Cancellation', 'Customer Cancellation')
group by driver_id;

#4. How many rolls of each type were delivered?

select roll_id, count(roll_id) Rolls_Ordered from 
(select * from 
(select *, case when cancellation in ('Cancellation', 'Customer Cancellation') then 'Cancelled' else 'Success' end as order_cancel_details 
from driver_order)a
where order_cancel_details = 'Success')b
inner join customer_orders c
on b.order_id = c.order_id
group by roll_id;

#5. How many veg and non veg rolls were ordered by each customer?

select a.*, b.roll_name 
from
(select customer_id, roll_id, count(roll_id) Rolls_Ordered
from customer_orders 
group by 1,2
order by 1)a 
inner join rolls b 
on a.roll_id = b.roll_id;

#6. What was the maximum number of rolls delivered in a single order?

select max(Rolls)
from 
(select c.order_id, count(c.roll_id) as Rolls from customer_orders c
inner join 
(select order_id from 
(select *, case when cancellation in ('Cancellation', 'Customer Cancellation')
then 'Cancelled' else 'Success' end as order_cancel_details 
from driver_order)a
where order_cancel_details = 'Success')b
on c.order_id = b.order_id
group by order_id)d;

#Alternate 

select * from 
(select *, rank() over(order by Rolls desc) rnk
from 
(select c.order_id, count(c.roll_id) as Rolls 
from customer_orders c
inner join 
(select order_id from 
(select *, case when cancellation in ('Cancellation', 'Customer Cancellation') 
then 'Cancelled' else 'Success' end as order_cancel_details 
from driver_order)a
where order_cancel_details = 'Success')b
on c.order_id = b.order_id
group by order_id)d)e
where rnk = 1;

#7. For each customer, how many delivered rolls had at least one change, and how many had no changes?

select customer_id, ch_no_ch, count(order_id) as at_least_1_change
from 
(select *, case when new_not_include_items = 0 and new_extra_items_included = 0 then 'No Change' else 'Change' end as ch_no_ch
from
(select order_id, roll_id, customer_id, case when not_include_items is null or not_include_items = '' then 0 else not_include_items end as new_not_include_items,
case when extra_items_included is null or extra_items_included = '' or extra_items_included = 'NaN' then 0 else extra_items_included end as new_extra_items_included
from
(select c.order_id, c.roll_id, customer_id, not_include_items, extra_items_included, order_cancel_details
from customer_orders c
inner join
(select * from 
(select *, case when cancellation in ('Cancellation', 'Customer Cancellation') then 'Cancelled' else 'Success' end as order_cancel_details
from driver_order)a
where order_cancel_details = 'Success')b
on b.order_id = c.order_id)d)e)f
group by 1,2;

#8. How many rolls were ordered that had both extras and exclusions?

select f.ch_no_ch, count(ch_no_ch) from 
(select *, case when new_not_include_items != 0 and new_extra_items_included != 0 then 'Have Both' else 'either 1' end as ch_no_ch
from
(select order_id, roll_id, customer_id, case when not_include_items is null or not_include_items = '' then 0 else not_include_items end as new_not_include_items,
case when extra_items_included is null or extra_items_included = '' or extra_items_included = 'NaN' then 0 else extra_items_included end as new_extra_items_included
from
(select c.order_id, c.roll_id, customer_id, not_include_items, extra_items_included, order_cancel_details
from customer_orders c
inner join
(select * from 
(select *, case when cancellation in ('Cancellation', 'Customer Cancellation') then 'Cancelled' else 'Success' end as order_cancel_details
from driver_order)a
where order_cancel_details = 'Success')b
on b.order_id = c.order_id)d)e)f
group by 1;

#9. What was the total number of rolls ordered from each hour of the day?

select * from customer_orders;

select time_block, count(order_id) from 
(select *, concat(hour(order_date), '-', hour(order_date)+1 ) time_block
from customer_orders)a
group by time_block
order by time_block;

#10. What was the number of orders for each day of the week?

select * from customer_orders;

select Day_of_week, count(distinct order_id) from 
(select *, dayname(order_date) as Day_of_week from customer_orders)a
group by 1;

#Driver and Customer Experience

#1. What was the average time each driver took to arrive at Fasos HQ to pickup the order?

select * from driver_order;
select * from customer_orders;

select e.driver_id, round(sum(TimeDifference)/count(order_id)) Average_mins from
(select *, case when M1<M2 then (M1+60 - M2) else (M1-M2) end as TimeDifference from
(select distinct order_id, customer_id, driver_id, M1, M2 from
(select a.order_id, a.customer_id, a.roll_id, a.order_date,
b.driver_id, b.pickup_time, minute(pickup_time) M1, minute(order_date) M2 from customer_orders a
inner join driver_order b
on a.order_id = b.order_id
where pickup_time is not null)c)d)e
group by driver_id;

#2. Is there any relationship b/w the number of rolls ordered and how long the order takes to prepare?

select * from customer_orders;
select * from driver_order;

select *, round(Time_taken/cnt, 1) Relationship from 
(select order_id, count(roll_id) cnt, round(sum(TimeDifference)/ count(roll_id)) Time_taken from
(select *, case when M1<M2 then (M1+60 - M2) else (M1-M2) end as TimeDifference from
(select order_id, customer_id, roll_id, driver_id, M1, M2 from
(select a.order_id, a.customer_id, a.roll_id, a.order_date,
b.driver_id, b.pickup_time, minute(pickup_time) M1, minute(order_date) M2 from customer_orders a
inner join driver_order b
on a.order_id = b.order_id
where pickup_time is not null)c)d)e
group by order_id)f;

#Observation: We can notice that there is a linear relation between the time taken and no. of roll, with only 2 outliers. 
#A Simple Linear Regression model can be prepared as follows: y = 10x 

#3. What was the average distance travelled for each customer?

select customer_id, sum(distance)/count(order_id) average from 
(select distinct a.order_id, a.customer_id, a.order_date,
b.driver_id, b.pickup_time, replace(b.distance,'km','')as  Distance, b.duration, b.cancellation from customer_orders a
inner join driver_order b
on a.order_id = b.order_id
where pickup_time is not null)c
group by customer_id;

#4. What was the difference between the longest and the shortest delivery times for all orders?

select * from driver_order;

select max(duration)-min(duration) from driver_order;

# Alternative, including data cleaning 

select max(duration)-min(duration) from
(select left(duration, 2) duration from 
driver_order 
where duration is not null)a;

#5. What was the average speed of each driver for each delivery and do you notice any trend for these values?

select order_id, driver_id, distance, cast((distance/duration) as decimal(4,2)) Speed from
(select * from driver_order where distance is not null)a;

#6. What is the successful delivery percentage for each driver?

select driver_id, round(sum(canc)/count(driver_id)*100) Success_Rate from
(select driver_id, case when (cancellation) like '%cancel%' then 0 else 1 end as canc from driver_order)a
group by driver_id;










