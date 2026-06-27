# Laporan Final Project TKA

## Anggota Kelompok

| Nama | NRP |
|--------|--------|
| Adiwidya Budi Pratama | 5027241012 |
| Erlangga Valdhio Putra Sulistio | 5027241030 |
| Jonathan Zelig Sutopo | 5027241047 |
| Raihan Fahri Ghazali | 5027241061 |
| Naila Cahyarani Idelia | 5027241063 |
| Fika Arka Nuriyah | 5027241071 |
| Muhammad Ahsani Taqwiim Rakhman | 5027241099 |
| Imam Mahmud Dalil Fauzan | 5027241100 |

---

# 1. Introduction

## Latar Belakang

Perkembangan platform e-commerce menuntut sistem yang mampu menangani lonjakan trafik secara cepat, stabil, dan efisien. Salah satu layanan inti dalam platform e-commerce adalah Order Processing Service yang bertanggung jawab untuk membuat pesanan, menyimpan data transaksi, memperbarui status pesanan, dan menampilkan riwayat transaksi.

Pada proyek ini dibangun sebuah sistem Order Processing Service berbasis Flask dan MongoDB yang diimplementasikan pada Microsoft Azure menggunakan arsitektur multi-VM. Untuk meningkatkan performa dan ketersediaan layanan, digunakan Nginx sebagai Load Balancer yang mendistribusikan request ke dua backend server.

## Tujuan

- Membangun sistem Order Processing Service berbasis cloud.
- Mengimplementasikan load balancing menggunakan Nginx dengan strategi `least_conn`.
- Menghubungkan backend Flask dengan MongoDB.
- Mengoptimalkan performa dengan Gunicorn gthread worker, connection pooling, dan Nginx keepalive.
- Melakukan pengujian performa menggunakan Locust.
- Menganalisis kemampuan sistem dalam menangani beban tinggi.

---

# 2. Arsitektur Cloud

## Diagram Arsitektur

![Diagram Arsitektur](assets/diagram.png)

## Topologi Sistem

| VM | Fungsi | IP Publik |
|-----|---------|-----------|
| VM 1 | Nginx Load Balancer + Frontend | `40.81.25.98` |
| VM 2 | Backend API 1 (Flask + Gunicorn) | `20.40.54.202` |
| VM 3 | Backend API 2 (Flask + Gunicorn) | `20.40.58.217` |
| VM 4 | MongoDB Database | `4.240.92.222` |

## Alur Sistem

1. User mengakses aplikasi melalui VM 1 (port 80).
2. Nginx menerima request dan mendistribusikannya ke VM 2 atau VM 3 menggunakan strategi `least_conn` dengan keepalive connection.
3. Backend Flask memproses request menggunakan Gunicorn gthread worker.
4. Backend terhubung ke MongoDB di VM 4 melalui connection pool (`maxPoolSize=400`).
5. Response dikirim kembali ke pengguna.

## Spesifikasi VM

| VM | Role | OS | vCPU | RAM | Harga/Bulan |
|-----|------|------|------|------|-------------|
| VM 1 | Load Balancer + Frontend | Ubuntu Server | 2 | 4 GB | $17.96 |
| VM 2 | Backend API 1 | Ubuntu Server | 2 | 4 GB | $17.96 |
| VM 3 | Backend API 2 | Ubuntu Server | 2 | 4 GB | $17.96 |
| VM 4 | MongoDB Database | Ubuntu Server | 2 | 4 GB | $17.96 |
| **Total** | | | | | **$71.84** |

Budget yang digunakan $71.84/bulan dari maksimal $75/bulan, sisa $3.16.

## Teknologi yang Digunakan

| Teknologi | Fungsi |
|-----------|---------|
| Nginx | Load Balancer dan Web Server Frontend |
| Flask | Framework Backend REST API |
| Gunicorn (gthread) | Application Server multi-worker multi-thread |
| MongoDB | Database NoSQL |
| Ubuntu Server | Sistem Operasi Virtual Machine |
| Microsoft Azure | Infrastruktur Cloud |
| Locust | Load Testing |

---

# 3. Implementasi

## 3.1 Konfigurasi Network Security Group (NSG)

Setiap VM dikonfigurasi NSG-nya agar hanya port yang diperlukan yang terbuka:

| VM | Port | Source | Keterangan |
|----|------|--------|------------|
| VM 1 | 80 | Any | HTTP publik |
| VM 1 | 22 | Admin IP | SSH |
| VM 2 | 5000 | `40.81.25.98` (VM1) | Flask backend |
| VM 2 | 22 | Admin IP | SSH |
| VM 3 | 5000 | `40.81.25.98` (VM1) | Flask backend |
| VM 3 | 22 | Admin IP | SSH |
| VM 4 | 27017 | `20.40.54.202`, `20.40.58.217` | MongoDB |
| VM 4 | 22 | Admin IP | SSH |

---

## 3.2 Deploy MongoDB (VM 4)

### Instalasi MongoDB

```bash
sudo apt update && sudo apt install -y mongodb
sudo systemctl enable mongod
sudo systemctl start mongod
```

### Konfigurasi `/etc/mongod.conf`

```yaml
storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 0.0.0.0
```

### Restore Dump Data

```bash
mongorestore --uri='mongodb://127.0.0.1:27017' --drop dump/
```

### Pembuatan Index

```javascript
use orderdb
db.orders.createIndex({ created_at: -1 })
db.orders.createIndex({ order_id: 1 })
```

---

## 3.3 Deploy Backend API (VM 2 & VM 3)

### Clone Source Code & Install Dependency

```bash
git clone https://github.com/imdfauzan/TKA-B2-FP.git
cd TKA-B2-FP
python3 -m venv .venv
source .venv/bin/activate
pip install -r Resources/BE/requirements.txt
```

### Konfigurasi Systemd Service (`/etc/systemd/system/gunicorn.service`)

```ini
[Unit]
Description=Gunicorn instance serving TKA-B2-FP Order Processing API
After=network.target

[Service]
User=azureuser
Group=www-data
WorkingDirectory=/home/azureuser/
Environment="MONGO_URI=mongodb://4.240.92.222:27017/orderdb?maxPoolSize=400"
Environment="JWT_SECRET=TK4_FPdul5"
ExecStart=/home/azureuser/.venv/bin/gunicorn -w 5 --threads 10 --worker-class gthread -b 0.0.0.0:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

**Penjelasan optimasi Gunicorn:**
- `-w 5` — 5 worker processes untuk memanfaatkan multi-core
- `--threads 10` — 10 threads per worker = 50 concurrent request handler
- `--worker-class gthread` — async threading, lebih efisien dari sync worker
- `maxPoolSize=400` — MongoDB connection pool besar untuk high concurrency

```bash
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
```

---

## 3.4 Konfigurasi Nginx Load Balancer (VM 1)

### Instalasi Nginx

```bash
sudo apt install nginx -y
```

### Konfigurasi `/etc/nginx/sites-available/loadbalancer`

```nginx
upstream backend_servers {
    least_conn;
    server 20.40.54.202:5000;
    server 20.40.58.217:5000;
    keepalive 64;
}

