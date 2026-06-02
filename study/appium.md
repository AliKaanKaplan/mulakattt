# Appium — QA Lead Mülakat Hazırlık Rehberi (Ruby)

> Mobile (iOS / Android) ve cross-platform UI test otomasyonu için endüstri standardı. Bu doküman QA Lead seviyesinde mülakata hazırlık içindir.
> **Tüm örnekler Ruby'dir** — `appium_lib_core` + RSpec ekosistemi kullanılmıştır.

---

## 1. Appium Nedir?

**Appium**, mobile uygulamalar için **cross-platform UI test otomasyonu** sağlayan açık kaynak bir framework'tür. Tek bir API ile **iOS, Android, Windows ve macOS** uygulamalarını test edebilirsin.

**Temel özellikleri:**
- **WebDriver protocol** üzerine kurulu (Selenium ile aynı protokol — W3C WebDriver standardı)
- **Native, hybrid ve web mobile** uygulamaları test eder
- Source code değişikliği gerektirmez (no instrumentation in production)
- **Dil-agnostic:** Java, Python, Ruby, JavaScript, C#, Robot Framework client'ları
- **Driver mimarisi:** Her platform için ayrı driver (UiAutomator2, XCUITest, Espresso, vb.)

**Ruby client'ları:**
- **`appium_lib_core`** — modern, hafif, sadece WebDriver komutları
- **`appium_lib`** — eski "full-fat" client, `appium_lib_core` üstüne helper'lar ekler (`text`, `button`, vb.). Yeni projelerde **`appium_lib_core`** önerilir.

**Versiyonlar:**
- **Appium 1.x:** Eski mimari, monolithic, deprecated.
- **Appium 2.x (current):** Driver/plugin tabanlı mimari, modüler. **Yeni projeler 2.x kullanmalı.**

---

## 2. Mimari

```
┌─────────────────────────────────────────────────────────────┐
│   Test Code (Ruby — appium_lib_core + RSpec)                │
│   ↓ WebDriver client kütüphanesi                            │
├─────────────────────────────────────────────────────────────┤
│   HTTP / JSON Wire Protocol (W3C WebDriver)                 │
│   ↓                                                         │
├─────────────────────────────────────────────────────────────┤
│   Appium Server (Node.js)                                   │
│   - Session yönetimi                                        │
│   - Command routing                                         │
│   ↓ uygun driver'ı seçer                                    │
├─────────────────────────────────────────────────────────────┤
│   Drivers (her platform için)                               │
│   - UiAutomator2 (Android native)                           │
│   - Espresso (Android native, in-process)                   │
│   - XCUITest (iOS native)                                   │
│   - Mac2 (macOS native)                                     │
│   - Windows (Windows native)                                │
│   ↓                                                         │
├─────────────────────────────────────────────────────────────┤
│   Platform's native automation framework                    │
│   ↓ uygulamayı kontrol eder                                 │
├─────────────────────────────────────────────────────────────┤
│   Mobile device / emulator / simulator                      │
└─────────────────────────────────────────────────────────────┘
```

**Kritik nokta:** Appium **bir proxy/server'dır**, otomatik uygulamayı **kendisi kontrol etmez** — platform'un native otomasyon framework'unu (UiAutomator2, XCUITest) sürer.

---

## 3. Kurulum (Appium 2.x + Ruby)

### Appium server
```bash
# Node.js gerekli (16+)
npm install -g appium

# Driver yükle
appium driver install uiautomator2       # Android
appium driver install xcuitest           # iOS
appium driver install espresso           # Android in-process

# Sunucuyu başlat
appium server --port 4723
```

### Ruby tarafı (Gemfile)
```ruby
# Gemfile
source "https://rubygems.org"

group :test do
  gem "appium_lib_core", "~> 9.0"
  gem "rspec"
  gem "rspec_junit_formatter"
  gem "allure-rspec"
  gem "parallel_tests"
  gem "pry-byebug"
end
```

```bash
bundle install
```

### Gerekli SDK'lar
| Platform | Gereksinim |
|---|---|
| **Android** | Android SDK, ADB, Java JDK 11+, emulator veya gerçek device |
| **iOS** | macOS, Xcode, Xcode Command Line Tools, simulator veya gerçek device + WebDriverAgent |

### Doctor — environment check
```bash
appium driver doctor uiautomator2
appium driver doctor xcuitest
```

### Inspector (locator keşfi için)
```bash
npm install -g appium-inspector
# veya GUI app olarak: github.com/appium/appium-inspector
```

---

## 4. Capabilities — Session Başlatma

Her test'te server'a hangi platform/cihaz/uygulama isteğinin yapıldığını söyleyen capability config'idir.

