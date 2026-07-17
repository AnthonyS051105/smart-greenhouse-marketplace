# Product Requirements Document (PRD) — Smart Greenhouse + Marketplace

> **Dokumen Induk Bersama** — Di-referensikan oleh ketiga repo (IoT, AI, Mobile).
> Final Project Bootcamp TETI 2026 — AI × IoT × Mobile Development.
> Tema: Agriculture Application.

---

## 1. Ringkasan Produk

**Smart Greenhouse + Marketplace** adalah sistem terintegrasi yang menggabungkan:
1. **Monitoring & otomasi greenhouse berbasis IoT** — sensor lingkungan + kamera memantau kondisi tanaman, dan aktuator (servo) merespons secara otomatis (closed-loop).
2. **Kecerdasan buatan (AI)** — dua model: (a) regresi kebutuhan irigasi dari data cuaca mikro, dan (b) computer vision untuk menilai kematangan/kesehatan tanaman dari citra.
3. **Marketplace hasil panen** — petani menjual hasil panen langsung ke pembeli, dengan "skor kesehatan lahan" dari AI sebagai nilai jual transparansi produk.

**Tanaman fokus (default, dapat diganti):** Cabai — dipilih karena umum di Indonesia, memiliki fase kematangan visual yang jelas (hijau → merah), dan dataset penyakit tersedia publik (PlantVillage).

---

## 2. Latar Belakang & Masalah

### 2.1 Masalah yang Diselesaikan
- **Untuk petani:** Pengelolaan greenhouse manual rentan tidak optimal (penyiraman berlebih/kurang, telat mendeteksi kondisi buruk). Selain itu, petani sering kesulitan mengakses pasar dengan harga adil.
- **Untuk pembeli:** Sulit menilai kualitas hasil panen dari jarak jauh — ada asimetri informasi antara penjual dan pembeli soal kondisi/kualitas produk.

### 2.2 Solusi
Sistem otomatis yang menjaga kondisi greenhouse optimal, menilai kualitas panen secara objektif via AI, dan menghubungkan petani langsung ke pembeli dengan transparansi data kualitas.

---

## 3. Tujuan & Sasaran

### 3.1 Tujuan Produk
| ID | Tujuan |
|----|--------|
| G-01 | Mengotomasi irigasi & ventilasi greenhouse berdasarkan keputusan AI (closed-loop). |
| G-02 | Menilai kematangan & kesehatan tanaman secara objektif dari citra. |
| G-03 | Menyediakan marketplace dengan transparansi kualitas berbasis data sensor & AI. |
| G-04 | Menyediakan monitoring real-time bagi petani dan kontrol manual override. |

### 3.2 Sasaran Keberhasilan (untuk konteks kompetisi)
- Sistem closed-loop berfungsi end-to-end saat demo (sensor → AI → aktuator).
- Kedua model AI dilatih sendiri oleh tim dengan evaluasi metrik yang jelas.
- Aplikasi mobile dua sisi (petani & pembeli) berfungsi dengan alur lengkap.
- Memenuhi seluruh syarat wajib rubrik ketiga bootcamp (lihat Bagian 9).

---

## 4. Persona Pengguna

### 4.1 Petani (Farmer)
- **Deskripsi:** Pemilik/pengelola greenhouse yang memantau kondisi tanaman dan menjual hasil panen.
- **Kebutuhan:** Monitoring real-time, kontrol manual aktuator, kemudahan membuat listing produk, penilaian kualitas otomatis.
- **Karakteristik:** Mungkin non-teknis; UI harus sederhana dan berbahasa Indonesia.

### 4.2 Pembeli (Buyer)
- **Deskripsi:** Konsumen/pedagang yang mencari hasil panen berkualitas.
- **Kebutuhan:** Pencarian mudah, kepercayaan pada kualitas produk (transparansi data), komunikasi langsung dengan petani, kemudahan transaksi.

---

## 5. Lingkup Produk (Scope)

### 5.1 Dalam Lingkup (In-Scope)
- Monitoring sensor lingkungan (suhu, kelembapan, tekanan) real-time.
- Pengambilan citra tanaman berkala via ESP32-CAM.
- Model AI regresi irigasi + model AI vision kematangan/kesehatan.
- Aktuator servo otomatis (irigasi pinch-valve + ventilasi louver) + fallback lokal.
- Aplikasi mobile petani (monitoring, kontrol, listing) & pembeli (marketplace, chat, checkout, rating).
- Fitur peta lokasi lahan (Google Maps).
- Dashboard IoT (Node-RED) + ekspor CSV data sensor.

### 5.2 Luar Lingkup (Out-of-Scope, untuk versi bootcamp)
- Payment gateway sungguhan (checkout disederhanakan tanpa pembayaran nyata).
- Sistem logistik/pengiriman nyata.
- Multi-bahasa (hanya Bahasa Indonesia).
- Skala produksi besar (sistem diposisikan sebagai prototipe/demo).

---

## 6. Fitur Utama (Ringkasan Lintas Bidang)

