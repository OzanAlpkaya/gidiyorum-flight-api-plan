# API sözleşmesi

[← README](../README.md)

Bu doküman `flight-service`'in dışarıya açtığı sözleşmeyi tanımlar. Sözleşme tedarikçiden bağımsızdır: hiçbir alan adı, enum değeri veya ID TourVisio'ya özgü değildir. TourVisio karşılıkları için: [tourvisio-mapping.md](tourvisio-mapping.md).

## Genel kurallar

| Konu | Kural |
|---|---|
| Versiyonlama | URL'de: `/v1`. Geriye uyumsuz değişiklik yeni versiyonla yapılır |
| Format | JSON, alan adları `camelCase` |
| Tarih | Takvim tarihi `YYYY-MM-DD`; zaman damgası ISO 8601 UTC (`2026-10-13T09:29:00Z`); uçuş saatleri havalimanının yerel saati, saat dilimi eki olmadan (`localTime`) |
| Para | `{ "amount": "3890.00", "currency": "TRY" }`. `amount` decimal string'dir, float kullanılmaz |
| Lokasyon ID'si | `{TİP}:{KOD}`, örneğin `AIRPORT:SAW`, `CITY:IST` |
| Kimlik doğrulama | `Authorization: Bearer <JWT>` |
| İzlenebilirlik | İstemci `X-Request-Id` gönderebilir; gönderilmezse gateway üretir. Yanıtta `X-Request-Id` ve W3C `traceparent` döner |
| Hata gövdesi | RFC 9457 (`application/problem+json`): `type`, `title`, `status`, `detail`, `code`, `traceId`; doğrulama hatalarında ek olarak `errors[]` |
| Dil | `Accept-Language` başlığı (`tr-TR`, `en-US`); varsayılan `tr-TR` |

## Hata kodları

| HTTP | `code` | Ne zaman |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Şema hatası, eksik veya hatalı formatlı alan |
| 401 | `UNAUTHORIZED` | Token yok, imzası geçersiz veya süresi dolmuş |
| 403 | `FORBIDDEN` | Token geçerli, scope yetersiz |
| 404 | `SEARCH_NOT_FOUND`, `OFFER_NOT_FOUND` | Kayıt yok veya isteyen istemciye ait değil |
| 409 | `RESULTS_CHANGED` | Listelemede istemcinin `resultsVersion` değeri güncel değil |
| 410 | `SEARCH_EXPIRED`, `OFFER_EXPIRED` | Sağlayıcının verdiği geçerlilik süresi doldu |
| 422 | `BUSINESS_RULE_VIOLATION` | Şema doğru, iş kuralı ihlal ediliyor (geçmiş tarih, bebek sayısı > yetişkin sayısı, toplam yolcu > 9, kalkış = varış, dönüş < gidiş) |
| 429 | `RATE_LIMITED` | İstemci kotası aşıldı; `Retry-After` döner |
| 502 | `SUPPLIER_ERROR` | Sağlayıcı hata döndü |
| 503 | `SUPPLIER_UNAVAILABLE` | Circuit açık veya sağlayıcı kotası doldu; `Retry-After` döner |
| 503 | `SEARCH_UNAVAILABLE` | Arama deposu erişilemez veya dolu, sonuç beklenmedik biçimde kaybolmuş ya da sağlayıcı yanıtı boyut sınırını aşmış; `Retry-After` döner, istemci yeniden arar |
| 504 | `SUPPLIER_TIMEOUT` | Sağlayıcı zamanında yanıt vermedi |

---

## 1. Token alma

`POST /v1/auth/token` (`auth-service`), `Content-Type: application/x-www-form-urlencoded`

```
grant_type=client_credentials&client_id=web-bff&client_secret=********&scope=flights:search
```

