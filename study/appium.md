# Appium — QA Lead Mülakat Hazırlık Rehberi

> Mobile (iOS / Android) ve cross-platform UI test otomasyonu için endüstri standardı. Bu doküman QA Lead seviyesinde mülakata hazırlık içindir.

---

## 1. Appium Nedir?

**Appium**, mobile uygulamalar için **cross-platform UI test otomasyonu** sağlayan açık kaynak bir framework'tür. Tek bir API ile **iOS, Android, Windows ve macOS** uygulamalarını test edebilirsin.

**Temel özellikleri:**
- **WebDriver protocol** üzerine kurulu (Selenium ile aynı protokol — W3C WebDriver standardı)
- **Native, hybrid ve web mobile** uygulamaları test eder
- Source code değişikliği gerektirmez (no instrumentation in production)
- **Dil-agnostic**: Java, Python, Ruby, JavaScript, C#, Robot Framework client'ları
- **Driver mimarisi:** Her platform için ayrı driver (UiAutomator2, XCUITest, Espresso, vb.)

**Versiyonlar:**
- **Appium 1.x:** Eski mimari, monolithic, deprecated.
- **Appium 2.x (current):** Driver/plugin tabanlı mimari, modüler. **Yeni projeler 2.x kullanmalı.**

---

## 2. Mimari

```
┌─────────────────────────────────────────────────────────────┐
│   Test Code (Java/Python/Ruby/JS)                           │
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

## 3. Kurulum (Appium 2.x)

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

Her test'te server'a hangi platform/cihaz/uygulama isteğinin yapıldığını söyleyen JSON config.

### Android örneği (Java)
```java
UiAutomator2Options options = new UiAutomator2Options()
    .setPlatformName("Android")
    .setPlatformVersion("13")
    .setDeviceName("Pixel_6_API_33")
    .setApp("/path/to/app.apk")
    .setAutomationName("UiAutomator2")
    .setAppPackage("com.example.app")
    .setAppActivity(".MainActivity")
    .setNoReset(false)
    .setFullReset(false);

AndroidDriver driver = new AndroidDriver(
    new URL("http://127.0.0.1:4723"), options
);
```

### iOS örneği
```java
XCUITestOptions options = new XCUITestOptions()
    .setPlatformName("iOS")
    .setPlatformVersion("17.0")
    .setDeviceName("iPhone 15")
    .setApp("/path/to/MyApp.app")
    .setAutomationName("XCUITest")
    .setBundleId("com.example.MyApp")
    .setUdid("00008110-XXXX")     // gerçek device için
    .setXcodeOrgId("ABC1234567")
    .setXcodeSigningId("iPhone Developer");

IOSDriver driver = new IOSDriver(new URL("http://127.0.0.1:4723"), options);
```

### Önemli capability'ler
| Capability | Açıklama |
|---|---|
| `platformName` | "Android" / "iOS" |
| `platformVersion` | OS sürümü |
| `deviceName` | Cihaz adı (emülatör/simülatör adı veya gerçek device) |
| `automationName` | "UiAutomator2" / "XCUITest" / "Espresso" |
| `app` | APK / .app / .ipa dosyasının absolute path'i |
| `appPackage` + `appActivity` | Android: yüklü uygulamayı başlatmak için |
| `bundleId` | iOS: yüklü uygulamayı başlatmak için |
| `udid` | Gerçek cihazın unique device ID'si |
| `noReset` | true → session sonunda app data silinmez |
| `fullReset` | true → app uninstall + reinstall (yavaş, izolasyon yüksek) |
| `newCommandTimeout` | İdle session timeout (default 60sn) |
| `autoGrantPermissions` | Android: runtime permission'ları auto-grant |
| `autoAcceptAlerts` | iOS: alert'leri auto-accept |
| `language` / `locale` | i18n test için |

---

## 5. Locator Stratejileri

WebDriver'ın `By` API'sı + Appium'un mobile özel locator'ları.

| Locator | Hız | Güvenilirlik | Kullanım |
|---|---|---|---|
| **accessibility id** | ⚡⚡⚡ | ✅ En iyi | iOS: `accessibilityIdentifier`, Android: `content-desc` |
| **id** | ⚡⚡⚡ | ✅ | Android: `resource-id`, iOS: name attribute |
| **class name** | ⚡⚡ | ⚠️ | UI widget class (`android.widget.Button`) |
| **xpath** | 🐢 | ⚠️ | En esnek, **en yavaş ve kırılgan** |
| **-android uiautomator** | ⚡⚡ | ✅ | Android-specific powerful selector |
| **-ios predicate string** | ⚡⚡ | ✅ | iOS-specific powerful selector |
| **-ios class chain** | ⚡⚡ | ✅ | iOS XPath benzeri ama hızlı |

### Örnekler
```java
// Accessibility ID — önerilen
driver.findElement(AppiumBy.accessibilityId("login_button"));

