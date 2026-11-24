# Производственная ИС (ПИС)

Информационная система для управления производственным предприятием: ведение справочников (клиенты, продукция, материалы, операции), спецификаций, заказов покупателей и заказов на производство, расчёт себестоимости, управление пользователями и заметками.

**Стек:** Python 3.10+, Flask 3.0, SQLAlchemy 2.0, PostgreSQL 14+, Jinja2, Pillow.

---

## Содержание

- [Требования](#требования)
- [Установка](#установка)
- [Настройка окружения](#настройка-окружения)
- [Инициализация базы данных (пошагово)](#инициализация-базы-данных-пошагово)
- [Запуск приложения](#запуск-приложения)
- [Запуск тестов](#запуск-тестов)
- [Линтинг и аудит](#линтинг-и-аудит)


---

## Требования

- **Python** 3.10 или выше
- **PostgreSQL** 14 или выше
- **pip** и **virtualenv** (или `venv`)

---

## Установка

1. Клонируйте репозиторий:

   ```bash
   git clone <url-репозитория>
   cd production-is
   
---
## Настройка окружения
# Скопируйте пример конфигурации:

cp .env.example .env
Отредактируйте .env, указав свои значения: env

# Flask
FLASK_ENV=development
SECRET_KEY=change-me-to-a-long-random-string

# PostgreSQL
DATABASE_URL=postgresql://postgres:password@localhost:5432/production_db

## Инициализация базы данных (пошагово)
### Шаг 1. Создать базу данных в PostgreSQL
Через psql:

sudo -u postgres psql
В консоли psql:

sql

CREATE DATABASE production_db;
\q
Или одной командой:

createdb -U postgres production_db
### Шаг 2. Указать строку подключения в .env

DATABASE_URL=postgresql://postgres:password@localhost:5432/production_db
Убедитесь, что пользователь postgres имеет права на создание таблиц.

### Шаг 3. Применить схему БД
Вариант A — через SQL-скрипт (рекомендуется для первого запуска):

psql -U postgres -d production_db -f database/schema.sql
Скрипт создаёт:

- все таблицы (clients, products, materials, operations, product_prices, material_prices, operation_prices, specification_materials, specification_operations, production_orders, production_order_items, sales_orders, sales_order_items, users, notes);

- индексы;

- CHECK-ограничения;

- тестовые данные (продукт «Стол кухонный Самобранка», материалы, операции, цены, спецификации, пользователи, заметки).

Вариант B — через миграции (для разработки):


- flask db init       # только при первом запуске
- flask db migrate -m "Initial migration"
- flask db upgrade
⚠️ Для продакшена используйте только миграции, а не db.create_all().

### Шаг 4. Проверить подключение

psql -U postgres -d production_db -c "\dt"
Должны отобразиться все таблицы.

### Шаг 5. (Опционально) Создать администратора
Если в database/schema.sql тестовые пароли — заглушки (dummyhash), создайте реального администратора:


python scripts/create_admin.py

Или вручную через flask shell:

python
from app.extensions import db
from app.models import User
from app.auth import hash_password

admin = User(
login="admin",
password_hash=hash_password("admin123"),
role="Администратор",
)
db.session.add(admin)
db.session.commit()
⚠️ Смените пароль после первого входа.

### Шаг 6. Проверить, что БД доступна из приложения

flask shell
или 
from app.extensions import db
db.engine.connect()

## Запуск приложения
### Вариант 1 — через flask run
Убедитесь, что переменная FLASK_APP указывает на фабрику:


export FLASK_APP=app:create_app    # Linux/macOS

 set FLASK_APP=app:create_app       # Windows (cmd)

 $env:FLASK_APP="app:create_app"    # Windows (PowerShell)
Запуск:

flask run
### Вариант 2 — через точку входа run.py

python run.py
### Вариант 3 — в режиме разработки с отладкой

flask run --debug

## Запуск тестов

pytest

С покрытием:


pytest --cov=app --cov-report=term-missing

## Линтинг и аудит
Стиль кода (flake8):


flake8 app/

С сохранением в файл:


flake8 app/ --output-file=docs/review/style_report.txt --max-line-length=120

Докстринги (pydocstyle):


pydocstyle app/

Аудит уязвимостей:


pip-audit -r requirements.txt


