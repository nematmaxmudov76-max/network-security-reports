# 4-BOB: TARMOQ QATLAMI (The Network Layer)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Tarmoq qatlamining asosiy vazifalari (Forwarding vs Routing, Data Plane vs Control Plane), routerlarning ichki apparat arxitekturasi va kommutatsiya matrisalari, IPv4 datagram formati va fragmentatsiya matematikasi, Subnetting (CIDR, quyi tarmoqlarni hisoblash), DHCP, NAT/PAT mexanizmi, ICMP protokoli, IPv6 arxitekturasi hamda global marshrutlash algoritmlari (Deykstra Link-State, Bellman-Ford Distance-Vector, OSPF va BGP) ni chuqur o‘rganish.

---

## 4.1. TARMOQ QATLAMI VAZIFALARI: FORWARDING VA ROUTING

Tarmoq qatlamining fundamental maqsadi — ma’lumot paketlarini **jo‘natuvchi xostdan qabul qiluvchi xostga** butun global tarmoq bo‘ylab, ko‘plab oraliq aloqa liniyalari va routerlar orqali yetkazishdir.

Buning uchun tarmoq qatlami ikkita tubdan farq qiluvchi jarayonni amalga oshiradi:

```
+-------------------------------------------------------------------------+
|                                ROUTER                                   |
|                                                                         |
|   [ CONTROL PLANE (Boshqaruv tekisligi: Dasturiy / Millisekundlar) ]    |
|   * OSPF, BGP marshrutlash algoritmlari                                |
|   * Butun tarmoq xaritasini tahlil qilish                               |
|   * Marshrutlash jadvalini (Routing Table) hisoblab chiqish             |
|                                 |                                       |
|                                 v (Jadvalni ko'chirib berish)           |
|                                                                         |
|   [ DATA PLANE (Ma'lumotlar tekisligi: Apparat / Nanosekundlar) ]       |
|   * Kiruvchi portga paket keldi                                         |
|   * Forwarding Table dan manzilni qidirish (Lookup)                     |
|   * Paketni kommutatsiya matrisi orqali chiquvchi portga surish        |
+-------------------------------------------------------------------------+
```

1. **Forwarding (Yo‘naltirish / Uzatish):**
   * Bu — **mahalliy (lokal), bitta router ichidagi** apparat jarayoni.
   * Paket routerning kiruvchi interfeysiga kelganda, router uning sarlavhasidagi `Destination IP` manzilini o‘qiydi va o‘zining **Yo‘naltirish jadvali (Forwarding Table)** ga qarab, paketni tegishli chiquvchi interfeysga chiqarib yuboradi. Bu jarayon nanosekundlarda (apparat chiplarida) kechadi.
2. **Routing (Marshrutlash):**
   * Bu — **global, butun tarmoq bo‘ylab** amalga oshiriladigan mantiqiy jarayon.
   * Paketning boshlang‘ich nuqtadan oxirgi manzilgacha bo‘lgan eng maqbul yo‘lini (Path) aniqlash uchun barcha routerlar o‘zaro marshrutlash protokollari (OSPF, BGP) orqali ma’lumot almashadi va yo‘naltirish jadvallarini shakllantiradi.

> **Hayotiy analogiya:**  
> **Routing** — bu haydovchining xaritani ochib, Toshkentdan Buxoroga qaysi viloyatlar va trassalar orqali borishni rejalashtirishi.  
> **Forwarding** — bu mashinaning aniq bir chorrahaga yetib kelganda svetofor ko‘rsatkichiga qarab o‘ngga yoki chapga burilishi!

---

## 4.2. VIRTUAL-KANAL (VC) VA DATAGRAM TARMOQLARI

Tarixda tarmoq qatlamini qurishning ikkita falsafasi bo‘lgan:
1. **Virtual-Circuit (VC) Tarmoqlari (ATM, Frame Relay, X.25):**
   * Xuddi telefon tarmog‘i kabi, ma’lumot uzatishdan oldin boshlang‘ich va oxirgi xost o‘rtasidagi barcha oraliq routerlarda ulanish o‘rnatiladi (**VC Setup**). Har bir paketga butun manzil emas, qisqa VC raqami yopishtiriladi. Routerlar har bir ulanish holatini (state) eslab turadi.
