# 1-BOB: KOMPYUTER TARMOQLARI VA INTERNET (Computer Networks and the Internet)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Kompyuter tarmoqlarining umumiy arxitekturasi, ishlash tamoyillari, Internetning jismoniy va mantiqiy tuzilishi, ma’lumotlarni uzatish mexanizmlari (paketlar va kanallar kommutatsiyasi), tarmoqdagi kechikishlar va yo‘qotishlarning matematik-fizik mohiyati hamda protokollarning qatlamli modelini fundamental darajada o‘rganish.

---

## 1.1. INTERNET NIMA O‘ZI? ARXITEKTURA VA ISHLASH MEXANIZMI

Internet tushunchasini to‘liq idrok etish uchun unga bir vaqtning o‘zida ikki xil nuqtai nazardan yondashish zarur:
1. **Tizimli-texnik (apparat va dasturiy qurilmalar) nuqtai nazari.**
2. **Ilovalarga xizmat ko‘rsatuvchi taqsimlangan infratuzilma nuqtai nazari.**

```
                                 GLOBAL INTERNET
                                        |
         +------------------------------+------------------------------+
         |                                                             |
         v                                                             v
TIZIMLI-APPARAT TUZILISHI                                    XIZMATLAR VA ILOVALAR
* Xostlar (End Systems: PC, Server, IoT)                     * Taqsimlangan ilovalar (Web, Mail, P2P)
* Aloqa liniyalari (Mis, Optika, Radio)                      * Socket API (Dasturiy interfeys)
* Paket kommutatorlari (Routerlar, Switchlar)                * Transport xizmatlari (Kafolatli/Kafolatsiz)
* ISP ierarxiyasi (Tier-1, Tier-2, IXP)                      * Aloqa qoidalari va standartlar
```

---

### 1.1.1. Tizimli-Texnik Tuzilish (Nuts-and-Bolts View)

Internet — bu butun yer sharini qamrab olgan, milliardlab hisoblash qurilmalarini yagona tizimga birlashtiruvchi global axborot infratuzilmasidir. Uning asosiy tarkibiy qismlari:

#### 1. Yakuniy tizimlar yoki Xostlar (End Systems / Hosts)
Tarmoqqa ulangan va undagi ma’lumotlarni yaratuvchi, qayta ishlovchi yoki iste’mol qiluvchi barcha qurilmalar **xost** deb ataladi. 
* **Mijozlar (Clients):** Foydalanuvchilar tomonidan boshqariladigan shaxsiy kompyuterlar, noutbuklar, smartfonlar, planshetlar.
* **Serverlar (Servers):** Katta hajmdagi ma’lumotlarni saqlaydigan, so‘rovlarni qayta ishlaydigan va veb-saytlar, ma’lumotlar bazalari, video oqimlarga xizmat ko‘rsatadigan yuqori quvvatli mashinalar. Hozirgi kunda aksariyat serverlar yuz minglab serverlardan iborat gigant bulutli Ma’lumotlar Markazlarida (Data Centers: Google, AWS, Microsoft) joylashgan.
* **IoT (Internet of Things — Narsalar Interneti):** So‘nggi yillarda tarmoqqa ulangan qurilmalarning asosiy qismini kompyuterlar emas, balki aqlli soatlar, xavfsizlik kameralari, datchiklar, avtomobillar va maishiy texnikalar tashkil qilmoqda.

#### 2. Aloqa liniyalari (Communication Links)
Xostlarni bir-biri bilan bog‘lovchi jismoniy yo‘llar. Liniyalar fizik xususiyatlariga ko‘ra har xil bo‘lib, ularning asosiy ko‘rsatkichi **Uzatish tezligi (Transmission Rate yoki Bandwidth — O‘tkazuvchanlik qobiliyati)** dir. Tezlik soniyasiga uzatiladigan bitlar soni bilan o‘lchanadi:
* $1 \text{ Kbps} = 10^3 \text{ bit/sekund}$
* $1 \text{ Mbps} = 10^6 \text{ bit/sekund}$
* $1 \text{ Gbps} = 10^9 \text{ bit/sekund}$
* $1 \text{ Tbps} = 10^{12} \text{ bit/sekund}$

#### 3. Paket kommutatorlari (Packet Switches)
Aloqa liniyalaridan kelayotgan ma’lumotlarni qabul qilib, ularning manzilini tahlil qilgan holda kerakli yo‘nalishga uzatuvchi oraliq qurilmalar. Ularning ikkita asosiy turi mavjud:
* **Routerlar (Marshrutizatorlar / Yo‘naltirgichlar):** Asosan tarmoq yadrosida (Network Core) joylashadi va paketlarni OSI modelining 3-qatlami (Tarmoq qatlami / IP-manzillar) asosida global miqyosda marshrutlaydi.
* **Link-Layer Switchlar (Kommutatorlar):** Asosan lokal kirish tarmoqlarida (Access Networks / LAN) qo‘llaniladi va ma’lumotlarni 2-qatlam (Kanal qatlami / MAC-manzillar) asosida yo‘naltiradi.

#### 4. Paket (Packet) tushunchasi
Ma’lumot uzatuvchi xost katta hajmdagi xabarni (masalan, 10 megabaytli rasm yoki faylni) birdaniga butunicha tarmoqqa chiqarmaydi. Buning o‘rniga ma’lumot kichik qismlarga — bo‘laklarga ajratiladi. Har bir bo‘lakka manzil va boshqaruv ma’lumotlari (Header — Sarlavha) biriktiriladi. Mana shu birlik **Paket (Packet)** deb ataladi. Paketlar tarmoq bo‘ylab alohida harakatlanadi va qabul qiluvchi xostda yana birlashtirilib, asl holatiga keltiriladi.

---

### 1.1.2. Xizmatlar Nuqtai Nazaridan Tavsif (Services View)

