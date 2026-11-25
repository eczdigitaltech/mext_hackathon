---
inclusion: always
---

# NEXUS – Veri Rehberi (Kiro steering)

> **Amaç:** Bu doküman, NEXUS agent'larını geliştirirken hangi verinin **hangi dosyanın hangi sayfasında** olduğunu, sayfaların **yapısını, anahtarlarını ve ilişkilerini** anlatır. Kod üretirken sütun adlarını **birebir** buradan al; uydurma sütun adı kullanma.
>
> Tüm şirket, tedarikçi, kişi ve rakamlar **sahte ama tutarlıdır**. Analiz tarihi (**D0 / "bugün"**) = **2026-11-25**. Tüm "gün" hesapları bu tarihten başlar (`Parametreler → AnalizTarihi`).

---

## 1. İki dosya, iki farklı rol (EN ÖNEMLİ KURAL)

| Rol | Orijinal dosya adı | Önerilen repo yolu | İçerik | Agent'lar okuyabilir mi? |
|---|---|---|---|---|
| **GİRDİ (runtime verisi)** | `NEXUS_Agent_Veri_Seti_TEKNİK (1).xlsx` | `data/nexus_agent_input.xlsx` | 15 veri sayfası + `00_Rehber`. Sabit değerler, formül yok. **Cevap yok.** | ✅ **Evet – agent'ların tek veri kaynağı** |
| **CEVAP ANAHTARI (test oracle)** | `NEXUS_Hackathon_Veri_Seti_Ana veri.xlsx` | `tests/fixtures/nexus_answer_key.xlsx` | Aynı ana veri + tüm agent çıktıları (risk puanı, etki, stratejiler, simülasyon, MILP, karar kaydı, KPI), formüller ve rehber. | ❌ **Hayır – yalnızca test/doğrulama ve formül referansı** |

**Kurallar:**
1. Agent kodu runtime'da **yalnızca GİRDİ dosyasını** okur. Cevap anahtarındaki hesaplanmış değerler (risk puanı, duruş günü, strateji sırası vb.) agent'a **verilmez, hardcode edilmez**; agent bunları kendisi hesaplar.
2. Cevap anahtarı; (a) birim/entegrasyon testlerinde beklenen değer, (b) formül mantığını öğrenmek için (`Formuller` sayfası) kullanılır.
3. Dosya adlarındaki boşluk ve Türkçe `İ` karakteri Unicode normalizasyon sorunu çıkarabiliyor → dosyaları yukarıdaki ASCII adlarla yeniden adlandır.

---

## 2. Hangi agent hangi veriyi nerede bulur?

| # | Agent | GİRDİ dosyasında okuduğu sayfalar | Üretmesi gereken çıktı | Doğru cevabın olduğu yer (cevap anahtarı) | LLM / Kod |
|---|---|---|---|---|---|
| 0 | **Orkestratör** | Tümü | Akış/durum makinesi: Sinyal → Risk → Etki → Seçenek → Karar → Bildirim | `Agent_Haritasi` | Deterministik |
| 1 | **Signal Agent** | `Risk_Sinyalleri` (**`Ham Metin (LLM Girdisi)`**), `Risk_Kaynaklari`, `Sevkiyatlar` (Takip No eşleştirme), `Acik_Siparisler` (PO No), `Tedarikciler`, `Tasiyicilar` | Olay tipi, ülke/bölge, ilgili taraf (TDxxxx/TSxx), ilgili sevkiyat (SV-...), malzeme (MT...), tahmini gecikme (gün), metindeki alternatif ipucu, sinyal güvenilirliği | `Risk_Sinyalleri` → `(LLM)` ekli sütunlar + `Sinyal Güvenilirliği (1-5)`, `Senaryo`, `Bağlı Risk No` | LLM (çıkarım) + kural (eşleştirme) |
| 2 | **Risk Assessment Agent** | Signal çıktısı, `Tedarikciler`, `Tasiyicilar`, `Parametreler` (W_*, Esik*), `Esik_Tablolari` | Olasılık (1-5), Etki (1-5), Risk Puanı (1-25), Kademe, Agent Aksiyonu, açıklama | `Risk_Skorlama` | Deterministik puanlama + LLM açıklama |
| 3 | **Impact Analysis Agent** | `Sevkiyatlar`, `Stok_Durumu`, `Uretim_Hatlari`, `Musteri_Siparisleri`, `Malzemeler`, `Parametreler` | Kullanılabilir tarih, stok kapsama, tükenme tarihi, açık/duruş günü, duruş maliyeti, risk altındaki siparişler, ceza, aksiyonsuz maliyet, etki puanı | `Etki_Analizi`; `Musteri_Siparisleri` → `Etki Durumu`, `Olası Ceza (TRY)`; `Stok_Durumu` → `Tükenme Tarihi` | Deterministik (formül) |
| 4a | **Sourcing Options Agent** | `Tedarik_Kaynaklari`, `Stok_Durumu` (`Transfer Edilebilir Miktar`), `Malzemeler` (kalifikasyon, özel koşul) | Transfer / alternatif tedarikçi / hazır stok seçenekleri, fiyat farkı, ek malzeme maliyeti | `Strateji_Bilesenleri` | Kural + LLM |
| 4b | **Logistics Options Agent** | `Rota_Secenekleri`, `Tasiyicilar`, `Sevkiyatlar`, `Tesisler` | Rota/mod/liman değişikliği, ekspres; hazır olma günü, lojistik maliyeti, CO2, uygulama riski | `Strateji_Bilesenleri` | Deterministik (rota matrisi) + LLM |
| 5 | **Optimization & Decision Agent** | 4a/4b çıktıları, `Parametreler` (W_Maliyet, W_Durus, W_CO2, W_Risk, PlanlamaTamponu, WhatIf*) | Gün gün stok simülasyonu, toplam maliyet, ÇKKV ağırlıklı skor, sıralama, önerilen strateji, (S1 için) MILP | `Zaman_Simulasyonu`, `Strateji_Karsilastirma`, `Optimizasyon_Modeli` | Deterministik (simülasyon + MILP); LLM yalnızca gerekçe metni |
| 6 | **Notification Agent** | 5'in çıktısı + Etki sonucu | Rol bazlı özet, onay kaydı, KPI | `Karar_Kaydi`, `KPI_Panel` | LLM (özet) + iş akışı |

