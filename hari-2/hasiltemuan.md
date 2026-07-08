# LAPORAN TEMUAN — PASAR PAGI

---

## Temuan 1: [BUG] Total harga tidak diformat dengan benar

- **Masalahnya apa:** Total harga di sidebar (`modal-total-price`) dan di modal review ditampilkan tanpa `.toFixed(2)`, sehingga bisa menghasilkan angka seperti `4.800000000000001` akibat floating-point.
- **Cara buktiinnya:**
  1. Belanja beberapa kombinasi barang (mis. Apel Fuji $1.5 + Jeruk Navel $2.0).
  2. Buka DevTools > Elements, lihat `#modal-total-price`.
  3. Total akan tampil tanpa 2 desimal, kadang dengan banyak angka di belakang koma.
- **Kenapa ini bahaya:** Pembeli bingung membaca harga, toko terlihat tidak profesional, dan bisa terjadi selisih pembulatan yang merugikan.
- **Cara betulinnya:** Gunakan `total.toFixed(2)` pada baris 123 (`renderCart`) dan baris 231 (`openReview`) di `main.js`.

**Lokasi:** `main.js:123` dan `main.js:231`

---

## Temuan 2: [BUG] Input jumlah barang kosong/tidak valid menyebabkan NaN

- **Masalahnya apa:** Saat user mengosongkan input `edit-quantity-input` atau mengisi huruf, `parseInt` mengembalikan `NaN`. Karena `NaN <= 0` adalah `false`, kode tetap menyimpan `NaN` sebagai `cart[id].count`, merusak data keranjang.
- **Cara buktiinnya:**
  1. Tambahkan barang ke keranjang.
  2. Di sidebar, hapus angka di input quantity (kosongkan).
  3. Perhatikan total menjadi `NaN` dan keranjang tidak bisa dioperasikan normal.
- **Kenapa ini bahaya:** Keranjang pengguna rusak, aplikasi dalam keadaan tidak terdefinisi, bisa menyebabkan kebingungan atau kehilangan data pesanan.
- **Cara betulinnya:** Validasi input di `updateQuantity`: jika `isNaN(quantity)` atau `quantity < 1`, hapus item atau kembalikan ke nilai sebelumnya.

**Lokasi:** `main.js:158-166`

---

## Temuan 3: [KEAMANAN] XSS (Cross-Site Scripting) pada preview catatan

- **Masalahnya apa:** Baris `preview.innerHTML = "Catatan: " + note;` memasukkan input user langsung ke DOM tanpa sanitasi. User bisa menyisipkan tag HTML/script berbahaya.
- **Cara buktiinnya:**
  1. Di textarea "Catatan buat petani", ketik: `<img src=x onerror="alert('XSS')">`
  2. Preview catatan di sidebar akan mengeksekusi script tersebut.
- **Kenapa ini bahaya:** Penyerang bisa mencuri cookie, mengarahkan pengguna ke situs phishing, atau memodifikasi tampilan toko.
- **Cara betulinnya:** Ganti `innerHTML` dengan `textContent` pada baris 115 (`main.js`).

**Lokasi:** `main.js:115`

---

## Temuan 4: [KEAMANAN] Kupon rahasia terlihat di source code

- **Masalahnya apa:** Kode kupon `TEMANFARMER` dengan diskon 90% disimpan sebagai konstanta di JavaScript (`main.js:32`). Siapa pun bisa membuka View Source atau DevTools dan melihatnya.
- **Cara buktiinnya:**
  1. Buka DevTools > Sources atau buka `main.js`.
  2. Cari string `KUPON_RAHASIA` — nilainya `"TEMANFARMER"` terlihat jelas.
  3. Masukkan kupon tersebut dan dapatkan diskon 90%.
- **Kenapa ini bahaya:** Semua pengguna bisa mendapat diskon 90% tanpa batas, toko merugi besar. Jika ini kupon internal terbatas, rahasianya bocor ke publik.
- **Cara betulinnya:** Validasi kupon harus dilakukan di server-side, bukan di client-side.

