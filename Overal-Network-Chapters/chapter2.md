# 2-BOB: DASTUR QATLAMI (Application Layer)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Tarmoq ilovalari arxitekturasi (Mijoz-Server va P2P), jarayonlararo muloqot va soketlar, Web va HTTP protokoli (versiyalari, ulanish turlari, keshlar, kukilar), elektron pochta (SMTP, IMAP), nomlar tizimi (DNS ierarxiyasi, yozuv turlari, so‘rovlar) hamda tarmoq dasturlash asoslarini fundamental darajada chuqur o‘rganish.

---

## 2.1. TARMOQ ILOVALARI TAMOYILLARI VA ARXITEKTURASI

Tarmoq ilovalarini ishlab chiqishning eng katta afzalligi — siz tarmoq yadrosidagi routerlar yoki kommutatorlar uchun kod yozmaysiz. Routerlar va kommutatorlar quyi qatlamlarda (L3 va L2) paketlarni yo‘naltirish bilan shug‘ullanadi. Tarmoq ilovalari esa faqat tarmoqning chetidagi **Yakuniy tizimlarda (End Systems / Hosts)** ishlaydi.

```
+-------------------------------------------------------------------------+
| TARMOQ CHETI (Xost A)                               TARMOQ CHETI (Xost B)|
| [ Veb-brauzer ilovasi ]                             [ Veb-server ilovasi] |
|           |                                                    ^         |
|           v                                                    |         |
|     (Socket eshigi)                                      (Socket eshigi)  |
|           |                                                    |         |
|  [ Transport Qatlami ]                              [ Transport Qatlami ]|
|           |                                                    |         |
+-----------|----------------------------------------------------|---------+
            \                                                    /
             \======== [ TARMOQ YADROSI (Routerlar to'ri) ] =====/
```

---

### 2.1.1. Tarmoq Ilovalari Arxitekturalari

Tarmoq ilovasi asosan ikkita klassik arxitekturadan biriga asoslanadi:

#### 1. Mijoz-Server Arxitekturasi (Client-Server Architecture)
Eng keng tarqalgan an’anaviy model (Web, E-mail, FTP, DNS):
* **Server:**
  * Doimo yoqiq holatda (**Always-on**) turadi.
  * O‘zgarmas, doimiy ommaviy IP-manzilga ega bo‘ladi.
  * Ko‘plab mijozlarning bir vaqtdagi so‘rovlariga javob beradi.
  * *Masshtablash muammosi:* Bitta jismoniy server millionlab foydalanuvchilar oqimini ko‘tara olmaydi. Shu sababli yirik kompaniyalar (Google, Meta, Amazon) yuz minglab serverlardan iborat qudratli virtual klasterlar — **Ma’lumotlar Markazlari (Data Centers)** dan foydalanadi.
* **Mijoz (Client):**
  * Foydalanuvchi qurilmasida kerak bo‘lganda ishga tushadi (brauzer, mobil ilova).
  * Server bilan aloqa o‘rnatadi, lekin **boshqa mijozlar bilan to‘g‘ridan-to‘g‘ri muloqot qilmaydi**.
  * IP-manzili o‘zgaruvchan (dinamik) bo‘lishi mumkin (masalan, Wi-Fi dan 4G ga o‘tganda).

#### 2. Peer-to-Peer (P2P) Arxitekturasi
Markaziy serverga tayanmaydigan taqsimlangan model (BitTorrent, Blockchain, Skype ning ilk versiyalari):
* Doimiy server yo‘q. Tarmoq ishtirokchilari (**Peers — Tengdoshlar**) to‘g‘ridan-to‘g‘ri bir-biri bilan ma’lumot almashadi.
* **O‘z-o‘zini masshtablash (Self-Scalability):** Har bir yangi qo‘shilgan ishtirokchi nafaqat ma’lumotni yuklab oladi (iste’molchi bo‘ladi), balki bir vaqtning o‘zida o‘zida bor bo‘laklarni boshqalarga yuklaydi (resurs yetkazuvchi bo‘ladi). Natijada tizim foydalanuvchilar soni ortgani sari quvvatini oshirib boradi.

---

