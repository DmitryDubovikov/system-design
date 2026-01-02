# System Design Quick Reference

> Краткая шпаргалка для повторения перед интервью

---

## Фреймворк (45 мин)

```
[0-5]   Уточнить scope, задать вопросы
[5-10]  Requirements (FR + NFR + числа)
[10-15] Back-of-envelope расчёты
[15-30] High-level design (рисуем!)
[30-40] Deep dive (1-2 компонента)
[40-45] Bottlenecks, масштабирование
```

**Золотые правила:**
- Думай вслух — интервьюер оценивает процесс
- Не молчи дольше 30 секунд
- Начни с простого, усложняй по запросу
- Слушай подсказки интервьюера

---

## Числа для расчётов

### Latency
```
L1 cache           0.5 ns
RAM                100 ns
SSD random         150 μs
HDD seek           10 ms
Network (DC)       0.5 ms
Network (cross-DC) 50-150 ms
```

### Throughput
```
SSD seq read    500 MB/s - 3 GB/s
HDD seq read    100 MB/s
1 Gbps network  125 MB/s
10 Gbps         1.25 GB/s
```

### Время
```
1 день    ≈ 100K секунд (86,400)
1 месяц   ≈ 2.5M секунд
1 год     ≈ 30M секунд
```

### QPS
```
1M DAU, 1 req/day   → ~12 QPS
1M DAU, 10 req/day  → ~120 QPS
Peak                → 2-3x average
```

### Storage
```
1 char UTF-8     1-4 bytes
Tweet/post       ~500 bytes
Photo (сжато)    200 KB - 2 MB
Video (1 min)    ~50 MB (720p)
```

### Типичные лимиты
```
Redis key        512 MB max
MySQL row        65 KB
MongoDB doc      16 MB
S3 object        5 TB
```

---

## Компоненты: когда что использовать

### Базы данных

| БД | Когда использовать |
|----|-------------------|
| **PostgreSQL/MySQL** | ACID нужен, relations, < 10TB |
| **MongoDB** | Flexible schema, документы |
| **Cassandra** | Write-heavy, time-series, AP |
| **Redis** | Cache, sessions, leaderboards |
| **Elasticsearch** | Full-text search, logs |
| **ClickHouse** | Analytics, OLAP |

### Message Queues

| Queue | Когда использовать |
|-------|-------------------|
| **Kafka** | High throughput, event log, replay |
| **RabbitMQ** | Complex routing, RPC |
| **SQS** | Simple queue, AWS |
| **Redis Pub/Sub** | Real-time, ephemeral |

### Cache

| Паттерн | Описание |
|---------|----------|
| **Cache-aside** | App читает/пишет cache отдельно |
| **Write-through** | Пишем в cache и DB вместе |
| **Write-behind** | Пишем в cache, async в DB |
| **Read-through** | Cache сам ходит в DB |

**Cache-aside (самый частый):**
```
Read:  cache miss → read DB → update cache
Write: update DB → invalidate cache
```

### Load Balancing

| Алгоритм | Когда |
|----------|-------|
| Round Robin | Одинаковые серверы |
| Least Connections | Разная нагрузка |
| IP Hash | Session affinity |
| Weighted | Разная мощность серверов |

---

## Паттерны масштабирования

### Database

```
           ┌─────────────┐
           │   Master    │ ← Writes
           └──────┬──────┘
        ┌─────────┼─────────┐
        ▼         ▼         ▼
    ┌───────┐ ┌───────┐ ┌───────┐
    │Replica│ │Replica│ │Replica│ ← Reads
    └───────┘ └───────┘ └───────┘
```

**Replication:** Read replicas для read-heavy
**Sharding:** Горизонтальное разделение данных
**Partitioning:** По времени, географии, ID

### Sharding Strategies

| Strategy | Как работает | Плюсы/Минусы |
|----------|--------------|--------------|
| Hash-based | `hash(key) % N` | Равномерно, но reshard сложно |
| Range-based | По диапазону | Просто, но hot spots |
| Consistent hashing | Hash ring | Легко добавлять ноды |
| Directory-based | Lookup table | Гибко, но extra hop |

### Horizontal vs Vertical

| Vertical | Horizontal |
|----------|------------|
| Bigger machine | More machines |
| Простой | Сложнее |
| Есть лимит | Почти безлимитно |
| Для БД на старте | Для stateless сервисов |

---

## CAP Theorem

```
    Consistency
        /\
       /  \
      /    \
     /  CA  \     (невозможно в distributed)
    /________\
   /\        /\
  /  \  CP  /  \
 / AP \    /    \
/______\  /______\
Availability  Partition Tolerance
```

**CP:** Консистентность важнее (банки, платежи)
**AP:** Доступность важнее (социалки, кэши)

**На практике:** Выбираем между C и A при partition

---

## Типичные паттерны

### Rate Limiting

**Token Bucket** (рекомендуется):
```
- Bucket size = burst capacity
- Refill rate = sustained rate
- Позволяет bursts
```

**Алгоритмы:**
```
Token Bucket    — bursts OK, smooth
Sliding Window  — точный, больше памяти
Fixed Window    — простой, burst на границе
Leaky Bucket    — строгий rate
```

### Идемпотентность

```python
# Client sends: Idempotency-Key: abc123
if key exists in DB:
    return cached_response
else:
    process request
    save response with key
    return response
```

### Retry с Exponential Backoff