Foydalanuvchi va dasturchi uchun Internet — bu shunchaki kabellar va routerlar emas, balki **taqsimlangan ilovalar (Distributed Applications)** faoliyatini ta’minlovchi ulkan maydondir.
* **Taqsimlangan ilovalar:** Bir vaqtning o‘zida turli xil geografik nuqtalardagi xostlarda ishlaydigan va o‘zaro ma’lumot almashadigan tizimlar (masalan, brauzer va veb-server, Telegram mijozi va Telegram bulut serveri, P2P fayl almashish tarmoqlari).
* **Socket API (Dasturiy interfeys):** Yakuniy tizimda ishlayotgan dastur boshqa tizimdagi dasturga ma’lumot jo‘natishi uchun operatsion tizim maxsus dasturiy interfeys taqdim etadi. Bu interfeys **Socket (Soket)** deb ataladi. Socket dasturchiga xuddi bitta eshik kabi xizmat qiladi: dastur o‘z xabarini soketga tashlaydi, Internet infratuzilmasi esa uni manziliga yetkazish majburiyatini oladi.
* **Xizmat turlari:** Internet ilovalarga ikki xil asosiy transport xizmatini taklif qiladi:
  1. *Ishonchli, ulanishga asoslangan xizmat (TCP):* Ma’lumotlar birorta ham bit yo‘qolmasdan, ketma-ketlikda yetkazilishini kafolatlaydi.
  2. *Yengil, ulanishsiz xizmat (UDP):* Ma’lumotni imkon qadar tez yetkazishga harakat qiladi, lekin paket yo‘qolmasligiga kafolat bermaydi.

---

### 1.1.3. Protokol nima? (What Is a Protocol?)

Kompyuter tarmoqlarida tartib va o‘zaro hamjihatlik faqat protokollar orqali ta’minlanadi.

```
INSONIY MULOQOT PROTOKOLI                          TARMOQ PROTOKOLI (TCP)
      [Inson A]            [Inson B]                      [Kompyuter A]         [Kompyuter B]
          |                    |                                |                    |
          |--- "Salom!" ------>|                                |-- TCP SYN req ---->|
          |<-- "Salom!" -------|                                |<- TCP SYN/ACK resp-|
          |                    |                                |                    |
          |--- "Soat necha?" ->|                                |-- TCP ACK -------->|
          |<-- "Soat 15:30" ---|                                |-- GET /file.html ->|
          |                    |                                |<- 200 OK (Data) ---|
```

> **Fundamental Ta’rif:**  
> **Tarmoq protokoli** — bu o‘zaro muloqot qiluvchi ikki yoki undan ortiq tarmoq sub’yektlari o‘rtasida almashinadigan **xabarlarning formati (tuzilishi), ularni uzatish va qabul qilish ketma-ketligi (tartibi)** hamda ushbu xabarlar kelganda yoki boshqa hodisalar yuz berganda **tizim amalga oshirishi shart bo‘lgan harakatlarni** qat’iy belgilovchi qoidalar majmuasidir.

Agar jo‘natuvchi bitta protokol bo‘yicha ishlasa-yu, qabul qiluvchi boshqa protokolni kutsa, muloqot butunlay barbod bo‘ladi. Shu sababli Internet protokollari **IETF (Internet Engineering Task Force)** xalqaro tashkiloti tomonidan qat’iy standartlashtiriladi va ularning spetsifikatsiyalari **RFC (Request for Comments)** hujjatlarida e’lon qilinadi.

---

## 1.2. TARMOQ CHETI (THE NETWORK EDGE): KIRISH TARMOQLARI VA ALOQA MUHITI

Tarmoq arxitekturasi shartli ravishda ikki qismga bo‘linadi:
1. **Network Edge (Tarmoq cheti):** Xostlar (kompyuterlar, serverlar) va ularni bog‘lovchi dasturlar.
2. **Network Core (Tarmoq yadrosi):** Paketlarni suruvchi routerlar va magistral kanallar tarmog‘i.

Xostlarni tarmoq yadrosidagi birinchi routerga (**Edge Router / Birinchi sakrash routeri**) bog‘lovchi qismga **Kirish tarmoqlari (Access Networks)** deyiladi.

---

### 1.2.1. Kirish Tarmoqlari Turlari va Ishlash Mexanizmi

#### 1. Raqamli Abonent Liniyasi — DSL (Digital Subscriber Line)
DSL texnologiyasi uylarda mavjud bo‘lgan an’anaviy mis telefon simlaridan bir vaqtning o‘zida ham telefon suhbati, ham yuqori tezlikdagi Internet uchun foydalanish imkonini beradi.
* **Ishlash mexanizmi:** Telefon simidagi signallar turli chastotalarga ajratiladi (**FDM — Frequency Division Multiplexing**):
  * `0 – 4 kHz`: An’anaviy ovozli telefon suhbati uchun.
  * `4 – 50 kHz`: Foydalanuvchidan tarmoqqa ma’lumot yuklash (Upstream) uchun.
  * `50 kHz – 1 MHz`: Tarmoqdan ma’lumot yuklab olish (Downstream) uchun.
* **Asimmetriklik (ADSL):** Odatda oddiy foydalanuvchilar ma’lumot yuklashdan ko‘ra ko‘proq ma’lumot yuklab oladi. Shuning uchun downstream kanali upstream kanalidan ancha keng qilinadi.
* **Apparat qismlari:**
  * Uyda: **DSL Modem** (raqamli kompyuter signallarini analog to‘lqinlarga aylantiradi) va **Splitter** (ovoz va ma’lumotni ajratuvchi filtr).
  * Telefon stansiyasida (Central Office): **DSLAM (DSL Access Multiplexer)** — yuzlab uylardan kelgan DSL signallarini yig‘ib, raqamli ma’lumotga aylantiradi va Internet routeriga uzatadi.
* **Cheklovi:** Uy telefon stansiyasidan qanchalik uzoqda joylashgan bo‘lsa (masalan, 3–5 km dan oshsa), signal so‘nishi tufayli tezlik shunchalik keskin pasayadi.

