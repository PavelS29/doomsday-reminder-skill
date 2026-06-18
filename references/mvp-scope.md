# MVP Scope и план фаз

## MVP ✅ — Входит

| Функция | Приоритет | Примечание |
|---------|-----------|------------|
| Онбординг (3 слайда) | P0 | |
| Вход / Регистрация (email + password) | P0 | |
| ForgotPassword flow | P0 | Через Supabase Auth |
| Гостевой режим (read-only, 7 дней) | P0 | Только просмотр |
| Профиль игрока (IGG ID, ник, TZ, аватар) | P0 | |
| Создание группы | P0 | |
| Вступление в группу (код + поиск + deeplink) | P0 | |
| Создание события (разово + еженед. + ежемес.) | P0 | |
| EventDetail экран | P0 | |
| Calendar view с фильтром | P0 | |
| Таймеры (обратный отсчёт) | P1 | |
| Локальные уведомления (личные события) | P0 | |
| Push FCM (групповые события) | P1 | |
| Резервный канал при FCM timeout | P1 | Для РФ/Китая |
| Оффлайн + sync_queue + conflict UI | P1 | |
| Настройки (TZ, язык, тихие часы, тема) | P0 | |
| Языки: en, ru, zh-CN | P0 | |
| Universal Links / App Links deeplink | P1 | |
| Оффлайн-баннер | P1 | |

**P0** = нельзя без этого запустить. **P1** = важно, но можно запустить без.

---

## НЕ входит в MVP ❌

| Функция | Фаза | Почему отложено |
|---------|------|-----------------|
| Чат в группе | Phase 2 | Требует Realtime, +2-3 недели |
| Google OAuth | Phase 2 | Нестабилен в РФ, требует настройки |
| Сложные RRULE (RFC 5545) | Phase 2 | Сложная логика генерации инстансов |
| Realtime обновления событий | Phase 2 | Supabase Realtime + Riverpod stream |
| Роль Moderator и permissions | Phase 2 | Упрощаем: только leader/member |
| Языки кроме en/ru/zh | Phase 2 | Переводы требуют времени |
| Аналитика / Sentry / Crashlytics | Phase 2 | Не критично для старта |
| Push для личных событий через FCM | Phase 2 | MVP: только локальные |
| Audit log | Phase 3 | История изменений |
| Планшеты / landscape | Phase 3 | Только портрет в MVP |
| Уведомления в Apple Watch / Android Wear | Phase 3 | |
| Web версия (Flutter Web) | Phase 3 | |

---

## Phase 2 — план добавления

### Чат в группе
- Таблица `messages (id, group_id, user_id, content, created_at)`
- Supabase Realtime: `supabase.from('messages').stream()`
- Riverpod `StreamProvider` для live-обновлений
- UI: простой чат в GroupDetailScreen (вкладка)

### Google OAuth
- Firebase Auth + Google Sign-In пакет
- Предупреждение для РФ: "Может работать медленно"
- Суть: `GoogleAuthProvider` → `supabase.auth.signInWithOAuth()`

### Realtime события
- `supabase.from('events').stream(primaryKey: ['id'])` → Riverpod StreamProvider
- Обновление локального Drift кэша при каждом изменении

### Сложные RRULE
- Пакет `rrule` для Dart
- Хранить в `events.recurrence_rule text` (RFC 5545 строка)
- Клиент парсит и генерирует инстансы

---

## Phase 3 — долгосрочный план

- Audit log (таблица `audit_log`, trigger на изменения)
- Планшеты (adaptive layout, двухколоночный режим)
- Web версия (Flutter Web, те же репозитории)
- Умные напоминания (ML — за сколько напоминать исходя из истории)
- Интеграция с игровым API Doomsday (если откроется)

---

## Временная оценка MVP

| Этап | Время | Что делается |
|------|-------|-------------|
| Неделя 1 | 5-7 дней | Настройка проекта, БД, Auth, Onboarding |
| Неделя 2 | 5-7 дней | Группы, вступление, deeplink |
| Неделя 3 | 5-7 дней | События, Calendar, EventEditor |
| Неделя 4 | 5-7 дней | Уведомления, Оффлайн-sync, Таймеры |
| Неделя 5 | 3-5 дней | Настройки, i18n, тесты, README |

**Итого: ~5 недель** до MVP с одним разработчиком.
