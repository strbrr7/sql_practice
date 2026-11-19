# 06. Віконні функції

```
функція() OVER (PARTITION BY [група] ORDER BY [порядок])
```

- **`PARTITION BY`** — для кого рахуємо (як GROUP BY, але рядки не схлопуються).
- **`ORDER BY`** у `OVER` — хто перший і хто «сусід». Обов'язковий для рангів, накопичувальних сум, LAG / LEAD.
- Віконні функції рахуються на етапі `SELECT`, тобто після `WHERE`. Тому фільтрувати їх результат можна тільки через CTE або підзапит.

| Задача | Функція | Дужки |
|---|---|---|
| значення групи поруч з рядком | `AVG / SUM / COUNT (колонка) OVER (PARTITION BY …)` | колонка |
| нумерація, топ-N | `ROW_NUMBER() / RANK() / DENSE_RANK()` | порожні |
| накопичувальна сума | `SUM(колонка) OVER (… ORDER BY …)` | колонка |
| попередній / наступний рядок | `LAG(колонка) / LEAD(колонка)` | колонка |
| перше значення в групі | `FIRST_VALUE(колонка)` | колонка |

## Частина 1. Платежі

Дані — таблиця `payments` з розділу [05](05_subqueries_cte.md#дані).

**1. Кожен платіж поруч із середньою сумою, кількістю платежів мерчанта та відхиленням від середнього.**
```sql
SELECT payment_id, merchant, amount,
       ROUND(AVG(amount) OVER (PARTITION BY merchant), 2)          AS avg_merchant,
       COUNT(*) OVER (PARTITION BY merchant)                       AS n_payments,
       ROUND(amount - AVG(amount) OVER (PARTITION BY merchant), 2) AS diff
FROM payments;
```
| payment_id | merchant | amount | avg_merchant | n_payments | diff |
|---|---|---|---|---|---|
| 9 | Ajax | 950 | 1033.33 | 3 | -83.33 |
| 10 | Ajax | 1200 | 1033.33 | 3 | 166.67 |
| 11 | Ajax | 950 | 1033.33 | 3 | -83.33 |
| 5 | Bolt | 120 | 128 | 5 | -8 |
| 6 | Bolt | 85 | 128 | 5 | -43 |
| 7 | Bolt | 300 | 128 | 5 | 172 |
| 8 | Bolt | 85 | 128 | 5 | -43 |
| 12 | Bolt | 50 | 128 | 5 | -78 |
| 1 | MEGOGO | 199 | 249 | 4 | -50 |
| 2 | MEGOGO | 199 | 249 | 4 | -50 |
| 3 | MEGOGO | 399 | 249 | 4 | 150 |
| 4 | MEGOGO | 199 | 249 | 4 | -50 |

> На відміну від GROUP BY, усі 12 рядків залишились. Колонка `diff` одразу показує платежі, які сильно відрізняються від звичних для мерчанта.

**2. ROW_NUMBER, RANK і DENSE_RANK: різниця на однакових сумах.**
```sql
SELECT merchant, amount,
       ROW_NUMBER() OVER (PARTITION BY merchant ORDER BY amount DESC) AS rn,
       RANK()       OVER (PARTITION BY merchant ORDER BY amount DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY merchant ORDER BY amount DESC) AS drnk
FROM payments
WHERE merchant = 'Bolt';
```
| merchant | amount | rn | rnk | drnk |
|---|---|---|---|---|
| Bolt | 300 | 1 | 1 | 1 |
| Bolt | 120 | 2 | 2 | 2 |
| Bolt | 85 | 3 | 3 | 3 |
| Bolt | 85 | 4 | 3 | 3 |
| Bolt | 50 | 5 | 5 | 4 |

> RANK пропускає номер після нічиєї (рахує, скільки рядків вище), DENSE_RANK не пропускає (рахує, скільки різних значень вище). ROW_NUMBER завжди дає унікальні номери.

**3. Два найбільші платежі кожного мерчанта.**
```sql
WITH ranked AS (
    SELECT merchant, amount,
           ROW_NUMBER() OVER (PARTITION BY merchant ORDER BY amount DESC) AS rn
    FROM payments
)
SELECT merchant, amount, rn
FROM ranked
WHERE rn <= 2;
```
Результат: Ajax 1200, 950; Bolt 300, 120; MEGOGO 399, 199.
> З `RANK` у Ajax було б три рядки: обидва платежі по 950 ділять друге місце.

**4. Накопичувальна сума платежів кожного мерчанта.**
```sql
SELECT merchant, created_at, amount,
       SUM(amount) OVER (PARTITION BY merchant ORDER BY created_at, payment_id) AS running_total
FROM payments;
```
Результат для MEGOGO: 199 → 398 → 797 → 996. Для кожного мерчанта сума починається заново.
> `payment_id` у сортуванні потрібен, щоб платежі одного дня рахувались по черзі, а не одним блоком.

**5. Зміна суми порівняно з попереднім платежем мерчанта.**
```sql
SELECT merchant, created_at, amount,
       LAG(amount) OVER (PARTITION BY merchant ORDER BY created_at, payment_id)          AS prev_amount,
       amount - LAG(amount) OVER (PARTITION BY merchant ORDER BY created_at, payment_id) AS change
FROM payments;
```
Перший платіж кожного мерчанта має NULL: попереднього платежу немає.

**6. Approval rate по днях і його зміна порівняно з попереднім днем.**
```sql
WITH daily AS (
    SELECT created_at,
           ROUND(100.0 * SUM(CASE WHEN status = 'approved' THEN 1 ELSE 0 END) / COUNT(*), 1) AS approval_rate
    FROM payments
    GROUP BY created_at
)
SELECT created_at, approval_rate,
       LAG(approval_rate) OVER (ORDER BY created_at) AS prev_rate,
       ROUND(approval_rate - LAG(approval_rate) OVER (ORDER BY created_at), 1) AS change
FROM daily;
```
| created_at | approval_rate | prev_rate | change |
|---|---|---|---|
| 2026-11-01 | 75.0 | | |
| 2026-11-02 | 50.0 | 75.0 | -25.0 |
| 2026-11-03 | 100.0 | 50.0 | 50.0 |

> Різке падіння approval rate (−25 п.п. 2 листопада) — сигнал перевірити, з якими еквайрами і кодами відмов це пов'язано.

## Частина 2. Northwind

База: [sql-practice.com](https://www.sql-practice.com/), `northwind.db` (SQLite).

**7. Попереднє замовлення кожного клієнта.**
```sql
SELECT c.customer_id, o.order_id, o.order_date,
       LAG(o.order_id) OVER (PARTITION BY c.customer_id ORDER BY o.order_date, o.order_id) AS previous_order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;
```

**8. Перше замовлення кожного клієнта.**
```sql
SELECT c.customer_id, o.order_id, o.order_date,
       FIRST_VALUE(o.order_id) OVER (PARTITION BY c.customer_id ORDER BY o.order_date, o.order_id) AS first_order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;
```
> `LAST_VALUE` з таким самим `ORDER BY` повернув би поточний рядок, бо за замовчуванням вікно закінчується на поточному рядку. Для останнього значення простіше взяти `FIRST_VALUE` з `ORDER BY … DESC`.

**9. Три найдорожчі товари в кожній категорії.**
```sql
WITH ranked AS (
    SELECT c.category_id, c.category_name, p.product_id, p.product_name, p.unit_price,
           RANK() OVER (PARTITION BY c.category_id ORDER BY p.unit_price DESC) AS product_rank
    FROM products p
    LEFT JOIN categories c ON c.category_id = p.category_id
)
SELECT category_id, category_name, product_id, product_name, unit_price, product_rank
FROM ranked
WHERE product_rank <= 3
ORDER BY category_id, product_rank;
```
Результат (Beverages): Côte de Blaye 263.5, Ipoh Coffee 46, Chang 19.

**10. Інтервал у днях між поточним і попереднім замовленням клієнта.**
```sql
WITH prev_d AS (
    SELECT c.customer_id, o.order_id, o.order_date,
           LAG(o.order_date) OVER (PARTITION BY c.customer_id ORDER BY o.order_date, o.order_id) AS previous_order_date
    FROM customers c
    JOIN orders o ON o.customer_id = c.customer_id
)
SELECT customer_id, order_id, order_date, previous_order_date,
       JULIANDAY(order_date) - JULIANDAY(previous_order_date) AS diff_days
FROM prev_d;
```
Результат для ALFKI: 39, 10, 94, 60 днів між замовленнями.

**11. Середній інтервал між замовленнями клієнтів по регіонах.**
```sql
WITH prev_od AS (
    SELECT c.region, o.order_date,
           LAG(o.order_date) OVER (PARTITION BY c.customer_id ORDER BY o.order_date, o.order_id) AS previous_order_date
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
),
diff_days AS (
    SELECT region,
           JULIANDAY(order_date) - JULIANDAY(previous_order_date) AS diff_days
    FROM prev_od
)
SELECT region,
       ROUND(AVG(diff_days), 2) AS avg_diff_days
FROM diff_days
GROUP BY region
ORDER BY avg_diff_days;
```
| region | avg_diff_days |
|---|---|
| Northern Europe | 44.22 |
| North America | 49.21 |
| Western Europe | 53.96 |
| Scandinavia | 54.28 |
| British Isles | 58.45 |
| South America | 65.28 |
| Eastern Europe | 84.00 |
| Southern Europe | 89.07 |
| Central America | 94.39 |

**Висновки.** Найчастіше повторні замовлення роблять у Northern Europe, найрідше — у Central America. Але щоб коректно порівнювати регіони, треба знати кількість клієнтів у кожному: якщо в Central America лише кілька клієнтів, середнє залежить від поведінки однієї-двох людей, і одна довга перерва між замовленнями помітно його підніме. Тому я б додала кількість клієнтів (`COUNT(DISTINCT customer_id)`) і подивилась на медіану, яка стійкіша до викидів. Також у розрахунок не потрапили клієнти з одним замовленням: у них немає інтервалу.

**12. Три працівники з найбільшою кількістю оброблених замовлень.**
```sql
WITH order_count_table AS (
    SELECT e.employee_id, e.first_name, e.last_name,
           COUNT(o.order_id) AS order_count
    FROM employees e
    LEFT JOIN orders o ON o.employee_id = e.employee_id
    GROUP BY e.employee_id, e.first_name, e.last_name
),
ranked AS (
    SELECT employee_id, first_name, last_name, order_count,
           RANK() OVER (ORDER BY order_count DESC) AS employee_rank
    FROM order_count_table
)
SELECT employee_id, first_name, last_name, order_count, employee_rank
FROM ranked
WHERE employee_rank <= 3
ORDER BY employee_rank;
```
> Тут `PARTITION BY` не потрібен: рейтинг загальний, серед усіх працівників.

**13. Три найкращі працівники в кожному регіоні.**
```sql
WITH order_count_table AS (
    SELECT e.employee_id, e.first_name, e.last_name, e.region,
           COUNT(o.order_id) AS order_count
    FROM employees e
    LEFT JOIN orders o ON o.employee_id = e.employee_id
    GROUP BY e.employee_id, e.first_name, e.last_name, e.region
),
ranked AS (
    SELECT employee_id, region, first_name, last_name, order_count,
           RANK() OVER (PARTITION BY region ORDER BY order_count DESC) AS employee_rank
    FROM order_count_table
)
SELECT region, employee_id, first_name, last_name, order_count, employee_rank
FROM ranked
WHERE employee_rank <= 3
ORDER BY region, employee_rank;
```
| region | employee_id | first_name | last_name | order_count | employee_rank |
|---|---|---|---|---|---|
| British Isles | 7 | Robert | King | 72 | 1 |
| British Isles | 6 | Michael | Suyama | 67 | 2 |
| British Isles | 9 | Anne | Dodsworth | 43 | 3 |
| North America | 4 | Margaret | Peacock | 156 | 1 |
| North America | 3 | Janet | Leverling | 127 | 2 |
| North America | 1 | Nancy | Davolio | 123 | 3 |
