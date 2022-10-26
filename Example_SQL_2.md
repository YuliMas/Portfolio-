## Кейс "Баланс учеников в онлайн-школе английского языка"


**Описание кейса.**

**Задача** — смоделировать изменение балансов студентов. Баланс — это количество уроков, которое есть у каждого студента. 
Чтобы проверить, всё ли в порядке с данными, составить список гипотез и вопросов, необходимо понимать: 
- сколько всего уроков было на балансе всех учеников за каждый календарный день;
- как это количество менялось под влиянием транзакций (оплат, начислений, корректирующих списаний) и уроков (списаний с баланса по мере прохождения уроков).
Также необходимо создать таблицу, где будут балансы каждого студента за каждый день.

В результате должен получиться запрос, который собирает данные о балансах студентов за каждый прожитый ими день.

- **Описание базы данных**
    
    **SKYENG_DB**
    
    **classes**
    
    ***Витрина с уроками***
    
    - **user_id** - уникальный идентификатор юзера
    - **id_class** - уникальный идентификатор урока
    - **class_start_datetime** - время начала урока
    - **class_end_datetime** - время конца урока
    - **class_removed_datetime** - время удаления записи о данном уроке
    - **id_teacher** - уникальный идентификатор учителя
    - **class_status** - статус урока (успешно проведен / отменен и тд)
    - **class_status_datetime  -** время проставления статуса по уроку
    
    **payments**
    
    ***Витрина с платежами по урокам***
    
    - **user_id** - уникальный идентификатор юзера
    - **id_transaction** - уникальный идентификатор транзакции
    - **operation_name** - название проведенной операции
    - **status_name** - статус проведенной операции (исполнена / не исполнена и тд)
    - **classes** - количество оплаченных уроков
    - **payment_amount** - выплаченная сумма
    - **transaction_datetime** - время проведения операции
    
    **students**
    
    ***Витрина со списком студентов***
    
    - **user_id** - уникальный идентификатор юзера
    - **student_sex** - пол юзера
    - **geo_cluster** - географическая агрегация
    - **country_name** - короткое название страны
    - **region_name** - название региона
    - **email_domain** - домен электронной почты
    
    **teachers**
    
    ***Витрина со списком учителей***
    
    - **id_teacher** - уникальный идентификатор учителя
    - **age** - возраст
    - **city** - город проживания учителя
    - **department** - направление, в котором работает учитель
    - **max_teaching_level** - название уровня языка у преподавателя
    - **id_teaching_level** - уникальный идентификатор уровня языка у преподавателя
    - **language_group** - основной язык преподавателя


``` sql

    with first_payments as (select user_id
            , min (transaction_datetime::date) as first_transaction_date
        from skyeng_db.payments
        where status_name = 'success'
        and id_transaction is not null
        group by 1
        order by 1),
    all_dates as (select date_trunc ('day', class_start_datetime) as dt
        from skyeng_db.classes
        where class_start_datetime between '2016-01-01' and '2017-01-01'
            and class_status = 'success'
        group by 1
        order by 1),
    payments_by_dates as (select user_id
            , date_trunc ('day', transaction_datetime) as payment_date
            , sum (classes) as transaction_balance_change
        from skyeng_db.payments
        where status_name = 'success'
        and id_transaction is not null
        group by 1, 2
        order by 1, 2),
    all_dates_by_user as (select b.user_id
            , a.dt
        from all_dates as a
            left join first_payments as b
                on a.dt >= b.first_transaction_date
        order by 1, 2),
    classes_by_dates as (select user_id
            , date_trunc ('day', class_start_datetime) as class_date
            , count (id_class) * -1 as classes
        from skyeng_db.classes
        where class_status in ('success', 'failed_by_student')
            and class_type != 'trial'
        group by 1, 2
        order by 1, 2),
    payments_by_dates_cumsum as (select c.user_id
            , c.dt
            , coalesce (d.transaction_balance_change, 0) as transaction_balance_change
            , sum (coalesce (d.transaction_balance_change, 0)) over (partition by c.user_id order by c.dt rows between unbounded preceding and current row) as transaction_balance_change_cs
        from all_dates_by_user as c
          left join payments_by_dates as d 
                on c.user_id = d.user_id and c.dt = d.payment_date
        order by 1, 2),
     classes_by_dates_dates_cumsum as (select e.user_id
            , e.dt
            , coalesce (f.classes, 0) as classes
            , sum (coalesce (f.classes, 0)) over (partition by e.user_id order by e.dt rows between unbounded preceding and current row) as classes_cs
        from all_dates_by_user as e
            left join classes_by_dates as f
                 on e.user_id = f.user_id and e.dt = f.class_date
        order by 1, 2),
    balances as (select g.user_id
            , g.dt
            , g.transaction_balance_change
            , g.transaction_balance_change_cs
            , h.classes 
            , h.classes_cs
            , h.classes_cs + g.transaction_balance_change_cs as balance
        from payments_by_dates_cumsum as g
         inner join classes_by_dates_dates_cumsum as h
            on g.user_id = h.user_id and g.dt = h.dt)
    select dt
        , sum (transaction_balance_change) as sum_transaction_balance_change
        , sum (transaction_balance_change_cs) as sum_transaction_balance_change_cs
        , sum (classes) as sum_classes
        , sum (classes_cs) as sum_classes_cs
        , sum (balance) as sum_balance
    from balances
    group by 1
    order by 1

```
