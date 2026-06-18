# Уведомления

## Логика: какой канал когда

| Тип события | Канал | Пакет |
|-------------|-------|-------|
| Личное событие | Локальное уведомление | `flutter_local_notifications` |
| Групповое событие | Push FCM | `firebase_messaging` |
| FCM недоступен (timeout) | Резервный: локальное | `flutter_local_notifications` |

---

## Резервный канал (критично для РФ, Китая, Ирана)

```dart
Future<void> sendGroupEventNotification(Event event, User user) async {
  try {
    // Пробуем FCM через Supabase Edge Function
    await supabase.functions.invoke(
      'push_notification',
      body: {'eventId': event.id, 'userId': user.id},
    ).timeout(const Duration(seconds: 10)); // timeout для РФ
  } catch (e) {
    // FCM недоступен → резервный канал: локальное уведомление
    await _scheduleLocalNotification(event, user);
  }
}
```

---

## Supabase Edge Function (TypeScript)

Файл: `functions/push_notification/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  const { eventId } = await req.json()
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  )

  // Получаем событие
  const { data: event } = await supabase
    .from('events')
    .select('*, groups(name)')
    .eq('id', eventId)
    .single()

  if (!event?.group_id) return new Response('ok')

  // Получаем участников и их FCM токены
  const { data: members } = await supabase
    .from('group_members')
    .select('user_id, users(preferred_language, devices(fcm_token, platform))')
    .eq('group_id', event.group_id)

  // Отправляем каждому на его языке
  const fcmUrl = 'https://fcm.googleapis.com/v1/projects/YOUR_PROJECT/messages:send'
  const fcmKey = Deno.env.get('FCM_SERVER_KEY')!

  for (const member of members ?? []) {
    const lang = member.users?.preferred_language ?? 'en'
    const title = getTitle(lang, event.groups?.name)
    const body  = getBody(lang, event.title, event.event_time_utc)

    for (const device of member.users?.devices ?? []) {
      await fetch(fcmUrl, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${fcmKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          message: {
            token: device.fcm_token,
            notification: { title, body },
            data: { eventId, type: 'event_reminder' },
          }
        })
      })
    }
  }

  return new Response(JSON.stringify({ ok: true }))
})

// Локализация текста push-уведомлений
function getTitle(lang: string, groupName?: string): string {
  const titles: Record<string, string> = {
    en: `Event in ${groupName ?? 'your group'}`,
    ru: `Событие в ${groupName ?? 'вашей группе'}`,
    zh: `${groupName ?? '您的联盟'}有活动`,
  }
  return titles[lang] ?? titles.en
}

function getBody(lang: string, title: string, timeUtc: string): string {
  const time = new Date(timeUtc).toLocaleTimeString(lang === 'zh' ? 'zh-CN' : lang)
  const bodies: Record<string, string> = {
    en: `"${title}" starts at ${time}`,
    ru: `"${title}" начинается в ${time}`,
    zh: `"${title}" 开始于 ${time}`,
  }
  return bodies[lang] ?? bodies.en
}
```

Триггер функции — в Supabase: Database Webhook на INSERT в `events` WHERE `group_id IS NOT NULL`.

---

## Локальные уведомления (flutter_local_notifications)

```dart
// Планируем уведомление за N минут до события
Future<void> scheduleEventReminder({
  required Event event,
  required int minutesBefore,
  required String userTimezone,
}) async {
  final location = tz.getLocation(userTimezone);
  final eventLocal = tz.TZDateTime.from(event.eventTimeUtc, location);
  final notifyAt = eventLocal.subtract(Duration(minutes: minutesBefore));

  if (notifyAt.isBefore(tz.TZDateTime.now(location))) return; // уже прошло

  await flutterLocalNotifications.zonedSchedule(
    event.hashCode + minutesBefore,
    event.title,
    'Начало через $minutesBefore мин', // локализовать через intl
    notifyAt,
    const NotificationDetails(
      android: AndroidNotificationDetails(
        'events_channel', 'События',
        importance: Importance.high,
        priority: Priority.high,
      ),
      iOS: DarwinNotificationDetails(
        presentAlert: true,
        presentBadge: true,
        presentSound: true,
      ),
    ),
    androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
    uiLocalNotificationDateInterpretation:
        UILocalNotificationDateInterpretation.absoluteTime,
  );
}
```

---

## Тихие часы — применение

Перед отправкой локального уведомления проверить:

```dart
Future<void> scheduleWithQuietHours({
  required Event event,
  required NotificationSettings settings,
  required String userTimezone,
}) async {
  for (final minutes in settings.remindBeforeMinutes) {
    final location = tz.getLocation(userTimezone);
    var notifyAt = tz.TZDateTime.from(event.eventTimeUtc, location)
        .subtract(Duration(minutes: minutes));

    // Если попадает в тихие часы — перенести на конец тихого периода
    if (settings.quietHoursEnabled) {
      final notifyTime = TimeOfDay(hour: notifyAt.hour, minute: notifyAt.minute);
      if (_isInQuietHours(notifyTime, settings)) {
        notifyAt = _nextQuietEnd(notifyAt, settings, location);
      }
    }

    await scheduleEventReminder(
      event: event,
      minutesBefore: 0, // уже рассчитано выше
      userTimezone: userTimezone,
    );
  }
}
```

---

## APNs настройка (iOS) — README шаги

1. Apple Developer → Certificates, Identifiers & Profiles → Keys → + New Key
2. Включить Apple Push Notifications service (APNs)
3. Скачать `.p8` файл
4. Firebase Console → Project Settings → Cloud Messaging → Apple app configuration
5. Загрузить `.p8`, указать Key ID и Team ID
6. В `ios/Runner/Info.plist` добавить:
   ```xml
   <key>UIBackgroundModes</key>
   <array>
     <string>fetch</string>
     <string>remote-notification</string>
   </array>
   ```

---

## ENV переменные для push

```env
# .env (не коммитить в git)
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...   # только для Edge Functions
FCM_SERVER_KEY=ya29...              # Firebase service account token
```
