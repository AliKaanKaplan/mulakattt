# Appium — QA Lead Mülakat Hazırlık Rehberi (Ruby)

> Mobile (iOS / Android) ve cross-platform UI test otomasyonu için endüstri standardı. Bu doküman QA Lead seviyesinde mülakata hazırlık içindir.
> **Tüm örnekler Ruby'dir** — `appium_lib_core` + RSpec ekosistemi kullanılmıştır.

---

## İçindekiler

1. [Appium Nedir? Felsefe](#1-appium-nedir-felsefe)
2. [Mimari — Stack Detayı](#2-mimari--stack-detayi)
3. [Temel Terimler — Sözlük](#3-temel-terimler--sozluk)
4. [Appium 1.x vs 2.x](#4-appium-1x-vs-2x)
5. [Driver Ekosistemi](#5-driver-ekosistemi)
6. [Kurulum](#6-kurulum)
7. [Proje Yapısı](#7-proje-yapisi)
8. [İlk Test — Hello World](#8-ilk-test--hello-world)
9. [Capabilities Detaylı](#9-capabilities-detayli)
10. [Locator Stratejileri](#10-locator-stratejileri)
11. [Element / Driver API](#11-element--driver-api)
12. [Touch Gestures & W3C Actions](#12-touch-gestures--w3c-actions)
13. [Mobile Gesture Extensions (`mobile:`)](#13-mobile-gesture-extensions)
14. [App Lifecycle Yönetimi](#14-app-lifecycle-yonetimi)
15. [Device Interaction](#15-device-interaction)
16. [Wait Stratejileri](#16-wait-stratejileri)
17. [Context — Native / Hybrid / Web](#17-context--native--hybrid--web)
18. [Network, Geolocation, Notifications](#18-network-geolocation-notifications)
19. [Permission Management](#19-permission-management)
20. [Deep Links & App URL Schemes](#20-deep-links)
21. [Performance Metrics](#21-performance-metrics)
22. [Page Object Model — Mobile](#22-page-object-model--mobile)
23. [Cross-Platform POM Stratejisi](#23-cross-platform-pom)
24. [Cloud Device Farms](#24-cloud-device-farms)
25. [CI/CD Entegrasyonu](#25-cicd-entegrasyonu)
26. [Reporting](#26-reporting)
27. [Debugging Araçları](#27-debugging-araclari)
28. [Test Türleri Matrix'i](#28-test-turleri-matrix-i)
29. [Test Data Management](#29-test-data-management)
30. [Cucumber Entegrasyonu](#30-cucumber-entegrasyonu)
31. [Visual Testing](#31-visual-testing)
32. [Sık Karşılaşılan Pitfall'lar](#32-pitfall-lar)
33. [Performans Optimizasyonu](#33-performans-optimizasyonu)
34. [Framework Tasarımı](#34-framework-tasarimi)
35. [QA Lead Seviyesi — Mimari Kararlar](#35-mimari-kararlar)
36. [En Sık Mülakat Soruları](#36-mulakat-sorulari)
37. [Çalışma Yol Haritası](#37-calisma-yol-haritasi)
38. [Kaynaklar](#38-kaynaklar)

---

## 1. Appium Nedir? Felsefe

**Appium**, mobile uygulamalar için **cross-platform UI test otomasyonu** sağlayan açık kaynak bir framework'tür. Tek bir API ile **iOS, Android, Windows ve macOS** uygulamalarını test edebilirsin.

### Temel özellikleri
- **WebDriver protokolü** üzerine kurulu (Selenium ile aynı — W3C WebDriver standardı)
- **Native, hybrid ve mobile web** uygulamaları test eder
- Source code değişikliği gerektirmez (no instrumentation in production)
- **Dil-agnostic:** Java, Python, Ruby, JavaScript, C#, Robot Framework client'ları
- **Driver mimarisi:** Her platform için ayrı driver (UiAutomator2, XCUITest, Espresso, vb.)

### Felsefe — Üç temel prensip

**1. "Otomasyon test'i, uygulama developer'larının değiştirmesini gerektirmemeli"**
Production build'i test edebilmelisin. Source code'a test hook'ı, framework dependency'si eklenmemeli.

**2. "WebDriver protokolüyle compatible — Selenium ekosistemini reuse et"**
Selenium ile aynı W3C protokol → aynı client kütüphaneleri, aynı senaryolar, aynı düşünce yapısı.

**3. "Cross-platform abstraction — tek code base, iki platform"**
Aynı test framework, aynı senaryo şablonu Android ve iOS'ta koşar.

### Ruby ekosistemi client'ları

| Client | Status | Açıklama |
|---|---|---|
| **`appium_lib_core`** | ✅ Modern, önerilen | Hafif, sadece WebDriver komutları |
| **`appium_lib`** | ⚠️ Legacy | "Full-fat" client, `appium_lib_core` üstüne helper'lar (`text`, `button`, vb.) ekler |

Yeni projeler **`appium_lib_core`** kullanmalı. `appium_lib` Appium 1.x'ten kalma helper'lara sahip ama bakımı yavaş.

### Capybara ile karşılaştırma
| | Capybara | Appium |
|---|---|---|
| Hedef | Web UI | Mobile (iOS/Android) UI |
| Auto-wait | ✅ Built-in | ❌ Manuel (Selenium Wait) |
| DSL | Yüksek seviyeli | Selenium-style, düşük seviyeli |
| Ruby client | `capybara` | `appium_lib_core` |
| Driver | Rack::Test, Selenium, Cuprite | UiAutomator2, XCUITest, Espresso, ... |

### Alternatif framework'ler
| Framework | Platform | Avantaj | Dezavantaj |
|---|---|---|---|
| **Appium** | iOS + Android + Windows | Cross-platform, dil-agnostic | Yavaş (proxy katmanı) |
| **Espresso** (Google) | Sadece Android | Çok hızlı (in-process), stable | Android only, Java/Kotlin only |
| **XCUITest** (Apple) | Sadece iOS | Native, çok hızlı | iOS only, Swift/ObjC only |
| **Detox** (Wix) | React Native | RN için optimal, gray-box | Sadece RN |
| **Maestro** | iOS + Android | YAML DSL, modern, hızlı | Yeni, ekosistem küçük |

**Lead seviyesi karar:** Cross-platform Ruby ekosisteminde **Appium**. Tek platform native team'lerinde **Espresso/XCUITest**.

---

## 2. Mimari — Stack Detayı

Appium **bir proxy/server'dır**. Otomatik uygulamayı **kendisi kontrol etmez** — platform'un native otomasyon framework'unu sürer.

### Tam stack

```
┌──────────────────────────────────────────────────────────────┐
│ Test Code (Ruby — appium_lib_core + RSpec)                   │
│   driver.find_element(:accessibility_id, "login_button")     │
│   ↓ WebDriver client library                                 │
├──────────────────────────────────────────────────────────────┤
│ HTTP / JSON Wire Protocol (W3C WebDriver)                    │
│   POST /session/{id}/element                                 │
│   { "using": "accessibility id", "value": "login_button" }   │
│   ↓                                                          │
├──────────────────────────────────────────────────────────────┤
│ Appium Server (Node.js)                                      │
│   - Session yönetimi                                         │
│   - Command routing                                          │
│   - Capability validation                                    │
│   ↓ uygun driver'ı seçer                                     │
├──────────────────────────────────────────────────────────────┤
│ Drivers (her platform için)                                  │
│   ┌────────────────┐  ┌──────────────┐  ┌────────────────┐ │
│   │ UiAutomator2   │  │ XCUITest     │  │ Espresso       │ │
│   │ (Android)      │  │ (iOS)        │  │ (Android)      │ │
│   └───────┬────────┘  └──────┬───────┘  └────────┬───────┘ │
│           ↓                  ↓                   ↓          │
├──────────────────────────────────────────────────────────────┤
│ Bootstrap apps                                               │
│   ┌────────────────┐  ┌──────────────────────┐              │
│   │ uiautomator2-  │  │ WebDriverAgent (WDA) │              │
│   │ server.apk     │  │ Xcode-built iOS app  │              │
│   └───────┬────────┘  └──────────┬───────────┘              │
│           ↓                      ↓                          │
├──────────────────────────────────────────────────────────────┤
│ Platform Native Automation Framework                         │
│   ┌────────────────┐  ┌──────────────────────┐              │
│   │ UI Automator   │  │ XCUITest framework   │              │
│   │ (Android SDK)  │  │ (Xcode/iOS SDK)      │              │
│   └───────┬────────┘  └──────────┬───────────┘              │
│           ↓                      ↓                          │
├──────────────────────────────────────────────────────────────┤
│ Mobile device / emulator / simulator                         │
│   ┌────────────────┐  ┌──────────────────────┐              │
│   │ Test App       │  │ Test App             │              │
│   │ (APK installed)│  │ (.app installed)     │              │
│   └────────────────┘  └──────────────────────┘              │
└──────────────────────────────────────────────────────────────┘
```

### Komut akışı — örnek

`driver.find_element(:accessibility_id, "login").click` çağırdığında:

1. **Ruby client** komutu W3C JSON formatına serialize eder.
2. **HTTP request** Appium server'a gönderilir (default `http://localhost:4723`).
3. **Appium server** session ID'sini kullanarak ilgili driver'a (örn. UiAutomator2) yönlendirir.
4. **UiAutomator2 driver** komutu `uiautomator2-server.apk`'sine forwards (ADB üzerinden).
5. Bu apk **UI Automator framework**'ünü kullanarak Android'in native automation API'sini çağırır.
6. Element bulunur → click event tetiklenir.
7. Response zinciri geri döner.

**Tek bir click 50-200ms sürebilir** — bu yüzden mobile test genel olarak slow.

### Mimari'nin sonuçları

| Etki | Açıklama |
|---|---|
| **Yavaşlık** | Çok katmanlı → 5-10 saniyelik click gecikmesi mümkün |
| **Driver bağımlılığı** | UiAutomator2 server crash → tüm session boşa |
| **Cross-platform abstraction** | Aynı kod farklı driver'lara routing yapar |
| **No-code-change** | Production APK/IPA test edilebilir |
| **Real device complexity** | iOS gerçek device'da WDA imzalama gerekli |

### Mülakat klasiği
**Soru:** *"Appium nasıl çalışır mimari olarak?"*

**Cevap:** Appium W3C WebDriver protokolü üstüne kurulu **bir proxy server**'dır (Node.js). Test client'ı HTTP üzerinden komut yollar, Appium server bunu platform-specific bir driver'a (UiAutomator2, XCUITest) routing yapar. Driver, native automation framework'unu (Android UI Automator, iOS XCUITest) kullanarak gerçek device/emulator'da işlemi gerçekleştirir. Appium **uygulamayı kendisi kontrol etmez**, native framework'leri sürer.

---

## 3. Temel Terimler — Sözlük

### Appium Server
**Node.js process'i** — port 4723'te HTTP request'leri dinler, ilgili driver'a route eder.

### Driver
**Platform-specific komut handler'ı.** Aynı WebDriver komutunu farklı platforma çevirir.
| Driver | Platform |
|---|---|
| `UiAutomator2` | Android (modern, default) |
| `XCUITest` | iOS (modern, default) |
| `Espresso` | Android (in-process, daha hızlı, instrumentation gerekir) |
| `Mac2` | macOS native |
| `Windows` | Windows native (WinAppDriver tabanlı) |
| `Gecko` | Firefox/Gecko-based mobile |
| `Safari` | Mobile Safari |
| `Chromium` | Mobile Chrome |

### Session
**Bir test run'ı boyunca aktif olan driver instance'ı.** `start_driver` ile başlar, `quit` ile biter. Her test session = bir device/emulator allocation.

### Capabilities
**Session başlatırken Appium'a "ne istediğini söyleyen" JSON config'i.** Platform, device, app path, automation strategy.

```ruby
caps = {
  platformName:               "Android",
  "appium:platformVersion":    "13",
  "appium:deviceName":         "Pixel_6_API_33",
  "appium:app":                "/path/to/app.apk",
  "appium:automationName":     "UiAutomator2"
}
```

### W3C Capability Prefixes
- **Standart capability'ler** (W3C tanımlı): prefix yok → `platformName`, `browserName`
- **Vendor capability'leri** (Appium-specific): `appium:` prefix → `appium:deviceName`, `appium:app`
- **Cloud-specific**: `lt:options`, `bstack:options`, `sauce:options`

### Bootstrap App
**Driver'ın device'a kurduğu helper uygulama.**
- Android: `uiautomator2-server.apk`, `appium-uiautomator2-server-debug-androidTest.apk`
- iOS: `WebDriverAgent.app` (WDA)

Bu app'ler test session'ı boyunca arka planda çalışır, device'daki UI'a komut gönderir.

### Element
**Bir UI widget'ının handle'ı.** `find_element` döner. Üzerinde `click`, `text`, `send_keys` gibi method'lar.

### Locator
**Element'i bulmak için kullanılan strateji + değer.**
```ruby
driver.find_element(:accessibility_id, "login_btn")
#                   strategy           value
```

### Context
**Web vs Native moda geçiş.**
- `NATIVE_APP` — native UI'da
- `WEBVIEW_com.example.app` — embedded WebView'da

### Inspector
**Appium'un GUI tool'u** — device'daki UI hierarchy'i göster, element seçimi yap, locator'ı kopyala.

### ADB (Android Debug Bridge)
**Android device ile iletişim kuran CLI tool.** Appium UiAutomator2 driver bunu kullanır.

### WDA (WebDriverAgent)
**iOS bootstrap app'i.** Facebook tarafından geliştirilmiş, Appium XCUITest driver'ı tarafından device'a kurulup test edilen app'i kontrol eder.

### `idb` (iOS Debug Bridge)
**iOS için ADB benzeri tool.** Apple resmi `xcrun simctl` + Meta'nın `idb` ile birlikte kullanılır.

### Implicit vs Explicit Wait
- **Implicit:** Driver global default timeout. Antipattern.
- **Explicit:** Belirli bir koşulu bekle. `Selenium::WebDriver::Wait`.

### Mobile Gesture Extensions (`mobile:` commands)
**Standart W3C action'ları yetmediğinde driver-specific kısa yollar.**
```ruby
driver.execute_script("mobile: swipeGesture", { direction: "up", ... })
driver.execute_script("mobile: tap", { x: 100, y: 200 })
```

### `appium driver doctor`
**Environment check tool'u.** Eksik dependency varsa söyler.
```bash
appium driver doctor uiautomator2
appium driver doctor xcuitest
```

---

## 4. Appium 1.x vs 2.x

**Appium 2.x (2022+)** ile mimari büyük ölçüde değişti. Yeni projeler **2.x kullanmalı**.

### Karşılaştırma

| | Appium 1.x | Appium 2.x |
|---|---|---|
| Mimari | Monolithic — tüm driver'lar built-in | Driver/plugin tabanlı modüler |
| Install | `npm install -g appium` (her şey gelir) | `npm install -g appium` + ayrı driver install |
| Driver upgrade | Appium upgrade et | Driver bağımsız upgrade et |
| Plugin sistemi | ❌ Yok | ✅ Plugin marketplace |
| Capability syntax | Eski JSON (deprecated) | W3C standard + `appium:` prefix |
| Status | ⚠️ Deprecated | ✅ Current |

### Migration: 1.x → 2.x

**Capability değişikliği:**
```ruby
# 1.x (eski)
caps = {
  platformName:     "Android",
  platformVersion:  "13",
  deviceName:       "Pixel",
  app:              "/path",
  automationName:   "UiAutomator2"
}

# 2.x (yeni)
caps = {
  platformName:               "Android",
  "appium:platformVersion":    "13",
  "appium:deviceName":         "Pixel",
  "appium:app":                "/path",
  "appium:automationName":     "UiAutomator2"
}
```

**Driver install:**
```bash
# 1.x — drivers built-in
appium

# 2.x — ayrı install
npm install -g appium
appium driver install uiautomator2
appium driver install xcuitest
appium
```

**Plugin system (2.x özel):**
```bash
appium plugin install images          # OCR-based element finding
appium plugin install relaxed-caps    # capability validation gevşet
appium plugin install execute-driver  # batch commands
appium --use-plugins=images,relaxed-caps
```

### Driver list ve install
```bash
appium driver list                       # available
appium driver list --installed           # installed
appium driver install uiautomator2
appium driver install xcuitest
appium driver install espresso
appium driver install mac2
appium driver install windows
appium driver update uiautomator2
appium driver uninstall espresso
```

---

## 5. Driver Ekosistemi

### Android driver'ları

#### **UiAutomator2** (default, en yaygın)
```bash
appium driver install uiautomator2
```

| Özellik | Açıklama |
|---|---|
| Çalışma şekli | Out-of-process — APK device'a kurulur, ADB üzerinden konuşur |
| Hız | Orta |
| App tipi | Native, hybrid, mobile Chrome |
| Source erişimi | Gerekli değil — production APK test edilebilir |
| Min Android | API 21 (Lollipop, 5.0) |

#### **Espresso**
```bash
appium driver install espresso
```

| Özellik | Açıklama |
|---|---|
| Çalışma şekli | In-process — test runner app içinde |
| Hız | **Çok hızlı** (UiAutomator2'den 3-5x) |
| App tipi | Native (WebView sınırlı) |
| Source erişimi | **Gerekli** — instrumented build gerekir |
| Min Android | API 19 |

**Lead karar:** Source erişimi varsa Espresso (hız), yoksa UiAutomator2 (production APK).

### iOS driver'ları

#### **XCUITest** (default, tek seçenek)
```bash
appium driver install xcuitest
```

| Özellik | Açıklama |
|---|---|
| Çalışma şekli | XCUITest framework + WDA (WebDriverAgent) bootstrap |
| Hız | Orta-yavaş |
| Gereksinim | macOS, Xcode, Xcode CLI tools |
| Real device | Apple Developer Account + WDA imzalama |
| Simulator | Setup kolay |

#### **UIAutomation** (legacy, kullanma)
Apple'ın eski automation API'si. iOS 9.3+ ile deprecated.

### Diğer driver'lar

#### **Mac2** (macOS native)
```bash
appium driver install mac2
```
macOS native uygulamalarını XCUITest tabanlı sürer.

#### **Windows** (WinAppDriver tabanlı)
```bash
appium driver install windows
```
Windows native uygulamaları için.

#### **Safari / Chromium / Gecko**
Mobile browser test'leri için. Selenium tarzı `browserName` capability ile.

### Driver karşılaştırma matrisi

| Driver | Platform | Hız | Setup | Production APK/IPA |
|---|---|---|---|---|
| UiAutomator2 | Android | ⚡⚡ | Kolay | ✅ |
| Espresso | Android | ⚡⚡⚡⚡ | Orta | ⚠️ Source gerekli |
| XCUITest | iOS | ⚡⚡ | Zor (WDA) | ✅ |
| Mac2 | macOS | ⚡⚡⚡ | Kolay | ✅ |
| Windows | Windows | ⚡⚡⚡ | Orta | ✅ |

---

## 6. Kurulum

### Appium Server (Node.js)
```bash
# Node.js 16+ gerekli
node --version

# Appium 2.x install
npm install -g appium

# Driver'ları yükle
appium driver install uiautomator2       # Android
appium driver install xcuitest           # iOS
appium driver install espresso           # Android in-process (opsiyonel)

# Server'ı başlat
appium server --port 4723
```

### Ruby tarafı (Gemfile)
```ruby
source "https://rubygems.org"

group :test do
  gem "appium_lib_core", "~> 9.0"
  gem "rspec"
  gem "rspec_junit_formatter"
  gem "allure-rspec"
  gem "parallel_tests"
  gem "pry-byebug"
  gem "yaml"
end
```

```bash
bundle install
```

### Android SDK gereksinimleri
```bash
# Android Studio kurulu olmalı veya command-line tools
# $ANDROID_HOME tanımlı olmalı

# Gerekli component'ler
sdkmanager "platform-tools"
sdkmanager "platforms;android-33"
sdkmanager "system-images;android-33;google_apis;x86_64"

# Emulator oluştur
avdmanager create avd -n Pixel_6_API_33 -k "system-images;android-33;google_apis;x86_64"

# Emulator başlat
emulator -avd Pixel_6_API_33 &

# Device kontrol
adb devices
```

### iOS Xcode gereksinimleri
```bash
# Xcode kurulu olmalı (App Store'dan)
xcode-select --install

# Xcode CLI tools
xcrun --version

# Simulator listele
xcrun simctl list devices

# WebDriverAgent
# Appium XCUITest driver otomatik install eder, ama gerçek device için imza gerekir
```

### Doctor — environment check
```bash
appium driver doctor uiautomator2
appium driver doctor xcuitest
```

Eksik dependency varsa söyler (JAVA_HOME, ANDROID_HOME, Xcode path, vb.).

### Inspector (locator keşfi için)
```bash
# CLI
npm install -g appium-inspector

# GUI app olarak indir
# https://github.com/appium/appium-inspector
```

Inspector başlatma:
```bash
appium-inspector
# veya GUI app aç
```

Inspector'da:
1. Server'a bağlan (`localhost:4723`)
2. Capability'leri ver (platform, app)
3. Start session
4. UI hierarchy'i görüntüle, element'i seç
5. Best locator'ı kopyala

### Version uyumluluğu
| Appium | Ruby client | Node.js | Android SDK | Xcode |
|---|---|---|---|---|
| 2.5+ | `appium_lib_core` 9.x | 18+ | 33+ | 15+ |
| 2.0-2.4 | `appium_lib_core` 8.x | 16+ | 30+ | 14+ |
| 1.22+ (legacy) | `appium_lib_core` 5.x | 14 | 29+ | 13+ |

---

## 7. Proje Yapısı

### Kapsamlı RSpec layout
```
.
├── Gemfile
├── Rakefile
├── .rspec
├── config/
│   ├── android.yml              # Android caps
│   ├── ios.yml                  # iOS caps
│   ├── lambdatest.yml           # Cloud caps
│   └── browserstack.yml
├── apps/                        # Build artifact'leri
│   ├── android/
│   │   ├── app-debug.apk
│   │   └── app-release.apk
│   └── ios/
│       └── MyApp.app
├── spec/
│   ├── spec_helper.rb
│   ├── support/
│   │   ├── appium_session.rb    # Driver factory + Thread.current
│   │   ├── caps_builder.rb      # Capability JSON builder
│   │   ├── platform.rb          # ENV based platform select
│   │   ├── pages/
│   │   │   ├── base_page.rb
│   │   │   ├── android/
│   │   │   │   ├── login_page.rb
│   │   │   │   └── dashboard_page.rb
│   │   │   ├── ios/
│   │   │   │   ├── login_page.rb
│   │   │   │   └── dashboard_page.rb
│   │   │   ├── shared/                # cross-platform POM
│   │   │   │   └── login_page.rb
│   │   │   └── components/
│   │   │       ├── header.rb
│   │   │       └── tab_bar.rb
│   │   ├── helpers/
│   │   │   ├── wait_helpers.rb
│   │   │   ├── gesture_helpers.rb
│   │   │   ├── screenshot.rb
│   │   │   ├── permission_helpers.rb
│   │   │   └── deep_link_helpers.rb
│   │   ├── hooks/
│   │   │   ├── failure_screenshot.rb
│   │   │   └── session_management.rb
│   │   └── shared_contexts/
│   │       └── authenticated_user.rb
│   ├── system/
│   │   ├── login_spec.rb
│   │   ├── checkout_spec.rb
│   │   └── profile_spec.rb
│   └── data/                    # YAML test fixtures
│       ├── users.yml
│       └── products.yml
└── tmp/
    ├── screenshots/
    └── allure-results/
```

---

## 8. İlk Test — Hello World

### `spec/spec_helper.rb`
```ruby
require "appium_lib_core"
require "rspec"
require "yaml"

Dir[File.expand_path("support/**/*.rb", __dir__)].sort.each { |f| require f }

PLATFORM = ENV.fetch("PLATFORM", "android").to_sym

RSpec.configure do |c|
  c.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  c.filter_run_when_matching :focus
  c.example_status_persistence_file_path = "tmp/rspec_status.txt"
  c.disable_monkey_patching!
  c.order = :random
  Kernel.srand c.seed

  c.before(:each) do
    AppiumSession.driver
  end

  c.after(:each) do |example|
    if example.exception
      ScreenshotHelper.save("tmp/screenshots/#{example.full_description.parameterize}.png")
    end
  end

  c.after(:suite) do
    AppiumSession.quit
  end
end
```

### `spec/support/appium_session.rb`
```ruby
class AppiumSession
  def self.driver
    Thread.current[:appium_driver] ||= start_driver
  end

  def self.quit
    Thread.current[:appium_driver]&.quit
    Thread.current[:appium_driver] = nil
  end

  def self.start_driver
    caps = CapsBuilder.build(platform: PLATFORM)
    Appium::Core.for(caps).start_driver
  end
end
```

### `spec/support/caps_builder.rb`
```ruby
class CapsBuilder
  def self.build(platform:)
    config = YAML.load_file("config/#{platform}.yml")
    {
      caps: config["caps"].transform_keys(&:to_sym),
      appium_lib: {
        server_url: ENV.fetch("APPIUM_URL", "http://127.0.0.1:4723")
      }
    }
  end
end
```

### `config/android.yml`
```yaml
caps:
  platformName: Android
  "appium:platformVersion": "13"
  "appium:deviceName": Pixel_6_API_33
  "appium:app": apps/android/app-debug.apk
  "appium:automationName": UiAutomator2
  "appium:appPackage": com.example.app
  "appium:appActivity": .MainActivity
  "appium:autoGrantPermissions": true
  "appium:noReset": false
  "appium:newCommandTimeout": 120
```

### İlk test — `spec/system/login_spec.rb`
```ruby
require "spec_helper"

RSpec.describe "Mobile Login", :appium do
  let(:driver) { AppiumSession.driver }

  it "kullanıcı başarılı giriş yapar" do
    wait = Selenium::WebDriver::Wait.new(timeout: 10)

    email_field = wait.until { driver.find_element(:accessibility_id, "email_input") }
    email_field.send_keys("test@example.com")

    password_field = driver.find_element(:accessibility_id, "password_input")
    password_field.send_keys("secret123")

    driver.find_element(:accessibility_id, "login_button").click

    dashboard = wait.until { driver.find_element(:accessibility_id, "dashboard_title") }
    expect(dashboard.text).to include("Welcome")
  end
end
```

### Çalıştırma
```bash
# Appium server (ayrı terminal)
appium server --port 4723

# Android emulator (ayrı terminal)
emulator -avd Pixel_6_API_33

# Test'i koştur
PLATFORM=android bundle exec rspec spec/system/login_spec.rb

# iOS
PLATFORM=ios bundle exec rspec spec/system/login_spec.rb
```

---

## 9. Capabilities Detaylı

Capability'ler **session başlatırken Appium'a "ne istediğini söyleyen" JSON config**'idir.

### W3C standart capability'leri (prefix yok)
| Capability | Açıklama |
|---|---|
| `platformName` | "Android" / "iOS" / "macOS" / "Windows" |
| `browserName` | Mobile browser test için ("Chrome", "Safari") |
| `browserVersion` | Browser version |
| `acceptInsecureCerts` | SSL cert validation skip |
| `pageLoadStrategy` | "normal", "eager", "none" |
| `unhandledPromptBehavior` | "accept", "dismiss", vb. |

### Appium vendor capability'leri (`appium:` prefix)

#### Genel
| Capability | Açıklama |
|---|---|
| `appium:platformVersion` | OS sürümü |
| `appium:deviceName` | Cihaz adı (emülatör adı veya udid) |
| `appium:automationName` | "UiAutomator2", "XCUITest", "Espresso", "Mac2" |
| `appium:app` | APK / .app / .ipa absolute path veya cloud URL |
| `appium:noReset` | true → session sonunda app data silinmez |
| `appium:fullReset` | true → app uninstall + reinstall (yavaş, izole) |
| `appium:newCommandTimeout` | Idle session timeout (default 60sn) |
| `appium:language` | Device language (örn. "tr") |
| `appium:locale` | Device locale (örn. "TR") |
| `appium:orientation` | "PORTRAIT" / "LANDSCAPE" |
| `appium:printPageSourceOnFindFailure` | Hata anında page source'u logla |

#### Android-specific
| Capability | Açıklama |
|---|---|
| `appium:appPackage` | Android app package name (`com.example.app`) |
| `appium:appActivity` | Başlangıç activity'si (`.MainActivity`) |
| `appium:appWaitActivity` | Hangi activity gelene kadar bekle |
| `appium:appWaitDuration` | App load timeout (default 20sn) |
| `appium:autoGrantPermissions` | Runtime permission'ları auto-grant |
| `appium:disableWindowAnimation` | UI animations kapat |
| `appium:dontStopAppOnReset` | Reset'te app'i kill etme |
| `appium:enforceAppInstall` | Force reinstall even if already installed |
| `appium:adbExecTimeout` | ADB command timeout |
| `appium:systemPort` | UiAutomator2 server port (paralel için unique) |

#### iOS-specific
| Capability | Açıklama |
|---|---|
| `appium:bundleId` | iOS app bundle ID (`com.example.MyApp`) |
| `appium:udid` | Gerçek device UDID (`auto` for simulator) |
| `appium:xcodeOrgId` | Apple Developer Team ID |
| `appium:xcodeSigningId` | Code signing identity (`iPhone Developer`) |
| `appium:autoAcceptAlerts` | Alert'leri auto-accept |
| `appium:autoDismissAlerts` | Alert'leri auto-dismiss |
| `appium:wdaLocalPort` | WDA local port (paralel için unique) |
| `appium:webDriverAgentUrl` | Pre-installed WDA URL (hız için) |
| `appium:useNewWDA` | Test başında WDA'yı reinstall |
| `appium:usePrebuiltWDA` | Pre-built WDA kullan |
| `appium:simulatorStartupTimeout` | Simulator boot timeout |

### `noReset` vs `fullReset` — kritik karar

| | Davranış | Hız | İzolasyon |
|---|---|---|---|
| `noReset: false, fullReset: false` (default) | Session arası app reset (data clear), ama app yüklü kalır | Orta | Orta |
| `noReset: true` | Session arası **hiçbir şey yapılmaz** — state önceki test'ten kalır | ⚡⚡⚡ | ❌ Yok |
| `fullReset: true` | Session başlangıcında uninstall + reinstall + data clear | 🐢 | ✅ Tam |

**Lead karar matrix:**
- Hızlı smoke test → `noReset: true` (state leak'e dikkat)
- Critical flow → `noReset: false` (default)
- Tam izolasyon gereken regression → `fullReset: true`

### Cloud-specific capability'ler

**LambdaTest:**
```ruby
caps = {
  platformName: "Android",
  "appium:deviceName": "Galaxy S22",
  "appium:platformVersion": "13",
  "appium:app": "lt://APP123456",
  "lt:options" => {
    user:      ENV["LT_USERNAME"],
    accessKey: ENV["LT_ACCESS_KEY"],
    build:     "Release #{Date.today}",
    name:      "Login test",
    video:     true,
    network:   true,
    visual:    true,
    console:   true,
    tunnel:    false,
    geoLocation: "TR"
  }
}
```

**BrowserStack:**
```ruby
caps = {
  platformName: "Android",
  "appium:deviceName": "Samsung Galaxy S22",
  "appium:app": "bs://abc123...",
  "bstack:options" => {
    userName:    ENV["BROWSERSTACK_USER"],
    accessKey:   ENV["BROWSERSTACK_KEY"],
    projectName: "MyApp",
    buildName:   "Build #{ENV['CI_BUILD']}",
    sessionName: "Login flow",
    local:       false,
    debug:       true,
    networkLogs: true,
    consoleLogs: "verbose",
    appiumLogs:  true,
    deviceLogs:  true,
    video:       true
  }
}
```

**Sauce Labs:**
```ruby
caps = {
  platformName: "Android",
  "appium:deviceName": "Google Pixel 6 GoogleAPI Emulator",
  "appium:app": "storage:filename=app.apk",
  "sauce:options" => {
    username:  ENV["SAUCE_USERNAME"],
    accessKey: ENV["SAUCE_ACCESS_KEY"],
    name:      "Test name",
    build:     "Build #{ENV['CI_BUILD']}",
    tags:      ["smoke", "android"]
  }
}
```

---

## 10. Locator Stratejileri

Element'i bulmak için kullanılan strateji + değer.

### Strateji karşılaştırma matrisi

| Strateji | Hız | Güvenilirlik | Cross-platform | Kullanım |
|---|---|---|---|---|
| **accessibility_id** | ⚡⚡⚡ | ✅ En iyi | ✅ İkisinde de | iOS: `accessibilityIdentifier`, Android: `content-desc` |
| **id** | ⚡⚡⚡ | ✅ | ❌ Platform-specific | Android: `resource-id`, iOS: name |
| **class name** | ⚡⚡ | ⚠️ | ❌ | UI widget tipi (`android.widget.Button`) |
| **xpath** | 🐢 | ⚠️ | ✅ | En esnek, **en yavaş ve kırılgan** |
| **uiautomator** | ⚡⚡ | ✅ | ❌ Android only | UiAutomator selector language |
| **predicate** | ⚡⚡ | ✅ | ❌ iOS only | iOS predicate string |
| **class chain** | ⚡⚡ | ✅ | ❌ iOS only | iOS class chain (XPath benzeri, hızlı) |
| **android view tag** | ⚡⚡ | ⚠️ | ❌ Android only | Espresso view tag |
| **-image** | 🐢🐢 | ⚠️ | ✅ | Image-based finding (Appium plugin) |

### 1. Accessibility ID — önerilen
```ruby
driver.find_element(:accessibility_id, "login_button")
```

**iOS karşılığı:** `view.accessibilityIdentifier = "login_button"` (developer set eder)
**Android karşılığı:** `view.contentDescription = "login_button"` (developer set eder)

**Lead bakışı:** Bu attribute zaten **accessibility için gerekli**. QA için ekstra bir şey değil — developer'larla anlaşman gerekir. Mülakat klasiği.

### 2. ID
```ruby
# Android — resource-id
driver.find_element(:id, "com.example.app:id/email_field")

# iOS — name attribute (limited)
driver.find_element(:id, "email_field")
```

### 3. Class Name
```ruby
buttons = driver.find_elements(:class, "android.widget.Button")    # Android
buttons = driver.find_elements(:class, "XCUIElementTypeButton")    # iOS
```

Çok generic — sayfada birden fazla butun varsa ambiguous.

### 4. XPath
```ruby
# Android
driver.find_element(:xpath, "//android.widget.Button[@text='Login']")
driver.find_element(:xpath, "//*[@content-desc='login_btn']")

# iOS
driver.find_element(:xpath, "//XCUIElementTypeButton[@name='Login']")
```

**Neden XPath kırılgan?**
- Hierarchy'ye duyarlı — küçük UI değişimi kırar
- Yavaş (10-100x daha yavaş accessibility_id'den)
- Index-based XPath (`//Button[3]`) en kırılgan

**Kural:** XPath son çare. Eğer kullanman gerekiyorsa **attribute-based** yaz (`@text=`, `@name=`), **path-based** yazma (`/hierarchy/...`).

### 5. UiAutomator (Android-only)

UiAutomator'un kendi selector dili — çok güçlü, hızlı.

```ruby
# Text-based
driver.find_element(
  :uiautomator,
  'new UiSelector().text("Login")'
)

# Text contains
driver.find_element(
  :uiautomator,
  'new UiSelector().textContains("Welcome")'
)

# Description
driver.find_element(
  :uiautomator,
  'new UiSelector().description("user_avatar")'
)

# Resource ID
driver.find_element(
  :uiautomator,
  'new UiSelector().resourceId("com.example:id/email")'
)

# Class + index
driver.find_element(
  :uiautomator,
  'new UiSelector().className("android.widget.Button").index(2)'
)

# Compound
driver.find_element(
  :uiautomator,
  'new UiSelector().className("android.widget.Button").text("Save")'
)

# Scroll into view (önemli!)
driver.find_element(
  :uiautomator,
  'new UiScrollable(new UiSelector().scrollable(true))' \
  '.scrollIntoView(new UiSelector().text("Settings"))'
)

# Clickable filter
driver.find_element(
  :uiautomator,
  'new UiSelector().clickable(true).text("OK")'
)
```

**Lead bakışı:** UiAutomator selector **XPath'ten 5-10x hızlı**. Android'de XPath kaçınmak için bir alternatif.

### 6. Predicate (iOS-only)

iOS Predicate strings — Cocoa style query.

```ruby
# Type + name
driver.find_element(
  :predicate,
  "type == 'XCUIElementTypeButton' AND name == 'Login'"
)

# Name CONTAINS
driver.find_element(
  :predicate,
  "type == 'XCUIElementTypeButton' AND name CONTAINS 'Login'"
)

# Label
driver.find_element(
  :predicate,
  "label == 'Sign in to continue'"
)

# Multiple conditions
driver.find_element(
  :predicate,
  "type == 'XCUIElementTypeButton' AND enabled == 1 AND visible == 1"
)

# Index-based
driver.find_element(
  :predicate,
  "type == 'XCUIElementTypeCell' AND name == 'product_row_0'"
)
```

### 7. Class Chain (iOS-only)

iOS Class Chain — XPath benzeri ama hızlı.

```ruby
# Type-based
driver.find_element(
  :class_chain,
  "**/XCUIElementTypeButton[`name == 'Login'`]"
)

# Index
driver.find_element(
  :class_chain,
  "**/XCUIElementTypeCell[1]"
)

# Compound
driver.find_element(
  :class_chain,
  "**/XCUIElementTypeButton[`label == 'Save' AND enabled == 1`]"
)

# Hierarchy
driver.find_element(
  :class_chain,
  "**/XCUIElementTypeTable/XCUIElementTypeCell[2]/XCUIElementTypeButton"
)
```

**Class Chain vs XPath:** Class chain XPath'ten **3-5x hızlı**, syntax benzer ama platform-specific.

### Locator önerilen sıra (Lead karar)
1. **accessibility_id** (en sağlam, en hızlı, cross-platform)
2. **id** (kararlı ama platform-specific)
3. **uiautomator** (Android), **predicate/class_chain** (iOS) — XPath alternatifi
4. **class name** (sadece tek bir tip varsa)
5. **xpath** (son çare, attribute-based yaz)

### Birden fazla element bulma
```ruby
buttons = driver.find_elements(:class, "android.widget.Button")
buttons.each(&:click)

items = driver.find_elements(:accessibility_id, "product_row")
items.size                          # element count
items[0].click                      # ilk element
```

### Inspector ile locator keşfi
Inspector açıkken:
1. Element'i seç → tüm attribute'ları gösterir (name, label, value, accessibility-id, content-desc, vb.)
2. "Find this element by" → her strateji için suggested locator
3. **"Best Match"** önerisi — accessibility_id, sonra id, sonra ne varsa

---

## 11. Element / Driver API

`appium_lib_core` Selenium WebDriver Ruby binding'ini kullanır. Element ve driver API'leri Selenium ile aynıdır.

### Driver API

```ruby
driver = Appium::Core.for(caps).start_driver

# Session info
driver.session_id
driver.capabilities

# Navigation (sadece WebView/browser)
driver.navigate.to("https://example.com")
driver.navigate.back
driver.navigate.forward
driver.navigate.refresh

# Page source
driver.page_source                       # XML hierarchy

# Screenshots
driver.screenshot_as(:base64)
driver.screenshot_as(:png)               # binary
driver.save_screenshot("tmp/debug.png")

# Window
driver.window_size                       # {width:, height:}
driver.window_handles                    # browser context'te

# Orientation
driver.rotation                          # current
driver.rotation = :landscape
driver.rotation = :portrait

# Quit
driver.quit
```

### Element finding

```ruby
# Tekil
el = driver.find_element(:accessibility_id, "btn")

# Çoklu
els = driver.find_elements(:class, "android.widget.Button")

# Element içinde sub-element
container = driver.find_element(:id, "list_container")
items = container.find_elements(:class, "android.widget.TextView")
```

### Element API — state sorgulama
```ruby
el.displayed?              # ekranda görünüyor mu
el.enabled?                # interactable mı
el.selected?               # checkbox/radio
el.location                # {x:, y:}
el.location_in_view        # scroll-aware
el.size                    # {width:, height:}
el.rect                    # {x:, y:, width:, height:}
el.tag_name                # "android.widget.Button"
```

### Element API — değer & attribute
```ruby
el.text                    # innerText
el.attribute("content-desc")
el.attribute("text")       # Android text
el.attribute("name")       # iOS name
el.attribute("label")      # iOS label
el.attribute("value")      # iOS value
el.attribute("enabled")    # bool
el.attribute("displayed")  # bool
el.attribute("focused")    # bool
el.attribute("checkable")  # Android
el.attribute("checked")    # Android
el.attribute("clickable")  # Android
el.attribute("focusable")
el.attribute("scrollable")
el.attribute("selected")
el.attribute("password")
el.attribute("resource-id") # Android
```

### Element API — action
```ruby
el.click
el.send_keys("text input")
el.send_keys("multi", "args", :enter)
el.clear
el.submit                  # form submit (limited)
```

### Driver-level keyboard
```ruby
# Android keycode
driver.press_keycode(4)              # AndroidKey.BACK
driver.press_keycode(3)              # AndroidKey.HOME
driver.press_keycode(187)            # AndroidKey.APP_SWITCH
driver.long_press_keycode(26)        # power

# Keycodes listesi: https://developer.android.com/reference/android/view/KeyEvent

# iOS keyboard
driver.hide_keyboard                 # close keyboard
driver.is_keyboard_shown
```

### Wait kullanımıyla element bulma
```ruby
wait = Selenium::WebDriver::Wait.new(timeout: 10)

el = wait.until { driver.find_element(:accessibility_id, "login") }

# Visibility wait
el = wait.until do
  e = driver.find_element(:accessibility_id, "btn")
  e if e.displayed?
end
```

---

## 12. Touch Gestures & W3C Actions

### W3C Actions API (Selenium 4 / WebDriver standard)

Selenium WebDriver 4'ün native action chain'i. Cross-driver çalışır.

```ruby
# Tap (touch click)
driver.action
      .move_to_location(100, 200)
      .pointer_down(:left)
      .pointer_up(:left)
      .perform

# Long press
driver.action
      .pointer_move(duration: 0, x: 100, y: 200)
      .pointer_down(:left)
      .pointer_move(duration: 2000, x: 100, y: 200)        # 2sn bekle
      .pointer_up(:left)
      .perform

# Swipe (drag)
driver.action
      .pointer_move(duration: 0, x: 500, y: 800)            # start
      .pointer_down(:left)
      .pointer_move(duration: 500, x: 500, y: 200)          # end (yukarı swipe)
      .pointer_up(:left)
      .perform

# Pinch zoom (multi-touch)
# İki finger oluştur
pointer1 = driver.action.devices.add_pointer_input(:touch, "finger1")
pointer2 = driver.action.devices.add_pointer_input(:touch, "finger2")
# ... composite action
```

### W3C action'lar zayıf yönü
W3C action API verbose ve düşük seviyeli. **Mobile gesture extensions** (`mobile:` commands) çok daha pratik (bölüm 13).

### Selenium-style touch (eski API, deprecated ama hâlâ çalışıyor)
```ruby
require "appium_lib_core"

# touch_action yerine W3C actions kullan
# touch_action deprecated, sadece referans için
```

### Element üzerinde gesture
```ruby
button = driver.find_element(:accessibility_id, "btn")

# Click
button.click

# Long press (W3C)
driver.action
      .move_to(button.native)
      .pointer_down(:left)
      .pause(duration: 2)
      .pointer_up(:left)
      .perform
```

---

## 13. Mobile Gesture Extensions (`mobile:`)

W3C action'lar yetmediğinde driver-specific kısa yollar. **`execute_script("mobile: ...", params)`** ile çağırılır.

### Tap
```ruby
# Koordinata tap
driver.execute_script("mobile: tap", { x: 100, y: 200 })

# Element merkezine tap
element = driver.find_element(:accessibility_id, "btn")
driver.execute_script("mobile: tap", { elementId: element.ref.last })
```

### Long press
```ruby
# Android
driver.execute_script("mobile: longClickGesture", {
  elementId: element.ref.last,
  duration:  2000          # ms
})

# iOS
driver.execute_script("mobile: touchAndHold", {
  element:  element.ref.last,
  duration: 2.0            # saniye
})
```

### Swipe
**Android (UiAutomator2):**
```ruby
# Element içinde swipe
driver.execute_script("mobile: swipeGesture", {
  elementId: scrollable_el.ref.last,
  direction: "up",         # up, down, left, right
  percent:   0.75          # mesafe yüzdesi
})

# Koordinata-based
driver.execute_script("mobile: swipeGesture", {
  left:      100,
  top:       500,
  width:     200,
  height:    800,
  direction: "up",
  percent:   0.75
})
```

**iOS (XCUITest):**
```ruby
# Element üzerinde
driver.execute_script("mobile: swipe", {
  element:   element.ref.last,
  direction: "left"         # up, down, left, right
})

# Belirli koordinata drag
driver.execute_script("mobile: dragFromToForDuration", {
  fromX:    100,
  fromY:    500,
  toX:      100,
  toY:      200,
  duration: 1.0
})
```

### Scroll
**Android:**
```ruby
driver.execute_script("mobile: scrollGesture", {
  left:      100,
  top:       100,
  width:     800,
  height:    1200,
  direction: "down",
  percent:   1.0
})

# Element içinde scroll
driver.execute_script("mobile: scrollGesture", {
  elementId: list_view.ref.last,
  direction: "down",
  percent:   2.0
})
```

**iOS:**
```ruby
driver.execute_script("mobile: scroll", {
  element:   list_view.ref.last,
  direction: "down"
})

# To predicate (text'i bulana kadar scroll)
driver.execute_script("mobile: scroll", {
  element:    list_view.ref.last,
  predicate: "label == 'Settings'"
})
```

### Pinch / Zoom
**Android:**
```ruby
driver.execute_script("mobile: pinchOpenGesture", {
  elementId: image.ref.last,
  percent:   0.75
})

driver.execute_script("mobile: pinchCloseGesture", {
  elementId: image.ref.last,
  percent:   0.75
})
```

**iOS:**
```ruby
driver.execute_script("mobile: pinch", {
  element: image.ref.last,
  scale:   2.0,        # > 1 = zoom in, < 1 = zoom out
  velocity: 1.0
})
```

### Double tap
**Android:**
```ruby
driver.execute_script("mobile: doubleClickGesture", {
  elementId: element.ref.last
})
```

**iOS:**
```ruby
driver.execute_script("mobile: doubleTap", {
  element: element.ref.last
})

# Koordinata
driver.execute_script("mobile: doubleTap", {
  x: 100, y: 200
})
```

### Drag & Drop
**Android:**
```ruby
driver.execute_script("mobile: dragGesture", {
  elementId: source.ref.last,
  endX:      500,
  endY:      800,
  speed:     2500
})
```

**iOS:**
```ruby
driver.execute_script("mobile: dragFromToForDuration", {
  element:  source.ref.last,
  duration: 1.0,
  fromX:    100,
  fromY:    500,
  toX:      500,
  toY:      800
})
```

### Helper module — gesture wrapper
```ruby
module GestureHelpers
  def tap(x:, y:)
    driver.execute_script("mobile: tap", { x: x, y: y })
  end

  def long_press(element, duration: 2000)
    if PLATFORM == :android
      driver.execute_script("mobile: longClickGesture", {
        elementId: element.ref.last,
        duration: duration
      })
    else
      driver.execute_script("mobile: touchAndHold", {
        element: element.ref.last,
        duration: duration / 1000.0
      })
    end
  end

  def swipe(direction:, element: nil)
    if element
      if PLATFORM == :android
        driver.execute_script("mobile: swipeGesture", {
          elementId: element.ref.last,
          direction: direction.to_s,
          percent: 0.75
        })
      else
        driver.execute_script("mobile: swipe", {
          element: element.ref.last,
          direction: direction.to_s
        })
      end
    else
      # screen-wide swipe
      size = driver.window_size
      case direction
      when :up    then swipe_by_coords(size, from_y_pct: 0.8, to_y_pct: 0.2)
      when :down  then swipe_by_coords(size, from_y_pct: 0.2, to_y_pct: 0.8)
      when :left  then swipe_by_coords(size, from_x_pct: 0.8, to_x_pct: 0.2)
      when :right then swipe_by_coords(size, from_x_pct: 0.2, to_x_pct: 0.8)
      end
    end
  end

  private

  def swipe_by_coords(size, from_x_pct: 0.5, to_x_pct: 0.5, from_y_pct: 0.5, to_y_pct: 0.5)
    driver.action
          .pointer_move(duration: 0, x: (size.width * from_x_pct).to_i, y: (size.height * from_y_pct).to_i)
          .pointer_down(:left)
          .pointer_move(duration: 500, x: (size.width * to_x_pct).to_i, y: (size.height * to_y_pct).to_i)
          .pointer_up(:left)
          .perform
  end
end

RSpec.configure { |c| c.include GestureHelpers }
```

---

## 14. App Lifecycle Yönetimi

### App start / stop
```ruby
# App'i aç (önceden yüklenmiş)
driver.activate_app("com.example.app")              # Android package / iOS bundle ID

# App'i kapat (force-stop)
driver.terminate_app("com.example.app")

# App durumu
driver.query_app_state("com.example.app")
# Android: 0=not installed, 1=not running, 3=in background, 4=in foreground
```

### Install / uninstall
```ruby
driver.install_app("/path/to/app.apk")
driver.install_app("/path/to/MyApp.app")
driver.install_app("/path/to/MyApp.ipa")

driver.remove_app("com.example.app")
driver.app_installed?("com.example.app")            # true/false

# Force reinstall
driver.install_app("/path/to/app.apk", replace: true)
```

### Background / foreground
```ruby
# App'i 5 saniye background'a al, sonra geri getir
driver.background_app(5)

# Süresiz background (manuel activate gerekir)
driver.background_app(-1)
driver.activate_app("com.example.app")
```

### Reset
```ruby
# noReset: true ile başlatıldıysa manuel reset
driver.reset                                         # app data sil + restart (deprecated)

# Modern alternatif
driver.terminate_app("com.example.app")
driver.activate_app("com.example.app")
```

### Activity / View management

**Android-specific:**
```ruby
# Current activity
driver.current_activity                              # ".MainActivity"
driver.current_package                               # "com.example.app"

# Specific activity başlat
driver.start_activity(
  app_package:    "com.example.app",
  app_activity:   ".SettingsActivity",
  app_wait_package: "com.example.app",
  app_wait_activity: ".SettingsActivity"
)

# Settings'i aç
driver.execute_script("mobile: shell", {
  command: "am",
  args:    ["start", "-a", "android.settings.SETTINGS"]
})
```

**iOS-specific:**
```ruby
# Bundle ID
driver.current_app                                   # bundleId döner

# Springboard'a dön (home screen)
driver.execute_script("mobile: pressButton", { name: "home" })
```

---

## 15. Device Interaction

### Hardware buttons
```ruby
# Android
driver.press_keycode(4)              # BACK
driver.press_keycode(3)              # HOME
driver.press_keycode(187)            # APP_SWITCH (recent apps)
driver.press_keycode(82)             # MENU
driver.press_keycode(24)             # VOLUME_UP
driver.press_keycode(25)             # VOLUME_DOWN
driver.press_keycode(26)             # POWER
driver.long_press_keycode(26)        # uzun basış

# iOS
driver.execute_script("mobile: pressButton", { name: "home" })
driver.execute_script("mobile: pressButton", { name: "volumeup" })
driver.execute_script("mobile: pressButton", { name: "volumedown" })
```

### Lock / unlock device
```ruby
driver.lock                          # device kilitle
driver.lock(5)                       # 5 saniye sonra otomatik unlock
driver.unlock                        # unlock
driver.device_locked?                # bool
```

### Orientation
```ruby
driver.rotation                       # :portrait / :landscape
driver.rotation = :landscape          # döndür
driver.rotation = :portrait
```

### Keyboard
```ruby
driver.hide_keyboard                  # klavyeyi kapat
driver.is_keyboard_shown              # bool

# Android-specific
driver.hide_keyboard(:key_name, "Done")
driver.hide_keyboard(:strategy, "tapOutside")

# iOS-specific
driver.hide_keyboard("Done")          # Done button'a tıkla
```

### Screen brightness, volume, etc. (Android)
```ruby
driver.execute_script("mobile: shell", {
  command: "settings",
  args:    ["put", "system", "screen_brightness", "128"]
})
```

### Time zone, language
```ruby
# Capability ile (start time)
caps = {
  "appium:language": "tr",
  "appium:locale":   "TR",
  "appium:tz":       "Europe/Istanbul"
}
```

### Screenshots
```ruby
# PNG binary
png_data = driver.screenshot_as(:png)

# Base64
b64 = driver.screenshot_as(:base64)

# File'a kaydet
driver.save_screenshot("tmp/screenshot.png")
```

### Page source (UI hierarchy)
```ruby
xml = driver.page_source              # XML formatında tüm UI tree

# Kaydet
File.write("tmp/page_source.xml", xml)
```

### Device info
```ruby
driver.device_time                    # device clock
driver.battery_info                   # {state:, level:}

# Android
driver.execute_script("mobile: deviceInfo")    # detailed info hash

# iOS
driver.execute_script("mobile: deviceInfo")
```

### ADB shell (Android)
```ruby
# Generic shell command
output = driver.execute_script("mobile: shell", {
  command: "ls",
  args:    ["/sdcard/"]
})

# Install permission
driver.execute_script("mobile: shell", {
  command: "pm",
  args:    ["grant", "com.example.app", "android.permission.CAMERA"]
})

# Network simulation
driver.execute_script("mobile: shell", {
  command: "svc",
  args:    ["wifi", "disable"]
})
```

---

## 16. Wait Stratejileri

**Appium kendi auto-wait sağlamaz** (Capybara'dan farklı olarak). Selenium WebDriver Wait kullanılır.

### Implicit Wait (önerilmez)
```ruby
driver.manage.timeouts.implicit_wait = 10

# Driver tüm find_element'larda max 10sn bekler
# Sorun: explicit ile karıştığında unpredictable
```

**Kural:** Implicit wait **kullanma**. Sadece explicit.

### Explicit Wait — `Selenium::WebDriver::Wait`
```ruby
wait = Selenium::WebDriver::Wait.new(timeout: 10, interval: 0.5)

# Element bulunana kadar bekle
el = wait.until { driver.find_element(:accessibility_id, "btn") }

# Visible olana kadar
el = wait.until do
  e = driver.find_element(:accessibility_id, "btn")
  e if e.displayed?
end

# Enabled
el = wait.until do
  e = driver.find_element(:accessibility_id, "btn")
  e if e.displayed? && e.enabled?
end

# Text içerene kadar
wait.until do
  driver.find_element(:accessibility_id, "msg").text.include?("Welcome")
end

# Element kaybolana kadar
wait.until do
  begin
    !driver.find_element(:accessibility_id, "spinner").displayed?
  rescue Selenium::WebDriver::Error::NoSuchElementError
    true
  end
end
```

### FluentWait (custom polling, exception ignore)
```ruby
wait = Selenium::WebDriver::Wait.new(
  timeout:  30,
  interval: 0.5,
  ignore:   [
    Selenium::WebDriver::Error::NoSuchElementError,
    Selenium::WebDriver::Error::StaleElementReferenceError
  ]
)

el = wait.until { driver.find_element(:accessibility_id, "dynamic_btn") }
```

### Helper modülü — clean
```ruby
# spec/support/helpers/wait_helpers.rb
module WaitHelpers
  DEFAULT_TIMEOUT = 10

  def wait_for(timeout: DEFAULT_TIMEOUT, interval: 0.5, ignore: [])
    Selenium::WebDriver::Wait
      .new(timeout: timeout, interval: interval, ignore: default_ignores + ignore)
      .until { yield }
  end

  def wait_visible(locator_type, locator_value, timeout: DEFAULT_TIMEOUT)
    wait_for(timeout: timeout) do
      el = driver.find_element(locator_type, locator_value)
      el if el.displayed?
    end
  end

  def wait_clickable(locator_type, locator_value, timeout: DEFAULT_TIMEOUT)
    wait_for(timeout: timeout) do
      el = driver.find_element(locator_type, locator_value)
      el if el.displayed? && el.enabled?
    end
  end

  def wait_invisible(locator_type, locator_value, timeout: DEFAULT_TIMEOUT)
    wait_for(timeout: timeout) do
      begin
        !driver.find_element(locator_type, locator_value).displayed?
      rescue Selenium::WebDriver::Error::NoSuchElementError
        true
      end
    end
  end

  def wait_text(locator_type, locator_value, text, timeout: DEFAULT_TIMEOUT)
    wait_for(timeout: timeout) do
      driver.find_element(locator_type, locator_value).text.include?(text)
    end
  end

  private

  def default_ignores
    [
      Selenium::WebDriver::Error::NoSuchElementError,
      Selenium::WebDriver::Error::StaleElementReferenceError,
      Selenium::WebDriver::Error::InvalidElementStateError
    ]
  end
end

RSpec.configure { |c| c.include WaitHelpers }
```

### Page Object'lerde kullanım
```ruby
class LoginPage < BasePage
  EMAIL_FIELD = [:accessibility_id, "email_input"].freeze

  def enter_email(email)
    wait_visible(*EMAIL_FIELD).send_keys(email)
    self
  end
end
```

### Kural
- **Implicit wait yasak**
- Her interaction öncesi **explicit wait** ile element'in clickable/visible olduğunu garanti et
- `sleep` **antipattern**'dir — sadece debug, asla CI
- Helper module ile DRY

---

## 17. Context — Native / Hybrid / Web

Mobile app'lerin 3 tipi:

| Tip | Açıklama | Test stratejisi |
|---|---|---|
| **Native** | Platform UI framework'üyle yazılmış (Swift/Kotlin/Java) | Native locator'lar, native driver |
| **Hybrid** | WebView içinde HTML render eden native app (Cordova, Ionic, Capacitor) | Context switching (NATIVE_APP ↔ WEBVIEW), iki tarafı da test et |
| **Mobile Web** | Mobile Chrome/Safari'de açılan responsive web | Selenium-style; `browserName: Chrome/Safari` |

### Context switching
```ruby
# Mevcut context
driver.context                          # "NATIVE_APP"

# Tüm context'ler
driver.available_contexts
# => ["NATIVE_APP", "WEBVIEW_com.example.app"]

# WebView'a geç
driver.set_context("WEBVIEW_com.example.app")

# WebView'da CSS selector kullan
driver.find_element(:css, "button#login").click
driver.find_element(:tag_name, "input").send_keys("text")

# Native'e dön
driver.set_context("NATIVE_APP")
```

### Hybrid debugging
- **Android WebView:** Chrome DevTools `chrome://inspect` ile inspect edilebilir
- **iOS WebView:** Safari → Develop menu → device → WebView

### Mobile Web test setup

**Android Chrome:**
```ruby
caps = {
  platformName: "Android",
  "appium:platformVersion": "13",
  "appium:deviceName": "Pixel_6_API_33",
  "appium:automationName": "UiAutomator2",
  browserName: "Chrome"
}

driver = Appium::Core.for(caps: caps).start_driver
driver.navigate.to("https://example.com")
driver.find_element(:css, "input#email").send_keys("x@y.com")
```

**iOS Safari:**
```ruby
caps = {
  platformName: "iOS",
  "appium:platformVersion": "17.0",
  "appium:deviceName": "iPhone 15",
  "appium:automationName": "XCUITest",
  browserName: "Safari"
}
```

### Hybrid app testi — örnek
```ruby
it "completes hybrid checkout flow" do
  # Native UI'da
  driver.find_element(:accessibility_id, "shop_tab").click

  # WebView ürün listesi
  driver.set_context("WEBVIEW_com.example.shop")
  driver.find_element(:css, ".product[data-sku='ABC']").click
  driver.find_element(:css, "button.add-to-cart").click

  # Native cart icon'a dön
  driver.set_context("NATIVE_APP")
  driver.find_element(:accessibility_id, "cart_icon").click

  # Native checkout
  driver.find_element(:accessibility_id, "checkout_btn").click
end
```

### Chromedriver mismatch — yaygın hata
Hybrid app'in WebView'ı için Chromedriver gerekir. Appium auto-download yapar ama version mismatch sıkça olur.

```ruby
caps = {
  # ...
  "appium:chromedriverExecutable": "/path/to/chromedriver",
  "appium:chromedriverAutodownload": true
}
```

---

## 18. Network, Geolocation, Notifications

### Network simulation
```ruby
# Android — network state set
# 0 = no connection, 1 = airplane mode, 2 = wifi only, 4 = data only, 6 = all
driver.network_connection = 4         # data only

# Toggle
driver.execute_script("mobile: setConnectivity", { wifi: false, data: true })

# Cloud provider'lar daha gelişmiş simülasyon sunar
# LambdaTest: 3g, 4g, 5g, no_network, offline
```

### Geolocation
```ruby
# Set device location
driver.set_location(latitude: 41.0082, longitude: 28.9784)    # Istanbul

# iOS'ta capability ile
caps = {
  # ...
  "appium:latitude":  41.0082,
  "appium:longitude": 28.9784
}

# Cloud provider'lar:
caps = {
  "lt:options" => {
    geoLocation: "TR"
  }
}
```

### Push notifications
**Android (UiAutomator2):**
```ruby
# Notification shade aç
driver.execute_script("mobile: shell", {
  command: "cmd",
  args:    ["statusbar", "expand-notifications"]
})

# Veya
driver.open_notifications

# Geri kapat
driver.press_keycode(4)               # BACK
```

**iOS:**
```ruby
# Notification simülasyonu sınırlı, genelde push'u kendi test backend'inden tetiklersin
# veya cloud provider'ın specific API'sini kullanırsın
```

### Test push'u manuel tetikleme
```ruby
# Firebase Test FCM
require "googleauth"

def send_test_notification(device_token, title:, body:)
  client = Google::Apis::FcmV1::FirebaseCloudMessagingService.new
  # ... FCM message construction
end

# Test
send_test_notification(device_token, title: "Test", body: "Hello")
driver.execute_script("mobile: shell", { command: "...", args: [...] })   # şade aç
driver.find_element(:accessibility_id, "Test").click       # notification'a tap
```

---

## 19. Permission Management

### Auto-grant tüm permission'lar (Android)
```ruby
caps = {
  # ...
  "appium:autoGrantPermissions": true     # tüm runtime permission'ları auto-grant
}
```

### Spesifik permission grant (Android)
```ruby
driver.execute_script("mobile: changePermissions", {
  action:      "grant",                   # veya "revoke"
  permissions: ["android.permission.CAMERA",
                "android.permission.ACCESS_FINE_LOCATION"],
  appPackage:  "com.example.app"
})

# Yeni Android — special permission'lar
driver.execute_script("mobile: changePermissions", {
  action:      "grant",
  permissions: ["all"],
  appPackage:  "com.example.app"
})
```

### Permission durumu sorgu
```ruby
permissions = driver.execute_script("mobile: getPermissions", {
  type:       "denied",                    # granted, denied, requested
  appPackage: "com.example.app"
})
```

### iOS permission auto-accept
```ruby
caps = {
  # ...
  "appium:autoAcceptAlerts": true        # tüm sistem alert'lerini accept et
}
```

iOS'ta runtime permission Android gibi granular grant API'si yok — `autoAcceptAlerts` ile alert geldiğinde otomatik tıklanır.

### Permission dialog handle
**Android (UiAutomator2 dialog için):**
```ruby
allow_btn = driver.find_element(:id, "com.android.permissioncontroller:id/permission_allow_button")
allow_btn.click
```

**iOS:**
```ruby
# Alert handling
driver.switch_to.alert.accept       # OK
driver.switch_to.alert.dismiss      # Cancel
driver.switch_to.alert.text         # alert text

# Veya mobile: alert
driver.execute_script("mobile: alert", { action: "accept" })
driver.execute_script("mobile: alert", { action: "dismiss" })
driver.execute_script("mobile: alert", { action: "getButtons" })
```

---

## 20. Deep Links & App URL Schemes

### Deep link açma
**Android:**
```ruby
driver.execute_script("mobile: deepLink", {
  url:     "myapp://product/123",
  package: "com.example.app"
})

# Alternatif — ADB
driver.execute_script("mobile: shell", {
  command: "am",
  args:    ["start", "-W", "-a", "android.intent.action.VIEW",
            "-d", "myapp://product/123", "com.example.app"]
})
```

**iOS:**
```ruby
driver.execute_script("mobile: deepLink", {
  url:    "myapp://product/123",
  bundleId: "com.example.app"
})

# Alternatif (eski)
driver.get("myapp://product/123")
```

### Use case: Bypass login via deep link
```ruby
# Login screen'i atla, doğrudan dashboard'a
def bypass_login_with_deep_link(user_token)
  url = "myapp://auth?token=#{user_token}"
  driver.execute_script("mobile: deepLink", { url: url, package: "com.example.app" })
end

# Test'te
it "skips login for speed" do
  bypass_login_with_deep_link(generate_test_token(create(:user)))
  # Doğrudan dashboard
  expect(driver.find_element(:accessibility_id, "dashboard_title")).to be_displayed
end
```

**Bu pattern test süresini 5-10 saniye kısaltır** (her test için UI'dan login yapmıyorsun).

### Universal Links (iOS) ve App Links (Android)
HTTPS deep link'ler:
```ruby
driver.execute_script("mobile: deepLink", {
  url:    "https://example.com/product/123",
  bundleId: "com.example.app"
})
```

App handler register edilmişse direkt app'te açılır.

---

## 21. Performance Metrics

Appium native API ile bazı perf metric'leri toplanabilir.

### Android performance data
```ruby
# Available data types
data_types = driver.execute_script("mobile: deviceInfo")

# CPU usage
cpu = driver.execute_script("mobile: getPerformanceData", {
  packageName: "com.example.app",
  dataType:    "cpuinfo",
  dataReadTimeout: 5
})

# Memory
memory = driver.execute_script("mobile: getPerformanceData", {
  packageName: "com.example.app",
  dataType:    "memoryinfo",
  dataReadTimeout: 5
})

# Battery
battery = driver.execute_script("mobile: getPerformanceData", {
  packageName: "com.example.app",
  dataType:    "batteryinfo"
})

# Network
network = driver.execute_script("mobile: getPerformanceData", {
  packageName: "com.example.app",
  dataType:    "networkinfo"
})
```

### iOS performance
Apple'ın resmi performance API'si Appium üzerinden sınırlıdır. Genelde Instruments / Xcode profiler tercih edilir.

### Use case: Memory leak detection
```ruby
it "doesn't leak memory on repeated actions" do
  initial = parse_memory(driver.execute_script("mobile: getPerformanceData",
    { packageName: "com.example.app", dataType: "memoryinfo" }
  ))

  50.times do
    driver.find_element(:accessibility_id, "open_modal").click
    driver.find_element(:accessibility_id, "close_modal").click
  end

  final = parse_memory(driver.execute_script("mobile: getPerformanceData",
    { packageName: "com.example.app", dataType: "memoryinfo" }
  ))

  expect(final).to be < (initial * 1.5)    # max %50 growth
end
```

### Logcat (Android)
```ruby
logs = driver.manage.logs.get(:logcat)
logs.each { |log| puts "#{log.level}: #{log.message}" }
```

---

## 22. Page Object Model — Mobile

### Temel POM
```ruby
# spec/support/pages/base_page.rb
class BasePage
  attr_reader :driver, :wait

  def initialize(driver)
    @driver = driver
    @wait   = Selenium::WebDriver::Wait.new(timeout: 10, interval: 0.5)
  end

  def loaded?
    raise NotImplementedError
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

  def wait_invisible(type, value)
    wait.until do
      begin
        !driver.find_element(type, value).displayed?
      rescue Selenium::WebDriver::Error::NoSuchElementError
        true
      end
    end
  end
end

# spec/support/pages/login_page.rb
class LoginPage < BasePage
  EMAIL_FIELD    = [:accessibility_id, "email_input"].freeze
  PASSWORD_FIELD = [:accessibility_id, "password_input"].freeze
  LOGIN_BTN      = [:accessibility_id, "login_button"].freeze
  ERROR_MSG      = [:accessibility_id, "error_msg"].freeze
  SIGNUP_LINK    = [:accessibility_id, "signup_link"].freeze

  def loaded?
    wait_visible(*EMAIL_FIELD).displayed?
  end

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

  def login_as(email:, password:)
    enter_email(email).enter_password(password).submit
  end

  def error_message
    wait_visible(*ERROR_MSG).text
  end

  def go_to_signup
    wait_clickable(*SIGNUP_LINK).click
    SignupPage.new(driver)
  end
end
```

### Test
```ruby
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

  it "yanlış şifre hatası gösterir" do
    login_page.login_as(email: "test@x.com", password: "wrong")

    expect(login_page.error_message).to include("Invalid")
  end
end
```

### Component pattern
```ruby
class DashboardPage < BasePage
  def header
    HeaderComponent.new(driver, wait_visible(:accessibility_id, "header"))
  end

  def tab_bar
    TabBarComponent.new(driver, wait_visible(:accessibility_id, "tab_bar"))
  end

  def loaded?
    wait_visible(:accessibility_id, "dashboard_title").displayed?
  end
end

class TabBarComponent
  def initialize(driver, container)
    @driver = driver
    @container = container
  end

  def go_to_profile
    @container.find_element(:accessibility_id, "tab_profile").click
    ProfilePage.new(@driver)
  end

  def go_to_settings
    @container.find_element(:accessibility_id, "tab_settings").click
    SettingsPage.new(@driver)
  end
end

# Test
profile = dashboard.tab_bar.go_to_profile
```

### Lazy element pattern
```ruby
class LoginPage < BasePage
  def email_field
    wait_visible(:accessibility_id, "email_input")
  end

  def password_field
    wait_visible(:accessibility_id, "password_input")
  end

  def submit_btn
    wait_clickable(:accessibility_id, "login_button")
  end

  def login(email, password)
    email_field.send_keys(email)
    password_field.send_keys(password)
    submit_btn.click
    DashboardPage.new(driver)
  end
end
```

**Avantaj:** Her method çağrısında element yeniden lookup → stale element exception riski yok.

---

## 23. Cross-Platform POM Stratejisi

Aynı UX, iki platformda. Tek POM iki driver'a hizmet etmeli.

### Yaklaşım 1: `locator` class macro DSL
```ruby
# spec/support/pages/base_page.rb
class BasePage
  PLATFORM = ENV.fetch("PLATFORM", "android").to_sym

  attr_reader :driver, :wait

  def initialize(driver)
    @driver = driver
    @wait   = Selenium::WebDriver::Wait.new(timeout: 10)
  end

  # Class-level DSL
  def self.locator(name, android:, ios:)
    define_method(name) do
      type, value = PLATFORM == :android ? android : ios
      wait.until do
        el = driver.find_element(type, value)
        el if el.displayed?
      end
    end
  end
end

# spec/support/pages/login_page.rb
class LoginPage < BasePage
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

### Yaklaşım 2: Subclass per platform (Strategy pattern)
```ruby
class LoginPage < BasePage
  def self.for(driver)
    case PLATFORM
    when :android then AndroidLoginPage.new(driver)
    when :ios     then IOSLoginPage.new(driver)
    end
  end

  def login(email:, password:)
    enter_email(email)
    enter_password(password)
    submit
    DashboardPage.for(driver)
  end
end

class AndroidLoginPage < LoginPage
  EMAIL_FIELD = [:accessibility_id, "email_input"].freeze
  SUBMIT_BTN  = [:uiautomator, 'new UiSelector().text("Login")'].freeze

  private

  def enter_email(email)
    wait_visible(*EMAIL_FIELD).send_keys(email)
  end

  # ...
end

class IOSLoginPage < LoginPage
  EMAIL_FIELD = [:accessibility_id, "email_input"].freeze
  SUBMIT_BTN  = [:class_chain, '**/XCUIElementTypeButton[`name == "Login"`]'].freeze

  private

  def enter_email(email)
    wait_visible(*EMAIL_FIELD).send_keys(email)
  end
end
```

**Kullanım:**
```ruby
login_page = LoginPage.for(driver)
login_page.login(email: "x@y.com", password: "secret")
```

### Yaklaşım 3: YAML config + selector lookup
```yaml
# config/selectors/login_page.yml
android:
  email_field:
    type: accessibility_id
    value: email_input
  submit_btn:
    type: uiautomator
    value: 'new UiSelector().text("Login")'

ios:
  email_field:
    type: accessibility_id
    value: email_input
  submit_btn:
    type: class_chain
    value: '**/XCUIElementTypeButton[`name == "Login"`]'
```

```ruby
class LoginPage < BasePage
  SELECTORS = YAML.load_file("config/selectors/login_page.yml")[PLATFORM.to_s]

  def locator(name)
    config = SELECTORS[name.to_s]
    [config["type"].to_sym, config["value"]]
  end

  def email_field
    wait_visible(*locator(:email_field))
  end
end
```

**Avantaj:** Non-Ruby team (designer, PO) selector'ları YAML'da güncelleyebilir.

### Lead karar matrix
| Yaklaşım | Avantaj | Dezavantaj | Ne zaman |
|---|---|---|---|
| `locator` macro | Kompakt, okunabilir | Inline | %80 senaryo (önerilen) |
| Subclass per platform | Tam ayrı behavior gerekirse | Code duplication | Platform UX'i çok farklıysa |
| YAML config | Non-tech team güncelleyebilir | Indirection | Locator'lar sık değişiyorsa |

---

## 24. Cloud Device Farms

Lokal emulator/simulator yerine **cloud device farm**'lar tercih edilir.

### Karşılaştırma

| Provider | Real device | Emulator | Avantaj | Fiyat |
|---|---|---|---|---|
| **BrowserStack App Automate** | ✅ 3000+ | ✅ | En mature, geniş device | Pahalı |
| **Sauce Labs** | ✅ | ✅ | Enterprise odaklı, DevOps | Pahalı |
| **LambdaTest** | ✅ | ✅ | Uygun fiyat, hızlı | Orta |
| **AWS Device Farm** | ✅ | ❌ | AWS ekosistemi | Orta |
| **Firebase Test Lab** | ✅ Android only | ✅ | Google ekosistemi | Ucuz |
| **HeadSpin** | ✅ | ❌ | Network/perf odaklı | Pahalı |
| **Perfecto** | ✅ | ✅ | Enterprise | Pahalı |

### LambdaTest setup (Ruby)
```ruby
caps = {
  caps: {
    platformName: "Android",
    "appium:deviceName": "Galaxy S22",
    "appium:platformVersion": "13",
    "appium:app": "lt://APP123456",   # LT'ye yüklenmiş app
    "lt:options" => {
      user:           ENV.fetch("LT_USERNAME"),
      accessKey:      ENV.fetch("LT_ACCESS_KEY"),
      build:          "Release-#{Date.today}",
      name:           "Login flow test",
      visual:         true,            # visual regression
      network:        true,            # network logs
      video:          true,
      console:        true,
      tunnel:         false,
      tunnelName:     "myTunnel",
      idleTimeout:    180,
      isRealMobile:   true,            # real device
      geoLocation:    "TR",
      app_url:        "lt://APP123456"
    }
  },
  appium_lib: {
    server_url: "https://mobile-hub.lambdatest.com/wd/hub"
  }
}

driver = Appium::Core.for(caps).start_driver
```

### BrowserStack setup
```ruby
caps = {
  caps: {
    platformName: "Android",
    "appium:deviceName": "Samsung Galaxy S22",
    "appium:platformVersion": "13.0",
    "appium:app": "bs://abc123...",          # bs://uploaded
    "bstack:options" => {
      userName:     ENV.fetch("BROWSERSTACK_USER"),
      accessKey:    ENV.fetch("BROWSERSTACK_KEY"),
      projectName:  "MyApp",
      buildName:    "Build #{ENV['CI_BUILD']}",
      sessionName:  "Login flow",
      local:        false,
      debug:        true,
      networkLogs:  true,
      consoleLogs:  "verbose",
      appiumLogs:   true,
      deviceLogs:   true,
      video:        true,
      idleTimeout:  180
    }
  },
  appium_lib: {
    server_url: "http://hub-cloud.browserstack.com/wd/hub"
  }
}
```

### App upload — BrowserStack
```bash
curl -u "$BROWSERSTACK_USER:$BROWSERSTACK_KEY" \
  -X POST "https://api-cloud.browserstack.com/app-automate/upload" \
  -F "file=@/path/to/app.apk"

# Response: {"app_url": "bs://abc123..."}
```

### Local tunneling — staging app'i test etme
Cloud device → company staging server (private) bağlantısı için tunnel:

**BrowserStack Local:**
```bash
# Install
brew install --cask browserstacklocal

# Run
BrowserStackLocal --key $BROWSERSTACK_KEY
```

Sonra:
```ruby
"bstack:options" => {
  local: true,
  # ...
}
```

**Sauce Labs Connect:**
```bash
sc -u $SAUCE_USERNAME -k $SAUCE_ACCESS_KEY
```

**LambdaTest Tunnel:**
```bash
./LT --user $LT_USERNAME --key $LT_ACCESS_KEY --tunnelName "myTunnel"
```

### Parallel cloud testing
Cloud farm'lar tipik olarak **paralel session limit** sunar (örn. 5 paralel session). `parallel_tests` ile birleştir:

```bash
bundle exec parallel_rspec spec/system -n 5
```

Her process kendi cloud session'ı açar.

### Cloud reporting
- **Video recording** — her test session'ı kaydedilir
- **Network logs** — request/response logu
- **Device logs** — logcat (Android), syslog (iOS)
- **Console logs** — JS console
- **Screenshot at failure** — otomatik
- **Performance metrics** — bazı provider'lar

Dashboard URL test çıktısına eklenmeli.

---

## 25. CI/CD Entegrasyonu

### GitHub Actions — Lokal emulator
```yaml
name: Android E2E
on: [push, pull_request]

jobs:
  android-test:
    runs-on: macos-latest        # macOS KVM gerektirir
    strategy:
      matrix:
        api-level: [29, 33]
    steps:
      - uses: actions/checkout@v4

      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install Appium
        run: |
          npm install -g appium
          appium driver install uiautomator2

      - name: AVD cache
        uses: actions/cache@v4
        with:
          path: |
            ~/.android/avd/*
            ~/.android/adb*
          key: avd-${{ matrix.api-level }}

      - name: Run Appium server
        run: appium server --port 4723 &

      - name: Run tests on emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: ${{ matrix.api-level }}
          target: google_apis
          arch: x86_64
          script: |
            PLATFORM=android bundle exec rspec spec/system \
              --format RspecJunitFormatter --out tmp/junit.xml

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: android-failures-api${{ matrix.api-level }}
          path: |
            tmp/screenshots/
            tmp/junit.xml
```

### GitHub Actions — LambdaTest (cloud)
```yaml
name: Mobile E2E (Cloud)
on: [push]

jobs:
  android-cloud:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Upload app to LambdaTest
        run: |
          UPLOAD_RESPONSE=$(curl -u "$LT_USERNAME:$LT_ACCESS_KEY" \
            -X POST "https://manual-api.lambdatest.com/app/upload/realDevice" \
            -F "appFile=@apps/app-release.apk")
          APP_URL=$(echo $UPLOAD_RESPONSE | jq -r '.app_url')
          echo "APP_URL=$APP_URL" >> $GITHUB_ENV
        env:
          LT_USERNAME: ${{ secrets.LT_USERNAME }}
          LT_ACCESS_KEY: ${{ secrets.LT_ACCESS_KEY }}

      - name: Run tests
        env:
          PLATFORM: android
          USE_CLOUD: true
          LT_APP_URL: ${{ env.APP_URL }}
          LT_USERNAME: ${{ secrets.LT_USERNAME }}
          LT_ACCESS_KEY: ${{ secrets.LT_ACCESS_KEY }}
        run: |
          bundle exec parallel_rspec spec/system -n 5

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: cloud-results
          path: tmp/allure-results/
```

### iOS CI (Mac runner gerekli)
```yaml
ios-test:
  runs-on: macos-14         # Apple Silicon, fast
  steps:
    - uses: actions/checkout@v4
    - uses: ruby/setup-ruby@v1
      with: { bundler-cache: true }

    - name: Select Xcode
      run: sudo xcode-select -s /Applications/Xcode_15.0.app

    - uses: actions/setup-node@v4

    - name: Install Appium
      run: |
        npm install -g appium
        appium driver install xcuitest

    - name: Boot simulator
      run: |
        xcrun simctl boot "iPhone 15" || true
        xcrun simctl bootstatus "iPhone 15" -b

    - name: Run Appium
      run: appium server --port 4723 &

    - name: Run tests
      env:
        PLATFORM: ios
      run: bundle exec rspec spec/system
```

### Jenkins pipeline
```groovy
pipeline {
    agent { label 'mobile-test' }
    parameters {
        choice(name: 'PLATFORM', choices: ['android', 'ios'])
        choice(name: 'TARGET',   choices: ['emulator', 'lambdatest'])
    }
    stages {
        stage('Setup') {
            steps {
                sh 'bundle install'
                sh 'npm install -g appium'
                sh "appium driver install ${params.PLATFORM == 'android' ? 'uiautomator2' : 'xcuitest'}"
            }
        }
        stage('Run Tests') {
            steps {
                sh """
                  appium server --port 4723 &
                  sleep 5
                  PLATFORM=${params.PLATFORM} USE_CLOUD=${params.TARGET == 'lambdatest'} \
                    bundle exec rspec spec/system
                """
            }
        }
    }
    post {
        always {
            junit 'tmp/junit.xml'
            archiveArtifacts artifacts: 'tmp/screenshots/**', allowEmptyArchive: true
        }
    }
}
```

### Paralel test stratejisi
```ruby
# spec/support/appium_session.rb
class AppiumSession
  def self.driver
    Thread.current[:appium_driver] ||= start_driver
  end

  def self.start_driver
    worker = ENV["TEST_ENV_NUMBER"].to_i   # parallel_tests: "", "2", "3", ...
    port   = 4723 + (worker.zero? ? 0 : worker - 1)
    caps   = CapsBuilder.build(platform: PLATFORM, server_port: port)
    Appium::Core.for(caps).start_driver
  end
end
```

```bash
# Lokal: her process kendi Appium server + emulator
bundle exec parallel_rspec spec/system -n 4

# Cloud: her process kendi cloud session (limit'e dikkat)
USE_CLOUD=true bundle exec parallel_rspec spec/system -n 5
```

---

## 26. Reporting

### RSpec formatter'lar
```bash
bundle exec rspec --format documentation
bundle exec rspec --format RspecJunitFormatter --out tmp/junit.xml
```

### Allure (en yaygın)
```ruby
gem "allure-rspec"
```

```ruby
# spec/spec_helper.rb
require "allure-rspec"

AllureRspec.configure do |c|
  c.results_directory         = "tmp/allure-results"
  c.clean_results_directory   = true
  c.environment_properties    = {
    platform:       PLATFORM,
    appium_version: "2.x",
    device:         AppiumSession.driver.capabilities[:device_name]
  }
end
```

### Screenshot embedding
```ruby
RSpec.configure do |c|
  c.after(:each, :appium) do |example|
    if example.exception
      png = AppiumSession.driver.screenshot_as(:png)
      Allure.add_attachment(
        name:   "Failure screenshot",
        source: png,
        type:   Allure::ContentType::PNG
      )

      # Page source XML
      Allure.add_attachment(
        name:   "Page source",
        source: AppiumSession.driver.page_source,
        type:   "application/xml"
      )
    end
  end
end
```

### Allure dashboard generate
```bash
bundle exec rspec --format AllureRspecFormatter
allure generate tmp/allure-results --clean -o tmp/allure-html
allure open tmp/allure-html
```

### Custom HTML reporter
```ruby
gem "rspec_html_reporter"
```

```bash
bundle exec rspec --format RspecHtmlReporter
# → reports/rspec_report.html
```

### Cloud native reporting
LambdaTest / BrowserStack / Sauce dashboard'larında **otomatik** olarak:
- Video recording
- Network logs
- Device logs (logcat / syslog)
- Console logs
- App logs (LambdaTest)
- Screenshot at failure
- Step-by-step timeline

Her test session'ın dashboard URL'i otomatik üretilir; CI çıktısına embed et:
```ruby
puts "Dashboard: https://automation.lambdatest.com/test?testID=#{driver.session_id}"
```

---

## 27. Debugging Araçları

### Appium Inspector
```bash
appium-inspector            # CLI
# veya GUI: appium-inspector.io
```

- UI hierarchy'i göster
- Element seç → tüm attribute'lar
- Best locator önerisi
- Action recording → kod export

### Page source dump
```ruby
puts driver.page_source
File.write("tmp/page_source.xml", driver.page_source)
```

XML hierarchy'i parse ederek custom element finder'lar yazılabilir.

### Screenshot ön debug
```ruby
driver.save_screenshot("tmp/before_action.png")
button.click
driver.save_screenshot("tmp/after_action.png")
```

### `binding.pry` ile interactive
```ruby
require "pry-byebug"

it "interactive debug" do
  visit_login
  binding.pry             # burada interactive console
  fill_in_email("x@y.com")
end
```

Pry içinde:
```
pry> driver.page_source
pry> driver.find_elements(:class, "android.widget.Button").map(&:text)
pry> driver.save_screenshot("tmp/debug.png")
```

### Logcat (Android)
```ruby
# Test sırasında
logs = driver.manage.logs.get(:logcat)
logs.each { |log| puts "#{log.level}: #{log.message}" }

# CLI
adb logcat com.example.app
adb logcat -s "MyAppTag:V"
```

### Syslog (iOS simulator)
```bash
xcrun simctl spawn booted log stream --predicate 'subsystem == "com.example.MyApp"'
```

### Appium server logs
```bash
# Verbose
appium server --port 4723 --log-level debug

# File output
appium server --port 4723 --log /tmp/appium.log
```

Appium server log'ları **çok ayrıntılı** — flaky test debug'da kritik.

### ADB (Android debug bridge)
```bash
# Device list
adb devices

# Shell
adb shell

# Install app
adb install -r app.apk

# Uninstall
adb uninstall com.example.app

# Logcat with filter
adb logcat com.example.app:V "*:S"

# Screen record
adb shell screenrecord /sdcard/test.mp4
# (Ctrl+C to stop)
adb pull /sdcard/test.mp4
```

### Chrome DevTools — Android WebView
1. Chrome aç: `chrome://inspect`
2. Device device list'te göründüğünde "inspect" tıkla
3. DevTools açılır, WebView'ı debug et

### Safari Web Inspector — iOS WebView
1. iOS device/simulator'da Settings → Safari → Advanced → Web Inspector ON
2. macOS Safari → Develop menu → device → WebView

### Trace mode (Appium server)
```bash
appium server --port 4723 --log-level debug:debug \
  --log /tmp/appium-trace.log
```

Tüm WebDriver komutları, response'lar, gecikme süreleri logged.

---

## 28. Test Türleri Matrix'i

QA Lead seviyesinde doğru test türünü seçmek önemli.

| Tip | Amaç | Konfigürasyon | Süre |
|---|---|---|---|
| **Smoke** | Critical flow ayakta mı? | 3-5 senaryo, tek device | 5-10 dk |
| **Regression** | Tam coverage | Tüm senaryo, tek device | 30-60 dk |
| **Cross-device** | Farklı device'larda | Matrix (Pixel 6 + S22 + iPhone 15) | 1-2 saat |
| **Cross-OS** | Farklı OS version | Android 11/12/13/14 | 1-2 saat |
| **Performance** | App startup, render hızı | Perf metrics topla | 30 dk |
| **Visual** | UI regression | Percy/Applitools | Smoke'a paralel |
| **Accessibility** | a11y | axe-android, axe-ios | Smoke'a paralel |
| **Network** | Offline, 3g, slow | Network simulation | 30 dk |
| **Localization** | Multi-language | tr, en, de, fr matrix | 30 dk × dil |
| **Monkey** | Stability | Espresso Monkey, AppCrawler | 1-2 saat |
| **Soak** | Memory leak, long-run | 4+ saat tekrarlı flow | 4-8 saat |

### Test piramidi — mobile özelinde
```
            E2E (Appium)             %10
          ─────────────────
       Integration / Mock Backend    %20
          ─────────────────
        Unit (Robolectric / iOS)     %70
       ─────────────────────────
```

Appium **piramidin tepesi** — pahalı, kırılgan. Critical user journey'leri için.

### Senaryo seçimi
**E2E'ye girer:**
- Login / signup / logout flow'u
- Checkout / payment (real money interaction)
- Push notification handling
- Deep link navigation
- Cross-screen state (kart oluştur → liste'de görün)
- Network failure recovery
- Permission flow (location, camera)

**E2E'ye girmez (unit/integration):**
- Form validation (sadece UI label test'i)
- Button enabled/disabled state (state machine test'i)
- API request/response (mock backend yeterli)
- Calculator logic (unit test)

---

## 29. Test Data Management

### YAML fixture
```yaml
# spec/data/users.yml
admin:
  email: admin@example.com
  password: admin123
  role: admin

basic:
  email: user@example.com
  password: user123
  role: user

deep_link_tokens:
  admin: "test_token_admin_abc123"
  user:  "test_token_user_xyz789"
```

```ruby
class TestData
  USERS = YAML.load_file("spec/data/users.yml").freeze

  def self.user(type)
    USERS.fetch(type.to_s)
  end
end

# Test
user = TestData.user(:admin)
LoginPage.new(driver).login(email: user["email"], password: user["password"])
```

### Test user pool — isolated user
```ruby
def create_unique_user(role: "user")
  email = "test-#{SecureRandom.hex(4)}@example.com"
  ApiClient.create_user(email: email, password: "test123", role: role)
end

it "registers a new user", :isolated do
  user = create_unique_user
  # ...
end
```

Her test kendi user'ı, data leak yok.

### Backend mock
**WireMock veya MockServer** ayrı bir process olarak çalışır, mobil app onu hit eder.

```bash
# Docker
docker run -p 8080:8080 wiremock/wiremock
```

Mobile app'in API URL'ini test ortamında `http://localhost:8080` (Android emulator için `10.0.2.2:8080`) ayarla.

```ruby
# Wiremock stub
require "net/http"

def stub_api(endpoint:, response_body:, status: 200)
  Net::HTTP.post(
    URI("#{ENV['WIREMOCK_URL']}/__admin/mappings"),
    {
      request: { method: "GET", url: endpoint },
      response: { status: status, body: response_body.to_json }
    }.to_json,
    "Content-Type" => "application/json"
  )
end

it "shows products from API" do
  stub_api(endpoint: "/api/products", response_body: [
    { id: 1, name: "iPhone", price: 1000 }
  ])

  driver.find_element(:accessibility_id, "products_tab").click
  wait_visible(:accessibility_id, "product_iPhone")
end
```

### Deep link state setup
```ruby
def bypass_login(role: :user)
  token = TestData.user(role).fetch("deep_link_token", "default_token")
  driver.execute_script("mobile: deepLink", {
    url:     "myapp://auth?token=#{token}",
    package: "com.example.app"
  })
  wait_visible(:accessibility_id, "dashboard_title")
end

it "tests dashboard quickly" do
  bypass_login(role: :admin)
  # Doğrudan dashboard'da
end
```

5-10 saniyelik UI login'i bypass ediyor → test suite 10 dk hızlanır.

---

## 30. Cucumber Entegrasyonu

Detay için bkz: `cucumber.md` (bu repo'da). Kısa örnek:

### Setup
```ruby
# Gemfile
gem "cucumber"
gem "appium_lib_core"
```

```ruby
# features/support/env.rb
require "appium_lib_core"
require_relative "appium_session"

PLATFORM = ENV.fetch("PLATFORM", "android").to_sym

module AppiumWorld
  def driver
    AppiumSession.driver
  end

  def wait
    @wait ||= Selenium::WebDriver::Wait.new(timeout: 10)
  end
end

World(AppiumWorld)

After do |scenario|
  if scenario.failed?
    png = driver.screenshot_as(:png)
    attach(png, "image/png")
  end
end

AfterAll do
  AppiumSession.quit
end
```

### Feature
```gherkin
@mobile @auth
Feature: Mobil giriş

  @smoke
  Scenario: Başarılı giriş
    Given mobil uygulama açık
    When kullanıcı "user@example.com" ile mobilde giriş yapar
    Then mobilde dashboard ekranı görünür
```

### Step definitions
```ruby
Given("mobil uygulama açık") do
  driver.activate_app("com.example.app")
end

When("kullanıcı {string} ile mobilde giriş yapar") do |email|
  @login_page = LoginPage.new(driver)
  @dashboard = @login_page
                 .enter_email(email)
                 .enter_password("default-password")
                 .submit
end

Then("mobilde dashboard ekranı görünür") do
  expect(@dashboard).to be_loaded
end
```

---

## 31. Visual Testing

### Applitools Eyes
```ruby
gem "eyes_appium"
```

```ruby
require "applitools/appium"

eyes = Applitools::Appium::Eyes.new
eyes.api_key = ENV["APPLITOOLS_API_KEY"]
eyes.open(driver: driver, app_name: "MyApp", test_name: "Login screen")

# Login screen'i check et
eyes.check_window("Login")

# Form filled state
fill_in_email("x@y.com")
eyes.check_window("Login with email")

eyes.close
```

### Percy
```ruby
gem "percy-appium-app"

require "percy/appium_app"

it "visual regression" do
  PercyAppium.percy_screenshot(driver, "Login screen")

  enter_email("x@y.com")
  PercyAppium.percy_screenshot(driver, "Login - filled")
end
```

### Cloud provider built-in
LambdaTest, BrowserStack visual testing **built-in**:
```ruby
"lt:options" => {
  visual: true
}
```

Test sonu visual diff dashboard'da.

---

## 32. Sık Karşılaşılan Pitfall'lar

### 1. XPath aşırı kullanımı
```ruby
# ❌ Kırılgan + yavaş
driver.find_element(:xpath, "/hierarchy/android.widget.FrameLayout[1]/.../Button")

# ✅ Accessibility ID
driver.find_element(:accessibility_id, "login_button")
```

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
btn.click   # StaleElementReferenceError

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

### 9. Platform-agnostic POM zorlamak
İki platformda aynı UI bile farklı render edilir. `locator` class macro DSL'i ile Android/iOS locator'larını **ayrı tut**.

### 10. App version drift
Test build'leri kontrol altında olmalı — staging app'i değişirse test fail eder. **Build versioning** zorunlu.

### 11. Chromedriver mismatch (hybrid app)
WebView Chrome version'u ile yüklü Chromedriver uyumsuz → context switch fail.
**Çözüm:** `appium:chromedriverAutodownload: true`

### 12. iOS bundle ID typo
Bir karakter yanlışsa app start fail. Capability'leri **doğrula**.

### 13. Permission dialog crash
Runtime permission gelirken test buna hazır değilse fail.
**Çözüm:** `autoGrantPermissions: true` (Android), `autoAcceptAlerts: true` (iOS), veya manuel dialog handle.

### 14. Cloud session timeout
Cloud provider'lar default 90sn idle timeout. Yavaş test'lerde session kapanır.
**Çözüm:** `"lt:options" => { idleTimeout: 300 }` veya benzeri.

### 15. App not yet ready after start
App start sonrası splash screen / animation devam ediyor.
**Çözüm:** İlk action öncesi **explicit wait** ile splash'in kaybolmasını bekle.

### 16. ADB connection lost
Uzun test suite'lerde ADB connection kopabilir.
**Çözüm:** `adb kill-server && adb start-server` between major suite blocks.

### 17. UI Animation timing
CSS-equivalent animation'lar test'i flaky yapar.
**Çözüm:** `"appium:disableWindowAnimation": true` (Android), Settings'te developer options "Animation scale off" (iOS).

### 18. Locale-specific text
Türkçe label'a `find_element(:accessibility_id, "Save")` İngilizce'de çalışır, Türkçe'de fail.
**Çözüm:** Accessibility ID **i18n'den bağımsız** olmalı (developer'larla anlaş).

### 19. Cross-platform unit test gibi yazmak
Aynı assertion'lar her platforma yarı uyumsuz çalışabilir.
**Çözüm:** Platform-specific test branch'i yaz veya feature toggle.

### 20. Hidden element click'i
Native'de "scrollable list" içindeki element ekranda yok ama element tree'de var.
**Çözüm:** `mobile: scroll` veya `UiScrollable.scrollIntoView` ile görünür yap.

---

## 33. Performans Optimizasyonu

### Locator
- **accessibility_id** > id > uiautomator/predicate > xpath
- XPath'ten kaçın — 10x yavaş

### Session reuse
```ruby
RSpec.configure do |c|
  c.before(:all, :appium) { AppiumSession.driver }
  c.after(:suite)         { AppiumSession.quit }
end
```

Her test için yeni session açmak yerine suite başında bir kere.

### Paralel test
- `parallel_tests` gem ile process-based paralel
- Cloud farm parallelism (5-50 device eş zamanlı)
- Her process için unique Appium port + systemPort + wdaLocalPort

### `noReset: true` (state leak'e dikkat)
Setup süresini düşürür ama izolasyon kaybı.

### Bypass UI via deep link
Login UI yerine deep link ile state setup.

### Backend mocking
Backend API yavaş ise WireMock/MockServer ile in-memory replace.

### Disable animation
```ruby
"appium:disableWindowAnimation": true
```

### App build optimization
- Debug build minify edilmemiş — slow render
- Release build test edilmeli mümkünse

### Cloud parallel session limit
LambdaTest 5 paralel, BrowserStack 50+ paralel. Plan'a göre değişir. CI'da parallel sayısını buna göre ayarla.

---

## 34. Framework Tasarımı

### Klasör yapısı (kapsamlı)
```
qa-mobile/
├── Gemfile
├── Rakefile
├── .rspec
├── config/
│   ├── android.yml
│   ├── ios.yml
│   ├── lambdatest.yml
│   ├── browserstack.yml
│   └── selectors/
│       ├── login_page.yml
│       └── checkout_page.yml
├── apps/
│   ├── android/
│   │   ├── app-debug.apk
│   │   └── app-release.apk
│   └── ios/
│       └── MyApp.app
├── spec/
│   ├── spec_helper.rb
│   ├── support/
│   │   ├── appium_session.rb
│   │   ├── caps_builder.rb
│   │   ├── platform.rb
│   │   ├── pages/
│   │   │   ├── base_page.rb
│   │   │   ├── shared/
│   │   │   │   ├── login_page.rb
│   │   │   │   └── dashboard_page.rb
│   │   │   ├── android/
│   │   │   └── ios/
│   │   ├── components/
│   │   │   ├── header.rb
│   │   │   └── tab_bar.rb
│   │   ├── helpers/
│   │   │   ├── wait_helpers.rb
│   │   │   ├── gesture_helpers.rb
│   │   │   ├── deep_link_helpers.rb
│   │   │   ├── permission_helpers.rb
│   │   │   └── screenshot_helpers.rb
│   │   ├── hooks/
│   │   │   ├── failure_screenshot.rb
│   │   │   ├── session_management.rb
│   │   │   └── allure.rb
│   │   ├── shared_contexts/
│   │   │   ├── authenticated_user.rb
│   │   │   └── new_install.rb
│   │   └── matchers/
│   │       └── displayed_matcher.rb
│   ├── system/
│   │   ├── authentication/
│   │   │   ├── login_spec.rb
│   │   │   ├── signup_spec.rb
│   │   │   └── password_reset_spec.rb
│   │   ├── checkout/
│   │   │   ├── cart_spec.rb
│   │   │   ├── payment_spec.rb
│   │   │   └── confirmation_spec.rb
│   │   ├── profile/
│   │   ├── notifications/
│   │   └── deep_links/
│   └── data/
│       ├── users.yml
│       └── products.yml
└── tmp/
    ├── screenshots/
    └── allure-results/
```

### Driver Factory pattern (thread-safe)
```ruby
class AppiumSession
  def self.driver
    Thread.current[:appium_driver] ||= start_driver
  end

  def self.quit
    Thread.current[:appium_driver]&.quit
    Thread.current[:appium_driver] = nil
  end

  def self.restart
    quit
    driver
  end

  def self.start_driver
    caps = CapsBuilder.build(platform: PLATFORM)
    Appium::Core.for(caps).start_driver
  end
end
```

### Caps builder
```ruby
class CapsBuilder
  def self.build(platform:)
    base_config = YAML.load_file("config/#{platform}.yml")
    caps = base_config["caps"].deep_dup

    apply_overrides!(caps, platform)
    apply_paralel_overrides!(caps, platform)

    {
      caps: caps.transform_keys(&:to_sym),
      appium_lib: { server_url: server_url(platform) }
    }
  end

  private

  def self.apply_overrides!(caps, platform)
    caps["appium:noReset"] = ENV["NO_RESET"] == "true" if ENV["NO_RESET"]
    caps["appium:fullReset"] = ENV["FULL_RESET"] == "true" if ENV["FULL_RESET"]
  end

  def self.apply_paralel_overrides!(caps, platform)
    worker = ENV["TEST_ENV_NUMBER"].to_i
    return if worker.zero?

    if platform == :android
      caps["appium:systemPort"] = 8200 + worker
    else
      caps["appium:wdaLocalPort"] = 8100 + worker
    end
  end

  def self.server_url(platform)
    if ENV["USE_CLOUD"] == "true"
      cloud_config = YAML.load_file("config/lambdatest.yml")
      cloud_config["server_url"]
    else
      "http://127.0.0.1:#{4723 + ENV['TEST_ENV_NUMBER'].to_i}"
    end
  end
end
```

### Tag'leme
```ruby
RSpec.describe "Critical path", :smoke, :appium, :android_only do
  # ...
end

RSpec.describe "Long flow", :slow, :regression do
  # ...
end
```

```bash
bundle exec rspec --tag smoke                      # PR'da
bundle exec rspec --tag "regression and ~slow"     # nightly
bundle exec rspec --tag android_only               # platform-specific
```

### Failure hooks
```ruby
RSpec.configure do |c|
  c.after(:each, :appium) do |example|
    if example.exception
      timestamp = Time.now.strftime("%Y%m%d_%H%M%S")
      safe_name = example.full_description.parameterize
      base_path = "tmp/failures/#{timestamp}_#{safe_name}"

      driver = AppiumSession.driver

      # Screenshot
      driver.save_screenshot("#{base_path}.png")

      # Page source
      File.write("#{base_path}.xml", driver.page_source)

      # Allure attachment
      Allure.add_attachment(
        name: "Screenshot",
        source: File.read("#{base_path}.png"),
        type: Allure::ContentType::PNG
      )

      # Cloud session URL (LambdaTest, BrowserStack)
      if ENV["USE_CLOUD"]
        puts "Cloud session: https://automation.lambdatest.com/test?testID=#{driver.session_id}"
      end
    end
  end
end
```

---

## 35. QA Lead Seviyesi — Mimari Kararlar

### Test stratejisi piramidi
1. **Unit** (developer) — %70 (XCTest, Robolectric)
2. **API integration** — %15
3. **UI integration (Appium)** — %10
   - Critical user journey'leri
   - Cross-platform regression
   - Cross-device coverage
4. **Monkey/exploratory** — %5 (manuel + AppCrawler)

### Device coverage matrix
| Tier | Android | iOS |
|---|---|---|
| **P0 (her PR)** | Pixel 6 (API 33) | iPhone 15 (iOS 17) |
| **P1 (nightly)** | Samsung S22, Pixel 4a, Tablet | iPhone 13, iPad Air |
| **P2 (release)** | Düşük-end (Samsung A series, Xiaomi), older API | iPhone SE, eski iPad |

### OS version coverage
- **Min supported version** + 2 son version → P0
- Aradaki version'lar → P1/P2

### Driver seçim kararı
| Senaryo | Driver |
|---|---|
| Source erişimi var, hız önemli | Espresso (Android) |
| Production APK test | UiAutomator2 (Android), XCUITest (iOS) |
| WebView heavy app | UiAutomator2 + chromedriver |
| Mobile web | Selenium-style + browserName cap |

### Cloud provider kararı
| Provider | Lead seçimi |
|---|---|
| Enterprise (3000+ device, mature) | BrowserStack |
| DevOps native, AWS ecosystem | AWS Device Farm + Sauce |
| Cost-effective startup | LambdaTest |
| Google ecosystem | Firebase Test Lab |
| Performance/network focus | HeadSpin |

### Test data stratejisi
- **Test user pool** — her test izole user
- **Deep link** ile state setup (5-10 sn tasarruf)
- **Backend mock** (WireMock) — external dependency'i ortadan kaldır
- **Fixture'lar** — YAML test data

### Maintenance SLA
- **Flaky test rate < %1** — üstündekiler quarantine
- **Test execution time** — >5dk olan senaryolar review
- **Locator audit** — XPath kullananları flag'le
- **App build versioning** — test koşulan build commit SHA tag'lensin

### Team scaling
- **CODEOWNERS** — feature team'lere POM ownership
- **Code review checklist:**
  - [ ] Locator strategy (accessibility_id > id > XPath)
  - [ ] Explicit wait kullanımı
  - [ ] Cross-platform POM uyumu
  - [ ] Paralel-safe mi (Thread.current)
  - [ ] Failure hook'ı kurulmuş
  - [ ] App version doğru
- **Onboarding doc** — junior 1 hafta içinde ilk PR
  - Local emulator setup
  - Appium server start
  - İlk test'i koşma
  - POM convention
- **Pair testing** — senior + junior çift mülakat

### Risk yönetimi
- **iOS gerçek device** — Apple Developer Program ($99/yıl), certificate management
- **Android fragmentation** — düşük-end device behavior farkı
- **App version uyumsuzluğu** — backend ↔ app version mismatch
- **Appium upgrade** — major version (2.x) breaking change'leri
- **Cloud provider downtime** — yedek strateji (multi-provider veya lokal fallback)

### Living documentation
- Critical journey'leri Cucumber feature olarak yaz → Product/Business okur
- Test isimleri user-facing senaryolar

---

## 36. En Sık Mülakat Soruları

| Soru | Hızlı Cevap |
|---|---|
| Appium nedir? | Cross-platform mobile UI test framework; W3C WebDriver protokolü tabanlı |
| Appium 1 vs 2? | 2.x driver/plugin tabanlı modüler mimari; capability'ler `appium:` prefix |
| Appium mimari? | Proxy server (Node.js) → driver (UiAutomator2/XCUITest) → bootstrap app → native framework → device |
| Driver nedir? | Platform-specific komut handler; her platform için ayrı (UiAutomator2, XCUITest, Espresso) |
| Bootstrap app nedir? | Driver'ın device'a kurduğu helper app (Android: uiautomator2-server, iOS: WDA) |
| Ruby client'ı? | `appium_lib_core` (modern, önerilen), `appium_lib` (legacy full-fat) |
| Capability nedir? | Session başlatırken Appium'a "ne istediğini söyleyen" JSON config |
| `appium:` prefix neden? | W3C: vendor-specific cap'leri prefix'le ayır; standart cap'ler prefix'siz |
| Native vs Hybrid vs Web? | Native: platform UI; Hybrid: native + WebView; Mobile web: browser'da responsive |
| Hybrid app'i nasıl test edersin? | `available_contexts` → `set_context("WEBVIEW_...")` |
| Context switching? | Native ↔ WebView geçişi; `set_context`, `available_contexts` |
| En iyi locator? | accessibility_id; cross-platform, hızlı, güvenilir |
| Accessibility ID neden? | a11y için zaten gerekli; QA için extra değil — developer set eder |
| XPath neden kötü? | Yavaş (10x), hierarchy'ye duyarlı, küçük UI değişiminde kırılır |
| Android-specific locator? | `:uiautomator` (UiSelector); XPath'ten 5-10x hızlı |
| iOS-specific locator? | `:predicate` (predicate string), `:class_chain` (XPath benzeri, hızlı) |
| Wait stratejisi? | Sadece explicit (`Selenium::WebDriver::Wait`); implicit + explicit karıştırma; `sleep` yasak |
| `noReset` vs `fullReset`? | noReset hızlı, state leak; fullReset yavaş, izole |
| `noReset: true` ne zaman? | Smoke test'te hız öncelikli, state leak tolere edilebiliyorsa |
| Paralel test? | `parallel_tests` gem + `Thread.current[:driver]` + unique systemPort/wdaLocalPort |
| Cloud farm tercihi? | BrowserStack (mature), LambdaTest (uygun fiyat), Sauce (enterprise) |
| iOS gerçek device challenge? | WDA imzalama + Apple Developer Cert; çoğu zaman cloud çözüm |
| Touch gesture API? | W3C Actions veya `mobile:` gesture extensions |
| `mobile:` ne işe yarar? | Driver-specific kısa yollar (tap, swipe, scroll, longClick) |
| Deep link açma? | `mobile: deepLink` veya ADB shell |
| Deep link use case? | Login UI'ı bypass et, doğrudan dashboard'a → test 5-10sn hızlanır |
| Permission management? | Android: `autoGrantPermissions`, `mobile: changePermissions`; iOS: `autoAcceptAlerts` |
| Network simulation? | Cloud provider built-in (3g/4g/offline); lokal `mobile: shell svc wifi disable` |
| Geolocation? | `driver.set_location(lat:, lon:)` veya capability'de |
| Cross-platform POM? | `locator` class macro DSL (önerilen); subclass per platform; YAML config |
| Espresso vs UiAutomator2? | Espresso in-process+hızlı+source gerekli; UiAutomator2 out-of-process+production APK |
| Reporting? | `allure-rspec` (en yaygın); rspec_junit_formatter; cloud native dashboard'lar |
| Failure screenshot? | `after(:each)` hook'ta `driver.save_screenshot` + Allure attach |
| Page source? | `driver.page_source` — XML UI hierarchy |
| Stale element nasıl önlenir? | Element ref cache etme; locator constant kullan |
| Flaky test çözümü? | Explicit wait; locator audit; data isolation; retry (son çare) |
| Inspector ne işe yarar? | UI hierarchy göster, element seç, locator copy |
| ADB nedir? | Android Debug Bridge — device ile iletişim CLI tool; UiAutomator2 bunu kullanır |
| WDA nedir? | WebDriverAgent — Facebook iOS bootstrap app; XCUITest driver kullanır |
| `appium:systemPort`? | UiAutomator2 server port; paralel test için unique olmalı |
| `appium:wdaLocalPort`? | WDA port; iOS paralel için unique |
| App lifecycle method'ları? | activate_app, terminate_app, background_app, install_app, remove_app |
| Performance metrics? | `mobile: getPerformanceData` (Android); iOS sınırlı |
| Mobile gesture extension örneği? | `mobile: swipeGesture`, `mobile: tap`, `mobile: longClickGesture` |
| Cloud session URL? | Test sonrası `driver.session_id` ile dashboard URL build |
| Local tunneling? | BrowserStack Local, Sauce Connect, LambdaTest Tunnel — staging app'i cloud'dan test |
| Test piramidi pozisyonu? | Mobile %10 (E2E); kritik flow'lar için |
| Device coverage matrix? | P0 (her PR), P1 (nightly), P2 (release) — tiered |
| Lead'in en önemli kararı? | Hangi case Appium'da, hangisi unit/API'de — test piramidi |

---

## 37. Çalışma Yol Haritası

### 1. Hafta — Temeller
- [ ] Appium 2.x kurulumu (UiAutomator2 + XCUITest driver)
- [ ] `appium_lib_core` gem'i ile ilk session
- [ ] Appium Inspector ile basit bir app'i incele
- [ ] İlk Android test'ini yaz (login flow) — RSpec ile
- [ ] Locator stratejilerini dene (accessibility_id, id, uiautomator, xpath)
- [ ] Capability'leri öğren (noReset, fullReset, autoGrantPermissions)

### 2. Hafta — Orta seviye
- [ ] `Selenium::WebDriver::Wait` ile explicit wait pattern'leri
- [ ] FluentWait pattern + ignore
- [ ] Touch gestures (W3C actions)
- [ ] **Mobile gesture extensions** (`mobile: swipe/scroll/tap`)
- [ ] Hybrid app context switching
- [ ] App lifecycle (activate, terminate, background)
- [ ] Permission management (auto-grant + manual)

### 3. Hafta — Framework
- [ ] **Page Object Model** — BasePage + 3 sayfa
- [ ] Component pattern
- [ ] Lazy locator pattern
- [ ] **Cross-platform POM** — `locator` class macro DSL
- [ ] DriverFactory + `Thread.current[:driver]` pattern
- [ ] `parallel_tests` ile paralel RSpec
- [ ] CapsBuilder + YAML config
- [ ] Test data fixture'ları (YAML)
- [ ] Failure screenshot hook
- [ ] `allure-rspec` reporting

### 4. Hafta — CI/CD & Lead
- [ ] GitHub Actions / Jenkins pipeline (lokal emulator)
- [ ] **LambdaTest** veya **BrowserStack** entegrasyonu
- [ ] Local tunneling (BrowserStack Local)
- [ ] Paralel cloud test (5+ device eş zamanlı)
- [ ] Deep link bypass pattern
- [ ] Backend mocking (WireMock)
- [ ] **Device coverage matrix** dökümante et
- [ ] Flake rate dashboard
- [ ] Code review checklist + onboarding doc

### Bonus — derin konular
- [ ] **Espresso** driver setup + karşılaştırma
- [ ] **Visual regression** (Percy/Applitools)
- [ ] **Accessibility** test (axe-android, axe-ios)
- [ ] **Performance metrics** (memory, CPU, network)
- [ ] **Push notification** testi
- [ ] **Geolocation** simülasyonu
- [ ] **Mobile web** (Chrome/Safari)
- [ ] Plugin sistem (Appium 2.x): images, relaxed-caps, execute-driver

---

## 38. Kaynaklar

### Resmi
- **Ana sayfa:** appium.io
- **GitHub:** github.com/appium/appium
- **Docs:** appium.io/docs/en/latest
- **Inspector:** github.com/appium/appium-inspector
- **Driver listesi:** appium.io/docs/en/latest/ecosystem/drivers/
- **Plugin listesi:** appium.io/docs/en/latest/ecosystem/plugins/

### Ruby client'ları
- **`appium_lib_core` (modern):** github.com/appium/ruby_lib_core
- **`appium_lib` (legacy):** github.com/appium/ruby_lib
- **Sample Ruby kod:** github.com/appium/ruby_lib_core/tree/master/example

### Driver'lar
- **UiAutomator2:** github.com/appium/appium-uiautomator2-driver
- **XCUITest:** github.com/appium/appium-xcuitest-driver
- **Espresso:** github.com/appium/appium-espresso-driver
- **WebDriverAgent:** github.com/appium/WebDriverAgent

### Reporting
- **`allure-cucumber` / `allure-rspec`:** github.com/allure-framework/allure-ruby
- **`rspec_junit_formatter`:** github.com/sj26/rspec_junit_formatter

### Visual & Accessibility
- **Applitools Appium:** github.com/applitools/eyes-appium-ruby
- **Percy Appium:** github.com/percy/percy-appium-app-ruby

### Cloud test providers
- **BrowserStack docs:** browserstack.com/docs/app-automate/appium
- **LambdaTest docs:** lambdatest.com/support/docs/appium-testing
- **Sauce Labs docs:** docs.saucelabs.com/mobile-apps

### Parallel
- **`parallel_tests`:** github.com/grosser/parallel_tests

### Mobile gesture & UiAutomator
- **UiAutomator selector docs:** developer.android.com/reference/androidx/test/uiautomator
- **iOS Predicate strings:** developer.apple.com/documentation/foundation/nspredicate
- **Mobile gesture extensions:** appium.io/docs/en/latest/guides/execute-methods/

### Kitaplar / kurslar
- **"Hands-On Mobile App Testing"** — Daniel Knott
- **"Appium Recipes"** — Shankar Garg
- **Test Automation University (free):** testautomationu.applitools.com/appium-ruby-tutorial/

### Topluluk
- **Appium Forum:** discuss.appium.io
- **Stack Overflow tag:** `appium`
- **Reddit:** r/QualityAssurance, r/softwaretesting
