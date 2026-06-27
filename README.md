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
4. Backend berkomunikasi dengan MongoDB pada VM 4 melalui port 27017 untuk membaca atau menyimpan data.
5. Response dikirim kembali ke pengguna.

## Spesifikasi VM

| VM        | Role                     | OS            | vCPU | RAM  | Harga/Bulan |
| --------- | ------------------------ | ------------- | ---- | ---- | ----------- |
| VM 1      | Load Balancer + Frontend | Ubuntu Server | 2    | 1 GB | $9.56       |
| VM 2      | Backend API 1            | Ubuntu Server | 2    | 4 GB | $17.96      |
| VM 3      | Backend API 2            | Ubuntu Server | 2    | 4 GB | $17.96      |
| VM 4      | MongoDB Database         | Ubuntu Server | 2    | 4 GB | $17.96      |
| **Total** |                          |               |      |      | **$63.44**  |

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

# 3. Implementasi

## 3.1 Deploy MongoDB

### Instalasi MongoDB

MongoDB Community Edition diinstal pada VM4 sebagai database server menggunakan repository resmi MongoDB.

```bash
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

### Konfigurasi MongoDB

Agar dapat diakses oleh Backend API pada VM2 dan VM3, konfigurasi `bindIp` diubah menjadi `0.0.0.0`, kemudian service MongoDB di-restart.

```yaml
bindIp: 0.0.0.0
```

```bash
sudo systemctl restart mongod
```

**Dokumentasi Konfigurasi MongoDB**

> **Screenshot 1** – Status MongoDB 

<img width="1451" height="335" alt="image" src="https://github.com/user-attachments/assets/3f7a3a0e-0495-4075-bf41-0b4b3b298c00" />

---

> **Screenshot 2** – Konfigurasi `bindIp` dan port 27017 

<img width="1177" height="97" alt="image" src="https://github.com/user-attachments/assets/53a6b90e-2e36-46a4-97b7-22bffbfbf356" />

---

### Restore Database

Database direstore menggunakan file dump yang tersedia pada repository.

```bash
mongorestore --drop ~/fp-tka-26/Resources/DB/dump
```

Verifikasi:

```javascript
use orderdb

show collections

db.products.countDocuments()
db.users.countDocuments()
db.orders.countDocuments()
```

**Dokumentasi Restore Database**

> **Screenshot 3** – Hasil restore dan verifikasi database

<img width="482" height="392" alt="image" src="https://github.com/user-attachments/assets/ce0084f5-bd41-4e8c-b07c-c2df7d1c399a" />

---

### Optimasi Database (Indexing)

Untuk meningkatkan performa query pada collection `orders`, dibuat index pada field `created_at`.

```javascript
db.orders.createIndex({ created_at: -1 })
db.orders.getIndexes()
```

**Dokumentasi Hasil Indexing**

> **Screenshot 4** – 

<img width="701" height="142" alt="image" src="https://github.com/user-attachments/assets/bad1ed24-d179-4f2e-90af-b7e2e6baea5a" />

---

## 3.2 Deploy Backend API

### Clone Source Code

```bash
git clone https://github.com/fuaddary/fp-tka-26
cd fp-tka-26/Resources/BE
cp app.py requirements.txt ~
cd ~
```

### Set Python Virtual Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```
<img width="818" height="75" alt="image" src="https://github.com/user-attachments/assets/0be7a1ff-3034-4f24-abb3-a9a4d6bbb473" />


### Install Dependency

```bash
pip install -r requirements.txt
```
<img width="1906" height="353" alt="image" src="https://github.com/user-attachments/assets/dd9c0fa1-1fe4-45a5-9698-d3e1089abdda" />

### Set System Variables

```bash
echo 'export MONGO_URI="mongodb://4.240.92.222:27017/orderdb"' >> ~/.bashrc
echo 'export JWT_SECRET="TK4_FPdul5"' >> ~/.bashrc
```
<img width="1284" height="49" alt="image" src="https://github.com/user-attachments/assets/74821351-e1ea-4230-af5b-f0d328a3c412" />

### Menjalankan Gunicorn

```bash
gunicorn -w 3 -b 0.0.0.0:5000 app:app
```
<img width="1572" height="178" alt="image" src="https://github.com/user-attachments/assets/3bf39166-7524-4e44-b50d-1bbf0b5e4d79" />

### Konfigurasi Service Daemon (Persistence)

