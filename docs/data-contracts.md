# Data Contracts — Kontrak Data Lintas Bidang

> **Dokumen Induk Bersama** — Sumber kebenaran tunggal (single source of truth) untuk format data yang mengalir antar bidang (IoT ↔ Backend ↔ AI ↔ Mobile).
> **PENTING:** Setiap perubahan pada dokumen ini harus disepakati seluruh tim karena berdampak lintas repo.

---

## 1. Topik & Payload MQTT (IoT ↔ Backend)

Broker: HiveMQ Cloud (atau Mosquitto). Format payload: **JSON**.

### 1.1 Topik: `greenhouse/{device_id}/sensor` (IoT → Backend)
Dikirim ESP32 setiap 5–10 menit.

```json
{
  "device_id": "gh-esp32-01",
  "timestamp": "2026-07-12T08:30:00Z",
  "temperature": 29.5,
  "humidity": 72.3,
  "soil_moisture": 45.0,
  "light_intensity": 850.0
}
```

| Field | Tipe | Satuan | Wajib | Keterangan |
|-------|------|--------|-------|------------|
| device_id | string | - | ✅ | ID unik perangkat ESP32 |
| timestamp | string (ISO 8601 UTC) | - | ✅ | Waktu pembacaan |
| temperature | float | °C | ✅ | Dari DHT11 |
| humidity | float | % | ✅ | Kelembapan udara, dari DHT11 |
| soil_moisture | float | % (0–100) | ✅ | Kelembapan tanah, dari Capacitive Soil Moisture Sensor v1.2 (dipetakan dari nilai ADC mentah ke persentase saat kalibrasi) |
| light_intensity | float | lux | ✅ | Intensitas cahaya, dari sensor BH1750 (I2C) |

### 1.2 Topik: `greenhouse/{device_id}/command` (Backend → IoT)
Dikirim backend saat AI/logika memutuskan aktuasi.

```json
{
  "command_id": "cmd-20260712-0830-01",
  "timestamp": "2026-07-12T08:30:05Z",
  "actuator": "irrigation",
  "action": "open",
  "duration_seconds": 180,
  "expires_at": "2026-07-12T08:35:05Z",
  "source": "ai_auto"
}
```

| Field | Tipe | Nilai | Keterangan |
|-------|------|-------|------------|
| command_id | string | - | ID unik command (untuk idempotensi) |
| actuator | string | `irrigation` \| `ventilation` | Target aktuator |
| action | string | `open` \| `close` | Aksi servo |
| duration_seconds | int | - | Durasi buka (0 untuk close) |
| expires_at | string (ISO 8601) | - | Command diabaikan jika diterima setelah waktu ini |
| source | string | `ai_auto` \| `manual` | Asal keputusan |

### 1.3 Topik: `greenhouse/{device_id}/status` (IoT → Backend)
Status aktuator & heartbeat, dikirim saat perubahan status atau tiap interval.

```json
{
  "device_id": "gh-esp32-01",
  "timestamp": "2026-07-12T08:30:10Z",
  "irrigation_state": "open",
  "ventilation_state": "closed",
  "wifi_rssi": -58,
  "mode": "online"
}
```

`mode`: `online` (terhubung server) | `fallback` (jalan mandiri karena offline).

---

## 2. Citra ESP-CAM (IoT → Backend → Cloudinary)

- ESP32-CAM mengambil gambar tiap 1–2 jam.
- Dikirim via **HTTP POST multipart** ke endpoint backend `POST /images` (bukan MQTT, karena payload besar).
- Backend meng-upload ke **Cloudinary**, menyimpan URL hasilnya ke Firestore.

**Request (multipart/form-data):**
```
device_id: gh-esp32cam-01
plot_id: plot-abc123
image: <binary jpeg>
```

**Response:**
```json
{
  "status": "ok",
  "image_id": "img-xyz789",
  "image_url": "https://res.cloudinary.com/.../crop.jpg"
}
```

---

## 3. Skema Firestore (Backend ↔ Mobile)

Seluruh koleksi bersifat **top-level** (bukan sub-collection) untuk kemudahan query lintas entitas. ID dokumen = ID entitas.

### 3.1 `users`
```json
{
  "uid": "firebase-auth-uid",
  "name": "Budi Tani",
  "email": "budi@example.com",
  "role": "farmer",
  "created_at": "2026-07-12T00:00:00Z"
}
```
`role`: `farmer` | `buyer`.

### 3.2 `farms`
```json
{
  "id": "farm-abc",
  "owner_uid": "firebase-auth-uid",
  "farm_name": "Greenhouse Cabai Sleman",
  "location_lat": -7.7250,
  "location_lng": 110.4100,
  "farm_size_m2": 120.0
}
```

### 3.3 `plots`
```json
{
  "id": "plot-abc123",
  "farm_id": "farm-abc",
  "crop_type": "cabai",
  "planting_date": "2026-06-01",
  "device_id": "gh-esp32-01"
}
```

### 3.4 `sensor_readings`
```json
{
  "id": "reading-...",
  "plot_id": "plot-abc123",
  "timestamp": "2026-07-12T08:30:00Z",
  "temperature": 29.5,
  "humidity": 72.3,
  "soil_moisture": 45.0,
  "light_intensity": 850.0
}
```

### 3.5 `irrigation_events`
```json
{
  "id": "event-...",
  "plot_id": "plot-abc123",
  "timestamp": "2026-07-12T08:30:05Z",
  "actuator": "irrigation",
  "duration_seconds": 180,
  "trigger_type": "auto",
  "ai_recommended_amount": 175.0
}
```
`trigger_type`: `auto` | `manual`.

