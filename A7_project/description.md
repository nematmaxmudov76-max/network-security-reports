# A7: DNS va Elektron Pochta Xavfsizligini Tekshirish
### 1-Variant: DNSSEC, STARTTLS va SPF–DKIM–DMARC Dalillarini Korrelyatsiyalash
> **Fayl haqida eslatma:** Mazkur qo‘llanma joriy ishchi papkadagi **`A7_Tarmoq_xavfsizligi_VMware_proxy.docx`** hujjatining to‘liq tahlili asosida tuzildi. Hujjatda berilgan barcha talablar, konfiguratsiyalar, xavfsizlik qoidalari, amaliy ssenariylar va baholash mezonlari batafsil umumlashtirildi.
>
> **Muhim muhit va tarmoq eslatmasi:** Qo‘llanma **Ubuntu Linux xost tizimida**, **VMware Workstation Pro for Linux** hamda **shaxsiy lokal Wi-Fi tarmog‘ida** ishlash uchun to‘liq moslashtirildi:
> - **Xost va gipervizor:** Ubuntu Linux (Host) -> VMware Workstation Pro for Linux -> 3 ta Ubuntu Server 24.04 LTS VM.
> - **Kataloglar va buyruqlar:** Windows yo‘llari va PowerShell o‘rniga Linux xost kataloglari (`~/A7-LAB/...`), `7z / p7zip-full`, `sha256sum` va Linux Bash buyruqlari qo‘llanildi.
> - **Tarmoq:** Universitetning ichki proksisi (`172.18.1.1:2500`) talab etilmaydi — barcha paketlarni o‘rnatish xostning lokal Wi-Fi interneti orqali to‘g‘ridan-to‘g‘ri (proksisiz) rejimda xatosiz ishlaydi.

---

