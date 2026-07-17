# Glossary — Istilah & Definisi Bersama

> Referensi istilah yang digunakan lintas dokumen dan bidang. Menjaga konsistensi pemahaman antar tim (dan AI agent).

| Istilah | Definisi |
|---------|----------|
| **Closed-loop** | Sistem di mana output sensor memicu keputusan yang langsung menggerakkan aktuator secara otomatis, tanpa intervensi manual. Alur: sensor → AI/logika → aktuator. |
| **Pinch-valve** | Mekanisme katup di mana servo menjepit/melepas selang lunak untuk mengontrol aliran air (prinsip seperti klem infus). Aktuator irigasi pada proyek ini. |
| **Louver** | Mekanisme bilah/flap yang dibuka-tutup servo untuk ventilasi, meniru jendela greenhouse. Aktuator ventilasi pada proyek ini. |
| **Fallback lokal** | Aturan threshold sederhana yang ditanam di firmware ESP32, dijalankan mandiri saat koneksi ke broker/server terputus, agar tanaman tetap terlindungi. |
| **health_score** | Skor kesehatan lahan 0–100 yang dihitung backend dari gabungan output model AI (kematangan + penyakit). Ditampilkan di listing marketplace. |
| **ripeness_class** | Kelas kematangan hasil model vision: `unripe`, `ripe`, `overripe`. |
| **disease_class** | Kelas kesehatan/penyakit daun hasil model vision: `healthy`, `<nama_penyakit>`, atau `null`. |
| **MQTT** | Protokol pesan ringan publish-subscribe untuk komunikasi IoT. Broker menjadi perantara antara ESP32 dan backend. |
| **Broker** | Server perantara MQTT (HiveMQ Cloud/Mosquitto) yang meneruskan pesan antar publisher & subscriber. |
| **device_id** | ID unik tiap perangkat fisik. Proyek ini punya 2: ESP32 utama & ESP32-CAM. |
| **plot** | Satu unit petak tanam dalam greenhouse, terhubung ke satu perangkat IoT. |
| **listing** | Entri produk hasil panen yang dijual petani di marketplace. |
| **Transfer learning** | Teknik melatih model AI dengan memanfaatkan model pretrained (mis. MobileNetV2) sebagai basis, lalu fine-tuning pada dataset spesifik. |
| **Regresi** | Model AI yang memprediksi nilai kontinu (di sini: durasi/volume kebutuhan irigasi). |
| **Klasifikasi** | Model AI yang memprediksi kategori (di sini: kelas kematangan/penyakit). |
| **Firestore** | Database NoSQL dokumen dari Firebase, menyimpan seluruh data terstruktur proyek. |
| **Cloudinary** | Layanan object/file storage untuk citra tanaman & foto produk. |
| **FCM** | Firebase Cloud Messaging — layanan push notification. |
| **Railway** | Platform PaaS tempat backend FastAPI di-deploy. |
| **Node-RED** | Tool dashboard flow-based untuk visualisasi data IoT & ekspor CSV. |
| **Override manual** | Tindakan petani menyalakan/mematikan aktuator secara manual dari app, membypass keputusan otomatis AI. |
