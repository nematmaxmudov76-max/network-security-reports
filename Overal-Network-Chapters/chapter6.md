# 6-BOB: SIMSIZ VA MOBIL TARMOQLAR (Wireless and Mobile Networks)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Simsiz aloqa kanallarining jismoniy xususiyatlari (signal so‘nishi, xalaqitlar, ko‘p nurlilik, SNR va BER bog‘liqligi), CDMA matematikasi, IEEE 802.11 Wi-Fi arxitekturasi va standartlari, CSMA/CA to‘qnashuvlarning oldini olish mexanizmi va RTS/CTS yechimi, 802.11 kadr tuzilishi (4 ta MAC manzil), 4G LTE uyali aloqa tarmog‘ining to‘liq IP yadrosi (EPC, eNodeB, MME, SGW, PGW) hamda mobil foydalanuvchilarni marshrutlash (Home Agent, Foreign Agent, Care-of-Address) tamoyillarini chuqur o‘rganish.

---

## 6.1. SIMSIZLIK VA MOBILLIKNING FUNDAMENTAL FARQI

Kundalik hayotda "Simsiz" (Wireless) va "Mobil" (Mobile) atamalari ko‘pincha aralashtirib yuboriladi. Biroq tarmoq muhandisligida ular ikkita mustaqil o‘lchovdir:

```
               MOBIL EMAS (Statsionar)                  MOBIL (Harakatdagi)
       +------------------------------------+------------------------------------+
       | Xodim ofisdagi stolida noutbukini  | Foydalanuvchi tezyurar poyezdda    |
SIMSIZ | Wi-Fi ga ulab ishlatmoqda.         | smartfonda video ko'rib ketmoqda.  |
       | (Simsiz, lekin joyidan jilmaydi)   | (Ham simsiz, ham to'liq mobil!)    |
       +------------------------------------+------------------------------------+
       | Kompyuter stolida turgan oddiy PC  | Foydalanuvchi noutbukini bir xona- |
SIMLI  | devordagi Ethernet rozetkasiga     | dagi Ethernet rozetkadan uzib,     |
       | kabel bilan ulangan.               | boshqa xonadagi rozetkaga ulaydi.  |
       | (Simli va statsionar)              | (Simli, lekin ko'chib yuradi)      |
       +------------------------------------+------------------------------------+
```

1. **Simsizlik (Wireless):** Aloqa liniyasi sifatida kabel o‘rniga **elektromagnit (radio) to‘lqinlar** ishlatiladi.
2. **Mobillik (Mobility):** Foydalanuvchi tarmoqqa ulangan holda bir geografik hududdan ikkinchisiga ko‘chib yuradi va tarmoq ulanish nuqtalarini (Access Point yoki uyali aloqa antennasini) **ochilgan sessiyalarni (TCP) uzib qo‘ymasdan** almashtiradi.

---

## 6.2. SIMSIZ ALOQA KANALLARINING JISMONIY XUSUSIYATLARI

Simsiz radio kanali simli kabel (mis yoki optika) bilan solishtirganda ancha injiq, noaniq va dushmanona muhit hisoblanadi:

```
                                  (Bino / Devor)
                                  +------------+
                                  |            |
[ Jo'natuvchi ] ~ ~ ~ ~ ~ ~ ~ ~ ~ |   To'siq   | ~ ~ ~ ~ ~ ~ ~ ~ > [ Qabul qiluvchi ]
      |                           |            |                     ^
      |                           +------------+                    /
      \                                                            /
       \====== (Ko'p nurlilik: Yerdan va binodan aks etish) ======/
```

### 1. Signal Kuchining So‘nishi (Path Loss / Attenuation)
Elektromagnit to‘lqin antennadan uzoqlashgan sari uning quvvati masofa kvadratiga ($1/d^2$) teskari proporsional ravishda pasayadi. Bundan tashqari, to‘lqin yo‘lidagi jismoniy to‘siqlar (beton devorlar, shisha, metall to‘rlar) signalni yutadi.

### 2. Boshqa Manbalar Xalaqiti (Interference)
Ommabop **2.4 GHz ISM (Industrial, Scientific, Medical)** chastota diapazonida bir vaqtning o‘zida:
* Wi-Fi routerlar,
* Bluetooth quloqchinlar va soatlar,
* Mikroto‘lqinli pechlar,
* Simsiz sichqoncha va klaviaturalar ishlaydi.
Ular bitta umumiy chastotada bir-birining signallariga kuchli shovqin soladi. (Shu sababli zamonaviy Wi-Fi 5 GHz va 6 GHz diapazonlariga o‘tgan).

