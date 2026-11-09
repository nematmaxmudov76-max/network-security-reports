# 8-BOB: KOMPYUTER TARMOQLARIDA XAVFSIZLIK (Security in Computer Networks)
## Mukammal va Keng Qamrovli Texnik Qo‘llanma

> **Ushbu bobning maqsadi:**  
> Kiberxavfsizlikning fundamental tamoyillari (CIA triadasi), simmetrik (AES) va asimmetrik (RSA matematikasi) shifrlash, xesh-funksiyalar (SHA-256) va raqamli imzolar, PKI va X.509 raqamli sertifikatlar zanjiri, SSL/TLS 1.3 protokoli va shifrlangan veb (HTTPS), Tarmoq darajasidagi xavfsizlik — IPsec VPN (AH, ESP, Tunnel rejimi, IKEv2), Simsiz tarmoq xavfsizligi (WEP zaifliklari, WPA2, WPA3-SAE, 802.11w PMF), Xavfsizlik devorlari (Stateless vs Stateful, DMZ zonalari) hamda Hujumlarni aniqlash va oldini olish tizimlari (IDS/IPS, Suricata) ni chuqur professional darajada o‘rganish.

---

## 8.1. TARMOQ XAVFSIZLIGI NIMA O‘ZI? FUNDAMENTAL TALABLAR

Tarmoq xavfsizligi — bu axborotni uzatish, saqlash va qayta ishlash jarayonida uni ruxsatsiz kirish, o‘zgartirish, yo‘q qilish yoki xizmatdan chiqarishdan himoya qilish choralari majmuasidir.

Kriptografiya va xavfsizlik nazariyasida doimo 3 ta ramziy qahramon ishlatiladi:
* **Alice va Bob:** O‘zaro xavfsiz ma’lumot almashmoqchi bo‘lgan ishonchli tomonlar.
* **Trudy (Intruder / Attacker):** Aloqa kanalini poylayotgan, xabarlarni tutib oluvchi, o‘zgartiruvchi yoki soxtalashtiruvchi tajovuzkor (xaker).

```
[ Alice (Jo'natuvchi) ] ====================================> [ Bob (Qabul qiluvchi) ]
                                      |
                              [ Trudy (Xaker) ]
                              * Eshitadi (Eavesdropping / Sniffing)
                              * O'zgartiradi (Tampering / Modification)
                              * Soxtalashtiradi (Spoofing / Impersonation)
                              * To'sib qo'yadi (Denial of Service)
```

---

### Xavfsiz Tizimning 5 ta Asosiy Ustuni:

1. **Maxfiylik (Confidentiality):**
   * Xabarni faqat mo‘ljallangan qabul qiluvchi (Bob) o‘qiy olishi kerak.
   * Agar Trudy paketni tutib olsa ham, u shifrlangan bo‘lgani uchun hech narsani tushunmasligi shart.
2. **Yaxlitlik (Message Integrity):**
   * Xabar jo‘natuvchidan chiqib qabul qiluvchiga yetib borguncha yo‘lda Trudy tomonidan tasodifan yoki qasddan o‘zgartirilmaganligi kafolatlanishi kerak. (Masalan, "Hisobimga 100$ o‘tkaz" degan xabar yo‘lda "100 000$ o‘tkaz" bo‘lib qolmasligi shart).
3. **Oxirgi nuqtalarni tekshirish (End-Point Authentication):**
   * Bob muloqot qilayotgan shaxs haqiqatan ham Alice ekanligiga 100% ishonch hosil qilishi kerak (Trudy o‘zini Alice deb aldab kirmasligi lozim).
4. **Vakolat va Kirishni cheklash (Operational Security / Access Control):**
   * Foydalanuvchilar faqat o‘zlariga ruxsat berilgan resurslargagina kira olishi, begona xizmatlar (portlar) tashqi tarmoq uchun butunlay yopiq bo‘lishi kerak.
5. **Rad etolmaslik (Non-Repudiation):**
   * Alice xabarni yuborgandan so‘ng: *"Men bu xatni yubormaganman"* deb o‘z harakatini inkor qila olmasligi shart (Raqamli imzo orqali ta’minlanadi).

