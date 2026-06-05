# 🧪 Modul Materi — Workshop Himasikom
# Wireflow - Where IoT Meets Automation

## 📚 Setup Awal Sebelum Mulai
> 
> Install Arduino IDE di: https://www.arduino.cc/en/software/
> 
> Buat Akun pada Platform n8n: https://n8n.io/

---

## Compile ESP32 & DHT11 💬
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
const unsigned long INTERVAL = 10000; //untuk set berapa lama data sensor masuk ke esp32

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
> Setup API Key Groq di: https://console.groq.com/keys
>
> Create Bot Telegram:
```
BotFather: https://t.me/BotFather
```


### Coba Ini!
```
Kamu: Halo, nama gue Budi
Kamu: Apa hobi yang cocok untuk introvert?
Kamu: Siapa nama gue tadi?   <-- test apakah Gemini ingat!
```

### Tantangan Bonus
```python
# Tambahkan system instruction untuk bikin persona khusus
model = genai.GenerativeModel(
    "gemini-2.0-flash",
    system_instruction="Kamu adalah asisten kuliner Indonesia yang friendly. "
                       "Selalu rekomendasikan makanan lokal dan pakai bahasa santai."
)
```

---

## Lab 2 — Vision & OCR 👁️
**Modul:** Google Cloud Vision / Gemini Vision  
**Durasi:** ~15 menit  
**Tujuan:** Analisis gambar dan ekstrak teks dari foto

### Konsep
Gemini adalah model multimodal — bisa menerima gambar + teks sekaligus dalam satu prompt. Nggak perlu API Vision terpisah untuk use case sederhana.

### Kode Demo — Analisis Gambar dari URL

```python
import google.generativeai as genai
import requests
from PIL import Image
from io import BytesIO

genai.configure(api_key="ISI_API_KEY_KALIAN")
model = genai.GenerativeModel("gemini-2.0-flash")

# ── Fungsi helper: download gambar dari URL ──
def load_image_from_url(url):
    response = requests.get(url)
    return Image.open(BytesIO(response.content))

# ── Demo 1: Deskripsikan gambar ──
image_url = "https://upload.wikimedia.org/wikipedia/commons/thumb/3/3a/Cat03.jpg/1200px-Cat03.jpg"
image = load_image_from_url(image_url)

response = model.generate_content([
    image,
    "Deskripsikan gambar ini dalam Bahasa Indonesia secara detail."
])
print("📸 Deskripsi Gambar:")
print(response.text)
```

### Kode Demo — OCR: Baca Teks dari Gambar

```python
# ── Demo 2: Ekstrak teks (OCR) ──
# Ganti URL ini dengan foto struk, papan nama, atau dokumen kalian
ocr_image_url = "https://upload.wikimedia.org/wikipedia/commons/thumb/a/af/Cara_menulis_surat_lamaran_kerja.jpg/220px-Cara_menulis_surat_lamaran_kerja.jpg"
ocr_image = load_image_from_url(ocr_image_url)

response = model.generate_content([
    ocr_image,
    "Baca dan ekstrak semua teks yang ada di gambar ini. Tampilkan apa adanya."
])
print("\n📄 Hasil OCR:")
print(response.text)
```

### Kode Demo — Analisis dari File Lokal

```python
# ── Demo 3: Upload gambar dari laptop sendiri ──
# Ganti path ini dengan file gambar kalian
local_image = Image.open("foto_struk.jpg")  # atau foto_ktp.jpg, menu.jpg, dll

response = model.generate_content([
    local_image,
    "Ini adalah struk belanja. Tolong ekstrak: nama toko, tanggal, daftar item, dan total harga."
])
print("\n🧾 Analisis Struk:")
print(response.text)
```

### Tantangan Bonus
```python
# Tanya hal lain tentang gambar yang sama
response = model.generate_content([
    image,
    "Kalau hewan ini bisa bicara, apa yang kira-kira dia katakan? Jawab dengan lucu!"
])
print(response.text)
```

---

## Lab 3 — RAG: Tanya Dokumen Sendiri 📚
**Modul:** Embedding & RAG  
**Durasi:** ~20 menit  
**Tujuan:** Bikin sistem yang bisa menjawab pertanyaan berdasarkan dokumen kita

