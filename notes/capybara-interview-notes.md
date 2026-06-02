# Question 1

## Question

Capybara nedir ve hangi amaçla kullanılır? Hangi programlama dili ekosisteminde yer alır ve temel olarak hangi tip testler için tasarlanmıştır?

## Candidate Answer

Capybara bir DSL dir. Action methodlarda oto waiti built-in dir. Methodlar daha okunaklıdır. Ruby programlama dilinde yer alır sadece. Genelde Web UI testleri için tasarlanmıştır.

## Score

6 / 10

## What Was Good

- Capybara'nın bir DSL (Domain Specific Language) olduğunu doğru ifade etti.
- Built-in auto-wait özelliğine değindi — bu Capybara'nın en kritik özelliklerinden biri.
- Okunabilir API yapısını vurguladı.
- Ruby ekosisteminde yer aldığını belirtti.
- Web UI testleri için tasarlandığını söyledi.

## Missing Points

- Capybara'nın özellikle **acceptance / integration / end-to-end** test türleri için tasarlandığı söylenmedi.
- **Driver-agnostic** olduğu (Selenium, Rack-Test, Cuprite, Apparition gibi farklı driver'larla çalışabildiği) belirtilmedi.
- Hangi test framework'leri ile entegre olduğu söylenmedi (**RSpec, Cucumber, Minitest, Test::Unit**).
- Kullanıcı davranışını taklit ederek (click, fill_in, visit gibi) gerçek bir kullanıcı gibi web uygulamasıyla etkileşime girdiği vurgulanmadı.
- Rack tabanlı uygulamalar için **JavaScript gerektirmeyen hızlı test** (Rack::Test driver) imkânı sunduğu belirtilmedi.

## Ideal Answer

Capybara, Ruby ekosisteminde geliştirilmiş bir **acceptance / integration test DSL'idir**. Temel amacı, gerçek bir kullanıcının web uygulamasıyla nasıl etkileşim kuracağını simüle ederek **end-to-end (E2E) testler** yazmayı kolaylaştırmaktır. `visit`, `click_on`, `fill_in`, `have_content` gibi yüksek seviyeli ve okunaklı method'lar sunar.

En önemli özelliklerinden biri **built-in auto-waiting / synchronization** mekanizmasıdır: asenkron olarak yüklenen elementlerin gelmesini otomatik bekler, böylece flaky test'leri azaltır.

Capybara **driver-agnostic** bir yapıdadır; Selenium (Chrome, Firefox), Rack::Test (JS gerektirmeyen, çok hızlı), Cuprite (CDP üzerinden Chrome) ve Apparition gibi farklı sürücülerle çalışabilir. **RSpec, Cucumber, Minitest** gibi test framework'leri ile sorunsuz entegre olur. Genellikle Rails projelerinin standart UI test katmanını oluşturur.

## Additional Notes

- Capybara, Jonas Nicklas tarafından geliştirilmiştir.
- "DSL" derken aslında **Capybara::DSL** modülünü kastediyoruz — test dosyalarına `include Capybara::DSL` yapıldığında method'lar global olarak erişilebilir hale gelir.
- QA Lead seviyesinde bir cevap için "ne işe yarar" kadar **"hangi mimari kararları sağlar"** da önemlidir: örneğin driver swap'ı sayesinde headless / headful, hızlı / yavaş test ayrımı kolayca yapılabilir.
- Capybara'nın Ruby dışında doğrudan portu yoktur, ancak Python'da Splinter, JavaScript'te WebdriverIO / Playwright benzer felsefeyi taşır.

# Question 2

## Question

Capybara'nın `visit` method'u ne işe yarar ve davranışı hangi konfigürasyona bağlıdır? Örneğin `visit("/login")` çağrısının arkasında ne olur?

## Candidate Answer

Hiçbir bilgim yok.

## Score

0 / 10

## What Was Good

- Bilmediğini dürüstçe ifade etmesi, mülakatta "uydurmaya çalışmamak" açısından olumlu bir davranıştır. Ancak teknik puanlama açısından cevap boş olduğu için puan verilemez.

## Missing Points

- `visit` method'unun ne yaptığı (verilen path/URL'ye gitmek).
- Path'in **`Capybara.app_host`** ve **`Capybara.default_host`** konfigürasyonlarına göre nasıl çözüldüğü.
- Driver'a göre davranış farklılığı (Rack::Test: HTTP request simülasyonu; Selenium: gerçek tarayıcıyı yönlendirir).
- Tam URL (`https://...`) verildiğinde nasıl davrandığı.
- `Capybara.app` (Rack uygulaması) ile ilişkisi.
- `visit` sonrası `current_path`, `current_url` ile sayfa doğrulaması.

