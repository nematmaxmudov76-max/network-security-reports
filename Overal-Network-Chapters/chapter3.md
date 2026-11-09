# 3-BOB: TRANSPORT QATLAMI (Transport Layer)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Transport qatlamining fundamental xizmatlari (multipleksirlash va demultipleksirlash), ulanishsiz UDP protokoli va nazorat summasi matematikasi, ishonchli ma’lumot uzatish (rdt 1.0 dan rdt 3.0 gacha evolyutsiyasi, Go-Back-N va Selective Repeat), TCP ning to‘liq arxitekturasi (segment strukturasi, 3-Way Handshake, oqim nazorati, RTT va timeout hisoblash) hamda tarmoq tiqilinch nazorati (TCP Congestion Control — AIMD, Slow Start, Tahoe vs Reno) algoritmlarini chuqur texnik darajada o‘rganish.

---

## 3.1. TRANSPORT QATLAMI XIZMATLARI VA MAZMUNI

Agar quyi qatlamlar (Tarmoq, Kanal, Fizik) paketlarni apparat vositalari orqali xostdan xostga ko‘chirib yurish bilan shug‘ullansa, **Transport qatlami** turli kompyuterlarda ishlayotgan **jarayonlar (dasturlar)** o‘rtasida **mantiqiy aloqa (Logical Communication)** o‘rnatadi.

```
+-------------------------------------------------------------------------+
| [XOST A]                                                       [XOST B] |
| Jarayon 1 (Brauzer)   Jarayon 2 (SSH)                Jarayon 1 (Veb-srv)|
|         \                /                                      ^       |
|          v              v                                       |       |
|      +---------------------+                         +----------------+ |
|      |  TRANSPORT QATLAMI  |                         | TRANSPORT QATL.| |
|      |    (TCP / UDP)      | <=====================> |   (TCP / UDP)  | |
|      +---------------------+   Mantiqiy aloqa kanali +----------------+ |
|                 |                                               |       |
|                 v                                               v       |
|      +---------------------+                         +----------------+ |
|      |   TARMOQ QATLAMI    |                         | TARMOQ QATLAMI | |
|      |        (IP)         |                         |      (IP)      | |
|      +---------------------+                         +----------------+ |
+-------------------------------------------------------------------------+
```

---

### 3.1.1. Transport va Tarmoq Qatlamlari Farqi: Uy va Pochta Analogiyasi

Kitobdagi afsonaviy o‘xshatishni keltiramiz:
* Tasavvur qiling, Toshkentdagi xonadonda 12 nafar bola yashaydi, Samarqanddagi xonadonda ham 12 nafar bola yashaydi.
* Bolalar o‘zaro muntazam xat yozishadi (masalan, Toshkentdagi Alisher Samarqanddagi Boburga).
* Toshkentdagi uyda katta aka **Sardor**, Samarqanddagi uyda katta opa **Zarina** xatlarni yig‘ib, pochta qutisiga eltish va pochtachi olib kelgan xatlarni aynan qaysi bolaga tegishli ekanini tekshirib tarqatish uchun mas’uldirlar.
* **O‘zbekiston Pochtasi (Pochta mashinalari va yo‘llar):** Bu — **Tarmoq qatlami (IP)**. U xatni Toshkentdagi xonadondan Samarqanddagi xonadonga (xostdan xostga) yetkazadi.
* **Sardor va Zarina:** Bu — **Transport qatlami (TCP/UDP)**. Ular xatni xonadon ichidagi aynan qaysi bolaga (qaysi jarayonga/portga) tegishli ekanini saralaydi!

> **Asosiy qoida:**  
> * Tarmoq qatlami (IP) faqat **xostlar** o‘rtasida aloqa o‘rnatadi.  
> * Transport qatlami esa xostlar ichidagi **jarayonlar** o‘rtasida aloqa o‘rnatadi!

---

## 3.2. MULTIPLEKSIRLASH VA DEMULTIPLEKSIRLASH MEXANIZMI

Bitta kompyuter ichida bir vaqtning o‘zida o‘nlab tarmoq dasturlari ishlaydi. Tarmoq kartasidan bitta kabel orqali kelayotgan yuzlab paketlar qaysi dasturga tegishli ekanini transport qatlami qanday aniqlaydi?

