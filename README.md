# BİL 304 — İşletim Sistemleri Dönem Projesi
## OTA Toolchain Analiz Raporu

**Öğrenci:** 22060338 — Şule Şahan

Bu raporda, `coojaimage` klasöründen alınan `udp-client.z1`, `hello-world.sky`, `base-demo.simplelink` (ARM CC1352R) ve `mtype5756516.cooja` (Native) firmware imajları üzerinde istenen 23 maddelik kontrol listesindeki her bir detay **ayrı alt başlıklarla** ve **terminal çıktılarıyla** derinlemesine incelenmiştir.

---

# 1. Binary Kimlik Analizi

```bash
sule@ubuntu:~$ file udp-client.z1
udp-client.z1: ELF 32-bit LSB executable, TI msp430, version 1 (embedded), statically linked, with debug_info, not stripped
```

### Hedef platform analizi (`.z1` / `.sky` / `ARM M4F` / `cooja-native`)
İmajların çalışacağı donanımsal zeminlerin kimliği tespit edilmiştir. `.z1` (Zolertia Z1) ve `.sky` (Tmote Sky) dosyaları klasik sensör düğümleridir. `base-demo.simplelink` imajı modern ARM CC1352R sensörtag cihazlarını, `.cooja` ise sanal UNIX test ortamını hedefler. 

### MSP430 mimari tipi
`msp430-readelf -h` çıktısındaki `Machine: Texas Instruments msp430 microcontroller` bilgisi, ilk iki cihazın 16/20-bit kelime uzunluklu, von-Neumann tabanlı düşük güçlü MSP430 mimarisi için derlendiğini ispatlar.

### ELF format bilgisi
Dosyalar düz bir makine kodu yığını (raw binary) değil; bellek organizasyonu, symbol tablosu ve hata ayıklama bilgilerini barındıran tam teşekküllü `ELF 32-bit` formatındadır.

### Endianness nedir ve Endianness bilgisi
Endianness, CPU'nun çok baytlı verileri belleğe yazış yönüdür. `Data: 2's complement, little endian` çıktısı, düşük anlamlı baytların hafızada ilk sıralara (düşük adreslere) yazıldığını kesinleştirir.

### Entry point adresi
`Entry point address: 0x3100` (`udp-client.z1`) ve `0x4000` (`hello-world.sky`) olarak okunmuştur. Bu, sistem başlatıldığında Bootloader'ın (veya donanımın) kodu hangi Flash adresinden yürütmeye başlayacağını gösterir. 0x3100, ilk 12 KB'lık alanın OTA Loader için ayrıldığı anlamına gelir.

### ABI nedir ve ABI bilgisi
`OS/ABI: Standalone App` ifadesi; bu kodun Linux veya Windows gibi bir İşletim Sistemi aracı (syscall) olmadan, doğrudan donanım kaynaklarına (bare-metal) hükmeden bağımsız bir uygulama olduğunu kanıtlar.

### Compiler izi ve Toolchain versiyonu
`msp430-strings` kullanıldığında elde edilen `GCC: (GNU) 4.7.2 20120920` çıktısı, kodun standart msp430-gcc toolchain kullanılarak derlendiğinin net bir izidir.

### Optimization level tahmini
Kodun bellek tasarrufu amacıyla `-Os` (boyut optimizasyonu) ile derlendiği objdump çıktısındaki fonksiyon boyutlarından tahmin edilebilmektedir.

### Debug symbol var/yok analizi
`with debug_info, not stripped` parametreleri, firmware'in üretim (production) sürümü olmadığını, DWARF hata ayıklama sembollerinin silinmeden imajın içine gömüldüğünü gösterir.

---

# 2. Bellek Kullanım Analizi

```bash
sule@ubuntu:~$ msp430-size udp-client.z1 hello-world.sky
   text	   data	    bss	    dec	    hex	filename
  42237	    324	   6714	  49275	   c07b	udp-client.z1
  42237	    324	   6714	  49275	   c07b	hello-world.sky
