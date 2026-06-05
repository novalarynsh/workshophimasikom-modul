# 🧪 Modul Materi — Workshop Himasikom
# Wireflow - Where IoT Meets Automation

## Setup Awal Sebelum Mulai
> 
> Install Arduino IDE di: https://www.arduino.cc/en/software/
> 
> Buat Akun pada Platform n8n: https://n8n.io/

---

## Compile ESP32 & DHT11
**Platform:** Arduino IDE   
**Tujuan:** Mengatur ESP32 sebagai alat pembaca suhu dan kelembaban ruangan yang otomatis menyetor datanya ke sistem n8n via internet.

### Konsep
Setelah terhubung ke WiFi, program akan terus-menerus membaca sensor DHT11 dan menembakkan hasilnya langsung ke alamat web n8n.

### Kode Demo

```ino
#include <Arduino.h>

#include <WiFi.h>
#include <HTTPClient.h>
#include <WiFiClientSecure.h>
#include <DHT.h>

const char* SSID    = "SSID WIFI KAMU";
const char* PASS    = "PASS WIFI KAMU";
const char* webhookURL = "URL WEBHOOK N8N KAMU";

DHT dht(4, DHT11);
unsigned long lastSend = 0;
const unsigned long INTERVAL = 10000; // bebas disesuaikan, satuan milidetik (10000 = 10 detik, 60000 = 1 menit)

void setup() {
  Serial.begin(115200);
  dht.begin();
  WiFi.begin(SSID, PASS);
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println("Tersambung WIFI");
}

void loop() {
  if (millis() - lastSend >= INTERVAL) {
    lastSend = millis();

    float suhu   = dht.readTemperature();
    float kelembaban = dht.readHumidity();

    String data = String(suhu) + "," + String(kelembaban);

    WiFiClientSecure client;
    client.setInsecure();
    HTTPClient http;
    http.begin(client, webhookURL);
    http.addHeader("Content-Type", "text/plain");
    http.POST(data);
    http.end();

    Serial.println(data);
  }
}
```

## 🔑 Setup API Key & Credentials

### Groq API Key
1. Buka https://console.groq.com/keys dan login
2. Klik **Create API Key**, beri nama, lalu klik **Submit**
3. Salin API Key yang muncul — simpan baik-baik, hanya tampil sekali

### Bot Telegram
1. Buka Telegram, cari **@BotFather** → https://t.me/BotFather
2. Kirim perintah `/newbot`
3. Ikuti instruksi: ketik nama bot, lalu username bot (harus diakhiri `bot`, contoh: `himasikom_bot`)
4. Salin **token** yang diberikan BotFather — ini credential bot kamu

---

---

## Setup Credentials di n8n

### Groq
1. Buka dashboard n8n → **Settings** → **Credentials**
2. Klik **Add Credential**, cari `Groq`
3. Masukkan API Key yang sudah didapat, klik **Save**

### Telegram
1. Klik **Add Credential**, cari `Telegram`
2. Di kolom **Access Token**, masukkan token dari BotFather
3. Klik **Save**

### Google Sheets
Siapkan dulu spreadsheet-nya:
1. Buka Google Drive, buat spreadsheet baru
2. Tambahkan 3 kolom header di baris pertama:

   | waktu | suhu | kelembaban |
   |-------|------|------------|

Kemudian di n8n:
1. Klik **Add Credential**, cari `Google Sheets OAuth2 API`
2. Klik **Sign in with Google**, pilih akun Google yang punya spreadsheet tadi
3. Izinkan akses, lalu klik **Save**

---

## Membuat Workflow n8n

Buat workflow baru di n8n, lalu tambahkan node-node berikut sesuai urutan alurnya.

Workflow ini terdiri dari **3 alur utama:**

```
[ALUR 1] Pengiriman Data Sensor
Webhook → Parse Data Sensor → Validasi Data Sensor → O/DATA → AI Agent → Balasan

[ALUR 2] Chat & Tanya Kondisi via Telegram  
Pesan Telegram → Validasi Perintah → O/TELE → AI Agent → Balasan

[ALUR 3] Rekap Data via Telegram
Pesan Telegram → Validasi Perintah → Ambil Data → Filter Data → Buat File XLS → Kirim File
```

---

### ALUR 1 — Pengiriman Data Sensor

#### Node 1 — Webhook
**Tipe:** Webhook  
**Fungsi:** Pintu masuk data dari ESP32. Setiap kali ESP32 kirim data suhu dan kelembaban, node ini yang pertama menerimanya.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| HTTP Method | POST |
| Path | `suhu` *(bebas diisi apa saja, misal `sensor`, `data`, dll)* |

> Setelah node ini dibuat, salin **Webhook URL**-nya dan tempel ke variabel `webhookURL` di kode ESP32.

---