#### 2. Kabel orqali kirish (Cable Internet / HFC — Hybrid Fiber-Coax)
Kabel televideniyesi (kabel TV) infratuzilmasidan foydalanuvchi texnologiya.
* **HFC Arxitekturasi:** Shahar bo‘ylab magistral kanallar optik toladan (Fiber), mahallalardan xonadonlargacha esa koaksial mis kabeldan (Coaxial) iborat.
* **DOCSIS (Data Over Cable Service Interface Specification):** Kabel orqali ma’lumot uzatish standarti.
* **Muhim xususiyati (Shared Medium — Umumiy taqsimlangan muhit):**
  * DSL da har bir xonadon stansiyaga alohida shaxsiy sim bilan ulangan bo‘lsa, kabel internetida bitta koaksial kabelga butun bir ko‘cha yoki bino birgalikda ulanadi.
  * Agar qo‘shnilaringiz bir vaqtda og‘ir torrent yoki 4K video yuklashni boshlasa, umumiy kanal band bo‘lib, sizning ham tezligingiz tushib ketadi!

#### 3. Xonadongacha Optik Tola — FTTH (Fiber to the Home)
Eng zamonaviy, ishonchli va o‘ta yuqori tezlik (1 Gbps – 10 Gbps+) taqdim etuvchi kirish texnologiyasi.
* Telefon stansiyasidan to‘g‘ridan-to‘g‘ri har bir xonadonga alohida optik tolali shisha kabel tortiladi.
* **PON (Passive Optical Network):** Har bir uyga stansiyadan alohida kabel tortish juda qimmatga tushishi sababli, stansiyadan bitta optik tola chiqadi va mahallada maxsus elektr quvvatisiz ishlovchi optik bo‘luvchi (**Optical Splitter**) yordamida o‘nlab xonadonlarga tarqatiladi.
* Xonadonda **ONT (Optical Network Terminal)** o‘rnatiladi va u optik yorug‘lik signallarini oddiy kompyuter Ethernet kabeliga aylantirib beradi.

#### 4. Ethernet va Wi-Fi (Korporativ va Uy LAN tarmoqlari)
* **Ethernet (IEEE 802.3):** Kompyuterlar **Twisted Pair (UTP Cat 5e/Cat 6)** mis kabellari orqali markaziy kommutatorga (**Switch**) ulanadi. Standart tezliklar: 100 Mbps, 1 Gbps, 10 Gbps.
* **Wi-Fi (IEEE 802.11):** Noutbuklar va telefonlar simsiz radio to‘lqinlar orqali **Access Point (AP — Kirish nuqtasi)** ga ulanadi. AP o‘z navbatida simli tarmoqqa ulangan bo‘ladi.

#### 5. Keng qamrovli mobil tarmoqlar (4G LTE va 5G)
Uyali aloqa operatorlari tomonidan taqdim etiladigan simsiz kirish. Qurilmalar o‘nlab kilometr masofadagi baza stansiyalari (minoralar) bilan aloqa o‘rnatadi. 4G LTE o‘nlab megabit, 5G esa gigabitgacha bo‘lgan simsiz tezlikni ta’minlaydi.

---

### 1.2.2. Jismoniy Aloqa Muhiti (Physical Media)

Bitlar jo‘natuvchidan qabul qiluvchiga jismoniy muhit orqali uzatiladi. Jismoniy muhit ikki toifaga bo‘linadi:

```
                              JISMONIY ALOQA MUHITI
                                        |
         +------------------------------+------------------------------+
         |                                                             |
         v                                                             v
BOSHQARILADIGAN (GUIDED MEDIA)                              BOSHQARILMAYDIGAN (UNGUIDED MEDIA)
To'lqinlar jismoniy qobiq ichida yo'naltiriladi:            To'lqinlar ochiq fazoda erkin tarqaladi:
* O'ralgan juftlik (UTP/STP mis simlar)                     * Yer usti radio to'lqinlari (Wi-Fi, 4G/5G)
* Koaksial kabel (Mis o'zak, ekranlangan)                   * Mikroto'lqinli rele liniyalari
* Optik tolali kabel (Shisha tola, yorug'lik)               * Sun'iy yo'ldosh aloqasi (Geostatsionar, LEO)
```

1. **O‘ralgan juftlik (Twisted Pair — UTP):**
   * Ikkita izolyatsiyalangan mis sim bir-biriga spiralsimon qilib o‘raladi.
   * *Nega o‘raladi?* Tashqi elektromagnit to‘lqinlar va qo‘shni simlarning o‘zaro xalaqitini (Crosstalk) bekor qilish uchun.
   * Kategoriya turlari: Cat 5e (1 Gbps gacha / 100 m), Cat 6/6a (10 Gbps gacha).
2. **Koaksial kabel:**
   * Markazda mustahkam mis o‘zak, uning atrofida dielektrik izolyatsiya, uning ustida to‘qilgan metall ekran va eng ustida tashqi qobiq. Yaxshi himoyalanganligi sababli ancha yuqori tezlik va masofani ushlaydi.
3. **Optik tolali kabel (Fiber Optics):**
   * Inson sochi qalinligidagi o‘ta toza shisha (kvars) tola. Signallar yorug‘lik impulslari (fotonlar) ko‘rinishida to‘liq ichki qaytish (Total Internal Reflection) hodisasi asosida yuguradi.
   * *Afzalliklari:* Elektromagnit xalaqitlarga mutlaqo ta’sirlanmaydi, signal so‘nishi nihoyatda past (yuzlab kilometrga kuchaytirishsiz yetadi), o‘ta ulkan o‘tkazuvchanlikka ega.
4. **Boshqarilmaydigan muhit (Radio to‘lqinlar):**
   * Kabelsiz, havo yoki vakuum orqali tarqaladi. Signal devorlardan aks etadi, ob-havo (yomg‘ir, tuman) tufayli so‘nadi va boshqa qurilmalar bilan xalaqitga uchraydi.

---

## 1.3. TARMOQ YADROSI (THE NETWORK CORE): KOMMUTATSIYA MEXANIZMLARI

