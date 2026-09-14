# Şekil Bahçesi

**Diyar:** Sayılar Şehri  
**Slug:** `sekil-bahcesi`  
**Bileşen:** `SekilBahcesi.vue`  
**Tür:** Puanlı şekil tanıma oyunu  
**İçerik havuzu:** Gerekli değil

Oyuncu kolay seviyede şeklin adını bulur, orta seviyede köşe sayısı ve ortak özellik eşleştirmesi yapar, zor seviyede şekil örüntüsünü tamamlar. Her cevap motor tarafından bir tur olarak değerlendirilir.

## Bileşen kontratı

Bileşen yalnızca `engine`, `params` ve `pool` proplarını alır. `defineEmits` kullanılmaz. Tur, puan ve oyun bitişi motor tarafından yönetilir.

## Parametreler

| Param | İzinli değerler | Varsayılan | Ne işe yarar |
|---|---|---|---|
| `level` | `kolay`, `orta`, `zor` | `kolay` | Sırasıyla şekil adı, köşe/özellik eşleştirmesi ve örüntü tamamlama mekaniğini seçer. |
| `rounds` | `3`–`30` | `5` | Toplam motor turu; bileşen bu değeri kendi ilerletmez. |
| `feedbackDurationMs` | `500`–`4000` | `1900` | Cevap geri bildiriminin ekranda kalma süresi. |

## Canlı ayarlar

Bileşen `{ sizeScale: 1, elementCount: null, speed: 100 }` varsayılanıyla `liveSettings` kullanır. `sizeScale` metinleri, `speed` geri bildirim süresini etkiler. `elementCount` bu mekanikte kullanılmaz.

## İçerik havuzu

Oyun geometrik şekilleri prosedürel olarak oluşturduğu için `pool` kullanmaz. `pool` değeri `null` olduğunda çalışmaya devam eder.

## Motor ve skorlama

- `engine.round.value` değiştiğinde yeni tur hazırlanır.
- Bileşen tur değerini elle artırmaz.
- Her turda çift cevabı engelleyen kilit bulunur.
- Geri bildirimden sonra `engine.answer(isCorrect, meta)` yalnızca bir kez çağrılır.
- `meta` içinde `tip`, `hedef`, `secilen`, `sekil`, `soruTuru` ve gerektiğinde `oruntu` gönderilir.
- Sonuç ekranını ve sunucu iletişimini ana sistem yönetir.

## Dosyalar ve bağımlılık

Teslim edilecek oyun bileşeni `src/SekilBahcesi.vue` dosyasıdır; bileşene özel stiller aynı dosyanın içindedir. `src/App.vue` yalnızca bilgisayarda çalıştırmak için kullanılan yerel test motorudur. Görseller `src/assets` altında WebP biçimindedir. Phaser kullanımı onaylanmıştır.

Yerel testte `?level=kolay`, `?level=orta` veya `?level=zor` sorgusu kullanılarak seviyeler ayrı ayrı açılabilir.