> **Prensip:** Sayısal her şey (puan, tarih, maliyet, optimizasyon) **deterministik kodla** hesaplanır. LLM yalnızca ham metinden bilgi çıkarımı, eşleştirme ipucu ve gerekçe/özet metni üretir.

---

## 3. Veri modeli (ilişkiler)

```
Risk_Kaynaklari (Kaynak Kodu) ──< Risk_Sinyalleri (Sinyal No)
                                        │  Ham Metin içinde geçen: Takip No / PO No / gemi adı / tedarikçi adı
                                        ▼
Tasiyicilar (Taşıyıcı No) ──< Sevkiyatlar (Sevkiyat No, Takip No) >── Acik_Siparisler (PO No)
Tedarikciler (Tedarikçi No) ──<     │                                        │
                                    │ Malzeme No, Nihai Varış Tesisi          │ Malzeme No, Tesis Kodu
                                    ▼                                        ▼
Malzemeler (Malzeme No) ──< Stok_Durumu (Tesis Kodu|Malzeme No) >── Tesisler (Tesis Kodu)
        │                                                                   │
        ├──< Tedarik_Kaynaklari (Malzeme No|Tedarikçi No) >── Tedarikciler  │
        └──< Uretim_Hatlari (Hat Kodu; Kritik Malzeme No, Tesis Kodu) ───────┘
                    │
                    └──< Musteri_Siparisleri (Sipariş No; Hat Kodu)

Rota_Secenekleri (Rota Kodu) >── Tasiyicilar (Taşıyıcı No)
Parametreler (key-value) · Esik_Tablolari (lookup)  → tüm agent'lar
```

**Kod formatları:**
- Kuruluş: `KARO`, `BANYO`, `MADEN`, `SAGLIK`
- Tesis: `T-<KR|BN|MD|SG>-0n` (fabrika/saha), `D-<..>-01` (depo) – 12 kayıt
- Malzeme: `MT1000xx` · Mamul: `MM3000xx` · Hat: `HAT-<KR|BN|SG|MD>-0n`
- Tedarikçi: `TD5xxx` · Taşıyıcı: `TS01–TS09` · Sinyal kaynağı: `KY01–KY12`
- Sinyal: `SG-2611-0xx` (güncel), `SG-2610-0xx` (geçmiş), `...H` ekli (kapatılmış)
- Sevkiyat: `SV-26-xxxx` · Takip No: `<taşıyıcı>-<çıkış>-26-xxxx` (örn. `AKH-MUN-26-1184`) · PO: `45002xxxxx`
- Rota: `R-<MOD>-<ÇIKIŞ>-<VARIŞ>` (SEA, AIR, TRK, COD, CST, EXP, RAIL)
- Müşteri siparişi: `SO-26-5xxx` · Risk: `R-01…R-08` · Strateji: `S<senaryo>-<KOD>` (örn. `S1-HIB`) · Karar: `NXS-26-0xx`

---

## 4. GİRDİ dosyası – sayfa şemaları (`data/nexus_agent_input.xlsx`)

Tüm sayfalarda **başlık 1. satır, veri 2. satırdan** başlar → `pd.read_excel(path, sheet_name=None)` doğrudan çalışır. (`00_Rehber` serbest metindir, veri değildir.)

### `Risk_Sinyalleri` — 25 satır
- **Ne:** Gelen ham sinyaller (e-posta, advisory, AIS, haber, ERP istisnası). **`Ham Metin (LLM Girdisi)`** LLM'in okuyacağı tek serbest metin alanıdır. Dil: EN / TR / UK→TR / ES→TR (çeviri). Durum: Açık – Analizde, İzlemede, Bilgi, Değerlendiriliyor, Yanlış Alarm, Kapatıldı.
- **Birincil anahtar:** Sinyal No
- **Bağlantılar:** Kaynak Kodu → Risk_Kaynaklari
- **Kullanan agent:** Signal
- **Sütunlar:** `Sinyal No` · `Yayın Zamanı` · `Tespit Zamanı` · `Kaynak Kodu` · `Kaynak Adı` · `Kategori` · `Dil` · `Ham Metin (LLM Girdisi)` · `Kaynak Taban Güvenilirlik` · `Doğrulandı mı?` · `Durum`