### Konsep
RAG = Retrieval-Augmented Generation. Alurnya:
1. **Chunk** dokumen jadi potongan kecil
2. **Embed** setiap potongan jadi vector angka
3. **Simpan** vector di "database" sederhana
4. Saat ada pertanyaan → embed pertanyaannya → **cari** potongan yang paling mirip → **kirim ke Gemini** sebagai konteks

### Kode Demo — RAG Sederhana (Tanpa Library Berat)

```python
import google.generativeai as genai
import numpy as np

genai.configure(api_key="ISI_API_KEY_KALIAN")

# ── STEP 1: Siapkan "dokumen" kita ──
# Contoh: FAQ perusahaan / knowledge base
dokumen = [
    "Jam operasional toko kami adalah Senin-Jumat pukul 09.00-18.00 WIB.",
    "Kami menerima pembayaran via transfer bank, GoPay, OVO, dan kartu kredit.",
    "Pengiriman gratis untuk pembelian di atas Rp 150.000 ke seluruh wilayah Jabodetabek.",
    "Produk dapat dikembalikan dalam 7 hari setelah pembelian dengan kondisi masih tersegel.",
    "Customer service kami bisa dihubungi di support@toko.com atau WhatsApp 0812-3456-7890.",
    "Kami menyediakan lebih dari 500 produk elektronik, mulai dari smartphone hingga aksesori.",
    "Program loyalitas kami memberikan poin reward untuk setiap transaksi di atas Rp 50.000.",
]

# ── STEP 2: Embed semua dokumen ──
print("⏳ Membuat embedding untuk semua dokumen...")

def embed_text(text):
    result = genai.embed_content(
        model="models/text-embedding-004",
        content=text,
        task_type="retrieval_document"
    )
    return np.array(result['embedding'])

doc_embeddings = [embed_text(doc) for doc in dokumen]
print(f"✅ {len(doc_embeddings)} dokumen berhasil di-embed!\n")

# ── STEP 3: Fungsi pencarian ──
def cari_dokumen_relevan(pertanyaan, top_k=2):
    """Cari potongan dokumen yang paling relevan dengan pertanyaan"""
    query_embedding = genai.embed_content(
        model="models/text-embedding-004",
        content=pertanyaan,
        task_type="retrieval_query"
    )['embedding']
    query_embedding = np.array(query_embedding)
    
    # Hitung cosine similarity
    similarities = []
    for i, doc_emb in enumerate(doc_embeddings):
        similarity = np.dot(query_embedding, doc_emb) / (
            np.linalg.norm(query_embedding) * np.linalg.norm(doc_emb)
        )
        similarities.append((similarity, i))
    
    # Ambil top-k yang paling mirip
    similarities.sort(reverse=True)
    return [dokumen[idx] for _, idx in similarities[:top_k]]

# ── STEP 4: Jawab pertanyaan pakai RAG ──
def tanya_dokumen(pertanyaan):
    print(f"❓ Pertanyaan: {pertanyaan}")
    
    konteks = cari_dokumen_relevan(pertanyaan)
    print(f"🔍 Konteks ditemukan: {konteks}\n")
    
    prompt = f"""Kamu adalah asisten customer service. 
Jawab pertanyaan berikut HANYA berdasarkan konteks yang diberikan.
Kalau jawabannya tidak ada di konteks, bilang "Maaf, saya tidak punya informasi tersebut."

Konteks:
{chr(10).join(f'- {k}' for k in konteks)}

Pertanyaan: {pertanyaan}
Jawaban:"""
    
    model = genai.GenerativeModel("gemini-2.0-flash")
    response = model.generate_content(prompt)
    print(f"🤖 Jawaban: {response.text}\n")
    print("-" * 50)

# ── TEST: Coba berbagai pertanyaan ──
tanya_dokumen("Apakah bisa bayar pakai GoPay?")
tanya_dokumen("Jam berapa toko tutup?")
tanya_dokumen("Bagaimana cara retur produk?")
tanya_dokumen("Apakah ada diskon hari ini?")  # <-- tidak ada di dokumen!
```

---

## Lab 4 — Speech: STT & TTS 🎙️
**Modul:** Speech to Text & Text to Speech  
**Durasi:** ~15 menit  
**Tujuan:** Konversi suara ke teks dan sebaliknya

