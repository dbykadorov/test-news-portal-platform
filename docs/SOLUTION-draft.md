# Медиа-платформа: архитектура

Self-hosted платформа с двумя контурами: **Студия** (редакция) и **Витрина** (сайт для читателей). Контуры изолированы — трафик и сбои на Витрине не трогают Студию. Связь только через Kafka.

**Компоненты**

- **Studio Web App** — интерфейс для редакторов и админов. Контент, таксономия, workflow.
- **Studio BFF** — агрегирует данные под экраны Студии, вызывает сервисы. Поток: Web App → BFF → Gateway → сервисы.
- **Internal Gateway** — auth, маршрутизация, rate limiting. Принимает трафик от BFF, не от браузера.
- **Public Web App** — сайт для читателей. Материалы, поиск, ленты.
- **Public Ingress** — единая точка входа Delivery. CDN → Ingress → API.
- **Public API / BFF** — отдаёт данные Витрине. Читает из Read Model, Redis, поиска. К Студии не ходит.

**Правило:** BFF владеет всеми контрактами экранов. Сервисы отдают только внутренние API.

---

## Контекст

Кто пользуется платформой и где границы.

```mermaid
flowchart TB
  Editor[Editor] --> StudioWebApp[Studio Web App]
  OpsAdmin["Ops / Admin"] --> StudioWebApp
  Reader[Reader] --> PublicWebApp[Public Web App]

  subgraph mediaPlatform [Media Platform]
    subgraph studio [Studio]
      StudioWebApp
      SLabel[Editorial / Master]
    end
    subgraph delivery [Delivery]
      PublicWebApp
      DLabel[Public Site]
    end
  end

  mediaPlatform --> IdP["IdP / SSO"]
  mediaPlatform --> ObjectStorage[Object Storage]
  delivery --> CDN["CDN / WAF"]
```

- **Редакторы** — через Studio Web App. Публичный трафик в Студию не идёт.
- **Читатели** — через Public Web App. Редакционные API не трогают.
- **Зависимости:** IdP/SSO (логин), Object Storage (медиа), CDN/WAF (кэш и защита). Контуры масштабируются отдельно.

---

## Компоненты и потоки

Сервисы, хранилища, Kafka.

```mermaid
flowchart TB
  subgraph studio [Studio Contour]
    StudioWebApp[Studio Web App]
    StudioBff[Studio BFF]
    InternalGateway[Internal Gateway]
    ContentService[Content Service]
    WorkflowService[Workflow Service]
    TaxonomyService[Taxonomy Service]
    MediaService[Media Service]
    AuthService[Auth / RBAC]
    AuditService[Audit Service]
    OutboxPublisher[Outbox Publisher]
    StudioPostgres[(Studio PostgreSQL)]
    StudioRedis[(Studio Redis)]
  end

  subgraph delivery [Delivery Contour]
    CdnWaf["CDN / WAF"]
    PublicWebApp[Public Web App]
    PublicIngress[Public Ingress]
    PublicAPI[Public API / BFF]
    ReadModelDB[(Read Model Store)]
    SearchService[Search Service]
    RedisCache[(Redis Cache)]
  end

  subgraph pipeline [Publish Pipeline]
    MessageBroker[Kafka]
    ProjectorConsumer[Projector Consumer]
    SearchIndexer[Search Indexer]
    CacheInvalidator[Cache Invalidator]
    MediaWorkers[Media Workers]
  end

  subgraph shared [Shared Infrastructure]
    MinIO[MinIO / S3]
    OpenSearch[OpenSearch]
    IamService["IAM Service"]
    Observability[Observability]
  end

  StudioWebApp --> StudioBff
  StudioBff --> InternalGateway
  InternalGateway --> ContentService
  InternalGateway --> WorkflowService
  InternalGateway --> TaxonomyService
  InternalGateway --> MediaService
  InternalGateway --> AuthService
  InternalGateway --> AuditService
  WorkflowService --> ContentService
  ContentService --> StudioPostgres
  ContentService --> OutboxPublisher
  OutboxPublisher --> MessageBroker
  AuthService --> StudioRedis
  AuthService --> IamService
  MessageBroker --> ProjectorConsumer
  MessageBroker --> SearchIndexer
  MessageBroker --> CacheInvalidator
  MessageBroker --> MediaWorkers
  ProjectorConsumer --> ReadModelDB
  SearchIndexer --> OpenSearch
  CacheInvalidator --> RedisCache
  MediaWorkers --> MinIO
  MediaWorkers --> MediaService
  MediaService --> MessageBroker
  CdnWaf --> PublicWebApp
  CdnWaf --> PublicIngress
  MinIO --> CdnWaf
  PublicIngress --> PublicAPI
  PublicAPI --> ReadModelDB
  PublicAPI --> RedisCache
  PublicAPI --> SearchService
  SearchService --> OpenSearch
  MediaService --> MinIO
  InternalGateway --> Observability
  PublicAPI --> Observability
  MessageBroker --> Observability
```

