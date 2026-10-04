# 🔌 Спецификация API-контрактов (REST API Contracts)

Все эндпоинты платформы имеют базовый адрес `https://api.ohbeat.com/v1`, принимают и отдают данные в формате JSON и требуют передачи заголовка `Authorization: Bearer <access_token>`, кроме публичных запросов каталога, авторизации и входящих вебхуков от платежного шлюза. 

Ошибки возвращаются в едином стандартизированном формате с машиночитаемым кодом и указанием пути к некорректному полю:

```json
{
  "error_code": "VALIDATION_ERROR",
  "message": "Некорректные данные запроса",
  "details": [
    { "field": "licenses[1].price", "issue": "Цена должна быть от 100 до 500000" }
  ]
}
```

---

## 1. Реестр эндпоинтов

| Метод и путь | Назначение | Инициатор вызова |
| :--- | :--- | :--- |
| `POST /beats` | Создать бит и получить формы прямой загрузки в S3 | Продюсер |
| `POST /beats/{id}/confirm` | Подтвердить факт загрузки файлов | Продюсер |
| `GET /beats` | Каталог битов с фильтрацией и пагинацией | Публичный (Гость / Артист) |
| `PUT /beats/{id}/licenses/{code}` | Изменить цену или активность лицензии | Продюсер (владелец бита) |
| `POST /orders` | Создать заказ и установить резерв | Артист |
| `POST /orders/{id}/payments` | Создать платеж в ЮKassa | Артист |
| `GET /orders/{id}` | Запросить статус заказа (опрос после оплаты) | Артист |
| `GET /orders/{id}/downloads` | Получить временные ссылки на скачивание файлов и договора | Артист (владелец заказа) |
| `POST /payments/webhook` | Прием уведомлений об изменении статуса оплаты | ЮKassa |
| `POST /auth/login` | Аутентификация пользователя и получение пары токенов | Все |
| `POST /auth/refresh` | Обновление пары токенов доступа | Все |

> Эндпоинты выпуска и проверки кодов подтверждения для эксклюзивных прав (`/orders/{id}/signature/...`) описаны в [RFC: Модуль цифровой подписи (ПЭП)](../02_Architecture/RFC_Digital_Signature.md).

---

## 2. Спецификация методов API

### 2.1. Создание бита (`POST /beats`)
Принимает метаданные и спецификацию загружаемых файлов, создает запись о бите в статусе `uploading` и возвращает Presigned POST URL для каждого файла. Сами бинарные файлы фронтенд передает напрямую в S3, минуя сервер приложений.

Имя поля тональности — `musical_key`, единое для API, экранных форм и физической БД.

| Поле | Тип | Обязательность | Ограничения и правила |
| :--- | :--- | :--- | :--- |
| `title` | String | Required | 3–60 символов, пробелы по краям обрезаются |
| `genres` | Array of String | Required | 1–3 значения из системного справочника жанров |
| `bpm` | Integer | Required | Целое число от 40 до 300 |
| `musical_key` | String | Required | Значение из справочника тональностей (например, `Dm`, `C#`) |
| `description` | String | Optional | До 1000 символов, допускается `null` |
| `licenses` | Array of Object | Required | Минимум одна активная лицензия в массиве |
| `licenses[].code` | String | Required | Допустимые коды: `basic`, `premium`, `trackout`, `exclusive` |
| `licenses[].price` | Integer | Required | От 100 до 500 000, рубли |
| `files` | Object | Required | Описание загружаемых файлов |
| `files.wav` | Object | Required | `name`, `size_bytes` (до 200 МБ), `content_type: audio/wav` |
| `files.cover` | Object | Required | `name`, `size_bytes` (до 5 МБ), `content_type`: `image/jpeg`, `image/png` или `image/webp` |
| `files.stems` | Object | Conditional | Обязателен при наличии лицензии `trackout` или `exclusive`; до 1 ГБ, `application/zip` |

#### Пример запроса:
```json
{
  "title": "Neon Nights",
  "genres": ["Synthwave"],
  "bpm": 120,
  "musical_key": "Dm",
  "description": null,
  "licenses": [
    { "code": "basic", "price": 1500 },
    { "code": "premium", "price": 3000 }
  ],
  "files": {
    "wav": { "name": "neon_master.wav", "size_bytes": 61234567, "content_type": "audio/wav" },
    "cover": { "name": "cover.jpg", "size_bytes": 845000, "content_type": "image/jpeg" }
  }
}
```

#### Пример ответа `201 Created`:
```json
{
  "id": 112233,
  "status": "uploading",
  "upload_expires_at": "2026-10-04T12:00:00Z",
  "uploads": {
    "wav": { 
      "url": "[https://s3.ohbeat.com/temp-uploads](https://s3.ohbeat.com/temp-uploads)", 
      "fields": { "key": "raw/112233/master.wav", "policy": "...", "x-amz-signature": "..." } 
    },
    "cover": { 
      "url": "[https://s3.ohbeat.com/temp-uploads](https://s3.ohbeat.com/temp-uploads)", 
      "fields": { "key": "raw/112233/cover.jpg", "policy": "...", "x-amz-signature": "..." } 
    }
  }
}
```

