# Monitoring Sistem K24Klik Order API

Merupakan stack monitoring sistem `Order API` untuk memantau apakah sistem berjalan dengan sehat, stack ini menggunakan Tools Docker, Prometheus,Grafana & PromQL

## Daftar Isi

- [Struktur Project](#-struktur-project)
- [Persyaratan](#-persyaratan)
- [Instalasi dan Menjalankan](#instalasi-dan-menjalankan)
- [Kredensial Default](#-kredensial-default)
- [Cara Verifikasi Dashboard](#-cara-verifikasi-dashboard)
- [Keputusan Teknis](#-keputusan-teknis)
- [Checklist Penting](#-checklist-penting)
- [Referensi](#-referensi)

## Struktur Project

```
monitoring-stack/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── rules.yml
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── prometheus.yml      
│       └── dashboards/
│           ├── dashboard.yml
│           └── k24klik-overview.json       
├── alertmanager/
│   └── alertmanager.yml
├── .env.example
└── README.md
```

## Persyaratan
- Docker Engine v20.10+
- Docker Compose v2.0+
- OS: Linux / macOS / Windows (Docker Desktop)

## Instalasi dan Menjalankan
1. **Clone repository**
```bash
git clone https://github.com/irvanrifai/skill-test-k24klik-order-api-monitoring-stack.git monitoring-stack
cd monitoring-stack
```
2. **Konfigurasi env**
```bash
cp .env.example .env

# isi `K24KLIK_ENV=production`.
# isi `GF_SECURITY_ADMIN_PASSWORD=k24monitoring2026`.
``` 
3. **Pastikan Docker Engine(Dekstop) sudah Berjalan**
4. **Jalankan Docker Compose**
```bash
docker compose up -d
```
5. Akses Prometheus di `http://localhost:9090`.
6. Akses Grafana di `http://localhost:3000`.
7. Akses Alert Manager di `http://localhost:9093`.
8. Akses Node Exporter di `http://localhost:9100`.
9. Akses Cadvisor di `http://localhost:8081`.

## Kredensial Default
- **Grafana**: admin / k24monitoring2026
- **Prometheus**: http://localhost:9090

## Cara Verifikasi Dashboard
1. Login ke Grafana (localhost:3000).
2. Klik menu **Dashboards**.
3. Pilih dashboard **"K24Klik Order API — Infrastructure Overview"**.
4. Pastikan 9 panel (Stat, Gauge, Time Series, Bar Gauge, Table) sudah terisi data.

## Keputusan Teknis
### Docker Compose
- Port Mapping cAdvisor: Di dokumen, cadvisor menggunakan port 8080. Namun, API Gateway K24Klik juga berjalan di port 8080. Ini akan terjadi conflict, sementara saya tetap menggunakan 8080 sebagai default port cadvisor.
- Healthcheck: Saya menambahkan wget sebagai metode check karena image Prometheus dan Grafana versi ini sangat minimalis.
- Persistence: Menggunakan named volumes (prometheus_data, grafana_data) memastikan data monitoring tidak hilang saat container dihapus atau di-update
### Prometheus
- Scrape Interval (15s): Dipilih agar data cukup real-time tanpa membebani resource server secara berlebihan.
- Absent Query: Gunakan absent() untuk mendeteksi kontainer yang hilang secara total dari metrik.
- Node Exporter: Metrik node_cpu_seconds_total digunakan untuk menghitung penggunaan CPU rata-rata dengan mengurangi waktu idle dari total kapasitas.
- Disk Usage Mountpoint: Monitoring Disk: Karena lingkungan Docker Desktop memiliki isolasi filesystem, saya mengarahkan query disk usage ke mountpoint yang aktif (seperti /var atau /tmp) untuk memastikan metrik tetap akurat meskipun berjalan di dalam container.
### Alert Manager
- Routing Logic: Kita membagi jalur pengiriman berdasarkan tingkat keparahan (severity). Alert Critical dikirim seketika (group_wait: 0s) karena butuh penanganan cepat, sementara Warning ditahan dulu selama 5 menit agar tidak berisik jika banyak masalah kecil muncul bersamaan.
- Inhibition Rules: Ini adalah bagian paling cerdas dari DevOps. Jika sebuah server sudah dalam kondisi Critical (misal CPU 95%), maka notifikasi Warning (CPU 80%) di server yang sama tidak perlu dikirim lagi karena kita sudah tahu ada masalah besar di sana.
- Slack Integration: Sesuai instruksi, kita menggunakan webhook receiver ke URL dummy http://localhost:5001 sebagai representasi integrasi Slack.
### Grafana
- Provisioning vs Manual Import: Saya memilih menggunakan Dashboard Provisioning (file JSON) daripada import manual. Hal ini menjamin bahwa siapapun yang menjalankan docker compose up akan mendapatkan visualisasi dashboard yang identik.

## Checklist Penting
- [x] docker compose up -d berjalan tanpa error
- [x] docker compose ps — semua service healthy
- [x] Prometheus (localhost:9090/targets) — semua target UP
- [x] Grafana (localhost:3000) — dashboard K24Klik muncul otomatis (admin/k24monitoring2026)
- [x] Alertmanager (localhost:9093) — dapat diakses
- [x] Alert rules terdefinisi di Prometheus (localhost:9090/rules)
- [x] Semua 8 alert ada di rules.yml dengan label dan annotation lengkap
- [x] README.md lengkap: setup guide + keputusan teknis
- [x] Tidak ada file sensitif (password, .env) yang ter-commit
- [x] Repository bersih: .gitignore sudah dikonfigurasi

## Referensi
- Docker Documentation chech [documentation](https://docs.docker.com/)
- Prometheus Query Language (PromQL) check [reference](https://prometheus.io/docs/prometheus/latest/querying/functions)
- Grafana Dashboard & Panel Visualization check [documentation](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/)