### Android örneği (Ruby)
```ruby
require "appium_lib_core"

caps = {
  caps: {
    platformName:            "Android",
    "appium:platformVersion": "13",
    "appium:deviceName":      "Pixel_6_API_33",
    "appium:app":             "/path/to/app.apk",
    "appium:automationName":  "UiAutomator2",
    "appium:appPackage":      "com.example.app",
    "appium:appActivity":     ".MainActivity",
    "appium:noReset":         false,
    "appium:fullReset":       false,
    "appium:newCommandTimeout": 120
  },
  appium_lib: {
    server_url: "http://127.0.0.1:4723"
  }
}

core   = Appium::Core.for(caps)
driver = core.start_driver
# ... test kodu ...
driver.quit
```

### iOS örneği (Ruby)
```ruby
caps = {
  caps: {
    platformName:               "iOS",
    "appium:platformVersion":    "17.0",
    "appium:deviceName":         "iPhone 15",
    "appium:app":                "/path/to/MyApp.app",
    "appium:automationName":     "XCUITest",
    "appium:bundleId":           "com.example.MyApp",
    "appium:udid":               "00008110-XXXX",   # gerçek device için
    "appium:xcodeOrgId":         "ABC1234567",
    "appium:xcodeSigningId":     "iPhone Developer"
  },
  appium_lib: {
    server_url: "http://127.0.0.1:4723"
  }
}

driver = Appium::Core.for(caps).start_driver
```

### Önemli capability'ler
| Capability | Açıklama |
|---|---|
| `platformName` | "Android" / "iOS" |
| `appium:platformVersion` | OS sürümü |
| `appium:deviceName` | Cihaz adı (emülatör/simülatör adı veya gerçek device) |
| `appium:automationName` | "UiAutomator2" / "XCUITest" / "Espresso" |
| `appium:app` | APK / .app / .ipa dosyasının absolute path'i |
| `appium:appPackage` + `appium:appActivity` | Android: yüklü uygulamayı başlatmak için |
| `appium:bundleId` | iOS: yüklü uygulamayı başlatmak için |
| `appium:udid` | Gerçek cihazın unique device ID'si |
| `appium:noReset` | true → session sonunda app data silinmez |
| `appium:fullReset` | true → app uninstall + reinstall (yavaş, izolasyon yüksek) |
| `appium:newCommandTimeout` | İdle session timeout (default 60sn) |
| `appium:autoGrantPermissions` | Android: runtime permission'ları auto-grant |
| `appium:autoAcceptAlerts` | iOS: alert'leri auto-accept |
| `appium:language` / `appium:locale` | i18n test için |

> **Not:** Appium 2.x'te capability isimleri **`appium:` prefix'i** ile yazılır. Sadece standart W3C capability'leri (`platformName`, `browserName`) prefix'sizdir.

---

## 5. Locator Stratejileri

Selenium'un `find_element` API'sı + Appium'un mobile özel locator'ları.

| Locator | Hız | Güvenilirlik | Kullanım |
|---|---|---|---|
| **accessibility_id** | ⚡⚡⚡ | ✅ En iyi | iOS: `accessibilityIdentifier`, Android: `content-desc` |
| **id** | ⚡⚡⚡ | ✅ | Android: `resource-id`, iOS: name attribute |
| **class name** | ⚡⚡ | ⚠️ | UI widget class (`android.widget.Button`) |
| **xpath** | 🐢 | ⚠️ | En esnek, **en yavaş ve kırılgan** |
| **uiautomator** | ⚡⚡ | ✅ | Android-specific powerful selector |
| **predicate** | ⚡⚡ | ✅ | iOS-specific powerful selector |
| **class chain** | ⚡⚡ | ✅ | iOS XPath benzeri ama hızlı |

### Örnekler (Ruby)
```ruby
# Accessibility ID — önerilen
driver.find_element(:accessibility_id, "login_button")

# ID
driver.find_element(:id, "com.example:id/email")

# Android UiAutomator
driver.find_element(
  :uiautomator,
  'new UiSelector().textContains("Welcome")'
)

driver.find_element(
  :uiautomator,
  'new UiScrollable(new UiSelector().scrollable(true))' \
  '.scrollIntoView(new UiSelector().text("Settings"))'
)

# iOS Predicate string
driver.find_element(
  :predicate,
  "type == 'XCUIElementTypeButton' AND name CONTAINS 'Login'"
)

# iOS Class Chain
driver.find_element(
  :class_chain,
  '**/XCUIElementTypeButton[`name == "Login"`]'
)

# XPath — son çare
driver.find_element(:xpath, "//android.widget.Button[@text='Login']")

# Birden fazla element
buttons = driver.find_elements(:class, "android.widget.Button")
```

