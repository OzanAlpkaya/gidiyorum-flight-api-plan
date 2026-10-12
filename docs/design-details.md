# Tasarım ayrıntıları

[← README](../README.md)

README'deki kararların uygulama ayrıntıları burada. Uçtan uca akış için README Bölüm 4'e bakın.

## 1. Arama verisi (Redis)

Üç kimlik birbirinden ayrıdır. Böylece aynı kriterle sonradan yapılan bir arama, önceki aramanın sonucunun üzerine yazamaz.

| Kimlik | Ne temsil eder | Üretimi |
|---|---|---|
| `searchId` | Bir istemcinin (oturumun) bir arama isteği | Her `POST /searches` için yeni UUIDv4 |
| `jobId` | Tek bir TourVisio arama işi; birden fazla `searchId` aynı işe bağlanabilir | Her yeni sağlayıcı çağrısı için yeni UUIDv4 |
| `snapshotId` | Bir işin normalize sonucu | İş tamamlanınca yeni UUIDv4; hiçbir zaman başka bir işin sonucuyla paylaşılmaz |

`criteriaHash` yalnızca bir arama isteğini mevcut bir işle eşleştirmek için kullanılır; sonuç anahtarı değildir. Hash normalize edilmiş kriterlerden üretilir: lokasyonlar, tarihler, yolcu dağılımı, kabin, para birimi, dil. Paylaşım oturumla sınırlıyken (varsayılan) hash'e oturum kimliği de eklenir.

İki ayrı Redis kullanılır:

- **Arama Redis'i** (Cluster, `noeviction`): arama ve iş kayıtları, snapshot'lar, yeniden kullanım işaretçileri, arama kilitleri ve lokasyon cache'i.
- **Token ve sayaç Redis'i** (küçük, replikalı): TourVisio token'ı, token yenileme kilidi, rate limit ve sağlayıcı eşzamanlılık sayaçları. Arama verisinin bellek baskısından etkilenmez.

| Anahtar | Depo | İçerik | Ömür |
|---|---|---|---|
| `search:{searchId}` | Arama | Kriterler, sahip (`client_id` + oturum), `jobId`, `createdAt` | 2 sa |
| `job:{jobId}` | Arama | `status` (`PENDING`, `COMPLETED`, `FAILED`), `failureCode`, `deadlineAt`, `snapshotId`, `expiresAt`, `resultsVersion` | En az 2 sa; kendisine bağlanan en son arama kaydıyla birlikte biter |
| `snap:{snapshotId}` | Arama | Sıkıştırılmış normalize sonuç, sıralama indeksleri, facet'ler, `nextOfferExpiresAt`, `resultsVersion` | Mutlak bitiş: `PXAT expiresAt` |
| `inflight:{criteriaHash}` | Arama | Devam eden işin `jobId`'si | `PX` = iş bütçesi + 2 sn |
| `reuse:{criteriaHash}` | Arama | Tamamlanmış ve yeniden kullanılabilir işin `jobId`'si | min(5 dk, `expiresAt` − şimdi) |
| `tv:token`, `tv:token:lock` | Token ve sayaç | TourVisio token'ı ve yenileme kilidi | `expiresOn` − 5 dk / 10 sn |
| `loc:{rol}:{hash}` | Arama | Lokasyon autocomplete yanıtı | 24 sa; yazma reddedilirse yanıt cache'lenmeden döner |

- `expiresAt` = min(TourVisio `expiresOn` − 60 sn, iş tamamlanma anı + 15 dk). İş tamamlanırken bir kez hesaplanır ve değişmez.
- Arama kayıtları 2 saat yaşar. Bir arama kaydı mevcut bir işe bağlandığında, iş kaydının bitişi `PEXPIREAT` ile en az bu arama kaydının bitişine çekilir (yalnızca ileri). Böylece iş kaydı, ona bağlı bir arama kaydından önce silinmez.
- Bu sürede dönen kullanıcı 410 ve orijinal kriterleri alır; sonrasında 404 alır. İki durumda da Next.js aynı kriterlerle yeniden arar.

## 2. Arama yaşam döngüsü

### Arama başlatma (`POST /searches`)

1. İstek doğrulanır, kriterler normalize edilir, `criteriaHash` hesaplanır.
2. `reuse:{criteriaHash}` varsa: Yeni bir `searchId`, mevcut tamamlanmış işe bağlanır ve 201 döner. TourVisio çağrılmaz.
3. Yoksa `SET inflight:{criteriaHash} <yeniJobId> NX PX <bütçe+2sn>` denenir:
   - **Kilit alındı:** `job:{jobId}` `PENDING` ve `deadlineAt` = şimdi + 20 sn ile yazılır. `searchId` bu işe bağlanır, 202 döner, iş arka planda başlar.
   - **Kilit alınamadı** (aynı arama zaten sürüyor): Kilitteki `jobId` okunur, yeni `searchId` bu işe bağlanır ve 202 döner. İkinci bir TourVisio çağrısı yapılmaz.

