# 9-BOB: TARMOQ BOSHQARUVI VA MONITORING (Network Management)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Tarmoq boshqaruvining xalqaro modeli (ISO FCAPS: Nosozliklar, Konfiguratsiya, Hisob-kitob, Unumdorlik va Xavfsizlik), tarmoq boshqaruvi infratuzilmasi (NMS, Agent, MIB, SMI va OID daraxti), SNMP protokoli (SNMPv1, v2c va xavfsiz SNMPv3 arxitekturasi), Trap va Inform asinxron bildirishnomalari, zamonaviy telemetriya (NetFlow/IPFIX, Syslog) hamda tarmoq monitoringi tizimlarini chuqur o‘rganish.

---

## 9.1. TARMOQ BOSHQARUVI NIMA O‘ZI? ISO FCAPS MODELI

Minglab routerlar, kommutatorlar, serverlar, simsiz kirish nuqtalari va aloqa liniyalaridan iborat zamonaviy korporativ tarmoqni boshqarish inson qo‘li bilan (har bir qurilmaga birma-bir ulanib) amalga oshirilishi imkonsiz.

> **Rasmiy Ta’rif:**  
> **Tarmoq boshqaruvi** — bu tarmoqning barcha apparat va dasturiy resurslarini uzluksiz, xavfsiz, yuqori unumdorlikda va maqbul xarajatlar bilan ishlashini ta’minlash maqsadida ularni kuzatish (monitoring), tahlil qilish, sozlash (konfiguratsiya) va nazorat qilish choralarining avtomatlashtirilgan majmuasidir.

---

### Xalqaro ISO FCAPS Boshqaruv Modeli:

Xalqaro standartlashtirish tashkiloti (ISO) tarmoq boshqaruvini 5 ta asosiy funksional sohaga ajratadi:

```
+-------------------------------------------------------------------------+
|                           ISO FCAPS MODELI                              |
+-------------------------------------------------------------------------+
| F - Fault Management (Nosozliklarni boshqarish)                         |
|     Tarmoqdagi uzilishlar, apparat nosozliklarini zudlik bilan aniqlash,|
|     xabardor qilish, izolyatsiyalash va tuzatish.                       |
+-------------------------------------------------------------------------+
| C - Configuration Management (Konfiguratsiyani boshqarish)              |
|     Qurilmalar sozlamalarini avtomatlashtirish, dasturiy ta'minotni     |
|     yangilash (firmware update), zaxira nusxalarini (backup) olish.     |
+-------------------------------------------------------------------------+
| A - Accounting Management (Hisob-kitob va Resurslar auditi)             |
|     Tarmoq resurslaridan (trafik, portlar, xotira) kim, qachon va       |
|     qancha hajmda foydalanganini hisoblash va billing yuritish.         |
+-------------------------------------------------------------------------+
| P - Performance Management (Unumdorlikni boshqarish)                    |
|     Kechikishlar, paketlar yo'qolishi, aloqa liniyalarining bandlik     |
|     darajasi (Throughput) va xizmat sifatini (QoS) doimiy o'lchash.     |
+-------------------------------------------------------------------------+
| S - Security Management (Xavfsizlikni boshqarish)                       |
|     Tarmoqqa kirish huquqlarini tekshirish, kalitlar va sertifikatlarni |
|     boshqarish, xavfsizlik devori qoidalari va hujumlar loglari auditi. |
+-------------------------------------------------------------------------+
```

---

## 9.2. TARMOQ BOSHQARUVI INFRATUZILMASI KOMPONENTLARI

Tarmoq boshqaruv tizimi 4 ta asosiy komponentdan tashkil topadi:

```
+-------------------------------------------------------------------------+
|                  MANAGING ENTITY (NMS - Boshqaruv Serveri)              |
|                  (Zabbix, Nagios, PRTG, Cisco DNA Center)               |
|                  [ Ma'lumotlar Bazasi ]   [ Boshqaruvchi Dastur ]       |
+-------------------------------------------------------------------------+
                                     ^
                                     | SNMP Protokoli (UDP 161 / 162)
                                     v
+-------------------------------------------------------------------------+
|                  MANAGED DEVICE (Boshqariladigan Qurilma)               |
|                  (Router, Switch, Server, Firewall)                     |
|                                                                         |
|   +--------------------------+          +---------------------------+   |
|   | SNMP Agent (Dastur)      | <======> | MIB (Ma'lumotlar Bazasi)  |   |
|   +--------------------------+          +---------------------------+   |
|                 ^                                     ^                 |
|                 |                                     |                 |
|     [ Port 1 ] [ Port 2 ]                 [ CPU Yuklamasi, Harorat ]    |
+-------------------------------------------------------------------------+
```

