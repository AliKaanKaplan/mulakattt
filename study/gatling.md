# Gatling — QA Lead Mülakat Hazırlık Rehberi

> Yüksek-performanslı load / stress testing framework'ü. Async I/O mimarisi sayesinde JMeter'ın aksine tek bir makineden binlerce eşzamanlı kullanıcıyı simüle eder.

---

## 1. Gatling Nedir?

**Gatling**, açık kaynak, **Scala / Java / Kotlin DSL** ile yazılan, **yüksek performanslı load testing** aracıdır.

**Temel özellikleri:**
- **Async, non-blocking I/O** (Netty + Akka tabanlı) → tek makinede 10K+ eşzamanlı virtual user
- **DSL tabanlı senaryo tanımı** — code as configuration
- **Detaylı HTML raporlar** — built-in, çok zengin
- **HTTP, JMS, MQTT, JDBC, gRPC** protokolleri (HTTP en yaygın)
- **CI/CD-friendly** — Maven, Gradle, sbt plugin'leri
- **Gatling Enterprise** (eski FrontLine) → dağıtık load injection, real-time dashboard

**Versiyonlar:**
- **Gatling 2.x:** Scala-only, eski
- **Gatling 3.x:** Java + Kotlin DSL eklendi (3.7+), Scala 3
- **Gatling Open Source vs Enterprise:** OS tek-node, Enterprise multi-node + collaboration

**Karşılaştırma:**
| Özellik | Gatling | JMeter | k6 |
|---|---|---|---|
| Architecture | Async (Netty) | Thread-per-user | Async (Go) |
| Single-node capacity | 10K+ VU | ~1K VU | 10K+ VU |
| DSL dili | Scala/Java/Kotlin | XML + GUI | JavaScript |
| Reporting | HTML built-in, zengin | Basic + plugin | Cloud dashboard |
| Learning curve | Orta | Düşük | Düşük |
| Code review | ✅ Kod, kolay | ⚠️ XML, zor | ✅ Kod, kolay |

---

## 2. Async I/O Mimarisi — Neden Önemli?

**JMeter** her virtual user için bir **OS thread** açar. 5000 VU = 5000 thread → context switching maliyeti, memory tükenir.

**Gatling** virtual user'ları **Akka actor'ları** veya event-driven coroutine'ler olarak yönetir. Bir VU bir HTTP request gönderdiğinde **thread'i bloke etmez** — response gelene kadar başka VU'lar aynı thread'i kullanabilir.

Sonuç: **tek bir 4-core makineden 20K+ VU** rahatlıkla simüle edilir.

**Lead seviyesinde mülakat sorusu:** "Neden Gatling JMeter'dan daha az kaynak tüketir?" → **Non-blocking I/O + actor model**.

---

## 3. Kurulum

### Standalone bundle
```bash
# gatling.io/open-source/ → ZIP indir
# bin/gatling.sh    veya  bin\gatling.bat
```

### Maven (Java DSL)
```xml
<dependency>
  <groupId>io.gatling.highcharts</groupId>
  <artifactId>gatling-charts-highcharts</artifactId>
  <version>3.10.5</version>
  <scope>test</scope>
</dependency>

<plugin>
  <groupId>io.gatling</groupId>
  <artifactId>gatling-maven-plugin</artifactId>
  <version>4.10.1</version>
</plugin>
```

```bash
mvn gatling:test
mvn gatling:test -Dgatling.simulationClass=simulations.LoadSimulation
```

### Gradle (Kotlin DSL)
```kotlin
plugins {
    id("io.gatling.gradle") version "3.10.5"
}
```

```bash
./gradlew gatlingRun
./gradlew gatlingRun-simulations.LoadSimulation
```

### sbt (Scala DSL — orijinal)
```scala
libraryDependencies += "io.gatling.highcharts" % "gatling-charts-highcharts" % "3.10.5" % Test
```

---

## 4. Temel Kavramlar

### Simulation
Bir test senaryosunun tamamı. Bir Scala/Java/Kotlin class'ıdır.

### Scenario
Bir virtual user'ın yapacağı action zinciri. Bir simulation içinde birden fazla scenario olabilir.