```
Attempt 1: сразу
Attempt 2: 1 sec
Attempt 3: 2 sec
Attempt 4: 4 sec
Attempt 5: 8 sec
+ jitter (случайная добавка)
```

### Circuit Breaker

```
CLOSED → (failures > threshold) → OPEN
OPEN → (timeout) → HALF-OPEN
HALF-OPEN → (success) → CLOSED
HALF-OPEN → (failure) → OPEN
```

---

## Типичные вопросы

### "Как масштабировать?"

1. **Stateless services:** Добавить серверы за LB
2. **Database:** Read replicas → Sharding → Caching
3. **Cache:** Redis Cluster, CDN
4. **Async:** Message queue, background jobs

### "Что если X упадёт?"

```
1. Detection:  Health checks, monitoring
2. Mitigation: Failover, replicas, circuit breaker
3. Recovery:   Auto-restart, data recovery
4. Prevention: Redundancy, backups
```

### "Как обеспечить consistency?"

- **Strong:** Sync replication, 2PC, ACID
- **Eventual:** Async replication, reconciliation
- **Causal:** Vector clocks, timestamps

### "Как избежать single point of failure?"

- Redundancy (replicas, multi-AZ)
- Failover (automatic, health checks)
- Graceful degradation

---

## Протоколы и форматы

### REST vs gRPC vs GraphQL

| | REST | gRPC | GraphQL |
|-|------|------|---------|
| Format | JSON | Protobuf | JSON |
| Speed | OK | Fast | OK |
| Когда | Public API | Internal, microservices | Flexible queries |

### WebSocket vs SSE vs Polling

| | WebSocket | SSE | Long Polling |
|-|-----------|-----|--------------|
| Direction | Bidirectional | Server→Client | Request-Response |
| Когда | Chat, gaming | Notifications, feeds | Fallback |

### Streaming

```
HLS:  HTTP Live Streaming (Apple, широко поддержан)
DASH: Dynamic Adaptive (стандарт)
RTMP: Real-Time Messaging (ingest)
WebRTC: Ultra low latency (P2P)
```

---

## Архитектурные решения

### Sync vs Async

| Sync | Async |
|------|-------|
| Простой | Сложнее |
| Latency включает всё | Быстрый ответ |
| Failures cascade | Decoupled |
| Для: читать данные | Для: email, processing |

### Push vs Pull

| Push | Pull |
|------|------|
| Real-time | Polling |
| Больше ресурсов | Задержка |
| Fan-out проблема | Проще |
| Для: chat, notifications | Для: feed refresh |

### Monolith vs Microservices

| Monolith | Microservices |
|----------|---------------|
| Проще начать | Сложнее |
| Всё вместе | Независимый deploy |
| Vertical scaling | Horizontal scaling |
| Для: MVP, маленькая команда | Для: большие системы |

---

## Чек-лист перед интервью

### Уточняющие вопросы
- [ ] Масштаб? (DAU, QPS, storage)
- [ ] Что в scope, что нет?
- [ ] Latency requirements?
- [ ] Consistency vs Availability?
- [ ] Географическое распределение?

### Не забыть упомянуть
- [ ] Load balancer
- [ ] Caching (Redis, CDN)
- [ ] Database choice + почему
- [ ] Message queue для async
- [ ] Monitoring & alerting

### Частые ошибки
- ❌ Рисовать сразу без вопросов
- ❌ Забыть про NFR (latency, availability)
- ❌ Не сделать capacity estimation
- ❌ Over-engineering с самого начала
- ❌ Молчать и думать долго
- ❌ Игнорировать подсказки

---

## Быстрые формулы

### QPS из DAU
```
QPS = (DAU × requests_per_user) / 86400
Peak QPS = QPS × 3
```

### Storage
```
Daily = records_per_day × record_size
Yearly = Daily × 365
With replication = Yearly × replication_factor
```

### Bandwidth
```
Bandwidth = QPS × avg_response_size
```

### Servers needed
```
Servers = Peak_QPS / QPS_per_server
+ headroom (2x for safety)
```

---

## Шаблон ответа на любую систему

```
1. CLARIFY
   "Let me ask a few questions..."
   - Scale, features, constraints

2. REQUIREMENTS
   "Based on that, our requirements are..."
   - Functional: what it does
   - Non-functional: latency, availability, consistency

3. ESTIMATES
   "Let me do some quick math..."
   - QPS, storage, bandwidth

4. HIGH-LEVEL DESIGN
   "Here's the overall architecture..."
   - Draw boxes: clients, LB, services, DB, cache
   - Explain data flow

5. DEEP DIVE
   "Let me dive deeper into..."
   - Pick 1-2 interesting components
   - Discuss trade-offs

6. WRAP UP
   "Some potential bottlenecks..."
   - What can fail, how to handle
   - Future improvements
```

---

## Фразы для интервью

**Начало:**
- "Before I start, let me ask a few clarifying questions..."
- "Let me make sure I understand the requirements..."

**Расчёты:**
- "Let me do some back-of-envelope calculations..."
- "Assuming X users and Y requests per day..."

**Дизайн:**
- "At a high level, I'd structure it like this..."
- "The key components would be..."

**Trade-offs:**
- "There's a trade-off here between X and Y..."
- "We could go with A for simplicity, or B for better performance..."

**Масштабирование:**
- "If we need to scale further, we could..."
- "A potential bottleneck here is..."

**Завершение:**
- "To summarize the key points..."
- "Some things we could improve in the future..."