**Önerilen sıra:** accessibility_id → id → uiautomator/predicate → xpath (son çare).

**XPath neden kırılgan?**
- DOM-benzeri tree'yi tarar → **çok yavaş** (büyük uygulamalarda 5-10 sn)
- UI hierarchy'sindeki küçük değişiklikler test'i kırar
- Index-tabanlı XPath (`//Button[3]`) en kırılgan

---

## 6. Sık Kullanılan Komutlar (Ruby)

### Element interaction
```ruby
el = driver.find_element(:accessibility_id, "login_btn")
el.click
el.send_keys("text input")
el.clear
el.text
el.displayed?
el.enabled?
el.attribute("content-desc")
```

### Touch gestures (Appium 2.x — W3C Actions)
```ruby
# Selenium WebDriver 4 actions API (Ruby)
driver.action
      .move_to_location(100, 200)
      .pointer_down(:left)
      .pointer_up(:left)
      .perform
```

**Daha pratik — `execute_script` ile mobile gesture extensions:**
```ruby
# Tap
driver.execute_script("mobile: tap", { x: 100, y: 200 })

# Swipe (Android UiAutomator2)
driver.execute_script("mobile: swipeGesture", {
  left: 100, top: 500, width: 200, height: 800,
  direction: "up", percent: 0.75
})

# Long press
driver.execute_script("mobile: longClickGesture", {
  elementId: el.ref.last,    # element id
  duration: 2000
})

# Scroll
driver.execute_script("mobile: scrollGesture", {
  left: 100, top: 100, width: 800, height: 1200,
  direction: "down", percent: 1.0
})

# iOS swipe
driver.execute_script("mobile: swipe", {
  direction: "left", element: el.ref.last
})
```

### App management
```ruby
driver.terminate_app("com.example.app")           # app'i kapat
driver.activate_app("com.example.app")            # app'i öne getir
driver.install_app("/path/to/app.apk")
driver.remove_app("com.example.app")
driver.app_installed?("com.example.app")

driver.background_app(5)                           # 5 sn background'a al
```

### Device interaction
```ruby
# Android key event
driver.press_keycode(4)                            # AndroidKey.BACK = 4
driver.press_keycode(3)                            # AndroidKey.HOME = 3

# Orientation
driver.rotation = :landscape
driver.rotation = :portrait

# Screenshot
driver.save_screenshot("tmp/debug.png")
File.write("page.xml", driver.page_source)         # tüm UI hierarchy XML
```

### Context switching (hybrid app)
```ruby
contexts = driver.available_contexts
# => ["NATIVE_APP", "WEBVIEW_com.example"]

driver.set_context("WEBVIEW_com.example")          # WebView'a geç
driver.find_element(:css, "button#login").click

driver.set_context("NATIVE_APP")                   # Native'e geri dön
```

---

## 7. Wait Stratejileri (Ruby)

Selenium'unki ile aynı — Appium **kendi auto-wait sağlamaz** (Capybara'dan farklı olarak), explicit wait kullanmalısın.

### Implicit Wait (önerilmez — kötü pratik)
```ruby
driver.manage.timeouts.implicit_wait = 10
```
Implicit + explicit karışırsa beklenmedik davranış çıkar. **Sadece explicit kullan.**

### Explicit Wait — `Selenium::WebDriver::Wait`
```ruby
wait = Selenium::WebDriver::Wait.new(timeout: 10)

el = wait.until { driver.find_element(:accessibility_id, "login") }

# Element clickable mi (visible + enabled)
el = wait.until do
  e = driver.find_element(:accessibility_id, "submit")
  e if e.displayed? && e.enabled?
end

# Text bekle
wait.until { driver.find_element(:accessibility_id, "msg").text.include?("Welcome") }

# Element kayboldu mu?
wait.until do
  begin
    !driver.find_element(:accessibility_id, "spinner").displayed?
  rescue Selenium::WebDriver::Error::NoSuchElementError
    true
  end
end
```

### FluentWait pattern (custom polling, exception ignore)
```ruby
wait = Selenium::WebDriver::Wait.new(
  timeout:  30,
  interval: 0.5,
  ignore:   [Selenium::WebDriver::Error::NoSuchElementError,
             Selenium::WebDriver::Error::StaleElementReferenceError]
)

el = wait.until { driver.find_element(:accessibility_id, "dynamic_btn") }
```

