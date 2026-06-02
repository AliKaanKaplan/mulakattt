# Capybara — QA Lead Mülakat Hazırlık Rehberi

> Ruby ekosisteminde web UI / acceptance test'lerin standart DSL'i. Bu doküman QA Lead seviyesinde mülakata hazırlık için yazılmıştır.

---

## 1. Capybara Nedir?

Capybara, Ruby ile yazılmış bir **acceptance / end-to-end test DSL'idir** (Domain Specific Language). Gerçek bir kullanıcının web uygulamasıyla nasıl etkileşim kuracağını simüle ederek **E2E testler** yazmayı kolaylaştırır.

**Temel özellikleri:**
- Yüksek seviyeli, okunabilir API (`visit`, `click_on`, `fill_in`, `have_content`)
- **Built-in auto-wait** — asenkron elementleri otomatik bekler, flaky test'leri azaltır
- **Driver-agnostic** — Selenium, Rack::Test, Cuprite, Apparition gibi farklı sürücülerle çalışır
- RSpec, Cucumber, Minitest ile entegre olur
- Özellikle **Rails projelerinin** standart UI test katmanıdır

**Geliştirici:** Jonas Nicklas
**Lisans:** MIT
**GitHub:** github.com/teamcapybara/capybara

---

## 2. Kurulum ve Setup

### Gemfile
```ruby
group :test do
  gem "capybara"
  gem "selenium-webdriver"
  gem "rspec-rails"      # RSpec için
  gem "cuprite"          # Alternatif: hızlı CDP driver
end
```

### RSpec için (`spec/rails_helper.rb`)
```ruby
require "capybara/rspec"

Capybara.configure do |config|
  config.default_driver           = :rack_test
  config.javascript_driver        = :selenium_chrome_headless
  config.default_max_wait_time    = 5
  config.app_host                 = "http://localhost:3000"
  config.server                   = :puma, { Silent: true }
  config.default_normalize_ws     = true   # whitespace normalize
  config.save_path                = "tmp/capybara"
end

RSpec.configure do |config|
  config.include Capybara::DSL, type: :system
end
```

### Test örneği
```ruby
# spec/system/login_spec.rb
require "rails_helper"

RSpec.describe "Login", type: :system do
  it "kullanıcı giriş yapabilir" do
    visit "/login"
    fill_in "Email",    with: "test@example.com"
    fill_in "Password", with: "secret123"
    click_button "Sign in"

    expect(page).to have_content("Welcome back")
    expect(page).to have_current_path("/dashboard")
  end
end
```

---

## 3. Driver'lar

| Driver | Tip | JS Desteği | Hız | Kullanım Senaryosu |
|---|---|---|---|---|
| **`:rack_test`** | In-memory Rack | ❌ | ⚡⚡⚡ | Sunucu yok, en hızlı; sadece server-side render |
| **`:selenium_chrome`** | Gerçek Chrome | ✅ | 🐢 | Görsel debug, gerçek tarayıcı davranışı |
| **`:selenium_chrome_headless`** | Headless Chrome | ✅ | 🐢🐢 | CI'da varsayılan |
| **`:cuprite`** | CDP (Chrome DevTools) | ✅ | ⚡⚡ | Hızlı, Selenium'suz, modern |
| **`:apparition`** | CDP (deprecated) | ✅ | ⚡⚡ | Cuprite'ın atası, artık kullanılmaz |

**Driver değiştirme:**
```ruby
# Tek test için
it "JS ile çalışan akış", js: true do
  # otomatik javascript_driver kullanılır
end

# Bir blok için
Capybara.using_driver(:selenium_chrome_headless) do
  visit "/dashboard"
end
```

---

## 4. DSL — Temel Method'lar

### Navigation
```ruby
visit "/login"
visit users_url(id: 1)         # Rails URL helper
page.current_path              # => "/login"
page.current_url               # => "http://localhost:3000/login"
page.go_back
page.go_forward
page.refresh
```

### Tıklama
| Method | Hedef | Locator |
|---|---|---|
| `click_button` | `<button>`, `<input type=submit/button/image/reset>` | text, value, id, name, title |
| `click_link` | `<a href>` | text, id, title, alt |
| `click_on` | İkisi de (alias: `click_link_or_button`) | yukarıdakilerin hepsi |

