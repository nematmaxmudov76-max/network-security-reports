# 5-BOB: KANAL QATLAMI VA LOKAL TARMOQLAR (The Link Layer and LANs)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Kanal qatlami xizmatlari va apparat darajasida amalga oshirilishi (NIC), xatoliklarni aniqlash va tuzatishning matematik asoslari (2D Parity, CRC tsiklik ortiqcha kod), ko‘p nuqtali kirish protokollari (CSMA/CD, Binary Exponential Backoff), MAC manzillash va ARP protokoli, Ethernet standarti va kadr strukturasi, aqlli kommutatorlar (Switch self-learning CAM table), Virtual LANlar (VLAN, 802.1Q Trunking) hamda Ma’lumotlar Markazi tarmoq arxitekturasini chuqur o‘rganish.

---

## 5.1. KANAL QATLAMI: XIZMATLAR VA TUSHUNCHALAR

Agar **Tarmoq qatlami (IP)** paketni jo‘natuvchi xostdan qabul qiluvchi xostgacha bo‘lgan butun global yo‘lni nazorat qilsa, **Kanal qatlami (Data Link Layer)** paketni **bitta aloqa liniyasi (link) bo‘ylab bevosita ulangan bir tugundan ikkinchisiga** o‘tkazib berish uchun javobgardir.

```
Hayotiy analogiya:
Global Sayohat: Toshkentdan Nyu-Yorkka borish (Tarmoq qatlami).
Yo'l bo'laklari (Link Layer):
* 1-bo'lak: Toshkentdagi uydan Aeroportga Taksi orqali (1-Link / Ethernet)
* 2-bo'lak: Toshkentdan Istanbulga Samolyot orqali (2-Link / Optik kabel)
* 3-bo'lak: Istanbuldan Nyu-Yorkka Kema orqali (3-Link / Sun'iy yo'ldosh)
Har bir transport vositasi o'ziga xos qoidalarga ega, lekin yo'lovchi (IP paket) o'zgarmaydi!
```

---

### 5.1.1. Kanal Qatlami Taqdim Etuvchi Asosiy Xizmatlar

1. **Kadrlash (Framing):**
   * Yuqori Tarmoq qatlamidan kelgan IP datagramni o‘zining **Kadr (Frame)** strukturasiga joylaydi.
   * Kadrga bosh sarlavha (**Header**) va dum (**Trailer**) qo‘shiladi. Sarlavhada L2 jismoniy manzillari (**MAC Addresses**) ko‘rsatiladi.
2. **Kanalga kirish nazorati (Link Access / MAC Protocol):**
   * Agar aloqa muhiti umumiy bo‘lsa (masalan, Wi-Fi efiri yoki koaksial kabel), kadrlar to‘qnashib ketmasligi uchun efirni kim qachon egallashini tartibga soluvchi qoidalar.
3. **Ishonchli yetkazish (Reliable Delivery):**
   * Xatolik ehtimoli juda yuqori bo‘lgan simsiz kanallarda (Wi-Fi) qo‘llaniladi (L2 darajasidagi tasdiqlar — ACK/NAK orqali). Mis yoki optik kabellarda esa bu xizmat ortiqcha yuk hisoblanib, yuqori qatlamga (TCP ga) qoldiriladi.
4. **Xatoliklarni aniqlash va tuzatish (Error Detection and Correction):**
   * Signal kabel bo‘ylab harakatlanayotganda elektromagnit xalaqitlar tufayli bitlar o‘zgarib qolishi mumkin. Kanal qatlami apparat darajasida bu xatolarni aniqlaydi va kerak bo‘lsa tuzatadi.

---

### 5.1.2. Kanal Qatlami Qayerda Amalga Oshirilgan?

