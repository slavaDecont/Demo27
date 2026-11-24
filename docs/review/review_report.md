# Итоговый отчёт по Code Review

---

## 5.1. Сводная таблица замечаний

**Легенда приоритетов:**
- 🔴 **Критично** — блокирует работу, уязвимость, потеря данных. Исправить немедленно.
- 🟠 **Важно** — серьёзный баг или риск безопасности. Исправить в ближайший спринт.
- 🟢 **Рекомендация** — улучшение поддерживаемости, стиля, UX. По возможности.

### Безопасность

| № | Приоритет | Файл | Замечание | Предложение по исправлению |
|---|-----------|------|-----------|----------------------------|
| 1 | 🔴 Критично | `app/config.py` | Дефолтный `SECRET_KEY` — предсказуемый ключ | `raise RuntimeError`, если `SECRET_KEY` не задан в env |
| 2 | 🔴 Критично | `app/extensions.py`, `app/__init__.py` | `CSRFProtect` не подключён | Добавить `Flask-WTF`, `csrf.init_app(app)` |
| 3 | 🔴 Критично | `web/*.html` | `innerHTML` с пользовательскими данными → XSS | Заменить на `textContent` / `createElement` |
| 4 | 🔴 Критично | `app/blueprints/web.py` | `captcha_ok = True` — капча отключена | Реализовать проверку `puzzle_captcha.verify()` |
| 5 | 🔴 Критично | `app/views/*.py` | `/api/*` без `@login_required` | Добавить декораторы на все CBV |
| 6 | 🔴 Критично | `web/login.html` | `token` капчи не передаётся в форму | `fetch('/api/puzzle-captcha/generate')` → скрытое поле |
| 7 | 🟠 Важно | `app/config.py` | Дефолтный пароль БД `"12345"` | `raise RuntimeError`, если `DATABASE_PASSWORD` не задан |
| 8 | 🟠 Важно | `app/blueprints/web.py` | Path traversal в `/image/<filename>` | `filepath.resolve().is_relative_to(IMAGE_FOLDER.resolve())` |
| 9 | 🟠 Важно | `web/base-dashboard.html` | `showNotification` — `innerHTML` с `result.message` | Использовать `textContent` |
| 10 | 🟡 Рекомендация | `app/config.py` | Cookie-флаги (`HTTPONLY`, `SAMESITE`) не заданы | `SESSION_COOKIE_HTTPONLY = True`, `SESSION_COOKIE_SAMESITE = "Lax"` |

### Логика и маршруты

| № | Приоритет | Файл | Замечание | Предложение по исправлению |
|---|-----------|------|-----------|----------------------------|
| 11 | 🟠 Важно | `app/views/production_order_view.py` | `date.fromisoformat` без `try/except` → 500 | Обернуть в `try/except` → 400 |
| 12 | 🟠 Важно | `app/views/sales_order_view.py` | `date.fromisoformat`, `float()` без `try/except` → 500 | Аналогично |
| 13 | 🟠 Важно | `app/views/client_view.py` | Нет проверки `client_type` (enum) | Валидация `client_type in ("Поставщик", "Покупатель")` → 400 |
| 14 | 🟠 Важно | `app/views/material_view.py`, `operation_view.py`, `product_view.py` | PUT `/price` без проверки `price >= 0` | Валидация → 400 |
| 15 | 🟡 Рекомендация | `app/views/specification_view.py` | Нет проверки `quantity_per_unit > 0`, `time_norm > 0` | Валидация → 400 |
| 16 | 🟡 Рекомендация | все CBV | Нет `db.session.rollback()` при `IntegrityError` | `try/except IntegrityError: db.session.rollback()` |
| 17 | 🟡 Рекомендация | `app/services/auth_service.py` | Сервис возвращает `jsonify` | Возвращать dict, `jsonify` — во view |

### База данных

| № | Приоритет | Файл | Замечание | Предложение по исправлению |
|---|-----------|------|-----------|----------------------------|
| 18 | 🔴 Критично | `database/schema.sql` | Нет `production_orders.sales_order_id` | `ALTER TABLE ... ADD COLUMN sales_order_id UUID` |
| 19 | 🔴 Критично | `app/models.py` | `UUID`/`gen_random_uuid()` несовместимы с SQLite | TypeDecorator `GUID` + `default=uuid.uuid4` |
| 20 | 🟠 Важно | `database/calculate_order_cost.sql` | `discount` — абсолют, в Python — % | Привести к % (единая трактовка) |
| 21 | 🟠 Важно | `app/views/production_order_view.py`, `sales_order_view.py` | Генерация номера через `text()` → гонка | Sequence / `FOR UPDATE` |
| 22 | 🟡 Рекомендация | `database/schema.sql` | Нет индекса на `sales_order_id` | `CREATE INDEX idx_prod_orders_sales` |

### Тесты

| № | Приоритет | Файл | Замечание | Предложение по исправлению |
|---|-----------|------|-----------|----------------------------|
| 23 | 🔴 Критично | `app/blueprints/auth.py` | Файл отсутствует | Создать с `/login`, `/me` |
| 24 | 🟠 Важно | `app/services/auth_service.py` | Сообщение «Авторизация успешна» ≠ тесту | Заменить на «Вы успешно авторизовались» |
| 25 | 🟠 Важно | `tests/` | Покрыты только auth и notes | Добавить тесты для clients, orders, specifications |
| 26 | 🟡 Рекомендация | `app/__init__.py` | `db.create_all()` в `create_app()` | Пропускать при `TESTING=True` |

