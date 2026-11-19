# SQL Payments Analytics

🇬🇧 [English](../README.md) | 🇺🇦 Українська

Мої розв'язки задач з SQL, згруповані за темами: від базових запитів до JOIN, CTE, віконних функцій і self join. Більшість задач побудовано на прикладах **платіжних транзакцій** (мерчанти, еквайри, статуси, approval rate), частина — на навчальній базі **Northwind**.

## Зміст

| # | Тема | Що всередині |
|---|---|---|
| 01 | [Основи](01_basics.md) | SELECT, WHERE, IN, BETWEEN, LIKE, IS NULL, ORDER BY, LIMIT, INSERT / UPDATE / DELETE |
| 02 | [Агрегація](02_aggregation.md) | COUNT, SUM, AVG, MAX, GROUP BY, HAVING, WHERE vs HAVING, робота з датами |
| 03 | [JOIN](03_joins.md) | INNER JOIN, LEFT JOIN, JOIN кількох таблиць, пошук записів без пари |
| 04 | [CASE WHEN](04_case_when.md) | умовні підрахунки, approval rate, категоризація |
| 05 | [Підзапити і CTE](05_subqueries_cte.md) | порівняння із середнім по таблиці, `WITH`, кілька CTE поспіль |
| 06 | [Віконні функції](06_window_functions.md) | `OVER`, `PARTITION BY`, ROW_NUMBER / RANK / DENSE_RANK, LAG, FIRST_VALUE, накопичувальна сума, топ-N у групі |
| 07 | [Self join](07_self_join.md) | працівник і керівник, пари записів без дублікатів |
| 08 | [Описова статистика](08_statistics.md) | середнє vs медіана, викиди, дисперсія, перцентилі |

## Дані

- **Платіжні таблиці** (`payments`, `transactions`, `clients`). На початку кожного розділу є SQL, який створює таблицю з даними, тому ці задачі можна запустити, наприклад, на [sqliteonline.com](https://sqliteonline.com).
- **Northwind** — класична навчальна база (замовлення, товари, клієнти, працівники). Запити виконувались на [sql-practice.com](https://www.sql-practice.com/), база `northwind.db`.
- Частина задач розділів 02, 05 і 07 написана для **MySQL** (функції `DATE_FORMAT`, `CURDATE`), це позначено в тексті.