Tarmoq yadrosi — bu butun dunyodagi xostlarni o‘zaro bog‘lab turuvchi paket kommutatorlari (routerlar) va magistral aloqa liniyalarining o‘rgimchak to‘ridir.

Tarmoq yadrosida ma’lumotlarni uzatishning ikkita tubdan farq qiluvchi strategiyasi mavjud:
1. **Paketlar kommutatsiyasi (Packet Switching)** — Internetning asosi.
2. **Kanallar kommutatsiyasi (Circuit Switching)** — An’anaviy telefon tarmog‘ining asosi.

---

### 1.3.1. Paketlar Kommutatsiyasi (Packet Switching)

Paketlar kommutatsiyasida xostlar ma’lumotlarni **paketlar** ko‘rinishida tarmoqqa chiqaradi. Paketlar manzilga yetib borguncha bir nechta aloqa liniyalari va routerlardan o‘tadi.

```
[Jo'natuvchi Xost] ===== (Liniya R1) =====> [Router] ===== (Liniya R2) =====> [Qabul qiluvchi]
Paket hajmi = L bit
```

#### 1. Store-and-Forward (Saqlash va Keyin Uzatish) Mexanizmi
Router paketni chiquvchi liniyaga uzatishdan oldin **uning barcha bitlarini to‘liq qabul qilib olib, o‘z xotirasida (buferida) saqlashi shart**!
* Agar paket uzunligi $L$ bit bo‘lsa, liniyaning uzatish tezligi $R$ bit/sekund bo‘lsa, bitta paketni bitta liniyaga to‘liq chiqarish uchun ketadigan vaqt:
  $$d_{\text{trans}} = \frac{L}{R}$$
* Agar jo‘natuvchi va qabul qiluvchi o‘rtasida $N$ ta bir xil tezlikdagi ($R$) liniyalar va $N-1$ ta router bo‘lsa, bitta paketning oxirigacha yetib borish vaqti (boshqa kechikishlarni hisobga olmaganda):
  $$d_{\text{end-to-end}} = N \times \frac{L}{R}$$

#### 2. Navbatdagi Kechikish (Queuing Delay) va Paketlarning Yo‘qolishi (Packet Loss)
Har bir router chiquvchi liniyalari uchun **Chiquvchi buferga (Output Buffer / Queue)** ega.
* Agar paketlarning routerga kelish tezligi ularning chiquvchi liniyaga chiqib ketish tezligidan yuqori bo‘lsa, paketlar buferda navbatga turadi (**Queuing Delay** yuzaga keladi).
* Router xotirasi (buferi) cheksiz emas. Agar bufer to‘lib ketsa va yangi paketlar kelishda davom etsa, router ularni qabul qila olmaydi va shunchaki tashlab yuboradi! Bu hodisa **Paket yo‘qolishi (Packet Loss yoki Packet Drop)** deb ataladi.

---

### 1.3.2. Kanallar Kommutatsiyasi (Circuit Switching)

An’anaviy telefon tarmoqlarida qo‘llanilgan tizim. Ikki tomon muloqot qilishi uchun avval butun yo‘l bo‘ylab **kanal o‘rnatiladi (Call Setup)**.
* Kanal o‘rnatilgach, butun suhbat davomida barcha oraliq liniyalardagi resurslar (o‘tkazuvchanlik qobiliyati) aynan shu ikki tomon uchun **band qilib qo‘yiladi**.
* Tomonlar gaplashmay, jim tursa ham, hech kim bu kanaldan foydalana olmaydi (resurslar isrof bo‘ladi).

```
FDM (Frequency-Division Multiplexing)         TDM (Time-Division Multiplexing)
Chastota                                      Vaqt
 ^                                             ^
 | [ Foydalanuvchi 4 ]                         | [1][2][3][4] [1][2][3][4] ...
 | [ Foydalanuvchi 3 ]                         | Har bir foydalanuvchiga davriy
 | [ Foydalanuvchi 2 ]                         | qat'iy vaqt oraliqlari (time slot)
 | [ Foydalanuvchi 1 ]                         | ajratiladi.
 +----------------------> Vaqt                 +------------------------------> Chastota
 Har biriga alohida chastota beriladi.
```

---

### 1.3.3. Nega Internet Paketlar Kommutatsiyasini Tanladi? (Paket vs Kanal)

Keling, kitobdagi mashhur matematik misolni ko‘raylik:
* Tasavvur qiling, aloqa liniyasi tezligi: **1 Mbps (1 000 000 bps)**.
* Har bir foydalanuvchi faol bo‘lganida **100 Kbps** tezlik talab qiladi.
* Lekin har bir foydalanuvchi vaqtning atigi **10% qismida (0.1)** faol bo‘ladi (masalan, sahifani ochadi va uni 2 daqiqa o‘qiydi, bu paytda hech narsa uzatmaydi).

1. **Kanallar kommutatsiyasida (Circuit Switching):**
   * Har bir foydalanuvchiga doimiy 100 Kbps ajratilishi shart.
   * Liniyaga ko‘pi bilan: $\frac{1 \text{ Mbps}}{100 \text{ Kbps}} = 10 \text{ ta foydalanuvchi}$ ulanishi mumkin. 11-foydalanuvchiga tarmoq "Band" (Busy signal) beradi.
2. **Paketlar kommutatsiyasida (Packet Switching):**
   * Resurslar oldindan band qilinmaydi. Kimga kerak bo‘lsa, o‘sha zahoti paket chiqaradi (**Statistical Multiplexing — Statistik multipleksirlash**).
   * Agar bu liniyaga **35 ta foydalanuvchi** ulasak:
     * 10 tadan ko‘p foydalanuvchining bir vaqtda faol bo‘lib qolish ehtimoli Binomial taqsimot formulasi bo‘yicha hisoblanganda **0.0004 (0.04%) dan kamroq** chiqadi!
     * Ya’ni vaqtning 99.96% qismida 35 ta foydalanuvchi birorta ham kechikishsiz, xuddi ularga shaxsiy liniya berilgandek ishlayveradi!