### Helper modülü ile temizlik
```ruby
# spec/support/wait_helpers.rb
module WaitHelpers
  def wait_for(timeout: 10, interval: 0.5)
    Selenium::WebDriver::Wait
      .new(timeout: timeout, interval: interval)
      .until { yield }
  end

  def wait_visible(locator_type, locator_value, timeout: 10)
    wait_for(timeout: timeout) do
      el = driver.find_element(locator_type, locator_value)
      el if el.displayed?
    end
  end

  def wait_clickable(locator_type, locator_value, timeout: 10)
    wait_for(timeout: timeout) do
      el = driver.find_element(locator_type, locator_value)
      el if el.displayed? && el.enabled?
    end
  end
end

RSpec.configure { |c| c.include WaitHelpers }
```

### Kural
- **Implicit wait kullanma**
- Her tıklama/etkileşim öncesi **explicit wait** ile elementin clickable/visible olduğunu garanti et
- `sleep` **anti-pattern**'dir — sadece debug için, asla CI test'inde

---

## 8. Native vs Hybrid vs Web

| Tip | Açıklama | Test stratejisi |
|---|---|---|
| **Native** | Platform UI framework'ü ile yazılmış (Swift/Java/Kotlin) | UiAutomator2 / XCUITest driver, native locator'lar |
| **Hybrid** | WebView içinde HTML render eden native app (Cordova, Ionic) | Context switching (NATIVE_APP ↔ WEBVIEW), iki tarafı da test et |
| **Web (mobile browser)** | Mobile Chrome/Safari'de açılan responsive web | Selenium ile aynı; `browserName: Chrome/Safari` capability |

**Hybrid debug:** Chrome DevTools (Android) veya Safari Web Inspector (iOS) ile WebView'ı inspect edebilirsin.

---

## 9. Page Object Model (Mobile, Ruby)

```ruby
# spec/support/pages/base_page.rb
class BasePage
  attr_reader :driver, :wait

  def initialize(driver)
    @driver = driver
    @wait   = Selenium::WebDriver::Wait.new(timeout: 10, interval: 0.5)
  end

  protected

  def wait_clickable(type, value)
    wait.until do
      el = driver.find_element(type, value)
      el if el.displayed? && el.enabled?
    end
  end

  def wait_visible(type, value)
    wait.until do
      el = driver.find_element(type, value)
      el if el.displayed?
    end
  end
end

# spec/support/pages/login_page.rb
class LoginPage < BasePage
  EMAIL_FIELD    = [:accessibility_id, "email_input"].freeze
  PASSWORD_FIELD = [:accessibility_id, "password_input"].freeze
  LOGIN_BTN      = [:accessibility_id, "login_button"].freeze
  ERROR_MSG      = [:accessibility_id, "error_msg"].freeze

  def enter_email(email)
    wait_visible(*EMAIL_FIELD).send_keys(email)
    self
  end

  def enter_password(password)
    wait_visible(*PASSWORD_FIELD).send_keys(password)
    self
  end

  def submit
    wait_clickable(*LOGIN_BTN).click
    DashboardPage.new(driver)
  end

  def error_message
    wait_visible(*ERROR_MSG).text
  end
end

# spec/system/login_spec.rb
require "rails_helper"

RSpec.describe "Login flow", :appium do
  let(:driver)     { AppiumSession.driver }
  let(:login_page) { LoginPage.new(driver) }

  it "kullanıcı başarılı giriş yapar" do
    dashboard = login_page
                  .enter_email("test@example.com")
                  .enter_password("secret")
                  .submit

    expect(dashboard).to be_loaded
  end
end
```

### Cross-platform POM (Ruby idiom)
Java'daki `@AndroidFindBy` + `@iOSXCUITFindBy` annotation'larının Ruby karşılığı: **platform-aware lazy locator helper**.

```ruby
# spec/support/pages/base_page.rb
class BasePage
  PLATFORM = ENV.fetch("PLATFORM", "android").to_sym

  def self.locator(name, android:, ios:)
    define_method(name) do
      type, value = PLATFORM == :android ? android : ios
      wait_visible(type, value)
    end
  end
end

# spec/support/pages/login_page.rb
class LoginPage < BasePage
  locator :email_field,
          android: [:accessibility_id, "email_input"],
          ios:     [:accessibility_id, "email_input"]

  locator :login_btn,
          android: [:uiautomator, 'new UiSelector().text("Login")'],
          ios:     [:class_chain, '**/XCUIElementTypeButton[`name == "Login"`]']

  def login(email:, password:)
    email_field.send_keys(email)
    # ...
    login_btn.click
    DashboardPage.new(driver)
  end
end
```

`locator` class macro'su (DSL) iki platform için tek POM tanımına izin verir — Java annotation'larından daha esnek.