Kanal qatlami operatsion tizimning dasturiy ta’minotida emas, balki asosan **apparat darajasida — Tarmoq Kartasida (NIC — Network Interface Card / Ethernet Adapter)** joylashgan.
* NIC ichida maxsus kontroller chipi mavjud bo‘lib, u kadrlash, CRC hisoblash va signallarni kabelga chiqarishni yadro protsessori (CPU) ishtirokisiz, apparat tezligida bajaradi.
* Faqatgina yuqori qismlari (tarmoq drayveri, ARP boshqaruvi) operatsion tizim yadrosida ishlaydi.

---

## 5.2. XATOLIKLARNI ANIQLASH VA TUZATISH USULLARI

Kadr uzatilayotganda ma’lumot $D$ ga qo‘shimcha $E$ xatolikni tekshirish bitlari qo‘shiladi:

```
[  Data D (Ma'lumot bitlari)  |  EDC (Xatolikni tekshirish bitlari)  ]
```

---

### 5.2.1. Paritet Tekshiruvi (Parity Checks)

1. **1-O‘lchamli Paritet:**
   * Barcha bitlardagi birlar soni juft (Even parity) bo‘lishi uchun oxiriga bitta bit qo‘shiladi.
   * *Kamchiligi:* Agar yo‘lda 1 ta bit o‘zgarsa, xato darhol seziladi. Lekin bir vaqtda 2 ta bit o‘zgarsa, birlar soni yana juft bo‘lib qoladi va xato sezilmay qoladi!
2. **2-O‘lchamli Paritet (2D Parity):**
   * Ma’lumot bitlari $i$ ta qator va $j$ ta ustunli matritsa shaklida yoziladi. Har bir qator va har bir ustun uchun alohida paritet biti hisoblanadi.
   * **Afzalligi:** Agar bitta bit buzilsa, buzilgan bitning ham qatori, ham ustuni xato ko‘rsatadi. Ushbu qator va ustun kesishgan joyidagi bit aniqlanadi va uning teskarisi olinib, **xato avtomatik tuzatiladi (Forward Error Correction — FEC)**!

```
0 1 1 0 0 | 0   (Qator pariteti)
1 1 0 1 0 | 1
0 1 1 1 1 | 0
----------+--
1 1 0 0 1 | 1   (Ustun pariteti)
```

---

### 5.2.2. Tsiklik Ortiqcha Kod — CRC (Cyclic Redundancy Check)

Bugungi kunda barcha zamonaviy tarmoqlarda (Ethernet, Wi-Fi, 4G/5G) qo‘llaniladigan eng mukammal va kuchli apparat tekshiruvi.

#### CRC Matematikasi (Modulo-2 Arifmetikasi):
Modulo-2 arifmetikasida qo‘shish va ayirish amallari o‘zaro teng bo‘lib, oddiy **XOR (Inkor etuvchi YOKI)** amali orqali bajariladi (o‘nlikka o‘tish yoki qarz olish yo‘q!):
* $0 \oplus 0 = 0$
* $1 \oplus 1 = 0$
* $1 \oplus 0 = 1$
* $0 \oplus 1 = 1$

**Algoritm qadamlari:**
1. Jo‘natuvchi va qabul qiluvchi oldindan kelishilgan $r+1$ bitli **Generator $G$** dan foydalanadi. ($G$ ning eng yuqori biti doim `1` bo‘lishi shart).
2. Jo‘natuvchi $d$-bitli ma’lumot $D$ ning oxiriga $r$ ta nol qo‘shadi ($D \cdot 2^r$).
3. $D \cdot 2^r$ soni Modulo-2 usulida $G$ ga bo‘linadi va $r$-bitli qoldiq $R$ topiladi:
   $$R = \text{remainder} \left( \frac{D \cdot 2^r}{G} \right)$$
4. Kadr oxiriga ushbu qoldiq $R$ tirkaladi va $\langle D, R \rangle$ jo‘natiladi.
5. **Tekshirish:** Qabul qiluvchi kelgan $\langle D, R \rangle$ kadrini $G$ ga bo‘ladi. Agar qoldiq **0** chiqsa — kadrda xatolik yo‘q! Agar qoldiq noldan farq qilsa — kadrda bitlar buzilgan.

