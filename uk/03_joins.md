# 03. JOIN

- **INNER JOIN** (або просто `JOIN`) залишає тільки рядки, для яких знайшлась пара в обох таблицях.
- **LEFT JOIN** залишає всі рядки з лівої таблиці; де пари немає, підставляється NULL.
- Якщо після LEFT JOIN у `WHERE` стоїть умова на колонку **правої** таблиці, LEFT JOIN фактично працює як INNER: NULL не проходить умову.

## Частина 1. Платежі і клієнти

### INNER JOIN

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS clients;
CREATE TABLE payments (id INTEGER PRIMARY KEY, client_id INTEGER, amount INTEGER);
CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT);
INSERT INTO payments VALUES (1, 1, 200), (2, 2, 150), (3, 1, 300);
INSERT INTO clients VALUES (1, 'Uklon'), (2, 'Megogo');
```

**1. Назва клієнта і сума для кожного платежу.**
```sql
SELECT c.name, p.amount
FROM clients c
JOIN payments p ON c.id = p.client_id;
```
Результат: Uklon 200, Megogo 150, Uklon 300.

### JOIN + WHERE + GROUP BY

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS clients;
CREATE TABLE payments (id INTEGER PRIMARY KEY, client_id INTEGER, amount INTEGER, status TEXT);
CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT, industry TEXT);
INSERT INTO payments VALUES
(1, 1, 200, 'success'),
(2, 2, 150, 'declined'),
(3, 1, 300, 'success'),
(4, 3, 500, 'success'),
(5, 2, 100, 'success');
INSERT INTO clients VALUES
(1, 'Uklon', 'transport'),
(2, 'Megogo', 'media'),
(3, 'Rozetka', 'ecommerce');
```

**2. Сума успішних платежів кожного клієнта.**
```sql
SELECT c.name, SUM(p.amount) AS total
FROM payments p
JOIN clients c ON c.id = p.client_id
WHERE p.status = 'success'
GROUP BY c.id, c.name
ORDER BY total DESC;
```
Результат: Uklon 500, Rozetka 500, Megogo 100.

### LEFT JOIN

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS clients;
CREATE TABLE payments (id INTEGER PRIMARY KEY, client_id INTEGER, amount INTEGER);
CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT);
INSERT INTO clients VALUES (1, 'Uklon'), (2, 'Megogo'), (4, 'Nova Post');
INSERT INTO payments VALUES (1, 1, 200), (2, 2, 150), (3, 1, 300);
```

**3. Усі клієнти і кількість їхніх платежів, навіть якщо платежів немає.**
```sql
SELECT c.name, COUNT(p.id) AS payments_count
FROM clients c
LEFT JOIN payments p ON c.id = p.client_id
GROUP BY c.id, c.name;
```
Результат: Uklon 2, Megogo 1, Nova Post 0.
> Після LEFT JOIN рахую `COUNT(p.id)`, а не `COUNT(*)`: у Nova Post є рядок з NULL, і `COUNT(*)` порахував би його як 1.

**4. Клієнти без жодного платежу.**
```sql
SELECT c.name
FROM clients c
LEFT JOIN payments p ON c.id = p.client_id
WHERE p.id IS NULL;
```
Результат: Nova Post.

## Частина 2. Змішаний раунд

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS clients;
CREATE TABLE payments (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,
    amount INTEGER,
    method TEXT,
    status TEXT,
    day TEXT
);
CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT, industry TEXT);
INSERT INTO payments VALUES
(1, 1, 200, 'card',       'success',  '2026-11-10'),
(2, 1, 100, 'apple_pay',  'declined', '2026-11-10'),
(3, 2, 300, 'card',       'success',  '2026-11-11'),
(4, 3,  50, 'card',       'declined', '2026-11-11'),
(5, 3, 400, 'google_pay', 'success',  '2026-11-12'),
(6, 1, 250, 'card',       'success',  '2026-11-12'),
(7, 2, 150, 'apple_pay',  'success',  '2026-11-12');
INSERT INTO clients VALUES
(1, 'Uklon',     'transport'),
(2, 'Megogo',    'media'),
(3, 'Rozetka',   'ecommerce'),
(4, 'Nova Post', 'logistics'),
(5, 'Bolt',      'transport');
```

**5. Усі успішні платежі карткою, від більшої суми до меншої.**
```sql
SELECT *
FROM payments
WHERE status = 'success'
  AND method = 'card'
ORDER BY amount DESC;
```
Результат: платежі 3 (300), 6 (250), 1 (200).
> Тут не потрібні ні GROUP BY, ні SUM: треба показати рядки, а не порахувати.

**6. Скільки різних клієнтів зробили хоча б один платіж.**
```sql
SELECT COUNT(DISTINCT client_id) AS clients_count
FROM payments;
```
Результат: 3.

**7. Клієнти із сумою успішних платежів понад 300.**
```sql
SELECT c.name, SUM(p.amount) AS total
FROM payments p
JOIN clients c ON c.id = p.client_id
WHERE p.status = 'success'
GROUP BY c.id, c.name
HAVING SUM(p.amount) > 300;
```
Результат: Uklon 450, Megogo 450, Rozetka 400.