1. **Managing Entity (Boshqaruvchi Server / NMS — Network Management Station):**
   * Tarmoq administratorining markaziy serveri.
   * Butun tarmoqdagi yuzlab qurilmalardan ma’lumotlarni yig‘adi, tahlil qiladi, grafiklar chizadi va xato yuz berganda administratorga signal beradi (Telegram, SMS, E-mail).
2. **Managed Device (Boshqariladigan Qurilma):**
   * Tarmoqda turgan har qanday nazorat qilinishi kerak bo‘lgan qurilma (Router, Switch, Server, Access Point, Firewall).
3. **Management Agent (Boshqaruv Agenti):**
   * Boshqariladigan qurilmaning ichida ishlaydigan maxsus dasturiy ta’minot (daemon/xizmat).
   * U NMS serveridan kelgan buyruqlarni bajaradi, apparat holatini tekshiradi va ma’lumotlarni qaytaradi.
4. **MIB (Management Information Base — Boshqaruv Ma’lumotlar Bazasi):**
   * Qurilma ichidagi barcha holat parametrlari (masalan, interfeys holati, kirgan baytlar soni, xotira hajmi) saqlanadigan standartlashtirilgan struktura.

---

## 9.3. INTERNET STANDART BOSHQARUV TIZIMI: SMI, MIB VA OID IERARXIYASI

Qurilma parametrlari turlicha bo‘lishi mumkin (biri Cisco, biri Juniper, biri Linux). Ularni qanday qilib yagona umumiy tilda tasvirlash mumkin?

### 9.3.1. SMI (Structure of Management Information)
SMI (RFC 2578) — bu MIB ichidagi ma’lumotlar qanday ta’riflanishi va qanday tiplarda saqlanishini belgilovchi qoidalardir. U **ASN.1 (Abstract Syntax Notation One)** tiliga asoslangan:
* Asosiy tiplar: `INTEGER`, `OCTET STRING` (matn yoki baytlar), `IPAddress`, `Counter32 / Counter64` (faqat o‘sib boruvchi hisoblagich, masalan o‘tgan baytlar soni), `Gauge32` (ko‘tarilib-tushuvchi son, masalan CPU harorati yoki foizi).

---

### 9.3.2. MIB va OID (Object Identifier) Ierarxik Daraxti

Dunyodagi barcha boshqariladigan ob’yektlar yagona global **Daraxtsimon ierarxiya (OID Tree)** ga joylashtiriladi. Har bir ob’yekt nuqtalar bilan ajratilgan raqamlar ketma-ketligi (**OID**) bilan belgilanadi:

```
                            iso (1)
                             |
                         org (3)
                             |
                         dod (6)
                             |
                       internet (1)
                             |
         +-------------------+-------------------+
         |                   |                   |
    mgmt (2)           security (5)        private (4)
         |                                       |
      mib-2 (1)                            enterprises (1)
         |                                       |
    +----+----+----+                       +-----+-----+
    |         |    |                       |           |
system (1)  ip (4) interfaces (2)      cisco (9)   mikrotik (14988)
```

#### Mashhur OID lar:
* `1.3.6.1.2.1` $\implies$ `iso.org.dod.internet.mgmt.mib-2` (Barcha standart qurilmalar uchun umumiy MIB-II).
* `1.3.6.1.2.1.1.1` $\implies$ `sysDescr` (Qurilmaning apparat va OS tavsifi).
* `1.3.6.1.2.1.1.3` $\implies$ `sysUpTime` (Qurilma yoqilganidan beri o‘tgan vaqt).
* `1.3.6.1.2.1.2.2.1.10.1` $\implies$ `ifInOctets.1` (1-portdan kirgan baytlar soni hisoblagichi).
* `1.3.6.1.4.1` $\implies$ `enterprises` (Kompaniyalar o‘zlarining xususiy mahsulotlari uchun shaxsiy MIB larni joylashtiradigan shoxcha).

---

## 9.4. SNMP PROTOKOLI (SIMPLE NETWORK MANAGEMENT PROTOCOL)

SNMP — bu NMS va Agent o‘rtasida MIB ma’lumotlarini o‘qish va o‘zgartirish uchun ishlatiladigan dastur darajasidagi (L7) protokoldir.
U transport qatlamida **UDP** dan foydalanadi:
* **UDP Port 161:** NMS dan Agentga so‘rovlar yuborish va javob olish uchun.
* **UDP Port 162:** Agent tomonidan NMS ga favqulodda bildirishnomalar (**Trap**) yuborish uchun.

