# Evaluasi & Optimasi Jaringan Rute Laut — Bvarta Bahari
### Ringkasan Teknis (writeup)

---

## 1. Ringkasan eksekutif

Analisis ini mengevaluasi 16 rute eksisting, mengusulkan rute baru, merekomendasikan penugasan
armada, dan membangun model prediksi permintaan sebagai dasar estimasi revenue. Tiga temuan
utama:

1. **Permintaan rute padat dapat diprediksi dengan defensible.** Model permintaan harian
   mengalahkan baseline seasonal-naive ~17% (MASE 0.83 vs 1.0) dengan interval prediksi yang
   terkalibrasi (cakupan ~89% pada target 90%). Ini menjadi dasar estimasi revenue, bukan
   sekadar deskripsi.
2. **Separuh jaringan merugi, tetapi sebagian besar kerugian bersifat struktural dan bisa
   diperbaiki** — terutama lewat penugasan kapal, bukan penutupan rute.
3. **Penugasan kapal adalah pengungkit terbesar yang selama ini terabaikan.** Rate pembatalan
   tiap rute hampir sama persis dengan peluang gelombang melampaui batas operasi kapal yang
   ditugaskan (korelasi 1.0). Memilih kelas kapal yang tepat memangkas pembatalan dari ~50%
   menjadi ~2% di rute laut terbuka.

---

## 2. Pendekatan & alur kerja

Alur: pemahaman data dan audit kualitas (EDA) → model permintaan → evaluasi rute → rute baru →
penugasan armada. Prinsip yang dipegang: setiap klaim "menang/kalah" disandarkan ke tabel angka;
validasi temporal (bukan random split) untuk semua yang berbau deret waktu; dan asumsi ditandai
eksplisit. Constraint domain (rute wajib tak dihapus, jarak laut bukan garis lurus, satu kapal
tak di dua tempat, batas draft dan gelombang) diperlakukan sebagai batasan keras.

**Penanganan data.** Beberapa ketidakkonsistenan yang disengaja ditangani sadar: `trips=0`
diperlakukan sebagai "tidak berlayar" (bukan nilai hilang); `tickets_sold`/`load_factor` punya
nilai hilang akibat downtime sistem dan tidak diimputasi paksa; format tanggal `weather_wind_wave`
(`DD/MM/YYYY`) berbeda dari file lain; join lintas-level memakai `ports` sebagai jembatan
(mobilitas/hujan = region, gelombang = sea_area, pasut = port).

---

## 3. Pemodelan inti — Prediksi permintaan (load factor harian)

**Target & horizon.** Load factor harian pada empat rute terpadat (R01, R02, R04, R11), horizon
28 hari. Rute ini dipilih karena seri panjang (3 tahun) dan kepadatan data memadai; rute tipis
tidak dipaksakan dimodelkan.

**Diagnostik pra-model.** ADF/KPSS menunjukkan seri stasioner di sekitar musiman; ACF/PACF
menampilkan musiman mingguan kuat plus pengaruh kalender (mudik Idul Fitri, Nataru). Censoring
pada `LF=1` (kapal penuh) dicatat sebagai keterbatasan — permintaan riil bisa melampaui kapasitas
sehingga load factor "terpotong" di 1.

**Metode.** Dua pendekatan dibandingkan secara adil: **SARIMAX** (musiman + regresor kalender)
dan **Gradient Boosting (GBM)** dengan fitur lag/rolling/kalender. Baseline: **naive** dan
**seasonal-naive** (mingguan).

**Validasi.** Backtesting *rolling-origin* (latih pada masa lalu, uji ke depan, jendela bergeser).
Random split dilarang karena membocorkan masa depan ke masa lalu.

**Hasil (MASE relatif terhadap seasonal-naive; <1 = lebih baik):**

| Rute | naive | seasonal-naive | SARIMAX | GBM |
|---|---|---|---|---|
| R01 | 1.254 | 1.000 | **0.830** | 0.927 |
| R02 | 1.103 | 1.000 | 0.975 | **0.939** |
| R04 | 1.253 | 1.000 | **0.761** | 0.767 |
| R11 | 1.266 | 1.000 | 0.805 | **0.768** |
| **Pooled** | 1.224 | 1.000 | **0.826** | 0.834 |

