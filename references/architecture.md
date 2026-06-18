# Архитектура и стек

## Стек

| Компонент | Выбор | Обоснование |
|-----------|-------|-------------|
| UI | Flutter (Material 3) | Cross-platform iOS/Android |
| State management | Riverpod | Тестируемость, immutability, DI |
| Локальная БД | Drift (SQLite) | ORM, миграции, type-safe |
| Навигация | go_router | Современный, deeplink поддержка |
| i18n | ARB + flutter_localizations + intl | Стандарт Flutter |
| Часовые пояса | timezone (пакет) + IANA IDs | Корректный DST |
| Push | firebase_messaging (FCM) | Android + iOS через APNs |
| HTTP | dio + retrofit | Interceptors, retry, timeout |
| Backend | Supabase (PostgreSQL + Auth) | EU Frankfurt region |
| Edge Functions | Supabase (TypeScript/Deno) | Push уведомления |

## pubspec.yaml (ключевые зависимости)

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter

  # State
  flutter_riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5

  # Navigation
  go_router: ^13.0.0

  # Backend
  supabase_flutter: ^2.3.0

  # Local DB
  drift: ^2.18.0
  sqlite3_flutter_libs: ^0.5.0

  # Push
  firebase_core: ^2.27.0
  firebase_messaging: ^14.7.0
  flutter_local_notifications: ^17.0.0

  # Timezone
  timezone: ^0.9.4

  # i18n
  intl: ^0.19.0

  # Network
  dio: ^5.4.0

  # Models
  freezed_annotation: ^2.4.1
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.9
  freezed: ^2.5.2
  json_serializable: ^6.8.0
  riverpod_generator: ^2.4.0
  drift_dev: ^2.18.0
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.4
```

## Deeplink стратегия

**НЕ использовать**: Firebase Dynamic Links (устарели, ненадёжны в РФ).

**Использовать**: Universal Links (iOS) + App Links (Android).

```
Формат: https://app.doomsdayreminder.com/join/ABC123
iOS:    apple-app-site-association на том же домене
Android: assetlinks.json на том же домене
```

go_router маршрут:
```dart
GoRoute(
  path: '/join/:code',
  builder: (context, state) => JoinGroupScreen(
    inviteCode: state.pathParameters['code']!,
  ),
),
```

Fallback веб-страница: определяет платформу, показывает кнопку скачать приложение.

## Clean Architecture слои

```
Presentation → Domain → Data
     ↓              ↓        ↓
  Riverpod    Usecases   Repositories
  Widgets     Entities   (Local + Remote)
```

- **Domain**: чистый Dart, без Flutter зависимостей.
- **Data**: LocalFirst — сначала читаем из Drift, затем синхронизируем с Supabase.
- **Presentation**: Riverpod AsyncNotifier на каждый экран/фичу.

## Ключевые usecase'ы (MVP)

- `CreateEventUseCase` — создаёт локально + добавляет в sync_queue
- `JoinGroupUseCase` — по коду / ссылке / поиску
- `SyncQueueUseCase` — обрабатывает очередь при появлении сети
- `GetUpcomingEventsUseCase` — события за 24ч вперёд для Home
- `CreateTimerUseCase` — локальный или групповой таймер
