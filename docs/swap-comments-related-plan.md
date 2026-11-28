# Rencana Implementasi: Tukar Komentar & Rekomendasi Video (StreamTube)

Menambahkan opsi kustom di halaman **Settings → Tweaks** untuk menukar posisi kolom komentar dan rekomendasi video (*Up next*) pada halaman tonton (`WatchPage`), terinspirasi dari ekstensi Chrome **Show YouTube Comments While Watching**.

## Ringkasan Keputusan Desain

| Aspek | Keputusan |
| ----- | ----- |
| **Lokasi Setting** | Tab **Tweaks** (`Section::Tweaks` di `src/ui/pages/settings.rs`) |
| **Nama Field Setting** | `swap_comments_related: bool` (default: `false`) |
| **Scroll Komentar (Sidebar)** | Panel ber-scroll independen (`comments_scroll: ScrollHandle`) agar video di kiri tetap diam saat membaca komentar |
| **Tampilan Rekomendasi (Bawah Video)** | Tetap **list vertikal** (`compact_row`) seperti bawaan `render_related` |
| **Mode Aktif** | Hanya pada **mode normal** (`!theater`) dan layout **2 kolom** (`two_col`, lebar tersedia `> 1000px`) |
| **Mode Theater & Window Sempit** | Kembali ke urutan bawaan tanpa perubahan |

## File yang Akan Dimodifikasi

| File | Perubahan Utama |
| ----- | ----- |
| `src/store.rs` | Tambah field `swap_comments_related: bool` pada struct `Settings` dan `Default::default()` |
| `src/ui/pages/settings.rs` | Tambah toggle switch baru di method `tweaks()` |
| `src/ui/pages/watch.rs` | Tambah `comments_scroll: ScrollHandle`, pisahkan logika render & infinite scroll saat `swap_layout` aktif, serta rapikan layout header komentar untuk lebar `402px` |

## Rincian Perubahan per File

### 1. `src/store.rs`

* Pada struct `Settings` (bagian `// Tweaks`, sekitar baris 129–160), tambahkan:

  ```rust
  /// Show comments in the right sidebar and related videos below the player (normal 2-column mode)
  pub swap_comments_related: bool,
  ```

* Pada `impl Default for Settings` (sekitar baris 267–281), set nilai awal:

  ```rust
  swap_comments_related: false,
  ```

* Karena `Settings` memakai `#[serde(default)]`, file `settings.json` pengguna lama tetap kompatibel tanpa migrasi tambahan.

### 2. `src/ui/pages/settings.rs`

* Di dalam `fn tweaks(&self, cx: &mut Context<Self>) -> Div` (baris 587–629), tambahkan satu baris switch:

  ```rust
  .child(row(
      "Swap comments and Up next",
      "Show comments beside the video in a scrollable sidebar and move Up next below (normal mode)",
      switch("swcr", s.swap_comments_related, |s, v| s.swap_comments_related = v, cx),
      cx,
  ))
  ```

* Tidak perlu perubahan di `after_change()`: switch otomatis memanggil `store::update_settings` dan `refresh_all_pages(cx)`, sehingga perubahan langsung terasa di `WatchPage` yang sedang terbuka.

### 3. `src/ui/pages/watch.rs`

#### A. State Scroll Independen

* Pada struct `WatchPage` (baris 41–78), tambahkan field:

  ```rust
  comments_scroll: ScrollHandle,
  ```

* Inisialisasi di `WatchPage::new()` dengan `comments_scroll: ScrollHandle::new()`.

#### B. Kondisi Aktif & Infinite Scroll

* Di awal `WatchPage::render()` (setelah menghitung `theater`, `avail`, dan `two_col`):

  ```rust
  let swap_layout = settings.swap_comments_related && !theater && two_col;
  ```

* Sesuaikan blok infinite scroll (baris 1885–1899):
  * **Trigger komentar (`load_comments`)**:
    * Jika `swap_layout`: cek `near_bottom(&self.comments_scroll, 300.0)` (saat pengguna scroll di dalam panel komentar sidebar).
    * Jika `!swap_layout`: tetap cek `near_bottom(&self.scroll, 600.0)` seperti semula.
  * **Trigger rekomendasi (`load_more_related`)**:
    * Tetap cek `near_bottom(&self.scroll, 600.0)` (baik saat di sidebar maupun saat di bawah video, keduanya berada di dalam `self.scroll`).

