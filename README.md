# MODUL 2 TKA 2026

## Soal 1 — Sistem Energi Arkhe (Neuvillette)

Neuvillette menugaskanmu untuk mengelola sistem penyimpanan energi Arkhe (Ousia dan Pneuma) agar tetap stabil. Sistem ini terdiri dari sebuah Energy Collector (Backend) untuk mencatat daya, sebuah Arkhe Core (Redis) untuk menyimpan data energi secara cepat, dan sebuah Monitoring Terminal (Nginx). Karena energi Arkhe sangat tidak stabil, kamu harus membatasi penggunaan sumber daya agar sistem tidak meledak. Susunlah Docker Compose dengan ketentuan berikut:

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
Contohnya, backend bisa mengakses Redis menggunakan hostname ‘arkhe-core’.”

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

Energy Node (Backend)
- Melakukan build dari direktori lokal.
- Hanya boleh berjalan jika arkhe-core sudah dalam kondisi healthy.
- Gunakan variabel lingkungan (environment variables) dari file .env untuk mengatur koneksi ke database.
- Resource Management:Batasi penggunaan memori maksimal 256MB dan CPU 0.5.

Melakukan build dari direktori lokal
Hanya boleh berjalan jika arkhe-core sudah healthy
Gunakan environment variables dari .env
Batasi resource CPU dan memori

💡 Penjelasan

🔸 Build dari lokal

sourcecode lokal: folder energy-node, kita bikin file Dockerfile, dan dipanggil di docker-compose.yml 
```
energy-node:
  build: ./energy-node
```

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

Penjelasan detail

🔸 build: ./energy-node

→ Docker akan baca Dockerfile di folder itu

🔸 env_file

→ ambil variabel dari .env

🔸 depends_on + condition

→ backend tidak start sebelum Redis healthy

🔸 deploy.resources

→ batas:

CPU max 0.5 core
RAM max 256MB

⚙️ Implementasi

📄 docker-compose.yml
```bash
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
```
📄 .env
```bash
REDIS_HOST=arkhe-core
📄 Backend (Flask)
r = redis.Redis(
    host=os.getenv("REDIS_HOST"),
    port=6379
)
```
### 1.c — Arkhe Core (Redis)
📌 Soal

Arkhe Core (Database Redis)
- Menggunakan image redis:alpine.
- Persistensi: Pastikan data energi tidak hilang saat sistem dimatikan
- Gunakan fitur healthcheck untuk memastikan layanan siap menerima koneksi.

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
```bash
arkhe-core:
  image: redis:alpine
  volumes: # mapping: /data di container ke volume Docker,  Redis otomatis simpan data di situ
    - redis_data:/data
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 5s
    retries: 5

📄 Volume
```bash
volumes:
  redis_data:
```

### 1.d — Monitoring Terminal (Nginx)

📌 Soal

Monitoring Terminal (Nginx)
- Berfungsi sebagai reverse proxy yang meneruskan trafik ke energy-node.
- Hubungkan file konfigurasi Nginx dari host menggunakan Bind Mount.

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
```bash
monitoring-terminal:
  image: nginx:latest
  ports:
    - "7070:80"
  volumes:
    - ./nginx/default.conf:/etc/nginx/conf.d/default.conf # config dari laptop langsung dipakai
  depends_on:
    - energy-node
```

📄 nginx/default.conf

```bash
server {
    listen 80;

    location / {
        proxy_pass http://energy-node:5000; # kirim request ke backend
    }

    location /energy {
        proxy_pass http://energy-node:5000/energy;
    }
}
```

🎯 Hasil Akhir

Setelah semua service dijalankan:
```bash
docker compose up -d --build
```
Aplikasi dapat diakses melalui:

```http://localhost:7070```
 → halaman utama
```http://localhost:7070/energy```
 → menampilkan data energi
💾 Uji Persistence

```bash
docker compose down
docker compose up -d
```
➡️ Data tetap tersimpan karena menggunakan Docker Volume

### Penjelasan untuk tiap file:

docker-compose.yml: Mengatur dan menjalankan semua container dalam satu sistem.

```bash
services:  # mendefinisikan semua container (service)

  arkhe-core:  # service Redis
    image: redis:alpine  # pakai image Redis versi ringan
    container_name: arkhe-core  # nama container
    volumes:
      - redis_data:/data  # simpan data Redis ke volume (biar tidak hilang)
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]  # cek Redis dengan ping
      interval: 5s  # cek setiap 5 detik
      retries: 5  # coba maksimal 5 kali

  energy-node:  # service backend
    build: ./energy-node  # build dari folder lokal energy-node
    container_name: energy-node  # nama container
    env_file:
      - .env  # ambil environment variable dari file .env
    depends_on:
      arkhe-core:
        condition: service_healthy  # hanya jalan kalau Redis sudah healthy
    deploy:
      resources:
        limits:
          cpus: "0.5"  # batasi CPU max 0.5 core
          memory: 256M  # batasi RAM max 256MB

  monitoring-terminal:  # service Nginx
    image: nginx:latest  # pakai image Nginx terbaru
    container_name: monitoring-terminal  # nama container
    ports:
      - "7070:80"  # akses dari localhost:7070 ke port 80 container
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf  # bind mount config nginx
    depends_on:
      - energy-node  # jalankan setelah backend

volumes:
  redis_data:  # deklarasi volume untuk Redis

```

📄 .env : Menyimpan konfigurasi (environment variable).

```bash
REDIS_HOST=arkhe-core  # hostname Redis (nama service di docker compose)
```

📄 energy-node/app.py : Backend aplikasi (API).

```bash
from flask import Flask  # import Flask untuk web server
import redis  # import Redis client
import os  # import untuk ambil environment variable

app = Flask(__name__)  # inisialisasi aplikasi Flask

# koneksi ke Redis
r = redis.Redis(
    host=os.getenv("REDIS_HOST"),  # ambil host dari .env
    port=6379,  # port default Redis
    decode_responses=True  # biar output string (bukan bytes)
)

@app.route("/")  # endpoint root
def home():
    return "Energy System Running"  # respon sederhana

@app.route("/energy")  # endpoint energy
def energy():
    value = r.incr("energy")  # increment nilai energy di Redis
    return f"Energy: {value}"  # tampilkan hasil

app.run(host="0.0.0.0", port=5000)  # jalankan server di port 5000
```

📄 energy-node/requirements.txt : Daftar library Python yang dibutuhkan.

```
flask  # library web framework
redis  # library untuk koneksi Redis
```

📄 energy-node/Dockerfile : Blueprint untuk membuat image backend.

```bash
FROM python:3.10  # base image Python

WORKDIR /app  # set working directory di container

COPY . .  # copy semua file ke dalam container

RUN pip install -r requirements.txt  # install dependency

CMD ["python", "app.py"]  # jalankan aplikasi saat container start
```

📄 nginx/default.conf : Konfigurasi Nginx sebagai reverse proxy.

```bash
server {
    listen 80;  # nginx listen di port 80

    location / {
        proxy_pass http://energy-node:5000;  # forward ke backend
    }

    location /energy {
        proxy_pass http://energy-node:5000/energy;  # forward endpoint energy
    }
}
```
