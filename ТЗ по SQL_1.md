
![](https://skyengpublic.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F5ee8499a-9e82-49f0-b8bd-0befe6209e06%2FUntitled.png?table=block&id=9cb25e91-38dc-4852-a2c1-82ff18f8e0cc&spaceId=0771f0bb-b4cb-4a14-bc05-94cbd33fc70d&width=2000&userId=&cache=v2)



### Задание 1
Получите список всех сотрудников (_таблица Employees_), отсортированную по алфавиту.
**Что нужно вывести:** все поля из таблицы employees.

``` sql
select *
from employees
order by last_nm asc
```
### Задание 2

Получите список сотрудников с названиями их позиций (_position_nm_), актуальных на 10.10.2020. **Что нужно вывести:** last_nm, first_nm, middle_nm, position_nm

``` sql
select emp.last_nm
	, emp.first_nm
	, emp.middle_nm
	, jobh.position_nm
from employees as emp
	inner join job_history as jobh
		on emp.position_id = jobh.position_id
where jobh.valid_from <= '2020-10-10'
	and jobh.valid_to >='2020-10-10'
```
Комментарий к решению: здесь необходимо отразить те позиции (position_nm), которые были актуальны на 10.10.2020.  Вероятно, информация об этом содержится в колонках valid_from и valid_to в таблице job_history: в первой из них, вероятно,  располагается информация о том, с какой даты актуальна та или иная позиция, во второй - до какой даты она действует (открыта). Таким образом, в моем коде отфильтрованы те позиции, которые начали действовать до 10.10.2020 (или в этот же день) и одновременно прекращают свое действие 10.10.2020 или после. Тем самым отсекаются те позиции, которые были открыты после 10.10.2020 или были закрыты до 10.10.2020, то есть неактуальные на эту дату.

### Задание 3

Получите список сотрудников, у которых ЗП (salary) от 20 000 до 50 000, актуальных на текущий момент
**Что нужно вывести:** last_nm, first_nm, middle_nm, salary, hire_dt
``` sql
select emp.last_nm
	, emp.first_nm
	, emp.middle_nm
	, jobh.salary
	, emp.hire_dt
from employees as emp
	inner join job_history as jobh
		on emp.position_id = jobh.position_id
where jobh.salary between 20000 and 50000
	and jobh.valid_to > = current_date
	and jobh.valid_from <= current_date
``` 
Комментарий к решению: здесь я руководствовалась той же логикой, что и в предыдущем задании, исходя из предположения, что нужная нам информация хранится в колонках valid_from и valid_to. Можно было бы предположить, что нужная нам информация хранится в столбце hire_dt из таблицы employees,
однако здесь, вероятно, записана дата, в которую сотрудник устроился на работу; при этом столбца, который бы содержал информацию о том, когда сотрудник ушел, нет. Таким образом, из данных в таблице employees невозможно сделать вывод, работает ли сотрудник на текущую дату или уже нет, в связи с чем я использовала для запроса колонки valid_from и valid_to из таблицы job_history.

### Задание 4
Получите список сотрудников в формате: «_Иванова - Наталья – Юрьевна»_ (ФИО должно быть прописано в одном столбике, разделение ‘-’)
**Что нужно вывести:** (новое поле, назовем его fio), birth_dt
``` sql
select concat (last_nm, ' - ', first_nm, ' - ', middle_nm) as fio
	, birth_dt
from employees
```

### Задание 5

Получите количество звонков (_таблица Calls_) по дням, период с 01.10.2020 по текущий день
**Что нужно вывести:** date (дата звонка), cnt_calls (количество звонков)
``` sql
select date_trunc('day', start_dttm)::date as date
	, count (call_id) as cnt_calls
from calls
where start_dttm::date between '2020-10-01' and current_date
group by 1
``` 

### Задание 6

Вывести %% дозвона для каждого дня _(%% дозвона – это доля принятых звонков (dozv_flg = 1) от всех поступивших звонков (dozv_flg =1 or dozv_flg = 0_)), период с 01.10.2020 по текущий день
**Что нужно вывести:** date, sla (%% дозвона)
``` sql
select date
	, dosv_cnt / all_calls_cnt * 100 as sla
from
	(select start_dttm::date as date
		, count(call_id) as all_calls_cnt
		, count (case when dozv_flg = 1 then call_id end) as dosv_cnt
	from calls
	where start_dttm::date between '2020-10-01' and current_date
	group by 1
	) c
``` 
### Задание 7

Получите длительность самого длинного разговора, и самого короткого за 28.10.2020 (start_dttm – начало звонка, end_dttm - окончание звонка)
**Что нужно вывести:** max_call_time, min_call_time
``` sql
select max(call_time) as max_call_time
	, min(call_time) as min_call_time
from
	(select end_dttm - start_dttm as call_time 
	from calls 
	where start_dttm::date = '2020-10-28'
	) c
``` 

### Задание 8

Получите информацию по всем звонкам, в какой системе они были обработаны (_таблица System_)
**Что нужно вывести:** **call_id, start_dttm, end_dttm, system_name
``` sql
select calls.call_id
	, calls.start_dttm::date
	, calls.end_dttm::date
	, system.system_name
from calls 
	inner join system 
		on calls.system_id = system.system_id
``` 		

### Задание 9

Получите информацию по клиентам (_таблица Customers_), которые звонили и не дозвонились 23.10.2020.
**Что нужно вывести:** date (дата звонка), last_nm, first_nm, middle_nm
``` sql
select calls.start_dttm::date as date
	, distinct customers.last_nm
	, customers.first_nm
	, customers.middle_nm
from customers 
	inner join calls 	
		on customers.customer_id = calls.customer_id
		and calls.start_dttm::date = '2020-10-23'
		and calls.dozv_flg = 0
```
### Задание 10

Получите информацию по продуктам, которые были открыты у клиентов во время звонка с 05.10.2020 по текущий день
**Что нужно вывести:** last_nm, first_nm, middle_nm, date (дата звонка), open_dt, product_nm
``` sql
select customers.last_nm
	, customers.first_nm
	, customers.middle_nm
	, calls.start_dttm::date as date 
	, customers.open_dt
	, customers.product_nm
from customers 
	inner join calls 
		on customers.customer_id = calls.customer_id
		and calls.start_dttm::date between '2020-10-05' and current_date
		and open_dt::date between '2020-10-05' and current_date
```

### Задание 11

С помощью оконных функций выведите информацию по тем звонкам, на которых данный employee_id уже не первый раз общается с данным customer_id
**Что нужно вывести:** customer_id, employee_id, start_dttm, end_dttm
``` sql
select customer_id
	, employee_id
	, start_dttm
	, end_dttm
from
	(select customer_id
		, employee_id
		, start_dttm::date
		, end_dttm::date
		, min(start_dttm) over (partition by customer_id, employee_id)::date as first_dttm
	from calls
	) t
where start_dttm > first_dttm 
```


