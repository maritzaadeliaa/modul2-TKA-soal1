# MODUL 2 TKA 2026

## Soal 1 — Sistem Energi Arkhe (Neuvillette)

Pada soal ini, kita diminta untuk membangun sistem berbasis Docker Compose yang terdiri dari:

Energy Node (Backend)
Arkhe Core (Redis)
Monitoring Terminal (Nginx)

Tujuan utama:

Menghubungkan ketiga service dalam satu sistem
Menjamin stabilitas dengan healthcheck & dependency
Menyimpan data secara persisten
Membatasi resource penggunaan container

### 1.a — Network & Service
📌 Soal

Terdapat 3 service: energy-node, arkhe-core, dan monitoring-terminal yang berada dalam satu network yang sama.

💡 Penjelasan

Semua container harus bisa saling berkomunikasi.
Docker Compose secara default sudah menyediakan 1 network internal, jadi kita cukup mendefinisikan semua service dalam satu file.

⚙️ Implementasi

Kita mendefinisikan 3 service dalam docker-compose.yml:

services:
  arkhe-core:
    image: redis:alpine

  energy-node:
    build: ./energy-node

  monitoring-terminal:
    image: nginx:latest

✔️ Dengan ini, semua service otomatis berada dalam network yang sama

### 1.b — Energy Node (Backend)
📌 Soal
Melakukan build dari direktori lokal
Hanya boleh berjalan jika arkhe-core sudah healthy
Gunakan environment variables dari .env
Batasi resource CPU dan memori
💡 Penjelasan
🔸 Build dari lokal

Backend tidak pakai image jadi, tapi dibuat dari source code sendiri (Flask).

🔸 Dependency (depends_on)

Backend tidak boleh jalan sebelum Redis siap → pakai:

depends_on
healthcheck
🔸 Environment Variable

Agar fleksibel, koneksi Redis tidak hardcode.

🔸 Resource Limit

Untuk mencegah sistem overload:

CPU max = 0.5
Memory max = 256MB
⚙️ Implementasi
📄 docker-compose.yml
energy-node:
  build: ./energy-node
  env_file:
    - .env
  depends_on:
    arkhe-core:
      condition: service_healthy
  deploy:
    resources:
      limits:
        cpus: "0.5"
        memory: 256M
📄 .env
REDIS_HOST=arkhe-core
📄 Backend (Flask)
r = redis.Redis(
    host=os.getenv("REDIS_HOST"),
    port=6379
)
### 1.c — Arkhe Core (Redis)
📌 Soal
Menggunakan image redis:alpine
Data harus persisten
Gunakan healthcheck
💡 Penjelasan
🔸 Redis sebagai database

Digunakan untuk menyimpan data energi.

🔸 Persistence

Agar data tidak hilang saat container dimatikan → gunakan volume

🔸 Healthcheck

Untuk memastikan Redis siap sebelum backend berjalan.

⚙️ Implementasi
arkhe-core:
  image: redis:alpine
  volumes:
    - redis_data:/data
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 5s
    retries: 5
📄 Volume
volumes:
  redis_data:
### 1.d — Monitoring Terminal (Nginx)
📌 Soal
Berfungsi sebagai reverse proxy
Menggunakan bind mount untuk konfigurasi
Sistem dapat diakses melalui http://localhost:7070
💡 Penjelasan
🔸 Reverse Proxy

Nginx bertugas meneruskan request dari user ke backend.

🔸 Bind Mount

Konfigurasi Nginx diambil dari file lokal → memudahkan edit tanpa rebuild image.

⚙️ Implementasi
📄 docker-compose.yml
monitoring-terminal:
  image: nginx:latest
  ports:
    - "7070:80"
  volumes:
    - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
  depends_on:
    - energy-node
📄 nginx/default.conf
server {
    listen 80;

    location / {
        proxy_pass http://energy-node:5000;
    }

    location /energy {
        proxy_pass http://energy-node:5000/energy;
    }
}
🎯 Hasil Akhir

Setelah semua service dijalankan:

docker compose up --build

Aplikasi dapat diakses melalui:

http://localhost:7070
 → halaman utama
http://localhost:7070/energy
 → menampilkan data energi
💾 Uji Persistence
docker compose down
docker compose up

➡️ Data tetap tersimpan karena menggunakan Docker Volume
