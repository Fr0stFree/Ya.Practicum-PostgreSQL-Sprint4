## Задание

Клиенты сервиса в свой день рождения получают скидку. Расчёт скидки и отправка клиентам промокодов происходит на стороне сервера приложения. Список клиентов возвращается из БД в приложение таким запросом:
```sql
SELECT
    user_id::text::uuid,
    first_name::text,
    last_name::text, 
    city_id::bigint,
    gender::text
FROM users
WHERE city_id::integer = 4
    AND date_part('day', to_date(birth_date::text, 'yyyy-mm-dd')) = date_part('day', to_date('31-12-2023', 'dd-mm-yyyy'))
    AND date_part('month', to_date(birth_date::text, 'yyyy-mm-dd')) = date_part('month', to_date('31-12-2023', 'dd-mm-yyyy'))
```
Каждый раз список именинников формируется и возвращается недостаточно быстро. Оптимизируйте этот процесс.

## Решение

### План запроса

```text
Seq Scan on public.users  (cost=0.00..2843.22 rows=1 width=120) (actual time=4.536..13.635 rows=5 loops=1)
  Output: ((user_id)::text)::uuid, (first_name)::text, (last_name)::text, city_id, (gender)::text
  Filter: (((users.city_id)::integer = 4) AND (date_part('day'::text, (to_date((users.birth_date)::text, 'yyyy-mm-dd'::text))::timestamp without time zone) = date_part('day'::text, (to_date('31-12-2023'::text, 'dd-mm-yyyy'::text))::timestamp without time zone)) AND (date_part('month'::text, (to_date((users.birth_date)::text, 'yyyy-mm-dd'::text))::timestamp without time zone) = date_part('month'::text, (to_date('31-12-2023'::text, 'dd-mm-yyyy'::text))::timestamp without time zone)))
  Rows Removed by Filter: 9217
Query Identifier: -5932910545661619613
Planning Time: 0.135 ms
Execution Time: 13.658 ms
```

### Аналитика

1. При анализе таблицы `users` было замечено, что некоторые столбцы имеет тип CHAR(500), которая всегда занимает 500 символов, дополняя строку пробелами. Это неоптимальная структура данных для хранения текстовой информации используется для полей `gender`, `birth_date`, `user_id`, `first_name`, `last_name`, `email`.
2. В запросе каждый раз явным образом приводятся типы данных к необходимым (`user_id` -> `UUID`, `birth_date` -> `DATE` ...), что создает накладные расходы для СУБД.
3. Гипотеза: в таблице имеется индекс `users_pkey`, но в запросе он не используется, вероятно, из-за преобразования значения к `INTEGER`.
4. Для фильтрации по дате используются функции, избыточно нагружающие процессор.

### Предложения

Необходимо привести типы колонок таблицы к наиболее оптимальным
	- user_id необходимо хранить в формате UUID, который занимает гораздо меньше места
	- first_name и last_name можно привести к типу VARCHAR(50), поскольку вряд ли встретится имя или фамилии длиннее данного значения
	- для city_id можно использовать SMALLINT вместо BIGINT
	- для gender можно использовать ENUM или CHAR(1)
	- registration_date и birth_date необходимо привести к DATE

```sql
ALTER TABLE users 
	ALTER COLUMN user_id TYPE UUID USING user_id::TEXT::UUID,
	ALTER COLUMN first_name TYPE VARCHAR(50) USING first_name::VARCHAR(50),
	ALTER COLUMN last_name TYPE VARCHAR(50) USING first_name::VARCHAR(50),
	ALTER COLUMN city_id TYPE SMALLINT,
	ALTER COLUMN gender TYPE CHAR(1)
		USING (
		    CASE 
		        WHEN gender = 'male' THEN 'M'
		        WHEN gender = 'female' THEN 'F'
		    END
		),
	ALTER COLUMN birth_date TYPE DATE USING TO_DATE(birth_date::TEXT, 'yyyy-mm-dd'),
	ALTER COLUMN registration_date TYPE DATE USING TO_DATE(birth_date::TEXT, 'yyyy-mm-dd');
```

Теперь из запроса можно убрать избыточные преобразования, а фильтрацию по дате упростить
```sql
SELECT
    user_id,
    first_name,
    last_name,
    city_id,
    gender
FROM users
WHERE city_id = 4 AND TO_CHAR(birth_date, 'DD-MM') = '31-12';
```

Получаем план запроса
```text
Seq Scan on public.users  (cost=0.00..283.44 rows=7 width=46) (actual time=0.402..1.539 rows=5 loops=1)
  Output: user_id, first_name, last_name, city_id, gender
  Filter: ((users.city_id = 4) AND (to_char((users.birth_date)::timestamp with time zone, 'DD-MM'::text) = '31-12'::text))
  Rows Removed by Filter: 9217
Query Identifier: 8550196113165767879
Planning Time: 0.222 ms
Execution Time: 1.561 ms
```

Видно, что затраты на снизились с `cost=0.00..2843.22` до `cost=0.00..283.44`, а время выполнения с 13.658 ms до 1.561 ms