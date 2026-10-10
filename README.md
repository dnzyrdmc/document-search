# PDF ve Doküman Kaynak Arama

PDF veya metni küçük parçalara ayır; soruyla eşleşen kaynakları açıkça göster.

![Uygulama ekranı](docs/screenshot.png)

**Durum:** Çalıştırılabilir yerel temel sürüm (v0.1). Yerel retrieval · opsiyonel Gemini adaptörü.

## Kurulum

Node.js 24.x ve npm gerekir. İlk kurulumda npm paketlerini indirmek için internet bağlantısı gerekir. Node 24 `node:sqlite` deneysel uyarısı yazabilir; bu uyarı tek başına hata değildir.

```bash
npm ci
npm run dev
```

Tarayıcı: http://localhost:3000. Giriş: **demo@example.com / Demo12345!**. İkinci hesap: **other@example.com / Demo12345!**.

İkinci terminalde, aynı proje klasöründe:

```bash
npm run seed
```

Seed komutu örnek kayıt ekler; tekrar çalıştırmak yeni örnek kayıtlar oluşturabilir. Bazı projelerde başlangıç kataloğu zaten hazırdır; seed bunu açıklar.

## Derleme ve test

```bash
npm test
npm run typecheck
npm run build
npm start
```

`npm run dev` sırasında değişiklikleri Vite işler. `npm start` için önce build gerekir. Testler RAM veritabanı ve rastgele portla çalışır; kendi verilerini oluşturur. Mevcut demo veritabanını değiştirmez. Typecheck TypeScript giriş/ortak bileşenlerini ve Vue şablonlarını kapsar; JavaScript backend'in tam tip doğrulaması değildir.

## Çalışan özellikler

- Metin/PDF yükleme
- 800 karakterlik parçalar
- 150 karakter örtüşme
- Kelime eşleşmesiyle sıralama
- Kaynak parça numarası

## Kapsam sınırı

Varsayılan mod yapay zekâ yanıtı üretmez. GEMINI_API_KEY ve GEMINI_MODEL ile isteğe bağlı Gemini adaptörü vardır; anahtar olmadığı için canlı sağlayıcı çağrısı test edilmedi. Embedding, vektör arama ve OCR yok. PDF ilk 30 sayfa ve 100.000 karakterle sınırlıdır; limitler arayüzde açıklanır.

Yerel mod anahtarsız çalışır. Opsiyonel `.env`: GEMINI_API_KEY=... ve GEMINI_MODEL=hesabınızda erişilebilir model kimliği. `node --env-file=.env server/index.js --dev` ile başlat. Arayüzde Gemini seçeneğini işaretlersen yalnız eşleşen parçalar sağlayıcıya gönderilir. Canlı API çağrısı bu ortamda test edilmedi; ücret/kota sağlayıcı hesabına bağlıdır.

## Dosyalar ve akış

| Dosya | Sorumluluk |
| --- | --- |
| client/Workspace.vue | Projeye özel formlar, listeler, kullanıcı eylemleri |
| client/App.vue | Oturum açma ve ortak sayfa düzeni |
| client/api.ts | Fetch, hata mesajı, para/tarih yardımcıları |
| server/project.js | Alan kuralları, SQL sorguları ve API uçları |
| server/core.js | Veritabanı, doğrulama, oturum ve SSE yardımcıları |
| server/index.js | Express başlatma, güvenlik başlıkları ve statik dosyalar |
| schema.sql | Uygulamanın gerçek tablo/indeks şeması; referans amaçlı |
| tests/project.test.js | Gerçek HTTP istekleriyle kritik iş kuralları |
| scripts/seed.js | Örnek veri ekleme |
| PROJECT_DETAILS.pdf | Beş sayfalık proje açıklaması, API, test ve geliştirme rehberi |

Arayüz → aynı origin `/api` → oturum/sahiplik/doğrulama → iş kuralı → SQLite → JSON → görünüm. SQLite `data/app.sqlite` dosyasında kalıcıdır. Bu küçük uygulamalarda tablolar açılışta `CREATE TABLE IF NOT EXISTS` ile kurulur; sürümlü migration sistemi henüz yoktur.

## İş kuralı

PDF metni parser ile çıkarılır; boş/taranmış PDF reddedilir. Metin 800 karakterlik pencerelerle 650 adımda bölünür. Sorunun üç ve üzeri karakterli kelimeleri tekilleştirilir. Her parça içerdiği farklı sorgu kelimesi sayısıyla puanlanır. Pozitif puanlı ilk üç parça döndürülür; kaynak yoksa cevap uydurulmaz.

