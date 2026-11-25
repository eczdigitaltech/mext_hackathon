---
inclusion: fileMatch
fileMatchPattern: "src/agents/decision/**"
---

# 05 · Decision Agent — Agentic Kural Seti, Kapsam ve Görev

> **Referans:** Docx **ADIM 7 – "En iyisi hangisi?"** · Kit §5 kutu 4 (Decision Agent, kod + LLM) ve onay ekranı · Kit §7 Ekran 3–4 · nexus-data.md §2 #5 (deterministik simülasyon + MILP; LLM yalnızca gerekçe metni)
> **Veri kuralları:** `05_decision_agent_veri_kurallari.md`

---

## 1. Rol

**"En iyisi hangisi?"** sorusunu cevaplar. Options'ın ürettiği stratejileri gün gün simüle eder, toplam maliyeti hesaplar, ağırlıklı skorla sıralar, (S1 için) MILP ile doğrular ve satınalmacının göreceği **tek öneri kartını** gerekçesiyle hazırlar. **Kararı vermez, karar için öneri sunar.**

> Docx: "2,7 milyon TL'lik kayıp yerine 110 bin TL harcayarak fabrikanın hiç durmamasını sağlıyoruz."
> Kit §3: "Hesabı optimizasyon modeli yapıyor, yapay zekâ okuyor ve açıklıyor. Karar her zaman insanda."

---

## 2. Akıştaki yeri

| | |
|---|---|
| **Tetikleyici** | Orkestratör: Options tamamlandı ve Risk Kademesi **Yüksek** |
| **Girdi** | `OptionsOutput`, `ImpactOutput`, `RiskOutput`, `Parametreler` |
| **Çıktı** | `DecisionOutput` → onay ekranı (Orkestratör üzerinden) |
| **Ekran** | Ekran 3 (sıralama tablosu: "neden bu seçenek?") · Ekran 4 (öneri kartı + Onayla/Reddet/Değiştir + what-if bağlantısı) |
| **What-if** | Yeniden çalışır ve önerisini değiştirebilir |

---

## 3. Kapsam

| İçinde | Dışında |
|---|---|
| Gün gün stok simülasyonu (her strateji) | Seçenek üretmek (Options) |
| Toplam maliyet kalemleri (duruş, ceza, lojistik, ek malzeme, test, stok taşıma) | Risk veya etki hesabı (Risk, Impact) |
| ÇKKV ağırlıklı skor, sıralama, "ÖNERİLEN" | **Kararı vermek** veya sipariş vermek |
| S1 için MILP doğrulaması | Karar kaydının durumunu "Onaylandı / Reddedildi" yapmak (Orkestratör, insan kararıyla) |
| Öneri kartı ve gerekçe metni (LLM) | KPI paneli (sunum malzemesi) |
| Karar kaydı taslağı (`Onay Bekliyor`) | |

---

## 4. Görev adımları

1. **Simülasyon (tool):** Her uygun strateji için D0–D60 gün gün stok, giriş ve duruş oranı.
2. **Maliyet (tool):** Duruş ve ceza (Impact girdileri ile simülasyon duruşu) + Options kalemleri + stok taşıma → toplam maliyet.
3. **Skor ve sıralama (tool):** Senaryo içi normalize, ağırlıklı skor, sıra, "ÖNERİLEN".
4. **MILP (tool, S1):** `x_T`, `x_A`, `y_C` ile maliyet minimizasyonu; sonucu ÇKKV önerisiyle karşılaştırır.
5. **Tutarlılık muhakemesi (LLM, Bedrock):** ÇKKV önerisi ile MILP çözümü uyumlu mu? Önerilen stratejinin duruşu 0 mı? Aksiyonsuz maliyetten düşük mü? 2. sıra ile fark anlamlı mı? Uyumsuzlukta öneriyi **değiştirmez**, karta "dikkat notu" ekler ve uyarı üretir.
6. **Gerekçe (LLM):** Satınalmacıya hitap eden kısa, rakamlı, Türkçe gerekçe yazar (ton: Kit §8).
7. **Öneri kartı ve karar kaydı taslağı:** `DecisionOutput`.
8. **What-if yanıtı:** Yeni girdilerle 1–7'yi tekrarlar ve önceki öneriyle farkı (mevcut çözümün yeni duruşu, yeni öneri) işaretler.

---

## 5. Reasoning · Tool · Otonomi

| Kriter | Decision Agent'ta nasıl sağlanır? |
|---|---|
| **A1 Reasoning** | LLM, optimizasyon ve skor sonuçlarını yorumlar, MILP–ÇKKV tutarlılığını değerlendirir, ödünleşimleri (maliyet ↔ duruş ↔ CO2 ↔ risk) açıklar, dikkat notu gerekip gerekmediğine karar verir ve gerekçeyi yazar. |
| **A2 Otonomi** | Options çıktısı ve what-if değişikliğiyle insan girdisi olmadan çalışır. İnsan yalnızca son onaydadır. |
| **A3 Entegrasyon** | Tool'lar: `simulate_inventory`, `compute_total_cost`, `score_mcda`, `rank_strategies`, `solve_milp` (kod çalıştırma, PuLP), `build_recommendation_card`, `draft_decision_record`. |

