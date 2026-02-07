## Product Choice

Yandex Go

<https://go.yandex>

Ordering taxis, food, groceries, goods, and parcel delivery, as well as car sharing, scooter rental, and transport schedules.

## Main components

![Yandex Go Component Diagram](diagrams/out/yandex-go/architecture-component/Component%20Diagram.svg)

[Yandex Go Component Diagram Code](diagrams/src/yandex-go/architecture-component.puml)

- Mobile App
The user-facing client for riders/drivers that creates ride requests, displays status/ETA, and communicates with the REST API. It handles local UI logic, input validation, and pushes events to the backend via the API Gateway.

- API Gateway
Central ingress point that authenticates, authorizes, rate-limits and routes incoming REST/gRPC calls to the appropriate backend services. It also performs protocol translation, simple aggregation, and can act as a choke point for monitoring and request-level routing.

- User Service
Manages user accounts, profiles, authentication tokens and basic user settings; reads/writes persistent user data to the operational DB and caches hot records in Redis. It is the authoritative source for user identity and account state.

- Dispatch Service
Core matching and orchestration engine that pairs riders with drivers, tracks driver states, and enforces assignment logic and priorities; it consumes maps/pricing info and publishes events to the message broker for downstream processing. High-throughput and low-latency — its decisions drive the product’s core behavior.

- Payment Service
Handles payment flows, transaction records, reconciliations and integration with external gateways (Yandex Pay). It persists payment events to the DB and emits events (success/failure/refunds) for analytics and notifications.

- Maps & Routing Service
Provides routing, ETA and geospatial computations by integrating with external map data (Yandex Maps). Other services (dispatch, pricing) query it for distances, travel times and route alternatives.

- Pricing Service
Computes fares using inputs like distance/ETA from maps, demand signals from analytics, and business rules; it can apply surge/dynamic pricing and pushes calculated prices back to dispatch and the client. It also reads aggregated historical data from the analytics DB to inform algorithms.

- Notification Service
Sends push notifications, SMS and emails to users and drivers based on events (ride accepted, payment succeeded, etc.). It subscribes to the message broker for event-driven delivery and handles retries/delivery status.

- Message Broker (Kafka / Logbroker)
Asynchronous event backbone that decouples services: transports user/dispatch/payment events to consumers (analytics, notifications, etc.), enables replay, and provides durable, ordered event streams for scaling pipelines.

## Data flow

![Yandex Go Component Diagram](diagrams/out/yandex-go/architecture-sequence/Sequence%20Diagram.svg)

[Yandex Go Component Diagram Code](diagrams/src/yandex-go/architecture-sequence.puml)

Booking & Async Dispatch Flow

Client → API Gateway
Пользователь в мобильном приложении выбирает класс машины и способ оплаты. App отправляет RPC-запрос createOrder(class, payment_id) на API Gateway.

API Gateway → Dispatch Service
Gateway пересылает запрос на Dispatch Service, который отвечает за поиск и назначение водителя.

Dispatch Service → User Service
Сервис проверяет, действителен ли способ оплаты пользователя, вызывая User Service. User Service возвращает результат проверки (OK).

Dispatch Service → Operational DB и Cache

DB: создаётся запись заказа со статусом SEARCHING.

Cache (State Cache): DispatchSvc выполняет геопространственный поиск водителей поблизости; кэш возвращает список кандидатов.

Internal Matching
Внутри Dispatch Service запускается алгоритм выбора водителя: учитываются расстояние, рейтинг, время отклика и загруженность.

Update & Event Publication

Заказ обновляется в DB со статусом ASSIGNED.

Событие RideAssigned публикуется в Kafka Event Bus для асинхронного уведомления других сервисов.

Gateway → App
Gateway возвращает подтверждение создания заказа. Клиент обновляет интерфейс (Searching UI), показывая, что система ищет водителя.

Async Notification

Push Service подписан на события Kafka. Получает RideAssigned.

Отправляет push-уведомление пользователю («Driver Found»).

App обновляет карту и показывает позицию водителя на экране.

## Deployment

![Yandex Go Component Diagram](diagrams/out/yandex-go/architecture-deployment/Deployment%20Diagram.svg)

[Yandex Go Component Diagram Code](diagrams/src/yandex-go/architecture-deployment.puml)

Client Tier

Yandex Go Mobile App — на смартфонах пользователей (iOS/Android).

Yandex Go Web App — в браузере на компьютерах пользователей.

Cloud Infrastructure

API Gateway — за Load Balancer в облаке, принимает HTTPS-запросы от клиентов.

Core Services (User, Payment, Dispatch, Pricing, Maps & Routing, Notification) — внутри Kubernetes Cluster (Application Tier), каждый сервис в контейнерах/Pod’ах.

Redis Cache — в отдельном Redis Cluster, для быстрого доступа к часто используемым данным.

Operational DB (YDB) и Analytics DB (ClickHouse) — в Data Storage Cluster, где YDB хранит транзакционные и оперативные данные, а ClickHouse — аналитические/агрегированные данные.

Kafka Event Bus — в Message Broker Cluster, используется для асинхронной передачи событий между сервисами.

Yandex Go: "I assume the pricing service handles surge pricing calculations based on demand and supply in real-time."

Yandex Go: "How does the actual load balancing mechanism work between the microservices in production?"
