---
inclusion: fileMatch
fileMatchPattern: "src/agents/signal/**"
---

# 01 · Signal Agent — Veri Kuralları

> **Kaynak:** nexus-data.md §2 (#1), §3, §4, §6, §7, §8 · Docx ADIM 1 · Kit §4 (1. Sinyal), §5 kutu 1
> **Agentic eşi:** `01_signal_agent_kapsam_gorev.md`

---

## 1. Veri kaynağı kuralı

- Runtime'da yalnızca `data/nexus_agent_input.xlsx` okunur.
- LLM'e verilen **tek serbest metin** alanı `Risk_Sinyalleri → Ham Metin (LLM Girdisi)` sütunudur.
- Cevap anahtarındaki `Risk_Sinyalleri` ek sütunları (`(LLM)` ekli olanlar ve devamı) **beklenen çıktıdır**. Yalnızca testte kullanılır; prompt'a, few-shot örneğe veya koda konmaz.

---

## 2. Okunan sayfalar ve sütunlar

### 2.1 `Risk_Sinyalleri` — 25 satır · PK `Sinyal No` · FK `Kaynak Kodu → Risk_Kaynaklari`
`Sinyal No` · `Yayın Zamanı` · `Tespit Zamanı` · `Kaynak Kodu` · `Kaynak Adı` · `Kategori` · `Dil` · `Ham Metin (LLM Girdisi)` · `Kaynak Taban Güvenilirlik` · `Doğrulandı mı?` · `Durum`

- **Dil:** EN / TR / UK→TR / ES→TR (`UK→TR` ve `ES→TR` çevrilmiş metinlerdir).
- **Durum değerleri:** Açık – Analizde · İzlemede · Bilgi · Değerlendiriliyor · Yanlış Alarm · Kapatıldı

### 2.2 `Risk_Kaynaklari` — 12 satır · PK `Kaynak Kodu` (KY01–KY12)
`Kaynak Kodu` · `Kaynak Adı` · `Kaynak Tipi` · `Taban Güvenilirlik (1-5)` · `Erişim Tipi` · `Güncelleme Sıklığı` · `Agent Okuma Yöntemi` · `Gerçek Hayattaki Karşılığı`

### 2.3 `Sevkiyatlar` — 14 satır · PK `Sevkiyat No` (eşleştirme için)
Kullanılan sütunlar: `Sevkiyat No` · `PO No` · `Kuruluş Kodu` · `Malzeme No` · `Malzeme Adı` · `Tedarikçi No` · `Taşıyıcı No` · `Taşıma Modu` · `Çıkış Noktası` · `Varış Noktası` · `Planlanan Varış (ETA)` · `Güncel Tahmini / Gerçekleşen Varış` · `ETA Sapması (gün)` · `Durum` · `Son Konum / Not` · `Sapma Nedeni` · `Takip No (Sahte)`

### 2.4 `Acik_Siparisler` — 10 satır · PK `PO No`
Kullanılan sütunlar: `PO No` · `Malzeme No` · `Tedarikçi No` · `Tesis Kodu` · `Sevkiyat No`

### 2.5 `Tedarikciler` — 11 satır · PK `Tedarikçi No` (TD5xxx)
Kullanılan sütunlar: `Tedarikçi No` · `Tedarikçi Adı (Sahte)` · `Ülke` · `Şehir/Bölge`

### 2.6 `Tasiyicilar` — 9 satır · PK `Taşıyıcı No` (TS01–TS09)
Kullanılan sütunlar: `Taşıyıcı No` · `Taşıyıcı Adı (Sahte)` · `Taşıma Modu` · `Hizmet Bölgesi`

### 2.7 `Malzemeler` — 7 satır · PK `Malzeme No` (MT1000xx)
Kullanılan sütunlar: `Malzeme No` · `Kuruluş Kodu` · `Malzeme Adı` (malzeme adının doğrulanması için)

---

## 3. Eşleştirme (join) kuralları

| Metinde geçen ipucu | Hedef sayfa → sütun | Sonuç anahtarı |
|---|---|---|
| Takip No (ör. `AKH-MUN-26-1184`) | `Sevkiyatlar → Takip No (Sahte)` | `Sevkiyat No` (ör. `SV-26-1184`) |
| PO numarası (ör. `your PO 45002xxxxx`) | `Acik_Siparisler → PO No` (**str**) | `Sevkiyat No`, `Malzeme No` |
| Gemi adı (ör. `MV AEGEAN STAR`) | `Sevkiyatlar → Son Konum / Not` (içerir araması) | `Sevkiyat No` |
| Tedarikçi adı | `Tedarikciler → Tedarikçi Adı (Sahte)` | `Tedarikçi No` (TD5xxx) |
| Taşıyıcı adı | `Tasiyicilar → Taşıyıcı Adı (Sahte)` | `Taşıyıcı No` (TSxx) |
| Eşleşen sevkiyat | `Sevkiyatlar → Malzeme No` → `Malzemeler` | `Malzeme No` (MT1000xx) |
| `Kaynak Kodu` | `Risk_Kaynaklari` | `Taban Güvenilirlik (1-5)` |

> **Takip No ≠ Sevkiyat No.** E-postada geçen numara doğrudan sevkiyat numarası olarak kullanılmaz; mutlaka `Takip No (Sahte)` üzerinden map edilir.

---

## 4. Deterministik kurallar (tool olarak kodlanır)

| Kural | Tanım |
|---|---|
| **Sinyal güvenilirliği** | `Taban Güvenilirlik (1-5)` + 1 (`Doğrulandı mı?` = "Evet" ise), en fazla 5 |
| **Gürültü filtresi** | `Durum` ∈ {Yanlış Alarm, Kapatıldı, Bilgi} **veya** `Sinyal No` `SG-2610-*` (geçmiş) **veya** `H` ekli (kapatılmış) → `AYIKLA`. Diğerleri Risk değerlendirmesine gider. |
| **Gecikme çapraz kontrolü** | LLM'in çıkardığı gecikme, eşleşen sevkiyatın `ETA Sapması (gün)` ve `Planlanan Varış (ETA)` → `Güncel Tahmini / Gerçekleşen Varış` farkıyla karşılaştırılır. Tutarsızlıkta uyarı üretilir. |
| **Tespit gecikmesi** | `Tespit Zamanı` − `Yayın Zamanı` (saat). Formül cevap anahtarındaki `Formuller` sayfasından teyit edilir. |
| **Boş değer** | `"-"` → `None` |
| **Çoklu değer** | Bir sinyal birden çok malzemeye bağlanabilir (ör. `"MT100032; MT100031"`, ayraç `"; "`). Çıktıda liste olarak tutulur. |
| **Risk gruplama** | Aynı olay/sevkiyata bağlanan sinyaller tek risk kaydında birleşir (S1: SG-2611-001 ve -002 → R-01). Format `R-0n`. |

---

## 5. Çıktı sözleşmesi — `SignalOutput`

> **Kanonik şema:** `00_ortak_veri_semalari.md` → `SignalOutput`. Aşağıdaki örnekte zarf alanları (`run_id`, `d0`, `durum` vb.) kısaltılmıştır. Alan adı, tip ve zorunluluk için şema esastır.

Kit §8: "Signal Agent çıktısı: sevkiyat no, malzeme no, gecikme günü, ipucu."

```json
{
  "sinyal_no": "SG-2611-001",
  "isleme_karari": "ANALIZ",
  "ayiklama_nedeni": null,
  "olay_tipi": "<LLM>",
  "ulke_bolge": "<LLM>",
  "ilgili_taraf": ["<TSxx | TD5xxx>"],
  "ilgili_sevkiyat": ["SV-26-1184"],
  "ilgili_malzeme": ["MT100054"],
  "tahmini_gecikme_gun": 8,
  "alternatif_ipucu": "Mersin'e liman değişikliği mümkün",
  "sinyal_guvenilirligi": "<1-5, tool>",
  "tespit_gecikmesi_saat": "<tool>",
  "senaryo": "S1",
  "bagli_risk_no": "R-01",
  "kanit": {
    "metin_alintilari": ["<Ham Metin'den birebir alıntı>"],
    "eslesme_yolu": "Takip No AKH-MUN-26-1184 → Sevkiyatlar.Takip No (Sahte) → SV-26-1184"
  },
  "kaynak": [{"sayfa": "Risk_Sinyalleri", "anahtarlar": ["SG-2611-001"]}],
  "dogrulama_uyarilari": []
}
```

`<LLM>` = Bedrock LLM'in çıkardığı alan · `<tool>` = deterministik kodun hesapladığı alan. Diğer değerler S1 için beklenen cevaplardır.

---

## 6. Test eşlemesi (cevap anahtarı → `Risk_Sinyalleri`)

| Çıktı alanı | Cevap anahtarı sütunu | S1 (SG-2611-001) beklenen |
|---|---|---|
| `tespit_gecikmesi_saat` | `Tespit Gecikmesi (saat)` | — (sayfadan) |
| `olay_tipi` | `Olay Tipi (LLM)` | — (sayfadan) |
| `ulke_bolge` | `Ülke/Bölge (LLM)` | — (sayfadan) |
| `ilgili_taraf` | `İlgili Taraf (LLM)` | — (sayfadan) |
| `ilgili_sevkiyat` | `İlgili Sevkiyat (LLM)` | **SV-26-1184** |
| `ilgili_malzeme` | `İlgili Malzeme (LLM)` | **MT100054** |
| `tahmini_gecikme_gun` | `Tahmini Gecikme (gün) (LLM)` | **8** |
| `alternatif_ipucu` | `Alternatif İpucu (LLM)` | **Mersin'e liman değişikliği mümkün** |
| `sinyal_guvenilirligi` | `Sinyal Güvenilirliği (1-5)` | — (sayfadan) |
| `senaryo` | `Senaryo` | **S1** |
| `bagli_risk_no` | `Bağlı Risk No` | **R-01** |

- **Serbest metin alanları** (`olay_tipi`, `alternatif_ipucu`) anlamsal eşdeğerlikle test edilir. **Kimlik alanları** (SV, MT, TD, TS, R) birebir eşitlikle test edilir.
- **Genelleme testleri:** S2 → SG-2611-004, -005, -006, -018 → SV-26-1203 / MT100001 · S3 → SG-2611-007, -008 → SV-26-1219 / MT100032 (çoklu malzeme olasılığı) · Gürültü → SG-2611-013…-017, SG-2610-*, `*H` → `AYIKLA`.

---

## 7. Okuma tuzakları (Signal'e özgü)

- `PO No` pandas'ta int okunur; `dtype={"PO No": str}` verilmeden join yapılmaz.
- Gemi adı ayrı bir sütunda değil, `Son Konum / Not` içinde geçer.
- Çevrilmiş metinlerde (`UK→TR`, `ES→TR`) özel isimler orijinal dilde kalabilir; eşleştirme normalize edilmiş (büyük/küçük harf, boşluk) metinle yapılır.
- Gürültü sinyalleri bilerek eklenmiştir ve filtreleme mantığı test edilecektir.

---

## 8. Yasaklar

- Cevap anahtarındaki `(LLM)` sütunlarını few-shot örneği olarak kullanmak.
- Girdi dosyasında olmayan bir `Sevkiyat No` / `Malzeme No` üretmek (LLM halüsinasyonu). Her kimlik girdi dosyasındaki bir kayıtla doğrulanır.
- Ham metindeki olası kişisel bilgileri (e-posta imzası vb.) çıktıya veya loga taşımak.
