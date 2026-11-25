---
inclusion: fileMatch
fileMatchPattern: "src/**"
---

# NEXUS – Ortak Veri Şemaları (Agent'lar arası sözleşmeler)

> **Amaç:** Agent'lar arasında taşınan tüm JSON çıktılarının **tek ve kanonik** tanımı. Agent veri md'lerindeki "Çıktı sözleşmesi" örnekleri bu dosyanın kısaltılmış hâlidir; alan adı, tip ve zorunluluk konusunda çelişki olursa **bu dosya esastır**.
> **Kaynak:** nexus-data.md §3 (kod formatları), §5 (kurallar), §8 (okuma tuzakları) · agent veri md'leri · Hazırlık Kiti §8 ("Agent'lar arası veri formatı: her agent'ın çıktısı bir sonrakinin girdisi")
> **Önerilen repo yolu:** `src/contracts/schemas.py` (Pydantic v2) → `src/contracts/json/*.json` (JSON Schema dışa aktarımı)

---

## 1. Sözleşme haritası

| Geçiş | Model | Üretici | Tüketici(ler) |
|---|---|---|---|
| Sinyal → analiz | `SignalOutput` | Signal | Orkestratör, Risk, Impact, Options (`alternatif_ipucu`) |
| Olasılık / risk kapısı | `RiskOutput` | Risk | Orkestratör (kapı), Decision (kart bağlamı) |
| Etki | `ImpactOutput` | Impact | Risk (`etki_puani`), Options, Decision |
| Seçenekler | `OptionsOutput` | Options | Decision |
| Öneri | `DecisionOutput` | Decision | Orkestratör → onay ekranı |
| What-if tetiği | `WhatIfIstegi` | Arayüz / parametre | Orkestratör |
| İnsan kararı | `InsanKarari` | Onay ekranı | Orkestratör |
| Karar kaydı | `KararKaydi` | Orkestratör | Kayıt (`Karar_Kaydi` formatı) |

