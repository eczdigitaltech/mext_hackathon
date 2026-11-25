---
inclusion: fileMatch
fileMatchPattern: "src/agents/decision/**"
---

# 05 · Decision Agent — Veri Kuralları

> **Kaynak:** nexus-data.md §2 (#5 Optimization & Decision; #6'nın karar kaydı kısmı), §4 (`Parametreler`), §5 (Seçenekler/karar: simülasyon, toplam maliyet, ÇKKV), §6 (what-if), §7 (`Zaman_Simulasyonu`, `Strateji_Karsilastirma`, `Optimizasyon_Modeli`, `Karar_Kaydi`) · Docx ADIM 7 · Kit §4 (5. Karar, Çıktı ve kontrol)
> **Agentic eşi:** `05_decision_agent_kapsam_gorev.md`

---

## 1. Veri kaynağı kuralı

- Runtime'da yalnızca `OptionsOutput`, `ImpactOutput`, `RiskOutput` ve girdi dosyasındaki `Parametreler` kullanılır.
- `Zaman_Simulasyonu`, `Strateji_Karsilastirma`, `Optimizasyon_Modeli`, `Karar_Kaydi` **yalnızca cevap anahtarında** bulunur; runtime'da okunmaz. Sıralama ve "ÖNERİLEN" etiketi hardcode edilmez.

---

## 2. Girdiler

### 2.1 Önceki agent'lardan
| Kaynak | Alan | Kullanım |
|---|---|---|
| `ImpactOutput` | `serbest_stok`, `gunluk_tuketim`, `kullanilabilir_gun`, `gunluk_durus_maliyeti_try`, `yeniden_baslatma_maliyeti_try`, `risk_altindaki_siparisler`, `aksiyonsuz_maliyet_try` | Simülasyon başlangıcı, ana sevkiyat girişi, duruş/ceza maliyeti, aksiyonsuz karşılaştırma |
| `OptionsOutput` | `stratejiler[].bilesenler[]` (miktar, hazır gün, lojistik, ek malzeme, test, CO2, risk, kısıt) | Simülasyon girişleri ve maliyet kalemleri |
| `RiskOutput` | `risk_puani`, `risk_kademesi`, `aciklama` | Öneri kartı bağlamı |

### 2.2 `Parametreler`
| Parametre | Değer | Kullanım |
|---|---|---|
| `W_Maliyet` | 0.45 | ÇKKV ağırlığı |
| `W_Durus` | 0.3 | ÇKKV ağırlığı |
| `W_CO2` | 0.1 | ÇKKV ağırlığı |
| `W_Risk` | 0.15 | ÇKKV ağırlığı |
| `PlanlamaTamponu` | 2 gün | Optimizasyon kısıtı / hibrit miktar |
| `StokTasimaOrani` | 0.3 (%/yıl) | Stok taşıma maliyeti |
| `USDTRY` | 45.9 | TRY dönüşümü |
| `WhatIfSenaryo`, `WhatIfGun` | S1, 0 | What-if bağlamı (değer Impact'te uygulanmış olarak gelir) |

---

## 3. Hesap kuralları (deterministik tool'lar)

| # | Hesap | Kural |
|---|---|---|
| H1 | Gün gün simülasyon | Ufuk D0…D60 (61 gün). Günlük kova: **gün başı giriş, gün içi tüketim**. Stok = dünkü stok + gelen − tüketim. Stok tüketimi karşılamıyorsa **kısmi duruş kesirli gün** olarak sayılır. Girişler: ana sevkiyat (`kullanilabilir_gun`) + strateji bileşenleri (`hazir_gun`). |
| H2 | Strateji duruşu | Simülasyondaki duruş oranlarının toplamı (gün) |
| H3 | Toplam maliyet | duruş + ceza + lojistik + ek malzeme + test + stok taşıma |
| H4 | ÇKKV skoru | `W_Maliyet`×maliyet + `W_Durus`×duruş + `W_CO2`×CO2 + `W_Risk`×risk; her kriter **senaryo içinde** 0–1 normalize edilir: (en kötü − bu) ÷ (en kötü − en iyi) |
| H5 | Sıralama | En yüksek ağırlıklı skor = **1. sıra = ÖNERİLEN**. Yalnızca kısıtı geçen stratejiler (`uygun_mu = true`) sıralanır. |
| H6 | MILP (yalnızca S1) | Karar değişkenleri: `x_T` (transfer), `x_A` (hava), `y_C` (COD, ikili). Amaç: maliyet minimizasyonu; kısıtlar Options C1–C6 ve duruş = 0 / tampon. PuLP ile kurulabilir (Kit §4). Sonuç ÇKKV önerisiyle karşılaştırılır. |
| H7 | Tasarruf | aksiyonsuz maliyet − önerilen strateji toplam maliyeti |

> H4'te en kötü = en iyi ise (bölen 0) normalize puan tanımsızdır. Bu durum hata fırlatmadan uyarıyla işaretlenir; davranış `Formuller` sayfasından teyit edilir.

**S1 tutarlılık kontrolleri** (beklenen değerlerle kontrol amaçlıdır, koda girmez):
| Strateji | Hesap | Duruş |
|---|---|---|
| Sadece transfer | (600 + 260) ÷ 60,69 = 14,17 gün kapsama → 15 − 14,17 | **0,83 gün** |
| Mersin COD | kullanılabilir gün 14 → 14 − 9,89 | **4,11 gün** |
| Hibrit | (600 + 260 + 250) ÷ 60,69 = 18,29 > 15; hava bileşeni 8. günde (< 9,89) | **0** |
| Hibrit, what-if +7 | ana sevkiyat 22. gün → 22 − 18,29 | **≈ 3,7 gün** |

---

## 4. Okuma tuzakları (test tarafı)

- `Zaman_Simulasyonu` **geniş formattadır**: satır 2 strateji kodu/adı, 3 senaryo, 4 başlangıç stoğu, 5 günlük tüketim, 6 ufuk, 7 toplam duruş; **satır 8 başlık**, satır 9–69 günler. Her strateji 3 sütundan oluşur: `Giriş`, `Gün Sonu Stok`, `Duruş Oranı`.
- `Optimizasyon_Modeli` tablo değil, form düzenindedir (7 numaralı bölüm).
- Cevap anahtarı formüllüdür; what-if beklenen değerleri için `WhatIfGun` Excel'de değiştirilip dosya yeniden hesaplatılmalıdır.

---

## 5. Karar kaydı taslağı (`Karar_Kaydi` formatı)

| Alan | Değer |
|---|---|
| `Olay No` | `NXS-26-0xx` |
| Karar | **`Onay Bekliyor`** (Decision yalnızca bu durumla oluşturur; güncellemeyi Orkestratör yapar) |
| Tasarruf | H7 |
| Zaman damgaları | öneri üretim anı (tespit anı Orkestratör'den) |

---

## 6. LLM'e verilecek bağlam (gerekçe metni için)

LLM'e **yalnızca** şu alanlar verilir: önerilen strateji adı ve bileşenleri, stok bitiş tarihi, malzeme kullanılabilir tarihi, önerilen strateji duruşu ve toplam maliyeti, aksiyonsuz maliyet, 2. sıradaki strateji ve farkı, risk kademesi. LLM bu alanlar dışında rakam kullanamaz.

---

## 7. Çıktı sözleşmesi — `DecisionOutput`

> **Kanonik şema:** `00_ortak_veri_semalari.md` → `DecisionOutput`. Aşağıdaki örnekte zarf alanları (`run_id`, `d0`, `durum` vb.) kısaltılmıştır. Alan adı, tip ve zorunluluk için şema esastır.

```json
{
  "risk_no": "R-01",
  "senaryo": "S1",
  "simulasyon": [{"strateji_kodu": "S1-HIB", "gunler": [{"gun": 0, "giris": 0, "gun_sonu_stok": 0, "durus_orani": 0}]}],
  "karsilastirma": [
    {"strateji_kodu": "S1-HIB", "uygun_mu": true, "durus_gun": 0, "maliyet_kalemleri": {"durus": 0, "ceza": 0, "lojistik": "<tool>", "ek_malzeme": "<tool>", "test": "<tool>", "stok_tasima": 823},
     "toplam_maliyet_try": 109567, "co2_kg": "<tool>", "risk": "<tool>",
     "normalize": {"maliyet": "<tool>", "durus": "<tool>", "co2": "<tool>", "risk": "<tool>"},
     "agirlikli_skor": "<tool>", "sira": 1, "oneri": "ÖNERİLEN"}
  ],
  "milp": {"durum": "OPTIMAL", "x_T": 260, "x_A": 250, "y_C": 0, "amac_try": 108745, "cckv_ile_uyumlu": true},
  "oneri_karti": {
    "baslik": "<ekran metni>",
    "gerekce": "<LLM, kısa Türkçe>",
    "dikkat_notu": null,
    "onerilen_strateji_kodu": "S1-HIB",
    "rakamlar": {"durus_gun": 0, "toplam_maliyet_try": 109567, "aksiyonsuz_maliyet_try": 2741018, "tasarruf_try": 2631451},
    "butonlar": ["Onayla", "Reddet", "Değiştir"]
  },
  "karar_kaydi_taslagi": {"olay_no": "NXS-26-0xx", "karar": "Onay Bekliyor", "tasarruf_try": 2631451, "oneri_zamani": "<ISO-8601>"},
  "what_if_karsilastirma": null,
  "uyarilar": []
}
```

---

## 8. Test eşlemesi (cevap anahtarı)

**`Strateji_Karsilastirma` (PK `Strateji Kodu`) — S1, satır 2–7:**
| Satır | Seçenek | G – Duruş | P – Toplam Maliyet | Z – Öneri |
|---|---|---|---|---|
| 2 | Hiçbir şey yapma | 5,11 gün | 2.741.018 TL | |
| 3 | Sadece depodan transfer | 0,83 gün | 554.863 TL | |
| 4 | Sadece kısmi uçak | 0 | 137.187 TL | |
| 5 | Tamamını uçakla getir | 0 | 350.075 TL | |
| 6 | Gemiyi Mersin'e yönlendir | 4,11 gün | 2.345.514 TL | |
| 7 | **Depodan 260 kg + uçakla 250 kg** | **0** | **109.567 TL** | **ÖNERİLEN** |

| Diğer kontroller | Cevap anahtarı | S1 beklenen |
|---|---|---|
| MILP amacı | `Optimizasyon_Modeli` | **108.745 TL** (transfer 260 kg + hava 250 kg) |
| Stok taşıma | `Strateji_Karsilastirma` maliyet kalemi | **823 TL** |
| Tasarruf | Kit §6 | **2.631.451 TL** |
| Simülasyon | `Zaman_Simulasyonu` (18 strateji × 61 gün) | gün bazında `Gün Sonu Stok`, `Duruş Oranı` |
| Karar kaydı formatı | `Karar_Kaydi` | alan ve durum değerleri |

Toleranslar: TL ±1 · gün ±0,01 · skor ±0,001.

**Genelleme:** S2 → **S2-HIB-A** ~1,39 M TL · S3 → **S3-HIB** ~0,06 M TL. Üç senaryoda toplam önerilen maliyet ~1,6 M TL (aksiyonsuz ~49 M TL).
