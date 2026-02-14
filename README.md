# Домашнее задание к занятию "`Индексы`" - `Рыбянцев Павел`

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.
```sql
SELECT 
    concat(ROUND(SUM(data_length)  / 1024 / 1024, 2), ' MB') "data_size",
    concat(ROUND(SUM(index_length) / 1024 / 1024, 2), ' MB')  "index_size",
    concat(ROUND(SUM(index_length) / 1024 / 1024, 2) + ROUND(SUM(data_length)  / 1024 / 1024, 2), ' MB') "total_size",
    concat(ROUND((SUM(index_length) / SUM(data_length + index_length)) * 100, 2), ' %') "index_percentage"
FROM 
    information_schema.TABLES
WHERE 
    table_schema = 'sakila';
```
```
data_size|index_size|total_size|index_percentage|
---------+----------+----------+----------------+
4.17 MB  |2.28 MB   |6.45 MB   |35.35 %         |
```

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
```
Декартово произведение (Cartesian Product): 
    В плане это Inner hash join (no condition). 
    Таблица film (f) никак не связана с остальными таблицами в WHERE.
    В итоге 634 строки из payment перемножились на 1000 строк из film, превратившись в 634 000 строк в памяти.

Таблица inventory (i) присутствует в FROM, 
но её данные не используются и не влияют на фильтрацию (так как rental и так ссылается на inventory_id).

Из за использования date() в условии и отсутсвия индекса на p.payment_date выполняется полный скан таблицы (Table scan on p).

Избыточность оконной функции и DISTINCT:
    SUM() OVER и тут же DISTINCT. 
    Это заставляет СУБД сначала рассчитать сумму для каждой из 642 тысяч строк, 
    а потом дедуплицировать их. Обычная GROUP BY сделает это в разы быстрее.


```
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.
```
добавляем индекс на p.payment_date, так как это единственное условие по которому происходит фильтарция в WHERE
```
```sql
CREATE INDEX idx_payment_date ON payment(payment_date);
```
```
судя по запросу от него требуется отразить суммы платежей в разрезе клиентов за конкретную дату.
1. в таком случае убираем избыточные таблицы из запроса  inventory i, film f, rental r;
2. Переписываем связи таблиц на join, используя для связи customer_id;
3. Используем группировку вместо оконной функции;
4. В приложении where меняем функцию date() на условие:
    p.payment_date >= '2005-07-30' AND p.payment_date < '2005-07-31' 
    чтобы не сканировать всю таблицу а зпдействовать созданный индекс;
5. Убираем distinct так как он нам больше не нужен из за отсутсвия дублей.
```
```sql
select 
	concat(c.last_name, ' ', c.first_name) "customer_name",
	sum(p.amount) "total_amount"
from payment p
join customer c on p.customer_id = c.customer_id 
	where 
	p.payment_date >= '2005-07-30' AND p.payment_date < '2005-07-31'
group by 1;
```
```
Итого удалось сократить время выполнения запроса с 4685 мс (4.6 сек) до 1.95 мс (в 2403 раза быстрее)
В 1012 раз меньше обработано строк чем было (634 против 642 000)
```


## Дополнительные задания (со звёздочкой*)
Эти задания дополнительные, то есть не обязательные к выполнению, и никак не повлияют на получение вами зачёта по этому домашнему заданию. Вы можете их выполнить, если хотите глубже шире разобраться в материале.

### Задание 3*

Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

*Приведите ответ в свободной форме.*