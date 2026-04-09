# 🎫 Zammad Docker Compose — Dokumentasi Teknis

Zammad adalah platform helpdesk & ticketing open-source. Dokumentasi ini menjelaskan secara teknis bagaimana project ini dikonfigurasi, mulai dari base image, penyatuan service, isolasi jaringan, hingga konfigurasi reverse proxy SSL/TLS.

---

## 📁 Struktur File

```
project/
├── docker-compose.yml        ← Konfigurasi utama seluruh service
├── .env.example              ← Template environment variables
├── .env                      ← File env Anda (buat dari .env.example)
├── nginx/                    ← [Profile: nginx] Reverse proxy Nginx
│   ├── conf.d/
│   │   └── zammad.conf       ← Konfigurasi Nginx reverse proxy + SSL
│   └── ssl/
│       ├── zammad.crt        ← SSL Certificate (digunakan juga oleh Traefik)
│       └── zammad.key        ← SSL Private Key (digunakan juga oleh Traefik)
└── traefik/                  ← [Profile: traefik] Reverse proxy Traefik
    ├── traefik.yml           ← Static config (entrypoints, providers)
    └── dynamic/
        └── zammad.yml        ← Dynamic config (router, middleware, TLS)
```

---

## 1. Base Image, Multi-Stage Build & `.dockerignore`

### 🔍 Base Image Zammad

Seluruh service Zammad dalam project ini **tidak membangun image dari scratch**. Semua service menggunakan **official pre-built image** dari Docker Hub:

```yaml
image: zammad/zammad-docker-compose:latest
```

Image `zammad/zammad-docker-compose:latest` adalah image **all-in-one** yang sudah dikemas oleh tim Zammad. Di dalamnya sudah berisi:
- **Ruby on Rails** (backend framework)
- **Ruby runtime & gem dependencies**
- Script entrypoint untuk setiap mode (`nginx`, `railsserver`, `websocket`, `scheduler`, `init`)

Setiap service hanya membedakan **command** yang dijalankan:

```yaml
zammad-nginx:       command: ["zammad-nginx"]
zammad-railsserver: command: ["zammad-railsserver"]
zammad-websocket:   command: ["zammad-websocket"]
```

Service pendukung menggunakan image resmi masing-masing vendor:

| Service | Base Image | Alasan Pemilihan |
|---|---|---|
| PostgreSQL | `postgres:15-alpine` | Alpine = ringan, versi 15 = LTS stabil |
| Elasticsearch | `elasticsearch:8.12.2` | Versi spesifik untuk kompatibilitas Zammad |
| Memcached | `memcached:1.6-alpine` | Alpine, ringan untuk cache session |
| Redis | `redis:7-alpine` | Alpine, versi 7 stabil untuk queue |
| Reverse Proxy | `nginx:alpine` | Alpine, minimal footprint |

### 🏗️ Prinsip Multi-Stage Build

Karena project ini menggunakan **pre-built image**, multi-stage build tidak diimplementasikan via `Dockerfile` eksplisit. Namun, **konsepnya tetap diterapkan secara arsitektur** melalui pemisahan concern antar container:

```
[Build/Init Stage]          [Runtime Stage]
zammad-init                 zammad-nginx
(inisialisasi DB,     →     zammad-railsserver
 migrasi, setup)            zammad-websocket
                            zammad-scheduler
```

`zammad-init` berperan sebagai **"build stage"** — hanya berjalan sekali untuk menyiapkan database lalu `exit 0`. Service runtime tidak memerlukan tools inisialisasi, persis seperti filosofi multi-stage build yang memisahkan build tools dari runtime image.

> **Referensi:** Jika membuat Dockerfile sendiri untuk Zammad kustom, pola multi-stage yang ideal adalah:
> ```dockerfile
> # Stage 1: Builder (install gem, compile assets)
> FROM ruby:3.x AS builder
> RUN bundle install && rake assets:precompile
>
> # Stage 2: Runtime (hanya copy hasil build, tanpa dev tools)
> FROM ruby:3.x-slim AS runtime
> COPY --from=builder /app /app
> ```

### 📄 `.dockerignore` untuk Efisiensi Context

Project ini tidak memerlukan `.dockerignore` karena tidak ada custom `Dockerfile` yang di-build. Namun jika ditambahkan custom build, `.dockerignore` yang direkomendasikan adalah:

```
# Exclude development & sensitive files
.env
*.log
node_modules/
.git/
.gitignore
nginx/ssl/*.key    # Jangan sertakan private key dalam image!
tmp/
coverage/
```

Tujuan `.dockerignore` adalah memperkecil **build context** yang dikirim ke Docker daemon, sehingga proses build lebih cepat dan image tidak mengandung file sensitif.