2. **Datagram Tarmoqlari (Internet):**
   * Tarmoq qatlamida hech qanday ulanish o‘rnatilmaydi!
   * Har bir paket mustaqil (**Datagram**) bo‘lib, sarlavhasida to‘liq maqsad IP-manzili yoziladi.
   * Ikkita ketma-ket yuborilgan paket tarmoqdagi o‘zgarishlarga qarab butunlay boshqa-boshqa routerlar orqali yetib borishi mumkin.

### Nega Internet Datagram Modelini Tanladi?
* **Internet falsafasi:** "Tarmoq yadrosi (routerlar) imkon qadar sodda, arzon va tezkor bo‘lishi kerak. Barcha murakkab vazifalar (ishonchlilik, ulanish, xatolarni tuzatish) chetdagi xostlarga (TCP ga) yuklanadi".
* Agar bitta router o‘chib qolsa, Datagram tarmog‘i darhol yangi yo‘lni topib ketaveradi (hech qanday ulanish buzilmaydi). VC tarmoqlarida esa router o‘chsa, unga bog‘langan barcha ulanishlar uzilib, qaytadan ochilishi kerak bo‘lardi.

---

## 4.3. ROUTER ICHIDA NIMA BOR? (ROUTER ARXITEKTURASI)

Zamonaviy router — bu maxsus ixtisoslashgan qudratli hisoblash arxitekturasidir:

```
                   +---------------------------------------+
                   |           ROUTING PROCESSOR           | (Boshqaruv tekisligi)
                   +---------------------------------------+
                                       |
                   +---------------------------------------+
                   |           SWITCHING FABRIC            | (Kommutatsiya matrisi)
                   +---------------------------------------+
                     /                 |                 \
                    v                  v                  v
             +-------------+    +-------------+    +-------------+
             | Input Port  |    | Output Port |    | Output Port |
             +-------------+    +-------------+    +-------------+
```

---

### 4.3.1. Kiruvchi Portlar va Eng Uzun Prefiks Mosligi (Longest Prefix Match)

Kiruvchi portda signal qabul qilinadi (L1), kadr yechiladi (L2) va paketning `Destination IP` manzili o‘qiladi.

Router qidiruvni qanday amalga oshiradi? Router jadvalida 4 milliardta IP saqlanmaydi, balki IP-prefiks diapazonlari saqlanadi.

#### Longest Prefix Match (LPM) Qoidasi:
Agar paketning IP-manzili jadvaldagi bir nechta qatorga to‘g‘ri kelsa, router **eng ko‘p bit mos kelgan (eng uzun prefiksli / eng aniq)** qatorni tanlaydi!

**Misol:**
Router jadvali:
* Qoida 1: `11001000 00010111 00010000` (`/24`) $\implies$ Port 1 ga chiqsin.
* Qoida 2: `11001000 00010111 00011000` (`/24`) $\implies$ Port 2 ga chiqsin.
* Qoida 3: `11001000 00010111 00011000 11000000` (`/26`) $\implies$ Port 3 ga chiqsin.
* Boshqa barcha IPlar $\implies$ Port 4 (Default Gateway).

Agar paketning manzili `11001000 00010111 00011000 11001010` bo‘lsa:
* U ham Qoida 2 ga (24 ta bit mos), ham Qoida 3 ga (26 ta bit mos) to‘g‘ri keladi.
* **LPM qoidasiga ko‘ra, eng uzun prefiks (26 bit) g‘olib bo‘ladi va paket Port 3 ga yo‘naltiriladi!**

Zamonaviy routerlarda bu qidiruv nanosekundlarda bajarilishi uchun **TCAM (Ternary Content Addressable Memory)** chiplari ishlatiladi.

---

### 4.3.2. Kommutatsiya Matrisi (Switching Fabric) Turlari