### `Risk_Kaynaklari` — 12 satır
- **Ne:** Sinyal kaynak envanteri + `Taban Güvenilirlik (1-5)`.
- **Birincil anahtar:** Kaynak Kodu (KY01–KY12)
- **Bağlantılar:** —
- **Kullanan agent:** Signal, Risk
- **Sütunlar:** `Kaynak Kodu` · `Kaynak Adı` · `Kaynak Tipi` · `Taban Güvenilirlik (1-5)` · `Erişim Tipi` · `Güncelleme Sıklığı` · `Agent Okuma Yöntemi` · `Gerçek Hayattaki Karşılığı`

### `Sevkiyatlar` — 14 satır
- **Ne:** Yoldaki / planlı / teslim edilmiş sevkiyatlar. **`Takip No (Sahte)`** e-posta metinlerinde geçen numaradır (örn. `AKH-MUN-26-1184` → `SV-26-1184`).
- **Birincil anahtar:** Sevkiyat No (SV-26-xxxx)
- **Bağlantılar:** PO No → Acik_Siparisler · Malzeme No → Malzemeler · Tedarikçi No → Tedarikciler · Taşıyıcı No → Tasiyicilar · Nihai Varış Tesisi → Tesisler
- **Kullanan agent:** Signal, Risk & Impact, Options
- **Sütunlar:** `Sevkiyat No` · `PO No` · `Kuruluş Kodu` · `Malzeme No` · `Malzeme Adı` · `Tedarikçi No` · `Taşıyıcı No` · `Taşıma Modu` · `Yük Tipi` · `Çıkış Noktası` · `Varış Noktası` · `Nihai Varış Tesisi` · `Miktar` · `Ölçü Birimi` · `Brüt Ağırlık (kg)` · `Çıkış Tarihi (ETD)` · `Planlanan Varış (ETA)` · `Güncel Tahmini / Gerçekleşen Varış` · `ETA Sapması (gün)` · `Uluslararası mı?` · `Gümrük (gün)` · `İç Nakliye (gün)` · `Kalite Serbest Bırakma (gün)` · `Durum` · `Son Konum / Not` · `Sapma Nedeni` · `Navlun (USD)` · `Takip No (Sahte)`

### `Acik_Siparisler` — 10 satır
- **Ne:** Açık satınalma siparişleri (PO). Metinlerde `your PO 45002xxxxx` şeklinde geçer.
- **Birincil anahtar:** PO No
- **Bağlantılar:** Sevkiyat No → Sevkiyatlar · Malzeme No · Tedarikçi No · Tesis Kodu
- **Kullanan agent:** Risk & Impact
- **Sütunlar:** `PO No` · `Kuruluş Kodu` · `Tesis Kodu` · `Malzeme No` · `Malzeme Adı` · `Tedarikçi No` · `Tedarikçi Adı` · `PO Tarihi` · `Miktar` · `Ölçü Birimi` · `Birim Fiyat` · `Para Birimi` · `Tutar (USD)` · `İlk Teyitli Kullanılabilir Tarih` · `Sevkiyat No` · `Durum`

### `Stok_Durumu` — 10 satır
- **Ne:** Tesis/depo bazında serbest, QC, bloke stok; günlük tüketim; `Transfer Edilebilir Miktar` (grup içi transfer seçeneği buradan).
- **Birincil anahtar:** Stok Anahtarı = `Tesis Kodu|Malzeme No`
- **Bağlantılar:** Malzeme No · Tesis Kodu
- **Kullanan agent:** Risk & Impact, Options
- **Sütunlar:** `Stok Anahtarı` · `Malzeme No` · `Malzeme Adı` · `Tesis Kodu` · `Ölçü Birimi` · `Serbest Stok` · `Kalite Kontrolde` · `Bloke Stok` · `Min. Rezerv (servis/ruhsat)` · `Günlük Tüketim` · `Emniyet Stoğu (gün)` · `Emniyet Stoğu Miktarı` · `Transfer Edilebilir Miktar` · `Stok Değeri (USD)`

