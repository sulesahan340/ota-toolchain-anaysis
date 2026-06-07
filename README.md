# MSP430 `.z1` / `.sky` / `ARM M4F(CC1352R)` / `cooja-native` Platformları için Üretilmiş Firmware’ler Üzerinde Yapılabilecek Analiz Türleri Kontrol Listesi

---
##### (* ARM Mimarisinde derlenmiş firmware analizi yapmak isteyen gruplar MSP430 Toolchain yanında ARM-Toolchain araçlarını da indirip, kullanmalıdırlar.)

``` bash
  $ wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-rm/9-2020q2/gcc-arm-none-eabi-9-2020-q2-update-x86_64-linux.tar.bz2
  $ tar -xjf gcc-arm-none-eabi-9-2020-q2-update-x86_64-linux.tar.bz2
```
---

# 1. Binary Kimlik Analizi

* Hedef platform analizi (`.z1` / `.sky` / `ARM M4F(CC1352R)` / `cooja-native`)
* MSP430 mimari tipi
* ELF format bilgisi
* Endianness nedir ve Endianness bilgisi
* Entry point adresi
* ABI nedir ve ABI bilgisi
* Compiler izi
* Toolchain versiyonu
* Optimization level tahmini
* Debug symbol var/yok analizi

Araçlar:
* `msp430-readelf`
* `msp430-objdump`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
```bash
sule@ubuntu:~$ file new-firmware.z1 hello-world.sky base-demo.simplelink mtype5756516.cooja
new-firmware.z1:      ELF 32-bit LSB executable, TI msp430, statically linked, not stripped
hello-world.sky:      ELF 32-bit LSB executable, TI msp430, statically linked, not stripped
base-demo.simplelink: ELF 32-bit LSB executable, ARM, EABI5, statically linked, not stripped
mtype5756516.cooja:   ELF 64-bit LSB shared object, x86-64, dynamically linked, not stripped
```

Z1 ve Sky platformlarına ait derlemeler 32-bitlik, statik olarak bağlanmış (statically linked) ve "Standalone App" (yalın donanım) ABI'sine sahip ELF çalıştırılabilir dosyalarıdır. `base-demo.simplelink` dosyası ise CC1352R LaunchPad için derlenmiş, ARM mimarisine sahip bir 32-bit ELF dosyasıdır. Native Cooja simülasyonu için oluşturulan `.cooja` uzantılı dosya ise Linux x86_64 mimarisinde dinamik linklenmiş bir paylaşımlı obje (Shared Object) olarak derlenmiştir. Tüm platformlar little-endian byte dizilimi kullanır. `new-firmware.z1` imajının `Entry point address` değeri `0x3100` olarak görülmektedir. Bu, linker dosyasının bu firmware'i doğrudan `0x0000` yerine bootloader payı (ilk 12 KB) bırakılarak `0x3100` adresinden flash belleğe yazılacak şekilde hizaladığını ifade eder. İmajlar debug sembollerinden arındırılmamıştır (not stripped).

---

# 2. Bellek Kullanım Analizi

* Flash, RAM, Stack, Heap anlamları
* Flash kullanım miktarı
* RAM kullanım miktarı
* `.text` boyutu
* `.data` boyutu
* `.bss` boyutu
* Stack kullanım tahmini
* Heap var/yok analizi
* Section dağılımı
* Memory map analizi
* Büyük veri yapılarının tespiti

Araçlar:
* `msp430-size`
* `msp430-readelf`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
```bash
sule@ubuntu:~$ size base-demo.simplelink mtype5756516.cooja new-firmware.z1
   text	   data	    bss	    dec	    hex	filename
  71393	   1408	  12968	  85769	  14f09	base-demo.simplelink
 324415	   9488	 292376	 626279	  98e67	mtype5756516.cooja
  71715	    336	   5706	  77757	  12fbd	new-firmware.z1
```

