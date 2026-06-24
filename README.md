# Laporan Final Project TKA

## Anggota Kelompok

| Nama                            | NRP        |
| ------------------------------- | ---------- |
| Adiwidya Budi Pratama           | 5027241012 |
| Erlangga Valdhio Putra Sulistio | 5027241030 |
| Jonathan Zelig Sutopo           | 5027241047 |
| Raihan Fahri Ghazali            | 5027241061 |
| Naila Cahyarani Idelia          | 5027241063 |
| Fika Arka Nuriyah               | 5027241071 |
| Muhammad Ahsani Taqwiim Rakhman | 5027241099 |
| Imam Mahmud Dalil Fauzan        | 5027241100 |

---

# 1. Introduction

## Latar Belakang

Perkembangan platform e-commerce menuntut sistem yang mampu menangani lonjakan trafik secara cepat, stabil, dan efisien. Salah satu layanan inti dalam platform e-commerce adalah Order Processing Service yang bertanggung jawab untuk membuat pesanan, menyimpan data transaksi, memperbarui status pesanan, dan menampilkan riwayat transaksi.

Pada proyek ini dibangun sebuah sistem Order Processing Service berbasis Flask dan MongoDB yang diimplementasikan pada Microsoft Azure menggunakan arsitektur multi-VM. Untuk meningkatkan performa dan ketersediaan layanan, digunakan Nginx sebagai Load Balancer yang mendistribusikan request ke dua backend server.

## Tujuan

- Membangun sistem Order Processing Service berbasis cloud.
- Mengimplementasikan load balancing menggunakan Nginx.
- Menghubungkan backend Flask dengan MongoDB.
- Mengoptimalkan performa database menggunakan indexing.
- Melakukan pengujian performa menggunakan Locust.
- Menganalisis kemampuan sistem dalam menangani beban tinggi.

---

# 2. Arsitektur Cloud

## Diagram Arsitektur

![Diagram Arsitektur](./assets/diagram.png)

## Topologi Sistem

| VM                    | Fungsi                           |
| --------------------- | -------------------------------- |
| VM 1 (`40.81.25.98`)  | Nginx Load Balancer + Frontend   |
| VM 2 (`20.40.54.202`) | Backend API 1 (Flask + Gunicorn) |
| VM 3 (`20.40.58.217`) | Backend API 2 (Flask + Gunicorn) |
| VM 4 (`4.240.92.222`) | MongoDB Database                 |

## Alur Sistem

1. User mengakses aplikasi melalui VM 1.
2. Nginx menerima request dan mendistribusikannya ke Backend API 1 atau Backend API 2 menggunakan strategi `least_conn`.
3. Backend memproses request.
4. Backend melakukan komunikasi dengan MongoDB pada VM 4 port 27017.
5. Response dikirim kembali ke pengguna.

## Spesifikasi VM

| VM        | Role                     | OS            | vCPU | RAM  | Harga/Bulan |
| --------- | ------------------------ | ------------- | ---- | ---- | ----------- |
| VM 1      | Load Balancer + Frontend | Ubuntu Server | 2    | 4 GB | $17.96      |
| VM 2      | Backend API 1            | Ubuntu Server | 2    | 4 GB | $17.96      |
| VM 3      | Backend API 2            | Ubuntu Server | 2    | 4 GB | $17.96      |
| VM 4      | MongoDB Database         | Ubuntu Server | 2    | 4 GB | $17.96      |
| **Total** |                          |               |      |      | **$71.84**  |

## Teknologi yang Digunakan

| Teknologi       | Fungsi                                |
| --------------- | ------------------------------------- |
| Nginx           | Load Balancer dan Web Server Frontend |
| Flask           | Framework Backend REST API            |
| Gunicorn        | Application Server untuk Flask (-w 3) |
| MongoDB         | Database NoSQL                        |
| Ubuntu Server   | Sistem Operasi Virtual Machine        |
| Microsoft Azure | Infrastruktur Cloud                   |
| Locust          | Load Testing                          |
| Postman         | Pengujian Endpoint API                |

---

## Estimasi Biaya

| VM    | Biaya/Bulan |
| ----- | ----------- |
| VM 1  | $17,96      |
| VM 2  | $17,96      |
| VM 3  | $17,96      |
| VM 4  | $17,96      |
| Total | $71,84      |

---

# 3. Implementasi

## 3.1 Deploy MongoDB

### Instalasi MongoDB

```bash
sudo apt update
sudo apt install mongodb -y
```

### Konfigurasi MongoDB

MongoDB dikonfigurasi agar menerima koneksi dari Backend API 1 dan Backend API 2 melalui port 27017.



### Pembuatan Index

Untuk meningkatkan performa endpoint `/orders`, dibuat index pada field `created_at`.

```javascript
db.orders.createIndex({ created_at: -1 });
```

> Tambahkan screenshot hasil indexing

---

## 3.2 Deploy Backend API

### Clone Source Code

```bash
git clone <repository-url>
cd backend
```

