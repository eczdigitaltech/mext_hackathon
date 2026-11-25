---
inclusion: fileMatch
fileMatchPattern: "src/agents/options/**"
---

# 04 · Options Agent — Veri Kuralları

> **Kaynak:** nexus-data.md §2 (#4a Sourcing, #4b Logistics), §3, §4 (`Tedarik_Kaynaklari`, `Stok_Durumu`, `Malzemeler`, `Tedarikciler`, `Rota_Secenekleri`, `Tasiyicilar`, `Sevkiyatlar`, `Tesisler`, `Esik_Tablolari`, `Parametreler`), §5 (Maliyet/kur, Seçenekler), §7 (`Strateji_Bilesenleri`), §8 · Docx ADIM 6 · Kit §4 (4. Seçenekler)
> **Agentic eşi:** `04_options_agent_kapsam_gorev.md`

Tek agent, iki alt modül: **Sourcing** (a: başka depoda var mı, b: başka tedarikçi var mı) ve **Logistics** (c: nasıl hızlı getiririz). Bkz. Kapsam kararı K3.

---

## 1. Veri kaynağı kuralı

- Runtime'da yalnızca `data/nexus_agent_input.xlsx` + `SignalOutput` + `ImpactOutput` kullanılır.
- `Strateji_Bilesenleri` (36 satır) **yalnızca cevap anahtarında** bulunur; runtime'da okunmaz.
- Fiyat karşılaştırması **daima iniş maliyetiyle** yapılır; sonuç maliyetler **TRY**'dir.

---

## 2. Sourcing alt modülü — okunan veriler

### 2.1 `Stok_Durumu` — PK `Stok Anahtarı` (docx ADIM 6a)
Kullanılan sütunlar: `Stok Anahtarı` · `Malzeme No` · `Tesis Kodu` · `Ölçü Birimi` · `Serbest Stok` · `Min. Rezerv (servis/ruhsat)` · `Günlük Tüketim` · `Emniyet Stoğu Miktarı` · **`Transfer Edilebilir Miktar`** · `Stok Değeri (USD)`
- Grup içi transfer kaynağı: aynı `Malzeme No`, farklı `Tesis Kodu` (genellikle depo `D-...`, `Günlük Tüketim = 0`).
- Kural: transfer edilebilir = stok − rezerv − kendi emniyet stoğu − `KorumaUfku` (7) günlük kendi ihtiyacı. Girdideki sütun kullanılır, formülle tutarlılık kontrol edilir.

### 2.2 `Tedarik_Kaynaklari` — PK `Kaynak Anahtarı` = `Malzeme No|Tedarikçi No` (docx ADIM 6b)
`Kaynak Anahtarı` · `Malzeme No` · `Tedarikçi No` · `Tedarikçi Adı` · `Ana Kaynak mı?` · `Kota Payı` · `Birim Fiyat` · `Para Birimi` · `Birim Fiyat (USD)` · `Incoterm` · `Navlun Fiyata Dahil mi?` · `Standart Navlun+Liman (USD/birim)` · `İniş Maliyeti (USD/birim)` · `Standart Tedarik Süresi (gün)` · `Acil Sevk Hazırlık (gün)` · `Hazır Stok (birim)` · `Ek Üretim Kapasitesi (birim/gün)` · `Spot MOQ (birim)` · `Kalifikasyon Durumu` · `Kalifikasyon Süresi (gün)` · `Çıkış Lokasyonu`

### 2.3 `Malzemeler` — PK `Malzeme No`
Kullanılan sütunlar: `Malzeme No` · `Kuruluş Kodu` · `Malzeme Kategorisi` · `Ölçü Birimi` · `Tek Kaynak mı?` · `Ana Tedarikçi` · `Standart İniş Maliyeti (USD/birim)` · `Birim Maliyet (TRY)` · `Brüt Ağırlık (kg/birim)` · `Gümrük Süresi (gün)` · `Giriş Kalite Serbest Bırakma (gün)` · **`Özel Koşul`** · **`Yeni Kaynak Kalifikasyonu (gün)`**

### 2.4 `Tedarikciler` — PK `Tedarikçi No`
Kullanılan sütunlar: `Tedarikçi No` · `Ülke` · `Statü` · `Performans Puanı (1-5)` · `Finansal Puan (1-5)`

---

## 3. Logistics alt modülü — okunan veriler

### 3.1 `Rota_Secenekleri` — 18 satır · PK `Rota Kodu` (`R-<MOD>-<ÇIKIŞ>-<VARIŞ>`; mod: SEA, AIR, TRK, COD, CST, EXP, RAIL) (docx ADIM 6c)
`Rota Kodu` · `Çıkış` · `Varış` · `Taşıma Modu` · `Taşıyıcı No` · `Hazırlık/Booking (gün)` · `Transit + Son Teslim (gün)` · `Kapı-Kapı Süre (gün)` · `Gümrük Var mı?` · `Değişken Maliyet (USD/kg)` · `Sabit Maliyet (USD/sevkiyat)` · `Sevkiyat Kapasitesi (kg)` · `Zamanında Varış Güvenilirliği` · `Mesafe (km)` · `Emisyon Faktörü (kgCO2e/ton-km)` · `Uygulama Riski (1-5)` · `Not`

### 3.2 `Tasiyicilar` — PK `Taşıyıcı No`
Kullanılan sütunlar: `Taşıyıcı No` · `Taşıma Modu` · `Hizmet Bölgesi` · `Zamanında Varış Oranı` · `Performans Puanı (1-5)`

### 3.3 `Sevkiyatlar` — mevcut (gecikmeli) sevkiyat, liman değişikliği (COD) için
Kullanılan sütunlar: `Sevkiyat No` · `Taşıyıcı No` · `Taşıma Modu` · `Varış Noktası` · `Nihai Varış Tesisi` · `Güncel Tahmini / Gerçekleşen Varış` · `Brüt Ağırlık (kg)` · `Navlun (USD)`

### 3.4 `Tesisler` — PK `Tesis Kodu`
Kullanılan sütunlar: `Tesis Kodu` · `Tesis Tipi` · `İl` · `En Yakın Liman` · `Liman Mesafesi (km)` · `En Yakın Kargo Havalimanı`

### 3.5 `Esik_Tablolari`
Rota riski puan eşikleri (`Tablo` · `Alt Sınır / Not` · `Puan (1-5)` · `Nasıl kullanılır?`).

---

## 4. Önceki agent'lardan girdiler

| Kaynak | Alan | Kullanım |
|---|---|---|
| `SignalOutput` | `alternatif_ipucu` | Metindeki gizli seçenek (S1: Mersin'e liman değişikliği → COD rotası) |
| `ImpactOutput` | `malzeme_no`, `tesis_kodu`, `serbest_stok`, `gunluk_tuketim`, `stok_kapsama_gun`, `kullanilabilir_gun`, `acik_durus_gun` | İhtiyaç miktarı, zamanlama, hibrit miktar |

### `Parametreler`
| Parametre | Değer | Kullanım |
|---|---|---|
| `USDTRY` / `EURTRY` / `EURUSD` | 45.9 / 47.4 / 1.0327 | Kur dönüşümü |
| `PlanlamaTamponu` | 2 gün | Hibrit miktar |
| `KorumaUfku` | 7 gün | Transfer edilebilir miktar kontrolü |
| `QCTestMaliyeti` | 25.000 TRY/parti | İlaç (SAGLIK) test maliyeti |
| `KilLabTestGunluk` | 3.500 TRY/gün | KARO test maliyeti |
| `ElektronikMuayeneMaliyeti` | 8.000 TRY/parti | BANYO muayene maliyeti |

---

## 5. Hesap ve kısıt kuralları (deterministik tool'lar)

| # | Hesap | Formül / kural |
|---|---|---|
| O1 | USD fiyat | USD ise aynen · EUR ise × `EURUSD` · TRY ise ÷ `USDTRY` |
| O2 | İniş maliyeti | USD fiyat + (`Navlun Fiyata Dahil mi?` = "Hayır" ise) `Standart Navlun+Liman (USD/birim)`. DAP'ta navlun ikinci kez sayılmaz. |
| O3 | Hazır günü | tedarikçi hazırlığı + kalifikasyon + rota kapı-kapı + gümrük + kalite onayı |
| O4 | Lojistik maliyet | (`Değişken Maliyet (USD/kg)` × brüt kg + `Sabit Maliyet (USD/sevkiyat)` × sevkiyat sayısı) × `USDTRY`. Navlun dahilse 0. |
| O5 | Brüt kg | miktar × `Malzemeler.Brüt Ağırlık (kg/birim)` (lojistik hesabı **daima brüt kg**) |
| O6 | CO2 (kg) | brüt ton × `Mesafe (km)` × `Emisyon Faktörü (kgCO2e/ton-km)` |
| O7 | Hibrit miktar | (ana sevkiyat günü + `PlanlamaTamponu`) × günlük tüketim − stok − transfer; yukarı yuvarla, ≥ `Spot MOQ (birim)` |
| O8 | Ek malzeme maliyeti / fiyat farkı | İniş maliyeti bazında; formül `Formuller` sayfasından birebir alınır. |
| O9 | Test maliyeti | Kuruluşa göre `QCTestMaliyeti` / `KilLabTestGunluk` / `ElektronikMuayeneMaliyeti` |

**Kısıtlar (`Kısıt Kontrolü`):**
| # | Kısıt |
|---|---|
| C1 | transfer ≤ `Transfer Edilebilir Miktar` |
| C2 | miktar ≥ `Spot MOQ (birim)` |
| C3 | miktar ≤ `Hazır Stok (birim)` + `Ek Üretim Kapasitesi (birim/gün)` × süre |
| C4 | `Kalifikasyon Durumu` = "Kalifikasyon Gerekli" → `Kalifikasyon Süresi (gün)` ufuk içinde çözüm olamaz → **ELENİR** |
| C5 | SAGLIK malzemelerinde yalnızca **"Ruhsatta Kayıtlı"** kaynak kullanılabilir (`Malzemeler.Özel Koşul`) |
| C6 | Rota brüt kg ≤ `Sevkiyat Kapasitesi (kg)` |

**S1 kontrolü (O7):** (15 + 2) × 60,69 − 600 − 260 = 171,73 → yukarı 172 → MOQ 250 ≥ 172 → **250 kg**.

---

## 6. Çıktı sözleşmesi — `OptionsOutput`

> **Kanonik şema:** `00_ortak_veri_semalari.md` → `OptionsOutput`. Aşağıdaki örnekte zarf alanları (`run_id`, `d0`, `durum` vb.) kısaltılmıştır. Alan adı, tip ve zorunluluk için şema esastır.

```json
{
  "risk_no": "R-01",
  "senaryo": "S1",
  "stratejiler": [
    {
      "strateji_kodu": "S1-HIB",
      "strateji_adi": "Depodan transfer + kısmi hava (hibrit)",
      "bilesenler": [
        {"bilesen_no": 1, "tip": "TRANSFER", "kaynak": "D-SG-01|MT100054", "miktar": 260,
         "hazir_gun": "<tool>", "lojistik_maliyet_try": "<tool>", "co2_kg": "<tool>",
         "uygulama_riski": "<tool>", "kisit_kontrolu": {"sonuc": "GEÇTİ", "kontroller": ["C1"]}},
        {"bilesen_no": 2, "tip": "ALTERNATIF_TEDARIK_ROTA", "kaynak": "MT100054|TD5012", "rota_kodu": "<R-AIR-...>",
         "miktar": 250, "hazir_gun": 8, "lojistik_maliyet_try": "<tool>", "ek_malzeme_maliyeti_try": "<tool>",
         "test_maliyeti_try": "<QCTestMaliyeti>", "co2_kg": "<tool>", "uygulama_riski": "<tool>",
         "kisit_kontrolu": {"sonuc": "GEÇTİ", "kontroller": ["C2","C3","C4","C5","C6"]}}
      ],
      "uygun_mu": true,
      "eleme_nedeni": null
    }
  ],
  "elenen_kaynaklar": [{"kaynak": "<Malzeme|Tedarikçi>", "neden": "Kalifikasyon Gerekli – süre ufuk dışında", "hazir_gun": "<tool>"}],
  "kaynak": [{"sayfa": "Stok_Durumu", "anahtarlar": ["D-SG-01|MT100054"]}, {"sayfa": "Tedarik_Kaynaklari", "anahtarlar": ["MT100054|TD5012"]}, {"sayfa": "Rota_Secenekleri", "anahtarlar": ["<R-AIR-...>"]}],
  "uyarilar": []
}
```
Strateji kodu formatı `S<senaryo>-<KOD>`'dur (ör. `S1-HIB`, `S2-HIB-A`). Test eşlemesi için kodlar `Strateji_Bilesenleri`'ndeki kodlarla aynı olmalıdır.

---

## 7. Test eşlemesi (cevap anahtarı → `Strateji_Bilesenleri`, PK `Strateji Kodu` + `Bileşen No`)

| Kontrol | Docx / Kit referansı | S1 beklenen |
|---|---|---|
| Transfer edilebilir | `Stok_Durumu` 3. satır (D-SG-01) → `Transfer Edilebilir Miktar` | **260 kg** |
| Alternatif tedarikçi | `Tedarik_Kaynaklari` 7. satır (Zhejiang) | Hazır Stok **1.500 kg** · Spot MOQ **250 kg** · **Ruhsatta Kayıtlı** |
| Hızlı rota | `Rota_Secenekleri` 4. satır (Şanghay → İstanbul uçak) | **2 gün**, **4,60 USD/kg** |
| Hava bileşeni hazır günü | `Strateji_Bilesenleri` 12. satır, hazır gün sütunu | **8. gün** (< 9,89 kapsama) |
| Strateji sayısı | `Strateji_Karsilastirma` 2–7. satırlar | **6** strateji |
| Kısıt sonucu | `Strateji_Bilesenleri → Kısıt Kontrolü` | satır bazında |

Toleranslar: gün ±0,01 · TL ±1 · kg ±0,5.

**Genelleme:** S2 → Uşak (T-KR-02) transferi + Çan yerli kil (TD5002) · S3 → servis deposu (D-BN-01) transferi + HK ekspres; Almanya (TD5023) **kalifikasyon gerektirdiği için elenir**.
