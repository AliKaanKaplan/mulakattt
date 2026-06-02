# Load Testing — QA Lead Mülakat Hazırlık Rehberi (Ruby)

> Performans / yük testi konsepti ve Ruby ekosisteminde uygulanması.
>
> **⚠️ Önemli not:** **Gatling'in resmi Ruby DSL'i YOKTUR.** Gatling sadece Scala, Java ve Kotlin ile yazılır. Eğer mülakatta "Gatling biliyor musun?" sorusu gelirse, doğru cevap **"konseptlerini biliyorum, ama biz Ruby ekosisteminde load testing için X kullanıyoruz"** olacaktır.
>
> Bu doküman:
> 1. **Load testing konseptlerini** anlatır (Gatling, JMeter, k6 hepsinde aynı: percentile, workload model, SLA, vb.)
> 2. **Ruby ekosisteminde** load testing nasıl yapılır — `ruby-jmeter`, `Typhoeus`, `concurrent-ruby` ile pratik örnekler
> 3. Gerçek Gatling'i kullanman gerekirse Scala DSL'inin temellerini özetler

---

## 1. Load Testing Nedir, Neden Önemli?

**Load testing**, bir sistemin **belirli bir yük altında performansını ölçen** test türüdür. QA Lead seviyesinde bunun karşılığı:
- **Functional test:** "Çalışıyor mu?" → evet/hayır
- **Load test:** "Çalışıyor ama nasıl?" → response time, throughput, error rate

**Lead'in cevap vermesi gereken sorular:**
- Production'da pik saatte sistemimiz ayakta kalır mı?
- Black Friday yükü altında p99 latency'miz SLA'nın altında mı?
- 10K → 50K user'a scale ederken ne kırılır?
- Yeni release'in performans regression'ı var mı?

---

## 2. Ruby Ekosisteminde Load Testing Araçları

