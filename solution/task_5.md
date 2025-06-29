## Задание

Маркетологи сервиса регулярно анализируют предпочтения различных возрастных групп. Для этого они формируют отчёт.
В столбцах spicy, fish и meat отображается, какой % блюд, заказанных каждой категорией пользователей, содержал эти признаки.
В возрастных интервалах верхний предел входит в интервал, а нижний — нет.
Также по правилам построения отчётов в них не включается текущий день.
Администратор БД Серёжа заметил, что регулярные похожие запросы от разных маркетологов нагружают базу, и в результате увеличивается время работы приложения.
Подумайте с точки зрения производительности, как можно оптимально собирать и хранить данные для такого отчёта. В ответе на это задание не пишите причину — просто опишите ваш способ получения отчёта и добавьте соответствующий скрипт.

## Решение

Поскольку отчет будет формироваться только за прошедшие дни, за которые не данные не будут обновляться, имеет смысл воспользоваться материализованным представлением.
Благодаря представлению можно игнорировать факт сложности запроса, поскольку REFRESH MATERIALIZED VIEW будет выполняться не чаще раза в день. В виду этого нет необходимости в поиске дополнительных путей оптимизации запроса.

```sql
CREATE MATERIALIZED VIEW age_group_favorite_dishes AS
	WITH user_orders AS (
		SELECT 
		    o.user_id,
		    u.birth_date,
		    o.order_dt::DATE AS day,
		    EXTRACT(YEAR FROM age(o.order_dt::DATE, birth_date)) AS age,
		    SUM(d.spicy) AS spicy_count,
		    SUM(d.fish) AS fish_count,
		    SUM(d.meat) AS meat_count,
		    COUNT(*) AS total_dishes
		FROM orders o
		JOIN order_items oi ON o.order_id = oi.order_id
		JOIN dishes d ON oi.item = d.object_id
		JOIN users u ON o.user_id = u.user_id
		WHERE o.order_dt::DATE != CURRENT_DATE
		GROUP BY o.order_id, o.user_id, u.birth_date, o.order_dt::DATE
	)
	SELECT
		day,
	    CASE
	        WHEN age < 20 THEN '0-20'
	        WHEN age >= 20 AND age < 30 THEN '20-30'
	        WHEN age >= 30 AND age < 40 THEN '30-40'
	        ELSE '40-100'
	    END AS age_group,
	    ROUND(100 * SUM(spicy_count) / SUM(total_dishes), 2) AS spicy,
	    ROUND(100 * SUM(fish_count) / SUM(total_dishes), 2) AS fish,
	    ROUND(100 * SUM(meat_count) / SUM(total_dishes), 2) AS meat
	FROM user_orders
	GROUP BY day, age_group;
```
