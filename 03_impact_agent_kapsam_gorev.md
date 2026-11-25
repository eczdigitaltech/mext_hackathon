---
inclusion: fileMatch
fileMatchPattern: "src/agents/impact/**"
---

# 03 · Impact Agent — Agentic Kural Seti, Kapsam ve Görev

> **Referans:** Docx **ADIM 2 – "Gecikmeli gemiye bakıyoruz"**, **ADIM 3 – "Depoda ne kadar malzeme var?"**, **ADIM 5 – "Bize maliyeti ne?"** · Kit §5 kutu 2 (Risk & Impact, kod) · nexus-data.md §2 #3 (deterministik, formül)
> **Veri kuralları:** `03_impact_agent_veri_kurallari.md`

---

## 1. Rol

Üç soruyu cevaplar: **"Malzeme ne zaman kullanılabilir?"**, **"Stok kaç gün yeter?"**, **"Hiçbir şey yapmazsak maliyeti ne?"**. Sonuçta aksiyonsuz maliyeti ve Risk Agent'ın ihtiyaç duyduğu **Etki Puanı**'nı üretir. NEXUS'taki en kritik hesap katmanıdır (Kit §4).

> Docx ADIM 5: "Hiçbir şey yapmazsak 2,7 milyon TL kaybederiz."

---

## 2. Akıştaki yeri

| | |
|---|---|
| **Tetikleyici** | Orkestratör, `SignalOutput.isleme_karari = ANALIZ` (Risk Olasılık ile paralel) |
| **Girdi** | `SignalOutput.ilgili_sevkiyat`, `ilgili_malzeme` + girdi sayfaları + `Parametreler` |
| **Çıktı** | `ImpactOutput` → Risk Agent (`etki_puani`), Options Agent, Decision Agent |
| **Ekran** | Ekran 2 · Etki (4 kutu: stok bitiş, malzeme hazır, duruş, aksiyonsuz maliyet + zaman çizelgesi) |
| **What-if** | **İlk yeniden çalışan agent.** `WhatIfGun` kullanılabilir tarihe eklenir, tüm zincir yeniden hesaplanır. |

---

## 3. Kapsam

| İçinde | Dışında |
|---|---|
| Kullanılabilir tarih (varış + gümrük + iç nakliye + kalite) | Sinyal yorumu (Signal) |
| Günlük ihtiyaç, stok kapsama, tükenme tarihi | Olasılık, Risk Puanı, Kademe (Risk) |
| Açık/duruş günü, duruş maliyeti | Transfer edilebilir miktar ve alternatif kaynaklar (Options) |
| Risk altındaki müşteri siparişleri ve cezalar | Strateji maliyeti, simülasyon (Decision) |
| Aksiyonsuz toplam maliyet, Etki Puanı | Öneri metni |

---

## 4. Görev adımları

