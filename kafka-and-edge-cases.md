# Kafka, краевые кейсы и идемпотентность consumer-а на примере Delivery Integration Service

## Контекст

Данный раздел описывает топики Kafka и обработку краевых кейсов для этапа создания доставки у внешнего партнёра.

Процесс начинается после публикации события DELIVERY_CREATION_REQUESTED в Kafka. Основной consumer события - Delivery Integration Service.

## Используемые Kafka-топики

| Топик | Назначение | Producer |	Consumer | 
|---|---|---|---|
|delivery_creation_requests|	Запросы на создание доставки|	Order Service |Delivery Integration Service|
|delivery_events|	Успешные события по доставке|	Delivery Integration Service|	Order Service|
|delivery_failed_events|	Ошибки создания доставки|	Delivery Integration Service|	Order Service|
|delivery_creation_requests_dlq|	Сообщения, которые не удалось обработать технически|	Delivery Integration Service / Kafka error handler |	Back-office service |

#### 2.1. delivery_creation_requests

| Параметр |	Значение | Комментарий |
|---|---|---|
|Key |	orderId | Позволяет сохранять порядок обработки событий по одному заказу в рамках одной партиции. |
|Partitions |	6 | Количество партиций|
|Replication factor |	3 | Количество копий каждой партиции (включая оригинал), которые будут распределены по разным брокерам |
|Retention|	7 дней |Срок хранения |
|Cleanup policy|	delete | Политика удаления (delete (по умолчанию) — удаляет данные по истечении заданного времени или при превышении лимита размера сегмента, compact — сжимает данные, оставляя только самое последнее значение для каждого ключа сообщения|

##### Producer settings для `Order Service`

| Параметр | Значение | Зачем нужно |
|---|---|---|
| `acks` | `all` | Producer считает сообщение успешно отправленным только после подтверждения записи Kafka и реплик |
| `enable.idempotence` | `true` | Защищает от дублей при повторной отправке producer’ом |
| `retries` | `enabled` | Producer повторяет отправку при временной ошибке Kafka |

##### Consumer settings для `Delivery Integration Service`

| Параметр | Значение | Зачем нужно |
|---|---|---|
| `group.id` | `delivery-integration-service` | Consumer group для обработки заявок на создание доставки |
| `enable.auto.commit` | `false` | Offset не фиксируется автоматически |
| Offset commit | После успешной обработки сообщения | Сообщение считается обработанным только после завершения бизнес-операции |
| `auto.offset.reset` | `earliest` | Определяет, откуда читать сообщения при первом запуске consumer group |
| `max.poll.records` | Например, `50` | Ограничивает количество сообщений, получаемых за один poll |
| Error handling | Retry, затем DLQ | Технически необработанные сообщения переносятся в DLQ |

#### 2.2. delivery_events

| Параметр |	Значение | Комментарий |
|---|---|---|
|Key|	orderId| |
|Partitions|	6| |
|Replication factor|	3| |
|Retention|	7 дней| |
|Cleanup policy|	delete| |
|Consumer group	| order-service| |

##### Producer settings для `Delivery Integration Service`
Аналогично пункту "Producer settings для `Order Service`"
##### Consumer settings для `Order Service`
Аналогично пункту "Consumer settings для `Delivery Integration Service`"

#### 2.3. delivery_failed_events

| Параметр |	Значение | Комментарий |
|---|---|---|
|Key|	orderId| |
|Partitions|	6| |
|Replication factor|	3| |
|Retention|	14 дней| |
|Cleanup policy|	delete| |
|Consumer group|	order-service| |

##### Producer settings для `Delivery Integration Service`
Аналогично пункту "Producer settings для `Order Service`"
##### Consumer settings для `Order Service`
Аналогично пункту "Consumer settings для `Delivery Integration Service`"

#### 2.4. delivery_creation_requests_dlq

| Параметр |	Значение | Комментарий |
|---|---|---|
|Key|	orderId| |
|Partitions|	6| |
|Replication factor|	3| |
|Retention|	30 дней| |
|Cleanup policy|	delete| |

## 3. Событие DELIVERY_CREATION_REQUESTED

Событие публикуется, когда заказ готов к созданию доставки.

``` json
{
  "eventId": "evt-123",
  "eventType": "DELIVERY_CREATION_REQUESTED",
  "occurredAt": "2026-06-28T12:00:00Z",
  "orderId": "ORD-100500",
  "customer": {
    "name": "Ivan Ivanov",
    "phone": "+79990000000"
  },
  "deliveryAddress": {
    "city": "Moscow",
    "street": "Tverskaya",
    "house": "1",
    "flat": "10"
  },
  "deliverySlot": {
    "date": "2026-07-01",
    "from": "10:00",
    "to": "14:00"
  },
  "items": [
    {
      "sku": "iphone-15-128-black",
      "quantity": 1,
      "weight": 0.3,
      "volume": 0.002
    }
  ]
}
```

## 4. Краевые кейсы

#### 4.1. Дубль события `DELIVERY_CREATION_REQUESTED`

Сценарий:
Delivery Integration Service получил повторное событие с тем же eventId или orderId.

Обработка:

* проверить, обрабатывалось ли событие ранее;
* проверить, есть ли уже созданная доставка по orderId;
* если доставка уже создана — не отправлять повторный запрос партнёру, пдтвержить офсет;

Для контроля повторной обработки использовать таблицу:

| Поле | Описание |
|---|---|
|eventId|	Идентификатор события|
|orderId|	Идентификатор заказа|
|processingStatus|	Статус обработки события|
|partnerDeliveryId|	Идентификатор доставки у партнёра|
|createdAt|	Дата создания записи|
|updatedAt|	Дата последнего обновления|
|errorCode|	Код ошибки, если обработка завершилась ошибкой|

Статусы обработки:

|Статус|Описание|
|---|---|
|PROCESSED|	Событие успешно обработано|
|FAILED_RETRYABLE|	Ошибка временная, можно повторить|
|FAILED_FINAL|	Ошибка финальная, автоматический retry не требуется|

#### 4.2. Некорректные данные в событии

Сценарий:
В событии отсутствует обязательное поле: orderId, адрес, телефон клиента, состав заказа или временной слот.

Ожидаемое поведение:
Доставка не создаётся.

Обработка:

* зафиксировать причину ошибки;
* опубликовать событие DELIVERY_CREATION_FAILED;

#### 4.3. Партнёр временно недоступен

Сценарий:
Внешний партнёр не отвечает, возвращает timeout или 5xx.

Ожидаемое поведение:
Система должна выполнить повторные попытки создания доставки.

Обработка:

* выполнить retry согласно политике повторных попыток;
* если retry успешен — опубликовать DELIVERY_CREATED;
* если retry исчерпан — опубликовать DELIVERY_CREATION_FAILED.

#### 4.4. Партнёр отказал в создании доставки

Сценарий:
Партнёр вернул бизнес-ошибку: регион недоступен, слот недоступен, превышен вес или объём заказа.

Ожидаемое поведение:
Автоматически повторять запрос не нужно.

Обработка:

* сохранить код и описание ошибки партнёра;
* опубликовать DELIVERY_CREATION_FAILED;
* перевести заказ в ручную обработку.

#### 4.5. Ответ партнёра потерялся

Сценарий:
Delivery Integration Service отправил запрос партнёру, партнёр создал доставку, но ответ не был получен из-за timeout.

Риск:
При повторном запросе может быть создана дубль-доставка.

Ожидаемое поведение:
Повторная обработка не должна создавать дубль доставки.

Обработка:

* использовать ключ идемпотентности при вызове партнёра;
* перед повторным созданием проверить наличие доставки по orderId;
* если доставка уже создана — сохранить partnerDeliveryId и опубликовать DELIVERY_CREATED.

#### 4.6. Сообщение не удалось обработать технически

Сценарий:
Consumer не может обработать сообщение из-за технической ошибки: ошибка парсинга, недоступность БД, ошибка маппинга, падение сервиса.

Ожидаемое поведение:
После исчерпания попыток обработки сообщение должно быть помещено в DLQ.

Обработка:

* выполнить повторные попытки обработки сообщения;
* если попытки исчерпаны — переместить сообщение в delivery_creation_requests_dlq;
* сообщение из DLQ должно быть доступно для ручного анализа и переобработки.
