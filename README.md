# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [X]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [ ]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [ ]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [X]  LRU / CLOCK gibi algoritmaları
- [ ]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [ ]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [ ]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [X]  WAL (Write Ahead Log) İlkesi
- [ ]  Log disk (fsync vs write) sistem çağrıları farkı

---

# Özet Tablo
| Kavram       | Bellek                 | Disk / DB             |
|--------------|------------------------|-----------------------|
| Adresleme    | Pointer                | Page + Offset         |
| Hız          | O(1)                   | Page IO               |
| Değiştirme   | LRU / CLOCK            | Buffer Pool Eviction  |
| Dayanıklılık | Uçucu (kalıcı değil)   | WAL (Write-Ahead Log) |
| Kurtarma     | Yok                    | WAL Redo              |
---

# Video [Linki](https://www.youtube.com/watch?v=Nw1OvCtKPII&t=2635s) 
Ekran kaydı. 2-3 dk. açık kaynak V.T. kodu üzerinde konunun gösterimi. Video kendini tanıtma ile başlamalıdır (Numara, İsim, Soyisim, Teknik İlgi Alanları). 

---

Blok Bazlı Erişim 

Diskteki veriye erişim genelde blok bazlıdır. Yani veritabanı diski satır satır okumaz, belirli boyutta bloklar veya sayfalar halinde okur yazar. Bu yüzden bir veriyi bulmak için genelde “hangi blokta” ve “blok içinde nerede” bilgisini kullanırız. Burada blok id, verinin bulunduğu sayfayı temsil eder, offset ise o sayfanın içindeki konumu gösterir. Bu mantık aslında adresleme gibidir, RAM’de pointer ile erişiyorsak diskte de page + offset ile erişiyoruz.

Bu yaklaşımın sebebi disk I/O’nun pahalı olmasıdır. Tek bir kaydı okumak için bile diskten bir sayfa gelir, çünkü diskten okuma birimi sayfadır. O yüzden veritabanları sayfaları standart boyutta tutar. Buffer pool da zaten bu sayfaların RAM’deki kopyalarını tutuyor. Yani blok bazlı erişim, buffer pool ile direkt bağlantılıdır. İstenen veri yoksa diskten ilgili sayfa gelir, varsa RAM’de zaten sayfa durduğu için hızlı olur. Bu yüzden “page + offset” mantığını anlamak, veritabanının neden buffer pool kullandığını ve neden LRU/CLOCK gibi algoritmalara ihtiyaç duyduğunu daha anlaşılır yapar.


Buffer Pool’da Sayfa Değiştirme LRU / CLOCK

Veritabanları veriyi kalıcı olarak diskte tutar ancak disk RAM’e göre çok yavaştır. Bu yüzden DBMS içinde buffer pool denen bellek alanı kullanılır. Diskteki veri sayfalarının RAM’deki kopyalarını tutar. Bir sorgu çalıştığında ihtiyaç duyulan sayfa önce buffer pool’da aranır. Bulunursa hit olur ve diskten okumaya gerek kalmaz, bulunmazsa miss oluşur, sayfa diskten okunur ve buffer pool’a yerleşir. Ama buffer pool kapasitesi sınırlı. Dolduğu anda yeni sayfa eklemek için mevcut sayfalardan birini çıkarır. Bu performansı doğrudan etkileyen sayfa değiştirme algoritmalarıyla yapılır.

LRU’nun mantığı basittir: sayfayı uzun süredir kullanmıyorsan büyük ihtimalle bidaha kullanmazsın, çöpe at mantığıyla işler. Böylece RAM’de daha güncel ve tekrar erişilme ihtimali yüksek sayfalar kalır. LRU’nun kodlanması sezgiseldir. Sayfalar erişim sırasına göre tutulur. Bir sayfaya her erişimde o sayfa en son kullanılan olur. Kapasite aşılırsa en eski yani uzun süredir kullanılmayan sayfa çıkarılır. Bu yöntem buffer pool’un hit oranını artırıp diskin yükünü azaltmayı hedefler.

Ancak tam LRU uygulamak her erişimde liste güncellemesi gerektirdiği için bazı sistemlerde maliyetli olur. Bu nedenle pratikte LRU’ya benzer ama daha ucuz bir yöntem olan CLOCK kullanılır. CLOCK, sayfaları dairesel bir listede tutar ve her sayfada kullanılıyor mu bilgisini gösteren bir reference bit bulunur. Sayfaya erişilince bit 1 yapılır. Sayfa atılacaksa saat ibresi dolaşır: Bit 0 olan bir sayfa görülürse o sayfa seçilir. Bit 1 ise sayfanın bit 0’a çekilir ve ibre ilerler. Bu sayede CLOCK, sık kullanılıyorsa koru fikrini LRU gibi taşır fakat  güncelleme maliyeti  düşüktür. Özetle buffer pool’da LRU/CLOCK seçimi, diske gidişi azaltarak veritabanının hızını belirler.


WAL  İlkesi

Veritabanlarında önemli hedeflerden biri, sistem aniden çökerse verinin tutarlı kalmasıdır. Çökme anında bazı işlemler yarım kalmış olabilir, veritabanı tekrar açıldığında değişiklik oldu mu? sorusunu cevaplamalıdır. Bu sorunun çözüm mekanizması WAL (Write-Ahead Log) yaklaşımıdır.

WAL’ın kuralı: Veri sayfaları diske yazılmadan önce yapılacak değişiklik log’a yazılmalıdır. Log, ardışık yazıldığı için diskte verimli çalışır. Bir güncelleme geldiğinde DBMS önce bu işlemin kaydını WAL dosyasının sonuna ekler. Ardından log’un diske kalıcı yazıldığından emin olur ve daha sonra asıl veri sayfası güncellenir. Böylece sistem veri sayfasını yazamadan çökerse bile log kaydı durduğu için yeniden başlatmada eksik kalan değişiklik logdan tekrar uygulanabilir.

Kurtarma sırasında veritabanı WAL’ı okuyarak tutarlılığı geri kurar. Basitçe logda var ama veri sayfasında yoksa yeniden yap. Bu sayede çökme olsa dahi kayıp minimuma iner ve veritabanı doğru duruma döner. WAL ayrıca eşzamanlılık ve performans için de avantajlıdır. Birçok küçük güncellemeyi anında veri sayfalarına zorla yazmak yerine, önce log’a hızlıca ekleyip sonra sayfaları uygun zamanda diske yazmak mümkün olur. Sonuç olarak WAL, veritabanının hem güvenilirliğini hem de pratikte performansını artıran temel bir tasarımdır.

## VT Üzerinde Gösterilen Kaynak Kodları

Açıklama [Linki](https://...) \
Açıklama [Linki](https://...) \
Açıklama [Linki](https://...) \
... \
...
