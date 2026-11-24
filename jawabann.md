## Bab 6 — Centralized Logging

**1. Alur log end-to-end**
- **Demo:** `docker compose ps --all`
- **Jelaskan:**
  1. Generator mencetak JSON ke stdout.
  2. Docker Engine menangkapnya dan logging driver `fluentd` mengirimnya ke `127.0.0.1:24224`, yaitu port Fluent Bit yang dipublikasikan di host.
  3. Input `forward` Fluent Bit menerima dan membuffer record, lalu output `pgsql` menulisnya ke `logs.fluentbit` (kolom `tag`, `time`, `data` JSONB).
  
  Generator berstatus `Exited (0)` karena selesai setelah 20 baris, sedangkan PostgreSQL dan Fluent Bit tetap `Up`.
- **Lanjutan:** Driver dijalankan oleh Docker daemon di host, bukan di jaringan Compose, jadi nama `fluent-bit` tidak bisa di-resolve. Karena `fluentd-async: "true"`, container tetap start, tetapi log menumpuk di buffer driver yang terbatas lalu hilang. Tabel tidak bertambah.

**2. Verifikasi record dan JSONB**
- **Demo:** query distribusi per tag dari `analysis.sql`, lalu `SELECT pg_typeof(data), count(*) FROM logs.fluentbit GROUP BY pg_typeof(data);`
- **Jelaskan:** Hasilnya `jsonb`. `container_name` dan `source` adalah key metadata dari Docker logging driver, ada di dalam objek `data` (`data->>'container_name'`, `data->>'source'`). Isi log aplikasi tetap berupa string di `data->>'log'`.
- **Lanjutan:** Gunakan parser JSON Fluent Bit (filter `parser` dengan `Key_Name log`, `Reserve_Data On`) supaya isinya menjadi field `severity`. Cara lain: view atau expression index di SQL, misalnya `(data->>'log')::jsonb->>'severity'`, dengan guard `left(ltrim(data->>'log'),1)='{'` agar log non-JSON tidak membuat cast gagal.

**3. Konfigurasi output PostgreSQL**
- **Demo:** tampilkan `fluent-bit/fluent-bit.conf`, lalu `SHOW search_path;` sebagai `fluentbit_user`.
- **Jelaskan:** Plugin hanya menulis ke tabel `fluentbit` tanpa opsi `Schema`. Skrip init menjalankan `ALTER ROLE fluentbit_user IN DATABASE ... SET search_path TO logs, public`, sehingga `fluentbit` otomatis menunjuk `logs.fluentbit`.
- **Lanjutan:** Yang saya periksa:
  1. `SHOW search_path` pada sesi baru milik user itu.
  2. `\dt logs.*`.
  3. Nama `Table` dan `Database` di conf.
  4. Apakah volume lama membuat skrip init tidak berjalan, karena init hanya jalan saat data directory kosong.
  5. Hak `USAGE` pada schema.

**4. Least privilege akun ingest**
- **Demo:** `\dp logs.fluentbit` dan `\dn+ logs`.
- **Jelaskan:** Akun ini punya `USAGE, CREATE` pada schema `logs`, serta `SELECT, INSERT` pada `logs.fluentbit`. Tidak ada `UPDATE`, `DELETE`, atau hak superuser. `CREATE` diperlukan karena plugin bisa membuat tabel sendiri, dan itu sebaiknya dicabut di produksi.
- **Lanjutan:** Jika `fluentbit_user` bocor, penyerang bisa menyisipkan log palsu, membaca log, dan membuat objek di schema `logs`, tetapi tidak bisa mengubah atau menghapus log lama. Jika admin bocor, ia bisa menghapus bukti, mengubah role, dan mengakses seluruh database.

**5. Buffering saat backend mati**
- **Demo:** `./scripts/test-buffering.sh`, lalu `docker compose logs --tail 200 fluent-bit`.
- **Jelaskan:** Terlihat error flush dan pesan retry dari output `pgsql`, tetapi container Fluent Bit tetap hidup. Yang bekerja adalah buffer filesystem (`storage.type filesystem`, volume `flb-storage`) yang menahan chunk sampai flush berhasil. Batasnya: `Mem_Buf_Limit 10M` pada input dan `storage.total_limit_size 256M` pada output.
- **Lanjutan:** Kegagalan output memang diharapkan saat backend mati. PASS berarti kegagalan terlihat, collector tetap jalan, dan jumlah record bertambah setelah PostgreSQL pulih. Tidak adanya error justru mencurigakan karena bisa berarti kegagalan tidak terdeteksi.

