# Kural Seti — Tedarik Zincirinde Agentic AI Hackathon'u

> **Kaynak:** Welcome Session sunumu (25 Kasım 2026). Yalnızca *Kurallar* (s.4), *Demo Sunumu* (s.6) ve *Değerlendirme Kriterleri* (s.8) sayfalarındaki teknik ve kapsam isterleri alınmıştır.
>
> **Etiketler:** `ZORUNLU` = uyulmazsa diskalifiye ya da puan kaybı riski · `ÖNERİLİR` = sunumda önerilen, zorunlu tutulmayan

---

## 1. Geliştirme Kuralları

| # | Kural | Seviye |
|---|---|---|
| R1 | Kod hackathon sırasında **sıfırdan** yazılır. Hackathon öncesinde yazılmış kod projeye eklenmez. | ZORUNLU |
| R2 | LLM çağrıları yalnızca **Amazon Bedrock** üzerinden yapılır. Bedrock dışında bir LLM sağlayıcısı ya da API kullanılmaz. | ZORUNLU |
| R3 | Çözümde aşağıdaki AWS servis ve araçlarından **en az biri** kullanılır. | ZORUNLU |
| R4 | Ajan, Amazon Bedrock AgentCore'un **en az bir temel bileşenini** kullanır. | ÖNERİLİR |

**R3 için izin verilen servis ve araçlar:**
- Amazon Bedrock
- Amazon Bedrock AgentCore
- AI Agent SDK: **Strands Agents** / **Nova Act SDK**
- Kiro (agentic geliştirme için kullanılabilir)
- Amazon Quick Desktop
- Amazon Quick Web

---

## 2. Yapay Zekâ Ajanı Kriterleri (AWS tanımı)

Çözüm, AWS'nin yapay zekâ ajanı tanımındaki üç kriterin **tamamını** karşılamalıdır:

| # | Kriter | Seviye |
|---|---|---|
| A1 | **Akıl yürütme:** Kararlar bir reasoning LLM (veya benzeri) ile verilir. | ZORUNLU |
| A2 | **Otonomi:** Ajan, insan girdisiyle ya da girdi olmadan otonom çalışabildiğini gösterir. | ZORUNLU |
| A3 | **Entegrasyon:** Ajan en az birine bağlanır: API, veritabanı, harici araç (web arama, kod çalıştırma vb.) veya başka agent'lar. | ZORUNLU |

---

## 3. Veri Kuralları

| # | Kural | Seviye |
|---|---|---|
| D1 | Yalnızca **anonimleştirilmiş** ya da **sentetik** üretim verisi kullanılır. | ZORUNLU |
| D2 | Veri güvenliği ve **KVKK/GDPR** uyumundan katılımcı firma sorumludur. | ZORUNLU |

---

## 4. Deployment Kuralları

| # | Kural | Seviye |
|---|---|---|
| P1 | Ajan(lar) **Amazon Bedrock AgentCore'a deploy edilir**. Demo **AWS Cloud üzerinden** yapılır. | ZORUNLU |
| P2 | **Yerelde (local) çalışan agent'lar değerlendirmeye alınmaz.** | ZORUNLU |
| P3 | Ajanın çalıştığı ya da deploy edildiği **AWS hesap ID'si** e-postayla jüriye iletilir. | ZORUNLU |

---

## 5. Teslimatlar ve Demo İsterleri

### 5.1 Sunumdan önce teslim edilecekler (e-posta ile)
- [ ] Sunum dosyası, **PowerPoint (.pptx)** formatında
- [ ] Kod: **GitHub linki** veya **zip** dosyası
- [ ] Çözümün deploy edildiği **AWS hesap ID'si**

### 5.2 Demo süresi
- Takım başına toplam **7 dakika**: **5 dk sunum** ve **2 dk soru-cevap**

### 5.3 Sunumda bulunması gerekenler
- [ ] Şirketin, ekibin ve yapılan işin kısa tanıtımı
- [ ] Bu fikrin / use-case'in neden seçildiği
- [ ] Çözümün şirkete katacağı değer ve etki
- [ ] Kullanılan servisler ve **mimari çizim**
- [ ] Geliştirilen agent'ın **canlı demosu** (AWS Cloud üzerinde)
- [ ] **Yedek plan:** Demo çalışmazsa kullanmak için video kaydı veya ekran görüntüleri

---

## 6. Değerlendirme Kriterleri ve Ağırlıklar

| # | Kriter | Ağırlık | Jürinin baktığı nokta |
|---|---|---|---|
| E1 | **Potential Value / Impact** | **30%** | Hangi gerçek problemi çözüyor? Etkisi ölçülebilir mi? |
| E2 | **Technical Execution** | **30%** | Kurallarda belirtilen teknolojiler kullanılmış mı? Mimari sağlam ve çözüm tekrar üretilebilir mi? |
| E3 | **Creativity** | **20%** | Problem ve yaklaşım ne kadar özgün? |
| E4 | **Functionality** | **10%** | Agent(lar) beklendiği gibi çalışıyor mu? Çözüm ölçeklenebilir mi? |
| E5 | **Demo Presentation** | **10%** | Uçtan uca agentic workflow gösteriliyor mu? Demo ne kadar kaliteli ve net? |

---

## 7. Son Kontrol Listesi (Teslimden Önce)

- [ ] Kodun tamamı hackathon sırasında yazıldı (R1)
- [ ] Bütün LLM çağrıları Bedrock üzerinden yapılıyor (R2)
- [ ] İzin verilen AWS servislerinden en az biri kullanıldı (R3); AgentCore'un bir temel bileşeni kullanıldı (R4)
- [ ] Kararlar reasoning LLM ile veriliyor (A1)
- [ ] Otonom çalışma gösterilebiliyor (A2)
- [ ] En az bir API, veritabanı, araç veya agent entegrasyonu var (A3)
- [ ] Veri anonimleştirilmiş ya da sentetik ve KVKK/GDPR'a uygun (D1–D2)
- [ ] Ajan AgentCore'a deploy edildi, demo bulutta çalışıyor; yerelde çalıştırma yok (P1–P2)
- [ ] AWS hesap ID'si, .pptx sunum ve kod (GitHub/zip) e-postayla gönderildi (P3, 5.1)
- [ ] Mimari çizim hazır
- [ ] Uçtan uca agentic workflow demosu 5 dakikaya sığıyor
- [ ] Yedek video veya ekran görüntüleri hazır
- [ ] Ölçülebilir bir etki metriği belirlendi (E1)