Bellek türleri cihazın donanımsal yeteneklerini doğrudan etkiler. Flash alanı yürütülebilir kodun (`.text`) saklandığı kalıcı bellektir. RAM ise çalışma anında değişken verilerin (`.data` ve `.bss`) tutulduğu uçucu bellektir. Çıktıdan görüldüğü üzere `new-firmware.z1` dosyasında `.text` boyutu 71 KB'a ulaşmıştır, çünkü içerisine 4096 baytlık `firmware_data.h` imajı ve ağ protokol yığını (IPv6/RPL) dahil edilmiştir. `new-firmware.z1` imajının `.bss` (başlangıç değeri atanmamış RAM değişkenleri) boyutu 5.7 KB iken, ARM mimarisindeki CC1352R'de 12.9 KB seviyesindedir. ARM cihazının çok daha geniş bir SRAM alanına sahip olması sebebiyle paket buffer'larının daha geniş tutulduğu görülmektedir. Cihazda dinamik bellek (Heap/malloc) tahsisine rastlanmamış, bellek yönetimi tamamen statik ve Stack üzerinden yürütülmüştür.

---

# 3. Symbol / Function Analizi

* Fonksiyon isimleri
* Global değişkenler
* Static değişkenler
* ISR (interrupt) fonksiyonları
* Contiki process entry’leri
* Radio driver fonksiyonları
* Timer callback’leri
* Networking callback’leri
* Sensor handler’ları
* Kullanılan kütüphaneler
* Kullanılmayan (dead) fonksiyonlar
* Function address mapping

Araçlar:
* `msp430-nm`
* `msp430-readelf`
* `msp430-objdump`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
```bash
sule@ubuntu:~$ msp430-nm -S -n new-firmware.z1 | head -n 8
00013e88 00000010 T printf
00013ef0 0000001c T snprintf
00014082 00000742 T vuprintf
0001484c 000000e8 T memcpy
00014934 000000e8 T memmove
```

Sembol tablosunda yer alan işlevler yazılımın yeteneklerini haritalandırır. `T` işareti bu sembollerin Flash bellekte yürütülen kod blokları olduğunu belirtir. Özellikle `vuprintf` fonksiyonu 1858 bayt (`0x0742`) alan kaplayarak sistemdeki en maliyetli ve ağır fonksiyonlardan birisi olduğunu kanıtlamıştır. Ayrıca donanım soyutlama (HAL) süreçlerine ait `cc2420_init` (Radio driver) ve ağ katmanına ait `udp_client_process` yapıları bellek adreslerine net bir şekilde haritalandırılmıştır.

---

# 4. String ve Metadata Analizi

* Debug mesajları
* printf logları
* IPv6 adresleri
* MAC adresleri
* Network node ID’leri
* Sensor isimleri
* Process isimleri
* Routing protokol isimleri
* TSCH/6LoWPAN/RPL stringleri
* Hidden diagnostic message’lar
* Hardcoded config değerleri
* Developer notları

Araçlar:
* `msp430-strings`
* `Ve üstteki aracın ARM versiyonu...`

**Uygulama ve Analiz:**
```bash
sule@ubuntu:~$ msp430-strings new-firmware.z1 | grep -E 'OTA|RPL|IPv6' | head -n 6
Tentative link-local IPv6 address: 
IPv6 addresses:
created a new RPL DAG
failed to create a new RPL DAG
IPv6 cache full, dropping DIO
dao_input: unknown RPL instance %u, discard
```

Firmware içerisinde yer alan string verileri, sistemin protokol özelliklerini ve gizli debug loglarını açıkça gösterir. Yukarıdaki analiz, cihazın 6LoWPAN uyumlu bir "RPL Lite" topolojisi çalıştırdığını, IPv6 protokol yığını ile iletişim kurduğunu ve hatta ağın şişmesi durumunda karşılaşılan hataları ("IPv6 cache full") raporladığını doğrular. Bu diagnostic mesajları, cihazın tersine mühendislik ile analiz edilmesinde kritik bilgiler barındırır.

---

# 5. Assembly / Instruction Analizi

* Instruction sequence analizi
* Function prologue/epilogue
* Register kullanımı
* Stack frame yapısı
* ISR akışı
* Loop yapıları
* Branch analizi
* Jump table analizi
* Function call graph
* Inline function tespiti
* Compiler optimization davranışı
* Delay loop analizi
* Busy-wait yapıları
* Context switching
* Protothread expansion
* Scheduler davranışı

Araçlar:
* `msp430-objdump`
* `msp430-as`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
```bash
sule@ubuntu:~$ msp430-objdump -d new-firmware.z1 | head -n 12
Disassembly of section .far.text:
00010000 <input>:
   10000:	7b 14       	pushm.a	#8,	r11	
   10002:	31 50 f2 ff 	add	#-14,	r1	;#0xfff2
   10006:	7f 40 0c 00 	mov.b	#12,	r15	;#0x000c
   1000a:	b0 13 c8 6a 	calla	#0x06ac8	
   1000e:	b0 13 d8 5e 	calla	#0x05ed8	
```

