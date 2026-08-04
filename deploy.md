# 🚀 Panduan Deploy ke Streamlit Community Cloud (Gratis)

## Struktur folder yang perlu di-push ke GitHub

```
rag-dashboard/
├── app.py
├── requirements.txt
├── packages.txt                ← dependency sistem (Tesseract OCR, Poppler)
├── .gitignore
├── .streamlit/
│   └── secrets.toml.example   ← ini boleh ikut, isinya cuma contoh
├── README.md
└── DEPLOY.md
```

`packages.txt` otomatis dibaca Streamlit Community Cloud untuk install package sistem (apt),
jadi fitur OCR (Tesseract) dan konversi PDF-ke-gambar (Poppler) langsung jalan di cloud tanpa
setup manual tambahan.

⚠️ **Jangan pernah push** `.streamlit/secrets.toml` (yang asli, isi API key beneran) atau
folder `chroma_db/` — keduanya sudah otomatis diabaikan lewat `.gitignore`.

---

## Langkah 1 — Buat repo di GitHub

1. Buka https://github.com/new
2. Kasih nama repo, misal `rag-dashboard`
3. Pilih **Public** (Streamlit Community Cloud gratis mensyaratkan repo public,
   kecuali kamu punya akun berbayar)
4. Klik **Create repository**

## Langkah 2 — Push project kamu

Di folder project (yang isinya `app.py`, `requirements.txt`, dll), jalankan:

```bash
git init
git add .
git commit -m "Initial commit: RAG dashboard"
git branch -M main
git remote add origin https://github.com/USERNAME/rag-dashboard.git
git push -u origin main
```

Ganti `USERNAME` dengan username GitHub kamu.

## Langkah 3 — Deploy di Streamlit Community Cloud

1. Buka https://share.streamlit.io
2. Login pakai akun GitHub kamu
3. Klik **"New app"**
4. Pilih repo `rag-dashboard`, branch `main`, dan file utama `app.py`
5. **Sebelum klik Deploy**, klik **"Advanced settings"** → bagian **Secrets**, isi:
   ```toml
   GROQ_API_KEY = "gsk_xxxxxxxxxxxxxxxxxxxxxxxx"
   ```
   (paste API key Groq asli kamu di sini, bukan yang di file `.example`)
6. Klik **Deploy**

Tunggu beberapa menit (install dependency + download model embedding pertama kali).
Setelah selesai, kamu dapat URL publik seperti `https://rag-dashboard-xxxx.streamlit.app`.

## Langkah 4 — Update aplikasi di kemudian hari

Setiap kali kamu `git push` ke branch `main`, Streamlit Cloud otomatis re-deploy versi terbaru.

```bash
git add .
git commit -m "Update fitur X"
git push
```

---

## ⚠️ Batasan penting yang perlu kamu tahu

1. **Penyimpanan tidak permanen (ephemeral).**
   Folder `chroma_db/` di server Streamlit Cloud bisa hilang kalau app di-restart, sleep
   (karena idle lama di free tier), atau di-redeploy. Artinya dokumen yang diupload user
   bisa hilang sewaktu-waktu di versi gratis ini.
   - Untuk **demo/prototype/pemakaian pribadi** → gapapa, tinggal upload ulang.
   - Untuk **produksi beneran** (banyak user, data harus awet) → ganti vector DB ke yang
     hosted, misal Chroma Cloud, Qdrant Cloud (ada free tier), atau Pinecone (ada free tier).
     Tinggal ganti bagian `get_chroma_client()` di `app.py` untuk connect ke layanan itu.

2. **1 API key dipakai bersama semua pengunjung** kalau kamu isi lewat Secrets (server-side).
   Karena app-nya public dan kuota gratis Groq terbatas per key, kalau banyak yang pakai
   sekaligus bisa cepat kena rate limit. Dua opsi:
   - Biarkan kolom "Groq API Key" di sidebar tetap bisa diisi manual oleh tiap pengunjung
     (mereka pakai key mereka sendiri) — ini yang sudah didesain di app kita.
   - Atau isi lewat Secrets kalau memang cuma buat kamu/tim kecil pakai sendiri.

3. **Free tier Streamlit Cloud** app akan "tidur" (sleep) kalau tidak diakses dalam
   beberapa waktu, dan perlu beberapa detik untuk "bangun" lagi saat diakses ulang —
   ini normal, bukan bug.

## Alternatif lain (kalau butuh persistent storage gratis)

- **Hugging Face Spaces** — mirip Streamlit Cloud, gratis, support Streamlit juga.
- **Railway / Render free tier** — bisa attach persistent disk, tapi ada limit jam
  aktif per bulan di tier gratisnya.