server {
    listen 80;
    server_name 40.81.25.98;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ ^/(auth|orders|products|admin|health) {
        proxy_pass http://backend_servers;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**Penjelasan optimasi Nginx:**
- `keepalive 64` — persistent connection ke backend, mengurangi overhead TCP handshake
- `proxy_http_version 1.1` + `proxy_set_header Connection ""` — wajib untuk keepalive
- `location ~ ^/(auth|orders|...)` — regex match mencakup semua sub-path sekaligus

```bash
sudo ln -s /etc/nginx/sites-available/loadbalancer /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

![Nginx Config Test](assets/nginx.png)

---

## 3.5 Deploy Frontend (VM 1)

```bash
sudo cp ~/TKA-B2-FP/resources/FE/index.html /var/www/html/
sudo cp ~/TKA-B2-FP/resources/FE/styles.css /var/www/html/
```

Akses frontend melalui: `http://40.81.25.98`

![Tampilan Frontend](assets/frontend.png)

![Tampilan Frontend 2](assets/frontend2.png)

![Tampilan Frontend View](assets/frontend-view.png)

---

# 4. Hasil Pengujian Endpoint

Kredensial yang digunakan:
- Admin: `admin1@tka.its.ac.id` / `Admin@12345`
- User: `kalimprakasa@example.org` / `User@12345`

## GET /health

```bash
curl http://40.81.25.98/health
```
```json
{"status":"ok","timestamp":"2026-06-23T18:32:25.393981+00:00"}
```

![Endpoint Health](assets/endpoint1.png)

---

## POST /auth/login

```bash
curl -X POST http://40.81.25.98/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin1@tka.its.ac.id","password":"Admin@12345"}'
```
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {"email":"admin1@tka.its.ac.id","name":"Admin TKA 1","role":"admin"}
}
```

![Endpoint Login](assets/endpoint2.png)

---

## GET /products

```bash
curl http://40.81.25.98/products -H "Authorization: Bearer $TOKEN"
```

Response: 92 produk dengan paginasi 20 per halaman.

![Endpoint Products](assets/endpoint3.png)

---

## POST /orders

```bash
curl -X POST http://40.81.25.98/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"product_id":"6a2f5aa3d7c8a947fb1afc8f","qty":1}]}'
```
```json
{
  "order_id": "877096a2-3b86-4b08-9efc-1c3b4a50c122",
  "status": "pending",
  "total": 1350000,
  "created_at": "2026-06-24T19:27:17.387970+00:00"
}
```

![Endpoint Create Order](assets/endpoint4.png)

---

## GET /orders

```bash
curl http://40.81.25.98/orders -H "Authorization: Bearer $TOKEN"
```

![Endpoint Get Orders](assets/endpoint5.png)

---

## GET /orders/\<order_id\>

```bash
curl http://40.81.25.98/orders/877096a2-3b86-4b08-9efc-1c3b4a50c122 \
  -H "Authorization: Bearer $TOKEN"
```

![Endpoint Get Order Detail](assets/endpoint6.png)

---

## PUT /orders/\<order_id\>/status

```bash
curl -X PUT http://40.81.25.98/orders/877096a2-3b86-4b08-9efc-1c3b4a50c122/status \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status":"processing"}'
```
```json
{"order_id":"877096a2-3b86-4b08-9efc-1c3b4a50c122","status":"processing"}
```

![Endpoint Update Status](assets/endpoint7.png)

---

# 5. Hasil Load Testing

Pengujian dilakukan menggunakan Locust dari perangkat lokal (berbeda dari server), host: `http://40.81.25.98`.

**Kondisi sistem saat pengujian:**
- VM 1: Nginx Load Balancer + keepalive 64
- VM 2 & VM 3: Flask + Gunicorn `-w 5 --threads 10 --worker-class gthread`
- VM 4: MongoDB dengan index `created_at` dan `order_id`, `maxPoolSize=400`

---

## Skenario 1 – Maximum RPS (0% Failure)

**Parameter:** Users ditingkatkan bertahap, spawn rate 5, durasi 60 detik.

| Metrik | Nilai |
|---------|---------|
| Users | 50 |
| Spawn Rate | 5 (bertahap) |
| RPS Maksimum | ~225 RPS |
| Failure Rate | 0% ✅ |
| 50th Percentile Response Time | ~170 ms |
| 95th Percentile Response Time | ~180 ms |

![Locust Skenario 1](assets/loc-sk1.png)

### Resource Utilization Skenario 1

| VM | CPU Usage | Load Average | Memory |
|----|-----------|-------------|--------|
| VM 1 (Nginx) | ~0.7% | 0.00 | 277M/843M |
| VM 2 (Backend) | ~22.6% | 0.76 | 484M/3.82G |
| VM 3 (Backend) | ~2.0% | 0.01 | 483M/3.82G |
| VM 4 (MongoDB) | — | — | — |

![htop VM1 Skenario 1](assets/vm1-sk1.png)
![htop VM2 Skenario 1](assets/vm2-sk1.png)
![htop VM3 Skenario 1](assets/vm3-sk1.png)
![htop VM4 Skenario 1](assets/vm4-sk1.png)

---

## Skenario 2 – Peak Concurrency (Spawn Rate 50)

**Parameter:** 100 users, spawn rate 50, durasi 60 detik.

| Metrik | Nilai |
|---------|---------|
| Concurrent Users | 100 |
| Spawn Rate | 50 |
| RPS Peak | ~470 RPS |
| Failure Rate | 0% ✅ |
| 50th Percentile Response Time | ~170 ms |
| 95th Percentile Response Time | ~180 ms |

![Locust Skenario 2](assets/loc-sk2.png)

### Resource Utilization Skenario 2

| VM | CPU Usage | Load Average | Memory |
|----|-----------|-------------|--------|
| VM 1 (Nginx) | ~1.4% | 0.00 | 278M/843M |
| VM 2 (Backend) | ~2.0% | 0.01 | 483M/3.82G |
| VM 3 (Backend) | ~23.0% | 0.25 | 483M/3.82G |
| VM 4 (MongoDB) | — | — | — |

![htop VM1 Skenario 2](assets/vm1-sk2.png)
![htop VM2 Skenario 2](assets/vm2-sk2.png)
![htop VM3 Skenario 2](assets/vm3-sk2.png)
![htop VM4 Skenario 2](assets/vm4-sk2.png)

---

## Skenario 3 – Peak Concurrency (Spawn Rate 100)

**Parameter:** 100 users, spawn rate 100, durasi 60 detik.

| Metrik | Nilai |
|---------|---------|
| Concurrent Users | 100 |
| Spawn Rate | 100 |
| RPS Peak | ~460 RPS |
| Failure Rate | 0% ✅ |
| 50th Percentile Response Time | ~170 ms |
| 95th Percentile Response Time | ~180 ms |

![Locust Skenario 3](assets/loc-sk3.png)

### Resource Utilization Skenario 3

| VM | CPU Usage | Load Average | Memory |
|----|-----------|-------------|--------|
| VM 1 (Nginx) | ~3.4% | 0.00 | 278M/843M |
| VM 2 (Backend) | ~23.0% | 0.25 | 483M/3.82G |
| VM 3 (Backend) | ~13.7% | 0.79 | 483M/3.82G |
| VM 4 (MongoDB) | — | — | — |

![htop VM1 Skenario 3](assets/vm1-sk3.png)
![htop VM2 Skenario 3](assets/vm2-sk3.png)
![htop VM3 Skenario 3](assets/vm3-sk3.png)
![htop VM4 Skenario 3](assets/vm4-sk3.png)

---

## Skenario 4 – Peak Concurrency (Spawn Rate 200)

**Parameter:** 100 users, spawn rate 200, durasi 60 detik.

| Metrik | Nilai |
|---------|---------|
| Concurrent Users | 100 |
| Spawn Rate | 200 |
| RPS Peak | ~460 RPS |
| Failure Rate | 0% ✅ |
| 50th Percentile Response Time | ~170 ms |
| 95th Percentile Response Time | ~180 ms |

![Locust Skenario 4](assets/loc-sk4.png)

### Resource Utilization Skenario 4

| VM | CPU Usage | Load Average | Memory |
|----|-----------|-------------|--------|
| VM 1 (Nginx) | ~2.0% | 0.00 | 278M/843M |
| VM 2 (Backend) | ~13.7% | 0.79 | 483M/3.82G |
| VM 3 (Backend) | ~0.7% | 0.52 | 483M/3.82G |
| VM 4 (MongoDB) | — | — | — |

![htop VM1 Skenario 4](assets/vm1-sk4.png)
![htop VM2 Skenario 4](assets/vm2-sk4.png)
![htop VM3 Skenario 4](assets/vm3-sk4.png)
![htop VM4 Skenario 4](assets/vm4-sk4.png)

---

## Skenario 5 – Peak Concurrency (Spawn Rate 500)

**Parameter:** 100 users, spawn rate 500, durasi 60 detik.

| Metrik | Nilai |
|---------|---------|
| Concurrent Users | 100 |
| Spawn Rate | 500 |
| RPS Peak | ~472 RPS |
| Failure Rate | 0% ✅ |
| 50th Percentile Response Time | ~170 ms |
| 95th Percentile Response Time | ~180 ms |

![Locust Skenario 5](assets/loc-sk5.png)

### Resource Utilization Skenario 5

| VM | CPU Usage | Load Average | Memory |
|----|-----------|-------------|--------|
| VM 1 (Nginx) | ~0.7% | 0.06 | 278M/843M |
| VM 2 (Backend) | ~0.7% | 0.52 | 483M/3.82G |
| VM 3 (Backend) | ~0.7% | 0.52 | 483M/3.82G |
| VM 4 (MongoDB) | — | — | — |

![htop VM1 Skenario 5](assets/vm1-sk5.png)
![htop VM2 Skenario 5](assets/vm2-sk5.png)
![htop VM3 Skenario 5](assets/vm3-sk5.png)
![htop VM4 Skenario 5](assets/vm4-sk5.png)

---

## Ringkasan Hasil Load Testing

| Skenario | Users | Spawn Rate | RPS Peak | Failure Rate | P50 | P95 |
|----------|-------|------------|----------|--------------|-----|-----|
| 1 – Max RPS | 50 | 5 | **~225 RPS** | 0% ✅ | 170 ms | 180 ms |
| 2 – Peak SR 50 | 100 | 50 | ~470 RPS | 0% ✅ | 170 ms | 180 ms |
| 3 – Peak SR 100 | 100 | 100 | ~460 RPS | 0% ✅ | 170 ms | 180 ms |
| 4 – Peak SR 200 | 100 | 200 | ~460 RPS | 0% ✅ | 170 ms | 180 ms |
| 5 – Peak SR 500 | 100 | 500 | ~472 RPS | 0% ✅ | 170 ms | 180 ms |

**Nilai RPS (Skenario 1):** (225/200) × 30 = **33.75 poin** — melebihi standar 200 RPS ✅

---

# 6. Analisis

Berdasarkan hasil pengujian:

- **Optimasi Gunicorn gthread** memberikan peningkatan signifikan — dari ~74 RPS (sync worker `-w 3`) menjadi ~225–470 RPS setelah menggunakan `-w 5 --threads 10 --worker-class gthread`. Kombinasi 5 worker × 10 threads menghasilkan 50 concurrent handler yang jauh lebih efisien.

- **Nginx keepalive** (`keepalive 64` + `proxy_http_version 1.1`) mengurangi overhead TCP handshake antar Nginx dan backend, sehingga latency berkurang signifikan — response time stabil di 170–180ms di semua skenario.

- **MongoDB connection pooling** (`maxPoolSize=400`) memastikan backend tidak bottleneck pada koneksi database saat concurrency tinggi.

- **CPU VM2 (Backend)** berada di rentang 0.7%–23% tergantung skenario — tidak ada tanda-tanda resource exhaustion, menunjukkan masih ada headroom untuk scale lebih lanjut.

- **VM1 (Nginx)** konsisten di bawah 3.4% CPU dengan memory stabil ~278MB, membuktikan Nginx efisien sebagai load balancer.

- **Response time sangat konsisten** di semua skenario (P50: 170ms, P95: 180ms) — menunjukkan sistem tidak degradasi meski spawn rate naik dari 50 hingga 500.

---

# 7. Kesimpulan dan Saran

## Kesimpulan

- Sistem Order Processing Service berhasil diimplementasikan pada Microsoft Azure dengan total biaya **$71.84/bulan**, di bawah budget $75.
- Nginx dengan `least_conn` + `keepalive 64` berhasil mendistribusikan trafik ke dua backend secara efisien.
- Optimasi Gunicorn (`gthread`, 5 workers, 10 threads) meningkatkan throughput dari ~74 RPS menjadi **~225 RPS** (Skenario 1) dan **~470 RPS** (Skenario 2–5).
- Sistem mampu melayani **100 concurrent users dengan 0% failure** di semua skenario, melampaui standar 200 RPS.
- Response time sangat stabil di **170–180ms** (P50–P95) di semua skenario.

## Saran

- Menambahkan **autoscaling** Azure VM Scale Set untuk menghadapi lonjakan traffic tak terduga.
- Mengimplementasikan **Redis caching** untuk endpoint `/products` dan `/admin/stats` yang bersifat read-heavy.
- Menggunakan **Azure Monitor + Grafana** untuk monitoring real-time yang lebih komprehensif.
- Menambahkan **HTTPS** dengan Let's Encrypt untuk keamanan komunikasi.
- Mempertimbangkan **MongoDB replica set** untuk meningkatkan availability database.
- Menambah jumlah Gunicorn workers sesuai jumlah CPU core tersedia untuk memaksimalkan throughput.

---

# Lampiran

## Struktur Repository

```text
cloud-load-balancing-optimization/
├── README.md
├── assets/
│   ├── diagram.png
│   ├── nginx.png
│   ├── frontend.png
│   ├── frontend2.png
│   ├── frontend-view.png
│   ├── endpoint1.png ~ endpoint7.png
│   ├── loc-sk1.png ~ loc-sk5.png
│   ├── vm1-sk1.png ~ vm1-sk5.png
│   ├── vm2-sk1.png ~ vm2-sk5.png
│   ├── vm3-sk1.png ~ vm3-sk5.png
│   └── vm4-sk1.png ~ vm4-sk5.png
├── resources/
│   ├── BE/
│   │   ├── app.py
│   │   └── requirements.txt
│   ├── DB/
│   │   ├── generate_dump.py
│   │   └── dump/orderdb/
│   ├── FE/
│   │   ├── index.html
│   │   └── styles.css
│   └── Test/
│       └── locustfile.py
└── config/
    ├── nginx-loadbalancer.conf
    ├── gunicorn-vm2.service
    ├── gunicorn-vm3.service
    └── mongodb.conf
```
