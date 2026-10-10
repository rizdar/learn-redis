# Tipe Data Redis

> Fase 1 dari roadmap. Tipe data adalah fondasi Redis — sebelum belajar caching, queue, atau lock, kuasai dulu kapan pakai tipe yang mana.

Redis menyimpan data dalam bentuk **key → value**, dan value-nya bisa salah satu dari beberapa tipe. Pilihan tipe data menentukan operasi apa yang bisa kamu lakukan (dan seberapa efisien).

---

## Ringkasan

| Tipe | Bentuk | Contoh Use Case |
|------|--------|-----------------|
| **String** | Teks / angka / binary | Cache nilai, counter, session, rate limiter |
| **Hash** | Map field → value | Profil user, objek dengan banyak atribut |
| **List** | Urutan berurut (bisa duplikat) | Antrian (queue), activity feed, recent items |
| **Set** | Kumpulan unik (tanpa urutan) | Tag, unique visitors, relasi (teman bersama) |
| **Sorted Set** | Set unik + skor (terurut) | Leaderboard, ranking, priority queue |

---

## 1. String

Tipe paling dasar. Satu key → satu nilai. Nilai bisa teks, angka, atau binary (maks 512 MB).

### Command utama

```bash
SET user:1:name "Rizdar"        # simpan nilai
GET user:1:name                 # ambil nilai → "Rizdar"

SETEX session:abc123 3600 "data"  # simpan + TTL 3600 detik (atomik)
SETNX lock:payment "owner"        # set hanya jika key belum ada (untuk lock)

INCR counter:visits             # +1 → atomic, ideal untuk counter
INCRBY counter:visits 10        # +10
DECR counter:visits             # -1

MSET a 1 b 2 c 3                # set banyak sekaligus
MGET a b c                      # get banyak sekaligus → ["1","2","3"]
```

### Use case khas

- **Cache nilai sederhana**: `GET product:123` → JSON string.
- **Counter atomik**: jumlah view, likes, stock. `INCR` aman meski banyak request bersamaan (karena single-threaded).
- **Rate limiter**: `INCR` + `EXPIRE` (versi naif, nanti di-atomic-kan dengan Lua).
- **Distributed lock**: `SET key token NX PX ttl`.
- **Session storage**: simpan session sebagai JSON string.

### Contoh di NestJS (ioredis)

```ts
await redis.set(`product:${id}`, JSON.stringify(product), 'EX', 300);
const cached = JSON.parse(await redis.get(`product:${id}`));
await redis.incr('counter:visits');
```

---

## 2. Hash

Satu key → banyak pasangan `field → value`. Cocok untuk **objek** yang atributnya diakses/diubah satu per satu.

### Command utama

```bash
HSET user:1 name "Rizdar" email "rizdar@mail.com" age 25
HGET user:1 name                # "Rizdar"
HMGET user:1 name email         # ["Rizdar","rizdar@mail.com"]
HGETALL user:1                  # semua field + value
HINCRBY user:1 age 1            # increment field angka
HDEL user:1 age                 # hapus satu field
HEXISTS user:1 email            # 1 / 0
HLEN user:1                     # jumlah field
```

### Use case khas

- **Profil user / objek data**: ganti `user:1` (String JSON) dengan Hash kalau kamu sering update satu field saja. Tidak perlu rewrite seluruh JSON.
- **Session detail** (field: userId, role, expiresAt).
- **Statistik per item**: `HSET product:123 views 1000 purchases 5`.

### Kapan pilih Hash vs String(JSON)?

| Situasi | Pilihan |
|---------|---------|
| Akses/update satu field saja | **Hash** (hemat, `HGET`/`HSET` O(1)) |
| Selalu baca/tulis seluruh objek | **String JSON** (lebih simpel) |
| Objek besar + sering `HGETALL` | Hati-hati — `HGETALL` itu O(N) dan memblok server (lihat catatan single-threaded) |

---

## 3. List

Urutan nilai berurut (boleh duplikat). Bisa dipakai sebagai **stack** atau **queue**.

### Command utama

```bash
LPUSH list:recent "item3" "item2" "item1"   # dorong ke kiri
RPUSH list:recent "item4"                    # dorong ke kanan
LRANGE list:recent 0 -1                      # ambil semua (0 = awal, -1 = akhir)
LRANGE list:recent 0 9                       # 10 item pertama
LPOP list:recent                             # ambil + hapus dari kiri
RPOP list:recent                             # ambil + hapus dari kanan
LLEN list:recent                             # jumlah item
LTRIM list:recent 0 99                       # simpan hanya 100 item terbaru
```