| Araç | Tür | Ruby DSL? | Capacity |
|---|---|---|---|
| **`ruby-jmeter`** | JMeter DSL Ruby'de | ✅ Tam | JMeter'ın kendisi kadar (binlerce VU) |
| **`Typhoeus`** | Parallel HTTP client | ✅ Native gem | Tek node'da ~1-5K VU |
| **`concurrent-ruby`** | Concurrency primitives | ✅ Native gem | Sınırı yok (thread pool size'a bağlı) |
| **`vegeta` (CLI)** | Go-based HTTP load tester | ❌ CLI'dan kullanılır; Ruby ile wrap edilebilir | 10K+ RPS |
| **`wrk` (CLI)** | C-based HTTP benchmark | ❌ CLI; Ruby script ile orchestrate edilir | 10K+ RPS |
| **`k6`** | JavaScript DSL, Go runtime | ❌ JS DSL | 10K+ VU |
| **Gatling** | Scala/Java/Kotlin DSL | ❌ **Ruby DSL'i yok** | 10K+ VU |

**QA Lead seçim kriteri:**
- **Küçük-orta sistem + Ruby team:** `ruby-jmeter` veya `Typhoeus` ile yazılı senaryolar
- **Büyük sistem + dedicated perf team:** Gatling (Scala) veya k6 (JS) — Ruby dışına çıkmak gerekir
- **Hızlı sanity / smoke benchmark:** `vegeta` veya `wrk` CLI

Bu dokümandaki örnekler **Ruby native (`ruby-jmeter`, `Typhoeus`, `concurrent-ruby`)** üzerine kurgulanmıştır.

---

## 3. Temel Kavramlar (Tüm araçlarda aynı)

### Virtual User (VU)
Test'te simüle edilen tek bir kullanıcı. Her VU bağımsız bir thread / fiber / actor.

### Throughput / RPS
Saniyede tamamlanan request sayısı. Sistemin **kapasitesini** ölçer.

### Response Time (RT)
Bir request'in başlangıcından response'unun tamamlanmasına kadar geçen süre.

### Percentile — mean'in tuzağı
Mean (ortalama) **yanıltıcıdır**. Bir kaç outlier ortalamayı şişirir.
- **p50 (median):** Kullanıcıların %50'si bu süreden hızlı yanıt aldı
- **p95:** Kullanıcıların %95'i bu süreden hızlı yanıt aldı (kötü deneyimi yakalar)
- **p99:** Kullanıcıların %99'u bu süreden hızlı (tail latency)
- **p99.9:** Outlier'lar (kritik sistemler için zorunlu)

**Lead seviyesinde altın kural:** SLA'yı `mean` ile değil **p95 / p99** ile tanımla.

### Open vs Closed Workload Model — Kritik Lead Konusu

**Open workload:**
- **Arrival rate** ile model (saniyede X user gelir)
- Yeni user'lar **mevcut user'ların durumundan bağımsız** gelir
- Gerçek dünyaya yakın (e-ticaret traffic, web sitesi)
- Sistem yavaşlarsa **queue büyür** → backpressure görülür

**Closed workload:**
- **Sabit sayıda concurrent user** korunur
- Bir VU bittiği an yeni biri başlar
- Performance test'lerde yaygın (sabit user pool)
- Sistem yavaşlarsa **throughput düşer** ama VU sayısı sabit

**Mülakat sorusu (klasik):** *"Open vs closed workload modelinin farkı nedir, hangisini ne zaman seçersin?"*
→ Cevap: Open → gerçek user trafiği simülasyonu (peak traffic, marketing campaign). Closed → mevcut user kapasitesini test etme (call center, fixed pool).

### Think Time
Gerçek user'lar her request arası düşünür (read content, navigate). Senaryolarda `pause` ile simüle edilir. **Olmazsa gerçekçi olmayan yük** çıkar.

### SLA (Service Level Agreement)
Sistemin karşılaması gereken kalite eşikleri. Örnek:
- p95 < 500ms
- p99 < 1000ms
- Error rate < %0.1
- Throughput > 1000 RPS

### Assertion
Test sonunda SLA kontrolü. Fail olursa CI build kırılır.

---

## 4. Load Testing Türleri (Tüm araçlarda geçerli)

| Tip | Amaç | Tipik konfig |
|---|---|---|
| **Smoke test** | Sistem cevap veriyor mu? | 1-5 VU, 1-2 dk |
| **Load test** | Normal yük altında performans | Production traffic seviyesi, 15-30 dk |
| **Stress test** | Sınırı bul, sistem nerede kırılır? | Sonsuza kadar ramp veya breaking point'e kadar |
| **Spike test** | Ani yük artışı | Anında %500 trafik → kurtarma davranışı |
| **Endurance / Soak test** | Memory leak, resource leak | Normal yük, 4-24 saat |
| **Capacity test** | Hangi VU sayısı SLA'yı bozmadan kaldırılabilir? | Adımlı ramp + assertion |
| **Scalability test** | Horizontal scaling ROI | Aynı yük, farklı pod/instance sayıları |

---

## 5. Ruby ile Load Testing — Yol 1: `ruby-jmeter`

`ruby-jmeter` Apache JMeter'ın altyapısını kullanır, ama senaryoları **Ruby DSL**'i ile yazarsın. JMeter XML yerine Ruby kodu — version control'e dost, code review yapılabilir, kompleks lojik kolay.

### Kurulum
```ruby
# Gemfile
gem "ruby-jmeter"
```

```bash
bundle install
# JMeter runtime gerekli:
# https://jmeter.apache.org/ → indir, $JMETER_HOME ayarla
```

### İlk test — Basit GET
```ruby
require "ruby-jmeter"

test do
  threads count: 100, rampup: 60, duration: 300 do
    visit name: "Home", url: "https://api.example.com/"

    think_time 1000, 3000   # 1-3 sn random pause

    visit name: "Search",
          url:  "https://api.example.com/search?q=phone" do
      extract name: "product_id", regex: 'id":"([^"]+)"'
      assert  contains: "results", scope: "main"
    end

    think_time 2000, 5000

    visit name: "Product detail",
          url:  "https://api.example.com/products/${product_id}"
  end
end.run(
  path:     "/Applications/jmeter/bin/",
  file:     "tmp/jmeter.jmx",
  log:      "tmp/jmeter.log",
  jtl:      "tmp/results.jtl"
)
```

### Login senaryosu — session değişken
```ruby
test do
  threads count: 50, rampup: 30, loops: 100 do
    submit name: "Login",
           url:  "https://api.example.com/login",
           fill_in: { email: "test@example.com", password: "secret" } do
      extract name: "auth_token", json: "$.token"
    end

    cookies clear_each_iteration: false

    header [
      { name: "Authorization", value: "Bearer ${auth_token}" }
    ]

    visit name: "Dashboard", url: "https://api.example.com/dashboard"
  end
end.run
```

### Feeder (CSV) — dinamik test data
```ruby
test do
  csv_data_set_config filename: "users.csv",
                      variable_names: "email,password",
                      recycle: true

  threads count: 100 do
    submit name: "Login",
           url:  "https://api.example.com/login",
           fill_in: { email: "${email}", password: "${password}" }
  end
end.run
```

**users.csv:**
```csv
email,password
user1@x.com,pass1
user2@x.com,pass2
```

### Assertion'lar
```ruby
visit name: "API call", url: "https://api.example.com/foo" do
  assert contains: "success"
  assert equals: 200, scope: "main"
  duration_assertion duration: 500
end
```

### Reporting
JMeter built-in HTML rapor:
```bash
jmeter -g tmp/results.jtl -o tmp/report
open tmp/report/index.html
```

---

## 6. Ruby ile Load Testing — Yol 2: `Typhoeus` (Parallel HTTP)

`Typhoeus` libcurl üstüne kurulu bir **paralel HTTP client**'dır. Senaryoları **doğrudan Ruby** ile yazarsın, JMeter runtime gerekmez. Tam kontrol istediğinde idealdir.

### Kurulum
```ruby
# Gemfile
gem "typhoeus"
```

### Basit paralel istek
```ruby
require "typhoeus"

hydra = Typhoeus::Hydra.new(max_concurrency: 100)

1000.times do |i|
  request = Typhoeus::Request.new(
    "https://api.example.com/products/#{i}",
    method:  :get,
    headers: { "Accept" => "application/json" }
  )

  request.on_complete do |response|
    puts "#{response.code} — #{response.total_time}s"
  end

  hydra.queue(request)
end

hydra.run
```

### Yük testi senaryosu — VU simülasyonu + metric toplama
```ruby
require "typhoeus"
require "json"

class LoadTest
  def initialize(base_url:, vu_count:, duration_seconds:)
    @base_url         = base_url
    @vu_count         = vu_count
    @duration_seconds = duration_seconds
    @results          = []
    @mutex            = Mutex.new
  end

  def run
    threads = Array.new(@vu_count) { |i| Thread.new { simulate_vu(i) } }
    threads.each(&:join)
    print_report
  end

  private

  def simulate_vu(vu_id)
    started = Time.now
    while (Time.now - started) < @duration_seconds
      record { Typhoeus.get("#{@base_url}/")          }   # Home
      sleep rand(1..3)                                     # think time
      record { Typhoeus.get("#{@base_url}/search?q=x") }  # Search
      sleep rand(2..5)
    end
  end

  def record
    t0 = Time.now
    response = yield
    dt_ms = ((Time.now - t0) * 1000).round
    @mutex.synchronize do
      @results << { code: response.code, ms: dt_ms }
    end
  end

  def print_report
    times    = @results.map { |r| r[:ms] }.sort
    success  = @results.count { |r| (200..299).cover?(r[:code]) }
    rate     = (success.to_f / @results.size * 100).round(2)
    p_idx    = ->(p) { times[((p / 100.0) * times.size).floor - 1] }

    puts "─────────── Load Test Report ───────────"
    puts "Total requests:    #{@results.size}"
    puts "Success rate:      #{rate}%"
    puts "p50 / p95 / p99:   #{p_idx.(50)}ms / #{p_idx.(95)}ms / #{p_idx.(99)}ms"
    puts "Max:               #{times.last}ms"
  end
end

LoadTest.new(
  base_url:         "https://api.example.com",
  vu_count:         100,
  duration_seconds: 60
).run
```

### Assertion ile CI gate
```ruby
report = LoadTest.new(...).run
abort "SLA fail: p95 = #{report[:p95]}ms > 500ms" if report[:p95] > 500
abort "SLA fail: success rate = #{report[:success_rate]}% < 99.5%" if report[:success_rate] < 99.5
```

---

## 7. Ruby ile Load Testing — Yol 3: `concurrent-ruby`

Daha gelişmiş concurrency primitives için (thread pool, future, async). Typhoeus gerektirmez, herhangi bir HTTP library ile çalışır.

### Kurulum
```ruby
# Gemfile
gem "concurrent-ruby"
gem "net-http-persistent"   # connection pool için
```

### Thread pool ile sabit concurrency (closed workload)
```ruby
require "concurrent-ruby"
require "net/http/persistent"
require "uri"

pool = Concurrent::FixedThreadPool.new(50)   # 50 concurrent VU
http = Net::HTTP::Persistent.new
metrics = Concurrent::Array.new

end_at = Time.now + 60   # 60 saniye boyunca koş

while Time.now < end_at
  pool.post do
    t0 = Time.now
    uri = URI("https://api.example.com/products")
    begin
      response = http.request(uri)
      metrics << {
        code: response.code.to_i,
        ms:   ((Time.now - t0) * 1000).round
      }
    rescue => e
      metrics << { code: 0, ms: nil, error: e.message }
    end
  end
end

pool.shutdown
pool.wait_for_termination

times = metrics.compact.map { |m| m[:ms] }.compact.sort
puts "p95: #{times[(times.size * 0.95).floor]}ms"
puts "p99: #{times[(times.size * 0.99).floor]}ms"
```

### Open workload — arrival rate (saniyede X user)
```ruby
require "concurrent-ruby"

executor   = Concurrent::ThreadPoolExecutor.new(
  min_threads: 5, max_threads: 500, max_queue: 0
)
arrival_rate = 20   # saniyede 20 yeni user
duration     = 60   # 60 saniye
metrics      = Concurrent::Array.new

scheduler = Concurrent::TimerTask.new(execution_interval: 1.0 / arrival_rate) do
  executor.post do
    t0 = Time.now
    response = Typhoeus.get("https://api.example.com/products")
    metrics << { code: response.code, ms: ((Time.now - t0) * 1000).round }
  end
end

scheduler.execute
sleep duration
scheduler.shutdown
executor.shutdown
executor.wait_for_termination

puts "Total requests: #{metrics.size}"
puts "p95: #{metrics.map { |m| m[:ms] }.sort[(metrics.size * 0.95).floor]}ms"
```

---

## 8. Senaryo Yapısı — Reusable Steps (Ruby idiom)

```ruby
# load_test/scenarios/login.rb
module Scenarios
  module Login
    def self.run(session:, base_url:, email:, password:)
      response = Typhoeus.post(
        "#{base_url}/login",
        body: { email: email, password: password }.to_json,
        headers: { "Content-Type" => "application/json" }
      )

      raise "Login failed" unless response.success?

      session[:auth_token] = JSON.parse(response.body)["token"]
      session
    end
  end
end

# load_test/scenarios/browse.rb
module Scenarios
  module Browse
    def self.run(session:, base_url:)
      Typhoeus.get(
        "#{base_url}/products",
        headers: { "Authorization" => "Bearer #{session[:auth_token]}" }
      )
    end
  end
end

# load_test/run.rb
session = {}
Scenarios::Login.run(session: session, base_url: ENV.fetch("BASE_URL"), email: "x@y.com", password: "secret")
Scenarios::Browse.run(session: session, base_url: ENV.fetch("BASE_URL"))
```

Bu Capybara'daki POM'un load test versiyonudur — **action chain'i sınıflarla** organize et.

---

## 9. Sık Karşılaşılan Pitfall'lar (Ruby ortamında)

### 1. Ruby GIL — gerçek CPU paralelizm yok
Ruby MRI'da GIL (Global Interpreter Lock) nedeniyle CPU-bound iş paralel çalışmaz. Ama load testing **I/O-bound**'dur, GIL bloke olmaz. Yine de:
- **10K+ VU** için tek MRI process yetersiz kalır → multiple process veya **JRuby/TruffleRuby** kullan
- Veya **Typhoeus + libcurl** (C-level paralel I/O) → GIL dışında çalışır

### 2. Test makinesi bottleneck olur
- Network out, CPU, file descriptor (`ulimit -n`) → izle
- `ss -s`, `htop` ile session ve CPU durumu
- **Distributed load injection** gerekir → AWS EC2 cluster, k6 cloud, JMeter slave node'ları

### 3. Think time eksikliği
```ruby
# ❌ Gerçekçi olmayan yük
loop { Typhoeus.get(url) }

# ✅
loop { Typhoeus.get(url); sleep rand(1..5) }
```

### 4. Cold cache / JIT warmup
İlk request'lerde database cache boş, JIT warmup yok → daha yavaş. **Warm-up phase** ekle ve sonuçlardan çıkar.

### 5. Production veritabanı kullanmak
Stress test production DB'sini mahvedebilir. Mutlaka:
- **Staging environment** (production-mirror)
- **Synthetic data** üret
- **Read-only test'leri ayır**, write-heavy'leri dikkatli koş

### 6. Assertion'ları unutmak
Assertion yoksa CI build fail olmaz → silent regression.

```ruby
# ✅ Her load test sonunda SLA gate
abort "p95 > 500ms (gerçek: #{p95}ms)" if p95 > 500
abort "Error rate > %0.5" if error_rate > 0.005
```

### 7. Mean'i baz almak
Outlier 1 request, mean'i 10x yapabilir. **p95/p99 kullan, mean'i sadece bilgi amaçlı raporla.**

### 8. Static asset'leri test etmek
CDN'den gelen image'leri test'e dahil etmek backend load test'inin amacını saptırır. Sadece API endpoint'lerini test et.

### 9. Stat'ları yanlış okumak
- **Mean** yanıltıcı; **percentile** kullan
- **Max** outlier olabilir; p99.9 daha bilgilendirici
- Response time histogramı her zaman incele

### 10. Production'da gerçekçi olmayan client davranışı
Ruby test'i HTTP request atar, ama gerçek user'ın **mobile network latency, DNS, TLS handshake** maliyetleri vardır. CDN/edge'yi gerçekçi test etmek için **distributed load injection** + farklı bölgeler gerekir.

---

## 10. CI/CD Entegrasyonu (Ruby)

### GitHub Actions örneği
```yaml
name: Load Test
on:
  schedule:
    - cron: "0 2 * * *"     # her gece 02:00 UTC
  workflow_dispatch:

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Run load test
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          DURATION: 300
          VU_COUNT: 200
        run: bundle exec ruby load_test/run.rb

      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: load-report
          path: tmp/load-report.html

      - name: Fail on SLA breach
        if: failure()
        run: |
          echo "::error::Load test SLA breached. Check the report artifact."
```

### Jenkins
```groovy
pipeline {
  agent any
  triggers { cron("H 2 * * *") }
  stages {
    stage("Load Test") {
      steps {
        sh "bundle exec ruby load_test/run.rb"
      }
    }
  }
  post {
    always {
      archiveArtifacts artifacts: "tmp/load-report.html"
      publishHTML target: [reportDir: "tmp", reportFiles: "load-report.html", reportName: "Load Report"]
    }
  }
}
```

---

## 11. Reporting (Ruby)

### Console summary (DIY)
```ruby
def print_summary(metrics)
  times = metrics.map { |m| m[:ms] }.compact.sort
  total = metrics.size
  ok    = metrics.count { |m| (200..299).cover?(m[:code]) }

  puts "─────────── Summary ───────────"
  puts "Total:   #{total}"
  puts "OK:      #{ok} (#{(ok.to_f / total * 100).round(2)}%)"
  puts "Mean:    #{(times.sum.to_f / total).round}ms"
  puts "p50:     #{times[(total * 0.50).floor]}ms"
  puts "p95:     #{times[(total * 0.95).floor]}ms"
  puts "p99:     #{times[(total * 0.99).floor]}ms"
  puts "Max:     #{times.last}ms"
end
```

### HTML rapor — `gruff` ile grafik
```ruby
# Gemfile
gem "gruff"

require "gruff"

g = Gruff::Line.new(800)
g.title = "Response Time over Time"
g.data("p95", p95_per_minute)
g.data("p99", p99_per_minute)
g.write("tmp/rt-chart.png")
```

### Allure entegrasyonu
Load test sonuçlarını Allure'a attachment olarak ekleyebilirsin:
```ruby
Allure.add_attachment(
  name: "Load report",
  source: File.read("tmp/load-report.html"),
  type: Allure::ContentType::HTML
)
```

---

## 12. Performans Test Stratejisi — QA Lead Bakışı

### Test takvimi
Her release için:
1. **PR-level smoke** — temel SLA, < 5 dk (Typhoeus ile basit script)
2. **Nightly load** — staging'de production-equivalent yük, 30 dk (ruby-jmeter)
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
- **Tek node:** ≤ 1-5K VU (Ruby + Typhoeus), hızlı kurulum
- **Distributed:** > 5K VU → JMeter slave node'ları, k6 cloud, Gatling Enterprise
- **Hybrid:** Ruby orchestration + vegeta/wrk CLI çağırma

### Observability entegrasyonu
Ruby script'in çıktısı sadece **client-side metric**. Server-side metric'ler için:
- **Grafana + Prometheus** dashboard
- **APM:** Datadog, New Relic, Skylight (Rails specifik)
- **Logs:** ELK / Loki

Test sırasında **side-by-side dashboard** açık olmalı: client RT vs server CPU/memory/DB pool.

### Bottleneck analizi
RT yüksek + error yoksa → bottleneck nerede?
1. **Application CPU?** APM profiling
2. **DB?** Slow query log, connection pool exhaustion (Rails'de `db:pool` limiti)
3. **External API?** Downstream dependency monitoring
4. **Network?** Bandwidth saturation
5. **GC?** MRI heap analysis, GC tuning

Lead'in görevi sadece **"yavaş"** demek değil, **"şu bottleneck yüzünden yavaş"** diye iddia üretmek.

### Test data management
- **Test user pool** — her test koşusunda 1000+ unique user
- **Idempotent operations** test'leri tekrar koşulabilir kalmalı
- **Cleanup:** soak/stress test sonrası DB cleanup script
- **Data refresh:** her hafta staging DB'yi production snapshot'tan refresh et (PII anonimize ederek)

### Team & process
- **Performance test ownership:** Backend team + QA shared
- **Pre-merge gate:** PR build'lerde smoke; merge sonrası nightly
- **Capacity planning quarterly:** Stress test sonuçları → infrastructure budget input
- **Postmortem:** Production incident'lerden öğren — incident senaryosunu load test'e ekle

---

## 13. Gatling Konseptleri (Mülakatta soru gelirse)

Gatling Ruby DSL'i olmasa da konseptlerini bilmek QA Lead seviyesinde beklenir. **Mülakatta sorulursa dürüstçe:**

> *"Gatling'i konsept olarak biliyorum; Ruby DSL'i olmadığı için pratikte ekibimde ruby-jmeter ve Typhoeus tabanlı çözümler kullandık. Eğer dedicated bir performance team kuracak olursak Gatling veya k6'ya geçiş için Scala/JS team kapasitesi de planlanır."*

### Gatling mimari (kısaca)
- **Async I/O** (Netty + Akka actor model) → JMeter'ın aksine her VU bir thread değil, actor → tek makinede 10K+ VU
- **Scala/Java/Kotlin DSL** ile scenario tanımı
- **Built-in HTML rapor** çok zengin
- **Open vs closed workload** native destek
- **CI/CD plugin'leri** (Maven, Gradle, sbt)
- **Gatling Enterprise** → multi-node, real-time dashboard (ücretli)

### Gatling Scala DSL örneği (referans için)
```scala
class BasicSimulation extends Simulation {
  val httpProtocol = http
    .baseUrl("https://api.example.com")
    .acceptHeader("application/json")

  val scn = scenario("Browse")
    .exec(http("Home").get("/"))
    .pause(1, 3)
    .exec(http("Search").get("/search?q=phone")
      .check(status.is(200))
      .check(jsonPath("$.results[0].id").saveAs("productId")))
    .pause(2)
    .exec(http("Product").get("/products/${productId}"))

  setUp(
    scn.inject(
      rampUsers(100).during(60),
      constantUsersPerSec(10).during(120)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile3.lt(500),   // p95 < 500ms
     global.successfulRequests.percent.gt(99.0)
   )
}
```

**Bilmen yeterli:** `setUp`, `inject`, `rampUsers`, `constantUsersPerSec`, `assertions`, `percentile`.

---

## 14. En Sık Mülakat Soruları (Hızlı Cevaplar)

| Soru | Hızlı Cevap |
|---|---|
| Load testing nedir? | Sistemin belirli yük altında performansını (RT, throughput, error rate) ölçer |
| Mean vs percentile? | Mean yanıltıcı; p95/p99 kullan — kötü deneyimi yakalar |
| Open vs closed workload? | Open: arrival rate (gerçek traffic); Closed: sabit concurrent VU (sabit pool) |
| Think time neden önemli? | Gerçek user davranışı; yoksa gerçekçi olmayan yük çıkar |
| Ruby ile load test? | ruby-jmeter (JMeter DSL), Typhoeus (parallel HTTP), concurrent-ruby (low-level) |
| Gatling Ruby DSL var mı? | **Hayır**, sadece Scala/Java/Kotlin. Ruby team Gatling kullanmak isterse Scala kapasitesi gerekir |
| ruby-jmeter ne işe yarar? | Apache JMeter'ı Ruby DSL ile yazmana izin verir; XML yerine kod |
| Typhoeus avantajı? | libcurl tabanlı, gerçek parallel HTTP; GIL dışında çalışır |
| GIL ve Ruby load test? | I/O-bound olduğu için GIL pek bloke olmaz; Typhoeus zaten C-level paralel |
| Smoke vs Load vs Stress vs Soak? | Smoke: sağlık; Load: normal; Stress: sınır; Soak: uzun süre leak |
| SLA assertion'ın amacı? | CI build pass/fail — silent regression engelleme |
| Bottleneck analizi? | Client metric + server APM/metric karşılaştırması |
| Production'da test? | Mümkünse staging-mirror; production'da sadece read-only smoke |
| Distributed load injection? | JMeter slave, Gatling Enterprise, k6 cloud, AWS EC2 cluster |
| Reporting? | ruby-jmeter → JMeter HTML; Typhoeus → DIY console + Gruff chart; cloud → Datadog APM |
| Lead'in en önemli kararı? | Hangi araç ekosistemle uyumlu (Ruby team → ruby-jmeter; perf team → Gatling/k6) |

---

## 15. Çalışma Yol Haritası

### 1. Hafta — Konseptler
- [ ] Percentile (p50/95/99) farkını anla, mean tuzağını gör
- [ ] Open vs closed workload modelini örneklerle öğren
- [ ] Smoke / load / stress / soak farkları
- [ ] Think time, ramp-up, plateau, ramp-down phase'leri

### 2. Hafta — Ruby pratik
- [ ] `Typhoeus` ile basit paralel request
- [ ] Custom metric toplama + percentile hesaplama scripti yaz
- [ ] `concurrent-ruby` ile thread pool ile closed workload simülasyonu
- [ ] Open workload (arrival rate) ile aynı sistemi karşılaştır

### 3. Hafta — Framework
- [ ] `ruby-jmeter` kurulumu + JMeter runtime
- [ ] CSV feeder ile dynamic test data
- [ ] Session değişkenleri + extract patterns
- [ ] SLA assertion ile CI gate

### 4. Hafta — Lead seviyesi
- [ ] Staging'e production-equivalent senaryo deploy et
- [ ] Server-side APM (Datadog/Grafana) ile korelasyon
- [ ] Bottleneck analizi (DB vs CPU vs network) practice
- [ ] Performance test stratejisi dokümante et (SLA tablosu, schedule, ownership)
- [ ] Gatling Scala DSL'ini en azından okuyacak kadar tanı (mülakat hazırlık)

---

## 16. Kaynaklar

### Ruby ekosistemi
- **`ruby-jmeter`:** github.com/flood-io/ruby-jmeter
- **`Typhoeus`:** github.com/typhoeus/typhoeus
- **`concurrent-ruby`:** github.com/ruby-concurrency/concurrent-ruby
- **`net-http-persistent`:** github.com/drbrain/net-http-persistent

### Genel performance testing
- **JMeter:** jmeter.apache.org
- **Gatling docs (konsept için):** docs.gatling.io
- **k6 (alternatif, JS):** k6.io
- **vegeta (CLI):** github.com/tsenart/vegeta
- **wrk (CLI):** github.com/wg/wrk

### Kitaplar / makaleler
- **"Continuous Delivery"** — Jez Humble (CI performance gate stratejisi)
- **"Site Reliability Engineering"** — Google (SLO/SLI/SLA tanımları)
- **"Designing Data-Intensive Applications"** — Martin Kleppmann (capacity planning)
- **Brendan Gregg — "Systems Performance"** — bottleneck analizi