**6. Monitoring collector**
- **Demo:** `curl -fsS .../api/v1/health` dan `.../api/v1/metrics`.
- **Jelaskan:** `health` hanya menunjukkan proses hidup. `metrics` berisi per plugin: `records` dan `bytes` yang diterima input, `proc_records` yang berhasil diproses output, serta `errors`, `retries`, `retries_failed`, dan `dropped_records`.
- **Lanjutan:** Pertama saya lihat apakah `records` di input masih naik, untuk memastikan data sampai ke Fluent Bit. Setelah itu saya periksa `errors`, `retries`, dan `dropped_records` di output pgsql.

**7. Rekonsiliasi kelengkapan event**
- **Demo:** `SELECT tag, count(*) FROM logs.fluentbit GROUP BY tag;` (filter rentang waktu run ini), bandingkan dengan 20 baris per generator dan angka input di metrics.
- **Jelaskan:** Selisih bisa disebabkan oleh:
  - run lama yang masih ada di tabel;
  - duplikasi akibat retry (lebih banyak);
  - event hilang karena collector belum siap, buffer driver penuh, atau buffer Fluent Bit penuh (lebih sedikit).
- **Lanjutan:** Deteksi dengan `GROUP BY service, sequence HAVING count(*)>1`. Pencegahan dilakukan dengan event ID unik dan deduplikasi (unique index atau dedup saat query), karena pipeline ini at-least-once, bukan exactly-once.

**8. Redaction dengan canary**
- **Demo:** tambahkan field `"api_token":"CANARY-XYZ-123"` ke satu generator. Pasang filter di Fluent Bit sebelum output (misalnya `lua` dengan `gsub`, atau parser lalu `modify`). Buktikan dengan `SELECT count(*) FROM logs.fluentbit WHERE data::text ILIKE '%CANARY%';`, hasilnya harus 0.
- **Jelaskan:** Nilai berada di dalam string `log`, jadi filter harus mengganti isi string, atau JSON di-parse dulu.
- **Lanjutan:** Canary masih bisa bocor ke: output `stdout` Fluent Bit (`docker compose logs`), buffer filesystem sebelum filter, buffer Docker driver, `docker inspect` (env), shell history, backup dan evidence export.

**9. Kebijakan retensi**
- **Demo:** tunjukkan `DELETE FROM logs.fluentbit WHERE time < CURRENT_TIMESTAMP - INTERVAL '7 days';` (yang dikomentari). Jalankan dulu `SELECT count(*)` untuk preview.
- **Jelaskan:** Rancangan kebijakannya:
  1. Preview jumlah baris.
  2. Jadwalkan harian (cron atau pg_cron).
  3. Hapus per batch agar tidak mengunci tabel lama.
  4. Idealnya partisi harian, lalu `DROP` partisi tertua.
  5. Catat jumlah baris yang dihapus.
- **Lanjutan:** Tipe `TIMESTAMP WITHOUT TIME ZONE` tidak menyimpan zona waktu. Cutoff retensi bisa bergeser jika timezone server atau sesi berbeda, dan korelasi dengan Prometheus (epoch UTC) atau Grafana (timezone browser) bisa meleset. Solusinya `timestamptz` dan UTC konsisten.

**10. Ekspor log sebagai evidence**
- **Demo:** `\copy (SELECT ... WHERE time BETWEEN ... ) TO STDOUT WITH CSV HEADER` lewat `docker compose exec -T`, diarahkan ke file di host, lalu `sha256sum file.csv`. (Jika memakai `TO '/path'`, file tertulis di dalam container.)
- **Jelaskan:** Ekspor harus menyertakan rentang waktu, query yang dipakai, zona waktu, jumlah baris, siapa dan kapan mengekspor, versi konfigurasi, serta checksum yang disimpan terpisah. Tidak boleh ada secret.
- **Lanjutan:** PostgreSQL tidak cocok lagi saat volume dan laju ingest sangat besar, retensi panjang menjadi mahal, atau butuh pencarian teks penuh dan skala horizontal. Kandidat penggantinya: Loki, OpenSearch, ClickHouse.

---

## Bab 7 — Monitoring

