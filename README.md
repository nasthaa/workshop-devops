# Persiapan Ujian Praktik DevSecOps — Bab 6, Bab 7, PMB

Naskah demo untuk tiap paket soal: perintah yang dijalankan, screenshot dari lab sendiri dengan **kotak merah bernomor**, kalimat penjelasan untuk tiap nomor, dan jawaban pertanyaan lanjutan.

## Daftar paket

| Paket | Soal | File |
|---|---|---|
| A | 1, 5, 13, 17, 21 | [paket-A.md](paket-A.md) |
| B | 2, 6, 11, 16, 22 | [paket-B.md](paket-B.md) |
| C | 3, 8, 14, 18, 23 | [paket-C.md](paket-C.md) |
| D | 4, 9, 12, 15, 24 | [paket-D.md](paket-D.md) |
| E | 7, 10, 19, 20, 25 | [paket-E.md](paket-E.md) |

Versi PDF (termasuk buku saku dan catatan belajar) ada di folder [`pdf/`](pdf/).

## Cara membaca

- **A. Tunjukkan**: perintah yang dijalankan.
- **B. Jelaskan**: nomor **[1], [2], ...** sama dengan nomor kotak merah di gambar.
- **C. Jawab lanjutan**: jawaban pertanyaan susulan dosen.
- 📘 = lokasi perintah di cheatsheet / README, 🔑 = kata kunci, 🗣️ = contoh penjelasan, 💡 = catatan, ⚠️ = peringatan.

## Aturan lab saat ujian

- **Satu tab terminal untuk satu lab**: Tab 1 Bab 7, Tab 2 Bab 6, Tab 3 PMB.
- Bab 6 dan PMB **tidak bisa hidup bersamaan** (port 24224 dan 2020). Matikan Bab 6 dulu sebelum menyalakan PMB.
- Kalau nama container berawalan `pmb-lab-` padahal sedang di folder Bab 6, tab-nya salah: jalankan `unset COMPOSE_PROJECT_NAME` atau buka tab baru.
- Request ID di screenshot pasti berbeda dengan saat ujian. Pakai ID dari run saat itu.