---

## 10. CI/CD Entegrasyonu

### Lokal Appium server + CI (GitHub Actions)
```yaml
jobs:
  android-test:
    runs-on: macos-latest        # KVM gereksin diye macos veya self-hosted
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: actions/setup-java@v4
        with: { java-version: '17', distribution: 'temurin' }
      - uses: actions/setup-node@v4
        with: { node-version: '20' }

      - name: Install Appium
        run: |
          npm install -g appium
          appium driver install uiautomator2

      - name: Start Android emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          target: google_apis
          arch: x86_64
          script: |
            appium server --port 4723 &
            bundle exec rspec spec/system --format RspecJunitFormatter --out tmp/rspec.xml
```

### Cloud device farms — Lead seviyesinde tercih
Lokal emulator/simulator yerine **cloud device farm**'lar tercih edilir.

| Provider | Özellik |
|---|---|
| **BrowserStack App Automate** | 3000+ gerçek device, geniş ekosistem |
| **Sauce Labs** | Mature, enterprise odaklı |
| **LambdaTest** | Uygun fiyat, hızlı, web + mobile |
| **AWS Device Farm** | AWS ekosisteminde |
| **Firebase Test Lab** | Sadece Android |

### LambdaTest capability örneği (Ruby)
```ruby
require "appium_lib_core"

caps = {
  caps: {
    platformName:               "Android",
    "appium:deviceName":         "Galaxy S22",
    "appium:platformVersion":    "13",
    "appium:app":                "lt://APP123456",   # LT'ye yüklenmiş app
    "lt:options" => {
      build:    "Release-2024-01",
      name:     "Login flow test",
      user:     ENV.fetch("LT_USERNAME"),
      accessKey: ENV.fetch("LT_ACCESS_KEY"),
      visual:   true,                                # visual regression
      network:  true,                                # network logs
      video:    true,
      console:  true
    }
  },
  appium_lib: {
    server_url: "https://mobile-hub.lambdatest.com/wd/hub"
  }
}

driver = Appium::Core.for(caps).start_driver
```

### Paralel test stratejisi (Ruby)
```bash
# parallel_tests gem'i ile RSpec'i paralel çalıştır
bundle exec parallel_rspec spec/system -n 4
```

```ruby
# spec/support/appium_session.rb — process bazlı capability override
class AppiumSession
  def self.driver
    @driver ||= begin
      worker = ENV["TEST_ENV_NUMBER"].to_i   # parallel_tests verir: "", "2", "3", ...
      port   = 4723 + (worker.zero? ? 0 : worker - 1)
      caps   = build_caps(server_port: port)
      Appium::Core.for(caps).start_driver
    end
  end
end
```

- Her test session'ı izole olmalı (login state, deep link, vb.)
- Test data leak'i engellemek için `noReset: false` veya `fullReset: true`
- Her parallel worker için ayrı Appium port + emulator

---

## 11. Reporting (Ruby)

| Tool | Kullanım |
|---|---|
| **Allure** (`allure-rspec`) | En yaygın; screenshot embed, step-by-step trace, history |
| **RSpec JUnit** (`rspec_junit_formatter`) | CI integration için (Jenkins, GitLab) |
| **RSpec HTML** | Built-in, basit |
| **rspec_html_reporter** gem | Renkli, detaylı HTML rapor |
| **Cucumber HTML** | BDD framework kullanılıyorsa |

### Allure örneği (Ruby)
```ruby
# Gemfile
gem "allure-rspec"

# spec/spec_helper.rb
require "allure-rspec"

AllureRspec.configure do |c|
  c.results_directory = "tmp/allure-results"
  c.clean_results_directory = true
  c.environment_properties = {
    platform: ENV.fetch("PLATFORM", "android"),
    appium_version: "2.x"
  }
end

# spec/system/login_spec.rb
RSpec.describe "Login", :appium do
  it "kullanıcı başarılı login olur" do
    Allure.add_attachment(
      name: "Before login",
      source: driver.screenshot_as(:png),
      type: Allure::ContentType::PNG
    )

    Allure.step("Login sayfasını aç") do
      driver.activate_app("com.example.app")
    end

    Allure.step("Credentials gir") do
      LoginPage.new(driver)
              .enter_email("x@y.com")
              .enter_password("secret")
              .submit
    end
  end

  after do |example|
    if example.exception
      Allure.add_attachment(
        name: "Failure screenshot",
        source: driver.screenshot_as(:png),
        type: Allure::ContentType::PNG
      )
    end
  end
end
```