// ID
driver.findElement(AppiumBy.id("com.example:id/email"));

// Android UiAutomator
driver.findElement(AppiumBy.androidUIAutomator(
    "new UiSelector().textContains(\"Welcome\")"
));

driver.findElement(AppiumBy.androidUIAutomator(
    "new UiScrollable(new UiSelector().scrollable(true))" +
    ".scrollIntoView(new UiSelector().text(\"Settings\"))"
));

// iOS Predicate
driver.findElement(AppiumBy.iOSNsPredicateString(
    "type == 'XCUIElementTypeButton' AND name CONTAINS 'Login'"
));

// iOS Class Chain
driver.findElement(AppiumBy.iOSClassChain(
    "**/XCUIElementTypeButton[`name == \"Login\"`]"
));

// XPath — son çare
driver.findElement(AppiumBy.xpath("//android.widget.Button[@text='Login']"));
```

**Önerilen sıra:** accessibility id → id → uiautomator/predicate → xpath (son çare).

**XPath neden kırılgan?**
- DOM-benzeri tree'yi tarar → **çok yavaş** (büyük uygulamalarda 5-10 sn)
- UI hierarchy'sindeki küçük değişiklikler test'i kırar
- Index-tabanlı XPath (`//Button[3]`) en kırılgan

---

## 6. Sık Kullanılan Komutlar

### Element interaction
```java
WebElement el = driver.findElement(AppiumBy.accessibilityId("login_btn"));
el.click();
el.sendKeys("text input");
el.clear();
el.getText();
el.isDisplayed();
el.isEnabled();
el.getAttribute("content-desc");
```

### Touch gestures (Appium 2.x — W3C Actions)
```java
PointerInput finger = new PointerInput(PointerInput.Kind.TOUCH, "finger");
Sequence tap = new Sequence(finger, 1)
    .addAction(finger.createPointerMove(Duration.ZERO, Origin.viewport(), 100, 200))
    .addAction(finger.createPointerDown(PointerInput.MouseButton.LEFT.asArg()))
    .addAction(finger.createPointerUp(PointerInput.MouseButton.LEFT.asArg()));
driver.perform(Arrays.asList(tap));
```

**Daha pratik — `executeScript` ile mobile gesture extensions:**
```java
// Tap
driver.executeScript("mobile: tap", Map.of("x", 100, "y", 200));

// Swipe (Android UiAutomator2)
driver.executeScript("mobile: swipeGesture", Map.of(
    "left", 100, "top", 500, "width", 200, "height", 800,
    "direction", "up", "percent", 0.75
));

// Long press
driver.executeScript("mobile: longClickGesture", Map.of(
    "elementId", ((RemoteWebElement) el).getId(),
    "duration", 2000
));

// Scroll
driver.executeScript("mobile: scrollGesture", Map.of(
    "left", 100, "top", 100, "width", 800, "height", 1200,
    "direction", "down", "percent", 1.0
));

// iOS swipe
driver.executeScript("mobile: swipe", Map.of(
    "direction", "left", "element", ((RemoteWebElement) el).getId()
));
```

### App management
```java
driver.terminateApp("com.example.app");           // app'i kapat
driver.activateApp("com.example.app");            // app'i öne getir
driver.installApp("/path/to/app.apk");
driver.removeApp("com.example.app");
driver.isAppInstalled("com.example.app");

driver.backgroundApp(Duration.ofSeconds(5));      // app'i 5 sn background'a al
driver.runAppInBackground(Duration.ofSeconds(5));
```