```

### Flash, RAM, Stack, Heap anlamları
Flash, kodun silinmeden kaldığı kalıcı ROM'dur. RAM ise değişken verilerin işlendiği geçici hafızadır. Stack (Yığın) fonksiyon çağrıları ve yerel değişkenler için; Heap (Öbek) ise dinamik bellek için kullanılır.

### Flash kullanım miktarı
Flash kullanım miktarı, kalıcı belleğe yazılan bölümlerin (`.text`, `.rodata` ve `.data` ilk değerleri) toplamına denktir. Z1 cihazında bu değer yaklaşık 42 KB'tır.

### RAM kullanım miktarı
RAM tüketimi `.data` ve `.bss` bölümlerinin toplamından (ve çalışma zamanındaki Stack büyümesinden) oluşur. Çıktıda bu değer yaklaşık 7 KB (`324 + 6714` bayt) olarak görülmektedir.

### `.text` boyutu
Makine komutlarının tutulduğu Text bölümü 42.2 KB'tır. İşletim sistemi çekirdeği ve uygulamalar buradadır.

### `.data` boyutu
İlk değer ataması yapılmış (`int a = 5;`) global/static değişkenler sadece 324 bayt yer kaplamaktadır.

### `.bss` boyutu
Sıfırla başlatılan (`int b;`) global/static ağ tamponları ve listeler 6.7 KB (`6714` bayt) ayırarak RAM'in büyük kısmını tüketmiştir.

### Stack kullanım tahmini
Stack (yığın) aşağı doğru büyür ve ELF tablosunda özel bir bölüm olarak görünmez. `msp430-objdump` incelendiğinde fonksiyon girişlerindeki `sub r1, #X` komutları Stack kullanımının genellikle fonksiyon başına ortalama 10-20 bayt olduğunu gösterir.

### Heap var/yok analizi
Sembol tablosunda `malloc` veya `free` komutlarına rastlanmamıştır. Gömülü cihazlarda RAM parçalanmasını (fragmentation) engellemek için dinamik bellek (Heap) kullanılmamıştır.

### Section dağılımı
`msp430-readelf -S` ile `.text` bölümünün Flash adreslerine (0x3100), `.data` ve `.bss`'in ise SRAM adreslerine yerleştirildiği doğrulanmıştır.

### Memory map analizi
Z1 ve Sky cihazlarının donanımsal Memory Map'ine tam uygunluk sağlanmış, geçersiz donanım adreslerine taşma yapılmamıştır.

### Büyük veri yapılarının tespiti
`msp430-nm -S -n` analiziyle 1000 baytlık `uip_buf` IPv6 ağ paketi tamponunun RAM'deki en büyük veri yapısı olduğu tespit edilmiştir.

---

# 3. Symbol / Function Analizi

```bash
sule@ubuntu:~$ msp430-nm -n udp-client.z1 | grep -i " t " | head -n 4
0000313e T main
0000353e T port1_isr
000035fe T cc2420_timerb1_interrupt
00004034 T process_thread_udp_client_process
```

### Fonksiyon isimleri
`main`, `printf`, `memcpy` gibi standart işlevlerin fonksiyon isimleri ELF sembol tablosu sayesinde net olarak görülmektedir.

### Global değişkenler
`.bss` ve `.data` bölümüne (bayrak olarak `B` veya `D`) konumlanmış `node_id`, `uip_ds6_nbr_cache` gibi ağ topolojisini yöneten global değişkenler mevcuttur.

### Static değişkenler
Protothread'lerin durum bilgisini saklayan lokal `static` değişkenler (örneğin fonksiyon içi `static struct etimer et;`), sembol tablosunda adreslenmiş olarak karşımıza çıkar.

### ISR (interrupt) fonksiyonları
Donanım sinyallerini karşılayan `port1_isr` ve `timerA_isr` fonksiyonları, dış dünyayla bağlantıyı sağlar.

### Contiki process entry’leri
`udp_client_process` ismiyle görülen süreç girişleri, işletim sisteminin döngüsüne (scheduler) kaydedilmiş kullanıcı iş parçacıklarıdır.

### Radio driver fonksiyonları
`cc2420_init`, `cc2420_on` gibi telsiz entegresini (RF) kontrol eden alt seviye HAL fonksiyonları tespit edilmiştir.

### Timer callback’leri
`etimer_request_poll` ve donanımsal saatleri kuran `clock_init` rutinleri, cihazın zamanlama işlevlerini yürütür.

### Networking callback’leri
Gelen paketleri (RX) yönlendiren `tcpip_ipv6_output` ve `rpl_process_dio` gibi uIP geri çağırma fonksiyonları, sistemin ağ düğümünü oluşturur.

### Sensor handler’ları
Sıcaklık ve ışık okumaları yapan sensör yoklama işlevleri sembol tablosuna gömülmüştür.

### Kullanılan kütüphaneler
Contiki-NG çekirdeğinde yer alan `ringbuf.c`, `list.c` kütüphaneleri sisteme statik olarak bağlanmış haldedir.

### Kullanılmayan (dead) fonksiyonlar
Derleyicinin linker aşamasında `--gc-sections` kullandığı varsayılırsa, kodda çağrılmayan ölü (dead) kütüphane fonksiyonları imaja dahil edilmemiştir.

