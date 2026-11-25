---
inclusion: fileMatch
fileMatchPattern: "src/orchestrator/**"
---

# 00 · Orkestratör — Agentic Kural Seti, Kapsam ve Görev

> **Referans:** Hazırlık Kiti §5 (uçtan uca akış, what-if döngüsü), §8 ("4 agent'ı sırayla çağıracak yapı") · nexus-data.md §2 #0 · Docx ADIM 1–7 sırası
> **Veri kuralları:** `00_orkestrator_veri_kurallari.md`

---

## 1. Rol

Orkestratör, NEXUS akışını yöneten **deterministik durum makinesidir**: Sinyal → Risk/Etki → Seçenek → Karar → Onay. Kendisi hesap yapmaz, metin yorumlamaz, öneri üretmez. Agent'ları doğru sırada çağırır, sözleşmeleri doğrular, risk kademesine göre akışı yönlendirir, what-if ve insan onayı döngülerini yönetir.

> Reasoning (A1) agent'ların içindedir. Orkestratörün deterministik olması nexus-data.md §2 #0'ın tanımıdır ve izlenebilirliği sağlar.

---

## 2. Kapsam

| İçinde | Dışında |
|---|---|
| Tetikleyicileri dinlemek (yeni sinyal, what-if, insan kararı) | Ham metni okumak (Signal) |
| Agent çağırma sırası ve paralellik | Risk, etki, maliyet, strateji hesabı (ilgili agent'lar) |
| Sözleşme şeması doğrulaması | Öneri gerekçesi yazmak (Decision) |
| Risk kademesine göre yönlendirme (kapı) | Sipariş vermek veya ERP'ye yazmak (**hiçbir agent yapmaz**) |
| What-if döngüsünü tetiklemek | KPI paneli üretmek (sunum malzemesi) |
| İnsan kararını (Onayla/Reddet/Değiştir) karar kaydına işlemek | |
| Adım bazlı log, zaman damgası, hata yönetimi | |

---

## 3. Durum makinesi

```
BEKLE
  │ yeni Sinyal No (otonom tetik)
  ▼
SINYAL ──(Signal: isleme_karari = AYIKLA)──► KAPAT (gerekçe loglanır)
  │ ANALIZ
  ▼
RISK_OLASILIK  ∥  ETKI           ← paralel (bkz. Kapsam kararı K1)
  │                 │
  └──── Etki Puanı ─┘
  ▼
RISK_KADEME
  ├─ Yüksek (≥ EsikYuksek=15) ─► SECENEK ─► KARAR ─► ONAY_BEKLIYOR ─► KAYIT
  ├─ Orta   (≥ EsikOrta=7)    ─► SECENEK (hazırda tutulur, onaya sunulmaz) ─► UYARI
  └─ Düşük                    ─► IZLEME
ONAY_BEKLIYOR
  ├─ Onayla  → Karar = "Onaylandı"
  ├─ Reddet  → Karar = "Reddedildi"
  └─ Değiştir→ Karar = "Revize Edildi"
WHAT_IF (WhatIfGun ≠ 0 veya aynı sevkiyata yeni sinyal)
  └─► ETKI → RISK_KADEME → SECENEK → KARAR → ONAY_BEKLIYOR   (Signal yeniden çalışmaz)
```

Kademe eşikleri ve aksiyonları nexus-data.md §5'tendir: ≥15 Yüksek (tam analiz + onaya sun) · ≥7 Orta (uyarı + seçenek hazırda) · altı Düşük (izle). Eşikler `Parametreler`'den okunur.

---

## 4. Görevler

1. **Tetikleme (otonomi, A2):** `Risk_Sinyalleri`'nde yeni bir `Sinyal No` görüldüğünde akışı insan girdisi olmadan başlatır. Kit'teki hedefe göre tespit süresi yaklaşık dakikalar mertebesindedir (S1: e-posta 09:12, yakalama 09:20, yani 8 dakika).
2. **Veri yükleme:** Girdi dosyasını bir kez yükler, `params` ve `D0`'ı hazırlar ve agent'lara iletir.
3. **Sıralama:** Signal'i çağırır. `ANALIZ` çıktısında Risk (Olasılık) ve Impact'i paralel çağırır, ardından Etki Puanı'nı Risk'e verip Kademe'yi alır.
4. **Kapı:** Kademe'ye göre Options ve Decision'ı çağırır veya akışı uyarı/izleme ile bitirir.
5. **Onay:** Decision'ın öneri kartını onay ekranına iletir ve insan kararını `Karar_Kaydi` formatında işler.
6. **What-if:** `WhatIfGun` değiştiğinde (veya aynı sevkiyat için yeni sinyal geldiğinde) Impact → Risk kademe → Options → Decision zincirini yeniden çalıştırır ve önceki öneriyle farkı işaretler.
7. **İzlenebilirlik:** Her adımın girdi özeti, çıktı özeti, süresi ve uyarıları loglanır (AgentCore Observability önerilir). Gerçek kişisel veri loglanmaz.

---

## 5. Agentic kurallar (zorunlu)

| # | Kural |
|---|---|
| O1 | Orkestratör **hiçbir sayıyı üretmez veya değiştirmez**. Sayının sahibi onu hesaplayan agent'tır. |
| O2 | Sözleşme doğrulamasından geçmeyen çıktı bir sonraki agent'a verilmez. Akış `HATA` durumuna geçer ve kısmi sonuç "tamamlanamadı" olarak işaretlenir. Eksik değer tahminle doldurulmaz. |
| O3 | Yalnızca **Yüksek** kademe onaya sunulur. Orta kademede seçenekler hazırda tutulur, Düşük kademe izlemeye alınır. |
| O4 | **Karar insandadır.** Onay ekranından karar gelmeden hiçbir aksiyon "uygulandı" sayılmaz. Sistem sipariş vermez. |
| O5 | What-if'te Signal Agent yeniden çalışmaz. Gecikme bilgisi `WhatIfGun` ile Impact Agent'ın kullanılabilir tarih hesabına eklenir. |
| O6 | LLM çağrısı yapan tüm agent'lar Bedrock'u kullanır. Orkestratör LLM çağırmaz. |
| O7 | Final akış **AgentCore Runtime** üzerinde çalışır. Yerel çalıştırma yalnızca geliştirme içindir. Deploy hesap ID'si ve bölge README'ye yazılır. |
| O8 | Her agent ayrı test edilebilir ve ayrı çağrılabilir olmalıdır (Kit §3: "her biri ayrı test edilebiliyor"). |

---

## 6. Hata durumları

| Durum | Davranış |
|---|---|
| Signal sevkiyat eşleştiremedi | Akış `IZLEME`'ye düşer, uyarı üretilir, Impact çalıştırılmaz. |
| Bir agent zaman aşımı veya hata verdi | Akış durur, hangi adımda kaldığı loglanır. Demo için yedek video/ekran görüntüsüne geçilir (Kit §1). |
| Decision'da kısıtı geçen strateji yok | Onay ekranına "uygun strateji yok" uyarısıyla yalnızca aksiyonsuz maliyet sunulur. Öneri uydurulmaz. |
| Aynı sevkiyata ikinci sinyal (ör. SG-2611-002) | Yeni risk açılmaz, mevcut `Risk No`'ya bağlanır ve what-if gibi yeniden hesap tetiklenir. |

---

## 7. Kabul kriterleri

- **S1 (canlı demo):** SG-2611-001'den başlayan akış onay ekranına kadar uçtan uca çalışır ("Bitti" tanımı, Kit §8). Ara değerler indeksteki S1 tablosuyla birebir aynıdır.
- **What-if:** `WhatIfGun = 7` ile zincir yeniden çalışır. Mevcut çözüm için 3,7 gün duruş görülür ve öneri değişir.
- **Gürültü:** SG-2611-013…-017, SG-2610-* ve `H` ekli sinyaller onaya ulaşmaz.
- **İzleme:** R-04…R-08 Orta/Düşük kademede uyarı veya izleme ile biter.
- **Zaman:** Uçtan uca workflow 5 dakikalık demoya sığar (kural seti §5.2, E5).
- **S2 / S3:** Aynı akış parametre değişikliğiyle çalışır (Kit: "aynı sistem bunları da çözüyor"). Canlı demoda yalnızca S1 gösterilir.

---

## 8. Kural uyum kontrolü

- [ ] Akış AgentCore Runtime'da çalışıyor (P1, P2) · AgentCore bileşeni kullanılıyor (R4)
- [ ] Otonom tetikleme gösterilebiliyor (A2)
- [ ] Agent'lar tool/agent olarak entegre (A3)
- [ ] Orkestratör LLM çağırmıyor; agent'ların LLM'i yalnızca Bedrock (R2)
- [ ] Loglarda kişisel veri yok (D1, D2)
- [ ] Karar kaydı insan kararı olmadan "Onaylandı" olmuyor
