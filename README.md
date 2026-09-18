# Grid Up Datathon — Trafo Bazlı Günlük Tüketim Tahmini

İzmir & Ege bölgesinde **7.036 dağıtım trafosunun** Nisan–Temmuz 2026 günlük aktif
tüketimini (kWh) tahmin etme problemi. Ocak 2025 – Mart 2026 arası 1,23M satır eğitim
verisi, hedef 4 ay ileride. Metrik **RMSLE** (düşük iyi). Kaggle In-Class yarışması.

> Bu repo bir skor kovalama hikâyesi değil, bir **ölçüm disiplini** hikâyesidir.
> En pahalı hatanın modelde değil **validasyonda** olduğunu; bir leaderboard'un
> cebirsel bir ölçüm aleti gibi kullanılabileceğini ve "işe yaramadı"nın da en az
> "işe yaradı" kadar değerli bir sonuç olduğunu anlatır.

---

## Problemin üç tanımlayıcı gerçeği

Bunlar varsayım değil, ölçülmüş yapısal gerçekler — çözümün tamamı bunların üstüne kuruldu:

1. **Test satırlarının %22'si cold-start.** 2.024 trafo eğitim verisinde hiç geçmiyor;
   yalnızca kurulu güç (`guc`) ve konum (`lokasyon`) biliniyor. Toplam hatanın **%63'ü**
   bu satırlardan geliyor. Cold trafoların %66'sı test penceresinin ortasında
   (2026-05-11) topluca devreye alınmış **sıfırdan yeni** varlıklar.

2. **Bu bir zaman serisi değil, bir profil + takvim problemi.** Test, eğitimin bittiği
   yerden 4 ay sonrası. Lag/rolling öznitelikleri (dünkü tüketim, son 7 gün ortalaması)
   test için üretilemez. Sinyal ancak *hedef tarihten önceki* veriden çıkarılabilir.

3. **Hatanın yarısından fazlası `tüketim = 0` satırlarından geliyor.** Bu satırlar
   verinin %4,5'i ama görülen trafolarda hatanın %47'si, cold-start'ta %62'si onlardan.
   298 trafo eğitim boyunca tamamen sıfır raporluyor. Modelin asıl işi seviye tahmini
   değil, **sıfırı yakalamak**.

---

## Sonuçlar — dürüst leaderboard geçmişi

| Sürüm | Public RMSLE | Ne değişti |
|---|---:|---|
| Taban (model yok, trafo medyanı) | 1.37798 | referans |
| v4 | 1.07899 | ayrı cold modeli + öğrenilmiş sıfır küçültmesi |
| v6 | 1.09635 | ❌ cold seviye kaydırması (kıştan yaza taşınmadı, geri alındı) |
| v8 | 1.07720 | hava durumu (iklim normali + anomali ayrıştırması) |
| v9b | 1.07264 | rejim eşleştirmesi + yaş öznitelikleri (yalnız cold modelinde) |
| **FINAL** | **1.06775** | 12 model topluluğu (tohum × mask_rate) + λ kalibrasyon |
| Tweedie cold | *yerelde en büyük kaldıraç* | cold regresörü Tweedie hedefiyle — sıfır-şişkinliği doğrudan modelliyor |

Model, taban çizgisinin (model yok) **0.31 önünde**. Yerel kazançların leaderboard'a
**%15–20 oranında** taşındığı ölçüldü — bu transfer oranı, her hamlenin gerçek değerini
tahmin etmenin anahtarı oldu.

---

## Yöntem

