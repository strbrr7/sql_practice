# 02. Агрегація: GROUP BY, HAVING, дати

**Правило, на якому тримається весь розділ:** кожна колонка в `SELECT` має бути або в `GROUP BY`, або всередині агрегатної функції. Умова на окремий рядок — у `WHERE`, умова на результат агрегації — у `HAVING`.

## Частина 1. Платежі

```sql
DROP TABLE IF EXISTS payments;
CREATE TABLE payments (
    id INTEGER PRIMARY KEY,
    merchant TEXT,
    amount INTEGER,
    currency TEXT,
    status TEXT,
    country TEXT,
    date TEXT
);
INSERT INTO payments VALUES
(1, 'Uklon',   200, 'UAH', 'success', 'UA', '2026-11-01'),
(2, 'Megogo',  150, 'UAH', 'failed',  'UA', '2026-11-01'),
(3, 'Uklon',   300, 'PLN', 'success', 'PL', '2026-11-02'),
(4, 'Rozetka', 500, 'UAH', 'success', 'UA', '2026-11-02'),
(5, 'Megogo',  120, 'EUR', 'failed',  'DE', '2026-11-03');
```

**1. Мерчант, сума і країна всіх відхилених платежів.**
```sql
SELECT merchant, amount, country
FROM payments
WHERE status = 'failed';
```
Результат: Megogo 150 UA; Megogo 120 DE.

**2. Успішні платежі з України або Польщі на суму понад 250.**
```sql
SELECT *
FROM payments
WHERE status = 'success'
  AND country IN ('UA', 'PL')
  AND amount > 250;
```
Результат: платежі 3 (Uklon, 300) і 4 (Rozetka, 500).

**3. Платежі з 1 по 2 листопада включно, від більшої суми до меншої.**
```sql
SELECT merchant, amount
FROM payments
WHERE date BETWEEN '2026-11-01' AND '2026-11-02'
ORDER BY amount DESC;
```
Результат: Rozetka 500, Uklon 300, Uklon 200, Megogo 150.

**4. Сума успішних платежів кожного мерчанта.**
```sql
SELECT merchant, SUM(amount) AS total
FROM payments
WHERE status = 'success'
GROUP BY merchant;
```
Результат: Uklon 500, Rozetka 500. Megogo немає: обидва його платежі відхилені.

**5. Кількість платежів у кожній країні.**
```sql
SELECT country, COUNT(*) AS payments_count
FROM payments
GROUP BY country
ORDER BY payments_count DESC;
```
Результат: UA 3, PL 1, DE 1.

**6. Мерчанти із загальною сумою платежів понад 300.**
```sql
SELECT merchant, SUM(amount) AS total
FROM payments
GROUP BY merchant
HAVING SUM(amount) > 300;
```
Результат: Uklon 500, Rozetka 500.

**7. Країни, де успішних платежів більше одного.**
```sql
SELECT country, COUNT(*) AS payments_count
FROM payments
WHERE status = 'success'
GROUP BY country
HAVING COUNT(*) > 1
ORDER BY payments_count DESC;
```
Результат: UA 2.

**8. Скільки всього відхилених платежів.**
```sql
SELECT COUNT(*) AS failed_count
FROM payments
WHERE status = 'failed';
```
Результат: 2.

**9. Кількість і сума платежів для кожного статусу.**
```sql
SELECT status, SUM(amount) AS total, COUNT(*) AS cnt
FROM payments
GROUP BY status
ORDER BY total DESC;
```
Результат: success 1000, 3; failed 270, 2.

**10. Мерчанти із середньою сумою платежу менше 200.**
```sql
SELECT merchant, AVG(amount) AS avg_amount
FROM payments
GROUP BY merchant
HAVING AVG(amount) < 200;
```
Результат: Megogo 135.

**11. Кількість успішних платежів за кожен день.**
```sql
SELECT date, COUNT(*) AS cnt
FROM payments
WHERE status = 'success'
GROUP BY date
ORDER BY date;
```
Результат: 2026-11-01 — 1; 2026-11-02 — 2.