Düşük seviyeli komut akışında, fonksiyon prologue (giriş) adımında `pushm.a` kullanılarak yazmaçların (r11 vb.) yığına tek komutta yedeklendiği görülür. Ardından `add #-14, r1` ile yığın göstericisi (Stack Pointer) 14 bayt kaydırılarak yerel değişkenler için Stack Frame tahsis edilmiştir. Z1 platformunun 20-bit adresleme destekleyen MSP430X mimarisinden dolayı, uzak atlamalar için `call` yerine `calla` (Call Absolute) komutu üretilmiştir. Protothread yapısına bağlı kalınarak context switching olayları C dilindeki gelişmiş `switch-case` yapılarına derleyici tarafından tercüme edilmiştir.

---

# 6. Source-Level Mapping Analizi

(Debug build varsa)
* Address → source line eşleme
* Function → source file eşleme
* ISR → source mapping
* Crash address çözümleme
* Optimization sonrası source mapping
* Inline edilmiş kodların tespiti

Araçlar:
* `msp430-addr2line`
* `msp430-objdump -S`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Z1 ve Sky firmware dosyaları "not stripped" formatında bırakıldığı için DWARF hata ayıklama sembolleri (debug symbols) aynen korunmuştur. `msp430-addr2line -e new-firmware.z1 0x3100` aracı kullanıldığında, bellek adreslerinin doğrudan ilgili C kodundaki dosya ve satır numarasına çözümlenebildiği doğrulanmıştır. Bu özellik üretim sahasında bir kilitlenme yaşandığında crash adresini hızla kaynak koduna haritalandırmak için kusursuz çalışır.

---

# 7. ELF Yapısı Analizi

* ELF header
* Section header
* Program header
* Symbol table
* Relocation entries
* Debug sections
* DWARF info
* Linker-generated metadata
* Startup section
* Vector table
* Initialization routines

Araçlar:
* `msp430-readelf`
* `msp430-elfedit`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
```bash
sule@ubuntu:~$ msp430-readelf -S new-firmware.z1 | grep -E '\.text|\.data|\.bss|\.vectors'
  [ 1] .far.text         PROGBITS        00010000 00cff2 004a78 00  AX  0   0  2
  [ 2] .text             PROGBITS        00003100 0000f4 00976e 00  AX  0   0  2
  [ 4] .data             PROGBITS        00001100 00ce62 000150 00  WA  0   0  2
  [ 5] .bss              NOBITS          00001250 00cfb2 001648 00  WA  0   0  2
  [ 7] .vectors          PROGBITS        0000ffc0 00cfb2 000040 00  AX  0   0  1
```

Bölüm (Section) başlıkları incelendiğinde `PROGBITS` işaretli `.text` (0x3100 başlangıçlı) ve `.far.text` (0x10000 başlangıçlı) alanları Flash üzerinde asıl veriyi taşıdığını gösterir. `NOBITS` türündeki `.bss` bölümü ise fiziksel dosyada yer kaplamayıp yalnızca bellekte RAM organizasyonu için metadata barındırır. `0xFFC0` adresindeki `.vectors` tablosu donanımsal kesme yönlendirme tablosunu ifade etmektedir.

---

# 8. Interrupt ve Donanım Analizi

* Interrupt vector table
* GPIO access pattern
* Timer interrupt kullanımı
* UART ISR
* Radio interrupt handler
* ADC access
* Sensor polling
* Low-power mode geçişleri
* Clock configuration
* MSP430 register erişimleri

Araçlar:
* `msp430-objdump`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
MSP430'un `0xFFC0` vektör adresinden başlayan tablosu `objdump` ile incelendiğinde, doğrudan `cc2420_timerb1_interrupt` (radyo kesmeleri) ve `port1_isr` (GPIO harici kesmeleri) sembollerine bağlandığı saptanmıştır. İşletim sistemi boşta kaldığında donanımsal uyku (Low-Power Mode - LPM3) devreye girer. Uyandırma işlemi ise yalnızca bu donanım tabanlı Timer ve Radio ISR'leri üzerinden asenkron olarak gerçekleştirilir.

---

# 9. Networking Analizi

