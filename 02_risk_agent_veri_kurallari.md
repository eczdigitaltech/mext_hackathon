---
inclusion: fileMatch
fileMatchPattern: "src/agents/risk/**"
---

# 02 · Risk Agent — Veri Kuralları

> **Kaynak:** nexus-data.md §2 (#2), §4 (`Tedarikciler`, `Tasiyicilar`, `Risk_Kaynaklari`, `Parametreler`, `Esik_Tablolari`), §5 (Risk puanı), §7 (`Risk_Skorlama`) · Docx ADIM 4 · Kit §4 (2. Risk puanı)
> **Agentic eşi:** `02_risk_agent_kapsam_gorev.md`

---

## 1. Veri kaynağı kuralı

- Runtime'da yalnızca `data/nexus_agent_input.xlsx` ve önceki agent çıktıları (`SignalOutput`, `ImpactOutput.etki_puani`) kullanılır.
- `Risk_Skorlama` **yalnızca cevap anahtarında** bulunur; runtime'da okunmaz.
- Ağırlıklar ve eşikler kodda sabit yazılmaz, `Parametreler`'den okunur (Kit §4: "kodda sabit yazmayalım, oradan okuyalım").

---

## 2. Girdiler

### 2.1 Önceki agent'lardan
| Kaynak | Alan | Kullanım |
|---|---|---|
| `SignalOutput` | `sinyal_guvenilirligi`, `ilgili_taraf`, `olay_tipi`, `ulke_bolge`, `kanit`, `bagli_risk_no` | Sinyal bileşeni, performans tarafının seçimi, dış koşul muhakemesi |
| `ImpactOutput` | `etki_puani` (1–5) | Risk Puanı'nın Etki çarpanı |

### 2.2 `Tedarikciler` — PK `Tedarikçi No`
Kullanılan sütunlar: `Tedarikçi No` · `Tedarikçi Adı (Sahte)` · `Finansal Sağlık Notu` · `OTIF Oranı` · `Ort. Gecikme (gün)` · `Performans Puanı (1-5)` · `Finansal Puan (1-5)`

### 2.3 `Tasiyicilar` — PK `Taşıyıcı No`
Kullanılan sütunlar: `Taşıyıcı No` · `Taşıyıcı Adı (Sahte)` · `Zamanında Varış Oranı` · `Ort. Sapma (gün)` · `Finansal Sağlık Notu` · `Performans Puanı (1-5)` · `Finansal Puan (1-5)`

### 2.4 `Risk_Kaynaklari` — PK `Kaynak Kodu`
`Taban Güvenilirlik (1-5)`: Signal'in güvenilirlik puanının tutarlılık kontrolü için.

### 2.5 `Esik_Tablolari` — 14 satır · PK `Tablo` + `Alt Sınır / Not`
`Tablo` · `Alt Sınır / Not` · `Puan (1-5)` · `Nasıl kullanılır?`
Performans ve finansal puan eşiklerinin kaynağıdır; kodda tablo sabit yazılmaz, buradan okunur.

### 2.6 `Parametreler`
| Parametre | Değer | Kullanım |
|---|---|---|
| `W_Sinyal` | 0.4 | Olasılık ağırlığı |
| `W_Performans` | 0.3 | Olasılık ağırlığı |
| `W_DisKosul` | 0.2 | Olasılık ağırlığı |
| `W_Finansal` | 0.1 | Olasılık ağırlığı |
| `EsikYuksek` | 15 | Kademe eşiği (puan) |
| `EsikOrta` | 7 | Kademe eşiği (puan) |

---

## 3. Hesap kuralları (deterministik tool'lar)

| Bileşen | Kural | Kaynak |
|---|---|---|
| **Sinyal puanı** | `SignalOutput.sinyal_guvenilirligi` (kaynak taban + 1 doğrulandıysa, maks. 5) | nexus-data.md §5 |
| **Performans puanı** | OTIF / zamanında varış oranı: ≥0,95 → 1 · ≥0,90 → 2 · ≥0,85 → 3 · ≥0,75 → 4 · altı → 5 (1 iyi, 5 kötü). Girdideki `Performans Puanı (1-5)` ile tutarlılık kontrol edilir. | nexus-data.md §5, `Esik_Tablolari` |
| **Finansal puan** | A → 1 · B → 2 · C → 3 · D → 5. Girdideki `Finansal Puan (1-5)` ile tutarlılık kontrol edilir. | nexus-data.md §5 |
| **Dış koşul puanı** | **Girdi dosyasında yoktur.** Agent/LLM bağlamdan (ör. bölgesel kriz, liman yoğunluğu) **gerekçeli** belirler; 1–5 ölçeği. | nexus-data.md §5 |
| **Olasılık** | `W_Sinyal`×sinyal + `W_Performans`×performans + `W_DisKosul`×dış koşul + `W_Finansal`×finansal | nexus-data.md §5 |
| **Etki** | `ImpactOutput.etki_puani` (1–5) | nexus-data.md §5 (Etki) |
| **Risk Puanı** | Olasılık × Etki (1–25) | nexus-data.md §5 |
| **Kademe** | ≥ `EsikYuksek` → **Yüksek** (tam analiz + onaya sun) · ≥ `EsikOrta` → **Orta** (uyarı + seçenek hazırda) · altı → **Düşük** (izle) | nexus-data.md §5 |

**Performans tarafının seçimi:** `SignalOutput.ilgili_taraf` taşıyıcıysa (TSxx) `Tasiyicilar`, tedarikçiyse (TD5xxx) `Tedarikciler` kullanılır. İkisi birden varsa seçim gerekçesi çıktıya yazılır ve `Formuller` sayfasındaki kuralla teyit edilir.

---

## 4. Okuma tuzakları

- Puan yönü: **1 iyi, 5 kötü** (performans/finansal). Ters çevrilmez.
- `"-"` → `None`. Eksik bileşen sıfır kabul edilmez; uyarıyla döner.
- Oranlar 0–1 aralığındadır (yüzde değil).

---

## 5. Çıktı sözleşmesi — `RiskOutput`

> **Kanonik şema:** `00_ortak_veri_semalari.md` → `RiskOutput`. Aşağıdaki örnekte zarf alanları (`run_id`, `d0`, `durum` vb.) kısaltılmıştır. Alan adı, tip ve zorunluluk için şema esastır.

```json
{
  "risk_no": "R-01",
  "senaryo": "S1",
  "bilesenler": {
    "sinyal": "<tool>",
    "performans": {"puan": "<tool>", "taraf": "<TSxx|TD5xxx>", "kaynak_sutun": "Zamanında Varış Oranı"},
    "dis_kosul": {"puan": "<LLM, 1-5>", "gerekce": "<LLM>", "kanit_sinyaller": ["SG-2611-001", "SG-2611-002"]},
    "finansal": "<tool>"
  },
  "olasilik": 4.4,
  "etki": 5,
  "risk_puani": 22,
  "risk_kademesi": "Yüksek",
  "agent_aksiyonu": "Tam analiz + onaya sun",
  "aciklama": "<LLM, kısa Türkçe>",
  "kaynak": [{"sayfa": "Parametreler", "anahtarlar": ["W_Sinyal","W_Performans","W_DisKosul","W_Finansal","EsikYuksek","EsikOrta"]}],
  "uyarilar": []
}
```

---

## 6. Test eşlemesi (cevap anahtarı → `Risk_Skorlama`, PK `Risk No`)

| Çıktı alanı | Cevap anahtarı sütunu | S1 (R-01) beklenen |
|---|---|---|
| `bilesenler.*` | Puan bileşenleri sütunları | — (sayfadan) |
| `olasilik` | `Olasılık` | **4,4** |
| `etki` | `Etki` | **5** |
| `risk_puani` | `Risk Puanı` | **22** |
| `risk_kademesi` | `Risk Kademesi` | **Yüksek** |
| `agent_aksiyonu` | `Agent Aksiyonu` | Tam analiz + onaya sun |
| `aciklama` | LLM açıklama örneği | Anlamsal kıyas |

- Toleranslar: olasılık ±0,01 · risk puanı ±0,01.
- **Dış koşul puanı** LLM'den geldiği için en hassas testtir. Beklenen değer `Risk_Skorlama`'daki bileşen sütunundan alınır; sapmada gerekçe incelenir.
- **Genelleme:** R-04…R-08 → Orta/Düşük (uyarı/izleme).