Paketni kiruvchi portdan kerakli chiquvchi portga o‘tkazib beruvchi ichki mexanizm:
1. **Xotira orqali (Switching via Memory):** Paket CPU xotirasiga o‘qiladi va chiquvchi portga yoziladi. O‘ta sekin, tezlik tizim xotirasi shinasiga bog‘liq.
2. **Shina orqali (Switching via Bus):** Barcha portlar bitta umumiy ichki shinaga ulangan. Bitta vaqtda faqat bitta paket o‘ta oladi, qolgan portlar kutib turadi.
3. **Krossbar orqali (Crossbar Switch):** $N$ ta kirish va $N$ ta chiqish liniyalarining panjarasimon to‘ri. Bir vaqtning o‘zida bir nechta paketlar turli portlarga to‘qnashuvsiz parallel uzatiladi (eng zamonaviy usul).

---

### 4.3.3. Navbatlar va Head-of-the-Line (HOL) Blocking

Agar kommutatsiya matrisi tez ishlasa ham, chiquvchi kabel tezligi past bo‘lsa, paketlar chiquvchi port buferida navbatga turadi.

Ammo kiruvchi portda ham navbat hosil bo‘lishi mumkin:
* **HOL Blocking (Boshdagi paketning tiqilib qolishi):**
  * Tasavvur qiling, 1-kirish portida turgan birinchi paket band bo‘lgan 1-chiqish portiga ketmoqchi va kutib turibdi.
  * Uning orqasida turgan ikkinchi paket esa mutlaqo bo‘sh bo‘lgan 2-chiqish portiga ketmoqchi.
  * Lekin birinchi paket yo‘lni to‘sib turgani sababli, orqadagi paket bo‘sh portga o‘ta olmaydi! Bu hodisa HOL Blocking deyiladi.

---

## 4.4. IP PROTOKOLI: IPv4, MANZILLASH, NAT VA IPv6

Internet Protocol (IP) — butun Internetni ushlab turgan asosiy poydevordir.

### 4.4.1. IPv4 Datagram Formati (Kamida 20 bayt)

```
 0                   15 16                  31
+----+----+----------+--+---------------------+
|Ver |IHL |Type of Srv|   Total Length (bytes)|
+----+----+----------+--+---------------------+
|    Identification     |Flags|Fragment Offset|
+-----------------------+-----+---------------+
|Time to Live| Protocol |   Header Checksum   |
+------------+----------+---------------------+
|                Source IP Address            |
+---------------------------------------------+
|              Destination IP Address         |
+---------------------------------------------+
|             Options (0 - 40 bayt)           |
+---------------------------------------------+
|                    Data                     |
+---------------------------------------------+
```

* **Version (4 bit):** IP versiyasi (`4` yoki `6`).
* **Header Length (IHL — 4 bit):** Sarlavhaning 32-bitli so‘zlardagi uzunligi (Options bo‘lmasa, $5 \times 4 = 20 \text{ bayt}$).
* **Type of Service (ToS / DiffServ — 8 bit):** Xizmat sifati (QoS) — ovoz yoki video paketlarini oddiy paketlardan ustun qo‘yish uchun.
* **Total Length (16 bit):** Butun datagramning (sarlavha + ma’lumot) baytlardagi umumiy hajmi (maksimal 65 535 bayt).
* **Identification, Flags, Fragment Offset:** Katta paketlarni oraliq liniyalardan o‘tkazish uchun bo‘laklarga (fragmentlarga) ajratish maydonlari.
  * `Flags (3 bit)`: 1-bit zaxira, `DF (Don't Fragment)` — bo‘lishni taqiqlash, `MF (More Fragments)` — orqada yana bo‘laklar borligini bildirish.
* **Time to Live (TTL — 8 bit):** Har safar paket routerdan o‘tganda TTL 1 taga kamayadi. $TTL = 0$ bo‘lsa, paket yo‘q qilinadi va router xostga `ICMP Time Exceeded` qaytaradi.
* **Protocol (8 bit):** Paket ichidagi transport protokoli raqami (`6` — TCP, `17` — UDP, `1` — ICMP).
* **Header Checksum (16 bit):** Faqat IP sarlavhasining xatosizligini tekshiradi (har bir routerda TTL o‘zgargani uchun checksum qayta hisoblanadi).
* **Source va Destination IP:** 32-bitli jo‘natuvchi va qabul qiluvchi manzillari.

