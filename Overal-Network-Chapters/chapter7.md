# 7-BOB: MULTIMEDIYA TARMOQLARI (Multimedia Networking)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Multimediya ilovalarining turlari (saqlangan video striming, jonli efir, interaktiv VoIP), video kodlash (CBR vs VBR), DASH (Dynamic Adaptive Streaming over HTTP) va CDN (Content Distribution Networks) arxitekturasi, real vaqt aloqasidagi kechikish va Jitter muammolari, paket yo‘qolishini qoplash usullari (FEC, Interleaving), RTP, RTCP va SIP protokollari hamda tarmoqda xizmat sifati (QoS / DiffServ) kafolatlarini chuqur o‘rganish.

---

## 7.1. MULTIMEDIYA TARMOQ ILOVALARI VA ULARNING XUSUSIYATLARI

Bugungi kunda butun global Internet trafigining **70–80% dan ortig‘ini** video oqimlar (YouTube, Netflix, TikTok, Zoom) tashkil qiladi. Multimediya ma’lumotlari an’anaviy matn yoki veb-sahifalardan tubdan farq qiluvchi xususiyatlarga ega:

```
+-------------------------------------------------------------------------+
|                  MULTIMEDIYA ILOVALARINING 3 TA ASOSIY TURI             |
+-------------------------------------------------------------------------+
| 1. Saqlangan Video/Audio Striming (Streaming Stored Video)              |
|    * YouTube, Netflix, Spotify.                                         |
|    * Foydalanuvchi butun fayl yuklanishini kutmaydi, yuklanish          |
|      davomida tomosha qilaveradi. Oldindan buferlash mumkin.            |
+-------------------------------------------------------------------------+
| 2. Jonli Efir Strimingi (Live Streaming)                                |
|    * Sport musobaqalari, yangiliklar, Twitch, YouTube Live.             |
|    * Voqea real vaqtda ro'y bermoqda. Ko'p sekund oldindan              |
|      buferlab bo'lmaydi, kechikish bir necha sekunddan oshmasligi kerak.|
+-------------------------------------------------------------------------+
| 3. Haqiqiy Vaqt Interaktiv Aloqa (Real-Time Interactive / Conversational)|
|    * Zoom, Google Meet, Skype, IP-telefoniya (VoIP).                    |
|    * Ikki tomonlama jonli suhbat. Qat'iy talab: Kechikish 150-200 ms    |
|      dan oshmasligi shart, aks holda odamlar bir-birining gapini bo'ladi!|
+-------------------------------------------------------------------------+
```

---

### 7.1.1. Raqamli Video va Audioning Jismoniy Xususiyatlari

* **Raqamli Video:**
  * Video — bu soniyasiga ma’lum tezlikda (odatda 24, 30 yoki 60 kadr/sekund — fps) almashtiriladigan tasvirlar ketma-ketligidir.
  * *Katta hajm muammosi:* Agar 1080p Full HD (1920x1080) videoni siqmasdan (uncompressed, har bir piksel 24 bit) uzatsak, soniyasiga **1.5 Gbps** o‘tkazuvchanlik kerak bo‘ladi! Hech qanday internet buni ko‘tara olmaydi.
  * **Video Siqish (Video Compression / Encoding):**
    * *Fazoviy ortiqchalik (Spatial Redundancy):* Bitta kadr ichidagi bir xil rangdagi qo‘shni piksellarni (masalan, moviy osmon fonini) bitta formula bilan siqish.
    * *Vaqtinchalik ortiqchalik (Temporal Redundancy):* Ketma-ket kelgan kadrlar o‘rtasida deyarli o‘zgarmagan fonni qayta yubormasdan, faqat harakatlanayotgan ob’yektning siljish vektorini yuborish.
  * **CBR (Constant Bit Rate):** Siqilgan video butun uzunligi bo‘ylab bir xil qat’iy tezlikda (masalan, 2 Mbps) uzatiladi.
  * **VBR (Variable Bit Rate):** Harakatsiz sahnalarda bitreyt pasayadi (500 Kbps), harakatga boy jangari sahnalarda esa bitreyt keskin ko‘tariladi (4 Mbps).
