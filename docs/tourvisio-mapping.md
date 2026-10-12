# TourVisio eşlemesi

[← README](../README.md)

Bu doküman, `TourVisioAdapter`'ın bizim sözleşmemizle TourVisio arasında yaptığı dönüşümleri listeler. Kaynak: [docs.santsg.com/tourvisio](https://docs.santsg.com/tourvisio/) (Authentication, Enumerations, Flight Product, Flight Service, Booking).

## Endpoint eşlemesi

| Bizim endpoint | TourVisio metodu | Adres | Token |
|---|---|---|---|
| (iç işlem) | Login | `POST /api/authenticationservice/login` | Hayır |
| `GET /v1/flights/locations/departures` | Departure Auto Complete | `POST /api/productservice/getdepartureautocomplete` | Evet |
| `GET /v1/flights/locations/arrivals` | Arrival Auto Complete | `POST /api/productservice/getarrivalautocomplete` | Evet |
| `POST /v1/flights/searches` | Price Search | `POST /api/productservice/pricesearch`; performans için opsiyonel: `POST /api/productservice/flightpricesearch` | Dokümana göre hayır ([açık soru 2](../README.md#tourvisioya-açık-sorular)); yine de her çağrıda gönderilir |
| `GET .../results` | Yok | Snapshot'tan okunur | |
| `GET .../offers/{offerId}` | Get Offers (opsiyonel) | Varsayılan snapshot; yalnızca `refresh=true` ile `POST /api/productservice/getoffers` (`productType: 3`, `searchId`, `offerIds[]`) | Evet |
| Rezervasyon fazı | GetFareRules | `POST /api/flightservice/getfarerules` (`transactionId` veya `reservationNumber` ister) | Evet |
| Rezervasyon fazı | BeginTransaction | `POST /api/bookingservice/begintransaction` (`offerIds` ile) | Evet |

## Enum eşlemesi

**Yolculuk tipi** (TourVisio'da string)

| Bizim değer | `ServiceType` / `ServiceTypes` |
|---|---|
| `ONE_WAY` | `"1"` |
| `ROUND_TRIP` | `"2"` |
| `MULTI_CITY` (kapsam dışı) | `"3"` |

**Lokasyon tipi.** Autocomplete yanıtındaki tip ile arama isteğindeki tip farklı enum'lardır:

| Autocomplete yanıtı (Autocomplete Response Type) | Bizim tip | Arama isteği (Location Type) |
|---|---|---|
| `1` City | `CITY` | `2` |
| `3` Airport | `AIRPORT` | `5` |
| Diğerleri (Country, Town, Village vb.) | Listelenmez | |

**Yolcu tipi**

| Bizim alan | `Passengers[].type` |
|---|---|
| `adults` | `1` (Adult) |
| `children` | `2` (Child) |
| `infants` | `3` (Infant) |

Sayısı 0 olan yolcu tipleri isteğe eklenmez.

**Kabin**

| Bizim değer | `flightClasses` |
|---|---|
| `ECONOMY` | `[0, 1]` (PROMO + ECONOMY; PROMO'nun ekonomi kabinine ait olduğu varsayıldı, [açık soru 15](../README.md#tourvisioya-açık-sorular)) |
| `BUSINESS` | `[2]` |

**Diğer**

| TourVisio | Bizim değer |
|---|---|
| Baggage Type `1` / `2` | `CHECKED` / `CABIN` |
| Baggage Unit Type `0` / `1` / `2` | `KG` / `LB` / `PIECE` |
| FlightRoute Type `1` / `2` | `outbound` / `inbound` |
| Message Type `1` (Error) | Hata |
| Message Type `3` (Information), `4` (Warning) | Loglanır, akış devam eder |

## Price search istek eşlemesi

| TourVisio alanı | Değer | Not |
|---|---|---|
| `ProductType` | `3` | Flight |
| `ServiceTypes` | `["1"]` veya `["2"]` | `tripType`'tan |
| `CheckIn` | `departureDate` | |
| `Night` | `returnDate` − `departureDate` (gün) | Yalnızca gidiş-dönüşte; zaman damgasıyla değil takvim tarihleriyle hesaplanır |
| `DepartureLocations` | `[{ "id": "IST", "type": 2 }]` | `origin`'den, lokasyon tipi eşlemesiyle |
| `ArrivalLocations` | `[{ "id": "AYT", "type": 5 }]` | `destination`'dan |
| `Passengers` | `[{ "type": 1, "count": 2 }]` | |
| `flightClasses` | Kabin eşlemesi | |
| `supportedFlightReponseListTypes` | `[3]` | Gidiş-dönüşte; alan adı dokümandaki yazımıyla |
| `showOnlyNonStopFlight` | `false` | Aktarma filtresi bizim tarafımızda uygulanır |
| `acceptPendingProviders` | `false` | Doküman örneğindeki değer |
| `forceFlightBundlePackage` | `false` | Doküman örneğindeki değer |
| `disablePackageOfferTotalPrice` | `true` | Doküman örneğindeki değer |
| `calculateFlightFees` | `true` | `fees` alanı yalnızca bu değerle döner; nihai fiyat hesabı için gerekli. Performans etkisi [açık soru 12](../README.md#tourvisioya-açık-sorular) |
| `Culture` | `Accept-Language` | `tr-TR` varsayılan |
| `Currency` | `currency` | `TRY` varsayılan |

## Price search yanıt eşlemesi

| TourVisio | Bizim model | Not |
|---|---|---|
| `header.requestId` | Log ve trace attribute'u | Sağlayıcıyla destek yazışmasındaki referans |
| `header.success`, `header.messages[]` | Hata eşlemesi (aşağıda) | |
| `body.searchId` | Snapshot içinde `supplierSearchId` | Dışarı verilmez |
| `body.expiresOn` | Snapshot'ın `expiresAt` değeri | `expiresAt` = min(`expiresOn` − 60 sn, oluşturma + 15 dk), mutlak ve sabit; saat dilimi [açık soru 5](../README.md#tourvisioya-açık-sorular) |
| `flights[].items[]` | Uçuş bacağı | `route`: 1 gidiş, 2 dönüş |
| `flights[].items[].segments[]` | `segments[]` | |
| `flights[].items[].marketingAirline` | `airline` | Yoksa `airline` kullanılır |
| `flights[].items[].airline` | `operatingAirline` | Pazarlayan havayolundan farklıysa doldurulur |
| `flights[].items[].duration` | `durationMinutes` | |
| `flights[].items[].stopCount` | `stops` | |
| `offers[].offerId` (tek yön) | Snapshot içinde `supplierOfferIds` | Dışarıya kısa ID (`of_...`) verilir |
| `offers[].offerIds[]` (`groupKey`, `offerId`) (gidiş-dönüş) | Aynı | Gidiş-dönüşte alan şekli farklıdır, adaptör iki şekli de işler |
| `offers[].groupKeys` | Gidiş ve dönüş eşleşmesi | Kesişim kuralı aşağıda |
| `offers[].price` | `price.fare` | Nihai `price.total` aşağıdaki kuralla hesaplanır |
| `offers[].priceBreakDown.items[]` | `price.breakdown[]` | `passengerType`, `passengerCount`, `price`, `airportTax` |
| `offers[].serviceFee` | Nihai fiyata dahil | |
| `offers[].fees` (`key`, `oneWay.price`, `roundTrip.price`) | `price.fees` | Toplam fiyat kuralında kullanılır (aşağıda) |
| `offers[].isPackageOffer` | Toplam fiyat kuralında kullanılır | |
| `offers[].seatInfo.availableSeatCount` | `seatsLeft` | |
| `offers[].flightBrandInfo.name` | `brand.name` | |
| `offers[].baggageInformations[]` | `baggage[]` | Yolcu tipine göre |
| `offers[].services[].name` | `services[]` | |
| `offers[].expiresOn` | Teklifin `expiresAt` değeri | |
| `offers[].reservableInfo.reservable` | `false` ise teklif listelenmez | |

## Toplam fiyat hesaplama

Kaynak: [Calculating the Total Price](https://docs.santsg.com/tourvisio/flightproduct/#calculating-the-total-price). Listede ve detayda gösterilen her fiyat bu kuralla hesaplanmış nihai tutardır.

```text
function total(outbound, inbound?):
  if inbound yok:                                   // tek yön
    return outbound.price + outbound.fees.oneWay.price

  fare = outbound.isPackageOffer
         ? max(outbound.price, inbound.price)        // paketli: yüksek olan alınır
         : outbound.price + inbound.price

  if outbound.fees.key == inbound.fees.key:
    fees = max(outbound.fees.roundTrip.price, inbound.fees.roundTrip.price)
  else:
    fees = outbound.fees.oneWay.price + inbound.fees.oneWay.price

  return fare + fees
```

- Hesap decimal tipte yapılır; float kullanılmaz.
- Paketli olmayan teklifte fiyatların toplanması dokümandaki kuralın doğal sonucudur. Paketli teklifin hangi dönüşlerle eşleşebileceği [açık soru 7](../README.md#tourvisioya-açık-sorular)'dedir.
- `calculateFlightFees: true` gönderildiği hâlde bir teklifte `fees` alanı yoksa, ücret **bilinmiyor** kabul edilir ve teklif listelenmez. Ücret yokmuş gibi davranmak yanlış düşük fiyat gösterme riski taşır; teklifi gizlemek ise yalnızca seçenek kaybıdır. Bu tekliflerin oranı metrik olarak izlenir. SAN TSG, alanın yokluğunun "ücret yok" anlamına geldiğini teyit ederse ([açık soru 12](../README.md#tourvisioya-açık-sorular)), bu teklifler bir konfigürasyonla ücret 0 olarak listelenir.
- Gidiş listesindeki `fromTotal`, her gidiş için uyumlu dönüşler arasında bu fonksiyonun en küçük değeridir. Snapshot yazılırken hesaplanır; bir teklifin süresi dolduğunda geçerli tekliflerden yeniden hesaplanır ([ayrıntı](design-details.md#2-arama-yaşam-döngüsü)). Gidiş ve dönüş sayısı birkaç yüzü geçmediği için maliyeti düşüktür.

## Gidiş-dönüş eşleşmesi

Dokümandaki örnek koda göre:

1. Seçilen gidiş teklifinin `groupKeys` değerleriyle kesişimi olan dönüş teklifleri uyumludur.
2. Rezervasyon için her iki tekliften, kesişimdeki `groupKey`'e karşılık gelen `offerIds[].offerId` değeri seçilir.
3. Bu iki `offerId`, ileride `BeginTransaction` (`offerIds`) çağrısına verilecek değerlerdir.

## TourVisio istek örnekleri

**Gidiş-dönüş arama.** Bizim `POST /v1/flights/searches` isteğimizin ([api-contract.md](api-contract.md#3-uçuş-arama)) adaptördeki karşılığı: `POST /api/productservice/pricesearch`

```json
{
  "ProductType": 3,
  "ServiceTypes": ["2"],
  "CheckIn": "2026-11-20",
  "Night": 7,
  "DepartureLocations": [{ "id": "IST", "type": 2 }],
  "ArrivalLocations": [{ "id": "AYT", "type": 5 }],
  "Passengers": [
    { "type": 1, "count": 2 },
    { "type": 2, "count": 1 }
  ],
  "flightClasses": [0, 1],
  "supportedFlightReponseListTypes": [3],
  "showOnlyNonStopFlight": false,
  "acceptPendingProviders": false,
  "forceFlightBundlePackage": false,
  "disablePackageOfferTotalPrice": true,
  "calculateFlightFees": true,
  "Culture": "tr-TR",
  "Currency": "TRY"
}
```

**Teklif tazeleme** (yalnızca `refresh=true`): `POST /api/productservice/getoffers`

```json
{
  "productType": 3,
  "searchId": "<snapshot'taki TourVisio searchId>",
  "offerIds": ["<snapshot'taki TourVisio offerId>"],
  "currency": "TRY",
  "culture": "tr-TR"
}
```

Örnek istek, dokümandaki Get Offers örneği temel alınarak yazıldı; entegrasyon testinde doğrulanacak.

## Hata eşlemesi

| TourVisio durumu | Bizim yanıt |
|---|---|
| HTTP 2xx, `success=true` | Normal akış (`Warning` mesajları loglanır) |
| HTTP 2xx, `success=false` (`messageType=1`) | 502 `SUPPLIER_ERROR`; mesaj istek doğrulamasıyla ilgiliyse 422 |
| HTTP 401 | Karşılaştırmalı geçersizleştirme: token yalnızca başarısız çağrıda kullanılan token hâlâ kayıtlıysa silinir. Token zaten yenilenmişse yeni token kullanılır, değilse kilitle tek login yapılır. Çağrı bir kez tekrarlanır; yine başarısızsa 503 ([ayrıntı](design-details.md#tourvisio-tokenı-ve-eşzamanlı-401)) |
| HTTP 429 veya kota mesajı | 503 `SUPPLIER_UNAVAILABLE` + `Retry-After` |
| HTTP 5xx | Retry bütçesi içinde 1 tekrar, sonra 502 |
| Timeout | 504 `SUPPLIER_TIMEOUT` (arama tekrarlanmaz) |
| Circuit açık | 503 `SUPPLIER_UNAVAILABLE` |

## Dokümandaki tutarsızlıklar

Adaptörün bu noktalarda toleranslı olması ve entegrasyon testlerinin bunları kapsaması gerekir:

- `pricesearch` için "Token Required: NO" yazıyor; diğer tüm metotlarda YES.
- Uçuş saatleri tek yön örneğinde saat dilimi eki olmadan, gidiş-dönüş örneğinde `Z` ekiyle geliyor. `expiresOn` ile `responseTime` arasında tüm örneklerde 3 sa 30 dk fark var.
- İstek alanlarının yazımı örnekler arasında farklı (`ProductType` / `productType`, `Currency` / `currency`). Adaptör tek bir yazım kullanır; entegrasyon testinde doğrulanır.
- `supportedFlightReponseListTypes` alan adı dokümanda bu yazımla geçiyor; adaptör de birebir aynı yazımı kullanır.
- Tek yön yanıtında `offers[].offerId`, gidiş-dönüş yanıtında `offers[].offerIds[]` kullanılıyor.
