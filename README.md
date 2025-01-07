# Zomato-Analysis
```sql
drop table if exists goldusers_signup;
CREATE TABLE goldusers_signup(userid integer,gold_signup_date date); 

INSERT INTO goldusers_signup(userid,gold_signup_date) 
 VALUES (1,'09-22-2017'),
(3,'04-21-2017');

drop table if exists users;
CREATE TABLE users(userid integer,signup_date date); 

INSERT INTO users(userid,signup_date) 
 VALUES (1,'09-02-2014'),
(2,'01-15-2015'),
(3,'04-11-2014');

drop table if exists sales;
CREATE TABLE sales(userid integer,created_date date,product_id integer); 

INSERT INTO sales(userid,created_date,product_id) 
 VALUES (1,'04-19-2017',2),
(3,'12-18-2019',1),
(2,'07-20-2020',3),
(1,'10-23-2019',2),
(1,'03-19-2018',3),
(3,'12-20-2016',2),
(1,'11-09-2016',1),
(1,'05-20-2016',3),
(2,'09-24-2017',1),
(1,'03-11-2017',2),
(1,'03-11-2016',1),
(3,'11-10-2016',1),
(3,'12-07-2017',2),
(3,'12-15-2016',2),
(2,'11-08-2017',2),
(2,'09-10-2018',3);


drop table if exists product;
CREATE TABLE product(product_id integer,product_name text,price integer); 

INSERT INTO product(product_id,product_name,price) 
 VALUES
(1,'p1',980),
(2,'p2',870),
(3,'p3',330);


select * from sales;
select * from product;
select * from goldusers_signup;
select * from users;

--1.What is total amount each customer spends in zomato

Select a.userid, sum(price) as total from sales as a join product as b on  a.product_id = b.product_id
group by userid

--2.How many days disd the customer visited zomato?


Select userid, count(distinct(created_date)) as numberofdays from sales group by userid

--3.What was the 1st product purchased by each of the customer?
Select * from 
(Select userid, product_id, created_date, rank() over (partition by userid order by created_date asc) as rank_ from sales) a
where rank_ = 1;

--4.What is the most purchased menu and how many times was it purchased by all cumtomers?

Select top 1 product_id from sales group by product_id order by count(product_id) desc

Select userid, count(product_id) as cnt from sales where product_id = 
(Select top 1 product_id from sales group by product_id order by count(product_id) desc)
group by userid

--5.Which item was the most popular for each customer?
Select userid,product_id, cnt from
(Select userid,product_id, cnt, rank() over (partition by userid order by cnt desc) as _rank from
(Select userid, product_id, count(product_id) as cnt from sales group by userid, product_id) a ) b
where _rank = 1

--6. Which item was purchased first by the cumtomer after theu became a gold member.
 Select d.userid, d.created_date, d.product_id, d.gold_signup_date from 
(Select  c.userid, c.created_date, c.product_id, c.gold_signup_date, rank() over (partition by userid order by c.created_date asc) as _rank from
(Select a.userid, a.created_date, a.product_id, b.gold_signup_date from sales as a join goldusers_signup as b  on a.userid = b.userid and created_date > gold_signup_date) as c) d
where _rank = 1

--7.Which item was purchased just before the customer became the member?
Select * from 
(Select c.*, rank() over (partition by userid order by created_date desc) as _rank from
(Select a.userid, a.created_date, a.product_id, b.gold_signup_date from sales as a join goldusers_signup as b  on a.userid = b.userid and created_date <= gold_signup_date) c )d where _rank = 1;

--8.what is the total order and amount spent for each member before they become the member??

Select d.userid, count(d.product_id) as total_order, sum(d.price) as totalamountspent from
(Select a.userid, a.created_date, a.product_id, b.price, c.gold_signup_date from sales as a join product as b on a.product_id = b.product_id join goldusers_signup as c on a.userid = c.userid and created_date <= gold_signup_date) d group by userid;

--9.if buying each product generates points for eg 5rs = 2 zomato points and each product has different purchasing points for eg for p1 5rs = 1 zomato point , for p2 10rs = 5 zomato points and p3 5rs = 1 zomato point  
Select f.userid, sum(zomatopoint) as totalpoints, sum(zomatopoint) * 2.5 as cashback  from
(Select e.*, amt/points as zomatopoint from
(Select d.*, case when product_id=1 then 5 when product_id = 2 then 2 when product_id =3 then 5 end as points from 
(Select c.userid, c.product_id,sum(c.price) as amt from
(Select a.userid, a.product_id, b.price from sales as a join product as b on a.product_id = b.product_id) c
group by c.userid, c.product_id) d) e) f
group by userid;

--10.Calculate points collected by each customer and for which product most point has given till now? 

Select top 1 f.product_id, sum(zomatopoint) as totalpoints from
(Select e.*, amt/points as zomatopoint from
(Select d.*, case when product_id=1 then 5 when product_id = 2 then 2 when product_id =3 then 5 end as points from 
(Select c.userid, c.product_id,sum(c.price) as amt from
(Select a.userid, a.product_id, b.price from sales as a join product as b on a.product_id = b.product_id) c
group by c.userid, c.product_id) d) e) f group by product_id order by totalpoints desc;

--10 In the first one year after a customer join the gold program (including their join date) irrespective
of what the customer has purchased they earn 5 zomato points for every 10rs spent who earned more 1 or 3 
and what was their points earnings in their first yr ?


Select  d.userid,d.created_date, d.product_id, d.price, d.gold_signup_date , d.price/2 as points from
(Select a.userid,a.created_date, a.product_id, b.price, c.gold_signup_date from sales as a join product as b on a.product_id = b.product_id join goldusers_signup as c on a.userid =c.userid and  created_date >= gold_signup_date and created_date <= dateadd(year, 1, gold_signup_date)) d;


--11. rank all the transactions of the customers 

Select a.userid,a.created_date,a.product_id, rank() over (partition by userid order by created_date) as _rank from Sales as a join product as b on a.product_id = b.product_id

--12. Rank all the transactions for each member whenever they are zomato gold member  for every non gold member transaction mark as na 
Select e.*, case when _rank = 0 then 'NA' else _rank end _rnk from
(Select c.*, cast((case when gold_signup_date is Null then 0 else rank() over (partition by userid order by created_date desc) end) as varchar) as _rank from
(Select a.userid,a.created_date,a.product_id,b.gold_signup_date from sales as a
left join goldusers_signup as b on a.userid = b.userid and gold_signup_date <= created_date) c) e;
```