> **Xulosa:**  
> Paketlar kommutatsiyasi resurslardan **3-4 barobar samaraliroq** foydalanish imkonini beradi, arzonroq va masshtablashtirish oson. Shu sababli butun zamonaviy dunyo paketli tarmoqlarga o‘tgan.

---

### 1.3.4. Tarmoqlar Tarmog‘i (ISP Ierarxiyasi)

Dunyoda yuz millionlab mahalliy provayderlar (Access ISP) mavjud. Ular qanday qilib bir-biri bilan global miqyosda bog‘langan?

```
                                +-----------------------------+
                                |      TIER-1 ISP LAR         | (Global magistrallar:
                                | (AT&T, Lumen, Telia, NTT...) |  O'zaro Peering bilan
                                +-----------------------------+  bepul ulanadi)
                                         /            \
                                        /  (Tranzit)   \
                     +----------------------+       +----------------------+
                     |     TIER-2 ISP       |       |     TIER-2 ISP       | (Mintaqaviy provayderlar)
                     +----------------------+       +----------------------+
                             |      \                     /      |
                             |       \==== IXP Peering ==/       |
                             v                                   v
                     +---------------+                   +---------------+
                     |  ACCESS ISP   |                   |  ACCESS ISP   | (Mahalliy provayderlar:
                     +---------------+                   +---------------+  Uztelecom, Beeline...)
                             |                                   |
                         [ Xostlar ]                         [ Xostlar ]
```

* **Tier-1 ISP lar:** Dunyoning umurtqa pog‘onasi (Backbone). Ular hech kimdan internet sotib olmaydi, chunki butun yer sharini qamrab oluvchi o‘z optik liniyalariga ega. Ular o‘zaro teng huquqli shartnomalar (**Peering**) asosida bir-birlariga bepul ulanadi.
* **Tier-2 / Regional ISP lar:** Mintaqaviy provayderlar. Ular global tarmoqqa chiqish uchun Tier-1 larga pul to‘laydi (**Transit fee**).
* **IXP (Internet Exchange Point):** Provayderlarning o‘zaro to‘g‘ridan-to‘g‘ri va arzon ma’lumot almashishi uchun tashkil etilgan maxsus jismoniy infratuzilma markazlari.
* **Kontent Provayderlar Tarmoqlari (Google, Meta, Microsoft, Cloudflare):** Katta IT gigantlar o‘zlarining xususiy ma’lumotlar markazlarini va shaxsiy optik tarmoqlarini qurib, Tier-1 larni chetlab o‘tgan holda to‘g‘ridan-to‘g‘ri IXP larda mahalliy ISP larga ulanadi.

---

## 1.4. PAKETLI TARMOQLARDA KECHIKISH, YO‘QOTISH VA O‘TKAZUVCHANLIK

Paket bitta xostdan chiqib boshqa xostga yetib borguncha yo‘ldagi har bir routerda **Umumiy tugun kechikishi (Total Nodal Delay)** ga uchraydi:

$$d_{\text{nodal}} = d_{\text{proc}} + d_{\text{queue}} + d_{\text{trans}} + d_{\text{prop}}$$

Keling, ushbu 4 ta fizik kechikishni chuqur tahlil qilamiz:

```
[Paket keldi] ---> [ d_proc ] ---> [ d_queue ] ---> [ d_trans ] =======> [ d_prop ] ---> [Keyingi router]
                   Sarlavhani      Navbatda         Kabelga              Kabel bo'ylab
                   tekshirish      kutish           chiqarish            yugurish
```

---

### 1.4.1. Kechikishning 4 ta Komponenti

#### 1. Qayta ishlash kechikishi ($d_{\text{proc}}$ — Nodal Processing Delay)
* Paket router portiga yetib kelgach, router uning bitlarida xatolik yo‘qligini (Checksum) tekshiradi.
* Paket sarlavhasidagi `Destination IP` manzilini o‘qiydi va o‘zining marshrutlash jadvalidan qaysi chiquvchi interfeysga yo‘naltirishni aniqlaydi (**Table Lookup**).
* Bu jarayon zamonaviy routerlarda maxsus apparat chiplarida (ASIC) bajariladi va odatda mikrosekundlar (yoki undan kam) vaqt oladi.

#### 2. Navbatdagi kechikish ($d_{\text{queue}}$ — Queuing Delay)
* Paket chiquvchi liniyaga yuborilishidan oldin buferda o‘z navbatini kutadi.
* Bu kechikish o‘zgaruvchan (dinamik) bo‘lib, tarmoqdagi tirbandlikka bog‘liq. Agar oldinda paketlar bo‘lmasa — 0 ga teng. Agar bufer to‘lib borayotgan bo‘lsa — o‘nlab millisekundlarga cho‘zilishi mumkin.

#### 3. Uzatish kechikishi ($d_{\text{trans}}$ — Transmission Delay)
* Paketning barcha bitlarini routerdan chiqarib, aloqa liniyasiga (kabelga) "itarib" joylashtirish uchun ketadigan vaqt.
* Bu liniyaning uzatish tezligiga ($R$ bit/sek) va paket hajmiga ($L$ bit) bog‘liq:
  $$d_{\text{trans}} = \frac{L}{R}$$
* *E’tibor bering:* Bu masofaga mutlaqo bog‘liq emas! 1 metrli kabel bo‘ladimi yoki 10 000 km kabel bo‘ladimi, $L/R$ bir xil bo‘ladi.

#### 4. Tarqalish kechikishi ($d_{\text{prop}}$ — Propagation Delay)
* Bitta bitning jismoniy aloqa muhiti (mis, optika yoki fazo) bo‘ylab bir routerdan keyingi routergacha jismonan yugurib o‘tish vaqti.
* Bu masofaga ($d$) va to‘lqinning ushbu muhitdagi tarqalish tezligiga ($s$) bog‘liq:
  $$d_{\text{prop}} = \frac{d}{s}$$
  (Bu yerda $s$ odatda yorug‘lik tezligiga yaqin: mis yoki shishada taxminan $2 \times 10^8 \text{ m/s}$).

