## Задание
Все действия пользователей в системе логируются и записываются в таблицу user_logs. Потом эти данные используются для анализа — как правило, анализируются данные за текущий квартал.

Время записи данных в эту таблицу сильно увеличилось, а это тормозит практически все действия пользователя. Подумайте, как можно ускорить запись. Вы можете сдать решение этой задачи без скрипта или — попробовать написать скрипт. Дерзайте!

Пример запроса на вставку:
```sql
INSERT INTO user_logs (user_id, event, visitor_uuid, datetime, log_date)
VALUES (GEN_RANDOM_UUID(), 'main_page', GEN_RANDOM_UUID(), NOW(), NOW());
```

Пример запроса на агрегацию:
```sql
SELECT COUNT(*) FROM user_logs 
WHERE log_date BETWEEN '2023-07-01' AND '2023-09-30'
```

## Решение

### План запроса

Вставка:
```text
Insert on public.user_logs  (cost=0.00..0.04 rows=0 width=0) (actual time=0.087..0.087 rows=0 loops=1)
  ->  Result  (cost=0.00..0.04 rows=1 width=584) (actual time=0.051..0.051 rows=1 loops=1)
        Output: gen_random_uuid(), gen_random_uuid(), 'main_page'::character varying(128), now(), now(), nextval('user_logs_log_id_seq'::regclass)
Query Identifier: -7866248341110819757
Planning Time: 0.039 ms
Execution Time: 0.109 ms
```

Агрегация:
```text
Aggregate  (cost=96826.01..96826.02 rows=1 width=8) (actual time=187.643..195.704 rows=1 loops=1)
  Output: count(*)
  ->  Gather  (cost=1000.00..96826.01 rows=1 width=0) (actual time=187.639..195.700 rows=0 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Parallel Seq Scan on public.user_logs  (cost=0.00..95825.91 rows=1 width=0) (actual time=164.451..164.451 rows=0 loops=3)
              Filter: ((user_logs.log_date >= '2023-07-01'::date) AND (user_logs.log_date <= '2023-09-30'::date))
              Rows Removed by Filter: 1550988
              Worker 0:  actual time=153.391..153.392 rows=0 loops=1
              Worker 1:  actual time=153.395..153.395 rows=0 loops=1
Query Identifier: -4348947636004933098
Planning Time: 0.880 ms
Execution Time: 195.872 ms
```

### Аналитика
- Вставка показывает довольно неплохие результаты
- Агрегация выглядит крайне дорогой
- Можно попробовать оптимизировать путем партицирования таблицы поквартально

### Предложения

Создаем партицированную таблицу:
```sql
CREATE TABLE user_logs_partitioned (
    log_id BIGSERIAL,
    user_id UUID,
    event VARCHAR(128),
    visitor_uuid UUID,
    datetime TIMESTAMP,
    log_date DATE,
    PRIMARY KEY (log_id, log_date)
) PARTITION BY RANGE (log_date);
```

Создадим партиции:
```sql
CREATE TABLE user_logs_y2021q1 PARTITION OF user_logs_partitioned
    FOR VALUES FROM ('2021-01-01') TO ('2021-04-01');

CREATE TABLE user_logs_y2021q2 PARTITION OF user_logs_partitioned
    FOR VALUES FROM ('2021-04-01') TO ('2021-07-01');

...

CREATE TABLE user_logs_y2025q4 PARTITION OF user_logs_partitioned
    FOR VALUES FROM ('2025-10-01') TO ('2026-01-01');
```

Переносим данные:
```sql
INSERT INTO user_logs_partitioned(
    user_id, event, visitor_uuid, datetime, log_date
)
	SELECT user_id, event, visitor_uuid::UUID, datetime, log_date
    FROM user_logs;
```

Новый план запроса на вставку лога:
```text
Insert on public.user_logs_partitioned  (cost=0.00..0.03 rows=0 width=0) (actual time=0.302..0.302 rows=0 loops=1)
  ->  Result  (cost=0.00..0.03 rows=1 width=326) (actual time=0.217..0.218 rows=1 loops=1)
        Output: nextval('user_logs_partitioned_log_id_seq'::regclass), gen_random_uuid(), 'main_page'::character varying(128), gen_random_uuid(), now(), now()
Query Identifier: 1376247848952293145
Planning Time: 0.073 ms
Execution Time: 0.353 ms
```

Новый план запроса на агрегацию:
```text
Aggregate  (cost=10.46..10.47 rows=1 width=8) (actual time=0.039..0.039 rows=1 loops=1)
  Output: count(*)
  ->  Index Only Scan using user_logs_y2023q3_pkey on public.user_logs_y2023q3 user_logs_partitioned  (cost=0.14..10.46 rows=1 width=0) (actual time=0.035..0.035 rows=0 loops=1)
        Index Cond: ((user_logs_partitioned.log_date >= '2023-07-01'::date) AND (user_logs_partitioned.log_date <= '2023-09-30'::date))
        Heap Fetches: 0
Query Identifier: 5594714076357519935
Planning Time: 1.236 ms
Execution Time: 0.084 ms
```

Достигнуто выраженное снижение cost=96826.01..96826.02 до cost=10.46..10.47 и времени с 195.872 ms до 0.084 ms в агрегирующих поквартальных запросах.