### `Uretim_Hatlari` — 8 satır
- **Ne:** Hat → mamul → kritik malzeme (BOM katsayısı), günlük ihtiyaç ve **`Günlük Duruş Maliyeti (TRY)`**.
- **Birincil anahtar:** Hat Kodu
- **Bağlantılar:** Tesis Kodu → Tesisler · Kritik Malzeme No → Malzemeler · Mamul No
- **Kullanan agent:** Risk & Impact
- **Sütunlar:** `Hat Kodu` · `Kuruluş Kodu` · `Tesis Kodu` · `Mamul No` · `Mamul Adı` · `Mamul Birimi` · `Kritik Malzeme No` · `Mamul Başına Malzeme` · `Fire Oranı` · `Planlı Günlük Üretim (mamul)` · `Günlük Malzeme İhtiyacı` · `Birim Satış Fiyatı (TRY)` · `Birim Değişken Maliyet (TRY)` · `Birim Katkı Payı (TRY)` · `Telafi Edilebilir Üretim Oranı` · `Duruşta Günlük Sabit Maliyet (TRY)` · `Yeniden Başlatma Maliyeti (TRY)` · `Günlük Duruş Maliyeti (TRY)`

### `Musteri_Siparisleri` — 22 satır
- **Ne:** Müşteri siparişleri: teslim tarihi, sevk süresi, `Gerekli Üretim Tarihi`, `Gecikme Cezası (TRY/gün)`, öncelik.
- **Birincil anahtar:** Sipariş No
- **Bağlantılar:** Hat Kodu → Uretim_Hatlari · Mamul No
- **Kullanan agent:** Risk & Impact
- **Sütunlar:** `Sipariş No` · `Kuruluş Kodu` · `Müşteri (Sahte)` · `Pazar` · `Hat Kodu` · `Mamul No` · `Mamul Adı` · `Miktar` · `Birim Fiyat (TRY)` · `Sipariş Tutarı (TRY)` · `Teslim Tarihi` · `Sevk Süresi (gün)` · `Gerekli Üretim Tarihi` · `Gecikme Cezası (TRY/gün)` · `Öncelik`

### `Malzemeler` — 7 satır
- **Ne:** Malzeme ana verisi: brüt ağırlık, emniyet stoğu (gün), gümrük ve giriş kalite süreleri, özel koşul (GMP/ADR vb.), yeni kaynak kalifikasyon süresi.
- **Birincil anahtar:** Malzeme No (MT1000xx)
- **Bağlantılar:** Ana Tedarikçi → Tedarikciler
- **Kullanan agent:** Tümü
- **Sütunlar:** `Malzeme No` · `Kuruluş Kodu` · `Malzeme Adı` · `Malzeme Kategorisi` · `Ölçü Birimi` · `ABC Kritiklik` · `Tek Kaynak mı?` · `Ana Tedarikçi` · `Standart İniş Maliyeti (USD/birim)` · `Birim Maliyet (TRY)` · `Brüt Ağırlık (kg/birim)` · `Değer Yoğunluğu (USD/kg)` · `Emniyet Stoğu (gün)` · `Gümrük Süresi (gün)` · `Giriş Kalite Serbest Bırakma (gün)` · `Özel Koşul` · `Yeni Kaynak Kalifikasyonu (gün)`

### `Tedarikciler` — 11 satır
- **Ne:** Tedarikçi ana verisi + karne: OTIF, ort. gecikme, finansal not, ESG, `Performans Puanı (1-5)`, `Finansal Puan (1-5)`.
- **Birincil anahtar:** Tedarikçi No (TD5xxx)
- **Bağlantılar:** —
- **Kullanan agent:** Risk & Impact, Options
- **Sütunlar:** `Tedarikçi No` · `Tedarikçi Adı (Sahte)` · `Ülke` · `Şehir/Bölge` · `Ana Kategori` · `İlgili Kuruluş` · `Statü` · `Incoterm` · `Para Birimi` · `Finansal Sağlık Notu` · `Son 6 Ay Teslimat Sayısı` · `Son 6 Ay OTIF Teslimat` · `OTIF Oranı` · `Ort. Gecikme (gün)` · `ESG Skoru (0-100)` · `Performans Puanı (1-5)` · `Finansal Puan (1-5)`

### `Tasiyicilar` — 9 satır
- **Ne:** Taşıyıcılar ve zamanında varış performansı, finansal not, puanlar.
- **Birincil anahtar:** Taşıyıcı No (TS01–TS09)
- **Bağlantılar:** —
- **Kullanan agent:** Risk & Impact, Options
- **Sütunlar:** `Taşıyıcı No` · `Taşıyıcı Adı (Sahte)` · `Taşıma Modu` · `Hizmet Bölgesi` · `Son 6 Ay Sevkiyat` · `Zamanında Varış` · `Zamanında Varış Oranı` · `Ort. Sapma (gün)` · `Finansal Sağlık Notu` · `Sözleşme Tipi` · `Performans Puanı (1-5)` · `Finansal Puan (1-5)`

### `Tesisler` — 12 satır
- **Ne:** Tesis ve depolar, koordinat, en yakın liman ve kargo havalimanı.
- **Birincil anahtar:** Tesis Kodu (T-xx-0n fabrika / D-xx-0n depo)
- **Bağlantılar:** —
- **Kullanan agent:** Options
- **Sütunlar:** `Tesis Kodu` · `Kuruluş Kodu` · `Tesis Adı` · `Tesis Tipi` · `İl` · `İlçe/Bölge` · `Enlem` · `Boylam` · `En Yakın Liman` · `Liman Mesafesi (km)` · `En Yakın Kargo Havalimanı`

