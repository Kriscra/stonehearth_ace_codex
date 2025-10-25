# Yeni Proje Taslağı: AI Odaklı Optimizasyon ve CPU Verimliliği

Bu doküman, mevcut kod tabanından bağımsız olarak **sıfırdan** inşa edilecek, AI davranışlarını ve CPU verimliliğini merkeze alan yeni oyunsal simülasyon projesinin gereksinimlerini ve teknik yol haritasını tarif eder.

## 1. Ürün Vizyonu
- **Odak:** Büyük ölçekli kolonilerde görev planlaması ve çevresel simülasyonları akıcı biçimde çalıştıran, AI kararlarını gerçek zamanlı optimize eden bir çekirdek.
- **Hedef Platform:** Çok çekirdekli masaüstü CPU'lar, isteğe bağlı düşük güçlü taşınabilir cihaz desteği.
- **Ölçülebilir Başarı Kriterleri:**
  - 200+ aktif ajan için ortalama tik süresi ≤ 33 ms.
  - Görev kuyruğu gecikmesi (iş seçme → icra başlangıcı) < 500 ms.
  - AI planlama hatasında %5'in altında sapma (beklenen hedefe ulaşamayan görev sayısı / tüm görevler).

## 2. Sistem Bileşenleri
### 2.1 Çekirdek Simülasyon Katmanı
- **Zamanlayıcı:** Deterministik `FixedDeltaTimeScheduler` (tick) + adaptif `AsyncWorkQueue`.
- **Dünya Temsili:** Bölgesel bölünmüş 3B grid (chunk tabanlı). Her chunk için veri:
  - Statik geometri (`voxel_type`, `navmesh_id`).
  - Dinamik durum (`temperature`, `water_level`, `resource_nodes`).
- **Veri Yerelliği:** Her chunk için `SoA` (Structure of Arrays) yerleşimi ile cache hit oranı artırılacak.

### 2.2 AI Davranış Katmanı
- **Karar Modeli:** Hedef odaklı planlama (GOAP) + hafif görev ağaçları.
- **Plan Önişleme:** Ajansız `PlannerService` tik başına en fazla `N` plan hesaplayacak, geri kalanı ardışık iş kuyruğuna aktarılacak.
- **Durumsal Bağlam:** `BlackboardComponent` aracılığıyla ajan başına minimal bellek ayak izi (sadece gerekli sensörler).
- **Önceliklendirme:** `UtilityScore` tabanlı çok kriterli değerlendirme (sağlık, mesafe, kaynak kıtlığı).

### 2.3 Kaynak ve Stok Yönetimi
- **Veri Modeli:** `InventoryShard` yapısı ile koloni envanteri bölgelere ayrılır.
- **Arama:** Her shard için kd-tree benzeri `SpatialIndex`. Aramalar `radius → filter` kombinasyonlarıyla logaritmik maliyetli olur.
- **Kısıtlama:** Stok okunması snapshot üzerinden gerçekleşir, yazma işlemleri `CommandBuffer` ile tek tikte uygulanır.

### 2.4 Görev Planlayıcı
- **Kuyruklama:** `TaskStream` → `ReadyQueue` → `WorkerLane` aşamaları.
- **Bekleme Analizi:** Her iş, hesaplanan CPU bütçesine göre `time_slice` etiketlenir. Limit aşılırsa görev bir sonraki tike sarkar.
- **İptal/Öncelik Güncelleme:** Lock-free `ticket` sistemi; iptal isteği O(1) amorti.

### 2.5 Sistem Servisleri
- **Navigasyon Servisi:** Dinamik navmesh güncellemeleri için `HierarchicalNavGraph`. Bariyer değişimlerinde lokal güncelleme.
- **Olay Sistemi:** `EventBus` (tek yazarlı, çok okuyuculu halka tamponu). CPU yükünü azaltmak için `event coalescing`.
- **Fizik / Çevre:** Su, ısı ve bitki büyümesi gibi sistemler `ComponentSystem` altında bağımsız modüller.

## 3. CPU Optimizasyon İlkeleri
1. **Çok Çekirdekli İşleme:**
   - `WorkerPool` ile IO dışı tüm işler için çoklu iş parçacığı.
   - `JobFence` ve `DependencyGraph` yardımıyla veri yarışları önlenir.