## Ideal Answer

`visit` method'u, Capybara'nın tarayıcıyı (veya driver'ı) belirtilen path / URL'ye yönlendirmesini sağlar. `visit("/login")` çağrısının arkasında şu süreç işler:

1. **Path mi, full URL mi?** Verilen string `http(s)://` ile başlıyorsa olduğu gibi kullanılır. Aksi halde Capybara onu bir path olarak yorumlar.
2. **Host çözümlemesi:** Capybara, `Capybara.app_host` veya `Capybara.default_host` konfigürasyonunu kullanarak base URL'yi belirler. `app_host` set edilmemişse:
   - Rack::Test driver'da Capybara'nın in-memory'de host ettiği **`Capybara.app`** (Rack uygulaması) hedeflenir.
   - Selenium gibi gerçek tarayıcı driver'larında `default_host` (varsayılan `http://www.example.com`) baz alınır; lokal Rails sunucusu için genellikle `Capybara.server_host` + `server_port` ile dinamik bir host ayarlanır.
3. **HTTP/Navigasyon:** Rack::Test, HTTP request'i in-process simüle eder (gerçek bir tarayıcı yoktur, çok hızlıdır). Selenium / Cuprite gerçek tarayıcıyı `driver.navigate.to(url)` ile o sayfaya yönlendirir.
4. **Sonrası:** Sayfa yüklendikten sonra `current_path`, `current_url`, `page.html` üzerinden assertion yazılabilir.

Örnek konfig:

```ruby
Capybara.app_host = "http://localhost:3000"
Capybara.run_server = false   # external server kullanıyorsak
visit("/login")               # => http://localhost:3000/login
```

## Additional Notes

- `visit` **idempotent değildir** — aynı URL'ye iki kez çağırırsan sayfa iki kez yüklenir; bu bazen testte beklenmedik state reset'lerine neden olur.
- `visit` sonrası asenkron JS varsa, doğrudan assertion yapmak yerine Capybara'nın `have_*` matcher'larını kullanmak gerekir; bunlar auto-wait yapar.
- QA Lead seviyesinde önemli bir nokta: **external uygulama testlerinde** (`Capybara.run_server = false`, `app_host = "https://staging.example.com"`) Capybara aslında bir HTTP client'a dönüşür ve in-memory app sunmaz.
- "Bilmiyorum" demek dürüstlük açısından olumlu, ancak mülakatta tamamen pas geçmek yerine **"şöyle tahmin ediyorum: muhtemelen bir URL'ye gidiyor, host'u config'ten alıyor olmalı..."** gibi mantıksal bir akıl yürütme denemesi puan getirir. Lead seviyesinde "bilmiyorum"u "akıl yürütüyorum"a çevirebilmek beklenir.

# Question 3

## Question

Capybara'da bir butona tıklamak için hangi method'ları kullanabilirsin? `click_button`, `click_link` ve `click_on` arasındaki farklar nelerdir? Hangi durumda hangisini tercih edersin?

## Candidate Answer

click_button, click_link ve click_on kullanabiliriz. Ancak arasındaki farklar hakkında bir bilgim yok.

## Score

2 / 10

## What Was Good

- Üç method'un da var olduğunu doğru tanımladı — method isimlerini hatırlaması temel bir farkındalık göstergesidir.
- Eksik bilgi olduğunu açıkça ifade etti (dürüstlük).

## Missing Points

- `click_button` → yalnızca `<button>` ve `<input type="submit|button|image|reset">` tipi elementleri hedefler.
- `click_link` → yalnızca `<a href="...">` tipi anchor elementlerini hedefler.
- `click_on` → hem link hem button için ortak (alias) bir method'dur; ikisini de bulabilir.
- Locator olarak text, id, name, value, title, alt gibi birden fazla attribute ile element bulabildikleri.
- `click_link_or_button` ile `click_on`'un aynı şey olduğu (alias).
- Üretim/test stratejisinde **spesifik** olanın (button/link) genelde tercih edilmesi: yanlış element seçme riskini azaltır, intent'i daha net belli eder.

## Ideal Answer

Capybara'da tıklama için üç ana method vardır:

1. **`click_button(locator)`**
   - Yalnızca form butonlarını hedefler: `<button>`, `<input type="submit">`, `<input type="button">`, `<input type="image">`, `<input type="reset">`.
   - Locator olarak button'ın **text içeriği, value, id, name, title** kullanılabilir.
   - Örnek: `click_button("Submit")`, `click_button("login-btn")`

2. **`click_link(locator)`**
   - Yalnızca `<a href="...">` anchor elementlerini hedefler. Href'i olmayan `<a>` etiketleri eşleşmez.
   - Locator olarak link'in **text, id, title, alt** kullanılabilir.
   - Örnek: `click_link("Sign up")`, `click_link("forgot-password")`

3. **`click_on(locator)`** (alias: `click_link_or_button`)
   - Hem button hem link arar. Önce eşleşeni bulup tıklar.
   - Test yazarken element tipinden emin değilsen veya kasıtlı olarak ikisini de kapsamak istiyorsan kullanılır.

**Hangisini ne zaman?**
- **Üretim test framework'lerinde genelde spesifik olan (`click_button` / `click_link`) tercih edilir.** Intent'i okunabilir kılar, yanlış element bulma riskini azaltır (örneğin sayfada aynı text'e sahip bir link de varsa).
- **Page Object pattern'de** method ismi zaten intent'i taşıdığı için (`def submit_login; click_button("Login"); end`) spesifik olan tercih edilir.
- `click_on` daha çok keşif/prototip aşamasında veya generic helper'larda işe yarar.

```ruby
click_button("Submit")              # sadece button arar
click_link("Forgot password?")      # sadece <a> arar
click_on("Save")                    # ikisini de arar
```

## Additional Notes

- Capybara 3+ sürümlerinde `click_link_or_button` yerine `click_on` daha yaygındır.
- Spesifik olmamak (hep `click_on` kullanmak) Selenium WebDriver dünyasından gelenler için cazip gelir, ancak QA Lead seviyesinde **kod okunabilirliği ve niyet ifadesi (intent reveal)** önemlidir → `click_button("Login")` test'i okuyana "bu bir form submit'i" der.
- Üç method da **Capybara'nın auto-wait mekanizmasını kullanır**: element DOM'a gelene kadar `Capybara.default_max_wait_time` kadar bekler.
- Method'lar **disabled** butonları default olarak görmezden gelir (`disabled: false`); ihtiyaç halinde `disabled: true` opsiyonu ile override edilebilir.
- "Bilmiyorum" yerine — method isimlerinden bile çıkarım yapılabilir: `click_button` → button için, `click_link` → link için, `click_on` → "generic, ikisini de bul" olmalı. Lead mülakatta bu tarz **dilbilgisel akıl yürütme** beklenir.

# Question 4

## Question

Capybara'nın `Capybara.default_max_wait_time` ayarı ne işe yarar? Default değeri kaçtır ve test framework'ünde global olarak nasıl değiştirilir? Bu değeri tek bir method çağrısı için (örneğin sadece bir `find` için) nasıl override edersin?

## Candidate Answer

Maximum elementin actionable olma süresidir. Override ise yazdığın kodun yanına ekstra süre yazarak override edebiliriz.

## Score

4 / 10

## What Was Good

- "Maximum" kavramına değinmesi doğru — bu bir fixed sleep değil, **upper bound** bir polling süresidir.
- "Element actionable olma" yönüne işaret etmesi temel olarak doğru — Capybara'nın auto-wait'inin element/condition bekleme amacını yakalamış.
- Override edilebileceğini bilmesi doğru — yöntem doğru istikamette ("yanına ekstra süre yazarak") ama syntax verilmemiş.

## Missing Points

- **Default değer**: 2 saniye (Capybara 2.0+).
- Global ayarın **nereye yazıldığı**: genelde `spec_helper.rb` / `rails_helper.rb` / `support/capybara.rb` dosyasında `Capybara.default_max_wait_time = 5`.
- Tek bir çağrı için override: **`wait:` named parametresi** (`find(".foo", wait: 10)`, `have_content("X", wait: 8)`, `click_button("Save", wait: 1)` vb.).
- Bir **blok** için override: **`using_wait_time(seconds) do ... end`** helper'ı.
- Bunun bir **fixed sleep değil, polling** mekanizması olduğu — element 0.3 sn'de gelirse 5 sn beklemez.
- `Capybara.default_max_wait_time = 0` ile auto-wait'i tamamen kapatabileceğin (genelde anti-pattern).
- Çok yüksek vermenin (örn. 30 sn) flaky test'leri maskelediği, çok düşük vermenin ise gereksiz failure ürettiği — QA Lead seviyesinde bu **trade-off**'u bilmek beklenir.

## Ideal Answer

`Capybara.default_max_wait_time`, Capybara'nın bir element / koşulun gerçekleşmesini beklerken kullanacağı **maksimum süredir (saniye cinsinden)**. Bu bir **polling timeout**'tur — Capybara verilen süre içinde periyodik olarak (ms düzeyinde) DOM'u kontrol eder ve eşleşme bulduğu an devam eder. Yani fixed sleep değildir.

- **Default değer:** 2 saniye.
- **Global ayar:**
  ```ruby
  # spec_helper.rb / rails_helper.rb / support/capybara.rb
  Capybara.default_max_wait_time = 5
  ```
- **Tek bir çağrı için override** — `wait:` opsiyonu:
  ```ruby
  find("#dashboard", wait: 10)
  expect(page).to have_content("Welcome", wait: 8)
  click_button("Save", wait: 1)
  ```
- **Bir blok için override** — `using_wait_time` helper'ı:
  ```ruby
  using_wait_time(15) do
    find(".slow-loading-modal")
    click_on("Confirm")
  end
  ```

**QA Lead bakış açısı — trade-off:**
- Çok düşük (1 sn) → CI gibi yavaş ortamlarda **false negative / flaky** test'ler.
- Çok yüksek (30 sn) → gerçek bug'ları **maskeler**, test suite süresini şişirir.
- Endüstri pratiği: global olarak 5-10 sn, sadece istisnai yavaş akışlarda (uzun süren API call, modal animasyonu) `wait:` ile override.

## Additional Notes

- Capybara 1.x'te bu ayar `Capybara.default_wait_time` olarak geçiyordu — eski dökümantasyonda görebilirsin.
- `wait: false` veya `wait: 0` → o anki DOM'u kontrol et, bekleme yapma. Negative assertion'larda (örn. `have_no_content`) dikkatli olunmalı — bu opsiyon `have_no_*` matcher'larının doğru çalışmasını engelleyebilir.
- Capybara'nın polling interval'i konfigüre edilemez (default ~50ms) — bu nedenle wait sürelerini saniye olarak düşünmek yeterlidir.
- "Element actionable olma süresi" tabiri **Playwright/Selenium**'dan gelen bir kavramdır; Capybara'da daha geneldir: matcher'ın **true** dönmesi için beklenen süre. Sadece element bulma değil, `have_content`, `have_no_text`, `have_current_path` gibi assertion'lar da bu mekanizmayı kullanır.
- Lead mülakatlarda bu sorunun devamı olarak şu da gelebilir: *"O zaman `sleep 5` ile `Capybara.default_max_wait_time = 5` farkı nedir?"* — cevap: ilki her zaman 5 sn bekler, ikincisi "en fazla 5 sn, mümkün olduğu kadar erken devam et" demektir.

# Question 5

## Question

Capybara ile bir input field'a metin yazmak istiyorsun. Bunun için hangi method'u kullanırsın? Method'un imzası nasıldır (parametreler) ve label, name, id, placeholder gibi farklı locator stratejileriyle nasıl çalışır?

## Candidate Answer

fill methodunu kullanırdım. Yani imzası şu şekilde fill(element, string) gibi biliyorum.

## Score

3 / 10

## What Was Good

- Doğru istikamet: input'a metin yazmanın "fill" kelimesi ile ifade edildiğini hatırladı.
- Locator + value mantığını kavramış (iki parametre olduğunu söylemesi).

## Missing Points

- Method'un gerçek adı **`fill_in`** — `fill` değil. (Capybara'da underscore ile yazılır.)
- İkinci parametrenin **keyword argument** olduğu: `with: "text"` syntax'ı. `fill_in(locator, "text")` çalışmaz.
- Capybara'nın **locator çözümleme sırası** belirtilmedi:
  - Label text (en yaygın — `<label for="email">Email</label>` ise `fill_in "Email", with: "..."`)
  - id (`#email`)
  - name attribute (`name="email"`)
  - placeholder attribute (`placeholder="Your email"`)
- Alternatif method belirtilmedi: `find(...).set("value")`.
- `currently_with:` opsiyonu (mevcut değere göre filtreleme).
- `fill_options:` (clear: :backspace gibi özel davranışlar).
- Auto-wait davranışı — element gelene kadar bekler.

## Ideal Answer

Capybara'da bir input'a metin yazmak için **`fill_in`** method'u kullanılır. İmzası:

```ruby
fill_in(locator, with: "value", **options)
```

İkinci parametre **`with:` keyword argument** olarak verilir — `fill_in(locator, "value")` syntax'ı çalışmaz, hata verir.

**Locator çözümleme önceliği** (Capybara `fill_in` için DOM'da şu sırayla arar):
1. **Label text:** En yaygın ve en okunaklı kullanım. `<label for="email">Email Address</label>` varsa:
   ```ruby
   fill_in "Email Address", with: "test@example.com"
   ```
2. **id:** `<input id="email">` için `fill_in "email", with: "..."`.
3. **name attribute:** `<input name="user[email]">` için aynı locator name'i geçerli.
4. **placeholder:** `<input placeholder="Enter email">` için.

Tüm bu locator'lar **auto-wait** mekanizmasından faydalanır — input DOM'a gelene kadar `default_max_wait_time` boyunca beklenir.

**Yaygın opsiyonlar:**
```ruby
fill_in "Email", with: "a@b.com"
fill_in "Email", with: "new@b.com", currently_with: "old@b.com"  # mevcut değere göre filtre
fill_in "Notes", with: "x", fill_options: { clear: :backspace }   # özel temizleme
```

**Alternatif: `find(...).set(value)`**
```ruby
find("#email").set("a@b.com")
```
`set` daha düşük seviyelidir; checkbox/radio için de kullanılabilir. `fill_in` ise sadece text-like input'lar (`text`, `email`, `password`, `textarea`, vb.) içindir.

## Additional Notes

- **Label-based locator** endüstri pratiğinde önerilir çünkü:
  - Accessibility (a11y) garantisi sağlar — label'sız form testten kalır.
  - Tasarım değişikliklerinde (class/id refactor) test kırılmaz.
  - Kullanıcının ekranda gördüğü metin ile test'in kullandığı locator aynı olur (intent reveal).
- **QA Lead seviyesinde** sık sorulan bir trap: *"`fill_in 'email', with: 'foo'` neden bazen çalışmıyor?"* — cevap: label `for` attribute'u eşleşmiyor olabilir, ya da custom React/Vue component label'ı `<label>` etiketi değil `<span>` ile yazıyordur; bu durumda Capybara label-based locator'ı bulamaz.
- `fill_in` Selenium driver'da gerçek keystroke göndermez (default olarak `value` property'sini set eder); JS event'lerine bağımlı SPA'larda `:input`, `:change` event'leri tetiklenmeyebilir → o zaman `find(...).set(value)` + manuel `trigger("change")` veya `send_keys` gerekebilir.
- Method ismini hatırlama trick'i: **"_in" ile biten Capybara helper'ları form input'larıyla ilgilidir** (`fill_in`, `check_in`/`choose`, `select`, `attach_file`). "fill" tek başına Capybara'da yoktur.

# Question 6

## Question

Capybara'da `find` ile `all` method'ları arasındaki fark nedir? Eşleşme bulunamadığında ne olur? Birden fazla element eşleşirse `find` nasıl davranır?

## Candidate Answer

Bu soru hakkında hiçbir fikrim yok.

## Score

0 / 10

## What Was Good

- Dürüstçe bilmediğini söyledi. Ancak Lead seviyesinde "bilmiyorum" yerine **method isimleri üzerinden çıkarım denemesi** beklenir: `find` → "tekil bul", `all` → "hepsini bul" gibi.

## Missing Points

- `find(selector)` → **tek element** bekler ve döner.
- `find` eşleşme yoksa → **`Capybara::ElementNotFound`** exception fırlatır (auto-wait süresi dolduktan sonra).
- `find` **birden fazla** eşleşme bulursa → **`Capybara::Ambiguous`** exception fırlatır (default davranış: `Capybara.match = :smart`).
- `all(selector)` → **eşleşen tüm elementleri** array benzeri (`Capybara::Result`) olarak döner.
- `all` eşleşme yoksa → exception fırlatmaz, **boş Result** döner (default olarak auto-wait yapmaz, ancak `minimum:`/`count:` ile bekleyebilir).
- `Capybara.match` modları: `:smart` (default), `:first`, `:one`, `:prefer_exact`.
- `find(...).first`, `first(selector)` ile ilk eşleşeni almak.
- `find` auto-wait yapar, `all` default olarak yapmaz.

## Ideal Answer

**`find(selector, **opts)`**
- DOM'da **tek bir element** bekler ve `Capybara::Node::Element` döner.
- **Auto-wait yapar** — `Capybara.default_max_wait_time` boyunca polling ile bekler.
- **Eşleşme yoksa:** `Capybara.default_max_wait_time` süresi dolduğunda `Capybara::ElementNotFound` fırlatır.
- **Birden fazla eşleşme varsa:** `Capybara.match` ayarına bağlıdır:
  - `:smart` (default) → birden fazla eşleşme `Capybara::Ambiguous` exception'ı fırlatır.
  - `:first` → ilk eşleşeni döner.
  - `:one` → birden fazla varsa exception (`:smart`'a benzer ama exact match'i farklı yönetir).
  - `:prefer_exact` → exact match varsa onu seçer, yoksa partial.

```ruby
find("#submit")                  # tek element bekler
find(".item", match: :first)     # birden fazla varsa ilkini al
find("li", text: "Foo")
```

**`all(selector, **opts)`**
- Eşleşen **tüm elementleri** `Capybara::Result` (Array-like) olarak döner.
- **Default olarak auto-wait yapmaz** (en az 1 element için bile). Bekleme istiyorsan `minimum:`/`maximum:`/`count:`/`between:` opsiyonu vermelisin.
- **Eşleşme yoksa:** Exception **fırlatmaz**, boş `Result` döner. Bu nedenle `all(...).empty?` kontrolü flaky olabilir (henüz yüklenmemiş olabilir).

```ruby
all(".item")                       # bekleme yok, anlık snapshot
all(".item", minimum: 3)           # en az 3 element gelene kadar bekler
all(".item", count: 5).map(&:text)
```

**Özet karar tablosu:**

| Durum | `find` | `all` |
|---|---|---|
| 0 eşleşme | `ElementNotFound` (after wait) | Boş Result |
| 1 eşleşme | Element döner | 1-elementlik Result |
| N eşleşme | `Ambiguous` (default) veya konfigure edildiği gibi | Tüm element'leri Result olarak döner |
| Auto-wait | ✅ Var | ❌ Yok (opsiyon vermezsen) |

## Additional Notes

- **`first(selector)`** = `all(selector).first` ancak auto-wait yapar; sık kullanılan kısayoldur.
- `all` default'ta auto-wait yapmadığı için **assertion'larda kullanmak antipattern**'dir. Bunun yerine `expect(page).to have_selector(".item", count: 3)` tercih edilir — bu Capybara'nın matcher tarafıdır ve auto-wait yapar.
- **QA Lead bakış:** Bir test'in `find(".item", match: :first)` ile yazılması bazen pragmatik görünür ama **niyeti gizler**. Eğer gerçekten "ilk item"ı test ediyorsan, daha spesifik bir locator yazmak (`first(".item")` veya `.item:first-child` veya parent ile scope etmek) daha iyidir.
- `Capybara::Ambiguous` exception'ı **CI'da çok değerlidir**: locator'larınızın gerçekten tek bir element'i hedeflediğini garanti eder. Bunu `match: :first` ile kapatmak silent failure'a sebep olabilir.
- "Hiçbir fikrim yok" yerine örnek bir akıl yürütme: *"İsimden tahmin edersem, `find` tek bir element bulur, `all` ise hepsini bulur. Tek bul derken bulunmazsa muhtemelen exception fırlatır, çünkü test'in akışı sürmemeli."* — bu cevap bilgi olmasa bile 3-4 puan getirir.

# Question 7

## Question

Capybara'da assertion yapmanın iki temel yolu vardır: truthy assertion (`have_*`) ve negative assertion (`have_no_*`). Neden `expect(page).not_to have_content("X")` yerine `expect(page).to have_no_content("X")` kullanılması tavsiye edilir?

## Candidate Answer

Bir fikrim yok.

## Score

0 / 10

## What Was Good

- Dürüst bir cevap. Ancak bu, Capybara mülakatlarında **en klasik trap sorularından biri** — bilinmediğinde "auto-wait" bilgisinin yarım anlaşıldığını gösterir.

## Missing Points

- İki ifadenin **auto-wait davranışının ters çalıştığı**.
- `expect(page).to have_no_content("X")` → "X kaybolana kadar bekle, kaybolmazsa fail" (auto-wait ile beraber çalışır).
- `expect(page).not_to have_content("X")` → "X şu an var mı? Hemen kontrol et" (auto-wait yapmaz, çünkü matcher'ın kendisi "var" diye doğruluyor, RSpec sonucu tersine çeviriyor).
- İkincisi **race condition / flaky test** üretir — özellikle DOM'dan kaybolması zaman alan elementler için.
- Aynı şey diğer `have_no_*` matcher'ları için de geçerli: `have_no_selector`, `have_no_link`, `have_no_button`, `have_no_text`, `have_no_field`, vb.
- Capybara'nın dahili polling mantığı (synchronize) → matcher'ın **kendisi** poll eder; RSpec'in `not_to`'su pas eder.

## Ideal Answer

İki ifade **mantıksal olarak aynı görünür ama Capybara'nın auto-wait mekanizması yüzünden tamamen farklı davranır.**

**`expect(page).to have_no_content("X")`:**
- Capybara matcher'ı `have_no_content`, **"X yok"** koşulunu polling ile bekler.
- Sayfa yüklenirken X bir an varsa ve sonra kaybolacaksa, matcher onun kaybolmasını bekler.
- `Capybara.default_max_wait_time` süresince poll eder; süre dolarsa fail eder.
- ✅ **Doğru ve güvenilir.**

**`expect(page).not_to have_content("X")`:**
- Capybara matcher'ı `have_content` çalışır — "X **var**" koşulunu polling ile bekler.
- X gerçekten yoksa, matcher `default_max_wait_time` boyunca bekler ve **false döner**.
- RSpec'in `not_to`'su sonucu tersine çevirir → assertion geçer **ama her seferinde gereksiz yere bekledi**.
- Daha kötüsü: X şu an **var** ama birazdan **kaybolacaksa**, matcher hemen `true` döner → `not_to` `false` yapar → **test fail eder**. Halbuki "kaybolmasını bekleyecektim" beklentisi vardı.
- ❌ **Race condition + her zaman yavaş.**

**Kural:** Negatif assertion'larda **her zaman** `have_no_*` matcher'ını kullan. Tüm Capybara matcher'larının negatif karşılığı vardır:

```ruby
expect(page).to have_no_content("Error")
expect(page).to have_no_selector(".spinner")
expect(page).to have_no_button("Save", disabled: true)
expect(page).to have_no_link("Logout")
expect(page).to have_no_field("Email", with: "old@x.com")
```

Bu kural **Capybara dökümantasyonunda da açıkça yazar** ve QA Lead mülakatlarında flaky test tartışmasının merkezindedir.

## Additional Notes

- Bunun teknik adı **"asymmetric matching with Capybara's synchronize"** — matcher'ın içinde `Capybara::Node::Base#synchronize` bloğu vardır ve sadece matcher'ın kendi tanımladığı pozitif/negatif koşulu poll eder. Dışarıdan `not_to` ile çevirmek poll mantığını tersine çevirmez.
- Aynı kural **`assert_no_selector`** (Minitest) için de geçerlidir: `assert_no_selector(".foo")` ≠ `refute(has_selector?(".foo"))`. İkincisi auto-wait yapmaz.
- **RuboCop-Capybara** veya **rubocop-rspec** plugin'lerinde `Capybara/NegationMatcher` ve `Capybara/NegationMatcherAfterVisit` gibi kurallar bu antipattern'i otomatik yakalar — QA Lead seviyesinde **CI'a böyle bir linter eklemek** beklenen bir karardır.
- Mülakatlarda sonraki adım: *"O zaman boolean predicate `has_no_content?` ile `!has_content?` farkı nedir?"* — aynı mantık: `has_no_content?` doğru poll yapar, `!has_content?` yapmaz.
- Lead seviyesinde flaky test analizinde **ilk bakılan yerlerden biri** budur — repository genelinde `not_to have_` aramak ve `have_no_`'ya dönüştürmek küçük ama yüksek-etkili bir flake-reduction kazanımıdır.

# Question 8

## Question

Capybara'da `page` objesi nedir? Bu objenin scope'u nedir (her test arasında ne olur)? Birden fazla sayfa/sekme ile çalışmak gerektiğinde nasıl yönetilir?

## Candidate Answer

Hiç bilmiyorum. Page object gibi bir şeye benziyor.

## Score

1 / 10

## What Was Good

- En azından bir tahmin yürüttü (`page` → Page Object). Bu doğru olmasa da "akıl yürütme" yaklaşımı puana değer.

## Missing Points

- `page` aslında **`Capybara::Session`** instance'ıdır — **Page Object pattern ile alakası yoktur**, isim benzerliği yanıltıcıdır.
- `page` mevcut **browser session**'ını (cookie'leri, current URL'i, açık tarayıcı window'unu) temsil eder.
- Capybara::DSL include edildiğinde `visit`, `click_on` gibi method'lar **örtük olarak `page` üzerinde** çalışır.
- **Lifecycle:** RSpec/Minitest'te her test sonrası `Capybara.reset_sessions!` otomatik çağrılır → session resetlenir, cookie/local storage temizlenir.
- **Multi-session:** `Capybara.using_session(:name) do ... end` ile farklı session'lar arasında geçiş yapılabilir (ör. admin ve user aynı test'te).
- **Multi-window/tab:** `page.windows`, `page.switch_to_window`, `within_window` ile sekmeler yönetilir.