```
Amaliy misol:
D = 101110, d = 6 bit.
G = 1001,   r = 3 bit (r+1 = 4 bit).
D * 2^r = 101110000

Bo'lish jarayoni (XOR bilan):
101110000 | 1001
1001      |---------
----      |
 01010    |
  1001    |
  ----    |
   01100  |
    1001  |
    ----  |
     01010|
      1001|
      ----|
       011 (Qoldiq R = 011)

Yuboriladigan kadr: D va R birlashmasi = 101110 011
Qabul qiluvchi 101110011 ni 1001 ga bo'lganda qoldiq 000 chiqadi!
```

---

## 5.3. KO‘P NUQTALI KIRISH PROTOKOLLARI (MULTIPLE ACCESS)

Aloqa liniyalari ikkiga bo‘linadi:
1. **Point-to-Point (Nuqtadan-nuqtaga):** Faqat ikkita tugun (masalan, ikkita router o‘rtasidagi optik sim).
2. **Broadcast Link (Umumiy efir):** Bir nechta tugunlar yagona umumiy aloqa kanaliga ulangan (Wi-Fi, koaksial kabel).

> **To‘qnashuv (Collision) muammosi:**  
> Agar ikki yoki undan ortiq tugunlar bir vaqtning o‘zida kadr uzatishni boshlasa, ularning signallari bir-birining ustiga tushib, xashakdek aralashib ketadi (**Collision**). Qabul qiluvchilar hech bir kadrni o‘qiy olmaydi.

Multiple Access protokollari 3 ta toifaga bo‘linadi:

```
                    +------------------------------------+
                    |   MULTIPLE ACCESS PROTOKOLLARI     |
                    +------------------------------------+
                               /        |         \
                              /         |          \
           +--------------------+  +----------+  +--------------------+
           | Channel Partition  |  |  Random  |  |    Taking-Turns    |
           +--------------------+  +----------+  +--------------------+
           | - TDMA (Vaqt)      |  | - ALOHA  |  | - Polling          |
           | - FDMA (Chastota)  |  | - CSMA/CD|  | - Token Passing    |
           | - CDMA (Kod)       |  | - CSMA/CA|  | (Navbatma-navbat)  |
           +--------------------+  +----------+  +--------------------+
```

---

### 5.3.1. Tasodifiy Kirish: CSMA/CD (Ethernet Standarti)

An’anaviy Ethernet kabelida to‘qnashuvlarni boshqarish uchun **CSMA/CD (Carrier Sense Multiple Access with Collision Detection)** algoritmi yaratilgan.

#### Algoritm Qadamlari:
1. **Carrier Sense (Efirni tinglash):** Tarmoq kartasi kadr jo‘natishdan oldin kabelni tinglaydi. Agar liniyada boshqa signal bo‘lsa, u kutiladi (Listen before transmit).
2. **Uzatish:** Efir bo‘shaganini sezgach, kadrni uzatish boshlanadi.
3. **Collision Detection (To‘qnashuvni sezish):** Kadrni uzatayotgan paytda ham NIC kabeldagi kuchlanishni o‘lchab turadi. Agar boshqa tugun ham bir vaqtda uzatishni boshlagan bo‘lsa, to‘qnashuv yuz beradi.
4. **Jam Signal:** To‘qnashuv sezilishi bilanoq uzatish darhol to‘xtatiladi va tarmoqdagi barcha tugunlar to‘qnashuv yuz berganini bilishi uchun 48-bitli maxsus **Jam Signal (To‘qnashuv xabari)** yuboriladi.
5. **Ikkilik Eksponentsial Chekinish (Binary Exponential Backoff):**
   * Tugun darhol qayta urinmaydi (aks holda yana to‘qnashuv bo‘lardi).
   * $m$-to‘qnashuvdan so‘ng tugun $\{0, 1, 2, \dots, 2^m - 1\}$ to‘plamdan tasodifiy $K$ butun sonini tanlaydi (bu yerda $m \le 10$).
   * Tugun $K \times 512 \text{ bit}$ vaqt kutib, keyin yana efirni tinglaydi va qayta urinadi.
   * Agar to‘qnashuvlar takrorlanaversa, kutish diapazoni 2 barobarga (eksponentsial) oshirib boriladi. 16 ta muvaffaqiyatsiz urinishdan so‘ng uzatish bekor qilinadi.