## MUNDARIJA
1. [1-QISM: Mavzuning Maqsadi va Kutiladigan Natijalar](#1-qism-mavzuning-maqsadi-va-kutiladigan-natijalar)
2. [2-QISM: Har Bir Mavzu Bo‘yicha Mukammal Nazariy-Amaliy Qo‘llanma](#2-qism-har-bir-mavzu-boyicha-mukammal-nazariy-amaliy-qollanma)
   - 2.1. Izolyatsiyalangan Tarmoq va Virtual Muhit Topologiyasi
   - 2.2. BIND9 Authoritative DNS va Hardening Siyosati
   - 2.3. DNSSEC (Domain Name System Security Extensions) Arxitekturasi
   - 2.4. Elektron Pochta Autentifikatsiya Qatlamlari: SPF, DKIM, DMARC
   - 2.5. Postfix SMTP Server: STARTTLS Majburiyligi va Relay Nazorati
   - 2.6. OpenDKIM Milter Integratsiyasi va Kalitlar Xavfsizligi
   - 2.7. PCAP Tarmoq Trafik Tahlili va Korrelyatsiya
   - 2.8. Qoldiq Risklar (Residual Risks) va Cheklovlar
3. [3-QISM: Mashqni Bajarish Ketma-Ketligi (Step-by-Step Amaliy Qo‘llanma)](#3-qism-mashqni-bajarish-ketma-ketligi-step-by-step-amaliy-qollanma)
   - 3.0. 0-Bosqich: Dastlabki Tayyorgarlik (Obrazlar, Lokal Wi-Fi, Paketlar, Snapshot)
   - 3.1. 1-Bosqich: Tarmoq Sozlamalari, Statik IP va Hostname Konfiguratsiyasi
   - 3.2. 2-Bosqich: MAIL-SRVda DKIM Kalit Juftini Yaratish
   - 3.3. 3-Bosqich: DNS-SRVda BIND9 va a7.test Zonasini Sozlash (DNSSEC bilan)
   - 3.4. 4-Bosqich: CLIENT orqali DNS, DNSSEC va Salbiy ACL Testlari
   - 3.5. 5-Bosqich: MAIL-SRVda Postfix va Majburiy STARTTLS Konfiguratsiyasi
   - 3.6. 6-Bosqich: OpenDKIMni Postfix Milteriga Bog‘lash
   - 3.7. 7-Bosqich: CLIENT orqali SMTP, STARTTLS va Relay Testlari (T3, T4, T5)
   - 3.8. 8-Bosqich: SPF, DKIM va DMARC Testlari (T6, T7 va Alignment)
   - 3.9. 9-Bosqich: PCAP Trafik Tahlili (tcpdump va tshark)
   - 3.10. 10-Bosqich: Audit Fayllarini Eksport Qilish, SHA-256 va Rollback
   - 3.11. Natijalarni Qayd Etish Jadvallari va Troubleshooting (Nosozliklarni Tuzatish)
   - 3.12. O‘qituvchi Uchun Himoya Savollari va Ularning Mukammal Javoblari
   - 3.13. Talabaning Yakuniy Xulosasi (Namuna)
4. [4-QISM: Rasmiy va Ishonchli Manbalar](#4-qism-rasmiy-va-ishonchli-manbalar)

---

## 1-QISM: MAVZUNING MAQSADI VA KUTILADIGAN NATIJALAR

### 1.1. Laboratoriya Mashg‘ulotining Maqsadi
Ushbu laboratoriya ishining asosiy maqsadi — talabaga to‘liq izolyatsiyalangan virtual muhitda zamonaviy tarmoq xizmatlari (DNS va SMTP) xavfsizligini ta’minlash, ularga nisbatan amalga oshirilishi mumkin bo‘lgan keng tarqalgan kiberhujumlar (DNS Spoofing/Cache Poisoning, DNS Amplification, Zone Reconnaissance, Man-in-the-Middle, E-mail Spoofing, Spam Relay)ning oldini olish hamda xavfsizlik nazorati mexanizmlarini joriy etishni amaliy o‘rgatishdan iborat.

Mashg‘ulot doirasida talaba:
1. **BIND9** asosida xavfsiz `authoritative` DNS serverini tashkil qiladi, unga **DNSSEC** imzolash siyosatini tatbiq etadi, keshlovchi rekursiyani (`recursion no`) va ruxsatsiz zona nusxalashni (`allow-transfer { none; }`) taqiqlaydi.
2. **Postfix** yordamida elektron pochta xizmatini yo‘lga qo‘yadi, 25-portda **majburiy STARTTLS** shifrlashini o‘rnatadi, ruxsatsiz tashqi relay (`reject_unauth_destination`) urinishlarini qat’iy bloklaydi.
3. **OpenDKIM** milterini sozlab, chiqish xatlariga 2048-bitli kriptografik raqamli imzo qo‘yishni ta’minlaydi.
4. DNS tizimiga **SPF (v=spf1)**, **DKIM (a7._domainkey)** va **DMARC (v=DMARC1)** yozuvlarini kiritib, pochta yuboruvchi serverning haqiqiyligi va domen identifikatsiyasining qat’iy mosligini (strict alignment) tahlil qiladi.
5. Xavfsizlik mexanizmlarining samaradorligini faqat deklarativ ko‘rinishda emas, balki **SMTP javob kodlari (530, 250, 554)**, **DNS bayroqlari (AA, RRSIG, DNSKEY, REFUSED)**, **tizim loglari** hamda **PCAP paketlar tahlili** orqali o‘zaro korrelyatsiyalash asosida ilmiy-amaliy tarzda isbotlaydi.

### 1.2. Kutiladigan Natijalar
Mashg‘ulot yakunida talaba quyidagi aniq natijalarni amalda ko‘rsata olishi shart:
- **DNS bo‘yicha:** DNS so‘roviga authoritative javob qaytganini (`status: NOERROR`, `flags: aa`), zona DNSKEY va RRSIG yozuvlari bilan imzolanganini ko‘rsatadi; begona domen uchun rekursiv so‘rov berilganda `status: REFUSED` qaytishini hamda ruxsatsiz AXFR so‘rovi bloklanishini isbotlaydi.
- **STARTTLS bo‘yicha:** Shifrlanmagan (oddiy matnli) SMTP sessiyasi orqali xat yuborishga urinilganda Postfix `530 5.7.0 Must issue a STARTTLS command first` kodi bilan rad etishini; STARTTLS faollashganda esa xabar `250 2.0.0 Ok: queued as ...` kodi bilan qabul qilinishini qayd etadi.
- **Relay himoyasi bo‘yicha:** Mijoz ruxsat berilmagan tashqi domenga xat yuborishga (open relay) uringanida Postfix `554 5.7.1 Relay access denied` xatosi bilan rad etishini ko‘rsatadi.
- **Pochta autentifikatsiyasi bo‘yicha:** Yetkazilgan xat sarlavhasida haqiqiy `DKIM-Signature` mavjudligini, ruxsat berilgan IP (10.70.10.20) uchun SPF `pass`, ruxsat etilmagan IP (10.70.10.30) uchun SPF `fail` qaytishini, DMARC qat’iy siyosati (`p=reject; adkim=s; aspf=s`) mazmunini tushuntirib beradi.
- **PCAP tahlili bo‘yicha:** `tcpdump`/`tshark` yordamida yozib olingan tarmoq trafigida STARTTLS dan keyingi SMTP muloqoti (xat mavzusi, matni, jo‘natuvchi ma’lumotlari) shifrlangan TLS tunnel ichida bo‘lib, ochiq matn (plaintext) ko‘rinishida oqib ketmasligini isbotlaydi.

### 1.3. Laboratoriya Ko‘rsatkichlari va Cheklovlari
| Ko‘rsatkich | Standart Qiymati | Izoh |
| :--- | :--- | :--- |
| **Bajarish vaqti** | 80 daqiqa | Barcha VM va paketlar tayyor holatda |
| **Dastlabki tayyorgarlik** | 60–90 daqiqa | VM nusxalarini ajratish, lokal Wi-Fi orqali to‘g‘ridan-to‘g‘ri (proksisiz) paketlarni o‘rnatish |
| **Xost tizim** | Ubuntu Linux (Host) + VMware Workstation Pro for Linux | Gipervizor muhiti |
| **Virtual mashinalar** | 3 ta VM: DNS-SRV, MAIL-SRV, CLIENT | Ubuntu Server 24.04 LTS |
| **Virtual tarmoq** | A7-LAB (`10.70.10.0/24`) | VMware LAN Segment (DHCP va Gateway yo‘q, izolyatsiyalangan) |
| **Lokal domen** | `a7.test` | RFC 6761 bo‘yicha maxsus sinov domeni |
| **Asosiy portlar** | DNS: 53/UDP, 53/TCP; SMTP: 25/TCP | Tashqi tarmoqqa chiqish qat’iyan taqiqlangan |
| **Baholash** | Maksimal 10 ball | Rubrika va mezonlar asosida |

---

## 2-QISM: HAR BIR MAVZU BO‘YICHA MUKAMMAL NAZARIY-AMALIY QO‘LLANMA

### 2.1. Izolyatsiyalangan Tarmoq va Virtual Muhit Topologiyasi
Laboratoriya xavfsizligining eng birinchi talabi — **qat’iy tarmoq izolyatsiyasi**dir. Mazkur ishda real domenlar, universitetning ichki tarmoqlari yoki tashqi Internet infratuzilmasiga hech qanday so‘rov yuborilmaydi va ulardan so‘rov qabul qilinmaydi.

Barcha arxitektura **Ubuntu Linux xost operatsion tizimi**, uning ichidagi **VMware Workstation Pro for Linux** hamda uning ichida ishlovchi **3 ta Ubuntu Server 24.04 VM** ustiga quriladi:

```
+----------------------------------------------------------------------------------+
|                       UBUNTU LINUX (XOST OPERATSION TIZIM)                       |
|                     VMware Workstation Pro / Player for Linux                    |
|                                                                                  |
|  +----------------------------------------------------------------------------+  |
|  |                     A7-LAB (Izolyatsiyalangan LAN Segment)                 |  |
|  |                               10.70.10.0/24                                |  |
|  +----------------------------------------------------------------------------+  |
|        |                               |                              |          |
|        |                               |                              |          |
|  +---------------+             +---------------+              +---------------+  |
|  |    DNS-SRV    |             |   MAIL-SRV    |              |    CLIENT     |  |
|  |  10.70.10.10  |             |  10.70.10.20  |              |  10.70.10.30  |  |
|  |  dns.a7.test  |             |  mail.a7.test |              | client.a7.test|  |
|  |               |             |               |              |               |  |
|  | • BIND9       |             | • Postfix     |              | • dig / drill |  |
|  | • DNSSEC      |             | • OpenDKIM    |              | • swaks       |  |
|  | • Auth Zone   |             | • STARTTLS    |              | • spfquery    |  |
|  | • No Recursion|             | • Port 25     |              | • tcpdump     |  |
|  +---------------+             +---------------+              +---------------+  |
+----------------------------------------------------------------------------------+
```

- **LAN Segment afzalligi:** VMware LAN Segment virtual shina tarzida ishlaydi. Unda xost operatsion tizimi bilan umumiy virtual adapter (`vnet`) bo‘lmaydi, ichki DHCP server ishlamaydi va hech qanday tashqi marshrutlash (routing/NAT) mavjud emas. Bu holat tajribalarning tozaligini va tasodifiy paketlarning tashqariga chiqib ketmasligini ta’minlaydi.
- **NAT adapteri:** Faqat 0-bosqichda operatsion tizim paketlarini (`apt update`, `apt install`) xost kompyuterning **lokal Wi-Fi tarmog‘i** orqali to‘g‘ridan-to‘g‘ri (proksisiz) yuklab olish uchun ishlatiladi. Paketlar to‘liq yuklanishi bilan NAT adapteri sozlamalardan uziladi (`Disconnect` qilinadi). Universitet ichki tarmog‘ida bo‘lmaganingiz sababli universitet proksisi (`172.18.1.1:2500`) mutlaqo ishlatilmaydi.

### 2.2. BIND9 Authoritative DNS va Hardening Siyosati
DNS (Domain Name System) Internetning asab tizimi bo‘lib, nomlarni IP manzillarga bog‘laydi. Ammo an’anaviy DNS protokoli (UDP/53) shifrlanmagan va autentifikatsiyalanmagan bo‘lgani uchun bir qator zaifliklarga ega:

1. **Authoritative (Vakolatli) Server nima?**
   Muayyan zona (bizning holatda `a7.test`) uchun rasmiy va birlamchi javob beruvchi server. U qaytargan javoblarda `AA` (*Authoritative Answer*) bayrog‘i o‘rnatilgan bo‘ladi. Bu javob keshdan emas, to‘g‘ridan-to‘g‘ri zonaning asl ma’lumotlar bazasidan kelganini bildiradi.
2. **Recursion no (Rekursiyani o‘chirish):**
   Keshlovchi rekursiv DNS serverlar butun dunyo bo‘ylab boshqa DNS serverlardan ma’lumot qidiradi. Agar authoritative serverda rekursiya ochiq qoldirilsa, u **Open Resolver**ga aylanadi. Hujumchilar soxta manba IP (IP spoofing) bilan so‘rov yuborib, serverni **DNS Amplification (Kuchaytirilgan DDoS)** hujumlarida qurol sifatida ishlatishi yoki keshni zaharlashi (Cache Poisoning) mumkin. `recursion no;` ko‘rsatmasi server faqat o‘ziga tegishli zonalarga javob berishini kafolatlaydi.
3. **allow-transfer { none; } (AXFR blokirovkasi):**
   `AXFR` (Zone Transfer) butun zona yozuvlarini bitta so‘rov bilan nusxalab olish mexanizmidir (odatda Primary va Secondary serverlar o‘rtasida sinxronizatsiya uchun kerak). Agar AXFR cheklanmasa, har qanday tashqi foydalanuvchi butun tarmoq xaritasini, ichki serverlar IP manzillarini bitta buyruq bilan qo‘lga kiritadi (**Reconnaissance / Axborot yig‘ish**). `allow-transfer { none; }` bu xavfni to‘liq bartaraf etadi.

### 2.3. DNSSEC (Domain Name System Security Extensions) Arxitekturasi
DNSSEC (RFC 4033, 4034, 4035) DNS javoblarining yaxlitligi (integrity) va manba haqiqiyligini (authenticity) asimmetrik kriptografiya orqali kafolatlaydi.
- **RRSIG (Resource Record Signature):** Har bir Resource Record Set (RRset — masalan, barcha A yozuvlari yoki MX yozuvlari) serverning maxfiy kaliti (ZSK - Zone Signing Key) bilan imzolanadi va DNS javobida RRSIG ko‘rinishida birga qaytariladi.
- **DNSKEY:** Zonani tekshirish uchun ochiq kalitlar (ZSK va KSK - Key Signing Key) saqlanadigan yozuv. Mijoz resolveri ushbu ochiq kalit yordamida RRSIG imzosini tekshiradi.
- **DS (Delegation Signer):** Yuqori darajadagi zona (masalan, `.test` yoki ildiz zonalar)ga kiritiladigan KSK xeshi. U ishonch zanjirini (**Chain of Trust**) hosil qiladi.
- **Laboratoriya cheklovi:** Bizning laboratoriyamizda `.test` domeni mahalliy bo‘lgani va Internet ildiz (Root) serverlariga ulanmaganligi sababli, global DS zanjiri mavjud bo‘lmaydi. Ammo lokal zona doirasida `inline-signing yes;` va `dnssec-policy default;` orqali to‘liq RRSIG va DNSKEY yozuvlari hosil qilinadi.

### 2.4. Elektron Pochta Autentifikatsiya Qatlamlari: SPF, DKIM, DMARC
Elektron pochta (SMTP) protokoli 1982-yilda yaratilganida jo‘natuvchini tekshirish mexanizmiga ega bo‘lmagan. Har kim xohlagan nomdan (masalan, `ceo@company.com`) xat yuborishi mumkin edi. Bunga qarshi zamonaviy uchlik ishlab chiqilgan:

```
+----------------------------------------------------------------------------------+
|                              DMARC (RFC 9989)                                    |
|   Siyosat: p=reject | Qat'iy moslik: adkim=s; aspf=s                             |
|   Tekshiradi: Header From (Foydalanuvchi interfeysida ko'rinadigan jo'natuvchi)  |
+----------------------------------------+-----------------------------------------+
                                         |
               +-------------------------+-------------------------+
               |                                                   |
               v                                                   v
+-------------------------------+               +----------------------------------+
|         SPF (RFC 7208)        |               |          DKIM (RFC 6376)         |
|  Tekshiradi: Envelope Sender  |               |  Tekshiradi: Kriptografik raqamli|
|  (Return-Path / MAIL FROM)    |               |  imzo va xabar tanasi yaxlitligi |
|  Mexanizm: ip4:10.70.10.20    |               |  Teglar: d=a7.test; s=a7         |
|  Siyosat: -all (Fail)         |               |  Kalit: RSA 2048-bit (DNS TXT)   |
+-------------------------------+               +----------------------------------+
```

1. **SPF (Sender Policy Framework - RFC 7208):**
   - Domen egasi DNS TXT yozuvida o‘z nomidan qaysi IP manzillar xat yuborish huquqiga ega ekanini e’lon qiladi:
     `v=spf1 ip4:10.70.10.20 -all`
   - Bu yerda `ip4:10.70.10.20` — ruxsat etilgan pochta serveri IPsi.
   - `-all` (*Fail*) — boshqa har qanday IP manzil qat’iyan rad etilishi kerakligini bildiradi (`~all` esa SoftFail bo‘ladi).
   - **Zaifligi:** SPF faqat SMTP konvertidagi `MAIL FROM` (Envelope Sender / Return-Path) manzilini tekshiradi, lekin foydalanuvchining pochta dasturida ko‘rinadigan `From:` (Header From) sarlavhasini himoya qila olmaydi.
2. **DKIM (DomainKeys Identified Mail - RFC 6376):**
   - Xat yuboruvchi MTA (Mail Transfer Agent) xabar sarlavhasi va tanasining xeshini oladi hamda uni o‘zining maxfiy kaliti (Private Key) bilan imzolaydi. Bu imzo xat sarlavhasiga `DKIM-Signature:` bo‘lib qo‘shiladi.
   - Qabul qiluvchi server DNS orqali jo‘natuvchi domendan ochiq kalitni (`selector._domainkey.domain`) yuklab oladi va imzoni tekshiradi.
   - Bu xat yo‘lda o‘zgartirilmaganini (Integrity) va ko‘rsatilgan domenga tegishli serverdan chiqqanini kafolatlaydi.
3. **DMARC (Domain-based Message Authentication, Reporting and Conformance - RFC 9989):**
   - SPF va DKIMni birlashtiruvchi va `Header From` manzilini himoya qiluvchi boshqaruv mexanizmi.
   - **Identifier Alignment (Identifikatorlar mosligi):** DMARC xatning `Header From` domeni bilan SPF tekshirgan domen va DKIM `d=` domeni bir xil bo‘lishini talab qiladi.
     * `aspf=s` (strict SPF): Header From aynan MAIL FROM ga teng bo‘lishi shart (subdomenlarga ruxsat berilmaydi).
     * `adkim=s` (strict DKIM): Header From aynan `d=` domeniga teng bo‘lishi shart.
   - **Siyosat (`p=`):** Agar tekshiruv muvaffaqiyatsiz bo‘lsa: `p=none` (faqat hisobot), `p=quarantine` (spampa tashlash), `p=reject` (butunlay qabul qilmaslik).

### 2.5. Postfix SMTP Server: STARTTLS Majburiyligi va Relay Nazorati
Elektron pochta xavfsizligida transport qatlami va ruxsat berish (authorization) qatlami ajratiladi:
- **STARTTLS Majburiyligi (`smtpd_tls_security_level = encrypt`):**
  Odatiy SMTP protokoli ochiq matn (plaintext) orqali ishlaydi. STARTTLS buyrug‘i ochiq TCP ulanishini TLS bilan shifrlangan kanalga aylantiradi. Agar serverda TLS majburiy qilinsa, mijoz STARTTLS komandasini bajarmasdan turib `MAIL FROM` yoki `RCPT TO` buyruqlarini bera olmaydi va server `530 5.7.0 Must issue a STARTTLS command first` xatosini qaytaradi. Bu orqali tarmoqda paketlarni tutib oluvchi shaxs (Sniffer/MitM) xat matnini ko‘ra olmaydi.
- **Relay Nazorati (`smtpd_relay_restrictions = permit_mynetworks, reject_unauth_destination`):**
  Spamerlar ko‘pincha himoyalanmagan SMTP serverlardan foydalanib, butun dunyoga o‘zlarining spam xabarlarini tarqatadilar (**Open Relay**). Postfixda faqat mahalliy domenlar (`mydestination = $myhostname, localhost, $mydomain`) uchun xat qabul qilinadi. Boshqa tashqi domenlarga (masalan, `victim@external.test`) relay qilish faqat `mynetworks` (ishonchli tarmoq) ga ruxsat etiladi. Biz laboratoriyada `mynetworks = 127.0.0.0/8` qilganimiz sababli, CLIENT (10.70.10.30) tashqi manzilga relay qila olmaydi va `554 5.7.1 Relay access denied` javobini oladi.

### 2.6. OpenDKIM Milter Integratsiyasi va Kalitlar Xavfsizligi
- **Milter (Mail Filter):** Postfix bilan tashqi xavfsizlik filtrlari o‘rtasidagi interfeys. OpenDKIM `inet:127.0.0.1:8891` soketida tinglaydi.
- **Fail-Secure Prinsipi (`milter_default_action = tempfail`):**
  Agar OpenDKIM xizmati to‘xtab qolsa yoki ishlamay qolsa, Postfix kelgan xatni imzosiz jo‘natib yubormaydi (bunday holat xavfsizlik bypassiga sabab bo‘ladi). Aksincha, xabarga `451 / tempfail` qaytarib, milter qayta tiklanguncha xatni ushlab turadi.
- **Kalitlar boshqaruvi (Key Hygiene):**
  DKIM private kaliti (`a7.private`) va TLS private kaliti (`a7-mail.key`) faqat `chmod 0600` ruxsatiga ega bo‘lishi, faqat tegishli tizim hisoblari (`opendkim`, `root`) tasarrufida bo‘lishi shart. Maxfiy kalit qiymatlari hech qachon DNSga, ommaviy hisobotlarga yoki skrinshotlarga chiqarilmaydi.

### 2.7. PCAP Tarmoq Trafik Tahlili va Korrelyatsiya
Tarmoq auditi dalillarni chuqur tahlil qilishni talab qiladi:
- **DNS trafigi:** UDP 53 portida DNS paketlari shifrlanmagan holda o‘tadi (DNS nomlari, so‘ralgan yozuv turlari ko‘rinadi). Bu an’anaviy DNS tabiatini va nima uchun ishlab chiqarishda DoT (DNS over TLS) yoki DoH (DNS over HTTPS) kerakligini ko‘rsatadi.
- **SMTP/TLS trafigi:** Dastlabki TCP handshake va SMTP banner (`220 mail.a7.test ESMTP Postfix`), mijozning `EHLO` va `STARTTLS` buyruqlari ochiq ko‘rinadi. Ammo `STARTTLS`dan so‘ng boshlangan TLS Handshake yakunlangach, barcha keyingi SMTP komandalari (`MAIL FROM`, `RCPT TO`, `DATA`) va xat tanasi Application Data sifatida shifrlanadi. `tshark` yordamida xat matni qidirilganda hech qanday moslik chiqmasligi transport shifrlanishining qat’iy isbotidir.

### 2.8. Qoldiq Risklar (Residual Risks) va Cheklovlar
Laboratoriyada joriy etilgan choralar muhim xavfsizlik poydevorini yaratsa-da, real tizimlarda quyidagi qoldiq risklar mavjud:
1. **Self-signed sertifikat:** Laboratoriyada yaratilgan TLS sertifikati o‘z-o‘zidan imzolangan (self-signed). Brauzerlar yoki ommaviy MTA lar unga ishonmaydi (Untrusted CA). Real ishlab chiqarishda Let's Encrypt kabi ishonchli PKI yoki DANE (TLSA yozuvlari) talab etiladi.
2. **DMARC ijrosi (Enforcement):** DMARC TXT yozuvini DNSga qo‘yish bilan ish tugamaydi. Xatni qabul qiluvchi tomon (Receiver MTA) o‘zida DMARC tekshiruvchi milter (masalan, OpenDMARC yoki Rspamd) o‘rnatgan bo‘lishi shart, aks holda `p=reject` siyosati bajarilmay qoladi.
3. **MTA-STS va TLS Downgrade:** STARTTLS "opportunistic" tabiatga ega bo‘lishi mumkin (agar qat’iy majburlanmasa). Tajovuzkor tarmoqda STARTTLS taklifini o‘chirib tashlashi (STRIPTLS hujumi) mumkin. Bunga qarshi MTA-STS (RFC 8461) va DANE qo‘llaniladi.

---

## 3-QISM: MASHQNI BAJARISH KETMA-KETLIGI (STEP-BY-STEP AMALIY QO‘LLANMA)

### 3.0. 0-Bosqich: Dastlabki Tayyorgarlik (Obrazlar, Lokal Wi-Fi, Paketlar, Snapshot)

> [!IMPORTANT]
> **TARMOQ VA XOST MUHITI (UBUNTU XOST + LOKAL WI-FI):**
> Siz mazkur laboratoriyani **Ubuntu Linux xost tizimida**, **VMware Workstation Pro for Linux** yordamida va **shaxsiy lokal Wi-Fi tarmog‘ingizda** bajarasiz. Shu sababli:
> 1. Universitetning ichki proksi serveri (`172.18.1.1:2500`) **mutlaqo kerak emas va ishlatilmaydi**! Agar VM ichiga ushbu proksi yozilsa, tizim tashqi internetga chiqa olmaydi va `apt update` xatolik beradi.
> 2. Universitet ichki fayl serveri (`\\172.18.233.234\files`) lokal Wi-Fi orqali to‘g‘ridan-to‘g‘ri ochilmaydi. Shuning uchun kerakli obrazlar xost kompyuteringizdagi katalogda tayyorlanadi.
> 3. Barcha yo‘llar Windows disklari (`D:\...`) emas, balki Linux xost yo‘li (`~/A7-LAB/...`) bo‘yicha amalga oshiriladi.

#### 0.1. Xostda kataloglarni yaratish va obrazlarni tayyorlash
Ubuntu xost terminalida laboratoriya uchun kerakli kataloglarni oching:
```bash
# Ubuntu xost terminalida bajaring:
mkdir -p ~/A7-LAB/downloads ~/A7-LAB/report ~/A7-LAB/DNS-SRV ~/A7-LAB/MAIL-SRV ~/A7-LAB/CLIENT
```
`~/A7-LAB/downloads/` papkasiga quyidagi fayllar yuklangan bo‘lishi kerak:
- `UbuntuServer_24.04_VM.7z` (yoki rasmiy Ubuntu Server 24.04 LTS tayyor obrazi/ISO)
- Agar VMware Workstation o‘rnatilmagan bo‘lsa, Linux uchun VMware Workstation Pro `.bundle` o‘rnatuvchisi.

#### 0.2. VMware Workstation va 3 ta mustaqil VM nusxasini ajratish
1. Agar 7-Zip o‘rnatilmagan bo‘lsa, Ubuntu xostida o‘rnating:
   ```bash
   # Ubuntu xost terminalida:
   sudo apt update && sudo apt install -y p7zip-full
   ```
2. Obraz arxivini xostda oching:
   ```bash
   # Ubuntu xost terminalida:
   7z x ~/A7-LAB/downloads/UbuntuServer_24.04_VM.7z -o~/A7-LAB/base_vm
   ```
3. Chiqqan tayyor VM fayllarini 3 ta alohida katalogga nusxalang:
   ```bash
   # Ubuntu xost terminalida:
   cp -r ~/A7-LAB/base_vm/* ~/A7-LAB/DNS-SRV/
   cp -r ~/A7-LAB/base_vm/* ~/A7-LAB/MAIL-SRV/
   cp -r ~/A7-LAB/base_vm/* ~/A7-LAB/CLIENT/
   ```
4. Ubuntu ilovalari orasidan VMware Workstation dasturini ishga tushiring (yoki terminalda `vmware &` deb yozing).
5. `File -> Open` orqali har uchala papkadagi `.vmx` faylini alohida oching:
   - `~/A7-LAB/DNS-SRV/*.vmx`
   - `~/A7-LAB/MAIL-SRV/*.vmx`
   - `~/A7-LAB/CLIENT/*.vmx`
6. `VM -> Settings -> Options -> General` menyusidan har bir VM nomini mos ravishda belgilang:
   - `DNS-SRV` (2 vCPU, 2 GB RAM)
   - `MAIL-SRV` (2 vCPU, 4 GB RAM)
   - `CLIENT` (2 vCPU, 2 GB RAM)
7. VM larni birinchi bor yoqayotganda VMware *"I copied it"* yoki *"I moved it"* deb so‘rasa, albatta **"I copied it"** bandini tanlang (bu har bir VM nusxasiga yangi va alohida unikal MAC manzil beradi).

> 📌 **0.1 va 0.2-bosqichdagi CLI buyruqlarining tahlili:**
> - `mkdir -p ~/A7-LAB/...`: `-p` (*parents*) bayrog‘i butun kataloglar zanjirini xatoliksiz yaratadi; agar katalog oldin mavjud bo‘lsa ham to‘xtab qolmaydi.
> - `sudo apt update`: Mahalliy paketlar ro‘yxatini eng so‘nggi repozitoriy indekslari bilan yangilaydi.
> - `sudo apt install -y p7zip-full`: 7-Zip (`.7z`) arxivlarini dekompressiya qilish vositasini o‘rnatadi. `-y` (*yes*) tasdiqlash so‘roviga avtomatik rozilik beradi.
> - `7z x ... -o...`: `x` (*extract with full paths*) arxivni ichki fayl tuzilishi va huquqlarini saqlagan holda ochadi; `-o` (*output*) arxiv ochiladigan maqsad papkani bo‘sh joylarsiz ko‘rsatadi.
> - `cp -r ~/A7-LAB/base_vm/* ...`: `-r` (*recursive*) barcha ichki virtual qattiq disk (`.vmdk`) va konfiguratsiya (`.vmx`) fayllarini to‘liq nusxalaydi.
> - `vmware &`: VMware dasturini fonda (`&`) ishga tushiradi, bu esa joriy terminal oynasini bo‘sh qoldirib, unda boshqa buyruqlarni bajarish imkonini beradi.

#### 0.3. Vaqtinchalik NAT va Lokal Wi-Fi Orqali Paketlarni O‘rnatish (Proksisiz Rejim)
VM larni yoqishdan oldin har birining 1-tarmoq adapterini vaqtinchalik **NAT** rejimiga qo‘ying (VMware NAT xostingizdagi lokal Wi-Fi internetini to‘g‘ridan-to‘g‘ri VM ga uzatadi).

Har bir VM konsoliga kirib, proksidan tozalash va paketlarni o‘rnatishni bajaring:

```bash
# 1. Proksisiz toza muhitni ta'minlash (agar avval proksi kiritilgan bo'lsa, tozalaymiz)
sudo rm -f /etc/apt/apt.conf.d/99proxy
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ftp_proxy FTP_PROXY

# 2. Internet borligini tekshirish (lokal Wi-Fi orqali to'g'ridan-to'g'ri chiqadi)
ping -c 2 8.8.8.8
curl -I --max-time 15 https://ubuntu.com
```

Endi har bir VM da o‘z roliga tegishli barcha paketlarni to‘g‘ridan-to‘g‘ri (proksisiz) o‘rnating:

```bash
# [DNS-SRV] konsolida bajaring:
sudo apt update
sudo apt install -y bind9 bind9-utils dnsutils tcpdump open-vm-tools

# [MAIL-SRV] konsolida bajaring:
sudo apt update
sudo DEBIAN_FRONTEND=noninteractive apt install -y \
  postfix opendkim opendkim-tools openssl mailutils tcpdump open-vm-tools

# [CLIENT] konsolida bajaring:
sudo apt update
sudo DEBIAN_FRONTEND=noninteractive apt install -y \
  dnsutils swaks openssl spfquery tcpdump tshark open-vm-tools
```

> [!TIP]
> Paketlar o‘rnatilgach, tizim vaqtini Ubuntu xost bilan solishtiring: `date -Is`. To‘g‘ri vaqt DNSSEC va TLS sertifikatlari uchun o‘ta muhimdir.

> 📌 **0.3-bosqichdagi CLI buyruqlarining tahlili:**
> - `sudo rm -f /etc/apt/apt.conf.d/99proxy`: `-f` (*force*) kaliti bilan fayl mavjud bo‘lsa uni so‘roqsiz o‘chiradi, yo‘q bo‘lsa xatolik bermaydi. Bu universitet proksisi qoldiqlarini tozalash uchun kerak.
> - `unset http_proxy https_proxy ...`: Joriy terminal sessiyasidagi proksi muhit o‘zgaruvchilarini bekor qiladi (o‘chiradi).
> - `ping -c 2 8.8.8.8`: `-c 2` (*count*) Google DNS serveriga faqat 2 ta ICMP Echo so‘rov yuborib, tashqi IP aloqasi borligini tez tekshiradi.
> - `curl -I --max-time 15 https://ubuntu.com`: `-I` (*head only*) to‘liq veb-sahifani yuklamasdan faqat HTTP response sarlavhasini oladi; `--max-time 15` ulanish 15 soniyada javob bermasa jarayonni to‘xtatadi. Bu HTTPS orqali tashqi internet ishlayotganini isbotlaydi.
> - `sudo DEBIAN_FRONTEND=noninteractive apt install -y ...`: `DEBIAN_FRONTEND=noninteractive` ko‘rsatmasi paketlarni o‘rnatishda (ayniqsa Postfix da) interaktiv ko‘k oynali dialoglar chiqib jarayon to‘xtab qolishining oldini oladi va standart sozlamalar bilan o‘rnatadi.
> - `date -Is`: Tizim vaqtini ISO-8601 xalqaro standartida chiqaradi (masalan, `2026-10-26T12:00:00+05:00`). Bu vaqt TLS sertifikatlari va DNSSEC imzo vaqtlari uchun xost bilan sinxron bo‘lishi shart.

#### 0.4. VMware Shared Folders va A7-base Snapshot
1. Ubuntu xostida umumiy hisobot papkasi tayyorlangan: `~/A7-LAB/report`.
2. VMware sozlamalarida (`VM -> Settings -> Options -> Shared Folders -> Always enabled`) ushbu papkani tanlang va `A7` nomi bilan ulang.
3. Har bir virtual mashina (Ubuntu Server) ichida tekshiring:
   ```bash
   sudo mkdir -p /mnt/a7share
   sudo vmhgfs-fuse .host:/A7 /mnt/a7share -o allow_other
   ls -la /mnt/a7share
   ```
4. Barcha paketlar o‘rnatilgach, VM larni to‘xtating (`sudo poweroff`).
5. VMware sozlamalarida NAT adapterini butunlay uzing (**Connected** va **Connect at power on** belgilarini olib tashlang).
6. Har bir VM ga bittadan tarmoq adapteri qo‘shing va uni yangi yaratilgan **A7-LAB** nomli **LAN Segment**ga ulang.
7. Har uchala VM uchun VMware menyusidan: `VM -> Snapshot -> Take Snapshot` orqali **`A7-base`** snapshotini oling!

> 📌 **0.4-bosqichdagi CLI buyruqlarining tahlili:**
> - `sudo mkdir -p /mnt/a7share`: VM fayl tizimida umumiy almashinuv papkasi ulanadigan montaj nuqtasini (`mount point`) yaratadi.
> - `sudo vmhgfs-fuse .host:/A7 /mnt/a7share -o allow_other`: `vmhgfs-fuse` VMware drayveri yordamida xostdagi `A7` deb nomlangan umumiy papkani `/mnt/a7share` ga ulaydi (`mount`). `-o allow_other` parametri nafaqat root, balki oddiy foydalanuvchilar ham ushbu katalogga fayl yozishi va o‘qishiga ruxsat beradi.
> - `ls -la /mnt/a7share`: Papka ichidagi barcha fayllarni (yashirin fayllar va ruxsatlari bilan) ro‘yxat shaklida ko‘rsatadi.
> - `sudo poweroff`: Virtual mashinani xavfsiz va to‘liq o‘chiradi.

---

### 3.1. 1-Bosqich: Tarmoq Sozlamalari, Statik IP va Hostname Konfiguratsiyasi
Uchala VMni yoqing. `ip -br link` buyrug‘i orqali A7-LAB ga ulangan interfeys nomini aniqlang (namunada `enp0s8`, ba’zan `ens33` yoki `ens37` bo‘lishi mumkin).

#### [DNS-SRV] da bajaring:
```bash
sudo hostnamectl set-hostname dns.a7.test
sudo ip addr flush dev enp0s8
sudo ip addr add 10.70.10.10/24 dev enp0s8
sudo ip link set enp0s8 up
```

#### [MAIL-SRV] da bajaring:
```bash
sudo hostnamectl set-hostname mail.a7.test
sudo ip addr flush dev enp0s8
sudo ip addr add 10.70.10.20/24 dev enp0s8
sudo ip link set enp0s8 up
sudo resolvectl dns enp0s8 10.70.10.10
sudo resolvectl domain enp0s8 '~a7.test'
```

#### [CLIENT] da bajaring:
```bash
sudo hostnamectl set-hostname client.a7.test
sudo ip addr flush dev enp0s8
sudo ip addr add 10.70.10.30/24 dev enp0s8
sudo ip link set enp0s8 up
sudo resolvectl dns enp0s8 10.70.10.10
sudo resolvectl domain enp0s8 '~a7.test'
```

#### Bog‘lanishni tekshirish:
CLIENT konsolida ping yuborib ko‘ring:
```bash
ping -c 2 10.70.10.10
ping -c 2 10.70.10.20
```
> **Muhim:** Har uchala VM `10.70.10.0/24` segmentida bir-birini ko‘rishi shart. Internetga chiqish yo‘li (default gateway) bo‘lmasligi kerak!

> 📌 **1-bosqichdagi CLI buyruqlarining tahlili:**
> - `sudo hostnamectl set-hostname <nom>`: Tizimning to‘liq tarmoq nomini (`FQDN`) o‘rnatadi (`dns.a7.test`, `mail.a7.test`, `client.a7.test`). Bu xizmatlar (ayniqsa Postfix va BIND) o‘zlarini to‘g‘ri identifikatsiya qilishi uchun kerak.
> - `sudo ip addr flush dev enp0s8`: Belgilangan tarmoq interfeysidagi (`enp0s8`) barcha avvalgi IP manzillar (masalan, DHCP dan olingan qoldiq manzillar)ni butunlay tozalaydi, ziddiyatlarning oldini oladi.
> - `sudo ip addr add 10.70.10.x/24 dev enp0s8`: Interfeysga statik IP manzil va `/24` (tarmoq maskasi: `255.255.255.0`) prefiksini biriktiradi.
> - `sudo ip link set enp0s8 up`: Tarmoq interfeysini faollashtiradi (yoqadi).
> - `sudo resolvectl dns enp0s8 10.70.10.10`: Ubuntu Server tizim resolveri (`systemd-resolved`) ga barcha DNS so‘rovlarini bizning laboratoriya DNS serverimiz `10.70.10.10` ga yo‘naltirishni buyuradi.
> - `sudo resolvectl domain enp0s8 '~a7.test'`: Qidiruv domeni qoidasini kiritadi. `~` (*tilda*) belgisi marshrutlash domenini anglatadi — faqat `a7.test` bilan tugagan so‘rovlar aynan shu interfeys va ko‘rsatilgan DNS serverga yuboriladi.
> - `ping -c 2 10.70.10.x`: `-c 2` (*count*) 2 ta ICMP Echo so‘rov yuborib, VMlar o‘rtasida L3 darajasida bog‘lanish mavjudligini tez tasdiqlaydi.

---

### 3.2. 2-Bosqich: MAIL-SRVda DKIM Kalit Juftini Yaratish
DKIM uchun 2048-bitli RSA kalit juftligi yaratiladi.

#### [MAIL-SRV] da bajaring:
```bash
# 1. Kalitlar katalogini yaratish
sudo install -d -o opendkim -g opendkim -m 0750 /etc/opendkim/keys/a7.test

# 2. Kalit juftini yaratish (domen: a7.test, selector: a7)
sudo opendkim-genkey -b 2048 -D /etc/opendkim/keys/a7.test -d a7.test -s a7

# 3. Ruxsatlarni qat'iy cheklash (Private kalit 0600 bo'lishi shart!)
sudo chown opendkim:opendkim /etc/opendkim/keys/a7.test/a7.private
sudo chmod 0600 /etc/opendkim/keys/a7.test/a7.private
sudo chmod 0644 /etc/opendkim/keys/a7.test/a7.txt

# 4. Ruxsatlarni tekshirish
sudo stat -c '%a %U:%G %n' /etc/opendkim/keys/a7.test/a7.private /etc/opendkim/keys/a7.test/a7.txt
# Natija: 600 opendkim:opendkim ... a7.private bo'lishi kerak

# 5. Hisobot uchun xavfsiz Public Key SHA-256 fingerprintini olish:
sudo openssl pkey -in /etc/opendkim/keys/a7.test/a7.private -pubout -outform DER 2>/dev/null | sha256sum

# 6. DNS yozuviga qo'yiladigan Ochiq Kalit (Public Key)ni ekranga chiqarish:
sudo cat /etc/opendkim/keys/a7.test/a7.txt
```
> [!CAUTION]
> `a7.private` fayli mazmuni maxfiy hisoblanadi! Uni hech qachon DNS serverga nusxalamang va hisobot skrinshotlariga chiqarmang. DNS-SRV ga faqat `a7.txt` dagi `p=MIIBIjANBg...` ochiq kaliti o‘tadi!

> 📌 **2-bosqichdagi CLI buyruqlarining tahlili:**
> - `sudo install -d -o opendkim -g opendkim -m 0750 <yo‘l>`: Yangi katalog yaratadi (`-d`), egasini (`-o opendkim`) va guruhini (`-g opendkim`) biriktiradi hamda ruxsatini `0750` qilib belgilaydi (faqat opendkim xizmati o‘qiy/yoza oladi).
> - `sudo opendkim-genkey -b 2048 -D ... -d a7.test -s a7`: OpenDKIM kalitlar generatori:
>   * `-b 2048`: Kriptografik mustahkam 2048-bitli RSA kalit uzunligi.
>   * `-D <papka>`: Yaratilgan kalitlar qaysi katalogga saqlanishi kerakligi.
>   * `-d a7.test`: Kalit tegishli bo‘lgan domen nomi (`domain`).
>   * `-s a7`: DKIM selektori (`selector`). Natijada ikkita fayl yaratiladi: `a7.private` (maxfiy imzolash kaliti) va `a7.txt` (DNSga qo‘yiladigan ochiq kalit TXT yozuvi).
> - `sudo chmod 0600 a7.private`: Xavfsizlikning eng qat’iy talabi! `0600` ruxsati maxfiy kalitni faqat uning egasi (`opendkim`) o‘qiy va yoza olishini, boshqa barcha foydalanuvchilar undan mahrum ekanligini kafolatlaydi.
> - `sudo stat -c '%a %U:%G %n' ...`: Fayllarning ruxsat oktali (`%a`), egasi (`%U`), guruhi (`%G`) va nomini (`%n`) formatlab ko‘rsatadi.
> - `sudo openssl pkey -in ... -pubout -outform DER 2>/dev/null | sha256sum`: Maxfiy kalit faylidan ochiq kalitni ajratib oladi (`-pubout`), uni binar shaklga o‘tkazadi (`-outform DER`) va SHA-256 xeshini oladi. Bu orqali maxfiy kalit oshkor etilmagan holda uning unikal xesh-barmoq izi (fingerprint) olinadi.
> - `sudo cat .../a7.txt`: BIND9 zona fayliga joylashtiriladigan public TXT yozuvining tayyor sintaksisini ekranga chiqaradi.

---

### 3.3. 3-Bosqich: DNS-SRVda BIND9 va a7.test Zonasini Sozlash (DNSSEC bilan)

#### [DNS-SRV] da bajaring:

1. DNSSEC kalitlari uchun BIND katalogini yaratish:
```bash
sudo install -d -o bind -g bind -m 0750 /var/cache/bind/keys
```

2. BIND xavfsizlik parametrlarini sozlash (`/etc/bind/named.conf.options`):
```bash
sudo tee /etc/bind/named.conf.options >/dev/null <<'EOF'
acl a7_lab { 127.0.0.1; 10.70.10.0/24; };

options {
    directory "/var/cache/bind";
    key-directory "/var/cache/bind/keys";
    listen-on { 127.0.0.1; 10.70.10.10; };
    listen-on-v6 { none; };
    recursion no;
    allow-query { a7_lab; };
    allow-transfer { none; };
    minimal-responses yes;
};
EOF
```

3. Zonani e’lon qilish (`/etc/bind/named.conf.local`):
```bash
sudo tee /etc/bind/named.conf.local >/dev/null <<'EOF'
zone "a7.test" {
    type primary;
    file "/var/cache/bind/db.a7.test";
    dnssec-policy default;
    inline-signing yes;
};
EOF
```

4. Zona ma’lumotlar bazasini yaratish (`/var/cache/bind/db.a7.test`):
> **Diqqat:** Quyidagi fayldagi `p=PUBLIC_KEY_A7_TXT_DAN_OLINADI` o‘rniga MAIL-SRVdagi `/etc/opendkim/keys/a7.test/a7.txt` faylida chiqqan `p=` qiymatini qo‘ying (agar kalit uzun bo‘lsa, 255 belgidan oshmasligi uchun qo‘shtirnoqlar bilan bo‘lib yozing):

```bash
sudo tee /var/cache/bind/db.a7.test >/dev/null <<'EOF'
$TTL 300
@ IN SOA dns.a7.test. admin.a7.test. (
  2026090501  300  120  604800  300 )
@       IN NS   dns.a7.test.
dns     IN A    10.70.10.10
mail    IN A    10.70.10.20
@       IN MX 10 mail.a7.test.
@       IN TXT  "v=spf1 ip4:10.70.10.20 -all"
_dmarc  IN TXT  "v=DMARC1; p=reject; adkim=s; aspf=s; pct=100"
a7._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa;" "p=PUBLIC_KEY_A7_TXT_DAN_OLINADI" )
EOF
```

5. Ruxsatlarni o‘rnatish, sintaksisni tekshirish va BIND9 ni qayta ishga tushirish:
```bash
sudo chown bind:bind /var/cache/bind/db.a7.test
sudo chmod 0640 /var/cache/bind/db.a7.test

# Sintaksis tekshiruvi (hech qanday xato chiqmasligi shart)
sudo named-checkconf
sudo named-checkzone a7.test /var/cache/bind/db.a7.test

# Xizmatni qayta ishga tushirish
sudo systemctl restart bind9
sleep 3

# DNSSEC imzolanish holatini tekshirish
sudo rndc signing -list a7.test
sudo rndc zonestatus a7.test
```

> 📌 **3-bosqichdagi CLI buyruqlarining tahlili:**
> - `sudo install -d -o bind -g bind -m 0750 /var/cache/bind/keys`: BIND9 ning avtomatik DNSSEC kalitlari saqlanadigan katalogini yaratib, unga `bind:bind` foydalanuvchisini biriktiradi.
> - `sudo tee <fayl> >/dev/null <<'EOF' ... EOF`: Administrator (sudo) huquqi bilan konfiguratsiya fayllariga bir nechta qatorli matnni xatosiz yozish usuli (`tee` standart oqimdan oladi, `>/dev/null` esa terminalga ortiqcha matn chiqishining oldini oladi).
> - `sudo chown bind:bind /var/cache/bind/db.a7.test`: Zona faylining egasini BIND9 xizmati hisobiga o‘tkazadi, chunki `inline-signing` rejimida BIND bu fayl yonida imzolangan `.signed` kesh fayllarini yaratishi uchun katalog va faylga to‘liq yozish huquqi kerak.
> - `sudo chmod 0640 /var/cache/bind/db.a7.test`: Fayl ruxsatini xavfsiz qiladi (egasi o‘qiydi/yozadi, bind guruhi o‘qiydi, boshqalar ko‘ra olmaydi).
> - `sudo named-checkconf`: BIND9 ning barcha konfiguratsiya fayllari (`named.conf`, `named.conf.options`, `named.conf.local`) sintaksisini to‘liq tekshiradi. Agar sintaksis to‘g‘ri bo‘lsa, hech qanday xato chiqarmaydi (exit code 0).
> - `sudo named-checkzone a7.test /var/cache/bind/db.a7.test`: Muayyan zonaning sintaksisini, SOA serialini, NS va A yozuvlari mantiqiy to‘g‘riligini tekshiradi (muvaffaqiyatli bo‘lsa `OK` qaytaradi).
> - `sudo systemctl restart bind9`: BIND9 DNS demonini yangi konfiguratsiya bilan qayta ishga tushiradi.
> - `sleep 3`: BIND9 servisi to‘liq yuklanishi va orqa fonda DNSSEC kalitlarini generatsiya qilib zonani imzolab olishi uchun 3 soniya kutish beradi.
> - `sudo rndc signing -list a7.test`: `rndc` (Remote Name Daemon Control) orqali `a7.test` zonasining DNSSEC avtomatik imzolash holati va unga biriktirilgan KSK/ZSK kalitlar holatini ko‘rsatadi.
> - `sudo rndc zonestatus a7.test`: Zonaning to‘liq dinamik holatini (serial raqami, DNSSEC faolligi, oxirgi yuklanish vaqti) tekshiradi.

---

### 3.4. 4-Bosqich: CLIENT orqali DNS, DNSSEC va Salbiy ACL Testlari

#### [CLIENT] da bajaring:

1. Ijobiy Authoritative va DNSSEC testlari:
```bash
# SOA va AA bayrog'ini tekshirish
dig @10.70.10.10 a7.test SOA +dnssec

# DNSKEY yozuvlarini tekshirish
dig @10.70.10.10 a7.test DNSKEY +dnssec +multiline

# A va MX yozuvlarini tekshirish
dig @10.70.10.10 mail.a7.test A +dnssec
dig @10.70.10.10 a7.test MX +short

# SPF, DKIM va DMARC yozuvlarini tekshirish
dig @10.70.10.10 a7.test TXT +short
dig @10.70.10.10 a7._domainkey.a7.test TXT +short
dig @10.70.10.10 _dmarc.a7.test TXT +short
```
*Kutilgan natija:* Javoblarda `flags: qr aa rd` (ayniqsa `aa` - Authoritative Answer) ko‘rinadi, `status: NOERROR` bo‘ladi, RRSIG va DNSKEY yozuvlari chiqadi.

2. **T1 Salbiy Testi — Rekursiyaning rad etilishi (Open Resolver emasligini isbotlash):**
```bash
dig @10.70.10.10 example.com A +time=2 +tries=1
```
*Kutilgan natija:* `status: REFUSED`, `ra` (Recursion Available) bayrog‘i yo‘q.

3. **T2 Salbiy Testi — Ruxsatsiz Zone Transfer (AXFR) bloklanganligi:**
```bash
dig @10.70.10.10 a7.test AXFR +time=2 +tries=1
```
*Kutilgan natija:* `Transfer failed.` yoki `status: REFUSED`.

> 📌 **4-bosqichdagi CLI buyruqlarining tahlili:**
> - `dig @10.70.10.10 a7.test SOA +dnssec`:
>   * `dig` (Domain Information Groper) — DNS so‘rovlarini yuboruvchi va tahlil qiluvchi asosiy tarmoq utilitasi.
>   * `@10.70.10.10`: So‘rov tizimning standart resolveriga emas, bevosita biz sozlagan DNS-SRV serveriga yuborilishini belgilaydi.
>   * `SOA`: Zonaning asosiy boshqaruv yozuvini (Start of Authority) so‘raydi.
>   * `+dnssec`: Javob bilan birga tegishli DNSSEC imzo yozuvini (`RRSIG SOA`) ham talab qiladi.
> - `dig @10.70.10.10 a7.test DNSKEY +dnssec +multiline`:
>   * `DNSKEY`: Zonaning ochiq kalitlarini (ZSK - Zone Signing Key va KSK - Key Signing Key) so‘raydi.
>   * `+multiline`: Uzun RSA ochiq kalit parametrlarini ko‘p qatorli, inson o‘qishiga qulay formatda chiqaradi.
> - `dig ... +short`: Barcha diagnostik sarlavhalar va bo‘limlarni tashlab, faqat so‘ralgan yozuv qiymatini (IP, matn) qisqa formatda chiqaradi.
> - `dig @10.70.10.10 example.com A +time=2 +tries=1` (T1 Testi):
>   * Authoritative serverdan begona tashqi domenni so‘raydi.
>   * `+time=2`: Javob kutish vaqtini 2 soniya bilan cheklaydi.
>   * `+tries=1`: Faqat bitta urinish bilan so‘rov yuboradi.
>   * *Vazifasi:* Server begona domenlarni rekursiv qidirmasligini va so‘rovni `REFUSED` (Rad etildi) kodi bilan darhol to‘xtatishini (Open Resolver emasligini) isbotlash.
> - `dig @10.70.10.10 a7.test AXFR +time=2 +tries=1` (T2 Testi):
>   * `AXFR` (*Authoritative Zone Transfer*): Butun zona yozuvlarini bir yo‘la ko‘chirish so‘rovi (TCP 53 orqali).
>   * *Vazifasi:* Server ruxsatsiz zona uzatishni bloklaganini (`Transfer failed / REFUSED`) isbotlash.

---

### 3.5. 5-Bosqich: MAIL-SRVda Postfix va Majburiy STARTTLS Konfiguratsiyasi

#### [MAIL-SRV] da bajaring:

1. Mahalliy pochta qabul qiluvchi `analyst` foydalanuvchisini yaratish:
```bash
sudo adduser --disabled-password --gecos '' analyst
```

2. Laboratoriya uchun 30 kunlik self-signed TLS sertifikatini yaratish:
```bash
sudo openssl req -x509 -newkey rsa:2048 -nodes -days 30 \
  -keyout /etc/postfix/a7-mail.key \
  -out /etc/postfix/a7-mail.crt \
  -subj '/CN=mail.a7.test' \
  -addext 'subjectAltName=DNS:mail.a7.test'

# Sertifikat va kalit ruxsatlari
sudo chown root:root /etc/postfix/a7-mail.key /etc/postfix/a7-mail.crt
sudo chmod 0600 /etc/postfix/a7-mail.key
sudo chmod 0644 /etc/postfix/a7-mail.crt
```

3. Postfix parametrlarini xavfsiz sozlash:
```bash
sudo postconf -e 'myhostname = mail.a7.test'
sudo postconf -e 'mydomain = a7.test'
sudo postconf -e 'myorigin = $mydomain'
sudo postconf -e 'inet_interfaces = all'
sudo postconf -e 'inet_protocols = ipv4'
sudo postconf -e 'mydestination = $myhostname, localhost, $mydomain'
sudo postconf -e 'mynetworks = 127.0.0.0/8'
sudo postconf -e 'smtpd_relay_restrictions = permit_mynetworks, reject_unauth_destination'
sudo postconf -e 'disable_vrfy_command = yes'
sudo postconf -e 'smtpd_helo_required = yes'

# STARTTLS majburiy (encrypt) rejimi
sudo postconf -e 'smtpd_tls_cert_file = /etc/postfix/a7-mail.crt'
sudo postconf -e 'smtpd_tls_key_file = /etc/postfix/a7-mail.key'
sudo postconf -e 'smtpd_tls_security_level = encrypt'
sudo postconf -e 'smtpd_tls_received_header = yes'
sudo postconf -e 'smtpd_tls_loglevel = 1'
```

> 📌 **5-bosqichdagi CLI buyruqlarining tahlili:**
> - `sudo adduser --disabled-password --gecos '' analyst`:
>   * Mahalliy pochta xabarlarini qabul qiluvchi `analyst` tizim foydalanuvchisini yaratadi.
>   * `--disabled-password`: Xavfsizlik maqsadida foydalanuvchiga interaktiv login paroli berilmaydi (faqat pochta qutisi ochiladi).
>   * `--gecos ''`: Foydalanuvchi haqidagi qo‘shimcha ma’lumotlarni (xona, telefon raqami) so‘ramasdan bo‘sh qoldiradi.
> - `sudo openssl req -x509 -newkey rsa:2048 -nodes -days 30 -keyout ... -out ...`:
>   * `openssl req`: Sertifikat yaratish va boshqarish buyrug‘i.
>   * `-x509`: Sertifikat markazi (CA) imzosisiz, o‘z-o‘zidan imzolangan (*self-signed*) test sertifikatini yaratadi.
>   * `-newkey rsa:2048`: Yangi 2048-bitli RSA maxfiy kalitini generatsiya qiladi.
>   * `-nodes` (*no DES*): Maxfiy kalitni parolsiz saqlaydi, bu esa Postfix servisi avtomatik qayta yoqilganda parol so‘rab to‘xtab qolishining oldini oladi.
>   * `-days 30`: Sertifikatning amal qilish muddatini 30 kun qilib belgilaydi.
>   * `-subj '/CN=mail.a7.test'`: Sertifikatning Common Name (asosiy server nomi) parametrini kiritadi.
>   * `-addext 'subjectAltName=DNS:mail.a7.test'`: Zamonaviy TLS talabi bo‘lgan SAN (*Subject Alternative Name*) kengaytmasini qo‘shadi.
> - `sudo postconf -e '<parametr> = <qiymat>'`:
>   * `postconf`: Postfix ning barcha konfiguratsiya parametrlarini boshqarish utilitasi.
>   * `-e` (*edit*): `/etc/postfix/main.cf` fayliga ko‘rsatilgan parametrni xatosiz yozadi yoki yangilaydi.
>   * `smtpd_tls_security_level = encrypt`: 25-portda STARTTLS shifrlashini majburiy qiladi; STARTTLSsiz kelgan har qanday xat uzatishni rad etadi.
>   * `smtpd_relay_restrictions = permit_mynetworks, reject_unauth_destination`: Faqat ishonchli tarmoqlarga va o‘z domenimizga kelgan xatlarni qabul qiladi, boshqa tashqi manzillarga ruxsatsiz relay qilishni (Open Relay) taqiqlaydi.
>   * `disable_vrfy_command = yes`: Spamerlar foydalanuvchilar mavjudligini tekshirish uchun ishlatadigan `VRFY` komandasini o‘chiradi.

---

### 3.6. 6-Bosqich: OpenDKIMni Postfix Milteriga Bog‘lash

#### [MAIL-SRV] da bajaring:

1. OpenDKIM jadvallarini yaratish:
```bash
# Key table
sudo tee /etc/opendkim/key.table >/dev/null <<'EOF'
a7._domainkey.a7.test a7.test:a7:/etc/opendkim/keys/a7.test/a7.private
EOF

# Signing table
sudo tee /etc/opendkim/signing.table >/dev/null <<'EOF'
*@a7.test a7._domainkey.a7.test
EOF

# Ishonchli xostlar (CLIENT 10.70.10.30 imzolanishi uchun qo'shiladi)
sudo tee /etc/opendkim/trusted.hosts >/dev/null <<'EOF'
127.0.0.1
10.70.10.30
EOF
```

2. Asosiy konfiguratsiya fayli (`/etc/opendkim.conf`):
```bash
sudo tee /etc/opendkim.conf >/dev/null <<'EOF'
Syslog                  yes
SyslogSuccess           yes
Canonicalization        relaxed/simple
Mode                    sv
SubDomains              no
OversignHeaders         From
Socket                  inet:8891@127.0.0.1
PidFile                 /run/opendkim/opendkim.pid
UserID                  opendkim
UMask                   007
KeyTable                refile:/etc/opendkim/key.table
SigningTable            refile:/etc/opendkim/signing.table
InternalHosts           refile:/etc/opendkim/trusted.hosts
ExternalIgnoreList      refile:/etc/opendkim/trusted.hosts
Nameservers             10.70.10.10
EOF
```

3. Postfix milter sozlamalari va qayta ishga tushirish:
```bash
sudo postconf -e 'milter_protocol = 6'
sudo postconf -e 'milter_default_action = tempfail'
sudo postconf -e 'smtpd_milters = inet:127.0.0.1:8891'
sudo postconf -e 'non_smtpd_milters = $smtpd_milters'

# Konfiguratsiyalarni tekshirish
sudo opendkim -n -x /etc/opendkim.conf
sudo postfix check

# Xizmatlarni qayta ishga tushirish
sudo systemctl restart opendkim postfix
sudo systemctl --no-pager --full status opendkim postfix

# Tinglanayotgan portlarni tekshirish (25 va 8891)
sudo ss -ltnp | grep -E ':25|:8891'

# DNSdagi ochiq kalit private kalitga mosligini tekshirish:
sudo opendkim-testkey -d a7.test -s a7 -vvv
# Natijada "key OK" chiqishi shart!

# Sertifikat fingerprinti (buni CLIENT bilan taqqoslaysiz):
sudo openssl x509 -in /etc/postfix/a7-mail.crt \
  -noout -subject -ext subjectAltName -fingerprint -sha256
```

> 📌 **6-bosqichdagi CLI buyruqlarining tahlili:**
> - `sudo postconf -e 'milter_protocol = 6'`: Postfix bilan tashqi milter o‘rtasidagi protokoli versiyasini 6 (Sendmail Milter v6) qilib o‘rnatadi.
> - `sudo postconf -e 'milter_default_action = tempfail'`: *Fail-Secure* xavfsizlik qoidasi! Agar OpenDKIM filtri ishlamay qolsa, Postfix kelgan xatni imzosiz yuborib yubormasdan, `451` xatosi bilan vaqtincha to‘xtatadi.
> - `sudo postconf -e 'smtpd_milters = inet:127.0.0.1:8891'`: Postfix ga kiruvchi/chiquvchi SMTP oqimini tekshirish uchun `127.0.0.1:8891` dagi OpenDKIM xizmatiga ulanishni buyuradi.
> - `sudo opendkim -n -x /etc/opendkim.conf`:
>   * `-n`: Konfiguratsiya faylini sinovdan o‘tkazish (dry run).
>   * `-x`: Barcha ogohlantirishlar va sintaktik xatolarni terminalga chiqaradi.
> - `sudo postfix check`: Postfix ning barcha fayl ruxsatlari va kataloglar to‘g‘riligini tekshiradi.
> - `sudo systemctl restart opendkim postfix`: Yangi milter sozlamalarini qo‘llash uchun ikkala xizmatni qayta ishga tushiradi.
> - `sudo ss -ltnp | grep -E ':25|:8891'`:
>   * `ss` (*Socket Statistics*) — tizimdagi tarmoq soketlarini ko‘rsatadi.
>   * `-l`: Faqat tinglanayotgan (`LISTEN`) portlar.
>   * `-t`: Faqat TCP trafigi.
>   * `-n`: Portlarni so‘z bilan emas, sonli qiymatda ko‘rsatish (`smtp` emas `25`).
>   * `-p`: Soketni ushlab turgan jarayon nomi va PID raqamini ko‘rsatish.
> - `sudo opendkim-testkey -d a7.test -s a7 -vvv`:
>   * `-d a7.test`: Domen nomi.
>   * `-s a7`: DKIM selektori.
>   * `-vvv`: Maksimal darajada batafsil chiqish (*verbose*).
>   * *Vazifasi:* DNS serverdagi `a7._domainkey.a7.test` ochiq kalitini tortib olib, uning diskdagi `a7.private` maxfiy kalitiga matematik mosligini tasdiqlash (`key OK`).
> - `sudo openssl x509 -in ... -noout -subject -ext subjectAltName -fingerprint -sha256`: Sertifikat matnini chiqarib o‘tirmasdan (`-noout`), uning mavzusi (`-subject`), muqobil nomlari (`-ext subjectAltName`) va SHA-256 barmoq izini (`-fingerprint -sha256`) ekranga chiqaradi.

---

### 3.7. 7-Bosqich: CLIENT orqali SMTP, STARTTLS va Relay Testlari (T3, T4, T5)

#### [CLIENT] da bajaring:

1. **T3 Testi — STARTTLSsiz xat yuborish urinishi (Rad etilishi shart):**
```bash
swaks --server mail.a7.test --port 25 \
  --from student@a7.test --to analyst@a7.test --quit-after MAIL
```
*Kutilgan natija:* Postfix javobi: `530 5.7.0 Must issue a STARTTLS command first`. Plaintext DATA yuborilmadi.

2. **T4 Testi — Majburiy TLS bilan mahalliy foydalanuvchiga xat yetkazish:**
```bash
swaks --server mail.a7.test --port 25 --tls \
  --from student@a7.test --to analyst@a7.test \
  --header 'From: student@a7.test' \
  --header 'To: analyst@a7.test' \
  --header 'Subject: A7 secure mail' \
  --body 'A7: STARTTLS va DKIM nazorat xabari.'
```
*Kutilgan natija:* TLS muzokaralari muvaffaqiyatli o‘tadi, oxirida: `250 2.0.0 Ok: queued as ...` xabari chiqadi.

3. **T5 Testi — TLS mavjud bo‘lsa ham begona domenga ruxsatsiz relay (Open Relay bloklanishi):**
```bash
swaks --server mail.a7.test --port 25 --tls \
  --from forged@a7.test --to victim@external.test --quit-after RCPT
```
*Kutilgan natija:* RCPT TO bosqichida rad etiladi: `554 5.7.1 <victim@external.test>: Relay access denied`.

4. **Sertifikat Fingerprintini out-of-band tekshirish:**
```bash
openssl s_client -starttls smtp -connect mail.a7.test:25 \
  -servername mail.a7.test </dev/null 2>/dev/null | \
  openssl x509 -noout -subject -issuer -ext subjectAltName -fingerprint -sha256
```
*Kutilgan natija:* SHA-256 Fingerprint qiymati MAIL-SRV konsolida olingan qiymat bilan 100% bir xil chiqishi kerak.

5. **MAIL-SRV da yetkazilgan xat va DKIM imzosini tekshirish:**
[MAIL-SRV] konsolida bajaring:
```bash
sudo grep -A6 '^DKIM-Signature:' /var/mail/analyst | tail -n 12
sudo grep -E '^(From:|To:|Subject:)' /var/mail/analyst | tail -n 6
sudo journalctl -u postfix -u opendkim --since '-10 min' --no-pager | tail -n 40
```
*Kutilgan natija:* Xat qutisida `DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=a7.test; s=a7...` imzosi mavjudligi tasdiqlanadi.

> 📌 **7-bosqichdagi CLI buyruqlarining tahlili:**
> - `swaks --server mail.a7.test --port 25 --from ... --to ... --quit-after MAIL` (T3 Testi):
>   * `swaks` (*Swiss Army Knife for SMTP*) — SMTP protokolini moslashuvchan va nazorat ostida sinash uchun mo‘ljallangan buyruq qatori mijozi.
>   * `--server mail.a7.test --port 25`: Ulanish manzili va porti.
>   * `--from student@a7.test`: Jo‘natuvchi pochtasi (konvertdagi `MAIL FROM`).
>   * `--to analyst@a7.test`: Qabul qiluvchi pochtasi (konvertdagi `RCPT TO`).
>   * `--quit-after MAIL`: Sessiyani `MAIL FROM` komandasidan keyinoq to‘xtatish. Shifrlanmagan ulanishda Postfix shu zahotiyoq `530 5.7.0 Must issue a STARTTLS command first` xatosini qaytarishi kerak.
> - `swaks ... --tls --header '...' --body '...'` (T4 Testi):
>   * `--tls`: Dastlabki `EHLO` salomlashuvidan so‘ng avtomatik `STARTTLS` buyrug‘ini beradi va TLS shifrlash tunnelini hosil qiladi.
>   * `--header`: Xatning MIME sarlavhalarini (`From:`, `To:`, `Subject:`) o‘rnatadi.
>   * `--body`: Xatning asosiy matnini yuboradi. Natijada xat `250 2.0.0 Ok: queued as ...` bilan muvaffaqiyatli qabul qilinadi.
> - `swaks ... --tls --from forged@a7.test --to victim@external.test --quit-after RCPT` (T5 Testi):
>   * Open Relay zaifligini sinash: garchi TLS ulangan bo‘lsa-da, ruxsat etilmagan tashqi domenga (`external.test`) xat yuborishga urinadi.
>   * `--quit-after RCPT`: Qabul qiluvchi tekshiruvi paytida to‘xtatadi. Postfix uni `554 5.7.1 Relay access denied` bilan rad etishi shart.
> - `openssl s_client -starttls smtp -connect mail.a7.test:25 ...`:
>   * OpenSSL ning SSL/TLS mijoz utilitasi.
>   * `-starttls smtp`: SMTP sessiyasini ochib, unga STARTTLS komandasini yuboradi va o‘rnatilgan TLS sertifikatini tortib oladi.
>   * `openssl x509 -fingerprint -sha256`: Tortib olingan sertifikatning SHA-256 barmoq izini hisoblaydi (buni MAIL-SRV dagi asl sertifikat fingerprinti bilan solishtirish uchun).
> - `sudo grep -A6 '^DKIM-Signature:' /var/mail/analyst`:
>   * Qabul qilingan xat qutisida xat boshidagi `DKIM-Signature` maydoni va uning keyingi 6 qatorini ekranga chiqaradi (imzo mavjudligini tasdiqlash uchun).
> - `sudo journalctl -u postfix -u opendkim --since '-10 min' --no-pager | tail -n 40`:
>   * Postfix va OpenDKIM xizmatlarining oxirgi 10 daqiqadagi tizim loglarini sahifalamasdan (`--no-pager`) chiqaradi va oxirgi 40 qatorini ko‘rsatadi.

---

### 3.8. 8-Bosqich: SPF, DKIM va DMARC Testlari (T6, T7 va Alignment)

#### [CLIENT] da bajaring:

1. **T6 Testi — Ruxsat etilgan jo‘natuvchi MTA IPsi (SPF pass):**
```bash
spfquery -i 10.70.10.20 -s student@a7.test -h mail.a7.test
```
*Kutilgan natija:* `result: pass` (chunki DNSda `ip4:10.70.10.20` ko‘rsatilgan).

2. **T7 Testi — Ruxsat etilmagan IP manzil (SPF fail):**
```bash
spfquery -i 10.70.10.30 -s student@a7.test -h client.a7.test
```
*Kutilgan natija:* `result: fail` (chunki DNS siyosatida `-all` mavjud).

3. **DMARC Identifier Alignment tahlili:**
```bash
dig @10.70.10.10 a7.test TXT +short
dig @10.70.10.10 _dmarc.a7.test TXT +short
```
*Xulosa:*
- `Header From:` `student@a7.test`
- `DKIM d=:` `a7.test` (Strict DKIM alignment — mos)
- `SPF Envelope Sender:` `student@a7.test` (Strict SPF alignment — mos)
- Ikkala autentifikatsiya ham qat’iy mos keladi (`adkim=s; aspf=s`). DMARC siyosati `p=reject` bo‘lgani sababli, har qanday noqonuniy soxtalashtirish to‘liq rad etiladi.

> 📌 **8-bosqichdagi CLI buyruqlarining tahlili:**
> - `spfquery -i 10.70.10.20 -s student@a7.test -h mail.a7.test`:
>   * `spfquery`: DNSdagi SPF siyosatini simulyatsiya qiluvchi standart buyruq qatori vositasi.
>   * `-i 10.70.10.20`: Qabul qiluvchi server ko‘rayotgan jo‘natuvchi MTA ning IP manzili.
>   * `-s student@a7.test`: SMTP konvertidagi jo‘natuvchi manzili (Envelope Sender / Return-Path).
>   * `-h mail.a7.test`: Yuboruvchi serverning HELO/EHLO xost nomi.
>   * *Vazifasi:* DNSdagi `ip4:10.70.10.20` qoidasiga ko‘ra natija `result: pass` bo‘lishini tekshirish.
> - `spfquery -i 10.70.10.30 ...`: Ruxsat berilmagan IP (`10.70.10.30`) orqali xat kelganda SPF siyosatidagi `-all` qoidasi tufayli `result: fail` qaytishini isbotlash.
> - `dig @10.70.10.10 _dmarc.a7.test TXT +short`: DMARC siyosati parametrlarini (`p=reject; adkim=s; aspf=s; pct=100`) tekshirish va alignment mosligini taqqoslash.

---

### 3.9. 9-Bosqich: PCAP Trafik Tahlili (tcpdump va tshark)

#### [CLIENT] da bajaring:

1. **Terminal 1 da paketlarni yozib olishni boshlang:**
```bash
sudo tcpdump -Z root -i enp0s8 -nn -s0 -w ~/A7_dns_smtp.pcap \
  '((host 10.70.10.10) and port 53) or ((host 10.70.10.20) and tcp port 25)'
```

2. **Terminal 2 da testlarni qayta yuboring:**
```bash
dig @10.70.10.10 a7.test SOA
swaks --server mail.a7.test --port 25 --from student@a7.test --to analyst@a7.test --quit-after MAIL
swaks --server mail.a7.test --port 25 --tls --from student@a7.test --to analyst@a7.test \
  --header 'From: student@a7.test' --header 'Subject: A7 secure mail' \
  --body 'A7: STARTTLS va DKIM nazorat xabari.'
```
Testlar tugagach, Terminal 1 da `Ctrl + C` bosing.

3. **Paketlarni tshark yordamida tahlil qilish:**
```bash
# DNS javoblari
tshark -r ~/A7_dns_smtp.pcap -Y 'dns.flags.response==1' \
  -T fields -e frame.number -e ip.src -e dns.qry.name -e dns.qry.type -e dns.flags.rcod

# STARTTLS buyrug'i va TLS Handshake
tshark -r ~/A7_dns_smtp.pcap \
  -Y 'smtp.req.command=="STARTTLS" || tls.handshake.type==1' \
  -T fields -e frame.number -e ip.src -e ip.dst -e smtp.req.command -e tls.handshake.type

# XABAR TANASI OCHIQ QOLMAGANINI ISBOTLASH (Bo'sh chiqishi kerak!):
tshark -r ~/A7_dns_smtp.pcap -Y smtp -V | grep -F 'A7: STARTTLS va DKIM nazorat xabari.' || true

# PCAP xeshini olish:
sha256sum ~/A7_dns_smtp.pcap
```
> [!NOTE]
> Oxirgi `grep` buyrug‘i bo‘sh natija qaytarsa, bu xat matni tarmoqda ochiq ketmaganini, balki TLS orqali to‘liq himoyalanganini isbotlaydi.

> 📌 **9-bosqichdagi CLI buyruqlarining tahlili:**
> - `sudo tcpdump -Z root -i enp0s8 -nn -s0 -w ~/A7_dns_smtp.pcap '...'`:
>   * `tcpdump` — tarmoq paketlarini ushlab oluvchi klassik tahlil vositasi.
>   * `-Z root`: Yozilayotgan PCAP faylining egasini root foydalanuvchisi qilib belgilaydi (faylga yozishdagi ruxsat xatolarining oldini oladi).
>   * `-i enp0s8`: Tinglanayotgan virtual tarmoq interfeysi.
>   * `-nn`: IP manzillar va port nomlarini matn ko‘rinishiga o‘girmasdan, sonli formatda yozadi (ortiqcha teskari DNS so‘rovlarining oldini oladi).
>   * `-s0` (*snapshot length 0*): Paketlarni cheklanmagan to‘liq hajmda (butun payload ma’lumotlari bilan) yozib oladi.
>   * `-w ~/A7_dns_smtp.pcap`: Uslangan paketlarni PCAP fayliga saqlaydi.
>   * Filtr: `'((host 10.70.10.10) and port 53) or ((host 10.70.10.20) and tcp port 25)'` — faqat laboratoriya uchun zarur bo‘lgan DNS (53) va SMTP (25) trafigini saralab oladi.
> - `tshark -r ~/A7_dns_smtp.pcap -Y '...' -T fields -e ...`:
>   * `tshark`: Wireshark dasturining konsol versiyasi.
>   * `-r <fayl>`: Saqlangan PCAP faylini o‘qish.
>   * `-Y '<display filter>'`: Paketlarni saralovchi Wireshark displey filtri (masalan, `dns.flags.response==1` faqat DNS javoblarini ajratadi; `smtp.req.command=="STARTTLS"` esa STARTTLS buyruqlarini ko‘rsatadi).
>   * `-T fields -e <maydon>`: Chiqishni jadval formatiga keltiradi va faqat ko‘rsatilgan parametrlar (paket raqami, manba IP, maqsad IP, komanda) bo‘yicha ustun qilib chiqaradi.
> - `tshark -r ... -Y smtp -V | grep -F 'A7: STARTTLS va DKIM nazorat xabari.' || true`:
>   * `-V`: Barcha paket protokollari sarlavhalari va ma’lumotlarini to‘liq ochib ko‘rsatadi.
>   * `grep -F '<matn>'`: Shifrlanmagan ochiq xat matnini qidiradi.
>   * `|| true`: Agar grep hech narsa topmasa, buyruq xato (exit 1) bilan to‘xtab qolmasdan toza exit=0 qaytaradi.
>   * *Vazifasi:* Natijaning bo‘sh chiqishi xat matni tarmoqda ochiq (plaintext) ketmaganini, balki TLS orqali to‘liq himoyalanganini isbotlaydi.

---

### 3.10. 10-Bosqich: Audit Fayllarini Eksport Qilish, SHA-256 va Rollback

#### [DNS-SRV] da audit eksporti:
```bash
mkdir -p ~/A7_hisobot
{ sudo named-checkconf; \
  sudo named-checkzone a7.test /var/cache/bind/db.a7.test; \
  dig @10.70.10.10 a7.test DNSKEY +dnssec; \
  dig @10.70.10.10 _dmarc.a7.test TXT +short; } \
  | tee ~/A7_hisobot/A7_dns_audit.txt
cp ~/A7_hisobot/A7_dns_audit.txt /mnt/a7share/
```

#### [MAIL-SRV] da audit eksporti:
```bash
mkdir -p ~/A7_hisobot
{ sudo postconf -n; \
  sudo opendkim-testkey -d a7.test -s a7 -vvv; \
  sudo openssl x509 -in /etc/postfix/a7-mail.crt \
    -noout -subject -ext subjectAltName -fingerprint -sha256; } \
  2>&1 | tee ~/A7_hisobot/A7_mail_audit.txt
cp ~/A7_hisobot/A7_mail_audit.txt /mnt/a7share/
```

#### [CLIENT] da audit eksporti:
```bash
mkdir -p ~/A7_hisobot
cp ~/A7_dns_smtp.pcap ~/A7_hisobot/
cp ~/A7_dns_smtp.pcap /mnt/a7share/
```

#### Ubuntu xostida yaxlitlik xeshlarini tekshirish va PCAP tahlili (Bash):
Virtual mashinalardan `/mnt/a7share/` orqali ko‘chirilgan barcha audit fayllari (`A7_dns_audit.txt`, `A7_mail_audit.txt`, `A7_dns_smtp.pcap`) avtomatik ravishda Ubuntu xostingizdagi `~/A7-LAB/report/` papkasida paydo bo‘ladi.

Ubuntu xost terminalini oching va quyidagi buyruqlarni bajaring:
```bash
# 1. Hisobot katalogiga o'tish
cd ~/A7-LAB/report

# 2. Barcha dalillar uchun yagona SHA-256 nazorat ro'yxatini yaratish
rm -f A7_SHA256SUMS.txt
sha256sum A7_* > A7_SHA256SUMS.txt
cat A7_SHA256SUMS.txt

# 3. (Ixtiyoriy) PCAP faylini grafik ko'rinishda tahlil qilish uchun Wireshark:
# sudo apt install -y wireshark
wireshark ~/A7-LAB/report/A7_dns_smtp.pcap &
```

> 📌 **10-bosqichdagi CLI buyruqlarining tahlili:**
> - `{ buyruq1; buyruq2; ... } | tee <fayl>`: Figurali qavs ichidagi bir nechta diagnostika buyruqlarini bitta ma’lumotlar oqimiga birlashtiradi; `tee` buyrug‘i esa ularning natijasini ham ekranda ko‘rsatadi, ham audit fayliga saqlaydi.
> - `postconf -n`: Postfix ning standart qiymatlaridan farqli ravishda o‘zgartirilgan barcha faol konfiguratsiya parametrlarini ro‘yxat qilib chiqaradi.
> - `sha256sum A7_* > A7_SHA256SUMS.txt`: Papkadagi barcha audit va PCAP fayllarining SHA-256 kriptografik xeshlarini hisoblab, ro‘yxat fayliga yozadi. Bu fayllarning yaxlitligi (integrity) va keyinchalik soxtalashtirilmaganligini isbotlovchi dalildir.
> - `wireshark ~/A7-LAB/report/A7_dns_smtp.pcap &`: Ubuntu xostida Wireshark grafik interfeysini ochib, ushlangan tarmoq paketlarini grafik tarzda vizual tahlil qilish imkonini beradi.

#### Xavfsiz tiklash (Rollback):
Laboratoriya tugagach, VM lar o‘chiriladi va barchasi boshlang‘ich toza `A7-base` snapshotiga qaytariladi. Bu diskda maxfiy kalitlar (`.key`, `.private`) qolib ketmasligini ta’minlaydi.

---

### 3.11. Natijalarni Qayd Etish Jadvallari va Troubleshooting (Nosozliklarni Tuzatish)

#### 1. Muhit va Xizmatlar Holati Jadvali
| Obyekt / Xizmat | Kutilgan Qiymat | Haqiqiy Qiymat | Dalil ID | Holati |
| :--- | :--- | :--- | :--- | :--- |
| **DNS-SRV** | 10.70.10.10; BIND 9.18 active | 10.70.10.10; active (running) | N-01 | PASS |
| **MAIL-SRV** | 10.70.10.20; Postfix, OpenDKIM | 10.70.10.20; active (running) | N-02 | PASS |
| **CLIENT** | 10.70.10.30; Resolver -> 10.70.10.10 | 10.70.10.30; DNS 10.70.10.10 | N-03 | PASS |
| **Izolyatsiya** | A7-LAB LAN Segment; NAT/Bridged yo‘q | Faqat lokal LAN interfeys | N-04 | PASS |
| **Kalit Ruxsatlari** | `a7.private` va `a7-mail.key` = 0600 | 0600 opendkim / root | N-05 | PASS |

#### 2. DNS va DNSSEC Natijalari Jadvali
| So‘rov / Nazorat | Kutilgan Natija | Kuzatilgan Natija | Dalil ID | Izoh |
| :--- | :--- | :--- | :--- | :--- |
| **SOA / A / MX** | `NOERROR` + `aa` flag | status: NOERROR, flags: qr aa rd | D-01 | Authoritative zona to‘g‘ri ishlayapti |
| **DNSKEY** | ZSK va KSK mavjudligi | 2 ta DNSKEY yozuvi qaytdi | D-02 | Zona kalitlari mavjud |
| **RRSIG** | Yozuvlar uchun imzo | RRSIG SOA, RRSIG A mavjud | D-03 | DNSSEC inline-signing faol |
| **SPF TXT** | `v=spf1 ip4:10.70.10.20 -all` | "v=spf1 ip4:10.70.10.20 -all" | D-04 | .20 ga ruxsat, qolganiga taqiq |
| **DKIM TXT** | `v=DKIM1; ... p=...` | "v=DKIM1; k=rsa; p=..." | D-05 | Public kalit to‘g‘ri e’lon qilingan |
| **DMARC TXT** | `v=DMARC1; p=reject; ...` | "v=DMARC1; p=reject; adkim=s..." | D-06 | Qat’iy siyosat o‘rnatilgan |
| **T1: Rekursiya** | `status: REFUSED` | status: REFUSED, ra yo‘q | D-07 | Open Resolver xavfi yo‘q |
| **T2: AXFR** | `Transfer failed / REFUSED`| Transfer failed. REFUSED | D-08 | Ruxsatsiz zona ko‘chirish bloklangan |

#### 3. Elektron Pochta Xavfsizlik Testlari Jadvali
| Test ID | Tavsif | Kutilgan Verdict | Kuzatilgan Natija | Dalil ID | Holat |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **T3** | STARTTLS siz MAIL FROM | `530 STARTTLS first` | 530 5.7.0 Must issue STARTTLS | M-01 | PASS |
| **T4** | Majburiy TLS bilan local xat | `TLS + 250 accepted` | 250 2.0.0 Ok: queued as ... | M-02 | PASS |
| **T5** | Ruxsatsiz tashqi relay | `554 Relay denied` | 554 5.7.1 Relay access denied | M-03 | PASS |
| **T6** | SPF ruxsat etilgan IP (.20) | `pass` | spfquery: result: pass | M-04 | PASS |
| **T7** | SPF ruxsat etilmagan IP (.30)| `fail` | spfquery: result: fail | M-05 | PASS |
| **DKIM** | Ochiq kalit va imzo tekshiruvi | `key OK; DKIM-Signature` | opendkim-testkey: key OK | M-06 | PASS |
| **Fingerprint**| Sertifikat SHA-256 taqqoslash | CLIENT = MAIL-SRV | Fingerprintlar 100% teng | M-07 | PASS |

#### Nosozliklarni Bartaraf Etish (Troubleshooting)
| Alomat / Muammo | Ehtimoliy Sabab | Qanday Tekshiriladi | Tuzatish Yo‘li |
| :--- | :--- | :--- | :--- |
| `apt update` xatosi (`Connection refused`, `Timeout`, `407`) | Shaxsiy lokal Wi-Fi da universitet proksisi (`172.18.1.1:2500`) yoki `99proxy` qolib ketgan | `apt-config dump \| grep -i proxy`<br>`env \| grep -i proxy` | `sudo rm -f /etc/apt/apt.conf.d/99proxy`<br>`unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY` |
| `SERVFAIL` yoki zona ochilmaydi | TXT qo‘shtirnoqlarida yoki serialda xato; ruxsat xato | `sudo named-checkconf`<br>`sudo named-checkzone a7.test /var/cache/bind/db.a7.test` | Sintaksisni to‘g‘rilash, serialni oshirish, `chown bind:bind` qilish |
| RRSIG yozuvlari chiqmayapti | DNSSEC avtomatik imzolash hali yakunlanmagan yoki kalit katalogi yozilmaydi | `sudo rndc signing -list a7.test`<br>`ls -la /var/cache/bind/keys` | Tizim vaqtini tekshirish, bind foydalanuvchisiga ruxsat berish |
| Tashqi so‘rovga javob beryapti (T1 fail) | Rekursiya yoki forwarders ochiq qolgan | `dig @10.70.10.10 google.com`<br>`cat /etc/bind/named.conf.options` | `recursion no;` ni qo‘yish, forwarders qatorini butunlay o‘chirish |
| AXFR ishlayapti (T2 fail) | `allow-transfer` ko‘rsatilmagan | `dig @10.70.10.10 a7.test AXFR` | `named.conf.options` ga `allow-transfer { none; };` qo‘shish |
| `opendkim-testkey: no key` | DNS selector TXT nomi yoki public kalit noto‘g‘ri | `dig a7._domainkey.a7.test TXT` | `a7.txt` dagi `p=` kalitini BIND zona fayliga to‘g‘ri ko‘chirish |
| Xat yetib boryapti, lekin DKIM imzo yo‘q | CLIENT IP si ishonchli xostlar ro‘yxatida yo‘q yoki milter ulanmagan | `cat /etc/opendkim/trusted.hosts`<br>`sudo postconf smtpd_milters` | `10.70.10.30` ni `trusted.hosts` ga qo‘shish, xizmatlarni qayta yoqish |
| T4 testi `530` xatosi bilan qolib ketmoqda | Swaks komandasiga `--tls` kaliti berilmagan | `swaks ... --tls` | Swaks buyrug‘iga `--tls` qo‘shish, sertifikat ruxsatini (0600) tekshirish |
| T5 ruxsatsiz relayga ruxsat beryapti | CLIENT IP si `mynetworks` ichiga kiritib qo‘yilgan | `sudo postconf mynetworks` | `mynetworks = 127.0.0.0/8` qilib toraytirish |

---

### 3.12. O‘qituvchi Uchun Himoya Savollari va Ularning Mukammal Javoblari

1. **Authoritative DNS javobidagi AA flag nimani anglatadi?**
   *Javob:* `AA` (*Authoritative Answer*) bayrog‘i so‘rovga qaytgan javob oraliq keshlovchi (recursive resolver) serverdan emas, balki aynan shu zonaning birlamchi (authoritative) ma’lumotlar bazasidan kelganini bildiradi.

2. **Recursionni authoritative serverda o‘chirish qaysi amplification riskini kamaytiradi?**
   *Javob:* Rekursiyani o‘chirish serverning **Open Resolver**ga aylanishining oldini oladi. Natijada tajovuzkorlar soxta (spoofed) manba IP manzillar bilan kichik DNS so‘rov yuborib, qurbon serverga katta hajmdagi javoblarni qaytarish orqali amalga oshiriladigan **DNS Amplification (Kuchaytirilgan DDoS)** hujumlarida ushbu serverdan foydalana olmaydilar.

3. **DNS zone transfer nega TCP/53 orqali ham tekshiriladi?**
   *Javob:* Odatiy DNS so‘rovlari tezkorlik uchun UDP/53 da ishlaydi, biroq to‘liq zonani ko‘chirish (`AXFR`) katta hajmdagi ma’lumotlarni talab qilganligi sababli faqat ishonchli TCP/53 protokoli orqali amalga oshiriladi. Shuning uchun AXFR blokirovkasi aynan TCP/53 da tekshiriladi.

4. **DNSKEY va RRSIG mavjudligi nega global DNSSEC validation bilan bir xil emas?**
   *Javob:* DNSKEY va RRSIG zona ma’lumotlari lokal darajada imzolanganini isbotlaydi. Ammo global tekshiruv (validation) uchun yuqori darajadagi zona (ildiz zonasi)da ushbu kalitning xeshi — **DS (Delegation Signer)** yozuvi bo‘lishi va ishonch zanjiri (**Chain of Trust**) uzilmasligi kerak. Mahalliy `.test` zonasida global DS bo‘lmagani uchun to‘liq global validatsiya bo‘lmaydi.

5. **SOA serial o‘zgarmasa zona yangilanishida qanday muammo yuz beradi?**
   *Javob:* Ikkilamchi (Secondary) DNS serverlar zonadagi o‘zgarishlarni aynan SOA yozuvidagi serial raqami oshganiga qarab aniqlaydilar. Agar serial oshirilmasa, secondary serverlar va keshlovchi tizimlar zona yangilanganini bilmaydi va eski (keshlangan) ma’lumotlarni berishda davom etadi.

6. **SPF qaysi identifikator va IPni tekshiradi, Header Fromni nega to‘liq himoyalamaydi?**
   *Javob:* SPF faqat SMTP konvertidagi `MAIL FROM` (Envelope Sender / Return-Path) domenini va xatni yuborayotgan serverning manba IP manzilini tekshiradi. Ammo foydalanuvchining pochta mijozi (MUA) interfeysida ko‘rinadigan `Header From:` sarlavhasini tekshirmaydi. Hujumchi konvertda o‘z domenini, sarlavhada esa soxta domenni ko‘rsatib SPFni chetlab o‘tishi mumkin.

7. **DKIM `d=` va `s=` maydonlari DNSdagi qaysi owner nomini hosil qiladi?**
   *Javob:* Bu maydonlar birlashib DNSdagi quyidagi nomni hosil qiladi: `<s=selector>._domainkey.<d=domain>`. Bizning laboratoriyada `s=a7` va `d=a7.test` bo‘lgani uchun DNS yozuvi `a7._domainkey.a7.test` bo‘ladi.

8. **DMARC strict alignmentda SPF yoki DKIMdan qaysi shartlar yetarli?**
   *Javob:* Strict rejimda (`adkim=s; aspf=s`) xat DMARC dan o‘tishi uchun:
   - Yo SPF `pass` bo‘lib, uning `MAIL FROM` domeni xatdagi `Header From` domeniga subdomensiz, 100% aynan teng bo‘lishi kerak;
   - Yo DKIM `pass` bo‘lib, uning `d=` domeni `Header From` domeniga 100% aynan teng bo‘lishi kerak.
   DMARC muvaffaqiyatli o‘tishi uchun ushbu ikkita shartdan kamida bittasi to‘liq bajarilishi kifoya.

9. **Nega SPF fail bo‘lsa ham aligned DKIM pass DMARCni o‘tkazishi mumkin?**
   *Javob:* DMARC spetsifikatsiyasiga (RFC 9989) ko‘ra, elektron xat qayta yo‘naltirilganda (forwarding) jo‘natuvchi server IPsi o‘zgaradi va SPF tabiiy ravishda fail bo‘ladi. Agar xat yo‘lda o‘zgarmagan bo‘lsa, DKIM kriptografik imzosi o‘z kuchini saqlab qoladi. Shuning uchun DMARC mantiqan `(SPF aligned pass) OR (DKIM aligned pass)` sharti asosida ishlaydi.

10. **STARTTLS shifrlash bilan SMTP relay authorization o‘rtasidagi farq nima?**
    *Javob:* STARTTLS — bu **transport qatlami shifrlashidir**, u mijoz va server o‘rtasidagi trafikni tinglashdan (sniffing) himoya qiladi. SMTP Relay Authorization esa **murojaat huquqini (avtorizatsiyani) tekshirishdir**, ya’ni u qaysi foydalanuvchi yoki qaysi IP manzil boshqa tashqi domenlarga xat yuborishga haqli ekanini belgilaydi. Shifrlangan kanal orqali ham noqonuniy spam relay qilishga urinish mumkin, shuning uchun ikkala mexanizm mustaqil ishlashi shart.

11. **Self-signed sertifikat fingerprintini out-of-band taqqoslash nimani kamaytiradi?**
    *Javob:* Self-signed sertifikatlar ommaviy CA (Certificate Authority) tomonidan tasdiqlanmaganligi sababli, oraliqdagi tajovuzkor soxta sertifikat tiqishtirishi (**Man-in-the-Middle**) mumkin. Xavfsiz kanal orqali (masalan, server konsolidan to‘g‘ridan-to‘g‘ri) olingan SHA-256 xeshini mijoz ko‘rgan xesh bilan taqqoslash (out-of-band verification) aynan haqiqiy serverga ulanganlikni kafolatlaydi.

12. **`milter_default_action=tempfail` xavfsizlik va mavjudlikka qanday trade-off beradi?**
    *Javob:* Agar OpenDKIM filtri qulasa:
    - `tempfail` xabarni vaqtincha rad etadi (451 xatosi). Bu xavfsizlik nuqtai nazaridan to‘g‘ri (**Fail-Secure**), chunki xatlar imzosiz chiqib ketmaydi, ammo xizmat mavjudligi (availability) vaqtincha pasayadi.
    - `accept` esa xizmat uzluksizligini saqlaydi, lekin xatlar imzosiz ketib, xavfsizlik siyosati buziladi (**Fail-Open**).

13. **PCAPda STARTTLSdan oldin va keyin qaysi ma’lumotlar ko‘rinadi?**
    *Javob:* STARTTLSdan oldin TCP handshake, SMTP server banti (`220`), `EHLO` salomlashuvi va `STARTTLS` buyrug‘i ochiq matnda ko‘rinadi. STARTTLS buyrug‘i va TLS handshake yakunlangach, barcha keyingi paketlar (jo‘natuvchi, qabul qiluvchi, xat sarlavhalari va matni) shifrlangan TLS Application Data sifatida ko‘rinadi va ularni o‘qib bo‘lmaydi.

14. **Ishlab chiqarish muhitida MTA-STS, DANE, OpenDMARC/Rspamd va monitoring qayerda qo‘shiladi?**
    *Javob:*
    - **MTA-STS va DANE (TLSA):** STARTTLSni majburlash va MITM downgrade hujumlaridan himoyalanish uchun DNS va HTTPS orqali qo‘shiladi;
    - **OpenDMARC / Rspamd:** Qabul qiluvchi (Inbound MX) serverda kiruvchi xatlarning DMARC siyosatini tekshirish va `p=reject` ni amalda ijro etish uchun o‘rnatiladi;
    - **Monitoring (Prometheus/Grafana, SIEM):** Postfix, BIND va OpenDKIM loglarini real vaqtda tahlil qilish, rad etilgan so‘rovlar (530, 554, REFUSED) bo‘yicha xavfsizlik hodisalarini kuzatib borish uchun ulanadi.

---

### 3.13. Talabaning Yakuniy Xulosasi (Namuna)
*(Ushbu matn laboratoriya hisobotining 8.1-bo‘limiga kiritish uchun 6–8 gaplik professional akademik xulosa namunasidir)*

> Mazkur laboratoriya ishi doirasida izolyatsiyalangan A7-LAB muhitida BIND9 va Postfix/OpenDKIM xizmatlarining o‘zaro integratsiyalashgan xavfsizlik arxitekturasi amaliy sinovdan o‘tkazildi. BIND9 authoritative serveri tomonidan qaytarilgan `AA` bayrog‘i va `inline-signing` natijasida hosil bo‘lgan `DNSKEY` hamda `RRSIG` yozuvlari zona yaxlitligi kriptografik himoyalanganini tasdiqladi. Keshlovchi rekursiyaning rad etilishi (`REFUSED`) va zone transferning bloklanishi serverning Open Resolver yoki razvedka manbaiga aylanmasligini isbotladi. Postfix pochtasida 25-portda STARTTLS majburiy qilinishi oqibatida shifrlanmagan ulanishlar `530` kodi bilan, ruxsatsiz tashqi relay urinishlari esa `554 Relay access denied` xatosi bilan qat’iy to‘xtatildi. OpenDKIM orqali mahalliy qabul qilingan xat muvaffaqiyatli `DKIM-Signature` bilan imzolandi hamda SPF tekshiruvida ruxsat berilgan IP (.20) uchun `pass`, begona IP (.30) uchun `fail` natijasi olindi. DMARC strict alignment tahlili orqali Header From, DKIM va SPF identifikatorlarining to‘liq muvofiqligi ko‘rsatildi, shuningdek, Wireshark/tshark PCAP tahlili STARTTLS dan keyingi xat mazmuni tarmoqda to‘liq shifrlanganini tasdiqladi.

---

## 4-QISM: RASMIY VA ISHONCHLI MANBALAR

Mazkur laboratoriya ishi va uning xavfsizlik siyosatlari quyidagi xalqaro standartlar (RFC) hamda dasturiy ta’minotlarning rasmiy hujjatlariga asoslangan:

1. **DNS va BIND9 bo‘yicha:**
   - **ISC BIND 9.18 Configuration Reference:** [https://bind9.readthedocs.io/en/v9.18.41/reference.html](https://bind9.readthedocs.io/en/v9.18.41/reference.html) — BIND 9 konfiguratsiya parametrlari, ACL va xavfsizlik optsiyalari bo‘yicha birlamchi qo‘llanma.
   - **ISC BIND 9.18 DNSSEC Guide:** [https://bind9.readthedocs.io/en/v9.18.41/dnssec-guide.html](https://bind9.readthedocs.io/en/v9.18.41/dnssec-guide.html) — DNSSEC kalitlari, avtomatik imzolash siyosati (`dnssec-policy`) va validatsiya qo‘llanmasi.
   - **RFC 4033, 4034, 4035:** DNS Security Introduction and Requirements, Resource Records for the DNS Security Extensions.
   - **RFC 6761:** Special-Use Domain Names (`.test` domeni bo‘yicha standart).

2. **Elektron Pochta Xavfsizligi va Postfix bo‘yicha:**
   - **Postfix SMTP Relay and Access Control:** [https://www.postfix.org/SMTPD_ACCESS_README.html](https://www.postfix.org/SMTPD_ACCESS_README.html) — Postfixda ruxsatsiz relayni bloklash, `smtpd_relay_restrictions` va `mynetworks` sozlash qo‘llanmasi.
   - **Postfix Configuration Parameters (postconf):** [https://www.postfix.org/postconf.5.html](https://www.postfix.org/postconf.5.html) — Barcha Postfix parametrlari katalogi.
   - **Postfix TLS Support:** [https://www.postfix.org/TLS_README.html](https://www.postfix.org/TLS_README.html) — STARTTLS shifrlashini majburlash va sertifikatlar boshqaruvi.

3. **DKIM va OpenDKIM bo‘yicha:**
   - **OpenDKIM Configuration Manual:** [https://manpages.debian.org/bookworm/opendkim/opendkim.conf.5.en.html](https://manpages.debian.org/bookworm/opendkim/opendkim.conf.5.en.html) — OpenDKIM milterini sozlash parametrlari.
   - **OpenDKIM testkey Manual:** [https://manpages.debian.org/bookworm/opendkim-tools/opendkim-testkey.8.en.html](https://manpages.debian.org/bookworm/opendkim-tools/opendkim-testkey.8.en.html) — DNSdagi public key mosligini tekshirish vositasi.
   - **RFC 6376:** DomainKeys Identified Mail (DKIM) Signatures.

4. **SPF va DMARC Standartlari bo‘yicha:**
   - **RFC 7208:** Sender Policy Framework (SPF) for Authorizing Use of Domains in Email.
   - **RFC 9989 (eski RFC 7489 o‘rniga yangilangan):** Domain-based Message Authentication, Reporting, and Conformance (DMARC).
   - **RFC 8461:** SMTP MTA Strict Transport Security (MTA-STS).

5. **Operatsion Tizim Paketlari:**
   - **Ubuntu 24.04 Noble Numbat Mail Packages:** [https://packages.ubuntu.com/noble/mail/](https://packages.ubuntu.com/noble/mail/) — Ubuntu 24.04 LTS uchun rasmiy pochta va xavfsizlik paketlari arxivi.
