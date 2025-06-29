## Оригинальный запрос

```sql
SELECT d.name, SUM(count) AS orders_quantity
FROM order_items oi
    JOIN dishes d ON d.object_id = oi.item
WHERE oi.item IN (
	SELECT item
	FROM (SELECT item, SUM(count) AS total_sales
		  FROM order_items oi
		  GROUP BY 1) dishes_sales
	WHERE dishes_sales.total_sales > (
		SELECT SUM(t.total_sales)/ COUNT(*)
		FROM (SELECT item, SUM(count) AS total_sales
			FROM order_items oi
			GROUP BY
				1) t)
)
GROUP BY 1
ORDER BY orders_quantity DESC;
```

## План выполнения

```text
Sort  (cost=4808.91..4810.74 rows=735 width=66) (actual time=39.544..39.557 rows=362 loops=1)
  Sort Key: (sum(oi.count)) DESC
  Sort Method: quicksort  Memory: 48kB
  InitPlan 1 (returns $0)
    ->  Aggregate  (cost=1501.65..1501.66 rows=1 width=32) (actual time=9.209..9.210 rows=1 loops=1)
          ->  HashAggregate  (cost=1480.72..1490.23 rows=761 width=40) (actual time=9.108..9.170 rows=761 loops=1)
                Group Key: oi_2.item
                Batches: 1  Memory Usage: 169kB
                ->  Seq Scan on order_items oi_2  (cost=0.00..1134.48 rows=69248 width=16) (actual time=0.008..2.492 rows=69248 loops=1)
  ->  HashAggregate  (cost=3263.06..3272.25 rows=735 width=66) (actual time=39.441..39.474 rows=362 loops=1)
        Group Key: d.name
        Batches: 1  Memory Usage: 105kB
        ->  Hash Join  (cost=1522.66..3147.65 rows=23083 width=42) (actual time=28.216..35.642 rows=35854 loops=1)
              Hash Cond: (oi.item = d.object_id)
              ->  Seq Scan on order_items oi  (cost=0.00..1134.48 rows=69248 width=16) (actual time=0.013..2.299 rows=69248 loops=1)
              ->  Hash  (cost=1519.48..1519.48 rows=254 width=50) (actual time=28.198..28.199 rows=366 loops=1)
                    Buckets: 1024  Batches: 1  Memory Usage: 39kB
                    ->  Hash Join  (cost=1497.85..1519.48 rows=254 width=50) (actual time=28.074..28.169 rows=366 loops=1)
                          Hash Cond: (d.object_id = dishes_sales.item)
                          ->  Seq Scan on dishes d  (cost=0.00..19.62 rows=762 width=42) (actual time=0.007..0.043 rows=762 loops=1)
                          ->  Hash  (cost=1494.67..1494.67 rows=254 width=8) (actual time=28.060..28.061 rows=366 loops=1)
                                Buckets: 1024  Batches: 1  Memory Usage: 23kB
                                ->  Subquery Scan on dishes_sales  (cost=1480.72..1494.67 rows=254 width=8) (actual time=27.943..28.035 rows=366 loops=1)
                                      ->  HashAggregate  (cost=1480.72..1492.13 rows=254 width=40) (actual time=27.942..28.014 rows=366 loops=1)
                                            Group Key: oi_1.item
                                            Filter: (sum(oi_1.count) > $0)
                                            Batches: 1  Memory Usage: 169kB
                                            Rows Removed by Filter: 395
                                            ->  Seq Scan on order_items oi_1  (cost=0.00..1134.48 rows=69248 width=16) (actual time=0.004..4.925 rows=69248 loops=1)
Planning Time: 0.580 ms
Execution Time: 39.678 ms
```

## Решение

Воспользуемся CTE вместо подзапросов - это позволит переиспользовать результаты агрегации, а также упростит чтение запроса.

Новый запрос:
```sql
WITH dish_sales AS (
    SELECT item, SUM(count) AS total_sales
    FROM order_items
    GROUP BY item
),
avg_sales AS (
    SELECT AVG(total_sales::numeric) AS avg_sales
    FROM dish_sales
),
popular_dishes AS (
    SELECT item
    FROM dish_sales, avg_sales
    WHERE dish_sales.total_sales > avg_sales.avg_sales
)
SELECT d.name, SUM(oi.count) AS orders_quantity
FROM order_items oi
JOIN dishes d ON d.object_id = oi.item
JOIN popular_dishes pd ON pd.item = oi.item
GROUP BY d.name
ORDER BY orders_quantity DESC;
```