### Virtual User (VU)
Test'te simüle edilen tek bir kullanıcı. Her VU bağımsız bir actor.

### Session
Bir VU'nun kendi key-value state'i. Feeder'dan gelen data, capture edilen response value'lar burada.

### Feeder
Senaryolara test data sağlayan iterator (CSV, JSON, custom).

### Check
Response üzerinde assertion ve/veya data extraction.

### Assertions
Tüm test sonunda global metrik doğrulamaları (örn. p99 latency < 500ms).

### Injection Profile
VU'ların zaman içinde nasıl enjekte edileceği (ramp-up, constant, peak vs).

---

## 5. İlk Simulation (Java DSL)

```java
package simulations;

import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;

public class BasicSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
        .baseUrl("https://api.example.com")
        .acceptHeader("application/json")
        .userAgentHeader("Gatling/Load");

    ScenarioBuilder scn = scenario("Basic Browse")
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

    {
        setUp(
            scn.injectOpen(
                rampUsers(100).during(60),
                constantUsersPerSec(10).during(120)
            )
        ).protocols(httpProtocol)
         .assertions(
            global().responseTime().percentile3().lt(500),
            global().successfulRequests().percent().gt(99.0)
         );
    }
}
```

**Açıklama:**
- `http` protokol konfigürasyonu — base URL, default header'lar
- `scenario(...).exec(...)` → action chain
- `pause(1)` → think time (gerçek kullanıcı davranışı simülasyonu)
- `check(...)` → response validation + data extraction
- `injectOpen(rampUsers(100).during(60))` → 60 saniyede 100 user ekle
- `assertions(...)` → test sonunda global SLA kontrol

---

## 6. HTTP Protocol Konfigürasyonu

```java
HttpProtocolBuilder httpProtocol = http
    .baseUrl("https://api.example.com")
    .baseUrls("https://api1.example.com", "https://api2.example.com")  // round-robin
    .acceptHeader("application/json")
    .acceptEncodingHeader("gzip, deflate")
    .userAgentHeader("Gatling/Load")
    .authorizationHeader("Bearer #{token}")
    .header("X-API-Version", "v2")
    .disableFollowRedirect()
    .disableCaching()
    .shareConnections()                       // connection pool paylaşımı
    .maxConnectionsPerHost(20)
    .connectionHeader("keep-alive")
    .silentResources()                        // CSS/JS/img'leri stat'lara dahil etme
    .inferHtmlResources()                      // HTML'den embedded resource'ları otomatik fetch
    .check(status().not(500), status().not(404))   // global check
    .extraInfoExtractor(ExtraInfoExtractor.... );
```

---

## 7. Scenario DSL

### Sequential actions
```java
scenario("Browse")
  .exec(http("Home").get("/"))
  .exec(http("Login").post("/login").formParam("user", "x").formParam("pass", "y"))
  .exec(http("Profile").get("/profile"))
```

### Pauses (think time)
```java
.pause(2)                                  // 2 saniye sabit
.pause(2, 5)                                // 2-5 saniye uniform random
.pause(Duration.ofSeconds(3))
.pause("#{thinkTime}")                     // session variable'dan
.pace(5)                                    // bir önceki exec'ten 5 saniye sonra (rate limiting)
```

### Conditional logic
```java
.doIf("#{loggedIn}")
  .then(exec(http("Profile").get("/profile")))

.doIfOrElse(session -> session.getInt("amount") > 100)
  .then(exec(http("Premium").get("/premium")))
  .orElse(exec(http("Free").get("/free")))

.doSwitch("#{userType}")
  .on(
    Choice.withKey("admin", exec(http("Admin").get("/admin"))),
    Choice.withKey("user",  exec(http("Home").get("/")))
  )

.randomSwitch().on(
    Choice.withWeight(70.0, exec(http("Search").get("/search"))),
    Choice.withWeight(30.0, exec(http("Browse").get("/browse")))
)
```

