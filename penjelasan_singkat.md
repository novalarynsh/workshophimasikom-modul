# IoT Room Monitoring System
Sistem monitoring suhu dan kelembaban ruangan secara realtime berbasis **ESP32** dan **DHT11**, terintegrasi dengan **n8n**, **Google Sheets**, **Telegram Bot**, dan **AI Agent**.

---

## Fitur Utama
- Monitoring suhu & kelembaban secara realtime
- Penyimpanan data otomatis ke Google Sheets
- Notifikasi otomatis via Telegram
- Analisis kondisi ruangan menggunakan AI Agent (Groq)
- Rekap data historis dengan export ke Excel (.xls)
- Deteksi kegagalan pembacaan sensor

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
#include <Arduino.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <WiFiClientSecure.h>
#include <DHT.h>

const char* SSID       = "SSID WIFI KAMU";
const char* PASS       = "PASS WIFI KAMU";
const char* webhookURL = "URL WEBHOOK N8N KAMU";

DHT dht(4, DHT11);
unsigned long lastSend = 0;
const unsigned long INTERVAL = 10000; // bebas disesuaikan, satuan milidetik (10000 = 10 detik, 60000 = 1 menit)
```

Data dikirim ke n8n via HTTP POST dalam format: `suhu,kelembaban`  
Contoh: `29,68`

---

## Konfigurasi n8n

**Webhook:** `POST /webhook/sensor`

**Parsing data (Code JS Node):**
```js
const body = $input.first().json.body ?? '';
const [suhu, kelembaban] = body.split(',').map(Number);
const waktu = new Date().toLocaleString('id-ID', { timeZone: 'Asia/Jakarta' });

return { suhu, kelembaban, waktu, data: !isNaN(suhu) && !isNaN(kelembaban) };
```

**Filter rekap (Code JS Node):**
```js
const pesan      = $('pesan telegram').first().json.message.text ?? '';
const angkaMatch = pesan.match(/\d+/);
const angka      = angkaMatch ? parseInt(angkaMatch[0]) : null;

const satuan     = pesan.includes('detik') ? 1/60
                 : pesan.includes('jam')   ? 60
                 : pesan.includes('hari')  ? 1440
                 : pesan.includes('bulan') ? 43200
                 : 1;

const sekarang = $('pesan telegram').first().json.message.date * 1000 + 25200000;
const batas    = angka !== null ? sekarang - angka * satuan * 60000 : 0;

return $input.all()
  .filter(r => {
    const [tgl, wkt] = (r.json['waktu'] ?? '').split(',');
    const [d, m, y]  = tgl.trim().split('/');
    const [h, mi, s] = wkt.trim().split('.');
    return new Date(+y, +m-1, +d, +h, +mi, +s).getTime() >= batas;
  })
  .map(({ json: { row_number, ...data } }) => ({ json: data }));
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
| Perintah        | Fungsi                        |
|-----------------|-------------------------------|
| Chat bebas      | Tanya kondisi ruangan terkini |
| `/rekap`        | Rekap seluruh data, dikirim sebagai file `.xls`      |
| `/rekap 60`     | Rekap 60 menit terakhir, dikirim sebagai file `.xls` |
| `/rekap 24 jam` | Rekap 24 jam terakhir, dikirim sebagai file `.xls`   |

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
2. **Buat Workflow n8n** — Buat workflow baru dan tambahkan node sesuai panduan di README
3. **Konfigurasi Credential:**
   - Telegram Bot Token
   - Connect Akun Google dengan n8n
   - Groq API Key (model bebas)
4. **Aktifkan Workflow** — Set status dari `Inactive` → `Active`
5. **Buka Telegram** dan tunggu notifikasi otomatis

---

## Tech Stack
`ESP32` · `DHT11` · `n8n` · `Google Sheets` · `Telegram Bot` · `Groq AI (model bebas)`