### `Tedarik_Kaynaklari` — 12 satır
- **Ne:** Malzeme × tedarikçi: fiyat, para birimi, Incoterm, iniş maliyeti, hazır stok, ek kapasite, Spot MOQ, kalifikasyon durumu/süresi.
- **Birincil anahtar:** Kaynak Anahtarı = `Malzeme No|Tedarikçi No`
- **Bağlantılar:** Malzeme No → Malzemeler · Tedarikçi No → Tedarikciler
- **Kullanan agent:** Options (Sourcing)
- **Sütunlar:** `Kaynak Anahtarı` · `Malzeme No` · `Tedarikçi No` · `Tedarikçi Adı` · `Ana Kaynak mı?` · `Kota Payı` · `Birim Fiyat` · `Para Birimi` · `Birim Fiyat (USD)` · `Incoterm` · `Navlun Fiyata Dahil mi?` · `Standart Navlun+Liman (USD/birim)` · `İniş Maliyeti (USD/birim)` · `Standart Tedarik Süresi (gün)` · `Acil Sevk Hazırlık (gün)` · `Hazır Stok (birim)` · `Ek Üretim Kapasitesi (birim/gün)` · `Spot MOQ (birim)` · `Kalifikasyon Durumu` · `Kalifikasyon Süresi (gün)` · `Çıkış Lokasyonu`

### `Rota_Secenekleri` — 18 satır
- **Ne:** Lojistik rota matrisi: mod, kapı-kapı süre, değişken/sabit maliyet, kapasite, güvenilirlik, mesafe, emisyon faktörü, uygulama riski.
- **Birincil anahtar:** Rota Kodu (R-...)
- **Bağlantılar:** Taşıyıcı No → Tasiyicilar
- **Kullanan agent:** Options (Logistics)
- **Sütunlar:** `Rota Kodu` · `Çıkış` · `Varış` · `Taşıma Modu` · `Taşıyıcı No` · `Hazırlık/Booking (gün)` · `Transit + Son Teslim (gün)` · `Kapı-Kapı Süre (gün)` · `Gümrük Var mı?` · `Değişken Maliyet (USD/kg)` · `Sabit Maliyet (USD/sevkiyat)` · `Sevkiyat Kapasitesi (kg)` · `Zamanında Varış Güvenilirliği` · `Mesafe (km)` · `Emisyon Faktörü (kgCO2e/ton-km)` · `Uygulama Riski (1-5)` · `Not`

### `Parametreler` — 24 satır
- **Ne:** Key–value tablo: kurlar, ağırlıklar, eşikler, tampon, what-if, test maliyetleri.
- **Birincil anahtar:** Parametre (ad)
- **Bağlantılar:** —
- **Kullanan agent:** Tümü
- **Sütunlar:** `Parametre` · `Değer` · `Birim` · `Açıklama`

### `Esik_Tablolari` — 14 satır
- **Ne:** Puanlama eşikleri (performans, finansal, rota riski).
- **Birincil anahtar:** Tablo + Alt Sınır / Not
- **Bağlantılar:** —
- **Kullanan agent:** Risk & Impact, Options
- **Sütunlar:** `Tablo` · `Alt Sınır / Not` · `Puan (1-5)` · `Nasıl kullanılır?`

### Parametreler – değerler
| Parametre | Değer | Birim |
|---|---|---|
| `AnalizTarihi` | 2026-11-25 | tarih |
| `USDTRY` | 45.9 | TRY |
| `EURTRY` | 47.4 | TRY |
| `EURUSD` | 1.0327 | USD |
| `StokTasimaOrani` | 0.3 | %/yıl |
| `PlanlamaTamponu` | 2 | gün |
| `KorumaUfku` | 7 | gün |
| `WhatIfSenaryo` | S1 | kod |
| `WhatIfGun` | 0 | gün |
| `QCTestMaliyeti` | 25000 | TRY/parti |
| `KilLabTestGunluk` | 3500 | TRY/gün |
| `ElektronikMuayeneMaliyeti` | 8000 | TRY/parti |
| `W_Sinyal` | 0.4 | ağırlık |
| `W_Performans` | 0.3 | ağırlık |
| `W_DisKosul` | 0.2 | ağırlık |
| `W_Finansal` | 0.1 | ağırlık |
| `EsikYuksek` | 15 | puan |
| `EsikOrta` | 7 | puan |
| `EtkiMaliyetYuksek` | 2000000 | TRY |
| `EtkiMaliyetOrta` | 500000 | TRY |
| `W_Maliyet` | 0.45 | ağırlık |
| `W_Durus` | 0.3 | ağırlık |
| `W_CO2` | 0.1 | ağırlık |
| `W_Risk` | 0.15 | ağırlık |

---

## 5. Kritik iş kuralları ve formüller (kısa)

Tam liste: cevap anahtarı → `Formuller` sayfası (53 formül, sade dil + S1 örneği + Excel formülü).

