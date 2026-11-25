---
inclusion: fileMatch
fileMatchPattern: "src/orchestrator/**"
---

# 00 · Orkestratör — Veri Kuralları

> **Kaynak:** nexus-data.md §1, §2 (#0), §4 (`Parametreler`), §6 (what-if), §7 (`Karar_Kaydi`, `Agent_Haritasi`, `Veri_Kontrolleri`), §9
> **Agentic eşi:** `00_orkestrator_kapsam_gorev.md`

---

## 1. Veri kaynağı kuralı (en önemli kural)

| Dosya | Repo yolu | Orkestratör ne yapar? |
|---|---|---|
| GİRDİ | `data/nexus_agent_input.xlsx` | Dosyayı **bir kez** yükler ve agent'lara aynı veri nesnesini verir. Agent'lar dosyayı ayrı ayrı yüklemez. |
| CEVAP ANAHTARI | `tests/fixtures/nexus_answer_key.xlsx` | **Asla** yüklemez. Yalnızca test suite'i okur. |

- Orijinal dosya adlarındaki boşluk ve Türkçe `İ` karakteri Unicode sorunu çıkarır. Dosyalar ASCII adlarla (yukarıdaki yollar) kullanılır.
- `00_Rehber` sayfası serbest metindir, veri değildir ve yüklemeden sonra çıkarılır.

```python
data = pd.read_excel("data/nexus_agent_input.xlsx", sheet_name=None, dtype={"PO No": str})
data.pop("00_Rehber", None)
params = dict(zip(data["Parametreler"]["Parametre"], data["Parametreler"]["Değer"]))
D0 = pd.Timestamp(params["AnalizTarihi"])   # 2026-11-25
```

---

## 2. Orkestratörün doğrudan okuduğu veriler

### 2.1 `Parametreler` (key–value, PK: `Parametre`)
Sütunlar: `Parametre` · `Değer` · `Birim` · `Açıklama`

| Parametre | Değer | Orkestratördeki kullanımı |
|---|---|---|
| `AnalizTarihi` | 2026-11-25 | Tüm agent'lara D0 olarak verilir. Gün indeksi = (tarih − D0).days |
| `WhatIfSenaryo` | S1 | What-if'in uygulanacağı senaryo |
| `WhatIfGun` | 0 | Ek gecikme günü. 0'dan farklıysa what-if döngüsü tetiklenir |

Diğer parametreler (W_*, Esik*, kurlar, tampon, test maliyetleri) ilgili agent'ların veri md'lerinde tanımlıdır. Orkestratör bunları yorumlamaz, yalnızca `params` sözlüğünü iletir.

### 2.2 `Risk_Sinyalleri` (tetikleyici olarak, PK: `Sinyal No`)
Orkestratör yalnızca **yeni sinyal tespiti** için şu sütunları okur: `Sinyal No` · `Tespit Zamanı` · `Durum`.
`Ham Metin (LLM Girdisi)` sütununu okumaz ve yorumlamaz; bu Signal Agent'ın işidir.

---

## 3. Agent'lar arası veri sözleşmeleri (kayıt defteri)

Orkestratör her geçişte sözleşme şemasını doğrular (`Model.model_validate`). **Kanonik tanımlar:** `00_ortak_veri_semalari.md`; aşağıdaki tablo hangi agent dokümanında örnek bulunduğunu gösterir. Şema dışı veya eksik zorunlu alan içeren çıktı bir sonraki agent'a **verilmez**.

| Geçiş | Sözleşme | Tanımlandığı doküman |
|---|---|---|
| Signal → Risk, Impact | `SignalOutput` | `01_signal_agent_veri_kurallari.md` §5 |
| Impact → Risk, Options, Decision | `ImpactOutput` | `03_impact_agent_veri_kurallari.md` §6 |
| Risk → Orkestratör (kapı) | `RiskOutput` | `02_risk_agent_veri_kurallari.md` §5 |
| Options → Decision | `OptionsOutput` | `04_options_agent_veri_kurallari.md` §6 |
| Decision → Onay ekranı | `DecisionOutput` | `05_decision_agent_veri_kurallari.md` §7 |

Ortak alanlar (`Zarf`, her çıktıda zorunlu):
```json
{
  "sema_versiyonu": "1.0",
  "run_id": "<uuid>",
  "agent": "signal|risk|impact|options|decision",
  "senaryo": "S1",
  "risk_no": "R-01",
  "d0": "2026-11-25",
  "what_if_gun": 0,
  "baslangic_zamani": "<ISO-8601>",
  "bitis_zamani": "<ISO-8601>",
  "durum": "TAMAM|UYARI|HATA",
  "uyarilar": [{"kod": "<UyariKodu>", "mesaj": "...", "alan": null}]
}
```

---

## 4. Karar kaydı formatı (son çıktı)

Kit §4: "Agent'ın son çıktısı `Karar_Kaydi`'na bir satır." Format, cevap anahtarındaki `Karar_Kaydi` sayfasıdır (başlık 1. satır, PK `Olay No`).

| Alan | Kural |
|---|---|
| `Olay No` | Format `NXS-26-0xx` |
| Zaman damgaları | Sinyal tespiti, öneri üretimi ve insan kararı anları (süre KPI'ları için) |
| Karar | `Onaylandı` / `Reddedildi` / `Revize Edildi` / `Onay Bekliyor` (tam alan listesi: şemadaki `KararKaydi`) |
| Tasarruf | Decision çıktısından: aksiyonsuz maliyet − önerilen toplam maliyet |
| Süreler | Tespit, öneri ve onay süreleri |

- Decision Agent kaydı `Onay Bekliyor` durumuyla oluşturur. Durum alanını yalnızca Orkestratör, insan kararına göre günceller.
- Kayıtta gerçek kişi adı tutulmaz; onaylayan kişi rol olarak (ör. "Satınalmacı") yazılır (D1, D2).

---

## 5. Test eşlemesi (yalnızca test suite)

| Test | Cevap anahtarı sayfası | Okuma notu |
|---|---|---|
| Akış sırası ve agent → sayfa eşlemesi | `Agent_Haritasi` | Başlık **3. satır** (`header=2`) |
| Tutarlılık testleri (örnek suite) | `Veri_Kontrolleri` | Başlık **3. satır**; hepsi `GEÇTİ` olmalı |
| Karar kaydı formatı | `Karar_Kaydi` | 27 olay; başlık 1. satır |
| Uçtan uca S1 | `Etki_Analizi`, `Risk_Skorlama`, `Strateji_Karsilastirma` | Bkz. ilgili agent veri md'leri |

> Cevap anahtarı formüllü bir dosyadır. pandas/openpyxl (`data_only=True`) önbellekteki değerleri okur. **What-if testleri için** `WhatIfGun` cevap anahtarında Excel'de değiştirilip dosya yeniden hesaplatılarak kaydedilmelidir; aksi hâlde beklenen değerler güncellenmez.

---

## 6. Yasaklar

- Cevap anahtarını runtime'da açmak, içinden değer kopyalamak.
- `WhatIfGun` / `WhatIfSenaryo` değerini kodda sabit yazmak.
- Agent çıktısındaki bir sayıyı Orkestratör içinde değiştirmek veya yeniden hesaplamak (sahibi ilgili agent'tır).
- Loglarda veya kayıtta gerçek kişisel veri tutmak.
