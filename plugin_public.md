# Platform Plugin Public API

Публичный API платформы для интеграции плагинов. Описывает protobuf-контракты и протокол обмена сообщениями (gRPC).
Аналогично как сделано в "Интеллекте".

Контракты — единственный source of truth. Всё SDK генерируется из `.proto` файлов.

---

## Содержание

- [Plugin Configuration](#plugin-configuration)
- [Common Types](#common-types)
- [Channel](#channel)
- [Plugin Resources](#plugin-resources)
- [Platform API](#platform-api)

---

## Plugin Configuration

Параметры плагина задаются в JSON-файле конфигурации:

- Типы объектов — какие сущности использует плагин.
- Топология (иерархия) — как объекты связаны и вложены друг в друга.
- Все возможные события — на что плагин реагирует.
- Все возможные реакции — что плагин делает в ответ.
- Публикуемые ресурсы — HTTP-серверы, статические файлы, внешние ссылки, которые плагин делает доступными через платформу.

### Databases (optional)

Если плагину нужен доступ к базе данных:

- Плагин указывает логическое имя БД в конфигурации (поле `databases`)
- Платформа по этому имени предоставляет connection string
- Для кластерных конфигураций платформа сама обеспечивает доступность БД

Если БД от платформы не нужна — поле `databases` не указывается.

- **Схема конфигурации:** [plugin_json_schema.json](plugin_json_schema.json)
- **Схема локализации:** [plugin_json_localization_schema.json](plugin_json_localization_schema.json)

Локализационные ключи (`name_key`, `description_key`) из конфигурации резолвятся через файл локализации.

### Resources (optional)

Плагин может публиковать собственные ресурсы (UI, API, ссылки), которые платформа сделает доступными для клиентов (WebUI и др.). Ресурсы декларируются в конфигурации:

```json
{
  "resources": [
    {
      "id": "web-ui",
      "type": "STATIC",
      "path_prefix": "/ui",
      "display_name_key": "resource.web_ui",
      "description_key": "resource.web_ui.desc"
    },
    {
      "id": "monitoring",
      "type": "LINK",
      "display_name_key": "resource.monitoring",
      "description_key": "resource.monitoring.desc"
    }
  ]
}
```

Здесь указываются только id, тип и ключи локализации. Конкретный `location` (адрес сервера, путь к папке, URL) передаётся в runtime при подключении через `RegisterResource` — см. [Plugin Resources](#plugin-resources).

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

// Ссылка на объект (лёгкая версия Unit)
message ObjectReference {
  ObjectType type = 1;  // тип объекта
  string id = 2;        // уникальный идентификатор
}

// Объект с иерархией (для Setup/Delete)
message Unit {
  ObjectReference object = 1;        // тип объекта
  string parent_id = 2;              // идентификатор родителя
  google.protobuf.Struct params = 3; // параметры объекта
}

// Действие над объектом (для Event/React)
message UnitAction {
  ObjectReference object = 1;              // тип объекта
  string action = 2;                       // действие: "MD_START", "REC" и т.д.
  string uuid = 3;                         // UUID действия
  google.protobuf.Timestamp timestamp = 4; // временная метка
  google.protobuf.Struct params = 5;       // параметры действия
}

// Тип публикуемого ресурса плагина
enum ResourceType {
  RESOURCE_TYPE_UNSPECIFIED = 0;
  RESOURCE_TYPE_PROXY = 1;   // HTTP(S)/WS сервер → платформа проксирует
  RESOURCE_TYPE_STATIC = 2;  // директория с файлами → платформа раздаёт
  RESOURCE_TYPE_LINK = 3;    // внешняя ссылка → платформа показывает как есть
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
  │── RegisterResource ────────────────>│  3. Публикация ресурсов плагина
  │<──────── RegisterResourceOk ────────│     Платформа возвращает public_url
  │                                     │
  │── Subscribe ───────────────────────>│  4. Подписка на события
  │<────────────────── SubscribeOk ─────│
  │                                     │
  │<──────────────────────── Event ─────│  5. События в реальном времени
  │── React ───────────────────────────>│  6. Команды объектам
  │<─────────────── ReactResponse ──────│     Ответ на команду
  │                                     │
  │<──────────── Setup (NEW/UPDATE) ────│  7. Изменения объектов (всегда)
  │<─────────────────────── Delete ─────│  8. Удаление объектов (всегда)
  │                                     │
  │<───────────────────────── Ping ─────│  9. Heartbeat
  │── Pong ────────────────────────────>│
  │                                     │
  │── UnregisterResource ──────────────>│  10. Снятие ресурса (или автоснятие при disconnect)
  │── Unsubscribe ─────────────────────>│  11. Отписка от событий
  │<────────────────── UnsubscribeOk ───│
  │                                     │
  │<──────────────────── Shutdown ──────│  12. Завершение соединения
  │<──────────────────────── Error ─────│  *  Ошибки (в любой момент)
```

### Аутентификация (gRPC metadata)

Данные плагина передаются в metadata при вызове `Connect`:


| Ключ               | Описание                                |
| ------------------ | --------------------------------------- |
| `authorization`    | `Bearer <token>`                        |
| `x-plugin-id`      | Идентификатор плагина                   |
| `x-plugin-version` | Версия плагина                          |
| `x-api-version`    | Версия протокола API                    |
| `x-plugin-license` | Лицензионная строка, выданная заказчику |


При успехе — платформа начинает стрим с `Setup (INITIAL)`.
При отказе — платформа закрывает стрим с gRPC-статусом:

- `UNAUTHENTICATED` — невалидный токен авторизации.
- `PERMISSION_DENIED` — невалидная или истёкшая лицензия.
- `FAILED_PRECONDITION` — неподдерживаемая версия API.

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

// Регистрация ресурса плагина
message RegisterResource {
  string resource_id = 1;        // id ресурса (из конфигурации плагина)
  ResourceType type = 2;         // тип ресурса
  string location = 3;           // зависит от type:
                                  //   PROXY:  "http://host:8080" (endpoint для проксирования)
                                  //   STATIC: "/opt/my-plugin/www" (путь к директории)
                                  //   LINK:   "https://grafana.local/d/abc" (внешний URL)
  string path_prefix = 4;        // суффикс публичного пути (напр. "/ui", "/api")
  map<string, string> meta = 5;  // произвольные метаданные
}

// Снятие ресурса
message UnregisterResource {
  string resource_id = 1;
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
  ObjectReference object = 1;
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

// Подтверждение регистрации ресурса
message RegisterResourceOk {
  string resource_id = 1;
  string public_url = 2;  // URL, по которому ресурс доступен через платформу
                           //   PROXY/STATIC: "/plugins/{plugin_id}/{path_prefix}/..."
                           //   LINK: оригинальный URL из location
}

// Подтверждение снятия ресурса
message UnregisterResourceOk {
  string resource_id = 1;
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
    RegisterResource register_resource = 5;
    UnregisterResource unregister_resource = 6;
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
    RegisterResourceOk register_resource_ok = 10;
    UnregisterResourceOk unregister_resource_ok = 11;
  }
}
```

---

## Plugin Resources

Плагин может публиковать собственные ресурсы — HTTP-серверы, статические файлы, внешние ссылки — делая их доступными через платформу (например, для WebUI).

### Типы ресурсов


| Тип      | `location`                   | Поведение платформы                                      |
| -------- | ---------------------------- | -------------------------------------------------------- |
| `PROXY`  | `http://host:port`           | Reverse proxy на endpoint плагина (HTTP, WebSocket, SSE) |
| `STATIC` | `/path/to/directory`         | Платформа раздаёт файлы из указанной директории          |
| `LINK`   | `https://external.service/…` | Платформа возвращает URL как есть (без проксирования)    |


### Жизненный цикл ресурса

1. Плагин **декларирует** возможные ресурсы в `plugin.json` (поле `resources`) — id, тип, ключи локализации. Это позволяет WebUI знать о ресурсах до подключения плагина.
2. При подключении плагин **регистрирует** каждый ресурс через `RegisterResource` с конкретным `location`, зависящим от окружения.
3. Платформа отвечает `RegisterResourceOk` с `public_url` — адресом, по которому ресурс доступен извне.
4. При отключении стрима платформа **автоматически снимает** все ресурсы плагина. Плагин также может явно снять ресурс через `UnregisterResource`.

> **Примечание:** Для `STATIC` платформа должна иметь доступ к файловой системе плагина (общий volume, сетевая папка и т.д.). Для `PROXY` платформа проксирует HTTP, WebSocket и SSE. Для `LINK` платформа не проксирует трафик — только передаёт URL клиенту.

> **TODO:** Модель разграничения доступа к ресурсам плагинов (ролевая модель) требует отдельной проработки.

---

## Platform API

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

