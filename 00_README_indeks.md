---
inclusion: always
---

# NEXUS – Agent Bazlı Görev ve Kapsam Doküman Seti (İndeks)

> Bu set, NEXUS'taki her agent için iki doküman içerir: **(1) Veri Kuralları** ve **(2) Agentic Kural Seti, Kapsam ve Görev**. Agent'lar arası tüm JSON sözleşmeleri ayrıca tek bir **ortak şema** dosyasında tanımlıdır.
> Analiz tarihi (D0, "bugün") **2026-11-25**'tir (`Parametreler → AnalizTarihi`). Tüm şirketler, kişiler ve rakamlar sahtedir ama birbiriyle tutarlıdır.

---

## 1. Kaynak hiyerarşisi

| Öncelik | Kaynak | Bu sette nasıl kullanıldı? |
|---|---|---|
| 1 | `hackathon-rules.md` + `hackathon_kural_seti.md` | **Sınır.** Hiçbir agent dokümanı bu iki dosyanın dışına çıkmaz: yalnızca Bedrock, AgentCore deploy, Strands/Nova Act, sentetik veri, bulutta demo. |
| 2 | `Hackathon_Adımlar_Detay.docx` | **Asıl akış (ADIM 1–7) ve agent kapsamı:** Signal, Risk, Impact, Options, Decision. |
| 3 | `nexus-data.md` | **Veri kapsamı:** sayfa, sütun, anahtar ve formüller. Agent veri md'leri bu dosyanın agent bazında bölünmüş hâlidir ve onun dışına çıkmaz. |
| 4 | `NEXUS – Hackathon Hazırlık Kiti` §5 | **Uçtan uca akış iskeleti:** tek yönlü akış, what-if döngüsü, insan onayı. Kitin diğer bölümleri (S1 detayı, ekranlar, jüri soruları, S2/S3) örnek senaryo ve kabul kriteri olarak kullanıldı. |

**Çakışma olursa öncelik sırası:** kural md'leri > docx (akış/kapsam) > nexus-data.md (veri) > Hazırlık Kiti.

---

## 2. Doküman listesi

| # | Agent | Docx adımı | Soru | Veri kuralları | Agentic kapsam & görev |
|---|---|---|---|---|---|
| — | **Ortak şema** | (tümü) | Agent'lar arası veri hangi formatta taşınır? | `00_ortak_veri_semalari.md` | — |
| 00 | **Orkestratör** | (akış) | Sıradaki adım ne? | `00_orkestrator_veri_kurallari.md` | `00_orkestrator_kapsam_gorev.md` |
| 01 | **Signal Agent** | ADIM 1 | Ne oldu? | `01_signal_agent_veri_kurallari.md` | `01_signal_agent_kapsam_gorev.md` |
| 02 | **Risk Agent** | ADIM 4 | Risk ne kadar ciddi? | `02_risk_agent_veri_kurallari.md` | `02_risk_agent_kapsam_gorev.md` |
| 03 | **Impact Agent** | ADIM 2, 3, 5 | Malzeme ne zaman gelir, stok kaç gün yeter, maliyeti ne? | `03_impact_agent_veri_kurallari.md` | `03_impact_agent_kapsam_gorev.md` |
| 04 | **Options Agent** | ADIM 6 | Ne yapabiliriz? | `04_options_agent_veri_kurallari.md` | `04_options_agent_kapsam_gorev.md` |
| 05 | **Decision Agent** | ADIM 7 | En iyisi hangisi? | `05_decision_agent_veri_kurallari.md` | `05_decision_agent_kapsam_gorev.md` |

---

## 3. Uçtan uca akış

```
 Sinyal gelir (Risk_Sinyalleri → Ham Metin (LLM Girdisi))
        │
        ▼
 [01] Signal Agent ── LLM çıkarım + kural eşleştirme ── gürültüyü ayıklar
        │  çıktı: sevkiyat, malzeme, gecikme günü, alternatif ipucu, güvenilirlik
        ├──────────────────────────────┐
        ▼                              ▼
 [02] Risk Agent (Olasılık)      [03] Impact Agent (ADIM 2-3-5)
        │                              │  kullanılabilir tarih, kapsama, duruş, aksiyonsuz maliyet, Etki Puanı
        │◄─────── Etki Puanı ──────────┘
        ▼
 [02] Risk Agent: Risk Puanı = Olasılık × Etki → Kademe (Yüksek / Orta / Düşük)
        │  Yüksek ise tam analiz
        ▼
 [04] Options Agent (Sourcing + Logistics alt modülleri) → strateji seçenekleri + kısıt kontrolü
        ▼
 [05] Decision Agent → gün gün simülasyon, toplam maliyet, ÇKKV skoru, (S1) MILP, LLM gerekçesi
        ▼
 Onay ekranı: KARAR İNSANDA (Onayla / Reddet / Değiştir) → karar kaydı
        ┆
        └┄┄ what-if (WhatIfGun değişir) ┄┄► [03] + [02] + [04] + [05] yeniden çalışır
```

