# RAG Dashboard (Groq + ChromaDB + Streamlit)

visit: https://q8qfazxhnk8sfyfsplqgbq.streamlit.app/ (kalau dia sleep bangunin aja pakai reboot (ribut2 bangun deh..))

Dashboard untuk tanya-jawab dokumen (RAG) dan analisis data tabular, jalan lokal atau bisa di-deploy. Semua tools yang dipakai gratis.

## Stack

- Streamlit untuk UI
- Groq API untuk LLM (Llama 3.3 / 3.1, Gemma2)
- sentence-transformers (all-MiniLM-L6-v2) untuk embedding, jalan lokal, tidak butuh API key
- ChromaDB untuk vector database, persisten di folder `./chroma_db`
- pypdf, python-docx, pandas untuk baca berbagai format file

## Setup

```bash
python3 -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate

pip install -r requirements.txt
```

Ambil API key gratis di https://console.groq.com/keys (daftar, klik "Create API Key", copy).

Simpan sebagai environment variable biar tidak perlu isi ulang tiap buka app:

```bash
export GROQ_API_KEY="gsk_xxxxxxxxxxxx"     # Mac/Linux
setx GROQ_API_KEY "gsk_xxxxxxxxxxxx"       # Windows (buka terminal baru setelah ini)
```

Kalau di-deploy dan API key sudah diisi lewat Secrets, kolom input di sidebar otomatis tidak muncul.

Jalankan:

```bash
streamlit run app.py
```

## Cara pakai

**Tab Tanya Dokumen (RAG)**
1. Upload PDF, DOCX, atau TXT, atau masukkan URL website untuk di-scrape, lalu klik "Proses Dokumen".
2. Ketik pertanyaan di kolom chat.
3. Sistem mencari potongan dokumen paling relevan, mengirimkannya ke LLM sebagai konteks, dan LLM menjawab berdasarkan itu.
4. Klik "Sumber jawaban" untuk lihat potongan mana yang dipakai, beserta skor relevansinya.

**Tab Analisis Data (CSV/Excel)**
Pendekatannya beda dari RAG, karena data tabular butuh perhitungan presisi, bukan pencarian teks mirip:

```
Pertanyaan → LLM tulis kode pandas → kode dijalankan di sandbox terbatas
   → kalau error, LLM coba perbaiki sekali
   → hasil dijelaskan dalam bahasa natural, kode & tabel mentah ditampilkan transparan
```

Kode yang dihasilkan LLM dijalankan dengan `exec()` di namespace terbatas (tanpa builtins, tanpa akses import/open/os/subprocess), plus ada pengecekan kata kunci berbahaya sebelum dijalankan. Ini mengurangi risiko tapi tidak menghilangkannya sepenuhnya — kalau dashboard ini diakses banyak orang asing, jangan expose tab ini tanpa autentikasi tambahan.

## Cara kerja RAG-nya

```
Upload dokumen → extract teks → chunking → embedding (lokal) → simpan ke ChromaDB
                                                                        │
Pertanyaan → embedding pertanyaan → cari chunk paling mirip (top-k) ───┘
      │
      ▼
pertanyaan + chunk relevan → Groq LLM → jawaban + sumbernya
```

## Format dokumen yang didukung

- PDF, termasuk OCR otomatis kalau ternyata hasil scan (butuh Tesseract + Poppler terinstall di sistem, bukan cuma lewat pip):
  - Ubuntu/Debian: `sudo apt install tesseract-ocr tesseract-ocr-ind poppler-utils`
  - macOS: `brew install tesseract tesseract-lang poppler`
  - Windows: install Tesseract-OCR (https://github.com/UB-Mannheim/tesseract/wiki) dan Poppler for Windows (https://github.com/oschwartz10612/poppler-windows/releases), tambahkan folder bin-nya ke PATH.
  - Status fitur ini bisa dicek di sidebar, expander "Status fitur opsional".
- DOCX, termasuk isi tabel
- TXT
- Website (scraping halaman statis; halaman yang butuh JavaScript berat mungkin tidak terbaca penuh)
- CSV, XLSX, XLS (lewat tab Analisis Data)

## Agentic RAG

Bisa dimatikan lewat toggle di sidebar. Dua hal yang dilakukan:

- Query decomposition: pertanyaan kompleks/multi-topik otomatis dipecah jadi beberapa sub-pertanyaan sebelum retrieval, hasilnya digabung sebelum dikirim ke LLM.
- Auto re-retrieval: kalau skor relevansi hasil pencarian pertama di bawah ambang batas (default 0.3, bisa diatur), sistem menyusun query alternatif dan mencari ulang otomatis.

Semua langkah ini bisa dilihat di panel "Proses Agentic" di bawah tiap jawaban.

## Hal-hal lain yang sudah ditangani

- Pertanyaan soal metadata ("berapa halaman", "dokumen apa saja") dijawab langsung dari metadata, bukan lewat pencarian vector, supaya selalu akurat.
- Chunking mengikuti batas paragraf, tidak memotong kata di tengah.
- Beberapa turn chat terakhir diikutkan ke prompt, jadi bisa nanya lanjutan tanpa mengulang konteks.
- Rate limit, API key salah, atau koneksi bermasalah ditampilkan sebagai pesan yang jelas, bukan bikin app crash.
- Jawaban LLM streaming, muncul kata per kata.

## Catatan

- Groq API gratis punya rate limit — cukup untuk pemakaian personal/tim kecil, cek batasnya di https://console.groq.com/settings/limits.
- Model embedding all-MiniLM-L6-v2 didownload otomatis (sekitar 90MB) saat pertama kali dijalankan, butuh koneksi internet sekali di awal.
- Jangan commit API key ke git — pakai environment variable atau Secrets, bukan hardcode di kode.
- Tombol "Reset Semua Data" di sidebar akan menghapus semua dokumen di `./chroma_db` dan data yang sudah diupload.