* Unicast kullanım tespiti
* Broadcast kullanım tespiti
* Multicast tespiti
* IPv6 stack kullanımı
* RPL routing analizi
* TSCH scheduler çağrıları
* MAC layer interaction
* Packet buffer kullanımı
* Neighbor table erişimi
* Radio transmission akışı
* Retransmission logic
* ACK mekanizmaları
* CSMA/TSCH farkları
* Contiki network API kullanımı

Araçlar:
* `msp430-nm`
* `msp430-objdump`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Sembol tablosundaki IPv6 fonksiyonları (uIP yığını) cihazın Unicast UDP üzerinden iletişim kurduğunu doğrular. Ağ katmanında "RPL-Lite" routing mantığı bulunmakta ve düğümler arası komşuluk tablosu (Neighbor Table) yönetilmektedir. MAC katmanı için ACK (onay) mekanizmalı bir paket akışı kodlara yansımıştır ve OTA senaryosunda "Stop-and-Wait ARQ" yapısı ile retransmission (yeniden gönderim) yapıldığı anlaşılmaktadır.

---

# 10. Wireless / TSCH Analizi

* TSCH slot operation
* Channel hopping logic
* ASN handling
* Radio timing loops
* Synchronization routines
* Schedule management
* Packet timing
* MAC timing critical path
* Drift compensation
* Low-power radio behavior

Araçlar:
* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
IEEE 802.15.4 standardı radyo iletişimi üzerine kurulu bu sistemde, senkron TSCH operasyonlarından ziyade asenkron CSMA (Carrier Sense Multiple Access) ve ContikiMAC duty-cycling kullanılmıştır. Sembol tablolarında yoğun bir Channel Hopping (Kanal Atlama) slot mantığı bulunmadığından cihazlar dinleme modlarında senkronizasyon döngülerine düşük güçte (Low-power radio behavior) cevap vermektedir.

---

# 11. Sensor ve Peripheral Analizi

* Button handler
* LED driver
* UART usage
* SPI access
* I2C access
* ADC routines
* Sensor polling interval
* Interrupt-driven sensor logic
* GPIO toggle behavior
* Peripheral initialization sequence

Araçlar:
* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Donanım başlatma sürecinde `gpio_hal_arch_init` çağrılarak LED ve Buton donanımları GPIO portları üzerinden bağlanmıştır. Z1 cihazında bulunan dahili sıcaklık ve x/y/z ivmeölçer sensörleri için ADC rutinleri mevcuttur. Sensör etkileşimi zamanlı-yoklama (polling interval) yerine kesme-odaklı (interrupt-driven) tetikleyicilerle ele alınarak gereksiz işlemci döngüleri engellenmiştir.

---

# 12. Algoritma Koşma / DSP / Matematiksel Analiz

* Floating-point kullanımı
* Fixed-point kullanımı
* Trigonometric computation
* Multiply/divide routines
* Software floating-point emulation
* DSP benzeri loop’lar
* Matrix operation izleri
* Signal processing pattern’leri
* Computational hotspot’lar
* Numerical optimization

Araçlar:
* `msp430-objdump`
* `msp430-gprof`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Z1 ve Sky cihazlarının MSP430 donanımında FPU (Kayan Nokta Birimi) bulunmamaktadır. Dolayısıyla OTA transferinde kullanılan CRC32 hesaplama algoritmaları ve ağ istatistikleri tamamen tam sayı (Fixed-point) matematiği üzerine optimize edilmiştir. Derleyici analizinde floating-point emülasyon izlerine rastlanmamıştır; şayet eklenseydi firmware flash boyutu çok daha gereksiz oranlarda şişecekti.

---

# 13. Güç ve Performans Analizi

* Low-power mode geçişleri
* CPU-intensive function’lar
* Busy-wait detection
* Sleep/wakeup flow
* Timer usage intensity
* Radio duty cycle tahmini
* ISR yoğunluğu
* Function execution cost
* Flash/RAM efficiency
* Energy-heavy computation bölgeleri

Araçlar:
* `msp430-gprof`
* `msp430-objdump`
* `msp430-size`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
`PROCESS_YIELD()` mantığı ile IoT cihazı CPU-yoğun döngülere (`while(1)` busy-wait) girmekten sakınmıştır. Event-driven mimari sayesinde sistem enerjisinin büyük kısmını radyoda harcarken, CPU uyku modlarında (LPM) bekletilmektedir. Enerjinin en çok tüketildiği hotspot bölgeleri CRC hesaplama anları (Flash okumaları) ve MAC radyo paket gönderme döngüleridir.