**LLM sınır modelleri** (LLM'in doldurabileceği tek yapılar, bkz. §6): `SignalLLMCikarimi`, `DisKosulBileseni`, `DecisionMetinLLM`.

---

## 2. Genel kurallar

| # | Kural |
|---|---|
| G1 | Her agent çıktısı **`Zarf`** alanlarını taşır (`run_id`, `agent`, `d0`, `durum` vb.). |
| G2 | `extra="forbid"`: şemada olmayan alan kabul edilmez. Uydurma alan, uydurma sütun adı gibi bir hatadır. |
| G3 | **Para:** TRY, alan sonu `_try`. USD gerekiyorsa `_usd`. Kur `Parametreler`'den gelir. |
| G4 | **Ağırlık:** brüt kg, alan sonu `_kg` (lojistik daima brüt kg). Orijinal miktar `miktar` + `olcu_birimi` (`KG` / `TON` / `ADET`) ile taşınır. |
| G5 | **Süre / gün:** `_gun`, float (kesirli gün serbest). **Gün indeksi** = (tarih − D0).days, D0 = `Parametreler → AnalizTarihi`. |
| G6 | **Tarih:** ISO-8601 `date` (`2026-12-10`). **Zaman:** ISO-8601 `datetime`. Veri tarihleri Excel'deki gibi yerel saat; çalışma zaman damgaları (`baslangic_zamani` vb.) `+03:00` ofsetli. |
| G7 | **Boş değer:** girdideki `"-"` → `null`. Bilinmeyen değer `null` + `uyarilar` kaydı olarak döner. **0 veya tahminle doldurulmaz.** |
| G8 | **Kimlikler** §3'teki desenlere uymalıdır ve girdi dosyasında karşılığı bulunmalıdır. Desen kontrolünü şema, var olma kontrolünü tool yapar. |
| G9 | **Çoklu değer** (ör. bir sinyal → birden çok malzeme) daima **liste** olarak taşınır; `"; "` ile birleştirilmiş string olarak taşınmaz. |
| G10 | **Sayı sahipliği:** Sayısal alanlar yalnızca deterministik tool'lar tarafından doldurulur. Tek istisna `DisKosulBileseni.puan` (LLM, gerekçe ve kanıt zorunlu). |
| G11 | Gerçek kişisel veri taşıyan alan yoktur. Kişi yerine **rol** yazılır (`onaylayan_rol = "Satınalmacı"`) (D1, D2). |
| G12 | **Versiyonlama:** Şema değişikliği önce bu dosyada yapılır, `SEMA_VERSIYONU` artırılır, sonra kod güncellenir. |

---

## 3. Kimlik desenleri (nexus-data.md §3)

| Kimlik | Desen (regex) | Örnek |
|---|---|---|
| Kuruluş | `^(KARO\|BANYO\|MADEN\|SAGLIK)$` | `SAGLIK` |
| Tesis / depo | `^[TD]-(KR\|BN\|MD\|SG)-0\d$` | `T-SG-01`, `D-SG-01` |
| Malzeme | `^MT1000\d{2}$` | `MT100054` |
| Mamul | `^MM3000\d{2}$` | — |
| Hat | `^HAT-(KR\|BN\|SG\|MD)-0\d$` | `HAT-SG-03` |
| Tedarikçi | `^TD5\d{3}$` | `TD5012` |
| Taşıyıcı | `^TS0[1-9]$` | `TS0x` |
| Sinyal kaynağı | `^KY(0[1-9]\|1[0-2])$` | `KY01` |
| Sinyal | `^SG-26(10\|11)-0\d{2}H?$` | `SG-2611-001` |
| Sevkiyat | `^SV-26-\d{4}$` | `SV-26-1184` |
| Takip No | `^[A-Z0-9]+-[A-Z0-9]+-26-\d{4}$` | `AKH-MUN-26-1184` |
| PO | `^45002\d{5}$` (**string**) | `4500201290` |
| Rota | `^R-(SEA\|AIR\|TRK\|COD\|CST\|EXP\|RAIL)-.+$` | `R-AIR-…` |
| Müşteri siparişi | `^SO-26-5\d{3}$` | `SO-26-5xxx` |
| Risk | `^R-\d{2}$` | `R-01` |
| Strateji | `^S[1-3]-[A-Z0-9-]+$` | `S1-HIB`, `S2-HIB-A` |
| Karar | `^NXS-26-0\d{2}$` | `NXS-26-0xx` |
| Stok anahtarı | `^[TD]-(KR\|BN\|MD\|SG)-0\d\|MT1000\d{2}$` | `T-SG-01\|MT100054` |
| Kaynak anahtarı | `^MT1000\d{2}\|TD5\d{3}$` | `MT100054\|TD5012` |

---

## 4. Ortak enum'lar

| Enum | Değerler | Kullanım |
|---|---|---|
| `Agent` | `orkestrator`, `signal`, `risk`, `impact`, `options`, `decision` | Zarf |
| `Durum` | `TAMAM`, `UYARI`, `HATA` | Zarf |
| `IslemeKarari` | `ANALIZ`, `AYIKLA` | Signal |
| `Kademe` | `Yüksek`, `Orta`, `Düşük` | Risk (eşikler `Parametreler`'den) |
| `BilesenTipi` | `TRANSFER`, `ALTERNATIF_TEDARIK_ROTA`, `ROTA_DEGISIKLIGI` | Options. "Bekle" stratejisinin bileşeni yoktur (`bilesenler = []`). |
| `KisitKodu` | `C1`…`C6` | Options kısıtları (04 veri md §5) |
| `KisitSonuc` | `GEÇTİ`, `KALDI` | Options (`Veri_Kontrolleri` diliyle aynı) |
| `KararDurumu` | `Onaylandı`, `Reddedildi`, `Revize Edildi`, `Onay Bekliyor` | Karar kaydı |
| `UyariKodu` | `ESLESME_YOK`, `COKLU_ADAY`, `CELISKI`, `EKSIK_BILESEN`, `KANIT_YETERSIZ`, `BIRIM_UYUMSUZ`, `HAZIR_DEGER_FARKI`, `KISIT_KALDI`, `UYGUN_STRATEJI_YOK`, `NORMALIZE_TANIMSIZ`, `MILP_YAPILAMADI`, `LLM_HATA`, `SEMA_HATASI` | Tüm agent'lar (hata tabloları) |

---

## 5. Pydantic v2 modelleri (kanonik)

```python
# src/contracts/schemas.py
from __future__ import annotations
from datetime import date, datetime
from typing import Annotated, Literal, Optional
from pydantic import BaseModel, ConfigDict, Field, StringConstraints, model_validator

SEMA_VERSIYONU = "1.0"

# ---------- Kimlik tipleri (§3) ----------
def _id(p: str):
    return Annotated[str, StringConstraints(pattern=p)]

KurulusKodu   = _id(r"^(KARO|BANYO|MADEN|SAGLIK)$")
TesisKodu     = _id(r"^[TD]-(KR|BN|MD|SG)-0\d$")
MalzemeNo     = _id(r"^MT1000\d{2}$")
HatKodu       = _id(r"^HAT-(KR|BN|SG|MD)-0\d$")
TedarikciNo   = _id(r"^TD5\d{3}$")
TarafNo       = _id(r"^(TD5\d{3}|TS0[1-9])$")
SinyalNo      = _id(r"^SG-26(10|11)-0\d{2}H?$")
SevkiyatNo    = _id(r"^SV-26-\d{4}$")
RotaKodu      = _id(r"^R-(SEA|AIR|TRK|COD|CST|EXP|RAIL)-.+$")
SiparisNo     = _id(r"^SO-26-5\d{3}$")
RiskNo        = _id(r"^R-\d{2}$")
StratejiKodu  = _id(r"^S[1-3]-[A-Z0-9-]+$")
KararNo       = _id(r"^NXS-26-0\d{2}$")
StokAnahtari  = _id(r"^[TD]-(KR|BN|MD|SG)-0\d\|MT1000\d{2}$")

Puan15 = Annotated[int, Field(ge=1, le=5)]

# ---------- Enum'lar (§4) ----------
Agent        = Literal["orkestrator", "signal", "risk", "impact", "options", "decision"]
Durum        = Literal["TAMAM", "UYARI", "HATA"]
Senaryo      = Literal["S1", "S2", "S3"]
Kademe       = Literal["Yüksek", "Orta", "Düşük"]
BilesenTipi  = Literal["TRANSFER", "ALTERNATIF_TEDARIK_ROTA", "ROTA_DEGISIKLIGI"]
KisitKodu    = Literal["C1", "C2", "C3", "C4", "C5", "C6"]
KisitSonuc   = Literal["GEÇTİ", "KALDI"]
KararDurumu  = Literal["Onaylandı", "Reddedildi", "Revize Edildi", "Onay Bekliyor"]
UyariKodu    = Literal["ESLESME_YOK", "COKLU_ADAY", "CELISKI", "EKSIK_BILESEN", "KANIT_YETERSIZ",
                       "BIRIM_UYUMSUZ", "HAZIR_DEGER_FARKI", "KISIT_KALDI", "UYGUN_STRATEJI_YOK",
                       "NORMALIZE_TANIMSIZ", "MILP_YAPILAMADI", "LLM_HATA", "SEMA_HATASI"]


class Sozlesme(BaseModel):
    model_config = ConfigDict(extra="forbid")


# ---------- Ortak parçalar ----------
class Uyari(Sozlesme):
    kod: UyariKodu
    mesaj: str
    alan: Optional[str] = None            # ilgili çıktı alanı


class KaynakRef(Sozlesme):
    sayfa: str                            # GİRDİ dosyasındaki sayfa adı, birebir
    anahtarlar: list[str]                 # PK değerleri (ör. "T-SG-01|MT100054")
    sutunlar: list[str] = []              # okunan sütun adları, birebir (opsiyonel)


class Zarf(Sozlesme):
    sema_versiyonu: str = SEMA_VERSIYONU
    run_id: str
    agent: Agent
    senaryo: Optional[Senaryo] = None     # girdide "-" ise None (izleme/gürültü)
    risk_no: Optional[RiskNo] = None
    d0: date                              # Parametreler → AnalizTarihi
    what_if_gun: float = 0                # Parametreler → WhatIfGun (uygulandığı hâli)
    baslangic_zamani: datetime
    bitis_zamani: datetime
    durum: Durum
    uyarilar: list[Uyari] = []

    @model_validator(mode="after")
    def _zaman(self):
        if self.bitis_zamani < self.baslangic_zamani:
            raise ValueError("bitis_zamani < baslangic_zamani")
        return self


# ---------- 01 · Signal ----------
class SignalKanit(Sozlesme):
    metin_alintilari: list[str]           # Ham Metin'den BİREBİR alıntılar
    eslesme_yolu: Optional[str] = None    # ör. "Takip No … → Sevkiyatlar.Takip No (Sahte) → SV-26-1184"


class SignalOutput(Zarf):
    agent: Literal["signal"] = "signal"
    sinyal_no: SinyalNo
    isleme_karari: Literal["ANALIZ", "AYIKLA"]
    ayiklama_nedeni: Optional[str] = None
    olay_tipi: Optional[str] = None                      # LLM
    ulke_bolge: Optional[str] = None                     # LLM
    ilgili_taraf: list[TarafNo] = []                     # LLM ipucu → tool doğrulama
    ilgili_sevkiyat: list[SevkiyatNo] = []               # tool (eşleştirme)
    ilgili_malzeme: list[MalzemeNo] = []                 # tool (eşleştirme)
    tahmini_gecikme_gun: Optional[float] = None          # LLM; metinde yoksa None
    alternatif_ipucu: Optional[str] = None               # LLM; değerlendirilmez
    sinyal_guvenilirligi: Puan15                         # tool
    tespit_gecikmesi_saat: Optional[float] = None        # tool
    bagli_risk_no: Optional[RiskNo] = None               # zarf risk_no ile aynı olmalı
    kanit: SignalKanit
    kaynak: list[KaynakRef]
    dogrulama_uyarilari: list[Uyari] = []

    @model_validator(mode="after")
    def _kurallar(self):
        if self.isleme_karari == "AYIKLA" and not self.ayiklama_nedeni:
            raise ValueError("AYIKLA için ayiklama_nedeni zorunlu")
        if self.bagli_risk_no and self.risk_no and self.bagli_risk_no != self.risk_no:
            raise ValueError("bagli_risk_no ≠ zarf risk_no")
        return self


# ---------- 02 · Risk ----------
class PerformansBileseni(Sozlesme):
    puan: Optional[Puan15] = None                        # tool (Esik_Tablolari)
    taraf: Optional[TarafNo] = None                      # TSxx → Tasiyicilar, TD5xxx → Tedarikciler
    kaynak_sutun: Optional[str] = None                   # "Zamanında Varış Oranı" | "OTIF Oranı"
    secim_gerekcesi: Optional[str] = None                # LLM


class DisKosulBileseni(Sozlesme):                        # LLM SINIR MODELİ (G10 istisnası)
    puan: Optional[Puan15] = None                        # kanıt yetersizse None
    gerekce: Optional[str] = None
    kanit_sinyaller: list[SinyalNo] = []


class RiskBilesenleri(Sozlesme):
    sinyal: Puan15
    performans: PerformansBileseni
    dis_kosul: DisKosulBileseni
    finansal: Optional[Puan15] = None


class RiskOutput(Zarf):
    agent: Literal["risk"] = "risk"
    bilesenler: RiskBilesenleri
    olasilik: Optional[float] = Field(None, ge=1, le=5)  # tool; eksik bileşende None
    etki: Optional[Puan15] = None                        # = ImpactOutput.etki_puani
    risk_puani: Optional[float] = Field(None, ge=1, le=25)
    risk_kademesi: Optional[Kademe] = None
    agent_aksiyonu: Optional[str] = None                 # tam analiz + onaya sun | uyarı + seçenek hazırda | izle
    aciklama: Optional[str] = None                       # LLM, kısa Türkçe
    kaynak: list[KaynakRef]

    @model_validator(mode="after")
    def _kurallar(self):
        if self.etki is None or self.olasilik is None:     # R-K4: etki yoksa puan/kademe yok
            if self.risk_puani is not None or self.risk_kademesi is not None:
                raise ValueError("etki/olasılık yokken risk_puani/kademe üretilemez")
        elif self.risk_puani is not None and abs(self.risk_puani - self.olasilik * self.etki) > 0.01:
            raise ValueError("risk_puani ≠ olasilik × etki")
        return self


# ---------- 03 · Impact ----------
class RiskAltindakiSiparis(Sozlesme):
    siparis_no: SiparisNo
    hat_kodu: Optional[HatKodu] = None
    gerekli_uretim_tarihi: Optional[date] = None
    gecikme_gun: float = Field(ge=0)
    gunluk_ceza_try: Optional[float] = Field(None, ge=0)
    ceza_try: float = Field(ge=0)


class ImpactOutput(Zarf):                                # malzeme başına bir çıktı (çoklu eşleşmede birden çok)
    agent: Literal["impact"] = "impact"
    sevkiyat_no: SevkiyatNo
    malzeme_no: MalzemeNo
    tesis_kodu: TesisKodu
    hat_kodu: HatKodu
    olcu_birimi: Optional[Literal["KG", "TON", "ADET"]] = None
    serbest_stok: float = Field(ge=0)                    # yalnızca Serbest Stok
    gunluk_tuketim: float = Field(gt=0)
    stok_kapsama_gun: float = Field(ge=0)
    tukenme_tarihi: datetime                             # kesirli gün → saat dahil
    planlanan_kullanilabilir_tarih: date
    guncel_kullanilabilir_tarih: date                    # varış + gümrük + iç nakliye + kalite (+ WhatIfGun)
    kullanilabilir_gun: float                            # gün indeksi (G5)
    acik_durus_gun: float = Field(ge=0)
    gunluk_durus_maliyeti_try: float = Field(ge=0)
    yeniden_baslatma_maliyeti_try: float = Field(ge=0)
    durus_maliyeti_try: float = Field(ge=0)
    risk_altindaki_siparisler: list[RiskAltindakiSiparis] = []
    olasi_ceza_try: float = Field(ge=0)
    aksiyonsuz_maliyet_try: float = Field(ge=0)
    etki_puani: Puan15
    kaynak: list[KaynakRef]

    @model_validator(mode="after")
    def _kurallar(self):
        if abs(self.aksiyonsuz_maliyet_try - (self.durus_maliyeti_try + self.olasi_ceza_try)) > 1:
            raise ValueError("aksiyonsuz ≠ duruş + ceza")
        if abs(self.olasi_ceza_try - sum(s.ceza_try for s in self.risk_altindaki_siparisler)) > 1:
            raise ValueError("olasi_ceza_try ≠ sipariş cezaları toplamı")
        return self


# ---------- 04 · Options ----------
class KisitKontrolu(Sozlesme):
    sonuc: KisitSonuc
    kontroller: list[KisitKodu]                          # uygulanan kısıtlar
    kalan_kisitlar: list[KisitKodu] = []                 # KALDI ise hangileri


class StratejiBileseni(Sozlesme):
    bilesen_no: int = Field(ge=1)
    tip: BilesenTipi
    kaynak: str                  # TRANSFER → Stok Anahtarı · ALTERNATIF_TEDARIK_ROTA → Kaynak Anahtarı · ROTA_DEGISIKLIGI → Sevkiyat No
    rota_kodu: Optional[RotaKodu] = None
    miktar: float = Field(gt=0)
    olcu_birimi: Optional[Literal["KG", "TON", "ADET"]] = None
    brut_kg: Optional[float] = Field(None, ge=0)
    hazir_gun: float = Field(ge=0)                       # gün indeksi (G5)
    lojistik_maliyet_try: float = Field(0, ge=0)
    ek_malzeme_maliyeti_try: float = 0                   # fiyat farkı; negatif olabilir
    test_maliyeti_try: float = Field(0, ge=0)
    co2_kg: float = Field(0, ge=0)
    uygulama_riski: Optional[Puan15] = None
    kisit_kontrolu: KisitKontrolu


class Strateji(Sozlesme):
    strateji_kodu: StratejiKodu
    strateji_adi: str
    bilesenler: list[StratejiBileseni] = []              # "Bekle" → []
    uygun_mu: bool
    eleme_nedeni: Optional[str] = None

    @model_validator(mode="after")
    def _kurallar(self):
        if not self.uygun_mu and not self.eleme_nedeni:
            raise ValueError("uygun olmayan strateji için eleme_nedeni zorunlu")
        return self


class ElenenKaynak(Sozlesme):
    kaynak: str                                          # Kaynak Anahtarı / Stok Anahtarı / Rota Kodu
    neden: str
    hazir_gun: Optional[float] = None                    # ör. kalifikasyon nedeniyle 56. gün


class OptionsOutput(Zarf):
    agent: Literal["options"] = "options"
    stratejiler: list[Strateji] = Field(min_length=1)
    elenen_kaynaklar: list[ElenenKaynak] = []
    kaynak: list[KaynakRef]

    @model_validator(mode="after")
    def _kurallar(self):
        if not any(len(s.bilesenler) == 0 for s in self.stratejiler):   # OP-K3
            raise ValueError("'Bekle / hiçbir şey yapma' stratejisi eksik")
        return self


# ---------- 05 · Decision ----------
class SimulasyonGunu(Sozlesme):
    gun: int = Field(ge=0, le=60)
    giris: float = Field(ge=0)                           # gün başı giriş
    gun_sonu_stok: float = Field(ge=0)
    durus_orani: float = Field(ge=0, le=1)               # kısmi duruş kesirli


class StratejiSimulasyonu(Sozlesme):
    strateji_kodu: StratejiKodu
    gunler: list[SimulasyonGunu] = Field(min_length=61, max_length=61)   # D0…D60


class MaliyetKalemleri(Sozlesme):
    durus: float = Field(ge=0)
    ceza: float = Field(ge=0)
    lojistik: float = Field(ge=0)
    ek_malzeme: float = 0
    test: float = Field(ge=0)
    stok_tasima: float = Field(ge=0)


class NormalizePuanlar(Sozlesme):                       # senaryo içi 0–1; bölen 0 ise None
    maliyet: Optional[float] = Field(None, ge=0, le=1)
    durus: Optional[float] = Field(None, ge=0, le=1)
    co2: Optional[float] = Field(None, ge=0, le=1)
    risk: Optional[float] = Field(None, ge=0, le=1)


class StratejiKarsilastirma(Sozlesme):
    strateji_kodu: StratejiKodu
    strateji_adi: Optional[str] = None
    uygun_mu: bool
    durus_gun: float = Field(ge=0)
    maliyet_kalemleri: MaliyetKalemleri
    toplam_maliyet_try: float = Field(ge=0)
    co2_kg: float = Field(ge=0)
    risk: Optional[float] = None
    normalize: NormalizePuanlar
    agirlikli_skor: Optional[float] = None
    sira: Optional[int] = Field(None, ge=1)              # uygun değilse None
    oneri: Optional[Literal["ÖNERİLEN"]] = None

    @model_validator(mode="after")
    def _kurallar(self):
        k = self.maliyet_kalemleri
        if abs(self.toplam_maliyet_try - (k.durus + k.ceza + k.lojistik + k.ek_malzeme + k.test + k.stok_tasima)) > 1:
            raise ValueError("toplam_maliyet_try ≠ kalemler toplamı")
        return self


class MilpSonucu(Sozlesme):                              # yalnızca S1
    durum: Literal["OPTIMAL", "COZUMSUZ", "ZAMAN_ASIMI"]
    x_T: Optional[float] = Field(None, ge=0)             # transfer (kg)
    x_A: Optional[float] = Field(None, ge=0)             # hava (kg)
    y_C: Optional[Literal[0, 1]] = None                  # COD (liman değişikliği)
    amac_try: Optional[float] = None
    cckv_ile_uyumlu: Optional[bool] = None


class OneriRakamlari(Sozlesme):
    durus_gun: float = Field(ge=0)
    toplam_maliyet_try: float = Field(ge=0)
    aksiyonsuz_maliyet_try: float = Field(ge=0)
    tasarruf_try: float


class DecisionMetinLLM(Sozlesme):                        # LLM SINIR MODELİ
    gerekce: str                                         # kısa, rakamlı, Türkçe; rakamlar yalnızca OneriRakamlari'ndan
    dikkat_notu: Optional[str] = None                    # MILP–ÇKKV çelişkisi vb.


class OneriKarti(Sozlesme):
    baslik: str                                          # ekran metni (iş birimi onaylı)
    gerekce: Optional[str] = None                        # DecisionMetinLLM.gerekce; LLM hatasında None
    dikkat_notu: Optional[str] = None
    onerilen_strateji_kodu: Optional[StratejiKodu] = None   # uygun strateji yoksa None
    rakamlar: OneriRakamlari
    butonlar: list[Literal["Onayla", "Reddet", "Değiştir"]] = ["Onayla", "Reddet", "Değiştir"]


class KararKaydiTaslagi(Sozlesme):
    olay_no: KararNo
    karar: Literal["Onay Bekliyor"] = "Onay Bekliyor"    # DC-K1
    tasarruf_try: Optional[float] = None
    oneri_zamani: datetime


class WhatIfKarsilastirma(Sozlesme):
    onceki_strateji_kodu: StratejiKodu
    onceki_stratejinin_yeni_durus_gun: float = Field(ge=0)   # S1 +7: ≈ 3,7
    yeni_strateji_kodu: Optional[StratejiKodu] = None
    oneri_degisti_mi: bool


class DecisionOutput(Zarf):
    agent: Literal["decision"] = "decision"
    simulasyon: list[StratejiSimulasyonu]
    karsilastirma: list[StratejiKarsilastirma] = Field(min_length=1)
    milp: Optional[MilpSonucu] = None
    oneri_karti: OneriKarti
    karar_kaydi_taslagi: KararKaydiTaslagi
    what_if_karsilastirma: Optional[WhatIfKarsilastirma] = None

    @model_validator(mode="after")
    def _kurallar(self):
        onerilen = [s for s in self.karsilastirma if s.oneri == "ÖNERİLEN"]
        if len(onerilen) > 1:
            raise ValueError("birden çok ÖNERİLEN")
        if onerilen:
            o = onerilen[0]
            if not o.uygun_mu or o.sira != 1:                                    # DC-K4
                raise ValueError("ÖNERİLEN, kısıtı geçen 1. sıra olmalı")
            if self.oneri_karti.onerilen_strateji_kodu != o.strateji_kodu:
                raise ValueError("kart stratejisi ≠ ÖNERİLEN")
        return self


# ---------- 00 · Orkestratör girdileri ve karar kaydı ----------
class WhatIfIstegi(Sozlesme):
    senaryo: Senaryo                                     # Parametreler → WhatIfSenaryo
    what_if_gun: float                                   # Parametreler → WhatIfGun (demo: 7)
    tetik: Literal["PARAMETRE", "YENI_SINYAL"]
    sinyal_no: Optional[SinyalNo] = None                 # YENI_SINYAL ise


class InsanKarari(Sozlesme):
    olay_no: KararNo
    karar: Literal["Onaylandı", "Reddedildi", "Revize Edildi"]
    onaylayan_rol: str                                   # rol; kişi adı DEĞİL (G11)
    karar_zamani: datetime
    revize_notu: Optional[str] = None                    # "Değiştir" seçildiyse


class KararKaydi(Sozlesme):
    olay_no: KararNo
    risk_no: RiskNo
    senaryo: Optional[Senaryo] = None
    sinyal_nolari: list[SinyalNo]
    sinyal_tespit_zamani: datetime
    oneri_zamani: datetime
    karar_zamani: Optional[datetime] = None
    karar: KararDurumu
    onaylayan_rol: Optional[str] = None
    onerilen_strateji_kodu: Optional[StratejiKodu] = None
    aksiyonsuz_maliyet_try: float = Field(ge=0)
    onerilen_maliyet_try: Optional[float] = Field(None, ge=0)
    tasarruf_try: Optional[float] = None
    tespit_suresi_dk: Optional[float] = Field(None, ge=0)
    oneri_suresi_dk: Optional[float] = Field(None, ge=0)
    onay_suresi_dk: Optional[float] = Field(None, ge=0)
    revize_notu: Optional[str] = None
```

---

## 6. LLM sınır modelleri (K6 uzlaşısının şemadaki karşılığı)

LLM, Bedrock üzerinden çağrıldığında **yalnızca aşağıdaki modelleri** doldurur. Sonuçları tool katmanı doğrular ve agent çıktısıyla birleştirir. LLM, `*Output` modellerini doğrudan üretmez.

| Agent | LLM modeli | LLM'in doldurduğu | Sonra ne olur? |
|---|---|---|---|
| Signal | `SignalLLMCikarimi` (aşağıda) | olay tipi, ülke/bölge, taraf adları, takip no / PO / gemi ipuçları, gecikme, alternatif ipucu, alıntılar | Tool'lar ipuçlarını kimliğe çevirir ve girdi dosyasında doğrular (G8) |
| Risk | `DisKosulBileseni` | dış koşul puanı + gerekçe + kanıt sinyalleri | Olasılık tool'da hesaplanır |
| Decision | `DecisionMetinLLM` | gerekçe, dikkat notu | Gerekçedeki rakamlar `OneriRakamlari` ile karşılaştırılır |

```python
class SignalLLMCikarimi(Sozlesme):                       # LLM SINIR MODELİ
    olay_tipi: Optional[str] = None
    ulke_bolge: Optional[str] = None
    taraf_adlari: list[str] = []                         # serbest metin adlar → tool: Tedarikciler / Tasiyicilar
    takip_no_ipuclari: list[str] = []                    # → tool: Sevkiyatlar.Takip No (Sahte)
    po_ipuclari: list[str] = []                          # → tool: Acik_Siparisler.PO No (str)
    gemi_adlari: list[str] = []                          # → tool: Sevkiyatlar.Son Konum / Not
    tahmini_gecikme_gun: Optional[float] = None          # metinde açık değilse None
    alternatif_ipucu: Optional[str] = None
    alintilar: dict[str, str] = {}                       # alan adı → Ham Metin'den birebir alıntı
```

Risk, Impact, Options ve Decision'daki diğer LLM katkıları (taraf seçimi, kayıt seçimi, strateji ailesi planı, tutarlılık değerlendirmesi) **metin/gerekçe** alanlarına yazılır ya da tool çağrı planı olarak kalır. Hiçbiri sayısal alan doldurmaz.

**Gerekçe rakam testi (Decision):** `DecisionMetinLLM.gerekce` içindeki her sayı, normalize edildikten sonra (ör. "110 bin" ≈ 109.567, "4 Aralık" = tükenme tarihi) `OneriRakamlari` veya `ImpactOutput` alanlarından birine karşılık gelmelidir. Karşılığı olmayan sayı → `LLM_HATA` uyarısı.

---

## 7. Test toleransları (tüm agent'lar)

| Büyüklük | Tolerans |
|---|---|
| Gün (kapsama, duruş, hazır gün) | ±0,01 |
| TRY | ±1 |
| kg | ±0,5 |
| Olasılık, risk puanı | ±0,01 |
| Normalize puan, ağırlıklı skor | ±0,001 |
| Kimlik alanları (SV, MT, TD, TS, R, S…) | birebir |
| Serbest metin (LLM) | anlamsal eşdeğerlik |

Beklenen değerler yalnızca `tests/fixtures/nexus_answer_key.xlsx`'ten okunur (nexus-data.md §1). What-if (`WhatIfGun = 7`) beklenen değerleri için cevap anahtarı Excel'de yeniden hesaplatılır.

---

## 8. Kullanım notları

- **JSON Schema dışa aktarımı:** `SignalOutput.model_json_schema()` vb. çıktılar `src/contracts/json/` altına yazılır. Strands tool tanımlarında ve AgentCore üzerindeki agent'lar arası çağrılarda aynı şema kullanılır.
- **Orkestratör doğrulaması:** Her geçişte `Model.model_validate(cikti)`. Hata olursa `SEMA_HATASI` uyarısı verilir, akış `HATA`'ya geçer ve çıktı bir sonraki agent'a verilmez (O2).
- **What-if:** Impact, Risk, Options ve Decision çıktıları yeni `run_id` ile ama **aynı** `risk_no` ile üretilir. `Zarf.what_if_gun` uygulanan değeri taşır, `DecisionOutput.what_if_karsilastirma` önceki öneriyle farkı gösterir.
- **Karar kaydının Excel'e yazımı:** `KararKaydi` alanları cevap anahtarındaki `Karar_Kaydi` sayfası başlıklarıyla eşlenir. nexus-data.md bu sayfanın sütun adlarını listelemediği için eşleme tablosu, başlıklar dosyadan okunduktan sonra bu bölüme eklenecek. Sütun adı tahmin edilmez.
- Pydantic bir Python kütüphanesidir. LLM veya servis değildir, izinli servis listesini (R3) değiştirmez.