---

## 2. Konfigurasi `docker-compose.yml`: Penyatuan Service & Isolasi Jaringan

### 🔗 Penyatuan Service Zammad

File `docker-compose.yml` menyatukan **9 service** dalam satu stack dengan dependency chain yang ketat:

```
[zammad-postgresql] ─┐
[zammad-elasticsearch] ─┤─→ [zammad-init] ─→ [zammad-nginx]
[zammad-memcached] ──┤              └──→ [zammad-railsserver] ─→ [zammad-nginx]
[zammad-redis] ──────┘              └──→ [zammad-websocket]
                                    └──→ [zammad-scheduler]
                                                    ↓
                                         [reverse-proxy-ssl]
                                          (port 80 / 443)
```

**Dependency condition** yang digunakan sangat presisi:

```yaml
depends_on:
  zammad-postgresql:
    condition: service_healthy              # Tunggu sampai healthcheck lulus
  zammad-init:
    condition: service_completed_successfully  # Tunggu sampai exit 0
  zammad-railsserver:
    condition: service_started              # Cukup sudah berjalan
```

Ini mencegah **race condition**, misalnya `railsserver` tidak akan mulai sebelum database siap diakses.

---

### 🌐 Arsitektur Isolasi Jaringan

Project menggunakan **3 network Docker** yang saling terisolasi:

```
INTERNET
    │
    ▼  port 80 / 443 (exposed ke host)
[reverse-proxy-ssl]
    │  network: proxy
    ▼
[zammad-nginx:8080]   ← hanya expose internal, tidak ke host
    │  network: frontend
    ├──► [zammad-railsserver]
    │         │  network: backend (internal=true)
    │         ├──► [zammad-postgresql]    🔒
    │         ├──► [zammad-elasticsearch] 🔒
    │         ├──► [zammad-memcached]     🔒
    │         └──► [zammad-redis]         🔒
    └──► [zammad-websocket]
```

#### Tabel Network:

| Network | `internal` | Container yang masuk | Tujuan |
|---|:---:|---|---|
| `zammad-backend` | ✅ `true` | postgresql, elasticsearch, memcached, redis, railsserver, scheduler, init | Database layer — tidak bisa akses internet |
| `zammad-frontend` | ❌ | nginx, railsserver, websocket, scheduler, init | Application layer — komunikasi antar service app |
| `zammad-proxy` | ❌ | reverse-proxy-ssl, zammad-nginx | Jalur masuk dari internet ke nginx internal |

#### Konfigurasi kritis — `backend` network terisolasi penuh:

```yaml
networks:
  backend:
    driver: bridge
    internal: true   # ← Tidak bisa akses internet langsung!
    name: zammad-backend
```

Flag `internal: true` berarti container di network ini **tidak dapat melakukan request keluar ke internet**, sehingga database tidak dapat dieksfiltrasi meski container berhasil disusupi.

#### Port Exposure yang Aman:

```yaml
# zammad-nginx — hanya internal
expose:
  - "8080"   # ← expose (antar container), BUKAN ports (ke host)

# reverse-proxy-ssl — satu-satunya pintu masuk
ports:
  - "80:80"    # HTTP  → redirect ke HTTPS
  - "443:443"  # HTTPS → SSL termination
```

Hanya `reverse-proxy-ssl` yang membuka port ke host. `zammad-nginx` hanya bisa diakses dari container lain di network `proxy`.

---

### 📦 Daftar Lengkap Service & Fungsinya

| Service | Fungsi | Network |
|---|---|---|
| `zammad-init` | Inisialisasi DB & konfigurasi awal | backend + frontend |
| `zammad-nginx` | Web server internal, expose port 8080 | frontend + backend + proxy |
| `zammad-railsserver` | Core aplikasi Ruby on Rails | backend + frontend |
| `zammad-websocket` | Koneksi real-time (notifikasi, chat) | frontend |
| `zammad-scheduler` | Background jobs (email, dll) | backend + frontend |
| `reverse-proxy-ssl` | SSL termination, port 80/443 ke host | proxy |
| `zammad-postgresql` | Database utama | backend |
| `zammad-elasticsearch` | Search engine full-text | backend |
| `zammad-memcached` | Cache session & performa | backend |
| `zammad-redis` | Queue & pub/sub background jobs | backend |

---

## 3. Reverse Proxy Nginx: SSL/TLS Termination

### 🏗️ Arsitektur SSL Termination

```
Client Browser
    │  HTTPS (port 443) — terenkripsi TLS
    ▼
[reverse-proxy-ssl]  ← SSL diproses & didekripsi di sini
    │  HTTP (port 8080) — plain HTTP tapi hanya di dalam Docker network
    ▼
[zammad-nginx]       ← Meneruskan ke railsserver
    │
    ▼
[zammad-railsserver]
```

