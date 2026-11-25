---
inclusion: fileMatch
fileMatchPattern: "src/agents/risk/**"
---

# 02 · Risk Agent — Agentic Kural Seti, Kapsam ve Görev

> **Referans:** Docx **ADIM 4 – "Risk ne kadar ciddi?"** · Kit §5 kutu 2 (Risk & Impact) · nexus-data.md §2 #2 (deterministik puanlama + LLM açıklama)
> **Veri kuralları:** `02_risk_agent_veri_kurallari.md`

---

## 1. Rol

**"Risk ne kadar ciddi?"** sorusunu cevaplar. Sinyalin gerçekleşme olasılığını puanlar, Impact'in etki puanıyla çarparak Risk Puanı'nı (1–25) ve Kademe'yi belirler. Kademe, akışın tam analize geçip geçmeyeceğinin **kapısıdır**.

> Docx: "4,4 × 5 = 22 (25 üzerinden) → Yüksek. Sistem 'tam analiz yap' diyor."

---

## 2. Akıştaki yeri

| | |
|---|---|
| **Tetikleyici** | Orkestratör, `SignalOutput.isleme_karari = ANALIZ` |
| **Çalışma şekli** | İki aşamalı: **(a)** Olasılık (Impact ile paralel) → **(b)** Impact'in `etki_puani` gelince Risk Puanı ve Kademe |
| **Girdi** | `SignalOutput`, `ImpactOutput.etki_puani`, karne sayfaları, `Parametreler` |
| **Çıktı** | `RiskOutput` → Orkestratör (kapı kararı) |
| **Ekran** | Ekran 1 · Uyarı paneli (risk kartı: Olasılık × Etki = Puan, Kademe; izlenen riskler listesi) |
| **What-if** | Yeniden çalışır (Etki değişirse puan ve kademe değişebilir) |

> **Sıra notu (Kapsam kararı K1):** Docx anlatımında Risk (ADIM 4), Impact'ten (ADIM 5) önce gelir. Etki bileşeni Impact'ten geldiği için runtime'da Olasılık ve Impact paralel hesaplanır, puan en son kesinleşir. Ekran ve anlatım sırası değişmez.

---

## 3. Kapsam

| İçinde | Dışında |
|---|---|
| Sinyal, performans, dış koşul, finansal bileşenlerini puanlamak | Ham metni yeniden okumak (Signal) |
| Dış koşul puanını gerekçeli belirlemek (LLM) | Etki puanını, duruşu, maliyeti hesaplamak (Impact) |
| Olasılık, Risk Puanı, Kademe, Agent Aksiyonu | Seçenek üretmek (Options) |
| Risk kartı için kısa açıklama (LLM) | Akışı yönlendirmek. Kademeyi üretir, yönlendirmeyi Orkestratör yapar. |

---

## 4. Görev adımları

1. **Sinyal bileşeni (tool):** `SignalOutput.sinyal_guvenilirligi`'ni alır ve `Risk_Kaynaklari` ile tutarlılığını kontrol eder.
2. **Taraf seçimi (LLM muhakemesi):** Olay tipine göre performansın taşıyıcıdan mı tedarikçiden mi okunacağına karar verir ve gerekçesini yazar.
3. **Performans ve finansal (tool):** `Esik_Tablolari`'na göre puanlar, girdideki hazır puan sütunlarıyla karşılaştırır.
4. **Dış koşul (LLM, Bedrock):** Signal kanıtları ve aynı riske bağlı diğer sinyallerden (ör. bölgesel kriz) 1–5 puan ve gerekçe üretir. **Bu, agent'ın ana muhakeme noktasıdır.**
5. **Olasılık (tool):** `W_*` ağırlıklarıyla ağırlıklı toplam.
6. **Etki'yi bekleme:** `ImpactOutput.etki_puani` gelince Risk Puanı = Olasılık × Etki (tool).
7. **Kademe ve aksiyon (tool):** `EsikYuksek` / `EsikOrta` eşiklerine göre.
8. **Açıklama (LLM):** Risk kartı için kısa, rakamlı Türkçe açıklama yazar. Örnek ton (Kit §1): *"Sinyal güvenilir, taşıyıcının geçmişi zayıf, Kızıldeniz krizi aktif. Sistem bunu yüksek risk olarak işaretliyor."*