> **Обоснование Presigned POST:** Выбран вместо Presigned PUT, так как политика безопасности S3 задает условие `content-length-range`: S3 на аппаратном уровне отклонит загрузку файла сверх установленного лимита.

**Возможные ошибки:**
* `400 VALIDATION_ERROR` — значение поля выходит за допустимые границы или содержит слова из стоп-листа (`issue: "Запрещенное слово"`).
* `401 AUTH_ERROR` — токен авторизации отсутствует или истек.
* `403 FORBIDDEN` — у пользователя отсутствует роль Продюсера.

---

### 2.2. Подтверждение загрузки (`POST /beats/{id}/confirm`)
Тело запроса отсутствует. Сервер выполняет синхронную проверку наличия файлов в S3 через `HeadObject` согласно [FR-7](../01_Requirements/FR_NFR.md) и ставит задачу в очередь обработки.

**Коды ответов:**
* `202 Accepted` — `{ "id": 112233, "status": "processing" }`.
* `409 INVALID_STATUS` — бит находится не в статусе `uploading` (уже подтвержден или удален).
* `422 FILES_MISSING` — в массиве `details` перечислены отсутствующие или превысившие лимит файлы.

---

### 2.3. Каталог битов (`GET /beats`)

Параметры строки запроса (Query Params):  
`GET /beats?genre=Hip-Hop&bpm_min=80&bpm_max=100&musical_key=Am&price_max=2000&sort=newest&limit=20&cursor=...`

Все параметры являются опциональными. Фильтр `price_max` применяется к наименьшей стоимости среди активных лицензий бита. В выдачу попадают сущности в статусах `active` и `reserved` (для зарезервированных проставляется флаг `is_reserved: true`). Пагинация реализована на базе курсоров (`cursor`), что обеспечивает консистентность выдачи при добавлении новых битов в реальном времени.

#### Пример ответа `200 OK`:
```json
{
  "items": [
    {
      "id": 112234,
      "title": "Street Vibes",
      "producer": { "id": 402, "display_name": "Kirill Beats" },
      "genres": ["Hip-Hop"],
      "bpm": 90,
      "musical_key": "Am",
      "duration_sec": 184,
      "cover_url": "[https://cdn.ohbeat.com/covers/112234_v1.jpg](https://cdn.ohbeat.com/covers/112234_v1.jpg)",
      "preview_url": "[https://cdn.ohbeat.com/previews/112234_v1.mp3](https://cdn.ohbeat.com/previews/112234_v1.mp3)",
      "is_reserved": false,
      "licenses": [
        { "code": "basic", "price": 1500 },
        { "code": "premium", "price": 3000 }
      ]
    }
  ],
  "next_cursor": "eyJpZCI6MTEyMjM0fQ"
}
```

---

### 2.4. Изменение цены лицензии (`PUT /beats/{id}/licenses/{code}`)
Позволяет точечно модифицировать стоимость и статус активности одной конкретной лицензии без необходимости передачи всей матрицы цен.

#### Пример запроса:
```json
{ "price": 2000, "is_active": true }
```

#### Пример ответа `200 OK`:
Возвращает объект бита с актуализированным списком цен:
```json
{
  "id": 112233,
  "title": "Neon Nights",
  "licenses": [
    { "code": "basic", "price": 1500, "is_active": true },
    { "code": "premium", "price": 2000, "is_active": true }
  ]
}
```

**Ошибки:**
* `403 FORBIDDEN` — попытка изменения чужого бита.
* `404 NOT_FOUND` — бит или лицензия не найдены.
* `409 BEAT_SOLD` — эксклюзив продан, операции с лицензиями заблокированы.

> Изменение цены не затрагивает ранее созданные заказы: цена фиксируется на момент создания в поле `order_items.price_at_purchase`.

---

### 2.5. Создание заказа и платежа

#### Создание заказа (`POST /orders`)
Инициирует процесс оформления покупки и атомарно устанавливает резерв на позиции Exclusive согласно [FR-18](../01_Requirements/FR_NFR.md).

**Тело запроса:**
```json
{ 
  "items": [ { "beat_id": 112233, "license": "exclusive" } ], 
  "offer_accepted": true 
}
```

**Ответ `201 Created`:**
```json
{ 
  "order_id": 889900, 
  "status": "pending", 
  "total": 25000, 
  "requires_signature": true, 
  "reserved_until": "2026-10-04T11:37:00Z" 
}
```

**Ошибки:** `409 BEAT_RESERVED`, `409 BEAT_SOLD`, `409 PRICE_CHANGED` (в `details` передаются обновленные цены).

#### Создание платежа (`POST /orders/{id}/payments`)
Формирует платежную транзакцию в шлюзе ЮKassa по правилам [FR-20](../01_Requirements/FR_NFR.md).