---

## 5.4. KOMMUTATSIYALANGAN LOKAL TARMOQLAR (SWITCHED LANS)

Bugungi kunda xablardan butunlay voz kechilgan va barcha lokal tarmoqlar **Kommutatorlar (Switchlar)** asosida quriladi.

### 5.4.1. MAC Manzillar va ARP Protokoli

* **MAC Manzil (LAN / Physical / Hardware Address):**
  * Hajmi: **48 bit (6 bayt)**. Format: `00-1A-2B-3C-4D-5E` yoki `00:1a:2b:3c:4d:5e`.
  * Dastlabki 24 bit — **OUI (Organizationally Unique Identifier)**, IEEE tomonidan chip ishlab chiqaruvchisiga (Intel, Realtek, Cisco) beriladi.
  * Qolgan 24 bit — tarmoq kartasining unikal seriya raqami.
  * *Muhim farq:* IP-manzil geografik bog‘liq (xuddi uy manzilidek), MAC-manzil esa qurilmaning shaxsiy pasport raqamidek o‘zgarmasdir.

#### ARP (Address Resolution Protocol — RFC 826)
L3 IP-manzilni L2 MAC-manzilga bog‘lovchi ko‘prik.
* Kompyuter paketni `192.168.1.5` ga yubormoqchi, lekin uning MAC manzilini bilmaydi.
* **ARP Request:** Kompyuter butun tarmoqqa Broadcast (`FF:FF:FF:FF:FF:FF`) yuboradi:  
  *"Kimda 192.168.1.5 bo‘lsa, o‘z MAC manzilingni ayt, men 192.168.1.1 man!"*
* **ARP Reply:** O‘sha IP egasi o‘z MAC manzili bilan to‘g‘ridan-to‘g‘ri (Unicast) javob qaytaradi.
* Kompyuter bu juftlikni o‘zining **ARP Jadvaliga (ARP Cache)** yozib oladi (TTL odatda 20 daqiqa).

> **Xavfsizlik Zaifligi — ARP Spoofing / Poisoning:**  
> ARP protokoli hech qanday autentifikatsiyaga ega emas! Xaker tarmoqqa: *"192.168.1.1 (Router) menman, mening MAC manzilim mana bu"* deb soxta ARP Reply yuborishi mumkin. Natijada barcha kompyuterlar internetga chiqayotgan trafikni routerga emas, xakerning kompyuteriga yuboradi (**Man-in-the-Middle hujumi**)!

---

### 5.4.2. Ethernet Kadr Formati (IEEE 802.3)

```
+------------+------------+------------+------+-----------------+---------+
|  Preamble  |  Dest MAC  | Source MAC | Type | Payload (IP pkt)|   CRC   |
|  (8 bayt)  |  (6 bayt)  |  (6 bayt)  | (2 b)|  (46-1500 bayt) | (4 bayt)|
+------------+------------+------------+------+-----------------+---------+
```