Dört parçalı bir reçete. Görülen (train'de olan) ve cold-start trafolar **farklı
rejimler**; tek model ikisini birden öğrenmekte zorlandığı için ayrıştırıldı.

| Parça | Model | Hedef | Öznitelikler |
|---|---|---|---|
| A | LightGBM regresör | `E[log1p(tüketim)]` | profil dahil hepsi |
| C | LightGBM regresör | cold-start seviyesi | **profilsiz** (cold'da profil yok) |
| K / KC | LightGBM sınıflandırıcı | `P(tüketim = 0)` | görülen / cold ayrı |

Nihai tahmin: `A` görülen satırlarda, `C` cold satırlarda; sıfır olasılığıyla küçültme
yalnız görülen tarafa; topluluk log uzayında ortalama.

**Sızıntısız kurulum:** her trafonun profili yalnızca *hedef tarihten önceki* veriden
hesaplanır. Eğitim, gerçek 1–4 aylık ufku taklit eden kaydırılmış zaman kesimlerinden
oluşur; `mask_rate` ile trafoların bir kısmı yapay olarak cold yapılıp modele cold rejimi
öğretilir.

**Öne çıkan öznitelik mühendisliği:**
- **Grup seviyesi istatistikleri** (`ilçe × güç` ortalaması, çeyreklikleri, sıfır oranı) —
  cold trafonun tek dayanağı komşularıdır.
- **Mevsimsellik `sin/cos` ile** — ham `dayofyear` gibi monoton-artan değişkenler ağaç
  modellerinde test aralığında son yaprağa sıkışır (ölçüldü: skor 1.00 → 1.67).
- **Hava durumu** — iklim normali (1995–2024) + anomali ayrıştırması, yarışma "31 Mart
  2026'da erişilebilir veri" kuralına uygun kuruldu (bkz. `grid_up_hava.ipynb`).
- **Yaş öznitelikleri** (yeni trafonun devreden bu yana geçen günü) — yalnız cold modelinde.

---

## Bu projenin asıl hikâyesi: ölçüm disiplini

### 1. Validasyon yanlış popülasyonu ölçüyordu

Yerel skor 0.9752, public skor 1.07899 — **+0.10 sistematik kayma**. Kaynağı tek tek
ölçüldü ve şu bulundu: cold-start'ı **yerleşik trafoları maskeleyerek** taklit ediyorduk,
oysa gerçek testin cold'ları **sıfırdan devreye alınmış yeni** trafolar. Aynı tahminciyle:

| Popülasyon | RMSLE | Sıfır oranı |
|---|---:|---:|
| Yerleşik, profili maskelenmiş (eski simülasyon) | 1.70 | %4,3 |
| Gerçekten yeni devreye alınmış (testin gerçeği) | **2.23** | %6,6 |

Fark **0.53**. Doğru validasyon harness'ı (kesimden sonra ilk kez görülen gerçek yeni
trafolar) kurulunca, sonraki tüm kararlar ilk kez testi temsil eden bir zeminde alındı.

### 2. Leaderboard'u cebirsel ölçüm aleti gibi kullanmak

Test etiketleri gizli, ama iki tahmin karışımının hata kovaryansı, ikisinin LB skorundan
ve aralarındaki farktan **tam olarak** çözülebilir. Kontrollü karışık submission'larla
(görülen = A, cold = B gibi) testin hata bütçesi çözüldü:

- Görülen taraf oracle'ın dibinde (~0.644 vs oracle 0.61) → **orada oda yok**.
- Boşluğun tamamı cold-start'ta (bizim ~1.92, liderin ~1.74).
- Cold tahminlerinin yayılımı **zaten kalibre** (optimal ölçekleme katsayısı 1.038) →
  büzme/kaydırma ailesinin tamamı kapalı.

Bu, "nereye ateş edeceğimizi" tahminle değil ölçümle belirledi.

### 3. Neyin işe yaradığını fizik söyler, yerel skor değil

En büyük yerel kaldıraç **Tweedie hedef fonksiyonu** oldu (cold RMSLE 2.20 → 2.06).
Log-MSE, sıfır-şişkin tüketimi zorla iki parçaya bölüyordu; Tweedie tek modelde doğal
modelliyor. Kritik olan: bu düzeltmenin yönü (cold seviyesini yukarı) hem yerel ölçümle
hem daha önce leaderboard'da öğrenilen dersle (v6'nın aşağı kaydırması kaybetmişti)
**aynı yönü** gösterdi — ilk kez iki bağımsız kanıt çelişmedi.

