# Sepet Rotası: Market Fiyatı'na eklenebilirlik (entegrasyon şeması)

Bu belge, prototipteki özelliklerin Market Fiyatı uygulamasına eklenebilmesi için **veri biçimlerini** ve **entegrasyon seçeneklerini** anlatır. Aşağıdaki JSON örnekleri tahmin değil, prototipin kendi kodunun ürettiği gerçek çıktılardır (fiyatlar demo).

## 1. Şu anki durum (dürüstçe)
- Prototip tek bir HTML dosyasıdır; ekran, mantık ve demo veri aynı dosyadadır.
- Mantık zaten ayrı işlevlerdir (listeyi okuma, paketlere çevirme, sepet karşılaştırma, fark denetimi) ve girdi/çıktısı JSON'a çevrilebilir. Ama henüz ayrı bir modüle bölünmedi. Uygulamaya gömmeden önce bu ayrıştırma yapılmalıdır.
- Veri demo: ürünler, boyutlar, fiyatlar, teslimat ücretleri ve kampanyalar örnektir. Fotoğraftan liste okuma henüz gerçek değil.

## 2. Üç katman
| Katman | Görevi | Prototipte | Uygulamada |
|---|---|---|---|
| Ekran | Liste yazma, kontrol, uyarılar, sonuç | HTML/CSS/JS | Uygulamanın kendi ekranı ya da WebView |
| Mantık | Yazıyı çözme, miktarı market paketine çevirme, sepetleri kurma, yazılanla seçileni karşılaştırma | Aynı dosyada işlevler | Servis veya paylaşılan modül |
| Veri bağdaştırıcı | Ürün ve fiyat kaynağı | Sentetik katalog | Market Fiyatı verisi |

Katalog değişince yalnızca üçüncü katman değişir, ekran ve mantık aynı kalır.

## 3. Veri biçimleri
### 3.1 İstek (uygulamadan servise)
```json
{
  "list": [
    "2 süt",
    "1 kg çiçek balı",
    "açık beyaz peynir 300 gr",
    "2 maka"
  ],
  "budget": 700,
  "brandPreference": null,
  "location": {
    "lat": 41.01,
    "lon": 28.97,
    "distanceKm": 5
  },
  "includeOnline": true
}
```
Not: Prototip konumu ve marka tercihini henüz kullanmıyor (konum ileride mağaza şubesini seçecek).

### 3.2 Katalog kaydı (Market Fiyatı'ndan beklenen)
Bir çeşit (örnek: Muzlu Süt) ve ürünleri:
```json
{
  "variantKey": "süt-aroma-muz",
  "groupKey": "süt",
  "displayName": "Muzlu Süt",
  "saleMode": "packaged",
  "unitFamily": "L",
  "products": [
    {
      "name": "Sütaş Muzlu Süt 180 ml",
      "netQuantity": {
        "value": 0.18,
        "unit": "L"
      },
      "prices": [
        {
          "market": "Migros",
          "channel": "store",
          "price": 15.5,
          "campaign": false
        },
        {
          "market": "Migros",
          "channel": "online",
          "price": 16.5,
          "campaign": false
        },
        {
          "market": "A101",
          "channel": "store",
          "price": 14,
          "campaign": false
        }
      ]
    },
    {
      "name": "Sütaş Muzlu Süt 200 ml",
      "netQuantity": {
        "value": 0.2,
        "unit": "L"
      },
      "prices": [
        {
          "market": "Migros",
          "channel": "store",
          "price": 15.5,
          "campaign": false
        },
        {
          "market": "Migros",
          "channel": "online",
          "price": 14,
          "campaign": true
        },
        {
          "market": "A101",
          "channel": "store",
          "price": 14,
          "campaign": false
        }
      ]
    }
  ]
}
```
Gerekli alanlar: ürün adı, net miktar (değer + birim), çeşit ve grup anahtarı, satış şekli (paketli / açık), market, kanal (mağaza / online), fiyat, kampanya. Barkod varsa eşleştirme kolaylaşır.

