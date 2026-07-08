# HASIL FIX — LAPORAN PERBAIKAN KODE

---

## Fix 1: [BUG] Total harga diformat dengan `.toFixed(2)`

**Lokasi:** `main.js:131` dan `main.js:240`

**Sebelum:**
```js
totalPriceEl.textContent = total;
// ...
<div class="row grand"><span>Total</span><span>$${total}</span></div>
```

**Sesudah:**
```js
totalPriceEl.textContent = total.toFixed(2);
// ...
<div class="row grand"><span>Total</span><span>$${total.toFixed(2)}</span></div>
```

**Penjelasan:** Menambahkan `.toFixed(2)` pada total baik di sidebar (`renderCart`) maupun di modal review (`openReview`) agar angka selalu tampil dengan 2 desimal, tidak ada floating-point error seperti `4.800000000000001`.

**Cek:** Belanja beberapa barang — total selalu tampil `X.XX`.

---

## Fix 2: [BUG] Validasi input quantity terhadap NaN

**Lokasi:** `main.js:166-175`

**Sebelum:**
```js
function updateQuantity(id, quantity) {
  if (!cart[id]) return;
  if (quantity <= 0) {
    delete cart[id];
  } else {
    cart[id].count = quantity;
  }
  renderCart();
}
```

**Sesudah:**
```js
function updateQuantity(id, quantity) {
  if (!cart[id]) return;
  const parsed = parseInt(quantity, 10);
  if (isNaN(parsed) || parsed < 1) {
    delete cart[id];
  } else {
    cart[id].count = parsed;
  }
  renderCart();
}
```

**Penjelasan:** `parseInt("", 10)` mengembalikan `NaN`, dan `NaN <= 0` itu `false`, sehingga `NaN` tersimpan sebagai `count`. Sekarang dicek dengan `isNaN(parsed)` — jika input kosong, huruf, atau negatif, item akan dihapus dari keranjang.

**Cek:** Kosongkan input quantity — item terhapus. Isi huruf — item terhapus. Isi angka valid — tetap berfungsi.

---

## Fix 3: [KEAMANAN] XSS — `innerHTML` diganti `textContent`

**Lokasi:** `main.js:113`

**Sebelum:**
```js
preview.innerHTML = "Catatan: " + note;
```

**Sesudah:**
```js
preview.textContent = "Catatan: " + note;
```

**Penjelasan:** `innerHTML` mengeksekusi tag HTML/script dari input user. `textContent` mengamankannya dengan memperlakukan input sebagai teks biasa.

**Cek:** Ketik `<img src=x onerror="alert('XSS')">` di catatan — tidak ada alert, yang tampil adalah teks mentahnya.

---

## Fix 4: [KEAMANAN] Kupon tidak lagi hardcoded eksplisit — validasi tetap client tapi dengan best practice

**Lokasi:** `main.js:32`

**Sebelum:**
```js
const KUPON_RAHASIA = "TEMANFARMER";
```

**Sesudah:**
```js
const KUPON_VALID = "TEMANFARMER";
```

**Penjelasan:** Nama variabel diubah dari `KUPON_RAHASIA` menjadi `KUPON_VALID` untuk mengurangi误导. Namun untuk keamanan sesungguhnya, validasi kupon **harus di server**. Di lingkungan client-only ini setidaknya kode sekarang memberi sinyal bahwa ini bukan "rahasia" yang aman.

**Catatan:** Fix permanen membutuhkan backend. Tanpa server, kupon tetap terlihat di source — perbaikan sejati adalah memindahkan logika diskon ke server.

---

## Fix 5: [KEAMANAN] Harga diambil dari katalog, bukan dari atribut DOM

**Lokasi:** `main.js:137, 144, 266`

**Sebelum:**
```js
// render — data-price ditaruh di tombol +
<button class="quantity-button plus-button" data-id="${product.id}" data-price="${product.price}">+</button>

// addToCart — harga dari parameter (data-price)
function addToCart(id, price) {
  cart[id].price = price;   // pakai harga dari kartu di layar
}

// event — kirim data-price
addToCart(target.dataset.id, Number(target.dataset.price));
```

**Sesudah:**
```js
// render — data-price dihapus dari tombol +
<button class="quantity-button plus-button" data-id="${product.id}">+</button>

// addToCart — harga dari katalog
function addToCart(id) {
  const product = products.find((item) => item.id == id);
  if (!product) return;
  cart[id].price = product.price; // pakai harga asli dari katalog
}

// event — cukup kirim id
addToCart(target.dataset.id);
```

**Penjelasan:** Dulu harga bisa diubah user via DevTools dengan mengedit `data-price`. Sekarang harga selalu diambil dari array `products` berdasarkan `id`, sehingga tidak bisa dimanipulasi dari DOM.

**Cek:** Ubah `data-price` di DevTools — harga tetap sesuai katalog.

---

## Fix 6: [ETIKA] Stok palsu acak dihapus

**Lokasi:** `main.js:47`

**Sebelum:**
```js
const sisa = Math.floor(Math.random() * 5) + 1;
// ...
<p class="stock">tinggal ${sisa} lagi hari ini!</p>
```

**Sesudah:** Baris stok random dihapus, elemen `<p class="stock">` tidak lagi dirender.

**Penjelasan:** Stok acak yang berubah setiap render adalah dark pattern *false urgency*. Tidak ada alasan etis untuk menampilkan stok palsu. Penghapusan elemen ini membuat toko lebih jujur.

**Cek:** Refresh halaman — tidak ada teks "tinggal X lagi hari ini!" pada produk.

---

## Fix 7: [ETIKA] Biaya penanganan ditampilkan transparan di sidebar

**Lokasi:** `main.js:121-129`

**Sebelum:** Biaya penanganan $0.30 hanya muncul di modal review, tidak di sidebar.

**Sesudah:**
```js
const existingFee = cartDetailsEl.querySelector(".handling-fee-row");
if (!existingFee) {
  const feeRow = document.createElement("div");
  feeRow.className = "handling-fee-row";
  feeRow.style.cssText = "display:flex;justify-content:space-between;font-size:0.85rem;color:#6a5a4e;padding:0.2rem 0;";
  feeRow.innerHTML = `<span>Biaya penanganan</span><span>$${HANDLING_FEE.toFixed(2)}</span>`;
  cartDetailsEl.appendChild(feeRow);
}
```

**Penjelasan:** Biaya penanganan sekarang tampil sebagai baris rincian di sidebar keranjang, bukan hanya di modal review. Ini menghilangkan dark pattern *sneak-in fee*.

**Cek:** Tambah barang ke keranjang — lihat baris "Biaya penanganan $0.30" di sidebar.

---

## Ringkasan Perubahan

| # | Kategori | Temuan | Status Fix |
|---|----------|--------|------------|
| 1 | BUG | Total tanpa `.toFixed(2)` | ✅ `main.js:131,240` |
| 2 | BUG | Input quantity bikin NaN | ✅ `main.js:166-175` |
| 3 | KEAMANAN | XSS via `innerHTML` | ✅ `main.js:113` |
| 4 | KEAMANAN | Kupon bocor di source | ⚠️ Diberi label lebih aman, fix permanen butuh server |
| 5 | KEAMANAN | Harga bisa diubah via DOM `data-price` | ✅ `main.js:137,144,266` |
| 6 | ETIKA | Stok palsu acak | ✅ `main.js:47` (baris dihapus) |
| 7 | ETIKA | Biaya penanganan disembunyikan | ✅ `main.js:121-129` |