---

## Denenip elenen 14 hipotez

Portfolyonun asıl değeri burada: hangi yolun **neden** kapalı olduğunu bilmek, açık yolu
bulmak kadar değerli. Hepsi ölçüldü, `deneyler.csv`'de kayıtlı.

| # | Hipotez | Neden elendi |
|---|---|---|
| 1 | `tanım` prefix'i fider/TM kodu olarak | cold RMSLE 2.03 → 2.33 (ilçeden kaba) |
| 2 | ID uzayında en yakın komşular (kNN) | `nb_mean` ile beraberlik (1.78 vs 1.81) |
| 3 | Geçen yıl aynı takvim penceresi | optimal harman ağırlığı 0 |
| 4 | Komşu özniteliklerinde eğitim/test uyumsuzluğu | grup medyanı 28 trafo, etki 1/28 |
| 5 | Kapsama deseninden sıfır tahmini | AUC 0.51 (rastgele) |
| 6 | Yeni-trafo rampası / devreye alma partileri | sapma −0.44…+0.89 arası, tutarsız |
| 7 | Hurdle modeli `(1-p)·E[ly\|y>0]` | sınıflandırıcı kalibre değil → tabandan kötü |
| 8 | Görülen/cold için ayrık katman setleri | ince geçmiş A modelini de bozuyor |
| 9 | Fourier vekiliyle mevsim sapması tahmini | −0.23 öngörü vs +0.08 gerçek |
| 10 | Geçmiş `p_zero` oranıyla yumuşak küçültme | her α'da sert eşikten kötü |
| 11 | Hiperparametre taraması | topluluk + küçültme aynı hatayı kapatıyor |
| 12 | CatBoost karışımı | tek başına 1.01, karışımda +0.0002 |
| 13 | Tahmin kalibrasyonu | katsayı kıştan yaza taşınmıyor (ters yön) |
| 14 | Ampirik-Bayes grup büzülmesi | +0.0006 (gürültü eşiğinin altında) |

---

## Repo yapısı

```
.
├── grid_up_final.ipynb   # ⭐ Ana model — sıfırdan çalışır, kendi kendine yeter
│                         #    (ayrı cold modeli + sıfır küçültmesi + hava + topluluk)
├── grid_up_hava.ipynb    # Hava durumu modeli + yarışma kuralına uygun harici veri beyanı
├── deneyler.csv          # Deney günlüğü — hikâyenin omurgası (14 elenmiş hipotez dahil)
├── hava_ham.csv          # Open-Meteo ERA5 sıcaklık (kamuya açık, dahil)
├── requirements.txt
└── data/
    └── README.md         # Veri neden dahil değil + nasıl edinilir
```

## Çalıştırmak için

```bash
pip install -r requirements.txt
```

Yarışma verisini (`train.csv`, `test.csv`, `sample_submission.csv`) repo kök dizinine
koyun — kurallar gereği repoya dâhil değil, bkz. [`data/README.md`](data/README.md).
Sonra `grid_up_final.ipynb`'yi "Run All" ile çalıştırın (~20 dk); `submission.csv` üretir.
Kaggle'da veri yolunu kendisi bulur.

---

## Geriye dönüp bakış

Bu problem klasik bir "daha iyi model" problemi değildi. Hatanın %63'ü, `guc` ve
`lokasyon` dışında hiçbir bilgisi olmayan yeni trafolardan geliyordu ve o satırların
seviyesi bu iki değişkenden **temel olarak bilinemez** (trafo-başına std 2.1). Cebir
bunu kanıtladı: görülen taraf zaten oracle'ın dibindeydi.

Bir hafta boyunca sürdürülen şey skor optimizasyonu değil, **belirsizliğin nerede
olduğunu ölçmek** oldu. Bir modelin ne kadar iyi olduğu kadar, **ne kadar iyi
olabileceğinin sınırını bilmek** de bir mühendislik çıktısıdır — ve bu repo asıl onu
belgeliyor.
