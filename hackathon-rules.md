---
inclusion: always
---

# Hackathon Kuralları — Agentic AI in Supply Chain

Bu proje bir hackathon kapsamında jüri tarafından değerlendirilecek. Aşağıdaki kurallar **zorunludur**. Kod üretirken, mimari önerirken veya dosya oluştururken her zaman bu kurallara uy. Bir talep bu kurallarla çelişiyorsa uygulamadan önce kullanıcıyı uyar.

## 1. LLM ve model erişimi
- Tüm LLM çağrıları **yalnızca Amazon Bedrock** üzerinden yapılır.
- OpenAI, Anthropic API, Gemini, Ollama, yerel model veya başka bir LLM sağlayıcısının SDK'sını/endpoint'ini **asla** ekleme ya da önerme.
- Ajanın karar verme adımlarında bir **reasoning LLM** (Bedrock üzerinden) kullan; kritik kararları sabit if/else mantığına indirgeme.

## 2. İzin verilen teknoloji yığını
Çözüm şu servis/araçlardan en az birini kullanmalı; mümkün olduğunca bunlarla kal:
- Amazon Bedrock
- Amazon Bedrock AgentCore — **en az bir temel bileşenini kullan** (ör. Runtime, Memory, Gateway, Identity, Observability)
- Agent SDK: **Strands Agents** veya **Nova Act SDK**
- Kiro (agentic geliştirme)
- Amazon Quick Desktop / Amazon Quick Web

Yeni bir framework veya servis önermeden önce bu listeyle uyumlu olup olmadığını kontrol et.

## 3. Ajan olma kriterleri (üçü birden sağlanmalı)
1. **Reasoning:** Kararlar LLM akıl yürütmesiyle verilir.
2. **Otonomi:** Ajan insan girdisiyle veya girdisiz otonom çalışabilmeli (ör. tetikleyici/zamanlanmış akış veya çok adımlı görev planlama).
3. **Entegrasyon:** En az bir API, veritabanı, harici araç (web arama, kod çalıştırma vb.) veya başka bir agent ile entegre olmalı. Bunları ajan **tool**'ları olarak tanımla.

## 4. Kod kuralları
- Kod hackathon sırasında **sıfırdan** yazılır. Hackathon öncesi yazılmış kod, hazır proje şablonu veya eski repo kopyalanmaz.
- Kod **tekrar üretilebilir** olmalı: bağımlılıkları sabitle (`requirements.txt` / `pyproject.toml`), kurulum ve deploy adımlarını `README.md`'de belgele.
- Mimariyi modüler tut (ajan, tool'lar, veri erişimi, konfigürasyon ayrı). Gizli bilgileri (anahtar, hesap bilgisi) koda gömme; ortam değişkeni veya AWS kimlik mekanizmalarını kullan.

## 5. Veri kuralları
- Yalnızca **anonimleştirilmiş veya sentetik** üretim verisi kullan.
- Gerçek kişisel veri (ad, TCKN, telefon, e-posta, adres vb.) içeren örnek veri **üretme**, ekleme veya loglama.
- Sentetik veri üretirken KVKK/GDPR'a uygun, kişi tanımlamayan alanlar kullan.

## 6. Deployment kuralları
- Ajan **Amazon Bedrock AgentCore'a deploy edilir** ve demo **AWS Cloud üzerinden** yapılır.
- **Yerelde çalışan agent değerlendirme dışıdır.** Yerel çalıştırma yalnızca geliştirme/test içindir; final akışı her zaman AgentCore deployment'ını hedeflemeli.
- Deploy edilen AWS hesap ID'si jüriye iletilecek; deployment adımlarını ve kullanılan hesap/bölge bilgisini README'de not et.

## 7. Değerlendirmeye göre öncelikler
Jüri ağırlıkları: **Etki %30 · Teknik Uygulama %30 · Yaratıcılık %20 · İşlevsellik %10 · Demo %10**. Buna göre:
- Gerçek bir tedarik zinciri problemine odaklan ve **ölçülebilir bir etki metriği** üretebilecek çıktılar tasarla (süre, maliyet, hata oranı vb.).
- Kurallardaki teknolojileri net biçimde kullan; temiz ve açıklanabilir bir mimari kur.
- Ajan beklendiği gibi çalışmalı ve ölçeklenebilir olmalı; hata durumlarını ele al.
- **Uçtan uca agentic workflow** 5 dakikalık canlı demoda gösterilebilir olmalı; akışı kısa ve net tut.

## 8. Teslimata hazırlık
Proje sonunda şunların hazır olmasını hedefle:
- Kod: GitHub linki veya zip
- Mimari çizim (kullanılan AWS servisleriyle)
- AgentCore üzerinde çalışan canlı demo + yedek video/ekran görüntüleri
- Sunum: PowerPoint (.pptx)
