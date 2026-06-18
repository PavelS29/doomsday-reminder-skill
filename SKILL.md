---
name: doomsday-reminder
description: >
  Full-stack mobile app development skill for the Doomsday Reminder (CalendA) project —
  a Flutter + Supabase event reminder app for Doomsday: Last Survivors players worldwide.
  Use this skill whenever the user asks to: build, scaffold, or extend the Doomsday Reminder app;
  generate Flutter screens, widgets, or providers; write Supabase SQL migrations, RLS policies,
  or Edge Functions; implement offline sync, push notifications (FCM), timezone logic, or i18n
  for this project; generate the project architecture, pubspec.yaml, or ARB translation files.
  Also trigger for questions about specific technical decisions in this app (e.g. "how do I handle
  timezones?", "write the sync_queue logic", "create the EventEditor screen").
  This skill contains the full product spec, DB schema, architecture decisions, and all constraints
  agreed upon during design — always consult it before generating any code or architecture for this project.
---

# Doomsday Reminder — Dev Skill

Полный спек мобильного приложения **CalendA / Doomsday Reminder**.
Стек: **Flutter (Material 3) + Supabase (EU Frankfurt)**.

> Перед генерацией кода — прочитай нужный reference файл из `references/`.
> Не изобретай решения, которые уже задокументированы здесь.

---

## Быстрый старт

| Нужно | Читай |
|-------|-------|
| Архитектура, стек, структура папок | `references/architecture.md` |
| БД: таблицы, индексы, RLS, SQL | `references/database.md` |
| Экраны: список, описание, состояния | `references/screens.md` |
| Логика времени, TZ, повторения | `references/timezone.md` |
| Уведомления, FCM, РФ-стратегия | `references/notifications.md` |
| i18n, ARB, плюрализация | `references/i18n.md` |
| Оффлайн, sync_queue, конфликты | `references/offline-sync.md` |
| MVP scope, Phase 2/3 план | `references/mvp-scope.md` |

---

## Ключевые принципы (всегда соблюдать)

1. **Production-качество**: Clean Architecture, SOLID, тестируемость.
2. **Никаких хардкод-секретов** — всё через ENV / конфиг с заглушками.
3. **Помечай каждый блок кода**: `// [MVP]`, `// [Phase 2]`, `// [Phase 3]`.
4. **UTC везде в БД** — `event_time_utc timestamptz`. Конвертация только на клиенте.
5. **Offline-first** — Drift локальная БД → UI. Supabase → синхронизация.
6. **sync_state флаг** на каждой локальной записи: `pending | synced | conflict | failed`.
7. **Никогда last-write-wins** — при конфликте показать UI выбора пользователю.
8. **РФ/Китай/Иран**: FCM нестабилен → резервный канал через локальные уведомления.
9. **Deeplink**: Universal Links (iOS) + App Links (Android), НЕ Firebase Dynamic Links.
10. **MVP языки**: только `en`, `ru`, `zh-CN`. Остальные — Phase 2.

---

## Рабочий процесс

При получении задачи:

1. Определи категорию (архитектура / БД / экран / логика / тесты).
2. Прочитай соответствующий reference файл.
3. Если задача касается нескольких областей — прочитай все нужные файлы.
4. Генерируй код с пометками `[MVP]` / `[Phase 2]`.
5. Если задача не описана в документации — уточни у пользователя перед реализацией.

---

## Структура Flutter проекта

```
lib/
  core/
    di/                    # Riverpod providers
    error/                 # AppException, failure classes
    utils/                 # timezone_helper.dart, date_formatter.dart
  data/
    local/                 # Drift: schema, DAOs, migrations
    remote/                # Supabase client, FCM service
    models/                # freezed + json_serializable
    repositories/          # LocalFirst + Remote sync impl
  domain/
    entities/              # Pure Dart entities
    repositories/          # Abstract interfaces
    usecases/              # CreateEvent, JoinGroup, SyncQueue...
  presentation/
    screens/               # onboarding/ auth/ home/ groups/ calendar/ timers/
    widgets/               # EventCard, TimerTile, GroupBadge...
    providers/             # Riverpod AsyncNotifiers
  l10n/
    app_en.arb
    app_ru.arb
    app_zh.arb
  main.dart

migrations/                # Supabase SQL (001_init.sql, 002_indexes.sql...)
functions/                 # Supabase Edge Functions (TypeScript)
test/                      # unit/ widget/ integration/
```

---

## Что выдавать в ответе (при генерации с нуля)

1. Уточняющие вопросы (если нужны) — максимум 5.
2. Архитектурный план (слои, роутинг, state, sync).
3. Скелет проекта: структура + `pubspec.yaml`.
4. SQL миграции: таблицы, индексы, RLS, функция `generate_invite_code()`.
5. ARB файлы (en, ru, zh) для всех MVP экранов.
6. Полная реализация: Login, Signup (3 шага), Home, GroupDetail, EventEditor, Calendar.
7. Остальные экраны — scaffold с `// TODO` комментариями.
8. Supabase Edge Function для push (TypeScript).
9. Примеры тестов: timezone конвертация, sync_state, widget test Login.
10. `README.md` с инструкциями Supabase / FCM / запуск.
11. Документ Phase 2/3: что добавить и как расширить.
