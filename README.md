# Monitoring Sistem K24Klik Order API

Merupakan stack monitoring sistem `Order API` untuk memantau apakah sistem berjalan dengan sehat, stack ini menggunakan Tools Docker, Prometheus, Grafana & PromQL

## Daftar Isi

- [Arsitektur](#arsitektur)
- [Struktur Project](#struktur-project)
- [Persyaratan](#persyaratan)
- [Instalasi dan Menjalankan](#instalasi-dan-menjalankan)
- [Kredensial Default](#kredensial-default)
- [Cara Verifikasi Dashboard](#cara-verifikasi-dashboard)
- [Keputusan Teknis](#keputusan-teknis)
- [Penjelasan PromQl](#penjelasan-promql)
- [Checklist Penting](#checklist-penting)
- [Referensi](#referensi)

## Arsitektur

       +-------------------------------------------------------------+
       |                         TARGET HOST                         |
       |  (Server / Docker Engine)                                   |
       +-------------------------------------------------------------+
               |                           |
       [ Node Exporter ]           [ cAdvisor ]
       (Hardware Metrics)      (Container Metrics)
               |                           |
               +-------------+-------------+
                             |
                             v [Pull Metrics every 15s]
                    +-------------------+
                    |    PROMETHEUS     |<--- [ Alert Rules ]
                    |  (Time Series DB) |
                    +-------------------+
                             |
               +-------------+-------------+
               |                           |
               v                           v
        +--------------+           +----------------+
        |   GRAFANA    |           |  ALERTMANAGER  |
        | (Dashboards) |           | (Notifications)|
        +--------------+           +----------------+
               |                           |
               v                           v
        [ Admin Browser ]           [ Slack/Webhook ]

### Penjelasan
- Target Host: Lingkungan tempat aplikasi K24Klik berjalan.
- Exporters (Node & cAdvisor): Bertugas mengambil data mentah dari sistem dan mengubahnya menjadi format metrik.
- Prometheus: "Otak" yang menarik (pull) data secara berkala dan mengevaluasi aturan alert.
- Grafana: Menarik data dari Prometheus untuk visualisasi dashboard 9 panel.
- Alertmanager: Menerima sinyal bahaya dari Prometheus dan meneruskannya ke Slack.

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
9. Akses Cadvisor di `http://localhost:8080`.

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

## Penjelasan PromQL

### CPU Usage Query
`100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
| Bagian | Penjelasan |
|---|---|
| `node_cpu_seconds_total` | Metrik dari Node Exporter — counter total detik yang dihabiskan CPU di setiap mode (idle, user, system, dll) |
| `{mode="idle"}` | Filter hanya mode "idle" — waktu CPU tidak mengerjakan apapun |
| `rate(...[5m])` | Menghitung laju perubahan counter per detik, dirata-rata selama 5 menit terakhir. `rate()` dipakai karena `node_cpu_seconds_total` adalah counter yang terus naik |
| `avg by(instance)` | Merata-rata semua CPU core per instance/host, sehingga hasil akhir adalah satu angka per server, bukan per core |
| `* 100` | Mengkonversi dari desimal (0–1) ke persentase (0–100) |
| `100 - (...)` | Membalik nilai idle menjadi usage — jika idle 30%, maka usage = 70% |
**Mengapa menggunakan `rate()` bukan `irate()`?**
`rate()` lebih stabil untuk alerting karena merata-rata dalam window 5 menit, sehingga tidak terpicu oleh spike sesaat. `irate()` lebih responsif tapi terlalu sensitif untuk threshold alert produksi.
**Mengapa window `[5m]`?**
Cukup panjang untuk meredam noise, tapi cukup pendek untuk tetap mendeteksi masalah real-time. Cocok dengan scrape_interval 15s karena ada ~20 data point dalam window tersebut.

### Memory Usage Query
`(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100`
| Bagian | Penjelasan |
|---|---|
| `node_memory_MemTotal_bytes` | Total RAM fisik di host |
| `node_memory_MemAvailable_bytes` | RAM yang tersedia untuk proses baru (sudah memperhitungkan buffer/cache yang bisa dibebaskan) |
| `1 - (Available / Total)` | Proporsi memori yang sedang terpakai |
| `* 100` | Konversi ke persentase |
**Mengapa `MemAvailable` bukan `MemFree`?**
`MemFree` hanya menghitung RAM yang benar-benar kosong, mengabaikan buffer dan cache yang sebenarnya bisa dibebaskan oleh kernel kapanpun dibutuhkan. `MemAvailable` lebih akurat mencerminkan kondisi memori yang sesungguhnya tersedia.

### Disk Usage Query
`(1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100`
| Bagian | Penjelasan |
|---|---|
| `node_filesystem_size_bytes` | Total kapasitas filesystem |
| `node_filesystem_avail_bytes` | Ruang yang tersedia untuk pengguna non-root |
| `{mountpoint="/"}` | Filter hanya root filesystem — mountpoint utama sistem |
| `1 - (avail / size)` | Proporsi disk yang terpakai |

### Container Memory Query (ContainerHighMemory alert)
`(container_memory_usage_bytes / node_memory_MemTotal_bytes) * 100 > 85`
| Bagian | Penjelasan |
|---|---|
| `container_memory_usage_bytes` | Memori yang dipakai oleh container (dari cAdvisor) |
| `node_memory_MemTotal_bytes` | Total RAM host — dipakai sebagai pembanding karena container berbagi RAM host |
| `> 85` | Threshold: alert jika satu container memakai lebih dari 85% total RAM host |

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
- Docker Documentation check [documentation](https://docs.docker.com/)
- Prometheus Query Language (PromQL) check [reference](https://prometheus.io/docs/prometheus/latest/querying/functions)
- Grafana Dashboard & Panel Visualization check [documentation](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/)