## API haritası

Oturum: `POST /api/auth/login` JSON `{"email":"demo@example.com","password":"Demo12345!"}`; `GET /api/auth/me`; `POST /api/auth/logout`. Çerez HttpOnly + SameSite=Strict. Tarayıcı aynı origin kullanır.

| Yöntem ve yol | Girdi | Başarı |
| --- | --- | --- |
| GET /api/documents | Gövde yok | 200 |
| POST /api/documents | JSON: name, base64, text | 201 |
| POST /api/documents/:id/ask | JSON: question, useLlm | 200 |
| DELETE /api/documents/:id | Gövde yok | 200 |

Uç nokta gövdelerinin somut örnekleri `tests/project.test.js` ve `scripts/seed.js` içinde bulunur. `:id` alanlarını önceki oluşturma yanıtından al. Hatalar JSON `{"error":"açıklama"}` biçimindedir; 401 giriş, 403 rol/origin, 404 kayıt/sahiplik, 409 çakışma, 422 doğrulama, 429 kota anlamına gelir. Listeler küçük yerel demo kapsamındadır; tümünde sayfalama yoktur.

```text
POST /api/documents
{"name":"Notlar","text":"SQLite verileri yerel dosyada ..."}
201 {"id":"<document-id>"}
```

## Kabul senaryoları

- [ ] SQLite sorusu SQLite içeren kaynak döndürmeli.
- [ ] Eşleşmeyen soru boş kaynak listesi üretmeli.
- [ ] Başka hesabın belgesi sorgulanamamalı.
- [ ] Örnek metin tabanlı PDF metni çıkarılmalı; canlı Gemini çağrısı ayrıca denenmeli.

## İlk gün yapacağın çalışma

Metin yükle ve belgede geçen bir terimi sor. Geçmeyen bir kelime sorup boş kaynak sonucunu göster. Örnek PDF’i de yükleyerek parser yolunu dene.

## Sonraki geliştirmeler

- [ ] Embedding ve vektör indeks ekle
- [ ] Gemini adaptörünü kendi anahtarınla doğrula
- [ ] Token bütçesi ve maliyet sınırı ekle
- [ ] Prompt injection ve belge izin testleri ekle

## GitHub sunumu

Önce kurulumu çalıştır, testleri oku ve en az bir davranışı kendin geliştir. Her gün yaptığın gerçek değişikliği açıklayan commit at. `feat: ...`, `fix: ...`, `test: ...`, `docs: ...` örnek öneklerdir. `docs/screenshot.png` başlangıç sürümünün ekranıdır; değişikliklerinden sonra kendi ekranınla güncelle.

`.gitignore`, node_modules, dist, data ve .env dosyalarını dışarıda bırakır. Veritabanını, anahtarları veya gerçek müşteri/aday belgelerini GitHub'a koyma. GitHub repo oluşturma/yükleme bu paket tarafından otomatik yapılmaz.

## Ortam ayarları

`.env.example` dosyasını `.env` olarak kopyala; dosya varsayılan npm komutlarında otomatik okunmaz. Kullanmak için `node --env-file=.env server/index.js --dev`. PORT varsayılan 3000, HOST 127.0.0.1, DB_PATH data/app.sqlite. DEMO_PASSWORD yalnız yeni veritabanında hesap oluşturulurken kullanılır; var olan parolayı değiştirmez. COOKIE_SECURE yalnız HTTPS ortamında 1 olmalı. Farklı projeleri aynı anda çalıştırırken farklı PORT kullan.

## Dağıtım notu

Bu sürüm yerel portfolyo/öğrenme içindir. Genel internete açmadan önce demo hesaplarını kaldırıp kayıt/parola sıfırlama ve gerçek kullanıcı yaşam döngüsü ekle. Tek süreç/senkron SQLite yaklaşımı yoğun trafikli hizmet için hedef mimari değildir. PostgreSQL geçişinde SQL tipleri, transaction sınırları, indeksler, migration ve yedeklemeyi ayrıca tasarla. Dockerfile genel Node uygulaması içindir; Docker runner ve FFmpeg gibi özel bağımlılıklar otomatik kurulmaz.

## Mülakat provası

Kelime eşleşmesi ile anlamsal arama arasındaki fark nedir? Kaynak gösterilmesi modelin her iddiasını doğrular mı?

## Lisans

MIT; bağımlılıkların kendi lisansları saklıdır.