**11. Arsitektur dan exposure**
- **Demo:** `docker compose ps` dan `ss -lntp`.
- **Jelaskan:** Hanya `127.0.0.1:9090` (Prometheus) dan `127.0.0.1:3000` (Grafana) yang terbuka ke host. `node-exporter:9100` dan `cadvisor:8080` tidak dipublikasikan. Prometheus menjangkaunya lewat DNS service di network `bab7-monitoring-net`.
- **Lanjutan:** cAdvisor membaca cgroup, `/sys`, `/var/lib/docker`, dan metadata runtime, sehingga butuh akses luas. Risikonya: `privileged` hampir setara root di host. Jangan dipakai di mesin produksi, multi-tenant, atau laptop berisi data sensitif.

**12. Validasi rule**
- **Demo:** `promtool check config` dan `promtool test rules` dengan `--entrypoint /bin/promtool`.
- **Jelaskan:** `check config` membuktikan sintaks `prometheus.yml` dan rule file valid. `test rules` membuktikan logika alert benar untuk data sintetis. Keduanya tidak membuktikan target nyata terjangkau atau notifikasi sampai.
- **Lanjutan:** Dengan `eval_time: 1m`, test gagal. `up` bernilai `0` mulai menit ke-1, sedangkan `for: 1m` baru terpenuhi di menit ke-2. Pada 1m alert baru `pending`, dan test hanya mencocokkan alert yang `firing`.

**13. Status target dan makna `up`**
- **Demo:** halaman Targets atau `curl .../api/v1/targets`, lalu `./scripts/verify-stack.sh`.
- **Jelaskan:** `up = 1` berarti scrape terakhir berhasil (endpoint merespons dan dapat di-parse). Itu tidak membuktikan aplikasi sehat secara bisnis.
- **Lanjutan:** Contoh: aplikasi mengembalikan 500 atau sangat lambat sementara exporternya tetap hidup. Metric yang menunjukkannya adalah error rate dan latency request (misalnya `http_requests_total{status=~"5.."}` dan histogram durasi).

**14. Counter dan `rate()`**
- **Demo:** query `node_cpu_seconds_total` mentah, lalu `rate(node_cpu_seconds_total[5m])`.
- **Jelaskan:** Counter mentah hanya naik terus (total detik CPU kumulatif), jadi grafiknya tangga naik yang tidak informatif. `rate()` mengubahnya menjadi laju per detik, yaitu berapa core yang sedang dipakai.
- **Lanjutan:** `rate()` butuh minimal dua sampel di dalam jendela. Sesaat setelah start belum cukup sampel (interval scrape 15 detik), jadi hasilnya kosong. Tunggu beberapa scrape.

**15. Recording rule CPU host**
- **Demo:** tunjukkan rule `instance:node_cpu_utilisation:ratio_rate5m` (`1 - avg by(instance)(rate(idle[5m]))`), lalu query manual `100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100`.
- **Jelaskan:** Maknanya sama, tetapi rule berskala rasio (0–1) dan manual berskala persen. Kalikan rule dengan 100 sebelum membandingkan.
- **Lanjutan:** Selisih kecil bisa karena waktu evaluasi berbeda (rule dievaluasi tiap 15 detik, query manual instan), `by (instance)` vs rata-rata keseluruhan, atau timing scrape.

**16. Load test dan denominator CPU**
- **Demo:** `docker compose --profile load up -d load-generator`, amati 2 menit, lalu hentikan.
- **Jelaskan:** Container naik mendekati 0,5 core (sesuai `cpus: 0.50`). CPU host dalam persen kecil karena denominatornya semua core. Misalnya 0,5 dari 4 core ≈ 12,5%. Di Docker Desktop, host yang diamati adalah VM WSL 2. Alert `ContainerHighCPU` (>0,3 core selama 1 menit) akan firing.
- **Lanjutan:** Buktikan dengan uji on/off terkontrol: kenaikan CPU host berawal dan berakhir bersamaan dengan generator, besarnya sebanding dengan CPU container, dan tidak ada container lain di `topk` yang naik. Ini korelasi yang kuat, tetapi tetap bukan bukti kausal murni.

**17. Negative test `TargetDown`**
- **Demo:** `docker compose stop node-exporter`, cek `ALERTS{alertname="TargetDown"}` sebelum dan sesudah 75 detik, lalu `start`.
- **Jelaskan:** Sebelumnya tidak ada series. Setelah stop, `up=0` lalu alert `pending`, dan setelah `for: 1m` menjadi `firing`. Setelah dinyalakan kembali, `up=1` dan alert hilang.
- **Lanjutan:** Yang belum terbukti: pengiriman notifikasi (stack ini tidak punya Alertmanager), routing, silence, respons on-call, dan kasus series yang hilang total (butuh `absent()`).

