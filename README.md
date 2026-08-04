# 📚 RAG Dashboard Gratis (Groq + ChromaDB + Streamlit)

Dashboard sederhana untuk RAG (Retrieval Augmented Generation) end-to-end.
Upload PDF/TXT → tanya isinya → jawab pakai LLM gratis.

## Stack yang dipakai (semua gratis)
- **Streamlit** → dashboard/UI
- **Groq API** → LLM (Llama 3.3, super cepat & gratis)
- **sentence-transformers** (`all-MiniLM-L6-v2`) → embedding, jalan lokal di laptop kamu (gak butuh API/key, gak bayar)
- **ChromaDB** → vector database, jalan lokal (in-memory)
- **pypdf** → baca isi PDF

## 1. Setup awal

```bash
# buat virtual environment (opsional tapi disarankan)
python3 -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate

# install dependency
pip install -r requirements.txt
```

## 2. Ambil API Key Groq (gratis)

1. Buka https://console.groq.com/keys
2. Login/daftar (gratis, gak perlu kartu kredit)
3. Klik "Create API Key", copy key-nya

Kamu bisa paste key itu langsung di kolom sidebar aplikasi,
atau simpan sebagai environment variable biar gak perlu isi ulang:

```bash
export GROQ_API_KEY="gsk_xxxxxxxxxxxx"     # Mac/Linux
# atau
setx GROQ_API_KEY "gsk_xxxxxxxxxxxx"       # Windows (buka terminal baru setelah ini)
```

## 3. Jalankan aplikasi

```bash
streamlit run app.py
```

Browser akan otomatis kebuka di `http://localhost:8501`.

## 4. Cara pakai

1. Di panel kiri, upload 1 atau beberapa file PDF/TXT, klik **"Proses Dokumen"**.
   (Ini akan: extract teks → potong jadi chunk → ubah jadi embedding → simpan ke ChromaDB)
2. Di panel kanan, ketik pertanyaan tentang isi dokumen kamu di kolom chat.
3. Aplikasi akan cari chunk paling relevan (retrieval), lalu kirim ke Groq LLM
   bersama pertanyaan kamu (generation) → itulah alur RAG-nya.
4. Klik "📄 Sumber jawaban" untuk lihat potongan dokumen mana yang dipakai sebagai referensi.

## Cara kerja RAG di sini (ringkas)

```
[Upload PDF/TXT]
      │
      ▼
[Extract teks] → [Chunking] → [Embedding (lokal, gratis)] → [Simpan ke ChromaDB]
                                                                     │
[Pertanyaan user] → [Embedding pertanyaan] → [Cari chunk mirip (top-k)] ──┘
      │
      ▼
[Kirim: pertanyaan + chunk relevan] → [Groq LLM] → [Jawaban + sumbernya]
```

## 🆕 Fitur baru: multi-format + analisis data

### Tab "📄 Tanya Dokumen (RAG)"
- **DOCX** — upload file Word langsung, otomatis di-extract (termasuk isi tabel).
- **Website scraping** — masukkan URL, kontennya otomatis diambil & masuk ke knowledge base
  (butuh `requests` + `beautifulsoup4`, sudah ada di requirements.txt). Cocok untuk artikel/
  halaman statis; halaman yang butuh JavaScript rendering berat mungkin tidak terbaca penuh.