---

### 9.4.1. SNMP Xabarlar Turlari (PDU — Protocol Data Units)

```
NMS (Boshqaruv Serveri)                                        SNMP Agent (Router)
         |                                                              |
         | 1. GetRequest (OID: ifInOctets.1)                            |
         |------------------------------------------------------------->| (Jadvaldan o'qiydi)
         |                                                              |
         | 2. Response (Qiymat: 458920194 bayt)                         |
         |<-------------------------------------------------------------|
         |                                                              |
         | 3. SetRequest (OID: ifAdminStatus.2 = 2 [DOWN])              |
         |------------------------------------------------------------->| (Portni o'chiradi!)
         |                                                              |
         | 4. Response (Status: Success)                                |
         |<-------------------------------------------------------------|
         |                                                              |
         |                                      [ KUTILMAGAN HODISA:    |
         |                                        Kabel uzildi! ]       |
         | 5. SNMP TRAP (Port 162 ga asinxron ogohlantirish!)           |
         |<=============================================================|
```

1. **GetRequest:** NMS agentdan bitta aniq OID qiymatini so‘raydi.
2. **GetNextRequest:** Daraxt bo‘ylab keyingi OID qiymatini olish (butun jadvalni qadamma-qadam aylanib chiqish uchun).
3. **GetBulkRequest (SNMPv2/v3):** Katta hajmdagi MIB jadvallarini bitta paketda tezkor yuklab olish.
4. **SetRequest:** NMS tomonidan agentdagi konfiguratsiya parametrini masofadan o‘zgartirish (masalan, router interfeysini yoqish yoki o‘chirish).
5. **Response:** Agentning NMS so‘roviga qaytargan ma’lumot javobi.
6. **SNMP Trap:** **Asinxron favqulodda xabar.** Agar routerda liniya uzilib qolsa yoki quvvat bloki kuyib ketsa, agent NMS ning so‘rovini kutib o‘tirmaydi, balki zudlik bilan 162-portga `Trap` otadi!
7. **InformRequest:** Trap ning tasdiqlanadigan (qabul qilingani haqida ACK talab qiluvchi) xavfsiz versiyasi.

---

### 9.4.2. SNMP Versiyalari va Xavfsizlik Evolyutsiyasi

```
+-------------------------------------------------------------------------+
|                           SNMP VERSIYALARI                              |
+-------------------------------------------------------------------------+
| SNMPv1 (Eskirgan va xavfli)                                             |
| * Autentifikatsiya: Ochiq matnli "Community String" (masalan, "public") |
| * Shifrlash yo'q! Parollar tarmoqda ochiq oqadi.                        |
+-------------------------------------------------------------------------+
| SNMPv2c (Eng keng tarqalgan, lekin xavfsiz emas)                        |
| * Yangi GetBulkRequest va 64-bitli hisoblagichlar (Counter64) qo'shildi.|
| * Lekin xavfsizlik yana o'sha ochiq Community String ligicha qolgan!    |
+-------------------------------------------------------------------------+
| SNMPv3 (Zamonaviy xavfsiz standart — RFC 3410 - 3418)                   |
| * USM (User-based Security Model): Haqiqiy login va parollar.           |
| * Autentifikatsiya: SHA-256 xesh tekshiruvi (Soxtalashtirish yo'q).     |
| * Maxfiylik (Privacy): Barcha SNMP paketlar AES-256 bilan to'liq        |
|   shifrlanadi (Sniffing qilish befoyda!).                               |
| * VACM (View-based Access Control): Kim qaysi OID ni ko'ra olishini     |
|   qat'iy cheklash.                                                      |
+-------------------------------------------------------------------------+
```

> **Xavfsizlik Ogohlantirishi:**  
> Agar korxona tarmog‘ida `SNMPv1` yoki `SNMPv2c` ishlatilsa va uning `Community String` i o‘zgartirilmasdan sukut bo‘yicha `public` (o‘qish) yoki `private` (yozish) qoldirilsa, tarmoqdagi har qanday xaker Wireshark orqali bu parolni ko‘rib oladi va `SetRequest` yuborib, butun tashkilotning asosiy routerlarini masofadan o‘chirib qo‘yishi mumkin!  
> **Korporativ talab:** Faqat **SNMPv3 (authPriv)** rejimini qo‘llash shart!

---

## 9.5. ZAMONAVIY TARMOQ MONITORINGI: NETFLOW, SYSLOG VA SIEM