Kedua model substantif mengalahkan baseline ~17%. Tidak ada pemenang mutlak: SARIMAX unggul di
R01/R04, GBM di R02/R11 — wajar mengingat karakter sinyal per rute berbeda. Bias kecil dan
terkendali (pooled +0.007 SARIMAX, +0.015 GBM).

**Analisis error.** Error naik di hari libur (MAE 0.078 vs 0.066 non-libur) — wajar karena
lonjakan mudik paling sulit diprediksi presisi. Hari sold-out (LF=1) sedikit lebih tinggi
(0.071 vs 0.068), konsisten dengan efek censoring. Error relatif stabil lintas horizon
(0.065–0.074 dari minggu ke-1 sampai ke-4), artinya model tidak cepat melar pada horizon panjang.

**Ketidakpastian.** Interval prediksi dibangun dengan *conformal* (untuk GBM) dan interval
SARIMAX. Pada target nominal 90%, cakupan empiris 88.9–89.5% dengan lebar ~0.25 load factor —
terkalibrasi baik, tidak overconfident.

**Dipakai untuk keputusan apa.** Prediksi load factor → estimasi tiket terjual → estimasi
revenue rute, dan interval prediksi memberi batas atas/bawah revenue. Ini yang menggerakkan
keputusan ekspansi kapasitas (R01, R11) dan penilaian kelayakan rute, bukan rata-rata historis
telanjang.

---

## 4. Evaluasi rute eksisting

P&L per rute disusun dari pendapatan (total tiket bulanan × tarif blended) vs `total_opex_idr`.
Hasil: **8 rute untung, 8 rugi**. Klasifikasi untung/rugi robust — tidak ada rute yang berbalik
tanda saat asumsi harga diubah (ekonomi-saja vs blended).

- **Untung & padat (ekspansi):** R01 (+76%), R11 (+51%) sering sold-out → tambah kapasitas.
- **Untung tapi jarang penuh (pertahankan/efisiensi):** R04 (+77%), R06, R07, R09, R10, R16.
- **Wajib & rugi (optimasi, bukan hapus):** R02 (−59%, load factor tinggi tapi tarif/biaya tak
  imbang), R03 (−56%). Tetap dijalankan sesuai regulasi; fokus pada efisiensi biaya/tarif.
- **Rugi berat & jarang penuh (rancang ulang/hentikan):** R13 (−187%), R14 (−283%), R15 (−119%).

---

## 5. Rekomendasi rute baru (estimasi biaya & revenue)

**Jarak laut.** Dikalibrasi dari rute eksisting: faktor koreksi = jarak laut aktual / jarak garis
lurus (haversine), median **1.27** (std 0.10, terkumpul rapat). Jarak rute baru = haversine × 1.27.

**Permintaan.** Diturunkan dari mobilitas antar-region × *capture rate* (porsi yang memakai laut).
**Keterbatasan terbesar ada di sini:** capture rate rute eksisting sangat bervariasi (0.05–12),
sehingga mobilitas adalah penanda potensi yang lemah untuk permintaan laut absolut. Karena itu
estimasi disajikan sebagai rentang (capture pesimis/tengah/optimis = 0.15/0.40/0.80), bukan titik.

**Biaya** memakai model bottom-up (bahan bakar, kru, jasa pelabuhan, perawatan) dikalibrasi dari
opex eksisting; pendapatan dan biaya variabel didiskon dengan porsi hari berlayar (1 − estimasi
pembatalan), dan kapal dipilih yang draft serta batas gelombangnya memadai.

**Hasil (margin bulanan skenario tengah, juta IDR; band = pesimis..optimis):**

| Rute baru | Koridor | Jarak | Margin tengah | Band |
|---|---|---|---|---|
| PRK–BBR | Jakarta–Palembang | 288 nm | **+4.213** | −1.298 .. +13.031 |
| SMG–PJG | Jateng–Lampung | 404 nm | **+3.142** | −1.581 .. +10.699 |
| PRK–BNO | Jakarta–Bali | 662 nm | +322 (marginal) | −4.339 .. +7.779 |
| SBY–BBR | Jatim–Sumsel | 686 nm | −4.028 (rugi) | −6.135 .. −657 |