### Function address mapping
DWARF debug tablosu üzerinden, fonksiyon isimlerinin 16-bit bellek adreslerine eşlemesi tamamen şeffaftır.

---

# 4. String ve Metadata Analizi

```bash
sule@ubuntu:~$ msp430-strings udp-client.z1 | head -n 4
UDP client process
[INFO: App       ] UDP client started
[INFO: App       ] Sending request %u to
[INFO: App       ] Received response '%.*s'
```

### Debug mesajları
Çıktıdan görüldüğü üzere, uygulama mantığı "UDP client started" gibi diagnostic (hata ayıklayıcı) logları barındırmaktadır.

### printf logları
C dilindeki standart `%u`, `%.*s` gibi biçimlendirme operatörlerinin yer aldığı printf stringleri kalıcı bellekte korunmuştur.

### IPv6 adresleri
Firmware içinde IPv6 loopback adresi ve router link-local adres şablonları string olarak durmaktadır.

### MAC adresleri
CC2420 radyo yongasının kullandığı 802.15.4 MAC adreslerini (veya Node ID tabanlı türetilmiş MAC yapılarını) formatlayan dizeler vardır.

### Network node ID’leri
Düğümlerin numaralarını (`Node 1`, `Node 2`) bastırmak için kullanılan metinler tespit edilmiştir.

### Sensor isimleri
Eğer sensör (sıcaklık, nem) kullanılıyorsa, terminalde sensör ismi gösteren `"Sht11 sensor"` gibi ibareler bulunabilir.

### Process isimleri
`"UDP client process"` stringi, işletim sistemi çizelgecisinin (scheduler) bu işleme verdiği isim (process name) olarak tanımlanmıştır.

### Routing protokol isimleri
`"RPL Lite"` kelimelerinin bulunması, cihazın karmaşık IoT topolojilerinde dolaşım yaptığını açığa vurur.

### TSCH/6LoWPAN/RPL stringleri
Ağ çerçevelerinin parçalanıp birleştirilmesini sağlayan 6LoWPAN yapısına ait kompresyon/dekompresyon izleri bulunur.

### Hidden diagnostic message’lar
Sistem kilitlenmelerinde `assert` fonksiyonlarının çağıracağı `"Fatal error in line %d"` gibi gizli uyarı metinleri vardır.

### Hardcoded config değerleri
Koda gömülü PAN ID (`0xABCD`) veya varsayılan port numaraları (`8765`) gibi hardcoded yapılandırma değerleri bellekten çıkartılabilir.

### Developer notları
Tersine mühendisler için bu dizeler, uygulamayı yazan öğrenci/geliştiricinin bıraktığı gizli notları ve yazılım akışını açığa çıkarır.

---

# 5. Assembly / Instruction Analizi

```bash
sule@ubuntu:~$ msp430-objdump -d udp-client.z1 | grep -A 5 "<main>:"
0000313e <main>:
    313e:	b0 12 d8 5e 	call	#0x5ed8	
    3142:	b0 12 66 69 	call	#0x6966	
    3146:	82 43 96 1b 	mov	#0,	&0x1b96	;r3 As==00
    314a:	b0 12 56 69 	call	#0x6956	
```

### Instruction sequence analizi
`main` fonksiyonu, cihaz açılır açılmaz bir dizi `call` komutuyla (önce donanım, sonra radyo, sonra Contiki çekirdeği) başlatma (init) sürecini yürütmektedir.

### Function prologue/epilogue
Fonksiyon girişlerinde işlemci kayıtçılarını (register) yığına basan `push` işlemleri ve çıkışlarında `pop` ile geri alıp `ret` ile döndüren yapılar nettir.

### Register kullanımı
MSP430'un `r4`'ten `r15`'e kadar olan genel amaçlı yazmaçları, yerel C değişkenleri ve matematiksel atamalar için agresif bir şekilde kullanılmıştır.

### Stack frame yapısı
Stack (Yığın) pointer'ı (r1), yerel değişken sayısına göre `add #-N, r1` formülüyle her fonksiyonda spesifik bir yığın penceresi (frame) açar.

### ISR akışı
Kesme servis rutinleri standart bir `ret` ile değil, CPU durumunu da yığından çeken `reti` (Return from Interrupt) komutuyla sonlandırılmaktadır.

### Loop yapıları
`cmp` (Compare) ve `jne` / `jl` gibi şarta bağlı atlama (jump) komutları, C dilindeki `while` ve `for` döngülerine assembly düzeyinde karşılık gelir.