### 2.1.2. Jarayonlararo Aloqa (Processes Communicating) va Soketlar

Dasturlar emas, balki operatsion tizim ichidagi **jarayonlar (processes)** muloqot qiladi:
* Bitta kompyuter ichidagi ikki jarayon operatsion tizimning IPC (Inter-Process Communication) mexanizmlari orqali gaplashadi.
* Turli kompyuterlardagi jarayonlar esa tarmoq orqali **Xabarlar (Messages)** almashadi.

#### Socket (Soket) — Dasturiy Eshik
* Jarayon va kompyuter tarmog‘i o‘rtasidagi dasturiy interfeys.
* Ilova o‘z xabarini soket orqali transport qatlamiga topshiradi. U xuddi xatni pochta qutisiga tashlashga o‘xshaydi.
* Dasturchi soketning ilova tomonini to‘liq boshqara oladi, lekin transport tomonida faqat bir nechta parametrlarni (protokol tanlash, bufer hajmi) belgilay oladi.

#### Jarayonni Tarmoqda Qanday Topamiz? (IP va Port)
Xabarni kerakli jarayonga yetkazish uchun ikkita ko‘rsatkich zarur:
1. **Xostning IP-manzili (32 bit):** Maqsad kompyuterni butun dunyo bo‘ylab aniqlaydi.
2. **Port raqami (16 bit):** O‘sha kompyuter ichida aynan qaysi dastur soketi kutayotganini aniqlaydi.
   * Standart portlar: `80` (HTTP), `443` (HTTPS), `25` (SMTP), `53` (DNS), `22` (SSH).

---

### 2.1.3. Ilovalar Talab Qiladigan Transport Xizmatlari

Har xil ilovalarning transport qatlamiga qo‘yadigan talablari turlicha bo‘ladi:

| Xizmat Talabi | Mohiyati | Ilovalar misoli |
| :--- | :--- | :--- |
| **Ishonchli uzatish (Reliability)** | Birorta ham bit yo‘qolmasligi, barcha paketlar to‘liq yetib borishi shart. | Veb (HTTP), E-mail, Fayllar (FTP), Moliyaviy tranzaksiyalar |
| **Yo‘qotishga chidamlilik (Loss-tolerant)** | Ozroq ma’lumot yo‘qolsa ham tizim ishlayveradi (ovoz yoki tasvirda xiralashish bo‘ladi). | Ovozli suhbatlar (VoIP), Video oqimlar, Onlayn o‘yinlar |
| **O‘tkazuvchanlik (Throughput)** | Minimal kafolatlangan tezlik talab qilinadi (masalan, 4K video uchun 25 Mbps). | Video striming (YouTube, Netflix) |
| **Vaqt va Kechikish (Timing)** | Paketlar qabul qiluvchiga 100–150 ms dan kechikmasdan yetib borishi kerak. | Real vaqt o‘yinlari (CS, PUBG), IP-telefoniya |
| **Xavfsizlik (Security)** | Maxfiylik (shifrlash), yaxlitlik va tomonlarni tekshirish. | Bank to‘lovlari, shaxsiy yozishmalar |

---

### 2.1.4. Internet Taqdim Etuvchi Transport Protokollari: TCP vs UDP

Internetning transport qatlami dasturchilarga ikkita tanlovni beradi:

```
                          TRANSPORT XIZMATLARI
                                   |
         +-------------------------+-------------------------+
         |                                                   |
         v                                                   v
TCP (Kafolatlangan va Og'ir)                 UDP (Tezkor va Yengil)
* Ulanishga yo'naltirilgan (3-Way Handshake) * Ulanishsiz (Connectionless)
* 100% ishonchli (Yo'qolgan paket qaytadi)  * Kafolat yo'q (Best-effort)
* Oqim nazorati (Flow Control: rwnd)         * Oqim nazorati yo'q
* Tiqilinch nazorati (Congestion Control)    * Tiqilinch nazorati yo'q (cheksiz uradi)
* Tartiblangan baytlar oqimi                 * Paketlar tartibsiz kelishi mumkin
* 20 bayt sarlavha (Katta qo'shimcha yuk)    * 8 bayt sarlavha (Minimal qo'shimcha yuk)
```