Bu tablo Hazırlık Kiti §5'teki "her agent'ın çıktısı bir sonrakinin girdisi" ilkesini izler. Kesikli ok what-if döngüsünü gösterir. Gecikme değişince Signal Agent yeniden çalışmaz.

---

## 4. Kapsam kararları (kaynaklar arasındaki farklar nasıl çözüldü?)

| # | Fark | Karar |
|---|---|---|
| K1 | Docx'te Risk (ADIM 4) ve Impact (ADIM 5) ayrı agent'lardır; Kit §5'te "Risk & Impact" tek kutudur. | **İki ayrı agent.** Risk Puanı = Olasılık × Etki'dir ve Etki (1–5), Impact'in aksiyonsuz maliyetinden türetilir (nexus-data.md §5). Bu yüzden runtime sırası şöyledir: Risk (Olasılık) ile Impact paralel çalışır, ardından Risk Puanı ve Kademe hesaplanır. Ekranlarda ve anlatımda docx sırası korunur. |
| K2 | Docx'teki ADIM 2–3 (Sevkiyatlar → kullanılabilir tarih, Stok_Durumu → kapsama) bir agent'a atanmamış. | nexus-data.md #3 ve Kit §5 kutu 2'ye göre **Impact Agent**'a verildi. |
| K3 | Docx'te tek Options Agent var (a/b/c alt soruları); nexus-data.md'de 4a Sourcing ve 4b Logistics ayrı. | **Tek agent, iki alt modül** (Sourcing, Logistics). Kit'teki "biri satın alma, biri lojistik tarafından seçenek üretiyor" anlatımı bu alt modüllerle karşılanır. |
| K4 | nexus-data.md'deki #6 Notification Agent docx kapsamında yok. | **Ayrı agent açılmadı.** Öneri kartı ve karar kaydı taslağı Decision Agent'ta, onay/red/revize kaydı Orkestratör'de. `KPI_Panel` sunum malzemesidir, agent çıktısı değildir. |
| K5 | Orkestratör docx'te bir adım değil. | nexus-data.md #0 ve Kit §8 ("4 agent'ı sırayla çağıracak yapı") gereği ayrı doküman çifti var. Orkestratör deterministik bir durum makinesidir. |
| K6 | Kural setleri "kararlar reasoning LLM ile verilir" der (A1); nexus-data.md ve Kit "sayısal her şey kodla hesaplanır" der. | **Uzlaşı:** Her agent, Bedrock üzerindeki bir reasoning LLM ile çalışan bir Strands agent'ıdır. LLM planlar, yorumlar, doğrular, gerekçelendirir ve tool çağırır. **Sayılar yalnızca deterministik tool'lardan gelir; LLM sayı üretmez ve değiştirmez.** Kritik muhakemeler (gürültü ayıklama, dış koşul puanı, seçenek kurgusu, öneri gerekçesi, eskalasyon) LLM'le yapılır, sabit if/else'e indirgenmez. LLM'in her agent'taki rolü nexus-data.md §2'deki "LLM / Kod" sütunuyla sınırlıdır. |

---

## 5. Tüm agent'lar için ortak kurallar

