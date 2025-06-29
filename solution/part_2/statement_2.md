## Оригинальный запрос

```sql
SELECT *
FROM user_logs
WHERE datetime::date > current_date;
```

## План выполнения

```text
Append  (cost=0.00..155994.71 rows=1550935 width=83) (actual time=289.468..289.469 rows=0 loops=1)
  ->  Seq Scan on public.user_logs user_logs_1  (cost=0.00..39193.25 rows=410081 width=83) (actual time=92.853..92.853 rows=0 loops=1)
        Output: user_logs_1.visitor_uuid, user_logs_1.user_id, user_logs_1.event, user_logs_1.datetime, user_logs_1.log_date, user_logs_1.log_id
        Filter: ((user_logs_1.datetime)::date > CURRENT_DATE)
        Rows Removed by Filter: 1230243
  ->  Seq Scan on public.user_logs_y2021q2 user_logs_2  (cost=0.00..108219.96 rows=1132418 width=83) (actual time=195.039..195.039 rows=0 loops=1)
        Output: user_logs_2.visitor_uuid, user_logs_2.user_id, user_logs_2.event, user_logs_2.datetime, user_logs_2.log_date, user_logs_2.log_id
        Filter: ((user_logs_2.datetime)::date > CURRENT_DATE)
        Rows Removed by Filter: 3397415
  ->  Seq Scan on public.user_logs_y2021q3 user_logs_3  (cost=0.00..826.82 rows=8435 width=83) (actual time=1.571..1.571 rows=0 loops=1)
        Output: user_logs_3.visitor_uuid, user_logs_3.user_id, user_logs_3.event, user_logs_3.datetime, user_logs_3.log_date, user_logs_3.log_id
        Filter: ((user_logs_3.datetime)::date > CURRENT_DATE)
        Rows Removed by Filter: 25304
  ->  Seq Scan on public.user_logs_y2021q4 user_logs_4  (cost=0.00..0.00 rows=1 width=584) (actual time=0.002..0.002 rows=0 loops=1)
        Output: user_logs_4.visitor_uuid, user_logs_4.user_id, user_logs_4.event, user_logs_4.datetime, user_logs_4.log_date, user_logs_4.log_id
        Filter: ((user_logs_4.datetime)::date > CURRENT_DATE)
Query Identifier: -1499737530410714232
Planning Time: 0.287 ms
JIT:
  Functions: 8
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 0.771 ms, Inlining 0.000 ms, Optimization 0.731 ms, Emission 7.815 ms, Total 9.317 ms
Execution Time: 290.313 ms
```

## Решение
Преобразование ::DATE приводит к тому, что планировщик игнорирует индексы и в результате сканирует каждую из партиций целиком с помощью `Seq Scan`. Этого можно избежать, если отказаться от преобразования. В таком случае планировщик будет использовать `Index Scan`, а запрос значительно ускорится.

Новый план выполнения:
```text
Append  (cost=0.43..25.22 rows=4 width=208) (actual time=0.036..0.038 rows=0 loops=1)
  ->  Index Scan using user_logs_datetime_idx on public.user_logs user_logs_1  (cost=0.43..8.45 rows=1 width=83) (actual time=0.016..0.017 rows=0 loops=1)
        Output: user_logs_1.visitor_uuid, user_logs_1.user_id, user_logs_1.event, user_logs_1.datetime, user_logs_1.log_date, user_logs_1.log_id
        Index Cond: (user_logs_1.datetime > CURRENT_DATE)
  ->  Index Scan using user_logs_y2021q2_datetime_idx on public.user_logs_y2021q2 user_logs_2  (cost=0.43..8.45 rows=1 width=83) (actual time=0.007..0.007 rows=0 loops=1)
        Output: user_logs_2.visitor_uuid, user_logs_2.user_id, user_logs_2.event, user_logs_2.datetime, user_logs_2.log_date, user_logs_2.log_id
        Index Cond: (user_logs_2.datetime > CURRENT_DATE)
  ->  Index Scan using user_logs_y2021q3_datetime_idx on public.user_logs_y2021q3 user_logs_3  (cost=0.29..8.30 rows=1 width=83) (actual time=0.005..0.005 rows=0 loops=1)
        Output: user_logs_3.visitor_uuid, user_logs_3.user_id, user_logs_3.event, user_logs_3.datetime, user_logs_3.log_date, user_logs_3.log_id
        Index Cond: (user_logs_3.datetime > CURRENT_DATE)
  ->  Seq Scan on public.user_logs_y2021q4 user_logs_4  (cost=0.00..0.00 rows=1 width=584) (actual time=0.006..0.006 rows=0 loops=1)
        Output: user_logs_4.visitor_uuid, user_logs_4.user_id, user_logs_4.event, user_logs_4.datetime, user_logs_4.log_date, user_logs_4.log_id
        Filter: (user_logs_4.datetime > CURRENT_DATE)
Query Identifier: -5740535644218160602
Planning Time: 0.720 ms
Execution Time: 0.096 ms
```

## Результат
- Execution Time: 290.313 ms -> 0.720 ms
- cost: 0.00..155994.71 -> 0.43..25.22 
