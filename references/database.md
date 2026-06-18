# База данных (Supabase / PostgreSQL)

## Таблицы

### users
```sql
CREATE TABLE users (
  id                uuid        PRIMARY KEY DEFAULT auth.uid(),
  email             text        NOT NULL,
  username          text        NOT NULL,
  igg_id            text,
  display_name      text,
  avatar_url        text,
  timezone          text        NOT NULL DEFAULT 'UTC',  -- IANA: 'Europe/Paris'
  preferred_language text       NOT NULL DEFAULT 'en',
  created_at        timestamptz NOT NULL DEFAULT now(),
  updated_at        timestamptz NOT NULL DEFAULT now()
);
```

### groups
```sql
CREATE TABLE groups (
  id           uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
  name         text        NOT NULL,
  description  text,
  leader_id    uuid        NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  invite_code  text        NOT NULL UNIQUE,  -- 6 символов, генерируется функцией
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now()
);
```

### group_members
```sql
CREATE TABLE group_members (
  id        uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id  uuid        NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  user_id   uuid        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role      text        NOT NULL DEFAULT 'member',  -- leader | moderator | member
  joined_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE(group_id, user_id)
);
```

### events
```sql
CREATE TABLE events (
  id               uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id         uuid        REFERENCES groups(id) ON DELETE CASCADE,  -- null = личное
  creator_id       uuid        NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  title            text        NOT NULL,
  description      text,
  event_time_utc   timestamptz NOT NULL,   -- ВСЕГДА UTC
  event_timezone   text        NOT NULL,   -- IANA TZ создателя, фиксируется навсегда
  visibility       text        NOT NULL DEFAULT 'group',  -- group | personal
  is_recurring     boolean     NOT NULL DEFAULT false,
  recurrence_type  text,                   -- weekly | monthly (MVP); RFC 5545 — Phase 2
  recurrence_end   timestamptz,
  notify_before    jsonb       NOT NULL DEFAULT '[15, 60]',  -- массив минут
  is_template      boolean     NOT NULL DEFAULT false,
  deleted_at       timestamptz,            -- soft delete
  created_at       timestamptz NOT NULL DEFAULT now(),
  updated_at       timestamptz NOT NULL DEFAULT now()
);
```

### notification_settings
```sql
CREATE TABLE notification_settings (
  user_id               uuid    PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  push_enabled          boolean NOT NULL DEFAULT true,
  sound_enabled         boolean NOT NULL DEFAULT true,
  quiet_hours_enabled   boolean NOT NULL DEFAULT false,
  quiet_start_local     time,              -- 'HH:MM' в локальном TZ пользователя
  quiet_end_local       time,
  remind_before_minutes jsonb   NOT NULL DEFAULT '[15, 60]',
  time_format_24h       boolean NOT NULL DEFAULT true,
  updated_at            timestamptz NOT NULL DEFAULT now()
);
```

### devices
```sql
CREATE TABLE devices (
  id            uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       uuid        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  fcm_token     text        NOT NULL,
  platform      text        NOT NULL,  -- ios | android
  device_name   text,                  -- 'iPhone 15', 'Samsung S24'
  last_seen_at  timestamptz NOT NULL DEFAULT now(),
  created_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE(user_id, fcm_token)
);
```

### sync_queue
```sql
CREATE TABLE sync_queue (
  id           uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      uuid        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  entity_type  text        NOT NULL,   -- event | group | member
  entity_id    uuid        NOT NULL,
  operation    text        NOT NULL,   -- create | update | delete
  local_data   jsonb       NOT NULL,   -- снимок данных на момент операции
  sync_state   text        NOT NULL DEFAULT 'pending',  -- pending|syncing|synced|conflict|failed
  retry_count  int         NOT NULL DEFAULT 0,
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now()
);
```

---

## Вспомогательная функция: генерация invite_code

```sql
CREATE OR REPLACE FUNCTION generate_invite_code()
RETURNS text AS $$
DECLARE
  chars text := 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
  code  text := '';
  i     int;
BEGIN
  FOR i IN 1..6 LOOP
    code := code || substr(chars, floor(random() * length(chars) + 1)::int, 1);
  END LOOP;
  -- Убедиться в уникальности
  IF EXISTS (SELECT 1 FROM groups WHERE invite_code = code) THEN
    RETURN generate_invite_code();
  END IF;
  RETURN code;
END;
$$ LANGUAGE plpgsql;
```

