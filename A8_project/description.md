# SIMSIZ TARMOQ XAVFSIZLIGINI SOZLASH (A8 AMALIY MASHG‘ULOT | 1-VARIANT)
## Virtual Wi-Fi Muhitida WPA3-SAE va Majburiy PMF (Protected Management Frames) Sozlash Bo‘yicha Mukammal Qo‘llanma

---

### MUNDARIJA
1. [Ish Muhiti va Texnik Kontekst (Ubuntu Host + VMware Workstation + Shaxsiy Wi-Fi)](#1-ish-muhiti-va-texnik-kontekst)
2. [1-BO‘LIM: Mavzuning Maqsadi va Dolzarbligi](#2-1-bolim-mavzuning-maqsadi-va-dolzarbligi)
   - 1.1. Amaliy mashg‘ulotning asosiy maqsadi
   - 1.2. Kutiladigan natijalar
   - 1.3. Nima uchun virtual radio (`mac80211_hwsim`) ishlatiladi?
   - 1.4. Nima uchun Network Namespace (`ip netns`) zarur?
   - 1.5. Asosiy xavfsizlik tushunchalari: SAE, PMF, CCMP
3. [2-BO‘LIM: Nazariy Asoslar va Mavzular Bo‘yicha To‘liq Qo‘llanma](#3-2-bolim-nazariy-asoslar-va-mavzular-boyicha-toliq-qollanma)
   - 2.1. Simsiz tarmoqlar xavfsizligi evolyutsiyasi (WEP dan WPA3 gacha)
   - 2.2. WPA2-Personal (PSK) ning kamchiliklari va hujum vektorlari (4-Way Handshake, Offline Brute-force, KRACK)
   - 2.3. WPA3-Personal va SAE (Dragonfly Handshake, Zero-Knowledge Proof, Forward Secrecy)
   - 2.4. PMF (Protected Management Frames - IEEE 802.11w) mexanizmi va chegaralari
   - 2.5. Ma’lumotlarni shifrlash: CCMP (AES-128 CCM)
   - 2.6. RSN (Robust Security Network) va Information Elements (IE)
   - 2.7. Linux `mac80211_hwsim` virtual efir arxitekturasi
   - 2.8. `hostapd` va `wpa_supplicant` ishlash tamoyillari
   - 2.9. Katta korxonalar arxitekturasi: WPA3-Personal cheklovlari va WPA3-Enterprise (802.1X/RADIUS) zarurati
4. [3-BO‘LIM: Mashqni Bajarish Bo‘yicha Bosqichma-Bosqich Qo‘llanma va CLI Buyruqlar Tahlili](#4-3-bolim-mashqni-bajarish-boyicha-bosqichma-bosqich-qollanma-va-cli-buyruqlar-tahlili)
   - 3.1. 0-bosqich: Virtual laboratoriya muhitini tayyorlash (Ubuntu Host -> VMware -> Ubuntu Server VM)
   - 3.2. 1-bosqich: Dasturiy ta’minotlarni o‘rnatish va tekshirish
   - 3.3. 2-bosqich: Virtual radiolarni yuklash va AP hamda STA ni alohida namespacelarga ajratish
   - 3.4. 3-bosqich: Boshlang‘ich holat — WPA2-Personal ulanishini sinash (T1 Testi)
   - 3.5. 4-bosqich: WPA3-Personal (SAE) va Majburiy PMF ga o‘tish (T2 Testi)
   - 3.6. 5-bosqich: Salbiy testlar — Parol va xavfsizlik profili mosligini tekshirish (T3 va T4 Testlari)
   - 3.7. 6-bosqich: Tizimni qayta tiklash (T5 Testi)
   - 3.8. 7-bosqich (Chuqurlashtirilgan): Virtual Efir Trafigini Qamrab Olish (PCAP, tcpdump, tshark)
   - 3.9. 8-bosqich: Audit hisobotini tayyorlash, maxfiy ma’lumotlarni tozalash va SHA-256 xesh ro‘yxati
   - 3.10. 9-bosqich: Dalillarni Ubuntu Server VM dan Ubuntu Host tizimiga xavfsiz ko‘chirish
5. [4-BO‘LIM: Nosozliklarni Aniqlash va Tuzatish (Troubleshooting Guide)](#5-4-bolim-nosozliklarni-aniqlash-va-tuzatish-troubleshooting-guide)
6. [5-BO‘LIM: Natijalarni Qayd Etish Jadvallari va Hisobot Namunasi](#6-5-bolim-natijalarni-qayd-etish-jadvallari-va-hisobot-namunasi)
   - 5.1. 3.1-jadval: Muhit va himoya sozlamalari
   - 5.2. 3.2-jadval: Ulanish tajribalari (T1–T5)
   - 5.3. 3.3-jadval: Dalil va xulosa o‘rtasidagi bog‘lanish
   - 5.4. 3.4-bo‘lim: Yakuniy professional xulosa (6–8 gap)
7. [6-BO‘LIM: Himoya Savollari va Ularning Mukammal Javoblari (1–12)](#7-6-bolim-himoya-savollari-va-ularning-mukammal-javoblari-112)
8. [7-BO‘LIM: Foydalanilgan va Tavsiya Etiladigan Manbalar](#8-7-bolim-foydalanilgan-va-tavsiya-etiladigan-manbalar)

---

## 1. Ish Muhiti va Texnik Kontekst

Ushbu laboratoriya qo‘llanmasi foydalanuvchining real ish muhiti sharoitlarini hisobga olgan holda maxsus moslashtirilgan:
1. **Host Operatsion Tizimi**: **Ubuntu Linux** (Windows emas!). Foydalanuvchining asosiy kompyuterida Ubuntu ishlamoqda.
2. **Virtualizatsiya Platformasi**: **VMware Workstation Pro / Player for Linux**.
3. **Mehmon Virtual Mashina (Guest OS)**: **Ubuntu Server 24.04 LTS**.
4. **Tarmoq Ulanishi**: **Shaxsiy (Lokal/Uy) Wi-Fi tarmog‘i** (Universitet Wi-Fi tarmog‘i emas!).
   - *Tayyorgarlik davrida*: Ubuntu host o‘zining shaxsiy Wi-Fi routeriga ulangan bo‘ladi. VMware NAT adapteri orqali Ubuntu Server VM ga internet beriladi va barcha zaruriy paketlar (`hostapd`, `wpa_supplicant`, `iw`, va h.k.) ombordan yuklab olinadi.
   - *Laboratoriya sinovlari davrida*: Paketlar o‘rnatilgach, VMware sozlamalaridan VM ning virtual tarmoq kabeli uziladi (**Disconnected** qilinadi). Laboratoriya to‘liq dasturiy `mac80211_hwsim` drayveri orqali VM ning ichki virtual efirida kechadi. Natijada foydalanuvchining shaxsiy Wi-Fi tarmog‘iga hech qanday soxta paketlar, qoidabuzarliklar yoki xalaqitlar yetkazilmaydi va laboratoriya mutlaqo izolyatsiyalangan holda o‘tadi.
5. **Eksport va Dalillarni Ko‘chirish**: Natijalar Windows emas, Ubuntu host tizimiga VMware Shared Folders (`/mnt/hgfs`), `scp`, yoki virtual xotira diski orqali ko‘chiriladi.

---

## 2. 1-BO‘LIM: Mavzuning Maqsadi va Dolzarbligi

### 1.1. Amaliy Mashg‘ulotning Asosiy Maqsadi
Ushbu amaliy mashg‘ulotning bosh maqsadi — zamonaviy simsiz tarmoqlarda (Wi-Fi) xavfsizlik arxitekturasini loyihalash, eskirgan WPA2-Personal standartidan eng so‘nggi **WPA3-Personal (SAE — Simultaneous Authentication of Equals)** standartiga o‘tish hamda **Majburiy PMF (Protected Management Frames — IEEE 802.11w)** mexanizmining ishlashini to‘liq virtual laboratoriya muhitida amaliy jihatdan tadqiq qilishdan iborat.

Talaba quyidagi fundamental vazifalarni bajaradi:
1. Linux yadrosining `mac80211_hwsim` drayveridan foydalanib, fizik Wi-Fi adapterisiz, to‘liq dasturiy 802.11 radiosimulyatsiya efirini barpo etadi.
2. Boshlang‘ich holat sifatida eskirgan va himoyalanmagan boshqaruv kadrlariga ega **WPA2-Personal (PSK)** tarmog‘ini sozlaydi (T1).
3. Kirish nuqtasi (Access Point) va mijozni (Station) faqat **WPA3-Personal (SAE)**, majburiy **PMF (`ieee80211w=2`)** va **CCMP** shifrlashiga o‘tkazadi (T2).
4. Havoda tarqatilayotgan **RSN (Robust Security Network)** axborot elementlarini (IE), e’lon qilingan va ikki tomonlama kelishilgan xavfsizlik siyosatini tahlil qiladi.
5. Salbiy testlar (Negative Testing) o‘tkazadi:
   - Noto‘g‘ri parol kiritilganda SAE mexanizmining xatti-harakatini tekshiradi (T3).
   - WPA3-only va majburiy PMF ga ega AP ga faqat WPA2-PSK ni qo‘llovchi eski mijoz bilan ulanishga urinish oqibatlarini o‘rganadi (T4).
6. To‘g‘ri WPA3 parametrlarini tiklab, tarmoq aloqasini qayta tiklaydi (T5).
7. Xavfsizlik auditini o‘tkazadi: parollarni konfiguratsiya fayllaridan xavfsiz tozalash, fayl ruxsatlarini (permissions) cheklash va dalillar yaxlitligini SHA-256 xeshlash orqali himoyalash.

---

### 1.2. Kutiladigan Natijalar
Mashg‘ulot yakunida talaba quyidagi kompetensiyalarga ega bo‘ladi:
* **Ulanish fazalarini aniq farqlash**: Wi-Fi ulanishidagi uchta mustaqil bosqichni bir-biridan ajrata oladi:
  1. *SSID efirda ko‘rindi* (Beacon va Probe javoblari qabul qilinmoqda, lekin bu ulanish mavjudligini bildirmaydi).
  2. *Autentifikatsiya va assotsiatsiya tugadi* (`wpa_state=COMPLETED`, kriptografik kalitlar muvaffaqiyatli kelishildi).
  3. *IP sathi ishlamoqda* (Marshrutlash va ICMP ping orqali L3 darajasida paketlar almashinmoqda).
* **PMF ning himoya chegarasini anglash**: PMF qaysi boshqaruv kadrlarini (Deauthentication / Disassociation) himoyalashi, ammo nima sababdan radio to‘sish (RF Jamming) yoki xizmat ko‘rsatishni rad etish (DoS) kabi fizik hujumlarni to‘xtata olmasligini ilmiy asoslab bera oladi.
* **Xavfsizlik profili nomuvofiqligini aniqlash**: Ulanish muvaffaqiyatsiz bo‘lganda, sabab noto‘g‘ri paroldanmi yoki xavfsizlik profillari (AKM, PMF talablari) mos kelmasligidan ekanligini tizim loglari orqali tahlil qila oladi.

---

### 1.3. Nega Virtual Radio (`mac80211_hwsim`) Ishlatiladi?
Virtualizatsiya tizimlarida (VMware Workstation, VirtualBox) mehmon operatsion tizimiga odatda virtual **Ethernet** adapteri (masalan, `e1000`, `vmxnet3`) taqdim etiladi. Host tizimi (Ubuntu) Internetga Wi-Fi orqali ulangan bo‘lsa ham, VMware buni mehmon OTga simli Ethernet kabeli sifatida ko‘rsatadi. Mehmon OT ichida Wi-Fi kirish nuqtasi (AP) yoki Wi-Fi skaneri yaratish uchun esa tizimga haqiqiy **IEEE 802.11** simsiz radio interfeysi kerak.

Buning uchun odatda USB Wi-Fi adapterlari talab etiladi. Biroq:
1. Ko‘plab kompyuterlarda ortiqcha tashqi USB Wi-Fi adapteri mavjud bo‘lmasligi mumkin.
2. Tashqi USB Wi-Fi adapterlarining chip va drayverlari (masalan, Realtek, Mediatek) har doim ham Access Point (AP) rejimini va WPA3-SAE ni to‘liq qo‘llab-quvvatlamaydi.
3. Efirda begona tarmoqlar, shovqinlar va xalaqitlar laboratoriya tajribalarining sofligiga putur yetkazadi.

**Yechim**: Linux yadrosining `mac80211_hwsim` (Hardware Simulator) drayveri.
* Ushbu drayver xotirada to‘liq dasturiy IEEE 802.11 virtual radiolarini (masalan, `phy0`, `phy1`) yaratadi.
* Bu radiolar virtual efir orqali bir-birini ko‘radi va haqiqiy Wi-Fi drayverlari kabi `nl80211` / `mac80211` yadro steki bilan ishlaydi.
* Natijada, apparat ta’minotidan qat’i nazar, istalgan Linux tizimida 100% barqaror, boshqariladigan va xavfsiz Wi-Fi tajribalarini o‘tkazish mumkin bo‘ladi.

---

### 1.4. Nega Network Namespace (`ip netns`) Zarur?
Laboratoriyada bitta virtual mashina ichida ham Kirish Nuqtasi (AP — `hostapd`), ham Mijoz (STA — `wpa_supplicant`) ishga tushiriladi.
Agar ikkala virtual radio interfeysi (`ap0` va `sta0`) Linuxning bitta umumiy tarmoq muhitida (default network namespace) qolsa:
* `ap0` ga `10.80.10.1/24`, `sta0` ga esa `10.80.10.20/24` IP manzili berilganda, Linux yadrosi marshrutlashni optimallashtirish maqsadida paketlarni **Loopback (mahalliy xotira)** orqali uzatadi.
* Ya’ni, talaba `ping 10.80.10.1` buyrug‘ini berganda, paketlar Wi-Fi radiosi (`mac80211_hwsim`) orqali efirga chiqmaydi, balki to‘g‘ridan-to‘g‘ri yadroning ichki xotirasida manzilga yetib boradi. Bu holatda Wi-Fi shifrlash ishlamasa ham, hatto Wi-Fi uzilib qolsa ham ping javob beraveradi!

**Yechim**: Linux Network Namespaces (`ip netns`).
* AP radiosi (`ap0`) asosiy tizimda qoldiriladi.
* STA radiosi (`sta0`) esa `a8sta` nomli alohida izolyatsiyalangan tarmoq nomlar fazosiga (namespace) ko‘chiriladi.
* Natijada, `a8sta` ichidan yuborilgan har bir paket faqat va faqat virtual Wi-Fi efiri (802.11 kadrlari, CCMP shifrlash, WPA3 kanali) orqali `ap0` ga o‘tishga majbur bo‘ladi. Bu laboratoriya natijalarining 100% haqqoniyligini ta’minlaydi.

---

### 1.5. Asosiy Xavfsizlik Tushunchalari
* **SAE (Simultaneous Authentication of Equals)**: WPA3-Personal uchun parol asosida o‘zaro teng huquqli autentifikatsiya va maxfiy sessiya kalitini kelishish protokoli (RFC 7664, Dragonfly algoritmi). U eskirgan WPA2 4-tomonlama handshakedagi offline parolni taxmin qilish (brute-force) zaifligini butunlay yo‘q qiladi.
* **PMF (Protected Management Frames - IEEE 802.11w)**: Tarmoq boshqaruv kadrlarini (xususan, Deauthentication va Disassociation) kriptografik jihatdan himoyalash mexanizmi. Hujumchilar soxta MAC manzil orqali mijozni tarmoqdan uzib tashlash (Deauth Attack) imkoniyatidan mahrum bo‘ladi.
* **CCMP (Counter Mode with CBC-MAC Protocol)**: 128-bitli AES shifrlashiga asoslangan ma’lumotlar maxfiyligi va yaxlitligini ta’minlovchi standart protokol.
* **SSID Yashirish va MAC Filtr haqida ogohlantirish**: SSID nomini yashirish (hidden SSID) yoki MAC oq ro‘yxati (whitelisting) kriptografik xavfsizlik chorasi hisoblanmaydi. Havaskor hujumchi ham ochiq efirdan MAC manzilni nusxalab (spoofing) olishi va yashirin SSID nomini osonlik bilan aniqlashi mumkin. Haqiqiy xavfsizlik faqat kuchli kriptografiya (WPA3-SAE va PMF) orqali kafolatlanadi.

---

## 3. 2-BO‘LIM: Nazariy Asoslar va Mavzular Bo‘yicha To‘liq Qo‘llanma

### 2.1. Simsiz Tarmoqlar Xavfsizligi Evolyutsiyasi
Simsiz tarmoqlarda uzatilayotgan signallar ochiq havoda tarqalgani sababli, har qanday radio qabul qiluvchi qurilma efirni tinglashi mumkin. Shu sababli Wi-Fi standartlari doimiy ravishda takomillashib borgan:

| Standart | Yil | Asosiy Algoritm | Kalit Boshqaruvi | Boshqaruv Kadrlari | Asosiy Kamchiliklari |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **WEP** | 1997 | RC4 (oqimli shifr), CRC-32 | Statik kalit (40/104 bit) | Ochiq (himoyasiz) | 24-bitli IV takrorlanishi; bir necha daqiqada parolni sindirish mumkin |
| **WPA** | 2003 | TKIP (RC4 takomillashgani), Michael MIC | PSK / 802.1X | Ochiq (himoyasiz) | Vaqtinchalik chora; hisoblash quvvati o‘sishi bilan zaiflashdi (Beck-Tews hujumi) |
| **WPA2** | 2004 | AES-CCMP | PSK (4-Way Handshake) | Ochiq (PMF ixtiyoriy — 802.11w) | Offline lug‘at hujumi (Dictionary Attack), KRACK, Deauth hujumlari |
| **WPA3** | 2018 | AES-CCMP (Personal) / GCMP-256 (Enterprise) | SAE (Dragonfly) / 802.1X CNSA | **Majburiy PMF** | Yangi standart; apparat va dasturiy moslikni talab qiladi |

---

### 2.2. WPA2-Personal (PSK) ning Kamchiliklari va Hujum Vektorlari
WPA2-Personal tizimida mijoz va kirish nuqtasi umumiy bir xil parolga (Pre-Shared Key — PSK) ega bo‘ladi. Ulanish jarayonida **4-tomonlama qo‘l berishish (4-Way Handshake)** sodir bo‘ladi:
1. AP mijozga tasodifiy son jo‘natadi (**ANonce**).
2. Mijoz o‘zining tasodifiy sonini (**SNonce**) va tarmoq paroli asosida hisoblangan **MIC (Message Integrity Code)** ni AP ga yuboradi.
3. AP tasdiqlash xabarini va guruh kalitini (**GTK**) jo‘natadi.
4. Mijoz ulanish muvaffaqiyatli yakunlanganini bildiradi.

```
+-----------------------------------------------------------------------------------+
|                        WPA2 4-Way Handshake Zaifligi                             |
+-----------------------------------------------------------------------------------+
|  [Kirish Nuqtasi (AP)]                                  [Mijoz (STA)]            |
|            |                                                 |                    |
|            |------- 1. ANonce ------------------------------>|                    |
|            |<------ 2. SNonce + MIC (EAPOL) -----------------| <-- Hujumchi       |
|            |------- 3. GTK + MIC (EAPOL) ------------------->|     ushlab oladi   |
|            |<------ 4. ACK ----------------------------------|     (Capture)      |
|            |                                                 |                    |
|  [Hujumchi (Offline Kompyuter)]:                                                 |
|  Parollar lug'ati (wordlist) -> PBKDF2 -> PMK -> PTK hisoblash -> MIC tekshirish |
|  * Hech qanday AP ga qayta so'rov yubormasdan, sekundiga millionlab parollarni     |
|    GPU (Hashcat) orqali tekshirib, haqiqiy Wi-Fi parolini topib oladi!            |
+-----------------------------------------------------------------------------------+
```

#### WPA2 ning 3 ta asosiy zaifligi:
1. **Offline Lug‘at va Brute-Force Hujumi**: Hujumchi 4-Way Handshake ning 1- va 2-bosqichidagi EAPOL kadrlarini havoda ushlab olsa bo‘ldi. Shundan so‘ng u tarmoqqa qayta murojaat qilmasdan, o‘zining kuchli grafik kartasi (GPU) orqali soniyasiga milliardlab parollarni sinab ko‘radi. Agar parol oddiy so‘z bo‘lsa, u bir necha daqiqada ochiladi.
2. **Oldingi Trafikni Ocha Olish (Lack of Forward Secrecy)**: Agar tajovuzkor tarmoqning barcha shifrlangan trafigini (pcap faylga) yozib borsa va 6 oydan keyin tarmoq parolini bilib olsa, u o‘tmishda yozib olingan barcha maxfiy ma’lumotlarni shifrdan yechib o‘qiy oladi!
3. **Boshqaruv Kadrlarining Ochiqligi (Deauthentication Attack)**: WPA2 da boshqaruv kadrlari shifrlanmaydi va imzolanmaydi. Hujumchi kirish nuqtasi nomidan soxta `Deauth` kadrlarini yuborib, istalgan mijozni tarmoqdan uzib qo‘yishi mumkin.

---

### 2.3. WPA3-Personal va SAE (Simultaneous Authentication of Equals)
WPA3-Personal ushbu barcha muammolarni bartaraf etish uchun **SAE (Teng huquqlilarning bir vaqtda autentifikatsiyasi)** mexanizmini kiritdi. SAE ning asosi — **Dragonfly Handshake** (RFC 7664) protokolidir.

```
+-----------------------------------------------------------------------------------+
|                           WPA3-SAE (Dragonfly Handshake)                          |
+-----------------------------------------------------------------------------------+
|  [Kirish Nuqtasi (AP)]                                  [Mijoz (STA)]            |
|            |                                                 |                    |
|            |<==== 1. SAE Commit (Scalar_A, Element_A) ======>|                    |
|            |                                                 |                    |
|            |<==== 2. SAE Confirm (Send-Confirm_A, _B) ======>|                    |
|            |                                                 |                    |
|  * Ikkala tomon parol asosida PWE (Password Element) hosil qiladi.               |
|  * Diffie-Hellman / Elliptic Curve (ECC) yordamida umumiy PMK kelishiladi.       |
|  * Efirda parolni ochib beruvchi HESH yoki MIC uzatilmaydi!                       |
|  * Hujumchi xabarlarni ushlab olgani bilan OFFLINE parolni tekshira olmaydi!      |
+-----------------------------------------------------------------------------------+
```

#### SAE ning inqilobiy afzalliklari:
1. **Nol Bilimli Isbot (Zero-Knowledge Proof)**: Tomonlar parolni to‘g‘ridan-to‘g‘ri yoki xesh ko‘rinishida uzatmasdan, o‘zlarida ushbu parol borligini matematik tarzda isbotlaydilar.
2. **Offline Hujumlardan 100% Himoya**: Efirdan olingan kadrlar orqali parolni offline tekshirib bo‘lmaydi. Har bir noto‘g‘ri parolni tekshirish uchun hujumchi AP ga real vaqt rejimida so‘rov yuborishi shart. AP esa ketma-ket bir necha xato urinishdan so‘ng blokirovka (rate limiting) qo‘yadi.
3. **Mukammal To‘g‘ridan-to‘g‘ri Maxfiylik (PFS — Perfect Forward Secrecy)**: Har bir sessiya uchun yangi efemer Elliptik Egri chiziq (ECC) kalitlari yaratiladi. Kelajakda tarmoq paroli oshkor bo‘lsa ham, o‘tmishdagi yozib olingan trafikni hech kim shifrdan yecha olmaydi.
4. **PWE (Password Element) Generatsiyasi**: Parol asosida egri chiziqda nuqta hosil qilish usuli. WPA3 da `sae_pwe=2` parametri *Hash-to-Element* va *Hunting-and-Pecking* usullarini boshqaradi, bu esa yon kanal (side-channel) orqali parolni aniqlash hujumlaridan (Dragonblood zaifligi) himoyalaydi.

---

### 2.4. PMF (Protected Management Frames - IEEE 802.11w) Mexanizmi va Chegaralari
Oddiy 802.11 tarmoqlarida 3 xil kadr mavjud:
1. **Data Frames (Ma’lumot kadrlari)** — foydalanuvchi trafigi (IP, TCP, UDP).
2. **Control Frames (Boshqaruv kadrlari)** — RTS, CTS, ACK.
3. **Management Frames (Tashkiliy/Ma’muriy kadrlar)** — Beacon, Probe Request/Response, Association Request/Response, Authentication, Deauthentication, Disassociation.

WPA2 da faqat *Data* kadrlari shifrlangan, *Management* kadrlari esa butunlay ochiq matn (plain-text) ko‘rinishida bo‘lgan.

#### IEEE 802.11w (PMF) nima beradi?
PMF kadrlar sarlavhasiga **MMIE (Management MIC Information Element)** qo‘shadi va muhim boshqaruv kadrlarini (Robust Management Frames) AES-CMAC yoki CCMP orqali kriptografik imzolaydi va shifrlaydi.
* **Unicast Deauthentication / Disassociation**: Shifrlanadi va imzolanadi. Begona shaxs soxta kadr yuborsa, qabul qiluvchi tomon MIC xatosini ko‘rib, kadrni tashlab yuboradi (Drop qiladi).
* **Broadcast Deauthentication**: Ochiq uzatiladi, lekin uning ichiga kirish nuqtasining shaxsiy maxfiy kaliti bilan hisoblangan raqamli imzo (BIP — Broadcast Integrity Protocol) joylashtiriladi. Soxta imzo darhol fosh bo‘ladi.

```
+-----------------------------------------------------------------------------------+
|                        PMF Nimalarni Himoyalaydi va Nimalarni Yo'q?               |
+-----------------------------------------------------------------------------------+
|  [HIMOYA QILINADI (802.11w PMF)]:                                                 |
|  [+] Deauthentication hujumlari (tarmoqdan uzib tashlash)                        |
|  [+] Disassociation hujumlari                                                     |
|  [+] Action kadrlarini soxtalashtirish (Radio Measurement, BSS Transition)        |
|                                                                                   |
|  [HIMOYA QILINMAYDI (PMF ning chegarasi)]:                                        |
|  [-] Radio To'sish (RF Jamming / Fizik shovqin qo'zg'ash)                          |
|  [-] Beacon ochiqligi (SSID, tarmoq parametrlari ochiq tarqatiladi)              |
|  [-] Probe Request / Probe Response (ulanishdan oldingi kadrlar)                  |
|  [-] L4-L7 darajadagi hujumlar (Fishing, MITM, zararli dasturlar, DNS soxtaligi)   |
+-----------------------------------------------------------------------------------+
```

#### PMF Sozlama Holatlari:
* `ieee80211w=0`: **Disabled** (O‘chirilgan) — an’anaviy WPA2 holati, barcha boshqaruv kadrlari himoyasiz.
* `ieee80211w=1`: **Capable / Optional** (Ixtiyoriy) — mijoz PMF ni qo‘llasa himoyalanadi, qo‘llamasa oddiy rejimda ulanadi (WPA2/WPA3 o‘tish rejimida ishlatiladi).
* `ieee80211w=2`: **Required** (Majburiy) — kirish nuqtasi FAQAT PMF ni to‘liq qo‘llaydigan mijozlarni qabul qiladi. PMF ni qo‘llamaydigan mijozlar tarmoqqa umuman kiritilmaydi (WPA3-Personal talabi).

---

### 2.5. Ma’lumotlarni Shifrlash: CCMP (AES-128 CCM)
Ushbu variantda ma’lumotlar kadrlarini shifrlash uchun **CCMP (Counter Mode with Cipher Block Chaining Message Authentication Code Protocol)** ishlatiladi:
* **AES Counter Mode (CTR)**: Ma’lumotlar oqimining maxfiyligini (confidentiality) ta’minlaydi.
* **CBC-MAC**: Kadrning yaxlitligi (integrity) va manba haqiqiyligini (replay protection uchun 48-bitli Packet Number - PN) kafolatlaydi.
* WPA3-Personal da CCMP standarti saqlab qolingan, biroq uning kalit kelishish mexanizmi WPA-PSK dan SAE ga almashtirilgan.

---

### 2.6. RSN (Robust Security Network) va Information Elements (IE)
Wi-Fi kirish nuqtasi har 100 millisekundda efirga **Beacon** (mayoq) kadrlarini tarqatadi. Ushbu kadr ichida **RSN Information Element (RSN IE, Element ID: 48)** mavjud bo‘lib, u tarmoqning xavfsizlik qoidalarini e’lon qiladi:
* **AKM Suite (Autentifikatsiya va kalit boshqaruvi)**:
  * `00-0F-AC:2` — WPA-PSK (WPA2-Personal)
  * `00-0F-AC:8` — WPA-SAE (WPA3-Personal)
* **Pairwise Cipher Suite**: `00-0F-AC:4` (CCMP)
* **RSN Capabilities (Imkoniyatlar bayroqlari)**:
  * **Bit 6 (MFP Capable)**: Kirish nuqtasi PMF ni tushunishini bildiradi.
  * **Bit 7 (MFP Required)**: Kirish nuqtasi PMF ni majburiy talab qilishini bildiradi.

Mijoz skanerlash o‘tkazganda (`iw scan`), aynan shu RSN blokini o‘qiydi va o‘zining imkoniyatlari AP talablariga mos kelishini aniqlaydi.

---

### 2.7. Linux `mac80211_hwsim` Virtual Efir Arxitekturasi
`mac80211_hwsim` — bu Linux yadrosidagi simulyator drayveri bo‘lib, dasturiy virtual Wi-Fi qurilmalarini hosil qiladi.
* Drayver yuklanganda (`modprobe mac80211_hwsim radios=2`), yadroda ikkita radio (`phy0` va `phy1`) hamda global monitoring interfeysi (`hwsim0`) paydo bo‘ladi.
* Bitta radiodan yuborilgan har bir 802.11 kadr drayver orqali ikkinchi radioga yetkaziladi. Xuddi haqiqiy antennalar kabi, ular bir xil chastotada (masalan, 2412 MHz / 1-kanal) ishlaganda kadrlar almashinuvi yuz beradi.
* Ushbu simulyatsiya faqat bitta kompyuter (Linux VM) ichida ishlaydi, tashqi fizik efirga chiqmaydi va tashqi efir signallarini qabul qilmaydi.

---

### 2.8. `hostapd` va `wpa_supplicant` Ishlash Tamoyillari
* **hostapd (Host Access Point Daemon)**: Linux tizimini to‘liq Wi-Fi kirish nuqtasiga (routerga) aylantiruvchi xizmat. U 802.11 Beacon kadrlarini tarqatish, autentifikatsiya qilish (SAE/PSK), mijozlarni assotsiatsiya qilish va shifrlash kalitlarini (PTK, GTK, IGTK) boshqarish vazifasini bajaradi.
* **wpa_supplicant**: Wi-Fi mijozi sifatida ishlovchi dastur. U efirni skanerlaydi, kerakli SSID ni topadi, AP bilan autentifikatsiya o‘rnatadi, parolni taqdim etadi va IP aloqa uchun interfeysni tayyorlaydi.
* Har ikki dastur `nl80211` netlink interfeysi orqali bevosita Linux yadrosidagi `mac80211` quyi tizimi bilan muloqot qiladi.

---

### 2.9. Katta Korxonalar Arxitekturasi: Nega WPA3-Personal Yetarli Emas?
WPA3-Personal uy va kichik ofislar uchun mukammal himoya taqdim etsa-da, yirik tashkilotlar (universitetlar, korporatsiyalar, banklar) uchun quyidagi jiddiy cheklovga ega:
* **Umumiy Parol Muammosi**: Barcha xodimlar bitta umumiy paroldan foydalanadi.
* **Xodimni Alohida Cheklash (Revocation) Imkonsizligi**: Agar bitta xodim ishdan bo‘shasa yoki uning qurilmasi o‘g‘irlansa, administrator butun tashkilotdagi barcha xodimlarning kompyuter va telefonlaridagi Wi-Fi parolini o‘zgartirib chiqishga majbur bo‘ladi!

**Yirik Tashkilotlar Yechimi — WPA3-Enterprise**:
* **802.1X / EAP (Extensible Authentication Protocol)** arxitekturasi va markaziy **RADIUS (masalan, FreeRADIUS)** serveri qo‘llaniladi.
* Har bir xodim o‘zining shaxsiy login/paroli (Active Directory / LDAP) yoki shaxsiy raqamli xavfsizlik sertifikati (PKI / TLS) orqali tarmoqqa kiradi.
* Agar xodim ishdan ketsa, administrator RADIUS serverda faqat o‘sha xodimning akkauntini o‘chirib qo‘yadi; tarmoqdagi qolgan yuzlab xodimlarning sozlamalariga tegilmaydi.

---

## 4. 3-BO‘LIM: Mashqni Bajarish Bo‘yicha Bosqichma-Bosqich Qo‘llanma va CLI Buyruqlar Tahlili

### 3.1. 0-bosqich: Virtual Laboratoriya Muhitini Tayyorlash

> [!IMPORTANT]
> **Host: Ubuntu Linux | Dastur: VMware Workstation | Tarmoq: Shaxsiy Wi-Fi**  
> Ushbu amaliyotda host kompyuteringiz Ubuntu tizimida ishlamoqda. Siz laboratoriyani Ubuntu ichidagi VMware Workstation yordamida yaratilgan **Ubuntu Server 24.04 LTS** virtual mashinasida bajarasiz.

#### 1-Qadam: VM yaratish va tarmoq sozlamasi
1. VMware Workstation dasturini oching va yangi **Ubuntu Server 24.04 LTS** virtual mashinasini yarating (Parametrlar: 2 vCPU, 4 GB RAM, 25 GB disk).
2. VM tarmoq adapterini **NAT** rejimiga qo‘ying.
3. Ubuntu host kompyuteringiz o‘zingizning **shaxsiy lokal Wi-Fi** tarmog‘ingizga ulangan bo‘lsin.
4. VM ni ishga tushiring va unga kiring.

#### 2-Qadam: Paketlarni o‘rnatish va tekshirish
VM konsolida internet mavjudligini tekshiring va quyidagi buyruqlarni bajaring:

```bash
sudo apt update
sudo apt install -y hostapd wpasupplicant iw iproute2 rfkill tcpdump
sudo env DEBIAN_FRONTEND=noninteractive apt install -y tshark
```

**Buyruqlarning vazifalari:**
* `sudo apt update`: Paketlar ro‘yxatini eng so‘nggi versiyalarga yangilaydi.
* `hostapd`: Virtual kirish nuqtasi (AP) dasturi.
* `wpasupplicant`: Wi-Fi mijozi dasturi.
* `iw`: Zamonaviy Linux simsiz tarmoq boshqaruv utilitasi (`nl80211`).
* `iproute2`: Tarmoq interfeyslari, marshrutlash va namespacelarni boshqarish vositalari to‘plami (`ip netns`, `ip link`, `ip addr`).
* `rfkill`: Radiochastotalarni dasturiy bloklash va ochish utilitasi.
* `tcpdump`: Tarmoq paketlarini qamrab olish (sniffing) vositasi.
* `DEBIAN_FRONTEND=noninteractive apt install -y tshark`: `tshark` (Wiresharkning konsol varianti) ni interaktiv savollarsiz (non-interactive) avtomatik o‘rnatish.

Endi yadro versiyasi va zaruriy modullarni tekshiring:

```bash
uname -r
modinfo mac80211_hwsim
hostapd -v
wpa_supplicant -v
```

**Buyruqlarning vazifalari:**
* `uname -r`: Joriy ishlayotgan Linux yadrosi versiyasini ko‘rsatadi (masalan, `6.8.0-xx-generic`).
* `modinfo mac80211_hwsim`: Tizimda virtual Wi-Fi drayveri moduli mavjudligini tekshiradi.
* `hostapd -v` va `wpa_supplicant -v`: Dasturlarning versiyasini tekshiradi (WPA3-SAE qo‘llab-quvvatlanishi uchun v2.10 yoki undan yuqori bo‘lishi lozim).

> [!NOTE]
> Agar `modinfo mac80211_hwsim` buyrug‘i `ERROR: Module mac80211_hwsim not found` xatosini bersa, qo‘shimcha yadro modullarini o‘rnating:
> ```bash
> sudo apt install -y linux-modules-extra-$(uname -r)
> ```

#### 3-Qadam: VM ni Tashqi Tarmoqdan To‘liq Uzish (Izolyatsiya)
Paketlar o‘rnatilgach, VM ni o‘chiring (`sudo poweroff`).
1. VMware Workstation oynasida virtual mashina sozlamalariga kiring (**VM -> Settings -> Network Adapter**).
2. **"Connected"** va **"Connect at power on"** kataklaridagi belgilarni olib tashlang (virtual tarmoq kabelini uzing).
3. VM ni qayta ishga tushiring.
4. *Natija*: Virtual mashina tashqi shaxsiy Wi-Fi dan mutlaqo uziladi. Endilikda barcha testlar faqat ichki virtual `mac80211_hwsim` efirida kechadi.

---

### 3.2. 1-bosqich: Ishchi Muhitni Shakllantirish

VM qayta yuklangach, barcha buyruqlar **root** foydalanuvchisi sifatida bitta konsol sessiyasida bajariladi.

```bash
sudo -i
mkdir -p /root/A8/report /run/a8
chmod 700 /root/A8 /root/A8/report
cd /root/A8
systemctl stop hostapd wpa_supplicant
```

**Har bir buyruqning batafsil tahlili:**
* `sudo -i`: Root foydalanuvchisining to‘liq interaktiv qobig‘iga (root shell) o‘tadi, barcha muhit o‘zgaruvchilarini root huquqida yuklaydi.
* `mkdir -p /root/A8/report /run/a8`:
  * `/root/A8`: Laboratoriyaning barcha konfiguratsiya fayllari va loglari saqlanadigan asosiy papka.
  * `/root/A8/report`: O‘qituvchiga topshiriladigan audit dalillari (status, ping, siyosat matnlari) to‘planadigan maxsus papka.
  * `/run/a8`: Tezkor xotirada (RAM — `tmpfs`) dasturlarning boshqaruv soketlari (`ctrl_interface`) va PID fayllari joylashadigan katalog.
  * `-p` bayrog‘i ota kataloglar mavjud bo‘lmasa ularni xatoliksiz yaratishni bildiradi.
* `chmod 700 /root/A8 /root/A8/report`: Katalog huquqlarini `rwx------` ko‘rinishida cheklaydi. Undagi fayllarni root dan boshqa hech qanday foydalanuvchi o‘qiy olmaydi yoki o‘zgartira olmaydi (xavfsizlik talabi).
* `cd /root/A8`: Ishchi katalogga o‘tadi.
* `systemctl stop hostapd wpa_supplicant`: Ubuntu tizimida standart avtomatik ishga tushuvchi `hostapd` va `wpa_supplicant` xizmatlarini to‘xtatadi. Sababi biz ularni orqa fonda o‘zimiz yozgan maxsus konfiguratsiyalar bilan qo‘lda boshqaramiz.

---

### 3.3. 2-bosqich: Virtual Radiolarni Yuklash va AP hamda STA ni Ajratish

```bash
modprobe mac80211_hwsim radios=2
rfkill unblock wlan
iw dev
iw phy
ip -br link
```

**Har bir buyruqning batafsil tahlili:**
* `modprobe mac80211_hwsim radios=2`: Linux yadrosiga `mac80211_hwsim` virtual Wi-Fi drayverini yuklaydi. `radios=2` parametri tizimda roppa-rosa 2 ta virtual Wi-Fi radiosi (`phy0` va `phy1`) hosil qilinishini belgilaydi.
* `rfkill unblock wlan`: Simsiz tarmoq qurilmalarining apparat yoki dasturiy blokirovkasini to‘liq ochadi (bloklangan bo‘lsa radio ishlamaydi).
* `iw dev`: Tizimdagi barcha simsiz tarmoq interfeyslarini ko‘rsatadi (dastlab `phy0` da `wlan0`, `phy1` da `wlan1` paydo bo‘ladi).
* `iw phy`: Yaratilgan fizik radiolarning apparat xususiyatlarini (chastota diapazonlari, quvvati) chiqaradi.
* `ip -br link`: Barcha tarmoq interfeyslarining qisqacha holatini ko‘rsatadi.

Endi kirish nuqtasi (`ap0`) va mijoz (`sta0`) interfeyslarini ajratamiz hamda mijozni alohida `a8sta` network namespaciga o‘tkazamiz:

```bash
ip link set wlan0 down
ip link set wlan0 name ap0
ip link set wlan1 down
ip netns add a8sta
ip netns exec a8sta sleep infinity &
STA_PID=$!
sleep 1
ip netns identify "$STA_PID"
iw phy phy1 set netns "$STA_PID"
ip netns exec a8sta ip link set wlan1 name sta0
ip netns exec a8sta ip link set lo up
ip netns exec a8sta ip link set sta0 up
ip link set ap0 up
ip addr add 10.80.10.1/24 dev ap0
ip netns exec a8sta ip addr add 10.80.10.20/24 dev sta0
sysctl -w net.ipv4.ip_forward=0
```

**Har bir buyruqning batafsil tahlili:**
* `ip link set wlan0 down`: `wlan0` interfeysini vaqtincha o‘chiradi (nomini o‘zgartirish uchun interfeys DOWN holatida bo‘lishi shart).
* `ip link set wlan0 name ap0`: `wlan0` nomini qulay va tushunarli bo‘lishi uchun `ap0` (Access Point 0) ga o‘zgartiradi.
* `ip link set wlan1 down`: `wlan1` interfeysini o‘chiradi.
* `ip netns add a8sta`: Tizimda `a8sta` nomli yangi tarmoq nomlar fazosini (network namespace) yaratadi.
* `ip netns exec a8sta sleep infinity &`: `a8sta` namespaci ichida fon rejimida abadiy uxlovchi jarayonni boshlaydi. Bu orqali ushbu namespace uchun barqaror Process ID (PID) vujudga keladi.
* `STA_PID=$!`: Oxirgi ishga tushgan fon jarayonining raqamini (PID) `STA_PID` o‘zgaruvchisiga o‘zlashtiradi.
* `sleep 1`: Jarayon to‘liq ishga tushishi uchun 1 soniya kutadi.
* `ip netns identify "$STA_PID"`: Ushbu PID haqiqatdan ham `a8sta` namespaciga tegishli ekanligini tasdiqlaydi.
* `iw phy phy1 set netns "$STA_PID"`: **Eng muhim buyruq!** `phy1` fizik radiosini asosiy tizimdan sug‘urib, `STA_PID` jarayoni joylashgan `a8sta` namespaciga o‘tkazadi. Endi `phy1` va uning interfeyslari asosiy tizimga ko‘rinmaydi!
* `ip netns exec a8sta ip link set wlan1 name sta0`: `a8sta` namespaci ichida `wlan1` nomini `sta0` (Station 0) ga o‘zgartiradi.
* `ip netns exec a8sta ip link set lo up`: `a8sta` ichidagi lokal halqa interfeysini (`lo` — loopback) faollashtiradi.
* `ip netns exec a8sta ip link set sta0 up`: `a8sta` ichidagi `sta0` Wi-Fi interfeysini ishchi (UP) holatga keltiradi.
* `ip link set ap0 up`: Asosiy tizimdagi `ap0` interfeysini ishchi (UP) holatga keltiradi.
* `ip addr add 10.80.10.1/24 dev ap0`: Kirish nuqtasiga (`ap0`) `10.80.10.1` statik IP manzilini va `/24` (255.255.255.0) tarmoq niqobini beradi.
* `ip netns exec a8sta ip addr add 10.80.10.20/24 dev sta0`: Mijozga (`sta0`) `a8sta` namespaci ichida `10.80.10.20` statik IP manzilini biriktiradi.
* `sysctl -w net.ipv4.ip_forward=0`: Yadro darajasida IP marshrutlashni (packet forwarding) butunlay o‘chiradi. Bu paketlar tasodifan Linux yadrosining ichki marshruti orqali o‘tib ketishining oldini oladi.

Tekshiruv buyruqlari:
```bash
ip -br addr show ap0
ip netns exec a8sta ip -br addr
ip netns exec a8sta ip route
ip netns exec a8sta iw dev sta0 link
```

**Buyruqlarning vazifalari:**
* `ip -br addr show ap0`: Asosiy muhitda faqat `ap0` va uning `10.80.10.1/24` IP si borligini ko‘rsatadi.
* `ip netns exec a8sta ip -br addr`: `a8sta` ichida faqat `sta0` va uning `10.80.10.20/24` IP si borligini ko‘rsatadi.
* `ip netns exec a8sta ip route`: `a8sta` ichida hech qanday tashqi standart shlyuz (default gateway) yo‘qligini, faqat `10.80.10.0/24 dev sta0` marshruti borligini tasdiqlaydi.
* `ip netns exec a8sta iw dev sta0 link`: Mijozning holatini ko‘rsatadi (hali AP ishga tushmagani uchun `Not connected` chiqishi normal holat).

---

### 3.4. 3-bosqich: Boshlang‘ich WPA2-Personal Ulanishini Sinash (T1 Testi)

WPA2-Personal va o‘chirilgan PMF holatini tekshiramiz.

```bash
umask 077
cat > ap-wpa2.conf <<'EOF'
interface=ap0
driver=nl80211
ctrl_interface=/run/a8/hostapd
ssid=A8-LAB-01
hw_mode=g
channel=1
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_passphrase=A8-Only-Lab-2026!
rsn_pairwise=CCMP
ieee80211w=0
wps_state=0
EOF

cat > sta-wpa2.conf <<'EOF'
ctrl_interface=/run/a8/wpa
update_config=0
network={
    ssid="A8-LAB-01"
    proto=RSN
    key_mgmt=WPA-PSK
    psk="A8-Only-Lab-2026!"
    pairwise=CCMP
    group=CCMP
    ieee80211w=0
    scan_freq=2412
}
EOF
```

**Konfiguratsiya parametrlarining tahlili:**
* `umask 077`: Yangi yaratiladigan barcha fayllarga faqat egasi uchun ruxsat beradi (`rw-------` / `600`), boshqalar o‘qiy olmaydi.
* `interface=ap0`: Kirish nuqtasi qaysi interfeysda ochilishini bildiradi.
* `driver=nl80211`: Linuxning standart simsiz drayver steki.
* `ctrl_interface=/run/a8/hostapd`: Boshqaruv soketi manzili (`hostapd_cli` muloqot qilishi uchun).
* `ssid=A8-LAB-01`: Wi-Fi tarmog‘ining nomi.
* `hw_mode=g`: 2.4 GHz diapazonida (802.11g) ishlash rejimi.
* `channel=1`: 1-kanal (markaziy chastotasi 2412 MHz).
* `auth_algs=1`: Open System Authentication (802.11 darajasida ochiq tizim autentifikatsiyasi).
* `wpa=2`: WPA2 (RSN) yoqilgan.
* `wpa_key_mgmt=WPA-PSK`: Kalit boshqaruvi uchun Pre-Shared Key (parol) ishlatiladi.
* `wpa_passphrase=A8-Only-Lab-2026!`: Wi-Fi paroli.
* `rsn_pairwise=CCMP`: Shifrlash uchun AES-CCMP algoritmi.
* `ieee80211w=0`: Protected Management Frames (PMF) butunlay o‘chirilgan (Disabled).
* `wps_state=0`: WPS zaifliklarini oldini olish uchun butunlay o‘chirilgan.
* `scan_freq=2412`: Mijoz faqat 2412 MHz (1-kanal) ni skanerlaydi, bu ulanish tezligini oshiradi.

Dasturlarni ishga tushiramiz:

```bash
hostapd -B -P /run/a8/ap.pid -f /root/A8/T1-ap.log ap-wpa2.conf
ip netns exec a8sta wpa_supplicant -B -D nl80211 -i sta0 \
    -c /root/A8/sta-wpa2.conf -P /run/a8/sta.pid \
    -f /root/A8/T1-sta.log
```

**Buyruqlarning vazifalari:**
* `hostapd -B`: `hostapd` ni orqa fonda (daemon) ishga tushirish.
* `-P /run/a8/ap.pid`: Jarayon raqamini (PID) faylga yozish (keyinchalik to‘xtatish uchun).
* `-f /root/A8/T1-ap.log`: Barcha xatolik va loglarni faylga yo‘naltirish.
* `ip netns exec a8sta wpa_supplicant ...`: `wpa_supplicant` ni `a8sta` namespaci ichida orqa fonda ishga tushirish.
* `-D nl80211`: Drayver turi.
* `-i sta0`: Ulanuvchi interfeys.
* `-c /root/A8/sta-wpa2.conf`: Mijoz konfiguratsiya fayli.
* `-f /root/A8/T1-sta.log`: Mijoz log fayli.

5–10 soniya kutib, T1 natijalarini olamiz:

```bash
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 status \
    | tee report/T1-status.txt
ip netns exec a8sta ping -c 3 -W 1 10.80.10.1 \
    | tee report/T1-ping.txt
ip netns exec a8sta iw dev sta0 link
```

**Kutilgan natija (T1):**
* `T1-status.txt`: `wpa_state=COMPLETED`, `key_mgmt=WPA-PSK`, `pairwise_cipher=CCMP`.
* `T1-ping.txt`: 3 ta ICMP paketi muvaffaqiyatli uzatiladi (`0% packet loss`).
* Bu WPA2-Personal ning PMF siz holatda to‘g‘ri ishlaganini isbotlaydi.

---

### 3.5. 4-bosqich: WPA3-Personal (SAE) va Majburiy PMF ga O‘tish (T2 Testi)

Endi tizimni to‘liq WPA3-Personal, majburiy PMF (`ieee80211w=2`) va SAE ga o‘tkazamiz.

Eski jarayonlarni to‘xtatamiz:
```bash
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 terminate
hostapd_cli -p /run/a8/hostapd -i ap0 terminate
sleep 1
```

**Buyruqlarning vazifalari:**
* `wpa_cli ... terminate`: Ishlayotgan mijoz supplicant jarayonini toza to‘xtatadi.
* `hostapd_cli ... terminate`: Ishlayotgan AP hostapd jarayonini toza to‘xtatadi.
* `sleep 1`: Soketlar va interfeyslar bo‘shashi uchun 1 soniya kutadi.

Konfiguratsiyalarni WPA3 ga yangilaymiz:
```bash
sed -e 's/wpa_key_mgmt=WPA-PSK/wpa_key_mgmt=SAE/' \
    -e 's/ieee80211w=0/ieee80211w=2/' \
    ap-wpa2.conf > ap-wpa3.conf
printf '\nsae_pwe=2\n' >> ap-wpa3.conf

sed -e 's/key_mgmt=WPA-PSK/key_mgmt=SAE/' \
    -e 's/ieee80211w=0/ieee80211w=2/' \
    sta-wpa2.conf > sta-wpa3.conf
sed -i '1i sae_pwe=2' sta-wpa3.conf

chmod 600 *.conf
```

**Parametrlarning tahlili:**
* `wpa_key_mgmt=SAE`: Kalit boshqaruvi PSK dan SAE (Simultaneous Authentication of Equals) ga o‘zgartirildi.
* `ieee80211w=2`: PMF majburiy qilindi (Required). PMF bo‘lmagan mijozlar ulanolmaydi.
* `sae_pwe=2`: PWE generatsiyasini belgilaydi.
* `chmod 600 *.conf`: Yangi fayllarga ham xavfsiz huquqlar berildi.

Dasturlarni ishga tushiramiz:
```bash
hostapd -B -P /run/a8/ap.pid -f /root/A8/T2-ap.log ap-wpa3.conf
ip netns exec a8sta wpa_supplicant -B -D nl80211 -i sta0 \
    -c /root/A8/sta-wpa3.conf -P /run/a8/sta.pid \
    -f /root/A8/T2-sta.log
```

5–10 soniya kutib, T2 dalillarini yig‘amiz:
```bash
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 status \
    | tee report/T2-status.txt
ip netns exec a8sta ping -c 3 -W 1 10.80.10.1 \
    | tee report/T2-ping.txt
hostapd_cli -p /run/a8/hostapd -i ap0 all_sta \
    | tee report/T2-ap-station.txt
ip netns exec a8sta iw dev sta0 scan > report/T2-scan.txt
```

**Buyruqlarning vazifalari va natijalar tahlili:**
* `wpa_cli status`: Mijoz holati. Unda `wpa_state=COMPLETED`, `key_mgmt=SAE`, `pmf=2` yoki `pmf=1` ko‘rinishi kerak.
* `ping`: 10.80.10.1 ga aloqa mavjudligini tasdiqlaydi.
* `hostapd_cli all_sta`: AP ga ulangan mijozlar ro‘yxati. Unda mijozning MAC manzili yonida `[AUTHORIZED]` va eng asosiysi **`[MFP]`** (Management Frame Protection) bayrog‘i bo‘lishi shart! Bu PMF ikki tomonlama kelishilganini isbotlaydi.
* `iw dev sta0 scan`: Efirni skanerlaydi. `A8-LAB-01` blokida:
  * `Authentication suites: SAE`
  * `Management Frame Protection: Required` (yoki `Capable` va `Required` bitlari) e’lon qilingan bo‘lishi shart.

---

### 3.6. 5-bosqich: Salbiy Testlar — Parol va Profil Mosligi (T3 va T4 Testlari)

Salbiy testlar xavfsizlik siyosati chetlanishlarni to‘g‘ri bloklayotganini isbotlaydi.

#### T3 Testi: WPA3, lekin noto‘g‘ri parol
```bash
sed 's/psk=".*"/psk="A8-Wrong-Password!"/' \
    sta-wpa3.conf > sta-bad.conf
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 terminate
sleep 1
ip netns exec a8sta wpa_supplicant -B -D nl80211 -i sta0 \
    -c /root/A8/sta-bad.conf -f /root/A8/T3-sta.log

# 15 soniya kuzatgach:
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 status \
    | tee report/T3-status.txt
ip netns exec a8sta ping -c 3 -W 1 10.80.10.1 \
    > report/T3-ping.txt 2>&1
tail -n 25 T3-sta.log
```

**T3 Tahlili va Kutilgan Natija:**
* `wpa_state` hech qachon `COMPLETED` bo‘lmaydi (`DISCONNECTED` yoki `SCANNING` holatida aylanadi).
* `ping` ishlamaydi (`Destination Host Unreachable` yoki `100% packet loss`).
* `T3-sta.log`: SAE commit/confirm bosqichida rad etish (`SAE: authentication failed` yoki `status code 1`) qayd etiladi.
* Bu SAE noto‘g‘ri parol bilan tarmoqqa kirishni to‘xtatganini isbotlaydi.

#### T4 Testi: To‘g‘ri parolli, lekin faqat WPA2-PSK mijoz (AP esa faqat SAE va PMF Required)
```bash
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 terminate
sleep 1
ip netns exec a8sta wpa_supplicant -B -D nl80211 -i sta0 \
    -c /root/A8/sta-wpa2.conf -f /root/A8/T4-sta.log

# 15 soniya kuzatgach:
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 status \
    | tee report/T4-status.txt
ip netns exec a8sta ping -c 3 -W 1 10.80.10.1 \
    > report/T4-ping.txt 2>&1
tail -n 25 T4-sta.log
```

**T4 Tahlili va Kutilgan Natija:**
* Mijoz paroli to‘g‘ri bo‘lsa ham, ulanish amalga oshmaydi.
* Sabab: AP faqat SAE (AKM) va majburiy PMF (`ieee80211w=2`) talab qiladi. Mijoz esa faqat WPA-PSK va `ieee80211w=0` ni taklif qilmoqda.
* Xavfsizlik profili mos kelmagani sababli ulanish rad etiladi.

---

### 3.7. 6-bosqich: Tizimni Qayta Tiklash (T5 Testi)

T3 va T4 dagi xatolar AP ning ishdan chiqqanidan emas, aynan xavfsizlik to‘sig‘idan ekanligini isbotlash uchun to‘g‘ri WPA3 profilini qayta ulaymiz:

```bash
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 terminate
sleep 1
ip netns exec a8sta wpa_supplicant -B -D nl80211 -i sta0 \
    -c /root/A8/sta-wpa3.conf -f /root/A8/T5-sta.log

# 5–10 soniyadan so‘ng
ip netns exec a8sta wpa_cli -p /run/a8/wpa -i sta0 status \
    | tee report/T5-status.txt
ip netns exec a8sta ping -c 3 -W 1 10.80.10.1 \
    | tee report/T5-ping.txt
```

**Kutilgan Natija (T5):**
* `wpa_state=COMPLETED`, `key_mgmt=SAE`.
* `ping` 100% muvaffaqiyatli ishlaydi.
* Bu AP butun vaqt davomida sog‘lom ishlaganini va salbiy testlar faqat xavfsizlik cheklovlari tufayli yuz berganini to‘liq isbotlaydi.

---

### 3.8. 7-bosqich (Chuqurlashtirilgan): Virtual Efir Trafigini Qamrab Olish

Agar vaqt bo‘lsa va chuqurroq o‘rganmoqchi bo‘lsangiz, `hwsim0` orqali virtual efir paketlarini tahlil qilishingiz mumkin.

Ubuntu Server VM da ikkinchi terminal oynasini oching (yoki tmux/screen orqali) va root sifatida bajaring:
```bash
ip link show hwsim0
ip link set hwsim0 up
tcpdump -i hwsim0 -s 0 -U -w /root/A8/report/A8-radio.pcap
```

Birinchi terminalda T5 ulanishini qayta ishga tushiring va ping yuboring, so‘ng `tcpdump` oynasida `Ctrl+C` bosing.

Paketlarni `tshark` yordamida tahlil qiling:
```bash
tshark -r /root/A8/report/A8-radio.pcap \
    -Y 'wlan.fc.type_subtype == 8' -V \
    > /root/A8/report/A8-beacon.txt
```

**Tahlil:**
* `wlan.fc.type_subtype == 8`: Bu filtr faqat **Beacon** kadrlarini ajratib oladi.
* Matn faylida RSN bo‘limidan `Auth Key Management (AKM) Suite: SAE`, `Management Frame Protection Capable: True` va `Required: True` bitlarini yaqqol ko‘rish mumkin.

---

### 3.9. 8-bosqich: Audit Hisobotini Tayyorlash va SHA-256 Yaxlitligi

Hisobotga maxfiy parollarni ochiq ko‘rinishda kiritish xavfsizlik qoidalariga zid. Shu sababli konfiguratsiyalardan parollarni filtrlab olamiz:

```bash
cd /root/A8
sed -E '/^[[:space:]]*(wpa_passphrase|wpa_psk|sae_password)=/d' \
    ap-wpa3.conf > report/A8-ap-policy.txt

sed -E '/^[[:space:]]*(psk|sae_password)=/d' \
    sta-wpa3.conf > report/A8-sta-policy.txt

{ 
    date -Is
    uname -r
    hostapd -v
    wpa_supplicant -v
    ip -br addr show ap0
    ip netns exec a8sta ip -br addr
} > report/A8-environment.txt 2>&1

stat -c '%a %n' *.conf > report/A8-permissions.txt
```

Fayllarning SHA-256 nazorat xesh summasini yaratamiz va tekshiramiz:
```bash
cd /root/A8/report
find . -maxdepth 1 -type f ! -name A8-SHA256SUMS.txt -print0 \
    | sort -z | xargs -0 sha256sum > A8-SHA256SUMS.txt
sha256sum -c A8-SHA256SUMS.txt
```

**Buyruqlarning vazifalari:**
* `sed -E ...`: Konfiguratsiyalardan parollarni o‘chirib, faqat xavfsizlik siyosati satrlarini `report` papkasiga saqlaydi.
* `stat -c '%a %n' *.conf`: Konfiguratsiya fayllarining ruxsat huquqlarini (600 ekanligini) audit uchun qayd etadi.
* `sha256sum`: Barcha hisobot fayllarining kriptografik nazorat summalarini generatsiya qiladi.
* `sha256sum -c`: Hisobot fayllarining yaxlitligini tekshiradi (barcha fayllar yonida `OK` chiqishi kerak).

---

### 3.10. 9-bosqich: Dalillarni Ubuntu Server VM dan Ubuntu Host Tizimiga Ko‘chirish

Sizning asosiy tizimingiz — **Ubuntu Linux**! Shuning uchun hisobot fayllarini Windowsga emas, o‘zingizning Ubuntu tizimingizga ko‘chirishning 2 ta eng qulay va toza usuli mavjud:

#### 1-Usul: VMware Shared Folders (Tavsiya etiladi)
1. VMware sozlamalarida Ubuntu hostdagi biron bir papkani (masalan, `~/Downloads/A8_report`) umumiy papka (Shared Folder) sifatida ulang.
2. Ubuntu Server VM ichida uni o‘qib oling:
   ```bash
   mkdir -p /mnt/hgfs
   vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other
   cp -r /root/A8/report /mnt/hgfs/A8_report/
   ```

#### 2-Usul: SSH / SCP orqali (Host-Only yoki NAT orqali)
1. VM da IP manzilni aniqlang (agar VM ga ikkinchi Host-Only adapter qo‘shilgan bo‘lsa yoki NAT orqali port uzatilgan bo‘lsa).
2. Ubuntu host terminalida turib buyruq bering:
   ```bash
   scp -r user@<VM_IP_MANZILI>:/root/A8/report ~/Downloads/
   ```

---

## 5. 4-BO‘LIM: Nosozliklarni Aniqlash va Tuzatish (Troubleshooting Guide)

| Muammo / Alomat | Kelib Chiqish Sababi | Aniq Tekshiruv va Tuzatish Usuli |
| :--- | :--- | :--- |
| **`modprobe: FATAL: Module mac80211_hwsim not found`** | Joriy yadro uchun `modules-extra` to‘plami o‘rnatilmagan. | `uname -r` ni ko‘ring va `sudo apt install -y linux-modules-extra-$(uname -r)` buyrug‘ini bering. |
| **`wlan0` yoki `phy1` nomlari ko‘rinmayapti** | Oldingi tajribadan interfeys nomlari o‘zgargan yoki interfeys boshqa nom bilan yaratilgan. | `iw dev` va `iw phy` buyrug‘i bilan mavjud nomlarni tekshiring. Agar kerak bo‘lsa, buyruqlardagi `wlan0/wlan1` nomlarini moslang. |
| **`Device or resource busy` (hostapd ishga tushmayapti)** | Interfeysni tizimdagi boshqa xizmat (masalan, `NetworkManager` yoki eski `hostapd`) egallab turibdi. | `systemctl stop NetworkManager hostapd wpa_supplicant` qiling. `ps aux \| grep hostapd` orqali eski jarayonlarni topib, `kill -9` qiling. |
| **`iw phy phy1 set netns` xatolik bermoqda** | `STA_PID` jarayoni to‘xtab qolgan yoki PID noto‘g‘ri olingan. | `ps -p "$STA_PID"` orqali jarayon tirikligini tekshiring. Qaytadan `ip netns exec a8sta sleep infinity &` qiling. |
| **SSID topilmayapti (`scan` da ko‘rinmaydi)** | AP ishga tushmagan, kanal chastotasi mos emas yoki interfeys DOWN holatida. | `cat /root/A8/T2-ap.log` ni o‘qing. `AP-ENABLED` yozuvi borligiga ishonch hosil qiling. Kanal 1 (2412 MHz) ekanini tekshiring. |
| **`wpa_state=COMPLETED` bor, lekin `ping` ishlamayapti** | 1. IP manzillar xato qo‘yilgan.<br>2. `sta0` namespace ichiga ko‘chmagan.<br>3. Lokal xavfsizlik devori (iptables/ufw) ICMP ni bloklamoqda. | `ip -br addr show ap0` va `ip netns exec a8sta ip -br addr` qilib, ikkala IP ham `/24` ekanini ko‘ring. `sudo ufw disable` qilib firewallni tekshiring. |
| **T3 yoki T4 da ham `ping` ishlab ketmoqda** | Eski muvaffaqiyatli `wpa_supplicant` jarayoni to‘xtatilmagan va tizim avvalgi ulanishdan foydalanmoqda. | Har bir testdan oldin `wpa_cli terminate` buyrug‘ini berganingizni va `ps aux \| grep wpa` toza ekanini tekshiring. |
| **`iw dev sta0 scan: Device or resource busy`** | Mijoz ayni paytda ulanish jarayonida yoki fon skanerlashini amalga oshirmoqda. | 2–3 soniya kutib, `iw dev sta0 scan` buyrug‘ini qayta bering. |

---

## 6. 5-BO‘LIM: Natijalarni Qayd Etish Jadvallari va Hisobot Namunasi

### 5.1. 3.1-jadval: Muhit va Himoya Sozlamalari

| Ko‘rsatkich | Kutilgan Qiymat | Amaldagi Qiymat / Dalil ID |
| :--- | :--- | :--- |
| **Yadro va paketlar** | `mac80211_hwsim` drayveri mavjud; `hostapd` >= 2.10, `wpa_supplicant` >= 2.10 | Mavjud (`6.8.0-xx-generic`), `hostapd v2.10`, `wpa_supplicant v2.10` / `A8-environment.txt` |
| **Izolyatsiya** | Tashqi NAT/Wi-Fi adapter uzilgan; STA alohida `a8sta` namespaci ichida | Virtual kabel uzilgan; STA faqat `a8sta` ichida / `A8-environment.txt` |
| **Radio va IP manzillar** | `ap0`: `10.80.10.1/24`; `sta0`: `10.80.10.20/24`; Shlyuz (Gateway) yo‘q | `ap0`=10.80.10.1, `sta0`=10.80.10.20, default route yo‘q / `A8-environment.txt` |
| **AP AKM va Shifrlash** | AKM: SAE; Ma’lumotlar shifri: CCMP (AES-128) | `wpa_key_mgmt=SAE`, `rsn_pairwise=CCMP` / `A8-ap-policy.txt` |
| **PMF Siyosati** | AP va STA uchun `ieee80211w=2` (Majburiy / Required) | `ieee80211w=2` har ikki tomonda mavjud / `A8-ap-policy.txt`, `A8-sta-policy.txt` |
| **Kelishilgan PMF** | AP `all_sta` natijasida mijozda `[MFP]` bayrog‘i | `all_sta` ro‘yxatida `[AUTHORIZED] [MFP]` tasdiqlangan / `T2-ap-station.txt` |
| **WPS va Fayl Huquqlari** | `wps_state=0` (O‘chiq); Konfiguratsiyalar: `600` | WPS o‘chirilgan; Ruxsatlar `600` / `A8-permissions.txt` |

---

### 5.2. 3.2-jadval: Ulanish Tajribalari (T1–T5)

| Test ID | Tavsif | Kutilgan Natija | Amaldagi Holat (`wpa_state`) | Ping Natijasi | Dalil Fayli |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **T1** | WPA2-Personal (PSK), PMF=0 | Ulanish tugaydi, ping ishlaydi | `COMPLETED` (`key_mgmt=WPA-PSK`) | 3/3 paket o‘tdi (0% loss) | `T1-status.txt`, `T1-ping.txt` |
| **T2** | WPA3-Personal (SAE), PMF=2 | Ulanish tugaydi, SAE va MFP faol, ping ishlaydi | `COMPLETED` (`key_mgmt=SAE`) | 3/3 paket o‘tdi (0% loss) | `T2-status.txt`, `T2-ping.txt`, `T2-ap-station.txt` |
| **T3** | WPA3, noto‘g‘ri parol | Ulanish rad etiladi, ping yo‘q | `DISCONNECTED` / `SCANNING` | Ping yo‘q (100% loss) | `T3-status.txt`, `T3-ping.txt`, `T3-sta.log` |
| **T4** | Faqat WPA2 mijoz, AP esa faqat WPA3 | Ulanish rad etiladi, ping yo‘q | `DISCONNECTED` / `SCANNING` | Ping yo‘q (100% loss) | `T4-status.txt`, `T4-ping.txt`, `T4-sta.log` |
| **T5** | To‘g‘ri WPA3 profilini tiklash | Ulanish qayta tiklanadi, ping ishlaydi | `COMPLETED` (`key_mgmt=SAE`) | 3/3 paket o‘tdi (0% loss) | `T5-status.txt`, `T5-ping.txt` |

---

### 5.3. 3.3-jadval: Dalil va Xulosa O‘rtasidagi Bog‘lanish

| Dalil | Nimani Ko‘rsatdi va Isbotladi? | Nimani Isbotlamaydi? |
| :--- | :--- | :--- |
| **Scan / Beacon RSN IE** | AP efirda SAE va Majburiy PMF (`MFP Required`) siyosatini to‘g‘ri e’lon qilayotganini ko‘rsatadi. | Mijoz ushbu talablarga rozi bo‘lib, ulanishni muvaffaqiyatli yakunlaganini isbotlamaydi. |
| **Mijoz `status: key_mgmt=SAE`** | Mijoz tomonida autentifikatsiya SAE algoritmi yordamida muvaffaqiyatli yakunlanganini isbotlaydi. | IP sathi (L3) aloqasi borligini yoki paketlar yo‘qolmasdan o‘tayotganini isbotlamaydi (ping kerak). |
| **AP `all_sta: [MFP]` bayrog‘i** | Kirish nuqtasi va mijoz o‘rtasida boshqaruv kadrlari himoyasi (PMF) o‘zaro kelishilganini isbotlaydi. | Tarmoq radio to‘sish (Jamming) yoki fizik shovqinlardan himoyalanganini isbotlamaydi. |
| **T3 / T4 Muvaffaqiyatsizligi** | Noto‘g‘ri parol (T3) va mos kelmaydigan xavfsizlik profili (T4) tarmoqqa kira olmasligini ko‘rsatadi. | T4 natijasi rad etilish aynan faqat PMF tufayli bo‘lganini isbotlamaydi (chunki AKM ham PSK vs SAE farqlanadi). |
| **T5 Qayta Ulanishi** | Kirish nuqtasi va tarmoq infratuzilmasi butun test davomida to‘liq sog‘lom ishlaganini isbotlaydi. | Tarmoqning real fizik Wi-Fi chipidagi qamrovi yoki antennasini isbotlamaydi (bu virtual simulyatsiya). |

---

### 5.4. 3.4-bo‘lim: Yakuniy Professional Xulosa (Hisobot uchun 8 gap)
> Ushbu amaliy mashg‘ulotda Linux yadrosining `mac80211_hwsim` drayveri yordamida izolyatsiyalangan virtual Wi-Fi laboratoriyasi muvaffaqiyatli yaratildi va boshlang‘ich WPA2-Personal (PSK) holatidan to‘liq WPA3-Personal (SAE) hamda majburiy PMF (`ieee80211w=2`) siyosatiga o‘tildi. Dastlabki T1 testida WPA2-PSK ulanishi o‘rnatilib, PMF o‘chirilgan holat boshlang‘ich xavfsizlik kamchiligi sifatida qayd etildi. T2 bosqichida kirish nuqtasi va mijoz faqat SAE AKM va CCMP shifrlashiga o‘tkazildi; efirdagi Beacon RSN axborot elementlarida `MFP-Required` e’lon qilingani hamda AP ning `all_sta` ma’lumotlarida `[MFP]` bayrog‘i kelishilgani to‘liq tasdiqlandi. Salbiy T3 testida noto‘g‘ri parol kiritilganda Dragonfly handshakening rad etilishi kuzatildi va offline parolni taxmin qilish imkoniyati yo‘qligi amalda ko‘rsatildi. Salbiy T4 testida faqat WPA2 ni qo‘llovchi mijozning WPA3-only va majburiy PMF ga ega AP ga ulana olmasligi xavfsizlik profillari nomuvofiqligi sifatida isbotlandi. T5 nazorat bosqichida to‘g‘ri WPA3 profili qayta ulanib, L3 darajasida ping aloqasi 100% tiklandi va oldingi xatolar AP ning buzilishidan emas, xavfsizlik filtrlaridan ekanligi oydinlashdi. Shuni ta’kidlash lozimki, PMF deautentifikatsiya kadrlarini soxtalashtirishdan himoya qilsa-da, u radiochastotani to‘sish (Jamming) yoki yuqori sathdagi fishing hujumlarining oldini ololmaydi. Umumiy xulosa shuki, WPA3-SAE va majburiy PMF kombinatsiyasi simsiz tarmoqning havoladagi kriptografik barqarorligini tubdan oshiradi.

---

## 7. 6-BO‘LIM: Himoya Savollari va Ularning Mukammal Javoblari

### 1-Savol: Host Wi-Fi orqali Internetga chiqsa, nega VMdagi Ethernet interfeys bilan `hostapd` AP yaratib bo‘lmaydi?
**Javob:** `hostapd` kirish nuqtasi dasturi faqat simsiz **IEEE 802.11 (nl80211 / mac80211)** rejimida ishlaydigan radio interfeyslarni boshqara oladi. VMware Workstation mehmon tizimga hostning Wi-Fi kartasini to‘g‘ridan-to‘g‘ri bermaydi, balki uni emulyatsiya qilingan simli **Ethernet (IEEE 802.3)** interfeysi (`e1000` yoki `vmxnet3`) sifatida uzatadi. Ethernet kartasida radiochastotalar, 802.11 kadr tuzilmasi, Beacon tarqatish yoki simsiz assotsiatsiya tushunchalari mavjud emas. Shu sababli `hostapd` Ethernet interfeysida ishga tusha olmaydi.

### 2-Savol: `mac80211_hwsim` qanday vazifani bajaradi va nimani modellashtirmaydi?
**Javob:** `mac80211_hwsim` — Linux yadrosining drayveri bo‘lib, dasturiy ravishda IEEE 802.11 virtual radiolarini yaratadi va ular o‘rtasida xotiradagi virtual efir orqali 802.11 kadrlarini uzatishni to‘liq simulyatsiya qiladi.  
*U nimani modellashtirmaydi:* U real fizik muhitdagi radiochastota (RF) to‘lqinlarining so‘nishi, masofaga bog‘liq signal kuchi (RSSI), devorlar va to‘siqlardan akslanish, tashqi elektromagnit shovqinlar, antennalar yo‘nalishi hamda fizik chip darajasidagi apparat nosozliklarini modellashtirmaydi.

### 3-Savol: STA radiosini alohida namespacega ko‘chirish ping tajribasiga nima uchun zarur?
**Javob:** Agar AP (`ap0`) va STA (`sta0`) bitta umumiy tarmoq muhitida (default namespace) bo‘lsa, Linux yadrosi marshrutlash jadvaliga qarab, ikkala IP ham o‘ziga tegishli ekanligini ko‘radi. Natijada `ping 10.80.10.1` yuborilganda, paketlar Wi-Fi efiriga (`mac80211_hwsim`) chiqmasdan, to‘g‘ridan-to‘g‘ri yadro xotirasidagi Loopback marshruti orqali aylanib o‘tadi (bypass qiladi). STA ni `a8sta` namespaciga ko‘chirish tarmoq steki va marshrutlashni to‘liq ajratadi va paketlarni faqat Wi-Fi radio ulanishi orqali o‘tishga majbur qiladi.

### 4-Savol: `hostapd` konfiguratsiyasidagi `wpa=2` bilan `wpa_key_mgmt=SAE` o‘rtasida ziddiyat bormi?
**Javob:** Hech qanday ziddiyat yo‘q. `hostapd` sintaksisida `wpa=1` WPA (v1) ni, `wpa=2` esa **RSN (Robust Security Network / IEEE 802.11i)** oilasini bildiradi. WPA3 standarti ham aynan RSN arxitekturasi ustiga qurilgan. Tarmoqning aynan WPA3-Personal ekanligini `wpa=2` emas, balki `wpa_key_mgmt=SAE` (Autentifikatsiya va kalit boshqaruvi) hamda majburiy PMF parametrlari belgilab beradi.

### 5-Savol: SAE, CCMP va PMF ning vazifalari qanday farqlanadi?
**Javob:** 
* **SAE (Simultaneous Authentication of Equals)**: Autentifikatsiya va kalit kelishish bosqichi uchun javobgar (Dragonfly handshakeda parol asosida tomonlarni tasdiqlaydi va umumiy maxfiy master kalit — PMK ni hosil qiladi).
* **CCMP (Counter Mode with CBC-MAC)**: Ma’lumotlar (Data) kadrlarining maxfiyligi va yaxlitligini ta’minlaydi (foydalanuvchi trafigini AES-128 bilan shifrlaydi).
* **PMF (Protected Management Frames)**: Boshqaruv (Management) kadrlarini (Deauth, Disassoc) kriptografik himoyalaydi va soxta kadrlar orqali tarmoqdan uzib tashlashning oldini oladi.

### 6-Savol: PMF capable va PMF required nimani bildiradi? E’lon qilingan siyosat bilan kelishilgan holat qanday ajratiladi?
**Javob:**
* **PMF Capable (Bit 6)**: Qurilma PMF ni qo‘llab-quvvatlay oladi, lekin uni majburiy talab qilmaydi (ixtiyoriy).
* **PMF Required (Bit 7)**: Qurilma PMF ni qat’iy talab qiladi; agar qarshi tomon PMF ni qo‘llamasa, ulanish rad etiladi.
* **E’lon qilingan siyosat** AP ning Beacon kadrlaridagi RSN IE da bo‘ladi (`iw scan` da ko‘rinadi). Bu faqat AP ning xohish-istagini bildiradi.
* **Kelishilgan holat** esa ulanish muvaffaqiyatli tugagach, AP dagi `all_sta` natijasida mijoz qarshisida `[MFP]` bayrog‘i chiqishi orqali tasdiqlanadi.

### 7-Savol: Nima uchun T4 natijasini faqat PMF ishlaganining mustaqil isboti deb bo‘lmaydi?
**Javob:** Chunki T4 testida mijoz konfiguratsiyasida birdaniga **ikkita** muhim parametr AP dan farq qiladi:
1. AKM (Mijozda `WPA-PSK`, AP da esa `SAE`).
2. PMF (Mijozda `ieee80211w=0`, AP da esa `ieee80211w=2`).  
Ulanish rad etilganda, rad etish aynan PMF yo‘qligi sabablimi yoki AKM (SAE o‘rniga PSK taklif qilingani) sabablimi — buni ajratib bo‘lmaydi. Faqat PMF ni tekshirish uchun AKM bir xil bo‘lib, faqat PMF parametrlari to‘qnashishi kerak edi.

### 8-Savol: T3 dan so‘ng T5 ni muvaffaqiyatli qaytarish nima uchun kerak?
**Javob:** T3 (noto‘g‘ri parol) va T4 (mos kelmaydigan profil) salbiy testlarida ping ishlamay qoladi. Agar T5 nazorat testi o‘tkazilmasa, o‘qituvchi yoki auditor: *"Balki AP jarayoni T2 dan keyin to‘xtab qolgandir yoki tarmoq interfeysi buzilgandir, shuning uchun ping yo‘qolgandir"*, deb gumon qilishi mumkin. T5 da to‘g‘ri profil bilan ping aloqasi yana 100% tiklanishi — avvalgi salbiy natijalar aynan xavfsizlik to‘siqlari tufayli yuz berganini ilmiy jihatdan isbotlaydi.

### 9-Savol: SSID ni yashirish yoki MAC filtrini qo‘shish WPA3 o‘rnini bosa oladimi?
**Javob:** Mutlaqo bosa olmaydi.
* **SSID ni yashirish (Hidden SSID)**: Faqat Beacon kadrida SSID maydoni bo‘sh qoldiriladi. Mijoz tarmoqqa ulanayotganda `Probe Request` yuboradi va unda tarmoq nomi ochiq ko‘rinadi. Har qanday erkin skaner (masalan, Kismet yoki Wireshark) yashirin SSID ni bir necha soniyada aniqlaydi.
* **MAC filtri**: MAC manzillar efirda ochiq, shifrlanmagan holda uzatiladi. Hujumchi qonuniy mijozning MAC manzilini efirdan tutib olib, o‘zining tarmoq kartasiga osongina yozib oladi (MAC Spoofing).
* Bular "Security through obscurity" (mavhumlik orqali xavfsizlik) bo‘lib, hech qanday kriptografik kuchga ega emas.

### 10-Savol: WPA3-Personal umumiy paroli tarqalib ketsa, foydalanuvchini alohida cheklashning qanday muammosi bor?
**Javob:** WPA3-Personal da barcha foydalanuvchilar bitta umumiy maxfiy so‘zdan (Pre-Shared Password) foydalanadi. Tizimda individual foydalanuvchi hisoblari (akkauntlar) mavjud emas. Agar bitta xodim ishdan bo‘shasa, uni tarmoqdan uzish uchun kirish nuqtasidagi Wi-Fi parolini o‘zgartirish kerak bo‘ladi. Bu esa tashkilotdagi qolgan barcha yuzlab xodimlarning qurilmalarida ham parolni qayta kiritish majburiyatini tug‘diradi.

### 11-Savol: PMF radiohalaqit, fishing yoki zararli veb-sahifadan himoya qiladimi?
**Javob:** Yo‘q, himoya qilmaydi.
* **Radiohalaqit (RF Jamming)**: Bu fizik sathdagi (L1) hujum. Hujumchi butun chastota polosasini kuchli shovqin bilan to‘ldirsa, qonuniy kadrlar qabul qilinmaydi. PMF fizik to‘lqinlarni to‘xtata olmaydi.
* **Fishing va zararli sahifalar**: Bu dasturiy va tarmoq sathidan yuqori (L7 / Application layer) xavflar. Wi-Fi shifrlash faqat mijoz va Wi-Fi router o‘rtasidagi radiohavolani himoyalaydi. Agar foydalanuvchi soxta veb-saytga kirsa yoki virus yuklab olsa, Wi-Fi xavfsizligi buni to‘xtata olmaydi.

### 12-Savol: Xesh ro‘yxati (`A8-SHA256SUMS.txt`) qanday holatni aniqlaydi va mualliflikni nega tasdiqlamaydi?
**Javob:**
* **Xesh ro‘yxati yaxlitlikni (Integrity) aniqlaydi**: Fayl yaratilgandan so‘ng uning biror bayti tasodifan yoki qasddan o‘zgartirilganmi-yo‘qligini darhol aniqlab beradi.
* **Nega mualliflikni tasdiqlamaydi**: SHA-256 simmetrik xesh funksiyasi bo‘lib, unda maxfiy shaxsiy kalit (Private Key) yoki raqamli imzo (Digital Signature) mavjud emas. Istalgan boshqa talaba fayllarni o‘zgartirib, yangi fayllar uchun qaytadan yangi `sha256sum` ro‘yxatini hisoblab qo‘yishi mumkin. Haqiqiy mualliflikni tasdiqlash uchun assimetrik kriptografik imzo (masalan, GPG/PGP kaliti) talab etiladi.

---

## 8. 7-BO‘LIM: Foydalanilgan va Tavsiya Etiladigan Manbalar

Ushbu amaliy qo‘llanma va undagi barcha texnik sozlamalar quyidagi xalqaro standartlar, Linux yadrosi hujjatlari va rasmiy ochiq kodli loyihalar asosida ishlab chiqildi:

1. **Linux Wireless rasmiy arxitekturasi: `mac80211_hwsim`**  
   Linux yadrosining dasturiy Wi-Fi modellashtirish drayveri bo‘yicha to‘liq qo‘llanma.  
   *Havola:* [https://wireless.docs.kernel.org/en/latest/en/users/drivers/mac80211_hwsim.html](https://wireless.docs.kernel.org/en/latest/en/users/drivers/mac80211_hwsim.html)

2. **`hostapd` Rasmiy Konfiguratsiya Qo‘llanmasi (w1.fi / AOSP)**  
   IEEE 802.11 AP, WPA2-PSK, WPA3-SAE, PMF parametrlarining to‘liq tavsifi.  
   *Havola:* [https://w1.fi/hostapd/](https://w1.fi/hostapd/)  
   *AOSP manbasi:* [https://android.googlesource.com/platform/external/wpa_supplicant_8/+/master/hostapd/hostapd.conf](https://android.googlesource.com/platform/external/wpa_supplicant_8/+/master/hostapd/hostapd.conf)

3. **`wpa_supplicant` Rasmiy Konfiguratsiya Namunasi**  
   Wi-Fi mijozi konfiguratsiyasi, SAE parametrlari, tarmoq bloklari va holat nazorati.  
   *AOSP manbasi:* [https://android.googlesource.com/platform/external/wpa_supplicant_8/+/master/wpa_supplicant/wpa_supplicant.conf](https://android.googlesource.com/platform/external/wpa_supplicant_8/+/master/wpa_supplicant/wpa_supplicant.conf)

4. **Linux Network Namespaces (`ip-netns`) Hujjatlari**  
   Linux yadrosida tarmoq steklarini izolyatsiya qilish va boshqarish bo‘yicha rasmiy man sahifalari.  
   *Havola:* [https://manpages.debian.org/unstable/iproute2/ip-netns.8.en.html](https://manpages.debian.org/unstable/iproute2/ip-netns.8.en.html)

5. **IETF RFC 7664: Dragonfly Key Exchange**  
   WPA3-SAE ning matematik asosi bo‘lgan "Simultaneous Authentication of Equals" protokoli tavsifi.  
   *Havola:* [https://datatracker.ietf.org/doc/html/rfc7664](https://datatracker.ietf.org/doc/html/rfc7664)

6. **IEEE 802.11w-2009 Standarti (Protected Management Frames)**  
   Simsiz tarmoqlarda boshqaruv kadrlarini kriptografik himoyalash spetsifikatsiyasi.  
   *Havola:* [https://standards.ieee.org/](https://standards.ieee.org/)

7. **Wi-Fi Alliance WPA3 Spetsifikatsiyalari va Xavfsizlik Tavsiyalari**  
   WPA3-Personal, WPA3-Enterprise, Protected Management Frames (PMF) texnik talablari.  
   *Havola:* [https://www.wi-fi.org/discover-wi-fi/security](https://www.wi-fi.org/discover-wi-fi/security)