| ID | Fitur | IoT | AI | Mobile |
|----|-------|-----|----|----|
| F-01 | Pembacaan & pengiriman data sensor lingkungan | ✅ | | |
| F-02 | Pengambilan citra tanaman berkala | ✅ | ✅ | |
| F-03 | Prediksi kebutuhan irigasi (regresi) | | ✅ | |
| F-04 | Klasifikasi kematangan/kesehatan (vision) | | ✅ | |
| F-05 | Aktuasi otomatis irigasi & ventilasi (closed-loop) | ✅ | ✅ | |
| F-06 | Fallback lokal saat offline | ✅ | | |
| F-07 | Monitoring real-time di app | ✅ | | ✅ |
| F-08 | Kontrol manual override | ✅ | | ✅ |
| F-09 | Buat & kelola listing (auto-fill skor kesehatan) | | ✅ | ✅ |
| F-10 | Marketplace: cari, filter, detail produk | | | ✅ |
| F-11 | Chat negosiasi petani-pembeli | | | ✅ |
| F-12 | Checkout & rating | | | ✅ |
| F-13 | Peta lokasi lahan + komoditas per marker | | | ✅ |
| F-14 | Data logging & ekspor CSV | ✅ | | |
| F-15 | Notifikasi push (status otomatis, order, chat) | | | ✅ |

---

## 7. Alur Nilai Utama (Value Flow)

```
Sensor membaca kondisi greenhouse
        ↓
AI memutuskan kebutuhan air & menilai kualitas tanaman
        ↓
Sistem otomatis menyalakan irigasi/ventilasi (closed-loop)
        ↓
Skor kesehatan lahan terisi otomatis ke listing marketplace
        ↓
Pembeli melihat produk dengan transparansi kualitas → transaksi
```

---

## 8. Arsitektur Tingkat Tinggi

- **Backend:** FastAPI (Python), di-deploy ke **Railway**. Berperan sebagai jembatan MQTT + AI Inference + logika kontrol aktuator. **Tidak menyimpan data sendiri.**
- **Penyimpanan data:** **Firestore** (semua data terstruktur).
- **Autentikasi:** **Firebase Authentication**.
- **Notifikasi:** **Firebase Cloud Messaging (FCM)**.
- **Object/file storage:** **Cloudinary** (citra ESP-CAM & foto produk) — *bukan* Firebase Storage.
- **Konektivitas IoT:** **MQTT broker** (HiveMQ Cloud / Mosquitto).
- **Dashboard IoT:** **Node-RED**.
- **Mobile:** **Kotlin + Jetpack Compose**, Android.
- **Peta:** **Google Maps SDK for Android** + Places + Geocoding.

> Detail lengkap arsitektur ada di `shared/Architecture.md`.

---

## 9. Pemetaan ke Rubrik Penilaian (Ringkas)

### IoT (bobot terbesar: Arsitektur Sistem 25%)
- Closed-loop penuh (sensor → AI → aktuator) + fallback lokal.
- Servo actuation sebagai solusi kreatif murah (kriteria kreativitas tools 20%).

### AI (bobot terbesar: Pemahaman Masalah & Implementasi Teknikal masing-masing 20%)
- Dua sub-masalah AI jelas: regresi & klasifikasi.
- Model dilatih sendiri + evaluasi metrik (MSE/RMSE/R², Accuracy/F1).

### Mobile Dev (bobot terbesar: Fitur & Demo masing-masing 20%)
- Alur e-commerce lengkap + monitoring + peta.
- Multi tech stack: Firestore SDK + REST API + FCM + Maps.

---

## 10. Batasan & Asumsi

### 10.1 Batasan
- Timeline sangat ketat (~10–11 hari efektif per track, paralel).
- Tim 4 orang mengerjakan 3 deliverable berbeda.
- Anggaran minim (hardware murah, layanan free/low-tier).

### 10.2 Asumsi
- Demo dilakukan dengan 1 unit perangkat greenhouse.
- Koneksi internet tersedia saat demo (dengan fallback lokal sebagai cadangan).
- Jenis tanaman fokus: **cabai** (dapat diganti; memengaruhi dataset AI).

---

## 11. Dependensi Antar Bidang

| Dependensi | Dari | Ke | Deskripsi |
|------------|------|----|-----------|
| Format payload sensor | IoT | AI, Backend | IoT harus kirim JSON sesuai kontrak (lihat `shared/data-contracts.md`). |
| Endpoint AI | AI | Backend | Model di-load & di-serve via FastAPI. |
| Hasil AI (skor) | AI | Mobile | Skor kesehatan tampil di listing & detail. |
| Command aktuator | Backend | IoT | Backend publish command MQTT → ESP32. |
| Struktur Firestore | Backend | Mobile | App baca/tulis Firestore sesuai skema bersama. |

> Kontrak data teknis lengkap: `shared/data-contracts.md`.

---

## 12. Referensi Dokumen Terkait

- `shared/Architecture.md` — arsitektur sistem menyeluruh.
- `shared/data-contracts.md` — kontrak data lintas bidang (MQTT, Firestore, API).
- `shared/glossary.md` — istilah & definisi bersama.
- `iot/`, `ai/`, `mobile/` — dokumen spesifik per bidang (SRS, SDD, Task Breakdown, CLAUDE.md).
