# Оффлайн-режим и синхронизация

## Архитектура

```
UI (Riverpod) ──reads──▶ Drift (локальная БД)
                               │
                         sync_queue
                               │
                           при сети
                               ▼
                        Supabase (remote)
```

- **Drift** — единственный источник правды для UI.
- **Supabase** — авторитетный источник правды для данных.
- Все записи в Drift имеют поле `syncState`.

---

## sync_state — значения

| Значение | Смысл |
|----------|-------|
| `synced` | Данные совпадают с сервером |
| `pending` | Ожидает отправки в Supabase |
| `syncing` | Сейчас отправляется |
| `conflict` | Локальная и серверная версии расходятся |
| `failed` | Ошибка после 5 попыток |

---

## Создание события (offline-first)

```dart
// CreateEventUseCase
Future<void> execute(CreateEventParams params) async {
  // 1. Сохраняем локально НЕМЕДЛЕННО
  final event = Event(
    id: const Uuid().v4(),
    syncState: SyncState.pending,
    // ...остальные поля
  );
  await localRepo.saveEvent(event);

  // 2. Добавляем в sync_queue
  await syncQueue.add(SyncOperation(
    entityType: 'event',
    entityId: event.id,
    operation: 'create',
    localData: event.toJson(),
  ));

  // 3. Если есть сеть — сразу пробуем синхронизировать
  if (await networkChecker.hasConnection()) {
    await syncQueueUseCase.processQueue();
  }
}
```

---

## Обработка очереди (SyncQueueUseCase)

```dart
Future<void> processQueue() async {
  final pending = await syncQueue.getPending(); // sync_state = 'pending'

  for (final op in pending) {
    try {
      await syncQueue.markSyncing(op.id);

      switch (op.operation) {
        case 'create': await remoteRepo.create(op);
        case 'update': await remoteRepo.update(op);
        case 'delete': await remoteRepo.softDelete(op);
      }

      await syncQueue.markSynced(op.id);
      await localRepo.updateSyncState(op.entityId, SyncState.synced);

    } catch (e) {
      if (e is ConflictException) {
        await _handleConflict(op);
      } else {
        await syncQueue.incrementRetry(op.id);
        if (op.retryCount >= 5) {
          await syncQueue.markFailed(op.id);
        }
      }
    }
  }
}
```

---

## Обработка конфликта

```dart
// НИКОГДА не использовать silent last-write-wins
Future<void> _handleConflict(SyncOperation op) async {
  // Получаем серверную версию
  final serverData = await remoteRepo.fetchById(op.entityId);

  // Помечаем локально как конфликт
  await localRepo.updateSyncState(op.entityId, SyncState.conflict);
  await localRepo.saveConflictData(op.entityId, serverData);

  // UI получит уведомление через Riverpod stream
  // и покажет ConflictResolutionBanner
}

// Пользователь выбирает разрешение:
Future<void> resolveConflict(String entityId, ConflictResolution resolution) async {
  switch (resolution) {
    case ConflictResolution.keepLocal:
      // Принудительно отправить локальную версию
      await remoteRepo.forceUpdate(entityId);
      await localRepo.updateSyncState(entityId, SyncState.synced);

    case ConflictResolution.acceptServer:
      // Заменить локальную версию серверной
      final serverData = await localRepo.getConflictData(entityId);
      await localRepo.replace(entityId, serverData);
      await localRepo.updateSyncState(entityId, SyncState.synced);
  }
}
```

---

## ConflictResolutionBanner (UI)

Показывать когда есть записи с `syncState == conflict`:

```dart
// В HomeScreen / GroupDetailScreen
if (hasConflicts) ...[
  Container(
    color: Colors.orange.shade50,
    padding: const EdgeInsets.all(12),
    child: Row(
      children: [
        const Icon(Icons.warning_amber, color: Colors.orange),
        const SizedBox(width: 8),
        Expanded(child: Text(context.l10n.conflictWarning)),
        TextButton(
          onPressed: () => context.push('/conflicts'),
          child: Text(context.l10n.resolve),
        ),
      ],
    ),
  ),
]
```

---

## Стратегия репликации

```dart
Future<void> initialSync() async {
  final now = DateTime.now().toUtc();

  // Pull: события за 30 дней назад + 6 месяцев вперёд
  final from = now.subtract(const Duration(days: 30));
  final to   = now.add(const Duration(days: 180));

  final remoteEvents = await remoteRepo.fetchEvents(from: from, to: to);
  await localRepo.upsertAll(remoteEvents);

  // Исторические данные (старше 30 дней) — только on demand
}
```

---

## Retry стратегия (экспоненциальный backoff)

```dart
Duration getRetryDelay(int retryCount) {
  // 1s → 2s → 4s → 8s → 16s
  return Duration(seconds: math.pow(2, retryCount).toInt());
}
```

---

## Offline баннер (UI)

```dart
// Показывать на всех основных экранах
StreamBuilder<ConnectivityResult>(
  stream: Connectivity().onConnectivityChanged,
  builder: (context, snapshot) {
    final isOffline = snapshot.data == ConnectivityResult.none;
    return AnimatedSwitcher(
      duration: const Duration(milliseconds: 300),
      child: isOffline
        ? Container(
            color: Colors.grey.shade800,
            padding: const EdgeInsets.symmetric(vertical: 4, horizontal: 12),
            child: Row(
              children: [
                const Icon(Icons.cloud_off, color: Colors.white, size: 14),
                const SizedBox(width: 6),
                Text(context.l10n.offlineBanner,
                     style: const TextStyle(color: Colors.white, fontSize: 12)),
                const Spacer(),
                Text(_lastSyncFormatted,
                     style: TextStyle(color: Colors.grey.shade400, fontSize: 10)),
              ],
            ),
          )
        : const SizedBox.shrink(),
    );
  },
)
```
