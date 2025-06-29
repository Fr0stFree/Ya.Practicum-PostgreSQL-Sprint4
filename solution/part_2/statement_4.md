## Оригинальный запрос

```sql
SELECT o.order_id, o.order_dt, o.final_cost, s.status_name
FROM order_statuses os
    JOIN orders o ON o.order_id = os.order_id
    JOIN statuses s ON s.status_id = os.status_id
WHERE o.user_id = 'c2885b45-dddd-4df3-b9b3-2cc012df727c'::uuid
	AND os.status_dt IN (
	SELECT max(status_dt)
	FROM order_statuses
	WHERE order_id = o.order_id
    );
```

## План выполнения

```text
Nested Loop  (cost=15.51..33355.49 rows=7 width=46) (actual time=40.274..65.102 rows=2 loops=1)
  Output: o.order_id, o.order_dt, o.final_cost, s.status_name
  Join Filter: (os.status_id = s.status_id)
  Rows Removed by Join Filter: 10
  ->  Hash Join  (cost=15.51..33353.79 rows=7 width=26) (actual time=40.259..65.084 rows=2 loops=1)
        Output: os.status_id, o.order_id, o.order_dt, o.final_cost
        Hash Cond: (os.order_id = o.order_id)
        Join Filter: (SubPlan 1)
        Rows Removed by Join Filter: 10
        ->  Seq Scan on public.order_statuses os  (cost=0.00..2059.34 rows=124334 width=20) (actual time=0.011..10.331 rows=124334 loops=1)
              Output: os.order_id, os.status_id, os.status_dt
        ->  Hash  (cost=15.48..15.48 rows=3 width=22) (actual time=0.042..0.043 rows=2 loops=1)
              Output: o.order_id, o.order_dt, o.final_cost
              Buckets: 1024  Batches: 1  Memory Usage: 9kB
              ->  Bitmap Heap Scan on public.orders o  (cost=4.31..15.48 rows=3 width=22) (actual time=0.032..0.034 rows=2 loops=1)
                    Output: o.order_id, o.order_dt, o.final_cost
                    Recheck Cond: (o.user_id = 'c2885b45-dddd-4df3-b9b3-2cc012df727c'::uuid)
                    Heap Blocks: exact=1
                    ->  Bitmap Index Scan on orders_user_id_idx  (cost=0.00..4.31 rows=3 width=0) (actual time=0.018..0.018 rows=2 loops=1)
                          Index Cond: (o.user_id = 'c2885b45-dddd-4df3-b9b3-2cc012df727c'::uuid)
        SubPlan 1
          ->  Aggregate  (cost=2370.19..2370.20 rows=1 width=8) (actual time=3.673..3.673 rows=1 loops=12)
                Output: max(order_statuses.status_dt)
                ->  Seq Scan on public.order_statuses  (cost=0.00..2370.18 rows=5 width=8) (actual time=3.653..3.671 rows=6 loops=12)
                      Output: order_statuses.order_id, order_statuses.status_id, order_statuses.status_dt
                      Filter: (order_statuses.order_id = o.order_id)
                      Rows Removed by Filter: 124328
  ->  Materialize  (cost=0.00..1.09 rows=6 width=28) (actual time=0.006..0.007 rows=6 loops=2)
        Output: s.status_name, s.status_id
        ->  Seq Scan on public.statuses s  (cost=0.00..1.06 rows=6 width=28) (actual time=0.006..0.007 rows=6 loops=1)
              Output: s.status_name, s.status_id
Query Identifier: -2653455960943331794
Planning Time: 0.700 ms
Execution Time: 65.196 ms
```

## Решение

План выполнения запроса показывает, что для каждой строки из таблицы `orders` выполняется `Seq Scan` по таблице `order_statuses`, что приводит к значительным затратам времени. Это происходит из-за того, что подзапрос в условии `WHERE` выполняется для каждой строки.
Чтобы оптимизировать запрос, можно использовать оконную функцию `ROW_NUMBER()` для получения последнего статуса для каждого заказа. Это позволит избежать многократного сканирования таблицы `order_statuses` и значительно ускорит выполнение запроса.