**12. Мерчанти, у яких більше одного платежу в UAH.**
```sql
SELECT merchant, COUNT(*) AS cnt
FROM payments
WHERE currency = 'UAH'
GROUP BY merchant
HAVING COUNT(*) > 1;
```
Результат: порожній — у кожного мерчанта лише один платіж в UAH.

**13. Кількість платежів і максимальна сума мерчантів, у яких максимум понад 250.**
```sql
SELECT merchant, COUNT(*) AS cnt, MAX(amount) AS max_amount
FROM payments
GROUP BY merchant
HAVING MAX(amount) > 250
ORDER BY max_amount DESC;
```
Результат: Rozetka 1, 500; Uklon 2, 300.

## Частина 2. Транзакції: закріплення

```sql
DROP TABLE IF EXISTS transactions;
CREATE TABLE transactions (
    id INTEGER PRIMARY KEY,
    client TEXT,
    amount INTEGER,
    method TEXT,
    status TEXT,
    country TEXT,
    day TEXT
);
INSERT INTO transactions VALUES
(1, 'Nova Post', 100, 'card',       'success',  'UA', '2026-11-05'),
(2, 'Nova Post', 250, 'apple_pay',  'success',  'UA', '2026-11-05'),
(3, 'Uklon',      80, 'card',       'declined', 'UA', '2026-11-05'),
(4, 'Uklon',     300, 'card',       'success',  'PL', '2026-11-06'),
(5, 'Megogo',     50, 'google_pay', 'success',  'UA', '2026-11-06'),
(6, 'Megogo',    400, 'card',       'declined', 'DE', '2026-11-06'),
(7, 'Nova Post', 150, 'card',       'declined', 'PL', '2026-11-07'),
(8, 'Uklon',     200, 'apple_pay',  'success',  'UA', '2026-11-07');
```

**14. Кількість транзакцій за кожним способом оплати.**
```sql
SELECT method, COUNT(*) AS cnt
FROM transactions
GROUP BY method
ORDER BY cnt DESC;
```
Результат: card 5, apple_pay 2, google_pay 1.

**15. Сума успішних транзакцій кожного клієнта.**
```sql
SELECT client, SUM(amount) AS total
FROM transactions
WHERE status = 'success'
GROUP BY client
ORDER BY total DESC;
```
Результат: Uklon 500, Nova Post 350, Megogo 50.

**16. Країни, де було більше двох транзакцій.**
```sql
SELECT country, COUNT(*) AS total_transactions
FROM transactions
GROUP BY country
HAVING COUNT(*) > 2;
```
Результат: UA 5.

**17. Кількість відхилених транзакцій за кожен день.**
```sql
SELECT day, COUNT(*) AS declined_count
FROM transactions
WHERE status = 'declined'
GROUP BY day;
```
Результат: по одній відхиленій транзакції 5, 6 і 7 листопада.

**18. Клієнти із середньою сумою транзакції понад 170.**
```sql
SELECT client, AVG(amount) AS avg_amount
FROM transactions
GROUP BY client
HAVING AVG(amount) > 170
ORDER BY avg_amount;
```
Результат: Uklon 193.33, Megogo 225. Nova Post (166.67) не проходить фільтр.

**19. Кількість транзакцій для кожної пари клієнт — статус.**
```sql
SELECT client, status, COUNT(*) AS cnt
FROM transactions
GROUP BY client, status
ORDER BY client;
```
Результат: Megogo — declined 1, success 1; Nova Post — declined 1, success 2; Uklon — declined 1, success 2.

**20. Кількість відхилень за кожним способом оплати.**
```sql
SELECT method, COUNT(*) AS declined_count
FROM transactions
WHERE status = 'declined'
GROUP BY method
ORDER BY declined_count DESC;
```
Результат: card 3. Висновок: усі відхилення були з карткою.