**8. Клієнти, які ще жодного разу не платили.**
```sql
SELECT c.name
FROM clients c
LEFT JOIN payments p ON c.id = p.client_id
WHERE p.id IS NULL;
```
Результат: Nova Post, Bolt.

**9. Кількість платежів для кожної галузі.**
```sql
SELECT c.industry, COUNT(p.id) AS payments_count
FROM payments p
JOIN clients c ON c.id = p.client_id
GROUP BY c.industry
ORDER BY payments_count DESC;
```
Результат: transport 3, media 2, ecommerce 2.

**10. Дні, де було більше одного платежу: кількість і середня сума.**
```sql
SELECT day, COUNT(*) AS payments_count, AVG(amount) AS avg_amount
FROM payments
GROUP BY day
HAVING COUNT(*) > 1;
```
Результат: 2026-11-10 — 2, 150; 2026-11-11 — 2, 175; 2026-11-12 — 3, 266.67.

## Частина 3. Northwind

База: [sql-practice.com](https://www.sql-practice.com/), `northwind.db`.

**11. Усі замовлення з іменем працівника, який його обробив.**
```sql
SELECT o.order_id, e.first_name, e.last_name
FROM orders o
LEFT JOIN employees e ON e.employee_id = o.employee_id
ORDER BY o.order_id;
```
> LEFT JOIN, бо потрібні **всі** замовлення, навіть якщо в якогось не вказаний працівник.

**12. Замовлення за жовтень 2016 року з назвою компанії клієнта, за зростанням дати.**
```sql
SELECT o.order_id, o.order_date, c.company_name
FROM orders o
LEFT JOIN customers c ON c.customer_id = o.customer_id
WHERE o.order_date >= '2016-10-01'
  AND o.order_date <  '2016-11-01'
ORDER BY o.order_date;
```
> `BETWEEN '2016-10-01' AND '2016-11-01'` захопив би й 1 листопада, тому межі задаю через `>=` і `<`.

**13. Товари постачальників з регіонів North America і Western Europe, за назвою постачальника.**
```sql
SELECT p.product_name, s.company_name, s.region
FROM products p
JOIN suppliers s ON s.supplier_id = p.supplier_id
WHERE s.region IN ('North America', 'Western Europe')
ORDER BY s.company_name;
```

**14. Для кожного замовленого товару: у скількох замовленнях він був і скільки одиниць продано.**
```sql
SELECT p.product_name,
       COUNT(DISTINCT od.order_id) AS n_orders,
       SUM(od.quantity)            AS sum_quantity
FROM order_details od
JOIN products p ON p.product_id = od.product_id
GROUP BY p.product_id, p.product_name
ORDER BY n_orders, p.product_name;
```
> `COUNT` рахує, скільки разів щось зустрічається, а `SUM` складає числа. Тут потрібні обидві функції.

**15. Клієнти, у яких понад 7 замовлень.**
```sql
SELECT c.company_name, COUNT(o.order_id) AS n_orders
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.company_name
HAVING COUNT(o.order_id) > 7
ORDER BY n_orders DESC;
```

**16. Працівники, які працюють у територіях, назва яких починається з «Santa».**
```sql
SELECT DISTINCT e.first_name, e.last_name
FROM employees e
JOIN employee_territories et ON et.employee_id = e.employee_id
JOIN territories t ON t.territory_id = et.territory_id
WHERE t.territory_description LIKE 'Santa%';
```
Результат: Robert King.
> Без `DISTINCT` Robert King з'являється тричі, бо працює в трьох таких територіях.

**17. Товари дорожчі за 20 з назвою постачальника.**
```sql
SELECT p.product_name, p.unit_price, s.company_name
FROM products p
LEFT JOIN suppliers s ON s.supplier_id = p.supplier_id
WHERE p.unit_price > 20
ORDER BY p.unit_price DESC;
```
> Тут умова у WHERE стоїть на **ліву** таблицю, тож LEFT JOIN справді зберігає товари без постачальника.

**18. Категорії, у яких понад 5 товарів.**
```sql
SELECT c.category_name, COUNT(p.product_id) AS n_products
FROM categories c
JOIN products p ON p.category_id = c.category_id
GROUP BY c.category_id, c.category_name
HAVING COUNT(p.product_id) > 5
ORDER BY n_products DESC;
```

## Частина 4. Задача без бази

**19. Імена клієнтів з регіону Kyiv і загальний баланс їхніх рахунків.**
Таблиці `customers (customer_id, customer_name, region)` і `accounts (account_id, customer_id, balance)`.
```sql
SELECT c.customer_name,
       COALESCE(SUM(a.balance), 0) AS total_balance
FROM customers c
LEFT JOIN accounts a ON a.customer_id = c.customer_id
WHERE c.region = 'Kyiv'
GROUP BY c.customer_id, c.customer_name;
```
> Групую за `customer_id`, а не тільки за ім'ям: у двох різних клієнтів може бути однакове ім'я. `COALESCE` показує 0 замість NULL для клієнтів без рахунків.