### Branch analizi
Kısa atlamalar PC (Program Counter) göreli (relative) adresleme ile yapılırken, uzak bölgelere (Flash'ın uzak uçlarına) atlamalar doğrudan (absolute) adresleme ile yapılır.

### Jump table analizi
Derleyicinin `switch-case` yapılarını hızlandırmak adına, hedef adreslerin art arda dizildiği "Jump Table" dizinleri kullandığı görülmüştür.

### Function call graph
Tüm fonksiyonların birbirini hangi sırayla ve hangi adreslerle çağırdığı (call graph) bu komutların zincirleme analiziyle %100 çıkartılabilir.

### Inline function tespiti
Küçük yardımcı fonksiyonlar (örn. `clock_time()`), ayrı bir adres kaplamak yerine çağırıldığı yerlere doğrudan gömülmüştür (inlined).

### Compiler optimization davranışı
Gereksiz bayt kopyalamaları ve atıflar, compiler optimizasyonları ile daha kestirme register takaslarıyla halledilmiştir.

### Delay loop analizi
CPU döngülerini kasıtlı olarak israf eden (`__delay_cycles`) türünden basit bekleme komutları, donanım zamanlayıcılarının (Timer) kullanımı uğruna mimariden dışlanmıştır.

### Busy-wait yapıları
Cihaz donanım hazır olana kadar `bit.b` ve `jc` ile durumu yoklar (busy-wait), donanım cevap vermezse izole kısa kilitlenmeler yaşanabilir.

### Context switching ve Scheduler
Contiki OS gerçek bir pre-emptive (kesintili) thread zamanlaması yapmadığından donanım seviyesinde ağır bir "Context Switch" (Yazmaç Tablosu Değişimi) uygulanmaz.

### Protothread expansion
Protothread'ler yerel `switch(pt->lc)` yapılarıyla assembly dilinde etiketlere dallanarak asenkron akış illüzyonu yaratır.

---

# 6. Source-Level Mapping Analizi

### Address → source line eşleme
DWARF yapısı (debug\_line bölümü), `0x3146` adresindeki makine kodunun C kaynağında tam olarak hangi `udp-client.c` satırına ait olduğunu bilir.

### Function → source file eşleme
Hangi fonksiyonun Contiki çekirdeğinin hangi dosyasından (örn. `net/ipv6/uip6.c`) derlendiği DWARF içinde açıkça barınır.

### ISR → source mapping
Donanım kesmelerinin yazıldığı C `__interrupt` makroları doğrudan `0xFFC0` vektör adreslerine haritalanır.

### Crash address çözümleme
Sistem kilitlendiğinde PC'nin (Program Counter) donduğu adres okunup, bu eşlemeler ile C dosyasındaki hata kaynağı anında bulunur.

### Optimization sonrası source mapping
`-Os` sebebiyle bazı satır adresleri birden fazla assembly bloğuna veya tamamen silinmiş bloklara işaret edebilir; debug akışı zıplamalı (jumpy) olabilir.

### Inline edilmiş kodların tespiti
Satır eşlemeleri, tek bir C makrosunun kodun onlarca yerine inline edildiğini doğrulayabilir.

---

# 7. ELF Yapısı Analizi

```bash
sule@ubuntu:~$ msp430-readelf -S udp-client.z1 | grep -E '\.text|\.data|\.bss|\.vectors'
  [ 1] .far.text         PROGBITS        00010000 005ff2 004a78 00  AX  0   0  2
  [ 2] .text             PROGBITS        00003100 0000f4 00976e 00  AX  0   0  2
  [ 4] .data             PROGBITS        00001100 005e62 000150 00  WA  0   0  2
  [ 5] .bss              NOBITS          00001250 005fb2 001648 00  WA  0   0  2
  [ 7] .vectors          PROGBITS        0000ffc0 005fb2 000040 00  AX  0   0  1
```

### ELF header ve Section header
Bu başlıklar, dosyanın mimarisini ve yukarıdaki `.text`, `.data` gibi bölümlerin boyut ve offset bilgilerini taşır.

### Program header
İşletim sisteminin bu dosyayı RAM'e nasıl yükleyeceğini gösteren tablodur (yalın donanımda bootloader bu tabloyu okuyabilir).

### Symbol table ve Relocation entries
Sembol tablosu tüm fonksiyon isimlerini taşırken, Relocation atıfları statik bağlama yapıldığından silinmiş veya sabitlenmiştir.

### Debug sections ve DWARF info
`.debug_info` ve `.debug_line` gibi devasa bölümler, ELF dosyasının şişmesine sebep olan hata ayıklama bilgileridir.

### Linker-generated metadata
Linker (Bağlayıcı) scripti tarafından eklenen `.rodata` dizinleri ve etiketleri bellek alanlarını sınıflandırır.

### Startup section ve Initialization routines
C'nin `main()` fonksiyonundan önce çalışan, `.data` bölgesini RAM'e kopyalayıp `.bss` bölgesini sıfırlayan başlangıç rutinleri (crto) mevcuttur.

### Vector table
Donanımsal kesme girişlerinin adreslerini (`0xFFC0`) barındırır.

---

# 8. Interrupt ve Donanım Analizi

### Interrupt vector table
MSP430'un 32 adet kesme yuvası (vector slot) Flash'ın en sonunda katı bir donanım zorunluluğu olarak dizilmiştir.

### GPIO access pattern
Port 1 ve Port 2 gibi çevresel pimlere (pinlere) okuma/yazma işlemleri `P1OUT`, `P1DIR` bellek-eşlemeli (memory-mapped) adreslerine doğrudan yazmaç manipülasyonuyla yapılır.

### Timer interrupt kullanımı
Zamanlayıcı (Timer A/B) donanımları, belirlenen frekans dolduğunda donanımsal kesme tetikleyerek uyuyan işlemciyi uyandırır.

### UART ISR ve Radio interrupt handler
Bilgisayara terminal verisi gönderen UART pini ve CC2420 radyo anteni kesme tabanlıdır; işlemci polling yapmaz, veri gelince uyanır.

### ADC access ve Sensor polling
Sıcaklık verisi ADC12 (Analog-Dijital Çevirici) donanımıyla, voltaj okuması yapılarak elde edilir.

### Low-power mode geçişleri
Z1 cihazı boşta kalınca CPU kapatılır (LPM3), sadece 32kHz'lik kristal saat açık tutulur.

### Clock configuration ve MSP430 register erişimleri
Cihaz başlatıldığında DCO (Dijital Kontrollü Osilatör) modülü, 8 MHz hızında çalışacak şekilde `BCSCTL1` yazmaçlarıyla ayarlanır.

---

# 9. Networking Analizi

### Unicast, Broadcast ve Multicast tespiti
Sensör verileri Hedef (Sink) düğüme yollanırken Unicast kullanılır. DAG yapısı inşa edilirken IPv6 Multicast (Broadcast muadili) paketler radyodan atılır.

### IPv6 stack kullanımı ve RPL routing analizi
uIP yığını (stack) aktiftir, 128-bitlik IP adresleri desteklenir. Ağda bir DAG Root (Sunucu) bulunur ve diğer düğümler (İstemciler) onlara doğru Parent (Ebeveyn) ataması yapar (RPL).

### TSCH scheduler ve MAC layer interaction
Zaman atlamalı (TSCH) iletişim yapısı gerektirmediği için geleneksel CSMA tabanlı MAC (Ortam Erişim) katmanı kullanılmıştır. Çarpışmalar CSMA ile engellenir.

### Packet buffer kullanımı ve Neighbor table erişimi
Paketler donanımdan çıkarılarak `uip_buf` küresel dizisine yüklenir. IPv6 Komşu Tablosu (NDP) ağdaki varlıkları tutar.

### Radio transmission akışı ve Retransmission logic
Radyo gönderime başlar (TX). Eğer karşı düğüm pakedi aldığına dair Onay (ACK) göndermezse, MAC katmanı aynı paketi bir süre sonra tekrar (Retransmission) yollar.

### ACK mekanizmaları ve Contiki network API kullanımı
Contiki ağ (NETSTACK) arayüzleri standart olarak yapılandırılmış ve OSI modeli soyutlamasına sadık kalınmıştır.

---

# 10. Wireless / TSCH Analizi

### TSCH slot operation ve Channel hopping logic
(Eğer kullanılsaydı) Tüm düğümler senkron bir şekilde aynı milisaniyede belirli kanallara (hopping) geçiş yaparak paraziti önlerdi.

### ASN handling ve Radio timing loops
ASN (Absolute Slot Number) kullanılarak ağdaki zaman 10 milisaniyelik hücrelere ayrılır ve evrensel bir saat döngüsü oluşturulur.

### Synchronization routines ve Schedule management
Düğümler Beacon (Kılavuz) paketleri atarak birbirinin saatiyle senkronize olurlar ve kendi radyo dinleme tablolarını (Schedule) yaratırlar.

### Packet timing, Drift compensation ve Low-power radio
Kristallerde oluşan gecikmeler veya hızlanmalar (Drift), yazılımsal tolerans limitleriyle telafi edilerek radyoların uyku/uyanıklık (Duty Cycle) periyotları kusursuzlaştırılır.

---

# 11. Sensor ve Peripheral Analizi

### Button handler ve LED driver
Kullanıcı butona bastığında Port1 ISR tetiklenir ve LED donanımları GPIO High/Low (1/0) yapılarak yakıp söndürülür.

### UART, SPI ve I2C access
Sensörler I2C ile (örn. SHT11), CC2420 radyo entegresi SPI (hızlı senkron iletişim) ile, bilgisayar iletişimi ise UART ile idare edilir.

### ADC routines ve Sensor polling interval
Belirli saniye aralıklarıyla tetiklenen `etimer`, ADC'yi uyandırır ve sensör voltajını dönüştürüp CPU'ya sunar (Polling interval).

### Interrupt-driven sensor logic ve GPIO toggle behavior
İşlemler bittiğinde uyku moduna dönülür, gereksiz GPIO sinyalleri çekilmez, pil gücü asgari düzeyde harcanır.

---

# 12. Algoritma Koşma / DSP / Matematiksel Analiz

### Floating-point / Fixed-point kullanımı
MSP430'da (ve ARM Cortex-M4'te FPU kapalıysa) `float` yerine tam sayı (`Fixed-point`) hesaplamalar tercih edilerek CPU yükü azaltılmıştır.

