# Platform Plugin Public API

Публичный API платформы для интеграции плагинов. Описывает protobuf-контракты и протокол обмена сообщениями (gRPC).
Аналогично как сделано в "Интеллекте".

Контракты — единственный source of truth. Всё SDK генерируется из `.proto` файлов.

---

## Содержание

- [Plugin Configuration](#plugin-configuration)
- [Common Types](#common-types)
- [Channel](#channel)

---

## Plugin Configuration

Параметры плагина — типы объектов, топология (иерархия), события, реакции, опциональная необходимость в получении БД от платформы — задаются в JSON-файле конфигурации.

- **Схема конфигурации:** [`plugin_json_schema.json`](plugin_json_schema.json)
- **Схема локализации:** [`plugin_json_localization_schema.json`](plugin_json_localization_schema.json)

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