### Konsep
- **STT (Speech-to-Text):** Rekam suara → jadi teks → proses dengan Gemini
- **TTS (Text-to-Speech):** Teks dari Gemini → jadi file audio → putar

### Kode Demo — TTS: Gemini Bicara
```python
from gtts import gTTS
import os
import google.generativeai as genai

genai.configure(api_key="ISI_API_KEY_KALIAN")

def gemini_bicara(pertanyaan, bahasa="id"):
    """Tanya Gemini → jawaban diubah jadi suara"""
    
    # 1. Dapat jawaban dari Gemini
    model = genai.GenerativeModel("gemini-2.0-flash")
    response = model.generate_content(pertanyaan)
    teks_jawaban = response.text
    
    print(f"📝 Jawaban Gemini: {teks_jawaban}\n")
    
    # 2. Ubah teks jadi suara (TTS)
    tts = gTTS(text=teks_jawaban, lang=bahasa, slow=False)
    tts.save("jawaban.mp3")
    
    print("🔊 Memainkan audio...")
    os.system("mpg321 jawaban.mp3")   # Linux
    # os.system("start jawaban.mp3")  # Windows
    # os.system("afplay jawaban.mp3") # Mac

# Coba!
gemini_bicara("Jelaskan apa itu kecerdasan buatan dalam 2 kalimat saja.")
gemini_bicara("Sebutkan 3 makanan khas Indonesia yang terkenal.")
```

### Kode Demo — STT: Bicara ke Gemini
```python
import speech_recognition as sr
import google.generativeai as genai

genai.configure(api_key="ISI_API_KEY_KALIAN")

def dengarkan_dan_tanya():
    """Rekam suara → ubah ke teks → tanya ke Gemini"""
    
    recognizer = sr.Recognizer()
    
    print("🎙️ Siap merekam... Bicara sekarang!")
    
    with sr.Microphone() as source:
        # Adjust untuk noise sekitar
        recognizer.adjust_for_ambient_noise(source, duration=1)
        audio = recognizer.listen(source, timeout=5)
    
    try:
        # STT: audio → teks
        teks = recognizer.recognize_google(audio, language="id-ID")
        print(f"✅ Kamu bilang: '{teks}'")
        
        # Kirim ke Gemini
        model = genai.GenerativeModel("gemini-2.0-flash")
        response = model.generate_content(teks)
        print(f"🤖 Gemini menjawab: {response.text}")
        
    except sr.UnknownValueError:
        print("❌ Maaf, tidak bisa mendengar dengan jelas. Coba lagi.")
    except sr.RequestError as e:
        print(f"❌ Error: {e}")

# Jalankan
dengarkan_dan_tanya()
```

### Pipeline Lengkap: Voice Assistant Sederhana
```python
# Gabungin STT + Gemini + TTS = Voice Assistant!
def voice_assistant():
    recognizer = sr.Recognizer()
    model = genai.GenerativeModel("gemini-2.0-flash")
    
    print("🤖 Voice Assistant aktif! Tekan Ctrl+C untuk berhenti.\n")
    
    while True:
        try:
            print("🎙️ Mendengarkan...")
            with sr.Microphone() as source:
                recognizer.adjust_for_ambient_noise(source, duration=0.5)
                audio = recognizer.listen(source, timeout=5)
            
            # STT
            teks_input = recognizer.recognize_google(audio, language="id-ID")
            print(f"Kamu: {teks_input}")
            
            # Gemini
            response = model.generate_content(teks_input)
            jawaban = response.text
            print(f"Asisten: {jawaban}\n")
            
            # TTS
            tts = gTTS(text=jawaban, lang="id")
            tts.save("response.mp3")
            os.system("mpg321 response.mp3")
            
        except KeyboardInterrupt:
            print("\n👋 Voice Assistant dimatikan.")
            break
        except Exception as e:
            print(f"⚠️ Error: {e}, mencoba lagi...")

voice_assistant()
```

---

## Lab 5 — On-Premise dengan Gemma & Ollama 🖥️
**Modul:** On-Premise Models  
**Durasi:** ~15 menit  
**Tujuan:** Jalankan AI lokal tanpa internet, bandingkan dengan Gemini API

