# **BAB 6 — Centralized Logging (Soal 1–10)**

**Gambaran besarnya:** Bab ini membahas bagaimana log dari beberapa aplikasi dikumpulkan ke satu tempat, yaitu PostgreSQL, menggunakan Fluent Bit.

[\[Docker\] Container Logging 에 대해 알아보자. — Developer's Delight](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdna%2FIrQuf%2FbtrIBTXNzWT%2FAAAAAAAAAAAAAAAAAAAAALuIX70Wd9-2wIbbdaiavjEkQMmwkTKgs2UZ0idOBs-q%2Fimg.png%3Fcredential%3DyqXZFxpELC7KVnFOS48ylbz2pIh7yKj8%26expires%3D1796050799%26allow_ip%3D%26allow_referer%3D%26signature%3DEcB2ErMXK7hRMaiGiCHhVbDI1SE%253D)

[sonseungha.tistory.com](https://sonseungha.tistory.com/618?utm_source=chatgpt.com)

### **1. Alur log dari aplikasi ke database**

- **Inti:** Aplikasi menghasilkan log, Docker menangkapnya, Fluent Bit mengumpulkan, lalu PostgreSQL menyimpannya.
- **Demo:** `docker compose ps --all`
- **Jawaban lisan:** "Aplikasi mengirim log ke Docker melalui stdout. Docker meneruskan log ke Fluent Bit melalui port 24224. Fluent Bit kemudian menyimpannya ke tabel `logs.fluentbit` di PostgreSQL."

### **2. Memeriksa isi log**

- **Inti:** Log disimpan dalam format JSONB agar datanya bisa dicari berdasarkan key.
- **Demo:** Periksa tipe data menggunakan `pg_typeof(data)`.
- **Jawaban lisan:** "Data log disimpan dalam format JSONB. Informasi seperti nama container dan sumber log menjadi metadata, sedangkan isi log aplikasi berada di dalam key `log`."

### **3. Konfigurasi output PostgreSQL**

- **Inti:** Fluent Bit harus tahu database dan tabel tujuan penyimpanan log.
- **Demo:** Tampilkan `fluent-bit/fluent-bit.conf` dan jalankan `SHOW search_path;`.
- **Jawaban lisan:** "Fluent Bit menulis ke tabel bernama `fluentbit`. Schema `logs` digunakan secara otomatis karena sudah diatur melalui `search_path` pada akun database."

### **4. Hak akses akun Fluent Bit**

- **Inti:** Akun pengirim log tidak boleh memiliki akses berlebihan.
- **Demo:** `\dp logs.fluentbit`
- **Jawaban lisan:** "Akun Fluent Bit hanya membutuhkan hak untuk membaca dan memasukkan data. Akun tersebut tidak memiliki hak mengubah atau menghapus log. Prinsip ini disebut least privilege."

### **5. Buffer ketika database mati**

- **Inti:** Log tidak langsung hilang ketika PostgreSQL mati karena Fluent Bit memiliki penyimpanan sementara.
- **Demo:** `./scripts/test-buffering.sh`
- **Jawaban lisan:** "Fluent Bit menggunakan filesystem buffer untuk menyimpan log sementara. Ketika database mati, Fluent Bit akan mencoba mengirim ulang. Setelah database hidup, log yang tertahan dikirim kembali."

### **6. Monitoring Fluent Bit**

- **Inti:** Kita harus memastikan Fluent Bit benar-benar menerima dan mengirim log.
- **Demo:** Periksa endpoint `/api/v1/health` dan `/api/v1/metrics`.
- **Jawaban lisan:** "Health menunjukkan apakah Fluent Bit hidup, sedangkan metrics menunjukkan aktivitas pengiriman log, seperti jumlah record, error, retry, dan dropped records."

### **7. Memastikan log tidak hilang atau berlebihan**

- **Inti:** Jumlah log yang diterima harus dibandingkan dengan jumlah log yang dihasilkan aplikasi.
- **Demo:** `SELECT tag, count(*) FROM logs.fluentbit GROUP BY tag;`
- **Jawaban lisan:** "Jika jumlah log berbeda, penyebabnya bisa berupa data lama, duplikasi karena retry, atau log yang hilang akibat buffer penuh. Solusinya adalah menggunakan ID unik dan deduplikasi."

### **8. Menyembunyikan informasi rahasia (Redaction)**

- **Inti:** Informasi sensitif seperti API token tidak boleh tersimpan di database log.
- **Demo:** Masukkan canary `CANARY-XYZ-123`, lalu pastikan tidak ditemukan di database.
- **Jawaban lisan:** "Redaction adalah proses menyamarkan atau menghapus informasi sensitif sebelum log disimpan. Namun, kita juga harus memastikan informasi tersebut tidak bocor ke buffer, terminal, atau file lain."

### **9. Retensi log**

- **Inti:** Log tidak harus disimpan selamanya. Contohnya, hanya disimpan selama tujuh hari.
- **Demo:** Periksa query `DELETE` berdasarkan waktu.
- **Jawaban lisan:** "Retensi adalah kebijakan jangka waktu penyimpanan log. Log yang sudah melewati batas waktu dihapus secara berkala agar kapasitas penyimpanan tidak terus bertambah."

### **10. Ekspor log sebagai bukti**

- **Inti:** Log dapat diekspor untuk keperluan audit atau pemeriksaan insiden.
- **Demo:** Ekspor menggunakan `COPY` atau `\copy`, kemudian hitung checksum dengan `sha256sum`.
- **Jawaban lisan:** "Log diekspor dalam format CSV dengan mencatat rentang waktu, jumlah data, waktu ekspor, dan checksum. Checksum digunakan untuk memeriksa apakah file berubah."

# **BAB 7 — Monitoring (Soal 11–20)**

**Gambaran besarnya:** Bab ini membahas bagaimana memantau kondisi server dan container menggunakan Prometheus dan Grafana.

[DevOps Area - Blog Details](https://images.openai.com/static-rsc-4/TB9667w0I8HVSN4tzSiteDgXH8cshEaMqbdZYfE6gMsol41mJ6APQ6Dav8cEvfuCjEcfjRMm5U7U2H69DXCbuevwOsc7wvnd8atJ5of7AICMbQRNXb4HzktUf11gL3XmCtmCcSY5wDXAJIaYHlGvXSCyj7k7jJ2UiMkdmjMyyps?purpose=inline)

[devopsarea.com](https://devopsarea.com/blog-details/prometheus?utm_source=chatgpt.com)

### **11. Arsitektur monitoring**

- **Inti:** Prometheus mengumpulkan metrics, sedangkan Grafana menampilkan visualisasinya.
- **Demo:** `docker compose ps` dan `ss -lntp`.
- **Jawaban lisan:** "Prometheus mengambil metrics dari Node Exporter dan cAdvisor melalui jaringan internal Docker. Grafana digunakan untuk menampilkan metrics dalam bentuk dashboard."

### **12. Validasi alert**

- **Inti:** Sebelum digunakan, aturan monitoring harus dipastikan benar.
- **Demo:** `promtool check config` dan `promtool test rules`.
- **Jawaban lisan:** "Check config memeriksa apakah konfigurasi valid, sedangkan test rules memeriksa apakah aturan alert bekerja sesuai kondisi yang diharapkan."

### **13. Status target dan nilai&#x20;**`up`

- **Inti:** `up = 1` berarti target berhasil dihubungi, sedangkan `up = 0` berarti gagal di-scrape.
- **Demo:** Buka halaman Targets Prometheus.
- **Jawaban lisan:** "`up = 1` tidak selalu berarti aplikasi sehat. Itu hanya menunjukkan bahwa Prometheus berhasil mengambil metrics dari target tersebut."

### **14. Counter dan rate**

- **Inti:** Counter menunjukkan total akumulasi, sedangkan rate menunjukkan perubahan per detik.
- **Demo:** Bandingkan query `node_cpu_seconds_total` dan `rate(node_cpu_seconds_total[5m])`.
- **Jawaban lisan:** "Counter terus bertambah sehingga kurang cocok untuk melihat penggunaan CPU saat ini. Dengan rate, kita bisa mengetahui kecepatan penggunaan CPU per detik."

### **15. Recording rule CPU**

- **Inti:** Recording rule digunakan untuk menyimpan hasil perhitungan metrics agar bisa digunakan kembali.
- **Demo:** Tampilkan rule `instance:node_cpu_utilisation:ratio_rate5m`.
- **Jawaban lisan:** "Recording rule menghitung penggunaan CPU berdasarkan waktu CPU yang tidak idle. Hasilnya berupa rasio 0 sampai 1, sedangkan query manual bisa menghasilkan persentase 0 sampai 100."

### **16. Load test CPU**

- **Inti:** Menguji apakah monitoring bisa mendeteksi peningkatan penggunaan CPU.
- **Demo:** `docker compose --profile load up -d load-generator`
- **Jawaban lisan:** "Load generator memberikan beban CPU pada container. Karena batas CPU-nya 0,5 core, penggunaan CPU container dapat meningkat mendekati angka tersebut. Prometheus kemudian mendeteksi peningkatan melalui metrics dan alert."

### **17. Negative test TargetDown**

- **Inti:** Menguji apakah sistem bisa mendeteksi ketika target monitoring mati.
- **Demo:** `docker compose stop node-exporter`
- **Jawaban lisan:** "Ketika Node Exporter dimatikan, Prometheus mendapatkan `up = 0`. Alert awalnya berstatus pending, lalu menjadi firing setelah kondisi tersebut berlangsung selama satu menit. Setelah Node Exporter hidup kembali, alert berhenti."

### **18. Provisioning dan akses Grafana**

- **Inti:** Dashboard dan datasource diatur melalui file konfigurasi.
- **Demo:** Tampilkan `datasources.yml` dan `provider.yml`.
- **Jawaban lisan:** "Grafana menggunakan provisioning agar datasource dan dashboard dikonfigurasi otomatis. Karena dashboard bersifat tidak dapat diedit melalui UI, perubahan harus dilakukan melalui file konfigurasi."

### **19. Cardinality dan TSDB**

- **Inti:** Terlalu banyak kombinasi label metrics dapat membebani Prometheus.
- **Demo:** Buka `/api/v1/status/tsdb`.
- **Jawaban lisan:** "Cardinality adalah jumlah time series yang terbentuk dari kombinasi metric dan label. Semakin banyak label yang berubah-ubah, semakin besar penggunaan penyimpanan dan memori Prometheus."

### **20. Integrasi log Bab 6 ke Grafana**

- **Inti:** Log dari PostgreSQL Bab 6 bisa ditampilkan bersama monitoring Bab 7.
- **Demo:** `./scripts/connect-bab6.sh`
- **Jawaban lisan:** "Integrasi dilakukan dengan menghubungkan PostgreSQL Bab 6 ke jaringan Bab 7. Kemudian Grafana diberikan akses read-only untuk membaca log. Dengan begitu, kita bisa membandingkan kondisi CPU dan log pada waktu yang sama."

# **STUDI KASUS PMB (Soal 21–25)**

**Gambaran besarnya:** Bagian ini menguji kemampuan menganalisis masalah pada sistem pendaftaran mahasiswa baru (PMB) menggunakan log dan database.

[Eraser: API Diagrams](https://images.openai.com/static-rsc-4/e-YRVu4Q6div_X87SQDL_hqqDAlWUAGuF8vyeKMgQcvkziS1Zs3SguMB91QhF0tCKAOUuRY6gzGXUtjsgq417RJPR6xjziq82pC7U7F5-OlrKtNRB_kXlwJlaWDWMNazM8KqYLvzW3FlsYF_UPMrxo7EO7TtPLU2vnCGXZAjfnk?purpose=inline)

[eraser.io](https://www.eraser.io/use-case/api-diagrams?utm_source=chatgpt.com)

### **21. Tahap A — Baseline (Kondisi Normal)**

- **Inti:** Mengetahui bagaimana alur pendaftaran ketika sistem berjalan normal.
- **Demo:** `./scripts/set-incident.sh none` lalu `./scripts/demo-normal.sh`.
- **Jawaban lisan:** "Pada kondisi normal, proses berjalan berurutan mulai dari login berhasil, pendaftaran dikirim, verifikasi dimulai, hingga verifikasi selesai. Status akhirnya adalah `VERIFIED`."

**Urutan event:**

1. `login_success` — Login berhasil.
2. `registration_submitted` — Pendaftaran dikirim.
3. `verification_started` — Verifikasi dimulai.
4. `verification_completed` — Verifikasi selesai.

Status akhir: VERIFIED

### **22. Tahap B — Failure Injection (Simulasi Kegagalan)**

- **Inti:** Sengaja membuat sistem mengalami gangguan untuk melihat bagaimana log merekam masalah.
- **Demo:** `./scripts/set-incident.sh timeout`.
- **Jawaban lisan:** "Ketika terjadi timeout, verification-worker mencoba melakukan verifikasi sebanyak tiga kali. Namun, proses tetap gagal karena dependency tidak merespons. Akhirnya muncul error `VERIFY-TIMEOUT` dan status pendaftaran menjadi `FAILED`."

**Urutan masalah:**

1. Worker mencoba verifikasi.
2. Terjadi retry sebanyak 3 kali.
3. Dependency mengalami timeout.
4. Muncul error `VERIFY-TIMEOUT`.

Status akhir: FAILED

### **23. Tahap C — Recovery (Pemulihan)**

- **Inti:** Memastikan sistem bisa kembali normal setelah gangguan selesai.
- **Demo:** `./scripts/set-incident.sh none`, lalu kirim pendaftaran baru.
- **Jawaban lisan:** "Setelah gangguan diperbaiki, transaksi baru berhasil diverifikasi. Namun, transaksi lama yang berstatus FAILED tidak otomatis diproses ulang karena worker hanya mengambil data berstatus SUBMITTED."

**Solusi yang disarankan:**

- Retry dengan batas percobaan.
- Menggunakan exponential backoff (jeda retry semakin lama).
- Dead-letter queue untuk transaksi yang terus gagal.
- Requeue manual dengan pencatatan.

### **24. Tahap D — Resilience Logging**

- **Inti:** Menguji apakah log tetap tersimpan ketika database logging mati.
- **Demo:** Matikan `log-db`, kirim transaksi, hidupkan kembali `log-db`, lalu periksa jumlah log.
- **Jawaban lisan:** "Ketika database log mati, Fluent Bit menyimpan log sementara di filesystem buffer. Setelah database hidup kembali, log dikirim ulang. Namun, mekanisme ini belum menjamin exactly-once karena masih ada kemungkinan duplikasi atau kehilangan log."

**Istilah penting:**

| Istilah         | Arti sederhana                       |
| --------------- | ------------------------------------ |
| Buffer          | Penyimpanan sementara                |
| Retry           | Mencoba mengirim ulang               |
| Duplicate       | Data terkirim lebih dari sekali      |
| Dropped records | Data log yang terbuang               |
| Exactly-once    | Setiap data diproses tepat satu kali |

### **25. Privasi, Kepercayaan, dan Kesiapan Produksi**

- **Inti:** Sistem harus aman, menjaga data pribadi, dan siap digunakan secara nyata.
- **Demo:** `./scripts/verify.sh` dan periksa `reports/config-sha256.txt`.
- **Jawaban lisan:** "Log tidak menyimpan nama dan email pengguna untuk menjaga privasi. Sistem menggunakan pengenal pseudonim seperti user_ref dan registration_id. Request ID dari pengguna juga divalidasi agar tidak digunakan untuk log injection."

**Hal yang perlu diperbaiki sebelum produksi:**

1. Menggunakan HTTPS/TLS untuk komunikasi.
2. Menggunakan secret manager untuk menyimpan credential.
3. Memberikan autentikasi pada Prometheus, Grafana, dan Fluent Bit.
4. Menambahkan Alertmanager untuk notifikasi.
5. Mengatur retensi dan hak akses log.
6. Menyediakan backup dan high availability.
7. Membatasi hak akses akun ingest.
8. Menghilangkan konfigurasi `privileged`.
9. Mengamankan cookie dan koneksi HTTPS.
10. Melakukan pemeriksaan keamanan Docker image.

# **Hafalan cepat sebelum ujian**

Ini bagian yang paling penting untuk diingat. Kalau lupa detail konfigurasi, pahami konsep dan fungsi masing-masing komponen.

## **1. Perbedaan komponen**

| Komponen      | Fungsi                             |
| ------------- | ---------------------------------- |
| Docker        | Menjalankan aplikasi dan container |
| Fluent Bit    | Mengumpulkan dan mengirim log      |
| PostgreSQL    | Menyimpan log                      |
| Prometheus    | Mengumpulkan metrics               |
| Grafana       | Menampilkan dashboard              |
| Node Exporter | Mengambil metrics server           |
| cAdvisor      | Mengambil metrics container        |

## **2. Istilah yang sering ditanyakan**

| Istilah             | Pengertian                                        |
| ------------------- | ------------------------------------------------- |
| Centralized Logging | Mengumpulkan log ke satu tempat                   |
| JSONB               | Format penyimpanan data JSON di PostgreSQL        |
| Buffering           | Menyimpan data sementara                          |
| Retry               | Mengirim ulang data yang gagal                    |
| Redaction           | Menghapus atau menyamarkan data sensitif          |
| Retention           | Batas waktu penyimpanan data                      |
| Least Privilege     | Memberikan hak akses seminimal mungkin            |
| Metrics             | Data pengukuran kondisi sistem                    |
| Scraping            | Proses Prometheus mengambil metrics               |
| Counter             | Nilai akumulasi yang terus bertambah              |
| Rate                | Laju perubahan nilai per detik                    |
| Alert               | Peringatan ketika kondisi tertentu terpenuhi      |
| Cardinality         | Jumlah kombinasi time series                      |
| Provisioning        | Konfigurasi otomatis melalui file                 |
| Recovery            | Proses pemulihan setelah gangguan                 |
| Idempotency         | Operasi yang diulang tidak menimbulkan efek ganda |

## **3. Perbedaan yang wajib dipahami**

**Logging vs Monitoring**

- Logging menjawab: *Apa yang terjadi?*
- Monitoring menjawab: *Bagaimana kondisi sistem?*

  **Fluent Bit vs Prometheus**
- Fluent Bit mengumpulkan log.
- Prometheus mengumpulkan metrics.

  **Prometheus vs Grafana**
- Prometheus mengambil dan menyimpan metrics.
- Grafana menampilkan metrics dalam dashboard.

  **Pending vs Firing**
- Pending: kondisi alert sudah terpenuhi, tetapi durasi `for` belum selesai.
- Firing: kondisi alert sudah terpenuhi selama durasi yang ditentukan.

  **Retry vs Exactly-once**
- Retry: mencoba mengirim ulang data.
- Exactly-once: menjamin setiap data diproses tepat satu kali. Retry saja tidak menjaminnya.

## **4. Alur besar yang perlu dihafalkan**

**Bab 6 — Logging**

Aplikasi menghasilkan log

Docker menangkap log

Fluent Bit mengumpulkan log

PostgreSQL menyimpan log

**Bab 7 — Monitoring**

Node Exporter + cAdvisor

Prometheus mengumpulkan metrics

Grafana menampilkan dashboard

## **5. Tips menjawab pertanyaan lanjutan penguji**

Kalau penguji bertanya "Mengapa?", gunakan pola sederhana ini:

1. **Sebutkan penyebab:** Apa yang membuat kondisi tersebut terjadi?
2. **Sebutkan dampak:** Apa akibatnya terhadap sistem?
3. **Sebutkan solusi:** Bagaimana cara mencegah atau memperbaikinya?

Contoh:

**Penguji:** Mengapa log bisa hilang ketika database mati?

**Jawaban:** "Karena Fluent Bit memiliki kapasitas buffer yang terbatas. Jika database terlalu lama mati hingga buffer penuh, log bisa terbuang. Solusinya adalah memantau kapasitas buffer, menyediakan ruang penyimpanan yang cukup, dan mempercepat pemulihan database."

**Saran belajar:** Prioritaskan memahami alur Bab 6 dan Bab 7, fungsi setiap komponen, serta kondisi normal → gagal → recovery pada PMB. Jangan hanya menghafalkan perintah karena penguji kemungkinan akan menanyakan alasan dan dampak dari setiap konfigurasi.
