# Platform Plugin Public API

Публичный API платформы для интеграции плагинов. Описывает protobuf-контракты и протокол обмена сообщениями (gRPC).
Аналогично как сделано в "Интеллекте".

Контракты — единственный source of truth. Всё SDK генерируется из `.proto` файлов.

---

## Содержание

- [Plugin Configuration](#plugin-configuration)
- [Common Types (public.common.proto)](#common-types-publiccommonproto)
- [Channel (public.channel.proto)](#channel-publicchannelproto)
- [Platform API (public.platform.proto)](#platform-api-publicplatformproto)

---

## Plugin Configuration

Параметры плагина задаются в JSON-файле конфигурации:

- Типы объектов — какие сущности использует плагин.
- Топология (иерархия) — как объекты связаны и вложены друг в друга.
- Все возможные события — на что плагин реагирует.
- Все возможные реакции — что плагин делает в ответ.

### Databases (optional)

Если плагину нужен доступ к базе данных:

- Плагин указывает логическое имя БД в конфигурации (поле `databases`)
- Платформа по этому имени предоставляет connection string
- Для кластерных конфигураций платформа сама обеспечивает доступность БД

Если БД от платформы не нужна — поле `databases` не указывается.

- **Схема конфигурации:** `[plugin_json_schema.json](plugin_json_schema.json)`
- **Схема локализации:** `[plugin_json_localization_schema.json](plugin_json_localization_schema.json)`

Локализационные ключи (`name_key`, `description_key`) из конфигурации резолвятся через файл локализации.

---

## Common Types

```protobuf
syntax = "proto3";

package platform.public.api.v1;

import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

option csharp_namespace = "Platform.Public.Api.V1";
option go_package = "platform/public/api/v1";

// Расширяемый тип объекта
// Поддерживаемые значения: "CAM", "SENSOR", "DETECTOR", "POS", "READER" и др.
message ObjectType {
  string type = 1;
}

// Объект с иерархией (для Setup/Delete)
message Unit {
  ObjectType type = 1;               // тип объекта
  string id = 2;                     // уникальный идентификатор
  string parent_id = 3;              // идентификатор родителя
  google.protobuf.Struct params = 4; // параметры объекта
}

// Действие над объектом (для Event/React)
message UnitAction {
  ObjectType type = 1;                     // тип объекта
  string id = 2;                           // идентификатор объекта
  string action = 3;                       // действие: "MD_START", "REC" и т.д.
  string uuid = 4;                         // UUID действия
  google.protobuf.Timestamp timestamp = 5; // временная метка
  google.protobuf.Struct params = 6;       // параметры действия
}
```

---

## Channel

### Жизненный цикл соединения

```
Плагин                                Платформа
  │                                     │
  │── Connect (metadata) ──────────────>│  1. gRPC вызов с аутентификацией
  │<────────────── Setup (INITIAL) ─────│  2. Начальная загрузка объектов
  │                                     │
  │── Subscribe ───────────────────────>│  3. Подписка на события
  │<────────────────── SubscribeOk ─────│
  │                                     │
  │<──────────────────────── Event ─────│  4. События в реальном времени
  │── React ───────────────────────────>│  5. Команды объектам
  │<─────────────── ReactResponse ──────│     Ответ на команду
  │                                     │
  │<──────────── Setup (NEW/UPDATE) ────│  6. Изменения объектов (всегда)
  │<─────────────────────── Delete ─────│  7. Удаление объектов (всегда)
  │                                     │
  │<───────────────────────── Ping ─────│  8. Heartbeat
  │── Pong ────────────────────────────>│
  │                                     │
  │── Unsubscribe ─────────────────────>│  9. Отписка от событий
  │<────────────────── UnsubscribeOk ───│
  │                                     │
  │<──────────────────── Shutdown ──────│  10. Завершение соединения
  │<──────────────────────── Error ─────│  *  Ошибки (в любой момент)
```

### Аутентификация (gRPC metadata)

Данные плагина передаются в metadata при вызове `Connect`:


| Ключ               | Описание              |
| ------------------ | --------------------- |
| `authorization`    | `Bearer <token>`      |
| `x-plugin-id`      | Идентификатор плагина |
| `x-plugin-version` | Версия плагина        |
| `x-api-version`    | Версия протокола API  |


При успехе — платформа начинает стрим с `Setup (INITIAL)`.
При отказе — платформа закрывает стрим с gRPC-статусом `UNAUTHENTICATED` или `FAILED_PRECONDITION` (неподдерживаемая версия API).

> **Примечание:** Платформа гарантирует, что все объекты будут переданы через `Setup` с `context = INITIAL` до отправки любых других сообщений (`Event`, `ReactResponse`, `Setup` с `NEW`/`UPDATE`). `Setup` и `Delete` приходят всегда, независимо от подписок. `Subscribe` управляет только фильтрацией `Event`.

### Proto-определение

```protobuf
syntax = "proto3";

package platform.public.api.v1;

import "platform/public/api/v1/common.proto";
import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";

option csharp_namespace = "Platform.Public.Api.V1";
option go_package = "platform/public/api/v1";

// gRPC-сервис канала сообщений (bidirectional streaming)
service ChannelService {
  rpc Connect(stream PluginMessage) returns (stream PlatformMessage);
}

// Контекст сообщения Setup
enum SetupContext {
  SETUP_CONTEXT_INITIAL = 0;   // начальная загрузка всех объектов
  SETUP_CONTEXT_NEW = 1;       // добавлен новый объект
  SETUP_CONTEXT_UPDATE = 2;    // объект обновлён
}

// ───── Сообщения плагин → платформа ─────

message Subscribe {
  string subscription_id = 1;  // идентификатор подписки (клиентский)
  message Filters {
    repeated ObjectType object_types = 1; // типы объектов
    repeated string actions = 2;          // действия (UnitAction.action)
  }
  Filters filters = 2;
}

message Unsubscribe {
  string subscription_id = 1;
}

message Pong {
  google.protobuf.Timestamp timestamp = 1;
}

// Команда объекту (плагин отправляет платформе)
message React {
  UnitAction action = 1;
}

// ───── Сообщения платформа → плагин ─────

// Создание или обновление объекта
message Setup {
  Unit object = 1;
  SetupContext context = 2;
  google.protobuf.Timestamp timestamp = 3;
}

// Удаление объекта (достаточно type + id)
message Delete {
  Unit object = 1;
  google.protobuf.Timestamp timestamp = 2;
}

// Событие от объекта
message Event {
  UnitAction action = 1;
}

// Ответ платформы на команду React
message ReactResponse {
  string uuid = 1;                         // uuid из UnitAction
  google.protobuf.Timestamp timestamp = 2; // временная метка ответа
  bool success = 3;                        // обработано успешно?
  google.protobuf.Struct result = 4;       // результат выполнения (опционально)
  string error_code = 5;                   // код ошибки (если !success)
  string error_message = 6;               // описание ошибки
}

message SubscribeOk {
  string subscription_id = 1;
}

message UnsubscribeOk {
  string subscription_id = 1;
}

message Ping {
  google.protobuf.Timestamp timestamp = 1;
}

message Shutdown {
  google.protobuf.Timestamp timestamp = 1;
}

message Error {
  google.protobuf.Timestamp timestamp = 1;
  string code = 2;
  string message = 3;
  google.protobuf.Struct details = 4;
}

// ───── Корневые сообщения ─────

// Сообщение от плагина к платформе
message PluginMessage {
  oneof payload {
    Subscribe subscribe = 1;
    Unsubscribe unsubscribe = 2;
    Pong pong = 3;
    React react = 4;
  }
}

// Сообщение от платформы к плагину
message PlatformMessage {
  oneof payload {
    Setup setup = 1;
    Delete delete = 2;
    Event event = 3;
    ReactResponse react_response = 4;
    SubscribeOk subscribe_ok = 5;
    UnsubscribeOk unsubscribe_ok = 6;
    Ping ping = 7;
    Shutdown shutdown = 8;
    Error error = 9;
  }
}
```

---

## Platform API (public.platform.proto)

```protobuf
syntax = "proto3";

package platform.public.api.v1;

import "platform/public/api/v1/common.proto";
import "google/protobuf/field_mask.proto";

option csharp_namespace = "Platform.Public.Api.V1";
option go_package = "platform/public/api/v1";

// gRPC-сервис для запросов к платформе (unary RPC)
service PlatformService {

  rpc ListObjects(ListObjectsRequest) returns (ListObjectsResponse);
  rpc ListChildren(ListChildrenRequest) returns (ListChildrenResponse);
  rpc GetObject(GetObjectRequest) returns (Unit);
}

// Получение объекта по типу и id
message GetObjectRequest {
  string type = 1;   // тип объекта
  string id = 2;     // id объекта
  google.protobuf.FieldMask mask = 3; // какие поля возвращать
}

// Получение объектов по типу
message ListObjectsRequest {
  string type = 1;   // тип объекта: "CAM", "SENSOR" и т.д.
  string page_token = 2;
  int32 page_size = 3;
  google.protobuf.FieldMask mask = 4; // какие поля возвращать для каждого Unit
}

message ListObjectsResponse {
  repeated Unit objects = 1;
  string next_page_token = 2;
}

// Получение дочерних объектов
message ListChildrenRequest {
  string parent_type = 1;  // тип родителя
  string parent_id = 2;    // id родителя
  string child_type = 3;   // тип дочерних объектов (пусто = все типы)
  string page_token = 4;
  int32 page_size = 5;
  google.protobuf.FieldMask mask = 6; // какие поля возвращать для каждого Unit
}

message ListChildrenResponse {
  repeated Unit objects = 1;
  string next_page_token = 2;
}
```

