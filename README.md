# Cinema 21 AI Agent
**Agentic AI Customer Service untuk Bioskop**

Proyek ini membangun AI chatbot berbasis agen untuk layanan pelanggan bioskop Cinema 21. Agent dapat menjawab pertanyaan, memberikan rekomendasi film, mengecek jadwal, dan memproses pemesanan tiket secara otomatis.

---

## Tech Stack

| Layer | Technology |
|---|---|
| LLM | Google Gemini 2.0 Flash |
| Backend | FastAPI + Uvicorn |
| Database | SQLite |
| Vector DB | FAISS + sentence-transformers |
| Knowledge Base | PDF (ReportLab) + pdfminer |
| UI | Streamlit |
| Tunnel | ngrok |

---

## Architecture

```
User (Streamlit UI)
       ↓
FastAPI /chat endpoint
       ↓
CinemaAgent — Gemini ReAct Loop
       ↓
Tool Router
       ├── rag_search          → FAISS search on cinema_knowledge.pdf
       ├── get_movie_info      → RAG sinopsis & rekomendasi film
       ├── search_movies       → SQLite movies table
       ├── get_showtimes       → SQLite showtimes + JOIN studios
       ├── check_seat_availability → SQLite seats + bookings
       └── create_booking      → INSERT into bookings
```

---

## Features

- **RAG** — PDF knowledge base untuk kebijakan, menu, promo, FAQ, dan sinopsis film
- **Function Calling** — Gemini memutuskan sendiri tool mana yang dipanggil
- **ReAct Loop** — multi-step reasoning (maksimal 8 iterasi per request)
- **Booking Flow** — cari film → jadwal → kursi → promo → konfirmasi → booking
- **Guardrails** — agent tidak booking tanpa konfirmasi eksplisit user
- **Proaktif** — agent inform promo relevan dan rekomendasikan snack setelah booking
- **Logging** — setiap tool call dan result di-log ke Colab console

---

## Database Schema

```
movies     — id, title, genre, duration, age_rating, description
cinemas    — id, name, location
studios    — id, cinema_id, studio_name, type (Regular/IMAX/4DX/Premiere)
showtimes  — id, movie_id, studio_id, show_date, show_time, price
seats      — id, studio_id, seat_number (A1-H8)
bookings   — id, customer_name, showtime_id, seat_number, status, created_at
```

---

## Seed Data

- 5 film (Avatar, Inception 2, Moana 2, Venom, Dune)
- 1 bioskop (Cinema 21 Grand Indonesia)
- 5 studio (Regular x2, IMAX, 4DX, Premiere)
- 64 kursi per studio (A1-H8)
- Jadwal tayang untuk 5 hari ke depan

---

## Setup — Google Colab

### Prerequisites
Tambahkan dua secret di Colab Secrets (ikon kunci di sidebar kiri):

| Secret | Cara dapat | Gratis? |
|---|---|---|
| `NGROK_TOKEN` | [ngrok.com](https://ngrok.com) → Dashboard → Authtoken | Ya |
| `GEMINI_KEY` | [aistudio.google.com](https://aistudio.google.com) → Get API key | Ya |

### Jalankan Cells Secara Berurutan

```
Cell 1  — Install dependencies
Cell 2  — Load API keys dari Colab Secrets
Cell 3  — Write database.py, generate_pdf.py, rag.py, agent.py
Cell 4  — Write tools.py
Cell 5  — Write agent.py
Cell 6  — Write main.py (FastAPI)
Cell 7  — Write app.py (Streamlit)
Cell 8  — Initialize database + generate PDF + build FAISS index
Cell 9  — Start FastAPI (background thread)
Cell 10 — Start Streamlit + ngrok tunnel
```

Setelah Cell 10 selesai, klik URL ngrok yang muncul dan masukkan Gemini API key di sidebar Streamlit.

---

## Contoh Percakapan

**Tanya kebijakan (RAG):**
```
User: Boleh tidak bawa makanan dari luar?
Agent: [rag_search] → Berdasarkan aturan bioskop, makanan dan minuman 
       dari luar TIDAK diperbolehkan masuk ke dalam studio...
```

**Rekomendasi film (get_movie_info):**
```
User: Film Avatar bagus tidak?
Agent: [get_movie_info("Avatar")] → Avatar: Fire and Ash cocok untuk 
       pecinta aksi dan visual spektakuler. Sangat direkomendasikan 
       di studio IMAX atau 4DX...
```

**Booking flow lengkap:**
```
User:  Saya mau nonton Avatar hari ini
Agent: [search_movies] [get_showtimes] → Ada 3 jadwal: 11:00, 14:00, 17:00 IMAX
User:  Jam 17:00 IMAX
Agent: [check_seat_availability] → 64 kursi tersedia. Saran kursi D4 (baris tengah)
User:  Oke, nama saya Budi
Agent: Konfirmasi booking Avatar 17:00 IMAX kursi D4 atas nama Budi. Lanjut?
User:  Ya
Agent: [create_booking] → BOOKING BERHASIL! ID #1
       [rag_search menu] → Rekomendasi: Combo 2 + Nachos untuk film aksi!
```

---

## Agent Guardrails

1. Tidak mengarang jadwal — selalu query SQLite
2. Tidak mengarang promo — selalu ambil dari RAG PDF
3. Tidak booking tanpa konfirmasi eksplisit user
4. Inform promo hanya yang relevan dengan hari ini
5. Rekomendasikan snack sesuai genre film
6. Rekomendasikan kursi baris D-E untuk posisi optimal

---

## RAG Knowledge Base

PDF (`cinema_knowledge.pdf`) berisi:
- Informasi umum bioskop
- Jenis studio dan fasilitas
- Aturan tiket dan rating usia
- Kebijakan refund dan pembatalan
- Menu makanan dan minuman
- Program membership
- Promo dan diskon
- FAQ
- Sinopsis dan rekomendasi film

Untuk update konten RAG, edit variable `KNOWLEDGE` di `generate_pdf.py` lalu rebuild index:
```python
generate_pdf.generate_pdf()
rag.build_index()
rag.load_index()
```

---

## Limitasi

- Free tier Gemini: 20 req/day (2.5 Flash) atau lebih banyak dengan model lain
- Tidak ada autentikasi user
- Tidak ada payment gateway
- Data film adalah seed data dummy, bukan jadwal bioskop real
- Colab session reset setiap ~12 jam (perlu re-run semua cell)
