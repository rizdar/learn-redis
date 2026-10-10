# Fase 2 — Integrasi Redis dengan NestJS

> Fokus: setup ioredis, bikin `RedisModule`, lalu API counter, leaderboard, dan rate limiter.
> Prasyarat: Redis sudah jalan (`docker run -d --name redis -p 6379:6379 redis:8`).

---

## 1. Pilih Library

| Library | Keterangan |
|---------|-----------|
| **ioredis** | ⭐ Direkomendasikan. Fitur lengkap, dukungan cluster, pipeline, Lua, `defineCommand`. |
| `redis` (node-redis v4) | Official, API modern, tapi ekosistemnya lebih muda. |
| `@nestjs/cache-manager` | Abstraksi cache bawaan NestJS (bukan akses Redis penuh). |

Kita pakai **ioredis** — paling fleksibel dan paling umum di produksi.

---

## 2. Instalasi

```bash
npm install ioredis
npm install @nestjs/config   # untuk baca .env
```

Buat file `.env`:

```env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
```

---

## 3. Struktur Folder

```
src/
├── redis/
│   ├── redis.module.ts      # provider REDIS_CLIENT (global)
│   └── redis.service.ts     # wrapper helper
├── counter/
│   ├── counter.controller.ts
│   └── counter.service.ts
├── leaderboard/
│   ├── leaderboard.controller.ts
│   └── leaderboard.service.ts
└── app.module.ts
```

---

## 4. Membuat RedisModule (Global)

`src/redis/redis.module.ts`:

```ts
import { Global, Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import Redis from 'ioredis';
import { RedisService } from './redis.service';

@Global() // ⬅️ cukup import sekali, bisa dipakai semua module
@Module({
  providers: [
    {
      provide: 'REDIS_CLIENT',
      inject: [ConfigService],
      useFactory: (config: ConfigService) => {
        return new Redis({
          host: config.get('REDIS_HOST', 'localhost'),
          port: config.get<number>('REDIS_PORT', 6379),
          password: config.get('REDIS_PASSWORD') || undefined,
          db: config.get<number>('REDIS_DB', 0),

          // Opsi produksi (di Fase 6 dibahas lebih dalam)
          enableReadyCheck: true,
          maxRetriesPerRequest: null, // wajib null kalau nanti dipakai BullMQ
        });
      },
    },
    RedisService,
  ],
  exports: ['REDIS_CLIENT', RedisService],
})
export class RedisModule {}
```

Kenapa pakai token `'REDIS_CLIENT'`? Karena kita membuat instance ioredis lewat `useFactory`, bukan class. Token string memudahkan injeksi.

---

## 5. Membuat RedisService (Wrapper)

`src/redis/redis.service.ts`:

```ts
import { Inject, Injectable, OnModuleDestroy } from '@nestjs/common';
import Redis from 'ioredis';

@Injectable()
export class RedisService implements OnModuleDestroy {
  constructor(
    @Inject('REDIS_CLIENT') private readonly redis: Redis,
  ) {}

  /** Akses client mentah kalau butuh command yang belum di-wrapper */
  getClient(): Redis {
    return this.redis;
  }

  // ---- String ----
  async get<T>(key: string): Promise<T | null> {
    const value = await this.redis.get(key);
    return value ? (JSON.parse(value) as T) : null;
  }

  async set(key: string, value: unknown, ttlSeconds?: number): Promise<void> {
    const serialized = JSON.stringify(value);
    if (ttlSeconds) {
      await this.redis.set(key, serialized, 'EX', ttlSeconds);
    } else {
      await this.redis.set(key, serialized);
    }
  }

  async incr(key: string): Promise<number> {
    return this.redis.incr(key);
  }

  async expire(key: string, seconds: number): Promise<number> {
    return this.redis.expire(key, seconds);
  }

  async del(...keys: string[]): Promise<number> {
    return this.redis.del(...keys);
  }

  // ---- Hash ----
  async hset(key: string, field: string, value: unknown): Promise<number> {
    return this.redis.hset(key, field, JSON.stringify(value));
  }

  async hget<T>(key: string, field: string): Promise<T | null> {
    const value = await this.redis.hget(key, field);
    return value ? (JSON.parse(value) as T) : null;
  }

  // ---- Sorted Set ----
  async zadd(key: string, score: number, member: string): Promise<number> {
    return this.redis.zadd(key, score, member);
  }

  async zrevrangeWithScores(
    key: string,
    start: number,
    stop: number,
  ): Promise<{ member: string; score: number }[]> {
    const raw = await this.redis.zrevrange(key, start, stop, 'WITHSCORES');
    const result: { member: string; score: number }[] = [];
    for (let i = 0; i < raw.length; i += 2) {
      result.push({ member: raw[i], score: Number(raw[i + 1]) });
    }
    return result;
  }

  async onModuleDestroy() {
    await this.redis.quit(); // tutup koneksi saat aplikasi mati
  }
}
```

> **Kenapa dibungkus?** Supaya logika JSON serialize/parse dan command yang sering dipakai terpusat. Controller tidak perlu tahu detail ioredis.

---

## 6. Daftarkan di AppModule

`src/app.module.ts`:

```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { RedisModule } from './redis/redis.module';
import { CounterModule } from './counter/counter.module';
import { LeaderboardModule } from './leaderboard/leaderboard.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    RedisModule, // global, jadi module lain tinggal pakai RedisService
    CounterModule,
    LeaderboardModule,
  ],
})
export class AppModule {}
```

---