### 3.3 Çözülmüş liste satırı (servisten uygulamaya, 1. çıktı)
Durum değerleri: `resolved` (çözüldü), `needs_variant` (çeşit sorulacak), `unrecognized` (anlaşılamadı, öneri var), `not_found`.
```json
[
  {
    "raw": "2 süt",
    "status": "resolved",
    "groupKey": "süt",
    "variantKey": "süt-tam",
    "saleMode": "packaged",
    "request": {
      "packs": 2,
      "packSize": {
        "value": 1,
        "unit": "L"
      }
    },
    "typedAmountBase": null,
    "suggestions": null,
    "warnings": []
  },
  {
    "raw": "1 kg çiçek balı",
    "status": "resolved",
    "groupKey": "bal",
    "variantKey": "bal-çiçek",
    "saleMode": "packaged",
    "request": {
      "packs": 2,
      "packSize": {
        "value": 0.46,
        "unit": "kg"
      }
    },
    "typedAmountBase": {
      "value": 1,
      "unit": "kg"
    },
    "suggestions": null,
    "warnings": []
  },
  {
    "raw": "açık beyaz peynir 300 gr",
    "status": "resolved",
    "groupKey": "peynir",
    "variantKey": "peynir-beyaz-açık",
    "saleMode": "open",
    "request": {
      "amount": {
        "value": 300,
        "unit": "gr"
      }
    },
    "typedAmountBase": null,
    "suggestions": null,
    "warnings": []
  },
  {
    "raw": "2 maka",
    "status": "unrecognized",
    "groupKey": null,
    "variantKey": null,
    "saleMode": null,
    "request": null,
    "typedAmountBase": null,
    "suggestions": [
      "Makarna"
    ],
    "warnings": [
      "\"2 Maka\" tam anlaşılamadı. Şunlardan birini mi kastettin? Makarna. Seç ya da yazımı düzelt."
    ]
  }
]
```

### 3.4 Karşılaştırma sonucu (2. çıktı)
Teslimat bilgisi (online kanallar için):
```json
[
  {
    "market": "Migros",
    "channel": "online",
    "deliveryFee": 29.9,
    "minBasket": 200,
    "freeAbove": 600
  },
  {
    "market": "CarrefourSA",
    "channel": "online",
    "deliveryFee": 24.9,
    "minBasket": 250,
    "freeAbove": 750
  }
]
```
Sepet (kısaltılmış, ilk 2 satır):
```json
{
  "id": "mixall",
  "title": "Karma sepet, tüm kanallar",
  "type": "karma",
  "total": 601.3,
  "itemsTotal": 576.4,
  "fees": [
    {
      "channel": "CarrefourSA online",
      "deliveryFee": 24.9,
      "minBasket": 250,
      "minBasketMet": false
    }
  ],
  "lines": [
    {
      "requested": "2 süt",
      "product": "Torku Tam Yağlı Süt 1 L",
      "channel": {
        "market": "CarrefourSA",
        "type": "online"
      },
      "packs": 2,
      "packSize": {
        "value": 1
      },
      "packPrice": 37,
      "unitPrice": {
        "value": 37,
        "per": "L"
      },
      "lineTotal": 74,
      "campaign": true,
      "askText": "2 adet (1 L)"
    },
    {
      "requested": "yarım yağlı süt",
      "product": "Pınar Yarım Yağlı Süt 1 L",
      "channel": {
        "market": "Şok",
        "type": "store"
      },
      "packs": 1,
      "packSize": {
        "value": 1
      },
      "packPrice": 37,
      "unitPrice": {
        "value": 37,
        "per": "L"
      },
      "lineTotal": 37,
      "campaign": false,
      "askText": "1 adet (1 L)"
    }
  ],
  "missing": []
}
```
Sepet türleri: kanal başına tek market sepeti, "Karma sepet, tüm kanallar", "Karma sepet, yalnızca mağazalar". Her satırda istenen metin, alınacak ürün, paket sayısı, birim fiyat ve satır toplamı vardır; böylece "neden bu ürün, bu markette" açıklanabilir.

### 3.5 Kullanıcı tercihi (kaydedilen seçim)
Kullanıcının seçimini hatırlamak için örnek kayıt:
```json
{
  "phrase": "süt",
  "choice": {
    "variantKey": "süt-tam",
    "packSize": {
      "value": 1,
      "unit": "L"
    }
  },
  "updatedAt": "2026-10-14T12:00:00Z"
}
```
Prototipte favori listeler tarayıcıda tutuluyor; uygulamada hesaba bağlı saklama gerekir.


