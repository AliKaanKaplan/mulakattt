# Gatling — QA Lead Mülakat Hazırlık Rehberi (Java DSL)

> Yüksek-performanslı open-source load / stress testing framework'ü. Async I/O mimarisi sayesinde **tek bir makineden binlerce eşzamanlı virtual user** simüle eder.
>
> **Bu doküman Gatling'in Java DSL'ini esas alır.** Ruby DSL'i yoktur — Gatling sadece **Scala, Java ve Kotlin** ile yazılır. Java DSL Gatling 3.7+ ile geldi ve en çok tercih edilen seçenektir (Scala bilmek gerektirmez, JVM ekosistemiyle entegre).

---

## İçindekiler

1. [Gatling Nedir?](#1-gatling-nedir)
2. [Mimari — Neden bu kadar performanslı?](#2-mimari--neden-bu-kadar-performansli)
3. [Java vs Scala vs Kotlin DSL](#3-java-vs-scala-vs-kotlin-dsl)
4. [Kurulum](#4-kurulum)
5. [Proje Yapısı](#5-proje-yapisi)
6. [Temel Terimler](#6-temel-terimler)
7. [İlk Simulation — Hello World](#7-ilk-simulation--hello-world)
8. [HTTP Protocol Konfigürasyonu](#8-http-protocol-konfigurasyonu)
9. [Scenario DSL — exec, pause, pace, groups](#9-scenario-dsl)
10. [HTTP Request Tipleri](#10-http-request-tipleri)
11. [Checks — Assertion ve Data Extraction](#11-checks)
12. [Gatling Expression Language (EL)](#12-gatling-expression-language-el)
13. [Feeders — Test Data](#13-feeders)
14. [Session Manipulation](#14-session-manipulation)
15. [Conditional Logic](#15-conditional-logic)
16. [Loops](#16-loops)
17. [Pause Models (Think Time)](#17-pause-models)
18. [Injection Profiles — Open vs Closed Workload](#18-injection-profiles)
19. [Throttling — Rate Limiting](#19-throttling)
20. [Assertions — Global SLA](#20-assertions)
21. [Test Türleri — Load, Stress, Spike, Soak](#21-test-turleri)
22. [Recorder — HTTP Proxy / HAR Import](#22-recorder)
23. [WebSocket, SSE, JMS, MQTT, gRPC](#23-diger-protokoller)
24. [Reporting — HTML Rapor](#24-reporting)
25. [Gatling Enterprise](#25-gatling-enterprise)
26. [CI/CD Entegrasyonu](#26-cicd-entegrasyonu)
27. [Performans Metrikleri Derin Bakış](#27-performans-metrikleri)
28. [Sık Karşılaşılan Pitfall'lar](#28-pitfall-lar)
29. [Framework Tasarımı — QA Lead Bakışı](#29-framework-tasarimi)
30. [QA Lead Seviyesi — Mimari Kararlar](#30-mimari-kararlar)
31. [En Sık Mülakat Soruları](#31-mulakat-sorulari)
32. [Çalışma Yol Haritası](#32-calisma-yol-haritasi)
33. [Kaynaklar](#33-kaynaklar)

---

## 1. Gatling Nedir?

**Gatling**, JVM üzerinde çalışan, **Scala / Java / Kotlin DSL** ile senaryolar yazılan açık kaynak **yüksek performanslı load testing** aracıdır. 2012'de Stéphane Landelle tarafından oluşturuldu.

**Temel özellikleri:**
- **Async, non-blocking I/O** (Netty + Akka actor model) → tek makinede 10K+ eşzamanlı virtual user
- **Code as configuration** — XML değil, derlenebilir kod (Java/Scala/Kotlin)
- **Detaylı HTML raporlar** — built-in, çok zengin
- **HTTP, JMS, MQTT, SSE, WebSocket** protokolleri (HTTP en yaygın)
- **CI/CD-friendly** — Maven, Gradle, sbt plugin'leri; assertion'lar ile build gate
- **Gatling Enterprise** (eski FrontLine) → dağıtık load injection, real-time dashboard, history (ücretli)

**Versiyonlar:**
| Sürüm | Önemli özellikler |
|---|---|
| **Gatling 2.x** | Scala-only, eski |
| **Gatling 3.0–3.6** | Performance iyileştirmeleri, Scala 2 |
| **Gatling 3.7+** | **Java + Kotlin DSL eklendi** — Scala bilmek artık zorunlu değil |
| **Gatling 3.9–3.11** | Java DSL stabilize, Akka → Pekko geçişi (lisans nedeniyle) |

**Karşılaştırma:**
| Özellik | Gatling | JMeter | k6 | Locust |
|---|---|---|---|---|
| Mimari | Async (Netty + Akka) | Thread-per-user | Async (Go) | Async (Python) |
| Single-node capacity | 10K+ VU | ~1K VU | 10K+ VU | ~1-5K VU |
| DSL dili | Scala / Java / Kotlin | XML + GUI | JavaScript | Python |
| Reporting | HTML built-in, zengin | Basic + plugin | Cloud dashboard | Real-time web UI |
| Learning curve | Orta | Düşük | Düşük | Düşük |
| Code review | ✅ Kod | ⚠️ XML | ✅ Kod | ✅ Kod |
| Distributed | Enterprise (paid) | Open source slave/master | Cloud + K8s | Built-in master/worker |

---

## 2. Mimari — Neden bu kadar performanslı?

### Sorun: JMeter'ın thread-per-user yaklaşımı
JMeter her virtual user için bir **OS thread** açar. 5000 VU = 5000 OS thread:
- Her thread ~1MB stack memory → 5GB RAM
- OS scheduler context switching → CPU israfı
- I/O wait sırasında thread bloke olur, kaynak boşa harcanır

Sonuç: tek makinede gerçekçi olarak ~1000-1500 VU.

### Çözüm: Gatling'in async actor modeli

Gatling iki teknolojinin üstüne kurulu:

**1. Netty (async I/O kütüphanesi)**
- Tek bir thread aynı anda yüzlerce HTTP connection'ı yönetebilir
- Bir VU HTTP request gönderir, Netty event loop diğer VU'lara döner
- Response geldiğinde callback ile devam eder
- **Thread'i bloke etmez**

**2. Akka / Pekko (actor model)**
- Her virtual user **lightweight actor**'dır (~600 byte, OS thread değil)
- Aynı OS thread'de binlerce actor scheduling yapılır
- Message-passing ile state yönetilir

```
JMeter:           5000 VU = 5000 OS thread = ~5GB RAM, GC overhead
Gatling:          5000 VU = 5000 actor + ~10 OS thread = ~100MB RAM
```

### QA Lead bakışı — mülakat klasiği
**Soru:** *"Gatling neden JMeter'dan daha az kaynak tüketir?"*
**Cevap:** **Non-blocking I/O (Netty) + actor model (Akka)**. VU'lar OS thread değil, aynı thread havuzunda scheduling yapılan lightweight actor. I/O wait sırasında thread bloke olmaz.

---

## 3. Java vs Scala vs Kotlin DSL

| Dil | Eklendiği sürüm | Avantaj | Dezavantaj |
|---|---|---|---|
| **Scala** | 1.x (orijinal) | En özlü, Gatling'in canonical DSL'i, en fazla örnek | Scala öğrenme eğrisi |
| **Java** | 3.7 (2021) | Tanıdık syntax, JVM ekosistemi, IDE desteği | Daha verbose (boilerplate fazla) |
| **Kotlin** | 3.7 (2021) | Scala'ya yakın özlülük, Java interop, modern syntax | Toplulukta daha az örnek |

**QA Lead seçim mantığı:**
- Team Java biliyor → **Java DSL** (öğrenme yok, hemen başla)
- Team Scala biliyor veya öğrenmek istiyor → **Scala DSL** (en idiomatic)
- Team Kotlin kullanıyor → **Kotlin DSL** (modern, expressive)

> Bu doküman boyunca **Java DSL** kullanıyoruz.

### Aynı senaryo, üç dilde — karşılaştırma

**Java:**
```java
ScenarioBuilder scn = scenario("Browse")
    .exec(http("Home").get("/"))
    .pause(2)
    .exec(http("Search").get("/search?q=phone"));
```

**Scala:**
```scala
val scn = scenario("Browse")
    .exec(http("Home").get("/"))
    .pause(2)
    .exec(http("Search").get("/search?q=phone"))
```

**Kotlin:**
```kotlin
val scn = scenario("Browse")
    .exec(http("Home").get("/"))
    .pause(2)
    .exec(http("Search").get("/search?q=phone"))
```

Pratikte üçü de neredeyse aynı — Java'da `;`, Scala/Kotlin'de `val` farkları dışında.

---

## 4. Kurulum

### Yöntem 1: Maven (Java DSL — önerilen)

**`pom.xml`:**
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>load-tests</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <gatling.version>3.11.5</gatling.version>
        <gatling-maven-plugin.version>4.10.2</gatling-maven-plugin.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>io.gatling.highcharts</groupId>
            <artifactId>gatling-charts-highcharts</artifactId>
            <version>${gatling.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.gatling</groupId>
                <artifactId>gatling-maven-plugin</artifactId>
                <version>${gatling-maven-plugin.version}</version>
            </plugin>
        </plugins>
    </build>
</project>
```

**Komutlar:**
```bash
# Tüm simulation'ları interactive seçimle koş
mvn gatling:test

# Belirli bir simulation'ı koş
mvn gatling:test -Dgatling.simulationClass=simulations.BasicSimulation

# Maven phase'e bağla (her test'te koşsun)
mvn verify
```

### Yöntem 2: Gradle
```kotlin
// build.gradle.kts
plugins {
    java
    id("io.gatling.gradle") version "3.11.5"
}
```

```bash
./gradlew gatlingRun
./gradlew gatlingRun-simulations.BasicSimulation
```

### Yöntem 3: Standalone bundle (CI dışı, lokal deney için)
```bash
# gatling.io/open-source/ → ZIP indir
unzip gatling-charts-highcharts-bundle-3.11.5-bundle.zip
cd gatling-charts-highcharts-bundle-3.11.5

# Simulation'ı user-files/simulations/ içine koy
bin/gatling.sh
```

### Gatling Recorder (proxy/HAR import — bölüm 22)
```bash
bin/recorder.sh
```

---

## 5. Proje Yapısı

Maven konvansiyonu:

```
load-tests/
├── pom.xml
├── src/
│   └── test/
│       ├── java/
│       │   ├── simulations/
│       │   │   ├── BasicSimulation.java
│       │   │   ├── LoadSimulation.java
│       │   │   ├── StressSimulation.java
│       │   │   └── SoakSimulation.java
│       │   ├── scenarios/
│       │   │   ├── LoginScenarios.java
│       │   │   ├── CheckoutScenarios.java
│       │   │   └── BrowseScenarios.java
│       │   ├── config/
│       │   │   ├── HttpConfig.java
│       │   │   └── Env.java
│       │   └── utils/
│       │       └── DataGenerator.java
│       └── resources/
│           ├── data/
│           │   ├── users.csv
│           │   └── products.json
│           ├── bodies/
│           │   └── createUser.json
│           ├── gatling.conf            # Gatling runtime config
│           └── logback-test.xml        # Logging
└── target/
    └── gatling/                        # Otomatik HTML rapor çıktısı
        └── basicsimulation-YYYYMMDD/
            └── index.html
```

**Java DSL'de package isimleri:** Simulation class'ları `simulations` package'ında, ama Gatling herhangi bir package'ı kabul eder. Convention: tüm load test kodu `src/test/java/` altında, **production kod'dan ayrı**.

---

## 6. Temel Terimler

Bu terimler her Gatling sorusunun temelidir. Mülakatta **kesin öğrenilmesi gereken** kavramlar:

### Simulation
Bir test senaryosunun **tamamı**. Bir Java class'ıdır, `io.gatling.javaapi.core.Simulation`'dan extend eder. Bir simulation içinde birden fazla scenario, protocol konfigürasyonu ve injection profile bulunabilir.

```java
public class BasicSimulation extends Simulation {
    // ...
}
```

### Scenario
Bir virtual user'ın yapacağı **action zinciri**. Bir simulation'da birden fazla scenario olabilir (örn. "browser" + "checkout" eş zamanlı).

```java
ScenarioBuilder browse = scenario("Browse")
    .exec(http("Home").get("/"))
    .pause(2)
    .exec(http("Search").get("/search"));
```

### Virtual User (VU)
Test'te simüle edilen **tek bir kullanıcı**. Her VU bağımsız bir actor; kendi state'i, kendi session'ı vardır. Aynı scenario binlerce VU tarafından paralel koşturulur.

### Session
Bir VU'nun **kendi key-value state'i**. Feeder'dan gelen data, capture edilen response value'lar burada saklanır. Her VU için izole.

```java
.exec(session -> session.set("userId", 42))
.exec(http("Profile").get("/users/#{userId}"))
```

### Feeder
Senaryolara **test data sağlayan iterator**. CSV, JSON, JDBC, Redis veya custom kaynaktan beslenir.

```java
FeederBuilder<String> users = csv("users.csv").random();
```

### Check
**Response üzerinde assertion ve/veya data extraction**. Status code, header, body içeriği kontrol eder; istersen response'dan değer çıkarıp session'a kaydeder.

```java
http("Login").post("/login")
    .check(status().is(200))
    .check(jsonPath("$.token").saveAs("authToken"))
```

### Assertion
**Test sonunda global metrikler üzerinde doğrulama**. Bunlar fail olursa **build fail olur** (CI/CD entegrasyonu için kritik).

```java
.assertions(
    global().responseTime().percentile3().lt(500),
    global().successfulRequests().percent().gt(99.0)
)
```

### Injection Profile
**VU'ların zaman içinde nasıl enjekte edileceği**. Ramp-up, constant, peak, vb.

```java
scn.injectOpen(rampUsers(100).during(60))
```

### Pause
**Think time** — gerçek user davranışı simülasyonu. Action'lar arasına eklenir.

```java
.pause(2)              // sabit 2 sn
.pause(2, 5)           // uniform 2-5 sn arası
```

### Pace
**Rate-limiting** — bir önceki exec'ten itibaren sabit bir periyot bekletir.

```java
.pace(5)               // exec'ler arası en az 5 sn (rate = 12/dk)
```

### Group
Raporlamada **birden fazla request'i tek bir blok olarak** gösteren wrapper.

```java
.group("Checkout flow").on(
    exec(http("Cart").get("/cart")),
    exec(http("Pay").post("/pay"))
)
```

### HTTP Protocol Configuration
Tüm scenario'ların kullanacağı **default HTTP ayarları** (base URL, header'lar, vb.).

```java
HttpProtocolBuilder httpProtocol = http
    .baseUrl("https://api.example.com")
    .acceptHeader("application/json");
```

### Throttling
**Saniyedeki request sayısını sınırlama**. Injection ile bağımsız, request seviyesinde rate limit.

```java
.throttle(reachRps(100).in(10), holdFor(60))
```

---

## 7. İlk Simulation — Hello World

```java
package simulations;

import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;

public class BasicSimulation extends Simulation {

    // 1. HTTP protokol konfigürasyonu
    HttpProtocolBuilder httpProtocol = http
        .baseUrl("https://api.example.com")
        .acceptHeader("application/json")
        .userAgentHeader("Gatling/LoadTest");

    // 2. Scenario tanımı
    ScenarioBuilder scn = scenario("Basic browse")
        .exec(http("Home")
            .get("/"))
        .pause(1)
        .exec(http("Search")
            .get("/search?q=phone")
            .check(status().is(200))
            .check(jsonPath("$.results[0].id").saveAs("productId")))
        .pause(2)
        .exec(http("Product detail")
            .get("/products/#{productId}")
            .check(jsonPath("$.name").exists()));

    // 3. Simulation kurulumu (injection + assertions)
    {
        setUp(
            scn.injectOpen(
                rampUsers(100).during(60),            // 60 sn'de 100 VU ekle
                constantUsersPerSec(10).during(120)   // 2 dk boyunca sn'de 10 VU
            )
        )
        .protocols(httpProtocol)
        .assertions(
            global().responseTime().percentile3().lt(500),
            global().successfulRequests().percent().gt(99.0)
        );
    }
}
```

**Açıklamalar:**
1. `http` builder ile **base URL** ve default header'lar
2. `scenario(...)` zincir API'si — `exec` ile request, `pause` ile think time, `check` ile assertion/extraction
3. **Instance initializer block (`{ ... }`)** içinde `setUp(...)` çağrısı — Java DSL'de simulation kurulumu burada yapılır (Scala'daki top-level statement karşılığı)
4. `rampUsers`, `constantUsersPerSec` → injection method'ları (bölüm 18)
5. `global().responseTime().percentile3().lt(500)` → p95 < 500ms assertion'ı

**Çalıştırma:**
```bash
mvn gatling:test -Dgatling.simulationClass=simulations.BasicSimulation
```

**Çıktı:** `target/gatling/basicsimulation-YYYYMMDD/index.html`

---

## 8. HTTP Protocol Konfigürasyonu

`HttpProtocolBuilder` tüm scenario'lar için **ortak HTTP ayarlarını** tanımlar.

```java
HttpProtocolBuilder httpProtocol = http
    // Base URL
    .baseUrl("https://api.example.com")
    .baseUrls("https://api1.example.com", "https://api2.example.com")   // round-robin

    // Default header'lar
    .acceptHeader("application/json")
    .acceptLanguageHeader("en-US,en;q=0.9")
    .acceptEncodingHeader("gzip, deflate, br")
    .userAgentHeader("Gatling/3.11")
    .authorizationHeader("Bearer #{token}")                              // session var'dan
    .header("X-API-Version", "v2")
    .header("X-Request-Id", session -> UUID.randomUUID().toString())     // dynamic

    // Redirect / cache
    .disableFollowRedirect()
    .maxRedirects(5)
    .disableCaching()

    // Connection
    .shareConnections()                                                  // connection pool paylaşımı
    .maxConnectionsPerHost(20)
    .connectionHeader("keep-alive")

    // Resource handling
    .silentResources()                                                   // CSS/JS/img stat'lara dahil değil
    .inferHtmlResources()                                                // HTML'den embedded resource'ları auto-fetch

    // Default check'ler (her response'a uygulanır)
    .check(status().not(500))
    .check(status().not(404))

    // Proxy
    .proxy(Proxy("proxy.example.com", 8080).httpsPort(8443))

    // Body handling
    .disableWarmUp()
    .warmUp("https://www.google.com")                                    // JIT warm-up için ilk request

    // Cookies
    .disableCookies();
```

### Önemli detay: `inferHtmlResources` ve `silentResources`

**`inferHtmlResources()`** — Browser gibi davranır:
- HTML response geldiğinde içindeki `<script>`, `<link>`, `<img>` tag'lerini parse eder
- Bu resource'ları otomatik fetch eder
- "Gerçek browser load" simülasyonu için iyidir

**Trade-off:**
- ✅ Gerçekçi: gerçek user CSS/JS/img'i de indirir
- ❌ Test'in ne ölçtüğü bulanıklaşır: backend API mi yoksa CDN mi?

**`silentResources()`** — Inferred resource'ları **rapora dahil etmez**:
- Fetch yapar ama metric'lerde görünmez
- Backend API performans'ını izole eder

**QA Lead kararı:**
- Saf backend API test → `silentResources` veya `inferHtmlResources` kapalı
- Full-page render time → `inferHtmlResources` açık, `silentResources` kapalı

---

## 9. Scenario DSL

### `exec` — Action tanımı
```java
scenario("Browse")
    .exec(http("Home").get("/"))                                  // tek request
    .exec(http("Login").post("/login")                             // POST + form
        .formParam("user", "x").formParam("pass", "y"))
    .exec(http("Profile").get("/profile"));
```

### `pause` — Think time
```java
.pause(2)                                       // sabit 2 sn
.pause(2, 5)                                    // uniform random 2-5 sn
.pause(Duration.ofSeconds(3))                   // Duration objesi
.pause("#{thinkTime}")                          // session var
.pause("#{thinkTime}", "#{thinkTimeMax}")
.pause(2, PauseType.Exponential)                // exponential distribution
```

### `pace` — Rate limiting (loop içinde)
```java
.during(60).on(
    exec(http("Poll").get("/poll"))
        .pace(5)                                // her exec arası en az 5 sn
)
```
**Fark:** `pause` request'ten **sonra** bekler. `pace` request'ten **önce** "son exec'ten beri 5 sn geçti mi?" diye kontrol eder. Sabit rate (örn. dakikada 12 request) için `pace` kullanılır.

### `group` — Raporlama wrapper
```java
.group("Login flow").on(
    exec(http("Open login").get("/login"))
        .exec(http("Submit").post("/login").formParam("u", "x"))
        .exec(http("Dashboard").get("/dashboard"))
)
```

Group'lar raporda **tek bir collapsed blok** olarak görünür: "Login flow" group'unun ortalama RT'si, p95'i, vb. Sub-request'leri ayrı ayrı da görebilirsin.

### Senaryo birleştirme — reusable chains
```java
ChainBuilder login = exec(http("Login")
    .post("/login")
    .formParam("email", "#{email}")
    .formParam("password", "#{password}")
    .check(jsonPath("$.token").saveAs("token")));

ChainBuilder browse = exec(http("Products")
    .get("/products")
    .header("Authorization", "Bearer #{token}"));

ScenarioBuilder fullFlow = scenario("Full")
    .exec(login)
    .pause(2)
    .exec(browse);
```

`ChainBuilder` Java'da action zincirini scenario'dan ayrı **reuse edilebilir** parça olarak tutar.

---

## 10. HTTP Request Tipleri

### GET
```java
http("Search")
    .get("/search")
    .queryParam("q", "phone")
    .queryParam("page", "1")
    .header("X-Locale", "tr-TR")
```

### POST — Form params
```java
http("Login")
    .post("/login")
    .formParam("email", "#{email}")
    .formParam("password", "#{password}")
```

### POST — JSON body
```java
http("Create user")
    .post("/users")
    .header("Content-Type", "application/json")
    .body(StringBody("{\"name\":\"John\",\"email\":\"john@x.com\"}"))
```

### POST — JSON body with EL
```java
http("Create order")
    .post("/orders")
    .body(StringBody("{\"userId\":\"#{userId}\",\"productId\":\"#{productId}\"}"))
    .asJson()                                  // Content-Type: application/json otomatik
```

### POST — Body dosyadan
```java
http("Create user")
    .post("/users")
    .body(RawFileBody("bodies/createUser.json"))            // raw, EL parsing yok
    .asJson()

http("Create user with EL")
    .post("/users")
    .body(ElFileBody("bodies/createUser.json"))             // EL parsing var
    .asJson()
```

**`bodies/createUser.json`:**
```json
{
  "name": "#{firstName}",
  "email": "#{email}",
  "createdAt": "#{now()}"
}
```

### Multipart / file upload
```java
http("Upload avatar")
    .post("/upload")
    .formUpload("file", "data/avatar.png")
    .formParam("userId", "#{userId}")
```

### PUT / PATCH / DELETE
```java
http("Update user")
    .put("/users/#{userId}")
    .body(StringBody("{\"name\":\"Updated\"}"))
    .asJson()

http("Delete user")
    .delete("/users/#{userId}")
```

### Headers — global vs per-request
```java
// Global (HttpProtocolBuilder)
http.header("X-API-Key", "12345")

// Per request
http("Foo").get("/foo").header("X-Custom", "bar")
```

---

## 11. Checks — Assertion ve Data Extraction

Check'ler iki amaca hizmet eder:
1. **Assertion:** Response istediğimiz gibi mi? (Olmazsa request fail sayılır.)
2. **Data extraction:** Response'dan değer çıkarıp **session'a kaydet** → sonraki request'lerde kullan.

### Status code
```java
.check(status().is(200))
.check(status().in(200, 201, 204))
.check(status().not(500))
.check(status().lt(400))
```

### Header
```java
.check(header("Content-Type").is("application/json"))
.check(header("Set-Cookie").exists())
.check(headerRegex("Set-Cookie", "session_id=(.+?);").saveAs("sessionId"))
```

### Response time
```java
.check(responseTimeInMillis().lt(500))
```

### JSON Path
```java
.check(jsonPath("$.id").exists())
.check(jsonPath("$.name").is("iPhone"))
.check(jsonPath("$.price").ofInt().gt(0))
.check(jsonPath("$.id").saveAs("productId"))
.check(jsonPath("$.tags[*]").findAll().saveAs("tags"))          // array as List
.check(jsonPath("$.items[*].id").findRandom().saveAs("randomId"))
.check(jsonPath("$.items[*].id").count().saveAs("itemCount"))
```

### JMESPath (AWS-style JSON query)
```java
.check(jmesPath("results[0].id").saveAs("firstId"))
```

### Regex (response body)
```java
.check(regex("user_id=([0-9]+)").saveAs("userId"))
.check(regex("<title>(.+?)</title>").find().is("My Shop"))
.check(regex("error").notExists())
```

### CSS selector (HTML)
```java
.check(css("h1.title", "innerText").saveAs("title"))
.check(css("input[name=csrf]", "value").saveAs("csrfToken"))
```

### XPath (XML/HTML)
```java
.check(xpath("//title").is("My Shop"))
```

### Substring (raw body)
```java
.check(substring("success").exists())
.check(substring("error").notExists())
.check(substring("Welcome").count().is(1))
```

### Raw body
```java
.check(bodyString().saveAs("rawBody"))
.check(bodyBytes().is(expectedBytes))
```

### Multiple checks tek request'te
```java
http("Login")
    .post("/login")
    .formParam("email", "#{email}")
    .formParam("password", "#{password}")
    .check(status().is(200))
    .check(jsonPath("$.token").exists().saveAs("token"))
    .check(jsonPath("$.user.id").saveAs("userId"))
    .check(responseTimeInMillis().lt(1000))
```

### Saved variable kullanımı
```java
.exec(http("Use saved")
    .get("/users/#{userId}/orders")
    .header("Authorization", "Bearer #{token}"))
```

`#{userId}` Gatling Expression Language (Gatling EL) — bölüm 12.

### Optional checks — `optional()`
```java
.check(jsonPath("$.optionalField").optional().saveAs("field"))
```
Bulamasa hata vermez, session'a koymaz.

### Default value — `withDefault()`
```java
.check(jsonPath("$.field").withDefault("fallback").saveAs("field"))
```

### Transform
```java
.check(jsonPath("$.price").ofInt().transform(p -> p * 1.18).saveAs("priceWithVat"))
```

---

## 12. Gatling Expression Language (EL)

Gatling EL, **session içindeki değerlere** ve **fonksiyonlara** erişmenin string-template yoludur.

### Temel syntax
```java
"#{variableName}"                      // session'dan oku
"User #{userId} created"                // string içine göm
"https://api.com/users/#{userId}"
```

### Built-in EL fonksiyonları
| Fonksiyon | Açıklama |
|---|---|
| `#{now()}` | Şu anki timestamp (millis) |
| `#{currentTimeMillis()}` | Aynı, alternatif isim |
| `#{currentDate(yyyy-MM-dd)}` | Formatted current date |
| `#{randomUuid()}` | UUID v4 |
| `#{randomInt(0, 100)}` | Random int |
| `#{randomLong()}` | Random long |
| `#{randomSecureUuid()}` | Cryptographically secure UUID |

### Collection erişimi
```java
"#{users(0)}"             // List'in 0. elemanı
"#{users(0).email}"       // List[0].email
"#{user.address.city}"    // nested object
"#{users.size()}"         // collection size
"#{users.random()}"       // random element
```

### Karmaşık expression — Java function olarak
EL yetmediğinde Java function olarak yaz:
```java
.exec(http("Foo")
    .get(session -> "/users/" + session.getInt("userId") + "/orders"))

.exec(http("Bar")
    .post("/items")
    .body(StringBody(session -> {
        int id = session.getInt("userId");
        return "{\"userId\":" + id + ",\"timestamp\":" + System.currentTimeMillis() + "}";
    })))
```

---

## 13. Feeders — Test Data

Feeder = senaryolara data sağlayan **iterator**. Her exec'te feeder'dan bir record alınır ve session'a yazılır.

### CSV
```java
FeederBuilder<String> usersFeeder = csv("data/users.csv").random();

scenario("Login")
    .feed(usersFeeder)
    .exec(http("Login")
        .post("/login")
        .formParam("email", "#{email}")
        .formParam("password", "#{password}"));
```

**`users.csv`:**
```csv
email,password
user1@x.com,pass1
user2@x.com,pass2
user3@x.com,pass3
```

### Feeder strategies
```java
csv("users.csv").queue();        // default — FIFO; VU > data → "no more data" hatası
csv("users.csv").random();       // sınırsız rastgele
csv("users.csv").shuffle();      // bir kez karıştır, sıra ile dağıt
csv("users.csv").circular();     // sona gelince başa dön (loop)
```

### Diğer formatlar
```java
jsonFile("data/products.json").random()
jsonUrl("https://api.com/test-data.json").random()
tsv("data/users.tsv").random()
ssv("data/users.ssv").random()          // semicolon-separated
```

### JDBC feeder
```java
jdbcFeeder(
    "jdbc:postgresql://db.example.com:5432/test",
    "user",
    "pass",
    "SELECT email, password FROM test_users"
).random()
```

### Redis feeder
```java
redisFeeder(RedisClientPool.create("redis://localhost:6379"), "users:queue")
```

### Programmatic feeder
```java
Iterator<Map<String, Object>> customFeeder = Stream.generate(
    (Supplier<Map<String, Object>>) () -> Map.of(
        "email",    "user" + ThreadLocalRandom.current().nextInt() + "@test.com",
        "password", "pass" + System.currentTimeMillis()
    )
).iterator();

scenario("Generated users")
    .feed(customFeeder)
    .exec(http("Register").post("/register").formParam("email", "#{email}"));
```

### Birden fazla record per feed
```java
.feed(usersFeeder, 5)        // 5 record al, session'da email1, email2, ..., email5
```

### Feed inside loop
```java
.repeat(3).on(
    feed(usersFeeder)
        .exec(http("Register").post("/register"))
)
```

---

## 14. Session Manipulation

Bazen response'tan extract etmek yetmez — session'a manuel değer set etmek/değiştirmek istersin.

```java
.exec(session -> session.set("counter", 0))

.exec(session -> {
    int c = session.getInt("counter");
    return session.set("counter", c + 1);
})

.exec(session -> session.remove("oldValue"))

.exec(session -> {
    if (session.getString("token") == null) {
        return session.markAsFailed();   // virtual user'ı fail et
    }
    return session;
})
```

### `Session` API
| Method | Açıklama |
|---|---|
| `session.set(key, value)` | Key-value ekle/güncelle |
| `session.setAll(Map)` | Birden fazla ekle |
| `session.remove(key)` | Sil |
| `session.contains(key)` | Var mı? |
| `session.get(key)` | Object olarak al |
| `session.getString(key)`, `getInt`, `getLong`, ... | Tipli getter |
| `session.markAsSucceeded()` | Bu VU'yu success olarak işaretle |
| `session.markAsFailed()` | Bu VU'yu fail olarak işaretle |

---

## 15. Conditional Logic

### `doIf` — single condition
```java
.doIf("#{loggedIn}").then(
    exec(http("Profile").get("/profile"))
)

.doIf(session -> session.getInt("amount") > 100).then(
    exec(http("Premium").get("/premium"))
)
```

### `doIfOrElse`
```java
.doIfOrElse(session -> session.getInt("amount") > 100)
    .then(exec(http("Premium").get("/premium")))
    .orElse(exec(http("Free").get("/free")))
```

### `doIfEquals` — quick equality check
```java
.doIfEquals("#{userType}", "admin").then(
    exec(http("Admin").get("/admin"))
)

.doIfEqualsOrElse("#{userType}", "admin")
    .then(exec(http("Admin").get("/admin")))
    .orElse(exec(http("Home").get("/")))
```

### `doSwitch` — explicit value mapping
```java
.doSwitch("#{userType}").on(
    Choice.withKey("admin", exec(http("Admin").get("/admin"))),
    Choice.withKey("user",  exec(http("Home").get("/"))),
    Choice.withKey("guest", exec(http("Public").get("/public")))
)

.doSwitchOrElse("#{userType}")
    .on(
        Choice.withKey("admin", exec(http("Admin").get("/admin")))
    )
    .orElse(exec(http("Default").get("/")))
```

### `randomSwitch` — ağırlıklı random
```java
.randomSwitch().on(
    Choice.withWeight(70.0, exec(http("Search").get("/search"))),
    Choice.withWeight(20.0, exec(http("Browse").get("/browse"))),
    Choice.withWeight(10.0, exec(http("Cart").get("/cart")))
)
```
%70 search, %20 browse, %10 cart. Toplam **100 olmak zorunda değil** — Gatling normalize eder.

### `uniformRandomSwitch` — eşit dağılım
```java
.uniformRandomSwitch().on(
    exec(http("Path A").get("/a")),
    exec(http("Path B").get("/b")),
    exec(http("Path C").get("/c"))
)
```

### `roundRobinSwitch` — sırayla
```java
.roundRobinSwitch().on(
    exec(http("Server 1").get("/api1")),
    exec(http("Server 2").get("/api2"))
)
```

---

## 16. Loops

### `repeat` — sabit sayıda
```java
.repeat(5).on(
    exec(http("Item").get("/item/#{i}"))    // #{i} index counter, 0..4
)

.repeat(5, "counter").on(                    // custom counter name
    exec(http("Item").get("/item/#{counter}"))
)

.repeat("#{itemCount}").on(                  // session var ile
    exec(http("Item").get("/item"))
)
```

### `during` — belirli süre boyunca
```java
.during(Duration.ofMinutes(5)).on(
    exec(http("Poll").get("/poll"))
        .pause(2)
)

.during(60).on(                              // 60 saniye
    exec(http("Heartbeat").get("/ping"))
        .pace(1)
)
```

### `forever` — sonsuz loop
```java
.forever().on(
    exec(http("Stream").get("/stream"))
        .pause(1)
)
```
Genellikle `maxDuration()` ile birlikte kullanılır.

### `foreach` — koleksiyon üzerinde iterate
```java
// jsonPath ile birden fazla ID extract et
.exec(http("Get items")
    .get("/items")
    .check(jsonPath("$.items[*].id").findAll().saveAs("itemIds")))

// Her ID için bir request
.foreach("#{itemIds}", "itemId").on(
    exec(http("Item detail")
        .get("/items/#{itemId}"))
)
```

### `asLongAs` / `doWhile` — condition-based
```java
.asLongAs(session -> session.getInt("count") < 10).on(
    exec(http("Increment")
        .post("/incr")
        .check(jsonPath("$.count").ofInt().saveAs("count")))
)

.doWhile(session -> session.getBoolean("hasMore")).on(
    exec(http("Fetch next").get("/next?cursor=#{cursor}"))
)
```

`asLongAs` → loop **başında** condition kontrol eder. `doWhile` → loop **sonunda** kontrol eder (en az 1 kez koşar).

---

## 17. Pause Models (Think Time)

Gerçek user'ların düşünme süresini simüle eder. Default tüm `pause(min, max)` çağrıları **uniform** dağılımdır.

### Pause distribution tipleri

```java
// Constant — her seferinde aynı
.pause(2)

// Uniform — min/max arası homojen
.pause(2, 5)

// Exponential — ortalamaya yakın daha sık
setUp(scn).pauses(PauseType.Exponential)

// Normal (Gaussian)
setUp(scn).pauses(PauseType.NormalWithStdDev(2.0))    // stddev = 2 sn
setUp(scn).pauses(PauseType.NormalWithPercentage(20)) // stddev = %20

// Disable pause (test'te think time'ı yok say)
setUp(scn).pauses(PauseType.Disabled)

// Custom function
setUp(scn).pauses(session ->
    java.time.Duration.ofMillis(ThreadLocalRandom.current().nextLong(500, 3000))
)
```

### Per-pause override
```java
.pause(Duration.ofSeconds(2), PauseType.Exponential)
```

### QA Lead kararı
- **Production-realistic test:** Uniform veya exponential, gerçek user davranışı
- **Maksimum stress:** `PauseType.Disabled` (think time yok, sistem üstüne tüm yük)
- **Hesaplanmış RPS:** `pace()` ile sabit rate

---

## 18. Injection Profiles — Open vs Closed Workload

**En kritik Lead seviyesi konu.** Mülakat klasiği.

### Open Workload Model
- **Arrival rate** (saniyede X user gelir) ile model
- Yeni user'lar **mevcut user'ların durumundan bağımsız** gelir
- Sistem yavaşlarsa **queue büyür** (backpressure görülür)
- Gerçek dünya: web sitesi, e-commerce traffic

### Closed Workload Model
- **Sabit sayıda concurrent user** korunur
- Bir VU senaryosunu bitirdiğinde yeni biri başlar (toplam concurrent sabit)
- Sistem yavaşlarsa **throughput düşer**, VU sayısı sabit kalır
- Gerçek dünya: call center, fixed-pool sistem, kapasite planlama

### Mülakat sorusu (klasik):
*"Open vs closed workload modelinin farkı nedir? Hangisini ne zaman seçersin?"*

**Cevap:**
- **Open** → gerçek user trafiği simülasyonu (peak traffic, marketing campaign). Sistem yavaşladığında **queue oluşur, gerçek darboğazı yansıtır**.
- **Closed** → mevcut user kapasitesini test etme. Sistem yavaşladığında **throughput düşer**, başarısızlık modu farklıdır.
- **Yanlış model seçimi yanlış sonuç verir.** Bir e-commerce sitesinde closed model kullanmak gerçek peak traffic davranışını maskeler.

### Open Injection Methods

```java
scn.injectOpen(
    nothingFor(Duration.ofSeconds(5)),              // 5 sn hiçbir şey yapma
    atOnceUsers(50),                                // anlık 50 VU
    rampUsers(200).during(120),                     // 2 dk'da 200 VU lineer ramp
    constantUsersPerSec(20).during(60),             // 1 dk boyunca sn'de 20 VU
    rampUsersPerSec(10).to(50).during(120),         // 2 dk'da rate 10→50/sn ramp
    constantUsersPerSec(100).during(60).randomized(), // Poisson dağılımı (gerçekçi)
    heavisideUsers(1000).during(60),                // S-curve (Heaviside step)
    stressPeakUsers(500).during(30),                // exponential peak
    incrementUsersPerSec(10)                        // step-wise increment
        .times(5)
        .eachLevelLasting(30)
        .startingFrom(10)
);
```

| Method | Davranış |
|---|---|
| `nothingFor(d)` | `d` süresince hiçbir şey enjekte etme (delay) |
| `atOnceUsers(n)` | Anında `n` VU |
| `rampUsers(n).during(d)` | `d` süresinde `n` VU lineer ramp |
| `constantUsersPerSec(r).during(d)` | `d` süresinde sn'de `r` VU enjekte (sabit rate) |
| `rampUsersPerSec(from).to(to).during(d)` | `d` süresinde rate `from`→`to` ramp |
| `.randomized()` | Lineer yerine Poisson dağılımı (daha gerçekçi) |
| `heavisideUsers(n).during(d)` | S-curve dağılım (matematiksel olarak smooth) |
| `stressPeakUsers(n).during(d)` | Exponential peak (hızlı ramp) |
| `incrementUsersPerSec(r).times(n).eachLevelLasting(d).startingFrom(s)` | Step-wise (her `d` sn'de rate `r` artar) |

### Closed Injection Methods

```java
scn.injectClosed(
    constantConcurrentUsers(100).during(Duration.ofMinutes(5)),
    rampConcurrentUsers(10).to(100).during(120),
    incrementConcurrentUsers(20)
        .times(5)
        .eachLevelLasting(60)
        .startingFrom(10)
);
```

| Method | Davranış |
|---|---|
| `constantConcurrentUsers(n).during(d)` | `d` süresince sabit `n` concurrent VU |
| `rampConcurrentUsers(from).to(to).during(d)` | Concurrent VU sayısı lineer ramp |
| `incrementConcurrentUsers(...)` | Step-wise concurrent VU artışı |

### Birden fazla scenario aynı simulation'da

```java
setUp(
    browseScenario.injectOpen(rampUsers(70).during(60)),
    checkoutScenario.injectOpen(rampUsers(30).during(60)),
    apiScenario.injectClosed(constantConcurrentUsers(20).during(120))
)
.protocols(httpProtocol)
.maxDuration(Duration.ofMinutes(10));         // toplam max süre
```

### Profile chaining — kompleks senaryo
```java
scn.injectOpen(
    // Warm-up
    rampUsers(10).during(30),
    nothingFor(10),
    // Plateau
    constantUsersPerSec(50).during(180),
    // Spike
    atOnceUsers(200),
    nothingFor(30),
    // Ramp-down
    rampUsersPerSec(50).to(0).during(60)
)
```

---

## 19. Throttling — Rate Limiting

Injection'dan **bağımsız**, request seviyesinde RPS sınırı koyar. Senaryolar olabildiğince çok request atmaya çalışır, throttle limit'i aşmazsa.

```java
setUp(
    scn.injectOpen(constantUsersPerSec(50).during(60))
)
.throttle(
    reachRps(100).in(30),         // 30 sn'de 100 RPS'ye çık
    holdFor(Duration.ofMinutes(2)), // 2 dk 100 RPS sabit
    jumpToRps(50),                  // anında 50 RPS'ye düş
    holdFor(60)                     // 60 sn 50 RPS
)
.protocols(httpProtocol);
```

| Throttle step | Açıklama |
|---|---|
| `reachRps(n).in(d)` | `d` süresinde `n` RPS'ye ramp |
| `holdFor(d)` | Mevcut RPS'yi `d` süresince koru |
| `jumpToRps(n)` | Anında `n` RPS |

**Injection vs Throttling karışıklığı:**
- **Injection** → kaç VU enjekte ediliyor
- **Throttling** → o VU'lar toplamda kaç RPS atabilir

Aynı anda kullanılabilir: 100 VU enjekte et + max 50 RPS at.

---

## 20. Assertions — Global SLA

Test sonunda **tüm metrikler üzerinde doğrulama**. Fail olursa CI build kırılır.

```java
setUp(scn.injectOpen(rampUsers(100).during(60)))
    .protocols(httpProtocol)
    .assertions(
        // Global response time
        global().responseTime().mean().lt(300),
        global().responseTime().percentile1().lt(200),     // p50 < 200ms
        global().responseTime().percentile2().lt(400),     // p75 < 400ms
        global().responseTime().percentile3().lt(500),     // p95 < 500ms
        global().responseTime().percentile4().lt(1000),    // p99 < 1000ms
        global().responseTime().max().lt(2000),
        global().responseTime().min().gt(0),

        // Success rate
        global().successfulRequests().percent().gt(99.5),
        global().failedRequests().count().is(0L),
        global().failedRequests().percent().lt(0.5),

        // Throughput
        global().requestsPerSec().gt(100.0),
        global().allRequests().count().gt(10000L),

        // Specific request grouping
        details("Login flow").responseTime().percentile3().lt(800),
        details("Search").failedRequests().percent().lt(1.0),

        // Specific request
        details("API call").responseTime().mean().lt(150),

        // forAll — her request'e ayrı ayrı uygulanır
        forAll().responseTime().percentile4().lt(1500),
        forAll().failedRequests().percent().lt(2.0)
    );
```

### Percentile mapping (ÇOK ÖNEMLİ)
Java DSL'de percentile numarası method adına gömülüdür:

| Method | Percentile | Yorum |
|---|---|---|
| `percentile1()` | **p50 (median)** | Kullanıcıların yarısı buradan hızlı |
| `percentile2()` | **p75** | %75'i buradan hızlı |
| `percentile3()` | **p95** | %95'i buradan hızlı (kötü deneyimi yakalar) |
| `percentile4()` | **p99** | Tail latency (sadece %1 daha yavaş) |

**Custom percentile** istersen `gatling.conf` üzerinden değiştirilebilir:
```hocon
gatling {
  charting {
    indicators {
      percentile1 = 50
      percentile2 = 75
      percentile3 = 95
      percentile4 = 99
    }
  }
}
```

### Assertion scope'ları
| Scope | Açıklama |
|---|---|
| `global()` | Tüm request'ler toplamı |
| `details("name")` | Belirli bir isimli request veya group |
| `forAll()` | Her request'e ayrı ayrı uygulanır |

### Comparison operators
```java
.is(value)           // eşit
.in(min, max)        // aralıkta
.lt(value)           // less than
.lte(value)          // less than or equal
.gt(value)           // greater than
.gte(value)          // greater than or equal
.between(min, max)
.between(min, max, true, false)  // inclusive/exclusive
```

---

## 21. Test Türleri — Load, Stress, Spike, Soak

QA Lead seviyesinde **doğru test türünü seçmek** önemli. Mülakatta her birinin tanımını ve injection profile'ını bilmek beklenir.

### Smoke Test
**Amaç:** Sistem ayakta mı? Senaryolar koşuyor mu?
**Konfig:** 1-5 VU, 1-2 dakika.
**Ne zaman:** Her PR'da, deployment sonrası sanity.

```java
public class SmokeSimulation extends Simulation {
    {
        setUp(scn.injectOpen(atOnceUsers(1)))
            .protocols(httpProtocol)
            .assertions(global().failedRequests().count().is(0L));
    }
}
```

### Load Test
**Amaç:** **Beklenen** yük altında performans SLA'yı karşılıyor mu?
**Konfig:** Production traffic seviyesi, 15-30 dakika sabit.
**Ne zaman:** Nightly, release öncesi.

```java
public class LoadSimulation extends Simulation {
    {
        setUp(
            scn.injectOpen(
                rampUsers(500).during(60),                  // ramp-up
                constantUsersPerSec(50).during(900)         // 15 dk plateau
            )
        )
        .protocols(httpProtocol)
        .assertions(
            global().responseTime().percentile3().lt(500),
            global().successfulRequests().percent().gt(99.5)
        );
    }
}
```

### Stress Test
**Amaç:** Sistemin **sınırını bul** — nerede kırılır?
**Konfig:** Yük SLA aşılana kadar artar, gözleme dayalı.
**Ne zaman:** Capacity planning, release öncesi.

```java
public class StressSimulation extends Simulation {
    {
        setUp(
            scn.injectOpen(
                incrementUsersPerSec(10)
                    .times(20)
                    .eachLevelLasting(60)               // her dakika 10 RPS artar
                    .startingFrom(10)
            )
        )
        .protocols(httpProtocol)
        .maxDuration(Duration.ofMinutes(30))
        .assertions(
            forAll().responseTime().percentile3().lt(2000)   // p95 > 2s ise breaking point
        );
    }
}
```

### Spike Test
**Amaç:** **Ani trafik artışına** karşı sistemin tepkisi.
**Konfig:** Düşük baseline → ani %500 peak → düşük.
**Ne zaman:** Marketing campaign, flash sale öncesi.

```java
public class SpikeSimulation extends Simulation {
    {
        setUp(
            scn.injectOpen(
                constantUsersPerSec(10).during(60),     // normal trafik
                atOnceUsers(500),                        // SPIKE
                constantUsersPerSec(10).during(60)      // recovery
            )
        )
        .protocols(httpProtocol);
    }
}
```

### Soak Test (Endurance Test)
**Amaç:** Uzun süreli yük altında **memory leak, resource leak, GC pressure** tespiti.
**Konfig:** Normal yük, 4-24 saat.
**Ne zaman:** Weekly, major release öncesi.

```java
public class SoakSimulation extends Simulation {
    {
        setUp(
            scn.injectOpen(
                rampUsersPerSec(0).to(30).during(300),     // 5 dk warm-up
                constantUsersPerSec(30).during(Duration.ofHours(8))
            )
        )
        .protocols(httpProtocol)
        .assertions(
            global().responseTime().percentile3().lt(800)
        );
    }
}
```

### Capacity Test
**Amaç:** **Hangi VU sayısı SLA'yı bozmadan kaldırılabilir?**
**Konfig:** Adımlı ramp + her adımda SLA assertion.

```java
public class CapacitySimulation extends Simulation {
    {
        setUp(
            scn.injectClosed(
                incrementConcurrentUsers(50)
                    .times(10)
                    .eachLevelLasting(120)
                    .startingFrom(50)                    // 50 → 100 → 150 → ... → 500
            )
        )
        .protocols(httpProtocol)
        .assertions(
            forAll().responseTime().percentile3().lt(500)
        );
    }
}
```

### Scalability Test
**Amaç:** **Horizontal scaling ROI** — 2 pod ile 4 pod arasında throughput nasıl değişiyor?
**Konfig:** Aynı yük profile'ı, farklı pod/instance sayılarıyla koş; sonuçları karşılaştır.

### Breakpoint Test
**Amaç:** Stress test'in pure formu — sistem **çökene** kadar yük.
**Konfig:** Sınırsız ramp.

```java
setUp(scn.injectOpen(rampUsersPerSec(1).to(10000).during(Duration.ofHours(1))))
```

### Karşılaştırma tablosu
| Test | Yük | Süre | Hedef |
|---|---|---|---|
| Smoke | Çok düşük | 1-2 dk | Sistem ayakta mı |
| Load | Production seviyesi | 15-30 dk | SLA karşılanıyor mu |
| Stress | Artan | Değişken | Sınırı bul |
| Spike | Ani peak | 5-10 dk | Recovery |
| Soak | Normal | 4-24 saat | Leak, drift |
| Capacity | Adımlı | 30 dk - 1 saat | Max güvenli VU |
| Breakpoint | Sonsuz ramp | Çökene kadar | Failure mode |

---

## 22. Recorder — HTTP Proxy / HAR Import

**Gatling Recorder**, mevcut bir browser/uygulama trafiğini yakalayıp Java/Scala/Kotlin senaryosuna otomatik dönüştürür.

```bash
# Standalone bundle
bin/recorder.sh

# Maven
mvn gatling:recorder
```

### İki mod

**1. HTTP Proxy mode**
- Recorder localhost'ta bir HTTP proxy başlatır (default 8000)
- Browser'ın proxy ayarını Recorder'a yönlendir
- Test akışını manuel browser'da gerçekleştir
- Recorder tüm request'leri yakalar → kod üretir

**2. HAR import mode**
- Chrome DevTools (Network tab) → "Save all as HAR with content"
- HAR dosyasını Recorder'a yükle
- Daha güvenilir (proxy SSL sorunları yok)

### QA Lead bakışı — Recorder trap'leri

Recorder hızlı prototip için iyidir **ama production senaryolarda re-write gerekir**:
- Recorder **her şeyi yakalar**: gereksiz analytics call'ları, static asset'ler, third-party widget request'leri
- Production senaryosunda sadece **kritik flow** kalmalı
- Hardcoded value'lar (user-specific token, timestamp) **parametrize** edilmeli (feeder)
- Think time **gerçek** değerlerle değiştirilmeli (recorder olduğu gibi yakalar)

**Best practice:** Recorder ile başla → ham kod elde et → manuel refactor:
1. Gereksiz request'leri sil
2. Hardcoded data'yı feeder'lara çevir
3. Check'leri (status, jsonPath) ekle
4. Group'lara böl (raporlama için)

---

## 23. Diğer Protokoller

### WebSocket
```java
ScenarioBuilder wsScn = scenario("WebSocket")
    .exec(ws("Connect").connect("/ws"))
    .exec(ws("Send hello").sendText("hello server"))
    .exec(ws("Wait response")
        .await(Duration.ofSeconds(5))
        .on(ws.checkTextMessage("welcome").check(regex("welcome.*"))))
    .exec(ws("Close").close());
```

### Server-Sent Events (SSE)
```java
.exec(sse("Subscribe").connect("/events"))
.exec(sse("Wait event")
    .await(Duration.ofSeconds(10))
    .on(sse.checkMessage("data").check(...)))
.exec(sse("Close").close())
```

### JMS
```java
JmsProtocolBuilder jmsProtocol = jms
    .connectionFactoryName("ConnectionFactory")
    .listenerCount(1);

scenario("JMS")
    .exec(jms("Send to queue")
        .send()
        .queue("orders")
        .textMessage("Order data"))
    .exec(jms("Request reply")
        .requestReply()
        .queue("orders")
        .replyQueue("responses")
        .textMessage("Order"))
```

### MQTT (community plugin)
```xml
<dependency>
    <groupId>com.github.ouertani</groupId>
    <artifactId>gatling-mqtt-protocol</artifactId>
</dependency>
```

### gRPC (community plugin)
```xml
<dependency>
    <groupId>com.github.phisgr</groupId>
    <artifactId>gatling-grpc</artifactId>
</dependency>
```

---

## 24. Reporting — HTML Rapor

Test bittiğinde otomatik olarak `target/gatling/<simulation>-<timestamp>/index.html` üretilir.

### Rapor içeriği

**1. Global Information panel**
- Toplam request sayısı
- Toplam süre
- Ortalama RT, min, max
- Standard deviation
- Success/fail count

**2. Indicators (her assertion için)**
- p50, p75, p95, p99 percentile'lar
- Yeşil/sarı/kırmızı renkleme

**3. Response time over time**
- p50, p75, p95, p99'un zaman içindeki seyri
- Yük artışıyla RT'nin nasıl davrandığını gösterir

**4. Active users over time**
- Anlık aktif VU sayısı (concurrent)
- Open injection için kuyruk birikimi burada görünür

**5. Requests per second**
- Saniyedeki gerçekleşen request sayısı

**6. Responses per second**
- Saniyedeki cevaplanan request sayısı
- RPS - Response/sec farkı backlog'u gösterir

**7. Per request breakdown**
- Her named request için ayrı: percentile, count, RT histogram
- Group'lar da burada

**8. Error log**
- Hangi request hangi sebepten fail olmuş

### Custom config — `gatling.conf`
`src/test/resources/gatling.conf`:
```hocon
gatling {
  charting {
    indicators {
      lowerBound = 800
      higherBound = 1200
      percentile1 = 50
      percentile2 = 75
      percentile3 = 95
      percentile4 = 99
    }
  }
  data {
    writers = [console, file]
    console {
      light = false
      writePeriod = 5
    }
  }
}
```

### Rapor saklamayı genişletme
Default'ta her run yeni klasör. Eski raporlar `target/gatling/` altında birikir.

CI/CD'de artifact olarak upload etmek için:
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: gatling-report
    path: target/gatling/**/index.html
```

---

## 25. Gatling Enterprise

**Open Source Gatling sınırları:**
- Tek node'dan yük (max ~10K VU)
- Rapor sadece run sonu HTML, **trend / history yok**
- Multi-node injection yok
- Real-time dashboard yok

**Gatling Enterprise (eski FrontLine):**
- **Distributed load injection** — onlarca node'dan koordineli
- **Real-time live dashboard**
- **Test history & trend analysis** — sürümler arası karşılaştırma
- **Multi-region** geographic distribution
- **Collaboration** — team workspace
- **CI/CD plugins** — Maven, Gradle, Jenkins, GitHub Actions, GitLab CI
- **SAML / SSO** — enterprise auth
- **Cloud (SaaS)** veya **self-hosted**

**Lisans:** Ücretli (Cloud subscription veya enterprise license).

### CI/CD integration örneği — Maven plugin
```xml
<plugin>
    <groupId>io.gatling.enterprise</groupId>
    <artifactId>gatling-enterprise-maven-plugin</artifactId>
    <version>1.x</version>
    <configuration>
        <apiToken>${env.GATLING_ENTERPRISE_API_TOKEN}</apiToken>
        <simulationId>SIMULATION_UUID_FROM_DASHBOARD</simulationId>
    </configuration>
</plugin>
```

```bash
mvn gatling-enterprise:start
```

### Lead seviyesinde Enterprise gerekçesi
- Tek node 5K VU üzerine çıkmak istenirse
- Geographic distribution (Avrupa + ABD + APAC eş zamanlı load)
- Performance trend dashboard (her gece koşan test'in zamanla nasıl davrandığı)
- Multi-team collaboration

---

## 26. CI/CD Entegrasyonu

### GitHub Actions
```yaml
name: Load Test
on:
  schedule:
    - cron: "0 2 * * *"     # her gece 02:00 UTC
  workflow_dispatch:
    inputs:
      simulation:
        description: "Simulation class"
        required: true
        default: "simulations.LoadSimulation"

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Run Gatling
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          USERS: 500
          DURATION: 600
        run: |
          mvn gatling:test \
            -Dgatling.simulationClass=${{ github.event.inputs.simulation || 'simulations.LoadSimulation' }}

      - name: Upload Gatling report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: gatling-report
          path: target/gatling/**

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            { "text": "🔥 Load test SLA breach: ${{ github.run_id }}" }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Jenkins (Declarative)
```groovy
pipeline {
    agent any
    triggers { cron("H 2 * * *") }
    environment {
        BASE_URL = credentials('staging-url')
    }
    stages {
        stage('Load Test') {
            steps {
                sh 'mvn gatling:test -Dgatling.simulationClass=simulations.LoadSimulation'
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'target/gatling/**/index.html', allowEmptyArchive: true
            publishHTML target: [
                reportDir: 'target/gatling',
                reportFiles: 'index.html',
                reportName: 'Gatling Report',
                keepAll: true
            ]
        }
        failure {
            slackSend(color: 'danger', message: "Load test failed: ${env.BUILD_URL}")
        }
    }
}
```

### GitLab CI
```yaml
load-test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn gatling:test -Dgatling.simulationClass=simulations.LoadSimulation
  artifacts:
    when: always
    paths:
      - target/gatling/
    expire_in: 30 days
  only:
    - schedules
```

### Environment-based config (Java)
```java
public class Env {
    public static String baseUrl() {
        return System.getProperty("baseUrl", "http://localhost:8080");
    }
    public static int users() {
        return Integer.parseInt(System.getProperty("users", "100"));
    }
    public static int duration() {
        return Integer.parseInt(System.getProperty("duration", "60"));
    }
}

public class LoadSimulation extends Simulation {
    HttpProtocolBuilder httpProtocol = http.baseUrl(Env.baseUrl());

    {
        setUp(scn.injectOpen(
            rampUsers(Env.users()).during(Env.duration())
        )).protocols(httpProtocol);
    }
}
```

```bash
mvn gatling:test -DbaseUrl=https://staging.example.com -Dusers=500 -Dduration=300
```

---

## 27. Performans Metrikleri Derin Bakış

### Mean (Ortalama) vs Percentile — Lead için kritik
**Mean yanıltıcıdır.** Birkaç outlier'ı dahil edersen ortalama şişer ama p99 farkı yakalar.

**Örnek:** 100 request, 99'u 100ms, 1'i 10000ms.
- Mean = (99×100 + 10000) / 100 = **199ms** → "iyi gibi"
- p50 = **100ms** → "iyi"
- p99 = **10000ms** → "kötü deneyim var"

**Kural:** SLA'yı `mean` ile değil **p95 / p99 / p99.9** ile tanımla.

### Percentile yorumlama
- **p50 (median):** Kullanıcıların yarısı bu süreden hızlı yanıt aldı
- **p95:** Kullanıcıların %95'i bu süreden hızlı (= %5 daha yavaş deneyim yaşadı)
- **p99:** %1 daha yavaş — tail latency
- **p99.9:** Critical sistemler için (örn. ödeme); 1000 request'te 1 outlier

### Little's Law
**L = λ × W**
- L = ortalama eşzamanlı user sayısı (concurrent)
- λ = arrival rate (request/sn)
- W = ortalama response time (sn)

**Pratik kullanım:** "Sistemim p95 = 500ms ile çalışıyor ve 50 RPS karşılıyor. Concurrent kaç user?"
→ L = 50 × 0.5 = **25 concurrent user**

Bu, **closed workload'da kaç VU enjekte etmen gerektiğini** hesaplamana yardım eder.

### Throughput vs Latency trade-off
- Düşük yük → düşük RT, düşük throughput
- Yük artar → RT genelde sabit kalır, throughput artar
- **"Knee" noktası**: RT exponential artmaya başlar, throughput plateu yapar veya düşer
- Bu noktanın bulunması **stress test'in amacı**

### Error rate ve cascade failure
- Error %0 → ideal
- Error < %0.1 → genel acceptable
- Error > %1 → user-facing impact
- Error > %5 → kritik incident

**Cascade failure:** Error'lar arttıkça user retry eder → yük artar → error daha artar → spiral. Lead test'in bunu bulması beklenir.

### Apdex Score (Application Performance Index)
Bir sistemin user satisfaction skorunu ölçer (0-1 arası):
- T = SLA threshold (örn. 500ms)
- Satisfied = RT ≤ T
- Tolerating = T < RT ≤ 4T
- Frustrated = RT > 4T

**Apdex = (Satisfied + Tolerating/2) / Total**

Gatling doğrudan Apdex sunmaz ama percentile'lardan hesaplayabilirsin.

---

## 28. Sık Karşılaşılan Pitfall'lar

### 1. Test makinesi bottleneck olur
Test atan makinenin kendisi yetersiz olabilir. **Network out, CPU, file descriptor limit (`ulimit -n`)** izlenmeli.
**Çözüm:** Gatling Enterprise multi-node, AWS EC2 cluster, k8s job pod.

### 2. Think time eksikliği
`pause` yoksa VU'lar arka arkaya request atar → **gerçekçi olmayan yük**.
```java
.pause(1, 5)   // her exec arası 1-5 sn random think
```

### 3. JIT / cache warmup effect
İlk request'lerde JVM JIT henüz optimize etmemiş, DB cache boş → daha yavaş.
**Çözüm:**
```java
scn.injectOpen(
    nothingFor(30),                       // warm-up phase
    rampUsers(10).during(60),             // gerçek test
    ...
)
```
Veya ayrı warm-up scenario.

### 4. Production database / data leak
Stress test production DB'sini mahvedebilir.
**Mutlaka:**
- Staging environment (production-mirror)
- Synthetic data
- Read-only test'leri ayır

### 5. `inferHtmlResources` ve metric karışıklığı
Browser-like load test backend API metric'ini bulanıklaştırır.
**Çözüm:** Saf API testi → `silentResources()` veya kapat.

### 6. Assertion'ları unutmak
Assertion yoksa CI build fail olmaz → silent regression.
**Her simulation'da en az 1-2 SLA assertion** olmalı.

### 7. Open vs closed model karıştırması
Web sitesi için closed model kullanmak yanlış sonuç.
**Kural:** Sistem davranışıyla model eşleşmeli.

### 8. Static asset'leri test etmek
CDN'den gelen image'leri test'e dahil etmek backend amacı saptırır.
**Çözüm:** `silentResources()` veya inferHtmlResources kapalı.

### 9. Real user latency yok
Gatling makineden HTTP request atar; gerçek mobile network latency, packet loss, jitter yok.
**Çözüm:** Distributed load injection + farklı bölgeler (Gatling Enterprise).

### 10. Stat'ları yanlış okumak
- Mean yanıltıcı; **percentile kullan**
- Max outlier olabilir; **p99.9 daha bilgilendirici**
- Response time histogramı **her zaman incele**

### 11. Recorder çıktısını olduğu gibi kullanmak
Recorder gereksiz request'leri de yakalar. **Manuel refactor şart.**

### 12. Concurrent connection limit
JVM default'ta host başına ~20 connection. Production benzeri yük için:
```java
http.maxConnectionsPerHost(100)
```

### 13. DNS caching
JVM DNS cache TTL default = infinity. Long-running test'lerde DNS değişikliği yakalanamaz.
```java
// JVM arg: -Dsun.net.inetaddr.ttl=30
```

### 14. Cookie isolation arası test
Default'ta cookie store **VU başına** izoledir. Eğer VU'lar arası cookie paylaşılıyor sanırsan yanlış olur.

### 15. Session memory leak
Loop içinde `session.set(...)` ile sürekli yeni key eklersen session büyür → memory.
**Çözüm:** Gerekmeyen değerleri `session.remove(...)` ile temizle.

---

## 29. Framework Tasarımı

### Proje yapısı (kapsamlı)
```
load-tests/
├── pom.xml
├── src/test/java/
│   ├── simulations/
│   │   ├── smoke/
│   │   │   └── SmokeSimulation.java
│   │   ├── load/
│   │   │   └── LoadSimulation.java
│   │   ├── stress/
│   │   │   └── StressSimulation.java
│   │   └── soak/
│   │       └── SoakSimulation.java
│   ├── scenarios/                          # reusable ChainBuilder'lar
│   │   ├── LoginScenarios.java
│   │   ├── CheckoutScenarios.java
│   │   └── BrowseScenarios.java
│   ├── config/
│   │   ├── HttpConfig.java                 # protocol builder factory
│   │   ├── Env.java                        # system properties
│   │   └── Feeders.java                    # feeder factory
│   ├── utils/
│   │   ├── DataGenerator.java
│   │   └── Assertions.java                 # custom SLA assertion sets
│   └── consts/
│       └── Endpoints.java
├── src/test/resources/
│   ├── data/                               # CSV/JSON feeder data
│   │   ├── users.csv
│   │   └── products.json
│   ├── bodies/                             # request body template'ler
│   │   ├── createUser.json
│   │   └── createOrder.json
│   ├── gatling.conf                        # Gatling runtime
│   └── logback-test.xml
└── target/gatling/                         # otomatik rapor
```

### Reusable scenarios
```java
public class LoginScenarios {
    public static ChainBuilder login() {
        return exec(http("Login")
            .post("/login")
            .formParam("email", "#{email}")
            .formParam("password", "#{password}")
            .check(status().is(200))
            .check(jsonPath("$.token").saveAs("authToken"))
        );
    }

    public static ChainBuilder logout() {
        return exec(http("Logout")
            .post("/logout")
            .header("Authorization", "Bearer #{authToken}")
        );
    }
}

// Simulation
ScenarioBuilder scn = scenario("Authenticated browse")
    .feed(Feeders.users())
    .exec(LoginScenarios.login())
    .pause(2)
    .exec(BrowseScenarios.browseProducts())
    .exec(CheckoutScenarios.completeCheckout())
    .exec(LoginScenarios.logout());
```

### Custom assertion helpers
```java
public class Assertions {
    public static Assertion[] readApiSla() {
        return new Assertion[] {
            global().responseTime().percentile3().lt(300),
            global().responseTime().percentile4().lt(500),
            global().successfulRequests().percent().gt(99.5)
        };
    }

    public static Assertion[] writeApiSla() {
        return new Assertion[] {
            global().responseTime().percentile3().lt(800),
            global().responseTime().percentile4().lt(1500),
            global().successfulRequests().percent().gt(99.0)
        };
    }
}

// Simulation
setUp(scn.injectOpen(...))
    .protocols(httpProtocol)
    .assertions(Assertions.readApiSla());
```

### Feeder factory
```java
public class Feeders {
    public static FeederBuilder<String> users() {
        return csv("data/users.csv").random();
    }

    public static FeederBuilder<Object> products() {
        return jsonFile("data/products.json").circular();
    }
}
```

---

## 30. QA Lead Seviyesi — Mimari Kararlar

### Performans test stratejisi
Her release için:
1. **PR-level smoke** — temel SLA, < 5 dk
2. **Nightly load** — staging'de production-equivalent yük, 30 dk
3. **Pre-release stress** — sistemin sınırı, 1-2 saat
4. **Weekly soak** — 8+ saat, leak detection
5. **Release-prep capacity** — production-trafic profili ile, 1 hafta önce

### SLA tanımları (örnek)
| Endpoint tipi | p50 | p95 | p99 | Error rate |
|---|---|---|---|---|
| Read API | < 100ms | < 300ms | < 500ms | < 0.1% |
| Write API | < 200ms | < 600ms | < 1000ms | < 0.5% |
| Search | < 300ms | < 800ms | < 1500ms | < 0.5% |
| Auth | < 150ms | < 400ms | < 700ms | < 0.05% |

Bunları **assertion'a çevir**, CI'da gate olarak kullan.

### Load injection mimarisi
| Senaryo | Önerilen |
|---|---|
| ≤ 5000 VU | Tek node, Open Source Gatling |
| > 5000 VU | Gatling Enterprise (multi-node) veya custom k8s cluster |
| Geographic test | Enterprise + multi-region |
| Spot performance check | Standalone bundle, lokal |

### Observability entegrasyonu
Gatling raporu **sadece client-side**. Server-side için:
- **Grafana + Prometheus** dashboard
- **APM:** Datadog, New Relic, Dynatrace
- **Logs:** ELK / Loki

Test sırasında **side-by-side dashboard** açık olmalı: client RT vs server CPU/memory/DB pool.

### Bottleneck analizi
RT yüksek + error yoksa → bottleneck nerede?
1. **Application CPU?** APM profiling
2. **DB?** Slow query log, connection pool exhaustion
3. **External API?** Downstream dependency monitoring
4. **Network?** Bandwidth saturation
5. **GC?** JVM heap analysis

Lead'in görevi sadece "yavaş" demek değil, **"şu bottleneck yüzünden yavaş"** diye iddia üretmek.

### Test data management
- **Test user pool**, her test koşusunda 1000+ unique user
- **Idempotent operations** — test tekrar koşulabilir kalmalı
- **Cleanup script** — soak/stress test sonrası DB cleanup
- **Data refresh** — her hafta staging DB'yi production snapshot'tan refresh (PII anonimize)

### Team & process
- **Performance test ownership:** Backend team + QA shared
- **Pre-merge gate:** PR build'lerde smoke; merge sonrası nightly
- **Capacity planning quarterly:** Stress test sonuçları → infrastructure budget
- **Postmortem:** Production incident'lerden öğren — incident senaryosunu Gatling'e ekle

---

## 31. En Sık Mülakat Soruları

| Soru | Hızlı Cevap |
|---|---|
| Gatling nedir? | JVM'de çalışan async I/O tabanlı yüksek perf load test framework'ü; Scala/Java/Kotlin DSL |
| JMeter'dan farkı? | Non-blocking I/O (Netty + Akka) → thread-per-user değil actor model; 10x daha fazla VU |
| Ruby DSL? | **Yok.** Sadece Scala, Java, Kotlin. Ruby team load test için ruby-jmeter veya başka tool kullanır |
| Simulation nedir? | Test senaryosunun tamamı, bir class olarak yazılır |
| Scenario nedir? | Bir VU'nun yapacağı action zinciri |
| Virtual User? | Test'te simüle edilen bağımsız bir actor; OS thread değil |
| Session? | VU'nun kendi key-value state'i |
| Feeder? | Test data iterator (CSV/JSON/JDBC/Redis/custom) |
| Feeder strategies? | queue (FIFO), random, shuffle, circular |
| Check? | Response assertion + data extraction (saveAs ile session'a yazma) |
| Open vs closed workload? | Open: arrival rate (real traffic); Closed: sabit concurrent VU (sabit pool) |
| Open injection method'ları? | atOnceUsers, rampUsers, constantUsersPerSec, rampUsersPerSec, stressPeakUsers, heavisideUsers |
| Closed injection method'ları? | constantConcurrentUsers, rampConcurrentUsers, incrementConcurrentUsers |
| Throttling vs injection? | Injection: kaç VU; Throttling: max RPS |
| Think time? | Gerçek user davranışı simülasyonu; yoksa gerçekçi olmayan yük |
| pause vs pace? | pause: request sonrası bekle; pace: sabit interval (RPS lock) |
| Pause type'lar? | Constant, Uniform, Exponential, Normal, Disabled, Custom |
| Assertion amacı? | Test sonu SLA kontrolü → CI build pass/fail |
| Percentile method'ları? | percentile1=p50, percentile2=p75, percentile3=p95, percentile4=p99 |
| Mean vs percentile? | Mean yanıltıcı; outlier'lar şişirir; **p95/p99** kullan |
| Smoke vs Load vs Stress vs Spike vs Soak? | Smoke: sanity; Load: normal yük; Stress: sınır; Spike: ani peak; Soak: uzun süre leak |
| Capacity test? | Adımlı ramp + assertion → max güvenli VU |
| Breakpoint test? | Sistem çökene kadar yük; pure stress |
| Static asset'ler? | `silentResources()` veya inferHtmlResources kapalı |
| Recorder ne işe yarar? | Browser proxy / HAR → senaryo prototype; production için manuel re-write gerekir |
| Multi-node injection? | Gatling Enterprise (paid) veya custom cloud cluster |
| CI/CD entegrasyon? | Maven/Gradle plugin + assertion gate; artifact olarak HTML rapor upload |
| Reporting? | Built-in HTML rapor (target/gatling/); Enterprise real-time + history |
| Gatling Enterprise gerekçesi? | > 5K VU, multi-region, history/trend, team collaboration |
| Bottleneck analizi? | Client RT + server APM/metric karşılaştırması; CPU/DB/network/GC ayırt et |
| Production'da test? | Mümkünse staging-mirror; production'da sadece read-only smoke |
| Little's Law? | L = λ × W (concurrent = arrival rate × RT); kapasite hesabı için |
| Apdex score? | T threshold ile satisfied/tolerating/frustrated user oranı |
| ChainBuilder ne işe yarar? | Reusable action chain; scenario'lara include edilir |
| Group ne işe yarar? | Raporlama wrapper; birden fazla request'i tek blok gösterir |
| EL nedir? | Gatling Expression Language — `#{var}` syntax'ı ile session erişimi |
| Java DSL Scala'dan ne zaman geldi? | Gatling 3.7 (2021) |
| Cascade failure? | Error → retry → daha çok yük → daha çok error spiral |

---

## 32. Çalışma Yol Haritası

### 1. Hafta — Konseptler ve temel DSL
- [ ] Gatling kur (Maven), ilk simulation koş
- [ ] **Simulation**, **scenario**, **VU**, **session** kavramlarını içselleştir
- [ ] `http` protocol builder
- [ ] `exec(http(...).get/post)` ile temel request'ler
- [ ] HTML raporu incele, **percentile'ları öğren** (p50/95/99 farkı)

### 2. Hafta — Orta seviye
- [ ] **Feeders** (CSV, random/queue/circular strategies)
- [ ] **Checks** ile data extraction + saveAs
- [ ] **Gatling EL** — `#{variable}` syntax, built-in functions
- [ ] **Session manipulation** — set, get, transform
- [ ] **Conditional logic** — doIf, doSwitch, randomSwitch
- [ ] **Loops** — repeat, during, foreach, asLongAs
- [ ] **Pause / pace** ile think time

### 3. Hafta — Injection ve Assertion
- [ ] **Open vs closed workload modelini** kavra; iki örnekle pratik
- [ ] Open injection method'larını tek tek dene: rampUsers, constantUsersPerSec, stressPeakUsers
- [ ] Closed injection: constantConcurrentUsers, rampConcurrentUsers
- [ ] **Throttling** ile RPS lock
- [ ] **Assertion** SLA gate'leri yaz
- [ ] **Smoke + load + stress + soak** senaryolarını yaz

### 4. Hafta — Framework ve Lead seviyesi
- [ ] **ChainBuilder** ile scenario reuse
- [ ] **Env-based config** (staging/prod/local — system property)
- [ ] Custom assertion helper class
- [ ] **CI/CD pipeline** (GitHub Actions veya Jenkins) + artifact upload
- [ ] **Server-side observability** ile korelasyon (Grafana/APM)
- [ ] **Bottleneck analizi** practice — APM dashboard ile birlikte koş
- [ ] **Performance test stratejisi** dokümante et (SLA tablosu, schedule, ownership)
- [ ] (Opsiyonel) Gatling Enterprise trial — distributed test deneyimi

### Bonus — derin konular
- [ ] **Little's Law** ile kapasite hesabı
- [ ] **Apdex score** hesaplama
- [ ] **Cascade failure** simülasyonu
- [ ] **Recorder** ile HAR import ve refactor
- [ ] **WebSocket / SSE** scenario
- [ ] Custom EL function

---

## 33. Kaynaklar

### Resmi
- **Ana sayfa:** gatling.io
- **Docs (Java DSL):** docs.gatling.io/reference/script/core/scenario/
- **GitHub:** github.com/gatling/gatling
- **Quickstart Java:** github.com/gatling/gatling-maven-plugin-demo-java

### Java DSL özel
- **Java DSL migration guide:** docs.gatling.io/tutorials/upgrading/3.7/
- **API reference:** gatling.io/docs/3.11/reference/script/

### Kavramsal kaynaklar (Lead seviyesi)
- **Brendan Gregg — "Systems Performance"** (kitap) — bottleneck analizi, USE method
- **"Site Reliability Engineering"** (Google) — SLO/SLI/SLA tanımları
- **"Designing Data-Intensive Applications"** (Kleppmann) — capacity planning
- **Wikipedia — Little's Law** — kapasite hesabı
- **Wikipedia — Apdex** — user satisfaction metriği

### Topluluk
- **community.gatling.io** — forum
- **gatling.io/blog** — case study'ler, best practice'ler
- **Stack Overflow tag:** `gatling`

### Karşılaştırma için
- **k6.io** — JavaScript alternatifi
- **locust.io** — Python alternatifi
- **jmeter.apache.org** — Java XML klasiği
- **tsenart/vegeta** — CLI alternatifi (Go)
