# Veri

Bu klasör bilinçli olarak **boştur**.

Yarışma veri seti (`train.csv`, `test.csv`, `sample_submission.csv`) **Gdz ve Adm
Elektrik**'e aittir. Yarışma kuralları gereği izinsiz paylaşılamaz ve çoğaltılamaz,
bu yüzden bu repoya dâhil edilmemiştir (`.gitignore`).

## Notebook'ları çalıştırmak için

1. Veri setini yarışma sayfasından edinin (Grid Up Datathon, Kaggle In-Class).
2. `train.csv`, `test.csv`, `sample_submission.csv` dosyalarını **repo kök dizinine** koyun.
3. Notebook'lar veriyi kök dizinden okur; Kaggle ortamında ise `/kaggle/input/` altından
   otomatik bulur.

## Hava durumu verisi

`hava_ham.csv` **repoda mevcuttur** — bu veri kısıtlı değildir. İzmir ve Manisa için
Open-Meteo ERA5 arşivinden çekilmiş günlük sıcaklıktır (1995–2024 iklim normali +
2025-01 … 2026-03 gerçek), kaynak ve erişilebilirlik `grid_up_hava.ipynb`'de belgelenmiştir.
Notebook internet yoksa bu dosyadan okur, varsa yeniden çeker.