2. **Cache Dostu Veri:**
   - Sık erişilen veri yapıları (ajan konumları, görev slotları) için `Struct of Arrays`.
   - `MemoryArena` ile sıcak veriyi tek blokta tutma.
3. **Adaptif Update Frekansı:**
   - Her servis kendine ait `update_budget_ms` belirler, aşılırsa güncelleme sıklığı azaltılır.
   - Önemsiz simülasyonlar (ör. dekoratif animasyonlar) `N` tikte bir çalıştırılır.
4. **Ölçüm ve Telemetri:**
   - Her alt sistem için `FrameProfiler` kancaları.
   - Canlı `PerfHUD` (ortalama tik süresi, görev kuyruğu uzunluğu).
5. **İşlem Önceliği:**
   - Ajansız (idle) ajanlar düşük öncelikli iş kuyruğuna taşınır.
   - Kritik görevler (saldırı, yangın) gerçek zamanlı lane'e alınır.

## 4. Geliştirme Yol Haritası
1. **Temel Altyapı (Sprint 1-2):**
   - `FixedDeltaTimeScheduler`, `ChunkWorld` ve `EventBus` prototipleri.
   - Profilleme altyapısı + ilk metrik panosu.
2. **AI Çekirdeği (Sprint 3-4):**
   - GOAP planlayıcı + `BlackboardComponent` temeli.
   - 10 temel aksiyon (kazı, taşıma, inşa) için davranış şablonları.
3. **Kaynak Yönetimi (Sprint 5):**
   - `InventoryShard`, `SpatialIndex`, snapshot mekanizması.
4. **Görev Planlayıcı (Sprint 6):**
   - `TaskStream` → `WorkerLane` ardışık düzeni.
   - CPU bütçesi takipleri ve önceliklendirme.
5. **Çok Çekirdekli Yayılım (Sprint 7):**
   - WorkerPool genişletme, veri yarışlarının giderilmesi.
   - Kritik servislerde paralellik testleri.
6. **Stres Test ve Optimizasyon (Sprint 8+):**
   - 200/400 ajan senaryoları ile yük testleri.
   - Profillerde %20'den fazla CPU kullanımına sahip modüller için hedefli iyileştirmeler.

## 5. Teknik Standardizasyon
- **Dil/Runtime:** C++20 veya Rust + Lua/AngelScript türü script katmanı.
- **CI/CD:** Otomatik performans regresyon testi (benchmark sahneleri + başarım metrikleri).
- **Kod Rehberi:**
  - Çekirdek modüller için `noexcept` ve `constexpr` kullanımına öncelik.
  - Script köprüsü için veri kopyası yerine `Handle`/`View` paylaşımlı yapılar.

## 6. Kullanım Kolaylığı (Tooling)
- **Geliştirici Konsolu:** Profil metrikleri, AI plan görselleştirici.
- **Modlama API'sı:** Görev tanımları ve aksiyon ağaçları için deklaratif JSON + script hook'ları.
- **Debug UI:**
  - Anlık görev kuyruğu listesi.
  - Ajan başına CPU bütçe tüketim grafiği.

## 7. Riskler ve Alınacak Önlemler
| Risk | Etki | Azaltma |
| --- | --- | --- |
| Çok çekirdekli eşzamanlılık hataları | Kritik (çökme/veri kaybı) | `ThreadSanitizer`, deterministik test senaryoları |
| GOAP planlama maliyetleri | Orta (tik süresi artışı) | Plan cache, benzer hedefler için şablon reuse |
| Veriye aç geliştirici araçları | Orta | UI prototiplerinin erken geliştirilmesi |

## 8. Sonraki Adımlar
1. Mimarinin bileşen diyagramını oluşturup ekip onayına sunmak.
2. Prototip için küçük ölçekli (20 ajan) deneme sahnesi hazırlamak.
3. Performans metriklerini CI pipeline'ına entegre etmek.

Bu plan, AI merkezli yeni bir kolonizasyon simülasyon projesinin hem CPU kullanımını optimize edecek hem de geliştirici/oyuncu deneyimini kolaylaştıracak temel yapı taşlarını belirler.