### Setup Ollama (Lakukan Sebelum Lab!)
```bash
# 1. Install Ollama (Linux/Mac)
curl -fsSL https://ollama.ai/install.sh | sh

# Windows: download dari https://ollama.ai/download

# 2. Pull model Gemma (pilih sesuai RAM kalian)
ollama pull gemma3:2b    # RAM 8GB - REKOMENDASI untuk lab ini
ollama pull gemma3:9b    # RAM 16GB
ollama pull gemma3:27b   # RAM 32GB+

# 3. Pastikan Ollama berjalan
ollama serve   # atau biasanya auto-start setelah install
```

### Kode Demo — Chat dengan Gemma Lokal

```python
import requests
import json

# Ollama expose API yang kompatibel dengan OpenAI di localhost
OLLAMA_URL = "http://localhost:11434/api/generate"

def tanya_gemma(pertanyaan, model="gemma3:2b"):
    """Kirim pertanyaan ke Gemma yang jalan lokal"""
    
    payload = {
        "model": model,
        "prompt": pertanyaan,
        "stream": False
    }
    
    print(f"🖥️ [LOCAL] Mengirim ke {model}...")
    response = requests.post(OLLAMA_URL, json=payload)
    result = response.json()
    
    return result["response"]

# Test!
pertanyaan = "Jelaskan apa itu machine learning dalam bahasa sederhana."
jawaban = tanya_gemma(pertanyaan)
print(f"Gemma: {jawaban}")
```

### Kode Demo — Bandingkan Gemma vs Gemini

```python
import requests
import google.generativeai as genai
import time

genai.configure(api_key="ISI_API_KEY_KALIAN")

def benchmark_model(pertanyaan):
    """Bandingkan respons Gemma (lokal) vs Gemini (cloud)"""
    
    print(f"\n{'='*60}")
    print(f"❓ Pertanyaan: {pertanyaan}")
    print(f"{'='*60}\n")
    
    # ── Test Gemma (Lokal) ──
    print("🖥️  [GEMMA - ON PREMISE]")
    start = time.time()
    payload = {"model": "gemma3:2b", "prompt": pertanyaan, "stream": False}
    response_local = requests.post("http://localhost:11434/api/generate", json=payload)
    jawaban_gemma = response_local.json()["response"]
    waktu_gemma = time.time() - start
    
    print(f"Jawaban: {jawaban_gemma[:300]}...")
    print(f"⏱️  Waktu: {waktu_gemma:.2f} detik")
    
    print()
    
    # ── Test Gemini (Cloud) ──
    print("☁️  [GEMINI - CLOUD API]")
    start = time.time()
    model = genai.GenerativeModel("gemini-2.0-flash")
    response_cloud = model.generate_content(pertanyaan)
    jawaban_gemini = response_cloud.text
    waktu_gemini = time.time() - start
    
    print(f"Jawaban: {jawaban_gemini[:300]}...")
    print(f"⏱️  Waktu: {waktu_gemini:.2f} detik")
    
    print(f"\n📊 Ringkasan: Gemma {waktu_gemma:.2f}s | Gemini {waktu_gemini:.2f}s")

# Coba beberapa pertanyaan
benchmark_model("Apa itu neural network? Jelaskan dengan analogi sederhana.")
benchmark_model("Tulis puisi pendek tentang Jakarta.")
```

### Kode Demo — Pakai OpenAI-Compatible API

```python
# Karena Ollama kompatibel dengan OpenAI, bisa pakai openai library juga!
from openai import OpenAI

# Arahkan ke Ollama lokal, bukan server OpenAI
client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # tidak dipakai, tapi wajib diisi
)

response = client.chat.completions.create(
    model="gemma3:2b",
    messages=[
        {"role": "system", "content": "Kamu adalah asisten yang helpful."},
        {"role": "user", "content": "Sebutkan 5 kota terbesar di Indonesia."}
    ]
)

print(response.choices[0].message.content)
```

---

## Lab 6 — Recommendation System 🌟
**Modul:** AI-Powered Recommendations  
**Durasi:** ~20 menit  
**Tujuan:** Bangun movie recommender berbasis semantic similarity

### Konsep
Pakai Gemini Embedding untuk representasikan setiap film sebagai vector. Film yang "mirip" akan punya vector yang berdekatan. Saat user pilih satu film → cari film lain yang vectornya paling dekat.

### Kode Demo — Movie Recommender