---

### 4.4.2. IP Fragmentatsiyasi va Qayta Yig‘ish Matematikasi

Har bir aloqa kanali o‘zining **MTU (Maximum Transmission Unit)** ko‘rsatkichiga ega (odatda Ethernet uchun 1500 bayt).

**Amaliy Masala:**
* Jo‘natuvchi xost **4000 baytlik** datagram yaratdi (20 bayt IP sarlavha + 3980 bayt ma’lumot).
* Yo‘ldagi kanalning MTU si **1500 bayt**.
* Router paketni qanday bo‘ladi?

```
ASL PAKET: Hajmi = 4000 bayt (20 bayt Header + 3980 bayt Data), ID = 777

1-FRAGMENT:
* Umumiy hajm: 1500 bayt (20 bayt Header + 1480 bayt Data)
* ID = 777, MF = 1 (yana bo'laklar bor)
* Offset = 0 / 8 = 0

2-FRAGMENT:
* Umumiy hajm: 1500 bayt (20 bayt Header + 1480 bayt Data)
* ID = 777, MF = 1
* Offset = 1480 / 8 = 185 (Offset 8 baytlik bloklarda o'lchanadi!)

3-FRAGMENT:
* Qolgan ma'lumot: 3980 - 1480 - 1480 = 1020 bayt.
* Umumiy hajm: 1040 bayt (20 bayt Header + 1020 bayt Data)
* ID = 777, MF = 0 (oxirgi bo'lak!)
* Offset = 2960 / 8 = 370
```

> **Oltin Qoida:** Oraliq routerlar bo‘lingan paketlarni qayta yig‘maydi! Fragmentlarni qayta yig‘ish faqat **eng oxirgi qabul qiluvchi xostda** bajariladi. Agar bitta bo‘lak yo‘lda yo‘qolsa, butun datagram tashlab yuboriladi va TCP butun boshli segmentni boshidan qayta uzatadi.

---

### 4.4.3. IPv4 Manzillash va Subnetting (CIDR Matematikasi)

Har bir IPv4 manzil **32 bit (4 bayt)** dan iborat bo‘lib, nuqta bilan ajratilgan o‘nlik sanoq tizimida yoziladi: `192.168.1.1`.

#### CIDR (Classless Interdomain Routing — Sinfsiz Marshrutlash)
Manzil formati: `a.b.c.d / x`
* `/x` — prefiks uzunligi (Subnet Mask dagi birlar soni).
* Dastlabki $x$ bit — **Tarmoq qismi (Network ID / Subnet Prefix)**.
* Qolgan $32 - x$ bit — **Xost qismi (Host ID)**.

```
Misol: 192.168.1.0/26 tarmog'i
Prefiks /26: Dastlabki 26 bit tarmoqqa tegishli.
Xost uchun bitlar soni: 32 - 26 = 6 bit.

Subnet Mask: 11111111.11111111.11111111.11000000 = 255.255.255.192
Jami IP lar soni: 2^6 = 64 ta.
Foydalanish mumkin bo'lgan xostlar: 2^6 - 2 = 62 ta.
* Birinchi manzil (192.168.1.0)  -> Tarmoq identifikatori (Network Address)
* Oxirgi manzil  (192.168.1.63) -> Umumiy efir manzili (Broadcast Address)
* Xostlar uchun: 192.168.1.1 dan 192.168.1.62 gacha.
```

#### Maxsus IP-Manzillar (RFC 1918 Xususiy IPlar):
Bu manzillar global Internetda marshrutlanmaydi, faqat ichki korporativ va uy tarmoqlarida ishlatiladi:
1. `10.0.0.0/8` (`10.0.0.0` dan `10.255.255.255` gacha — 16.7 million manzil).
2. `172.16.0.0/12` (`172.16.0.0` dan `172.31.255.255` gacha — 1 million manzil).
3. `192.168.0.0/16` (`192.168.0.0` dan `192.168.255.255` gacha — 65 536 manzil).
* `127.0.0.1`: Loopback (Localhost — o‘z-o‘ziga murojaat).
* `169.254.0.0/16`: APIPA (DHCP server topilmaganda kompyuter o‘ziga beradigan avtomatik manzil).