### İşin süre bütçesi

Tüm iş tek bir **20 sn**'lik bütçeye (`deadlineAt`) tabidir. Bütçeye gerekirse login, sağlayıcı denemeleri, retry beklemesi ve normalizasyon dahildir.

- Her deneme için timeout = min(12 sn, kalan süre − 1 sn normalizasyon payı).
- Retry yalnızca kalan süre en az 5 sn ise başlar.
- `deadlineAt` geldiğinde iş `AbortController` ile iptal edilir ve `FAILED` (`SUPPLIER_TIMEOUT`) yazılır.
- **Tamamlanma sırası:**
  1. Önce `snap:{snapshotId}` `PXAT expiresAt` ile yazılır.
  2. Snapshot yazımı reddedilirse (ör. bellek dolu), iş `FAILED` (`SEARCH_UNAVAILABLE`) olur. Snapshot'ı olmayan bir iş asla `COMPLETED` yapılmaz.
  3. Ardından atomik bir koşulla (Lua script) iş `COMPLETED` yapılır ve `snapshotId` kaydedilir. Koşul: kayıt hâlâ `PENDING` ve `deadlineAt` geçmemiş olmalı. Koşul sağlanmazsa geç sonuç yayımlanmaz ve yazılan snapshot silinir.
  4. Son olarak `reuse:{criteriaHash}` bu işe yönlendirilir, `inflight:{criteriaHash}` silinir.
- `inflight` kilidi bütçeden 2 sn uzun yaşar. Pod çökerse kilit kendiliğinden düşer; `deadlineAt`'i geçmiş `PENDING` iş okunurken `FAILED` sayılır.

### Listeleme okuması (`GET /results`)

Durum, snapshot'ın var olup olmamasından **önce** değerlendirilir. "Snapshot yok" tek başına "süre doldu" anlamına gelmez; arama henüz sürüyor da olabilir.

| Koşul | Yanıt |
|---|---|
| `search:{searchId}` yok veya sahiplik eşleşmiyor | 404 `SEARCH_NOT_FOUND` |
| Arama kaydı var ama iş kaydı yok (beklenmedik kayıp) | 410 `SEARCH_EXPIRED` + arama kaydındaki kriterler; istemci yeniden arar |
| İş `PENDING` ve `deadlineAt` geçmemiş | 202 + `Retry-After: 1` |
| İş `PENDING` ve `deadlineAt` geçmiş, ya da iş `FAILED` | `failureCode`'a göre 502 `SUPPLIER_ERROR`, 503 `SUPPLIER_UNAVAILABLE` veya 504 `SUPPLIER_TIMEOUT` |
| İş `COMPLETED` ve `expiresAt` geçmiş | 410 `SEARCH_EXPIRED` + orijinal kriterler |
| İş `COMPLETED`, süre dolmamış ama snapshot yok (beklenmedik veri kaybı) | 503 `SEARCH_UNAVAILABLE`; Next.js aynı kriterlerle yeniden arar |
| İş `COMPLETED`, snapshot var | 200 |
| İstemcinin gönderdiği `resultsVersion` güncel değil | 409 `RESULTS_CHANGED` + güncel `resultsVersion` |

### Sürümlü sonuçlar ve sayfa tutarlılığı

Teklifler tek tek dolduğu için sonuç kümesi zamanla değişebilir. Bu yüzden "değişmeyen sonuç" yerine **sürümlü sonuç** garantisi verilir:

- Snapshot, en erken dolacak teklifin zamanını (`nextOfferExpiresAt`) tutar. Bu zaman geçince süresi dolan teklifler çıkarılır. `fromTotal`, sıralama indeksleri ve facet'ler kalan geçerli tekliflerden yeniden hesaplanır ve `resultsVersion` bir artar.
- Yeniden yazma aynı `snapshotId` altında, sürüm karşılaştırmalı (eski sürüm hâlâ geçerliyse) ve **aynı `PXAT expiresAt`** ile yapılır. Böylece snapshot'ın ömrü uzamaz ve iki pod aynı anda yeniden hesaplasa bile yalnızca biri yazar.
- Her listeleme yanıtı `resultsVersion` döner. İstemci sonraki sayfa isteklerinde bu değeri gönderir. Sürüm değişmişse sayfa kaydırması yapılmaz; 409 `RESULTS_CHANGED` döner. Next.js listeyi birinci sayfadan yeniden yükler ve kullanıcıya "Sonuçlar güncellendi" der. Böylece hiçbir uçuş atlanmaz veya iki kez gösterilmez.
- Aynı sürüm içinde sıralama deterministiktir (seçilen alan, eşitlikte süre, sonra `offerId`).
- Gidiş-dönüşte geçerli uyumlu dönüşü kalmayan gidiş teklifi yeniden hesaplamada listeden çıkar. Kullanıcı bir gidiş seçmiş ve uyumlu dönüş kalmamışsa `leg=inbound` 200 ve boş liste döner; Next.js gidiş seçimine geri döner.