#### Node 2 — Parse Data Sensor
**Tipe:** Code  
**Fungsi:** Memecah data mentah yang dikirim ESP32 (format `"28.5,65"`) menjadi tiga variabel terpisah: `suhu`, `kelembaban`, dan `waktu` (ditambahkan otomatis saat data masuk).

**Konfigurasi:**
```javascript
const body = $input.first().json.body ?? '';
const [suhu, kelembaban] = body.split(',').map(Number);
const waktu = new Date().toLocaleString('id-ID', { timeZone: 'Asia/Jakarta' });

return { suhu, kelembaban, waktu, data: !isNaN(suhu) && !isNaN(kelembaban) };
```

---

#### Node 3 — Validasi Data Sensor
**Tipe:** IF  
**Fungsi:** Mengecek apakah data yang masuk valid (bukan NaN / kosong). Kalau valid, lanjut ke alur normal. Kalau tidak valid, lanjut ke node Error.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Condition | `{{ $json.data }}` |
| Operator | is true |

---

#### Node 4 — O/DATA
**Tipe:** Set  
**Fungsi:** Merapikan output dari data sensor menjadi satu variabel `output` yang siap dikirim ke AI Agent.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| output | `={{ $json.suhu }}\n{{ $json.kelembaban }}\n{{ $json.waktu }}` |

---

#### Node 5 — Error
**Tipe:** Telegram  
**Fungsi:** Kalau data sensor tidak valid (misal sensor dicabut atau rusak), node ini otomatis kirim notifikasi error ke Telegram.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Credential | Telegram account |
| Chat ID | ID chat kamu (bisa dicek via @userinfobot) |
| Text | `=pada : {{ $json.waktu }}\nSensor tidak terdeteksi!!` |

---

### ALUR 2 & 3 — Chat & Rekap via Telegram

#### Node 1 — Pesan Telegram
**Tipe:** Telegram Trigger  
**Fungsi:** Memantau pesan yang masuk ke bot Telegram kamu. Setiap ada pesan baru, node ini langsung aktif dan meneruskan isi pesannya ke node berikutnya.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Credential | Telegram account |
| Updates | message |

---

#### Node 2 — Validasi Perintah
**Tipe:** IF  
**Fungsi:** Memilah pesan masuk. Kalau pesannya mengandung `/rekap`, diarahkan ke alur rekap data. Kalau bukan, diarahkan ke alur chat dengan AI Agent.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Condition | `{{ $json.message.text }}` |
| Operator | contains |
| Value | `/rekap` |

> - **True** (mengandung `/rekap`) → lanjut ke node **Ambil Data**
> - **False** (chat biasa) → lanjut ke node **O/TELE**

---

#### Node 3a — O/TELE *(cabang chat biasa)*
**Tipe:** Set  
**Fungsi:** Mengambil teks pesan dari Telegram dan memasukannya ke variabel `output` supaya bisa diteruskan ke AI Agent.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| output | `={{ $json.message.text }}` |

---

#### Node 3b — Ambil Data *(cabang /rekap)*
**Tipe:** Google Sheets  
**Fungsi:** Membaca seluruh data dari spreadsheet Google Sheets untuk disiapkan jadi bahan rekap.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Credential | Google Sheets OAuth2 API |
| Operation | Get Many Rows |
| Document | Pilih spreadsheet kamu |
| Sheet | Sheet1 |

---

#### Node 4 — Filter Data *(cabang /rekap)*
**Tipe:** Code  
**Fungsi:** Memfilter data berdasarkan rentang waktu yang diminta user. Misalnya `/rekap 30 menit` hanya menampilkan data 30 menit terakhir.

**Konfigurasi:**
```javascript
const pesan    = $('pesan telegram').first().json.message.text ?? '';
const angkaMatch = pesan.match(/\d+/);
const angka    = angkaMatch ? parseInt(angkaMatch[0]) : null;

const satuan   = pesan.includes('detik') ? 1/60
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

> **⚠️ Perhatikan dua hal ini sebelum paste kode di atas:**
> 
> 1. `$('pesan telegram')` — sesuaikan dengan **nama node Telegram Trigger** kamu. Kalau nama node-nya beda, ganti `pesan telegram` dengan nama yang sesuai.
> 2. `r.json['waktu']` — sesuaikan dengan **nama header kolom di Google Sheets** kamu. Kalau header kolom waktu-nya pakai huruf kapital (misal `Waktu`), ganti `'waktu'` jadi `'Waktu'`.

> **Catatan waktu:** Filter ini bekerja secara realtime — data yang diambil disesuaikan dengan waktu sekarang saat perintah `/rekap` dikirim. Pastikan waktu yang tersimpan di Google Sheets sudah sesuai timezone lokal kamu (WIB/UTC+7), karena kode di node **Parse Data Sensor** sudah menggunakan `Asia/Jakarta` sebagai timezone.

---

#### Node 5 — Buat File XLS *(cabang /rekap)*
**Tipe:** Convert to File  
**Fungsi:** Mengubah data hasil filter menjadi file Excel (`.xls`) yang siap dikirim ke Telegram.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Operation | XLS |
| File Name | `rekap.xls` |

---

#### Node 6 — Kirim File *(cabang /rekap)*
**Tipe:** Telegram  
**Fungsi:** Mengirimkan file rekap `.xls` ke chat Telegram user yang mengirim perintah `/rekap`.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Credential | Telegram account |
| Operation | Send Document |
| Chat ID | `={{ $('pesan telegram').item.json.message.chat.id }}` |
| Binary Data | aktifkan |

---

### AI AGENT

#### Node — AI Agent
**Tipe:** AI Agent  
**Fungsi:** Otak dari sistem. Menerima input dari O/DATA (data sensor) maupun O/TELE (chat Telegram), lalu memutuskan apakah perlu menyimpan data, membaca data, atau cukup menjawab langsung.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Prompt | `={{ $json.output }}` |
| Language Model | Groq Chat Model |

**System Message:**
```
Kamu asisten sensor ruangan. 