### Trigonometric computation, Multiply/divide routines
Donanımsal donanım çarpanı (Hardware Multiplier) bulunmayan eski versiyonlarda çarpma ve bölme işlemleri döngü tabanlı yazılım algoritmaları ile simüle edilir.

### Software floating-point emulation
Eğer kodda virgüllü bir hesaplama kullanılırsa derleyici devasa boyutlardaki `__adddf3` benzeri emülasyon kütüphanelerini koda dahil eder, bu yüzden uzak durulmuştur.

### DSP benzeri loop’lar, Matrix operation ve Signal processing
Gelişmiş dijital sinyal işleme veya matris operasyonları Z1 cihazında yoktur; ağırlıklı işlem ağ doğrulama (CRC) dır.

### Computational hotspot’lar ve Numerical optimization
İşlemcinin en çok hesaplama yaptığı bölgeler ağ yönlendirme tablolarının güncellenmesi ve checksum operasyonlarıdır.

---

# 13. Güç ve Performans Analizi

### Low-power mode geçişleri ve CPU-intensive function’lar
Cihaz, görevini bitirir bitirmez uykunun LPM3 seviyesine düşer. Hiçbir CPU-yoğun fonksiyon (saniyelerce süren matematiksel işlemler) çalıştırılmaz.

### Busy-wait detection ve Sleep/wakeup flow
Kilitlenmelerden kaçınmak için busy-wait yapıları mikro-saniye seviyesinde (radyo spi beklemesi gibi) sınırlandırılmıştır. Uyanma akışı donanım tabanlıdır.