* **Preamble (8 bayt):** Dastlabki 7 bayt `10101010` va oxirgi 1 bayt `10101011` (SFD — Start Frame Delimiter). Qabul qiluvchi tarmoq kartasining takt chastotasini kiruvchi signal bilan sinxronlash uchun xizmat qiladi.
* **Destination & Source MAC (6 baytdan):** Qabul qiluvchi va jo‘natuvchi qurilmalarning jismoniy MAC manzillari.
* **Type (2 bayt):** Kadr ichidagi yuqori qatlam protokolini bildiradi (`0x0800` — IPv4, `0x0806` — ARP, `0x86DD` — IPv6).
* **Payload (46 dan 1500 baytgacha):** IP datagram. (Agar IP paket 46 baytdan kichik bo‘lsa, Ethernet minimal talabni bajarish uchun kadrga padding nollarini qo‘shadi).
* **CRC (4 bayt):** Xatolikni tekshiruvchi 32-bitli tsiklik ortiqcha kod.

---

### 5.4.3. Link-Layer Switchlar (Kommutatorlar)

Switch — bu L2 darajasida ishlovchi, ko‘p portli aqlli qurilmadir:
* **To‘qnashuvsizlik (Collision-Free):** Har bir port alohida to‘qnashuv domeni (Collision Domain) hisoblanadi. Bir vaqtda bir nechta portlar to‘liq dupleksda (Full-Duplex) bemalol bir-biriga xalaqit bermay ma’lumot uzatadi.

#### Switch O‘z-o‘zini Qanday O‘rganadi? (Self-Learning CAM Table)
Switchga hech qanday boshlang‘ich konfiguratsiya kerak emas (**Plug-and-Play**). U o‘zining **Kommutatsiya Jadvalini (Switch Table / CAM Table)** o‘zi tuzadi:

```
[Port 1: PC A]              [Port 2: PC B]              [Port 3: PC C]
       \                          |                          /
        +---------------------------------------------------+
        |                SWITCH (CAM Jadvali)               |
        | [ MAC Manzil ]           [ Port ]       [ Vaqt ]  |
        | AA:AA:AA:AA:AA:AA   ->   Port 1      -> 14:00     |
        | BB:BB:BB:BB:BB:BB   ->   Port 2      -> 14:01     |
        +---------------------------------------------------+
```

1. **O‘rganish:** Switch portiga kadr kelishi bilan, switch kadrning **Source MAC (Manba MAC)** manzilini o‘qiydi va uni kelgan port raqami bilan jadvalga yozib qo‘yadi.
2. **Yo‘naltirish / Filtrlash:**
   * Switch kadrning **Destination MAC** manzilini o‘qiydi.
   * *Agar manzil jadvalda bor bo‘lsa:* Kadrni faqat va faqat o‘sha portga chiqaradi (**Forwarding**).
   * *Agar manzil jadvalda yo‘q bo‘lsa (yoki Broadcast bo‘lsa):* Switch kadrni kelgan portidan tashqari barcha portlarga chiqarib yuboradi (**Flooding**). Kerakli xost javob berishi bilan uning manzili ham jadvalga yoziladi!

---

### 5.4.4. Virtual LANlar (VLAN) va 802.1Q Trunking

An’anaviy tarmoqda switchega ulangan barcha kompyuterlar yagona umumiy efir hududida (**Broadcast Domain**) bo‘ladi.
* **Muammo:** Bitta kompyuterga virus tushib broadcast yuborsa, tarmoqdagi barcha 500 ta kompyuterning protsessori qotib qoladi. Bo‘limlar (Buxgalteriya, Xavfsizlik, Mehmonlar) bir-birining trafigini ko‘rib turadi.

#### VLAN (Virtual Local Area Network):
Bitta jismoniy switchni bir nechta mustaqil mantiqiy tarmoqlarga ajratish imkonini beradi.
* **Port-Based VLAN:** Masalan, 1–10 portlar VLAN 10 (Moliya), 11–20 portlar VLAN 20 (Talabalar).
* Ular bitta switchda turgan bo‘lsa ham, **o‘zaro routersiz mutlaqo gaplasha olmaydi** (L2 darajasida qat’iy izolyatsiya qilinadi).

```
[Switch A] ================= (TRUNK PORT: 802.1Q) ================= [Switch B]
VLAN 10 va VLAN 20 trafiklari yagona kabeldan maxsus teg (Tag) bilan o'tadi.
```

