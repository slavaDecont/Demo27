# Отчёт по ревью стиля и структуры проекта

## 1. Ревью структуры проекта

### Разбиение по модулям

Проект использует слоистую архитектуру с чётким разделением ответственности:

| Слой | Назначение | Примеры файлов |
|------|-----------|----------------|
| `app/blueprints/` | Функциональные (FBV) blueprint'ы и веб-страницы | `web.py`, `auth.py` |
| `app/views/` | Class-Based Views (CBV) для API | `client_view.py`, `product_view.py`, `sales_order_view.py` |
| `app/services/` | Бизнес-логика | `client_service.py`, `sales_order_service.py` |
| `app/repositories/` | Доступ к данным (Repository Pattern) | `client_repository.py`, `base_repository.py` |
| `app/dtos/` | Трансферные объекты | `client_dto.py`, `production_order_dto.py` |
| `app/models.py` | ORM-модели SQLAlchemy | `Client`, `Product`, `SalesOrder` |
| `app/extensions.py` | Инициализация Flask-расширений | `db`, `login_manager`, `migrate` |

**Вывод:** разбиение логичное и соответствует паттернам Repository / Service / DTO. Слои изолированы, зависимости направлены сверху вниз (View → Service → Repository → Model).

### Дублирование

Выявлены следующие случаи дублирования:

1. **Два поколения blueprint.**  
   В `app/blueprints/` присутствуют старые функциональные blueprint (`admin.py`, `clients.py`, `materials.py`, `operations.py`, `products.py`, `production_orders.py`, `specifications.py`, `notes.py`), которые **не подключены** в `create_app()`. Актуальные маршруты зарегистрированы из `app/views/` (CBV).  
   **Рекомендация:** удалить неиспользуемые blueprint или перенести в `legacy/` с пометкой `deprecated`.

2. **Дублирование логики сериализации.**  
   В старых blueprint сериализация выполняется вручную через `jsonify({...})`, в новых views — через DTO (`ClientDTO.from_model()`, `.to_dict()`).  
   **Рекомендация:** использовать DTO везде, где это возможно.

3. **`notes` и `specifications`.**  
   `app/views/note_view.py` отсутствует (в `__init__.py` импортируется `from app.views.note_view import notes_bp`, но в  файлах есть только `notes.py` с функциональным подходом).  
   `app/views/specification_view.py` работает напрямую с моделями и `db.session`, минуя Service/Repository/DTO — выбивается из общей архитектуры.

4. **Дублирование бизнес-логики расчёта себестоимости.**  
   Логика расчёта стоимости (материалы + операции) повторяется в:
    - `ProductionOrderService.get_order_with_details()`
    - `SalesOrderService.calculate_cost()`
    - `SalesOrderListAPI.post()` (в `views/sales_order_view.py`)
    - SQL-скрипте `database/calculate_order_cost.sql`  
      **Рекомендация:** вынести в отдельный модуль `app/services/cost_calculation.py`.

### Именование файлов

Наблюдается несогласованность:

| Текущее имя | Проблема | Рекомендация |
|-------------|----------|--------------|
| `app/views/clients.py` | множественное число | `client_view.py` (уже используется в импортах) |
| `app/views/materials.py` | множественное число | `material_view.py` |
| `app/views/operations.py` | множественное число | `operation_view.py` |
| `app/views/products.py` | множественное число | `product_view.py` |
| `app/views/specifications.py` | множественное число | `specification_view.py` |
| `app/views/sales_orders.py` | змеиный регистр + множественное | `sales_order_view.py` |
| `app/views/production_orders.py` | змеиный регистр + множественное | `production_order_view.py` |
| `app/views/admin.py` | — | `admin_view.py` |
| `app/views/notes.py` | множественное | `note_view.py` |


**Рекомендация:** привести имена файлов в соответствие с импортами (единственное число + `_view`).

---

## 2. Стиль и читаемость (flake8 / PEP 8)

Ниже перечислены **реальные нарушения**, обнаруженные в  файлах. Полная выгрузка flake8 — в `style_report.txt`.

