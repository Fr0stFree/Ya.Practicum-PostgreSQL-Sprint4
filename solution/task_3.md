## Задание

Также пользователи жалуются, что оплата при оформлении заказа проходит долго.
Разработчик сервера приложения Матвей проанализировал ситуацию и заключил, что оплата «висит» из-за того, что выполнение процедуры add_payment требует довольно много времени по меркам БД. 

```sql
CREATE OR REPLACE PROCEDURE public.add_payment(
    IN p_order_id bigint,
    IN p_sum_payment numeric
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO order_statuses (order_id, status_id, status_dt)
    VALUES (p_order_id, 2, statement_timestamp());
    
    INSERT INTO payments (payment_id, order_id, payment_sum)
    VALUES (nextval('payments_payment_id_sq'), p_order_id, p_sum_payment);
    
    INSERT INTO sales(sale_id, sale_dt, user_id, sale_sum)
    SELECT NEXTVAL('sales_sale_id_sq'), statement_timestamp(), user_id, p_sum_payment
    FROM orders WHERE order_id = p_order_id;
END;
$$
;
```
## Решение

### План запроса

Первая вставка :
```text
Insert on public.order_statuses  (cost=0.00..0.01 rows=0 width=0) (actual time=0.050..0.050 rows=0 loops=1)
  ->  Result  (cost=0.00..0.01 rows=1 width=20) (actual time=0.002..0.003 rows=1 loops=1)
        Output: '71'::bigint, 2, statement_timestamp()
Query Identifier: 1768409430379712834
Planning Time: 0.226 ms
Execution Time: 0.097 ms
```

Вторая вставка:
```text
Insert on public.payments  (cost=0.00..0.01 rows=0 width=0) (actual time=0.145..0.146 rows=0 loops=1)
  ->  Result  (cost=0.00..0.01 rows=1 width=34) (actual time=0.108..0.109 rows=1 loops=1)
        Output: nextval('payments_payment_id_sq'::regclass), '71'::bigint, 1000.00::numeric(14,2)
Query Identifier: -4182830878171279704
Planning Time: 0.090 ms
Execution Time: 0.178 ms
```

Третья вставка:
```text
Insert on public.sales  (cost=0.29..8.32 rows=0 width=0) (actual time=0.081..0.081 rows=0 loops=1)
  ->  Subquery Scan on "*SELECT*"  (cost=0.29..8.32 rows=1 width=50) (actual time=0.060..0.062 rows=1 loops=1)
        Output: "*SELECT*".nextval, "*SELECT*".statement_timestamp, "*SELECT*".user_id, "*SELECT*"."?column?"
        ->  Index Scan using orders_order_id_idx on public.orders  (cost=0.29..8.32 rows=1 width=36) (actual time=0.054..0.055 rows=1 loops=1)
              Output: nextval('sales_sale_id_sq'::regclass), statement_timestamp(), orders.user_id, 1000
              Index Cond: (orders.order_id = 71)
Query Identifier: -1721699486430091764
Planning Time: 0.466 ms
Execution Time: 0.112 ms
```

### Аналитика

1. Процедура уже кажется выполняется достаточно быстро.
2. Есть возможность ускорить третью вставку.

### Предложения

Перестроить индекс `orders_order_id_idx`:
```sql
DROP INDEX orders_order_id_idx;
CREATE UNIQUE INDEX orders_order_id_idx ON orders(order_id) INCLUDE (user_id);
```

Новый план запроса:
```text
Insert on public.sales  (cost=0.42..8.45 rows=0 width=0) (actual time=0.087..0.087 rows=0 loops=1)
  ->  Subquery Scan on "*SELECT*"  (cost=0.42..8.45 rows=1 width=50) (actual time=0.078..0.079 rows=1 loops=1)
        Output: "*SELECT*".nextval, "*SELECT*".statement_timestamp, "*SELECT*".user_id, "*SELECT*"."?column?"
        ->  Index Only Scan using orders_order_id_idx on public.orders  (cost=0.42..8.44 rows=1 width=36) (actual time=0.076..0.076 rows=1 loops=1)
              Output: nextval('sales_sale_id_sq'::regclass), statement_timestamp(), orders.user_id, 1000
              Index Cond: (orders.order_id = 71)
              Heap Fetches: 0
Query Identifier: -1721699486430091764
Planning Time: 0.249 ms
Execution Time: 0.105 ms
```

Улучшения производительности достигнуто не было, однако теперь используется `Index Only Scan` вместо `Index Scan`.