### Device interaction
```java
driver.pressKey(new KeyEvent(AndroidKey.BACK));   // Android
driver.pressKey(new KeyEvent(AndroidKey.HOME));

driver.rotate(ScreenOrientation.LANDSCAPE);
driver.rotate(ScreenOrientation.PORTRAIT);

driver.getScreenshotAs(OutputType.FILE);
driver.getPageSource();                           // tüm UI hierarchy XML
```

### Context switching (hybrid app)
```java
Set<String> contexts = driver.getContextHandles();
// ["NATIVE_APP", "WEBVIEW_com.example"]

driver.context("WEBVIEW_com.example");            // WebView'a geç
driver.findElement(By.cssSelector("button#login")).click();

driver.context("NATIVE_APP");                     // Native'e geri dön
```

---

## 7. Wait Stratejileri

Selenium'unki ile aynı — Appium **kendi auto-wait sağlamaz** (Capybara'dan farklı olarak), explicit wait kullanmalısın.

### Implicit Wait (önerilmez — kötü pratik)
```java
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
```
Implicit + explicit karışırsa beklenmedik davranış çıkar. **Sadece explicit kullan.**

### Explicit Wait
```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
WebElement el = wait.until(
    ExpectedConditions.elementToBeClickable(AppiumBy.accessibilityId("login"))
);

wait.until(ExpectedConditions.presenceOfElementLocated(...));
wait.until(ExpectedConditions.visibilityOfElementLocated(...));
wait.until(ExpectedConditions.textToBePresentInElement(el, "Welcome"));
wait.until(ExpectedConditions.invisibilityOfElementLocated(...));
```

### FluentWait (custom polling, exception ignore)
```java
Wait<WebDriver> fluentWait = new FluentWait<>(driver)
    .withTimeout(Duration.ofSeconds(30))
    .pollingEvery(Duration.ofMillis(500))
    .ignoring(NoSuchElementException.class);

WebElement el = fluentWait.until(d ->
    d.findElement(AppiumBy.accessibilityId("dynamic_btn"))
);
```

### Kural
- **Implicit wait kullanma**
- Her tıklama/etkileşim öncesi **explicit wait** ile elementin clickable/visible olduğunu garanti et
- `Thread.sleep` **anti-pattern**'dir — sadece debug için, asla CI test'inde

---

## 8. Native vs Hybrid vs Web

| Tip | Açıklama | Test stratejisi |
|---|---|---|
| **Native** | Platform UI framework'ü ile yazılmış (Swift/Java/Kotlin) | UiAutomator2 / XCUITest driver, native locator'lar |
| **Hybrid** | WebView içinde HTML render eden native app (Cordova, Ionic) | Context switching (NATIVE_APP ↔ WEBVIEW), iki tarafı da test et |
| **Web (mobile browser)** | Mobile Chrome/Safari'de açılan responsive web | Selenium ile aynı; `browserName: Chrome/Safari` capability |

**Hybrid debug:** Chrome DevTools (Android) veya Safari Web Inspector (iOS) ile WebView'ı inspect edebilirsin.

---

## 9. Page Object Model (Mobile)

```java
// BasePage
public abstract class BasePage {
    protected AppiumDriver driver;
    protected WebDriverWait wait;

    public BasePage(AppiumDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    protected WebElement waitClickable(By locator) {
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }

    protected WebElement waitVisible(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }
}

// LoginPage
public class LoginPage extends BasePage {
    private final By emailField    = AppiumBy.accessibilityId("email_input");
    private final By passwordField = AppiumBy.accessibilityId("password_input");
    private final By loginBtn      = AppiumBy.accessibilityId("login_button");
    private final By errorMessage  = AppiumBy.accessibilityId("error_msg");

    public LoginPage(AppiumDriver driver) {
        super(driver);
    }

    public LoginPage enterEmail(String email) {
        waitVisible(emailField).sendKeys(email);
        return this;
    }

    public LoginPage enterPassword(String password) {
        waitVisible(passwordField).sendKeys(password);
        return this;
    }

    public DashboardPage submit() {
        waitClickable(loginBtn).click();
        return new DashboardPage(driver);
    }

    public String getErrorMessage() {
        return waitVisible(errorMessage).getText();
    }
}

// Test (TestNG)
@Test
public void successfulLogin() {
    new LoginPage(driver)
        .enterEmail("test@example.com")
        .enterPassword("secret")
        .submit()
        .assertLoaded();
}
```

