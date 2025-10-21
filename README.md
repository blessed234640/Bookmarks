# Bookmarks

## Описание

Этот проект представляет собой социальный веб-сайт на Django для закладок (bookmarks), где пользователи могут делиться изображениями, следить друг за другом, просматривать активности и взаимодействовать асинхронно. Он включает аутентификацию (включая социальную через Google), профили пользователей, систему follow, activity stream, bookmarklet для добавления контента с других сайтов, thumbnails, infinite scroll, трекинг просмотров с Redis и оптимизации производительности.

Проект демонстрирует продвинутые возможности Django: кастомные бэкенды аутентификации, generic relations, signals для денормализации, JavaScript для асинхронных действий, Redis для кэша и ранжирования, а также инструменты вроде Django Debug Toolbar.

## Требования

- Python 3.12+
- Django 5+
- Дополнительные библиотеки: Pillow (для изображений), Requests (для HTTP), easy-thumbnails (для миниатюр), Redis (для кэша), django-debug-toolbar (для отладки)
- Виртуальное окружение
- Для HTTPS в dev: certutil или аналогичные инструменты

## Установка

1. **Клонируйте репозиторий**:
   ```
   git clone https://github.com/blessed234640/Bookmarks.git
   cd Bookmarks
   ```

2. **Создание виртуального окружения**:
   ```
   python -m venv env
   source env/bin/activate  # Для Unix/Mac
   env\Scripts\activate     # Для Windows
   ```

3. **Установка зависимостей** (создайте requirements.txt, если нужно):
   ```
   pip install django pillow requests easy-thumbnails redis django-debug-toolbar
   ```
   (Дополните другими из проекта, как social-auth-app-django для Google).

4. **Применение миграций**:
   ```
   python manage.py makemigrations
   python manage.py migrate
   ```

5. **Создание суперюзера**:
   ```
   python manage.py createsuperuser
   ```

6. **Настройка Redis** (установите Redis сервер):
   - В settings.py добавьте конфигурацию для Redis (как кэш или для Celery, если используется).
   - Запустите Redis: `redis-server`.

7. **Запуск сервера** (для HTTPS в dev используйте runserver_plus из django-extensions):
   ```
   python manage.py runserver
   ```
   Доступно по http://127.0.0.1:8000/ (или HTTPS для тестов социального логина).

## Функциональность

- **Аутентификация и профили**: Логин/логаут, смена/сброс пароля с использованием встроенных views Django. Регистрация пользователей, кастомный User модель или профиль с полями (например, фото с Pillow). Кастомный backend для email-авторизации, предотвращение дубликатов email. Социальная аутентификация через Google (с созданием профиля для новых пользователей). Фреймворк сообщений для уведомлений.
- **Шаринг контента**: Модель Image с many-to-many (для пользователей, лайков). Bookmarklet (JavaScript) для добавления изображений с других сайтов (очистка форм, fetch с Requests, переопределение save() в ModelForm). Детальный view изображений, thumbnails с easy-thumbnails. Асинхронные действия (лайки/unlike) с JavaScript (fetch, CSRF). Infinite scroll для списка изображений.
- **Follow-система и активности**: Many-to-many с intermediate моделью для follow/unfollow (асинхронно с JS). Профили пользователей с list/detail views. Activity stream с contenttypes (generic relations), избежание дубликатов, оптимизация QuerySets (select_related/prefetch_related). Шаблоны для действий, signals для денормализации счётчиков (ready() в AppConfig).
- **Трекинг и отладка**: Redis для подсчёта просмотров изображений и ранжирования (zset для топа). Django Debug Toolbar для анализа (панели, команды). HTTPS в dev для тестов.

## Визуальная архитектура

Архитектура в виде flowchart (request/response cycle в Django MVT), с акцентом на социальные функции, аутентификацию и Redis.

```mermaid
flowchart TD
    A["Клиент: HTTP-запрос (логин, шаринг, follow)"] --> B{"Middleware: Аутентификация, сессии, CSRF (settings.py)"}
    B --> C{"URL-резолвер: mysite/urls.py (include account/urls.py)"}
    C -->|"Паттерны: login/, images/, users/"| D["Views: Кастомные (LoginView, ImageCreateView) или CBV (ListView)"}
    D -->|"Данные: QuerySets с оптимизацией (select/prefetch)"| E["Models: User/Profile, Image (M2M), Action (GenericRelation)"}
    E -->|"Миграции, индексы"| F["Migrations: account/migrations/"]
    F --> E
    D -->|"Формы: ModelForm (ImageForm с clean/save override)"| G["Forms: account/forms.py (UserRegistrationForm)"}
    G --> D
    D -->|"Асинхронно: JS fetch для like/follow"| H["JS: Bookmarklet, infinite scroll (templates/static/js)"}
    H --> D
    D -->|"Рендеринг: base.html, image/detail.html"| I["Templates: account/templates/ (with {% extends %})"]
    I -->|"HTTP-ответ: с thumbnails (easy-thumbnails)"| J["Клиент: Обновление UI"]
    subgraph "Аутентификация"
        D -->|"Social: Google backend"| K["Custom Backend: authenticate() с email"]
        K -->|"Messages framework для уведомлений"| L["Messages: success/error"]
        L --> I
    end
    subgraph "Activity & Follow"
        E -->|"Signals: post_save для actions/counts"| M["Signals: denormalize counts (AppConfig ready())"]
        M --> E
        D -->|"Follow/Unfollow: JS actions"| N["Views: user_follow()"]
        N --> E
        E -->|"Activity stream: contenttypes"| O["Generic relations in Action model"]
        O --> I
    end
    subgraph "Redis & Отладка"
        D -->|"Просмотры/Ранг: incr/zadd"| P["Redis: для image views/ranking (python-redis)"}
        P --> D
        B -->|"Debug: Toolbar middleware"| Q["Django Debug Toolbar: panels/SQL"]
        Q --> I
    end
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style J fill:#f9f,stroke:#333,stroke-width:2px
```

Эта диаграмма показывает поток: от запроса через middleware/URL к views/models, с интеграцией форм, JS, signals, Redis и отладкой.

## Использование

- **Админ-панель**: http://127.0.0.1:8000/admin/ (регистрируйте модели в admin.py).
- **Регистрация/Логин**: /account/register/, /account/login/ (с социальным через Google).
- **Добавление изображений**: Bookmarklet (drag to bookmarks bar) или форма /images/create/.
- **Профили**: /users/<username>/ (с follow/unfollow).
- **Activity stream**: /account/ (или dedicated view).
- **Infinite scroll**: В списке изображений (/images/).
- **Ранжирование**: Топ изображений по просмотрам (view с Redis).

Для HTTPS в dev: `python manage.py runserver_plus --cert-file cert.crt`.

## Дополнительные ресурсы

- Документация Django: https://docs.djangoproject.com/
- easy-thumbnails: https://github.com/SmileyChris/easy-thumbnails
- Redis: https://redis.io/docs/connect/clients/python/
- Django Debug Toolbar: https://django-debug-toolbar.readthedocs.io/
