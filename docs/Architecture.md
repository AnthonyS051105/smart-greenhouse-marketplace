# Architecture.md — Arsitektur Sistem Menyeluruh

> **Dokumen Induk Bersama** — Di-referensikan oleh ketiga repo (IoT, AI, Mobile).
> Menjelaskan arsitektur sistem end-to-end, keputusan teknis, dan interaksi antar komponen.

---

## 1. Diagram Arsitektur (Tekstual)

```
┌─────────────────────────────────────────────────────────────────────┐
│                       LAPISAN PERANGKAT (IoT)                        │
│                                                                       │
│  ┌──────────────────┐        ┌──────────────────┐                    │
│  │  ESP32 (utama)   │        │   ESP32-CAM      │                    │
│  │  DHT11, BME280,  │        │  (board terpisah)│                    │
│  │  MQ(opsional),   │        │  Ambil citra     │                    │
│  │  OLED, 2×Servo   │        │  tanaman berkala │                    │
│  └────────┬─────────┘        └────────┬─────────┘                    │
└───────────┼───────────────────────────┼─────────────────────────────┘
            │ MQTT (sensor/command/status)│ HTTP POST multipart (citra)
            ▼                             ▼
┌─────────────────────────┐   ┌──────────────────────────────────────┐
│   MQTT BROKER           │   │                                       │
│  (HiveMQ Cloud /        │   │                                       │
│   Mosquitto)            │   │                                       │
└───────────┬─────────────┘   │                                       │
            │                 │                                       │
            ▼                 ▼                                       │
┌─────────────────────────────────────────────────────────────────┐  │
│         BACKEND — FastAPI (di-deploy ke RAILWAY)                 │  │
│  • MQTT Subscriber (terima sensor, publish command)             │  │
│  • AI Inference Service (model regresi + vision di-load in-proc) │  │
│  • Logika keputusan aktuator (closed-loop)                      │  │
│  • Verifikasi Firebase ID token                                 │  │
│  • Upload citra → Cloudinary                                    │  │
│  • Tulis/baca data → Firestore                                  │  │
└───┬─────────────────┬───────────────────┬──────────────────────┘  │
    │                 │                   │                          │
    ▼                 ▼                   ▼                          │
┌─────────┐   ┌───────────────┐   ┌──────────────┐                  │
│FIRESTORE│   │  CLOUDINARY   │   │  NODE-RED     │◄─────────────────┘
│(semua   │   │ (citra & foto)│   │  Dashboard IoT│  (subscribe MQTT
│ data)   │   │               │   │  + CSV export)│   langsung)
└────┬────┘   └───────────────┘   └──────────────┘
     │
     │ Firebase SDK (Auth, Firestore, FCM)  +  REST API (Retrofit)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│              APLIKASI MOBILE (Android, Kotlin + Compose)         │
│  ┌────────────────────┐        ┌──────────────────────┐          │
│  │   APP PETANI       │        │    APP PEMBELI        │          │
│  │  Monitoring,       │        │  Marketplace, Chat,   │          │
│  │  Kontrol, Listing  │        │  Checkout, Rating,    │          │
│  │                    │        │  Peta (Google Maps)   │          │
│  └────────────────────┘        └──────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Keputusan Arsitektur Kunci (Architecture Decision Records)

### ADR-01: Backend FastAPI + Firebase (tanpa PostgreSQL)
- **Keputusan:** Backend FastAPI hanya sebagai jembatan MQTT + AI; seluruh penyimpanan data di Firestore.
- **Alasan:** Kecepatan prototyping untuk tim 4 orang dalam ~10 hari. Menghindari mengelola & sinkronisasi 2 database.
- **Konsekuensi:** Ekspor CSV & query relasional perlu kode manual (loop Firestore), bukan SQL. Diterima karena skala data kecil.

### ADR-02: Object storage = Cloudinary (bukan Firebase Storage)
- **Keputusan:** Gunakan Cloudinary untuk citra/foto.
- **Alasan:** Free tier 25 GB tanpa kartu kredit; Firebase Storage kini wajib akun Blaze (kartu kredit).
- **Konsekuensi:** Satu integrasi eksternal tambahan, tapi menghindari kewajiban kartu kredit.

### ADR-03: Deploy backend ke Railway
- **Keputusan:** Railway Hobby plan (~$5/bulan).
- **Alasan:** Perlu proses always-on (MQTT subscriber) yang tidak sleep — tier gratis Render/Railway lama tidak cocok.
- **Konsekuensi:** Biaya kecil; perlu waspada riwayat downtime Railway (siapkan fallback demo).

### ADR-04: Aktuator berbasis Servo
- **Keputusan:** Servo SG90 (pinch-valve irigasi + louver ventilasi), bukan pompa/relay.
- **Alasan:** Murah, wiring sederhana, aman (tidak kontak air), visual meyakinkan saat demo.
- **Konsekuensi:** Skala debit kecil (prototipe demo, bukan produksi).

### ADR-05: ESP32 & ESP32-CAM board terpisah
- **Keputusan:** 2 board — ESP32 utama (sensor+aktuator) & ESP32-CAM (citra).
- **Alasan:** ESP32-CAM punya keterbatasan pin GPIO (banyak dipakai kamera); memisah menghindari konflik pin dengan sensor & servo.
- **Konsekuensi:** 2 device_id berbeda; keduanya kirim ke broker/backend yang sama.

### ADR-06: Mobile = Kotlin + Jetpack Compose
- **Keputusan:** Kotlin dengan Jetpack Compose (declarative UI), bukan XML View tradisional.
- **Alasan:** Dikonfirmasi boleh oleh mentor saat coaching. Compose mempercepat pembangunan UI (lebih sedikit boilerplate), state management lebih natural dengan ViewModel, dan cocok untuk UI dinamis seperti grafik sensor real-time & daftar marketplace.
- **Konsekuensi:** Perlu pemahaman paradigma declarative & recomposition. Navigasi memakai Navigation Compose (bukan Fragment). Beberapa library (mis. Google Maps) memakai varian Compose (`maps-compose`).
- **Riwayat:** Keputusan awal adalah XML (mengikuti materi bootcamp); diubah setelah coaching mengonfirmasi Compose diperbolehkan.

---

## 3. Pembagian Jalur Komunikasi Mobile

| Operasi | Jalur | Alasan |
|---------|-------|--------|
| Login/register | Firebase Auth SDK langsung | Tidak perlu endpoint custom |
| CRUD listing, order, review | Firestore SDK langsung | Cepat, tanpa boilerplate backend |
| Chat | Firestore SDK (realtime listener) | Realtime bawaan |
| Notifikasi | FCM SDK | Standar Android |
| Upload foto produk | Cloudinary SDK/upload | Object storage |
| Peta & marker | Firestore query + Maps SDK | Data farm dari Firestore |
| **Kontrol aktuator manual** | **REST API → FastAPI** | Hanya backend punya koneksi MQTT |
| **Trigger inferensi AI** | **REST API → FastAPI** | Model di-load di backend |
| **Ekspor CSV** | **REST API → FastAPI** | Perlu proses agregasi |

**Prinsip:** Railway hanya menangani beban IoT+AI, bukan seluruh traffic aplikasi → biaya rendah & beban server minimal.

---

## 4. Alur Data Closed-Loop (Detail)

```
1. ESP32 baca sensor → publish MQTT `greenhouse/{id}/sensor`
2. Backend (subscriber) terima → simpan ke Firestore `sensor_readings`
3. Backend panggil model regresi → dapat rekomendasi durasi irigasi
4. Jika rekomendasi > ambang → backend publish MQTT `greenhouse/{id}/command`
5. ESP32 (subscriber command) → gerakkan servo pinch-valve
6. ESP32 publish `greenhouse/{id}/status` → backend simpan `irrigation_events`
7. Paralel: ESP32-CAM kirim citra → backend upload Cloudinary → model vision → simpan `crop_images`

