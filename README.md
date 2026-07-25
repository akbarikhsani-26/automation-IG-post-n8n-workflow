# Dokumentasi & Panduan Setup Workflow n8n
## Otomasi Scraper Properti Rumah123, AI Caption Generator, Posting Instagram & Tracking Insights

Dokumentasi ini disusun khusus untuk memberikan panduan komprehensif mengenai struktur, cara kerja, kredensial, serta langkah-langkah setup workflow n8n bernama **"AI Test an Automation intagram post"**.

---

## 1. Ringkasan Eksekutif & Arsitektur Workflow

Workflow n8n ini dirancang secara otomatis untuk menyelesaikan empat tugas utama dalam operasi pemasaran properti digital:

1. **Scraping & Monitoring Listing Terbaru:** Mengambil data listing properti terbaru dari platform Rumah123 secara berkala setiap hari pukul **07:00 WIB**.
2. **Generasi Caption Kreatif berbasis AI (Gemini):** Memproses data properti (judul, harga, lokasi) menggunakan model Google Gemini AI (`gemini-3.1-flash-lite`) dengan kerangka copywriting **5W1H** yang energetik, persuasif, dan relevan bagi target audiens (Milenial, Gen Z, First-Home Buyer, dan Investor).
3. **Publikasi Otomatis ke Instagram & Database Logging:** Mengunggah konten gambar dan caption ke Instagram Business Account melalui Meta Graph API v25.0, serta menyimpan riwayat publikasi di Google Sheets.
4. **Tracking Engagement Harian:** Menarik statistik *likes* dan *comments* dari setiap postingan yang telah terbit setiap hari pukul **17:00 WIB** dan mengutakhirkan datanya di Google Sheets.

---

## 2. Struktur & Pemetaan Node (Workflow Map)

Workflow ini terbagi menjadi dua jalur eksekusi terpisah berdasarkan waktu picu (*schedule trigger*):

### Jalur A: Otomasi Scraping, AI Caption Generation & Posting Instagram (Trigger 07:00 WIB)

| Nama Node | Tipe Node | Fungsi Utama |
| :--- | :--- | :--- |
| `Schedule Trigger` | Schedule Trigger | Memicu alur kerja utama setiap hari pada pukul 07:00 pagi. |
| `HTTP Request` / `HTTP Request2` | HTTP Request | Mengambil halaman HTML dari `rumah123.com/jual/cari/?sort=posted-desc` (Halaman 1 & 2). |
| `extract_card` & `extract_HTML` | HTML Extract | Mengekstrak elemen card properti, judul, harga, lokasi, serta URL gambar listing. |
| `Edit Fields` | Set / Edit Fields | Membersihkan URL gambar dari wrapper URL encoder Rumah123. |
| `Get row(s) in sheet` & `Check New Listings (Page 1)` | Google Sheets & Code | Mengecek apakah judul listing sudah pernah ada di Google Sheets untuk mencegah duplikasi post. |
| `Has New Listings? (Page 1)` | If Condition | Percabangan: Jika ada listing baru, lanjut ke posting; jika tidak, jalankan perulangan ke Halaman 2 (`Wait1`). |
| `Limit` | Limit | Membatasi jumlah listing yang akan diproses maksimal 2 item per eksekusi. |
| `Message a model` | Google Gemini AI | Memanggil model `gemini-3.1-flash-lite` untuk menyusun caption Instagram dengan kaidah 5W1H. |
| `Edit text caption` | Set / Edit Fields | Mengambil hasil respons AI dan menyimpannya sebagai variabel `deskripsi`. |
| `Facebook Graph API` (Media Container) | Facebook Graph API | Membuat container media foto di Meta API dengan parameter `image_url` dan `caption`. |
| `Wait` | Wait Node | Memberikan jeda waktu agar server Meta selesai memproses/mengolah gambar. |
| `Facebook Graph API1` (Publish) | Facebook Graph API | Mempublikasikan container media menggunakan `creation_id` ke feed Instagram. |
| `Append or update row in sheet` | Google Sheets | Menyimpan data listing yang berhasil dipost beserta Instagram Media ID ke Google Sheets. |