Rekomendasi: **PRK–BBR dan SMG–PJG** layak diuji lewat *pilot* frekuensi rendah dulu untuk
memvalidasi capture rate sebelum komitmen armada penuh. Koridor jarak sangat jauh terbebani biaya
bahan bakar.

---

## 6. Penugasan armada

**Temuan kunci.** Rate pembatalan tiap rute praktis sama dengan fraksi hari gelombang di perairan
asal melampaui batas operasi kapal yang ditugaskan (korelasi 1.0, MAE 0.001 terhadap data aktual).
Artinya pembatalan bukan acak — ia konsekuensi pemilihan kelas kapal. Fast Ferry (batas 1.8 m)
batal ~47–54% hari di laut terbuka; Passenger Ship (batas 3.5 m) hanya ~0–2%.

Constraint pengikat ternyata gelombang dan kapasitas waktu, bukan draft (semua penugasan saat ini
lolos draft). Rekomendasi:

- **R05 (SMG–SBY):** ganti Fast Ferry → Passenger Ship. Estimasi pembatalan turun 54% → 2%
  (hari berlayar ~2.1× lebih banyak), kapasitas per pelayaran naik tajam.
- **R01 (wajib):** tambah kapal kedua (utilisasi 0.95 + sering sold-out).
- **R06:** untung tetapi utilisasi >1 (satu kapal tak bisa 5 keberangkatan/minggu) → kurangi ke
  4/minggu.
- **R13, R14, R15:** rugi berat → hentikan/realokasi; R12 dirancang ulang (ganti kelas kapal).
- **Rute baru:** ditugaskan Passenger Ship (laut terbuka butuh batas gelombang tinggi), bukan
  Fast Ferry.

Lima kapal dibebaskan menjadi cadangan rotasi perawatan. Fast Ferry secara struktural kurang cocok
untuk jaringan ini (setiap perairan melampaui 1.8 m pada ~40–59% hari) dan paling baik untuk
penyeberangan pendek di jendela tenang.

---

## 7. Asumsi utama

- Tarif blended = 70/25/5 (Ekonomi/Bisnis/Kabin) atau 80/20 bila tanpa Kabin; tarif baru
  dikalibrasi per-nm dari median eksisting.
- Konstanta biaya: bahan bakar 13.500 IDR/L, jasa pelabuhan 1,5 juta IDR/panggilan, perawatan
  +1,5%, keterisian target 70% untuk sizing kapasitas.
- Faktor koreksi jarak laut tunggal (1.27) untuk semua perairan.
- Capture rate rute baru di rentang 0.15–0.80.
- Rate pembatalan diestimasi dari distribusi gelombang historis per perairan (statis per kapal).

---

## 8. Keterbatasan & langkah berikutnya

- **Estimasi permintaan rute baru lemah** (capture rate tidak stabil). Perbaikan: data
  perpindahan moda atau survei asal-tujuan; itulah alasan rekomendasi pilot, bukan skala penuh.
- **Censoring LF=1** membuat permintaan puncak under-estimate. Perbaikan: model permintaan
  laten/tobit atau pakai data daftar tunggu.
- **Penugasan armada** baru sampai utilisasi mingguan; bentrok jadwal harian (turnaround,
  sinkronisasi pelabuhan) butuh penjadwalan operasional lanjutan.
- **Batas gelombang statis.** Langkah lanjut: model prediksi/klasifikasi hari tidak-layar dari
  cuaca untuk menggantikan estimasi rata-rata dengan probabilitas harian — langsung menyambung ke
  perencanaan jadwal dan SLA keandalan.
- **Pasang surut** (`tides_hourly`) belum dimodelkan; relevan untuk jendela layar aman di
  pelabuhan dangkal saat draft mepet — kandidat model deret waktu berikutnya.
- Model permintaan baru di empat rute padat; ekstensi ke rute menengah bila kepadatan data
  membaik.

---

*Reproducibility: seluruh analisis ada di `notebooks/01`–`05`, fungsi dan konstanta terdokumentasi,
seed ditetapkan, dependensi di `requirements.txt`. Jalankan via venv lokal (`venv\Scripts\python.exe`).*