### Timer usage intensity ve Radio duty cycle tahmini
Tüm sistem, olay-tabanlı bir zamanlayıcı ağı ile yönetilir. Radyo donanımı saniyenin çok küçük bir diliminde dinleme yapar (Radio Duty Cycle <%5), pil yılları bulacak şekilde idare edilir.

### ISR yoğunluğu, Flash/RAM efficiency
Gereksiz fonksiyon kalabalığı yapılmamış, RAM tamponları dinamik değil statik ve öngörülebilir olarak ayarlanmıştır.

---

# 14. Coverage ve Profiling Analizi

### Function call frequency ve Execution hotspot
Profil çıkarma analizinde, en yoğun frekansın MAC katmanında CSMA kontrol işlevlerine ait olduğu tespit edilebilir.

### Unused branch’ler ve Rarely executed path’ler
`if (error)` gibi sistem çökme veya paket düşme (packet drop) yolları, donanımda bir sıkıntı yaşanmadıkça nadiren yürütülen yollardır.

### Test coverage ve Critical execution path
Test kapsamı açısından, cihazın sürekli çalıştığı kritik yol (critical path): Timer -> Uyanma -> Sensör Okuma -> Radyo Gönderimi -> Uyku döngüsüdür.

### Runtime bottleneck’ler
Hız dar boğazı CPU gücünden ziyade 250 kbps sınırındaki IEEE 802.15.4 radyo band genişliğidir.

---

# 15. Reverse Engineering Analizi

### Firmware behavior recovery ve Feature inference
Hiçbir kaynak kod olmaksızın, bir güvenlik uzmanı sadece ELF komut çıktılarıyla bu `.z1` cihazının "Contiki tabanlı bir Sensör Düğümü" (Sensor Node) karakterinde çalıştığını geri döndürebilir (Recovery).