**Maliyet / kur**
- USD fiyat: USD ise aynen · EUR ise × `EURUSD` · TRY ise ÷ `USDTRY`
- İniş maliyeti = USD fiyat + (`Navlun Fiyata Dahil mi?` = "Hayır" ise) `Standart Navlun+Liman (USD/birim)`. Fiyat karşılaştırması **daima iniş maliyetiyle**; DAP'ta navlun ikinci kez sayılmaz.

**Stok / tarih**
- Günlük ihtiyaç = Planlı Günlük Üretim × Mamul Başına Malzeme × (1 + Fire Oranı)
- Kapsama (gün) = Serbest Stok ÷ Günlük Tüketim · Tükenme tarihi = D0 + kapsama
- **Kullanılabilir tarih = Güncel Tahmini Varış + Gümrük + İç Nakliye + Kalite Serbest Bırakma (+ `WhatIfGun`)**. İlaçta (GMP) API kalite onayı olmadan kullanılamaz.
- Transfer edilebilir = stok − rezerv − kendi emniyet stoğu − `KorumaUfku` (7) günlük kendi ihtiyacı

**Risk puanı**
- Sinyal güvenilirliği = kaynak taban puanı + 1 (`Doğrulandı mı?` = "Evet"), maks. 5
- Performans puanı (OTIF / zamanında varış): ≥0,95→1 · ≥0,90→2 · ≥0,85→3 · ≥0,75→4 · altı→5 (1 iyi, 5 kötü)
- Finansal puan: A→1 · B→2 · C→3 · D→5
- **Olasılık = 0,40×sinyal + 0,30×performans + 0,20×dış koşul + 0,10×finansal** (`W_*` parametreleri). *Dış koşul puanı girdi dosyasında yoktur → agent/LLM bağlamdan gerekçeli belirler.*
- Risk = Olasılık × Etki · Kademe: ≥15 Yüksek (tam analiz + onaya sun) · ≥7 Orta (uyarı + seçenek hazırda) · altı Düşük (izle)

**Etki**
- Açık/duruş günü = max(0, kullanılabilir gün (D0+) − kapsama)
- Duruş maliyeti = açık × `Günlük Duruş Maliyeti (TRY)` + (açık > 0 ise bir kez) `Yeniden Başlatma Maliyeti (TRY)`
- Günlük duruş maliyeti = üretim × katkı payı × (1 − telafi oranı) + boşta sabit maliyet
- Müşteri siparişi risk altında: `Gerekli Üretim Tarihi` stok bitişi ile malzeme gelişi arasına düşüyorsa · Ceza = gecikme günü × günlük ceza
- Aksiyonsuz maliyet = duruş maliyeti + müşteri cezaları
- Etki puanı: açık varsa ≥2 M TL→5 · ≥0,5 M TL→4 · altı→3; açık yoksa tampon < emniyet stoğu→2, aksi→1

**Seçenekler / karar**
- Hazır günü = tedarikçi hazırlığı + kalifikasyon + rota kapı-kapı + gümrük + kalite onayı
- Lojistik maliyet = (USD/kg × brüt kg + sabit × sevkiyat sayısı) × USDTRY; navlun dahilse 0
- CO2 (kg) = brüt ton × km × emisyon faktörü
- Kısıtlar: transfer ≤ transfer edilebilir · miktar ≥ Spot MOQ · miktar ≤ hazır stok + kapasite · kalifikasyon gerektiren kaynak süre içinde kullanılamaz
- Hibrit miktar = (ana sevkiyat günü + `PlanlamaTamponu`) × günlük tüketim − stok − transfer; yukarı yuvarla, ≥ MOQ
- Simülasyon: günlük kova (gün başı giriş, gün içi tüketim), kısmi duruş kesirli gün
- Toplam maliyet = duruş + ceza + lojistik + ek malzeme + test (`QCTestMaliyeti` vb.) + stok taşıma
- ÇKKV skoru = 0,45×maliyet + 0,30×duruş + 0,10×CO2 + 0,15×risk (senaryo içinde 0-1 normalize: (en kötü − bu) ÷ (en kötü − en iyi)); 1. sıra = ÖNERİLEN

---

## 6. Gömülü senaryolar (verinin "hikâyesi")