> **Muhim xavfsizlik eslatmasi (SSL / TLS):**  
> Na TCP, na UDP boshidanoq shifrlash imkoniyatiga ega emas (parollar ochiq matn ko‘rinishida o‘tadi). Shuning uchun ilova va TCP o‘rtasida **TLS (Transport Layer Security)** qatlami qo‘llaniladi. TLS bilan qurollangan TCP biz bilgan **HTTPS (Port 443)** ga aylanadi!

---

## 2.2. VEB VA HTTP PROTOKOLI (HYPERTEXT TRANSFER PROTOCOL)

Veb — bu Internetning eng asosiy xizmatidir. U **HTTP (RFC 1945, RFC 2616, RFC 7540)** protokoli asosida ishlaydi.
* Veb-sahifa ko‘plab **Ob’yektlardan (Objects)** tashkil topadi.
* Ob’yekt — bu HTML fayl, JPEG rasm, CSS uslublar jadvali, JavaScript skripti yoki video fayl bo‘lishi mumkin.
* Har bir ob’yekt o‘zining **URL (Uniform Resource Locator)** manziliga ega:
  $$\text{URL: } \underbrace{\text{http://www.someSchool.edu}}_{\text{Xost nomi}} / \underbrace{\text{someDept/pic.gif}}_{\text{Yo'l (Path)}}$$

---

### 2.2.1. HTTP ning Doimiy Bo‘lmagan va Doimiy Ulanishlari

HTTP transport qatlamida ishonchli **TCP** protokoliga tayanadi. Mijoz veb-sahifadagi 10 ta rasmni yuklab olmoqchi bo‘lsa, u buni qanday amalga oshiradi?

#### 1. Doimiy bo‘lmagan ulanish (Non-Persistent HTTP / HTTP 1.0)
* Har bir ob’yekt (har bitta rasm) uchun **alohida yangi TCP ulanish** ochiladi va ob’yekt yuklab olingach, ulanish darhol yopiladi.
* **RTT (Round-Trip Time — Borib-kelish vaqti):** Kichik paketning mijozdan serverga borib, yana qaytib kelishi uchun ketadigan vaqt.
* Bitta ob’yektni yuklab olish vaqti:
  $$\text{Vaqt} = 2 \times \text{RTT} + \text{Faylni uzatish vaqti}$$
  *(1-RTT ulanish o‘rnatishga (TCP SYN/ACK), 2-RTT HTTP GET so‘roviga va javob boshlanishiga ketadi).*
* *Kamchiligi:* Agar sahifada 50 ta rasm bo‘lsa, 50 marta TCP ulanish ochiladi. Server har bir ulanish uchun alohida xotira ajratadi, tarmoq RTT kechikishlariga to‘lib ketadi.

#### 2. Doimiy ulanish (Persistent HTTP / HTTP 1.1)
* Server javob bergandan keyin TCP ulanishni yopmaydi, balki ochiq qoldiradi (**Keep-Alive**).
* Barcha 50 ta rasm shu yagona ochiq ulanish orqali ketma-ket (pipelining bilan) yuklab olinadi. Natijada boshlang‘ich 1 ta RTT dan so‘ng barcha ob’yektlar tezda yuklanadi.

---

### 2.2.2. HTTP Xabarlar Formati (Tuzilishi)

HTTP xabarlari odam o‘qiy oladigan oddiy ASCII matn ko‘rinishida bo‘ladi.

#### 1. HTTP So‘rov Xabari (Request Message)
```http
GET /index.html HTTP/1.1
Host: www.bank.uz
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: text/html,application/xhtml+xml
Accept-Language: uz-UZ,uz;q=0.9,en-US;q=0.8
Connection: keep-alive
If-Modified-Since: Sat, 07 Nov 2026 12:00:00 GMT

[Bo'sh qator: Sarlavhalar tugaganini bildiradi]
[Entity Body: POST so'rovlarida yuboriladigan ma'lumotlar]
```

* **So‘rov qatori (Request Line):** Uchta maydondan iborat: `Metod`, `URL` va `HTTP Versiyasi`.
  * `GET`: Serverdan sahifa yoki faylni talab qilish.
  * `POST`: Serverga ma’lumot yuklash (forma to‘ldirilganda, login/parol kiritilganda ma’lumot Entity Body da yuboriladi).
  * `HEAD`: Xuddi GET kabi, lekin server ob’yektning o‘zini emas, faqat uning sarlavhalarini qaytaradi (diagnostika uchun).
  * `PUT`: Serverdagi belgilangan URL ga yangi fayl yuklash/almashtirish.
  * `DELETE`: Serverdagi resursni o‘chirish.

#### 2. HTTP Javob Xabari (Response Message)
```http
HTTP/1.1 200 OK
Date: Sat, 07 Nov 2026 15:30:00 GMT
Server: Nginx/1.24.0 (Ubuntu)
Last-Modified: Sat, 07 Nov 2026 10:15:20 GMT
Content-Length: 4520
Content-Type: text/html; charset=UTF-8

<!DOCTYPE html>
<html>
... HTML Ma'lumotlari ...
</html>
```

* **Status Qatori (Status Line):** Protokol versiyasi, Status kodi va qisqa xabar:
  * `200 OK`: So‘rov muvaffaqiyatli bajarildi, so‘ralgan ma’lumot ilova qilinmoqda.
  * `301 Moved Permanently`: Qidirilgan resurs butunlay boshqa manzilga ko‘chgan (yangi URL `Location:` sarlavhasida ko‘rsatiladi).
  * `304 Not Modified`: Sahifa o‘zgarmagan (Keshdan o‘qiyvering, ortiqcha trafik tejaladi).
  * `400 Bad Request`: Server mijozning so‘rovini tushunmadi (sintaktik xato).
  * `403 Forbidden`: Resurs mavjud, lekin mijozning unga kirish huquqi yo‘q.
  * `404 Not Found`: Bunday sahifa serverda topilmadi.
  * `500 Internal Server Error`: Serverda dasturiy xatolik yuz berdi.
  * `503 Service Unavailable`: Server haddan tashqari yuklangan yoki vaqtincha texnik xizmat ko‘rsatilmoqda.

---

### 2.2.3. Cookies (Kukilar) va Foydalanuvchi Holatini Saqlash

HTTP protokoli tabiatan **Holatsiz (Stateless)** dir — ya’ni server avvalgi so‘rov kimdan kelganini eslab qolmaydi. Lekin zamonaviy Internetda savatga tovar qo‘shish, avtorizatsiyadan o‘tish va shaxsiy tavsiyalar qanday ishlaydi?

Buning uchun **Cookie** texnologiyasi (RFC 6265) yaratilgan bo‘lib, u 4 ta komponentdan iborat:

```
[Mijoz (Brauzer)]                                            [Veb-Server (Amazon)]
       |                                                              |
       | 1. Oddiy HTTP So'rov (Birinchi marta kirdi)                  |
       |------------------------------------------------------------->| Server yangi ID yaratadi: 1678
       |                                                              | Ma'lumotlar bazasiga yozadi
       | 2. HTTP Javob + Set-cookie: 1678                             |
       |<-------------------------------------------------------------|
       | Brauzer cookie fayliga saqlaydi                              |
       |                                                              |
       | 3. Ikkinchi HTTP So'rov + Cookie: 1678                       |
       |------------------------------------------------------------->| Server ID orqali taniydi:
       |                                                              | "Bu Alisher, savatida 2 kitob bor"
```

1. HTTP javob xabaridagi maxsus sarlavha: `Set-cookie: 1678`.
2. Foydalanuvchi kompyuterida brauzer tomonidan boshqariladigan cookie fayllar bazasi.
3. Keyingi har bir HTTP so‘rovida brauzer yuboradigan sarlavha: `Cookie: 1678`.
4. Veb-sayt serveridagi ma’lumotlar bazasi (ID raqamga bog‘langan xaridlar tarixi, profil ma’lumotlari).

---

### 2.2.4. Veb-Keshlash (Web Caching) va Proksi Serverlar

Veb-kesh (Proksi server) — bu mijozlar va asosiy veb-server o‘rtasida turuvchi tarmoq ob’yekti.

```
[Mijoz A] ----+
              |-----> [ LOKAL VEB-KESH (Proksi) ] ===== (Internet) =====> [ Asosiy Server ]
[Mijoz B] ----+            (Universitet tarmog'ida)                          (AQShda joylashgan)
```

* **Ishlash mexanizmi:**
  1. Brauzer so‘rovni to‘g‘ridan-to‘g‘ri internetga emas, avval o‘zining lokal Proksi-keshiga yuboradi.
  2. Agar so‘ralgan ob’yekt keshda mavjud bo‘lsa, u **darhol mahalliy LAN tezligida (1 Gbps)** qaytariladi.
  3. Agar keshda bo‘lmasa, Proksi server o‘zi internetdagi asosiy serverdan faylni yuklab oladi, nusxasini o‘zida saqlaydi va mijozga beradi.
* **Afzalliklari:** Foydalanuvchi uchun sahifalar ochilish vaqti keskin qisqaradi, tashkilotning qimmatbaho tashqi internet kanali trafigi 50–70% gacha tejaladi.

#### Shartli GET (Conditional GET)
Keshdagi nusxa eskirib qolmaganini qanday bilamiz?
* Proksi server asosiy serverga maxsus sarlavha bilan so‘rov yuboradi:
  `GET /logo.png HTTP/1.1`  
  `If-Modified-Since: Sat, 07 Nov 2026 08:00:00 GMT`
* Agar serverda fayl o‘sha vaqtdan beri o‘zgarmagan bo‘lsa, server katta rasmni qayta jo‘natmaydi, balki atigi bir necha baytlik javob qaytaradi:
  `HTTP/1.1 304 Not Modified`
* Proksi o‘zida bor nusxani xotirjam mijozga beraveradi.

---

## 2.3. ELEKTRON POCHTA VA PROTOKOLLAR (SMTP, POP3, IMAP)

Pochta tizimining arxitekturasi veb tizimidan tubdan farq qiladi va 3 ta asosiy komponentga bo‘linadi:

```
[User Agent (Mijoz)]           [Jo'natuvchi Mail Server]             [Qabul qiluvchi Mail Server]        [User Agent]
(Outlook / Thunderbird)            (mail.uztelecom.uz)                    (mail.google.com)               (Mobil ilova)
         |                                   |                                     |                           |
         |---- 1. SMTP (yoki Web/HTTP) ----->|                                     |                           |
         |                                   |======== 2. SMTP (Port 25) =========>|                           |
         |                                   |          (Serverlar o'rtasida)      |                           |
         |                                   |                                     |<-- 3. IMAP / POP3 --------|
```

1. **User Agent (Foydalanuvchi agenti):** Pochta dasturlari (Outlook, Apple Mail, Thunderbird).
2. **Mail Serverlar:**
   * Har bir foydalanuvchi uchun **Pochta qutisi (Mailbox)** saqlanadi.
   * Xatlarni uzatish uchun **Xabarlar navbati (Message Queue)** mavjud. Agar qabul qiluvchi server vaqtincha o‘chiq bo‘lsa, server har 30 daqiqada qayta urinib ko‘radi.
3. **SMTP (Simple Mail Transfer Protocol — RFC 5321):**
   * Serverlar o‘rtasida xatni uzatishning asosiy protokoli.
   * Ishonchli **TCP (Port 25)** ustida ishlaydi.
   * **Push protokoli:** Jo‘natuvchi tomon xatni qabul qiluvchi serverga "itarib" beradi.
   * *Cheklovi:* SMTP faqat 7-bitli ASCII matnni qabul qiladi. Rasm, audio yoki o‘zbek tilidagi harflarni uzatish uchun ular **MIME (Multipurpose Internet Mail Extensions — RFC 2045)** yordamida Base64 formatiga o‘giriladi.

### Pochta O‘qish Protokollari (Mail Access Protocols): Nega SMTP bilan o‘qib bo‘lmaydi?
SMTP faqat xatni serverga yetkazadi (Push). Foydalanuvchi esa o‘z kompyuteriga xatni serverdan tortib olishi (**Pull**) kerak.
* **POP3 (Post Office Protocol 3 — RFC 1939):**
  * O‘ta sodda protokol (Port 110).
  * Xatlarni serverdan shaxsiy kompyuterga yuklab oladi va sukut bo‘yicha serverdan o‘chiradi ("Download-and-Delete").
  * *Muammo:* Agar xatni ish kompyuteringizda o‘qisangiz, uyga kelganda noutbukingizda yoki telefoningizda u xat ko‘rinmaydi!
* **IMAP (Internet Message Access Protocol — RFC 3501):**
  * Zamonaviy, qudratli protokol (Port 143).
  * Barcha xatlar va papkalar **serverning o‘zida markazlashgan holda** saqlanadi.
  * Siz bir vaqtning o‘zida telefondan, kompyuterdan va veb-brauzerdan kirsangiz ham barcha xatlar, o‘qilganlik belgilari va papkalar to‘liq sinxron holatda turadi.

---

## 2.4. DNS — INTERNETNING MA’LUMOTNOMA XIZMATI (DOMAIN NAME SYSTEM)

Kompyuterlar sonli IP-manzillar (`142.250.180.3`) bilan ishlashga moslashgan, insonlar esa eslab qolish oson bo‘lgan so‘zlarni (`google.com`) ishlatadi. Bu ikki olamni bir-biriga bog‘lovchi tizim — **DNS (RFC 1034, 1035)** dir.

DNS — bu shunchaki bitta server emas, u **ierarxik, markazlashtirilmagan ulkan ma’lumotlar bazasi** hamda **UDP 53-port** ustida ishlovchi ilova protokolidir.

---

### 2.4.1. Nega DNS Markazlashtirilmagan?
Tasavvur qiling, butun dunyo uchun yagona bitta Markaziy DNS Server bo‘lsa nima bo‘lardi?
1. **Single Point of Failure (Yagona zaiflik nuqtasi):** Agar o‘sha bitta server ishdan chiqsa, butun yer yuzida Internet to‘xtaydi!
2. **Ulkan tirbandlik (Traffic Volume):** Dunyodagi barcha milliardlab so‘rovlar bitta nuqtaga uriladi.
3. **Geografik masofa kechikishi:** Avstraliyadagi foydalanuvchi AQShdagi serverga so‘rov yuborsa, har safar yuzlab millisekund kechikish bo‘ladi.
4. **Boshqaruvning imkonsizligi:** Butun dunyodagi yangi saytlarni yagona serverda ro‘yxatdan o‘tkazish ulkan byurokratiyaga aylanadi.

---

### 2.4.2. DNS Ierarxiyasi

```
                                [ ROOT DNS SERVERLAR ]
                                (Dunyo bo'ylab 13 ta mantiqiy guruh)
                                          |
         +--------------------------------+--------------------------------+
         |                                |                                |
   [ .COM TLD Server ]            [ .UZ TLD Server ]              [ .ORG TLD Server ]
         |                                |
 [ google.com Authoritative ]     [ gov.uz Authoritative ]
```

1. **Root DNS Serverlar:** Ierarxiyaning eng yuqori cho‘qqisi. Ular bevosita IP-manzilni bilmaydi, lekin har bir yuqori domen (`.com`, `.uz`, `.net`) ning TLD serverlari qayerda joylashganini biladi.
2. **TLD (Top-Level Domain) Serverlar:** `.com`, `.org`, `.edu` kabi umumiy (gTLD) hamda `.uz`, `.ru`, `.uk` kabi milliy (ccTLD) domenlar uchun mas’ul bo‘lgan serverlar.
3. **Authoritative (Vakolatli) DNS Serverlar:** Har bir tashkilotning o‘z domeniga tegishli rasmiy yozuvlarini saqlovchi serverlar.
4. **Local DNS Server (Rezolver):** Provayderingiz (ISP) yoki korxonangiz ichidagi oraliq DNS server. Sizning kompyuteringiz so‘rovni faqat o‘zining Local DNS serveriga beradi.

---

### 2.4.3. DNS So‘rovlari Qanday Amalga Oshiriladi? (Iterativ vs Rekursiv)

```
[Mijoz PC] ---> 1. "google.com IPsi nima?" ---> [Local DNS Server]
                                                       |
     +--- 2. "google.com kimda?" ----------------------+
     v
[Root DNS] ===> 3. "Men bilmayman, mana bu .com TLD IPsi" ===> (Qaytaradi)
                                                       |
     +--- 4. "google.com kimda?" ----------------------+
     v
[.com TLD] ===> 5. "Mana google.com Authoritative IPsi" ====> (Qaytaradi)
                                                       |
     +--- 6. "google.com IPsi nima?" ------------------+
     v
[Google DNS] => 7. "Uning IPsi: 142.250.180.3" =============> (Qaytaradi)
                                                       |
[Mijoz PC] <--- 8. "IP: 142.250.180.3" <---------------+
```

* **Iterativ so‘rov (Surishtiruvchi):** Local DNS server Root serverga boradi. Root unga tayyor javobni emas, balki TLD ning manzilini beradi. Local DNS o‘zi borib TLD dan so‘raydi. TLD unga Authoritative manzilini beradi. Barcha qiyin izlanishni Local DNS ning o‘zi qadamma-qadam bajaradi.
* **DNS Keshlash (Caching):** Local DNS server olingan javobni o‘z xotirasida belgilangan vaqtga (**TTL — Time to Live**, masalan, 24 soat) saqlab qo‘yadi. Keyingi minglab foydalanuvchilar `google.com` ni so‘raganda, u Root yoki TLD larga bormasdan, o‘z keshidan bir necha millisekundda javob beradi!

---

### 2.4.4. DNS Yozuvlari Turlari (Resource Records — RR)

DNS ma’lumotlar bazasida saqlanadigan yozuvlar 4 ta maydondan iborat:  
`(Name, Value, Type, TTL)`

* **Type = A (Address):**
  * Domen nomini IPv4 manzilga bog‘laydi.  
  * Misol: `(relay1.bar.foo.com, 145.37.93.126, A, 86400)`
* **Type = AAAA:**
  * Domen nomini yangi 128-bitli IPv6 manzilga bog‘laydi.
* **Type = NS (Name Server):**
  * Ushbu domen uchun qaysi server Vakolatli (Authoritative) ekanini bildiradi.  
  * Misol: `(foo.com, dns.foo.com, NS, 86400)`
* **Type = CNAME (Canonical Name):**
  * Murakkab xost nomi uchun qisqa taxallus (Alias) yaratish.  
  * Misol: `(www.ibm.com, servereast.backup2.ibm.com, CNAME, 86400)`
* **Type = MX (Mail Exchange):**
  * Ushbu domenga keluvchi elektron xatlar aynan qaysi pochta serveriga borishi kerakligini ko‘rsatadi.  
  * Misol: `(gmail.com, alt1.gmail-smtp-in.l.google.com, MX, 86400)`

---

## 2.5. PEER-TO-PEER (P2P) FAYL TARQATISH MEXANIZMI

Katta hajmdagi faylni (masalan, 5 GB li Ubuntu Linux distributivini) 10 000 ta foydalanuvchiga tarqatish kerak. Qaysi arxitektura tezroq?

### Matematik Taqqoslash:
* $F$ — fayl hajmi (bit).
* $N$ — yuklab oluvchi mijozlar soni.
* $u_s$ — serverning ma’lumot yuklash (upload) tezligi.
* $d_i$ — $i$-mijozning yuklab olish (download) tezligi.

```
Fayl tarqatish
vaqti (D)
   ^
   |                                 /  Client-Server (Chiziqli o'sadi: NF/u_s)
   |                                /
   |                               /
   |                              /
   |                             /
   |             .............../
   |____________/____________________ P2P (Foydalanuvchilar ko'paysa ham vaqt deyarli o'zgarmaydi!)
   0---------------------------------> Mijozlar soni (N)
```

1. **Client-Server modelida:**
   * Server barcha $N$ ta nusxani o‘zi yuklashi kerak (umumiy yuklanadigan hajm: $N \times F$).
   * Vaqt: $D_{\text{cs}} \ge \frac{N \cdot F}{u_s}$. Agar $N$ 10 barobar oshsa, yuklash vaqti ham 10 barobar oshadi!
2. **P2P modelida (BitTorrent):**
   * Har bir mijoz faylning bir qismini olgach, o‘zining yuklash tezligi ($u_i$) bilan uni boshqalarga uzata boshlaydi.
   * Vaqt: $D_{\text{p2p}} \ge \frac{N \cdot F}{u_s + \sum u_i}$.
   * Yangi mijoz qo‘shilishi tarmoqqa nafaqat yuklama, balki **yangi uzatish quvvatini ham olib keladi**!

---

## 2.6. PYTHON SOCKET DASTURLASH (AMALIY MISOL)

Tarmoq ilovasining ishlashini to‘liq his qilish uchun Python da oddiy Client-Server dasturini yozamiz.

### 1. TCP Server (Ulanishli, ishonchli)
Server ulanishlarni kutadi, kelgan matnni katta harflarga o‘girib qaytaradi:

```python
# TCPServer.py
from socket import *

serverPort = 12000
# 1. TCP soket yaratamiz (AF_INET = IPv4, SOCK_STREAM = TCP)
serverSocket = socket(AF_INET, SOCK_STREAM)

# 2. Soketni portga bog'laymiz
serverSocket.bind(('', serverPort))

# 3. Ulanishlarni eshitishni boshlaymiz (navbat hajmi = 1)
serverSocket.listen(1)
print("TCP Server 12000-portda ulanishlarni kutmoqda...")

while True:
    # 4. Mijoz kelganda 3-Way Handshake bajariladi va yangi soket ochiladi
    connectionSocket, clientAddr = serverSocket.accept()
    print(f"Ulanish o'rnatildi: {clientAddr}")
    
    # 5. Ma'lumotni qabul qilamiz
    sentence = connectionSocket.recv(1024).decode()
    capitalizedSentence = sentence.upper()
    
    # 6. Javobni qaytaramiz va soketni yopamiz
    connectionSocket.send(capitalizedSentence.encode())
    connectionSocket.close()
```

### 2. TCP Client
Mijoz serverga ulanadi, matn yuboradi va javobni ekranga chiqaradi:

```python
# TCPClient.py
from socket import *

serverName = '127.0.0.1'  # Server IP manzili
serverPort = 12000

# 1. Mijoz TCP soketini yaratamiz
clientSocket = socket(AF_INET, SOCK_STREAM)

# 2. Server bilan 3-Way Handshake o'rnatamiz
clientSocket.connect((serverName, serverPort))

# 3. Foydalanuvchidan matn olib jo'natamiz
sentence = input("Kichik harflarda matn kiriting: ")
clientSocket.send(sentence.encode())

# 4. Serverdan javobni o'qiymiz
modifiedSentence = clientSocket.recv(1024).decode()
print("Serverdan javob:", modifiedSentence)

# 5. Soketni yopamiz
clientSocket.close()
```

---

## 2.7. 2-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. **Dastur qatlami** tarmoq arxitekturasining cho‘qqisi bo‘lib, Yakuniy tizimlarda soketlar orqali ishlaydi.
2. **HTTP** protokoli Web ning poydevori bo‘lib, ulanishlarni tejash uchun **Persistent HTTP (Keep-Alive)**, holatni saqlash uchun **Cookies**, tezlikni oshirish uchun esa **Kesh/Proksi** serverlardan foydalanadi.
3. **DNS** ierarxik tizimi inson tushunadigan nomlarni kompyuter tushunadigan raqamli IP larga aylantirib, butun global internet yo‘nalishini ta’minlaydi.

### 🔗 Joriy papkadagi laboratoriyalar bilan bog‘liqlik:
* Ushbu bobdagi DNSSEC, Authoritative serverlar, MX yozuvlari va E-mail arxitekturasi to‘g‘ridan-to‘g‘ri **`A7_project`** (DNSSEC, STARTTLS, SPF, DKIM, DMARC xavfsizligi) laboratoriyasining asosiy poydevoridir!
* HTTP xabarlarining tuzilishi, Status kodlar va Proksi mexanizmi esa **`A13_project`** (Apache Reverse Proxy va ModSecurity WAF) mashg‘ulotida xakerlik hujumlarini (SQL Injection, XSS) tahlil qilish uchun asos bo‘ladi!