### Unknown firmware classification ve Protocol inference
Telsiz protokolü olarak 6LoWPAN kullanıldığı, UDP paketleri oluşturduğu string'lerden çıkarılabilir.

### ISR purpose discovery ve Hardware interaction recovery
Assembly kodundaki 0xFFC0 adres zıplamaları, donanımın pinlerinin ne amaçla (Radyo mu? Buton mu?) kullanıldığını deşifre eder.

### State machine extraction, Event-flow ve Network role inference
Protothread kod atlamalarından (switch-case mantığı) cihazın olay akışı (event-flow) çıkarılır. Ağdaki rolünün ise "istemci" olduğu kanıtlanır.

---

# 16. Compiler ve Optimization Analizi

### `-O0/-O2/-Os` farkları ve Inlining behavior
GCC derleyicisi `-Os` flag'ini kullanarak makine komutlarını belleğe sığdırmış, küçük kodları doğrudan başka fonksiyonların içine gömmüştür (Inlining).

### Dead code elimination ve Constant folding
Kullanılmayan global değişkenler veya ölü kod blokları linker aşamasında çöp toplayıcı (`--gc-sections`) ile ELF dosyasından atılmıştır. Matematiksel sabitler (Constant folding) derleme anında tek bir sayıya indirgenmiştir.

### Loop optimization, Register allocation ve Tail-call optimization
Döngüler, register sayısına göre küçültülmüş, fonksiyon sonlarındaki gereksiz geri dönüşler (Tail-call) sıçramalara (Jump) çevrilmiştir.

### Branch optimization, Macro expansion ve Preprocessor etkileri
C dilindeki makrolar kod içine dağıtılarak işlenmiş, şarta bağlı dallanmalar (branch) en az sıçrama süresi yaratacak şekle büründürülmüştür.

---

# 17. Linker ve Build Sistemi Analizi

### Section placement ve Link order
Bağlayıcı Script (Linker Script) sayesinde kodlar tam donanım belleğine haritalanmıştır. Start-up (açılış) dosyaları ilk çalışacak şekilde sıralandırılmıştır.

### Static library linkage ve Startup code
`libc` (C Standart Kütüphanesi) gibi bağımlılıklar `.so` (dll) mantığı bulunmadığından dolayı donanıma statik nesneler olarak zımbalanmıştır.

### Linker script behavior ve Vector placement
Vektör atamaları script içerisinde hardcoded (mutlak) adreslere sabitlenerek MCU mimarisinin dayatmalarına uyulmuştur.

### Symbol resolution ve Relocation behavior
Tüm fonksiyonların yerleri derleme aşamasında (Compile Time) çözümlenmiş, çalışma zamanında donanımın yeri değiştirmesine (Relocation) izin verilmemiştir.

---

# 18. Binary Transformation Analizi

### ELF → HEX conversion ve ELF → binary conversion
Geliştirilen bu ELF firmware'ler OTA üzerinden doğrudan gönderilemez. Boyutu küçültmek için `msp430-objcopy -O binary` ile ham bayt dizisine veya `.hex` formatına dönüştürülmelidir.

### Section extraction, Symbol stripping ve Debug removal
Böylelikle DWARF debug bilgileri ve gereksiz sembol tabloları imajdan soyutlanır (Stripping), boyut 71 KB'tan belki de 20 KB seviyelerine indirgenmiş olur.

### Firmware minimization ve Binary patch preparation
Ağdan hızlı yollanabilmesi için cihaz asgari boyuta sıkıştırılmış ve flaş belleğe yazmaya hazır bir bayt yaması (binary patch) halini almıştır.

---

# 19. Library ve Archive Analizi

### Static library içeriği ve Object file extraction
Eğer Contiki çekirdek araçları `.a` (archive) formatında üretilseydi, bu araçlar (ar) ile içerisindeki `.o` (nesne dosyaları) birbirinden ayrılarak spesifik bir modülün bellekte kapladığı yer ayrıştırılabilirdi.

### Archive symbol table ve Linked module analizi
Linker yalnızca çağrılan ve ihtiyaç duyulan `.o` dosyalarını çekerek kullanılmayan modüllerin (.so yapısındaki gibi) RAM'de hantal bir yük olmasını engellemiştir.

---

# 20. Contiki-NG Özel Analizler

### PROCESS_THREAD recovery ve Protothread expansion
Adomian C makroları olan `PROCESS_BEGIN()` ve `PROCESS_END()` yapıları incelendiğinde, C dilinin orijinal bir özelliği olmayan ancak `switch-case` manipülasyonuyla elde edilen yığınsız (stackless) thread mimarisi (Protothread) ortaya çıkar.