### Loops
```java
.repeat(5).on(
    exec(http("Item").get("/item/#{i}"))
)

.during(Duration.ofMinutes(5)).on(
    exec(http("Poll").get("/poll")).pause(2)
)

.forever().on(
    exec(http("Stream").get("/stream"))
)

.foreach("#{productIds}", "productId").on(
    exec(http("Product").get("/products/#{productId}"))
)
```

### Groups (raporlama için)
```java
.group("Login flow").on(
    exec(http("Open login").get("/login"))
    .exec(http("Submit").post("/login").formParam(...))
    .exec(http("Dashboard").get("/dashboard"))
)
```
Group'lar raporda tek bir blok olarak gösterilir.

---

## 8. Checks (Assertions & Data Extraction)

```java
http("Get product")
  .get("/products/123")
  .check(status().is(200))
  .check(status().in(200, 201, 204))
  .check(header("Content-Type").is("application/json"))
  .check(responseTimeInMillis().lt(500))
  .check(jsonPath("$.name").is("iPhone"))
  .check(jsonPath("$.price").ofInt().gt(0))
  .check(jsonPath("$.id").saveAs("productId"))
  .check(jsonPath("$.tags[*]").findAll().saveAs("tags"))
  .check(regex("user_id=([0-9]+)").saveAs("userId"))
  .check(xpath("//title").is("My Shop"))
  .check(css("h1.title", "innerText").saveAs("title"))
  .check(bodyString().saveAs("rawBody"))
  .check(substring("success").exists())
  .check(jmesPath("results[0].id").saveAs("id"))   // AWS JMESPath
```

**Saved variable'ları kullanmak:**
```java
.exec(http("Use saved")
    .get("/users/#{userId}/orders"))
```
`#{userId}` Gatling Expression Language (Gatling EL).

---

## 9. Feeders — Test Data

### CSV
```java
FeederBuilder<String> csvFeeder = csv("users.csv").random();
// circular(), shuffle(), random(), queue() (default, FIFO)

scenario("Login")
  .feed(csvFeeder)
  .exec(http("Login")
      .post("/login")
      .formParam("email", "#{email}")
      .formParam("password", "#{password}"))
```

**users.csv:**
```csv
email,password
user1@x.com,pass1
user2@x.com,pass2
```

### JSON / Custom
```java
FeederBuilder<Object> jsonFeeder = jsonFile("data.json").random();

// Programmatic
Iterator<Map<String, Object>> customFeeder =
    Stream.generate((Supplier<Map<String, Object>>) () ->
        Map.of("randomEmail", "user" + ThreadLocalRandom.current().nextInt() + "@test.com")
    ).iterator();
```

### Feeder strategies
| Strategy | Davranış |
|---|---|
| `queue()` | FIFO; VU > data → "no more data" hatası |
| `random()` | Rastgele seçim, sınırsız |
| `shuffle()` | Tek seferlik karıştır, sonra sıralı |
| `circular()` | Listeyi döndür (loop) |

---

## 10. Injection Profiles — Yük Modelleri

### Open vs Closed Workload Model — Kritik Lead Konusu

**Open workload (`injectOpen`):**
- Arrival rate (saniyede X user gel) ile model
- Yeni user'lar **mevcut user'ların durumundan bağımsız** gelir
- Gerçek dünyaya yakın (örn. e-ticaret traffic)
- Sistem yavaşlarsa queue büyür → backpressure

**Closed workload (`injectClosed`):**
- **Sabit sayıda concurrent user** korunur
- Bir VU bittiği an yeni biri başlar
- Performance test'lerde yaygın (sabit pool)
- Sistem yavaşlarsa throughput düşer ama VU sayısı sabit

**Lead seviyesinde mülakat sorusu:** *"Open vs closed workload modelinin farkı nedir, hangisini ne zaman seçersin?"*
→ Cevap: Open → gerçek user trafiği simülasyonu (peak traffic, marketing campaign). Closed → mevcut user kapasitesini test etme (call center, fixed pool).