**18. Provisioning dan akses Grafana**
- **Demo:** tunjukkan `datasources.yml` dan `provider.yml`, lalu coba edit panel di UI.
- **Jelaskan:** Dashboard dan datasource di-provision dari file (`editable: false`, `allowUiUpdates: false`), jadi menyimpan perubahan lewat UI ditolak. Perubahan harus lewat file JSON dan version control. Akses: login admin dari `.env`, anonymous dan sign-up dimatikan, hanya `127.0.0.1:3000`.
- **Lanjutan:** Urutan diagnosis:
  1. Periksa rentang waktu.
  2. Periksa UID datasource (`prometheus-bab7`).
  3. Jalankan query di Grafana Explore.
  4. Jalankan query yang sama di Prometheus.
  5. Periksa apakah recording rule sehat dan target `up`.
  6. Periksa log Grafana.

**19. Cardinality dan TSDB**
- **Demo:** `curl .../api/v1/status/tsdb` atau Status → TSDB, lalu tunjukkan metric dengan series terbanyak.
- **Jelaskan:** Biasanya metric `container_*` dari cAdvisor yang teratas, karena dikali jumlah container dan label.
- **Lanjutan:** Pertahankan label yang stabil seperti `name`. Buang label yang berubah tiap container dibuat ulang, seperti `id` (path cgroup), label image, dan `container_label_*` yang ribut, lewat `metric_relabel_configs`.

**20. Integrasi log Bab 6 ke Grafana**
- **Demo:** `./scripts/connect-bab6.sh`, lalu query `logs.fluentbit` dengan `$__timeFilter(time)` di Explore.
- **Jelaskan:** Skrip:
  1. Memuat `.env` Bab 6 dan 7.
  2. Mencari container PostgreSQL Bab 6.
  3. Menyambungkannya ke `bab7-monitoring-net` dengan alias `postgres-db`.
  4. Membuat role read-only (`SELECT` pada schema `logs`).
  5. Me-restart Grafana.
  
  Kesimpulan yang sah: kenaikan CPU dan log berada pada rentang waktu yang sama. Itu korelasi, bukan bukti sebab-akibat.
- **Lanjutan:** "Network Bab 7 belum tersedia" artinya network `bab7-monitoring-net` belum ada karena stack Bab 7 belum jalan. Jalankan `docker compose up -d` di folder Bab 7, cek dengan `docker network ls | grep bab7`, lalu ulangi skrip.

---

## Studi Kasus Portal PMB (soal 21–25)

**21. Tahap A — Baseline**
- **Demo:**
  1. `./scripts/set-incident.sh none`
  2. `./scripts/demo-normal.sh` (mencetak `request_id`). Atau kirim lewat browser dan ambil dari header `X-Request-ID`.
  3. Isi `investigated_request_id` di `queries/incident-analysis.sql`, lalu jalankan.
  4. Cek DB: `SELECT public_id, status, request_id FROM registrations ...`, harus `VERIFIED`.
- **Jelaskan:** Urutannya `login_success → registration_submitted → verification_started → verification_completed`.
- **Lanjutan:** Ada tiga waktu berbeda:
  - **Event time:** field `timestamp` di dalam JSON, dibuat aplikasi saat peristiwa terjadi.
  - **Waktu collector:** `time` yang dicap Docker driver atau Fluent Bit saat record diterima.
  - **Waktu simpan di database:** tabel ini tidak punya kolom sendiri untuk itu, dan `time` yang tersimpan adalah waktu record.
  
  Untuk kronologi saya pakai `timestamp` event, karena tidak terpengaruh buffering atau backlog. `time` dipakai sebagai pembanding.

**22. Tahap B — Failure injection**
- **Demo:** `./scripts/set-incident.sh timeout`, kirim pendaftaran, lalu jalankan query kronologi dan query "WARN dan ERROR per service".
- **Jelaskan:** Service pertama yang menghasilkan `WARN` adalah `verification-worker` (`verification_retry`, tiga kali). Setelah itu muncul `ERROR` `dependency_timeout` dengan kode `VERIFY-TIMEOUT`. Dampaknya di DB: status `FAILED`, `attempt=3`, `failure_code=VERIFY-TIMEOUT`.
- **Lanjutan:** `Up` hanya berarti proses hidup, tidak berarti transaksi berhasil. Worker bisa hidup sambil gagal memproses, jadi yang membuktikannya adalah event dan status transaksi.

