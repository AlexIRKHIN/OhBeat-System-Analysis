# Асинхронное взаимодействие (RabbitMQ)

Core API и Media Worker общаются только через RabbitMQ: команда на обработку уходит в одну сторону, результат — в другую. Worker не пишет в базу Core API, поэтому остается изолированным сервисом. Событие публикуется через transactional outbox: смена статуса бита и запись события происходят в одной транзакции, и событие не теряется, если брокер временно недоступен.

---

### 1. Топология

| Параметр | Команда на обработку | Результат обработки |
| --- | --- | --- |
| Exchange | `media.events` (direct) | `media.events` (direct) |
| Routing key | `beat.media.process` | `beat.media.processed` |
| Очередь | `beat_processing_queue` | `beat_results_queue` |
| Producer | Core API (через outbox) | Media Worker |
| Consumer | Media Worker | Core API |
| Когда отправляется | После подтверждения загрузки ([FR-7](../01_Requirements/FR_NFR.md#12-синхронные-проверки-core-api)) | После проверки и конвертации ([FR-9–FR-11](../01_Requirements/FR_NFR.md#13-асинхронная-обработка-media-worker)) |

---

### 2. Transactional outbox

1. При подтверждении загрузки Core API в одной транзакции меняет статус бита на `processing` и вставляет строку в таблицу `outbox`.
2. Фоновый процесс (relay) раз в секунду выбирает неопубликованные строки, публикует их в RabbitMQ с publisher confirms и проставляет `published_at`.
3. Если relay упал после публикации, но до отметки, сообщение уйдет повторно. Доставка — «как минимум один раз», поэтому обработчики идемпотентны (см. пункт 4).

---

### 3. Сообщение-команда beat.media.process

| Поле | Тип | Обязательность | Описание |
| --- | --- | --- | --- |
| `event_id` | UUID | Required | Уникальный ID события для трассировки и идемпотентности |
| `event_type` | String | Required | `beat.media.process` |
| `occurred_at` | String (ISO 8601, UTC) | Required | Время создания события |
| `beat_id` | Integer | Required | ID бита |
| `producer_id` | Integer | Required | ID продюсера |
| `files.wav_key` | String | Required | Ключ мастер-WAV во временном бакете |
| `files.stems_key` | String | Optional | Ключ ZIP со стемами, null если нет |
| `files.cover_key` | String | Required | Ключ обложки |
| `options.voice_tag` | Boolean | Required | Накладывать ли войстег на превью |
| `options.tag_interval_sec` | Integer | Required | Интервал повтора войстега, секунды |

```json
{
  "event_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "event_type": "beat.media.process",
  "occurred_at": "2026-10-04T08:00:00Z",
  "beat_id": 112233,
  "producer_id": 402,
  "files": {
    "wav_key": "raw/112233/master.wav",
    "stems_key": "raw/112233/stems.zip",
    "cover_key": "raw/112233/cover.jpg"
  },
  "options": { "voice_tag": true, "tag_interval_sec": 20 }
}
```

---

### 4. Сообщение-результат beat.media.processed

```json
{
  "event_id": "5f0c2a7e-8d41-4c1b-9a63-7e2b4d9c1f08",
  "event_type": "beat.media.processed",
  "occurred_at": "2026-10-04T08:01:42Z",
  "source_event_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "beat_id": 112233,
  "result": "rejected",
  "duration_sec": null,
  "outputs": null,
  "error": { "code": "ARCHIVE_CONTAINS_NON_AUDIO", "message": "В архиве найдены файлы, кроме WAV: readme.exe" }
}
```

При успехе `result: "processed"`, `duration_sec` заполнен, а `outputs` содержит постоянные ключи: `master`, `stems`, `clean_mp3`, `preview_mp3`, `cover`.

**Идемпотентность.** Worker пишет результаты по детерминированным ключам (`beats/112233/preview_v1.mp3`), поэтому повторная обработка перезаписывает те же объекты. Core API применяет результат, только если бит еще в статусе `processing`; повторное сообщение подтверждается (ACK) без изменений.

---

### 5. Ошибки и отказоустойчивость

| Ситуация | Пример | Поведение Worker | Итог |
| --- | --- | --- | --- |
| Файл не прошел проверку | Длительность больше 10 минут, вирус в ZIP | Публикует результат `rejected` с причиной, затем ACK | Бит `rejected`, продюсер видит причину |
| Временный сбой | Таймаут S3, обрыв сети | NACK без повторной постановки; сообщение уходит в очередь задержки и возвращается через 30 с, 2 мин, 10 мин | Не более 3 попыток |
| Попытки исчерпаны или непредвиденная ошибка | Падение FFmpeg | После 3-й попытки сообщение попадает в `beat_processing_dlq` | Алерт дежурному, разбор вручную; бит остается в `processing` |

Очереди задержки — отдельные очереди с `x-message-ttl` и `x-dead-letter-exchange`, указывающим обратно на `media.events`. Номер попытки определяется по заголовку `x-death`.