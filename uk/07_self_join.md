# 07. Self join

**Self join** — це з'єднання таблиці самої з собою. Таблиця одна, але в запиті вона згадується двічі з різними аліасами, і я працюю з нею як із двома окремими таблицями.

Типові задачі:
- **ієрархія** (працівник → керівник), коли зв'язок записаний в одній таблиці;
- **пари записів**, які мають щось спільне (однакова дата, однакова зарплата, одне місто).

## Ієрархія: працівник і керівник

**1. Працівники, найняті у 2013 році, та їхні керівники** (Northwind, SQLite).
```sql
SELECT e.first_name, e.last_name, e.hire_date,
       m.first_name || ' ' || m.last_name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.reports_to = m.employee_id
WHERE e.hire_date >= '2013-01-01'
  AND e.hire_date <  '2014-01-01'
ORDER BY e.hire_date;
```
| first_name | last_name | hire_date | manager_name |
|---|---|---|---|
| Margaret | Peacock | 2013-05-03 | Andrew Fuller |
| Steven | Buchanan | 2013-10-17 | Andrew Fuller |
| Michael | Suyama | 2013-10-17 | Steven Buchanan |

> **LEFT JOIN**, бо в головного керівника `reports_to = NULL`, і з INNER JOIN він зник би з результату.
> Фільтр по року через діапазон, а не `LIKE '2013%'`: так запит працює і там, де дата зберігається як тип `DATE` (наприклад, у PostgreSQL).

## Пари записів без дублікатів

Якщо з'єднати таблицю саму з собою за спільною ознакою, кожен запис знайде в парі **сам себе**, а кожна пара з'явиться **двічі** (A–B і B–A). Обидві проблеми прибирає одна умова: `t1.id < t2.id`.

| t1 | t2 | `t1.id < t2.id` | |
|---|---|---|---|
| A (1) | A (1) | 1 < 1 — ні | ❌ сам із собою |
| A (1) | B (2) | 1 < 2 — так | ✅ |
| B (2) | A (1) | 2 < 1 — ні | ❌ дзеркальна пара |

**2. Пари клієнтів, які робили транзакції в один день** (без бази, MySQL).
Таблиця `transactions (transaction_id, customer_id, transaction_date)`.
```sql
SELECT DISTINCT t1.customer_id AS customer_1,
                t2.customer_id AS customer_2
FROM transactions t1
JOIN transactions t2 ON t1.transaction_date = t2.transaction_date
WHERE t1.customer_id < t2.customer_id;
```
> `DISTINCT` тут потрібен: одна й та сама пара клієнтів могла збігтися в кількох різних днях.

**3. Пари співробітників з однаковою зарплатою** (без бази).
Таблиця `employees (employee_id, name, salary)`.
```sql
SELECT e1.employee_id, e1.name,
       e2.employee_id, e2.name,
       e1.salary
FROM employees e1
JOIN employees e2 ON e1.salary = e2.salary
WHERE e1.employee_id < e2.employee_id;
```
> Тут `DISTINCT` не потрібен: у кожного співробітника один рядок, тож кожна пара з'являється рівно один раз.

**Формула:** self join + `=` по спільній ознаці + `<` по ID (+ `DISTINCT`, якщо в одного об'єкта може бути кілька рядків).