### Cloud provider native reporting
LambdaTest / BrowserStack: video recording, network logs, device logs, console logs, app logs **otomatik** sağlar. Her test için dashboard URL'i.

---

## 12. Sık Karşılaşılan Pitfall'lar (Ruby)

### 1. XPath aşırı kullanımı
```ruby
# ❌ Kırılgan + yavaş
driver.find_element(:xpath, "/hierarchy/android.widget.FrameLayout[1]/.../Button")

# ✅ Accessibility ID
driver.find_element(:accessibility_id, "login_button")
```
**Çözüm:** Developer'lardan accessibility ID set etmesini iste — accessibility için zaten gerekli.

### 2. `sleep` kullanımı
```ruby
# ❌
click_login
sleep 3
expect(dashboard).to be_displayed

# ✅
click_login
wait.until { dashboard.displayed? }
```

### 3. Implicit + Explicit wait karışımı
İkisini birden kullanma → unpredictable timeout. Sadece **explicit** kullan.

### 4. Stale element exception
DOM yeniden render olunca eski element reference kaybolur.
```ruby
# ❌
btn = driver.find_element(:accessibility_id, "btn")
refresh_screen
btn.click   # Selenium::WebDriver::Error::StaleElementReferenceError

# ✅ — locator'ı sakla, element'i değil
BTN_LOCATOR = [:accessibility_id, "btn"].freeze

refresh_screen
driver.find_element(*BTN_LOCATOR).click
```

### 5. `noReset` ile state kirliliği
`noReset: true` hızlıdır ama test'ler arası state leak yapar. Critical flow'larda `fullReset: true` tercih et.

### 6. iOS WebDriverAgent setup sorunları
Gerçek iOS device'da WDA imzalama zor. Çözüm: cloud farm kullan, veya `xcodeOrgId` + `xcodeSigningId` capability'lerini doğru ver.

### 7. Emulator vs gerçek device farkı
Emulator'da çalışan test gerçek device'da fail edebilir (performans, GPU, sensor farkları). **Critical path'leri gerçek device'da çalıştır.**

### 8. Network koşullarını test etmemek
Offline, yavaş 3G, packet loss → cloud provider'lar network simulation sunar.

### 9. Locator stratejisinde platform-agnostic yazmaya çalışmak
İki platformda aynı UI bile farklı render edilir. Cross-platform POM'da Android ve iOS locator'larını **`locator` class macro'su** ile ayrı tut (bölüm 9'a bak).

### 10. App version drift
Test build'leri kontrol altında olmalı — staging app'i değişirse test fail eder. **Build versioning** zorunlu.

---

## 13. Performans Optimizasyonu

- **Accessibility ID > XPath** — locator hızı 10x
- **Session reuse** — her test için yeni session açma; RSpec `before(:all)`'da bir kere
- **Paralel test** — `parallel_tests` gem'i ile process bazlı paralel
- **Cloud parallelism** — 10-50 cihazda eş zamanlı çalıştır
- **noReset: true** (state kirliliğine dikkat ederek) → setup süresini düşürür
- **Lazy locator'lar** — POM'da locator constant'lar, element'ler değil

```ruby
# Session reuse
RSpec.configure do |c|
  c.before(:all, :appium) { AppiumSession.driver }
  c.after(:suite) { AppiumSession.driver&.quit }
end
```

---

## 14. Framework Tasarımı — QA Lead Bakışı (Ruby)

### Klasör yapısı (RSpec)
```
.
├── Gemfile
├── Rakefile
├── .rspec
├── config/
│   ├── android.yml
│   ├── ios.yml
│   └── lambdatest.yml
├── apps/
│   ├── app-debug.apk
│   └── MyApp.app
├── spec/
│   ├── spec_helper.rb
│   ├── support/
│   │   ├── appium_session.rb     # Driver factory + thread-local
│   │   ├── caps_builder.rb       # Capability JSON builder
│   │   ├── pages/
│   │   │   ├── base_page.rb
│   │   │   ├── login_page.rb
│   │   │   ├── dashboard_page.rb
│   │   │   └── components/
│   │   │       ├── header.rb
│   │   │       └── sidebar.rb
│   │   ├── helpers/
│   │   │   ├── wait_helpers.rb
│   │   │   ├── gesture_helpers.rb
│   │   │   └── screenshot.rb
│   │   └── hooks/
│   │       └── failure_screenshot.rb
│   ├── system/
│   │   ├── login_spec.rb
│   │   ├── checkout_spec.rb
│   │   └── profile_spec.rb
│   └── data/
│       └── users.yml             # test data
└── tmp/
    └── allure-results/
```