> **Kitobdagi Karvon (Caravan) Analogiyasi:**  
> Tasavvur qiling, 10 ta mashinadan iborat karvon yo‘lga chiqdi.  
> Yo‘lda to‘lov punkti (Tollbooth) bor.  
> * To‘lov punktining xodimi har bir mashinaga chek berib o‘tkazishi — bu **Uzatish kechikishi ($d_{\text{trans}}$)**.  
> * Mashinalarning bitta to‘lov punktidan keyingi to‘lov punktigacha trassa bo‘ylab 100 km/soat tezlikda harakatlanishi — bu **Tarqalish kechikishi ($d_{\text{prop}}$)**!

---

### 1.4.2. Navbat Kechikishi va Tirbandlik Intensivligi (Traffic Intensity)

Navbat kechikishi qachon kritik darajaga yetishini aniqlash uchun **Tirbandlik intensivligi** formulasi ishlatiladi:

$$\text{Traffic Intensity} = \frac{L \cdot a}{R}$$

Bu yerda:
* $L$ — paketning o‘rtacha uzunligi (bit).
* $a$ — router buferiga paketlarning kelish tezligi (paket/sekund).
* $R$ — liniyaning uzatish tezligi (bit/sekund).

```
Navbatdagi
Kechikish
   ^
   |                                  / (Cheksizlikka qarab ketadi)
   |                                 /
   |                                /
   |                               /
   |                   .........../
   |__________________/
   0-----------------> 0.5 ---------> 1.0 ---------> Traffic Intensity (La/R)
```

1. **Agar $\frac{La}{R} \approx 0$ bo‘lsa:** Paketlar juda kamdan-kam keladi. Navbat deyarli nolga teng.
2. **Agar $\frac{La}{R} \to 1$ ga yaqinlashsa:** Paketlarning kelishi ularning ketish tezligiga tenglashadi. Navbat keskin o‘sib, kechikish eksponentsial tarzda cheksizlikka qarab ketadi!
3. **Agar $\frac{La}{R} > 1$ bo‘lsa:** Paketlar chiquvchi liniya ulgurganidan tezroq keladi. Bufer to‘lib, paketlar tashlab yuboriladi (**Packet Loss**).

---

### 1.4.3. Oxirigacha Kechikish va Traceroute Dasturi

Paketning manbadan manzilgacha bo‘lgan to‘liq yo‘li ko‘plab routerlardan iborat:
$$d_{\text{end-end}} = \sum_{i=1}^{Q} \left( d_{\text{proc}}^{(i)} + d_{\text{queue}}^{(i)} + d_{\text{trans}}^{(i)} + d_{\text{prop}}^{(i)} \right)$$

* **Traceroute diagnostika vositasi:**
  * Tarmoqdagi oraliq routerlarni aniqlash uchun IPv4 sarlavhasidagi **TTL (Time to Live)** maydonidan foydalanadi.
  * Jo‘natuvchi ketma-ket paketlar chiqaradi:
    * 1-paket: $TTL = 1 \implies$ Birinchi router uni qabul qiladi, $TTL$ ni 0 ga tushiradi va paketni o‘chirib, jo‘natuvchiga `ICMP Time Exceeded` xabarini qaytaradi. Jo‘natuvchi 1-routerning IP si va unga borib-kelish vaqtini (RTT) bilib oladi.
    * 2-paket: $TTL = 2 \implies$ Birinchi routerdan o‘tadi, ikkinchi routerda $TTL=0$ bo‘ladi va u javob beradi.
    * Bu jarayon manzilga yetguncha davom etadi.

---

### 1.4.4. Tarmoq O‘tkazuvchanligi (Throughput)

Tarmoq orqali vaqt birligida muvaffaqiyatli uzatilgan ma’lumot miqdori (bit/sekund).
* **Lahzali o‘tkazuvchanlik (Instantaneous Throughput):** Muayyan vaqt momentidagi tezlik.
* **O‘rtacha o‘tkazuvchanlik (Average Throughput):** Masalan, $F$ bitli faylni yuklab olish uchun $T$ sekund ketgan bo‘lsa: $\text{Throughput} = F/T$.

#### Tor Bo‘g‘iz Liniyasi (Bottleneck Link)
Tasavvur qiling, ma’lumot quvur orqali oqmoqda. Agar yo‘lda keng quvurlar bo‘lsa-yu, bitta joyda quvur o‘ta ingichka bo‘lsa, butun tizimning oqim tezligi o‘sha ingichka quvur bilan cheklanadi.
Tarmoqda ham xuddi shunday: jo‘natuvchi va qabul qiluvchi o‘rtasidagi barcha zanjirda **eng past uzatish tezligiga ega bo‘lgan liniya Bottleneck Link** deb ataladi va yakuniy tezlikni aynan u belgilaydi:

$$\text{Throughput} = \min(R_1, R_2, \dots, R_n)$$

---

## 1.5. PROTOKOL QATLAMALARI VA ULARNING XIZMAT MODELLARI

Kompyuter tarmoqlari insoniyat yaratgan eng murakkab tizimlardan biridir (milliardlab qurilmalar, turli operatsion tizimlar, kabellar, sun’iy yo‘ldoshlar). Ushbu murakkablikni boshqarish uchun **Qatlamli Arxitektura (Layered Architecture)** qo‘llaniladi.

### 1.5.1. Qatlamli Tizimning Mohiyati va Afzalliklari
Har bir qatlam o‘zining yuqori qatlamiga aniq bir **xizmatni (Service)** taqdim etadi va buning uchun o‘zidan pastdagi qatlamning xizmatlaridan foydalanadi.
* **Modullilik (Modularity):** Bitta qatlamning ichki ishlash mexanizmini o‘zgartirish qolgan qatlamlarga ta’sir qilmaydi! Masalan, kompyuteringizni mis kabeldan Wi-Fi ga o‘tkazsangiz (1- va 2-qatlam o‘zgaradi), brauzeringiz yoki veb-saytlar (5-qatlam) bundan mutlaqo bexabar holda o‘zgarishsiz ishlayveradi.

