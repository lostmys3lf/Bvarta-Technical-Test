# Bvarta Bahari — Evaluasi & Optimasi Rute Laut

Take-home Senior Data Scientist: mengevaluasi jaringan rute laut Bvarta Bahari, mengusulkan
rute baru, merekomendasikan penugasan armada, dan membangun model prediksi permintaan sebagai
dasar keputusan revenue.

## Cara menjalankan

Butuh Python 3.11. Dari root repo:

```bash
python -m venv venv
venv\Scripts\python.exe -m pip install -r requirements.txt
venv\Scripts\python.exe -m jupyter lab        # atau: jupyter notebook
```

Jalankan notebook berurutan `01` → `06` (tiap notebook berdiri sendiri, membaca dari `data/`).
Path relatif dan seed sudah ditetapkan, jadi hasil reproducible.

## Struktur

```
data/        Data mentah (read-only) — lihat DATA_DICTIONARY.md
notebooks/   Analisis (urut 01-06)
reports/     Writeup teknis + ringkasan untuk klien
outputs/     Figur & tabel hasil (dipakai di reports/ & presentasi)
requirements.txt
```

## Alur & isi notebook

| Notebook | Isi | Pertanyaan assessment |
|---|---|---|
| `01_bvarta.ipynb` | Pemahaman data, audit kualitas (missing/zero/format), diagnostik sinyal | — |
| `02_demand_forecast.ipynb` | **Model prediksi permintaan** (SARIMAX & GBM, backtest temporal, baseline, interval prediksi) | #4 (inti) |
| `03_route_evaluation.ipynb` | Evaluasi P&L rute eksisting + matriks rekomendasi | #1 |
| `04_new_routes.ipynb` | Usulan rute baru + estimasi biaya & revenue | #2 |
| `05_fleet_assignment.ipynb` | Penugasan armada (draft, gelombang, utilisasi, realokasi) | #3 |
| `06_client_output.ipynb` | Peta jaringan, tabel rekomendasi, visual (output non-teknis) | — |

## Deliverable

1. **Notebook** — `notebooks/01`–`06`.
2. **Writeup teknis (±3 hal)** — [`reports/writeup_bvarta.md`](reports/writeup_bvarta.md):
   pendekatan, metodologi & validasi model, asumsi, keterbatasan, langkah berikutnya.
3. **Output klien** — [`reports/ringkasan_klien.md`](reports/ringkasan_klien.md) + figur di `outputs/`.

## Catatan singkat

- **Constraint domain dijaga:** rute `wajib` tidak dihapus (hanya dioptimasi); jarak laut rute
  baru dikalibrasi dari faktor koreksi rute eksisting (bukan garis lurus); kelayakan kapal dicek
  terhadap draft pelabuhan dan batas gelombang; satu kapal tidak ditugaskan ke dua rute.
- **Penanganan data:** `trips=0` = tidak berlayar (bukan missing); nilai hilang pada
  `tickets_sold`/`load_factor` tidak diimputasi paksa; format tanggal `weather_wind_wave`
  (`DD/MM/YYYY`) berbeda dari file lain.
- Asumsi yang diambil ditandai eksplisit di tiap notebook dan dirangkum di writeup.