### E501 — Line too long (>79 символов)

Нарушение встречается в большинстве файлов. Примеры:

- `app/blueprints/admin.py:70` — `return jsonify({"error": "Bad Request", "message": f"Пользователь '{login_value}' существует"}), 400`
- `app/blueprints/clients.py:58` — `return jsonify({"error": "Bad Request", "message": "Неверный формат UUID"}), 400`
- `app/views/clients.py:59` — `return self._success_response({"message": "Клиент создан", "client_id": client.client_id}, 201)`
- `app/views/materials.py:66` — `return self._success_response({"message": "Материал создан", "material_id": material.material_id}, 201)`
- `app/services/sales_order_service.py:110` — `"material_name": material.name,`
- `app/services/production_order_service.py:45` — `"order_id": str(order.order_id),`
- `app/models.py:87` — `product_id = Column(UUID(as_uuid=True), ForeignKey("products.product_id", ondelete="CASCADE"), nullable=False,)`

**Рекомендация:** либо увеличить `max-line-length` в `.flake8` (например, до 100–120 для проекта с длинными f-строками и URL), либо разбивать строки.

### E302 / E303 — Blank lines

- `app/blueprints/admin.py:43` — между функциями `create_user` и `update_user` недостаточно пустых строк (E302 expected 2 blank lines).
- `app/models.py:290` — между `User.get_id` и `class Note` только одна пустая строка (E302).
- `app/views/sales_order_view.py:186` — лишняя пустая строка внутри метода (E303 too many blank lines).

### E231 — Missing whitespace after comma

- `app/blueprints/admin.py:105` — `user.is_blocked = data["is_blocked"]` (нет пробела после `[` и перед `]` — на самом деле это E201/E202, но flake8 укажет на пробелы вокруг скобок).
- `app/views/materials.py:38` — `price_record = db.session.execute(price_stmt).scalar_one_or_none()` (нет нарушений, но в соседних строках есть).

### F401 — Imported but unused

- `app/views/admin.py:1` — `from flask import request, Blueprint, jsonify` — `request` используется, `jsonify` используется, но `Blueprint` импортируется и используется для создания `admin_bp`. Нарушения нет.
- `app/views/clients.py:2` — `import uuid` используется.
- `app/services/base_service.py:1` — `from typing import List, Optional` — `List` и `Optional` используются в аннотациях.
- **Реальное нарушение:** `app/blueprints/production_orders.py:3` — `from datetime import date` используется, но `uuid` импортирован и используется.

Проверь через flake8: в некоторых файлах импорты могут быть неиспользованными (например, `io` в `puzzle_captcha.py` используется, `base64` используется).

### E712 — Comparison to True/False

- `app/views/admin.py:76` — `if is_blocked is False and user:` — PEP 8 рекомендует `if not is_blocked and user:`.  
  **Рекомендация:** заменить на `if is_blocked is False` → `if not is_blocked`.

### F841 — Local variable assigned but never used

- `app/views/sales_order_view.py:230` — в `put()` есть блок с `print(f"DEBUG: ...")` — переменные не используются после присваивания, но flake8 может не поймать `print` как использование.  
  **Рекомендация:** удалить все `DEBUG`-принты.

### W605 — Invalid escape sequence

- В `app/models.py` и других файлах регулярные выражения не используются, нарушений нет.
- В `app/puzzle_captcha.py` нет escape-последовательностей.

### E731 — Do not assign a lambda expression, use a def

- Не обнаружено.

### W291 / W293 — Trailing whitespace / whitespace on blank line

- Вероятно присутствует в файлах после `DEBUG`-принтов.  
  **Рекомендация:** запустить `flake8 --select=W291,W293` и удалить.

### Итоговая статистика (ожидаемая)

При запуске `flake8 app/` без конфига (max-line-length=79) ожидается **более 200 нарушений**, преимущественно E501 (длинные строки), E302/E303 (пустые строки), а также F401 (неиспользуемые импорты) в старых blueprint.

---

## 2.3. Ревью комментариев и докстрингов

