# Capybara — QA Lead Mülakat Hazırlık Rehberi (Ruby)

> Ruby ekosisteminde **web UI / acceptance test'lerin standart DSL'i**. Gerçek kullanıcı davranışını simüle eder, built-in auto-wait mekanizması ile flaky test'leri azaltır, driver-agnostic mimarisi ile JS gerektirmeyen hızlı test'ten gerçek tarayıcı test'ine kadar geniş bir yelpazeyi kapsar.

---

## İçindekiler

1. [Capybara Nedir? Felsefe](#1-capybara-nedir-felsefe)
2. [Mimari — Capybara Stack'i](#2-mimari--capybara-stack-i)
3. [Temel Terimler — Sözlük](#3-temel-terimler--sozluk)
4. [Kurulum](#4-kurulum)
5. [Proje Yapısı](#5-proje-yapisi)
6. [İlk Test — Hello World](#6-ilk-test--hello-world)
7. [Driver'lar Derinlemesine](#7-driver-lar-derinlemesine)
8. [Konfigürasyon — Tüm Options](#8-konfigurasyon--tum-options)
9. [Navigation API](#9-navigation-api)
10. [Click ve Action Family](#10-click-ve-action-family)
11. [Form Interaction](#11-form-interaction)
12. [Element Finding — `find`, `all`, `first`](#12-element-finding)
13. [Match Strategies — Ambiguity Resolution](#13-match-strategies)
14. [Element / Node API](#14-element--node-api)
15. [Scoping — `within`, `within_frame`, `within_window`](#15-scoping)
16. [Matcher'lar — `have_*` Family](#16-matcher-lar)
17. [Match Options — `wait`, `exact`, `count`, `visible`](#17-match-options)
18. [Auto-Wait & `synchronize` Derinlemesine](#18-auto-wait--synchronize)
19. [Locator Stratejileri — Built-in Selector Types](#19-locator-stratejileri)
20. [Custom Selectors, Filters, Expressions](#20-custom-selectors)
21. [JavaScript Interaction](#21-javascript-interaction)
22. [Sessions ve Windows](#22-sessions-ve-windows)
23. [Server & External App Testing](#23-server--external-app)
24. [Cookies, Headers, Storage](#24-cookies-headers-storage)
25. [File Upload / Download](#25-file-upload--download)
26. [Page Object Model — POM](#26-page-object-model)
27. [Cross-Browser & Cloud Testing](#27-cross-browser--cloud)
28. [Test Data Management](#28-test-data-management)
29. [Cucumber & Minitest Entegrasyonu](#29-cucumber--minitest)
30. [Reporting](#30-reporting)
31. [Debugging Araçları](#31-debugging-araclari)
32. [Flaky Test Stratejisi](#32-flaky-test-stratejisi)
33. [Visual & Accessibility Testing](#33-visual--accessibility)
34. [Performans Optimizasyonu](#34-performans-optimizasyonu)
35. [Sık Karşılaşılan Pitfall'lar](#35-pitfall-lar)
36. [Framework Tasarımı](#36-framework-tasarimi)
37. [QA Lead Seviyesi — Mimari Kararlar](#37-mimari-kararlar)
38. [CI/CD Entegrasyonu](#38-cicd-entegrasyonu)
39. [En Sık Mülakat Soruları](#39-mulakat-sorulari)
40. [Çalışma Yol Haritası](#40-calisma-yol-haritasi)
41. [Kaynaklar](#41-kaynaklar)

---

## 1. Capybara Nedir? Felsefe

**Capybara**, Ruby ile yazılmış, **acceptance / integration / end-to-end** test'ler için tasarlanmış bir **DSL (Domain Specific Language)**'dir. Gerçek bir kullanıcının web uygulamasıyla nasıl etkileşim kuracağını simüle eder.

### Tarihçe
- **2009:** Jonas Nicklas tarafından oluşturuldu
- **2012:** Capybara 2.x — multi-session, modern API
- **2017:** Capybara 3.x — modern Ruby (2.4+), Selenium 3 entegrasyonu
- **2019+:** Capybara 3.30+ — built-in `test_id` selector, ARIA support

### Felsefe — Üç temel prensip

**1. Behavior-driven, intent-revealing API**
```ruby
# ❌ Selenium-style — implementation detail
driver.find_element(By.id("submit-btn")).click

# ✅ Capybara — user intent
click_button "Submit"
```

**2. Auto-wait first — flaky test'lerin %80'ini ortadan kaldırır**
Asenkron olarak yüklenen elementleri otomatik bekler, `sleep` ihtiyacını ortadan kaldırır.

**3. Driver-agnostic — aynı test, farklı driver'da koşar**
Sunucu-side render için `rack_test` (anlık), JS-heavy SPA için `cuprite` veya `selenium`. Aynı DSL.

### Capybara'nın yeri — test piramidinde nereye düşer?

```
        /\
       /  \    %10  E2E / System (Capybara + browser)
      /----\
     /      \  %20  Integration / Request specs
    /--------\
   /          \ %70  Unit (Model, Service, Helper)
  /____________\
```

**Capybara piramidin tepesidir:** en pahalı, en kırılgan, en yavaş test. **Bu yüzden sadece critical user journey'leri** için kullanılmalı — her detayı E2E ile test etmek anti-pattern.

### Kullanılan framework'ler
| Test framework | Capybara entegrasyonu |
|---|---|
| **RSpec** | `capybara/rspec` (en yaygın) |
| **Cucumber** | `capybara/cucumber` (BDD) |
| **Minitest** | `capybara/minitest` |
| **Test::Unit** | `capybara/dsl` manuel include |

### Capybara vs alternatifler
| | Capybara | RSpec system spec | Watir | Playwright-Ruby |
|---|---|---|---|---|
| Dil | Ruby | Ruby | Ruby | Ruby (binding) |
| Auto-wait | ✅ Built-in | ⚠️ RSpec, manuel | ⚠️ Manuel | ✅ Native |
| Driver çeşitliliği | ✅ Çok | Capybara üzerine | Selenium | Playwright runtime |
| Rails entegrasyonu | ✅ Standart | ✅ Standart | Manuel | Yeni, gelişiyor |
| Kullanım | Rails, Sinatra | Aynı | Custom | Modern projeler |

**Lead seviyesi karar:** Rails projesi → **Capybara**. Modern, non-Rails → **Playwright-Ruby** değerlendirilebilir.

---

## 2. Mimari — Capybara Stack'i

Capybara çağrısı yaptığında ne olur?

```
┌────────────────────────────────────────────────────────────┐
│   Test Code (RSpec / Cucumber / Minitest)                  │
│       visit "/login"; fill_in "Email", with: "x@y.com"     │
│                            ↓                                │
├────────────────────────────────────────────────────────────┤
│   Capybara DSL Layer                                        │
│   - Capybara::DSL module                                    │
│   - Capybara::Session (= `page`)                            │
│   - Capybara::Node::* (Element / Document / Simple)         │
│                            ↓                                │
├────────────────────────────────────────────────────────────┤
│   Driver Abstraction (Capybara::Driver::Base)               │
│   - Rack::Test / Selenium / Cuprite / Custom                │
│                            ↓                                │
├────────────────────────────────────────────────────────────┤
│   ┌──────────────────────┐    ┌──────────────────────────┐ │
│   │ Rack::Test driver    │    │ Selenium / Cuprite       │ │
│   │ - In-process Rack    │    │ - HTTP / CDP              │ │
│   │ - No JS, no browser  │    │ - Real browser            │ │
│   │ - Çok hızlı          │    │ - JS execution            │ │
│   └─────────┬────────────┘    └────────────┬─────────────┘ │
│             ↓                              ↓                │
├────────────────────────────────────────────────────────────┤
│   ┌──────────────────────┐    ┌──────────────────────────┐ │
│   │ Capybara.app         │    │ Capybara Server          │ │
│   │ (Rack app obj)       │    │ (Puma — your app)        │ │
│   └──────────────────────┘    └────────────┬─────────────┘ │
│                                            ↓                │
│                               ┌──────────────────────────┐ │
│                               │ Browser (Chrome / FF)    │ │
│                               └──────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### Driver'a göre davranış farkı

**`rack_test` driver:**
1. Test kodun `visit "/login"` çağırır.
2. Capybara::RackTest::Driver, `Capybara.app` (Rack app objesi, örn. Rails app) üzerine **doğrudan HTTP request** simüle eder.
3. Network yok, browser yok, JS yok. **Çok hızlı.**

**Selenium / Cuprite driver:**
1. Test başlangıcında Capybara **Puma server'ını fork eder** (`Capybara.run_server = true`).
2. Server, `Capybara.app` (Rails app) için rastgele bir portta `localhost:XXXX` üzerinde HTTP serve eder.
3. `visit "/login"` → driver browser'ı `http://localhost:XXXX/login`'e yönlendirir.
4. Browser request gönderir → Puma server → Rails app → response.
5. JS de browser'da execute olur.

### Neden bu mimari önemli?

**Mülakat sorusu:** *"Rack::Test ile Selenium arasındaki temel mimari fark nedir?"*

**Cevap:** Rack::Test in-process HTTP simülasyonudur — JS yok, network yok, **gerçek browser yok**. Selenium gerçek bir tarayıcıyı driver protocol (W3C WebDriver) üzerinden uzaktan sürer, Capybara da bunun için bir **HTTP server'ı fork eder** (`Capybara.server`).

---

## 3. Temel Terimler — Sözlük

QA Lead seviyesinde bu terimler kesin oturmalı.

### `Capybara::Session` (= global `page` objesi)
**Browser session handle'ıdır.** Mevcut URL, cookie'ler, açık window'lar, response — hepsi burada. `visit`, `click_*`, `find` gibi tüm DSL method'ları arka planda `page` üzerinde çalışır.

```ruby
page                  # current session
page.current_url
page.current_path
page.html             # current document HTML
page.title
```

> **Page Object Model ile karıştırılmamalı.** `page` = framework session; POM = senin yazdığın `LoginPage` gibi sınıflar.

### `Capybara::Node::Element`
**`find`'ın döndürdüğü tek bir DOM elementi.** Üzerinde `click`, `text`, `value`, `[attr]` gibi metodlar var.

```ruby
button = find("#submit")          # Capybara::Node::Element
button.click
button.text
button[:type]                     # attribute access
button.value
button.visible?
```

### `Capybara::Node::Document`
Mevcut sayfanın **root node'u**. `page` aslında session'dır ama içindeki `document` da bir Node'dur. Genelde direkt kullanılmaz.

### `Capybara::Node::Simple`
**Statik HTML parse etmek için.** Capybara'nın `string` selector'larını runtime'dan bağımsız HTML üzerinde test etmek için.

```ruby
node = Capybara.string("<div><a href='/foo'>Click</a></div>")
node.find("a")[:href]   # => "/foo"
```

### `Capybara::Result`
`all` method'unun döndürdüğü **Array-like wrapper**. Filter ve lazy evaluation destekler.

```ruby
items = all(".item")              # Capybara::Result
items.size
items.first
items.map(&:text)
items.filter { |el| el.text.include?("foo") }
```

### `Capybara::Window`
**Açık tarayıcı sekmesi/penceresi.** Multi-tab senaryolarda yönetilir.

```ruby
page.windows                      # tüm açık window'lar
page.current_window
new = window_opened_by { click_link "Open" }
new.close
```

### `Capybara::Driver::*`
**Driver abstraction layer.** Driver = "browser ile nasıl konuşurum" stratejisi.

| Class | Driver |
|---|---|
| `Capybara::RackTest::Driver` | Rack-Test |
| `Capybara::Selenium::Driver` | Selenium WebDriver |
| `Capybara::Cuprite::Driver` | (3rd party) Cuprite |

### `Capybara.app`
**Test edilen Rack application object'i.** Genelde Rails app'i:
```ruby
Capybara.app = MyApp.application    # Rails ise
Capybara.app = Rack::Builder.new { run MyRackApp.new }
```

### `Capybara.server`
**Embedded HTTP server'ı.** Default Puma. Selenium/Cuprite driver kullanıldığında `Capybara.app`'i bir portta serve etmek için.

### Selector
**DOM'da bir element nasıl bulunur sorusunun strateji ismi.** Built-in: `:css`, `:xpath`, `:id`, `:button`, `:link`, `:field`, vb. (Detay: bölüm 19)

### Matcher
**Assertion için RSpec/Minitest entegrasyonu.** `have_content`, `have_selector`, `have_no_content`, vb. (Bölüm 16)

### Auto-wait / Synchronize
**Capybara matcher'larının kendi içlerinde polling yapma davranışı.** (Bölüm 18)

### `Capybara::DSL`
**Test'lerinde `visit`, `click_button` gibi method'ları global olarak kullanmanı sağlayan module.**
```ruby
include Capybara::DSL              # her method `page` üstüne delegate olur
```

---

## 4. Kurulum

### Saf Capybara (Sinatra, Rack app)
```ruby
# Gemfile
group :test do
  gem "capybara"
  gem "rspec"
  gem "rack-test"               # Rack::Test driver için
  gem "selenium-webdriver"      # JS-heavy testler için
  gem "webdrivers"              # ChromeDriver/GeckoDriver auto-download
end
```

### Capybara + Rails (en yaygın)
```ruby
# Gemfile
group :test do
  gem "capybara", "~> 3.40"
  gem "selenium-webdriver"
  gem "webdrivers"
  gem "cuprite"                 # Selenium yerine modern hızlı alternatif
  gem "database_cleaner-active_record"
  gem "factory_bot_rails"
  gem "rspec-rails"
end
```

```bash
rails generate rspec:install
```

### Capybara + Cucumber
```ruby
gem "capybara"
gem "cucumber"
gem "selenium-webdriver"
```

`features/support/env.rb`:
```ruby
require "capybara/cucumber"
```

### Capybara + Minitest
```ruby
gem "capybara"
gem "minitest"
```

```ruby
# test/test_helper.rb
require "capybara/minitest"

class ApplicationSystemTestCase < Minitest::Test
  include Capybara::DSL
  include Capybara::Minitest::Assertions

  def teardown
    Capybara.reset_sessions!
  end
end
```

### Version uyumluluk matrix'i
| Capybara | Ruby | Rails | Selenium |
|---|---|---|---|
| 3.40+ | 2.7+ | 6+ | 4+ |
| 3.36-3.39 | 2.6+ | 5+ | 4 |
| 3.x (eski) | 2.4+ | 5 | 3 |

---

## 5. Proje Yapısı

### Rails + RSpec klasör yapısı
```
.
├── Gemfile
├── spec/
│   ├── rails_helper.rb              # Rails-aware spec helper
│   ├── spec_helper.rb               # Saf RSpec helper
│   ├── system/                      # ← Capybara test'leri (BURASI)
│   │   ├── authentication_spec.rb
│   │   ├── checkout_spec.rb
│   │   └── dashboard_spec.rb
│   ├── support/
│   │   ├── capybara.rb              # Capybara configuration
│   │   ├── drivers.rb               # Custom driver registration
│   │   ├── pages/                   # Page Object'ler
│   │   │   ├── base_page.rb
│   │   │   ├── login_page.rb
│   │   │   ├── dashboard_page.rb
│   │   │   └── components/
│   │   │       ├── header.rb
│   │   │       └── footer.rb
│   │   ├── helpers/
│   │   │   ├── authentication_helper.rb
│   │   │   ├── wait_helpers.rb
│   │   │   └── screenshot_helper.rb
│   │   └── factories/               # FactoryBot fixtures
│   │       └── users.rb
│   ├── factories/                   # FactoryBot (alternatif konum)
│   └── fixtures/
│       └── files/
│           ├── avatar.png
│           └── document.pdf
├── tmp/
│   ├── capybara/                    # screenshot/HTML save_path
│   └── screenshots/
└── config/
    └── application.rb
```

### Non-Rails (saf RSpec + Capybara)
```
.
├── Gemfile
├── lib/
│   └── my_app.rb                    # Rack app (test edilen)
├── spec/
│   ├── spec_helper.rb
│   ├── support/
│   │   ├── capybara.rb
│   │   └── pages/
│   └── system/
│       └── home_page_spec.rb
└── config.ru
```

### Cucumber yapısı (kısaca)
```
.
├── features/
│   ├── login.feature
│   ├── step_definitions/
│   │   └── login_steps.rb
│   └── support/
│       ├── env.rb                   # require "capybara/cucumber"
│       └── pages/
```

---

## 6. İlk Test — Hello World

### `spec/rails_helper.rb` (Rails RSpec entry)
```ruby
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rspec/rails"
require "capybara/rspec"

Dir[Rails.root.join("spec/support/**/*.rb")].sort.each { |f| require f }

RSpec.configure do |config|
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
  config.use_transactional_fixtures = true
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!
end
```

### `spec/support/capybara.rb`
```ruby
require "capybara/rspec"
require "selenium-webdriver"

Capybara.configure do |config|
  config.default_driver           = :rack_test
  config.javascript_driver        = :selenium_chrome_headless
  config.default_max_wait_time    = 5
  config.server                   = :puma, { Silent: true }
  config.default_normalize_ws     = true       # whitespace normalize
  config.save_path                = Rails.root.join("tmp/capybara")
  config.disable_animation        = true       # CSS animations skip
  config.test_id                  = "data-testid"
end

Capybara.register_driver :selenium_chrome_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless=new")
  options.add_argument("--no-sandbox")
  options.add_argument("--disable-gpu")
  options.add_argument("--window-size=1400,1400")
  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

RSpec.configure do |c|
  c.include Capybara::DSL, type: :system
end
```

### İlk test — `spec/system/login_spec.rb`
```ruby
require "rails_helper"

RSpec.describe "Authentication", type: :system do
  let!(:user) { create(:user, email: "user@example.com", password: "secret123") }

  context "doğru kimlik bilgileriyle giriş" do
    it "kullanıcıyı dashboard'a yönlendirir" do
      visit "/login"

      fill_in "Email",    with: "user@example.com"
      fill_in "Password", with: "secret123"
      click_button "Sign in"

      expect(page).to have_current_path("/dashboard")
      expect(page).to have_content("Welcome back, #{user.name}")
    end
  end

  context "yanlış şifreyle giriş", :js do
    it "hata mesajı gösterir ve giriş sayfasında kalır" do
      visit "/login"

      fill_in "Email",    with: "user@example.com"
      fill_in "Password", with: "wrong-password"
      click_button "Sign in"

      expect(page).to have_content("Invalid credentials")
      expect(page).to have_current_path("/login")
    end
  end
end
```

**Notlar:**
- `:js` metadata → JavaScript driver kullanılır (config'de tanımlandığı için `:selenium_chrome_headless`).
- `type: :system` → RSpec'in `system spec` tipi, Capybara::DSL otomatik dahil.

### Çalıştırma
```bash
bundle exec rspec spec/system/login_spec.rb
bundle exec rspec spec/system/login_spec.rb:8     # belirli line
bundle exec rspec --tag js                         # sadece JS test'leri
bundle exec rspec --tag ~js                        # JS olmayanlar
```

---

## 7. Driver'lar Derinlemesine

Capybara'nın en güçlü yanı **driver-agnostic** mimari. Aynı test, aynı kod, farklı driver'da koşar.

### Driver karşılaştırması

| Driver | Server | JS | Hız | Real browser | Tipik kullanım |
|---|---|---|---|---|---|
| **`:rack_test`** | In-process Rack | ❌ | ⚡⚡⚡⚡ | ❌ | Server-side render, controller test |
| **`:selenium`** | Capybara Puma | ✅ | 🐢 | ✅ Chrome/Firefox/Edge/Safari | Full browser test, debug |
| **`:selenium_chrome_headless`** | Capybara Puma | ✅ | 🐢🐢 | ✅ Headless Chrome | CI default |
| **`:selenium_firefox`** | Capybara Puma | ✅ | 🐢 | ✅ Firefox | Cross-browser |
| **`:cuprite`** | Capybara Puma | ✅ | ⚡⚡ | ✅ Chrome (CDP) | Modern hızlı seçenek |
| **`:apparition`** | Capybara Puma | ✅ | ⚡⚡ | ✅ Chromium | Deprecated, kullanma |

### `:rack_test` — Default, Hızlı, JS yok

**Avantajları:**
- **Çok hızlı** — gerçek tarayıcı yok, in-process HTTP
- **Headless dependency'siz** — Chrome/Firefox yüklü olmasına gerek yok
- Session/cookie/header doğrudan manipüle edilebilir

**Sınırları:**
- **JavaScript yok** — JS koşmaz, AJAX, React, Vue test edilemez
- **CSS render yok** — visibility/animation test edilemez
- File upload simülasyonu sınırlı (gerçek file picker yok)

**Setup:**
```ruby
Capybara.default_driver = :rack_test
```

**Kullanım örneği:**
```ruby
it "displays product list", driver: :rack_test do
  visit "/products"
  expect(page).to have_content("iPhone")
end
```

**Yetenekler:**
```ruby
# Cookie manuel set
page.driver.browser.set_cookie("session_id=abc123")

# Header set
page.driver.header("Authorization", "Bearer xyz")

# Custom request
page.driver.submit :post, "/api/login", { email: "x@y.com" }
```

### `:selenium_chrome_headless` — CI Default

**En yaygın CI seçimi.** Headless Chrome ile gerçek browser behavior + headless = ekran yok.

```ruby
Capybara.register_driver :selenium_chrome_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new

  # Headless
  options.add_argument("--headless=new")           # Chrome 109+ new headless
  options.add_argument("--no-sandbox")             # Linux Docker için
  options.add_argument("--disable-gpu")
  options.add_argument("--disable-dev-shm-usage")  # Docker memory fix

  # Window
  options.add_argument("--window-size=1400,1400")

  # Performance
  options.add_argument("--disable-extensions")
  options.add_argument("--disable-popup-blocking")

  # Logging (debug için)
  options.add_argument("--enable-logging")
  options.add_argument("--v=1")

  # User agent
  options.add_argument("--user-agent=MyTestBot/1.0")

  Capybara::Selenium::Driver.new(
    app,
    browser: :chrome,
    options: options
  )
end
```

### `:selenium_chrome` (görsel debug için)

Headless **değil** — Chrome ekranda açılır. Lokal debug, demo, görsel kontrol için.

```ruby
Capybara.register_driver :selenium_chrome do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--window-size=1400,1400")
  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

# Sadece bu test'i görerek koştur:
it "manual debug", driver: :selenium_chrome do
  visit "/login"
  binding.pry          # bu noktada browser açık, manuel kontrol edebilirsin
end
```

### `:cuprite` — Modern hızlı CDP

**Chrome DevTools Protocol (CDP)** üzerinden doğrudan Chrome'u sürer. Selenium'a göre daha hızlı, daha kararlı.

```ruby
# Gemfile
gem "cuprite"
```

```ruby
# spec/support/capybara.rb
require "capybara/cuprite"

Capybara.register_driver :cuprite do |app|
  Capybara::Cuprite::Driver.new(
    app,
    window_size:     [1400, 1400],
    headless:        true,
    timeout:         15,
    process_timeout: 15,
    js_errors:       true,             # JS error → exception
    inspector:       ENV["INSPECTOR"], # `INSPECTOR=true rspec` ile DevTools açık
    browser_options: { "no-sandbox": nil }
  )
end

Capybara.javascript_driver = :cuprite
```

**Cuprite avantajları:**
- Selenium'a göre **2-3x daha hızlı**
- Network intercept (request/response manipulation)
- JS error otomatik exception fırlatır
- DevTools entegrasyonu

**Dezavantajları:**
- Sadece Chrome/Chromium
- Cross-browser test gerekirse Selenium gerekli

### `:apparition` (deprecated, kullanma)

Cuprite'ın atası. **Artık maintain edilmiyor.** Yeni projeler için **Cuprite**.

### Custom driver registration
```ruby
Capybara.register_driver :my_custom do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--lang=tr-TR")              # Turkish locale
  options.add_argument("--accept-lang=tr-TR")

  options.add_preference("download.default_directory", "/tmp/downloads")
  options.add_preference("download.prompt_for_download", false)

  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

Capybara.javascript_driver = :my_custom
```

### Driver dinamik değişimi
```ruby
# Bir test için
it "ihtiyaca göre seçim", js: true do
  # otomatik javascript_driver
end

it "explicit driver", driver: :cuprite do
  # cuprite
end

# Bir blok için
Capybara.using_driver(:selenium_chrome) do
  visit "/dashboard"
  # ... burada selenium_chrome
end
# sonra default'a döner

# Manuel
Capybara.current_driver = :selenium_chrome
# ...
Capybara.use_default_driver
```

### Driver seçim karar matrisi

| Senaryo | Önerilen |
|---|---|
| Server-side render, JS yok | `:rack_test` |
| Critical user flow, gerçek browser | `:selenium_chrome_headless` veya `:cuprite` |
| Lokal görsel debug | `:selenium_chrome` (headless değil) |
| Yüksek perf gerekli CI | `:cuprite` |
| Cross-browser (Firefox/Edge/Safari) | Selenium'un her varyantı |
| Cloud test (BrowserStack/LambdaTest) | `:selenium_remote` |

---

## 8. Konfigürasyon — Tüm Options

```ruby
Capybara.configure do |config|
  # ───── Sunucu ─────
  config.server                  = :puma, { Silent: true }
  config.server_host             = "127.0.0.1"
  config.server_port             = 0              # rastgele port (collision-safe)
  config.server_errors           = [Exception]    # server-side error → raise

  # External app için (Capybara server kullanma)
  config.run_server              = true           # false → external server
  config.app_host                = nil            # external URL
  config.always_include_port     = true           # URL'lere port ekle

  # ───── Driver'lar ─────
  config.default_driver          = :rack_test     # JS gerektirmeyen test'ler
  config.javascript_driver       = :selenium_chrome_headless
  config.current_driver          = nil            # runtime override

  # ───── Bekleme (auto-wait) ─────
  config.default_max_wait_time   = 5              # saniye

  # ───── Match strategies ─────
  config.match                   = :smart         # :first, :one, :prefer_exact
  config.exact                   = false          # exact text match
  config.exact_text              = false          # find :button "Save" → exact "Save"
  config.visible_text_only       = true           # find :text görünen text'i

  # ───── Element davranışı ─────
  config.automatic_label_click   = false          # disabled label'a tıklama
  config.enable_aria_label       = true           # aria-label ile label match
  config.enable_aria_role        = false          # ARIA roles ile element bul
  config.ignore_hidden_elements  = true           # visible: :visible default
  config.default_normalize_ws    = false          # whitespace normalize matcher'larda

  # ───── Test ID ─────
  config.test_id                 = "data-testid"  # built-in data-testid support
  # → find_button "submit" hem text="submit" hem data-testid="submit"

  # ───── Browser davranışı ─────
  config.predicates_wait         = true           # has_*? predicate'leri auto-wait
  config.disable_animation       = false          # CSS animation skip
  config.threadsafe              = false          # multi-threaded test

  # ───── Save/Debug ─────
  config.save_path                  = "tmp/capybara"
  config.save_and_open_page_path    = "tmp/capybara"
  config.save_screenshot_on_failure = false       # capybara-screenshot gem

  # ───── Hata davranışı ─────
  config.raise_server_errors        = true        # server-side exception → raise
  config.reuse_server               = true        # session'lar arası server reuse

  # ───── Default visit options ─────
  config.default_host               = "http://www.example.com"
  config.session_options            = nil
end
```

### En sık değiştirilen ayarlar (öncelik sırası)
1. `default_max_wait_time` — CI'da 5-10 sn
2. `default_driver` — `:rack_test`
3. `javascript_driver` — `:selenium_chrome_headless` veya `:cuprite`
4. `app_host` — external test environment
5. `test_id` — `"data-testid"`

### `Capybara.test_id` — Built-in helper

Capybara 3.20+ ile `data-testid` (veya benzeri) attribute'u **built-in selector** olarak desteklenir.

```ruby
Capybara.test_id = "data-testid"

# Şimdi tüm finder'lar bu attribute'u da arar:
find_button "submit-btn"     # text="submit-btn" VEYA data-testid="submit-btn"
click_on "login-link"
fill_in "email-input", with: "x@y.com"
```

**Lead bakışı:** Bu kullanıldığında **HTML'e gizli bir bağımlılık** doğar — developer'larla anlaşman gerekir: "Test edilebilir her interactive element `data-testid` taşımalı." Bu konsensüs olmadan inconsistency çıkar.

---

## 9. Navigation API

### Visit / URL
```ruby
visit "/login"                          # path
visit users_url(id: 1)                  # Rails URL helper
visit "https://external.com/api"        # absolute URL (app_host override)

page.current_url                        # "http://127.0.0.1:3000/login"
page.current_path                       # "/login"
page.current_host                       # "http://127.0.0.1:3000"
page.current_scheme                     # "http"
page.current_window.size                # [width, height]
```

### History
```ruby
page.go_back
page.go_forward
page.refresh
```

> **Driver constraint:** `go_back`, `go_forward`, `refresh` **Rack::Test'te yoktur** — sadece gerçek browser driver'larda çalışır.

### Status code (sadece Rack::Test)
```ruby
page.status_code                        # 200, 404, ...
page.response_headers["Content-Type"]   # "text/html"
```

> Selenium'da yoktur — gerçek browser status code'u expose etmez.

### Visit options
```ruby
# Capybara'nın visit'i çok basittir, query string vb. manuel yapılır
visit "/search?q=phone&page=2"

# Rails URL helper ile
visit search_path(q: "phone", page: 2)
```

### App host management
```ruby
# Global
Capybara.app_host = "https://staging.example.com"

# Sadece bu blok için
Capybara.using_host("https://api.example.com") do
  visit "/users"          # → https://api.example.com/users
end
```

---

## 10. Click ve Action Family

### `click_button` — sadece button
**Hedef elementler:** `<button>`, `<input type="submit">`, `<input type="button">`, `<input type="image">`, `<input type="reset">`

```ruby
click_button "Submit"                              # text content
click_button "submit-btn"                          # id
click_button "user[save]"                          # name attribute
click_button "Save", disabled: false               # disabled filter
click_button "Save", class: "primary"              # class filter
click_button "Submit", wait: 10                    # auto-wait override
click_button "Save", match: :first                 # ambiguity resolution
```

**Locator olarak çalışan attribute'ler:**
- Text content (button içeriği)
- `id`
- `name`
- `value`
- `title`
- `alt` (image button)
- `data-testid` (`Capybara.test_id` ayarlı ise)

### `click_link` — sadece `<a href>`
```ruby
click_link "Sign in"
click_link "forgot-password"                       # id
click_link "Click here", href: "/dashboard"        # href filter
click_link "Click here", href: %r{/dashboard/\d+}  # regex href
click_link "Click here", title: "Go home"
```

**Locator:** text, id, title, alt.

### `click_on` — Universal (= `click_link_or_button`)
Hem button hem link için. Element type bilinmiyorsa kullan.

```ruby
click_on "Save"
click_on "Forgot password?"
```

### `Node#click` — düşük seviye
**Bulduğun bir element'e** doğrudan click. Locator değil; element üzerinden.

```ruby
find(".custom-clickable").click
find("li.item", text: "First").click
find(".btn", visible: false).click(force: true)    # invisible'e tıkla
```

### Click options (Capybara 3.x+)
```ruby
button.click                       # normal
button.click(:control, :shift)     # modifier'larla
button.right_click                 # sağ click
button.double_click                # çift click
button.send_keys(:enter)           # Enter tuşu
```

### `Node#hover`, `drag_to`, `drop_on`
```ruby
find(".menu-trigger").hover
find(".sub-item").click

source = find(".draggable")
target = find(".drop-zone")
source.drag_to(target)
```

> Drag/drop **Selenium / Cuprite**'da çalışır, Rack::Test'te değil.

### Trigger native events
```ruby
button.trigger("click")             # Cuprite/Selenium
button.trigger("mouseover")
button.send_keys(:tab)
```

### Form submit
```ruby
# Implicit — submit button'a tıkla
click_button "Submit"

# Explicit — form'u JS ile submit et
find("form#login").submit
```

### Click family pitfalls
```ruby
# ❌ Ambiguous — sayfada 5 "Edit" linki varsa Capybara::Ambiguous fırlatır
click_link "Edit"

# ✅ Scope ile
within "#user-123" do
  click_link "Edit"
end

# ✅ veya match: :first
click_link "Edit", match: :first
```

---

## 11. Form Interaction

### `fill_in` — Text input'a değer yaz
```ruby
fill_in "Email",        with: "user@example.com"
fill_in "user_email",   with: "..."                  # id ile
fill_in "Notes",        with: "Multi\nline\ntext"
fill_in "Search",       with: "", clear: true        # önce temizle
fill_in "Notes", with: "x", fill_options: { clear: :backspace }
```

**Çalıştığı input tipleri:**
- `<input type="text|email|password|search|tel|url|number|date|time|color">`
- `<textarea>`
- `contenteditable` element'ler

**Locator çözümleme önceliği:**
1. `<label for="email">Email</label>` → `<input id="email">`
2. `id`
3. `name`
4. `placeholder`
5. `data-testid` (ayarlı ise)

### `find_field(locator).set(value)` — alternatif
Daha düşük seviye, daha esnek:
```ruby
find_field("Email").set("user@example.com")
find("#user_email").set("user@example.com")        # CSS selector ile

# Native value set (JS event tetiklemez!)
find("#email").set("x@y.com")
```

### `choose` — Radio button
```ruby
choose "Male"
choose "user_gender_male"                          # id
choose "gender", option: "male"                    # name + value
```

### `check` / `uncheck` — Checkbox
```ruby
check "I accept terms"
check "user_admin"                                 # id
check "Subscribe", allow_label_click: true

uncheck "Newsletter"
```

`allow_label_click: true` → checkbox label'ına tıklanır (modern UI framework'lerde bazen gerekli).

### `select` / `unselect` — Dropdown
```ruby
select "Turkey",        from: "Country"
select "Turkey",        from: "user_country"        # id
select "Apple",         from: "Fruits", match: :first

# Multi-select
select "Red",   from: "Colors"
select "Blue",  from: "Colors"

unselect "Red", from: "Colors"

# Visible text vs value
find("select#country").find("option[value='TR']").select_option
```

### `attach_file` — File upload
```ruby
attach_file "Avatar", "spec/fixtures/files/avatar.png"

# Multi-file
attach_file "Photos", [
  "spec/fixtures/files/img1.png",
  "spec/fixtures/files/img2.png"
]

# Field locator ile
attach_file "user_avatar", file_path

# Hidden file input (modern drag-drop UI)
attach_file "Avatar", file_path, make_visible: true
```

`make_visible: true` → JS ile `display: block` enjekte eder, drop zone'la sarılmış file input'lara file ekleyebilirsin.

### `find_field` — Field bul, manipüle et
```ruby
email_field = find_field("Email")
email_field.set("x@y.com")
email_field.value                                  # mevcut değeri
email_field.tag_name                               # "input"
```

### Form helper'lar
```ruby
within "form#new_user" do
  fill_in "Name",  with: "John"
  fill_in "Email", with: "john@x.com"
  click_button "Sign up"
end
```

### `submit_form!` (custom helper örnek)
Capybara'da `submit_form!` yoktur, ama custom helper sık yazılır:
```ruby
module FormHelpers
  def submit_form_in(scope, fields = {})
    within(scope) do
      fields.each { |label, value| fill_in label, with: value }
      yield if block_given?
    end
  end
end
```

---

## 12. Element Finding — `find`, `all`, `first`

### `find(selector, **options)`
**Tek bir element bekler.** Auto-wait yapar. Bulamazsa exception.

```ruby
find("#submit")
find(".item", text: "First")
find(:button, "Save")
find("li", text: "Foo", wait: 10)
find(:xpath, "//div[@class='alert']")
```

**Eşleşme yoksa:** `Capybara::ElementNotFound` (default_max_wait_time sonrası).
**Birden fazla eşleşme varsa:** `Capybara::Ambiguous` (default `match: :smart`).

### `all(selector, **options)`
**Eşleşen TÜM element'leri** `Capybara::Result` olarak döner.

```ruby
all(".item")                       # bekleme YOK, anlık snapshot
all(".item", minimum: 3)           # en az 3 element için bekler
all(".item", maximum: 10)
all(".item", count: 5)
all(".item", between: 3..7)
```

**Eşleşme yoksa:** Exception **fırlatmaz**, boş `Result` döner.

**Önemli:** `all` **default'ta auto-wait yapmaz**. `minimum`, `maximum`, `count`, `between` ile bekletilebilir.

### `first(selector, **options)`
**İlk eşleşen element.** Auto-wait yapar.

```ruby
first(".item")                     # = all(".item").first ama auto-wait
first(".item", text: "Foo")
first(".item", visible: :all, wait: 0)
```

### `find_*` aileleri
```ruby
find_button "Save"                 # = find(:button, "Save")
find_link "Logout"                 # = find(:link, "Logout")
find_field "Email"                 # = find(:field, "Email")
find_by_id "user_42"               # = find("#user_42")
```

### Çoklu locator (filter chain)
```ruby
find("li", text: "Apple", visible: true, wait: 5)
all("button.primary", disabled: false)
find(:button, "Save", title: "Save changes")
```

### Comparison tablosu
| Durum | `find` | `all` | `first` |
|---|---|---|---|
| 0 eşleşme | `ElementNotFound` (after wait) | Boş `Result` | `nil` veya `ElementNotFound` |
| 1 eşleşme | Element döner | 1-elementlik `Result` | Element |
| N eşleşme | `Ambiguous` (default) | Tüm element'ler | İlk element |
| Auto-wait | ✅ Var | ❌ Yok | ✅ Var |

### `has_selector?` / `has_no_selector?` predicate'ler
```ruby
page.has_selector?(".item")              # true/false; auto-wait
page.has_selector?(".item", count: 3)
page.has_no_selector?(".error")          # auto-wait'li negatif

# Element üstünde
find(".container").has_css?(".item")
find(".container").has_xpath?(".//li")
```

### Lead seviyesi anti-pattern uyarısı
```ruby
# ❌ Assertion'da all kullanmak — auto-wait yok, flaky
expect(all(".item").size).to eq(3)

# ✅ Matcher kullan — auto-wait var
expect(page).to have_selector(".item", count: 3)
```

---

## 13. Match Strategies — Ambiguity Resolution

`find` birden fazla element bulduğunda ne yapacağını **`Capybara.match`** belirler.

### Match strategy'leri

| Mode | Davranış |
|---|---|
| **`:smart`** (default) | Exact match varsa onu seç; birden fazla exact varsa `Ambiguous`. Yoksa partial; partial'da birden fazla → `Ambiguous`. |
| **`:first`** | İlk eşleşeni döndür, sessizce devam et. |
| **`:one`** | **Tek bir** eşleşme bekler; birden fazla → `Ambiguous` (exact'i tercih etmez). |
| **`:prefer_exact`** | Exact varsa onu; yoksa partial'ın ilkini. **Sessizce devam eder.** |

```ruby
Capybara.match = :smart            # global
find("button", "Save", match: :first)
```

### Pratik örnek
HTML:
```html
<button>Save</button>
<button>Save changes</button>
```

| Match | Sonuç |
|---|---|
| `:smart` | "Save" exact → exact bulunur, döner |
| `:first` | "Save" döner (ilk eşleşen) |
| `:one` | `Ambiguous` (iki partial match) |
| `:prefer_exact` | "Save" döner |

### `exact: true` ile birlikte
```ruby
find :button, "Save", exact: true       # sadece exact text eşleşmesi
find :link, "Edit",   exact: false      # partial match (default)
```

### Lead seviyesi tavsiye
- `:smart` default — bırak.
- Test özelinde override etmek istersen `match:` parametresi.
- **`match: :first`'i kötüye kullanma** — silent failure'a yol açar. Spesifik locator (scope, `:id`, `data-testid`) tercih et.

---

## 14. Element / Node API

`find` veya `all.first` `Capybara::Node::Element` döner. Bu objenin tüm method'ları:

### State sorgulama
```ruby
el.visible?                      # ekranda görünüyor mu
el.disabled?                     # disabled mi
el.checked?                      # checkbox/radio checked mi
el.selected?                     # select option selected mi
el.obscured?                     # başka element üstte mi
el.readonly?                     # readonly mi
el.multiple?                     # multi-select mi
el.path                          # XPath path'i
```

### Değer / içerik
```ruby
el.text                          # innerText
el.text(:all)                    # hidden text dahil
el.text(:visible)                # sadece visible text (default)
el.value                         # input.value
el.tag_name                      # "button", "input", ...
el[:href]                        # attribute access
el[:disabled]
el[:data]                        # data-* attribute'ları
el.style("display")              # computed style (Selenium/Cuprite)
```

### Native API erişimi
```ruby
el.native                        # raw driver-specific element
el.native.click                  # driver-specific call
```

> **Lead uyarısı:** `native` driver-specific'tir; **portability'i bozar**. Driver değiştirirseniz kırılır.

### Action
```ruby
el.click
el.click(:control, :shift)       # modifier'larla
el.right_click
el.double_click
el.hover
el.send_keys("text", :enter)
el.set("value")                  # input.value set
el.select_option                 # <option> elementi seç
el.unselect_option
el.trigger("change")             # JS event tetikle
el.drag_to(target_element)
el.drop_on(target_element)
el.scroll_to(:bottom)            # scroll into view
el.scroll_to(:center)
el.execute_script("arguments[0].click()")
el.evaluate_script("arguments[0].textContent")
```

### Container olarak kullanma
Element kendisi bir context'tir; içinde başka element aranabilir.
```ruby
container = find(".user-card")
container.find(".name").text
container.has_link?("Edit")
container.click_link "Delete"
within(container) { click_button "Save" }
```

### Rect ve location
```ruby
el.rect                          # {x:, y:, width:, height:} (Selenium)
el.location                      # [x, y]
el.size                          # [width, height]
```

### Stale element handling
```ruby
button = find(".btn")
visit "/refresh"
button.click                     # StaleElementReferenceError!

# ✅ Locator'ı sakla, element'i değil
BUTTON_LOCATOR = ".btn".freeze
visit "/refresh"
find(BUTTON_LOCATOR).click
```

### Helper method örneği
```ruby
def safe_click(selector, retries: 3)
  retries.times do |attempt|
    find(selector).click
    return
  rescue Capybara::ElementNotFound, Selenium::WebDriver::Error::StaleElementReferenceError
    raise if attempt == retries - 1
    sleep 0.5
  end
end
```

---

## 15. Scoping — `within`, `within_frame`, `within_window`

### `within(selector)` — DOM içine zoom
Spesifik bir parent altında arama yapmak için.

```ruby
within ".sidebar" do
  click_link "Logout"
end

within "form#new_user" do
  fill_in "Email",    with: "x@y.com"
  fill_in "Password", with: "secret"
  click_button "Sign up"
end

# Selector + filter
within ".user-card", text: "John" do
  click_link "Edit"
end

# Element üstünden
card = find(".user-card", text: "John")
within(card) do
  click_link "Edit"
end
```

**Lead bakışı:** `within` ambiguity'yi engeller, intent'i okunabilir kılar, test'in **niyetini** belli eder.

```ruby
# ❌ Sayfada 10 "Delete" linki var, hangisi?
click_link "Delete"

# ✅ Hangi user için sildiğimiz net
within "#user-#{user.id}" do
  click_link "Delete"
end
```

### `within_frame(selector_or_name)` — iframe içine gir
```ruby
within_frame "payment_iframe" do
  fill_in "Card Number", with: "4111111111111111"
  click_button "Pay"
end

# CSS selector ile
within_frame find("iframe.payment") do
  click_button "Confirm"
end

# index ile
within_frame frame: 0 do
  click_button "OK"
end
```

> **Driver constraint:** Rack::Test iframe desteklemez.

### `within_window(window)` — sekme/pencere içine gir
```ruby
new_window = window_opened_by { click_link "Open in new tab" }

within_window(new_window) do
  expect(page).to have_content("Second page")
  click_link "Close"
end
# Window otomatik kapanır mı? Hayır, manuel:
new_window.close
```

### `within_element` — alias `within`
Aynı şey, isim daha açık.

### Nested scoping
```ruby
within "table#users" do
  within "tr", text: "John" do
    within "td.actions" do
      click_link "Edit"
    end
  end
end
```

**Kural:** 3'ten fazla nested `within` → POM'a çevir. Okunamaz olur.

---

## 16. Matcher'lar — `have_*` Family

RSpec/Minitest assertion matcher'ları. **Auto-wait yapar.**

### Content & text
```ruby
expect(page).to have_content("Hello")
expect(page).to have_content(/Welcome.*/)               # regex
expect(page).to have_text("Hello", wait: 10)
expect(page).to have_text("Hello", normalize_ws: true)
expect(page).to have_text(:visible, "Hello")            # sadece visible
expect(page).to have_text(:all, "Hidden too")           # hidden dahil
expect(page).to have_no_content("Error")                # ✅ doğru negative
```

### Selector / CSS / XPath
```ruby
expect(page).to have_selector(".item")
expect(page).to have_selector(".item", count: 3)
expect(page).to have_selector(".item", minimum: 2)
expect(page).to have_selector(".item", maximum: 10)
expect(page).to have_selector(".item", between: 3..5)
expect(page).to have_selector(".item", text: "Foo")
expect(page).to have_selector(".item", visible: :all)
expect(page).to have_selector(:button, "Save", disabled: false)

expect(page).to have_css(".btn-primary")
expect(page).to have_xpath("//div[@id='foo']")
```

### URL / path
```ruby
expect(page).to have_current_path("/dashboard")
expect(page).to have_current_path("/dashboard", ignore_query: true)
expect(page).to have_current_path(%r{/users/\d+})
expect(page).to have_current_url("http://localhost:3000/dashboard")
```

### Form element matcher'ları
```ruby
expect(page).to have_field("Email")
expect(page).to have_field("Email", with: "x@y.com")          # current value
expect(page).to have_field("Email", type: "email")
expect(page).to have_field("Email", placeholder: "Your email")
expect(page).to have_field("Email", readonly: false)
expect(page).to have_field("Email", disabled: false)

expect(page).to have_button("Submit")
expect(page).to have_button("Submit", disabled: false)
expect(page).to have_button("Submit", type: "submit")

expect(page).to have_link("Sign in")
expect(page).to have_link("Sign in", href: "/login")
expect(page).to have_link("Sign in", href: %r{/login})

expect(page).to have_checked_field("Remember me")
expect(page).to have_unchecked_field("Newsletter")

expect(page).to have_select("Country", selected: "Turkey")
expect(page).to have_select("Country", with_options: ["Turkey", "Greece"])

expect(page).to have_option("Turkey", selected: true)
```

### Tablolar
```ruby
expect(page).to have_table("Users")
expect(page).to have_table("Users", with_rows: [
  ["John", "Admin"],
  ["Jane", "User"]
])
expect(page).to have_table("Users", with_cols: [
  { "Name" => ["John", "Jane"] }
])
```

### Title
```ruby
expect(page).to have_title("Dashboard")
expect(page).to have_title(/Welcome/)
```

### Negative matcher'lar
**Her `have_*`'ın bir `have_no_*` karşılığı var.** Negative assertion için **her zaman bunları** kullan.

```ruby
expect(page).to have_no_content("Error")              # ✅
expect(page).to have_no_selector(".spinner")           # ✅
expect(page).to have_no_button("Save", disabled: true) # ✅
expect(page).to have_no_link("Logout")                 # ✅
expect(page).to have_no_field("Email", with: "old@x.com")
expect(page).to have_no_current_path("/login")
```

**Antipattern:** `expect(page).not_to have_*` — bölüm 18'de açıklanan auto-wait race condition.

---

## 17. Match Options — `wait`, `exact`, `count`, `visible`

Tüm finder ve matcher'lar bu options'ları destekler.

### `wait:` — auto-wait override
```ruby
find(".dashboard", wait: 10)
expect(page).to have_content("Welcome", wait: 8)
expect(page).to have_no_content("Loading", wait: 15)
find("#x", wait: 0)                # bekleme yok, anlık
```

### `exact:` — text match strictness
```ruby
find :button, "Save",   exact: true         # sadece exact "Save"
find :button, "Save",   exact: false        # "Save" partial match (default)
have_content "Hello",   exact: true
```

### `match:` — ambiguity resolution
```ruby
find ".item", match: :first
find ".item", match: :one        # birden fazla → exception
find ".item", match: :smart      # default
find ".item", match: :prefer_exact
```

### `count:`, `minimum:`, `maximum:`, `between:`
```ruby
all(".item", count: 5)
all(".item", minimum: 3)
all(".item", between: 3..7)
have_selector(".item", count: 3)
```

### `visible:` — visibility filter
```ruby
find(".item", visible: true)          # sadece visible (default)
find(".item", visible: false)         # hidden dahil
find(".item", visible: :all)          # = visible: false
find(".item", visible: :visible)      # = visible: true
find(".item", visible: :hidden)       # SADECE hidden
```

`Capybara.ignore_hidden_elements = false` → global default değiştirir.

### `text:` filter — find ile text eşleştir
```ruby
find("li", text: "Apple")
find("li", text: /[Aa]pple/)
find(".item", text: "Foo", exact_text: true)
```

### `class:`, `id:`, `title:`, `name:`, vb.
```ruby
find :button, "Save", class: "primary"
find :button, "Save", id: "save-btn"
find :link, "Click here", title: "Open"
find :field, name: "user[email]"
```

### Custom filter (selector-specific)
```ruby
find :field, "Email", placeholder: "your@email.com"
find :button, disabled: false
find :link, href: %r{/users/\d+}
```

### Options matrix — pratik referans

| Option | Çalıştığı method'lar | Default |
|---|---|---|
| `wait:` | Tüm finder ve matcher | `Capybara.default_max_wait_time` |
| `exact:` | find, have_*, click_button (text match) | `Capybara.exact` (false) |
| `match:` | find | `Capybara.match` (:smart) |
| `count:` | all, have_selector, has_css? | yok |
| `minimum:`, `maximum:`, `between:` | all, have_selector | yok |
| `visible:` | Tüm finder | `Capybara.ignore_hidden_elements` |
| `text:` | find, all, have_selector | nil |

---

## 18. Auto-Wait & `synchronize` Derinlemesine

Capybara'nın **en kritik özelliği**. Mülakat klasiği.

### Çalışma mantığı

Tüm finder ve matcher'lar **`Capybara::Node::Base#synchronize`** bloğu içinde çalışır:

```ruby
# Simplified internal logic
def synchronize(seconds = Capybara.default_max_wait_time)
  start_time = Time.now
  loop do
    begin
      return yield
    rescue Capybara::ElementNotFound, Capybara::Ambiguous => e
      raise e if Time.now - start_time > seconds
      sleep(0.05)
    end
  end
end
```

Yani: **bir element/koşul tatmin edilene kadar her ~50ms'de bir yeniden dene**, max `default_max_wait_time` saniye.

### `default_max_wait_time`
```ruby
Capybara.default_max_wait_time = 5     # default 2 saniye
```

### Override yöntemleri
```ruby
# 1. Tek bir method için
find("#dashboard", wait: 10)
expect(page).to have_content("Welcome", wait: 8)
click_button "Save", wait: 1

# 2. Bir blok için
using_wait_time(15) do
  find(".slow-modal")
  click_on "Confirm"
end

# 3. Bekleme yok (anlık)
find("#x", wait: 0)
all(".item", wait: 0)
```

### Polling vs Sleep — kritik fark
| | Davranış |
|---|---|
| `sleep 5` | **Her zaman** 5 sn bekler |
| `default_max_wait_time = 5` | **En fazla** 5 sn; gelirse hemen devam |

**Tipik kazanım:** Eğer element 200ms'de geldiyse, sleep 4.8 sn boşa beklerken Capybara hemen devam eder. 100 test × 4.8 sn = **8 dakika tasarruf** her koşumda.

### Custom synchronize bloğu
Kendi polling logic'in için:
```ruby
def wait_for_redux_store(key, value, timeout: 5)
  using_wait_time(timeout) do
    page.document.synchronize do
      raise Capybara::ElementNotFound unless
        page.evaluate_script("window.__REDUX_STORE.state.#{key}") == value
    end
  end
end
```

### `have_no_*` vs `not_to have_*` — klasik mülakat trap'i

| | Davranış |
|---|---|
| `expect(page).to have_no_content("X")` | "X kaybolana kadar bekle" — **doğru** |
| `expect(page).not_to have_content("X")` | "X var mı?" matcher polling yapar, **then negate** — **race condition** |

**Neden?** Matcher polling'i **`have_content`'in pozitif çıkmasını** bekler. Eğer X kaybolacaksa, matcher anlık X'i görür → true döner → `not_to` false yapar → **test fail**.

`have_no_content` ise polling'i tersine kurar — "X yok" koşulunu bekler.

**Kural:** Tüm negative assertion'lar `have_no_*` ile yazılmalı.

### Capybara matcher predicate'ler (`has_*?`)
RSpec dışında da auto-wait yararı:
```ruby
page.has_css?(".item")             # auto-wait'li true/false
page.has_no_css?(".error")         # auto-wait'li negative
page.has_button?("Save")
page.has_link?("Home")
```

### Lead seviyesi pitfall'lar
```ruby
# ❌ all + size — auto-wait yok
expect(all(".item").size).to eq 3

# ✅ have_selector + count
expect(page).to have_selector(".item", count: 3)

# ❌ Capybara matcher değil, anlık check
expect(page.current_path).to eq "/dashboard"

# ✅ Capybara matcher, auto-wait
expect(page).to have_current_path("/dashboard")

# ❌ String comparison, no wait
expect(find(".msg").text).to eq "Welcome"

# ✅ Matcher
expect(page).to have_selector(".msg", text: "Welcome")
```

### `synchronize` source code analizi (Lead için)

```ruby
# capybara/lib/capybara/node/base.rb (özetlenmiş)
def synchronize(seconds = nil, errors: nil)
  start_time = Capybara::Helpers.monotonic_time
  seconds ||= Capybara.default_max_wait_time
  catch_error(errors) do
    begin
      yield
    rescue *(errors || [Capybara::ElementNotFound]) => e
      raise e if (Capybara::Helpers.monotonic_time - start_time) >= seconds
      sleep(0.05)
      reload if Capybara::Helpers.monotonic_time - start_time >= seconds
      retry
    end
  end
end
```

**Mülakat sorusu:** *"Capybara'nın auto-wait'i nasıl çalışır?"*
**Cevap:** `synchronize` bloğu içinde method'u **`default_max_wait_time` boyunca her ~50ms'de retry eder**. Method başarılı sonuç döndürürse devam, timeout'ta exception fırlatır.

---

## 19. Locator Stratejileri — Built-in Selector Types

Capybara, CSS/XPath'in ötesinde **iş anlamı taşıyan selector type'ları** sağlar.

### Built-in selector type'ları

| Selector | Bulduğu | Locator |
|---|---|---|
| `:css` | CSS selector | string |
| `:xpath` | XPath expression | string |
| `:id` | id attribute | id |
| `:button` | `<button>`, `<input type=submit/...>` | text, value, id, name, title, alt, data-testid |
| `:link` | `<a href>` | text, id, title, alt, data-testid |
| `:link_or_button` | İkisi de | yukarıdakiler |
| `:field` | Form field (text/email/password/checkbox/radio/select/textarea) | label, id, name, placeholder |
| `:fillable_field` | Text-like field (text/email/password/textarea/...) | label, id, name, placeholder |
| `:radio_button` | `<input type=radio>` | label, id, name |
| `:checkbox` | `<input type=checkbox>` | label, id, name |
| `:select` | `<select>` | label, id, name |
| `:option` | `<option>` | text |
| `:datalist_input` | datalist-bound input | label, id, name |
| `:file_field` | `<input type=file>` | label, id, name |
| `:label` | `<label>` | text |
| `:fieldset` | `<fieldset>` | id, legend text |
| `:table` | `<table>` | caption, id |
| `:row` | `<tr>` | row data array |
| `:table_row` | `<tr>` (hash-based) | column hash |
| `:frame` | `<iframe>`, `<frame>` | name, id |
| `:element` | Generic | tag + attributes |

### Kullanım örnekleri

```ruby
find :css, ".btn"
find :xpath, "//button[text()='OK']"
find :id, "submit"
find :button, "Sign in"
find :button, "Save", disabled: false
find :link, "Forgot password?"
find :link, "Click here", href: %r{/dashboard}
find :field, "Email"
find :fillable_field, "Notes"
find :radio_button, "Male"
find :checkbox, "I accept terms"
find :select, "Country"
find :option, "Turkey"
find :file_field, "Avatar"
find :label, "Email Address"
find :table, "Users"
find :row, ["John", "admin@x.com", "Active"]
find :element, :button, type: "submit"
find :frame, "payment_iframe"
```

### `Capybara.default_selector`
Selector type belirtilmediğinde ne kullanılacağı:
```ruby
Capybara.default_selector = :css        # default
Capybara.default_selector = :xpath      # tüm find çağrıları XPath olarak yorumlanır
```

```ruby
find("#submit")                 # :css mode → "#submit"
find("//button")                # :xpath mode → "//button"
```

### Selector vs filter
- **Selector**: ne tip eleman (`:button`, `:link`)
- **Filter**: o tipi nasıl daraltırız (`text:`, `disabled:`, `class:`)

```ruby
find :button, "Save", disabled: false, class: "primary"
#    selector type +  + locator + filters
```

### Locator önceliği — best practice
```
1. data-testid (en sağlam — tasarım refactor'dan etkilenmez)
2. Label text (a11y'yi garanti eder, kullanıcı diline yakın)
3. id (kararlı ama tasarım değişebilir)
4. CSS class (kırılgan; tasarım değişince kırılır)
5. XPath (en kırılgan; son çare)
```

### XPath'ten kaçınma sebepleri
- Hierarchy'ye duyarlı (`/div[1]/span[2]` küçük DOM değişiminde kırılır)
- Yavaş (tree traversal)
- Okunması zor
- Developer'ın değişikliklerinden tezgah dışında etkilenir

XPath sadece şu durumlarda:
- Çok kompleks DOM (tablo cell'i içeren span'in parent'ı)
- Built-in selector eklenip eklenmedi (`find :xpath, "//svg/path[@d='M3...']"`)

---

## 20. Custom Selectors, Filters, Expressions

Capybara'nın selector sistemini genişletmek için.

### Custom selector tanımlama
```ruby
Capybara.add_selector(:data_testid) do
  css { |id| "[data-testid='#{id}']" }
end

# Kullanım
find :data_testid, "login-submit"
all :data_testid, "user-row"
```

### XPath-based custom selector
```ruby
Capybara.add_selector(:phone) do
  xpath { |digits| ".//a[starts-with(@href, 'tel:#{digits}')]" }
end

find :phone, "+90"
```

### Selector with filter
```ruby
Capybara.add_selector(:product) do
  css { |sku| "[data-product-sku='#{sku}']" }

  filter(:in_stock, :boolean) do |node, value|
    (node["data-in-stock"] == "true") == value
  end
end

# Kullanım
find :product, "ABC-123", in_stock: true
```

### Selector with multiple expressions
```ruby
Capybara.add_selector(:menu_item) do
  css(:menu) { |label| ".menu li[data-label='#{label}']" }
  css(:sidebar) { |label| ".sidebar .item[data-name='#{label}']" }
end

# Kullanım
find :menu_item, "Home", :menu
find :menu_item, "Settings", :sidebar
```

### Locator builder
```ruby
Capybara.add_selector(:user_card) do
  css { |id| ".user-card" }

  filter(:user_id) { |node, id| node["data-user-id"] == id.to_s }
  filter(:role)    { |node, role| node[".role"]&.text == role }

  describe { |options| " for user_id #{options[:user_id]} with role #{options[:role]}" }
end

# Kullanım
find :user_card, user_id: 42, role: "admin"
```

`describe` block → hata mesajında daha okunabilir output.

### Inheritance — built-in'i extend etmek
```ruby
Capybara.modify_selector(:button) do
  filter(:variant) { |node, variant| node[:class].split.include?("btn-#{variant}") }
end

# Kullanım
find :button, "Save", variant: "primary"
```

### Lead seviyesi tavsiye
Custom selector'lar tasarım sistemine has olabilir. Örneğin:
- `find :data_testid, "..."` — tüm test'lerde kullanılır (built-in `test_id` config ile alternatif)
- `find :ant_button, "Save"` — Ant Design component'i için custom
- `find :material_field, "Email"` — Material UI için

**Kural:** Custom selector → tasarım sistemiyle entegre. Bir kez tanımla, her yerde kullan.

---

## 21. JavaScript Interaction

Sadece JS-capable driver'larda (Selenium, Cuprite). Rack::Test'te yoktur.

### Script execute
```ruby
# Sync — sonucu döndürmez
page.execute_script("window.scrollTo(0, 200)")
page.execute_script("document.body.style.background = 'red'")

# Argument geçme (Cuprite/Selenium)
page.execute_script("arguments[0].click()", find(".btn"))

# Eval — sonucu döndürür
title = page.evaluate_script("document.title")
count = page.evaluate_script("document.querySelectorAll('.item').length")

# Async (callback-based)
result = page.evaluate_async_script(<<~JS)
  var callback = arguments[arguments.length - 1];
  setTimeout(() => callback("done"), 1000);
JS
```

### Alert / Confirm / Prompt
```ruby
# Alert kabul et
page.accept_alert { click_button "Delete" }

# Confirm — OK
page.accept_confirm { click_button "Delete" }

# Confirm — Cancel
page.dismiss_confirm { click_button "Delete" }

# Prompt — değer gir
page.accept_prompt(with: "John") { click_button "Set Name" }

# Prompt — Cancel
page.dismiss_prompt { click_button "Set Name" }

# Alert text yakala
text = page.accept_alert { click_link "About" }
puts text

# Text expectation
page.accept_alert("Are you sure?") { click_button "Delete" }
```

### Browser console & network logs
```ruby
# Selenium — browser logs
page.driver.browser.logs.get(:browser).each do |log|
  puts "#{log.level}: #{log.message}"
end

# Cuprite — network requests intercept
page.driver.network_traffic.each do |req|
  puts "#{req.method} #{req.url} → #{req.response.status}"
end

# Cuprite — JS errors
Capybara.register_driver :cuprite do |app|
  Capybara::Cuprite::Driver.new(app, js_errors: true)
  # JS error → exception
end
```

### Mouse events
```ruby
element = find(".target")
element.hover
element.right_click
element.double_click

# Click coordinates
page.driver.browser.action
            .move_to(element.native, 10, 20)
            .click
            .perform
```

### Scroll
```ruby
page.execute_script("window.scrollTo(0, document.body.scrollHeight)")
find(".element").scroll_to(:center)
find(".element").scroll_to(:bottom)

# Element scroll
find(".container").scroll_by(x: 0, y: 100)
```

### Keyboard
```ruby
find("body").send_keys([:control, "a"])         # select all
find("body").send_keys([:control, "c"])         # copy
find("body").send_keys(:escape)
find("body").send_keys(:tab, :tab, :enter)
```

### Cookies (Selenium/Cuprite)
```ruby
page.driver.browser.manage.add_cookie(
  name: "session", value: "abc123", path: "/"
)
page.driver.browser.manage.cookie_named("session")
page.driver.browser.manage.delete_cookie("session")
page.driver.browser.manage.delete_all_cookies
```

### Local/Session Storage
```ruby
# Set
page.execute_script("window.localStorage.setItem('key', 'value')")

# Get
value = page.evaluate_script("window.localStorage.getItem('key')")

# Clear
page.execute_script("window.localStorage.clear()")
```

---

## 22. Sessions ve Windows

### Multi-session (iki kullanıcı, aynı test)
```ruby
it "admin onaylar, user görür" do
  Capybara.using_session(:admin) do
    visit "/admin/posts"
    fill_in "Email", with: "admin@x.com"
    fill_in "Password", with: "secret"
    click_button "Login"
    click_on "Approve"
  end

  Capybara.using_session(:user) do
    visit "/posts"
    expect(page).to have_content("Approved")
  end
end
```

Her `using_session(:name)` bağımsız bir browser session açar.

### Session yönetim API
```ruby
Capybara.current_session
Capybara.session_name              # :default
Capybara.session_name = :other
Capybara.reset_sessions!           # tüm session'ları reset

# Session-specific
session = Capybara::Session.new(:cuprite, MyApp.application)
session.visit "/"
```

### Window management
```ruby
# Mevcut window'lar
page.windows                         # [Window1, Window2, ...]
page.current_window
page.current_window.size
page.current_window.maximize
page.current_window.fullscreen
page.current_window.resize_to(1920, 1080)

# Yeni window aç
new_window = window_opened_by { click_link "Open new" }

# Belirli window'a switch
page.switch_to_window(new_window)
# veya
page.switch_to_window(page.windows.last)

# Window içinde
within_window(new_window) do
  expect(page).to have_content("Second page")
end

# Window kapat
new_window.close
```

### Window opened by — pattern
```ruby
# Standart pattern
new_window = window_opened_by(wait: 5) do
  click_link "Open dashboard"
end

within_window(new_window) do
  # Yeni sekmede ne varsa test et
end

new_window.close
# Otomatik default window'a döner
```

### Lead seviyesi pitfall
```ruby
# ❌ Window close + switch unutursan diğer test'lere sızar
new_window = window_opened_by { click_link "Open" }
# new_window'da bir şey yap
# unutma: close edilmedi! sonraki test'in current_window beklediği değil

# ✅ Block-scoped, otomatik cleanup
within_window(new_window) do
  click_button "OK"
end
new_window.close
```

### Driver constraint
- **Rack::Test:** Window/tab desteklemez.
- **Selenium / Cuprite:** Tam destek.

### Reset sessions
```ruby
# Manual
Capybara.reset_sessions!

# RSpec'te her test sonrası otomatik
after(:each) { Capybara.reset_sessions! }    # default davranış
```

`reset_sessions!`:
- Cookie'ler temizlenir
- URL `about:blank`'a döner
- Local/session storage temizlenir
- **Browser process kapatılmaz** — performans için yeniden kullanılır

---

## 23. Server & External App Testing

### Capybara server modes

**1. Built-in server (default)** — `Capybara.run_server = true`
Capybara, `Capybara.app`'i Puma ile localhost'ta serve eder.
```ruby
Capybara.run_server = true
Capybara.app         = MyApp.application
Capybara.server      = :puma, { Silent: true, Threads: "0:4" }
Capybara.server_port = 0                          # rastgele
Capybara.server_host = "127.0.0.1"
```

**2. External server** — `Capybara.run_server = false`
Capybara hiçbir şey serve etmez. Sen kendin başka bir yerde serve edersin.
```ruby
Capybara.run_server = false
Capybara.app_host   = "https://staging.example.com"
```

### Use case: Staging environment testing
```ruby
# spec/support/capybara.rb
if ENV["TEST_TARGET"] == "staging"
  Capybara.configure do |c|
    c.run_server = false
    c.app_host   = "https://staging.example.com"
    c.default_driver    = :selenium_chrome_headless    # rack_test çalışmaz
    c.javascript_driver = :selenium_chrome_headless
  end
else
  Capybara.configure do |c|
    c.run_server = true
    c.app        = Rails.application
  end
end
```

```bash
TEST_TARGET=staging bundle exec rspec
```

### Server options (Puma)
```ruby
Capybara.server = :puma, {
  Silent: true,
  Threads: "0:4",                     # min:max thread
  workers: 0                          # process count (0 = single)
}
```

### Custom server
```ruby
Capybara.server = :webrick           # Puma yerine WEBrick
Capybara.server = ->(app, port, host) do
  Puma::Server.new(app).run(host: host, port: port)
end
```

### Server host & port
```ruby
Capybara.server_host = "0.0.0.0"     # tüm interface'ler (Docker)
Capybara.server_port = 3001          # belirli port

# Server URL
Capybara.server_url                  # → "http://127.0.0.1:3001"
```

### `Capybara.always_include_port`
```ruby
Capybara.always_include_port = true   # URL'lere :port ekle
# → visit "/foo" → http://127.0.0.1:3001/foo
```

### Multi-host
```ruby
Capybara.using_host("https://api.example.com") do
  visit "/users"        # → https://api.example.com/users
end
```

### Server errors raise
```ruby
Capybara.raise_server_errors = true   # default; server-side exception → test fail
Capybara.raise_server_errors = false  # suppress
```

### Lead seviyesi karar
| Senaryo | Konfig |
|---|---|
| Rails / Sinatra app test | `run_server = true`, `app = MyApp.application` |
| Staging environment test | `run_server = false`, `app_host = "https://..."` |
| Smoke production | `run_server = false`, `app_host = "https://prod..."` (dikkatli) |
| Docker'da test | `server_host = "0.0.0.0"` |

---

## 24. Cookies, Headers, Storage

### Rack::Test driver
```ruby
# Cookie set
page.driver.browser.set_cookie("session=abc123")
page.driver.set_cookie("user_id", "42")

# Cookie get
page.driver.request.cookies              # hash
page.driver.browser.current_session.cookie_jar

# Custom header
page.driver.header("Authorization", "Bearer xyz")
page.driver.header("X-API-Key", "secret")

# Manuel HTTP request
page.driver.submit :post, "/api/users", { name: "John" }
page.driver.put "/api/users/1", { name: "Jane" }
```

### Selenium driver
```ruby
# Cookie API
page.driver.browser.manage.add_cookie(
  name:   "session",
  value:  "abc123",
  path:   "/",
  domain: "127.0.0.1"
)

cookie = page.driver.browser.manage.cookie_named("session")
all_cookies = page.driver.browser.manage.all_cookies

page.driver.browser.manage.delete_cookie("session")
page.driver.browser.manage.delete_all_cookies
```

### Cuprite driver
```ruby
# Set
page.driver.set_cookie("session", "abc123",
  domain: "127.0.0.1", path: "/")

# Get
page.driver.cookies

# Clear
page.driver.clear_cookies

# Network intercept (Cuprite özel)
page.driver.network_traffic                     # request history
page.driver.clear_network_traffic
```

### Local/Session Storage (JS driver'lar)
```ruby
# Set
page.execute_script("localStorage.setItem('key', 'value')")
page.execute_script("sessionStorage.setItem('temp', 'data')")

# Get
val = page.evaluate_script("localStorage.getItem('key')")

# Clear
page.execute_script("localStorage.clear()")
page.execute_script("sessionStorage.clear()")
```

### Header injection pattern (auth bypass için)
```ruby
# Custom helper
def login_via_api(user)
  token = ApiAuth.generate_token(user)
  page.driver.header("Authorization", "Bearer #{token}")     # Rack::Test
  # veya
  page.execute_script("localStorage.setItem('token', '#{token}')")   # JS driver
  visit "/dashboard"
end
```

Bu E2E'yi hızlandırır — UI'dan login yerine direkt token enjekte.

---

## 25. File Upload / Download

### Upload
```ruby
attach_file "Avatar", "spec/fixtures/files/avatar.png"

# Field locator ile
attach_file "user[avatar]", file_path
attach_file "user_avatar",  file_path

# Multi-file
attach_file "Photos", [
  "spec/fixtures/files/img1.png",
  "spec/fixtures/files/img2.png"
]

# Hidden file input (modern drag-drop UI)
attach_file "Avatar", file_path, make_visible: true
```

`make_visible: true` → JS ile `display: block` enjekte eder, sonra dosyayı ekler.

### Download — Selenium Chrome
```ruby
Capybara.register_driver :selenium_chrome_download do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless=new")
  options.add_preference("download.default_directory", "/tmp/downloads")
  options.add_preference("download.prompt_for_download", false)
  options.add_preference("download.directory_upgrade", true)

  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

# Test
it "downloads report" do
  click_link "Download CSV"

  # Wait for file
  download_path = "/tmp/downloads/report.csv"
  wait_for_file(download_path, timeout: 10)

  expect(File.exist?(download_path)).to be true
  expect(File.read(download_path)).to include("Header1,Header2")
end

def wait_for_file(path, timeout:)
  Timeout.timeout(timeout) do
    sleep 0.1 until File.exist?(path)
  end
end
```

### Download — Cuprite
```ruby
Capybara.register_driver :cuprite do |app|
  Capybara::Cuprite::Driver.new(app,
    save_path: "/tmp/downloads"
  )
end

# Cuprite native API
page.driver.set_download_path("/tmp/downloads")
click_link "Download"
page.driver.wait_for_download(timeout: 10)
```

### Drag-and-drop file upload (modern UI)
```ruby
# Drop zone target
drop_zone = find(".drop-zone")

# Capybara built-in drop
attach_file "Photos", file_path, drop_target: drop_zone

# veya JS ile
script = <<~JS
  var dt = new DataTransfer();
  // ... file create
  var event = new DragEvent('drop', { dataTransfer: dt });
  arguments[0].dispatchEvent(event);
JS
page.execute_script(script, drop_zone)
```

### Lead bakışı: dosya path'leri
```ruby
# ✅ Path constant'ları
module FixtureHelpers
  FIXTURES = Rails.root.join("spec/fixtures/files")

  def fixture_file(name)
    FIXTURES.join(name).to_s
  end
end

# Test
attach_file "Avatar", fixture_file("avatar.png")
```

---

## 26. Page Object Model — POM

`Capybara::Session` (= `page`) ile karıştırma. POM **senin yazdığın sınıflar**dır.

### Temel POM
```ruby
# spec/support/pages/base_page.rb
class BasePage
  include Capybara::DSL

  def self.path
    raise NotImplementedError
  end

  def visit_page
    visit self.class.path
    self
  end

  def loaded?
    raise NotImplementedError
  end

  def wait_for_loaded(timeout: 10)
    Capybara.using_wait_time(timeout) { loaded? }
    self
  end
end

# spec/support/pages/login_page.rb
class LoginPage < BasePage
  PATH = "/login".freeze

  def self.path
    PATH
  end

  def loaded?
    has_field?("Email") && has_field?("Password")
  end

  def login_as(email:, password:)
    fill_in "Email",    with: email
    fill_in "Password", with: password
    click_button "Sign in"
    DashboardPage.new
  end

  def error_message
    find(".alert-danger").text
  end

  def forgot_password
    click_link "Forgot password?"
    ForgotPasswordPage.new
  end
end

# spec/support/pages/dashboard_page.rb
class DashboardPage < BasePage
  PATH = "/dashboard".freeze

  def self.path
    PATH
  end

  def loaded?
    has_current_path?(PATH) && has_selector?("h1", text: "Dashboard")
  end

  def welcome_message
    find("h1").text
  end

  def logout
    click_link "Logout"
    LoginPage.new
  end
end
```

### Test — fluent chain
```ruby
RSpec.describe "Login flow", type: :system do
  let(:login_page) { LoginPage.new }

  it "dashboard'a yönlendirir" do
    dashboard = login_page
                  .visit_page
                  .login_as(email: "user@example.com", password: "secret")
                  .wait_for_loaded

    expect(dashboard.welcome_message).to eq("Welcome back!")
  end
end
```

### Component pattern (büyük sayfalar için)
```ruby
class DashboardPage < BasePage
  def header
    HeaderComponent.new(find("header"))
  end

  def sidebar
    SidebarComponent.new(find(".sidebar"))
  end

  def main_content
    ContentComponent.new(find("main"))
  end
end

class HeaderComponent
  def initialize(node)
    @node = node
  end

  def user_menu
    @node.find(".user-menu")
  end

  def logout
    user_menu.click
    @node.click_link "Logout"
    LoginPage.new
  end
end

# Test
dashboard.header.logout
```

### Lazy element pattern
```ruby
class LoginPage < BasePage
  def email_field
    find_field "Email"           # her çağrıda yeniden bulur (stale-safe)
  end

  def password_field
    find_field "Password"
  end

  def submit_btn
    find_button "Sign in"
  end

  def login_as(email:, password:)
    email_field.set(email)
    password_field.set(password)
    submit_btn.click
    DashboardPage.new
  end
end
```

**Avantaj:** `email_field` cached değil, her çağrıda yeniden lookup → stale element exception olmaz.

### `SitePrism` gem (advanced POM)
```ruby
gem "site_prism"
```

```ruby
class LoginPage < SitePrism::Page
  set_url "/login"

  element :email_field,    "input[name='user[email]']"
  element :password_field, "input[name='user[password]']"
  element :submit_btn,     "button[type='submit']"

  sections :alerts, ".alert" do
    element :message, ".message"
  end

  def login_as(email:, password:)
    email_field.set(email)
    password_field.set(password)
    submit_btn.click
    DashboardPage.new
  end
end

# Test
login_page = LoginPage.new
login_page.load
login_page.login_as(email: "x@y.com", password: "secret")
```

**SitePrism avantajları:**
- DSL ile declarative element tanımı
- Lazy evaluation
- Section/component yapısı
- `wait_until_*_visible` built-in

**Lead bakışı:** SitePrism vs custom POM — team tercihi, küçük projelerde custom yeterli, büyükte SitePrism kazanım sağlar.

### Method chaining ilkeleri
Her action method bir sonraki page'i (veya self'i) dönmeli:
```ruby
def login_as(email:, password:)
  fill_in "Email", with: email
  fill_in "Password", with: password
  click_button "Sign in"
  DashboardPage.new     # ← bir sonraki page
end

def update_email(email)
  fill_in "Email", with: email
  click_button "Save"
  self                  # ← self (aynı sayfada kalıyor)
end
```

---

## 27. Cross-Browser & Cloud Testing

### Selenium Remote driver
```ruby
Capybara.register_driver :selenium_remote do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  Capybara::Selenium::Driver.new(
    app,
    browser: :remote,
    url:     "http://selenium-hub:4444/wd/hub",
    capabilities: options
  )
end

Capybara.javascript_driver = :selenium_remote
```

### BrowserStack
```ruby
Capybara.register_driver :browserstack do |app|
  caps = Selenium::WebDriver::Remote::Capabilities.new(
    "browserName"            => "Chrome",
    "browserVersion"         => "latest",
    "bstack:options" => {
      "os"               => "OS X",
      "osVersion"        => "Ventura",
      "buildName"        => "Build #{ENV['CI_BUILD']}",
      "sessionName"      => "Login flow",
      "userName"         => ENV["BROWSERSTACK_USER"],
      "accessKey"        => ENV["BROWSERSTACK_KEY"],
      "local"            => false,
      "debug"            => true,
      "consoleLogs"      => "verbose",
      "networkLogs"      => true,
      "video"            => true
    }
  )

  Capybara::Selenium::Driver.new(
    app,
    browser: :remote,
    url:     "https://hub-cloud.browserstack.com/wd/hub",
    capabilities: caps
  )
end
```

### LambdaTest
```ruby
Capybara.register_driver :lambdatest do |app|
  caps = {
    "browserName"    => "Chrome",
    "browserVersion" => "latest",
    "LT:Options" => {
      "platformName" => "Windows 11",
      "build"        => "Build #{ENV['CI_BUILD']}",
      "name"         => "Login flow test",
      "user"         => ENV["LT_USERNAME"],
      "accessKey"    => ENV["LT_ACCESS_KEY"],
      "video"        => true,
      "network"      => true,
      "console"      => true,
      "visual"       => true
    }
  }

  Capybara::Selenium::Driver.new(
    app,
    browser: :remote,
    url:     "https://#{ENV['LT_USERNAME']}:#{ENV['LT_ACCESS_KEY']}@hub.lambdatest.com/wd/hub",
    capabilities: caps
  )
end
```

### Sauce Labs
```ruby
Capybara.register_driver :sauce do |app|
  caps = {
    "browserName"    => "chrome",
    "browserVersion" => "latest",
    "platformName"   => "Windows 11",
    "sauce:options" => {
      "build"      => "Build #{ENV['CI_BUILD']}",
      "name"       => "Test name",
      "username"   => ENV["SAUCE_USERNAME"],
      "accessKey"  => ENV["SAUCE_ACCESS_KEY"]
    }
  }

  Capybara::Selenium::Driver.new(
    app,
    browser: :remote,
    url:     "https://ondemand.us-west-1.saucelabs.com:443/wd/hub",
    capabilities: caps
  )
end
```

### Cross-browser matrix testi
```ruby
RSpec.describe "Critical flow", type: :system do
  [:selenium_chrome, :selenium_firefox, :selenium_edge].each do |browser|
    context "on #{browser}" do
      before { Capybara.current_driver = browser }
      after  { Capybara.use_default_driver }

      it "completes login" do
        # ...
      end
    end
  end
end
```

### Lead seviyesi cloud kararı
| Provider | Güçlü yön | Zayıf yön | Lead seçimi |
|---|---|---|---|
| **BrowserStack** | 3000+ device, mature | Pahalı | Enterprise |
| **Sauce Labs** | DevOps entegrasyonu, eski | Pahalı | Enterprise |
| **LambdaTest** | Uygun fiyat, hızlı | BrowserStack kadar geniş değil | Startup, mid-size |
| **Self-hosted Grid** | Tam kontrol | Bakım maliyeti | Çok büyük scale |

### Cross-browser cost-benefit
- Chrome %75 user base → ana test target
- Firefox %5, Safari %15, Edge %5 → spot test
- Mobile (iOS Safari, Chrome Android) → ayrı framework (Appium) genelde

**Karar:** Critical path'i 4 browser'da test et; geri kalan sadece Chrome'da. Cross-browser maliyeti smoke suite'e sınırla.

---

## 28. Test Data Management

### FactoryBot entegrasyonu
```ruby
# Gemfile
gem "factory_bot_rails"

# spec/rails_helper.rb
RSpec.configure do |c|
  c.include FactoryBot::Syntax::Methods
end
```

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password         { "password123" }

    trait :admin do
      role { "admin" }
    end

    trait :with_avatar do
      after(:build) do |user|
        user.avatar.attach(
          io: File.open(Rails.root.join("spec/fixtures/avatar.png")),
          filename: "avatar.png"
        )
      end
    end
  end
end
```

```ruby
# Test'te
let!(:admin) { create(:user, :admin, email: "admin@example.com") }
let(:user)   { build(:user) }       # save etmeden

it "admin görür" do
  sign_in_as(admin)
  visit "/admin"
  expect(page).to have_content("Admin Panel")
end
```

### DatabaseCleaner
```ruby
# Gemfile
gem "database_cleaner-active_record"

# spec/support/database_cleaner.rb
RSpec.configure do |c|
  c.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  c.before(:each) do
    # Selenium/Cuprite gibi multi-process driver'larda transaction yerine truncation
    DatabaseCleaner.strategy = if [:cuprite, :selenium_chrome_headless].include?(Capybara.current_driver)
                                  :truncation
                                else
                                  :transaction
                                end
    DatabaseCleaner.start
  end

  c.after(:each) do
    DatabaseCleaner.clean
  end
end
```

**Neden Selenium'da transaction çalışmaz?**
Selenium driver ayrı bir process (browser). Rails app ve test farklı thread'ler/connection'lar kullanır → transaction visibility sorunu. Çözüm: truncation veya `use_transactional_tests: false`.

### Rails 5.1+ system test built-in çözüm
```ruby
# Rails 5.1+
RSpec.configure do |c|
  c.before(:each, type: :system) do
    driven_by :rack_test         # JS gerekmiyorsa
  end

  c.before(:each, type: :system, js: true) do
    driven_by :selenium_chrome_headless
  end
end
```

Rails 5.1'in `ActiveRecord::TestFixtures.use_transactional_tests = true` + Selenium kombinasyonu **otomatik shared connection** kullanır → transaction'lar görünür.

### YAML fixture
```yaml
# spec/fixtures/users.yml
admin:
  email: admin@example.com
  password: admin123
  role: admin

basic:
  email: user@example.com
  password: user123
  role: user
```

```ruby
# Helper
class TestData
  USERS = YAML.load_file("spec/fixtures/users.yml").freeze

  def self.user(type)
    USERS.fetch(type.to_s)
  end
end

# Test
user = TestData.user(:admin)
sign_in_as(email: user["email"], password: user["password"])
```

### Test data isolation pattern
```ruby
# spec/support/data_helpers.rb
module DataHelpers
  def create_isolated_user(attrs = {})
    # Each test gets a unique user — no data leak
    create(:user,
      email: "test-#{SecureRandom.hex(4)}@example.com",
      **attrs
    )
  end
end

# Test
let(:user) { create_isolated_user(role: "admin") }
```

### VCR / WebMock — external API stub
```ruby
# Gemfile
gem "vcr"
gem "webmock"

# spec/support/vcr.rb
require "vcr"

VCR.configure do |c|
  c.cassette_library_dir = "spec/fixtures/vcr_cassettes"
  c.hook_into            :webmock
  c.configure_rspec_metadata!
  c.allow_http_connections_when_no_cassette = false
end

# Test
it "fetches user data", :vcr do
  visit "/integrations/stripe"
  expect(page).to have_content("Connected")
end
```

VCR external API call'ları **kaydedip replay eder** → testler hızlı, deterministic.

---

## 29. Cucumber & Minitest Entegrasyonu

### Cucumber
```ruby
# features/support/env.rb
require "capybara/cucumber"

Capybara.configure do |c|
  c.default_driver           = :rack_test
  c.javascript_driver        = :selenium_chrome_headless
  c.default_max_wait_time    = 5
end
```

`capybara/cucumber` Capybara::DSL'i tüm step definition'larına otomatik dahil eder.

```ruby
# features/step_definitions/auth_steps.rb
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

Detaylı entegrasyon için bkz: `cucumber.md` (bu repo'da).

### Minitest
```ruby
# test/test_helper.rb
require "capybara/minitest"

class ActionDispatch::SystemTestCase
  include Capybara::Minitest::Assertions

  def teardown
    Capybara.reset_sessions!
    super
  end
end

# test/system/login_test.rb
require "test_helper"

class LoginTest < ActionDispatch::SystemTestCase
  test "kullanıcı giriş yapabilir" do
    visit "/login"
    fill_in "Email", with: "user@x.com"
    fill_in "Password", with: "secret"
    click_button "Sign in"

    assert_current_path("/dashboard")
    assert_text("Welcome")
    assert_no_text("Error")
  end
end
```

Minitest assertion karşılıkları:
| RSpec | Minitest |
|---|---|
| `expect(page).to have_content("X")` | `assert_content("X")` veya `assert_text("X")` |
| `expect(page).to have_no_content("X")` | `assert_no_content("X")` |
| `expect(page).to have_selector(".x")` | `assert_selector(".x")` |
| `expect(page).to have_button("Save")` | `assert_button("Save")` |

---

## 30. Reporting

### RSpec formatter'lar
```bash
rspec --format documentation                              # detaylı liste
rspec --format progress                                    # kısa nokta-tabanlı
rspec --format html --out tmp/rspec.html                   # HTML
rspec --format json --out tmp/rspec.json
rspec --format RspecJunitFormatter --out tmp/junit.xml    # CI için
```

```ruby
# .rspec
--format documentation
--format RspecJunitFormatter --out tmp/junit.xml
--color
```

### `rspec_junit_formatter` (CI için)
```ruby
gem "rspec_junit_formatter"
```

Jenkins, GitLab, CircleCI gibi CI sistemleri JUnit XML parse eder.

### Allure (en yaygın gelişmiş rapor)
```ruby
# Gemfile
gem "allure-rspec"
```

```ruby
# spec/rails_helper.rb
require "allure-rspec"

AllureRspec.configure do |c|
  c.results_directory         = "tmp/allure-results"
  c.clean_results_directory   = true
  c.environment_properties    = {
    rails:    Rails.version,
    capybara: Capybara::VERSION,
    driver:   Capybara.javascript_driver
  }
end
```

```ruby
# Failure screenshot
RSpec.configure do |c|
  c.after(:each, type: :system) do |example|
    if example.exception
      Allure.add_attachment(
        name:   "Failure screenshot",
        source: page.driver.browser.screenshot_as(:png),
        type:   Allure::ContentType::PNG
      )
      Allure.add_attachment(
        name:   "HTML",
        source: page.html,
        type:   "text/html"
      )
    end
  end
end
```

```bash
bundle exec rspec --format AllureRspecFormatter
allure generate tmp/allure-results --clean -o tmp/allure-html
allure open tmp/allure-html
```

### `capybara-screenshot` gem (otomatik failure screenshot)
```ruby
gem "capybara-screenshot"

# spec/rails_helper.rb
require "capybara-screenshot/rspec"

Capybara::Screenshot.register_driver(:selenium_chrome_headless) do |driver, path|
  driver.browser.save_screenshot(path)
end
```

Failed test'lerde otomatik `tmp/capybara/screenshot_*.png` kaydeder.

### CI report upload örneği (GitHub Actions)
```yaml
- name: Run tests
  run: bundle exec rspec --format RspecJunitFormatter --out tmp/junit.xml

- name: Upload test results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: |
      tmp/junit.xml
      tmp/capybara/
      tmp/allure-results/

- name: Publish JUnit report
  uses: mikepenz/action-junit-report@v4
  if: always()
  with:
    report_paths: tmp/junit.xml
```

---

## 31. Debugging Araçları

### `save_and_open_page`
HTML dosyasını kaydet ve **default browser'da aç**.
```ruby
it "debugs" do
  visit "/users"
  save_and_open_page              # → tmp/capybara/capybara-*.html
end
```

### `save_and_open_screenshot`
```ruby
save_and_open_screenshot          # PNG kaydet + aç
```

### `save_page` / `save_screenshot`
Açmadan sadece kaydet:
```ruby
page.save_page("tmp/debug.html")
page.save_screenshot("tmp/debug.png")
```

### Path config
```ruby
Capybara.save_path = "tmp/capybara"
```

### `binding.pry` / `binding.irb` — interactive debug
```ruby
gem "pry-byebug"
```

```ruby
it "interactive debug" do
  visit "/login"
  fill_in "Email", with: "test@x.com"
  binding.pry                  # ← burada IRB açılır, browser açık kalır
  click_button "Sign in"
end
```

Pry içinde:
```
pry> page.current_url
pry> page.html
pry> find(".error").text
pry> all(".item").map(&:text)
pry> page.driver.browser.save_screenshot("tmp/debug.png")
```

### Browser console logs
```ruby
# Selenium
page.driver.browser.logs.get(:browser).each { |l| puts l.message }

# Cuprite
page.driver.console_messages         # JS console output
page.driver.error_messages           # JS errors
```

### Network logs (Cuprite)
```ruby
page.driver.network_traffic.each do |request|
  puts "#{request.method} #{request.url}"
  puts "  Status: #{request.response&.status}"
  puts "  Type: #{request.response&.content_type}"
end
```

### `puts page.html` — raw HTML dump
```ruby
it "debug HTML" do
  visit "/dashboard"
  puts "─" * 80
  puts page.html
  puts "─" * 80
end
```

### `page.current_path` ve `page.title`
```ruby
puts page.current_url
puts page.current_path
puts page.title
puts page.response_headers.inspect       # Rack::Test
```

### Visual debugger — `:selenium_chrome` (headless DEĞİL)
Browser ekranda açılır, manuel inspect yapabilirsin.
```ruby
it "manual debug", driver: :selenium_chrome do
  visit "/dashboard"
  binding.pry
end
```

### Pause + screenshot helper
```ruby
def debug_pause(msg = "Paused", screenshot: true)
  puts "\n🔍 #{msg}"
  page.save_screenshot("tmp/debug_#{Time.now.to_i}.png") if screenshot
  STDIN.gets                    # Enter'a basana kadar bekle
end

# Test'te
fill_in "Email", with: "x@y.com"
debug_pause("Email girildi mi?")
click_button "Submit"
```

### Trace mode (Cuprite)
```ruby
Capybara::Cuprite::Driver.new(app,
  logger:    STDOUT,             # CDP komutlarını logla
  inspector: ENV["INSPECTOR"]    # `INSPECTOR=true rspec` → DevTools açık
)
```

```bash
INSPECTOR=true bundle exec rspec spec/system/login_spec.rb
```

---

## 32. Flaky Test Stratejisi

Flaky test = rastgele fail/pass eden test. **QA Lead'in en büyük headache'i.**

### Tespit
```bash
# Aynı test'i tekrar tekrar koştur
for i in {1..20}; do bundle exec rspec spec/system/checkout_spec.rb || break; done

# Seed ile reproduce et
bundle exec rspec --seed 12345

# Bisect ile bağımlı test'leri bul
bundle exec rspec --bisect
```

### Sık nedenler & çözümler

| Neden | Çözüm |
|---|---|
| `sleep` kullanımı | Capybara matcher'ları ile değiştir |
| `not_to have_*` | `have_no_*` |
| `all().size` assertion | `have_selector count:` |
| Test data leak | DatabaseCleaner, isolated factories |
| Yarış koşulu (race condition) | `wait:` opsiyonu ile artır |
| External API timeout | WebMock + VCR stub |
| Browser process leak | `reset_sessions!`, driver restart |
| Async JS animation | `Capybara.disable_animation = true` |
| Order dependency | `--order random`, isolated state |
| Cookie/session leak | `reset_sessions!` her test sonrası |
| Time-sensitive logic | Timecop / ActiveSupport::Testing::TimeHelpers |

### Disable animations
```ruby
Capybara.disable_animation = true       # Capybara 3.5+
```

Veya manuel CSS injection:
```ruby
RSpec.configure do |c|
  c.before(:each, type: :system, js: true) do
    page.execute_script(<<~JS) if Capybara.current_driver.to_s.include?("selenium")
      var style = document.createElement('style');
      style.innerHTML = '*, *::before, *::after { transition: none !important; animation: none !important; }';
      document.head.appendChild(style);
    JS
  end
end
```

### Retry pattern — `rspec-retry`
```ruby
# Gemfile
gem "rspec-retry"

# spec/rails_helper.rb
require "rspec/retry"

RSpec.configure do |c|
  c.verbose_retry = true
  c.display_try_failure_messages = true

  c.around :each, :flaky do |ex|
    ex.run_with_retry(retry: 3)
  end
end
```

```ruby
it "occasionally flaky", :flaky do
  # ... up to 3 retries
end
```

**Lead uyarısı:** Retry **son çare**. Önce root cause analiz et:
1. Test gerçekten flaky mi yoksa bir feature gerçekten broken mi?
2. Auto-wait yetersiz mi? `wait:` artır.
3. Async operation kullanılıyor mu? Doğru bekleme?
4. Test data leak var mı?

### Knapsack Pro — paralel + history-based split
```ruby
gem "knapsack_pro"
```

CI node'ları arasında test'i optimal şekilde böler, **flaky test'leri tarihçesi üzerinden tespit eder**.

### Flaky test dashboard
Kendi dashboard'unu kur:
- Her CI run'da test sonuçları JSON olarak kaydedilir
- Test bazında pass/fail oranı izlenir
- %1'in üstünde flake rate olan test'ler quarantine'e alınır
- Sprint sonu flaky test backlog

### Quarantine pattern
```ruby
RSpec.configure do |c|
  c.filter_run_excluding :quarantine unless ENV["RUN_QUARANTINE"]
end

it "known flaky", :quarantine do
  # CI'da default skip, manuel çalıştırma için ayrı job
end
```

---

## 33. Visual & Accessibility Testing

### Visual regression — Percy
```ruby
# Gemfile
gem "percy-capybara"
```

```ruby
# spec/support/percy.rb
require "percy/capybara"
```

```ruby
it "homepage looks correct" do
  visit "/"
  Percy.snapshot(page, name: "Homepage")
end
```

Percy snapshot'ı server'a gönderir; previous baseline ile karşılaştırır, görsel diff dashboard'da gösterir.

### Visual regression — Applitools Eyes
```ruby
gem "eyes_selenium"
```

```ruby
it "checks visual consistency" do
  eyes = Applitools::Selenium::Eyes.new
  eyes.api_key = ENV["APPLITOOLS_API_KEY"]
  eyes.open(driver: Capybara.current_session.driver.browser,
            app_name: "My App",
            test_name: "Login page")

  visit "/login"
  eyes.check_window("Login screen")

  eyes.close
end
```

### Accessibility — axe-capybara
```ruby
gem "axe-core-capybara"
gem "axe-core-rspec"
```

```ruby
require "axe-rspec"

it "is accessible" do
  visit "/dashboard"
  expect(page).to be_axe_clean
end

# Specific rules
it "meets WCAG AA" do
  visit "/login"
  expect(page).to be_axe_clean.according_to(:wcag2aa)
end

# Skip specific rules
it "is accessible except color contrast" do
  visit "/legacy"
  expect(page).to be_axe_clean.excluding(:color_contrast)
end
```

`axe-core` Deque Labs'in açık kaynak accessibility engine'i — WCAG 2.0/2.1 standart kontrolleri.

### Lead seviyesi karar
| Test türü | Önerilen | Sıklık |
|---|---|---|
| Visual regression | Percy / Applitools | Critical sayfalar, every PR |
| Accessibility | axe-capybara | Tüm test'lerde inline |
| Cross-browser visual | BrowserStack Visual Tests | Release-prep |

---

## 34. Performans Optimizasyonu

Capybara test'lerinin yavaşlığı tipik bir Lead headache. Optimizasyon stratejileri:

### Driver seçimi
- **`:rack_test`** mümkünse — 5-10x hızlı (JS gerektirmiyor mu?)
- **`:cuprite`** > `:selenium` — CDP daha hızlı
- Headless > headful

### Session reuse
```ruby
# Browser process'i her test'te kapatıp açma — Capybara reset_sessions! yeterli
# Manuel quit istemiyorsan:
RSpec.configure do |c|
  c.after(:suite) do
    # Selenium process kapatma
    Capybara.current_session.driver.quit if Capybara.current_session.driver.respond_to?(:quit)
  end
end
```

### Parallel test
```ruby
gem "parallel_tests"
```

```bash
bundle exec parallel_rspec spec/system -n 4
```

Konfigürasyon:
```ruby
# spec/support/capybara.rb
Capybara.server_port = 0           # her process rastgele port
```

```ruby
# config/database.yml — Rails
test:
  database: myapp_test<%= ENV['TEST_ENV_NUMBER'] %>
```

### Disable animations
```ruby
Capybara.disable_animation = true
```

### Asset caching (Rails)
```ruby
# config/environments/test.rb
config.assets.debug         = false
config.assets.digest        = false
config.assets.compile       = false
config.public_file_server.enabled = true
```

### Bypass UI for setup (login)
```ruby
# Helper
def login_as_user_bypass_ui(user)
  if Capybara.current_driver == :rack_test
    page.driver.browser.set_cookie("user_id=#{user.id}")
  else
    # Custom test endpoint
    page.driver.post "/test/login", user_id: user.id
  end
  visit "/dashboard"
end
```

UI'dan login yapmadan, doğrudan session set et. **5-10 saniye/test tasarruf.**

### Knapsack Pro — CI optimization
```bash
gem "knapsack_pro"
```

CI'da test'i node'lar arası optimal böler:
```yaml
strategy:
  matrix:
    ci_node_index: [0, 1, 2, 3]
    ci_node_total: [4]

steps:
  - run: bundle exec rake knapsack_pro:rspec
```

### Database optimization
```ruby
# spec/rails_helper.rb
config.use_transactional_fixtures = true     # her test sonrası rollback (hızlı)
config.before(:suite) do
  ActiveRecord::Migration.maintain_test_schema!
end
```

### `Capybara.threadsafe = true`
Multi-threaded ortam (parallel_tests) için thread-safe mode:
```ruby
Capybara.threadsafe = true
```

---

## 35. Sık Karşılaşılan Pitfall'lar

### 1. `sleep` kullanmak
```ruby
# ❌
click_button "Submit"
sleep 3
expect(page).to have_content("Saved")

# ✅
click_button "Submit"
expect(page).to have_content("Saved")    # auto-wait
```

### 2. `not_to have_*` kullanmak
```ruby
# ❌ Race condition
expect(page).not_to have_content("Loading")

# ✅
expect(page).to have_no_content("Loading")
```

### 3. `all().size` ile assertion
```ruby
# ❌ Auto-wait yok
expect(all(".item").size).to eq 3

# ✅
expect(page).to have_selector(".item", count: 3)
```

### 4. `visit` sonrası anlık assertion
```ruby
# ❌ Sayfa yüklenmemiş olabilir
visit "/dashboard"
expect(page.current_path).to eq "/dashboard"

# ✅
expect(page).to have_current_path("/dashboard")
```

### 5. Çoklu element için scope yok
```ruby
# ❌ Sayfada 5 "Edit" varsa Ambiguous
click_link "Edit"

# ✅
within "#user-123" do
  click_link "Edit"
end
```

### 6. `execute_script` ile click
```ruby
# ❌ Gerçek user davranışı değil, event listener'ları tetiklemez
page.execute_script "document.querySelector('#submit').click()"

# ✅
click_button "Submit"
```

### 7. Match strategy yanlış kullanımı
```ruby
# ❌ Silent failure — yanlış element seçilebilir
find ".item", match: :first

# ✅ Spesifik locator
find "#item-42"
within(".list") { find ".item" }
```

### 8. Cached element referansı
```ruby
# ❌ Stale exception riski
button = find(".btn")
visit "/refresh"
button.click

# ✅ Locator constant
BTN = ".btn".freeze
visit "/refresh"
find(BTN).click
```

### 9. Driver specific API portability bozma
```ruby
# ❌ Sadece Selenium'da çalışır
page.driver.browser.manage.window.maximize

# ✅ Capybara API (driver-agnostic)
page.current_window.maximize
```

### 10. `visible: false` antipattern
```ruby
# ❌ Hidden element click yapmak kullanıcı davranışı değil
find(".hidden-btn", visible: false).click

# ✅ Önce visible yap, sonra click
find(".trigger").hover         # menu open
find(".sub-btn").click
```

### 11. `within` yerine deep nested locator
```ruby
# ❌ Okunamaz, fragile
find("table#users tr td a.edit")

# ✅
within "table#users" do
  within "tr", text: "John" do
    click_link "Edit"
  end
end
```

### 12. `default_max_wait_time`'ı global yükseltmek
```ruby
# ❌ Tüm test suite yavaşlar
Capybara.default_max_wait_time = 30

# ✅ Spesifik sorunlu yerde wait artır
find(".slow-modal", wait: 30)
```

### 13. JavaScript driver gereksiz kullanımı
```ruby
# ❌ Server-side render için gereksiz Selenium
it "displays static page", js: true do
  visit "/about"
  expect(page).to have_content("About us")
end

# ✅ Rack::Test 10x hızlı
it "displays static page" do
  visit "/about"
  expect(page).to have_content("About us")
end
```

### 14. Cookie/session manuel yönetim
```ruby
# ❌ Capybara.reset_sessions! sonrası driver-specific cookie set
Capybara.reset_sessions!
page.driver.browser.manage.add_cookie(...)   # bir sonraki test'e sızar mı?

# ✅ Helper ile encapsulate
def with_user_session(user, &block)
  Capybara.using_session(user.email) do
    sign_in_as(user)
    yield
  end
end
```

### 15. CI'da `--headless=new` flag eksik
Chrome 109+ "new headless" çok daha güvenilir; "old headless" deprecated.
```ruby
options.add_argument("--headless=new")    # ✅
options.add_argument("--headless")        # ⚠️ deprecated, eski headless mode
```

### 16. `assert_*` ile `expect` karıştırma (Minitest/RSpec)
```ruby
# RSpec'te
expect(page).to have_content("X")        # ✅ Capybara matcher
assert_text "X"                          # ⚠️ Minitest helper, RSpec'te de var ama tutarsız
```

### 17. iframe scope unutmak
```ruby
# ❌ iframe içindeki element'i parent'tan ararsan bulamazsın
visit "/checkout"
fill_in "Card Number", with: "4111..."   # iframe içinde → ElementNotFound

# ✅
within_frame "stripe_iframe" do
  fill_in "Card Number", with: "4111..."
end
```

### 18. Local'de çalışıp CI'da fail
Tipik nedenler:
- Headless'ta animations farklı
- CI server slower → wait süresi yetmez
- Different DB state
- Locale farkı (`tr_TR` vs `en_US`)

Çözüm: `Capybara.default_max_wait_time = 10` CI'da; environment-specific config.

---

## 36. Framework Tasarımı

### Kapsamlı klasör yapısı
```
.
├── Gemfile
├── Rakefile
├── .rspec
├── config/
│   ├── application.rb
│   └── database.yml
├── spec/
│   ├── rails_helper.rb              # Rails entry
│   ├── spec_helper.rb               # Saf RSpec
│   ├── system/                      # Capybara test'leri
│   │   ├── authentication/
│   │   │   ├── login_spec.rb
│   │   │   ├── logout_spec.rb
│   │   │   └── password_reset_spec.rb
│   │   ├── checkout/
│   │   │   ├── cart_spec.rb
│   │   │   └── payment_spec.rb
│   │   └── admin/
│   │       └── user_management_spec.rb
│   ├── support/
│   │   ├── capybara.rb              # Capybara configuration
│   │   ├── drivers/
│   │   │   ├── chrome.rb
│   │   │   ├── cuprite.rb
│   │   │   └── lambdatest.rb
│   │   ├── helpers/
│   │   │   ├── authentication.rb
│   │   │   ├── navigation.rb
│   │   │   ├── data_helpers.rb
│   │   │   ├── wait_helpers.rb
│   │   │   └── screenshot.rb
│   │   ├── pages/                   # Page Object'ler
│   │   │   ├── base_page.rb
│   │   │   ├── login_page.rb
│   │   │   ├── dashboard_page.rb
│   │   │   ├── checkout_page.rb
│   │   │   └── components/
│   │   │       ├── header.rb
│   │   │       ├── sidebar.rb
│   │   │       └── footer.rb
│   │   ├── matchers/
│   │   │   └── custom_matchers.rb
│   │   ├── shared_contexts/
│   │   │   └── authenticated_user.rb
│   │   ├── shared_examples/
│   │   │   └── requires_login.rb
│   │   ├── vcr.rb
│   │   ├── database_cleaner.rb
│   │   └── allure.rb
│   ├── factories/
│   │   ├── users.rb
│   │   └── products.rb
│   └── fixtures/
│       └── files/
│           ├── avatar.png
│           └── document.pdf
└── tmp/
    ├── capybara/                    # screenshot, HTML
    ├── allure-results/
    └── screenshots/
```

### Custom helper örneği
```ruby
# spec/support/helpers/authentication.rb
module AuthenticationHelper
  def sign_in_as(user_or_email, password: "password123")
    user = case user_or_email
           when User then user_or_email
           when String then User.find_by(email: user_or_email)
           when Symbol then create(:user, user_or_email)
           end

    visit "/login"
    fill_in "Email",    with: user.email
    fill_in "Password", with: password
    click_button "Sign in"

    expect(page).to have_content("Signed in")
    user
  end

  def sign_out
    click_link "Logout"
    expect(page).to have_current_path("/")
  end
end

RSpec.configure do |c|
  c.include AuthenticationHelper, type: :system
end
```

### Shared context
```ruby
# spec/support/shared_contexts/authenticated_user.rb
RSpec.shared_context "authenticated user" do
  let(:current_user) { create(:user) }

  before do
    sign_in_as(current_user)
  end
end

# Test
RSpec.describe "Dashboard", type: :system do
  include_context "authenticated user"

  it "shows dashboard" do
    visit "/dashboard"
    expect(page).to have_content("Welcome, #{current_user.name}")
  end
end
```

### Shared examples
```ruby
# spec/support/shared_examples/requires_login.rb
RSpec.shared_examples "requires login" do |path|
  it "redirects to login" do
    visit path
    expect(page).to have_current_path("/login")
    expect(page).to have_content("Please sign in")
  end
end

# Test
RSpec.describe "Admin", type: :system do
  it_behaves_like "requires login", "/admin"
end
```

### Tag'leme & filtreleme
```ruby
RSpec.describe "Heavy E2E", :slow, :critical, type: :system do
  # ...
end
```

```bash
rspec --tag slow                    # sadece slow
rspec --tag ~slow                   # slow olmayanlar (PR build)
rspec --tag critical                # critical path (smoke)
rspec --tag "critical and ~slow"    # critical ama hızlı
```

### Custom RSpec configuration
```ruby
# .rspec
--require spec_helper
--format documentation
--format RspecJunitFormatter --out tmp/junit.xml
--color
--order random
--profile 10
```

`--profile 10` → en yavaş 10 test'i listeler (optimization fırsatı).

---

## 37. QA Lead Seviyesi — Mimari Kararlar

### Driver seçim stratejisi
| Senaryo | Önerilen | Gerekçe |
|---|---|---|
| Default test suite | `:rack_test` | Hız |
| JS-heavy / SPA | `:cuprite` | Selenium'dan 2x hızlı |
| Cross-browser smoke | `:selenium_remote` (cloud) | BrowserStack/LambdaTest |
| Lokal debug | `:selenium_chrome` (headful) | Görsel inspect |

### Test piramidi
```
                Capybara (E2E)        %10
              ─────────────────
           RSpec request specs        %20
              (API integration)
         ─────────────────────
       Unit (Model, Service, Helper)  %70
     ─────────────────────────────
```

**Lead'in kuralı:** Bir feature için **en az 1 unit test, opsiyonel API test, sadece kritik flow için Capybara E2E**. Her detayı E2E ile test etmek anti-pattern.

### SLA tanımları
| Test türü | Süre limit | Flaky rate |
|---|---|---|
| Unit test (suite) | < 1 dk | < %0.1 |
| Request spec (suite) | < 5 dk | < %0.5 |
| Capybara smoke | < 10 dk | < %1 |
| Capybara regression (full) | < 45 dk | < %1 |

### Maintenance stratejisi
- **Flake rate < %1** hedef; üstündekiler quarantine
- **Test execution time monitoring** — `--profile 10` yavaş test'leri yakalar
- **Locator audit** — `data-testid`'siz test'leri flag'le
- **Coverage review** — critical journey'leri belge halinde tut
- **POM ownership** — CODEOWNERS ile feature team'lere ata

### Test data stratejisi
1. **FactoryBot factory'leri** — küçük, isolated, idempotent
2. **DatabaseCleaner truncation** — Selenium için
3. **VCR cassette'leri** — external API stub
4. **`before(:suite)` seeds** — kalıcı reference data (countries, currencies)
5. **Test user pool** — login için cached fixture'lar

### Cloud testing kararı
- **Smoke test** (her PR): Headless Chrome lokal
- **Regression** (nightly): Headless Chrome lokal
- **Cross-browser** (release-prep): BrowserStack / LambdaTest
- **Production smoke** (her deploy): Cloud provider

### Team scaling
- **Onboarding doc**: 1 hafta içinde junior ilk PR'ını açabilmeli
  - CONTRIBUTING.md + örnek test
  - Local setup guide
  - POM convention
  - Common helper'lar listesi
- **Code review checklist**:
  - [ ] Locator strategy (data-testid > label > id)
  - [ ] Auto-wait kullanımı (sleep yok)
  - [ ] `have_no_*` kullanımı
  - [ ] Within scope ambiguity için
  - [ ] POM uyumu
  - [ ] Test data izolasyonu
- **Linter**: `rubocop-capybara`, `rubocop-rspec`
- **Pair testing**: senior + junior çift mülakat
- **Glossary**: `docs/testing.md` — terimlerin team-specific anlamı

### Risk yönetimi
- **Capybara upgrade**: Major version'lar nadiren breaking; minor'da test
- **Driver upgrade** (Chrome/Selenium): Auto-update CI'da break edebilir → pinned version
- **Rails upgrade**: System test API'sinde değişiklik olabilir
- **Browser updates**: CI image'larında pinned Chrome version

### Living documentation
Test'lerin kendi belge olması:
- Critical user journey'leri **adım adım** Capybara test'i olarak yaz
- Test isimleri user-facing senaryolar olmalı
- New developer test'leri okuyarak feature'ı anlayabilmeli

---

## 38. CI/CD Entegrasyonu

### GitHub Actions — kapsamlı örnek
```yaml
name: Test Suite
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        ci_node_total: [4]
        ci_node_index: [0, 1, 2, 3]

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: myapp_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports: ["6379:6379"]

    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/myapp_test
      REDIS_URL: redis://localhost:6379/0

    steps:
      - uses: actions/checkout@v4

      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Install Chrome
        uses: browser-actions/setup-chrome@v1
        with:
          chrome-version: stable

      - name: Setup database
        run: bundle exec rake db:schema:load

      - name: Precompile assets
        run: bundle exec rake assets:precompile

      - name: Run tests (sharded)
        env:
          KNAPSACK_PRO_CI_NODE_TOTAL: ${{ matrix.ci_node_total }}
          KNAPSACK_PRO_CI_NODE_INDEX: ${{ matrix.ci_node_index }}
          KNAPSACK_PRO_TEST_SUITE_TOKEN_RSPEC: ${{ secrets.KNAPSACK_RSPEC }}
        run: bundle exec rake knapsack_pro:rspec

      - name: Upload screenshots
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: failures-${{ matrix.ci_node_index }}
          path: |
            tmp/capybara/
            tmp/screenshots/

      - name: Upload JUnit report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: junit-${{ matrix.ci_node_index }}
          path: tmp/junit.xml

      - name: Publish JUnit
        uses: mikepenz/action-junit-report@v4
        if: always()
        with:
          report_paths: tmp/junit.xml
```

### CircleCI örneği
```yaml
version: 2.1

orbs:
  ruby: circleci/ruby@2.0
  browser-tools: circleci/browser-tools@1.4

jobs:
  test:
    docker:
      - image: cimg/ruby:3.2-browsers
      - image: cimg/postgres:15.0
        environment:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
    parallelism: 4
    steps:
      - checkout
      - ruby/install-deps
      - browser-tools/install-chrome
      - run: bundle exec rake db:schema:load
      - run:
          name: Run tests
          command: |
            TESTFILES=$(circleci tests glob "spec/**/*_spec.rb" | circleci tests split --split-by=timings)
            bundle exec rspec $TESTFILES --format RspecJunitFormatter --out tmp/junit.xml
      - store_test_results:
          path: tmp/junit.xml
      - store_artifacts:
          path: tmp/capybara
```

### Jenkins pipeline (declarative)
```groovy
pipeline {
    agent any

    environment {
        RAILS_ENV = 'test'
    }

    stages {
        stage('Setup') {
            steps {
                sh 'bundle install --jobs 4'
                sh 'bundle exec rake db:schema:load'
            }
        }

        stage('Test') {
            steps {
                sh 'bundle exec rspec --format RspecJunitFormatter --out tmp/junit.xml'
            }
        }
    }

    post {
        always {
            junit 'tmp/junit.xml'
            archiveArtifacts artifacts: 'tmp/capybara/**', allowEmptyArchive: true
        }
        failure {
            slackSend(color: 'danger', message: "Tests failed: ${env.BUILD_URL}")
        }
    }
}
```

### Parallel test stratejisi
```ruby
# Gemfile
gem "parallel_tests"
gem "knapsack_pro"
```

**parallel_tests** (lokal/single-machine paralel):
```bash
bundle exec parallel_rspec spec/system -n 4
```

**Knapsack Pro** (CI multi-node):
- Test'leri history-based time'a göre optimal split
- Node'lar arası dengeli yük

### Headless Chrome optimizasyonu CI'da
```ruby
Capybara.register_driver :ci_chrome do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless=new")
  options.add_argument("--no-sandbox")
  options.add_argument("--disable-gpu")
  options.add_argument("--disable-dev-shm-usage")     # /dev/shm küçük Docker'da
  options.add_argument("--window-size=1400,1400")
  options.add_argument("--disable-extensions")
  options.add_argument("--disable-popup-blocking")
  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end
```

---

## 39. En Sık Mülakat Soruları

| Soru | Hızlı Cevap |
|---|---|
| Capybara nedir? | Ruby acceptance/E2E test DSL'i; auto-wait + driver-agnostic |
| `page` objesi nedir? | `Capybara::Session` instance'ı, mevcut browser session handle'ı |
| `page` ile POM farkı? | `page` = framework session; POM = senin yazdığın sınıflar |
| Driver-agnostic nedir? | Aynı test, farklı driver'da koşar (rack_test, selenium, cuprite) |
| Driver tipleri? | rack_test, selenium_chrome (headless/headful), cuprite, apparition (deprecated) |
| Rack::Test vs Selenium farkı? | Rack::Test in-process Rack HTTP simulation, JS yok; Selenium gerçek browser |
| `find` vs `all`? | `find` tek+auto-wait+exception; `all` çoklu+wait yok+boş Result |
| `first` ne yapar? | İlk eşleşen element, auto-wait'li |
| Auto-wait nasıl çalışır? | Matcher içinde `synchronize` bloğu polling yapar (~50ms her seferinde) |
| `default_max_wait_time`? | Polling timeout; default 2sn; `wait:` ile override |
| `have_no_*` vs `not_to have_*`? | İlki auto-wait'li doğru, ikincisi race condition |
| `sleep` neden kötü? | Her zaman bekler; Capybara matcher gelirse hemen devam eder |
| Match strategy'leri? | :smart (default), :first, :one, :prefer_exact |
| Ambiguous exception? | Birden fazla eşleşme + match: :smart/:one |
| `within` neden önemli? | Scoping; ambiguity engeller, intent reveal |
| Multi-tab nasıl? | `window_opened_by { ... }`, `within_window`, `switch_to_window` |
| Multi-session? | `Capybara.using_session(:name) do ... end` |
| `Capybara.app` nedir? | Test edilen Rack application (Rails app vb.) |
| `Capybara.server` nedir? | Selenium/Cuprite için embedded HTTP server (Puma) |
| `run_server = false`? | External app testing (staging, production smoke) |
| `app_host`? | External URL — staging environment için |
| Locator önceliği? | data-testid > label > id > CSS class > XPath |
| XPath neden kırılgan? | Hierarchy'ye duyarlı, yavaş, küçük değişimde kırılır |
| Built-in selector type'lar? | :css, :xpath, :button, :link, :field, :label, :select, :option, :table |
| Custom selector? | `Capybara.add_selector(:name) { css { |arg| ... } }` |
| `Capybara.test_id`? | data-testid built-in support; tüm finder'lar bu attribute'u arar |
| `fill_in` syntax? | `fill_in "Email", with: "x@y.com"` — `with:` keyword |
| `fill_in` locator çözümleme? | label → id → name → placeholder |
| `find_field.set` ile `fill_in` farkı? | `set` low-level, JS event'leri kaçırabilir; `fill_in` user-like |
| File upload? | `attach_file "Avatar", "path/to/file"` |
| Hidden file input? | `attach_file ..., make_visible: true` |
| Alert/Confirm/Prompt? | `accept_alert { ... }`, `dismiss_confirm`, `accept_prompt(with: "...")` |
| JS execute? | `page.execute_script("...")`, `evaluate_script` (return value) |
| Cookie management? | Driver-specific (Rack::Test: `set_cookie`, Selenium: `browser.manage.add_cookie`) |
| Cross-browser test? | Selenium remote + cloud (BrowserStack, LambdaTest, Sauce) |
| Cloud provider seçimi? | BrowserStack enterprise, LambdaTest startup, Sauce DevOps |
| Paralel test? | parallel_tests gem, `server_port = 0`, per-process DB |
| Knapsack Pro? | CI node'ları arası optimal test split (time-based) |
| `Capybara.disable_animation`? | CSS animations skip — flaky test azaltır |
| Flaky test stratejisi? | sleep yerine matcher; not_to yerine have_no_*; data isolation; retry son çare |
| Test piramidi pozisyonu? | Capybara %10 (E2E), kritik flow'lar için |
| FactoryBot? | Test fixtures gem; factory + trait + sequence pattern |
| DatabaseCleaner? | Selenium için truncation strategy; Rack::Test transaction yeterli |
| Visual regression? | Percy, Applitools — baseline'a göre görsel diff |
| Accessibility? | axe-capybara, axe-rspec → WCAG kontrolleri |
| `save_and_open_page`? | HTML kaydet + default browser'da aç (debug) |
| `binding.pry`? | Test ortasında IRB aç, manuel debug |
| Element API method'ları? | `click`, `text`, `value`, `[attr]`, `visible?`, `disabled?`, `send_keys`, `hover` |
| Stale element nasıl önlenir? | Element ref cache etme; locator constant kullan |
| `native` method'u? | Driver-specific raw element; portability'i bozar |
| Capybara::DSL nedir? | `visit`, `click_*` global olarak `page` üstüne delegate eden module |
| iframe handling? | `within_frame "name" do ... end` (Rack::Test desteklemez) |
| Capybara::Result nedir? | `all` döndürdüğü Array-like wrapper, filter destekler |
| Capybara::Window? | Açık tarayıcı sekmesi/penceresi |
| Lead'in en önemli kararı? | Test piramidi: hangi case E2E'de, hangisi unit/integration'da |

---

## 40. Çalışma Yol Haritası

### 1. Hafta — Temeller
- [ ] Capybara install + ilk test
- [ ] `visit`, `click_*`, `fill_in`, `find`, `all` — temel DSL
- [ ] Driver kavramı — rack_test ile çalışma
- [ ] **Auto-wait** mekanizmasını içselleştir
- [ ] `have_content`, `have_selector` matcher'ları
- [ ] Basit bir Sinatra/Rails app'i kur ve 10 test yaz

### 2. Hafta — Orta seviye
- [ ] **`have_no_*` vs `not_to have_*`** farkını pratikte gör
- [ ] `within`, `within_frame`, `within_window`
- [ ] Match strategies (`:smart`, `:first`, `:one`)
- [ ] Multi-session, multi-window
- [ ] Element API derinlemesine (`click`, `send_keys`, `hover`, `drag_to`)
- [ ] Custom selector tanımla
- [ ] Auto-wait `synchronize` source code'unu oku

### 3. Hafta — Driver & framework
- [ ] **Selenium Chrome** kurulumu (headed + headless)
- [ ] **Cuprite** kurulumu, hız karşılaştırması
- [ ] Driver registration özelleştirme
- [ ] JavaScript interaction (`execute_script`, alerts)
- [ ] **Page Object Model** — BasePage + 3 sayfa
- [ ] Component pattern uygula
- [ ] Fluent API ile method chaining
- [ ] Lazy locator pattern

### 4. Hafta — Lead seviyesi
- [ ] **FactoryBot + DatabaseCleaner** entegrasyonu
- [ ] **VCR + WebMock** external API stub
- [ ] **`Capybara.disable_animation = true`**
- [ ] Tag'leme (`:smoke`, `:critical`, `:slow`) ile filtreleme
- [ ] GitHub Actions workflow + JUnit upload
- [ ] **`parallel_tests`** ile paralel test
- [ ] **Cloud provider** (LambdaTest/BrowserStack) entegrasyonu
- [ ] **Allure** raporlama
- [ ] **Flake rate dashboard**
- [ ] **Code review checklist** yaz
- [ ] **Test piramidi analizi** dokümante et

### Bonus — derin konular
- [ ] **Visual regression** (Percy veya Applitools)
- [ ] **Accessibility** (axe-capybara)
- [ ] **`Capybara.threadsafe = true`** ile multi-threaded
- [ ] Custom selector + filter + expression DSL
- [ ] Server-side error raise behavior
- [ ] External app testing (`run_server = false`)
- [ ] Headers/cookies driver-specific API

---

## 41. Kaynaklar

### Resmi
- **Ana sayfa:** github.com/teamcapybara/capybara
- **API docs:** rubydoc.info/github/teamcapybara/capybara
- **Wiki:** github.com/teamcapybara/capybara/wiki

### Driver'lar
- **Cuprite:** github.com/rubycdp/cuprite
- **Selenium WebDriver Ruby:** github.com/SeleniumHQ/selenium/tree/trunk/rb
- **Apparition (deprecated):** github.com/twalpole/apparition

### POM ve framework
- **SitePrism:** github.com/site-prism/site_prism
- **FactoryBot:** github.com/thoughtbot/factory_bot
- **DatabaseCleaner:** github.com/DatabaseCleaner/database_cleaner

### Reporting
- **allure-rspec:** github.com/allure-framework/allure-ruby
- **rspec_junit_formatter:** github.com/sj26/rspec_junit_formatter
- **capybara-screenshot:** github.com/mattheworiordan/capybara-screenshot

### Visual & Accessibility
- **Percy:** percy.io
- **Applitools:** applitools.com
- **axe-capybara:** github.com/dequelabs/axe-core-gems

### Cloud test providers
- **BrowserStack:** browserstack.com
- **LambdaTest:** lambdatest.com
- **Sauce Labs:** saucelabs.com

### Parallel & optimization
- **parallel_tests:** github.com/grosser/parallel_tests
- **Knapsack Pro:** knapsackpro.com

### Linter
- **rubocop-capybara:** github.com/rubocop/rubocop-capybara
- **rubocop-rspec:** github.com/rubocop/rubocop-rspec

### Kitaplar
- **"Effective Testing with RSpec 3"** — Myron Marston, Ian Dees
- **"Rails 7 Test Prescriptions"** — Noel Rappin
- **"Everyday Rails Testing with RSpec"** — Aaron Sumner

### Blog
- **Thoughtbot:** thoughtbot.com/blog (Capybara best practices)
- **Evil Martians:** evilmartians.com/chronicles (Cuprite, modern testing)

### Topluluk
- **Stack Overflow tag:** `capybara`
- **Rails forum:** discuss.rubyonrails.org
- **Reddit:** r/ruby, r/rails