```
[Mijoz A: Brauzer]                 [Server B: Veb-server (Nginx)]
Port: 52341 (Tasodifiy)            Port: 80 (Standart HTTP)
         |                                  ^
         v                                  |
[Multipleksirlash]                 [Demultipleksirlash]
Header qo'shiladi:                 Header o'qiladi:
Src Port: 52341                    "Bu port 80 ga ketyapti,
Dest Port: 80                       demak Nginx ga beraman!"
```

* **Multipleksirlash (Multiplexing — Jo‘natuvchida):** Har xil soketlardan kelgan ma’lumotlarni yig‘ib, ularga transport sarlavhasini (manba va maqsad portlarini) qo‘shish va quyi qatlamga (IP ga) topshirish.
* **Demultipleksirlash (Demultiplexing — Qabul qiluvchida):** Kelgan segment sarlavhasidagi port raqamlarini o‘qib, uni to‘g‘ri soketga yetkazib berish.

---

### 3.2.1. Ulanishsiz Demultipleksirlash (UDP)

UDP soketlari **2-lik kortej** bilan aniqlanadi:
$$\{ \text{Destination IP Address}, \text{Destination Port Number} \}$$

* Agar ikkita har xil kompyuterdan (yoki bitta kompyuterning turli dasturlaridan) bitta maqsad IP va portga UDP paket kelsa, server ularni **bitta umumiy soket orqali** qabul qilaveradi. Jo‘natuvchining IP si faqat javob qaytarish uchun kerak bo‘ladi.

---

### 3.2.2. Ulanishga Yo‘naltirilgan Demultipleksirlash (TCP)

TCP soketlari qat’iy **4-lik kortej** bilan aniqlanadi:
$$\{ \text{Source IP}, \text{Source Port}, \text{Destination IP}, \text{Destination Port} \}$$

```
[Mijoz A (192.168.1.5:10001)] -----\
                                    =====> [ Veb-Server (142.250.180.3:80) ]
[Mijoz B (192.168.1.6:20002)] -----/      Serverda 2 ta ALOHIDA soket ochiladi:
                                          Soket 1: {192.168.1.5:10001 <-> :80}
                                          Soket 2: {192.168.1.6:20002 <-> :80}
```

* Agar bitta veb-serverga bir vaqtda 10 000 ta mijoz bog‘lansa, server xotirasida 10 000 ta mustaqil bog‘langan soket ochiladi. Qabul qiluvchi operatsion tizim kelgan paketning barcha 4 ta parametrini tekshirib, uni aynan o‘sha mijoz uchun ochilgan maxsus soketga yo‘naltiradi!

---

## 3.3. ULANISHSIZ TRANSPORT: UDP (USER DATAGRAM PROTOCOL)

UDP (RFC 768) — bu deyarli minimal qo‘shimcha yukka ega bo‘lgan transport protokoli. U IP protokoliga faqat **portlar** va **nazorat summasi (checksum)** nina qo‘shadi, xolos.

### Nega dasturchilar ishonchsiz UDP ni tanlaydi?
1. **Ulanish o‘rnatishda kechikish yo‘q:** 3-Way Handshake yo‘q, birinchi millisekunddanoq ma’lumot yo‘lga chiqadi (DNS shuning uchun tez).
2. **Ulanish holati saqlanmaydi:** Server xotirasida buferlar, ketma-ketlik raqamlari saqlanmaydi. Bitta server millionlab UDP mijozlariga bemalol xizmat qila oladi.
3. **Kichik sarlavha:** TCP sarlavhasi kamida 20 bayt, UDP sarlavhasi esa atigi **8 bayt**!
4. **Tiqilinch nazorati yo‘q:** UDP ilovasi xohlagan tezlikda tarmoqqa ma’lumot purkayveradi (real vaqt ovoz/video uzatish uchun juda muhim, chunki TCP tiqilinchda tezlikni tushirib yuboradi).

---

### 3.3.1. UDP Segment Formati va Checksum Matematikasi

UDP sarlavhasi atigi 4 ta maydondan (har biri 2 baytdan, jami 8 bayt) iborat:

```
 0                   15 16                  31
+----------------------+----------------------+
|  Source Port (16 bit)|  Dest Port (16 bit)  |
+----------------------+----------------------+
|  Length (16 bit)     |  Checksum (16 bit)   |
+----------------------+----------------------+
|             Ilova Ma'lumotlari (Payload)    |
+---------------------------------------------+
```

