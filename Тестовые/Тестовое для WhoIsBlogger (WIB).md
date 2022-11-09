# Тестовое задание_1
1) Код для создания тестируемой базы данных: 
``` sql 
create table users
(user_id int, age int)

insert into users 
(user_id, age)
values (1, 27),
(2, 35),
(3, 61),
(4, 45),
(5, 33),
(6, 18),
(7, 20),
(8, 37),
(9, 53),
(10, 25)
---
create table purchases
(purchase_id int, user_id int, item_id int, date date)

insert into purchases
(purchase_id, user_id, item_id, date)
values (1, 5, 6, '2020-05-03'),
(2, 3, 6, '2020-07-03'),
(3, 1, 1, '2020-05-07'),
(4, 7, 2, '2021-01-29'),
(5, 6, 1, '2021-04-03'),
(6, 5, 9, '2021-05-29'),
(7, 4, 10, '2021-12-15'),
(8, 9, 4, '2022-03-09'),
(9, 2, 3, '2022-02-21'),
(10, 8, 7, '2022-07-15'),
(11, 3, 4, '2022-09-01'),
(12, 6, 8, '2022-04-22'),
(13, 10, 5, '2022-10-08')
---
create table items
(item_id int, price int)

insert into items
(item_id, price)
values (1, 100),
(2, 450),
(3, 670),
(4, 325),
(5, 210),
(6, 432),
(7, 180),
(8, 95),
(9, 370),
(10, 215)
``` 
2) Запросы для вычисления требуемых метрик.
А) 
- какую сумму в среднем в месяц тратят пользователи в возрастном диапазоне от 18 до 25 лет включительно:
``` sql
select count (distinct user_id) as cnt_users
    , count (distinct date_trunc ('month', date)) as cnt_month
    , sum (price) as all_sum
    , sum (price)::float / count (distinct user_id) / count (distinct date_trunc ('month', date)) as avg_payment_on_user_per_month
from
    (select u.user_id
        , u.age
        , p.purchase_id
        , p.item_id
        , p.date
        , i.price
    from users as u 
        inner join purchases as p
            on u.user_id = p.user_id and u.age between 18 and 25
        inner join items as i
            on p.item_id = i.item_id
    ) as t
``` 
- какую сумму в среднем в месяц тратят пользователи в возрастном диапазоне от 26 до 35 лет включительно:
``` sql
select count (distinct user_id) as cnt_users
    , count (distinct date_trunc ('month', date)) as cnt_month
    , sum (price) as all_sum
    , sum (price)::float / count (distinct user_id) / count (distinct date_trunc ('month', date)) as avg_payment_on_user_per_month
from
    (select u.user_id
        , u.age
        , p.purchase_id
        , p.item_id
        , p.date
        , i.price
    from users as u 
        inner join purchases as p
            on u.user_id = p.user_id and u.age between 26 and 35
        inner join items as i
            on p.item_id = i.item_id
    ) as t
``` 

Б) в каком месяце года выручка от пользователей в возрастном диапазоне 35+ самая большая:
Если нужно выяснить, какой месяц был наиболее прибыльным по выручке за все время:
``` sql
select date_trunc ('month', p.date) as month_of_purchase
    , sum (i.price) as all_sum
from users as u 
    inner join purchases as p
        on u.user_id = p.user_id and u.age >= 35
    inner join items as i
        on p.item_id = i.item_id
group by 1
order by 2 desc
limit 1
``` 
Можно также вывести лучший по выручке месяц за каждый конкретный год:
``` sql
select month_of_purchase
    , all_sum
from
    (select *
        , max (all_sum) over (partition by date_trunc('year', month_of_purchase)) as max_purchases_in_year
    from
        (select date_trunc ('month', p.date) as month_of_purchase
            , sum (i.price) as all_sum
        from users as u 
            inner join purchases as p
                on u.user_id = p.user_id and u.age >= 35
            inner join items as i
                on p.item_id = i.item_id
        group by 1
        ) as t
    ) as d
where all_sum = max_purchases_in_year
``` 
В) какой товар обеспечивает наибольший вклад в выручку за последний год (в случае нашей тестовой базы - за 2022 год, но нужно учитывать, что он еще не закончился и данные не совсем полные):
``` sql
select item_id
    , cnt_purchases * price as sum_purchases
from
    (select i.item_id
        , i.price
        , count (p.purchase_id) as cnt_purchases
    from purchases as p
        inner join items as i
            on p.item_id = i.item_id
    where date_trunc ('year', p.date) = '2022-01-01'
    group by 1, 2
    ) as t
order by 2 desc
limit 1
``` 
Г) топ-3 товаров по выручке и их доля в общей выручке за любой год.
Если имеется в виду топ-3 товаров за любой, но конкретный год из нашей базы (к примеру, за 2021):
``` sql
with items_sold as (select i.item_id
                            , i.price
                            , count (p.purchase_id) as cnt_purchases
                    from purchases as p
                            inner join items as i
                                on p.item_id = i.item_id
                    where date_trunc ('year', p.date) = '2021-01-01'
                    group by 1, 2)
select item_id
    , cnt_purchases * price as sum_purchases
    , sum (cnt_purchases * price) over () as all_sum_purchases
    , cnt_purchases * price::float / sum (cnt_purchases * price) over () * 100 as percentage
from items_sold
order by 2 desc
limit 3
``` 
Если имеется в виду топ-3 товаров вообще за все время (независимо от года):
``` sql
with items_sold as (select i.item_id
                            , i.price
                            , count (p.purchase_id) as cnt_purchases
                    from purchases as p
                            inner join items as i
                                on p.item_id = i.item_id
                    group by 1, 2
			)
select item_id
    , cnt_purchases * price as sum_purchases
    , sum (cnt_purchases * price) over () as all_sum_purchases
    , cnt_purchases * price::float / sum (cnt_purchases * price) over () * 100 as percentage
from items_sold
order by 2 desc
limit 3
``` 









