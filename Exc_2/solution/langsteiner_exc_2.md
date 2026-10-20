# 2)
## 2.1)
### Setup
```bash
DROP TABLE IF EXISTS orders;

CREATE TABLE orders (
    customer_name TEXT,
    product_category TEXT,
    quantity INTEGER,
    price_per_unit NUMERIC(10,2),
    order_date DATE,
    country TEXT
);
```
```bash
\COPY orders(customer_name, product_category, quantity, price_per_unit, order_date, country) FROM '/data/orders_1M.csv' DELIMITER ',' CSV HEADER;
```
Testing (count was 1000000)
```bash
postgres=# SELECT * FROM orders LIMIT 5;
  customer_name  | product_category | quantity | price_per_unit | order_date | country 
-----------------+------------------+----------+----------------+------------+---------
 Lucas Thomas    | Fashion          |        1 |          52.41 | 2024-04-14 | Brazil
 Victor Williams | Grocery          |        1 |           4.92 | 2024-08-11 | Germany
 Bob Gonzalez    | Grocery          |        5 |          72.17 | 2025-12-19 | Sweden
 Zoe Moore       | Health & Beauty  |        2 |         244.73 | 2024-01-07 | India
 Sara Moore      | Home & Garden    |        4 |          94.63 | 2024-12-10 | Canada
(5 rows)

Time: 0.679 ms
postgres=#
```
### a)
```bash
SELECT *
FROM orders
ORDER BY price_per_unit DESC
LIMIT 1;
```
```bash
customer_name | product_category | quantity | price_per_unit | order_date | country 
---------------+------------------+----------+----------------+------------+---------
 Emma Brown    | Automotive       |        3 |        2000.00 | 2024-10-11 | Italy
(1 row)

Time: 170.417 ms
```
### b)
```bash
SELECT product_category, SUM(quantity) AS total_quantity
FROM orders
GROUP BY product_category
ORDER BY total_quantity DESC
LIMIT 3;
```
```bash
product_category | total_quantity 
------------------+----------------
 Health & Beauty  |         300842
 Electronics      |         300804
 Toys             |         300598
(3 rows)

Time: 195.229 ms
```
### c)
```bash
SELECT
    product_category,
    SUM(price_per_unit * quantity) AS total_revenue
FROM orders
GROUP BY product_category
ORDER BY total_revenue DESC;
```
```bash
product_category | total_revenue 
------------------+---------------
 Automotive       |  306589798.86
 Electronics      |  241525009.45
 Home & Garden    |   78023780.09
 Sports           |   61848990.83
 Health & Beauty  |   46599817.89
 Office Supplies  |   38276061.64
 Fashion          |   31566368.22
 Toys             |   23271039.02
 Grocery          |   15268355.66
 Books            |   12731976.04
(10 rows)

Time: 419.328 ms
```
### d)
```bash
SELECT
    customer_name,
    SUM(price_per_unit * quantity) AS total_spending
FROM orders
GROUP BY customer_name
ORDER BY total_spending DESC
LIMIT 5;
```
```bash
customer_name  | total_spending 
----------------+----------------
 Carol Taylor   |      991179.18
 Nina Lopez     |      975444.95
 Daniel Jackson |      959344.48
 Carol Lewis    |      947708.57
 Daniel Young   |      946030.14
(5 rows)

Time: 524.141 ms
```

### e)
```bash
SELECT
    customer_name,
    COUNT(*) AS order_count,
    SUM(price_per_unit * quantity) AS total_spending
FROM orders
GROUP BY customer_name
ORDER BY total_spending DESC
LIMIT 20;
```
```bash
customer_name  | order_count | total_spending 
----------------+-------------+----------------
 Carol Taylor   |        1028 |      991179.18
 Nina Lopez     |         980 |      975444.95
 Daniel Jackson |        1033 |      959344.48
 Carol Lewis    |         943 |      947708.57
 Daniel Young   |         973 |      946030.14
 Alice Martinez |        1009 |      935100.02
 Ethan Perez    |         953 |      934841.24
 Leo Lee        |        1000 |      934796.48
 Eve Young      |         975 |      933176.86
 Ivy Rodriguez  |         971 |      925742.64
 Nina Robinson  |         989 |      923627.53
 John Thomas    |        1029 |      922858.59
 Eve Taylor     |         999 |      917620.11
 Henry Smith    |        1004 |      916273.48
 Mia Wilson     |         997 |      914740.05
 Liam Johnson   |         986 |      914266.79
 Alice Brown    |        1027 |      913262.30
 Bob Walker     |        1002 |      913202.74
 Olivia Sanchez |         960 |      912017.65
 Amelia Moore   |        1025 |      911627.44
(20 rows)

```

on average a customer has 947 orders as there are 1 million orders split between only 32 firstnames and 33 lastnames => 1056 customers. the orders are assigned randomly therefore the top customers just randomly got more orders / orders with a higher total cost. 

## 2.2)
The goal of the query is to find out how many pairs of people from the same county can be found. 

It is very slow as the join scales in O(n^2). 

### Step 1

| rows in table | join result (`COUNT(*)`) | time |
|---|---|---|
| 50 000 |27501822 |1670.751 ms |
| 100 000 |109946508 | 7674.990 ms|
| 200 000 |439395606 |34396.900 ms |

this also suggests a scaling of O(n^2) 
rounded time values would be 2sec -> 8sec -> 32sec

doubling the size of the problem takes 4 times as long

### Step 2
```bash
CREATE INDEX idx_people_100k_country
ON people_100k(country);

ANALYZE people_100k;
CREATE INDEX
Time: 97.168 ms
ANALYZE
Time: 93.321 ms
```

```bash
SELECT COUNT(*)
FROM people_100k p1
JOIN people_100k p2
    ON p1.country = p2.country;
```
using an index barely makes it faster (5290.152 ms). the problem is still the fact that we create an exessive amount of joins, the index cant really fix that.

### Step 3
grouping by countries, counting the people, square the counts and take the sum makes more sense. 

```bash
SELECT SUM(cnt * cnt)
FROM (
    SELECT country, COUNT(*) AS cnt
    FROM people_big
    WHERE country IS NOT NULL
    GROUP BY country
) t;
```
```bash
    sum     
-------------
 10983941260
(1 row)

Time: 406.771 ms
```


### Step 4

Optimization: rewriting the query using group by makes a much bigger difference than adding additional hardware as it avoids billions of joins. 

Actual Pairs: if all the actual pairs are needed, additional hardware would help of course (parallel processing) but the O(n^2) remains a big issue. 

OLTP databases are optimized for short transactions, not large analytical queries. such queries should be moved to dedicated OLAP systems. 