### Install Dependency

```bash
pip install -r requirements.txt
```

### Menjalankan Gunicorn

```bash
gunicorn -w 3 -b 0.0.0.0:5000 app:app
```

> Tambahkan screenshot backend berjalan

---

## 3.3 Konfigurasi Firewall

Port 5000 hanya dibuka untuk IP publik VM 1 (Load Balancer).

> Tambahkan screenshot Network Security Group (NSG)

---

## 3.4 Konfigurasi Nginx Load Balancer

### Instalasi Nginx

```bash
sudo apt install nginx -y
```

### Konfigurasi Upstream

```nginx
upstream backend_servers {
    server <IP_VM2>:5000;
    server <IP_VM3>:5000;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend_servers;
    }
}
```

### Restart Nginx

```bash
sudo systemctl restart nginx
```

> Tambahkan screenshot konfigurasi Nginx

---

## 3.5 Deploy Frontend

File frontend (`index.html` dan `styles.css`) ditempatkan pada direktori:

```bash
/var/www/html/
```

Frontend dapat diakses melalui IP publik VM 1.

> Tambahkan screenshot frontend

---

# 4. Hasil Pengujian Endpoint

Pengujian endpoint dilakukan menggunakan `curl` dari lokal dan Postman.

Kredensial yang digunakan:

- Admin: `admin1@tka.its.ac.id` / `Admin@12345`
- User: `kalimprakasa@example.org` / `User@12345`

## GET /health

```bash
curl http://40.81.25.98/health
```

**Response:**

```json
{ "status": "ok", "timestamp": "2026-06-23T18:32:25.393981+00:00" }
```

>

---

## POST /auth/login

```bash
curl -X POST http://40.81.25.98/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin1@tka.its.ac.id","password":"Admin@12345"}'
```

**Response:**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "email": "admin1@tka.its.ac.id",
    "id": "6a2f5aa3d7c8a947fb1afce6",
    "name": "Admin TKA 1",
    "role": "admin"
  }
}
```

>

---

## GET /products

```bash
curl http://40.81.25.98/products \
  -H "Authorization: Bearer <TOKEN>"
```

**Response:** Mengembalikan 92 produk dengan paginasi (20 per halaman).

>

---

## POST /orders

```bash
curl -X POST http://40.81.25.98/orders \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"product_id":"6a2f5aa3d7c8a947fb1afc8f","qty":1}]}'
```

**Response (201):**

```json
{
  "order_id": "f82c03bc-4358-4804-9be6-3e1227282fa0",
  "status": "pending",
  "total": 1350000,
  "customer_email": "admin1@tka.its.ac.id",
  "created_at": "2026-06-23T18:52:17.921567+00:00"
}
```

>

---

## GET /orders

```bash
curl http://40.81.25.98/orders \
  -H "Authorization: Bearer <TOKEN>"
```

**Response:** List seluruh order milik user, diurutkan terbaru.

>

---

## GET /orders/\<order_id\>

```bash
curl http://40.81.25.98/orders/6a3c2f9526fa41d4a74a1f42 \
  -H "Authorization: Bearer <TOKEN>"
```

**Response (200):** Detail order berdasarkan order_id.

>

---

## PUT /orders/\<order_id\>/status

```bash
curl -X PUT http://40.81.25.98/orders/f82c03bc-4358-4804-9be6-3e1227282fa0/status \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"status":"processing"}'
```

**Response:**

```json
{ "order_id": "f82c03bc-4358-4804-9be6-3e1227282fa0", "status": "processing" }
```

>

---

## GET /admin/stats

```bash
curl http://40.81.25.98/admin/stats \
  -H "Authorization: Bearer <TOKEN>"
```

**Response:** Aggregasi statistik dashboard (total orders, revenue, top products, dll).

>

---

## Tampilan Frontend

>

---

# 5. Hasil Load Testing

Pengujian dilakukan menggunakan Locust dari perangkat lokal (berbeda dari server aplikasi), diarahkan ke `http://40.81.25.98`.

**Kondisi saat pengujian:**

- VM 1: Nginx Load Balancer aktif
- VM 2: Flask + Gunicorn 3 workers aktif
- VM 3: Flask + Gunicorn 3 workers aktif
- VM 4: MongoDB aktif dengan index pada `created_at` dan `order_id`

---

## Skenario 1 – Maximum RPS (0% Failure)

**Parameter:** Users ditingkatkan bertahap, spawn rate 5, durasi 60 detik.

| Metrik                | Nilai   |
| --------------------- | ------- |
| Users                 | 50      |
| Spawn Rate            | 5       |
| RPS Maksimum          | ~38 RPS |
| Failure Rate          | 0% ✅   |
| Average Response Time | ~250 ms |

>

---

## Skenario 2 – Peak Concurrency (Spawn Rate 50)

**Parameter:** 100 users, spawn rate 50, durasi 60 detik.

