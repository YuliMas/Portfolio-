
### Задание 1.
Код для создания тестовой таблицы и заполнения ее данными:

``` sql
create table PDCL 
(date date, customer int, deal int, currency varchar, sum int);

insert into PDCL 
(date, customer, deal, currency, sum)
values ('2021-10-05', 1, 1, 'RUR', 2500),
('2021-08-05', 2, 1, 'RUR', -1500),
('2021-05-03', 3, 2, 'RUR', 7500),
('2021-05-07', 3, 2, 'RUR', -3500),
('2021-05-13', 3, 2, 'RUR', -2000),
('2021-10-21', 1, 1, 'RUR', -1000),
('2021-10-25', 1, 1, 'RUR', 3600),
('2021-06-07', 3, 2, 'RUR', 2400),
('2021-10-07', 4, 3, 'RUR', 1300),
('2021-10-13', 4, 3, 'RUR', -500),
('2021-10-19', 4, 3, 'RUR', -800),
('2021-10-23', 4, 3, 'RUR', 2100),
('2021-10-05', 5, 4, 'RUR', 2800),
('2021-10-09', 5, 4, 'RUR', -2800)
```

Код запроса и результат:
``` sql
with table_1 as (select *, 
                    sum(sum) over (partition by deal) as all_sum, -- Общая сумма долга на настоящий момент
                    sum(sum) over (partition by deal order by date) as cum_sum -- Кумулятивный долг, накопившийся на момент текущей операции
                from PDCL),
table_2 as (select *,
                case when cum_sum = 0 then 0 else 1 end as flag -- Этот столбец задает значения 0 и 1 в зависимости от того, какая сумма долга накопилась 
                                                                -- на счету:
            from table_1),                                      -- если 0 - то 0, если больше 0 - то 1
table_3 as (select *,
                row_number() over (partition by deal, flag order by date) as range_number -- Отранжируем наши записи (операции) внутри группы "номер кредита 
                                                                                          -- + значение flag": таким образом, каждая новая запись с 
            from table_2),                                                                -- обнулением счета по данному кредиту (т.е. закрытием просрочки) 
table_4 as (select *,                                                                     -- будет получать ранг 1. Такой же ранг получит самая первая запись 
	                                                                                      -- с появлением положительного числа на счете.
                max(date) over (partition by deal, range_number) as date_of_start, -- Выберем наибольшее значение даты в окне "номер кредита + ранг". 
                                                                                   -- Это позволит получить самую позднюю дату для всех записей с рангом 1.
                lead(date) over (partition by deal order by date) as next_date     -- Подтянем к каждой из имеющихся в таблице дат дату следующей операции.
            from table_3)                                                       
select deal,
        all_sum,
        max(case when flag = 0 then next_date else date_of_start end) as start_of_delay,             -- Зададим условие: в случае если значение колонки flag 
                                                                                                     -- равно 0 (т.е. просрочка закончилась), выведем значение
        current_date - max(case when flag = 0 then next_date else date_of_start end) as cnt_delay_days --следующей за ней даты; если значение равно 1 (т.е. 
                                                                                                       -- сумма просрочки) положительная и не обнулялась,
from table_4                                                                                           -- тогда берем дату, лежащую в столбце date_of_start
where all_sum > 0 and range_number = 1
group by 1, 2


```

