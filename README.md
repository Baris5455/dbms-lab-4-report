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

- [X]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [ ]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [ ]  LRU / CLOCK gibi algoritmaları
- [X]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [X]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
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

| Kavram      | Bellek          | Disk / DB      |
| ----------- | --------------- | -------------- |
| Adresleme   | Pointer         | Page + Offset  |
| Hız         | O(1)            | Page IO        |
| PK          | Yok             | Index anahtarı |
| Veri yapısı | Array / Pointer | B+Tree         |
| Cache       | CPU cache       | Buffer Pool    |

---

# Video [Linki](https://youtu.be/UbSR06WYu14) 


---

# Açıklama (Ort. 600 kelime)

# Veri Yapısı
Veritabanı sistemlerinde performansın temel belirleyicilerinden biri disk erişim sayısıdır. Verilerin kalıcı olması gerektiği için SQLite gibi veritabanları veriyi SSD veya HDD üzerinde saklar. Ancak disk erişimi, belleğe (RAM) kıyasla oldukça yavaştır. Bu nedenle her sorguda doğrudan diske erişmek ciddi bir performans problemi oluşturur.
Bu problemi azaltmak için veritabanları, veriyi satır satır değil, sabit boyutlu veri blokları olan sayfalar (pages) halinde organize eder. Diskler tek tek kayıt okumaz; her zaman bir sayfayı komple okur. Bu nedenle bir kayda erişildiğinde, o kaydın bulunduğu sayfanın tamamı belleğe yüklenir ve sayfa bellekteyken birden fazla işlem tekrar diske gitmeden yapılabilir.
SQLite’ta veriler genellikle 4 KB boyutundaki sayfalara bölünerek diskte saklanır. Diskten okunan bu sayfaların RAM’deki karşılığı MemPage veri yapısıdır. struct MemPage, sayfanın yaprak mı yoksa iç düğüm mü olduğunu, kaç kayıt içerdiğini ve sayfa içindeki veri düzenini tutar. Bu sayfa tabanlı yaklaşım sayesinde sık kullanılan sayfalar RAM’de tutulur ve disk I/O sayısı önemli ölçüde azalır.

# Arama Algoritması
SQLite, bu sayfaları disk üzerinde B+Tree veri yapısı ile organize eder. B+Tree; kök, ara düğümler ve yaprak düğümlerden oluşan dengeli bir ağaçtır. Gerçek veriler yalnızca yaprak düğümlerde bulunur; üst düğümler yalnızca yönlendirme bilgisi tutar. SQLite’ta B+Tree’nin her düğümü doğrudan bir disk sayfasına karşılık gelir. Bu sayede milyonlarca kayıt içeren tablolarda bile arama işlemleri genellikle yalnızca birkaç sayfa erişimiyle tamamlanır ve O(log N) zaman karmaşıklığında gerçekleştirilir.
B-Tree üzerinde gerçekleştirilen tüm işlemler BtCursor adı verilen bir cursor nesnesi aracılığıyla yürütülür. Cursor, belirli bir B-Tree’ye bağlanarak ağaç üzerinde konumlanma, arama ve sıralı gezinme işlemlerini gerçekleştirir.
SQLite’ta iki temel B-Tree türü bulunur: Table B-Tree ve Index B-Tree.
Table B-Tree’lerde gerçek satır verileri saklanır ve anahtar olarak rowid (INTEGER) kullanılır.
Index B-Tree’lerde ise gerçek satırlar bulunmaz; bunun yerine bir veya birden fazla sütundan oluşan indeks anahtarı ve ilgili satırı işaret eden rowid tutulur. Bu ayrım, arama algoritmasının nasıl çalışacağını doğrudan belirler.
Table B-Tree’lerinde arama işlemleri sqlite3BtreeTableMoveto(...) fonksiyonu ile yapılır. Bu fonksiyon, verilen rowid değerine göre B-Tree’nin kök sayfasından başlayarak aşağı iner ve hedef kaydın bulunabileceği yaprak sayfada cursor’u konumlandırır.
Index B-Tree’lerinde ise sqlite3BtreeIndexMoveto(...) fonksiyonu kullanılır. İndeks anahtarları birden fazla kolondan oluşabildiği için arama anahtarı UnpackedRecord yapısı ile temsil edilir. Fonksiyon, indeks anahtarları üzerinde ikili arama yaparak cursor’u uygun konuma taşır. Bu mekanizma, WHERE, JOIN ve ORDER BY gibi sorguların tam tablo taramasına dönüşmesini engelleyerek ciddi performans kazanımı sağlar.
Bu iki arama mekanizmasını yöneten ortak yapı btreeMoveto(...) fonksiyonudur. Bu fonksiyon, cursor’un bağlı olduğu B-Tree’nin tablo mu yoksa indeks mi olduğunu belirler ve uygun arama fonksiyonunu çağırır. Böylece tablo ve indeks aramaları için modüler ve yeniden kullanılabilir bir altyapı sağlanır.
Arama sonrasında, B+Tree yapısında yaprak düğümlerin birbirine bağlı olması sayesinde sqlite3BtreeFirst, sqlite3BtreeLast, sqlite3BtreeNext ve sqlite3BtreePrevious fonksiyonları ile hızlı sıralı dolaşım yapılabilir. Cursor’un ağaç üzerinde güvenli şekilde hareket etmesi ise moveToRoot, moveToChild ve moveToParent gibi yardımcı fonksiyonlar tarafından sağlanır.