Katta korporativ tarmoqlarda faqat SNMP ning o‘zi yetarli emas. Zamonaviy monitoring uchta ustunga tayanadi:

```
                            ZAMONAVIY TARMOQ TELEMETRIYASI
                                           |
         +---------------------------------+---------------------------------+
         |                                 |                                 |
         v                                 v                                 v
   SNMPv3 (Metrikalar)             NETFLOW / IPFIX (Oqimlar)        SYSLOG (Voqealar logi)
* Qurilma holati                   * Kim? Kim bilan? Qachon?        * Tizim hodisalari
* CPU, RAM, Harorat                * 5-lik kortej statistikasi      * "Port 1 down bo'ldi"
* Portlarning o'tgan baytlari      * O'tkazilgan trafik hajmi       * "SSH login muvaffaqiyatsiz"
```

---

### 9.5.1. NetFlow va IPFIX (Oqimlar Monitoringi)

SNMP har bir interfeysdan qancha bayt o‘tganini aytadi, lekin **o‘sha baytlarning ichida kim nima qilayotganini** bilmaydi.
* **Oqim (Flow) tushunchasi:** Tarmoqda yagona umumiy **5-lik kortej** ga ega bo‘lgan paketlar ketma-ketligi:
  $$\{ \text{Source IP}, \text{Dest IP}, \text{Source Port}, \text{Dest Port}, \text{Protocol} \}$$
* Router har bir ochilgan oqim uchun hisob yuritadi (qancha bayt o‘tdi, qancha paket, qachon boshlandi, qachon tugadi) va oqim yopilgach, yig‘ilgan statistikani **NetFlow Collector (Masalan, nfdump / Elasticsearch)** ga yuboradi.
* **Xavfsizlikdagi o‘rni:** Xaker tarmoqdan maxfiy ma’lumotlarni o‘g‘irlayotganini (Data Exfiltration) yoki ichki tarmoqda kimdir portlarni skanerlayotganini NetFlow oqimlari orqali darhol ko‘rish mumkin.

---

### 9.5.2. Syslog Protokoli (RFC 5424)

Tarmoq qurilmalari o‘zlarida yuz bergan har bir muhim voqeani (Event) markaziy **Syslog Server (UDP Port 514)** ga yozib boradi.

#### Syslog Jiddiylik Darajalari (Severity Levels: 0 dan 7 gacha):
* `0 - Emergency`: Tizim butunlay ishdan chiqdi.
* `1 - Alert`: Zudlik bilan chora ko‘rish shart.
* `2 - Critical`: Kritik xatolik (masalan, apparat moduli quladi).
* `3 - Error`: Xatolik holati.
* `4 - Warning`: Ogohlantirish.
* `5 - Notice`: Normal, lekin muhim hodisa (masalan, interfeys holati o‘zgardi).
* `6 - Informational`: Oddiy axborot xabari (masalan, foydalanuvchi tizimga kirdi).
* `7 - Debug`: Dasturchilar uchun eng quyi diagnostika loglari.

---

## 9.6. 9-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. **Tarmoq boshqaruvi** ISO FCAPS modeli bo‘yicha nosozliklar, konfiguratsiya, resurslar, unumdorlik va xavfsizlikni boshqaradi.
2. **SMI va MIB** butun dunyodagi qurilma parametrlarini daraxtsimon **OID** kodlari bilan standartlashtiradi.
3. **SNMPv3** autentifikatsiya (SHA) va shifrlash (AES) orqali qurilmalarni masofadan xavfsiz boshqarish imkonini beradi.
4. **NetFlow/IPFIX va Syslog** tarmoqdagi xavfsizlik hodisalarini kuzatish, anomaliyalarni fosh qilish va kiber-tergovlar o‘tkazishning tayanchidir.

---

### 🔗 JORIY PAPKADAGI BARCHA LABORATORIYALAR BILAN BEVOSITA INTEGRATSIYA:

* **`A9_project`:** Wireshark va **NetFlow (softflowd, nfdump)** yordamida paketlar va oqimlarni o‘zaro taqqoslab, tarmoq monitoringini amalga oshirish!
* **`A16_project`:** Virtual routerni xavfsiz sozlash (**Hardening**), boshqaruv xizmatlarini (SSH, SNMP) cheklash va xavfsizlik auditini o‘tkazish!
* **`A17_project`:** Tarmoq xavfsizligi insidentlarini boshqarish (**Incident Response**) — tarmoq loglari va Syslog hodisalarini tahlil qilib, buzib kirilgan manbani aniqlash, dalillarni to‘plash va xizmatni qayta tiklash!