### Jalur B: Otomasi Tracking Engagement Post (Trigger 17:00 WIB)

| Nama Node | Tipe Node | Fungsi Utama |
| :--- | :--- | :--- |
| `Schedule Trigger1` | Schedule Trigger | Memicu pemantauan performa harian pada pukul 17:00 sore. |
| `Get all posts` | Google Sheets | Membaca seluruh baris data dari Google Sheets. |
| `Only Published Posts` | Filter | Menyaring baris yang memiliki Instagram Media ID (`id` tidak kosong). |
| `Get Post Insights` | Facebook Graph API | Mengambil jumlah `like_count` dan `comments_count` dari Meta API. |
| `Map Insights` | Set / Edit Fields | Format ulang data jumlah like, comment, dan timestamp pemeriksaan (`last_checked`). |
| `Update Likes & Comments` | Google Sheets | Memperbarui statistik di Google Sheets berdasarkan matching ID. |

---

## 3. Prasyarat & Persiapan Kredensial

Sebelum mengimpor dan menjalankan workflow ini, Anda wajib menyiapkan 3 kredensial utama:

### 1. Google Sheets API Credential
- Gunakan OAuth2 Credential untuk mengkoneksikan akun Google Sheets Anda.
- Buat Spreadsheet baru dan siapkan header pada baris pertama (Row 1) sebagai berikut:
  ```text
  judul | harga | lokasi | gambar | deskripsi | id | likes | comments | last_checked
  ```
- *Catatan ID Spreadsheet pada workflow asal:* `1L0XWv5c15JtV-tM6VQ9cgT8ox77Tb0IVcX370RcRKQQ`