> PuLP bir Python optimizasyon kütüphanesidir ve tool içinde çalışır. LLM veya servis değildir, izinli servis listesini (R3) değiştirmez.

---

## 6. Agentic kurallar (zorunlu)

| # | Kural |
|---|---|
| DC-K1 | **Karar insandadır.** Decision öneri sunar; karar kaydını yalnızca `Onay Bekliyor` durumuyla oluşturur. |
| DC-K2 | **LLM sıralamayı ve rakamları değiştirmez.** Gerekçedeki her rakam `DecisionOutput` alanlarından gelir (veri md §6). |
| DC-K3 | Her önerinin yanında **hiçbir şey yapmamanın maliyeti** gösterilir (Kit §2: "yöneticinin dilinden konuşuyoruz"). |
| DC-K4 | Yalnızca kısıtı geçen stratejiler "ÖNERİLEN" olabilir. |
| DC-K5 | ÇKKV ağırlıkları `Parametreler`'den okunur (maliyet %45, duruş %30, CO2 %10, risk %15). |
| DC-K6 | MILP ile ÇKKV çelişirse ÇKKV önerisi gösterilir, çelişki dikkat notu ve uyarı olarak karta eklenir. Sessizce birini seçmek yasaktır. |
| DC-K7 | Gerekçe tonu: kısa, rakamlı, Türkçe, satınalmacıya hitap eden. Örnek (Kit §8): *"Stok 4 Aralık'ta bitiyor, gemi 10 Aralık'ta hazır. Depodaki 260 kg ve uçakla 250 kg ile hat durmuyor; maliyet 110 bin TL."* |
| DC-K8 | What-if'te ekranda sayılar okunmaz, **değişen öneri** gösterilir (Kit §1 demo kuralı). |
| DC-K9 | LLM yalnızca Bedrock üzerinden çağrılır. Cevap anahtarındaki karşılaştırma ve öneri değerleri prompt'a/koda konmaz. |

---

## 7. Hata durumları

| Durum | Davranış |
|---|---|
| Uygun strateji yok | Kart "uygun strateji bulunamadı" uyarısı ve aksiyonsuz maliyetle döner; öneri uydurulmaz. |
| Normalize böleni 0 | Uyarı; ilgili kriter skoru tanımsız olarak işaretlenir (veri md H4 notu). |
| MILP çözümsüz / zaman aşımı | ÇKKV sonucu sunulur, "MILP doğrulaması yapılamadı" uyarısı eklenir. |
| Bedrock gerekçe üretemedi | Kart rakamlarla ve sabit şablon başlıkla sunulur, gerekçe alanı boş kalır ve uyarı eklenir. Rakamlar etkilenmez. |

---

## 8. Kabul kriterleri

**S1:**
| Kontrol | Beklenen |
|---|---|
| Önerilen | **Depodan 260 kg + Zhejiang'dan uçakla 250 kg** (S1-HIB) |
| Duruş | **0** |
| Toplam maliyet | **109.567 TL** (MILP amacı 108.745 + stok taşıma 823) |
| Aksiyonsuz | 2.741.018 TL → tasarruf **2.631.451 TL** |
| Sıralama | 6 strateji; Strateji_Karsilastirma 2–7. satırlarla birebir |

**What-if (+7 gün, Kit §1):** Mevcut çözümde **3,7 gün** duruş görülür ve sistem yeni öneriye geçer. Yeni önerinin beklenen değeri cevap anahtarı `WhatIfGun = 7` ile yeniden hesaplatılarak alınır.

**Ekran 4 (Kit §7):** Tek kart. Gerekçe metnini LLM yazar, rakamları kod hesaplar. What-if bağlantısıyla Ekran 2–4 yeniden hesaplanır.

**Demo cümlesi (Kit §1):** "Karar insanda. Satınalmacı tek ekranda gerekçeyi görüyor ve onaylıyor. Karar kaydediliyor, sistem buradan öğreniyor."

**Örnek senaryolar:** S2 → S2-HIB-A ~1,39 M TL · S3 → S3-HIB ~0,06 M TL.

**Jüri soruları (Kit §3):** "Sistem kendi başına sipariş veriyor mu?" → Hayır. · "LLM yanlış hesaplarsa?" → Hesabı LLM yapmıyor. · "Gecikme daha da uzarsa?" → What-if ile yeniden hesaplanır.

---

## 9. Kural uyum kontrolü

- [ ] LLM = Bedrock (R2) · Strands/Nova Act (R3)
- [ ] Yorum, tutarlılık ve gerekçe LLM muhakemesiyle (A1) · otonom (A2) · simülasyon/MILP tool'ları (A3)
- [ ] İnsan onayı olmadan aksiyon yok
- [ ] Ölçülebilir etki metriği üretiliyor: aksiyonsuz maliyet, önerilen maliyet, tasarruf, duruş günü (E1)
- [ ] AgentCore Runtime (P1, P2)