---

### 4.4.4. DHCP (Dynamic Host Configuration Protocol)

Yangi kompyuter yoqilganda tarmoq sozlamalarini avtomatik oluvchi protokol.

**DORA Bosqichlari (UDP Port 67 server, 68 mijoz):**

```
[Yangi Xost]                                            [DHCP Server]
      |                                                        |
      | 1. DHCP DISCOVER (src: 0.0.0.0, dst: 255.255.255.255)  |
      |------------------------------------------------------->|
      | "Men yangi keldim, menga IP kerak!"                    |
      |                                                        |
      | 2. DHCP OFFER (src: 192.168.1.1, dst: 255.255.255.255) |
      |<-------------------------------------------------------|
      | "Senga 192.168.1.15 IP sini taklif qilaman"            |
      |                                                        |
      | 3. DHCP REQUEST (src: 0.0.0.0, dst: 255.255.255.255)   |
      |------------------------------------------------------->|
      | "Roziman, 192.168.1.15 IP sini menga rasmiylashtiring" |
      |                                                        |
      | 4. DHCP ACK (src: 192.168.1.1, dst: 255.255.255.255)   |
      |<-------------------------------------------------------|
      | "Tasdiqlandi! Mask: /24, Gateway: .1, DNS: 8.8.8.8"    |
```

---

### 4.4.5. NAT (Network Address Translation) va PAT (Port Address Translation)

Dunyo bo‘yicha 4.3 milliardlik IPv4 tugab qolgani sababli har bir xonadonga bittadan ommaviy IP beriladi, uy ichida esa yuzlab qurilmalar xususiy IP (`192.168.1.x`) ishlatadi.

```
[ICHKI TARMOQ: 192.168.1.0/24]      [ROUTER (NAT)]        [INTERNET]
Kompyuter: 192.168.1.10:54321 -----> WAN IP: 213.230.1.5 -----> Veb-Server: 142.250.180.3:80
```

* **NAT Qanday Ishlaydi?**
  1. Ichki kompyuter paket chiqaradi: `src: 192.168.1.10:54321`, `dst: 142.250.180.3:80`.
  2. Router paketni ushlab oladi va uning `src IP` sini o‘zining tashqi ommaviy IP siga (`213.230.1.5`), `src port` ini esa yangi portga (`10001`) almashtiradi.
  3. Buni o‘zining **NAT Tarjima Jadvali (NAT Translation Table)** ga yozib qo‘yadi:
     $$\text{WAN tomon: } 213.230.1.5:10001 \longleftrightarrow \text{LAN tomon: } 192.168.1.10:54321$$
  4. Google serveridan javob `213.230.1.5:10001` ga kelganda, router jadvalga qarab paketni ichkaridagi `192.168.1.10:54321` ga o‘zgartirib uzatadi!

---

### 4.4.6. ICMP (Internet Control Message Protocol)

Xostlar va routerlar o‘rtasida tarmoq xatoliklarini xabarlash protokoli:
* `Type 8, Code 0`: Echo Request (Ping so‘rovi).
* `Type 0, Code 0`: Echo Reply (Ping javobi).
* `Type 11, Code 0`: TTL Expired in Transit (Traceroute ishlash asosi).
* `Type 3, Code 3`: Destination Port Unreachable.

---

### 4.4.7. IPv6 (Kelajak Arxitekturasi)