```ruby
click_button "Submit"
click_link "Forgot password?"
click_on "Save"                # generic
```

### Form interaction
```ruby
fill_in "Email", with: "x@y.com"              # text, textarea, email, password, ...
choose "Male"                                  # radio button
check "I accept terms"                         # checkbox
uncheck "Subscribe to newsletter"
select "Turkey", from: "Country"               # dropdown
attach_file "Avatar", "spec/fixtures/me.png"
```

### Element bulma
```ruby
find("#submit")                          # tek; auto-wait, ElementNotFound atar
find(".item", match: :first)             # birden fazla varsa ilki
find("li", text: "Foo")
all(".item")                             # tüm; auto-wait YAPMAZ
all(".item", minimum: 3)                 # auto-wait için minimum/count
first(".item")                           # auto-wait'li ilk eşleşen
```

### Scoping
```ruby
within ".sidebar" do
  click_link "Logout"
end

within "form#new_user" do
  fill_in "Email", with: "x@y.com"
end

within_frame "iframe_id" do
  click_button "Submit"
end
```

### JavaScript & async
```ruby
page.execute_script "window.scrollTo(0, 200)"
page.evaluate_script "document.title"
page.accept_alert { click_button "Delete" }
page.dismiss_confirm { click_button "Cancel" }
page.accept_prompt(with: "John") { click_button "Name?" }
```

---

## 5. Auto-Wait & Synchronization

**Capybara'nın en kritik özelliği.** Element/koşul gerçekleşene kadar polling yapar.

```ruby
Capybara.default_max_wait_time = 5    # Default: 2 saniye
```

**Override yöntemleri:**
```ruby
# Tek call için
find("#dashboard", wait: 10)
expect(page).to have_content("Welcome", wait: 8)

# Blok için
using_wait_time(15) do
  find(".slow-modal")
  click_on "Confirm"
end

# Bekleme yok
find("#x", wait: 0)
```

**Polling vs Sleep — kritik fark:**
| | Davranış |
|---|---|
| `sleep 5` | **Her zaman** 5 sn bekler |
| `default_max_wait_time = 5` | **En fazla** 5 sn; gelirse hemen devam |

**Kural:** Capybara DSL içinde `sleep` kullanma. Auto-wait'i kullan.

---

## 6. Assertion'lar — `have_*` vs `have_no_*` (Klasik trap)

Capybara matcher'ları **kendi içlerinde** polling yapar. RSpec'in `not_to`'su bu polling'i tersine çevirmez.

| İfade | Davranış | Sonuç |
|---|---|---|
| `expect(page).to have_no_content("X")` | "X kaybolana kadar bekle" | ✅ Auto-wait doğru |
| `expect(page).not_to have_content("X")` | "X var mı? matcher poll'lar, sonra ters çevir" | ❌ Race condition |

**Her zaman negatif assertion için `have_no_*` kullan.**

```ruby
expect(page).to have_no_content("Error")
expect(page).to have_no_selector(".spinner")
expect(page).to have_no_button("Save", disabled: true)
```

**Diğer matcher'lar:**
```ruby
have_content "Hello"
have_text /Welcome.*/
have_selector "h1.title", count: 1
have_css ".btn-primary"
have_xpath "//div[@id='foo']"
have_current_path "/dashboard", ignore_query: true
have_link "Sign in", href: "/login"
have_button "Submit", disabled: false
have_field "Email", with: "x@y.com"
have_checked_field "Remember me"
have_select "Country", selected: "Turkey"
have_table "Users", with_rows: [["John", "Admin"]]
```

---

## 7. Locator Stratejileri

### Built-in selector tipleri
```ruby
find :css, ".btn"
find :xpath, "//button[contains(., 'OK')]"
find :id, "submit"
find :button, "Sign in"
find :link, "Forgot password?"
find :field, "Email"
find :label, "Email Address"
find :fillable_field, "Email"
find :select, "Country"
find :option, "Turkey"
find :table, "Users"
find :row, ["John", "Admin"]
```

