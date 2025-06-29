## Задание

Клиенты сервиса начали замечать, что после нажатия на кнопку Оформить заказ система на какое-то время подвисает.

Вот команда для вставки данных в таблицу orders, которая хранит общую информацию о заказах:
```sql
INSERT INTO orders
    (order_id, order_dt, user_id, device_type, city_id, total_cost, discount, 
    final_cost)
SELECT MAX(order_id) + 1, current_timestamp, 
    '329551a1-215d-43e6-baee-322f2467272d', 
    'Mobile', 1, 1000.00, null, 1000.00
FROM orders;
```

### Решение

#### План запроса

```text
Insert on public.orders  (cost=0.32..0.35 rows=0 width=0) (actual time=0.361..0.364 rows=0 loops=1)
  ->  Subquery Scan on "*SELECT*"  (cost=0.32..0.35 rows=1 width=122) (actual time=0.082..0.085 rows=1 loops=1)
        Output: "*SELECT*"."?column?", "*SELECT*"."current_timestamp", '329551a1-215d-43e6-baee-322f2467272d'::uuid, 'Mobile'::character varying, "*SELECT*"."?column?_3", "*SELECT*"."?column?_4", NULL::numeric(14,2), "*SELECT*"."?column?_6"
        ->  Result  (cost=0.32..0.33 rows=1 width=180) (actual time=0.076..0.079 rows=1 loops=1)
              Output: ($0 + 1), CURRENT_TIMESTAMP, NULL::unknown, NULL::unknown, 1, 1000.00, NULL::unknown, 1000.00
              InitPlan 1 (returns $0)
                ->  Limit  (cost=0.29..0.32 rows=1 width=8) (actual time=0.064..0.066 rows=1 loops=1)
                      Output: orders_1.order_id
                      ->  Index Only Scan Backward using orders_order_id_idx on public.orders orders_1  (cost=0.29..805.15 rows=27684 width=8) (actual time=0.062..0.062 rows=1 loops=1)
                            Output: orders_1.order_id
                            Index Cond: (orders_1.order_id IS NOT NULL)
                            Heap Fetches: 1
Query Identifier: -948664044709798107
Planning Time: 0.338 ms
Execution Time: 0.442 ms
```

#### Аналитика

При анализе плана запроса видно, что проблемным местом является выражение `SELECT MAX(order_id) + 1 FROM orders` поскольку оно провоцирует базу данных использовать `Index Only Scan` по всему индексу `orders_order_id_idx`. Цена запроса уже составляет `cost=0.29..805.15`. Чем больше будет строк в таблице, тем больше времени займет сканирование. Так же это выражение уязвимо к конкурентным запросам вставки - для сохранения уникальности первичного ключа необходимо блокировать таблицу на вставку на время вычисления нового id. 

Так же я заметил, что на таблице orders поддерживается 10 индексов, что, по-моему мнению, является не только избыточным, но и может привести к значительному замедлению запрос на вставку ввиду необходимости обновления индексов.

```sql
SELECT indexname, indexdef 
FROM pg_indexes 
WHERE tablename = 'orders';
```
Output:
```text
orders_city_id_idx	CREATE INDEX orders_city_id_idx ON public.orders USING btree (city_id)
orders_device_type_city_id_idx	CREATE INDEX orders_device_type_city_id_idx ON public.orders USING btree (device_type, city_id)
orders_device_type_idx	CREATE INDEX orders_device_type_idx ON public.orders USING btree (device_type)
orders_discount_idx	CREATE INDEX orders_discount_idx ON public.orders USING btree (discount)
orders_final_cost_idx	CREATE INDEX orders_final_cost_idx ON public.orders USING btree (final_cost)
orders_order_dt_idx	CREATE INDEX orders_order_dt_idx ON public.orders USING btree (order_dt)
orders_order_id_idx	CREATE INDEX orders_order_id_idx ON public.orders USING btree (order_id)
orders_total_cost_idx	CREATE INDEX orders_total_cost_idx ON public.orders USING btree (total_cost)
orders_total_final_cost_discount_idx	CREATE INDEX orders_total_final_cost_discount_idx ON public.orders USING btree (total_cost, discount, final_cost)
orders_user_id_idx	CREATE INDEX orders_user_id_idx ON public.orders USING btree (user_id)
```

#### Возможное решение

Очевидным решением кажется использование встроенных в СУБД метода определения значения первичного ключе - `sequence`.

```sql
CREATE SEQUENCE orders_order_id_seq;
ALTER TABLE orders ALTER COLUMN order_id SET DEFAULT nextval('orders_order_id_seq');
```