* **Raqamli Audio (PCM — Pulse Code Modulation):**
  * Ovoz to‘lqini soniyasiga 8 000 marta (telefoniyada) yoki 44 100 marta (CD sifatda) diskretlanadi (Sample).
  * Har bir namuna 8 bit (256 daraja) yoki 16 bit bilan kvantlanadi. Telefon sifati uchun: $8000 \times 8 = 64 \text{ Kbps}$ o‘tkazuvchanlik kerak bo‘ladi.

---

## 7.2. SAQLANGAN VIDEO STRIMING VA DASH (DYNAMIC ADAPTIVE STREAMING OVER HTTP)

### 7.2.1. Nega UDP Strimingdan Voz Kechildi?
90-yillarda video striming UDP ustida qurilgan edi. Lekin bugungi kunda **YouTube va Netflix to‘liq TCP va HTTP** dan foydalanadi. Nega?
1. **Xavfsizlik devorlari (Firewalls/NAT):** Deyarli barcha korporativ xavfsizlik devorlari va proksilar notanish UDP portlarni qat’iy bloklaydi, lekin HTTP/HTTPS (Port 80/443) hamma joyda ochiq!
2. **Server yuklamasi:** UDP da server har bir mijozning tezligini alohida kuzatishi kerak. HTTP da esa oddiy veb-serverlar va mavjud arzon Veb-Keshlar (CDN) ishlatiladi.

---

### 7.2.2. DASH (Dynamic Adaptive Streaming over HTTP) Mexanizmi

DASH — zamonaviy Internet video strimingining standartidir (YouTube, Netflix, HBO shu asosda ishlaydi).

```
[ ASOSIY VIDEO FAYL ]
        |
        v (Video turli sifatlarda kodlanadi va 2-5 sekundlik bo'laklarga ajratiladi)
+-------------------------------------------------------------------------+
| Sifat darajasi: | Bo'lak 1 (0-4 sek) | Bo'lak 2 (4-8 sek) | Bo'lak 3... |
| 4K (15 Mbps)    | [Chunk 1 - 4K]     | [Chunk 2 - 4K]     | [Chunk 3]   |
| 1080p (5 Mbps)  | [Chunk 1 - 1080p]  | [Chunk 2 - 1080p]  | [Chunk 3]   |
| 720p (2.5 Mbps) | [Chunk 1 - 720p]   | [Chunk 2 - 720p]   | [Chunk 3]   |
| 360p (800 Kbps) | [Chunk 1 - 360p]   | [Chunk 2 - 360p]   | [Chunk 3]   |
+-------------------------------------------------------------------------+
                                    |
                                    v
            [ Manifest Fayli (MPD - Media Presentation Description) ]
            (Qaysi bo'lak qayerda va qanday tezlikdaligini ko'rsatuvchi URLlar)
```

#### Mijozning Aqlli Moslashuv Algoritmi:
1. Brauzer birinchi bo‘lib kichik **Manifest faylini (MPD)** yuklab oladi.
2. Brauzer video pleyeri o‘zining yuklab olish tezligini (Throughput) har soniyada o‘lchab turadi.
3. **Avtomatik moslashuv:**
   * Agar Wi-Fi aloqasi zo‘r bo‘lsa va bufer to‘lib tursa $\implies$ Mijoz keyingi 4 soniyalik bo‘lakni **1080p yoki 4K** formatda so‘raydi (`GET /video_chunk3_4k.m4s`).
   * Agar tarmoqda tirbandlik boshlansa va bufer bo‘shab qolsa $\implies$ Pleyer videoni to‘xtatib qotirmaslik (re-buffering qilmaslik) uchun darhol keyingi bo‘lakni **360p yoki 480p** formatda so‘raydi (`GET /video_chunk4_360p.m4s`).
4. Natijada foydalanuvchi qotib qolmasdan videoni uzluksiz tomosha qilaveradi!

---

### 7.2.3. Kontent Yetkazib Berish Tarmoqlari — CDN (Content Distribution Networks)

