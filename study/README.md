# QA Lead Mülakat Hazırlık — Çalışma Dosyaları

Bu klasörde QA Lead pozisyonu mülakatına hazırlık için üç ayrı framework rehberi bulunmaktadır.

## Dosyalar

| Dosya | İçerik | Süre tahmini |
|---|---|---|
| [`capybara.md`](./capybara.md) | Ruby web UI / acceptance test DSL'i | ~4 hafta |
| [`appium.md`](./appium.md) | Mobile (iOS / Android) UI otomasyonu | ~4 hafta |
| [`gatling.md`](./gatling.md) | Yük / performans test framework'ü | ~4 hafta |

## Her dosya şu yapıyı izler

1. **Framework nedir?** — temel tanım
2. **Mimari** — nasıl çalışır
3. **Kurulum** — başlangıç adımları
4. **Temel kavramlar ve DSL**
5. **İleri özellikler** (driver'lar, locator'lar, async, vb.)
6. **Best practice'ler** — endüstri standartları
7. **Pitfall'lar** — sık yapılan hatalar
8. **Framework tasarımı** — proje yapısı
9. **QA Lead seviyesi mimari kararlar** — strateji, scaling
10. **En sık mülakat soruları** — hızlı cevap tablosu
11. **4 haftalık çalışma planı**
12. **Kaynaklar**

## Önerilen çalışma sırası

1. **Capybara** — eğer iş Ruby/Rails projesindeyse öncelik
2. **Appium** — mobile testing odaklı pozisyonlar için
3. **Gatling** — performance/load testing sorumluluğu varsa

QA Lead mülakatlarının çoğu **bir framework derinliği + iki framework genel kültür** ister. Önce hedef pozisyonun ana framework'ünü ezbere, diğerlerini "ne işe yarar, ne zaman seçilir" düzeyinde öğren.

## İnterview notları

`../notes/capybara-interview-notes.md` dosyasında daha önce yaptığımız mock interview'un soru-cevap-değerlendirme kayıtları var. Düşük puan aldığın konular:

- **Soru 1 (6/10):** Capybara nedir? — Driver-agnostic ve framework entegrasyonu eksikti
- **Soru 2 (0/10):** `visit` ve config — bilinmiyor
- **Soru 3 (2/10):** click method'ları farkları — bilinmiyor
- **Soru 4 (4/10):** `default_max_wait_time` — partial
- **Soru 5 (3/10):** `fill_in` syntax — method ismi yanlış
- **Soru 6 (0/10):** `find` vs `all` — bilinmiyor
- **Soru 7 (0/10):** `have_no_*` vs `not_to have_*` — bilinmiyor (kritik trap)
- **Soru 8 (1/10):** `page` objesi — Page Object Model ile karıştırıldı

**Öncelikli çalışma:** `capybara.md` bölüm 4-8 (DSL, auto-wait, assertion'lar, locator stratejileri).
