# Offline-first Cache Sync

Dart/Flutter библиотека для offline-first работы с данными. Локальный кэш на Drift + синхронизация с сервером.

**Принцип:** читаем локально → пишем локально + в outbox → sync() отправляет и получает данные.

## Содержание

- [Offline-first Cache Sync](#offline-first-cache-sync)
  - [Содержание](#содержание)
  - [Быстрый старт](#быстрый-старт)
    - [1. Установка](#1-установка)
    - [2. Настройка базы данных](#2-настройка-базы-данных)
    - [3. Настройка SyncEngine](#3-настройка-syncengine)
    - [4. Модель данных](#4-модель-данных)
  - [Работа с данными](#работа-с-данными)
    - [Чтение](#чтение)
    - [Создание](#создание)
    - [Обновление](#обновление)
    - [Удаление](#удаление)
    - [Синхронизация](#синхронизация)
  - [Разрешение конфликтов](#разрешение-конфликтов)
    - [Стратегии](#стратегии)
    - [autoPreserve](#autopreserve)
    - [Ручное разрешение](#ручное-разрешение)
    - [Кастомный merge](#кастомный-merge)
    - [Стратегия для отдельных таблиц](#стратегия-для-отдельных-таблиц)
  - [События и статистика](#события-и-статистика)
  - [Требования к серверу](#требования-к-серверу)
    - [Обязательные эндпоинты](#обязательные-эндпоинты)
    - [Обязательные поля в моделях](#обязательные-поля-в-моделях)
    - [Детекция конфликтов](#детекция-конфликтов)
    - [Идемпотентность](#идемпотентность)
    - [Force-update](#force-update)
    - [Пагинация](#пагинация)
    - [Примеры реализации](#примеры-реализации)
    - [Чеклист для сервера](#чеклист-для-сервера)

---

## Быстрый старт

### 1. Установка

```yaml
dependencies:
  synchronize_cache:
    path: packages/synchronize_cache
  synchronize_cache_rest:
    path: packages/synchronize_cache_rest
  drift: ^2.0.0
```

### 2. Настройка базы данных

```dart
import 'package:drift/drift.dart';
import 'package:synchronize_cache/synchronize_cache.dart';

// Таблицы для синхронизации (копируем структуру, используем типы из пакета)
@UseRowClass(SyncOutboxData)
class SyncOutboxLocal extends Table {
  TextColumn get opId => text()();
  TextColumn get kind => text()();
  TextColumn get entityId => text()();
  TextColumn get op => text()();
  TextColumn get payload => text().nullable()();
  IntColumn get ts => integer()();
  IntColumn get tryCount => integer().withDefault(const Constant(0))();
  IntColumn get baseUpdatedAt => integer().nullable()();
  TextColumn get changedFields => text().nullable()();

  @override
  Set<Column> get primaryKey => {opId};
  @override
  String get tableName => 'sync_outbox';
}

@UseRowClass(SyncCursorsData)
class SyncCursorsLocal extends Table {
  TextColumn get kind => text()();
  IntColumn get ts => integer()();
  TextColumn get lastId => text()();

  @override
  Set<Column> get primaryKey => {kind};
  @override
  String get tableName => 'sync_cursors';
}

// База данных
@DriftDatabase(tables: [DailyFeelings, SyncOutboxLocal, SyncCursorsLocal])
class AppDatabase extends _$AppDatabase with SyncDatabaseMixin {
  AppDatabase(super.e);
  @override
  int get schemaVersion => 1;
}
```

### 3. Настройка SyncEngine

```dart
import 'package:synchronize_cache_rest/synchronize_cache_rest.dart';

final transport = RestTransport(
  base: Uri.parse('https://api.example.com'),
  token: () async => 'Bearer ${await getToken()}',
);

final engine = SyncEngine(
  db: db,
  transport: transport,
  tables: [
    SyncableTable<DailyFeeling>(
      kind: 'daily_feeling',
      table: db.dailyFeelings,
      fromJson: DailyFeeling.fromJson,
      toJson: (e) => e.toJson(),
      toInsertable: (e) => e.toInsertable(),
    ),
  ],
);
```

### 4. Модель данных

```dart
@UseRowClass(DailyFeeling, generateInsertable: true)
class DailyFeelings extends Table {
  TextColumn get id => text()();
  IntColumn get mood => integer().nullable()();
  IntColumn get energy => integer().nullable()();
  TextColumn get notes => text().nullable()();
  DateTimeColumn get date => dateTime()();
  DateTimeColumn get updatedAt => dateTime()();  // обязательно!
  
  @override
  Set<Column> get primaryKey => {id};
}
```

---

## Работа с данными

### Чтение

Всегда читаем из локальной БД — мгновенно, работает офлайн:

```dart
// Все записи
final feelings = await db.select(db.dailyFeelings).get();

// С фильтром
final today = await (db.select(db.dailyFeelings)
  ..where((t) => t.date.equals(DateTime.now())))
  .getSingleOrNull();

// Реактивный stream для UI
db.select(db.dailyFeelings).watch().listen((list) {
  setState(() => _feelings = list);
});
```

### Создание

Сохраняем локально + добавляем в outbox:

```dart
Future<void> create(DailyFeeling feeling) async {
  // 1. Сразу в локальную БД (UI обновится мгновенно)
  await db.into(db.dailyFeelings).insert(feeling);
  
  // 2. В очередь на отправку
await db.enqueue(UpsertOp(
  opId: uuid.v4(),
    kind: 'daily_feeling',
    id: feeling.id,
  localTimestamp: DateTime.now().toUtc(),
    payloadJson: feeling.toJson(),
  ));
}
```

### Обновление

При обновлении важно указать `baseUpdatedAt` и `changedFields` для корректного merge:

```dart
Future<void> update(DailyFeeling updated, Set<String> changedFields) async {
  // 1. Обновляем локально
  await db.update(db.dailyFeelings).replace(updated);
  
  // 2. В outbox с метаданными для merge
  await db.enqueue(UpsertOp(
    opId: uuid.v4(),
    kind: 'daily_feeling',
    id: updated.id,
    localTimestamp: DateTime.now().toUtc(),
    payloadJson: updated.toJson(),
    baseUpdatedAt: updated.updatedAt,  // когда получили с сервера
    changedFields: changedFields,       // что изменил пользователь
  ));
}

// Пример использования
await update(
  feeling.copyWith(mood: 5),
  {'mood'},  // изменили только mood
);
```

### Удаление

```dart
Future<void> delete(String id, DateTime? serverUpdatedAt) async {
  // 1. Удаляем локально
  await (db.delete(db.dailyFeelings)..where((t) => t.id.equals(id))).go();
  
  // 2. В outbox
  await db.enqueue(DeleteOp(
    opId: uuid.v4(),
  kind: 'daily_feeling',
    id: id,
  localTimestamp: DateTime.now().toUtc(),
    baseUpdatedAt: serverUpdatedAt,
  ));
}
```

### Синхронизация

```dart
// Вручную
final stats = await engine.sync();

// Автоматически каждые 5 минут
engine.startAuto(interval: Duration(minutes: 5));
engine.stopAuto();

// Для конкретных таблиц
await engine.sync(kinds: {'daily_feeling', 'health_record'});
```

---

## Разрешение конфликтов

Конфликт возникает когда данные изменились и на клиенте, и на сервере.

### Стратегии

| Стратегия | Описание |
|-----------|----------|
| `autoPreserve` | **(по умолчанию)** Умный merge — сохраняет все данные |
| `serverWins` | Серверная версия побеждает |
| `clientWins` | Клиентская версия побеждает (force push) |
| `lastWriteWins` | Побеждает более поздний timestamp |
| `merge` | Кастомная функция слияния |
| `manual` | Ручное разрешение через callback |

### autoPreserve

Стратегия по умолчанию — объединяет данные без потерь:

```dart
// Локально: {mood: 5, notes: "My notes"}
// На сервере: {mood: 3, energy: 7}
// Результат:  {mood: 5, energy: 7, notes: "My notes"}
```

Как работает:
1. Берёт серверные данные как базу
2. Применяет локальные изменения (только `changedFields` если указаны)
3. Списки объединяет без дубликатов
4. Вложенные объекты мержит рекурсивно
5. Системные поля (`id`, `updatedAt`, `createdAt`) берёт с сервера
6. Отправляет результат с `X-Force-Update: true`

### Ручное разрешение

```dart
final engine = SyncEngine(
  // ...
  config: SyncConfig(
    conflictStrategy: ConflictStrategy.manual,
    conflictResolver: (conflict) async {
      // Показать диалог пользователю или решить программно
      
      return AcceptServer();       // взять серверную версию
      return AcceptClient();       // взять клиентскую версию
      return AcceptMerged({...});  // свой результат merge
      return DeferResolution();    // отложить (оставить в outbox)
      return DiscardOperation();   // отменить операцию
    },
  ),
);
```

### Кастомный merge

```dart
final engine = SyncEngine(
  // ...
  config: SyncConfig(
    conflictStrategy: ConflictStrategy.merge,
    mergeFunction: (local, server) {
      return {...server, ...local};
    },
  ),
);

// Встроенные утилиты
ConflictUtils.defaultMerge(local, server);
ConflictUtils.deepMerge(local, server);
ConflictUtils.preservingMerge(local, server, changedFields: {'mood'});
```

### Стратегия для отдельных таблиц

```dart
final engine = SyncEngine(
  // ...
  tableConflictConfigs: {
    'user_settings': TableConflictConfig(
      strategy: ConflictStrategy.clientWins,
    ),
  },
);
```

---

## События и статистика

```dart
// Подписка на события
engine.events.listen((event) {
  switch (event) {
    case SyncStarted(:final phase):
      print('Начало: $phase');
    case SyncProgress(:final done, :final total):
      print('Прогресс: $done/$total');
    case SyncCompleted(:final stats):
      print('Готово: pushed=${stats.pushed}, pulled=${stats.pulled}');
    case ConflictDetectedEvent(:final conflict):
      print('Конфликт: ${conflict.entityId}');
    case SyncErrorEvent(:final error):
      print('Ошибка: $error');
  }
});

// Статистика после sync
final stats = await engine.sync();
print('Отправлено: ${stats.pushed}');
print('Получено: ${stats.pulled}');
print('Конфликтов: ${stats.conflicts}');
print('Разрешено: ${stats.conflictsResolved}');
print('Ошибок: ${stats.errors}');
```

---

## Требования к серверу

Для полноценной работы offline-first синхронизации сервер должен реализовать определённый контракт.

### Обязательные эндпоинты

| Метод | URL | Описание |
|-------|-----|----------|
| `GET` | `/{kind}` | Получить список с фильтрацией |
| `POST` | `/{kind}` | Создать запись |
| `PUT` | `/{kind}/{id}` | Обновить запись |
| `DELETE` | `/{kind}/{id}` | Удалить запись |

`{kind}` — тип сущности (например `daily_feeling`, `health_record`).

### Обязательные поля в моделях

Каждая синхронизируемая сущность должна иметь:

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | `string` | Уникальный идентификатор (UUID) |
| `updatedAt` | `datetime` | Время последнего изменения в UTC |
| `deletedAt` | `datetime?` | Время удаления (для soft-delete) |

**Важно:** `updatedAt` должен устанавливаться **сервером** при каждом изменении!

```json
{
  "id": "abc-123",
  "mood": 5,
  "energy": 7,
  "updatedAt": "2025-01-15T10:30:00Z",
  "deletedAt": null
}
```

### Детекция конфликтов

Клиент отправляет `_baseUpdatedAt` — timestamp когда данные были получены с сервера.

**Запрос клиента:**
```http
PUT /daily_feeling/abc-123
Content-Type: application/json
X-Idempotency-Key: op-uuid-456

{
  "id": "abc-123",
  "mood": 5,
  "energy": 7,
  "_baseUpdatedAt": "2025-01-15T10:00:00Z"
}
```

**Логика сервера:**

```python
def update(kind: str, id: str, data: dict):
    existing = db.get(kind, id)
    if not existing:
        return 404, {"error": "not_found"}
    
    # Извлекаем _baseUpdatedAt из payload
    base_updated = data.pop('_baseUpdatedAt', None)
    
    # Проверяем конфликт
    if base_updated:
        base_dt = parse_datetime(base_updated)
        if existing.updated_at > base_dt:
            # Конфликт! Данные изменились после того как клиент их получил
            return 409, {
                "error": "conflict",
                "current": existing.to_dict()
            }
    
    # Нет конфликта — обновляем
    existing.update(data)
    existing.updated_at = datetime.utcnow()  # сервер ставит время!
    db.save(existing)
    
    return 200, existing.to_dict()
```

**Ответ при конфликте (409):**
```json
{
  "error": "conflict",
  "current": {
    "id": "abc-123",
    "mood": 4,
    "energy": 8,
    "updatedAt": "2025-01-15T11:30:00Z"
  }
}
```

### Идемпотентность

Клиент отправляет заголовок `X-Idempotency-Key` с UUID операции.

**Сервер должен:**
1. Сохранять ключи выполненных операций (минимум 24 часа)
2. При повторном запросе с тем же ключом — возвращать тот же результат
3. Не выполнять операцию повторно

```python
def handle_request(idempotency_key: str, handler):
    # Проверяем кэш
    cached = cache.get(f"idempotency:{idempotency_key}")
    if cached:
        return cached
    
    # Выполняем операцию
    result = handler()
    
    # Сохраняем результат
    cache.set(f"idempotency:{idempotency_key}", result, ttl=86400)
    return result
```

### Force-update

После merge клиент отправляет заголовок `X-Force-Update: true`.

**Сервер должен:**
- Принять данные **без проверки** `_baseUpdatedAt`
- Это позволяет клиенту записать объединённые данные

```http
PUT /daily_feeling/abc-123
Content-Type: application/json
X-Force-Update: true
X-Idempotency-Key: op-uuid-789

{
  "id": "abc-123",
  "mood": 5,
  "energy": 8,
  "notes": "Merged data"
}
```

```python
def update(kind: str, id: str, data: dict, headers: dict):
    force_update = headers.get('X-Force-Update') == 'true'
    
    if not force_update:
        # Обычная проверка конфликта
        base_updated = data.pop('_baseUpdatedAt', None)
        if base_updated and existing.updated_at > parse_datetime(base_updated):
            return 409, {"error": "conflict", "current": existing.to_dict()}
    
    # Force-update или нет конфликта — обновляем
    existing.update(data)
    existing.updated_at = datetime.utcnow()
    db.save(existing)
    return 200, existing.to_dict()
```

### Пагинация

**GET запрос с параметрами:**
```http
GET /daily_feeling?updatedSince=2025-01-01T00:00:00Z&limit=500&afterId=xyz&includeDeleted=true
```

| Параметр | Описание |
|----------|----------|
| `updatedSince` | Получить записи изменённые после этого времени |
| `limit` | Максимум записей (рекомендуется 500) |
| `afterId` | ID последней полученной записи (для стабильной пагинации) |
| `includeDeleted` | Включить удалённые записи (soft-delete) |

**Ответ:**
```json
{
  "items": [
    {"id": "abc-123", "mood": 5, "updatedAt": "2025-01-15T10:00:00Z"},
    {"id": "def-456", "mood": 3, "updatedAt": "2025-01-15T10:05:00Z"}
  ],
  "nextPageToken": "eyJ0cyI6IjIwMjUtMDEtMTVUMTA6MDU6MDBaIiwiaWQiOiJkZWYtNDU2In0="
}
```

**Важно — стабильная пагинация:**

Чтобы не терять и не дублировать записи при пагинации:

```sql
-- Используем составной курсор (updatedAt + id)
SELECT * FROM daily_feeling
WHERE 
  (updated_at > :updatedSince) 
  OR (updated_at = :updatedSince AND id > :afterId)
ORDER BY updated_at ASC, id ASC
LIMIT :limit
```

### Примеры реализации

<details>
<summary><b>Python (FastAPI)</b></summary>

```python
from fastapi import FastAPI, Header, HTTPException
from datetime import datetime
from typing import Optional

app = FastAPI()

@app.get("/{kind}")
async def list_items(
    kind: str,
    updatedSince: Optional[datetime] = None,
    limit: int = 500,
    afterId: Optional[str] = None,
    includeDeleted: bool = True,
):
    query = db.query(get_model(kind))
    
    if updatedSince:
        if afterId:
            query = query.filter(
                or_(
                    Model.updated_at > updatedSince,
                    and_(Model.updated_at == updatedSince, Model.id > afterId)
                )
            )
        else:
            query = query.filter(Model.updated_at >= updatedSince)
    
    if not includeDeleted:
        query = query.filter(Model.deleted_at.is_(None))
    
    items = query.order_by(Model.updated_at, Model.id).limit(limit).all()
    
    next_token = None
    if len(items) == limit:
        last = items[-1]
        next_token = encode_cursor(last.updated_at, last.id)
    
    return {"items": [i.to_dict() for i in items], "nextPageToken": next_token}


@app.put("/{kind}/{id}")
async def update_item(
    kind: str,
    id: str,
    data: dict,
    x_force_update: Optional[str] = Header(None),
    x_idempotency_key: Optional[str] = Header(None),
):
    # Идемпотентность
    if x_idempotency_key:
        cached = cache.get(f"idempotency:{x_idempotency_key}")
        if cached:
            return cached
    
    existing = db.get(get_model(kind), id)
    if not existing:
        raise HTTPException(404, {"error": "not_found"})
    
    # Проверка конфликта (если не force-update)
    if x_force_update != "true":
        base_updated = data.pop("_baseUpdatedAt", None)
        if base_updated:
            base_dt = datetime.fromisoformat(base_updated.replace("Z", "+00:00"))
            if existing.updated_at > base_dt:
                return JSONResponse(409, {
                    "error": "conflict",
                    "current": existing.to_dict()
                })
    else:
        data.pop("_baseUpdatedAt", None)
    
    # Обновляем
    for key, value in data.items():
        if key not in ("id", "updatedAt", "createdAt"):
            setattr(existing, key, value)
    existing.updated_at = datetime.utcnow()
    db.commit()
    
    result = existing.to_dict()
    
    # Сохраняем для идемпотентности
    if x_idempotency_key:
        cache.set(f"idempotency:{x_idempotency_key}", result, ttl=86400)
    
    return result
```
</details>

<details>
<summary><b>Node.js (Express)</b></summary>

```javascript
app.get('/:kind', async (req, res) => {
  const { kind } = req.params;
  const { updatedSince, limit = 500, afterId, includeDeleted = 'true' } = req.query;
  
  let query = db(kind);
  
  if (updatedSince) {
    if (afterId) {
      query = query.where(function() {
        this.where('updated_at', '>', updatedSince)
          .orWhere(function() {
            this.where('updated_at', '=', updatedSince).andWhere('id', '>', afterId);
          });
      });
    } else {
      query = query.where('updated_at', '>=', updatedSince);
    }
  }
  
  if (includeDeleted !== 'true') {
    query = query.whereNull('deleted_at');
  }
  
  const items = await query.orderBy('updated_at').orderBy('id').limit(limit);
  
  let nextPageToken = null;
  if (items.length === limit) {
    const last = items[items.length - 1];
    nextPageToken = Buffer.from(JSON.stringify({
      ts: last.updated_at,
      id: last.id
    })).toString('base64');
  }
  
  res.json({ items, nextPageToken });
});

app.put('/:kind/:id', async (req, res) => {
  const { kind, id } = req.params;
  const forceUpdate = req.headers['x-force-update'] === 'true';
  const idempotencyKey = req.headers['x-idempotency-key'];
  
  // Идемпотентность
  if (idempotencyKey) {
    const cached = await cache.get(`idempotency:${idempotencyKey}`);
    if (cached) return res.json(cached);
  }
  
  const existing = await db(kind).where({ id }).first();
  if (!existing) {
    return res.status(404).json({ error: 'not_found' });
  }
  
  const data = { ...req.body };
  const baseUpdatedAt = data._baseUpdatedAt;
  delete data._baseUpdatedAt;
  
  // Проверка конфликта
  if (!forceUpdate && baseUpdatedAt) {
    if (new Date(existing.updated_at) > new Date(baseUpdatedAt)) {
      return res.status(409).json({
        error: 'conflict',
        current: existing
      });
    }
  }
  
  // Обновляем
  delete data.id;
  delete data.updatedAt;
  delete data.createdAt;
  data.updated_at = new Date().toISOString();
  
  await db(kind).where({ id }).update(data);
  const result = await db(kind).where({ id }).first();
  
  // Сохраняем для идемпотентности
  if (idempotencyKey) {
    await cache.set(`idempotency:${idempotencyKey}`, result, 86400);
  }
  
  res.json(result);
});
```
</details>

### Чеклист для сервера

- [ ] Поле `updatedAt` в каждой модели, устанавливается сервером
- [ ] Поле `deletedAt` для soft-delete (опционально)
- [ ] GET с параметрами `updatedSince`, `limit`, `afterId`, `includeDeleted`
- [ ] Стабильная пагинация: `ORDER BY updatedAt, id`
- [ ] Формат ответа: `{"items": [...], "nextPageToken": "..."}`
- [ ] Проверка `_baseUpdatedAt` при PUT для детекции конфликтов
- [ ] Ответ 409 с `{"error": "conflict", "current": {...}}`
- [ ] Поддержка заголовка `X-Force-Update: true`
- [ ] Поддержка заголовка `X-Idempotency-Key` с кэшированием 24ч
- [ ] Возврат обновлённой записи в ответе PUT/POST
