# Redis Learning Roadmap

> Untuk Fullstack Developer (NestJS + ReactJS)
> Tujuan: Memahami Redis dari dasar sampai produksi, dengan fokus pada integrasi NestJS.

---

## Fase 0 — Persiapan (Pra-Requirement)

- [ ] Sudah paham dasar JavaScript/TypeScript & NestJS (modular, DI, providers).
- [ ] Paham dasar HTTP, REST API, dan konsep stateful vs stateless.
- [ ] Install Redis di lokal:
  - **Windows**: WSL2 + Redis, atau Docker (disarankan).
  - **Docker**: `docker run -d --name redis -p 6379:6379 redis:8`
- [ ] Install GUI client (pilih salah satu):
  - RedisInsight (resmi, gratis)
  - Another Redis Desktop Manager (ARDM)
  - `redis-cli` (wajib dikuasai untuk debugging)

---

## Fase 1 — Konsep Dasar Redis (Minggu 1)

- [ ] Apa itu Redis? (In-memory data store, key-value, kecepatan vs durability).
- [ ] Kapan Redis dibutuhkan dan kapan TIDAK (bukan pengganti database utama).
- [ ] Redis itu *single-threaded event loop*: satu command lambat memblok seluruh server — hindari `HGETALL` pada hash raksasa, `KEYS *`, `SMEMBERS` pada set besar (ini alasan anti-pattern di Fase 6).
  - **String**: `SET`, `GET`, `SETEX`, `INCR`, `DECR`, `MSET`, `MGET`.
  - **Hash**: `HSET`, `HGET`, `HGETALL`, `HINCRBY`.
  - **List**: `LPUSH`, `RPUSH`, `LRANGE`, `LPOP`, `RPOP`.
  - **Set**: `SADD`, `SREM`, `SMEMBERS`, `SISMEMBER`, `SINTER`, `SUNION`.
  - **Sorted Set**: `ZADD`, `ZRANGE`, `ZRANK`, `ZINCRBY`.
- [ ] Expiry: `EXPIRE`, `TTL`, `PERSIST`, `SETEX`.
- [ ] Latihan praktik di `redis-cli` dengan semua command di atas.

**Proyek mini:** Buat counter sederhana dan leaderboard menggunakan `redis-cli`.

---

## Fase 2 — Integrasi NestJS (Minggu 2)

- [ ] Library Redis untuk NestJS (pilih satu):
  - `ioredis` (direkomendasikan, fitur lengkap, cluster support).
  - `@nestjs/cache-manager` (abstraksi cache bawaan NestJS).
  - `redis` (node-redis v4, official).
- [ ] Setup dasar dengan `ioredis`:
  - Buat `RedisModule` (custom module / global module).
  - Buat `RedisService` wrapper.
  - Konfigurasi via `ConfigService` (host, port, password).
- [ ] Buat project contoh:
  - Endpoint `POST /counter/incr` → `INCR`.
  - Endpoint `GET /counter` → `GET`.
  - Simpan session/user profile dengan `Hash`.
  - Leaderboard dengan `Sorted Set`.

**Proyek mini:** Buat API leaderboard game sederhana + rate limiter manual (menggunakan `INCR` + `EXPIRE`). *Catatan: versi ini punya race condition (langkah non-atomic), akan diperbaiki dengan Lua di Fase 4.*

---

## Fase 3 — Pola Umum (Caching Patterns) (Minggu 3)

- [ ] **Cache-Aside (Lazy Loading)** — pola paling umum:
  1. Cek cache → hit? return.
  2. Miss → query DB → simpan ke cache → return.
- [ ] **Write-Through** dan **Write-Behind** (konsep, bukan wajib implementasi).
- [ ] **TTL Strategy**:
  - Short TTL (detik-menit) vs Long TTL (jam-hari).
  - Jangan pakai TTL `-1` (never expire) untuk data dinamis.
- [ ] **Cache Invalidation** (bagian tersulit):
  - Hapus cache saat data berubah (delete, bukan update).
  - Pattern: invalidate-on-write.
  - Pertimbangan *stale data* dan *eventual consistency*.
- [ ] **Cache Stampede** & cara mengatasinya:
  - TTL jitter (random TTL).
  - Locking / recompute cache (contoh: `SETNX` lock).
- [ ] **Cache Key Naming Convention**:
  - Contoh: `user:123`, `article:list:page:2`, `product:detail:456`.
  - Gunakan prefix untuk namespacing & memudahkan invalidation.

**Proyek mini:** Cache hasil query database PostgreSQL/MySQL di endpoint `GET /products/:id` dengan pola cache-aside + invalidasi saat update/delete.

---

## Fase 4 — Fitur Lanjutan Redis (Minggu 4)

- [ ] **Pub/Sub** (publish-subscribe):
  - `PUBLISH`, `SUBSCRIBE`, `PSUBSCRIBE`.
  - Integrasi dengan NestJS (WebSocket/SSE untuk realtime ke React).
  - Catat kelemahan: message tidak disimpan, tidak ada guarantee delivery → untuk kasus penting pakai Redis Streams.
  - Subscriber butuh koneksi terpisah (di ioredis: `redis.duplicate()`) — koneksi dalam mode subscribe tidak bisa mengirim command lain.
- [ ] **Redis Streams** (untuk antrian & event log):
  - `XADD`, `XREAD`, `XREADGROUP`, `XACK`, `XPENDING`.
  - Consumer groups.
- [ ] **Lua Scripting** (`EVAL`, `EVALSHA`):
  - Atomic operations multi-command.
  - Contoh: rate limiter atomic.