### 3. Ko‘p Nurlilik (Multipath Propagation)
Radio to‘lqin manbadan to‘g‘ri chiziq bo‘yicha chiqqani kabi, yo‘ldagi yer, devorlar va mebellardan aks etib, qabul qiluvchi antennaga **bir necha millisekund farq bilan turli yo‘llardan** yetib keladi. Natijada signallar fazalari bir-birini so‘ndiradi (Fading hodisasi).

---

### 6.2.1. Signal-Shovqin Nisbati (SNR) va Dinamik Tezlik Moslashuvi

* **SNR (Signal-to-Noise Ratio):** Foydali signal quvvatining tarmoqdagi shovqin quvvatiga nisbati (detsibellarda — dB o‘lchanadi).
* **BER (Bit Error Rate):** Uzatilgan bitlarning qancha qismi xato bo‘lib yetib borishi ehtimoli.
* **SNR qancha yuqori bo‘lsa, BER shuncha past (yaxshi) bo‘ladi.**

```
BER (Xatolik darajasi)
  ^
  |      \    \    \  
  |       \    \    \   BPSK (Oddiy modulyatsiya: Past tezlik, lekin shovqinga o'ta chidamli)
  |        \    \    \
  |         \    \    \  QAM16
  |          \    \    \
  |           \    \    \ QAM256 (Murakkab modulyatsiya: Yuqori tezlik, lekin yuqori SNR talab qiladi)
  +-------------------------------------> SNR (Signal kuchi)
```

#### Dinamik Tezlik Moslashuvi (Rate Adaptation):
Noutbukingiz routerdan uzoqlashgani sari signal kuchi (SNR) tushib boradi. Agar router yuqori tezlikda paket yuborishda davom etsa, barcha bitlar xato bo‘lib tushadi.  
Shu sababli Wi-Fi kontroller avtomatik ravishda modulyatsiya turini o‘zgartiradi:
* Router yonida turganda: **QAM256** modulyatsiyasi $\implies$ **54 Mbps – 600 Mbps**.
* Xonadan uzoqqa chiqqanda: **BPSK** modulyatsiyasiga tushadi $\implies$ **11 Mbps – 1 Mbps**.  
*(Tezlik pasayadi, lekin aloqa uzilib qolmaydi!)*

---

### 6.2.2. Yashirin Terminal Muammosi (The Hidden Terminal Problem)

Simsiz tarmoqlarda to‘qnashuvlarning eng asosiy sababi:

```
[ Tugun A ] ~ ~ ~ ~ ~ ~ > [ Access Point (AP) ] < ~ ~ ~ ~ ~ ~ [ Tugun B ]
(Chap xonada)                 (Zalda o'rtada)                 (O'ng xonada)
      |                                                             |
      \======= O'rtada qalin beton devor: A va B bir-birini =======/
                             eshita olmaydi!
```

1. **A tugun** efirni tekshiradi. Devor ortidagi B ni eshita olmagani uchun u efirni mutlaqo bo‘sh deb o‘ylaydi va AP ga paket uzatishni boshlaydi.
2. **B tugun** ham efirni tekshiradi. U ham A ni eshita olmaydi va efir bo‘sh deb o‘ylab, ayni paytda AP ga paket uzatishni boshlaydi.
3. **Natija:** AP antennasida ikkala signal bir-birining ustiga tushib, **to‘qnashuv (Collision)** yuz beradi! Har ikkala ma’lumot ham yo‘qoladi.

---

### 6.2.3. CDMA (Code Division Multiple Access) Matematikasi

Uyali aloqa (3G) va sun’iy yo‘ldosh tizimlarida bir vaqtning o‘zida bitta umumiy chastotada to‘qnashuvsiz ma’lumot uzatish imkonini beruvchi kodli bo‘linish texnologiyasi.

#### Matematik Modeli:
* Har bir $i$-tugunga $M$ bitdan iborat unikal **Chip Kriptografik Kodi** beriladi: $c_i = (c_{i,1}, c_{i,2}, \dots, c_{i,M})$, bu yerda $c \in \{-1, +1\}$.
* Barcha kodlar o‘zaro **Ortogonal (perpendikulyar)** bo‘lishi shart:
  $$c_i \cdot c_j = \frac{1}{M} \sum_{m=1}^{M} c_{i,m} \cdot c_{j,m} = 0 \quad (i \ne j)$$
  $$c_i \cdot c_i = 1$$