---

# 14. Coverage ve Profiling Analizi

* Function call frequency
* Execution hotspot
* Unused branch’ler
* Rarely executed path’ler
* Test coverage
* Critical execution path
* Runtime bottleneck’ler

Araçlar:
* `msp430-gcov`
* `msp430-gprof`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Cihazın çalışma ortamında en büyük darboğazı (bottleneck) CPU yürütme hızı değil, asenkron radyo paket onay beklemeleridir. Nadiren çalıştırılan rotalar hata yakalama (Error Catch) blokları iken, `critical execution path` doğrudan UDP paket alım (RX) donanım ISR'leri ile bellek kopyalama (`memcpy`) komutlarından oluşmaktadır.

---

# 15. Reverse Engineering Analizi

* Firmware behavior recovery
* Unknown firmware classification
* Feature inference
* Protocol inference
* ISR purpose discovery
* Hardware interaction recovery
* State machine extraction
* Scheduler reconstruction
* Event-flow reconstruction
* Network role inference

Araçlar:
* `msp430-objdump`
* `msp430-nm`
* `msp430-readelf`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Firmware dosyası `.z1` veya `.sky` uzantısına sahip bir kara kutu gibi görünse de, `file` ve `readelf` analizleriyle bunun klasik bir UNIX ELF objesi olduğu çözümlenmiştir. Not-stripped (sembolleri saklanmamış) yapısı sayesinde ISR vektörlerinden ağdaki rolüne (Client/Server) kadar sistemin tüm State Machine mimarisi rahatlıkla geri döndürülebilmektedir (Recovery).

---

# 16. Compiler ve Optimization Analizi

* `-O0/-O2/-Os` farkları
* Inlining behavior
* Dead code elimination
* Constant folding
* Loop optimization
* Register allocation
* Tail-call optimization
* Branch optimization
* Macro expansion
* Preprocessor etkileri

Araçlar:
* `msp430-gcc`
* `msp430-cpp`
* `msp430-objdump`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Derleyici bayraklarında boyut iyileştirmesi anlamına gelen `-Os` komutu etkin olarak çalıştırılmıştır. Objdump çıktısında inline edilmiş (çağrı yerine doğrudan gömülmüş) kod bloklarının kısıtlılığı, dead-code eliminasyonu yapıldığını ve makine kodunun Flash sınırlarında tutulabilmesi için maksimum optimizasyon gösterdiğini ispatlar.

---

# 17. Linker ve Build Sistemi Analizi

* Section placement
* Link order
* Static library linkage
* Startup code
* Linker script behavior
* Vector placement
* Symbol resolution
* Relocation behavior

Araçlar:
* `msp430-ld`
* `msp430-ar`
* `msp430-ranlib`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Z1 ve Sky derlemelerinde kullanılan `contiki-z1.ld` (Linker Script) belgesi, yürütülebilir kodun (`.text`) 0x3100 veya 0x4000 adresinden başlatılmasını katı kurallarla emretmiştir. Sembollerin donanımsal çip kısıtlarına tam olarak oturması, relokasyon işlemlerinin (relocation) tamamen statik gerçekleştiği anlamına gelir. 

---

# 18. Binary Transformation Analizi

* ELF → HEX conversion
* ELF → binary conversion
* Section extraction
* Symbol stripping
* Debug removal
* Firmware minimization
* Binary patch preparation

Araçlar:
* `msp430-objcopy`
* `msp430-strip`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Bir firmware ağ üzerinden havadan (OTA) gönderileceği zaman orijinal ELF dosyası bu işlemi gerçekleştirmek için haddinden büyüktür. Bu sebeple `msp430-objcopy -O binary` (veya Intel HEX) format dönüşümleri uygulanarak sadece saf çalıştırılabilir makine komutları (Metadata ve semboller kırılarak) çıkarılmış ve hex tablosuna dönüştürülmüştür.

---

# 19. Library ve Archive Analizi

* Static library içeriği
* Object file extraction
* Archive symbol table
* Linked module analizi

Araçlar:
* `msp430-ar`
* `msp430-gcc-ar`
* `msp430-ranlib`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Contiki-NG çekirdeğinde yer alan `memb`, `list`, `ringbuf` gibi modüler C dosyaları dinamik linkleme kavramı mikroişlemcilerde mümkün olmadığı için `.a` arşiv dosyaları haline getirilip, bağlayıcı (Linker) tarafından projeye statik objeler olarak katılmıştır.

