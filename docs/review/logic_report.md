# Отчёт по ревью логики: маршруты, база данных, тесты



## 3.1. Ревью маршрутов и обработчиков

### Общая картина

Активные маршруты регистрируются в `create_app()` из `app/views/` (CBV) + `web_bp` + `auth_bp`.  
Старые `app/blueprints/*.py` (кроме `web.py` и `auth.py`) **не подключены** — их проблемы не влияют на рантайм, но они остаются в кодовой базе.

Все CBV наследуются от `BaseAPIView` (`_success_response`, `_error_response`, `_not_found_response`) — это обеспечивает единый формат ответов.

### 3.1.1. Валидация входных данных

| Маршрут | Валидация | Проблема |
|---------|-----------|-------|
| `POST /api/clients/` | Проверка `name`, `client_type` |  Не проверяется допустимое значение `client_type` (должно быть `Поставщик` / `Покупатель`, CHECK в БД есть, но API вернёт 500 при нарушении, а не 400) |
| `PUT /api/clients/<id>` | `uuid.UUID(client_id)` + `data` |  ок |
| `POST /api/materials/` | `name`, `code` | ⚠ Нет проверки на пустые строки (`name=""` пройдёт валидацию, но CHECK в БД `NOT NULL` не сработает — пустая строка допустима) |
| `PUT /api/materials/<id>/price` | `price` в JSON |  Не проверяется, что `price` — число и `>= 0` (CHECK в БД есть, но вернётся 500, а не 400) |
| `POST /api/products/` | `name`, `code` | ⚠ Аналогично материалам — пустые строки и дубликаты `code` → 500 (UNIQUE в БД) |
| `POST /api/operations/` | `name`, `code` | ⚠ То же |
| `POST /api/specifications/materials/` | `product_id`, `material_id`, `quantity_per_unit` | ⚠️ `quantity_per_unit` не проверяется на `> 0` (CHECK в БД есть) |
| `POST /api/specifications/operations/` | `product_id`, `operation_id` | ⚠`time_norm` / `op_quantity` по умолчанию 1.0, не проверяются на `> 0` |
| `POST /api/production-orders/` | `order_date` (обязательно) | ⚠ `date.fromisoformat(od)` может бросить `ValueError` при неверном формате — **не обёрнуто в try/except → 500** |
| `PUT /api/production-orders/<id>` | `order_date` | ⚠ То же: `date.fromisoformat` без try/except |
| `POST /api/sales-orders/` | `order_date`, `client_id`, `executor_id` | ⚠ `date.fromisoformat` без try/except; UUID — ок |
| `PUT /api/sales-orders/<id>` | `order_date` | ⚠ `date.fromisoformat` без try/except |
| `POST /api/admin/users/` | `login`, `password` |  ок (проверка существующего логина, роли) |
| `PUT /api/admin/users/<id>` | `role`, `is_blocked` |  ок |
| `POST /api/auth/login` | `login`, `password` |  ок (в `AuthService`) |
| `POST /api/notes/` (web) | `title`, `content` |  ок |

**Вывод:** валидация присутствует, но **неполная**: не проверяются диапазоны значений (`price >= 0`, `quantity > 0`), не проверяются enum-значения (`client_type`, `status`), не обрабатываются ошибки парсинга дат.

### 3.1.2. Обработка ошибок

**Сильные стороны:**
- `BaseAPIView` даёт единые `_error_response` / `_not_found_response`.
- В `create_app()` зарегистрированы обработчики 400/404/500, возвращающие JSON.
- В `notes` (functional) есть `try/except` с `db.session.rollback()`.

**Проблемы:**

1. **Необработанные исключения → 500.**
    - `date.fromisoformat(od)` в `production_order_view.py` (post/put) и `sales_order_view.py` (post/put) — при неверной дате будет `ValueError` → 500 вместо 400.
    - `float(item_data.get("quantity", 1))` в `sales_order_view.py` — при строке `"abc"` → `ValueError` → 500.
    - `int(...)` в `format_note_id`? — нет, там `str`.

2. **Нарушение CHECK-ограничений БД → 500.**
    - `client_type="Другой"` → `IntegrityError` → 500 (ожидается 400).
    - `price=-5` → `IntegrityError` → 500.
    - Дубликат `code` → `IntegrityError` → 500.