Agar butun dunyodagi yuz millionlab odamlar Netflix filmini Kaliforniyadagi yagona bitta serverdan so‘rasa, transatlantik aloqa liniyalari qulab tushgan bo‘lardi.

**CDN yechimi:** Kontent nusxalarini butun dunyo bo‘ylab, **foydalanuvchilarga geografik jihatdan imkon qadar yaqin** joylashgan minglab serverlarga oldindan ko‘chirib qo‘yish.

```
                                [ Markaziy Server (Origin) ]
                                              |
                   +--------------------------+--------------------------+
                   |                                                     |
                   v                                                     v
          [ CDN Server: Toshkent ]                              [ CDN Server: Frankfurt ]
          (Uztelecom IXP ichida)                                (DE-CIX IXP ichida)
                   |                                                     |
             [ O'zbekistonlik ]                                     [ Germaniyalik ]
              foydalanuvchilar                                       foydalanuvchilar
```

* **Enter Deep falsafasi (Akamai):** CDN serverlarini butun dunyodagi mahalliy ISP larning ichiga (yerto‘lasigacha) olib kirib joylashtirish.
* **Bring Home falsafasi (Limelight):** Yirik IXP nuqtalarida katta klasterlar qurish.
* **DNS Anycast va Yo‘naltirish:** Foydalanuvchi `video.netflix.com` ni so‘raganda, CDN ning vakolatli DNS serveri mijozning Local DNS IP sini tahlil qilib, unga dunyodagi eng yaqin turgan CDN serverining IP manzilini qaytaradi!

---

## 7.3. HAQIQIY VAQT INTERAKTIV ALOQASI: VOIP (VOICE OVER IP)

IP-telefoniya va video konferensiyalarning bosh dushmani — bu **Kechikish (Delay)** va **Jitter (Kechikishlar farqi)** dir.

### 7.3.1. Jitter Muammosi va Playout Buferi

* **Jitter (Titroq / Tebranish):** Paketlarning router navbatlarida har xil vaqt kutishi sababli qabul qiluvchiga notekis oraliqlarda yetib kelishi.
* Agar jo‘natuvchi har 20 ms da bitta audio paket yuborsa, qabul qiluvchiga paketlar 20 ms, 5 ms, 60 ms, 10 ms farq bilan kelishi mumkin. Agar audio darhol dinamikka berilsa, ovoz robotlashib, uzilib-uzilib eshitiladi.

```
Paket jo'natildi:   [1]----(20ms)----[2]----(20ms)----[3]----(20ms)----[4]
                                  TARMOQDAGI JITTER
Paket keldi:        [1]----------[2]-----------------------[3]-[4]
                                  PLAYOUT BUFFER (Kechiktirib eshittirish)
Dinamikda ijro:     ...... kutadi ...... [1]----(20ms)----[2]----(20ms)----[3]----(20ms)----[4]
```

* **Yechim: Qabul qiluvchi tomonidagi Playout Delay Buferi:**
  * Kelgan birinchi audio paket darhol eshittirilmaydi, balki buferda ma’lum vaqt (masalan, 100 ms) ushlab turiladi.
  * Shundan so‘ng barcha paketlar qat’iy 20 ms oraliqda bir tekis dinamikka chiqariladi. Jitter to‘liq yo‘qotiladi!
  * *Cheklov:* Umumiy kechikish (tarmoq + bufer) **150–200 ms dan oshmasligi kerak**, 400 ms dan oshsa suhbatlashish imkonsiz bo‘lib qoladi.

---

### 7.3.2. Paket Yo‘qolishini Qoplash Usullari (Loss Recovery)

VoIP da yo‘qolgan paketni qayta uzatish (TCP kabi retransmit qilish) befoyda, chunki paket qaytib kelguncha suhbatdosh allaqachon boshqa gapni gapirayotgan bo‘ladi.
Yo‘qotishlarni bartaraf etishning 3 ta aqlli usuli bor:

#### 1. Forward Error Correction (FEC — Xatoni Oldindan Tuzatish)
* *Usul A (Kichik sifatli nusxa qo‘shish):* Har bir paketga o‘zidan oldingi paketning past sifatli (masalan, GSM 13 Kbps) audio nusxasi qo‘shib yuboriladi. Agar $n$-paket yo‘qolsa, $n+1$-paket ichidagi nusxadan audio qayta tiklanadi!
* *Usul B (XOR paritet):* Har 4 ta paketga ularning XOR yig‘indisidan iborat 5-paritet paketi qo‘shiladi. Bitta paket yo‘qolsa, qolgan 4 tasi orqali yo‘qolgan paket to‘liq matematik tiklanadi.

#### 2. Interleaving (Aralashtirish)
Ketma-ket kelgan 5 ms lik audio bo‘laklari bitta paketga solinmaydi, balki turli paketlarga sochib yuboriladi:
* Agar bitta paket yo‘qolsa, butun boshli so‘z emas, balki turli so‘zlarning bilinmas 5 millisekundlik mikroskopik bo‘laklari yo‘qoladi. Inson miyasi buni o‘zi avtomatik to‘ldirib eshitaveradi!

#### 3. Error Concealment (Xatoni yashirish)
Paket yo‘qolsa, pleyer sukunat (jimjitlik) qilmaydi, balki o‘zidan oldingi paketning tovush to‘lqinini bir oz so‘ndirib cho‘zib beradi.

---

## 7.4. REAL VAQT PROTOKOLLARI: RTP, RTCP VA SIP

### 7.4.1. RTP (Real-Time Transport Protocol — RFC 3550)

RTP — audio va video ma’lumotlarni UDP paketlari ichiga solib uzatish uchun xalqaro standart sarlavhadir:

```
+---------------+---------------+---------------+-------------------------------+
| V=2 | P | X | CC| M | Payload |     Sequence Number (16 bit)                  |
+---------------+---------------+-----------------------------------------------+
|                           Timestamp (32 bit)                                  |
+-------------------------------------------------------------------------------+
|                Synchronization Source Identifier (SSRC - 32 bit)              |
+-------------------------------------------------------------------------------+
|                              Audio / Video Data                               |
+-------------------------------------------------------------------------------+
```

* **Payload Type (7 bit):** Oqim qaysi kodek bilan siqilganini bildiradi (masalan, `0` — PCM $\mu$-law 64 Kbps, `26` — Motion JPEG, `31` — H.261).
* **Sequence Number (16 bit):** Har bir RTP paket chiqqanda 1 taga oshadi. Qabul qiluvchi yo‘qolgan paketlarni va paketlar tartibini aniqlaydi.
* **Timestamp (32 bit):** Ushbu audio namunasining hosil bo‘lgan aniq vaqt belgisi (Jitter buferini tekislash uchun ishlatiladi).
* **SSRC (32 bit):** Oqim manbasining (mikrofon yoki kameraning) unikal identifikatori.

---

### 7.4.2. RTCP (RTP Control Protocol)
RTP bilan parallel ishlaydi. U ma’lumot uzatmaydi, balki suhbatdoshlar o‘rtasida **tarmoq sifati haqidagi statistik hisobotlarni** almashib turadi:
* "Men sendan 1000 ta paket kutdim, 25 tasi yo‘qoldi, Jitterning o‘rtacha qiymati 12 ms bo‘ldi".
* Bu statistikaga qarab jo‘natuvchi tomon o‘z kodekining sifatini pasaytirishi yoki oshirishi mumkin.

---

### 7.4.3. SIP (Session Initiation Protocol — RFC 3261)

RTP ma’lumotni tashiydi, lekin suhbat boshlanishidan oldin:
* Suhbatdoshni qanday topamiz?
* Uning qo‘ng‘irog‘ini qanday jiringlatamiz?
* Ikkala telefon qaysi video/audio kodekda gaplashishini (SDP protokoli orqali) qanday kelishamiz?
Bu vazifalarni **SIP protokoli** bajaradi!