FALLBACK: Jika ESP32 kehilangan koneksi broker > N detik,
jalankan threshold lokal (mis. suhu>32°C → buka ventilasi) mandiri.
```

---

## 5. Keamanan

- **Autentikasi:** Firebase Auth; backend verifikasi ID token via Firebase Admin SDK di setiap endpoint terproteksi.
- **Otorisasi:** Firestore Security Rules membatasi akses berdasarkan `role` & kepemilikan (petani hanya bisa ubah farm miliknya).
- **Kredensial:** Disimpan sebagai environment variables di Railway (bukan hardcode). Meliputi: `FIREBASE_CREDENTIALS`, `MQTT_BROKER_URL`, `CLOUDINARY_URL`.
- **Device:** MQTT broker memakai kredensial (username/password) agar tidak sembarang device bisa publish.

---

## 6. Teknologi per Bidang (Ringkas)

| Bidang | Bahasa/Framework | Kunci |
|--------|------------------|-------|
| IoT | C++ (Arduino/ESP-IDF) | ESP32, ESP32-CAM, PubSubClient (MQTT), ESP32Servo |
| AI | Python | scikit-learn (regresi), TensorFlow/Keras (vision), pandas, FastAPI (serving) |
| Backend | Python | FastAPI, paho-mqtt, firebase-admin, cloudinary |
| Mobile | Kotlin + Jetpack Compose | Retrofit, Firebase SDK, maps-compose, Coil, Navigation Compose |
| Dashboard | Node-RED | Flow-based, CSV export node |

---

## 7. Deployment & Environment

| Komponen | Lokasi Deploy |
|----------|---------------|
| Backend FastAPI | Railway (Hobby plan) |
| Firestore, Auth, FCM | Firebase (Spark plan) |
| Cloudinary | Cloudinary cloud (free tier) |
| MQTT Broker | HiveMQ Cloud (free) atau Mosquitto (self-host di Railway) |
| Node-RED | Lokal saat demo, atau Railway |
| Mobile APK | Build lokal, distribusi manual (APK) |

**Environment variables backend (Railway):**
```
FIREBASE_CREDENTIALS=<service-account-json>
MQTT_BROKER_URL=<url>
MQTT_USERNAME=<user>
MQTT_PASSWORD=<pass>
CLOUDINARY_URL=cloudinary://<key>:<secret>@<cloud>
```

---

## 8. Referensi

- `shared/PRD.md` — kebutuhan produk.
- `shared/data-contracts.md` — kontrak data teknis lintas bidang.
- `shared/glossary.md` — istilah.
- Dokumen SDD spesifik per bidang di `iot/`, `ai/`, `mobile/`.