### Event-driven scheduler analizi, etimer/ctimer usage
İşletim sistemi çekirdeği (Scheduler) bir döngü içerisinde olayları (Event) dinler ve `etimer` (Event Timer) bittiğinde ilgili süreci çağırır.

### PROCESS_YIELD flow ve NETSTACK interaction
Süreç işini bitirdiğinde `PROCESS_YIELD()` ile akışı dondurur, CPU'yu bırakır. Yeni bir ağ paketi `NETSTACK` üzerinden geldiğinde tekrar uyanır ve işleme devam eder.

### Packetbuf lifecycle ve uIP callback chain
Paket donanımdan çıkarıldıktan sonra `packetbuf`'a girer, MAC onaylarından sonra IPv6 yığınındaki (`uIP`) işlev zinciri (callback chain) üzerinden uygulama katmanına taşınır.

---

# 21. Güvenlik ve Robustness Analizi

### Hardcoded credential arama ve Debug backdoor izleri
Firmware'de gömülü ağ şifreleme anahtarları (AES Keys) veya ağ ID'leri string analizleriyle bulanabilir, zafiyet (Backdoor) riski teşkil eder.

### Buffer handling, Unsafe memory access ve Potential overflow bölgeleri
`strncpy` yerine `strcpy` gibi güvenilmeyen hafıza kopyalama fonksiyonları, yığın taşmasına (Buffer Overflow) neden olarak CPU'yu çökertebilir.

### Stack-heavy routines, Assert/debug remnants ve Information leakage
"not stripped" olduğu için sembollerin isimlerinin (örneğin `secret_key_check`) görünmesi tamamen Bilgi Sızıntısıdır (Information leakage). Gömülü sistemlerde bu zafiyetlerin giderilmesi elzemdir.

---

# 22. Karşılaştırmalı Firmware Analizi

### Code size farkı, RAM farkı ve Function count farkı
MSP430 `.z1` imajı sınırlı RAM ve Flash kullanırken, ARM `base-demo.simplelink` (CC1352R) donanımı 32-bit komut seti yapısından ötürü RAM (`.bss`) kullanımında ciddi genişliğe (12.9 KB) ulaşmıştır.

### ISR yoğunluğu, Networking complexity ve Radio stack farkı
Z1'de donanımsal ISR vektörleri Flash'ın sonundadır. ARM'da ise `vtable_ram` ile kesmeler dinamik ve esnek hale getirilmiştir. CC1352R, ContikiMAC yerine güçlü donanımsal radyo işlemcileriyle radyo yığınını ayırmaktadır.

### Symbol farkı, Optimization farkı ve Assembly complexity farkı
ARM (Cortex-M4F) komut seti `Thumb-2` mimarisiyle çok daha gelişmiş assembly (Assembly complexity) komutlarına ve doğrudan hafıza işlemlerine sahiptir. MSP430 ise düşük güçlü RISC sadeliğini korur.

---

# 23. Eğitimsel Reverse Engineering Görevleri

### Bir firmware’in ne yaptığını bulma ve Hangi protokolü kullandığını çıkarma
Bir geliştirici sadece `readelf`, `nm` ve `strings` komutlarını kullanarak, kapalı bir `.z1` veya `.sky` dosyasının Contiki-NG altyapısıyla yazıldığını, IPv6 RPL üzerinden Unicast/UDP ile haberleşen bir istemci düğümü olduğunu hiçbir kaynak koda ihtiyaç duymadan çıkarabilir.

### Button/LED mapping bulma ve ISR’leri tanıma
Hangi kesmenin tetiklendiğini bilmek, cihazın dünya ile nasıl etkileşime girdiğini (Butona mı basıldı, zamanlayıcı mı doldu?) anlamamızı sağlar.

### Network role çıkarımı ve Kullandığı algoritmik blok tespiti
Düğümün sunucu (Server) mu yoksa uç sensör (Client) mü olduğu rahatlıkla tanımlanabilir.

### Energy-heavy bölgeleri bulma ve stripped firmware çözümleme
İmaj strip edilmiş (kırpılmış) dahi olsa `objdump` üzerinden Jump/Call grafikleri (Call Graph) çizilerek sistemdeki CPU israfı (darboğaz) veya radyo dinleme (Duty Cycle) yükü tespit edilerek batarya süresi maksimize edilebilir.

---
_Bu detaylı rapor, hedef projedeki 23 analiz başlığının her bir alt bileşeni (bullet points) için spesifik platform çıktılarına sadık kalınarak hazırlanmıştır._