- [ ] **Transactions** (`MULTI`/`EXEC`, `WATCH`):
  - Pahami keterbatasan (tidak ada rollback).
- [ ] **Pipelines**:
  - Mengurangi round-trip network (`ioredis.pipeline()`).
- [ ] **Data Persistence** (penting untuk produksi):
  - RDB (snapshot) vs AOF (append-only file).
  - Trade-off durability vs performance.

**Proyek mini:** Buat notifikasi realtime sederhana (Pub/Sub + Socket.IO) dan job queue sederhana dengan Streams.

---

## Fase 5 — Kasus Penggunaan Nyata di Aplikasi Fullstack (Minggu 5-6)

- [ ] **Session Management**: simpan session di Redis (untuk multi-instance / horizontal scaling).
- [ ] **Rate Limiting**: implementasi dengan `INCR` + `EXPIRE` atau `@nestjs/throttler` + Redis storage.
- [ ] **Caching**:
  - Caching API response.
  - Caching hasil query DB.
  - Full-page / HTTP caching.
- [ ] **Distributed Lock**: `SET key <token-unik> NX PX <ttl>` untuk mencegah double processing. Simpan *nilai unik* (token) di lock, dan hanya pemilik token yang boleh melepasnya (release atomic via Lua: cek token lalu `DEL`).
- [ ] **Leaderboard / Ranking**: Sorted Set.
- [ ] **Queue / Job Processing**: Redis Streams atau BullMQ (disarankan untuk NestJS).
  - **BullMQ** + `@nestjs/bullmq` untuk background job & antrian.
  - ⚠️ Instance Redis untuk BullMQ wajib `noeviction` — pisahkan dari Redis cache (kalau tidak, job bisa hilang/stuck).
- [ ] **Realtime Features**: Pub/Sub + WebSocket.
- [ ] **Idempotency Key**: mencegah duplicate request (payment, form submit).

**Proyek utama (capstone):** Buat aplikasi fullstack (NestJS + React) yang menggabungkan:
- Cache API + invalidasi.
- Rate limiting.
- Leaderboard.
- Background job (BullMQ) untuk kirim email/notifikasi.
- Realtime notifikasi via WebSocket + Pub/Sub.

---

## Fase 6 — Produksi, Operasional & Best Practice (Minggu 7)

- [ ] **Redis di Docker / Docker Compose**:
  - Volume untuk persistence.
  - Password auth (`requirepass`).
- [ ] **Konfigurasi Keamanan**:
  - `requirepass`, jangan expose port 6379 ke publik.
  - ACL dasar: buat user terbatas (bukan full access) untuk aplikasi.
  - Bind ke localhost/internal network.
- [ ] **Monitoring**:
  - `INFO`, `MONITOR`, `SLOWLOG`.
  - RedisInsight monitoring.
  - Memori: `maxmemory`, `maxmemory-policy` (e.g. `allkeys-lru`).
- [ ] **Eviction Policy**: pahami LRU vs LFU vs TTL. Redis untuk queue (BullMQ) wajib `noeviction`; eviction hanya untuk Redis cache.
- [ ] **Connection Pooling & Reconnect** di `ioredis`.
- [ ] **Error Handling**: fallback saat Redis down (jangan crash API).
- [ ] **Scaling** (konsep):
  - Replication (master-replica).
  - Redis Sentinel (high availability).
  - Redis Cluster (sharding).
- [ ] **Anti-pattern yang harus dihindari**:
  - `KEYS *` di produksi → pakai `SCAN`.
  - Menyimpan data besar/Blob di Redis.
  - Redis sebagai primary database.
  - Tidak set TTL → memori penuh.

---

## Fase 7 — Advanced / Opsional

- [ ] Redis JSON (`RedisJSON`) untuk menyimpan dokumen JSON.
- [ ] Redis Search (`RediSearch`) untuk full-text search.
- [ ] Redis TimeSeries untuk data time-series.
- [ ] Redis Bloom Filter untuk "mungkin ada / pasti tidak ada".
- [ ] Redis Graph (sudah discontinued, skip).
- [ ] Redis OM untuk Node.js (object mapping).
- [ ] Sertifikasi / materi lanjutan: redis.io/university (gratis).

---

## Sumber Belajar

| Sumber | Link | Keterangan |
|--------|------|------------|
| Dokumentasi resmi | https://redis.io/docs | Referensi utama command |
| Redis University | https://redis.io/university | Kursus gratis |
| Try Redis | https://try.redis.io | Latihan interaktif |
| ioredis docs | https://github.com/redis/ioredis | Library Node.js |
| BullMQ docs | https://docs.bullmq.io | Job queue untuk NestJS |
| RedisInsight | https://redis.io/insight | GUI resmi |

---

## Checklist Ringkas (Ringkasan Kemampuan Akhir)

- [ ] Mahir `redis-cli` dan semua tipe data dasar.
- [ ] Bisa setup Redis di NestJS (ioredis / cache-manager).
- [ ] Paham caching patterns + cache invalidation.
- [ ] Bisa implement rate limiter, session, distributed lock.
- [ ] Paham Pub/Sub dan Streams/BullMQ.
- [ ] Paham persistence (RDB/AOF), eviction policy, dan keamanan.
- [ ] Siap deploy Redis ke produksi (Docker + monitoring + backup).

---

*Selamat belajar! Mulai dari Fase 1, praktik langsung di setiap fase. Jangan hanya membaca — tulis kode dan jalankan.*
