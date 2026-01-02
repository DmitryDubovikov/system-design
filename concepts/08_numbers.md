# Ключевые числа для System Design

## Latency Numbers Every Programmer Should Know

```
┌─────────────────────────────────────────────────────────────┐
│ Operation                              │ Latency            │
├─────────────────────────────────────────────────────────────┤
│ L1 cache reference                     │ 0.5 ns             │
│ L2 cache reference                     │ 7 ns               │
│ Main memory reference                  │ 100 ns             │
│ SSD random read                        │ 150 μs             │
│ HDD seek                               │ 10 ms              │
│ Round trip same datacenter             │ 0.5 ms             │
│ Round trip CA → Netherlands            │ 150 ms             │
└─────────────────────────────────────────────────────────────┘
```

### Визуализация (если бы 1 ns = 1 сек)

```
L1 cache:           1 секунда
L2 cache:           7 секунд
RAM:                2 минуты
SSD read:           2 дня
HDD seek:           4 месяца
CA → Europe:        5 лет
```

**Вывод:** Кэш в памяти на порядки быстрее диска. Network — самое медленное.

---

## Типичные latency для сервисов

| Операция | Latency | Примечание |
|----------|---------|------------|
| Redis GET | 0.1-1 ms | В том же datacenter |
| PostgreSQL simple query | 1-5 ms | С индексом |
| PostgreSQL complex query | 10-100 ms | JOIN, без индекса |
| Elasticsearch query | 10-100 ms | Зависит от сложности |
| HTTP API call (internal) | 5-50 ms | Включая network |
| HTTP API call (external) | 50-500 ms | Зависит от провайдера |
| CDN asset | 5-50 ms | Edge server близко |

---

## Throughput Numbers

### База данных

| БД | Writes/sec | Reads/sec | Примечание |
|----|------------|-----------|------------|
| PostgreSQL (single) | 10-50K | 50-100K | Зависит от железа |
| MySQL (single) | 10-50K | 50-100K | InnoDB |
| Redis (single) | 100K+ | 100K+ | In-memory |
| Cassandra (cluster) | 100K+ per node | 100K+ | Linear scaling |
| Kafka | 1M+ msg/sec | 1M+ msg/sec | Per broker |

### Web серверы

| Сервер/Runtime | RPS (typical) | Примечание |
|----------------|---------------|------------|
| Nginx (static) | 50-100K | Per server |
| Node.js | 10-30K | Per process |
| Go HTTP | 50-100K | Per server |
| Python (gunicorn) | 1-5K | Per worker |

---

## Capacity Estimation Формулы

### Базовые конверсии

```
Время:
1 день = 86,400 секунд ≈ 100K секунд
1 месяц ≈ 2.5M секунд
1 год ≈ 30M секунд

Данные:
1 KB = 1,000 bytes (для estimation)
1 MB = 1,000 KB
1 GB = 1,000 MB
1 TB = 1,000 GB
1 PB = 1,000 TB
```

### Traffic estimation

```
Дано: 10M DAU, каждый делает 10 actions/day

Daily requests = 10M × 10 = 100M requests/day

Average RPS = 100M / 86,400 ≈ 100M / 100K ≈ 1,000 RPS

Peak RPS = Average × 3-5 = 3,000-5,000 RPS
(правило большого пальца: peak в 3-5 раз выше average)
```

### Storage estimation

```
Дано: Twitter-like система
- 500M tweets/day
- Average tweet: 200 bytes text + 50 bytes metadata
- 20% имеют фото (500KB avg)

Text storage/day:
500M × 250 bytes = 125 GB/day

Photo storage/day:
500M × 20% × 500KB = 50TB/day

Total/year:
Text: 125GB × 365 = 45TB
Photos: 50TB × 365 = 18PB
```

### Bandwidth estimation

```
Дано: Video streaming service
- 1M concurrent users
- Average bitrate: 5 Mbps

Bandwidth = 1M × 5 Mbps = 5 Tbps (terabits per second)

В Gbps: 5,000 Gbps
В servers (10Gbps each): 500 servers для bandwidth
```

---

## Quick Reference Numbers

### Users и Traffic

| Метрика | Small | Medium | Large | Huge |
|---------|-------|--------|-------|------|
| DAU | 10K | 1M | 100M | 1B+ |
| RPS (avg) | 10 | 1K | 100K | 1M+ |
| RPS (peak) | 50 | 5K | 500K | 5M+ |

### Storage