| Metrik                | Nilai     |
| --------------------- | --------- |
| Concurrent Users      | 100       |
| Spawn Rate            | 50        |
| RPS Peak              | ~74.2 RPS |
| Failure Rate          | 0% ✅     |
| Average Response Time | ~855 ms   |

>

---

## Skenario 3 – Peak Concurrency (Spawn Rate 100)

**Parameter:** 100 users, spawn rate 100, durasi 60 detik.

| Metrik                | Nilai     |
| --------------------- | --------- |
| Concurrent Users      | 100       |
| Spawn Rate            | 100       |
| RPS Peak              | ~73.4 RPS |
| Failure Rate          | 0% ✅     |
| Average Response Time | ~1086 ms  |

>

---

## Skenario 4 – Peak Concurrency (Spawn Rate 200)

**Parameter:** 100 users, spawn rate 200, durasi 60 detik.

| Metrik                | Nilai     |
| --------------------- | --------- |
| Concurrent Users      | 100       |
| Spawn Rate            | 200       |
| RPS Peak              | ~74.3 RPS |
| Failure Rate          | 0% ✅     |
| Average Response Time | ~1163 ms  |

>

---

## Skenario 5 – Peak Concurrency (Spawn Rate 500)

**Parameter:** 100 users, spawn rate 500, durasi 60 detik.

| Metrik                | Nilai     |
| --------------------- | --------- |
| Concurrent Users      | 100       |
| Spawn Rate            | 500       |
| RPS Peak              | ~12,5 RPS |
| Failure Rate          | 0% ✅     |
| Average Response Time | ~306 ms   |

>

---

## Ringkasan Hasil Load Testing

| Skenario        | Users | Spawn Rate | RPS Peak  | Failure Rate | Avg Response Time |
| --------------- | ----- | ---------- | --------- | ------------ | ----------------- |
| 1 – Max RPS     | 50    | 5          | ~38 RPS   | 0% ✅        | ~250 ms           |
| 2 – Peak SR 50  | 100   | 50         | ~74.2 RPS | 0% ✅        | ~855 ms           |
| 3 – Peak SR 100 | 100   | 100        | ~73.9 RPS | 0% ✅        | ~1095 ms          |
| 4 – Peak SR 200 | 100   | 200        | ~74.6 RPS | 0% ✅        | ~1108 ms          |
| 5 – Peak SR 500 | 100   | 500        | ~9.3 RPS  | 0% ✅        | ~306 ms           |

---

## Monitoring Resource

> Tambahkan screenshot htop VM 1, VM 2, VM 3 saat load testing berlangsung (CPU & RAM)

---

# 6. Analisis

Berdasarkan hasil pengujian:

- **Load Balancing berjalan efektif** — Nginx dengan strategi `least_conn` berhasil mendistribusikan request ke VM 2 dan VM 3 secara merata sehingga tidak ada satu backend yang kelebihan beban.
- **RPS stabil di sekitar 74 RPS** pada skenario 2, 3, dan 4 dengan 100 concurrent users — menunjukkan batas kapasitas sistem dengan konfigurasi saat ini.
- **Skenario 5 (spawn rate 500)** menghasilkan RPS drop ke ~9.3 karena 100 users di-spawn sekaligus dalam waktu sangat singkat (0.2 detik), menyebabkan spike request besar di awal yang membuat server throttling, namun sistem tetap stabil tanpa failure.
- **Response time meningkat** seiring spawn rate yang lebih tinggi (Skenario 3 & 4 ~1100ms) karena antrian request lebih panjang saat users tiba bersamaan.
- **Indexing MongoDB** pada field `created_at` dan `order_id` membantu mempercepat query history orders secara signifikan.
- **Seluruh 100 concurrent users berhasil dilayani tanpa failure** di semua skenario.

---

# 7. Kesimpulan dan Saran

## Kesimpulan

- Sistem Order Processing Service berhasil diimplementasikan pada Microsoft Azure.
- Nginx berhasil mendistribusikan trafik ke dua backend server.
- MongoDB berhasil digunakan sebagai database terpusat.
- Seluruh endpoint berjalan dengan baik.
- Sistem mampu menangani beban tinggi berdasarkan hasil pengujian Locust.

## Saran

- Menggunakan Azure Load Balancer atau Application Gateway untuk implementasi skala produksi.
- Menambahkan mekanisme autoscaling pada backend server.
- Mengimplementasikan Redis sebagai caching layer.
- Menambahkan monitoring dan logging terpusat.
- Menggunakan HTTPS dan domain publik untuk meningkatkan keamanan.

---

# Lampiran

## Struktur Repository

```text
TKA-B2-FP/
├── README.md               ← Laporan utama
├── resources/
│   ├── BE/
│   │   └── app.py          ← Backend Flask
│   ├── FE/
│   │   ├── index.html
│   │   └── styles.css
│   └── Test/
│       └── locustfile.py   ← Script load testing
└── result/
    ├── locust_rps.png
    ├── locust_concurrency_*.png
    └── cpu_usage_*.png
```