Новый запрос:
```sql
WITH latest_statuses AS (
    SELECT
        o.order_id,
        o.order_dt,
        o.final_cost,
        s.status_name,
        ROW_NUMBER() OVER (PARTITION BY o.order_id ORDER BY os.status_dt DESC) AS rank
    FROM orders o
    JOIN order_statuses os ON os.order_id = o.order_id
    JOIN statuses s ON s.status_id = os.status_id
    WHERE o.user_id = 'c2885b45-dddd-4df3-b9b3-2cc012df727c'::uuid
)
SELECT
    order_id,
    order_dt,
    final_cost,
    status_name
FROM latest_statuses
WHERE rank = 1;
```

Новый план выполнения:
```text
Subquery Scan on latest_statuses  (cost=2543.72..2544.14 rows=1 width=46) (actual time=19.385..19.389 rows=2 loops=1)
  Output: latest_statuses.order_id, latest_statuses.order_dt, latest_statuses.final_cost, latest_statuses.status_name
  Filter: (latest_statuses.rank = 1)
  ->  WindowAgg  (cost=2543.72..2543.98 rows=13 width=62) (actual time=19.384..19.387 rows=2 loops=1)
        Output: o.order_id, o.order_dt, o.final_cost, s.status_name, row_number() OVER (?), os.status_dt
        Run Condition: (row_number() OVER (?) <= 1)
        ->  Sort  (cost=2543.72..2543.75 rows=13 width=54) (actual time=19.380..19.382 rows=12 loops=1)
              Output: o.order_id, os.status_dt, o.order_dt, o.final_cost, s.status_name
              Sort Key: o.order_id, os.status_dt DESC
              Sort Method: quicksort  Memory: 25kB
              ->  Nested Loop  (cost=15.51..2543.48 rows=13 width=54) (actual time=19.291..19.339 rows=12 loops=1)
                    Output: o.order_id, os.status_dt, o.order_dt, o.final_cost, s.status_name
                    Join Filter: (os.status_id = s.status_id)
                    Rows Removed by Join Filter: 60
                    ->  Hash Join  (cost=15.51..2541.24 rows=13 width=34) (actual time=19.275..19.316 rows=12 loops=1)
                          Output: o.order_id, o.order_dt, o.final_cost, os.status_dt, os.status_id
                          Hash Cond: (os.order_id = o.order_id)
                          ->  Seq Scan on public.order_statuses os  (cost=0.00..2059.34 rows=124334 width=20) (actual time=0.013..9.290 rows=124334 loops=1)
                                Output: os.order_id, os.status_id, os.status_dt
                          ->  Hash  (cost=15.48..15.48 rows=3 width=22) (actual time=0.029..0.030 rows=2 loops=1)
                                Output: o.order_id, o.order_dt, o.final_cost
                                Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                ->  Bitmap Heap Scan on public.orders o  (cost=4.31..15.48 rows=3 width=22) (actual time=0.022..0.024 rows=2 loops=1)
                                      Output: o.order_id, o.order_dt, o.final_cost
                                      Recheck Cond: (o.user_id = 'c2885b45-dddd-4df3-b9b3-2cc012df727c'::uuid)
                                      Heap Blocks: exact=1
                                      ->  Bitmap Index Scan on orders_user_id_idx  (cost=0.00..4.31 rows=3 width=0) (actual time=0.012..0.012 rows=2 loops=1)
                                            Index Cond: (o.user_id = 'c2885b45-dddd-4df3-b9b3-2cc012df727c'::uuid)
                    ->  Materialize  (cost=0.00..1.09 rows=6 width=28) (actual time=0.001..0.001 rows=6 loops=12)
                          Output: s.status_name, s.status_id
                          ->  Seq Scan on public.statuses s  (cost=0.00..1.06 rows=6 width=28) (actual time=0.009..0.009 rows=6 loops=1)
                                Output: s.status_name, s.status_id
Query Identifier: 8976742196456578637
Planning Time: 0.761 ms
Execution Time: 19.508 ms
```

## Результат
- Execution Time: 65.196 ms -> 19.508 ms
- cost: 15.51..33355.49 -> 2543.72..2544.14
