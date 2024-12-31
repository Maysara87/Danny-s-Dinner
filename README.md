# Danny-s-Dinner
SQL Case Study at 8weeksqlchallenge.com

Schema SQL
CREATE SCHEMA dannys_diner;
SET search_path = dannys_diner;

CREATE TABLE sales (
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);

INSERT INTO sales
  ("customer_id", "order_date", "product_id")
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);

INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');

/* --------------------
   Case Study Questions
   --------------------*/

-- 1. What is the total amount each customer spent at the restaurant?
-- 2. How many days has each customer visited the restaurant?
-- 3. What was the first item from the menu purchased by each customer?
-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
-- 5. Which item was the most popular for each customer?
-- 6. Which item was purchased first by the customer after they became a member?
-- 7. Which item was purchased just before the customer became a member?
-- 8. What is the total items and amount spent for each member before they became a member?
-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

-- Example Query:
SELECT
  	product_id,
    product_name,
    price
FROM dannys_diner.menu
ORDER BY price DESC
LIMIT 5;


-- 1. What is the total amount each customer spent at the restaurant?

select S.customer_id, sum(M.price) as total_spent
from sales S
join menu M on S.product_id = M.product_id
group by S.customer_id;


-- 2. How many days has each customer visited the restaurant?

select customer_id, count(order_date)
from sales
group by customer_id;


-- 3. What was the first item from the menu purchased by each customer?

select S.customer_id, M.product_name
from sales S
join menu M on S.product_id = M.product_id
where (S.customer_id, S.order_date) in (select customer_id, min(order_date) from sales group by customer_id)
order by S.customer_id;


-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

select M.product_name, Count(S.product_id)
from sales S
join menu M on S.product_id = M.product_id
group by M.product_name
order by count(S.product_id) desc 
limit 1;


-- 5. Which item was the most popular for each customer?

select customer_id, product_name, number_ordered
from (
	select 
		S.customer_id, 
		M.product_name, 
    	count(S.product_id) as number_ordered, 
    	rank() over (partition by S.customer_id order by count(S.product_id) desc) as ranks
	from sales S
	join menu M on S.product_id = M.product_id
	group by S.customer_id, M.product_name
) as filtered
where ranks = 1;


-- 6. Which item was purchased first by the customer after they became a member?

select S.customer_id, M.product_name, B.join_date, S.order_date
from sales S
join menu M on S.product_id = M.product_id
join members B on S.customer_id = B.customer_id
where S.order_date >= B.join_date 
	and (S.customer_id, S.order_date) in (
      select SA.customer_id, min(SA.order_date)
      from sales SA
      join members MB on SA.customer_id = MB.customer_id
      where SA.order_date >= MB.join_date
      group by SA.customer_id);


-- 7. Which item was purchased just before the customer became a member?

select S.customer_id, M.product_name, B.join_date, S.order_date
from sales S
left join menu M on S.product_id = M.product_id
left join members B on S.customer_id = B.customer_id
where (S.customer_id, S.order_date) in (
      select SA.customer_id, max(SA.order_date)
      from sales SA
      join members MB on SA.customer_id = MB.customer_id
      where SA.order_date < MB.join_date
      group by SA.customer_id) or
      (S.customer_id, S.order_date) in (
        select customer_id, max(order_date)
		from sales
		where customer_id not in (select customer_id from members)
		group by customer_id);        


-- 8. What is the total items and amount spent for each member before they became a member?

select S.customer_id, count(S.product_id), sum(M.price)
from sales S
left join menu M on S.product_id = M.product_id
left join members B on S.customer_id = B.customer_id
where S.order_date < B.join_date or
      S.customer_id not in (select customer_id from members)        
group by S.customer_id;


-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

select 
    S.customer_id, 
    sum(M.price * case when M.product_name = 'sushi' then 2 else 1 end) as total_points
from 
    sales S
join 
    menu M on S.product_id = M.product_id
group by 
    S.customer_id;



-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

select 
	S.customer_id,
    sum(M.price * case 
       when S.order_date >= B.join_date and S.order_date <= B.join_date + interval '7 days' then 2 else 
        case when M.product_name = 'sushi' then 2 else 1 end end) AS total_points
FROM sales S
left join menu M ON S.product_id = M.product_id
left join members B on S.customer_id = B.customer_id
group by S.customer_id;

