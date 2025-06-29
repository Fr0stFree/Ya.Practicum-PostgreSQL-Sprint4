## Оригинальный запрос

```sql
SELECT event, datetime
FROM user_logs
WHERE visitor_uuid = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'
ORDER BY 2;
```

## План выполнения

```text
Gather Merge  (cost=92121.53..92144.87 rows=200 width=19) (actual time=135.233..136.637 rows=10 loops=1)
  Output: user_logs.event, user_logs.datetime
  Workers Planned: 2
  Workers Launched: 2
  ->  Sort  (cost=91121.51..91121.76 rows=100 width=19) (actual time=118.822..118.823 rows=3 loops=3)
        Output: user_logs.event, user_logs.datetime
        Sort Key: user_logs.datetime
        Sort Method: quicksort  Memory: 25kB
        Worker 0:  actual time=111.042..111.042 rows=2 loops=1
          Sort Method: quicksort  Memory: 25kB
        Worker 1:  actual time=111.030..111.031 rows=3 loops=1
          Sort Method: quicksort  Memory: 25kB
        ->  Parallel Append  (cost=0.00..91118.19 rows=100 width=19) (actual time=18.174..118.768 rows=3 loops=3)
              Worker 0:  actual time=21.026..110.984 rows=2 loops=1
              Worker 1:  actual time=16.697..110.976 rows=3 loops=1
              ->  Parallel Seq Scan on public.user_logs_y2021q2 user_logs_2  (cost=0.00..66465.16 rows=60 width=18) (actual time=58.534..94.443 rows=2 loops=3)
                    Output: user_logs_2.event, user_logs_2.datetime
                    Filter: ((user_logs_2.visitor_uuid)::text = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'::text)
                    Rows Removed by Filter: 1132470
                    Worker 0:  actual time=21.024..110.981 rows=2 loops=1
                    Worker 1:  actual time=86.176..86.176 rows=0 loops=1
              ->  Parallel Seq Scan on public.user_logs user_logs_1  (cost=0.00..24071.52 rows=32 width=18) (actual time=14.779..34.515 rows=2 loops=2)
                    Output: user_logs_1.event, user_logs_1.datetime
                    Filter: ((user_logs_1.visitor_uuid)::text = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'::text)
                    Rows Removed by Filter: 615119
                    Worker 1:  actual time=16.696..24.797 rows=3 loops=1
              ->  Parallel Seq Scan on public.user_logs_y2021q3 user_logs_3  (cost=0.00..570.06 rows=10 width=18) (actual time=3.932..3.932 rows=0 loops=1)
                    Output: user_logs_3.event, user_logs_3.datetime
                    Filter: ((user_logs_3.visitor_uuid)::text = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'::text)
                    Rows Removed by Filter: 25304
              ->  Parallel Seq Scan on public.user_logs_y2021q4 user_logs_4  (cost=0.00..10.96 rows=1 width=282) (actual time=0.002..0.002 rows=0 loops=1)
                    Output: user_logs_4.event, user_logs_4.datetime
                    Filter: ((user_logs_4.visitor_uuid)::text = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'::text)
Query Identifier: -5326245778553925066
Planning Time: 0.479 ms
Execution Time: 136.736 ms
```
## Решение

Поскольку отсутствует индекс на user_id, планировщик вынужден обойти каждую из партиций с помощью `Seq Scan` и отфильтровать по user_id, что крайне неэффективно для большой таблицы с логамии.

Можно добавить индексы с целью ускорения поиска пользователя среди логов. Если позволяет память, можно использовать покрывающий индекс добавив информацию по времени и событию:
```sql
CREATE INDEX ON user_logs_y2021q2 (visitor_uuid) INCLUDE (EVENT, datetime);
CREATE INDEX ON user_logs_y2021q3 (visitor_uuid) INCLUDE (EVENT, datetime);
CREATE INDEX ON user_logs_y2021q4 (visitor_uuid) INCLUDE (EVENT, datetime);
CREATE INDEX ON user_logs (visitor_uuid) INCLUDE (EVENT, datetime);
```

Новый план выполнения выглядит следующим образом:
```text
Sort  (cost=44.39..44.99 rows=240 width=19) (actual time=0.244..0.246 rows=10 loops=1)
  Output: user_logs.event, user_logs.datetime
  Sort Key: user_logs.datetime
  Sort Method: quicksort  Memory: 25kB
  ->  Append  (cost=0.55..34.90 rows=240 width=19) (actual time=0.160..0.226 rows=10 loops=1)
        ->  Index Only Scan using user_logs_visitor_uuid_event_datetime_idx on public.user_logs user_logs_1  (cost=0.55..9.90 rows=77 width=18) (actual time=0.159..0.161 rows=5 loops=1)
              Output: user_logs_1.event, user_logs_1.datetime
              Index Cond: (user_logs_1.visitor_uuid = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'::text)
              Heap Fetches: 0
        ->  Index Only Scan using user_logs_y2021q2_visitor_uuid_event_datetime_idx on public.user_logs_y2021q2 user_logs_2  (cost=0.56..15.09 rows=145 width=18) (actual time=0.043..0.045 rows=5 loops=1)
              Output: user_logs_2.event, user_logs_2.datetime
              Index Cond: (user_logs_2.visitor_uuid = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'::text)
              Heap Fetches: 0
        ->  Index Only Scan using user_logs_y2021q3_visitor_uuid_event_datetime_idx on public.user_logs_y2021q3 user_logs_3  (cost=0.41..8.71 rows=17 width=18) (actual time=0.013..0.013 rows=0 loops=1)
              Output: user_logs_3.event, user_logs_3.datetime
              Index Cond: (user_logs_3.visitor_uuid = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'::text)
              Heap Fetches: 0
        ->  Seq Scan on public.user_logs_y2021q4 user_logs_4  (cost=0.00..0.00 rows=1 width=282) (actual time=0.003..0.003 rows=0 loops=1)
              Output: user_logs_4.event, user_logs_4.datetime
              Filter: ((user_logs_4.visitor_uuid)::text = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'::text)
Query Identifier: -5326245778553925066
Planning Time: 1.496 ms
Execution Time: 0.302 ms
```

## Результат
- Execution Time: 136.736 ms -> 0.302 ms
- cost: 92121.53..92144.87 -> 44.39..44.99
- memory: ~ +35% (очевидный минус)