3. **`AuthService.login` возвращает `(jsonify, code)`** — это нарушает чистоту сервисного слоя: сервис не должен знать про HTTP. Логичнее возвращать `(result, error)` или бросать доменные исключения, а ответ формировать во view.

4. **`db.session.rollback()`** вызывается только в `notes` и в глобальном обработчике 500. В CBV-views при ошибке (например, `IntegrityError`) rollback не делается явно — сессия может остаться в «грязном» состоянии до следующего запроса. Глобальный обработчик 500 спасает, но лучше делать rollback в `try/except` вокруг `commit()`.

5. **`DEBUG`-принты** в `sales_order_view.py` — не ошибка, но шум в логах.

### 3.1.3. Коды ответов

| Ситуация | Ожидаемо | Фактически | Проблема |
|----------|----------|------------|----------|
| Успешный GET | 200 | 200 | ✅ |
| Успешный POST (создание) | 201 | 201 | ✅ |
| Успешный PUT/DELETE | 200 | 200 | ✅ |
| Неверный JSON / нет полей | 400 | 400 | ✅ |
| Неверный формат UUID | 400 | 400 | ✅ |
| Неверный формат даты | 400 | **500** | ❌ |
| Нарушение CHECK (enum, диапазон) | 400 | **500** | ❌ |
| Дубликат UNIQUE | 400 или 409 | **500** | ❌ |
| Не найдено | 404 | 404 | ✅ |
| Не авторизован | 401 | 401 | ✅ |
| Не админ | 403 | 403 | ✅ |
| Удаление себя | 400 | 400 | ✅ |

**Вывод:** базовые коды корректны, но **нарушения БД и ошибки парсинга дают 500 вместо 400**. Это главная проблема маршрутов.

### 3.1.4. Сводная таблица замечаний по маршрутам

| № | Файл / маршрут | Проблема | Приоритет |
|---|----------------|----------|-----------|
| 1 | `views/production_order_view.py` POST/PUT | `date.fromisoformat` без try/except → 500 | Высокий |
| 2 | `views/sales_order_view.py` POST/PUT | `date.fromisoformat`, `float()` без try/except → 500 | Высокий |
| 3 | `views/clients.py` POST | Нет проверки `client_type` на допустимые значения → 500 при нарушении CHECK | Высокий |
| 4 | `views/materials.py`, `views/operations.py`, `views/products.py` PUT `/price` | Нет проверки `price` на число и `>= 0` → 500 | Высокий |
| 5 | `views/specifications.py` POST | Нет проверки `quantity_per_unit > 0`, `time_norm > 0`, `op_quantity > 0` → 500 | Средний |
| 6 | `views/materials.py`, `views/operations.py`, `views/products.py` POST | Дубликат `code` → 500 (ожидается 400/409) | Средний |
| 7 | `services/auth_service.py` | Сервис возвращает `jsonify` — нарушение слоёв | Средний |
| 8 | Все CBV при `IntegrityError` | Нет явного `db.session.rollback()` в `try/except` | Средний |
| 9 | `views/sales_order_view.py` | DEBUG-принты в продакшн-коде | Низкий |
| 10 | `blueprints/*.py` (legacy) | Не подключены, но содержат баги (например, `get_client` обновляет клиента) | Низкий (удалить) |

---

## 3.2. Ревью работы с базой данных (`database/`)

### 3.2.1. Файлы

- `database/schema.sql` — DDL + тестовые данные (PostgreSQL).
- `database/calculate_order_cost.sql` — 3 варианта расчёта стоимости (LATERAL-подзапросы + PL/pgSQL-функция).

### 3.2.2. Параметры запросов и SQL-инъекции

**ORM (SQLAlchemy):**  
Все запросы в `repositories/`, `services/`, `views/` выполняются через ORM (`db.select`, `db.session.execute(select(...))`) или через параметризованные запросы. **SQL-инъекции через ORM невозможны** при условии, что не используется `text()` с конкатенацией.

**Проблема: `text()` в `views/production_order_view.py` и `views/sales_order_view.py`:**

```python
result = db.session.execute(text(
    "SELECT order_number FROM production_orders WHERE order_number IS NOT NULL AND order_number != ''"
)).scalars().all()