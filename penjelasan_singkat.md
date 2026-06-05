# IoT Room Monitoring System

Sistem monitoring suhu dan kelembaban ruangan secara realtime berbasis **ESP32** dan **DHT11**, terintegrasi dengan **n8n**, **Google Sheets**, **Telegram Bot**, dan **AI Agent**.

---

## Fitur Utama

- Monitoring suhu & kelembaban secara realtime
- Penyimpanan data otomatis ke Google Sheets
- Notifikasi otomatis via Telegram
- Analisis kondisi ruangan menggunakan AI Agent (Llama 3.3 70B)
- Rekap data historis dengan export ke Excel (.xls)
- Deteksi kegagalan pembacaan sensor

---

## Arsitektur Sistem

```
DHT11 → ESP32 → Webhook (n8n) → Validasi → AI Agent → Google Sheets
                                                      → Telegram
```

---

## Hardware

| Komponen     | Jumlah     |
|--------------|------------|
| ESP32        | 1          |
| DHT11        | 1          |
| Jumper Wire  | Secukupnya |

### Wiring

```
DHT11 VCC  → ESP32 3.3V
DHT11 DATA → ESP32 GPIO4
DHT11 GND  → ESP32 GND
```

---

## Konfigurasi ESP32

```cpp
#include <DHT.h>
#define DHTPIN 4
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

const char* SSID = "YOUR_WIFI";
const char* PASS = "YOUR_PASSWORD";
const char* webhookURL = "https://domain.com/webhook/sensor";
```

Data dikirim ke n8n via HTTP POST dalam format: `suhu,kelembaban`
Contoh: `29,68`

---

## Konfigurasi n8n

**Webhook:** `POST /webhook/sensor`

**Parsing data (Code JS Node):**
```js
const [suhu, kelembaban] = (($input.first().json.body ?? '').split(',').map(Number));
return { suhu, kelembaban, ok: !isNaN(suhu) && !isNaN(kelembaban) };
```

**Alur Workflow Sensor:**
- Data valid → AI Agent → Google Sheets → Telegram
- Data tidak valid → Alert ke Telegram (`Sensor tidak terdeteksi`)

**Alur Workflow Telegram Bot:**
- `/rekap` → ambil data Google Sheets → filter → export Excel → kirim ke Telegram
- Chat biasa → AI Agent → Google Sheets → Telegram

---

## Kategori Kondisi Ruangan

| Suhu       | Status  | Kelembaban | Status  |
|------------|---------|------------|---------|
| < 20°C     | Dingin  | < 40%      | Kering  |
| 20 – 28°C  | Nyaman  | 40 – 70%   | Nyaman  |
| 28 – 33°C  | Hangat  | > 70%      | Lembab  |
| > 33°C     | Panas   |            |         |

---

## Perintah Telegram Bot

| Perintah       | Fungsi                        |
|----------------|-------------------------------|
| Chat bebas     | Tanya kondisi ruangan terkini |
| `/rekap`       | Rekap 24 jam terakhir         |
| `/rekap 60`    | Rekap 60 menit terakhir       |
| `/rekap 24 jam`| Rekap 24 jam terakhir         |

**Contoh output:**
```
Waktu      : 08:00
Suhu       : 29°C
Kelembaban : 68%
Ruangan terasa nyaman.
```

---

## Cara Menjalankan

1. **Upload ke ESP32** — Buka Arduino IDE, pilih board ESP32, upload `workshop.ino`
2. **Import Workflow n8n** — Import from JSON → pilih `My Workflow.json`
3. **Konfigurasi Credential:**
   - Telegram Bot Token
   - Connect Akun Google dengan n8n
   - Groq API Key
4. **Aktifkan Workflow** — Set status dari `Inactive` → `Active`
5. **Buka Telegram** dan tunggu notifikasi otomatis

---

## Tech Stack

`ESP32` · `DHT11` · `n8n` · `Google Sheets` · `Telegram Bot` · `Groq AI (Llama 3.3 70B)`