KONDISI SUHU: dingin <20°, nyaman 20-28°, hangat 28-33°, panas >33°C
KONDISI KELEMBABAN: kering <40%, nyaman 40-70%, lembab >70%

Kalau ada data suhu dan kelembaban masuk, simpan ke Google Sheets dulu, lalu balas:
Waktu: [waktu]
Suhu: [nilai]°C
Kelembaban: [nilai]%
[komentar kondisi 1 kalimat]

Kalau ditanya kondisi, baca baris terakhir Google Sheets dan balas dengan format yang sama.
Kalau chat biasa, balas singkat dan natural.

berikan jawaban dengan bahasa santai, to the point dan konsisten.
```

---

#### Node — Chat Model
**Tipe:** Chat Model  
**Fungsi:** Model AI yang dipakai oleh AI Agent. Dihubungkan sebagai **Language Model** ke node AI Agent.

> **Catatan:** Chat model bebas pakai provider apa saja, misal Groq Chat Model, OpenAI, Gemini, dll. Begitu juga modelnya, pilih sesuai yang tersedia di provider yang kamu pakai.

---

#### Node — Simpan Data *(Tool AI Agent)*
**Tipe:** Google Sheets Tool  
**Fungsi:** Tool yang bisa dipakai AI Agent untuk menyimpan data sensor baru ke spreadsheet. AI Agent otomatis memanggil tool ini setiap ada data masuk dari ESP32.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Credential | Google Sheets OAuth2 API |
| Operation | Append Row |
| Document | Pilih spreadsheet kamu |
| Sheet | Sheet1 |

> **✨ Mapping kolom:** Setelah pilih spreadsheet dan sheet, klik tombol **Generate** (ikon bintang/magic wand) di bagian kolom — n8n akan otomatis generate mapping `$fromAI()` untuk setiap kolom di sheet kamu. Gausah isi manual satu-satu!

---

#### Node — Baca Data *(Tool AI Agent)*
**Tipe:** Google Sheets Tool  
**Fungsi:** Tool yang bisa dipakai AI Agent untuk membaca data dari spreadsheet. Digunakan saat user tanya kondisi ruangan lewat Telegram.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Credential | Google Sheets OAuth2 API |
| Operation | Get Many Rows |
| Document | Pilih spreadsheet kamu |
| Sheet | Sheet1 |

---

#### Node — Balasan
**Tipe:** Telegram  
**Fungsi:** Mengirimkan jawaban dari AI Agent kembali ke user di Telegram.

**Konfigurasi:**
| Field | Value |
|-------|-------|
| Credential | Telegram account |
| Chat ID | `=7264187096` (ganti dengan Chat ID kamu, cek via @userinfobot) |
| Text | `={{ $json.output }}` |
| Append Attribution | nonaktifkan |

---

## ▶️ Aktifkan Workflow

Setelah semua node terhubung dengan benar, klik tombol **Activate** di pojok kanan atas n8n.

Setelah aktif, ini yang bakal terjadi secara otomatis:

- **ESP32** mulai mengirim data suhu dan kelembaban setiap 10 detik ke webhook n8n
- **AI Agent** menerima data, menyimpannya ke Google Sheets, lalu mengirim balasan ke Telegram dengan format waktu, suhu, kelembaban, dan komentar kondisi
- Kalau data sensor tidak terbaca / error, bot langsung kirim notifikasi error ke Telegram
- Kamu bisa **chat bebas** ke bot Telegram kapan saja untuk tanya kondisi ruangan — AI Agent akan baca data terakhir di Google Sheets dan balas
- Kirim `/rekap` (atau `/rekap 1 jam`, `/rekap 2 hari`, dst) untuk dapat file Excel riwayat data langsung di Telegram

---

*Happy building! — Workshop HIMASIKOM 2026*
