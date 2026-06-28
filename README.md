# delivery_way_ecom
Доставка внешним партнером в интернет-магазине

# System Design Case: Новый способ доставки для интернет-магазина

## Контекст

Интернет-магазину необходимо добавить новый способ доставки, предоставляемый внешним логистическим партнёром.
Цель кейса — спроектировать процесс оформления заказа с новой доставкой, интеграции между системами, API, события, статусы, обработку ошибок и компенсационные действия.

## Что спроектировано

- BPMN-процесс оформления заказа с внешней доставкой
- Sequence diagram для основного сценария
- Sequence diagram для обновления статуса доставки
- API для создания доставки и получения статусов
- События для асинхронной интеграции
- Retry policy
- Компенсации

## Основные системы

- Order Service
- Delivery Integration Service
- Back-office Service
- External Delivery Partner
- Kafka

## Бизнес, функциональные и нефункциональные требования

[Перейти к описанию](/initial-requirements.md)

## BPMN схема
![BPMN схема](/diagrams/BPMN_delivery.png)

## Интеграционная схема

![Интеграционнная схема](/integration_scheme.png)

## Kafka, topics, настройки гарантий доставки и идемпотентность

[Перейти к описанию](/kafka-and-edge-cases.md)

## Диаграммы

![Sequence diagram](/diagrams/sequence-diagram.png)
![Activity diagram](/diagrams/activity-diagram.png)

