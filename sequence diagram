
``` plant uml
@startuml
title Создание доставки через внешнего партнёра

participant "Order Service" as Order
queue "Kafka topic:\ndelivery_creation_requests" as Requests
participant "Delivery Integration Service" as Delivery
participant "External Delivery Partner" as Partner
queue "Kafka topic:\ndelivery_events" as Events
queue "Kafka topic:\ndelivery_failed_events" as Failed
participant "Back-office" as BO

activate Order
Order -> Order: Перевести заказ в DELIVERY_CREATION_PENDING
Order -> Requests: DELIVERY_CREATION_REQUESTED
deactivate Order

activate Requests
Delivery -> Requests: Получить событие
deactivate Requests

activate Delivery
Delivery -> Delivery: Проверить данные заявки

Delivery -> Partner: POST /deliveries

activate Partner
alt Партнёр подтвердил создание доставки
    Partner --> Delivery: 201 Created\npartnerDeliveryId
    Delivery -> Delivery: Сохранить partnerDeliveryId
    Delivery -> Events: DELIVERY_CREATED
    activate Events
    Events -> Order: DELIVERY_CREATED
    deactivate Events
    Order -> Order: Перевести заказ в DELIVERY_CREATED
else Партнёр отказал / вернул ошибку
    Partner --> Delivery: Error
    deactivate Partner
    Delivery -> Failed: DELIVERY_CREATION_FAILED
    activate Failed
    Failed -> Order: DELIVERY_CREATION_FAILED
    deactivate Failed
    Order -> Order: Перевести заказ в MANUAL_REVIEW_REQUIRED
    BO -> Order: Получить заказы для ручной обработки
end
deactivate Delivery


@enduml
```
