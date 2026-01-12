# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [x]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [x]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [ ]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [x]  LRU / CLOCK gibi algoritmaları
- [ ]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [ ]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [X]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [X]  WAL (Write Ahead Log) İlkesi
- [ ]  Log disk (fsync vs write) sistem çağrıları farkı

---

# Özet Tablo

| Kavram             | İşletim Sistemi / Bellek (RAM) | Veritabanı / Disk (PostgreSQL)      |
| :----------------- | :----------------------------- | :---------------------------------- |
| **Erişim Birimi**  | Byte / Word                    | **Page (Sayfa/Blok)** (8KB)         |
| **Adresleme**      | Memory Pointer                 | **Block_id + Offset** (ItemPointer) |
| **Hız / Maliyet**  | O(1) / Nanosecond              | **Page I/O** / Millisecond          |
| **Veri Yapısı**    | Array / Linked List            | **B-Tree (Index) + Heap (Data)**    |
| **Cache Yönetimi** | OS Page Cache (LRU)            | **Buffer Pool** (Clock Sweep Alg.)  |
| **Kalıcılık**      | Volatile (Uçucu)               | **WAL** (Write Ahead Log) + fsync   |

---

# Video [Linki](https://www.youtube.com/watch?v=VmCt-j3aA5M) 

---

# Açıklama (Ort. 600 kelime)

Bu çalışma, açık kaynaklı ilişkisel veritabanı yönetim sistemi olan PostgreSQL'in kaynak kodları incelenerek; sistem programlama (işletim sistemi, disk I/O, bellek yönetimi) ve veri yapıları (B-Tree, Heap, WAL) perspektifinden performans optimizasyonlarını analiz etmek amacıyla hazırlanmıştır. Veritabanı sistemleri, işletim sisteminin sunduğu genel amaçlı dosya sistemi soyutlamalarının ötesine geçerek, disk erişim maliyetlerini minimize etmek ve veri bütünlüğünü sağlamak için özelleşmiş mimariler kullanır.

**Sistem Perspektifi ve Disk Erişimi**
Veritabanlarında performansın birincil darboğazı disk I/O işlemleridir. İşletim sistemi seviyesinde veriler byte akışı olarak görülse de, PostgreSQL veriyi blok bazlı (Page) yönetir. src/include/storage/bufpage.h dosyasında tanımlanan PageHeaderData yapısı, her bir 8KB'lık sayfanın başında yer alan meta verileri tutar. Bu yapı, sayfanın doluluk oranını, boş alan başlangıcını (pd_lower) ve satırların başlangıcını (pd_upper) takip eder. Veritabanı, veriye erişmek istediğinde diske rastgele (random access) gitmek yerine, bu blokları belleğe alarak okuma yapar.

Sayfa belleğe alındığında, içerisindeki spesifik bir satıra (tuple) erişim, satır/sayfa okuması mantığıyla yürütülür. src/backend/storage/page/bufpage.c içerisindeki PageGetItem fonksiyonu, sayfa başındaki "Line Pointer" dizisini kullanarak verinin sayfa içindeki ofsetine (offset) O(1) karmaşıklığında ulaşır. Bu yöntem, veri parçalansa (fragmentation) bile erişim hızının korunmasını sağlar.

**Bellek Yönetimi (Buffer Pool ve Clock Sweep)** Disk erişim maliyetini düşürmenin en etkili yolu, sık kullanılan sayfaların RAM’de tutulmasıdır (Caching). Ancak RAM sınırlı bir kaynaktır. PostgreSQL, hangi sayfanın bellekte kalıp hangisinin atılacağına karar vermek için "Clock Sweep" (bir tür LRU türevi) algoritmasını kullanır. src/backend/storage/buffer/freelist.c dosyasındaki StrategyGetBuffer fonksiyonu incelendiğinde, sistemin bir "saat" gibi buffer havuzunu taradığı, usage_count değeri 0 olan sayfaları bulup diske tahliye ettiği (eviction) ve yeni sayfaya yer açtığı görülür. Bu algoritma, sık erişilen verilerin bellekte kalmasını garanti altına alarak disk I/O sayısını minimize eder.

**Veri Yapıları: Heap ve Index Ayrımı** PostgreSQL, verinin fiziksel olarak saklandığı yapı (Heap) ile veriye hızlı erişimi sağlayan yapıyı (Index) birbirinden ayırır. Verinin kendisi src/backend/access/heap/heapam.c dosyasındaki yöntemlerle (Heap Access Method) sırasız bir yığın olarak saklanırken; arama işlemleri için B-Tree gibi dengeli ağaç yapıları kullanılır. src/backend/access/nbtree/nbtsearch.c içerisindeki _bt_search fonksiyonu, B-Tree üzerinde kökten yaprağa doğru inerek aranan anahtarın bulunduğu sayfanın ID'sini (TID) döndürür. Bu ayrım, aynı veri üzerinde birden fazla indeks tanımlanabilmesine olanak tanır ancak her yazma işleminde hem Heap'in hem de indekslerin güncellenmesi maliyetini doğurur.

**Veri Bütünlüğü ve WAL (Write Ahead Log)** Performans kadar kritik olan bir diğer konu veri dayanıklılığıdır (Durability). Sistem çökmelerine karşı PostgreSQL, WAL (Write Ahead Log) ilkesini benimser. Veri sayfaları diske yazılmadan önce, yapılan değişikliğin kaydı src/backend/access/transam/xloginsert.c dosyasındaki XLogInsert fonksiyonu ile sıralı (sequential) bir log dosyasına yazılır. Rastgele disk yazma işlemi (random write) maliyetli olduğundan, logların sıralı yazılması performansı artırır ve fsync çağrılarıyla verinin kalıcılığı garanti altına alınır.

Sonuç olarak; PostgreSQL'in kaynak kodları incelendiğinde, veritabanının sadece bir veri deposu olmadığı; disk bloklarını yöneten, özel bellek tahliye algoritmaları çalıştıran ve karmaşık veri yapılarını disk üzerinde organize eden sofistike bir sistem yazılımı olduğu görülmektedir.

## VT Üzerinde Gösterilen Kaynak Kodları

Blok bazlı disk erişimi [Linki](https://github.com/postgres/postgres/blob/master/src/include/storage/bufpage.h) \
Satır okuması [Linki](https://github.com/postgres/postgres/blob/master/src/backend/storage/page/bufpage.c) \
Buffer pool clock [Linki](https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/freelist.c) \
Heap yapısı [Linki](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/heapam.c) \
İndex yapısı [Linki](https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/nbtsearch.c) \
WAL ilkesi [Linki](https://github.com/postgres/postgres/blob/master/src/backend/access/transam/xloginsert.c)
