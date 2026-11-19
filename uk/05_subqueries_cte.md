# 05. Підзапити і CTE

**Коли потрібен підзапит:** коли для перевірки умови треба спершу щось порахувати по всій таблиці (наприклад, середнє) або взяти список з іншого запиту. `WHERE` бачить лише один рядок, тому `WHERE amount > AVG(amount)` не працює.

**CTE (`WITH … AS`)** — це іменований підзапит: спершу рахуємо проміжну таблицю, даємо їй назву, а потім працюємо з нею як зі звичайною. Читається легше, ніж вкладені підзапити, і дозволяє будувати запит «поверхами».

## Дані

Таблиця платежів з мерчантами та еквайрами (використовується також у розділі 06).

```sql
DROP TABLE IF EXISTS payments;
CREATE TABLE payments (
    payment_id INTEGER PRIMARY KEY,
    merchant   TEXT,
    acquirer   TEXT,
    country    TEXT,
    amount     REAL,
    status     TEXT,
    created_at TEXT
);
INSERT INTO payments VALUES
(1,  'MEGOGO', 'Bank A', 'UA', 199,  'approved', '2026-11-01'),
(2,  'MEGOGO', 'Bank B', 'PL', 199,  'declined', '2026-11-01'),
(3,  'MEGOGO', 'Bank A', 'UA', 399,  'approved', '2026-11-02'),
(4,  'MEGOGO', 'Bank B', 'DE', 199,  'approved', '2026-11-03'),
(5,  'Bolt',   'Bank A', 'PL', 120,  'approved', '2026-11-01'),
(6,  'Bolt',   'Bank A', 'PL', 85,   'declined', '2026-11-02'),
(7,  'Bolt',   'Bank C', 'DE', 300,  'approved', '2026-11-02'),
(8,  'Bolt',   'Bank C', 'UA', 85,   'approved', '2026-11-03'),
(9,  'Ajax',   'Bank B', 'UA', 950,  'approved', '2026-11-01'),
(10, 'Ajax',   'Bank B', 'PL', 1200, 'declined', '2026-11-02'),
(11, 'Ajax',   'Bank C', 'DE', 950,  'approved', '2026-11-03'),
(12, 'Bolt',   'Bank A', 'UA', 50,   'approved', '2026-11-03');
```

## Підзапити

**1. Платежі, сума яких більша за середню по всій таблиці.**
```sql
SELECT payment_id, merchant, amount
FROM payments
WHERE amount > (SELECT AVG(amount) FROM payments)
ORDER BY amount DESC;
```
Результат: середня сума 394.67; вище за неї платежі 10 (Ajax 1200), 9 і 11 (Ajax 950), 3 (MEGOGO 399).
> Підзапит виконується першим і повертає одне число, яке потім підставляється в умову.

**2. Мерчанти, у яких був відхилений платіж понад 500.**
```sql
SELECT DISTINCT merchant
FROM payments
WHERE merchant IN (
    SELECT merchant
    FROM payments
    WHERE status = 'declined'
      AND amount > 500
);
```
Результат: Ajax.

**3. Клієнти, які хоча б раз заплатили більше за середню суму транзакції** (MySQL, задача без бази).
Таблиця `transactions (transaction_id, customer_id, amount, transaction_date)`.
```sql
SELECT DISTINCT customer_id
FROM transactions
WHERE amount > (SELECT AVG(amount) FROM transactions);
```
> `DISTINCT`, бо в одного клієнта може бути кілька великих транзакцій, а потрібен список клієнтів.

## CTE

**4. Approval rate кожного еквайра, від найгіршого до найкращого.**
```sql
WITH stats AS (
    SELECT acquirer,
           COUNT(*) AS total,
           SUM(CASE WHEN status = 'approved' THEN 1 ELSE 0 END) AS approved
    FROM payments
    GROUP BY acquirer
)
SELECT acquirer, total, approved,
       ROUND(100.0 * approved / total, 1) AS approval_rate
FROM stats
ORDER BY approval_rate;
```
| acquirer | total | approved | approval_rate |
|---|---|---|---|
| Bank B | 4 | 2 | 50.0 |
| Bank A | 5 | 4 | 80.0 |
| Bank C | 3 | 3 | 100.0 |

**5. Еквайри з approval rate нижчим за середній по всіх еквайрах** (два CTE поспіль + підзапит).
```sql
WITH stats AS (
    SELECT acquirer,
           COUNT(*) AS total,
           SUM(CASE WHEN status = 'approved' THEN 1 ELSE 0 END) AS approved
    FROM payments
    GROUP BY acquirer
),
rated AS (
    SELECT acquirer,
           ROUND(100.0 * approved / total, 1) AS approval_rate
    FROM stats
)
SELECT acquirer, approval_rate
FROM rated
WHERE approval_rate < (SELECT AVG(approval_rate) FROM rated);
```
Результат: Bank B (50.0).
> Кожен наступний CTE бере дані з попереднього, а фінальний запит — з останнього. `WITH` пишеться один раз, між CTE стоїть кома.

> Висновок для аналізу: у Bank B лише 4 платежі, тому перед рішенням (наприклад, перенаправити трафік) варто перевірити більший період, щоб це не була випадковість на малій вибірці.