## Ideal Answer

`page` Capybara'da global olarak erişilebilen **`Capybara::Session`** objesidir. **Page Object pattern ile karıştırılmamalıdır** — isim benzerliği rastlantısaldır.

Mevcut tarayıcı session'ını temsil eder: hangi URL'desin, hangi cookie'ler var, açık window'lar, vb. `visit`, `click_on`, `find`, `fill_in` gibi tüm DSL method'ları arka planda **`page.visit`, `page.find`, ...** olarak `page` üzerinde çalışır. Capybara::DSL include edildiğinde bu çağrılar örtük (implicit receiver) hâle gelir.

**Scope / Lifecycle:**
- Default driver kullanılırken `page` her test arasında **`Capybara.reset_sessions!`** ile resetlenir (RSpec'te `after(:each)` hook'u Capybara tarafından otomatik kurulur).
- Reset → cookie'ler temizlenir, current URL `about:blank` olur, ancak Selenium'da browser process'i ayakta kalır (performans).
- `Capybara.session_name` ile farklı session'lar yönetilebilir; default `:default`'tur.

**Multi-session (aynı test'te iki kullanıcı):**
```ruby
Capybara.using_session(:admin) do
  visit "/admin"
  click_on "Approve"
end

Capybara.using_session(:user) do
  visit "/profile"
  expect(page).to have_content("Approved")
end
```

**Multi-window / tab:**
```ruby
new_window = window_opened_by { click_link "Open in new tab" }
within_window(new_window) do
  expect(page).to have_content("Welcome to second page")
end

# veya manuel
page.switch_to_window(new_window)
page.current_window.close
```

`page.windows` → tüm açık window'ları döner; `page.current_window` → aktif olan; `window_opened_by { ... }` → blok içinde açılan yeni window'ı yakalar.

## Additional Notes

- **`page` ≠ Page Object Model.** POM kendi yazdığın `LoginPage`, `DashboardPage` gibi sınıflardır. Capybara'nın `page`'i ise framework'ün sana sunduğu session handle'ıdır. POM sınıfları içinde de `page.visit("/")`, `page.find(...)` çağırabilirsin.
- `current_session` = `Capybara.current_session` = `page` — üçü de aynı şeye işaret eder.
- **QA Lead bakışı:** Multi-tab senaryoları (OAuth popup, "open in new tab" flow'ları) Cuprite/Selenium'da güvenilir; **Rack::Test driver'da window/tab kavramı yoktur** → bu senaryolar driver seçimini etkiler.
- `Capybara.reset_sessions!` Selenium'da window'ları kapatmaz ama cookie'leri temizler — uzun test suite'lerde memory leak şüphesinde manuel `page.driver.quit` çağrılabilir.
- "Page object'e benziyor" sezgisi yanlış olsa da öğretici: mülakatta **"isim benzerliğine rağmen aslında session handle'ı, doğru mu hatırlıyorum?"** demek bilgi gösterir.







