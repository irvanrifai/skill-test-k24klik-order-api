# Monitoring Sistem K24Klik Order API
Stack monitoring

# Docker Compose
- Port Mapping cAdvisor: Di dokumen, cadvisor menggunakan port 8080. Namun, API Gateway K24Klik juga berjalan di port 8080. Maka, saya arahkan port host-nya ke 8081 untuk menghindari bentrokan (conflict).
- Healthcheck: Saya menambahkan wget sebagai metode check karena image Prometheus dan Grafana versi ini sangat minimalis.
- Persistence: Menggunakan named volumes (prometheus_data, grafana_data) memastikan data monitoring tidak hilang saat container dihapus atau di-update

# Prometheus
- Scrape Interval (15s): Dipilih agar data cukup real-time tanpa membebani resource server secara berlebihan.
- Absent Query: Gunakan absent() untuk mendeteksi kontainer yang hilang secara total dari metrik.
- Node Exporter: Metrik node_cpu_seconds_total digunakan untuk menghitung penggunaan CPU rata-rata dengan mengurangi waktu idle dari total kapasitas.

# Alert Manager
- Routing Logic: Kita membagi jalur pengiriman berdasarkan tingkat keparahan (severity). Alert Critical dikirim seketika (group_wait: 0s) karena butuh penanganan cepat, sementara Warning ditahan dulu selama 5 menit agar tidak berisik jika banyak masalah kecil muncul bersamaan.
- Inhibition Rules: Ini adalah bagian paling cerdas dari DevOps. Jika sebuah server sudah dalam kondisi Critical (misal CPU 95%), maka notifikasi Warning (CPU 80%) di server yang sama tidak perlu dikirim lagi karena kita sudah tahu ada masalah besar di sana.
- Slack Integration: Sesuai instruksi, kita menggunakan webhook receiver ke URL dummy http://localhost:5001 sebagai representasi integrasi Slack.