| Тип данных | Размер |
|------------|--------|
| Tweet/post | 250 bytes |
| User profile | 1-10 KB |
| Photo (compressed) | 200KB - 1MB |
| Video (1 min, compressed) | 50-100 MB |
| Log entry | 100-500 bytes |

### Servers

| Тип | CPU | RAM | Storage | RPS capacity |
|-----|-----|-----|---------|--------------|
| Web/API | 4-8 cores | 8-16 GB | 100GB SSD | 5-10K |
| Database | 16-32 cores | 64-256 GB | 1-10TB SSD | 10-50K queries |
| Cache | 8-16 cores | 64-256 GB | - | 100K+ ops |

---

## SLA и Availability

### Сколько стоит downtime

```
99.9% = 8.76 часов downtime/год
99.99% = 52.6 минут downtime/год
99.999% = 5.26 минут downtime/год

Цена downtime (примерная для e-commerce):
$10K revenue/hour → 99.9% стоит $87K/год в потерях
Каждая девятка уменьшает потери в 10 раз
```

### Сколько серверов для N nines

```
Single server: ~99% (несколько дней downtime/год)
2 servers (active-passive): ~99.9%
3+ servers (active-active): ~99.99%
Multi-region: ~99.999%

Формула (упрощённая):
A(system) = 1 - (1 - A(node))^n
где n = количество redundant nodes
```

---

## Примеры Back-of-Envelope

### Пример 1: URL Shortener

```
Requirements:
- 100M URLs created/month
- Read:Write = 100:1

Writes:
100M / month = 100M / 2.5M sec ≈ 40 writes/sec

Reads:
40 × 100 = 4,000 reads/sec

Storage (5 years):
- Short URL: 7 chars = 7 bytes
- Long URL: 100 chars = 100 bytes
- Total per entry: ~150 bytes (with metadata)
- 100M × 12 months × 5 years × 150 bytes = 9TB

Вывод:
- One PostgreSQL легко справится
- Add Redis для hot URLs
- CDN для redirects
```

### Пример 2: Chat System (WhatsApp-like)

```
Requirements:
- 500M DAU
- 40 messages/user/day

Messages:
500M × 40 = 20B messages/day
20B / 86,400 ≈ 230K messages/sec (peak: ~1M/sec)

Storage (per message):
- Content: 100 bytes avg
- Metadata: 50 bytes
- Total: 150 bytes
- Daily: 20B × 150 = 3TB/day
- Yearly: 1PB/year

Bandwidth (sending):
230K × 150 bytes = 35 MB/sec = 280 Mbps avg

Architecture implications:
- Sharded database by user_id
- WebSocket servers (connection heavy)
- Message queues для delivery
- CDN для media
```

### Пример 3: News Feed (Facebook-like)

```
Requirements:
- 1B DAU
- Average 500 friends
- 10 feed refreshes/day
- 50 posts per refresh

Fan-out on write:
- User posts → push to all followers' feeds
- 1B DAU × 1 post/day average = 1B posts/day
- Each post → 500 feeds = 500B feed updates/day
- 500B / 86,400 = 5.8M updates/sec (!!!)

Fan-out on read:
- User opens feed → pull from friends
- 1B × 10 refreshes = 10B feed reads/day
- Each read → query 500 friends = 5T queries/day
- Cached with Redis: OK

Hybrid approach (actual):
- Fan-out on write для обычных users
- Fan-out on read для celebrities (много followers)
- Precomputed feeds в Redis
```

---

## Чеклист для Capacity Estimation

```
□ Users
  - Total users
  - DAU/MAU
  - Concurrent users (peak)

□ Traffic
  - Requests per day
  - Average RPS
  - Peak RPS (3-5x average)
  - Read:Write ratio

□ Storage
  - Per-record size
  - Records per day
  - Retention period
  - Total storage (years)

□ Bandwidth
  - Ingress (uploads)
  - Egress (downloads)
  - Peak bandwidth

□ Resources
  - Number of servers
  - Database size/type
  - Cache size
  - CDN requirements
```

---

## Memory Tricks

```
Powers of 2:
2^10 = 1,024 ≈ 1 thousand (KB)
2^20 = 1,048,576 ≈ 1 million (MB)
2^30 = ~1 billion (GB)
2^40 = ~1 trillion (TB)

Quick math:
1 million seconds ≈ 11.5 days
1 billion seconds ≈ 31.7 years

Requests:
1 request/sec = 86,400/day ≈ 2.5M/month
1000 RPS = 86M/day ≈ 2.5B/month
```