**21. Оборот (сума успішних платежів) в Україні по днях, тільки дні з оборотом понад 200.**
```sql
SELECT day, SUM(amount) AS turnover
FROM transactions
WHERE status = 'success'
  AND country = 'UA'
GROUP BY day
HAVING SUM(amount) > 200;
```
Результат: 2026-11-05 — 350.

**22. Кількість транзакцій кожного клієнта в кожній країні, найактивніші зверху.**
```sql
SELECT client, country, COUNT(*) AS cnt
FROM transactions
GROUP BY client, country
ORDER BY cnt DESC, client;
```
Результат: Nova Post UA 2, Uklon UA 2, решта комбінацій по 1.

**23. Скільки різних клієнтів платили карткою.**
```sql
SELECT COUNT(DISTINCT client) AS clients_count
FROM transactions
WHERE method = 'card';
```
Результат: 3.

**24. Клієнти із сумою відхилених транзакцій понад 100.**
```sql
SELECT client, COUNT(*) AS declined_count, SUM(amount) AS declined_amount
FROM transactions
WHERE status = 'declined'
GROUP BY client
HAVING SUM(amount) > 100
ORDER BY declined_amount DESC;
```
Результат: Megogo 1, 400; Nova Post 1, 150.

## Частина 3. Задачі без бази (MySQL)

Ці задачі розв'язувались без доступу до бази, як на співбесіді, тільки за описом таблиць.

**25. Загальна сума продажів для кожного товару.**
Таблиця `sales (sale_id, product, amount)`.
```sql
SELECT product, SUM(amount) AS total_amount
FROM sales
GROUP BY product;
```
> Колонку `amount` окремо в SELECT не додаю: в одного товару багато продажів, і без агрегатної функції незрозуміло, яку суму показати.

**26. Клієнти із загальною сумою замовлень понад 1000.**
Таблиця `orders (order_id, customer_id, amount)`.
```sql
SELECT customer_id, SUM(amount) AS total_amount
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```

**27. Сума успішних транзакцій кожного клієнта, тільки клієнти із сумою понад 500.**
Таблиця `transactions (transaction_id, customer_id, amount, status)`.
```sql
SELECT customer_id, SUM(amount) AS total_completed
FROM transactions
WHERE status = 'completed'
GROUP BY customer_id
HAVING SUM(amount) > 500;
```
> Статус фільтрується у WHERE, ще до групування. Інакше відхилені транзакції потрапили б у суму.

**28. Сума транзакцій за кожен місяць, тільки місяці із сумою понад 1000.**
Таблиця `transactions (transaction_id, transaction_date, amount)`.
```sql
SELECT DATE_FORMAT(transaction_date, '%Y-%m') AS month,
       SUM(amount) AS total_amount
FROM transactions
GROUP BY DATE_FORMAT(transaction_date, '%Y-%m')
HAVING SUM(amount) > 1000
ORDER BY month;
```
> Групую за роком і місяцем (`%Y-%m`), а не тільки за місяцем: інакше листопад різних років злився б в одну групу.

**29. Сума успішних замовлень для кожної пари регіон — клієнт, тільки пари із сумою понад 2000.**
Таблиця `orders (order_id, customer_id, region, amount, status)`.
```sql
SELECT region, customer_id, SUM(amount) AS total_completed
FROM orders
WHERE status = 'completed'
GROUP BY region, customer_id
HAVING SUM(amount) > 2000;
```

**30. Клієнти з успішними транзакціями на суму понад 1000 за поточний місяць.**
Таблиця `transactions (transaction_id, customer_id, amount, transaction_date, status)`.
```sql
SELECT customer_id
FROM transactions
WHERE DATE_FORMAT(transaction_date, '%Y-%m') = DATE_FORMAT(CURDATE(), '%Y-%m')
  AND status = 'completed'
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```
> Обидві частини порівняння приведені до одного формату `рік-місяць`. Варіант для великих таблиць — фільтр діапазоном: `transaction_date >= DATE_FORMAT(CURDATE(), '%Y-%m-01') AND transaction_date < DATE_FORMAT(CURDATE(), '%Y-%m-01') + INTERVAL 1 MONTH`.