```
[ Alice (Mijoz) ]              [ SIP Registrar Server ]             [ Bob (Mijoz) ]
       |                                  |                                |
       | 1. INVITE sip:bob@domain.uz ---->|                                |
       |    (Kodeklar ro'yxati SDP da)    |---- 2. INVITE ---------------->|
       |                                  |<--- 3. 180 Ringing (Jiringladi)|
       |<-- 4. 180 Ringing ---------------|                                |
       |                                  |<--- 5. 200 OK (Bob ko'tardi)---|
       |<-- 6. 200 OK --------------------|                                |
       |                                                                   |
       |== 7. ACK (Muloqot tasdiqlandi) ==================================>|
       |                                                                   |
       |<======== 8. TO'G'RIDAN-TO'G'RI RTP AUDIO OQIMI (UDP) ============>|
       |                                                                   |
       |-- 9. BYE (Go'shak qo'yildi) ------------------------------------->|
       |<-- 10. 200 OK ----------------------------------------------------|
```

* SIP xabarlari xuddi HTTP kabi ASCII matn ko‘rinishida bo‘ladi (`INVITE`, `ACK`, `BYE`, `200 OK`).
* Qo‘ng‘iroq o‘rnatilgach, server oraliqdan chiqadi va audio paketlar to‘g‘ridan-to‘g‘ri **RTP orqali Peer-to-Peer** oqadi!

---

## 7.5. TARMOQNING QO‘LLAB-QUVVATLAShI: XIZMAT SIFATI (QoS — QUALITY OF SERVICE)

An’anaviy Internet — bu **"Best-Effort" (Qo‘ldan kelgancha xizmat)** tizimidir. Router uchun oddiy fayl yuklanishi bilan jonli yurak operatsiyasini boshqarayotgan jarrohning videosi bir xil narsa.

Buni to‘g‘rilash uchun **DiffServ (Differentiated Services — RFC 2475)** arxitekturasi yaratilgan:

```
[Paket kirdi] ---> [ Packet Classifier (Saralash) ] ---> [ Meter / Policer (O'lchash) ]
                          |                                     |
                          +---> Muhim VoIP trafigi  ----------> Priority Queue (Navbatsiz o'tadi!)
                          +---> Oddiy Web trafigi   ----------> Best-Effort Queue (Kutib turadi)
```

1. **Paketlarni belgilash (Marking):** IPv4 sarlavhasidagi `Type of Service (ToS)` maydoniga (yoki IPv6 da `Traffic Class`) paketning sinfi (DSCP kodi) yoziladi. Masalan, VoIP uchun `EF (Expedited Forwarding)` belgilanadi.
2. **Tirbandlikni cheklash (Traffic Policing & Shaping):** Foydalanuvchi belgilangan shartnomadan ortiq paket chiqarmasligini nazorat qilish uchun **Leaky Bucket (Teshik chelak)** va **Token Bucket (Jetonsimon chelak)** algoritmlari ishlatiladi.
3. **Imtiyozli rejalashtirish (Priority Queuing / WFQ):** Router chiqish portida muhim paketlar hech qanday navbatsiz, birinchi bo‘lib liniyaga chiqariladi.

---

## 7.6. 7-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. Multimediya ilovalari kechikishga o‘ta ta’sirchan, lekin oz miqdordagi paket yo‘qolishiga chidamlidir.
2. Zamonaviy video striming (YouTube, Netflix) **DASH** protokoli orqali HTTP/TCP ustida ishlaydi va **CDN** orqali keshlanadi.
3. Jonli VoIP aloqasi **RTP** (ma’lumot tashish), **RTCP** (sifat nazorati) va **SIP** (qo‘ng‘iroqni boshqarish) protokollari triadasiga tayanadi.
4. Jitter buferlari va yo‘qotishni qoplash (FEC, Interleaving) audio sifatini saqlab qoladi.

### 🔗 Joriy papkadagi laboratoriyalar bilan bog‘liqlik:
* Ushbu bobdagi video oqimlar, kechikishlar va xizmat sifati tushunchalari **`A12_project`** (DDoS hujumlaridan himoyalanishda audio/video xizmatlarining o‘tkazuvchanligini kafolatlash) hamda **`A15_project`** (IoT datchiklaridan uzluksiz telemetriya oqimlarini boshqarish) laboratoriyalari uchun muhim poydevordir!