**200 OK**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjYtMTAifQ...",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "flights:search"
}
```

Alan adları OAuth2 standardı (RFC 6749) gereği `snake_case`'tir; servisin geri kalanında `camelCase` kullanılır.

## 2. Lokasyon arama

`GET /v1/flights/locations/departures?q=ist&tripType=ONE_WAY`

`GET /v1/flights/locations/arrivals?q=ant&from=CITY:IST&tripType=ONE_WAY`

| Parametre | Zorunlu | Açıklama |
|---|---|---|
| `q` | Evet | En az 2 karakter |
| `tripType` | Evet | `ONE_WAY` veya `ROUND_TRIP` |
| `from` | Yalnızca `arrivals` için | Seçilen kalkış lokasyonunun `id`'si |

**200 OK** (isimler `Accept-Language`'e göre sağlayıcıdan gelir; değerler örnektir)

```json
{
  "items": [
    { "id": "CITY:IST", "type": "CITY", "code": "IST", "name": "İstanbul, Türkiye (Tümü)" },
    { "id": "AIRPORT:IST", "type": "AIRPORT", "code": "IST", "name": "İstanbul Havalimanı, Türkiye (IST)" },
    { "id": "AIRPORT:SAW", "type": "AIRPORT", "code": "SAW", "name": "İstanbul Sabiha Gökçen Uluslararası Havalimanı, Türkiye (SAW)" }
  ]
}
```

## 3. Uçuş arama

`POST /v1/flights/searches`

| Alan | Tip | Zorunlu | Kural |
|---|---|---|---|
| `tripType` | `ONE_WAY` \| `ROUND_TRIP` | Evet | |
| `origin`, `destination` | Lokasyon ID'si | Evet | Birbirinden farklı olmalı |
| `departureDate` | Tarih | Evet | Bugün veya sonrası, en fazla 330 gün ileri |
| `returnDate` | Tarih | `ROUND_TRIP` ise | `departureDate`'ten önce olamaz |
| `passengers.adults` | Tam sayı | Evet | En az 1 |
| `passengers.children` | Tam sayı | Hayır | Varsayılan 0 |
| `passengers.infants` | Tam sayı | Hayır | Varsayılan 0; yetişkin sayısını aşamaz |
| `cabin` | `ECONOMY` \| `BUSINESS` | Hayır | Varsayılan `ECONOMY` |
| `currency` | ISO 4217 | Hayır | Varsayılan `TRY` |

Toplam yolcu (yetişkin + çocuk + bebek) en fazla 9'dur.

**İstek (gidiş-dönüş)**

```json
{
  "tripType": "ROUND_TRIP",
  "origin": "CITY:IST",
  "destination": "AIRPORT:AYT",
  "departureDate": "2026-11-20",
  "returnDate": "2026-11-27",
  "passengers": { "adults": 2, "children": 1, "infants": 0 },
  "cabin": "ECONOMY",
  "currency": "TRY"
}
```

**202 Accepted** (yeni arama). Başlıklar: `Location: /v1/flights/searches/{searchId}/results`, `Retry-After: 1`

```json
{
  "searchId": "7b2e4d90-1c5a-4f3b-9e6d-0a8c2f5b7d14",
  "status": "PENDING",
  "links": { "results": "/v1/flights/searches/7b2e4d90-1c5a-4f3b-9e6d-0a8c2f5b7d14/results" }
}
```

**201 Created** (aynı kriterle 5 dk içinde yapılmış bir aramanın sonucu hazır)

```json
{
  "searchId": "a71d0c52-6e3f-4b18-9d24-58f0b3c6e911",
  "status": "COMPLETED",
  "expiresAt": "2026-10-13T09:29:00Z",
  "links": { "results": "/v1/flights/searches/a71d0c52-6e3f-4b18-9d24-58f0b3c6e911/results" }
}
```

**422 Unprocessable Content**

```json
{
  "type": "/problems/business-rule-violation",
  "title": "İş kuralı ihlali",
  "status": 422,
  "code": "BUSINESS_RULE_VIOLATION",
  "detail": "Bebek sayısı yetişkin sayısını aşamaz.",
  "traceId": "0af7651916cd43dd8448eb211c80319c",
  "errors": [{ "field": "passengers.infants", "rule": "LTE_ADULTS" }]
}
```

## 4. Listeleme

`GET /v1/flights/searches/{searchId}/results`

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `page` | 1 | 1'den başlar |
| `pageSize` | 20 | En fazla 50 |
| `sort` | `price` | `price`, `duration`, `departureTime` |
| `order` | `asc` | `asc`, `desc` |
| `airlines` | | Virgülle ayrılmış havayolu kodları: `TK,PC` |
| `stops` | | `0`, `1`, `2` (`2` = iki ve daha fazla aktarma); virgülle çoklu seçim |
| `departureFrom`, `departureTo` | | Kalkış saati aralığı, `HH:mm` |
| `leg` | `outbound` | Gidiş-dönüşte `outbound` veya `inbound` |
| `outboundOfferId` | | `leg=inbound` ise zorunlu; seçilen gidiş teklifi |
| `resultsVersion` | | Birinci sayfadan sonraki isteklerde, önceki yanıttaki değer gönderilir. Güncel sürümle eşleşmezse 409 `RESULTS_CHANGED` döner |

**Kurallar**

- Listedeki her öğe bir uçuştur. `offers[]`, o uçuşun seçilebilir tüm marka/paket tekliflerini taşır ve fiyata göre sıralıdır. İstemci herhangi birini detay için seçebilir.
- Tüm fiyatlar aramadaki tüm yolcular için ve ücretler dahil **nihai** tutardır ([hesaplama kuralı](tourvisio-mapping.md#toplam-fiyat-hesaplama)).
- **Tek yön:** `offers[].price` nihai tutardır; `lowestPrice` = `offers[0].price`. `sort=price` `lowestPrice` değerine göre sıralar.
- **Gidiş-dönüş, `leg=outbound`:**
  - `offers[].legPrice` yalnızca gidiş teklifinin tutarıdır.
  - `offers[].fromTotal`, o gidiş teklifinin uyumlu bir dönüşle oluşturabileceği en düşük nihai toplamdır.
  - `offers[]` `fromTotal`'a göre sıralıdır; öğedeki `fromTotal` = `offers[0].fromTotal`. Bu teklif en ucuz gidiş teklifi olmayabilir: ucuz bir gidiş teklifinin uyumlu dönüşleri pahalıysa toplamı daha yüksek olur.
  - `sort=price` `fromTotal` değerine göre sıralar.
- **Gidiş-dönüş, `leg=inbound`:** Yalnızca seçilen gidişle `groupKeys` kesişimi olan dönüş teklifleri listelenir; `offers[].total`, çiftin paketli teklif ve ücret kuralı uygulanmış nihai toplamıdır. Öğenin `total` değeri `offers[0].total`'dır; `sort=price` ve `facets` içindeki `minPrice` bu çift toplamından üretilir. Uyumlu geçerli dönüş kalmamışsa 200 ve boş liste döner; Next.js gidiş seçimine geri döner.
- Uyumlu geçerli dönüşü olmayan gidiş teklifleri listelenmez.
- **Sürüm ve sıralama:** Aynı `resultsVersion` içinde sıralama deterministiktir (seçilen alan, eşitlikte süre, sonra `offerId`). Bir teklifin süresi dolunca teklif çıkarılır; fiyatlar, sıralama ve facet'ler yeniden hesaplanır ve `resultsVersion` artar ([ayrıntı](design-details.md#sürümlü-sonuçlar-ve-sayfa-tutarlılığı)).
- `facets` filtrelenmemiş kümeden hesaplanır. `minPrice`: tek yönde `lowestPrice`, `leg=outbound`'da `fromTotal`, `leg=inbound`'da seçilen gidişle oluşan çiftin `total` değeri cinsindendir.
- Havayolu filtresi ve gösterimi pazarlayan havayoluna göredir; operatör farklıysa `operatingAirline` ayrıca döner.

**202 Accepted** (henüz hazır değil). Başlık: `Retry-After: 1`

```json
{ "searchId": "7b2e4d90-1c5a-4f3b-9e6d-0a8c2f5b7d14", "status": "PENDING" }
```

**200 OK.** Bölüm 3'teki gidiş-dönüş araması (2 yetişkin, 1 çocuk), gidiş bacağı:

```json
{
  "searchId": "7b2e4d90-1c5a-4f3b-9e6d-0a8c2f5b7d14",
  "status": "COMPLETED",
  "expiresAt": "2026-10-13T09:44:00Z",
  "resultsVersion": 1,
  "leg": "outbound",
  "page": 1,
  "pageSize": 20,
  "totalItems": 33,
  "totalPages": 2,
  "sort": { "by": "price", "order": "asc" },
  "appliedFilters": { "stops": [0] },
  "items": [
    {
      "flightId": "fl_6d0b3e8a21",
      "departure": { "airport": "SAW", "city": "İstanbul", "localTime": "2026-11-20T09:35:00" },
      "arrival": { "airport": "AYT", "city": "Antalya", "localTime": "2026-11-20T10:50:00" },
      "durationMinutes": 75,
      "stops": 0,
      "airline": { "code": "PC", "name": "Pegasus" },
      "operatingAirline": null,
      "segments": [
        {
          "flightNumber": "PC2009",
          "from": "SAW",
          "to": "AYT",
          "departureLocalTime": "2026-11-20T09:35:00",
          "arrivalLocalTime": "2026-11-20T10:50:00",
          "durationMinutes": 75
        }
      ],
      "fromTotal": { "amount": "12405.00", "currency": "TRY" },
      "offers": [
        {
          "offerId": "of_5d2e8b71c3",
          "brand": "Basic",
          "legPrice": { "amount": "5835.00", "currency": "TRY" },
          "fromTotal": { "amount": "12405.00", "currency": "TRY" },
          "checkedBaggage": { "quantity": 15, "unit": "KG" }
        },
        {
          "offerId": "of_5d2e8b71c4",
          "brand": "Promo",
          "legPrice": { "amount": "5510.00", "currency": "TRY" },
          "fromTotal": { "amount": "12960.00", "currency": "TRY" },
          "checkedBaggage": null
        },
        {
          "offerId": "of_5d2e8b71c5",
          "brand": "Essentials",
          "legPrice": { "amount": "7020.00", "currency": "TRY" },
          "fromTotal": { "amount": "13590.00", "currency": "TRY" },
          "checkedBaggage": { "quantity": 20, "unit": "KG" }
        }
      ]
    }
  ],
  "facets": {
    "airlines": [
      { "code": "PC", "name": "Pegasus", "count": 22, "minPrice": { "amount": "12405.00", "currency": "TRY" } },
      { "code": "TK", "name": "Turkish Airlines", "count": 19, "minPrice": { "amount": "13280.00", "currency": "TRY" } }
    ],
    "stops": [
      { "value": 0, "count": 33, "minPrice": { "amount": "12405.00", "currency": "TRY" } },
      { "value": 1, "count": 8, "minPrice": { "amount": "14010.00", "currency": "TRY" } }
    ],
    "departureTimeRanges": [
      { "from": "00:00", "to": "06:00", "count": 4 },
      { "from": "06:00", "to": "12:00", "count": 17 },
      { "from": "12:00", "to": "18:00", "count": 12 },
      { "from": "18:00", "to": "24:00", "count": 8 }
    ],
    "price": {
      "min": { "amount": "12405.00", "currency": "TRY" },
      "max": { "amount": "31870.00", "currency": "TRY" }
    }
  }
}
```

Örnekteki ilk teklifin `fromTotal` değeri: gidiş 5.835 TRY + en ucuz uyumlu dönüş 6.120 TRY (paketli değil) + ücret anahtarları aynı olduğu için gidiş-dönüş ücretlerinin büyüğü 450 TRY = 12.405 TRY. `Promo` teklifinin gidişi daha ucuzdur (5.510 TRY), ama uyumlu dönüşleri pahalı olduğu için toplamı daha yüksektir ve ikinci sıradadır.

`facets` filtrelenmemiş kümeden hesaplandığı için `stops` facet'inde 1 aktarmalı uçuşlar da görünür; `totalItems` ise uygulanan `stops=0` filtresinden sonraki sayıdır.

Uçuş bulunamazsa 200 döner: `items: []`, `totalItems: 0`.

**409 Conflict** (istemcinin `resultsVersion` değeri eski). Next.js listeyi birinci sayfadan yeniden yükler:

```json
{
  "type": "/problems/results-changed",
  "title": "Sonuçlar güncellendi",
  "status": 409,
  "code": "RESULTS_CHANGED",
  "detail": "Bazı tekliflerin süresi doldu; liste birinci sayfadan yeniden yüklenmeli.",
  "traceId": "9f3c2a71be0d4e58a6c1f04d2b7e8a35",
  "resultsVersion": 2
}
```

**410 Gone** (arama tamamlanmış ve `expiresAt` geçmiş; kriterler arama kaydından döner). `Content-Type: application/problem+json`

```json
{
  "type": "/problems/search-expired",
  "title": "Arama sonucunun süresi doldu",
  "status": 410,
  "code": "SEARCH_EXPIRED",
  "detail": "Fiyatlar güncelliğini yitirdi; aynı kriterlerle yeniden arama yapılmalı.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "criteria": {
    "tripType": "ROUND_TRIP",
    "origin": "CITY:IST",
    "destination": "AIRPORT:AYT",
    "departureDate": "2026-11-20",
    "returnDate": "2026-11-27",
    "passengers": { "adults": 2, "children": 1, "infants": 0 },
    "cabin": "ECONOMY",
    "currency": "TRY"
  }
}
```

## 5. Teklif detayı

`GET /v1/flights/searches/{searchId}/offers/{offerId}`

Gidiş-dönüşte nihai toplamın hesaplanabilmesi için seçilen dönüş teklifi de gönderilir: `?inboundOfferId=of_...`. Bu parametre olmadan yalnızca gidiş bacağının bilgisi döner.

| Parametre | Açıklama |
|---|---|
| `inboundOfferId` | Gidiş-dönüşte seçilen dönüş teklifi. Verilmezse yalnızca gidiş bacağı döner, nihai toplam hesaplanmaz |
| `refresh` | `true` ise teklif TourVisio `getoffers` ile tazelenir. Next.js bunu yalnızca kullanıcı "Devam" dediğinde gönderir; her detay görüntülemede çağırmak kotayı tüketir |

Akış:

1. Sahiplik ve geçerlilik kontrol edilir.
2. Snapshot'taki teklif okunur. `refresh` yoksa yanıt buradan döner: `priceVerified: false`, `freshness.source: SNAPSHOT`. Bu, arama anındaki fiyattır ve listedeki fiyatla aynıdır.
3. `refresh=true` ise TourVisio `getoffers` çağrılır. Bu çağrı başarısız olursa snapshot verisi `priceVerified: false` ve `freshness.source: SNAPSHOT` ile döner; istek hata vermez.
4. Fiyat snapshot'takinden farklıysa yeni fiyat, `priceChanged: true` ve önceki fiyatla birlikte döner. Listeleme snapshot'ı değişmez.

**"Devam" adımında fiyatın kesinleşmesi:** `priceVerified: false` olsa da kullanıcı yolcu bilgisi adımına geçebilir, çünkü bu adımda para çekilmez. Ödeme öncesi son fiyat kontrolü rezervasyon fazında `BeginTransaction` yanıtıyla yapılır; bu fiyatın işlem süresi boyunca garanti olup olmadığı SAN TSG'ye sorulacak ([açık soru 8](../README.md#tourvisioya-açık-sorular)). Bu fiyat gösterilenden farklıysa kullanıcıya yeni fiyat gösterilir ve ödemeye geçmeden önce onayı alınır. `getoffers`'ın güncel fiyat döndürdüğü ise SAN TSG'den teyit edilecek ([açık soru 9](../README.md#tourvisioya-açık-sorular)); teyit gelmezse `priceVerified` her zaman `false` olur.

**200 OK.** README'deki tek yön araması (`3f6c1e8a-...`, 2 yetişkin), listede 3.890 TRY görünen teklif, `refresh=true` ile:

```json
{
  "offerId": "of_8c1f2a9d4e",
  "flightId": "fl_2c9e71a04b",
  "expiresAt": "2026-10-13T09:29:00Z",
  "availability": "AVAILABLE",
  "seatsLeft": 8,
  "brand": { "name": "Basic" },
  "price": {
    "total": { "amount": "3890.00", "currency": "TRY" },
    "fare": { "amount": "3590.00", "currency": "TRY" },
    "fees": { "amount": "300.00", "currency": "TRY" },
    "breakdown": [
      {
        "passengerType": "ADULT",
        "count": 2,
        "fare": { "amount": "3590.00", "currency": "TRY" },
        "tax": { "amount": "1052.40", "currency": "TRY" }
      }
    ],
    "priceChanged": false,
    "previousTotal": null,
    "priceVerified": false
  },
  "baggage": [
    { "passengerType": "ADULT", "type": "CHECKED", "quantity": 15, "unit": "KG", "pieces": 1 },
    { "passengerType": "ADULT", "type": "CABIN", "quantity": 8, "unit": "KG", "pieces": 1 }
  ],
  "services": ["Checked baggage allowance 15 kg", "Carry on baggage allowance 8 kg"],
  "fareRules": { "available": false, "reason": "AVAILABLE_AT_BOOKING" },
  "freshness": { "source": "SUPPLIER_REFRESH", "refreshedAt": "2026-10-13T09:18:41Z" }
}
```

- `price.total` = `fare` + `fees`. `tax`, `fare`'in içindedir; kırılım TourVisio'da yoksa uydurulmaz, `breakdown` boş döner. `fees` alanı dönmeyen teklifler listelenmez ([ücret kuralı](tourvisio-mapping.md#toplam-fiyat-hesaplama)).
- `priceVerified` iki koşul birlikte sağlandığında `true` olur: `refresh=true` çağrısı başarılıdır ve SAN TSG, `getoffers`'ın güncel fiyat döndürdüğünü teyit etmiştir ([açık soru 9](../README.md#tourvisioya-açık-sorular)). Teyit gelene kadar örnekteki gibi tazeleme başarılı olsa bile değer `false`'tur.
- `freshness.source`: `SUPPLIER_REFRESH` (`getoffers` ile tazelendi) veya `SNAPSHOT` (arama anındaki veri).
- `availability`: `AVAILABLE` veya `ON_REQUEST`. `ON_REQUEST` teklifler sağlayıcı onayı gerektirir (24 saate kadar sürebilir) ve kullanıcıya ayrıca gösterilir ([açık soru 11](../README.md#tourvisioya-açık-sorular)).
- `services`, sağlayıcının döndürdüğü metinlerdir; ileride çeviri tablosuyla yerelleştirilebilir.
- `fareRules`: Kural metni şimdilik rezervasyon adımında alınır ([açık soru 8](../README.md#tourvisioya-açık-sorular)).

**410 Gone**

```json
{
  "type": "/problems/offer-expired",
  "title": "Teklifin süresi doldu",
  "status": 410,
  "code": "OFFER_EXPIRED",
  "detail": "Bu teklif artık geçerli değil; aynı kriterlerle yeniden arama yapılmalı.",
  "traceId": "5b8aa5a2d2c872e8321cf37308d69df2"
}
```