### 3.6 Sipariş geçmişi (favori listenin son 3 ödemesi)
Favori liste başına son 3 sipariş tutulur. Online siparişte tutar son adımda otomatik yazılır, markette alışverişte fiş ya da ekran görüntüsünden okunan tutar kullanıcı onayıyla yazılır.
```json
{
  "favorite": "Haftalık market",
  "orders": [
    {"date": "2026-10-02", "total": 620.00, "source": "online", "market": "Migros Online"},
    {"date": "2026-10-09", "total": 645.00, "source": "online", "market": "Migros Online"},
    {"date": "2026-10-12", "total": 690.50, "source": "store", "market": "Migros"}
  ]
}
```
Online siparişte onaylanan ürünler de siparişle saklanır (marka dahil), böylece favoride "geçen sefer hangi markayı aldım" görünür:
```json
{
  "date": "2026-10-09", "total": 645.00, "source": "online", "market": "Migros Online",
  "lines": [
    {"requested": "2 tam yağlı süt", "product": "Sütaş Tam Yağlı Süt 1 L", "brand": "Sütaş", "packs": 2, "lineTotal": 82.00, "market": "Migros Online"},
    {"requested": "dana kıyma 500 gr", "product": "Dana Kıyma 500 g", "brand": null, "packs": 1, "lineTotal": 262.00, "market": "Migros Online"}
  ]
}
```
Marka, ürün adındaki ilk kelimeden çıkarılır; markasız ürünlerde (dana kıyma, kuşbaşı, açık ürünler) `brand` boştur. Market Fiyatı'nın gerçek verisinde marka ayrı bir alan olarak geliyorsa o kullanılmalıdır.
Not: Market Fiyatı satış yapmadığı için "ödenen" tutar uygulamada iki yoldan gelir: kullanıcının market sitesine geçmeden önce gördüğü sepet toplamı (tahmin) ve mağaza fişi (gerçek). Gerçek online ödeme ancak market tarafından geri bildirilirse kesinleşir.

## 4. Uç nokta taslağı (öneri, henüz yok)
| İstek | İş |
|---|---|
| `POST /v1/list/parse` | Metin veya fotoğraf alır, 3.3 biçiminde satırlar döner (belirsizler `needs_variant`). |
| `POST /v1/baskets/compare` | Onaylanmış satırlar, bütçe, konum alır; 3.4 biçiminde sepetler döner. |
| `GET/PUT /v1/preferences` | 3.5 biçiminde tercihleri okur ve yazar. |
| `POST /v1/favorites/{id}/orders` | 3.6 biçiminde sipariş kaydı ekler (son 3 tutulur). Fiş görseli gelirse sunucu tutarı okur ve onaya sunar. |

## 5. Uygulamaya gömme seçenekleri
1. **WebView ekranı:** En hızlısı. Gereken: HTTPS adresi, konum ve kamera izninin WebView'a geçirilmesi, yazı tipinin sayfa içine gömülmesi (şu an Google yazı tipi yedekli), tarayıcı depolamasının uygulama içinde kalıcı olması.
2. **Yerel ekran + servis:** Uygulama 3.1-3.4 biçimlerini kullanır, mantık sunucuda çalışır. En sağlamı, en çok iş.
3. **Yalnızca servis:** Mevcut Market Fiyatı ekranları 3.4 sonucunu kendi tasarımında gösterir.

## 6. Market Fiyatı verisiyle eşleştirme
- **Bilinen** (topluluk MCP aracından): anahtar kelimeyle arama, barkod veya kimlikle arama, konum ve yarıçap (km). Bu, 3.2 katalog kaydı için ürün adı ve fiyatı sağlar.
- **Doğrulanmadı:** Market Fiyatı yanıtının alan adları (net miktar, birim fiyat, şube bilgisi). Kaynak koduna ve siteye erişemedim.
- **Ayrıca gerekecek:** Ürün adından net miktarı ve birimi ayıran küçük bir eşleme katmanı ("Pınar Süt 1 L" -> süt, 1 L) ve çeşit/grup sınıflaması.
- Online kanal, teslimat ücreti ve kampanya bilgisi Market Fiyatı'nda olmayabilir; olmazsa demo kalır ve sunumda öyle belirtilmelidir.