---

# 20. Contiki-NG Özel Analizler

* PROCESS_THREAD recovery
* Protothread expansion
* Event-driven scheduler analizi
* etimer/ctimer usage
* PROCESS_BEGIN/END expansion
* PROCESS_YIELD flow
* NETSTACK interaction
* Packetbuf lifecycle
* uIP callback chain
* Rime stack usage

Araçlar:
* `msp430-cpp`
* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Contiki işletim sisteminin devrim niteliğindeki Protothread (Stackless Thread) özelliği assembly düzeyinde incelendiğinde, `PROCESS_BEGIN()` ve `PROCESS_YIELD()` makrolarının C dilindeki yerel `switch-case` yapılarına C-PreProcessor tarafından açıldığı saptanmıştır. İş parçacığı durumunu hatırlamak için yerel değişkenler `static` anahtarı ile RAM `.bss`/`.data` alanlarına sabitlenmiştir. 

---

# 21. Güvenlik ve Robustness Analizi

* Hardcoded credential arama
* Debug backdoor izleri
* Buffer handling
* Unsafe memory access
* Stack-heavy routines
* Potential overflow bölgeleri
* Assert/debug remnants
* Information leakage string’leri

Araçlar:
* `msp430-strings`
* `msp430-objdump`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

**Uygulama ve Analiz:**
Sistemde dinamik hafıza tespiti (malloc/free) yapılmaması sayesinde Buffer Overflow (yığın taşması) ve bellek sızıntısı zafiyetleri donanımsal boyutta asgariye indirilmiştir. Ancak `.symtab` debug izlerinin firmware içerisine gömülü gelmesi sebebiyle, kötü niyetli bir geliştirici için ağ altyapısını çözmek veya backdoor bulmak çok kolaydır (Information Leakage). Ağa ve cihaza donanımsal reset atmak adına Watchdog Timber kullanıldığı sembol tablosundan teyit edilmiştir.

---

# 22. Karşılaştırmalı Firmware Analizi

İki firmware arasında:

* Code size farkı
* RAM farkı
* Function count farkı
* ISR yoğunluğu
* Networking complexity
* Radio stack farkı
* Symbol farkı
* Optimization farkı
* Assembly complexity farkı

**Uygulama ve Analiz:**
Z1 platformu ile ARM CC1352R platformu kıyaslandığında;
- **Code Size ve RAM:** ARM CC1352R firmware'inde `.text` ve `.bss` boyutları MSP430 tabanlı cihazlara göre oldukça geniştir. İşlemcinin 32-bit (ARMv7E-M) olması bellek yapılarını, buffer genişliklerini dolayısıyla doğrudan boyutu büyütmüştür.
- **Complexity:** ARM donanımı, karmaşık DMA (Direct Memory Access) ve SPI kanalları barındırır. `readelf -S` raporlarında ARM çipinin özel `.dmaSpi` ve donanım kontrol bölgelerinin bağımsız sectionlara atandığı tespit edilmiştir. 
- **ISR:** MSP430'da sabit donanımsal kesme tablosu (`0xFFC0`) bulunurken, CC1352R ARM çipinde kesmeler RAM tabanlı dinamik bir tablo (`vtable_ram` 0x20002000 adresinde) ile yönetilmektedir.

---

# 23. Eğitimsel Reverse Engineering Görevleri

* Bir firmware’in ne yaptığını bulma
* hangi protokolü kullandığını çıkarma
* button/LED mapping bulma
* ISR’leri tanıma
* network role çıkarımı
* Kullandığı algoritmik blok tespiti
* energy-heavy bölgeleri bulma
* stripped firmware çözümleme

**Uygulama ve Analiz:**
Bu projenin nihai amacı, kaynağı bilinmeyen bir firmware objesinin (`.z1` / `.sky`) statik ELF analizi teknikleri ile işlevselliğinin tamamen çözümlenebilmesidir. Başarılı bir şekilde cihazın bir Contiki-NG "UDP İstemcisi" olduğu, RPL routing protokolü kullandığı, IPv6 ağı üzerinden paket yolladığı ve CRC32 doğrulama barındırdığı hiçbir kaynak koda bakılmaksızın (yalnızca derlenmiş imaj üzerinden) çıkartılmıştır.