**Ответ `200 OK`:**
```json
{ 
  "payment_id": "22e12f66-000f-5000-8000-18db351245c7", 
  "confirmation_url": "[https://yoomoney.ru/checkout/payments/v2/contract?orderId=](https://yoomoney.ru/checkout/payments/v2/contract?orderId=)..." 
}
```

**Ошибки:**
* `410 RESERVATION_EXPIRED` — 15-минутный резерв на эксклюзив истек.
* `428 SIGNATURE_REQUIRED` — заказ содержит Exclusive-лицензию, но договор еще не подписан кодом ПЭП.

---

### 2.6. Статус заказа и скачивание

* **Статус заказа (`GET /orders/{id}`):**  
  Возвращает `{ "order_id": 889900, "status": "paid" }`. Жизненный цикл статусов: `pending`, `paid`, `expired`, `canceled` (см. [Модель данных PostgreSQL](../05_Database/Database_Schema.md)).

* **Получение ссылок на скачивание (`GET /orders/{id}/downloads`):**  
  Генерирует временные presigned-ссылки со сроком жизни 1 час для файлов купленной лицензии (согласно [FR-28](../01_Requirements/FR_NFR.md) и [типам лицензий](../01_Requirements/User_Stories.md#22-типы-лицензий)). Ссылки формируются «на лету» при каждом обращении.

#### Пример ответа `200 OK`:
```json
{
  "expires_at": "2026-10-04T12:22:00Z",
  "items": [
    {
      "beat_id": 112233,
      "license": "exclusive",
      "files": [
        { "kind": "mp3", "url": "[https://s3.ohbeat.com/private-media/beats/112233/clean.mp3?X-Amz-Signature=](https://s3.ohbeat.com/private-media/beats/112233/clean.mp3?X-Amz-Signature=)..." },
        { "kind": "wav", "url": "[https://s3.ohbeat.com/private-media/beats/112233/master.wav?X-Amz-Signature=](https://s3.ohbeat.com/private-media/beats/112233/master.wav?X-Amz-Signature=)..." },
        { "kind": "stems", "url": "[https://s3.ohbeat.com/private-media/beats/112233/stems.zip?X-Amz-Signature=](https://s3.ohbeat.com/private-media/beats/112233/stems.zip?X-Amz-Signature=)..." },
        { "kind": "contract", "url": "[https://s3.ohbeat.com/private-media/contracts/889900-1.pdf?X-Amz-Signature=](https://s3.ohbeat.com/private-media/contracts/889900-1.pdf?X-Amz-Signature=)..." }
      ]
    }
  ]
}
```

**Ошибки:** `403 FORBIDDEN` (заказ оформлен на другого пользователя); `409 ORDER_NOT_PAID` (заказ еще не оплачен).

---

### 2.7. Webhook ЮKassa (`POST /payments/webhook`)
Принимает входящие системные оповещения платежного шлюза. Endpoint не требует токена авторизации; проверка легитимности запроса осуществляется по списку доверенных IP-адресов шлюза и последующим подтверждающим запросом `GET /v3/payments/{id}` согласно [FR-22](../01_Requirements/FR_NFR.md).

#### Пример уведомления о холдировании средств (`waiting_for_capture`):
```json
{
  "type": "notification",
  "event": "payment.waiting_for_capture",
  "object": {
    "id": "22e12f66-000f-5000-8000-18db351245c7",
    "status": "waiting_for_capture",
    "paid": true,
    "amount": { "value": "25000.00", "currency": "RUB" },
    "metadata": { "order_id": "889900" }
  }
}
```

> **Обработка ответа:** Сервер обязан вернуть HTTP-статус `200 OK` сразу после сохранения события в таблицу `payment_events`, в том числе при повторной доставке. Любой отличный статус воспринимается провайдером как сбой с последующей повторной отправкой уведомления.

---

### 2.8. Аутентификация и управление сессиями

* **Вход в систему (`POST /auth/login`):**  
  Принимает тело `{ "email": "artist@example.com", "password": "..." }`. Пароль передается внутри TLS-соединения.

#### Пример ответа `200 OK`:
```json
{ 
  "access_token": "eyJhbGciOiJIUzI1NiIs...", 
  "refresh_token": "dGhpcy1pcy1hLXJlZnJlc2g...", 
  "token_type": "Bearer", 
  "expires_in": 3600 
}
```

Поле `expires_in` регламентирует время жизни access-токена в секундах (1 час).  
**Ошибки:** 
* `401 INVALID_CREDENTIALS` — возвращает единое сообщение «Неверный email или пароль» для предотвращения раскрытия наличия учетной записи в системе.
* `429 TOO_MANY_ATTEMPTS` — блокировка при превышении порога в 5 неудачных попыток за 15 минут.

* **Обновление токена (`POST /auth/refresh`):**  
  Принимает `{ "refresh_token": "..." }`, возвращает новую пару токенов. Использованный refresh-токен немедленно аннулируется (механизм ротации).