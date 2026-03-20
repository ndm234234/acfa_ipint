# Unified Device SDK — Архитектура единого SDK для интеграции устройств

## 1. Контекст и проблема

Платформа видеонаблюдения и безопасности имеет два независимых SDK для интеграции оборудования:

| | IpInt (IP-камеры) | ACFA (СКУД) |
|---|---|---|
| **Артефакт плагина** | Native DLL (C/C++) | .NET сборка |
| **Контракт** | C/C++ интерфейсы, экспортируемые функции | .NET interface (контракт) |
| **Типичные задачи** | Получение RTSP-потока, разбор, отдача кадров в платформу | Управление контроллерами доступа, события, команды |
| **Среда исполнения** | In-process, загрузка DLL в адресное пространство платформы | .NET host, загрузка сборки |

### Проблемы текущего подхода

- **Два разных SDK** — разные технологии, разные контракты, разная документация.
- **Высокий порог входа** — интегратору нужно знать C++ или C# и внутренние интерфейсы платформы.
- **Привязка к языку** — невозможно написать плагин на Python, Go, Rust или другом языке.
- **Ограниченная изоляция** — платформа реализует защитные механизмы (SEH, exception handling, watchdog), но in-process плагин не может быть полностью изолирован: утечки памяти, повреждение кучи (heap corruption), конфликты зависимостей (DLL hell) и deadlock'и внутри плагина остаются угрозой стабильности всей платформы.
- **Сложность отладки** — attach к процессу платформы, невозможность запуска плагина автономно.

---

## 2. Цель

**Единое SDK**, которое:

1. Заменяет оба существующих SDK одним набором контрактов.
2. Позволяет писать плагины на **любом языке** (Python, C#, C++, Go, Rust, Java, TypeScript и др.).
3. Максимально **простое** — минимальный шаблонный код, понятные абстракции.
4. Обеспечивает **настоящую изоляцию** — плагин работает в отдельном процессе, падение/утечка/зависание не влияет на платформу.
5. Поддерживает **обратную совместимость** — существующие плагины продолжают работать через адаптеры.

---

## 3. Ключевое архитектурное решение

### Плагин = отдельный процесс, общение с платформой через gRPC

```
┌──────────────────────────────────────────────────────────────┐
│                        ПЛАТФОРМА                              │
│                                                               │
│  ┌───────────────┐  ┌───────────────┐  ┌──────────────────┐ │
│  │  Video Core   │  │   ACS Core    │  │  Другие модули   │ │
│  │  (хранение,   │  │  (контроль    │  │  (аналитика,     │ │
│  │   отображение,│  │   доступа,    │  │   оповещения)    │ │
│  │   аналитика)  │  │   карты)      │  │                  │ │
│  └───────┬───────┘  └───────┬───────┘  └────────┬─────────┘ │
│          │                  │                    │            │
│          └──────────────────┼────────────────────┘            │
│                             │                                 │
│                    ┌────────▼─────────┐                       │
│                    │   Plugin Host    │                       │
│                    │   (gRPC Server)  │                       │
│                    │                  │                       │
│                    │ • Запуск плагинов│                       │
│                    │ • Health check   │                       │
│                    │ • Перезапуск     │                       │
│                    │ • Маршрутизация  │                       │
│                    └────────┬─────────┘                       │
│                             │                                 │
└─────────────────────────────┼─────────────────────────────────┘
                              │
              gRPC (Named Pipes / TCP / Unix Sockets)
                              │
   ┌──────────────────────────┼──────────────────────────────┐
   │              │           │           │                  │
┌──▼───────┐ ┌───▼──────┐ ┌──▼────────┐ ┌▼───────────────┐
│ Plugin A │ │ Plugin B │ │ Plugin C  │ │ Plugin D       │
│ Hikvision│ │ Dahua    │ │ ZKTeco    │ │ ATOL POS       │
│ (Python) │ │ (C++)    │ │ (C#)     │ │ (Python)       │
│          │ │          │ │          │ │                │
│ VideoSvc │ │ VideoSvc │ │ ACSvc    │ │ POSSvc         │
│ EventSvc │ │ PTZSvc   │ │ EventSvc │ │ EventSvc       │
└──────────┘ └──────────┘ └──────────┘ └────────────────┘
  Процесс A   Процесс B    Процесс C    Процесс D
```

### Почему gRPC

| Критерий | gRPC | REST/JSON | Shared Memory | Named Pipes (raw) |
|---|---|---|---|---|
| Кодогенерация для 10+ языков | Да | Через OpenAPI | Нет | Нет |
| Streaming (видео, события) | Нативно | Workarounds | Да | Вручную |
| Строгая типизация | protobuf | JSON Schema | Вручную | Вручную |
| Производительность | Высокая (бинарный, HTTP/2) | Средняя | Максимальная | Высокая |
| Версионирование контрактов | Встроено в protobuf | Вручную | Вручную | Вручную |
| Экосистема (health check, TLS, interceptors) | Из коробки | Частично | Нет | Нет |

gRPC — оптимальный баланс между производительностью, удобством и языковой независимостью.

---

## 4. Protobuf-контракты

Контракты — единственный source of truth. Всё SDK генерируется из `.proto` файлов.

### 4.1. Общие типы (`common.proto`)

```protobuf
syntax = "proto3";
package platform.plugin.v1;

option csharp_namespace = "Platform.Plugin.V1";

// Возможности, которые плагин может предоставлять
enum DeviceCapability {
  CAPABILITY_UNKNOWN = 0;
  CAPABILITY_VIDEO_STREAM = 1;
  CAPABILITY_PTZ = 2;
  CAPABILITY_ACCESS_CONTROL = 3;
  CAPABILITY_EVENTS = 4;
  CAPABILITY_IO = 5;
  CAPABILITY_AUDIO = 6;
  CAPABILITY_INTERCOM = 7;
  CAPABILITY_SENSORS = 8;
  CAPABILITY_POS = 9;
}

// Информация о плагине — передаётся при регистрации
message PluginInfo {
  string plugin_id = 1;
  string display_name = 2;
  string vendor = 3;
  string version = 4;
  repeated DeviceCapability capabilities = 5;
}

// Информация об устройстве
message DeviceInfo {
  string device_id = 1;
  string display_name = 2;
  string address = 3;
  int32 port = 4;
  map<string, string> credentials = 5;
  map<string, string> params = 6;
}

// Универсальный статус операции
message OperationStatus {
  bool success = 1;
  string error_code = 2;
  string error_message = 3;
}

```

### 4.2. Lifecycle-контракт (`lifecycle.proto`)

Общий для всех плагинов — управление жизненным циклом.

```protobuf
syntax = "proto3";
package platform.plugin.v1;

import "platform/plugin/v1/common.proto";

service PluginLifecycle {
  // Плагин регистрируется в платформе после старта
  rpc Register(PluginInfo) returns (RegisterResponse);

  // Платформа проверяет, что плагин жив
  rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);

  // Платформа передаёт конфигурацию плагину
  rpc Configure(ConfigureRequest) returns (OperationStatus);

  // Платформа добавляет устройство в плагин
  rpc AddDevice(DeviceInfo) returns (OperationStatus);

  // Платформа удаляет устройство из плагина
  rpc RemoveDevice(RemoveDeviceRequest) returns (OperationStatus);

  // Платформа запрашивает список устройств, которые плагин обслуживает
  rpc ListDevices(ListDevicesRequest) returns (ListDevicesResponse);

  // Платформа просит плагин завершить работу
  rpc Shutdown(ShutdownRequest) returns (OperationStatus);
}

message RegisterResponse {
  bool accepted = 1;
  string session_id = 2;
  map<string, string> platform_params = 3;
}

message HealthCheckRequest {}

message HealthCheckResponse {
  enum Status {
    HEALTHY = 0;
    DEGRADED = 1;
    UNHEALTHY = 2;
  }
  Status status = 1;
  string details = 2;
  int32 active_devices = 3;
}

message ConfigureRequest {
  map<string, string> settings = 1;
}

message RemoveDeviceRequest {
  string device_id = 1;
}

message ListDevicesRequest {}

message ListDevicesResponse {
  repeated DeviceInfo devices = 1;
}

message ShutdownRequest {
  int32 timeout_seconds = 1;
}
```

### 4.3. Видео-контракт (`video.proto`)

```protobuf
syntax = "proto3";
package platform.plugin.v1;

import "platform/plugin/v1/common.proto";

service VideoService {
  // Запросить видеопоток — платформа получает stream кадров
  rpc StartStream(StartStreamRequest) returns (stream VideoFrame);

  // Получить одиночный снимок
  rpc GetSnapshot(SnapshotRequest) returns (SnapshotResponse);

  // Получить список доступных каналов/потоков устройства
  rpc GetChannels(GetChannelsRequest) returns (GetChannelsResponse);
}

// Профиль потока
enum StreamProfile {
  STREAM_MAIN = 0;
  STREAM_SUB = 1;
  STREAM_THIRD = 2;
}

// Кодек видео
enum VideoCodec {
  CODEC_UNKNOWN = 0;
  CODEC_H264 = 1;
  CODEC_H265 = 2;
  CODEC_MJPEG = 3;
}

message StartStreamRequest {
  string device_id = 1;
  string channel_id = 2;
  StreamProfile profile = 3;
}

message VideoFrame {
  int64 timestamp_us = 1;       // PTS в микросекундах
  VideoCodec codec = 2;
  bool is_key_frame = 3;
  bytes data = 4;               // H.264/H.265 NAL units или MJPEG frame
  int32 width = 5;
  int32 height = 6;
}

message SnapshotRequest {
  string device_id = 1;
  string channel_id = 2;
  int32 width = 3;              // 0 = оригинальный размер
  int32 height = 4;
}

message SnapshotResponse {
  bytes jpeg_data = 1;
  int32 width = 2;
  int32 height = 3;
  int64 timestamp_us = 4;
}

message GetChannelsRequest {
  string device_id = 1;
}

message GetChannelsResponse {
  repeated ChannelInfo channels = 1;
}

message ChannelInfo {
  string channel_id = 1;
  string display_name = 2;
  repeated StreamProfile available_profiles = 3;
  repeated VideoCodec supported_codecs = 4;
}
```

### 4.4. PTZ-контракт (`ptz.proto`)

```protobuf
syntax = "proto3";
package platform.plugin.v1;

import "platform/plugin/v1/common.proto";

service PTZService {
  rpc ContinuousMove(PTZMoveRequest) returns (OperationStatus);
  rpc AbsoluteMove(PTZAbsoluteRequest) returns (OperationStatus);
  rpc Stop(PTZStopRequest) returns (OperationStatus);
  rpc GoToPreset(PTZPresetRequest) returns (OperationStatus);
  rpc GetPresets(PTZGetPresetsRequest) returns (PTZGetPresetsResponse);
}

message PTZMoveRequest {
  string device_id = 1;
  string channel_id = 2;
  float pan_speed = 3;          // -1.0 .. 1.0
  float tilt_speed = 4;         // -1.0 .. 1.0
  float zoom_speed = 5;         // -1.0 .. 1.0
}

message PTZAbsoluteRequest {
  string device_id = 1;
  string channel_id = 2;
  float pan = 3;
  float tilt = 4;
  float zoom = 5;
}

message PTZStopRequest {
  string device_id = 1;
  string channel_id = 2;
}

message PTZPresetRequest {
  string device_id = 1;
  string channel_id = 2;
  string preset_id = 3;
}

message PTZGetPresetsRequest {
  string device_id = 1;
  string channel_id = 2;
}

message PTZGetPresetsResponse {
  repeated PTZPreset presets = 1;
}

message PTZPreset {
  string preset_id = 1;
  string display_name = 2;
}
```

### 4.5. СКУД-контракт (`access_control.proto`)

```protobuf
syntax = "proto3";
package platform.plugin.v1;

import "platform/plugin/v1/common.proto";

service AccessControlService {
  // Получить список точек доступа (дверей, турникетов, шлагбаумов)
  rpc GetAccessPoints(GetAccessPointsRequest) returns (GetAccessPointsResponse);

  // Команды управления
  rpc GrantAccess(AccessCommand) returns (OperationStatus);
  rpc DenyAccess(AccessCommand) returns (OperationStatus);
  rpc LockPoint(AccessCommand) returns (OperationStatus);
  rpc UnlockPoint(AccessCommand) returns (OperationStatus);

  // Управление пользователями/картами на контроллере
  rpc SyncCredentials(SyncCredentialsRequest) returns (OperationStatus);

  // Подписка на события от контроллера
  rpc SubscribeEvents(SubscribeEventsRequest) returns (stream AccessEvent);
}

// Точка доступа
message AccessPoint {
  string point_id = 1;
  string display_name = 2;
  AccessPointType type = 3;
  AccessPointState state = 4;
}

enum AccessPointType {
  POINT_DOOR = 0;
  POINT_TURNSTILE = 1;
  POINT_GATE = 2;
  POINT_BARRIER = 3;
}

enum AccessPointState {
  STATE_UNKNOWN = 0;
  STATE_LOCKED = 1;
  STATE_UNLOCKED = 2;
  STATE_OPEN = 3;
  STATE_ALARM = 4;
}

message GetAccessPointsRequest {
  string device_id = 1;
}

message GetAccessPointsResponse {
  repeated AccessPoint points = 1;
}

message AccessCommand {
  string device_id = 1;
  string point_id = 2;
  map<string, string> params = 3;
}

// Событие СКУД
message AccessEvent {
  string device_id = 1;
  string point_id = 2;
  int64 timestamp_us = 3;
  AccessEventType event_type = 4;
  Severity severity = 5;
  string credential = 6;         // номер карты, PIN, биометрия ID
  string person_name = 7;        // если известно
  string photo_url = 8;          // фото с камеры верификации, если есть
  map<string, string> extra = 9;
}

enum AccessEventType {
  ACCESS_EVENT_UNKNOWN = 0;
  ACCESS_EVENT_GRANTED = 1;
  ACCESS_EVENT_DENIED = 2;
  ACCESS_EVENT_DOOR_OPENED = 3;
  ACCESS_EVENT_DOOR_CLOSED = 4;
  ACCESS_EVENT_DOOR_HELD_OPEN = 5;
  ACCESS_EVENT_FORCED_OPEN = 6;
  ACCESS_EVENT_TAMPER = 7;
  ACCESS_EVENT_ALARM = 8;
  ACCESS_EVENT_BUTTON_PRESSED = 9;
}

message SyncCredentialsRequest {
  string device_id = 1;
  repeated PersonCredential credentials = 2;
}

message PersonCredential {
  string person_id = 1;
  string display_name = 2;
  repeated string card_numbers = 3;
  repeated string pin_codes = 4;
  repeated string access_point_ids = 5;
  int64 valid_from_us = 6;
  int64 valid_until_us = 7;
}

message SubscribeEventsRequest {
  string device_id = 1;
  repeated AccessEventType event_filter = 2; // пустой = все события
}
```

### 4.6. Универсальные события (`events.proto`)

```protobuf
syntax = "proto3";
package platform.plugin.v1;

import "platform/plugin/v1/common.proto";

service EventService {
  // Универсальная подписка на события любого типа
  rpc Subscribe(EventSubscribeRequest) returns (stream DeviceEvent);
}

message EventSubscribeRequest {
  string device_id = 1;
  repeated string event_types = 2; // пустой = все
}

message DeviceEvent {
  string device_id = 1;
  string event_id = 2;
  string event_type = 3;
  int64 timestamp_us = 4;
  Severity severity = 5;
  string description = 6;
  map<string, string> data = 7;
  bytes snapshot = 8;             // опциональный JPEG-снимок
}
```

### 4.7. Сенсоры (`sensors.proto`)

Датчики: температура, влажность, дым, движение, затопление, газ, вибрация и т.д. Сюда же относятся носимые устройства мониторинга (например, Whoop — пульс, температура кожи, SpO2, акселерометр). Типичный паттерн — поток показаний + тревоги при превышении порогов.

```protobuf
syntax = "proto3";
package platform.plugin.v1;

import "platform/plugin/v1/common.proto";

service SensorService {
  rpc GetSensors(GetSensorsRequest) returns (GetSensorsResponse);
  rpc GetReadings(GetReadingsRequest) returns (GetReadingsResponse);
  rpc SubscribeReadings(SubscribeReadingsRequest) returns (stream SensorReading);
  rpc SubscribeAlarms(SubscribeAlarmsRequest) returns (stream SensorAlarm);
}

message SensorInfo {
  string sensor_id = 1;
  string display_name = 2;
  SensorType type = 3;
  string unit = 4;              // "°C", "%", "ppm", "lux"
  double min_value = 5;
  double max_value = 6;
}

enum SensorType {
  SENSOR_UNKNOWN = 0;
  SENSOR_TEMPERATURE = 1;
  SENSOR_HUMIDITY = 2;
  SENSOR_SMOKE = 3;
  SENSOR_MOTION = 4;
  SENSOR_FLOOD = 5;
  SENSOR_GAS = 6;
  SENSOR_VIBRATION = 7;
  SENSOR_LIGHT = 8;
  SENSOR_DOOR_CONTACT = 9;
  SENSOR_GLASS_BREAK = 10;
  SENSOR_CUSTOM = 100;
}

message SensorReading {
  string device_id = 1;
  string sensor_id = 2;
  int64 timestamp_us = 3;
  double value = 4;
  SensorState state = 5;
}

enum SensorState {
  SENSOR_STATE_NORMAL = 0;
  SENSOR_STATE_WARNING = 1;
  SENSOR_STATE_ALARM = 2;
  SENSOR_STATE_OFFLINE = 3;
  SENSOR_STATE_ERROR = 4;
}

message SensorAlarm {
  string device_id = 1;
  string sensor_id = 2;
  int64 timestamp_us = 3;
  Severity severity = 4;
  string description = 5;
  double trigger_value = 6;
  double threshold = 7;
}

message GetSensorsRequest {
  string device_id = 1;
}

message GetSensorsResponse {
  repeated SensorInfo sensors = 1;
}

message GetReadingsRequest {
  string device_id = 1;
  repeated string sensor_ids = 2; // пустой = все
}

message GetReadingsResponse {
  repeated SensorReading readings = 1;
}

message SubscribeReadingsRequest {
  string device_id = 1;
  repeated string sensor_ids = 2;
  int32 interval_ms = 3;         // минимальный интервал между показаниями, 0 = максимальная частота
}

message SubscribeAlarmsRequest {
  string device_id = 1;
  repeated string sensor_ids = 2;
}
```

### 4.8. POS-терминалы (`pos.proto`)

POS-интеграция — поток транзакций (чеки, операции) с привязкой к камере для верификации. Платформа накладывает текст транзакций на видео.

```protobuf
syntax = "proto3";
package platform.plugin.v1;

import "platform/plugin/v1/common.proto";

service POSService {
  rpc GetTerminals(GetTerminalsRequest) returns (GetTerminalsResponse);
  rpc GetTerminalStatus(TerminalStatusRequest) returns (TerminalStatusResponse);
  rpc SubscribeTransactions(SubscribeTransactionsRequest) returns (stream POSTransaction);
}

message TerminalInfo {
  string terminal_id = 1;
  string display_name = 2;
  string location = 3;
  string linked_camera_id = 4;   // привязка к камере для наложения на видео
}

message POSTransaction {
  string device_id = 1;
  string terminal_id = 2;
  int64 timestamp_us = 3;
  string transaction_id = 4;
  POSEventType event_type = 5;
  repeated POSItem items = 6;
  POSTotals totals = 7;
  string raw_text = 8;           // сырой текст чека для наложения на видео
  string operator_id = 9;
  string operator_name = 10;
  map<string, string> extra = 11;
}

enum POSEventType {
  POS_EVENT_UNKNOWN = 0;
  POS_EVENT_RECEIPT_START = 1;
  POS_EVENT_ITEM_ADDED = 2;
  POS_EVENT_ITEM_VOIDED = 3;
  POS_EVENT_SUBTOTAL = 4;
  POS_EVENT_PAYMENT = 5;
  POS_EVENT_RECEIPT_END = 6;
  POS_EVENT_REFUND = 7;
  POS_EVENT_NO_SALE = 8;        // открытие кассы без продажи
  POS_EVENT_DRAWER_OPEN = 9;
  POS_EVENT_ERROR = 10;
}

message POSItem {
  string name = 1;
  int32 quantity = 2;
  double price = 3;
  double total = 4;
  bool voided = 5;
}

message POSTotals {
  double subtotal = 1;
  double tax = 2;
  double discount = 3;
  double total = 4;
  string payment_method = 5;     // "cash", "card", "mixed"
}

message GetTerminalsRequest {
  string device_id = 1;
}

message GetTerminalsResponse {
  repeated TerminalInfo terminals = 1;
}

message TerminalStatusRequest {
  string device_id = 1;
  string terminal_id = 2;
}

message TerminalStatusResponse {
  string terminal_id = 1;
  bool online = 2;
  int64 last_transaction_us = 3;
  string last_error = 4;
}

message SubscribeTransactionsRequest {
  string device_id = 1;
  string terminal_id = 2;        // пустой = все терминалы
  repeated POSEventType event_filter = 3;
}
```

---

## 5. Manifest плагина

Каждый плагин поставляется с файлом `manifest.json`, описывающим его метаданные и способ запуска.

```json
{
  "id": "hikvision-camera",
  "display_name": "Hikvision IP Camera Plugin",
  "vendor": "Hikvision",
  "version": "1.2.0",
  "sdk_version": "1.0.0",
  "capabilities": [
    "VIDEO_STREAM",
    "PTZ",
    "EVENTS"
  ],
  "entry": {
    "command": "python",
    "args": ["-m", "hikvision_plugin"],
    "env": {
      "PYTHONUNBUFFERED": "1"
    }
  },
  "transport": {
    "type": "grpc",
    "port_mode": "dynamic"
  },
  "health_check": {
    "interval_sec": 10,
    "timeout_sec": 5,
    "max_failures": 3
  },
  "resource_limits": {
    "max_memory_mb": 512,
    "max_cpu_percent": 50
  }
}
```

### Структура директории плагина

```
plugins/
├── hikvision-camera/
│   ├── manifest.json
│   ├── hikvision_plugin/
│   │   ├── __main__.py
│   │   ├── camera.py
│   │   └── ...
│   └── requirements.txt
│
├── dahua-camera/
│   ├── manifest.json
│   └── dahua_plugin.exe
│
├── zkteco-acs/
│   ├── manifest.json
│   ├── ZkTecoPlugin.dll
│   └── ZkTecoPlugin.runtimeconfig.json
│
├── bolid-sensors/
│   ├── manifest.json
│   └── BolidSensorsPlugin.dll
│
└── atol-pos/
    ├── manifest.json
    ├── atol_pos_plugin/
    │   └── __main__.py
    └── requirements.txt
```

---

## 6. Plugin Host (текущий AppHost) — компонент платформы

Plugin Host (текущий AppHost) — модуль внутри платформы, отвечающий за весь жизненный цикл плагинов.

### Обязанности

```
Plugin Host
│
├── Discovery
│   └── Сканирует plugins/, читает manifest.json
│
├── Process Manager
│   ├── Запуск процесса плагина
│   ├── Передача параметров (порт, session ID)
│   ├── Мониторинг процесса (alive / exit code)
│   └── Перезапуск при падении (с backoff)
│
├── gRPC Client Pool
│   ├── Подключение к каждому плагину
│   ├── Маршрутизация вызовов (device_id → plugin)
│   └── Балансировка (если плагин обслуживает много устройств)
│
├── Health Monitor
│   ├── Периодический HealthCheck RPC
│   ├── Метрики (latency, error rate)
│   └── Алерты при деградации
│
└── Device Registry
    ├── Маппинг: device_id → plugin_id
    ├── AddDevice / RemoveDevice
    └── Хранение конфигурации устройств
```

### Протокол запуска плагина

```
Plugin Host                              Plugin Process
    │                                         │
    │  1. Запуск процесса                     │
    │  (command из manifest.json)              │
    ├────────────────────────────────────────►│
    │                                         │
    │  2. Плагин стартует gRPC-сервер         │
    │     на динамическом порту               │
    │                                         │
    │  3. Плагин пишет порт в stdout          │
    │◄────────────────────────────────────────┤
    │  (формат: "GRPC_PORT=50051")            │
    │                                         │
    │  4. Plugin Host подключается            │
    │     и вызывает Register()               │
    ├────────────────────────────────────────►│
    │                                         │
    │  5. Plugin Host вызывает Configure()    │
    ├────────────────────────────────────────►│
    │                                         │
    │  6. Plugin Host вызывает AddDevice()    │
    │     для каждого устройства              │
    ├────────────────────────────────────────►│
    │                                         │
    │  7. Плагин готов к работе               │
    │                                         │
    │  ... периодический HealthCheck() ...    │
    ├────────────────────────────────────────►│
    │◄────────────────────────────────────────┤
```

---

## 7. Примеры реализации плагинов

### 7.1. Плагин IP-камеры на Python

```python
import grpc
from concurrent import futures

from platform_sdk.v1 import (
    common_pb2,
    lifecycle_pb2, lifecycle_pb2_grpc,
    video_pb2, video_pb2_grpc,
)
from my_camera_lib import connect_rtsp, read_nalu


class HikvisionPlugin(
    lifecycle_pb2_grpc.PluginLifecycleServicer,
    video_pb2_grpc.VideoServiceServicer,
):
    def __init__(self):
        self.devices = {}

    # --- Lifecycle ---

    def AddDevice(self, request, context):
        self.devices[request.device_id] = request
        return common_pb2.OperationStatus(success=True)

    def RemoveDevice(self, request, context):
        self.devices.pop(request.device_id, None)
        return common_pb2.OperationStatus(success=True)

    def HealthCheck(self, request, context):
        return lifecycle_pb2.HealthCheckResponse(
            status=lifecycle_pb2.HealthCheckResponse.HEALTHY,
            active_devices=len(self.devices),
        )

    # --- Video ---

    def StartStream(self, request, context):
        device = self.devices[request.device_id]
        url = f"rtsp://{device.address}:{device.port}/Streaming/Channels/101"

        session = connect_rtsp(
            url,
            username=device.credentials.get("username"),
            password=device.credentials.get("password"),
        )

        try:
            for nalu in read_nalu(session):
                if context.is_active():
                    yield video_pb2.VideoFrame(
                        timestamp_us=nalu.pts,
                        codec=video_pb2.CODEC_H264,
                        is_key_frame=nalu.is_idr,
                        data=nalu.data,
                        width=nalu.width,
                        height=nalu.height,
                    )
                else:
                    break
        finally:
            session.close()

    def GetSnapshot(self, request, context):
        device = self.devices[request.device_id]
        jpeg = fetch_snapshot_http(device)
        return video_pb2.SnapshotResponse(jpeg_data=jpeg)


def main():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=8))
    plugin = HikvisionPlugin()

    lifecycle_pb2_grpc.add_PluginLifecycleServicer_to_server(plugin, server)
    video_pb2_grpc.add_VideoServiceServicer_to_server(plugin, server)

    port = server.add_insecure_port("127.0.0.1:0")
    server.start()

    print(f"GRPC_PORT={port}", flush=True)
    server.wait_for_termination()


if __name__ == "__main__":
    main()
```

### 7.2. Плагин СКУД на CSharp

```csharp
using Grpc.Core;
using Platform.Plugin.V1;

public class ZkTecoPlugin : AccessControlService.AccessControlServiceBase
{
    private readonly Dictionary<string, ZkController> _controllers = new();

    public override Task<OperationStatus> AddDevice(
        DeviceInfo request, ServerCallContext context)
    {
        var controller = new ZkController(request.Address, request.Port);
        controller.Connect(
            request.Credentials["username"],
            request.Credentials["password"]);
        _controllers[request.DeviceId] = controller;

        return Task.FromResult(new OperationStatus { Success = true });
    }

    public override async Task<GetAccessPointsResponse> GetAccessPoints(
        GetAccessPointsRequest request, ServerCallContext context)
    {
        var controller = _controllers[request.DeviceId];
        var doors = await controller.GetDoorsAsync();

        var response = new GetAccessPointsResponse();
        foreach (var door in doors)
        {
            response.Points.Add(new AccessPoint
            {
                PointId = door.Id.ToString(),
                DisplayName = door.Name,
                Type = AccessPointType.PointDoor,
                State = MapState(door.State),
            });
        }
        return response;
    }

    public override async Task SubscribeEvents(
        SubscribeEventsRequest request,
        IServerStreamWriter<AccessEvent> responseStream,
        ServerCallContext context)
    {
        var controller = _controllers[request.DeviceId];

        await foreach (var evt in controller.ReadEventsAsync(context.CancellationToken))
        {
            await responseStream.WriteAsync(new AccessEvent
            {
                DeviceId = request.DeviceId,
                PointId = evt.DoorId.ToString(),
                TimestampUs = evt.Timestamp.ToUnixTimeMicroseconds(),
                EventType = MapEventType(evt.Type),
                Credential = evt.CardNumber,
                PersonName = evt.PersonName ?? "",
            });
        }
    }

    public override Task<OperationStatus> GrantAccess(
        AccessCommand request, ServerCallContext context)
    {
        var controller = _controllers[request.DeviceId];
        controller.OpenDoor(int.Parse(request.PointId));
        return Task.FromResult(new OperationStatus { Success = true });
    }
}
```

### 7.3. Плагин на Go (скелет)

```go
package main

import (
    "fmt"
    "net"
    "google.golang.org/grpc"
    pb "github.com/mycompany/platform-sdk/gen/go/platform/plugin/v1"
)

type OnvifPlugin struct {
    pb.UnimplementedPluginLifecycleServer
    pb.UnimplementedVideoServiceServer
    devices map[string]*pb.DeviceInfo
}

func (p *OnvifPlugin) AddDevice(ctx context.Context, req *pb.DeviceInfo) (*pb.OperationStatus, error) {
    p.devices[req.DeviceId] = req
    return &pb.OperationStatus{Success: true}, nil
}

func (p *OnvifPlugin) StartStream(req *pb.StartStreamRequest, stream pb.VideoService_StartStreamServer) error {
    device := p.devices[req.DeviceId]
    // ... подключение к ONVIF камере, чтение RTSP ...
    for frame := range readFrames(device) {
        if err := stream.Send(&pb.VideoFrame{
            TimestampUs: frame.PTS,
            Codec:       pb.VideoCodec_CODEC_H264,
            IsKeyFrame:  frame.IsIDR,
            Data:        frame.NALData,
        }); err != nil {
            return err
        }
    }
    return nil
}

func main() {
    lis, _ := net.Listen("tcp", "127.0.0.1:0")
    s := grpc.NewServer()
    plugin := &OnvifPlugin{devices: make(map[string]*pb.DeviceInfo)}

    pb.RegisterPluginLifecycleServer(s, plugin)
    pb.RegisterVideoServiceServer(s, plugin)

    fmt.Printf("GRPC_PORT=%d\n", lis.Addr().(*net.TCPAddr).Port)
    s.Serve(lis)
}
```

### 7.4. Плагин Whoop на Python (сенсоры)

```python
from platform_sdk.helpers import PluginBase, run

class WhoopPlugin(PluginBase):
    capabilities = ["SENSORS"]

    def on_add_device(self, device):
        self.api = WhoopAPI(device.credentials["access_token"])

    def on_subscribe_readings(self, device_id, sensor_ids, interval_ms):
        for sample in self.api.stream_realtime():
            yield SensorReading(
                device_id=device_id,
                sensor_id="heart_rate",
                value=sample.hr,
                state=SENSOR_STATE_ALARM if sample.hr > 180 else SENSOR_STATE_NORMAL,
            )

run(WhoopPlugin)
```

### 7.5. Плагин POS-терминала на Python (чеки)

```python
from platform_sdk.helpers import PluginBase, run

class AtolPOSPlugin(PluginBase):
    capabilities = ["POS"]

    def on_add_device(self, device):
        self.conn = AtolConnection(device.address, device.port)

    def on_subscribe_transactions(self, device_id, terminal_id, event_filter):
        for receipt in self.conn.listen():
            yield POSTransaction(
                device_id=device_id,
                terminal_id=receipt.terminal,
                transaction_id=receipt.id,
                event_type=POS_EVENT_RECEIPT_END,
                raw_text=receipt.text,
                totals=POSTotals(total=receipt.total, payment_method=receipt.payment),
            )

run(AtolPOSPlugin)
```

---

## 8. Передача видеопотока — оптимизация

### Базовый режим: gRPC Server Streaming

Подходит для большинства сценариев (до ~50 камер на плагин). H.264/H.265 NAL units передаются как `bytes` без перекодирования. Overhead gRPC минимален по сравнению с размером видеоданных.

### Высоконагруженный режим: Shared Memory + gRPC Signaling

Для сценариев с сотнями камер или 4K-потоками:

```
Plugin Process                          Platform Process
┌──────────────┐                       ┌──────────────┐
│              │   1. Создаёт shared   │              │
│              │      memory segment   │              │
│              │                       │              │
│  Ring Buffer ├───── mmap / ────────►│  Ring Buffer │
│  (write)     │   CreateFileMapping   │  (read)      │
│              │                       │              │
│              │   2. gRPC: только     │              │
│              │      метаданные       │              │
│              ├──────────────────────►│              │
│              │   {offset, size,      │              │
│              │    timestamp, codec}  │              │
└──────────────┘                       └──────────────┘
```

Это опциональная оптимизация. Плагин сообщает о поддержке через `manifest.json`:

```json
{
  "transport": {
    "type": "grpc",
    "video_transport": "shared_memory",
    "shm_buffer_size_mb": 64
  }
}
```

---

## 9. Обратная совместимость — Bridge-адаптеры

Чтобы существующие плагины продолжали работать без переписывания, создаются два адаптера:

### 9.1. IpInt Bridge (Native DLL → gRPC)

```
┌─────────────────────────────────────────────────┐
│  IpInt Bridge Process                            │
│                                                  │
│  ┌──────────────┐     ┌───────────────────────┐ │
│  │ gRPC Server  │◄───►│ Legacy DLL Loader     │ │
│  │ (VideoSvc,   │     │                       │ │
│  │  Lifecycle)  │     │ LoadLibrary()         │ │
│  │              │     │ GetProcAddress()      │ │
│  │  Транслирует │     │ Вызов C-интерфейсов   │ │
│  │  gRPC вызовы │     │ старого плагина       │ │
│  │  в вызовы    │     │                       │ │
│  │  legacy API  │     │ ┌───────────────────┐ │ │
│  │              │     │ │ hikvision_old.dll │ │ │
│  └──────────────┘     │ └───────────────────┘ │ │
│                       └───────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### 9.2. ACFA Bridge (.NET Assembly → gRPC)

```
┌─────────────────────────────────────────────────┐
│  ACFA Bridge Process (.NET)                      │
│                                                  │
│  ┌──────────────┐     ┌───────────────────────┐ │
│  │ gRPC Server  │◄───►│ .NET Assembly Loader  │ │
│  │ (AccessCtl,  │     │                       │ │
│  │  Lifecycle)  │     │ Assembly.LoadFrom()   │ │
│  │              │     │ Activator.Create()    │ │
│  │  Транслирует │     │ Вызов .NET interface  │ │
│  │  gRPC вызовы │     │ старого плагина       │ │
│  │  в вызовы    │     │                       │ │
│  │  legacy API  │     │ ┌───────────────────┐ │ │
│  │              │     │ │ ZkTeco_old.dll    │ │ │
│  └──────────────┘     │ └───────────────────┘ │ │
│                       └───────────────────────┘ │
└─────────────────────────────────────────────────┘
```

Для платформы оба bridge выглядят как обычные gRPC-плагины. Миграция прозрачна.

---

## 10. Структура репозитория SDK

```
unified-device-sdk/
│
├── proto/                                # Source of truth — protobuf контракты
│   └── platform/plugin/v1/
│       ├── common.proto
│       ├── lifecycle.proto
│       ├── video.proto
│       ├── ptz.proto
│       ├── access_control.proto
│       ├── sensors.proto
│       ├── pos.proto
│       └── events.proto
│
├── sdk-python/                           # pip install platform-device-sdk
│   ├── platform_sdk/
│   │   ├── v1/                           # сгенерированный код из proto
│   │   └── helpers/                      # удобные обёртки
│   │       ├── plugin_base.py            # базовый класс плагина
│   │       └── server.py                 # запуск gRPC-сервера в одну строку
│   ├── pyproject.toml
│   └── examples/
│
├── sdk-dotnet/                           # NuGet: Platform.DeviceSDK
│   ├── Platform.DeviceSDK/
│   │   ├── Generated/                    # сгенерированный код
│   │   └── Helpers/
│   │       ├── PluginBase.cs
│   │       └── PluginHost.cs
│   └── Platform.DeviceSDK.csproj
│
├── sdk-cpp/                              # Conan / vcpkg
│   ├── generated/
│   ├── include/platform/plugin/
│   └── CMakeLists.txt
│
├── sdk-go/                               # Go module
│   ├── gen/go/platform/plugin/v1/
│   └── helpers/
│
├── bridges/                              # Адаптеры для legacy плагинов
│   ├── ipint-bridge/                     # Native DLL → gRPC
│   └── acfa-bridge/                      # .NET Assembly → gRPC
│
├── plugin-runner/                        # Утилита для локальной разработки
│   ├── runner.py                         # эмулирует платформу
│   └── test_devices.json                 # тестовые устройства
│
├── examples/
│   ├── python-hikvision-camera/
│   ├── csharp-zkteco-acs/
│   ├── go-onvif-camera/
│   ├── python-atol-pos/
│   ├── csharp-bolid-sensors/
│   └── cpp-custom-device/
│
├── docs/
│   ├── getting-started.md
│   ├── plugin-development-guide.md
│   ├── migration-from-ipint.md
│   └── migration-from-acfa.md
│
├── buf.yaml                              # buf.build — линтинг и breaking change detection
├── buf.gen.yaml                          # правила кодогенерации
└── Makefile                              # make generate, make build-all
```

---

## 11. Helper-библиотеки — обёртки для каждого языка

Для каждого поддерживаемого языка SDK поставляет **helper-библиотеку**, которая полностью скрывает gRPC. Разработчик плагина не создаёт gRPC-сервер, не выбирает порт, не реализует health check и регистрацию. Он работает только с бизнес-логикой.

### Что helper-библиотека делает за разработчика

| Задача | Без helper (голый gRPC) | С helper-библиотекой |
|---|---|---|
| Создать gRPC-сервер | Вручную (~20 строк) | Автоматически внутри `run()` |
| Выбрать порт, сообщить платформе | Вручную (`add_insecure_port`, `print`) | Автоматически |
| Реализовать `HealthCheck` | Вручную | Встроено, отвечает `HEALTHY` по умолчанию |
| Реализовать `Register` | Вручную | Автоматически из `capabilities` |
| Управлять реестром устройств | Вручную (`dict`, `AddDevice`, `RemoveDevice`) | Встроено (`self.devices`) |
| Логирование | Настроить самому | Готовый `self.log` |
| Graceful shutdown | Обработка сигналов вручную | Встроено |

### Что остаётся разработчику плагина

1. Наследоваться от базового класса
2. Указать `capabilities`
3. Реализовать `on_*` методы — **только бизнес-логику**
4. Вызвать `run()`

### Helper-библиотеки по языкам

| Язык | Пакет | Базовый класс | Запуск |
|---|---|---|---|
| Python | `pip install platform-plugin-sdk` | `PluginBase` | `run(MyPlugin)` |
| C# | NuGet `Platform.Plugin.SDK` | `PluginBase<T>` | `PluginHost.Run<MyPlugin>()` |
| C++ | vcpkg `platform-plugin-sdk` | `PluginBase` | `plugin::Run(myPlugin)` |
| Go | `go get platform-plugin-sdk` | `PluginBase` (embed struct) | `plugin.Run(&MyPlugin{})` |

### Минимальный плагин на Python

```python
from platform_plugin_sdk import PluginBase, run

class MyPlugin(PluginBase):
    capabilities = ["VIDEO_STREAM"]

    def on_add_device(self, device):
        self.log.info(f"Device added: {device.address}")

    def on_start_stream(self, device_id, channel_id, profile):
        url = self.get_rtsp_url(device_id)
        for frame in self.read_rtsp(url):
            yield frame

run(MyPlugin)
```

### Минимальный плагин на CSharp

```csharp
using Platform.Plugin.V1;
using Platform.Plugin.SDK;

public class MyPlugin : PluginBase<MyPlugin>
{
    public override string[] Capabilities => ["VIDEO_STREAM"];

    public override void OnAddDevice(DeviceInfo device)
        => Log.Info($"Device added: {device.Address}");

    public override IEnumerable<VideoFrame> OnStartStream(
        string deviceId, string channelId, StreamProfile profile)
    {
        foreach (var frame in ReadRtsp(deviceId))
            yield return frame;
    }
}

// Program.cs
PluginHost.Run<MyPlugin>();
```

### Минимальный плагин на Go

```go
package main

import plugin "github.com/mycompany/platform-plugin-sdk-go"

type MyPlugin struct {
    plugin.PluginBase
}

func (p *MyPlugin) Capabilities() []string {
    return []string{"VIDEO_STREAM"}
}

func (p *MyPlugin) OnStartStream(deviceID, channelID string, profile int) <-chan *plugin.VideoFrame {
    ch := make(chan *plugin.VideoFrame)
    go func() {
        defer close(ch)
        for frame := range p.ReadRTSP(deviceID) {
            ch <- frame
        }
    }()
    return ch
}

func main() {
    plugin.Run(&MyPlugin{})
}
```

### Plugin Runner — отладка без платформы

Для локальной разработки и тестирования SDK поставляет утилиту `plugin-runner`, которая эмулирует платформу:

```bash
# Запуск плагина с тестовым устройством
plugin-runner --plugin ./my-plugin/ --test-device 192.168.1.64

# Автоматическая валидация контрактов
plugin-runner --plugin ./my-plugin/ --validate
# ✓ manifest.json валиден
# ✓ gRPC сервер стартует
# ✓ Register() отвечает
# ✓ HealthCheck() отвечает HEALTHY
# ✓ AddDevice() принимает тестовое устройство
# ✓ StartStream() отдаёт кадры
```

Разработчик может полностью написать и отладить плагин, не имея доступа к платформе.

---

## 12. Версионирование и совместимость

### Protobuf гарантии

- Новые поля добавляются с новыми номерами — старые плагины продолжают работать.
- Удалённые поля помечаются `reserved` — защита от переиспользования номеров.
- `buf breaking` в CI — автоматическая проверка обратной совместимости при каждом PR.

### Версии SDK

```
proto/platform/plugin/v1/   ← текущая стабильная версия
proto/platform/plugin/v2/   ← будущая версия (когда потребуется breaking change)
```

Plugin Host поддерживает несколько версий одновременно. Плагин указывает `sdk_version` в `manifest.json`.

---

## 13. Безопасность

| Аспект | Решение |
|---|---|
| Изоляция процессов | Каждый плагин — отдельный процесс с ограниченными правами |
| Сетевой доступ | gRPC только на loopback (127.0.0.1), не доступен извне |
| Аутентификация | mTLS или token-based auth между Plugin Host и плагином |
| Ресурсы | `resource_limits` в manifest — контроль памяти и CPU |
| Credentials | Пароли устройств передаются через `Configure()` / `AddDevice()`, не хранятся в manifest |

---

## 14. Этапы реализации

| Этап | Описание | Результат |
|---|---|---|
| **1. Контракты** | Определить и стабилизировать `.proto` файлы | Набор proto-файлов, прошедших ревью |
| **2. Кодогенерация** | Настроить `buf.build`, генерация для Python, C#, C++ | SDK-пакеты для 3 языков |
| **3. Plugin Host** | Реализовать в платформе: discovery, запуск, health check, маршрутизация | Платформа умеет работать с gRPC-плагинами |
| **4. Первый плагин** | Написать плагин камеры на Python как proof of concept | Работающий пример, валидация архитектуры |
| **5. Helper-библиотеки** | `PluginBase`, `run()`, удобные обёртки для каждого языка | Минимальный шаблонный код для разработчика |
| **6. Plugin Runner** | Утилита для локальной разработки и тестирования | Разработчик может тестировать без платформы |
| **7. Bridge IpInt** | Адаптер для существующих native DLL плагинов | Legacy камерные плагины работают через новый SDK |
| **8. Bridge ACFA** | Адаптер для существующих .NET СКУД плагинов | Legacy СКУД плагины работают через новый SDK |
| **9. Документация** | Getting started, migration guides, API reference | Полная документация для интеграторов |
| **10. Миграция** | Постепенный перевод существующих плагинов на новый SDK | Отказ от legacy SDK |

---

## 15. Сравнение: до и после

| | Было | Стало |
|---|---|---|
| SDK | Отдельные SDK для камер, СКУД, сенсоров, POS | 1 единый SDK для всех типов устройств |
| Контракт | C-интерфейсы + .NET interface (разные для каждого SDK) | Protobuf (единый для всех) |
| Подсистемы | Видео, СКУД, сенсоры, POS — каждая со своим подходом | Единые контракты: VideoService, AccessControlService, SensorService, POSService |
| Языки | Только C++ / только C# | Любой (Python, C#, C++, Go, Rust, Java, ...) |
| Изоляция | In-process (SEH/try-catch, но heap corruption и утечки остаются угрозой) | Out-of-process (гарантированная изоляция на уровне ОС) |
| Отладка | Attach to platform process | Запуск плагина отдельно |
| Тестирование | Только с платформой | Plugin Runner — автономно |
| Обновление плагина | Перезапуск платформы | Hot-reload отдельного плагина |
| Порог входа | Высокий | Низкий (минимальный плагин ~20 строк) |
| Документация | Раздельная для каждого SDK | Единая, генерируется из proto-файлов |

---

## 16. Пример комплексного решения: мониторинг лошадей на ипподроме

Пример демонстрирует, как единое SDK объединяет разные типы устройств (камера + сенсоры) и аналитику (CV-модель) в одном плагине.

### Сценарий

Лошадь с датчиками на шее бежит по кругу. Датчики снимают пульс, скорость, температуру. Камера наблюдает за треком. Нейросеть (CV-модель) по видеопотоку трекает лошадь и фиксирует пересечение финишной линии. Плагин подсчитывает круги, объединяет данные датчиков и кругов, вычисляет **strain** — показатель нагрузки лошади — и отдаёт его платформе как готовое значение. Платформе не нужно знать формулу — она получает strain как обычное показание сенсора.

### Архитектура

```
┌─────────────────┐     ┌──────────────────┐
│  Датчик на шее  │     │  IP-камера       │
│  (пульс, GPS,   │     │  (обзор трека)   │
│   акселерометр)  │     │                  │
└────────┬────────┘     └────────┬─────────┘
         │                       │
         └───────────┬───────────┘
                     │
              ┌──────▼──────────────┐
              │  HorseStrainPlugin  │
              │  (Python)           │
              │                     │
              │  BLE ← датчики      │
              │  RTSP ← камера      │
              │  CV-модель → круги  │
              │  strain = f(...)    │
              │                     │
              │  SensorSvc          │
              │  EventSvc           │
              └──────────┬──────────┘
                         │
                  Plugin Host (платформа)
                         │
              ┌──────────▼──────────┐
              │  Отображение strain │
              │  в интерфейсе       │
              └─────────────────────┘
```

### Плагин HorseStrainPlugin

Один плагин читает датчики, трекает лошадь по камере, считает strain и отдаёт платформе готовый результат:

```python
from platform_plugin_sdk import PluginBase, run
import threading

class HorseStrainPlugin(PluginBase):
    capabilities = ["SENSORS", "EVENTS"]

    def on_add_device(self, device):
        self.ble = BLEConnection(device.params["ble_address"])
        self.camera_url = device.params["rtsp_url"]

        # Плагин сам загружает нейросеть для трекинга
        self.detector = YOLO(device.params["yolo_weights"])
        self.tracker = DeepSORT()
        self.finish_line = FinishLine(x1=100, y1=400, x2=500, y2=400)

        self.lap_count = 0
        self.prev_crossed = False
        self.latest = {"hr": 0, "speed": 0, "temp": 0}

    def on_subscribe_readings(self, device_id, sensor_ids, interval_ms):
        for sample in self.ble.stream():
            self.latest["hr"] = sample.hr
            self.latest["speed"] = sample.speed
            self.latest["temp"] = sample.temp

            yield SensorReading(device_id=device_id, sensor_id="heart_rate", value=sample.hr)
            yield SensorReading(device_id=device_id, sensor_id="speed_kmh", value=sample.speed)
            yield SensorReading(device_id=device_id, sensor_id="temperature", value=sample.temp)
            yield SensorReading(device_id=device_id, sensor_id="strain", value=self._calc_strain())

    def on_subscribe_events(self, device_id, event_types):
        for frame in self.read_rtsp(self.camera_url):
            # Инференс: детекция лошади на кадре
            detections = self.detector.predict(frame)

            # Трекинг: привязка детекции к конкретной лошади
            tracks = self.tracker.update(detections, frame)

            for track in tracks:
                crossed = self.finish_line.is_crossed(track.prev_pos, track.pos)

                if crossed and not self.prev_crossed:
                    self.lap_count += 1
                    yield DeviceEvent(
                        device_id=device_id,
                        event_type="lap_completed",
                        data={
                            "lap_number": str(self.lap_count),
                            "strain": str(self._calc_strain()),
                        },
                        snapshot=frame.to_jpeg(),
                    )
                self.prev_crossed = crossed

    def _calc_strain(self):
        return (
            self.lap_count * 100
            - self.latest["hr"] * 0.5
            + self.latest["speed"] * 2.0
            - self.latest["temp"] * 0.3
        )

run(HorseStrainPlugin)
```

### Что демонстрирует этот пример

| Аспект | Как решается через SDK |
|---|---|
| Датчики на лошади (BLE/IoT) | `on_subscribe_readings()` → `SensorService.SubscribeReadings()` |
| Трекинг по камере (CV-модель) | `on_subscribe_events()` → RTSP + нейросеть → пересечение финишной линии |
| Подсчёт кругов | Плагин считает пересечения финишной линии |
| Вычисление strain | Плагин считает `_calc_strain()`, отдаёт как `SensorReading` |
| Что получает платформа | Готовые значения: пульс, скорость, температура, strain, события кругов |

Один плагин, ~50 строк. Вся бизнес-логика (формула strain, трекинг, подсчёт кругов) — внутри плагина. Платформа получает готовые данные и просто отображает их.
