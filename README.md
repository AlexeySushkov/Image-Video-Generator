# Generate Images Bot v5 + DB + start

Telegram-бот для генерации и редактирования изображений через n8n workflow, с хранением пользователей, истории и файлов в Supabase.

## Архитектура

```
Telegram Bot  →  n8n Workflow  →  OpenRouter API (image models)
                     ↕
               Supabase (Postgres + Storage)
```

## Что нового в v5

- Добавлен workflow-файл `Generate Images Bot v5 + DB + start.json`
- `/start` автоматически:
  - создает пользователя в `ImageUsers` (если его нет)
  - выставляет команды бота через Telegram API (`setMyCommands`)
- Добавлен выбор модели генерации через inline-кнопки меню
- Обновлена схема БД: используются таблицы `ImageUsers` и `ImageHistory`
- После генерации отправляется и `photo`, и исходный файл (`document`) без сжатия

## Возможности

- Генерация изображений по текстовому промпту
- Редактирование с контекстом из ранее загруженных изображений
- Поддержка мультиконтекста (несколько изображений в активной сессии)
- Очистка контекста (`/clear_context` и кнопка `Новая генерация`)
- Выбор модели прямо в меню
- Сохранение контекстных и итоговых изображений в Supabase Storage (`photos`)
- История промптов и usage в БД

## Команды бота

| Команда | Описание |
|---------|----------|
| `/start` | Регистрация/проверка пользователя + установка команд бота |
| `/menu` | Показ главного меню |
| `/clear_context` | Очистка активного контекста |
| `/help` | Справка *(пока заглушка)* |

## Inline-кнопки

| Кнопка | Действие |
|--------|----------|
| Новая генерация | Очищает активный контекст |
| Продолжить редактирование | Убирает меню и предлагает продолжить |
| Nano Banana | Переключает на `google/gemini-2.5-flash-image` |
| Nano Banana Pro | Переключает на `google/gemini-3-pro-image-preview` |
| ChatGPT mini | Переключает на `openai/gpt-5-image-mini` |
| Fast | Переключает на `black-forest-labs/flux.2-klein-4b` |

## Используемые AI-модели

| Модель | Назначение |
|--------|-----------|
| `openai/gpt-5-image-mini` | Быстрая и недорогая генерация |
| `google/gemini-2.5-flash-image` | Качественная универсальная генерация |
| `google/gemini-3-pro-image-preview` | Продвинутая генерация/редактирование |
| `black-forest-labs/flux.2-klein-4b` | Очень быстрый режим (default для новых пользователей) |

## Структура базы данных (Supabase)

### Таблица `ImageUsers`

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | Первичный ключ |
| `telegram_id` | string | Telegram ID пользователя |
| `user_name` | string | Username пользователя |
| `last_sign_in_at` | timestamp/string | Время последнего `/start` |
| `model` | string | Текущая выбранная модель |

### Таблица `ImageHistory`

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | Первичный ключ |
| `user_id` | int | FK → `ImageUsers.id` |
| `prompt` | string | Промпт пользователя |
| `image_id` | string | ID объекта в Supabase Storage |
| `image_name` | string | Key объекта в bucket `photos` |
| `image_status` | string | `active` или `cleared` |
| `image_type` | string | `context` или `result` |
| `usage` | json/string | Usage от OpenRouter (для result-изображений) |

## Поток обработки (коротко)

1. `Telegram Trigger` получает `message`/`callback_query`
2. `Set Params` нормализует данные пользователя и события
3. Проверка пользователя в `ImageUsers`
4. Роутинг:
   - команды (`/start`, `/menu`, `/clear_context`, `/help`)
   - фото/текст/callback
5. При генерации:
   - загрузка активного контекста из `ImageHistory`
   - скачивание файлов из Supabase Storage
   - преобразование в base64
   - запрос в OpenRouter
6. Результат:
   - отправка пользователю
   - загрузка в Storage
   - запись в `ImageHistory`
   - обновление статусов предыдущих `active` записей на `cleared`

## Необходимые credentials в n8n

| Credential | Тип | Назначение |
|-----------|-----|-----------|
| `Telegram Image Video Generator` | Telegram API | Основной бот |
| `OpenRouter for Image-Video Generator` | OpenRouter API | Генерация изображений |
| `Supabase for ImageVideoGenerator` | Supabase API | Работа с таблицами |
| `Header Auth for ImageVideoGenerator` | HTTP Header Auth | Доступ к Supabase Storage |
| `VseLLM Images` | HTTP Bearer Auth | Авторизация HTTP-запросов к Storage |

Также нужен env-параметр в n8n:

- `TELEGRAM_BOT_TOKEN` — используется узлом `Create main menu` для вызова `setMyCommands`

## Установка и настройка

1. Импортируйте в n8n файл `Generate Images Bot v5 + DB + start.json`
2. Создайте Telegram-бота через [@BotFather](https://t.me/BotFather)
3. Создайте проект Supabase:
   - таблицы `ImageUsers` и `ImageHistory`
   - bucket `photos`
4. Получите OpenRouter API ключ на [openrouter.ai](https://openrouter.ai/)
5. Настройте все credentials в n8n
6. Добавьте env-переменную `TELEGRAM_BOT_TOKEN`
7. Активируйте workflow

## Статус

- [x] Регистрация пользователя и автокоманды на `/start`
- [x] Загрузка изображений в контекст
- [x] Генерация изображений с учетом контекста
- [x] Выбор модели через inline-кнопки
- [x] Сохранение контекста и результатов в Supabase
- [x] Очистка контекста
- [x] Отправка сжатого и оригинального файла
- [x] Базовая обработка ошибок
- [ ] Полноценная справка `/help`
