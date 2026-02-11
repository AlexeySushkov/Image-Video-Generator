# Generate Images Bot v4 + DB

Telegram-бот для генерации и редактирования изображений с помощью нейросетей, реализованный как n8n workflow с хранением данных в Supabase.

## Архитектура

```
Telegram Bot  →  n8n Workflow  →  OpenRouter API (LLM)
                     ↕
               Supabase (DB + Storage)
```

## Стек технологий

| Компонент | Технология |
|-----------|-----------|
| Автоматизация | [n8n](https://n8n.io/) |
| Бот-платформа | Telegram Bot API |
| Генерация изображений | [OpenRouter API](https://openrouter.ai/) — Gemini 3 Pro Image Preview, FLUX.2-klein-4b |
| База данных | [Supabase](https://supabase.com/) (PostgreSQL) |
| Хранилище файлов | Supabase Storage (bucket: `photos`) |

## Возможности

- **Генерация изображений по текстовому описанию** — бот принимает текстовый промпт и генерирует изображение через Gemini 3 Pro
- **Редактирование изображений** — отправьте фото боту, добавьте в контекст, затем опишите что нужно изменить
- **Мультиконтекст** — можно загрузить несколько фотографий перед генерацией, все они будут переданы в LLM
- **Управление контекстом** — очистка контекста для начала новой генерации
- **Регистрация пользователей** — автоматическая регистрация через `/start`
- **История генераций** — все загруженные и сгенерированные изображения сохраняются в базе данных
- **Отправка оригинального файла** — помимо сжатого фото, бот отправляет оригинальный файл без сжатия Telegram

## Команды бота

| Команда | Описание |
|---------|----------|
| `/start` | Регистрация нового пользователя и установка команд бота |
| `/menu` | Главное меню с inline-кнопками |
| `/clear_context` | Очистить контекст (удалить все загруженные фото из активной сессии) |
| `/help` | Справка *(в разработке)* |

## Inline-кнопки

| Кнопка | Действие |
|--------|----------|
| Новая генерация | Очищает контекст и начинает новую сессию |
| Продолжить редактирование | Продолжить работу с текущим контекстом |
| Настройки | Настройки бота *(в разработке)* |

## Структура базы данных (Supabase)

### Таблица `Users`

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | Первичный ключ |
| `telegram_id` | string | Telegram ID пользователя |
| `user_name` | string | Имя пользователя |

### Таблица `History`

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | Первичный ключ |
| `user_id` | int | FK → Users.id |
| `prompt` | string | Текст промпта пользователя |
| `image_id` | string | ID файла в Supabase Storage |
| `image_name` | string | Путь (Key) файла в Storage |
| `image_status` | string | `active` — в текущем контексте, `cleared` — очищено |
| `image_type` | string | `context` — загружено пользователем, `result` — сгенерировано LLM |

## Схема работы workflow

```
Telegram Trigger
       │
       ▼
   Set Params  ──►  Извлечение user_id, username, text, photo, callback_data
       │
       ▼
   Get a User  ──►  Проверка пользователя в Supabase
       │
       ▼
   Summarize   ──►  Подсчёт найденных записей
       │
       ▼
    Switch1
    ┌───┼───────────────────┐
    ▼   ▼                   ▼
bot_cmd  ok (user found)   Unknown user ──► Alert админу
    │    │
    ▼    ▼
Switch   Switch message type
Bot Cmd  ┌──────┬──────┬──────────┬──────────────┬─────────┐
    │    ▼      ▼      ▼          ▼              ▼         ▼
    │  photo   text   file    callback_edit  callback_   fallback
    │    │      │      │          │          clear_ctx      │
    │    ▼      ▼      ▼          ▼              ▼         ▼
    │  Download Get    File     Edit          Clear     Unknown
    │  Photo   History unsup.  Message       Context    data type
    │    │      │
    │    ▼      ▼
    │  Upload  Download photos from DB
    │  to DB     │
    │    │       ▼
    │    ▼     Convert to base64
    │  Create    │
    │  context   ▼
    │  record  Prepare JSON → Gemini Pro → Prepare file
    │                                         │
    │                                         ▼
    │                              Send photo to user + Save to DB
    │                                         │
    │                                         ▼
    │                                    Create menu
    │
    ▼
 /start ──► SetCommands ──► Create User ok
 /clear_context ──► Update rows ──► Context cleared
 /menu ──► Create menu (inline keyboard)
 /help ──► TODO (в разработке)
```

## Необходимые credentials в n8n

| Credential | Тип | Назначение |
|-----------|-----|-----------|
| Telegram Image Video Generator | Telegram API | Основной бот-токен |
| OpenRouter Voice Fact-Checker | OpenRouter API | Доступ к моделям генерации изображений |
| Supabase for ImageVideoGenerator | Supabase API | Доступ к базе данных |
| Header Auth for ImageVideoGenerator | HTTP Header Auth | Авторизация в Supabase Storage |

## Используемые AI-модели

| Модель | Назначение |
|--------|-----------|
| `google/gemini-3-pro-image-preview` | Основная модель для генерации и редактирования изображений с контекстом |
| `black-forest-labs/flux.2-klein-4b` | Быстрая генерация простых изображений (512×512) |

## Установка и настройка

1. **Импортировать workflow** — загрузите файл `Generate Images Bot v4 + DB.json` в ваш n8n instance
2. **Создать Telegram бота** — через [@BotFather](https://t.me/BotFather) и получить токен
3. **Настроить Supabase**:
   - Создать проект
   - Создать таблицы `Users` и `History` согласно схеме выше
   - Создать storage bucket `photos`
   - Получить API ключи
4. **Получить OpenRouter API ключ** — зарегистрироваться на [openrouter.ai](https://openrouter.ai/) и получить ключ
5. **Настроить credentials** в n8n для всех интеграций
6. **Активировать workflow**

## Статус разработки

- [x] Регистрация пользователей
- [x] Загрузка фото в контекст
- [x] Генерация изображений по тексту с контекстом
- [x] Сохранение истории генераций
- [x] Очистка контекста
- [x] Главное меню
- [x] Отправка оригинального файла
- [x] Обработка ошибок
- [ ] Команда `/help`
- [ ] Настройки бота
- [ ] Выбор модели генерации