1. **Kayıt bulma (tool):** Sevkiyat → malzeme → stok anahtarı → hat → müşteri siparişleri zincirini kurar (veri md §3).
2. **Kayıt seçimi (LLM muhakemesi):** Birden çok malzeme (ör. S3'te `MT100032; MT100031`) veya birden çok hat eşleşirse hangi kaydın kritik olduğunu gerekçeli belirler. Hepsini ayrı hesaplatabilir.
3. **ADIM 2 – Kullanılabilir tarih (tool):** I4–I6.
4. **ADIM 3 – Stok (tool):** I1–I3; yalnızca `Serbest Stok`.
5. **ADIM 5 – Maliyet (tool):** I7–I12.
6. **Etki Puanı (tool):** I13. Risk Agent'a iletilir.
7. **Tutarlılık muhakemesi (LLM):** Hesaplanan değerleri girdideki hazır sütunlarla (`Günlük Tüketim`, `Günlük Duruş Maliyeti (TRY)`) ve `Malzemeler` süreleriyle karşılaştırır. Birim uyumsuzluğu veya çelişki varsa uyarı yazar. **Sayıyı düzeltmez.**
8. **Çıktı:** `ImpactOutput`.

---

## 5. Reasoning · Tool · Otonomi

| Kriter | Impact Agent'ta nasıl sağlanır? |
|---|---|
| **A1 Reasoning** | LLM, tool çağrı planını kurar, çoklu eşleşmede kayıt seçimini ve veri tutarlılığı değerlendirmesini muhakemeyle yapar. **Sayısal çıktıların tamamı tool'lardan gelir** (nexus-data.md §2: deterministik). |
| **A2 Otonomi** | Signal çıktısıyla ve what-if parametresiyle insan girdisi olmadan tetiklenir. |
| **A3 Entegrasyon** | Tool'lar: `resolve_chain`, `compute_usable_date`, `compute_daily_need`, `compute_coverage`, `compute_gap`, `compute_downtime_cost`, `find_orders_at_risk`, `compute_penalty`, `compute_impact_score`. |

---

## 6. Agentic kurallar (zorunlu)

| # | Kural |
|---|---|
| I-K1 | **LLM hiçbir sayıyı hesaplamaz, tahmin etmez veya düzeltmez.** Tüm sayılar formül tool'larından gelir (Kit §3: "Hesabı LLM yapmıyor"). |
| I-K2 | Yalnızca `Serbest Stok` kullanılır; QC ve bloke stok kapsama hesabına girmez. |
| I-K3 | Kullanılabilir tarih **daima** varış + gümrük + iç nakliye + kalite onayıdır. Limana varış tarihi tek başına kullanılmaz (GMP). |
| I-K4 | `WhatIfGun` yalnızca `WhatIfSenaryo` ile eşleşen senaryonun kullanılabilir tarihine eklenir. |
| I-K5 | Yeniden başlatma maliyeti, açık > 0 ise **bir kez** eklenir. |
| I-K6 | Etki puanı eşikleri `Parametreler`'den okunur. |
| I-K7 | Müşteri adları sahtedir (`Müşteri (Sahte)`). Çıktıda sipariş no ile raporlanır, gerçek kişi/şirket verisi eklenmez (D1). |
| I-K8 | Cevap anahtarındaki `Etki_Analizi` değerleri koda veya prompt'a girmez. |

---

## 7. Hata durumları

| Durum | Davranış |
|---|---|
| Stok anahtarı bulunamadı | `HATA`; kapsama ve duruş hesaplanmaz, Risk'e etki puanı verilmez. |
| `Günlük Tüketim` = 0 (tüketen tesis için) | Kapsama tanımsız; uyarı ve `null`. Depo kaydı yanlışlıkla seçilmiş olabilir. |
| Açık = 0 | Duruş maliyeti 0. Etki puanı "açık yok" kuralıyla (1 veya 2) hesaplanır. |
| Hazır sütun ile hesaplanan değer farklı | Hesaplanan raporlanır, fark uyarıya yazılır. |

---

## 8. Kabul kriterleri

**S1 (R-01):**
| Kutu (Ekran 2) | Beklenen |
|---|---|
| Stok bitiyor | 600 kg → 9,89 gün → **04.12 21:16** |
| Malzeme hazır | ETA 05.12 → **10.12 (15. gün)** |
| Duruş | **5,11 gün**, 4 müşteri siparişi risk altında |
| Aksiyonsuz maliyet | **2.741.018 TL** (duruş 2.291.018 + ceza 450.000) |

**Zaman çizelgesi (Kit §6):** 02.11 gemi Mundra'dan çıkar · 24.11 09:12 e-posta · 25.11 analiz · 04.12 21:16 stok biter · 05.12 Ambarlı varış · 10.12 malzeme kullanılabilir.

**Demo cümlesi (Kit §1):** "Gemi limana 5 Aralık'ta varıyor ama ilaçta kalite onayı olmadan malzeme kullanılamıyor. Yani asıl tarih 10 Aralık. Bu arada hat 5 gün duruyor."

**What-if (+7 gün):** kullanılabilir gün 15 → 22. Etki zinciri yeniden hesaplanır ve Decision'a iletilir.

**Örnek senaryolar:** S2 ~14 gün / ~38,1 M TL · S3 ~9,2 gün / ~8,07 M TL. Üç senaryonun toplam aksiyonsuz maliyeti ~49 M TL'dir (Kit §2).

---

## 9. Kural uyum kontrolü

- [ ] LLM (yalnızca planlama/tutarlılık) = Bedrock (R2) · Strands/Nova Act (R3)
- [ ] Tool entegrasyonu (A3) · otonom tetik (A2) · reasoning ile kayıt seçimi (A1)
- [ ] Sentetik veri (D1)
- [ ] AgentCore Runtime (P1, P2)