### 2. Google Gemini API Key
- Dapatkan API Key melalui [Google AI Studio](https://aistudio.google.com/).
- Masukkan ke dalam kredensial `googlePalmApi` di n8n.
- Workflow menggunakan model: `models/gemini-3.1-flash-lite`.

### 3. Meta / Facebook Graph API Token (Instagram Graph API)
- Diperlukan Facebook Developer App yang memiliki produk **Instagram Graph API**.
- Akun Instagram wajib merupakan **Akun Profesional (Bisnis / Kreator)** yang sudah terhubung dengan **Facebook Page**.
- *Permissions (Izin Access Token):*
  - `instagram_basic`
  - `instagram_content_publish`
  - `pages_read_engagement`
  - `pages_show_list`
- Dapatkan **Instagram Business Account ID** (contoh ID pada node: <details><summary>`17841443897669040`</summary></details>).

---

## 4. Panduan Setup Langkah Demi Langkah (Step-by-Step)

### Langkah 1: Import File JSON Workflow ke n8n
1. Buka dashboard n8n Anda.
2. Pilih menu **Workflows** > **Add Workflow**.
3. Klik titik tiga di pojok kanan atas, pilih **Import from File** atau **Import from URL/Text**.
4. Tempelkan seluruh isi JSON workflow dan klik **Import**.

### Langkah 2: Dapatkan & Hubungkan Kredensial di n8n
1. Buka node `Message a model` → Pilih/tambahkan kredensial **Google Gemini API** (`googlePalmApi`).
2. Buka node `Facebook Graph API`, `Facebook Graph API1`, dan `Get Post Insights` → Pilih/tambahkan kredensial **Facebook Graph API**.
3. Buka node `Get row(s) in sheet`, `Append or update row in sheet`, `Get all posts`, dan `Update Likes & Comments` → Hubungkan kredensial **Google Sheets OAuth2**.

### Langkah 3: Sesuaikan ID Spreadsheet & Instagram ID
1. Buat Google Spreadsheet baru di Drive Anda, lalu salin **Spreadsheet ID** dari URL browser.
2. Buka setiap node Google Sheets pada canvas, ganti nilai `Document ID` dengan ID Spreadsheet Anda.
3. Buka node `Facebook Graph API` dan `Facebook Graph API1`, ganti nilai parameter `node` (ID <details><summary>`17841443897669040`</summary></details>) dengan **Instagram Business Account ID** milik Anda.

### Langkah 4: Pengujian & Aktivasi Workflow
1. Lakukan uji coba eksekusi manual per node dengan mengklik **Test Step** atau **Execute Workflow**.
2. Pastikan data ter-scrape dengan sempurna, caption tergenerasi oleh AI, postingan berhasil terbit di Instagram, dan baris baru tercatat di Google Sheets.
3. Aktifkan workflow secara permanen dengan menggeser sakelar **Inactive** menjadi **Active** di pojok kanan atas.

---

## 5. Sistem Prompt & Formula Copywriting AI (Gemini)

Node AI `Message a model` menggunakan prompt terstruktur berbasis kerangka **5W1H** sebagai berikut:

```text
Kamu adalah konsultan properti profesional yang sangat antusias, enerjik, dan dapat dipercaya. Tugasmu adalah membuat caption Instagram yang memikat untuk mempromosikan listing properti terbaru.
Berikut adalah data propertinya:
- Nama/Tipe Properti: {{ $json.judul }}
- Harga: {{ $json.harga }}
- Lokasi: {{ $json.lokasi }}

Tulis caption paragraf demi paragraf dengan mengintegrasikan metode 5W1H secara natural (tanpa perlu menuliskan poin 5W1H secara kaku). Pastikan kontenmu mencakup aspek berikut dengan mulus:
1. What: Jelaskan apa properti yang ditawarkan dan daya tarik utamanya secara memukau.
2. Who: Panggil audiens secara langsung! Sebutkan bahwa properti ini adalah peluang emas yang dirancang khusus untuk Milenial, Gen Z, pembeli rumah pertama kali, dan investor cerdas.
3. Where: Berikan sorotan positif pada lokasi {{ $json.lokasi }} dan betapa nyamannya hidup atau berinvestasi di sana.
4. When: Ciptakan urgensi yang positif dan menyenangkan mengenai mengapa waktu paling tepat untuk mengambil langkah pembelian adalah sekarang juga.
5. Why: Berikan alasan kuat mengapa ini adalah keputusan finansial yang sangat menguntungkan dan mendukung gaya hidup modern yang luar biasa.
6. How: Tutup dengan ajakan bertindak (Call to Action) yang sangat jelas. Arahkan mereka untuk segera menghubungi via DM atau klik tautan di bio untuk menjadwalkan survei hari ini juga.

Gunakan nada yang sangat menyenangkan, penuh semangat, dan tetap menjaga kredibilitas profesional. Jaga agar kalimat tetap mudah dipahami, rapi, dan menarik perhatian sejak kalimat pertama! langsung untuk membuat captionnya saja tanpa menggunakan awalan dan berbentuk string
```

---

## 6. Tips Pemeliharaan & Troubleshooting

> **Penting untuk Diperhatikan:**
> 
> - **Perubahan Elemen HTML Scraping:** Website Rumah123 sewaktu-waktu dapat mengubah struktur DOM/HTML-nya. Jika scraping gagal, periksa selector CSS pada node `extract_card` (`div[data-test-id^="property-card-"]`) dan node `extract_HTML`.
> - **Masa Berlaku Access Token Facebook:** Pastikan Anda menggunakan *Page Long-Lived Access Token* (berlaku 60 hari) atau *System User Token* permanen agar proses otomatisasi tidak terhenti secara mendadak.
> - **Fungsi Wait Node sebelum Publish:** Meta/Instagram memerlukan jeda waktu untuk memproses pengunggahan gambar ke server mereka. Jangan menghapus node `Wait` sebelum node `Facebook Graph API1` untuk menghindari error `Media container is not ready`.
> - **Aksesibilitas Public URL Gambar:** Pastikan gambar yang diambil dari website properti bersifat publik dan dapat diakses langsung oleh crawler server Meta.