**SSL Termination** artinya enkripsi SSL/TLS **berakhir** di reverse proxy. Komunikasi antar container di dalam Docker network berjalan plain HTTP — ini aman karena sudah terisolasi dalam jaringan Docker internal.

---

### 📋 Konfigurasi Nginx — `nginx/conf.d/zammad.conf`

Terdiri dari **2 server block**:

#### Block 1 — HTTP → HTTPS Redirect (port 80)

```nginx
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}
```

Semua akses HTTP otomatis diarahkan ke HTTPS. Kode `301` = *permanent redirect*, yang juga membantu SEO dan browser caching.

#### Block 2 — HTTPS Server (port 443)

**a) Certificate SSL:**
```nginx
ssl_certificate     /etc/nginx/ssl/zammad.crt;
ssl_certificate_key /etc/nginx/ssl/zammad.key;
```
File ini di-mount dari `./nginx/ssl/` di host (berisi `zammad.crt` dan `zammad.key`).

**b) Hardening Protocol & Cipher:**
```nginx
ssl_protocols             TLSv1.2 TLSv1.3;   # Hanya protokol modern
ssl_ciphers               ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...;
ssl_prefer_server_ciphers off;    # Biarkan client pilih cipher terbaik
ssl_session_cache         shared:SSL:10m;    # Cache SSL session agar lebih cepat
ssl_session_tickets       off;              # Nonaktifkan untuk forward secrecy
```

**c) Security Headers:**
```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
# ↑ HSTS: paksa browser pakai HTTPS selama ~2 tahun, tidak bisa downgrade

add_header X-Frame-Options        SAMEORIGIN  always;       # Cegah clickjacking
add_header X-Content-Type-Options nosniff     always;       # Cegah MIME sniffing
add_header X-XSS-Protection       "1; mode=block" always;  # Proteksi XSS
add_header Referrer-Policy        "strict-origin-when-cross-origin" always;
```

**d) Proxy Forwarding ke Zammad:**
```nginx
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto https;  # Beritahu Zammad bahwa request asalnya HTTPS

location / {
    proxy_pass http://zammad-nginx:8080;   # Forward ke Zammad internal
}
```

**e) WebSocket Support untuk real-time Zammad:**
```nginx
proxy_http_version 1.1;
proxy_set_header   Upgrade    $http_upgrade;
proxy_set_header   Connection "upgrade";
```
Konfigurasi ini penting agar fitur **real-time notification dan chat** Zammad (yang menggunakan WebSocket) tetap berfungsi melalui reverse proxy.

---

### 🔒 Mount Volume Nginx — Read-Only

```yaml
reverse-proxy-ssl:
  volumes:
    - ./nginx/conf.d:/etc/nginx/conf.d:ro   # Config nginx (read-only)
    - ./nginx/ssl:/etc/nginx/ssl:ro          # SSL cert & key (read-only)
```

Flag `:ro` (read-only) memastikan container Nginx **tidak bisa memodifikasi** file konfigurasi atau certificate dari dalam container — best practice keamanan untuk mencegah tampering dari dalam container.

---

### 🔑 Ringkasan Alur Keamanan End-to-End

```
1. Client → reverse-proxy-ssl (HTTPS 443)
   └─ SSL handshake, TLS 1.2/1.3
   └─ Security headers ditambahkan (HSTS, X-Frame-Options, dll)

2. reverse-proxy-ssl → zammad-nginx (HTTP 8080, network: proxy)
   └─ Header X-Forwarded-Proto: https dikirim agar Zammad tahu asal HTTPS
   └─ Komunikasi aman karena hanya dalam Docker internal network

3. zammad-nginx → zammad-railsserver (HTTP, network: frontend)
   └─ Proses request Ruby on Rails

4. zammad-railsserver → database (network: backend, internal: true)
   └─ Database tidak bisa diakses dari luar Docker network sama sekali
```

---

## 🔄 Mengganti Reverse Proxy: Nginx ↔ Traefik

Project ini mendukung **dua pilihan reverse proxy** yang bisa diganti kapan saja tanpa menghapus konfigurasi satu sama lain. Caranya menggunakan **Docker Compose Profiles**.

### Perbandingan Nginx vs Traefik