---

### 1.5.2. Internetning 5 Qatlamli Modeli

```
+-----------------------------------------------------------------------------------+
| 5. APPLICATION (Dastur)     | Xabarlar (Messages): HTTP, SMTP, DNS, FTP           |
+-----------------------------------------------------------------------------------+
| 4. TRANSPORT (Transport)   | Segmentlar (Segments): TCP, UDP                      |
+-----------------------------------------------------------------------------------+
| 3. NETWORK (Tarmoq)        | Datagramlar / Paketlar (Packets): IPv4, IPv6, ICMP   |
+-----------------------------------------------------------------------------------+
| 2. DATA LINK (Kanal)       | Kadrlar (Frames): Ethernet, 802.11 Wi-Fi, PPP        |
+-----------------------------------------------------------------------------------+
| 1. PHYSICAL (Fizik)        | Bitlar (0 va 1 signallar): Mis sim, Optika, Radio    |
+-----------------------------------------------------------------------------------+
```

1. **Dastur qatlami (Application Layer):** Tarmoq ilovalari va ularning protokollari joylashgan joy. Bu yerda ma’lumot birligi — **Message (Xabar)** deyiladi.
2. **Transport qatlami (Transport Layer):** Ilovalar o‘rtasida xabarlarni yetkazib berish. Asosiy protokollar: **TCP** (ishonchli, oqim va tiqilinch nazorati bilan) va **UDP** (yengil, tezkor). Ma’lumot birligi — **Segment**.
3. **Tarmoq qatlami (Network Layer):** Paketlarni manba xostdan qabul qiluvchi xostga yetkazish. Asosiy protokol — **IP (Internet Protocol)** va marshrutlash algoritmlari (OSPF, BGP). Ma’lumot birligi — **Datagram (yoki Paket)**.
4. **Kanal qatlami (Data Link Layer):** Paketni bitta tugundan unga to‘g‘ridan-to‘g‘ri ulangan qo‘shni tugunga yetkazish (masalan, xostdan routergacha, routerdan switchgacha). Protokollar: Ethernet, Wi-Fi. Ma’lumot birligi — **Frame (Kadr)**.
5. **Fizik qatlam (Physical Layer):** Kadr ichidagi alohida bitlarni jismoniy muhit orqali signallar (elektr kuchlanishi, yorug‘lik nuri, radiochastota) ko‘rinishida uzatish.

---

### 1.5.3. Qadoqlash (Encapsulation) Mexanizmi

Jo‘natuvchi xostdan qabul qiluvchi xostgacha ma’lumot qanday kiyintiriladi va yechiladi?

```
[JO'NATUVCHI XOST]                                       [ROUTER (L3 gacha)]
Application  | Ma'lumot (M)                                        ^
     |       v                                                     |
Transport    | [ TCP Header | M ] (Segment)                        |
     |       v                                                     |
Network      | [ IP Header | TCP | M ] (Paket)  -----------------> | L3: IP ni o'qiydi
     |       v                                                     | L2: Yangi MAC qo'yadi
Data Link    | [ MAC Header | IP | TCP | M | CRC ] (Kadr)          v
     |       v                                                     |
Physical     | 01010101101010100010111010101010... (Bitlar) ------->
```

* **Encapsulation (Qadoqlash):**
  * Ilova xabar yaratadi ($M$).
  * Transport qatlami unga o‘z sarlavhasini ($H_t$ — portlar) qo‘shadi $\implies$ Segment hosil bo‘ladi.
  * Tarmoq qatlami unga o‘z sarlavhasini ($H_n$ — IP-manzillar) qo‘shadi $\implies$ Datagram (Paket) hosil bo‘ladi.
  * Kanal qatlami unga o‘z sarlavhasi ($H_l$ — MAC-manzillar) va dumi ($T_l$ — CRC xatolik nazorati) ni qo‘shadi $\implies$ Kadr hosil bo‘ladi.
* **Oraliq qurilmalar qaysi qatlamgacha ko‘radi?**
  * **Link-Layer Switchlar:** Faqat L1 va L2 qatlamlargacha ko‘radi (MAC-manzilni o‘qiydi, IP yoki TCP ga tegmaydi).
  * **Routerlar:** L1, L2 va L3 qatlamlargacha ko‘radi (Kadrni yechadi, IP sarlavhasini o‘qiydi, marshrutlaydi va yangi chiquvchi kanal uchun yangi kadrga o‘raydi).
  * **Xostlar (Mijoz va Server):** Barcha 5 ta qatlamni to‘liq ochadi va qayta ishlaydi.

---

## 1.6. TARMOQ XAVFSIZLIGI ASOSLARI (NETWORKS UNDER ATTACK)

Internet yaratilgan davrda (1960–70 yillar) foydalanuvchilar bir-biriga to‘liq ishonadigan akademik doiralar bo‘lgan. Shuning uchun protokollarga xavfsizlik va shifrlash boshidanoq kiritilmagan. Bugun esa bu jiddiy xavflarni keltirib chiqarmoqda.

### 1. Zararli dasturlar (Malware)
* **Viruslar:** Mustaqil ishlay olmaydi. Ular biror faylga (masalan, dasturga yoki hujjatga) yopishib oladi va foydalanuvchi ushbu faylni ochgandagina tizimni zararlaydi.
* **Qurtlar (Worms):** O‘ta xavfli. Foydalanuvchi aralashuvisiz, tarmoqdagi ochiq zaif portlarni skanerlab, mustaqil ravishda bir kompyuterdan ikkinchisiga o‘z-o‘zidan ko‘payib tarqaladi.
* **Botnetlar:** Qurtlar yoki troyanlar orqali zararlangan millionlab kompyuterlar bitta jinoyatchi (Botmaster) tomonidan boshqariladigan markaziy serverga (**Command and Control — C&C**) ulanadi va global kiberhujumlarda qurol sifatida ishlatiladi.