**Lokasi:** `main.js:32-33`

---

## Temuan 5: [KEAMANAN] Harga bisa diubah melalui DevTools (data-price di DOM)

- **Masalahnya apa:** Harga yang dipakai menghitung total diambil dari `data-price` di tombol "+" (`main.js:257`), bukan dari data produk asli. User bisa mengubah `data-price` via DevTools.
- **Cara buktiinnya:**
  1. Buka DevTools > Elements.
  2. Cari tombol "+" pada produk Apel Fuji, ubah `data-price="1.5"` menjadi `data-price="0.01"`.
  3. Klik tombol "+" — harga yang dipakai $0.01, bukan $1.5.
- **Kenapa ini bahaya:** Pembeli nakal bisa membeli barang dengan harga semau mereka. Toko rugi besar. Integritas harga jual hancur.
- **Cara betulinnya:** Ambil harga dari array `products` asli berdasarkan `id`, bukan dari atribut DOM.

**Lokasi:** `main.js:136` dan `main.js:257`

---

## Temuan 6: [ETIKA] Stok palsu / false scarcity

- **Masalahnya apa:** Setiap kali `renderProducts()` dipanggil, `sisa` dihitung ulang secara acak (`Math.floor(Math.random() * 5) + 1`). Angka stok berubah-ubah setiap kali user menekan tombol +/-.
- **Cara buktiinnya:**
  1. Catat angka "tinggal X lagi hari ini!" untuk suatu produk.
  2. Tambah/kurangi barang di keranjang.
  3. Angka stok produk berubah secara acak, tidak konsisten.
- **Kenapa ini tidak adil:** Ini dark pattern "false urgency" — sengaja membuat pembeli panik dan terburu-buru membeli dengan ilusi stok terbatas.
- **Cara betulinnya:** Hapus elemen stok palsu, atau gunakan stok riil yang konsisten dan tidak berubah tanpa alasan.

**Lokasi:** `main.js:47`

---

## Temuan 7: [ETIKA] Biaya penanganan tidak diungkap dari awal

- **Masalahnya apa:** Biaya penanganan $0.30 (`HANDLING_FEE`) ditambahkan ke total di `renderCart()` dan `openReview()`, tapi tidak pernah ditampilkan di sidebar — hanya muncul di modal review yang mungkin terlewat oleh pembeli.
- **Cara buktiinnya:**
  1. Tambahkan barang ke keranjang.
  2. Lihat total di sidebar, misal $1.50 + $0.30 = $1.80, yang terlihat hanya $1.80 tanpa penjelasan.
  3. Biaya baru terlihat saat klik "Lanjut ke Pembayaran" di modal.
- **Kenapa ini tidak adil:** Ini dark pattern "sneak-in fee" — biaya disembunyikan dan muncul diam-diam di langkah terakhir. Pembeli bisa merasa ditipu.
- **Cara betulinnya:** Tampilkan rincian biaya penanganan secara jelas di sidebar bersama total, bukan hanya di modal review.

**Lokasi:** `main.js:29, 120-121, 223-224`

---

## REFLEKSI

**Bedanya "kode jalan" sama "kode benar & jujur" itu apa, menurutmu, setelah level ini?**

Kode yang "jalan" hanya memenuhi syarat fungsional: tidak error, menampilkan sesuatu, user bisa klik. Tapi kode yang "benar & jujur" memenuhi syarat keamanan, akurasi data, transparansi, dan etika. Di dunia nyata (terutama yang melibatkan uang), kode yang "jalan tapi cacat" bisa merugikan banyak orang — pembeli kehilangan uang, toko kehilangan reputasi, data bocor. Tanggung jawab engineer bukan cuma bikin kode jalan, tapi bikin kode yang bisa dipercaya.