### `@FindBy` annotation (PageFactory)
```java
public class LoginPage extends BasePage {
    @AndroidFindBy(accessibility = "email_input")
    @iOSXCUITFindBy(accessibility = "email_input")
    private WebElement emailField;

    @AndroidFindBy(uiAutomator = "new UiSelector().text(\"Login\")")
    @iOSXCUITFindBy(iOSClassChain = "**/XCUIElementTypeButton[`name == \"Login\"`]")
    private WebElement loginBtn;

    public LoginPage(AppiumDriver driver) {
        super(driver);
        PageFactory.initElements(new AppiumFieldDecorator(driver), this);
    }
}
```

**Cross-platform POM:** `@AndroidFindBy` + `@iOSXCUITFindBy` ile tek POM iki platforma servis verir.

---

## 10. CI/CD Entegrasyonu

### Lokal Appium server + CI
```yaml
# GitHub Actions örneği
jobs:
  android-test:
    runs-on: macos-latest        # KVM gereksin diye macos veya self-hosted
    steps:
      - uses: actions/checkout@v4
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
            ./gradlew connectedAndroidTest
            mvn test
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

### LambdaTest capability örneği
```java
DesiredCapabilities caps = new DesiredCapabilities();
caps.setCapability("platformName", "Android");
caps.setCapability("deviceName", "Galaxy S22");
caps.setCapability("platformVersion", "13");
caps.setCapability("app", "lt://APP123456");      // LT'ye yüklenmiş app
caps.setCapability("build", "Release-2024-01");
caps.setCapability("name", "Login flow test");
caps.setCapability("user", System.getenv("LT_USERNAME"));
caps.setCapability("accessKey", System.getenv("LT_ACCESS_KEY"));
caps.setCapability("visual", true);               // visual regression
caps.setCapability("network", true);              // network logs
caps.setCapability("video", true);
caps.setCapability("console", true);

URL hubUrl = new URL("https://mobile-hub.lambdatest.com/wd/hub");
AppiumDriver driver = new AndroidDriver(hubUrl, caps);
```

### Paralel test stratejisi
- **TestNG `parallel="methods"`** + cloud farm → onlarca cihazda eş zamanlı
- Her test session'ı izole olmalı (login state, deep link, vb.)
- Test data leak'i engellemek için `noReset: false` veya `fullReset: true`

---

## 11. Reporting

| Tool | Kullanım |
|---|---|
| **Allure** | En yaygın; screenshot embed, step-by-step trace, history |
| **ExtentReports** | Java odaklı, görsel HTML rapor |
| **TestNG HTML report** | Built-in, basit |
| **JUnit XML** | CI integration için |
| **Cucumber HTML report** | BDD framework kullanılıyorsa |

### Allure örneği (Java)
```java
@Test
@Description("Kullanıcı başarılı login olur")
@Severity(SeverityLevel.CRITICAL)
@Story("Authentication")
public void successfulLogin() {
    Allure.step("Login sayfasını aç", () -> {
        driver.get(LOGIN_URL);
    });

    Allure.step("Credentials gir", () -> {
        new LoginPage(driver).enterEmail("x@y.com").enterPassword("...");
    });

    // Screenshot on failure
    Allure.addAttachment("Final screenshot", "image/png",
        new ByteArrayInputStream(driver.getScreenshotAs(OutputType.BYTES)), ".png");
}
```

### Cloud provider native reporting
LambdaTest / BrowserStack: video recording, network logs, device logs, console logs, app logs **otomatik** sağlar. Her test için dashboard URL'i.

---

## 12. Sık Karşılaşılan Pitfall'lar