Konfigurasi file `/etc/systemd/system/gunicorn-backend.service`:

```service
[Unit]
Description=Gunicorn instance serving TKA-B2-FP Order Processing API
After=network.target

[Service]
User=azureuser
Group=www-data
WorkingDirectory=/home/azureuser/
Environment="MONGO_URI=mongodb://4.240.92.222:27017/orderdb"
Environment="JWT_SECRET=TK4_FPdul5"
ExecStart=/home/azureuser/.venv/bin/gunicorn -w 3 -b 0.0.0.0:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```
<img width="1241" height="508" alt="image" src="https://github.com/user-attachments/assets/97a76c05-e4a7-48e4-ac92-b9dd2149ea28" />


### Mengaktifkan Autostart Backend

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn-backend
```
![Uploading image.png…]()

---

## 3.3 Konfigurasi Firewall

Port 5000 pada kedua Backend API hanya mengizinkan koneksi dari VM 1 (Load Balancer) melalui aturan Network Security Group (NSG), sehingga backend tidak dapat diakses langsung dari internet.

> VM 1
<img width="1553" height="715" alt="image" src="https://github.com/user-attachments/assets/f9393d48-ebf6-4615-97fc-857996148de1" />

> VM 2
<img width="1080" height="1451" alt="08e9ed74-84bb-493e-b8fd-eef35d55d621" src="https://github.com/user-attachments/assets/178c7e14-aecd-4d59-ae4a-a5bffaa170a6" />

> VM 3
<img width="1600" height="681" alt="08f06ea8-3fb3-45df-be7f-6568cc0e582a" src="https://github.com/user-attachments/assets/0458d87b-8aed-4b74-b20f-c1fa86e8b1d6" />

---

## 3.4 Konfigurasi Nginx Load Balancer

### Instalasi Nginx

```bash
sudo apt install nginx -y
```

### Konfigurasi Upstream

```nginx
upstream backend_servers {
    least_conn;
    server 20.40.54.202:5000;
    server 20.40.58.217:5000;
}

