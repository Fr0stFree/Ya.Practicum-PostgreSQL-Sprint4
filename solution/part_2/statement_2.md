```sql
SELECT *
FROM user_logs
WHERE datetime::date > current_date;
```