## 7. BİLGEM'den istenecekler
1. 50-100 ürünlük örnek liste: ad, barkod, net miktar ve birim, kategori, market, şube, fiyat, tarih.
2. Yanıt alan adlarını gösteren örnek bir kayıt veya şema belgesi.
3. Kullanım izni ve sınırları (servis olarak çağırma, önbellekleme).
4. Varsa online kanal, teslimat ve kampanya verisi.
5. Uygulamaya ekleme yöntemi (WebView mi, yerel mi) hakkında teknik görüş.

## 7b. Fiş okuma ve uygulama
Prototipte fiş ve ekran görüntüsü okuma, sayfanın içinden yapay zekâ servisinin görüntü yeteneğiyle yapılır; bu yalnızca yapay zekâ servisinin kendi görüntüleyicisinde çalışır. Market Fiyatı uygulamasının içindeki bir WebView'da bu yetenek yoktur, orada düğme "bu görünümde kapalı" der. Uygulamada aynı özellik için fotoğrafı sunucuya gönderen bir servis gerekir: uygulama kameradan ya da galeriden görseli alır, `POST /v1/favorites/{id}/orders` ile gönderir, sunucu görseli okuyup tutarı, market adını ve tarihi döner, kullanıcı onaylayınca kayıt yazılır. Görselde kişisel veri olabileceği için saklanmaması ve KVKK açısından kullanıcıya bilgi verilmesi gerekir.