- **Студия:** Web App → BFF → Gateway → Content/Workflow/Taxonomy/Media/Auth/Audit. Публикация: Workflow → Content (snapshot + outbox) → Outbox Publisher → Kafka. В Витрину синхронно не шлём.
- **Pipeline:** Outbox Publisher и Media Service пишут в Kafka. Consumers (Projector, Search Indexer, Cache Invalidator, Media Workers) строят read-side: PostgreSQL, OpenSearch, Redis, MinIO. Согласованность — eventual.
- **Витрина:** CDN → Public Web App (HTML) и CDN → Ingress → Public API / BFF (JSON). BFF читает Read Model, Redis, поиск. К Студии не ходит.

**Ключевые правила**

- **Публикация:** только Content Service шлёт события в Delivery (через outbox). Workflow оркестрирует, Taxonomy/Media встраиваются в snapshot.
- **Publish flow:** Workflow → Content (snapshot + outbox) → после OK Content Workflow ставит published. Команды повторяемы, consumers идемпотентны.
- **Auth:** Gateway проверяет права. IAM — пользователи, Auth Service — политики. При падении IAM — запрет на запись.
- **Audit:** HTTP push от сервисов. At-least-once, дедуп по eventId.
- **БД:** каждый сервис пишет в свои таблицы. Чтение чужого — через API или события.
- **Медиа:** публикуем только при готовности всех медиа (strict ready). Media Workers пишут только в Media Service.
- **Идентичность:** `(type, slug)`.
- **События:** одна схема для всех consumers (domain-model).

**Поток публикации**

1. Редактор → Web App → BFF → Gateway → Workflow → Content (snapshot + outbox).
2. Outbox Publisher → Kafka.
3. Consumers → Read Model, OpenSearch, Redis, MinIO.
4. Читатель → CDN → Web App / BFF. BFF читает Read Model, Redis, поиск.

**Публикация (диаграмма)**

```mermaid
sequenceDiagram
    participant Editor
    participant StudioBFF as Studio BFF
    participant Gateway as Internal Gateway
    participant Auth as Auth / RBAC
    participant Workflow as Workflow Service
    participant Content as Content Service
    participant Taxonomy as Taxonomy Service
    participant Postgres as Studio PostgreSQL
    participant Outbox as Outbox Publisher
    participant KafkaBroker as Kafka
    participant Projector as Projector Consumer
    participant ReadModel as Read Model Store

    Editor->>StudioBFF: publish(articleId)
    StudioBFF->>Gateway: POST /v1/workflow/{id}/publish
    Gateway->>Auth: checkAccess(token, publish)
    Auth-->>Gateway: allowed
    Gateway->>Workflow: publish(articleId)
    Workflow->>Content: createSnapshot(articleId)
    Content->>Taxonomy: getTaxonomyData(ids)
    Taxonomy-->>Content: taxonomy values
    Content->>Postgres: BEGIN -> INSERT snapshot outbox -> COMMIT
    Content-->>Workflow: snapshotCreated
    Workflow->>Postgres: UPDATE status = published
    Workflow-->>Gateway: 200 OK
    Gateway-->>StudioBFF: 200 OK
    Note over Outbox,KafkaBroker: Async relay (polling)
    Outbox->>Postgres: SELECT FROM outbox SKIP LOCKED
    Outbox->>KafkaBroker: produce(content-events)
    Outbox->>Postgres: DELETE outbox row
    KafkaBroker->>Projector: consume(content-events)
    Projector->>ReadModel: upsert read model
```

**Auth (диаграмма)**

```mermaid
sequenceDiagram
    participant Editor
    participant StudioBFF as Studio BFF
    participant Gateway as Internal Gateway
    participant Auth as Auth / RBAC
    participant IAM as IAM Service
    participant Redis as Studio Redis

    Editor->>StudioBFF: request + SSO token
    StudioBFF->>Gateway: request + token
    Gateway->>Auth: validateToken(token)
    Auth->>Redis: get cached session
    alt cache hit
        Redis-->>Auth: session context
    else cache miss
        Auth->>IAM: introspect(token)
        IAM-->>Auth: user identity + groups
        Auth->>Redis: cache session (TTL)
    end
    Auth->>Auth: PolicyEngine.check(user, resource, action)
    Auth-->>Gateway: allowed / denied + user context
    alt allowed
        Gateway->>Gateway: route to target service
    else denied
        Gateway-->>StudioBFF: 403 Forbidden
    end
```

**Загрузка медиа (диаграмма)**