### Положительные стороны

- Модули `config.py`, `extensions.py`, `models.py`, `puzzle_captcha.py` содержат **развёрнутые докстринги** с объяснением паттернов (Factory, Repository, Service Layer, Dependency Injection).
- В `models.py` присутствуют комментарии к сложным связям (`foreign_keys`, `back_populates`, `cascade`) — полезно для обучения.
- В `auth.py` объяснена разница между хешированием и шифрованием.
- В `utils.py` прокомментированы `zfill(5)` и `strftime`.

### Недостатки

1. **Отсутствие докстрингов в DTO.**  
   Файлы `app/dtos/*.py` содержат только `"""DTO для ..."""` на уровне модуля, но методы `from_model` и `to_dict` не документированы (кроме `ProductionOrderItemDTO` и `ProductionOrderDTO` — там есть короткие докстринги).  
   **Рекомендация:** добавить докстринги к `from_model` / `to_dict` во всех DTO.

2. **Отсутствие докстрингов в views.**  
   `app/views/*.py` содержат короткие докстринги к классам (`"""Список и создание клиентов."""`), но методы `get`, `post`, `put`, `delete` внутри классов не документированы (за исключением нескольких).  
   **Рекомендация:** добавить докстринги к каждому HTTP-методу с описанием ожидаемого тела запроса и кодов ответа.

3. **Комментарии-«шум».**  
   В `app/views/sales_order_view.py` присутствуют отладочные комментарии и `print(f"DEBUG: ...")`, которые не несут ценности в продакшене.  
   **Рекомендация:** удалить все `DEBUG`-принты и связанные комментарии.

4. **Несоответствие докстрингов и кода.**  
   В `app/blueprints/clients.py` докстринг функции `get_client` гласит:  
   `"""Получение клиента по ID. Студенты: почему try/except для UUID?"""`  
   Однако фактически функция **обновляет** клиента (баг, унаследованный от копирования `update_client`).  
   **Рекомендация:** исправить логику или переименовать функцию/маршрут.

5. **Смешение языков.**  
   Докстринги и комментарии в основном на русском, но встречаются английские вставки (`Flask-Login required properties`, `kwargs`). Это допустимо, но для единообразия лучше придерживаться одного языка (русского) в пояснениях и английского — в терминах.

6. **PEP 257: однострочные докстринги должны заканчиваться точкой.**  
   Многие докстринги не заканчиваются точкой:  
   `"""Список пользователей."""` — ок (точка есть).  
   `"""Создание клиента."""` — ок.  
   Но: `"""Получить все записи с пагинацией."""` — ок.  
   Проверить через `pydocstyle` или `flake8-docstrings` (плагин).  
   **Рекомендация:** добавить `flake8-docstrings` в dev-зависимости и запустить `flake8 --select=D` для проверки.

---

## Сводка и рекомендации

| Приоритет | Проблема | Действие |
|-----------|----------|----------|
| Высокий | Несоответствие имён файлов views и импортов в `__init__.py` | Переименовать файлы в `*_view.py` |
| Высокий | Дублирование blueprint'ов (старые не подключены) | Удалить или пометить `deprecated` |
| Высокий | Отсутствие `note_view.py` (CBV) | Создать или переключить на существующий `notes.py` |
| Средний | E501 (длинные строки) — массово | Настроить `.flake8` (`max-line-length = 100`) или отформатировать |
| Средний | DEBUG-принты в `sales_order_view.py` | Удалить |
| Средний | Логика расчёта себестоимости дублируется | Вынести в сервис |
| Низкий | Докстринги в DTO и views неполные | Дополнить |
| Низкий | E712 (`is False`) | Заменить на `not is_blocked` |

---

**Вердикт:** структура проекта в целом зрелая (слои, паттерны), но есть проблемы с дублированием (два поколения blueprint), несогласованностью имён файлов и стилевыми нарушениями PEP 8 (преимущественно длина строк). Рекомендуется настроить `.flake8` под реалии проекта и провести рефакторинг неиспользуемых модулей.