#### C. Penyesuaian `render_comments()` untuk Mode Sidebar (`compact: bool`)

* Ubah signature internal `render_comments(&mut self, w: &WatchInfo, compact: bool, cx: &mut Context<Self>) -> AnyElement` (atau buat helper `render_comments_sidebar()` yang memanggil komponen header + body terpisah).
* Ketika `compact == true` (berada di sidebar kanan lebar `402px`):
  * **Container luar**: kotak ber-border mengikuti gaya `render_chat()` (`flex().flex_col().rounded_xl().border_1().border_color(p.border).overflow_hidden()`) dengan tinggi tetap adaptif terhadap viewport, misalnya `h((vp.height - px(96.)).max(px(480.)))`, agar pengguna dapat menggulir komentar secara mandiri sementara player di kiri tetap terlihat penuh.
  * **Header sticky di atas**: berisi jumlah komentar (`text_base` / `text_lg`) dan chip filter (`Top`, `Timed`, `Newest`) dengan `.flex_wrap()` dan `.gap_2()` serta `border_b_1().border_color(p.border)` agar tidak overflow pada lebar `402px`.
  * **Body scroll independen**: area tombol tulis komentar/editor + daftar komentar dibungkus:

    ```rust
    div()
        .id("watch-comments-scroll")
        .flex_1()
        .min_h_0()
        .overflow_y_scroll()
        .track_scroll(&self.comments_scroll)
        .flex()
        .flex_col()
        .gap_4()
        .px_3()
        .py_3()
    ```

  * Jika `settings.hide_comments` aktif, kembalikan elemen kosong (jangan tampilkan kotak border kosong di sidebar).
  * Pada `render_comment_t()` (baris 1098–1160), badan komentar sudah memakai `.flex_1().min_w_0()` sehingga teks komentar otomatis wrap dengan aman di lebar `402px`. Saat `compact == true`, gap avatar dapat diperkecil sedikit (misalnya `gap_3` jika perlu) agar ruang baca lebih lega.

#### D. Penyusunan Kolom Kiri & Kanan di `WatchPage::render()`

* **Saat `swap_layout == true`** (hanya berlaku di cabang `!theater` & `two_col`):
  * **Kolom kiri (`primary_info`)**: `render_info(w, cx)` diikuti `div().pt_4().child(render_related(w, cx))` (tetap vertikal dengan `compact_row`).
  * **Kolom kanan (`secondary`)**: `chapters_panel` → `chat_panel` → `queue_panel` → `comments_sidebar` (jika `!settings.hide_comments`).
* **Saat `swap_layout == false`** (toggle mati, Theater mode, atau window sempit `<= 1000px`):
  * Tetap menggunakan urutan asli: kolom kiri `render_info` + `render_comments`, kolom kanan `chapters_panel` + `chat_panel` + `queue_panel` + `render_related`.

## Rencana Pengujian & Verifikasi

1. **Build & Test**:
   * Jalankan `cargo build --release --locked` dan `cargo test --release --locked` untuk memastikan tidak ada error kompilasi maupun regresi unit test.
2. **Uji Manual UI**:
   * **Toggle di Settings → Tweaks**: aktifkan/nonaktifkan *Swap comments and Up next*, pastikan tersimpan di `settings.json` setelah restart dan langsung berefek saat kembali ke halaman video.
   * **Mode Normal (lebar > 1000px)**: pastikan komentar muncul di sidebar kanan dalam panel scroll independen, video tetap diam saat komentar di-scroll, infinite scroll memuat komentar lanjutan saat mendekati dasar panel, dan *Up next* muncul vertikal di bawah deskripsi.
   * **Header & Wrapping di 402px**: cek chip `Top / Timed / Newest`, editor komentar, balasan (*replies*), dan komentar panjang agar tidak terpotong secara horizontal.
   * **Mode Theater**: masuk Theater mode saat toggle aktif, pastikan layout kembali ke urutan normal (tidak tertukar).
   * **Window Sempit (<= 1000px)**: perkecil jendela menjadi 1 kolom, pastikan layout fallback dengan rapi tanpa elemen tumpang tindih.
   * **Kombinasi Setting Lain**: uji bersama `hide_comments`, `hide_related`, Live Chat, dan panel Chapters/Queue untuk memastikan tidak ada kotak kosong atau bentrok tinggi.
