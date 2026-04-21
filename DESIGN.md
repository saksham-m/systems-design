# URL Shortener — System Design

## Requirements

### Functional
- Create a short URL from a long URL
- Support custom alias as short code
- Redirect short URL to original URL
- Deduplicate: same long URL always returns same short code
- Fixed TTL on all URLs

### Non-Functional
- 200ms latency on redirects
- 100M DAU, 1B URLs stored
- Short codes must be globally unique

### Out of Scope
- Auth
- Analytics
- URL deletion

---

## Capacity Estimation

| Metric | Value |
|---|---|
| Writes/sec | ~115 |
| Reads/sec | ~11,500 |
| Read/write ratio | 100:1 |
| Total storage | ~500 GB |
| Hot cache size | ~100 GB |
| Short code length | 6 chars (base62 → 56B combinations) |

---

## API

```
POST /urls
  Body:     { original_url, alias? }
  Response: { short_url, expires_at }
  Errors:   409 if alias already taken

GET /{shortCode}
  Response: 302 redirect to original_url
  Errors:   404 if not found or expired
```

---

## Core Entities

- **User** — creates short URLs
- **ShortURL** — maps a short code to an original URL, owned by a user
- **OriginalURL** — the long URL, deduplicated

---

## Data Model

### URL Table

| Column | Type | Notes |
|---|---|---|
| short_code | varchar(7) | PK |
| long_url | text | Unique index for dedup |
| created_at | timestamp | |
| expires_at | timestamp | created_at + fixed TTL |
| user_id | FK → User | First creator |

### User Table

| Column | Type | Notes |
|---|---|---|
| user_id | uuid | PK |

---

## Short Code Generation

- Counter stored in **Redis** using atomic `INCR`
- Counter integer → base62 encoded → short code
- Custom alias skips counter, checked for uniqueness against DB
- Redis AOF persistence enabled to survive restarts

---

## Architecture

### Components

| Component | Purpose |
|---|---|
| Load Balancer | Distribute traffic across app servers |
| App Servers | Stateless, horizontally scalable |
| Redis | LRU cache (redirects) + atomic counter (code gen) |
| PostgreSQL | Source of truth |

### Write Path — Create Short URL

```
Client → Load Balancer → App Server → Redis INCR
                                          ↓
                                      base62 encode
                                          ↓
                                      Check DB (dedup / alias conflict)
                                          ↓
                                      Insert into PostgreSQL
                                          ↓
                                      Return { short_url, expires_at }
```

### Read Path — Redirect

```
Client → Load Balancer → App Server → Redis cache lookup
                                          ↓ hit
                                      302 redirect
                                          ↓ miss
                                      PostgreSQL lookup
                                          ↓
                                      Populate Redis (lazy)
                                          ↓
                                      302 redirect / 404
```

---

## Cache Design (Redis)

| Property | Value |
|---|---|
| Key | short_code |
| Value | long_url |
| Eviction policy | allkeys-lru |
| TTL | aligned to expires_at (auto-evicts expired URLs) |
| Cache warming | Lazy (populated on first miss) |