* Agar $i$-tugun $d_i \in \{-1, +1\}$ bitini jo‘natmoqchi bo‘lsa, u $d_i \times c_i$ signallar vektorini efirga chiqaradi.
* Efirda barcha jo‘natuvchilarning signallari bir-biriga qo‘shiladi: $S = \sum d_i \cdot c_i$.
* **Qabul qiluvchi $i$-tugunning ma’lumotini qanday ajratib oladi?**  
  U butun efir yig‘indisi $S$ ni o‘sha tugunning shaxsiy kodi $c_i$ ga skalyar ko‘paytiradi:
  $$\text{Chiqish} = S \cdot c_i = \left( \sum d_k c_k \right) \cdot c_i = d_i (c_i \cdot c_i) + \sum_{k \ne i} d_k (c_k \cdot c_i) = d_i \cdot 1 + 0 = d_i$$
  *(Barcha begona shovqinlar ortogonallik tufayli nolga aylanib ketadi va faqat kerakli ma’lumot ajralib chiqadi!)*

---

## 6.3. WI-FI: IEEE 802.11 SIMSIZ LOKAL TARMOQLARI

### 6.3.1. 802.11 Standartlarining Evolyutsiyasi

| Standart | Chiqqan yili | Chastota | Maksimal Tezlik | Asosiy yangilik |
| :--- | :--- | :--- | :--- | :--- |
| **802.11b** | 1999 | 2.4 GHz | 11 Mbps | Dastlabki ommaviy Wi-Fi |
| **802.11a** | 1999 | 5 GHz | 54 Mbps | 5 GHz diapazoni, OFDM |
| **802.11g** | 2003 | 2.4 GHz | 54 Mbps | 2.4 GHz da OFDM |
| **802.11n (Wi-Fi 4)** | 2009 | 2.4 / 5 GHz | 600 Mbps | **MIMO** (Ko‘p antennali uzatish) |
| **802.11ac (Wi-Fi 5)**| 2014 | 5 GHz | 3.46 Gbps | Keng kanallar (80/160 MHz), MU-MIMO |
| **802.11ax (Wi-Fi 6)**| 2019 | 2.4 / 5 / 6 GHz| 9.6 Gbps | **OFDMA** (Kichik sub-kanallar), Target Wake Time |

---

### 6.3.2. 802.11 Arxitekturasi: BSS va Skanerlash

* **BSS (Basic Service Set):** Wi-Fi tarmog‘ining eng kichik g‘ishti.
  * Odatda 1 ta **Access Point (AP — Kirish nuqtasi)** va unga ulangan simsiz xostlardan iborat.
* **SSID (Service Set Identifier):** Tarmog‘ingizning nomi (masalan, `TATU_Student_WiFi`).
* **BSSID:** AP ning L2 48-bitli MAC manzili.
* **Kanallar:** 2.4 GHz diapazoni 11 ta kanalga bo‘lingan. O‘zaro chastotasi kesishmaydigan (to‘qnashmaydigan) kanallar faqat **1, 6 va 11-kanallar**dir.

#### Tarmoqni Qidirish (Scanning):
1. **Passiv skanerlash:** AP har 100 ms da barcha xostlarga eshitiladigan qilib **Beacon Frame (Mayoq kadr)** tarqatadi. Noutbuk bularni tinglab, mavjud Wi-Fi lar ro‘yxatini tuzadi.
2. **Aktiv skanerlash:** Noutbukning o‘zi `Probe Request` kadrini havoga chiqaradi, yaqin atrofdagi barcha AP lar unga `Probe Response` qaytaradi.

---

### 6.3.3. 802.11 MAC Protokoli: CSMA/CA (Collision Avoidance)

Nega Wi-Fi da simli Ethernet kabi to‘qnashuvni sezuvchi **CSMA/CD** ishlamaydi?
1. Tarmoq kartasining o‘z uzatuvchi signali qabul qiluvchi antennani "kar qilib qo‘yadi" — jo‘natish paytida o‘z signalidan million marta zaif to‘qnashuvni eshita olmaydi.
2. Yashirin terminal muammosi tufayli to‘qnashuv jo‘natuvchida emas, qabul qiluvchi AP da yuz beradi.

Shu sababli Wi-Fi da **CSMA/CA (To‘qnashuvlarning oldini olish)** qo‘llaniladi:

```
[Efirni tekshirish] ---> Efir bandmi? ---> Ha: Kutib turadi
                               |
                               v Yo'q: Bo'sh
                 [ DIFS vaqt kutadi ]
                               |
                               v
            [ Random Backoff taymerini yoqadi ]
                               | (Har safar efir bo'sh bo'lsa taymer kamayadi)
                               v Taymer 0 bo'ldi!
                    [ Butun kadrni jo'natadi ]
                               |
                               v
              [ SIFS vaqt kutib L2 ACK oladi ]
```