### DriverFactory pattern (Ruby — thread-local)
```ruby
# spec/support/appium_session.rb
require "appium_lib_core"

class AppiumSession
  def self.driver
    Thread.current[:appium_driver] ||= start_driver
  end

  def self.quit
    Thread.current[:appium_driver]&.quit
    Thread.current[:appium_driver] = nil
  end

  def self.start_driver
    caps = CapsBuilder.build(platform: ENV.fetch("PLATFORM", "android"))
    Appium::Core.for(caps).start_driver
  end
end
```

`Thread.current[...]` — paralel test'lerde her thread'in kendi driver'ı olur (Java'daki `ThreadLocal` karşılığı).

### Cross-platform abstraction (Ruby idiom)
```ruby
# spec/support/pages/login_page.rb
class LoginPage < BasePage
  include PlatformLocators

  locator :email_field,
          android: [:accessibility_id, "email_input"],
          ios:     [:accessibility_id, "email_input"]

  locator :password_field,
          android: [:accessibility_id, "password_input"],
          ios:     [:accessibility_id, "password_input"]

  locator :submit_btn,
          android: [:uiautomator, 'new UiSelector().text("Login")'],
          ios:     [:class_chain, '**/XCUIElementTypeButton[`name == "Login"`]']

  def login(email:, password:)
    email_field.send_keys(email)
    password_field.send_keys(password)
    submit_btn.click
    DashboardPage.new(driver)
  end
end
```

### Failure screenshot hook
```ruby
# spec/support/hooks/failure_screenshot.rb
RSpec.configure do |c|
  c.after(:each, :appium) do |example|
    if example.exception
      path = "tmp/screenshots/#{example.full_description.parameterize}.png"
      AppiumSession.driver.save_screenshot(path)
      puts "Screenshot saved: #{path}"
    end
  end
end
```

### Tag'leme ve filtreleme
```ruby
RSpec.describe "Heavy E2E", :slow, :appium do
  # ...
end
```

```bash
bundle exec rspec --tag slow                 # sadece slow
bundle exec rspec --tag ~slow                # slow olmayanlar (smoke)
bundle exec rspec --tag appium               # sadece Appium test'leri
```

---

## 15. QA Lead Seviyesi — Mimari Kararlar

### Test stratejisi
1. **Unit test** (developer) — %70
2. **API test** — %20
3. **UI integration (Appium)** — %10
   - Critical user journey'ler (login, checkout, payment, signup)
   - Cross-platform regression (Android + iOS)
   - Cross-device (tablet vs phone)
4. **Monkey/exploratory** — manual + Espresso monkey tool

### Device coverage matrix
Lead seviyesinde her release için cover edilecek matrix'i belirlemek senin sorumluluğun:
| Tier | Android | iOS |
|---|---|---|
| **P0 (her PR)** | Pixel 6 (API 33) | iPhone 15 (iOS 17) |
| **P1 (nightly)** | Samsung S22, Pixel 4a | iPhone 13, iPad Air |
| **P2 (release)** | Düşük-end Samsung A, Xiaomi | iPhone SE, eski iPad |

### Test data yönetimi
- **Test user pool** — her test'in kendi user'ı (data leak yok)
- **Deep link** ile state'i hızlıca kur (login bypass)
- **Mock backend** opsiyonu (WebMock + VCR; veya in-process WireMock)
- **Fixture'lar** — `spec/data/*.yml` test data dosyaları

```ruby
# spec/data/users.yml
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
# spec/support/test_data.rb
class TestData
  DATA = YAML.load_file("spec/data/users.yml").freeze

  def self.user(type)
    DATA.fetch(type.to_s)
  end
end

# Test'te
user = TestData.user(:admin)
login_page.login(email: user["email"], password: user["password"])
```

### Maintenance
- **Flaky test SLA:** %1'in altında flake rate; üstündekiler quarantine
- **Locator audit:** XPath kullanan test'leri flag'le, refactor için ticket aç
- **Test execution time monitoring:** Yavaş test'leri optimize et
- **App build versioning:** Test koşulan app build'i tag'le (semver + commit SHA)

### Team scaling
- **CODEOWNERS** ile POM ownership
- **Code review checklist:** locator strategy, wait kullanımı, paralel-safe mi?
- **Onboarding doc:** Local emulator setup + first PR guide
- **Pair testing:** Senior + junior çift mülakat
- **BDD layer (opsiyonel):** Cucumber + Gherkin → non-tech stakeholder'lar test okusun
  - Ruby ekosisteminde `cucumber` gem'i ile native entegrasyon

