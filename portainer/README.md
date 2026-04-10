# Panduan Deploy: Zammad Docker Swarm + Portainer

## Gambaran Arsitektur

```
Internet (HTTP/HTTPS)
        │
        ▼
  ┌─────────────┐
  │   Traefik   │ :80 (→redirect HTTPS), :443 (SSL), :8088 (dashboard)
  └──────┬──────┘
         │ overlay network: zammad-proxy
         ▼
  ┌─────────────┐
  │zammad-nginx │ :8080 (internal)
  └──────┬──────┘
         │ overlay network: zammad-frontend
    ┌────┴────┐──────────────┐
    ▼         ▼              ▼
┌────────┐ ┌────────┐ ┌──────────┐
│ Rails  │ │  WS    │ │Scheduler │
└────┬───┘ └───┬────┘ └─────┬────┘
     │  overlay network: zammad-backend
     ▼
┌──────────────────────────────────┐
│  PostgreSQL │ ES │ Redis │ Cache │
└──────────────────────────────────┘

Portainer (port 9000/9443) ←→ portainer-agent (global, tiap node)
```

---

## Prasyarat di Server Ubuntu

```bash
# 1. Pastikan Docker terinstall (versi 20.10+)
docker --version

# 2. Pastikan Docker Swarm belum aktif atau aktifkan
docker info | grep "Swarm"
```

---

## Step 1: Inisialisasi Docker Swarm

```bash
# Jika ini adalah single-node Swarm (server sandbox):
docker swarm init --advertise-addr 172.21.3.99

# Output akan menampilkan token untuk menambah worker node (simpan jika perlu)
# Contoh output:
# To add a worker to this swarm, run the following command:
#     docker swarm join --token SWMTKN-1-xxxxx 172.21.3.99:2377
```

---

## Step 2: Siapkan File di Server

```bash
# Copy semua file dari mesin lokal ke server
# Dari mesin lokal Windows:
scp -r C:\Users\Christopher Evan\Desktop\BINUS\ZammadTest user@172.21.3.99:~/zammad

# Atau gunakan git jika project sudah di repo:
git clone <repo-url> ~/zammad

# Masuk ke direktori
cd ~/zammad
```

---

## Step 3: Buat Sertifikat SSL Self-Signed

> **Catatan**: Karena menggunakan IP address (bukan domain), gunakan opsi `subjectAltName` agar browser menerimanya.

```bash
# Buat direktori SSL
mkdir -p nginx/ssl

# Generate sertifikat self-signed untuk IP 172.21.3.99
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/ssl/zammad.key \
  -out nginx/ssl/zammad.crt \
  -subj "/C=ID/ST=Jakarta/L=Jakarta/O=BINUS/CN=172.21.3.99" \
  -addext "subjectAltName=IP:172.21.3.99"

# Verifikasi sertifikat
openssl x509 -in nginx/ssl/zammad.crt -noout -text | grep -E "Subject:|IP Address:"
```

---

## Step 4: Konfigurasi File `.env`

```bash
# Edit file .env dan sesuaikan dengan kebutuhan
nano .env

# Pastikan nilai berikut sudah benar:
# ZAMMAD_FQDN=172.21.3.99
# RAILS_TRUSTED_PROXIES=127.0.0.1,::1
# (sisanya sudah sesuai default)
```

---

## Step 5: Deploy Stack

```bash
# Export environment variables dari .env ke shell session
export $(cat .env | grep -v '^#' | grep -v '^$' | xargs)

# Deploy Zammad stack ke Swarm
docker stack deploy -c docker-stack.yml zammad

# Output yang diharapkan:
# Creating network portainer-net
# Creating network zammad-backend
# Creating network zammad-frontend
# Creating network zammad-proxy
# Creating service zammad_zammad-postgresql
# Creating service zammad_zammad-elasticsearch
# ...
# Creating service zammad_portainer
# Creating service zammad_portainer-agent
```

---

## Step 6: Monitor Status Service

```bash
# Cek semua service Swarm
docker service ls

# Contoh output yang diharapkan (semua REPLICAS harus 1/1 atau n/n):
# ID             NAME                          MODE         REPLICAS
# abc123         zammad_zammad-postgresql      replicated   1/1
# def456         zammad_zammad-elasticsearch   replicated   1/1
# ghi789         zammad_zammad-redis           replicated   1/1
# ...
# xyz123         zammad_portainer              replicated   1/1
# uvw456         zammad_portainer-agent        global       1/1

# Cek log service tertentu
docker service logs -f zammad_zammad-init
docker service logs -f zammad_zammad-railsserver

# Cek status task (container) per service
docker service ps zammad_zammad-railsserver
```

> **Perhatian**: `zammad-init` akan menampilkan status `Shutdown` setelah selesai — ini **normal** karena init container berjalan sekali lalu exit.

---

## Step 7: Akses Portainer

Buka browser dan akses:

| URL | Keterangan |
|-----|------------|
| `http://172.21.3.99:9000` | Portainer GUI (HTTP) |
| `https://172.21.3.99:9443` | Portainer GUI (HTTPS dengan cert bawaan Portainer) |

**Setup awal Portainer:**
1. Buka `http://172.21.3.99:9000`
2. Buat akun admin (username + password baru)
3. Pilih **"Get Started"** → akan otomatis mendeteksi environment Swarm
4. Anda bisa melihat semua service, stacks, volumes, dan networks lewat GUI

---

## Step 8: Akses Zammad

| URL | Keterangan |
|-----|------------|
| `https://172.21.3.99` | Zammad (via Traefik, HTTPS) |
| `http://172.21.3.99` | Redirect otomatis ke HTTPS |
| `http://172.21.3.99:8088` | Traefik Dashboard |

> **Catatan Browser**: Karena sertifikat self-signed, browser akan menampilkan peringatan "Not Secure". Klik **Advanced → Proceed** untuk melanjutkan.

---

## Troubleshooting Umum

### Service tidak mau naik (REPLICAS 0/1)
```bash
# Lihat detail error
docker service ps --no-trunc zammad_<nama-service>

# Lihat log
docker service logs zammad_<nama-service>
```

### Elasticsearch gagal start (vm.max_map_count)
```bash
# Di server host, jalankan:
sudo sysctl -w vm.max_map_count=262144

# Agar permanen, tambahkan ke /etc/sysctl.conf:
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### Portainer tidak bisa connect ke agent
```bash
# Pastikan network portainer-net terbuat
docker network ls | grep portainer

# Restart portainer service
docker service update --force zammad_portainer
```

### Update stack setelah perubahan config
```bash
# Re-export env dan re-deploy (Swarm hanya mengupdate yang berubah)
export $(cat .env | grep -v '^#' | grep -v '^$' | xargs)
docker stack deploy -c docker-stack.yml zammad
```

### Hapus stack sepenuhnya
```bash
docker stack rm zammad

# Hapus volumes (HATI-HATI: data akan hilang!)
docker volume rm zammad_postgresql-data zammad_elasticsearch-data \
  zammad_redis-data zammad_zammad-storage zammad_portainer-data
```

---

## Ringkasan Port yang Digunakan

| Port | Service | Keterangan |
|------|---------|------------|
| `80` | Traefik | HTTP → redirect ke HTTPS |
| `443` | Traefik | HTTPS → Zammad |
| `8088` | Traefik | Dashboard Traefik |
| `9000` | Portainer | GUI Portainer (HTTP) |
| `9443` | Portainer | GUI Portainer (HTTPS) |
| `2377` | Docker Swarm | Port komunikasi Swarm manager |