### 1. XPath aşırı kullanımı
```java
// ❌ Kırılgan + yavaş
driver.findElement(By.xpath("/hierarchy/android.widget.FrameLayout[1]/.../Button"));

// ✅ Accessibility ID
driver.findElement(AppiumBy.accessibilityId("login_button"));
```
**Çözüm:** Developer'lardan accessibility ID set etmesini iste — accessibility için zaten gerekli.

### 2. `Thread.sleep` kullanımı
```java
// ❌
click(login);
Thread.sleep(3000);
assertVisible(dashboard);

// ✅
click(login);
wait.until(ExpectedConditions.visibilityOf(dashboard));
```

### 3. Implicit + Explicit wait karışımı
İkisini birden kullanma → unpredictable timeout. Sadece **explicit** kullan.

### 4. Stale element exception
DOM yeniden render olunca eski element reference kaybolur.
```java
// ❌
WebElement btn = driver.findElement(...);
refreshPage();
btn.click();   // StaleElementException

// ✅
By btnLocator = AppiumBy.accessibilityId("btn");
refreshPage();
driver.findElement(btnLocator).click();
```

### 5. `noReset` ile state kirliliği
`noReset: true` hızlıdır ama test'ler arası state leak yapar. Critical flow'larda `fullReset: true` tercih et.

### 6. iOS WebDriverAgent setup sorunları
Gerçek iOS device'da WDA imzalama zor. Çözüm: cloud farm kullan, veya `XcodeOrgId` + `XcodeSigningId` capability'lerini doğru ver.

### 7. Emulator vs gerçek device farkı
Emulator'da çalışan test gerçek device'da fail edebilir (performans, GPU, sensor farkları). **Critical path'leri gerçek device'da çalıştır.**

### 8. Network koşullarını test etmemek
Offline, yavaş 3G, packet loss → cloud provider'lar network simulation sunar.

### 9. Locator stratejisinde platform-agnostic yazmaya çalışmak
İki platformda aynı UI bile farklı render edilir. Cross-platform POM'da `@AndroidFindBy` ve `@iOSXCUITFindBy` ayrı tut.

### 10. App version drift
Test build'leri kontrol altında olmalı — staging app'i değişirse test fail eder. **Build versioning** zorunlu.

---

## 13. Performans Optimizasyonu

- **Accessibility ID > XPath** — locator hızı 10x
- **Session reuse** — her test için yeni session açma; `@BeforeClass`'ta bir kere
- **Paralel test** — TestNG `parallel="classes"` veya `methods`
- **Cloud parallelism** — 10-50 cihazda eş zamanlı çalıştır
- **noReset: true** (state kirliliğine dikkat ederek) → setup süresini düşürür
- **Smart locator caching** — POM'da `@FindBy` lazy init kullan

---

## 14. Framework Tasarımı — QA Lead Bakışı

### Klasör yapısı (Java/Maven örneği)
```
src/test/java/
  ├── base/
  │   ├── BaseTest.java         # driver init/teardown
  │   └── BasePage.java         # POM base
  ├── pages/
  │   ├── android/
  │   ├── ios/
  │   └── common/               # cross-platform POM
  ├── tests/
  │   ├── login/
  │   ├── checkout/
  │   └── profile/
  ├── utils/
  │   ├── DriverFactory.java    # capability builder
  │   ├── ConfigReader.java
  │   ├── DeviceManager.java
  │   └── ScreenshotUtil.java
  ├── data/
  │   └── TestData.java         # JSON/YAML test data
  └── listeners/
      └── TestListener.java     # screenshot on failure
src/test/resources/
  ├── config/
  │   ├── android.properties
  │   └── ios.properties
  ├── apps/
  │   ├── app-debug.apk
  │   └── MyApp.app
  └── testng.xml
```

### DriverFactory pattern
```java
public class DriverFactory {
    private static final ThreadLocal<AppiumDriver> driver = new ThreadLocal<>();

    public static AppiumDriver getDriver() {
        return driver.get();
    }

    public static void initDriver(String platform) throws MalformedURLException {
        URL url = new URL(ConfigReader.get("appium.url"));
        if (platform.equalsIgnoreCase("android")) {
            driver.set(new AndroidDriver(url, AndroidCaps.build()));
        } else {
            driver.set(new IOSDriver(url, IOSCaps.build()));
        }
    }

    public static void quitDriver() {
        if (driver.get() != null) {
            driver.get().quit();
            driver.remove();
        }
    }
}
```
**`ThreadLocal`** — paralel test'lerde her thread'in kendi driver'ı olur.