IPv4 kamchiliklarini bartaraf etish uchun yaratilgan:
* **Manzil hajmi:** **128 bit (16 bayt)**. Format: 8 ta 4-xonali o‘n oltilik bloklar (`2001:0db8:85a3:0000:0000:8a2e:0370:7334`).
* **Sarlavha qat’iy 40 bayt:** Qidiruv tezligini oshirish uchun o‘zgarmas o‘lchamda qilingan.
* **Router fragmentatsiyasi olib tashlandi:** Faqat jo‘natuvchi xost Path MTU Discovery orqali fragmentlaydi.
* **Checksum olib tashlandi:** Routerlar ortiqcha vaqt yo‘qotmasligi uchun.
* **IPv4 dan IPv6 ga o‘tish:**
  * *Dual-Stack:* Qurilmalar bir vaqtning o‘zida ikkala IP bilan ishlaydi.
  * *Tunneling (6to4):* IPv6 paketlari eski IPv4 routerlar orqali o‘tishi uchun IPv4 paketlarining ichiga o‘rab (encapsulate qilib) jo‘natiladi.

---

## 4.5. MARSHRUTLASH ALGORITMLARI (ROUTING ALGORITHMS)

Tarmoq — bu tugunlar $N$ (routerlar) va ularni bog‘lovchi qirralar $E$ (aloqa liniyalari) dan iborat matematik **Graf $G = (N, E)$** dir. Har bir liniya o‘zining **Narxi ($c(x,y)$ — Cost)** ga ega (masalan, kechikish, masofa yoki pul).

```
        (2)
    u ------- v
   / |       / |
(1)/ |(2) (1)/ |(3)
  /  |   /   |
 x --+--y    |
  \  | /     |
(1)\ |/(1)   |(1)
    w ------- z
        (2)
```

---

### 4.5.1. Link-State (LS) Algoritmi: Deykstra (Dijkstra)

* **Tamoyili:** Har bir router butun tarmoq xaritasini to‘liq biladi (Link-State Broadcast orqali hamma routerlar bir-biriga qo‘shnilarining narxini e’lon qiladi).
* Deykstra algoritmi bitta manba tugundan qolgan barcha tugunlargacha bo‘lgan eng qisqa yo‘llar daraxtini (Shortest Path Tree) hisoblab chiqadi.
* **Algoritm qadamlari:**
  * $N'$ — eng qisqa yo‘li allaqachon aniqlangan tugunlar to‘plami.
  * $D(v)$ — manbadan $v$ tugungacha bo‘lgan joriy eng arzon narx.
  * $p(v)$ — $v$ ning yo‘ldagi bevosita oldingi qo‘shnisi.
  1. *Boshlanishi:* $N' = \{u\}$, qo‘shnilar uchun $D(v) = c(u,v)$, boshqalar uchun $D(w) = \infty$.
  2. *Qadam:* $N'$ da bo‘lmagan tugunlar ichidan $D(w)$ eng kichik bo‘lgan $w$ tugun tanlanadi va $N'$ ga qo‘shiladi.
  3. Barcha qo‘shnilar uchun narx yangilanadi:
     $$D(v) = \min(D(v), D(w) + c(w,v))$$
  4. Barcha tugunlar $N'$ ga kirmaguncha qadam takrorlanadi.
* Murakkabligi: $O(n^2)$ yoki min-heap bilan $O(n \log n)$.
* **Protokol:** OSPF (Open Shortest Path First).

---

### 4.5.2. Distance-Vector (DV) Algoritmi: Bellman-Ford

* **Tamoyili:** Taqsimlangan, asinxron va iterativ algoritm. Hech bir router butun xaritani bilmaydi!
* Har bir router faqat o‘zining qo‘shnilariga: *"Mendan boshqa barcha manzillargacha bo‘lgan masofa vektorlari mana shulardan iborat"* deb vaqti-vaqti bilan xabar berib turadi.
* **Bellman-Ford Tenglamasi:**
  $$d_x(y) = \min_v \{ c(x,v) + d_v(y) \}$$
  *(ya’ni $x$ dan $y$ gacha borish — barcha qo‘shni $v$ larning biriga o‘tish va $v$ dan $y$ gacha borish yig‘indisining eng kichigidir).*

