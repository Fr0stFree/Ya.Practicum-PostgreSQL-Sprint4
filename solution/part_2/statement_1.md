## Оригинальный запрос

```sql
SELECT count(*)
FROM order_statuses os
    JOIN orders o ON o.order_id = os.order_id
WHERE (SELECT count(*)
	   FROM order_statuses os1
	   WHERE os1.order_id = o.order_id AND os1.status_id = 2) = 0
	AND o.city_id = 1;
```

## План выполнения

```text
Aggregate  (cost=61608580.82..61608580.83 rows=1 width=8) (actual time=8647.897..8647.898 rows=1 loops=1)
  Output: count(*)
  ->  Nested Loop  (cost=0.30..61608580.60 rows=90 width=0) (actual time=60.428..8647.771 rows=1190 loops=1)
        ->  Seq Scan on public.order_statuses os  (cost=0.00..2059.34 rows=124334 width=8) (actual time=0.006..4.648 rows=124334 loops=1)
              Output: os.order_id, os.status_id, os.status_dt
        ->  Memoize  (cost=0.30..2681.36 rows=1 width=8) (actual time=0.069..0.069 rows=0 loops=124334)
              Output: o.order_id
              Cache Key: os.order_id
              Cache Mode: logical
              Hits: 96650  Misses: 27684  Evictions: 0  Overflows: 0  Memory Usage: 1994kB
              ->  Index Scan using orders_order_id_idx on public.orders o  (cost=0.29..2681.35 rows=1 width=8) (actual time=0.310..0.310 rows=0 loops=27684)
                    Output: o.order_id
                    Index Cond: (o.order_id = os.order_id)
                    Filter: ((o.city_id = 1) AND ((SubPlan 1) = 0))
                    Rows Removed by Filter: 1
                    SubPlan 1
                      ->  Aggregate  (cost=2681.01..2681.02 rows=1 width=8) (actual time=2.163..2.163 rows=1 loops=3958)
                            Output: count(*)
                            ->  Seq Scan on public.order_statuses os1  (cost=0.00..2681.01 rows=1 width=0) (actual time=1.383..2.162 rows=1 loops=3958)
                                  Output: os1.order_id, os1.status_id, os1.status_dt
                                  Filter: ((os1.order_id = o.order_id) AND (os1.status_id = 2))
                                  Rows Removed by Filter: 124333
Query Identifier: -6346275186277548500
Planning Time: 0.155 ms
JIT:
  Functions: 19
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 0.786 ms, Inlining 7.688 ms, Optimization 23.710 ms, Emission 14.421 ms, Total 46.606 ms
Execution Time: 8648.778 ms
```

## Решение
Как видно из запроса, функция агрегации включающая в себя `Seq Scan` всей таблицы выполнялась 3958 раз. Это поведение можно оптимизировать.

Переделанный запрос:
```sql
SELECT count(*)
FROM order_statuses os
    JOIN orders o ON o.order_id = os.order_id
WHERE o.city_id = 1 AND NOT EXISTS (
	SELECT TRUE
   	FROM order_statuses os1
   	WHERE os1.order_id = o.order_id AND os1.status_id = 2
)
```

Новый план выполнения:
```text
Aggregate  (cost=5792.38..5792.39 rows=1 width=8) (actual time=27.040..27.042 rows=1 loops=1)
  Output: count(*)
  ->  Hash Join  (cost=3201.50..5779.32 rows=5224 width=0) (actual time=17.276..27.006 rows=1190 loops=1)
        Hash Cond: (os.order_id = o.order_id)
        ->  Seq Scan on public.order_statuses os  (cost=0.00..2059.34 rows=124334 width=8) (actual time=0.022..4.580 rows=124334 loops=1)
              Output: os.order_id, os.status_id, os.status_dt
        ->  Hash  (cost=3186.96..3186.96 rows=1163 width=8) (actual time=17.202..17.203 rows=1190 loops=1)
              Output: o.order_id
              Buckets: 2048  Batches: 1  Memory Usage: 63kB
              ->  Hash Right Anti Join  (cost=715.52..3186.96 rows=1163 width=8) (actual time=16.951..17.102 rows=1190 loops=1)
                    Output: o.order_id
                    Hash Cond: (os1.order_id = o.order_id)
                    ->  Seq Scan on public.order_statuses os1  (cost=0.00..2370.18 rows=19549 width=8) (actual time=0.007..10.129 rows=19330 loops=1)
                          Output: os1.order_id, os1.status_id, os1.status_dt
                          Filter: (os1.status_id = 2)
                          Rows Removed by Filter: 105004
                    ->  Hash  (cost=666.05..666.05 rows=3958 width=8) (actual time=4.702..4.703 rows=3958 loops=1)
                          Output: o.order_id
                          Buckets: 4096  Batches: 1  Memory Usage: 187kB
                          ->  Seq Scan on public.orders o  (cost=0.00..666.05 rows=3958 width=8) (actual time=0.040..4.043 rows=3958 loops=1)
                                Output: o.order_id
                                Filter: (o.city_id = 1)
                                Rows Removed by Filter: 23726
Query Identifier: 5774809395607120176
Planning Time: 0.760 ms
Execution Time: 27.145 ms
```

## Результат
- Execution Time: 8648.778 ms -> 27.145 ms
- cost: 61608580.82..61608580.83 -> 5792.38..5792.39