### Cross-platform abstraction
```java
public interface LoginPage {
    LoginPage enterEmail(String email);
    LoginPage enterPassword(String password);
    DashboardPage submit();
}

public class AndroidLoginPage extends BasePage implements LoginPage { ... }
public class IOSLoginPage extends BasePage implements LoginPage { ... }

public class PageFactory {
    public static LoginPage loginPage(AppiumDriver driver, String platform) {
        return platform.equals("android")
            ? new AndroidLoginPage(driver)
            : new IOSLoginPage(driver);
    }
}
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
- **Mock backend** opsiyonu (WireMock, MockServer) → external dependency'i ortadan kaldır
- **Fixture'lar** — JSON test data dosyaları

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
| Native vs Hybrid vs Web? | Native: platform UI; Hybrid: WebView içinde HTML; Web: mobile browser |
| Hybrid app'i nasıl test edersin? | `getContextHandles` → `context("WEBVIEW_...")` ile switch |
| En iyi locator? | accessibility id; en hızlı, en güvenilir |
| XPath neden kötü? | Yavaş (tree traversal) + kırılgan (hierarchy değişimine duyarlı) |
| Wait stratejisi? | Sadece explicit wait; implicit + explicit karıştırma; `Thread.sleep` yasak |
| `noReset` vs `fullReset`? | noReset hızlı ama state leak; fullReset yavaş ama izole |
| Paralel test? | TestNG parallel + ThreadLocal driver + cloud farm |
| Cloud farm tercihi? | BrowserStack, Sauce Labs, LambdaTest — gerçek device + paralel |
| iOS gerçek device challenge? | WebDriverAgent imzalama + Apple cert; çoğu zaman cloud çözüm |
| Reporting? | Allure (en yaygın), ExtentReports, cloud native dashboard'lar |
| Cross-platform POM? | `@AndroidFindBy` + `@iOSXCUITFindBy` annotation'ları |
| Flaky test çözümü? | Explicit wait, locator audit, retry mekanizması (son çare), data isolation |
| CI/CD'de emulator? | macos runner + KVM, self-hosted, veya cloud farm |
| Test data yönetimi? | Test user pool, deep link ile state setup, mock backend |
| Device coverage? | P0 (her PR), P1 (nightly), P2 (release) tiered matrix |
| Lead'in en önemli kararı? | Hangi case Appium'da, hangisi API/unit'te → test piramidi |

---

## 17. Çalışma Yol Haritası

### 1. Hafta — Temeller
- [ ] Appium 2.x kurulumu (UiAutomator2 + XCUITest driver)
- [ ] Appium Inspector ile basit bir app'i incele
- [ ] İlk Android test'ini yaz (login flow)
- [ ] Locator stratejilerini dene (accessibility id, id, uiautomator, xpath)

### 2. Hafta — Orta
- [ ] Explicit wait + ExpectedConditions
- [ ] Touch gestures (swipe, scroll, long press) — mobile gesture extensions
- [ ] Hybrid app context switching
- [ ] Page Object Model — BasePage + 3 sayfa
- [ ] `@AndroidFindBy` + `@iOSXCUITFindBy` ile cross-platform POM

### 3. Hafta — Framework
- [ ] DriverFactory + ThreadLocal pattern
- [ ] TestNG paralel test
- [ ] Config management (properties / YAML)
- [ ] Test data fixture'ları
- [ ] Screenshot on failure listener
- [ ] Allure reporting entegrasyonu

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
- **Java client:** github.com/appium/java-client
- **Python client:** github.com/appium/python-client
- **Allure:** docs.qameta.io/allure/
- **Sample projeler:** github.com/appium-boneyard/sample-code
- **LambdaTest docs:** lambdatest.com/support/docs/appium-testing/