---

## 5. Reasoning · Tool · Otonomi

| Kriter | Risk Agent'ta nasıl sağlanır? |
|---|---|
| **A1 Reasoning** | Dış koşul puanı (girdide yok) ve performans tarafı seçimi LLM muhakemesiyle, gerekçeli olarak verilir. Açıklama LLM'le yazılır. |
| **A2 Otonomi** | Signal çıktısıyla insan girdisi olmadan tetiklenir. |
| **A3 Entegrasyon** | Tool'lar: `score_signal`, `score_performance`, `score_financial`, `compute_probability`, `compute_risk_score`, `classify_tier`. Ayrıca Impact Agent çıktısını tüketir. |

---

## 6. Agentic kurallar (zorunlu)

| # | Kural |
|---|---|
| R-K1 | LLM yalnızca Bedrock üzerinden çağrılır. |
| R-K2 | **LLM'in sayısal katkısı yalnızca dış koşul puanıdır** (1–5, gerekçe ve kanıt zorunlu). Olasılık, puan ve kademe tool'larla hesaplanır. |
| R-K3 | Ağırlık ve eşikler `Parametreler`'den okunur. "Risk 15 ve üstü Yüksek" iş kuralıdır, kodda sabit yazılmaz (Kit §8). |
| R-K4 | Etki puanı olmadan Risk Puanı ve Kademe üretilmez; Olasılık tek başına raporlanır. |
| R-K5 | Düşük güvenilirlikli sinyal (ör. sosyal medya söylentisi) düşük puan alır ve izlemeye düşer. Bu, yanlış alarm oranını düşük tutar (Kit §3: pilot %12,5, hedef < %15). |
| R-K6 | Açıklamadaki her rakam `RiskOutput` içindeki bir alandan gelir. Metinde yeni rakam üretilmez. |
| R-K7 | Cevap anahtarındaki `Risk_Skorlama` değerleri prompt'a veya koda konmaz. |

---

## 7. Hata durumları

| Durum | Davranış |
|---|---|
| Taraf karnesi bulunamadı | Performans/finansal bileşeni `null`, uyarı. Olasılık "eksik bileşen" bayrağıyla raporlanır; Orkestratör onaya sunmaz. |
| Dış koşul için yeterli kanıt yok | LLM "kanıt yetersiz" gerekçesiyle orta değer önermez; `null` döner ve uyarı üretir. |
| Girdideki hazır puan ile eşikten hesaplanan puan farklı | Hesaplanan kullanılır, fark uyarıya yazılır. |

---

## 8. Kabul kriterleri

**S1 (R-01):** Olasılık **4,4** · Etki **5** · Risk Puanı **22** · Kademe **Yüksek** · Aksiyon: tam analiz + onaya sun.

**Örnek senaryolar:**
| Risk | Bağlam (nexus-data.md §6) | Beklenen |
|---|---|---|
| R-04 | Sonraki parti OOS | Orta/Düşük → uyarı veya izleme |
| R-05 | Ambarlı yoğunluk | Orta/Düşük |
| R-06 | Tek sinyal → ikinci malzeme | Orta/Düşük |
| R-07 | Castellón grevi | Orta/Düşük |
| S2, S3 riskleri | Liman kapanması, enerji kısıtı | Yüksek (tam analiz) |

**Jüri sorusu (Kit §3):** "Yanlış alarm olursa ne olur?" → Her sinyalin güvenilirlik puanı vardır; düşük puanlı sinyal yalnızca izlemeye girer.

---

## 9. Kural uyum kontrolü

- [ ] LLM = Bedrock (R2) · Strands/Nova Act (R3)
- [ ] Dış koşul ve taraf seçimi LLM muhakemesiyle (A1) · otonom tetik (A2) · tool + agent entegrasyonu (A3)
- [ ] Parametreler sabit yazılmadı
- [ ] AgentCore Runtime (P1, P2)
