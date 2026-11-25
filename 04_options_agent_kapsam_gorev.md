---
inclusion: fileMatch
fileMatchPattern: "src/agents/options/**"
---

# 04 · Options Agent — Agentic Kural Seti, Kapsam ve Görev

> **Referans:** Docx **ADIM 6 – "Ne yapabiliriz?"** (a, b, c) · Kit §5 kutu 3 (Options Agent, kod) · nexus-data.md §2 #4a (Sourcing: kural + LLM), #4b (Logistics: deterministik rota matrisi + LLM)
> **Veri kuralları:** `04_options_agent_veri_kurallari.md`

---

## 1. Rol

**"Ne yapabiliriz?"** sorusunu cevaplar. Satın alma (Sourcing) ve lojistik (Logistics) seçeneklerini **birlikte** kurar. Her strateji için hazır olma günü, maliyet kalemleri, CO2, uygulama riski ve kısıt kontrolünü üretir. Strateji **sıralamaz**; sıralama Decision'ın işidir.

> Docx: "Agent çözüm için 3 yere bakıyor: a) Başka depoda bu malzeme var mı? b) Başka tedarikçi var mı? c) Nasıl hızlı getiririz?"
> Kit §2: "Bu hibrit çözümü yalnızca satın almaya ya da yalnızca lojistiğe bakan bir sistem göremezdi."

---

## 2. Akıştaki yeri

| | |
|---|---|
| **Tetikleyici** | Orkestratör: Risk Kademesi **Yüksek** (tam analiz) veya **Orta** (seçenek hazırda tutulur) |
| **Girdi** | `ImpactOutput`, `SignalOutput.alternatif_ipucu`, girdi sayfaları, `Parametreler` |
| **Çıktı** | `OptionsOutput` → Decision Agent |
| **Ekran** | Ekran 3 · Seçenekler (strateji satırları) |
| **What-if** | Yeniden çalışır (zamanlama ve hibrit miktar değişir) |

---

## 3. Kapsam

| İçinde | Dışında |
|---|---|
| **Sourcing:** grup içi transfer (a), alternatif/yedek tedarikçi (b), fiyat farkı ve ek malzeme maliyeti | Strateji sıralaması, ÇKKV skoru (Decision) |
| **Logistics:** rota/mod/liman değişikliği (COD), ekspres, hava (c) | Duruş ve ceza maliyeti simülasyonu (Decision) |
| Signal'in alternatif ipucunu seçeneğe dönüştürmek (S1: Mersin) | Öneri metni (Decision) |
| Strateji kurgusu: bekle, tek bileşenli ve hibrit seçenekler | Sipariş vermek, tedarikçiyle iletişim |
| Kısıt kontrolü ve eleme gerekçesi | |

**S1'in 6 stratejisi (Kit §1, docx ADIM 7):** (1) Hiçbir şey yapma · (2) Sadece depodan transfer · (3) Sadece kısmi uçak · (4) Tamamını uçakla getir · (5) Gemiyi Mersin'e yönlendir · (6) Depodan 260 kg + uçakla 250 kg (hibrit).

---

## 4. Görev adımları

1. **İhtiyacı anlama (LLM muhakemesi):** `ImpactOutput`'tan açığı (kaç gün, kaç kg) ve zaman penceresini (stok bitişi → malzeme gelişi) yorumlar ve hangi strateji ailelerinin anlamlı olduğunu planlar.
2. **a) Transfer (Sourcing tool):** Aynı malzemenin diğer tesis/depolardaki `Transfer Edilebilir Miktar`'ını bulur (S1: D-SG-01 → 260 kg).
3. **b) Alternatif tedarikçi (Sourcing tool + LLM):** `Tedarik_Kaynaklari`'nı tarar; iniş maliyeti, hazır stok, MOQ ve kalifikasyonu değerlendirir. LLM, `Özel Koşul`'u (GMP/ruhsat, ADR vb.) yorumlar ve eleme gerekçesini yazar.
4. **c) Rota (Logistics tool + LLM):** `Rota_Secenekleri`'nden uygun modları seçer. Signal'in ipucunu (Mersin COD) rota seçeneğine çevirir. Hazır günü, lojistik maliyeti ve CO2'yi hesaplatır.
5. **Strateji kurgusu (LLM planı + tool hesabı):** "Bekle" baz stratejisini, tek bileşenli stratejileri ve hibrit stratejiyi kurar. Hibrit miktarı O7 formülüyle tool'a hesaplatır (S1: 250 kg).
6. **Kısıt kontrolü (tool):** C1–C6. Geçemeyen bileşen/strateji `uygun_mu = false` ve gerekçeyle işaretlenir.
7. **Çıktı:** `OptionsOutput`.

---

## 5. Reasoning · Tool · Otonomi

