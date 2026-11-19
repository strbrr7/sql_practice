# 01. Basics: SELECT, WHERE, ORDER BY, LIMIT, modifying data

Database: **Northwind** ([sql-practice.com](https://www.sql-practice.com/), `northwind.db`).

## Selecting data

**1. Names of all product categories.**
```sql
SELECT category_name
FROM categories;
```

**2. First and last names of all employees.**
```sql
SELECT first_name, last_name
FROM employees;
```

**3. All columns of the products table.**
```sql
SELECT *
FROM products;
```

**4. Company name and city of all customers.**
```sql
SELECT company_name, city
FROM customers;
```

## Filtering (WHERE)

**5. Products priced above 33 per unit.**
```sql
SELECT *
FROM products
WHERE unit_price > 33;
```

**6. Customers from Germany and Spain.**
```sql
SELECT *
FROM customers
WHERE country IN ('Germany', 'Spain');
```

**7. Products that are not discontinued and have more than 10 units in stock.**
```sql
SELECT product_name, units_in_stock, discontinued
FROM products
WHERE discontinued = 0
  AND units_in_stock > 10;
```

**8. Employees not based in the USA.**
```sql
SELECT first_name, last_name, country
FROM employees
WHERE country != 'USA';
```

**9. Customers with no region specified.**
```sql
SELECT contact_name, region
FROM customers
WHERE region IS NULL;
```
> `= NULL` does not work: NULL means "unknown value", and comparing with it never returns TRUE. Use `IS NULL` / `IS NOT NULL` instead.

**10. Products priced between 20 and 50 inclusive.**
```sql
SELECT product_name, unit_price
FROM products
WHERE unit_price BETWEEN 20 AND 50;
```
> `BETWEEN` includes both boundaries.

**11. Customers whose contact name starts with "C".**
```sql
SELECT contact_name
FROM customers
WHERE contact_name LIKE 'C%';
```

## Sorting and LIMIT

**12. Products sorted by price, ascending.**
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price;
```

**13. Employees in alphabetical order by last name.**
```sql
SELECT first_name, last_name
FROM employees
ORDER BY last_name;
```

**14. The five most expensive products.**
```sql
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC
LIMIT 5;
```

**15. The first three customers by company name.**
```sql
SELECT contact_name, company_name
FROM customers
ORDER BY company_name
LIMIT 3;
```

## Modifying data: INSERT, UPDATE, DELETE

After each change I ran a `SELECT` to check the result.

**16. Add a new category.**
```sql
INSERT INTO categories (category_id, category_name, description)
VALUES (9, 'Salads', 'Garden Salad');

SELECT * FROM categories;
```

**17. Change the description of the category with id = 1.**
```sql
UPDATE categories
SET description = 'Soft drinks and juices'
WHERE category_id = 1;

SELECT * FROM categories;
```

**18. Delete categories with id 3 and 4.**
```sql
DELETE FROM categories
WHERE category_id IN (3, 4);

SELECT * FROM categories;
```

**19. Raise prices by 10% for products from suppliers 3, 4 and 7.**
```sql
UPDATE products
SET unit_price = unit_price * 1.1
WHERE supplier_id IN (3, 4, 7);

SELECT supplier_id, unit_price FROM products;
```

**20. Delete products that are out of stock** (check them with a SELECT first).
```sql
SELECT product_id, product_name, units_in_stock
FROM products
WHERE units_in_stock = 0;

DELETE FROM products
WHERE units_in_stock = 0;
```
Result: products with `product_id` 5, 17, 29, 31, 53 were deleted.

> Before `UPDATE` and `DELETE`, it's worth running a `SELECT` with the same `WHERE` condition to see which rows will change. Without `WHERE`, the whole table is affected.