---

### 2. Xizmat Ko‘rsatishni Rad Etish (DoS va DDoS)
Tizimni qonuniy foydalanuvchilar kira olmaydigan holatga keltirish hujumi.
* **DoS turlari:**
  1. *Zaifliklarga hujum (Vulnerability attack):* Maxsus tuzilgan bir nechta buzilgan paketlarni yuborib, server operatsion tizimini qotirib qo‘yish (Crash qilish).
  2. *Tarmoqni to‘ldirish (Bandwidth Flooding):* Serverga ulanuvchi aloqa kanalining quvvatidan ortiq hajmda (masalan, 10 Gbps lik kanalga 50 Gbps hajmda) behuda paketlar toshqinini yuborish.
  3. *Ulanishlarni to‘ldirish (Connection Flooding):* Serverning xotirasini to‘ldirish uchun yarim ochiq TCP so‘rovlarini (TCP SYN Flood) tinimsiz yuborish.
* **DDoS (Distributed Denial-of-Service):**
  Hujum bitta kompyuterdan emas, balki butun dunyoga tarqalgan botnetdagi yuz minglab kompyuterlardan bir vaqtda bitta nishonga qarata uyushtiriladi. Bitta IP ni bloklash orqali himoyalanib bo‘lmaydi.

---

### 3. Paketlarni Poylash (Packet Sniffing)
* Tarmoqdagi kompyuterlar odatda faqat o‘ziga tegishli kadrlarinigina qabul qiladi.
* Lekin tarmoq kartasi **Promiscuous Mode (Aralash rejim)** ga o‘tkazilsa, u simsiz efirdan yoki umumiy switche/hub dan o‘tayotgan barcha begona paketlarni ham o‘z xotirasiga yozib oladi.
* **Wireshark** kabi dasturlar yordamida ochiq holda o‘tayotgan (HTTP, FTP, Telnet) parollar, xabarlar va bank ma’lumotlari o‘g‘irlanadi.
* **Yagona himoya chorasi:** Barcha ma’lumotlarni kriptografik shifrlash (**HTTPS, SSH, TLS, IPsec**). Shunda xaker paketni tutib olsa ham, faqat tushunarsiz raqamlar qatorini ko‘radi.

---

### 4. IP-Manzilni Soxtalashtirish (IP Spoofing)
* Paket sarlavhasidagi `Source IP` (Jo‘natuvchi IP) maydoniga o‘zining haqiqiy IP si o‘rniga boshqa bir vakolatli kompyuterning IP sini yozib yuborish.
* Natijada qabul qiluvchi server so‘rovni ishonchli manbadan keldi deb o‘ylab, javobni jabrlanuvchiga yuboradi (Reflected DDoS).
* **Himoya:** Routerlarda kiruvchi paketlarni filtrlash (**Ingress Filtering / BCP 38**) — router o‘z interfeysidan faqat o‘sha segmentga tegishli IP lar bilan chiqayotgan paketlarnigina o‘tkazishi shart.

---

## 1.7. INTERNETNING TARIXI VA EVOLYUTSIYASI

* **1961–1972: Paketlar kommutatsiyasi davri.** Leonard Kleinrock paketlar nazariyasini ishlab chiqdi. 1969-yilda AQSh Mudofaa vazirligi (DARPA) moliyalashtirgan birinchi 4 tugunli tarmoq — **ARPANET** ishga tushdi.
* **1972–1980: Tarmoqlararo bog‘lanish (Internetworking).** Vint Cerf va Robert Kahn turli mustaqil tarmoqlarni yagona tizimga bog‘lovchi **TCP/IP** arxitekturasini yaratdilar.
* **1980–1990: Standartlashuv va DNS.** 1983-yil 1-yanvarda ARPANET to‘liq TCP/IP ga o‘tdi. Xost nomlarini IP ga aylantiruvchi **DNS** tizimi yaratildi.
* **1990-yillar: Veb portlashi (The Web Explosion).** CERN laboratoriyasida Tim Berners-Lee tomonidan **HTML, HTTP, Veb-server va Brauzer** ixtiro qilindi. Internet tijorat maqsadlarida barcha uchun ochildi.
* **2000-yillardan hozirgacha:** Yuqori tezlikdagi keng polosali ulanish, Wi-Fi, Smartfonlar, Bulutli hisoblashlar (AWS, Azure), Ijtimoiy tarmoqlar va IoT inqilobi davri.

---

## 1.8. 1-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. **Tarmoq asosi:** Internet xostlar, aloqa liniyalari va paket kommutatorlaridan (router, switch) iborat bo‘lib, paketlar kommutatsiyasiga tayanadi.
2. **Kechikishlar fizikasi:** Paket yo‘lda 4 ta kechikishga uchraydi: $d_{\text{proc}}$ (protsessor tekshiruvi), $d_{\text{queue}}$ (navbatda kutish), $d_{\text{trans}}$ ($L/R$ — kabelga chiqarish) va $d_{\text{prop}}$ ($d/s$ — kabelda yugurish).
3. **Qatlamlilik:** 5 qatlamli model (Dastur $\to$ Transport $\to$ Tarmoq $\to$ Kanal $\to$ Fizik) ma’lumotlarni **Encapsulation** (qadoqlash) orqali boshqaradi.

### 🔗 Joriy papkadagi laboratoriyalar bilan bog‘liqlik:
* Ushbu bobdagi aktivlar, tahdidlar, xakerlik xurujlari (sniffing, spoofing, DoS) tushunchalari to‘g‘ridan-to‘g‘ri **`A1_project`** (Aktivlar va xavflar tahlili) hamda **`A9_project`** (Wireshark bilan paketlar tahlili) ning poydevori hisoblanadi.
* Cisco Packet Tracer da **1-Laboratoriya**ni bajarish aynan shu bobdagi Store-and-Forward va kechikish mexanizmlarini ko‘z bilan ko‘rish imkonini beradi!