## 7. Contoh 1 — Counter API (String)

`src/counter/counter.service.ts`:

```ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '../redis/redis.service';

@Injectable()
export class CounterService {
  constructor(private readonly redis: RedisService) {}

  async increment(name: string): Promise<number> {
    return this.redis.incr(`counter:${name}`); // INCR atomik
  }

  async get(name: string): Promise<number> {
    return (await this.redis.get<number>(`counter:${name}`)) ?? 0;
  }
}
```

`src/counter/counter.controller.ts`:

```ts
import { Controller, Get, Param, Post } from '@nestjs/common';
import { CounterService } from './counter.service';

@Controller('counter')
export class CounterController {
  constructor(private readonly counterService: CounterService) {}

  @Post(':name/incr')
  increment(@Param('name') name: string) {
    return this.counterService.increment(name);
  }

  @Get(':name')
  get(@Param('name') name: string) {
    return this.counterService.get(name);
  }
}
```

Coba:

```bash
curl -X POST localhost:3000/counter/visits/incr   # → 1
curl -X POST localhost:3000/counter/visits/incr   # → 2
curl localhost:3000/counter/visits                # → 2
```

---

## 8. Contoh 2 — Leaderboard (Sorted Set)

`src/leaderboard/leaderboard.service.ts`:

```ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '../redis/redis.service';

@Injectable()
export class LeaderboardService {
  constructor(private readonly redis: RedisService) {}

  async addScore(player: string, score: number): Promise<void> {
    // ZINCRBY: tambah skor (atomik), bukannya overwrite
    await this.redis.getClient().zincrby('leaderboard', score, player);
  }

  async top(n = 10) {
    return this.redis.zrevrangeWithScores('leaderboard', 0, n - 1);
  }
}
```

`src/leaderboard/leaderboard.controller.ts`:

```ts
import { Body, Controller, Get, Post } from '@nestjs/common';
import { LeaderboardService } from './leaderboard.service';

@Controller('leaderboard')
export class LeaderboardController {
  constructor(private readonly leaderboardService: LeaderboardService) {}

  @Post('score')
  addScore(@Body() body: { player: string; score: number }) {
    return this.leaderboardService.addScore(body.player, body.score);
  }

  @Get('top')
  top() {
    return this.leaderboardService.top(10);
  }
}
```

Coba:

```bash
curl -X POST localhost:3000/leaderboard/score \
  -H "Content-Type: application/json" \
  -d '{"player":"rizdar","score":150}'
curl localhost:3000/leaderboard/top
```

---

## 9. Contoh 3 — Rate Limiter Manual (String, naif)

`src/rate-limit/rate-limit.service.ts`:

```ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '../redis/redis.service';

@Injectable()
export class RateLimitService {
  constructor(private readonly redis: RedisService) {}

  /**
   * Batasi: maksimal `limit` request per `windowSec` detik.
   * ⚠️ VERSI NAIEF — punya race condition (lihat catatan di bawah).
   */
  async isAllowed(userId: string, limit: number, windowSec: number) {
    const key = `rate:${userId}`;
    const count = await this.redis.incr(key); // atomic

    if (count === 1) {
      // set TTL hanya saat request pertama
      await this.redis.expire(key, windowSec); // ⚠️ bukan atomic dengan INCR di atas
    }

    return count <= limit;
  }
}
```

**Catatan race condition (jembatan ke Fase 4):**

```
INCR rate:user1      → 1   (berhasil)
                         → server CRASH di antara dua command ini!
EXPIRE rate:user1 60 → tidak pernah jalan
                         → key jadi TANPA TTL → user keblokir selamanya
```

Versi ini belum atomic (dua command terpisah). Solusinya pakai **Lua script** (satu unit atomik) — dibahas di Fase 4.

---

## 10. Error Handling Dasar

Jangan biarkan Redis down menjatuhkan seluruh API:

```ts
async get<T>(key: string): Promise<T | null> {
  try {
    const value = await this.redis.get(key);
    return value ? (JSON.parse(value) as T) : null;
  } catch (err) {
    // log, lalu fallback (misal query DB langsung)
    return null;
  }
}
```

Prinsipnya: **Redis adalah cache/akselerator** — kalau dia mati, aplikasi harus tetap jalan (fallback ke DB), bukan ikut crash.

---

## Proyek Mini (Latihan)

Buat endpoint-endpoint ini di project NestJS-mu:

1. `POST /counter/:name/incr` dan `GET /counter/:name` — counter atomik.
2. `POST /leaderboard/score` dan `GET /leaderboard/top` — leaderboard pakai Sorted Set.
3. `POST /session` — simpan session ke Hash (`HSET`), dan `GET /session/:id` (`HGET`).
4. Bonus: rate limiter manual di salah satu endpoint, amati bahwa TTL baru ter-set di request pertama (`TTL` di redis-cli).

---

## Ringkasan

| Konsep | Poin |
|--------|------|
| `RedisModule` | `@Global()`, provider `REDIS_CLIENT` via `useFactory` + `ConfigService` |
| `RedisService` | Wrapper untuk JSON serialize + command umum |
| Counter | `INCR` — atomic |
| Leaderboard | `ZINCRBY` + `ZREVRANGE ... WITHSCORES` |
| Rate limiter naif | `INCR` + `EXPIRE` — **belum atomic**, diperbaiki dengan Lua di Fase 4 |
| Error handling | Redis down ≠ aplikasi crash |

---

*Lanjut ke Fase 3: Caching Patterns (cache-aside, TTL, invalidation).*