```python
import google.generativeai as genai
import numpy as np

genai.configure(api_key="ISI_API_KEY_KALIAN")

# ── Dataset film sederhana ──
film_database = [
    {"judul": "Inception", "deskripsi": "Pencuri yang masuk ke mimpi orang untuk mencuri rahasia. Thriller sci-fi penuh twist dengan visual mind-bending."},
    {"judul": "Interstellar", "deskripsi": "Astronot melakukan perjalanan melewati wormhole untuk menyelamatkan umat manusia. Drama keluarga bertema luar angkasa dan fisika."},
    {"judul": "The Matrix", "deskripsi": "Hacker menemukan bahwa dunia nyata adalah simulasi komputer. Action sci-fi tentang pembebasan dari sistem kontrol."},
    {"judul": "Avengers: Endgame", "deskripsi": "Para superhero berkumpul untuk melawan Thanos dan membalikkan bencana. Aksi superhero epic dengan banyak karakter."},
    {"judul": "Spider-Man: No Way Home", "deskripsi": "Spider-Man berhadapan dengan villain dari multiverse. Film superhero penuh nostalgia dan aksi seru."},
    {"judul": "La La Land", "deskripsi": "Kisah cinta antara musisi jazz dan aktris di Los Angeles. Drama musikal romantis dengan sinematografi indah."},
    {"judul": "Titanic", "deskripsi": "Romansa tragis antara dua penumpang kapal Titanic yang karam. Drama historical romance yang mengharukan."},
    {"judul": "Parasite", "deskripsi": "Keluarga miskin menginfiltrasi keluarga kaya. Thriller sosial Korea dengan plot twist mengejutkan."},
    {"judul": "Get Out", "deskripsi": "Pria kulit hitam mengunjungi keluarga pacarnya yang kulit putih dan menemukan rahasia gelap. Horror thriller tentang rasisme."},
    {"judul": "Everything Everywhere All at Once", "deskripsi": "Pemilik laundromat melintasi multiverse untuk menyelamatkan dunia. Sci-fi komedi tentang keluarga imigran China-Amerika."},
    {"judul": "Dune", "deskripsi": "Pewaris dinasti noble memimpin pemberontakan di planet padang pasir penghasil rempah. Epic sci-fi penuh politik dan takdir."},
    {"judul": "Joker", "deskripsi": "Asal usul villain Batman, komedian gagal yang berubah jadi simbol anarki. Drama psikologis gelap tentang masyarakat yang gagal."},
]

# ── STEP 1: Embed semua film ──
print("⏳ Membuat embedding untuk semua film...")

def embed_film(film):
    teks = f"{film['judul']}: {film['deskripsi']}"
    result = genai.embed_content(
        model="models/text-embedding-004",
        content=teks,
        task_type="retrieval_document"
    )
    return np.array(result['embedding'])

film_embeddings = [(film, embed_film(film)) for film in film_database]
print(f"✅ {len(film_embeddings)} film berhasil di-embed!\n")

# ── STEP 2: Fungsi rekomendasi ──
def rekomendasikan(judul_film, top_n=3):
    """Rekomendasikan film berdasarkan kemiripan semantik"""
    
    # Cari film yang dipilih
    film_pilihan = next((f for f in film_database if f["judul"] == judul_film), None)
    if not film_pilihan:
        print(f"❌ Film '{judul_film}' tidak ditemukan di database.")
        return
    
    # Embed film pilihan
    query_emb = embed_film(film_pilihan)
    
    # Hitung similarity dengan semua film lain
    scores = []
    for film, emb in film_embeddings:
        if film["judul"] == judul_film:
            continue  # skip film yang sama
        
        similarity = np.dot(query_emb, emb) / (
            np.linalg.norm(query_emb) * np.linalg.norm(emb)
        )
        scores.append((similarity, film))
    
    scores.sort(reverse=True)
    
    print(f"\n🎬 Karena kamu suka '{judul_film}', kami rekomendasikan:\n")
    for i, (score, film) in enumerate(scores[:top_n], 1):
        print(f"{i}. {film['judul']} (similarity: {score:.3f})")
        print(f"   📝 {film['deskripsi']}\n")

# ── TEST ──
rekomendasikan("Inception")
rekomendasikan("La La Land")
rekomendasikan("Avengers: Endgame")
```

### Kode Demo — Rekomendasi Berbasis Deskripsi (Tanpa Pilih Film)