#### UDP Checksum Qanday Hisoblanadi? (1's Complement Arithmetic)
Checksum segmentdagi bitlar yo‘lda (shovqin tufayli) o‘zgarmaganini tekshiradi:
1. Segmentdagi barcha 16-bitli so‘zlar ketma-ket qo‘shiladi.
2. Agar yig‘indida eng chapdagi (17-chi) ko‘chish biti (carry bit) hosil bo‘lsa, u oxiriga olib kelib qo‘shiladi (**End-around carry / Wraparound**).
3. Hosil bo‘lgan 16-bitli sonning teskarisi (barcha 0 lar 1 ga, 1 lar 0 ga aylantirilib — **1's complement**) olinadi va Checksum maydoniga yoziladi.
4. **Tekshirish:** Qabul qiluvchi tomon barcha so‘zlarni va checksum ning o‘zini qo‘shib chiqadi. Agar barcha bitlar faqat `1111111111111111` chiqsa — paket toza! Agar bitta bo‘lsa ham `0` chiqsa — segmentda xatolik bor va u tashlab yuboriladi!

---

## 3.4. ISHONCHLI MA’LUMOT UZATISH TAMOYILLARI (RDT EVOLYUTSIYASI)

Kurose & Ross kitobining eng buyuk metodik kashfiyoti — bu **Ishonchsiz kanal ustida Ishonchli aloqa protokoli (rdt — Reliable Data Transfer)** ni qadamma-qadam loyihalashdir:

```
+-------------------------------------------------------------------------+
| rdt 1.0 : Ideal kanal (Xato yo'q, yo'qotish yo'q).                      |
|    |                                                                    |
|    v                                                                    |
| rdt 2.0 : Bit xatolari bor kanal -> ACK / NAK (Stop-and-Wait).          |
|    |                                                                    |
|    v                                                                    |
| rdt 2.1 : ACK/NAK buzilishi -> Sequence Number (0 va 1).                |
|    |                                                                    |
|    v                                                                    |
| rdt 2.2 : NAK dan voz kechish -> Duplicate ACK.                         |
|    |                                                                    |
|    v                                                                    |
| rdt 3.0 : Paketlar yo'qoladigan kanal -> Countdown Timer va Retransmit. |
+-------------------------------------------------------------------------+
```

---

### 1. rdt 2.0: Bit Xatolari Bor Kanal (Stop-and-Wait)
Kanalda bitlar buzilishi mumkin, lekin paketlar yo‘qolmaydi:
* **ACK (Positive Acknowledgment):** Qabul qiluvchi: *"Paketni to‘g‘ri oldim"*.
* **NAK (Negative Acknowledgment):** Qabul qiluvchi: *"Paketda xato bor, qaytadan yubor"*.
* Bu usul **Stop-and-Wait (To‘xta va Kut)** deyiladi — jo‘natuvchi javob kelguncha yangi ma’lumot yubormay kutib turadi.

### 2. rdt 2.1 va 2.2: ACK yoki NAK ning O‘zi Buzilsachi?
Agar qabul qiluvchi yuborgan ACK yo‘lda buzilsa, jo‘natuvchi nima qilishni bilmay qoladi. Agar qaytadan yuborsa, qabul qiluvchi uni yangi paket deb o‘ylab, faylni ikkita nusxa qilib qo‘yadi (Dublikat muammosi)!
* **Yechim (rdt 2.1):** Har bir paketga **Tartib raqami (Sequence Number)** qo‘shiladi. Stop-and-Wait uchun atigi 1 bit (**0 va 1**) yetarli!
* **rdt 2.2 (NAK-siz protokol):** NAK xabari butunlay olib tashlanadi. Buning o‘rniga qabul qiluvchi oxirgi to‘g‘ri olgan paketining raqami bilan ACK qaytaradi. Agar jo‘natuvchi bitta raqamli ACK ni ikki marta olsa (Duplicate ACK), u keyingi paket buzilganini tushunadi.

### 3. rdt 3.0: Paketlar Yo‘lda Yo‘qolib Qolsachi?
Agar paket yoki ACK yo‘lda butunlay yo‘qolib ketsa, tizim abadiy qotib qoladi (Deadlock).
* **Yechim:** Jo‘natuvchiga **Ortga hisoblash taymeri (Countdown Timer / Timeout)** qo‘shiladi.
* Agar jo‘natuvchi belgilangan vaqt ichida ACK ololmasa, u paket yo‘qolgan deb hisoblaydi va uni **qaytadan uzatadi (Retransmission)**!

---

### 3.4.1. Konveyerli Protokollar: Go-Back-N (GBN) vs Selective Repeat (SR)

Stop-and-Wait protokollarida liniyaning unumdorligi o‘ta past bo‘ladi.
* Misol: 1 Gbps lik kanal orqali AQShdan Yevropaga (RTT = 30 ms) 1000 baytlik paket yuborilsa, kanalning unumdorligi atigi **0.027%** bo‘ladi! Qolgan 99.97% vaqtda kanal bekor kutib turadi.
* **Yechim: Pipelining (Konveyer):** Javobini kutmasdan birdaniga bir nechta ($N$ ta) paketlarni tarmoqqa chiqarish.

```
GO-BACK-N (GBN)                                SELECTIVE REPEAT (SR)
[0][1][2][3][4] (Oyna = 4 ta)                  [0][1][2][3][4] (Oyna = 4 ta)
Paket 2 yo'qoldi!                              Paket 2 yo'qoldi!
Qabul qiluvchi: Faqat ACK 1 qaytaradi          Qabul qiluvchi: 3 va 4 ni buferga saqlaydi
(2, 3, 4 ni tashlab yuboradi).                 va ACK 3, ACK 4 qaytaradi.
Jo'natuvchi timeout bo'lgach:                  Jo'natuvchi timeout bo'lgach:
2, 3 va 4-paketlarni QAYTADAN yuboradi!        FAQAT 2-paketning o'zini qayta yuboradi!
```

| Xususiyati | Go-Back-N (GBN) | Selective Repeat (SR) |
| :--- | :--- | :--- |
| **ACK turi** | Kumulyativ (Cumulative ACK: $n$ kelsa, $n$ gacha hammasi kelgan) | Individual (Har bir paket uchun alohida ACK) |
| **Qabul qiluvchi buferi** | Yo‘q (Tartibsiz kelgan paketlar tashlab yuboriladi) | Bor (Tartibsiz kelgan paketlar saqlanadi) |
| **Qayta uzatish** | Bitta paket yo‘qolsa, butun oyna qayta yuboriladi | Faqat yo‘qolgan bitta paket qayta yuboriladi |
| **Murakkabligi** | Sodda, xotira talab qilmaydi | Murakkab, har bir paket uchun alohida taymer |

---

## 3.5. ULANISHGA YO‘NALTIRILGAN TRANSPORT: TCP (TRANSMISSION CONTROL PROTOCOL)

TCP (RFC 793, 1122, 1323, 2018, 2581) — bu Internetning eng asosiy, eng murakkab va ishonchli protokol arxitekturasidir.
* **Point-to-Point:** Bir vaqtda faqat bitta jo‘natuvchi va bitta qabul qiluvchi o‘rtasida ishlaydi (Multicast yoki Broadcast qila olmaydi).
* **Reliable, In-Order Byte Stream:** Ma’lumotlar alohida xabarlar sifatida emas, balki yaxlit uzluksiz tartiblangan baytlar oqimi sifatida uzatiladi.
* **Full-Duplex:** Bitta TCP ulanishi orqali bir vaqtning o‘zida ikkala tomonga ham ma’lumot oqishi mumkin.
* **MSS (Maximum Segment Size):** Bitta TCP segmentiga sig‘adigan maksimal ma’lumot hajmi (odatda Ethernet MTU 1500 bayt bo‘lsa, IP va TCP sarlavhalari ayirilib, $MSS = 1460 \text{ bayt}$ bo‘ladi).

---

### 3.5.1. TCP Segmentining Strukturasi (20 baytlik sarlavha)

```
 0                   15 16                  31
+----------------------+----------------------+
|  Source Port (16 bit)|  Dest Port (16 bit)  |
+----------------------+----------------------+
|              Sequence Number (32 bit)       |
+---------------------------------------------+
|           Acknowledgment Number (32 bit)    |
+------+-----------+--+-+-+-+-+-+-------------+
|Header| Reserved  |U|A|P|R|S|F| Receive Window|
|Length|  (4 bit)  |R|C|S|S|Y|I|   (16 bit)   |
| (4b) |           |G|K|H|T|N|N|              |
+------+-----------+--+-+-+-+-+-+-------------+
|   Checksum (16 bit)  | Urgent Pointer(16bit)|
+----------------------+----------------------+
|             Options (0 - 40 bayt)           |
+---------------------------------------------+
|                 Data (Payload)              |
+---------------------------------------------+
```

* **Sequence Number (Ketma-ketlik raqami — 32 bit):** Segmentdagi birinchi ma’lumot baytining umumiy baytlar oqimidagi tartib raqami. (Masalan, 500 000 baytlik fayl yuborilayotgan bo‘lsa va $MSS=1000$ bo‘lsa, 1-segment seq=0, 2-segment seq=1000, 3-segment seq=2000 bo‘ladi).
* **Acknowledgment Number (Tasdiq raqami — 32 bit):** Qabul qiluvchi tomon **kutayotgan keyingi baytning tartib raqami**! TCP kumulyativ tasdiqdan foydalanadi.
* **Bayroqlar (Flags — 6 bit):**
  * `SYN`: Ulanishni ochish so‘rovi (faqat 3-Way Handshake paytida 1 bo‘ladi).
  * `ACK`: Acknowledgment maydonidagi raqam haqiqiy ekanini tasdiqlaydi.
  * `FIN`: Ulanishni yopish so‘rovi.
  * `RST`: Ulanishni majburiy to‘xtatish / uzish (Reset).
  * `PSH`: Ma’lumotni buferda ushlab turmasdan darhol ilovaga topshirish.
  * `URG`: Shoshilinch ma’lumot mavjudligi.
* **Receive Window (`rwnd` — 16 bit):** Qabul qiluvchi tomonning buferida qancha bo‘sh joy qolganini bildiradi (Oqim nazorati uchun).

---

### 3.5.2. RTT Baholash va Timeout Taymerini Hisoblash Matematikasi

TCP yo‘qolgan paketni qayta yuborishi uchun Timeout taymeri qancha vaqtga qo‘yilishi kerak?
* Agar timeout o‘ta qisqa qo‘yilsa: Paket yo‘lda sekin harakatlansa ham, asossiz qayta uzatishlar (Duplicate packets) ko‘payib ketadi.
* Agar timeout o‘ta uzun qo‘yilsa: Paket yo‘qolganda tizim uzoq kutib qoladi va tarmoq tezligi tushib ketadi.
* **Yechim:** TCP tarmoqdagi RTT (Round-Trip Time) ni dinamik o‘lchab, taymerni doimiy yangilab turadi:

1. **SampleRTT:** Bitta segment yuborilganidan unga ACK kelguncha o‘tgan real vaqt.
2. **EstimatedRTT (Eksponentsial o‘rtacha qiymat — EWMA):**
   $$\text{EstimatedRTT} = (1 - \alpha) \times \text{EstimatedRTT} + \alpha \times \text{SampleRTT}$$
   *(Standart tavsiya: $\alpha = 0.125$)*
3. **DevRTT (Varyatsiya / Og‘ish darajasi):**
   $$\text{DevRTT} = (1 - \beta) \times \text{DevRTT} + \beta \times |\text{SampleRTT} - \text{EstimatedRTT}|$$
   *(Standart tavsiya: $\beta = 0.25$)*
4. **Timeout Taymeri (TimeoutInterval):**
   $$\text{TimeoutInterval} = \text{EstimatedRTT} + 4 \times \text{DevRTT}$$

---

### 3.5.3. Tezkor Qayta Uzatish (Fast Retransmit)

Taymerning tugashini (Timeout) kutish — bu sekin jarayon. TCP da paket yo‘qolganini tezda aniqlash mexanizmi bor:
* Agar qabul qiluvchiga paketlar tartibsiz kelsa (masalan, 1, 2 keldi, 3 yo‘qoldi, 4 va 5 keldi), qabul qiluvchi har safar kutayotgan baytini so‘rab **Dublikat ACK (Duplicate ACK 3)** yuboradi.
* **Fast Retransmit qoidasi:**  
  Agar jo‘natuvchi bitta paket uchun ketma-ket **3 ta bir xil Dublikat ACK (Triple Duplicate ACKs — jami 4 ta bir xil ACK)** olsa, u taymerning tugashini kutmasdan, **o‘sha yo‘qolgan paketni darhol qayta yuboradi**!

---

### 3.5.4. Oqim Nazorati (Flow Control)

Agar server sekin ishlasa (CPU band bo‘lsa), lekin jo‘natuvchi mijoz gigabit tezlikda paket yuboraversa, serverning qabul qilish buferi to‘lib, barcha ma’lumotlar tashlab yuboriladi.
* **Flow Control** jo‘natuvchi tezligini qabul qiluvchining o‘qish tezligi bilan tenglashtiradi.
* Qabul qiluvchi o‘z buferidagi bo‘sh joyni hisoblab turadi:
  $$\text{rwnd} = \text{RcvBuffer} - [\text{LastByteRcvd} - \text{LastByteRead}]$$
* Qabul qiluvchi har bir ACK segmentining `Receive Window` maydoniga ushbu `rwnd` qiymatini yozib yuboradi.
* Jo‘natuvchi qat’iy qoidaga bo‘ysunadi:
  $$\text{LastByteSent} - \text{LastByteAcked} \le \text{rwnd}$$
* Agar `rwnd = 0` bo‘lsa, jo‘natuvchi ma’lumot yuborishni to‘xtatadi (lekin server buferi bo‘shaganini bilish uchun vaqti-vaqti bilan 1 baytlik tekshiruv segmenti yuborib turadi).

---

### 3.5.5. TCP Ulanishini Boshqarish: 3-Way Handshake va 4-Way Teardown

#### 1. Ulanishni o‘rnatish (3-Way Handshake):
```
[Mijoz (Client)]                                                  [Server]
       |                                                             |  CLOSED
       |  1. SYN: [seq = client_isn, ACK=0, SYN=1]                   |  LISTEN
       |------------------------------------------------------------>|  SYN_RCVD
       |  (Mijoz SYN_SENT holatiga o'tadi)                           |
       |                                                             |
       |  2. SYN-ACK: [seq = server_isn, ack = client_isn+1,         |
       |               ACK=1, SYN=1]                                 |
       |<------------------------------------------------------------|
       |  (Mijoz ESTABLISHED holatiga o'tadi)                        |
       |                                                             |
       |  3. ACK: [seq = client_isn+1, ack = server_isn+1,           |
       |           ACK=1, SYN=0] (Ilova ma'lumoti bo'lishi mumkin)   |
       |------------------------------------------------------------>|  ESTABLISHED
```

> **Xavfsizlik tahdidi — SYN Flood:**  
> Xaker soxta IP lardan minglab SYN paketlarini yuboradi, lekin oxirgi ACK ni qaytarmaydi. Server xotirasi yarim ochiq ulanishlar (SYN_RCVD) bilan to‘lib qoladi va qonuniy foydalanuvchilarni qabul qilolmay qoladi.  
> **Himoya:** **SYN Cookies** texnologiyasi (server ulanish to‘liq ochilguncha xotira ajratmaydi, parametrni xesh qilib sequence raqamiga yashiradi).

#### 2. Ulanishni yopish (4-Way Teardown):
1. Mijoz `FIN=1` bayrog‘ini yuboradi $\implies$ Server `ACK` qaytaradi.
2. Server o‘zida qolgan ma’lumotlarni jo‘natib bo‘lgach, o‘zining `FIN=1` bayrog‘ini yuboradi $\implies$ Mijoz `ACK` qaytaradi.
3. Mijoz `TIME_WAIT` holatida **2 MSL (Maximum Segment Lifetime — taxminan 60–120 soniya)** kutadi. Nega? Agar mijoz yuborgan oxirgi ACK yo‘lda yo‘qolib qolsa, server FIN ni qayta yuboradi, mijoz uni qabul qilib yana ACK qaytara olishi kerak!

---

## 3.6. TCP TIQILINCH NAZORATI (CONGESTION CONTROL ALGORITMI)

Agar tarmoqdagi barcha kompyuterlar bir vaqtda maksimal tezlikda ma’lumot yuborsa, oraliq routerlarning buferlari to‘lib ketadi, paketlar ommaviy yo‘qoladi va butun Internet qotib qoladi (**Congestion Collapse**).

TCP buni bartaraf etish uchun o‘zining **Tiqilinch Oynasi (`cwnd` — Congestion Window)** o‘lchamini tarmoq holatiga qarab avtomatik boshqaradi:
$$\text{Tarmoqqa yuboriladigan hajm} \le \min(\text{cwnd}, \text{rwnd})$$

---

### 3.6.1. AIMD Mexanizmi (Additive-Increase, Multiplicative-Decrease)

```
cwnd (MSS)
  ^
  |               /\                /\
  |              /  \              /  \
  |             /    \            /    \
  |       /\   /      \    /\    /      \     <-- Arraga o'xshash profil (Sawtooth)
  |      /  \ /        \  /  \  /        \
  +---------------------------------------------> Vaqt
```

* **Additive-Increase (Chiziqli oshirish):** Har safar barcha yuborilgan paketlar muvaffaqiyatli yetib borsa, har bir RTT da $cwnd$ ehtiyotkorlik bilan **1 ta MSS ga oshiriladi** (tarmoqda bo‘sh joy qidiriladi).
* **Multiplicative-Decrease (Karrali kamaytirish):** Paket yo‘qolishi (Loss) sezilishi bilanoq, $cwnd$ darhol **2 barobarga (50% ga) qisqartiriladi**!

---

### 3.6.2. TCP ning 3 ta Asosiy Fazasi

```
cwnd (MSS)
  ^
  |                                  (Loss yuz berdi: 3 ta Dup ACK)
32|                                         *
  |                                        / \
  |                                       /   \  Fast Recovery
16|               * (ssthresh)           /     \--------
  |              / \                    /
 8|             /   \ Congestion Avoid /
 4|            /     ------------------ (ssthresh = 16 ga tushdi)
 2|           / Slow
 1|----------/  Start
  +----------------------------------------------------> Vaqt
```

1. **Sekin Boshlash (Slow Start):**
   * Ulanish ochilganda $cwnd = 1 \text{ MSS}$ bo‘ladi.
   * Har bir kelgan ACK uchun $cwnd$ 1 taga oshadi. Ya’ni har bir RTT da oyna **2 barobarga (eksponentsial)** o‘sadi: $1 \to 2 \to 4 \to 8 \to 16...$
   * Oyna belgilangan bo‘sag‘aga (**ssthresh — Slow Start Threshold**) yetganda, sekin start to‘xtaydi va tizim xavfsiz rejimga o‘tadi.
2. **Tiqilinchdan Qochish (Congestion Avoidance):**
   * Bu fazada oyna eksponentsial emas, balki har bir RTT da atigi **1 ta MSS ga chiziqli** o‘sadi.
3. **Tezkor Tiklanish (Fast Recovery / TCP Reno):**
   * Agar paket yo‘qolishi **3 ta Dublikat ACK** orqali aniqlansa (yengil tiqilinch):
     * Yangi bo‘sag‘a belgilanadi: $ssthresh = cwnd / 2$.
     * $cwnd = ssthresh + 3$ qilinadi va oyna yana chiziqli o‘sishda davom etadi.
   * Agar paket yo‘qolishi **Timeout (taymer tugashi)** orqali yuz bersa (og‘ir tiqilinch):
     * $ssthresh = cwnd / 2$ qilinadi.
     * $cwnd = 1 \text{ MSS}$ ga tushirib yuboriladi va tizim yana eng boshidan (Slow Start) boshlaydi (**TCP Tahoe usuli**).

---

## 3.7. 3-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. **Transport qatlami** dasturlar o‘rtasida muloqot o‘rnatadi va 4-lik kortej orqali demultipleksirlashni amalga oshiradi.
2. **UDP** — ulanishsiz, minimal sarlavhali, tezkor protokol bo‘lib, DNS va video oqimlar uchun ayni muddao.
3. **TCP** — 100% ishonchli, tartiblangan baytlar oqimini taqdim etadi. Uning poydevori — **3-Way Handshake, Sequence/ACK raqamlari, Oqim nazorati (`rwnd`) va Tiqilinch nazorati (`cwnd`)** hisoblanadi.

### 🔗 Joriy papkadagi laboratoriyalar bilan bog‘liqlik:
* Ushbu bobdagi **3-Way Handshake va SYN Flood mexanizmi** to‘g‘ridan-to‘g‘ri **`A12_project`** (DDoS hujumlaridan himoyalanish, `synproxy` sozlash va rate-limiting) laboratoriyasining nazariy asosidir!
* TCP bayroqlari, oyna hajmi va oqimlarni tahlil qilish esa **`A9_project`** (Wireshark bilan paketlar tahlili) da paketlarni filtrlab, xakerlik harakatlarini ushlash uchun zarurdir!
