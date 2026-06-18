# Логика времени и часовых поясов

## Главное правило

> **БД хранит только UTC. Конвертация — только на клиенте.**

---

## Поля в таблице events

| Поле | Тип | Правило |
|------|-----|---------|
| `event_time_utc` | `timestamptz` | Всегда UTC. Неизменен после создания. |
| `event_timezone` | `text` | IANA TZ создателя на момент создания. Неизменен. |
| `users.timezone` | `text` | Текущий TZ пользователя. Используется только для отображения. |

---

## При создании события

```dart
// Пользователь выбирает дату/время в UI (локально)
final localDT = DateTime(2025, 6, 20, 18, 30); // 18:30 в TZ пользователя

// Конвертируем в UTC для сохранения в БД
import 'package:timezone/timezone.dart' as tz;

final userLocation = tz.getLocation(user.timezone); // 'Europe/Paris'
final tzDT = tz.TZDateTime.from(localDT, userLocation);
final utcDT = tzDT.toUtc(); // сохраняем это в event_time_utc

// event_timezone = user.timezone (фиксируется навсегда)
```

## При отображении события

```dart
// Читаем из БД
final utcDT = event.eventTimeUtc; // DateTime в UTC

// Конвертируем в TZ читателя
final readerLocation = tz.getLocation(currentUser.timezone);
final localDT = tz.TZDateTime.from(utcDT, readerLocation);

// Форматируем по локали
final formatted = DateFormat.yMd(locale).add_Hm().format(localDT);
```

## При смене TZ пользователем

- `users.timezone` обновляется.
- `event_time_utc` в БД **не меняется** — он всегда UTC.
- `event_timezone` события **не меняется** — исторический контекст.
- Все события автоматически отображаются в новом TZ при следующем рендере.

---

## Повторяющиеся события (MVP)

**Разрешены только**: `weekly` и `monthly`.
Сложные RRULE (RFC 5545) — Phase 2.

```dart
// Генерация инстансов на клиенте (6 месяцев вперёд)
List<DateTime> generateInstances(Event event) {
  final instances = <DateTime>[];
  var current = event.eventTimeUtc;
  final end = DateTime.now().add(const Duration(days: 180));

  while (current.isBefore(end)) {
    instances.add(current);
    current = switch (event.recurrenceType) {
      'weekly'  => current.add(const Duration(days: 7)),
      'monthly' => DateTime(current.year, current.month + 1, current.day,
                            current.hour, current.minute),
      _         => end, // стоп
    };
  }
  return instances;
}
```

> Инстансы интерпретируются в `event.eventTimezone`, не в TZ читателя.

---

## DST (летнее/зимнее время)

Используй **только** `timezone` пакет, никогда `DateTime.utc()` напрямую для конвертации:

```dart
// ❌ НЕПРАВИЛЬНО — не учитывает DST
final wrong = utcDT.add(const Duration(hours: 2));

// ✅ ПРАВИЛЬНО — timezone пакет учитывает DST
final location = tz.getLocation('Europe/Paris');
final correct = tz.TZDateTime.from(utcDT, location);
```

**Обязательные тесты DST:**

| Timezone | DST | Тест-кейс |
|----------|-----|-----------|
| `Europe/Moscow` | ❌ нет | Базовый UTC+3, без переходов |
| `Europe/Paris` | ✅ есть | Переход март/октябрь |
| `America/New_York` | ✅ есть | Другие даты перехода |
| `Asia/Shanghai` | ❌ нет | UTC+8, без переходов |
| `Asia/Kolkata` | ❌ нет | UTC+5:30, нецелый сдвиг |

---

## Тихие часы и TZ

Тихие часы применяются к **локальному TZ пользователя** (`users.timezone`), не к TZ события.

```dart
bool isQuietHours(NotificationSettings settings, String userTimezone) {
  if (!settings.quietHoursEnabled) return false;

  final location = tz.getLocation(userTimezone);
  final now = tz.TZDateTime.now(location);
  final currentTime = TimeOfDay(hour: now.hour, minute: now.minute);

  return currentTime.isAfter(settings.quietStart!) &&
         currentTime.isBefore(settings.quietEnd!);
}
```

---

## timezone пакет — инициализация

```dart
// main.dart — обязательно до запуска приложения
import 'package:timezone/data/latest.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  tz.initializeTimeZones(); // загружает все IANA TZ данные
  runApp(const ProviderScope(child: App()));
}
```