### Risk yönetimi
- **iOS gerçek device:** Apple Developer Program gerekir, certificate management
- **Android fragmentation:** Düşük-end device'larda farklı renderer behavior
- **Appium version upgrade:** Major version (2.x) breaking changes — major upgrade'ler PR olarak yapılmalı

---

## 16. En Sık Mülakat Soruları (Hızlı Cevaplar)

| Soru | Hızlı Cevap |
|---|---|
| Appium nedir? | Cross-platform mobile UI test framework; WebDriver protokolü tabanlı |
| Appium 1 vs 2? | 2.x driver/plugin tabanlı modüler mimari; yeni projeler 2.x kullanmalı |
| Ruby client'ı? | `appium_lib_core` (modern, önerilen) ve `appium_lib` (eski full-fat) |
| Native vs Hybrid vs Web? | Native: platform UI; Hybrid: WebView içinde HTML; Web: mobile browser |
| Hybrid app'i nasıl test edersin? | `available_contexts` → `set_context("WEBVIEW_...")` |
| En iyi locator? | accessibility_id; en hızlı, en güvenilir |
| XPath neden kötü? | Yavaş (tree traversal) + kırılgan (hierarchy değişimine duyarlı) |
| Wait stratejisi? | Sadece explicit wait (`Selenium::WebDriver::Wait`); `sleep` yasak |
| `noReset` vs `fullReset`? | noReset hızlı ama state leak; fullReset yavaş ama izole |
| Paralel test? | `parallel_tests` gem + `Thread.current[:driver]` + cloud farm |
| Cloud farm tercihi? | BrowserStack, Sauce Labs, LambdaTest — gerçek device + paralel |
| iOS gerçek device challenge? | WebDriverAgent imzalama + Apple cert; çoğu zaman cloud çözüm |
| Reporting? | `allure-rspec` (en yaygın), `rspec_junit_formatter`, cloud native dashboard |
| Cross-platform POM? | `locator` class macro DSL'i ile platform-aware POM |
| Flaky test çözümü? | Explicit wait, locator audit, retry (son çare), data isolation |
| CI/CD'de emulator? | macos runner + KVM, self-hosted, veya cloud farm |
| Test data yönetimi? | YAML fixture, test user pool, deep link state setup, mock backend |
| Device coverage? | P0 (her PR), P1 (nightly), P2 (release) tiered matrix |
| Lead'in en önemli kararı? | Hangi case Appium'da, hangisi API/unit'te → test piramidi |

---

## 17. Çalışma Yol Haritası

### 1. Hafta — Temeller
- [ ] Appium 2.x kurulumu (UiAutomator2 + XCUITest driver)
- [ ] `appium_lib_core` gem'i ile ilk session
- [ ] Appium Inspector ile basit bir app'i incele
- [ ] İlk Android test'ini yaz (login flow) — RSpec ile
- [ ] Locator stratejilerini dene (accessibility_id, id, uiautomator, xpath)

### 2. Hafta — Orta
- [ ] `Selenium::WebDriver::Wait` ile explicit wait pattern'leri
- [ ] Touch gestures (swipe, scroll, long press) — `execute_script "mobile: ..."`
- [ ] Hybrid app context switching
- [ ] Page Object Model — BasePage + 3 sayfa
- [ ] `locator` class macro DSL'i ile cross-platform POM

### 3. Hafta — Framework
- [ ] DriverFactory + `Thread.current[:driver]` pattern
- [ ] `parallel_tests` ile paralel RSpec
- [ ] Config management (YAML config + `ENV` override)
- [ ] Test data fixture'ları (YAML)
- [ ] Failure screenshot hook
- [ ] `allure-rspec` reporting entegrasyonu

### 4. Hafta — CI/CD & Lead
- [ ] GitHub Actions / Jenkins pipeline
- [ ] LambdaTest veya BrowserStack entegrasyonu
- [ ] Paralel cloud test (5+ device eş zamanlı)
- [ ] Device coverage matrix dökümante et
- [ ] Flake rate dashboard
- [ ] Code review checklist + onboarding doc

---

## 18. Kaynaklar

- **Resmi:** appium.io
- **GitHub:** github.com/appium/appium
- **Inspector:** github.com/appium/appium-inspector
- **Driver listesi:** appium.io/docs/en/latest/ecosystem/drivers/
- **Ruby client (modern):** github.com/appium/ruby_lib_core
- **Ruby client (full-fat):** github.com/appium/ruby_lib
- **`allure-rspec`:** github.com/allure-framework/allure-ruby
- **`parallel_tests`:** github.com/grosser/parallel_tests
- **Sample Ruby projeler:** github.com/appium/ruby_lib_core/tree/master/example
- **LambdaTest docs:** lambdatest.com/support/docs/appium-testing/
