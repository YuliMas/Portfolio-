
### Задание 1.
Код для создания тестовой таблицы и заполнения ее данными:

``` sql
create table PDCL 
(date date, customer int, deal int, currency varchar, sum int);

insert into PDCL 
(date, customer, deal, currency, sum)
values
('2009-12-12', 111110, 111111, 'RUR', 12000),
('2009-12-25', 111110, 111111, 'RUR', 5000),
('2009-12-12', 111110, 122222, 'RUR', 10000),
('2010-01-12', 111110, 111111, 'RUR', -10100),
('2009-11-20', 220000, 222221, 'RUR', 25000),
('2009-12-20', 220000, 222221, 'RUR', 20000),
('2009-12-31', 220001, 222221, 'RUR', -10000),
('2009-12-29', 111110, 122222, 'RUR', -10000),
('2009-11-27', 220001, 222221, 'RUR', -30000)
```

Код запроса и результат:
``` sql
with table_1 as (select *, 
                    sum(sum) over (partition by deal) as all_sum, -- общая сумма долга на настоящий момент
                    sum(sum) over (partition by deal order by date) as cum_sum -- кумулятивный долг, накопившийся на момент текущей операции
                from PDCL),
table_2 as (select *,
                case when cum_sum <= 0 then 0 else 1 end as flag -- 0 или 1 в зависимости от того, чему равна cum_sum на счете
            from table_1),                                      
table_3 as (select *,
                row_number() over (partition by deal, flag order by date) as range_number  -- ранг операции внутри окна deal + flag                      
            from table_2),                                                               
table_4 as (select *,                                                                                                                                        
                max(date) over (partition by deal, range_number) as date_of_start, -- наибольшее значение даты в окне "номер кредита + ранг" 
                lead(date) over (partition by deal order by date) as next_date     -- дата следующей операции для каждой даты
            from table_3)                                                       
select deal,
        all_sum,
        max(case when flag = 0 then next_date else date_of_start end) as start_of_delay, -- дата начала текущей просрочки 
        current_date - max(case when flag = 0 then next_date else date_of_start end) as cnt_delay_days -- количество дней текущей просрочки
from table_4                                                                                           
where all_sum > 0 and range_number = 1
group by 1, 2


```

![N|Solid](https://i.ibb.co/yfpLbZW/2022-12-06-172141.jpg)

Логика решения.
1. Получаем из нашего датасета общую сумму по полю deal (колонка all_sum, отражает наличие задолженности по кредиту на текущий момент) и кумулятивную сумму по каждой конкретной записи (cum_sum, позволяет отследить, когда значение просрочки стало равным 0, т.е. период просрочки закончился).
2. Создаем колонку flag, значения в которой зависят от того, погасил ли клиент просрочку или нет: если значение столбца cum_sum <= 0, проставляем 0, если значение больше 0, проставляем 1.
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
            date_of_sale - interval '1 day' as dow_6,
            date_of_sale - interval '2 day' as dow_5,
            date_of_sale - interval '3 day' as dow_4,
            date_of_sale - interval '4 day' as dow_3,
            date_of_sale - interval '5 day' as dow_2,
            date_of_sale - interval '6 day' as dow_1,
            payment_sum/5 as avg_per_day
            from sales)
    select gs.date_of_sale, coalesce (avg_per_day, 0) as avg_per_day,
            extract (month from gs.date_of_sale) as month_of_sale,
            extract (dow from gs.date_of_sale) as number_of_day
    from generate_series('2013-02-20'::date, '2013-06-18'::date ,'1 day') as gs(date_of_sale) 
        left join days_of_week
            on  gs.date_of_sale = days_of_week.date_of_sale 
                or gs.date_of_sale = days_of_week.dow_1 
                or gs.date_of_sale = days_of_week.dow_2 
                or gs.date_of_sale = days_of_week.dow_3 
                or gs.date_of_sale = days_of_week.dow_4 
                or gs.date_of_sale = days_of_week.dow_5
                or gs.date_of_sale = days_of_week.dow_6
    where extract (dow from gs.date_of_sale) != 6 and extract (dow from gs.date_of_sale) !=0) as table_2
group by 1
order by 1

```
![N|Solid](https://i.ibb.co/PY0tqbj/2022-12-06-180853.jpg)

Код процедуры, позволяющей автоматически получать данные (номер месяца, сумму) по введенному параметру MonthNumber (номер месяца):
 ``` sql
CREATE PROCEDURE week_to_month (MonthNumber integer)
LANGUAGE SQL
AS $$
select month_of_sale,
        sum (avg_per_day) as payments_sum
from
    (with days_of_week as (select *, 
            date_of_sale - interval '1 day' as dow_6,
            date_of_sale - interval '2 day' as dow_5,
            date_of_sale - interval '3 day' as dow_4,
            date_of_sale - interval '4 day' as dow_3,
            date_of_sale - interval '5 day' as dow_2,
            date_of_sale - interval '6 day' as dow_1,
            payment_sum/5 as avg_per_day
            from sales)
    select gs.date_of_sale, coalesce (avg_per_day, 0) as avg_per_day,
            extract (month from gs.date_of_sale) as month_of_sale,
            extract (dow from gs.date_of_sale) as number_of_day
    from generate_series('2013-02-20'::date, '2013-06-18'::date ,'1 day') as gs(date_of_sale) 
        left join days_of_week
            on  gs.date_of_sale = days_of_week.date_of_sale 
                or gs.date_of_sale = days_of_week.dow_1 
                or gs.date_of_sale = days_of_week.dow_2 
                or gs.date_of_sale = days_of_week.dow_3 
                or gs.date_of_sale = days_of_week.dow_4 
                or gs.date_of_sale = days_of_week.dow_5
                or gs.date_of_sale = days_of_week.dow_6
    where extract (dow from gs.date_of_sale) != 6 and extract (dow from gs.date_of_sale) !=0) as table_2
where month_of_sale = MonthNumber
group by 1
order by 1;
$$;

CALL week_to_month (2);

 ```