#### IEEE 802.1Q Trunking Formati:
Ikkita switch o‘rtasida bir nechta VLAN trafiklarini bitta kabel orqali olib o‘tish uchun kadr sarlavhasiga **4 baytlik 802.1Q tegi** qo‘shiladi:
* `TPID (2 bayt)`: `0x8100` (802.1Q tegi ekanini bildiradi).
* `Priority (3 bit)`: QoS xizmat sifati ustunligi.
* `VLAN ID (VID — 12 bit)`: Qaysi VLAN ga tegishli ekanini bildiruvchi raqam ($2^{12} = 4096$ ta VLAN yaratish mumkin).

---

## 5.5. MA’LUMOTLAR MARKAZI TARMOQLARI (DATA CENTER NETWORKING)

Zamonaviy bulutli ma’lumotlar markazlarida (Google, AWS) yuz minglab serverlar o‘zaro ulanadi.

```
                    [ BORDER ROUTERS (Tashqi Internet) ]
                                   |
                         [ ACCESS ROUTERS ]
                                   |
                  +----------------+----------------+
                  |                                 |
         [ SPINE SWITCH 1 ]                [ SPINE SWITCH 2 ]
             /          \                      /          \
            /            \                    /            \
    [ LEAF SWITCH 1 ]  [ LEAF SWITCH 2 ] [ LEAF SWITCH 3 ] [ LEAF SWITCH 4 ]
           |                  |                 |                 |
     (ToR Switch)       (ToR Switch)      (ToR Switch)       (ToR Switch)
           |                  |                 |                 |
       [ SERVER ]         [ SERVER ]        [ SERVER ]         [ SERVER ]
       [ SERVER ]         [ SERVER ]        [ SERVER ]         [ SERVER ]
```

* **ToR (Top-of-Rack) Switch:** Har bir server stoykasining (Rack) tepasida turadi va stoykadagi 20–40 ta serverni o‘zaro ulaydi.
* **Spine-and-Leaf Arxitekturasi:** Har bir Leaf (barg) switch barcha Spine (umurtqa) switchlariga ulanadi. Natijada har qanday serverdan boshqa ixtiyoriy servergacha bo‘lgan masofa doimo qat’iy **bir xil va minimal kechikishli (Non-blocking fabric)** bo‘ladi!

---

## 5.6. 5-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. **Kanal qatlami** paketlarni kadrlar ko‘rinishida jismoniy muhit orqali apparat darajasida (NIC) yetkazadi.
2. **CRC** — apparatda bitlar xatosini aniqlashning eng ishonchli usuli.
3. **ARP** protokoli IP-manzillarni 48-bitli MAC-manzillarga bog‘laydi.
4. **Switchlar** kadrning Manba MAC manzilini o‘qib, o‘z-o‘zini o‘rganadi (CAM table) va portlar bo‘yicha to‘qnashuvsiz kommutatsiya qiladi.
5. **VLAN va 802.1Q** korporativ tarmoqni xavfsiz mantiqiy guruhlarga ajratishning tayanchidir.

### 🔗 Joriy papkadagi laboratoriyalar bilan bog‘liqlik:
* Ushbu bobdagi **VLAN segmentatsiyasi, MAC nazorati va kirishni cheklash** to‘g‘ridan-to‘g‘ri **`A10_project`** (DLP va NAC Siyosatlarini amalda qo‘llash / 802.1X) laboratoriyasining nazariy poydevoridir!
* Kadrlar tahlili va ARP xabarlari esa **`A9_project`** (Wireshark bilan tarmoq monitoringi) da paketlarni L2 darajasida tergov qilish uchun zarurdir!
* Cisco Packet Tracer da **2- va 3-Laboratoriyalar** (VLAN, Trunk, Port Security) aynan shu bob bilimlarini amalda mustahkamlaydi!
