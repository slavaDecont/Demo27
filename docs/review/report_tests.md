# Отчёт по ревью тестов



---

## 1. Структура тестов

| Файл | Назначение | Кол-во тестов |
|------|-----------|---------------|
| `conftest.py` | Настройка окружения: `DATABASE_URL=sqlite:///:memory:`, добавление корня проекта в `sys.path` | — |
| `test_auth.py` | Тесты авторизации и блокировки (Задание 4, TZ.md) | 7 |
| `test_notes.py` | Тесты API заметок: формат, статусы, пагинация (Задание 5, API.md) | 7 |
| **Итого** | | **14** |

---

## 2. Покрытие ключевых сценариев

### 2.1. Авторизация (`test_auth.py`)

| Сценарий | Тест | Покрыт | Комментарий |
|----------|------|--------|-------------|
| Успешная авторизация | `test_login_success` | ✅ | Проверяет 200, `message`, `login`, `role` |
| Неверный пароль | `test_login_wrong_password` | ✅ | Проверяет 401 и текст ошибки |
| Несуществующий пользователь | `test_login_nonexistent_user` | ✅ | Проверяет 401 |
| Блокировка после 3 неверных попыток | `test_account_blocking` | ✅ | 3 неверных + 1 верный → 403 |
| Отсутствие обязательных полей | `test_login_missing_fields` | ✅ | Пустой JSON → 400 |
| `/api/auth/me` без авторизации | `test_get_current_user_unauthenticated` | ✅ | 401 |
| `/api/auth/me` после авторизации | `test_get_current_user_authenticated` | ✅ | 200, `login`, `role` |

**Оценка:** сценарии авторизации покрыты **полностью** в рамках ТЗ.

### 2.2. Заметки (`test_notes.py`)

| Сценарий | Тест | Покрыт | Комментарий |
|----------|------|--------|-------------|
| Список заметок → 200 | `test_get_notes_returns_200` | ✅ | Проверяет `notes`, `total` |
| Формат ответа по API.md | `test_get_notes_format` | ✅ | `id` — 5 цифр, `title_user` с `" - "`, `formatted_date` — ДД.ММ.ГГГГ |
| Пустая таблица | `test_get_notes_empty` | ✅ | `notes=[]`, `total=0` |
| Неверные `skip`/`limit` | `test_get_notes_invalid_params` | ✅ | `skip=-1`, `limit=0` → 400 |
| JSON-тип ответа | `test_get_notes_json_valid` | ✅ | `content_type == "application/json"` |
| Одна заметка по ID | `test_get_notes_single` | ✅ | 200, `id == f"{note_id:05d}"` |
| Несуществующая заметка | `test_get_notes_not_found` | ✅ | 404 |

**Оценка:** API заметок покрыто **полностью** в рамках ТЗ (GET-операции).

### 2.3. Непокрытые сценарии

| Область | Сценарии | Покрыт |
|---------|----------|--------|
| **Клиенты** | GET список, GET по ID, POST, PUT, DELETE | ❌ |
| **Материалы** | CRUD + PUT `/price` | ❌ |
| **Операции** | CRUD + PUT `/price` | ❌ |
| **Продукция** | CRUD + GET/PUT `/price` | ❌ |
| **Спецификации материалов** | GET, POST | ❌ |
| **Спецификации операций** | GET, POST | ❌ |
| **Заказы покупателя** | CRUD, `/cost-calculation`, `/production-order` | ❌ |
| **Заказы на производство** | CRUD, автогенерация номера | ❌ |
| **Админ-панель** | Создание, блокировка, удаление пользователей | ❌ |
| **Права доступа** | 403 для не-админа | ❌ |
| **Расчёт себестоимости** | `calculate_cost` | ❌ |
| **Капча** | generate, verify | ❌ |
| **Создание заметки** | POST `/api/notes/` (в `web.py`) | ❌ |
| **Валидация дат** | Неверный формат `order_date` | ❌ |
| **Валидация enum** | Неверный `client_type`, `status` | ❌ |
| **Валидация диапазонов** | Отрицательные `price`, `quantity` | ❌ |
| **Обработка IntegrityError** | Дубликат `code`, UNIQUE violation | ❌ |

**Оценка:** покрыты **2 модуля из 10+**. Основная бизнес-логика (заказы, расчёты, спецификации) **не тестируется**.

---

## 3. Запускаемость тестов

### 3.1. Критические блокеры

#### Блокер 1. Несовместимость моделей с SQLite

**Проблема:**  
В `app/models.py` используется:

```python
from sqlalchemy.dialects.postgresql import UUID
...
client_id = Column(UUID(as_uuid=True), primary_key=True, default=func.gen_random_uuid())