![N|Solid](https://i.ibb.co/DK2MpcL/1.jpg)

Логика решения.
1. Получаем из нашего датасета общую сумму по полю deal (колонка all_sum, отражает наличие задолженности по кредиту на текущий момент) и кумулятивную сумму по каждой конкретной записи (cum_sum, позволяет отследить, когда значение просрочки стало равным 0, т.е. период просрочки закончился).
2. Создаем колонку flag, значения в которой зависят от того, погасил ли клиент просрочку или нет: если значение столбца cum_sum = 0, проставляем 0, если значение больше 0, проставляем 1.
3. Задаем ранги каждой операции по окну "номер кредита и значение flag". Ранг 1 получает либо каждая новая запись, в которой значение flag (и cum_sum) = 0, либо самая первая запись с появлением положительной суммы долга на счету (в случае, если просрочка по кредиту ни разу не была погашена, т.е. значения вышеуказанных колонок не были равными 0).
4. Отбираем наибольшую дату для окна "номер кредита и значение flag" (столбец date_of_start). Это позволит получить самую позднюю дату для всех записей с рангом 1.
5. Задаем условие: в случае если значение колонки flag равно 0 (т.е. просрочка закончилась), выведем значение следующей за ней даты (т.е. даты, когда клиент провалился в новую просрочку). Если значение flag равно 1 (т.е. сумма просрочки оставалась положительной и не обнулялась), выведем дату из столбца date_of_start.
6. Задаем нужные нам условия и получаем необходимые данные.


### Задание 2

Код для создания тестовой таблицы и заполнения ее данными:

``` sql
create table sales 
(date_of_sale date, payment_sum float);

insert into sales
(date_of_sale, payment_sum)
values ('2013-02-26', 312.00),
('2013-03-05', 833.00),
('2013-03-12', 225.00),
('2013-03-19', 453.00),
('2013-03-26', 774.00),
('2013-04-02', 719.00),
('2013-04-09', 136.00),
('2013-04-16', 133.00),
('2013-04-23', 157.00),
('2013-04-30', 850.00),
('2013-05-07', 940.00),
('2013-05-14', 933.00),
('2013-05-21', 422.00),
('2013-05-28', 952.00),
('2013-06-04', 136.00),
('2013-06-11', 701.00)

```

Код обычного запроса и результат (таблица с номерами месяцев и суммой продаж):
``` sql
select month_of_sale,
        sum (avg_per_day) as payments_sum
from
    (with days_of_week as (select *, 
            date_of_sale + interval '1 day' as dow_1,
            date_of_sale + interval '2 day' as dow_2,
            date_of_sale + interval '3 day' as dow_3,
            date_of_sale + interval '4 day' as dow_4,
            date_of_sale + interval '5 day' as dow_5,
            date_of_sale + interval '6 day' as dow_6,
            date_of_sale + interval '7 day' as dow_7,
            payment_sum/5 as avg_per_day
            from sales)
    select gs.date_of_sale, coalesce (avg_per_day, 0) as avg_per_day,
            extract (month from gs.date_of_sale) as month_of_sale,
            extract (dow from gs.date_of_sale) as number_of_day
	    from generate_series('2013-02-26'::date, '2013-06-18'::date ,'1 day') as gs(date_of_sale) 
	        left join days_of_week
	            on gs.date_of_sale = days_of_week.dow_1 
	                or gs.date_of_sale = days_of_week.dow_2 
	                or gs.date_of_sale = days_of_week.dow_3 
	                or gs.date_of_sale = days_of_week.dow_4 
	                or gs.date_of_sale = days_of_week.dow_5
	                or gs.date_of_sale = days_of_week.dow_6
	                or gs.date_of_sale = days_of_week.dow_7 
    where extract (dow from gs.date_of_sale) != 6 and extract (dow from gs.date_of_sale) !=0
   	) as table_2
group by 1
order by 1


```
![N|Solid](https://i.ibb.co/25sbtn7/2.jpg)

Код процедуры, позволяющей автоматически получать данные (номер месяца, сумму) по введенному параметру MonthNumber (номер месяца):
 ``` sql
CREATE PROCEDURE week_to_month (MonthNumber integer)
LANGUAGE SQL
AS $$
select month_of_sale,
        sum (avg_per_day) as payments_sum
from
    (with days_of_week as (select *, 
                            date_of_sale + interval '1 day' as dow_1,
                            date_of_sale + interval '2 day' as dow_2,
                            date_of_sale + interval '3 day' as dow_3,
                            date_of_sale + interval '4 day' as dow_4,
                            date_of_sale + interval '5 day' as dow_5,
                            date_of_sale + interval '6 day' as dow_6,
                            date_of_sale + interval '7 day' as dow_7,
                            payment_sum/5 as avg_per_day
                          from sales
                          )
      select gs.date_of_sale, coalesce (avg_per_day, 0) as avg_per_day,
              extract (month from gs.date_of_sale) as month_of_sale,
              extract (dow from gs.date_of_sale) as number_of_day
	      from generate_series('2013-02-26'::date, '2013-06-18'::date ,'1 day') as gs(date_of_sale) 
	          left join days_of_week
	              on gs.date_of_sale = days_of_week.dow_1 
	                  or gs.date_of_sale = days_of_week.dow_2 
	                  or gs.date_of_sale = days_of_week.dow_3 
	                  or gs.date_of_sale = days_of_week.dow_4 
	                  or gs.date_of_sale = days_of_week.dow_5
	                  or gs.date_of_sale = days_of_week.dow_6
	                  or gs.date_of_sale = days_of_week.dow_7 
	      where extract (dow from gs.date_of_sale) != 6 and extract (dow from gs.date_of_sale) !=0
	 ) as table_2
where month_of_sale = MonthNumber
group by 1
order by 1;
$$;

CALL week_to_month (2);

 ```