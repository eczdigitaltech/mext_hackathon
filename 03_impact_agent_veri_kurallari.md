---
inclusion: fileMatch
fileMatchPattern: "src/agents/impact/**"
---

# 03 · Impact Agent — Veri Kuralları

> **Kaynak:** nexus-data.md §2 (#3), §3, §4 (`Sevkiyatlar`, `Stok_Durumu`, `Uretim_Hatlari`, `Musteri_Siparisleri`, `Malzemeler`, `Acik_Siparisler`, `Parametreler`), §5 (Stok/tarih, Etki), §6 (what-if), §7, §8 · Docx ADIM 2, 3, 5 · Kit §4 (3. Etki)
> **Agentic eşi:** `03_impact_agent_kapsam_gorev.md`

---

## 1. Veri kaynağı kuralı

- Runtime'da yalnızca `data/nexus_agent_input.xlsx` + `SignalOutput` kullanılır.
- `Etki_Analizi` sayfası ve `Stok_Durumu` / `Musteri_Siparisleri` / `Sevkiyatlar` / `Acik_Siparisler` sayfalarındaki **ek cevap sütunları yalnızca cevap anahtarında** bulunur; runtime'da okunmaz.
- Stok olarak **yalnızca `Serbest Stok`** kullanılır. `Kalite Kontrolde` ve `Bloke Stok` kullanılmaz (Kit §8 Soru-Cevap).

---

## 2. Okunan sayfalar ve sütunlar

### 2.1 `Sevkiyatlar` — PK `Sevkiyat No` (docx ADIM 2)
`Sevkiyat No` · `PO No` · `Kuruluş Kodu` · `Malzeme No` · `Malzeme Adı` · `Nihai Varış Tesisi` · `Miktar` · `Ölçü Birimi` · `Brüt Ağırlık (kg)` · `Planlanan Varış (ETA)` · `Güncel Tahmini / Gerçekleşen Varış` · `ETA Sapması (gün)` · `Uluslararası mı?` · `Gümrük (gün)` · `İç Nakliye (gün)` · `Kalite Serbest Bırakma (gün)` · `Durum`

### 2.2 `Stok_Durumu` — PK `Stok Anahtarı` = `Tesis Kodu|Malzeme No` (docx ADIM 3)
`Stok Anahtarı` · `Malzeme No` · `Malzeme Adı` · `Tesis Kodu` · `Ölçü Birimi` · `Serbest Stok` · `Kalite Kontrolde` · `Bloke Stok` · `Min. Rezerv (servis/ruhsat)` · `Günlük Tüketim` · `Emniyet Stoğu (gün)` · `Emniyet Stoğu Miktarı`

> `Transfer Edilebilir Miktar` sütununu **Options Agent** kullanır; Impact kullanmaz.

### 2.3 `Uretim_Hatlari` — PK `Hat Kodu`
`Hat Kodu` · `Kuruluş Kodu` · `Tesis Kodu` · `Mamul No` · `Mamul Adı` · `Mamul Birimi` · `Kritik Malzeme No` · `Mamul Başına Malzeme` · `Fire Oranı` · `Planlı Günlük Üretim (mamul)` · `Günlük Malzeme İhtiyacı` · `Birim Satış Fiyatı (TRY)` · `Birim Değişken Maliyet (TRY)` · `Birim Katkı Payı (TRY)` · `Telafi Edilebilir Üretim Oranı` · `Duruşta Günlük Sabit Maliyet (TRY)` · `Yeniden Başlatma Maliyeti (TRY)` · `Günlük Duruş Maliyeti (TRY)`

### 2.4 `Musteri_Siparisleri` — PK `Sipariş No` · FK `Hat Kodu`
`Sipariş No` · `Kuruluş Kodu` · `Müşteri (Sahte)` · `Pazar` · `Hat Kodu` · `Mamul No` · `Mamul Adı` · `Miktar` · `Birim Fiyat (TRY)` · `Sipariş Tutarı (TRY)` · `Teslim Tarihi` · `Sevk Süresi (gün)` · `Gerekli Üretim Tarihi` · `Gecikme Cezası (TRY/gün)` · `Öncelik`

### 2.5 `Malzemeler` — PK `Malzeme No`
Kullanılan sütunlar: `Malzeme No` · `Malzeme Adı` · `Malzeme Kategorisi` · `Ölçü Birimi` · `ABC Kritiklik` · `Tek Kaynak mı?` · `Emniyet Stoğu (gün)` · `Gümrük Süresi (gün)` · `Giriş Kalite Serbest Bırakma (gün)` · `Özel Koşul`

### 2.6 `Acik_Siparisler` — PK `PO No` (**str**)
Kullanılan sütunlar: `PO No` · `Malzeme No` · `Tesis Kodu` · `İlk Teyitli Kullanılabilir Tarih` · `Sevkiyat No` · `Durum`

### 2.7 `Parametreler`
| Parametre | Değer | Kullanım |
|---|---|---|
| `AnalizTarihi` | 2026-11-25 | D0; gün indeksi |
| `WhatIfSenaryo` | S1 | What-if uygulanacak senaryo |
| `WhatIfGun` | 0 | Kullanılabilir tarihe eklenen ek gecikme |
| `EtkiMaliyetYuksek` | 2.000.000 TRY | Etki puanı eşiği |
| `EtkiMaliyetOrta` | 500.000 TRY | Etki puanı eşiği |

---

## 3. Join zinciri

```
SignalOutput.ilgili_sevkiyat ─► Sevkiyatlar (Sevkiyat No)
      ├─ Malzeme No + Nihai Varış Tesisi ─► Stok_Durumu (Stok Anahtarı = "Tesis|Malzeme")
      ├─ Malzeme No ─► Malzemeler
      ├─ PO No ─► Acik_Siparisler
      └─ Kritik Malzeme No = Malzeme No & Tesis Kodu = Nihai Varış Tesisi ─► Uretim_Hatlari (Hat Kodu)
                                                                             └─► Musteri_Siparisleri (Hat Kodu)
```
S1: `SV-26-1184` → `MT100054` → `T-SG-01|MT100054` → `HAT-SG-03` → müşteri siparişleri.

---

## 4. Hesap kuralları (deterministik tool'lar)

| # | Hesap | Formül | S1 kontrolü |
|---|---|---|---|
| I1 | Günlük ihtiyaç | `Planlı Günlük Üretim (mamul)` × `Mamul Başına Malzeme` × (1 + `Fire Oranı`) | 8.500 kutu × 7 g × (1 + fire) → **60,69 kg/gün** (`Uretim_Hatlari` 2. satır K sütunu; `Stok_Durumu.Günlük Tüketim` ile tutarlı) |
| I2 | Stok kapsama | `Serbest Stok` ÷ `Günlük Tüketim` | 600 ÷ 60,69 = **9,89 gün** |
| I3 | Tükenme tarihi | D0 + kapsama (kesirli gün, saat dahil) | **04.12 21:16** |
| I4 | Kullanılabilir tarih | `Güncel Tahmini / Gerçekleşen Varış` + `Gümrük (gün)` + `İç Nakliye (gün)` + `Kalite Serbest Bırakma (gün)` (+ `WhatIfGun`, senaryo = `WhatIfSenaryo` ise) | 05.12 + 1 + 1 + 3 = **10.12** |
| I5 | Planlanan kullanılabilir tarih | `Planlanan Varış (ETA)` + aynı süreler | 27.11 + 5 = 02.12 |
| I6 | Kullanılabilir gün indeksi | (kullanılabilir tarih − D0).days | **15. gün** |
| I7 | Açık / duruş | max(0, kullanılabilir gün − kapsama) | 15 − 9,89 = **5,11 gün** |
| I8 | Günlük duruş maliyeti | üretim × katkı payı × (1 − telafi oranı) + boşta sabit maliyet. Girdideki `Günlük Duruş Maliyeti (TRY)` ile tutarlılık kontrol edilir. | **424.550 TL/gün** |
| I9 | Duruş maliyeti | açık × `Günlük Duruş Maliyeti (TRY)` + (açık > 0 ise **bir kez**) `Yeniden Başlatma Maliyeti (TRY)` | **2.291.018 TL** |
| I10 | Risk altındaki sipariş | `Gerekli Üretim Tarihi` stok bitişi ile malzeme gelişi arasına düşüyorsa → "Risk Altında" | **4 sipariş** |
| I11 | Müşteri cezası | gecikme günü × `Gecikme Cezası (TRY/gün)`. Gecikme günü tanımı `Formuller` sayfasından birebir alınır. | **450.000 TL** |
| I12 | Aksiyonsuz maliyet | duruş maliyeti + müşteri cezaları | **2.741.018 TL** |
| I13 | Etki puanı | Açık varsa: ≥ `EtkiMaliyetYuksek` → 5 · ≥ `EtkiMaliyetOrta` → 4 · altı → 3. Açık yoksa: tampon < emniyet stoğu → 2, aksi → 1 | **5** |

**İlaç (GMP) kuralı:** API, kalite onayı olmadan kullanılamaz. Bu yüzden **limana varış tarihi ≠ kullanılabilir tarih** (docx ADIM 2: "Asıl tarih 10 Aralık").

**Süre tutarlılığı:** Gümrük ve kalite süreleri `Sevkiyatlar`'dan alınır; `Malzemeler.Gümrük Süresi (gün)` ve `Giriş Kalite Serbest Bırakma (gün)` ile tutarlılık kontrol edilir. Fark varsa uyarı üretilir.

---

## 5. Okuma tuzakları

- Tarihler Excel datetime'dır. Tükenme gibi hesaplar kesirli gün (saat) içerir; yuvarlama yalnızca gösterimde yapılır.
- Birimler karışıktır: `TON` (1 ton = 1000 kg brüt), `KG`, `ADET`. Birim dönüşümü olmadan karşılaştırma yapılmaz.
- Depolarda (`D-...`) `Günlük Tüketim = 0`'dır; kapsama hesabı yalnızca tüketen tesis (fabrika) için yapılır.
- `"-"` → `None`.

---

## 6. Çıktı sözleşmesi — `ImpactOutput`

> **Kanonik şema:** `00_ortak_veri_semalari.md` → `ImpactOutput`. Aşağıdaki örnekte zarf alanları (`run_id`, `d0`, `durum` vb.) kısaltılmıştır. Alan adı, tip ve zorunluluk için şema esastır.

```json
{
  "risk_no": "R-01",
  "senaryo": "S1",
  "sevkiyat_no": "SV-26-1184",
  "malzeme_no": "MT100054",
  "tesis_kodu": "T-SG-01",
  "hat_kodu": "HAT-SG-03",
  "serbest_stok": 600,
  "gunluk_tuketim": 60.69,
  "stok_kapsama_gun": 9.89,
  "tukenme_tarihi": "2026-12-04T21:16",
  "planlanan_kullanilabilir_tarih": "2026-12-02",
  "guncel_kullanilabilir_tarih": "2026-12-10",
  "kullanilabilir_gun": 15,
  "acik_durus_gun": 5.11,
  "gunluk_durus_maliyeti_try": 424550,
  "yeniden_baslatma_maliyeti_try": "<girdi>",
  "durus_maliyeti_try": 2291018,
  "risk_altindaki_siparisler": [{"siparis_no": "SO-26-5xxx", "gecikme_gun": "<tool>", "ceza_try": "<tool>"}],
  "olasi_ceza_try": 450000,
  "aksiyonsuz_maliyet_try": 2741018,
  "etki_puani": 5,
  "kaynak": [{"sayfa": "Sevkiyatlar", "anahtarlar": ["SV-26-1184"]}, {"sayfa": "Stok_Durumu", "anahtarlar": ["T-SG-01|MT100054"]}, {"sayfa": "Uretim_Hatlari", "anahtarlar": ["HAT-SG-03"]}],
  "uyarilar": []
}
```
Gösterilen sayılar S1 için **beklenen** değerlerdir; runtime'da tool'lar hesaplar.

---

## 7. Test eşlemesi (cevap anahtarı)

| Çıktı alanı | Cevap anahtarı sayfa → sütun | S1 beklenen |
|---|---|---|
| `stok_kapsama_gun` | `Stok_Durumu → Stok Kapsama (gün)` | 9,89 |
| `tukenme_tarihi` | `Stok_Durumu → Tükenme Tarihi` | 04.12 21:16 |
| (ops.) | `Stok_Durumu → Emniyet Stoğu Altına İnme Tarihi`, `Stok Sağlığı` | sayfadan |
| `planlanan_kullanilabilir_tarih` | `Sevkiyatlar → Planlanan Kullanılabilir Tarih` | 02.12 |
| `guncel_kullanilabilir_tarih` | `Sevkiyatlar → Güncel Kullanılabilir Tarih` | 10.12 |
| (ops.) | `Acik_Siparisler → Güncel Beklenen Kullanılabilir Tarih`, `Kayma (gün)` | sayfadan |
| `acik_durus_gun` | `Etki_Analizi → Açık / Duruş (gün)` | 5,11 |
| `durus_maliyeti_try` | `Etki_Analizi → Duruş Maliyeti` | 2.291.018 |
| `olasi_ceza_try` | `Etki_Analizi → Olası Müşteri Cezası` | 450.000 |
| `aksiyonsuz_maliyet_try` | `Etki_Analizi → Aksiyonsuz Toplam Maliyet` | 2.741.018 |
| `risk_altindaki_siparisler` | `Musteri_Siparisleri → Etki Durumu` = "Risk Altında", `Tahmini Gecikme (gün)`, `Olası Ceza (TRY)` | 4 satır |
| `etki_puani` | `Etki_Analizi` etki puanı / `Risk_Skorlama → Etki` | 5 |

Toleranslar: gün ±0,01 · TL ±1.

**Genelleme (nexus-data.md §6):** S2 (SV-26-1203 / MT100001 / HAT-KR-01, T-KR-01) → ~14 gün duruş, ~38,1 M TL · S3 (SV-26-1219 / MT100032 / HAT-BN-04, T-BN-01) → ~9,2 gün, ~8,07 M TL.