## 7c. Favoriden çağırınca marka
Favorinin son siparişindeki markalar (3.6'daki `lines[].brand`) yeni sipariş hazırlanırken tercih olarak uygulanır: önce o marka aranır, o boyda ya da markette yoksa başka marka konur ve kullanıcıya söylenir. Bu yalnızca favori listelerde çalışır.

## 8. Telefonda kullanım (uygulama içi sayfa için)
Dokunma hedefleri en az 44 px, telefonda seçenek butonları tam genişlik, hata sayacı durum çubuğunun altında sabit, favoriler sayfanın başında kapalı bir panel, "Kamerayla çek" doğrudan arka kamerayı açar. Fotoğraf okuma henüz sahte; gerçek okuma için ayrı bir adım gerekir.

## 9. KVKK rıza kaydı (web ve uygulama)

Prototip rızayı cihazda (`as_kvkk_v1`) tutar. BİLGEM uygulamasında ve web sitesinde aynı üç ayrı rıza kullanılmalı, kayıt sunucuda da tutulmalıdır.

```json
{ "v": "2026-10-14-v1", "core": true, "rc": false, "vo": true, "seen": true, "d": "2026-10-14T17:30:00.000Z" }
```

- `core`: favori, sipariş tutarı ve marka saklama. `rc`: fiş, ekran görüntüsü ve liste fotoğrafı okuma. `vo`: sesli liste.
- Öneri uç noktaları: `PUT /v1/consents` (sürüm, seçimler, zaman damgası, kanal: web ya da app), `GET /v1/consents`, `DELETE /v1/me/data` (rıza geri alma ve silme).
- Metin sürümü değişince `v` değişir ve rıza yeniden istenir. Rıza geçmişi kanıt için saklanmalıdır.
- Fiş ve ses işleme yurt dışındaki bir hizmetle yapılacaksa KVKK m.9 güvencesi gerekir (rapora bakın); yurt içi model bu yükü kaldırır.
- Konuşma tanıma: web ve WebView'de `SpeechRecognition` bulunmayabilir. Uygulamada yerel konuşma tanıma veya sunucu tarafı çevirme servisi gerekir; ses çevirildikten sonra silinmelidir.
- Rıza penceresi ve metin her iki kanalda aynıdır; uygulamada sistem mikrofon ve kamera izni ayrıca istenir.

## 10. Markette unutma kontrolü (web ve uygulama)

İstemci listeyi ve fiş görüntüsünü sunucuya gönderir; sunucu görüntüyü okuyup her liste satırı için durum döner.

```json
// İstek: POST /v1/receipt/match  (rıza: rc)
{ "list": ["2 süt", "yağ", "deterjan"], "kind": "sepet" | "fis", "images": ["<jpeg/png/webp>", "..."] }
// Yanıt
{ "items": [ { "n": 1, "s": "var", "line": "PINAR SUT 1L" }, { "n": 2, "s": "yok", "line": null }, { "n": 3, "s": "belirsiz", "line": null } ] }
```

- `s`: `var`, `yok` ya da `belirsiz`. İstemci yalnızca `var` olanları işaretler; `yok` ve `belirsiz` uyarıda kalır.
- `kind: sepet` için görünmeyen ürün `yok` değil `belirsiz` döner; birden çok fotoğraf istemcide ayrı ayrı gönderilip birleştirilir (en fazla 6).
- Görüntü okunduktan sonra saklanmaz. Elle işaretleme sunucu gerektirmez.
- Uygulamada kamera izni sistemden ayrıca istenir; WebView'de dosya seçici kamera seçeneğini sunmuyorsa yerel kamera köprüsü gerekir.

## 11. Marketten bağımsız kullanım

- `POST /v1/compare` bir `markets` listesi alır (örn. `["Migros","BİM"]`); yalnızca bu marketlerin kanalları ve karma sepetler hesaplanır. Liste boşsa istek reddedilir.
- Fiyat karşılaştırması yalnızca Market Fiyatı verisi olan marketlerde yapılır. Başka bir markete giden kullanıcı için `compare` çağrılmaz; sesli liste (6), fiş okuma (4), unutma kontrolü (5) ve favoriler (3) market veya katalog bilgisi gerektirmez.
- Fiş okuma yanıtındaki `store` alanı serbest metindir; katalogdaki bir markete eşlenmesi gerekmez.
- Prototipte bu iki durum "Hangi marketlerde karşılaştırılsın?" seçimi (Karşılaştırma sekmesinde market rozetleri) ve rozetlerin yanındaki son seçenek "Başka bir markete gidiyorum" ile gösterilir; market logoları için resmî logo dosyaları ve kullanım izni gerekir.

## 12. Ortak liste (paylaşımlı liste)

Davet edilen kişiler (aile, arkadaş, ev arkadaşı) aynı listeye ürün ekler, çıkarır ve aldıklarını işaretler. Alışveriş tamamlanınca liste kapanır; kapalı listede ekleme ve çıkarma yapılamaz. Prototip bunu sayfanın paylaşımlı veri tabanıyla gösterir. Uygulamada bir sunucu servisi gerekir.

**Veri modeli**
- `lists`: `id`, `name`, `ownerId`, `createdAt`, `status` (`open` veya `closed`), `statusBy`, `statusAt`, `closedBy`, `closedAt`, `reopenedBy`, `reopenedAt`.
- `list_members`: `listId`, `userId`, `role` (`owner` ya da `member`), `invitedBy`, `joinedAt`.
- `list_items`: `id`, `listId`, `text`, `addedBy`, `addedAt`, `done`, `doneBy`. Ürünler ayrı kayıttır; iki kişi aynı anda ekleme yapınca biri diğerini ezmez.

**Uç noktalar**
- `POST /v1/lists`, `GET /v1/lists`, `GET /v1/lists/{id}`, `DELETE /v1/lists/{id}` (yalnızca oluşturan).
- `POST /v1/lists/{id}/invite` ve `POST /v1/lists/{id}/members/{userId}/remove`.
- `POST /v1/lists/{id}/items`, `PATCH /v1/lists/{id}/items/{itemId}` (işaretleme), `DELETE /v1/lists/{id}/items/{itemId}`.
- `POST /v1/lists/{id}/close` ve `POST /v1/lists/{id}/reopen`: gruptaki her üye çağırabilir.
- `GET /v1/lists/{id}/stream`: liste ve ürün değişikliklerini canlı iletir (WebSocket ya da SSE).

**Ürün özellikleri.** Ortak listedeki ürün yazı alanı (`text`) ve isteğe bağlı bir `spec` nesnesi taşır: seçilen ürün çeşidi (`productKey`), miktar (`quantity`), birim (`unit`), boyut (`size`), marka (`brand`). Özellikler Kontrol ekranında seçilir, "Ortak listeyi güncelle" bunları `spec` olarak yazar; liste yeniden açılınca aynı seçimler gelir. `PATCH /v1/lists/{id}/items/{itemId}` `spec` alanını da güncellemeli, düzeltme `addedBy` alanını değiştirmemeli, ayrı bir `updatedBy` tutulmalı. Favori kaydındaki `items[]` dizisi aynı alanları taşır.

**Alışveriş (ödeme) geçmişi.** Her liste kendi son alışverişlerini taşır. Favori listede kayıt yalnızca sahibine aittir (`GET /v1/favorites/{id}/orders`); ortak listede kayıtlar gruba açıktır: `GET /v1/lists/{id}/orders` ve `POST /v1/lists/{id}/orders` (`amount`, `date`, `source` = `online` ya da `store`, `merchant`, `lines[]`, `receiptId`). Kaydı oluşturan kimlik `createdBy` ile tutulur ve ekranda ad olarak gösterilir. Hangi listeye yazılacağı istemcinin o anda çağırdığı listeden gelir, kullanıcı ayrıca seçmez. Prototip tüm kayıtları saklar ve son 3 tanesini gösterir; fiş görseli sunucuya gönderilmez, yalnızca okunan tutar ve market adı yazılır. Kapalı listeye de alışveriş kaydı yazılabilmelidir (`409 list_closed` bu uç noktaya uygulanmaz). Üye olmayan kişi bu uç noktayı çağıramaz.

**Ortak listede fotoğraf ve sesle ekleme.** Ortak liste ekranı ana listedeki aynı okuma servislerini kullanır: fotoğraftan satır okuma (görüntü okuma servisi) ve konuşma tanıma. Sonuç doğrudan kaydedilmez; kullanıcı kutuda düzeltir, onaylayınca her satır `POST /v1/lists/{id}/items` ile ayrı kayıt olarak eklenir. Emin olunamayan satırlar `lowConfidence: true` işaretiyle gelir ve istemci bunların düzeltilmeden eklenmesine izin vermez. Fotoğraf ve ses için kullanıcı onayı (KVKK "fiş ve fotoğraf okuma", "sesli liste") sunucuda da kaydedilmelidir.

**Yemek önerisi.** Kontrol ekranındaki "Bu listeyle neler yapılır" kartı sunucuya istek atmaz; liste satırlarının ürün gruplarını (`group`) hazır bir tarif tablosuyla eşleştirir. Tarif tablosu ileride `GET /v1/recipes?groups=...` gibi bir servise taşınabilir; ürün grupları Market Fiyatı kategorileriyle eşlenmelidir. Öneriler yapay zekâ ile üretilmez, yalnızca eşleşen tarifler gösterilir.

**Yapay zekâ asistanı (ajan).** Asistan, istemciden yapay zekâ servisiyle çağrılır ve üç araç kullanır: `urunara(sorgu)` ürün kataloğunu, `tarifler()` tarif tablosunu, `planKontrol(satirlar)` fiyat ve toplam hesabını döndürür. Canlı sürümde bu üç araç Market Fiyatı ürün arama, tarif servisi ve sepet hesaplama uç noktalarına bağlanır. Güvenlik ilkesi: asistan sayı üretmez; toplam her zaman `planKontrol` sonucundan, istemci tarafında yeniden hesaplanır. Katalogda bulunmayan ürün kodları atılır. Asistanın konuşma metni sunucuda saklanmamalı, yurt dışı aktarım kararı KVKK bölümüne bağlıdır.

**Markette unutma kontrolü.** Kullanıcı sepet ya da fiş fotoğrafını uygulama içinde yaptığı listelerle eşleştirebilir: yazılan liste, favori liste ya da ortak liste. Ortak liste için ürünler `GET /v1/lists/{id}/items` ile okunur; favori için kayıtlı satırlar kullanılır. Fotoğraf eşleştirme servisi her satır için `var`, `yok` ya da `belirsiz` döndürür; `belirsiz` kullanıcıya "kendin bak" olarak gösterilir. İleride eşleşen ürünler ortak listede `done: true` olarak işaretlenebilir (henüz yapılmadı).

**Yüklenen fotoğraf önizlemesi.** Unutma kontrolünde yüklenen sepet ya da fiş fotoğrafı ekranda küçük önizleme olarak gösterilir. Önizleme tarayıcıda geçici bir adresle üretilir, sunucuya gönderilmez ve yeni yüklemede silinir. Okuma için gönderilen görüntü ayrı bir rıza gerektirir (kural 43–44).

**Liste sınırları.** Ana liste 60, ortak liste 100 ürünle sınırlıdır; ortak listede sınır dolunca tüm üyeler uyarıyı görür. Unutma kontrolü 100 satıra kadar çalışır. Sınır sayıları sunucu tarafında da doğrulanmalıdır, tarayıcı kontrolü yalnızca kullanıcı uyarısıdır. Ortak listede ürün ekleme eşzamanlı yapılırsa sınır birkaç ürün aşılabilir; gerçek sistemde bu işlem sunucuda tek adımda sayılarak yapılmalıdır. Deno için gün ve satır sınırı yoktur; çok uzun planlarda yanıt süresi ve maliyet artar.

**Çoklu fotoğraf ve silme.** Liste ve unutma kontrolü alanları birden çok fotoğraf (en fazla 6) alır; fazlası atlanır ve kullanıcıya bildirilir. Yanlış yüklenen fotoğraf küçük resmindeki × ile silinir; ondan okunan satırlar listeden ya da ortak liste kutusundan çıkarılır, unutma kontrolünde işaretler kalan fotoğraflardan yeniden hesaplanır. Fiş toplamı (Alışverişlerim) tek fotoğraftır. Gerçek sistemde fotoğraf sunucuya gidiyorsa silinen fotoğrafın saklanmadığı da doğrulanmalıdır.

**Kamera ve simüle demo.** Fotoğraf alanları tarayıcı kamerasını (getUserMedia, HTTPS gerekir) kullanır; izin verilmezse dosya seçiciye döner. Yakalanan görüntü sunucuya gidiyorsa saklanmaması, boyutunun sınırlanması ve KVKK rızası sunucuda doğrulanmalıdır. Herkese açık demo gerçek okuma yapmaz: fotoğraf, fiş ve Deno cevapları hazır örnektir ve ekranda "Örnek sonuç (simüle)" yazar. Fiyat ve toplamları kural motoru gerçekten hesaplar.

**Son alışverişlerim: sesle tutar ve kamera.** Her alışveriş kaydına fiş fotoğrafı (kamera ya da dosya), sesle söylenen tutar ya da elle yazılan tutar eklenebilir; okunan değer kullanıcı onayıyla kaydedilir. Ses tanıma tarayıcıdadır (Web Speech API; Firefox'ta yoktur), ses kaydı saklanmaz. Ortak listede kayıt, ekleyen kişinin kimliğiyle sunucuda tutulur ve yetki kontrolü (yalnızca üyeler) sunucuda yapılır. Herkese açık demoda ortak liste yerel kayıtla tek kişilik simüle edilir; gerçek çok kişili paylaşım için hesap ve sunucu gerekir.

**Kaynağı güncelle.** Kullanıcı ortak listeyi karşılaştırmaya alıp değiştirirse "Ortak listeyi güncelle" tek bir istek olmalı: `PUT /v1/lists/{id}/items` gövdesinde yeni ürün listesi, yanıtta eklenen ve çıkarılan ürünler. Sunucu liste kapalıysa `409 list_closed`, kullanıcı üye değilse `404` döndürür. Çıkarılan ürünlerin ve ekleyenin bilgisi güncelleme kaydına yazılır (`updatedBy`). Prototip ürünleri tek tek siler ve ekler; iki kişi aynı anda güncelleme yaparsa sonuncusu geçer. Favori güncelleme yalnızca kullanıcının kendi favori kaydını (`PUT /v1/favorites/{id}`) değiştirir; sipariş geçmişi korunur.

**Üyelik ve görünürlük sunucuda uygulanmalı.** Prototipte listenin `members` alanı vardır ve ekran yalnızca üyesi olunan listeleri gösterir; ama bu süzme istemcidedir, sayfaya erişen biri veriyi teknik olarak okuyabilir. Gerçek serviste `GET /v1/lists` yalnızca çağıranın üye olduğu listeleri döndürmeli, üye olmayan `GET /v1/lists/{id}` için `404` almalı. Üye eklemek için kullanıcı araması (`GET /v1/users?q=`) yalnızca davet edilebilir kişileri döndürmeli; davet kabul edilmeden liste görünmemeli. Ekranda "Listedekiler" ve "Ekleyenler: Ayşe 3" özeti için üye adları ve ürün başına `addedBy` kimlik üzerinden çözülür, ad kaydedilmez.

**Son saat (zamanlayıcı) sunucuda çalışmalı.** Listede `lockAt` (UTC zaman damgası, boş olabilir) alanı tutulur; gruptaki her üye koyabilir ya da kaldırabilir. Saat gelince sunucu listeyi kendisi `closed` yapar (`closedBy = system`) ve gruba bildirim gönderir; bunun için zamanlanmış iş (cron/kuyruk) gerekir. Ekleme, çıkarma ve işaretleme istekleri `lockAt` geçmişse `409 list_closed` ile reddedilmeli. Prototipte kalan süreyi açık sayfa hesaplar ve kapatmayı sayfa yazar; sayfa açık olmayan bir cihazda süre dolarsa kapanış bir sonraki açılışta yazılır, ama kilit her durumda önce kontrol edilir. Yeniden açma `lockAt` değerini temizler. Sunucu saati esas alınmalı, cihaz saati değil.

**Kilit sunucuda uygulanmalı.** Prototipte kapalı listenin kilidi ekranda uygulanıyor ve her değişiklikten hemen önce liste durumu yeniden okunuyor. İki kişi aynı saniyede işlem yaparsa nadiren biri geçebilir. Gerçek serviste ekleme, çıkarma ve işaretleme istekleri `status = closed` iken `409 list_closed` ile reddedilmeli; durum kontrolü ve yazma aynı işlemde (transaction) yapılmalı. Kapatma ile yeniden açma da sürüm numarasıyla (`If-Match`) korunmalı.

**Bildirim.** Liste kapanınca ya da yeniden açılınca gruptaki herkese bildirim gitmeli. Prototip yalnızca sayfayı açık tutanlara uyarı gösterir; sayfada olmayana ilk açışında "Sen yokken: …" der. Telefona gerçek bildirim için push servisi (FCM/APNs) ve her kullanıcı için cihaz kaydı gerekir. Bildirim metni: "“Liste adı” listesi Ayşe tarafından tamamlandı ve kapandı."

**Yetki ve KVKK.** Üye olmayan listeyi okuyamaz. Listedeki ürünler ve ekleyenin adı yalnızca üyelere gösterilir. Kullanıcı ortak listeye ilk eklemeden önce "ürünlerimin ve adımın paylaştığım kişilere gösterilmesini" açıkça kabul eder; bu onay sürümüyle birlikte rıza kaydında (bölüm 9) tutulur. Davet edilen kişi kabul etmeden listeyi göremez. Hesap silinince kişinin eklediği ürünlerde ad "Bir kişi" olarak anonimleştirilir.

**Favori ile ilişki.** "Ortak yap" favoriyi kopyalar, "Favoriye ekle" ortak listeyi kopyalar. İkisi birbirine bağlı değildir ve otomatik eşitlenmez. Kopyalama istemcide yapılır; sunucuda ayrı bir uç nokta gerekmez, mevcut favori ve liste servisleri yeterlidir.


**Ana ekrana eklenebilen web uygulaması (PWA) ve mağaza uygulaması.** Herkese açık demo bir PWA'dır: `manifest.webmanifest` (ad, simge, tam ekran), `sw.js` (önce ağ, ağ yoksa son kayıtlı sayfa) ve simgeler. HTTPS gerekir (GitHub Pages sağlar). Sunucuda yeni bir şey gerekmez. Kurumun kendi alan adında yayınlanırsa `start_url` ve `scope` o adrese göre ayarlanır, önbellek sürümü (`V`) her yayında artırılır. Gerçek fiyat verisi bağlanınca fiyat ve liste istekleri önbelleğe alınmamalı (bayat fiyat riski); yalnızca sayfa kabuğu önbelleğe alınır. iOS'ta PWA bildirimleri ve arka plan çalışması sınırlıdır; gerçek push bildirimi için mağaza uygulaması (FCM/APNs) gerekir. Mağaza uygulaması aynı arayüzü bir kabuk içinde (ör. Capacitor ya da React Native/Flutter ile yeniden yazım) çalıştırır; hesap motoru ve API sözleşmesi aynı kalır. Prototipte mağaza uygulaması yoktur.