| Fitur | Nginx | Traefik |
|---|---|---|
| Konfigurasi | File `.conf` statis | YAML hot-reload |
| Dashboard | ❌ | ✅ `http://localhost:8088/dashboard/` |
| SSL | Manual (file cert) | Manual atau otomatis (Let's Encrypt) |
| WebSocket | ✅ (manual headers) | ✅ (middleware) |
| Security headers | ✅ `add_header` | ✅ middleware |
| Cocok untuk | Setup sederhana & familiar | Lingkungan dynamic / microservices |

---

### 🟦 Menjalankan dengan Nginx (default lama)

```bash
docker compose --profile nginx up -d
```

Container yang aktif: `zammad-reverse-proxy-nginx`
Konfigurasi: `nginx/conf.d/zammad.conf`

---

### 🟪 Menjalankan dengan Traefik

```bash
docker compose --profile traefik up -d
```

Container yang aktif: `zammad-reverse-proxy-traefik`
Konfigurasi: `traefik/traefik.yml` + `traefik/dynamic/zammad.yml`
Dashboard Traefik: [http://localhost:8088/dashboard/](http://localhost:8088/dashboard/)

---

### 🔁 Cara Berpindah (contoh: dari Nginx ke Traefik)

```bash
# 1. Hentikan service dengan profile yang sedang aktif
docker compose --profile nginx down

# 2. Jalankan dengan profile yang baru
docker compose --profile traefik up -d
```

> ⚠️ **Jangan jalankan kedua profile sekaligus!** Keduanya menggunakan port 80 dan 443 sehingga akan konflik. Selalu hentikan satu sebelum menjalankan yang lain.

---

### Catatan: SSL Certificate Digunakan Bersama

Baik Nginx maupun Traefik menggunakan **sertifikat SSL yang sama** dari folder `nginx/ssl/`:

```
nginx/ssl/zammad.crt  →  Nginx:   /etc/nginx/ssl/zammad.crt
                      →  Traefik: /etc/traefik/ssl/zammad.crt
nginx/ssl/zammad.key  →  Nginx:   /etc/nginx/ssl/zammad.key
                      →  Traefik: /etc/traefik/ssl/zammad.key
```

Sertifikat saat ini bersifat **self-signed** (cocok untuk development).
Untuk produksi, ganti dengan sertifikat dari **Let's Encrypt** atau CA resmi.

---

## 🚀 Cara Menjalankan

### 1. Salin dan edit file environment
```bash
cp .env.example .env
# Edit .env — wajib ganti POSTGRES_PASSWORD & POSTGRESQL_PASS !
```

### 2. (Khusus Linux) Atur vm.max_map_count untuk Elasticsearch
```bash
sudo sysctl -w vm.max_map_count=262144

# Agar permanen setelah reboot:
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 3. Jalankan Docker Compose
```bash
docker compose up -d
```

> Proses pertama kali membutuhkan waktu **5–10 menit** karena Zammad perlu menginisialisasi database dan mengindeks Elasticsearch.

### 4. Pantau log inisialisasi
```bash
docker compose logs -f zammad-init   # Tunggu sampai "exited with code 0"
docker compose logs -f               # Pantau semua service
```

### 5. Akses Zammad
```
https://localhost    ← lewat HTTPS (reverse proxy port 443)
```

---

## 🔧 Perintah Berguna

```bash
# Melihat status semua container
docker compose ps

# Menghentikan semua container
docker compose down

# Menghentikan dan HAPUS semua data (hati-hati!)
docker compose down -v

# Restart satu service saja
docker compose restart zammad-railsserver

# Masuk ke shell railsserver (untuk debug)
docker compose exec zammad-railsserver bash

# Melihat log satu service
docker compose logs -f zammad-nginx
docker compose logs -f reverse-proxy-ssl
```

---

## ❗ Troubleshooting

**Container `zammad-init` gagal / restart terus:**
- Pastikan Elasticsearch sudah healthy: `docker compose ps zammad-elasticsearch`
- Pastikan `.env` sudah diisi dengan benar, terutama password PostgreSQL

**Elasticsearch gagal start:**
- Jalankan perintah `vm.max_map_count` di langkah 2
- Pastikan RAM host minimal **4 GB**

**Browser menampilkan "Not Secure" / SSL error:**
- Certificate `zammad.crt` bersifat self-signed, browser akan menampilkan warning — ini normal untuk development
- Untuk produksi, ganti dengan certificate dari **Let's Encrypt** atau CA resmi

**Zammad tidak bisa dibuka:**
- Cek log reverse proxy: `docker compose logs reverse-proxy-ssl`
- Cek log nginx internal: `docker compose logs zammad-nginx`
- Pastikan port 80 dan 443 tidak diblokir firewall

---

## 📋 Kebutuhan Minimum

- Docker Engine 24+
- Docker Compose v2+
- RAM: **minimal 4 GB** (rekomendasi 8 GB)
- CPU: SSE4.2 support (hampir semua CPU modern)
- Storage: minimal 10 GB
