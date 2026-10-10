# Learn Redis

Repo belajar **Redis** untuk fullstack developer (NestJS + ReactJS). Berisi dokumentasi, catatan, dan progres pembelajaran — ditulis sambil praktik langsung.

> Tujuan: memahami Redis dari dasar sampai siap produksi, dengan fokus integrasi NestJS.

---

## Isi Repo

| File | Isi |
|------|-----|
| [01-apa-itu-redis.md](01-apa-itu-redis.md) | Pengertian & konsep dasar Redis |
| [02-redis-roadmap.md](02-redis-roadmap.md) | Roadmap belajar lengkap (Fase 0–7) |

---

## Progres Belajar

> Centang `[x]` setiap kali selesai satu fase.

- [ ] **Fase 0 — Persiapan**: install Redis (Docker), redis-cli, GUI client
- [ ] **Fase 1 — Konsep Dasar**: tipe data (String, Hash, List, Set, Sorted Set), expiry, single-threaded
- [ ] **Fase 2 — Integrasi NestJS**: ioredis, RedisModule, counter/leaderboard API
- [ ] **Fase 3 — Caching Patterns**: cache-aside, TTL, invalidation, cache stampede
- [ ] **Fase 4 — Fitur Lanjutan**: Pub/Sub, Streams, Lua, Pipeline, persistence (RDB/AOF)
- [ ] **Fase 5 — Kasus Nyata**: rate limiter, session, distributed lock, BullMQ, realtime
- [ ] **Fase 6 — Produksi & Best Practice**: keamanan (ACL), monitoring, eviction, scaling
- [ ] **Fase 7 — Advanced (Opsional)**: RedisJSON, RediSearch, Redis OM

---

## Prasyarat

- Node.js & NestJS
- Redis (via Docker): `docker run -d --name redis -p 6379:6379 redis:8`
- Redis client: `redis-cli` + RedisInsight

---

## Sumber Belajar

- [Dokumentasi resmi Redis](https://redis.io/docs)
- [Redis University](https://redis.io/university) (kursus gratis)
- [Try Redis](https://try.redis.io) (latihan interaktif)
- [ioredis](https://github.com/redis/ioredis) — library Node.js
- [BullMQ](https://docs.bullmq.io) — job queue untuk NestJS

---

## Kontribusi

Repo ini dokumentasi belajar pribadi, tapi kalau menemukan kesalahan atau ingin menambahkan materi, silakan buka *issue* atau *pull request*.