### Стиль и структура

| № | Приоритет | Файл | Замечание | Предложение по исправлению |
|---|-----------|------|-----------|----------------------------|
| 27 | 🟠 Важно | `app/views/*.py` | Имена файлов ≠ импортам в `__init__.py` | Переименовать в `*_view.py` |
| 28 | 🟠 Важно | `app/blueprints/` | Legacy-модули не подключены | Удалить или перенести в `legacy/` |
| 29 | 🟠 Важно | `app/views/note_view.py` | Отсутствует | Создать CBV для заметок |
| 30 | 🟡 Рекомендация | `app/services/*.py` | Дублирование расчёта себестоимости | Вынести в `cost_calculation.py` |
| 31 | 🟡 Рекомендация | `app/` | E501 (длинные строки) — массово | Настроить `.flake8` (`max-line-length = 120`) |
| 32 | 🟡 Рекомендация | `app/services/*.py`, `app/views/*.py` | F401 (неиспользуемые импорты) | Удалить |
| 33 | 🟢 Рекомендация | `app/services/*.py`, `app/views/*.py` | W293 (whitespace в пустых строках) | `flake8 --select=W293` + автофикс |
| 34 | 🟢 Рекомендация | `app/dtos/*.py`, `app/views/*.py` | Неполные докстринги | Дополнить |

### Фронтенд

| № | Приоритет | Файл | Замечание | Предложение по исправлению |
|---|-----------|------|-----------|----------------------------|
| 35 | 🟠 Важно | `web/materials.html`, `web/operations.html` | Создание не работает (`PUT` вместо `POST`) | `method: clientId ? 'PUT' : 'POST'` |
| 36 | 🟠 Важно | `web/production-orders.html` | Форма не отправляется | Добавить кнопку «Сохранить» в `modal-footer` |
| 37 | 🟠 Важно | `web/production-orders.html` | HTML невалиден (незакрытый `<div>`) | Исправить вложенность |
| 38 | 🟠 Важно | `web/clients.html` | `editClient(${JSON.stringify(client)})` | Передавать `client_id`, брать объект из `clientsList` |
| 39 | 🟠 Важно | `web/admin-dashboard.html` | `editUser(..., '${user.login}', ...)` | Аналогично — передавать `user_id` |
| 40 | 🟡 Рекомендация | `web/*.html` | Кнопки без `aria-label` | Добавить `aria-label="Редактировать"` и т.д. |
| 41 | 🟢 Рекомендация | `web/base-dashboard.html`, `login.html` | Инлайн `<style>` (400+ / 150+ строк) | Вынести в `styles.css` |
| 42 | 🟢 Рекомендация | `web/*.html` | Неинформативные тексты («Попытки», «Клиент ID») | Заменить на «Неудачные попытки», «Клиент» |

### Зависимости

| № | Приоритет | Файл | Замечание | Предложение по исправлению |
|---|-----------|------|-----------|----------------------------|
| 43 | 🟠 Важно | `requirements.txt` | Prod-версии не зафиксированы (`>=`) | Заменить на `==` |
| 44 | 🟠 Важно | `requirements.txt` | Нет `flask-wtf` | Добавить `Flask-WTF==1.2.1` |
| 45 | 🟠 Важно | `requirements.txt` | Минимальные версии допускают CVE | Поднять до `Flask>=3.0.3`, `Werkzeug>=3.0.3`, `Pillow>=10.3.0` |
| 46 | 🟡 Рекомендация | `requirements.txt` | Prod и dev смешаны | Разделить на `requirements.txt` + `requirements-dev.txt` |
| 47 | 🟢 Рекомендация | `requirements.txt` | `alembic` избыточен | Убрать (транзитивно от `Flask-Migrate`) |

### Документация

| №  | Приоритет | Файл | Замечание | Предложение по исправлению |
|----|-----------|------|-----------|----------------------------|
| 48 | 🟠 Важно | `README.md` | Нет блоков «Структура», «Архитектура», «API», «Ограничения» | Дополнить (см. `docs_review.md`) |
| 49 | 🟠 Важно | `docs/ER-diagram.mmd` | Рассинхрон с `schema.sql` (нет `sales_order_id`) | Синхронизировать после п. 18 |
| 50 | 🟠 Важно | корень | Нет `.env.example` | Создать |
| 51 | 🟡 Рекомендация | корень | Нет `requirements-dev.txt` | Создать |
| 52 | 🟡 Рекомендация | `docs/review/` | Нет `README.md` с навигацией | Создать |
| 53 | 🟡 Рекомендация | `report_dbase.txt` | Формат `.txt` вместо `.md` | Переименовать в `db_report.md` |

**Итого:** 10 критично / 24 важно / 21 рекомендация (всего 55 замечаний).

---

## 5.2. Приоритетные правки (3–5) в отдельных коммитах

Выбраны **5 критических** проблем, которые нужно исправить первыми. Каждая — отдельный коммит с префиксом `fix:`.

### Коммит 5: `docs: add review_report with consolidated findings`

**Что:** добавлен итоговый `docs/review/review_report.md` с таблицей замечаний и приоритетами.

```bash
git add docs/review/review_report.md
git commit -m "docs: add consolidated review report with priorities"