- **OCR untuk PDF hasil scan** — kalau PDF ternyata gambar (teks kosong), sistem otomatis coba
  OCR pakai `pytesseract` + `pdf2image`. **Butuh instalasi tambahan di level sistem operasi**
  (bukan cuma `pip install`):
  - **Ubuntu/Debian**: `sudo apt install tesseract-ocr tesseract-ocr-ind poppler-utils`
  - **macOS**: `brew install tesseract tesseract-lang poppler`
  - **Windows**: install [Tesseract-OCR](https://github.com/UB-Mannheim/tesseract/wiki) dan
    [Poppler for Windows](https://github.com/oschwartz10612/poppler-windows/releases), lalu
    tambahkan folder `bin`-nya ke PATH.
  - Cek status fitur ini di sidebar → expander "ℹ️ Status fitur opsional".
- **Streaming jawaban** — jawaban LLM sekarang muncul kata-per-kata secara real-time, bukan
  nunggu full response sekaligus.

### Tab "📊 Analisis Data (CSV/Excel)"
Ini **bukan RAG** — pendekatannya beda karena data tabular butuh perhitungan presisi (rata-rata,
total, filter, dsb), bukan pencarian teks mirip. Alurnya:

```
Pertanyaan → LLM tulis kode pandas → kode dijalankan di sandbox terbatas
   → kalau error, LLM coba perbaiki sekali (auto-retry)
   → hasil dijelaskan LLM dalam bahasa natural
   → kode & hasil mentah ditampilkan transparan (expander "🔧 Kode & hasil mentah")
```

**Soal keamanan eksekusi kode**: kode yang dihasilkan LLM dijalankan dengan `exec()` di
namespace terbatas (tanpa builtins, tanpa akses `import`/`open`/`os`/`subprocess`/dll), dan
ada pemeriksaan kata kunci berbahaya sebelum dijalankan. Ini mengurangi risiko, tapi **exec()
kode hasil generate LLM tetap punya risiko inheren** — jangan expose fitur ini ke publik tanpa
autentikasi/rate-limit tambahan kalau kamu deploy untuk banyak orang asing.

Kalau hasil analisis berupa tabel/series numerik (≤30 baris), otomatis muncul bar chart juga.

## 🤖 Fitur Agentic RAG

Bisa diaktif/nonaktifkan lewat toggle "Aktifkan query decomposition + auto re-retrieval" di sidebar.

- **Query decomposition** — kalau pertanyaan kamu kompleks/multi-topik (misal "bandingkan A dan
  B, terus kasih rekomendasi"), LLM otomatis memecahnya jadi beberapa sub-pertanyaan yang lebih
  fokus, retrieval dijalankan untuk tiap sub-pertanyaan, baru hasilnya digabung sebelum dikirim
  ke LLM untuk jawaban akhir.
- **Auto re-retrieval** — kalau skor relevansi hasil pencarian pertama di bawah ambang batas
  (bisa diatur di sidebar, default 0.3), sistem otomatis menyusun query alternatif (rephrase)
  dan mencari ulang, tanpa kamu perlu mengulang pertanyaan secara manual.
- **Transparansi proses** — semua langkah "berpikir" ini (pertanyaan dipecah jadi apa, kapan
  re-retrieval dipicu, query alternatif apa yang dicoba) ditampilkan di panel "🧠 Proses Agentic"
  di bawah tiap jawaban.

Ini beda dari RAG biasa yang alurnya selalu tetap (1x retrieval → langsung jawab) — di sini LLM
ikut "mengambil keputusan" soal langkah apa yang perlu diambil sebelum menjawab.

## Fitur robust yang sudah ditambahkan
- **Persistent storage**: dokumen tersimpan di folder `./chroma_db`, jadi kalau app
  di-refresh atau di-restart, dokumen yang sudah diupload tidak hilang.
- **Deteksi pertanyaan meta**: pertanyaan seperti "berapa halaman dokumen ini?" atau
  "dokumen apa saja yang ada?" dijawab langsung dari metadata (bukan lewat pencarian
  vector), jadi selalu akurat — karena info itu memang bukan bagian dari *isi* teks.
- **Chunking sadar-paragraf**: potongan teks tidak lagi motong kata di tengah, mengikuti
  batas paragraf supaya konteks yang diambil lebih utuh dan bermakna.
- **Skor relevansi**: tiap sumber jawaban menampilkan skor kemiripan (0–1, makin tinggi
  makin relevan) beserta cuplikan teksnya, jadi kamu bisa cek sendiri apakah jawabannya
  didukung konteks yang tepat.
- **Riwayat percakapan**: beberapa turn chat terakhir diikutkan ke prompt, jadi bisa
  nanya lanjutan seperti "jelasin lebih detail soal itu" tanpa perlu ulang konteks.
- **Deteksi PDF hasil scan**: kalau PDF ternyata gambar (teks kosong/sangat sedikit),
  akan muncul peringatan bahwa dokumen itu butuh OCR dulu.
- **Penanganan error API**: rate limit, API key salah, atau koneksi bermasalah akan
  ditampilkan sebagai pesan yang jelas, bukan bikin app crash.

## Catatan penting
- Groq API gratis punya rate limit (misal sekian request/menit) — cukup untuk belajar
  dan development, tapi cek https://console.groq.com/settings/limits kalau kena limit.
- Model embedding `all-MiniLM-L6-v2` akan otomatis didownload (±90MB) saat pertama kali
  dijalankan, butuh koneksi internet sekali di awal.
- **Jangan commit API key ke Git.** Simpan di environment variable seperti contoh di atas,
  atau pakai file `.env` yang di-gitignore.
- Kalau mau mulai dari nol lagi, klik tombol **"Reset Knowledge Base"** di sidebar —
  ini akan menghapus semua dokumen yang tersimpan di `./chroma_db`.

## Ide pengembangan lanjut
- Tambah dukungan format lain (docx, csv, website via scraping).
- Tambah OCR (misal `pytesseract`) untuk PDF hasil scan.
- Tambah streaming response dari Groq biar jawaban muncul kata-per-kata.
- Deploy ke Streamlit Community Cloud (gratis) biar bisa diakses online.