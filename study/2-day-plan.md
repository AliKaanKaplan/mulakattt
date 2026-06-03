# 2 Günlük Plan — QA Lead Mülakat Hazırlık

> Müşteri görüşmesi 2 gün sonra. Bu plan **mimari karar + senaryo anlatımı** ağırlıklı, ezber değil.

---

## Strateji

**Tüm 12.500 satırı okumayacaksın.** %20 high-leverage konu, mülakat'ın %80'ini cover eder. Plan bu prensibe göre.

**Hedef**: Lead seviyesinde **akıl yürütebilen**, **trade-off konuşabilen**, **deneyim hikayesi** anlatabilen birisi gibi görünmek.

---

## Gün 0 (Bugün gece, 30 dk)

Eğer bu gece vaktin varsa hazırlık:

- [ ] Kendine bir not defteri aç (kağıt veya digital): "Bilmediklerim", "Kapanması gereken sorular", "Hikayelerim"
- [ ] Müşteri görüşmesinin **formatını öğren**: Slide sunum mu, soru-cevap mu, ikisi de mi? (Mümkünse mülakat veren'e sor)
- [ ] **Pozisyon tanımını oku**: Hangi tech stack vurgulanmış? Hangi sorumluluklar? Bu 2 günde sadece **o vurgulu konulara** odaklan.

---

## GÜN 1 — Capybara Core + Hands-on (~6-7 saat)

### 🕐 Blok 1 (1 saat) — Kavramsal oturma

**Oku** (sadece bu bölümler):
- `capybara.md` bölüm **1, 2, 3** (Felsefe, Mimari, Temel Terimler)

**Çıktı:**
- "Capybara nedir, neden var, kim için" tek nefeste anlatabilmelisin (60 saniye)
- "page", "Session", "Element", "Result" terimlerini doğru kullanabilmelisin
- "Driver-agnostic" ne demek, neden önemli — bir mimari avantaj hikayesi

### 🕐 Blok 2 (2 saat) — DSL ve Auto-wait

**Oku:**
- `capybara.md` bölüm **9, 10, 11, 12, 16, 17, 18** (Navigation, Click, Form, Find, Matchers, Match Options, Auto-Wait)

**Vurgu:**
- ⭐ **Bölüm 18 (Auto-wait & synchronize)** — Mülakat'ın #1 trap'i: `have_no_*` vs `not_to`. **Bunu içselleştirmek zorundasın.**
- `fill_in` syntax (`with:` keyword)
- `find` vs `all` farkı
- `Capybara::Ambiguous` ne zaman fırlar

**Self-quiz (15 dk):**
- "`expect(page).not_to have_content('Loading')` neden kötü?" — 30 saniyede açıkla
- "`find(.btn)` 2 buton bulursa ne olur?" — Cevap + 3 farklı çözüm
- "Default max wait time kaç saniye, nasıl override edilir?"

### ☕ Mola (30 dk)

### 🕐 Blok 3 (2 saat) — Hands-on (KRİTİK)

**Bu blok atlanamaz.** Okuma retention'ı %20'dir; hands-on %60.

**Görev:** Basit bir Rails veya Sinatra app'i kur (veya github.com/teamcapybara/capybara'nın sample app'ini klonla):

```ruby
RSpec.describe "Login flow", type: :system do
  it "user logs in" do
    visit "/login"
    fill_in "Email", with: "test@x.com"
    fill_in "Password", with: "secret"
    click_button "Sign in"
    expect(page).to have_current_path("/dashboard")
  end
end
```

**Bilerek hata yap:**
- `fill` yaz (fill_in yerine) → hatayı gör
- `not_to have_content` yaz, sayfa yavaşken dene → flaky göreceksin
- `find(".btn")` ile ambiguous selector dene
- `sleep 3` ekle vs Capybara matcher kullan, süreyi karşılaştır

**Sebep:** Mülakat'ta "Capybara ile çalıştın mı?" sorusuna **"evet, şu hatayı yaşadık"** diye gerçek hikaye anlatabilesin.

### 🕐 Blok 4 (1 saat) — Locator + POM

**Oku:**
- `capybara.md` bölüm **19** (Built-in Selector Types — sadece tablo) ve **26** (POM)

**Vurgu:**
- Locator önceliği: `data-testid > label > id > CSS class > XPath`
- POM = senin yazdığın sınıflar, `page` = framework session (karıştırma)
- Method chaining (her action bir sonraki page'i döner)

### 🕐 Blok 5 (1 saat) — Mini Mock

Bana gel, 10 soruluk mini mock yap (Capybara fundamentals):
- Süre limit: her soruya 60 saniye düşünme
- Bilmediğinde "akıl yürütüyorum" formatında deneme yap (asla "bilmiyorum" deme)
- Skor hedefi: **6/10 ortalama**

**Akşam 30 dk:**
- Gün 1 boşlukları not et
- Yarın sabah bunlara tekrar bak

---

## GÜN 2 — Lead Boyutu + Ekosistem + Final Mock (~7 saat)

### 🕐 Blok 1 (45 dk) — Sabah review

- Dün notlarını gözden geçir (zayıf yerleri)
- ⭐ `capybara.md` bölüm **39** (Mülakat soruları tablosu) — 5 dakikada hızlı oku, "bu cevabı bilmiyorum" diyenleri işaretle

### 🕐 Blok 2 (1.5 saat) — Lead-level Capybara

**Oku:**
- `capybara.md` bölüm **32** (Flaky test strategy)
- `capybara.md` bölüm **34** (Performans optimizasyonu)
- `capybara.md` bölüm **36, 37** (Framework tasarımı, Mimari kararlar)

**Hazır olman gerekenler (her birini 90 saniyede anlatabil):**
- "Flaky test'i nasıl tespit edip çözersin?" — sleep yerine matcher, have_no_*, DB isolation
- "Hangi case Capybara'da, hangisi unit'te?" — test piramidi cevabı
- "Driver seçimini nasıl yaparsın?" — trade-off matrix

### 🕐 Blok 3 (1 saat) — Cucumber

**Oku:**
- `cucumber.md` bölüm **1** (BDD felsefesi)
- `cucumber.md` bölüm **20** (Best practices)
- `cucumber.md` bölüm **23** (Cucumber'a değer mi? — ⭐ Lead trap'i)

**Kritik cevap:** "Cucumber'ı ne zaman kullanırsın?"
> "Sadece non-tech paydaş gerçekten Gherkin okuyorsa. Aksi halde RSpec yeterli. 3 amigos session'ı çalışan team'de değerli; sadece QA yazıyorsa overhead."

Bu cevap **seni Lead seviyesinde** gösterir — araç vs amaç ayrımı.

### ☕ Mola (30 dk)

### 🕐 Blok 4 (1 saat) — Appium overview

**Oku** (sadece üst seviye):
- `appium.md` bölüm **1, 2** (Felsefe, Mimari)
- `appium.md` bölüm **9, 10** (Capabilities tablosu, Locator stratejisi)
- `appium.md` bölüm **35** (Lead mimari kararlar)

**Pozisyon mobile-heavy değilse** burayı %60 yüzeysel geç. Sadece şunu bilmen yeterli:
- Appium = W3C WebDriver proxy, native automation framework'ünü sürer
- Locator önceliği: accessibility_id > id > uiautomator/predicate > xpath
- noReset vs fullReset trade-off
- Cloud farm'lar: BrowserStack, LambdaTest, Sauce

### 🕐 Blok 5 (45 dk) — Gatling concept

**Oku:**
- `gatling.md` bölüm **18** (Open vs Closed Workload — ⭐ Lead trap'i)
- `gatling.md` bölüm **21** (Test türleri matrix)
- `gatling.md` bölüm **27** (Performans metrikleri — mean vs percentile)

**Critical:**
- "Gatling Ruby DSL'i yok, Scala/Java/Kotlin. Ruby team için ruby-jmeter veya Typhoeus alternatif."
- "Mean yanıltıcı, p95/p99 kullan"
- Open workload = arrival rate (gerçek traffic), Closed = sabit concurrent VU

### 🕐 Blok 6 (45 dk) — Hikaye hazırlığı (KRİTİK)

**3 hikaye yaz**, STAR formatında (Situation/Task/Action/Result):

1. **"Çözdüğüm bir flaky test problemi"** — gerçek değilse Gün 1'deki hands-on'dan üret
   - **S:** Bir test rastgele fail oluyordu
   - **T:** Root cause + suite stabilize
   - **A:** `not_to have_content` → `have_no_content` değiştirdim, animation disable ettim
   - **R:** Flake rate %15'ten %0.5'e düştü

2. **"Test mimari kararı verdiğim an"** — driver seçimi, POM mı yoksa düz spec mi, vb.

3. **"Team scaling sırasında karşılaştığım challenge"** — junior'a nasıl öğrettin, locator audit nasıl yaptın

**Yoksa uydurma — yalan söyleme.** Onun yerine "Ben okuduğum kadarıyla şöyle yapardım" diyebilirsin, ama hipotetik olduğunu belirt.

### 🕐 Blok 7 (1 saat) — Final Mock

Bana gel, 20-25 soruluk Lead-seviye mock yap:
- Karışık konular (Capybara %60, Cucumber %15, Appium %15, Gatling %10)
- Bazı sorular **trade-off** odaklı (X vs Y, neden, ne zaman)
- Bazıları **senaryo** odaklı ("şu durumda ne yapardın?")
- Hedef: **7/10 ortalama**

### 🕐 Blok 8 (30 dk) — Soft skills + mülakatcıya soru hazırla

Hazırla:
- **3-5 akıllı soru** mülakatcıya soracak (deneyim sinyali):
  - "Mevcut test suite'inizin flake rate'i nedir?"
  - "Test piramidiniz nasıl şu an? Hangi seviyeye ağırlık veriyorsunuz?"
  - "QA team'inde junior/senior dağılımı nasıl, onboarding sürecini nasıl yapıyorsunuz?"
  - "CI'da en uzun test suite ne kadar sürüyor? Optimization ihtiyacı var mı?"
  - "Mobile test stratejiniz var mı, varsa hangi tool?"

**Gece** (rahatlama):
- Ezber yapma, beyni yor
- Hafif tekrar (sabah 5 dk gözden geçirilebilecek özet notu)
- Erken yat

---

## Mülakat günü (kısa not)

**1 saat önce:**
- Sabah notlarını oku (5 dk özet)
- Su iç, kahve fazla içme (titreme yapar)
- Pozitif self-talk: "X yıl deneyimim var, bilmediğim yerde dürüstçe akıl yürüteceğim"

**Mülakat sırasında:**
- Bilmediğin soruda: "Şöyle akıl yürütüyorum: ..." (asla "bilmiyorum")
- Trade-off sorularında: "Şuna bağlı; eğer X varsa A, yoksa B"
- Karar sorularında: hep bir **gerekçe** ekle ("çünkü ...")
- Slow down — düşünmeden cevap verme; "iyi soru, bir saniye düşüneyim" demek profesyonel görünür

---

## Skor projeksiyonu

| Konum | Mevcut tahmin | 2 gün sonra hedef |
|---|---|---|
| Junior/Mid QA | 6/10 → ✅ Geçer | 8/10 → 💪 Kuvvetli |
| Senior QA | 4/10 → ⚠️ Sınır | 7/10 → ✅ Geçer |
| QA Lead | 2/10 → ❌ Zayıf | 5-6/10 → ⚠️ Sınır |

**Tahmin gerçekçi**: Lead seviyesi için ideal değil ama **pozisyonu kaçırmayacak** kadar gelişme mümkün, özellikle hikayelerin iyiyse.

---

## Hızlı referans — en kritik 10 konu

Bu 10 konu mülakat'ın en az %60'ını cover eder. Diğer her şey buna nazaran tali.

1. **Auto-wait + `synchronize`** — Capybara'nın can damarı
2. **`have_no_*` vs `not_to have_*`** — klasik trap
3. **`find` vs `all` vs `first`** — exception/wait davranışı
4. **Locator stratejisi** — data-testid > label > id > class > XPath
5. **POM vs Capybara `page`** — karıştırılmamalı
6. **Driver seçimi** — rack_test / selenium / cuprite trade-off
7. **Test piramidi** — neden Capybara %10
8. **Flaky test stratejisi** — sleep yerine matcher, DB isolation
9. **Open vs Closed workload** (Gatling) — Lead trap
10. **"Cucumber'a değer mi?"** — non-tech paydaş kriteri