| Kural | Kaynak |
|---|---|
| Tüm LLM çağrıları **yalnızca Amazon Bedrock** üzerinden yapılır. Başka sağlayıcı, SDK, yerel model kullanılmaz. | R2 · rules §1 |
| Agent'lar **Strands Agents** (veya Nova Act SDK) ile yazılır. Yeni framework/servis önermeden önce izinli listeyle uyumu kontrol edilir. | R3 · rules §2 |
| Deploy hedefi **Amazon Bedrock AgentCore Runtime**'dır ve demo AWS Cloud'da yapılır. Yerelde çalışma yalnızca geliştirme/test içindir ve değerlendirme dışıdır. | P1, P2 · rules §6 |
| AgentCore'un en az bir temel bileşeni kullanılır. Runtime deploy hedefidir; izlenebilirlik için Observability, karar kaydı için Memory önerilir. | R4 · rules §2 |
| Her agent üç kriteri birden sağlar: **Reasoning** (LLM muhakemesi), **Otonomi** (tetikleyiciyle insan girdisi olmadan çalışma), **Entegrasyon** (tool'lar, veri erişimi, diğer agent'lar). | A1–A3 · rules §3 |
| Runtime'da **yalnızca** `data/nexus_agent_input.xlsx` okunur. `tests/fixtures/nexus_answer_key.xlsx` yalnızca testlerde ve formül referansı olarak kullanılır. Cevap değerleri agent'a verilmez, kodda hardcode edilmez. | nexus-data.md §1 |
| Yalnızca sentetik/anonim veri kullanılır. Gerçek kişisel veri (ad, TCKN, telefon, e-posta, adres) üretilmez, eklenmez, loglanmaz. | D1, D2 · rules §5 |
| Parametreler (ağırlık, eşik, kur, tampon, what-if) kodda sabit yazılmaz, `Parametreler` sayfasından okunur. | nexus-data.md §4 · Kit §4 |
| Sütun adları **birebir** nexus-data.md'den alınır; uydurma sütun adı kullanılmaz. | nexus-data.md (amaç) |
| **Karar her zaman insandadır.** Sistem sipariş vermez, gerekçeli öneri sunar. | Kit §3, §8 |
| **Rakam uydurulmaz.** Bilinmeyen değer `null` ve açıklama ile döner. | Kit §3 |
| Kod hackathon sırasında sıfırdan yazılır. Bağımlılıklar sabitlenir, kurulum/deploy README'de belgelenir, gizli bilgiler ortam değişkeni veya AWS kimlik mekanizmasıyla yönetilir. | R1 · rules §4 |
| Agent'lar arası veri, JSON sözleşmeleriyle taşınır (kanonik tanım: `00_ortak_veri_semalari.md`). Her çıktı `kaynak` (sayfa + anahtar) ve `kanit` alanı taşır; böylece her adımın neden öyle sonuçlandığı izlenebilir. | Kit §3, §8 |

---

## 6. S1 cevap anahtarı özeti (tüm agent'ların ortak test referansı)

| Agent | Adım | Beklenen değer |
|---|---|---|
| Signal | Sinyal → sevkiyat/malzeme | SG-2611-001 → **SV-26-1184**, **MT100054**, gecikme **8 gün**, ipucu: **Mersin'e liman değişikliği mümkün** |
| Impact | Kullanılabilir tarih | ETA 27.11 → **05.12**; + 1 gümrük + 1 iç nakliye + 3 kalite = **10.12 (15. gün)** |
| Impact | Tüketim / kapsama | **60,69 kg/gün**; 600 kg serbest stok → **9,89 gün**, tükenme **04.12 21:16** |
| Impact | Duruş / maliyet | **5,11 gün** duruş · duruş maliyeti **2.291.018 TL** · ceza **450.000 TL** (4 sipariş) · aksiyonsuz **2.741.018 TL** |
| Risk | Puan | Olasılık **4,4** × Etki **5** = **22 → Yüksek** |
| Options | Seçenekler | 6 strateji. D-SG-01'den transfer edilebilir **260 kg**; Zhejiang (TD5012) hazır **1.500 kg**, MOQ **250 kg**, Ruhsatta Kayıtlı; Şanghay → İstanbul uçak **2 gün, 4,60 USD/kg**; hava bileşeni **8. gün** gelir |
| Decision | Öneri | **Depodan 260 kg + Zhejiang'dan uçakla 250 kg**, duruş **0**, toplam **109.567 TL** (MILP amacı 108.745 + stok taşıma 823) |
| Decision | What-if +7 gün | Mevcut çözümde **3,7 gün** duruş görülür, sistem yeni öneriye geçer |

---

## 7. Kiro'da kullanım ve önerilen repo yapısı

Dokümanlar Kiro steering formatındadır. `00_README_indeks.md` her zaman (`inclusion: always`), agent dokümanları ilgili klasörde, ortak şema ise `src/**` altında çalışılırken (`inclusion: fileMatch`) devreye girer. Kendi klasör yapınız farklıysa `fileMatchPattern` değerlerini değiştirmeniz yeterli.

```
.kiro/steering/            ← bu dokümanlar + nexus-data.md + kural md'leri
src/orchestrator/          ← 00
src/agents/signal/         ← 01
src/agents/risk/           ← 02
src/agents/impact/         ← 03
src/agents/options/        ← 04 (sourcing/, logistics/)
src/agents/decision/       ← 05
src/contracts/             ← ortak şema (Pydantic v2 + JSON Schema dışa aktarımı)
src/tools/                 ← deterministik hesap tool'ları (agent'lar arası paylaşılmaz, sahibi tek agent)
src/data_access/           ← Excel okuma katmanı (yalnızca GİRDİ dosyası)
config/                    ← ortam değişkenleri şablonu (gizli bilgi yok)
data/nexus_agent_input.xlsx
tests/fixtures/nexus_answer_key.xlsx
```

Mimari, rules §4 gereği modülerdir: ajan, tool'lar, veri erişimi ve konfigürasyon ayrı tutulur.