Новый план выполнения:
```text
Sort  (cost=3352.34..3354.18 rows=735 width=66) (actual time=31.893..31.908 rows=362 loops=1)
  Output: d.name, (sum(oi.count))
  Sort Key: (sum(oi.count)) DESC
  Sort Method: quicksort  Memory: 48kB
  CTE dish_sales
    ->  HashAggregate  (cost=1480.72..1490.23 rows=761 width=40) (actual time=19.361..19.419 rows=761 loops=1)
          Output: order_items.item, sum(order_items.count)
          Group Key: order_items.item
          Batches: 1  Memory Usage: 169kB
          ->  Seq Scan on public.order_items  (cost=0.00..1134.48 rows=69248 width=16) (actual time=0.004..5.213 rows=69248 loops=1)
                Output: order_items.order_id, order_items.item, order_items.count
  ->  HashAggregate  (cost=1817.93..1827.12 rows=735 width=66) (actual time=31.804..31.836 rows=362 loops=1)
        Output: d.name, sum(oi.count)
        Group Key: d.name
        Batches: 1  Memory Usage: 105kB
        ->  Hash Join  (cost=77.68..1702.67 rows=23052 width=42) (actual time=19.773..28.027 rows=35854 loops=1)
              Output: d.name, oi.count
              Hash Cond: (oi.item = d.object_id)
              ->  Seq Scan on public.order_items oi  (cost=0.00..1134.48 rows=69248 width=16) (actual time=0.017..2.925 rows=69248 loops=1)
                    Output: oi.order_id, oi.item, oi.count
              ->  Hash  (cost=74.51..74.51 rows=254 width=50) (actual time=19.750..19.751 rows=366 loops=1)
                    Output: d.name, d.object_id, dish_sales.item
                    Buckets: 1024  Batches: 1  Memory Usage: 39kB
                    ->  Hash Join  (cost=46.27..74.51 rows=254 width=50) (actual time=19.624..19.724 rows=366 loops=1)
                          Output: d.name, d.object_id, dish_sales.item
                          Hash Cond: (dish_sales.item = d.object_id)
                          ->  Nested Loop  (cost=17.13..41.87 rows=254 width=8) (actual time=19.525..19.592 rows=366 loops=1)
                                Output: dish_sales.item
                                Join Filter: (dish_sales.total_sales > (avg(dish_sales_1.total_sales)))
                                Rows Removed by Join Filter: 395
                                ->  Aggregate  (cost=17.13..17.14 rows=1 width=32) (actual time=19.522..19.522 rows=1 loops=1)
                                      Output: avg(dish_sales_1.total_sales)
                                      ->  CTE Scan on dish_sales dish_sales_1  (cost=0.00..15.22 rows=761 width=32) (actual time=19.363..19.486 rows=761 loops=1)
                                            Output: dish_sales_1.item, dish_sales_1.total_sales
                                ->  CTE Scan on dish_sales  (cost=0.00..15.22 rows=761 width=40) (actual time=0.000..0.024 rows=761 loops=1)
                                      Output: dish_sales.item, dish_sales.total_sales
                          ->  Hash  (cost=19.62..19.62 rows=762 width=42) (actual time=0.092..0.092 rows=762 loops=1)
                                Output: d.name, d.object_id
                                Buckets: 1024  Batches: 1  Memory Usage: 66kB
                                ->  Seq Scan on public.dishes d  (cost=0.00..19.62 rows=762 width=42) (actual time=0.007..0.046 rows=762 loops=1)
                                      Output: d.name, d.object_id
Query Identifier: -2562085024214437705
Planning Time: 0.584 ms
Execution Time: 32.031 ms
```

## Результат
- Execution Time: 39.678 ms -> 32.031 ms
- cost: 4808.91..4810.74 -> 3352.34..3354.18
- читаемость кода: ЗНАЧИТЕЛЬНОЕ УЛУЧШЕНИЕ
