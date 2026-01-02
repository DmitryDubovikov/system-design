# System Design Interview Cheat Sheet

## Содержание

### Classics (базовые системы)
| Система | Сложность | Ключевые концепции |
|---------|-----------|-------------------|
| [URL Shortener](classics/url_shortener.md) | ⭐ | Hashing, Base62, Read-heavy |
| [Rate Limiter](classics/rate_limiter.md) | ⭐⭐ | Token Bucket, Sliding Window, Distributed |
| [Notification System](classics/notification_system.md) | ⭐⭐ | Multi-channel, Priority, Delivery guarantees |
| [Chat System](classics/chat_system.md) | ⭐⭐⭐ | WebSockets, Presence, Message ordering |

### Social (социальные платформы)
| Система | Сложность | Ключевые концепции |
|---------|-----------|-------------------|
| [Twitter Feed](social/twitter_feed.md) | ⭐⭐⭐ | Fan-out, Ranking, Timeline |
| [Instagram](social/instagram.md) | ⭐⭐⭐ | Media storage, CDN, Feed generation |

### Streaming (стриминг и real-time)
| Система | Сложность | Ключевые концепции |
|---------|-----------|-------------------|
| [YouTube](streaming/youtube.md) | ⭐⭐⭐ | Video processing, Adaptive bitrate, CDN |
| [Live Streaming](streaming/live_streaming.md) | ⭐⭐⭐ | Low latency, RTMP, Transcoding |

### Fintech (финансовые системы)
| Система | Сложность | Ключевые концепции |
|---------|-----------|-------------------|
| [Payment System](fintech/payment_system.md) | ⭐⭐⭐ | Idempotency, Exactly-once, Reconciliation |
| [Digital Wallet](fintech/digital_wallet.md) | ⭐⭐⭐ | Double-entry, Ledger, ACID |
| [Remittance System](fintech/remittance_system.md) | ⭐⭐⭐ | FX, Compliance, Settlement |
| [Fraud Detection](fintech/fraud_detection.md) | ⭐⭐⭐ | Real-time ML, Feature store, Rules engine |

---

## Универсальный фреймворк (45 минут)

### Распределение времени
```
[0-5 мин]   Уточняющие вопросы, scope
[5-10 мин]  Requirements (FR + NFR)
[10-15 мин] Capacity estimation
[15-30 мин] High-level design + API
[30-40 мин] Deep dive в 1-2 компонента
[40-45 мин] Bottlenecks, trade-offs
```

### Чек-лист вопросов интервьюеру

**Масштаб:**
- Сколько пользователей? DAU/MAU?
- Какой объём данных? Рост?
- Географическое распределение?

**Функциональность:**
- Какие основные use cases?
- Что в scope, что вне scope?
- Есть ли real-time требования?

**Ограничения:**
- Какая допустимая latency?
- Требования к availability? (99.9%? 99.99%?)
- Consistency vs Availability — что важнее?

---

## Шпаргалка по числам

### Latency (порядок величин)
```
L1 cache:           0.5 ns
L2 cache:           7 ns
RAM:                100 ns
SSD random read:    150 μs
HDD seek:           10 ms
Network (same DC):  0.5 ms
Network (cross DC): 50-150 ms
```

### Throughput
```
SSD sequential:     500 MB/s - 3 GB/s
HDD sequential:     100 MB/s
Network (1 Gbps):   125 MB/s
Network (10 Gbps):  1.25 GB/s
```

### Capacity
```
1 миллион секунд ≈ 11.5 дней
1 миллиард секунд ≈ 31.7 лет

Секунд в день: 86,400 ≈ 100K
Секунд в месяц: ≈ 2.5M
Секунд в год: ≈ 30M

1 символ UTF-8: 1-4 bytes
Средний tweet: ~300 bytes
Средний пост с meta: ~1 KB
Фото (сжатое): 200 KB - 2 MB
Видео (1 мин, 720p): ~50 MB
```

### QPS правила
```
1M DAU → ~12 QPS (если 1 запрос/день)
1M DAU → ~120 QPS (если 10 запросов/день)

Пиковая нагрузка: 2-3x от средней
```

---

## Паттерны и когда их использовать

### База данных
| Паттерн | Когда использовать |
|---------|-------------------|
| Sharding | Данные не влезают в одну машину |
| Read replicas | Read-heavy workload |
| Write-ahead log | Durability, recovery |
| CQRS | Разные модели для read/write |

### Кэширование
| Паттерн | Когда использовать |
|---------|-------------------|
| Cache-aside | Общий случай, read-heavy |
| Write-through | Consistency важна |
| Write-behind | Write-heavy, можно потерять |
| CDN | Статический контент, геораспределение |

### Messaging
| Паттерн | Когда использовать |
|---------|-------------------|
| Message Queue | Асинхронная обработка |
| Pub/Sub | Fan-out, multiple consumers |
| Event Sourcing | Audit log, replay events |

### Масштабирование
| Паттерн | Когда использовать |
|---------|-------------------|
| Load Balancer | Распределение нагрузки |
| Horizontal scaling | Stateless services |
| Vertical scaling | Быстрый fix, БД |
| Auto-scaling | Variable load |

---

## Типичные ошибки на интервью

❌ **Сразу рисовать диаграмму** без уточнения requirements
❌ **Забыть про non-functional** requirements (latency, availability)
❌ **Не делать capacity estimation** — числа важны!
❌ **Over-engineering** — начни с простого, усложняй по запросу
❌ **Молчать** — думай вслух, объясняй trade-offs
❌ **Игнорировать подсказки** интервьюера
❌ **Застрять на одном компоненте** — следи за временем

---

## Как отвечать на "What if X fails?"

```
1. Identify: Какой компонент упал?
2. Impact: Что перестанет работать?
3. Detection: Как узнаем о проблеме?
4. Mitigation: Как минимизируем impact?
5. Recovery: Как восстанавливаемся?
```

**Пример для Database:**
- Impact: Часть writes/reads недоступна
- Detection: Health checks, monitoring
- Mitigation: Failover to replica
- Recovery: Repair/replace failed node

---

## Quick Reference: Технологии

### Databases
- **PostgreSQL/MySQL**: ACID, relations, до ~10TB
- **MongoDB**: Flexible schema, horizontal scale
- **Cassandra**: Write-heavy, time-series, AP
- **Redis**: Cache, pub/sub, leaderboards
- **DynamoDB**: Managed, auto-scale, key-value

### Message Queues
- **Kafka**: High throughput, log-based, replay
- **RabbitMQ**: Traditional MQ, routing, reliability
- **SQS**: Managed, simple, AWS integration

### Search
- **Elasticsearch**: Full-text, analytics, logs
- **Algolia**: Managed, typo-tolerant, fast

### Storage
- **S3**: Objects, 99.999999999% durability
- **CloudFront/Cloudflare**: CDN, edge caching