# WAL ilkesi
Sonuç olarak SQLite, verileri sayfa tabanlı olarak saklayıp B+Tree veri yapısı ile organize ederek disk erişimlerini minimize eder. Sayfaların bellekte MemPage yapısı ile temsil edilmesi ve tüm işlemlerin cursor tabanlı yürütülmesi sayesinde, ölçeklenebilir ve yüksek performanslı bir veritabanı altyapısı sunar.
SQLite’ta WAL (Write-Ahead Logging) modu etkinleştirildiğinde, veritabanındaki değişiklikler doğrudan ana veritabanı dosyasına yazılmaz. Bunun yerine tüm güncellemeler önce WAL dosyasına kaydedilir. Bu yaklaşım, okuma ve yazma işlemlerinin birbirini engellemeden çalışmasını sağlar.
WAL altyapısı sqlite3WalOpen fonksiyonu ile başlatılır. Bu aşamada WAL dosyası ve wal-index (shared memory) hazırlanır ve SQLite artık rollback journal yerine WAL kullanacağını bilir. Yazma işlemleri öncesinde sqlite3WalBeginWriteTransaction, aynı anda yalnızca tek bir yazıcının çalışmasını garanti altına alır.
Değiştirilen sayfaların WAL’a yazılmasından sqlite3WalFrames sorumludur. Bu fonksiyon, değişen her sayfayı bir frame olarak WAL dosyasına ekler ve commit işlemini işaretler. Bu sırada ana veritabanı dosyasına dokunulmaz. Okuma sırasında ise sqlite3WalFindFrame ve sqlite3WalReadFrame fonksiyonları devreye girer; SQLite önce WAL’da daha güncel bir sayfa olup olmadığını kontrol eder, varsa WAL’dan okur, yoksa ana veritabanı dosyasına başvurur.
Zamanla büyüyen WAL dosyasının temizlenmesi ve kalıcı hâle getirilmesi sqlite3WalCheckpoint ile yapılır. Bu işlem, WAL’daki güvenli değişiklikleri ana veritabanı dosyasına kopyalar. Okuyucuların tutarlı bir görünüm (snapshot) görmesini sağlayan mekanizma ise walIndexReadHdr / walIndexWriteHdr fonksiyonlarıdır. Bu sayede yeni commit’ler, hâlihazırda okuma yapan işlemleri etkilemez.


## VT Üzerinde Gösterilen Kaynak Kodları

[Linki](https://github.com/mackyle/sqlite/blob/master/src/btreeInt.h) btreeInt.h: SQLite’ın B+Tree yapısının iç veri yapılarını, MemPage, BtCursor ve düşük seviyeli yardımcı tanımları içeren dahili başlık dosyasıdır. \
[Linki](https://github.com/mackyle/sqlite/blob/master/src/btree.c) btree.c: Tablo ve indeksler üzerinde gezinme, arama ve ekleme işlemlerini yapan B+Tree algoritmalarının ana implementasyonunu barındırır. \
[Linki](https://github.com/mackyle/sqlite/blob/master/src/wal.c) wal.c: Write-Ahead Logging (WAL) mekanizmasını; log dosyasına yazma, okuma, checkpoint ve eşzamanlılık kontrolünü yöneten kodları içerir.