Триггер автоматической генерации:
```sql
CREATE OR REPLACE FUNCTION set_invite_code()
RETURNS trigger AS $$
BEGIN
  IF NEW.invite_code IS NULL OR NEW.invite_code = '' THEN
    NEW.invite_code := generate_invite_code();
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_groups_invite_code
  BEFORE INSERT ON groups
  FOR EACH ROW EXECUTE FUNCTION set_invite_code();
```

---

## Индексы

```sql
-- Ближайшие события группы (главный запрос Home + Calendar)
CREATE INDEX idx_events_group_time
  ON events(group_id, event_time_utc)
  WHERE deleted_at IS NULL;

-- Личные события пользователя
CREATE INDEX idx_events_creator_time
  ON events(creator_id, event_time_utc)
  WHERE deleted_at IS NULL;

-- Членство в группах
CREATE INDEX idx_group_members_group ON group_members(group_id, user_id);
CREATE INDEX idx_group_members_user  ON group_members(user_id);

-- Вступление по коду
CREATE INDEX idx_groups_invite_code ON groups(invite_code);

-- FCM токены
CREATE INDEX idx_devices_user ON devices(user_id);

-- Очередь синхронизации
CREATE INDEX idx_sync_queue_user_state ON sync_queue(user_id, sync_state);
```

---

## RLS политики

### Включить RLS на всех таблицах
```sql
ALTER TABLE users               ENABLE ROW LEVEL SECURITY;
ALTER TABLE groups              ENABLE ROW LEVEL SECURITY;
ALTER TABLE group_members       ENABLE ROW LEVEL SECURITY;
ALTER TABLE events              ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_settings ENABLE ROW LEVEL SECURITY;
ALTER TABLE devices             ENABLE ROW LEVEL SECURITY;
ALTER TABLE sync_queue          ENABLE ROW LEVEL SECURITY;
```

### users
```sql
CREATE POLICY "users_select_own" ON users
  FOR SELECT USING (auth.uid() = id);

CREATE POLICY "users_update_own" ON users
  FOR UPDATE USING (auth.uid() = id);
```

### groups
```sql
-- Читать может любой участник группы
CREATE POLICY "groups_select_member" ON groups
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM group_members
      WHERE group_id = groups.id AND user_id = auth.uid()
    )
  );

-- Создавать может любой авторизованный
CREATE POLICY "groups_insert_auth" ON groups
  FOR INSERT WITH CHECK (auth.uid() = leader_id);

-- Редактировать / удалять только leader
CREATE POLICY "groups_update_leader" ON groups
  FOR UPDATE USING (auth.uid() = leader_id);

CREATE POLICY "groups_delete_leader" ON groups
  FOR DELETE USING (auth.uid() = leader_id);
```

### events
```sql
-- Читать: личные свои ИЛИ групповые если участник, deleted_at IS NULL
CREATE POLICY "events_select" ON events
  FOR SELECT USING (
    deleted_at IS NULL AND (
      (group_id IS NULL AND creator_id = auth.uid()) OR
      EXISTS (
        SELECT 1 FROM group_members
        WHERE group_id = events.group_id AND user_id = auth.uid()
      )
    )
  );

-- Создавать: авторизованный (group_id проверяется в приложении)
CREATE POLICY "events_insert" ON events
  FOR INSERT WITH CHECK (auth.uid() = creator_id);

-- Редактировать: создатель или leader группы
CREATE POLICY "events_update" ON events
  FOR UPDATE USING (
    auth.uid() = creator_id OR
    EXISTS (
      SELECT 1 FROM groups
      WHERE id = events.group_id AND leader_id = auth.uid()
    )
  );

-- Soft delete: только создатель
CREATE POLICY "events_soft_delete" ON events
  FOR UPDATE USING (auth.uid() = creator_id)
  WITH CHECK (deleted_at IS NOT NULL);
```

### notification_settings, devices, sync_queue
```sql
-- Только владелец
CREATE POLICY "notif_settings_own" ON notification_settings
  FOR ALL USING (auth.uid() = user_id);

CREATE POLICY "devices_own" ON devices
  FOR ALL USING (auth.uid() = user_id);

CREATE POLICY "sync_queue_own" ON sync_queue
  FOR ALL USING (auth.uid() = user_id);
```

---

## Soft delete — паттерн

Все запросы к `events` всегда добавляют `WHERE deleted_at IS NULL`.
RLS это обеспечивает на уровне БД автоматически (см. policy выше).
Физическое удаление — только через cron-job раз в 30 дней (Phase 2).
