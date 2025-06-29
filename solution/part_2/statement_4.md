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

## Результат
- Execution Time: ->
- cost:  -> 