#### Count-to-Infinity Muammosi va Poisoned Reverse:
Agar tarmoqda liniya uzilib qolsa, routerlar bir-biriga eski ma’lumotlarni berib, xabar narxini $1, 2, 3... 16$ qilib cheksizlikka qarab oshirib yuborishi mumkin (Routing Loop).
* **Poisoned Reverse yechimi:** Agar $Z$ router $X$ ga borish uchun $Y$ orqali marshrut tanlagan bo‘lsa, $Z$ o‘zining $Y$ qo‘shnisiga: *"Mendan $X$ gacha bo‘lgan masofa cheksiz ($d_Z(X) = \infty$)"* deb yolg‘on xabar beradi. Natijada $Y$ hech qachon $Z$ orqali $X$ ga borishga urinmaydi va halqa hosil bo‘lmaydi!

---

## 4.6. INTERNETDAGI MARSHRUTLASH PROTOKOLLARI (OSPF VA BGP)

Butun dunyo tarmoqlari **Avtonom Tizimlar (Autonomous Systems — AS)** ga bo‘lingan. AS — bu bitta ma’muriy tashkilot nazoratidagi routerlar hududidir.

```
+---------------------------+       +---------------------------+
|    AS 100 (Uztelecom)     |       |       AS 200 (Beeline)    |
|                           |       |                           |
|   [ Router A ]            |       |            [ Router C ]   |
|         \ (OSPF / IGP)    |       |            / (OSPF / IGP) |
|          v                |       |           v               |
|      [ Gateway R1 ] <==== BGP (Inter-AS / EGP) ====> [ Gateway R2 ]
+---------------------------+       +---------------------------+
```

1. **Intra-AS Routing (AS Ichidagi Marshrutlash — IGP):**
   * Bitta tashkilot ichidagi marshrutlash. Bosh maqsad — eng yuqori unumdorlik va eng tezkor yo‘l.
   * **OSPF (RFC 2328):** Link-State ga asoslangan, xavfsiz (MD5 shifrlash), ierarxik hududlarga (Areas) bo‘linuvchi zamonaviy korporativ protokol.
2. **Inter-AS Routing (AS lar O‘rtasidagi Marshrutlash — BGP):**
   * **BGP (Border Gateway Protocol — RFC 4271):** Butun global Internetni bir-biriga bog‘lab turuvchi eng buyuk protokol!
   * BGP da eng qisqa yo‘l emas, balki **Siyosat (Policy-Based Routing)** birinchi o‘rinda turadi (masalan, davlatlararo shartnomalar, pullik tranzit qoidalari).
   * **BGP Atributlari:**
     * `AS-PATH`: Paket nishonga yetib borguncha qaysi AS raqamlaridan o‘tganini ko‘rsatadi (masalan, `AS 100 -> AS 209 -> AS 15169`). Agar router o‘z AS raqamini ko‘rsa, marshrutni rad etadi (halqalar yo‘qoladi).
     * `NEXT-HOP`: Keyingi sakrashdagi chegara routerining aniq IP-manzili.

---

## 4.7. 4-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. **Tarmoq qatlami** mahalliy **Forwarding** (LPM bo‘yicha paket surish) va global **Routing** (yo‘l hisoblash) vazifalarini bajaradi.
2. **IPv4** datagramlari 20 bayt sarlavhaga ega bo‘lib, kerak bo‘lganda fragmentlanadi. IPv4 yetishmovchiligi **CIDR Subnetting** va **NAT/PAT** texnologiyalari orqali hal qilingan. Kelajak to‘liq **IPv6** ga tegishli.
3. Global Internet routerlari ichkarida **OSPF**, global miqyosda esa **BGP** protokoli orqali marshrutlanadi.

### 🔗 Joriy papkadagi laboratoriyalar bilan bog‘liqlik:
* Ushbu bobdagi **IPv4 Subnetting, L3 Routing va NAT/PAT mexanizmi** to‘g‘ridan-to‘g‘ri **`A4_project`** (Stateful nftables siyosati, LAN uchun SNAT va DMZ uchun DNAT/Port-Forwarding) laboratoriyasining 100% negizidir!
* Routerning ichki tuzilishi va xavfsizligi esa **`A16_project`** (Tarmoq qurilmalarini hardening qilish va audit o‘tkazish) mashg‘ulotida virtual routerni xavfsiz himoyalash uchun talab etiladi!