---

## 8.2. KRIPTOGRAFIYA TAMOYILLARI

Kriptografiya — bu ochiq matnni (**Plaintext $m$**) maxfiy kalit (**Key $K$**) yordamida shifrlangan matnga (**Ciphertext $c$**) aylantirish va aksincha jarayonni o‘rganuvchi fan.

$$\text{Shifrlash: } c = K_A(m) \qquad \text{Deshifrlash: } m = K_B(c)$$

> **Kerkxoffs Qoidasi (Kerckhoffs' Principle):**  
> Xavfsizlik shifrlash algoritmining maxfiyligiga tayanmasligi kerak! Barcha zamonaviy algoritmlar (AES, RSA) dunyo uchun ochiq va ma’lum. Xavfsizlik **faqat va faqat Kalitning ($K$) maxfiyligiga** tayanishi shart!

---

### 8.2.1. Simmetrik Kalitli Kriptografiya (Symmetric-Key Cryptography)

Jo‘natuvchi va qabul qiluvchi **yagona bir xil umumiy maxfiy kalitdan ($K_S$)** foydalanadi:

$$c = K_S(m) \qquad m = K_S(c)$$

```
[ Ochiq Matn m ] ---> [ AES Shifrlash ] === (Ciphertext c) ===> [ AES Deshifrlash ] ---> [ Ochiq Matn m ]
                             ^                                          ^
                             |                                          |
                      (Maxfiy Kalit K)                          (Bir xil Kalit K)
```

1. **DES (Data Encryption Standard):**
   * 1977-yilda qabul qilingan. 64-bitli bloklar va atigi **56-bitli kalit** ishlatadi.
   * Bugungi kunda eskirgan va xavfli: zamonaviy kompyuterlar 56-bitli kalitni Brute-force (barcha variantlarni terib chiqish) orqali bir necha soatda buzib tashlaydi.
2. **AES (Advanced Encryption Standard — FIPS 197):**
   * 2001-yilda Rijndael algoritmi asosida qabul qilingan bugungi kunning jahon standarti.
   * Ma’lumotlarni **128-bitli bloklar** ko‘rinishida shifrlaydi.
   * Kalit uzunligi: **128 bit, 192 bit yoki 256 bit** bo‘lishi mumkin.
   * *Buzishning imkoni bormi?* 256-bitli kalitning kombinatsiyalar soni $2^{256} \approx 1.15 \times 10^{77}$ ta! Dunyodagi barcha superkompyuterlar milliardlab yil hisoblasa ham uni kuch bilan (brute-force) sindira olmaydi.

#### Blok Shifrlash Rejimlari (Cipher Block Modes):
* **ECB (Electronic Codebook):** Har bir 128-bitli blok bir xil kalit bilan alohida shifrlanadi. *Juda xavfli!* Agar ma’lumotda bir xil so‘zlar takrorlansa, shifrlangan matnda ham bir xil qismlar ko‘rinib qoladi (taniqli "ECB Penguin" tasviri).
* **CBC (Cipher Block Chaining):** Har bir blok shifrlanishidan oldin o‘zidan oldingi shifrlangan blok bilan **XOR** qilinadi. Eng birinchi blok uchun esa tasodifiy **IV (Initialization Vector — Boshlang‘ich Vektor)** ishlatiladi. Natijada bir xil so‘zlar butunlay boshqa-boshqa tushunarsiz baytlarga aylanadi.

---

### 8.2.2. Asimmetrik (Ochiq Kalitli) Kriptografiya: RSA Algoritmi

Simmetrik shifrlashning eng katta muammosi: Alice va Bob umrida birinchi marta internet orqali uchrashganda, umumiy maxfiy kalitni bir-biriga qanday qilib Trudy o‘g‘irlamaydigan qilib yetkazishi mumkin?

1976-yilda Diffie va Hellman, so‘ngra Rivest, Shamir va Adleman (**RSA**) bunga dahshatli yechim topdi: **Har bir foydalanuvchida 2 ta kalit bo‘ladi!**
1. **Ochiq kalit ($K^+$ — Public Key):** Dunyodagi hamma uchun ochiq, internetga e’lon qilinadi.
2. **Shaxsiy maxfiy kalit ($K^-$ — Private Key):** Faqat egasining o‘zida qat’iy sir saqlanadi.

```
Alice Bobga xat yozmoqchi:
[ Xat m ] ---> [ RSA Shifrlash (Bobning OCHIQ kaliti K_B^+ bilan) ] ===> (Ciphertext c)
                                                                               |
[ Xat m ] <--- [ RSA Deshifrlash (Faqat Bobning MAXFIY kaliti K_B^- bilan) ] <-+
```

* Bobning ochiq kaliti bilan shifrlangan xatni **faqat va faqat Bobning shaxsiy maxfiy kaliti ochishi mumkin**! Hatto xatni shifrlagan Alice ham uni qayta o‘qiy olmaydi.

---

### 8.2.3. RSA Algoritmining Matematik Asosi (Qadam-baqadam)

RSA algoritmi **katta sonlarni tub ko‘paytuvchilarga ajratish (faktoriallash) o‘ta qiyin** ekanligiga tayanadi.

#### Algoritm Bosqichlari:
1. Ikkita juda katta tub son tanlanadi: $p$ va $q$ (amaliyotda har biri 1024 bitdan).
2. Ularning ko‘paytmasi hisoblanadi:
   $$n = p \times q \qquad z = (p - 1) \times (q - 1)$$
3. $z$ soni bilan umumiy bo‘luvchiga ega bo‘lmagan (o‘zaro tub) $e$ soni tanlanadi ($e < n$).
4. Quyidagi matematik tenglamani qanoatlantiruvchi $d$ soni topiladi:
   $$e \times d \equiv 1 \pmod z \quad \iff \quad (e \times d - 1) \text{ soni } z \text{ ga qoldiqsiz bo'linadi}$$
5. **Kalitlar tayyor:**
   * **Ochiq kalit (Public Key):** Juftlik $(n, e)$
   * **Maxfiy kalit (Private Key):** Juftlik $(n, d)$

#### Shifrlash va Deshifrlash Formulalari:
* **Shifrlash:** Ochiq matn $m$ soniga aylantiriladi ($m < n$):
  $$c = m^e \bmod n$$
* **Deshifrlash:**
  $$m = c^d \bmod n$$

#### Kichik Sonlar Bilan Jonli Matematik Misol:
1. $p = 5$, $q = 7$ deb olamiz (ikkalasi tub son).
2. $n = 5 \times 7 = 35$, $z = (5 - 1) \times (7 - 1) = 4 \times 6 = 24$.
3. $z = 24$ bilan o‘zaro tub bo‘lgan $e = 5$ sonini tanlaymiz ($\text{EKUB}(5, 24) = 1$).
4. $5 \times d \equiv 1 \pmod{24}$ tenglamadan $d$ ni topamiz:
   * Agar $d = 29$ bo‘lsa: $5 \times 29 = 145$. $145 \bmod 24 = 1$. (Demak, $d = 29$ yoki eng kichigi $d = 5$). Keling, $e=7$ deb olsak: $7 \times d \equiv 1 \pmod{24} \implies 7 \times 7 = 49 \implies 49 \equiv 1 \pmod{24}$. Demak $d=7$.
   * Keling, $e = 5$ va $d = 29$ juftligini olamiz.
5. **Amalda sinaymiz:**
   * Bizning xabarimiz: $m = 12$.
   * **Shifrlash:**
     $$c = 12^5 \bmod 35 = 248832 \bmod 35 = 17$$
     *(Xaker faqat $17$ sonini ko‘radi!)*
   * **Deshifrlash:**
     $$m = 17^{29} \bmod 35 = 12$$
     *(Mo‘jiza! Asl $12$ xabari to‘liq qaytib chiqdi!)*

---

## 8.3. XABAR YAXLITLIGI, RAQAMLI IMZO VA PKI

### 8.3.1. Kriptografik Xesh-Funksiyalar (Cryptographic Hash Functions)

Xesh-funksiya ixtiyoriy ulkan hajmdagi ma’lumotni ($m$) qat’iy belgilangan o‘zgarmas uzunlikdagi qisqa qatorga — **Dajestga (Digest / Hash $H(m)$)** aylantirib beradi.

Kriptografik xesh-funksiya 2 ta qat’iy shartni bajarishi shart:
1. **Bir tomonlama bo‘lishi (One-Way):** $H(m)$ berilgan bo‘lsa, undan asl $m$ matnini qayta hisoblab topish amalda imkonsiz bo‘lishi kerak.
2. **Kolliziyaga chidamlilik (Collision-Resistant):** Bir xil xesh qiymat beruvchi ikkita har xil matnni ($m_1 \ne m_2$, lekin $H(m_1) = H(m_2)$) topish hisoblash nuqtai nazaridan imkonsiz bo‘lishi shart.

* **Eskirgan va xavfli:** `MD5` (128 bit) va `SHA-1` (160 bit) — bugungi kunda kolliziyalari topilgan, ulardan foydalanish taqiqlangan!
* **Zamonaviy xavfsiz standart:** **SHA-256 (SHA-2 oilasi)** va **SHA-3** — 256 bitlik xesh hosil qiladi, bugungi kriptografiyaning tayanchidir.

---

### 8.3.2. Raqamli Imzo (Digital Signatures)

Haqiqiy hayotda hujjat ostiga shaxsiy imzo qo‘yiladi. Raqamli dunyoda buni qanday qilamiz?
Raqamli imzo — bu **xabar xeshining jo‘natuvchining SHAXSIY MAXFIY KALITI bilan shifrlangan holatidir**!

```
[ Alice (Xat yozuvchi) ]                                 [ Bob (Tekshiruvchi) ]
Xat m                                                    Xat m keldi
  |                                                        |
  +---> SHA-256 xesh: H(m)                                 +---> SHA-256 xesh: H(m)
            |                                                        |
            v                                                        v
      [ RSA Shifrlash ]                                      (Taqqoslaydi: == ?)
      (Alice MAXFIY kaliti K_A^- bilan!)                             ^
            |                                                        |
      Raqamli Imzo S = K_A^-(H(m))                                   |
            |                                                        |
            +======== Xat m va Imzo S Bobga yuboriladi ============> [ RSA Ochish ]
                                                                     (Alice OCHIQ kaliti K_A^+ bilan!)
```

#### Raqamli Imzo Nima Beradi?
1. **Autentifikatsiya:** Imzoni faqat Alice ning ochiq kaliti ocha oldimi? Demak, bu xatni dunyoda faqat Alice yozgan (chunki uning maxfiy kaliti faqat o‘zida bor!).
2. **Yaxlitlik:** Agar yo‘lda Trudy xatdagi bitta harfni o‘zgartirsa, Bob hisoblagan xesh imzo ichidagi xeshga mos kelmaydi va soxtalik darhol fosh bo‘ladi!
3. **Rad etolmaslik:** Alice sud oldida: *"Bu xatni men yozmaganman"* deya olmaydi.

---

### 8.3.3. Ochiq Kalitlar Infratuzilmasi (PKI) va Raqamli Sertifikatlar

Alice Bobning ochiq kalitini internetdan yuklab oldi. Lekin u kalit haqiqatan ham Bobnikimi?
Agar o‘rtada turgan Trudy: *"Bobning ochiq kaliti mana shu"* deb o‘zining kalitini berib qo‘ysachi (**Man-in-the-Middle hujumi**)?

Buning oldini olish uchun **Sertifikatlash Markazlari (CA — Certification Authority)** va **PKI (Public Key Infrastructure)** tizimi yaratilgan.

```
+-------------------------------------------------------------------------+
|                  X.509 RAQAMLI SERTIFIKATI                              |
+-------------------------------------------------------------------------+
| * Egasi: www.bank.uz (Toshkent, O'zbekiston)                            |
| * Egasining Ochiq Kaliti: K_bank^+ (RSA 2048 bit / ECC 256 bit)         |
| * Amal qilish muddati: 2026-yil 1-yanvardan 2027-yil 1-yanvargacha     |
| * Beruvchi (Issuer): Let's Encrypt / DigiCert CA                        |
|                                                                         |
| [ CA NING RAQAMLI IMZOSI: K_CA^-( H(Ushbu barcha ma'lumotlar) ) ]       |
+-------------------------------------------------------------------------+
```

* **Zanjirli Ishonch (Chain of Trust):**
  * Dunyodagi barcha operatsion tizimlar (Windows, Linux, Android) va brauzerlar (Chrome, Firefox) o‘z ichida dunyoning eng ishonchli **Root CA (Ildiz sertifikatlashtirish markazlari)** ro‘yxati va ularning ochiq kalitlari bilan birga o‘rnatiladi.
  * Saytga kirganingizda, brauzer sayt sertifikatidagi CA imzosini o‘zida bor ochiq kalit bilan tekshiradi. Agar imzo to‘g‘ri bo‘lsa, brauzerda **"Yashil qulfcha" (Xavfsiz ulanish)** yonadi!

---

## 8.4. XAVFSIZ TCP ULANISHLARI: SSL VA TLS (TRANSPORT LAYER SECURITY)

Oddiy HTTP orqali uzatilgan barcha parollar tarmoqda ochiq matn ko‘rinishida oqadi. Unga xavfsizlik berish uchun TCP ustida **TLS (avvalgi nomi SSL — Secure Sockets Layer)** ishga tushadi.

TLS bilan himoyalangan protokol **HTTPS (Port 443)** deb ataladi.

```
+---------------------------+
| 5. Application (HTTP)     |
+---------------------------+
|    TLS / SSL Qatlami      |  <--- Shifrlash, Yaxlitlik, Autentifikatsiya
+---------------------------+
| 4. Transport (TCP)        |
+---------------------------+
| 3. Network (IP)           |
+---------------------------+
```

---

### 8.4.1. TLS 1.3 Handshake Jarayoni (Qanday Aloqa O‘rnatiladi?)

TLS ikki xil kriptografiyaning kuchini birlashtiradi:
* **Asimmetrik shifrlash (RSA/Diffie-Hellman):** O‘ta sekin, lekin sessiya kalitini xavfsiz almashish uchun ishlatiladi.
* **Simmetrik shifrlash (AES-GCM):** O‘ta tezkor, barcha keyingi asosiy ma’lumotlarni shifrlash uchun ishlatiladi.

```
[ Brauzer (Mijoz) ]                                        [ Veb-Server ]
         |                                                        |
         | 1. ClientHello (Qo'llab-quvvatlaydigan shifrlash       |
         |    usullari: Cipher Suites, Tasodifiy son R_C)         |
         |------------------------------------------------------->|
         |                                                        |
         | 2. ServerHello (Tanlangan shifr: TLS_AES_256_GCM,      |
         |    Tasodifiy son R_S) + RAQAMLI SERTIFIKAT             |
         |<-------------------------------------------------------|
         |                                                        |
         | (Brauzer CA orqali sertifikatni tekshiradi!)           |
         |                                                        |
         | 3. Pre-Master Secret kaliti yaratiladi va serverning   |
         |    ochiq kaliti bilan shifrlanib yuboriladi            |
         |------------------------------------------------------->|
         |                                                        |
         | [ Ikkala tomon mustaqil ravishda SIMMETRIK SESSIYA     |
         |   KALITLARINI (Master Secret) hisoblab chiqaradi! ]    |
         |                                                        |
         |<=========== 4. TO'LIQ SHIFRLANGAN HTTPS MA'LUMOTLAR ===>|
         |             (AES-256 va HMAC bilan himoyalangan)       |
```

---

## 8.5. TARMOQ QATLAMI XAVFSIZLIGI: IPSEC VA VPN (VIRTUAL PRIVATE NETWORKS)

TLS faqat L4/L5 darajasida (brauzerda) ishlaydi. Lekin butun bir tashkilotning barcha tarmoq trafigini (DNS, Ping, ma’lumotlar bazasi, ichki fayllar) shifrlash kerak bo‘lsachi?
Buning uchun L3 Tarmoq qatlamida ishlovchi **IPsec (IP Security — RFC 4301)** ishlatiladi.

### 8.5.1. VPN Turlari
1. **Site-to-Site VPN:** Ikkita uzoq masofadagi filial tarmoqlarini (masalan, Toshkent va Samarqand ofislari) ochiq Internet orqali yagona xavfsiz ichki korporativ tarmoqqa birlashtirish.
2. **Remote-Access VPN:** Masofadan turib uyda ishlayotgan xodimning o‘z noutbukidan turib korxona ichki tarmog‘iga xavfsiz ulanishi.

---

### 8.5.2. IPsec Protokollari: AH va ESP

* **AH (Authentication Header — IP Protocol 51):**
  * Paketning manbasi haqiqiyligini va yaxlitligini ta’minlaydi.
  * **Lekin ma’lumotni shifrlamaydi!** Paket ichidagi ma’lumotlar ochiq ketadi (amaliyotda kam ishlatiladi).
* **ESP (Encapsulating Security Payload — IP Protocol 50):**
  * Asosiy protokol. **Ham to‘liq shifrlaydi (Maxfiylik), ham yaxlitlik va autentifikatsiyani ta’minlaydi.**

---

### 8.5.3. IPsec Rejimlari: Transport Mode vs Tunnel Mode

```
1. TRANSPORT REJIMI (Xostdan xostga aloqa uchun):
[ Asl IP Header ] | [ ESP Header ] | [ Shifrlangan TCP/Payload ] | [ ESP Trailer ] | [ ESP Auth ]
(Asl IP sarlavha ochiq qoladi, faqat ma'lumot qismi shifrlanadi).

2. TUNNEL REJIMI (VPN shlyuzlari o'rtasida — ENG ASOSIY REJIM):
[ YANGI TASHQI IP ] | [ ESP ] | [ ASL ICHKI IP ] | [ TCP ] | [ Data ] | [ ESP Trailer ] | [ ESP Auth ]
                                <------------ BUTUNLAY SHIFRLANGAN ------------>
```

* **Tunnel rejimining qudrati:**  
  Ichki korporativ IP-manzillar (`10.0.0.5` dan `10.2.0.8` ga) tashqi dushman ko‘zidan to‘liq yashiriladi! Butun boshli asl IP paket shifrlanadi va uning ustiga Internetda marshrutlanuvchi **yangi tashqi IP sarlavha** yopishtiriladi.

---

## 8.6. SIMSIZ TARMOQ XAVFSIZLIGI (WI-FI XAVFSIZLIGI)

Simsiz efir ochiq bo‘lgani sababli xaker ko‘chada mashinada o‘tirib ham tarmog‘ingizdagi barcha paketlarni tutib olishi mumkin.

1. **WEP (Wired Equivalent Privacy):**
   * Ilk Wi-Fi standarti. **Halokatli darajada zaif!**
   * RC4 oqimli shifrini va atigi 24-bitli ochiq IV (Initialization Vector) ishlatgan.
   * *Natija:* 24-bitli IV bir necha soatda takrorlanadi (Keystream reuse). `Aircrack-ng` vositasi yordamida WEP paroli 1 daqiqa ichida buzib tashlanadi!
2. **WPA2 (Wi-Fi Protected Access 2 / IEEE 802.11i):**
   * Mustahkam **AES-CCMP** shifrlashiga asoslangan.
   * Zaifligi: 4-Way Handshake paytida xaker ulanishni ushlab olsa, lug‘at orqali oflayn parolni taxmin qilib ko‘rishi mumkin (WPA2 Handshake Cracking).
3. **WPA3-SAE (Simultaneous Authentication of Equals / Dragonfly):**
   * Zamonaviy standart.
   * **Oflayn lug‘at hujumlariga mutlaqo chidamli:** Xaker millionlab parollarni kompyuterida tekshira olmaydi, har bir urinishda bevosita router bilan muloqot qilishi shart bo‘ladi.
   * **802.11w PMF (Protected Management Frames):** Wi-Fi boshqaruv kadrlarini (Deauthentication freymlarini) kriptografik himoyalaydi. Xaker endi soxta kadr yuborib, sizni tarmoqdan uzib tashlay olmaydi!

---

## 8.7. AMALIY XAVFSIZLIK: FIREWALLS VA IDS/IPS TIZIMLARI

### 8.7.1. Xavfsizlik Devorlari (Firewalls) Turlari

Firewall — bu tarmoq chegarasida turuvchi va o‘zidan o‘tayotgan har bir paketni xavfsizlik siyosati qoidalariga solishtirib, yo o‘tkazuvchi (**Accept / Permit**), yo yo‘q qiluvchi (**Drop / Deny**) tizimdir.

```
                          XAVFSIZLIK DEVORLARI TURLARI
                                       |
         +-----------------------------+-----------------------------+
         |                                                           |
         v                                                           v
STATELESS PACKET FILTER                                     STATEFUL PACKET INSPECTION
(Holatsiz filtrlash)                                        (Holatli nazorat — conntrack)
* Har bir paketni alohida ko'radi                           * Ulanishlar jadvalini tutadi
* Ulanish tarixini eslamaydi                                * NEW, ESTABLISHED, RELATED
* Oson aldash mumkin (SYN bo'lmagan TCP)                     * Javob paketlarini avtomatik o'tkazadi
```

1. **Stateless Packet Filter:** Paket sarlavhasidagi 5 ta parametrga qarab filtrlaydi: Manba IP, Maqsad IP, Protokol (TCP/UDP), Manba port, Maqsad port.
2. **Stateful Firewall (Holatli Filtr — Linux nftables/iptables):**
   * Linux yadrosidagi `conntrack` moduli har bir ochiq sessiyani kuzatib boradi.
   * *Qoidasi:* Agar ichki LAN foydalanuvchisi Google ga so‘rov yuborsa (holati: `NEW`), Google dan qaytayotgan javob paketi avtomatik ravishda `ESTABLISHED` holatida deb topiladi va xavfsizlik devoridan to‘siqsiz o‘tkaziladi!

---

### 8.7.2. DMZ (Demilitarized Zone — Demilitarizatsiyalangan Hudud) Arxitekturasi

Korxonaning ommaviy xizmatlari (Veb-server, Pochta) internetdan keluvchi barcha uchun ochiq bo‘lishi shart. Lekin ularni ichki LAN (buxgalteriya, xodimlar) tarmog‘iga qo‘yish o‘ta xavfli (agar veb-server buzilsa, butun tarmoq qo‘ldan ketadi).

**Yechim: 3 ta Zonalik Xavfsizlik Devori:**

```
                            [ GLOBAL INTERNET (WAN) ]
                                        |
                                        v
                            +-----------------------+
                            |   STATEFUL FIREWALL   |
                            +-----------------------+
                                   /         \
                                  /           \
                                 v             v
                    [ DMZ ZONASI ]             [ LAN ZONASI ]
                    * Veb-server (Port 80/443)  * Xodimlar kompyuterlari
                    * Mail server (Port 25)    * Maxfiy ma'lumotlar bazasi
                    (Internetdan kirish bor)   (Internetdan kirish MUTLAQO YOPIQ!)
```

* **Xavfsizlik Siyosati Qoidalari:**
  1. WAN $\to$ DMZ: Faqat belgilangan portlarga (80, 443) ruxsat.
  2. WAN $\to$ LAN: Qat’iy taqiqlangan (`DROP`).
  3. LAN $\to$ WAN / DMZ: Ruxsat berilgan.
  4. DMZ $\to$ LAN: Qat’iy taqiqlangan! (Agar xaker DMZ dagi veb-serverni buzib olsa ham, u ichki korporativ LAN ga o‘ta olmaydi).

---

### 8.7.3. Hujumlarni Aniqlash va Oldini Olish: IDS vs IPS

Firewall faqat portlar va IP larni tekshiradi, lekin u qonuniy 80-port ichida kelayotgan zararli SQL Injection yoki virusni aniqlay olmaydi. Buning uchun **IDS/IPS tizimlari (Suricata, Snort)** ishlatiladi.

```
+------------------------------------+------------------------------------+
| IDS (Intrusion Detection System)   | IPS (Intrusion Prevention System)  |
+------------------------------------+------------------------------------+
| * Passiv kuzatuvchi                | * Faol to'siq (Inline rejimda)     |
| * Nusxalangan trafikni (SPAN)      | * Paket router orqali to'g'ridan-  |
|   tahlil qiladi                    |   to'g'ri uning ichidan o'tadi     |
| * Hujumni sezsa faqat administrator| * Hujumni sezishi bilanoq paketni  |
|   uchun OGOHLANTIRISH (Alert) yozadi|   yo'lda BLOKLAYDI (Drop qiladi)   |
+------------------------------------+------------------------------------+
```

#### Tahlil Usullari:
1. **Signature-Based (Imzolarga asoslangan):** Hujumlarning ma’lum andozalari (qoidalari) bazasi bilan solishtiradi.
   * *Suricata Qoidasi Misoli:*
     ```suricata
     alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (msg:"EXPLOIT SQL Injection urinishi"; content:"UNION SELECT"; nocase; sid:1000001;)
     ```
2. **Anomaly-Based (Anomaliyalarga asoslangan):** Tarmoqdagi odatiy me’yorni (baseline) o‘rganadi. Agar to‘satdan odatdagidan 1000 marta ko‘p ICMP so‘rovlari yoki noodatiy portlar oqimi boshlansa, hatto yangi (Zero-Day) hujum bo‘lsa ham, uni anomaliya sifatida ushlaydi.

---

## 8.8. 8-BOB BO‘YICHA ASOSIY XULOSALAR VA AMALIYOTGA BOG‘LIQLIK

1. Xavfsizlikning asosi — **Maxfiylik, Yaxlitlik, Autentifikatsiya va Kirishni cheklashdir**.
2. **Simmetrik shifrlash (AES)** tezkor ma’lumotlarni shifrlash uchun, **Asimmetrik shifrlash (RSA)** esa kalitlarni almashish va raqamli imzolar uchun ishlatiladi.
3. **PKI va X.509 Sertifikatlari** ochiq kalitlarni soxtalashtirishdan himoya qiladi va **TLS/HTTPS** ning ishonch zanjirini quradi.
4. **IPsec VPN (Tunnel Mode)** butun tarmoqlar o‘rtasida xavfsiz shifrlangan korporativ yo‘lak ochadi.
5. **Stateful Firewall va DMZ** tarmoq perimetrini himoyalaydi, **Suricata IDS/IPS** esa paketlar ichidagi chuqur tahdidlarni aniqlaydi va bloklaydi.

---

### 🔗 JORIY PAPKADAGI BARCHA LABORATORIYALAR BILAN BEVOSITA INTEGRATSIYA:

Ushbu 8-bob siz turgan papkadagi amaliyotlarning **markaziy yadrosidir**:

* **`A1_project`:** Tarmoq aktivlari, zaifliklar va xatarlarni tahlil qilish (Threat Modeling).
* **`A2_project`:** **TLS, SSH va IPsec** protokollari yordamida ko‘p qatlamli himoya kanallarini sozlash.
* **`A4_project`:** Linux **nftables** yordamida Stateful Firewall, LAN SNAT va DMZ DNAT siyosatlarini amalga oshirish.
* **`A5_project`:** **Suricata IDS/IPS** tizimini o‘rnatish, qoidalar yozish va Linux **NFQUEUE** orqali hujumlarni faol bloklash.
* **`A6_project`:** **WireGuard VPN** yordamida eng kam ruxsatli masofaviy kirish arxitekturasini qurish.
* **`A7_project`:** **DNSSEC, STARTTLS va SPF-DKIM-DMARC** pochta xavfsizligi mexanizmlari.
* **`A8_project`:** Simsiz tarmoqlarda **WPA3-SAE va 802.11w PMF** majburiy himoyasini sozlash.
* **`A12_project`:** **DDoS** hujumlariga qarshi ko‘p qatlamli himoya (`synproxy`, rate limiting).
* **`A13_project`:** **Apache Reverse Proxy + ModSecurity WAF** bilan veb-ilovalarni himoyalash.
* **`A16_project`:** Router xavfsizligini mustahkamlash (**Hardening**) va audit o‘tkazish.
* **`A17_project`:** Tarmoq xavfsizligi insidentlarini boshqarish (**Incident Response va Digital Forensics**)!