**23. Tahap C — Recovery**
- **Demo:** `./scripts/set-incident.sh none`, kirim transaksi baru, lalu tampilkan `registrations`.
- **Jelaskan:** Transaksi baru `VERIFIED`, yang lama tetap `FAILED`. Worker hanya mengambil baris berstatus `SUBMITTED` (`claim_registration`), sehingga `FAILED` tidak pernah diproses ulang. Usulan perbaikan: retry terkontrol dengan batas percobaan dan exponential backoff, plus dead-letter queue untuk yang tetap gagal, dan requeue manual yang tercatat.
- **Lanjutan:** Tanpa idempotensi, pemrosesan ulang otomatis bisa menghasilkan efek ganda (verifikasi ganda, notifikasi ganda) dan retry storm ke dependency yang sedang bermasalah. Perlu idempotency key (`public_id`/`request_id`) dan state machine yang jelas.

**24. Tahap D — Resilience logging**
- **Demo:**
  1. Catat `SELECT count(*) FROM logs.fluentbit;`
  2. `docker compose stop log-db`
  3. Kirim transaksi baru.
  4. `docker compose logs --tail 100 fluent-bit` (retry).
  5. `docker compose start log-db`
  6. Hitung ulang, jumlah harus naik.
  
  Deteksi duplikat:
  ```sql
  SELECT (data->>'log')::jsonb->>'request_id' AS rid,
         (data->>'log')::jsonb->>'event'      AS ev,
         (data->>'log')::jsonb->>'attempt'    AS att,
         count(*)
  FROM logs.fluentbit
  WHERE tag LIKE 'pmb.%' AND left(ltrim(data->>'log'),1)='{'
  GROUP BY 1,2,3 HAVING count(*) > 1;
  ```
- **Jelaskan:** Backlog ditahan di buffer filesystem Fluent Bit dan terkirim setelah `log-db` pulih. Ini belum membuktikan exactly-once, karena retry bisa menduplikasi dan buffer penuh bisa menghilangkan data.
- **Lanjutan:** Saat output mencapai `storage.total_limit_size` 256 MB, Fluent Bit mulai membuang chunk (yang tertua) sehingga log hilang, dan `dropped_records` naik. Mitigasinya: alert pemakaian buffer, disk lebih besar, dan pemulihan backend lebih cepat.

**25. Privasi, kepercayaan, kesiapan produksi**
- **Demo:**
  - Jalankan `./scripts/verify.sh`, atau query "pemeriksaan secret" (hasil harus 0 baris).
  - Tunjukkan `reports/config-sha256.txt` dari `./scripts/collect-evidence.sh`.
  - Untuk `request_id` dari client, kirim `curl -i -H 'X-Request-ID: bad id' http://127.0.0.1:8080/login` dan satu lagi dengan ID valid, lalu bandingkan header respons.
- **Jelaskan:**
  - Nama dan email tidak masuk log karena log tersimpan lama, diakses lebih banyak orang, dan sulit dihapus (prinsip minimisasi data). Log memakai pengenal pseudonim (`user_ref`, `registration_id`).
  - `request_id` dari client divalidasi regex `^[A-Za-z0-9][A-Za-z0-9._:-]{7,127}$`. Jika tidak cocok, server membuat `REQ-<uuid>` baru. Ini mencegah log injection dan nilai terlalu panjang, tetapi nilainya tetap tidak dipercaya untuk otorisasi atau keunikan.
- **Lanjutan:** Kontrol yang perlu ditambahkan sebelum produksi:
  1. TLS untuk Forward, PostgreSQL, dan HTTP.
  2. Secret manager dengan rotasi credential.
  3. Autentikasi pada Prometheus, Grafana, dan endpoint Fluent Bit (`:2020`).
  4. Alertmanager dan on-call.
  5. Retensi dan kontrol akses log, plus integritas log (append-only).
  6. HA dan backup.
  7. Hak DDL dicabut dari akun ingest.
  8. Hilangkan `privileged`.
  9. Secure cookie dan HTTPS.
  10. Pin dan scan image.