* **Majburiy L2 ACK:** Qabul qiluvchi tomon kadrni CRC xatosiz olgach, **SIFS** vaqt kutib, qat’iy **ACK** qaytaradi. Agar jo‘natuvchi belgilangan vaqtda ACK ololmasa, kadr yo‘qolgan deb hisoblanadi va ortga chekinish taymeri 2 barobar oshirilib, qayta uzatiladi.

---

### 6.3.4. Yashirin Terminallarga Yechim: RTS / CTS Mexanizmi

To‘qnashuvlarning oldini olish uchun qisqa boshqaruv kadrlari ishlatiladi:

```
[Xost A]                                [Access Point (AP)]                           [Xost B]
   |                                             |                                       |
   |-- 1. RTS (Request to Send: 1000 bayt) ----->|                                       |
   |                                             |-- 2. CTS (Clear to Send: A ga ruxsat)-|
   |                                             |      (Barcha xostlar eshitadi)        v
   |<-- 2. CTS ----------------------------------|                                   [ NAV taymeri:
   |                                             |                                    B 5 ms jim turadi ]
   |== 3. KADRNI XOTIRJAM JO'NATISH ============>|                                       |
   |<-- 4. ACK ----------------------------------|                                       |
```

1. Xost A kadr uzatishdan oldin AP ga qisqa **RTS (Request to Send)** yuboradi.
2. AP barcha eshitadigan qilib **CTS (Clear to Send)** yuboradi.
3. Devor ortidagi Xost B ham CTS ni eshitadi va o‘zining **NAV (Network Allocation Vector)** taymerini yoqib, A ning uzatishi tugaguncha efirga chiqmay kutib turadi!

---

### 6.3.5. 802.11 Kadr Formati: Nega 4 ta MAC Manzil Maydoni Bor?

```
+---------------+---------------+---------------+----------+---------------+
| Frame Control |   Address 1   |   Address 2   | Address 3| Sequence Ctrl |
|   (2 bayt)    |    (6 bayt)   |    (6 bayt)   |  (6 bayt)|    (2 bayt)   |
+---------------+---------------+---------------+----------+---------------+
|   Address 4   |            Payload            |    CRC   |
|    (6 bayt)   |          (0-2312 bayt)        |  (4 bayt)|
+---------------+-------------------------------+----------+
```

* **Address 1 (Qabul qiluvchi simsiz MAC):** Kadrni havodan bevosita tutib oluvchi AP ning MAC manzili.
* **Address 2 (Jo‘natuvchi simsiz MAC):** Kadrni havoga nurlantirgan noutbukning MAC manzili.
* **Address 3 (Router interfeysi MAC):** AP bu kadrni simli Ethernetga chiqarganda, u qaysi Default Gateway routeriga borishi kerakligini ko‘rsatuvchi MAC manzil. *(AP oddiy L2 ko‘prik bo‘lgani uchun L3 routerning manzilini bilishi shart!)*
* **Address 4:** Faqat AP lar bir-biri bilan simsiz ko‘prik (Ad-hoc / Wireless Distribution System — WDS) rejimida gaplashganda ishlatiladi.

---

## 6.4. UYALI ALOQA TARMOQLARI (4G LTE ARXITEKTURASI)

An’anaviy telefon tarmoqlaridan farqli o‘laroq, **4G LTE (Long-Term Evolution)** da ovozli kanallar butunlay olib tashlangan — **barcha ma’lumotlar (ovoz ham, video ham) to‘liq IP paketlar** ko‘rinishida uzatiladi.

```
+------------------------------------+------------------------------------+
| E-UTRAN (Radio Kirish Tarmog'i)   | EPC (Evolved Packet Core - Yadro)  |
|                                    |                                    |
|   [ Smartfon (UE) ]                |     +------- [ HSS ] (Foydalanuvchi|
|           |                        |     |                 bazasi)      |
|     (LTE Radio)                    |     v                              |
|           v                        |  [ MME ] (Boshqaruv / Ro'yxat)     |
|       [ eNodeB ] ===================>    |                              |
|    (Baza minorasi)                 |     v                              |
|                                    |  [ SGW ] (Serving Gateway)         |
|                                    |     |                              |
|                                    |     v                              |
|                                    |  [ PGW ] (Packet Data Network GW)  |
+------------------------------------+-----|------------------------------+
                                           v
                                    [ INTERNET ]
```

