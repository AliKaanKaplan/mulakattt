# Cucumber — QA Lead Mülakat Hazırlık Rehberi (Ruby)

> Behavior-Driven Development (BDD) framework'ü. **Gherkin** doğal dilini kullanarak iş kuralı / test senaryolarını yazılır, Ruby step definition'ları ile koşturulur.
> Bu doküman **Capybara + Cucumber** (web) ve **Appium + Cucumber** (mobile) ekosistemlerine odaklıdır. Tüm örnekler Ruby'dir.

---

## İçindekiler

1. [Cucumber Nedir? BDD Felsefesi](#1-cucumber-nedir-bdd-felsefesi)
2. [Gherkin Dili — Temel Syntax](#2-gherkin-dili)
3. [Kurulum](#3-kurulum)
4. [Proje Yapısı](#4-proje-yapisi)
5. [Feature ve Scenario](#5-feature-ve-scenario)
6. [Step Definitions](#6-step-definitions)
7. [Background](#7-background)
8. [Scenario Outline & Examples](#8-scenario-outline)
9. [Data Tables](#9-data-tables)
10. [Doc Strings](#10-doc-strings)
11. [Tags ve Filtering](#11-tags-ve-filtering)
12. [Hooks — Before, After, Around, AfterStep](#12-hooks)
13. [World ve Helper'lar](#13-world)
14. [Parameter Types — Custom Transforms](#14-parameter-types)
15. [Profiles — cucumber.yml](#15-profiles)
16. [Cucumber + Capybara Entegrasyonu](#16-cucumber--capybara)
17. [Cucumber + Appium Entegrasyonu](#17-cucumber--appium)
18. [Reporting — pretty, HTML, JUnit, Allure](#18-reporting)
19. [Parallel Test Execution](#19-parallel-execution)
20. [Best Practices — Senaryo Yazımı](#20-best-practices)
21. [Sık Karşılaşılan Pitfall'lar](#21-pitfall-lar)
22. [Framework Tasarımı](#22-framework-tasarimi)
23. [QA Lead Seviyesi — Mimari Kararlar](#23-mimari-kararlar)
24. [En Sık Mülakat Soruları](#24-mulakat-sorulari)
25. [Çalışma Yol Haritası](#25-calisma-yol-haritasi)
26. [Kaynaklar](#26-kaynaklar)

---

## 1. Cucumber Nedir? BDD Felsefesi

**Cucumber**, **Behavior-Driven Development (BDD)** için geliştirilmiş açık kaynak test framework'üdür. **Gherkin** adında, neredeyse doğal İngilizce/Türkçe gibi okunan bir dil ile **iş kurallarını test senaryolarına** dönüştürür.

### BDD nedir, TDD'den farkı?

**TDD (Test-Driven Development):** "Fonksiyon `add(2,3)` 5 dönmeli." → Developer odaklı.

**BDD (Behavior-Driven Development):** "Müşteri sepetine 2 ürün eklediğinde toplam fiyat doğru hesaplanmalıdır." → İş paydaşı odaklı.

**Üç paydaş (3 amigos):**
1. **Product Owner / Business Analyst** — neyin test edileceğine karar verir
2. **Developer** — sistemin nasıl çalıştığını bilir
3. **QA / Tester** — senaryoları yazar ve koşturur

Cucumber bu üçünün **ortak dilde anlaşmasını** sağlar — Gherkin.

### Cucumber'ın temel avantajı (ve dezavantajı)

✅ **Avantaj:** Non-technical paydaşlar (Product, Business) senaryoları **okuyabilir** ve **yazıma katılabilir**. Living documentation.

❌ **Dezavantaj:** Step definition layer eklemek **bakım maliyeti** doğurur. Eğer non-tech paydaşlar senaryolara dokunmuyorsa, Cucumber katmanı **gereksiz fazlalık** olur — saf RSpec yeterli olabilir.

**Lead seviyesi karar:** Cucumber'a değer mi? → Sadece Product/Business **gerçekten okuyor ve yazıyor** ise.

### Cucumber-Ruby tarihi
- **2008:** Aslak Hellesøy tarafından geliştirildi, ilk Ruby framework'ü
- **2014:** Cucumber 2.x — `World` kavramı, hook iyileştirmeleri
- **2018:** Cucumber 3.x — Gherkin 6, Rule keyword
- **2021+:** Cucumber 7.x/8.x — modern Ruby (2.7+/3.x), parallel execution

---

## 2. Gherkin Dili — Temel Syntax

Gherkin **plain text** formatındadır, dosya uzantısı `.feature`. 70'ten fazla dil destekler.

### Anahtar kelimeler (İngilizce)

```gherkin
Feature: Giriş yapma
  Kullanıcı doğru kimlik bilgileri ile giriş yapabilmelidir.

  Background:
    Given giriş sayfası açık

  Scenario: Başarılı giriş
    When kullanıcı "user@example.com" e-postası ile giriş yapar
    Then dashboard sayfası görüntülenir
    And karşılama mesajı "Hoş geldin" gösterilir

  Scenario Outline: Çeşitli rollerle giriş
    When kullanıcı "<email>" e-postası ile giriş yapar
    Then "<dashboard>" sayfası görüntülenir

    Examples:
      | email             | dashboard      |
      | admin@example.com | /admin         |
      | user@example.com  | /home          |
      | guest@example.com | /public        |
```

### Anahtar kelimeler sözlüğü

| Anahtar | Açıklama |
|---|---|
| `Feature:` | Bir özelliği tanımlar. Bir `.feature` dosyasında bir adet olur. |
| `Rule:` | (Gherkin 6+) Feature içinde bir iş kuralını gruplandırır. |
| `Background:` | Tüm scenario'lardan önce çalışacak ortak adımlar. |
| `Scenario:` (= `Example:`) | Tek bir test senaryosu. |
| `Scenario Outline:` (= `Scenario Template:`) | Parametrik scenario, `Examples` tablosu ile çalışır. |
| `Given` | **Önkoşul** — sistemin başlangıç state'i ("verili olarak"). |
| `When` | **Aksiyon** — kullanıcının yaptığı şey ("o zaman yaptığında"). |
| `Then` | **Beklenti** — assertion ("o zaman olmalı"). |
| `And` / `But` | Önceki `Given/When/Then`'ı genişletir (sadece okunabilirlik için). |
| `*` (asterisk) | And/But yerine kullanılabilir. |
| `@tag` | Scenario veya feature etiketi (filtreleme için). |
| `#` | Yorum satırı. |
| `"""` | Doc string (çok satırlı text). |
| `|` | Data table delimiter. |

### Gherkin Türkçe desteği

Cucumber Gherkin'i Türkçe dahil 70+ dilde destekler. İlk satıra `# language: tr` eklenir:

```gherkin
# language: tr
Özellik: Giriş yapma
  Kullanıcı doğru kimlik bilgileri ile giriş yapabilmelidir.

  Senaryo: Başarılı giriş
    Diyelim ki giriş sayfası açık
    Eğer kullanıcı "user@example.com" e-postası ile giriş yaparsa
    O zaman dashboard sayfası görüntülenir
    Ve karşılama mesajı "Hoş geldin" gösterilir
```

**Türkçe anahtar kelimeler:**
| EN | TR |
|---|---|
| Feature | Özellik |
| Background | Geçmiş |
| Scenario | Senaryo |
| Scenario Outline | Senaryo Taslağı |
| Examples | Örnekler |
| Given | Diyelim ki |
| When | Eğer ki |
| Then | O zaman |
| And | Ve |
| But | Fakat / Ama |

**Lead seviyesi karar:** Team uluslararası ise İngilizce. Türkçe konuşan Product/Business varsa Türkçe — okunabilirlik amacının ta kendisi.

---

## 3. Kurulum

### Saf Cucumber-Ruby
```ruby
# Gemfile
source "https://rubygems.org"

group :test do
  gem "cucumber", "~> 9.2"
  gem "rspec-expectations"        # `expect(...).to ...` matcher'ları için
end
```

```bash
bundle install
bundle exec cucumber --init       # features/ klasör yapısını oluşturur
bundle exec cucumber              # tüm feature'ları koş
```

### Cucumber + Capybara (web)
```ruby
group :test do
  gem "cucumber", "~> 9.2"
  gem "capybara"
  gem "selenium-webdriver"
  gem "cuprite"                   # alternatif hızlı driver
  gem "rspec-expectations"
  gem "database_cleaner-active_record"   # Rails ise
  gem "site_prism"                # Page Object DSL (opsiyonel)
end
```

### Cucumber + Capybara + Rails
```ruby
group :test do
  gem "cucumber-rails", require: false   # rails generator + helper'lar
  gem "capybara"
  gem "selenium-webdriver"
end
```

```bash
rails generate cucumber:install
```

### Cucumber + Appium (mobile)
```ruby
group :test do
  gem "cucumber", "~> 9.2"
  gem "appium_lib_core", "~> 9.0"
  gem "rspec-expectations"
  gem "allure-cucumber"
  gem "parallel_tests"
end
```

---

## 4. Proje Yapısı

### Standart layout
```
.
├── Gemfile
├── Rakefile
├── cucumber.yml                          # profile config
├── config/
│   └── environments/
│       ├── staging.yml
│       └── production.yml
├── features/
│   ├── login.feature                     # iş senaryoları
│   ├── checkout.feature
│   ├── support/
│   │   ├── env.rb                        # global setup (en önce çalışır)
│   │   ├── hooks.rb                      # Before/After hook'ları
│   │   ├── world.rb                      # World extensions
│   │   ├── capybara_config.rb            # Capybara config (web ise)
│   │   ├── appium_session.rb             # Appium driver (mobile ise)
│   │   └── pages/                        # Page Object'ler
│   │       ├── base_page.rb
│   │       ├── login_page.rb
│   │       └── dashboard_page.rb
│   ├── step_definitions/
│   │   ├── common_steps.rb
│   │   ├── login_steps.rb
│   │   └── checkout_steps.rb
│   └── data/                             # test data fixture'ları
│       └── users.yml
└── reports/
    ├── allure-results/
    └── cucumber.html
```

**Kritik dosyalar:**
- **`features/support/env.rb`** — En önce yüklenir. Capybara/Appium config burada.
- **`features/support/hooks.rb`** — Before, After, AfterStep hook'ları.
- **`features/step_definitions/*.rb`** — Gherkin step'lerinin Ruby implementasyonu.

### Cucumber çalıştırma
```bash
cucumber                                        # tüm feature'lar
cucumber features/login.feature                 # belirli feature
cucumber features/login.feature:15              # belirli line (scenario)
cucumber --tags @smoke                          # tag filtre
cucumber --tags "@smoke and not @wip"           # bileşik filtre
cucumber --profile smoke                        # cucumber.yml'den profile
cucumber --format pretty --format html --out reports/cucumber.html
cucumber --dry-run                              # step match'leri kontrol et, koşturma
cucumber --wip                                  # @wip tag'li senaryoları koş
```

---

## 5. Feature ve Scenario

### Anatomy
```gherkin
@auth @web
Feature: Kullanıcı girişi
  Müşterilerimizin sisteme güvenli giriş yapabilmesi için
  e-posta ve şifre tabanlı authentication sağlamalıyız.

  Background:
    Given giriş sayfası açık

  @smoke
  Scenario: Başarılı giriş — geçerli kimlik bilgileri
    When kullanıcı "user@example.com" ve "validPass1!" ile giriş yapar
    Then dashboard sayfasına yönlendirilir
    And karşılama mesajı görüntülenir

  @regression
  Scenario: Başarısız giriş — yanlış şifre
    When kullanıcı "user@example.com" ve "wrongPassword" ile giriş yapar
    Then "Geçersiz e-posta veya şifre" hata mesajı görüntülenir
    And dashboard sayfasına yönlendirilmez
```

### Feature description (önemli ama opsiyonel)
Feature başlığının altına yazılan bağlam açıklaması. **Cucumber çalıştırmaz**, sadece dokümantasyon amaçlıdır. Best practice: **Connextra formatında**:

```gherkin
Feature: Sepete ürün ekleme

  As a alışveriş yapan kullanıcı
  I want to istediğim ürünleri sepete eklemek
  So that ileride toplu olarak satın alabileyim
```

### Rule keyword (Gherkin 6+, Cucumber 4+)
İş kurallarını feature içinde gruplandırır:

```gherkin
Feature: Para çekme

  Rule: Hesap bakiyesi yeterli olmalı
    Example: Bakiye yeterli — para çekilebilir
      Given hesap bakiyesi 1000 TL
      When kullanıcı 500 TL çekmek ister
      Then 500 TL çekilir
      And bakiye 500 TL olur

    Example: Bakiye yetersiz — para çekilemez
      Given hesap bakiyesi 100 TL
      When kullanıcı 500 TL çekmek ister
      Then işlem reddedilir

  Rule: Günlük limit aşılamaz
    Example: Limit içinde
      Given günlük çekim limiti 5000 TL
      And bugün çekilen toplam 2000 TL
      When kullanıcı 1000 TL çekmek ister
      Then işlem onaylanır
```

`Rule` altında `Example` (= `Scenario`) kullanılır. Mantıksal gruplama sağlar.

---

## 6. Step Definitions

Gherkin'deki her `Given/When/Then` cümlesi bir Ruby **step definition**'a karşılık gelir.

### Basit step
```ruby
# features/step_definitions/login_steps.rb

Given("giriş sayfası açık") do
  visit "/login"
end

When("kullanıcı {string} ve {string} ile giriş yapar") do |email, password|
  fill_in "Email",    with: email
  fill_in "Password", with: password
  click_button "Giriş"
end

Then("dashboard sayfasına yönlendirilir") do
  expect(page).to have_current_path("/dashboard")
end
```

### Step matcher tipleri

#### 1. Cucumber Expressions (önerilen)
Daha okunabilir, type-safe:
```ruby
Given("giriş sayfası açık")
When("kullanıcı {string} ile giriş yapar") do |email|
When("kullanıcı {int} ürün ekler") do |count|
When("fiyat {float} TL'dir") do |price|
When("tarih {word}") do |word|       # tek kelime
```

**Built-in parameter types:**
| Type | Eşleşen | Convert |
|---|---|---|
| `{string}` | `"..."` veya `'...'` | String |
| `{int}` | tamsayı | Integer |
| `{float}` | ondalıklı | Float |
| `{word}` | boşluksuz kelime | String |
| `{}` | herhangi bir şey | String |

#### 2. Regular Expressions (klasik)
```ruby
Given(/^giriş sayfası açık$/) do
When(/^kullanıcı "([^"]*)" ile giriş yapar$/) do |email|
When(/^(\d+) ürün ekler$/) do |count|
```

Regex daha güçlüdür ama daha az okunabilirdir. **Best practice:** Cucumber Expressions kullan, gerektiğinde regex'e düş.

### Step organization — feature bazlı dosyalar
```
features/step_definitions/
├── common_steps.rb              # ortak step'ler (navigation, login)
├── login_steps.rb               # login-specific
├── checkout_steps.rb            # checkout-specific
├── search_steps.rb
└── api_steps.rb
```

**Lead bakışı:** Tüm step'leri tek dosyada toplama — 500 satırı geçen step file'lar **kod kokusu**. Feature bazlı bölünmüş olmalı.

### Step reuse — call step from step
```ruby
Given("oturum açmış bir kullanıcı") do
  step %{giriş sayfası açık}
  step %{kullanıcı "default@x.com" ve "pass123" ile giriş yapar}
end
```

`step "..."` ile başka bir step çağrılabilir. **Lead uyarısı:** Step reuse'u kötüye kullanmak (deep nested call'lar) hata izlerini bozar — debug zorlaşır. Bunun yerine **Ruby helper method'u** tercih et.

### Step pending / skip
```ruby
Then("complex assertion") do
  pending "Henüz implement edilmedi"          # PENDING — sarı
end

Then("skip this") do
  skip_this_scenario                            # tüm scenario skip
end
```

---

## 7. Background

`Background:` her scenario'dan önce **otomatik koşar**. DRY için ideal — ortak setup tek bir yere alınır.

```gherkin
Feature: Sepet yönetimi

  Background:
    Given giriş yapmış bir kullanıcı
    And sepette aşağıdaki ürünler vardır:
      | ürün     | adet |
      | iPhone   | 1    |
      | Kulaklık | 2    |

  Scenario: Ürün silme
    When kullanıcı "iPhone" ürününü siler
    Then sepette 2 ürün kalır

  Scenario: Toplam fiyat hesabı
    Then sepet toplamı "1100 TL" olarak görüntülenir
```

### Kurallar
- Bir feature'da **sadece bir Background** olur.
- Background **sadece `Given`** içermelidir (When/Then antipattern).
- Background **3-4 satırı geçmemeli** — daha fazlası senaryoyu okunamaz yapar.
- Background **Examples'tan önce** çalışır (Scenario Outline'da).

### Background vs Hook farkı

| | Background | Before hook |
|---|---|---|
| Görünürlük | Feature dosyasında, okunabilir | Ruby kodunda, gizli |
| Skope | Bir feature | Tag/global |
| Amaç | İş context'i | Teknik setup (driver, DB cleanup) |
| Kullanım | "Sepette ürünler var" | "Driver başlat", "DB temizle" |

**Kural:** İş context'i Background'a, teknik setup hook'a.

---

## 8. Scenario Outline & Examples

Aynı scenario'yu **farklı veri setleriyle** koşturmak için. Parametrize edilmiş senaryolar.

```gherkin
Feature: Form validation

  Scenario Outline: Hatalı e-posta validation'ı
    Given kayıt formu açık
    When kullanıcı "<email>" e-postasını girer
    And "Kaydet" butonuna tıklar
    Then "<error>" hata mesajı görüntülenir

    Examples: Yaygın hatalar
      | email                | error                            |
      | invalid              | E-posta formatı geçersizdir      |
      | @nodomain.com        | E-posta formatı geçersizdir      |
      | spaces in@email.com  | E-posta formatı geçersizdir      |
      |                      | E-posta zorunludur               |

    Examples: Uluslararası adresler
      | email                  | error                          |
      | user@example.中国       |                                |
      | user@münchen.de        |                                |
```

### Kurallar
- `<paramName>` placeholder'ları step text'inde olur
- `Examples:` tablosu placeholder'ları doldurur
- Her satır **ayrı bir scenario** olarak koşulur
- Birden fazla `Examples:` bloğu olabilir (gruplama için)

### Outline + Tag
```gherkin
Scenario Outline: Login attempts
  When kullanıcı "<email>" ile giriş yapar
  Then sonuç "<result>" olur

  @valid
  Examples:
    | email          | result   |
    | valid@x.com    | success  |

  @invalid @regression
  Examples:
    | email          | result   |
    | invalid        | error    |
    | wrong@x.com    | error    |
```

Tag'ler **Examples bloğuna** uygulanabilir → o satırlar tag'i alır.

### Lead uyarısı: Outline kötüye kullanımı
Outline **veri varyasyonu** için: aynı akış, farklı input.
**Antipattern:** Tamamen farklı akışları Outline'da toplamak.

```gherkin
# ❌ Antipattern — farklı akışları Outline'a sıkıştırma
Scenario Outline: <action>
  When kullanıcı <action> yapar
  Then sonuç <result> olur

  Examples:
    | action       | result          |
    | giriş        | dashboard       |
    | kayıt        | welcome page    |
    | parola sıfır | reset email     |

# ✅ Doğru — her biri ayrı Scenario
Scenario: Kullanıcı giriş yapar
  When kullanıcı giriş yapar
  Then dashboard sayfası açılır

Scenario: Kullanıcı kayıt olur
  When kullanıcı kayıt olur
  Then welcome page açılır
```

---

## 9. Data Tables

Step'e tablo şeklinde data geçirmek için.

### Hash table (header'lı)
```gherkin
Given aşağıdaki kullanıcılar mevcut:
  | name  | email          | role  |
  | John  | john@x.com     | admin |
  | Jane  | jane@x.com     | user  |
```

```ruby
Given("aşağıdaki kullanıcılar mevcut:") do |table|
  table.hashes.each do |row|
    User.create!(name: row["name"], email: row["email"], role: row["role"])
  end
end
```

`table.hashes` → array of hashes:
```ruby
[
  { "name" => "John", "email" => "john@x.com", "role" => "admin" },
  { "name" => "Jane", "email" => "jane@x.com", "role" => "user"  }
]
```

### Raw table (header'sız)
```gherkin
When aşağıdaki kategorileri ekler:
  | Elektronik |
  | Giyim      |
  | Kitap      |
```

```ruby
When("aşağıdaki kategorileri ekler:") do |table|
  table.raw.each do |row|
    Category.create!(name: row.first)
  end
end
```

`table.raw` → array of arrays:
```ruby
[["Elektronik"], ["Giyim"], ["Kitap"]]
```

### Vertical table (rows)
```gherkin
Given aşağıdaki kullanıcı bilgisi:
  | name   | John       |
  | email  | john@x.com |
  | age    | 30         |
```

```ruby
Given("aşağıdaki kullanıcı bilgisi:") do |table|
  user_data = table.rows_hash         # { "name" => "John", "email" => "...", "age" => "30" }
  User.create!(user_data)
end
```

### Tablo metodları cheat sheet
| Method | Döndürdüğü |
|---|---|
| `table.raw` | Array of arrays |
| `table.hashes` | Array of hashes (header'lı) |
| `table.rows` | Body satırları (header hariç) |
| `table.rows_hash` | İki sütunlu vertical hash |
| `table.headers` | Header'lar |
| `table.transpose` | Tabloyu transpoze et |
| `table.map_column!` | Sütun değerlerini dönüştür |

### Tip dönüşümü
```ruby
Given("aşağıdaki fiyatlar:") do |table|
  table.map_column!("price")    { |p| p.to_f }
  table.map_column!("active")   { |a| a == "true" }

  table.hashes.each do |row|
    # row["price"] artık Float, row["active"] Boolean
  end
end
```

### DataTable comparison (assertion)
```gherkin
Then sepet aşağıdaki ürünleri içermelidir:
  | name      | quantity | price |
  | iPhone    | 1        | 1000  |
  | Kulaklık  | 2        | 50    |
```

```ruby
Then("sepet aşağıdaki ürünleri içermelidir:") do |expected_table|
  actual_table = cart.items.map { |i| [i.name, i.quantity.to_s, i.price.to_s] }
  expected_table.diff!(actual_table)
end
```

`diff!` farklı ise hata fırlatır, **highlighted diff** gösterir.

---

## 10. Doc Strings

Çok satırlı text geçirmek için. `"""` ile sarılır.

```gherkin
When kullanıcı aşağıdaki mesajı gönderir:
  """
  Merhaba,

  Bu çok satırlı bir mesajdır.
  Birden fazla paragraf içerir.

  Saygılarımla.
  """
Then mesaj başarıyla iletilir
```

```ruby
When("kullanıcı aşağıdaki mesajı gönderir:") do |docstring|
  fill_in "Message", with: docstring
  click_button "Send"
end
```

### Content type ile doc string
```gherkin
When kullanıcı aşağıdaki JSON'u gönderir:
  """json
  {
    "name": "John",
    "age": 30
  }
  """
```

Content type (örn. `json`) doc string'i annotate eder — Cucumber'a değil dokümantasyona; ama IDE'ler syntax highlighting verir.

---

## 11. Tags ve Filtering

### Tag uygulama
```gherkin
@smoke @auth
Feature: Login

  @critical
  Scenario: Doğru giriş
    ...

  @wip @flaky
  Scenario: Edge case test
    ...
```

Tag'ler **feature** veya **scenario** seviyesinde uygulanır. Feature tag'i tüm scenario'lara inherit edilir.

### Filtering
```bash
# Tek tag
cucumber --tags @smoke

# AND
cucumber --tags "@smoke and @critical"

# OR
cucumber --tags "@smoke or @regression"

# NOT
cucumber --tags "not @wip"

# Karmaşık
cucumber --tags "(@smoke or @regression) and not @flaky"
```

### Yaygın tag convention'ları
| Tag | Anlam |
|---|---|
| `@smoke` | Hızlı, kritik path; her PR'da koşar |
| `@regression` | Tam kapsam; nightly |
| `@critical` | Production-impact; release-blocker |
| `@wip` | Work-in-progress; CI'da skip |
| `@flaky` | Bilinen flaky; quarantine |
| `@skip` | Geçici devre dışı |
| `@manual` | Manuel test, otomatik koşulmaz |
| `@web`, `@mobile` | Platform filtreleme |
| `@api` | API testleri |
| `@slow` | Yavaş test (>1 dk) |

### Tag-based hook
```ruby
Before("@web") do
  Capybara.current_driver = :selenium_chrome_headless
end

Before("@mobile") do
  AppiumSession.start
end

After("@flaky") do |scenario|
  # Flaky scenario için ekstra screenshot
end
```

### `@wip` özel davranışı
```bash
cucumber --wip                 # sadece @wip senaryoları koş
cucumber --tags "not @wip"     # @wip dışındakileri koş
```

`cucumber.yml` profile ile WIP CI'da default skip edilebilir.

---

## 12. Hooks — Before, After, Around, AfterStep

`features/support/hooks.rb` dosyasında.

### Before
Her scenario'dan **önce** çalışır.
```ruby
Before do
  @scenario_started_at = Time.now
end

Before("@web") do
  Capybara.current_driver = :selenium_chrome_headless
end

Before("@mobile") do
  AppiumSession.start
end

Before("@requires_login") do
  step "kullanıcı sisteme giriş yapmış"
end
```

### After
Her scenario'dan **sonra** çalışır. **Test başarısız olsa da koşar** (cleanup için ideal).

```ruby
After do |scenario|
  if scenario.failed?
    save_screenshot
    save_page_source
  end
  Capybara.reset_sessions!
end

After("@cleanup_data") do
  TestData.cleanup_users
end
```

`scenario` objesi:
- `scenario.name` — adı
- `scenario.status` — `:passed`, `:failed`, `:undefined`, `:pending`, `:skipped`
- `scenario.failed?` — boolean
- `scenario.exception` — failure exception (varsa)
- `scenario.source_tag_names` — tag'leri
- `scenario.location` — dosya:satır

### AfterStep
Her **step'ten sonra** koşar. Debug için yararlı, production'da nadiren kullanılır.

```ruby
AfterStep do
  sleep 0.5    # ❌ Sadece debug için; CI'da KESINLIKLE kullanma
end

AfterStep do |scenario|
  puts "Step ran. Current URL: #{current_url}"
end
```

### Around
Scenario'nun tamamını **sarmalar** (block-based).

```ruby
Around("@slow") do |scenario, block|
  Timeout.timeout(300) do
    block.call
  end
end

Around do |scenario, block|
  start = Time.now
  block.call
  puts "Scenario duration: #{Time.now - start} sec"
end
```

### BeforeAll / AfterAll
Tüm test suite'in başında/sonunda **bir kere** çalışır.

```ruby
# features/support/env.rb

# Cucumber 5+ syntax
BeforeAll do
  TestDatabase.seed!
  Appium::Core.for(caps).start_driver
end

AfterAll do
  TestDatabase.teardown!
  AppiumSession.driver.quit
end
```

**Lead seviyesi uyarı:** `BeforeAll` test izolasyonunu bozma riski taşır. Sadece **gerçekten kez-bir** kurulması gereken şeyler için (örn. mock server start).

### Hook ordering
Birden fazla `Before` hook'u varsa **tanımlanma sırasına** göre koşar. Tersini istemek için `prepend_before`:

```ruby
Before { puts "1" }
Before { puts "2" }
prepend_before { puts "0" }     # ilk koşar

# Çıktı: 0, 1, 2
```

`After` hook'ları **ters sırada** koşar (LIFO):
```ruby
After { puts "1" }
After { puts "2" }

# Çıktı: 2, 1   ← son tanımlanan ilk koşar (stack mantığı)
```

### Failure screenshot (en yaygın kullanım)
```ruby
After do |scenario|
  if scenario.failed?
    timestamp = Time.now.strftime("%Y%m%d_%H%M%S")
    safe_name = scenario.name.gsub(/[^a-z0-9]/i, "_")
    path = "reports/screenshots/#{timestamp}_#{safe_name}.png"

    page.save_screenshot(path)                      # Capybara
    # AppiumSession.driver.save_screenshot(path)    # Appium

    attach(path, "image/png")                        # Allure / Cucumber raporuna ekle
  end
end
```

`attach(file_path, mime_type)` Cucumber 7+ — failure screenshot'ı doğrudan rapora gömer.

---

## 13. World ve Helper'lar

**`World`** = step definition'ların koştuğu **scope**. `self`'in ne olduğunu belirler. Her scenario için **yeni bir World instance** oluşturulur — izolasyon sağlanır.

### Module include — en yaygın pattern
```ruby
# features/support/world_helpers.rb
module NavigationHelpers
  def go_to_login
    visit "/login"
  end

  def login_as(email:, password:)
    go_to_login
    fill_in "Email",    with: email
    fill_in "Password", with: password
    click_button "Giriş"
  end
end

World(NavigationHelpers)   # her step definition'da bu method'lar kullanılabilir
```

```ruby
# step_definitions/auth_steps.rb
Given("kullanıcı giriş yapmış") do
  login_as(email: "default@x.com", password: "pass123")
  # ↑ NavigationHelpers'tan gelen method
end
```

### Birden fazla module
```ruby
World(NavigationHelpers, ApiHelpers, AssertionHelpers)
```

### Custom World class
```ruby
class CustomWorld
  include Capybara::DSL
  include RSpec::Matchers

  def base_url
    ENV.fetch("BASE_URL", "http://localhost:3000")
  end
end

World { CustomWorld.new }
```

### Instance variables
Step'ler arası state taşımak için. **Sadece scenario içinde** persist eder (her scenario yeni World).

```ruby
Given("bir kullanıcı oluşturulmuş") do
  @user = User.create!(email: "test@x.com")
end

When("kullanıcı giriş yapar") do
  fill_in "Email", with: @user.email     # ← önceki step'ten gelen state
  fill_in "Password", with: "pass123"
  click_button "Giriş"
end
```

**Lead uyarısı:** Aşırı instance var kullanımı → hidden coupling. Bunun yerine **Page Object'lere state taşı** veya **explicit method parametrelerine** dön.

---

## 14. Parameter Types — Custom Transforms

Built-in `{string}`, `{int}`, `{float}` yetmediğinde **custom parameter type** tanımla.

### Tanımlama
```ruby
# features/support/parameter_types.rb

ParameterType(
  name:         "user",
  regexp:       /[A-Za-z_]+/,
  type:         User,
  transformer:  ->(name) { User.find_by!(username: name) }
)

ParameterType(
  name:         "currency",
  regexp:       /\d+(?:\.\d+)?\s?(?:TL|USD|EUR)/,
  type:         Money,
  transformer:  ->(s) { Money.parse(s) }
)
```

### Kullanım
```gherkin
Scenario: Para transferi
  Given {user} ve {user} sisteme kayıtlı
  When "john" {currency} parayı "jane" kullanıcısına gönderir
  Then "jane" kullanıcısının bakiyesi {currency} olur
```

```ruby
When("{string} {currency} parayı {string} kullanıcısına gönderir") do |sender, amount, receiver|
  # amount artık Money objesi (transformer çalıştı)
  sender_user = User.find_by!(username: sender)
  sender_user.transfer(amount, to: receiver)
end
```

### Built-in ile birlikte
Custom parameter, built-in'leri **override edebilir**. Örnek: `{int}` her zaman Integer'a çevirir; `{positive_int}` tanımlayıp sadece pozitifi kabul edebilirsin.

---

## 15. Profiles — cucumber.yml

`cucumber.yml` dosyasında farklı koşum konfigürasyonları tanımlanır.

```yaml
# cucumber.yml
default: --format pretty --strict --no-source

smoke:    --tags @smoke --format progress
regression: --tags "not @wip and not @manual" --format pretty
ci:       --format pretty --format html --out reports/cucumber.html --format junit --out reports/junit
wip:      --tags @wip --wip
parallel: --format progress
```

```bash
cucumber                          # default profile
cucumber --profile smoke
cucumber --profile ci
cucumber -p regression
```

### Profile inheritance
```yaml
default: --format pretty --strict
nightly: <%= "--profile default" %> --tags "@regression"
```

### ENV-based profile
```yaml
default: <%= ENV["STAGE"] == "ci" ? "--format junit --out reports/" : "--format pretty" %>
```

---

## 16. Cucumber + Capybara Entegrasyonu (Web)

### `features/support/env.rb`
```ruby
require "capybara/cucumber"            # Capybara DSL'i Cucumber World'üne enjekte eder
require "rspec/expectations"
require "selenium-webdriver"

Capybara.configure do |config|
  config.default_driver           = :rack_test
  config.javascript_driver        = :selenium_chrome_headless
  config.default_max_wait_time    = 5
  config.app_host                 = ENV.fetch("BASE_URL", "http://localhost:3000")
  config.save_path                = "reports/capybara"
end

Capybara.register_driver :selenium_chrome_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless=new")
  options.add_argument("--window-size=1400,1400")
  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end
```

### Page Object + Cucumber step
```ruby
# features/support/pages/login_page.rb
class LoginPage
  include Capybara::DSL

  URL = "/login".freeze

  def visit_page
    visit URL
    self
  end

  def login_as(email:, password:)
    fill_in "Email",    with: email
    fill_in "Password", with: password
    click_button "Giriş"
    DashboardPage.new
  end

  def error_message
    find(".alert-danger").text
  end
end
```

```ruby
# features/step_definitions/auth_steps.rb
Given("giriş sayfası açık") do
  @login_page = LoginPage.new.visit_page
end

When("kullanıcı {string} ve {string} ile giriş yapar") do |email, password|
  @dashboard = @login_page.login_as(email: email, password: password)
end

Then("dashboard sayfasına yönlendirilir") do
  expect(page).to have_current_path("/dashboard")
end

Then("{string} hata mesajı görüntülenir") do |expected_error|
  expect(@login_page.error_message).to eq(expected_error)
end
```

### Driver swap via tag
```ruby
# features/support/hooks.rb
Before("@javascript") { Capybara.current_driver = Capybara.javascript_driver }
Before("@rack_test")  { Capybara.current_driver = :rack_test }
After                  { Capybara.use_default_driver }
```

### Failure screenshot — Capybara
```ruby
After do |scenario|
  if scenario.failed?
    name = scenario.name.gsub(/\W+/, "_")
    path = "reports/screenshots/#{Time.now.strftime('%Y%m%d_%H%M%S')}_#{name}.png"
    page.save_screenshot(path)
    attach(path, "image/png")
  end
end
```

### Capybara matcher tuzakları (Cucumber'da da geçerli)
```ruby
# ❌ Race condition
expect(page).not_to have_content("Loading...")

# ✅ Auto-wait'li doğru
expect(page).to have_no_content("Loading...")
```

Bu kural Cucumber step'lerinde de aynen geçerlidir.

---

## 17. Cucumber + Appium Entegrasyonu (Mobile)

### `features/support/env.rb`
```ruby
require "appium_lib_core"
require "rspec/expectations"
require_relative "appium_session"

PLATFORM = ENV.fetch("PLATFORM", "android").to_sym
```

### `features/support/appium_session.rb`
```ruby
require "appium_lib_core"

class AppiumSession
  CAPS = {
    android: {
      caps: {
        platformName:               "Android",
        "appium:platformVersion":    "13",
        "appium:deviceName":         "Pixel_6_API_33",
        "appium:app":                File.expand_path("apps/app-debug.apk"),
        "appium:automationName":     "UiAutomator2",
        "appium:autoGrantPermissions": true
      },
      appium_lib: { server_url: "http://127.0.0.1:4723" }
    },
    ios: {
      caps: {
        platformName:               "iOS",
        "appium:platformVersion":    "17.0",
        "appium:deviceName":         "iPhone 15",
        "appium:app":                File.expand_path("apps/MyApp.app"),
        "appium:automationName":     "XCUITest"
      },
      appium_lib: { server_url: "http://127.0.0.1:4723" }
    }
  }.freeze

  def self.driver
    Thread.current[:appium_driver] ||= Appium::Core.for(CAPS[PLATFORM]).start_driver
  end

  def self.quit
    Thread.current[:appium_driver]&.quit
    Thread.current[:appium_driver] = nil
  end
end

module AppiumWorld
  def driver
    AppiumSession.driver
  end

  def wait
    @wait ||= Selenium::WebDriver::Wait.new(timeout: 10, interval: 0.5)
  end
end

World(AppiumWorld)
```

### `features/support/hooks.rb`
```ruby
Before("@mobile") do
  AppiumSession.driver        # session'ı başlat
end

After("@mobile") do |scenario|
  if scenario.failed?
    name = scenario.name.gsub(/\W+/, "_")
    path = "reports/screenshots/#{Time.now.strftime('%Y%m%d_%H%M%S')}_#{name}.png"
    AppiumSession.driver.save_screenshot(path)
    attach(path, "image/png")
  end
end

AfterAll do
  AppiumSession.quit
end
```

### Mobile Page Object
```ruby
# features/support/pages/mobile/login_page.rb
class Mobile::LoginPage
  include AppiumWorld

  EMAIL_FIELD    = [:accessibility_id, "email_input"].freeze
  PASSWORD_FIELD = [:accessibility_id, "password_input"].freeze
  LOGIN_BTN      = [:accessibility_id, "login_button"].freeze
  ERROR_MSG      = [:accessibility_id, "error_msg"].freeze

  def enter_email(email)
    wait.until { driver.find_element(*EMAIL_FIELD).displayed? }
    driver.find_element(*EMAIL_FIELD).send_keys(email)
    self
  end

  def enter_password(password)
    driver.find_element(*PASSWORD_FIELD).send_keys(password)
    self
  end

  def submit
    driver.find_element(*LOGIN_BTN).click
    Mobile::DashboardPage.new
  end

  def error_message
    wait.until { driver.find_element(*ERROR_MSG).displayed? }
    driver.find_element(*ERROR_MSG).text
  end
end
```

### Step definitions — mobile
```ruby
# features/step_definitions/mobile_login_steps.rb

Given("mobil uygulama açık") do
  AppiumSession.driver.activate_app("com.example.app")
end

When("kullanıcı {string} ve {string} ile mobilde giriş yapar") do |email, password|
  @dashboard = Mobile::LoginPage.new
                                .enter_email(email)
                                .enter_password(password)
                                .submit
end

Then("mobilde dashboard ekranı görünür") do
  expect(@dashboard).to be_displayed
end
```

### Feature dosyası — mobil
```gherkin
@mobile @auth
Feature: Mobil giriş

  @smoke
  Scenario: Başarılı giriş
    Given mobil uygulama açık
    When kullanıcı "user@example.com" ve "pass123" ile mobilde giriş yapar
    Then mobilde dashboard ekranı görünür
```

---

## 18. Reporting

### Built-in formatter'lar
| Formatter | Kullanım |
|---|---|
| `pretty` | Renkli console (default) |
| `progress` | Nokta-tabanlı kısa çıktı (CI için) |
| `html` | HTML rapor |
| `json` | JSON çıktı (custom tooling için) |
| `junit` | JUnit XML (Jenkins entegrasyonu) |
| `rerun` | Failed scenario'ları dosyaya yaz, sonra rerun için |
| `summary` | Sadece özet |
| `usage` | Step kullanım analizi (dead step bulma) |

```bash
cucumber --format pretty
cucumber --format html --out reports/cucumber.html
cucumber --format json --out reports/cucumber.json
cucumber --format junit --out reports/junit/
cucumber --format pretty --format html --out reports/cucumber.html   # birden fazla
cucumber --format rerun --out reports/rerun.txt                       # failed scenarios
cucumber @reports/rerun.txt                                            # sadece failed'leri rerun
```

### Allure entegrasyonu
```ruby
# Gemfile
gem "allure-cucumber"

# features/support/env.rb
require "allure-cucumber"

AllureCucumber.configure do |c|
  c.results_directory = "reports/allure-results"
  c.clean_results_directory = true
  c.environment_properties = {
    platform: ENV.fetch("PLATFORM", "web"),
    base_url: ENV.fetch("BASE_URL", "n/a")
  }
end
```

```bash
cucumber --format AllureCucumber::CucumberFormatter --out reports/allure-results
allure generate reports/allure-results --clean -o reports/allure-html
allure open reports/allure-html
```

### Screenshot attachment
```ruby
After do |scenario|
  if scenario.failed?
    png = page.save_screenshot("reports/#{scenario.name}.png")    # Capybara
    attach(png, "image/png")
  end
end
```

`attach(file_or_data, mime_type)` Cucumber'ın yapısı — hem HTML report'a hem Allure'a embed edilir.

---

## 19. Parallel Test Execution

Cucumber-Ruby native paralel desteği yok; **`parallel_tests`** veya **`parallel_cucumber`** gem'i kullanılır.

### `parallel_tests` ile
```ruby
# Gemfile
gem "parallel_tests"
```

```bash
# 4 paralel process
bundle exec parallel_cucumber features/ -n 4

# Tag filtre ile
bundle exec parallel_cucumber features/ -n 4 -o "--tags @regression"

# Profile ile
bundle exec parallel_cucumber features/ -n 4 -o "--profile ci"
```

### Process isolation gereklilikleri
Paralel koşum için **her process izole** olmalı:

```ruby
# features/support/env.rb
# Database
if defined?(ActiveRecord)
  worker_num = ENV["TEST_ENV_NUMBER"].to_i   # parallel_tests verir: "", "2", "3", ...
  db_name = "test_app_test#{worker_num.zero? ? '' : "_#{worker_num}"}"
  ActiveRecord::Base.establish_connection(database: db_name)
end

# Capybara — port collision'ı engelle
Capybara.server_port = 0                     # rastgele port

# Appium — her process kendi Appium server'ı
worker = ENV["TEST_ENV_NUMBER"].to_i
appium_port = 4723 + (worker.zero? ? 0 : worker - 1)
# AppiumSession capability'lerinde server_url override
```

### Cucumber-Cloud / SaaS alternatifleri
- **Knapsack Pro** → CI node'lar arası **dynamic split** (optimal balancing)
- **CircleCI test splitting** → built-in time-based split
- **BuildKite parallelism**

---

## 20. Best Practices — Senaryo Yazımı

### 1. Declarative > Imperative
**Senaryolar `ne yapılacak` üzerine olmalı, `nasıl yapılacak` üzerine değil.**

```gherkin
# ❌ Imperative — UI detayına gömülmüş, kırılgan
Scenario: Login
  Given URL "/login" açık
  When email field'ına "user@x.com" yaz
  And password field'ına "pass" yaz
  And "submit-btn" id'li butona tıkla
  Then URL "/dashboard" olur
  And "h1" tag'i "Hoş geldin" içerir

# ✅ Declarative — iş niyeti net
Scenario: Başarılı giriş
  When kullanıcı geçerli kimlik bilgileri ile giriş yapar
  Then dashboard sayfasına yönlendirilir
  And karşılama mesajı görüntülenir
```

UI detayları step definition'a kapatılır. Tasarım değişince senaryolar kırılmaz.

### 2. One scenario = one behavior
Bir scenario **bir** iş davranışını test etmeli. AND'leri zorlayarak birleştirme.

```gherkin
# ❌ İki ayrı davranış tek senaryoya sıkıştırılmış
Scenario: Login ve profile güncelleme
  When kullanıcı giriş yapar
  Then dashboard açılır
  When kullanıcı profile sayfasını açar
  Then profil güncellenir

# ✅ İki ayrı scenario
Scenario: Başarılı giriş
  When ...
  Then ...

Scenario: Profile güncelleme
  Given kullanıcı giriş yapmış
  When ...
  Then ...
```

### 3. Given/When/Then mantığı bozulmamalı
- **Given** = state (önkoşul)
- **When** = action (aksiyon)
- **Then** = outcome (beklenti)

Then'den sonra When gelmesin. Then'lerin sıralı olması okunabilirliği artırır.

### 4. UI element ismi kullanma
```gherkin
# ❌
When "submit-btn" id'li butona tıklar
When ".alert-danger" class'lı element gözükür

# ✅
When kullanıcı "Giriş" butonuna tıklar
When hata mesajı görüntülenir
```

### 5. Test data scenario'da görünür olmalı
```gherkin
# ❌ Magic data — okuyucu ne olduğunu bilmez
Given test verileri yüklenir
When kullanıcı sipariş oluşturur

# ✅ Veriler okunabilir
Given aşağıdaki ürünler kataloğa eklenir:
  | name   | price |
  | iPhone | 1000  |
When kullanıcı "iPhone" siparişi oluşturur
```

### 6. Scenario isimlendirme
```gherkin
# ❌ Müphem
Scenario: Test 1
Scenario: Login işlemi

# ✅ Sonuç odaklı
Scenario: Geçerli kimlik bilgileri ile giriş başarılı olur
Scenario: Yanlış şifre girildiğinde hata mesajı gösterilir
```

### 7. Tag'leri tutarlı kullan
Team içinde tag convention'ı belge halinde tut. `@smoke`, `@regression`, `@flaky` ne demek herkes bilmeli.

### 8. Background'ı 3-4 satırı geçmesin
Aşırı uzun background = scenario'lar context'i unutuyor demektir.

### 9. Step'leri kısa tut
Bir step bir cümle olmalı, sub-clauses olmadan.

```gherkin
# ❌
When kullanıcı giriş yaptıktan sonra dashboard'a yönlendirilir ve menüden profili seçer

# ✅
Given kullanıcı dashboard'da
When profil menüsünü açar
```

### 10. Reuse'a güven, kopyala-yapıştır yapma
Aynı step'i birden fazla yerde tanımlamak Cucumber'ın `Ambiguous` exception'ına neden olur. Common step'leri ayır.

---

## 21. Sık Karşılaşılan Pitfall'lar

### 1. Cucumber'ı RSpec yerine kullanmak
RSpec ile ne kadar test yazılırdı, Cucumber'da onun **2-3 katı** kod ortaya çıkar (step layer overhead). Eğer non-tech paydaş senaryoyu okumuyorsa, Cucumber **gereksiz**dir.

**Karar kriteri:** Product Owner Gherkin'i **okuyor mu** ve hatta **yazıyor mu**? Hayır → RSpec yeterli.

### 2. Imperative step'ler
Yukarıda görüldü. UI detayları step'te kapatılmalı.

### 3. Step bloat
500+ satırlık step file'lar. Feature bazlı bölünmedi.

### 4. Ambiguous step
İki step definition aynı text'e match ederse Cucumber `Ambiguous match` fırlatır:
```ruby
When("kullanıcı tıklar") { ... }
When(/^kullanıcı .+ tıklar$/) { ... }    # ikisi de match eder
```

### 5. Slow steps + sleep
```ruby
# ❌
When("user clicks login") do
  click_button "Login"
  sleep 3
end

# ✅
When("user clicks login") do
  click_button "Login"
  expect(page).to have_content("Welcome")    # Capybara auto-wait
end
```

### 6. World instance variable kirliliği
Önceki scenario'dan kalan `@user`, sonraki scenario'da unutulup kullanılırsa false-positive doğar.
**Çözüm:** Her scenario yeni World instance alır — bunu doğru anla ve `@user`'ı sadece bir scenario içinde kullan.

### 7. Background'ı When/Then ile doldurmak
Background **sadece Given** olmalı. When/Then antipattern.

### 8. Scenario Outline'da çok fazla sütun
```gherkin
Examples:
  | a | b | c | d | e | f | g | h | i | j | k |
```
Okunamaz. 4-5 sütundan fazla → ayrı senaryolar yaz.

### 9. Tag spam
```gherkin
@a @b @c @d @e @smoke @regression @auth @web
Feature: ...
```
Anlam taşımayan tag'ler. **Her tag bir filtreleme amacına** hizmet etmeli.

### 10. Failure'da debug yapamamak
Step layer hatayı **gizler**. Stacktrace step file'larından geçer, gerçek hata kaybolur.
**Çözüm:**
- `--backtrace` flag'i
- Step'i kısa tut
- `binding.pry` ile debug

### 11. Cucumber sadece end-to-end için
Aslında **API testleri**, **integration testler** Cucumber'da yazılabilir. Sadece UI testi için değil. Ama:

### 12. Test piramidini ihmal
Cucumber UI test'i pahalıdır. Test'lerin %80'i unit/API'de olmalı; %20'si Cucumber+UI.

---

## 22. Framework Tasarımı

### Kapsamlı klasör yapısı
```
qa-framework/
├── Gemfile
├── Rakefile
├── cucumber.yml
├── .rspec
├── config/
│   ├── environments/
│   │   ├── local.yml
│   │   ├── staging.yml
│   │   └── production.yml
│   └── capybara_drivers.rb
├── features/
│   ├── login.feature
│   ├── checkout.feature
│   ├── support/
│   │   ├── env.rb                       # global setup
│   │   ├── hooks.rb                     # Before/After
│   │   ├── world_helpers.rb             # World modules
│   │   ├── parameter_types.rb           # custom param types
│   │   ├── pages/                       # Page Object'ler
│   │   │   ├── base_page.rb
│   │   │   ├── web/
│   │   │   │   ├── login_page.rb
│   │   │   │   └── dashboard_page.rb
│   │   │   └── mobile/
│   │   │       ├── login_page.rb
│   │   │       └── dashboard_page.rb
│   │   ├── components/
│   │   │   ├── header.rb
│   │   │   └── sidebar.rb
│   │   ├── helpers/
│   │   │   ├── api_helpers.rb
│   │   │   ├── data_helpers.rb
│   │   │   └── wait_helpers.rb
│   │   └── drivers/
│   │       ├── capybara_config.rb
│   │       └── appium_session.rb
│   └── step_definitions/
│       ├── common_steps.rb
│       ├── auth_steps.rb
│       └── checkout_steps.rb
├── data/
│   ├── users.yml
│   └── products.yml
├── apps/                                 # mobile app builds
│   ├── app-debug.apk
│   └── MyApp.app
└── reports/
    ├── screenshots/
    ├── cucumber.html
    └── allure-results/
```

### Multi-platform support — tek framework, hem web hem mobile
```ruby
# features/support/env.rb
PLATFORM = ENV.fetch("PLATFORM", "web").to_sym

case PLATFORM
when :web
  require "capybara/cucumber"
  require_relative "drivers/capybara_config"
when :android, :ios
  require_relative "drivers/appium_session"
end
```

```gherkin
# Aynı feature, farklı platform'da koşar
@web @mobile
Feature: Giriş

  Scenario: Başarılı giriş
    ...
```

```bash
PLATFORM=web cucumber --tags @web
PLATFORM=android cucumber --tags @mobile
```

### Configuration loader
```ruby
# features/support/config.rb
require "yaml"

class Config
  ENV_NAME = ENV.fetch("STAGE", "staging")
  DATA     = YAML.load_file("config/environments/#{ENV_NAME}.yml").freeze

  def self.base_url
    DATA.fetch("base_url")
  end

  def self.timeout
    DATA.fetch("timeout", 10)
  end
end
```

```bash
STAGE=staging cucumber
STAGE=production cucumber --tags @smoke
```

### CI/CD entegrasyonu — GitHub Actions
```yaml
name: Cucumber Tests
on: [push, pull_request]

jobs:
  web-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: browser-actions/setup-chrome@v1

      - name: Run smoke tests
        env:
          STAGE: staging
          BASE_URL: ${{ secrets.STAGING_URL }}
          PLATFORM: web
        run: bundle exec cucumber --profile ci --tags @smoke

      - name: Upload artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: cucumber-report
          path: |
            reports/cucumber.html
            reports/screenshots/
            reports/allure-results/

  mobile-test:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
      - uses: actions/setup-node@v4

      - name: Install Appium
        run: |
          npm install -g appium
          appium driver install uiautomator2

      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          script: |
            appium server --port 4723 &
            PLATFORM=android bundle exec cucumber --tags @mobile
```

---

## 23. QA Lead Seviyesi — Mimari Kararlar

### Cucumber'a değer mi? — Lead'in temel kararı
Cucumber'a **değer**:
- Product Owner, BA, business stakeholder senaryoları **gerçekten okuyor**
- Sözleşme/regulasyon senaryoları (audit trail)
- Multi-team feature collaboration (3 amigos session'ları)
- Acceptance test sayısı az ama kritik (e-ticaret checkout, ödeme)

Cucumber'a **değmez**:
- Sadece QA team senaryoları yazıyor, kimse okumuyor
- Pure regression test (RSpec yeterli)
- API testleri için (RSpec + Airborne daha pratik)
- High-frequency, hızlı testler (overhead fazla)

### Test piramidi pozisyonu
```
        /\
       /  \    %10  E2E (Cucumber + Capybara/Appium)
      /----\
     /      \  %20  Integration (RSpec + API + DB)
    /--------\
   /          \ %70  Unit (RSpec)
  /____________\
```

Cucumber, **piramidin en üstü**dür. Acceptance criteria'yı doğrular — mimari kararlarını test etmez.

### Step definition ownership
Büyük team'lerde:
- **Common steps** (login, navigation) → core QA team owns
- **Feature-specific steps** → ilgili feature team owns
- **CODEOWNERS** ile zorla

### Living documentation
Cucumber'ın **en büyük katma değeri**: feature dosyaları **canlı dokümantasyon**.
- Stakeholder Gherkin'den iş kuralını okur
- Code review'da Gherkin'i Product Owner inceler
- "Bu feature ne yapıyor?" sorusunun cevabı `.feature` dosyasında

Lead bu kültürü kurmalı:
- Product Owner her sprint feature'ları **gözden geçirir**
- Acceptance criteria → Gherkin scenario
- DoD (Definition of Done) bir Gherkin scenario yazılmış olmasını içerir

### Maintenance stratejisi
- **Step audit:** 3 ayda bir `cucumber --format usage` ile dead step'leri bul, sil
- **Flake rate** %1 altında olmalı; üstündekiler quarantine (`@flaky`)
- **Test execution time:** > 1 dk olan scenario'lar (`@slow`) ayrı suite
- **Locator audit:** Page Object'lerde data-testid kullanım oranı

### Team scaling
- **Onboarding doc:** Junior tester 1 hafta içinde ilk Cucumber PR'ını açabilmeli
- **Pairing:** Senior + junior çift Gherkin yazımı
- **Convention enforcement:** `rubocop-cucumber` (varsa) veya custom linter
- **Tag glossary:** `docs/tags.md` — tüm tag'lerin anlamı
- **3 amigos session'ları:** her sprint başında PO + Dev + QA Gherkin review

### Risk yönetimi
- **Cucumber upgrade:** Major version (8→9) breaking change'leri olabilir; staged rollout
- **Ruby version:** Cucumber ile Ruby version uyumluluğu (8+ → Ruby 2.7+)
- **Capybara/Appium upgrade:** Cucumber katmanı stable kalır, alttaki driver upgrade'leri PR-test edilmeli

---

## 24. En Sık Mülakat Soruları

| Soru | Hızlı Cevap |
|---|---|
| Cucumber nedir? | BDD framework'ü; Gherkin DSL'i + Ruby step definition |
| BDD vs TDD farkı? | BDD: iş paydaşı odaklı, behavior; TDD: developer odaklı, technical |
| Gherkin nedir? | Doğal dil benzeri DSL; Feature/Scenario/Given/When/Then |
| Given/When/Then mantığı? | Given: önkoşul/state; When: aksiyon; Then: beklenti/outcome |
| Background nedir? | Tüm scenario'lardan önce çalışan ortak Given'lar |
| Scenario Outline? | Parametrize edilmiş scenario, Examples tablosu ile çalışır |
| Examples'da kaç sütun ideal? | 4-5; daha fazlası okunabilirliği bozar |
| Data Table tipleri? | hashes (header'lı), raw (header'sız), rows_hash (vertical) |
| Doc String? | `"""` ile çok satırlı text (mesaj, JSON, vb.) |
| Step matcher tipleri? | Cucumber Expressions (`{string}`, `{int}`); Regex |
| Hook tipleri? | Before, After, AfterStep, Around, BeforeAll, AfterAll |
| After hook ordering? | LIFO — son tanımlanan ilk koşar |
| Tag amacı? | Filtreleme (`--tags @smoke`) + hook tetikleme |
| `@wip` özel mi? | `--wip` flag'i ile sadece wip'ler veya `not @wip` exclude |
| Custom parameter type? | `ParameterType` ile regex + transformer tanımla |
| World nedir? | Step definition'ların koştuğu scope; her scenario için yeni instance |
| Instance var scope? | Bir scenario içinde persist; sonraki scenario'da reset |
| Profile (cucumber.yml)? | Konfigürasyon preset'leri (`--profile ci`) |
| Capybara entegrasyon? | `require "capybara/cucumber"` → DSL World'e enjekte |
| Appium entegrasyon? | `appium_lib_core` + custom World module + Thread.current[:driver] |
| Driver swap (tag-based)? | `Before("@js") { Capybara.current_driver = :selenium }` |
| Failure screenshot? | `After { \|s\| attach(page.save_screenshot, "image/png") if s.failed? }` |
| Reporting formatter'lar? | pretty, progress, html, json, junit, rerun, summary, usage |
| Allure entegrasyonu? | `allure-cucumber` gem; AllureCucumber formatter |
| Parallel test? | `parallel_tests` gem + per-process DB/port izolasyonu |
| Declarative vs Imperative? | Declarative iş niyeti; imperative UI detayı (kötü pratik) |
| One scenario = one behavior? | Evet; AND'lerle iki davranış birleştirme |
| Background'da When? | Antipattern — sadece Given olmalı |
| Step reuse — `step "..."`? | Mümkün ama debug zorlaşır; Ruby helper tercih |
| Ambiguous step? | İki step aynı text'e match → exception |
| Cucumber ne zaman değmez? | Non-tech paydaş okumuyorsa overhead; RSpec yeterli |
| Test piramidinde yer? | En üst — %10; acceptance kriteri |
| Living documentation? | `.feature` dosyaları iş kuralını canlı belge olarak tutar |
| Dead step nasıl bulunur? | `cucumber --format usage` |
| `--dry-run` ne işe yarar? | Step match'leri kontrol et, koşturma |
| `--rerun` formatter? | Failed scenario'ları dosyaya yaz, rerun için |
| Rule keyword? | Gherkin 6+; iş kurallarını feature içinde gruplandırır |
| Türkçe Gherkin? | İlk satıra `# language: tr`; Özellik/Senaryo/Diyelim ki |
| BeforeAll antipattern? | Test izolasyonunu bozma riski; sadece gerçekten kez-bir |
| Cucumber vs RSpec? | Cucumber: iş paydaşı; RSpec: developer odaklı |
| Lead'in ana kararı? | Product Owner gerçekten okuyor mu? Hayır → RSpec yeterli |

---

## 25. Çalışma Yol Haritası

### 1. Hafta — Gherkin ve temel Cucumber
- [ ] Cucumber install + `cucumber --init`
- [ ] **Gherkin syntax** — Feature, Scenario, Given/When/Then
- [ ] Background, And/But
- [ ] İlk feature dosyası + step definitions
- [ ] `cucumber` komutu, flag'ler (`--tags`, `--dry-run`, `--format`)

### 2. Hafta — Orta seviye Gherkin
- [ ] **Scenario Outline + Examples**
- [ ] **Data Tables** (hashes, raw, rows_hash)
- [ ] **Doc Strings**
- [ ] **Tags** ve filtreleme (`@smoke`, `@regression`, `not @wip`)
- [ ] **Cucumber Expressions** (`{string}`, `{int}`, `{float}`)
- [ ] **Custom parameter types**

### 3. Hafta — Hooks, World, Framework
- [ ] **Hooks** — Before, After, AfterStep, Around, BeforeAll
- [ ] **World** ve helper modülleri
- [ ] **Profile** (cucumber.yml) — smoke, regression, ci
- [ ] **Page Object Model** + Cucumber step entegrasyonu
- [ ] Failure screenshot hook
- [ ] **Reporting** — HTML, JUnit, Allure

### 4. Hafta — Entegrasyon ve Lead seviyesi
- [ ] **Capybara + Cucumber** entegrasyonu (web)
- [ ] **Appium + Cucumber** entegrasyonu (mobile)
- [ ] Driver swap tag-based
- [ ] **Parallel test** (`parallel_tests`)
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] **Living documentation** kültürü — 3 amigos
- [ ] Dead step audit (`--format usage`)
- [ ] Code review checklist + onboarding doc

---

## 26. Kaynaklar

### Resmi
- **Ana sayfa:** cucumber.io
- **Cucumber-Ruby GitHub:** github.com/cucumber/cucumber-ruby
- **Docs:** cucumber.io/docs/cucumber/
- **Gherkin reference:** cucumber.io/docs/gherkin/reference/

### Ruby ekosistemi
- **`cucumber-rails`:** github.com/cucumber/cucumber-rails
- **`allure-cucumber`:** github.com/allure-framework/allure-ruby
- **`parallel_tests`:** github.com/grosser/parallel_tests
- **`site_prism`:** github.com/site-prism/site_prism (Page Object DSL)

### Capybara + Cucumber
- **Capybara docs:** github.com/teamcapybara/capybara
- **`capybara/cucumber`** require'ı — DSL otomatik enjekte edilir

### Appium + Cucumber
- **`appium_lib_core`:** github.com/appium/ruby_lib_core
- **`appium_lib`:** github.com/appium/ruby_lib

### Kitaplar
- **"The Cucumber Book"** — Matt Wynne, Aslak Hellesøy (kurucular)
- **"Specification by Example"** — Gojko Adzic (BDD felsefesi)
- **"BDD in Action"** — John Ferguson Smart

### BDD felsefesi
- **Dan North — "Introducing BDD"** (orijinal blog post)
- **3 amigos** — atlassian.com/agile/project-management/three-amigos

### Topluluk
- **community.cucumber.io** — forum
- **Stack Overflow tag:** `cucumber-ruby`
- **Cucumber Slack:** cucumberbdd.slack.com