### `Capybara.default_selector`
```ruby
Capybara.default_selector = :css        # default
Capybara.default_selector = :xpath      # tüm find çağrıları XPath olarak yorumlanır
```

### Custom selector tanımlama
```ruby
Capybara.add_selector(:data_testid) do
  css { |id| "[data-testid='#{id}']" }
end

# Kullanım
find :data_testid, "login-submit"
```

**Endüstri pratiği:** `data-testid` attribute'u kullanmak — CSS/refactor'dan etkilenmez, tasarım kararlarından bağımsızdır.

### Locator önceliği — best practice
1. **`data-testid`** (en sağlam, tasarım refactor'dan etkilenmez)
2. **Label text** (a11y'yi garanti eder, kullanıcı diline yakın)
3. **id** (kararlı ama tasarım değişebilir)
4. **CSS class** (kırılgan; tasarım değişince kırılır)
5. **XPath** (en kırılgan; son çare)

---

## 8. Page Object Model (POM)

Capybara'nın `page`'i ile karıştırılmamalı! POM senin yazdığın sınıflardır.

### Temel POM
```ruby
# spec/support/pages/base_page.rb
class BasePage
  include Capybara::DSL

  def loaded?
    raise NotImplementedError
  end
end

# spec/support/pages/login_page.rb
class LoginPage < BasePage
  URL = "/login".freeze

  def visit_page
    visit URL
    self
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
end

# Test
RSpec.describe "Login flow" do
  let(:login_page) { LoginPage.new }

  it "kullanıcı dashboard'a yönlendirilir" do
    dashboard = login_page.visit_page.login_as(
      email: "user@example.com",
      password: "secret"
    )
    expect(dashboard).to be_loaded
  end
end
```

### Method chaining (Fluent Interface)
Her action method bir sonraki page'i (veya self'i) dönmeli. Bu, test akışını okunabilir kılar.

### Component pattern (büyük sayfalar için)
```ruby
class DashboardPage < BasePage
  def sidebar
    SidebarComponent.new(find(".sidebar"))
  end

  def header
    HeaderComponent.new(find("header"))
  end
end

class SidebarComponent
  def initialize(node)
    @node = node
  end

  def logout
    @node.click_link "Logout"
    LoginPage.new
  end
end
```

---

## 9. Konfigürasyon Detayları

```ruby
Capybara.configure do |config|
  # Sunucu
  config.server              = :puma, { Silent: true }
  config.server_host         = "127.0.0.1"
  config.server_port         = 0           # rastgele port
  config.app_host            = nil         # external app için
  config.always_include_port = true

  # Driver'lar
  config.default_driver      = :rack_test
  config.javascript_driver   = :selenium_chrome_headless

  # Bekleme
  config.default_max_wait_time = 5
  config.match                 = :smart   # :first, :one, :prefer_exact
  config.exact                 = false    # true → tüm matcher'lar exact match

  # Davranış
  config.default_set_options   = {}
  config.automatic_label_click = false
  config.enable_aria_label     = true
  config.enable_aria_role      = false
  config.test_id               = "data-testid"   # built-in test id support

  # Debug
  config.save_path                  = "tmp/capybara"
  config.save_and_open_page_path    = "tmp/capybara"
  config.raise_server_errors        = true
end
```

**`test_id` config'i (Capybara 3.20+):**
```ruby
Capybara.test_id = "data-testid"

# Artık tüm finder'lar bu attribute'u da arar:
find_button "submit"   # text="submit" VEYA data-testid="submit"
```

---

## 10. Multi-Session ve Multi-Window

### Multi-session (iki kullanıcı, aynı test)
```ruby
it "admin onaylar, user görür" do
  Capybara.using_session(:admin) do
    visit "/admin/posts"
    click_on "Approve"
  end

  Capybara.using_session(:user) do
    visit "/posts"
    expect(page).to have_content("Approved")
  end
end
```

### Multi-window
```ruby
new_window = window_opened_by { click_link "Open report" }
within_window(new_window) do
  expect(page).to have_content("PDF Report")
end

# Manuel
page.windows                   # [Window1, Window2]
page.current_window
page.switch_to_window(new_window)
new_window.close
```

**Driver constraint:** Multi-window sadece Selenium / Cuprite'da çalışır. **Rack::Test'te window kavramı yoktur.**

---

## 11. Sık Karşılaşılan Pitfall'lar

### 1. `sleep` kullanmak
```ruby
# ❌ Antipattern
click_button "Submit"
sleep 3
expect(page).to have_content("Saved")

# ✅ Doğru
click_button "Submit"
expect(page).to have_content("Saved")    # auto-wait yapar
```

### 2. `not_to have_*` kullanmak
```ruby
# ❌ Race condition
expect(page).not_to have_content("Loading...")

# ✅ Doğru
expect(page).to have_no_content("Loading...")
```

### 3. `all` ile assertion
```ruby
# ❌ Auto-wait yok, flaky
expect(all(".item").size).to eq 3

# ✅ Matcher kullan
expect(page).to have_selector(".item", count: 3)
```

### 4. `visit` sonrası anlık assertion
```ruby
# ❌ Sayfa belki yüklenmedi
visit "/dashboard"
expect(page.current_path).to eq "/dashboard"   # Capybara matcher değil!

# ✅ Auto-wait'li matcher
expect(page).to have_current_path("/dashboard")
```

### 5. Çoklu element için spesifik olmayan locator
```ruby
# ❌ Ambiguous
click_link "Edit"     # sayfada 5 tane "Edit" linki varsa Ambiguous fırlatır

# ✅ Scope ile
within("#user-123") { click_link "Edit" }
```

### 6. `execute_script` ile DOM manipule etmek
```ruby
# ❌ Kullanıcı davranışı değil
page.execute_script "document.querySelector('#submit').click()"

# ✅ Capybara DSL
click_button "Submit"
```

---

## 12. Flaky Test Stratejisi

### Tespit
- **Aynı test rastgele fail/pass** → flaky
- `rspec --seed N` ile reproduce et
- `rspec --bisect` ile bağımlı test'leri bul

### Sık nedenler ve çözümler
| Neden | Çözüm |
|---|---|
| `sleep` kullanımı | Capybara matcher'ları ile değiştir |
| `not_to have_*` | `have_no_*` |
| `all().size` assertion | `have_selector count:` |
| Test data leak | DatabaseCleaner ile transaction isolation |
| Yarış koşulu (race condition) | `wait:` opsiyonu ile bekle |
| External API timeout | WebMock + VCR + stub |
| Browser process leak | `Capybara.reset_sessions!`, `Capybara.use_default_driver` |

### Stratejik araçlar
- **DatabaseCleaner:** Test arası DB state temizleme
- **VCR / WebMock:** External HTTP stub'lama
- **Knapsack Pro:** Paralel test çalıştırma (CI'da)
- **RSpec retry:** Geçici flake'leri tolere etmek için (`config.verbose_retry = true`)

---

## 13. CI/CD Entegrasyonu

### GitHub Actions örneği
```yaml
name: Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s

    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Install Chrome
        uses: browser-actions/setup-chrome@v1

      - run: bundle exec rake db:setup
      - run: bundle exec rspec --format progress
```

### Headless ayarı
```ruby
Capybara.register_driver :selenium_chrome_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless=new")
  options.add_argument("--no-sandbox")
  options.add_argument("--disable-gpu")
  options.add_argument("--window-size=1400,1400")
  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end
```

### Paralel test
- **parallel_tests** gem'i → process-level paralel
- **Knapsack Pro** → CI node'lar arası split
- `Capybara.server_port = 0` → port collision'ı önler
- DB için her process'e ayrı DB (`TEST_ENV_NUMBER`)

---

## 14. Debugging

### Save & open
```ruby
save_and_open_page                          # HTML kaydet ve aç
save_and_open_screenshot                    # PNG screenshot
page.save_screenshot("tmp/debug.png")
puts page.html
puts page.driver.browser.manage.logs.get(:browser)   # browser console
```

### Pry / byebug
```ruby
it "debug" do
  visit "/login"
  binding.pry                # bu noktada session interactive olur
  fill_in "Email", with: "x"
end
```

### Capybara.save_path
```ruby
Capybara.save_path = "tmp/capybara"
# Failed test sonrası screenshot otomatik kaydedilir (capybara-screenshot gem)
```

### Selenium logs
```ruby
page.driver.browser.logs.get(:browser).each { |l| puts l.message }
```

---

## 15. Framework Tasarımı — QA Lead Bakışı

### Klasör yapısı
```
spec/
  system/                    # Capybara test'ler (system spec)
    login_spec.rb
    dashboard_spec.rb
  support/
    pages/                   # Page Object'ler
      base_page.rb
      login_page.rb
      dashboard_page.rb
    components/              # Reusable component'ler
      header.rb
      sidebar.rb
    helpers/
      authentication.rb
      data_factory.rb
    capybara.rb              # Capybara konfig
    drivers.rb               # Custom driver registration
  factories/                 # FactoryBot factory'leri
  fixtures/                  # Test fixture'lar (dosya yüklemeler, vb.)
  rails_helper.rb
```

### Custom helper'lar
```ruby
# spec/support/helpers/authentication.rb
module AuthenticationHelper
  def sign_in_as(user)
    visit "/login"
    fill_in "Email",    with: user.email
    fill_in "Password", with: "password123"
    click_button "Sign in"
    expect(page).to have_content("Signed in")
  end
end

RSpec.configure do |c|
  c.include AuthenticationHelper, type: :system
end
```

### Tag'leme ve filtreleme
```ruby
RSpec.describe "Heavy E2E", :slow, :browser do
  # ...
end

# Çalıştırma
rspec --tag slow                 # sadece slow tag'liler
rspec --tag ~slow                # slow olmayanlar (smoke için)
```

### Smoke vs Regression
- **Smoke suite:** ~5 dakika, her PR'da çalışır, critical path
- **Regression suite:** ~30+ dakika, nightly çalışır, tüm akışlar

### Test piramidi pozisyonu
- **Unit:** ~70%
- **Integration / API:** ~20%
- **E2E / Capybara:** ~10% (en pahalı, en kırılgan)

Capybara test'leri **yalnızca critical user flow'lar** için yaz. Her detayı E2E ile test etmek anti-pattern.

---

## 16. QA Lead Seviyesi — Mimari Kararlar

### Driver seçimi
| Senaryo | Önerilen Driver |
|---|---|
| Sunucu-side render, JS yok | `:rack_test` |
| SPA, JS heavy | `:cuprite` (hızlı) veya `:selenium_chrome_headless` |
| Cross-browser test | `:selenium_remote` (BrowserStack, LambdaTest) |
| Görsel debug | `:selenium_chrome` (headless değil) |

### Cloud testing entegrasyonu
- **BrowserStack, Sauce Labs, LambdaTest** → real device + cross-browser
- Selenium Grid üzerinden bağlantı:
```ruby
Capybara.register_driver :lambdatest do |app|
  caps = {
    "browserName" => "Chrome",
    "browserVersion" => "latest",
    "LT:Options" => {
      "platformName" => "Windows 11",
      "build" => ENV["BUILD_ID"],
      "username" => ENV["LT_USERNAME"],
      "accessKey" => ENV["LT_ACCESS_KEY"]
    }
  }
  Capybara::Selenium::Driver.new(app,
    browser: :remote,
    url: "https://#{ENV['LT_USERNAME']}:#{ENV['LT_ACCESS_KEY']}@hub.lambdatest.com/wd/hub",
    capabilities: caps
  )
end
```

### Reporting
- **RSpec formatter:** `--format documentation`, `--format json`, `--format html`
- **Allure:** `allure-rspec` gem'i → detaylı görsel raporlar, screenshot embedded
- **RSpec JUnit XML:** CI sistemleri için (`rspec_junit_formatter`)
- **Custom dashboard:** Test trend, flake rate, çalışma süresi metrikleri

### Maintenance stratejisi
- **Flake rate < %1** hedefi
- **Test execution time** monitör et — yavaşlayan test'leri refactor
- **Locator audit** — `data-testid`'siz test'leri flag'le
- **Coverage:** Critical user journey'leri belgele, her release öncesi smoke
- **Page Object'leri reusability'ye göre refactor** — 3+ test'te kullanılan akış POM'a alınmalı

### Team scaling
- **Naming convention** zorunlu (page object, helper, spec dosya isimleri)
- **Code review checklist:** locator strategy, auto-wait kullanımı, POM uyumu
- **Linter:** rubocop-capybara, rubocop-rspec
- **Onboarding:** 1 hafta içinde junior bir tester ilk PR'ını açabilmeli → CONTRIBUTING.md + örnek test
- **Pair testing:** Senior + junior çift mülakat → bilgi transferi

---

## 17. En Sık Mülakat Soruları (Hızlı Cevaplar)

| Soru | Hızlı Cevap |
|---|---|
| Capybara nedir? | Ruby acceptance/E2E test DSL'i; auto-wait + driver-agnostic |
| `page` objesi nedir? | `Capybara::Session` instance'ı, mevcut browser session handle'ı |
| `find` vs `all`? | `find` tek+auto-wait+exception; `all` çoklu+wait yok+boş Result |
| `default_max_wait_time`? | Polling timeout; default 2sn; `wait:` ile override |
| `have_no_*` vs `not_to have_*`? | İlki auto-wait'li doğru, ikincisi race condition |
| Driver seçimi? | Rack::Test (hızlı/JS yok), Selenium (gerçek browser), Cuprite (hızlı CDP) |
| Flaky test stratejisi? | `sleep` yerine matcher; `not_to` yerine `have_no_*`; DB cleanup; retry sadece son çare |
| POM ile `page`'in farkı? | `page` = framework session; POM = senin yazdığın sınıflar |
| `within` neden önemli? | Scoping; ambiguous selector'ı önler, intent reveal |
| Multi-tab nasıl? | `window_opened_by { ... }`, `within_window`, `switch_to_window` |
| Cross-browser? | Selenium remote + cloud (BrowserStack, LambdaTest) |
| Paralel test? | parallel_tests gem, port=0, per-process DB |
| Locator önceliği? | data-testid > label > id > CSS class > XPath |
| Custom selector? | `Capybara.add_selector(:name) { css { |arg| ... } }` |
| Auto-wait nasıl çalışır? | Matcher içinde `synchronize` bloğu polling yapar |

---

## 18. Çalışma Yol Haritası

### 1. Hafta — Temeller
- [ ] `visit`, `click_*`, `fill_in`, `find`, `all`
- [ ] `have_*` matcher'ları
- [ ] Konfigürasyon (`default_max_wait_time`, driver registration)
- [ ] Basit bir Sinatra/Rails app'i kur ve 10 test yaz

### 2. Hafta — Orta
- [ ] `within`, `within_frame`, `within_window`
- [ ] `have_no_*` vs `not_to have_*` farkını pratikte gör
- [ ] Multi-session, multi-window senaryoları
- [ ] Custom selector tanımla
- [ ] Auto-wait mekanizmasını `synchronize` source code'undan oku

### 3. Hafta — Page Object & Framework
- [ ] BasePage + 3 sayfa için POM yaz
- [ ] Component pattern uygula
- [ ] Fluent API ile method chaining yap
- [ ] Helper module'leri ayır
- [ ] Tag'leme (slow, smoke, regression) ile filtreleme

### 4. Hafta — CI/CD & Lead
- [ ] GitHub Actions workflow yaz
- [ ] Headless Chrome + paralel test'i kur
- [ ] Cloud provider (LambdaTest/BrowserStack) entegre et
- [ ] Allure/JUnit raporlama
- [ ] Flake rate dashboard'u kur
- [ ] Code review checklist'i yaz
- [ ] Test piramidi analizi: hangi case'ler E2E'ye girer?

---

## 19. Kaynaklar

- **Resmi:** github.com/teamcapybara/capybara
- **API:** rubydoc.info/github/teamcapybara/capybara
- **Cuprite:** github.com/rubycdp/cuprite
- **Kitap:** "Effective Testing with RSpec 3" — Myron Marston
- **Blog:** "thoughtbot.com/blog" — Capybara best practices
- **rubocop-capybara:** github.com/rubocop/rubocop-capybara