| Kriter | Options Agent'ta nasıl sağlanır? |
|---|---|
| **A1 Reasoning** | Hangi strateji ailelerinin kurulacağı, metindeki ipucunun seçeneğe çevrilmesi, `Özel Koşul` ve kalifikasyon yorumları ile eleme gerekçeleri LLM muhakemesiyle yapılır. |
| **A2 Otonomi** | Risk kademesine bağlı olarak insan girdisi olmadan tetiklenir. |
| **A3 Entegrasyon** | Sourcing tool'ları: `find_transfer_sources`, `list_alt_suppliers`, `compute_landed_cost`, `check_qualification`. Logistics tool'ları: `list_routes`, `compute_ready_day`, `compute_logistics_cost`, `compute_co2`. Ortak: `compute_hybrid_qty`, `check_constraints`. |

---

## 6. Agentic kurallar (zorunlu)

| # | Kural |
|---|---|
| OP-K1 | **Yalnızca onaylı kaynaklar önerilir.** İlaçta ruhsatta kayıtlı olmayan tedarikçi elenir (Kit §3). |
| OP-K2 | "Kalifikasyon Gerekli" kaynak, kalifikasyon süresi ufuk dışına düştüğü için elenir; eleme nedeni ve hazır günü raporlanır. Kit örneği: kalifikasyonu olmayan Alman tedarikçi (45 gün) → malzeme 56. gün → işe yaramaz. |
| OP-K3 | "Bekle / hiçbir şey yapma" stratejisi **her zaman** listede yer alır (aksiyonsuz maliyet karşılaştırması için). |
| OP-K4 | Fiyatlar daima **iniş maliyeti** üzerinden karşılaştırılır; lojistik daima **brüt kg** üzerinden hesaplanır. |
| OP-K5 | LLM sayı üretmez. Miktar, gün ve maliyet tool'lardan gelir; LLM yalnızca strateji kurgusunu planlar ve gerekçe yazar. |
| OP-K6 | Options sıralama yapmaz, "önerilen" etiketi koymaz. |
| OP-K7 | Kur, tampon, koruma ufku ve test maliyetleri `Parametreler`'den okunur. |
| OP-K8 | Pahalı ama kısmen fayda sağlayan seçenekler (S1: Mersin COD, 1 gün kazandırır) elenmez; kısıtı geçiyorsa Decision'a gider. Karar maliyet/duruş karşılaştırmasıyla verilir. |

---

## 7. Hata durumları

| Durum | Davranış |
|---|---|
| Transfer kaynağı yok | Transfer stratejileri üretilmez; uyarı. Diğer aileler devam eder. |
| Hiç uygun alternatif tedarikçi yok | Yalnızca transfer/rota stratejileri ve "Bekle". |
| Rota kapasitesi miktara yetmiyor | Sevkiyat sayısı artırılır (O4) veya C6 ile elenir; gerekçe yazılır. |
| Tüm stratejiler kısıttan kalıyor | Yalnızca "Bekle" uygun döner; Orkestratör "uygun strateji yok" uyarısı verir. |

---

## 8. Kabul kriterleri

**S1:**
| Kontrol | Beklenen |
|---|---|
| a) Transfer | D-SG-01 → **260 kg** |
| b) Yedek tedarikçi | Zhejiang Kimya (TD5012): hazır **1.500 kg**, MOQ **250 kg**, **Ruhsatta Kayıtlı** |
| c) Rota | Şanghay → İstanbul uçak, **2 gün**, **4,60 USD/kg** |
| Hava bileşeni hazır günü | **8. gün** (stok 9,89. günde biter, yani malzeme stok bitmeden gelir) |
| Hibrit miktar | **250 kg** (hesap 172 kg, MOQ'ya yükseltilir) |
| Strateji sayısı | **6** |
| Mersin COD | Kısıtı geçer, Decision'a gider (1 gün kazandırır, pahalı) |

**Demo cümlesi (Kit §1):** "Sistem satın alma ve lojistiği birlikte değerlendiriyor. Mersin'e liman değişikliği 1 gün kazandırıyor ama pahalı."

**Örnek senaryolar:** S2 → Uşak (T-KR-02) transferi + Çan yerli kil (TD5002) hibriti · S3 → D-BN-01 servis deposu transferi + HK ekspres; TD5023 (Almanya) **elenir**.

**Jüri sorusu (Kit §3):** "Alternatif tedarikçi gerçekten kullanılabilir mi?" → Sistem yalnızca onaylı kaynakları önerir.

---

## 9. Kural uyum kontrolü

- [ ] LLM = Bedrock (R2) · Strands/Nova Act (R3)
- [ ] Strateji kurgusu ve eleme gerekçesi LLM muhakemesiyle (A1) · otonom (A2) · Sourcing/Logistics tool'ları (A3)
- [ ] Sentetik tedarikçi/taşıyıcı verisi (D1)
- [ ] AgentCore Runtime (P1, P2)