| Senaryo | Kuruluş | Olay | Ana sinyaller | Sevkiyat / Malzeme / Hat | Beklenen sonuç (cevap anahtarı) |
|---|---|---|---|---|---|
| **S1 (ana demo)** | SAGLIK | Klaritromisin API konteyneri Cidde aktarmasında 8 gün gecikti | SG-2611-001, -002 | SV-26-1184 / MT100054 / HAT-SG-03, T-SG-01 | ~5,1 gün duruş, aksiyonsuz ~2,74 M TL → **S1-HIB**: D-SG-01 transferi + Zhejiang (TD5012) kısmi hava, duruş 0, ~0,11 M TL. COD (Mersin) pahalı → seçilmez |
| **S2** | KARO | Chornomorsk limanı kapandı, 6.000 t kil 18 gün gecikti | SG-2611-004, -005, -006, -018 | SV-26-1203 / MT100001 / HAT-KR-01, T-KR-01 | ~14 gün duruş, ~38,1 M TL → **S2-HIB-A**: Uşak (T-KR-02) transferi + Çan yerli kil (TD5002), ~1,39 M TL |
| **S3** | BANYO | Shenzhen enerji kısıtı, kontrol kartı sevki 12 gün kaydı | SG-2611-007, -008 | SV-26-1219 / MT100032 / HAT-BN-04, T-BN-01 | ~9,2 gün duruş, ~8,07 M TL → **S3-HIB**: servis deposu (D-BN-01) transferi + HK ekspres, ~0,06 M TL. Almanya (TD5023) kalifikasyon gerektirdiği için **elenmeli** |
| İzleme | çeşitli | R-04 (sonraki parti OOS), R-05 (Ambarlı yoğunluk), R-06 (tek sinyal → ikinci malzeme), R-07 (Castellón grevi), R-08 | SG-2611-003, -009, -010, -011, -012 | — | Orta/Düşük kademe → uyarı veya izleme |
| Gürültü | — | Yanlış alarm, kapatılmış, bilgi amaçlı, geçmiş sinyaller | SG-2611-013…-017, SG-2610-*, *H | — | Agent bunları **analize sokmamalı / ayıklamalı** |

**What-if:** `Parametreler → WhatIfSenaryo` (S1/S2/S3) + `WhatIfGun` (ek gecikme günü). Kod bu iki parametreyi okuyup kullanılabilir tarihe eklemeli; tüm zincir (etki → simülasyon → sıralama) yeniden hesaplanmalı.

---

## 7. CEVAP ANAHTARI dosyası – nerede ne var (`tests/fixtures/nexus_answer_key.xlsx`)

**Girdi sayfalarına göre fazladan (cevap) sütunlar** – teste karşılaştırma için:

| Sayfa | Cevap anahtarında olup girdide olmayan sütunlar |
|---|---|
| `Risk_Sinyalleri` | `Tespit Gecikmesi (saat)`, `Olay Tipi (LLM)`, `Ülke/Bölge (LLM)`, `İlgili Taraf (LLM)`, `İlgili Sevkiyat (LLM)`, `İlgili Malzeme (LLM)`, `Tahmini Gecikme (gün) (LLM)`, `Alternatif İpucu (LLM)`, `Sinyal Güvenilirliği (1-5)`, `Senaryo`, `Bağlı Risk No` |
| `Sevkiyatlar` | `Senaryo`, `Planlanan Kullanılabilir Tarih`, `Güncel Kullanılabilir Tarih` |
| `Acik_Siparisler` | `Güncel Beklenen Kullanılabilir Tarih`, `Kayma (gün)` |
| `Stok_Durumu` | `Stok Kapsama (gün)`, `Tükenme Tarihi`, `Emniyet Stoğu Altına İnme Tarihi`, `Stok Sağlığı` |
| `Musteri_Siparisleri` | `Etki Durumu`, `Tahmini Gecikme (gün)`, `Olası Ceza (TRY)` |

**Yalnızca cevap anahtarında olan sayfalar:**

| Sayfa | İçerik | Okuma notu |
|---|---|---|
| `Okuma_Rehberi` | Proje özeti, demo akışı, senaryolar, varsayımlar | Serbest metin |
| `Formuller` | 53 formül (sade dil, örnek, sayfa→sütun, agent, Excel formülü) | Başlık **4. satır** (`header=3`); bölüm başlıkları ayrı satırlardır |
| `Risk_Skorlama` | R-01…R-08: puan bileşenleri, Olasılık, Etki, Risk Puanı, Kademe, Agent Aksiyonu, LLM açıklama örneği | Başlık 1. satır · PK `Risk No` |
| `Etki_Analizi` | Risk başına stok, kapsama, kullanılabilir gün, açık, duruş maliyeti, risk altındaki sipariş/ceza, aksiyonsuz maliyet, etki puanı | Başlık 1. satır · PK `Risk No` |
| `Strateji_Bilesenleri` | 36 satır: her stratejinin bileşenleri (kaynak, rota, miktar, gün, maliyet kalemleri, CO2, risk, `Kısıt Kontrolü`) | PK `Strateji Kodu` + `Bileşen No` |
| `Zaman_Simulasyonu` | 18 strateji × 61 gün (D0…D60) stok simülasyonu | **Geniş format:** satır 2 strateji kodu/adı, 3 senaryo, 4 başlangıç stoğu, 5 günlük tüketim, 6 ufuk, 7 toplam duruş; **satır 8 başlık**, satır 9-69 günler. Her strateji 3 sütun: `Giriş`, `Gün Sonu Stok`, `Duruş Oranı` |
| `Strateji_Karsilastirma` | 18 strateji: maliyet kalemleri, duruş, CO2, risk, normalize puanlar, ağırlıklı skor, `Sıra`, `Öneri` | Başlık 1. satır · PK `Strateji Kodu` |
| `Optimizasyon_Modeli` | S1 MILP: parametreler, karar değişkenleri (x_T transfer, x_A hava, y_C COD), ara hesaplar, amaç, kısıtlar, Solver ayarları, yorum | Form düzeni (tablo değil); 7 numaralı bölüm |
| `Karar_Kaydi` | 27 olay (2026 pilot + onay bekleyen): zaman damgaları, karar (Onaylandı/Reddedildi/Revize Edildi/Onay Bekliyor), tasarruf, süreler | Başlık 1. satır · PK `Olay No` |
| `KPI_Panel` | Tespit/öneri/onay süreleri, yanlış alarm oranı vb. + hedef + manuel kıyas | Başlık **4. satır** |
| `Veri_Kontrolleri` | Otomatik tutarlılık testleri (GEÇTİ/KALDI) | Başlık **3. satır** – kendi test suite'ine örnek |
| `Agent_Haritasi` | Agent → okuduğu sayfa → ürettiği alan → LLM/deterministik → tetikleyici → örnek çıktı | Başlık **3. satır** |
| `Veri_Sozlugu` | Sayfa, katman, PK/FK, gerçek hayattaki kaynak sistem (ERP/TMS/WMS), güncelleme sıklığı | Başlık **3. satır** |

