# System Design Cheat Sheet

> Краткая шпаргалка: только то, что реально спрашивают на интервью

## Структура интервью (45 мин)

```
1. Requirements      (5 мин)  → Уточни: users, scale, latency, features
2. Capacity          (5 мин)  → DAU → RPS → Storage → Bandwidth
3. High-Level Design (15 мин) → Нарисуй архитектуру, обоснуй выбор
4. Deep Dive         (15 мин) → Детали критичных компонентов
5. Wrap-up           (5 мин)  → Bottlenecks, что бы улучшил
```

---

## Быстрые расчёты (запомнить)

```
1 день ≈ 100K секунд
1 месяц ≈ 2.5M секунд

10M DAU × 10 actions = 100M/day ≈ 1,000 RPS (peak: 3-5K)

Tweet: 250 bytes    Photo: 500KB    Video (1 min): 50MB
```

---

## Scaling

| Vertical | Horizontal |
|----------|------------|
| Мощнее сервер | Больше серверов |
| Просто, есть потолок | Сложнее, безлимитно |
| Stateful OK | Нужен stateless |

**Load Balancer алгоритмы:**
- Round Robin — серверы одинаковые
- Least Connections — запросы разной длины
- IP Hash — sticky sessions

---

## Databases

```
SQL (PostgreSQL):           NoSQL (Cassandra/Mongo):
├── Транзакции, ACID        ├── Eventual consistency
├── JOIN, сложные запросы   ├── Horizontal scaling
├── Структурированные       ├── Flexible schema
└── Платежи, inventory      └── Logs, feeds, IoT
```

**Sharding** — данные между серверами
- Hash-based: `shard = hash(user_id) % N` — равномерно, но range queries сложны
- Shard key: высокая кардинальность, часто в запросах

**Replication** — копии для отказоустойчивости
- Sync: гарантия данных, выше latency
- Async: быстрее, может отставать (replication lag)

---

## CAP Theorem

**При network partition выбираешь:**

| CP (Consistency) | AP (Availability) |
|------------------|-------------------|
| PostgreSQL, ZooKeeper | Cassandra, DynamoDB |
| Платежи, бронирование | Лента, счётчики, логи |
| Лучше отказ, чем неверные данные | Лучше stale, чем ничего |

---

## Caching

**Стратегии:**
```
Cache-Aside:  App проверяет кэш → miss → читает DB → пишет в кэш
Write-Through: Пишем в кэш → кэш пишет в DB (синхронно)
Write-Behind:  Пишем в кэш → OK → кэш пишет в DB (асинхронно)
```

**Redis vs Memcached:**
- Redis: структуры данных, persistence, pub/sub
- Memcached: простой KV, быстрее, экономнее по памяти

**Cache Stampede:** TTL истёк → все в DB → решение: locking или early refresh

---

## Message Queues

| Kafka | RabbitMQ |
|-------|----------|
| Log (хранит данные) | Queue (удаляет после consume) |
| Replay возможен | Replay невозможен |
| 1M+ msg/sec | 20-50K msg/sec |
| Event sourcing, logs | Task queues, RPC |

**Delivery guarantees:**
- At-most-once: может потеряться
- At-least-once: может задублироваться (нужна idempotency)
- Exactly-once: сложно (transactional outbox + idempotency)

---

## API

| REST | GraphQL | gRPC |
|------|---------|------|
| JSON, HTTP/1.1 | JSON, schema | Binary, HTTP/2 |
| Public APIs | Сложные клиенты | Microservices |
| Overfetching | Точные данные | Низкая latency |

**Real-time:**
- **Polling** — просто, высокая latency
- **Long Polling** — лучше, держит connections
- **SSE** — server→client, auto-reconnect
- **WebSocket** — bidirectional, сложнее scale

---

## Reliability

**Availability:**
```
99.9%  = 8.76 часов downtime/год
99.99% = 52 минуты downtime/год
```

**Паттерны:**
```
Circuit Breaker: downstream мёртв → быстро fail, не ждать
Retry + Backoff: 1s → 2s → 4s → 8s (+ jitter)
Idempotency Key: повторный запрос = тот же результат
Timeout: ВСЕГДА ставить, никогда бесконечный
```

---

## Типичные вопросы → Быстрые ответы

### "SQL или NoSQL?"
```
SQL: транзакции, JOIN, структурированные данные, ACID важен
NoSQL: scale, flexible schema, write-heavy, eventual consistency OK
```

### "Как масштабировать до 1M RPS?"
```
1. Stateless сервисы + horizontal scaling
2. Load balancer (несколько уровней)
3. Кэш (90%+ hit rate снижает нагрузку на DB)
4. Sharding базы данных
5. CDN для статики
```

### "Как обеспечить high availability?"
```
1. Redundancy: 2+ инстанса каждого компонента
2. Health checks + auto-failover
3. Multi-AZ deployment
4. Circuit breakers
5. Graceful degradation
```

### "Kafka или RabbitMQ?"
```
Kafka: event sourcing, нужен replay, высокий throughput, logs
RabbitMQ: task queue, сложная маршрутизация, RPC pattern
```

### "Как кэшировать?"
```
1. Ближе к клиенту = лучше (CDN > App cache > DB cache)
2. Cache-aside для read-heavy
3. TTL + invalidation при update
4. Redis для sessions, hot data, rate limiting
```

### "REST или gRPC для микросервисов?"
```
gRPC: internal service-to-service, низкая latency, streaming
REST: public API, простота, совместимость с браузерами
```

---

## Архитектурные блоки

```
┌─────────────────────────────────────────────────────────────────┐
│  Client                                                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  CDN (static assets, cached API responses)                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Load Balancer (L7: routing, SSL termination)                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  API Gateway (auth, rate limiting, routing)                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Service A   │  │  Service B   │  │  Service C   │
└──────────────┘  └──────────────┘  └──────────────┘
        ↓                 ↓                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Cache (Redis) — sessions, hot data, rate limits                │
└─────────────────────────────────────────────────────────────────┘
        ↓                 ↓                 ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │  Cassandra   │  │  Elasticsearch│
│  (primary)   │  │  (sharded)   │  │  (search)    │
└──────────────┘  └──────────────┘  └──────────────┘
        ↓
┌─────────────────────────────────────────────────────────────────┐
│  Message Queue (Kafka) — async processing, event sourcing       │
└─────────────────────────────────────────────────────────────────┘
        ↓
┌──────────────┐  ┌──────────────┐
│  Workers     │  │  Analytics   │
└──────────────┘  └──────────────┘
```

---

## Чеклист перед интервью

- [ ] Уточнить requirements (не прыгать в решение)
- [ ] Посчитать нагрузку (DAU → RPS → Storage)
- [ ] Начать с простого, добавлять по мере обсуждения
- [ ] Обосновывать каждый выбор ("потому что...")
- [ ] Обсудить trade-offs (нет идеального решения)
- [ ] Думать вслух (интервьюер хочет видеть процесс)

---

*Детали по каждой теме: см. папку `concepts/`*