```python
def rekomendasikan_dari_deskripsi(deskripsi_keinginan, top_n=3):
    """User deskripsikan mood/genre → sistem rekomendasikan film"""
    
    print(f"\n🔍 Mencari film untuk: '{deskripsi_keinginan}'\n")
    
    # Embed keinginan user
    result = genai.embed_content(
        model="models/text-embedding-004",
        content=deskripsi_keinginan,
        task_type="retrieval_query"
    )
    query_emb = np.array(result['embedding'])
    
    # Hitung similarity
    scores = []
    for film, emb in film_embeddings:
        similarity = np.dot(query_emb, emb) / (
            np.linalg.norm(query_emb) * np.linalg.norm(emb)
        )
        scores.append((similarity, film))
    
    scores.sort(reverse=True)
    
    print(f"🎬 Top {top_n} rekomendasi:\n")
    for i, (score, film) in enumerate(scores[:top_n], 1):
        print(f"{i}. {film['judul']} (score: {score:.3f})")
        print(f"   {film['deskripsi']}\n")

# Coba dengan deskripsi mood!
rekomendasikan_dari_deskripsi("film yang bikin mikir keras, penuh twist, dan sci-fi")
rekomendasikan_dari_deskripsi("film romantis yang mengharukan")
rekomendasikan_dari_deskripsi("superhero aksi seru")
rekomendasikan_dari_deskripsi("film yang gelap dan psikologis")
```

### Bonus — Penjelasan AI untuk Rekomendasi

```python
def rekomendasikan_dengan_penjelasan(film_kesukaan):
    """Rekomendasikan + minta Gemini jelaskan kenapa cocok"""
    
    film = next((f for f in film_database if f["judul"] == film_kesukaan), None)
    if not film:
        return
    
    query_emb = embed_film(film)
    scores = [(np.dot(query_emb, emb) / (np.linalg.norm(query_emb) * np.linalg.norm(emb)), f)
              for f, emb in film_embeddings if f["judul"] != film_kesukaan]
    scores.sort(reverse=True)
    top_rekomendasi = scores[0][1]
    
    # Minta Gemini jelaskan kenapa direkomendasikan
    model = genai.GenerativeModel("gemini-2.0-flash")
    prompt = f"""User suka film "{film_kesukaan}" yang dideskripsikan sebagai:
"{film['deskripsi']}"

Sistem merekomendasikan film "{top_rekomendasi['judul']}" yang dideskripsikan sebagai:
"{top_rekomendasi['deskripsi']}"

Jelaskan dalam 2-3 kalimat singkat mengapa kedua film ini cocok untuk orang yang sama.
Gunakan bahasa yang casual dan engaging."""
    
    response = model.generate_content(prompt)
    
    print(f"\n🎬 Rekomendasi untuk fans '{film_kesukaan}':")
    print(f"   ➡️  {top_rekomendasi['judul']}")
    print(f"\n💬 Kenapa cocok? {response.text}")

rekomendasikan_dengan_penjelasan("Inception")
rekomendasikan_dengan_penjelasan("La La Land")
```

---

## 📋 Cheat Sheet: Referensi Cepat

| Lab | Model/API | Import Utama | Fungsi Kunci |
|-----|-----------|--------------|--------------|
| Lab 1 | `gemini-2.0-flash` | `google.generativeai` | `model.start_chat()` |
| Lab 2 | `gemini-2.0-flash` | `PIL.Image`, `requests` | `model.generate_content([image, text])` |
| Lab 3 | `text-embedding-004` | `numpy` | `genai.embed_content()` |
| Lab 4 | `gemini-2.0-flash` | `gTTS`, `speech_recognition` | `gTTS()`, `recognizer.listen()` |
| Lab 5 | `gemma3:2b` (Ollama) | `requests` | `POST localhost:11434/api/generate` |
| Lab 6 | `text-embedding-004` | `numpy` | `np.dot()` untuk cosine similarity |

---

## 🔗 Resource Tambahan

- **Google AI Studio (API Key):** https://aistudio.google.com
- **Gemini API Docs:** https://ai.google.dev/docs
- **Ollama:** https://ollama.ai
- **Embedding Guide:** https://ai.google.dev/gemini-api/docs/embeddings

---

*Happy coding! 🚀 — Mini Bootcamp Google Generative AI*