> Cevap anahtarı formüllü bir dosyadır; pandas/openpyxl (`data_only=True`) **önbelleğe alınmış değerleri** okur. Girdi verisini değiştirirsen cevap anahtarındaki değerler otomatik güncellenmez.

---

## 8. Okuma tuzakları ve veri kalitesi notları

- **`PO No` pandas'ta int okunur** → join'lerden önce `dtype={"PO No": str}` ver; metindeki `PO 4500201290` ile string eşleştir.
- **Takip No ≠ Sevkiyat No:** e-postalarda `AKH-MUN-26-1184` geçer; `Sevkiyatlar.Takip No (Sahte)` üzerinden `Sevkiyat No`'ya map et. Gemi adı (`MV AEGEAN STAR`) `Son Konum / Not` sütununda geçer.
- **Bir sinyal → birden çok malzeme** olabilir (cevapta `"MT100032; MT100031"`, ayraç `"; "`). Çıktı şemanı liste olarak tasarla.
- **`"-"` boş değer işaretidir** (`Senaryo`, `İlgili Taraf` vb.) → `None`'a çevir.
- **Birimler karışık:** `TON` (kil; 1 ton = 1000 kg brüt), `KG`, `ADET`. Lojistik hesabı daima **brüt kg** (`Malzemeler.Brüt Ağırlık (kg/birim)` × miktar) üzerinden.
- **Para birimi:** `Birim Fiyat` orijinal para biriminde (USD/EUR/TRY); USD sütunları ayrıca verilmiş. Sonuç maliyetler **TRY**.
- **Tarihler** Excel datetime; tükenme tarihi gibi hesaplar kesirli gün (saat) içerir. Gün indeksi = (tarih − 2026-11-25).days.
- **Dil:** ham metinler EN/TR; `UK→TR`, `ES→TR` çevrilmiş metinlerdir.
- Gürültü sinyalleri (`Yanlış Alarm`, `Kapatıldı`, `Bilgi`, `SG-2610-*`) bilinçli eklenmiştir; filtreleme mantığı test edilecek.
- `Stok_Durumu`'nda depolar (`D-...`) **günlük tüketim = 0**; transfer kaynağı olarak kullanılır.
- `Tedarik_Kaynaklari.Kalifikasyon Durumu` = "Kalifikasyon Gerekli" olan kaynak (örn. TD5023) `Kalifikasyon Süresi (gün)` yüzünden ufuk içinde çözüm olamaz → elenmeli.
- İlaç (SAGLIK) malzemelerinde yalnızca **ruhsatta kayıtlı** kaynak kullanılabilir (`Malzemeler.Özel Koşul`).

---

## 9. Hızlı başlangıç (Python)

```python
import pandas as pd

INPUT = "data/nexus_agent_input.xlsx"            # agent'lar SADECE bunu okur
ANSWER = "tests/fixtures/nexus_answer_key.xlsx"  # SADECE testler okur

data = pd.read_excel(INPUT, sheet_name=None, dtype={"PO No": str})
data.pop("00_Rehber", None)

params = dict(zip(data["Parametreler"]["Parametre"], data["Parametreler"]["Değer"]))
D0 = pd.Timestamp(params["AnalizTarihi"])        # 2026-11-25

stok = data["Stok_Durumu"].set_index("Stok Anahtarı")       # "T-SG-01|MT100054"
kaynak = data["Tedarik_Kaynaklari"].set_index("Kaynak Anahtarı")  # "MT100054|TD5012"
sevk = data["Sevkiyatlar"].set_index("Sevkiyat No")
takip2sevk = dict(zip(data["Sevkiyatlar"]["Takip No (Sahte)"], data["Sevkiyatlar"]["Sevkiyat No"]))

# Test örneği (pytest):
# exp = pd.read_excel(ANSWER, sheet_name="Etki_Analizi").set_index("Risk No")
# assert abs(my_impact["R-01"].durus_gun - exp.loc["R-01", "Açık / Duruş (gün)"]) < 0.01   # ≈ 5.11
```