### Open injection
```java
scn.injectOpen(
    nothingFor(5),                                  // 5 sn boş
    atOnceUsers(50),                                // anlık 50 VU
    rampUsers(200).during(120),                     // 2 dk'da 200 VU ekle
    constantUsersPerSec(20).during(60),             // 1 dk boyunca sn'de 20 VU
    rampUsersPerSec(10).to(50).during(120),         // 2 dk'da rate 10→50/sn
    heavisideUsers(1000).during(60),                // S-curve ramp
    stressPeakUsers(500).during(30)                 // hızlı peak
)
```

### Closed injection
```java
scn.injectClosed(
    constantConcurrentUsers(100).during(60),
    rampConcurrentUsers(10).to(100).during(120)
)
```

### Birden fazla scenario, ağırlıklı
```java
setUp(
    browseScenario.injectOpen(rampUsers(70).during(60)),
    checkoutScenario.injectOpen(rampUsers(30).during(60))
).protocols(httpProtocol)
 .maxDuration(Duration.ofMinutes(10));
```

---

## 11. Assertions — Global SLA

Test sonunda tüm metrikler üzerinde doğrulama. Bunlar fail olursa **build fail olur** (CI/CD entegrasyonu için kritik).

```java
.assertions(
    global().responseTime().mean().lt(300),
    global().responseTime().percentile3().lt(500),   // p95 < 500ms
    global().responseTime().percentile4().lt(1000),  // p99 < 1000ms
    global().responseTime().max().lt(2000),

    global().successfulRequests().percent().gt(99.5),
    global().failedRequests().count().is(0L),

    global().requestsPerSec().gt(100.0),

    details("Login flow").responseTime().percentile3().lt(800),
    forAll().responseTime().percentile4().lt(1500)    // her request için p99
);
```

**Percentile mapping (önemli):**
| Method | Percentile |
|---|---|
| `percentile1()` | 50th (median) |
| `percentile2()` | 75th |
| `percentile3()` | 95th |
| `percentile4()` | 99th |

---

## 12. Reporting

Test sonunda otomatik HTML raporu `target/gatling/.../index.html` veya `results/.../index.html`'e yazar.

**Rapor içeriği:**
- Global summary (toplam request, ortalama RT, error rate)
- **Response time distribution** (50/75/95/99/max percentile)
- **Requests per second** zaman grafiği
- **Active users** zaman grafiği
- **Response time** zaman grafiği (her percentile)
- **Per request grouping** — detaylı request bazlı breakdown
- **Error log** — hangi request neden fail olmuş