server{
    listen 80;
    server_name 40.81.25.98;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /health {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /auth/ {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /products {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /orders {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /admin/ {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Symlink Routing Nginx

```bash 
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/loadbalancer /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Check Nginx Berjalan

```bash
sudo systemctl is-enabled nginx
```

![nginx running](./assets/nginx.png)

---

## 3.5 Deploy Frontend

File frontend (`index.html` dan `styles.css`) ditempatkan pada direktori:

```bash
/var/www/html/
```

Frontend di-host menggunakan Nginx pada VM 1 sehingga dapat diakses melalui IP publik Load Balancer.

![forntend](./assets/frontend.png)
![frontend 2](./assets/frontend2.png)

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

![endpoint1](./assets/endpoint1.png)

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

![endpoint2](./assets/endpoint2.png)

---

## GET /products

```bash
curl http://40.81.25.98/products \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI2YTJmNWFhM2Q3YzhhOTQ3ZmIxYWZjZTYiLCJyb2xlIjoiYWRtaW4iLCJleHAiOjE3ODI0MzA4NjQsImlhdCI6MTc4MjM0NDQ2NH0.4V2q3QYZePKOCM7T1smUwp57xeOvHIM1JxbAnlmIw8s"
```

**Response:** Mengembalikan 92 produk dengan paginasi (20 per halaman).

![endpoint3](./assets/endpoint3.png)

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

![endpoint4](./assets/endpoint4.png)

---

## GET /orders

```bash
curl http://40.81.25.98/orders \
  -H "Authorization: Bearer <TOKEN>"
```

**Response:** List seluruh order milik user, diurutkan terbaru.

![endpoint5](./assets/endpoint5.png)

---

## GET /orders/\<order_id\>

```bash
curl http://40.81.25.98/orders/6a3c2f9526fa41d4a74a1f42 \
  -H "Authorization: Bearer <TOKEN>"
```

**Response (200):** Detail order berdasarkan order_id.

<img width="449" height="83" alt="Screenshot 2026-06-25 110926" src="https://github.com/user-attachments/assets/ae6fd2a7-b96a-49d7-925a-ccf84c78ed21" />

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

![endpoint6](./assets/endpoint6.png)

---

## GET /admin/stats

```bash
curl http://40.81.25.98/admin/stats \
  -H "Authorization: Bearer <TOKEN>"
```

**Response:** Aggregasi statistik dashboard (total orders, revenue, top products, dll).

![endpoint7](./assets/endpoint7.png)

---

## Tampilan Frontend

![frontend-view](./assets/frontend-view.png)

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
| RPS Maksimum          | ~37.3 RPS |
| Failure Rate          | 0% ✅   |
| Average Response Time | ~283 ms |

![locust1](./assets/locust1.png)

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

![locust2](./assets/locust2.png)

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

![locust3](./assets/locust3.png)

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

![locust4](./assets/locust4.png)

---

## Skenario 5 – Peak Concurrency (Spawn Rate 500)

**Parameter:** 100 users, spawn rate 500, durasi 60 detik.

| Metrik                | Nilai     |
| --------------------- | --------- |
| Concurrent Users      | 100       |
| Spawn Rate            | 500       |
| RPS Peak              | ~12,5 RPS |
| Failure Rate          | 0% ✅     |
| Average Response Time | ~312 ms   |

![locust4](./assets/locust5.png)

---

## Ringkasan Hasil Load Testing

| Skenario        | Users | Spawn Rate | RPS Peak  | Failure Rate | Avg Response Time |
| --------------- | ----- | ---------- | --------- | ------------ | ----------------- |
| 1 – Max RPS     | 50    | 5          | ~37.3 RPS | 0% ✅        | ~283 ms           |
| 2 – Peak SR 50  | 100   | 50         | ~74.2 RPS | 0% ✅        | ~855 ms           |
| 3 – Peak SR 100 | 100   | 100        | ~73.9 RPS | 0% ✅        | ~1095 ms          |
| 4 – Peak SR 200 | 100   | 200        | ~74.6 RPS | 0% ✅        | ~1108 ms          |
| 5 – Peak SR 500 | 100   | 500        | ~12.5 RPS | 0% ✅        | ~312 ms           |

---

## Monitoring Resource

> Tambahkan screenshot htop VM 1, VM 2, VM 3 saat load testing berlangsung (CPU & RAM)

---

# 6. Analisis

Berdasarkan hasil pengujian:

- **Load Balancing berjalan efektif** — Nginx dengan strategi `least_conn` berhasil mendistribusikan request ke VM 2 dan VM 3 secara merata, sehingga beban kerja dapat dibagi ke kedua server backend.
- **RPS stabil di sekitar 74 RPS** pada skenario 2, 3, dan 4 dengan 100 concurrent users — menunjukkan kapasitas maksimum sistem dengan konfigurasi saat ini.
- **Skenario 5 (spawn rate 500)** menghasilkan RPS drop ke ~12.5. Hal ini disebabkan seluruh user dibuat hampir bersamaan sehingga terjadi lonjakan request dalam waktu yang sangat singkat. Meskipun demikian, sistem tetap dapat memproses seluruh request tanpa failure.
- **Response time meningkat** seiring spawn rate yang lebih tinggi (Skenario 3 & 4) karena antrian request lebih panjang saat users tiba bersamaan.
- **Indexing MongoDB** pada field `created_at` serta `order_id` membantu proses pengambilan data histori pesanan sesuai implementasi yang dilakukan.
- **Seluruh 100 concurrent users berhasil dilayani tanpa failure** di semua skenario.

---

# 7. Kesimpulan dan Saran

## Kesimpulan

- Sistem Order Processing Service berhasil diimplementasikan pada Microsoft Azure.
- Nginx berhasil berfungsi sebagai load balancer yang mendistribusikan request ke dua backend Flask.
- Backend API berhasil terhubung dengan MongoDB sebagai database terpusat.
- Seluruh endpoint aplikasi dapat diakses dan berfungsi sesuai dengan kebutuhan sistem.
- Berdasarkan hasil pengujian menggunakan Locust, sistem mampu melayani hingga 100 concurrent users pada seluruh skenario tanpa mengalami failure, meskipun terjadi peningkatan response time ketika jumlah request yang masuk secara bersamaan semakin tinggi.

## Saran

- Menggunakan Azure Load Balancer atau Application Gateway untuk implementasi skala produksi.
- Menambahkan mekanisme autoscaling pada backend server.
- Menambahkan monitoring dan logging terpusat.
- Menggunakan HTTPS dan domain publik untuk meningkatkan keamanan.
- Melakukan pengujian dengan jumlah concurrent users yang lebih besar dan durasi pengujian yang lebih lama untuk mengetahui batas maksimum performa sistem.

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