1. **UE (User Equipment):** Smartfon yoki 4G modem.
2. **eNodeB (Evolved Node B):** Baza stansiyasi minorasi. U shunchaki antenna emas, balki radio resurslarini taqsimlovchi va shifrlashni amalga oshiruvchi aqlli tugundir.
3. **MME (Mobility Management Entity):** Butun tizimning "miyasi". Foydalanuvchining autentifikatsiyasini tekshiradi (HSS bazasidan) va telefon minoradan-minoraga o‘tganda uning marshrutini boshqaradi.
4. **SGW (Serving Gateway):** Ma’lumotlar paketlarini marshrutlovchi mahalliy shlyuz.
5. **PGW (PDN Gateway):** Mobil tarmoqning tashqi Internet bilan chegarasi. Smartfoningizga IP-manzil beruvchi va NAT/QoS siyosatlarini qo‘llovchi asosiy shlyuz.

---

## 6.5. MOBILLIKNI BOSHQARISH VA MARSHRUTLASH (MOBILE IP)

Foydalanuvchi poyezdda harakatlanib, Toshkentdan Samarqandga o‘tganda uning **IP-manzili o‘zgarmasligi kerak**, aks holda barcha ochiq TCP ulanishlar uzilib ketadi!

### Asosiy Tushunchalar:
* **Home Network (Uy tarmog‘i):** Qurilmaning doimiy ro‘yxatdan o‘tgan tarmog‘i.
* **Home Agent:** Uy tarmog‘idagi qurilmaga xizmat qiluvchi router.
* **Permanent IP Address:** Qurilmangizning o‘zgarmas doimiy IP manzili.
* **Foreign Network (Mehmon tarmog‘i):** Qurilma hozir kirib borgan begona tarmoq.
* **Care-of-Address (COA):** Qurilmaning begona tarmoqda olgan vaqtinchalik manzili.

```
[ Do'stingiz (Sender) ]                                  [ Sizning Uyingiz (Home Agent) ]
        |                                                              |
        | 1. Do'stingiz doimiy IP ga paket yuboradi                   |
        |------------------------------------------------------------->| Home Agent sizning qayerdaligingizni
        |                                                              | biladi va paketni yangi IP ga
        |                                                              | IP-in-IP qilib TUNEL qiladi
        |                                                              v
        |                                                 [ Siz mehmonda turgan tarmoq ]
        |<=========== 3. Siz to'g'ridan-to'g'ri javob qaytarasiz ======== [ Sizning Smartfoningiz ]
```

* **Bilvosita marshrutlash (Indirect Routing / Triangle Routing):**
  1. Do‘stingiz sizning doimiy IP manzilingizga paket yuboradi.
  2. Paket uyingizdagi **Home Agent**ga boradi.
  3. Home Agent paketni oladi va uni ikkinchi IP sarlavhasiga o‘rab (**IP-in-IP Tunneling**), siz turgan Samarqanddagi COA manziliga yo‘naltiradi.
  4. Smartfoningiz paketni yechib oladi va javobni to‘g‘ridan-to‘g‘ri do‘stingizga yuboradi.

---

## 6.6. 6-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. **Simsiz kanallar** so‘nish, shovqin va ko‘p nurlilik kabi og‘ir fizik xususiyatlarga ega bo‘lib, dinamik tezlik moslashuvidan foydalanadi.
2. **Wi-Fi (802.11)** to‘qnashuvlardan qochish (**CSMA/CA**), majburiy L2 ACK va yashirin terminallarni yechish uchun **RTS/CTS** mexanizmini ishlatadi.
3. 802.11 kadrida L2 va L3 ko‘prigini ta’minlash uchun **4 ta MAC manzil** mavjud.
4. **4G LTE** barcha xizmatlarni to‘liq paketli IP arxitekturasi (**EPC**) orqali yetkazadi.
5. Mobillikni boshqarish foydalanuvchi joylashuvini o‘zgartirganda ham TCP sessiyalarni uzmasdan ushlab turadi.

### 🔗 Joriy papkadagi laboratoriyalar bilan bog‘liqlik:
* Ushbu bobdagi 802.11 boshqaruv kadrlari, Beacon freymlar va simsiz himoya tushunchalari to‘g‘ridan-to‘g‘ri **`A8_project`** (Simsiz tarmoq xavfsizligini sozlash: Virtual Wi-Fi muhitida WPA3-SAE va 802.11w PMF majburiy himoyasi) laboratoriyasining nazariy tayanchidir!