### Paylaşım

- **Varsayılan:** Bir iş yalnızca aynı oturumun tekrar eden aramalarında yeniden kullanılır (sayfa yenileme, çift tıklama, geri dönme). Aynı oturumdan eşzamanlı gelen aynı istekler tek TourVisio çağrısına iner.
- **SAN TSG onaylarsa** ([açık soru 16](../README.md#tourvisioya-açık-sorular)): Hash'ten oturum kimliği çıkarılır ve işler oturumlar arasında paylaşılır. Her oturum yine kendi `searchId`'sini alır; sahiplik kontrolü korunur. Bu kod değil konfigürasyon değişikliğidir.

### TourVisio token'ı ve eşzamanlı 401

- Token, `expiresOn` − 5 dk'da `tv:token:lock` kilidiyle tek pod tarafından yenilenir.
- Bir çağrı 401 alırsa koşulsuz silme yapılmaz; **karşılaştırmalı geçersizleştirme** yapılır. Bir Lua script, `tv:token` hâlâ başarısız çağrıda kullanılan token ise onu siler; değilse dokunmaz.
- Token başka bir pod tarafından zaten yenilenmişse çağrı yeni token'la bir kez tekrarlanır. Aynı token'sa normal yenileme yolu (kilit + tek login) izlenir, sonra çağrı bir kez tekrarlanır.
- Böylece gecikmiş bir 401, başka bir pod'un yeni aldığı token'ı silemez; eşzamanlı 401'ler de tek login'e iner.

### Pod kapanması ve Redis kesintisi

- Planlı kapanışta graceful shutdown devam eden işleri bütçeleri içinde tamamlar (`terminationGracePeriodSeconds` = 25 sn).
- Arama yan etkisiz bir okuma olduğu için kalıcı iş kuyruğu kullanılmadı; kaybolan bir iş bütçesi dolunca `FAILED` olur ve yeniden yapılabilir.
- Redis replikalı bir Cluster olarak çalışır; tek düğüm kaybında otomatik failover olur. Cluster tamamen erişilemezse API sözleşmesi değişmez, yalnızca hata döner:
  - Arama, listeleme ve teklif detayı 503 `SEARCH_UNAVAILABLE` + `Retry-After` döner. Sonuç ve sahiplik bilgisi saklanamayacağı için TourVisio çağrılmaz.
  - Lokasyon araması cache'siz, pod başına daha sıkı bir eşzamanlılık limitiyle çalışır.
  - TourVisio token'ı pod belleğindeki son kopyayla kullanılır.
  - Alarm üretilir.
- "Senkron, yalnızca ilk sayfa" gibi bir yedek mod bilinçli olarak seçilmedi; aynı endpoint'in yük altında farklı bir sözleşme sunması istemciyi karmaşıklaştırır.

## 3. Dayanıklılık parametreleri

| Konu | Uygulama |
|---|---|
| Timeout | Bağlantı kurma 1 sn; login 5 sn; autocomplete 2 sn. Arama işi: toplam 20 sn bütçe; deneme başına en fazla 12 sn (Bölüm 2) |
| Retry | Yalnızca bağlantı hatası, 502 ve 503'te; en fazla 1 kez, jitter'lı backoff ile ve yalnızca iş bütçesinde en az 5 sn kaldıysa. Timeout'ta arama tekrarlanmaz (retry fırtınası riski). Retry bütçesi toplam isteklerin %10'u |
| Circuit breaker | 30 sn'lik pencerede en az 20 istek ve %50 hata oranında açılır; 15 sn sonra yarı açık durumda deneme isteği gönderilir. Açıkken lokasyonda bayat cache, aramada 503 + `Retry-After` döner |
| Bulkhead | Autocomplete, arama ve `getoffers` için ayrı bağlantı havuzu ve ayrı eşzamanlılık limiti |
| Global eşzamanlılık | TourVisio'ya giden toplam eşzamanlılık, pod sayısından bağımsız olarak Redis'teki bir sayaçla sınırlanır |
| HPA | CPU ve "devam eden TourVisio çağrısı sayısı" özel metriği |
| Graceful shutdown | SIGTERM alındığında yeni iş kabul edilmez, devam eden işler bütçeleri içinde tamamlanır (`terminationGracePeriodSeconds` = 25 sn) |

## 4. Metrikler ve alarmlar

| Alan | Metrikler | Alarm |
|---|---|---|
| TourVisio | Gecikme p50/p95/p99, mesaj koduna göre hata oranı, timeout sayısı, circuit durumu, devam eden çağrı sayısı, kota kullanımı, `fees` alanı eksik olduğu için listelenmeyen teklif oranı | p99 > 10 sn, hata > %5, kota > %80, circuit açık |
| API (RED) | Endpoint bazında istek hızı, hata oranı, gecikme | 5xx > %2 |
| Cache | Lokasyon ve arama isabet oranı, birleştirme oranı, snapshot yeniden hesaplama ve 409 `RESULTS_CHANGED` oranı, bütçe aşımıyla atılan geç sonuç sayısı, Redis bellek ve gecikme | Redis bellek > %70 |
| Runtime | Event loop gecikmesi, heap, pod CPU ve bellek, replika sayısı | Event loop gecikmesi > 200 ms |
| Ürün | Boş sonuç oranı, 410 oranı, ilk sonuca kadar geçen süre, aramadan detaya dönüşüm, `priceChanged` oranı | Boş sonuç oranında ani artış |

## 5. Redis bellek tahmini

Duyarlılık hesabıdır, trafik tahmini değildir. Kayıt boyutları test hesabıyla ölçülecek.

| Veri | Hesap (50 yeni arama/sn sürekli) | Boyut |
|---|---|---|
| Snapshot | 50 × 900 sn ≈ 45.000 (paylaşım kapalıyken üst sınır) | 25 KB ile ≈ 1,1 GB; 100 KB ile ≈ 4,5 GB |
| Arama ve iş kayıtları | 50 × 7.200 sn ≈ 360.000 arama kaydı, en fazla aynı sayıda iş kaydı | Kayıt başına ~0,5 KB varsayımıyla ≈ 0,36 GB |
| Toplam üst sınır | | ≈ 1,5–4,9 GB + Redis yapı ek yükü + replikalar |

- Arama kayıtlarının ömrü bilinçli olarak 24 sa değil 2 sa tutuldu. 24 sa olsaydı aynı trafikte 4,3 milyon kayıt (≈ 2,2 GB) oluşurdu. 410 yanıtının kullanıcıya faydası, sayfaya kısa sürede dönen kullanıcı içindir.
- Snapshot başına sıkıştırılmış boyut sınırı konur; değer test hesabıyla ölçülen gerçek boyutlara göre belirlenir. Sınır aşılırsa sonuçlar kırpılmaz; eksik bir sonuç kümesi sıralamayı, filtreleri ve gidiş-dönüş eşleşmelerini bozar. Bunun yerine iş `FAILED` olur, istemci 503 `SEARCH_UNAVAILABLE` alır ve olay metrik/alarm olarak izlenir.
- **Bellek dolarsa:** Arama Redis'i `maxmemory-policy noeviction` ile çalışır: sessiz silme olmaz, yazma komutları reddedilir, okuma komutları çalışmaya devam eder. Bu durumda:
  - Yeni aramalar 503 `SEARCH_UNAVAILABLE` alır.
  - Yeniden hesaplama gerektirmeyen geçerli sonuçlar okunmaya devam eder.
  - Bir teklifin süresi dolduğu için yeni sürüm yazılması gerekiyorsa ve yazma reddedilirse, süresi dolmuş teklif yayımlanmaz; 503 `SEARCH_UNAVAILABLE` döner.
  - Lokasyon yanıtları cache'e yazılamaz ama sağlayıcıdan dönmeye devam eder.
  - Alarm %70'te tetiklenir. Token ve sayaç Redis'i ayrı olduğu için etkilenmez.

## 6. KVKK (ileride yolcu verisi)

- **Veri minimizasyonu:** Yalnızca TourVisio'nun `requiredFields` listesinde istenen ve yasal olarak zorunlu alanlar toplanır.
- **Şifreleme ve erişim:** TCKN ve pasaport numarası alan düzeyinde şifrelenir (KMS ile envelope encryption); erişim rol bazlıdır, her erişim audit log'a yazılır.
- **Gözlemlenebilirlik araçları:** Log, trace ve hata raporlarında (Sentry) kişisel veri bulunmaz; maskeleme varsayılan davranıştır.
- **Saklama süresi:** Tanımlı bir saklama süresi vardır; süre dolunca veriyi silen veya anonimleştiren bir iş çalışır.
- **Hukuki dayanak:** Aydınlatma metni hazırlanır; SAN TSG ile veri işleyen sözleşmesi yapılır. Veriler yurt dışında barındırılıyorsa yurt dışına aktarım hükümleri değerlendirilir ([açık soru 13](../README.md#tourvisioya-açık-sorular)).