### 3.6 `crop_images`
```json
{
  "id": "img-xyz789",
  "plot_id": "plot-abc123",
  "timestamp": "2026-07-12T09:00:00Z",
  "image_url": "https://res.cloudinary.com/.../crop.jpg",
  "ripeness_class": "ripe",
  "disease_class": "healthy",
  "confidence_score": 0.94
}
```
`ripeness_class`: `unripe` | `ripe` | `overripe`.
`disease_class`: `healthy` | `<nama_penyakit>` | `null`.

### 3.7 `listings`
```json
{
  "id": "listing-...",
  "farm_id": "farm-abc",
  "plot_id": "plot-abc123",
  "crop_type": "cabai",
  "quantity_kg": 15.0,
  "price_per_kg": 35000,
  "health_score": 88.5,
  "harvest_date": "2026-07-12",
  "status": "available",
  "created_at": "2026-07-12T10:00:00Z",
  "description": "Cabai rawit merah segar, dipanen pagi ini.",
  "pre_order_enabled": false
}
```
`status`: `available` | `sold`.
`health_score`: 0–100 (dihitung backend dari hasil AI, lihat Bagian 4).
`description` (string, opsional, default `""`): deskripsi bebas produk yang diisi petani di form Buat Listing. Field mobile-only — tidak dibaca/ditulis IoT maupun AI, ditambahkan 2026-07-19 atas permintaan Mobile Dev (form sudah ada duluan di `CreateListingModels.kt` sebelum field ini didokumentasikan di sini).
`pre_order_enabled` (boolean, opsional, default `false`): penanda apakah listing menerima pre-order. Field mobile-only, alur pre-order itu sendiri belum didefinisikan di `SRS.md`/`UIUX-Flow.md` manapun — disimpan agar tidak hilang saat form disambungkan ke Firestore, tapi belum ada kebutuhan fungsional yang memakainya.

### 3.8 `orders`
```json
{
  "id": "order-...",
  "buyer_uid": "firebase-auth-uid",
  "listing_id": "listing-...",
  "quantity_kg": 5.0,
  "total_price": 175000,
  "status": "pending",
  "created_at": "2026-07-12T11:00:00Z"
}
```
`status`: `pending` | `confirmed` | `completed` | `cancelled`.

### 3.9 `chat_messages`
```json
{
  "id": "msg-...",
  "sender_uid": "...",
  "receiver_uid": "...",
  "listing_id": "listing-...",
  "message": "Apakah bisa nego untuk 10kg?",
  "sent_at": "2026-07-12T11:05:00Z"
}
```

### 3.10 `reviews`
```json
{
  "id": "review-...",
  "order_id": "order-...",
  "rating": 5,
  "comment": "Cabai segar, sesuai deskripsi.",
  "created_at": "2026-07-13T08:00:00Z"
}
```

---

## 4. Kontrak Endpoint FastAPI (Mobile/IoT ↔ Backend)

Base URL: `https://<nama>.up.railway.app`. Semua endpoint (kecuali ingestion dari device) memerlukan header `Authorization: Bearer <firebase_id_token>`.

### 4.1 `POST /sensors` (IoT → Backend)
Menerima data sensor (alternatif MQTT untuk device yang HTTP-only). Body = payload sensor (Bagian 1.1).

### 4.2 `POST /images` (IoT → Backend)
Lihat Bagian 2.

### 4.3 `GET /plots/{plot_id}/irrigation/recommendation`
Response:
```json
{ "recommended_duration_sec": 180, "confidence": 0.87 }
```

### 4.4 `POST /irrigation/trigger` (Mobile → Backend)
Override manual. Body:
```json
{ "plot_id": "plot-abc123", "actuator": "irrigation", "action": "open", "duration_seconds": 120 }
```

### 4.5 `POST /vision/analyze` (Mobile/Backend → Backend)
Body: `{ "image_id": "img-xyz789" }`. Response:
```json
{ "ripeness_class": "ripe", "disease_class": "healthy", "confidence_score": 0.94 }
```

### 4.6 `POST /listings/auto-fill-health-score` (Mobile → Backend)
Body: `{ "plot_id": "plot-abc123" }`. Response:
```json
{ "health_score": 88.5, "source_image_id": "img-xyz789" }
```

### 4.7 `GET /export/csv?plot_id={id}` (Mobile/Web → Backend)
Response: file CSV data sensor (memenuhi syarat data logging IoT).

---

## 5. Perhitungan `health_score`

Backend menggabungkan output kedua model AI menjadi satu skor 0–100:

```
health_score = w1 * ripeness_component + w2 * disease_component

ripeness_component:
  ripe    → 100
  unripe  → 60
  overripe→ 40

disease_component:
  healthy → 100
  diseased→ 30

Bobot default: w1 = 0.4, w2 = 0.6
(dapat dikalibrasi tim; disepakati bersama)
```

> Formula ini didefinisikan di backend dan dikonsumsi Mobile sebagai angka jadi. Mobile TIDAK menghitung sendiri.

---

## 6. Aturan Konsistensi

1. **Timestamp** selalu ISO 8601 UTC (`Z`).
2. **ID** menggunakan format `<entitas>-<unik>` agar mudah di-debug.
3. **Enum** (status, role, class) harus persis sesuai dokumen ini — case-sensitive.
4. Perubahan kontrak → update dokumen ini + broadcast ke seluruh tim SEBELUM implementasi.
