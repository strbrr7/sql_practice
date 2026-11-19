# 01. Основи: SELECT, WHERE, ORDER BY, LIMIT, зміна даних

База: **Northwind** ([sql-practice.com](https://www.sql-practice.com/), `northwind.db`).

## Вибірка даних

**1. Назви всіх категорій товарів.**
```sql
SELECT category_name
FROM categories;
```

**2. Імена та прізвища всіх працівників.**
```sql
SELECT first_name, last_name
FROM employees;
```

**3. Усі поля таблиці товарів.**
```sql
SELECT *
FROM products;
```

**4. Назва компанії та місто всіх клієнтів.**
```sql
SELECT company_name, city
FROM customers;
```

## Фільтрація (WHERE)

**5. Товари, що коштують понад 33 за одиницю.**
```sql
SELECT *
FROM products
WHERE unit_price > 33;
```

**6. Клієнти з Німеччини та Іспанії.**
```sql
SELECT *
FROM customers
WHERE country IN ('Germany', 'Spain');
```

**7. Товари, які не зняті з продажу і мають на складі понад 10 одиниць.**
```sql
SELECT product_name, units_in_stock, discontinued
FROM products
WHERE discontinued = 0
  AND units_in_stock > 10;
```

**8. Працівники не з США.**
```sql
SELECT first_name, last_name, country
FROM employees
WHERE country != 'USA';
```

**9. Клієнти, у яких не заповнений регіон.**
```sql
SELECT contact_name, region
FROM customers
WHERE region IS NULL;
```
> `= NULL` не працює: NULL означає «невідоме значення», і порівняння з ним не дає TRUE. Для перевірки потрібні `IS NULL` / `IS NOT NULL`.

**10. Товари з ціною від 20 до 50 включно.**
```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price BETWEEN 20 AND 50;
```
> `BETWEEN` включає обидві межі.

**11. Клієнти, ім'я контактної особи яких починається на «C».**
```sql
SELECT contact_name
FROM customers
WHERE contact_name LIKE 'C%';
```

## Сортування і LIMIT

**12. Товари за зростанням ціни.**
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price;
```

**13. Працівники в алфавітному порядку за прізвищем.**
```sql
SELECT first_name, last_name
FROM employees
ORDER BY last_name;
```

**14. П'ять найдорожчих товарів.**
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC
LIMIT 5;
```

**15. Перші три клієнти за назвою компанії.**
```sql
SELECT contact_name, company_name
FROM customers
ORDER BY company_name
LIMIT 3;
```

## Зміна даних: INSERT, UPDATE, DELETE

Після кожної зміни виконувала `SELECT`, щоб перевірити результат.

**16. Додати нову категорію.**
```sql
INSERT INTO categories (category_id, category_name, description)
VALUES (9, 'Salads', 'Garden Salad');

SELECT * FROM categories;
```

**17. Змінити опис категорії з id = 1.**
```sql
UPDATE categories
SET description = 'Soft drinks and juices'
WHERE category_id = 1;

SELECT * FROM categories;
```

**18. Видалити категорії з id 3 і 4.**
```sql
DELETE FROM categories
WHERE category_id IN (3, 4);

SELECT * FROM categories;
```

**19. Підвищити на 10% ціни товарів постачальників 3, 4 і 7.**
```sql
UPDATE products
SET unit_price = unit_price * 1.1
WHERE supplier_id IN (3, 4, 7);

SELECT supplier_id, unit_price FROM products;
```

**20. Видалити товари, яких немає на складі** (спершу перевірити їх SELECT-ом).
```sql
SELECT product_id, product_name, units_in_stock
FROM products
WHERE units_in_stock = 0;

DELETE FROM products
WHERE units_in_stock = 0;
```
Результат: видалено товари з `product_id` 5, 17, 29, 31, 53.

> Перед `UPDATE` і `DELETE` варто запускати `SELECT` з тією самою умовою `WHERE`, щоб побачити, які рядки зміняться. Без `WHERE` зміниться вся таблиця.
