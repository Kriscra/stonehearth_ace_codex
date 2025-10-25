# Düzenleme ve Optimizasyon Gereksinimleri

Aşağıdaki maddeler, depodaki kodu incelerken tespit edilen bakım ve performans konularını özetler. Her bölümde ilgili dosya, sorun/iyileştirme alanı ve beklenen kazanç belirtilmiştir.

## renderers/connection/connection_renderer.lua
- **Sorun:** `ConnectionRenderer:_update` içinde vurgulama adımı TODO olarak devre dışı bırakılmış. UI mod değiştiğinde bağlayıcıların görsel durumu güncellenmediği için oyuncular bağlantı durumunu takip etmekte zorlanıyor.
- **İhtiyaç:** `self:_do_hilight_check()` çağrısını yeniden etkinleştirip hatalı aydınlatmayı düzeltmek veya alternatif bir görsel geri bildirim geliştirmek.
- **Beklenen Kazanç:** Görsel netlik artışı ve bağlantı hatalarının daha hızlı fark edilmesi.

## components/auto_craft/auto_craft_component.lua
- **Sorun:** Bileşen, oyuncu kimliği değişimini, aktif sipariş iptallerini ve tarif etkinlik durumlarını yönetmiyor; ayrıca görev deposuyla entegrasyon tamamlanmamış. Kod içinde kapsamlı TODO listesi bulunuyor.
- **İhtiyaç:** Bileşene olay dinleyicileri ve durum geçişleri ekleyerek kimlik değişimleri, iptal edilen siparişler ve tarif etkinliği senaryolarını ele almak; görev deposunu ayrı bir varlıkla ayrıştırmak.
- **Beklenen Kazanç:** Otomatik üretim atölyelerinin hatasız ve tasarlanan şekilde çalışması.

## monkey_patches/ace_storage_component.lua
- **Sorun:** `storage_contains_filter_fn` ve `eval_best_passing_item` fonksiyonları neredeyse aynı mantığı tekrarlıyor; filtre önbelleği yönetimi karmaşık ve TODO açıklamasına göre performans darboğazı oluşturuyor.
- **İhtiyaç:** Filtre sonuçları için ortak yardımcı fonksiyon çıkarılması, geçersiz varlıkların daha agresif temizlenmesi ve gerekirse artımlı değerlendirme stratejisi uygulanması.
- **Beklenen Kazanç:** Depo filtrelemesi sırasında CPU kullanımının azalması ve daha az gecikme.

## monkey_patches/ace_water_component.lua
- **Sorun:** Su kanalı hesaplamasında sabit değerler kod içine gömülü (`removal` değeri vb.) ve `csg::GetAdjacent` gibi daha uygun yardımcılar kullanılmıyor. Bölgesel güncellemeler sırasında manuel kenar yönetimi karmaşık ve hataya açık.
- **İhtiyaç:** Sabitlerin yapılandırılabilir hâle getirilmesi, kenar belirleme için mevcut CSG yardımcılarının değerlendirilmesi ve ıslanma hacmi azaltma mantığının yeniden gözden geçirilmesi.
- **Beklenen Kazanç:** Su fiziği güncellemelerinde tutarlılık, bakımı kolaylaştırma ve potansiyel performans iyileştirmesi.

## monkey_patches/ace_hydrology_service.lua
- **Sorun:** Su işlemcileri her tikte tek tek işleniyor; yorumlara göre net değişimlerin birleştirilmesi ve sıralı uygulama performansı artırabilir. Ayrıca minimum/maksimum yükseklik aralığı boş olduğunda gereksiz döngüler hâlâ çalıştırılabiliyor.
- **İhtiyaç:** İşlemci değişikliklerini toplu hâlde işleyip tek seferde uygulayacak bir tampon katman tasarlamak; yükseklik aralığı doğrulamalarını erken çıkışlarla güçlendirmek.
- **Beklenen Kazanç:** Statik su sistemlerinde belirgin performans artışı ve tik başına daha kısa işlem süresi.

## services/server/inventory/restock_director.lua
- **Sorun:** Errand değerlendirmesi sırasında `_is_errand_valid` çağrısı her denemede çalıştırılıyor; kodda bunun pahalı olabileceği belirtilmiş. Kuyruk ve önbellek yapıları karmaşık, hata durumlarında tekrar kuyruğa ekleme maliyetli.
- **İhtiyaç:** Geçerlilik kontrollerini aksiyon başlangıcına erteleyip sonuçları daha uzun süre önbellekte tutmak veya artımlı doğrulama uygulamak. Hatalı maddeler için yeniden deneme sıklığını oyuncu ayarlarına bağlamak düşünülebilir.
- **Beklenen Kazanç:** Dinamik stok yenileme sırasında işlem yükünün azalması ve iş kuyruğu gecikmelerinin düşmesi.

## ai/lib/healing_lib.lua
- **Sorun:** `make_healing_filter` her hedef için benzersiz filtre anahtarları üreterek çok sayıda AI filtresi oluşturuyor; yorum performans riskine işaret ediyor.
- **İhtiyaç:** Filtre anahtarlarını normalleştirip gereksiz kombinasyonları azaltmak veya sonuçları belirli bir süre önbelleğe almak.
- **Beklenen Kazanç:** Şifa işlevleri sırasında AI yükünün ve gereksiz filtre yaratımlarının azalması.

## services/client/heatmap/heatmap_service.lua
- **Sorun:** Heatmap listesi JSON'dan tek seferlik okunuyor; TODO notu dinamik olarak eklenip kaldırılabilen ısı haritaları için datastore kullanımını öneriyor. Şu an yeni haritalar eklemek kod güncellemesi gerektiriyor.
- **İhtiyaç:** Isı haritalarını konfigürasyon dosyası veya datastore aracılığıyla kayıt altına alacak esnek bir sistem kurmak ve UI'ya yeniden yükleme kabiliyeti eklemek.
- **Beklenen Kazanç:** Modülerlik artışı ve yeni ısı haritalarını oyuna ekleme sürecinin kolaylaşması.

## Diğer İzlenimler
- `renderers/connection`, `monkey_patches` ve `services` klasörleri yoğun olarak TODO ve performans yorumları içeriyor; düzenli bir teknik borç temizliği planlanmalı.
- Global log çağrıları (`log:debug`, `log:spam`) üretim ortamında yüksek hacimli günlük oluşturabilir; yapılandırılabilir seviye eşikleri düşünülmeli.

