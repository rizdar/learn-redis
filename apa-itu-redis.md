# Apa Itu Redis?

## Redis secara singkat

Redis (**Re**mote **Di**ctionary **S**erver) adalah **in-memory data store** — database yang menyimpan data di RAM, bukan di disk. Secara teknis dia termasuk keluarga *key-value store*, tapi dengan struktur data yang jauh lebih kaya daripada sekadar "key → string".

## Karakteristik inti

| Aspek | Penjelasan |
|-------|-----------|
| **In-memory** | Data hidup di RAM → baca/tulis sangat cepat (sub-milidetik). Tapi artinya data *volatile* (hilang kalau restart), kecuali dikonfigurasi persistence. |
| **Struktur data kaya** | String, Hash, List, Set, Sorted Set, Stream, dan lainnya — bukan cuma key-value polos. |
| **Single-threaded event loop** | Satu command dieksekusi satu per satu → operasi atomik alami, tapi command lambat memblok semua. |
| **Serbaguna** | Bisa jadi cache, session store, queue, pub/sub, leaderboard, counter, distributed lock. |

## Kenapa cepat?

1. Data di RAM (tanpa I/O disk di jalur utama).
2. Protokol & struktur datanya sederhana.
3. Tanpa *locking* antar-thread (karena single-threaded).

## Posisinya di stack (NestJS + React)

Redis **bukan pengganti** PostgreSQL/MySQL — dia pendamping. Pola paling umum:

```
Request → NestJS → cek Redis (cache) → hit? return cepat
                          ↓ miss
                      query DB → simpan ke Redis → return
```

Jadi perannya biasanya:

- **Cache** hasil query/API (mengurangi beban DB)
- **Session store** (supaya bisa scale horizontal)
- **Rate limiter** (membatasi request)
- **Queue** (BullMQ untuk background job)
- **Realtime** (Pub/Sub → WebSocket ke React)

## Analogi sederhana

Kalau database utama (PostgreSQL) itu **gudang besar** yang lambat diakses, Redis itu **meja kerja** di sampingmu — barang yang sering dipakai ditaruh di meja supaya cepat diambil. Tapi mejanya terbatas (RAM mahal), jadi cuma simpan yang penting dan sering dipakai.