- **Queue**: `LPUSH` (produksi) + `RPOP` (konsumsi), atau `RPUSH` + `LPOP`.
- **Blocking pop**: `BLPOP`/`BRPOP` — menunggu sampai ada item (untuk antrian sederhana).

### Use case khas

- **Antrian sederhana** (job queue ringan). Untuk queue serius → pakai **Streams/BullMQ**.
- **Recent items / activity feed**: `LPUSH` + `LTRIM` supaya cuma menyimpan N item terbaru.
- **Stack** (LIFO).

```bash
# Simpan 10 aktivitas terbaru user
LPUSH user:1:activity "login"
LPUSH user:1:activity "view product 123"
LTRIM user:1:activity 0 9
```

---

## 4. Set

Kumpulan nilai **unik**, tanpa urutan. Operasi antar-set sangat cepat (intersection, union, diff).

### Command utama

```bash
SADD article:1:tags "redis" "nestjs" "react"
SADD article:1:tags "redis"        # duplikat → diabaikan
SMEMBERS article:1:tags            # semua anggota (acak, O(N))
SISMEMBER article:1:tags "redis"   # 1 (ada) / 0 (tidak) → O(1)
SCARD article:1:tags               # jumlah anggota
SREM article:1:tags "react"        # hapus anggota

SINTER setA setB                   # irisan (di kedua set)
SUNION setA setB                   # gabungan
SDIFF setA setB                    # di A tapi tidak di B
SPOP setA                          # ambil + hapus satu (acak)
```

### Use case khas

- **Tag / kategori** pada artikel/produk.
- **Unique visitors**: `SADD visitors:today <ip>` → `SCARD visitors:today`.
- **Relasi sosial**: `SINTER user:1:friends user:2:friends` → teman bersama.
- **Blacklist / whitelist**: cek keanggotaan dengan `SISMEMBER` (O(1)).

---

## 5. Sorted Set

Seperti Set, tapi setiap anggota punya **skor** (angka) yang menentukan urutan. Ini tipe data **bintang** untuk leaderboard.

### Command utama

```bash
ZADD leaderboard 100 "playerA"
ZADD leaderboard 250 "playerB" 150 "playerC"
ZINCRBY leaderboard 50 "playerA"        # tambah skor (atomik)

ZRANGE leaderboard 0 -1                  # urut naik (skor kecil → besar)
ZREVRANGE leaderboard 0 -1               # urut turun (skor besar → kecil)
ZREVRANGE leaderboard 0 9 WITHSCORES     # top 10 + skor

ZRANK leaderboard "playerA"              # peringkat (0-based, naik)
ZREVRANK leaderboard "playerA"           # peringkat (dari terbesar)
ZSCORE leaderboard "playerA"             # skor anggota
ZCARD leaderboard                        # jumlah anggota
ZREM leaderboard "playerA"               # hapus anggota
ZPOPMAX leaderboard                      # ambil + hapus skor tertinggi
```

### Use case khas

- **Leaderboard game**: `ZADD` + `ZINCRBY` + `ZREVRANGE ... WITHSCORES`.
- **Ranking berdasarkan waktu**: skor = timestamp, `ZADD news <timestamp> "artikel:123"`.
- **Priority queue**: skor = prioritas.
- **Rate limiting berbasis window**: skor = timestamp request (untuk sliding window).

```bash
# Ambil top 10 pemain beserta skornya
ZREVRANGE leaderboard 0 9 WITHSCORES
```

---

## Kapan Pakai Tipe yang Mana?

| Kebutuhan | Tipe |
|-----------|------|
| Simpan satu nilai / counter / cache | **String** |
| Objek dengan field terpisah, update sebagian | **Hash** |
| Antrian / urutan / feed terbaru | **List** |
| Keunikan / keanggotaan / relasi himpunan | **Set** |
| Urutan + skor / leaderboard / ranking | **Sorted Set** |
| Log event / antrian dengan consumer group | **Stream** (Fase 4) |

---

## Catatan Penting (single-threaded)

Redis mengeksekusi command **satu per satu** di satu event loop. Jadi hindari command yang **O(N) pada data besar** karena memblok seluruh server:

- `HGETALL` pada hash raksasa → pertimbangkan `HSCAN` / `HMGET` field tertentu.
- `SMEMBERS` pada set besar → pertimbangkan `SSCAN`.
- `LRANGE list 0 -1` pada list panjang → batasi range.
- `KEYS *` → selalu pakai `SCAN` di produksi.

> Aturan praktis: operasi yang kamu jalankan di produksi idealnya **O(1)** atau **O(log N)**. Operasi O(N) hanya untuk data kecil atau maintenance.

---

*Lanjut ke Fase 2: integrasi dengan NestJS (ioredis + RedisModule).*