### Gatling Enterprise
- Real-time dashboard
- Multi-node load injection (binlerce node'dan koordineli)
- Test trend analysis (historical comparison)
- Collaborative workspace
- CI/CD plugin'leri
- **Ücretli** (Enterprise lisans)

### CI/CD entegrasyonu
```yaml
# GitHub Actions
- name: Run Gatling
  run: mvn gatling:test -Dgatling.simulationClass=simulations.LoadSimulation

- name: Upload report
  uses: actions/upload-artifact@v4
  with:
    name: gatling-report
    path: target/gatling
```

---

## 13. Gatling Recorder — Test Senaryosu Kaydetme

Browser proxy üzerinden HTTP trafiğini yakalayıp Scala/Java senaryosuna çevirir.

```bash
bin/recorder.sh
```

- **HTTP Proxy mode:** Browser proxy ayarı → Recorder traffic'i yakalar
- **HAR import mode:** Chrome DevTools'tan export edilen HAR dosyasını import et

**Lead bakışı:** Recorder hızlı prototip için iyidir ama **production senaryolarda re-write gerekir** — recorder her şeyi yakalar (gereksiz analytics call'ları, static asset'ler), production senaryosunda sadece kritik flow kalmalı.

---

## 14. Performans Test Türleri — Hangi Senaryoda Hangisi?

QA Lead seviyesinde **doğru test türünü seçmek** önemli:

| Tip | Amaç | Tipik konfig |
|---|---|---|
| **Smoke test** | Sistem cevap veriyor mu? | 1-5 VU, 1-2 dk |
| **Load test** | Normal yük altında performans | Production traffic seviyesi, 15-30 dk |
| **Stress test** | Sınırı bul, sistem nerede kırılır? | Ramp up sonsuza kadar veya kırılana |
| **Spike test** | Ani yük artışı | Anında %500 trafik → kurtarma davranışı |
| **Endurance / Soak test** | Memory leak, resource leak | Normal yük, 4-24 saat |
| **Capacity test** | Hangi VU sayısı SLA'yı bozmadan kaldırılabilir? | Adımlı ramp + assertion |
| **Scalability test** | Horizontal scaling ROI | Aynı yük, farklı pod/instance sayıları |

---

## 15. Gatling DSL — Diğer Protokoller

### WebSocket
```java
exec(ws("Connect").connect("/ws"))
.exec(ws("Send").sendText("hello"))
.exec(ws("Wait").await(5).on(
    ws.checkTextMessage("welcome").check(regex("welcome"))
))
.exec(ws("Close").close())
```

### Server-Sent Events (SSE)
```java
exec(sse("Subscribe").connect("/events"))
.exec(sse("Wait event").await(10).on(sse.checkMessage("data").check(...)))
```

### JMS
```java
JmsProtocolBuilder jmsProtocol = jms
    .connectionFactoryName("ConnectionFactory")
    .listenerCount(1);
```

### MQTT (Gatling 3.x plugin)
### gRPC (community plugin)

---

## 16. Sık Karşılaşılan Pitfall'lar

### 1. Aynı makineden çok yüksek VU
Test makinesinin kendisi bottleneck olabilir. **Network out, CPU, file descriptor limit**'lerini izle.

Çözüm:
- Gatling Enterprise multi-node injection
- AWS EC2 cluster + scenario distribute
- Test makinesinin metric'lerini paralel monitör et

### 2. Think time eksikliği
`pause` yoksa user'lar arka arkaya request atar → **gerçekçi olmayan yük**. Production'da kullanıcılar düşünür.

```java
.pause(1, 5)   // her exec arasında 1-5 sn random think
```

### 3. Cache / connection effect
İlk request'lerde JIT warmup + connection pool warm-up → daha yavaş.

Çözüm: `nothingFor(30)` ile warmup phase'i sonuçlardan çıkar, veya **warm-up scenario** ekle.

### 4. Production database / data leak
Stress test production DB'sini mahvedebilir. Mutlaka:
- **Staging environment** kullan (production-mirror)
- **Synthetic data** üret
- **Read-only test'leri ayır**, write-heavy test'leri dikkatli koş

### 5. Inferred HTML resources kapatma
`inferHtmlResources()` HTML'in içindeki CSS/JS/img'i otomatik fetch eder. Test'in ne ölçtüğüne dikkat — sadece API mi yoksa full page mi?

### 6. Assertion'ları unutmak
Assertion yoksa CI build fail olmaz → silent regression. **Her simulation'da en az 1-2 SLA assertion** olmalı.

### 7. Open vs closed model karıştırması
Closed model ile peak traffic simüle etmeye çalışmak yanlış sonuç verir. Modelin sistemin **gerçek user behavior'ı** ile eşleşmesi gerekir.

### 8. Static asset'leri test etmek
CDN'den gelen image'leri test'e dahil etmek backend load test'inin amacını saptırır. `.silentResources()` ile filtrele veya inferred resource'ları kapat.

### 9. Real user latency yok
Gatling makineden HTTP request atar; **gerçek mobile network latency**, packet loss, jitter yok. CDN/edge'yi gerçekçi test etmek için **distributed load injection** + farklı bölgeler gerekir.

### 10. Stat'ları yanlış okumak
- **Mean** yanıltıcıdır; **percentile** kullan (özellikle p95, p99)
- **Max** outlier olabilir; p99.9 daha bilgilendirici
- **Response time distribution** histogramı her zaman incele

---

## 17. Framework Tasarımı — QA Lead Bakışı

### Proje yapısı (Maven, Java DSL)
```
src/test/java/
  ├── simulations/
  │   ├── smoke/
  │   │   └── SmokeSimulation.java
  │   ├── load/
  │   │   └── LoadSimulation.java
  │   ├── stress/
  │   │   └── StressSimulation.java
  │   └── soak/
  │       └── SoakSimulation.java
  ├── scenarios/
  │   ├── LoginScenarios.java
  │   ├── CheckoutScenarios.java
  │   └── BrowseScenarios.java
  ├── config/
  │   ├── HttpConfig.java          # protocol builder
  │   └── Env.java                 # environment vars
  └── utils/
      └── DataGenerator.java
src/test/resources/
  ├── data/
  │   ├── users.csv
  │   └── products.json
  └── application.conf
```

### Scenario reuse
```java
public class LoginScenarios {
    public static ChainBuilder login() {
        return exec(http("Login")
            .post("/login")
            .formParam("email", "#{email}")
            .formParam("password", "#{password}")
            .check(jsonPath("$.token").saveAs("authToken"))
        );
    }
}

// Simulation'da reuse
ScenarioBuilder scn = scenario("Authenticated browse")
    .feed(csvFeeder)
    .exec(LoginScenarios.login())
    .exec(BrowseScenarios.browseProducts())
    .exec(CheckoutScenarios.completeCheckout());
```

### Environment-based config
```java
public class Env {
    public static String baseUrl() {
        String env = System.getProperty("env", "staging");
        return switch (env) {
            case "prod"    -> "https://api.example.com";
            case "staging" -> "https://api-staging.example.com";
            default        -> "http://localhost:8080";
        };
    }

    public static int users() {
        return Integer.parseInt(System.getProperty("users", "100"));
    }

    public static int duration() {
        return Integer.parseInt(System.getProperty("duration", "60"));
    }
}
```

```bash
mvn gatling:test -Denv=staging -Dusers=500 -Dduration=300
```

---

## 18. QA Lead Seviyesi — Mimari Kararlar

### Performans test stratejisi
Her release için:
1. **PR-level smoke** — temel SLA, < 5 dk
2. **Nightly load** — staging'de production-equivalent yük, 30 dk
3. **Pre-release stress** — sistemin sınırı, 1-2 saat
4. **Weekly soak** — 8+ saat, memory leak detection
5. **Release-prep capacity** — production-trafic profili ile, 1 hafta önce

### SLA tanımları (örnek)
| Endpoint tipi | p50 | p95 | p99 | Error rate |
|---|---|---|---|---|
| Read API | < 100ms | < 300ms | < 500ms | < 0.1% |
| Write API | < 200ms | < 600ms | < 1000ms | < 0.5% |
| Search | < 300ms | < 800ms | < 1500ms | < 0.5% |
| Auth | < 150ms | < 400ms | < 700ms | < 0.05% |

Bunları assertion'a çevir, CI'da gate olarak kullan.

### Load injection mimarisi
- **Tek node:** ≤ 5000 VU, hızlı kurulum
- **Gatling Enterprise multi-node:** > 5K VU veya geographic distribution gerekli
- **k6 + Gatling kombinasyonu:** k6 ile distributed cloud, Gatling ile detaylı complex flow

### Observability entegrasyonu
Gatling'in kendi raporu **sadece client-side metric**. Server-side metric'ler için:
- **Grafana + Prometheus** dashboard
- **APM:** Datadog, New Relic, Dynatrace
- **Logs:** ELK / Loki

Test sırasında **side-by-side dashboard** açık olmalı: client RT vs server CPU/memory/DB pool.

### Bottleneck analizi
RT yüksek + error yoksa → bottleneck nerede?
1. **Application CPU?** APM profiling
2. **DB?** Slow query log, connection pool exhaustion
3. **External API?** Downstream dependency monitoring
4. **Network?** Bandwidth saturation, packet loss
5. **GC?** JVM heap analysis

Lead'in görevi sadece **"yavaş"** demek değil, **"şu bottleneck yüzünden yavaş"** diye iddia üretmek.

### Test data management
- **Test user pool**, her test koşusunda 1000+ unique user
- **Idempotent operations** test'leri tekrar koşulabilir kalmalı
- **Cleanup:** soak/stress test sonrası DB cleanup script
- **Data refresh:** her hafta staging DB'yi production snapshot'tan refresh et (PII anonimize ederek)

### Team & process
- **Performance test ownership:** Backend team + QA shared
- **Pre-merge gate:** PR build'lerde smoke; merge sonrası nightly
- **Capacity planning quarterly:** Stress test sonuçları → infrastructure budget input
- **Postmortem:** Production incident'lerden öğren — incident senaryosunu Gatling'e ekle

---

## 19. En Sık Mülakat Soruları (Hızlı Cevaplar)

| Soru | Hızlı Cevap |
|---|---|
| Gatling nedir? | Scala/Java/Kotlin DSL ile yazılan async I/O tabanlı yüksek perf load test framework'ü |
| JMeter'dan farkı? | Non-blocking I/O (Netty + Akka) → tek makineden 10x daha fazla VU |
| Simulation nedir? | Test senaryosunun tamamı, bir class olarak yazılır |
| Scenario nedir? | Bir VU'nun yapacağı action zinciri |
| Virtual User? | Test'te simüle edilen bağımsız bir kullanıcı (actor) |
| Session? | VU'nun kendi key-value state'i |
| Feeder? | Test data iterator (CSV/JSON/custom) |
| Check? | Response assertion + data extraction |
| Open vs closed workload? | Open: arrival rate (real traffic); Closed: sabit concurrent VU (sabit pool) |
| Think time neden önemli? | Gerçek user davranışı; yoksa gerçekçi olmayan yük çıkar |
| Assertion'ın amacı? | Test sonu SLA kontrolü → CI build pass/fail |
| Percentile method'ları? | p1=50, p2=75, p3=95, p4=99 |
| Smoke vs Load vs Stress vs Soak? | Smoke: sağlık; Load: normal; Stress: sınır; Soak: uzun süre leak |
| Mean vs percentile? | Mean yanıltıcı; p95/p99 kullan |
| Static asset'ler? | `.silentResources()` veya inferHtmlResources kapalı tut |
| Recorder ne işe yarar? | Browser proxy / HAR → senaryo prototype; production için re-write gerekir |
| Multi-node injection? | Gatling Enterprise veya cloud cluster |
| CI/CD entegrasyon? | Maven/Gradle plugin + assertion gate |
| Reporting? | Built-in HTML rapor; Enterprise real-time + history |
| Bottleneck analizi? | Client RT + server-side APM/metric karşılaştırması |
| Production'da test? | Mümkünse staging-mirror; production'da sadece read-only smoke |

---

## 20. Çalışma Yol Haritası

### 1. Hafta — Temeller
- [ ] Gatling bundle'ını indir, ilk simulation'ı koş
- [ ] Java/Kotlin DSL ile basit bir HTTP test'i yaz
- [ ] HTML raporu incele, percentile'ları öğren
- [ ] Recorder ile bir flow kaydet ve refactor et

### 2. Hafta — Orta
- [ ] Feeder (CSV) ile dynamic test data
- [ ] Check'ler ile data extraction + session variable
- [ ] Conditional logic, loops, groups
- [ ] Pause / pace ile think time modelleme

### 3. Hafta — Injection & Assertions
- [ ] Open vs closed workload modelini pratikte gör
- [ ] Farklı injection profile'ları dene
- [ ] Assertion'lar ile CI gate kur
- [ ] Smoke + load + stress + soak senaryolarını yaz

### 4. Hafta — Framework & Lead
- [ ] Scenario reuse + util class'lar
- [ ] Env-based config (staging/prod/local)
- [ ] CI/CD pipeline + report artifact
- [ ] Server-side observability ile korelasyon
- [ ] Bottleneck analizi practice — APM dashboard ile birlikte koş
- [ ] Performance test stratejisi dökümante et (SLA, schedule, ownership)

---

## 21. Kaynaklar

- **Resmi:** gatling.io
- **Docs:** docs.gatling.io
- **GitHub:** github.com/gatling/gatling
- **Java DSL docs:** docs.gatling.io/reference/script/core/scenario/
- **Sample sims:** github.com/gatling/gatling-maven-plugin-demo-java
- **Performance Testing in DevOps:** Pluralsight / Udemy courses
- **Kitap:** "Continuous Delivery" — Jez Humble (CI gate stratejisi)
- **Blog:** gatling.io/blog
- **Topluluk:** community.gatling.io