```mermaid
sequenceDiagram
    participant Editor
    participant StudioBFF as Studio BFF
    participant Gateway as Internal Gateway
    participant Media as Media Service
    participant MinIO as MinIO
    participant KafkaBroker as Kafka
    participant Workers as Media Workers
    participant ReadModel as Read Model Store

    Editor->>StudioBFF: upload(file, metadata)
    StudioBFF->>Gateway: POST /v1/media/upload
    Gateway->>Media: upload(file, metadata)
    Media->>MinIO: putObject(original)
    MinIO-->>Media: objectId
    Media->>Media: AssetRegistry.register(objectId, metadata)
    Media->>KafkaBroker: produce(media-jobs, {objectId, type})
    Media-->>Gateway: 202 Accepted {assetId, status: processing}
    Note over Workers,KafkaBroker: Async processing
    KafkaBroker->>Workers: consume(media-jobs)
    Workers->>Workers: process (resize/thumbnail/video)
    Workers->>MinIO: putObject(derivatives)
    Workers->>Media: callback(assetId, status: ready, urls)
    Media->>Media: DerivativeTracker.update(ready)
    Workers->>ReadModel: upsert media metadata
```

**Не в v1:** уведомления (пока polling + Audit), комментарии (нужна модерация, auth читателей), топ статей (нужен clickstream).

**Стек**

| Назначение | Решение |
|------------|---------|
| БД Студии | PostgreSQL, HA — Patroni |
| Read Model | PostgreSQL (отдельный кластер) |
| Кэш | Redis |
| Очереди | Kafka |
| Медиа | MinIO / S3 |
| Поиск | OpenSearch |
| Auth | IAM + Auth Service |
| Edge | CDN / WAF (Varnish, Nginx или Cloudflare) |
| Мониторинг | Prometheus, Grafana, Loki, Tempo |
| Аналитика | ClickHouse |

**PostgreSQL (Студия)**

1 primary + 2 sync standby. Patroni + etcd. PgBouncer перед БД. RPO=0, RTO < 30s.

```mermaid
flowchart TB
  subgraph services [Studio Services]
    ContentSvc[Content Service]
    WorkflowSvc[Workflow Service]
    TaxonomySvc[Taxonomy Service]
    MediaSvc[Media Service]
    AuthSvc[Auth / RBAC]
    AuditSvc[Audit Service]
  end

  subgraph pooling [Connection Pooling]
    PgBouncer[PgBouncer]
  end

  subgraph pgCluster ["PostgreSQL HA Cluster"]
    Primary[Primary]
    Standby1[Sync Standby 1]
    Standby2[Sync Standby 2]
    Patroni[Patroni + etcd]
  end

  services --> PgBouncer
  PgBouncer --> Primary
  AuditSvc -.->|"read queries"| Standby1
  Primary --> Standby1
  Primary --> Standby2
  Patroni --> Primary
  Patroni --> Standby1
  Patroni --> Standby2
  Primary --> MinIO["MinIO WAL Archive"]
```

**Kafka**

3 брокера, KRaft. RF=3. Topics: content-events, media-jobs, DLQ. Retention 7/3 дня. Consumer groups на каждый consumer.

```mermaid
flowchart TB
  subgraph producers [Producers]
    OutboxPub[Outbox Publisher]
    MediaSvc[Media Service]
  end

  subgraph kafkaCluster ["Kafka Cluster (KRaft)"]
    Broker1[Broker 1]
    Broker2[Broker 2]
    Broker3[Broker 3]
  end

  subgraph consumers [Consumer Groups]
    ProjectorCG[projector-cg]
    SearchCG[search-indexer-cg]
    CacheCG[cache-invalidator-cg]
    MediaCG[media-workers-cg]
  end

  subgraph dlqTopics ["DLQ Topics"]
    OutboxDlq[outbox-dlq]
    MediaDlq[media-jobs-dlq]
    PipelineDlq[pipeline-dlq]
  end

  producers --> kafkaCluster
  kafkaCluster --> consumers
  consumers --> dlqTopics
  kafkaCluster --> Observability[Observability]
```

**Audit:** HTTP push, bounded buffer + retry + fallback file. At-least-once.

**API:** `/v1/...` в пути. Breaking changes — новая major.

**Балансировка:** CDN → edge, Ingress → BFF, K8s Service → pods, PgBouncer → БД, Kafka → partitions. Без service mesh в v1. HPA для stateless.

**Read Model:** PostgreSQL, отдельный кластер. Primary + replicas + PgBouncer. Projector пишет, BFF читает.

**Деградация:** CDN — stale лучше ошибки. Circuit breakers на BFF. Read Model упал → 503 + CDN кэш. Поиск упал → ленты работают. Контуры изолированы.

**Метрики:** Prometheus (ops) + ClickHouse (бизнес: публикации, активность, просмотры). Kafka topic business-metrics → Metrics Consumer → ClickHouse → Grafana.
